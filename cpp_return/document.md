---
title: C++ return文で起こること
author: onihusube
date: 2023/08/12
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

本書はC++20を基にして記述されています。そのため、C++20までに導入されている機能については特に導入バージョンについて触れません。ただし、一部C++23に関する部分があり、それについてはC++23で変更・追加された機能であることを明示します。

## サンプルコードのお約束

- ヘッダのインクルードは全て省略し、サンプルコードの最初で`import std;`しているものとします。
- 主題と関係ないところを`...`で省略していることがあります。
- 行の末尾コメントで`ok/ng`と表示することで、それがコンパイル可能かどうかを示しています。`// ok`はコンパイルが通り未定義動作がないこと、`// ng`はコンパイルエラーとなる事をそれぞれ表しています。

コードブロック中で標準出力をしている時、直後のブロックでその出力例を示していることがあります。例えば次のようになっています

```cpp
int main() {
  std::cout "hello world!";
}
```
```{style=planetext}
hello world!
```

\clearpage

# `return`文の基礎

ここでは、`return`文の基本事項をまとめておきます。そんなん知っとるわ！という方は読み飛ばしても構いません。

## 文法

C++の文法定義中では、`return`は`statement`の内の`jump-statement`の一種として定義されています

```
statement:
  labeled-statement
  attribute-specifier-seq(opt) expression-statement
  attribute-specifier-seq(opt) compound-statement
  attribute-specifier-seq(opt) selection-statement
  attribute-specifier-seq(opt) iteration-statement
  attribute-specifier-seq(opt) jump-statement
  declaration-statement
  attribute-specifier-seq(opt) try-block

jump-statement:
  break ;
  continue ;
  return expr-or-braced-init-list(opt);
  coroutine-return-statement
  goto identifier ;
```

これはC++の規格にある文法定義のEBNFを少し改変（下付き文字の`opt`を`(opt)`に変更）したものです。

`statement`とは文の定義を指しており、`return`は文（*statement*）であり式（*expression*）ではありません。

`statement`中の`jump-statement`参照箇所の定義より、属性指定を`return`文に行う場合は`return`の前で行います（現在のところ標準の属性にはそのような属性は存在しませんが）。

```cpp
auto f() {
  // returnに対する属性指定
  [[attribute]] return ...;

  // こうでも良い
  [[attribute]]
  return ...;
}
```

この`jump-statement`が直接参照される規則はなく、常に`statement`を介して参照されます。`statement`は直接参照されるか`compound-statement`を介して参照されます

```
compound-statement:
  { statement-seq(opt) label-seq(opt) }

statement-seq:
  statement
  statement-seq statement
```

`statement-seq`とは`statement`の列で、`compound-statement`とはいわゆるブロック（`{}`）構造のことを指しています。

この`compound-statement`及び`statement`は他の文や構文などから参照されます。

```
// if/switch
selection-statement:
  if constexpr(opt) ( init-statement(opt) condition ) statement
  if constexpr(opt) ( init-statement(opt) condition ) statement else statement
  if !(opt) consteval compound-statement
  if !(opt) consteval compound-statement else statement
  switch ( init-statement(opt) condition ) statement

// for/while
iteration-statement:
  while ( condition ) statement
  do statement while ( expression ) ;
  for ( init-statement condition(opt) ; expression(opt) ) statement
  for ( init-statement(opt) for-range-declaration : for-range-initializer ) statement

// ラムダ式
lambda-expression:
  lambda-introducer attribute-specifier-seq(opt) lambda-declarator compound-statement
  lambda-introducer < template-parameter-list > requires-clause(opt) attribute-specifier-seq(opt) lambda-declarator compound-statement

// 関数宣言
function-body:
  ctor-initializer(opt) compound-statement
  function-try-block
  = default ;
  = delete ;

// try-catch
try-block:
  try compound-statement handler-seq
function-try-block:
  try ctor-initializer(opt) compound-statement handler-seq
handler-seq:
  handler handler-seq(opt)
handler:
  catch ( exception-declaration ) compound-statement
```

これらの構文中の、`compound-statement`及び`statement`の場所に`return`文を書くことができるわけです。C++23時点ではその場所はこれが全てのはずです。

長々と文法を示してきましたがここで驚くべきことは特にありません。`return`文は皆さんが通常書けると思っているところに書くことができ、書けないと思っているであろう場所には書くことができません。

## `return`の構文と効果

