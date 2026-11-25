---
title: C++26 リフレクション
author: onihusube
date: 2026/12/30
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

## リフレクションとは

リフレクションとは、プログラムがその実行時に自分自身についての情報を取得して、それを基に動作することができる機能の事です。特に、自分自身のソースコードやその意味解析後（AST）の情報を読み取り、それを基にソースコードやプログラム要素そのものをプログラミングすることができるようになります。

リフレクションは、動的なコード生成、メタプログラミングをサポートする強力な機能です。

現代のC++（C++23まで）ではテンプレートメタプログラミングがその基礎を強力に支えており、通常なら言語サポートを必要とするような機能をライブラリレベルで実装することができるなど、その威力の程はご存じだと思われます。

しかし、テンプレートメタプログラミングにはそれゆえの限界があります

- コンパイル負荷が高い
- 型指向である事
- 言語構造へのクエリ手段が不足している
- アノテーション機能の欠如
- メタデータの取り扱い手段の困難

など。

テンプレートメタプログラミングはテンプレートという本来別の機能のために偶然生じた仕様や実装の抜け穴を利用しているもので、どこかでその壁にぶつかることになります。テンプレートメタプログラミングの文脈内でそれを超えようとすることには、テンプレート機能及び周辺言語機能の複雑化や実装可能性などいくつもの困難があります。

リフレクション機能はその困難を超えるためのもので、メタプログラミングや動的コード生成のための専用の機能です。テンプレートメタプログラミングによる限界を克服し、現代C++のコード内で更に強力なメタプログラミングを可能にします。

## 静的リフレクション

C++26で導入されるリフレクション機能は静的リフレクション機能であり、リフレクションを利活用できるのはコンパイル時になります。対して、JavaやC#などが備えているリフレクション機能は動的リフレクションで、実行時にもリフレクションを利用できます。

動的なリフレクションにおいては、実行時にソースコードやそのASTについての情報を残しておく必要があります。JavaやC#のような中間言語にコンパイルされる言語やJavascriptのようなスクリプト言語においてはその性質上容易に達成できるため、動的リフレクションとしてリフレクション機能を自然に利用することができました。

機械語に直接コンパイルされる言語、特にC++のようにゼロオーバーヘッドを原則とする言語ではそのようなコストは通常受け入れられないため、動的リフレクション機能は利用できないか、外部のコードジェネレーターのようなものを頼ることになります。

一方で、C/C++の場合はマクロやテンプレートといった機能を活用して、コンパイル時にメタプログラミングを行うことがある意味で一般的でした。プリプロセッサメタプログラミングやテンプレートメタプログラミングは黒魔術と揶揄されるほど複雑かつ難解であるものの、通常のプログラミングでは達成できないような事を実現することができました。

静的リフレクションはこの流れをくむもので、コンパイル時にコンパイラから取得できるソースコードとその解析情報を使用してメタプログラミング・コード生成を行うものです。動的リフレクションと完全に同等なことが可能なわけではありませんが、静的リフレクションの実現性や実力についてはプリプロセッサメタプログラミングおよびテンプレートメタプログラミングの実績が証明になっています。

C++26で導入されるのは主にコンパイル時に自身について問い合わせるための機能と、それを用いてソースコードを生成する最小の機能です。C++のリフレクション機能はこれで完成というわけではなく、これを核として発展させていくための最初の一歩となるものです。

これはC++26 Contractsと同様に、最小の機能セットから導入することで言語機能として採用するためにかかる負荷（委員会における議論やレビューのコスト）を抑えることで必要な機能を素早く導入するとともに、段階的に進化させていくことを目指すものです。

## 本書独自の訳語について

C++26 静的リフレクションにおいては、リフレクション（reflection、reflect）という言葉が複数の意味を持って使用されています。直訳するとリフレクション（機能そのもの）、リフレクション演算子（`^^`）、リフレクト式（`^^int`）、リフレクション（値のこと）などになりますが、日本語においては非常に曖昧になります。

そこで、大きく3種類のリフレクションという言葉の用法について区別するために、本書では独自に訳語を定義して使用しています。

1. 機能としてのリフレクション: （静的）リフレクション機能
2. 操作としてのリフレクション: 掬映（きくえい）、掬映操作、掬映演算子
3. 値としてのリフレクション: 映象（えいしょう）、`int`の映象、映象値

まず、掬映という言葉は「掬水月在手」という句から着想を得たものです。これは、中国・唐代の詩人である于良史（うりょうし）の五言律詩『春山夜月（しゅんざんやげつ）』の一節（承句）で、日本語に直すと「水を掬すれば月手に在り、花を弄すれば香衣に満つ」となります。

（https://www.myoshinji.or.jp/tokyo-zen-center/howa/h30_4 より引用）

ここから掬を貰い、月ではなくリフレクション操作としてのイメージ・語感から来る鏡映という言葉と合わせて、掬映としました。

水面に映る月を手元に掬い取って得る有様が、リフレクションという操作に非常にマッチしていると考えています。

映象については、値としてのリフレクション値が鏡像、象、ソースコード構成要素を象るものであることなどと、掬映との対応から映象としました。

## 注意など

本書はC++26をベースとして記述されています。そのため、C++23までに導入されている機能については特に導入バージョンについて触れず、既知のものとして詳しい解説なども行いません。

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

# `^^` - リフレクション演算子

リフレクションの出発点はまずソースコードについての情報をクエリすることにあります。C++26 静的リフレクション機能ではそのために、ソースコードの構成要素をプログラムコードから扱うことのできる中間形式としての値に落とし、ソースコード情報のクエリはこの値に対して統一的に専用のメタ関数（後述）によって行います。

このソースコードの構成要素を表現する値を取得する操作は、`^^`演算子によって行います。

```cpp
constexpr auto refl = ^^int;
```

`^^`演算子は単項演算子であり、対象のソースコード構成要素の前に指定します。`^^`がそのオペランドに対して作用することで、そのソースコード構成要素を表現するものを値として取得します。このように取得した値は、定数式内ではあるものの通常のC++コードから通常の値とほぼ同じように扱うことができます。

リフレクション演算子の結果は対象のものを表現する値を返します。これはリフレクションの対象に関わらず`std::meta::info`という単一の型を持ちます。先ほどの例の`refl`は`int`型を表現する値です。

既に3つのものをリフレクションと呼んでおり紛らわしいのでここで区別のためにも訳語を導入しておきます。

まず、`^^`の事を本書では**掬映（きくえい）**（掬映演算子、掬映操作、掬映）と呼びます。そして、掬映操作によって得られるものの事を**映象（えいしょう）**（`int`の映象、映象の型、映象値）と呼びます。この言葉の選択についての解説は一章を参照してください。

`^^`の演算子としての優先順位は上から3番目で右結合です。C++26での制限として、`^^`のオペランドには一般の式を置くことができず、置けるのは何らかの名前に限定されています。

```cpp
int a = 0;

^^::;                   // グローバル名前空間を指定
^^std::filesystem;      // ネストした名前空間を指定
^^std::template vector; // std::vector（テンプレート）を指定
^^int;                  // 型名を指定
^^a;                    // 変数名を指定
```

なお、掬映演算子のオペランドをかっこで囲むことはできません。

```cpp
^^(int);        // ng
^^(std)::list;  // ng
^^(a == b);     // ng
(^^int);        // ok
```

掬映演算子のオペランドは評価されないオペランド（unevaluated operand）であるため、その式は評価されません。これにより、変数名などを指定する掬映操作においてその値ではなくその名前が示す直接のものの映象が取得されます。

```cpp
void f();
int a = 0;
constexpr int N = 1;

^^a;  // a（id式）の評価結果である0ではなく、aそのものの掬映
^^f;  // f（id式）の評価結果である関数ポインタではなく、fそのものの掬映
^^N;  // 同上
```

