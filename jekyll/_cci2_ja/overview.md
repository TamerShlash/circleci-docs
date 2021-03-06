---
layout: classic-docs
title: "管理者向けの概要"
category:
  - administration
order: 1
description: "CircleCI インストールプロセスの概要"
---

システム管理者を対象に、以下のセクションに沿って CircleCI 2.0 インストール、機能、環境、およびアーキテクチャの概要について説明します。

* 目次
{:toc}

CircleCI は、最新の継続的インテグレーションおよび継続的デリバリー (CI/CD) のプラットフォームです。 プライベートクラウドまたはデータセンター内にインストールでき、一定期間は無料でお試しいただけます。 [評価版ライセンス](https://circleci.com/enterprise-trial-install)をご希望の方は、お問い合わせください。

CircleCI 2.0 では、以下の改善点を含む新しいインフラストラクチャが提供されています。

* 任意の数のジョブとワークフローを組織化できる新しい設定機能
* ジョブごとの実行を行えるカスタムイメージ
* カスタムキャッシュ、ジョブごとの CPU またはメモリの割り当てなど、きめ細かいパフォーマンス

ユーザーまたは開発者の方は、「[CircleCI を始める]({{ site.baseurl }}/ja/2.0/first-steps/)」を参照しながら、ホスティングされているアプリケーションの使用を開始してください。

## インストールオプション

お使いの環境に CircleCI をインストールするには、3 つの基本的な方法があります (現在いずれのオプションでも AWS が必須です)。

1. 評価版ライセンスの利用時や小規模チームに適している[シングルボックスインストール]({{ site.baseurl }}/ja/2.0/single-box/)
2. 多くのチームでの本稼働使用に適している[クラスタ化インストール]({{ site.baseurl }}/ja/2.0/aws/)
3. 高度な稼働時間要件を満たすための[高可用性設定]({{ site.baseurl }}/ja/2.0/high-availability/) (ライセンスでサポートされている場合)

## ビルド環境

デフォルトでは、CircleCI 2.0 Builder インスタンスは、`.circleci/config.yml` ファイルでジョブごとに設定されたイメージに従ってコンテナを自動的にプロビジョニングします。 CircleCI は、CircleCI 2.0 のプライマリジョブスケジューラとして Nomad を使用します。ジョブスケジューラ、およびクライアントとクラスタの基本的な操作方法については、「[CircleCI での Nomad クラスタの操作ガイド]({{ site.baseurl }}/ja/2.0/nomad/)」を参照してください。

## アーキテクチャ

CircleCI は、Services と Nomad クライアントという 2つのプライマリコンポーネントで構成されます。 Services は通常、コアアプリケーション、ストレージ、ネットワーク通信機能から成る単一のインスタンス上で動作します。 任意の数の Nomad クライアントがジョブを実行し、Services と通信します。 2つのコンポーネントはどちらも、以下のアーキテクチャ図に示すように、ネットワーク上で GitHub または GitHub Enterprise のホスティングされたインスタンスにアクセスする必要があります。

![CircleCI のアーキテクチャの図]({{site.baseurl}}/assets/img/docs/architecture-v1.png)

### Services

Service インスタンスが動作するマシンは、再起動してはならず、組み込みの VM スナップショット機能を使用してバックアップできます。 **メモ：** 高可用性を実現するには、PostgreSQL と Mongo を使用して外部データストレージを設定します。また、データベースのバックアップには、標準のツールを使用します。詳細については、[高可用性のための外部データベースホストの追加に関するドキュメント]({{ site.baseurl }}/ja/2.0/high-availability/)を参照してください。 DNS 解決は、Services がインストールされているマシンの IP アドレスを指す必要があります。 以下の表に、Service インスタンスでトラフィックに使用されるポートを示します。

| ソース                          | ポート                | 用途                |
| ---------------------------- | ------------------ | ----------------- |
| エンドユーザー                      | 80、443、4434        | HTTP/HTTPS トラフィック |
| 管理者                          | 22                 | SSH               |
| 管理者                          | 8800               | 管理者コンソール          |
| Builder Boxes                | すべてのトラフィック/すべてのポート | 内部通信              |
| GitHub (Enterprise または .com) | 80、443             | 受信 Web フック        |
{: class="table table-striped"}

### Nomad クライアント

Nomad クライアントは実行時に状態を保存しないため、必要に応じてコンテナ数を増減することができます。 すべてのビルドを処理するために十分な数のクライアントマシンが確実に実行されるようにするには、キューイングされているビルドを追跡し、必要に応じて Nomad クライアントマシンを増やして負荷を分散させます。

マシンには、ビルドを調整するために 2つの CPU と 4 GB のメモリが予約されています。 そのうえで、残りのプロセッサーとメモリでコンテナが作成されます。 マシンの規模が大きくなれば、その分多くのコンテナを実行することができますが、調整用に予約してある 2つ以外に使用できるコアの数によって制限されます。 以下の表に、Builder インスタンスで使用されるポートを示します。

| ソース                    | ポート                | 用途                                                               |
| ---------------------- | ------------------ | ---------------------------------------------------------------- |
| エンドユーザー                | 64535-65535        | [ビルド機能への SSH 接続](https://circleci.com/docs/2.0/ssh-access-jobs/) |
| 管理者                    | 80 または 443         | CircleCI API アクセス (正しいシャットダウンなど)                                 |
| 管理者                    | 22                 | SSH                                                              |
| Services Box           | すべてのトラフィック/すべてのポート | 内部通信                                                             |
| Nomad クライアント (それ自身を含む) | すべてのトラフィック/すべてのポート | 内部通信                                                             |
{: class="table table-striped"}

### GitHub

CircleCI では、認証に GitHub または GitHub Enterprise 認証情報が使用され、認証時には LDAP、SAML、または SSH を使用してアクセスできます。 つまり CircleCI では、一元管理された SSO インフラストラクチャによってサポートされる認証が継承されます。 以下の表に、GitHub を実行するマシンで Services および Builder インスタンスと通信する際に使用されるポートを示します。

| ソース          | ポート    | 用途       |
| ------------ | ------ | -------- |
| Services     | 22     | Git アクセス |
| Services     | 80、443 | API アクセス |
| Nomad クライアント | 22     | Git アクセス |
| Nomad クライアント | 80、443 | API アクセス |
{: class="table table-striped"}

## お客様事例

**Coinbase** は、セキュリティと信頼性を確保するために CircleCI をファイアウォールの内側で実行しています。 [Coinbase のケーススタディレポート](https://circleci.com/customers/coinbase/)によれば、CircleCI を使用することでマージからデプロイまでの平均時間が半分になり、CI のメンテナンスに費やす操作時間は要員 1人の時間の 50% を占めていたものが週 1時間未満に短縮され、開発者のスループットが 20% 増加しています。

GM 傘下の **Cruise Automation** では、CircleCI の導入により、コードの実地テストを行う前に多数のシミュレーションを行うことが可能になりました。詳しくは [Cruise のケーススタディレポート](https://circleci.com/customers/cruise/)をご覧ください。

[CircleCI のカスタマーページ](https://circleci.com/customers/)では、CircleCI を導入されている大企業および中小企業のすべてのケーススタディを参照していただけます。
