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

本書はC++23で追加された標準ライブラリ機能のうち、ヘッダ単位未満の小さめの機能について解説する本です。特に、「C++23 ライブラリ機能1」で取り扱われなかったC++23で追加されたライブラリ機能の残りの部分の解説を行っています。

## 注意など

本書はC++23をベースとして記述されています。そのため、C++20までに導入されている機能については特に導入バージョンについて触れず、既知のものとして詳しい解説なども行いません。

文中では基本的に`std::`を省略していませんが、文脈上明らかな場合などに一部で`std::`を省略することがあります。また、通常の関数の表記（`func()`）に対してメンバ関数の表記（`.member_func()`）は先頭に`.`を付加して区別します。

## サンプルコードのお約束

- ヘッダのインクルード/インポートは基本的に省略し、サンプルコードの最初で`import std;`しているものとします。
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

CTADでは、明示的に定義された推論補助に加えてコンストラクタから生成した推論補助が使用されます。ここではアロケータを受け取るコピーコンストラクタから生成された推論補助が使用されています。

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
// stringとvector<string>の一時オブジェクトが作られ、コピーされる
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
    explicit(...) constexpr pair(const T1& x, const T2& y); // (1)

    // 引数を完全転送して構築するコンストラクタ
    template <class U, class V>
    explicit(...) constexpr pair(U&& x, V&& y); // (2)
    ...
  };
}
```

このコンストラクタのうち`(2)`を使用してほしいわけです。しかし、引数に`{}`を指定していると`{}`からコンストラクタ引数のテンプレートパラメータを推論することができないため`(2)`はオーバーロード解決時に候補から外され、代わりに`(1)`が選択されます。`(1)`はコンストラクタテンプレートではなく`std::pair`のテンプレートパラメータが直接使用されているため`{}`の型が分かります。

`(1)`のコンストラクタの呼び出しにおいては、コンストラクタ引数の型が`std::pair`のテンプレートパラメータに指定された型と一致しない場合、それぞれ`T1, T2`に暗黙変換されその一時オブジェクトがコンストラクタに渡され、そこからコピーして要素が初期化されることになります。

先ほどの例では、`"hello"`は一時`std::string`オブジェクトに、`{}`は一時`std::vector<std::string>`オブジェクトに変換され、`std::pair`のコンストラクタに渡されます。`(1)`のコンストラクタ内部からは2つの引数はどちらも`const &`なので各要素の初期化においてはコピーが行われます。これにより、一時オブジェクトを生成してそこからさらにコピーが両方の引数で起こります。

この場合に`(2)`のコンストラクタを選択しようとすると、次のように書く必要があります

```cpp
std::pair<std::string, std::vector<std::string>> p("hello", std::vector<std::string>{});
```

しかしこれはかなり冗長な記述です。

`std::pair`の初期化において`{}`を活用できるようにするために、C++23からは`(2)`の転送コンストラクタのテンプレートパラメータに`std::pair`のテンプレートパラメータがデフォルト引数として指定されるようになります。

```cpp
namespace std {

  template<typename T1, typename T2>
  struct pair {
    ...

    // 引数をコピーして構築するコンストラクタ
    explicit(...) constexpr pair(const T1& x, const T2& y); // (1)

    // 引数を完全転送して構築するコンストラクタ
    template <class U = T1, class V = T2>
    explicit(...) constexpr pair(U&& x, V&& y); // (2)

    ...
  };
}
```

これによって、先ほどの例では`(2)`のコンストラクタが選択されるようになり、コンストラクタ引数は完全転送されて`std::pair`内部で各要素のコンストラクタに直接引き渡されるようになります。そのため、`std::pair`のコンストラクタへの引数渡しで一時オブジェクトが生成され、さらにそこからコピーのようなことは起こらなくなります。

```cpp
// 引数は完全転送され、各要素は引数から直接初期化される（(2)のコンストラクタが選択される
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

ここでは、`std::string`をキーとする連想コンテナによって有効化する方法を見てみます。

### （順序付き）連想コンテナ

順序付き連想コンテナ（`std::set, std::map`とその`multi`バージョン）の場合、デフォルトの比較クラスがテンプレートパラメータに指定された型の比較しかできないことから`is_transparent`が定義されていないため有効化されません。デフォルトの比較クラス`std::less`等は`void`に対する特殊化に対してのみ`is_transparent`が定義される（比較がテンプレートになる）のでそれを使うか、C++20で追加された`std::ranges`にある同名の比較クラスを使うことで有効化できます。

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
  using is_transparent = void;  // 👈 これが必要

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

## `<ranges>`/`<iterator>`

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

C++20にて`<ranges>`の追加と共にイテレータライブラリが改修された際、`move_iterator<I>::iterator_concept`は常に`input_iterator`となるようにされており、C++20`move_iterator`は常に`input_iterator`となります。

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

当初のRangeライブラリでは右辺値`range`をこのように非常に高度なテクニックによってオプトインするようにしていましたがこれには問題が多く（あまりにも高度過ぎる）、これは後に`enable_borrowed_range`変数テンプレートによるより明確かつわかりやすいオプトイン方法に置き換えられ、現在に至っています。現在の`std::ranges::begin`をはじめとするCPOでは、`enable_borrowed_range`な型`R`に対して右辺値の入力を左辺値に実体化した上でディスパッチを行う（入力は常に左辺値として扱う）ことで規格の表現と実装を簡素化しています（規格ではこのことを*reified object*という用語で表現しています）。

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

`A`のオブジェクトに対する`std::ranges::begin`はADLによって`A`の`begin()`（*Hidden frineds*定義のもの）を見つけてくれるはずで、そこでは2つのPoison Pillオーバーロードを含めたオーバーロード解決が行われます（`begin(auto&)`と`begin(const auto&)`）。前述のように、オーバーロード解決は実際の引数型の値カテゴリに関わらず`A`の左辺値オブジェクト（`A&`）に対して行われ、それに対しては`begin(const A&)`よりも`begin(auto&)`の宣言の方が優先順位が高くなります。そしてそれは`delete`されているため、素の型`A`に対する`range<A>`は`false`になります。しかし、`range<const A>`だとオーバーロード解決は`const A&`にマッチするため、定義されている`begin(const A&)`がPoison Pillオーバーロードよりも優先され、`range<const A>`は`true`となります。

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

\clearpage

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

\clearpage

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

  buffer.resize_and_overwrite(total_len,
    [=](char* data, std::size_t length) {
      char* dst = data + before_len;
      std::size_t append_len = 0;

      for (auto append_str : new_data) {
        const std::size_t step = append_str.size();
        memcpy(dst + append_len, append_str.data(), step);

        append_len += step;
      }
      
      assert((before_len + append_len) == length); // この場合は一致する

      return length;  // 書き込んだ領域サイズを返す
    }
  );
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

```cpp{style=cppstddecl}
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

```cpp{style=cppstddecl}
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
// C++23から、要素型/文字型によらずどちらもパスすることが保証される
static_assert(std::is_trivially_copyable_v<std::string_view>);
static_assert(std::is_trivially_copyable_v<std::span<int>>);
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

- `m`: 囲み文字をなくし、区切り文字を`: `にして出力
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

// formattableでテストしたほうがデバッグしやすい（かも
static_assert(std::formattable<MyType, char>);

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

　

`tuple`のフォーマットを利用することで、パラメータパック（可変長テンプレート引数）のフォーマットを簡単に行うことができます。

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

パラメータパックを愚直にフォーマットしようとするとその要素数に合わせてフォーマット文字列を生成しなくてはなりませんが、`std::tie()`を使用して`tuple`のフォーマットを利用することで簡単にフォーマットできるようになります。

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

### `basic_format_string`をユーザーが使用可能にする

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

C++20ではこの`std::format_string<Args...>`のような型が利用できなかったため、フォーマット文字列をこのように受け取ることはできず、普通の文字列で受け取って実行時フォーマットを行う、標準ライブラリの内部実装を使用する、などの方法しかありませんでした。

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

### デバッグオプション（`?`）

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

標準出力ストリーム（`basic_ostream`）に対するストリーム出力演算子（`<<`）では、`const void*`（非`volatile`ポインタ）の出力を行うオーバーロードはあり、普通のポインタはこれを使用してアドレスが出力されます。しかし、CV修飾が合わないため`volatile`ポインタはこのオーバーロードを使用できず、ポインタ->`bool`の変換を介して`bool`値として出力されます。このため、`nullptr`ではない`volatile`ポインタを出力すると`1`（`true`）になります。

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

C++コードからファイルをオープン（作成）しようとする場合、`<fstream>`のファイルストリームを使用することが多いと思います。ファイルの作成時に考慮すべきことは色々あるのですが、その中に「ファイルを新規で作成するが既に存在する場合は何もしない（作成に失敗してほしい）」というものがあります。一見単純なこの要件はセキュリティに敏感なアプリケーションで必要になりますが、厳密に達成することが難しい場合があります。

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

  if (!ofs) {
    // ファイルオープンに失敗（既に存在していたなど）
    throw std::runtime_error("file open failed");
  }

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

\clearpage

# `<functional>`

## `std::invoke_r()`

C++17以前から、関数呼び出しという操作を規格上で統一的に表現するために`INVOKE`という仮想操作が定義されており、C++17ではそれに応じた呼び出しを行うライブラリ関数である`std::invoke`が追加されました。

また、`INVOKE`した結果を指定した戻り値型`R`に変換する`INVOKE<R>`という操作も定義されており、こちらに対応するものとして`std::invoke<R>`のような形で提案されていましたが、C++17時点では不要であるとしてドロップされました。

しかし、`INVOKE<void>(f, args...)`のような呼び出しは戻り値を明示的に破棄するために使用でき、`std::is_invocable_r`や`std::is_nothrow_invocable_r`は指定した戻り値型で呼び出せるかを調べられるようになっています。さらに、`std::visit`には戻り値型を指定する`std::visit<R>`が用意されています。

このように、これらの有用性は既に示されていたため、これらのものに準じる形で`std::invoke_r`がC++23で追加されます。

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
auto f2(T&&) -> T&&;

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

これは`std::function`自体をコピー可能にするために、保持するものに対してもコピー可能であることを要求するためだと思われます。また、`std::function`はC++にムーブの概念が無かった時代に設計された`boost::function`をベースとしており、C++11でムーブセマンティクスとほぼ同時に導入されているため、ムーブについて考慮する時間が無かったことも理由にありそうです。

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

