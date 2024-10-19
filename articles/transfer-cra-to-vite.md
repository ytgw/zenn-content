---
title: "Create React AppからViteへの移行"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "vite", "createreactapp"]
published: true
---

## 概要

昔開発した[Source Code Comment Translator](https://ytgw.github.io/CodeCommentTranslator/)というSPAはCreate React App(CRA)をベースに開発しました。
しかし、CRAの開発は停止しており、依存ライブラリも脆弱性がある状況で、GitHubからアラートが来ていました。
そこで、CRAからViteというフロントエンドビルドツールに移行しました。
本記事では移行に対して実施した内容を紹介します。


<!-- ---------------------------------------------------------------------- -->


## 実施内容
ここではざっくりと説明します。
詳細は[2024年10月18日から19日にかけてのGitHubのコミット履歴](https://github.com/ytgw/CodeCommentTranslator/commits/master/?since=2024-10-18&until=2024-10-19)を参照ください。

移行先としてViteを選んだ理由は、ときどきCRAからの移行先として紹介されるWEBサイトを見かけたからです。
また、[Reactの公式サイト](https://ja.react.dev/learn/start-a-new-react-project)にて、フレームワークなしで React を使う場合のときのツールとして少しだけ紹介されていました。

また、移行時には、[こちらの記事](https://zenn.dev/akineko/articles/765f8388e84c06)と[Viteの公式サイト](https://ja.vite.dev/guide/)を主に参考しました。


### Viteのインストール
以下のコマンドでViteをインストールしました。
エラーで入らなかったので、forceオプションをつけています。

```bash
npm install -D --force vite
```


### index.htmlの修正
主に以下を実施しました。
- index.htmlのディレクトリを移動

    CRAではpublic以下にエントリーポイントであるindex.htmlを配置します。
    Viteではプロジェクトルートに配置しますので、単に移動します。

- index.htmlにscriptタグを追記

    index.html内に```<script type="module" src="/src/index.tsx"></script>```を追記しました。

詳細は[こちら](https://ja.vite.dev/guide/#index-html-and-project-root)を参照ください。

このタイミングで```npx vite```を実行して開発サーバーを起動し、Vite上で問題なくアプリが動作するか確認しました。


### CRAのeject
CRAを削除する前にejectを行います。
CRAで作成した際のREADMEに書いてあるとおり```npm run eject```を実行します。
実行すると、いろいろなファイルが作成され、package.jsonが変更されます。


### よく使うコマンドをVite用に修正
package.jsonのscriptsをVite用に変更し、よく使うコマンドを変更・追加します。
[こちら](https://zenn.dev/akineko/articles/765f8388e84c06#vite-%E3%81%B8%E3%81%AE%E7%A7%BB%E8%A1%8C%E6%89%8B%E9%A0%86)によると、Vite は build 時に型検査を行わないので build 前に tsc --noEmit を追加して別途型検査を行うようにしています。

```json:package.json抜粋
  "scripts": {
    ...
    "start": "vite",
    "build": "tsc --noEmit && vite build",
    "preview": "vite preview",
    "test": "jest"
  },
```

[Viteの公式サイトのコマンドラインインターフェイス](https://ja.vite.dev/guide/#%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%88%E3%82%99%E3%83%A9%E3%82%A4%E3%83%B3%E3%82%A4%E3%83%B3%E3%82%BF%E3%83%BC%E3%83%95%E3%82%A7%E3%82%A4%E3%82%B9)を参考にしました。


### 不要なパッケージの削除
変更前後で以下のコマンドが成功するか確認し、アプリが壊れていないか確認しながら進めました。

```bash
npx eslint --fix src/  # リント
npm test               # テスト
npm run build          # プロダクション用のビルド
npm run preview        # プロダクション用ビルドをローカルでプレビュー
npm start              # 開発サーバーを起動
```

```npm uninstall <パッケージ名>```のコマンドで削除します。
依存関係のエラーが出た場合は削除する順番を変えたりしています。
今回は以下の順番で削除しました。

1. @types/node
1. webpack, tailwindcss
1. [webpack関連パッケージ](https://github.com/ytgw/CodeCommentTranslator/commit/89a5bad1f52cedfc31f0b87cbdf0d7c3b8086ac2)
1. bfj, browserslist, camelcase, css-loader, dotenv, dotenv-expand, file-loader, fs-extra, identity-obj-proxy, mini-css-extract-plugin
1. [postcss関連パッケージ](https://github.com/ytgw/CodeCommentTranslator/commit/84448e0b50889922281aad674785a55e05564475)
1. prompts, resolve, resolve-url-loader, sass-loader, semver, source-map-loader, style-loader, web-vitals


### 不要な設定やファイルの削除
package.jsonからbrowserslistとの項目を削除しました。

また、ejectで作成されたけど不要なファイルとして以下のファイルを削除しました。
- config/env.js
- config/getHttpsConfig.js
- config/modules.js
- config/paths.js
- config/webpack/persistentCache/createEnvironmentHash.js
- config/webpack.config.js
- config/webpackDevServer.config.js

不要かどうかは以下のコマンドが成功するかどうかで判断しています。
```bash
npx eslint --fix src/  # リント
npm test               # テスト
npm run build          # プロダクション用のビルド
npm run preview        # プロダクション用ビルドをローカルでプレビュー
npm start              # 開発サーバーを起動
```


### パッケージのメジャーバージョン更新
Vite移行と直接関係ないですが、いい機会だったので利用しているパッケージのバージョンアップしました。

```npm outdated```で更新があるパッケージを表示し、以下の順番で更新しました。

- [react関連パッケージ](https://github.com/ytgw/CodeCommentTranslator/commit/95ed78c724e01fd0994191dceb994a480cc8ba95#diff-7ae45ad102eab3b6d7e7896acd08c427a9b25b346470d7bc6507b6481575d519L33)
- [テスト関連パッケージ](https://github.com/ytgw/CodeCommentTranslator/commit/113b6614339bf64294f0cc236303472e7c22cc69)
- babel-loader
- [eslint関連パッケージ](https://github.com/ytgw/CodeCommentTranslator/commit/6a26d71b588d340486dc09faaef50dd5a3a8b6f5)

    警告やエラーが発生したので、都度メッセージを見て設定ファイルの形式を変更したり、新しいパッケージを追加したりしました。

- typescript


### GitHub Pages関連の修正
GitHub Pagesにデプロイしているので、それに関する変更をしました。

- package.jsonの修正

    CRAはbuildディレクトリにプロダクション用のビルドファイルを出力しますが、Viteはdistディレクトリに出力するので、それに合わせて修正します。

    ```diff json:package.json
    "scripts": {
       ...
    -  "deploy": "gh-pages -d build",
    +  "deploy": "gh-pages -d dist",
       ...
    },
    ```

- Viteの設定にbaseを追加

    デプロイ先がルートディレクトリ(http://ytgw.github.io/)でなく、サブディレクトリ以下なので(http://ytgw.github.io/CodeCommentTranslator/)を追加します。

    ```js:vite.config.mjs
    import { defineConfig } from "vite";

    export default defineConfig({
        base: "/CodeCommentTranslator/"
    })
    ```


<!-- ---------------------------------------------------------------------- -->


## まとめ
Create React App(CRA)をベースに開発したSPAをViteに移行したときに実施した内容を備忘録を兼ねて紹介しました。
未来の自分や皆さんの参考になれば幸いです。


<!-- ---------------------------------------------------------------------- -->


## 参考サイト
- [CRAからViteへの移行記事](https://zenn.dev/akineko/articles/765f8388e84c06)
- [Viteの公式サイト](https://ja.vite.dev/guide/)
- [今回移行したアプリ](https://ytgw.github.io/CodeCommentTranslator/)
- [移行時のコミット履歴](https://github.com/ytgw/CodeCommentTranslator/commits/master/?since=2024-10-18&until=2024-10-19)
