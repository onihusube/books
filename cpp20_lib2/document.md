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

`const_cast`を用いる2つの制約式は、`Out`のオブジェクト`o`の`*o`が*prvalue*を返す場合にのみその上にある2つの制約式と異なった振る舞いをし、`*o`を`const T&&`にキャストした後でも出力が可能かどうかを調べることでプロクシイテレータとそうでないものを判別しています。

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

`new_iter_alg()`はC++20の`forward_iterator`を受け入れます。C++20イテレータは当然問題なく使用できますが、C++17イテレータは定義のされ方によっては`forward_iterator`コンセプトを満たせずにエラーになるかもしれません。コンセプトのないC++17以前はイテレータの定義をしっかりと調べ固定化するのは簡単ではなかったため、これは仕方ないことです。ただし、C++17イテレータが`forward_iterator`コンセプトを満たしているならば`new_iter_alg()`はエラーを起こしません。C++17イテレータに対しての`std::iter_value_t`や`std::iter_difference_t`等のものもC++17以前にイテレータとして使用できていたものに対してはきちんと動作するためです。

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
}
```

逆に、`iterator_category`を定義しないことでC++20イテレータが後方互換性を持たないことを表明することもでき、これも`<ranges>`の`view`型でよく行われています。

### `iterator_traits`の特殊化

`iterator_traits`がイテレータ型について明示的に特殊化されている場合、イテレータコンセプトも`std::iterator_traits`もその特殊化を優先して見に行きます。その後の流れは同じなのですが、この振る舞いによって、`std::iterator_traits`の特殊化はイテレータ型に対する非侵入的なカスタマイゼーションポイントとしての役割が与えられます。

特に、`std::iterator_traits`経由でC++20イテレータの問い合わせが行われる場合にはC++17コードから利用されていることがわかっているため、`std::iterator_traits`を特殊化しておくことで`reference`や`value_type`などを適切に（自動取得よりも正確に）提供することができます。`std::iter_value_t`と`std::iter_difference_t`も`std::iterator_traits`が特殊化されている場合はそちらから取得するようになります（これらの説明の中で「基本的には」と言っていたのはこの点です）。

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