名前を指定する式は規格ではid式（id-expression）という式であり、その式の評価結果は変数名や構造化束縛ならそれが示す値になり、関数名なら関数ポインタの値になります。しかし、掬映演算子のオペランドは評価されないため、掬映操作はその値の映象ではなく名前が示すエンティティそのものの映象を取得します。

なお、値の映象は`std::meta::reflect_constant()`や`std::meta::reflect_object()`などのメタ関数を使用することで取得することができます。

その他のタイプの式については構文規則の制限により`^^`のオペランドとしてパースされないことや掬映が禁止されていることなどによって、C++26では指定することができません。

## 掬映式の文法について

掬映式（`^^...`）は文法では`reflect-expression`として定義され、`reflect-expression`は`unary-expression`のうちの一つとして既存文法に追加されます。

これにより、`^^`演算子の優先順位は上から3番目で、前置インクリメント演算子や他の単項演算子（`+`や`*`など）と同じです。C++26時点においては優先度3以上の演算子は次のものです

1. nested-name-specifier
    - スコープ解決演算子: `::`
2. postfix-expression
    - 後置インクリメント・デクリメント演算子: `a++`, `a--`
    - 関数スタイルキャスト: `int()`
    - 関数呼び出し演算子: `f()`
    - 添え字演算子: `a[]`
    - メンバアクセス演算子: `o.m`、`p->m`
3. unary-expression
    - 前置インクリメント・デクリメント演算子: `++a`, `--a`
    - 単項プラス/マイナス演算子: `+a`、`-a`
    - 論理NOT/ビットNOT演算子: `!b`、`~b`
    - Cスタイルキャスト: `(int)a`
    - 間接参照演算子: `*p`
    - アドレス演算子: `&a`
    - リフレクション演算子: `^^a`
    - `sizeof`
    - `co_await`
    - `new`、`new[]`
    - `delete`、`delete[]`

これ以外の演算子は`^^`よりも優先度が低いため、`^^`の後にくる場合は`^^`の方が先に結合します。また、結合は右結合（右から左）であり、優先順位3のもの同士の間ではより右側にあるものが先に結合します。

そして、`reflect-expression`が受理する文法がかなり制限されていることで、優先順位2と3の演算子による式が`^^`の後に続くような記述は禁止されます。

```
reflect-expression:
  ^^ ::
  ^^ reflection-name
  ^^ type-id
  ^^ id-expression
```

この4つの形式のオペランドがそれぞれ何を表すのかは後述しますが、一般の式は含まれずほぼ何らかの名前を指定することしかできません。

この結果、C++26時点の掬映演算子は一般の式に対して直接作用できないようになっています。上記の演算子の中で実際に優先順位が有効なのはスコープ解決演算子のみです。

```cpp
^^S::I;      // ok、オペランドはS::I
^^std::list; // ok、オペランドはstd::list
^^a++;       // ng、^^aに続く++としてパースされる
^^p->m;      // ng、^^pに続く->mとしてパースされる
^^f();       // fが型名なら（関数名ではない）ならok
^^!a;        // ng、受理する文法がない
^^a == b;    // (^^a) == bとしてパースされる（bが映象ならok
^^^^a;       // ng、受理する文法がない
```

前述のように`^^`のオペランドを直接かっこでくくることはできません。

このように、掬映演算子の結合と優先順位はそのほかの演算子におけるそれとは少し異なる挙動を取ります。これは式の掬映を防止するための規則であり、C++26時点では式の掬映に関する仕様を確定させないための措置です（それが何を表しどういう意味を持つべきかは29以降の検討事項）。

掬映式（`reflect-expression`）のパースに関しては最長一致であり、`reflect-expression`を構成しうる可能な限り長い文字列をパースします。

```cpp
^^int() const noexcept; // ok、オペランドは関数型（noexceptまで含める）
^^int && true;          // ng、(^^int&&)に続くtrueとしてパースされる
^^int & true;           // ng、(^^int&)に続くtrueとしてパースされる
^^int &&&& true;        // ng、int&&&&は有効な型ではない
```

また、掬映式のオペランドがテンプレートである時にそれに続いて`<`が出現する場合、その掬映式はテンプレートの特殊化の掬映となる必要があり、そうならない場合はエラーになります。

```cpp
// テンプレート
template<bool>
struct X {};

// 比較演算子を用意する
consteval bool operator<(std::meta::info, X<false>) { return false; }

consteval void g(X<false> xv) {
  ^^X < xv;       // ng、テンプレートをオペランドとする^^の後に<が現われる
  (^^X) < xv;     // ok、^^Xとxvの<比較
  ^^X<true>;      // ok、テンプレート特殊化X<true>の掬映
  ^^X<true> < xv; // ok、^^X<true>とxvの<比較
}
```

これは、C++の構文規則においてテンプレート名に続く`<`を特別扱いしている事との整合性を取り、パースが曖昧にならないようにするための制限です。

次からは、`reflect-expression`の4つの文法それぞれが受理する構文形式毎に、何が掬映されるのかを見ていきます。

## `^^ ::`の形式

この形式の掬映式はグローバル名前空間を表す映象を取得します。

```cpp
constexpr auto r = ^^::;
```

この形式に合致するものはこのほかにありません。

## `^^ reflection-name`の形式

この形式の掬映式は名前空間やクラスにネストした名前に対する映象を取得します。

`reflection-name`は次のような文法として定義されています

```
reflection-name:
  nested-name-specifier(opt) identifier
  nested-name-specifier template identifier
```

`nested-name-specifier`は`::`によってネストしてアクセスされる名前を表し、`identifier`はいわゆる識別子名で変数名やクラス名のような名前を表します。

1つ目の規則において`nested-name-specifier(opt)`（省略可能）となっていることで名前単体（グローバル名前空間指定を省略する場合など）をオペランドとする`^^`もマッチしますが、それは他の形式にもマッチするため、どの形式が採用されるかは構文規則だけでは決まらず名前が表すものによって決定されます。

したがって、`^^a`のような掬映式の効果を決定するためにはまず名前探索を必要とし、その名前探索が何を見つけるかによってその種類ごとに結果が決定されます。文法形式のマッチ自体もこの名前探索の結果によって決まります。

その名前探索において、`using`宣言によって導入されている名前が見つかった場合不適格（エラー）となります。ここでの`using`宣言は現在のスコープに別のスコープの名前を取り込む`using`の使用法のことで、`using namespace`やエイリアス宣言ではありません。

```cpp
using std::string;    // using宣言
using namespace std;  // usingディレクティブ
using I = int;        // エイリアス宣言
```

```cpp
namespace A {
  void f();
}
namespace B {
  using A::f;
}

struct S1 { 
  struct S {};
  void g();
};
struct S2 : S1 { 
  using S1::S;
};

consteval void example() {
  ^^B::f;   // ng、using宣言の掬映
  ^^S2::S;  // ng、using宣言の掬映
  ^^S2::g;  // ok、gはusing宣言によって導入されていない
}
```

また、`using enum`も`using`宣言の一種です。

```cpp
enum class E {
  e1,
  e2,
};

consteval void example() {
  using enum E;

  ^^e1;   // ng、using宣言の掬映
}
```

これにより、C言語のライブラリをC++のライブラリとしている標準ヘッダ（`<cstring>`や`<cstdint>`など）においては、どうやらほとんどの標準ライブラリ実装がCの宣言を`using`宣言によって`std`名前空間に取り込んでいる事によって、そのような宣言に対する掬映がエラーになります。

```cpp
^^std::int32_t; // ng、using宣言の掬映
^^std::memcpy;  // ng、using宣言の掬映
```

本書執筆時点では少なくともlibc++/libstdc++のどちらにおいてもエラーになります。

このような`using`宣言の問題がない場合に掬映式の結果がどうなるかは、そのオペランド（まずは`reflection-name`）の名前に対する名前探索結果で何が見つかるかによって変化します。

