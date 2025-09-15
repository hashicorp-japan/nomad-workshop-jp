## Nomad で Java アプリケーションを動かす

Nomad では Docker のようなコンテナのワークロードにとどまらず、スタンドアロンの`Java`, `RaW Exec`や`Qemu`など様々なタイプの Task を実行できます。

各 Task はクライアント上の`Task Driver`によってリソースが Isolation され実行されます。Task Driver はプラガブルで、各ドライバーの定義は Job 定義の Taskn 内の`plugin stanza`で設定します。

ここではいくつかの Java アプリケーションを Nomad 上で動かしてみます。また、この章では Nomad でのログ管理も試してみます。

## Java Task Driver を扱う

Java Task Driver はその名の通り、Java アプリケーションを実行させるための Driver です。これを利用することで Docker Image の pull や実行させるために必要な Volume やネットワークの設定を宣言的に行うことが可能です。

まず一つ簡単な Java アプリを Nomad 上で稼働させてみましょう。

**ローカルに Java がインストールされていることを確認してください。**

**Linux OS の方はローカルマシンのファイルをプロセスから取得することが不可能なためこの手順はスキップして「外部の Artifact Source から取得する」に進んでください。**

以下のように Job の定義ファイルを作成します。

```shell
$ cd nomad-workshop
$ export DIR=$(pwd)
$ git clone https://github.com/tkaburagi/simplest-web
$ cd simplest-web
$ ./mvnw clean package -DskipTests
$ cd ..
$ cat << EOF > java.nomad
job "hello-web-java" {
  datacenters = ["dc1"]

  type = "service"

  group "java-web" {
    count = 1
    task "java-web-task" {
      driver = "java"
      config {
        jar_path    = "${DIR}/simplest-web/target/demo-0.0.1-SNAPSHOT.jar"
        jvm_options = ["-Xmx512m", "-Xms256m"]
      }
      resources {
          cpu    = 500
          memory = 300
          network {
          port "http" {}
        }
      }
    }
  }
}
EOF
```

今回は Java Driver を利用するために`driver`の値に`java`と入れています。これで Java のドライバを有効化し、`config`で JVM の設定を行うことが可能です。今回は`jar_path`に先ほどビルドしたアプリのパスと、`jvm_potion`にヒープサイズの Max と Min を設定しています。JVM 起動時に付与するオプションはここに設定します。

それでは起動してみましょう。

```shell
$ nomad job run -hcl1 java.nomad
```

しばらくするとアプリが起動しているはずです。


```console
$ curl 127.0.0.1:7070
Hey you.

$ nomad job status hello-web-java

ID            = hello-web-java
Name          = hello-web-java
Submit Date   = 2020-01-29T19:26:02+09:00
Type          = service
Priority      = 50
Datacenters   = dc1
Status        = running
Periodic      = false
Parameterized = false

Summary
Task Group  Queued  Starting  Running  Failed  Complete  Lost
java-web    0       0         1        3       0         0

Latest Deployment
ID          = ac218ecd
Status      = successful
Description = Deployment completed successfully

Deployed
Task Group  Desired  Placed  Healthy  Unhealthy  Progress Deadline
java-web    1        4       1        3          2020-01-29T19:40:42+09:00

Allocations
ID        Node ID   Task Group  Version  Desired  Status   Created    Modified
24f8e8e9  6477d9ed  java-web    0        run      running  1m2s ago   52s ago
```

GUI でも確認できるので興味のある方は`http://localhost:4646/ui/jobs`をブラウザから見てください。

一度 Java のプロセスを停止させ、自動再起動されることを確認してみましょう。

```shell
$ pkill java
```

もう一度`nomad job status hello-web-java`を実行すると`failed`のステータスになり、しばらくしてもう一度実行すると`running`のステータスに self-healing されるはずです。

完全に Stop し Job を削除してみましょう。

```shell
$ nomad job stop -purge hello-web-java
```

## 外部の Artifact Source から取得する

今回はローカルでビルドしたアプリを起動しましたが、CI と組み合わせるような際には外部の Artifact レポジトリから取得したいです。試してみましょう。

```shell
$ cd nomad-workshop
$ cat << EOF > java.nomad
job "hello-web-java" {
  datacenters = ["dc1"]

  type = "service"

  group "java-web" {
    count = 1
    task "java-web-task" {
      driver = "java"
      artifact {
        source = "https://jar-tkaburagi.s3-ap-northeast-1.amazonaws.com/demo-0.0.1-SNAPSHOT.jar"
      }
      config {
        jar_path    = "local/demo-0.0.1-SNAPSHOT.jar"
        jvm_options = ["-Xmx2048m", "-Xms256m"]
      }
      resources {
          cpu    = 500
          memory = 300
          network {
              port "http" {}
          }
      }
    }
  }
}
EOF
```

