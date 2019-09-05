# アプリケーションのアップデート

さて、ここではアプリケーションのアップデートをやってみましょう。

まず以下のようなJobファイルを準備します。ファイル名はdemo-webapp.nomadにしてください。
`region`と`datacenters`の値は各自の設定に合わせてください。

```hcl
job "demo-webapp" {
	region = "local"
  datacenters = ["dc1"]

  group "demo" {
    count = 4
    task "server" {

      env {
        PORT    = "${NOMAD_PORT_http}"
        NODE_IP = "${NOMAD_IP_http}"
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

      service {
        name = "demo-webapp"
        port = "http"

        tags = [
          "masa_demo",
					"v1"
        ]

        check {
          type     = "http"
          path     = "/"
          interval = "2s"
          timeout  = "2s"
        }
      }
    }
  }
}
```

このjobファイルでは、`image = "hashicorp/demo-webapp:v1"`というコンテナを4つデプロイします。
それでは以下のコマンドでデプロイしてみましょう。

```shell
nomad run demo-webapp.nomad
```

Jobが起動したか以下のコマンドで確認してみましょう。

```console
$ nomad job status demo-webapp
ID            = demo-webapp
Name          = demo-webapp
Submit Date   = 2019-09-03T17:21:07+09:00
Type          = service
Priority      = 50
Datacenters   = consul_cluster
Status        = running
Periodic      = false
Parameterized = false

Summary
Task Group  Queued  Starting  Running  Failed  Complete  Lost
demo        0       0         4        0       16        1

Allocations
ID        Node ID   Task Group  Version  Desired  Status    Created     Modified
b7d758e5  8c3b173d  demo        2        run      running   6m59s ago   6m43s ago
d4d65c78  8c3b173d  demo        2        run      running   6m59s ago   6m44s ago
bcdd416f  8c3b173d  demo        2        run      running   6m59s ago   6m42s ago
d33a9399  8c3b173d  demo        2        run      running   6m59s ago   6m43s ago
```

これで`image = "hashicorp/demo-webapp:v1"`というアプリケーションのバージョンが`v1`のものがデプロイされました。

アプリケーションが動いているか確認するために、Allocationからポート番号を確認しましょう。
Allocationのステータスを確認するには、`nomad alloc status <alloc_id>`コマンドを使います。`alloc_id`は上記のjob statusで表示されたものを指定してください。

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

この例では、`http: 172.20.20.11:20009`でアプリケーションが動いてることがわかります。
アクセスしてみると、以下のように表示されます。
![app_v1](https://user-images.githubusercontent.com/45160975/64308132-383bc100-cfd3-11e9-8117-caa4d038593a.png)

次に、このアプリケーションを`v2`にアップデートしてみましょう。
それを行うには、アップデート用のJobファイルを準備します。

先ほどのJobファイルをコピーしてdemo-webapp-update.nomadなどにしてください。そして、以下のように書き換えます。

```hcl
job "demo-webapp-masademo" {
	region = "local"
  datacenters = ["dc1"]

  update {
    max_parallel     = 1
    min_healthy_time = "10s"
    healthy_deadline = "1m"
    auto_revert      = true
  }

  group "demo" {
    count = 4
    task "server" {

      env {
        PORT    = "${NOMAD_PORT_http}"
        NODE_IP = "${NOMAD_IP_http}"
      }

      driver = "docker"
      config {
        image = "hashicorp/demo-webapp:v2"
      }

      resources {
				memory = 32

        network {
          mbits = 10
          port "http" {}
        }
      }

      service {
        name = "demo-webapp"
        port = "http"

        tags = [
					"masa_demo",
					"v2"
        ]

        check {
          type     = "http"
          path     = "/"
          interval = "2s"
          timeout  = "2s"
        }
      }
    }
  }
}
```

追加した部分は以下になります。

```hcl
  update {
    max_parallel     = 1
    min_healthy_time = "10s"
    healthy_deadline = "1m"
    auto_revert      = true
  }
```

この設定では、アプリケーションを一つづつアップデートするローリングアップデートを行います。