### 名前空間エイリアス

名前探索によって名前空間エイリアスが見つかった場合、掬映式の結果はその名前空間エイリアスを表現する映象です。

```cpp
namespace Alias = std;

namespace A {
  namespace C = std::chrono;
}

consteval void example() {
  // どちらも名前空間エイリアスを表現する映象を得る
  auto r1 = ^^Alias;
  auto r2 = ^^A::C;
}
```

エイリアス先の名前空間ではなく、名前空間エイリアスそのものの映象であることに注意してください。

この時、名前に対して複数のエイリアス名が見つかる場合、そのエイリアスの宣言された親のスコープ（ターゲットスコープ）はすべて同一である必要があります。

```cpp
namespace A {
  namespace Alias = std; // ターゲットスコープはnamespace A
}

namespace B {
  namespace Alias = std; // ターゲットスコープはnamespace B
  using namespace A;
}

namespace C {
  namespace Alias = std; // ターゲットスコープはnamespace C
  namespace Alias = std; // ターゲットスコープはnamespace C
}

consteval void example() {
  auto r1 = ^^B::Alias; // ng、異なるスコープで定義された同名のエイリアスの掬映
  auto r2 = ^^C::Alias; // ok、ターゲットスコープは同一
}
```

### 名前空間

名前探索によって名前空間が見つかった場合、掬映式の結果はその名前空間を表現する映象です。

```cpp
namespace A::B::C {
}

consteval void example() {
  // どちらも名前空間を表現する映象を得る
  auto r1 = ^^std;
  auto r2 = ^^A::B::C;
}
```

### コンセプト

名前探索によってコンセプトが見つかった場合、掬映式の結果はそのコンセプトを表現する映象です。

```cpp
#include <concepts>
#include <ranges>

consteval void example() {
  // どちらもコンセプトを表現する映象を得る
  auto r1 = ^^std::integral;
  auto r2 = ^^std::ranges::range;
}
```

型を指定したコンセプト名はここの形式では受理できず（`identifier`で終わる必要があるため）別の形式（`^^ type-id`）で受理されますが、それは型名ではなく式であるため掬映式のオペランドとして不適格になります。

```cpp
consteval void example() {
  auto r = ^^std::integral<int>; // ng、式の掬映
}
```

### テンプレート

名前探索によってテンプレートが見つかった場合、掬映式の結果は次のようになります

1. 名前探索によって注入されたクラス名（`injected-class-name`）が見つかった場合
    1. `reflection-name`が`nested-name-specifier template identifier`の形式ならば、掬映式の結果はその注入されたクラス名が指定するクラステンプレートを表現する映象
    2. それ以外の場合、注入されたクラス名を型名として解釈した時に曖昧でなければ、掬映式の結果はその注入されたクラス名が指定するクラステンプレートの特殊化を表現する映象
2. 名前探索によってオーバーロード集合が見つかった場合
    - そのオーバーロード集合には単一の関数テンプレート`F`の宣言のみが含まれている必要があり、その場合は掬映式の結果は`F`を表現する映象
3. 名前探索によってクラステンプレート/変数テンプレート/エイリアステンプレートのいいずれかが見つかった場合
    - 掬映式の結果はそのテンプレートを表現する映象

なお、名前探索によって部分的特殊化や明示的特殊化が見つかることはありません。

1の注入されたクラス名とは、クラステンプレートの定義内で使用できるテンプレートパラメータを省略したクラステンプレート名の事です。

```cpp
template<typename T>
struct B {};

template<typename T>
struct C : B<T> {};

struct D : C<int> {
  void example() {
    // 1-1 
    constexpr auto r1 = ^^C::template B; // ok、Bの映象
    // 1-2 
    constexpr auto r2 = ^^C::B;          // ok、B<int>の映象
  }
};
```

`D::example()`内では、`D`、`C`、`C::B`という名前が注入されたクラス名になります。

この例において`r1`が表現するのは特殊化前のクラステンプレートそのもので、`B<int>`ではありません。一方で、`r2`が表現するのは特殊化されたクラス`B<int>`であり、これはテンプレートの映象ではありません。

2のオーバーロード集合の掬映は、オーバーロード集合そのものではなくあくまでそこに含まれる単一の関数の映象を取得します。

```cpp
template<typename T>
void f(T);    // 宣言1
template<typename T>
void f(T) {}  // 宣言2（再宣言）

template<typename T>
void f(T&) {} // オーバーロード1
template<typename T>
void f(T*) {} // オーバーロード2

consteval void example() {
  auto r1 = ^^f; // ok、関数テンプレートfの映象
  auto r2 = ^^g; // ng、オーバーロード集合に複数のgが含まれている
}
```

3はそのままで、テンプレートの掬映はテンプレートの映象を取得します。

```cpp
consteval void example() {
  auto r1 = ^^std::vector;        // ok、クラステンプレートfの映象
  auto r2 = ^^std::is_void_v;     // ok、変数テンプレートの映象
  auto r3 = ^^std::pmr::vector;   // ok、エイリアステンプレートの映象
  auto r3 = ^^S<int>::template f; // ok、メンバ関数テンプレートの映象
}
```

注入されたクラス名ではない`nested-name-specifier template identifier`の形式も2か3のいずれかによって映象の値が決定されます。

```cpp
template<typename T>
struct S {
  template<typename U>
  void f();

  template<typename U>
  struct S2{};
};

consteval void example() {
  auto r1 = ^^S<int>::template f;   // ok、メンバ関数テンプレートの映象
  auto r2 = ^^S<int>::template S2;  // ok、メンバクラステンプレートの映象
}
```

### 型エイリアス（型テンプレートパラメータ）

名前探索によって型エイリアスが見つかった場合、掬映式の結果は次のようになります

1. 見つかった宣言のいずれかがテンプレートパラメータ`T`である場合、`T`の基底エンティティを表現する詠唱
2. それ以外の場合、名前探索で見つかった全ての宣言のターゲットスコープが同一であるなら、その型エイリアスを表現する映象

型エイリアスの掬映は、エイリアス先の型ではなく型エイリアスそのものを表す映象を取得します。

```cpp
namespace A {
  using T = int;
}
namespace B {
  using T = int;
}

using namespace A;
using namespace B;

consteval void example() {
  auto r1 = ^^A::T; // ok、型エイリアスTの映象
  auto r2 = ^^T;    // ng、Tは異なるターゲットスコープを持つ
}
```

1のテンプレートパラメータに関する規則は、規格上は型テンプレートパラメータもそのスコープ内で型エイリアスとして扱われていることによります。すなわち、型テンプレートパラメータの掬映はテンプレートパラメータの映象ではなくテンプレートパラメータに指定された型の映象を取得します。

```cpp
template<typename T>
consteval void example() {
  auto r1 = ^^T;  // テンプレートパラメータ（型エイリアス）ではなく型の映象

  static_assert(^^T == ^^int);  // ✅ Tがintで置換される場合
}

int main() {
  example<int>();
}
```

映象の比較については後で解説しています、ここでは同じものを表現していれば`true`と考えてください。

なお、ここでの型エイリアスは`using`によるものだけではなく、`typedef`によるものも含まれます。

```cpp
typedef struct X {} Y;
typedef struct Z {} Z;  // 同名エイリアス

consteval void example() {
  auto r1 = ^^Y; // ok、型エイリアスYの映象
  auto r2 = ^^Z; // ok、型エイリアスZの映象（型の映象ではない）
}
```

### クラス・列挙型名

名前探索によってクラス（構造体・共用体）か列挙型の名前が見つかった場合、掬映式の結果はその型を表現する映象です。

```cpp
struct S {};
union U {};
enum E1 {};
enum class E2 {};

consteval void example() {
  auto r1 = ^^S;  // ok、クラス（構造体）Sの映象
  auto r2 = ^^U;  // ok、クラス（共用体）Uの映象
  auto r3 = ^^E1; // ok、列挙型E1の映象
  auto r4 = ^^E2; // ok、列挙型E2の映象
}
```

