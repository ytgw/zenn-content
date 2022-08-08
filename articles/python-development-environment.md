---
title: "Python開発環境の備忘録"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "black", "flake8", "mypy", "vscode"]
published: false
---


## 概要

Python開発環境、主にフォーマッター、リンター、型チェックについての備忘録です。
類似の記事も多数あり何番煎じかも分かりませんが、自分でまとめないと毎回ツールや設定方法を検索する羽目になりそうなので、未来の自分のためにメモを残します。


<!-- ---------------------------------------------------------------------- -->


## フォーマッター

フォーマッターを導入した理由は、コーディングスタイルで悩みたくないからです。
私は細かい部分を修正したくなる神経質な面があります。
その割に好みのスタイルが数カ月で変わり、過去書いたソースコードをその時の気分で直したくなり、スタイルを混在させてしまうことがあります。
他にも時間を置いて機能を追加する場合などは、元々のものを忘れ違うスタイルで書いてしまうことがあります。
また、チームで開発する際にコーディングスタイルを統一するのに役立ちます。
そういった理由からフォーマッターを導入しました。

導入したフォーマッターはblackとisortです。


<!-- ---------------------------------------------------------------------- -->


### [black](https://black.readthedocs.io/en/stable/index.html)

上記のような理由があって、カスタマイズできる項目が少なく制約の強いblackを導入しました。
正直気に入らないルールもありますが、主観的で気分的な理由がほとんどす。
また、チーム開発する際に好みのスタイルの押し付け合いになるのも不毛なので、blackのデフォルトに統一することにします。

下記コマンドでインストールできます。

```bash
pip install black
```

また、下記コマンドで指定のファイルおよびディレクトリ内のファイルをフォーマットします。

```bash
black {source_file_or_directory}
```

[公式ドキュメント](https://black.readthedocs.io/en/stable/usage_and_configuration/file_collection_and_discovery.html#gitignore)によると、--excludeオプションが設定されていない場合は.gitignoreで指定されているファイルはフォーマットしません。
また、.gitignoreで指定されているものに追加してフォーマットしたくないファイルがある場合は、--extend-excludeオプションで指定します。
フォーマット対象外設定も含めて、設定内容はコマンドラインで実行時にも指定できますが、pyproject.tomlに書くこともできます。

```toml:pyproject.toml
[tool.black]
# 公式ドキュメント
# https://black.readthedocs.io/en/stable/usage_and_configuration/the_basics.html#configuration-format

# ライン行数はデフォルトから変更する場合はコメントアウト
# line-length = 88

# デフォルトに追加したいフォーマット対象外指定ファイルやディレクトリ
extend-exclude = '''
(
      \.git
    | migrations/  # 自動生成されたDB migrationファイル
)
'''
```


<!-- ---------------------------------------------------------------------- -->


### [isort](https://pycqa.github.io/isort/)

isortはインポートをアルファベット順にソートするフォーマッターです。
ライブラリのソートはblackでは行わないため、isortを導入しました。

下記コマンドでインストールできます。

```bash
pip install isort
```

また、下記コマンドで指定のファイルおよびディレクトリ内のファイルをフォーマットします。

```bash
isort {source_file_or_directory}
```

設定はpyproject.tomlに記載します。
blackと共存して使う場合は設定ファイルpyproject.tomlにプロファイル設定を書きます。

```toml:pyproject.toml
[tool.isort]
# blackと共存して使う設定
# https://pycqa.github.io/isort/docs/configuration/black_compatibility.html
profile = "black"

# .gitignoreファイルで指定されているファイルを除外
skip_gitignore = true

# デフォルトに追加したいフォーマット対象外指定ファイルやディレクトリ
extend_skip_glob = ["**/migrations/*"]  # 自動生成されたDB migrationファイル
```


<!-- ---------------------------------------------------------------------- -->


## リンター

フォーマッターでは直せない問題を検知するためにリンターを導入します。
リンターはflake8を選びました。


<!-- ---------------------------------------------------------------------- -->


### [flake8](https://flake8.pycqa.org/en/latest/)

flake8は下記コマンドでインストールできます。

```bash
pip install flake8
```

また、下記コマンドで指定のファイルおよびディレクトリ内のファイルをチェックします。

```bash
flake8 {source_file_or_directory}
```

設定はsetup.cfgに記載します。

```:setup.cfg
[flake8]
# https://flake8.pycqa.org/en/latest/user/configuration.html

# 最大行数はblackに合わせる
max-line-length = 88

# デフォルトに追加したいフォーマット対象外指定ファイルやディレクトリ
# https://flake8.pycqa.org/en/latest/user/options.html#cmdoption-flake8-exclude
# flake8には.gitignoreファイルで指定されているファイルを一括で除外するオプションはない
# デフォルトは.svn,CVS,.bzr,.hg,.git,__pycache__,.tox,.nox,.eggs,*.egg
extend-exclude = **/migrations/*,venv/,.venv/

# 無視するルールの設定
# https://flake8.pycqa.org/en/latest/user/options.html#cmdoption-flake8-ignore
# blackとの競合を避けるための設定
# Line too long (82 > 79 characters) (E501)
# Line break occurred before a binary operator (W503)
ignore = E501,W503
```


<!-- ---------------------------------------------------------------------- -->


## 型チェック

型ヒントを使いながらコーディングすることで、メソッドが補完されたり、コードの意図を伝えたりできるため、私は意識して使っています。
型ヒントを使っていない箇所や、型と合っていない操作をしている箇所などをチェックするツールとして、私はmypyを導入しました。
VSCodeで開発している場合Microsoftが開発しているPylanceの型チェック機能を使う人が多いかもしれません。
しかし、例えばGitのHookやCI、Djangoなどフレームワークのmypyプラグインなどの他ツールとの連携を考えると、Pythonの公式が開発しているmypyが良さそうです。


<!-- ---------------------------------------------------------------------- -->


### [mypy](https://mypy.readthedocs.io/en/stable/index.html)

mypyは下記コマンドでインストールできます。

```bash
pip install mypy
```

また、下記コマンドで指定のファイルおよびディレクトリ内のファイルをチェックします。

```bash
mypy {source_file_or_directory}
```

設定はpyproject.tomlに記載します。

```toml:pyproject.toml
[tool.mypy]
# https://mypy.readthedocs.io/en/stable/config_file.html#example-pyproject-toml

# strictモードを有効化
strict = true

# プラグイン設定
# plugins = ["mypy_django_plugin.main"]

# mypyのDjangoプラグインの設定
# [tool.django-stubs]
# https://github.com/typeddjango/django-stubs
# django_settings_module = "mysite.settings"
```


<!-- ---------------------------------------------------------------------- -->


## 外部ツール連携

### Visual Studio Code

VSCodeのユーザーもしくはワークスペースの設定に以下を記載します。
挙動を見た感じだとCLI用の設定ファイルの設定を読み込んでいるようです。

```json:settings.json
{
    "[python]": {
        // 保存時にblackでフォーマットするが、blackのexclude設定は無視される
        // https://stackoverflow.com/questions/70199494/how-to-make-vscode-honor-black-excluded-files-in-pyproject-toml-configuration-wh
        "editor.formatOnSave": true,
        // 保存時にisortでフォーマットする
        "editor.codeActionsOnSave": {
            "source.organizeImports": true
        }
    },
    "python.analysis.typeCheckingMode": "off",  // mypyを使うのでVSCode本体の型チェックは無効化
    "python.formatting.provider": "black",
    "python.linting.flake8Enabled": true,
    "python.linting.mypyEnabled": true,
}
```


<!-- ---------------------------------------------------------------------- -->


### Git