`std::move_only_function`では保持する関数型の指定に`const`を指定することができ、これによって保持する関数の`const`有無を選択して呼び出すことができ、また`std::move_only_function`の`const`性が呼び出しコンテキストに適用されるようになります。

```cpp
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
|非`const`|どちらも（非`const`優先）|`🤷`|
|`const`|どちらも（非`const`優先）|`🤷`|

`🤷`は指定の方法が無い（`std::function<R() const>`はコンパイルエラー）ことを表しています。

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

\clearpage

# `<memory>`

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

その際にはまず、`P`という型を次のように取得します

1. `Pointer`: `is_void_v<Pointer> == false`の場合
2. `T::pointer`: 1が取得できず、`T::pointer`が有効であり型名である場合
3. `T::element_type*`: 2が取得できず、`T::element_type`が有効であり型名である場合
4. `pointer_traits<T>::element_type*`: 3が取得できない場合

これは例えば`Pointer`を指定していないとすると、`std::unique_ptr<T>`に対して`T*`（`std::unique_ptr<T>::pointer`）になります。

そのうえで、次のものを構築して直接`return`します

- `std::out_ptr()`: `std::out_ptr_t<Smart, P, Args&&...>(s, std::forward<Args>(args)...)`
- `std::inout_ptr()`: `std::inout_ptr_t<Smart, P, Args&&...>(s, std::forward<Args>(args)...)`

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
    Smart& s;                   // 説明専用
    tuple<Args...> a;           // 説明専用
    Pointer p;                  // 説明専用
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
    Smart& s;                   // 説明専用
    tuple<Args...> a;           // 説明専用
    Pointer p;                  // 説明専用
  };
}
```

`std::out_ptr`/`std::inout_ptr`のやることはほぼこのクラスのコンストラクタとデストラクタ、および変換演算子によって行われています。

1. コンストラクタ: C API関数呼び出し前に、受け取ったスマートポインタをリセットする
    - `std::out_ptr_t`: 保持するリソースを開放する
    - `std::inout_ptr_t`: 保持するリソース（ポインタ）を所有権付きで取り出す
    - `s, a`はコンストラクタで受け取った対応する引数で初期化される
2. 変換演算子: C API関数への引数初期化時に、メンバで保持しているリソース受け取り用ポインタ（`p`）のアドレスをキャストして渡す
    - 例えば、`std::unique_ptr<T>`に対して、`T**`か`void**`のどちらかに変換して渡す
      - 変換先はC API関数の引数型に応じて自動で決定される
3. デストラクタ: C API関数の呼び出し終了後に、メンバで保持しているリソース受け取り用ポインタ（`p`）の値をコンストラクタで受け取ったスマートポインタ（`s`）にセットする
    - 取得に失敗していたら（`nullptr`がセットされていたら）何もしない
    - コンストラクタで受け取った追加の引数（`a`に保存されているもの）をスマートポインタに転送するのもここで行われる

`std::inout_ptr`の説明の際に手でスマートポインタを操作していたコードと対応付けると分かりやすいかもしれません、C API呼び出し前の処理をコンストラクタで、呼び出し後の処理をデストラクタで行います。少しトリッキーかもしれませんが、`c_api_re_create_handle(24, std::inout_ptr(resource));`のように呼び出している時、`std::inout_ptr_t`の一時オブジェクト（第二引数に渡されている）は`c_api_re_create_handle()`の呼び出しが終了してこの文の実行が終了する際に破棄され、その時にデストラクタが呼ばれます。

従って特に、デストラクタが呼び出されることが`std::out_ptr`/`std::inout_ptr`にとって非常に重要になります。`std::out_ptr`/`std::inout_ptr`の戻り値を一回受けて左辺値として取り回すことは可能ですが、それを行うとうまく動作しなくなります。

また、テンプレート化されていることから分かるかもしれませんが、`std::out_ptr`/`std::inout_ptr`は任意のスマートポインタで使用することができ、それは標準ライブラリ内のものだけが対象ではありません。例えば、Boostの提供するスマートポインタを使用することもできます。ただ、そのためには標準のスマートポインタとある程度インターフェース互換である必要があります。

なお、これらの実装は特にC++20など特有の機能を使用していないので、C++11以降であればここで例示している宣言の定義を具体的に実装していけばほぼそのまま実装できるはずです。

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
  static constexpr int ID = 1;
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

例えば次のような`std::vector::reserve()`の呼び出しにおいて

```cpp
std::vector<char> v;
v.reserve(37);

...

v.reserve(38);
```

使用される`operator new`（あるいはその内部の確保関数）が実際に37バイト丁度を確保する、という事はほぼありません（アライメントの制約やパフォーマンスの向上など、実装の都合による）。

しかし、`std::vector`の内部実装からは実際にどれだけの量を確保したのかを知る方法が無いため、1回目の`reserve()`時に37バイト分のメモリ確保依頼しか行っていなければ、2回目の`reserve(38)`で再度のメモリ確保を行わずに38バイト目を安全に使用する方法はありません。

`std::vector::reserve()`の実装は簡単には次のようになります

```cpp
void vector::reserve(size_t new_cap) {
  if (capacity_ >= new_cap) return;

  const size_t bytes = new_cap;
  void *newp = alloc_.allocate(new_cap);
  memcpy(newp, ptr_, capacity_);
  
  ptr_ = newp;
  capacity_ = bytes;
}
```

`capacity_`というのが使用可能なメモリ量を記録している`std::vector`のメンバに対応しますが、この実装においてはこれはあくまでユーザーが指定した値`new_cap`で更新されます。3行目の`::operator new`が実際に確保している`new_cap`を超える部分の領域サイズを知る方法はありません（実際の実装ではメモリ確保の回数を要素数に対して償却定数に抑える工夫がなされるのでもう少し複雑になる）。

僅かではありますが、この余剰部分の量を知ることができればメモリ確保を行う回数を削減できる可能性があります。

この用途のために、C++23では`std::allocator_traits`及び`std::allocator`のインターフェースとして`allocate_at_least()`が追加されます。

```cpp{style=cppstddecl}
namespace std {
  template<typename Pointer, typename SizeType = size_t>
  struct allocation_result {
    Pointer ptr;
    SizeType count;
  };

  template<typename T>
  class allocator {
    
    ...

    constexpr T* allocate(size_t n);
    
    constexpr allocation_result<T*> allocate_at_least(size_t n);  // 👈
    
    ...

  };
}
```

`allocate_at_least()`は`allocate()`同様にメモリ確保を行う関数ですが、その戻り値として確保した領域のポインタに加えて実際に確保して使用可能なサイズを返します。このサイズ情報を活用することで、要求したサイズを超えた部分のメモリ領域を安全に使用できるようになります。

これによって、先ほどの`reserve()`実装は次のように改善できます

```cpp
void vector::reserve(size_t new_cap) {
  if (capacity_ >= new_cap) return;

  auto [newp, new_size] = alloc_.allocate_at_least(new_cap, return_size);  // 実際の確保サイズを受け取る
  memcpy(newp, ptr_, capacity_);

  ptr_ = newp;
  capacity_ = new_size; // 実際に使用可能なサイズでキャパシティを更新
}
```

この例のように標準ライブラリ実装内で活用される機能ではありますが、独自のコンテナを実装している場合も同様に恩恵を受けることができます。

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

## `std::allocator_traits`のユーザー定義特殊化の禁止

C++11で導入された`std::allocator_traits`には、主に次の2つの目的がありました

1. アロケータのデフォルト実装を提供することで、アロケータの要件を最小限にする
2. 将来的に、アロケータ要件を変更せずに（既存アロケータを変更せずに）アロケータインターフェースを拡張できる仕組みを提供する

しかし、C++20時点ではユーザーが`std::allocator_traits`の特殊化を定義することが許可されているため、2つ目の目的は達成されていません。ユーザーが`std::allocator_traits`を特殊化する場合は標準のものと同じインターフェースを備えることを要求していますが、`std::allocator_traits`のAPIが拡張されるとそのユーザー定義特殊化は非準拠なものになってしまいます。もちろん、ユーザー定義特殊化を更新して新しいAPIに対応させることは可能ですが、それが行われるまでの間は非準拠な状態になります。

このことは既に悪影響を及ぼしており、少し前の`allocate_at_least()`は当初この問題を考慮していたためフリー関数も追加されていました。

C++23ではこの問題を解決するために、`std::allocator_traits`のユーザー定義特殊化が禁止されます。現在そのようなことをしているコードはサポートされなくなります。

\clearpage

# `<utility>`

## `std::to_underlying()`

`std::to_underlying()`はある列挙型の値をその基底型の整数値へ変換する関数です。

```cpp{style=cppstddecl}
namespace std {
  template <class T>
  constexpr underlying_type_t<T> to_underlying(T value) noexcept {
    return static_cast<underlying_type_t<T>>(value);
  }
}
```
```cpp
#include <utility>

enum class E : std::uint32_t {
  A = -1,
  B = 10,
  C = 23,
};

int main() {
  E e = E::B;
  auto se = std::to_underlying(e);

  assert(se == 10);  // ✅
}
```

非常に小さく単純な関数ではありますが、このような変換処理は頻繁に必要になるため様々なコードベースで見かけることができます。また、上記の実装例のような正しいキャストを行い実装するのが案外難しかったことから標準ライブラリに追加されています。

## `std::forward_like()` 

`std::forward_like()`はDeducing thisを用いた関数においてそのメンバ変数を適切に完全転送するために使用するユーティリティ関数であり、`std::forward()`とよく似たものです。

Deducing thisについては「C++23 コア言語機能」を参照していただくことにして、そこでは`std::optional`の様なクラス型においての`.value()`関数の実装を集約する例を挙げていました。分かりやすく単純化すると次のようなコードになりますが、この時にメンバの完全転送をどのように記述するかが問題になります

```cpp
template<typename T>
struct wrap {
  T v;

  template<typename Self>
  auto value(this Self&& self) -> decltype(auto) {
    return std::forward<Self>(self).v;  // あってる？
  }
};
```

`this`引数をテンプレート化することによって、`*this`の値カテゴリをそのテンプレートパラメータ`Self`から取得できるようになり、それを用いることで4種のオーバーロードを1つにまとめられるようになりました。しかし、`std::forward<Self>(self).v`は本当にあらゆる場合に適切な完全転送になっているのでしょうか？

`*this`の状態（`const`と値カテゴリ）とメンバ変数の宣言型（`const`/参照修飾の有無）、それらの組み合わせ時の`std::forward<Self>(self).v`の型を一覧すると次の表のようになります

