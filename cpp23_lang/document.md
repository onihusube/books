---
title: C++23 コア言語機能
author: onihusube
date: 2024/12/30
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

## 注意など

本書はC++23をベースとして記述されています。そのため、C++20までに導入されている機能については特に導入バージョンについて触れず、知っているものとして詳しい解説なども行いません。

文中では、`std::ranges`もしくは`std::views`から始まるものについては`std::`を省略しています。また、通常の関数の表記（`func()`）に対してメンバ関数の表記（`.member_func()`）は先頭に`.`を付加して区別します。

## 提案文書

C++標準への機能の追加は提案文書を介して行われます。提案文書はC++標準化委員会のメンバによって提出され、それを3段階くらいのレビューステージを通して吟味し、最終的にC++標準化委員会全体の会議で投票にかけられて次のC++バージョンの機能として採用されます。

提案文書はほぼ全て公開されており、英語ではありますが誰でも読むことができます。ただし、C++11以前の古い時代のものは公開されていないものもあります。現在は1ヶ月に一度、毎月20日前後に最新の提案文書が公開されています。

提案文書は`PxxxxRn`のような形式の番号によって管理されています（例えば、P0734R0とかP0515R3など）。`Pxxxx`の部分は提案そのものの個別番号で、`Rn`の数字`n`はその提案のリビジョン、すなわち更新回数を表します。提案はレビューとフィードバックのループを繰り返しながらブラッシュアップされ、その都度その変更・改善を反映した新しい版の文書が公開されます。その際はPから始まる番号はそのままで、R以降の数字がインクリメントされます（たまにRは2桁に達することがあります）。

提案文書内およびC++コミュニティにおいて特定の提案文書を指すときには、タイトルの代わりにこの番号が使用されるのが一般的です。現在は使用されていませんが、C++11前後くらいの時期の古い提案（およびC標準化委員会）ではPではなくN始まりの数字が使用されていることがあり、その場合はリビジョン毎に番号が変わります。

本書では各機能紹介の冒頭で直接の提案文書および関連性の高いものをリストで示してあります。紙で読まれている方は申し訳ないですが、おそらくP番号だけで検索をかけても文書にたどりつけると思います（出てこない場合はC++を追加すると確実）。

## 欠陥報告（DR）

欠陥報告（*Defect Report* : DR）とは、仕様に対する欠陥（バグ）の報告に伴う解決のための提案であり、その変更は過去のバージョンに遡って適用されます。

DRとされた問題については一部のコンパイラは早期に実装している可能性があり、実装済みのコンパイラでは古いバージョンの指定時（C++20対応コンパイラに`-std=c++11`を指定するなど）にも変更が適用されてコンパイルされるようになります。ただし、DRがどのバージョンまで遡って適用されるかは指定されていない場合もあり、その場合はその仕様が最初に導入されたバージョンまで遡るようです。

本書では、機能タイトルの後ろに(DR)としてそれを注記しています。また、特にカテゴライズされないものについて最後の「Defect Report」章にまとめてあります。

## サンプルコードのお約束

- ヘッダのインクルード/インポートは全て省略し、サンプルコードの最初で`import std;`しているものとします。
- 主題と関係ないところを`...`で省略していることがあります。
- 行の末尾コメントで`ok/ng`と表示することで、それがコンパイル可能かどうかを示しています。`// ok`はコンパイルが通り未定義動作がないこと、`// ng`はコンパイルエラーとなる事をそれぞれ表しています。

コードブロック中で標準出力をしている時、直後のブロックでその出力例を示していることがあります。例えば次のようになっています

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

# クラス

## Deducing this

https://wg21.link/P0847R7

## アクセス制御の異なるメンバ変数のレイアウトを宣言順に規定