`return`文の文法定義（`return expr-or-braced-init-list(opt);`）を少し解いてみると、その構文は次のいずれかの形になります

```
return;
return expr;
return {...};
```

一番上の形式は戻り値型が`void`の関数における`return`文の書き方で、残りは何かしら戻り値を返す場合の`return`文の書き方です。

`return`文の効果は、実行がそこに到達すると現在呼び出し中の関数を終了して呼び出し元に戻る、というものです。その際、式あるいは初期化子（`return`文のオペランド式）が指定されている場合（下2つの形式）は、そのオペランド式から戻り値型のオブジェクトを構築して呼び出し元に返します。

関数の実行が`return`文に到達すると、次の順序で関数の終了処理が行われます

1. `return`文のオペランド式の実行
    - `return`文のオペランド式の実行によって一時オブジェクトが生成される場合、その解法は戻り値初期化の後かつローカル変数解放の前
2. 戻り値の初期化
3. ローカル変数の解放（デストラクタ実行とスタック解放）
    - 宣言と逆順で解放する
4. 関数引数の解放
    - 解放の順序は呼び出し規約によって異なる
    - 呼び出し規約によっては、関数引数の解放は呼び出し元が行う（関数の終了後に行われる）
5. 関数の終了（呼び出し元への復帰）

### 暗黙に`return`されるところ

次の場所では、明示的な`return`文なしに関数が正常に`return`します

1. 戻り値型が`void`な関数の本体（関数`try`ブロックを含む）終端
2. コンストラクタ/デストラクタの終端
3. `main()`の終端

1と2の場合は`return;`が暗黙に実行されたかのように扱われ、3の場合は`return 0;`が暗黙に実行されたかのように扱われます。

```cpp
void f() {
  int n = 10;
  std::cout << n;
  //return;
}

struct S {
  int n;

  S(int m) : n{m} {
    std::cout << n;
    //return;
  }

  ~S() {
    std::cout << n;
    //return;
  }
}

int main() {
  f();
  //return 0;
}
```

これ以外の場所で`return`文を省略すると未定義動作となります。

```cpp
int f() {
  int n = 10;
  std::cout << n;

  // UB!
}

int g(bool b) {
  if (b) {
    return 20;
  }

  // UB!
}
```

これはコンパイルエラーではなく未定義動作となります。コンパイラはおそらく警告してくれますがエラーにはしてくれないため、そのようなパスが実行されると意図しない結果をもたらします（おそらく実行時にストップせず通過し、戻り値の利用が安全ではなくなります）。

これがエラーにならないのは、例外を投げるなどをした場合に到達し得ないパスがあり得るためです。

```cpp
int g(bool b) {
  if (b) {
    return 20;
  }

  throw std::runtime_error{"b = false!"};
  // ここには絶対に到達しない
}
```

コンパイラによっては、この場合に`return`文がないという警告が抑制される場合があります。

### `[[noreturn]]`属性

先ほどの例のように到達しないことがわかっている場合に`return`文を省略した時でも、コンパイラによっては`return`文がないという警告が表示される場合があります。そのような場合に警告抑制を確実にするためには、`[[noreturn]]`属性を付加した関数で実質最後の実行パス（先ほどの例なら`throw`）をラップします。

```cpp
[[noreturn]]
void err() {
  throw std::runtime_error{"b = false!"};
}

int g(bool b) {
  if (b) {
    return 20;
  }

  err();
  // ここには絶対に到達しない
  // return文がないという警告は抑制される
}
```

`[[noreturn]]`属性はその関数呼び出しが正常に`return`しないことを表明する関数に対する属性指定です。なお、`[[noreturn]]`属性が指定された関数が正常に`return`してしまうと、それはそれで未定義動作になります。

```cpp
[[noreturn]]
void err(bool b) {
  if (b) {
    throw std::runtime_error{"b = false!"};
  }

  // UB!
}
```

## 戻り値型推論

関数の戻り値型にはプレースホルダ型指定（`auto`や`decltype(auto)`など）を置くことができて、この場合は戻り値型が関数の`return`文から推論されます。

```cpp
// 戻り値型推論を行う関数宣言のバリエーション
auto f();
decltype(auto) f();
auto& f();
auto&& f();
auto* f();
```

これらに加えて、`auto`+`const/volatile`修飾（ただし参照/ポインタ修飾がない場合は無意味）及び後置戻り値型（後置と前置で意味は変わらない）においても同様のプレースホルダ型指定の使い方によって戻り値型推論が可能です。以降は、参照以外に対するCV修飾及び後置戻り値型に関しては考慮せずにこれらのバリエーションについて見ていきます。複雑なことですが、これらのバリエーションでは最終的に推論される型が異なることがあります。