|`this`（`self`）|メンバ（`v`）|`std::forwad(self).v`|
|---|---|---|
|||`&&`|
|`&`||`&`|
|`&&`||`&&`|
|`const`||`const &&`|
|`const &`||`const &`|
|`const &&`||`const &&`|

表中の空白は`const`も参照修飾もない状態です（`this`の場合はコピーされている、すなわち*prvlaue*の状態、変数の場合は`const`でも参照でもない状態）。

メンバ変数が普通に（`const`でも参照でもなく）宣言されている場合は問題ないように見えます。

次はメンバ変数が`const`である場合です

|`this`（`self`）|メンバ（`v`）|`std::forwad(self).v`|
|---|---|---|
||`const`|`const &&`|
|`&`|`const`|`const &`|
|`&&`|`const`|`const &&`|
|`const`|`const`|`const &&`|
|`const &`|`const`|`const &`|
|`const &&`|`const`|`const &&`|

ここも正しい結果になっていそうです。では次はメンバ変数が参照である場合です

|`this`（`self`）|メンバ（`v`）|`std::forwad(self).v`|
|---|---|---|
||`&`|`&`|
|`&`|`&`|`&`|
|`&&`|`&`|`&`|
|`const`|`&`|`&`|
|`const &`|`&`|`&`|
|`const &&`|`&`|`&`|
||`&&`|`&`|
|`&`|`&&`|`&`|
|`&&`|`&&`|`&`|
|`const`|`&&`|`&`|
|`const &`|`&&`|`&`|
|`const &&`|`&&`|`&`|

ここで問題が起きています。`std::forwad(self).v`ではメンバが参照型だと`*this`の値カテゴリや状態に関係なく左辺値（`&`）として扱われてしまいます。`const`は適用されることが望ましく、両方が右辺値（参照）であるならば結果もまた右辺値になる方が直観的です。

`const`と参照の組み合わせでも同じ問題が発生しています

|`this`（`self`）|メンバ（`v`）|`std::forwad(self).v`|
|---|---|---|
||`const &`|`const &`|
|`&`|`const &`|`const &`|
|`&&`|`const &`|`const &`|
|`const`|`const &`|`const &`|
|`const &`|`const &`|`const &`|
|`const &&`|`const &`|`const &`|
||`const &&`|`const &`|
|`&`|`const &&`|`const &`|
|`&&`|`const &&`|`const &`|
|`const`|`const &&`|`const &`|
|`const &`|`const &&`|`const &`|
|`const &&`|`const &&`|`const &`|

このように、`std::forwad(self).v`ではメンバ変数が参照である場合の完全転送が正確ではありません。

`std::forward_like()`は、このような場合に`*this`の状態とメンバ変数の宣言型に応じてそれらをマージした形の最適な完全転送を実現するためのユーティリティです。

```cpp
template<typename T>
struct wrap {
  T v;

  template<typename Self>
  auto value(this Self&& self) -> decltype(auto) {
    return std::forward_like<Self>(self.v);  // 👈
  }
};
```

`std::forward_like()`の場合、先ほど問題のあったメンバ変数が参照である場合にも正しい完全転送を行えます。先ほどの表に`std::forward_like()`の結果を追記するとつぎのようになります（列表記を一部省略しています）

|`this`|`v`|`std::forwad`|`std::forward_like`|
|---|---|---|---|
||`&`|`&`|`&&`|
|`&`|`&`|`&`|`&`|
|`&&`|`&`|`&`|`&&`|
|`const`|`&`|`&`|`const &&`|
|`const &`|`&`|`&`|`const &`|
|`const &&`|`&`|`&`|`const &&`|
||`&&`|`&`|`&&`|
|`&`|`&&`|`&`|`&`|
|`&&`|`&&`|`&`|`&&`|
|`const`|`&&`|`&`|`const &&`|
|`const &`|`&&`|`&`|`const &`|
|`const &&`|`&&`|`&`|`const &&`|

このように、メンバ変数が参照である場合も適切に`*this`の状態が適用されるようになります。なお、メンバ変数が参照でも`const`でもない場合は`std::forwad`の結果と変わりません。

メンバが`const`参照である場合も同様に修正されます

|`this`|`v`|`std::forwad`|`std::forward_like`|
|---|---|---|---|
||`const &`|`const &`|`const &&`|
|`&`|`const &`|`const &`|`const &`|
|`&&`|`const &`|`const &`|`const &&`|
|`const`|`const &`|`const &`|`const &&`|
|`const &`|`const &`|`const &`|`const &`|
|`const &&`|`const &`|`const &`|`const &&`|
||`const &&`|`const &`|`const &&`|
|`&`|`const &&`|`const &`|`const &`|
|`&&`|`const &&`|`const &`|`const &&`|
|`const`|`const &&`|`const &`|`const &&`|
|`const &`|`const &&`|`const &`|`const &`|
|`const &&`|`const &&`|`const &`|`const &&`|

`std::forward_like()`の型決定モデルでは、`*this`の`const`/値カテゴリの状態とメンバ変数の宣言型の`const`/参照修飾の状態をマージした結果を生成します。この時、`*this`の状態を参照そのものではなく参照先にも適用するような決定モデルになっています。そのため、参照メンバは`*this`が右辺値であれば右辺値参照になります。

```cpp{style=cppstddecl}
namespace std {
  template<class T, class U>
  constexpr auto forward_like(U&& x) noexcept -> ...;
}
```

`std::forward_like`のテンプレートパラメータ`T`はユーザーが明示的に指定するもので、これは通常`this Slef&& self`のようにテンプレートパラメータ`Self`で`this`引数を受けているときの`Self`を指定します。`U`はテンプレート引数推論によって決定されるもので通常ユーザーが指定する必要はなく、`std::forward_like<Self>(x)`の`x`には完全転送対象のメンバ変数を指定します。

また、`this auto&& self`のように引数を受ける場合は`decltype(self)`を`T`に指定すればokです

```cpp
auto value(this auto&& self) -> decltype(auto) {
  return std::forward_like<decltype(self)>(self).v;
}
```

## `std::unreachable()`

プログラム中のある制御フローにおいて、プログラマはそこに到達しないことを知っていたとしても、コンパイラにとっては必ずしもいつもそれが分かるわけではありません。そのような場合に、コンパイラにそれを通知する方法があるとより効率的なプログラムを作成できる可能性があります。

論理的には到達しうるが実際は到達しえないようなプログラム中の部分は例えば、`switch`文でよく見る事ができます

```cpp
void do_something(int number_that_is_only_0_1_2_or_3) {
  // 引数として0, 1, 2, 3のいずれかだけが来ることが分かっているとする
  switch (number_that_is_only_0_1_2_or_3) {
  case 0:
  case 2:
    handle_0_or_2();
    break;
  case 1:
    handle_1();
    break;
  case 3:
    handle_3();
    break;
  }
}
```

このような場合でもコンパイラは4以上の入力を処理するためのコードを生成します。この時、4以上の入力が決して来ないことがわかっていて、それをコンパイラに伝える事ができればそのような余分なコードの生成を抑止する事ができます。

このような要望は決して小さくはなかったようで、コンパイラは独自にこのような伝達手段を用意している場合があります

- GCC、clang、ICC : `__builtin_unreachable()`
- MSVC : `__assume(false)`

サポートがないコンパイラでは意図的に未定義動作を起こすコードを挿入することでこれと同等の効果を得られる場合もありますが、直観的ではなく未定義動作の結果でしかないため確実ではなく、移植可能な方法はありませんでした。

C++23では、プログラマの把握する到達不可能コードをコンパイラに通知するための関数として`std::unreachable()`が用意されます。

```cpp{style=cppstddecl}
namespace std {
  [[noreturn]] void unreachable();
}
```

これは、先ほどの`switch`文では次のように使用できます

```cpp
void do_something(int number_that_is_only_0_1_2_or_3) {
  switch (number_that_is_only_0_1_2_or_3) {
  case 0:
  case 2:
    handle_0_or_2();
    break;
  case 1:
    handle_1();
    break;
  case 3:
    handle_3();
    break;
  default:
    std::unreachable(); // 👈 ここには到達しないことを表明
  }
}
```

このように、`std::unreachable()`の呼び出しを書いておくことでそれが呼び出されている場所には制御が到達しないことを表明します。コンパイラが`std::unreachable()`の情報を有効活用した場合、この関数は0~3以外の値を考慮するコードを削除して最適化することができます。

ただし、`std::unreachable()`が実行時に実際に呼び出されてしまうと未定義動作となります。なお、この関数の呼び出しそのものには何の効果も規定されませんが、実装によってはログ出力やその場での終了などを行うことがあるようです。

規格的には、`std::unreachable()`は決して満たされない事前条件を持っていることで呼び出しが行われた瞬間に事前条件違反が発生することで未定義動作となり、未定義の呼び出しを伴うことによってコンパイラはその場所が実行されないと仮定する事ができ、それによって到達不能性を表現します。ちなみに、この決して満たされない事前条件は"`false` is `true`"と指定されています。

## `std::exchange`の`noexcept`指定

`std::exchange`はオブジェクトの値の交換を行うことができる関数であり、うまく利用すると地味に便利に使用することができるユーティリティ関数です。しかし、C++20までは`std::exchange`は`noexcept`指定されておらず、これを使用した関数の`noexcept`を書くのが少し面倒でした。

```cpp{style=cppstddecl}
namespace std {
  // 規格に規定されているexchangeの定義の例
  template<class T, class U = T>
  constexpr T exchange(T& obj, U&& new_val) { // noexceptは規定にない
    T old_val = std::move(obj);
    obj = std::forward<U>(new_val);
    return old_val;
  }
}
```

指定されている実装を見ると、例外を投げうるのは1行目の`T`のムーブ構築と、2行目の`U -> T`のムーブ代入、`return`文における暗黙ムーブ時の`T`のムーブ構築です。そのため、`std::exchange`が例外を投げるかどうかは`std::is_nothrow_move_constructible<T>`と`std::is_nothrow_assignable<T&, U>`によって求めることができます。

C++23からは、これらを条件とした`noexcept`が`std::exchange`に付加されるようになります。

```cpp{style=cppstddecl}
namespace std {
  // 規格に規定されているexchangeの定義の例
  template<class T, class U = T>
  constexpr T exchange(T& obj, U&& new_val)
    noexcept(is_nothrow_move_constructible<T> && is_nothrow_assignable<T&, U>)
  {
    ...
  }
}
```