ここでのクラスとは`class`によって定義されるものだけでなく、構造体および共用体も含みます。

### 無名共用体のメンバ

名前探索によって無名共用体のメンバ名が見つかった場合、掬映式の結果はそのメンバを表現する映象です。

```cpp
struct S {
  // 無名共用体
  union {
    int a;
  };
  
  void example() {
    constexpr auto r1 = ^^a; // ok、無名共用体のメンバaを表す映象
  }
};

consteval void example() {
  auto r2 = ^^S::a;  // ok、無名共用体のメンバaを表す映象
}
```

無名共用体のメンバはその共用体のターゲットスコープに注入されるという特殊な性質を持っているため、`^^ id-expression`の形式とは区別して扱われます。

### それ以外の場合

名前探索の結果がここまでのいずれにも該当しない場合、その掬映式は`^^ reflection-name`の形式ではなく`^^ id-expression`の形式として扱われます。こちらの詳細は後述します。

## `^^ type-id`の形式

この形式の掬映式は型名に対する映象を取得します。

`type-id`は以前からある文法規則でそこそこ複雑なのでここでは深堀しませんが、CV修飾付きの型名（基本型やクラス型、関数型、配列型、各種ポインタ型など）にマッチする文法です。

この形式の掬映式の結果は、`type-id`に指定されているものの種類によって次のようになります

1. `type-id`がプレースホルダ型の場合、不適格
2. `type-id`が`nested-name-specifier(opt) template(opt) simple-template-id`という形式であり、その末尾の名前がエイリアステンプレートを指す場合、その型エイリアの映象
3. それ以外の場合、`type-id`が指定する型の映象

1のプレースホルダ型というのは`auto`や`decltype(auto)`などの事で、`^^auto`などはコンパイルエラーになります。

2はエイリアステンプレートの特殊化に対するもので、エイリアステンプレートを特殊化した名前に対する掬映は型エイリアスとしての映象を取得します。

```cpp
namespace N {
  template<typename T>
  using Vec = std::vector<T>;
}

consteval void example() {
  auto r1 = ^^N::Vec<int>;  // ok、型エイリアスの映象
  auto r2 = ^^N::Vec;       // ok、エイリアステンプレートの映象
  auto r3 = ^^N::template Vec<int>;  // ok、
}
```

`r1`が2の規則が指定するエイリアステンプレートを特殊化した型エイリアスの映象です。これはエイリアステンプレートの映象`r2`とは異なるものです。`r3`は冗長な指定ですが、2の規則の前段の条件に合致する記述の例です。

3に該当するのは、`^^ reflection-name`の形式でカバーされていない型名が指定された場合です。これには関数型や配列型、クラステンプレートの特殊化などに加えて、組み込み型も含まれます。

```cpp
consteval void example() {
  auto r1 = ^^void(int);        // ok、関数型の映象
  auto r2 = ^^void(*)(int);     // ok、関数ポインタ型の映象
  auto r3 = ^^int[];            // ok、配列型の映象
  auto r4 = ^^int(&)[5];        // ok、配列参照型の映象
  auto r5 = ^^std::vector<int>; // ok、クラステンプレートの特殊化の映象
  auto r6 = ^^long long;        // ok、long long型の映象
  auto r7 = ^^int&;             // ok、参照型の映象
}
```

少し変わったところだと、型パックのインデックスアクセスもここでカバーされます。

```cpp
template<typename... Ts>
void f(Ts... pack) {
  constexpr auto r = ^^Ts...[0];  // ok、型の映象
}
```

パックのインデックスアクセスはC++26で導入された機能で、このようにパラメータパックに対して定数値によるインデックスアクセスを行えます。

## `^^ id-expression`の形式

この形式の掬映式は変数名に対する映象を取得します。

`id-expression`とは変数名を指定する式であり、式が評価されると変数の値を示すものです。

まず初めに、`id-expression`に指定されている名前が次のものである場合、その掬映式は不適格となります。

1. 初期化キャプチャで宣言された変数
2. 関数ローカルな事前定義変数（function-local predefined variable）
    - 規格上のものは`__func__`のみ
3. `requires`式によって導入されたローカルパラメータ
4. ラムダ式内部からその外側で定義されたローカルなものの名前を指定している場合

```cpp
template<typename T>
  requires requires(T t) {
    ^^t;  // ng、requires式のローカルパラメータ
  }
void example(int p) {
  int l = 0;

  [i = l] {
    constexpr auto r1 = ^^i;  // ng、初期化キャプチャ
    constexpr auto r2 = ^^l;  // ng、ラムダ式外のローカル変数
    constexpr auto r3 = ^^p;  // ng、ラムダ式外のローカル変数
  };

  constexpr auto r1 = ^^__func__; // ng、関数ローカルな事前定義変数
}
```

これら以外のオペランドに対する掬映式の結果は、その名前の種類によって決定されます。

### 関数

`id-expression`に指定されている名前が関数名である場合でその関数がオーバーロードされていなければ、掬映式の結果はそのメンバを表現する映象です。このとき、その関数が`delete`されていてもその関数の映象を取得できます。

```cpp
void f();
void g() = delete;

consteval void example() {
  auto r1 = ^^f; // ok、関数f()の映象
  auto r2 = ^^g; // ok、削除されている関数g()の映象
}
```

オーバーロードされている関数名に対する掬映はコンパイルエラーになります。

```cpp
void f(int);
void f(double);

consteval void example() {
  auto r = ^^f; // ng、f()はオーバーロードされている
}
```

関数テンプレートでも同様でしたが、関数オーバーロードの集合を表現するような映象を取得することはできません。

### 変数等

`id-expression`に指定されている名前が何らかの変数の名前である場合、掬映式の結果はその変数そのもの（値ではなく）を表現する映象です。この何らかの変数とは次のものです

- 変数
- 構造化束縛
- 列挙子
- 非静的メンバ変数

```cpp
auto f() -> std::pair<int, double> {
  return { 0, 2.0 };
}

enum E {
  e = 0,
};

int G = 0;

consteval void example() {
  int l = 0;
  auto [s, _] = f();

  auto r1 = ^^G;  // ok、グローバル変数
  auto r2 = ^^l;  // ok、ローカル変数
  auto r3 = ^^s;  // ok、構造化束縛
  auto r4 = ^^e;  // ok、列挙子
}

class S {
  int m_;

  inline static int N = 0;

  void example() {
    constexpr auto r5 = ^^m_; // ok、メンバ変数
    constexpr auto r6 = ^^N;  // ok、静的変数
  }
};
```

非静的メンバ変数の掬映においては、そのオペランドにおいて暗黙的な`this`経由のアクセスに変換されず直接的にメンバ変数名をオペランドとして参照します。すなわち、上記例の`^^m_`は`^^(*this).m_`のように書き換えられているわけではありません。

### ここまでのいずれにも当てはまらない場合

どの形式のどのパターンにも当てはまらない場合、その掬映式は不適格です。例えば、式としてのコンセプトや定数テンプレートパラメータ（NTTP）名、値パックのパックインデックス式などが該当します。

パックインデックス結果の掬映は型パックの場合にのみ許可されており、値パックに対して掬映できません。これは、値パックのインデックスアクセスが定数式であることと`^^`のオペランドが評価されないことが衝突するのを回避するためのようです。

```cpp
template<typename... Ts>
void f(Ts... pack) {
  constexpr auto r = ^^pack...[0];  // ng、値パックのインデックスアクセス結果の掬映
}
```

定数テンプレートパラメータ（およびコンセプト）も定数式であることが理由で許可されていないようです。

```cpp
template<int N>
consteval void f() {
  auto r = ^^N; // ng、定数テンプレートパラメータの掬映
}
```

