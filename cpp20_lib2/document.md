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

この本で取り上げるライブラリ機能は主に、既存ライブラリの改善などの小さめの機能です。ヘッダ単位などの大きな機能は『C++20 ライブラリ機能1』にて取り上げています。

なお、本書ではC++17までのライブラリ機能に関しては前提知識として説明しません。また、C++20のコア言語機能に関しては拙著『C++20 コア言語機能』を、C++20 Rangeライブラリについては拙著『C++20 ranges』をご参照ください。

## サンプルコード等のお約束

- そこで主題となっているライブラリ機能のためのヘッダのみを明示的にインクルードし、他のヘッダのインクルードは省略します。
- 主題と関係ないところを`...`で省略していることがあります。
- 行の末尾コメントで`ok/ng`と表示することで、それがコンパイル可能かどうかを示しています。`// ok`はコンパイルが通り未定義動作がないこと、`// ng`はコンパイルエラーとなる事をそれぞれ表しています。
- 本文中でメンバ関数を表記する際、`.menber_func()`のように先頭に`.`を付加して区別しています

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

　

標準ライブラリ中での宣言を例示する際、コードブロックの見た目を分けて表示しています（上と左の線が二重線 + 角丸）。例えば次のようになっています

```{style=cppstddecl}
// std::vectorの宣言例
namespace std {
  template<class T, class Allocator = allocator<T>>
  class vector;
}
```

\clearpage

# イテレータ

C++20では、コンセプトと`<ranges>`の導入に伴って、イテレータライブララリ周りも大幅に改修されています。特に明記しない場合、この章で紹介している機能は`<iterator>`ヘッダに配置されます。

## イテレータ情報の問合せ

これまで、イテレータ型の各種情報を問い合わせるのには`std::iterator_traits`を使用していましたが、C++20からはそれに変わるより簡易な手段が提供されるようになります。また同時に、従来はなかった追加の情報を取得するための手段も提供されます。

### 距離型 - `difference_type`

`difference_type`はイテレータの距離を表す型で、イテレータの差分操作（`operator-`）の戻り値型でもあります。

従来は`std::iterator_traits<I>::difference_type`から取得していましたが、C++20からは`std::iter_difference_t<I>`を用いる事で同じものを取得できます。

```cpp
#include <iterator>

using std::ranges::iota_view;

using iota_view_iter= std::ranges::iterator_t<iota_view<int>>;

static_assert(std::same_as<
  std::iter_difference_t<std::vector<int>::iterator>, 
  std::ptrdiff_t
>);

static_assert(std::same_as<
  std::iter_difference_t<int*>,
  std::ptrdiff_t
>);

static_assert(std::same_as<
  std::iter_difference_t<iota_view_iter>,
  std::ptrdiff_t
>);
```

#### `std::incrementable_traits`

　

`std::iter_difference_t`は基本的にはC++20で追加された`std::incrementable_traits`を用いて`difference_type`を取得していて、`std::incrementable_traits`はいくつかの経路を使って`difference_type`を探してくれます。

イテレータ型を`I`とすると、次のいずれかから取得します

- `I::difference_type`
- `I`の`oeprator-`（2項演算）の戻り値型
- `std::incrementable_traits<I>`の明示的特殊化
    - `difference_type`メンバ型から取得

C++20からのイテレータ型は上記いずれかで取得できるようにしておけばいいわけです。

なお、`difference_type`はイテレータコンセプトの最小の要求の一部でもあるので、イテレータ型では必ず`std::iter_difference_t`から取得可能である必要があります。おそらく多くの場合は入れ子の`difference_type`を定義するのが簡単でしょう（つまり今まで通り）。

### 値型 - `value_type`

`value_type`はイテレータの指す要素の型を表す型です。大抵はイテレータの間接参照の戻り値型から参照を除去した型になるはずです。

従来は`std::iterator_traits<I>::value_type`から取得していましたが、C++20からは`std::iter_value_t<I>`を用いる事で同じものを取得できます。

```cpp
#include <iterator>

using std::ranges::iota_view;

using iota_view_iter = std::ranges::iterator_t<iota_view<unsigned int>>;

static_assert(std::same_as<
  std::iter_value_t<std::vector<int>::iterator>,
  int
>);

static_assert(std::same_as<
  std::iter_value_t<double*>,
  double
>);

static_assert(std::same_as<
  std::iter_value_t<iota_view_iter>,
  unsigned int
>);
```

#### `std::indirectly_readable_traits`

　

`std::iter_value_t<I>`は基本的にはC++20で追加された`std::indirectly_readable_traits`を用いて`value_type`を取得し、`std::indirectly_readable_traits`はいくつかの経路を使って`value_type`を探してくれます。

イテレータ型を`I`とすると、次のいずれかから取得します

- `I::value_type`
- `I::element_type`
- `std::indirectly_readable_traits<I>`の明示的特殊化
    - `value_type`メンバ型から取得

この`value_type`も他の場所で使用されていることがあるので必ず定義しておいたほうがいいでしょう。おそらく入れ子の`value_type`を定義するのが簡単でしょう（これも今まで通り）。

### 参照型 - `reference`

`reference`はイテレータの指す要素を参照する参照型で、これはイテレータの間接参照の戻り値型です。

従来は`std::iterator_traits<I>::reference`から取得していましたが、C++20からは`std::iter_reference_t<I>`を用いる事で同じものを取得できます。

```cpp
#include <iterator>

using iota_view_iter = std::ranges::iterator_t<std::ranges::iota_view<unsigned int>>;

static_assert(std::same_as<
  std::iter_reference_t<std::vector<int>::iterator>,
  int&
>);

static_assert(std::same_as<
  std::iter_reference_t<double*>,
  double&
>);

static_assert(std::same_as<
  std::iter_reference_t<iota_view_iter>,
  unsigned int
>);
```

`reference`というのは歴史的経緯から来る名前で、イテレータの間接参照の戻り値型は必ずしも参照型ではなくてもokです。

`std::iter_reference_t`は次のように定義されています。

```{style=cppstddecl}
namespace std {
  template<dereferenceable I>
  using iter_reference_t = decltype(*declval<I&>());
}
```

これはそのまま間接参照の戻り値型を取得しており、イテレータ型を定義するにあたっては`operator*`を定義すると自動で取得可能になります。

### 右辺値参照型

`std::iter_rvalue_reference_t`は`std::iterator_traits`にはなかったもので、イテレータの要素をムーブするための右辺値参照型を表すものです。

```cpp
#include <iterator>

using iota_view_iter = std::ranges::iterator_t<std::ranges::iota_view<unsigned int>>;

static_assert(std::same_as<
  std::iter_rvalue_reference_t<std::vector<int>::iterator>,
  int&&
>);

static_assert(std::same_as<
  std::iter_rvalue_reference_t<double*>,
  double&&
>);

static_assert(std::same_as<
  std::iter_rvalue_reference_t<iota_view_iter>,
  unsigned int
>);
```

イテレータを`i`とすると、大抵の場合は`decltype(std::move(*i))`の型を取得することになりますが、`*i`がprvalueを返す場合はその型をそのまま取得します。

例えばイテレータの要素への右辺値を別の型（例えば`std::tuple`など）に詰めて転送したい場合、`*i`がprvalueを返す時に右辺値参照を使用すると危険な（ダングリング参照になる）ためその場合は素の型をそのまま使用するようにするハンドリングを自動で行いたい場合に利用できます。他には、コンセプトによる制約を使用する場合に`*i`をムーブするための適切な型を取得するためにも使用できます。

以下、ここからは`std::iterator_traits`にはなかったものが続きます。

### 共通の参照型

`std::iter_common_reference_t`は、`std::iter_value_t<I>&`と`std::iter_reference_t<I>`の両方を束縛することのできるような共通の参照型を表すものです。

```cpp
#include <iterator>

using iota_view_iter = std::ranges::iterator_t<std::ranges::iota_view<unsigned int>>;

static_assert(std::same_as<
  std::iter_common_reference_t<std::vector<int>::iterator>,
  int&
>);

static_assert(std::same_as<
  std::iter_common_reference_t<double*>,
  double&
>);

static_assert(std::same_as<
  std::iter_common_reference_t<iota_view_iter>,
  unsigned int
>);
```

これも`reference`といいつつ、必ずしも参照型であるとは限りません。

`std::iter_common_reference_t`は次のように定義されています。

```{style=cppstddecl}
namespace std {
  template<indirectly_readable I>
  using iter_common_reference_t = common_reference_t<iter_reference_t<I>, iter_value_t<I>&>;
}
```

普通のイテレータでは`std::iter_reference_t<I>`と同じ型になると思われますが、例えば間接参照がprvalueを返すイテレータではその型を`T`とするとそのまま`T`になります。

イテレータを用いたアルゴリズムを書く際にこのような性質を持つ型が必要になることがよくあるため、それを簡易に求めたい時に使用できます。

### イテレータを用いた関数呼び出しの結果型

`std::indirect_result_t`は、イテレータ型を間接参照して関数に入力して呼び出した時の戻り値型を取得するものです。

```cpp
#include <iterator>

template<typename T>
auto comp(const T& lhs, const T& rhs) -> bool;

using f_t = decltype(&comp<int>);
using vecit = std::vector<int>::iterator;

static_assert(std::same_as<
  std::indirect_result_t<f_t, int*, int*>, 
  bool
>);

static_assert(std::same_as<
  std::indirect_result_t<f_t, vecit, vecit>, 
  bool
>);
```

イテレータ`i1, i2, ..., in`とその要素を渡して呼びだす関数`f`がある時、`f(*i1, *i2, ..., *in)`と呼んだ時の戻り値型を取得するものです。

例えばこのような比較関数や`swap`など、イテレータを用いたアルゴリズム中でこのような呼び出しを行う関数は頻出します。コンセプトの文脈でそれらの呼び出しやその結果を制約する際に`std::indirect_result_t`を使用できます。

## イテレータコンセプト

C++20より、イテレータという概念はコンセプトによって定義されるようになります。それに伴い、`<iterator>`には細分化され階層化されたイテレータコンセプトが定義されるようになります。

### `indirectly_readable`

`std::indirectly_readable`は間接参照によって値を読み出す事ができることを表すコンセプトです。

```{style=cppstddecl}
namespace std {
  // 説明専用コンセプト
  template<class In>
  concept indirectly-readable-impl = ...;

  template<class In>
  concept indirectly_readable = indirectly-readable-impl<remove_cvref_t<In>>;
}
```

`std::indirectly_readable<In>`は、`In`のオブジェクトから間接参照演算子（`operator*`）によって何らかの値を読み出す事ができる場合に`true`となります。これは、イテレータ型だけではなくポインタ型やスマートポインタ型でも満たす事ができます。

`indirectly-readable-impl`は次のように定義される説明専用のコンセプトで、これを経由しているのは入力の型`In`を`remove_cvref_t`に通すためです。なお、`remove_cvref_t`はC++20から導入された型特性で、型から参照と`const`を取り除くものです（型特性の章で紹介しています）。

```{style=cppstddecl}
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

等しさの保持とは、式の実行に伴って副作用を及ぼしたり内部状態に依存して結果が変わらないことを言います。つまりは、`*i`が副作用や内部状態の更新を持たないことを言っています。

### `indirectly_writable`

`std::indirectly_writable`は間接参照先に値を書き込む事ができることを表すコンセプトです。

```{style=cppstddecl}
namespace std {
  template<class Out, class T>
  concept indirectly_writable = 
    requires(Out&& o, T&& t) {
      *o = std::forward<T>(t);
      *std::forward<Out>(o) = std::forward<T>(t);
      const_cast<const iter_reference_t<Out>&&>(*o) = std::forward<T>(t);
      const_cast<const iter_reference_t<Out>&&>(*std::forward<Out>(o)) = std::forward<T>(t);
    };
}
```

`std::indirectly_writable<Out, T>`は、`Out`のオブジェクトが間接参照演算子（`operator*`）によって型`T`の値を書き込む事ができる場合に`true`となります。これもまた、イテレータ型だけではなくポインタ型やスマートポインタ型でも満たす事ができます。

このコンセプトを構成する4つの制約式は全て、等しさを保持することを要求されません。つまりは、`*o`が内部状態を更新することを認めています。

このコンセプトには意味論要件が指定されています。型`T`の値`e`と間接参照可能な型`Out`のオブジェクト`o`について

\clearpage

- 型`Out, T`が`indirectly_readable<Out> && same_as<iter_value_t<Out>, decay_t<T>>`のモデルとなる場合、`e`を4つの制約式のいずれかによって出力した後で`*o`と`e`は等値となる

これはイテレータを介した出力操作に当然要求される事でしょう。これを満たさないようなものは出力とは言えません。前提条件の`Out, T`へのコンセプト要求は、出力だけが可能で読み取りできない場合や出力時に変換される場合を除くための条件です。

定義中の`const_cast`を用いる制約式は、間接参照が*pravlue*を返すようなイテレータを受け入れるためのものです。例えば、プロクシイテレータと呼ばれるイテレータは要素へのプロクシオブジェクトを`operator*`から返し、その戻り値型は参照型ではなくプロクシオブジェクトの型そのものになります。その場合でも、プロクシオブジェクトは`operator*`によって参照先へ書き込むことができるため、プロクシイテレータは`indirectly_writable`となる必要があります。一方で、`std::string`など、*prvalue*への出力（`std::string() = std::string()`）ができてしまうけれど明らかにプロクシオブジェクトではないようなものが考えられ、そのような場合には`indirectly_writable`とならないようにしなければなりません。

`const_cast`を用いる2つの制約式は、`Out`のオブジェクト`o`の`*o`が*prvalue*を返す場合にのみその上にある2つの制約式と異なった振る舞いをし、`*o`を`const T&&`にキャストした後でも出力が可能かどうかを調べることでプロクシイテレータとそうでないもの（`std::string() = std::string()`のようなもの）を判別しています。

### `weakly_incrementable`

`std::weakly_incrementable`は、インクリメント操作が可能であることを表すコンセプトです。

```{style=cppstddecl}
template<class I>
concept weakly_incrementable =
  movable<I> &&
  requires(I i) {
    typename iter_difference_t<I>;
    requires is-signed-integer-like<iter_difference_t<I>>;
    { ++i } -> same_as<I&>;
    i++;
  };
```

`std::weakly_incrementable<I>`は、`I`が前置/後置インクリメントが可能でそれが何らかの進行を表す（インクリメントによって距離が変化する）場合に`true`となります。これはイテレータ型だけでなく整数型でも満たすことができます。

このコンセプトで定義されるインクリメント操作（後ろ2つの制約式）は等しさを保持することが要求されません。すなわち、`I`がイテレータ型の場合にマルチパス保証を提供しません。マルチパス保証とはある範囲を複数のイテレータから同時に走査できる保証で、これができるイテレータはイテレータの操作によって参照する範囲の状態を変更しないものです。

`is-signed-integer-like<T>`とは説明専用の変数テンプレートで、`T`が符号付き整数型である場合に`true`となるもので、`std::signed_integral`を使用しないのは非標準の整数型を含めることを意図しているためです。`std::weakly_incrementable`は後述のイテレータコンセプトからも参照されており、これによってイテレータの距離型（`difference_type`）が符号付き整数型であることを要求します。

このコンセプトには意味論要件が指定されています。型`I`のオブジェクト`i`について

- `++i`と`i++`は同じ定義域をもつ
- `i`がインクリメント可能ならば、`++i`と`i++`は`i`を次の要素へ進める
- `i`がインクリメント可能ならば、`addressof(i)`はインクリメント前後で変化しない

定義域とは、式（関数）の引数全体からその式を不正にするような入力を除いた残りの部分（集合）の事を言います。インクリメント操作は前置/後置に関わらず進行に関しては同じ効果を持たなければなりません。

後ろ2つの要件内の「インクリメント可能」とは、`i`が前置/後置`++`の定義域にある場合のことです。これは、`end`イテレータや符号付き整数型の正の最大値のようにインクリメントできない（未定義動作となる）値を除くための条件です。

重要なのはおそらく2つ目の要件で、これによってインクリメント操作が何らかの進行をするものであることを定義しています。イテレータの場合はそのままの意味ですが、整数型の場合は+1した値になると言う意味になります。

### `incrementable`

`std::incrementable`は、`weakly_incrementable`な型が`regular`であり、インクリメント操作が副作用を持たないことを表すコンセプトです。

```{style=cppstddecl}
template<class I>
concept incrementable =
  regular<I> &&
  weakly_incrementable<I> &&
  requires(I i) {
    { i++ } -> same_as<I>;
  };
```

`std::incrementable<I>`は`std::weakly_incrementable<I>`と`std::regular<I>`（コピー・ムーブ構築/代入と等値比較可能）が`true`であり後置インクリメントが`I`の*prvalue*を返す場合に`true`となります。

`std::weakly_incrementable`との構文的な違いは、後置インクリメント（`i++`）の戻り値型がその型の*prvalue*であると指定されているところで、`std::weakly_incrementable`では戻り値なし（`void`）でも良かったところが厳しくなっています。ただし、これはC++17までの多くのイテレータで普通だったことであり、`weakly_incrementable`なイテレータの後置インクリメントの戻り値型が`void`でもいいのはC++17イテレータとの非互換ポイントでもあります。

このコンセプトには意味論要件が指定されています。まず

- `std::weakly_incrementable`の前置/後置`++`は等しさを保持する

次に、型`I`のインクリメント可能なオブジェクト`a, b`について

- `bool(a == b)`が`true`ならば`bool(a++ == b)`
- `bool(a == b)`が`true`ならば`bool(((void)a++, a) == ++b)`

ここでの「インクリメント可能」は`std::weakly_incrementable`の時と同じ意味で、`std::incrementable`とは異なる意味です。

これらの意味論要件によって、`std::weakly_incrementable`では求められていなかったインクリメント操作の副作用禁止が要求されるようになっています。特に、後の2つの条件はイテレータに対するマルチパス保証を表しています。

`incrementable`なイテレータは同じ範囲を複数のイテレータによって同時に走査することができ、それが保証されます。

### `input_or_output_iterator`

`std::input_or_output_iterator`はC++におけるイテレータの最小の要件を表すコンセプトです。

```{style=cppstddecl}
template<class I>
concept input_or_output_iterator =
  requires(I i) {
    { *i } -> can-reference;
  } &&
  weakly_incrementable<I>;
```

`std::input_or_output_iterator<I>`は`I`が最低限イテレータ型だと見做せる場合に`true`となります。ここで要求されているのは、`*`で何か読み出しができることとインクリメント操作ができること、及びムーブ可能であることだけです。

イテレータには追加の性質がありますがここではそれは何ら指定されておらず、別のコンセプトによって指定されます。とはいえ、全てのイテレータはまずこのコンセプトのモデルとなる必要があります。

### `sentinel_for`

`std::sentinel_for`はイテレータ型に対する番兵型であることを表すコンセプトです。

```{style=cppstddecl}
template<class S, class I>
concept sentinel_for =
  semiregular<S> &&
  input_or_output_iterator<I> &&
  weakly-equality-comparable-with<S, I>;
```

`std::sentinel_for<S, I>`は`S`がイテレータ型`I`に対する番兵型である場合に`true`となります。

番兵とはイテレータ範囲`[begin, end)`の`end`イテレータのことを言い、番兵型とはその型のことを言います。C++17イテレータでは番兵型とイテレータ型は同一であることが前提とされていましたが、C++20ではそうでなくてもよく、このコンセプトはイテレータ型に対する番兵型を定義するものです。

番兵型に対する要求はイテレータ型と比較して非常に弱く、`input_or_output_iterator`であることも求められておらずイテレータ型`I`と`S`の間で等値比較可能であることがほぼ全てです。

このコンセプトには意味論要件が指定されています。型`S, I`のオブジェクトをそれぞれ`s, i`とし、`[i, s)`は範囲を表すとして

- `i == s`が未定義動作を含まない
- `bool(i != s)`が`true`の場合、`i`は間接参照可能であり`[++i, s)`も範囲を表す
- `I, S`は`std::assignable_from<I&, S>`のモデルとなるか、構文的にも満たさない

ここでの`==`の定義域は静的なものではなく実行時に変化しうるものです。例えば、`[i, s)`が有効な範囲であり`i != s`であるときに、`i == oi`となるような別のイテレータ`oi`をインクリメントした後でも`[i, s)`が有効であり続ける必要はなく、`i == s`が有効（未定義とならない）であり続ける必要もありません。このようなことは、イテレータのインクリメントが参照する範囲の状態を変更するような場合に起こり得ます。

2つ目の要件は`i`が`s`に到達していない間は間接参照とインクリメントが有効であることを言っています。これは範囲を走査するイテレータの基本的な性質の一部でもあります。

3つ目の要件は、イテレータオブジェクトに対して番兵オブジェクトを代入できる（それぞれのオブジェクトを`i, s`として、`i = s`が可能である）性質についての要件で、代入が有効でない場合は`std::assignable_from<I&, S> == false`とならなければならないことを指定しています。イテレータ型と番兵型が同じ型ならば番兵を代入することで範囲の最後のイテレータを得るという操作は簡単に実装できますが、イテレータ型と番兵型が異なる場合は必ずしも実装可能ではありません。この要件は、そのような操作が番兵型にとって必須ではないことを言っています。

### `sized_sentinel_for`

`std::sized_sentinel_for`はイテレータ型に対して距離を定義可能な番兵型であることを表すコンセプトです。

```{style=cppstddecl}
template<class S, class I>
concept sized_sentinel_for =
  sentinel_for<S, I> &&
  !disable_sized_sentinel_for<remove_cv_t<S>, remove_cv_t<I>> &&
  requires(const I& i, const S& s) {
    { s - i } -> same_as<iter_difference_t<I>>;
    { i - s } -> same_as<iter_difference_t<I>>;
  };
```

`std::sized_sentinel_for<S, I>`は、`std::sentinel_for<S, I>`が`true`であり2項`opreator-`によってイテレータと番兵の間で距離を求めることができる場合に`true`となります。

このコンセプトには意味論要件が指定されています。イテレータが型`I`と番兵型`S`のオブジェクト`i, s`とそれによって示される範囲を`[i, s)`、`bool(i == s)`が`true`となるために必要な`++i`の適用回数を`N`として

- `N`が`iter_difference_t<I>`型で表現可能である場合、`s - i`は未定義動作を含まず、`N`に等しい
- `-N`が`iter_difference_t<I>`型で表現可能である場合、`i - s`は未定義動作を含まず、`-N`に等しい

イテレータと番兵の間の距離についてを定義するとともに、イテレータと番兵の間の引き算（`operator-`）はそのイテレータと番兵との間の距離を表すことを言っています。

`std::disable_sized_sentinel_for<S, I>`は変数テンプレートで、`std::sized_sentinel_for<S, I> == true`となる時でも上記意味論要件を満たすことができない場合に`std::sized_sentinel_for`を無効化するためのものです。利用する場合は、`S, I`について`true`となるように明示的特殊化を定義します。

### `input_iterator`

`std::input_iterator`は入力イテレータであることを表すコンセプトです。

```{style=cppstddecl}
template<class I>
concept input_iterator =
  input_or_output_iterator<I> &&
  indirectly_readable<I> &&
  requires { typename ITER_CONCEPT(I); } &&
  derived_from<ITER_CONCEPT(I), input_iterator_tag>;
```

`std::input_iterator<I>`は、`I`が入力イテレータである場合に`true`となります。

このコンセプトはC++20イテレータにおける入力イテレータの定義でもあります。ここで求められているのは`I`が`input_or_output_iterator`であることと`indirectly_readable`であること、すなわち`operator*`による読み込みとインクリメントによる進行とムーブ可能であることです。

このコンセプトは`incrementable`を要求しないので、入力イテレータにはマルチパス保証がありません。

単に`input_iterator`であるイテレータには例えば、`std::istream_iterator`があります。

#### `ITER_CONCEPT(I)`

　

`std::input_iterator`を構成する制約式のうち後2つはイテレータ型からイテレータタグ型を引き出しチェックするためのもので、イテレータの要件そのものに直接関与しているものではありません。そこで使用されている`ITER_CONCEPT(I)`はイテレータから取得したイテレータタグ型を表すエイリアステンプレートのようなものです。

まず、型`IT`を次のように定義して

- `std::iterator_traits<I>`の明示的特殊化がない場合
    - `using IT = I;`
- `std::iterator_traits<I>`が特殊化されている場合
    - `using IT = std::iterator_traits<I>;`

`ITER_CONCEPT(I)`は次の順番でタグ型を取得しようとします

1. `IT::iterator_concept`
2. `IT::iterator_category`
3. `std::iterator_traits<I>`の明示的特殊化がない場合
    - `std::random_access_iterator_tag`

3番目までに当てはまらない場合、`ITER_CONCEPT(I)`は型名を示しません。

3番目のケースはフォールバックのためのもので、イテレータタグ型が取得できない場合はとりあえずランダムアクセスイテレータと思っておいて、実際のインターフェースからイテレータの性質を決定します。

```{style=cppstddecl}
template<class I>
concept input_iterator =
  ...
  requires { typename ITER_CONCEPT(I); } &&
  derived_from<ITER_CONCEPT(I), input_iterator_tag>;
```

この2つの制約式のうち、上は`ITER_CONCEPT(I)`が何らかの型名となることをチェックしていて、タグ型が取得できない場合はこの制約式を満たすことができずコンセプト全体は`false`となります。

下の制約式は取得したタグ型がイテレータコンセプトに応じたタグ型（ここでは`std::input_iterator_tag`）に合っているかをチェックしています。`std::derived_from`で継承関係をチェックしているのは、より強いイテレータタグを扱えるようにするため（例えば、ランダムアクセスイテレータは入力イテレータでもある）と、将来的にイテレータカテゴリが追加された時に受け入れられるようにするためです。後者については、実際C++17で*contiguous iterator*と言う新しいイテレータがカテゴリが追加されています。

他のイテレータコンセプトもこの部分は同じような定義になっています。

### `output_iterator`

`std::output_iterator`は出力イテレータであることを表すコンセプトです。

```{style=cppstddecl}
template<class I, class T>
concept output_iterator =
  input_or_output_iterator<I> &&
  indirectly_writable<I, T> &&
  requires(I i, T&& t) {
    *i++ = std::forward<T>(t);  // 等しさを保持することを要求しない
  };
```

`std::output_iterator<I, T>`は、型`I`が`T`の値を出力可能である場合に`true`となります。

最後の制約式は、後置インクリメントの戻り値型が`I`を返し、右辺値イテレータからも出力可能であることを制約しています。後置インクリメントの戻り値型がイテレータ型であることの要求は`std::incrementable`で行われていますがこのコンセプトはそれを包摂しておらず、マルチパス保証もありません。

このコンセプトには意味論要件が指定されています。型`T`の値`t`と`I`の値`i`について

- `*i++ = t`は次の式と等価

```cpp
*i = t;
++i;
```

この要件によって、後置`++`が自身の型のオブジェクト以外のものを返すことや返したイテレータが別の範囲を参照しているなどを禁止しています。インクリメント操作は後置と前置で（戻り値を除いて）意味が変わってはいけません。

単に`output_iterator`であるイテレータには例えば`std::ostream_iterator`があり、`*i`が左辺値参照を返す殆どのイテレータは同時に`output_iterator`でもあります。

### `forward_iterator`

`std::forward_iterator`は前方向イテレータであることを表すコンセプトです。

```{style=cppstddecl}
template<class I>
concept forward_iterator =
  input_iterator<I> &&
  derived_from<ITER_CONCEPT(I), forward_iterator_tag> &&
  incrementable<I> &&
  sentinel_for<I, I>;
```

`std::forward_iterator<I>`は、`I`が前方向イテレータである場合に`true`となります。

`std::input_iterator`を包摂していることで、`std::forward_iterator`は`input_iterator`でもありかつ`std::input_iterator`よりも強い制約となります。また、`std::incrementable`を包摂していることでマルチパス保証をもち、`std::sentinel_for<I, I>`を包摂していることで自身がそのまま番兵型となることができます。なお、この場合でも別に番兵型を持つことはできます。

このコンセプトには意味論要件が指定されています

- `==`の定義域は同じ範囲を参照するイテレータの全体
    - デフォルト構築されたイテレータ同士の比較は常に`true`となる（デフォルト構築されたイテレータは空の範囲の終端を指しているかのように振る舞う）
- 範囲`[i, s)`を参照するイテレータから取得された参照やポインタは、`[i, s)`が有効である限り有効

マルチパス保証やこれらの要件によって、`forward_iterator`であるイテレータが参照する範囲はかなりしっかりとしたものである必要があり、イテレータの操作によって範囲の状態が変更されることはありません。1つ目の要件は異なる範囲を参照するイテレータ間の比較はこのコンセプトによって保証されないことを言っています。

標準ライブラリの全てのコンテナのイテレータは少なくとも`forward_iterator`であり、単に`forward_iterator`であるイテレータに`std::forward_list`のイテレータがあります。

### `bidirectional_iterator`

`std::bidirectional_iterator`は双方向イテレータであることを表すコンセプトです。

```{style=cppstddecl}
template<class I>
concept bidirectional_iterator =
  forward_iterator<I> &&
  derived_from<ITER_CONCEPT(I), bidirectional_iterator_tag> &&
  requires(I i) {
    { --i } -> same_as<I&>;
    { i-- } -> same_as<I>;
  };
```

`std::bidirectional_iterator<I>`は、`I`が`forward_iterator`でありデクリメント操作（`--`）による後進が可能である場合に`true`となります。

このコンセプトには意味論要件が指定されています。まず

- 双方向イテレータ`r`は`++q == r`となるようなイテレータ`q`が存在する場合にのみデクリメント可能である。
    - デクリメント可能なイテレータはデクリメント操作（`--`）の定義域に含まれる

次に、`I`の等しいオブジェクト`a, b`（同じ要素を指すイテレータ）について

- `a, b`がデクリメント可能ならば、次の全ては`true`となる
    - `addressof(--a) == addressof(a)`
    - `bool(a-- == b)`
    - `a--, --b`の評価の後でも、`bool(a == b)`は`true`
    - `bool(++(--a) == b)`
- `a, b`がデクリメント可能ならば、`bool(--(++a) == b)`

ここでのデクリメント操作には等しさを保持することが求められていることとこれらの要件によって、ここでのデクリメント操作に対する要求は`std::weakly_incrementable`ではなく`std::incrementable`の`++`を`--`に置き換えたようなものになっています。

単に`bidirectional_iterator`であるイテレータには例えば、`std::list`や`std::set, std::map`等のイテレータがあります。

### `random_access_iterator`

`std::random_access_iterator`はランダムアクセスイテレータであることを表すコンセプトです。

\clearpage

```{style=cppstddecl}
template<class I>
concept random_access_iterator =
  bidirectional_iterator<I> &&
  derived_from<ITER_CONCEPT(I), random_access_iterator_tag> &&
  totally_ordered<I> &&
  sized_sentinel_for<I, I> &&
  requires(I i, const I j, const iter_difference_t<I> n) {
    { i += n } -> same_as<I&>;
    { j +  n } -> same_as<I>;
    { n +  j } -> same_as<I>;
    { i -= n } -> same_as<I&>;
    { j -  n } -> same_as<I>;
    {  j[n]  } -> same_as<iter_reference_t<I>>;
  };
