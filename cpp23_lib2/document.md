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

### string_viewとspanをトリビアルコピー可能と規定

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2251r1.pdf

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

### `noreplace`オープンモード

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
