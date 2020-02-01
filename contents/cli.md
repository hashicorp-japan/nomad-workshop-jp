# Nomad CLIを色々と試す

ここではnomad cliの基本的な使い方のガイドを通してNomadの様々な機能を解説してみたいと思います。

Nomadが起動できていない方は`初めてのNomad`を参照してNomadクラスターを起動させて下さい。

## completion

Nomadではコマンド入力をサポートするための入力補足を用意しています。以下のコマンドでインストールしてみましょう。

```shell
$ nomad -autocomplete-install
```

これを利用するとタブでサブコマンドの入力を補完してくれます。

## node

まずはクライアントノードの情報を取得してみます。

```console
$ nomad node status
ID        DC   Name           Class   Drain  Eligibility  Status
6477d9ed  dc1  Takayukis-MBP  <none>  false  eligible     ready
4b635091  dc1  Takayukis-MBP  <none>  false  eligible     ready
8f5984b6  dc1  Takayukis-MBP  <none>  false  eligible     ready
```

クライアントノードのリストと状態が取得出来ました。これはどのコマンドにも共通ですが、`-verbose`のオプションをつけるとより詳細な結果を見ることが出来ます。

```console
$ nomad node status -verbose
ID                                    DC   Name           Class   Address    Version  Drain  Eligibility  Status
6477d9ed-8bfd-2258-91f9-cab5965ff787  dc1  Takayukis-MBP  <none>  127.0.0.1  0.10.0   false  eligible     ready
4b635091-f444-34d6-5b33-a8aedcc49664  dc1  Takayukis-MBP  <none>  127.0.0.1  0.10.0   false  eligible     ready
8f5984b6-faf6-a485-c3cf-2729e26fc775  dc1  Takayukis-MBP  <none>  127.0.0.1  0.10.0   false  eligible     ready
```

また、`-json`オプションをつけるとJSONで結果が返ってきます。出力結果から値を取得したいような際にはとても便利です。

```shell
$ nomad node status -json
```

<details><summary>出力結果例</summary>
	