```

`std::random_access_iterator<I>`は、`I`が`bidirectional_iterator`かつ`totally_ordered`かつ`sized_sentinel_for`であり、整数との`+ - += -=`および添え字演算子`[]`が提供されている場合に`true`となります。

このコンセプトには意味論要件が指定されています。`std::iter_difference_t<I>`を`D`、`D`の値を`n`、`I`の有効なイテレータを`a, b`、`b`は`++a`を`n`回適用すると到達するとして

- `(a += n)`は`b`と等値
- `addressof(a += n)`は`addressof(a)`と等値
    - `+=`は`*this`を返す
- `(a + n)`は`(a += n)`と等値
- `D`の2つの正の値`x, y`について`(a + D(x + y))`が有効ならば、`(a + D(x + y))`は`((a + x) + y)`と等値
    - 結合則
- `(a + D(0))`は`a`と等値
- `(a + D(n - 1))`が有効ならば、`(a + n) `は`[](I c){ return ++c; }(a + D(n - 1))`と等値
- `(b += D(-n))`は`a`と等値
- `(b -= n)`は`a`と等値
- `addressof(b -= n)`は`addressof(b)`と等値
    - `-=`は`*this`を返す
- `(b - n)`は`(b -= n)`と等値
- `b`が間接参照可能ならば、`a[n]`は有効であり`*b`と等値
- `bool(a <= b) == true`

大量かつ複雑な要件ですが、ほぼ整数値`n`との演算がイテレータの進行を表すものであることを言っています。最後の要件は、イテレータ間の順序とは指している要素の相対的な位置関係によることを言っています。

生の配列や`std::array`、`std::vector`等のイテレータが`random_access_iterator`ですが、単に`random_access_iterator`であるイテレータには`std::deque`のイテレータがあります。

### `contiguous_iterator`

`std::contiguous_iterator`は隣接イテレータであることを表すコンセプトです。

```{style=cppstddecl}
template<class I>
concept contiguous_iterator =
  random_access_iterator<I> &&
  derived_from<ITER_CONCEPT(I), contiguous_iterator_tag> &&
  is_lvalue_reference_v<iter_reference_t<I>> &&
  same_as<iter_value_t<I>, remove_cvref_t<iter_reference_t<I>>> &&
  requires(const I& i) {
    { to_address(i) } -> same_as<add_pointer_t<iter_reference_t<I>>>;
  };
```

`std::contiguous_iterator<I>`は`I`が`random_access_iterator`であり参照する範囲の要素同士がメモリ上で連続している場合に`true`となります。

`contiguous_iterator`の参照する範囲はメモリ上で固まって存在しているはずのものであるため、間接参照結果は常に左辺値参照が得られるはずです。

`std::to_address()`はポインタ型からは`std::ponter_traits::to_address()`を用いて、それ以外の型からは`operator->`を用いてそのアドレスを取得するものです。他のイテレータでは`->`は要求されていませんでしたが、`contiguous_iterator`では実質的に`->`が要求されます。

対応するタグ型の`std::contiguous_iterator_tag`はC++20で追加されたものです。

このコンセプトには意味論要件が指定されています。`a, b`を間接参照可能な型`I`のイテレータ、`c`を間接参照不可能な型`I`のイテレータとして`b`は`a`から、`c`は`b`から到達可能であり、`std::iter_difference_t<I>`を`D`として

- `std::to_address(a) == std::addressof(*a)`
- `std::to_address(b) == std::to_address(a) + D(b - a)`
- `std::to_address(c) == std::to_address(a) + D(c - a)`
- `std::ranges::iter_move(a)`は`std::move(*a)`と同じ型と値カテゴリを持ち同じ効果となる
- `std::ranges::iter_swap(a, b)`が利用可能である場合、その効果は`std::ranges::swap(a, b)`と同じになる

イテレータの進行がメモリ上の位置の同距離の移動と一致することを言っています。すなわち、`contiguous_iterator`の参照する要素はメモリ上で連続していることを要求しています。

`contiguous_iterator`であるイテレータには生配列のポインタや`std::vector, std::string`のイテレータがあります。これらのイテレータとは実質的にポインタであり、ポインタ以外の`contiguous_iterator`はピンとこないものがあるかもしれません。非ポインタの`contiguous_iterator`の例としては、C++20からの`std::counted_iterator`にポインタ型を指定した特殊化（`std::counted_iterator<T*>`）があります。

## `iterator_traits`の役割の変化

ここまで見たように、C++20ではイテレータからの情報取得のための簡易な窓口とイテレータコンセプトが整備されたため、`std::iterator_traits`を使用する必要はなくなっています。その場合でもC++17以前のコードでは`std::iterator_traits`を用いることが一般的であり、コードがアップデートされない場合は使用され続けることでしょう。逆に言うと、C++20のコード内で`std::iterator_traits`を用いているところではC++17以前のレガシーなイテレータを使用しているとみなすことができるため、C++20からの`std::iterator_traits`はC++17以前のイテレータを期待するコードに対する互換レイヤとして機能するようになります。

```cpp
// C++20によるイテレータを用いる処理
template<std::forward_iterator I>
auto new_iter_alg(I i) {
  using V = std::iter_value_t<I>;
  using R = std::iter_reference_t<I>;
  using D = std::iter_difference_t<I>;

  ...
}

// C++17以前のイテレータを用いる処理
template<typename I>
auto old_iter_alg(I i) {
  using traits = std::iterator_traits<I>;

  static_assert(std::is_base_of_v<std::forward_iterator_tag, traits::iterator_category>);

  using V = typename traits::value_type;
  using R = typename traits::reference;
  using D = typename traits::difference_type;

  ...
}
```

C++20のコード（`new_iter_alg()`）からはイテレータの性質は主にコンセプトを通じて問い合わせられており、C++17のコード（`old_iter_alg()`）からはイテレータの性質は`std::iterator_traits`を通して問い合わせられています。

この2つの関数にC++20イテレータ（イテレータコンセプトに準拠したイテレータ）とC++17イテレータ（C++17以前に書かれたイテレータ、イテレータコンセプトへの準拠は不明）を入力してみた時のことを考えてみましょう。

`new_iter_alg()`はC++20の`forward_iterator`を受け入れます。C++20イテレータは当然問題なく使用できますが、C++17イテレータは定義のされ方によっては`forward_iterator`コンセプトを満たせずにエラーになるかもしれません。コンセプトのないC++17以前はイテレータの定義をしっかりと調べ固定化するのは簡単ではなかったため、これは仕方ないことです。ただし、C++17イテレータが`forward_iterator`コンセプトを満たしているならば`new_iter_alg()`はエラーを起こしません。C++17イテレータに対しての`std::iter_value_t`や`std::iter_difference_t`等のものは、C++17以前にイテレータとして使用できていたものに対してはきちんと動作するためです。

`old_iter_alg()`はC++17の*forward iterator*を受け入れます。C++17イテレータは問題なく使用できているはずで、C++20イテレータもC++17*forward iterator*要件を満たしているならば問題なく使用できます。その場合の`std::iterator_traits`の各種メンバ型は`std::iter_reference_t`などを用いて自動で取得されます。

基本的に、C++20のイテレータコンセプトの要求はC++17のイテレータ要件よりも厳しくなっており、C++17イテレータをC++20イテレータとして利用することは難しい一方で、C++20イテレータはC++17イテレータとしても利用することができる場合があります。ただし、細部の要件の非互換（`operator*`の戻り値型が参照型でなくても（*prvalue*でも）良いなど）のためにそのままのイテレータカテゴリ（特に、*forward*以上）のままでは利用できない場合があります。その場合に、C++17コードからは`std::iterator_traits`を通してイテレータの性質が取得されることを利用して、`std::iterator_traits`を介した場合にC++20イテレータがより弱いC++17イテレータとして見えるようになっています。

その際に重要なのが、`iterator_cateogry`と`iterator_concept`の2種類のメンバ型です。`iterator_cateogry`が以前からそうであるように、この2つのメンバ型はイテレータのタグ型を指定しておくことでそのイテレータ型がどの種類のイテレータなのかを表明するものです。そして、イテレータコンセプトは常に`iterator_concept`を優先して見に行き、`std::iterator_traits`は`iterator_cateogry`しか見に行きません。これによって、C++20コードから利用された時とC++17コードから利用された時で取得されるイテレータタグ型を切り替えることができるわけです。例えばそれはポインタ型に対する`std::iterator_traits`特殊化に見ることができます。

```{style=cppstddecl}
namespace std {
  template<class T>
    requires is_object_v<T>
  struct iterator_traits<T*> {
    // C++20からのカテゴリ
    using iterator_concept  = contiguous_iterator_tag;
    // C++17までのカテゴリ
    using iterator_category = random_access_iterator_tag;
  
    using value_type        = remove_cv_t<T>;
    using difference_type   = ptrdiff_t;
    using pointer           = T*;
    using reference         = T&;
  };
}
```

例えば配列イテレータ（ポインタ）を先ほどの2つの関数に入力すると、そこでは取得されるイテレータカテゴリが異なります。

```cpp
int main() {
  int arr[] = {1, 2, 3, 4, 5};

  // 隣接イテレータとして認識される
  new_iter_alg(std::ranges::begin(arr));

  // ランダムアクセスイテレータとして認識される
  old_iter_alg(std::ranges::begin(arr));
}
```

（制約が`forward_iterator`によって行われていることは無視してください・・・）

`<ranges>`の`view`型では、`iterator_category`を常に`input_iterator_tag`にしておくことでC++17コードに対する後方互換性を確保する、と言うことがよく行われています。

```{style=cppstddecl}
namespace std::ranges {

  // split_viewのイテレータ定義（細部は省略）
  template<forward_range V, forward_range Pattern>
  class split_view<V, Pattern>::iterator {
    ...

  public:
    // C++20コードから利用されるときはforward iteratorとなる
    using iterator_concept = forward_iterator_tag;

    // C++17コードから利用されるときはinput iteratorとなる
    using iterator_category = input_iterator_tag;

    ...
  };
}
```

逆に、`iterator_category`を定義しないことでC++20イテレータが後方互換性を持たないことを表明することもでき、これも`<ranges>`の`view`型でよく行われています。

### `iterator_traits`の特殊化

`std::iterator_traits`がイテレータ型について明示的に特殊化されている場合、イテレータコンセプトも`std::iterator_traits`もその特殊化を優先して見に行きます。`std::iter_value_t`と`std::iter_difference_t`も`std::iterator_traits`が特殊化されている場合はそちらから取得するようになります（これらの説明の中で「基本的には」と言っていたのはこの点です）。その後の流れは同じなのですが、この振る舞いによって、`std::iterator_traits`の特殊化はイテレータ型に対する非侵入的なカスタマイゼーションポイントとしての役割が与えられます。

例えば、C++20で追加されたイテレータラッパである`std::counted_iterator<I>`では、これを利用してラップするイテレータ型が`contiguous_iterator`である場合にC++17以前のコードに向けて`pointer`型を提供するようにしています。

```{style=cppstddecl}
namespace std {

  template<input_iterator I>
    requires /*iterator_traits<T>が特殊化されている場合*/
  struct iterator_traits<counted_iterator<I>> : iterator_traits<I> {
    using pointer = conditional_t<contiguous_iterator<I>,
                                  add_pointer_t<iter_reference_t<I>>, void>;
  };
}
```

これは主にポインタをラップした`std::counted_iterator`がC++17コードから利用された時にもきちんと*random access iterator*として利用されるためのものです。直接見えていませんが、`iterator_traits<I>`を継承していることで他の型も適切に提供されます。

変則的な使用法としては、`std::iterator_traits`の特殊化に`iterator_concept`を定義しておくことで、元のイテレータ型に変更を加えずにC++20イテレータであることを表明することもできます。例えばマクロによって言語バージョンでその存在を切り替えたり、あるいは`std::common_iterator`（これもC++20で追加）のようにそもそもC++17コードから利用されることを意図しているイテレータでは`std::iterator_traits`の特殊化によって`iterator_concept`も含めた各種イテレータ情報を提供するようになっています。

## イテレータ経由の関数呼び出しに関するコンセプト

比較関数や`swap`など、イテレータを用いたアルゴリズムではイテレータをデリファレンスして関数を呼び出す、ということがよく行われます。それを制約するためのコンセプトが用意されます。

### `indirectly_unary_invocable`

`std::indirectly_unary_invocable`はイテレータの要素型による単項（1引数）呼び出しが可能であることを表すコンセプトです。

```{style=cppstddecl}
template<class F, class I>
concept indirectly_unary_invocable =
  indirectly_readable<I> &&
  copy_constructible<F> &&
  invocable<F&, iter_value_t<I>&> &&
  invocable<F&, iter_reference_t<I>> &&
  invocable<F&, iter_common_reference_t<I>> &&
  common_reference_with<
    invoke_result_t<F&, iter_value_t<I>&>,
    invoke_result_t<F&, iter_reference_t<I>>>;
```

`std::indirectly_unary_invocable<F, I>`は、`I`の要素型によって`F`が呼び出し可能（`invocable`）である場合に`true`となります。`F, I`のオブジェクトを`f, i`とすると、`std::invoke(f, *i)`の呼び出しが可能であることを表しており、これは簡単には`f(*i)`のような呼び出しです。

ただし、`*i`の直接の結果だけでなく、その値型及び共通の参照型（`common_reference`）によっても呼び出し可能である必要があります。

```cpp
template<std::forward_iterator I, std::indirectly_unary_invocable<I> F>
void f(F&& f, I i) {
  f(*i);  // 参照型による呼び出し

  auto e = *i;
  f(e);   // 値型の左辺値による呼び出し

  using CI = std::iter_common_reference_t<I>;
  // 共通の参照型による呼び出し
  f(CI(*i));
  f(CI(e));
}
```

`indirectly_unary_invocable`な`F, I`では上記の全ての呼び出しが可能です。

さらには、呼び出し方法の違いによって戻り値型が大きく変わらないこと（`common_reference`を持つこと）も求められています。

このような要件を必要とするイテレータアルゴリズムには`std::for_each()`があり、そのRange版ではこのコンセプトが使用されます。

```{style=cppstddecl}
namespace std::ranges {
  // ranges::for_each()の宣言
  template<input_range R, 
           class Proj = identity,
           indirectly_unary_invocable<projected<iterator_t<R>, Proj>> Fun>
  constexpr
    for_each_result<borrowed_iterator_t<R>, Fun>
      for_each(R&& r, Fun f, Proj proj = {});
}
```

### `indirectly_regular_unary_invocable`

`std::indirectly_regular_unary_invocable`はイテレータの要素型による単項呼び出しが可能であり、それが副作用を持たないことを表すコンセプトです。

```{style=cppstddecl}
template<class F, class I>
concept indirectly_regular_unary_invocable =
  indirectly_readable<I> &&
  copy_constructible<F> &&
  regular_invocable<F&, iter_value_t<I>&> &&
  regular_invocable<F&, iter_reference_t<I>> &&
  regular_invocable<F&, iter_common_reference_t<I>> &&
  common_reference_with<
    invoke_result_t<F&, iter_value_t<I>&>,
    invoke_result_t<F&, iter_reference_t<I>>>;
```

`std::indirectly_regular_unary_invocable<F, I>`は`I`の要素型によって`F`が`regular_invocable`である場合に`true`となります。

このコンセプトは`std::indirectly_unary_invocable`の副作用を禁止する（ことを表明する）バージョンであり、違いは意味論要件のみです。それは、コンセプト定義中の`invocable`が`regular_invocable`に置き換えられていることによって要求されていて、それ以外の部分は同一です。

### `indirect_unary_predicate`

`std::indirect_unary_predicate`は、イテレータの要素型による単項述語（*unary predicate*）であることを表すコンセプトです。

```{style=cppstddecl}
template<class F, class I>
concept indirect_unary_predicate =
  indirectly_readable<I> &&
  copy_constructible<F> &&
  predicate<F&, iter_value_t<I>&> &&
  predicate<F&, iter_reference_t<I>> &&
  predicate<F&, iter_common_reference_t<I>>;
```

`std::indirect_unary_predicate<F, I>`は、`I`の要素型によって`F`が単項述語となる場合に`true`となります。`F, I`のオブジェクトを`f, i`とすると、`bool b = f(*i)`のような呼び出しと変換が可能であることを表します。

`std::predicate`コンセプトは述語を定義するもので、その呼出に伴う副作用がなく、結果は`bool`に変換可能である必要があるので、このコンセプトにおいても呼び出しが副作用を持ってはならないわけです。

このような要件を必要とするイテレータアルゴリズムには`std::find_if()`があり、そのRange版ではこのコンセプトが使用されます。

```{style=cppstddecl}
namespace std::ranges {
  template<input_range R, 
           class Proj = identity,
           indirect_unary_predicate<projected<iterator_t<R>, Proj>> Pred>
  constexpr
    borrowed_iterator_t<R>
      find_if(R&& r, Pred pred, Proj proj = {});
}
```

### `indirect_binary_predicate`

`std::indirect_binary_predicate`は、イテレータの要素型による2項述語（*binary predicate*）であることを表すコンセプトです。

```{style=cppstddecl}
template<class F, class I1, class I2>
concept indirect_binary_predicate =
  indirectly_readable<I1> && indirectly_readable<I2> &&
  copy_constructible<F> &&
  predicate<F&, iter_value_t<I1>&, iter_value_t<I2>&> &&
  predicate<F&, iter_value_t<I1>&, iter_reference_t<I2>> &&
  predicate<F&, iter_reference_t<I1>, iter_value_t<I2>&> &&
  predicate<F&, iter_reference_t<I1>, iter_reference_t<I2>> &&
  predicate<F&, iter_common_reference_t<I1>, iter_common_reference_t<I2>>;
```

`std::indirect_binary_predicate<F, I1, I2>`は、`I1, I2`の要素型によって`F`が2項述語となる場合に`true`となります。`F, I1, I2`のオブジェクトを`f, i1, i2`とすると、`bool b = f(*i1, *i2)`のような呼び出しと変換が可能であることを表します。

このような要件を必要とするイテレータアルゴリズムには`std::find()`があり、そのRange版ではこのコンセプトが使用されます。

```{style=cppstddecl}
namespace std::ranges {
  template<input_range R,
           class T,
           class Proj = identity>
    requires indirect_binary_predicate<ranges::equal_to, projected<iterator_t<R>, Proj>, const T*>
  constexpr
    borrowed_iterator_t<R>
      find(R&& r, const T& value, Proj proj = {});
}
```

### `indirect_equivalence_relation`

`std::indirect_equivalence_relation`は、イテレータの要素型による同値関係を表すコンセプトです。

```{style=cppstddecl}
template<class F, class I1, class I2 = I1>
concept indirect_equivalence_relation =
  indirectly_readable<I1> && indirectly_readable<I2> &&
  copy_constructible<F> &&
  equivalence_relation<F&, iter_value_t<I1>&, iter_value_t<I2>&> &&
  equivalence_relation<F&, iter_value_t<I1>&, iter_reference_t<I2>> &&
  equivalence_relation<F&, iter_reference_t<I1>, iter_value_t<I2>&> &&
  equivalence_relation<F&, iter_reference_t<I1>, iter_reference_t<I2>> &&
  equivalence_relation<F&, iter_common_reference_t<I1>, iter_common_reference_t<I2>>;
```

`std::indirect_equivalence_relation<F, I1, I2>`は、`F`が`I1, I2`の要素型の同値関係となる場合に`true`となります。`F, I1, I2`のオブジェクトを`f, i1, i2`とすると、`bool b = f(*i1, *i2)`のような呼び出しと変換が可能であり、結果が同値関係を示すことを表します。

`std::equivalence_relation`と同様に、同値関係になっていることについては意味論要件として要求されます。

### `indirect_strict_weak_order`

`std::indirect_strict_weak_order`は、イテレータの要素型による狭義弱順序関係を表すコンセプトです。

```{style=cppstddecl}
template<class F, class I1, class I2 = I1>
concept indirect_strict_weak_order =
  indirectly_readable<I1> && indirectly_readable<I2> &&
  copy_constructible<F> &&
  strict_weak_order<F&, iter_value_t<I1>&, iter_value_t<I2>&> &&
  strict_weak_order<F&, iter_value_t<I1>&, iter_reference_t<I2>> &&
  strict_weak_order<F&, iter_reference_t<I1>, iter_value_t<I2>&> &&
  strict_weak_order<F&, iter_reference_t<I1>, iter_reference_t<I2>> &&
  strict_weak_order<F&, iter_common_reference_t<I1>, iter_common_reference_t<I2>>;
```

`std::indirect_strict_weak_order<F, I1, I2>`は、`F`が`I1, I2`の要素型の狭義弱順序関係となる場合に`true`となります。`F, I1, I2`のオブジェクトを`f, i1, i2`とすると、`bool b = f(*i1, *i2)`のような呼び出しと変換が可能であり、結果が狭義弱順序関係を示すことを表します。

`std::strict_weak_order`と同様に、同値関係になっていることについては意味論要件として要求されます。

狭義弱順序関係とは、`<`による順序関係であって、比較不可能な値同士を同値として扱って順序付けするような順序関係です。これは、`std::sort`など標準ライブラリ内で並べ替えを行う際に仮定される順序関係として要求されます。

## カスタマイゼーションポイントオブジェクト

イテレータを介した操作を効率化するためのCPOが用意されます。

### `ranges::iter_move`

`std::ranges::iter_move`は、イテレータから要素をムーブ抽出するCPOです。

```cpp
auto example(std::input_iterator auto i) {
  // イテレータから要素をムーブし抽出する
  auto v = std::ranges::iter_move(i);

}
```

デフォルトの振る舞いは、`decltype(*i)`が左辺値参照型の場合は`std::move(*i)`と等しく、*prvalue*の場合は`*i`と等しくなります。

また、カスタマイゼーションポイントを備えており、ユーザー定義非メンバ`iter_move()`を呼び出すこともできます。

```cpp
struct I {
  int n = 0;

  // (1)
  int& operator*();


  // (2) HIdden friendsとして定義
  friend auto iter_move(I& self) -> int {
    return std::move(self).n;
  }
};

int main() {
  I i{20};

  auto n = std::ranges::iter_move(i); // (2)を呼び出す
}
```

このカスタマイゼーションポイントは主に、イテレータラッパを作成するときにラップ対象のイテレータがカスタマイズしている`iter_move()`を正しく呼び出すために利用します（そのままだと、`operator*`と`std::move()`が使用される）。

ではそもそもこの`ranges::iter_move`を使用する意味は何かというと、`decltype(*i)`が*prvalue*の場合にムーブを効率化するため（あるいは非効率化しないため）です。

`auto v = std::move(*i)`だと、`decltype(*i)`（この型を`V`とします）が*prvalue*の場合に一瞬`V&&`にキャストしてから`v`を初期化することになり、*prvalue*の実体化が発生します。これの何が問題なのかというと、*prvalue*が実体化してしまうと値のコピー省略保証の対象外になってしまいます。コピー省略が効く場合、`*i`の定義内`return`での`V`の構築が`v`で直接行われたかのようになり、不要なムーブコンストラクタの呼び出しを抑制することができます。

普通の関数で2つの構築方法の違いを見てみると次のようになります

```cpp
// 戻り値型がprvalueの関数
auto f() -> T;

int main() {
  // 一瞬prvalueが実体化してしまう
  T&& t1 = f();
  T t2 = std::move(t1); // コピー省略が妨げられる

  // prvalueを実体化させない
  T t3 = f(); // コピー省略により直接構築
}
```

`ranges::iter_move`のデフォルト動作の振る舞いの違いは、普通の関数で例示すると次のような感じです

```cpp
auto f() -> T;
auto g() -> T&; // ダングリングしないとする

void pseudo_iter_move(std::invocable auto&& func) {
  if constexpr (/*func()の戻り値型がprvalue？*/) {
    T t = func(); // コピー省略が期待できる
  } else {
    // 戻り値型が左辺値の場合
    T t = std::move(func());
  }
}

int main() {
  pseudo_iter_move(f);
  pseudo_iter_move(g);
}
```

この`f(), g()`がイテレータの`operator*()`に相当するわけです。このように、`ranges::iter_move`は`*i`の戻り値型の値カテゴリに応じてその振る舞いを切り替えることで、イテレータのデリファレンスとムーブの複合操作を自動的に効率化するものです。

### `ranges::iter_swap`

`ranges::iter_swap`は、2つのイテレータの参照先の値を`swap`するCPOです。

```cpp
template<std::forward_iterator I>
auto example(I i1, I i2) {
  // i1とi2の参照先をswapする
  std::ranges::iter_move(i1, i2);
}
```

このCPOにもカスタマイゼーションポイントがあり、ユーザー定義非メンバ`iter_swap()`を探してくれます（`ranges::iter_move`とほぼ同じ感じなのでサンプルコードは省略）。`ranges::iter_move`同様に、これはイテレータラッパにおいてラップ対象イテレータがもつ`iter_swap()`を適切に呼び出すために使用します。

これは単純には`std::ranges::swap(*i1, *i2)`の簡易構文でもあります。長さ的にもそう変わらないのにこれを使用する意味は何かというと、関節参照`*`を経由して`swap`するのが必ずしも正しくない場合があるためです。それはイテレータラッパにおいて起きうることで、`*i`が追加のことをしていたり*prvalue*を返したりする場合には`std::ranges::swap(*i1, *i2)`では元の範囲の上での`swap`を行うことが非効率となるかできなくなります。その場合にこのCPOを通じて`iter_swap()`経由で、元のイテレータを用いて直接`swap`することで元の範囲の上での`swap`を行います。これは最終的には非ラッパイテレータに対して`std::ranges::swap(*i1, *i2)`のような`swap`を行うことになるでしょう。重要なのはイテレータラッパで`swap`する時に`*`を使用しないところです。

そのようなイテレータには例えばムーブイテレータがあり、C++23では`zip_view`のイテレータも該当します。

また、非常に稀ではありますが、イテレータとその参照する範囲の種類・実装によっては、実際の値の交換よりも効率的にイテレータの指す要素の`swap`を行うことができる可能性があります。`iter_swap()`をカスタムすることでそのような最適化を有効にすることができる場合があるかもしれません。

## 射影関連のユーティリティ

射影（*projection*）とは、イテレータの要素を引き当てる際にその方法を指定する関数の事です。例えば、`pair<T, U>`を要素とする範囲から`T`だけに注目したい場合に`pair<T, U>& -> T&`のような関数を渡すことで要素の参照をカスタマイズします。これは、Rangeアルゴリズムにおいて活用されており、Rangeアルゴリズムでは引数列の最後の方で入力として使用する範囲のための射影を受け取るようになっています。

そのため、射影に関する制約やデフォルトの提供のために関連するユーティリティが用意されます。

### 射影操作の結果型

`std::projected<I, P>`はイテレータ型`I`と射影操作`P`を渡して、イテレータに対して射影を適用した結果を`indirectly_readable`な型として扱うためのクラステンプレートです。

```{style=cppstddecl}
namespace std {

  template<indirectly_readable I, indirectly_regular_unary_invocable<I> Proj>
  struct projected {
    using value_type = remove_cvref_t<indirect_result_t<Proj&, I>>;

    indirect_result_t<Proj&, I> operator*() const;  // 宣言のみ
  };
}
```

`std::projected<I, P>`は、射影`P`をイテレータ`I`に適用した結果を`::value_type`に持つとともに、`std::projected<I, P>`自体が`std::indirectly_readable`コンセプトを満たす型です。`std::projected<I, P>`が`indirectly_readable`となることで、イテレータ関連コンセプト（主に`indirectly_`から始まる系のコンセプト）を再利用することができます。

そのような利用はRangeアルゴリズムにおいて頻出します。

```{style=cppstddecl}
namespace std::ranges {
  template<input_iterator I, 
           sentinel_for<I> S,
           class Proj = identity,
           indirect_unary_predicate<projected<I, Proj>> Pred>
  constexpr I find_if(I first, S last, Pred pred, Proj proj = {});
}
```

例えばこの`std::ranges::find_if()`は述語（`pred`）を満たす最初の要素を`[first, last)`から見つけるものですが、要素を引き当てるのに射影操作を利用することができます（例えば`std::pair<T, U>`による範囲から`T`のみに注目して`find_if`する場合など）。その場合、`pred`に渡されるのは射影の結果型ですが、それを求めて`unary_predicate`のようなコンセプトに入力するのは面倒です。`std::projected`を用いることで`std::indirect_unary_predicate`の第二引数（`indirectly_readable`な型を要求）にそのまま入れることができ、制約を単純化することができます。

ただし、この性質上使用するのはコンセプトの文脈においてのみであり、実際にそのオブジェクトを作って間接参照することはできません。おそらくコンパイルエラーとなるはずです。

### `identity`

`std::identity`は引数をそのまま返す関数オブジェクトのクラス型です。

```{style=cppstddecl}
// <functional>内で定義される
namespace std {
  struct identity {
    template<class T>
    constexpr T&& operator()(T&& t) const noexcept {
      return std::forward<T>(t);
    }

    using is_transparent = unspecified;
  };
}
```

これは主に、Rangeアルゴリズムにおける射影操作のデフォルトとして利用されています。

```{style=cppstddecl}
namespace std::ranges {
  template<input_iterator I, 
           sentinel_for<I> S,
           class Proj = identity, // デフォルト射影
           indirect_unary_predicate<projected<I, Proj>> Pred>
  constexpr I find_if(I first, S last, Pred pred, Proj proj = {});
}
```

射影操作は便利ではありますが多くの場合は要素型をそのまま利用することになるため、デフォルトでは省略することができるようになっています。デフォルトの振る舞いは範囲の要素をそのまま引き当てるものであり、そのために`std::identity`が指定されます。

イテレータの文脈でよく使用されるものであるためここで紹介していますが、`std::identity`は`<functional>`に配置されています。

## イテレータアルゴリズムに関するコンセプト

ここまでに出てきたコンセプトなどを用いて、イテレータアルゴリズムで頻出する操作を制約するためのコンセプトが用意されます。ここのコンセプトは主に、イテレータを介して入力範囲を変更する（並び替える）場合の操作に関するものです。

### `indirectly_movable`

`std::indirectly_movable`は、イテレータの要素型を別のイテレータへムーブしつつ出力することができることを表すコンセプトです。

```{style=cppstddecl}
template<class In, class Out>
concept indirectly_movable =
  indirectly_readable<In> &&
  indirectly_writable<Out, iter_rvalue_reference_t<In>>;
