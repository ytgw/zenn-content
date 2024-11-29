---
title: "Azure DevOps/Pipelinesでの自動ビルド環境構築"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure", "cpp", "docker", "githubactions"]
published: false
---

## 概要
Azure DevOps/Pipelinesでの自動ビルド環境構築について具体例を示しながら紹介します。
Azure DevOpsはGitHub、Azure PipelinesはGitHub Actionsのようなサービスです。
Azure DevOps上のGitリポジトリに対してpushされたりPull Requestが発行されたりするのをトリガーに、ビルドやテスト、デプロイなどを自動で実行することができます。
GitHubでなくMicrosoft社のAzure DevOpsなら職場環境上使いやすいという方もいると思います。


## 自動化の内容
この記事ではC++のアプリケーション開発を念頭に、最終的に以下の処理を自動化します。

1. Dockerでのアプリケーションビルド環境の構築
1. アプリケーションのビルド
1. アプリケーションのテスト
1. ビルドした実行ファイルの発行

    WEBブラウザを使って実行ファイルをダウンロードできるようになります。
    また、パイプライン上の別処理からも参照できます。


## パイプラインの初期設定
以下の手順でWEBのGUIから設定ファイルを作成します。

1. 左パネルにあるロケットマークのPipelines
1. 右上のNew pipeline
1. Azure Repos Git(Azure DevOps上にソースコードがある場合)
1. リポジトリ選択
1. Starter pipeline(パイプライン設定ファイルのテンプレートを選択)
1. 右上のSave and Run

デフォルトの設定ファイルのファイル名はazure-pipelines.ymlです。
このファイルに自動実行したい内容を記述します。


## アプリケーションの自動ビルドの実施
まずは、mainブランチへのプッシュをトリガーに以下の内容を実行できるようにします。
1. Dockerでのアプリケーションビルド環境の構築
1. アプリケーションのビルド

この場合は以下のような内容にします。
コメント含めて読んでいただくと内容がわかりやすいと思います。

```yaml:azure-pipelines.yml
# mainブランチにプッシュをトリガーにパイプラインを実行
trigger:
- main

# 実行する仮想マシンのOSを指定
pool:
  vmImage: ubuntu-24.04

# stepsを順番に実行
# リポジトリのルートフォルダで実行されます。
steps:
  # scriptはコンソールで実行したい命令を記述する。
  # インストール済みのソフトを使った内容であれば、基本的にローカル環境のコンソールで実行したものと同じ結果になる。
  # vmImageでubuntuを指定した場合はDockerなどはインストール済み。
  # インストール済みのソフトウェア: https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/hosted?view=azure-devops&tabs=yaml#software
  # 以下のscriptはDockerでのアプリケーションビルド環境の構築に対応する。
- script: docker compose build
  displayName: 'Build Docker Image'

  # アプリケーションのビルド
- script: docker compose run --rm app g++ -o main main.cpp hello.cpp
  displayName: 'Build Application'
```

上記の通りstepsの中に自動実行したい内容をscriptという項目で記述していきます。

その他必要なファイルを列挙しますが、Azure Pipelines自体には関係ないので、ざっと読んでもらっても構いません。

```docker:Dockerfile
FROM ubuntu:24.04

# ビルドツール等のインストール
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y \
    build-essential cmake git

# テストフレームワーク(Google Test)のインストール
RUN git clone https://github.com/google/googletest.git
RUN cd googletest && mkdir build && cd build && cmake .. && make && make install
```

```docker:compose.yaml
services:
  app:
    build: '.'
    volumes:
      - './:/app/'
    working_dir: '/app/'
```

```cpp:hello.cpp
#include <string>
#include "hello.hpp"

std::string hello() {
    return "Hello, World!";
}
```

```cpp:hello.hpp
#pragma once
#include <string>

std::string hello();
```

```cpp:main.cpp
#include <iostream>
#include "hello.hpp"

int main() {
    std::cout << hello() << std::endl;
}
```


## テストの自動実行の実施
先程のアプリケーションのビルドに加えて、テストのビルドと実行も追加してみます。
このとき設定ファイルは以下のようにします。
最初の方はコメントを抜いただけで前と同じ内容です。

```yaml:azure-pipelines.yml
trigger:
- main

pool:
  vmImage: ubuntu-24.04

steps:
- script: docker compose build
  displayName: 'Build Docker Image'

- script: docker compose run --rm app g++ -o main main.cpp hello.cpp
  displayName: 'Build Application'

  # テストの自動実行のために以降を追加
  # テストのビルド
- script: docker compose run --rm app g++ -o test test.cpp hello.cpp -lgtest -lgtest_main
  displayName: 'Build Test'

  # テストの実行
- script: docker compose run --rm app ./test
  displayName: 'Run Test'
```

テストコードは下記のとおりです。

```cpp:test.cpp
#include <gtest/gtest.h>
#include "hello.hpp"

TEST(Hello, hello)
{
  EXPECT_EQ("Hello, World!", hello());
}
```

## ビルドした実行ファイルの発行の実施
今までの処理に加えて、ビルドした実行ファイルを発行します。
これをすることでWEBブラウザからダウンロードできるようになり、実行ファイルを共有しやすくなります。

今まではstepsの中ではscriptのみを使いましたが、ここではtaskを使います。
taskはよく使う機能を切り出したもので多数ありますが、今回はPublishPipelineArtifactを使います。
詳しくは[公式のリファレンス](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/tasks/reference/publish-pipeline-artifact-v1?view=azure-pipelines)を確認ください。

```yaml
trigger:
- main

pool:
  vmImage: ubuntu-24.04

steps:
- script: docker compose build
  displayName: 'Build Docker Image'

- script: docker compose run --rm app g++ -o main main.cpp hello.cpp
  displayName: 'Build Application'

- script: docker compose run --rm app g++ -o test test.cpp hello.cpp -lgtest -lgtest_main
  displayName: 'Build Test'

- script: docker compose run --rm app ./test
  displayName: 'Run Test'

  # ビルドファイルの発行タスクを追加
- task: PublishPipelineArtifact@1
  displayName: 'Publish Build File'
  inputs:
    # 発行したいファイルやディレクトリのパス
    targetPath: main
    # ダウンロード等での識別用の名前
    artifact: BuildFile
```


## まとめ
Azure DevOps/Pipelinesでの自動ビルド環境構築について紹介しました。
azure-pipelines.ymlという設定ファイルに自動で実行したい内容を書くことで実現できます。
紹介しませんでしたが、プルリクエスト前にパイプラインを実行させることももちろんできます。

未来の自分や皆さんの参考になれば幸いです。