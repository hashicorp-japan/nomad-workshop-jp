# NomadとVaultの連携機能を試す

NomadにはHashiCorp Vaultと連携をしダイナミックシークレットの発行やアプリからの利用など高度なシークレット管理を行うこと可能です。

Vaultとの連携はNomadの設定ベースで行うことができます。Vaultを初めて触る方は、このハンズオンを実行する前に[Vault Workshop](https://github.com/hashicorp-japan/vault-workshop)を実施することをお勧めします。(資料を読んで「初めてのVault」と「AWS」で十分です)

まずはVaultをインストールして起動してみましょう。

## Vaultのインストール

[こちら](https://www.vaultproject.io/downloads/)のWebサイトからご自身のOSに合ったものをダウンロードしてください。

パスを通します。以下はmacOSの例ですが、OSにあった手順で vaultコマンドにパスを通します。

```shell
$ mv /path/to/vault /usr/local/bin
$ chmod +x /usr/local/bin/vault
```
新しい端末を立ち上げ、Vaultのバージョンを確認します。

```console
$ vault --version
Vault v1.5.1
```

これでインストールは完了です。別端末でVaultを`dev`モードで立ち上げます。

```shell
$ vault agent -dev
```

起動ログに表示される`Root Token`の値をメモしてください。

## Nomadサーバの設定

NomadとVaultが連携をすることで以下のようなことを実現できます。

* NomadサーバがVaultのトークンを動的に発行し安全に格納
* 上記で格納したトークンを利用し、Vaultにシークレットの発行を依頼

これを利用することで`Secret Zero Problem`という「最初のトークンをどのように発行し、どのようにアプリから利用させるか」という問題を解決することができます。

まずNomadサーバにVaultのVaultのトークンを発行させるための設定を行いNomadサーバを再起動します。

```shell
$ cd /path/to/nomad-workshop
$ export DIR=$(pwd)

$ cat << EOF > nomad-vault-config-server.hcl
data_dir  = "${DIR}/datadir/local-vault-single-data"

bind_addr = "127.0.0.1"

vault {
  enabled = true
  address = "http://127.0.0.1:8200"
}

server {
  enabled = true
  bootstrap_expect = 1
}

advertise {
  http = "127.0.0.1"
  rpc  = "127.0.0.1"
  serf = "127.0.0.1"
}
EOF

$ cat << EOF > nomad-vault-config-client.hcl
data_dir  = "${DIR}/datadir/datadir/local-vault-single-data"

bind_addr = "127.0.0.1"

vault {
  enabled = true
  address = "http://127.0.0.1:8200"
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

plugin "raw_exec" {
  config {
    enabled = true
  }
}
EOF

$ cat << EOF > run-vault-local.sh
#!/bin/sh
pkill nomad
sleep 10

nomad agent -config=${DIR}/nomad-vault-config-server.hcl & 
nomad agent -config=${DIR}/nomad-vault-config-client.hcl &
EOF

$ chmod +x run-vault-local.sh
```

`NomadサーバがVaultのトークンを動的に発行`と記述しましたが、これはNomadサーバがVault APIを実行し、Vaultのトークンを発行するという挙動になります。そのため、`NomadサーバがVault APIを実行`するための大元のトークンが必要です。

環境変数にその大元のトークンをセットし起動することで以降、そのトークンを利用して実際にアプリが利用するトークンを生成していきます。`<ROOT_TOKEN>`を先ほどメモした`Root Token`に置き換えてください。(通常はRoot Tokenは利用しません)

```shell
$ export VAULT_TOKEN=<ROOT_TOKEN>

$ ./run-vault-local.sh
```

これでVaultとNomadサーバの連携の設定が完了し、Vaultトークンを動的に発行できるようになりました。

## Vault側の設定

次にVault側の設定を行います。今回の例ではNomadサーバが発行したトークンを利用して、Vault経由でアプリからAWSのキーを発行します。そのためVaultにAWSのAPIを実行させるための設定を行います。

```shell
$ vault secrets enable aws
$ vault write aws/config/root \
    access_key=************ \
    secret_key=************ \
    region=ap-northeast-1
```

次に`実際にアプリが利用するトークン`に与える権限の設定です。Vaultのポリシーの設定を作成します。

```shell
$ cat << EOF > aws.hcl
path "aws/*" {
  capabilities = [ "read" ]
}
EOF

$ vault policy write aws /Users/kabu/hashicorp/vault/configs/policies/aws.hcl
```

作ったポリシーを確認します。

```console
$ vault policy list
aws
default
root
```

このポリシーに基づいたトークンをNomadサーバに発行させ、その発行されたトークンを利用してアプリからAWSのキーを発行します。

次にVaultが発行するAWSのキーに与える権限をロールとして設定します。この例は`s3`への権限のあるロールです。

```shell
$ vault write aws/roles/s3-role \
    credential_type=iam_user \
    policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::*"
    }
  ]
}
EOF
```

このロールを利用してアプリがAWSのキーをVault経由で発行するとアプリはS3への操作を行う権限のあるキーを受け取るというイメージです。

## Nomadジョブの起動

それでは最後にNomadのジョブを作っていきます。

```shell
$ cat << EOF > hello-vault.nomad
job "hello-vault" {
  datacenters = ["dc1"]

  type = "service"

  group "hello-vault" {
    task "hello-vault" {
      env {
        VAULT_ADDR = "http://127.0.0.1:8200"
      }
      driver = "raw_exec"
      vault {
        policies = ["aws"]
        change_mode = "restart"
      }
      config {
        command = "/usr/local/bin/vault"
        args    = ["read", "aws/creds/s3-role"]
      }
    }
  }
}
EOF
```

`vault`スタンザ内でアプリが利用するVaultトークンのポリシーを設定しています。`change_mode`はVaultトークンを再作成するタイミングの設定でこの場合は`restart`と再起動時として指定しています。

`config`内ではアプリの実行コマンドでVaultにAWSのキーを発行するようにしています。

```console
$ nomad job run hello-vault.nomad

==> Monitoring evaluation "1246a789"
    Evaluation triggered by job "hello-vault"
    Allocation "6c82c783" created: node "435fd68a", group "hello-vault"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "1246a789" finished with status "complete"
```

Allicationの値をメモしてください。以下の`<Allocation>`を置き換えます。

```console
$ export ALLOC_ID=$(nomad alloc status -json <Allocation> | jq -r '.ID')
$ cat datadir/local-vault-single-data/alloc/${ALLOC_ID}/hello-vault/secrets/vault_token
s.SJt5TIsM4tpbVSgxgBHT5pgK
```

トークンの確認をしてみます。

```console
$ vault token lookup s.SJt5TIsM4tpbVSgxgBHT5pgK

Key                  Value
---                  -----
accessor             BGXyLqipkISuZxuCK1Th8PGi
creation_time        1581687393
creation_ttl         72h
display_name         token-cd6ce353-05b8-0e84-416a-cde8eb01a25e-hello-vault
entity_id            n/a
expire_time          2020-02-17T22:36:33.331253+09:00
explicit_max_ttl     0s
id                   s.SJt5TIsM4tpbVSgxgBHT5pgK
issue_time           2020-02-14T22:36:33.313292+09:00
last_renewal         2020-02-14T22:36:33.331253+09:00
last_renewal_time    1581687393
meta                 map[AllocationID:cd6ce353-05b8-0e84-416a-cde8eb01a25e NodeID:435fd68a-7712-1c43-5b03-bc86ff2e4cd2 Task:hello-vault]
num_uses             0
orphan               false
path                 auth/token/create
period               72h
policies             [aws default]
renewable            true
ttl                  71h59m24s
type                 service
```

指定したポリシーのトークンであることがわかるでしょう。アプリからこのトークンを透過的に利用してVaultのAPIを実行します。次にアプリの実行コマンドの結果を見てみましょう。アプリがトークンを使ってAWSのキーが発行したため、キーが一つ生成されているはずです。

```console
$ nomad fs ${ALLOC_ID} /alloc/logs/hello-vault.stdout.0
Key                Value
---                -----
lease_id           aws/creds/s3-role/BFQnLnRUU3KQAAgYrutelON0
lease_duration     48h
lease_renewable    true
access_key         ***************
secret_key         ***************
security_token     <nil>
```

Nomad上のアプリからVaultを利用してAWSのキーを動的に発行できることがわかりました。興味のある方はこのキーを利用してAWSのキーの権限を試してみてください。

最後にジョブをストップしてみます。

```console
$ nomad job stop hello-vault
$ vault token lookup s.SJt5TIsM4tpbVSgxgBHT5pgK
Error looking up token: Error making API request.

URL: POST http://127.0.0.1:8200/v1/auth/token/lookup
Code: 403. Errors:

* bad token
```

`change_mode`で`restart`と指定しているためジョブが停止するとトークンがRevokeされ利用不可能になっています。再度起動すると新しいトークンが都度発行されるためとても安全にVaultを利用することができます。

今回はシェルスクリプトを実行するダメの簡単なアプリでしたがWebのアプリでデータベースやクラウドのシークレットを発行するような際も同じように扱うことができます。`Secret Zero Problem`を解消しVaultの強力なダイナミックシークレットを安全に、かつ簡単に扱うことができます。


## 参考リンク
* [Vault Integration](https://www.nomadproject.io/docs/vault-integration/index.html)
* [Vault Job Configuration](https://www.nomadproject.io/docs/job-specification/vault.html)
* [Vault Agent Configuration](https://nomadproject.io/docs/configuration/vault/)