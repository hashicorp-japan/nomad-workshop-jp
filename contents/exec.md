## Exec Driverで様々なタスクを動かす

NomadではNomadが稼働しているOS上で動作する様々なタイプのアプリケーションを実行することができます。このタイプのジョブには`Raw Exec`と`Isolated Exec`という二つのタイプのドライバーが存在します。

出来ることは同じですが、以下のような違いがあります。

* `Raw Exec`: リソースIsolationの機能がない。
* `Isolated Exec`:　リソースIsolationの機能がある。

`Isolated Exec`はRoot権限やcgroupファイルシステムがマウントされている必要があるため、今回は簡易的な`Raw Exec`ドライバーを使います。

## Exec Driverを扱う

Execドライバーを扱うためにはまずクライアント(Jobを実行するサーバ)側の機能をEnableにする必要があります。ここでは一つのクライアントで有効化します。

`nomad-local-config-client-1.hcl`ファイルに下記の行を追加してください。

```
plugin "raw_exec" {
  config {
    enabled = true
  }
}
```

次にクラスタを再起動させます。

```shell
$ cd nomad-workshop
$ ./run.sh
```

これにより`raw_exec`が有効化されます。これは`Evaluation`のフェーズで判断され有効化されているクライアントに`Allocation`されます。そのため`Raw Exec`タイプのアプリケーションはこれを設定したクライアント上でのみ実行されることになります。

次にジョブのファイルを作っていきます。

```shell
$ cd nomad-workshop
$ cat << EOF > exec.nomad
job "hello-exec-batch" {
  datacenters = ["dc1"]

  type = "batch"

  group "hello-exec" {
    task "echo" {
      driver = "raw_exec"
      config {
        # When running a binary that exists on the host, the path must be absolute.
        command = "/bin/bash"
        args    = ["-c", "echo Hi Nomad Exec Driver && sleep 180" ]
      }
    }
  }
}
EOF
```

`driver`に`raw_exec`を指定し、`config`に実行するコマンドとその引数を指定することが出来ます。

あとはこれは`nomad`コマンドで実行するだけです。

```console
$ nomad job run -hcl1 exec.nomad
```

さて、実行結果は標準出力に吐かれているため、ログの中を確認して見ましょう。上で出力される`Allocation ID`をコピーしてください。

```shell
$ nomad job status -verbose hello-exec-batch
$ export ALLOC=<ALLOCATIO_ID>
```

取得した`Application ID`上のログを見ると`echo`で出力したテキストがログとして残っているでしょう。

```console
$ nomad fs ${ALLOC} alloc/logs/echo.stdout.0
Hi Nomad Exec Driver
```

以上のようにとても簡単にOSの機能を使ってスクリプトを実行することが出来ます。

## 様々なタイプのアプリを稼働させる

さて、今までは簡単なシェルスクリプトを実行させてきました。ExecドライバーではNomadクライアントのOSが対応していれば様々なタイプのワークロードを稼働させることが可能です。

ここでは`Goland`と`PHP`の二つのシンプルなWebアプリケーションを動かしてみます。

