# 最低限押さえておきたい Nomad の用語集

Nomad では Docker のようなコンテナのワークロードにとどまらず、スタンドアロンの`Java`, `RaW Exec`や`Qemu`など様々なタイプのアプリケーションを実行できます。

そのために Nomad では様々な定義を宣言的に行いますが、ここでは以降の章で扱う実際のアプリのデプロイの前に最低限理解が必要な用語をまとめておきます。

ハンズオン作業はないので次に進む前に一読してください。

## Nomad Job 定義

Nomad の Job は大きく`Job`, `Task Group`, `Task`の 3 階層になって分けて定義されます。

* Job
	* Nomad にデプロイするアプリケーションは Job という単位で定義されます。Job には`desired status`つまり、そのアプリケーションのあるべき状態を定義します。以下が代表的な定義の例です、
		* 「どこで」実行されるか(どこのデータセンターか、どこのリージョンか)
		* 「どのような」タイプのジョブか(Batch or Long Running Process)
		* 「何を」動かすか
		* 「どのように」ジョブのアップデートを行うか
	* Job で稼働するアプリケーションの詳細な内容の設定は以下の Task Group で構成されます。
* Task Group
	* Job の中に定義し、一つの Job に複数の Group を定義可能です。Task Group の中には以下のような定義をします。
		* Restart: 再起動ポリシー
		* Count: 同時に実行するタスクの数
		* Task: 実際のタスクの定義。この下にさらに詳細なタスクの定義
* Task
	* Task Group の中に定義し、一つの Task Group に複数の Task を定義可能です。Task の中には以下のような定義をします。
		* Driver: アプリケーションの種類を指定。(java, docker, exec etc)
		* Config: 各ドライバーを実行するためのコンフィグレーション。Java なら jar のパスや JVM オプション、Docker なら Port Mapping などを指定。
		* Resouce: タスクに割り当てる CPU, Memory, Network などを指定。

これらの定義は実際にアプリをデプロイする前に`HCL`や`JSON`などの形式で定義し、アプリをデプロイする際に`nomad`コマンドの引数としてセットします。Nomad サーバがそれを解釈し宣言された通りに Nomad クライアントへアプリをデプロイします。

次はその Nomad サーバが行う内部処理の用語を整理します。

## Nomad の内部処理

Nomad のスケージューリングのフローは下記のイメージです。

![](https://www.nomadproject.io/assets/images/nomad-data-model-39de5cfc.png)

ここではそれぞれの処理について簡単に説明をします。

* Evaluation
	* 意図的な、及び意図しない変更によって状態が変化した時に実行されるプロセスで、Nomad サーバで実行されます。新しいジョブが発生した時、既存のジョブがアップデートされた時、削除された時など Desired が変更された際や、障害などの意図しない変更の際に Create され、Desired Status に状態を更新します。Evaluation はサーバ側の Evaluation Broker にキューイングされてスケージューラによってデキューイングされます。
* Allocation
	* 一つのジョブで何百、何千というタスクが稼働する際、どこのノードで処理をさせるのかを決定することはとても重要です。スケージューラが Evalution の処理を実際に処理するために作られるプロセスで、どのジョブがどのノードで稼働するかを決定します。スケジューラは Evaluation を取得したら、Applocation Plan を生成します。Allocation Plan は二つのフェーズがあります。
		* Cheking: 「unhealthy のノードはないか」「タスクで定義されたワークロードを稼働させるための Driver がないノードはないか」などを確認するフェーズ
		* Ranking: タスク実行にあたって最適なノードをランク付けするフェーズ。`Bin Packging`の結果を元にスコアされる。
	* Allocation Plan でランキングされたら、高いランクのノードに Allocation Plan にアサインされます。キュイングされた Allocation はノードによって取得され、実際のタスクが実行されます。
* Bin Packing
	* リソースの使用率とノード上のアプリケーションの密度を最適化するための処理です。ワークロードを実行しているノード上の配置やリソース配分を最適化し、インフラ全体のコストを改善するために実行されます。

## 参考リンク
* [Nomad Architecture](https://www.nomadproject.io/docs/internals/architecture.html)
* [Nomad Scheduling](https://www.nomadproject.io/docs/internals/scheduling/scheduling.html)
* [Nomad Job Specification](https://www.nomadproject.io/docs/job-specification/index.html)