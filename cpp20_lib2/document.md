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

C++20では、コンセプトと`<ranges>`の導入に伴って、イテレータライブララリ周りも大幅に改修されています。特に明記しない場合、この章で紹介している機能は`<iterator>`ヘッダに配置されます。

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

比較関数や`swap`など、イテレータを用いたアルゴリズムではイテレータをデリファレンスして関数を呼び出す、ということがよく行われます。それを制約するためのコンセプトが用意されます。

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

## カスタマイゼーションオブジェクト

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

```cpp
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

```cpp
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

```cpp
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

```cpp
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

```cpp
template<class In, class Out>
concept indirectly_movable =
  indirectly_readable<In> &&
  indirectly_writable<Out, iter_rvalue_reference_t<In>>;
```

`std::indirectly_movable<In, Out>`は、`indirectly_readable`な`In`から`indirectly_writable`な`Out`へムーブで出力できる場合に`true`となります。`In, Out`のオブジェクトを`in, out`とすると、`*out = std::move(*in)`のような操作が可能であることを表しています。

### `indirectly_movable_storable`

`std::indirectly_movable_storable`は、`indirectly_movable`の操作が中間オブジェクトを介しても可能であることを表すコンセプトです。

```cpp
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

このコンセプトには意味論要件が指定されています

関節参照可能な`In`のオブジェクト`i`について

- 次のように初期化された`obj`はこの直前の`*i`の値と等しい

```cpp
std::iter_value_t<In> obj(std::ranges::iter_move(i));
```

- この時、`std::iter_rvalue_reference_t<In>`が右辺値参照型を示す場合、この後の`*i`の値は有効だが未規定

ムーブという操作がその意味通りの振る舞いをすることを言っています。「有効だが未規定」というのは標準ライブラリ型オブジェクトがムーブされた後の状態を指定する常套句であり、別の値を代入しない限りその操作は未定義となります。

### `indirectly_copyable`

`std::indirectly_copyable`は、イテレータの要素型を別のイテレータへコピーしつつ出力することができることを表すコンセプトです。

```cpp
template<class In, class Out>
concept indirectly_copyable =
  indirectly_readable<In> &&
  indirectly_writable<Out, iter_reference_t<In>>;
```

`std::indirectly_copyable<In, Out>`は、`indirectly_readable`な`In`から`indirectly_writable`な`Out`へコピーして出力できる場合に`true`となります。`In, Out`のオブジェクトを`in, out`とすると、`*out = *in`のような操作が可能であることを表しています。

### `indirectly_copyable_storable`

`std::indirectly_copyable_storable`は、`indirectly_copyable`の操作が中間オブジェクトを介しても可能であることを表すコンセプトです。

```cpp
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

このコンセプトには意味論要件が指定されています

関節参照可能な`In`のオブジェクト`i`について

- 次のように初期化された`obj`はこの直前の`*i`の値と等しい

```cpp
std::iter_value_t<In> obj(std::ranges::iter_move(i));
```

- この時、`std::iter_reference_t<In>`が右辺値参照型を示す場合、この後の`*i`の値は有効だが未規定

これも、コピーがその意味通りの結果を持つことを言っています。

### `indirectly_swappable`

`std::indirectly_swappable`は、2つのイテレータの間でその要素の`swap`が行えることを表すコンセプトです。

```cpp
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

```cpp
template<class I1, class I2, class R, class P1 = identity, class P2 = identity>
concept indirectly_comparable =
  indirect_binary_predicate<R, projected<I1, P1>, projected<I2, P2>>;
```

`std::indirectly_comparable<I1, I2, R>`は、`indirectly_readable`な型`I1`と`I2`の要素型が二項関係`R`によって比較可能である場合に`true`となります。型`I1, I2, R`のオブジェクトを`i1, i2, r`とすると`bool b = r(*i1, *i2)`の様な呼び出しが可能出ることを表しており、この時に要素の引き当てに射影を使用することもできます。

`R`に相当する型としては`std::less<>`や`std::equal_to<>`があり、標準ライブラリでは主にその`range`版が利用されます。

### `permutable`

`std::permutable`は、イテレータ範囲の要素をムーブや`swap`によってin-placeで並べ替えできることを表すコンセプトです

```cpp
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

