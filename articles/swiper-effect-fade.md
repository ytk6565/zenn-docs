---
title: "Swiper の effect: fade でスライドが重なる"
emoji: "🌠"
type: "tech"
topics: ["css", "javascript"]
published: true
---

# はじめに

[Swiper](https://swiperjs.com/) というスライダーライブラリでフェードエフェクトを利用した際に、
アクティブ以外のスライドも重なって見えてしまう問題に対応したためメモとしてまとめます。

# 原因

[公式のデモ](https://swiperjs.com/demos/#fade_effect) では `background-size: cover;` のためとくに問題なく表示されておりますが、
`img` タグや `background-size: contain;` などの場合は**縦横比が統一されない可能性があるため**スライドが重なって見えてしまいます。

# 解決方法

解決方法は非常に簡単で下記のようにオプションを設定することです。
`fadeEffect: { crossFade: true }` を設定することでスライドが重ならずにアクティブのスライドのみ表示されます。

```js
{
  effect: 'fade',
  fadeEffect: {
    crossFade: true
  },
}
```

下記 GIF ファイルが挙動のデモです。
左側がデフォルトで右側が `fadeEffect: { crossFade: true }` を設定した状態です。

![デモ](https://ytk6565.s3-ap-northeast-1.amazonaws.com/zenn.dev/swiper-effect-fade/demo.gif)

# おわりに

デフォルトの挙動よりも汎用的と感じたため Stack Overflow に投稿してみました。

情報の誤り、誤字脱字などございましたら https://github.com/ytk6565/zenn-docs に Issue を建てていただけますと幸いです。

# 参考にさせていただいたページ

- https://swiperjs.com/
- https://swiperjs.com/api/#fade-effect
- https://swiperjs.com/demos/#fade_effect
