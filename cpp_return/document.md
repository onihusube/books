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

`return`文の効果は、実行がそこに到達すると現在呼び出し中の関数からその呼び出し元に戻る、というものです。その際、式あるいは初期化子（まとめて戻り値式とする）が指定されている場合（上記下2つの形式）は、その戻り値式から戻り値型のオブジェクトを構築して呼び出し元に返します。

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

## 戻り値の初期化

## 暗黙変換

## `co_return`


\clearpage

# 戻り値最適化（RVO）

## コピー省略

### Named Constructor Idiom

### NRVO

## 暗黙ムーブ

### バージョンごとの変化

### C++23 ローカル参照返しの禁止

## ABIから見たRVO

\clearpage


# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
