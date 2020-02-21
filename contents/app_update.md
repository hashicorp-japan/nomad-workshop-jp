# アプリケーションのアップデート

Nomadではアプリケーションを設定ベースでアップデートするための仕組みが備わっており、 ジョブ定義ファイルの`update`スタンザに記述することで実現可能です。

アップデートには様々な設定を行うことができ、`Rolling`と`Bleu-Green (a.k.a Canary)`という二つの方法が用意されています。

## Rolling Updateを試す

まずは`Rolling`を試してみます。

最初に題材となるアプリをデプロイします。以下のようなJobファイルを準備します。

```shell
$ cat << EOF > my-first-update.nomad
job "update-demo-webapp" {
  datacenters = ["dc1"]

  group "update-demo-webapp" {
    count = 10
    task "server" {

      env {
        PORT    = "\${NOMAD_PORT_http}"
        NODE_IP = "\${NOMAD_IP_http}"
      }

      driver = "docker"
      config {
        image = "hashicorp/demo-webapp:v1"
      }

      resources {
        cpu = 100
        memory = 32

        network {
          mbits = 10
          port "http" {}
        }
      }
    }
  }
}
EOF
```

このjobファイルでは、`image = "hashicorp/demo-webapp:v1"`というコンテナを10個デプロイします。
それでは以下のコマンドでデプロイしてみましょう。

```shell
$ nomad job run my-first-update.nomad
```

Jobが起動したか以下のコマンドで確認してみましょう。

```console
$  nomad job status update-demo-webapp
ID            = update-demo-webapp
Name          = update-demo-webapp
Submit Date   = 2020-02-05T16:57:24+09:00
Type          = service
Priority      = 50
Datacenters   = dc1
Status        = running
Periodic      = false
Parameterized = false

Summary
Task Group          Queued  Starting  Running  Failed  Complete  Lost
update-demo-webapp  0       0         10       0       0         0

Latest Deployment
ID          = f488febb
Status      = successful
Description = Deployment completed successfully

Deployed
Task Group          Desired  Placed  Healthy  Unhealthy  Progress Deadline
update-demo-webapp  10       10      10       0          2020-02-05T17:07:37+09:00

Allocations
ID        Node ID   Task Group          Version  Desired  Status   Created    Modified
19ff0a95  6477d9ed  update-demo-webapp  1        run      running  1m12s ago  59s ago
30693f24  6477d9ed  update-demo-webapp  1        run      running  1m12s ago  1m ago
d305cc7c  4b635091  update-demo-webapp  1        run      running  1m12s ago  1m ago
69465f26  6477d9ed  update-demo-webapp  1        run      running  1m12s ago  59s ago
99176943  4b635091  update-demo-webapp  1        run      running  1m12s ago  1m ago
b564afcb  4b635091  update-demo-webapp  1        run      running  1m12s ago  1m ago
581cd0ca  6477d9ed  update-demo-webapp  1        run      running  2m6s ago   1m2s ago
c2a8d9c9  4b635091  update-demo-webapp  1        run      running  2m6s ago   1m2s ago
30fd6816  8f5984b6  update-demo-webapp  1        run      running  2m6s ago   1m2s ago
dd5e8fe8  4b635091  update-demo-webapp  1        run      running  2m6s ago   1m2s ago
```

これで`image = "hashicorp/demo-webapp:v1"`というアプリケーションのバージョンが`v1`のものがデプロイされました。

アプリケーションが動いているか確認するために、Allocationからポート番号を確認しましょう。
Allocationのステータスを確認するには、`nomad alloc status <alloc_id>`コマンドを使います。`alloc_id`は上記のjob statusで表示されたものを一つ指定してください。

