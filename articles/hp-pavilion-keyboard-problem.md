---
title: "Ryzen CPU搭載のUbuntuノートPCでのキーボードトラブル"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ryzen", "ubuntu"]
published: true
---

## 概要

Ryzen CPU搭載ノートPCにUbuntuをインストールすると、キーボードの挙動がおかしいことに気づきました。
具体的には、キーを長押しするとキーを離した後もずっと同じものが入力されたり、キーを押してから入力されるまでの時間が長かったりしました。

最終的にはLinuxのカーネルを変更することで解決しましたので、その内容をまとめました。


## 環境情報

- ハードウェア: HP Pavilion Aero 13-be2000
- CPU: AMD Ryzen 7 7735U with Radeon Graphics
- OS: Ubuntu 22.04.3 LTS
- Linuxカーネルバージョン(トラブル発生時): 6.2
- 内蔵キーボードで発生し、USB接続の外付けキーボードでは発生しない


## 解決方法

[HP Pavilion Laptop Keyboard Issue in Ubuntu 22.04というWEBページ](https://askubuntu.com/questions/1476206/hp-pavilion-laptop-keyboard-issue-in-ubuntu-22-04)や[こちら](https://bbs.archlinux.org/viewtopic.php?id=285327)などが参考になりました。

Linuxカーネル5.19.10から「最新のPCではIRQオーバーライドは必要ない」という判断で変更した箇所が原因のようです。
カーネルのバージョンを5.19.9以前(試してはいません)か、もしくは6.5以降にすることで解決するようです。

以下のコマンドでカーネルのバージョンを変更し、再起動後問題が発生しないことを確認しました。

```bash
# 現在のカーネルバージョンの確認
uname -r  # 発生時は6.2.0-39-generic

# カーネルのアップデート
# アップデートするバージョンはapt search linux-image | grep genericなどで探す
sudo apt install --install-recommends --install-suggests linux-image-6.5.0-14-generic

# 再起動
sudo reboot now

# 現在のカーネルバージョンの確認
uname -r  # 6.5.0-14-generic
```
