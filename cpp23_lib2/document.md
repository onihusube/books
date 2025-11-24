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

## `std::stack`/`std::queue`にイテレータペアを取るコンストラクタを追加

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

## コンテナのCTADにおけるアロケータ引数の推論の改善

C++17でクラステンプレートのテンプレート引数の実引数推定（CTAD）が導入され、同時に`polymorphic_allocator`を使用するコンテナ（`std::pmr::vector`など）も導入されました。しかし、これら2つを組み合わせると思わぬエラーが起こる場合がありました。

```cpp
namespace pmr = std::pmr;
pmr::monotonic_buffer_resource mr;
pmr::polymorphic_allocator<int> a = &mr;
pmr::vector<int> pv(a);

// CTADを使用しない構築
auto s1 = std::stack<int, pmr::vector<int>>(pv);       // ok
auto s2 = std::stack<int, pmr::vector<int>>(pv, a);    // ok
auto s3 = std::stack<int, pmr::vector<int>>(pv, &mr);  // ok

// CTADを使用する構築
auto ds1 = std::stack(pv);      // ok 
auto ds2 = std::stack(pv, a);   // ok
auto ds3 = std::stack(pv, &mr); // ng
```

この例の`s1~s3`はCTADを使用していない初期化であり、全て問題の無い例です。しかし、これらの初期化に対応するCTADによる初期化においては`ds3`だけがエラーになります。これは、`&mr`（`monotonic_buffer_resource*`）が`polymorphic_allocator`型ではないことで、CTADでアロケータ型を推論できずエラーになるためです。

`stack`をはじめとするコンテナアダプタのアロケータ引数は推論補助においてCTADに寄与しないようになっています。従って、対応するCTADを使用しない構築の時と同様にコンテナ型からの推論を行うのが望ましいはずです。

```cpp
namespace std {
  template<typename Container, typename Allocator>
  class stack;

  // stackの2引数推論補助
  template<class Container, class Allocator>
  stack(Container, Allocator) -> stack<typename Container::value_type, Container>;
}
```

しかし、この推論補助には制約が規定されており、ここでの`Allocator`はアロケータとして適格な型が渡される必要があり、そうでない場合この推論補助は使用されません。`ds3`においてここに渡されているのは`monotonic_buffer_resource*`であり、アロケータ型ではないためこの推論補助は使用されず、結果としてCTADでアロケータ型を推論できずエラーになってしまいます。

この問題は、他のすべてのコンテナアダプタ（`queue`や`priority_queue`など）でも同様に発生します。

さらに、CTADの一貫性に関するよく似た問題が`std::vector`そのものにも存在しています。

```cpp
namespace pmr = std::pmr;
pmr::monotonic_buffer_resource mr;
pmr::polymorphic_allocator<int> a = &mr;
pmr::vector<int> pv(a);

// CTADによらない構築
auto v1 = std::vector<int, pmr::polymorphic_allocator<int>>(pv);       // ok
auto v2 = std::vector<int, pmr::polymorphic_allocator<int>>(pv, a);    // ok
auto v3 = std::vector<int, pmr::polymorphic_allocator<int>>(pv, &mr);  // ok

// CTADを使用する構築
auto dv1 = std::vector(pv);       // ok
auto dv2 = std::vector(pv, a);    // ok
auto dv3 = std::vector(pv, &mr);  // ng
```

ここで起きていることは先ほどとは少し異なっており、CTADが暗黙に生成される推論補助を利用する経路で問題が起きています。

CTADでは、明示的に定義された推論補助に加えてコンストラクタから生成した推論補助が使用されます。ここではアロケータを受け取るコピーコンストラクタから生成された推論補助が使用されており、

```cpp
namespace std {
  template<typename T, typename Allocator>
  class vector {
    ...

    // アロケータを受け取るコピーコンストラクタ
    vector(const vector<T, Allocator>&, const Allocator&);

    ...
  };
  
  // 生成される推論補助
  template<typename T, typename Allocator>
  vector(const vector<T, Allocator>&, const Allocator&) -> vector<T, Allocator>;
}
```

この推論補助を使用すると、第1引数から`T = int, Allocator = std::polymorphic_allocator<int>`が導出され、第2引数から`Allocator = std::pmr::monotonic_buffer_resource*`が導出されます。同一のテンプレート引数に対して衝突する候補が発生しているので、推論は失敗しコンパイルエラーとなります。

この2つ目の問題は、他のすべてのコンテナでも同様に発生します。

これらの問題ではどちらも、アロケータ引数自体は1引数目のコンテナ型から決定可能であり、2引数目のアロケータ引数は追加の情報をもたらさないためCTADに寄与するべきではありませんでした。C++23ではこの第2アロケータ引数がCTADに寄与しないようにコンテナの仕様が調整されます。

まず1つ目のコンテナアダプタにおける問題については原因が推論補助の制約にあるため、2引数推論補助（`Container`と`Allocator`を取るもの）では`Allocator`引数に対する制約を削除します。これにより、アロケータ型は`Container`型からのみ推論されるようになり、CTADが正しく動作するようになります。

2つ目の`std::vector`等コンテナにおける問題については、2番目のアロケータ引数を`std::type_identity_t`で包むことでアロケータ引数をCTAD推論の対象から外します。これにより、アロケータ引数はCTADに寄与しなくなり、1引数目からのみテンプレート引数が推論されるようになり、CTADが正しく動作するようになります。

```cpp
namespace std {

  template<typename T, typename Allocator>
  class vector {

    // C++20
    vector(const vector<T, Allocator>&, const Allocator&);

    // C++23
    vector(const vector<T, Allocator>&, const type_identity_t<Allocator>&);
  };
  
  // 生成される推論補助
  template<typename T, typename Allocator>
  vector(const vector<T, Allocator>&, const type_identity_t<Allocator>&) -> vector<T, Allocator>;
}
```

CTADにおける推論時には関数テンプレート呼び出し時のテンプレート引数推論と同じことが行われているため、`std::type_identity_t<T>`の様に宣言された引数に対して`T`の値を渡しても、その引数はテンプレート引数`T`の推論に寄与しなくなります。

```cpp
template<typename T>
void f(T);

template<typename T>
void g(std::type_identity<T>::type);

int main() {
  int x = 42;

  f(10);  // ok、T = int
  g(10);  // ng、Tは推論できない
}
```

これは少し難しいですが、`const vector<T, Allocator>&`の引数に`vector`オブジェクトを渡す場合（上記生成される推論補助の第1引数で行われる推論）とは異なります。

このような修正をすべてのコンテナのコピー/ムーブコンストラクタに対応するアロケータ引数付きのコンストラクタに対して行うことで2番目の問題はすべてのコンテナで解決されます。

```cpp
namespace pmr = std::pmr;
pmr::monotonic_buffer_resource mr;
pmr::polymorphic_allocator<int> a = &mr;
pmr::vector<int> pv(a);

auto ds3 = std::stack(pv, &mr);   // ok、C++23
auto dv3 = std::vector(pv, &mr);  // ok、C++23
```

## `std::pair`の転送コンストラクタにおける`{}`初期化子への対応

C++20の`std::pair`では、要素のデフォルトコンストラクタ呼び出しを意図して`{}`を引数に渡すと、静かに一時オブジェクトを生成してそれをコピーしてしまう場合があります。

```cpp
// std::stringとstd::vector<std::string>の一時オブジェクトが作られ、コピーされる
std::pair<std::string, std::vector<std::string>> p("hello", {});
```

この例では、2つの引数どちらも`std::string`と`std::vector<std::string>`の一時オブジェクトを生成し、さらにそれらをコピーして`pair`の各要素を構築しています。

`std::pair`にはいくつものコンストラクタがあり、その中には引数を転送して構築するコンストラクタもあります。これが使用されていればこのような問題は起こらないはずです。

```cpp
namespace std {

  template<typename T1, typename T2>
  struct pair {
    ...

    // 引数をコピーして構築するコンストラクタ
    explicit(...) constexpr pair(const T1& x, const T2& y); // #1

    // 引数を完全転送して構築するコンストラクタ
    template <class U, class V>
    explicit(...) constexpr pair(U&& x, V&& y); // #2

    ...
  };
}
```

このコンストラクタのうち`#2`を使用してほしいわけです。しかし、引数に`{}`を指定していると`{}`からコンストラクタ引数のテンプレートパラメータを推論することができないため`#2`はオーバーロード解決時に候補から外され、代わりに`#1`が選択されます。`#1`はコンストラクタテンプレートではないため`std::pair`のテンプレートパラメータが指定されていれば良く、`{}`の型が分かります。

`#1`のコンストラクタの呼び出しにおいては、コンストラクタ引数の型が`std::pair`のテンプレートパラメータに指定された型と一致しない場合、それぞれ`T1, T2`に暗黙変換されその一時オブジェクトがコンストラクタに渡され、そこからコピーして要素が初期化されることになります。

先ほどの例では、`"hello"`は一時`std::string`オブジェクトに、`{}`は一時`std::vector<std::string>`オブジェクトに変換され、`std::pair`のコンストラクタに渡されます。`#1`のコンストラクタ内部からは2つの引数はどちらも`const &`なので各要素の初期化においてはコピーが行われます。これにより、一時オブジェクトを生成してそこからさらにコピーが両方の引数で起こります。

この場合に`#2`のコンストラクタを選択しようとすると、次のように書く必要があります

```cpp
std::pair<std::string, std::vector<std::string>> p("hello", std::vector<std::string>{});
```

しかしこれはかなり冗長な記述です。

`std::pair`の初期化において`{}`を活用できるようにするために、C++23からは`#2`の転送コンストラクタのテンプレートパラメータに`std::pair`のテンプレートパラメータがデフォルト引数として指定されるようになります。

```cpp
namespace std {

  template<typename T1, typename T2>
  struct pair {
    ...

    // 引数をコピーして構築するコンストラクタ
    explicit(...) constexpr pair(const T1& x, const T2& y); // #1

    // 引数を完全転送して構築するコンストラクタ
    template <class U = T1, class V = T2>
    explicit(...) constexpr pair(U&& x, V&& y); // #2

    ...
  };
}
```

これによって、先ほどの例では`#2`のコンストラクタが選択されるようになり、コンストラクタ引数は完全転送されて`std::pair`内部で各要素のコンストラクタに直接引き渡されるようになります。そのため、`std::pair`のコンストラクタへの引数渡しで一時オブジェクトが生成され、さらにそこからコピーのようなことは起こらなくなります。

```cpp
// 引数は完全転送され、各要素は引数から直接初期化される（#2のコンストラクタが選択される
std::pair<std::string, std::vector<std::string>> p("hello", {});
```

## 透過的な連想コンテナ削除操作の有効化

C++20では、非順序連想コンテナに対して透過的な検索を行うことのできるオーバーロードが追加されました。「透過的」というのは連想コンテナのキーの型と直接比較可能な型については、キー型への変換を必要とせずにキーの比較を行う事が出来ることを指します（このようなオーバーロードをHeterogeneous Overloadとも呼びます）。C++14で順序付き連想コンテナの検索操作のHeterogeneous Overloadが追加されていたので、全ての連想コンテナで透過的な検索がサポートされています。

C++23ではさらに、削除操作についてもHeterogeneous Overloadが追加されます。これにより、キー型と直接比較可能な型を使用して要素の削除を行うことができるようになります。これはすべての連想コンテナ（順序付き・非順序問わず）に対して追加され、次の2つの関数で利用可能になります。

- `.erase()`
- `.extract()`
    - 厳密には削除ではない

`.erase()`を例にすると、キーを受け取るオーバーロードについては次の2種類が提供されるようになります

```cpp
size_type erase(const key_type& x); // C++03

template<class K>
size_type erase(K&& x); // C++23
```

2つ目の関数テンプレートのものがHeterogeneous Overloadで、`key_type`と比較可能な型`K`を受け取ることができ、`x`は変換などを行われずに要素のキーとの直接比較によって処理されます。これによって使いやすさが向上するだけでなく、`key_type`の一時オブジェクトを作る必要がなくなるため効率的になります。

ただし、Heterogeneous Overloadはデフォルトでは使用できず、有効化するためには連想コンテナならば比較クラス`Compare`に、非順序連想コンテナならばそれに加えてハッシュクラス`Hash`に、`is_transparent`メンバ型が定義されていて、`Key`と異なる型との直接比較および一貫したハッシュ生成をサポートしている必要があります。

ここでは、`std::string`をキーとする連想コンテナによって有効する方法を見てみます。

### （順序付き）連想コンテナ

順序付き連想コンテナ（`std::set, std::map`とその`multi`バージョン）の場合、デフォルトの比較クラスがテンプレートパラメータに指定された型の比較しかできないため、`is_transparent`が定義されていないため有効化されません。デフォルトの比較クラス`std::less`等は`void`に対する特殊化に対してのみ`is_transparent`が定義される（比較がテンプレートになる）のでそれを使うか、C++20で追加された`std::ranges`にある同名の比較クラスを使うことで有効化できます。