```console
$ nomad alloc status 659d44e1
ID                  = 659d44e1
Eval ID             = 6145d268
Name                = demo-webapp.demo[2]
Node ID             = 8c3b173d
Job ID              = demo-webapp
Job Version         = 4
Client Status       = running
Client Description  = Tasks are running
Desired Status      = run
Desired Description = <none>
Created             = 7m11s ago
Modified            = 6m57s ago

Task "server" is "running"
Task Resources
CPU        Memory          Disk     Addresses
2/100 MHz  2.0 MiB/32 MiB  300 MiB  http: 172.20.20.11:20009

Task Events:
Started At     = 2019-09-05T02:35:19Z
Finished At    = N/A
Total Restarts = 0
Last Restart   = N/A

Recent Events:
Time                       Type        Description
2019-09-05T11:35:19+09:00  Started     Task started by client
2019-09-05T11:35:16+09:00  Task Setup  Building Task Directory
2019-09-05T11:35:16+09:00  Received    Task received by client
```

この例では、`http://172.20.20.11:20009`でアプリケーションが動いてることがわかります。
アクセスしてみると、以下のように表示されます。

```console
$ curl http://172.20.20.11::20009
<!DOCTYPE html>
<html>
<body>

<h1 style="color:red;">Welcome! This is <i>version 1</i> of your application!</h1>
<h1 style="color:red;">You are on node 172.20.10.4</h1>

</body>
</html>
````

次に、このアプリケーションを`v2`にアップデートしてみましょう。
それを行うには、アップデート用のJobファイルを準備します。まずは最低限の設定でアップデートを行ってみます。

```shell
$ cat << EOF > my-first-update-v2.nomad
job "update-demo-webapp" {
  datacenters = ["dc1"]
  update {
  }
  group "update-demo-webapp" {
    count = 10
    task "server" {

      env {
        PORT    = "\${NOMAD_PORT_http}"
        NODE_IP = "\${NOMAD_IP_http}"
      }

      driver = "docker"
      config {
        image = "hashicorp/demo-webapp:v2"
      }

      resources {
        cpu = 100
        memory = 32

        network {
          mbits = 10
          port "http" {}
        }
      }
    }
  }
}
EOF
```

追加した部分は以下になります。

```hcl
update {
}
```

この設定を入れることでローリングアップグレードが有効となり、デフォルトではアプリケーションを一つづつアップデートします。
それではアップデートを行ってみましょう。その前にジョブの挙動を確認するためにジョブのステータスを監視しておきます。

別の端末で下記を実行してください。

```shell
$ watch -n 1 nomad job status update-demo-webapp
```

元の端末に戻り下記を実行します。

```shell
$ nomad run my-first-update-v2.nomad
```

`watch`コマンドの実行結果を見ると各アプリケーションが順番にアップデートされていきます。
再度アプリケーションのIPアドレスとポートを確認してアクセスしてみます。

```console
$ curl http://172.20.10.4:23049
<!DOCTYPE html>
<html>
<body>

<h1 style="color:blue;">Welcome! This is <i>version 2</i> of your application!</h1>
<h1 style="color:blue;">You are on node 172.20.10.4</h1>