```

`std::indirectly_movable<In, Out>`は、`indirectly_readable`な`In`から`indirectly_writable`な`Out`へムーブで出力できる場合に`true`となります。`In, Out`のオブジェクトを`in, out`とすると、`*out = std::move(*in)`のような操作が可能であることを表しています。

### `indirectly_movable_storable`

`std::indirectly_movable_storable`は、`indirectly_movable`の操作が中間オブジェクトを介しても可能であることを表すコンセプトです。

```{style=cppstddecl}
template<class In, class Out>
  concept indirectly_movable_storable =
    indirectly_movable<In, Out> &&
    indirectly_writable<Out, iter_value_t<In>> &&
    movable<iter_value_t<In>> &&
    constructible_from<iter_value_t<In>, iter_rvalue_reference_t<In>> &&
    assignable_from<iter_value_t<In>&, iter_rvalue_reference_t<In>>;
```

`std::indirectly_movable_storable<In, Out>`は、`indirectly_movable<In, Out>`であり`std::iter_value_t<In>`型の中間オブジェクトに`In`から一旦ストアしたあとでそこから`Out`へムーブ出力することができる場合に`true`となります。`In, Out`のオブジェクトを`in, out`とすると、次のような操作が可能であることを表しています

```cpp
auto tmp = *in;
...
*out = std::move(tmp);
```

このコンセプトには意味論要件が指定されています。関節参照可能な`In`のオブジェクト`i`について

- 次のように初期化された`obj`はこの直前の`*i`の値と等しい

```cpp
std::iter_value_t<In> obj(std::ranges::iter_move(i));
```

- この時、`std::iter_rvalue_reference_t<In>`が右辺値参照型を示す場合、この後の`*i`の値は有効だが未規定

ムーブという操作がその意味通りの振る舞いをすることを言っています。「有効だが未規定」というのは標準ライブラリ型のオブジェクトがムーブされた後の状態を指定する常套句であり、別の値を代入しない限りそのほとんどの操作は未定義となります。

### `indirectly_copyable`

`std::indirectly_copyable`は、イテレータの要素型を別のイテレータへコピーしつつ出力することができることを表すコンセプトです。

```{style=cppstddecl}
template<class In, class Out>
concept indirectly_copyable =
  indirectly_readable<In> &&
  indirectly_writable<Out, iter_reference_t<In>>;
```

`std::indirectly_copyable<In, Out>`は、`indirectly_readable`な`In`から`indirectly_writable`な`Out`へコピーして出力できる場合に`true`となります。`In, Out`のオブジェクトを`in, out`とすると、`*out = *in`のような操作が可能であることを表しています。

### `indirectly_copyable_storable`

`std::indirectly_copyable_storable`は、`indirectly_copyable`の操作が中間オブジェクトを介しても可能であることを表すコンセプトです。

```{style=cppstddecl}
template<class In, class Out>
concept indirectly_copyable_storable =
  indirectly_copyable<In, Out> &&
  indirectly_writable<Out, iter_value_t<In>&> &&
  indirectly_writable<Out, const iter_value_t<In>&> &&
  indirectly_writable<Out, iter_value_t<In>&&> &&
  indirectly_writable<Out, const iter_value_t<In>&&> &&
  copyable<iter_value_t<In>> &&
  constructible_from<iter_value_t<In>, iter_reference_t<In>> &&
  assignable_from<iter_value_t<In>&, iter_reference_t<In>>;
```

`std::indirectly_copyable_storable<In, Out>`は、`indirectly_copyable<In, Out>`であり`std::iter_value_t<In>`型の中間オブジェクトに`In`から一旦ストアしたあとでそこから`Out`へコピー出力することができる場合に`true`となります。`In, Out`のオブジェクトを`in, out`とすると、次のような操作が可能であることを表しています

```cpp
const auto tmp = *in;
...
*out = tmp;
```

このコンセプトには意味論要件が指定されています。関節参照可能な`In`のオブジェクト`i`について

- 次のように初期化された`obj`はこの直前の`*i`の値と等しい

```cpp
std::iter_value_t<In> obj(*i);
```

- この時、`std::iter_reference_t<In>`が右辺値参照型を示す場合、この後の`*i`の値は有効だが未規定

これも、コピーがその意味通りの結果を持つことを言っています。

### `indirectly_swappable`

`std::indirectly_swappable`は、2つのイテレータの間でその要素の`swap`が行えることを表すコンセプトです。

```{style=cppstddecl}
template<class I1, class I2 = I1>
concept indirectly_swappable =
  indirectly_readable<I1> && indirectly_readable<I2> &&
  requires(const I1 i1, const I2 i2) {
    ranges::iter_swap(i1, i1);
    ranges::iter_swap(i2, i2);
    ranges::iter_swap(i1, i2);
    ranges::iter_swap(i2, i1);
  };
```

`std::indirectly_swappable<I1, I2>`は、`indirectly_readable`な型`I1`と`I2`の要素型が`swap`可能である場合に`true`となります。その意味合いについては制約式が雄弁に物語っているかと思います。

制約式にあるように、このコンセプトを用いて制約したコンテキストにおいてのイテレータ要素の`swap`には`ranges::iter_swap`を用いた方がより適切です。

### `indirectly_comparable`

`std::indirectly_comparable`は、2つのイテレータの間でその要素の比較が行えることを表すコンセプトです。

```{style=cppstddecl}
template<class I1, class I2, class R, class P1 = identity, class P2 = identity>
concept indirectly_comparable =
  indirect_binary_predicate<R, projected<I1, P1>, projected<I2, P2>>;
```

`std::indirectly_comparable<I1, I2, R>`は、`indirectly_readable`な型`I1`と`I2`の要素型が二項関係`R`によって比較可能である場合に`true`となります。型`I1, I2, R`のオブジェクトを`i1, i2, r`とすると`bool b = r(*i1, *i2)`の様な呼び出しが可能出ることを表しており、この時に要素の引き当てに射影を使用することもできます。

`R`に相当する型としては`std::less<>`や`std::equal_to<>`があり、標準ライブラリでは主にその`range`版が利用されます。

### `permutable`

`std::permutable`は、イテレータ範囲の要素をムーブや`swap`によってin-placeで並べ替えできることを表すコンセプトです

```{style=cppstddecl}
template<class I>
concept permutable =
  forward_iterator<I> &&
  indirectly_movable_storable<I, I> &&
  indirectly_swappable<I, I>;
```

`std::permutable<I>`は、`I`が`forward_iterator`かつ`indirectly_movable_storable`かつ`indirectly_swappable`である場合に`true`となります。

ここでのin-placeで並べ替えとは、長さ`N`の範囲に対して1要素分程度の追加領域の使用によって並べ替えることができるという意味合いです。

### `mergeable`

`std::mergeable`は、2つのソート済みイテレータ範囲をマージしつつコピーして別のイテレータに出力可能であることを表すコンセプトです。

```{style=cppstddecl}
template<class I1, class I2,
         class Out, class R = ranges::less,
         class P1 = identity, class P2 = identity>
concept mergeable =
  input_iterator<I1> &&
  input_iterator<I2> &&
  weakly_incrementable<Out> &&
  indirectly_copyable<I1, Out> &&
  indirectly_copyable<I2, Out> &&
  indirect_strict_weak_order<R, projected<I1, P1>, projected<I2, P2>>;
```

`std::mergeable<I1, I2, Out, R, P1, P2>`は、`input_iterator`である`I1, I2`の範囲を`R`の比較によってマージし、結果要素を`Out`へコピーして出力することができる場合に`true`となります。また、その際に射影`P1, P2`を使用します。

非常に複雑ですが、`std::merge()`が可能であるための最小の要求を表しており、実際に`std::ranges::merge`で使用されています。

```{style=cppstddecl}
namespace std::ranges {
  template<input_iterator I1, sentinel_for<I1> S1, 
           input_iterator I2, sentinel_for<I2> S2,
           weakly_incrementable O, class Comp = ranges::less,
           class Proj1 = identity, class Proj2 = identity>
    requires mergeable<I1, I2, O, Comp, Proj1, Proj2>   // ここ
  constexpr
    merge_result<I1, I2, O>
      merge(I1 first1, S1 last1, I2 first2, S2 last2, O result, Comp comp = {}, Proj1 proj1 = {}, Proj2 proj2 = {});
}
```

### `sortable`

`std::sortable`はイテレータ範囲がソート可能であることを表すコンセプトです。

```{style=cppstddecl}
template<class I, class R = ranges::less, class P = identity>
concept sortable =
  permutable<I> &&
  indirect_strict_weak_order<R, projected<I, P>>;
```

`std::sortable<I, R, P>`は、`I`が`permutable`であり`R`が`I`の要素型について狭義弱順序関係を示す場合に`true`となります。また、その際に射影`P`を使用します。

これは標準ライブラリにおけるソート可能という要件をコンセプトに表したものです。ソートに伴う操作のためには`permutable`が必要であり、比較は狭義の弱順序によっていなければなりません。

これは`std::sort()`を行うための最小の要求であり、実際に`std::ranges::sort`で使用されています。

```{style=cppstddecl}
namespace std::ranges {
  template<random_access_iterator I, sentinel_for<I> S, 
           class Comp = ranges::less, class Proj = identity>
    requires sortable<I, Comp, Proj>  // ここ
  constexpr I sort(I first, S last, Comp comp = {}, Proj proj = {});
}
```

## 進行と距離

`std::advance()`等のイテレータを進行させる関数とイテレータ間の距離を求める`std::distance()`はイテレータコンセプトの導入とイテレータ定義の若干の変更に伴って新しいバージョンが追加されます。これらはすべて、`std::ranges`名前空間の下に配置されます。

### `std::ranges::advance`

`std::ranges::advance()`は`std::advance()`のC++20バージョンです。基本的には渡されたイテレータを指定されただけ進める処理であるところは変わっていません。また、`std::ranges::advance()`はすべてのオーバーロードにおいて渡されたイテレータを直接書き換えます。

この関数にはオーバーロードが3つあります。

```{style=cppstddecl}
// ranges::advance() 1
template<input_or_output_iterator I>
constexpr void advance(I& i, iter_difference_t<I> n);
```

1つ目は、イテレータ`i`を`n`進めるものです。これはほぼ従来の`std::advance()`そのままです。

`I`が`bidirectional_iterator`であるならば、`n`には負の数を指定することができます。

```cpp
#include <iterator>

int main() {
  // 何かしらの範囲オブジェクトとする
  std::ranges::range auto seq = {1, 2, 3, 4};

  std::bidirectional_iterator auto it = seq.begin();

  std::ranges::advance(it);     // itを1進める
  // *it == 2

  std::ranges::advance(it, 2);  // itを2進める
  // *it == 4

  std::ranges::advance(it, -3); // itを3戻す
  // *it == 1
}
```

この処理は入力イテレータのカテゴリによって変化します。

- `std::random_access_iterator` : `i += n`
- それ以外 : `n`回のインクリメント/デクリメント

このように、コンセプトによってそのイテレータに最適な方法で入力イテレータを進めます。

```{style=cppstddecl}
// ranges::advance() 2
template<input_or_output_iterator I, sentinel_for<I> S>
constexpr void advance(I& i, S bound);
```

2つ目は、イテレータ`i`を番兵`bound`の位置まで進めるものです。

```cpp
#include <iterator>

int main() {
  std::ranges::range auto seq = {1, 2, 3, 4};

  std::bidirectional_iterator auto it = seq.begin();
  auto bound = std::ranges::advance(it, 2);

  std::ranges::advance(it, bound);
  // *it == 3

  boud = seq.begin();
  
  std::ranges::advance(it, bound);
  // *it == 1
}
```

これも入力イテレータのカテゴリによってやることが変化します。

- `I, S`が`std::assignable_from<I&, S>`のモデルとなる : `i = std::move(bound)`
- `S, I`が`std::sized_sentinel_for<S, I>`のモデルとなる : `std::ranges::advance(i, bound - i)`
- それ以外 : `i == bound`となるまで`i`をインクリメント

`std::sentinel_for`の意味論要件によって、`i = bound`のような代入が可能（意味がある）かどうかを`std::assignable_from<I&, S>`を調べることによってチェックすることができます（できない場合はこれを満たさないようにしなければならないため）。`std::assignable_from<I&, S>`を満たしていなければ番兵のイテレータへの直接代入はできないので、他の方法によって`bound`の位置までイテレータを進めます。

注意としては、3つ目の処理が行われる場合は`bound`まで戻るのようなことはできない点で、したがってジェネリックな文脈でこのオーバーロードを後退のために使用するのは不適切です。これは、上2つの処理に該当しない場合は`bound`が`i`の前後どちらにあるかを判定することができないためで、この場合は後ろにあるものとみなして進行させています。

```{style=cppstddecl}
// ranges::advance() 3
template<input_or_output_iterator I, sentinel_for<I> S>
constexpr iter_difference_t<I>
  advance(I& i, iter_difference_t<I> n, S bound); 
```

3つめは、イテレータ`i`を`bound`までの間で`n`進めるものです。

```cpp
#include <iterator>

int main() {
  std::ranges::range auto seq = {1, 2, 3, 4};

  std::bidirectional_iterator auto it = seq.begin();
  auto bound = std::ranges::advance(it, 2);

  auto m = std::ranges::advance(it, 1, bound);
  // *it == 2、 m == 0

  auto m = std::ranges::advance(it, 10, bound);
  // *it == 3、 m == 9
}
```

このオーバロードにだけ戻り値があり、指定した`n`に対して進めなかった距離を返します。

これも入力イテレータのカテゴリによってやることが変化します。

- `S, I`が`std::sized_sentinel_for<S, I>`のモデルであり
    - $|n| \geq |bound - i|$の場合 : `std::ranges::advance(i, bound)`
    - $|n| < |bound - i|$の場合 : `std::ranges::advance(i, n)`
- それ以外の場合 : `i != bound`の間、最大$|n|$回のインクリメント/デクリメント

イテレータと番兵の間で距離が求められる（`std::sized_sentinel_for<S, I>`）ならば、そのサイズ情報を用いて前2つの`std::ranges::advance()`に処理を委ねますが、そうでないならば愚直に一歩づつ進行させます。

このように、`std::ranges::advance()`は`std::advance()`に比べるとコンセプトを用いて要件や効果を適切に定義していますが、その最も大きな違いは番兵という概念を考慮した進行ができるかどうかにあります。

### `std::ranges::next`

`std::ranges::next()`は`std::next()`のC++20バージョンです。基本的には渡されたイテレータを指定された分進めて返すものであることに変化はありません。

この関数にはオーバーロードが4つあります。

```{style=cppstddecl}
namespace std::ranges {
  // iを1つ進める
  template<input_or_output_iterator I>
  constexpr I next(I i) {
    ++i;
    return i;
  }

  // iをn進める
  template<input_or_output_iterator I>
  constexpr I next(I i, iter_difference_t<I> n) {
    std::ranges::advance(i, n);
    return i;
  }
}
```

まず最初の2つはイテレータ`i`を`n`進めるものです。1つ目のイテレータのみを取るオーバーロードはイテレータを1つだけ進めます。

```cpp
#include <iterator>

int main() {
  // 何かしらの範囲オブジェクトとする
  std::ranges::range auto seq = {1, 2, 3, 4};

  std::forward_iterator auto it = seq.begin();

  auto it2 = std::ranges::next(it);     // itを1進める
  // *it2 == 2
  // *it  == 1

  auto it3 = std::ranges::next(it, 2);  // itを2進める
  // *it3 == 4
  // *it  == 1
}
```

この2つはこれまでの`std::next()`とほぼ同様の振る舞いをします。`n`には負数を渡しても意図通りになりますが、この関数の意味的には常に正の値を渡すべきで、イテレータの後退をしたい場合は次の項の`std::ranges::prev()`を使用した方が良いでしょう。

```{style=cppstddecl}
namespace std::ranges {
  // iをboundまで進める
  template<input_or_output_iterator I, sentinel_for<I> S>
  constexpr I next(I i, S bound) {
    std::ranges::advance(i, bound);
    return i;
  }

  // iをboundまでの間でn進める
  template<input_or_output_iterator I, sentinel_for<I> S>
  constexpr I next(I i, iter_difference_t<I> n, S bound) {
    std::ranges::advance(i, n, bound);
    return i;
  }
}
```

残り2つのオーバーロードは番兵`bound`を取るもので、実際の進行の処理はどちらも`ranges::advance()`に委ねます。したがって、`ranges::advance()`で行われていたイテレータの性質に応じた最適な処理の選択はここでも行われます。

```cpp
#include <iterator>

int main() {
  std::ranges::range auto seq = {1, 2, 3, 4};

  std::bidirectional_iterator auto it = seq.begin();
  auto bound = std::ranges::advance(it, 2);

  auto it2 = std::ranges::next(it, bound);
  // *it2 == 3
  // *it  == 1

  auto it3 = std::ranges::next(it, 10, bound);
  // *it3 == 3
  // *it  == 1
}
```

### `std::ranges::prev`

`std::ranges::prev()`は`std::prev()`のC++20バージョンです。基本的には渡されたイテレータを指定された分戻して返すものであることに変化はありません。

この関数にはオーバーロードが3つあります。

```{style=cppstddecl}
namespace std::ranges {
  // iを1つ戻す
  template<bidirectional_iterator I>
  constexpr I prev(I i) {
    --i;
    return i;
  }

  // iをn戻す
  template<bidirectional_iterator I>
  constexpr I prev(I i, iter_difference_t<I> n) {
    std::ranges::advance(i, -n);
    return i;
  }
}
```

まず最初の2つはイテレータ`i`を`n`戻すものです。1つ目のイテレータのみを取るオーバーロードはイテレータを1つだけ戻します。

```cpp
#include <iterator>

int main() {
  // 何かしらの範囲オブジェクトとする
  std::ranges::range auto seq = {1, 2, 3, 4};

  std::bidirectional_iterator auto it = std::ranges::advance(seq.begin(), 3);

  auto it2 = std::ranges::prev(it);     // itを1戻す
  // *it2 == 3
  // *it  == 4

  auto it3 = std::ranges::prev(it, 2);  // itを2戻す
  // *it3 == 2
  // *it  == 4
}
```

この2つはこれまでの`std::prev()`とほぼ同様の振る舞いをします。`std::prev()`同様に、戻す距離`n`は戻したい数を指定するもので、負の値を指定すると戻るのではなく進むことになります。

\clearpage

```{style=cppstddecl}
namespace std::ranges {
  // iをboundまでの間でn戻す
  template<input_or_output_iterator I, sentinel_for<I> S>
  constexpr I prev(I i, iter_difference_t<I> n, S bound) {
    std::ranges::advance(i, -n, bound);
    return i;
  }
}
```

残りのオーバーロードは番兵`bound`を取るもので、`ranges::next()`同様に処理は`ranges::advance()`に委ねます。こちらには`bound`まで戻すオーバーロードは提供されません。なぜなら、`ranges::advance(i, bound)`が`bound`までの進行しか扱っていないためです。

```cpp
#include <iterator>

int main() {
  std::ranges::range auto seq = {1, 2, 3, 4};

  std::bidirectional_iterator auto it = std::ranges::advance(seq.begin(), 3);
  auto bound = seq.begin();
  
  auto it2 = std::ranges::prev(it, 10, bound);
  // *it2 == 1
  // *it  == 4
}
```

### `std::ranges::distance`

`std::ranges::distance()`は`std::distance()`のC++20バージョンです。基本的な役割も変化はなく、渡されたイテレータ間の距離を求めるものです。

この関数にはオーバーロードが3つあります。

```{style=cppstddecl}
namespace std::ranges {
  template<input_or_output_iterator I, sentinel_for<I> S>
    requires (!sized_sentinel_for<S, I>)
  constexpr iter_difference_t<I> distance(I first, S last);

  template<input_or_output_iterator I, sized_sentinel_for<I> S>
  constexpr iter_difference_t<I> distance(const I& first, const S& last);
}
```

まず2つは、イテレータ間距離を求めるものです。

```cpp
#include <iterator>

int main() {
  std::ranges::range auto seq = {1, 2, 3, 4};

  std::forward_iterator auto it = seq.begin();
  std::sized_sentinel_for<decltype(it)> auto end = seq.end();
  
  auto d1 = std::ranges::distance(it, end);
  // d1 == 4

  auto it2 = std::ranges::next(seq.begin(), 2);
  
  auto d2 = std::ranges::distance(it, it2);
  // d2 == 2

  auto d3 = std::ranges::distance(it2, it);
  // d3 == -2
}
```

1つ目のオーバーロードは`sized_sentinel_for`ではないイテレータと番兵のペアを受け取るためのもので、次のような結果を返します

- `first`から`last`に到達するのに必要なインクリメントの回数

2つ目のオーバーロードは`sized_sentinel_for`であるイテレータと番兵のペアを受け取るためのもので、差分操作によって効率的に距離を求めます

- `last - first`

2つ目のオーバーロードがイテレータと番兵の`const`参照を取っているのは、ムーブオンリーかつ`sized_sentinel_for`なイテレータを受けるためのものです。1つ目のオーバーロードがイテレータと番兵を値で受けているのは、そこで行う距離の計算の都合上ムーブオンリーイテレータを渡す意味がないためです。このため、ムーブオンリーかつ`sized_sentinel_for`ではないイテレータと番兵のペアの間では`std::ranges::distance()`を使用できません。

`std::ranges::distance()`にはもう一つオーバーロードがあり、`range`オブジェクトを直接渡してその長さを得ることができます。

```{style=cppstddecl}
namespace std::ranges {
  template<range R>
  constexpr range_difference_t<R> distance(R&& r);
}
```

使用例

```cpp
#include <iterator>

int main() {
  std::ranges::range auto seq = {1, 2, 3, 4};
  
  auto d = std::ranges::distance(seq);
  // d == 4
}
```

こちらのオーバーロードの処理では、`range`型`R`の満たす性質によって最適な操作が選択されます

- `R`が`sized_range`である : `static_cast<range_difference_t<R>>(ranges::size(r))`
- それ以外の場合 : `ranges::distance(ranges::begin(r), ranges::end(r));`

`R`が`sized_range`である、すなわち`ranges::size()`が利用可能である場合はその結果を返し、そうでない場合は先頭と終端のイテレータ/番兵のペアを`ranges::distance()`に渡して距離を求めます。

### ADLを無効化する関数定義

`advance(), next(), prev(), distance()`の4つの`ranges`関数には追加で特別な性質が規定されています。それは、あるコンテキストからこれらの関数名が見えているときに、ADLを無効化するという効果です。

```cpp
void foo() {
  // std::ranges名前空間を取り込む
  using namespace std::ranges;

  std::vector<int> vec{1,2,3};
  distance(begin(vec), end(vec)); // ranges::distance()が使用される
}
```

このような`distance()`の呼び出しは、従来ならばADL経由で`std::distance()`を発見し、そちらの方がより特殊化されている（テンプレートパラメータ1つで2引数受けている）ため`std::distance()`が選択されていたはずです。しかし、`ranges::distance()`はそれが見えているコンテキストで同名に対するADLを無効化するため、`std::distance()`は一切考慮されることなく`ranges::distance()`が呼ばれます。この効果は見えていなければ働かないため、最初の`using namespace std::ranges`をコメントアウトすると`std::distance()`がADL経由で呼ばれます。

これは、`std::ranges`にあるリファインされた関数を呼び出すつもりのところで意図せずに古い同名関数を呼び出さないようにするための仕組みです。これら4つの関数だけではなく、`std::ranges`名前空間に配置されている関数はほぼこの性質を持っています。

この性質は通常の関数定義では実現不可能であり、通常これらの関数は関数オブジェクトとして実装されます。

## 番兵型

C++20では終端イテレータが番兵としてイテレータとは別に扱われるようになっています。それに伴って、汎用的な2つの番兵型が標準に用意されます。

### `default_sentinel`

`std::default_sentinel`は、汎用的に使用可能な番兵です。

```{style=cppstddecl}
namespace std {
  struct default_sentinel_t {};

  inline constexpr default_sentinel_t default_sentinel{};
}
```

これは主に、`std::default_sentinel_t`をイテレータや範囲を定義する際の番兵型として使用し、実際の番兵値として`std::default_sentinel`を使用します。

```cpp
// 自作のイテレータ型とする
struct my_iterator {
  ...

  using difference_type = std::ptrdiff_t;


  // default_sentinel_tとの比較によって終端判定
  friend bool operator==(const my_iterator& lhs, std::default_sentinel_t rhs) {
    // 終端判定の処理
    ...
  }


  // 2項-によって距離を求める
  friend auto operator-(std::default_sentinel_t lhs, const my_iterator& rhs) -> difference_type {
    // 距離計算の処理
    ...
  }
  friend auto operator-(const my_iterator& lhs, std::default_sentinel_t rhs) -> difference_type {
    return -(rhs - lhs);
  }

};
```

イテレータ型`I`に対して、`std::default_sentinel_t`との比較を定義することで`std::sentinel_for<I>`となり、`std::default_sentinel_t`との間で差分操作（`operator-`）を定義することで`std::sized_sentinel_for<I>`となります。

なお、`operator==`はC++20一貫比較仕様によって逆順の`==`と対応する`!=`が導出されますが`operator-`にはそのような仕組みがなく、`std::sized_sentinel_for`は`-`に対して引数順を入れ替えたものも要求するため、番兵-イテレータとイテレータ-番兵の2パターンの定義が必要です。とはいえ、例にあるように一方はもう片方を利用することで簡単に実装できるはずです。

`std::default_sentinel`はこのように定義したイテレータと番兵の比較や差分が必要な時の値として使用します。

```cpp
// 上記my_iteratorを使う処理
auto use_myiter(my_iterator it) {
  // default_sentinelとの比較による終端チェック
  while (it != std::default_sentinel) {
    ...

    // 終端との間の距離の計算
    std::ptrdiff_t d = std::default_sentinel - it;

    ...
  }
}
```

標準ライブラリ内でも、後述する`std::counted_iterator`などで使用されています。

### `unreachable_sentinel`

`std::unreachable_sentinel`は、イテレータの終端チェックをスキップするために使用できる特殊な番兵型です。

使用可能なところは非常に限られおり、イテレータを用いた検索などで範囲を走査する際に、終端チェックとは別の条件によって終端に到達する前に範囲の走査が終了することが確実にわかっている場合に使用できます。

```cpp
#include <iterator>

// 文字範囲から'a'の位置を検索する例
int main() {
  // 'a'を必ず含む入力
  std::string str = "unreachable_sentinel";
  
  // 終端チェックがスキップされることで高速な検索が可能となりうる
  // もし入力範囲に'a'が含まれていないと未定義動作
  auto pos = std::ranges::find(i, std::unreachable_sentinel, 'a');
  // *pos == 'a' （5文字目）
}
```

`std::unreachable_sentinel`の`operator==`は任意のイテレータとの比較が可能になっており、それは常に`false`を返します。

```{style=cppstddecl}
namespace std {
  struct unreachable_sentinel_t {
    
    // 任意のイテレータとの比較を可能とする
    template<weakly_incrementable I>
    friend constexpr bool operator==(unreachable_sentinel_t, const I&) noexcept { 
      return false; // 終端チェックはいつも終端に到達しない
    }
  };

  inline constexpr unreachable_sentinel_t unreachable_sentinel{};
}
```

これによって、範囲`for`やアルゴリズムに隠れているものも含めたイテレータの終端チェック処理では常に`false`（`!=`を使用する場合`true`）が帰るため、最適化によってその比較を除去することが可能となります。

ただし、実際の終端に到達してもイテレータの進行は止まらないため、事前条件（範囲の操作はその終端に到達する前に終了する）が満たされない場合は容易に未定義動作を引き起こすことになります。利用する際は慎重な検討が必要です。

あるいは、イテレータから番兵を分離することでこういうことが可能になるという例の一つとして見るのが良いかもしれません・・・

## `common_iterator`

`std::common_iterator`はイテレータ型と番兵型が異なる場合にそれらをラッピングすることで型を共通化するイテレータラッパです。これは主に、C++17以前の番兵型を考慮していないコードにイテレータ型と番兵型が異なるイテレータ範囲を入力するのに使用します。

例えば、先ほどの`std::unreachable_sentinel`をC++17までのアルゴリズム関数で使用したいとします。

```cpp
#include <iterator>

int main() {
  // 'a'を必ず含む入力
  std::string str = "unreachable_sentinel";

  // 引数型が一致しないため、そのままだと使用できない
  auto pos = std::find(str.begin(), std::unreachable_sentinel, 'a');  // ng

  using CI = std::common_iterator<std::string::iterator, std::unreachable_sentinel_t>;

  // common_iteratorでラップする
  CI it = str.begin();
  CI end = std::unreachable_sentinel;

  auto pos = std::find(it, end, 'a');  // ok
}
```

そのまま渡すと、`std::find`のテンプレートパラメータは1つのパラメータでイテレータ範囲の先頭と終端を受けていることから、イテレータ型（第1引数 `std::string::iterator`）と番兵型（第2引数 `std::unreachable_sentinel_t`）が異なるためコンパイルエラーとなります。そこで、`std::common_iterator`を用いてそれらを包むことでイテレータ型を共通化することができ、このようなエラーを回避することができます。

`std::common_iterator<I, S>`は`I`にイテレータ型、`S`に番兵型を指定し、`I, S`のオブジェクトどちらからも構築でき、どちらのオブジェクトも保持することができます。

　

```{style=cppstddecl}
namespace std {
  // common_iteratorの定義例
  template<input_or_output_iterator I, sentinel_for<I> S>
    requires (!same_as<I, S> && copyable<I>)
  class common_iterator {
    ...

    constexpr common_iterator(I i); // イテレータを受ける

    constexpr common_iterator(S s); // 番兵を受ける

    ...