戻り値型としてプレースホルダ型指定がなされた関数においては、その戻り値型はその関数本体の`return`文のオペランド（戻り値式）の型から推論されます。もし関数に`return`文がない場合、戻り値型は`return;`から推論されます。

そして、この推論はプレースホルダ型指定を`auto`として`return`文を`return expr;`とすると、次のような変数`r`の初期化時に推論される型と同じ型になります。

```cpp
auto r = expr;
```

プレースホルダ型指定の異なるバリエーションでは、この例の`auto`を置き換えたものと同じになります。

そしてこの時の推論は、左辺の初期化式（`return`文のオペランド）`expr`の型から推論されます。ただし、`auto`と`decltype(auto)`では少し異なる推論がなされます。

なお、関数テンプレートにおける戻り値型推論は、`return`文がテンプレートパラメータに依存していなかったとしてもその関数テンプレートがインスタンス化されるときに推論が行われます（つまり、two-phase name lookupの1回目の時には戻り値型は推論されません）。

### `decltype(auto)`の場合

プレースホルダ型指定が`decltype(auto)`の場合は割と単純で、`return`文のオペランドの式である`expr`を`decltype`に渡して得られる型がそのまま戻り値型になります。

```cpp
decltype(auto) f() {
  ...

  // このreturn文からの戻り値型推論は
  return expr;
}

// このような変数宣言におけるrの型として取得され
decltype(auto) r = expr;

// それは次のようなRとなる
using R = decltype(expr);
```

この`R`のような`decltype`の結果は式`expr`の種類によって2つに場合分けされます

- `expr`が変数名を指定する式（つまり変数名だけ）、もしくはメンバ名を指定する式の場合
    - その変数の型を取得する
- `expr`がそれ以外の式の場合
    - 式（`expr`）の型と値カテゴリを取得する

式`expr`の型とはその式を最後まで実行したときに得られる結果の型であり、これは式によりますが多くの場合は推測可能だと思います。式の値カテゴリは、式の結果の値が左辺値となるか右辺値となるかを参照修飾`&`と`&&`で区別します。

式の型を`T`（参照修飾を含まない）とすると式の値カテゴリによって最終的な型は次のようになります

- *lvalue*
    - `T&`
- *xvalue*
    - `T&&`
- *prvalue*
    - `T`

式の値カテゴリがどれになるかは式によりますが、式の種類によって細かく規定されているので少し複雑です。多くの場合、即値ではなく一時オブジェクトを生成しないような式（代入演算子や左辺値オブジェクト経由のメンバアクセス、左辺値参照型戻り値の関数呼び出しなど）の場合はほぼ左辺値（*lvalue*）になります。即値（リテラル、一時オブジェクト）や非参照型を返す式（二項演算子や非参照戻り値型の関数呼び出し）、組み込みの値を生成する式（アドレス取得やラムダ式、`this`など）の場合は*prvalue*というカテゴリになり、*xvalue*となるのは明示的に`std::move`のようなキャストをしているかそのようなものを返す式（右辺値参照戻り値型の関数呼び出しや*xvalue*オブジェクト経由のメンバアクセスなど）の場合くらいのはずです。

```cpp
int n = 0;

decltype(auto) r1 = 0;
// 型はint、式の値カテゴリはprvalue(修飾なし)

decltype(auto) r2 = n;
// 型はint、変数名を指定する式

decltype(auto) r3 = (n);
// 型はint&、式の値カテゴリはlvalue(&)

decltype(auto) r4 = std::move(n);
// 型はint&&、式の値カテゴリはxvalue(&&)
```

これは関数宣言と`return`文に直すと次のようになります

```cpp
int n = 0;

decltype(auto) f1() {
  return 0;
}
// 戻り値型はint

decltype(auto) f2() {
  return n;
}
// 戻り値型はint

decltype(auto) f3() {
  return (n);
}
// 戻り値型はint&

decltype(auto) f4() {
  return std::move(n);
}
// 戻り値型はint&&
```

`decltype(n)`のオペランドは変数名を指定する式ですが、`decltype((n))`のオペランドはかっこに囲まれた式であり、前者は変数の型を取得し、後者は式の型と値カテゴリを取得します。これによって、結果に左辺値参照修飾が含まれるかどうかが違ってきます。