これらのものでも将来的に掬映可能になる可能性はあります。一般の式がそうですが、C++26のリフレクション機能としてはまだその掬映が何を表すべきかを決定できていないものが多くあり、そのようなものはコンパイルエラーになるようにされています。

ただし、掬映式から取得できないものの映象であっても、後述するメタ関数を使用して取得できるものがいくつかあります。例えば、すでに言及した値・オブジェクトに対する映象や、関数引数、クラスや名前空間のメンバ、テンプレートパラメータなどがあります。

## `std::meta::info`

掬映式の結果、映象の型は`std::meta::info`という単一の型です。この型は`<meta>`ヘッダにおいて次のように定義されています

```cpp
namespace std::meta {
  using info = decltype(^^::);
}
```

`std::meta::info`型その値として掬映式の結果となるソースコードの構成要素を表現するもの（映象）を保持します。

`std::meta::info`はその値が表現するものによらず単一の型を維持し、これは型指向のテンプレートメタプログラミングと大きく異なる点です。処理対象のものの情報が型などに露出せず、インスタンス化のような手順を必要としません。リフレクション処理に関与する関数はテンプレートにする必要がなく、`std::meta::info`を引数に取る単一の関数として実装できます。

`std::meta::info`型は次のような特性を持ちます

- 不透明型
- スカラ型
- デフォルト構築可能
- 同値比較可能

`std::meta::info`は不透明型であり、公開されたメンバ関数は持たず（規定されず）型の具体的な構造は隠蔽されています。これによって、言語構造のコンパイラ実装への依存を回避するとともに、言語の将来の進化・拡張を妨げないようになっています。この型の実装詳細はコンパイラ実装に委ねられていますが、おそらくASTへの何らかの参照を保持するような実装になるのではないかと思われます。

デフォルト構築された`std::meta::info`は空の映象（null reflection）となります。これは何も表現しない映象です。

```cpp
consteval void example() {
  std::meta::info r{};  // ok、空の映象
}
```

これは`std::meta::info`をクラスの非静的メンバとして保持した時に、そのクラスのデフォルト構築可能性を妨げないようにするための性質です。

`std::meta::info`の値同士は`==`（`!=`）演算子による同値比較が可能で、次のいずれかの場合に`true`になります

1. 両オペランドが空の映象の場合
2. 両オペランドがテンプレート引数として等価な値を表現している場合
3. 両オペランドが同一のオブジェクトを表現している場合
4. 両オペランドが同一のエンティティを表現している場合
5. 両オペランドが同一のアノテーションを表現している場合
6. 両オペランドが同一の直接の基底クラス関係を表現している場合
7. 両オペランドが等しいデータメンバ記述を表現している場合

いずれにも該当しない場合は`false`になり、他の型との比較は定義されていません。

この比較の例を挙げるには、掬映式の直接の結果に加えて後述するメタ関数によるクエリ結果も交える必要があります。そのため、ここではメタ関数の一部を先取りして使用し、その際簡単な説明のみを行います。

まず最初の空の映象の場合、空の映象同士の間でのみ`true`となり、それ以外の映象との間の比較は`false`になります。

```cpp
consteval void example1(std::meta::info not_null) {
  // 空の映象同士は同値
  static_assert(std::meta::info() == std::meta::info());  // ✔

  // 空の映象は空ではない映象とは同値にならない
  static_assert(not_null != std::meta::info()); // ✔ not_nullは空ではない映象とする
}
```

順番が前後しますが次に、4の同一のエンティティについての比較は、映象が同じものを掬映している場合に`true`となります。

```cpp
// 型の映象の比較
static_assert(^^int == ^^int);
static_assert(^^int != ^^const int);
static_assert(^^int != ^^int&);

using namespace std::meta;

// 型エイリアスの映象の比較
using Alias = int;
static_assert(^^int != ^^Alias);
static_assert(^^int == dealias(^^Alias));

// 名前空間エイリアスの映象の比較
namespace AliasNS = ::std;
static_assert(^^::std != ^^AliasNS);
static_assert(^^:: == parent_of(^^::std));
```

`std::meta::dealias()`はエイリアスを表現する映象に対して、エイリアスを解除した（エイリアス先の）ものを表現する映象を取得するメタ関数で、`std::meta::parent_of(r)`は`r`の表現するものが所属するスコープの映象を取得するメタ関数です。

`^^ id-expression`の形式の掬映式から得られる映象の場合、`id-expression`が指定するもの（エンティティ）が同一である映象間のみが`true`となります。

```cpp
int x;
int x2;
struct S { static int y; };

static_assert(^^x == ^^x);
static_assert(^^x != ^^x2);
static_assert(^^x != ^^S::y);
static_assert(^^S::y == static_data_members_of(^^S)[0]);
```

`std::meta::static_data_members_of()`はクラス・構造体を表現する映象からその静的メンバ変数に関する映象の列を取得するメタ関数です。

2は`std::meta::reflect_constant()`や`std::meta::constant_of()`で得られる定数値の映象の比較についてで、これはその値の等価性による比較になります。

```cpp
using namespace std::meta;

static_assert(reflect_constant(10) == reflect_constant(10));
static_assert(reflect_constant(10) != reflect_constant(11));
static_assert(reflect_constant(10) != reflect_constant(10u)); // 型が異なる

constexpr int i = 42, j = 42;

static_assert(^^i != ^^j);  // 異なるエンティティである
static_assert(constant_of(^^i) == constant_of(^^j));  // 値は同じ
```

この2つの関数の違いは少し複雑なので後述します。

この場合の値の比較は通常の比較とは異なり、テンプレート引数として等価（template-argument-equivalent）である必要があります。これは定数テンプレートパラメータ（NTTP）の等価性と同じもので、その値に加えて型も同一である必要があります。

3は`std::meta::reflect_object()`で得られるオブジェクトの映象の比較についてで、この場合同じオブジェクトの映象間のみが`true`となります。

```cpp
int i = 42, j = 42;

static_assert(reflect_object(i) == reflect_constant(i));
static_assert(reflect_object(i) != reflect_constant(j));
static_assert(reflect_object(i) != ^^i);  // 変数の映象とオブジェクトの映象は異なる
```

5のアノテーションはリフレクション機能の一部として導入されているもので、各種の宣言に対してユーザーが注釈を指定しそれを取得できるようにする仕組みです。詳細は後述しますが、このアノテーションの等価性は同じアノテーションを表現する場合に等値となります。

```cpp
[[=1]] void f();
[[=1]] void g();

using namespace std::meta;

static_assert(annotations_of(^^f) == annotations_of(^^f));
static_assert(annotations_of(^^f) != annotations_of(^^g));
```

6の直接の基底クラス関係（direct base class relationship）とは、あるクラス型についてそのクラスが直接的に継承しているクラス型の組（順序対）の事です。これは`std::meta::bases_of()`メタ関数によって取得することができ、この直接の基底クラス関係を表す映象同士の比較は同じ組となっている場合に`true`となります。

```cpp
struct B {};
struct D1 : B {};
struct D2 : D1 {};

consteval void example() {
  using namespace std::meta;

  // これらはstd::vector<std::meta::info>型
  auto dbcr1 = bases_of(^^D1, access_context::unchecked());
  auto dbcr2 = bases_of(^^D2, access_context::unchecked());

  contract_assert(dbcr1.size() == 1);  // ✔️
  contract_assert(dbcr2.size() == 1);  // ✔️ 直接継承している基底クラスのみが含まれる

  contract_assert(dbcr1[0] == dbcr1[0]); // ✔️
  contract_assert(dbcr1[0] != dbcr2[0]); // ✔️
}

int main() {
  example();
}
```

`bases_of()`は`std::meta::info`の`std::vector`を返すのですが、コンパイル時の`std::vector`の使用時は実行時に残らない（コンパイル中に破棄される）ようにする必要があるため、この例では`consteval`関数内で`contract_assert`（C++26からの契約アサーション）を使用しています。