  private:
    variant<I, S> v_; // 説明専用メンバ
  };
}
```

`std::common_iterator`は専ら後方互換のためにあるものなので、イテレータ型`I`の性質をそのまま受け継ぐことはせず、最も強くても`forward_iterator`にしかなりません。`I`に`random_access_iterator`を指定した時でも`+ -`や比較は利用できず、`--`などで後退することもできなくなります。これは、`std::common_iterator<I, S>`をC++20イテレータとして扱ったときでも++17イテレータとして扱ったときでも変わりません。

したがって、`std::common_iterator<I, S>`では基本的なイテレータ操作しか利用できません。

|利用可能な操作|意味|
|---|---|
|`*`|間接参照|
|`->`|メンバアクセス|
|`++`|インクリメント（進行）|
|`==/!=`|終端チェック|
|`-`|距離を求める|
|`iter_move`|`I`の`iter_move`呼び出し|
|`iter_swap`|`I`の`iter_swap`呼び出し|

これらの操作のうち`->`と`-`は`I`（と`S`）で利用可能な場合にのみ有効化されます。

## `counted_iterator`

`std::counted_iterator`はイテレータを指定されたカウント数の範囲を指すものへと変換するイテレータラッパです。元の範囲の部分範囲を簡易に得る時に使用できます。

```cpp
#include <iterator>

bool is_odd(int n) {
  return n % 2 == 1;
}

int main() {
  // 何かしらの範囲オブジェクトとする
  std::ranges::range auto seq = {1, 3, 5, 7, 10, 11, 12};

  // seqの先頭から4要素を参照するイテレータを得る
  std::counted_iterator it{std::ranges::begin(seq), 4};

  // 範囲内の数値が全て奇数かを調べる
  bool b = std::ranges::all_of(it, std::default_sentinel, is_odd);
  // b == true
}
```

`std::counted_iterator`に対する番兵は`std::default_sentinel`であるため、終端イテレータを計算したり取得する必要がありません。元の範囲から部分範囲を得たい場合に、イテレータを取得してコピーして足して・・・のようなことをやるよりも簡易に同じことを達成できます。

```{style=cppstddecl}
namespace std {
  // counted_iteratorの宣言例
  template<input_or_output_iterator I>
  class counted_iterator {
    ...

    // イテレータとカウントを受け取るコンストラクタ
    constexpr counted_iterator(I x, iter_difference_t<I> n); 

    ...
  };
}
```

`std::counted_iterator`はイテレータ型`I`をテンプレートパラメータとして指定しますが、コンストラクタ引数からのクラステンプレート実引数推定（C++17 CTAD）によって基本的にはテンプレートパラメータの指定を省略することができます。

動作イメージは、コンストラクタに渡されたカウント値を出発点として、進行（`++`）の度にカウントを1減らしていきます。カウントが`0`になったときに終端に到達し、`std::default_sentinel`との比較（`==`）が`true`を返すようになります。すなわち、コンストラクタに渡すカウント値には、同時に渡すイテレータの位置も含めた範囲の要素数（生成する部分範囲の長さ）を渡すようにします。

```cpp
#include <iterator>

int main() {
  // 何かしらの範囲オブジェクトとする
  std::ranges::range auto seq = {1, 3, 5, 7, 9, 11, 13, 15};

  // seqの先頭から3要素を参照するイテレータ
  std::counted_iterator it{std::ranges::begin(seq), 3};
  // 内部カウント : 3
  
  ++it;
  // 内部カウント : 2
  
  it++;
  // 内部カウント : 1
  bool b1 = it == std::default_sentinel;
  // b1 == false
  
  ++it;
  // 内部カウント : 0

  bool b2 = it == std::default_sentinel;
  // b2 == true
}
```

ただし、`std::counted_iterator`は終端を通り越した場合のケアを特にしてくれません。終端チェックを省略して進行していると元の範囲の終端すら飛び越える危険性があります。

`std::counted_iterator<I>`はイテレータ型`I`の性質を可能な限り継承しようとし、`std::counted_iterator<I>`のイテレータカテゴリは`I`のカテゴリと同じになります。例えば、`I`が`contiguous_iterator`の時でも`std::counted_iterator<I>`は`contiguous_iterator`となります。これによって、後退（`--`）や和による進行（`+ +=`）や添字（`[]`）などが`I`のカテゴリ次第で利用可能となります。

ただし、`std::counted_iterator<I>`は、C++20イテレータとしてみた時は`I`のC++20イテレータとしての性質を、C++17イテレータとしてみた時はC++17イテレータとしての性質をそれぞれ継承します。もし`I`についてのそれらが異なる場合は`std::counted_iterator<I>`のそれら性質も異なることになります。

\clearpage

# コンテナ

ここでは、標準ライブラリのうちコンテナ関連のC++20での追加や変更を見ていきます。

## 連想コンテナ関連

連想コンテナとは次のものです

- 連想コンテナ
    - 順序付
      - `std::set`
      - `std::map`
      - `std::multiset`
      - `std::multimap`
    - 非順序
      - `std::unordered_set`
      - `std::unordered_map`
      - `std::unordered_multiset`
      - `std::unordered_multimap`

特に記載がない場合、ここで紹介される機能はこれら全ての連想コンテナに対して追加されます。

### `.contains()`

`.contains()`は、連想コンテナにある要素が含まれているかを調べる関数です。

従来同じことをしようとすると`.find()`の戻り値を`.end()`と比較する、としなければならず少し手間がかかっていました。単に要素の存在チェックをしたいだけの場合は少し不便でしたが、それが解消されます。

```cpp
#include <set>

int main() {
  std::set set = {1, 3, 5, 7, 9};

  // 従来の方法 
  {
    bool exists = set.find(5) != set.end();
  }

  // C++20から
  {
    bool exists = set.contains(5);
  }
}
```

これによって、やりたいことを簡潔に書くことができるとともに、`contains`というわかりやすい名前がついていることによってやっていることが明確になります。

なお、`.contains()`の計算量は`find()`と同じ（要素数`N`として、`unordered`なら定数or最悪`O(N)`、それ以外なら`N`に対して対数）になります。

C++23では、この関数を任意の`range`に対して一般化した`std::ranges::contains`アルゴリズムが追加されます。

### 透過的な検索

透過的な検索とは、連想コンテナの要素検索（引き当て）時に、その`key_type`と異なる型のオブジェクトを渡して検索する機能です。

C++14では、順序付連想コンテナ（`unordered`でないもの）の検索系関数に対してこの透過的検索サポートが追加されました。C++20では、非順序連想コンテナ（`unordered`なもの）に対しても拡大されます。

検索系関数とは次のものです

- `.find()`
- `.count()`
- `.lower_bound()`
- `.upper_bound()`
- `.equal_range()`

例えば、`std::map::find()`は次の2つのオーバーロードがC++14からあります

```cpp
// 通常の検索
iterator find(const key_type& x);


// 透過的検索
template <class K>
iterator find(const K& x);
```

この2つ目の関数テンプレート`.find()`が透過的検索を行うための関数で、連想コンテナのキー型（`key_type`）と異なる型のオブジェクトをとって検索を行うことができます。これによって、そのようなことをしたい場合に`key_type`に一度変換してから渡す必要性がなくなり、利便性が増すとともに不要なオブジェクトの構築を回避することで効率化できます。

おそらくこのことは、`std::string`をキーとする連想コンテナを使用する際によく出会うことになるでしょう。

```cpp
#include <map>

using namespace std::literals;

int main() {
  std::map<std::string, int> map = { {"1", 1}, {"2", 2}, {"3", 3} };

  bool b1 = map.find("1"sv) != map.end(); // ng
  bool b2 = map.find("4") != map.end();   // ok、ただし一時オブジェクトが作成されている
}
```

`std::string_view`から`std::string`の変換は、暗黙的なアロケーションが伴うこともあって暗黙変換が禁止されており、それによってエラーになります。文字列リテラルを渡す方はコンパイルは通りますが、`std::string`の一時オブジェクトが作成されており、非効率になっています。

これは非順序連想コンテナにおいても同様で、透過的な検索はこのような問題を回避しこの例のどちらも一時オブジェクトの作成を回避して意図通りになるようにするものですが、そのままC++20でコンパイルしても同じ結果になります。透過的な検索を有効化するには追加の手続きが必要となるためです。

透過的な検索は、連想コンテナの比較関数型に`is_transparent`メンバ型が定義され、`key_type`と異なる型との比較をサポートしている必要があり、非順序連想コンテナではそれに加えてハッシュクラスに`is_transparent`メンバ型と`key_type`と異なる型のオブジェクトからの一貫したハッシュ生成をサポートしている必要があります。

順序付連想コンテナについては、`std::less<>`もしくは`std::ranges::less`などの比較関数を用いることで有効化できます。この2つ（及びその同種）は比較がテンプレートで定義されていてメンバ型に`is_transparent`を持っています。

```cpp
// 比較関数型にranges::lessを使用
template<typename Key, typename Value>
using hetero_map = std::map<Key, Value, std::ranges::less>;

int main() {
  hetero_map<std::string, int> map = { {"1", 1}, {"2", 2}, {"3", 3} };

  // どちらも、直接比較によって検索される
  bool b1 = map.find("1"sv) != map.end(); // ok
  bool b2 = map.find("4") != map.end();   // ok
}
```

非順序連想コンテナでも比較は同様ですが、ハッシュクラスを別に用意しなければなりません。ハッシュクラスの場合、異なる型の間での一貫したハッシュ計算を実装するのが困難そうですが、`std::string`を始めとする文字列の場合は入り口さえ用意すれば共通の処理によって計算可能です。

```cpp
struct string_hash {
  using hash_type = std::hash<std::string_view>;

  // 型はなんでもいいものの必須
  using is_transparent = void;

  // string_viewを介して文字列をハッシュ化
  std::size_t operator()(std::string_view str) const {
    return hash_type{}(str);
  }
};


// ハッシュクラスと比較関数型を入れ替えたunordered_map
template<typename Key, typename Value>
using hetero_umap = std::unordered_map<Key, Value, string_hash, std::ranges::equal_to>;

int main() {
  hetero_umap<std::string, int> map = { {"1", 1}, {"2", 2}, {"3", 3} };

  // どちらも、直接比較によって検索される
  bool b1 = map.find("1"sv) != map.end(); // ok
  bool b2 = map.find("4") != map.end();   // ok
}
```

このようにすることで、全ての連想コンテナにおいて用意されている透過的検索関数を有効化できます。

このような透過的な検索を行う関数群のことを、*Heterogeneous (comparison) lookup*や*Heterogeneous Overload*と呼んだりもします。

これらの*Heterogeneous Overload*はC++14以降徐々に連想コンテナ全体に対して波及しています。

|関数|順序付連想コンテナ|非順序連想コンテナ|備考|
|---|---|---|---|
|`find()`|C++14|C++20||
|`count()`|C++14|C++20||
|`lower_bound()`|C++14|C++20||
|`upper_bound()`|C++14|C++20||
|`equal_range()`|C++14|C++20||
|`erase()`|C++23|C++23||
|`extract()`|C++23|C++23||
|`insert()`|C++26|C++26|非`multi`の`set`系のみ|
|`insert_or_assign()`|C++26|C++26|非`multi`の`map`系のみ|
|`try_emplace()`|C++26|C++26|非`multi`の`map`系のみ|
|`operator[]`|C++26|C++26|非`multi`の`map`系のみ|
|`bucket()`|-|C++26|非順序のみ|
|`contains()`|C++20|C++20|C++20で追加|

※ C++26予定のものはまだ確定していません。

### 非順序連想コンテナの比較の調整

これは非順序連想コンテナの比較演算子（`== !=`）に対する変更です。

従来の非順序連想コンテナの比較演算子の規定では、2つの非順序連想コンテナ間の比較を行う際に、その比較関数型（`Pred`）とハッシュクラス（`Hash`）が同一の振る舞いをすることが求められており、そうならない場合は未定義動作とされていました。しかし、`Hash`に関しては異なるシードによってランダム化されたハッシュを用いるユースケースがあり、それは正当なものであるはずで、この規定はそのようなカスタムハッシュを排除していました。

C++20からは、非順序連想コンテナの比較演算子においてハッシュクラスの振る舞いの同一性が要求されなくなり、そのようなカスタムハッシュの利用が可能となります。

```cpp
#include <unordered_map>

// ハッシュを混ぜる
void hash_combine(std::size_t& seed, std::size_t hash) {
  seed ^= hash + 0x9e3779b97f4a7c15llu + (seed << 12) + (seed >> 4);
}

// ハッシュをコンテナごとにランダム化
template<typename T>
struct randomized_hash {
  // ランダムな攪乱用ハッシュ値
  std::size_t seed = std::hash<unsigned int>{}(std::random_device{}());

  std::size_t operator()(const T& v) const {
    std::size_t h = 0;

    // ランダムなハッシュ値と混合する
    hash_combine(h, std::hash<T>{}(v));
    hash_combine(h, seed);

    return h;
  }
};

template<typename Key, typename Value>
using rnd_umap = std::unordered_map<Key, Value, randomized_hash<Key>>;

int main() {
  // 型は同じだが異なるハッシュを使用する
  rnd_umap<std::string, int> map1 = { {"1", 1}, {"2", 2} };
  rnd_umap<std::string, int> map2 = { {"1", 1}, {"2", 2} };

  bool is_equal = map1 == map2; // ok
  // is_equal == false
}
```

これは厳密には、以前のバージョン（C++11）に対する欠陥の解決です。実際、ほとんどの処理系では当初の実装からこうなっていたようです。

## `erase/erase_if`

`std::erase()`、`std::erase_if()`は任意のコンテナから要素を削除する関数です。この関数はすべてのコンテナに対して特殊化されていて、コンテナごとに最適な削除処理を行うことができます。

コンテナ型を`C`とすると、おおよそ次のように宣言されています。

```{style=cppstddecl}
namespace std {
  template<class T, class U>
  constexpr typename C::size_type
    erase(C& c, const U& v);
    
  template <class T, class Predicate>
  constexpr typename C::size_type
    erase_if(C& c, Predicate pred);
}
```

ここでの`C`は説明のためのもので、テンプレートパラメータではありません。

`std::erase(c, v)`はコンテナ`c`から値`v`と同じ要素を削除し、`std::erase_if(c, pred)`はコンテナ`c`から述語`pred`が`true`を返す要素を削除します。戻り値はどちらも、削除した要素数を返します。

```cpp
#include <vector>
#include <list>

int main() {
  std::vector vec = {1, 2, 3, 4, 5, 6};
  std::list list = {1, 2, 3, 4, 5, 6};

  // 6を削除
  auto n1 = std::erase(vec, 6);
  auto n2 = std::erase(list, 6);
  // n1 == n2 == 1

  auto even = [](int n) { return n % 2 == 0; };

  // 偶数を削除
  auto n3 = std::erase_if(vec, even);
  auto n4 = std::erase_if(list, even);
  // n3 == n4 == 2
}
```

従来コンテナの削除処理はコンテナごとに最適な方法がまちまちでした。例えば`std::vector`ではErase-Removeイディオムが使用される一方で、`std::list`ではメンバ関数`.erase()`を使用し、連想コンテナでは先頭から見ていく必要があったりしました。

`std::erase()`、`std::erase_if()`は上記例のように削除処理をコンテナ型によらず統一的に書きつつ、実際の処理は各コンテナ型に合わせて最適な方法で実行してくれるものです。

なお、コンテナ型に`std::string`が含まれている一方で、`std::erase()`は連想コンテナに対しては用意されません。

## `to_array()`

`std::to_array()`は、生配列を`std::array`に変換するキャスト関数です。

```cpp
#include <array>

int main() {
  // CTADによって推論されるもののcharの配列にならない
  std::array a1 = {"array"};
  // decltype(a1) : std::array<const char*, 1>  

  // charの配列として変換
  std::array a2 = std::to_array("array");
  // decltype(a2) : std::array<char, 6>

  std::array a3 = std::to_array({1, 2, 3});
  // decltype(a3) : std::array<int, 3>

  // 要素型の変換
  std::array a4 = std::to_array<short>({1, 2, 3});
  // decltype(a4) : std::array<short, 3>
  
  // 集成体の配列の作成
  std::array a5 = std::to_array<std::array<int, 1>>({ {1}, {2} });
  // decltype(a5) : std::array<std::array<int, 1>, 2>
}
```

基本的には`std::to_array(arr)`のように生配列`arr`をそのまま渡して使用しますが、`std::to_array<T>(arr)`のようにして変換後の`std::array<T, N>`の要素型`T`を指定し、要素型の変換を同時に行うこともできます。

この例はすべて右辺値ですが、左辺値の入力に対しては要素をコピーして`std::array`を作成します。

```cpp
#include <array>

int main() {
  int raw_arr[] = {1, 2, 3};


  // 各要素をコピーして変換
  std::array arr = std::to_array(raw_arr);
  // decltype(arr) : std::array<int, 3>
}
```

文字列リテラルを`char`の配列として扱いたいときや関数から配列を返したいときなどに役立つかもしれません。`return std::to_array(arr);`とする場合は、コピー省略が働くことも期待できます。

なお、`std::to_array()`は1次元配列専用で、多次元配列には非対応です。

## `ssize()`

`std::ssize()`はコンテナの要素数を符号付き整数型で取得するものです。これは、`<iterator>`に配置されます。

```cpp
#include <iterator>

int main() {
  std::vector vec = {1, 2, 3};

  // size()は符号無し整数型で取得
  std::unsigned_integral auto l1 = std::size(vec);
  // ssize()は符号付き整数型で取得
  std::signed_integral   auto l2 = std::ssize(vec);
  // l1 == l2 == 3
}
```

これは、`std::size()`及びコンテナの`.size()`が符号無し整数型を返しているのが問題となる場合にそれを簡単に回避するためのものです。

コンテナの要素のイテレーションには範囲`for`が利用されますが、インデックスを取得したいなどの事情によって通常の`for`文を使用することもあります。その場合、要素数分ループするためにコンテナの`.size()`から要素数を取得してループの終了条件を指定するコードがよく書かれます。一方で、ループのインデックスには単に`int`などの符号付き整数が使用されます。

```cpp
// 連続して同じ要素が並んでいる箇所がある場合にtrueを返す関数
template<typename C>
bool has_repeated_values(const C& container) {
  // インデックスが欲しいので通常のforを使用
  for (int i = 0; i < container.size() - 1; ++i) {
    //                ^^^^^^^^^^^^^^^^^^^^
    if (container[i] == container[i + 1]) return true;
  }
  return false;
}
```

この場合に`container`の要素数が0だと何が起こるでしょうか？`0ull - 1`は`std::size_t`の最大値を返し、ループはコンテナ範囲を大きく超えてイテレートしようとします。つまりバッファオーバーランが起こります。

仮に`.size() - 1`をしていなくても、符号付き整数（`i`）と符号無し整数（`.size()`）の比較は暗黙変換が入った結果意図通りにならなくなる場合があり危険なので、多くのコンパイラはここに警告を発します。警告を抑制しようとすると、`.size()`を`static_cast<int>()`とかしなければならず、かなり面倒になります。

このような時に`std::ssize()`を使用すると、それらの問題を回避し意図通りの振る舞いを手に入れることができます。

```cpp
// 連続して同じ要素が並んでいる箇所がある場合にtrueを返す関数
template<typename C>
bool has_repeated_values(const C& container) {
  // インデックスが欲しいので通常のforを使用
  for (int i = 0; i < std::ssize(container) - 1; ++i) {
    if (container[i] == container[i + 1]) return true;
  }
  return false;
}
```

`std::ssize()`の使用によって`container`の要素数が0の時も、`0ll - 1`は`-1`になり、正の整数値`i`との比較は`false`となるため意図通りにループは終了します。

## リストの削除関数の戻り値型変更

リストとは`std::list`と`std::forward_list`のことで、削除関数とは次の3つのことです

- `.remove()`
    - 指定された要素を削除
- `.remove_if()`
    - 条件に合う要素を削除
- `.unique()`
    - 重複している要素を削除

これらの関数はどれも要素の削除を行うものでC++17までは戻り値なしでしたが、C++20からは削除された要素数を返すようになります。

```cpp
#include <forward_list>

int main() {
  std::forward_list li = {1, 2, 2, 3};

  bool removed = li.unique() != 0ull;
  // removed == true
}
```

これによって、これらの関数では実際に削除が行われたかどうか？を簡単に確認できるようになります。

C++17までは、これら削除操作の前にコンテナの要素数（`.size()`）を保存しておいて削除後に比較、とすることでチェックすることができました。しかし、`std::forward_list`はその要素数を定数時間で求めることができないため、同様の方法だと非効率になり得ます。これらの削除操作が削除した要素数を返すようにすることで、`std::forward_list`においても効率的に、かつ単純に削除が行われたかどうかを検出できるようにされました。

`std::list`にも変更が及んだのは、リストという種類のコンテナの操作について一貫性を重視したためです。

\clearpage

# アルゴリズム

`<algorithm>`と`<numeric>`にあるイテレータ/数値アルゴリズムにいくつか新顔が追加されます。

## `shift_left/shift_right`

`std::shift_left/std::shift_right`アルゴリズムは、範囲の要素を左/右に指定した数だけシフトさせる（ずらす）ものです。

```{style=cppstddecl}
// 宣言例
namespace std {
  template<class ForwardIterator>
  constexpr ForwardIterator
    shift_left(ForwardIterator first,
               ForwardIterator last,
               typename iterator_traits<ForwardIterator>::difference_type n);

  template<class ForwardIterator>
  constexpr ForwardIterator
    shift_right(ForwardIterator first,
                ForwardIterator last,
                typename iterator_traits<ForwardIterator>::difference_type n);
}
```

`ForwardIterator`と指定されている通り、どちらも入力は最低でも*forward iterator*である必要があります。第3引数`n`に正の値でシフト量を指定します。

```cpp
#include <algorithm>

int main() {
  using std::ranges::subrange;

  {
    std::vector vec = {1, 2, 3, 4, 5};

    // 戻り値はシフト後の終端位置
    auto end = std::shift_left(vec.begin(), vec.end(), 2);

    for (int n : subrange{vec.begin(), end}) {
      std::cout << n << ", ";
    }
  }
  std::cout << '\n';
  {
    std::vector vec = {1, 2, 3, 4, 5};

    // 戻り値はシフト後の開始位置
    auto begin = std::shift_right(vec.begin(), vec.end(), 2);

    for (int n : subrange{begin, vec.end()}) {
      std::cout << n << ", ";
    }
  }
}
```

```{style=planetext}
3, 4, 5, 
1, 2, 3, 
```

このアルゴリズムは循環シフトではないため、範囲外に出た値は捨てられ、シフト後の左端/右端`n`の領域は未規定の状態（ムーブ後オブジェクト）となります。戻り値はシフト後の無効領域と有効領域の境界を指すイテレータが返され、シフト後範囲へのアクセスのために使用できます。

`n`は符号付き整数なので負の値を指定できますが、負の値の指定はこれらの関数の事前条件違反で未定義動作となります。`n`の型がイテレータの`difference_type`なのは、イテレータが*bidirectional*である時に`std::prev()`による最適化が可能で、その引数型に合わせるためのようです。

`std::shift_left/std::shift_right`の動作例（結果のみ）

```{style=planetext}
input : [1, 2, 3, 4, 5]

shift_left(1)   : [2, 3, 4, 5, ?]
shift_right(1)  : [?, 1, 2, 3, 4]
shift_left(4)   : [5, ?, ?, ?, ?]
shift_right(4)  : [?, ?, ?, ?, 1]
shift_left(0)   : [1, 2, 3, 4, 5]
shift_right(0)  : [1, 2, 3, 4, 5]
```

?になっているところは未規定の状態です。アクセスしないようにしましょう。

なお、これには対応するRangeアルゴリズム（`std::ranges`名前空間のもの）がC++20時点では提供されず、C++23から利用可能になります。導入がほぼ同時期だったので漏れてしまったようです。一方で、並列アルゴリズム（第1引数で`ExecutionPolicy`を取るもの）は利用可能です。

入力範囲の要素数を`N`とすると、`std::shift_left`は最大で`N - n`回の代入が行われ、`std::shift_right`は`N - n`回の代入あるいは`swap`が行われます。

## `lexicographical_compare_three_way`

`std::lexicographical_compare_three_way`は2つの範囲を辞書式順序（*lexicographical order*）によって三方比較するアルゴリズムです。

```{style=cppstddecl}
namespace std {
  template<class InputIterator1, class InputIterator2, class Cmp>
  constexpr auto
    lexicographical_compare_three_way(InputIterator1 b1, InputIterator1 e1,
                                      InputIterator2 b2, InputIterator2 e2,
                                      Cmp comp)
      -> decltype(comp(*b1, *b2));

  template<class InputIterator1, class InputIterator2>
  constexpr auto
    lexicographical_compare_three_way(InputIterator1 b1, InputIterator1 e1,
                                      InputIterator2 b2, InputIterator2 e2);
}
```

辞書式順序とは、2つの範囲の先頭から対応する要素を比較していき、最初に異なる要素の順序によって2つの範囲の順序付けを行う方法です。この比較は三方比較（`<=>`）によるため、戻り値は`<compare>`にある3つの比較カテゴリ型のいずれかの値として得られます（`bool`値ではありません）。

そのため、一番最後の引数で比較をカスタマイズできますが、その戻り値型は`bool`ではなく比較カテゴリ型である必要があります。

```cpp
#include <algorithm>

void str_cmp(std::string_view lhs, std::string_view rhs) {
  auto comp = std::lexicographical_compare_three_way(lhs.begin(), lhs.end(), rhs.begin(), rhs.end());

  std::cout << "> : " << (comp > 0) << std::endl;
  std::cout << "= : " << (comp == 0) << std::endl;
  std::cout << "< : " << (comp < 0) << std::endl;
}

int main() {
  std::cout << std::boolalpha;

  str_cmp("abc", "abcd");
  std::cout << "---\n";
  str_cmp("abcd", "abc");
  std::cout << "---\n";
  str_cmp("abc", "abc");
}
```

```{style=planetext}
> : false
= : false
< : true
---
> : true
= : false
< : false
---
> : false
= : true
< : false
```

ただし、処理としては、2つの入力範囲のうち短い方を基準として要素の比較を行っていき、全て等しかった場合に長さの比較を行うため、長さが短い方が常に小さくなるとは限りません。

```cpp
#include <algorithm>

...

int main() {
  std::cout << std::boolalpha;

  str_cmp("bbc", "abcd");
}
```

```{style=planetext}
> : true
= : false
< : false
```

なお、このアルゴリズムに対応するRangeアルゴリズムおよび並列アルゴリズムは提供されません。

## ベクトル化を指定する実行ポリシー

C++17で並列アルゴリズム（*Parallel algorithm*）とその並列化を指示するための4つの実行ポリシーが追加されました。C++20ではそこに、ベクトル化を指定するための`std::execution::unseq`ポリシーが追加されます。ベクトル化とは一般的に、シングルスレッドにおいてSIMD命令によって実行することを指します。

|ポリシー|意味|バージョン|
|---|---|---|
|`seq`|並列化なし|C++17|
|`par`|マルチスレッド化|C++17|
|`par_unseq`|マルチスレッド and/or ベクトル化|C++17|
|`unseq`|ベクトル化|C++20|

`par_unseq`から`unseq`が分離された形です。C++17で入らなかったのは、シングルスレッドでベクトル化という指定の有用性（実装可能性や恩恵の程度など）を判断するのが間に合わなかったためだと思われます。

```cpp
#include <execution>

// どでかい配列を返すとする
auto enormous_vector() -> std::vector<int>;

int main() {
  auto vec = enormous_vector();

  // ベクトル化による並列化が期待できる
  std::sort(std::execution::unseq, vec.begin(), vec.end());
}
```

ただし、これは指定されたポリシーに基づく並列化を許可するものであって、必ずその並列化が行われるわけではありません。

## `midpoint`

`std::midpoint()`は2つの数値の中点を求めるものです。これは`<numeric>`に配置されています。

```{style=cppstddecl}
// <numeric>内
namespace std {
  template <class T>
  constexpr T midpoint(T a, T b) noexcept; 
}
```

`std::midpoint(a, b)`のように使用して、`a`と`b`の間の真ん中の値を返します。`T`には数値型かポインタ型（同じ配列内のものに限る）が使用可能です。

```cpp
#include <numeric>

int main() {
  {
    int a = 0;
    int b = 10;

    int m = std::midpoint(a, b);

    std::cout << m << '\n';
  }
  {
    int a = 0;
    int b = 9;

    int m = std::midpoint(a, b);

    std::cout << m << '\n';
  }
  {
    double a = 1.0;
    double b = 2.0;

    double m = std::midpoint(a, b);

    std::cout << m << '\n';
  }
}
```

```{style=planetext}
5
4
1.5
```

整数型で結果が実数値になる場合は、`a`の側に丸められます。

このような数値の中点を求めることはそう難しい計算ではなく、`(a + b)/2`が一番すぐ思いつきます。にも関わらずこの関数が用意されているのは、オーバーフローや数値誤差、負の数、非正規化数等々、実際のプログラミングにおいて考慮しなければならないことをライブラリ側でハンドリングするためです。

```cpp
#include <numeric>

// 単純midpoint実装
template<typename T>
T midpoint(T a, T b) {
  return (a + b) / T(2.0);
}

int main() {
  {
    // 10億と15億
    int a = 1000000000;
    int b = 1500000000;

    auto m1 = std::midpoint(a, b);
    auto m2 = midpoint(a, b); // UB

    std::cout << m1 << " : " << m2 << '\n';
  }
  {
    double a = DBL_MAX - 1;
    double b = DBL_MAX;

    auto m1 = std::midpoint(a, b);
    auto m2 = midpoint(a, b);

    std::cout << m1 << " : " << m2 << '\n';
  }
}
```

```{style=planetext}
1250000000 : -897483648
1.79769e+308 : inf
```

中点を求めるという単純な処理はしかし、数値型のハードウェア実装の都合などによってかなり奥が深い処理です。CppCon2019では、`std::midpoint()`の紹介と解説で1時間費やした発表が行われています（検索するとYoutubeで見つかるでしょう）。

## `lerp`

`std::lerp()`は、2つの浮動小数点数値間の値の線形補完を行う関数です。これは、`<cmath>`に配置されています。

```{style=cppstddecl}
namespace std {
  constexpr double lerp(double a, double b, double t) noexcept;
}
```

他にも`float`と`long double`のオーバーロードがありますが、これは浮動小数点数型に対するもののみが提供されます。


`std::lerp(a, b, t)`のように使用して、`a`と`b`の間を1としたときの時刻`t`の値を求めます。

```cpp
#include <cmath>

