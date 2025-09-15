# Nomad でシステムジョブを動かす

今まで様々な Task Driver を利用して Web アプリケーションをメインに動かしてきました。Nomad ではジョブ設定ファイルの`job`スタンザ内の`type`というパラメータに`Scheduler`を指定することでざまざまなタイプのワークロードを稼働させることが出来ます。

* Service Scheduler: 
	* Long running なタスク。
	* Operator が Stop するまで Running 状態が維持される。
* Batch Scheduler: 
	* Short lived なタスク。
	* Task が終了するまで実行される。
* System Scheduler: 
	* 全ての Client 上で実行されるタスク。
	* Operator が Stop するか、Preempted されるまで Running 状態を維持される。
	* 新たな Client が Join した際にも起動される。

今まで`service`や`batch`を試してきました。ここでは`System Scheduler`を利用してジョブを実行してみたいと思います。

https://www.nomadproject.io/docs/schedulers.html#system
