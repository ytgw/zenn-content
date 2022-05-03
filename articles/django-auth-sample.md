---
title: "Djangoのユーザ認証を試してみた"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "Django"]
published: false
---


## 概要
PythonのフルスタックWEBフレームワークのDjangoのユーザ認証機能を試したくて、簡単なToDoリストを作ってみましたので、それを紹介します。
本記事の前提知識は、[Djangoのチュートリアル](https://docs.djangoproject.com/ja/4.0/intro/tutorial01/)に相当する内容です。
また、Djangoのユーザ認証機能については、[公式のドキュメント](https://docs.djangoproject.com/ja/4.0/topics/auth/)を参考にしています。
なお、作成したアプリは[GitHubのリポジトリ](https://github.com/ytgw/django-auth-sample)に置いています。

アプリの概要としては下記のとおりです。
- ユーザごとに複数のタスクを登録できます。
- タスクの確認・更新・削除は、登録したユーザのみ行えます。
- ユーザ認証のために、登録・ログイン・ログアウトのページがあります。

タスクの編集画面は、htmlやcssを工夫していないので分かりにくいですが、下記の画像のとおりです。
![](/images/django-auth-sample/task_manage_page.png)


## Djangoの認証機能
### ユーザの作成
django.contrib.auth.models.UserがDjangoのユーザモデルで、主要な属性は以下のとおりです。
- username
- password
- email

なお、usernameとpasswordは必須項目です。

ユーザを作成するには下記の通り、```create_user()```関数を使います。

```python
from django.contrib.auth.models import User
user = User.objects.create_user('username', 'foo@example.com', 'password')
```


### ユーザ認証
DjangoはHTTPリクエストに対して認証の有無を判別します。
リクエスト```request```には現在のユーザーを示す```request.user```属性が付与されています。
もしユーザーが現在ログインしていない場合、この属性には```AnonymousUser```のインスタンスが、ログインしている場合は```User```のインスタンスがセットされます。
この二者は```is_authenticated```を用いて次のように識別する事ができます。

```python
if request.user.is_authenticated:
    # Do something for authenticated users.
    ...
else:
    # Do something for anonymous users.
    ...
```

また、ログインしているユーザが、あるユーザ```foo_user```かどうか識別するには下記のようにします。
```python
if request.user == foo_user:
    # Do something for foo_user.
    ...
```


### ログイン
ログインするには```login()```関数を使います。

```python
from django.contrib.auth import authenticate, login

def my_view(request):
    username = request.POST['username']
    password = request.POST['password']
    user = authenticate(request, username=username, password=password)
    if user is not None:
        login(request, user)
        # Redirect to a success page.
        ...
    else:
        # Return an 'invalid login' error message.
        ...
```

### ログアウト
ログアウトするには```logout()```関数を使います。

```python
from django.contrib.auth import logout

def logout_view(request):
    logout(request)
    # Redirect to a success page.
```
