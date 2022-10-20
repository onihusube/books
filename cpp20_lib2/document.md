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

これまで、イテレータ型の各種情報を問い合わせるのには`std::iterator_traits`を使用していましたが、C++20からはそれに変わるより簡易な手段が提供されるようになります。また同時に、従来はなかった追加の情報を取得するための手段も提供されます。

### 距離型 - `difference_type`

`difference_type`はイテレータの距離を表す型で、イテレータの差分操作（`operator-`）の戻り値型でもあります。

従来は`std::iterator_traits<I>::difference_type`から取得していましたが、C++20からは`std::iter_difference_t<I>`を用いる事で同じものを取得できます。

```cpp
#include <iterator>

int main() {
  using iota_view_iter = std::ranges::iterator_t<std::ranges::iota_view<int>>;

  static_assert(std::same_as<std::iter_difference_t<std::vector<int>::iterator>, std::ptrdiff_t>);
  static_assert(std::same_as<std::iter_difference_t<int*>, std::ptrdiff_t>);
  static_assert(std::same_as<std::iter_difference_t<iota_view_iter>, std::ptrdiff_t>);
}
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

int main() {
  using iota_view_iter = std::ranges::iterator_t<std::ranges::iota_view<unsigned int>>;

  static_assert(std::same_as<std::iter_value_t<std::vector<int>::iterator>, int>);
  static_assert(std::same_as<std::iter_value_t<double*>, double>);
  static_assert(std::same_as<std::iter_value_t<iota_view_iter>, unsigned int>);
}
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

int main() {
  using iota_view_iter = std::ranges::iterator_t<std::ranges::iota_view<unsigned int>>;

  static_assert(std::same_as<std::iter_reference_t<std::vector<int>::iterator>, int&>);
  static_assert(std::same_as<std::iter_reference_t<double*>, double&>);
  static_assert(std::same_as<std::iter_reference_t<iota_view_iter>, unsigned int>);
}
```

`reference`というのは歴史的経緯から来る名前で、イテレータの間接参照の戻り値型は必ずしも参照型ではなくてもokです。

`std::iter_reference_t`は次のように定義されています。

```cpp
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

int main() {
  using iota_view_iter = std::ranges::iterator_t<std::ranges::iota_view<unsigned int>>;
  
  static_assert(std::same_as<std::iter_rvalue_reference_t<std::vector<int>::iterator>, int&&>);
  static_assert(std::same_as<std::iter_rvalue_reference_t<double*>, double&&>);
  static_assert(std::same_as<std::iter_rvalue_reference_t<iota_view_iter>, unsigned int>);
}
```

イテレータを`i`とすると、大抵の場合は`decltype(std::move(*i))`の型を取得することになりますが、`*i`がprvalueを返す場合はその型をそのまま取得します。

例えばイテレータの要素への右辺値を別の型（例えば`std::tuple`など）に詰めて転送したい場合、`*i`がprvalueを返す時に右辺値参照を使用すると危険な（ダングリング参照になる）ためその場合は素の型をそのまま使用するようにするハンドリングを自動で行いたい場合に利用できます。他には、コンセプトによる制約を使用する場合に`*i`をムーブするための適切な型を取得するためにも使用できます。

以下、ここからは`std::iterator_traits`にはなかったものが続きます。

### 共通の参照型

`std::iter_common_reference_t`は、`std::iter_value_t<I>&`と`std::iter_reference_t<I>`の両方を束縛することのできるような共通の参照型を表すものです。

```cpp
#include <iterator>

int main() {
  using iota_view_iter = std::ranges::iterator_t<std::ranges::iota_view<unsigned int>>;

  static_assert(std::same_as<std::iter_common_reference_t<std::vector<int>::iterator>, int&>);
  static_assert(std::same_as<std::iter_common_reference_t<double*>, double&>);
  static_assert(std::same_as<std::iter_common_reference_t<iota_view_iter>, unsigned int>);
}
```

これも`reference`といいつつ、必ずしも参照型であるとは限りません。

`std::iter_common_reference_t`は次のように定義されています。