最後のデータメンバ記述（data member description）とは`std::meta::data_member_spec()`メタ関数によって取得できるもので、後述する`define_aggregate()`を使用する際のメンバ変数の詳細を指定するものです。これは、型・名前・アラインメント・ビットフィールド幅・`[[no_unique_address]]`指定の5つ組になっており、これらの指定が完全に一致する場合に2つのデータメンバ記述（を表現する映象）は等値になります。

```cpp
using namespace std::meta;

// これらもmeta::info型の値
constexpr auto dmd1 = data_member_spec(^^int, { .name = "a" });
constexpr auto dmd2 = data_member_spec(^^int, { .name = "b" });
constexpr auto dmd3 = data_member_spec(^^int, { .name = "a", .alignment = 4 });

static_assert(dmd1 == dmd1);
static_assert(dmd1 != dmd2);  // 名前が異なる
static_assert(dmd1 != dmd3);  // アラインメント指定が異なる
```

データメンバ記述および`define_aggregate()`の詳細は後述します。

### `std::meta::info`が保持できるもの

映象には掬映式から取得できるもの以外にもメタ関数から取得できるものがあり、それらすべてが`std::meta::info`型の値として取り扱われます。`std::meta::info`型の値である映象は、C++26では次のものを表現することができます

- スカラ型の値
- 静的記憶域期間のオブジェクト
- 変数
- 構造化束縛
- 関数
- 関数引数
- 列挙子
- アノテーション
- 型エイリアス
- 型
- クラスメンバ
- 無名のビットフィールド
- クラステンプレート
- 関数テンプレート
- 変数テンプレート
- エイリアステンプレート
- コンセプト
- 名前空間エイリアス
- 名前空間
- 直接の基底クラス関係
- データメンバ記述

```cpp
int arr[] = {1, 2, 3};
auto [a1, a2, a3] = arr;
[[=1]] void fn(int n);
enum Enum { A };
using Alias = int;
struct B {};
struct S : B {
  int mem;
  int : 0;
};
template<auto> struct TCls {};
template<auto> void TFn();
template<auto> int TVar;
template<auto N> using TAlias = TCls<N>;
template<auto> concept Concept = requires { true; };
namespace NS {};
namespace NSAlias = NS;

constexpr auto ctx = std::meta::access_context::current();

constexpr auto r1 = std::meta::reflect_constant(42);     // 値（42）
constexpr auto r2 = std::meta::reflect_object(arr[1]);   // オブジェクト
constexpr auto r3 = ^^arr;                               // 変数
constexpr auto r4 = ^^a3;                                // 構造化束縛
constexpr auto r5 = ^^fn;                                // 関数
constexpr auto r6 = std::meta::parameters_of(^^fn)[0];   // 関数引数
constexpr auto r7 = ^^Enum::A;                           // 列挙子
constexpr auto r8 = std::meta::annotations_of(^^fn)[0];  // アノテーション
constexpr auto r9 = ^^Alias;                             // 型エイリアス
constexpr auto r10 = ^^S;                                // 型
constexpr auto r11 = ^^S::mem;                           // クラスメンバ
constexpr auto r12 = std::meta::members_of(^^S, ctx)[1]; // 無名ビットフィールド
constexpr auto r13 = ^^TCls;                             // クラステンプレート
constexpr auto r14 = ^^TFn;                              // 関数テンプレート
constexpr auto r15 = ^^TVar;                             // 変数テンプレート
constexpr auto r16 = ^^TAlias;                           // エイリアステンプレート
constexpr auto r17 = ^^Concept;                          // コンセプト
constexpr auto r18 = ^^NSAlias;                          // 名前空間エイリアス
constexpr auto r19 = ^^NS;                               // 名前空間
constexpr auto r20 = std::meta::bases_of(^^S, ctx)[0];   // 直接の基底クラス関係
constexpr auto r21 =
  std::meta::data_member_spec(^^int, {.name="member"});  // データメンバ記述
```

このリストは将来にわたって固定化されたものではなく、C++29以降で順次拡張されていく予定です。

### 保持可能な値の制限

`std::meta::reflect_constant()`や`std::meta::constant_of()`を使用すると式（定数式）の値の映象を取得することができます。この定数値の映象もまた`std::meta::info`型の値となりますが、`std::meta::info`が保持できる値の型は構造的な型（structural type）に制限されています。

構造的な型は定数テンプレートパラメータ（NTTP）として使用可能な型の分類で、次のいずれかの型です

1. スカラ型
2. 左辺値参照型
    - ただし、`std::meta::info`は参照を保持できない
3. 次の条件を満たすクラス型
    - 全ての基底クラス及び非静的メンバ変数が`public`かつ`mutable`、かつ
    - 全ての基底クラス及び非静的メンバ変数が構造的な型、もしくは構造的な型の配列
4. 一部の標準ライブラリ型
    - `std::string_view`
    - `std::span`
    - `std::tuple`
        - 要素型がすべて構造的な型である場合のみ

スカラ型は整数型や浮動小数点数型、ポインタ型、`std::meta::info`そのものなどが含まれます。

`std::meta::reflect_constant()`や`std::meta::constant_of()`の引数に渡す式はこれらの型（参照型を除く）の定数値となるものである必要があります。

```cpp
// 構造的な型
struct S {
  int n = 0;
};

// 構造的な型ではない
class NS {
  int n = 0;  // プライベートメンバ
};

consteval void example() {
  using namespace std::meta;

  constexpr int N = 0;

  info r1 = reflect_constant(10);     // ok
  info r2 = reflect_constant(N);      // ok
  info r3 = reflect_constant(^^int);  // ok
  info r4 = reflect_constant(S{});    // ok
  info r5 = reflect_constant(NS{});   // ng
}
```

### 関連名前空間

`std::meta::info`型は厳密にいえば型エイリアスであり、エイリアス先の型は明確に規定されたものではありません。このため、`std::meta::info`型の値を用いた関数呼び出し時のADLにおける関連名前空間が定義できないという問題があり、`std::meta::info`型の関連名前空間には`std::meta`が含まれることが明示的に指定されています

```cpp
#include <meta>

struct S {};
std::string name2 = std::meta::identifier_of(^^S);  // ok、修飾名呼び出し
std::string name1 = identifier_of(^^S);             // ok、ADL
```

`std::meta::identifier_of()`は映象が表現するものの名前文字列を取得するメタ関数です。引数に`std::meta::info`を取るため、ADLで呼び出すことができます。

なお、`std::meta::info`型の関連名前空間には`std::meta`以外の名前空間は含まれません。

### コンパイル時専用の値

`std::meta::info`型の値はコンパイル時専用の値（consteval only value）であり、定数式においてのみ使用することができ、実行時に持ち越すことができません。

これは、映象の値の表現方法が実装定義であり、コンパイル時にしか存在しないASTを参照するものである可能性があるなど、実行時に使用できるようにすると実装難易度の上昇や追加のコストなどの問題があるためです。

コンパイル時専用の値は次のいずれかです

- 空ではない映象の値
- `consteval`関数へのポインタ
- 即値オブジェクトへのポインタ
- 3、4の末尾の先（past the end）を指すポインタ

即値オブジェクト（immediate object）とは、そのオブジェクト内に次のいずれかが含まれているものです

- コンパイル時専用の値
- 即値オブジェクト/`consteval`関数を参照するもの

コンパイル時専用の値は`std::meta::info`の値（空ではないもの）のほかに、`consteval`関数ポインタや、`std::meta::info`をメンバに持つクラスのオブジェクトなどがあります。

そして、即値オブジェクト（コンパイル時専用の値）は次のいずれかの使用法がなされている必要があります

1. `constexpr`変数もしくは、そのサブオブジェクトに関連付けられている
2. 定数テンプレートパラメータもしくは、そのサブオブジェクトに関連付けられている
3. 定数式の評価中に生存期間が満了するオブジェクト

