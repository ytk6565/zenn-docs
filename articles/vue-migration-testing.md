---
title: "1つのソースコードで Vue 2 と Vue 3 の両方の環境で動作する UI ライブラリのテスト環境構築"
emoji: "🐧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vue", "jest", "cypress", "vite", "turborepo"]
published: true
---

# はじめに

現在開発しているアプリケーションで Vue 2 から Vue 3 へのバージョンアップを進めている。
この開発の中で、Vue 2 のアプリケーションで利用しているコンポーネントが Vue 3 のアプリケーションから利用した際に振る舞いが損なわれないことを担保しながら開発を進めることを目的として、1つのソースコードで Vue 2 と Vue 3 の両方の環境で動作する UI ライブラリのテスト環境の構築を行った。
今回はこのような環境の構築方法についてまとめる。
UI ライブラリのサンプルは `vite` で構築するが `webpack` や `esbuild` でも同様のことが実現できるはず。

## リポジトリ

https://github.com/ytk6565/vue-migration-testing

# 環境構築

## UI ライブラリ

### 依存パッケージ

#### Vue のバージョン切り替え

- Vue 2 と Vue 3 の両方の環境で動作させたいので [vue-demi](https://github.com/vueuse/vue-demi) をインストールしている。
- `vue-demi` のバージョン切り替え用のスクリプトを `setup:2`, `setup:3` で定義している。

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/package.json#L8-L9

- また、それぞれのスクリプトに `VUE_VERSION` という環境変数を与えてテスト・ビルドで処理を分岐する。

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/package.json#L11-L12

#### npm パッケージにエイリアスを定義

- `vue` と `@testing-library/vue` は Vue 2 と Vue 3 でパッケージが分かれていないため npm のエイリアスを定義して、別名で異なるバージョンをインストールしている。

##### `vue`

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/package.json#L49

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/package.json#L52

##### `@testing-library/vue`

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/package.json#L31-L32

#### Vue のコンパイラ

- Vue 3 のコンパイラは vue の 3.2.13 以降はパッケージに内包されているので別途インストールする必要はない。
  - `vue/compiler-sfc` で参照できる。
  - https://github.com/vuejs/core/tree/main/packages/compiler-sfc#vuecompiler-sfc
- Vue 2 のコンパイルのために `vue-template-compiler` をインストールしている。

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/package.json#L50

### コンポーネント

シンプルなカウンターを実装する。 composition 関数は別ファイルに切り出す。

#### Vue コンポーネント

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/src/VCounter/index.vue

#### composition 関数

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/src/VCounter/compositions/useCounter.ts

### Vite

#### バージョン切り替え

環境変数の `VUE_VERSION` をもとに以下の設定を切り替える。

- `vue` の参照先。

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/vite.config.ts#L10-L14

- vite 用の vue プラグイン。

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/vite.config.ts#L15-L19

#### ビルド設定

- ライブラリモードでビルドする。

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/vite.config.ts#L23-L27

- `vue`, `vue-demi` はバンドルせずに利用側で解決する。

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/vite.config.ts#L28-L36

### Jest

#### 設定

環境変数の `VUE_VERSION` をもとに以下の設定を切り替える。

- `.vue` ファイルを解決するライブラリ。

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/jest.config.js#L9

- `vue`, `@testing-library/vue` の参照先。

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/jest.config.js#L13-L18

#### composition 関数のテスト

- global に定義されている jest の `expect` の型が cypress とコンフリクトするので `@jest/globals` からインポートする。

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/src/VCounter/compositions/\_\_tests\_\_/useCounter.test.ts

#### ノードベースのコンポーネントテスト

- `@testing-library/vue` でノードベースのコンポーネントテストを実装する。

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/src/VCounter/\_\_tests\_\_/component.test.ts

#### 動作確認

Vue 2 と Vue 3 の両方の環境で動作することを確認する。

```bash
$ npm -w @packages/ui run test:2
$ npm -w @packages/ui run test:3
```

### Cypress Component Testing

#### 設定

devServer は `vite` で立ち上げる。

環境変数の `VUE_VERSION` をもとに以下の設定を切り替える。

- `cypress/vue` の参照先。

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/cypress.config.ts#L10-L13

#### ブラウザベースのコンポーネントテスト

- cypress でブラウザベースのコンポーネントテストを実装する。
- `cy` に `mount` のコマンドを追加して型を拡張すると jest とコンフリクトするので、 `cypress/vue` からインポートする。

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/packages/ui/src/VCounter/\_\_tests\_\_/component.cy.ts

#### 動作確認

Vue 2 と Vue 3 の両方の環境で動作することを確認する。

```bash
$ npm -w @packages/ui run cy:run:2
$ npm -w @packages/ui run cy:run:3
```

### ビルド

#### 動作確認

Vue 2 と Vue 3 の両方の環境で動作することを確認する。

```bash
$ npm -w @packages/ui run build:2
$ npm -w @packages/ui run build:3
```

## アプリケーション

UI ライブラリが Vue 2 と Vue 3 の両方の環境で動作することが確認できたので、あとは任意のアプリケーションから利用すれば良い。

一応 Vue 2 は Vue CLI で、 Vue 3 は Vite でアプリケーションのサンプルを作成した。
また、 Turborepo でコマンド一つで依存ライブラリがビルドされるように設定した。

### Vue 2

https://github.com/ytk6565/vue-migration-testing/tree/b04352cb01b91d18e409090b3136a42a208e732a/apps/vue2

### Vue 3

https://github.com/ytk6565/vue-migration-testing/tree/b04352cb01b91d18e409090b3136a42a208e732a/apps/vue3

### Turborepo

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/package.json#L12-L15

https://github.com/ytk6565/vue-migration-testing/blob/b04352cb01b91d18e409090b3136a42a208e732a/turbo.json

# 最後に

お読みいただきありがとうございます。
改善点や不明点があればコメントお願いいたします。