```cpp
namespace std {
  template<indirectly_readable I>
  using iter_common_reference_t = common_reference_t<iter_reference_t<I>, iter_value_t<I>&>;
}
```

普通のイテレータでは`std::iter_reference_t<I>`と同じ型になると思われますが、例えば間接参照がprvalueを返すイテレータではその型を`T`とすると`const T&`などになります。

イテレータを用いたアルゴリズムを書く際にこのような性質を持つ型が必要になることがよくあるため、それを簡易に求めたい時に使用できます。

### イテレータを用いた関数呼び出しの結果型

`std::indirect_result_t`は、イテレータ型を間接参照して関数に入力して呼び出した時の戻り値型を取得するものです。

```cpp
#include <iterator>

template<typename T>
auto comp(const T& lhs, const T& rhs) -> bool;

int main() {
  using f_t = decltype(&comp<int>);
  using vecit = std::vector<int>::iterator;

  static_assert(std::same_as<
                  std::indirect_result_t<f_t, int*, int*>, 
                  bool>);
  static_assert(std::same_as<
                  std::indirect_result_t<f_t, vecit, vecit>, 
                  bool>);
}
```

イテレータ`i1, i2, ..., in`とその要素を渡して呼びだす関数`f`がある時、`f(*i1, *i2, ..., *in)`と呼んだ時の戻り値型を取得するものです。

例えばこのような比較関数や`swap`など、イテレータを用いたアルゴリズム中でこのような呼び出しを行う関数は頻出します。コンセプトの文脈でそれらの呼び出しやその結果を制約する際に`std::indirect_result_t`を使用できます。

## イテレータコンセプト

C++20より、イテレータという概念はコンセプトによって定義されるようになります。それに伴い、`<iterator>`には細分化され階層化されたイテレータコンセプトが定義されるようになります。

### `indirectly_readable`

`std::indirectly_readable`は間接参照によって値を読み出す事ができることを表すコンセプトです。

```cpp
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

等しさの保持とは、式の実行に伴って副作用を及ぼしたり内部状態に依存して結果が変わらないことを言います。つまりは、`*i`が副作用や内部状態の更新を持たないことを言っています。

### `indirectly_writable`

`std::indirectly_writable`は間接参照先に値を書き込む事ができることを表すコンセプトです。

```cpp
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

`std::indirectly_writable<Out>`は、`Out`のオブジェクトが間接参照演算子（`operator*`）によって型`T`の値を書き込む事ができる場合に`true`となります。これもまた、イテレータ型だけではなくポインタ型やスマートポインタ型でも満たす事ができます。

このコンセプトを構成する4つの制約式は全て、等しさを保持することを要求されません。つまりは、`*o`が内部状態を更新することを認めています。

このコンセプトには意味論要件が指定されています

型`T`の値`e`と間接参照可能な型`Out`のオブジェクト`o`について

- 型`Out, T`が`indirectly_readable<Out> && same_as<iter_value_t<Out>, decay_t<T>>`のモデルとなる場合、`e`を4つの制約式のいずれかによって出力した後で`*o`と`e`は等値となる

これはイテレータを介した出力操作に当然要求される事でしょう。これを満たさないようなものは出力とは言えません。前提条件の`Out, T`へのコンセプト要求は、出力だけが可能で読み取りできない場合や出力時に変換される場合を除くための条件です。

定義中の`const_cast`を用いる制約式は、間接参照が*pravlue*を返すようなイテレータを受け入れるためのものです。例えば、プロクシイテレータと呼ばれるイテレータは要素へのプロクシオブジェクトを`operator*`から返し、その戻り値型は参照型ではなくプロクシオブジェクトの型そのものになります。その場合でも、プロクシオブジェクトは`operator*`によって参照先へ書き込むことができるため、プロクシイテレータは`indirectly_writable`となる必要があります。一方で、`std::string`など、*prvalue*への出力（`std::string() = std::string()`）ができてしまうけれど明らかにプロクシオブジェクトではないようなものが考えられ、そのような場合には`indirectly_writable`とならないようにしなければなりません。