```cpp
consteval int plus1(int x) { return x + 1; }

template <auto V>
struct C { };

auto a = plus1;           // ng、非constexpr変数
constexpr auto b = plus1; // ok
auto c = C<plus1>();      // ok、定数テンプレートパラメータに関連付けられている

auto d = ^^int;       // ng、非constexpr変数
auto e = C<^^char>(); // ok、定数テンプレートパラメータに関連付けられている
```

```cpp
struct S {
  std::meta::info r;
};

int main() {
  S s1{};             // ok、空の映象
  S s2{ .r = ^^int }; // ng、非constexpr変数
  constexpr S s3{ .r = ^^int }; // ok
}
```

定数式の評価中に生存期間が満了する場合とは、典型的には`consteval`関数内や`constexpr`変数の初期化式内での使用です。

```cpp
consteval bool f(std::meta::info r) {
  auto r2 = ^^int;
  return r == r2;
}

int main() {
  bool b = f(^^int);  // ok、定数式内での使用
}
```

ここまでの例においても、この制限を回避するために`consteval`関数や`constexpr`変数を使用していました。

# アノテーション

アノテーションは、ソースコード上で何らかの宣言に付与してリフレクション機能を通してそれをコンパイル時に取得し読み取ることができるようにする機能です。他の言語ではユーザー定義属性や単に属性として導入されていることがあります。

アノテーションを用いることで、ソースコード上のエンティティに対してユーザーが任意のメタデータを指定し、それをリフレクション機能によるメタプログラミングにおいて活用することができます。

C++26のアノテーションは属性構文を拡張した`[[=expr]]`の形式で、属性を置くことができる場所に付与できます。ただし付与できるものは制限されており、次の宣言に対してのみ付与可能です

1. クラス定義における基底クラスリスト内の各クラス名
2. 型
3. 型エイリアス
4. 変数
5. 関数
6. 非`void`型の関数引数
7. 名前空間
8. 列挙子
9. 非静的メンバ変数

ただし、クラスや関数の宣言のうち定義を伴わない`friend`宣言に対しては付与できません。

```cpp
// 関数と関数引数
[[=1]] void f();
void g [[=2]] ([[=3]] int a [[=0]]) {
  // 変数
  int n [[=4]] = 0;
  [[=4]] int m = 0;
}

// クラス
struct [[="a"]] S {
  // 非静的メンバ変数
  [[=5.0]]
  int m [[=0]];
  
  // 変数（静的メンバ）
  [[=-1.0]]
  inline static int N [[=4]] = 0;
};

// 基底クラス
struct D : [[=6]] S {};

// 型エイリアス
using A [[=1]] = S;

// 型エイリアス（エイリアステンプレート）
template<typename T>
using AT [[=true]] = std::vector<T>;

// 名前空間
namespace [[=7]] NS {}

// 列挙子
enum class E {
  e [[=0]] = 1,
};
```

属性の指定が実際にどこで可能なのかはC++の文法を紐解く必要があり、かなり複雑になるのでここでは深入りしません。上記例のように、基本的には名前の後ろになりますが、クラス（基底クラス指定も）と名前空間の場合は名前の前になり、関数および関数引数と変数（メンバ変数も含む）の場合は宣言の先頭でも指定することができます。

`[[=expr]]`の`expr`部分には定数式を記述することができ、その式は最終的に構造的な型の定数値に評価される必要があります。これはアノテーションの値が最終的に映象（`std::meta::info`型の値）として取得されることによる制限です。

例えば映象自体をアノテーションに指定することができます。

```cpp
template<typename T>
[[=^^T]]
void f(); // ok
```

アノテーションは繰り返すことができ、`[[=expr1, =expr2, ...]]`のように、カンマと`=`で区切って複数同時に付与できます。あるいは`[[]]`を複数指定することもできます。

```cpp
[[=1, ='a', =3.0]]
void f(); // ok

[[=1]][[=2]][[=3]]
void g(); // ok
```

アノテーションにおいてパラメータパックを展開することもでき、この場合パックの各要素を`elem`とすると`=elem`がカンマ区切りのリストとして展開されます。すなわち、`[[=pack...[0], =pack...[1], ...]]`のように書いた場合と同じになります。

```cpp
constexpr auto f() -> std::tuple<int, char, short> {
  return {1, '2', 3};
}

template<typename... Ts>
void example() {
  // 構造化束縛でのパック導入とconstexpr指定、C++26から
  constexpr auto [...pack] = f();

  // この2つのアノテーションは同じ指定となる
  int n1 [[=pack...]];
  int n2 [[=pack...[0], =pack...[1], =pack...[2]]];
}
```

属性とアノテーションは1つの`[[]]`の中に一緒に書くことはできず、分けて書く必要があります。この時、属性とアノテーションに順番はありません。

```cpp
[[nodiscard, = 1]]
int f(); // ng

[[nodiscard]][[= 1]]
int g(); // ok

[[= 1]][[nodiscard]]
int h(); // ok
```

## アノテーションのクエリ

このように宣言に対して付与されたアノテーションは、専用のメタ関数によってクエリすることができます。アノテーションに関連するメタ関数は次の3つです

```cpp{style=cppstddecl}
namespace std::meta {
  consteval bool is_annotation(info);

  consteval vector<info> annotations_of(info item);
  consteval vector<info> annotations_of_with_type(info item, info type);
}
```

`std::meta::is_annotation(r)`は引数の映象がアノテーションを表現している場合に`true`を返し、それ以外の場合は`false`を返す関数です。

アノテーションのクエリは主に`std::meta::annotations_of()`で行います。この関数は引数`item`が表現するものに対して付与されているアノテーションをすべて取得し、`std::vector`で返します。各アノテーションは`std::meta::info`の値として返されます。

```cpp
int f();

[[= 1]]
int g();

[[= 5]]
int h [[= 10]]();

consteval void example() {
  using namespace std::meta;

  std::vector<info> v1 = annotations_of(^^f);
  std::vector<info> v2 = annotations_of(^^g);
  std::vector<info> v3 = annotations_of(^^h);

  contract_assert(v1.empty());      // ✔️
  contract_assert(v2.size() == 1);  // ✔️
  contract_assert(v3.size() == 2);  // ✔️

  contract_assert(is_annotation(v2[0]));  // ✔️

  [[=1]] int x, y;

  static_assert(annotations_of(^^x)[0] == annotations_of(^^y)[0]);
}

int main() {
  example();
}
```

関数は何度も再宣言することができ、その都度アノテーションを付与することができます。そのような関数のアノテーションをクエリすると、すべての宣言に指定されたアノテーションをすべて集めたものを取得できます。

```cpp
[[= 1]]
void f();

void f [[= 10]] ();

[[= 5]]
void f [[= 6]]();

consteval void example() {
  using namespace std::meta;

  std::vector<info> v = annotations_of(^^f);

  contract_assert(v.size() == 4);  // ✔️
}

int main() {
  example();
}
```

この時、`annotations_of()`の返す`vector`の順序は基本的にそのアノテーションの登場順になります。例えば上記例の場合は`1, 10, 5, 6`の順番になります。ただし、翻訳単位が異なる場合で双方の翻訳単位のクエリ地点から見える宣言が異なる場合などはそれぞれ異なる順序になる場合があります。

`std::meta::annotations_of()`の引数にはアノテーションを付与できるものを表現した映象を渡さなければならず、それ以外の映象を渡すとエラーになります。

```cpp
consteval void example() {
  using namespace std::meta;

  auto [a, _] = std::pair{0, 1};

  annotations_of(^^std::integral);  // ng、コンセプトはアノテーションを付与できない
  annotations_of(^^a);              // ng、構造化束縛はアノテーションを付与できない
}

int main() {
  example();
}
```