</body>
</html>
```

バージョンがv2になっていることが確認できます。

次はローリングアップデートで設定できる項目をいくつか利用してアップデートしてみたいと思います。今回は便宜上コンテナとしては`v2`から`v1`への切り戻しを行いますが、バージョンアップだと想定してください。(出力文字列が変わっているだけです。)

```shell
$ cat << EOF > my-first-update-v3.nomad
job "update-demo-webapp" {
  datacenters = ["dc1"]

  update {
    max_parallel = 3
    health_check  = "task_states"
    min_healthy_time = "5s"
    healthy_deadline = "10m"
    progress_deadline = "20m"
  }

  group "update-demo-webapp" {
    count = 10
    task "server" {

      env {
        PORT    = "\${NOMAD_PORT_http}"
        NODE_IP = "\${NOMAD_IP_http}"
      }

      driver = "docker"
      config {
        image = "hashicorp/demo-webapp:v1"
      }

      resources {
        cpu = 100
        memory = 32

        network {
          mbits = 10
          port "http" {}
        }
      }
    }
  }
}
EOF
```

ここでは五つのパラメータを追加しました。

* `max_parallel`: 同時にアップデートするタスクの数を指定
* `health_check`: ヘルスチェックの種類を指定
    `checks`: タスクが正常に起動しているかに加えて特定のエンドポイントが正常かなどもチェック(Consulが必要)
    `task_state`: タスクの状態のみチェック
* `min_healthy_time`: `healthy`のステートに移行するまでの最小限の時間を指定
* `healthy_deadline`: `healthy`のステートに移行するまでの最大時間を指定。これ以降は自動的に`unhealthy`とされる
* `progress_deadline`: deploymentの中の最初のアロケーションが開始され、最初のアロケーションが成功するまでの時間を指定。この時間を過ぎるとdeploymentを`failed`とされる

それではこれをデプロイしてみます。今回はアップデートにおける正しい手順である`nomad job plan`コマンドを実行して変更点を確認してから適用したいと思います。

```console
$ nomad job plan my-first-update-v3.nomad
+/- Job: "update-demo-webapp"
+/- Task Group: "update-demo-webapp" (1 create/destroy update, 9 ignore)
  +/- Update {
        AutoPromote:      "false"
        AutoRevert:       "false"
        Canary:           "0"
    +/- HealthCheck:      "task_states" => "checks"
    +/- HealthyDeadline:  "300000000000" => "600000000000"
    +/- MaxParallel:      "1" => "3"
    +/- MinHealthyTime:   "10000000000" => "5000000000"
    +/- ProgressDeadline: "600000000000" => "1200000000000"
      }
  +/- Task: "server" (forces create/destroy update)
    +/- Config {
      +/- image: "hashicorp/demo-webapp:v2" => "hashicorp/demo-webapp:v1"
        }

Scheduler dry-run:
- All tasks successfully allocated.

Job Modify Index: 14808
To submit the job with version verification run:

nomad job run -check-index 14808 my-first-update.nomad

When running the job with the check-index flag, the job will only be run if the
server side version matches the job modify index returned. If the index has
changed, another user has modified the job and the plan's results are
potentially invalid.
```

`plan`コマンドによりアップデートをする際の差分を事前に把握し、安全に作業を行うことができます。

それでは、適用してみます。先ほどの`watch`コマンドの出力を見ながら実施してください。

```shell
$ nomad job run my-first-update-v3.nomad
```

`watch`コマンドの実行結果を見ると各アプリケーションが順番にアップデートされていきます。先ほどと違い3つずつアプリが更新されていることがわかるはずです。

以上のように宣言型でアップデートの設定をすることで簡単にローリングアップグレードが出来ることがわかりました。次に`Bleu-Green(a.k.a. Canary)`を試してみましょう。

## Canary Updateを試す

ローリングアップデートの場合、起動に失敗するようなアプリでも全てのインスタンスが次々にアップデートされてしまい場合によってはサービスに影響を及ぼします。

一方Canaryアップデートは`canary`で指定する代表的なインスタンスを最初にアップデートしそれが失敗するとその時点でアップデートを中止するような方法です。これを利用するとより安全にアプリケーションのアップデートを行うことが出来ます。

まず意図的に起動しないアプリへアップデートを試してみます。ここでは便宜上全く別のダミーアプリへのバージョンアップをしていますが、気にしないでください(同じアプリでうまく起動しないものと想定してください)。

```shell
$ cat << EOF > my-first-update-v4.nomad
job "update-demo-webapp" {
  datacenters = ["dc1"]

  update {
    max_parallel = 1
    health_check  = "task_states"
    min_healthy_time = "5s"
    healthy_deadline = "1m"
    progress_deadline = "2m"
    canary = 3
  }

  group "update-demo-webapp" {
    count = 10
    task "server" {

      env {
        PORT    = "\${NOMAD_PORT_http}"
        NODE_IP = "\${NOMAD_IP_http}"
      }

      driver = "docker"
      config {
        image = "dummy/dummy:v99.9"
      }

      resources {
        cpu = 100
        memory = 32

        network {
          mbits = 10
          port "http" {}
        }
      }
    }
  }
}
EOF
```

ここでは`image = "dummy/dummy:v99.9"`とし、ダミーのコンテナーを指定いるため、起動時にエラーとなります。このジョブを稼働させてみましょう。

```shell
$ nomad job run my-first-update-v4.nomad
```

`watch`の出力結果を見ると`running`の10インスタンスとは別に`canary`で指定した3インスタンスが新たに稼働しているでしょう。少し時間が経つと`failed`となり、そこでアップデートが中断されます。

```
ID            = update-demo-webapp
Name          = update-demo-webapp
Submit Date   = 2020-02-06T13:01:31+09:00
Type          = service
Priority      = 50
Datacenters   = dc1
Status        = running
Periodic      = false
Parameterized = false