```cpp
using namespace std::literals;

int main() {
  std::map<std::string, int, std::ranges::less> map = {
    {"1", 1}, {"2", 2}, {"3", 3}
  };

  map.erase("1"sv); // ok
  auto nh = map.extract("3"sv); // ok
}
```

この場合、`std::string_view`から`std::string`への暗黙変換ができないことによって、Heterogeneous Overloadが有効化されていない場合どちらの呼び出しもコンパイルエラーとなります。

### 非順序連想コンテナ

非順序連想コンテナ（`std::unorderd_map, std::unorderd_set`とその`multi`バージョン）の場合、比較クラスは順序付き連想コンテナの場合と同じ方法で対応できますが、ハッシュクラスに関しては自前で用意する必要があります。

```cpp
using namespace std::literals;

// string_viewとしてハッシュ化することで透過的なハッシュ計算を行う
struct string_hash {
  using hash_type = std::hash<std::string_view>;
  using is_transparent = void;  // 👈これが必要

  std::size_t operator()(std::string_view str) const {
    return hash_type{}(str);
  }
};

int main() {
  std::unordered_map<std::string, int, string_hash, std::ranges::equal_to> map = {
    {"1", 1}, {"2", 2}, {"3", 3}
  };

  map.erase("1"sv); // ok
  auto nh = map.extract("3"sv); // ok
}
```

この`string_hash`は`const char*, std::string, std::string_view`の3つの型（+`string_view`に変換できる型）に対してハッシュ計算を提供できます。これによって、`const char*`と`std::string_view`でHeterogeneous Overloadが有効化されます。

`std::string`系の文字列クラスならこのようにするとハッシュの定義は簡単なのですが、他の型の場合はハッシュ共通化が難しい場合もあるかもしれません。そういう場合は自分でハッシュ実装をする必要があります。

## イテレータ

### `move_iterator<T*>`を`random_access_iterator`にする

`std::move_iterator<T*>`のイテレータカテゴリについてC++17以前に議論された際には、パフォーマンス上の理由により`move_iterator<I>`は`I`の性質を継承するようにされました。

```cpp
std::vector<A> v;
std::vector<A> temp = ...;

using MI = std::move_iterator<std::vector<A>::iterator>;

// MIがランダムアクセスイテレータであることで、ここでのアロケーションを一回で済ませられる
v.insert(v.begin() + N, MI(temp.begin()), MI(temp.end()));
```

`vector::insert`では、挿入される要素数が予めわかっていれば`vector`の領域をあらかじめ拡張しておくことでアロケーションの回数を最小にできます。それができない場合、挿入要素数分`push_back()`するのと同じことになります。

C++20にて`<ranges>`追加と共にイテレータライブラリが改修された際、`move_iterator<I>::iterator_concept`は常に`input_iterator`であるようにされており、C++20`move_iterator`は常に`input_iterator`となります。

C++20ではイテレータ間距離（サイズ）を求められるかどうかはイテレータカテゴリから分離され、`sized_sentinel_for`コンセプトによって別に表されます。`move_iterator<I>`は自身のカテゴリとは関係なく、`I`が`sized_sentinel_for<I, I>`であれば`-`によって距離を定数時間で求めることができるため、`move_iterator`が`random_access_iterator`である必要はありません。

すなわち、`std::move_iterator<T*>`はC++17イテレータとしてはランダムアクセスイテレータになりますが、C++20イテレータとしては`input_iterator`にしかなりません。

```cpp
// C++17イテレータとしてのイテレータカテゴリ
static_assert(
  std::same_as<
    std::iterator_traits<std::move_iterator<int*>>::iterator_category,
    std::random_access_iterator_tag
  >
); // ok

// C++20イテレータとしてのイテレータカテゴリ
static_assert(std::random_access_iterator<std::move_iterator<int*>>); // ng、C++20ではinput_iterator
```

C++17イテレータとC++20イテレータではイテレータの性質の定義やその取得経路が変わったことによってイテレータの性質（この場合は特にカテゴリ）の判定結果が変化しているものが他にもあります。しかし、イテレータカテゴリが変わる場合は`move_iterator`以外は全てC++20の方が強くなっており、これはその唯一の例外となっていました。

C++20の`move_iterator`が`input_iterator`にしかならないのは、イテレータ範囲の中を自由に行き来できたとしても、一度`*`で参照してしまえば同じ要素への2回目以降のアクセスは安全では無くなるためです。ただ、この`move_iterator`を全面的に利用する`views::as_rvalue`がその影響を受けて常に`input_range`になってしまうなどの問題もありました。

このため、C++23では`std::move_iterator<I>`は`I`のカテゴリを可能な限り継承するように変更されます（ただし、`contiguous_iterator`にはならない）。これにより、`std::move_iterator<T*>`は再び`random_access_iterator`になります。

```cpp
// C++20イテレータとしてのイテレータカテゴリ
static_assert(std::random_access_iterator<std::move_iterator<int*>>); // ok、C++23から
```

### Poison Pillオーバーロードの削除

Poison Pillオーバーロードとは、カスタマイゼーションポイントオブジェクト（CPO）の実装において使用されるテクニックで、`std`名前空間にある同名の関数を呼び出さないようにするためのものです。

```cpp
namespace std::ranges {

  namespace impl {

    // Poison Pillオーバーロード
    // std::begin()を（名前探索的な意味で）毒殺する
    void begin(auto&) = delete;
    void begin(const auto&) = delete;

    // ranges::begin CPOの実体
    // 制約等は省略
    struct begin_fn {

      // ADLでbegin()を呼び出す
      auto operator()(auto&& range) const {
        return begin(range);  // ここではstd::begin()が見つからない
      }

    };
  }

  inline namespace cpo {
    // ranges::begin CPO
    inline constexpr impl::begin_fn begin;
  }
}
```

この例は`std::ranges::begin`CPOの効果の1つ（ADLによる非メンバ関数の探索）の簡略化された実装例です。`std::ranges::begin`の非メンバ関数探索ではADLによって探索が行われますが、`std::ranges::begin`自体が`std`名前空間で定義されているため非修飾名探索（ADLの1つ前の探索）において`std::begin()`が見つかってしまいます。この関数はC++20以前の古いものでありコンセプトによるチェックなどはなく使用する候補として適切ではないため、`std::ranges::begin`ではこれを呼び出さないようにしています。

このことは標準ライブラリにあるほとんどのCPOに当てはまり、同様のテクニックが多用されています。

一方で、このPoison PillオーバーロードはCPOにアダプトしたい型に対して不要な影響を及ぼしています。

```cpp
struct A {
  friend auto begin(const A&) -> int const*;
  friend auto end(const A&)   -> int const*;
};

struct B {
  friend auto begin(B&) -> int*;
  friend auto end(B&) -> int*;
};
```

この2つの型はどちらも`range`コンセプトを満たすことが期待されます。しかし、`B`と`const A`は`range`ですが、`A`は`range`ではありません。

これは、当初のPoison Pillオーバーロードが担っていたもう1つの役目である右辺値オブジェクトからのイテレータ取得禁止という効果の名残の悪影響によるものです。

当初の`range`コンセプトでは右辺値の`range`は完全に禁止されていましたが、`std::string_view`のようにそのオブジェクトの生存期間とそこから取得できるイテレータの有効性が無関係であるような型は右辺値オブジェクトからイテレータを取得しても問題ないため、そのような型の右辺値も`range`コンセプトを満たすようにしたいという要望が上がりました。

当初のRangeライブラリでは、そのハンドリングのためにPoison Pillオーバーロード（`begin(auto&&) = delete;`）を活用しました。型`A`で左辺値として`range`になれている状態でそれ以上何も準備せずに`begin(A{})`のように呼んだ時、Poison Pillオーバーロード（`begin(auto&&)`のもの）がADLによる候補よりも優先されるため`const`参照を取るオーバーロードだけでは右辺値オブジェクトからイテレータを取得できなくしています。右辺値オブジェクトに対して`begin`を有効にするには、`begin(A&&)`か`begin(A)`のようなオーバーロードを追加することでそれを明示するようにします。

当初のRangeライブラリでは右辺値`range`をこのように非常に高度なテクニックによってオプトインするようにしていましたがこれには問題が多く（あ、あまりにも高度過ぎる）、これは後に`enable_borrowed_range`変数テンプレートによるより明確かつわかりやすいオプトイン方法に置き換えられ、現在に至っています。現在の`std::ranges::begin`をはじめとするCPOでは、`enable_borrowed_range`な型`R`に対して右辺値の入力を左辺値に実体化した上でディスパッチを行う（入力は常に左辺値として扱う）ことで規格の表現と実装を簡素化しています（規格ではこのことを*reified object*という用語で表現しています）。

そのため、現在のPoison Pillオーバーロードには最初に紹介した`std`名前空間の同名関数の毒殺以外の役割はもはやありません。また、その変更によって、`std::ranges::begin`の行うディスパッチでは右辺値を直接扱うことがなくなったため`begin(auto&&)`という宣言ではPoison Pillオーバーロードが意図通りに機能しなくなり、現在の`auto&`と`const auto&`の2つのオーバーロードに置き換えられました。

ここで、先ほどの例に戻ります。

```cpp
struct A {
  friend auto begin(const A&) -> int const*;
  friend auto end(const A&)   -> int const*;
};

struct B {
  friend auto begin(B&) -> int*;
  friend auto end(B&) -> int*;
};
```

`A`のオブジェクトに対する`std::ranges::begin`はADLによって`A`の`begin()`（*Hidden frineds*定義のもの）を見つけてくれるはずで、そこでは2つのPoison Pillオーバーロードを含めたオーバーロード解決が行われます（`begin(auto&)`と`begin(const auto&)`）。前述のように、オーバーロード解決は実際の引数型の値カテゴリに関わらず`A`の左辺値オブジェクト（`A&`）に対して行われ、それに対しては`begin(const A&)`よりも`begin(auto&)`の宣言の方が優先順位が高くなります。そしてそれは`delete`されているため、素の型`A`に対する`range<A>`は`false`になります。しかし、`range<const A>`だとオーバーロード解決は`const A&`にマッチするため、定義されている`begin(const A&)`がPoison Pillオーバーロードよりも優先順位され、`range<const A>`は`true`となります。

この例のようなコードは完全に合法かつ合理的であり、Poison Pillオーバーロードはそれを妨げています。Poison Pillオーバーロードを削除するとこの問題を解消できますが、Poison Pillオーバーロードにはまだ役割（`std`名前空間の同名関数の排除）が残ってます。

しかしよく考えてみると、`std::begin(r)`は`r`がメンバ`begin()`を持っていたらそれを呼び出してくれるので、少なくとも標準ライブラリのものについてそれが呼び出されて困る理由はなく、`borrowed_range`であるか否かはすでに変数テンプレートによって弁別されるため、Poison Pillオーバーロードの有用な役割は実はもうありません。

実のところ、Poison Pillオーバーロードの残った1つの役割が有用なのは`std::ranges::swap`のように、同名のメンバ関数を呼ぶのではなくADLで探しに行くようなCPOにおいてです。`std::swap()`は無制約であり、これを呼び出すと`swap`操作を定義しない任意の型（標準ライブラリ外のものも含めて）に対して`swap(a, b)`の呼び出しができてしまうため`std::swap()`は使用したくないのです。このことは特に、`std::swappable`コンセプトが`std::ranges::swap`を用いて定義されているため、そのコンセプトの有用性に直結します（`std`名前空間のものに対しては無条件で`std::swappable`が`true`になりうる）。

ただこちらも、C++17以降はきちんと制約されていることが規定されているため、`std::swap()`にもPoison Pillオーバーロードは必要ありません。

結局、標準ライブラリで唯一Poison Pillオーバーロードが必要なのは`std::ranges::iter_swap`だけです。これは`std::iter_swap()`について`std::swap()`と同様の問題がありますが、C++20でも`std::iter_swap()`は無制約であるため、ADLで`iter_swap()`を探しに行く際はPoison Pillオーバーロードが必要になります。

これらの問題の対応のため、C++23では`std::ranges::iter_swap`以外のすべてのCPOの定義から、Poison Pillオーバーロードが取り除かれます。

ただし、CPOが自分自身を発見しないためとグローバル名前空間で不必要な探索を行わないようにするために、CPOが非メンバ関数を探索する際はADLによって探索される（非修飾名探索がおこなわれない）ことが確実になるように追記します。この実装は単に、既存のPoison Pillオーバーロードが引数無しの宣言になるだけです。例えば`std::ranges::begin`なら、`void begin() = delete;`のようになります。

