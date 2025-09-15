## Nomad で Docker イメージを動かす

Nomad では Docker のようなコンテナのワークロードにとどまらず、スタンドアロンの`Java`, `RaW Exec`や`Qemu`など様々なタイプの Task を実行できます。

各 Task はクライアント上の`Task Driver`によってリソースが Isolation され実行されます。Task Driver はプラガブルで、各ドライバーの定義は Job 定義の Taskn 内の`plugin stanza`で設定します。

ここではいくつかの Docker イメージを Nomad 上で動かしてみます。また、この章では永続ディスクも試してみます。

## Docker Task Driver を扱う

Docker Task Driver はその名の通り、Docker Image を実行させるための Driver です。これを利用することで Docker Image の pull や実行させるために必要な Volume やネットワークの設定を宣言的に行うことが可能です。

まず一つ簡単な Docker Image を Nomad 上で稼働させてみましょう。

以下のように Job の定義ファイルを作成します。

```shell
$ cd nomad-workshop
$ export DIR=$(pwd)
$ cat << EOF > mysql.nomad
job "mysql-5.7" {
  datacenters = ["dc1"]

  type = "service"

  group "mysql-group" {
    count = 1
    task "mysql-task" {
      driver = "docker"
      config {
        image = "mysql:5.7.28"
        port_map {
          db = 3306
        }
      }
      env {
        "MYSQL_ROOT_PASSWORD" = "rooooot"
      }
      resources {
        cpu    = 500
        memory = 256

        network {
          mbits = 10
          port "db" {
            static = 3306
          }
        }
      }
    }
  }
}
EOF
```

`job`の`datacenters`にはデフォルトの`dc1`としています。`type`にはジョブのタイプをしていますが、今回は MySQL で Long Running Process なので`service`としています。

`group`が Task Group で`count`は Task の数です。今回は MySQL1 インスタントとしますので`1`です。`task`の中が実際のアプリです。`driver`は Docker を指定していて、それ以降が Docker 関連の`image`, `port_map`, `env`の設定です。

`image`はデフォルトだと Docker Hub から Pull してきますが、URL を記述することで他のレジストリからも取得できます。`env`には MySQL のイメージ起動に必要な root パスワードを設定しています。

`network`の`port`には今回は MySQL なので Static に`3306`として設定しています。また、これを`task.config.port_map`で指定した`3306`にポートフォワードしローカルから疎通できるようにしています。

それでは MySQL を動かしてみましょう。

```shell
$ nomad job run -hcl1 mysql.nomad
```

しばらくすると Docker プロセスが立ち上がります。

```console
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                                   NAMES
9072cdae373f        mysql:5.7.28        "docker-entrypoint.s…"   About an hour ago   Up About an hour    192.168.3.183:3306->3306/tcp, 192.168.3.183:3306->3306/udp, 33060/tcp   mysql-task-9752bf56-26bf-1f39-ae69-e53b654521c9
```

```console
$ nomad job status mysql-5.7
ID            = mysql-5.7
Name          = mysql-5.7
Submit Date   = 2020-02-19T05:30:16Z
Type          = service
Priority      = 50
Datacenters   = dc1
Status        = running
Periodic      = false
Parameterized = false

Summary
Task Group   Queued  Starting  Running  Failed  Complete  Lost
mysql-group  0       0         1        0       0         0

Latest Deployment
ID          = a85e7c45
Status      = successful
Description = Deployment completed successfully

Deployed
Task Group   Desired  Placed  Healthy  Unhealthy  Progress Deadline
mysql-group  1        1       1        0          2020-02-19T05:40:39Z

Allocations
ID        Node ID   Task Group   Version  Desired  Status   Created  Modified
fc0c5fcf  31528bd6  mysql-group  0        run      running  27s ago  4s ago
```

ログインしてみましょう。IP アドレスは環境によって異なります。`docker ps`の出力結果のものをコピーしてください。

```console
$ mysql -u root -p -h192.168.3.183
Enter password:rooooot
```

## Persistence Disk を扱う

次にデータを永続化するための Persistence Disk の設定を行います。まずそのままの状態で試しにデータを書き込んで再起動し、データが永続化されていないことを確認してみます。

データを投入して再起動してみましょう。

```
mysql> create database handson;
mysql> show databases;
```

この状態で Nomad Job を再起動します。現在は Nomad にストレージの設定を行っておらず Job は永続ディスクを持っていません。そのため、再起動するとデータは揮発するはずです。