`const_cast`を用いる2つの制約式は、`Out`のオブジェクト`o`の`*o`が*prvalue*を返す場合にのみその上にある2つの制約式と異なった振る舞いをし、`*o`を`const T&&`にキャストした後でも出力が可能かどうかを調べることでプロクシイテレータとそうでないもの（`std::string() = std::string()`のようなもの）を判別しています。

### `weakly_incrementable`

`std::weakly_incrementable`は、インクリメント操作が可能であることを表すコンセプトです。

```cpp
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

このコンセプトには意味論要件が指定されています

型`I`のオブジェクト`i`について

- `++i`と`i++`は同じ定義域をもつ
- `i`がインクリメント可能ならば、`++i`と`i++`は`i`を次の要素へ進める
- `i`がインクリメント可能ならば、`addressof(i)`はインクリメント前後で変化しない

定義域とは、式（関数）の引数全体からその式を不正にするような入力を除いた残りの部分（集合）の事を言います。インクリメント操作は前置/後置に関わらず進行に関しては同じ効果を持たなければなりません。

後ろ2つの要件内の「インクリメント可能」とは、`i`が前置/後置`++`の定義域にある場合のことです。これは、`end`イテレータや符号付き整数型の正の最大値のようにインクリメントできない（未定義動作となる）値を除くための条件です。

重要なのはおそらく2つ目の要件で、これによってインクリメント操作が何らかの進行をするものであることを定義しています。イテレータの場合はそのままの意味ですが、整数型の場合は+1した値になると言う意味になります。

### `incrementable`

`std::incrementable`は、`weakly_incrementable`な型が`regular`であり、インクリメント操作が副作用を持たないことを表すコンセプトです。

```cpp
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

このコンセプトには意味論要件が指定されています

まず

- `std::weakly_incrementable`の前置/後置`++`は等しさを保持する

次に、型`I`のインクリメント可能なオブジェクト`a, b`について

- `bool(a == b)`が`true`ならば`bool(a++ == b)`
- `bool(a == b)`が`true`ならば`bool(((void)a++, a) == ++b)`

ここでの「インクリメント可能」は`std::weakly_incrementable`の時と同じ意味で、`std::incrementable`とは異なる意味です。

これらの意味論要件によって、`std::weakly_incrementable`では求められていなかったインクリメント操作の副作用禁止が要求されるようになっています。特に、後の2つの条件はイテレータに対するマルチパス保証を表しています。

`incrementable`なイテレータは同じ範囲を複数のイテレータによって同時に走査することができ、それが保証されます。

### `input_or_output_iterator`

`std::input_or_output_iterator`はC++におけるイテレータの最小の要件を表すコンセプトです。

```cpp
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

```cpp
template<class S, class I>
concept sentinel_for =
  semiregular<S> &&
  input_or_output_iterator<I> &&
  weakly-equality-comparable-with<S, I>;
```

`std::sentinel_for<S, I>`は`S`がイテレータ型`I`に対する番兵型である場合に`true`となります。

番兵とはイテレータ範囲`[begin, end)`の`end`イテレータのことを言い、番兵型とはその型のことを言います。C++17イテレータでは番兵型とイテレータ型は同一であることが前提とされていましたが、C++20ではそうでなくてもよく、このコンセプトはイテレータ型に対する番兵型を定義するものです。

番兵型に対する要求はイテレータ型と比較して非常に弱く、`input_or_output_iterator`であることも求められておらずイテレータ型`I`と`S`の間で等値比較可能であることがほぼ全てです。

このコンセプトには意味論要件が指定されています

型`S, I`のオブジェクトをそれぞれ`s, i`とし、`[i, s)`は範囲を表すとして

- `i == s`が未定義動作を含まない
- `bool(i != s)`が`true`の場合、`i`は間接参照可能であり`[++i, s)`も範囲を表す
- `I, S`は`std::assignable_from<I&, S>`のモデルとなるか、構文的にも満たさない

ここでの`==`の定義域は静的なものではなく実行時に変化しうるものです。例えば、`[i, s)`が有効な範囲であり`i != s`であるときに、`i == oi`となるような別のイテレータ`oi`をインクリメントした後でも`[i, s)`が有効であり続ける必要はなく、`i == s`が有効（未定義とならない）であり続ける必要もありません。このようなことは、イテレータのインクリメントが参照する範囲の状態を変更するような場合に起こり得ます。

2つ目の要件は`i`が`s`に到達していない間は間接参照とインクリメントが有効であることを言っています。これは範囲を走査するイテレータの基本的な性質の一部でもあります。

3つ目の要件は、イテレータオブジェクトに対して番兵オブジェクトを代入できる（それぞれのオブジェクトを`i, s`として、`i = s`が可能である）性質についての要件で、代入が有効でない場合は`std::assignable_from<I&, S> == false`とならなければならないことを指定しています。イテレータ型と番兵型が同じ型ならば番兵を代入することで範囲の最後のイテレータを得るという操作は簡単に実装できますが、イテレータ型と番兵型が異なる場合は必ずしも実装可能ではありません。この要件は、そのような操作が番兵型にとって必須ではないことを言っています。

### `sized_sentinel_for`

`std::sized_sentinel_for`はイテレータ型に対して距離を定義可能な番兵型であることを表すコンセプトです。

```cpp
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