```cpp
namespace std::ranges {

  namespace impl {

    // C++20のPoison Pillオーバーロード
    //void begin(auto&) = delete;
    //void begin(const auto&) = delete;

    // C++23で必要なもの
    void begin() = delete;

    struct begin_fn {

      auto operator()(auto&& range) const {
        return begin(range);
      }

    };
  }
}
```

# 文字列・フォーマット

## `std::string`/`std::string_view`

### `.contains()`

C++20でメンバ関数に`.starts_with()`/`.ends_with()`が追加されたのに引き続いて、C++23では`.contains()`が追加されます。これは指定された部分文字列が含まれているかどうかを判定するものです。

```cpp
std::string str = "hello world";

// C++20
if (str.find("world") != std::string::npos) {
  // 含まれている場合の処理
  ...
}

// C++23
if (str.contains("world")) {
  // 含まれている場合の処理
  ...
}
```

`.find()`を使用した部分文字列の包含判定は次のような問題がありました

- 含まれているかを調べているのに`!=`を使用する
    - 肯定的なことを調べているのに否定的に書かなければならない
- 調べているのは文字の位置なのか、含まれているかどうかなのか、含まれていないかどうかなのか、一見して分かりづらい

対して、`.contains()`というメンバ関数は意図が明確で書くときも読むときもこれらの問題は起こらず、直観的に記述することができます。

`std::string`/`std::string_view`共に、`.contains()`には3種類のオーバーロードが用意されます。

```cpp
constexpr bool contains(basic_string_view<charT, traits> str) const noexcept;
constexpr bool contains(charT ch) const noexcept;
constexpr bool contains(const charT* str) const;
```

```cpp
std::string str = "hello world";

bool b1 = str.contains("world"sv);  // string_viewによる部分文字列の包含チェック
bool b2 = str.contains('w');        // 単一文字の包含チェック
bool b3 = str.contains("world");    // 文字列ポインタによる部分文字列の包含チェック
```

### `nullptr`を取るコンストラクタ

`std::string`/`std::string_view`はどちらも、`const char*`を取るコンストラクタが用意されています。これは文字列ポインタを渡すものですが、ポインタを引数に取るので`nullptr`を渡すことが可能です。

```cpp
std::string str1(nullptr);  // 💀 UB
std::string str2(0);        // 💀 UB
std::string str3 = 0;       // 💀 UB
```

このように構築された`std::string`/`std::string_view`オブジェクトは未定義動作を引き起こします。

こんなコードは正気なら書かないと思われるのですが、それでも在野のコードベースにはこのようなコードが少なからず存在しているようです。C++23からは、このような構築をコンパイルエラーとして弾くために、`std::string`/`std::string_view`に`std::nullptr_t`を取るコンストラクタが`delete`で追加されます。

```cpp
namespace std {
  template <class charT,
            class traits = char_traits<charT>,
            class Allocator = allocator<charT> >
  class basic_string {
    ...
    
    basic_string(nullptr_t) = delete; // C++23で追加
    
    ...
  };

  template <class CharT, class Traits = char_traits<CharT>>
  class basic_string_view {
    ...
    
    basic_string_view(nullptr_t) = delete; // C++23で追加
    
    ...
  };
}
```

上記例のようなコードは、このコンストラクタが選択されることでコンパイルエラーになります。

```cpp
// C++23から
std::string str1(nullptr);  // ng
std::string str2(0);        // ng
std::string str3 = 0;       // ng
```
### `.resize_and_overwrite()`

`std::string`をバッファとして使用して、順次到着する文字列をバッファの`std::string`に追記していくような場合を考えます。この場合、`std::string`の長さはどんどん増加していくため、適宜キャパシティの増大とその領域への書き込みが発生します。この時

1. メモリの確保回数を最小にする
2. 増大したメモリを初期化（0埋め）しない
3. キャパシティのチェックを省略する

というパフォーマンスを向上させるために考慮すべき3つの要件をすべて同時に満たす方法が存在していませんでした。

```cpp
void f(std::string& buffer, std::span<std::string_view> new_data) {
  // 毎回append()
  // 追加の度にキャパシティチェック・必要なら増大・確保した領域の0埋め
  // が行われる
  for (auto append_str : new_data) {
    buffer.append(append_str);  
  }
}
```
```cpp
// 配列中の文字列の長さの合計を求める関数とする
auto calc_length(std::span<std::string_view> data) -> std::size_t;

void f(std::string& buffer, std::span<std::string_view> new_data) {
  // reserve()してappend()
  // 追加の度にキャパシティチェックが行われる
  const std::size_t total_len = buffer.size() + calc_length(new_data);
  buffer.resreve(total_len);

  for (auto append_str : new_data) {
    buffer.append(append_str);  
  }
}
```
```cpp
auto calc_length(std::span<std::string_view> data) -> std::size_t;

void f(std::string& buffer, std::span<std::string_view> new_data) {
  // resize()してmemcpy()
  // resize()時に0埋めが行われる
  const std::size_t total_len = buffer.size() + calc_length(new_data);
  buffer.resize(total_len);

  char* dst = buffer.data() + buffer.size();
  std::size_t append_len = 0;
  for (auto append_str : new_data) {
    const std::size_t step = append_str.size();
    memcpy(dst + append_len, append_str.data(), step);

    append_len += step;
  }
}
```

この後ろ2つの方法でも、上記要件の2/3しか満たせていません。パフォーマンスに敏感なコードではこのオーバーヘッドが問題になることがあり、そのようなコードベースでは独自の文字列型に対してこの手の操作をオーバーヘッドなし（上記3つの要件をすべて満たす）で行うためのAPIが用意されていたりもします。

C++23からは、このような操作においてゼロオーバーヘッドを達成するために、`.resize_and_overwrite()`というメンバ関数が追加されます。上記の例はこの関数を用いて次のように書くことができるようになります

```cpp
auto calc_length(std::span<std::string_view> data) -> std::size_t;

void f(std::string& buffer, std::span<std::string_view> new_data) {
  // C++23 resize_and_overwrite()
  // 上記3要件をすべて満たす
  const std::size_t before_len = buffer.size();
  const std::size_t total_len = before_len + calc_length(new_data);

  buffer.resize_and_overwrite(total_len, [=](char* data, std::size_t length) {
    char* dst = data + before_len;
    std::size_t append_len = 0;

    for (auto append_str : new_data) {
      const std::size_t step = append_str.size();
      memcpy(dst + append_len, append_str.data(), step);

      append_len += step;
    }
    
    assert((before_len + append_len) == length); // この場合は一致する

    return length;  // 書き込んだ領域サイズを返す
  });
}
```

`str.resize_and_overwrite(new_length, op)`は次のような処理を行っています

1. `new_length`がキャパシティよりも大きい場合、領域を確保する
    - `str.reserve(new_length)`相当
2. `auto r = op(str.data(), new_length)`を呼び出す
    - 正確には、`std::move(op)(str.data(), new_length)`の様な呼び出しになる
3. `r`を新しい文字列の長さとする
    - この後で、`str.size() == r`となる

これによって、文字列を追加する際に必要な長さのメモリを確保しつつもその領域をゼロ埋めせず、キャパシティのチェックは`.resize_and_overwrite()`の呼び出しにつき一回のみ、という動作が達成されます。したがって、ここまでの例の様な操作において最初の3要件をすべて満たす事ができます。

```cpp
// 疑似コードでの動作例
template <class charT,
         class traits = char_traits<charT>,
         class Allocator = allocator<charT> >
class basic_string {
  charT* data_;
  std::size_t length_;

  ...

  template <class Operation>
  constexpr void resize_and_overwrite(size_type n, Operation op) {
    this->reserve(n);
    auto r = std::move(op)(data_, n);
    lenght_ = static_cast<std::size_t>(r);
  }
};
```

このコードは細部をかなり単純化しているため正確ではないですが、おおむねこのような処理が行われます。

渡す処理の`op`には文字列領域の先頭ポインタ（必要ならメモリ確保後）と`.resize_and_overwrite()`に渡した長さ`n`（第一引数）が渡されます。`op`の中では、サイズ拡大後の`std::string`の持つメモリ領域にほぼ自由にアクセスすることができ、その最大長は`n`になります。また、この領域は確保された後で初期化（0埋め）されていません。

そのため、`op`の中では`std::string`の管理するメモリ領域に直接書き込みを行うことになります。この領域には`.resize_and_overwrite()`呼び出し以前にその`std::string`オブジェクトが保持していた文字列がそのまま配置されているため、上書きしない場合はこの領域を避ける必要があります（上記の例では`before_len`を使用して行っている）。

書き込みをしたら書き込んだ長さを`op`の戻り値として返す必要があります。このとき、長さ`n`の領域全てに書き込みをする必要はなく`n`未満の数を返すことができ、`op`から返された値が新しい文字列長となります。なお、`op`の戻り値型は整数型である必要があり、`0`未満や`n`以上の値を返すと未定義動作になります。

### 効率的な部分文字列の取得

`std::string`の`.substr()`は部分文字列を取得する関数ですが、結果はコピーされて新しい`std::string`オブジェクトとして返されます。

```cpp
// コマンドライン引数の一部を取り出す
auto benchmark = std::string(argv[i]).substr(12);

// prvalueなstringの一部を切り出す
auto name = obs.stringValue().substr(0, 32);
```

これらの処理はどちらも現在はコピーとアロケーションが発生していますが、元の`std::string`オブジェクトが右辺値である場合は元の文字列の保持する領域を再利用することで余計なコピーとアロケーションを回避する実装を取ることができます。

C++23からは、右辺値参照オーバーロードを利用してそのような最適化が有効になります。

現在`.substr()`は`const`修飾されたものだけ定義されていますが、C++23からは`&&, const &`の2つのオーバーロードが定義されるようになります。

```cpp
// C++20まで
constexpr basic_string substr(size_type pos = 0, size_type n = npos) const;

// C++23から
constexpr basic_string substr(size_type pos = 0, size_type n = npos) const &;
constexpr basic_string substr(size_type pos = 0, size_type n = npos) &&;
```

`const &`のオーバーロードは従来と同じ動作をするもので、`&&`オーバーロードが新しく追加されたものです。`&&`オーバーロードでは、元の`std::string`オブジェクトが右辺値であることが分かっているため、その保持する文字列領域を再利用して部分文字列を構築します。

```cpp
auto f() -> std::string;

// C++20 : 一時オブジェクトのstringから部分文字列のstringをコピーして作成
// C++23 : 一時オブジェクトのstringのリソースを再利用して部分文字列を保持するstringオブジェクトを作成
auto str1 = f().substr(...);

// C++20 : 一時オブジェクトのstringから部分文字列のstringをコピーして作成
// C++23 : 一時オブジェクトのstringのリソースを再利用して部分文字列を保持するstringオブジェクトを作成
auto str2 = std::string(...).substr(...);

auto s = std::string(...);

// C++20 : sから部分文字列のstringをコピーして作成、sは変更されない
// C++23 : sのリソースを再利用して部分文字列を保持するstringオブジェクトを作成、sは有効だが未規定な状態となる
auto str3 = std::move(s).substr(...);
```

どれもC++20からの動作変更にはなりますが、`str3`の場合を除いて観測可能ではなかったので破壊的な変更ではありません。しかし、`str3`の場合の`s`の状態は破壊的変更となり、以前は変更されなかったのがムーブされることでムーブ後状態（有効だが未規定）となります。

とはいえ、C++20でこのように記述するメリットはないためこう書かれることはあまりなく、書いたとしても明示的に`move`しているため`s`の値にはもはや関心が無いことを理解した上でコンパイラにそれを伝えているはずなので、この提案の変更によってその意図した振る舞いが得られることになります。

また、部分文字列からの構築を行うコンストラクタにも同様に右辺値オーバーロードが追加されます。

```cpp
constexpr basic_string(basic_string&& str, size_type pos, const Allocator& a = Allocator());
constexpr basic_string(basic_string&& str, size_type pos, size_type n, const Allocator& a = Allocator());
```

`.substr()`はいずれもこのタイプのコンストラクタを通して実装されるため、この2つは部分文字列からの構築という操作の異なる表記法になります。

```cpp
auto f() -> std::string;

// この2つは同じ意味の操作
std::string str1{f(), 5, 10};
auto str2 = f().substr(5, 10);
```

ただし、コンストラクタの場合はアロケータを指定することができ、デフォルト構築できないようなカスタムアロケータを使用している場合により有用です。追加されたこのコンストラクタ（及び`.str() &&`）はどちらも、右辺値`string`の持つアロケータと渡されたアロケータ`a`が等しい場合（`str.get_allocator() == a`）にのみそのメモリの再利用が行われます。

### `string_view`と`span`がトリビアルコピー可能であることを保証

`std::span`と`std::string_view`は想定される定義及び特殊メンバ関数の規定からしてトリビアルコピー可能（*trivially copyable*）であることが期待されます。しかし、標準はその実装について何も規定しておらず、トリビアルコピー可能であるかどうかについても触れられていません。

