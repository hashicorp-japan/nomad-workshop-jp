# 初めての Nomad

ここでは Nomad を簡単に触ってみましょう。

まず作業用のディレクトリを作成しておきます。

```shell
$ mkdir nomad-workshop
$ cd nomad-workshop
```

[こちら](https://www.nomadproject.io/downloads.html)より、お使いの OS に合ったものをダウンロードしてください。

Nomad は他の HashiCorp 製品と同様にシングルバイナリですので、ダウンロードしたバイナリに Path を通すだけで使用可能です。

```console
$ unzip nomad*.zip
$ chmod + x nomad
$ mv nomad /usr/local/bin
$ nomad -version
Nomad v1.9.5
BuildDate 2025-01-14T18:35:12Z
Revision 0b7bb8b60758981dae2a78a0946742e09f8316f5+CHANGES
```

Nomad のバージョンが表示されるか確認してみましょう。

```console
$ nomad -version

Nomad v1.9.5
BuildDate 2025-01-14T18:35:12Z
Revision 0b7bb8b60758981dae2a78a0946742e09f8316f5+CHANGES
```
次に Dev モードでサーバーを起動してみます。

**もし MacOS をお使いで JDK や Java 関連のメッセージが出ましたら、[こちら](https://support.apple.com/kb/DL1572?locale=en_US)にある Java パッケージをインストールしてください。**

```shell
$ nomad agent -dev
```

> sudo で Docker を起動している場合は `sudo nomad agent -dev`

Nomad は通常実際のワークロードを稼働させる Client とスケジュールングなど管理系の機能を提供する Server を別オプションで起動させます。

Dev モードでは Nomad の機能を確認したりテストするのを容易にするため、サーバーととクライアント両方の特性を持って起動されます。また、Dev モードではリスナーやストレージの設定などがプリコンフィグレーションされています。

サーバーのステータスやリストを見るには以下のコマンドを実行します。

```console
$ nomad server members

Name                                Address    Port  Status  Leader  Raft Version  Build  Datacenter  Region
hiro.wakabayashi-JL42TVM46C.global  127.0.0.1  4648  alive   true    3             1.9.5  dc1         global
```

クライアントのステータスをみるには以下のコマンドを実行します。

```console
$ nomad node status
ID        Node Pool  DC   Name                         Class   Drain  Eligibility  Status
d7bbd92f  default    dc1  hiro.wakabayashi-JL42TVM46C  <none>  false  eligible     ready
```

簡単な Job を実行してみましょう。

Nomad にはサンプルの Job ファイルを作成する機能があります。

```shell
$ nomad job init -short     # -short オプションをつけるとコメント無しの Job ファイルが作成されます
```

ディレクトリ内に example.nomad.hcl というファイルが作られます。Nomad の Job ファイルは .nomad.hcl という拡張子になり、HCL のフォーマットが利用可能です。 \
Job ファイルの詳細については[こちら](https://www.nomadproject.io/docs/job-specification/index.html)を参照ください。

```shell
$ cat example.nomad.hcl

job "example" {

  group "cache" {
    network {
      port "db" {
        to = 6379
      }
    }

    task "redis" {
      driver = "docker"

      config {
        image          = "redis:7"
        ports          = ["db"]
        auth_soft_fail = true
      }

      identity {
        env  = true
        file = true
      }

      resources {
        cpu    = 500
        memory = 256
      }
    }
  }
}
```

Job ファイルには、「何を」「どこに」デプロイするかを*宣言的*に記述します。
この例では、Docker を使って `redis:7` イメージのコンテナを一つ起動します。また、500Mhz の CPU と 256MB のメモリ容量と 10Mbit/s のネットワーク帯域を満たしている Node でのみ実行されます。

では実際に実行してみましょう。

```console
$ nomad job run example.nomad.hcl 
==> 2025-02-03T11:37:15+09:00: Monitoring evaluation "b4f64e56"
    2025-02-03T11:37:15+09:00: Evaluation triggered by job "example"
    2025-02-03T11:37:16+09:00: Evaluation within deployment: "11915738"
    2025-02-03T11:37:16+09:00: Allocation "7ca34a50" created: node "d7bbd92f", group "cache"
    2025-02-03T11:37:16+09:00: Evaluation status changed: "pending" -> "complete"
==> 2025-02-03T11:37:16+09:00: Evaluation "b4f64e56" finished with status "complete"
==> 2025-02-03T11:37:16+09:00: Monitoring deployment "11915738"
  ⠦ Deployment "11915738" in progress...
# ...
```

実際にコンテナが起動したか確認してみましょう。

```console
$ docker container ls
CONTAINER ID   IMAGE                       COMMAND                  CREATED          STATUS          PORTS                                                  NAMES
eed3eb50795d   redis:7                     "docker-entrypoint.s…"   25 seconds ago   Up 24 seconds   127.0.0.1:25305->6379/tcp, 127.0.0.1:25305->6379/udp   redis-7ca34a50-34c4-c97d-99be-25b0d2db3f2e
```

`redis:7` が起動していることが確認できます。
ログは `nomad alloc logs` コマンドからみれます。引数に Allocation ID を指定します。今回の例では、Job を run したときの出力から Allocation ID は `7ca34a50` であることがわかります。

```shell
$ nomad alloc logs 7ca34a50  
1:C 03 Feb 2025 02:37:24.962 * oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 03 Feb 2025 02:37:24.962 * Redis version=7.4.2, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 03 Feb 2025 02:37:24.962 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
1:M 03 Feb 2025 02:37:24.962 * monotonic clock: POSIX clock_gettime
1:M 03 Feb 2025 02:37:24.962 * Running mode=standalone, port=6379.
1:M 03 Feb 2025 02:37:24.962 * Server initialized
1:M 03 Feb 2025 02:37:24.963 * Ready to accept connections tcp
```

それでは、次に Job ファイルに変更を加えてみましょう。起動するコンテナの数をデフォルトの 1 から 3 に増やしてみます。
Job ファイルを開き `group` スタンザに[`count` アトリビュート](https://www.nomadproject.io/docs/job-specification/group.html#count)を追加します。

```hcl
group "cache" {
  count = 3

  task "redis" {
```

Nomad には Job ファイルの変更により、何がどう変わるかを Plan してくれる機能があります。これにより、変更した内容の検証及びその変更が想定したものと問題ないか、事前に確認できます。

```console
$ nomad job plan example.nomad.hcl
+/- Job: "example"
+/- Task Group: "cache" (2 create, 1 in-place update)
  +/- Count: "1" => "3" (forces create)
      Task: "redis"

Scheduler dry-run:
- All tasks successfully allocated.

Job Modify Index: 17
To submit the job with version verification run:

nomad job run -check-index 17 example.nomad.hcl

When running the job with the check-index flag, the job will only be run if the
job modify index given matches the server-side version. If the index has
changed, another user has modified the job and the plan's results are
potentially invalid.
```

Plan 内容をみてみると、Count が 1 から 3 に変更されたことが確認できます。
新しい Job ファイルを実行してみましょう。

```console
$ nomad job run example.nomad.hcl
==> 2025-02-03T11:44:35+09:00: Monitoring evaluation "1c38209e"
    2025-02-03T11:44:35+09:00: Evaluation triggered by job "example"
    2025-02-03T11:44:36+09:00: Evaluation within deployment: "e3fa89f5"
    2025-02-03T11:44:36+09:00: Allocation "7ca34a50" modified: node "d7bbd92f", group "cache"
    2025-02-03T11:44:36+09:00: Allocation "68955b0e" created: node "d7bbd92f", group "cache"
    2025-02-03T11:44:36+09:00: Allocation "d64cb1dc" created: node "d7bbd92f", group "cache"
    2025-02-03T11:44:36+09:00: Evaluation status changed: "pending" -> "complete"
==> 2025-02-03T11:44:36+09:00: Evaluation "1c38209e" finished with status "complete"
==> 2025-02-03T11:44:36+09:00: Monitoring deployment "e3fa89f5"
  ⠸ Deployment "e3fa89f5" in progress...
# ...

```

しばらくおいて `docker container ls` でコンテナの情報を見てみましょう。

```console
$ docker container ls
CONTAINER ID   IMAGE                       COMMAND                  CREATED          STATUS          PORTS                                                  NAMES
8fdc9b553899   redis:7                     "docker-entrypoint.s…"   30 seconds ago   Up 30 seconds   127.0.0.1:21776->6379/tcp, 127.0.0.1:21776->6379/udp   redis-d64cb1dc-d53c-9256-b913-f320e80ac55b
2a592d954494   redis:7                     "docker-entrypoint.s…"   30 seconds ago   Up 30 seconds   127.0.0.1:23743->6379/tcp, 127.0.0.1:23743->6379/udp   redis-68955b0e-1e3c-6e98-37e9-eb6066b89447
eed3eb50795d   redis:7                     "docker-entrypoint.s…"   7 minutes ago    Up 7 minutes    127.0.0.1:25305->6379/tcp, 127.0.0.1:25305->6379/udp   redis-7ca34a50-34c4-c97d-99be-25b0d2db3f2e
```

Job ファイルの定義通りコンテナが 3 つ起動していることが確認できます。
さて、Nomad は Job ファイルに書かれている状態を維持するよう Job を監視します。そこで、強制的にひとつのコンテナを `kill` してみましょう。ここでは Container ID が `8fdc9b553899` のコンテナを終了させます。

```console
$ docker kill 8fdc9b553899
8fdc9b553899

$ docker container ls
CONTAINER ID   IMAGE                       COMMAND                  CREATED         STATUS         PORTS                                                  NAMES
2a592d954494   redis:7                     "docker-entrypoint.s…"   2 minutes ago   Up 2 minutes   127.0.0.1:23743->6379/tcp, 127.0.0.1:23743->6379/udp   redis-68955b0e-1e3c-6e98-37e9-eb6066b89447
eed3eb50795d   redis:7                     "docker-entrypoint.s…"   9 minutes ago   Up 9 minutes   127.0.0.1:25305->6379/tcp, 127.0.0.1:25305->6379/udp   redis-7ca34a50-34c4-c97d-99be-25b0d2db3f2e
```

コンテナの数がひとつ減っています。
Nomad は裏側でその変更を監視しており、本来の「あるべき姿」へ修正してくれます。
しばらく待ってから、再度 Docker の状態を確認してみましょう。

```console
$ docker container ls

CONTAINER ID   IMAGE                       COMMAND                  CREATED          STATUS          PORTS                                                  NAMES
6319d3df9005   redis:7                     "docker-entrypoint.s…"   37 seconds ago   Up 36 seconds   127.0.0.1:21776->6379/tcp, 127.0.0.1:21776->6379/udp   redis-d64cb1dc-d53c-9256-b913-f320e80ac55b
2a592d954494   redis:7                     "docker-entrypoint.s…"   3 minutes ago    Up 3 minutes    127.0.0.1:23743->6379/tcp, 127.0.0.1:23743->6379/udp   redis-68955b0e-1e3c-6e98-37e9-eb6066b89447
eed3eb50795d   redis:7                     "docker-entrypoint.s…"   10 minutes ago   Up 10 minutes   127.0.0.1:25305->6379/tcp, 127.0.0.1:25305->6379/udp   redis-7ca34a50-34c4-c97d-99be-25b0d2db3f2e
```

Job ファイルで定義した状態に戻っています。
この Nomad の作業状態をみるには、`nomad job status` コマンドを使います。

```console
$ nomad job status example
ID            = example
Name          = example
Submit Date   = 2025-02-03T11:44:35+09:00
Type          = service
Priority      = 50
Datacenters   = *
Namespace     = default
Node Pool     = default
Status        = running
Periodic      = false
Parameterized = false

Summary
Task Group  Queued  Starting  Running  Failed  Complete  Lost  Unknown
cache       0       1         2        0       0         0     0

Latest Deployment
ID          = e3fa89f5
Status      = successful
Description = Deployment completed successfully

Deployed
Task Group  Desired  Placed  Healthy  Unhealthy  Progress Deadline
cache       3        3       3        0          2025-02-03T11:54:45+09:00

Allocations
ID        Node ID   Task Group  Version  Desired  Status   Created     Modified
68955b0e  d7bbd92f  cache       1        run      running  4m36s ago   4m25s ago
d64cb1dc  d7bbd92f  cache       1        run      pending  4m36s ago   5s ago
7ca34a50  d7bbd92f  cache       1        run      running  11m56s ago  4m25s ago
```

1 つのコンテナが `pending` になっており、Nomad により Desired の状態になっているかがチェックされてることがわかります。


Nomad の Job を終了させるには `nomad job stop` コマンドを使います。

```console
$ nomad job stop example
==> 2025-02-03T11:52:15+09:00: Monitoring evaluation "2e13ac71"
    2025-02-03T11:52:15+09:00: Evaluation triggered by job "example"
    2025-02-03T11:52:16+09:00: Evaluation within deployment: "e3fa89f5"
    2025-02-03T11:52:16+09:00: Evaluation status changed: "pending" -> "complete"
==> 2025-02-03T11:52:16+09:00: Evaluation "2e13ac71" finished with status "complete"
==> 2025-02-03T11:52:16+09:00: Monitoring deployment "e3fa89f5"
  ✓ Deployment "e3fa89f5" successful
    
    2025-02-03T11:52:16+09:00
    ID          = e3fa89f5
    Job ID      = example
    Job Version = 1
    Status      = successful
    Description = Deployment completed successfully
    
    Deployed
    Task Group  Desired  Placed  Healthy  Unhealthy  Progress Deadline
    cache       3        3       3        0          2025-02-03T11:54:45+09:00
```

終了すると Nomad が Allocation した Job がクリーンアップされます。

## Nomad を通常モードで起動する

次は dev モードではなく通常モードで起動します。通常モードではサーバとクライアントを分けて起動することができたり、様々な柔軟な設定を行うことが出来ます。 今回はサーバ側を 1 台、クライアント側を 3 台とする構成で試してみます。

サーバ用に次のファイルを作ってください。

```shell
$ cd nomad-workshop
$ MY_PATH=$(pwd)

$ cat << EOF > nomad-local-config-server.hcl
data_dir  = "${MY_PATH}/local-nomad-data"

bind_addr = "127.0.0.1"

server {
  enabled          = true
  bootstrap_expect = 1
}

advertise {
  http = "127.0.0.1"
  rpc  = "127.0.0.1"
  serf = "127.0.0.1"
}
EOF
```

次にクライアント用のファイルを作ります。今回は全てローカルで起動し、それぞれのポート番号を変更する必要があるため、三つのファイルを作ります。

```shell
$ cat << EOF > nomad-local-config-client-1.hcl

data_dir  = "${MY_PATH}/local-cluster-data-1"

bind_addr = "127.0.0.1"

client {
  enabled = true
  servers = ["127.0.0.1:4647"]
}

advertise {
  http = "127.0.0.1"
  rpc  = "127.0.0.1"
  serf = "127.0.0.1"
}

ports {
  http = 5641
  rpc  = 5642
  serf = 5643
}
EOF

$ cat << EOF > nomad-local-config-client-2.hcl

data_dir  = "${MY_PATH}/local-cluster-data-2"

bind_addr = "127.0.0.1"

client {
  enabled = true
  servers = ["127.0.0.1:4647"]
}

advertise {
  http = "127.0.0.1"
  rpc  = "127.0.0.1"
  serf = "127.0.0.1"
}

ports {
  http = 5644
  rpc  = 5645
  serf = 5646
}
EOF

$ cat << EOF > nomad-local-config-client-3.hcl

data_dir  = "${MY_PATH}/local-cluster-data-3"

bind_addr = "127.0.0.1"

client {
  enabled = true
  servers = ["127.0.0.1:4647"]
}

advertise {
  http = "127.0.0.1"
  rpc  = "127.0.0.1"
  serf = "127.0.0.1"
}

ports {
  http = 5647
  rpc  = 5648
  serf = 5649
}
EOF
```

起動用のシェルを作ります。

```shell
$ cat << EOF > run.sh
#!/bin/sh
pkill nomad
pkill java

sleep 10

nomad agent -config=${MY_PATH}/nomad-local-config-server.hcl &

nomad agent -config=${MY_PATH}/nomad-local-config-client-1.hcl &
nomad agent -config=${MY_PATH}/nomad-local-config-client-2.hcl &
nomad agent -config=${MY_PATH}/nomad-local-config-client-3.hcl &
EOF
```

<details><summary>sudoでDockerを起動している場合</summary>
  
```shell
$ cat << EOF > run.sh
#!/bin/sh
sudo pkill nomad
sudo pkill java

sleep 10

sudo nomad agent -config=${MY_PATH}/nomad-local-config-server.hcl &

sudo nomad agent -config=${MY_PATH}/nomad-local-config-client-1.hcl &
sudo nomad agent -config=${MY_PATH}/nomad-local-config-client-2.hcl &
sudo nomad agent -config=${MY_PATH}/nomad-local-config-client-3.hcl &
EOF
```
</details>

Nomad を起動させてみましょう。

```shell
$ chmod +x run.sh
$ ./run.sh
```

`http://localhost:4646/ui/`にブラウザでアクセスし、一つのサーバと三つのクライアントが起動していることを確認してください。

> サーバ上で実行しローカルからブラウザにアクセスできない方はポートフォワーディングの設定をしてみて下さい。 
> macOS の場合はローカルマシンで ssh -L 4646:127.0.0.1:4646 <username>@<SERVERS_PUBLIC_IP> -N です。

試しに先ほどと同じジョブを起動させて見ます。

```console
$ nomad job run example.nomad
==> Monitoring evaluation "164d6cf5"
    Evaluation triggered by job "example"
    Allocation "8cf4812b" created: node "33a379fc", group "cache"
    Allocation "b8c71633" created: node "33a379fc", group "cache"
    Allocation "bb7db1e3" created: node "33a379fc", group "cache"
    Allocation "bb7db1e3" status changed: "pending" -> "running" (Tasks are running)
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "164d6cf5" finished with status "complete"
```

ここで表示される
* `Allocation`はメモしておいて下さい(上の例だと`bb7db1e3`)。
* `Evaluation`はメモしておいて下さい(上の例だと`164d6cf5`)。

これ以降、この環境を使って Nomad の様々な機能を試していきます。

## 参考リンク

* [Nomad Architecture](https://www.nomadproject.io/docs/internals/architecture.html)
* [Nomad Agent Configuration](https://www.nomadproject.io/docs/configuration/index.html)