`artifact`スタンザを追加し、`source`にサンプルのパブリックアクセス可能な Jar ファイルを取得しています。取得にキーが必要な場合は`options`にキーの値を入力します。また、`options`には`checksum`なども入れることが可能です。

取得したアーティファクトはデフォルトだと Nomad サーバの`local/:filename`に保存されますのでこちらを指定しています。

それでは起動してみましょう。

```console
$ nomad job run -hcl1 java.nomad

==> Monitoring evaluation "49b3e00f"
    Evaluation triggered by job "hello-web-java"
    Evaluation within deployment: "171bf67a"
    Allocation "f049149a" created: node "6477d9ed", group "java-web"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "49b3e00f" finished with status "complete"
```

しばらくするとアプリが起動しているでしょう。

```console
$ curl 127.0.0.1:8080
Hey you. This Jar is from S3
```

これは Java Driver 以外にも様々なタイプのジョブで利用することが可能です。

## ログを扱う

次はロギングです。Nomad ではサーバや Job のログを保存し、サイズによってローテーションさせるような機能があります。この機能は`Docker`の章で実施した`Volume`や`Artifact`と同様、Java Driver 特有の機能ではありませんが、ついでに試してみます。

Nomad では各サーバで指定したデータの入るディレクトリにログは出力されます。今回はローカルファイルシステムを利用しているのではまずは Java のアプリがホストされているノードを探して、そこのデータ用のディレクトリから実際のログを見てみます。

```console
$ nomad job status hello-web-java

ID            = hello-web-java
Name          = hello-web-java
Submit Date   = 2020-01-29T20:56:51+09:00
Type          = service
Priority      = 50
Datacenters   = dc1
Status        = running
Periodic      = false
Parameterized = false

Summary
Task Group  Queued  Starting  Running  Failed  Complete  Lost
java-web    0       0         1        4       4         0

Latest Deployment
ID          = 0cd38315
Status      = successful
Description = Deployment completed successfully

Deployed
Task Group  Desired  Placed  Healthy  Unhealthy  Progress Deadline
java-web    1        1       1        0          2020-01-29T21:07:56+09:00

Allocations
ID        Node ID   Task Group  Version  Desired  Status    Created     Modified
50fbdd29  4b635091  java-web    5        run      running   12m47s ago  11m41s ago
```

こちらの出力結果から、`running`と成功している Allocation から`Allocation ID`を取得します。上の例だと`50fbdd29`に相当します。

これが Java がホストされているノードです。このノードがどのディレクトリなのかを識別するために次のコマンドを実行してください。

```console
$ export ALLOC=50fbdd29
$ nomad fs ${ALLOC} alloc/logs

Mode        Size     Modified Time         Name
prw-------  0 B      2020-02-19T11:13:13Z  .java-web-task.stderr.fifo
prw-------  0 B      2020-02-19T11:13:17Z  .java-web-task.stdout.fifo
-rw-r--r--  0 B      2020-02-19T11:13:13Z  java-web-task.stderr.0
-rw-r--r--  1.8 KiB  2020-02-19T11:13:17Z  java-web-task.stdout.0
```

`stderr.0`と`stdout.0`とあることがわかるでしょう。`stdout.0`の方を見ると、起動ログと以下のようなアプリログが確認できるはずです。

```shell
$ nomad fs ${ALLOC} alloc/logs/java-web-task.stdout.0
```

```
2020-01-29 21:21:43.227  INFO 43147 --- [nio-7070-exec-3] com.example.demo.SimplestWeb             : API has been called!!
2020-01-29 21:21:44.254  INFO 43147 --- [nio-7070-exec-5] com.example.demo.SimplestWeb             : API has been called!!
```

ログは必ずこのディレクトリに出力されるため、`/opt/nomad/alloc/*/alloc/logs/*`のような設定でログをフォワードすることで各タスクのログをログサーバに集約することも可能です。

また、ログのローテーションの設定は各`task`スタンザ内に

```
logs {
	max_files     = 10
	max_file_size = 10
}
```

このように記述することで最大保管するファイル数と 1 ファイルの最大`MB`数を指定することができます。

最後にジョブを停止しておきましょう。

```shell
$ nomad job stop hello-web-java
```

### 参考リンク
* [Java Driver](https://www.nomadproject.io/docs/drivers/java.html)
* [Log Configuration](https://www.nomadproject.io/docs/job-specification/logs.html)
* [Artifact Configuration](https://www.nomadproject.io/docs/job-specification/artifact.html)