とても単純な見方としては、`decltype(auto)`による戻り値型推論は`return`文のオペランドを`decltype(auto)`の`auto`に突っ込んで推論される、と見ることができます。

```cpp
decltype(auto) f() { return expr; }
//        ^                 ~~~~
//        |__________________|
```

なお、プレースホルダ型指定に`decltype(auto)`を使用する場合は`const`や参照修飾などを付加することはできず、ちょうど`decltype(auto)`だけが使用可能です。

```cpp
decltype(auto) & ng();        // ng
const decltype(auto) ng();    // ng
const decltype(auto) & ng();  // ng
```

### `auto`（+修飾）の場合

`auto`の場合は仕組み的には関数テンプレートの実引数推論と同じ推論が行われます。そこでは、戻り値型指定内の`auto`をテンプレートパラメータ`T`に置換した形の引数を持つ1引数関数テンプレートを作成し、その仮の関数テンプレートに`return`文のオペランド（`expr`）を渡した時に`T`に推論される型が戻り値型となります。

```cpp
auto f() {
  ...

  // このreturn文からの戻り値型推論は
  return expr;
}

// このような変数宣言におけるrの型として取得され
auto r = expr;

// それはこのような仮の関数テンプレートに対して
template<typename T>
void hf(T r);

// こう渡したときのTが推論結果となる
hf(expr);
```

このとき修飾が付いている場合（`auto&, auto*`など）、修飾はそのままに`auto`だけをテンプレートパラメータに置換した関数テンプレートを考えます

```cpp
// const auto&の場合の仮の関数テンプレート
const auto& f();

template<typename T>
void hf(const T&);


// auto*の場合の仮の関数テンプレート
auto* f();

template<typename T>
void hf(T*);
```

実際にこのような関数テンプレートで推論される型というのは非常に複雑なルールで決まります。その詳細はあまりに複雑なので（書いてる人も雰囲気でしかわかってないので）踏み込まず、`return`文のオペランド（`expr`）の型（値カテゴリを含む）と戻り値型のプレースホルダ指定の組み合わせ毎に戻り値型がどうなるのかをまとめておきます

|`expr`の型|`auto`|`auto&`|`const auto&`|`auto&&`|`auto*`|
|---|---|---|---|---|---|
|`T`|`T`|×|`const T&`|`T&&`|×|
|`T&`|`T`|`T&`|`const T&`|`T&`|×|
|`T&&`|`T`|×|`const T&`|`T&&`|×|
|`const T`|`T`|×|`const T&`|`T&&`|×|
|`const T&`|`T`|`const T&`|`const T&`|`const T&`|×|
|`const T&&`|`T`|`const T&`|`const T&`|`const T&&`|×|
|`T*`|`T*`|×|`T const *&`|`T*&&`|`T*`|
|`T const *`|`T const *`|×|`T const * const &`|`T const *&&`|`T const *`|
|`T * const`|`T*`|×|`T const *&`|`T*&&`|`T*`|
|`T const * const`|`T const *`|×|`T const * const &`|`T const *&&`|`T const *`|
|`T(&)[N]`|`T*`|`T(&)[N]`|`const T(&)[N]`|`T(&)[N]`|`T*`|
|`R(*)(Args)`|`R(*)(Args)`|×|`R(* const &)(Args)`|`R(*&)(Args)`|`R(*)(Args)`|
|`R(&)(Args)`|`R(*)(Args)`|`R(&)(Args)`|`R(const &)(Args)`|`R(&)(Args)`|`R(*)(Args)`|

表中の×は推論不可、すなわち戻り値型が定まらないためコンパルエラーとなる組み合わせです。`T`には任意の型をはめてください。`Args`は関数引数型を表すパラメータパックだと思ってください。

プレースホルダ型指定の`const auto&&`はおそらく使う機会がないと思われるため省略しています。それを考えるときは、`auto&&`の結果にトップレベルの`const`を付加したものと同じになるはずです。

例えば、プレースホルダ指定が`auto`で`return`文が`return 10;`（`10`の型は参照なしの`int`）の場合は戻り値型は`int`と推論され、プレースホルダ指定が`auto&&`になると戻り値型は`int&&`と推論され、プレースホルダ指定を`auto&`にするとコンパイルエラーとなります。

```cpp
auto f1() {
  return 10;
}
// 戻り値型はint

auto&& f2() {
  return 10;
}
// 戻り値型はint&&

auto& f3() {
  return 10;
}
// これはエラー
```

