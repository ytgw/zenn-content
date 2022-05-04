---
title: "Djangoのユーザ認証を試してみた"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "Django"]
published: true
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
usernameとpasswordはPOSTリクエストから取得することを想定しています。

```python
from django.contrib.auth import authenticate, login

def login_view(request):
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


## ToDoリストアプリの紹介
ToDoリストアプリの概要としては下記のとおりです。
- ユーザごとに複数のタスクを登録できます。
- タスクの確認・更新・削除は、登録したユーザのみ行えます。
- ユーザ認証のために、登録・ログイン・ログアウトのページがあります。

### タスクモデルからDjangoユーザモデルを参照する
タスクモデルはDjangoユーザとタスク名のみ属性を持ちます。
タスクモデルからDjangoユーザモデルを参照するには、下記の通り```django.conf.settings.AUTH_USER_MODEL```を外部参照キーにします。
なお、```name```属性をユニークにしているのは、本記事に関係ない事情で設定時の挙動を試したかったからなので、気にしないでください。


```python:todo_list/models.py
from django.conf import settings
from django.db import models


class Task(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    name = models.CharField(max_length=255, unique=True)

    def __str__(self):
        return self.name
```


### タスクのCRUDに関するユーザ認証

#### タスクの読取
第三者にタスクが見られないようにします。
```tasks = Task.objects.filter(user=request.user)```により、ログインユーザに紐づくタスクのみを抽出し、表示関数```render()```に渡しています。

```python:todo_list/views.pyの抜粋
from django.http import HttpRequest, HttpResponse, HttpResponseRedirect
from django.shortcuts import render

from .models import Task


def index(request: HttpRequest) -> HttpResponse:
    if not request.user.is_authenticated:
        return HttpResponseRedirect('login/')

    tasks = Task.objects.filter(user=request.user)
    return render(
        request,
        'todo_list/todo_list.html',
        {'username': request.user.username, 'tasks': tasks}
    )
```

HTMLのテンプレートファイルは下記のとおりです。

```html:todo_list/templates/todo_list/register.html
<h1>ToDo List of {{ username }}</h1>

{% if tasks %}
    <ul>
    {% for task in tasks %}
        <li>
        <div style="display:inline-flex">
            <form action="/task/update/" method="post">
                {% csrf_token %}
                <input type="hidden" name="id" value="{{ task.id }}">
                <input type="text" name="name" value="{{ task.name }}" required>
                <input type="submit" value="update">
            </form>
            <form action="/task/delete/" method="post">
                {% csrf_token %}
                <input type="hidden" name="id" value="{{ task.id }}">
                <input type="submit" value="delete">
            </form>
        </div>
        </li>
    {% endfor %}
    </ul>
{% else %}
    <p>No task is registered.</p>
{% endif %}

<form action="/task/create/" method="post">
    {% csrf_token %}
    New task:
    <input type="text" name="name" required>
    <input type="submit" value="add">
</form>

<form action="/logout/" method="post">
    {% csrf_token %}
    <input type="submit" value="logout">
</form>
```


#### タスクの作成
タスクの作成は、ログインしているユーザのみ可能です。
タスク作成時はHTMLのフォームを通して、POSTリクエストを```create_task()```に渡します。
```create_task()```では最初に```check_task_request()```によって、POST以外のリクエストや、未ログイン時のリクエストを拒否しています(404応答を返します)。
また、POSTリクエスト内の```name```属性をもとにタスクを作成し、最後にタスク表示ページへリダイレクトします。

```python:todo_list/views.pyの抜粋
from django.http import Http404, HttpRequest, HttpResponse, HttpResponseRedirect
from .models import Task


def check_task_request(request: HttpRequest) -> None:
    if request.method != 'POST':
        raise Http404('Request method is not POST.')

    user=request.user
    if not user.is_authenticated:
        raise Http404('User is not authenticated.')


def create_task(request: HttpRequest) -> HttpResponse:
    check_task_request(request=request)

    name = request.POST.get('name')
    if name is None:
        Http404('Task name is not set.')

    task = Task(user=request.user, name=name)
    task.save()
    return HttpResponseRedirect('../../')
```


#### タスクの更新と削除
タスクの更新と削除は、そのタスクを作成したユーザのみ可能です。
タスク更新と削除ははHTMLのフォームを通して、POSTリクエストをそれぞれ```update_task()```と```delete_task()```に渡します。
タスク作成と同じように、最初に```check_task_request()```によって、POST以外のリクエストや、未ログイン時のリクエストを拒否しています(404応答を返します)。
また、POSTリクエスト内の```id```属性をもとに更新・削除したいタスクを特定します。
```if task.user != request.user```の部分でそのタスクに紐づくユーザとログインユーザの一致を確認し、違う場合に404応答を返します。
問題なければ、更新や削除を行いタスク表示ページへリダイレクトします。

```python:todo_list/views.pyの抜粋
from django.http import Http404, HttpRequest, HttpResponse, HttpResponseRedirect
from .models import Task


def check_task_request(request: HttpRequest) -> None:
    if request.method != 'POST':
        raise Http404('Request method is not POST.')

    user=request.user
    if not user.is_authenticated:
        raise Http404('User is not authenticated.')


def update_task(request: HttpRequest) -> HttpResponse:
    check_task_request(request=request)

    id_ = request.POST.get('id')
    name = request.POST.get('name')
    if id is None or name is None:
        Http404('Task id or name is not set.')

    task = Task.objects.get(pk=id_)
    if task.user != request.user:
        raise Http404("Another user's task can't be edited.")

    task.name = name
    task.save()
    return HttpResponseRedirect('../../')


def delete_task(request: HttpRequest) -> HttpResponse:
    check_task_request(request=request)

    id_ = request.POST.get('id')
    if id is None:
        Http404('Task id is not set.')

    task = Task.objects.get(pk=id_)
    if task.user != request.user:
        raise Http404("Another user's task can't be edited.")

    task.delete()
    return HttpResponseRedirect('../../')
```


## まとめ
Djangoのユーザ認証機能を試すために作成したToDoリストアプリを紹介しました。

PythonでWEBアプリ開発というとDjango以外にもFlaskなどが選択肢に入ります。
Flaskの場合データベース連携や認証はプラグインのパッケージを導入する必要が有り、その選定が面倒だということでDjangoを選びました。
実際に使ってみましたが、大きく戸惑うことなく利用できました。

Django自体使ってみて一週間も立っておらず、イマイチなコードになっているかもしれませんので、お気づきになられた方はコメント等いただけると嬉しいです。