int main() {
  double a = 1.0;
  double b = 10.0;

  std::cout << std::fixed << std::setprecision(2);
  std::cout << std::lerp(a, b, 0.0) << '\n';
  std::cout << std::lerp(a, b, 0.1) << '\n';
  std::cout << std::lerp(a, b, 0.5) << '\n';
  std::cout << std::lerp(a, b, 0.75) << '\n';
  std::cout << std::lerp(a, b, 1.0) << '\n';

  // tは1を超えていてもok
  std::cout << std::lerp(a, b, 1.1) << '\n';
  std::cout << std::lerp(a, b, 1.5) << '\n';
}
```

```{style=planetext}
1.00
1.90
5.50
7.75
10.00
10.90
14.50
```

このような処理は線形補完としてよく知られており、これは`a + t * (b - a)`などとして簡単に求めることができそうに思えます。しかし実際には。`std::midpoint()`のように考慮すべきこと（数値誤差やオーバーフロー、非正規化数など）が多いため、それらをハンドリングするライブラリ機能として提供されます。

\clearpage

# 関数オブジェクト

ここで紹介するものは、`<functional>`に配置されています。

## `reference_wrapper`の不完全型対応

`std::reference_wrapper<T>`は`T`の参照オブジェクトを作成するためのクラスで、C++11で追加されました。導入当初から`T`には完全型（定義が見えている型）が要求されていましたが、C++20からはそれがなくなり、不完全型を使用可能になります。

```cpp
#include <functional>

// 定義はどこか別の場所にある
struct S;

// 定義はどこか別の場所にある
auto get_S() -> S&;

// Sのreference_wrapperを受け取る関数、定義はどこか
void f(std::reference_wrapper<S>);  // ok、C++17まではng

int main() {
  std::reference_wrapper<S> rws = get_S();  // ok、C++17まではng
  f(rws); // ok
}
```

このように、型がライブラリ内部など別の翻訳単位内で定義されている場合や、同じ翻訳単位でも定義が後から現れる場合などに`std::reference_wrapper<T>`を使用することができるようになります。ただし、`.get()`や暗黙変換による生参照の取得までは大丈夫ですが、`operator()`を呼び出そうとすると完全型が要求されます。

なお、どうやら一部の実装では最初からこれが可能な実装になっていたらしく、C++17でもエラーにならないかもしれません。

## `bind_front`

`std::bind_front()`は関数呼び出し可能なものとその引数を前方のものから順番に受けて、部分適用した関数オブジェクトを返す関数です。

```cpp
#include <functional>

// コールバック関数型
using callback = std::function<void(int, std::error_code&)>;

// ライブラリのコールバック関数登録処理
void register_callback(callback);

// コールバック処理を実装する型
struct S {

  // コールバック処理、非const関数
  void callback_impl(std::string_view, std::string_view, int, std::error_code&) {
    ...
  }
};

int main() {
  using namespace std::placeholders;

  // ラムダ式による部分適用
  register_callback([s = S{}](int n, std::error_code& ec) mutable {
    s.callback_impl("bound", "lambda", n, ec);
  });

  // std::bindによる部分適用
  register_callback(std::bind(&S::callback_impl, S{}, "bound", "bind()",  _1, _2));

  // std::bind_frontによる部分適用
  register_callback(std::bind_front(&S::callback_impl, S{}, "bound", "bind_front()"));

  ...
}
```

このように、ラムダ式（非`const`呼び出しのために`mutable`が必要、長い）や`std::bind`（プレースホルダが必要、値カテゴリが伝播されない）と比較するとシンプルに同じ部分適用をより正確に達成できます。渡す関数呼び出し可能なものに対する`const`性と値カテゴリの伝播を適切に自動化できるほか、この関数は引数列の前側の部分適用をするものであるため、残りの引数を気にする必要が無くなります。

`std::bind_front(f, boud_args...)`の戻り値は、左辺値で呼ばれると渡された`f`を左辺値として、右辺値で呼ばれると`f`を右辺値として呼び出し、自身の`const`をその呼び出しにまで伝播します。この性質によって、`const`性と値カテゴリの伝播が自動化され適切に解決されます。

```cpp
#include <functional>

struct F {
  void operator()(int n) & {
    std::cout << n << " : F&\n";
  }

  void operator()(int n) const & {
    std::cout << n << " : const F&\n";
  }

  void operator()(int n) && {
    std::cout << n << " : F&&\n";
  }

  void operator()(int n) const && {
    std::cout << n << " : const F&&\n";
  }
};

int main() {
  auto f = std::bind_front(F{});

  // 左辺値呼び出し
  f(1);

  // const左辺値呼び出し
  std::as_const(f)(2);

  // 右辺値呼び出し
  std::bind_front(F{})(3);

  // const右辺値呼び出し
  std::move(std::as_const(f))(4);
}
```

```{style=planetext}
1 : F&
2 : const F&
3 : F&&
4 : const F&&
```

呼び出しに当たっては`std::invoke()`が使用されるためこれによって呼び出せさえすればなんでも（ラムダクロージャ、メンバポインタ、`std::reference_wrapper`などなど）束縛させることができ、`std::bind_front(f, boud_args...)(call_args...)`の呼び出しは、`std::invoke(f, boud_args..., call_args...)`のような呼び出しと等価になります。渡された引数`f`と`boud_args...`はコピー/ムーブされて戻り値の関数オブジェクト内に保持されており、参照を渡したい場合は`std::ref()`や`std::cref()`を使う必要があります。

なおこれの逆版（関数引数の後ろ側を部分適用）はC++23から`std::bind_back()`として利用できる予定です。

\clearpage

# 文字列

主に`std::string`・`std::string_view`と`char8_t`関連です。

## `starts_with/ends_with`

`.starts_with()`は文字列の前方が指定された文字列と一致しているかを調べる関数です。これは`std::string`と`std::string_view`（他の文字型/アロケータ型特殊化も含めて）の両方で利用可能です。`.ends_with()`はその逆に、文字列の前方が指定された文字列と一致しているかを調べます。

```{style=cppstddecl}
namespace std {
  template <class CharT, class Traits = char_traits<CharT>>
  class basic_string_view {
    ...
  public:

    bool starts_with(basic_string_view) const noexcept;

    bool ends_with(basic_string_view) const noexcept;
  }
}
```

この他に、1文字（`CharT`型）を受け取るオーバーロードもあります。

`std::string`もしくは`std::string_view`のオブジェクト`str`に対して`str.starts_with("...")`のように使用して、引数に渡した文字列が先頭にマッチすれば`true`、そうでないなら`false`が返ります。

```cpp
#include <string>

int main() {
  std::cout << std::boolalpha;

  std::string str = "abcdefg";

  // 文字列によるマッチング
  std::cout << str.starts_with("abc") << '\n';
  std::cout << str.ends_with("abc") << '\n';

  // 文字によるマッチング
  std::cout << str.starts_with('g') << '\n';
  std::cout << str.ends_with('g') << '\n';
}
```

```{style=planetext}
true
false
false
true
```

連想コンテナの`.contains()`と同様に、やりたいことを簡潔に書くことができるとともに、わかりやすい名前がついていることによってやっていることが明確になります。

C++23では、この関数を任意の`range`と`range`に一般化した`std::ranges::starts_with`と`std::ranges::ends_with`アルゴリズムが追加されます。

## `reserve()`の縮小機能の廃止

`std::string`の`.reserve()`は指定された数の文字を追加のメモリ確保無しで保持できるように内部キャパシティ（`.capacity()`、現在確保しているメモリ量）を増大させる（すなわち、指定された分のメモリを確保する）関数です。この関数には数値を指定しますが、C++17まではその数値が現在のキャパシティよりも小さい場合に確保しているメモリ量を縮小することができていました（実装定義だったようですが）。

`std::vector`の`.reserve()`は縮小できなかったこともあり、この`std::string`独特の振る舞いは気づきづらい罠であり、ジェネリックなコード記述の妨げになっていました。

C++20ではそれが正され、`std::string`の`.reserve()`は指定された量が現在のキャパシティより小さい場合は何もしなくなりました。

\clearpage

```cpp
template<typename T>
  requires requires(T& t) {
    t.reserve(1ull);
  }
auto read(T& buf) {
  constexpr std::size_t min_len = ...;

  // メモリ量域を確保
  // tのキャパシティをmin_len以上にする
  // キャパシティがすでにmin_len以上なら何もしない（でほしい
  buf.reserve(min_len);

  // 何かデータの読み込み
  // 読み込み中に追加のメモリ確保を回避する
  read_to(std::back_inserter(buf));

  ...
}
```

ファイルやネットワークのI/Oなどでこのようなことを行いたいことは非常に良くあります。ある一定の量まではメモリ確保を起こさないようにするために`.reserve()`を使用しますが、この入力のバッファが使いまわされている場合などには既に十分な量が確保されている可能性があり、その場合は何もしないことが期待されます。C++20からはその期待に沿う振る舞いをするようになり、`std::string`オブジェクトの現在のキャパシティを気にせずに使用予定メモリ領域の予約を行えます。

## `string_view`のイテレータペアを受け取るコンストラクタ

`std::basic_string`はその文字型を要素とするイテレータのペアを受けて構築することができます。一方、`std::basic_string_view`はできません。`std::vector`を文字列バッファとして使用している場合など、文字の範囲から`std::string_view`を構築したいことはよくありますが、C++17まではその先頭アドレスと長さを与えることでしか行えませんでした。

C++20からは、`std::basic_string_view`にイテレータペアからの構築のためのコンストラクタが追加されます。

```cpp
#include <string_view>

int main() {
  std::string_view input = "split_view takes a view and a delimiter";

  // スペースをデリミタとしてsplitする
  for (auto inner_range : input | std::views::split(' ')) {
    // イテレータペアから構築
    std::string_view str{inner_range.begin(), inner_range.end()};

    // split_viewの内側rangeは文字列ではないので、string_viewに変換すると便利
    //std::cout << inner_range; // ng
    std::cout << str << '\n'; // ok
  }
}
```
```{style=planetext}
split_view
takes
a
view
and
a
delimiter
```

C++23ではさらに、`explicit`ではあるものの、`range`オブジェクトからの変換コンストラクタが追加されます。

## `char8_t`文字列型

言語機能としてUTF-8文字を表す`char8_t`が追加されたのに伴って、`char8_t`によって特殊化された文字列型（`std::string`, `std::string_view`）が追加されます。

```{style=cppstddecl}
namespace std {
  // <string>内
  using u8string  = basic_string<char8_t>;

  namespace pmr {
    using u8string  = basic_string<char8_t>;
  }

  // <string_view>内
  using u8string_view = basic_string_view<char8_t>;
}
```

文字型だけを特殊化したものなので、`char8_t`の文字/文字列を扱う以外は普通の（`char`の）文字列型と同様に扱えます。

```cpp
#include <string>
#include <string_view>

int main() {
  std::u8string u8str = u8"char8_t(UTF-8) string";
  std::u8string_view u8sv = u8str;
}
```

また、文字列型を生成するリテラル（`""s`、`""sv`）にも`char8_t`オーバーロードが追加されます。

```{style=cppstddecl}
namespace std {
  inline namespace literals {
    // <string>内
    inline namespace string_literals {
      constexpr u8string operator""s(const char8_t* str, size_t len);
    }

    // <string_view>内
    inline namespace string_view_literals {
      constexpr std::u8string_view operator""sv(const char8_t* str, std::size_t len) noexcept;
    }
  }
}
```

```cpp
#include <string>
#include <string_view>

using namespace std::literals;

int main() {
  std::u8string u8str = u8"string literal"s;
  std::u8string_view u8sv = u8"string view literal"sv;
}
```

従来の`u8""`に対するこれらのリテラルは`char`の文字列型を返していたため、これはC++11/17に対する破壊的変更となります。

## `char8_t`による破壊的変更

`char8_t`は言語機能に破壊的に導入されると同時にライブラリ機能に対しても破壊的変更をもたらしており、次の関数の戻り値型が変更となります。

|関数|C++17|C++20|
|---|---|---|
|`u8""s`|`std::string`|`std::u8string`|
|`u8""sv`|`std::string_view`|`std::u8string_view`|
|`path::u8string()`|`std::string`|`std::u8string`|
|`path::generic_u8string()`|`std::string`|`std::u8string`|

残念ながら、C++17までに書かれたコードでこの戻り値型に依存している部分はコンパイルエラーになってしまいます。言語バージョンを下げる以外に回避手段はないので、都度修正するしかありません（一応、GCCやMSVCには`char8_t`を無効化するオプションが用意されていたりはします）。

## `char8_t/char`文字エンコーディングの相互変換

C++20で`char8_t`が導入され型で区別されたUTF-8文字列を扱えるようになりましたが、文字エンコーディングの変換に関してはほぼサポートがありません。それどころか、`std::codecvt`はエンコーディング種別に関わらず非推奨とされたため（セキュリティ上の問題とのこと）、C++は文字エンコーディングの変換能力をほとんど失っています。

ほとんどと書いたのはCのライブラリにはその能力があり、C++でも利用可能だからです。C++20（C23）でも、`char8_t`（UTF-8）文字と`char`との相互変換のための関数が追加されています。

```cpp
namespace std {
  // charのエンコーディング -> UTF-8
  size_t mbrtoc8(char8_t* dst, const char* src, size_t n, mbstate_t* ps);

  // UTF-8 -> charのエンコーディング
  size_t c8rtomb(char* dst, char8_t src, mbstate_t* ps);
}
```

`char`のエンコーディングとは少なくともASCIIとの互換性があるエンコーディングで、例えばWindowsならANSI（日本語の場合Shift-JIS）、LinuxならUTF-8になります。

`std::mbrtoc8()`は`char`のエンコーディングからUTF-8へ、`std::c8rtomb()`はUTF-8から`char`のエンコーディングへと変換するものですが、これらの関数は文字列ではなく1文字を変換します。

どちらの関数も、第1引数は変換結果の格納先を指定し、第2引数に変換元の文字or文字列を指定します。

`std::mbrtoc8(dst, src, n, ps)`では、`dst`は変換結果出力先の`char`変数へのポインタ、`src`は変換元`char8_t`文字（文字列）へのポインタ、`n`は入力から1文字を読み取るのに必要な最大バイト数を指定します。`n`はUTF-8の場合常に4で良いはずです。

`mbstate_t`はどう使えばいいのかよくわからないので触れないでおきます。

```cpp
#include <cuchar>

using namespace std::literals;

int main() {
  auto cstr = "あ"sv;
  char8_t res[4]{};
  std::mbstate_t ms{}

  auto r = std::mbrtoc8(&res, cstr.data(), 4, &ms);

  assert(r < std::size_t(-3));

  std::cout << r << " : ";
  print(res);
}
```

`dst`には`char8_t`に変換した結果が書き込まれますが、ユニコードの任意の1文字はUTF-8で最大4バイトになり`char8_t`1つに収まらないため、少なくとも4要素の配列を渡すのが安全です。文字列ではなく文字の変換を行うため、出力はnull終端されません。

後述しますが、C++20ではユニコード文字型の出力機能も失われているため、この結果を標準の範囲内で視覚的に直接確認する方法がありません。そのためここでは、`print()`を文字表現を保ったまま`char8_t`文字を標準出力に出力する関数とします（標準出力はユニコード文字を表示可能とします）。すると、次のような結果が得られるはずです

```{style=planetext}
3 : あ
```

`std::mbrtoc8()`の戻り値`r`はエラー報告を兼ねており、次のような値と意味を持ちます

- `0` : 変換結果はnull文字（`u8'\0'`）
- `size_t(-3)` : 変換結果の文字が複数のコード単位から構成されている（サロゲートペアなど）
    - 後ろの方のコード単位に対応する値が`dst`に書き込まれる
    - 2022年現在はこの結果が返ることはないはず
- `size_t(-2)` : `n`バイトを読み込んでも、UTF-8の1文字に変換可能な文字が現れない
    - `dst`には何も書き込まれない
- `size_t(-1)` : `src`の値についてのエンコーディングエラー（不正な値が現れているなど）
    - `dst`には何も書き込まれない
- それ以外 : 変換は正常完了。値は入力文字列（`src`）の先頭から変換のために消費した（読み込んだ）バイト数

これは1文字を変換するものなので文字列の変換のためにはこれを連続適用する必要があり、正常時の戻り値もそのような使用法を想定しているようです。

```cpp
#include <cuchar>

// charのエンコーディング -> UTF-8 の文字列変換を行う
auto str_to_u8str(std::string_view src) -> std::u8string {
  const char* ptr = src.data();
  std::mbstate_t ms{};
  std::size_t bytes = 0;
  std::size_t consumed = 0;

  std::u8string u8str{};
  u8str.reserve(src.length());

  while (consumed < src.length()) {
    char8_t res[4]{};
    bytes = std::mbrtoc8(res, ptr, 4, &ms);

    assert(0 < bytes && bytes < std::size_t(-3));

    consumed += bytes;
    ptr += bytes;

    u8str.append(res);
  }

  return u8str;
}
```

　

`std::c8rtomb(dst, src, ps)`では、`dst`は変換結果出力先の`char8_t`変数へのポインタ、`src`は変換元`char`文字（文字列）へのポインタを指定します。

```cpp
#include <cuchar>

using namespace std::literals;

int main() {
  auto u8str = u8"a"sv;
  char res[4]{};
  std::mbstate_t ms{}

  auto r = std::c8rtomb(&res, u8str[0], &ms);
  assert(0 < r && r < std::size_t(-1));

  std::cout << r << " : " << std::string_view{res};
}
```
```{style=planetext}
3 : a
```

`std::c8rtomb()`の戻り値`r`はエラー報告を兼ねており、次のような値と意味を持ちます

- `0` : `src`は1バイトのUTF-8エンコーディングではない
- `size_t(-1)` : `char`のエンコーディングが`src`の文字を表現できない
- それ以外 : 変換は正常完了。値は出力文字列（`dst`）に書き込まれたバイト数。

この関数の入力`src`は`char8_t`へのポインタではなく`char8_t`の値を1つだけ取ります。UTF-8エンコーディングは最大4バイト（`char8_t`4つ分）になり、UTF-8で1バイトになる文字とはすなわちASCII範囲内の文字です。1バイトのUTF-8はASCIIと互換性があるため変換はキャストで十分であり、実質的にこの関数を使用する意味はなさそうです。戻り値を見て、`char8_t`1つがASCII範囲内の文字かどうかを判定するのには使えるかもしれませんが。

`std::c8rtomb()`がこのような残念仕様になってしまっているのは`<cuchar>`にある他の関数とインターフェースを一貫させた結果だと思われます。`<cuchar>`には他にも`char16_t`と`char32_t`に対して同様の役割の2対の関数が用意されており、確かに`char16_t`や`char32_t`から`char`の変換では`src`が1変数分でも十分に意味があります。`char16_t`と`char32_t`に対する関数はC++11（C11）から存在しているため、そちらのインターフェースに合わせざるを得なかったのだと思われます・・・

\clearpage

# `std::atomic`

## `char8_t`に対する特殊化

`std::atomic<T>`は`T`が整数型（及び`bool`型）のものに対して明示的特殊化を定義することで、整数型に特化したアトミック操作を提供しています。

C++20で追加された`char8_t`型についても同様に明示的特殊化が定義され、その名前付き型とロックフリープロパティも用意されます。

```{style=cppstddecl}
// <atomic>で定義

// char8_tのアトミックロックフリープロパティ
#define ATOMIC_CHAR8_T_LOCK_FREE ...

namespace std {

  // std::atomicのchar8_t特殊化
  template<> 
  struct atomic<char8_t> {
    ...
  };

  // 名前付きアトミック型
  using atomic_char8_t = atomic<char8_t>;
}
```

ロックフリープロパティの値は`0, 1, 2`のどれかの数字になり、`0`がロックフリーではないことを、`1`がロックフリーになるかもしれないことを、`2`が常にロックフリーであることを表します。ただし、このことは環境によって変化する可能性があるため、値は未規定（実装定義）です。

## 浮動小数点数型に対する特殊化

C++11以降、`std::atomic<T>`は`T`が浮動小数点数型の場合も利用可能であり、ほぼ期待通りの振る舞いを得られていました。ただそれは、整数型のように特殊化されたものではなくプライマリテンプレートによる汎用的なものだったため、数値演算で一般的な`+=`や`-=`のような操作（Read-Modify-Write : RMW）が使用できませんでした。

C++20では、それらRMW操作を提供するために`T`が浮動小数点数型である`std::atomic<T>`について明示的特殊化が定義されるようになります。ここで提供される固有の操作は次の4つのみです

- `.fetch_add()`
- `.fetch_sub()`
- `+=`
- `-=`

演算子は`std::atomic<T>`について`T`の値のみを受け取りますが、関数の方は、それに加えて`std::memory_order`の値を第2引数で受け取ることができ、必要に応じてより弱いメモリオーダーを指定することができます。

```cpp
#include <atomic>

std::atomic<double> a = 1.0;

void thread_op() {
  a += 1.0;
  a -= 1.0;
}

void thread_func() {
  // デフォルトはstd::memory_order_seq_cst
  a.fetch_add(1.0);
  a.fetch_sub(1.0);

  // メモリオーダーを明示的に指定
  a.fetch_add(1.0, std::memory_order_acq_rel);
  a.fetch_sub(1.0, std::memory_order_acq_rel);
}
```

これらの関数を使用することで、浮動小数点数に対するアトミックな数値演算をより直接的に書くことができるようになるとともに、一部のハードウェアではより効率的に実行される可能性があります。ただし、整数型に対する`std::atomic`特殊化が四則演算等他の数値演算をサポートしているのに対して、浮動小数点数型の特殊化ではこの4つのRMW操作しか提供されません。

## `shared_ptr/weak_ptr`に対する特殊化

`std::shared_ptr`には元々、そのオブジェクト（ポインタ）そのものにアトミックにアクセスするための非メンバ関数群が用意されていました。しかし、アトミックアクセスしたい`std::shared_ptr`オブジェクトは型としてそうではないものと区別がつかず、アトミックアクセス関数を通して使用しないとアトミックアクセスは保証されません。また、`std::shared_ptr`オブジェクトを間接参照（デリファレンス）しようとするときは、別の`std::shared_ptr`オブジェクトにアトミックにロードしてからそのローカル化した`std::shared_ptr`オブジェクトを間接参照する、という手間を踏まなければポインタにアトミックにアクセスできません。

```cpp
#include <memory>

std::shared_ptr<int> atomic_ptr{};

void thread_f() {
  // アトミックにshared_ptrを更新
  std::atomic_store(&atomic_ptr, std::make_shared<int>(20));

  // ポインタへのアクセスがアトミックになっていない
  auto n = *atomic_ptr;

  // こうすると、ポインタへのアクセスがアトミックになる
  auto ptr = std::atomic_load(&atomic_ptr);
  auto m = *ptr;
  // 一行で書いても良い
  // auto m = *std::atomic_load(&atomic_ptr);
}

void f() {
  // アトミック操作関数を経由しなければ非アトミックアクセス
  atomic_ptr = std::make_shared<int>(20);
  auto n = *atomic_ptr;
}
```

このようなAPIは間違えやすく、意図通りに使用することが難しかったため、C++20にて`std::shared_ptr`に対する`std::atomic`の部分的特殊化を追加することによって置き換えられました。`std::weak_ptr`も所有権を必要としないだけで同様に扱うことができるため（こちらにはアトミックアクセスAPIすらなかった）、同時に`std::weak_ptr`に対する`std::atomic`の部分的特殊化も追加されます。

```{style=cppstddecl}
// <memory>で定義
namespace std {

  template <class T>
  struct atomic<shared_ptr<T>>;

  template <class T>
  struct atomic<weak_ptr<T>>;
}
```

これは、`std::atomic`のポインタ型に対する特殊化のようにポインタそのもののアクセスをアトミックにするものです。例えば、ハザードポインタやあるいはもっと単純なマルチスレッドにおけるデータの共有のために使用することができます。

```cpp
#include <memory>

std::atomic<std::shared_ptr<int>> atomic_ptr{};
std::atomic<std::weak_ptr<int>> atomic_wptr{};

void thread_shread() {
  // アトミックにshared_ptrを更新
  atomic_ptr = std::make_shared<int>(20);
  // あるいはstore()する
  atomic_ptr.store(std::make_shared<int>(20));

  // 間接参照演算子はないので間違えようがない
  auto n1 = *atomic_ptr; // ng

  // 間接参照しようとすると必然的にアトミックアクセスになる
  auto ptr = atomic_ptr.load();
  auto m = *ptr;
  // 一行で書いても良い
  m = *atomic_ptr.load();
}

void thread_weak() {
  // weak_ptrのアトミック更新
  atomic_wptr = atomic_ptr.load();

  // weak_ptrの間接参照
  // load()でweak_ptrをアトミックに取得し
  // lock()でshared_ptrを取得
  if (auto ptr = atomic_wptr.load().lock(); ptr) {
    auto m = *ptr;
  }
}
```

先ほどの非メンバ関数のAPIと比べると、ポインタへのアトミックアクセスがより明示的になっていることが分かります。特に、間接参照したい場合に間違った手順がコンパイルエラーになるようになります。

## アトミック型のデフォルトコンストラクタの修正

`std::atomic`のデフォルトコンストラクタはC++17まで、どのように初期化されるかは未規定とされていました。

```cpp
#include <atomic>

int main() {
  // どちらも、状態は未規定
  std::atomic<int> a;
  std::atomic<int> b{};

  // どちらも、未定義動作
  a += 1;
  ++b;

  // 初期化にはatomic_init()を使用
  std::atomic<int> c;
  std::atomic_init(&c, 0);
  // c.load() == 0

  c += 1;
  // c.load() == 1
}
```

これはCの`_Atomic`との互換性を意識した仕様でしたが、リスト初期化の結果すら未規定というのは多くのC++プログラマの期待に反しており、バグの元でした。

`std::atomic`の特殊化はC++のみでしか使用されない（使用できない）ため、初期化が初期値を提供しない理由はないとして、C++20からはアトミック型のデフォルトコンストラクタが値初期化を行うようになります。そして同時に、`constexpr`と条件付き`noexcept`指定が追加されます。

この対象のアトミック型とは、`std::atomic`のプライマリテンプレートと、C++20で追加されたものも含めた全ての標準定義の特殊化、および`std::atomic_flag`です。

```cpp
#include <atomic>

std::atomic<int> g{}; // 0で定数初期化される

int main() {
  // どちらも0で初期化される
  std::atomic<int> a;
  std::atomic<int> b{};

  // どちらも1になる
  a += 1;
  ++b;
}
```

値初期化とは、組み込み型に対しては0（に相当する値）で初期化し、クラス型に対してはデフォルトコンストラクタを呼び出して初期化する、ようなことです。

C++20以降だけを相手にするならこの挙動を仮定しても安全ですが、C++17以前とコードを共有する場合やCと相互運用する場合などは初期化結果が未規定となる可能性が出てくるため、従来の初期化方法を使用するかどこかで初期値を代入しておく必要があります。

## `atomic_flag::test()`

`std::atomic_flag`は操作がアトミックなフラグ型であり、`std::atomic<bool>`と似たものです。`std::atomic<bool>`、そして全ての`std::atomic`特殊化と異なるところは、その操作が常にロックフリーであることが保証されている点です。その保証のために、操作はフラグ値の交換（`.test_and_set()`）とクリア（`.clear()`）のみと最低限なものしか提供されておらず、特にフラグ値を変更せずに読み取るということができませんでした。

```cpp
#include <atomic>

// フラグをクリア状態で初期化
std::atomic_flag lock{ATOMIC_FLAG_INIT};

// 簡単なスピンロック
void thread_f() {
  // ロックを取得
  while (lock.test_and_set()) {
    // test_and_set()は以前の値を返すので、falseが返るまで待機
  }

  // ロックが必要な処理
  ...

  // ロックの解放
  bool before = lock.clear(); // この関数も以前の値を返す
}

// フラグ状態を覗きたい
void g() {
  // フラグ状態を変更せずに読み取る方法がない
  bool lock_state = lock.???;
}
```

これは、一部のプラットフォーム（CPU）ではロックフリーなアトミック操作として`exchang`相当の命令しかサポートしていないことから、現在の２つのメンバ関数以外をロックフリーで実装できないためです。しかし、これによって`std::atomic_flag`はとても使いづらく、本来適しているはずの場所でも`std::atomic<bool>`がよく使用されるなど、利用されない原因になっていました。

そのため、C++20からは変更せずに値を読み取る為の関数`.test()`が追加されます。前述の、アトミック操作として`exchang`相当の命令しかサポートしていないプラットフォームは古いものであり、もはやC++ををサポートしていなかったため可能になったようです。 

また前述のように、C++20からデフォルトコンストラクタの効果はフラグをクリア状態にするようになります。これによって、`ATOMIC_FLAG_INIT`マクロを使用する必要がなくなります。

```cpp
#include <atomic>

// フラグをクリア状態で初期化
std::atomic_flag lock{};  // デフォルト構築はクリア状態

// 簡単なスピンロック
void thread_f() {
  // ロックを取得
  while (lock.test_and_set()) {
    // test_and_set()は以前の値を返すので、falseが返るまで待機
  }

  // ロックが必要な処理
  ...

  // ロックの解放
  lock.clear();
}

// フラグ状態を覗きたい
void g() {
  // フラグ状態を変更せずに取得
  bool lock_state = lock.test();

  std::cout << lock_state << '\n';

  // フリー関数版
  bool b = std::atomic_flag_test(&lock);
}
```

`.test()`は引数としてメモリオーダーを指定でき、デフォルトでは`std::memory_order::seq_cst`が指定されています。フリー関数版でメモリオーダーを指定するには`std::atomic_flag_test_explicit()`を使用します。

## `atomic_ref`

`std::atomic_ref<T>`は、`std::atomic`ではない型`T`のオブジェクトに対して`std::atomic`と同等のアトミック操作を提供するものです。

```cpp
#include <atomic>

int g = 0;

int main() {
  // 非アトミックアクセス
  g = 1;

  // アトミックアクセス
  std::atomic_ref{g}.store(2);
}
```

あるオブジェクトに対するほとんどの操作にアトミック性が必要ない場合に、一部のアトミックアクセスのために`std::atomic`オブジェクトにしてしまうと、アトミック性が不要なアクセス時にもアトミック命令や同期が入ってしまうことでパフォーマンスが低下してしまう可能性があります。かといって、非アトミックオブジェクトに対して局所的にアトミックアクセスを行おうとすると他の同期プリミティブ（`std::mutex`など）による同期が必要となり、手間がかかるばかりか`std::atomic`に比べて非効率にもなり得ます。

`std::atomic_ref`は非アトミックオブジェクトを参照して保持することで、アトミックアクセスが必要な箇所で適宜非アトミックオブジェクトに対するアトミックアクセスを行うことができます。例えば、本来`std::atomic`でラップすることのできないコンテナ型の要素アクセスをアトミックに行うことができます。

```cpp
#include <atomic>

