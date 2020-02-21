# Nomadでシステムジョブを動かす

今まで様々なTask Driverを利用してWebアプリケーションをメインに動かしてきました。Nomadではジョブ設定ファイルの`job`スタンザ内の`type`というパラメータに`Scheduler`を指定することでざまざまなタイプのワークロードを稼働させることが出来ます。

* Service Scheduler: 
	* Long runningなタスク。
	* OperatorがStopするまでRunning状態が維持される。
* Batch Scheduler: 
	* Short livedなタスク。
	* Taskが終了するまで実行される。
* System Scheduler: 
	* 全てのClient上で実行されるタスク。
	* OperatorがStopするか、PreemptedされるまでRunning状態を維持される。
	* 新たなClientがJoinした際にも起動される。

今まで`service`や`batch`を試してきました。ここでは`System Scheduler`を利用してジョブを実行してみたいと思います。

https://www.nomadproject.io/docs/schedulers.html#system