式の型に関しては式毎に処理を追って確かめるか、`decltype((expr))`で取得することもできます。ただし前述のように、変数名を指定する`decltype()`は式の型を取得しないので、変数名であるかに関わらず式の型を取得したい場合はオペランドを`()`で囲っておく必要があります。ただ、変数名を指定する式の型はその変数の型`T`に対して`T&`になるので、そこまで難しくはないでしょう。

```cpp
int n = 10;

auto f1() {
  return n;
}
// 戻り値型はint

auto&& f2() {
  return n;
}
// 戻り値型はint&

auto& f3() {
  return n;
}
// 戻り値型はint&
```

### `void`に推論されるとき

`auto`系にせよ`decltype`にせよ、`return`文にオペランドがない場合はオペランドとして`void()`が使用され、`return`文がない場合は関数の最後で`return;`があるかのように扱われます。つまり結局、どちらの場合も`return void();`のような`return`文として結果を考えることができます。

とはいえ、これはあまり考えることもなく`void`に推論されます。このとき、`auto&`や`auto*`がプレースホルダ指定に使用されている場合はコンパイルエラーとなります。

```cpp
auto f1() {
  return;
}
// 戻り値型はvoid

decltype(auto) f2() {
  return;
}
// 戻り値型はvoid

auto f3() {
}
// 戻り値型はvoid

auto& f4() {
  return;
}
// エラー

auto* f5() {
  return;
}
// エラー、void*にはならない
```

また、`return`文のオペランドの結果型が`void`となる場合も戻り値型は`void`に推論されます。これは別の関数の呼び出し結果を直接`return`しているような場合が該当します。この場合、`return`文から`void`の結果を返そうとしているように見えて、一見するとエラーになりそうに見えますがエラーにはなりません。実のところ、`void`戻り値型の関数の`return`文のオペランドに結果が`void`となる式（多くの場合は関数呼び出し）を書いてもエラーにはなりません。

```cpp
auto f(std::invocable<int> auto func) {
  return func(10);  // この時点では、func()の戻り値型は不明
}

int f1(int n) {
  return 2*n;
}

void f2(int n) {
  std::cout << n << '\n';
}

void f3() {
  return f2(20);  // ok
}

int main() {
  f(f1);  // ok、f()の戻り値型はint
  f(f2);  // ok、f()の戻り値型はvoid
}
```

この挙動によって、この`f()`のように関数テンプレートで呼び出し可能なものを受け取りその呼び出し結果を直接返すもののその結果型がどうなるかわからないような場合でも、`return`文を同じように書くことができる（結果型を調べて分岐する必要がない）ためTMPの文脈で重宝します。

### 初期化子リストからの戻り値型推論

`return`文のオペランドが初期化子リスト（`{...}`）である場合、戻り値型推論は常に失敗します。

```cpp
auto f1() {
  return {1}; // ng、戻り値型が推論できない
}

auto&& f2() {
  return {1, 2};  // ng、戻り値型が推論できない
}

decltype(auto) f3() {
  return {1, 2, 3}; // ng、戻り値型が推論できない
}
```

これは初期化子リストの要素数や`auto`の修飾に関わらず常に失敗します。`initializer_list`には推論されません。

変数宣言の場合、`auto`は初期化子リストから`initializer_list`を推論できますが`decltype(auto)`はできないという違いがあり、戻り値型推論では挙動を`decltype(auto)`に合わせた振る舞いになっていると思われます。

```cpp
int main() {
  auto           il1 = {1, 2, 3};  // ok、initializer_list<int>
  decltype(auto) il2 = {1, 2, 3};  // ng
}
```

### `return`文が複数ある場合

戻り値型推論を行う関数に複数の`return`文があるとき、戻り値型は`return`文それぞれに対して個別に推論され、最後に推論された全ての戻り値型が一致している場合にのみその型を戻り値型として採用します。もしも、`return`文の間で推論された戻り値型が異なる場合、推論は失敗します。

```cpp
auto f1(bool b) {
  if (b) {
    return 10;    // ng、推論結果はint
  } else {
    return 10.0;  // ng、推論結果はdouble
  }
}

auto f2(bool b) {
  int n = 10;
  const int& r = n;

  if (b) {
    return n; // ok、推論結果はint
  } else {
    return r; // ok、推論結果はint
  }
}

auto& f3(bool b) {
  int n = 10;
  const int& r = n;

  if (b) {
    return n; // ng、推論結果はint&
  } else {
    return r; // ng、推論結果はconst int&
  }
}

// 危険なので真似しないこと！！
decltype(auto) f4(bool b) {
  int n = 10;
  int& r = n;

  if (b) {
    return (n); // ok、推論結果はint&
  } else {
    return r;   // ok、推論結果はint&
  }
}
```

