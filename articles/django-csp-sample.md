---
title: "DjangoでのContent-Security-Policy設定方法"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Django", "ContentSecurityPolicy", "CSP", "セキュリティ", "Python"]
published: false
---

## はじめに
年末年始の休暇に [体系的に学ぶ 安全なWebアプリケーションの作り方](https://www.amazon.co.jp/%E4%BD%93%E7%B3%BB%E7%9A%84%E3%81%AB%E5%AD%A6%E3%81%B6-%E5%AE%89%E5%85%A8%E3%81%AAWeb%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%AE%E4%BD%9C%E3%82%8A%E6%96%B9-%E7%AC%AC2%E7%89%88-%E8%84%86%E5%BC%B1%E6%80%A7%E3%81%8C%E7%94%9F%E3%81%BE%E3%82%8C%E3%82%8B%E5%8E%9F%E7%90%86%E3%81%A8%E5%AF%BE%E7%AD%96%E3%81%AE%E5%AE%9F%E8%B7%B5-%E5%BE%B3%E4%B8%B8/dp/4797393165) という本を読みました。
その中で [OWASP ZAP](https://www.zaproxy.org/) という脆弱性診断ツールの使い方が紹介されていました。

そこで、以前開発したサービスに対してOWASP ZAPでの脆弱性診断を行ったところ、Medium以上のリスクが一つ見つかり、それはContent-Security-Policy(CSP)のヘッダが未設定というものでした。
そのサービスはDjangoを使って開発したので、それ向けのCSP設定ツールであるDjango-CSPを導入し、見つけたリスクに対処しました。

以前開発したものは [TCP Exposer](https://www.tcpexposer.com/) というサービスで、プライベートIPアドレスしか持たないローカルサーバーに、インターネットからアクセスできるようにします。
こちらもぜひ使ってください。


## [Content-Security-Policy(CSP)](https://developer.mozilla.org/ja/docs/Web/HTTP/CSP) とは
以下MDNからの抜粋です。

> コンテンツセキュリティポリシー (CSP) は、クロスサイトスクリプティング (Cross-site_scripting) やデータインジェクション攻撃などのような、特定の種類の攻撃を検知し、影響を軽減するために追加できるセキュリティレイヤーです。 これらの攻撃はデータの窃取からサイトの改ざん、マルウェアの拡散に至るまで、様々な目的に用いられます。
>
> (中略)
>
> CSP を有効にするには、ウェブサーバーから Content-Security-Policy HTTP ヘッダーを返すように設定する必要があります（X-Content-Security-Policy ヘッダーに関する記述が時々ありますが、これは古いバージョンのものであり、今日このヘッダーを指定する必要はありません）。

サイトの管理者は、コンテンツ(JavaScript, CSS, 画像など)ごとに信頼するドメインを設定します。
ポリシーに合わないコンテンツの場合は、ダウンロードだったり、表示だったり、実行だったりをブラウザは実施しません。
これによって、WEBサイトの管理者が意図していないコンテンツを使った攻撃を防ぎます。

例えば、任意のドメインからの画像読み込みを許可し、スクリプトは信頼するドメインを許可し、それ以外のコンテンツはサイト自身のドメイン（サブドメインを除く）のみを許可したい場合は以下のように設定します。
```
Content-Security-Policy: default-src 'self'; img-src *; script-src foo.example.com
```

CSPヘッダーの説明は[MDNのドキュメント](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Content-Security-Policy)を参照ください。


## DjangoでのCSP設定
Django-CSPというDjango向けミドルウェアを使ってCSPを設定しました。
[ドキュメント](https://django-csp.readthedocs.io/en/latest/)の量は多くないので、導入を検討する際には一通り目を通すと良いと思います。


### Django-CSPの選定理由
選定理由は以下のとおりです。
- 開発したサービスはDjangoというフレームワークを使っていた。
- Django CSP というキーワードで検索すると、Django-CSPばかり言及されていた。
- GitHubを見るとFirefoxの開発元であるMozillaが開発している。


### 使い方
pipを使って以下のようにインストールします。

```
pip install django-csp
```

設定するには、Djangoプロジェクトのsettings.pyに以下のような記述をします。
```import django-csp``` などは不要です。

```python:settings.py
# ...
MIDDLEWARE = [
    # ...
    "csp.middleware.CSPMiddleware",
    # ...
]
# ...
# 以下の設定は例なので、自分の環境に合わせて変更必要
CSP_DEFAULT_SRC = ("'self'", "www.google-analytics.com")
CSP_SCRIPT_SRC = ("'self'", "www.googletagmanager.com")
CSP_INCLUDE_NONCE_IN = ("script-src",)

# ポリシーの効果を監視する (ただし強制はしない) ことによりポリシーを試行する場合はTrue。デフォルトはFalse
# CSP_REPORT_ONLY = True
```

CSP_DEFAULT_SRCやCSP_SCRIPT_SRCはそれぞれContent-Security-Policyヘッダーのdefault-srcとscript-srcに対応します。

```CSP_INCLUDE_NONCE_IN = ("script-src",)```の設定は、インラインスクリプト(```<script>```タグ内に直接書くJavaScriptコード)を書く際に使用します。
例えば以下のようにnonceを設定しなければ、直接書いたスクリプトは実行されません。

```html:template/foo.html
<script nonce="{{request.csp_nonce}}">
    var hello="world";
</script>
```

詳しくは[公式ドキュメント](https://django-csp.readthedocs.io/en/latest/nonce.html)を参照ください。

CSP_DEFAULT_SRCのwww.google-analytics.com、CSP_SCRIPT_SRCのwww.googletagmanager.com、CSP_INCLUDE_NONCE_INの設定はGoogleAnalyticsのために設定しました。

## まとめ
自作したサービスに対して、セキュリティ向上のため、Django-CSPを使ってContent-Security-Policyを設定しました。