これにより、`std::exchange`を使用する関数の`noexcept`指定は`noexcept(noexcept(std::exchange(...)))`のように書くことかできるようになります。

## `std::apply`の`noexcept`指定

C++20では、`std::exchange`同様に`std::apply`でも`noexcept`指定がされていませんでした。

```cpp{style=cppstddecl}
namespace std {
  // 説明専用関数
  template<class F, class Tuple, size_t... I>
  constexpr decltype(auto)
    apply-impl(F&& f, Tuple&& t, index_sequence<I...>) {
      return invoke(forward<F>(f), get<I>(forward<Tuple>(t))...)
    }

  template<class F, class Tuple>
  constexpr decltype(auto)
    apply(F&& f, Tuple&& t) { // noexceptは規定にない
      return apply-impl(forward<F>(f), forward<Tuple>(t),
                  make_index_sequence<tuple_size_v<remove_reference_t<Tuple>>>{});
    }
}
```

こちらの場合は少し実装が複雑ですが、`apply-impl()`で利用可能なテンプレート引数等を用いると、`noexcept(invoke(forward<F>(f), get<I>(forward<Tuple>(t))...))`のように指定でき、C++23ではこれが`std::apply()`の`noexcept`として指定されるようになります。

## `std::pair`と2要素`std::tuple`/`tuple`ライクな型との間での互換性の向上

`std::pair<T, U>`と2要素の`std::tuple<T, U>`は意味的に同一の型であり、標準ライブラリに両方が存在しているのは歴史的経緯以外の理由がありません。この2つの型は多くのインターフェースを共有していますが、一部非互換な部分があります。

```cpp
int main() {
  std::pair p1{1, 3.0};

  // pairからtupleを構築できるが
  std::tuple t1{p1};  // ok

  // tupleからpairを構築できない
  std::pair<int, double> p2{t1};  // ng

  // pairとtupleで比較できない
  bool b = p1 == t1;  // ng
  auto c = p1 <=> t1; // ng
}
```

`std::tuple`のインターフェースのことをタプルプロトコルと呼び、タプルプロトコルを実装している型は`std::tuple`を使用できる場所で`std::tuple`の代わりに使用することができます。`std::pair`と2要素`std::tuple`の非互換はタプルプロトコルを介してその影響を大きくしています。標準ライブラリ中でタプルプロトコルを実装している型には次のものがあります

- `std::array`
- `subrange`
- `view::enumrate`の参照型

例えば、`std::map`等の連想コンテナはその要素型として`std::pair`を使用していますが、`std::pair`が`std::tuple`あるいは`tuple`ライクな型から構築できないことによって、`<ranges>`のAPIでは2要素の`std::tuple`の代わりに`std::pair`を使用するようにしています。

```cpp
auto tuple_seq() -> std::vector<std::tuple<int, std::string_view>>;
auto pair_seq() -> std::vector<std::pair<int, std::string_view>>;

int main() {
  namespace ranges = std::ranges;

  auto m1 = tuple_seq() | ranges::to<std::map>(); // ng
  auto m2 = pair_seq() | ranges::to<std::map>();  // ok
}
```

C++23では、このような`std::tuple`およびそれと互換のある型と`std::pair`との間の非互換性がかなり軽減されます。具体的には

- 2要素`tuple`ライクな型から`std::pair`を構築できる
- 現在`std::pair`から構築できる場所では、2要素の`tuple`ライクな型からも構築できる
    - `std::tuple`そのものでも
- `tuple`と`tuple`ライクな型との比較が定義される
    - `==`と`<=>`
- `std::tuple`と`tuple`ライクな型との間の`common_type`/`common_reference`が定義される

などの変更がなされます。

これによって、最初の例でできなかったことが可能になるとともに、それは`tuple`ライクな型へも展開されます。

```cpp
int main() {
  std::pair p1{1, 3.0};

  // pairからtupleを構築できる
  std::tuple t1{p1}; // ok

  // tupleからpairを構築できる
  std::pair<int, double> p2{t1};  // ok、C++23から

  // pairとtupleで比較できる
  bool b = p1 == t1;  // ok、C++23から
  auto c = p1 <=> t1; // ok、C++23から

  // このことはtupleライクな型へも拡張される
  std::array<int, 2> arr{1, 2};

  std::pair<int, int> p3{arr};  // ok、C++23から
  std::tuple<int, int> t2{arr}; // ok、C++23から

  p3 == arr;  // ok、C++23から
  t2 == arr;  // ok、C++23から
}
```

また、`std::map`等連想コンテナが`std::pair`を使用し続けることは変わらないものの、この非互換性の緩和とそれを考慮したサポートの追加によって、2要素`tuple`ライクな型のシーケンス（イテレータ）からの変換が行えるようになります。

```cpp
auto tuple_seq() -> std::vector<std::tuple<int, std::string_view>>;
auto pair_seq() -> std::vector<std::pair<int, std::string_view>>;

int main() {
  namespace ranges = std::ranges;

  auto m1 = tuple_seq() | ranges::to<std::map>(); // ok、C++23から
  auto m2 = pair_seq() | ranges::to<std::map>();  // ok
}
```

ただし、これらの変更によって既存のコードがコンパイルエラーを起こす場合があります。

一つは、条件演算子の結果型です

```cpp
// C++20まではxの型はtuple<int, int>
// C++23からはコンパイルエラー（相互に変換可能になるため）
auto x = true ? tuple{0,0} : pair{0,0};
```

もう一つは`std::tuple`への変換演算子とタプルプロトコルを両方備えていた場合です

```cpp
struct M {
  operator tuple<int, int>() const { return {1, 1}; }
};

namespace std {
  template <> struct tuple_size<M> : integral_constant<size_t, 2> { };
  template <int I> struct tuple_element<I, M> { using type = int; };
  template <int I> auto get(M) { return 2; }
}

int main() {
  M m{};

  // C++20まで、tuple<int, int>{1, 1}
  // C++23から、tuple<int, int>{2, 2}
  std::tuple t{};
}
```

これは、新しく追加される`tuple`ライクな型から構築するコンストラクタが変換演算子呼び出しよりも優先されるためです。

しかし、どちらのケースもかなり稀であり存在する可能性が低いと判断されたため、破壊的変更が受け入れられています。

\clearpage

# コンセプトと型特性

## `std::is_scoped_enum`

`std::is_scoped_enum<E>`は、型`E`が列挙型かつスコープ付き列挙型（`enum class`）であるかを判定する型特性です。

```cpp
struct S {};
enum E1 { A, B, C };          // スコープなし列挙型
enum class E2 { A, B, C };    // スコープ付き列挙型

static_assert(std::is_scoped_enum_v<S> == false);
static_assert(std::is_scoped_enum_v<E1> == false);
static_assert(std::is_scoped_enum_v<E2>); // enum classだけでtrue
```

スコープ付き列挙型だけを検出したい場合というのは非常に稀とは思われますが、提案ではC++11以前のレガシーなコードからの移行の際に旧来の列挙型からスコープ付き列挙型への変更を検出するのに使用した、という実用例が報告されています。

### 実装について

列挙型を検出することは`std::is_enum`で可能ですが、それがスコープ付き列挙型かどうかを検出するという部分がこの型特性の肝になります。これは単純に、列挙型の値がその基底型へ暗黙変換可能であるかどうかを調べることによって実装できます。

```cpp
template<typename E, bool = std::is_enum_v<E>>
inline constexpr bool is_scoped_enum_check = false;

// enum classは整数型へ暗黙変換できないのでそれを追加でチェックする
template<typename E>
inline constexpr bool is_scoped_enum_check<E, true>
  = std::is_convertible_v<E, std::underlying_type_t<E>> == false;

template<typename E>
struct is_scoped_enum : std::bool_constant<is_scoped_enum_check<E>> {};
```

スコープ付き列挙型ではその列挙型の値を基底型含めた整数型に暗黙変換できないため、列挙型`E`に対する`std::is_convertible_v<E, std::underlying_type_t<E>>`は常に`false`となり、これを利用してスコープ付き列挙型を検出できます。

## 一時オブジェクトが参照に束縛されたことを検出する型特性

標準ライブラリを始めとするジェネリックなライブラリでは、ある型`T`を別の型の値から変換して初期化する事がよく必要になります。この時、`T`が`const`参照型だと容易にダングリング参照が作成されます。

```cpp
using namespace std::string_literals;

std::tuple<const std::string&> t1("hello");  // ub、ダングリング参照
```

例えばこの例での`t1`は常にダングリング参照を生成します。`std::tuple`のメンバとなる`const std::string&`は一時オブジェクトの寿命延長を行うことができる参照であり、完全転送されてくる引数`"hello"`は`std::string`ではない文字列リテラルです。ここで、`const std::string&`を初期化するために`std::string`の一時オブジェクトが作成され、その一時オブジェクトは`std::tuple`のメンバとなる`const std::string&`を初期化した後、コンストラクタの完了と共に寿命が尽きます。このため、`std::string_view`で受けた時とは異なりこのコードは常に未定義動作となります。

また、別の例として参照を返す`std::function`があります。

```cpp
std::function<const std::string&()> f = [] { return ""; };

auto& str = f();  // ダングリング参照を返す
```

この場合は、`std::function`の`operator()`の中で保持する関数が呼び出される際にその戻り値が`std::function`の関数型に指定された戻り値型に暗黙変換されることで一時オブジェクトが生成され、その一時オブジェクトへの参照が返されます。このため、`str`は常にダングリング参照となります。なお、この場合の暗黙変換は`std::function`の`operator()`の`return`文で起こります。

この問題は上記例のように、標準ライブラリ内部ですら普通に起こりえます。そのうえ、一部の実装では標準ライブラリ内部で起こるコンパイラの警告を抑制していることがあるため、コンパイラの警告で気付くことができなくなる場合もあります。

C++23では、このような一時オブジェクトが参照に束縛されたことを検出する型特性が2種類追加されます。

```cpp{style=cppstddecl}
namespace std {
  template<class T, class U>
  struct reference_constructs_from_temporary;

  template<class T, class U>
  struct reference_converts_from_temporary;

  template<class T, class U>
  inline constexpr bool reference_constructs_from_temporary_v
    = reference_constructs_from_temporary<T, U>::value;

  template<class T, class U>
  inline constexpr bool reference_converts_from_temporary_v
    = reference_converts_from_temporary<T, U>::value;
}
```

