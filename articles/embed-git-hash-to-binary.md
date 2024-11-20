---
title: "C/C++の実行ファイルにGitのコミットハッシュ値を埋め込む"
emoji: "©"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cpp", "c", "git", "makefile"]
published: true
---

## 概要
C/C++の実行ファイルにGitのコミットハッシュ値を埋め込む方法を紹介します。
本当に実行ファイルが更新されたかを確認したいときに役立つと思います。
自動ビルドのパイプラインに組み込んでも良いかもしれません。


## 方法
ビルド時にGitのハッシュ値をDオプションで渡すことで埋め込みます。

C++での簡単な例を以下に示します。

```cpp:main.cpp
#include <iostream>

#ifndef GIT_HASH
#define GIT_HASH "unknown"
#endif

int main() {
    // GIT_HASHがビルド時に定義されていればそれを、されていなければunknownを表示する
    std::cout << "GIT_HASH: " << GIT_HASH << std::endl;
}
```

ビルド時にGIT_HASHを定義することで、実際のハッシュ値を埋め込むことができます。例えば以下のコマンドでビルドします。

```bash
# 文字列(この例ではfoo)をGIT_HASHに埋め込む
g++ -D GIT_HASH=\"foo\" -o main main.cpp

# コマンドの結果を呼び出してGIT_HASHに埋め込む
g++ -D GIT_HASH=\"$(echo bar)\" -o main main.cpp

# gitのcommit hashをGIT_HASHに埋め込む(example: 007697f08129d72c6ef041e02ea4ca88715a7d5d)
g++ -D GIT_HASH=\"$(git show --no-patch --format='%H')\" -o main main.cpp

# gitのabbreviated commit hashをGIT_HASHに埋め込む(example: 007697f)
g++ -D GIT_HASH=\"$(git show --no-patch --format='%h')\" -o main main.cpp
```

Makefileでもコマンド実行結果を変数に格納できるので類似の方法でハッシュを埋め込むことができます。
CMakeでは試したことがありませんが、おそらくできると思います。

```makefile:Makefile
GIT_HASH = "$(shell git show --no-patch --format='%H')"

build: main.cpp
	$(CXX) -D GIT_HASH=\"$(GIT_HASH)\" -o main $^
```
