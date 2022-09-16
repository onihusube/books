---
title: C++20 ライブラリ機能 2
author: onihusube
date: 2022/12/31
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

本書は、C++20で導入された新しいライブラリ機能についての紹介と解説を行うものです。コア言語機能と同様にC++20のライブラリ拡張もそこそこ大きな範囲に及んでおり、全般的に使用できるような機能（*vocabulary type* : 語彙型、と呼ばれます）だけではなく特定の用途において有用なものもあるなど、その全体像を理解するには多くの提案や規格文書を読む必要があるとともに、多様な前提知識も必要とします。そのため、ただ変更点を眺めていたりライブラリリファレンスを見ているだけでは、必ずしもその機能の意義や有用性を把握できず、C++20ライブラリの全体像を掴む事ができません。

本書は、C++20の新機能に興味はあるもののついて行けないという人や、ある程度C++20機能を知っているものの深入りできていないという人向けに、C++20で新しく導入されたライブラリ機能の内容や意義についての解説を試みるものです。

この本で取り上げるライブラリ機能は主に、新しく導入されたヘッダを中心とした大きな機能です。既存ライブラリの改善などの小さめの機能は後ほど刊行（予定）のライブラリ機能 2で紹介する予定です。

なお、本書ではC++17までのライブラリ機能に関しては前提知識として説明しません。また、C++20のコア言語機能に関しては拙著『C++20 コア言語機能』を、C++20 Rangeライブラリについては拙著『C++20 ranges』をご参照ください。

## サンプルコードのお約束

- そこで主題となっているライブラリ機能のためのヘッダのみを明示的にインクルードし、他のヘッダのインクルードは省略します。
- 主題と関係ないところを`...`で省略していることがあります。
- 行の末尾コメントで`ok/ng`と表示することで、それがコンパイル可能かどうかを示しています。`// ok`はコンパイルが通り未定義動作がないこと、`// ng`はコンパイルエラーとなる事をそれぞれ表しています。

コードブロック中で標準出力をしている時、直後のブロックでその出力例を示していることがあります。例えば次のようになっています

\clearpage

```cpp
int main() {
  std::cout "hello world!";
}
```
```{style=planetext}
hello world!
```

\clearpage

# イテレータ

C++20では、コンセプトと`<ranges>`の導入に伴って、イテレータライブララリ（`<iterator>`）周りも大幅に改修されています。

## イテレータへの問合せ

### 値型
### 参照型
### 右辺値参照型
### 距離型

### イテレータを用いた関数呼び出しの結果型
### 射影操作の結果型


## イテレータコンセプト

C++20より、イテレータという概念はコンセプトによって定義されるようになります。

### `indirectly_readable`

`std::indirectly_readable`は間接参照によって値を読み出す事ができることを表すコンセプトです。

```cpp
namespace std {
  // 説明専用コンセプト
  template<class In>
  concept indirectly-readable-impl = ...;

  template<class In>
  concept indirectly_­readable = indirectly-readable-impl<remove_cvref_t<In>>;
}
```

`std::indirectly_readable<In>`は、`In`のオブジェクトから間接参照演算子（`operator*`）によって何らかの値を読み出す事ができる場合に`true`となります。これは、イテレータ型だけではなくポインタ型やスマートポインタ型でも満たす事ができます。

`indirectly-readable-impl`は次のように定義される説明専用のコンセプトで、これを経由しているのは入力の型`In`を`remove_cvref_t`に通すためです。なお、`remove_cvref_t`はC++20から導入された型特性で、型から参照と`const`を取り除くものです（型特性の章で紹介しています）。

```cpp
template<class In>
concept indirectly-readable-impl =
  requires(const In in) {
    typename iter_value_t<In>;
    typename iter_reference_t<In>;
    typename iter_rvalue_reference_t<In>;
    { *in } -> same_as<iter_reference_t<In>>;
    { ranges::iter_move(in) } -> same_as<iter_rvalue_reference_t<In>>;
  } &&
  common_reference_with<iter_reference_t<In>&&, iter_value_t<In>&> &&
  common_reference_with<iter_reference_t<In>&&, iter_rvalue_reference_t<In>&&> &&
  common_reference_with<iter_rvalue_reference_t<In>&&, const iter_value_t<In>&>;
```

複雑ですが、`iter_value_t`などによって各種型の問い合わせが可能であることやそれらの型の間に`common_reference`があることなどを指定しています。また、`requires`式内部の4つ目の式で間接参照可能性が制約されていますが、ここで使用されている`in`は`const`オブジェクトであるため、このコンセプトを満たすための間接参照演算子は`const`修飾されている必要があります。

このコンセプトには意味論要件が指定されています。

- 型`In`のオブジェクト`i`に対して、`*i`は等しさを保持する

等しさの保持とは、式の実行に伴って副作用を及ぼしたり内部状態に依存して結果が変わらないことを言います。

### `indirectly_writable`
### `weakly_incrementable`
### `incrementable`
### `input_or_output_iterator`
### `sized_sentinel_for`
### `input_iterator`
### `output_iterator`
### `forward_iterator`
### `bidirectional_iterator`
### `random_access_iterator`
### `contiguous_iterator`

## `iterator_traits`の役割の変化

## 進行と距離

## `counted_iterator`

## `common_iterator`

## カスタマイゼーションオブジェクト
### `ranges::iter_move`
### `ranges::iter_swap`

## 番兵型

# コンテナ

## 連想コンテナ関連

### `.contains()`

### 透過的な検索

### 比較の調整

## `erase/erase_if`

## `to_array()`

## `ssize()`

## リストの一部メンバ関数の戻り値型変更

# アルゴリズム

## `shift_left/shift_right`
## `lexicographical_compare_three_way`
## `midpoint`
## `lerp`
## `std::execution::unseq`

# 関数オブジェクト

## `reference_wrapper`の不完全型対応

## `unwrap_reference`

## `bind_front`

## `identity`

# 文字列

## `starts_with/ends_with`

## `reserve()`の縮小機能の廃止

# `std::atomic`

## 浮動小数点数方に対する特殊化

## 整数型のロックフリーエイリアス

## `std::atomic_ref`

## `atomic_flag::test()`

## `wait()/notify_one()/notify_all()`

## `std::shared_ptr/std::weak_ptr`の`std::atomic`特殊化

## `std::memory_order`の定義の変更

## Compare-and-exchangeとパディング

# `iostream`

## 配列の出力修正

## ユニコード文字型の出力禁止

## `std::basic_stringbuf::view()`

## 文字列のバッファ/ストリームクラスのアロケータ対応

# スマートポインタ

## `make_shared`の配列対応

## スマートポインタをデフォルト構築する

# メモリ

## `assume_aligned`

## uses-allcator構築のためのユーティリティ  

## `polymorphic_allocator`の改良


# `constexpr`化

## `std::string/std::vector`及びアロケータ

# 型特性

## `std::remove_cvref`
## `std::type_identity`
## `std::is_nothrow_convertible`
## `std::is_bounded_array`

## レイアウト/ポインタ互換性の判定

#　その他

## 安全な整数型の比較

## 標準ライブラリ型の`<=>`

## `[[nodiscard]]`

# 非推奨と削除

\clearpage

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