このコンセプトには意味論要件が指定されています

イテレータが型`I`と番兵型`S`のオブジェクト`i, s`とそれによって示される範囲`[i, s)`、`bool(i == s)`が`true`となるために必要な`++i`の適用回数を`N`として

- `N`が`iter_difference_t<I>`型で表現可能である場合、`s - i`は未定義動作を含まず、`N`に等しい
- `-N`が`iter_difference_t<I>`型で表現可能である場合、`i - s`は未定義動作を含まず、`-N`に等しい

イテレータと番兵の間の距離についてを定義するとともに、イテレータと番兵の間の引き算（`operator-`）はそのイテレータと番兵との間の距離を表すことを言っています。

`std::disable_sized_sentinel_for<S, I>`は変数テンプレートで、`std::sized_sentinel_for<S, I> == true`となる時でも上記意味論要件を満たすことができない場合に`std::sized_sentinel_for`を無効化するためのものです。利用する場合は、`S, I`について`true`となるように明示的特殊化を定義します。

### `input_iterator`

`std::input_iterator`は入力イテレータであることを表すコンセプトです。

```cpp
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

```cpp
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

```cpp
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

このコンセプトには意味論要件が指定されています

型`T`の値`t`と`I`の値`i`について

- `*i++ = t`は次の式と等価

```cpp
*i = t;
++i;
```

この要件によって、後置`++`が自身の型のオブジェクト以外のものを返すことや返したイテレータが別の範囲を参照しているなどを禁止しています。インクリメント操作は後置と前置で（戻り値を除いて）意味が変わってはいけません。

単に`output_iterator`であるイテレータには例えば`std::ostream_iterator`があり、`*i`が左辺値参照を返す殆どのイテレータは同時に`output_iterator`でもあります。

### `forward_iterator`

`std::forward_iterator`は前方向イテレータであることを表すコンセプトです。

```cpp
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

例えば、標準ライブラリの全てのコンテナのイテレータは少なくとも`forward_iterator`であり、単に`forward_iterator`であるイテレータに`std::forward_list`のイテレータがあります。

### `bidirectional_iterator`

`std::bidirectional_iterator`は双方向イテレータであることを表すコンセプトです。

```cpp
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

このコンセプトには意味論要件が指定されています

まず

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

```cpp
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

このコンセプトには意味論要件が指定されています

`std::iter_difference_t<I>`を`D`、`D`の値を`n`、`I`の有効なイテレータを`a, b`、`b`は`++a`を`n`回適用すると到達するとして

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

配列や`std::vector`等のイテレータが`random_access_iterator`ですが、単に`random_access_iterator`であるイテレータには`std::deque`があります。

### `contiguous_iterator`

`std::contiguous_iterator`は隣接イテレータであることを表すコンセプトです。

