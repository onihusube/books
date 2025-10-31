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
