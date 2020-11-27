# Nomadでバッチジョブを動かす

今まで様々なTask Driverを利用してWebアプリケーションをメインに動かしてきました。Nomadではジョブ設定ファイルの`job`スタンザ内の`type`というパラメータに`Scheduler`を指定することでざまざまなタイプのワークロードを稼働させることが出来ます。

* Service Scheduler: 
	* Long runningなタスク。
	* OperatorがStopするまでRunning状態が維持される。
* Batch Scheduler: 
	* Short livedなタスク。
	* Taskが終了するまで実行される。
* System Scheduler: 
	* 全てのClient上で実行されるタスク。
	* OperatorがStopするか、PreemptedされるまでRunning状態を維持される。
	* 新たなClientがJoinした際にも起動される。

今までのWebのアプリケーションは`service`を指定しており、`exec.nomad`では`batch`を指定していました。

ここでは`Batch Scheduler`を利用して様々なジョブを実行してみたいと思います。

## 基本的なBatch Job

まずは簡単なSpring Boot Batchアプリのジョブを実行してみましょう。まずはアプリケーションの準備をします。

```shell
$ cd /path/to/nomad-workshop
$ export DIR=$(pwd)
$ git clone https://github.com/tkaburagi/gs-batch-processing
$ cd gs-batch-processing/complete
$ ./mvnw clean package -DskipTests
```

このアプリは`src/main/resources/sample-data.csv`ファイルに記載されているテキストを大文字に変換するための簡単なバッチ処理を実装したアプリです。

```console
$ cd /path/to/nomad-workshop
$ cat gs-batch-processing/complete/src/main/resources/sample-data.csv
Jill,Doe
Joe,Doe
Justin,Doe
Jane,Doe
John,Doe
```

次にこのアプリを実行するためのNomadのジョブ定義ファイルを作成します。

```shell
$ cat << EOF > hello-java-batch.nomad
job "hello-java-batch" {
  datacenters = ["dc1"]

  type = "batch"

  group "java-batch" {
    count = 1
    task "to-upper" {
      driver = "java"
      config {
        jar_path    = "${DIR}/gs-batch-processing/complete/target/batch-processing-0.0.1-SNAPSHOT.jar"
      }
      resources {
          cpu    = 500
          memory = 300
      }
    }
  }
}
EOF
```

<details><summary>Linuxの場合</summary>

```shell
cat << EOF > hello-java-batch.nomad
job "hello-java-batch" {
  datacenters = ["dc1"]

  type = "batch"

  group "java-batch" {
    count = 1
    task "to-upper" {
      driver = "java"
      artifact {
        source = "https://jar-tkaburagi.s3-ap-northeast-1.amazonaws.com/batch-processing-0.0.1-SNAPSHOT.jar"
      }
      config {
        jar_path    = "local/batch-processing-0.0.1-SNAPSHOT.jar"
        jvm_options = ["-Xmx2048m", "-Xms256m"]
      }
      resources {
          cpu    = 500
          memory = 300
      }
    }
  }
}
EOF
```
</details>

ここでは`batch`タイプのジョブを`java`のドライバを利用して実行する定義をしています。このジョブを実際に稼働させてみましょう。

```console
$ nomad job run hello-java-batch.nomad
==> Monitoring evaluation "87f0bff4"
    Evaluation triggered by job "hello-java-batch"
    Evaluation within deployment: "4bd45813"
    Allocation "a5059c59" created: node "4b635091", group "java-batch"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "87f0bff4" finished with status "complete"
```