// 複数スレッドからアクセスされる
// ただし、要素数は変更されないものとする
std::vector<int> vec(10);

// 範囲をアトミックな範囲に変換するRangeアダプタ
auto as_atomic = std::views::transform([](auto& e) { 
                                         return std::atomic_ref{e};
                                       });

// あるタイミングでvecの内容を確認
void observe() {
  for (auto an : vec | as_atomic) {
    // 要素毎にアトミックに読み込み
    std::cout << an.load() << '\n';
  }
}

// vecをゼロクリア
void clear() {
  // 要素毎にアトミックに0を書き込み
  std::ranges::fill(vec | as_atomic, 0);
}
```

（些細な注意点ですが、このようなことをする場合は`transform`に渡しているラムダ式の引数型が参照型になるように注意しましょう）

`std::atomic_ref<T>`はその事前定義特殊化やメンバ関数など、ほとんど`std::atomic<T>`と同様に扱うことができます。ただし、`std::atomic_ref`はコンストラクタで指定されたオブジェクトを参照で保持し所有権を持たないため、参照先のオブジェクトの寿命に注意を払う必要があります。例えば上の例で、`transform`引数のラムダ式の引数型の`&`を１文字忘れるとそれに出会うことができます。

## アトミックオブジェクトを介した待機・通知機構

`std::atomic`をはじめとするアトミック型はスピンロック等の待ち合わせ処理に簡単に使用することができます。しかし、その待機の方法はビジーループに限られており、これは無駄にCPU時間を消費するなど効率的な待機方法ではありませんでした。

```cpp
#include <atomic>

// 複数スレッド立ち上げ時に、処理の開始を待ち合わせる例
int main() {
  using namespace std::chrono_literals;
  
  std::atomic<bool> start{false};

  // スレッドの立ち上げ
  std::jthread th_array[5];
  for (auto& th : th_array) {
    th = std::jthread{[&start]{
      // スレッドの開始通知を待機
      while(start.load() == false) {
        // ビジーループをスリープさせて待機
        std::this_thread::sleep_for(100ms);
      }

      // メイン処理
      ...
    }};
  }

  ...

  // スレッド処理を遅延して開始
  start.store(true);
}
```

このような用途には`std::atomic_flag`を使った方がより正確ですが、どちらを用いても結局ビジーループを回避できません。ループ中にスリープを挿入するとビジーループの影響を低減できますが、スリープの時間設定によってはマルチスレッドパフォーマンスの低下や同期バグの原因になる可能性があります。このことは`std::atomic`を用いてよりハイレベルの同期プリミティブ（セマフォなど）を作成する際に、非効率性が静かに埋め込まれる事にもつながります。

この問題の解決のために、C++20からはアトミックオブジェクトを介した効率的な待機・通知機構が整備されます。そのために、`std::atomic`のメンバ関数として待機を行う`.wait()`、通知を行う`.notify_all()`、`.notify_one()`が追加されます。

```cpp
#include <atomic>

// 複数スレッド立ち上げ時に、処理の開始を待ち合わせる例
int main() {
  using namespace std::chrono_literals;
  
  std::atomic<bool> start{false};

  // スレッドの立ち上げ
  std::jthread th_array[5];
  for (auto& th : th_array) {
    th = std::jthread{[&start]{
      // スレッドの開始通知を待機
      start.wait(false);

      // メイン処理
      ...
    }};
  }

  ...

  // スレッド処理を遅延して開始
  // 値を代入してから
  start.store(true);
  // 通知
  start.notify_all();
}
```

待機関数`.wait(old)`は、待機時点のアトミックオブジェクトの現在の値（`old`）を指定して呼び出し、アトミックオブジェクトの値が`old`と異なっているならば通知関数が呼ばれるまでそのスレッドの実行をブロックします。このブロックはビジーループによらずOSの機能を用いるなどして効率的なスレッド実行停止が行われることを期待できます。

通知関数`.notify_all()`、`.notify_one()`は、引数なしで呼び出して、そのアトミックオブジェクトで`.wait()`しているスレッドのブロッキングを解除（再開）します。その際、`.notify_all()`は待機している全てのスレッドを再開し、`.notify_one()`は待機しているスレッドのうちの1つだけを再開します。

これらは`std::atomic`に特化した`std::condition_variable`と見ることもできます。

これらの待機・通知機構は次のもので構成されています

- メンバ関数 : `.wait()/.notify_one()/.notify_all()`
    - `std::atomic`
    - `std::atomic_flag`
    - `std::atomic_ref`
- `std::atomic`のためのフリー関数
    - `std::atomic_wait()`
    - `std::atomic_wait_explicit()`
    - `std::atomic_notify_one()`
    - `std::atomic_notify_all()`
- `std::atomic_flag`のためのフリー関数
    - `std::atomic_flag_wait()`
    - `std::atomic_flag_wait_explicit()`
    - `std::atomic_flag_notify_one()`
    - `std::atomic_flag_notify_all()`

フリー関数でプリフィックスに`_explicit`がつくものは、メモリオーダーを指定することができるものです。

この機構を用いると、C++20で追加されたよりハイレベルな同期プリミティブ（`std::latch`、`std::barrier`、`std::counting_semaphore`）を`std::atomic`によって簡易かつ効率的に実装することができます。実際に、いくつかの標準ライブラリ実装ではこれらを`std::atomic`を用いて実装しています。

```cpp
#include <atomic>

// 簡単な、std::latch実装
class my_latch {
  // 内部カウンタ
  std::atomic<std::ptrdiff_t> m_counter;

public:

  my_latch(std::ptrdiff_t count) : m_counter(count) {}

  void count_down(std::ptrdiff_t update = 1) {
    // 事前条件1
    assert(0 < update);

    // update分減算後の値を取得
    // アトミックアクセスを最小化するための措置
    const auto current = m_counter.fetch_sub(update) - update;

    // 事前条件2
    assert(0 <= current);

    if (current == 0) {
      m_counter.notify_all();
    }
  }

  void wait() const {
    // 内部カウンタの値が更新されても0になってはいないかもしれない
    // 0になるまで待機するためにチェックをループする
    for(;;) {
      const auto current = m_counter.load();

      if (current == 0) return;

      // オブジェクト不変条件
      assert(0 < current);

      m_counter.wait(current);
    }
  }
}
```

この実装は、MSVC STLの`std::latch`実装を参考にしています  
（https://github.com/microsoft/STL/blob/main/stl/inc/latch ）

## 整数型のロックフリーエイリアス

整数型の`std::atomic`特殊化は多くの環境で`is_always_lock_free`プロパティ（静的メンバ定数）が`true`となり、それが期待されますが、CPUによってはそれは期待できないこともあり、整数型の`std::atomic`特殊化が常にロックフリーに振る舞うかは`is_always_lock_free`プロパティを調べる必要がありました。

利便性のため、整数型の`std::atomic`特殊化であって`is_always_lock_free`プロパティが`true`となることが保証される型、のエイリアスが提供されます。

\clearpage

```{style=cppstddecl}
// ロックフリーな整数型のatomic特殊化の宣言例
namespace std {
  using atomic_signed_lock_free   = std::atomic<...>;
  using atomic_unsigned_lock_free = std::atomic<...>;
}
```

これらのエイリアスの`std::atomic<T>`の`T`は少なくとも整数型ですが、型は実装定義です。

また、追加のプロパティとして、これらの型はアトミックな待機と通知操作（`wait()/notify_one()/notify_all()`）が最も効率的に行える型であることが指定されています。

環境によって該当する特殊化が存在しない場合はこのエイリアスは定義されません。その場合はコンパイルエラーによって気づくことができるはずです。

## Compare-and-exchangeとパディング

`std::atomic<T>`の比較・交換（*Compare-and-exchange*）操作において`T`にパディングビットを含む構造体を使用すると、パディングビットが比較に参加する可能性があることによって、比較結果が意図通りにならない可能性がありました。

```cpp
#include <atomic>

struct padded {
  char clank = 0x42;/*
  ここにパディングが入る環境だとする
  */unsigned biff = 0xC0DEFEFE;
};

// padは他の場所で変更されないものとする
std::atomic<padded> pad = ATOMIC_VAR_INIT({});  // C++17までの初期化方法

bool zap() {
  // 値はpadと等しいが、パディングビットがどうなっているかは保証がない
  padded expected{};
  // 値もpadと異なる
  padded desired { 0, 0 };

  // 常にtrueが返ることが期待されるが、falseが返る可能性がある
  // 同様に、padがdesiredになる可能性がある
  return pad.compare_exchange_strong(expected, desired);
}
```

`.compare_exchange_strong(expected, desired)`は、現在のアトミックオブジェクトの値（`this`の値）と`expected`の値を比較し、`true`（等値）ならば現在の値を`expected`で置き換え、`false`（等しくない）ならば現在の値を`desired`で置き換えます。そして、戻り値として比較結果を返します。

この時の比較はバイト毎の比較であり、値の表現に参加していないバイト（パディングビット）が比較に参加する可能性があります。それによってプログラマが想定する値の比較（`operator==`による比較）による結果とは異なる可能性があります。

これは、C++においてオブジェクトの初期化やコピーの際のパディングビットの状態について何も保証がないことと、一般の構造体に対する`std::atomic<T>`の操作がアトミックであるために、オブジェクトの全てのバイトをアトミックにアクセスするような実装を取るためです（このため、`T`は*Trivially copyable*であることが求められます）。

この挙動は直感的ではなくバグの元にもなり得るため、C++20からは比較・交換操作における比較時にはその値の比較を行い、パディングビットは比較に参加しない（無視される）ことが規定されます。

```cpp
#include <atomic>

struct padded {
  char clank = 0x42;/*
  ここにパディングが入る環境だとする
  */unsigned biff = 0xC0DEFEFE;
};

// padは他の場所で変更されないものとする
std::atomic<padded> pad{};

bool zap() {
  // 値はpadと等しいが、パディングビットがどうなっているかは保証がない
  padded expected{};
  // 値もpadと異なる
  padded desired { 0, 0 };

  // C++20からは常にtrueが返ることが保証される
  return pad.compare_exchange_strong(expected, desired);
}
```

比較・交換操作とは、`std::atomic`のメンバ関数である`compare_exchange_strong()`と`compare_exchange_weak()`および、その非メンバ関数版と`explicit`付きのものです。

## `std::memory_order`の定義の変更

C++20から、`std::memory_order`列挙型とその値の定義が変更されます。

```{style=cppstddecl}
namespace std {
  // C++17までの定義
  typedef enum memory_order {
    memory_order_relaxed, memory_order_consume, memory_order_acquire,
    memory_order_release, memory_order_acq_rel, memory_order_seq_cst
  } memory_order;

  // C++20からの定義
  enum class memory_order : ... {
    relaxed, consume, acquire, release, acq_rel, seq_cst
  };

  inline constexpr memory_order memory_order_relaxed = memory_order::relaxed;
  inline constexpr memory_order memory_order_consume = memory_order::consume;
  inline constexpr memory_order memory_order_acquire = memory_order::acquire;
  inline constexpr memory_order memory_order_release = memory_order::release;
  inline constexpr memory_order memory_order_acq_rel = memory_order::acq_rel;
  inline constexpr memory_order memory_order_seq_cst = memory_order::seq_cst;
}
```

スコープ付き列挙型になったことで整数型との暗黙変換が禁止され、意味のない演算（`|`や`~`など）が行えなくなります。

C++17までの定義はCとの互換性を意識したものでしたが、Cが`_Atomic`を採用したことや`std`名前空間で定義されていることからあまり意味がなかったため修正される事になりました。

`std::memory_order`列挙値の名前を変更しているのは名前中の`memory_order`の重複を避けるためで、`std`名前空間に配置された`memory_order_~`な変数はC++17以前のコードの互換性のためのものです。列挙子の型を変更するとABIを破損する事になるため、新しい`std::memory_order`の基底型は未規定（実装定義）とされています。

結局、この変更はAPIもABIも破損していないため、利用する分にはあまり気にする必要はありません。

\clearpage

# `iostream`

## 配列への入力の修正

ストリーム入力（`>>`）では、生配列に入力することができます。

```cpp
#include <iostream>

int main() {
  char buffer[20]{};

  // 19文字を超えた入力があるとオーバーフローする
  std::cin >> buffer; // ok、ただし危険
}
```

ただし、これはバッファオーバーフローの危険性がありました。なぜなら、ここで使用されている`>>`は次のように宣言されているポインタを受け取るもので、配列長を考慮していないからです。

```{style=cppstddecl}
// C++17まで、配列等バッファへの入力演算子
namespace std {
  template<class CharT, class Traits>
  basic_istream<CharT, Traits>& operator>>(basic_istream<CharT, Traits>& is, CharT* s);

  template<class Traits>
  basic_istream<char, Traits>& operator>>(basic_istream<CharT, Traits>& is, unsigned char* s);

  template<class Traits>
  basic_istream<char, Traits>& operator>>(basic_istream<CharT, Traits>& is, signed char* s);
}
```

要するに、この演算子オーバーロードはCの`gets()`と同じ問題を抱えています。その使用は危険であり`gets()`と同様に削除すべきですが、上記のようにあらかじめ長さが分かっている配列バッファへの入力などは有用なユースケースであり、よく利用されていました。

そこで、上記のような有用なユースケースを保護しつつ危険な使用を削除するために、これらポインタを受けるオーバーロードを配列を受けるように変更します。

```{style=cppstddecl}
// C++20から、生配列への入力演算子
namespace std {
  template<class CharT, class Traits, std::size_t N>
  basic_istream<CharT, Traits>& operator>>(basic_istream<CharT, Traits>& is, CharT (&s)[N]);

  template<class Traits, std::size_t N>
  basic_istream<char, Traits>& operator>>(basic_istream<CharT, Traits>& is, unsigned char (&s)[N]);

  template<class Traits, std::size_t N>
  basic_istream<char, Traits>& operator>>(basic_istream<CharT, Traits>& is, signed char (&s)[N]);
}
```

この変更は以前のポインタを受けるオーバーロードを書き換える形で行われています。そのため、配列ではなくポインタを渡すコードはコンパイルエラーとなるようになります。

```cpp
#include <iostream>

int main() {
  char buffer[20]{};

  std::cin >> buffer; // ok、最大19文字まで読み込む

  char* ptr = buffer;
  std::cin >> ptr;  // ng
}
```

この変更によって、C++17以前からのコードのうち有用な使用はより安全になり、危険な使用はコンパイルエラーとなるようになります。

## ユニコード文字型の出力禁止

C++17までは、`u8`文字/文字列リテラルをストリーム出力（`<<`）するとその文字が出力されていました（実行時エンコーディングがUTF-8ではない場合は文字化けします）。C++20で`char8_t`が導入されたことでこの振る舞いは変化し、`u8`文字リテラルに対してはその数値が、`u8`文字列リテラルに対してはそのポインタ値が出力されるようになってしまっていました。

```cpp
#include <iostream>

int main() {
  std::cout << u8'a' << '\n';
  std::cout << u8"あ" << '\n';
}
```

C++17までの出力例（実行時エンコーディングがUTF-8の場合）

```{style=planetext}
a
あ
```

`char8_t`導入直後の出力例

```{style=planetext}
97
0x402004
```

この出力におけるポインタの値は実行によって異なります。

この静かで意図しない変更の影響を低減するために、出力ストリーム（`std::basic_ostream`）のストリーム出力演算子`<<`に、`char8_t`文字/文字列をとるオーバーロードが`delete`として追加されます。

```cpp
#include <iostream>

int main() {
  // どちらもC++20からコンパイルエラー
  std::cout << u8'a' << '\n';
  std::cout << u8"あ" << '\n';
}
```

`char16_t`（`u`リテラル）`char32_t`（`U`リテラル）に関しても以前から同様の問題があったため、無意味な使用をコンパイルエラーにするために同様に`delete`定義が追加されます。

```cpp
#include <iostream>

int main() {
  // 全てC++20からコンパイルエラー
  std::cout << u'a' << '\n';
  std::cout << u"あ" << '\n';
  std::cout << U'a' << '\n';
  std::cout << U"あ" << '\n';
}
```

C++17までの出力例

```{style=planetext}
97
0x402008
97
0x40200c
```

C++17までとC++20からの各文字（列）型と標準出力ストリームの振る舞い

|リテラル|C++17 `cout`|C++17 `wcout`|C++20 `cout`|C++20 `wcout`|
|---|:-:|:-:|:-:|:-:|
|`'c'`|◎|○|◎|○|
|`"str"`|◎|○|◎|○|
|`L'c'`|△|◎|×|◎|
|`L"str"`|△|◎|×|◎|
|`u8'c'`|○|△|×|×|
|`u8"str"`|○|△|×|×|
|`u'c'`|△|△|×|×|
|`u"str"`|△|△|×|×|
|`U'c'`|△|△|×|×|
|`U"str"`|△|△|×|×|

凡例

- ◎ : 適切な使用である
- ○ : 文字/文字列として出力はできるものの結果は他の要因の影響を受ける
- △ : 文字型ならばその数値が、文字列型ならばポインタ値が出力される
- × : コンパイルエラー

元々無意味だった利用がコンパイルエラーになるようになりますが、これによってユニコード文字型（`wchar_t`を除く）の文字/文字列を出力する方法はC++標準ライブラリには用意されなくなります。`std::basic_string`や`std::basic_string_view`の出力も内部でこれらのオーバーロードを使用することから、出力の可否は各文字型のリテラルと同様です。

この変更は、`std::basic_ostream<CharT, traits>`の非メンバ演算子オーバーロードとして追加されているため、これを継承する全ての出力ストリームが影響を受け、同様の結果になります。

## 文字列バッファ/ストリームの`.view()`

文字列ストリームクラス（`std::basic_stringstream`とその特殊化）は入出力ストリームであり、その入出力が内部の文字列バッファに対して行われるストリームです。このクラスはストリーム入出力の結果としての内部文字列を取り出すためのメンバ関数`.str()`を備えていますが、これは内部の文字列バッファの内容をコピーするもので単に参照だけしたい場合に非効率でした。

そのため、`std::basic_stringstream`の内部バッファ参照時のコピーを回避するために、それを参照する`std::string_view`を返すメンバ関数`.view()`が追加されます

```cpp
#include <sstream>

int main() {
  std::stringstream ss;

  ss << "C++";
  ss << 20;

  // C++17まではコピーせざるを得なかった
  std::string copy = ss.str();

  // C++20からは単に参照できる
  std::string_view view = ss.view();

  std::cout << copy << '\n';
  std::cout << view << '\n';
}
```

```{style=planetext}
C++20
C++20
```

また、`.str()`には右辺値参照修飾されたオーバーロードが追加され、`std::basic_stringstream`オブジェクトが右辺値の時に内部バッファの文字列をムーブして返します。

```cpp
#include <sstream>

int main() {
  std::stringstream ss;

  ss << "C++";
  ss << 20;

  // コピーされている
  std::string cpstr = ss.str();

  // ムーブされている
  std::string mvstr = std::move(ss).str();
}
```

同様の変更は`std::basic_stringstream`の内部バッファとして使用される文字列バッファクラス（`std::basic_stringbuf`とその特殊化）にも適用されます（が、これを直接使用することはほぼないでしょう）。

## 文字列バッファ/ストリームクラスのアロケータ対応

文字列ストリームクラスおよび文字列バッファクラスは内部バッファとして実質的に`std::string`を使用しており、どちらのクラスもそのアロケータ型を自身のクラステンプレートのテンプレート引数として受け取ります。

```{style=cppstddecl}
// 文字列ストリーム関連クラスの宣言例
namespace std {
  template< 
    class CharT, 
    class Traits = char_traits<CharT>, 
    class Allocator = allocator<CharT> 
  >
  class basic_stringbuf : public basic_streambuf<CharT, Traits>;

  template< 
    class CharT, 
    class Traits = char_traits<CharT>,
    class Allocator = allocator<CharT>
  >
  class basic_stringstream : public basic_iostream<CharT, Traits>;
}
```

しかし、このアロケータ型をカスタムした後でカスタムしたアロケータオブジェクトを渡すには、それを内包した`std::string`オブジェクトを渡すしか方法がありませんでした。文字列ストリームの場合は、必ずしも初期文字列を指定しないのでこの使用法は直感的ではありません。

そのため、`std::basic_stringstream`と`std::basic_stringbuf`のコンストラクタに直接アロケータを渡せるようにした上で、異なるアロケータ型を持つ`std::string`からの変換もサポートするようになります。

```cpp
#include <sstream>

// 自作アロケータ型とする
struct myalloc {...};

using my_sstream = std::basic_stringstream<char, std::char_traits<char>, myalloc>;

int main() {
  myalloc alloc{};
  std::basic_string str("custom alloc", alloc); // CTAD
  std::string str2("default alloc");

  my_sstream ss1{str};   // ok、C++11から
  my_sstream ss2{alloc}; // ok、C++20から
  my_sstream ss3{str2};  // ok、C++20から
}
```

型名`S`を`std::basic_stringstream`と`std::basic_stringbuf`のどちらかとして`S<CharT, Traits, Alloc>`に対して、追加されるコンストラクタは次のようになります

1. `explicit S(const Alloc&)`
    - `std::basic_stringbuf`のみ
2. `S(openmode, const Alloc&)`
    - `openmode`にはデフォルト引数がない
3. `explicit S(basic_string<CharT, Traits, Alloc>&&, openmode)`
4. `S(const basic_string<CharT, Traits, SAlloc>&, const Alloc&)`
5. `S(const basic_string<CharT, Traits, SAlloc>&, openmode, const Alloc&)`
6. `explicit S(const basic_string<CharT, Traits, SAlloc>&, openmode)`
7. `S(basic_stringbuf&&, const Alloc&)`
    - `std::basic_stringbuf`のみ

リスト中の`SAlloc`は`S<CharT, Traits, Alloc>`の`Alloc`と異なるアロケータ型を表します。また、2のコンストラクタを除いて`openmode`（`std::ios_base::openmode`）の引数にはデフォルト値として`std::ios_base::in | std::ios_base::out`が指定されています。

さらに、`.str()`もカスタムアロケータを考慮するようになることで、コピーする際に使用されるアロケータをカスタマイズ可能となります。

```cpp
#include <sstream>

// 自作アロケータ型とする
struct myalloc {...};

int main() {
  myalloc alloc{};
  std::stringstrem ss;

  ss << "C++";
  ss << 20;

  // アロケータを渡してそれを用いてコピー先メモリを確保
  auto restr = ss.str(alloc);
  // restr.get_allocator() == alloc

  std::basic_string str("replace", alloc);

  // 異なるアロケータを持つ文字列によるバッファ置換
  // ただし、アロケータを置き換えるわけではない
  ss.str(str);
}
```

また、`std::basic_stringbuf`には使用しているアロケータを取得するための`.get_allocator()`が追加されます。

```cpp
#include <sstream>

template<typename Alloc>
using sstream = std::basic_stringstream<char, std::char_traits<char>, Alloc>;

template<typename Alloc>
void f(sstream<Alloc>& strm) {
  // 内部のstd::stringbufからアロケータを取得
  auto alloc = strm.rdbuf()->get_allocator();

  // 取得したアロケータを使用したstd::string
  std::basic_string str("custom alloc", alloc);

  ...
}
```

\clearpage

# スマートポインタ

## `make_shared`の配列対応

`std::make_shared<T>()`は`std::shared_ptr<T>`を作成するヘルパ関数であり、その作成にあたって例外安全性や効率性を提供してくれる優れものです。ただ、`std::shared_ptr`の配列版サポートはあるのに、この関数はなぜか`T`が配列型である場合のサポートがありませんでした。そのため、`T`が配列型の場合のためのオーバーロードが追加されます。

```{style=cppstddecl}
// <memory>で定義
namespace std {
  
  // 要素数不明の配列型（U[]）のためのもの
  // 各要素は値初期化される
  template<class T>
  shared_ptr<T> make_shared(size_t N);

  // 要素数既知の配列型（U[N]）のためのもの
  // 各要素は値初期化される
  template<class T>
  shared_ptr<T> make_shared();

  // 要素数不明の配列型（U[]）のためのもの
  // uを初期値として各要素を初期化する
  template<class T>
  shared_ptr<T> make_shared(size_t N,
                            const remove_extent_t<T>& u);

  // 要素数既知の配列型（U[N]）のためのもの
  // uを初期値として各要素を初期化する
  template<class T>
  shared_ptr<T> make_shared(const remove_extent_t<T>& u);
}
```

これはまた、`std::allocate_shared()`にも同様に追加されます。この関数は、指定されたアロケータを用いてメモリを確保しオブジェクトを初期化しつつ適切に解放を行うカスタムデリータを仕込み、尚且つそれらを1回のメモリ確保で行ってくれるものです。

```{style=cppstddecl}
// <memory>で定義
namespace std {

  // 要素数不明の配列型（U[]）のためのもの
  // 各要素は値初期化される
  template<class T, class A>
  shared_ptr<T> allocate_shared(constA& a, size_t N);

  // 要素数既知の配列型（U[N]）のためのもの
  // 各要素は値初期化される
  template<class T, class A>
  shared_ptr<T> allocate_shared(constA& a);

  // 要素数不明の配列型（U[]）のためのもの
  // uを初期値として各要素を初期化する
  template<class T, class A>
  shared_ptr<T> allocate_shared(constA& a, size_t N,
                                const remove_extent_t<T>& u);

  // 要素数既知の配列型（U[N]）のためのもの
  // uを初期値として各要素を初期化する
  template<class T, class A>
  shared_ptr<T> allocate_shared(constA& a, const remove_extent_t<T>& u);
}
```

`std::make_shared()`を用いた例

```cpp
#include <memory>

using std::views::counted;
using std::ranges::all_of;

template<int N>
auto eq = [](int n) { return n == N; };

int main() {
  std::cout << std::boolalpha;
  
  // intの配列を10要素で初期化
  auto sp1 = std::make_shared<int[]>(10);
  auto sp2 = std::make_shared<int[10]>();

  // 要素は全て値初期化（intなら0）されている
  std::cout << all_of(counted(sp1.get(), 10), eq<0>) << '\n';
  std::cout << all_of(counted(sp2.get(), 10), eq<0>) << '\n';
  
  // intの配列を10要素、初期値20で初期化
  auto sp3 = std::make_shared<int[]>(10, 20);
  auto sp4 = std::make_shared<int[10]>(20);
  
  // 要素は全て指定した値で初期化されている
  std::cout << all_of(counted(sp3.get(), 10), eq<20>) << '\n';
  std::cout << all_of(counted(sp4.get(), 10), eq<20>) << '\n';
}
```
```{style=planetext}
true
true
true
true
```

`views::counted`はイテレータ1つと要素数から範囲を生成するRangeアダプタです。配列版スマートポインタはそのままでは`range`にならないため、これを用いると簡単に`range`化できます（結果型は`std::span`になります）。

`std::allocate_shared()`は第1引数にアロケータを渡す以外は同様です。

初期値を取らない関数の場合は各要素を値初期化（組み込み型はゼロ相当、クラス型はデフォルトコンストラクタ呼び出し）します。そのため、初期化後の各要素の値が不定にはなりません（クラス型でメンバを初期化しない場合は不定値を抱え込む場合があります）。

## スマートポインタをデフォルト構築する

前節の`std::make_shared()`や`std::allocate_shared()`の配列版では、初期値を指定しない場合に値初期化が行われていました。この挙動は非配列版や`std::make_unique()`でも同様であり、これはすなわち確保したメモリのゼロクリアが常に行われることを示しています。

確保するメモリ量が大きくなるとこのオーバーヘッドは無視できなくなり、パフォーマンス低下の可能性があります。そのため、これらのスマートポインタ生成ヘルパ関数に対してサフィックスに`for_overwrite`と付く、確保した領域に何もしない関数が追加されます。

```{style=cppstddecl}
// <memory>で定義
namespace std {

  // 非配列型のためのもの
  template<class T>
  constexpr unique_ptr<T> make_unique_for_overwrite();
  
  // 配列型のためのもの
  template<class T>
  constexpr unique_ptr<T> make_unique_for_overwrite(size_t n);
  
  // 要素数が既知の配列型では使用不可
  template<class T, class... Args>
  unspecified make_unique_for_overwrite(Args&&...) = delete;


  // 非配列型のためのもの
  template<class T>
  shared_ptr<T> make_shared_for_overwrite();
  
  // 非配列型のためのもの
  template<class T, class A>
  shared_ptr<T> allocate_shared_for_overwrite(const A& a);

  // 配列型のためのもの
  template<class T>
  shared_ptr<T> make_shared_for_overwrite(size_t N);
  
  // 配列型のためのもの
  template<class T, class A>
  shared_ptr<T> allocate_shared_for_overwrite(const A& a, size_t N);
}
```

これらの関数はその意図として初期化後の領域の状態が不定になります。従って、読み出しの前に適切に各要素を初期化する必要があります。

`make_unique`が要素数が既知の配列型（`T[N]`）に対して`delete`されているのは、`std::make_unique<T>()`が`std::unique_ptr<T>`を返すという一貫性を維持していることと、`delete`定義が無い場合に`std::make_unique<T[N]>()`を使用すると非配列オーバーロードが使用されて非配列版`std::unique_ptr<T[N]>`が返されてしまうことを防止するためです。

## 未初期化メモリに対するアルゴリズム

前節の`for_overwrite`系の関数で配列としてメモリを確保したとき、その領域の初期化は少し面倒な作業です。典型的な`for`ループではなく、アルゴリズム的にさっと書いて済ませられると便利です。それを行う関数としてC++11から`std::uninitialized_~`系の未初期化メモリに対する操作アルゴリズム関数が用意されています。

それらの現在のものはイテレータペアを取るものであり、`<ranges>`導入に伴って`range`を取ることができコンセプトで制約されたRange版が追加されます。これらの関数はRangeアルゴリズムと同様に`std::ranges`名前空間に配置されます。

|関数|効果|
|---|---|
|`uninitialized_default_construct(r)`|各要素をデフォルト構築|
|`uninitialized_default_construct_n(i, n)`|先頭`n`要素をデフォルト構築|
|`uninitialized_value_construct(r)`|各要素を値初期化|
|`uninitialized_value_construct_n(i, n)`|先頭`n`要素を値初期化|
|`uninitialized_copy(in_r, out_r)`|各要素を別の範囲からコピーして初期化|
|`uninitialized_copy_n(ii, n, oif, oil)`|先頭`n`要素を別の範囲からコピーして初期化|
|`uninitialized_move(in_r, out_r)`|各要素を別の範囲からムーブして初期化|
|`uninitialized_move_n(ii, n, oif, oil)`|先頭`n`要素を別の範囲からムーブして初期化|
|`uninitialized_fill(r, init)`|各要素を`init`で初期化|
|`uninitialized_fill_n(i, n, init)`|先頭`n`要素を`init`で初期化|
|`destroy(r)`|各要素を破棄|
|`destroy_n(i, n)`|先頭`n`要素を破棄|

表内引数の意味

- `r, out_r` : 初期化対象の範囲（`ranges`）
- `n` : 要素数
- `in_r` : 初期化するオブジェクトを提供する他の範囲
- `ii` : `in_r`の先頭イテレータ
- `oif, oil` : `out_r`のイテレータペア
- `init` : 初期値
- `i` : `out_r`の先頭イテレータ
- `p` : ポインタ
- `args...` : コンストラクタ引数

範囲（`r, out_r, in_r`）を受け取る関数は、その位置にイテレータペアを受け取るオーバーロードも用意されています。

`_default_construct`系は範囲をデフォルト初期化（トリビアルな型は何もせず、それ以外の型はデフォルトコンストラクタ呼び出し）し、`_value_construct`系は範囲を値初期化するものですので、これらはスマートポインタの生成ヘルパ関数の結果に対して使うことはないでしょう。前者の状態が欲しい場合は`for_overwrite`系の関数を、後者の状態が欲しい場合は`for_overwrite`ではない関数を使用すればよいためです。

```cpp
#include <memory>

