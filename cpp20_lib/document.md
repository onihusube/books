---
title: C++20 ライブラリ機能（α）
author: onihusube
date: 2022/08/14
geometry:
  width: 188mm
  height: 263mm
#coverimage: cover.jpg
#backcoverimage: backcover.jpg
titlecolor:
  color1:
    r: 0.7882
    g: 0.6745
    b: 0.9092
    c: 0.25
    m: 0.3
    y: 0.0
    k: 0.0
  color2:
    r: 0.6
    g: 0.0
    b: 0.6
    c: 0.3
    m: 0.95
    y: 0.0
    k: 0.0
okuduke:
  revision: 初版
  printing: ねこのしっぽ
---
\clearpage

# はじめに

## ok/ng

サンプルコード中では、特定の行の末尾のコメントで`ok/ng`と表示することで、それがコンパイル可能かどうかを示しています。`// ok`はコンパイルが通り未定義動作がないこと、`// ng`はコンパイルエラーとなる事をそれぞれ表しています。ただし、どの言語バージョン（C++17 or 20）で`ok/ng`なのかは文脈によっている事があります。

\clearpage

# `<concepts>`
\clearpage
# `<ranges>`

## Rangeコンセプト
## Rangeアダプタ
## Rangeアルゴリズム
\clearpage
# `<format>`
\clearpage
# `<bit>`
\clearpage
# `<numbers>`
\clearpage
# `<span>`
\clearpage
# `<source_location>`
\clearpage
# `<coroutine>`
\clearpage
# `<syncstream>`
\clearpage
# `<stop_token>`と`std::jthread`

## `<stop_token>`

`<stop_token>`ヘッダでは、マルチスレッド処理における協調的な処理のキャンセル操作を扱うためのライブラリ機能を提供します。

### `std::stop_source/std::stop_token`

キャンセル操作のためには`std::stop_source/std::stop_token`の2つのクラスを利用し、基本的なキャンセル処理の流れは次のようになります

1. メインスレッドで、`std::stop_source`オブジェクトを初期化
2. メインスレッドで、`std::stop_source::get_token()`から`std::stop_token`を取得
3. 2で作った`std::stop_token`オブジェクトを処理スレッドへ受け渡す
4. 処理スレッド内では、`std::stop_token::stop_requested()`によってキャンセル要求をチェック
5. 処理のキャンセルは、`std::stop_source::request_stop()`によって行う。

```cpp
#include <stop_token>
#include <future>

int main() {
  // 1. stop_sourceの初期化
  std::stop_source ss{};

  // 2. stop_tokenの取得
  // 3. stop_tokenの受け渡し
  auto f = std::async(std::launch::async, [st = ss.get_token()] {

    // 4. キャンセルされたかをチェック
    while(st.stop_requested() == false) {
      // 別スレッドで行う処理
      ...
    }

  });

  // 5. 処理のキャンセル
  ss.request_stop();
}
```

`std::stop_source/std::stop_token`のアイデアは`future`パターンと同じで、スレッド間の共有状態を用いてキャンセル要求を伝達するものです。そして、キャンセルされる側ではキャンセル要求をチェックしながら処理を実行する必要があります。

`<stop_token>`の提供するマルチスレッド処理のキャンセルとは、スレッドをいきなり中断するようなものではありません。スレッドの中断はよくやりたくなりますが、RAIIと相性が悪くリソースリークの危険性があり、そのスレッドに関連する状態が不定となるなど、本質的に危険なのでやるべきではありません。面倒かもしれませんが、`std::stop_source/std::stop_token`のような協調的なキャンセルを行ってください。

### コールバック

## `std::jthread`

\clearpage
# `<semaphore>`
\clearpage
# `<latch>`と`<barrier>`

## `std::latch` - 一回の同期
## `std::barrier` - 複数回の同期

\clearpage
# `<chrono>`
\clearpage
## カレンダー
## タイムゾーン
## `std::format`

\clearpage
# `<compare>`
## 比較関数オブジェクト

\clearpage
# `<version>`

`<version>`ヘッダでは、C++標準ライブラリのバージョンに関わる情報を提供します。

\clearpage
# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- C++マルチスレッド一巡り(https://zenn.dev/yohhoy/books/cpp-stdlib-multithreading)
- std::threadデストラクタ動作検討の歴史(https://zenn.dev/yohhoy/scraps/393bce83b4f3f0)
- threadの利用と例外安全（その1）(https://yohhoy.hatenadiary.jp/entry/20120209/p1)