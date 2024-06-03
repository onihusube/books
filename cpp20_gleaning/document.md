---
title: C++20 落穂拾い
author: onihusube
date: 2024/08/12
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

本書は既発行の『C++20 コア言語機能』『C++20 ranges』『C++20 ライブラリ機能1, 2』などで時間の制約などの都合で扱っていなかったトピックについて解説するものです。

## 注意など

本書はC++20をベースとして記述されています。そのため、C++17までに導入されている機能については特に導入バージョンについて触れず、知っているものとして詳しい解説なども行いません。また、C++20の他の機能に関しても、すでに以前の書籍で説明済みのものは知っているものとして使用します。そちらの説明は以前発行の書籍（『C++20 コア言語機能』『C++20 ranges』『C++20 ライブラリ機能1, 2』）もしくはcpprefjpなどの解説サイトを参照してください。

## サンプルコードのお約束

- そこで主題となっているライブラリ機能のためのヘッダのみを明示的にインクルードし、他のヘッダのインクルードは省略します
- 主題と関係ないところを`...`で省略していることがあります
- 行の末尾コメントで`ok/ng`と表示することで、それがコンパイル可能かどうかを示しています。`// ok`はコンパイルが通り未定義動作がないこと、`// ng`はコンパイルエラーとなる事をそれぞれ表しています
- 本文中でメンバ関数を表記する際、`.menber_func()`のように先頭に`.`を付加して区別しています

コードブロック中で標準出力をしている時、直後のブロックでその出力例を示していることがあります。例えば次のようになっています

```cpp
int main() {
  std::cout << "hello world!";
}
```
```{style=planetext}
hello world!
```

　

標準ライブラリ中での宣言を例示する際、コードブロックの見た目を分けて表示しています（上と左の線が二重線 + 角丸）。例えば次のようになっています

```{style=cppstddecl}
// std::vectorの宣言例
namespace std {
  template<class T, class Allocator = allocator<T>>
  class vector;
}
```

\clearpage

# コンセプトの半順序、もう少し

# 一貫比較仕様、比較演算子の合成について

# `<chrono>`のスキップしたもの

## `hhmmss`

## タイムゾーンデータベース

# `consteval`コンストラクタによるコンパイル時検査について

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- Compiler Explorer(https://godbolt.org/)

また、大阪C++読書会において以前に発行したC++20コア言語機能及びC++20ライブラリ機能1を輪読していただき、感想等のフィードバックを頂いております。モチベーションになっています、ありがとうございます！

- [大阪C++読書会 - connpass](https://cpp-osaka.connpass.com/ )
- [大阪C++読書会 - scrapbox](https://scrapbox.io/cpp-osaka/ )
