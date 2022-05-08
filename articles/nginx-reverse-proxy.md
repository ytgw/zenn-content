---
title: "NGINXのリバースプロキシ機能を使う"
emoji: "🤖"
type: "tech"
topics: ["nginx", "リバースプロキシ", "Docker"]
published: true
---


## 概要
WEBアプリを本番環境デプロイのため、NGINXのリバースプロキシについて調べた内容を紹介します。
一番シンプルな設定では、下記内容のファイルを/etc/nginx/conf.d/foo.confに保存すれば良いみたいです。
この設定では、クライアントからfoo.example.com宛のリクエストをNGINXが受け取った際に、[http://localhost:5000/](http://localhost:5000/)に転送します。
ソースコード等は[GitHub](https://github.com/ytgw/reverse-proxy-sample)に置いています。

```
server {
    server_name  foo.example.com;
    location / {
        proxy_pass http://localhost:5000/;
    }
}
```

## 固定のサブドメインに転送する設定
下記の構成でWEBブラウザから http://localhost/ にアクセスするとNGINXのトップページが表示され、 http://flask.localhost/ にアクセスするとFlaskで作ったサンプルWEBアプリが表示されます。

### 全体構成
環境全体はDocker Composeの構成で試しており、下記のようなネットワーク構成になっています。

![](/images/nginx-reverse-proxy/network-structure.png)

また、docker-compose.ymlは下記のとおりです。

```:docker-compose.yml
version: '3'
services:
  flask:
    build: ./flask/
  nginx:
    build: ./nginx/
    ports:
     - "80:80"
```

### nginxサービス
NGINXの大元の設定ファイルは/etc/nginx/nginx.confに```include /etc/nginx/conf.d/*.conf;```という記述があります。
そのため、NGINXの構成としては下記の通りで、flask.localhostに来た通信を http://flask:5000/ に転送する設定ファイルを/etc/nginx/conf.d/に保存しただけです。

```docker:nginx/Dockerfile
FROM nginx
COPY localhost.conf /etc/nginx/conf.d/
```

```nginx:nginx/localhost.conf
server {
    server_name  flask.localhost;
    location / {
        # WEBリクエストをリダイレクト
        proxy_pass http://flask:5000/;
    }
}
```

### flaskサービス(WEBアプリサーバー)
Pythonのflaskというパッケージを使って、トップページがアクセスされたときにHello Flask!という文字を表示するだけのWEBアプリサーバーを構築しました。

```docker:flask/Dockerfile
FROM python
ADD . /code
WORKDIR /code
RUN pip install flask
CMD ["python", "app.py"]
```

```python:flask/app.py
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello Flask!'

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)
```

## ワイルドカードサブドメインに転送する設定
ワイルドカードサブドメインに転送する方法は下記のとおりです。
下記の構成でWEBブラウザから http://localhost/ にアクセスするとNGINXのトップページが表示され、 http://flask.localhost/ にアクセスするとFlaskで作ったサンプルWEBアプリが表示され、 http://foo.localhost/ (fooは任意の文字列)にアクセスすると別の画面(pythonサービスの8080ポートに対応する画面)が表示されます。

```nginx:nginx/localhost.conf
server {
    server_name  flask.localhost;
    location / {
        # WEBリクエストをリダイレクト
        proxy_pass http://flask:5000/;
    }
}

server {
    server_name  *.localhost;
    location / {
        # WEBリクエストをリダイレクト
        proxy_pass http://python:8080/;
    }
}
```

```:docker-compose.yml
version: '3'
services:
  flask:
    build: ./flask/
  python:
    build: ./python/
  nginx:
    build: ./nginx/
    ports:
     - "80:80"
```

## まとめ
NGINXのリバースプロキシについて調べた内容を紹介しました。

今回は紹介しませんでしたが、転送先がリクエスト情報を知るためには、HTTPのヘッダーも設定したほうが良さそうです。
詳しくは [NGINX リバースプロキシ(有志の公式ドキュメントの日本語訳と思われる)](http://mogile.web.fc2.com/nginx/admin-guide/reverse-proxy.html)の「リクエストヘッダを渡す」という部分や、[こちらのはてなブロク](https://christina04.hatenablog.com/entry/2016/10/25/190000)を確認してみてください。
