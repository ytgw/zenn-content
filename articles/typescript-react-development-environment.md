---
title: "TypeScriptとReactの開発環境構築の備忘録"
emoji: "⚛️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "react", "eslint", "prettier"]
published: false
---


## 概要

TypeScriptとReact開発環境についての備忘録です。
類似の記事も多数あり何番煎じかも分かりませんが、自分でまとめないと毎回ツールや設定方法を検索する羽目になりそうなので、未来の自分のためにメモを残します。


<!-- ---------------------------------------------------------------------- -->


## プロジェクト作成

[ReactのWEBサイト](https://beta.reactjs.org/learn/start-a-new-react-project#getting-started-with-a-minimal-toolchain)で推奨されていたので、Create React App(CRA)を選びました。
もしかしたら近い将来はViteなどを使うかもしれませんが、一旦はCRAにしました。
[CRAのWEBサイト](https://create-react-app.dev/docs/getting-started/#creating-a-typescript-app)を参考にTypeScriptテンプレートを使ってプロジェクトを作ります。

```bash
mkdir foo
cd foo
npx create-react-app . --template typescript
```


<!-- ---------------------------------------------------------------------- -->


## リンター

[ReactのEditor Setup](https://beta.reactjs.org/learn/editor-setup)に記載があったので、ESLintを選びました。

### インストール

[ESLintのGetting Started](https://eslint.org/docs/latest/use/getting-started)と[こちらの記事](https://zenn.dev/ro_komatsuna/articles/eslint_setup)を参考にしました。

```bash
npm init @eslint/config
```

このコマンド実行時に聞かれる質問には、下記で答えました。

- How would you like to use ESLint?

    To check syntax, find problems, and enforce code style

- What type of modules does your project use?

    JavaScript modules

- Which framework does your project use?

    React

- Does your project use TypeScript?

    Yes

- Where does your code run?

    Browser

- How would you like to define a style for your project?

    Use a popular style guid

- Which style guide do you want to follow?

    Airbnbを選んでいる記事が多かったのですが、私が実行したときは選べなかったので、Standardにしました。

- What format do you want your config file to be in?

    JSON


設定後は下記コマンドでリンターを実行できます。

```bash
# --fixをつけると一部自動で修正する。
npx eslint --fix .
```

また、VSCodeと連携する場合は[こちらの拡張機能](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)をインストールします。


### 設定

設定内容は.eslintrc.jsonという名前のファイルに書かれています。
インストール直後のファイルをベースに下記を変更しました。

- 実行時のエラー対処

    ```json:.eslintrc.jsonの抜粋
    {
        "parserOptions": {
            "project": "./tsconfig.json",
        },
        "settings": {
            "react": {
                "version": "detect"
            }
        }
    }
    ```

- ビルドファイルの除外

    ```json:.eslintrc.jsonの抜粋
    {
        "ignorePatterns": ["build/"]
    }
    ```

- prettierとの競合回避

    extendsの最後に記述します。

    ```json:.eslintrc.jsonの抜粋
    {
        "extends": [
            "other extends",
            "prettier"
        ],
    }
    ```

- React Hook関連のルール追加

    [ReactのWEBサイト](https://beta.reactjs.org/learn/editor-setup#linting)に案内がありました。
    設定は[こちら](https://www.npmjs.com/package/eslint-plugin-react-hooks)を参考にしました。

    ```json:.eslintrc.jsonの抜粋
    {
        "plugins": ["react-hooks"],
        "rules": {
            "react-hooks/rules-of-hooks": "error",
            "react-hooks/exhaustive-deps": "warn"
        }
    }
    ```

上記の変更を加えたことで最終的に以下の内容になりました。

```json:.eslintrc.json
{
    "env": {
        "browser": true,
        "es2021": true
    },
    "extends": ["plugin:react/recommended", "standard-with-typescript", "prettier"],
    "ignorePatterns": ["build/"],
    "overrides": [],
    "parserOptions": {
        "ecmaVersion": "latest",
        "project": "./tsconfig.json",
        "sourceType": "module"
    },
    "plugins": ["react", "react-hooks"],
    "rules": {
        "react-hooks/rules-of-hooks": "error", // Checks rules of Hooks
        "react-hooks/exhaustive-deps": "warn" // Checks effect dependencies
    },
    "settings": {
        "react": {
        "version": "detect"
        }
    }
}
```


<!-- ---------------------------------------------------------------------- -->


## フォーマッター


[ReactのEditor Setup](https://beta.reactjs.org/learn/editor-setup)に記載があったので、Prettierを選びました。

### インストール

[PrettierのWEBサイト](https://prettier.io/docs/en/install.html)と[こちらの記事](https://zenn.dev/ro_komatsuna/articles/prettier_setup)を参考にしました。
また、ESLintと同時に使う場合は競合回避のための設定が必要で、[こちら](https://prettier.io/docs/en/install.html#eslint-and-other-linters)を参考にしました。

```bash
# prettier本体
npm install --save-dev --save-exact prettier

# ESLintとの競合を防ぐためのモジュール
npm install --save-dev eslint-config-prettier
```

設定後は下記コマンドでフォーマットできます。

```bash
npx prettier --write .
```

また、VSCodeと連携する場合は[こちらの拡張機能](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode)をインストールします。

### 設定

設定全般は.prettierrc.jsonに、フォーマット対象外のファイルは.prettierignoreに書きます。
それぞれ下記のように設定しました。

- .prettierrc.json

    列数を GitHub のコードレビュー画面のデフォルトに合わせて変更。

    ```json:.prettierrc.json
    {
        "printWidth": 119
    }
    ```

- .prettierignore

    ビルドファイルなどフォーマット不要なものを追加。

    ```
    # Ignore artifacts:
    build
    coverage
    ```


<!-- ---------------------------------------------------------------------- -->


## まとめ

TypeScriptとReact開発環境をしたときの備忘録を書きました。
未来の自分や皆さんの参考になれば幸いです。