[Go](https://golang.org/dl/)と[PHP](https://www.php.net/manual/en/install.php)がインストールされていることを確認してください。

まずはGoで試してみます。以下のファイルを作ってください。

```shell
$ cd /path/to/nomad-workshop
$ cat <<EOF > main.go
package main

import (
        "fmt"
        "net/http"
)

func main() {
        http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
                fmt.Fprint(w, "Hello World! Go is running on Nomad")
        })
        http.ListenAndServe(":"+"8080", nil)
}
EOF
```

次にこのアプリを稼働させるためのNomadの定義ファイルを作っていきます。

```shell
$ export DIR=$(pwd)
$ export PATH_TO_GO=$(which go)
$ cat <<EOF > hello-exec-go.nomad
job "hello-exec-go-web" {
  datacenters = ["dc1"]

  type = "service"

  group "go-web" {
    count = 1
    task "go-web" {
      driver = "raw_exec"
      config {
        command = "${PATH_TO_GO}"
        args    = ["run", "${DIR}/main.go"]
      }
    }
  }
}
EOF
```

シェルスクリプト同様、`command`と`args`を利用してアプリケーションの実行コマンドを記述するだけです。それではこれをNomadで動かしてみます。

```shell
$ nomad job run -hcl1 hello-exec-go.nomad
```

アクセスしてみましょう。

```console
$ curl localhost:8080
Hello World! Go is running on Nomad
```

とても簡単です。興味のある方は`http://127.0.0.1:4646/ui/jobs/hello-exec-go-web`にアクセスしてJobの様子も確認して見ましょう。

次はPHPのアプリです。同様にアプリのファイルを作ってそれを稼働させるためのNomadファイルを作ります。Goと同様、`/usr/bin/php`は環境によって異なりますので`which php`で確認して置き換えてください。

```shell
$ cat <<EOF > hello.php
<?php 
Print "Hello World! PHP is running on Nomad";
?>
EOF
```

```shell
$ export PATH_TO_PHP=$(which php)
$ cat <<EOF > hello-exec-php.nomad
job "hello-exec-php-web" {
  datacenters = ["dc1"]

  type = "service"

  group "php-web" {
    count = 1
    task "php-web" {
      driver = "raw_exec"
      config {
        command = "${PATH_TO_PHP}"
        args    = ["-S", "localhost:8888", "${DIR}/hello.php"]
      }
    }
  }
}
EOF
```

同様に稼働させ、アプリにリクエストしてみます。

```console
$ nomad job run -hcl1 hello-exec-php.nomad
$ curl localhost:8888
Hello World! PHP is running on Nomad
```

今まで同様にアプリケーションがNomad上で稼働していることがわかります。以上のようにNomadではDockerコンテナ化されたアプリケーションやJavaアプリケーションを専用のドライバーで稼働させることのみならず、コンテナ化されてないようなアプリケーションも簡単に稼働させられることがわかりました。

Nomadはコンテナのエコシステムを最大限利用できる一方、簡単なスクリプト、バッチ処理やあらゆるWebのワークロードも実行させることが出来ます。

最後にジョブを停止しておきましょう。

```shell
$ nomad job stop hello-exec-go-web
$ nomad job stop hello-exec-php-web
```

## SpreadとAffinityとConstraintでJobのデプロイを制御する

最後に`affinity`と`spread`と`constraint`という機能を使ってJobのデプロイメント先の制御を行ってみます。この機能は今まで利用した`Volume`, `Logs`や`Artifact`と同様、Execドライバー以外でも利用することが出来ます。

### Spread

`Spread`はJob定義の中の`task group`内に定義します。Spreadを利用することでデータセンター、ラックやノードに跨ったデプロイが可能となり、それぞれにどのくらいの割合でデプロイするかなどを設定ベースで制御することが可能となります。

ここでは`初めてのNomad`の章で起動させた3ノードクラスタに対してJobを分散させてみます。ここではポートの重複等を気にしなくていいように`exec.nomad`で定義したシェルスクリプトのジョブを利用することにします。

まず、各クライアントの設定を書き換えます。

```shell
cd /path/to/nomad-workshop
```

以下の三つのコマンドを実行してください。

<details><summary>nomad-local-config-client-1.hcl</summary>
	
```shell
$ export DIR=$(pwd)
$ cat << EOF > nomad-local-config-client-1.hcl

data_dir  = "${DIR}/local-cluster-data-1"

bind_addr = "0.0.0.0"

client {
  enabled = true
  servers = ["127.0.0.1:4647"]
  meta {
    "Name" = "client-1"
  }
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

plugin "raw_exec" {
  config {
    enabled = true
  }
}

telemetry {
  publish_allocation_metrics = true
  publish_node_metrics       = true
  prometheus_metrics         = true
}
EOF
```
</details>

<details><summary>nomad-local-config-client-2.hcl</summary>
	
```shell
$ cat << EOF > nomad-local-config-client-2.hcl

data_dir  = "${DIR}/local-cluster-data-2"

bind_addr = "0.0.0.0"

client {
  enabled = true
  servers = ["127.0.0.1:4647"]
  meta {
    "Name" = "client-2"
  }
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

plugin "raw_exec" {
  config {
    enabled = true
  }
}

telemetry {
  publish_allocation_metrics = true
  publish_node_metrics       = true
  prometheus_metrics         = true
}
EOF
```
</details>

<details><summary>nomad-local-config-client-3.hcl</summary>
	
```shell
$ cat << EOF > nomad-local-config-client-3.hcl

data_dir  = "${DIR}/local-cluster-data-3"

bind_addr = "0.0.0.0"

client {
  enabled = true
  servers = ["127.0.0.1:4647"]
  meta {
    "Name" = "client-3"
  }
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

plugin "raw_exec" {
  config {
    enabled = true
  }
}

telemetry {
  publish_allocation_metrics = true
  publish_node_metrics       = true
  prometheus_metrics         = true
}
EOF
```
</details>

変更点は各クライアントで`raw-exec`を有効化している点と、`meta.Name`の値を加えている点です。`meta.Name`は各ジョブの`spread`スタンザ内でジョブの分散の基準として利用します。

再起動します。

```shell
$ ./run.sh
```

次にジョブの定義を書き換えましょう。

```shell
$ cat << EOF > exec-spread.nomad

job "hello-exec-batch" {
  datacenters = ["dc1"]

  type = "batch"

  group "example" {
    count = 10
    spread {
      attribute = "\${meta.Name}"
      target "client-1" {
        percent = 50
      }
      target "client-2" {
        percent = 0
      }
      target "client-3" {
        percent = 50
      }
    }
    task "echo" {
      driver = "raw_exec"
      config {
        # When running a binary that exists on the host, the path must be absolute.
        command = "/bin/echo"
        args    = ["Hi Nomad Exec Driver"]
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

`group`内の`spread`に`client-1`に`50%`,`client-2`に`0%`,`client-3`に`50%`の割合でデプロイすることを指定しています。

`attribute`が基準となる値で今回は`meta.Name`を利用しています。この他にも[こちら](https://www.nomadproject.io/docs/runtime/interpolation.html#interpreted_node_vars)に記載されているようなリージョン、データセンターなど様々な指定が可能です。

また`count = 10`とし、同じジョブを10個稼働させています。

これをデプロイしてみます。

```shell
$ nomad job run -hcl1 exec-spread.nomad
```

デプロイが完了したらどこのノードで実行されたかを確認してみましょう。

```console
$ nomad job status hello-exec-batch

~~~~~

Allocations
ID        Node ID   Task Group  Version  Desired  Status    Created     Modified
12f3a5ec  8f5984b6  example     2        run      complete  42m38s ago  42m36s ago
5aa47d79  4b635091  example     2        run      complete  42m38s ago  42m36s ago
439d5b6b  8f5984b6  example     2        run      complete  42m38s ago  42m36s ago
f94a16a8  4b635091  example     2        run      complete  42m38s ago  42m36s ago
3d9f0279  8f5984b6  example     2        run      complete  42m38s ago  42m36s ago
3544040a  4b635091  example     2        run      complete  42m38s ago  42m36s ago
257619f8  4b635091  example     2        run      complete  42m38s ago  42m36s ago
f263c806  4b635091  example     2        run      complete  42m38s ago  42m36s ago
1e7d2c93  8f5984b6  example     2        run      complete  42m38s ago  42m37s ago
fb919166  8f5984b6  example     2        run      complete  42m38s ago  42m36s ago
c7a2f6c2  4b635091  example     1        run      complete  2h55m ago   44m44s ago
```

`Version 2`の`Node ID`を見ると50%ずつ二つのノードに分散されていることがわかるでしょう。

次に`7:2:1`の割合に変更してみます。

```shell
$ cat << EOF > exec-spread.nomad
job "hello-exec-batch" {
  datacenters = ["dc1"]

  type = "batch"

  group "example" {
    count = 10
    spread {
      attribute = "\${meta.Name}"
      target "client-1" {
        percent =  70
      }
      target "client-2" {
        percent = 20
      }
      target "client-3" {
        percent = 10
      }
    }
    task "echo" {
      driver = "raw_exec"
      config {
        # When running a binary that exists on the host, the path must be absolute.
        command = "/bin/echo"
        args    = ["Hi Nomad Exec Driver"]
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

再度起動します。

```shell
$ nomad job run -hcl1 exec-spread.nomad
```

デプロイが完了したらどこのノードで実行されたかを確認してみましょう。

```console
$ nomad job status hello-exec-batch

~~~~~

Allocations
ID        Node ID   Task Group  Version  Desired  Status    Created     Modified
7de7f1d3  4b635091  example     3        run      complete  6s ago      3s ago
f401040d  6477d9ed  example     3        run      complete  6s ago      4s ago
e3988073  4b635091  example     3        run      complete  6s ago      3s ago
3056a9ef  4b635091  example     3        run      complete  6s ago      3s ago
e389dd78  8f5984b6  example     3        run      complete  6s ago      4s ago
3860661c  4b635091  example     3        run      complete  6s ago      3s ago
cde6fcad  4b635091  example     3        run      complete  6s ago      3s ago
4090dc39  4b635091  example     3        run      complete  6s ago      3s ago
a8b49583  4b635091  example     3        run      complete  6s ago      3s ago
a8a028d3  8f5984b6  example     3        run      complete  6s ago      5s ago
12f3a5ec  8f5984b6  example     2        run      complete  42m38s ago  42m36s ago
5aa47d79  4b635091  example     2        run      complete  42m38s ago  42m36s ago
439d5b6b  8f5984b6  example     2        run      complete  42m38s ago  42m36s ago
f94a16a8  4b635091  example     2        run      complete  42m38s ago  42m36s ago
3d9f0279  8f5984b6  example     2        run      complete  42m38s ago  42m36s ago
3544040a  4b635091  example     2        run      complete  42m38s ago  42m36s ago
257619f8  4b635091  example     2        run      complete  42m38s ago  42m36s ago
f263c806  4b635091  example     2        run      complete  42m38s ago  42m36s ago
1e7d2c93  8f5984b6  example     2        run      complete  42m38s ago  42m37s ago
fb919166  8f5984b6  example     2        run      complete  42m38s ago  42m36s ago
c7a2f6c2  4b635091  example     1        run      complete  2h55m ago   44m44s ago
```

`Version 3`の`Node ID`の結果を見ると設定した通りに分散しているでしょう。

このようにSpreadの機能を利用すると設定ベースで簡単にジョブを分散させ、その割合を設定することが出来ます。

### Affinity

次は`Affinity`です。Affinityは各ジョブを実行する場所を条件を指定して制御出来る機能です。同一環境内に複数のOSやカーネルのバージョンやGPUなどのマシンが混在している際、ランダムで分散させると起動しないようなワークロードを指定して稼働させたいような時に利用できます。

Affinityは条件にマッチしない場合稼働出来るものは稼働するという性質を持っているため、同一マシン上では機能を試すことは出来ません。ここではAffinityルールの例をいくつか紹介します。

1. カーネルバージョンでの指定

```hcl
affinity {
  attribute = "${attr.kernel.version}"
  operator  = "version"
  value     = "> 3.19"
  weight    = 50
}
```

2. OSでの指定(複数条件がある際は`weight`で優先度を決める)

```hcl
affinity {
  attribute = "${attr.os.name}"
  operator  = "="
  value     = "ubuntu"
  weight    = 50
}

affinity {
  attribute = "${attr.os.version}"
  operator  = ">="
  value     = "14.04"
  weight    = 100
}
```

3. AWSインスタンサイズでの指定

```hcl
affinity {
  attribute = "${attr.platform.aws.instance-type}"
  operator = "regexp"
  value     = "m4.xlarge"
  weight    = 50
}
```

以上のようなイメージです。単純に分散させるだけでなくアプリの特性ごとにデプロイ先を制御できます。`operator`パラメータを利用することで正規表現などでも基準を定義することが可能です。

### Constraint

最後に`Constraint`です。これはAffinityよりも厳しいルールで、条件に合わないノードへのデプロイを完全に制限するためのものです。

今回は同一マシンで動かしているため、OSの制限をかけて意図的に全てのジョブがデプロイされないことを確認してみます。それぞれ自分のOSと違う設定をするため以下のコマンドを実行してください。

* `macOS`の場合: `export MY_OS=windows`
* `Ubuntu`の場合: `export MY_OS=windows`
* `Cent OS`の場合: `export MY_OS=windows`
* `Windows`の場合: `export MY_OS=darwin`

Nomadのジョブ定義を作ります。

```shell
$ cat << EOF > exec-constraint.nomad
job "hello-exec-batch" {
  datacenters = ["dc1"]
  type = "batch"

  group "example" {
    count = 10
    constraint {
      attribute = "\${attr.os.name}"
      value     = "${MY_OS}"
    }
    task "echo" {
      driver = "raw_exec"
      config {
        # When running a binary that exists on the host, the path must be absolute.
        command = "/bin/echo"
        args    = ["Hi Nomad Exec Driver"]
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

この状態でジョブの軌道を試してみましょう。
```console
$ nomad job run -hcl1 exec-constraint.nomad
==> Monitoring evaluation "e183a9bb"
    Evaluation triggered by job "hello-exec-batch"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "e183a9bb" finished with status "complete" but failed to place all allocations:
    Task Group "example" (failed to place 10 allocations):
      * Constraint "${attr.os.name} = windows" filtered 3 nodes
    Evaluation "eb9b4c68" waiting for additional capacity to place remainder
```

Constraintに引っかかり`Evaluation`のフェーズでエラーになるはずです。次に正しいOSに直してみます。

* `macOS`の場合: `export MY_OS=darwin`
* `Ubuntu`の場合: `export MY_OS=ubuntu`
* `Cent OS`の場合: `export MY_OS=centos`
* `Windows`の場合: `export MY_OS=windows`

```shell
$ cat << EOF > exec-constraint.nomad
job "hello-exec-batch" {
  datacenters = ["dc1"]
  type = "batch"

  group "example" {
    count = 10
    constraint {
      attribute = "\${attr.os.name}"
      value     = "${MY_OS}"
    }
    task "echo" {
      driver = "raw_exec"
      config {
        # When running a binary that exists on the host, the path must be absolute.
        command = "/bin/echo"
        args    = ["Hi Nomad Exec Driver"]
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

起動してみます。

```console
$ nomad job run -hcl1 exec-constraint.nomad
==> Monitoring evaluation "b5edcaa7"
    Evaluation triggered by job "hello-exec-batch"
    Allocation "48154342" created: node "6477d9ed", group "example"
    Allocation "517f5bc8" created: node "8f5984b6", group "example"
    Allocation "836fa934" created: node "8f5984b6", group "example"
    Allocation "98a3fa08" created: node "8f5984b6", group "example"
    Allocation "dc3d5762" created: node "4b635091", group "example"
    Allocation "29c0eaee" created: node "4b635091", group "example"
    Allocation "3269fe54" created: node "4b635091", group "example"
    Allocation "e899a04c" created: node "8f5984b6", group "example"
    Allocation "2fcf96f6" created: node "4b635091", group "example"
    Allocation "59e8f4b4" created: node "8f5984b6", group "example"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "b5edcaa7" finished with status "complete"
```

ジョブが正しく稼働されるはずです。今回は同一マシンであったため全てのクライアントで起動するかしないかでしたが、クライアントが複数混在している際はConstraintにマッチするノードのみでジョブが実行されます。

このようにNomadでは混在の環境でもJobを安全に実行するためのスケージューリングをサポートするための機能も充実しています。

最後にジョブを停止しておきましょう。

```shell
$ nomad job stop hello-exec-batch
```

## 参考リンク
* [Raw Exec Driver](https://www.nomadproject.io/docs/drivers/raw_exec.html)
* [Exec Driver](https://www.nomadproject.io/docs/drivers/exec.html)
* [Spread Configuration](https://www.nomadproject.io/docs/job-specification/spread.html)
* [Affinity Configuration](https://www.nomadproject.io/docs/job-specification/affinity.html)
* [Interpreted Variables](https://www.nomadproject.io/docs/runtime/interpolation.html#interpreted_node_vars)