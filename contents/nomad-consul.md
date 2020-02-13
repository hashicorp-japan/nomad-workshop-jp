# NomadとConsulの連携機能を試す

NomadにはHashiCorp Consulと連携をしService Discovery, Load Brancing, Health Checking, Sidecarなどより高度なサービス管理を行うことがきます。

Consulとの連携はNomadの設定ベースで行うことができます。Consulを初めて触る方は、このハンズオンを実行する前に[Consul Workshop](https://github.com/hashicorp-japan/consul-workshop)を実施することをお勧めします。

まずはConsulをインストールして起動してみましょう。

## Consulのインストール

[こちら](https://www.consul.io/downloads.html)のWebサイトからご自身のOSに合ったものをダウンロードしてください。

パスを通します。以下はmacOSの例ですが、OSにあった手順で consulコマンドにパスを通します。

```shell
$ mv /path/to/consul /usr/local/bin
$ chmod +x /usr/local/bin/consul
```
新しい端末を立ち上げ、Consulのバージョンを確認します。

```console
$ consul --version
Consul v1.5.1
```

これでインストールは完了です。別端末でConsulを`dev`モードで立ち上げます。

```shell
$ consul agent -dev
```

## Nomadサーバの起動

NomadサーバにConsulとの連携機能を入れて再起動します。まずはNomadエージェントがConsulサーバと通信し、Consulの管理対象に参加させることが必要です。

```shell
$ cd /path/to/nomad-workshop
$ export DIR=$(pwd)
$ cat << EOF > nomad-consul-config-server.hcl 
data_dir  = "${DIR}/local-consul-single-data"

bind_addr = "127.0.0.1"

consul {
  address = "127.0.0.1:8500"
}

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

$ cat << EOF > nomad-consul-config-client.hcl
data_dir  = "${DIR}/local-consul-single-data"

bind_addr = "127.0.0.1"

consul {
  address = "127.0.0.1:8500"
}

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

$ cat << EOF > run-nomad-consul.sh
#!/bin/sh
pkill nomad
pkill java 
sleep 10

nomad agent -config=${DIR}/nomad-consul-config-server.hcl & 
nomad agent -config=${DIR}/nomad-consul-config-client.hcl &
EOF

$ chmod +x run-nomad-consul.sh
```

ここでは`consul`スタンザを利用してNomadエージェントがConsulのサービスとして登録されるための設定を行いました。TLS通信やConsulでACLの設定をしている際には追加で設定が必要ですが今回は`dev`モードで起動しているため`address`のみ指定していればOKです。

それではNomadを起動します。

```shell
$ ./run-nomad-consul.sh
```

Nomadサーバが起動したら`http://127.0.0.1:8500/ui/dc1/services`にブラウザでアクセスしてください。

`nomad`, `nomad-client`の二つがServicesが登録されているでしょう。`Health Checks`が失敗している際は少し待ってからブラウザを更新してみてください。

これでConsul, Nomadのサーバ側の準備は完了です。次以降でConsulの機能を実際のジョブから利用してみます。

## Service DiscoverytとHealth Checking

Service DiscoveryはConsulの一番基本的な機能です。通常IPアドレスやホスト名など物理的なロケーションでサービス同士を接続させますが、Consulを利用することでConsulがDNSのように振る舞い、サービス名で接続先を指定することが可能です。

Consulを利用するためのNomadのジョブ定義ファイルを作っていきます。

```shell
$ cat << EOF > hello-consul.nomad
job "hello-consul" {
  datacenters = ["dc1"]

  type = "service"

  group "web-server" {
    count = 1
    task "nginx" {

      service {
        name = "nginx"
        tags = ["nginx", "nomad"]
        meta {
          meta = "for your service"
        }
        check {
          type     = "http"
          protocol = "http"
          port     = "nginx"
          interval = "10s"
          timeout  = "2s"
          path = "/index.html"
        }
      }

      driver = "docker"

      config {
        image = "nginx"
      }

      resources {
        cpu    = 500
        memory = 256

        network {
          mbits = 10
          port "nginx" {
            static = "80"
          }
        }
      }
    }
  }
}
EOF
```

ここでは主に二つの設定を行なっています。

* `service`: Serviceに関する基本情報を登録します。`name`で指定したサービス名で`<NAME>.service.consul`のURLでLook-upするとこのサービス名をキーにConsulが実際のロケーションを返します。
* `check`: `service`内に定義する設定で、ヘルスチェックに関する設定です。プロトコルやヘルスチェックをするエンドポイント、間隔などを指定します。


このジョブを起動してみましょう。

```shell
nomad job run hello-consul.nomad
```

コンテナが一つ起動しているはずです。

```console
docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                              NAMES
7fa8d93d93d1        nginx               "nginx -g 'daemon of…"   3 minutes ago       Up 3 minutes        192.168.3.38:80->80/tcp, 192.168.3.38:80->80/udp   nginx-4ceeb4d7-04f0-02f8-5b5b-0473f003e029
```

Consul側のサービスカタログを確認すると`nginx`というサービスがConsulに登録されているはずです。ここで表示される`CONTAINER ID`はメモしておいて下さい。

`http://localhost:8500/ui/dc1/services`

Nomad AgentとConsul Agentが連携をし、Consul側には設定を行うことなくサービスカタログが更新されます。ConsulのDNSインタフェースを利用してこのサービスをLook-upしてみます。ConsulのDNS用のポートはデフォルトで`8600`です。

```console
dig @127.0.0.1 -p 8600 nginx.service.consul. SRV

; <<>> DiG 9.10.6 <<>> @127.0.0.1 -p 8600 hello-consul-web-server-nginx.service.consul. SRV
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16210
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 3
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;hello-consul-web-server-nginx.service.consul. IN SRV

;; ANSWER SECTION:
hello-consul-web-server-nginx.service.consul. 0	IN SRV 1 1 0 c0a80326.addr.dc1.consul.

;; ADDITIONAL SECTION:
c0a80326.addr.dc1.consul. 0	IN	A	192.168.3.38
Takayukis-MBP.node.dc1.consul. 0 IN	TXT	"consul-network-segment="

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Thu Feb 06 23:46:43 JST 2020
;; MSG SIZE  rcvd: 188
```

Consulのサービスカタログから実際のIPアドレスが返されます。実際のアプリケーションからもこのように問い合わせることで物理ロケーションを意識することなくアクセス可能です。

次にヘルスチェックを確認してみます。Consulのエンドポイントにアクセスして死活状況を見てみます。

```console
$ curl http://127.0.0.1:8500/v1/health/checks/hello-consul-web-server-nginx | jq '.[].Status, .[].Output'

"passing"
"HTTP GET http://192.168.3.38:80/index.html: 200 OK Output: <!DOCTYPE html>\n<html>\n<head>\n<title>Welcome to nginx!</title>\n<style>\n    body {\n        width: 35em;\n        margin: 0 auto;\n        font-family: Tahoma, Verdana, Arial, sans-serif;\n    }\n</style>\n</head>\n<body>\n<h1>Welcome to nginx!</h1>\n<p>If you see this page, the nginx web server is successfully installed and\nworking. Further configuration is required.</p>\n\n<p>For online documentation and support please refer to\n<a href=\"http://nginx.org/\">nginx.org</a>.<br/>\nCommercial support is available at\n<a href=\"http://nginx.com/\">nginx.com</a>.</p>\n\n<p><em>Thank you for using nginx.</em></p>\n</body>\n</html>\n"
```

指定した`index.html`にアクセスを実行し、`200`のレスポンスコードとHTMLが正しく返ってきています。Consulは`passing`とし正常なサービスとしてみなされています。

エラーを返すようにnginxの設定を編集していきます。`<CONTAINER_ID>`は先ほどメモした内容に置き換えて下さい。

```shell
$ docker exec -it <CONTAINER_ID> bin/bash
```

`index.html`を削除してエラーを返すように編集していきます。

```shell
$ rm /usr/share/nginx/html/index.html
```

これで`index.html`へのアクセスはエラーになるはずです。もう一度Consulのエンドポイントにアクセスして死活状況を見てみます。

```console
$ http://127.0.0.1:8500/v1/health/checks/hello-consul-web-server-nginx | jq '.[].Status, .[].Output'

"critical"
"HTTP GET http://192.168.3.38:80/index.html: 404 Not Found Output: <html>\r\n<head><title>404 Not Found</title></head>\r\n<body>\r\n<center><h1>404 Not Found</h1></center>\r\n<hr><center>nginx/1.17.8</center>\r\n</body>\r\n</html>\r\n"
```

エラーが返り、Consulは`critical`として正異常なサービスとしてみなされています。この状態で再度ConsulのDNSにサービス名で問い合わせをしてみましょう。

```console
$ dig @127.0.0.1 -p 8600 hello-consul-web-server-nginx.service.consul. SRV

; <<>> DiG 9.10.6 <<>> @127.0.0.1 -p 8600 hello-consul-web-server-nginx.service.consul. SRV
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 6124
;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;hello-consul-web-server-nginx.service.consul. IN SRV

;; AUTHORITY SECTION:
consul.     0 IN  SOA ns.consul. hostmaster.consul. 1581042348 3600 600 86400 0

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Fri Feb 07 11:25:48 JST 2020
;; MSG SIZE  rcvd: 123

```

このようにエラーとなったインスタンスは返されず自動的にサービスカタログから除外されます。

一旦Nomadを停止しておきましょう。

```shell
$ pkill nomad
```

## Sidecarを利用するアプリのデプロイ

**この手順はLinux上のみで実行可能です。**

次にConsul ConnectのSidecar Proxyを利用したアプリをどのようにNomadから定義して実行するかを試してみます。Consul Connectを利用するためにはCNI Pluginを有効化する必要があります。

[こちら](https://nomadproject.io/guides/integrations/consul-connect/#cni-plugins)の手順でインストールをして下さい。

インストールが終了したら`-dev-connect`のモードでNomadを起動します。

```shell
$ sudo nomad agent -dev-connect
```

次にJobを作っていきます。デプロイするアプリのイメージは以下の通りです。

<kbd>
  <img src="https://github-image-tkaburagi.s3.ap-northeast-1.amazonaws.com/consul-workshop/intentions-1.png">
</kbd>

`japanapp`,`corpapp`,`hashiapp`,`hashicorpjapanapp`の四つのアプリが定義し、それぞれのコンテナにSidecar Proxyが同居しています。

トラフィックの流れとしては図の通りで`japanapp`が`Japan`の文字列を返し、`corpapp`が`japanapp`の結果に`Corp`の文字列をAppendして`Corp Japan`を返します。そして`hashiapp`が`Hashi`を返し、`hashicorpjapanapp`が`hashiapp`と`corpapp`にリクエストして結果の文字列を連結させ`HashiCorp Japan`をクライアントに返しています。

```shell
$ cat << EOF > hcj.nomad
job "hcj" {
  datacenters = ["dc1"]

  type = "service"

  group "japan" {
    count = 1 

    network {
      mode = "bridge"
    }

    service {
      name = "japan-app"
      port = "8080"
      tags = ["japanapp", "nomad"]
      connect {
        sidecar_service {
        }
      }
    }

    task "japanapp" {
      driver = "docker"
      config {
        image = "tkaburagi/japanapp:v1"
      }
    }
  }

  group "corp" {
    network {
      mode = "bridge"
    }

    service {
      name = "corp-app"
      port = "8080"
      tags = ["corpapp", "nomad"]
      connect {
        sidecar_service {
          proxy {
            upstreams {
              destination_name = "japan-app"
              local_bind_port  = 5000
            }
          }
        }
      }
    }

    task "corpapp" {
      driver = "docker"
      config {
        image = "tkaburagi/corpapp:v1"
      }
    }
  }

  group "hashi" {
    count = 1 

    network {
      mode = "bridge"
    }

    service {
      name = "hashi-app"
      port = "8080"
      tags = ["hashiapp", "nomad"]
      connect {
        sidecar_service {
        }
      }
    }

    task "hashiapp" {
      driver = "docker"
      config {
        image = "tkaburagi/hashiapp:v1"
      }
    }
  }

  group "hashicorpjapan" {
    network {
      mode = "bridge"
      port "http" {
        static = 8080
        to     = 8080
      }
    }

    service {
      name = "hashicorpjapan-app"
      port = "8080"
      tags = ["hashicorpjapanapp", "nomad"]
      connect {
        sidecar_service {
          proxy {
            upstreams {
              destination_name = "corp-app"
              local_bind_port  = 5001
            }
            upstreams {
              destination_name = "hashi-app"
              local_bind_port  = 5000
            }
          }
        }
      }
    }

    task "hashicorpjapan" {
      driver = "docker"
      config {
        image = "tkaburagi/hashicorpjapanapp:v1"
      }
    }
  }
}
EOF
```

`service`内の`sidecar_service`がサイドカーの設定です。サイドカーのポート番号などを指定する事も出来ます。`proxy`にはサイドカープロキシの設定として`upstream`をしています。

`destination_name`にはUpstream先のサービス名、`local_bind_port`ローカルでリスンするポートを指定しています。アプリの図を見ながら`destination_name`を確認するとイメージが出来るでしょう。

それではこのアプリをデプロイしてみましょう。

```shell
$ nomad job run hcj.nomad
```

Consulのブラウザにアクセスしてサービスを確認してみましょう。このようになっているはずです。

<kbd>
  <img src="https://github-image-tkaburagi.s3-ap-northeast-1.amazonaws.com/nomad-workshop/Screen+Shot+2020-02-13+at+18.16.40.png">
</kbd>

`hashicorpjapanapp`にアクセスするとアプリが正常に動いているはずです。

```console
$ curl 127.0.0.1:8080
HashiCorp Japan
```

サイドカープロキシを利用する際もConsulに対して設定を行うことなく、Nomadのジョブ設定でConsulと連携しJobを定義することが可能ということがわかります。Consulにはその他にもConfiguration Managementなどの機能があり、他の機能もNomadから利用することができます。

## 参考リンク
* [Consul Conguration](https://www.nomadproject.io/docs/configuration/consul.html)
* [Service Stanza](https://www.nomadproject.io/docs/job-specification/service.html)
* [Sidecar Service Stanza](https://www.nomadproject.io/docs/job-specification/sidecar_service.html)
* [Sidecar Task Stanza](https://www.nomadproject.io/docs/job-specification/sidecar_task.html)
* [Upstream Stanza](https://www.nomadproject.io/docs/job-specification/upstreams.html)
* [Connect Stanza](https://www.nomadproject.io/docs/job-specification/connect.html)
* [Check Restart Stanza](https://www.nomadproject.io/docs/job-specification/check_restart.html)
* [Update Stanza](https://www.nomadproject.io/docs/job-specification/service.html#check-parameters)