# 最低限押さえておきたいNomadの用語集

NomadではDockerのようなコンテナのワークロードにとどまらず、スタンドアロンの`Java`, `RaW Exec`や`Qemu`など様々なタイプのアプリケーションを実行できます。

そのためにNomadでは様々な定義を宣言的に行いますが、ここでは以降の章で扱う実際のアプリのデプロイの前に最低限理解が必要な用語をまとめておきます。

ハンズオン作業はないので次に進む前に一読してください。

## Nomad Job定義

NomadのJobは大きく`Job`, `Task Group`, `Task`の3階層になって分けて定義されます。

* Job
	* NomadにデプロイするアプリケーションはJobという単位で定義されます。Jobには`desired status`つまり、そのアプリケーションのあるべき状態を定義します。以下が代表的な定義の例です、
		* 「どこで」実行されるか(どこのデータセンターか、どこのリージョンか)
		* 「どのような」タイプのジョブか(Batch or Long Running Process)
		* 「何を」動かすか
		* 「どのように」ジョブのアップデートを行うか
	* Jobで稼働するアプリケーションの詳細な内容の設定は以下のTask Groupで構成されます。
* Task Group
	* Jobの中に定義し、一つのJobに複数のGroupを定義可能です。Task Groupの中には以下のような定義をします。
		* Restart: 再起動ポリシー
		* Count: 同時に実行するタスクの数
		* Task: 実際のタスクの定義。この下にさらに詳細なタスクの定義
* Task
	* Task Groupの中に定義し、一つのTask Groupに複数のTaskを定義可能です。Taskの中には以下のような定義をします。
		* Driver: アプリケーションの種類を指定。(java, docker, exec etc)
		* Config: 各ドライバーを実行するためのコンフィグレーション。JavaならjarのパスやJVMオプション、DockerならPort Mappingなどを指定。
		* Resouce: タスクに割り当てるCPU, Memory, Networkなどを指定。

これらの定義は実際にアプリをデプロイする前に`HCL`や`JSON`などの形式で定義し、アプリをデプロイする際に`nomad`コマンドの引数としてセットします。Nomadサーバがそれを解釈し宣言された通りにNomadクライアントへアプリをデプロイします。

次はそのNomadサーバが行う内部処理の用語を整理します。

## Nomadの内部処理

Nomadのスケージューリングのフローは下記のイメージです。

![](https://www.nomadproject.io/assets/images/nomad-data-model-39de5cfc.png)

ここではそれぞれの処理について簡単に説明をします。

* Evaluation
	* 意図的な、及び意図しない変更によって状態が変化した時に実行されるプロセスで、Nomadサーバで実行されます。新しいジョブが発生した時、既存のジョブがアップデートされた時、削除された時などDesiredが変更された際や、障害などの意図しない変更の際にCreateされ、Desired Statusに状態を更新します。Evaluationはサーバ側のEvaluation Brokerにキューイングされてスケージューラによってデキューイングされます。
* Allocation
	* 一つのジョブで何百、何千というタスクが稼働する際、どこのノードで処理をさせるのかを決定することはとても重要です。スケージューラがEvalutionの処理を実際に処理するために作られるプロセスで、どのジョブがどのノードで稼働するかを決定します。スケジューラはEvaluationを取得したら、Applocation Planを生成します。Allocation Planは二つのフェーズがあります。
		* Cheking: 「unhealthyのノードはないか」「タスクで定義されたワークロードを稼働させるためのDriverがないノードはないか」などを確認するフェーズ
		* Ranking: タスク実行にあたって最適なノードをランク付けするフェーズ。`Bin Packging`の結果を元にスコアされる。
	* Allocation Planでランキングされたら、高いランクのノードにAllocation Planにアサインされます。キュイングされたAllocationはノードによって取得され、実際のタスクが実行されます。
* Bin Packing
	* リソースの使用率とノード上のアプリケーションの密度を最適化するための処理です。ワークロードを実行しているノード上の配置やリソース配分を最適化し、インフラ全体のコストを改善するために実行されます。

## 参考リンク
* [Nomad Architecture](https://www.nomadproject.io/docs/internals/architecture.html)
* [Nomad Scheduling](https://www.nomadproject.io/docs/internals/scheduling/scheduling.html)
* [Nomad Job Specification](https://www.nomadproject.io/docs/job-specification/index.html)