```json
[
    {
        "Address": "127.0.0.1",
        "CreateIndex": 8,
        "Datacenter": "dc1",
        "Drain": false,
        "Drivers": {
            "java": {
                "Attributes": {
                    "driver.java": "true",
                    "driver.java.version": "12",
                    "driver.java.runtime": "OpenJDK Runtime Environment (build 12+33)",
                    "driver.java.vm": "OpenJDK 64-Bit Server VM (build 12+33, mixed mode, sharing)"
                },
                "Detected": true,
                "HealthDescription": "Healthy",
                "Healthy": true,
                "UpdateTime": "2020-01-30T13:12:15.609103+09:00"
            },
            "exec": {
                "Attributes": null,
                "Detected": false,
                "HealthDescription": "exec driver unsupported on client OS",
                "Healthy": false,
                "UpdateTime": "2020-01-30T13:12:15.518818+09:00"
            },
            "qemu": {
                "Attributes": null,
                "Detected": false,
                "HealthDescription": "",
                "Healthy": false,
                "UpdateTime": "2020-01-30T13:12:15.519712+09:00"
            },
            "raw_exec": {
                "Attributes": {
                    "driver.raw_exec": "true"
                },
                "Detected": true,
                "HealthDescription": "Healthy",
                "Healthy": true,
                "UpdateTime": "2020-01-30T13:12:15.520184+09:00"
            },
            "docker": {
                "Attributes": {
                    "driver.docker.volumes.enabled": "true",
                    "driver.docker.bridge_ip": "172.17.0.1",
                    "driver.docker.runtimes": "runc",
                    "driver.docker.os_type": "linux",
                    "driver.docker": "true",
                    "driver.docker.version": "19.03.5"
                },
                "Detected": true,
                "HealthDescription": "Healthy",
                "Healthy": true,
                "UpdateTime": "2020-01-30T13:12:15.554516+09:00"
            }
        },
        "ID": "6477d9ed-8bfd-2258-91f9-cab5965ff787",
        "ModifyIndex": 5688,
        "Name": "Takayukis-MBP",
        "NodeClass": "",
        "SchedulingEligibility": "eligible",
        "Status": "ready",
        "StatusDescription": "",
        "Version": "0.10.0"
    },
    {
        "Address": "127.0.0.1",
        "CreateIndex": 7,
        "Datacenter": "dc1",
        "Drain": false,
        "Drivers": {
            "java": {
                "Attributes": {
                    "driver.java": "true",
                    "driver.java.version": "12",
                    "driver.java.runtime": "OpenJDK Runtime Environment (build 12+33)",
                    "driver.java.vm": "OpenJDK 64-Bit Server VM (build 12+33, mixed mode, sharing)"
                },
                "Detected": true,
                "HealthDescription": "Healthy",
                "Healthy": true,
                "UpdateTime": "2020-01-30T13:12:16.643388+09:00"
            },
            "exec": {
                "Attributes": null,
                "Detected": false,
                "HealthDescription": "exec driver unsupported on client OS",
                "Healthy": false,
                "UpdateTime": "2020-01-30T13:12:16.561606+09:00"
            },
            "raw_exec": {
                "Attributes": {
                    "driver.raw_exec": "true"
                },
                "Detected": true,
                "HealthDescription": "Healthy",
                "Healthy": true,
                "UpdateTime": "2020-01-30T13:12:16.561737+09:00"
            },
            "qemu": {
                "Attributes": null,
                "Detected": false,
                "HealthDescription": "",
                "Healthy": false,
                "UpdateTime": "2020-01-30T13:12:16.561927+09:00"
            },
            "docker": {
                "Attributes": {
                    "driver.docker.runtimes": "runc",
                    "driver.docker.os_type": "linux",
                    "driver.docker": "true",
                    "driver.docker.version": "19.03.5",
                    "driver.docker.volumes.enabled": "true",
                    "driver.docker.bridge_ip": "172.17.0.1"
                },
                "Detected": true,
                "HealthDescription": "Healthy",
                "Healthy": true,
                "UpdateTime": "2020-01-30T13:12:16.604953+09:00"
            }
        },
        "ID": "4b635091-f444-34d6-5b33-a8aedcc49664",
        "ModifyIndex": 5689,
        "Name": "Takayukis-MBP",
        "NodeClass": "",
        "SchedulingEligibility": "eligible",
        "Status": "ready",
        "StatusDescription": "",
        "Version": "0.10.0"
    },
    {
        "Address": "127.0.0.1",
        "CreateIndex": 6,
        "Datacenter": "dc1",
        "Drain": false,
        "Drivers": {
            "qemu": {
                "Attributes": null,
                "Detected": false,
                "HealthDescription": "",
                "Healthy": false,
                "UpdateTime": "2020-01-30T13:12:16.584247+09:00"
            },
            "raw_exec": {
                "Attributes": {
                    "driver.raw_exec": "true"
                },
                "Detected": true,
                "HealthDescription": "Healthy",
                "Healthy": true,
                "UpdateTime": "2020-01-30T13:12:16.584363+09:00"
            },
            "docker": {
                "Attributes": {
                    "driver.docker.bridge_ip": "172.17.0.1",
                    "driver.docker.runtimes": "runc",
                    "driver.docker.os_type": "linux",
                    "driver.docker": "true",
                    "driver.docker.version": "19.03.5",
                    "driver.docker.volumes.enabled": "true"
                },
                "Detected": true,
                "HealthDescription": "Healthy",
                "Healthy": true,
                "UpdateTime": "2020-01-30T13:12:16.613387+09:00"
            },
            "java": {
                "Attributes": {
                    "driver.java.runtime": "OpenJDK Runtime Environment (build 12+33)",
                    "driver.java.vm": "OpenJDK 64-Bit Server VM (build 12+33, mixed mode, sharing)",
                    "driver.java": "true",
                    "driver.java.version": "12"
                },
                "Detected": true,
                "HealthDescription": "Healthy",
                "Healthy": true,
                "UpdateTime": "2020-01-30T13:12:16.66597+09:00"
            },
            "exec": {
                "Attributes": null,
                "Detected": false,
                "HealthDescription": "exec driver unsupported on client OS",
                "Healthy": false,
                "UpdateTime": "2020-01-30T13:12:16.584213+09:00"
            }
        },
        "ID": "8f5984b6-faf6-a485-c3cf-2729e26fc775",
        "ModifyIndex": 5690,
        "Name": "Takayukis-MBP",
        "NodeClass": "",
        "SchedulingEligibility": "eligible",
        "Status": "ready",
        "StatusDescription": "",
        "Version": "0.10.0"
    }
]
```
</details>