using namespace std::ranges;
using namespace std::views;

template<int N>
auto eq = [](int n) { return n == N; };

int main() {
  // 配列を動的確保しスマートポインタに詰める
  // 領域は未初期化
  auto up = std::make_unique_for_overwrite<int[]>(10);
  auto sp = std::make_shared_for_overwrite<int[]>(10);

  // [0, 10)の数列で初期化
  uninitialized_copy(iota(0, 10), counted(up.get(), 10));

  // 全要素を20で埋める
  uninitialized_fill(counted(sp.get(), 10), 20);

  std::cout << std::boolalpha;
  std::cout << equal(counted(up.get(), 10), iota(0, 10)) << '\n';
  std::cout << all_of(counted(sp.get(), 10), eq<20>) << '\n';

  // 範囲の各要素のオブジェクトを破棄（デストラクタ呼び出し）
  destroy(counted(up.get(), 10));
  
  // upの先頭5要素をspの範囲からムーブして初期化
  uninitialized_move_n(sp.get(), 5, up.get(), up.get() + 10);
  
  std::cout << all_of(counted(up.get(), 5), eq<20>) << '\n';
}
```
```{style=planetext}
true
true
true
```

`uninitialized_copy`と`uninitialized_move`は最初の引数に初期値を提供する範囲を指定し、最後の引数で初期化対象の未初期化範囲を指定するのですが、初期化したい範囲を指定する引数の位置がほかと異なっているため注意が必要かもしれません。

なお、これらの関数はRangeアルゴリズムや`std::ranges`のイテレータ関連ユーティリティ（`ranges::advance()`等）と同様にADLを無効化する性質を持っています。

ここまでは流れでこれらのアルゴリズム関数を`for_overwrite`系のスマートポインタ生成関数で例示していましたが、スマートポインタ生成関数はいずれも確保した領域にオブジェクトを構築するまでが仕事であるため、厳密にはこれらアルゴリズムが対象とする未初期化メモリ領域ではありません（そのため、デストラクタがユーザー定義されているクラス型では先ほどのような上書きは安全ではありません）。本当の未初期化メモリとは、`::operator new()`（`new`式ではなく）や`malloc()`、システムやライブラリの提供するメモリ確保関数など、メモリの確保だけを行う関数によって取得されたメモリ領域です。そのような素のメモリ領域には想定するであろう型のオブジェクトが構築されていないため、ポインタを通したアクセスが未定義動作となります。

```cpp
#include <memory>

class C {
  int n = 10;
public:
  C() = default;

  int get() {
    return this->n;
  }
};

// Cはトリビアルではない
static_assert(not std::is_trivially_default_constructible_v<C>);

using namespace std::ranges;
using namespace std::views;

template<int N>
auto eq = [](C& c) { return c.get() == N; };


int main() {
  // 5要素分のCの領域を確保
  C* p1 = static_cast<C*>(::operator new( sizeof(C[5]) ));
  C* p2 = static_cast<C*>(   std::malloc( sizeof(C[5]) ));

  // UB!! p1,p2の指す領域にオブジェクトが構築されていない
  //std::cout << p1->get();
  //std::cout << p2->get();

  // 領域を値初期化
  uninitialized_value_construct(counted(p1, 5));
  // 領域をデフォルト初期化
  uninitialized_default_construct(counted(p2, 5));

  std::cout << std::boolalpha;
  std::cout << all_of(counted(p1, 5), eq<10>) << '\n';
  std::cout << all_of(counted(p2, 5), eq<10>) << '\n';
}
```
```{style=planetext}
true
true
```

この例では、値初期化でもデフォルト初期化でも`C`のデフォルトコンストラクタが呼ばれ、デフォルトメンバ初期化の`10`で`C::n`は初期化されています。クラス`C`のメンバ`n`のデフォルトメンバ初期化子（`= 10`）を消すと、デフォルト初期化の場合に値が不定になります。

ただし、*implicit-lifetime types*と呼ばれる型のオブジェクトはこのような場合でも暗黙的に構築されます（詳細は「C++20 言語機能」を参照）。組み込み型は少なくともその型のグループに含まれています。

\clearpage

# メモリ関連

## コンパイル時メモリ確保関連

C++20で解禁されたコンパイル時動的メモリ確保は、言語仕様の変更と一部のライブラリ機能の特別扱いによって達成されています。その詳細は「C++20 言語機能」を参照してください。ここではライブラリの変更のみを扱います。

まず、グローバルな置換可能な（非ユーザー定義の）`operator new`と`operator delete`が定数式で呼び出し可能になりますが、これは特別扱いされており特に`constexpr`指定されているわけではありません。

次に、これを標準コンテナなどで利用するために、アロケータクラスが`constexpr`対応します。次のクラスとそのメンバ関数が`constexpr`指定されるようになります

- `std::allocator`
    - コンストラクタ
    - デストラクタ
    - `.allocate()`
    - `.deallocate()`
    - `operator=`
    - `operator==`（非メンバ関数）
    - `operator!=`（非メンバ関数）
- `std::allocator_traits`
    - `.allocate()`
    - `.deallocate()`
    - `.max_size()`
    - `.construct()`
    - `.destroy()`
    - `.select_on_container_copy_construction()`

`std::allocator_traits`の`.construct()`と`.destroy()`の定数式での実行においては、前者は配置`new`後者は`~T()`（デストラクタ呼び出し）が必要で、これらの実行においてはポインタの再解釈が必要となります。定数式でそれは許可されないため、これらの操作を行うライブラリ関数を`constexpr`指定します。

- オブジェクト構築（配置`new`）を行う
    - `std::construct_at()`
    - `std::ranges::construct_at()`
- オブジェクト破棄（デストラクタ呼び出し）を行う
    - `std::destroy_at()`
    - `std::ranges::destroy_at()`
    - `std::destroy()`
    - `std::destroy_n()`
    - `std::ranges::destroy()`
    - `std::ranges::destroy_n()`

`ranges::construct_at()`と`ranges::destroy_at()`はADLを無効化するという性質を持つ点だけが`std`名前空間にあるものと異なり、範囲を受け取って一括処理するものではありません。`destroy`系関数については前節を参照してください。

これらのものはユーザーが直接使用することもできますが、その時でもコンパイル時動的メモリ確保のルールに従っている必要があります。最も、多くの場合は`std::string`と`std::vector`を介して使用することになるかと思われます。

```cpp
#include <memory>

// ok、C++20から
constexpr int f() {
  std::allocator<int> alloc{};

  // メモリ確保とオブジェクト構築
  int* p = alloc.allocate(1);
  p = std::construct_at(p);

  *p = 20;
  int n = *p;

  // オブジェクト破棄とメモリ解放
  std::destroy_at(p);
  alloc.deallocate();

  return n;
}
```

## `assume_aligned`

`std::assume_aligned()`は、メモリ領域のアライメントについての仮定をコンパイラに伝える関数です。この目的はより積極的な最適化にあります。`std::assume_aligned<N>(ptr)`のように使用して、`ptr`のメモリ領域が`N`アライメントでアラインされていることをコンパイラに伝達します。なお、`N`は２のべき乗の正の整数値である必要があります。

```{style=cppstddecl}
namespace std {
  template<size_t N, class T>
  [[nodiscard]]
  constexpr T* assume_aligned(T* ptr);
}
```

`std::assume_aligned()`の戻り値は引数の`ptr`がそのまま返されますが、`[[nodiscard]]`がついているように、戻り値は仮定されたアライメントを満たしている領域へのポインタとみなされる新しいポインタであり、この関数の恩恵を受けるにはこの戻り値のポインタを使用しなければなりません。

```cpp
#include <memory>

void add(float* v1, float* v2) {
  constexpr int N = 4;

  float* p1 = std::assume_aligned<16>(v1);
  float* p2 = std::assume_aligned<16>(v2);

  for (int i = 0; i < N; ++i) {
    // assume_alignedを通したポインタを使用する
    p1[i] += p2[i];
  }
}
```

このコードをx86-64環境のg++ 12.2 -std=c++20 -O3でコンパイルし、出力アセンブリを見てみます（Compiler Explorerを使用しました）。比較のために、`std::assume_aligned()`を使用しない場合も載せておきます

`std::assume_aligned()`を使用する場合

```asm
add(float*, float*):
        movaps  xmm0, XMMWORD PTR [rdi]
        addps   xmm0, XMMWORD PTR [rsi]
        movaps  XMMWORD PTR [rdi], xmm0
        ret
```

`movaps`はSSE命令の1つで、4つの`float`値をSSEレジスタ（`xmm0`）にコピーするものです。`addps`もSSE命令で、第1オペランド（`xmm0`）の4つの`float`値に、第2オペランドの4つの`float`値を足しこむものです。3行目の`movaps`は結果を第1引数（`v1`）の領域に書き戻しています。

`std::assume_aligned()`を使用しない場合（サンプルコード中の`p1, p2`の初期化を`v1, v2`をそのまま代入するように変更）

```asm
add(float*, float*):
        lea     rdx, [rsi+4]
        mov     rax, rdi
        sub     rax, rdx
        cmp     rax, 8
        jbe     .L2
        movups  xmm0, XMMWORD PTR [rsi]
        movups  xmm1, XMMWORD PTR [rdi]
        addps   xmm0, xmm1
        movups  XMMWORD PTR [rdi], xmm0
        ret
.L2:
        movss   xmm0, DWORD PTR [rdi]
        addss   xmm0, DWORD PTR [rsi]
        movss   DWORD PTR [rdi], xmm0
        movss   xmm0, DWORD PTR [rdi+4]
        addss   xmm0, DWORD PTR [rsi+4]
        movss   DWORD PTR [rdi+4], xmm0
        movss   xmm0, DWORD PTR [rdi+8]
        addss   xmm0, DWORD PTR [rsi+8]
        movss   DWORD PTR [rdi+8], xmm0
        movss   xmm0, DWORD PTR [rdi+12]
        addss   xmm0, DWORD PTR [rsi+12]
        movss   DWORD PTR [rdi+12], xmm0
        ret
```

まずやたら長いので少なくともコードが変わっていることが分かります。ここで行われているのは、まず入力配列のアライメントをチェックして8バイトアライメントよりも大きければ`movups`命令（アライメント要求が無い`movaps`）によって先程とほぼ同様にSSE命令によって処理されます。8バイトアライメント以下である場合、ループ（展開されてはいますが）によって1要素づつ足しています。

`std::assume_aligned()`を使用しない場合でもコンパイラは相当頑張った最適化をしていますが、`std::assume_aligned()`によってアライメント仮定が伝わることでアライメントの考慮をしなくて済むようになり、最適化が促進されていることが分かります。

このように、適切に使用すると最適化を促進できる可能性がありますが、`std::assume_aligned()`を通したポインタが本当に仮定したアライメント通りにアラインされていることはプログラマが保証しなければなりません。そうでない場合未定義動作です。例えば、上記の`std::assume_aligned()`を使用する`add()`に、何も考えずに作成した`float`配列を渡すとアライメント違反の例外によってプログラムが終了するでしょう。

```cpp
#include <memory>

// 定義略
void add(float* v1, float* v2);

int main() {
  float a1[4] = {1, 2, 3, 4};
  float a2[4] = {1, 2, 3, 4};

  add(a1, a2);  // UB、x86環境では一般保護例外が発生する
}
```

最適化の前にまず計測はよく言われることですが、この関数を使用する際は本当に必要かどうか慎重に検討したうえで、仮定が満たされるかについても考慮する必要があります。

## uses-allcator構築のためのユーティリティ

uses-allcator構築は、アロケータを用いてメモリを確保しその領域にオブジェクトを構築する際に、その使用したアロケータを適切に伝播させるための構築法の事で、特にスコープアロケータモデルにおいて必須の操作です。

このアロケータ伝播経路はかなりアドホックなものであり、アロケータを用いて構築可能な型が`std::pair`に包まれていたりするとアロケータの伝播が阻害されていました。また、その特殊ケースを処理するために`std::polymorphic_allocator::construct()`では複雑な構築をサポートしており、このような構築が将来必要な別の型についても同様の記述が必要になることが予想されていました。

uses-allcator構築における`std::pair`のハンドリングと、標準のuses-allcator構築の規定を簡素化し集約するために、uses-allcator構築を行うユーティリティが追加されます。

まず、オブジェクト構築の際にuses-allcator構築のために必要な引数列を求めるために、`std::uses_allocator_construction_args()`が追加されます。これは、`std::uses_allocator_construction_args<T>(alloc, args...)`のように使用して、型`T`のuses-allcator構築のために必要な引数列が戻り値の`std::tuple`に詰められて得られます。例えば、`T`がアロケータを用いなければ単に`alloc`が無視され、`T`が`std::pair`の場合はその内部にアロケータを伝播させるために適切な`std::pair`コンストラクタを呼び出すための引数列を返します。

`std::uses_allocator_construction_args()`は、ネストしている`std::pair`に対しても内部の`std::pair`にまで再帰的にuses-allcator構築を行います。

そして、この関数を使用してアロケータとコンストラクタ引数からオブジェクト構築を行う`std::make_obj_using_allocator()`と、未初期化領域に対してuses-allcator構築を行って初期化するための`std::uninitialized_construct_using_allocator()`が追加されます。

多くの場合はこの2つの関数を使用すればよく、`std::uses_allocator_construction_args()`を直接使用する必要はないでしょう。

```cpp
#include <memory>

using pair = std::pair<std::pmr::string, int>;

int main() {
  std::pmr::monotonic_buffer_resource mr{};
  std::pmr::polymorphic_allocator<pair> alloc{&mr};

  // uses-allcator構築によるオブジェクト構築
  pair p = std::make_obj_using_allocator<pair>(alloc, "first", 10);

  std::cout << std::boolalpha;
  std::cout << (p.first.get_allocator() == alloc) << '\n';
  std::cout << p.second << '\n';

  // メモリ領域の確保
  pair* ptr = alloc.allocate(sizeof(pair));
  // uses-allcator構築によるオブジェクト構築
  ptr = std::uninitialized_construct_using_allocator(ptr, alloc, "first", 10);
  
  std::cout << (ptr->first.get_allocator() == alloc) << '\n';
  std::cout << ptr->second << '\n';

  // オブジェクト破棄とメモリ解放
  std::destroy_at(ptr);
  alloc.deallocate(ptr, sizeof(pair));
}
```
```{style=planetext}
true
10
true
10
```

`std::make_obj_using_allocator()`は、渡されたアロケータを用いてメモリを確保してそこにオブジェクトを構築する関数ではなく、渡されたアロケータを用いたuses-allcator構築を行うだけの関数です。返されたオブジェクトはスタック領域上に配置されています。

## `polymorphic_allocator`の改良

`std::pmr::polymorphic_allocator`はC++17で追加されたアロケータで、そのメモリ確保戦略を型に示すことなく実行時に切り替えられるアロケータです。メモリアロケータの実体は`std::pmr::memory_resource`というインターフェースを実装した型で、`std::pmr::polymorphic_allocator`には`std::pmr::memory_resource`を実装したオブジェクトのポインタを渡すことでアロケータ実装を注入します。

`std::pmr::memory_resource`は非クラステンプレートであり、`std::allocator<T>`などと異なりアロケート対象の型に束縛されません。一方、`std::pmr::polymorphic_allocator<T>`はクラステンプレートであり、従来のアロケータに倣って確保する要素型`T`をテンプレートパラメータに取ります。前述のように実際にはその型はアロケーションに使用されておらず殆ど無意味でした。むしろ、このテンプレートパラメータがあることによって異なる型のアロケータが必要になった際にアロケータのrebindが必要になるなど、使用感を損ねていました。

そのため、C++20からはテンプレートパラメータのデフォルト引数として`std::byte`が指定されるようになり、`std::pmr::polymorphic_allocator<>`の形で利用できるようになります。

```{style=cppstddecl}
// <memory_resource>で定義
namespace std::pmr {

  // C++20からの宣言例
  template <class Tp = byte>
  class polymorphic_allocator {
    ...
  };
}
```

また、この変更によって`std::allocator_traits`を介さない利用がより促進されたため、単体でアロケータクラスとして活用できるように便利なアロケーション関数が追加されます。

1. `allocate_bytes(n, al)`
    - `al`アライメントで`n`バイトのメモリを確保する
    - 戻り値は確保領域のポインタ
2. `deallocate_bytes(p, n, al)`
    - `allocate_bytes()`で確保した領域を解放する
3. `allocate_object<T>(n)`
    - `T[n]`を配置するのに十分なメモリを確保する
    - 戻り値は確保領域のポインタ
4. `deallocate_object<T>(p, n)`
    - `allocate_object()`で確保した領域を解放する
5. `new_object<T>(args...)`
    - メモリを確保し`T`のオブジェクトを構築する
    - 戻り値は構築したオブジェクトのポインタ
6. `delete_object<T>(p)`
    - `new_object()`で構築したオブジェクトを破棄しメモリを解放する

```cpp
#include <memory_resource>

using std::uninitialized_construct_using_allocator;
using namespace std::ranges;
using namespace std::views;

template<int N>
auto eq = [](int& n) { return n == N; };

int main() {
  std::pmr::monotonic_buffer_resource mr{};
  std::pmr::polymorphic_allocator<> alloc{&mr};

  std::cout << std::boolalpha;
  {
    // メモリ確保と初期化
    void* ptr = alloc.allocate_bytes(sizeof(std::string), alignof(std::string));
    // アロケータ型が異なるため、アロケータは伝播しない
    auto* ps = uninitialized_construct_using_allocator(static_cast<std::string*>(ptr), alloc, "str");

    std::cout << *ps << '\n';

    // オブジェクト破棄とメモリ解放
    std::destroy_at(ps);
    alloc.deallocate_bytes(ptr, sizeof(std::string), alignof(std::string));
  }
  {
    // メモリ確保と初期化
    int* ptr = alloc.allocate_object<int>(5);
    uninitialized_fill(counted(ptr, 5), 20);

    std::cout << all_of(counted(ptr, 5), eq<20>) << '\n';

    // オブジェクト破棄とメモリ解放
    destroy(counted(ptr, 5));
    alloc.deallocate_object(ptr, 5);
  }
  {
    // メモリ確保と初期化
    auto* ps = alloc.new_object<std::pmr::string>("pmrstr");

    std::cout << *ps << '\n';
    // uses-allocator構築が行われている
    std::cout << (ps->get_allocator() == alloc) << '\n';

    // オブジェクト破棄とメモリ解放
    alloc.delete_object(ps);
  }
}
```
```{style=planetext}
str
true
pmrstr
true
```

これらの調整は`std::pmr::polymorphic_allocator<>`を語彙型（*vocabulary type*）として活用可能とすることを意図しています。

\clearpage

# 型特性（*type traits*）

この章の機能は全て、`<type_traits>`ヘッダに配置されます。

## `remove_cvref`

C++17まで、型特性`std::remove_reference`と`std::remove_cv`はそれぞれ個別に存在していました。しかし、実際のTMPにおいてはこれを2つ組み合わせてCV修飾と参照修飾を同時に除去したいことがほとんどでした。しかしちょうどこの2つを組み合わせた型特性は用意されておらず、近い振る舞いをする`std::decay`はその名前の由来となっている配列と関数のポインタへの減衰（*decay*）作用が不要だったり、ちょうどいいものがありませんでした。

この問題は標準ライブラリにおいても同様であり、利便性と標準ライブラリ規定の簡略化と適切な指定のために、CV修飾と参照修飾の除去だけを行う型特性として`std::remove_cvref`が用意されます。

```cpp
#include <type_traits>

static_assert(std::same_as<
  std::remove_cvref_t<const volatile int&>,
  int
>);

// トップレベルのCV修飾を除去する
static_assert(std::same_as<
  std::remove_cvref_t<int const * const>,
  int const *
>);

static_assert(std::same_as<
  std::remove_cvref_t<int&&>,
  int
>);


// 関数型を減衰しない
static_assert(std::same_as<
  std::remove_cvref_t<void(int)>,
  void(int)
>);

// 配列参照も参照だけを除去
static_assert(std::same_as<
  std::remove_cvref_t<int(&)[5]>,
  int[5]
>);
```

前述のように、これは`std::remove_reference`と`std::remove_cv`を組み合わせただけのものであるので、C++17までの環境においても簡単に実装できます。

```cpp
template<typename T>
using remove_cvref = std::remove_reference<std::remove_cv_t<T>>;
```

## `type_identity`

`std::type_identity<T>`は受け取った型`T`をそのまま返すだけの型特性です。

```cpp
// なにもしない
static_assert(std::same_as<
  std::type_identity_t<const int volatile>,
  const int volatile
>);
```

この関数の一番の利用シーンは、関数テンプレートにおいて特定の引数の型推論を無効化したい場合でしょう。

```cpp
#include <type_traits>

// 両方のテンプレートパラメータは同じ型
template<typename T>
void f(T x, T y) {}

int main() {
  f(10, 1.0); // ng 
}
```

この例では、`f()`は2つの引数の型が同じ一つのテンプレートパラメータによって指定されています。それによって、2つの引数のテンプレートパラメータ推論が異なる型を導く場合にテンプレートパラメータを決定できずにエラーになります。このような場合に、どちらかの引数の推論結果から残りの引数型を決定してもらうには、片方の引数をテンプレートパラメータの推論対象から外す必要があります。

```cpp
#include <type_traits>

// 第1引数だけを推論してもらう
template<typename T>
void f(T x, std::type_identity_t<T> y) {}

int main() {
  f(10, 1.0); // ok
}
```

テンプレートパラメータの推論においては、テンプレートパラメータに対してある引数の型が直接そのテンプレートパラメータを使用していない場合に型推論の対象になりません（これを、推論されないコンテキストと言ったりします）。それはまさにこの例のように、他のクラステンプレートのテンプレートパラメータとして使用されている場合が該当しており、`std::type_identity`は型に何も触らずに意図的にそれを起こすことができます。

他の使用法としては、テンプレートパラメータに型を渡す場合に渡すその場所（`T<int, char, ...>`の`<>`内）で型が不正（名前が型名を示さない場合）になると、たとえその型が使用されなくてもそこでハードエラーを起こしますが、それを回避するのにも使用できます。

```cpp
#include <type_traits>

// bによっては::typeは無効な型となる
template<bool, typename T>
struct maybe_invalid {
  using type = T;
};

template<typename T>
struct maybe_invalid<false, T> {};

// bがfalseだと無効な型名となる
template<bool b, typename T>
using maybe_invalid_t = typename maybe_invalid<b, T>::type;

template<typename T, typename U>
using cond_t = typename std::conditional_t<
    true, // 真になるとして
    T,    // 常にこっちが選択されるものの
    maybe_invalid_t<false, U> // ここでハードエラー
  >;

using test = cond_t<int, int>;  // ng
```

これを回避するには一旦クラステンプレート内部の`::type`に埋め込んで、`std::conditional_t`の結果が確定してから`type`を取得する、のようにして`::type`の取得を遅延させる方法があります。ただこの場合`T`は素の型なので`::type`に埋め込こむには別のクラス定義が必要になりますが、それはまさに`std::type_identity`です。

```cpp
#include <type_traits>

template<bool, typename T>
struct maybe_invalid {
  using type = T;
};

template<typename T>
struct maybe_invalid<false, T> {};

template<typename T, typename U>
using cond_t = typename std::conditional_t<
    true, // 真になるとして
    std::type_identity<T>,  // Tを::typeに埋め込む
    maybe_invalid<false, U> // 型名を示しているのでok
  >::type;  // 結果が出てから型名を取得

using test = cond_t<int, int>;  // ok
```

他にも色々な所で（主にTMPにおいて）活用できる地味に便利な機能です。

## `is_nothrow_convertible`

`std::is_nothrow_convertible<From, To>`は、型`From`から型`To`へ例外を投げずに暗黙変換可能であるかを調べる型特性です。`std::is_convertible`はC++11で追加されていましたが、なぜか長らくこの型特性はありませんでした。

これは、主に条件付き`noexcept`指定において使用します。

```cpp
template<typename T>
struct wrap {
  T t;

  // ラップするTに暗黙変換可能なUから構築
  template<typename U>
    requires std::is_convertible_v<U, T>
  wrap(U&& u) noexcept(noexcept(std::is_nothrow_convertible_v<U, T>))
    : t(std::forward<U>(u))
  {}
};
```

これ以外のところでは`std::is_convertible`で十分だと思われるため、条件付き`noexcept`以外ではあまり使用することはないでしょう。

## 配列型の要素数の有無の検出

`std::make_shared()`や`std::make_unique()`では、配列型がその要素数が既知かどうかで振る舞いが変わっていました。このように、配列型では要素数が既知かどうかでほぼ別の型として扱う必要が出てくることがあります。しかし、標準ライブラリにはこれを検出する方法が用意されていませんでした。

その検出のために、要素数が既知である場合に`true`となる`std::is_bounded_array`と要素数が不明の場合に`true`となる`std::is_unbounded_array`が追加されます。

```cpp
#include <type_traits>

// どちらも配列型ではある
static_assert(std::is_array_v<int[]>);
static_assert(std::is_array_v<int[1]>);

// 要素数が既知である場合にtrue
static_assert(not std::is_bounded_array_v<int[]>);
static_assert(std::is_bounded_array_v<int[1]>);

// 要素数不明の場合にtrue
static_assert(std::is_unbounded_array_v<int[]>);
static_assert(not std::is_unbounded_array_v<int[1]>);
```

\clearpage

## `unwrap_reference`

`std::unwrap_reference`は`std::reference_wrapper<T>`から`T&`を取得するための型特性です。

```cpp
#include <type_traits>

template<typename T>
auto f(T t) {
  using Ref = std::unwrap_reference_t<T>&;

  // 常に、生の参照として取得する
  Ref r = t;

  ...
}
```

`std::unwrap_reference<T>`の`T`が`std::reference_wrapper<U>`ならばその`::type`は`U&`となり、そうではない場合はその`::type`は`T`となります。この例のように、結果に`&`をつけておくとどちらにせよ参照型が得られます。

同時に、`std::unwrap_reference` + `std::decay`（参照外しや配列からポインタへの変換など）を行う`std::unwrap_ref_decay<T>`も追加されます。

```cpp
#include <type_traits>
 
template<typename T>
  requires std::constructible_from<std::unwrap_ref_decay_t<T>, int>
auto f() {
  ...
}
```

`std::unwrap_ref_decay_t<U>`の型は、関数テンプレートにおいて`U`あるいは`std::reference_wrapper<T>`の`T`の値を渡したときに推論される型、あるいは、`auto`による変数宣言時に初期化子に同じものを渡して推論される型と同じ型になります。

これらは主に、テンプレートの文脈で`std::reference_wrapper<T>`とそれ以外の型（の参照型）を統一的に扱いたい場合に使用できるでしょう。

これは、標準ライブラリ内の関数（`std::make_pair`など）の規定を簡単にするためにも使用されています。

## `common_reference`

イテレータのところで`std::iter_common_reference`として共通の参照型の概念を既に紹介していましたが、その実装に使用され、イテレータ型に限らない２つの型の間の共通の参照型を求めるための型特性が`std::common_reference`です。

```cpp
#include <type_traits>

static_assert(std::same_as<
  std::common_reference_t<int, int&>,
  int
>);

static_assert(std::same_as<
  std::common_reference_t<int&, const int&>,
  const int&
>);

static_assert(std::same_as<
  std::common_reference_t<std::string&, std::string_view>,
  std::string_view
>);
```

イテレータの時も言っていましたが、参照型と言いつつも必ずしも参照型ではありません。両方の型の値を束縛できるような型を言っています。

`std::common_reference`は`std::common_type`と似たような感じで力技によって共通の参照型を求めているため、共通の基底クラスや共通で変換できる型など、直接2つの型の間に現れないような共通の参照型を求めることができません。その場合にそれを手動でアダプトするために、`std::basic_common_reference`が用意されています。

```{style=cppstddecl}
// <type_traits>で定義
namespace std {
  template<class T, class U, template<class> class TQual, template<class> class UQual>
  struct basic_common_reference {};
}
```

通常`std`名前空間のテンプレートはユーザーが勝手に特殊化を追加することが禁じられていますが、これはその数少ない例外の1つです。例えば次のように部分特殊化を追加して、`std::common_reference`の結果をカスタムします。

```cpp
#include <type_traits>

// reference_wrapper<T>とT&の共通参照型をT&にする
namespace std {
  template<class T, template<class> class TQual, template<class> class UQual>
  struct basic_common_reference<T, reference_wrapper<T>, TQual, UQual> {
    using type = common_reference_t<TQual<T>, T&>;
  };