Summary
Task Group          Queued  Starting  Running  Failed  Complete  Lost
update-demo-webapp  0       0         10       13      10        0

Future Rescheduling Attempts
Task Group          Eval ID   Eval Time
update-demo-webapp  b39ee3ae  25s from now

Latest Deployment
ID          = 55bef947
Status      = running
Description = Deployment is running but requires manual promotion

Deployed
Task Group          Promoted  Desired  Canaries  Placed  Healthy  Unhealthy  Progress Deadline
update-demo-webapp  false     10       3         3       0        3          2020-02-06T13:21:32+09:0
0

Allocations
ID        Node ID   Task Group          Version  Desired  Status    Created    Modified
111888dc  6477d9ed  update-demo-webapp  4        run      failed    46s ago    1s ago
2a1da85b  6477d9ed  update-demo-webapp  4        run      failed    46s ago    1s ago
adc033a5  6477d9ed  update-demo-webapp  4        run      failed    46s ago    1s ago
0ad95958  8f5984b6  update-demo-webapp  3        run      running   1m23s ago  1m11s ago
18ea57f8  8f5984b6  update-demo-webapp  3        run      running   1m24s ago  1m11s ago
f50f78ce  8f5984b6  update-demo-webapp  3        run      running   1m24s ago  1m11s ago
d0776f17  4b635091  update-demo-webapp  3        run      running   1m24s ago  1m11s ago
f4095268  4b635091  update-demo-webapp  3        run      running   1m24s ago  1m11s ago
1e3956ea  4b635091  update-demo-webapp  3        run      running   1m25s ago  1m13s ago
~~~~~
```

次に定義ファイルを更新して正常にCanaryアップデートを試してみます。

```shell
$ cat << EOF > my-first-update-v5.nomad
job "update-demo-webapp" {
  datacenters = ["dc1"]

  update {
    max_parallel = 2
    health_check  = "task_states"
    min_healthy_time = "5s"
    healthy_deadline = "1m"
    progress_deadline = "2m"
    canary = 3
  }

  group "update-demo-webapp" {
    count = 10
    task "server" {

      env {
        PORT    = "\${NOMAD_PORT_http}"
        NODE_IP = "\${NOMAD_IP_http}"
      }

      driver = "docker"
      config {
        image = "hashicorp/demo-webapp:v2"
      }

      resources {
        cpu = 100
        memory = 32

        network {
          mbits = 10
          port "http" {}
        }
      }
    }
  }
}
EOF
```

今度は正常に稼働するコンテナのため、`canary`の稼働が完了し、残りのインスタンスをローリングでアップデートしていきます。まず更新をプランしてみましょう。

```console
$ nomad job plan my-first-update-v5.nomad
+/- Job: "update-demo-webapp"
+/- Task Group: "update-demo-webapp" (3 destroy, 10 in-place update)
  +/- Update {
        AutoPromote:      "false"
        AutoRevert:       "false"
        Canary:           "3"
        HealthCheck:      "task_states"
    +/- HealthyDeadline:  "30000000000" => "60000000000"
        MaxParallel:      "1"
        MinHealthyTime:   "5000000000"
    +/- ProgressDeadline: "60000000000" => "120000000000"
      }
  +/- Task: "server" (forces create/destroy update)
    +/- Config {
      +/- image: "dummy/dummy:v99.9" => "hashicorp/demo-webapp:v2"
        }