Nomadには、`drain`というコマンドを実行することでサーバメンテナンスやOSアップグレードなどの際にワークロードを移行することが出来ます。その際に各ノードを`ineligible`というモードに切り替え、新しいタスクのアロケーションなどを停止します。

```console
$ nomad node drain -enable -yes <NODE_ID>
2020-01-31T15:25:13+09:00: Ctrl-C to stop monitoring: will not cancel the node drain
2020-01-31T15:25:13+09:00: Node "6477d9ed-8bfd-2258-91f9-cab5965ff787" drain strategy set
2020-01-31T15:25:13+09:00: Drain complete for node 6477d9ed-8bfd-2258-91f9-cab5965ff787
2020-01-31T15:25:13+09:00: All allocations on node "6477d9ed-8bfd-2258-91f9-cab5965ff787" have stopped.
```

実際のアプリが稼働している際はこの処理で別のノードへ移行されます。再度ステータスを確認すると、`ineligible`に切り替わっています。この状態になると新しいタスクはこのノードにはアロケーションされません。

```console
$ nomad node status -verbose
ID                                    DC   Name           Class   Address    Version  Drain  Eligibility  Status
6477d9ed-8bfd-2258-91f9-cab5965ff787  dc1  Takayukis-MBP  <none>  127.0.0.1  0.10.0   false  ineligible   ready
4b635091-f444-34d6-5b33-a8aedcc49664  dc1  Takayukis-MBP  <none>  127.0.0.1  0.10.0   false  eligible     ready
8f5984b6-faf6-a485-c3cf-2729e26fc775  dc1  Takayukis-MBP  <none>  127.0.0.1  0.10.0   false  eligible     ready
```

メンテナンスが終わり再度タスクを受け付けたいような際は再度`eligible`の状態に切り替えます。

```console
$ nomad node eligibility -enable 6477d9ed-8bfd-2258-91f9-cab5965ff787
$ nomad node  status -verbose
ID                                    DC   Name           Class   Address    Version  Drain  Eligibility  Status
6477d9ed-8bfd-2258-91f9-cab5965ff787  dc1  Takayukis-MBP  <none>  127.0.0.1  0.10.0   false  eligible     ready
4b635091-f444-34d6-5b33-a8aedcc49664  dc1  Takayukis-MBP  <none>  127.0.0.1  0.10.0   false  eligible     ready
8f5984b6-faf6-a485-c3cf-2729e26fc775  dc1  Takayukis-MBP  <none>  127.0.0.1  0.10.0   false  eligible     ready
```

## alloc

`nomad alloc`を使うとアロケーションの様々な情報を取得出来ます。試しにログを取得してみます。

`初めてのNomad`の章でメモしたアロケーションIDを環境変数にセットしてから実施します。

```shell
$ export ALLOC=<ALLOCAION_ID>
$ nomad alloc logs $ALLOC
```

次にこのアロケーションに関するファイルシステムを操作します。通常大元のディレクトリなどを辿っていかないといけませんが`fs`コマンドを利用することでアロケーションIDに紐づいた情報を取得してくれます。