  template<class T, template<class> class TQual, template<class> class UQual>
  struct basic_common_reference<reference_wrapper<T>, T, TQual, UQual> {
    using type = common_reference_t<UQual<T>, T&>;
  };
}
```

これは現在C++23に向けて提案されている最中のもの（[P2655R0](https://wg21.link/p2655r0)）から拝借してきたものです。C++20時点では`std::reference_wrapper<T>`と`T&`の`std::common_reference`は`T`でしたが、これは非自明で有用ではない結果だったので修正されようとしています。

`std::basic_common_reference`を2つの型に対してこのように部分特殊化し、メンバ型`type`にその共通参照型を指定します。`std::common_reference<T, U>`と`std::common_reference<U, T>`で同じ結果を得るために引数順を入れ替えた2種類の特殊化が必要となります。

`std::basic_common_reference`の後ろ2つのテンプレート引数`TQual`と`UQual`はそれぞれ、`std::common_reference<T, U>`としたときの`T`と`U`の参照・CV修飾を伝播させるためのエイリアステンプレートです。例えば`TQual<C>`とすると、`T`の修飾を`C`にコピーした型が得られます（`T`が`int&`なら`C&`、`const int`なら`const C`など）。特殊化する際はこの部分に触れてはいけません。

`std::common_reference<T, U>`としたとき、`std::basic_common_reference<A, B>`の引数`A, B`には`T, U`を`std::remove_cvref`した型が渡され、`T, U`の修飾情報は`TQual, UQual`に保存されます。そのため、`std::basic_common_reference`の部分特殊化を定義する際は定義したい2つの型の修飾を考慮する必要はなく、順番を入れ替えた2種類だけを定義すれば良いわけです。

この例の`std::basic_common_reference`定義では、型`T`と`std::reference_wrapper<T>`の共通参照型を、非`reference_wrapper`引数から修飾を`T`にコピーした型と`T&`の間の共通参照型として求めています。別の見方では、この特殊化を通すことで`std::reference_wrapper<T>`を`T&`に変換してから`std::common_reference`を求めていると見ることもできます。

`std::common_reference<T&, std::reference_wrapper<T>>`としたときの`T&`の修飾情報は`TQual`（逆順の場合は`UQual`）に保存されているので、内部でそれを`T`に対して復帰することで、通常の参照型と参照型の間の`std::common_reference`計算に帰着させています。仮に`T`が非参照の場合は結果も非参照型になります。これは現在の振る舞いや参照型と非参照型の共通参照型の結果と一致します。

```cpp
// 先程の特殊化が定義されているとする

static_assert(std::same_as<
  std::common_reference_t<std::reference_wrapper<int>, int&>,
  int&
>);

static_assert(std::same_as<
  std::common_reference_t<std::reference_wrapper<int>, const int&>,
  const int&
>);

static_assert(std::same_as<
  std::common_reference_t<std::reference_wrapper<int>, int&&>,
  const int&
>);

static_assert(std::same_as<
  std::common_reference_t<std::reference_wrapper<int>, int>,
  int
>);
```

## レイアウト/ポインタ互換性の判定

クラスのレイアウトにまつわるいくつかの事を判定するための型特性が4つ追加されます。

```{style=cppstddecl}
namespace std {
  
  // 2つの型のレイアウト互換性を判定する
  template<class T, class U>
  struct is_layout_compatible;

  // 派生クラスと基底クラスのポインタの相互変換可能性を判定する
  template<class Base, class Derived>
  struct is_pointer_interconvertible_base_of;
  
  // クラスのポインタとその先頭メンバのポインタの相互変換可能性を判定する
  template<class S, class M>
  constexpr bool is_pointer_interconvertible_with_class(M S::*m) noexcept;

  // 2つのメンバポインタがクラス内で対応する位置にあるかを判定
  template<class S1, class S2, class M1, class M2>
  constexpr bool is_corresponding_member(M1 S1::*m1, M2 S2::*m2) noexcept;
}
```

`std::is_pointer_interconvertible_with_class()`と`std::is_corresponding_member()`は、他の型特性と異なり`constexpr`な関数です。

これらの型特性はクラスのレイアウト（メモリ上にどのように配置されているか）に関連するものであり、レイアウトに互換性があればオブジェクトの型と異なる型として値の読み出しが可能となる場合があり、それによってポインタの相互変換が可能となることがあります。これらのものが必要になるケースはあまり多くなさそうですが、クラスのレイアウトに直接触れる場合やポインタの変換を行う場合などにこれらの型特性によって対象の型に静的なチェックをかけておくことができます。

ただし、クラス型の場合にこれらのことが可能となるのは殆どの場合にスタンダードレイアウト型に限られています。スタンダードレイアウト型とは、とても簡単に言えばCの構造体のようなクラス型のことです（トリビアル型でなくても良いため、コンストラクタ等を定義することはできます）。

以下、いくつかの例。

```cpp
#include <type_traits>

struct B {
  int n = 0;
};

struct D1 : B {
};

struct D2 : B {
  int m = 0;
};

static_assert(
  std::is_pointer_interconvertible_base_of_v<B, D1>;
);


// 派生クラスがメンバを持っているとfalseになる
static_assert(
  not std::is_pointer_interconvertible_base_of_v<B, D2>;
);


void test1(D1* p) {
  B* p2 = reinterpret_cast<B*>(p);  // ok
}

void test2(D2* p) {
  B* p2 = reinterpret_cast<B*>(p);  // ub
}
```

2つのクラスの共通先頭メンバ列（*common initial sequence*）とは、2つのクラスの先頭からの非静的メンバの型の互換性と並びが一致している部分を言います。これは特に、共用体に詰めた時にもそのアクティブメンバにかかわらずアクセスすることができる領域であり、タグ付き共用体で活用されます。

`std::is_corresponding_member()`はメンバポインタを2つ受けて、それぞれがクラスの内部で対応する位置に配置されていてなおかつレイアウト互換のある型かを判定するものです。厳密には共通先頭メンバ列を判定するものではありませんが、位置とレイアウトの対応関係をチェックしておくことができます。

```cpp
#include <type_traits>

struct T1 {
  int idx = 0;
  std::string str;
};

struct T2 {
  const int id = 1;
  double d;
};

union U {
  T1 t1;
  T2 t2;
};

static_assert(
  std::is_corresponding_member(&T1::idx, &T2::id)
);


void test(U & u) {
  // 共通先頭メンバ列はアクティブメンバによらず読み出せる
  int i = u.t1.idx;  // ok

  if (i == 0) {
    std::cout << u.str;
  } else if (i == 1) {
    std::cout << u.d;
  }
}
```

これらの例のように、これらの型特性を使用した`static_assert`によるチェックを入れておくことでコンパイル毎にチェックが自動化され、後から型の定義を変更して期待する性質が失われた場合にもコンパイルエラーとして早期発見することができます。これらの型特性によって指定される性質は非常に難解であり、思わぬ変更によって容易に失われてしまう可能性があるため、このような静的なチェックを行っておくと確実です。

\clearpage

# その他

ここでは、ここまでのもののようにある程度のカテゴリに納めづらいものをまとめておきます。

## `filesystem::path`の`char8_t`文字列からの構築

`filesystem::path`はコンストラクタにパス文字列を渡して構築しますが、C++17までは文字型でUTF-8エンコーディングを判別できないことからUTF-8の文字列を渡すことが禁止されていました。例えば、C++17までの`u8""`リテラルは`char`型の文字列でしたが、Windows環境では`char`型のエンコーディングはほとんどの場合UTF-8ではなく、`char`の文字列が渡されたときにどのエンコーディングで解釈するべきかが曖昧になります（C++17では非UTF-8と仮定していました）。

このため、`filesystem::path`をUTF-8文字列から構築するためのヘルパ関数`filesystem::u8path()`が用意されていました。

```cpp
#include <filesystem>

namespace fs = std::filesystem;

int main() {
  // 環境によっては文字化けする（charのエンコーディングとして構築）
  fs::path p1{u8"C:\\フォルダ\\ファイル"};
  // 文字化けしない（UTF-8エンコーディングから変換して構築）
  auto p2 = fs::u8path(u8"C:\\フォルダ\\ファイル");

  std::cout << p1 << '\n';
  std::cout << p2 << '\n';
}
```

godboltのMSVC 19.33で`/std:c++17`を指定した時の結果

```{style=planetext}
"C:\\ãƒ•ã‚©ãƒ«ãƒ€\\ãƒ•ã‚¡ã‚¤ãƒ«"
"C:\\フォルダ\\ファイル"
```

C++20で`char8_t`が導入され`u8""`リテラルも`char8_t`文字列型を返すようになり、文字型によってUTF-8エンコーディングを判別可能となったため、`filesystem::path`のコンストラクタに`char8_t`文字列からの構築が追加され、`u8""`リテラルから直接構築できるようになります。

```cpp
#include <filesystem>

namespace fs = std::filesystem;

int main() {
  // どちらも文字化けしない（UTF-8エンコーディングから変換して構築）
  fs::path p1{u8"C:\\フォルダ\\ファイル"};
  auto p2 = fs::u8path(u8"C:\\フォルダ\\ファイル");

  std::cout << p1 << '\n';
  std::cout << p2 << '\n';
}
```

godboltのMSVC 19.33で`/std:c++20`を指定した時の結果

```{style=planetext}
"C:\\フォルダ\\ファイル"
"C:\\フォルダ\\ファイル"
```

ちなみに、`filesystem::u8path()`も`char8_t`文字列を受け取れるようにされているため`u8""`リテラルの破壊的変更の影響を受けません。ただし、`filesystem::path`のコンストラクタが`char8_t`文字列から構築できるようになったことで、`filesystem::u8path()`の役割は無くなってしまったため、C++20からは非推奨となっています。

## ディレクトリ作成系関数のエラー挙動の変更

`<filesystem>`の`create_directory()`と`create_directories()`はどちらもディレクトリを作成するもので、ディレクトリが作成された場合に`true`を返します。ただ、ディレクトリは作成できなかったもののエラーではないとみなされる場合があり、この戻り値とエラーの関係に関しては複雑です。

```cpp
#include <filesystem>

namespace fs = std::filesystem;

int main() {
  std::error_code ec;

  if(fs::create_directory("a/b/c", ec) == false) {
    // ここに来るのはどんな時？
    assert(bool(ec)); // ecの状態は??
  }

  if (fs::create_directories("d/e/f", ec) == false) {
    // ここに来るのはどんな時？
    assert(bool(ec)); // ecの状態は??
  }
}
```

`create_directory()`は指定されたディレクトリだけを作成し、`create_directories()`はそのパス中に含まれる存在していないディレクトリも作成するものです。`create_directories()`は`create_directory()`をパスの先頭から順番に試行していくものなので、本質的な問題は`create_directory()`にあります。

この戻り値とエラー値の状態に関しては紆余曲折がありましたが、最終的な`create_directory("path", ec)`の呼び出しに対する、事後状態と戻り値及びエラー（`ec`）状態の組み合わせは次のようになります。

|状態|戻り値|`ec`|
|---|---|---|
|ディレクトリを作成した|`true`|正常|
|ディレクトリが既に存在する|`false`|正常|
|同名のファイルが既に存在する|`false`|エラー|
|パス中のディレクトリが存在しない|`false`|エラー|
|その他のディレクトリ作成失敗|`false`|エラー|

`ec`のエラー状態とは、`bool(ec) == true`となる状態です。

振る舞いとしては、とにかくディレクトリを作成した場合にのみ`true`を返し、ディレクトリを作成せず関数が戻った後もディレクトリが存在しない場合にエラーとして通知されます。

これはC++17に対する欠陥報告なので、一部の実装では以前からこうなっていた可能性があります。

## 安全な整数型の比較

C++における整数型の比較においては、符号有無が混ざると複雑な暗黙変換が介在することによって意図しない結果となることがあります。

```cpp
int main() {
  std::cout << std::boolalpha;
  
  std::cout << (-1 < 0u) << '\n';
  std::cout << (-1 < 0) << '\n';
}
```
```{style=planetext}
false
true
```

`-1 < 0u`では、左辺のオペランドが`int`型（符号付き32bit整数）、右辺のオペランドが`unsigned int`型（符号なし32bit整数）であり、暗黙変換（汎整数拡張）によって両編の型が`unsigned int`に揃えられ、左辺の値は`unsigned int`にキャストされ32bit符号なし整数値の最大値`4294967295`になります。結果、`-1 < 0u`は`4294967295 < 0`という比較をしていることになり、これは`false`になります。

一方、`-1 < 0`の方は両辺が共に`int`型のため変換は行われず、その値通りの比較結果となります。

これはよく知られている問題であるため、コンパイラは符号有無が混在した整数値の比較に対して警告を発してくれます。

```cpp
void f(std::vector<int> vec) {
  int n = 10;

  // ここの条件式で警告される
  if (n < vec.size()) {
    ...
  }

  // 真ん中の継続条件の式で警告される
  for (int i = 0; i < vec.size(); ++i) {

  }
}
```

この警告を消すには両辺の整数型の符号有無を揃える必要があるのですが、そのためにはキャスト（C++的に正しいのは`static_cast`）をしなければならず、これは構文的にかなりノイズになります。また、どちらのオペランドをキャストするのかも重要になります。そのような事情によって、この警告への対処は簡単にいつも行えるものではなく無視されてしまうことすらあります。

これらの問題に対処するために、安全な整数値の比較を行う関数が追加されます。

```{style=cppstddecl}
// <utility>で定義
namespace std {
  // t == u
  template <class T, class U>
  constexpr bool cmp_equal(T t, U u) noexcept;

  // t != u
  template <class T, class U>
  constexpr bool cmp_not_equal(T t, U u) noexcept;

  // t < u
  template <class T, class U>
  constexpr bool cmp_less(T t, U u) noexcept;

  // t > u
  template <class T, class U>
  constexpr bool cmp_greater(T t, U u) noexcept;

  // t <= u
  template <class T, class U>
  constexpr bool cmp_less_equal(T t, U u) noexcept;

  // t >= u
  template <class T, class U>
  constexpr bool cmp_greater_equal(T t, U u) noexcept;
}
```

これらの関数は`t`を左辺オペランド、`u`を右辺オペランドとしてその名前通りの比較を行う関数です。なおかつ、2つの引数型の符号有無が異なっている場合でもその実際の値による比較と一致した結果を返し、警告も表示されません。

```cpp
#include <utility>

int main() {
  std::cout << std::boolalpha;

  std::cout << std::cmp_less(-1, 0u) << '\n';
  std::cout << std::cmp_less(-1, 0) << '\n';
  std::cout << std::cmp_greater(-1, 0u) << '\n';
  std::cout << std::cmp_greater(-1, 0) << '\n';
}
```
```{style=planetext}
true
true
false
false
```

これらの関数を用いることで、符号有無が混在する整数値の比較をしているようなコードにおける安全性を改善し警告の抑制を行うことができます。

また、ある整数値がある整数型で表現可能かどうか（その整数型の最小値と最大値の範囲内に収まっているか）を調べるための`std::in_range()`も用意されます。

```cpp
#include <utility>

int main() {
  std::cout << std::boolalpha;

  std::cout << std::in_range<std::uint32_t>(-1) << '\n';
  std::cout << std::in_range<std::int8_t>(128) << '\n';
  std::cout << std::in_range<std::uint32_t>(0) << '\n';
  std::cout << std::in_range<std::int8_t>(127) << '\n';
}
```
```{style=planetext}
false
false
true
true
```

`std::in_range<T>(n)`が`false`になるということは、値`n`を型`T`にキャストすると意図しない値になる可能性があるということです。

## `condition_variable_any`の待機関数の`stop_token`サポート

`std::stop_token`の導入に伴って、`std::condition_variable_any`の待機系関数に`std::stop_token`を取るオーバーロードが追加されます。対象となる関数は、`.wait()`、`.wait_until()`、`.wait_for()`の3つです。

これらの関数における条件変数の待機では、条件変数に対する起床シグナル（`.notify_all()`）に加えて`std::stop_token`に対する停止要求（`std::stop_source::request_stop()`）でも待機が解除されます。

これによって、条件変数使用時に待機の解除よりも先に処理が完了（キャンセル）される場合の後処理を簡易化できるようになります。

```cpp
#include <condition_variable>

using namespace std::chrono_literals;

int main() {
  int shared_state = 0;
  std::mutex mtx;
  std::condition_variable_any cv;

  // データ読み取りスレッド
  std::jthread reader{[&shared_state, &mtx, &cv](std::stop_token st) {
    int before = 0;

    while (true) {
      std::unique_lock lock{mtx};

      // データ更新を待機
      // ここの待機は、stに対する停止要求によっても解除される
      cv.wait(lock, st, [&before, &shared_state] {
        return before < shared_state;
      });

      // 停止要求がなされた時だけ終了する
      if (st.stop_requested()) {
        return;
      }

      // データを使用する
      before = shared_state;
      ...
    }
  }};

  // データ更新スレッド
  std::jthread writer{[&shared_state, &mtx, &cv](std::stop_token st) {
    while (not st.stop_requested()) {
      {
        std::lock_guard lock{mtx};
        // データの更新
        shared_state += 1;
        // 待機の解除
        cv.notify_all();
      }
      // 100ms周期でデータ更新
      std::this_thread::sleep_for(100ms);
    }
  }};

  // 5秒後に処理を停止
  std::this_thread::sleep_for(5s);
}
```

この例では、2つのスレッドの終了は`main()`の終了に伴う`std::jthread`のデストラクタからの停止要求によってのみ行われます。`writer`スレッドの方は`stop_token`の`.stop_requested()`をチェックしてループしているため、`100ms`のウェイトが明ければ素直に終了します。一方、`reader`スレッドは条件変数を用いて待機していますが、`.wait()`に`stop_token`を渡していることによって条件変数が`stop_token`に対する停止要求でも起床するため、起床明けの`.stop_requested()`チェックによって正常に終了することができます。

このように、条件変数によって待機するスレッドにおいても`stop_token`による安全な協調的キャンセルの恩恵を受けることができます。

3つの関数すべてで、`std::stop_token`を取るオーバーロードが追加されるのは述語を取るオーバーロードに対してで、`std::stop_token`は第2引数（ロックオブジェクトの後）で受け取ります。述語が必須となっているのは、*Spurious Wakeup*と呼ばれる現象（通知も停止要求も起きていないのに待機が解除される現象）が起きてしまった場合に判定がややこしくなるのを回避するためだと思われます。

なお、これらのものは`std::condition_variable`の同名関数群には用意されていません。事情は複雑ですがどうやら、安全に実装しようとすると余計なオーバーヘッドがかかってしまい、本来`std::condition_variable`の提供する効率性が（静かに）失われてしまうためのようです。

## `visit<R>()`

`std::variant<Ts...>`に対する`std::visit(f, v)`は`std::variant`オブジェクト`v`の状態に応じて`f`の適切なオーバーロードを呼び分けてくれる関数で、`std::variant`を扱う時にとても便利な関数です。ただし、この関数は`std::variant<Ts...>`の型`Ts...`毎に呼ばれる関数の全てが同じ戻り値型を返すことを要求しており、戻り値型が異なる場合でも変換してくれたりはせずにコンパイルエラーになります。

暗黙変換せずにコンパイルエラーにするという挙動は安全に倒した設計であるため問題があるわけではありませんが、意図的に戻り値型を集約したい場合などに少し不便でした。そのため、明示的に戻り値型を指定することで暗黙変換による戻り値型の集約を行うオーバーロード、`std::visit<R>()`が追加されます。

　

```cpp
#include <variant>

int main() {
  std::variant<int, double> var{1.0};

  // ng、戻り値型が一致しない
  double r = std::visit([](auto v) {
    return v;
  }, var);
  
  // ok、doubleに変換する
  double r = std::visit<double>([](auto v) {
    return v;
  }, var);
}
```

`std::visit<R>()`は集約先の戻り値型`R`を明示的に指定すること以外は`std::visit()`と同様に使用できます。

## `to_address()`

より一般的なアロケータを考慮すると、そのメモリ確保関数（`allocate()`）によって返されるポインタは必ずしも生のポインタではない可能性があります。アロケータの確保関数はオブジェクトを構築しないので、確保後にその領域にオブジェクトを構築する必要があり、そのためには確保した領域の生のポインタ（アドレス）が必要になります。メモリ確保関数が生のポインタを返していない場合、そこからアドレスを正しく取得しなければなりません。

標準ライブラリにはオブジェクトからその配置ストレージのポインタを取得するための`std::addressof()`がありますが、これは既に構築済みのオブジェクトからその配置アドレスを取得するものであり、オブジェクトが構築されていない場合に使用できるものではありません。

```cpp
// 確保した領域をshared_ptrによって返すアロケータ
template<typename T>
struct my_allocator {

  auto allocate() -> std::shared_ptr<T> {
    ...
  }

  ...
};

int main() {
  my_allocator<int> alloc{};

  // メモリ確保
  std::shared_ptr<int> p = alloc.allocate();  // ok

  // オブジェクト構築
  std::allocator_traits<A>::construct(a, std::addressof(*p), 0);  // ub
  //                                                    ^^

}
```

スマートポインタであるかどうかにかかわらず、オブジェクトの構築されていない領域へのポインタのデリファレンスは未定義動作になります。

このような場合に適切にそのアドレスを取得するためのものがありませんでしたが、C++20からは`std::to_address()`が用意されます。

`std::to_address(p)`はポインタlikeな（ポインタ意味論を持つ）型のオブジェクトからその参照する領域の生のポインタを取得する関数です。これは生のポインタ型に対しても、スマートポインタ型に対しても動作します。

```cpp
// 確保した領域をshared_ptrによって返すアロケータ
template<typename T>
struct my_allocator {

  auto allocate() -> std::shared_ptr<T> {
    ...
  }

  ...
};

int main() {
  my_allocator<int> alloc{};

  // メモリ確保
  std::shared_ptr<int> p = alloc.allocate();  // ok

  // オブジェクト構築
  std::allocator_traits<A>::construct(a, std::to_address(p), 0);  // ok

}
```

`std::to_address(p)`はユーザー定義のポインタlikeな型に対しては`std::pointer_traits::to_address(p)`か`std::to_address(p.operator->())`のどちらかからそのアドレスを取得しようとし、生のポインタ型に対しては`p`をそのまま返します。そのため、ユーザー定義型では`std::pointer_traits`を特殊化するか、`operator->`によってアドレスを返すようにしておきます。

`std::to_address()`はまた、イテレータが`contiguous_iterator`であるかをチェックする際にも使用されます。その場合、イテレータ型はポインタ型ではないため、`std::pointer_traits`特殊化ではなく`operator->`によって`std::to_address()`にアダプトすることが推奨され、これによって`contiguous_iterator`では実質的に`operator->`が要求されます。

\clearpage

# 大域的な変更

ここでは、大きくはないものの同種の変更がより広いライブラリ機能にわたるものについて簡単にまとめておきます。その性質上、個別に列挙する場合は抜けがあるかもしれませんがご容赦ください。

## 標準ライブラリ型の`<=>`

C++20で三方比較演算子`<=>`が追加され、クラス型の比較演算子の実装をかなり簡単に行えるようになります。しかし、それにはクラスのメンバ型が全て再帰的に`<=>`を持っている必要があり、標準ライブラリのクラス型が`<=>`を持っていないと標準ライブラリのクラス型をメンバに持つクラスでは`<=>`の恩恵に与れなくなります。そのため、標準ライブラリのほとんどのクラス型はC++20の時点で`<=>`を持っています。量が多いので全部を記しませんが、C++17時点で比較演算子を持っているクラス型は全て`<=>`を持つようになっているはずです。

```cpp
#include <compare>

int main() {
  std::vector v1 = {1, 2, 3};
  std::vector v2 = {1, 2};

  auto cmp1 = v1 <=> v2;  // ok

  std::tuple t1 = {1, 1.0};
  std::tuple t2 = {1, 1.1};

  auto cmp2 = t1 <=> t2;  // ok
}
```

もっとも、通常比較が必要なところで`<=>`を使用することはほぼないでしょう。前述のように、これは`<=>`による比較演算子の自動実装を妨げないために定義されています。

```cpp
#include <compare>

struct S {
  std::vector<int> vec;
  std::pair<int, double> pair;
  std::string_view str;

  auto operator<=>(const S&) const noexcept = default;  // ok
};

int main() {
  S s1{};
  S s2{};

  auto cmp = s1 <=> s2; // ok
}
```

一部の標準ライブラリのクラス型にも、`<=>/==`による自動実装を活用して演算子定義を削減しているものがあります。

## デフォルトコンストラクタの非`explicit`化

標準ライブラリの一部の型では、デフォルトコンストラクタが`explicit`指定されていることがあり、その場合はコピーリスト初期化が行えなくなります。

```cpp
#include <stack>

int main() {
  std::stack<int> s1;       // ok
  std::stack<int> s2{};     // ok
  std::stack<int> s2 = {};  // ng、空リストによるコピーリスト初期化
  s1 = {};                  // ng、空リストによるコピーリスト初期化
}
```

これはこれらの型が別のクラスのメンバになっている場合にそのクラスにまで伝播します。

意図的に`explicit`されているものを除いて、この`std::stack`などのデフォルトコンストラクタが`explicit`指定されているのは事故だったようで、C++20では`explicit`が取り除かれます。

コンテナアダプタ、乱数エンジン、分布生成器、文字列ストリーム等がこの変更の対象であり、C++17まではそのコンストラクタが1つ以上の引数を受けているものの、デフォルト引数が指定されていることによってデフォルトコンストラクタとなっていた、ようなコンストラクタが対象です。

C++20では、そのようなコンストラクタが引数を取らない非`explicit`デフォルトコンストラクタとデフォルト引数無しのコンストラクタに分割されることによって、デフォルトコンストラクタの`explicit`が取り除かれます。このため、API的には変更がありません。

## `[[nodiscard]]`

`[[nodiscard]]`属性はC++17で追加されたもので、関数の戻り値やオブジェクトを使わずに捨ててしまう場合に警告するものです。C++20では、この属性がいくつかの標準ライブラリの関数に指定されるようになります。

- `::operator new`
- `::operator new[]`
- 標準アロケータの`allocate()`
    - `std::allocator`
    - `std::allocator_traits`
    - `std::scoped_allocator_adaptor`
- `std::pmr::memory_resurce`の`allocate()`
- `std::pmr::polymorphic_allocator`
    - `.allocate()`
    - `.allocate_bytes()`
    - `.deallocate_bytes()`
    - `.allocate_object()`
- `std::lunder()`
- `std::assume_aligned()`
- `std::async()`
- 標準コンテナの`.empty()`及び`std::empty()`
    - `std::string`、`std::string_view`、`std::filesystem::path`、`std::match_results`も含む
- `std::stop_source`
    - `.get_token()`
    - `.stop_possible()`
    - `.stop_requested()`
    - `operator==/!=`
- `std::stop_token`
    - `.stop_possible()`
    - `.stop_requested()`
    - `operator==/!=`
- `std::barrier`
    - `.arrive()`
- `std::jthread`
    - `.get_stop_source()`
- `std::ranges::subrange`
    - `.begin()`
        - イテレータが`copyable`ではない場合のみ
    - `.next()`
- `std::rotl()`
- `std::rotr()`

## `constexpr`

次の関数はC++20から`constexpr`指定されるようになり、定数式で使用可能となります。

- 全メンバ関数
    - `std::array`
    - `std::vector`
    - `std::string`
    - `std::allocator`
    - `std::allocator_traits`
    - `std::back_insert_iterator`
    - `std::front_insert_iterator`
    - `std::insert_iterator`
    - `std::reference_wrapper`
    - `std::complex`
    - `std::pair`
    - `std::tuple`
- `std::back_inserter()`
- `std::front_inserter()`
- `std::inserter()`
- `<algorithm>` : 以下のものを除くイテレータアルゴリズム
    - 並列アルゴリズム
    - `std::shuffle()`
    - `std::sample()`
    - `std::stable_sort()`
    - `std::stable_partition()`
    - `std::inplace_merge()`
- `<numeric>`
    - 並列アルゴリズムを除く数値アルゴリズム
    - `std::gcd()`
    - `std::lcm()`
    - `std::midpoint()`
- `std::invoke()`
- `std::not_fn()`
- `std::bind()`
- `std::mem_fn()`
- `std::char_traits`
    - `move()`
    - `copy()`
    - `assign()`
- `std::pointer_traits::pointer_to()`
- `std::swap()`
- `std::exchange()`

## 非推奨

次のものは、C++20で非推奨となりました。削除されるまでは使用することができますが、なるべく早期に使用を辞めることが推奨されます。

- `std::relops`
- `std::is_pod`
- `volatile`なクラステンプレート特殊化
    - `std::tuple_size`
    - `std::tuple_element`
    - `std::variant_size`
    - `std::variant_alternative`
    - `std::move_iterator::operator->()`
    - `std::basic_string::reserve()`のデフォルト引数
    - `std::filesystem::u8path()`
- アトミック関連
    - `std::atomic_init()`
    - `ATOMIC_VAR_INIT`
    - `ATOMIC_FLAG_INIT`
    - `std::shared_ptr`のためのアトミックフリー関数
    - ロックフリーではない`std::atomic`特殊化の`volatile`メンバ関数
- ロケールカテゴリファセット
    - `std::codecvt<char16_t, char, mbstate_t>`
    - `std::codecvt<char32_t, char, mbstate_t>`
    - `std::codecvt_byname<char16_t, char, mbstate_t>`
    - `std::codecvt_byname<char32_t, char, mbstate_t>`

## 削除

次のものは、C++20で削除されたものです。非推奨と異なり、削除されたものは使用できなくなります。

- `std::allocator`のメンバ
    - `size_type`
    - `difference_type`
    - `pointer`
    - `const_pointer`
    - `reference`
    - `const_reference`
    - `rebind`
    - `.address()`
    - `.allocate()`
    - `.max_size()`
    - `.construct()`
    - `.destroy()`
- `std::allocator<void>`
- `std::is_literal_type`
- `std::get_temporary_buffer()`
- `std::return_temporary_buffer()`
- `std::raw_storage_iterator`
- `std::not1()`
- `std::not2()`
- `std::unary_negate`
- `std::binary_negate`
- 標準関数オブジェクトのメンバ型
    - `result_type`
    - `argument_type`
    - `first_argument_type`
    - `second_argument_type`
- `std::shared_ptr::unique()`
- `std::result_of`
- `std::uncaught_exception()`
- C互換ヘッダ
    - `<ccomplex>`
    - `<cstdalign>`
    - `<cstdbool>`
    - `<ctgmath>`
    - `<ciso646>`

\clearpage

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- cppmap(https://cppmap.github.io/ : ライセンスはCC0 パブリックドメイン)
- C++マルチスレッド一巡り  
  (https://zenn.dev/yohhoy/books/cpp-stdlib-multithreading)
- C++の64bit版のhash_combine(https://suzulang.com/cpp-64bit-hash-combine/)
- C++20 atomic_ref - Marius Bancila's Blog  
  (https://mariusbancila.ro/blog/2020/04/21/cpp20-atomic_ref/)