---
title: "【個人開発】ローカルサーバーにインターネットからアクセス可能にするサービスを作ったので使ってほしい"
emoji: "🐈‍⬛"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["個人開発", "SSH", "ngrok", "localhostrun"]
published: true
---

## サービスの概要

「[TCP Exposer](https://www.tcpexposer.com/)」という、プライベートIPアドレスしか持たないサーバー(以降はローカルサーバーと呼びます)に、インターネットからアクセスできるようにするサービスを作りました。
多くの人に使ってもらいたいのでPRも兼ねて本記事を書いています。
下図は本サービスのトップページのスクリーンショットです。

![トップページ](/images/tcpexposer-intoroduction/tcpexposer_top.png)
*本サービスのトップページ*

使い方は簡単で ```ssh -R 80:localhost:8080 anonymous@tcpexposer.com``` のようなコマンドをローカルサーバーで実行するだけです。
これを実行すると下記シーケンス図のように、ローカルサーバーの8080番ポートにインターネットからアクセスできるようになります。

```mermaid
sequenceDiagram
    actor client as リモートクライアント
    participant exposer as TCP Exposer
    participant local as ローカルサーバー

    local ->> exposer : SSHで接続
    exposer -->> local : 接続用のサブドメインもしくはTCPポート番号の通知

    Note over client,local: TCP Exposerとローカルサーバー間のSSH接続確立後

    client ->> exposer : 接続用のサブドメインもしくはTCPポート番号へリクエスト
    exposer ->> local : リクエストを転送

    local -->> exposer : リクエストへのリプライ
    exposer -->> client : リプライを転送
```

また、本サービスを開発・運用する際に利用したソフトウェアや他サービスとその選定理由については[こちらのZennの記事](https://zenn.dev/teasy/articles/tcpexposer-tech)に記載しています。
ご興味が出たらチェックしてみてください。


<!-- ###################################################################### -->


## 開発背景

できるだけお金をかけずにWEBサービスを公開したいというのがきっかけです。
静的なWEBページであればGitHub Pagesなど無料で公開できるのですが、動的なものはDBを常時稼働させる必要があり、クラウドサービスの無料枠で収まらない可能性があります。
(ただし、実際はHerokuのPostgreSQL無料枠などをうまく使えば、WEBサービスによっては無料や少額で運用できると思います。)

そのような課題感から、今は使わなくなったRaspberryPiやPCをサーバーとして利用し、インターネットに公開することで解決できないか検討しました。
インターネットにサーバーを公開する場合、基本的にはグローバルIPアドレスが必須です。
しかし、多くのインターネットサービスプロバイダーでは、グローバルIPアドレスを利用するには費用がかかったり、そもそもプライベートIPアドレスしか使えなかったりします。

プライベートIPアドレスしか使えない場合は、別途グローバルIPアドレスを持つサーバーを用意し、それとローカルサーバーをVPNなどで接続することが必要です。
そして、グローバルにあるサーバーを経由することで、インターネットからローカルサーバーにアクセスできるようになります。

グローバルにあるサーバーは通信の暗号化や転送をするだけなので、(転送先が少ないうちは)必要なスペックは少ないため、同じ悩みを持つ人と共有することができるのではないか、というのが本サービスを開発した理由です。
(ちなみに当初作ろうとしていたサービスは、イケてなさそうだったので開発しませんでした。)

開発途中で類似サービスの存在を知って開発をやめるか迷いましたが、物足りない点があったので開発を続行しサービス開始することにしました。


<!-- ###################################################################### -->


## 類似サービスとの違い

### サービスの種類

ngrokなどのサービスもしくはVPNサービスが競合となると考えています。

ngrokや本サービスは、ローカルサーバーと経由サーバーの間にファイアウォールやNATを超える通信経路を確立します。
ローカルサーバーのクライアントは、その通信経路を経由してアクセスします。
これらはローカルサーバーで動くアプリを公開したい人向けのサービスと言えます。

また、例えば外出先から自宅のサーバーにアクセスしたいというユースケースなどでは、VPNサービスも競合となると考えています。
VPNサービスと違い本サービスは認証機能を提供しませんが、例えばSSHサービスのポート番号を公開する場合は、SSHサーバー自体の認証情報を知っている人のみしかアクセスできません。


### 類似サービスの紹介

類似サービスのうち、私が知っている範囲では下記サービスが比較的有名だと思います。
なお、localhost.runは少し使いましたが、基本的には公式ドキュメント等を斜め読みしたものなので、間違いがあるかもしれません。

- [ngrok](https://ngrok.com/)

    この手のサービスでは一番有名だと思います。
    本サービスと同様に、HTTP(S)サーバーだけでなくTCPサーバーも公開できます。
    ngrokは専用のクライアントソフトをローカルサーバーにインストールする必要がありますが、本サービスは通常のSSHクライアントで良いのでインストールは不要です。

- [Cloudflare Zero Trust / Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)

    CDNサービスで有名なCloudflareも類似のサービスを出しています。
    本サービスのようにもVPNサービスのようにも使えそうです。
    ドメインの取得が必要なのと、専用のクライアントソフトをローカルサーバーにインストールする必要があります。

- [Tailscale](https://tailscale.com/)

    VPNサービスです。
    そのため、ローカルサーバーを広く一般に公開することはできませんし、それを目的にしたサービスではありません。
    しかし、本サービスと違い、ローカルサーバーとそのクライアントはTailscaleを経由せずに通信するため通信速度が速いようです。
    一方専用のクライアントをローカルサーバーとクライアントにインストールする必要があります。

- [localhost.run](https://localhost.run/)

    本サービスと非常に似ています。
    通常のSSHクライアントを使うため専用のクライアントソフトは不要で、実行するコマンドも本サービスとほぼ同じです。
    localhost.runはTCPサーバーを公開することはできませんが、本サービスはできます。


### 本サービスの特徴

本サービスの特徴は下記であると思っています。

- 専用クライアントが不要です。

    これが一番の売りです。
    WindowsやMac、Linuxには最初からインストールされているソフトウェア(SSHクライアント)で利用できます。
    専用ソフトをインストールするのは個人的にハードルが高いのですが、それが不要です。
    また、会社のPCなど自由にソフトをインストールできない場合でも利用できます。
    プロキシ環境においても[こちらのWEBサイト](https://qiita.com/u-koji/items/c23c06ef594e34655666)にあるような設定を行えば接続できます。

- ユーザー登録が不要です。

    いくつか制限がありますが、サービス利用のハードルを下げたかったため、お試しで使う分には登録不要にしています。

- TCPポート番号を公開できます。

    上に列挙した以外にも類似サービスはあるのですが、HTTP(S)のみサポートしていることが多いです。
    本サービスはそれに加えてTCPにも対応しています。


<!-- ###################################################################### -->


## 最後に

次のようなケースで [TCP Exposer](https://www.tcpexposer.com/) をぜひ使ってみてください。

- 自宅のPCを使ってWEBサービスを公開したい。
- 自宅のPCに外出先からアクセスしたい。
- ローカルPCで開発中のアプリのトライアルなどで一時的に関係者や他アプリからアクセスできるようにしたい。

また、本サービスを開発・運用する際に利用したソフトウェアや他サービスとその選定理由については[こちらのZennの記事](https://zenn.dev/teasy/articles/tcpexposer-tech)に記載しています。
ご興味が出たらチェックしてみてください。