```cpp
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

```cpp
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

```cpp
template<class I, class R = ranges::less, class P = identity>
concept sortable =
  permutable<I> &&
  indirect_strict_weak_order<R, projected<I, P>>;
```

`std::sortable<I, R, P>`は、`I`が`permutable`であり`R`が`I`の要素型について狭義弱順序関係を示す場合に`true`となります。また、その際に射影`P`を使用します。

これは標準ライブラリにおけるソート可能という要件をコンセプトに表したものです。ソートに伴う操作のためには`permutable`が必要であり、比較は狭義の弱順序によっていなければなりません。

これは`std::sort()`を行うための最小の要求であり、実際に`std::ranges::sort`で使用されています。

```cpp
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

```cpp
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

```cpp
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

```cpp
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

```cpp
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

この2つはこれまでの`std::next()`とほぼ同様の振る舞いをします。`n`には負数を渡しても意図通りになりますが、この関数の意味的には常に正の値を渡すべきで、イテレータの後退をしたい場合は次の`std::ranges::prev()`を使用した方が良いでしょう。

```cpp
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

```cpp
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

```cpp
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

```cpp
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

```cpp
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

これは、`std::ranges`にあるリファインされた関数を呼び出すつもりのところで意図せずに古い同名関数を呼び出さないようにするための仕組みです。

この性質は通常の関数定義では実現不可能であり、通常これらの関数は関数オブジェクトとして実装されます。

## 番兵型

C++20では終端イテレータが番兵としてイテレータとは別に扱われるようになっています。それに伴って、汎用的な2つの番兵型が標準に用意されます。

### `default_sentinel`

`std::default_sentinel`は、汎用的に使用可能な番兵です。

```cpp
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
    auto d = std::default_sentinel - it;

    ...
  }
}
```

標準ライブラリ内でも、後述する`std::counted_iterator`などで使用されています。

### `unreachable_sentinel`

`std::unreachable_sentinel`は、イテレータの終端チェックをスキップするために使用できる特殊な番兵型です。

使用可能なところは非常に限られおり、イテレータを用いた検索など範囲を操作する際に、終端チェックとは別の条件によって、終端に到達する前に範囲の走査が終了することが確実にわかっている場合に使用できます。

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

```cpp
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

```cpp
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

bool is_prime(int n) {
  return /* 素数判定処理 */
}

int main() {
  // 何かしらの範囲オブジェクトとする
  std::ranges::range auto seq = {1, 3, 5, 7, 9, 11, 13, 15};

  // seqの先頭から4要素を参照するイテレータを得る
  std::counted_iterator it{std::ranges::begin(seq), 4};

  // 範囲内の数値が全て素数かを調べる
  bool b = std::ranges::all_of(it, std::default_sentinel, is_prime);
  // b == true
}
```

`std::counted_iterator`に対する番兵は`std::default_sentinel`であるため、終端イテレータを計算したり取得する必要がありません。元の範囲から部分範囲を得たい場合に、イテレータを取得してコピーして足して・・・のようなことをやるよりも簡易に同じことを達成できます。

```cpp
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

# コンテナ

ここでは、標準ライブラリのうちコンテナ関連のC++20での追加や変更を見ていきます。

## 連想コンテナ関連

連想コンテナとは次のものです

- `std::set`
- `std::map`
- `std::multiset`
- `std::multimap`
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

この2つ目のテンプレート`.find()`が透過的検索を行うための関数で、連想コンテナのキー型（`key_type`）と異なる型のオブジェクトをとって検索を行うことができます。これによって、そのようなことをしたい場合に`key_type`に一度変換してから渡す必要性がなくなり、利便性が増すとともに不要なオブジェクトの構築を回避することで効率化できます。