この一致は修飾等も含めて正確に同じ型でなければならず、暗黙変換などは考慮されません。そして、この例の`f2`と`f3`のように`auto`に付加する修飾によって推論結果が一致するかどうかが変化する場合があります。

`f3`や`f4`の例はあくまでどういう推論がなされて一致するかの例を示しているものに過ぎません。このような関数を実際書くことは大変危険なのでやめましょう。

### 例

ここには、いくつかの推論の例を置いておきます。

```cpp
// 配列参照を返す関数
auto test() -> int(&)[10];

auto f1() {
  return test();  // 戻り値型はint*
}

auto& f2() {
  return test();  // 戻り値型はint(&)[10]
}

decltype(auto) f3() {
  return test();  // 戻り値型はint(&)[10]
}
```

```cpp
// 戻り値型が1つに定まらないためエラーとなる例
auto f(int n) {
  if (n < 0) {
    return std::optional{n}; // 推論結果はoptional<int>
  } else {
    return std::nullopt;     // 推論結果はnullopt_t
  }
}
```


```cpp
int main() {
  auto l1 = [](int n) {
    return n; // 戻り値型はint
  };

  auto l2 = [](int n) -> decltype(auto) {
    return (n); // 戻り値型はint&
  };

  auto l3 = [](int n) -> auto&& {
    return (n); // 戻り値型はint&
  };
}
```

ラムダ式の場合、戻り値型としてデフォルトで暗黙的に`auto`指定が使用されています。これを変更するには、後置戻り値型指定で別のプレースホルダ指定を行います。あまり意味はない（と思われる）のですが、普通の関数でも後置戻り値型指定でプレースホルダを指定して戻り値型推論を指示することができます。


## 戻り値の初期化と暗黙変換

`return`文はそこに到達すると関数を終了し呼び出し元に帰ると共に、戻り値型が`void`ではない場合は戻り値の初期化を行います。その際の初期化は`return`文のオペランドからコピー初期化によって行われます。

また、戻り値初期化時には`return`文のオペランドの型と戻り値型が異なる場合に暗黙変換が行われます。これはそれぞれの型の種別に関係なく実行され、関数呼び出し時の引数初期化時や変数の初期化時に起こる暗黙変換と同じものです。この時、暗黙変換の経路が見つからないか、複数見つかる場合はコンパイルエラーとなります。

暗黙変換をするか否か、どの経路で行うかについては戻り値型が決定した後、すなわち戻り値型推論の後で（コンパイル時に）決定されます。戻り値型推論が行われている場合はおそらくあまり大きな変換は行われず、CV修飾や参照修飾の調整、参照からポインタへの変換などに留まるはずです。

関数の戻り値型を`R`とし、`return`文のオペランドを`expr`とすると、戻り値の初期化は単純には次のような変数`r`の初期化と同様です

```cpp
R r = expr;
```

`expr`を関数呼び出しだとするとこれは当たり前に見えるかもしれませんが、この`r`はあくまで戻り値の初期化であってそれを受ける変数の初期化とは異なります。関数呼び出し側で戻り値を受ける変数はこの`r`を用いて再度初期化されることになります。

```cpp
R f() {
  return expr;
}

int main() {
  // このようなvの初期化においては
  auto v = f();

  // この2段階の初期化が行われている（C++意味論的には）
  R r = expr;             // f()の戻り値の初期化
  auto v = std::move(r);  // 変数vの初期化
}
```

以降、この節で説明するのはこの`r`の初期化であって`r`を用いた`v`の初期化ではありません。`v`の初期化は話が少し異なるため一応区別しておきます。ただ、ここには戻り値最適化が関わってくることでその2つは混じり合っており、それによって戻り値の初期化もこの`r`の初期化ほど単純ではなくなっています。戻り値最適化は後半の章の、そして本書のメインテーマです。

`expr`がリスト初期化（`{...}`）の場合はこれはコピーリスト初期化という形の初期化になります。どちらの場合でも`explicit`な変換は禁止され、コピーリスト初期化の場合はさらに縮小変換も禁止されます。