```console
$ nomad alloc fs $ALLOC
Mode        Size   Modified Time              Name
drwxrwxrwx  160 B  2020-01-31T15:55:22+09:00  alloc/
drwxrwxrwx  160 B  2020-01-31T15:55:22+09:00  redis/

$ nomad alloc fs $ALLOC alloc/
Mode        Size   Modified Time              Name
drwxrwxrwx  64 B   2020-01-31T15:55:22+09:00  data/
drwxrwxrwx  192 B  2020-01-31T15:55:22+09:00  logs/
drwxrwxrwx  64 B   2020-01-31T15:55:22+09:00  tmp/

$ nomad alloc fs -stat $ALLOC alloc/logs/redis.stdout.0
Mode        Size     Modified Time              Content Type               Name
-rw-r--r--  2.0 KiB  2020-01-31T15:55:23+09:00  text/plain; charset=utf-8  redis.stdout.0

$ nomad alloc fs $ALLOC alloc/logs/redis.stdout.0
```

ログが取得出来たはずです。`fs`はディレクトリだと`ls`, ファイルだと`cat`と同様の出力になります。また`-stat`オプションを使うことでファイルの情報も確認できました。この他にも`-tail`オプションなども用意されています。

また、`nomad alloc status`はアロケーションのステータスを確認することが出来ます。

```console
$ nomad alloc status $ALLOC
ID                  = 4d51a7ea
Eval ID             = 401db6fb
Name                = example.cache[0]
Node ID             = 4b635091
Node Name           = Takayukis-MBP
Job ID              = example
Job Version         = 1
Client Status       = running
Client Description  = Tasks are running
Desired Status      = run
Desired Description = <none>
Created             = 24m25s ago
Modified            = 1m33s ago
Deployment ID       = 5ca611b5
Deployment Health   = unhealthy

Task "redis" is "running"
Task Resources
CPU        Memory           Disk     Addresses
7/500 MHz  984 KiB/256 MiB  300 MiB  db: 192.168.3.38:26883

Task Events:
Started At     = 2020-01-31T07:18:15Z
Finished At    = N/A
Total Restarts = 1
Last Restart   = 2020-01-31T16:17:57+09:00

Recent Events:
Time                       Type             Description
2020-01-31T16:18:15+09:00  Started          Task started by client
2020-01-31T16:17:57+09:00  Restarting       Task restarting in 16.669875916s
2020-01-31T16:17:57+09:00  Terminated       Exit Code: 137, Exit Message: "Docker container exited with non-zero exit code: 137"
2020-01-31T16:17:57+09:00  Signaling        Task being sent a signal
2020-01-31T16:00:22+09:00  Alloc Unhealthy  Task not running for min_healthy_time of 10s by deadline
2020-01-31T15:55:23+09:00  Started          Task started by client
2020-01-31T15:55:22+09:00  Task Setup       Building Task Directory
2020-01-31T15:55:22+09:00  Received         Task received by client
```

これはアロケーションが失敗した際など、非常によく使うコマンドです。

その他にも`restart`, `stop`などライフサイクルを扱うコマンドも用意されています。

## job

`job`コマンドはジョブに対する様々の操作や情報を表示するためのコマンドです。

まずは`deployments`を試してみましょう。アロケーションの履歴とステータスを取得出来ます。

```console
$ nomad job deployments example
ID        Job ID     Job Version  Status      Description
56f8ca80  mysql-5.7  0            successful  Deployment completed successfully
```

`history`はバージョンを出力します。ジョブを変更してバージョンを上げ行った際に現状のバージョンやバージョンアップされた日時などを確認する際に利用します。ここでは実施しませんが`job revert`コマンドでジョブのロールバックなどを行います。

```console
$ nomad job history mysql-5.7
Version     = 0
Stable      = true
Submit Date = 2020-01-31T15:53:24+09:00
```

`eval`コマンドはEvaluationのステータスを確認するためのコマンドです。`初めてのNomad`の章でメモしたEvaluation IDをコピーして下さい。

```console
$ nomad job eval status <EVAL_ID>

ID                 = 720ff6d0
Create Time        = 35m23s ago
Modify Time        = 35m23s ago
Status             = complete
Status Description = complete
Type               = service
TriggeredBy        = job-register
Job ID             = mysql-5.7
Priority           = 50
Placement Failures = true

Failed Placements
Task Group "mysql-group" (failed to place 1 allocation):
  * Constraint "missing compatible host volumes" filtered 2 nodes

Evaluation "0d142ec3" waiting for additional capacity to place remainder
```