`std::span`と`std::string_view`はどちらも次のような特徴があります。

- コピーコンストラクタは`default`定義
- コピー代入演算子は`default`定義
- デストラクタは`default`定義
- 生ポインタと`std::size_t`によるサイズの2つのメンバ変数を持つ型として説明される
- 多くのメンバは`constexpr`であり、ともに*trivially destructible*な型（これはC++17のトリビアル型の要件）

この様に共通する性質がありその実装もほぼ同じで、必然的にトリビアルコピー可能であるはずです。実際、clangとGCCの実装はトリビアルコピー可能となっています。

C++23からは、明示的にトリビアルコピー可能であることが規定され、保証されるようになります。

```cpp
// C++23から、どちらも必ずパスする
static_assert(std::is_trivially_copyable_v<std::string_view>);  // 文字型によらない
static_assert(std::is_trivially_copyable_v<std::span<int>>);    // 要素型によらない
```

## `std::format`/`std::print`

C++23では`range`の出力対応のほかに、追加でいくつかの標準の型が`std::format`/`std::print`対応しています（`range`の対応に関しては、「C++23 ranges」をご覧ください）。

### `pair`/`tuple`の対応

`range`の対応において連想コンテナの出力には再帰的に`pair`のフォーマットが利用されており、`pair`と`tuple`のフォーマットもC++23で追加されています。

```cpp
int main() {
  std::pair pair{"C++", 23};
  std::tuple tuple{"C++", 23, 3.14, std::vector{1, 3, 5}};

  std::println("{}", pair);
  std::println("{}", tuple);
}
```
```{style=planetext}
("C++", 23)
("C++", 23, 3.14, [1, 3, 5])
```

`std::format`/`std::print`は下回りの機構（`std::formatter`周り）が共通しているので、ここでは`std::print`を主に使用します。

`pair`/`tuple`はそれぞれに個別の`std::fromatter`が用意されているのではなく、共通のものを使用しています。そのため、そのフォーマット仕様も共通しています。

`pair`/`tuple`のフォーマットでは基本的に`()`で囲まれて出力され、要素型のフォーマットは再帰的にそれぞれの要素型のフォーマッタが使用されて出力されます。したがって、`pair`/`tuple`の全ての要素がフォーマット可能でないと出力できません（コンパイルエラーになる）。

#### フォーマットオプション

指定可能なオプションの全体像は次のようになっています

```
{ index : fill align width tuple-type }
```

`index`から`width`までは他の型と共通したオプションです。また、全てのオプションは省略可能です。

```cpp
int main() {
  std::pair pair{"C++", 23};

  std::println("{:>20}", pair);
  std::println("{:^20}", pair);
  std::println("{:<20}", pair);
  std::println("{:*^20}", pair);
}
```
```{style=planetext}
         ("C++", 23)
    ("C++", 23)     
("C++", 23)         
****("C++", 23)*****
```

`pair`/`tuple`固有なのは最後の`tuple-type`オプションで、次の2つのどちらかが有効です

- `m`: 囲み文字を失くし、区切り文字を`: `にして出力
    - `pair`もしくは、2要素の`tuple`限定
- `n`: 囲み文字を省略する

```cpp
int main() {
  std::pair pair{"C++", 23};
  std::tuple tuple{"C++", 23, 3.14, std::vector{1, 3, 5}};

  std::println("{:n}", pair);
  std::println("{:m}", tuple);
}
```
```{style=planetext}
"C++": 23
"C++", 23, 3.14, [1, 3, 5]
```

`m`オプションは連想コンテナをフォーマットする際のデフォルトでもあります。

#### `std::formatter`の特殊なメンバ関数

`pair`/`tuple`の`std::formatter`には他の型には無い特殊なメンバ関数が2つ追加されています

```cpp{style=cppstddecl}
namespace std {
  template<class charT, formattable<charT>... Ts>
  struct formatter<pair-or-tuple<Ts...>, charT> {
    ...
  public:
    constexpr void set_separator(basic_string_view<charT> sep) noexcept;
    constexpr void set_brackets(basic_string_view<charT> opening,
                                basic_string_view<charT> closing) noexcept;

    ...
  };

  template<class... Ts>
    constexpr bool enable_nonlocking_formatter_optimization<pair-or-tuple<Ts...>> =
      (enable_nonlocking_formatter_optimization<Ts> && ...);
}
```

- `.set_separator()`: 各要素の区切り文字を指定する
- `.set_brackets()`: 囲み文字を指定する

どちらの関数も直接呼び出したところで通常のフォーマットに影響を与えることはできませんが、自作の型のカスタムフォーマッタを実装する際にこのフォーマッタを再利用して、これらの関数を呼び出すことで区切り文字や囲み文字の変更を簡単に指定できます。

```cpp
struct MyType {
  int n;
  std::string_view str;
};

template<>
struct std::formatter<MyType, char> : public std::formatter<std::tuple<int, std::string_view>, char> {
  template<class ParseContext>
  constexpr auto parse(ParseContext& ctx) {
    // デフォルトの区切り文字と囲み文字を変更する
    this->set_separator(" -> ");
    this->set_brackets("{", "}");

    // tupleのフォーマッタに委譲
    return std::formatter<std::tuple<int, std::string_view>, char>::parse(ctx);
  }

  template <class FormatContext>
  constexpr auto format(const MyType& mt, FormatContext& ctx) const {
    // tupleのフォーマッタに委譲
    return std::formatter<std::tuple<int, std::string_view>, char>::format(
      std::make_tuple(mt.n, mt.str), ctx);
  }
};

static_assert(std::formattable<MyType, char>);  // formattableでテストしたほうがデバッグしやすい（かも

int main() {
  MyType m{.n = 0, .str = "zero"};

  std::println("{}", m);
}
```
```{style=planetext}
{0 -> "zero"}
```

このような集成体型のフォーマット対応においては`pair`/`tuple`のフォーマッタを再利用することで比較的楽にフォーマット対応させることができ、これらの関数はその際に使用できます。

#### パラメータパックのフォーマット

`tuple`のフォーマットを利用することで、パラメータパック（あるいは可変長引数）のフォーマットを簡単に行うことができます。

```cpp
void print_pack(auto&&... args) {
  std::print("{}", std::tie(args...));
}

int main() {
  print_pack(1, "str", 3.1415, 'c', true);
}
```
```{style=planetext}
(1, "str\u{0}", 3.1415, 'c', true) 
```

パラメータパックを愚直にフォーマットしようとするとその要素数に合わせてフォーマット文字列を生成しなくてはならないため、`std::tie()`を使用して`tuple`のフォーマットを利用することで簡単にフォーマットできるようになります。

### `std::thread::id`

非常に地味なところですが、`std::thread::id`のフォーマット対応もされています。

```cpp
int main() {
  std::println("{}", std::this_thread::get_id());
}
```
```{style=planetext}
137625994856256
```

使用可能なオプションは最低限で、独自のオプションはありません。

```
{ index : fill align width }
```

### `basic_format_string`を使用可能にする

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2508r1.html

`std::format()`の引数型は`std::basic-format-string<charT, Args...>`というフォーマット文字列のコンパイル時チェックを行うための説明専用の型です。これは標準ライブラリ内部でのみ使用することを意図したものでユーザーは利用できないのですが、これをユーザーも利用したい場合があります。

例えば、自前のログ出力ラッパ内で`std::format()`を使用してログメッセージをフォーマットしたい場合などです。

```cpp
template <typename... Args>
void log(std::format_string<Args...> s, Args&&... args) {
  // logging_enabledによってログを制御する
  if (logging_enabled) {
    log_raw(std::format(s, std::forward<Args>(args)...));
  }
}
```

C++20まではこの`std::format_string<Args...>`のような型が利用できなかったため、普通の文字列で受け取って実行時フォーマットを行う、標準ライブラリの内部実装を使用する、などの方法しかありませんでした。

C++23からはこの`std::format_string<Args...>`などのコンパイル時チェック付のフォーマット文字列型がユーザーが利用可能な形で定義されるようになります。

```cpp{style=cppstddecl}
namespace std {
  template<class charT, class... Args>
  struct basic_format_string {
  private:
    basic_string_view<charT> str; // 説明専用メンバ変数

  public:
    template<class T>
    consteval basic_format_string(const T& s);

    constexpr basic_string_view<charT> get() const noexcept { return str; }
  };

  // 文字型特化エイリアス
  template<class... Args>
  using format_string = basic_format_string<char, type_identity_t<Args>...>;
  
  template<class... Args>
  using wformat_string = basic_format_string<wchar_t, type_identity_t<Args>...>;
}
```

先ほどのログ出力ラッパのサンプルコードはそのまま動作するようになります。

`basic_format_string`の`consteval`コンストラクタにおいてフォーマット文字列のコンパイル時チェックが行われているため、これを使用して受け取った文字列に対してはフォーマット文字列のコンパイル時チェックが行われています。

### デバッグオプション（`:?`）

文字・文字列のフォーマットオプションとして、`?`が利用できるようになります。これは、デバッグ用のログ出力時などに有用なもので、文字/文字列をエスケープして出力するものです。

```cpp
int main() {
  std::println("{:?}, {:?}", "h\tllo", '\n');
}
```
```{style=planetext}
"h\tllo", '\n'
```

`?`は文字・文字列型の`type`オプションの一種であるため、他のオプションと組み合わせる場合は一番最後に指定します。

```cpp
int main() {
  std::println("{:*^10?}", '\n');
}
```
```{style=planetext}
***'\n'***
```

`?`オプションのフォーマットでは、その文字列をC++コード上で再現可能な文字列リテラルとして有効な文字列を出力します。`?`で出力される文字列はそれをコピペしてC++ソースコードに貼り付けると、文字列リテラルとして元の文字列を得ることができます。このため、特にエスケープシーケンスの扱いが異なります。

```cpp
int main() {
  std::println("{0:}\n{0:?}", "h\tllo");
  std::println("{0:}\n{0:?}", "\n");
  std::println("{0:}\n{0:?}", '\'');
}
```
```{style=planetext}
h	llo
"h\tllo"


"\n"
'
'\''
```

ユニコード文字が含まれている場合も、ユニバーサル文字名にエスケープされます。

```cpp
int main() {
  std::println("{0:}\n{0:?}", "🤷🏻‍♂️");
  std::println("{0:}\n{0:?}", "\u0301");
  std::println("{0:}\n{0:?}", "e\u0301\u0323");
}
```
```{style=planetext}
🤷🏻‍♂️
"🤷🏻\u{200d}♂️"
́
"\u{301}"
ẹ́
"ẹ́"
```

ただし、全ての文字がユニバーサル文字名に置き換えられるわけではありません。この規則は少し複雑ですが、基本的には単独で印字不可能な文字や他の文字に結合して作用する文字などがユニバーサル文字名に置き換えられるようになっています。

`std::formatter`の文字/文字列型に対する特殊化にはこのオプションの指定を制御するために`.set_debug_format()`メンバ関数が追加されます。`.set_debug_format()`は呼び出す（引数無し戻り値なし）とフォーマッタの状態を`?`オプションが指定されたものとして更新します。直接使用することはあまりないですが、`pair`/`tuple`のフォーマッタの`.set_separator()`等と同様にカスタムのフォーマッタを定義する際に利用できます。

## iostream

### `basic_ostream`の`volatile`ポインタ出力のバグ修正

C++20まで、標準出力ストリームを使用して`volatile`ポインタを出力すると、予期しない値が出力されていました。

```cpp
int main() {
           int* p0 = reinterpret_cast<         int*>(0xdeadbeef);
  volatile int* p1 = reinterpret_cast<volatile int*>(0xdeadbeef);

  std::cout << p0 << std::endl;
  std::cout << p1 << std::endl;
}
```
```{style=planetext}
0xdeadbeef
1
```

標準出力ストリーム（`basic_ostream`）に対するストリーム出力演算子（`<<`）では、`const void*`（非`volatile`ポインタ）の出力を行うオーバーロードはあり、普通のポインタはこれを使用してアドレスが出力されます。しかし、CV修飾が合わないため`volatile`ポインタはこのオーバーロードを使用できず、ポインタ->`bool`の変換を介して`bool`値として出力されます。このため、`volatile`ポインタを出力すると`1`になります。

C++23からは`const volatile void*`に対するストリーム出力演算子のオーバーロードが追加され、`volatile`ポインタもアドレスとして出力されるようになります。

```cpp{style=cppstddecl}
namespace std {
  template<class charT, class traits = char_traits<charT>>
  class basic_ostream : virtual public basic_ios<charT, traits> {
  public:
    ...

    // C++20まで、const volatile void*の出力時にはこれが使用されていた
    basic_ostream& operator<<(bool n);
    
    ...

    // C++23から追加されるconst volatile void*の出力オーバーロード
    basic_ostream& operator<<(const volatile void* val);  // 👈

    // 既存のポインタ関連の出力オーバーロード
    basic_ostream& operator<<(const void* p);
    basic_ostream& operator<<(nullptr_t);
    basic_ostream& operator<<(basic_streambuf<char_type, traits>* sb);

    ...
  };
}
```

