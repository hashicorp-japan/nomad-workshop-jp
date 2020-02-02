# HashiCorp Nomad Workshop

[Nomad](https://www.nomadproject.io/)はHashiCorpが中心に開発をするOSSのオーケストレーションツールです。軽量なアーキテクチャかつ幅広いタイプのアプリケーションに対応し、高度なオーケストレーション機能を提供しつつシンプルな運用モデルを提供します。Nomadはマルチプラットフォームでかつ全ての機能をHTTP APIで提供しているため、環境やクライアントを問わず利用することができます。また、VaultやConsulなどのHashiCorp製品と連携することで、ロードバランシング、Service Discoveryやシークレット管理など高度なプラットフォームとしての機能を持たせることも可能です。

本ワークショップはOSSの機能を中心に様々なユースケースに合わせたハンズオンを用意しています。

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
* [初めてのNomad](contents/hello_nomad.md)
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
* HashiCorp Consulとの連携
* HashiCorp Vaultとの連携
* Enterprise版機能の紹介