- P1847R4 Make declaration order layout mandated(https://wg21.link/P1847R4)

C++20までの規定では、クラスのアクセス制御（`private, public, protected`）が異なる場合、実装はデータメンバを並べ替えてメモリに配置することができます。この利点は、型のアライメントの違いによって生じるメンバ変数間のパディングを最小化して構造体サイズを削減することで、メモリ効率を向上させることにあります。

ただし、実際にそれを行う処理系は存在せず、多くのC++プログラマはは並べ替えを考慮していないことがほとんどです。この提案は、そのような慣行に従うように規定を修正し、クラスのデータメンバのメモリレイアウトが常にコード上の宣言順と一致するようにするものです。

```cpp
// C++23から、構造体のデータレイアウトは宣言順と一致する
struct S {
  std::int32_t n;
private:
  double d;
protected:
  std::int8_t s;
};
```

これによってクラスレイアウトに関する規則が単純になるとともに、将来クラスレイアウトをコントロールするための機能を追加する際の土台とすることを意図しています。

以下余談です。

このクラスレイアウトの並べ替えの許可という仕様はC++11から入っていたものですが、意図して導入したものではなく、標準レイアウトクラスという分類を導入する際に誤って混入してしまったもののようです。幸いなことにその仕様を活用する処理系は現れなかったため、この度めでたく消すことができました。

## 添字演算子の多次元サポート

- P2128R6 Multidimensional subscript operator(https://wg21.link/P2128R6)

C++20で添字演算子（`[]`）内でのカンマの使用が非推奨化されていましたが、C++23ではこれに続いて添字演算子のオーバーロードを複数の引数を取るように定義できるようになります。

```cpp
template<class T>
class my_mdspan {

  // 多引数添字演算子オーバーロード
  template<class... IndexType>
  constexpr T& operator[](IndexType...);

  // 関数呼び出し演算子オーバーロード
  template<class... IndexType>
  constexpr T& operator()(IndexType...);

  ...
};
```

複数の引数を取ることができるようになると共に0個の引数でもオーバーロード可能になったため（すなわち引数の個数に関する制限が完全になくなった）、ユーザー定義添字演算子は関数呼び出し演算子と同じようにオーバーロードすることができ、その違いは見た目（`[]`と`()`）のみになります。

Eigenなどの線形代数ライブラリの多次元行列型では添字演算子が引数1つでしかオーバーロードできなかったことから関数呼び出し演算子を代わりに要素アクセスに使用していたりしていました（もっと変なライブラリでは、カンマ演算子オーバーロードを活用していました）。C++23からは、そのような多次元配列型においても添字演算子を使用できるようになり、C++23の`std::mdspan`で早速活用されています。

```cpp
#include <mdspan>

template<typename T>
void print_mat(std::mdspan<T, std::extents<std::size_t, 3, 3>> mat33) {
  for (int y : std::views::iota(0, 3)) {
    for (int x : std::views::iota(0, 3)) {
      // 多次元添字演算子による要素アクセス
      std::cout << mat33[y, x];
    }
  }
}
```

実のところ、この提案のモチベーションの大部分は`std::mdspan`でこれを使用できるようにすることにありました。

## static `operator()`

- P1169R4 static `operator()`(https://wg21.link/P1169R4)

関数呼び出し演算子をオーバーロードする場合はクラスの非静的メンバ関数として定義する必要がありますが、非静的メンバ関数は暗黙に`this`引数を第一引数に取っています。これは、`callable(args...)`のように呼び出し可能オブジェクトに対して関数呼び出しを行う場合にいつも見えない`this`引数が一つ追加で渡されているということです。

通常これは問題にはなりませんが、関数呼び出し演算子を定義するものの`this`に全く依存しない、ステートレスな呼び出し可能型を定義する場合、この暗黙の`this`引数の存在は避け難いオーバーヘッドとなり得ます。多くの場合最適化（インライン展開）によって削除されますが、最適化しきれない場合にはこのオーバーヘッドから逃れることができません。このことはゼロオーバーヘッド原則にも反しています。

特に、C++20`<ranges>`のCPOやRangeアダプタの多くはステートレスな関数オブジェクトであり、その特性上ラッパ層のオーバーヘッドは可能な限り薄くしておく必要があります。

そのため、C++23からは`operator()`のオーバーロードを`static`メンバ関数として定義できるようになります。

```cpp
template<typename T>
struct eq_t {

  // static operator()
  static bool operator()(const T& l, const T& r) {
    return l == r;
  }
};

int main() {
  eq_t<int> eq{};

  bool b = eq(23, 23); // ok、static operator()が呼ばれる
}
```

この`static`な`operator()`は完全に指定された数の引数しか取らず、暗黙の`this`引数はありません。そのため、最適化なしでも追加の引数渡しによるオーバーヘッドは無くなります。ただし当然のことながら、クラスの状態（非静的データメンバ）を読み取ることはできません。

また、同時にラムダ式に対しても`static`指定が可能になり、ラムダ式のクロージャ型で定義される関数呼び出し演算子が`static`で定義されるようになります。

```cpp
int main() {
  // ラムダ式の関数呼び出し演算子がstaticになる
  auto add10 = [](std:integral auto n) static {
    return n + 10;
  }:

  int m = add10(13);  // ok
}
```

ラムダ式の`static`関数呼び出し演算子はこの`static`指定によって有効になるオプトインなもので、`static`無しのラムダ式はこれまで通り非`static`な関数呼び出し演算子を持ちます。

ただし、この`static`指定を行えるのはキャプチャをしていない（ステートレスな）ラムダ式のみです。

```cpp
int main() {
  const int b = 10;

  // キャプチャをしているラムダ式はstatic指定できない
  auto ng1 = [&](std:integral auto n) static  // ng
  {
    return n + b;
  }:

  // 初期化キャプチャでも不可
  auto ng2 = [bias=b](std:integral auto n) static // ng
  {
    return n + bias;
  }:
}
```

### static `operator[]` 

- P2589R0 static `operator[]`(https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2589r0.pdf)

前述のように、C++23の`operator[]`の`operator()`との違いは演算子の見た目以外にはありません。そのため、`operator()`と同様の理由で`operator[]`も`static`でオーバーロードできるようになります。

```cpp
template<typename T>
struct eq_t {

  // static operator()
  static bool operator()(const T& l, const T& r) {
    return l == r;
  }

  // static operator[]
  static bool operator[](const T& l, const T& r) {
    return l == r;
  }
};

int main() {
  eq_t<int> eq{};

  bool b1 = eq(23, 23); // ok、static operator()が呼ばれる
  bool b2 = eq[20, 23]; // ok、static operator[]が呼ばれる
}
```

とはいえ`[]`の意味を考えるとあまり使用機会は多くないかもしれません。

なお、提案分かれているのは`operator[]`の複数引数許可と`static operator()`が同時に進行していたためで、`static operator()`が議論されていた段階ではまだ`operatpr[]`はC++20までの仕様のままだったためです。

# 定数式

## if consteval

- P1938R3 `if consteval`(https://wg21.link/P1938R3)

C++20では`std::is_constant_evaluated()`によってコンパイル時に実行される文脈と実行時に実行される文脈を区別できるようになりました。しかし、この関数の利用時にはいくつか注意点があります。

まず1つは、`std::is_constant_evaluated()`を`constexpr if`と共に使用してしまう場合の注意点です。この場合は常に`std::is_constant_evaluated()`は`true`を返すため、意図通りになりません。

```cpp
#include <type_traits>

constexpr int f() {
  if constexpr (std::is_constant_evaluated()) {
    return 20;
  } else {
    return 0;
  }
}

int main() {
  // コンパイル時
  constexpr int n = f();
  // 実行時
  int m = f();
  
  std::cout << n << '\n' << m << std::endl;
  // 20
  // 20
}
```

`std::is_constant_evaluated()`はコンパイル時に呼び出されたときに`true`を返し実行時に呼び出されると`false`を返す関数、ではなく、コンパイル時に呼び出されることが確実な特定のコンテキストで呼ばれている場合に`true`を返す関数です（このことは2つ目の問題にも関わってきます）。`constexpr if`の条件式はそのコンテキストに該当するため、ここに書かれていると常に`true`を返します。

これは誤用ですが理由を知らなければ気づき辛く、コンパイラも誤用を検出しづらいという点が問題でした。

2つ目の注意点は、`std::is_constant_evaluated()`を用いた分岐内で`consteval`関数を呼び出す場合の問題です。このような呼び出しは常にエラーとなっていました。

```cpp
consteval int f(int i) { return i; }

constexpr int g(int i) {
  if (std::is_constant_evaluated()) {
      return f(i) + 1; // ng
  } else {
      return 42;
  }
}

consteval int h(int i) {
  return f(i) + 1;  // ok
}
```

`consteval`関数は必ずコンパイル時に呼ばれる関数であり、コンパイル時に呼び出されない呼び出しはエラーとなります。しかし、`g()`は`h()`を実行時でも呼び出せるように変更しただけのもので、`consteval`関数`f()`は変わらずコンパイル時にしか実行されないコンテキストで呼ばれているので一見すると問題ないように見えます。

しかし実際には`g()`内での`f(i)`の呼び出しはどこに書いたとしてもエラーになります。これは、`constexp`関数の引数が定数式として使用できないことによって起こるエラーですが、`consteval`関数の場合はその本体の全体がコンパイル時に呼び出されることが確実なコンテキストであるため、その制約を受けなくなっているためエラーになりません。

`if (std::is_constant_evaluated()) {}`の`true`の分岐はそのようなコンテキストに無く、ライブラリ関数を呼び出している普通の`if`文であるためそのような特別扱いを望むことはできません。とはいえこのことも理由を知らなければなぜエラーになっているのかが分からず、その理由は難解です。

C++23では`if (std::is_constant_evaluated())`のこれらの問題を解決するために、この構文糖衣である新しいタイプの`if`文である`if consteval`が追加されます。

```cpp
constexpr int f() {
  if consteval {
    return 20;  // コンパイル時の処理
  } else {
    return 0;   // 実行時の処理
  }
}
```

その効果は`if (std::is_constant_evaluated())`と同様であり、ただ短くなっただけに見えますが次のような違いがあります

- `<type_traits>`のインクルードが必要ない
- 構文が異なるため、誤用や誤解のしようがない
    - `constexpr if`で間違って使用は起こり得ない
    - コンパイル時に評価されているかをチェックする適切な方法についての混乱を完全に解消できる
- `if consteval`を使用して`consteval`関数を呼び出すことができる

まず1つ目の問題点が解消されていることはすぐに分かります。2つ目の問題点は、`if consteval`の`true`となるブロック（つまりコンパイル時に実行される方の分岐）の内部が必ずコンパイル時に実行されるコンテキストとして扱われることで、その内部で`consteval`関数の呼び出しがほぼ自由に行えるようになり、解消されます。

```cpp
constexpr int g(int i) {
  if consteval {
    // このブロック内は必ずコンパイル時に実行されるコンテキスト
    return f(i) + 1; // ok
  } else {
    return 42;
  }
}
```

つまり、`if consteval`の`true`となるブロックは`consteval`関数本体内と同じ扱いを受けており、これによってこの内部からならば`constexpr`関数の引数も定数式として使用することができ、`consteval`関数がその引数を読み取ることができます。

ちなみに、条件を反転させたい場合は`consteval`の前に`!`を入れます。このとき、代替トークンを使用することもできます。

```cpp
int f(int n) {
  if !consteval {
    // 実行時はこっちにくる
    return 0;
  }

  // コンパイル時にこっちにくる
  // この場合、コンテキストの特別扱いはない
  ...
}

int f2(int n) {
  // こう書いてもok
  if not consteval {
    // 実行時
    return 0;
  }

  // コンパイル時
  return 1;
}
```

## 一部の定数式の文脈での `bool` への縮小変換を許可

- P1401R5 Narrowing contextual conversions to bool(https://wg21.link/P1401R5)

定数式においては整数型から`bool`型への縮小変換は禁止されており、それが行われるとコンパイルエラーになります（ただし、整数値が`0`か`1`の場合は縮小変換にならないためエラーにはなりません）。特に、`static_assert`や`constexpr if`の条件式においてもこれはエラーにされていました

```cpp
enum Flags { Write = 1, Read = 2, Exec = 4 };

template <Flags flags>
int f() {
  if constexpr (flags & Flags::Exec) // 縮小変換が起きるとコンパイルエラー
  // if constexpr (bool(flags & Flags::Exec)) とするとok
    return 0;
  else
    return 1;
}

template <std::size_t N>
class Array {
  static_assert(N, "no 0-size Arrays"); // 縮小変換が起きるとコンパイルエラー
  // static_assert(N != 0); とするとok

  // ...
};

int main() {
  int n = f<Flags::Exec>();  // ng
  Array<16> a;  // ng
}
```

このような条件式における整数型から`bool`への縮小変換は実行時のアサーションや`if`文では普通に行われており、コンパイルエラーも実行時エラーも起こさずに行うことができます。このことは、実行時とコンパイル時で条件式を記述する際に余分に注意しなければならず、一貫性がありませんでした。

そのため、C++23では`static_assert`や`constexpr if`の条件式に限って整数型から`bool`型への縮小変換が許可されます。これにより先程のサンプルコードは修正なしでコンパイルが通るようになり、なおかつ意図通りに動作します。

ただし、これ以外の文脈（例えば`explicit(expr)`や`noexcept(expr)`など）では引き続き許可されず、`1`か`0`からの変換しか行えません。

余談ですが、そもそも`static_assert`や`constexpr if`の条件式でさえも`bool`への縮小変換が禁止されていたのは、`noexcept`式での縮小変換を禁止した時に巻き込まれてしまったためのようです。

関数`f()`の例外仕様が別の関数`g()`に従う場合、`noexcept`を2つ重ねたうえで内側に`g()`の呼び出し式を書く必要があります。しかし、その場合に書き間違えて関数名だけを書いたり、`noexcept`を1つにしてしまった時に`g()`が`constexpr`関数だったりすると、思わぬバグを踏むことがあります。

```cpp
// int型を返す関数がある
constexpr int g() noexcept(...) { ... }

// 例外を投げるかどうかがg()による関数
int f() noexcept(noexcept(g()))  // noexcept指定の中にnoexcept式を書く
{
  ...

  int n = g();  // g()を内部で呼び出している

  ...
}

int f() noexcept(g);    // 関数ポインタからの暗黙変換、エラーにならない
int f() noexcept(g());  // 定数評価の結果bool値へ変換されるとエラーにならない
```

このような小さいながらも気づきにくいバグを防ぐために、定数式での文脈的な`bool`変換の際の縮小変換は禁止されました。その仕様変更（C++14～17のタイミング）は本来、`noexcept`式にだけ適用するつもりで`static_assert`でまで禁止するつもりはなかったようですが、その意図に反して両方で縮小変換が禁止されてしまい、その文言を踏襲する形で`constexpr if`でも禁止されてしまっていたようです。

## `static_assert()`の診断メッセージの文字エンコーディングの制限緩和

- P2246R1 Character encoding of diagnostic text(https://wg21.link/P2246R1)

`static_assert()`には2つ目の引数に診断メッセージを文字列リテラルとして渡して、条件が`false`になった場合にコンパイラのエラーメッセージとしてその文字列を出力させることができます。

C++20までの`static_assert()`の規定では、ソースコードのエンコーディングと実行時のエンコーディングが異なっている場合に実行時エンコーディングで表現可能ではない文字は出力する必要は無い、のように規定されていました。

しかし、診断メッセージの出力は実行時ではなくコンパイル時にコンパイラの実行環境のエンコーディングによって行われ、かつソースコードの文字列のエンコーディングとコンパイル時のコンパイラのメッセージ出力先（ターミナル）のエンコーディングには直接的な関係はありません。すなわち、`static_assert()`のこのような規定はほぼ無意味なものとなっていました。

さらに、`[[deprecated]]`や`#error`などのコンパイル時に診断メッセージを出力する同様の機能にはそのような規定は備わっていなかったこともあり、`static_assert()`にあったエンコーディングに関する規定は削除されることになりました。

ただし、診断メッセージの文字がコンパイラの実行環境のターミナルのエンコーディングで表示できない場合にどのように出力されるかは、実装品質の問題とされ、どのように出力されるかが指定されるようになるわけではありません。

## 定数式内での非リテラル変数、静的変数・スレッドローカル変数およびgotoとラベルの存在を許可する

- P2242R3 Non-literal variables (and labels and gotos) in constexpr functions(https://wg21.link/P2242R3)

`constexpr`関数内部では通常定数式で実行できなさそうなものは禁止されており、書かれているだけでコンパイルエラーになります。一方で、C++20では`std::is_constant_evaluated()`によってコンパイル時に呼ばれる文脈と実行時に呼ばれる文脈を分けて書くことができるようになり、1つの関数定義でコンパイル時と実行時の処理を一緒に書くことができるようになっています。

```cpp
template<typename T>
constexpr bool f() {
  if (std::is_constant_evaluated()) {
    // .コンパイル時の処理
    ...
    return true;
  } else {
    T t;  // コンパイル時に評価されない
    ...
    return true;
  }
}

struct nonliteral { nonliteral(); };

static_assert(f<nonliteral>()); // ng（ただしエラーにしないコンパイラもある
```

この例では`T`がリテラル型ではない型（定数式で構築できない型）の場合コンパイルエラーを起こす場合があり（コンパイラによって異なる）、規格的にはエラーになるのが正しい動作となります。しかしこの場合、エラーになる文は実行時の文脈に書かれており、コンパイル時に実行されることはありません。

`std::is_constant_evaluated()`を活用することでコンパイル時と実行時で関数の実装をより共有することができ、C++20ではそれを促進するために`constexpr`関数において`throw`式やインラインアセンブリが書かれていてもコンパイル時にそこに到達しなければエラーにしないようになりました。

C++23ではそれがさらに促進され、`constexpr`関数に定数式で構築できない変数の宣言や静的ストレージ期間を持つ変数の宣言、および`goto`文やそのラベルが書かれていてもそこに到達しなければエラーにならなくなりました。

```cpp
template<typename T>
constexpr bool f() {
  if (std::is_constant_evaluated()) {
    // .コンパイル時の処理
    ...
    return true;
  } else {
    T t;  // ok、常にエラーにならない
    ...
    return true;
  }
}

constexpr int example1(int n) {
  if (23 <= n) {
    static int v = n;   // ok（コンパイル時に到達しなければ
    thread_local t = n; // ok（コンパイル時に到達しなければ

    return v + t;
  }

  return n;
}

constexpr int example2(int n) {
  int result = n;

  if (!std::is_constant_evaluated()) {
    // 実行時の処理
    ...
    goto label; // ok
  }

  // コンパイル時の処理
  ...

label:  // ok、コンパイル時に通過する場合もエラーにならない

  return result;
}
```

ただし、あくまで`constexpr`関数定義内にこれらのものが書かれているだけではエラーにならなくなっただけで、コンパイル時に実行できるようになったわけではありません。特に、`goto`文がコンパイル時に使えるようになるわけではありません（おそらく将来的にも）。

## 定数式における未知の参照の利用の許可

- P2280R4 Using unknown pointers and references in constant expressions(https://wg21.link/P2280R4)

配列に対する`std::size()`の適用をコンパイル時に行おうとすると、たまに不可思議な現象に出会うことがあります。

```cpp
void check(int const (&param)[3]) {
  int local[] = {1, 2, 3};

  constexpr auto s0 = std::size(local); // ok
  constexpr auto s1 = std::size(param); // ng
}
```

この`s0, s1`はともに`constexpr`ローカル変数であり、その初期化式（ここでは`std::size()`の呼び出し）は定数式でなければなりません。しかし、ここでは変数`local`は定数式として扱われるのに対して、変数`param`は定数式として扱われておらず、それによってエラーになります。

これは、`param`は定数式で使用できない参照の条件に当てはまっているため使用できず、それは関数引数`param`が初期化されているかどうかが不明であるためです。ただ、`std::size()`の配列（`std::array`含む）についての実装は配列そのものに一切アクセスせず、その型情報からそのサイズを取得します。そのため、この場合参照`param`が初期化されていようがいまいが関係ないはずです。

同じことは、暗黙の`this`引数においても出会うことがあります。

```cpp
struct S {
  int buffer[2048];

  void foo() {
    constexpr size_t N = std::size(buffer); // ng

    ...
  }
};
```

この場合のエラーは、`std::size(buffer)`の引数において`this`が定数式で使用できないために起こっています。ここの`buffer`はメンバ変数なので、その参照には`this`が必要ですが、このコンテキスト（`N`の初期化式）での`this`は定数式ではないため定数式で使用できません。なお、この場合に`foo()`を`constexpr`にしても変わりません。

また、最初のコードを`std::array`と`.size()`メンバ関数呼び出しに置き換えると、`this`による同様のエラーに出会うことができます。

```cpp
void check(const std::array<int, 3>& param) {
  std::array<int, 3> local = {1, 2, 3};

  constexpr auto s0 = local.size(); // ok
  constexpr auto s1 = param.size(); // ng
}
```

この`this`でエラーの起きている例はいずれも、`this`を介して（間接参照して）アクセスしようとしているわけではありません。

その他のケースでも稀に同様のエラーに出会うことがあるかもしれません。しかし、このようなエラーにおいては一貫して、定数式のある局所的なコンテキストにおいて初期化されているかどうかが未知の参照を使用（読み取り、コピー）しようとしているものの、その参照の参照先を取得しようとはしていません。C++20までの規定では、定数式での参照の安全性担保のためにそのような未知の参照の使用そのものを一切禁止していました。

しかし、ここまで見てきたようなエラーになっている例はその制限によって引き起こされる副作用であり、それがなぜエラーになるのかが非常に分かりづらいものであったため、C++23では未知の参照と`this`の定数式における単なる使用（読み取り、コピー）が許可されます。これによって、上記のエラーになっていた例はそのままエラーにならなくなります。ただし、未知の参照や`this`の間接参照が許可されたわけではなく、あくまでそれらを読み取って別の関数に渡したり、`this`を介さない形でメンバアクセスしたりすることが許可されるのみです。

同様の問題はポインタでも起こるのですが、ポインタの場合は可能な操作が多いことから緩和作業が複雑になるため、この提案では未知の参照と`this`の単なる使用（読み取り）を許可するに留めています。

そして、この提案の修正はC++11に対する欠陥報告であるため、これを実装したコンパイラでは以前の言語バージョンでもこの問題が修正されます。

### メタプログラミングにおける影響

未知の参照というものは基本的に関数引数で遭遇することが多いと思われますが、関数引数ではない場所、特に`requires`式の引数の参照もその対象となります。

```cpp
template<typename T>
concept to_char_narrowing_check = requires(T&& t) {
  { std::int8_t{ t.size() } };  // -128 ~ +127 の範囲内はtrue
};

static_assert(to_char_narrowing_check<std::array<int, 1>>);
static_assert(to_char_narrowing_check<std::array<int, 127>>);
static_assert(to_char_narrowing_check<std::array<int, 128>> == false);
```

この`to_char_narrowing_check`コンセプトは、`std::int8_t{ t.size() }`という式の有効性をチェックしています。引数型がすべて`std::array`であるとすると、その`.size()`の戻り値型は`std::size_t`なので常に縮小変換となり、`{}`初期化では縮小変換が許可されないことから常にコンパイルエラーになります（この提案以前は）。ただし、このような縮小変換は定数式で行われた場合にのみ、変換元の値が変換先の型で表現可能であれば許可されます。

ただ前述のように、`std::array::size()`は`this`ポインタのコピーが必要となり、それがC++20までは定数式ではなかったので定数式における縮小変換のチェックは行われませんでした。C++20以前（正確にはP2280R4以前）は、この例の上2つの`static_assert`が満たされることはありません。しかし、C++23以降（正確にはP2280R4以降）はこの例の上2つの`static_assert`は満たされるようになります。

何が起きているかというと、`requires`式内部の`std::int8_t{ t.size() }`の式の妥当性チェックの際に、定数式で縮小変換が可能かどうかがチェックされており、P2280R4の緩和によってそれを妨げるものが無くなった（`this`のコピーが定数式で可能になった）ことで、`t.size()`の値が取得されてその値がチェックされるようになっています。

ただしここでは、`array`のオブジェクト（`t`の参照先）は具体的に使用されておらず、`t.size()`の値は`t`の型情報から取得されています。

コンセプトを用いないSFINAEにおいては、`requires`式の引数のような参照は`std::declval()`によって生成していました。`std::declval()`は定義が無い関数であるため通常定数式で使用できないため、このような定数式での縮小変換チェックは行えません。しかし、`std::declval()`の使用を避けてうまく関数引数の参照によってSFINAEの検知を行うようにすると、同様に定数式での縮小変換チェックを有効化することができます（実装が複雑になるため省略します、気になる人はP2280R4でググるとこれについて詳しく書いてあるブログ記事が出てきます）。

なお前述のように、この提案P2280R4は欠陥報告としてC++11に適用されているため、実装済みのコンパイラであればC++20以前の言語モードでもこれらの恩恵を受けることができます。

## `constexpr`関数が定数実行できない場合でも適格とする

- P2448R2 Relaxing some `constexpr` restrictions(https://wg21.link/P2448R2)

C++20の`constexpr`関数の仕様ではその関数のあらゆる入力に対して定数式で実行できない場合、コンパイルエラーになります。

```cpp
extern int f(int);  // 定義が無い関数

constexpr int g(int n) {
  return f(n);  // ng、f()はコンパイル時に実行できない
}

constexpr int h(int n) {
  std::cout << n; // ng。コンパイル時に実行できない
  return n;
}
```

このような関数はたとえ実行されていなくても書いてあるだけでコンパイルエラーになります。

しかし、標準ライブラリ機能を呼び出している関数などではバージョンによってライブラリ機能が`constexpr`であるかが異なることがあります。

```cpp
constexpr void f(std::optional<int>& o) {
  o.reset();  // C++20まで非constexpr
}
```

`std::optional::reset()`はC++20までは`constexpr`指定されていなかったたため定数式で使用できず、この関数はエラーになります。しかし、C++23からは`constexpr`指定されているため定数式で使用でき、エラーにはなりません。これはC++20で共用体のアクティブメンバ切り替えが定数式で可能になったことでC++23からこの操作も定数式で可能になりました。このように、関数内で使用されているある機能（言語/ライブラリ）はある時点で定数式で使用可能ではなくても、将来的には定数式で使用可能になる可能性があります。特に、C++11以降多くの機能が積極的に定数式対応されて来ました。

複数の言語バージョンをターゲットとしてコンパイルされるるプログラムやライブラリなどでは、このような関数を`constexpr`指定しておくのは困難です。言語機能か標準ライブラリの場合は機能テストマクロが提供されていますが、在野の多くのライブラリはそのようなものを用意しません。機能テストマクロを使って分岐したとしても、機能ごとに細かくどのバージョンから定数式で使用可能になったのかを把握して調整しなければならず、その作業はあまり現実的ではありません。

このようなエラー方針は、定数式で実行可能なものが大きく制限されていたC++11の時代には有用だったのかもしれませんが、より多くのものが定数式で実行可能となった現在（C++23）の環境ではむしろ足枷となっていました。そのため、C++23からはある`constexpr`関数が仮にすべての入力に対して定数式で実行できなくても、実際に実行されるまではエラーにならないようになります。

それに伴って、`constexpr`指定されたクラスの特殊メンバ関数が`constexpr`となる条件も、定義の時点で定数式で使用可能でなければエラーになっていた（例えばメンバ変数の中に対応する特殊メンバ関数が非`constexpr`なものが含まれている場合など）のが、同様に実際にその特殊メンバ関数が定数式で使用されるまではエラーにならなくなります。

```cpp
struct S1 {
  std::list list;

  constexpr S1() = default;  // ok、C++20まではng
};

struct S2 {
  std::any up;  // anyはデフォルトコンストラクタのみconstexpr

  constexpr S2() = default; // ok（C++20でもok

  // 次のいずれかの宣言を追加するとC++20まではng、C++23からは全てok
  constexpr S2(const S2&) = default;
  constexpr S2(S2&&) = default;
  constexpr S2& operator=(const S2&) = default;
  constexpr S2& operator=(S2&&) = default;
  constexpr ~S2() = default;
};
```

このような、コンパイル時に実際に実行してみて実行不可能なものに出会うまではエラーにしないという方針は、C++20の`std::is_constant_evaluated()`の導入以降の定数式制限緩和の暗黙的な方向性でした（インラインアセンブラの許可や`throw`式/`try-catch`ブロックの許可など）。前節の未知の参照利用許可もその一環でしたが、この提案によってその方針が明文化されかつ`constexpr`関数全体に適用されることになります。

## constexpr 関数内での static constexpr 変数を許可

- P2647R1 Permitting `static constexpr` variables in `constexpr` functions(https://wg21.link/P2647R1)

まず、次のような関数を持っているとします。

```cpp
// 数値を文字に変換する
char xdigit(int n) {
  static constexpr char digits[] = "0123456789abcdef";  // ok
  return digits[n];
}
```

この関数は16進数1桁の整数値を対応する英数文字に変換する関数であり、正しく動作します。この関数を定数式でも使用したくなったので、単純に`constexpr`を付加することにしました。

```cpp
constexpr char xdigit(int n) {
  // constexpr関数内でstatic変数を宣言できないためエラー
  static constexpr char digits[] = "0123456789abcdef";  // ng
  return digits[n];
}
```

すると、途端にコンパイルが通らなくなります。

これを正しく回避しようとすると、名前空間（グローバル）スコープに`static constexpr`変数を置いておくか、別の`consteval`関数で定数を返すようにするなどの冗長なワークアラウンドを取らざるを得なくなります。なお、単に`static`を外せばコンパイル時に動作するようにはなりますが、今度は実行時の最適化が困難になる場合があるようで、それを考慮したワークアラウンドはこのような配列をローカルの`constexpr`変数として導入しないようにするものになります。

この制限は`constexpr`関数内で`static`変数の宣言が禁止されていることによってエラーになっています。これは、`static`変数の初期化にまつわる問題（副作用や初期化順序など）を定数式に持ち込まないために禁止されていました。

しかし、`static constexpr`変数の初期化のタイミングは非`constexpr`の静的変数とは異なりコンパイル時に行われるため、`static constexpr`変数には通常の`static`（`thread_local`）変数にあるような初期化の問題は生じません。そのため、C++23ではこの制限が取り払われ、`constexpr`（`consteval`）関数内で`static constexpr`変数が意図通りに使用できるようになります。これにより、上記のコードはそのままコンパイルが通り実行時とコンパイル時の両方で実行できるようになります。

なお、前節の変更によって、非`constexpr`な`static`（`thread_local`）変数も、`constexpr`関数内でコンパイル時にその初期化に到達しない限り宣言しておくことはできます。

## constexpr 関数内で consteval 関数を呼び出せない問題を軽減

https://wg21.link/P2564R0


# テンプレート

## 変数テンプレートの部分特殊化の仕様明確化

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2096r2.html

## 継承コンストラクタからのクラステンプレート引数の推論

https://wg21.link/P2582R1

# 文字・文字列リテラル

## 異なる文字エンコーディングをもつ文字列リテラルの連結を不適格とする

https://wg21.link/P2201R1

## エスケープシーケンスの区切り	

https://wg21.link/P2290R3

## 文字・文字列リテラル中の数値・ユニバーサルキャラクタのエスケープに関する問題解決

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2029r4.html

## 1ワイド文字に収まらないワイド文字リテラルを禁止する

https://wg21.link/P2362R3

## 名前付きユニバーサルキャラクタ名

https://wg21.link/P2071R2

## wchar_t の制限の緩和

https://wg21.link/P2460R2

# ラムダ式

## ラムダ式で () を省略できる条件を緩和

https://wg21.link/P1102R2

## ラムダ式に対する属性	

https://wg21.link/P2173R1

## ラムダ式の後置戻り値型のスコープ変更

https://wg21.link/P2036R3
https://wg21.link/P2579R0

# プリプロセッサ

## 文字リテラルエンコーディングを一貫させる

https://wg21.link/P2316R2

## elif / elifdef / elifndef のサポートを追加

https://wg21.link/P2334R1

## #warning のサポートを追加	

https://wg21.link/P2437R1

## 汎用的なソースコードのエンコーディングとしてUTF-8をサポート

https://wg21.link/P2295R6

# その他

## decay-copy

https://wg21.link/P0849R8

## size_t リテラルのためのサフィックス

https://wg21.link/P0330R8

## 暗黙ムーブを簡略化

https://wg21.link/P2266R3

## コード内容の仮定をコンパイラに伝えるassume属性

https://wg21.link/P1774R8

## 初期化文での型の別名宣言を許可

https://wg21.link/P2360R0

## 範囲for文が範囲初期化子内で生じた一時オブジェクトを延命することを規定

https://wg21.link/P2718R0

## 拡張不動小数点型のサポート

https://wg21.link/P1467R9

# とてもマイナーな変更

## スコープと名前ルックアップの仕様整理

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p1787r6.html

## 複合文の末尾へのラベルを許可

https://wg21.link/P2324R2

## 参照するPOSIX規格を更新

http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2227r0.html

## 行末スペースを無視するよう規定

https://wg21.link/P2223R2

## GCサポートの削除

https://wg21.link/P2186R2

## P2314R0 Character sets and encodings

https://wg21.link/P2314R4

# Defect Report

## 無意味なexport宣言を禁止する

https://wg21.link/P2615R1

## 識別子に使用可能な文字の制限

https://wg21.link/P1949R7

## 属性指定の重複の許可

https://wg21.link/P2156R1

## P2943R0

https://wg21.link/P2493R0

## ==演算子の導出の調整

https://wg21.link/P2468R2

## char8_t互換性の修正

https://wg21.link/P2513R4

## CWG2518

https://cplusplus.github.io/CWG/issues/2518.html

\clearpage

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- Compiler Explorer(https://godbolt.org/)
