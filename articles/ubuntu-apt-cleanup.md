---
title: "Ubuntuのapt環境の整理整頓"
emoji: "🧹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ubuntu", "apt"]
published: false
---

## はじめに
最近Ubuntuを最新のLTSバージョンにアップグレードしました。
その際に、aptのサードパーティのsources.listが無効になり、再度有効にする必要がでてきました。
いい機会なので使っていないものは削除して整理整頓することにしましたので、その時の作業内容を備忘録として残します。


## 現状の確認

下記コマンドでapt周りの現状を確認できます。

```bash
# aptでインストールしたパッケージの確認
apt list --installed
# 末尾が[installed]となっているものは明示的にインストールしたパッケージ
# 末尾が[installed,automatic]となっているものは、依存関係で暗黙的にインストールしたパッケージ
# 末尾が[installed,local]となっているものは、sources.listに含まれていないが過去にインストールしたパッケージ

# 現在の有効なsources.listの確認
apt-add-repository --list
# /etc/apt/sources.listと/etc/apt/sources.list.d/foo.listの内容を読み込んでいる
# GUIアプリケーションの「ソフトウェアとアップデート」のUbuntuのソフトウェアと他のソフトウェアからでも確認できる

# aptでのインストールや更新時の認証キーの確認
apt-key list
# apt-keyコマンドは非推奨
# /etc/apt/trusted.gpgと/etc/apt/trusted.gpg.d/foo.gpgの内容を読み込んでいる
# GUIアプリケーションの「ソフトウェアとアップデート」の認証からでも確認できる
```

/etc/apt/sources.listと/etc/apt/sources.list.d/foo.listを確認し、以下のように分類しました。

- 今後もaptで更新するもの
    - Docker
    - Dropbox
    - Google Chrome
    - Firefox ESR
    - NVIDIA Container Toolkit(NVIDIA Docker)
    - NVIDIA Driver
    - Visual Studio Code

- 今後はsnapで更新するもの
    snapだとaptと比較して早く最新のものを試すことができます。
    以下は最新版を使いたいがためにサードパーティのソースリストを使っていました。
    しかし、これらはsnapからもインストールできるので、今後はそうします。
    なお、DockerやVisual Studio Codeもsnapにありました。
    しかしDockerはホームディレクトリにしかアクセスできない、VSCodeは日本語入力できないなど、snap特有の問題があるので、aptからインストールします。
    - Golang
    - Node.js

- 廃棄するもの
    - Robot Operating System

aptの関連の設定は/etc/apt以下に保存されています。
操作を間違った場合に復旧するため、そのディレクトリをコピーしておきます。


## 不要なソフトや設定の削除

ソフト本体は```sudo apt remove foo```で削除します。

sources.listの削除は、/etc/apt/sources.listの該当する行の削除と、/etc/apt/sources.list.d/foo.listのファイル削除を行うことで実施できます。
アップグレードした場合だと、前のバージョン用のURLで残っています。
その場合は一旦sources.listから削除し、後で記載するように再度インストールすることで有効化します。

認証キーの削除には```sudo apt-key del <keyid>```で行います。
<keyid>は```apt-key list```で確認できるハッシュ値のような文字列の下8桁を指定します。


## 再設定と再インストール

### 今後もaptで更新するもの

以下のソフトウェアは今後もaptで更新していきます。

- Docker

    ```bash
    # https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository
    sudo mkdir -m 0755 -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

    echo \
        "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
        "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
        sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt update
    sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```

- Dropbox

    ```bash
    # 公式サイトから.debファイルをダウンロード
    # 再度インストールすることで有効化
    sudo apt install ~/Downloads/dropbox_foo.deb
    ```

- Google Chrome

    ```bash
    # なぜか有効なままだった
    sudo apt update
    sudo apt install google-chrome-stable
    ```

- Firefox ESR

    ```bash
    # https://gihyo.jp/admin/serial/01/ubuntu-recipe/0710#sec4
    sudo add-apt-repository ppa:mozillateam/ppa

    sudo apt update
    sudo apt install firefox-esr firefox-esr-locale-ja
    ```

- NVIDIA Container Toolkit

    ```bash
    # https://github.com/NVIDIA/nvidia-docker
    # https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker
    distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
        && curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
        && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
                sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
                sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

    sudo apt update
    sudo apt install nvidia-container-toolkit
    ```

- NVIDIA Driver

    ```bash
    # https://qiita.com/porizou1/items/74d8264d6381ee2941bd
    sudo add-apt-repository ppa:graphics-drivers/ppa

    # インストール可能なバージョンを表示する
    ubuntu-drivers devices

    # 上記コマンドで推奨となっているバージョンをインストール
    sudo apt update
    sudo apt install nvidia-driver-foo
    ```

- Visual Studio

    ```bash
    # 公式サイトから.debファイルをダウンロード
    # 再度インストールすることで有効化
    sudo apt install ~/Downloads/code_foo.deb
    ```

### 今後はsnapで更新するもの

- Golang

    ```bash
    sudo apt remove golang-go
    sudo snap install --classic go
    ```

- Node.js

    ```bash
    sudo apt remove nodejs
    sudo snap install --classic node
    ```

## まとめ

Ubuntuのバージョンアップの際に行ったapt周りの整理整頓の作業内容を備忘録として残しました。
未来の自分や皆さんの参考になれば幸いです。