これらの型特性の名前の`constructs`と`converts`は検出する対象の構築（`T t(u)`）と変換（`T t = u`）の違いを表しています。この2つの型特性は、型`U`から`T`の構築/変換時に一時オブジェクトの寿命延長が発生する場合に`true`を返すものです。

`R t(u);`あるいは`R t = u;`、`U = decltype(u)`として、このような変数初期化時に一時オブジェクトの寿命延長が発生するのは、次の場合です

- `R`が`const T&`の参照型であり、`U`が`T`のprvalueである場合
- `R`が`const T&`の参照型であり、`U`が`T`に変換可能である場合
- `R`が`T&&`の参照型であり、`U`が`T`のprvalueである場合
- `R`が`T&&`の参照型であり、`U`が`T`に変換可能である場合

このいずれかに該当する場合に、参照`t`は`u`あるいは`u`から変換される一時オブジェクトを束縛し、なおかつその一時オブジェクトの寿命は`t`の寿命まで延長されます。通常の実行パスの変数として使用する際は問題ないのですが、クラスのメンバの初期化や関数の戻り値でこれが起こると、その一時オブジェクトの寿命は初期化や関数のスコープを抜けた時点で終了してしまい、ダングリング参照が発生します。

したがって、この2つの型特性が`true`を返す場合はまず`T`が参照型である必要があります。

```cpp
// 単純に寿命延長が行われる場合
static_assert(std::reference_constructs_from_temporary_v<const int&, int>);
static_assert(std::reference_constructs_from_temporary_v<int&&, int>);
static_assert(std::reference_converts_from_temporary_v<const int&, int>);
static_assert(std::reference_converts_from_temporary_v<int&&, int>);

// 変換された一時オブジェクトの寿命延長が行われる場合
static_assert(std::reference_constructs_from_temporary_v<const int&, short>);
static_assert(std::reference_constructs_from_temporary_v<int&&, short>);
static_assert(std::reference_converts_from_temporary_v<const int&, short>);
static_assert(std::reference_converts_from_temporary_v<int&&, short>);

// 寿命延長が行われない（そもそもコンパイルエラーになる）場合
static_assert(std::reference_constructs_from_temporary_v<int&, int> == false);
static_assert(std::reference_converts_from_temporary_v<int&, int> == false);

// 寿命延長が行われない（非参照の）場合
static_assert(std::reference_constructs_from_temporary_v<int, int> == false);
static_assert(std::reference_converts_from_temporary_v<int, int> == false);

// 寿命延長が行われない（uが参照である）場合
static_assert(std::reference_constructs_from_temporary_v<int&&, int&&> == false);
static_assert(std::reference_converts_from_temporary_v<int&&, int&&> == false);
static_assert(std::reference_constructs_from_temporary_v<const int&, int&&> == false);
static_assert(std::reference_converts_from_temporary_v<const int&, int&&> == false);
```

ここでは`T`として`int`を例に使用していますが他のクラス型等でも同じです。ただし、クラス型の場合は変換可能のケースに変換コンストラクタや派生クラスから基底クラスへの変換なども含まれるようになります。

`reference_constructs_from_temporary`と`reference_converts_from_temporary`は名前が異なり扱う初期化形式も異なっているものの、C++23時点では同じ`T, U`に対して同じ結果となるはずです。これは、参照の初期化において`T&& t(u);`と`T&& t = u;`の両方で寿命延長が発生する場合の条件が同じであるためです。直接初期化とコピー初期化の形式は値の初期化あれば`explicit`な変換が考慮されるかが異なりますが、参照の初期化の場合はどちらにおいても考慮されません。

名前が異なっているのはおそらく、将来的に両者の条件が異なる可能性を残しているためと考えられます。あるいは、`reference_constructs_from_temporary`はコンストラクタの文脈で、`reference_converts_from_temporary`は暗黙変換（関数戻り値）の文脈で使用するという使い分けを意識しているのかもしれません。

### 標準ライブラリへの適用

さらに、この新しい2つの型特性を利用して標準ライブラリ内で最初の例のような問題が起こる場所で使用して、ダングリング参照を生成するような変換が行われる場合にコンパイルエラーとして検出するようになります。これは次のものに対して適用されます

- `std::pair`のコンストラクタ
- `std::tuple`のコンストラクタ
- `std::make_from_tuple()`
- `std::invoke_r()`
    - 正確には、標準規定で使用されている`INVOKE<R>`操作に対して適用される

これによって、最初のダングリング参照を生成するような例はコンパイルエラーになるようになります。

```cpp
// ダングリング参照の形成を検出することでコンパイルエラーになる
std::tuple<const std::string&> t1("hello");  // ng
std::function<const std::string&()> f = [] { return ""; };  // ng
```

これは単純化したクラスで再現すると、次のようなことが行われています

```cpp
// std::tupleの単純化クラス
template<typename T>
struct wrap {
  T t;

  // Tに変換可能な型から構築するコンストラクタ
  template<std::convertible_to<T> U>
    requires (std::reference_constructs_from_temporary_v<T, U> == false) // 👈
  wrap(U&& u)
    : t(std::forward<U>(u))
  {}
};

// std::functionの単純化クラス
template<std::invocable F>
struct func {
  using R = const int&;

  F f;

  func(F&& f)
    requires (std::reference_converts_from_temporary_v<R, std::invoke_result_t<F>> == false) // 👈
    : f(std::forward<F>(f))
  {}

  auto operator()() -> R {
    return std::invoke_r<R>(f);
  }
}
```

`wrap`のコンストラクタの場合は保持するメンバの初期化時、すなわち`T t(u);`のような初期化式においてダングリング参照が生成される（参照の寿命延長が起こる）かを検出したいため`reference_constructs_from_temporary`を使用しています。

`func`のコンストラクタの場合はラップする関数の呼び出し時の戻り値型変換時、すなわち`R r = f();`の様な初期化式においてダングリング参照が生成されるかを検出したいため`reference_converts_from_temporary`を使用しています。

なお、これらの型特性の実装はユーザーコードでは再現できず、コンパイラのサポートが必要です。

## `std::is_implicit_lifetime`

C++20では*implicit-lifetime type*と呼ばれる型のカテゴリが導入されましたが、それは特定の一部の型というわけではなく、例えばユーザー定義型も含まれます。

*implicit-lifetime type*ではない型に対しては配置`new`を使用して明示的に生存期間を開始する必要があるなど、プログラマが*implicit-lifetime type*を識別できる必要がありますがその手段は提供されていませんでした。

他にも、自作のクラス型が当初は*implicit-lifetime type*だったものの変更の過程でその性質が失われた場合、静かに未定義動作を引き起こすことになります。これを静的に検査できればそのようなバグをコンパイルエラーとして報告することができます。

`std::is_implicit_lifetime<T>`は、このために導入される*implicit-lifetime type*を識別するための型特性です。

```cpp{style=cppstddecl}
namespace std {
  template<class T>
  struct is_implicit_lifetime;

  template<class T>
  inline constexpr bool is_implicit_lifetime_v = is_implicit_lifetime<T>::value;
}
```

\clearpage

C++23時点で、*implicit-lifetime type*に該当する型は次のいずれかのものです

- スカラ型
- *implicit-lifetime class types*
    - ユーザー定義デストラクタを持たない集成体型
    - もしくは、少なくとも一つの資格のある（*eligible*）トリビアルなコンストラクタを持ち、かつ削除されていないトリビアルなデストラクタを持つ型
- 配列型（要素型は問わない）
- 上記の型のCV修飾付きの型

特に、*implicit-lifetime class types*の「ユーザー定義デストラクタを持たない集成体型」という条件において、デストラクタがユーザー定義されているかどうかをユーザーコードで検出する方法が無いため、これをユーザーコードで再現することはできません。このことも、この型特性が必要になる理由の一つです。

```cpp
struct A { int x; };
struct B { ~B() {} }; // implicit-lifetime typeではない
union C { int x; double y; };
struct D {
  D() = default;  // 資格のあるトリビアルなコンストラクタ
  ~D() = default; // 削除されていないトリビアルなデストラクタ
  int x;
};

// スカラ型と配列型
static_assert(std::is_implicit_lifetime_v<int>);
static_assert(std::is_implicit_lifetime_v<std::string[10]>);

// クラス型
static_assert(std::is_implicit_lifetime_v<std::string> == false);
static_assert(std::is_implicit_lifetime_v<A>);
static_assert(std::is_implicit_lifetime_v<B> == false);
static_assert(std::is_implicit_lifetime_v<C>);
static_assert(std::is_implicit_lifetime_v<D>);
```

## `std::reference_wrapper<T>`と`T&`の間の`common_reference`を`T&`にする

`std::common_reference`はC++20で追加された2つの型の間の共通の参照型を求める型特性です。

`std::reference_wrapper<T>`は`T`の参照となるクラス型で、ほぼ`T&`と同様の働きをし、相互に変換することができます。

```cpp
int i = 1;
std::reference_wrapper<int> jr = j; // ok、暗黙変換
int& ir = std::ref(i);              // ok、暗黙変換

int j = 2;
int& r = false ? i : std::ref(j); // ng、型が一つに定まらない
```

ただ、2つの型が相互変換可能であるために条件演算子では型を1つに絞ることができずにエラーとなります。

`std::common_reference`では、`std::basic_common_reference`が特殊化されていない場合にこの例の最後の条件演算子の結果型としてそれを求めようとし、それもだめなら`std::common_type`に頼ります。

そのため、`std::common_reference_t<T&, std::reference_wrapper<T>>`の結果は、`std::common_type_t<T&, std::reference_wrapper<T>>`と同じになり、この型は`T`となります。

```cpp
static_assert(
  std::same_as<
    std::common_reference_t<int&, std::reference_wrapper<int>>,
    int
  >
);  // ✅
```

C++23ではこれが修正され、`std::common_reference_t<T&, std::reference_wrapper<T>>`の結果が`T&`になるように変更されます。

```cpp
static_assert(
  std::same_as<
    std::common_reference_t<int&, std::reference_wrapper<int>>,
    int&
  >
);  // ✅
```

また、`std::reference_wrapper`に対するCVと参照修飾も正しく扱えるようになります。

```cpp
static_assert(
  std::same_as<
    std::common_reference_t<int&, std::reference_wrapper<int>&>,
    int&  // 以前はint
  >
);  // ✅

static_assert(
  std::same_as<
    std::common_reference_t<int&, const std::reference_wrapper<int>&>,
    int&  // 以前はconst int&
  >
);  // ✅
```

