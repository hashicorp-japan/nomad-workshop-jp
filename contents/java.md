## NomadでJavaアプリケーションを動かす

NomadではDockerのようなコンテナのワークロードにとどまらず、スタンドアロンの`Java`, `RaW Exec`や`Qemu`など様々なタイプのTaskを実行できます。

各Taskはクライアント上の`Task Driver`によってリソースがIsolationされ実行されます。Task Driverはプラガブルで、各ドライバーの定義はJob定義のTaskn内の`plugin stanza`で設定します。

ここではいくつかのJavaアプリケーションをNomad上で動かしてみます。また、この章ではログフォワーディングも試してみます。

## Java Task Driverを扱う

Java Task Driverはその名の通り、Javaアプリケーションを実行させるためのDriverです。これを利用することでDocker Imageのpullや実行させるために必要なVolumeやネットワークの設定を宣言的に行うことが可能です。

まず一つ簡単なJavaアプリをNomad上で稼働させてみましょう。**ローカルにJavaがインストールされていることを確認してください。**

以下のようにJobの定義ファイルを作成します。

```shell
$ cd nomad-workshop
$ export DIR=$(pwd)
$ git clone https://github.com/tkaburagi/simplest-web
$ cd simplest-web
$ ./mvnw clean package -DskipTests
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

今回はJava Driverを利用するために`driver`の値に`java`と入れています。これでJavaのドライバを有効化し、`config`でJVMの設定を行うことが可能です。今回は`jar_path`に先ほどビルドしたアプリのパスと、`jvm_potion`にヒープサイズのMaxとMinを設定しています。JVM起動時に付与するオプションはここに設定します。

それでは起動してみましょう。

```shell
$ nomad job run java.nomad
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

GUIでも確認できるので興味のある方は`http://localhost:4646/ui/jobs`をブラウザから見てください。

一度Javaのプロセスを停止させ、自動再起動されることを確認してみましょう。

```shell
$ pkill java
```

もう一度`nomad job status hello-web-java`を実行すると`failed`のステータスになり、しばらくしてもう一度実行すると`running`のステータスにself-healingされるはずです。

完全にStopしJobを削除してみましょう。

```shell
$ nomad job stop -purge hello-web-java
```

## 外部のArtifact Sourceから取得する

今回はローカルでビルドしたアプリを起動しましたが、CIと組み合わせるような際には外部のArtifactレポジトリから取得したいです。試してみましょう。

```shell
$ cd /path/to/nomad-workshop
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

`artifact`スタンザを追加し、`source`にサンプルのパブリックアクセス可能なJarファイルを取得しています。取得にキーが必要な場合は`options`にキーの値を入力します。また、`options`には`checksum`なども入れることが可能です。

取得したアーティファクトはデフォルトだとNomadサーバの`local/:filename`に保存されますのでこちらを指定しています。

それでは起動してみましょう。

```console
$ nomad job run java.nomad

==> Monitoring evaluation "49b3e00f"
    Evaluation triggered by job "hello-web-java"
    Evaluation within deployment: "171bf67a"
    Allocation "f049149a" created: node "6477d9ed", group "java-web"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "49b3e00f" finished with status "complete"
```

しばらくするとアプリが起動しているでしょう。

```console
$ curl 127.0.0.1:7070
Hey you. This Jar is from S3
```

これはJava Driver以外にも様々なタイプのジョブで利用することが可能です。

## ログを扱う

次はロギングです。NomadではサーバやJobのログを保存し、サイズによってローテーションさせるような機能があります。この機能は`Docker`の章で実施した`Volume`や`Artifact`と同様、Java Driver特有の機能ではありませんが、ついでに試してみます。

Nomadでは各サーバで指定したデータの入るディレクトリにログは出力されます。今回はローカルファイルシステムを利用しているのではまずはJavaのアプリがホストされているノードを探して、そこのデータ用のディレクトリから実際のログを見てみます。

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

こちらの出力結果から、`running`と成功しているAllocationから`Node ID`を取得します。上の例だと`4b635091`に相当します。またAllocation IDもメモしてください。上の例だと`50fbdd29`に相当します。

これがJavaがホストされているノードです。このノードがどのディレクトリなのかを識別するために次のコマンドを実行してください。

```console
$ export NODE_ID=4b635091
$ nomad node status -verbose -json ${NODE_ID} | jq -r '.HTTPAddr'
127.0.0.1:5641
```

結果によって以下を実行してください。
* `127.0.0.1:5641` -> `cd ${DIR}/datadir/local-cluster-data-1`
* `127.0.0.1:5644` -> `cd ${DIR}/datadir/local-cluster-data-2`
* `127.0.0.1:5647` -> `cd ${DIR}/datadir/local-cluster-data-3`

ログのファイルは各アロケーションの`alloc/logs`以下に保存されています。

```shell
$ ls alloc/<ALLOCATION>/alloc/logs
webservice.stderr.0 webservice.stdout.0
```

`ALLOCATION`が複数ある場合は、先ほどメモしたAllocation IDが接頭辞になっているディレクトリを選択します。ここの例だと`50fbdd29-************`です。

`stderr.0`と`stdout.0`とあることがわかるでしょう。`stdout.0`の方を見ると、起動ログと以下のようなアプリログが確認できるはずです。

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

このように記述することで最大保管するファイル数と1ファイルの最大`MB`数を指定することができます。

### 参考リンク
* [Java Driver](https://www.nomadproject.io/docs/drivers/java.html)
* [Log Configuration](https://www.nomadproject.io/docs/job-specification/logs.html)
* [Artifact Configuration](https://www.nomadproject.io/docs/job-specification/artifact.html)