`status`はよく利用します。ジョブの状態やジョブに紐づくアロケーションIDを確認することが出来ます。

```
$ nomad job status example
ID            = example
Name          = example
Submit Date   = 2020-01-31T15:50:09+09:00
Type          = service
Priority      = 50
Datacenters   = dc1
Status        = running
Periodic      = false
Parameterized = false

Summary
Task Group  Queued  Starting  Running  Failed  Complete  Lost
cache       0       0         1        2       0         0

Latest Deployment
ID          = 5ca611b5
Status      = failed
Description = Failed due to progress deadline

Deployed
Task Group  Desired  Placed  Healthy  Unhealthy  Progress Deadline
cache       1        2       0        2          2020-01-31T16:00:09+09:00

Allocations
ID        Node ID   Task Group  Version  Desired  Status   Created     Modified
4d51a7ea  4b635091  cache       1        run      running  43m35s ago  20m43s ago
96209690  6477d9ed  cache       1        stop     failed   49m ago     43m35s ago
3761cbaf  4b635091  cache       0        stop     failed   53m32s ago  43m48s ago
```

`inspect`はNomadのジョブの情報の詳細を取得するためのコマンドです。

```shell
$ nomad inspect example
```

<details><summary>出力結果例</summary>
	
```json
{
    "Job": {
        "Affinities": null,
        "AllAtOnce": false,
        "Constraints": null,
        "CreateIndex": 6992,
        "Datacenters": [
            "dc1"
        ],
        "Dispatched": false,
        "ID": "example",
        "JobModifyIndex": 7017,
        "Meta": null,
        "Migrate": null,
        "ModifyIndex": 7103,
        "Name": "example",
        "Namespace": "default",
        "ParameterizedJob": null,
        "ParentID": "",
        "Payload": null,
        "Periodic": null,
        "Priority": 50,
        "Region": "global",
        "Reschedule": null,
        "Spreads": null,
        "Stable": false,
        "Status": "running",
        "StatusDescription": "",
        "Stop": false,
        "SubmitTime": 1580453409843865000,
        "TaskGroups": [
            {
                "Affinities": null,
                "Constraints": null,
                "Count": 1,
                "EphemeralDisk": {
                    "Migrate": false,
                    "SizeMB": 300,
                    "Sticky": false
                },
                "Meta": null,
                "Migrate": {
                    "HealthCheck": "checks",
                    "HealthyDeadline": 300000000000,
                    "MaxParallel": 1,
                    "MinHealthyTime": 10000000000
                },
                "Name": "cache",
                "Networks": null,
                "ReschedulePolicy": {
                    "Attempts": 0,
                    "Delay": 30000000000,
                    "DelayFunction": "exponential",
                    "Interval": 0,
                    "MaxDelay": 3600000000000,
                    "Unlimited": true
                },
                "RestartPolicy": {
                    "Attempts": 2,
                    "Delay": 15000000000,
                    "Interval": 1800000000000,
                    "Mode": "fail"
                },
                "Services": null,
                "Spreads": null,
                "Tasks": [
                    {
                        "Affinities": null,
                        "Artifacts": null,
                        "Config": {
                            "port_map": [
                                {
                                    "db": 6379.0
                                }
                            ],
                            "image": "redis:3.2"
                        },
                        "Constraints": null,
                        "DispatchPayload": null,
                        "Driver": "docker",
                        "Env": null,
                        "KillSignal": "",
                        "KillTimeout": 5000000000,
                        "Kind": "",
                        "Leader": false,
                        "LogConfig": {
                            "MaxFileSizeMB": 10,
                            "MaxFiles": 10
                        },
                        "Meta": null,
                        "Name": "redis",
                        "Resources": {
                            "CPU": 500,
                            "Devices": null,
                            "DiskMB": 0,
                            "IOPS": 0,
                            "MemoryMB": 256,
                            "Networks": [
                                {
                                    "CIDR": "",
                                    "Device": "",
                                    "DynamicPorts": [
                                        {
                                            "Label": "db",
                                            "To": 0,
                                            "Value": 0
                                        }
                                    ],
                                    "IP": "",
                                    "MBits": 10,
                                    "Mode": "",
                                    "ReservedPorts": null
                                }
                            ]
                        },
                        "Services": [
                            {
                                "AddressMode": "auto",
                                "CanaryTags": null,
                                "CheckRestart": null,
                                "Checks": [
                                    {
                                        "AddressMode": "",
                                        "Args": null,
                                        "CheckRestart": null,
                                        "Command": "",
                                        "GRPCService": "",
                                        "GRPCUseTLS": false,
                                        "Header": null,
                                        "Id": "",
                                        "InitialStatus": "",
                                        "Interval": 10000000000,
                                        "Method": "",
                                        "Name": "alive",
                                        "Path": "",
                                        "PortLabel": "",
                                        "Protocol": "",
                                        "TLSSkipVerify": false,
                                        "TaskName": "",
                                        "Timeout": 2000000000,
                                        "Type": "tcp"
                                    }
                                ],
                                "Connect": null,
                                "Id": "",
                                "Meta": null,
                                "Name": "redis-cache",
                                "PortLabel": "db",
                                "Tags": [
                                    "global",
                                    "cache"
                                ]
                            }
                        ],
                        "ShutdownDelay": 0,
                        "Templates": null,
                        "User": "",
                        "Vault": null,
                        "VolumeMounts": null
                    }
                ],
                "Update": {
                    "AutoPromote": false,
                    "AutoRevert": false,
                    "Canary": 0,
                    "HealthCheck": "checks",
                    "HealthyDeadline": 300000000000,
                    "MaxParallel": 1,
                    "MinHealthyTime": 10000000000,
                    "ProgressDeadline": 600000000000,
                    "Stagger": 30000000000
                },
                "Volumes": null
            }
        ],
        "Type": "service",
        "Update": {
            "AutoPromote": false,
            "AutoRevert": false,
            "Canary": 0,
            "HealthCheck": "",
            "HealthyDeadline": 0,
            "MaxParallel": 1,
            "MinHealthyTime": 0,
            "ProgressDeadline": 0,
            "Stagger": 30000000000
        },
        "VaultToken": "",
        "Version": 1
    }
}
```
</details>