標準ライブラリ中でも`<ranges>`などで`std::common_reference`が使用されているため、この変更によってそれらの振る舞いも改善されます（コピーや一時オブジェクト生成が参照になるなど）。

## 比較コンセプトのムーブオンリー型対応

`std::equality_comparable_with`を始めとする比較可能性を定義するコンセプトは、その定義内で`std::common_reference`を用いて共通参照型を求めて、そのうえでの比較の成立を要求しているところがあります。例えば、C++23時点では`std::equality_comparable_with`は次のように定義されていました

```cpp{style=cppstddecl}
namespace std {
  template<class T, class U>
  concept equality_comparable_with =
    equality_comparable<T> && equality_comparable<U> &&
    common_reference_with<
      const remove_reference_t<T>&, 
      const remove_reference_t<U>&
    > &&
    equality_comparable<
      common_reference_t<
        const remove_reference_t<T>&,
        const remove_reference_t<U>&>> &&
    weakly-equality-comparable-with<T, U>;
}
```

このうち、`common_reference_with`によって`T, U`の共通参照型が存在することを要求しており、その後の`common_reference_t<const T&, const U&>`に対する`equality_comparable`によって共通参照型の上での同値比較可能性を要求しています。

`common_reference_with<T, U>`で表現される共通参照型の定義においては、`T, U`はその共通参照型に変換可能であることを要求しており、これが`T`あるいは`U`がムーブオンリー型である場合に問題となります。

```cpp
// Tが何でもあったとしても
static_assert(std::equality_comparable_with<std::unique_ptr<T>, std::nullptr_t>); // ng

int main() {
  std::unique_ptr<int> p;
  p == nullptr; // ok、比較は可能であるはず
}
```

`common_reference_with<T, U>`は次のように定義されます

```cpp{style=cppstddecl}
namespace std {
  template<class T, class U>
  concept common_reference_with =
    same_as<common_reference_t<T, U>, common_reference_t<U, T>> &&
    convertible_to<T, common_reference_t<T, U>> &&
    convertible_to<U, common_reference_t<T, U>>;
}
```

問題になるのは最後にある`convertible_to`の制約です。

`std::equality_comparable_with<std::unique_ptr<T>, std::nullptr_t>`の場合、`std::common_reference_t<const std::unique_ptr<T>&, const std::nullptr_t&>`は`std::unique_ptr<T>`となり、`std::convertible_to<const std::unique_ptr<T>&, std::unique_ptr<T>>`が要求されますが、これは`std::unique_ptr`に`copyable`であることを要求することと同じであり、`std::unique_ptr`はムーブオンリー型なので満たされません。

このため、`equality_comparable_with`中の`common_reference_with`の制約が満たされず、`std::equality_comparable_with<std::unique_ptr<T>, std::nullptr_t>`は常に満たされなくなります。

これと同様のことが他の異種型間比較のコンセプトでも起こっており、次のものが該当します

- `std::three_way_comparable_with`
- `std::equality_comparable_with`
- `std::totally_ordered_with`

そもそもこのような共通参照型の上での比較可能性の要求は、`equality_comparable_with<T, U>`のように異なる型`T, U`の間の比較を定義するコンセプトにおいて2つの異なる型の間での比較（二項関係）を数学的に考えた時に、その2つの型を包含するような型`V`が存在してその上での二項関係が定義されている必要がある、という考え方によるものです。この型`V`を定義するのが`common_reference_t<T, U>`であり、`common_reference_with<T, U>`はそのような型`V`が存在することを指定します。

これは実行時にそのような変換が行われるわけではなく、数学的な比較の定義をC++のコンセプト定義にエンコードする際に`common_reference`を使用しているだけです。そのため、`common_reference_with`を使用していることに実利的な意味があるわけではありません。また、その過程で`const`参照を使用していることにも大きな意味はありません。

C++23では、これらの異種型比較のコンセプトの`common_reference_with`要件がムーブオンリー型に対応できる別の説明専用コンセプトに置き換えられることで、ムーブオンリー型で動作しない問題が修正されます。ただし、共通参照型の上での比較可能性の要求とその背景にある数学的な考察についての意図には変化がありません。

具体的にはまず、`common_reference_with`の代わりになる`comparison-common-type-with`という説明専用のコンセプトを導入します

```cpp{style=cppstddecl}
namespace std {
  template<
    class T,
    class U,
    class C = common_reference_t<const T&, const U&>
  >
  concept comparison-common-type-with-impl =
    same_as<
      common_reference_t<const T&, const U&>,
      common_reference_t<const U&, const T&>
    > &&
    requires {
      requires
        convertible_to<const T&, const C&> ||
        convertible_to<T, const C&>;
      requires
        convertible_to<const U&, const C&> ||
        convertible_to<U, const C&>;
    };

  template<class T, class U>
  concept comparison-common-type-with = 
    comparison-common-type-with-impl<remove_cvref_t<T>, remove_cvref_t<U>>;
}
```

非常に複雑ですが、先ほどの`common_reference_with`の定義をベースにして、`convertible_to`の部分を`T`や`U`がムーブオンリー型である場合にも対応できるように変更したものです。比較系コンセプトでのみ使用するものであるため、そこで行われている`const`参照化の処理もここで行われています。

そして、`std::equality_comparable_with`などの異種型比較のコンセプトにおいて、`common_reference_with`の部分をこの新しい`comparison-common-type-with`に置き換えます。

```cpp{style=cppstddecl}
namespace std {
  template<class T, class U>
  concept equality_comparable_with =
    equality_comparable<T> && equality_comparable<U> &&
    comparison-common-type-with<T, U> &&  // 👈
    equality_comparable<
      common_reference_t<
        const remove_reference_t<T>&,
        const remove_reference_t<U>&>> &&
    weakly-equality-comparable-with<T, U>;
}
```

これらの変更によって、`std::unique_ptr`における比較可能性を正しく表現できるようになります。

```cpp
// Tが何でもあったとしても
static_assert(std::equality_comparable_with<std::unique_ptr<T>, std::nullptr_t>); // ok
```

また同時に、異種型間比較系コンセプトの意味論要件についても同様にムーブオンリー型を考慮するように調整が行われています。とはいえ、意味論要件が大きく変更されるわけではなく、共通参照型の上での比較に関連するところでコピーではなくムーブも考慮するようになるだけです。

\clearpage

# その他のヘッダやカテゴリ

## `std::byteswap()`

※この項目は「C++20 ライブラリ機能1」に早まって掲載されていたものとほぼ同一です。`std::byteswap`はC++20ではなく23から利用可能になります。

バイトオーダーとは値のバイト配列をメモリ上に保存する際の順序の事で、エンディアンとも呼ばれます。大別すると、ビッグエンディアンとリトルエンディアンの2つがあります。

エンディアンは値（オブジェクト）をバイト列として読み出すときに問題となるほか、ネットワークバイトオーダーはビッグエンディアンである一方でx86 CPUはリトルエンディアンであることから、ネットワーク越しに送受信したバイト列を読み替える（*punning*する）際に問題となったりします。

リトルエンディアンとビッグエンディアンはバイトスワップという操作によって簡単に相互変換することができますが、これまでのC++にはそれを行うための標準的な方法がありませんでした（一応POSIX規格に`htonl()/ntohl()`という関数はありました）。

C++23からはそのために、`std::byteswap()`が用意されます。この`std::byteswap()`の入力は、符号付きも含めた整数型を使用可能です。

```cpp{style=cppstddecl}
namespace std {
  template<std::integral T>
  constexpr T byteswap(T value) noexcept;
}
```

```cpp
#include <bit>

int main() {
  // プログラマが扱うのはビッグエンディアン
  constexpr std::uint32_t n = 0x1234ABCD;

  // リトルエンディアンに変換
  std::uint32_t lite = std::byteswap(n);
  // ビッグエンディアンに変換
  std::uint32_t bige = std::byteswap(lite);

  std::println("{:#X}", lite);
  std::println("{:#X}", bige);
}
```
```{style=planetext}
0XCDAB3412
0X1234ABCD
```

バイトスワップは、バイト列の真ん中（全体の長さが偶数の場合はバイト境界、奇数の場合は中心バイト）を中心とした点対称のような交換によってバイト列の並べ替えを行います。

ややこしい点として、先ほどのサンプルコードはx86 CPU等リトルエンディアンのCPUで動かしたとき、想定（コメントの記述）とメモリ上の実際の配置は逆になります。例えば、最初の`n`はメモリ上ではリトルエンディアンで格納されており、それを`byteswap()`するとメモリ上の配置としてはビッグエンディアンに見えるようになりますが、CPUはいつもリトルエンディアンとして読み出すため、その値（のバイト表現）はまた実際のメモリ上の配置とは逆になります。

```cpp
#include <bit>

int main() {
  // 0x1234ABCDをビッグエンディアンで配置
  std::array<std::uint8_t, 4> bytes{0x12, 0x34, 0xAB, 0xCD};

  auto n = std::bit_cast<std::uint32_t>(bytes);
  // メモリ配置は|12|34|AB|CD|
  // 値は0xCDAB3412

  std::uint32_t lite = std::byteswap(n);
  // メモリ配置は|CD|AB|34|12|
  // 値は0x1234ABCD

  std::uint32_t bige = std::byteswap(lite);
  // メモリ配置は|12|34|AB|CD|
  // 値は0xCDAB3412

  std::println("{:#X}", lite);
  std::println("{:#X}", bige);
}
```
```{style=planetext}
0X1234ABCD
0XCDAB3412
```

すなわち、この関数はメモリ上の配置としてバイトスワップを行うのではなく、読み出した値のバイト表現についてバイトスワップを行います（そしてその上で再度環境のエンディアンによってメモリに格納されます）。とはいえ、リトルエンディアンとビッグエンディアンは相互変換可能であることもあり、多くの場合にこのことを意識する必要はないはずです。

## `std::visit()`の制限緩和

`std::visit`は引数列の最後に受け取る`std::variant`をテンプレートで受け取っています。

```cpp{style=cppstddecl}
namespace std {
  // visit()の宣言例（戻り値型指定をする方
  template<typename R, typename Visitor, typename... Variants>
  constexpr R visit(Visitor&& vis, Variants&&... vars);
}
```

可変長引数で受けているのは複数の`std::variant`を同時に扱うことができるためですが、この`vars`に渡すことのできる型としては`std::variant`の特殊化のみでした。