この各アノテーションの映象は、`[[=expr]]`の`expr`を定数式で評価した結果の定数値を表現するものです。アノテーションの映象から定数値を取り出すにはいくつか方法がありますが、一番簡単なのは`std::meta::extract<T>()`を使用することです。

```cpp
[[= 10]]
int f();

consteval void example() {
  using namespace std::meta;

  constexpr info r = annotations_of(^^f)[0];
  constexpr int N = extract<int>(r);

  static_assert(N == 10);
}
```

`std::meta::annotations_of_with_type()`は`std::meta::annotations_of(item, type)`の結果に対して、アノテーションの映象が表現する値が型の映象`type`と（`const`修飾を除いて）同じ型を持つものだけをフィルタした結果を返します。

```cpp
[[=1, =2, =3.0, ="4", = 5]]
int f();

consteval void example() {
  using namespace std::meta;

  std::vector<info> v = annotations_of_with_type(^^f, ^^int);

  contract_assert(v.size() == 3);           // ✔️
  contract_assert(extract<int>(v[0]) == 1); // ✔️
  contract_assert(extract<int>(v[1]) == 2); // ✔️
  contract_assert(extract<int>(v[2]) == 5); // ✔️
}

int main() {
  example();
}
```

`std::meta::extract()`も含めた各関数の詳細は後述します。

## その他の注意事項

### アノテーションを付与できない宣言

先ほど取り上げたアノテーションを付与できる宣言でも、次のいずれかに該当する場合その宣言にアノテーションを付与できません

1. 宣言のホストスコープがターゲットスコープと異なる
2. 定義の無い`friend`宣言

宣言のターゲットスコープとは、その宣言が置かれているスコープの事です。宣言のホストスコープもおおむねターゲットスコープと同じになるのですが、一部の宣言では異なることがあります。その分類には深入りしませんが、ここで該当するのはどうやらローカルスコープにおける`extern`変数の宣言のみです。

```cpp
void f() {
  extern int global [[=1]]; // ng
}
```

このような宣言は、別の翻訳単位で定義されている外部リンケージを持つグローバル変数をローカルスコープに持ち込むために有効です。しかし、そのローカル`extern`宣言にアノテーションを付与できません。

2の定義の無い`friend`宣言とは、関数`friend`宣言において定義を行わないものです。

```cpp
struct S {
  friend void f1 [[=1]] ();     // ng
  friend void f2 [[=1]] () {}   // ok、定義がある
};
```

クラスやクラステンプレートの`friend`宣言においては、属性が指定できないようになっているためそもそもアノテーションを付与できません。

```cpp
class C;

struct S {
  friend class [[=1]] C;  // ng、無効な属性（アノテーション）の配置
};
```

### 関数引数のアノテーション

関数引数に対するアノテーションは、関数引数とそれに対応する変数の両方に付与されます。

```cpp
void f([[=1]] int x);

void f([[=2]] int y) {
  using namespace std::meta;

  constexpr auto rp = parameters_of(^^f)[0];  // f()の第一引数の映象
  constexpr auto ry = variable_of(rp);        // ↑の引数に対応する変数の映象

  static_assert(ry == ^^y); // ryは変数（引数）yの映象である

  // f()の第一引数にはアノテーションが2つ付与されている
  static_assert(annotations_of(rp).size() == 2);    // `1`, `2`の2つ

  // 変数yにはアノテーションは1つ付与されている
  static_assert(annotations_of(ry).size() == 1);    // `2`のみ

  // f()の第一引数の2つ目のアノテーションは、変数yのアノテーションに等しい
  static_assert(annotations_of(rp)[1] == annotations_of(ry)[0]);
}
```

関数定義内でその関数のある引数に対応する変数は一つですが、関数の仮引数宣言は複数存在する場合があります。リフレクション機能では両者の映象を区別してそれぞれ取得することができ、関数引数に対するアノテーションはその両方に対して付与されます。

関数引数の映象に関する詳細は後述します。

### テンプレートとアノテーション

コンセプト以外のテンプレートの宣言にはアノテーションを付与することができますが、テンプレート特有の特性があります。

まず、テンプレートに付与したアノテーションはインスタンス化毎に異なるものになります。

```cpp
// 変数テンプレート
template<int>
int x [[=1]];

using namespace std::meta;

// 異なるインスタンス化のアノテーションは異なるアノテーション
static_assert(annotations_of(^^x<0>) != annotations_of(^^x<1>));
```

アノテーションの映象の等価性はその値（定数値）の等価性とは異なりますが、宣言されている場所が同一であれば同じアノテーションである訳でもありません。

「同じアノテーション」の定義は難しいですが、基本的にはあるアノテーション指定（`[[=expr]]`）によるアノテーションと同じアノテーションになるのは同じアノテーション指定によるもののみで、テンプレートの場合は同じアノテーション指定によるアノテーションでもインスタンス化毎に異なるアノテーションになります。

また、アノテーションにはほぼ任意の式を指定することができますが、これはテンプレートにおいて直接的なコンテキスト（immediate context）とみなされず、SFIINAEの対象になりません。

```cpp
template<class T>
[[=T::type()]]
void f(T t);  // (1)

void f(int);  // (2)

void g() {
  f(0);         // ok、引数型がマッチするため(1)が選択される
  f('0');       // ng、(1)の候補検討時にアノテーション内の式においてハードエラー
}
```

# メタ関数

## エラーハンドリング

## `std::meta::access_context`

## 式のリフレクションと値/オブジェクトのリフレクション

We for now restrict the space of reflectable values to those of structural type in order to meet two requirements:

## 関数仮引数


## メタ関数


# `[: r :]` - スプライス式

[ 注1: トークン `[:` および `:]` は、ダイグラフ（二文字表記）から構成することはできない。 — 注釈終わり ]

represents と designates

TU-local（スプライス）

move-eligible splice-expression

primary-expression

クラスの非静的データメンバまたは暗黙的オブジェクトメンバ関数を指定するid式またはスプライス式は、以下の場合にのみ使用できる。

- (4.3) そのid式またはスプライス式が非静的データメンバを指定しており、かつ、それが評価されないオペランド内に現れる場合。

`typename template [:^^TCls:]<3>::type`、typename/templateが無いと`<`が比較演算子扱いになる

```cpp
constexpr auto i = e<[:^^h:]>;
  // error: unparenthesized splice-expression used as template argument
constexpr auto j = e<([:^^h:])>;  // OK
```

`obj.template [:R:]`、`ptr->template [:R:]`は`template [:R:]`とは異なる



## スプライスの制限

- コンストラクタリフレクションのスプライス
- スプライスによる名前空間の定義
- `using enum`/`usin namespace`でのスプライスの使用
- テンプレートパラメータ宣言での使用
- 指示付き初期化における指示子としての使用

# expansion statement

# `consteval`ブロック


# `define_aggregate`

injected declaration

characteristic sequence

injected declarationの生成は副作用（translation environment
`consteval`ブロックではないコア定数式から注入宣言（injected declaration）が生成されることを禁止する。

\clearpage

# 実践例

\clearpage

# メモ

TU-local（リフレクト式）

[ 編集者注: 本稿全体を通じて、リフレクション（`std::meta::info` 型のオブジェクト）はあるソース構成要素を「表現」し、そのリフレクションをスプライスすることは当該ソース構成要素を「指し示す」ものとして記述しています。例えば、`^^int` は型 `int` を表現し、`[: ^^int :]` は型 `int` を指し示します。 ]

コア定数式 ([expr.const]) としての式の評価中に、個々の演算子のオペランドと、順序付けされていない、または不定に順序付けされている個々の式の部分式の評価は、字句順に評価されます。

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- Compiler Explorer(https://godbolt.org/)
- yohhoyの日記(https://yohhoy.hatenadiary.jp/)


https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2996r13.html
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3096r12.pdf
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3293r3.html
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3394r4.html
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p3795r2.html
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4101r1.html

\clearpage