```cpp
auto f1() -> bool {
  std::optional<int> opt = 10;

  return opt; // ng、optionalのbool変換はexplicit
}

auto f2() -> std::shared_ptr<int> {
  int* p = new int{20};

  return p; // ng、shared_ptrの生ポインタコンストラクタはexplicit
}

struct S {
  float f;
};

auto f3(double d) -> S {
  // コピーリスト初期化
  return {d}; // ng、double->floatの縮小変換が起こる
}

auto f4(double d) -> float {
  // コピー初期化
  return d;   // ok、縮小変換は許可される
}
```

縮小変換とはざっくりいうと値の表現が失われるような変換のことで、おおよそ次のいずれかに該当する変換です

- 浮動小数点数型から整数型への変換
- 精度が劣化する浮動小数点数型同士の変換
    - 変換元の値が定数式であり、変換でオーバーフローが発生しない場合は可能
- 整数型から浮動小数点数型への変換
    - 変換元の値が定数式であり、変換でオーバーフローが発生しない場合は可能
- 整数型またはスコープなし列挙型から、元の整数値を全て表現できない整数型への変換
    - 変換元の値が定数式であり、その値が変換先の型で表現可能な場合は可能
- （メンバ）ポインタ型から`bool`型への変換

ただし、コピーリスト初期化における縮小変換は呼ばれるコンストラクタに渡す値が縮小変換を受ける時にだけ禁止されます。コンストラクタでは変換なしで受け取って、コンストラクタ内部で変換を行うようなコンストラクタが呼ばれる場合は禁止されません。

```cpp
struct S {
  int n;
  float f;
};

auto f1(double d) -> S {
  return {d, d}; // ng、集成体初期化において縮小変換が発生する
}

auto f2(double d) -> std::pair<int, float> {
  return {d, d}; // ok、pairのコンストラクタ内部で縮小変換が発生する
}
```

`std::pair`のこの場合呼ばれるコンストラクタは、テンプレート化されてそれぞれの型が対応する要素型に変換できれば選択される（`template<class U, class V> pair(U, V)`のような）もので、コンストラクタ引数では無変換で受け取って、コンストラクタ内で要素を初期化する際に変換するものです。このコンストラクタは条件付き`explicit`指定されているため暗黙変換が禁止されている型では選択されませんが、縮小変換が起こるだけだとリスト初期化時にも選択されます。

コピー初期化はコピーと付いていますがムーブが行われないわけではなく、`expr`が式である場合その値カテゴリによって`R`のコピー/ムーブコンストラクタが適切に選択されます。

```cpp
auto f(int n) -> std::vector<int> {
  if (n == 0) {
    return {1, 2, 3, 4};  // 戻り値はムーブ構築される
  } else {
    std::vector<int> v;
    v.resize(n, 1);

    return v; // 戻り値はコピー構築される
  }
}

int main() {
  auto v1 = f(0);
  auto v2 = f(5);
}
```

ただし、これは後述の暗黙ムーブやコピー省略を考慮しない（あるいは無効化した）場合の挙動になります。実際には暗黙ムーブによって2つ目の`return`文における戻り値はムーブ構築され、コピー省略によって1つ目の`return`文における戻り値は呼び出し側変数（`v1`）に直接構築されます。そして、NRVOが適用されるとどちらの戻り値も呼び出し側変数（`v1, v2`）で直接構築されるようになります。

### 戻り値の初期化までのコンストラクタ呼び出しの回数



```cpp
auto f() -> std::vector<int> {
  return {1, 2, 3, 4}; 
}
```


## `co_return`

（コルーチンの仕様の詳細については、コア言語機能/ライブラリ機能1を参照いただくか、cpprefjpのコルーチンページなどの解説をご覧ください）

`co_return`はコルーチンにおける`return`文であり、文法的には`return`文の一種です。そのため、コルーチン内部である必要がありますが、書ける場所は`return`文と同じになり書き方も同じです。

```
// co_returnの可能な文法
co_return;
co_return expr;
co_return {...};
```

関数に`co_return`文があるとその関数はコルーチンだとみなされますが、コルーチンに普通の`return`文があるとコンパイルエラーになります。

```cpp
auto coro_f() {
  ...

  co_return;  // これによって、この関数はコルーチンとなる

  return; // ng、コルーチンにはreturn文を含められない
}
```