その他にも`run`, `stop`などジョブの起動停止を行うようなコマンドやイベント処理をinvokeする`dispatch`、ジョブをCanaryでプロモーションするための`promote`など様々な操作を`job`コマンドを利用して行います。

## monitor

`monitor`コマンドはNomadエージェントのログを取得するためのコマンドで、デバッグログなどを確認したいときにとても便利です。

```console
$ nomad monitor -log-level=DEBUG
2020-02-01T23:38:02.316+0900 [WARN]  nomad: raft: Heartbeat timeout from "" reached, starting election
2020-02-01T23:38:02.316+0900 [INFO]  nomad: raft: Node at 127.0.0.1:4647 [Candidate] entering Candidate state in term 34
2020-02-01T23:38:02.353+0900 [DEBUG] nomad: raft: Votes needed: 1
2020-02-01T23:38:02.353+0900 [DEBUG] nomad: raft: Vote granted from 127.0.0.1:4647 in term 34. Tally: 1
```

`-node-id`や`-server-id`でログを絞ったり、`-json`でJSON形式でログを出力することも可能です。

## その他のコマンド

そのほかに`namespace`, `quota`や`sentinel`などエンタープライズ版のみ利用可能なコマンドも用意されています。この辺りは`Enterprise機能の紹介`のスライドで確認してみて下さい。

また`operator`コマンドに関しては[Consul Workshop](https://github.com/hashicorp-japan/consul-workshop/blob/master/contents/cli.md)に同様の仕組みの解説と手順が載っているので興味のある方は試してみて下さい。

## 参考リンク
* [nomad cli](https://www.nomadproject.io/docs/commands/index.html)