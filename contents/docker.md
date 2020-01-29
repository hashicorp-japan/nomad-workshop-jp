## NomadでDockerイメージを動かす

NomadではDockerのようなコンテナのワークロードにとどまらず、スタンドアロンの`Java`, `RaW Exec`や`Qemu`など様々なタイプのTaskを実行できます。

各Taskはクライアント上の`Task Driver`によってリソースがIsolationされ実行されます。Task Driverはプラガブルで、各ドライバーの定義はJob定義のTaskn内の`plugin stanza`で設定します。

ここではいくつかのDockerイメージをNomad上で動かしてみます。また、この章では永続ディスクも試してみます。

## Docker Task Driverを扱う

Docker Task Driverはその名の通り、Docker Imageを実行させるためのDriverです。これを利用することでDocker Imageのpullや実行させるために必要なVolumeやネットワークの設定を宣言的に行うことが可能です。

まず一つ簡単なDocker ImageをNomad上で稼働させてみましょう。

以下のようにJobの定義ファイルを作成します。

```shell
$ cd nomad-workshop
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

`job`の`datacenters`にはデフォルトの`dc1`としています。`type`にはジョブのタイプをしていますが、今回はMySQLでLong Running Processなので`service`としています。

`group`がTask Groupで`count`はTaskの数です。今回はMySQL1インスタントとしますので`1`です。`task`の中が実際のアプリです。`driver`はDockerを指定していて、それ以降がDocker関連の`image`, `port_map`, `env`の設定です。

`image`はデフォルトだとDocker HubからPullしてきますが、URLを記述することで他のレジストリからも取得できます。`env`にはMySQLのイメージ起動に必要なrootパスワードを設定しています。

`network`の`port`には今回はMySQLなのでStaticに`3306`として設定しています。また、これを`task.config.port_map`で指定した`3306`にポートフォワードしローカルから疎通できるようにしています。

それではMySQLを動かしてみましょう。

```shell
$ nomad job run mysql.nomad
```

しばらくするとDockerプロセスが立ち上がります。

```console
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                                   NAMES
9072cdae373f        mysql:5.7.28        "docker-entrypoint.s…"   About an hour ago   Up About an hour    192.168.3.183:3306->3306/tcp, 192.168.3.183:3306->3306/udp, 33060/tcp   mysql-task-9752bf56-26bf-1f39-ae69-e53b654521c9
```

ログインしてみましょう。IPアドレスは環境によって異なります。上の出力結果のものをコピーしてください。

```console
$ mysql -u root -p -h192.168.3.183
Enter password:rooooot
```

## Persistence Diskを扱う

次にデータを永続化するためのPersistence Diskの設定を行います。まずそのままの状態で試しにデータを書き込んで再起動し、データが永続化されていないことを確認してみます。

データを投入して再起動してみましょう。

```
mysql> create database handson;
mysql> show databases;
```

この状態でNomad Jobを再起動します。現在はNomadにストレージの設定を行っておらずJobは永続ディスクを持っていません。そのため、再起動するとデータは揮発するはずです。

```shell
$ nomad job stop mysql-5.7
$ nomad job run mysql.nomad
```

再度ログインして、データを参照してみます。

```shell
$ mysql -u root -p -h192.168.3.183
```

```
mysql> show databases;
```

先ほど作成した`handson`データベースは消えているでしょう。Dockerを使いつつ、データベースなどディスクが要求されるStatefulなワークロードを扱うにはNomadの`volume`機能を利用します。

Persistent Diskを使うためには

1. Clientでホストのボリュームを扱う設定を行う
2. Jobの定義でそれを利用するための設定を行う

の二つが必要です。

まずはClient側の設定をします。

```shell
$ mkdir /tmp/mysql
$ cd nomad-workshop
```

`nomad-local-config-client-1.hcl`,`nomad-local-config-client-2.hcl`,`nomad-local-config-client-3.hcl`の各ファイルの`client`の項目を下記のように上書きして下さい。他はそのままです。

```hcl
client {
  enabled = true
  servers = ["127.0.0.1:4647"]
  host_volume "mysql-vol" {
    path = "/tmp/mysql"
    read_only = false
  }
}
```

3つのファイルを書き換えたら再起動します。

```shell
$ ./run.sh
```

次にMySQLのジョブのファイルを下記のように書き換えます。

```shell
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
```

ここでは`host_volume`で作ったVolumeをTask Groupにマッピングし、実際のTaskにマウントしています。

ゔぉぅこれを使ってMySQLを起動します。

```shell
$ nomad job run mysql.nomad
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

Nomad Jobを再起動します。

```shell
$ nomad job stop mysql.nomad
$ nomad job run mysql.nomad
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
$ ls /tmp/mysql
auto.cnf           client-key.pem     ib_logfile1        performance_schema server-key.pem
ca-key.pem         handson            ibdata1            private_key.pem    sys
ca.pem             ib_buffer_pool     ibtmp1             public_key.pem
client-cert.pem    ib_logfile0        mysql              server-cert.pem
```

MySQLのデータがホストに保存されていることがわかるはずです。またVolumeのタイプは現時点では`host_volume`のみの対応となっており、今後その他ストレージもサポートしていく予定になっています。

ここではDocker Driverの基本とPersistence Diskの設定を行いましたが、まだまだ様々な設定を行いことができます。

### 参考リンク
* [Drivers](https://www.nomadproject.io/docs/drivers/index.html)
* [Docker Driver](https://www.nomadproject.io/docs/drivers/docker.html)
* [Volume Configuration](https://www.nomadproject.io/docs/job-specification/volume.html)
* [Volume Mount Configuration](https://www.nomadproject.io/docs/job-specification/volume_mount.html)