```cpp
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

このコンセプトには意味論要件が指定されています

`a, b`を間接参照可能な型`I`のイテレータ、`c`を間接参照不可能な型`I`のイテレータとして`b`は`a`から、`C`は`b`から到達可能であり、`std::iter_difference_t<I>`を`D`として

- `std::to_­address(a) == std::addressof(*a)`
- `std::to_­address(b) == std::to_­address(a) + D(b - a)`
- `std::to_­address(c) == std::to_­address(a) + D(c - a)`
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

基本的に、C++20のイテレータコンセプトの要求はC++17のイテレータ要件よりも厳しくなっており、C++17イテレータをC++20イテレータとして利用することは難しい一方で、C++20イテレータはC++17イテレータとしても利用することができる場合があります。ただし、細部の要件の非互換（`operator*`の戻り値型が参照型でなくても（*prvalue*でも）良いなど）のためにそのままのイテレータカテゴリ（特に、*forward*以上）のままでは利用できない場合があります。その場合に、C++17コードからは`std::iterator_traits`を通してイテレータの性質が取得されることを利用して、`std::iterator_traits`を介した場合にC++20イテレータがより弱いC++17イテレータとして見えるようにできるようになっています。

その際に重要なのが、`iterator_cateogry`と`iterator_concept`の2種類のメンバ型です。`iterator_cateogry`が以前からそうであるように、この2つのメンバ型はイテレータのタグ型を指定しておくことでそのイテレータ型がどの種類のイテレータなのかを表明するものです。そして、イテレータコンセプトは常に`iterator_concept`を優先して見に行き、`std::iterator_traits`は`iterator_cateogry`しか見に行きません。これによって、C++20コードから利用された時とC++17コードから利用された時で取得されるイテレータタグ型を切り替えることができるわけです。例えばそれはポインタ型に対する`std::iterator_traits`特殊化に見ることができます。

```cpp
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

```cpp
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

```cpp
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

比較関数や`swap`など、イテレータを用いたアルゴリズムではイテレータをデリファレンスして関数を呼び出す、ということがよく行われます。それを制約するためのコンセプトも用意されます。

### `indirectly_unary_invocable`

`std::indirectly_unary_invocable`はイテレータの要素型による単項（1引数）呼び出しが可能であることを表すコンセプトです。

```cpp
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
  f(e);   // 値型による呼び出し

  using CI = std::iter_common_reference_t<I>;
  CI b1 = *i;
  CI b2 = e;

  // 共通の参照型による呼び出し
  f(b1);
  f(b2);
}
```

`indirectly_unary_invocable`な`F, I`では上記の全ての呼び出しが可能です。

さらには、呼び出し方法の違いによって戻り値型が大きく変わらないこと（`common_reference`を持つこと）も求められています。

このような要件を必要とするイテレータアルゴリズムには`std::for_each()`があり、そのRange版ではこのコンセプトが使用されます。

```cpp
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

```cpp
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

このコンセプトは`std::indirectly_unary_invocable`の副作用を禁止する（ことを表明する）バージョンであり、違いは意味論要件のみです。それは、コンセプト定義中の`invocable`が`regular_invocable`に置き換えられていることによって要求されています。それ以外の部分は同一です。

### `indirect_unary_predicate`

`std::indirect_unary_predicate`は、イテレータの要素型による単項述語（*unary predicate*）であることを表すコンセプトです。

```cpp
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

```cpp
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

```cpp
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

```cpp
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

```cpp
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

```cpp
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

## イテレータアルゴリズムに関するコンセプト

### `indirectly_movable`
### `indirectly_movable_storable`
### `indirectly_copyable`
### `indirectly_copyable_storable`
### `indirectly_swappable`
### `indirectly_comparable`
### `permutable`
### `mergeable`
### `sortabl`

## 射影操作の結果型

`std::projected<I, P>`はイテレータ型`I`と射影操作`F`を渡して、イテレータに対して射影を適用した結果を

```cpp
namespace std {

  template<indirectly_readable I, indirectly_regular_unary_invocable<I> Proj>
  struct projected {
    using value_type = remove_cvref_t<indirect_result_t<Proj&, I>>;

    indirect_result_t<Proj&, I> operator*() const;  // 宣言のみ
  };
}
```

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
- C++マルチスレッド一巡り  
  (https://zenn.dev/yohhoy/books/cpp-stdlib-multithreading)
