---
title: C++23 ライブラリ機能2
author: onihusube
date: 2025/12/31
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

C++の最新の事情を知るにはほぼ毎月公開されている提案文書を読むのが確実ですが、それにはかなりの労力が必要です。それをやってくれている人ももはや少なく、最新のC++に関する（特に日本語の）記事もあまりない中で、それでもC++の最新の機能に興味がある方の要望に応えるために本書があります。本書を読むことによって、C++23の一端に触れることができ、それがどのように使用できてどのように役立つのかを学ぶことができます。

本書はC++23で追加された標準ライブラリ機能のうち、ヘッダ単位でまとまるような大きなものについて解説する本です。C++23の言語機能に関する解説は既刊の「C++23 コア言語機能」で解説しています。また、次のライブラリヘッダについては「C++23 コア言語機能」で解説を行っているため、ここでは省略しています

- `<stdfloat>`
- `<generator>`

これら以外のライブラリ機能の細かい修正や改善等については、引き続き「C++23 ライブラリ機能2」として刊行予定です。

## 注意など

本書はC++23をベースとして記述されています。そのため、C++20までに導入されている機能については特に導入バージョンについて触れず、既知のものとして詳しい解説なども行いません。

文中では基本的に`std::`を省略していませんが、文脈上明らかな場合などに一部で`std::`を省略することがあります。また、通常の関数の表記（`func()`）に対してメンバ関数の表記（`.member_func()`）は先頭に`.`を付加して区別します。

## サンプルコードのお約束

- ヘッダのインクルード/インポートはすべて省略し、サンプルコードの最初で`import std;`しているものとします。
- 主題と関係ないところを`...`で省略していることがあります。
- 行の末尾コメントで`ok/ng`と表示することで、それがコンパイル可能かどうかを示しています。`// ok`はコンパイルが通り未定義動作がないこと、`// ng`はコンパイルエラーとなることをそれぞれ表しています。

コードブロック中で標準出力を行っている時、直後のブロックでその出力例を示していることがあります。例えば次のようになっています

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

# コンテナ

## `std::stack`/`std::queue`にイテレータペアを取るコンストラクタ
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p1425r4.pdf

`std::stack`と`std::queue`はイテレータペアを受け取るコンストラクタがなかったため、他のコンテナのコンストラクタとの一貫性を欠いていました。それによってC++23で追加された`ranges::to`はそのままだとこの2つのコンテナアダプタへの変換ができないという問題がありました。

そこで、この2つのコンテナアダプタにイテレータペアを取るコンストラクタを追加することで、他のコンテナとの一貫性を改善し統一的な扱いができるようにするとともに、そのまま`ranges::to`で変換可能になります。また、同時にイテレータペアからの推論補助も追加されているため、CTADで要素型を推論することもできるようになっています。

```cpp
int main() {
  std::array<int, 4> arr = {1, 2, 3, 4};

  // C++20まで、こう書けばできた
  std::stack<int> st1{{arr.begin(), arr.end()}};
  std::queue<int> qu1{{arr.begin(), arr.end()}};

  // C++23からはそのまま+CTADが利用可能
  std::stack st2{arr.begin(), arr.end()};
  std::queue qu2{arr.begin(), arr.end()};
}
```
```cpp
int main() {
  namespace ranges = std::ranges;
  std::list lst = {1, 2, 3};

  // ranges::toによる変換
  auto stk = lst | ranges::to<std::stack>();  // ok
  auto que = lst | ranges::to<std::queue>();  // ok
}
```

## アロケータ引数のCTADに関する修正
## `std::pair`の転送対応
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p1951r1.html

## 連想コンテナ削除操作のヘテロジニアス化

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2077r3.html

# 文字列・フォーマット

## `std::string`/`std::string_view`

### `.contains()`

### `nullptr`コンストラクタ

### `.resize_and_overwrite()`

### `std::string::substr() &&`

### string_viewとspanをトリビアルコピー可能と規定

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2251r1.pdf

## `std::format`/`std::print`

### `pair`/`tuple`の対応

### `std::thread::id`

### `basic_format_string`を使用可能にする

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2508r1.html

### デバッグオプション

## iostream

### `basic_ostream`の出力の`const volatile void*`対応

### `noreplace`オープンモード

# functional

## `std::invoke_r()`

## `std::move_only_function`

## `std::bind_back()`

# memory

## `std::out_ptr`/`std::inout_ptr`

## `allocate_at_least()`

## `std::start_lifetime_as()`

## `std::byteswap()`

##  std::allocator_traitsの特殊化禁止

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2652r2.html

# utility

## `std::visit()`の制限緩和

## `std::to_underlying()`

## `std::forward_like()` 

## `std::optional`のモナディック操作

## `std::unreachable()`

## `std::exchange`の`noexcept`指定調整

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2401r0.html

## `std::apply`の`noexcept`調整
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2517r1.html

## tuple-likeオブジェクトの相互変換
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2165r4.pdf


## `std::barrier`の同期保証の変更

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2588r3.html

# type_trais

## `std::is_scoped_enum`

## 一時オブジェクトが参照に束縛されたことを検出する型特性

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2255r2.html

## `std::is_implicit_lifetime`

## reference_wrapper のcommon_reference_tが参照型となる

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2655r3.html

## 比較コンセプトのムーブオンリー型対応

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2404r3.pdf

# chrono

## `time_point<>::clock`の要件緩和

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2212r2.html

## エンコーディングによる`format()`に振舞いの明確化

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2419r2.html

# イテレータ

## `move_iterator<T*>`をランダムアクセスイテレータにする
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2520r0.html

## Poison Pillオーバーロードの削除
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2602r2.html

# その他変更

## constexpr対応

### `std::unique_ptr`の`constxpr`対応

### `std::bitset`の`constexpr`対応

### `<cmath>`/`<cstdllib>`の数学関数

### `std::type_info::operator==`の`constexpr`対応

### `to_chars`/`form_chars`の整数型オーバーロード

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2291r3.pdf

### `std::optional, std::variant`の一部の関数など
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2231r1.html

## 非推奨化・削除

### std::aligned_storage and std::aligned_union

### std::numeric_limits::has_denorm
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2614r2.pdf

### ガベージコレクション関連API
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2186r2.html

### いくつかのCヘッダの非推奨化解除
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2340r1.html

## フリースタンディング対応

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p1642r11.html

\clearpage

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- Compiler Explorer(https://godbolt.org/)
- yohhoyの日記(https://yohhoy.hatenadiary.jp/)

\clearpage