これにより、先ほどのコードは変更なしでそのまま正しく出力されるようになります。

```{style=planetext}
0xdeadbeef
0xdeadbeef
```

### 排他モードでのファイルオープン（`noreplace`）

C++コードからファイルをオープン（作成）しようとする場合、`<fstream>`のファイルストリームを使用することが多いと思います。ファイルの作成時に考慮すべきことは色々あるのですが、その中に「ファイルを新規で作成するが既に存在する場合は何もしない（作成に失敗してほしい）」というものがあります。一見単純なこの要件はセキュリティに敏感なアプリケーションでは厳密に達成することが難しい場合があります。

```cpp
using fs = std::fielsystem;

auto file_open(fs::path filename) -> std::ofstream {
  // ファイルの存在をチェックしてから
  if (fs::exists(filename)) {
    return {};
  }

  // #1

  // ファイルオープン
  std::ofstream ofs{filename};

  ...

  return ofs;
}
```

ファイル存在をチェックしてからファイルを作成するまでのわずかな間、`#1`の箇所で別の手段でファイルが作成されているとファイルをオープンしようとするときに既存ファイルを開いてしまいます。しかし、コード上からはそれを伺い知ることは困難であり、`fstream`もそれを判別可能な情報を提供しません。

`ofstream`のデフォルトオプション（`ios_base::out`）は既存ファイルを開いて書き込みを行う場合ファイルの内容を上書きしてしまいます（`ios_base::trunc`を含んでいる）。例えば、その作成されたファイルが別のファイルへのシンボリックリンクだったりすると、そのシステムに存在する任意のファイルを上書きできてしまうことになり、その影響は計り知れません。

このような攻撃（あるいはバグ）はTime-of-check to time-of-use（TOCTTOU）と呼ばれており、ユーザーコードだけでは完全に対処することが困難な問題として知られています（ファイルオープンに限った問題ではありません）。

ファイルオープン時のTOCTTOUを防止するためにはOSの提供するファイルシステムAPIのサポートが必要で、POSIXやWindows APIではそれぞれ`O_EXCL`、`CREATE_NEW`というフラグによって提供されています。これらのフラグによるファイルオープンでは、「ファイルの存在チェック」と「ファイルの新規作成」を不可分に（他の処理が挟まる余地なく）行う必要があり、この意味でこのオープンモードの事を排他モードと呼びます。

C++23では`noreplace`という新しいファイルオープンモードオプションによって、排他モードでのファイルオープンが可能になります。

```cpp
using fs = std::fielsystem;

auto file_open(fs::path filename) -> std::ofstream {
  // ファイルの存在をチェックしてからファイルオープン
  std::ofstream ofs{filename, ofs.out | ofs.noreplace};

  ...

  return ofs;
}
```

排他モードでのファイルオープンではファイルの存在チェックが行われ、ファイルが存在しない場合にのみファイルの新規作成が行われファイルオープンに成功します。OSのファイルシステムAPIのサポートによりこの処理の途中に他の処理が挟まらないことが保証されることによって、TOCTTOUが防止されます。

ちなみに、Cの`fopen()`には排他モードでのファイルオープンを行うためのオープンモード`x`がC11で追加されています。

```cpp
auto file_open(const char* filename) -> FILE* {
  // ファイルの存在をチェックしてからファイルオープン
  FILE* fp = fopen(filename, "wx");

  ...

  return fp;
}
```

# functional

## `std::invoke_r()`

C++17以前から、関数呼び出しという操作を規格上で統一的に表現するために`INVOKE`という仮想操作が定義されており、C++17ではそれに応じた呼び出しを行うライブラリ関数である`std::invoke`が追加されました。

また、`INVOKE`した結果を指定した戻り値型`R`に変換する`INVOKE<R>`という操作も定義されており、こちらに対応するものとして`std::invoke<R>`のような形で提案されていましたが、C++17時点では不要であるとしてドロップされました。

しかし、`INVOKE<void>(f, args...)`のような呼び出しは戻り値を明示的に破棄するために使用でき、`std::is_invocable_r`や`std::is_nothrow_invocable_r`は指定した戻り値型で呼び出せるかを調べられるようになっています。さらに、`std::visit`には戻り値型を指定する`std::visit<R>`が用意されています。

これらの有用性は既に示されていたため、これらのものに準じる形で`std::invoke_r`がC++23で追加されます。

```cpp{style=cppstddecl}
namespace std {
  // C++17 invoke()
  template<class F, class... Args>
  constexpr invoke_result_t<F, Args...> invoke(F&& f, Args&&... args)
    noexcept(is_nothrow_invocable_v<F, Args...>);

  // C++23 invoke_r()
  template <class R, class F, class... Args>
  constexpr R invoke_r(F&& f, Args&&... args)
    noexcept(is_nothrow_invocable_r_v<R, F, Args...>);
}
```

`std::invoke_r<R>(f, args...)`は`f`と`args...`を`std::invoke()`した結果を`R`に暗黙変換するものです。特に、`R`が`void`の場合は`static_cast<void>()`されます。

`std::invoke_r`は`std::invoke`と比較して次のような利点があります。

- `void`を指定すると戻り値を破棄できる
- 戻り値型を変換して呼び出しできる
    - 例えば、`T&&`を返す関数を`T`（*prvalue*）を返す関数に変換できる
- 複数の戻り値型を返しうる呼び出しを指定した1つの型を返すように統一できる
    - 例えば、共変戻り値型をアップキャストする

```cpp
[[nodiscard]]
auto f1(int) -> int;

template<typename T>
auto f2(T) -> T&&;

void example() {
  // 戻り値の破棄
  std::invoke_r<void>(f1, 0);

  // 戻り値型の変換
  int pr = std::invoke_r<int>(f2, 0);
}

struct base {
  ...
};

template<typename F, typename... Args>
  requires std::derived_from<std::invoke_result_t<F, Args...>, base>
auto f3(F&& f, Args&&... args) -> base {
  // 共変戻り値型のアップキャスト
  return std::invoke_r<base>(std::forward<F>(f), std::forward<Args>(args)...);
}
```

## `std::move_only_function`

`std::function`にはいくつか問題点が知られており、そのうちの一つにムーブのみが可能な（コピーできない）Callableオブジェクトを格納できないというものがあります。

```cpp
int main() {
  auto lam = [m = std::make_unique<int>(10)](int n) -> int {
    ...
  };

  static_assert(std::is_copy_constructible_v<decltype(lam)> == false);  // ✔

  std::function<int(int)> func{std::move(lam)}; // ng
}
```

これは、`std::function`が格納するものに対してコピー可能であることを求めているためです。

これは`std::function`自体をコピー可能にするために、保持するものに対してもコピー可能であることを要求するためだと思われます。また、`std::function`はC++にムーブの概念が無かった時代に設計された`boost::function`をベースとしており、C++11でムーブセマンティクスとほぼ同時に導入されているため、ムーブについて考慮する時間が無かったのも理由にありそうです。

C++23では、ムーブオンリーなCallableを格納するための汎用関数型として、`std::move_only_function`が導入されます。

```cpp
int main() {
  auto lam = [m = std::make_unique<int>(10)](int n) -> int {
    ...
  };

  static_assert(std::is_copy_constructible_v<decltype(lam)> == false);  // ✔

  std::move_only_function<int(int)> func{std::move(lam)}; // ok
}
```

ただし、`std::move_only_function`自身もムーブのみが可能でコピー不可な型となります。

```cpp
int main() {
  ...

  std::move_only_function<int(int)> func{std::move(lam)};     // ok
  std::move_only_function<int(int)> func2 = func;             // ng
  std::move_only_function<int(int)> func3 = std::move(func);  // ok
}
```

### `const`の伝播と指定

また、`std::function`の別の問題として`cosnt`性を正しく伝播して呼び出しが行えないというものもありました。

```cpp
#include <functional>

struct F {
  auto operator()() -> std::string_view {
    return "operator()";
  }

  auto operator()() const -> std::string_view {
    return "operator() const";
  }
};

int main() {
  F f1{};
  const F& f2 = f;
  
  std::println("{}", f1());  // 非const版が呼ばれる
  std::println("{}", f2());  // const版が呼ばれる

  std::function<std::string_view()> func1{F{}};
  const auto& func2 = func1;

  std::println("{}", func1());  // 非const版が呼ばれる
  std::println("{}", func2());  // 非const版が呼ばれる
}
```
```{style=planetext}
operator()
operator() const
operator()
operator()
```

`std::function`自体の`operator()`は`const`修飾されているものの、内部で保持しているものの呼び出しを行うコンテキストは非`const`であるため、`std::function`は保持するメンバ関数の`const`修飾されたものを呼び出すことができず、その方法も提供されていません。

標準ライブラリの`const`メンバ関数にはスレッドセーフ保証の意味合いもあるため、このことはともすればデータ競合を引き起こす可能性があります。

`std::move_only_function`では保持する関数型の指定に`const`を指定することができ、これによって保持する関数の`const`有無を選択して呼び出すことができ、また`std::move_only_function`の`const`性が呼び出しコンテキスに適用されるようになります。

```cpp
#include <functional>

struct F {
  auto operator()() -> std::string_view {
    return "operator()";
  }

  auto operator()() const -> std::string_view {
    return "operator() const";
  }
};

int main() {
  std::move_only_function<std::string_view()> func1{F{}};
  const auto& func2 = func1;
  std::move_only_function<std::string_view() const> func3{F{}};
  const auto& func4 = func3;

  std::println("{}", func1());  // 非const版が呼ばれる
  std::println("{}", func2());  // ng
  std::println("{}", func3());  // const版が呼ばれる
  std::println("{}", func4());  // const版が呼ばれる
}
```

`func2`の呼び出しがコメントアウトされ改行出力されているとすると

```{style=planetext}
operator()

operator() const
operator() const
```

`std::move_only_function`のオブジェクト状態（`const`/非`const`）と`std::move_only_function`に指定された関数型によって呼び出し可能な関数は次のようになります

|オブジェクト状態＼関数型|`R(Args...)`|`R(Args...) const`|
|---|---|---|
|非`const`|どちらも（非`const`優先）|`const`|
|`const`|`❌`|`const`|

行は`std::move_only_function`オブジェクトが`const`かどうか、列は`std::move_only_function<F>`の`F`に指定されている関数型で、表の要素はその組み合わせで呼び出すことのできる関数の`const`有無を表しています。そして、`❌`は呼び出し時にコンパイルエラーになることを表しています。

すなわち、関数型に`const`と指定したら非`const`関数を呼び出すことはできず、関数型に`const`を指定しない場合に`const`状態から関数呼び出しを行うことができません。

また、関数型に`const`を指定している場合は`std::move_only_function`オブジェクトそのものの`const`状態とは無関係に呼び出しを行うことができ、この時でも関数型で指定されたもの（`const`版）が呼び出されます。

この動作は通常のメンバ関数の`const`修飾とオブジェクトの`const`の関係と一致しています。上記例だと`F`が両方を備えていたから少しわかりづらかったかもしれません。

```cpp
#include <functional>

struct F {
  // 非constオーバーロードを備えない
  auto operator()() const -> std::string_view {
    return "operator() const";
  }
};

int main() {
  F f1{};
  const F& f2 = f1;

  std::println("f1: {}", f1()); // const版が呼ばれる
  std::println("f2: {}", f2()); // const版が呼ばれる

  std::move_only_function<std::string_view() const> func1{F{}};
  const auto& func2 = func1;

  std::println("func1: {}", func1());  // const版が呼ばれる
  std::println("func2: {}", func2());  // const版が呼ばれる
  
  std::move_only_function<std::string_view()> func3{F{}};
  // 構築はできるものの、呼び出し可能な対象はない
  const std::move_only_function<std::string_view()> func4{F{}};

  std::println("func3: {}", func3()); // const版が呼ばれる
  std::println("func4: {}", func4()); // ng
}
```

`func4`の呼び出しがコメントアウトされているとすると

```{style=planetext}
f1: operator() const
f2: operator() const
func1: operator() const
func2: operator() const
func3: operator() const
```

ちなみに`std::function`で先程と同じ表を作るとこうなります

|オブジェクト状態＼関数型|`R(Args...)`|`R(Args...) const`|
|---|---|---|
|非`const`|どちらも（非`const`優先）|`🤷‍♂️`|
|`const`|どちらも（非`const`優先）|`🤷‍♂️`|

`🤷‍♂️`は指定の方法が無い（`std::function<R() const>`はコンパイルエラー）ことを表しています。