ここで出力される`Allocation ID`を環境変数にセットしログを見てましょう。
**以下の手順はLinuxだと実行不可能です。[ISSUE](https://github.com/hashicorp/nomad/issues/6931)あり。**

```console
$ export ALLOC=a5059c59
$ nomad fs ${ALLOC} alloc/logs/to-upper.stdout.0
~~~~~~
2020-02-01 22:32:16.448  INFO 38164 --- [           main] c.e.batchprocessing.PersonItemProcessor  : Converting (firstName: Jill, lastName: Doe) into (firstName: JILL, lastName: DOE)
2020-02-01 22:32:16.448  INFO 38164 --- [           main] c.e.batchprocessing.PersonItemProcessor  : Converting (firstName: Joe, lastName: Doe) into (firstName: JOE, lastName: DOE)
2020-02-01 22:32:16.448  INFO 38164 --- [           main] c.e.batchprocessing.PersonItemProcessor  : Converting (firstName: Justin, lastName: Doe) into (firstName: JUSTIN, lastName: DOE)
2020-02-01 22:32:16.448  INFO 38164 --- [           main] c.e.batchprocessing.PersonItemProcessor  : Converting (firstName: Jane, lastName: Doe) into (firstName: JANE, lastName: DOE)
2020-02-01 22:32:16.448  INFO 38164 --- [           main] c.e.batchprocessing.PersonItemProcessor  : Converting (firstName: John, lastName: Doe) into (firstName: JOHN, lastName: DOE)
~~~~~~
```

以下のようなログが出力され、CSVファイルのデータが大文字に変換されていることがわかります。ジョブのステータスを確認しておきましょう。

```console
$ nomad job status hello-java-batch
ID            = hello-java-batch
Name          = hello-java-batch
Submit Date   = 2020-02-01T22:34:49+09:00
Type          = batch
Priority      = 50
Datacenters   = dc1
Status        = dead
Periodic      = false
Parameterized = false

Summary
Task Group  Queued  Starting  Running  Failed  Complete  Lost
java-batch  0       0         0        0       1         0

Allocations
ID        Node ID   Task Group  Version  Desired  Status    Created   Modified
3d04b6fc  4b635091  java-batch  0        run      complete  5m3s ago  5m ago
```

`status`が`dead`となっており停止していることがわかります。この状態のジョブは正常終了とされ、再起動などはされません。以上のように`type`に`batch`を指定することで一回限りのジョブを実行できることがわかります。

以下で`batch`のタイプのジョブの中でもNomadの機能を利用して様々なジョブを実行してみます。

## Parameterized Job

まずは`Prameterized Job`を試してみましょう。これはインプットされた値に対して特定の処理をするためのジョブで、ファンクションのように扱うことが可能です。このジョブをデプロイすると`nomad job dispatch`やAPIコールでinvokeすることが出来ます。ジョブをディスパッチするとペイロードやメタデータがインプットとしてジョブにインジェクションされ処理が実行されます。

`Prameterized Job`は`type`が`batch`である必要があります。

```shell
$ cd /path/to/nomad-workshop
$ export DIR=$(pwd)
$ PATH_TO_OPENSSL=$(which openssl)
$ cat << EOF > parameterized-encrypter.nomad
job "parameterized-encrypter" {
  datacenters = ["dc1"]

  type = "batch"

  parameterized {
    payload = "required"
  }

  group "parameterized-encrypter" {
    count = 1
    task "encrypter" {
      driver = "raw_exec"
      artifact {
        source = "https://jar-tkaburagi.s3-ap-northeast-1.amazonaws.com/pass.txt"
      }
      config {
        # When running a binary that exists on the host, the path must be absolute.
        command = "${PATH_TO_OPENSSL}"
        args    = ["aes-256-cbc", "-e", "-in", "\${NOMAD_TASK_DIR}/rawtext.txt", "-out", "${DIR}/encrypted.txt", "-pass", "file:local/pass.txt"]
      }
      dispatch_payload {
        file = "rawtext.txt"
      }
    }
  }
}
EOF
```

`type`に`batch`を指定し、ジョブスタンザ内に`prameterized`の設定を入れ`Prameterized Job`を有効化しています。インプットにはペイロードやメタデータを扱うことが出来ますが、まずはペイロードのみ`required`としています。

タスクスタンザ内が実際のタスクの定義で今回は`openssl`コマンドを使って入力された値を暗号化するとても簡単なタスクです。復号化に必要なパスワードのファイルは簡易的にパブリックアクセス可能なS3に保存されており、それを`artifact`で取得しています。

このジョブを稼働させるときにペイロードとしてテキストを入力すると、それが起動時にタスク内のディレクトリに`rawtext.txt`として保存され、処理内で扱うことが出来ます。

暗号化したファイルは`${DIR}/encrypted.txt`内に出力するようにしています。

それではまずはジョブを登録してみましょう。

```console
$ nomad job run parameterized-encrypter.nomad
Job registration successful
```

`nomad job run`を実行すると今までと違いジョブは`Evaluation`や`Allocation`されず登録だけが完了していることがわかります。

```console
$ nomad job status parameterized-encrypter
ID            = parameterized-encrypter
Name          = parameterized-encrypter
Submit Date   = 2020-02-02T12:03:31+09:00
Type          = batch
Priority      = 50
Datacenters   = dc1
Status        = running
Periodic      = false
Parameterized = true

Parameterized Job
Payload           = required
Required Metadata = <none>
Optional Metadata = <none>

Parameterized Job Summary
Pending  Running  Dead
0        0        0

Dispatched Jobs
```

ステータスを見ると、`Dispatched Jobs`が空となっています。それではこれをinvokeしてみましょう。このコマンドでは`nomad job dispatch <Parameterzed Job> <Input Source>`を使ってジョブをinvokeし、`Input Source`に`"this is a raw text!!"`を渡しています。

これが`raxtext.txt`として扱われファイルの暗号化を実施します。

```shell
$ cat << EOF | nomad job dispatch parameterized-encrypter -
"this is a raw text!!"
EOF
```

以下のような出力がされるはずなので`Allocation ID`を環境変数にセットしておきましょう。

```
Dispatched Job ID = parameterized-encrypter/dispatch-1580613031-6b384981
Evaluation ID     = f538cb71

==> Monitoring evaluation "f538cb71"
    Evaluation triggered by job "parameterized-encrypter/dispatch-1580613031-6b384981"
    Allocation "21ba8573" created: node "4b635091", group "parameterized-encrypter"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "f538cb71" finished with status "complete"
```

```shell
$ export ALLOC=21ba8573
```

もう一度ステータスを見ると`Dispatched Jobs`に先ほどの実行履歴が追加されているはずです。

```console
$ nomad job status -verbose parameterized-encrypter
ID            = parameterized-encrypter
Name          = parameterized-encrypter
Submit Date   = 2020-02-02T12:03:31+09:00
Type          = batch
Priority      = 50
Datacenters   = dc1
Status        = running
Periodic      = false
Parameterized = true

Parameterized Job
Payload           = required
Required Metadata = <none>
Optional Metadata = <none>

Parameterized Job Summary
Pending  Running  Dead
0        0        1

Dispatched Jobs
ID                                                    Status
parameterized-encrypter/dispatch-1580613031-6b384981  dead
```

タスクのディレクトリ内を見てみましょう。

```console
$ nomad fs ${ALLOC} encrypter/local/rawtext.txt
"this is a raw text!!"

$ nomad fs ${ALLOC} encrypter/local/pass.txt
password
```

コマンドで渡したインプットのファイルと`artifact`で取得したパスワードファイルが保存されています。

次に暗号化されたファイルを確認してみましょう。`${DIR}`内に出力されているはずです。

```console
$ cat encrypted.txt
Salted__ �-
           �Esf�U�S�8���;T��(-�z�����+�K%
```

暗号化されたデータが格納されています。余談ですが、興味のある方は以下の手順でファイルを復号化してみてください。

```console
$ curl https://jar-tkaburagi.s3-ap-northeast-1.amazonaws.com/pass.txt > pass.txt
$ openssl aes-256-cbc -d -in encrypted.txt -out decrypted.txt -pass file:./pass.txt
$ cat decrypted.txt
"this is a raw text!!"
```

このように`Parameterized Job`を扱うことで`batch`タイプのワンタイムのジョブをイベント処理のように利用することが可能です。

## Periodic Job

次に`Periodic Job`を扱います。これも`batch`タイプのジョブでスケジューリング可能な種類のジョブで`Periodic`は`定期的な`という意味です。

その名の通り`cron`で定期的なジョブを実行させるための機能です。

試してみましょう。

```shell
$ cat << EOF > periodic-echo.nomad
job "periodic-echo" {
  datacenters = ["dc1"]
  type = "batch"

  periodic {
    cron             = "*/1 * * * * *"
    prohibit_overlap = true
    time_zone = "Asia/Tokyo"
  }

  group "periodic-echo" {
    count = 1
    task "periodic-echo" {
      driver = "raw_exec"
      artifact {
        source = "https://jar-tkaburagi.s3-ap-northeast-1.amazonaws.com/periodic-task.sh"
      }
      config {
        # When running a binary that exists on the host, the path must be absolute.
        command = "local/periodic-task.sh"
      }
    }
  }
}
EOF
```

`periodic`の設定をジョブスタンザ内に入れています。`periodic`内に設定するのはスケジューリングの設定である`cron`, `time_zone`と同一ジョブをオーバーラップして実行させることを許可するかどうかの`prohibit_overlap`です。

`cron`は[こちら](https://github.com/gorhill/cronexpr#implementation)の記述方法に従い、`time_zone`はGolangが解釈できるフォーマットで記述します。

今回は`echo`で文字列を表示させ30秒スリープするだけの簡単なジョブにしています。これをNomadに登録していきます。

```console
$ nomad job run periodic-echo.nomad
Job registration successful
Approximate next launch time: 2020-02-02T13:04:00+09:00 (22s from now)
```

`parameterised`と同様にジョブ登録成功のメッセージと次の実行時間が表示されます。`watch`コマンド使ってJobのステータスを監視しましょう。

別のターミナルを立ち上げて以下のコマンドを実行してください。

```console
$ watch -n 1 nomad job status periodic-echo

ID                   = periodic-echo
Name                 = periodic-echo
Submit Date          = 2020-02-02T13:03:38+09:00
Type                 = batch
Priority             = 50
Datacenters          = dc1
Status               = dead (stopped)
Periodic             = true
Parameterized        = false
Next Periodic Launch = 2020-02-02T12:57:00+09:00 (22s from now)

Children Job Summary
Pending  Running  Dead
0        0        2

Previously Launched Jobs
ID                                 Status
periodic-echo/periodic-1580615160  dead
```

`Previously Launched Jobs`が最初は空ですがスケジューリングが始まると`running`になり、処理が終了すると`dead`になるはずです。また1分後に次の処理が実行され、`Previously Launched Jobs`に一つ履歴が増えるはずです。2回くらい様子を見てください。

このジョブのログを見て`Previously Launched Jobs`のリストの中の任意のジョブのIDをコピーしてください。

**Linuxの方はRunningのステータスのものを選択して下さい。**

```console
$ export JOB=periodic-echo/periodic-1580615160
$ nomad job status -verbose ${JOB}
ID            = periodic-echo/periodic-1580616420
Name          = periodic-echo/periodic-1580616420
Submit Date   = 2020-02-02T13:07:00+09:00
Type          = batch
Priority      = 50
Datacenters   = dc1
Status        = dead
Periodic      = false
Parameterized = false

Summary
Task Group     Queued  Starting  Running  Failed  Complete  Lost
periodic-echo  0       0         0        0       1         0

Evaluations
ID                                    Priority  Triggered By  Status    Placement Failures
027508eb-111f-92ab-65ec-ff432a546082  50        periodic-job  complete  false

Allocations
ID                                    Eval ID                               Node ID                               Node Name      Task Group     Version  Desired  Status    Created                    Modified
e91ce6b2-ab15-b490-c79a-f662e527dcb3  027508eb-111f-92ab-65ec-ff432a546082  4b635091-f444-34d6-5b33-a8aedcc49664  Takayukis-MBP  periodic-echo  0        run      complete  2020-02-02T13:07:00+09:00  2020-02-02T13:07:31+09:00
```

ここで一番下に出力される`Allocation ID`をコピーしてください。

```console
$ export ALLOC=e91ce6b2-ab15-b490-c79a-f662e527dcb3
$ nomad fs ${ALLOC} alloc/logs/periodic-echo.stdout.0
Hi Nomad Periodic Scheduler
```

以下のようにジョブが正しく実行出来ていることがわかるはずです。

ここではNomadの`Batch Schduler`を利用して

* ワンタイムのバッチ処理
* `Parameterized Job`を使ったイベント処理
* `Periodic Job`を使った定期処理

を実行させてみました。Nomadではこのように`Batch Scheduler`を使って様々なワークロードを稼働させることが可能です。

最後にジョブを停止しておきましょう。

```shell
$ nomad job stop periodic-echo
$ nomad job stop hello-java-batch
$ nomad job stop parameterized-encrypter
```

## 参考リンク
* [Batch Schduler](https://www.nomadproject.io/docs/schedulers.html#batch)
* [Batch Job Example](https://www.nomadproject.io/docs/job-specification/job.html#batch-job)
* [Parameterized Configuration](https://www.nomadproject.io/docs/job-specification/parameterized.html)
* [Dispatch Payload ](https://www.nomadproject.io/docs/job-specification/dispatch_payload.html)
* [dispatch command](https://www.nomadproject.io/docs/commands/job/dispatch.html)
* [Periodic Configuration](https://www.nomadproject.io/docs/job-specification/periodic.html)