おそらくこのことは、`std::string`をキーとする連想コンテナを使用する際によく出会うことになるでしょう。

```cpp
#include <map>

using namespace std::literals;

int main() {
  std::map<std::string, int> map = { {"1", 1}, {"2", 2}, {"3", 3}};

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
  hetero_map<std::string, int> map = { {"1", 1}, {"2", 2}, {"3", 3}};

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

```cpp
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
  std::array a5 = std::to_array<std::array<int, 1>>({{1}, {2}});
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
  std::signed_integral   auto l1 = std::size(vec);
  // ssize()は符号付き整数型で取得
  std::unsigned_integral auto l2 = std::ssize(vec);
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

# アルゴリズム

`<algorithm>`と`<numeric>`にあるイテレータ/数値アルゴリズムにいくつか新顔が追加されます。

## `shift_left/shift_right`

`std::shift_left/std::shift_right`アルゴリズムは、範囲の要素を左/右に指定した数だけシフトさせる（ずらす）ものです。

```cpp
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

## `lexicographical_compare_three_way`

`std::lexicographical_compare_three_way`は2つの範囲を辞書式順序（*lexicographical order*）によって三方比較するアルゴリズムです。

```cpp
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

```cpp
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

中点を求めるという単純な処理はしかし、数値型のハードウェア実装の都合などによってかなり奥が深い処理です。CppCon2019では、`std::midpoint()`の紹介と解説で1時間費やした発表が行われています（検索するとYotubeで見つかるでしょう）。

## `lerp`

`std::lerp()`は、2つの浮動小数点数値間の値の線形補完を行う関数です。これは、`<cmath>`に配置されています。

```cpp
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

`std::bind_front(f, boud_args...)`の戻り値は、左辺値で呼ばれると渡された`f`を左辺値として、右辺値で呼ばれると`f`を右辺値として呼び出し、自信の`const`をその呼び出しにまで伝播します。この性質によって、`const`性と値カテゴリの伝播が自動化され適切に解決されます。

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

# 文字列

## `starts_with/ends_with`

`.starts_with()`は文字列の前方が指定された文字列と一致しているかを調べる関数です。これは`std::string`と`std::string_view`（他の文字型/アロケータ型特殊化も含めて）の両方で利用可能です。`.ends_with()`はその逆に、文字列の前方が指定された文字列と一致しているかを調べます。

```cpp
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

`std::string`の`.reserve()`は指定された数の文字を追加のメモリ確保無しで保持できるように内部キャパシティ（`.capacity()`、現在確保しているメモリ量）を増大させる（すなわち、指定された分のメモリを確保する）関数です。この関数には数値を指定しますが、C++17まではその数値が現在のキャパシティよりも小さい場合に確保しているメモリ量を縮小することができていました（実装定義だったようです）。

`std::vector`の`.reserve()`は縮小できなかったこともあり、この`std::string`独特の振る舞いは気づきづらい罠であり、ジェネリックなコード記述の妨げになっていました。

C++20ではそれが正され、`std::string`の`.reserve()`は指定された量が現在のキャパシティより小さい場合は何もしなくなりました。

```cpp
template<typename T>
  requires requires(T& t) {
    t.reserve(1ull);
  }
