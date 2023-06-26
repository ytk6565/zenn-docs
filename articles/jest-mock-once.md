---
title: "Jest の mockImplementationOnce で定義した関数が mockClear でリセットされない"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["jest"]
published: true
---

## はじめに

Jest でテストを書いていて、一度だけ実行することを想定するモック関数に `mockImplementationOnce` を使ったところ `mockClear` でリセットされなかったので原因と解決方法のメモを書く。

- Jest のバージョンは `29.5.0` を使用する。

## 対象について

- テスト対象となる関数の要件
  - クライアントから2つのエンドポイント（A, B）にリクエストを送り結果を返す関数
  - 1つ目のエンドポイント（A）からのレスポンスを2つ目のエンドポイント（B）のリクエストのパラメータとして利用
- テストしたい項目（ここで記載しない項目もあるが省略する）
  - ①Aのリクエストに失敗したとき、エラーがスローされ、Bのリクエストが実行されないこと
  - ②Bのリクエストが失敗したとき、エラーがスローされること

## 問題のあるソースコード

```ts:index.js
export const fetchData = async (fetchA, fetchB) => {
  const a = await fetchA();
  const b = await fetchB(a);
  return b;
};
```

```ts:index.test.js
import { jest, describe, test, expect } from "@jest/globals";
import { fetchData } from "./";

describe("fetchData", () => {
  const fetchAMock = jest.fn();
  const fetchBMock = jest.fn();

  afterEach(() => {
    fetchAMock.mockClear();
    fetchBMock.mockClear();
  });

  test("①Aのリクエストに失敗したとき、エラーがスローされ、Bのリクエストが実行されないこと", async () => {
    expect.hasAssertions();
    fetchAMock.mockImplementationOnce(() => Promise.reject("error from A"));
    fetchBMock.mockImplementationOnce(() => Promise.resolve("data from B"));

    try {
      await fetchData(fetchAMock, fetchBMock);
    } catch (e) {
      expect(e).toBe("error from A");
      expect(fetchAMock).toHaveBeenCalled();
      expect(fetchBMock).not.toHaveBeenCalled();
    }
  });

  test("②Bのリクエストが失敗したとき、エラーがスローされること", async () => {
    expect.hasAssertions();
    fetchAMock.mockImplementationOnce(() => Promise.resolve("data from A"));
    fetchBMock.mockImplementationOnce(() => Promise.reject("error from B"));

    try {
      await fetchData(fetchAMock, fetchBMock);
    } catch (e) {
      expect(e).toBe("error from B");
      expect(fetchAMock).toHaveBeenCalled();
      expect(fetchBMock).toHaveBeenCalledWith("data from A");
    }
  });
});
```

## 問題

テストを実行すると、「①Aのリクエストに失敗したとき、エラーがスローされ、Bのリクエストが実行されないこと」はパスするが、「②Bのリクエストが失敗したとき、エラーがスローされること」は失敗する。

②のテストは catch 句内のアサーションではなく、 `expect.hasAssertions();` のアサーションで失敗している。
そのため、②のテストでは処理が catch 句に入っていない。
試しに try 句にパスするアサーション（例：`expect(true).toBe(true);`）を追記したところテストがパスした。

## 原因

`mockImplementationOnce` の引数で受け取った関数は、内部的に配列で管理されており、モック関数がコールされるたびに先頭の関数をコールして配列から削除している。
`mockClear` ではこの配列の初期化を行なっていないためリセットされない。

### 該当のソースコード

1. `mockImplementationOnce` の引数に渡された関数を内部の配列にプッシュしている箇所

https://github.com/jestjs/jest/blob/efe3eddae6506bc2429b464374fe048ce5cfa8e7/packages/jest-mock/src/index.ts#L821-L827

2. 1で `mockConfig.specificMockImpls` にプッシュされた関数がコールされる箇所

https://github.com/jestjs/jest/blob/efe3eddae6506bc2429b464374fe048ce5cfa8e7/packages/jest-mock/src/index.ts#L714-L722

## 解決方法

1. 確実に実行される箇所以外で `mockImplementationOnce` を使わないようにする
2. `mockReset` または `mockRestore` でリセットする

`mockImplementationOnce` は名前から**一度だけ**実行することを想定しているので、1が最も適切な解決方法だと考えている。

### 1. 確実に実行される箇所以外では `mockImplementationOnce` ではなく `mockImplementation` を使う

`mockImplementationOnce` で受け取った関数は `mockImplementation` で受け取った関数よりも優先して実行される。

https://github.com/jestjs/jest/blob/efe3eddae6506bc2429b464374fe048ce5cfa8e7/packages/jest-mock/src/index.ts#L716-L719

`mockImplementationOnce` は実行されないと、残り続け、その後に `mockImplementation` や `mockImplementationOnce` をしても最後に設定した関数が実行されない。

これに対して `mockImplementation` では、引数で受け取った関数を内部の変数に代入しているだけなので、**既存の関数を上書き**することができる。

そのため、確実に実行される箇所以外では `mockImplementationOnce` ではなく `mockImplementation` を使うようにすることで解決する。

### 2. `mockReset` または `mockRestore` でリセットする

`mockReset` と `mockRestore` は `mockImplementationOnce` によって関数をプッシュする配列もリセットするためこれらのいずれかを利用することで解決する。

`jest.spyOn()` を使っていれば `mockRestore` 、使っていなければ `mockReset` を利用する。

## おわりに

Jest のソースコードを読む中で [`clearMocks`](https://jestjs.io/docs/configuration#clearmocks-boolean), [`resetMocks`](https://jestjs.io/docs/configuration#resetmocks-boolean), [`restoreMocks`](https://jestjs.io/docs/configuration#restoremocks-boolean) などのグローバルな設定でモック関数のリセットを指定できることを知れてよかった。
他にも `WeakMap` で関数をキーとした値の紐付けができることを知れたのもよかった。

今後も OSS のコードリーディングをして、自分の言葉でアウトプットするのを続けていきたい。