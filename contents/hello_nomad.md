# 初めてのNomad

ここではNomadを簡単に触ってみましょう。

[こちら](https://www.nomadproject.io/downloads.html)より、お使いのOSに合ったものをダウンロードしてください。

Nomadは他のHashiCorp製品と同様にシングルバイナリですので、ダウンロードしたバイナリにPathを通すだけで使用可能です。

Nomadのバージョンが表示されるか確認してみましょう。

```console
$ nomad -version

Nomad v0.11.1 (1cbb2b9a81b5715be2f201a4650293c9ae517b87)
```
次にDevモードでサーバーを起動してみます。

**もしMacOSをお使いでJDKやJava関連のメッセージが出ましたら、[こちら](https://support.apple.com/kb/DL1572?locale=en_US)にあるJavaパッケージをインストールしてください。**

```shell
$ nomad agent -dev
```
Nomadは通常実際のワークロードを稼働させるClientとスケジュールングなど管理系の機能を提供するServerを別オプションで起動させます。

DevモードではNomadの機能を確認したりテストするのを容易にするため、サーバーととクライアント両方の特性を持って起動されます。また、Devモードではリスナーやストレージの設定などがプリコンフィグレーションされています。

サーバーのステータスやリストを見るには以下のコマンドを実行します。

```console
$ nomad server members

Name                        Address    Port  Status  Leader  Protocol  Build  Datacenter  Region
masa-mackbook.local.global  127.0.0.1  4648  alive   true    2         0.9.5  dc1         global
```

クライアントのステータスをみるには以下のコマンドを実行します。

```console
$ nomad node status
ID        DC   Name                 Class   Drain  Eligibility  Status
33a379fc  dc1  masa-mackbook.local  <none>  false  eligible     ready
```

簡単なJobを実行してみましょう。

NomadにはサンプルのJobファイルを作成する機能があります。

```shell
$ nomad job init -short     # -shortオプションをつけるとコメント無しのJobファイルが作成されます
```

ディレクトリ内にexample.nomadというファイルが作られます。NomadのJobファイルは.nomadという拡張子になります。Jobファイルの詳細については[こちら](https://www.nomadproject.io/docs/job-specification/index.html)を参照ください。

```console
$ cat example.nomad

job "example" {
  datacenters = ["dc1"]

  group "cache" {
    task "redis" {
      driver = "docker"

      config {
        image = "redis:3.2"
        port_map {
          db = 6379
        }
      }

      resources {
        cpu    = 500
        memory = 256
        network {
          mbits = 10
          port "db" {}
        }
      }

      service {
        name = "redis-cache"
        tags = ["global", "cache"]
        port = "db"
        check {
          name     = "alive"
          type     = "tcp"
          interval = "10s"
          timeout  = "2s"
        }
      }
    }
  }
```

Jobファイルには、「何を」「どこに」デプロイするかを*宣言的*に記述します。
この例では、Dockerを使って`redis:3.2`イメージのコンテナを一つ起動します。また、500MhzのCPUと256MBのメモリ容量と10Mbit/sのネットワーク帯域を満たしているNodeでのみ実行されます。

では実際に実行してみましょう。

```console
$ nomad job run example.nomad

==> Monitoring evaluation "9b9e5f9b"
    Evaluation triggered by job "example"
    Evaluation within deployment: "a3022d8f"
    Allocation "701f3254" created: node "33a379fc", group "cache"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "9b9e5f9b" finished with status "complete"
```

実際にコンテナが起動したか確認してみましょう。

```console
$ docker ps

CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                                                  NAMES
8dd1bfd98166        redis:3.2           "docker-entrypoint.s…"   About a minute ago   Up About a minute   127.0.0.1:28646->6379/tcp, 127.0.0.1:28646->6379/udp   redis-0450729c-179f-b373-0cc9-513514275d91
```

`redis:3.2`が起動していることが確認できます。
ログは`nomad logs`コマンドからみれます。引数にAllocation IDを指定します。今回の例では、Jobをrunしたときの出力から`701f3254`であることがわかります。

```console
$ nomad logs 701f3254

1:C 29 Aug 06:53:41.954 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 3.2.12 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 1
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

1:M 29 Aug 06:53:41.955 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 29 Aug 06:53:41.955 # Server started, Redis version 3.2.12
1:M 29 Aug 06:53:41.955 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 29 Aug 06:53:41.955 * The server is now ready to accept connections on port 6379
```

それでは、次にJobファイルに変更を加えてみましょう。起動するコンテナの数をデフォルトの１から３に増やしてみます。
Jobファイルを開き`group`スタンザに[`count`アトリビュート](https://www.nomadproject.io/docs/job-specification/group.html#count)を追加します。

```hcl
group "cache" {
  count = 3

  task "redis" {
```

NomadにはJobファイルの変更により、何がどう変わるかをPlanしてくれる機能があります。これにより、変更した内容の検証及びその変更が想定したものと問題ないか、事前に確認できます。

```console
$ nomad job plan example.nomad

+/- Job: "example"
+/- Stop: "true" => "false"
+/- Task Group: "cache" (3 create)
  +/- Count: "1" => "3" (forces create)
      Task: "redis"

Scheduler dry-run:
- All tasks successfully allocated.

Job Modify Index: 254
To submit the job with version verification run:

nomad job run -check-index 254 example.nomad

When running the job with the check-index flag, the job will only be run if the
server side version matches the job modify index returned. If the index has
changed, another user has modified the job and the plan's results are
potentially invalid.
```

Plan内容をみてみると、Countが１から３に変更されたことが確認できます。
新しいJobファイルを実行してみましょう。

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

しばらくおいて`docker ps`でコンテナの情報を見てみましょう。

```console
$ docker ps

CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                  NAMES
94401e464ba7        redis:3.2           "docker-entrypoint.s…"   56 seconds ago      Up 55 seconds       127.0.0.1:31303->6379/tcp, 127.0.0.1:31303->6379/udp   redis-8cf4812b-c0fe-c817-acfa-3b42920743a2
834de1552749        redis:3.2           "docker-entrypoint.s…"   56 seconds ago      Up 56 seconds       127.0.0.1:28871->6379/tcp, 127.0.0.1:28871->6379/udp   redis-bb7db1e3-c015-a9fe-1388-dd3016791e8b
a47d0ae31dd2        redis:3.2           "docker-entrypoint.s…"   56 seconds ago      Up 55 seconds       127.0.0.1:23735->6379/tcp, 127.0.0.1:23735->6379/udp   redis-b8c71633-c333-c6e9-8217-a4c0bdddf180
```

Jobファイルの定義通りコンテナが３つ起動していることが確認できます。
さて、NomadはJobファイルに書かれている状態を維持するようJobを監視します。そこで、強制的にひとつのコンテナを`kill`してみましょう。ここではContainer IDが`94401e464ba7`のコンテナを終了させます。

```console
$ docker kill 94401e464ba7

94401e464ba7

$ docker ps

CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                  NAMES
834de1552749        redis:3.2           "docker-entrypoint.s…"   2 minutes ago       Up 2 minutes        127.0.0.1:28871->6379/tcp, 127.0.0.1:28871->6379/udp   redis-bb7db1e3-c015-a9fe-1388-dd3016791e8b
a47d0ae31dd2        redis:3.2           "docker-entrypoint.s…"   2 minutes ago       Up 2 minutes        127.0.0.1:23735->6379/tcp, 127.0.0.1:23735->6379/udp   redis-b8c71633-c333-c6e9-8217-a4c0bdddf180

```

コンテナの数がひとつ減っています。
Nomadは裏側でその変更を監視しており、本来の「あるべき姿」へ修正してくれます。
しばらく待ってから、再度Dockerの状態を確認してみましょう。

```console
$ docker ps

CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                  PORTS                                                  NAMES
d0d3dd071082        redis:3.2           "docker-entrypoint.s…"   1 second ago        Up Less than a second   127.0.0.1:31303->6379/tcp, 127.0.0.1:31303->6379/udp   redis-8cf4812b-c0fe-c817-acfa-3b42920743a2
834de1552749        redis:3.2           "docker-entrypoint.s…"   2 minutes ago       Up 2 minutes            127.0.0.1:28871->6379/tcp, 127.0.0.1:28871->6379/udp   redis-bb7db1e3-c015-a9fe-1388-dd3016791e8b
a47d0ae31dd2        redis:3.2           "docker-entrypoint.s…"   2 minutes ago       Up 2 minutes            127.0.0.1:23735->6379/tcp, 127.0.0.1:23735->6379/udp   redis-b8c71633-c333-c6e9-8217-a4c0bdddf180
```

Jobファイルで定義した状態に戻っています。
このNomadの作業状態をみるには、`nomad job status`コマンドを使います。

```console
$ nomad job status example

ID            = example
Name          = example
Submit Date   = 2019-08-30T13:42:31+09:00
Type          = service
Priority      = 50
Datacenters   = dc1
Status        = running
Periodic      = false
Parameterized = false

Summary
Task Group  Queued  Starting  Running  Failed  Complete  Lost
cache       0       1         2        1       5         0

Allocations
ID        Node ID   Task Group  Version  Desired  Status    Created     Modified
b65b0dba  33a379fc  cache       7        run      pending   0s ago      0s ago
8cf4812b  33a379fc  cache       7        stop     failed    24m5s ago   0s ago
b8c71633  33a379fc  cache       7        run      running   24m5s ago   23m51s ago
bb7db1e3  33a379fc  cache       7        run      running   24m5s ago   23m45s ago
701f3254  33a379fc  cache       5        stop     complete  22h12m ago  4h54m ago
```

1つのコンテナが`Failed`になっており、それを復旧すべく`Starting`のAllocationが実行されていることが確認できます。


NomadのJobを終了させるには`nomad job stop`コマンドを使います。

```console
$ nomad job stop example

==> Monitoring evaluation "8b5de473"
    Evaluation triggered by job "example"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "8b5de473" finished with status "complete"
```

終了するとNomadがAllocationしたJobがクリーンアップされます。

## Nomadを通常モードで起動する

次はdevモードではなく通常モードで起動します。通常モードではサーバとクライアントを分けて起動することができたり、様々な柔軟な設定を行うことが出来ます。 今回はサーバ側を1台、クライアント側を3台とする構成で試してみます。

サーバ用に次のファイルを作ってください。

```shell
$ mkdir nomad-workshop
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

sleep 10

nomad agent -config=${MY_PATH}/nomad-local-config-server.hcl &

nomad agent -config=${MY_PATH}/nomad-local-config-client-1.hcl &
nomad agent -config=${MY_PATH}/nomad-local-config-client-2.hcl &
nomad agent -config=${MY_PATH}/nomad-local-config-client-3.hcl &
EOF
```

Nomadを起動させてみましょう。

```shell
$ chmod +x run.sh
$ ./run.sh
```

`http://localhost:4646/ui/`にブラウザでアクセスし、一つのサーバと三つのクライアントが起動していることを確認してください。

これ以降、この環境を使ってNomadの様々な機能を試していきます。

## 参考リンク

* [Nomad Architecture](https://www.nomadproject.io/docs/internals/architecture.html)
* [Nomad Agent Configuration](https://www.nomadproject.io/docs/configuration/index.html)