このように、`std::move_only_function`は保持する関数の`const`指定及び、呼び出し時の`const`性の正しい伝播をサポートしています。

なお、`volatile`についてはサポートされていません。

### 値カテゴリの伝播と指定

`const`と同種の問題として、`std::function`は関数の参照修飾も正しく扱うことができず、`const`の場合と同様にオブジェクト状態を考慮した呼び出しを行えません。

```cpp
struct F1 {
  auto operator()() & -> std::string_view {
    return "operator() &";
  }
  auto operator()() && -> std::string_view {
    return "operator() &&";
  }
};

struct F2 {
  auto operator()() && -> std::string_view {
    return "operator() &&";
  }
};

int main() {
  F1 f1{};

  std::println("f1 & : {}", f1()); // &版が呼ばれる
  std::println("f1 &&: {}", std::move(f1)()); // &&版が呼ばれる

  std::function<std::string_view()> func1{F1{}};

  std::println("func1 & : {}", func1());  // &版が呼ばれる
  std::println("func1 &&: {}", std::move(func1)());  // &版が呼ばれる


  std::function<std::string_view()> func2{F2{}};  // ng
}
```

`func2`の構築がコメントアウトされているとすると

```{style=planetext}
f1 & : operator() &
f1 &&: operator() &&
func1 & : operator() &
func1 &&: operator() &
```

`std::move_only_function`では`const`の時と同様に、関数型に対して参照修飾を指定することができ、指定された関数型と`move_only_function`オブジェクトの状態によって適切な関数呼び出しが行われます。

```cpp
struct F1 {
  auto operator()() & -> std::string_view {
    return "operator() &";
  }
  auto operator()() && -> std::string_view {
    return "operator() &&";
  }
};

int main() {
  std::move_only_function<std::string_view() &> func1{F1{}};

  std::println("func1 & : {}", func1());  // &版が呼ばれる
  std::println("func1 &&: {}", std::move(func1)());  // ng

  std::move_only_function<std::string_view() &&> func2{F2{}};
  
  std::println("func2 & : {}", func2());  // ng
  std::println("func2 &&: {}", std::move(func2)());  // &&版が呼ばれる
  
  std::move_only_function<std::string_view()> func3{F1{}};
  
  std::println("func3 & : {}", func3());  // &版が呼ばれる
  std::println("func3 &&: {}", std::move(func3)());  // &版が呼ばれる!?
}
```

ngになるケースは改行のみが出力されているとすると

```{style=planetext}
func1 & : operator() &


func2 &&: operator() &&
func3 & : operator() &
func3 &&: operator() &
```

参照修飾を指定する場合は指定された参照修飾に合致する関数のみを呼び出すようになり、呼び出しにおいてはオブジェクトの状態（値カテゴリ）を考慮するようになります。

参照修飾を指定しない場合の挙動は少し直感的ではないかもしれませんが、これもまた通常のCallableオブジェクトの呼び出しを再現しています。

```cpp
struct F1 {
  auto operator()() -> std::string_view {
    return "operator()";
  }
};

int main() {
  F1 f1{};
  
  std::println("f1 &  : {}", f1()); 
  std::println("f1 && : {}", std::move(f1)());  // ok
  
  std::move_only_function<std::string_view()> func{F1{}};
  
  std::println("func & : {}", func());
  std::println("func &&: {}", std::move(func)()); // ok
}
```
```{style=planetext}
f1 &  : operator()
f1 && : operator()
func & : operator()
func &&: operator()
```

オブジェクト状態と関数型の参照修飾の指定による呼び出し可能な関数は次のようになります

|オブジェクト状態＼関数型|`R(Args...)`|`R(Args...) &`|`R(Args...) &&`|
|---|---|---|---|
|`&`|無/`&`|無/`&`|`❌`|
|`&&`|無/`&`|`❌`|無/`&&`|

`❌`は呼び出せない（コンパイルエラーとなる）ことを表し、無/`&`/`&&`は呼び出される`operator()`の種別を表しています。無は参照修飾無しで`operator()`が宣言されていることを表しており、無と`&`/`&&`ありは同時にオーバーロードすることはできませんが、無で定義された`operator()`はどちらの値カテゴリ/参照修飾に対してもマッチし呼び出されます。

### `noexcept`の指定

さらに、`std::move_only_function`では関数の`noexcept`指定もサポートしています。

```cpp
auto f1() -> std::string_view {
  return "f1()";
}

auto f2() noexcept -> std::string_view {
  return "f2() noexcept";
}

int main() {
  std::move_only_function<std::string_view()> func1{f1};
  std::move_only_function<std::string_view() noexcept> func2{f2};
  std::move_only_function<std::string_view()> func3{f2};  // ok
  std::move_only_function<std::string_view() noexcept> func4{f1}; // ng

  std::println("func1 : {}", func1());
  std::println("func2 : {}", func2());
  std::println("func3 : {}", func3());
}
```

`func4`の構築がコメントアウトされているとすると

```{style=planetext}
func1 : f1()
func2 : f2() noexcept
func3 : f2() noexcept
```

指定しない場合は`noexcept`性は考慮されずに呼び出されますが、指定した場合は`noexcept`なもののみが呼び出されます。`noexcept`指定した場合に`noexcept`指定されている関数しか呼び出せない（保持できない）ようになるだけで、指定しない場合に指定されていない関数しか扱えなくなるわけではありません。

`noexcept`なしと`noexcept(false)`、`noexcept`ありと`noexcept(true)`は同じ指定として扱われます。

関数の`noexcept`指定は関数型に適用されるものの関数呼び出し時には考慮されないため、それによるオーバーロードや呼び分けはできません。`noexcept`の指定はあくまで、`move_only_function`の関数呼び出しが例外を投げないことを強く保証（あるいは要求）することができるようにするためのものです。そして、このために`move_only_function`の関数呼び出し時のnullチェックは強い事前条件になっており、`move_only_function`の関数呼び出しは`noexcept`指定によって完全に無例外にすることができます。

`const`/参照修飾と`noexcept`の指定は互いに組み合わせて使用できます。例えば、`move_only_function<R() const & noexcept>`のような指定が可能です。この組み合わせによる呼び出し可能な候補について表を書くこともできなくはないのですが、かなり巨大かつ難解になってしまうのでここでは省略します（書いてみると勉強になるかもしれません）。


### 改善点のまとめ

まとめると、`std::move_only_function`は`std::function`に対して次のような改善がなされたものです

1. ムーブのみが可能なものを扱える
    - `std::unique_ptr`をキャプチャしたラムダのように、コピー不可能な*Callable*オブジェクトを受け入れられる
2. 関数型に`const`/参照修飾や`noexcept`を指定可能
    - `const`性や値カテゴリを正しく伝播できる
3. 呼び出しには強い事前条件が設定される
    - これによって、呼び出し時の`null`チェックが省略される
    - 呼び出し時に`move_only_function`自体が例外送出しない
4. `target_type()`および`target()`を持たない
    - RTTIが不用になる

なお、この1以外の改善を`std::function`に適用したもの（`move_only_function`のコピー対応版）については、`std::copyable_function`としてC++26で導入される予定です。

### 実装について

`std::move_only_function`の実装の基本は`std::function`と同じで、使用しないプライマリテンプレートに対して関数型によって部分特殊化したテンプレートを使用して行われます。

```cpp{style=cppstddecl}
namespace std {

  // プライマリテンプレート
  template<typename...>
  class move_only_function;

  // 関数型で特殊化
  template<typename R, typename... Args>
  class move_only_function<R(Args...)> {
    ...

    R operator()(Args... args) {
      // targe_funcは保持しているCallableとする
      return std::invoke_r<R>(targe_func, std::forward<Args>(args)...);
    }
  };
}
```

この部分特殊化の中で型消去によって任意のCallableを保持し、`operator()`を介して呼び出すようにします。実際は型消去によって呼び出しはさらに別の場所で行われ、実装の自由度としてSOO(Small Object Optimization)が許可されているためもう少し複雑になります。

`std::move_only_function`が`std::function`と異なる最大の点は`const`/参照修飾に加えて`noexcept`の指定までサポートしている点です。これらはテンプレート化できないため別の方法が必要になりそうにも思えますが、実際には力業で解決されます。

```cpp{style=cppstddecl}
namespace std {
  template<typename...>
  class move_only_function;

  template<typename R, typename... Args>
  class move_only_function<R(Args...)>;

  template<typename R, typename... Args>
  class move_only_function<R(Args...) const>;
  
  template<typename R, typename... Args>
  class move_only_function<R(Args...) &>;
  
  template<typename R, typename... Args>
  class move_only_function<R(Args...) noexcept>;

  ...
  
  template<typename R, typename... Args>
  class move_only_function<R(Args...) const && noexcept>;
}
```

このように、部分特殊化の関数型で可能な組み合わせを網羅しておくことですべての指定を受け入れるようにすることができます。

あとは、これらの特殊化それぞれでその指定に合わせた`operator()`と呼び出しを行うようにすることで、`move_only_function`は実装されます。

```cpp{style=cppstddecl}
namespace std {

  ...

  template<typename R, typename... Args>
  class move_only_function<R(Args...) const> {
    ...

    R operator()(Args... args) const {
      // targe_funcは保持しているCallableとする
      return std::invoke_r<R>(std::as_const(targe_func), std::forward<Args>(args)...);
    }
  };
  
  template<typename R, typename... Args>
  class move_only_function<R(Args...) &> {
    ...

    R operator()(Args... args) & {
      return std::invoke_r<R>(targe_func, std::forward<Args>(args)...);
    }
  };
  
  template<typename R, typename... Args>
  class move_only_function<R(Args...) noexcept> {
    ...

    R operator()(Args... args) noexcept {
      return std::invoke_r<R>(targe_func, std::forward<Args>(args)...);
    }
  };

  ...
  
  template<typename R, typename... Args>
  class move_only_function<R(Args...) const && noexcept> {
    ...

    R operator()(Args... args) const && noexcept {
      return std::invoke_r<R>(std::move(std::as_const(targe_func)), std::forward<Args>(args)...);
    }
  };
}
```

実際にはここまでべた書きはされず可能な限り実装は共通化されるでしょうが、基本的にはこのように実装されるはずです。

部分特殊化している関数型に指定されているものはそのままその特殊化の`operator()`に適用されるようにされます。`operator()`ではそれをさらに保持するCallableに適用することで`move_only_function`オブジェクト自体の状態と指定された関数型の両方を考慮した呼び出しを行います。

このようになっているため、`move_only_function`の呼び出しを行った際に保持しているCallableに対してどのオーバーロードが呼び出されるのかは、`move_only_function`のテンプレートパラメータに指定したものに基づいて決定されます。`move_only_function`の関数型に修飾/`noexcept`を指定するとそれに合致するものしか呼び出せなくなるというよりは、呼び出し時にそれを指定して呼び出すようになるという方が素直かもしれません（`noexcept`は少し異なりますが）。

保持するCallableが指定された関数型と修飾および`noexcept`有無によって呼び出し可能かどうかは`move_only_function`の構築・代入の時点で静的に検査され、呼び出し可能な候補が無い場合はそこでコンパイルエラーになります。`move_only_function`に対する呼び出し時点では、その状態（`const`/値カテゴリ）で呼び出し可能な`move_only_function::operator()`があるかどうか、だけが静的に検査されます。

```cpp{style=cppstddecl}
namespace std {

  ...
  
  // 修飾等無しの場合の例
  template<typename R, typename... Args>
  class move_only_function<R(Args...)> {
    ...
    
    template<typename F>
      requires std::is_invocable_r_v<R, std::decay_t<F> , Args...> &&
               std::is_invocable_r_v<R, std::decay_t<F>&, Args...>
    move_only_function(F&& f) {
      ...
    }

    ...

    // 修飾無しメンバ関数は非const状態なら値カテゴリによらず呼び出しが可能
    R operator()(Args... args) {
      // ただし、*thisは常に左辺値として扱う（区別可能な情報がない
      return std::invoke_r<R>(targe_func, std::forward<Args>(args)...);
    }
  };

  ...

  // & noexcept の場合の例
  template<typename R, typename... Args>
  class move_only_function<R(Args...) & noexcept> {
    ...

    template<typename F>
      requires std::is_nothrow_invocable_r_v<R, std::decay_t<F>&, Args...>
    move_only_function(F&& f) {
      ...
    }

    ...

    // targe_funcの呼び出しがnoexceptであることはコンストラクタでチェック済み
    R operator()(Args... args) & noexcept {
      return std::invoke_r<R>(targe_func, std::forward<Args>(args)...);
    }
  };

  ...
}
```

実際には制約はもう少し複雑になります。

# memory

## `std::out_ptr`/`std::inout_ptr`