C++23からはそれが少し緩和され、`std::variant`（の任意の特殊化）を曖昧でない`public`な基底クラスとして持つ型でも呼べるようになります。

```cpp
// variantによってステートマシンを表現する例
struct State : public std::variant<Disconnected, Connecting, Connected> {
  using std::variant::variant;
  
  // variantにはないこの用途での専用APIを追加する
  bool is_connected() const {
    return std::holds_alternative<Connected>(*this);
  }
  
  friend std::ostream& operator<<(std::ostream&, const State&) {
    ...
  }
};

int main() {
  State state = ...;

  if (state.is_connected()) {
    ...
  }

  // 状態に応じた処理にvisit()を使用
  std::visit(overloaded {
      [](const Disconnected& s) { ... },
      [](const Connecting& s) { ... },
      [](const Connected& s) { ... },
    }, state);  // ok、State型でもvisit()が使える
}
```

## `std::optional`のモナドインターフェース

「C++23 ライブラリ機能1」で紹介した`std::expected`はモナドインターフェースを持っていましたが、`std::optional`にもほぼ同じものが導入されます。ただし`std::optional`の無効値が`std::nullopt`固定という事情もあり、一部だけかつ微妙に違いがあります。

まず、`std::optional`のモナドインターフェースとして導入されるのは次の3つです

- `.and_then(f)`: `std::optional<T>` -> `std::optional<U>`
    - `f`: `T` -> `std::optional<U>`
- `.or_else(f)`: `std::optional<T>` -> `std::optional<T>`
    - `f`: `void` -> `std::optional<U>`
- `.transform(f)`: `std::optional<T>` -> `std::optional<U>`
    - `f`: `T` -> `U`

`.and_then()`と`.transform()`は`std::expected`のそれと同じものとなりますが、`.or_else()`だけが少し異なっています。

`.or_else()`に渡す関数は引数として`std::nullopt_t`を受け取るのではなく何も受け取らず、戻り値は任意の`std::optional<U>`ではなく呼び出した`std::optional<T>`と同じ型の`std::optional<T>`を返す必要があります（その有効状態を切り替えることはできます）。

そのほか動作や役割などは`std:expected`のものと共通しています（詳しい説明は「C++23 ライブラリ機能1」もご覧ください）。

`.and_then()`のサンプルコード

```cpp
// ファイルを開く
auto attempt_to_open(std::filesystem::path file)
  -> std::optional<std::ifstream>;

// ファイルの内容を読み込む
auto extract_content(std::ifstream ifs)
  -> std::optional<std::string>;

void example() {
  // ファイルを開いて内容を出力してから返す
  auto res = attempt_to_open("file.txt")
                .and_then(extract_content)
                .and_then([](auto file_str) {
                   std::println("file content: {:s}", file_str);
                   return std::optional<std::string>{std::move(file_str)};
                 });

  ...
}
```

`attempt_to_open()`か`extract_content()`が失敗によって無効値を保持する`optional`を返した場合、`res`は無効状態になります。

`.or_else()`のサンプルコード

```cpp
// ファイルを開く
auto attempt_to_open(std::filesystem::path file)
  -> std::optional<std::ifstream>;

// ファイルの内容を読み込む
auto extract_content(std::ifstream ifs)
  -> std::optional<std::string>;

// ファイルオープンエラーをハンドリング、
auto on_error() -> std::optional<std::string> {
  // 空文字のstringに変換
  return { "" };
}

void example() {
  // ファイルを開いてエラーを処理する
  auto res = attempt_to_open("file.txt")
                .and_then(extract_content)
                .or_else(on_error);

  assert(res.has_value());  // ✅

  ...
}
```

この場合、`.or_else(on_error)`は前の2つの処理が失敗（無効値）を返した場合にのみ呼ばれて、エラー状態を空文字列に変換することで最終的な`res`は常に有効状態となります。

`.transform()`のサンプルコード

```cpp
// ファイルを開く
auto attempt_to_open(std::filesystem::path file)
  -> std::optional<std::ifstream>;

// ファイルの内容を読み込む
auto extract_content(std::ifstream ifs) -> std::string;

void example() {
  // ファイルを開いて内容を出力してから返す
  auto res = attempt_to_open("file.txt")
                .transform(extract_content)
                .transform([](auto file_str) {
                  std::println("file content: {:s}", file_str);
                  return file_str;
                 });

  ...
}
```

`.transform()`に渡す処理は`optional`が有効状態の場合のみ呼ばれるため、この場合の`res`は`attempt_to_open()`の結果によって無効状態になりえます。`.transform()`は`optional`の有効状態を変更できないため、この場合に`extract_content`のエラーを`optional`の無効値という形で報告することはできません。

なおこちらの`.transform()`は渡す関数の戻り値型を`void`にすることはできません。

この3つの関数はいずれも`std::optional`そのものを返すため、メソッドチェーンの形で組み合わせて使用できます。

```cpp
// ファイルを開く
auto attempt_to_open(std::filesystem::path file)
  -> std::optional<std::ifstream>;

// ファイルオープンエラーをハンドリング
auto attempt_recovery()
  -> std::optional<std::ifstream>;

// ファイルの内容を読み込む
auto extract_content(std::ifstream ifs)
  -> std::string;

// 読み取った内容を検証
auto ensure_valid_content(std::string file_data)
  -> std::optional<std::string>;


void example() {
  // ファイルを開いて内容を読み取って返す
  auto res = attempt_to_open("file.txt")
                .or_else(attempt_recovery)
                .transform(extract_content)
                .and_then(ensure_valid_content);

  ...
}
```

## `std::barrier`の同期保証の調整

`std::barrier`はFork-Joinモデルのような並行処理の実装に活用することができる繰り返し使用可能な同期機構です。典型的には、同期に参加する全スレッドが並列実行される部分と一つのスレッドだけで実行される同期部分から構成される処理単位の繰り返しのような処理になり、その繰り返しの中の1つの処理単位のことをバリアフェーズと呼びます。

`std::barrier`はテンプレート引数として完了関数（`CompletionFunction`）の型を受け取って、そのオブジェクトを保持しておくことでバリアフェーズの最後にどこか一つのスレッドでそれを実行してから、待機しているスレッドを再開します。

```cpp
// 複数のスレッドで呼ばれる処理
template<typename CF>
void fork_proc(std::barrier<CF>& sync) {
  // キャンセルされるまで行われる連続処理
  // ループごとに全スレッドで同期しつつ実行される
  while(/*キャンセルの検出*/) {

    // メインの処理
    ...

    // 全スレッドはここで待ち合わせる（1ループの処理の完了を同期する）
    // バリアフェーズに参加する全てのスレッドがここに到達した時、CFの処理を実行してから次のバリアフェーズを開始する
    sync.arrive_and_wait(); // 同期ポイント
  }
}

int main() {
  // 並列数
  constexpr std::size_t N = 10;

  // 完了関数
  // バリアフェーズの最後にどこか1スレッドが実行する
  auto completion_function = [] {
    // 処理対象データの更新など、同期が必要なシングルスレッド処理
    ...
  };

  // スレッド数でバリアを初期化
  // 同時に、同期ポイントで再開直前に実行する完了関数を指定
  std::barrier sync{N, completion_function};

  for ([[maybe_unused]] auto i : std::views::iota(0, N)) {
    std::thread{[&sync]{
      fork_proc(sync);
    }}.detach();
  }

  // 完了を待機する処理など
  ...
}
```

バリアで同期する（バリアフェーズに参加する）各スレッドは`.arive()`（同期ポイント到達通知）と`.wait()`（他スレッドの待機）もしくは`.arrive_and_wait()`（`.arive()`と`.wait()`の複合操作）を呼び出すことで同期を取ります。完了関数は、バリアフェーズに参加しているスレッドが全て同期ポイントに到達した後、バリアフェーズに参加するスレッドのいずれかで実行され、その実行が完了した後で次のバリアフェーズが開始されます。

C++20の標準の規定では、この完了関数がどのスレッドで実行されるかは実装の自由として明確に規定していません。しかし、`std::barrier`の保証やメンバ関数の規定などの相互作用から生じる（意図しない）制約によって、実質的に同期ポイントに最後に到達したスレッドで実行する実装しか取れないようになっていました。それによって、ハードウェア（GPUなど）が持っているスレッド同期機構やそのサポート機構を用いて`std::barrier`を効率的に実装することが妨げられていました。

C++23では、この意図しない制限が取り払われたことで、`std::barrier`の効率的な実装が可能になります。

これは厳密には振る舞いの変更であり破壊的変更となりますが、元の規定による振る舞いが多くの場合に意外なものであったことと、C++20で追加されたばかりだったことなどから、破壊的変更の影響は小さいと判断されました。

### C++20の意図しない制約と変更の詳細

C++20時点の意図しない制限というのは、`std::barrier`の`.arive()`（同期ポイント到達通知）と`.wait()`（他スレッドの待機）の操作が分かれている事に原因があります。`.arive()`は戻り値として`arrival_token`というものを返し、`.wait()`はその`arrival_token`を受け取って待機します。この2つの操作が分割されていることによって、バリア同期ポイント通知とバリア同期の待機を異なるスレッドで行うことができるようになっています。

これによって、恣意的ではあるものの次のようなことが起こります

```cpp
// 参加スレッド数2で初期化、完了関数cfをセット
std::barrier<CF> b{2, cf};

// arrival_tokenの型
using tok_t = decltype(b.arrive());

void thread() {
  new tok_t(b.arrive());  // A: 同期ポイント到達を通知するが、返されるトークンを消費しない（リークしてる
}                         // B: スレッド終了

// C: 2つのスレッドを起動
auto t0 = std::thread(thread);
auto t1 = std::thread(thread);

// D: 2つのスレッド終了待機
t0.join();                      
t1.join();

// E: 完了関数cfが呼ばれていることが保証される
```

この例では、2つの`arrival_token`が有効であり続けながら`std::barrier`の`.wait()`は呼ばれておらず、バリアフェーズに参加している（完了を待機している）スレッドが居なくなっています。現在の標準の規定はこの時でも完了関数が呼ばれていることを保証しているため、このような極端な場合でもそれを確実に達成するためには、最後に`.arive()`したスレッドで完了関数を実行するという実装を取らざるを得ません。

これは、C++20の`std::barrier`仕様が保証していた次の2つのことの相互作用の結果です