auto read(T& buf) {
  constexpr std::size_t min_len = ...;

  // メモリ量域を確保
  // tのキャパシティをmin_len以上にする
  // キャパシティがすでにmin_len以上なら何もしない
  buf.reserve(min_len);

  // 何かデータの読み込み
  // 読み込み中に追加のメモリ確保を回避する
  read_to(std::back_inserter(buf));

  ...
}
```

ファイルやネットワークのI/Oなどでこのようなことを行いたいことは非常に良くあります。ある一定の量まではメモリ確保を起こさないようにするために`.reserve()`を使用しますが、この入力のバッファが使いまわされている場合など、既に確保されている可能性もあり、その場合は何もしないことが期待されます。C++20からはその期待に沿う振る舞いをするようになり、`std::string`オブジェクトの現在のキャパシティを気にせずに使用予定メモリ領域の予約を行えます。

## `char8_t/char`文字列の相互変換

C++20で`char8_t`が導入され型で区別されたUTF-8文字列を扱えるようになりましたが、文字エンコーディングの変換に関してはほぼサポートがありません。それどころか、`std::codecvt`はエンコーディング種別に関わらず非推奨とされたため（セキュリティ上の問題とのこと）、C++は文字エンコーディングの変換能力をほとんど失っています。

ほとんどと書いたのはCのライブラリにはその能力があり、C++でも利用可能だからです。C++20（C23）でも、`char8_t`（UTF-8）文字列とそれ以外との相互変換のための関数が追加されています。

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
  char8_t res{};
  std::mbstate_t ms{}

  auto r = std::mbrtoc8(&res, cstr.data(), 4, &ms);

  assert(r < std::size_t(-3));

  std::cout << r << " : ";
  print(res);
}
```

後述しますが、C++20ではユニコード文字型の出力機能も失われているため、この結果を標準の範囲内で視覚的に直接確認する方法がありません。そのためここでは、`print()`を文字表現を保ったまま`char8_t`文字を標準出力に出力する関数とします（標準出力はユニコード文字を表示可能とします）

```{style=planetext}
3 : あ
```

`std::mbrtoc8()`の戻り値`r`はエラー報告を兼ねており、次のような値と意味を持ちます

- `0` : 変換結果はナル文字（`u8'\0'`）
- `size_t(-3)` : 文字が複数のコード単位から構成されている（サロゲートペアなど）
    - 何も処理されない
    - 2022年現在はこの結果が返ることはないはず
- `size_t(-2)` : `n`バイトを読み込んでも、UTF-8の1文字として完成しない（正しくない）
    - `dst`には何も書き込まれない
    - `n`に4未満を渡すと返りうる
- `size_t(-1)` : UTF-8エンコーディングとして不正
    - `dst`には何も書き込まれない
    - 例えば`0xFF`の値が含まれている場合など
- それ以外 : 変換は正常完了。値は入力文字列（`src`）の先頭から変換のために消費した（読み込んだ）バイト数




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

## `remove_cvref`
## `type_identity`
## `is_nothrow_convertible`
## `is_bounded_array`

## `unwrap_reference`

`std::unwrap_reference`は`std::reference_wrapper<T>`から`T&`を取得するための型特性です。

```cpp
#include <functional>

template<typename T>
auto f(T t) {
  using Ref = std::unwrap_reference<T>::type&;

  // 常に、生の参照として取得する
  Ref r = t;

  ...
}
```

同時に、`std::unwrap_reference` + `std::decay`（参照外しや配列からポインタへの変換など）を行う`std::unwrap_ref_decay<T>`も追加されます。

```cpp
#include <functional>
 
template<typename T>
  requires std::constructible_from<std::unwrap_ref_decay_t<T>, int>
auto f() {
  ...
}
```

`std::unwrap_ref_decay_t<U>`の型は、関数テンプレートにおいて`U`あるいは`std::reference_wrapper<T>`の`T`の値を渡したときに推論される型、あるいは、`auto`による変数宣言時に初期化子に同じものを渡して推論される型と同じ型になります。

これらは主に、テンプレートの文脈で`std::reference_wrapper<T>`とそれ以外の型（の参照型）を統一的に扱いたい場合に使用できるでしょう。

これは、標準ライブラリ内の関数（`std::make_pair`など）の規定を簡単にするためにも使用されています。

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
- cppmap(https://cppmap.github.io/ : ライセンスはCC0 パブリックドメイン)
- C++マルチスレッド一巡り  
  (https://zenn.dev/yohhoy/books/cpp-stdlib-multithreading)
- C++の64bit版のhash_combine(https://suzulang.com/cpp-64bit-hash-combine/)