C言語で書かれたコードやライブラリというものはまだまだ現役で、そのようなコードも比較的簡単に扱うことができるのがC++の強みの一つでもあります。Cのライブラリにおいてもリソース（特にメモリ）を実行時に確保して使用するタイプのライブラリは一般的であり、そのようなライブラリでは多くの場合リソースを確保する関数にポインタを渡して呼び出すとそのポインタに確保されたリソースが返り、リソースの解放も専用の関数を使用するというAPIを提供しています。

```cpp
// 引数に取ったポインタに確保したリソースを返却する
error_num c_api_create_handle(int seed_value, int** p_handle);
// リソース解放を行う
void c_api_delete_handle(int* handle);


void use_c_api() {
  int* resource_handle{};

  // リソースの取得
  auto ec = c_api_create_handle(24, &resource_handle);

  if (ec == C_API_ERROR_CONDITION) {
    // エラーハンドル
    ...
  }

  // リソースの使用
  ...

  // リソースの解放
  c_api_delete_handle(resource_handle);
}
```

C++においてはこのようなパターンはRAIIで管理を自動化するのが定石であり、`std::unique_ptr`はそのためのクラスでもあります。一応現在でも、上記のようなC APIの返すリソースは`std::unique_ptr`で管理することはできます。

```cpp
/// Cライブラリのリソース管理API
error_num c_api_create_handle(int seed_value, int** p_handle);
void c_api_delete_handle(int* handle);

void use_c_api() {
  using deleter = decltype([](int* handle) { c_api_delete_handle(handle); });
  // カスタムデリータをセットしたunique_ptr
  std::unique_ptr<int, deleter> resource{};

  // unique_ptrの管理するポインタのアドレス（ダブルポインタ）を取得する方法が無い
  int* temp_ptr{};
  auto ec = c_api_create_handle(24, &temp_ptr);

  if (ec == C_API_ERROR_CONDITION) {
    // エラーハンドル
    ...
  }

  // 一旦ポインタで受けてからスマートポインタにリソースをセットするしかない
  resource.reset(temp_ptr);

  // リソースの使用
  ...

  // リソースの解放は自動化される
}
```

これでもRAIIによって管理は自動化されているので最初のコードからはだいぶましにはなります。しかし、リソース取得周辺のコードはこのためのハックによって少し汚くなっています。

### `std::out_ptr`

C++23ではこのような目的でスマートポインタを使用するためのユーティリティとして`std::out_ptr`が用意されます。`std::out_ptr`を使用すると上記`std::unique_ptr`を使おうとするコードをかなり簡略化できます。

```cpp
/// Cライブラリのリソース管理API
error_num c_api_create_handle(int seed_value, int** p_handle);
void c_api_delete_handle(int* handle);

void use_c_api() {
  using deleter = decltype([](int* handle) { c_api_delete_handle(handle); });
  // カスタムデリータをセットしたunique_ptr
  std::unique_ptr<int, deleter> resource{};

  // std::out_ptr()を介してスマートポインタを直接渡すことができる
  auto ec = c_api_create_handle(24, std::out_ptr(resource));  // 👈

  if (ec == C_API_ERROR_CONDITION) {
    // エラーハンドル
    ...
  }

  // リソースの使用
  ...

  // リソースの解放は自動
}
```

`std::out_ptr`を使用することで、リソース取得時のコードが生ポインタを直接使う一番最初のコードと同等にシンプルになり、なおかつスマートポインタによってリソース管理が自動化されています。

例では`std::uniue_ptr`を使用していますが、`std::shared_ptr`でもほぼ同様に使用できます。

```cpp
/// Cライブラリのリソース管理API
error_num c_api_create_handle(int seed_value, int** p_handle);
void c_api_delete_handle(int* handle);

void use_c_api() {
  using deleter = decltype([](int* handle) { c_api_delete_handle(handle); });

  std::shared_ptr<int> resource{};

  // shared_ptrの場合はデリータをここで渡す
  auto ec = c_api_create_handle(24, std::out_ptr(resource, deleter{}));  // 👈

  if (ec == C_API_ERROR_CONDITION) {
    // エラーハンドル
    ...
  }

  // リソースの使用
  ...

  // リソースの解放は自動
}
```

カスタムデリータを渡す場所に注意が必要です。`std::shared_ptr`で`std::out_ptr`を使用する場合、カスタムデリータの指定を省略するとコンパイルエラーとなります。これは、内部で`std::shared_ptr`にリソースの所有権を移す際にカスタムデリータのリセットが発生するのを防止するためです。


### `std::inout_ptr()`

さらに、C APIの中には適応的にリソースの再取得を行うタイプのものがあります。このようなAPIでは受け取ったポインタがリソースを保持していればそれを開放し、新しく確保したリソースを再設定します。

```cpp
/// Cライブラリのリソース管理API
error_num c_api_create_handle(int seed_value, int** p_handle);
void c_api_delete_handle(int* handle);
// 必要に応じてリソースの解放と再取得を行うAPI
error_num c_api_re_create_handle(int seed_value, int** p_handle);

void use_c_api() {
  int* resource_handle{};

  // リソースの取得
  auto ec = c_api_create_handle(24, &resource_handle);

  if (ec == C_API_ERROR_CONDITION) {
    // エラーハンドル
    ...
  }

  // リソースの使用
  ...
  
  // リソースの再取得
  auto ec = c_api_re_create_handle(24, &resource_handle);

  if (ec == C_API_ERROR_CONDITION) {
    // エラーハンドル
    ...
  }

  // リソースの解放
  c_api_delete_handle(resource_handle);
}
```

Cの使用感では変わらないのですが、これを`std::out_ptr`で扱おうとすると少し面倒になります。

```cpp
/// Cライブラリのリソース管理API
error_num c_api_create_handle(int seed_value, int** p_handle);
void c_api_delete_handle(int* handle);
error_num c_api_re_create_handle(int seed_value, int** p_handle);

void use_c_api() {
  using deleter = decltype([](int* handle) { c_api_delete_handle(handle); });
  // カスタムデリータをセットしたunique_ptr
  std::unique_ptr<int, deleter> resource{};

  auto ec = c_api_create_handle(24, std::out_ptr(resource));

  if (ec == C_API_ERROR_CONDITION) {
    // エラーハンドル
    ...
  }

  // リソースの使用
  ...

  // このタイプのAPIに渡すためにはまず、スマートポインタの管理から外さなければならない
  auto* temp_handle = resource.release();

  // リソースの再取得
  auto ec = c_api_re_create_handle(24, &temp_handle);

  if (ec == C_API_ERROR_CONDITION) {
    // エラーハンドル
    ...
  }

  // 再度手動でセットする
  resource.reest(temp_handle);

  // リソースの解放は自動
}
```

このタイプのAPIは以前に確保されたリソースのポインタを受け取り、内部で必要に応じてそれを開放し再確保したリソースのポインタをセットします。`std::out_ptr`は渡されたスマートポインタがリソースを保持している場合、それを開放してから新しいリソースをセットしようとしますが、渡したリソースの解放自体はC APIが行うため、`std::out_ptr`で元のリソースを開放してしまうと二重開放になってしまいます。

すなわち、`std::out_ptr`はこのタイプのAPIで正しく使用できないため、この例のようにスマートポインタから所有権ごとポインタを取り出してそれをC APIに渡した後で手動でセットする、というコードを書かなければならなくなっています。

そこで、このタイプのAPIを簡易に扱うために`std::inout_ptr`が用意されます。

```cpp
/// Cライブラリのリソース管理API
error_num c_api_create_handle(int seed_value, int** p_handle);
void c_api_delete_handle(int* handle);
error_num c_api_re_create_handle(int seed_value, int** p_handle);

void use_c_api() {
  using deleter = decltype([](int* handle) { c_api_delete_handle(handle); });
  // カスタムデリータをセットしたunique_ptr
  std::unique_ptr<int, deleter> resource{};

  auto ec = c_api_create_handle(24, std::out_ptr(resource));

  if (ec == C_API_ERROR_CONDITION) {
    // エラーハンドル
    ...
  }

  // リソースの使用
  ...

  // リソースの再取得
  auto ec = c_api_re_create_handle(24, std::inout_ptr(resource)); // 👈

  if (ec == C_API_ERROR_CONDITION) {
    // エラーハンドル
    ...
  }

  // リソースの解放は自動
}
```

`std::inout_ptr`は一つ前の例で行われていた、スマートポインタから所有権ごとポインタを取り出してそれをC APIに渡した後で手動でセットする、というコードを自動化します。これによってこのリソースを再取得するタイプのAPIにおいてスマートポインタを使用するための雑多なコードが取り除かれます。

なお、`std::shared_ptr`が共同所有されているときに所有権を一旦手放す方法が無いため、`std::inout_ptr`では`std::shared_ptr`を使用できません。

### パフォーマンス上のメリット

`std::out_ptr`/`std::inout_ptr`はC APIで使用する際に手動でやる必要があったスマートポインタを初期化するための操作を自動化してくれるもので、コード上でちょっと邪魔なコードを消し去ることができる程度のものに思えるかもしれません。

しかし、そのようなコードをマニュアルで書く場合と比較して使用時のパフォーマンスが向上する場合があります。これは`std::inout_ptr`で特に顕著になります。

さらに、`std::unique_ptr`や`std::shared_ptr`に特化した実装（内部に侵入できる実装）を取ることで最適化することができ、Cべた書きコードとほぼ同等まで近づけることができます。

これらのパフォーマンス向上は

- スマートポインタの内部ポインタを直接使うことによって中間ポインタが必要なくなる
- `.reset()`/`.release()`の個別呼び出しによるオーバーヘッドを排除できる
- 関連するスマートポインタ操作コードがコード上で離れることによる最適化機会の喪失を回避できる

等によるようです。

このような最適化が行われるかどうかは実装定義ではありますが、少なくとも`std::inout_ptr`はナイーブな実装でも同等の手書きコードよりもパフォーマンスが向上するはずです。

### 実装の詳細

`std::out_ptr`/`std::inout_ptr`自体はどちらも単なる関数です。

```cpp{style=cppstddecl}
namespace std {
  template<class Pointer = void, class Smart, class... Args>
  auto out_ptr(Smart& s, Args&&... args);

  template<class Pointer = void, class Smart, class... Args>
  auto inout_ptr(Smart& s, Args&&... args);
}
```

これらの関数はそれぞれ、`std::out_ptr_t`、`std::inout_ptr_t`というクラステンプレートのオブジェクトを返します。

まず、`P`という型を次のように取得します

1. `Pointer`: `is_void_v<Pointer> == false`の場合
2. `T::pointer`: 1が取得できず、`T::pointer`が有効であり型名である場合
3. `T​::​element_type*`: 2が取得できず、`T​::​element_type`が有効であり型名である場合
4. `pointer_traits<T>​::​element_type*`: 3が取得できない場合

これは例えば`Pointer`を指定していないとすると、`std::unique_ptr<T>`に対して`T*`（`std::unique_ptr<T>::pointer`）になります。

そのうえで、次のものを構築して直接`return`します

- `std::out_ptr()`: `std::out_ptr_t<Smart, P, Args&&...>(s, std​::​forward<Args>(args)...)`
- `std::inout_ptr()`: `std::inout_ptr_t<Smart, P, Args&&...>(s, std​::​forward<Args>(args)...)`

`std::out_ptr`/`std::inout_ptr`はほぼ、この`return`文一行の単純な関数となります。

`std::out_ptr_t`、`std::inout_ptr_t`は次のようなクラステンプレートです

```cpp{style=cppstddecl}
namespace std {
  template<class Smart, class Pointer, class... Args>
  class out_ptr_t {
  public:
    explicit out_ptr_t(Smart&, Args...);
    out_ptr_t(const out_ptr_t&) = delete;

    ~out_ptr_t();

    operator Pointer*() const noexcept;
    operator void**() const noexcept;

  private:
    Smart& s;                   // exposition only
    tuple<Args...> a;           // exposition only
    Pointer p;                  // exposition only
  };


  template<class Smart, class Pointer, class... Args>
  class inout_ptr_t {
  public:
    explicit inout_ptr_t(Smart&, Args...);
    inout_ptr_t(const inout_ptr_t&) = delete;

    ~inout_ptr_t();

    operator Pointer*() const noexcept;
    operator void**() const noexcept;

  private:
    Smart& s;                   // exposition only
    tuple<Args...> a;           // exposition only
    Pointer p;                  // exposition only
  };
}
```

`std::out_ptr`/`std::inout_ptr`のやることはほぼこのクラスのコンストラクタとデストラクタ、および変換演算子によって行われています。

1. コンストラクタ: C API関数呼び出し前に、受け取ったスマートポインタをリセットする
    - `std::out_ptr_t`: 保持するリソースを開放する
    - `std::inout_ptr_t`: 保持するリソース（ポインタ）を所有権付きで取り出す
    - `s, a`はコンストラクタで受け取った対応する引数で初期化される