- スレッドが`.wait()`を呼び出さなくてもいい自由
- 完了関数（バリアフェーズ完了ステップ）は、バリアに参加しているすべてのスレッドが同期ポイントに到達した時に常に実行される

数百万のHWスレッドを持つアーキテクチャでは、わずかな並列化されていない処理によって大きく並列化のスケーラビリティが制限されます（アムダールの法則）。`.arive()`と`.wait()`を分割しているAPIはそのオーバーヘッドを削減するためのもので、完了関数が実行するシングルスレッド処理と`std::barrier`の同期の処理を並列化し、同期をさらに別のスレッドやHWアクセラレータで実行することで同期のコストを完了関数の処理の背後に隠蔽することを意図したものでした。しかし、上記の問題によってそのような実装は実質的に許可されていませんでした。

C++23では、`std::barrier`の保証を次のように変更することでこの問題が解決されています

- 累積性は以下の2つの場所で確立される
    - バリアフェーズに参加しているすべてのスレッドと完了関数を実行するスレッドの間
    - 完了関数を実行するスレッドと`.wait()`の呼び出しによってバリアフェーズ完了を待機しているすべてのスレッドの間
- 完了関数は、バリアに参加しているすべてのスレッドが同期ポイントに到達した後、ちょうど一度だけ、次のバリアフェーズ開始前に実行される
- 完了関数は、そのバリアフェーズに参加している（参加していた）スレッドの一つで実行される
- バリアフェーズ完了を待機するスレッドが無い場合（だれも`.wait()`を呼んでいない場合）、完了関数が実行されるかは実装定義

累積性（*Cumulativity*）とは、2つの処理（A, B）を接続するある地点（A -> C -> BのC）について、AがCに到達してからBが実行されるという順序関係を言うもので、特にマルチスレッド処理の場合にはAの並行処理がすべて終わってから（各スレッドの完了がCに累積してから）、Bの処理開始が実行（累積）されることを表現しています。

すなわち、この累積性の規定によって、あるバリアフェーズの終了によって実行される完了関数からは、そのバリアフェーズで行われた任意のスレッドの変更が確実に同期されており（可視になっており）、またその完了関数の終了によって待機が解除されて次のバリアフェーズを開始するスレッドからは完了関数で行われた変更が確実に同期（可視になる）されています。

バリアフェーズA -> 完了関数 -> バリアフェーズB、のような処理の流れの中で、Aで行われた変更は競合等なく完了関数から安全に利用可能になり、完了関数で行われた変更も競合等なく安全にBの開始時点で利用可能になる、ということを言っています。これは`std::barrier`のセマンティクスからすると当然提供されるべき保証であり、（分かりづらくはあるものの）特に驚きの無い当然の保証です。

そして、完了関数の実行は最後に同期ポイントに到達したスレッドではなく、そのバリアフェーズに参加して居た任意のスレッドで実行することができるようになり、次のバリアフェーズに参加するスレッドがない（だれも`.wait()`を呼んでいない）場合に必ずしも完了関数を実行しなくてもよくなっています。

このような変更によって、完了関数の実行場所の自由度が向上し、元来意図されていたようにHWアクセラレータを活用することもできるようになっています。

## `<chrono>`

### `time_point`の時計型の要件緩和

`std::chrono::time_point`は時間軸上の一点を指定する型で、その時間基準（始点（*epoch*）と分解能）を指定するために時計型と時間間隔を表す2つのテンプレートパラメータを受け取ります。

```cpp{style=cppstddecl}
namespace std::chrono {
  template <class Clock, class Duration = typename Clock::duration>
  class time_point;
}
```

うち1つ目の時計型`Clock`には*Cpp17Clock*要件という要求がなされています。これは、`std::chrono::system_clock`や`std::chrono::steady_clock`などと同じ扱いができることを要求しています。

一方、C++20ではローカル時間を表す型として`std::chrono::local_t`が導入されました。これは、`std::chrono::time_zone`とともに用いることで任意のタイムゾーンにおけるローカルの時刻を表現することができるものです。

`local_t`に対する特殊化`std::chrono::time_point<local_t, Duration>`も提供されますが、`local_t`は空のクラスであり*Cpp17Clock*要件を満たしていません。規格では`local_t`だけを特別扱いする事で`time_point`を特殊化することを許可しています。

この`local_t`のように、時計型ではないが任意の時間を表す型を定義し、その時間表現として`time_point`を特殊化することはユーザーコードでも有用ではあったものの、C++20では*Cpp17Clock*要件を満たすように実装する必要があり、かなり煩雑かつ制限がありました（`now()`が`static`メンバ関数でないとならないなど）。

C++23ではこの制限が緩和され、`std::chrono::time_point`の`Clock`テンプレートパラメータ（1つ目の引数）としてユーザーが提供する任意の型を使用できるようになります。

この提案文書では、次のようなユースケースが紹介されています

- 状態を持つ時計型を扱いたい場合（非`static`な`now()`を提供したい）
- 異なる`time_point`で1日の時間を表現したい場合
    - 年月日のない24時間のタイムスタンプで表現される時刻を扱う
- 手を出せないシステムが使用している`time_point`を再現したい場合
- 異なるコンピュータ間のタイムスタンプを比較・処理する場合

この変更がなされたとしても既存のライブラリ機能とそれを使ったコードに影響はなく、`local_t`が既にあることから実装にも影響はありません。

### リテラルエンコーディングとロケールエンコーディングが異なっている場合の`chrono`型のフォーマット結果の明確化

C++20で追加された`chrono`型のフォーマット対応において、ロケールオプションが指定されている場合にリテラルエンコーディング（文字列リテラルのエンコーディング）とロケールで指定されたエンコーディングの間に不一致がある場合の振る舞いが規定されていませんでした。

```cpp
std::locale::global(std::locale("Russian.1251"));
auto s = std::format("День недели: {:L}", std::chrono::Monday);

// 出力例（リテラルエンコーディングがUTF-8の場合）
// "День недели: \xcf\xed"
```

この例では、リテラルエンコーディング（文字列リテラルのエンコーディング）がUTF-8の場合、グローバルロケールに指定されている`Russian.1251`エンコーディングとの間に不一致があります。

C++23ではこの場合の振る舞いが規定され、文字列リテラルのエンコーディングがユニコードでありロケールの指定するエンコーディングと異なる場合、ロケールによる文字列置換結果は、文字列リテラルのエンコーディング（すなわちユニコード）に変換されて出力される、ようになります。

```cpp
std::locale::global(std::locale("Russian.1251"));
auto s = std::format("День недели: {:L}", std::chrono::Monday);

// 出力（リテラルエンコーディングがユニコードの場合）
// "День недели: Пн"
```

なお、このことは`std::print()`などでも同様に適用されます。

\clearpage

# 大域的な変更

ここでは、大きくはないものの同種の変更がより広いライブラリ機能にわたるものについて簡単にまとめておきます。その性質上、個別に列挙する場合は抜けがあるかもしれませんがご容赦ください。

## constexpr対応

次の関数はC++23から`constexpr`指定されるようになり、定数式で使用可能となります。

- `std::unique_ptr`のメンバ関数と関連関数
    - 全メンバ関数の`constexpr`対応
    - `nullptr`との比較ではない順序付け比較演算子（`<`など）は除く
    - ハッシュサポートは除く
- `std::bitset`のメンバ関数と関連関数
    - `<<`は除く
    - ハッシュサポートは除く
- `<cmath>`/`<cstdllib>`の数学関数の一部
    - `<cstdllib>`の`abs`/`div`系関数
    - `<cmath>`では三角関数・双曲線関数・指数/対数関数・平方根などの主要な関数は対象外
- `std::type_info::operator==`
- `std::to_chars`/`std::form_chars`の整数型用オーバーロード
- `std::optional, std::variant`の一部の関数
    - これによって、`std::optional, std::variant`は全てのメンバ関数が`constexpr`対応した

## 非推奨化・削除

次のものは、C++23で非推奨となりました。削除されるまでは使用することができますが、なるべく早期に使用を辞めることが推奨されます。

- `std::aligned_storage`と`std::aligned_union`
    - 正しく使用するのが難しく、間違って使用するのが簡単だったため
- `std::std::allocator::is_always_equal`
    - アロケータの状態の有無を表現する型だったが、定義を忘れることによるバグが多かったため
- `std::numeric_limits::has_denorm`関連
    - `std::numeric_limits::has_denorm_loss`
    - `std::numeric_limits::has_denorm`
    - `std::float_denorm_style`
    - 非正規化数のサポート状況はコンパイル時定数で表現可能な性質ではなく、有効に使用することができなかったため

次のものは、C++23で削除されたものです。非推奨と異なり、削除されたものは使用できなくなります。

- ガベージコレクション関連API
    - `std::declare_reachable`
    - `std::undeclare_reachable`
    - `std::declare_no_pointers`
    - `std::undeclare_no_pointers`
    - `std::get_pointer_safety`
    - `std::pointer_safety`
    - `__STDCPP_STRICT_POINTER_SAFETY__`
    - GCサポートのために役に立っていなかったため

## 非推奨化の解除

次のものは、以前に非推奨だったもののC++23で解除されました。

- Cヘッダ
  - `<cxxx>`に対する`xxx.h`
  - Cの標準ライブラリヘッダ

## フリースタンディング対応

次のものは、C++23からフリースタンディングライブラリ機能であることが指定されるようになります。フリースタンディング機能は、組み込み環境などOSが無い環境で動くプログラムコードにおいても使用できるものです。

- ヘッダの全体
    - `<cstddef>`
    - `<version>`
    - `<limits>`
    - `<climits>`
    - `<cfloat>`
    - `<cstdint>`
    - `<new>`
    - `<typeinfo>`
    - `<source_location>`
    - `<exception>`
    - `<initializer_list>`
    - `<compare>`
    - `<coroutine>`
    - `<cstdarg>`
    - `<concepts>`
    - `<utility>`
    - `<tuple>`
    - `<type_traits>`
    - `<ratio>`
    - `<bit>`
- ヘッダ内の一部の機能
    - `<cstdlib>`
    - `<memory>`
    - `<functional>`
    - `<iterator>`
    - `<ranges>`
    - `<atomic>`

フリースタンディング機能の性質上、動的メモリ確保やスレッドサポート、iostreamに関するものは除かれています（ただし`new/delete`は含まれます）。

\clearpage

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- Compiler Explorer(https://godbolt.org/)
- yohhoyの日記(https://yohhoy.hatenadiary.jp/)

\clearpage