その意味論は`return`文とは異なり、`co_return`文はそこに到達するとその時点で実行中のコルーチンを終了させます。より厳密には、`co_return`文にオペランドがあればそれをプロミス型の`.return_value()`に渡し、なければ`.return_void()`を呼び出して、それらの呼び出しの完了後にコルーチンの最終サスペンドポイントへ移行し、それによってコルーチンを終了させます。

```cpp
auto coro_f() {
  ...

  // これは、exprによって次のどちらかに展開される
  co_return expr;

  // exprがあり、voidの式ではない場合
  {
    promise.return_value(expr);
    goto final_suspend; // コルーチンの最終サスペンドポイントへ移行
  }

  // それ以外（exprが無いか、voidの式）
  {
    {
      expr;
      promise.return_void();
    }
    goto final_suspend; // コルーチンの最終サスペンドポイントへ移行
  }
}
```

`co_return`文は範囲`for`のような言語組み込みマクロのようなもので、コルーチンの明示的終了と終了時処理を実行するためのものですが、その詳細な振る舞いはコルーチンのプロミス型のメンバ関数（`.return_value()/.return_void()`）を通してユーザーが定義します。例えば、コルーチン中で`co_return expr;`としている場合に`expr`の結果の値を`.return_value()`を通してコルーチンの最後の生成結果として返すといったことができます。

なお、`.return_value()/.return_void()`は両方定義するとコンパイルエラーとなり、`.return_void()`が定義されていない状態でコルーチンの終端に到達すると未定義動作となります（`.return_void()`が定義されている場合にのみコルーチン終端到達は`co_return;`と等価となる）。`co_return`のカスタマイズが必要ない場合でも、`.return_void()`は空で定義しておくのが安全です。

\clearpage

# 暗黙ムーブ

## バージョンごとの変化

## C++23 ローカル参照返しの禁止

\clearpage

# コピー省略

前章の「戻り値の初期化と暗黙変換」の節では、戻り値の初期化とそれを受ける変数の初期化を分けて説明していました。

```cpp
R f() {
  return expr;
}

int main() {
  // このようなvの初期化においては
  auto v = f();

  // この2段階の初期化が行われている（C++意味論的には）
  R r = expr;             // f()の戻り値の初期化
  auto v = std::move(r);  // 変数vの初期化
}
```

C++のオブジェクト意味論の要求によって、関数の`return`から呼び出し側での戻り値取得は`expr -> r -> v`のような経路を辿らねばならず、これにはコンストラクタ/デストラクタの呼び出しが伴います。

このような経路は明らかに無駄であり、省けるならば省きたいと思うのはC++という言語としては当然の考えです。そこで、このような場合に`expr -> v`のように経路を圧縮し`return`文のオペランド`expr`によって呼び出し側の変数`v`を直接初期化する最適化のことを値のコピー省略（*Copy Elision*）と呼びます。また、`return`文におけるコピー省略のことを特にReturn Value Optimization(RVO)とも呼びます。本書では単にコピー省略と呼ぶことにします。

## C++17 値のコピー省略保証

C++17以降、このようなコピー省略は必須とされています。

## NRVO

`return`文におけるコピー省略はそのオペランドが*prvalue*である場合にのみ保証されています。例えばローカル変数名を`return`文のオペランドにする場合（オペランドは左辺値）などにはコピー省略は保証されません。

```cpp
int f(int n) {
  int m = 2 * n;

  ...

  return m; // オペランドは左辺値
}

int main() {
  int r = f(2); // コピー省略は行われない
}
```

このような場合にもコピー省略を行うのがNamed Return Value Optimization(NRVO)です。

起こることのイメージとしては、関数引数に戻り値を受けている変数への参照を受けて、`return`文のオペランドとなっている変数への代入をそちらに行うようにする感じのことが起こります。

```cpp
void f(int n, int& r) {
  r = 2 * n;

  ...

  //return m; // オペランドは左辺値
}

int main() {
  int r;
  f(2, r);
}
```

NRVOが行われた場合にこのような書き換えが行われるわけではありません。あくまでイメージです。

NRVOはC++規格において許可されているものの保証されてはいません。そのため、これはコンパイラの最適化の一環として行われ、いつどこで行われるかはコンパイラの裁量によります。

## コピー省略もNRVOも行われない場合

```cpp
int f(bool b, int n) {
  int m1 = n + 10;
  int m2 = n + 20;

  if (b) {
    return m1;
  } else {
    return m2;
  }
}
```

## ABIから見たRVO

\clearpage


# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