2. 変換演算子: C API関数への引数初期化時に、メンバで保持しているリソース受け取り用ポインタ（`p`）をキャストして渡す
    - 例えば、`std::unique_ptr<T>`に対して、`T**`か`void**`のどちらかに変換して渡す
      - 変換先はC API関数の引数型に応じて自動で決定される
3. デストラクタ: C API関数の呼び出し終了後に、メンバで保持しているリソース受け取り用ポインタ（`p`）の値をコンストラクタで受け取ったスマートポインタ（`s`）にセットする
    - 取得に失敗していたら（`nullptr`がセットされていたら）何もしない
    - コンストラクタで受け取った追加の引数（`a`に保存されているもの）をスマートポインタに転送するのもここで行われる

`std::inout_ptr`の説明の際に手でスマートポインタを操作していたコードと対応付けると分かりやすいかもしれません、C API呼び出し前の処理をコンストラクタで、呼び出し後の処理をデストラクタで行います。少しトリッキーかもしれませんが、`c_api_re_create_handle(24, std::inout_ptr(resource));`のように呼び出している時、`std::inout_ptr_t`の一時オブジェクト（第二引数に渡されている）は`c_api_re_create_handle()`の呼び出しが終了してこの文の実行が終了する際に破棄され、その時にデストラクタが呼ばれます。

従って特に、デストラクタが呼び出されることが`std::out_ptr`/`std::inout_ptr`にとって非常に重要になります。`std::out_ptr`/`std::inout_ptr`の戻り値を一回受けて左辺値として取り回すことは可能ですが、それを行うとうまく動作しなくなります。

なお、テンプレート化されていることから分かるかもしれませんが、`std::out_ptr`/`std::inout_ptr`は任意のスマートポインタで使用することができ、それは標準ライブラリ内のものだけが対象ではありません。例えば、Boostの提供するスマートポインタを使用することもできます。ただ、そのためには標準のスマートポインタとある程度インターフェース互換である必要があります。

これらの実装は特にC++20など特有の機能を使用していないので、C++11以降であればここで例示している宣言の定義を具体的に実装していけばほぼそのまま実装できるはずです。

### ポインタ型の明示的な指定

`std::out_ptr`/`std::inout_ptr`共に、テンプレートパラメータの`Pointer`は引数から推論されるものではなく、デフォルトで`void`が指定されていることで内部で適切なポインタ型を自動で決定します。この`Pointer`引数に`void`以外の型を指定することで、`std::out_ptr_t`/`std::inout_ptr_t`内部で保持されるポインタ型（`p`メンバの型）を明示的に指定することができます。

```cpp
int main() {
  using fclose_deleter = decltype([](auto* f) { std::fclose(f); });
  std::unique_ptr<std::FILE, fclose_deleter> file_ptr;

  constexpr const char* file_name = ...;
  if (fopen_s(std::out_ptr<std::FILE*>(file_ptr), file_name, "r+b") != 0) {
    return 1;
  }

  // *file_ptrは有効
  ...
  
  return 0;
}
```

この例は実際には指定する必要がない例ですが、例えばスマートポインタが基底クラスのポインタを保持するもので、APIが派生クラスのポインタを返すような場合に指定が必要になる場合があります。

```cpp
struct base { ... };
struct derived : public base { 
  int ID = 1;
  ...
};

// ポインタに派生クラスを渡すようなAPI
error_num api_create_derived(int id, void** p_handle);

void use_api() {
  std::unique_ptr<base> ptr{};

  // shared_ptrの場合はデリータをここで渡す
  auto ec = api_create_derived(derived::ID, std::out_ptr<derived*>(ptr));  // 👈

  if (ec == API_ERROR_CONDITION) {
    // エラーハンドル
    ...
  }

  // リソースの使用
  ...

  // リソースの解放は自動
}
```

`std::out_ptr(ptr)`として呼ぶ場合、`api_create_derived()`の中で`p_handle`に対して`derived`のポインタを直接代入しているとき（`*p_handle = new derived{}`のように）、`out_ptr_t::p`に保持されているポインタは`base**` -> `void**`の変換が行われてAPIに渡され、API内部ではそれをデリファレンスした`void*`に対して`derived*`を代入しており、ポインタ型の不整合が発生して未定義動作となってしまいます。

このような場合に`std::out_ptr<derived*>(ptr)`のように明示的にポインタ型を指定することで`out_ptr_t::p`に保持されているポインタは`derived*`型となり、APIに渡す際には`derived**` -> `void*`の変換が行われて渡されるため、そこに`derived*`を直接入れられてもポインタ型の不整合を回避することができ、最後にデストラクタでスマートポインタに書き戻される際には`out_ptr_t::p`を`ptr`にセットするため`derived*` -> `base*`の変換となり、これは安全な変換であるため全ての過程においてポインタ型の変換に問題は無くなります。

とはいえこのようなAPIはもはやC APIではありませんが、これに近いAPIを持つものとして例えばWindowsのCOMがあり、そのようなAPIにおいては内部的に不正なポインタの変換が起こってしまう場合があるため、このように`std::out_ptr`/`std::inout_ptr`の第一テンプレートパラメータでポインタ型を明示的に指定することが必要になる場合があります。

このような場合に`std::out_ptr`/`std::inout_ptr`を使用して起こることを簡単なコードで展開して書いてみるとつぎのようになっています

```cpp
struct base { };
struct derived : base { };

// 派生クラスのポインタを返すAPI
void pseudo_api(void** ptr) {
  // derived* -> void*の変換
  // *ptrがderived*であれば問題ない
  *ptr = new derived{};  
}

int main() {
  std::unique_ptr<base> ptr{};

  // pseudo_api(std::outptr(ptr))と呼んだ時に起こること
  {
    ptr.reset();

    base* tmp_ptr{};  // std::unique_ptr<base>::pointerから取得された型
    pseudo_api(reinterpret_cast<void**>(&tmp_ptr));
    ptr.reset(tmp_ptr);
  }

  // pseudo_api(std::outptr<derived*>(ptr))と呼んだ時に起こること
  {
    ptr.reset();

    derived* tmp_ptr{}; // 👈 ここで使用されるポインタ型が異なる
    pseudo_api(reinterpret_cast<void**>(&tmp_ptr));
    ptr.reset(tmp_ptr);
  }
}
```

非常に難しい問題であり判断が難しいですが、スマートポインタのポインタ型に対してAPIが返す実際の型が異なる場合、には想定されるポインタ型を指定しておいた方がいいかもしれません。

## `allocate_at_least()`

## `std::start_lifetime_as()`

C++20で導入された*implicit-lifetime type*と呼ばれるカテゴリの型は、その型のメモリが確保された後に暗黙的に生存期間が開始されます。

```cpp
struct X { int a, b; };

X* make_x() {
  X *p = (X*)malloc(sizeof(struct X));

  p->a = 1; // ok、C++20から
  p->b = 2; // ok、C++20から

  return p;
}
```

この言語機能については、C++20 言語機能本の「トリビアルな型のオブジェクトを暗黙的に構築する」を参照してください。

この仕様によって暗黙的に生存期間が開始されるのはいいのですが、ユーザーがそれを明示する手段はありませんでした。すなわち、*implicit-lifetime type*に対して明示的に同じ効果をもたらす関数が用意されていませんでした。

C++23では、それが`std::start_lifetime_as<T>(p)`として提供されます。`std::start_lifetime_as<T>(p)`は、`p`の領域で`T`のオブジェクトの生存期間を開始して、そのオブジェクトへのポインタを返します。返されたポインタを介してオブジェクトを扱うことで、安全に使用できます。

```cpp
struct X { int a, b; };

X* make_x() {
  X *p = std::start_lifetime_as<X>(malloc(sizeof(struct X)));

  p->a = 1; // ok
  p->b = 2; // ok

  return p;
}
```

とはいえこの例では無い場合と変わりません。*implicit-lifetime type*の生存期間が自動で開始されるのは、指定されている一部のライブラリ関数（と実装定義の関数）のみであるため、例えばユーザー定義のアロケータやメモリ確保関数などではそれが起こりません。`std::start_lifetime_as()`は、その場合に明示的に生存期間を開始するために使用できます。

```cpp
// ユーザー定義のメモリ確保関数
// メモリの確保のみを行うものとする
auto allocate_memory(std::size_t N) -> std::byte*;

struct X { int a, b; };

int main() {
  std::byte* mem = allocate_memory(sizeof(X));

  // Xはimplicit-lifetime typeではあるものの、生存期間が開始されていない（ユーザー定義関数であるため）
  X* ub_ptr1 = reinterpret_cast<X*>(mem);  
  ub_ptr1->a = 1; // ub
  ub_ptr1->b = 2; // ub

  // launderはすでに生存期間が開始されているオブジェクトへのポインタを取得するものなので、ここで使用できない
  X* ub_ptr2 = std::launder<X>(reinterpret_cast<X*>(mem));  
  ub_ptr2->a = 1; // ub
  ub_ptr2->b = 2; // ub

  // start_lifetime_as()を通すことで生存期間を開始する
  X* valid_ptr = std::start_lifetime_as<X>(mem);
  valid_ptr->a = 1; // ok
  valid_ptr->b = 2; // ok
}
```

また、この関数は呼び出されていることによって指定された領域における指定された型のオブジェクトの生存期間を開始することをコンパイラに伝達する関数であるので、実行時に何かをするわけではありません。おそらく定義としてはほぼ何もしない関数になるでしょう。

```cpp{style=cppstddecl}
namespace std {
  // 単一オブジェクト用
  template<class T>
  T* start_lifetime_as(void* p) noexcept;

  template<class T>
  const T* start_lifetime_as(const void* p) noexcept;
  template<class T>
  volatile T* start_lifetime_as(volatile void* p) noexcept;
  template<class T>
  const volatile T* start_lifetime_as(const volatile void* p) noexcept;

  // 配列用
  template<class T>
  T* start_lifetime_as_array(void* p, size_t n) noexcept;

  template<class T>
  const T* start_lifetime_as_array(const void* p, size_t n) noexcept;
  template<class T>
  volatile T* start_lifetime_as_array(volatile void* p, size_t n) noexcept;
  template<class T>
  const volatile T* start_lifetime_as_array(const volatile void* p, size_t n) noexcept;
}
```

単一オブジェクト用の関数と配列用の関数で2種類×4オーバーロードが用意されています。4パターンのオーバーロードは単にポインタのCV修飾を網羅するためのもので、関数の本質的な動作は同じです。

配列用の`std::start_lifetime_as_array()`では、配列の領域の先頭ポインタとその長さを受け取って、その場所で`T`の配列オブジェクトの生存期間を開始してそのオブジェクトへのポインタを返します。注意点ですが、この関数は要素数が実行時に決まる配列型に対して使用するもので、要素数が静的に既知の配列型に対しては`start_lifetime_as()`を使用します。

```cpp
auto allocate_memory(std::size_t N) -> std::byte*;

int main() {
  std::byte* mem = allocate_memory(sizeof(int) * 10);

  // 同じ領域に何度もオブジェクトを構築していることは無視して
  auto* p1 = std::start_lifetime_as<int[5]>(mem); // ok、要素数既知の配列型の生存期間を開始
  auto* p2 = std::start_lifetime_as<int[]>(mem);  // ng、要素数不明の配列型には使用できない

  auto* p3 = std::start_lifetime_as_array<int>(mem, 10);    // ok、配列型`int[10]`の生存期間を開始
  auto* p4 = std::start_lifetime_as_array<int[]>(mem, 10);  // ng、間違った使い方
  auto* p5 = std::start_lifetime_as_array<int[5]>(mem, 2);  // ok、`int[5][2]`の生存期間を開始
}
```

なお、`std::start_lifetime_as<T>()`で使用可能な型`T`は*implicit-lifetime type*である必要があり、その他の型では使用できません。ただし、配列型の場合は配列型そのものが要素型に関わらず*implicit-lifetime type*であるため、`std::start_lifetime_as_array()`では気にする必要はありません。

もしこのようなメモリ領域に*implicit-lifetime type*ではない型のオブジェクトを構築したい場合は、配置`new`などを使用して明示的にコンストラクタを呼び出します。

```cpp
auto allocate_memory(std::size_t N) -> std::byte*;

int main() {
  std::byte* mem = allocate_memory(sizeof(std::vector<int>));
  
  auto* p1 = std::start_lifetime_as<std::vector<int>>(mem); // ng、std::vector<int>はimplicit-lifetime typeではない
  auto* p2 = new (mem) std::vector<int>();    // ok、配置newでコンストラクタを呼び出し生存期間を開始

  ...

  std::destroy_at(p2);  // 明示的にデストラクタを呼び出す
}
```

*implicit-lifetime type*ではない型の生存期間を終了するためにはデストラクタを明示的に呼び出す必要があります。

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