```shell
$ nomad job stop mysql-5.7
$ nomad job run -hcl1 mysql.nomad
```

再度ログインして、データを参照してみます。

```shell
$ mysql -u root -p -h192.168.3.183
```

```
mysql> show databases;
```

先ほど作成した`handson`データベースは消えているでしょう。Docker を使いつつ、データベースなどディスクが要求される Stateful なワークロードを扱うには Nomad の`volume`機能を利用します。

Persistent Disk を使うためには

1. Client でホストのボリュームを扱う設定を行う
2. Job の定義でそれを利用するための設定を行う

の二つが必要です。

まずは Client 側の設定をします。

```shell
$ cd nomad-workshop
$ mkdir mysql-data
```

`nomad-local-config-client-1.hcl`,`nomad-local-config-client-2.hcl`,`nomad-local-config-client-3.hcl`の各ファイルの`client`の項目を下記のよう追記して下さい。他はそのままです。`<DIR>`はカレントディレクトリの絶対パスに置き換えて下さい。

```hcl
client {
  enabled = true
  servers = ["127.0.0.1:4647"]
  host_volume "mysql-vol" {
    path = "<DIR>/mysql-data"
    read_only = false
  }
}
```

3 つのファイルを書き換えたら再起動します。

```shell
$ ./run.sh
```

次に MySQL のジョブのファイルを下記のように書き換えます。

```shell
$ cd nomad-workshop
$ sudo mkdir /var/lib/mysql
$ cat << EOF > mysql.nomad
job "mysql-5.7" {
  datacenters = ["dc1"]

  type = "service"

  group "mysql-group" {
    count = 1

    volume "mysql-vol" {
      type      = "host"
      read_only = false
      source    = "mysql-vol"
    }


    task "mysql-task" {
      driver = "docker"

      volume_mount {
        volume      = "mysql-vol"
        destination = "/var/lib/mysql"
        read_only   = false
      }

      config {
        image = "mysql:5.7.28"
        port_map {
          db = 3306
        }
      }
      env {
        "MYSQL_ROOT_PASSWORD" = "rooooot"
      }
      resources {
        cpu    = 500
        memory = 256

        network {
          mbits = 10
          port "db" {
            static = 3306
          }
        }
      }
    }
  }
}
EOF
```

ここでは`host_volume`で作った Volume を Task Group にマッピングし、実際の Task にマウントしています。

これを使って MySQL を起動します。

```shell
$ nomad job run -hcl1 mysql.nomad
``` 

あとは同じようにデータを投入して再起動します。

```console
$ mysql -u root -p -h192.168.3.183
Enter password:rooooot
```

データを投入して再起動してみましょう。

```
mysql> create database handson;
mysql> show databases;
```

Nomad Job を再起動します。

```shell
$ nomad job stop mysql-5.7
$ nomad job run -hcl1 mysql.nomad
```

再度ログインして、データを参照してみます。

```shell
$ mysql -u root -p -h192.168.3.183
```

```
mysql> show databases;
```

`handson`のデータが残っていることがわかるはずです。また、ホストのディレクトリを見てみましょう。

```console
$ ls mysql-data
auto.cnf           client-key.pem     ib_logfile1        performance_schema server-key.pem
ca-key.pem         handson            ibdata1            private_key.pem    sys
ca.pem             ib_buffer_pool     ibtmp1             public_key.pem
client-cert.pem    ib_logfile0        mysql              server-cert.pem
```

MySQL のデータがホストに保存されていることがわかるはずです。今回は`host_volume`を利用しましたが Nomad は`CSI Plugin`に対応しており、様々なタイプのストレージを扱うことが可能です。

ここでは Docker Driver の基本と Persistence Disk の設定を行いましたが、まだまだ様々な設定を行いことができます。

最後にジョブを停止しておきましょう。

```shell
$ nomad job stop mysql-5.7
```

### 参考リンク
* [Drivers](https://www.nomadproject.io/docs/drivers/index.html)
* [Docker Driver](https://www.nomadproject.io/docs/drivers/docker.html)
* [Volume Configuration](https://www.nomadproject.io/docs/job-specification/volume.html)
* [Volume Mount Configuration](https://www.nomadproject.io/docs/job-specification/volume_mount.html)
* [CSI Plugin](https://www.hashicorp.com/blog/hashicorp-nomad-container-storage-interface-csi-beta/)
* [CSI Plugin Configuration](https://www.nomadproject.io/docs/job-specification/csi_plugin/)