Scheduler dry-run:
- All tasks successfully allocated.

Job Modify Index: 16466
To submit the job with version verification run:

nomad job run -check-index 16466 my-first-update-v5.nomad

When running the job with the check-index flag, the job will only be run if the
server side version matches the job modify index returned. If the index has
changed, another user has modified the job and the plan's results are
potentially invalid.
```

プラン内容を確認したらジョブをデプロイしてみます。

```shell
$ nomad job run my-first-update-v5.nomad
```

`watch`の出力結果を見ると`running`の10インスタンスとは別に`canary`で指定した3インスタンスが新たに稼働しているでしょう。今度は`running`になっているはずです。

```
ID            = update-demo-webapp
Name          = update-demo-webapp
Submit Date   = 2020-02-06T13:30:00+09:00
Type          = service
Priority      = 50
Datacenters   = dc1
Status        = running
Periodic      = false
Parameterized = false

Summary
Task Group          Queued  Starting  Running  Failed  Complete  Lost
update-demo-webapp  0       0         13       3       0         0

Latest Deployment
ID          = 997d92cc
Status      = running
Description = Deployment is running but requires manual promotion

Deployed
Task Group          Promoted  Desired  Canaries  Placed  Healthy  Unhealthy  Progress Deadline
update-demo-webapp  false     10       3         3       2        0          2020-02-06T13:32:07+09:00

Allocations
ID        Node ID   Task Group          Version  Desired  Status   Created    Modified
81ef93f3  8f5984b6  update-demo-webapp  5        run      running  6s ago     5s ago
38d64760  4b635091  update-demo-webapp  5        run      running  6s ago     0s ago
84daa24c  8f5984b6  update-demo-webapp  5        run      running  6s ago     0s ago
ec1c225e  8f5984b6  update-demo-webapp  4        stop     failed   1m4s ago   6s ago
17545de5  4b635091  update-demo-webapp  4        stop     failed   1m4s ago   6s ago
95fdd7ed  8f5984b6  update-demo-webapp  4        stop     failed   1m4s ago   6s ago
114e4000  4b635091  update-demo-webapp  3        run      running  1m20s ago  1m12s ago
3e2b3392  4b635091  update-demo-webapp  3        run      running  1m20s ago  1m12s ago
716c02bd  4b635091  update-demo-webapp  3        run      running  1m20s ago  1m12s ago
7ddddc9b  8f5984b6  update-demo-webapp  3        run      running  1m20s ago  1m13s ago
3b9e345c  8f5984b6  update-demo-webapp  3        run      running  1m20s ago  1m13s ago
3b139fad  4b635091  update-demo-webapp  3        run      running  1m20s ago  1m11s ago
1ca99ae5  8f5984b6  update-demo-webapp  3        run      running  1m20s ago  1m12s ago
e5afef9f  4b635091  update-demo-webapp  3        run      running  1m20s ago  1m12s ago
143c9fe9  6477d9ed  update-demo-webapp  3        run      running  1m20s ago  1m13s ago
f67715ee  8f5984b6  update-demo-webapp  3        run      running  1m20s ago  1m12s ago
~~~~~
```

Canaryの成功が確認された後、`promotion`を行うことで残りのインスタンスを`max_parallel`で指定した数ごとローリングアップデートしていきます。

```shell
$ nomad job promote update-demo-webapp
```

`watch`の出力結果を見るとローリングアップデートが始まっている様子がわかるでしょう。最終的にこのような状態になっていればOKです。

```
ID            = update-demo-webapp
Name          = update-demo-webapp
Submit Date   = 2020-02-06T13:30:00+09:00
Type          = service
Priority      = 50
Datacenters   = dc1
Status        = running
Periodic      = false
Parameterized = false

Summary
Task Group          Queued  Starting  Running  Failed  Complete  Lost
update-demo-webapp  0       0         10       3       10        0

Latest Deployment
ID          = 997d92cc
Status      = successful
Description = Deployment completed successfully

