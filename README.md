# HashiCorp Nomad Workshop

[Nomad](https://www.nomadproject.io/) は HashiCorp が中心に開発をする OSS のオーケストレーションツールです。軽量なアーキテクチャかつ幅広いタイプのアプリケーションに対応し、高度なオーケストレーション機能を提供しつつシンプルな運用モデルを提供します。Nomad はマルチプラットフォームでかつ全ての機能を HTTP API で提供しているため、環境やクライアントを問わず利用することができます。また、Vault や Consul などの HashiCorp 製品と連携することで、ロードバランシング、Service Discovery やシークレット管理など高度なプラットフォームとしての機能を持たせることも可能です。

本ワークショップは OSS の機能を中心に様々なユースケースに合わせたハンズオンを用意しています。

## Pre-requisite

* 環境
	* macOS or Linux

* ソフトウェア
	* Nomad
	* Docker
	* Java 12(いつか直します...)
	* jq, watch, wget, curl, openssl
	* PHP
	* Go

## 資料

* [Nomad Overview](https://docs.google.com/presentation/d/1NtORrEVI0kovBeQSgmsYbs1InnEnRqv9uke8F_HzP-U/edit?usp=sharing)

## アジェンダ
* [初めての Nomad](contents/hello_nomad.md)
* [Nomad 用語集](contents/words.md)
* [nomad cli](contents/cli.md)
* Task Drivers
	* [Docker Task Driver](contents/docker.md) (+ Volume)
	* [Java Task Driver](contents/java.md) (+ Artifact & Logs)
	* [Exec Task Driver](contents/exec.md) (+ Affinity & Spread & Constraint)
* Schedulers
	* [Batch Scheduler](contents/batch.md)
	* System Scheduler
* [アプリケーションのアップデート](contents/app_update.md)
* [HashiCorp Consul との連携](contents/nomad-consul.md)
* [HashiCorp Vault との連携](contents/nomad-vault.md)
* [Enterprise 版機能の紹介](https://docs.google.com/presentation/d/1pNWXiETt9t5gOQY3dsvuQJJc0adzW9SEq6U73UhZIVI/edit?usp=sharing)