Deployed
Task Group          Promoted  Desired  Canaries  Placed  Healthy  Unhealthy  Progress Deadline
update-demo-webapp  true      10       3         10      10       0          2020-02-06T13:34:46+09:00

Allocations
ID        Node ID   Task Group          Version  Desired  Status    Created    Modified
416294b6  8f5984b6  update-demo-webapp  5        run      running   3m38s ago  3m30s ago
45654771  4b635091  update-demo-webapp  5        run      running   3m46s ago  3m39s ago
76f95103  6477d9ed  update-demo-webapp  5        run      running   3m55s ago  3m48s ago
4722c102  6477d9ed  update-demo-webapp  5        run      running   4m4s ago   3m56s ago
d2683b38  6477d9ed  update-demo-webapp  5        run      running   4m13s ago  4m5s ago
c8b3e8d7  6477d9ed  update-demo-webapp  5        run      running   4m21s ago  4m14s ago
a60a844d  6477d9ed  update-demo-webapp  5        run      running   4m30s ago  4m22s ago
38d64760  4b635091  update-demo-webapp  5        run      running   6m16s ago  25s ago
84daa24c  8f5984b6  update-demo-webapp  5        run      running   6m16s ago  25s ago
81ef93f3  8f5984b6  update-demo-webapp  5        run      running   6m16s ago  25s ago
95fdd7ed  8f5984b6  update-demo-webapp  4        stop     failed    7m14s ago  6m16s ago
17545de5  4b635091  update-demo-webapp  4        stop     failed    7m14s ago  6m16s ago
ec1c225e  8f5984b6  update-demo-webapp  4        stop     failed    7m14s ago  6m16s ago
114e4000  4b635091  update-demo-webapp  3        stop     complete  7m30s ago  4m29s ago
7ddddc9b  8f5984b6  update-demo-webapp  3        stop     complete  7m30s ago  4m3s ago
716c02bd  4b635091  update-demo-webapp  3        stop     complete  7m30s ago  4m29s ago
3e2b3392  4b635091  update-demo-webapp  3        stop     complete  7m30s ago  3m37s ago
3b9e345c  8f5984b6  update-demo-webapp  3        stop     complete  7m30s ago  4m29s ago
3b139fad  4b635091  update-demo-webapp  3        stop     complete  7m30s ago  4m11s ago
1ca99ae5  8f5984b6  update-demo-webapp  3        stop     complete  7m30s ago  4m20s ago
e5afef9f  4b635091  update-demo-webapp  3        stop     complete  7m30s ago  3m54s ago
143c9fe9  6477d9ed  update-demo-webapp  3        stop     complete  7m30s ago  4m29s ago
f67715ee  8f5984b6  update-demo-webapp  3        stop     complete  7m30s ago  3m46s ago
```

これでアップデートの内容は完了です。

この他に`auto_revert`で失敗したら自動で最新の安定のバージョンに切り戻す機能や`auto_promotion`で`canary`が成功したら自動でプロモーションするような設定も可能です。

以上のようにNomadでは設定ベースでのアップデートのスケジューリングをシンプルに実行できることがわかりました。Consulのヘルスチェックの機能と組み合わせることでさらに高度な設定が可能です。

この辺りは`HashiCorp Consulとの連携`の章で試してみたいと思います。

最後にジョブを停止しておきましょう。

```shell
$ nomad job stop update-demo-webapp
```

## 参考リンク
* [Nomad Job Lifecycle](https://www.hashicorp.com/blog/building-resilient-infrastructure-with-nomad-job-lifecycle/)
* [Rolling Upgrades](https://www.nomadproject.io/guides/operating-a-job/update-strategies/rolling-upgrades.html)
* [Canary Deployments](https://www.nomadproject.io/guides/operating-a-job/update-strategies/blue-green-and-canary-deployments.html)
* [job promote command](https://www.nomadproject.io/docs/commands/job/promote.html)
* [Update Configuration](https://www.nomadproject.io/docs/job-specification/update.html)