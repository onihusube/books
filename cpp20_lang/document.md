---
title: C++20 コア言語機能
author: onihusube
date: 2022/08/13
geometry:
  width: 188mm
  height: 263mm
coverimage: cover.jpg
backcoverimage: backcover.jpg
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
  revision: 第2版
  printing: ねこのしっぽ
---
\clearpage

# はじめに

本書はC++20で新規に導入された言語機能を紹介するものです。C++20はC++11以来とも言われる大型アップデートとなっており、その全景を把握するのは困難です。本書はその一助となるべく書かれました。各機能を大まかなグループで章立てし、個別機能の節ではその機能がなぜ必要になったのか？どういう問題を解決するのか？という点を重視してサンプルコードと共に解説しています。全体を通してはある程度の事前知識を必要としますが、すべての機能が複雑ではないはずです。知らない概念や用語・機能については適宜検索されると日本語で解説が見つかるはずで、見つかりそうにないものについては簡単に解説しています。

言語機能に的を絞っているためライブラリ機能についてはほとんど触れていません。また、C++17以前に追加された機能がいつ頃追加されたのかも詳しく記していません。お手数ですが、cpprefjpなどで適宜調べてください。

以下では、いくつかの用語の紹介をしておきます。

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

## ok/ng

サンプルコード中では、特定の行の末尾のコメントで`ok/ng`と表示することで、それがコンパイル可能かどうかを示しています。`// ok`はコンパイルが通り未定義動作がないこと、`// ng`はコンパイルエラーとなる事をそれぞれ表しています。ただし、どの言語バージョン（C++17 or 20）で`ok/ng`なのかは文脈によっている事があります。

\clearpage

# 大きな機能

## `<=>`

`<=>`（三方比較演算子、宇宙船演算子などと呼ばれる）は新しく追加される比較演算子で、`a <=> b`のように使用して`a, b`の大小関係を表す1つの値を返します。その値からは、`0`との比較によって大小関係を取り出すことができます。

```cpp
#include <compare>

int main() {
  int a = 10;
  int b = 20;

  auto comp = a <=> b;

  if (comp < 0) {
    std::cout << "a < b"; // ここに来る
  } else if (comp > 0) {
    std::cout << "a > b";
  } else if (comp == 0) {
    std::cout << "a = b";
  }
}
```

これだけ見ると利用が煩雑な比較演算子が一個増えただけで意味が分からないですが、`<=>`の真価はクラス型に対して定義した場合に発揮されます。

```cpp
#include <compare>

struct C {
  int n;
  double d;

  // <=>のdefault定義
  auto operator<=>(const C&) const = default;
};

int main() {
  C a{ 10, 0.0 };
  C b{ 10, 1.0 };

  // 以下6種類の比較が可能となる
  std::cout << (a == b) << '\n';
  std::cout << (a != b) << '\n';
  std::cout << (a <  b) << '\n';
  std::cout << (a <= b) << '\n';
  std::cout << (a >  b) << '\n';
  std::cout << (a >= b) << '\n';
}
```

クラス型に対して`<=>`を定義しておくと`< <= > >=`の4つの演算子が自動で実装され、それが`dafault`の場合は`== !=`の2つも自動で実装されます。そして、これらの自動実装される比較演算子は`<=>/==`を利用して実装されています。`== !=`だけ扱いが異なるのは、`!=`は`==`演算子を用いて実装されていて、`<=>`が`default`で宣言されている場合は暗黙的に`==`も`default`定義されているためです。

```cpp
struct C {
  int n;
  double d;
  char str[3];

  // <=>のdefault定義
  auto operator<=>(const C&) const = default;
  // ==のdefault定義（暗黙）
  //bool operator==(const C&) const = default;
};

int main() {
  C a{ 10, 0.0, "te" };
  C b{ 10, 1.0, "te" };

  // <=> ==を用いて次のように実装されている
  std::cout << (a == b) << '\n';
  std::cout << (!(a == b)) << '\n';
  std::cout << (a <=> b < 0)  << '\n';
  std::cout << (a <=> b <= 0) << '\n';
  std::cout << (a <=> b > 0)  << '\n';
  std::cout << (a <=> b >= 0) << '\n';
}
```

自動実装に使用されている`default`の`<=> ==`もまた自動実装されており、最小のコードで比較演算子自動実装が可能となっています。この`<=> ==`の`default`定義をあえて明示的に書いてみると、次のようになります

```cpp
struct C {
  int n;
  double d;
  char str[3];

  // <=>のdefault定義の展開
  auto operator<=>(const C& that) const {
    if (auto cmp = n <=> that.n; cmp != 0) return cmp;
    if (auto cmp = d <=> that.d; cmp != 0) return cmp;
    if (auto cmp = str[0] <=> that.str[0]; cmp != 0) return cmp;
    if (auto cmp = str[1] <=> that.str[1]; cmp != 0) return cmp;
    return str[2] <=> that.str[2];
  }

  // ==のdefault定義の展開
  bool operator==(const C&) const {
    if (bool cmp = n == that.n; cmp != true) return cmp;
    if (bool cmp = d == that.d; cmp != true) return cmp;
    if (bool cmp = str[0] == that.str[0]; cmp != true) return cmp;
    if (bool cmp = str[1] == that.str[1]; cmp != true) return cmp;
    return str[2] == that.str[2];
  }
};
```

`<=> ==`どちらでも同様に、各メンバを宣言順に`<=> ==`で比較していて、配列メンバは展開され要素ごとの比較となります。もしここに基底クラスがある場合、最初に基底クラスがその継承順で同様に比較されます。当然ですが、その中の一つに`<=> ==`で比較できないものがあるとこの実装は成り立たず、その場合`default`定義は暗黙`delete`されます。

こうしてみてみると、`int`等の基本型に`<=>`が用意されているのはこの`default`実装のためであることが分かります。`<=>`は普段使いする演算子というよりは、比較演算子自動実装のためのマーカーのようなものとしての使用がメインでしょう。

ここまで`auto`でぼかしてきましたが、`<=>`の戻り値型は`bool`でなければ`int`等の整数型でもなく、`0`リテラルのみと比較可能な特殊な型で、比較カテゴリ型と呼ばれます。比較カテゴリ型は`<compare>`ヘッダに3種類が用意されています。なお、`<compare>`は自動でインクルードされたりしないので明示的にインクルードする必要があります。

|比較カテゴリ型名|対応する順序関係|属する基本型|
|---|---|---|
|`std::strong_ordering`|全順序|整数型、列挙型、ポインタ型|
|`std::weak_ordering`|弱順序|なし|
|`std::partial_ordering`|半順序|浮動小数点数型|

3つの比較カテゴリ型はそれぞれ数学的な順序関係と対応しており、ある比較カテゴリ型を返す比較はそれに対応する順序関係の上での比較（順序付け）となることを表しています。

- 全順序 : すべての値の間に順序をつけることができる
- 弱順序 : 一部の値の間で順序付けできない（比較不可能な）ペアが存在する
- 半順序 : 他の全ての値と順序付けできない（比較不可能な）値が存在する

浮動小数点数型の比較が半順序となるのはあらゆる値と比較不可能な`NaN`が存在するためです。`weak_ordering`（弱順序）となる比較には例えば、大文字小文字を識別しない文字列の大小比較があります。


### 異種型間比較と比較演算子の自動生成

`<=>/==`からの比較演算子自動実装は、各比較演算子を使用した時のオーバーロード候補に`<=> ==`を使用した候補を挿入し、最終的に比較演算子の書き換えを行うことによって行われています。

型`T1, T2`のオブジェクト`a, b`に対して、`a @ b`（`@`は任意の比較演算子を表す）のように比較演算子が呼ばれたとき

1. オーバーロード候補に、`a @ b`を加える
2. `@`が`<=>`ならば、オーバーロード候補に`<=>`を使って生成した逆順の式`b <=> a`を加える
3. `@`が`< <= > >=`のいずれかならば、オーバーロード候補に`<=>`を使って生成した2種類の式`a <=> b, b <=> a`を加える
4. `@`が`!=`ならば、そのオーバーロード候補に`==`を使って生成した2種類の式`a == b, b == a`を加える
5. `@`が`==`ならば、そのオーバーロード候補に`==`を使って生成した逆順の式`b == a`を加える
6. オーバーロード解決時、`@`→正順の式→逆順の式 の順で優先される
7. オーバーロード解決の結果生成された候補が選択された時、その候補に応じて次のように最終的な式を生成する
    - `@`が`<=>`ならば、`0 <=> (b <=> a)`
    - `@`が`< <= > >=`のいずれかならば、`(a <=> b) @ 0`もしくは`0 @ (b <=> a)`
    - `@`が`!=`ならば、`!(a == b)`もしくは`!(b == a)`
8. 元の呼び出し`a @ b`を7で生成された式で書き換える

複雑ではありますが、この仕組みによって`T1`と`T2`が異なる場合の比較（異種型間比較）においても、片方の（例えば`T1, T2`の順に引数をとる）演算子さえ定義しておけば、逆順（`T2, T1`の順）の比較演算子も自動で導出されます。

```cpp
struct H {
  int n;

  // 自身の比較
  auto operator<=>(const H&) const = default;
  bool operator==(const H&) const = default;  // 明示的定義が必要

  // intとの比較
  friend auto operator<=>(const H& self, int m) {
    return self.n <=> m;
  }

  // こっちも定義しなければならない
  friend bool operator==(const H& self, int m) {
    return self.n == m;
  }
};

int main() {
  H h = {10};
  int n = 20;

  // 正順、ok
  std::cout << (h != n) << '\n';
  std::cout << (h <  n) << '\n';
  std::cout << (h <= n) << '\n';
  std::cout << (h >  n) << '\n';
  std::cout << (h >= n) << '\n';
  
  // 逆順、ok
  std::cout << (n != h) << '\n';
  std::cout << (n <  h) << '\n';
  std::cout << (n <= h) << '\n';
  std::cout << (n >  h) << '\n';
  std::cout << (n >= h) << '\n';
}
```

このように、異種型間比較を行う場合でも最小の演算子の定義（2×2=4）で合計18個の演算子が利用可能となります。

ただしこの場合、`default`定義されていない`<=>`があるので自身の比較のための`==`が暗黙定義されなくなり、異種型間比較のための`==`は`default`定義できません。そのため、2つの`==`は明示的に書かなければなりません。とはいえ、自身の比較の`==`は`default`定義可能なので、比較そのものを書く必要があるのは異種型間比較のためのものだけです。

### 符号付整数型の内部表現は2の補数であると規定

- P0907R4 Signed Integers are Two's Complement (https://wg21.link/p0907r4)

C++の重箱の隅に詳しい人は、符号付整数型の内部表現は規定されていないので2の補数とは限らず、符号付整数型の比較は全順序とはかぎらないはず！と思われるかもしれません。

それはC++17まではその通りだったのですが、ハッシュ値の簡易な計算のためやこの`<=>`のためなどの理由と、実際には2の補数以外を使用するC++処理系の実装が無かったことから、C++20からは符号付整数型の内部表現は2の補数であると規定されるようになります。

これによって、整数型は符号付も符号なしも含めて全順序の上での比較が可能となります。

\clearpage

## コンセプト

テンプレートパラメータには任意の型を渡すことができ、それを制限したり条件付けしたりする方法（これをテンプレートパラメータの制約と言います）は提供されていませんでした。そのため、テンプレートなライブラリの使用者は予めそのテンプレートエンティティのテンプレートパラメータに要求されていることを知った上で利用することが求められており、テンプレートライブラリの利用を難しくしていました。また、SFINAEと呼ばれる高度で複雑なテクニックを駆使すればテンプレートパラメータに制約を行うことはできましたが、それはテンプレートを正しく書く事を困難にしコードの可読性を低下させていました。コンセプトはそのようなテンプレートパラメータに対する制約を簡単に行い、また制約を簡単に書けるようにするためのSFINAEに変わる言語機能です。

コンセプトによる制約を行うには、予め定義されたコンセプト（ややこしいですが、これは事前定義された制約条件の事を指します）をテンプレートパラメータの`class/typename`の部分に指定する事で行います。

```cpp
// いくつかの基礎的なコンセプトが提供されている
#include <concepts>

template<std::integral T>
void f(T);  // (1)、整数型用

template<std::floating_point T>
void f(T);  // (2)、浮動小数点数型用

int main() {
  f(10);  // (1)が呼ばれる
  f(1.0); // (2)が呼ばれる
}
```

（ここでは関数定義を省略しています）

ここで使用されている`std::integral`と`std::floating_point`の2つのコンセプトは`<concepts>`ヘッダにて提供されている標準コンセプトで、それぞれ型が整数型、浮動小数点数型である事を制約するものです。このように、`template<C T>`のようにコンセプト`C`によって制約されたテンプレートパラメータ`T`は、`T`の型が`C`の制約を満たしていない場合にオーバーロードの候補から外され、それによって選択されなくなります（この時呼び出せる候補がない場合はコンパイルエラーとなります）。

コンセプトによる制約はまた、`requires`節と呼ばれる構文を用いても行えます。

```cpp
#include <concepts>

// 前置requires節
template<typename T>
  requires std::integral<T>
void f1(T);

template<typename T>
  requires std::floating_point<T>
void f1(T);

// 後置requires節
template<typename T>
void f2(T) requires std::integral<T>;

template<typename T>
void f2(T) requires std::floating_point<T>;
```

最初の例の`f()`とここでの`f1(), f2()`は構文は違えど制約としての効果は同一となります。基本的に、`class/typename`の部分にコンセプトを指定する制約は制約が1つだけの場合に使用し、`requires`節は制約が複数ある場合に使用します。

```cpp
template<std::integral T>
  requires (sizeof(T) == 4)
void f(T);  // 整数型かつ4バイト

template<typename T>
  requires std::integral<T> || std::floating_point<T>
void g(T);  // 整数型か浮動小数点数型

template<typename T>
  requires (!std::integral<T>)
void h(T);  // 整数型ではない

template<std::copyable T>
  requires std::equality_comparable<T> && std::convertible_to<T, int>
void i(T);  // コピー可能かつ、==/!=による比較が可能かつ、intに変換可能

template<std::copyable T>
  requires std::equality_comparable<T>
void j(T) requires std::convertible_to<T, int>;  // i()と同じ
```

`requires`節において複数の制約を行う場合は、`&&`（かつ）`||`（または）によって制約を繋いで記述し、`!`（否定）も使用できます。複雑な事をしなければ、これは通常の論理演算と一致した結果になりますが、ド・モルガンの法則が成り立たないなど直感的でない部分があります。特に、`||`が入った場合のコンセプトによる関数呼び出しの優先順位付けは非常に難解なので、基本的にはそうしたことは避けたほうが無難です。

`f()`の制約で行っているように、`requires`節では制約条件としてコンセプトだけではなく定数式で`bool`値に評価される任意の式を使用できます。コンセプトを単に`&& ||`で繋いでいく分には気にしなくていいのですが、`!`とか`==`などが制約条件に直接現れる場合は`()`が必須となります（上記`f(), h()`）。このように制約条件を記述した定数式で`bool`値に評価される式のことを制約式と呼びます。

`class/typename`の部分の指定と`requires`節を同時に使用した場合は、それぞれに指定されている制約を`()`で括って`&&`でつないだ上で`requires`節に書いたようになります。

また、後置`requires`節は特に、クラステンプレートの非テンプレートメンバ関数を制約するのに用いることができます。

```cpp
template<typename T>
struct S {
  T t;

  // Tがムーブ可能なら有効化
  auto move_get() && requires std::movable<T> -> T;

};
```

ちらっと出ていた`convertible_to`コンセプトがそうであるように、コンセプトのテンプレート引数は1つではなく、2つ以上のテンプレート引数を取ることができます。それを`requires`節で使用する場合は迷うことはないと思いますが、`class/typename`の部分には使用できなさそうに見えます。しかし、2つ以上の引数を取るコンセプトでも`class/typename`の部分に使用することができます。

```cpp
template<std::convertible_to<int> T>
void f(T);

template<typename T>
  requires std::convertible_to<T, int>
void f(T);
```

この二つの`f()`は同じ制約がなされています。つまり、2つ以上のテンプレート引数を取るコンセプトを`class/typename`の部分に使用するときは、第1引数は省略して第2引数以降を明示的に指定してやればいいわけです。

2つ以上の引数をとるコンセプトを`class/typename`の部分に用いる場合、制約対象のテンプレートパラメータがその1つめの引数として暗黙に補われており、1引数コンセプトの場合は`<>`をも省略することができます。`std::convertible_to`コンセプトは`std::convertible_to<From, To>`のように`From -> To`への変換可能性を表現するコンセプトですが、この仕様によって`std::convertible_to<To> From`のように書いて使用して可読性の向上を図ることができます。

```cpp
// memory_resourceから派生したT
template<std::derived_from<std::pmr::memory_resource> T>
void f(T*);

// int, double, char*から構築可能なT
template<std::constructible_from<int, double, char*> T>
void f(T);
```

ここまで関数で例示してきましたが、コンセプトによる制約はクラステンプレートに対しても同じように行うことができます。ただし、後置`requires`節は利用できません。

```cpp
template<typename T>
struct S1 {};

template<std::integral T>
struct S1<T> {};

template<typename T>
struct S2 {};

template<typename T>
  requires std::integral<T>
struct S2<T> {};
```

変数テンプレートでできることはクラステンプレートに準じているので、変数テンプレートでも同様です。

```cpp
template<typename T>
constexpr std::uint8_t v1 = 1;

template<std::integral T>
constexpr std::uint8_t v1<T> = 2;

template<typename T>
constexpr std::uint8_t v2 = 1;

template<typename T>
  requires std::integral<T>
constexpr std::uint8_t v2<T> = 2;
```

ただし、クラス/変数テンプレートは関数テンプレートとは異なりそのままではオーバーロードできないため、オーバーロードを表現するためには部分特殊化として定義する必要があります。そして、部分特殊化の仕様のために、プライマリテンプレートに制約を行うと謎のエラーに悩まされることになりますので、プライマリテンプレートには制約を行わないことをお勧めします（また、それを説明すると長くなるのでここでは説明しません）。

ここまではコンセプトの利用側を見てきましたが、ここからはコンセプトを作成する方法を見ていきます。コンセプトは`concept`というキーワードを用いたテンプレートによって定義されます。先程までよく見ていた`std::integral`などは例えば次のように定義されています

```cpp
template<typename T>
concept integral = std::is_integral_v<T>;

template<typename T>
concept floating_point = std::is_floating_point_v<T>;
```

このように、テンプレートパラメータ宣言に続いて`concept name = expr;`のような形式で定義し、`expr`の部分には`requires`節の時と同じ記法によって任意の制約式を指定していくことができます。ただし、コンセプトそのものに何らかの制約を行うことはできません。

```cpp
// 符号付整数型
template<class T>
concept signed_integral = integral<T> && std::is_signed_v<T>;

// これはng
template<integral T>
concept signed_integral = std::is_signed_v<T>;
```

さらに細かい制約を行うために、`requires`式を使用できます。

```cpp
// TとUの値の間で+による和を取れる
template<typename T, typename U>
concept addable = requires(const T& t, const U& u) {
  t + u;
  u + t;
};

// Tがメンバ型value_typeを持つ
template<typename T>
concept has_value_type = requires {
  typename T::value_type;
};
```

この`requires`式では、`requires(Args...)`のようにして関数宣言と同じように引数を宣言することができ、この引数は`requires`式内で制約を書くのに使用できます。引数が必要ない場合は単に`requires { ... };`のようにも書けます。`requires`式内では任意の式を宣言的に記述していき、それらの式が利用可能であることを要求する制約を記述することができます。`requires`式はこれで1つの制約式であり、その全体は内部に記述された各式が全て利用可能であれば`true`となります。なお、`requires`式内は評価されない文脈であり、そこに書かれている式が実行されることはありません。`requires`式内の各式はあくまで、書かれている式が存在するor利用可能である事をチェックするだけです。

`requires`式内の各式に対して、その戻り値型をチェックすることもできます

```cpp
// TとUの値の間で+による和を取れる
// +の結果は常にT
template<typename T, typename U>
concept addable = requires(const T& t, const U& u) {
  {t + u} -> std::same_as<T>;
  {u + t} -> std::same_as<T>;
};

// Tがメンバ型value_typeを持つ
// value_typeはint
template<typename T>
concept has_value_type = requires {
  typename T::value_type;
  requires std::same_as<typename T::value_type, int>;
};
```

`{ expr } -> constraint;`の形式で記述して、式`expr`の戻り値型が制約式`constraint`による制約を満たすことを表します。この場合、`expr`が利用可能であることに加えて、その型が`constraint`を満たしていなければなりません。`constraint`のところにはコンセプトを指定して、そこでは左辺の`expr`の戻り値型（`decltype((expr))`）が1つ目の引数として補われており、2引数以上のコンセプトを`class/typename`の部分に指定する時と同じ補間がなされています。

メンバ型の制約は少し特殊な書き方（`requires constraint;`）をする必要があります。これは入れ子要件と呼ばれ、bool値を返す制約式`constraint`が満たされている事を`requires`式の中で表現するための構文です。`typename T::value_type;`によって`T`が`value_type`を持つ事、`requires std::same_as<typename T::value_type, int>;`によってその`value_type`が`int`である事を制約しています。

### 制約とオーバーロード優先順位

コンセプトによって制約された関数テンプレート、クラス・変数テンプレートの部分特殊化はコンセプトの有無及びコンセプトの内容によってオーバーロード解決時に優先順位がつけられます。ただしその順序は半順序（順序付けできるとは限らない）であり、詳細はかなり複雑です。ここでは、何とか理解できそうな部分だけを取り上げます。

まず、コンセプトによって制約されていない関数と制約されている関数の間では、制約されている関数のほうが優先順位が高くなります（以下、関数テンプレートで説明しますがクラス・変数テンプレートでも同様です）。

```cpp
template<typename T>
void f(T);  // (1)

template<std::integral T>
void f(T);  // (2)

int main() {
  f(10);  // (2)が呼ばれる
}
```

次に、コンセプトによって制約されている関数の間では、片方の制約がもう片方の制約を完全に包含しているとき、包含しているほうが優先順位が高くなります。

```cpp
template<std::copyable T>
void f(T);  // (1) copyableのみ

template<std::copyable T>
  requires std::convertible_to<T, int>
void f(T);  // (2) copyable + convertible_to

int main() {
  int n = 10;

  f(n);  // (2)が呼ばれる
}
```

2つの関数の間の制約の包含関係は、`&&`でつながれている各制約式（原子制約と言います）について、一方の関数（A）の全ての原子制約が他方の関数（B）の制約に全て現れていて、かつ他方（B）はそれに加えてさらに制約されている時、A ⊂ Bの制約の包含関係が成立し、包含している方がより制約されているとしてB > Aの順序付けがなされます。別の言い方をすると、関数オーバーロードAとBそれぞれの原子制約を要素とする2つの集合があった時、その間に成立する包含関係（片方は他方の真部分集合）がコンセプトの包含関係となります。そして、そのような包含関係が成り立たない時は順序付け不可能となり、2つの関数オーバーロードの呼び出しは曖昧となります。

```cpp
template<typename T>
concept A = /*...*/;

template<typename T>
concept B = /*...*/;

template<typename T>
concept C = /*...*/;

template<typename T>
concept D = /*...*/;

template<typename T>
  requires A<T>;
void f(T); // (1)

template<typename T>
  requires A<T> && B<T>;
void f(T); // (2)

template<typename T>
  requires A<T> && B<T> && C<T>;
void f(T); // (3)

template<typename T>
  requires A<T> && B<T> && D<T>;
void f(T); // (4)
```

この時、各`f()`の制約の包含関係は(1) ⊂ (2) ⊂ (3)となり、(3) > (2) > (1)の順で優先順位が付きます。一方、(3)と(4)の制約の間にはお互いに包含関係が成立しない（`C`は(4)の制約に現れず、`D`は(3)の制約に現れない）ため順序が付きません。したがって、(3)と(4)の間ではオーバーロード解決が曖昧となります。

`&&`だけで繋がれている場合はこのように比較的理解できるのですが、ここに`||`が入ると一気に複雑になります。非常に難解なのでここでは説明しませんが、避けられるならば避けた方が良いでしょう・・・

そしてもう一つややこしいことに、原子制約にコンセプトではなく単なる制約式が混じっている場合、このような順序付けができなくなることがあります。

\clearpage

```cpp
template<typename T>
constexpr bool A = /*...*/;

template<typename T>
concept B = /*...*/;

// このf()は順序がつかない
template<typename T>
  requires A<T>
void f(T);

template<typename T>
  requires A<T> && B<T>
void f(T);
```

`A`は変数テンプレートなのに対して`B`はコンセプトです。このとき、`A, B`の両方を満たすような`T`に対して`f()`が呼ばれると、順序がつかずオーバーロード解決に失敗します。`A`がコンセプトではないのが原因で、全く同じ文字列で同じ結果になるとしても、コンセプトとして定義されていない2つの制約式は同値とみなされず、制約の包含関係を正しく成り立たせることができません。

これは`A`がほかの形の定数式であったとしても同じです（例えば、コンセプト`C`に対して`!C`など）。ただし、この場合に`A`がコンセプトで`B`が`bool`定数式であるならば順序付けを行うことができ、コンセプトとして制約されている部分だけで包含関係が成り立っているのであれば順序付けを行うことができます。とはいえ非常にややこしい問題ですので、コンセプトによってオーバーロード解決の順序付けを行う場合は`&&`をなるべく使用するようにしたうえで、使用する制約式は単一のコンセプトとして定義しておくことが推奨されます。

```cpp
template<typename T>
concept A = /*...*/;

template<typename T>
concept B = /*...*/;

// このf()は順序がつかない
template<typename T>
  requires (!A<T>)
void f(T);

template<typename T>
  requires (!A<T> && B<T>)
void f(T);

// 個別のコンセプトにする
template<typename T>
concept not_A = !A<T>;

// このg()は順序がつく
template<typename T>
  requires not_A<T>
void g(T);

template<typename T>
  requires not_A<T> && B<T>
void g(T);
```

### 特殊メンバ関数の条件付きトリビアル定義

- [P0848R3 Conditionally trivial special member functions](https://wg21.link/p0848r3)

このコンセプトを応用して、クラステンプレートにおいてそのテンプレートパラメータの満たす性質に応じて自クラスの特殊メンバ関数の`default`定義をコントロールすることができるようになります。

```cpp
template<typename T>
class optional {
  union {
    char dummy;
    T storage;
  };

public:

  // 特殊メンバ関数の条件付きdefault定義
  optional(const optional&) requires std::is_trivially_copyable_v<T> = default;
  ~optional() requires std::is_trivially_destructible_v<T> = default;

  optional(const optional&) {
    // コピー処理
  }

  ~optional() {
    // 破棄処理
  }

};
```

クラスの特殊メンバ関数の`default`宣言に対して後置`requires`節による制約を行うことで、その制約が満たされるかどうかでその`default`宣言の有効性をコントロールできます。制約が満たされる場合は`default`宣言は有効であり、コンセプトによる制約の半順序（無制約関数は優先度最低）に応じて残った同じ種類の特殊メンバ関数は暗黙`delete`されます。逆に制約が満たされない場合は制約された`default`宣言が暗黙`delete`されます。

これは通常のクラステンプレートではあまり使用する機会はなさそうですが、`optional`や`variant`といった複雑な実装を伴うクラスではTMPによる複雑な実装をかなり削減することができます。

\clearpage

## モジュール

モジュールとは、プログラムをある単位で個々のプログラムとして分割する機能、およびそのように分割された個々のプログラムから1つのプログラムを構成する機能のことで、そのように分割された個々のプログラムのこともモジュールと呼びます。

今どきの言語はたいてい何かしらのモジュールシステムを持っており、従来のC++ではそれは翻訳単位の分割とヘッダファイルインクルードが担っていました。しかし、ヘッダのインクルードはモジュールシステムとしては少し問題が多く、それに代わる専用の機能としてのモジュールが追及された結果、長い議論の末にC++20にて正式な言語機能として導入されました。C++20で言語機能として用意されるのは、プログラムのある単位を1つのモジュールとして構成する方法と別に定義されたモジュールをプログラム中で利用する方法、およびそれらを円滑に実現するための仕組みです。残念ながら、パッケージマネージャなどビルドにかかわる部分はほぼなにも用意されません（ただし、そうした部分を整備しようという動きはあります）。

まずはモジュールの利用側を見てみます。モジュールの利用、つまりは別に定義されたモジュールを使用する場合は`import`宣言によって行います。以下、あらかじめモジュール`A, B`が用意されているとします。

```cpp
import A; // モジュールAのインポート
import B; // モジュールBのインポート

int main() {
  // モジュールAからエクスポートされているとする
  A::func();  // ok

  // モジュールBからエクスポートされているとする
  B::C c{};   // ok
}
```

`import name;`の形式の宣言によって`name`に指定した名前のモジュールを現在の翻訳単位にインポートします。モジュールをインポートすると、そのモジュールでエクスポートされている宣言がその翻訳単位から見えるようになり、利用可能となります。

`import`宣言がヘッダファイルの`#include`と異なるのはモジュールの内容をその場にコピペしたりしないことです。モジュール側では外部に公開したいものを選んでエクスポートすることができ、モジュールをインポートしたときはエクスポートされたものの名前が可視（名前探索でみつかる）になり、その名前の持つC++プログラムとしてのあらゆるプロパティが到達可能（利用できる）となります。したがって、同じモジュールを指定する`import`宣言は何度行われても多重定義などの問題を起こしません。

```cpp
import A;
import B;
import A; // ok、問題は起こらない

int main() {}
```

また、`import`宣言はあらゆる宣言の外側に現れていなければならず、`import`をマクロ置換したり途中で改行することはできません。

```cpp
#define I import

I A;  // ng、import宣言とみなされない
import
A;    // ng、1行で書く

#define M A
import M; // ok、モジュール名はマクロ置換可能

namespace A {
  import A; // ng、名前空間スコープ
}

int main() {
  import A; // ng、ローカルスコープ
}
```

このように使用されるモジュールは、別のファイルでモジュール宣言によって作成し、通常のモジュールは2つのファイルから構成されます。

```cpp
/// modA.cpp
export module A;

/// modA_impl.cpp
module A;
```

（ここでは、`///`から始まるコメントでファイル分割を表しています）

`module name;`の形の宣言をモジュール宣言と呼び、これがファイルの先頭（その他の宣言の前）でなされているファイルはモジュールファイルとなります。`name`はモジュール名で、インポートするときはここに指定した名前を`import`宣言に指定します。モジュール名には通常のクラスや変数名に使える文字に加えて`.`を使用できます。

`modA.cpp`ファイルでは、`export`から始まるモジュール宣言によってそのファイルがモジュールインターフェース単位であることを宣言し、`modA_impl.cpp`では`export`のないモジュール宣言によってそのファイルがモジュール実装単位であることを宣言しています。ファイルとしても翻訳単位としても分かれていますが、同じ名前が指定されていることでコンパイラはモジュール`A`がこの2つのファイルによって構成されることを認識します。インターフェース単位は従来のヘッダファイルに、実装単位はそのヘッダに対応するソースファイルにそれぞれ対応するものです。ただし、1つのモジュール内にはインターフェース単位は1つだけ存在している必要があり、0にも2以上にもなってはいけません。

インターフェース単位では外部に公開したい宣言を`export`を用いて宣言し、実装単位はそれに対応する定義（実装）を記述します。

```cpp
/// modA.cpp
export module A;

namespace A {
  // func()のexport宣言
  export void func();
}

/// modA_impl.cpp
module A;

namespace A {

  // funcの定義
  void func() {
    // ...
  }
}
```

ある宣言の先頭に`export`を付加することでその宣言は`export`宣言となり、`export`宣言によるものだけがそのモジュールからエクスポートされます。`export`宣言はモジュールインターフェース単位でのみ行うことができ、名前空間スコープにある名前を持つほぼ全ての宣言は`export`宣言とすることができます。モジュールをインポートするときはインターフェース単位の内容だけがインポートされており、実装単位の内容はそのモジュール外部から完全に隠蔽されます。実装単位に定義されたものの呼び出しは、エクスポートされた名前を介した通常の翻訳単位超えのルールによって行われます。

```cpp
/// modA.cpp
export module A;

namespace A {
  export void func();
}

/// modA_impl.cpp
module A;

namespace A {

  // 実装用関数
  void inner() {
    // ...
  }

  void func() {
    // ...
  }

  // ng、実装単位ではexport宣言を行えない
  export g();
}

/// main.cpp
import A;

int main() {
  A::func();  // ok
  A::inner(); // ng、エクスポートされていない
}
```

なお、実装単位からは対応するインターフェース単位のものが`export`の有無にかかわらず見えています（ただし、内部リンケージを持つものは除きます）。

モジュール内をさらにある単位で分割して1つのモジュールを構成したいこともあるでしょう。そのために、モジュールパーティションを利用できます。

```cpp
/// modA_part.cpp
// インターフェースパーティション
export module A:part;

/// modA.cpp
export module A;
export import :part;  // パーティションの再export

/// modA_part_impl.cpp
// 実装パーティション
module A:part;

/// modA_impl.cpp
module A;
```

`module name:partition_name;`のように、モジュール名に続いて`:`がある形式のモジュール宣言によってモジュールパーティションを宣言でき、モジュールパーティションは`name`に指定されたモジュールの一部となります。モジュールパーティションの宣言もまた`export`の有無によってインターフェースパーティションと実装パーティションに区別されます。インターフェースパーティションと区別するために通常のインターフェース単位のことをプライマリインターフェース単位と呼び、プライマリインターフェースが1つしか存在できないのに対してインターフェースパーティションは複数存在することができます。

モジュールの一部とは言ってもパーティションは自動的に他のモジュールから参照可能になるわけではないため、必要に応じて明示的にインポートする必要があります。特に、インターフェースパーティションはプライマリインターフェース単位からエクスポートしておかないと外部から参照することはできません。`export import name;`のような`export`宣言は再エクスポートと呼ばれ、`name`のモジュールをインポートすると同時にエクスポートします。あるモジュール（`M`）から再エクスポートされているモジュールは、`M`をインポートした際に自動的にインポートされます。再エクスポートは任意のモジュールに対して行うことができ、パーティションに対しては`name`のところを`:partition_name`（`:`の左側は空白）のようにします。

```cpp
/// modA.cpp
export module A;
export import :part;  // パーティションの再export
export import B;      // 別モジュールの再export

/// main.cpp
import A;

int main() {
  // モジュールBからエクスポートされているとする
  B::C c{};   // ok
}
```

モジュールのパーティションへの分割は外部からは観測不可能であり、あるモジュールのパーティションは同じモジュール内からしかインポートすることができません。それ以外の部分やできることは通常のモジュールのインターフェース/実装単位と同じです。

全てがモジュールとして利用できればいいのですが、流石にまだまだヘッダファイルが主流であり標準ライブラリも例外ではありません。モジュール内ではグローバルモジュールフラグメントと呼ばれる領域でヘッダファイルのインクルードを行います。

\clearpage

```cpp
/// modA.cpp
module; // グローバルモジュールフラグメントの開始

#include <ranges>
#include "fmt/format.h"

export module A;
```

グローバルモジュールフラグメントはファイル先頭（他のあらゆるトークンの前）の`module;`によって始まり、モジュール宣言によって終了します。グローバルモジュールフラグメントにはプリプロセッシングディレクティブ以外のものが直接現れてはならず、`#include`のほかはマクロ定義を行うのにも利用できます。モジュールにおけるODRのルールは従来（および非モジュールの場所）に比べて厳しくなっているため、モジュールでヘッダファイルをインクルードする際はグローバルモジュールフラグメントを使用することが推奨されますが、標準ライブラリヘッダについては特にグローバルモジュールフラグメント以外の場所で`#include`できません（未定義動作となります）。

なお、グローバルモジュールフラグメントとモジュール宣言は、現れる場合ファイルの先頭（空白とコメント以外のあらゆる文字の前）に現れる必要があり、`#if/#ifdef`などでそれを切り替えることはできません。

```cpp
// ng、これはできない
#ifdef A_IS_HEADER
#pragma once
#else
export module A;
#endif
```

ヘッダファイルを利用する方法としては、もう一つヘッダユニットというものも提供されます。これは、従来のヘッダファイルをモジュールとして扱うもので、ヘッダファイルを利用しながらモジュールの恩恵を受けることができます。

```cpp
// ヘッダユニットのインポート
import <ranges>;
import "fmt/format.h";

int main() {
}
```

ヘッダユニットは`import`宣言においてモジュール名ではなくヘッダファイルを指定する事でインポートします。このようにインポートされたヘッダユニットは1つのモジュールとして扱われ、ヘッダ内で外部リンケージを持つ全ての宣言がエクスポートされています。また、通常のモジュールはマクロをエクスポートする方法がありませんが、ヘッダユニットからはマクロもエクスポートされます。

標準ライブラリヘッダはC互換ヘッダ（`<cxxx>`）以外の全てのヘッダがヘッダユニットとして利用できますが、それ以外のヘッダファイルがヘッダユニットとして利用できるかは実装定義とされています。例えばMSVCでは、コンパイラオプションで指定することで任意のヘッダをヘッダユニットとして扱えるようになっています。

### 標準ライブラリモジュール

- P2465R1 Standard Library Modules `std` and `std.compat` (https://wg21.link/p2465r1)

C++20では標準ライブラリのモジュール化は間に合いませんでしたが、それはC++23で予定されています。今の所、`std`名前空間のものを全て含む`std`モジュールと、それに加えてグローバル名前空間にある標準のものを含む`std.compat`という2つのモジュールを導入する方向で議論が進んでいます（本書執筆時点では正式に決定していません）。

```cpp
import std;

int main() {
  std::cout << "Hello World!" << std::endl;
}
```

もう少し細かい単位で分割したモジュールも提案されてはいますが、こちらはもう少し時間がかかりそうです。

\clearpage

## コルーチン

コルーチンとは、中断と再開が可能な関数のことです。コルーチンを利用することで、ジェネレータ（範囲の生成）や非同期処理とその継続の記述を簡易化できるなどの恩恵がありますが、C++コルーチンそのものはそうしたコルーチンアプリケーションを何ら提供しないため、そのままでは特に何ができるわけでもありません。

C++20コルーチンでは、主に次のものが用意されるだけです

- C++コルーチンフレームワーク
    - スタックレスコルーチンのための種々の概念と動作戦略などの規定
    - コルーチン制御のための多数のカスタマイゼーションポイント
- `co_await, co_yield, co_return`演算子
- 最低限のコルーチンサポートライブラリ

実際にコルーチンを活用し、便利なコルーチンアプリケーションを実現するために必要なユーティリティは、ライブラリ機能としてC++23以降に順次導入していく予定です。C++20でコルーチンアプリケーションをどうしても使用したい場合は、cppcoroなどのライブラリを利用するといいでしょう。

### `std::generator<T>`

- P2168R3 `std::generator`: Synchronous Coroutine Generator for Ranges (https://wg21.link/p2168r3)

`std::generator<T>`は範囲を生成するコルーチンジェネレータのためのユーティリティ型です。`std::generator<T>`は`T`の範囲として`std::ranges::range`コンセプト（簡単には、`begin()/end()`からイテレータを取得可能な型）を満たします。

```cpp
#include <generator>
#include <ranges>

// フィボナッチ数列を生成する関数
std::generator<int> fib (int max) {
  co_yield 0;

  auto a = 0, b = 1;
  for (auto n : std::views::iota(0, max)) {
    auto next = a + b;
    a = b, b = next;
    co_yield next;
  }
}

int main() {
  for (int n : fib(100) | std::views::take(7)) {
    std::cout << n; // 0112358
  }
}
```

`fib()`はコルーチンです。その実行に際し、`co_yield`に到達する度に呼び出し元に値を返して中断します。ここでは隠蔽されていますが、中断したコルーチン（`fib()`）は`std::generator<T>`のイテレータの進行（`++`）のタイミングなどで再開され、次の`co_yield`まで進みます。`fib()`内ではループによって`max`個の値を生成しており、`co_yield`によって1つづつ生成された上で返されています。生成された値はイテレータを介して取得されるため、範囲`for`によるイテレーションが可能となります。

`std::generator<T>`はこのような値の連続生成のためのコルーチンの制御と、コルーチンの呼び出し・再開処理のイテレータインターフェースへの変換を担うクラスです。現在はC++23導入を目指して作業されています（本書執筆時点では導入されていません）。

### `std::lazy<T>`

- P1056R1 Add lazy coroutine (coroutine task) type (https://wg21.link/p1056r1)

`std::lazy<T>`は非同期処理とその継続をコルーチンによって表現するためのユーティリティ型です。C#などのほかの言語では`Task`クラスと呼ばれるものに相当しています。

```cpp
// 何かのデータ型
struct record {
  int id;
  std::string name;
  std::string description;
};

// recordをどこかから読み出す非同期処理
std::lazy<record> load_record(int id);
// recordをどこかに保存する非同期処理
std::lazy<> save_record(record r);

// recordの更新処理
std::lazy<void> modify_record() {
  // これらの処理はこの順番に実行される
  record r = co_await load_record(123);
  r.description = “Look, ma, no blocking!”;
  co_await save_record(std::move(r));
  std::cout << "modify end.\n"
}

// modify_record()の非コルーチン版
void modify_record_async() {
  auto f1 = load_record_async(123, [](record r) {
    r.description = “Look, ma, no blocking!”;
    auto f2 = save_record_async(std::move(r), [](){
      std::cout << "modify end.\n"
    });

    f2.wait();
  });

  f1.wait();
  // f1, f2はstd::future相当のもの
}
```

`load_record(), save_record(), modify_record()`はコルーチンです。`modify_record()`では、`co_await`によってコルーチンを呼び出すことで呼び出したコルーチンの処理を待機するとともに、現在関数の実行を中断します。この振る舞いによって`modify_record()`の各行はコードの記述通りに順番に実行されることが保証されます。この時、この非同期実行完了待機と後続処理の呼び出しは`modify_record()`を呼び出したスレッドで行われるとは限らず、`std::lazy<T>`の`co_await`は現在の実行を中断するとともに現在のスレッドを解放し、呼び出した非同期処理が完了したスレッドで再開します。`std::lazy<T>`は、このような`co_await`における非同期処理完了待機と中断・再開に関する制御と、非同期処理結果の取得を担うクラスです。

このような、非同期（並行）処理の連続した呼び出しが、通常の同期処理と同様にコードの見た目通りに実行されること（または非同期処理を同期処理とほぼ同様に記述できること、その保証）を*Structured Concurrency*と言います。 `modify_record_async()`は`modify_record()`をコールバックによるコードで書き直した一例で、比較することで`std::lazy<T>`による*Structured Concurrency*が何を保証し、可能としているのかがわかるかもしれません。

`std::lazy<T>`はコルーチンによって*Structured Concurrency*を実現するために必須のクラスであり、C++23に向けて提案されてはいますが、長期にわたって進展がないので間に合わないかもしれません。

### コルーチン動作概要

コルーチンは規定された一連の動作のいくつものタイミングにカスタマイゼーションポイントが仕込まれており、それによって上記2例のようにコルーチンの上に多様なアプリケーションを実装することができます。ここでは、その動作の流れの概要を見ていきます。

コルーチンとは言語仕様によって特別扱いされる関数であり、その関数本体（*function body*）に`co_await, co_yield, co_return`いずれかの呼び出しを含む場合、その関数はコルーチンとして認識されます。

```cpp
// コルーチンとする
R coro_f(Args args...) {
  // function body
  // ...
}
```

ここでの`Args args...`は0個以上任意の仮引数を表すものとして、可変長テンプレートから構文を拝借している本書独自の構文です。

この`coro_f()`をコルーチンとすると、コルーチンは次のように書き換えられた関数として実行されます

```cpp
R coro_f(Args args...){
  bool initial_await_resume_called = false;

  // プロミス型の導出
  using promise_type = std::coroutine_traits<R, Args...>::promise_type;
  // プロミスオブジェクトの初期化
  promise_type promise(/*promise-constructor-arguments*/);

  R result = promise.get_return_object();

  try {

    // 初期サスペンドポイント
    // ここでresultを返す（必要なら中断する）
    co_await promise.initial_suspend();
    // コルーチン再開の直後にinitial_await_resume_calledにtrueが設定される

    // function body
    // ...

  } catch (...) {
    if (!initial_await_resume_called) {
      // コルーチン初期化完了までの例外は外に投げる
      throw;
    }
    // コルーチン本体例外処理の委譲
    promise.unhandled_exception();
  }

final_suspend :
  // 最終サスペンドポイント
  co_await promise.final_suspend();
}
```

この書き換えは一例でありC++コードとして完全なものではなく、コルーチン仕様によって補われ、また隠蔽されている部分があります。

コルーチン中の`co_await`式（*function body*にあるものも含めて）の実行に際してコルーチンは中断し、`result`として取得したコルーチンハンドルを呼び出し元へ返します。呼び出し元はそのコルーチンハンドルを介して適宜コルーチンを再開し、再開されたコルーチンは中断箇所（最後に実行した`co_await`式の場所）から再開され実行を継続します。中断している間のコルーチンの状態はコルーチンステートとして保持されているため、コルーチン引数およびコルーチンスコープ内のローカル変数は中断再開を通しても生存しています。コルーチンステートは、コルーチン開始時に動的メモリ確保（`::operator new(size_t, nothrow_t)`）によって確保され、コルーチンの終了に伴って解放される領域に配置されます。ただし、そのような動的メモリ確保を避ける最適化（*Coroutine ellision*）が許可されており一部の処理系は実装しているため、動的メモリ確保の省略が期待できるはずです。このように、C++の言語機能がサポートするのはコルーチンステートの管理および`co_await`による中断と再開の管理の部分で、それ以外の部分は上記のような書き換えとカスタマイゼーションポイントを介したユーザー指定の動作によってコルーチンは動作します。

上記例中の`promise_type`はプロミス型と呼ばれるユーザー定義の任意の型で、コルーチンカスタムの中心となるものです。名前の通りFutureパターンにおける*promise*に対応しており、それになぞらえるとコルーチンハンドル（例中`result`）が*future*に、コルーチンステートが共有状態に対応します。`promise_type`およびコルーチンハンドル型`R`（正確にはコルーチンハンドルを内包する型）はユーザー定義する必要があり、基本的にはその2つの型を介してコルーチン動作のカスタマイズを行います。

`promise_type`は`std::coroutine_traits`を介して戻り値型`R`と引数型`Args...`から求められ、`std::coroutine_traits`が特殊化されていない時のデフォルトは`R::promise_type`から取得されます。`promise_type`のオブジェクトは*promise-constructor-arguments*から初期化され、それは次のように決まります

1. `promise_type(args...)`による初期化が可能なら、*promise-constructor-arguments*はコルーチン引数列`args...`
    - `promise_type promise(args...);`
2. それ以外の場合、*promise-constructor-arguments*は空
    - `promise_type promise();`

そうして初期化したプロミスオブジェクト`promise`から、戻り値となるコルーチンハンドル（を内包した）オブジェクト`result`を`promise.get_return_object()`にて取得します。

その後まず、コルーチンは初期サスペンドポイントと呼ばれる地点に到達します。初期サスペンドポイントは`co_await promise.initial_suspend()`の呼び出しで、その振る舞いおよびここで中断するか否かを`promise.initial_suspend()`メンバ関数（の戻り値）によって制御します。`co_await`式は`awaitable`なオブジェクトを受け取ってその3つのメンバ関数を通してコルーチンの中断と再開を制御するもので、標準にも非常に簡易な2つの`awaitable`型が用意されています

```cpp
namespace std {
  // 常に中断しないawaitable
  struct suspend_never {
    constexpr bool await_ready() const noexcept { return true; }
    constexpr void await_suspend(coroutine_handle<>) const noexcept {}
    constexpr void await_resume() const noexcept {}
  };

  // 常に中断するawaitable
  struct suspend_always {
    constexpr bool await_ready() const noexcept { return false; }
    constexpr void await_suspend(coroutine_handle<>) const noexcept {}
    constexpr void await_resume() const noexcept {}
  };
}
```

`std::suspend_never`は`co_await`式で評価されるとそこでコルーチンを中断しない`awaitable`型で、`std::suspend_always`はそこでコルーチンを中断する`awaitable`型です。この二つは中断の制御以外に特に何もしない`awaitable`型ですが、中断・再開時の動作のカスタマイズが必要な場合はこの3つのメンバ関数を持つように`awaitable`型をユーザー定義して使用することができます。

`co_await`式では`awaitable`型のメンバ関数`await_ready()`の戻り値によってコルーチンの中断を制御し（`false`で中断）、中断する場合はメンバ関数`await_suspend()`を現在のコルーチンハンドルを渡して呼び出して中断するときの処理を実行します（上記2クラスは中断時の処理を何もしていない）。また、`awaitable`型に対して`co_await`演算子をオーバーロードしておくことで、`co_await`式開始時の動作をカスタマイズすることもできます。

`promise.initial_suspend()`の戻り値の`awaitable`のメンバ関数`await_ready()`が`true`を返した、もしくはコルーチンが再開された時、まず`initial_await_resume_called`変数に`true`を設定し、その後`awaitable`のメンバ関数`await_resume()`が実行されコルーチン再開時の処理が実行されます（上記2クラスは再開時も何もしない）。その後初期サスペンドポイントの次に処理が進み、定義されたコルーチン本体（*function body*）の実行に入ります。

コルーチン本体が例外を投げた時、`initial_await_resume_called == false`ならば例外を素通しします。`initial_await_resume_called`が`false`となるのは、初期サスペンドポイントにおいて`await_resume()`が呼び出される前に例外が発生した時なので、コルーチンの初期化が完了していないときの例外と判断でき、この例外の処理は呼び出し側の責任というわけです。コルーチンの初期化が完了していれば、`promise.unhandled_exception()`にその処理が委ねられます。この関数もユーザー定義可能であるため、コルーチンの本体が例外を投げた時の振る舞いを`promise`型を通してカスタムすることができます。

コルーチン本体の実行がつつがなく完了した場合、コルーチンの処理は最終サスペンドポイント（`co_await promise.final_suspend()`の呼び出し）に到達します。最終サスペンドポイントでは初期サスペンドポイントとほぼ同様に`promise.final_suspend()`メンバ関数（の戻り値）によってコルーチン終了時の振る舞いを制御します。ここでコルーチンが中断されない場合、コルーチンの実行は終了し、プロミスオブジェクトおよびコルーチンステートが破棄され、コルーチンステートの領域も解放されます。コルーチンハンドルは`R`のオブジェクトを介して呼び出し側に渡っているため、そのハンドルが解放されるまでコルーチンステートを維持したい場合などにここでコルーチンを中断しておくことができるようになっています。その時でも、コルーチンハンドルの解放に伴ってコルーチンは再開され、その後にコルーチンは終了します。

コルーチン本体内（*function body*）にある`co_await`式でもその引数に対して初期（最終）サスペンドポイントと同様に適宜中断と再開が行われますが一つだけ違いがあり、`co_await`の引数`e`に対して`promise.await_transform(e)`メンバ関数が適用可能ならそれを適用して`awaitable`オブジェクトを取得しようとします。これは、コルーチン本体の（つまりユーザーが記述する）`co_await`式において任意の値を`awaitable`へと変換するためのカスタマイゼーションポイントであり、これによってユーザーは`awaitable`について知らなくてもコルーチンを使用可能となります。

コルーチン本体内の`co_yield`式はその引数を`e`とすると、`co_await promise.yield_value(e)`のように書き換えられて実行されるため、`co_yield`式は結局`co_await`式として実行されており、その際の振る舞いを`promise.yield_value()`メンバ関数によってカスタムすることができます。ただし、`co_yield`式からの`co_await`式実行時は`await_transform()`は使用されないため、`yield_value()`は`awaitable`を返す必要があります。

ここまでの例には一切登場していませんが、`co_return`はコルーチンの実行を明示的に終了するために使用でき、次のように書き換えられて実行されます

```cpp
// これは
co_return expr;

// こう実行される
{
  S;
  goto final_suspend;
}
```

その引数の式`expr`が`{}`初期化であるか`void`ではない式である場合、`S`は`promise.return_value(expr)`となり、それ以外の場合`S`は`{ expr; promise.return_void();}`となります。

```cpp
// exprが{}初期化か`void`ではない式
{
  promise.return_value(expr);
  goto final_suspend;
}

// それ以外
{
  {
    expr;
    promise.return_void();
  }
  goto final_suspend;
}
```

いずれにせよ、`co_return`の振る舞いは`promise`型の`return_value()/return_void()`メンバ関数によってカスタムでき、最終的には最終サスペンドポイントに到達するわけです。

非常に複雑ではありますが、このようにいくつものカスタマイゼーションポイントが用意されていることによってコルーチンの実行・振る舞いを細かく制御することが可能になっており、最小限の言語機能によって多様なコルーチンアプリケーションを構築することができるようになっています。それらコルーチンカスタマイゼーションポイントを大雑把にまとめておくと次のようになります

\clearpage

- `promise`型 : コルーチン各部の振る舞いの制御
    - 各種メンバ関数
- コルーチンの戻り値型`R`と引数型`Args` : プロミス型導出とコルーチン再開タイミングの制御
    - `R`もしくは`std::coroutine_traits<R, Args...>`の特殊化のメンバ型`::promise_type`
    - `R`によるコルーチンハンドル（`std::coroutine_handle<promise_type>`）の管理
- `awaitable`型 : コルーチン中断と再開時の制御
    - `await_ready()`
    - `await_suspend()`
    - `await_resume()`
    - `co_await`演算子オーバーロード

`promise`型はコルーチン戻り値型`R`から得られ、`awaitable`は通常`promise`型のメンバ関数（`initial_suspend()/final_suspend()/await_transform()/yield_value()`）を介して得られます。コルーチンハンドルを内包する`R`は呼び出し元に渡ってユーザーによって操作されることでコルーチンの進行を行い、`promise`型はコルーチン側で再開等の各タイミングでの動作カスタムを担っており、`awaitable`型はコルーチン内`co_await`による中断・再開時の動作カスタマイズに集中するものです。

\clearpage

# 定数式

## 仮想関数

- P1064R0 Allowing Virtual Function Calls in Constant Expressions  
  (https://wg21.link/p1064r0)

従来、定数式で実行できるものは厳しく制限されており仮想関数は実行可能ではありませんでしたが、C++20からはそれが解禁されます。

```cpp
struct base {
  virtual constexpr int f() const {
    return 0;
  }
};

struct derived : base {
  constexpr int f() const override {
    return 20;
  }
};

constexpr int g(const base& b) {
  return b.f();
}

int main() {
  constexpr derived d{};
  static_assert(g(d) == 20);  // ok
}
```

定数式においては未定義動作はコンパイルエラーとする事が求められているため、定数式においてコンパイラはあるポインタ（参照）の指すオブジェクトの動的型（実際に構築されている型）を追跡しています。それを利用すれば定数式でも仮想関数を実行可能であるため、定数式で仮想関数呼び出しを禁止する理由がなかった事からこの制限は撤廃されました。これはまた、`std::error_code`をはじめとする`<system_error>`のものを改善しようとする行動の一環でもあります。

ちなみに、非`constexpr`な仮想関数を`constexpr`としてオーバーライドすることも、`constexpr`な仮想関数を非`constexpr`としてオーバーライドすることもでき、その場合は最派生クラスでオーバーライドされている関数（実際に呼ばれるもの）が`constexpr`であれば定数式で実行可能となります。

## `dynamic_cast/typeid`

- P1327R1 Allowing `dynamic_cast`, polymorphic `typeid` in Constant Expressions (https://wg21.link/p1327r1)

仮想関数が解禁されたのと同様の理由によって、定数式における`dynamic_cast`および多態的な型に対する`typeid`が許可されます。

`dynamic_cast`の例

```cpp
struct base {
  virtual constexpr int f() const {
    return 0;
  }
};

struct base2 {
  virtual int g() const = 0;
};

struct derived3 : public base, public base2 {
  constexpr int f() const override {
    return 30;
  }

  constexpr int g() const override {
    return 30;
  }
};

constexpr int side_cast(const base* p) {
  // コンパイル時サイドキャスト
  const base2* b2 = dynamic_cast<const base2*>(p);
  return b2->g();
}

int main() {
  constexpr derived3 d{};
  static_assert(side_cast(&d) == 30); // ok
}
```

`typeid`の例

```cpp
struct base {
  virtual constexpr int f() const {
    return 0;
  }
};

struct derived1 : public base {
  constexpr int f() const override {
    return 20;
  }
};

struct derived2 : public base {
  constexpr int f() const override {
    return 30;
  }
};

constexpr auto& get_typeinfo(const base* p) {
  // コンパイル時に動的型のtype_infoを取得する
  return typeid(*p);
}

int main() {
  constexpr derived1 d1{};
  constexpr derived2 d2{};

  constexpr auto& tid1 = get_typeinfo(&d1); // ok
  constexpr auto& tid2 = get_typeinfo(&d2); // ok

  static_assert(tid1 != tid2);  // ok、C++23から
}
```

これは、`std::error_category`の改善（派生させて別のクラスにメンバを追加する）時に問題となり、仮想関数の流れを受けて許可されることになりました。ただし、`std::type_info`オブジェクトの比較に関してはC++20に間に合わず、遅れてC++23でできるようになります。

## `try-catch`

- P1002R1 Try-catch blocks in constexpr functions (https://wg21.link/p1002r1)

C++17まで`try-catch`ブロックは定数式に現れてはならず、`constexpr`関数では書くことすらできませんでした。C++20からは単に現れるだけは許可されるようになります。ただし、`throw`式の定数式での実行は許可されていないため`try-catch`ブロックが定数式で実行されることはありません。

```cpp
template<typename F>
constexpr int f(F&& fn) {
  try {
    return fn() + 1;
  } catch(...) {
    return -1;
  }
}

int main() {
  constexpr int n1 = f([]{ return 10; });           // ok
  constexpr int n2 = f([]{ throw 10; return 0; });  // ng、throwは定数式で使用不可
}
```

定数式で`throw`式に到達するとそこでコンパイルエラーとなるため、定数式における`try`ブロックは常に通過し`catch`ブロックは実行されません。

これは、`std::vector`などを`constexpr`対応させる際の障壁の一つとして問題となっていたため、許可されることになりました。

## 共用体のアクティブメンバ切り替え

- P1330R0 Changing the active member of a union inside constexpr  
  (https://wg21.link/p1330r0)

共用体のアクティブメンバとは、共用体のオブジェクトのある時点において一番最後に（一番最近）初期化されたメンバ変数のことです。C++のオブジェクト生存期間（*lifetime*）のルールとしては、その領域を共有している共用体内の各メンバは常にその中の一つだけが生存期間内にあり、共用体のオブジェクトから合法的にアクセス可能なメンバとは生存期間内にあるメンバ、すなわちアクティブメンバだけです。共用体のオブジェクト初期化後に非アクティブメンバを初期化することによって、その時点のアクティブメンバの生存期間が終了し新しく初期化されたメンバがアクティブメンバとなります。これをアクティブメンバの切り替えと言います。

C++17まで、共用体を定数式で使用することはできたものの、アクティブメンバの切り替えは定数式では行えませんでした。C++20からはこの制限が撤廃され、アクティブメンバの切り替えが定数式で行えるようになります。

```cpp
union Foo {
  int i;
  float f;
};

constexpr int use() {
  // 先頭のメンバ（i）をデフォルト初期化
  Foo foo{};  
  foo.i = 3;

  // アクティブメンバの切り替え、C++17までng
  foo.f = 1.2f; // ok、C++20

  return 1;
}

int main() {
  static_assert(use() == 1);
}
```

`std::optional`や`std::variant`、`std::string`はその実装に共用体が用いられており、それらのクラスを`constexpr`対応させるに当たってこのことが問題となったため、この制限は撤廃されることになりました。

## トリビアルなデフォルト初期化

- P1331R2 Permitting trivial default initialization in constexpr contexts  
  (https://wg21.link/p1331r2)

トリビアルなデフォルト初期化とは、型`T`の変数宣言において初期化子を書かずに初期化した際に、組み込み型およびコンストラクタがなくデフォルトメンバ初期化のされていないメンバだけを持つクラス型について行われる初期化方法のことです。つまりはその初期化に際してコンストラクタ呼び出しを行う必要がない初期化のことで、規格的にはその値は不定となり、通常の実装ではその初期化処理は省略されます。トリビアルなデフォルト初期化が可能な、組み込み型およびコンストラクタがなくデフォルトメンバ初期化のされていないメンバだけを持つクラス型、はまとめて*trivially default constructible*と呼ばれます。

これは初期化後の値の読み取りが未定義動作となるため、定数式で現れることが禁止されていました。

```cpp
template <typename T>
constexpr T f(const T& other) {
  T t;    // #1、型によって初期化方法が変わる
  t = other;
  return t;
}

struct trivial {
  bool b;   // 初期化子がない、トリビアル
};

struct non_trivial {
  bool b{}; // 初期化子がある、非トリビアル
};

int main() {
  constexpr auto x = f(1);              // ng、#1はトリビアルなデフォルト初期化
  constexpr auto y = f(trivial{});      // ng、#1はトリビアルなデフォルト初期化
  constexpr auto z = f(non_trivial{});  // ok、#1はデフォルト初期化
}
```

この制限は、この例のようにテンプレートな`constexpr`関数において入力の型次第でエラーとなるか変化する問題を引き起こしており、`constexpr`関数の処理について実行時のものとコードを共有することを妨げていました。トリビアルなデフォルト初期化がされている変数でも、その値の読み取りをしなければ実行時でも未定義動作とはなりません。

`constexpr`の実行時とコンパイル時の振る舞いを一貫させるためにこの制限は撤廃され、トリビアルなデフォルト初期化されている変数はその値が読み取られない限り定数式で現れることが許可されるようになります。

```cpp
int main() {
  constexpr auto x = f(1);              // ok、C++20
  constexpr auto y = f(trivial{});      // ok、C++20
  constexpr auto z = f(non_trivial{});  // ok、C++11
}
```

## `consteval`

- P1073R3 Immediate functions (https://wg21.link/p1073r3)

`constexpr`関数はコンパイル時と実行時のどちらでも実行できる関数で、定数式で実行されていることは保証されません。定数式で実行しようとしたときに実行できない場合に静かに実行時に実行を遅延することがあります。

そのような振る舞いは望ましくなく確実にコンパイル時に実行されていることが保証されていてほしい場合が多々あり、そのために新しく`consteval`関数（*immediate function* : 即時関数）が導入されます。

```cpp
consteval int f(int n) {
  return n * n;
}

int main() {
  int n = f(10);  // コンパイル時にf()は呼び出し済み
}
```

`consteval`は関数にのみ付加できて、`constexpr`と同じように使用できます。`consteval`関数は呼び出されたら確実にコンパイル時に実行され、実行できない場合は常にコンパイルエラーとなります。また、`consteval`関数の呼び出しは確実にコンパイル時に実行されなければならず、どうしても定数式で実行できないところに書いてある場合、例えそこが実際には実行されないところであってもコンパイルエラーとなります。

```cpp
int g(int n) {
  int n1 = f(n);  // ng、nは定数式で使用可能ではない
  int n2 = f(2);  // ok

  return n;
}

int main() {
  int n = 10;
  int m = f(n); // ng、nは定数式で使用可能ではない
}
```

関数`g()`内の1つ目の`f()`の呼び出し`f(n)`は、`g()`が使用されなかったとしてもコンパイルエラーとなります。逆に、2つ目の`f(2)`は`g()`の呼び出しとは関係なくコンパイル時に必ず実行されています。

### `consteval`コンストラクタ

`consteval`はコンストラクタにも付加することができて、そのコンストラクタを即時関数にします。すなわち、`consteval`コンストラクタの呼び出しは必ず定数式で実行されなければなりません。それを利用すると、引数のコンパイル時バリデーションを実装することができます。

```cpp
struct literal_zero {

  // constevalコンストラクタ、intからの暗黙変換が可能
  consteval literal_zero(int&& n) {
    // 引数値のチェックの実装
    if (n != 0) {
      throw "Compare with zero only!";
    }
  }
};

struct C {
  // 0リテラルのみと比較可能とする
  friend bool operator==(const C&, literal_zero) {
    return true;
  }
};

int main() {
  C c{};
  
  // これはok
  bool b1 = (c == 0);
  bool b2 = (0 == c);

  // これはng、すべてコンパイルエラーとなる
  bool b3 = (c == 1);
  bool b4 = (c == -1);
  bool b5 = (c == 10);
}
```

これは`<=>`の比較カテゴリ型の超簡易な実装で、ゼロリテラルとの比較以外をコンパイルエラーとしています。`C`の`==`の引数にて`literal_zero`で比較対象の整数値を受けていますが、`literal_zero`の唯一のコンストラクタは`consteval`であり、整数リテラルから`literal_zero`への暗黙変換は必ずコンパイル時に行われ、`literal_zero`の`consteval`コンストラクタで行われる引数値のチェックもまた必ずコンパイル時に行われます。そして、値が`0`でない場合は`throw`式で例外を投げようとすることでコンパイルエラーを起こします。

このテクニックはC++20で導入された`<format>`（文字列フォーマットライブラリ）にて、フォーマット文字列のコンパイル時チェックの実装に使用されています。

## `std::is_constant_evaluated()`

- P0595R2 `std::is_constant_evaluated()` (https://wg21.link/p0595r2)

`constexpr`関数は実行時とコンパイル時の両方で実行可能な関数ですが、その実装は実行時とコンパイル時で共通でなければなりません。実行時により効率的な実装が選択できる場合など、実行時とコンパイル時で処理を切り替えたい場合がよくありました。

そのために、`std::is_constant_evaluated()`が導入されます。これはコンパイル時に評価された場合に`true`を返し、それ以外の場合は`false`を返す関数です。これを用いると、コンパイル時に実行される処理と実行時の処理とで実装を分けることができます。

```cpp
constexpr double power(double b, int x) {
  if (std::is_constant_evaluated() && x >= 0) {
    // コンパイル時用実装
    double r = 1.0, p = b;
    unsigned u = (unsigned)x;
    while (u != 0) {
      if (u & 1) r *= p;
      u /= 2;
      p *= p;
    }
    return r;
  } else {
    // 実行時はstd::powを使用する
    return std::pow(b, (double)x);
  }
}

int main() {
  constexpr double kilo = power(10.0, 3); // コンパイル時実行
  int n = 3;
  double mucho = power(10.0, n);  // 実行時実行
}
```

ただし、`std::is_constant_evaluated()`は厳密には、定数式で`true`を返すのではなく定数式であることが確実な文脈で`true`を返すものです。そのため、`cosntexpr if`や`static_assert`の条件式では常に`true`となり、定数伝播などコンパイラの最適化の一環としてコンパイル時に評価される場合は`false`を返します。

```cpp
double thousand() {
  return power(10.0, 3);  // 必ず実行時実行
}

int f() {
  if constexpr (std::is_constant_evaluated()) {
    return 0; // 常にこちらが選択される
  } else {
    return 1;
  }
}

int main() {
  int n = f();  // 常に0
  static_assert(!std::is_constant_evaluated()); // 常にエラー
}
```

`thousand()`自体は`constexpr`ではないのでその中で呼ばれている`power(10.0, 3)`は定数式で実行可能な文脈ではないとみなされ、`power()`定義の`std::is_constant_evaluated()`は常に`false`を返します。ただし、引数はコンパイル時に確定しているためコンパイラは最適化の一環としてコンパイル時に実行しようとするかもしれません。しかしその場合でも、`power()`定義の`std::is_constant_evaluated()`は常に`false`を返します。

### `if consteval`

- P1938R3 `if consteval`(https://wg21.link/p1938r3)

`std::is_constant_evaluated()`は`constexpr if`で使用された時には常に`true`となります。しかし、この挙動はかなり非直感的でした。さらに、`consteval`関数とともに使用した時に分かりづらい振る舞いをすることがありました

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

関数`f()`を同じように呼び出しているはずですが、`g()`と`h()`で結果が異なっています。どちらの`f()`の呼び出しも必ずコンパイル時に呼ばれるはずで、何もおかしいところはないはずですが・・・

`g()`と`h()`の差異は、関数引数`i`が定数式であるか否かです。`consteval`関数は必ずコンパイル時に呼び出されるのでその引数は必ず定数式ですが、`constexpr`関数は実行時にも呼び出せるためそうではありません。そのため、`g()`内部では非定数式の`i`を用いて`consteval`関数`f()`を呼び出そうとしてエラーとなります。とはいえ、`std::is_constant_evaluated()`による分岐によって`f(i)`が呼び出されるのはコンパイル時だけであることは（プログラマからは）わかっています。

C++23では、これらの問題の解消のために`if (std::is_constant_evaluated())`の糖衣構文である`if consteval`が導入されます。これによって、`std::is_constant_evaluated()`を`constexpr if`で使おうとする原因（`if`が実行時実行であるように思える）が取り除かれ、関数引数が定数式とみなされない問題も解消されます。

```cpp
constexpr int g(int i) {
  if consteval {
    return f(i) + 1; // ok、C++23から
  } else {
    return 42;
  }
}
```

`if consteval`の`true`となるブロックは必ずコンパイル時に実行される文脈として特別扱いされ、`constexpr`関数の引数の扱いが`consteval`と同等になります。

## インラインアセンブリ

- P1668R1 Enabling `constexpr` Intrinsics By Permitting Unevaluated inline-assembly in `constexpr` Functions (https://wg21.link/p1668r1)

前項の`std::is_constant_evaluated()`を用いると、1つの関数定義に実行時とコンパイル時の両方の処理を書いておくことができるようになりますが、インラインアセンブリを用いているコードは`asm`宣言が定数式で現れることができないため`constexpr`とするには別の関数にする必要がありました。インラインアセンブリを用いているコードを最小限の変更で`constexpr`化したいという需要から、`asm`宣言は定数式で評価することはできないものの`constexpr`関数に書くことができるようになります。

```cpp
constexpr double fma(double a, double b, double c) {
  if (std::is_constant_evaluated()) {
    // コンパイル時のコード
    return a * b + c;
  } else {
    // 実行時のコード、定数式でここに到達するとエラー
    asm volatile ("vfmadd213sd %0,%1,%2" : "+x"(a) : "x"(b),"x"(c));
    return a;
  }
}
```

副次的ですが、このように定数式用のシンプルなコードと実行時用の複雑なアセンブラが併記されている事で、インラインアセンブリが何をしているのかがわかりやすくなります。

## `constinit`

- P1143R2 Adding the constinit keyword (https://wg21.link/p1143r2)

グローバル変数（名前空間スコープの変数、静的変数）は定数初期化が可能なものはコンパイル時に初期化され、できないものは実行時（基本的にはプログラム開始時）に動的初期化されます。動的初期化においては一部の変数の初期化順序が不定となるため、初期化順に依存するコードが未定義動作に陥る問題があり、「Static Initialization Order Fiasco」として知られています。

静的変数は可能であれば静的初期化してコンパイル時に初期化を終わらせてしまうことでそのような問題を回避できますが、ある静的変数が動的初期化されているのかいないのか、あるいは静的初期化されているのかいないのかは簡単には判断できず、少しのコードの変更でそれが変化してしまっても気づくことは困難でした。

`constinit`指定子は静的変数に対して付加して、その変数が静的初期化（特に、定数初期化）されていることを保証し、動的初期化が必要となる場合にコンパイルエラーとするものです。

```cpp
constinit int n = 10;       // ok
constinit double pi = 3.14; // ok

constinit double pi2 = pi * 2.0; // ng、piは定数式で使用不可

int main() {
  constinit static int m1 = 20; // ok

  constinit int m2 = 30;  // ng、ローカル変数

  n = 20; // ok、constinitは暗黙constではない
}
```

`constinit`は変数宣言にのみ指定することができ、その変数は静的変数（グローバル変数や関数ローカル`static`変数など）である必要があります。`pi2`のように、その初期化がコンパイル時に完了しない`constinit`変数はコンパイルエラーとなります。

定数初期化は非リテラル型のオブジェクトについても条件を満たせば行うことができて、そのようなクラス型には例えば`std::unique_ptr`や`std::mutex`があります。

```cpp
// 共ににok
constinit std::mutex m{};
constinit std::unique_ptr<int> p = nullptr;

// これはng
constinit std::unique_ptr<int> p = new int{};

int main() {}
```

## 動的メモリ確保

- P0784R7 More constexpr containers (https://wg21.link/p0784r7)

当然のことながら、C++17までは定数式において動的メモリ確保（`new/delete`や`std::allocator`の使用）はできませんでした。C++20ではそれが許可され、同時に`std::string`と`std::vector`が定数式で使用できるようになります。

```cpp
constexpr int sum_vector(const std::vector<int>& vec) {
  int sum = 0;

  for (int n : vec) {
    sum += n;
  }

  return sum;
}

constexpr bool check_string(const std::string& str) {
  return str.find("cpp") != std::string::npos;
}

// どちらもok
static_assert(sum_vector({1, 2, 3, 4, 5}) == 15);
static_assert(check_string("cpp20 constexpr"));
```

これは、提案中の静的リフレクションのために`std::string`と`std::vector`をコンパイル時に利用できる必要性が生じたため、そのための障害を取り払うための変更です。

この定数式における動的メモリ確保はユーザーコードでも行うことができます。その際に使用可能なメモリ確保/解放の方法は`new/delete`式と`std::allocator/std::allocator_trits`のみで、`malloc/free`などその他のものは許可されていません。また、それらが呼び出す`::operator new/::operator delete`演算子はグローバルに定義されたオリジナルなものだけで、オーバーロードされた`::operator new/::operator delete`演算子を定数式で呼び出すことはできません。

当然ですが、動的に構築しようとしている型はリテラル型であり、定数式で構築（初期化）と破棄（デストラクタ呼び出し）ができる必要があります。

```cpp
constexpr int f() {
  int* p = new int; // ok、C++20

  *p = 20;
  int n = *p;

  delete p; // ok、C++20

  return n;
}
```

C++20時点では、コンパイル時に確保したメモリ領域を実行時に持ち越すことはできません。したがって、`new`で確保されたメモリは必ず`delete`されていなければならず、これは`std::allocator/std::allocator_trits`においても同じです。

```cpp
constexpr int f() {
  int* p = new int; // ng、deleteされていない

  *p = 20;
  int n = *p;

  return n;
}
```

これらの、定数式で呼び出される`::operator new`演算子の呼び出しは、コンパイラの最適化の一環として常に省略され、代わりの領域（おそらくスタック領域）があてがわれています。`std::allocator/std::allocator_trits`も内部（`allocate()`）で`::operator new`を呼び出すため同様で、対応する`::operator delete`演算子の呼び出しも省略されます。すなわち、C++20コンパイル時動的メモリ確保は定数式にヒープ領域を導入しているのではなく、デフォルトの`::operator new`の呼び出しを別の領域をあてがう形に置き換えることで行われています。

\clearpage

# テンプレート

## `auto`による関数テンプレートの簡易定義

- P1141R2 Yet another approach for constrained declarations  
  (https://wg21.link/p1141r2)

C++14ではジェネリックラムダが導入され、ラムダ式の仮引数型を`auto`で宣言する事ができるようになりました。

```cpp
// C++14、ジェネリックラムダ
auto l = [](auto n, auto m) { return n; };
```

C++20では、この記法が通常の関数宣言にももたらされます。

```cpp
// auto引数宣言
void f(auto n, auto m);

// これは次の宣言と等価
template<typename T, typename U>
void f(T n, U m);
```

これは、制約付き`auto`によって制約された関数テンプレートを簡易定義できるようにするために導入されました。

```cpp
// コンセプトによる制約
void f(std::integral auto n, std::floating_point auto m);
```

ここでのコンセプトの第一引数は`std::integral<decltype((n))>`のように補われています（これは`class/typename`の位置にコンセプトを指定して制約するときなどと同じです）。この`auto`は変数宣言や戻り値型の`auto`とほぼ同じ事が可能で、`auto&&`や`const auto&`などのようにもできます。ただ、`const`が入る時はコンセプト指定の位置に注意が必要です。

```cpp
// autoに対するコンセプトはautoの直前
void f1(const std::integral auto& n);  // ok
void f2(std::integral auto const & n); // ok
void f3(std::integral const auto& n);  // ng
```

なお、この`auto`の位置に`decltype(auto)`を使用することはできません。

## `typename`の省略

- P0634R3 Down with `typename`! (https://wg21.link/p0634r3)

テンプレートパラメータそのもの、あるいはそれに依存しているクラス名からそのメンバ型を取得する時、静的メンバ変数との区別がつかない事から`typename`によって型名であることを指定しなければなりません。ただし、このルールには僅かばかり例外があり、基底クラスのリストとコンストラクタ初期化子リストでは`typename`は不要でした（むしろその2カ所では`typename`を使用できません）。

```cpp
template<typename T>
struct X {
  using Y = T;
};

template<typename U>
void f(U u) {
  X<U>::Y y{};  // ng、typenameが必要
}

template<typename T>
struct S : T::type  // ok、基底クラスリスト
{
  S() : T::type{}   // ok、コンストラクタ初期化子リスト
  {};
}
```

特別扱いされているこの2つの箇所では型のみが使用可能であり、型名以外が現れない事が明確であるため`typename`は不要とされていました。同様に型名以外が現れ得ない場所はいくつか存在しており、C++20からはそれらの所でも`typename`を省略できるようになります。

型名しか現れないコンテキストとは次の場所です

- `new`式の型名指定
    - `auto* p = new T::type {};`
- `using`による型エイリアス宣言の右辺
- 型変換演算子宣言の型名指定
- 後置戻り値型
- テンプレートパラメータのデフォルト引数
- 各種キャスト式の型名指定
- 名前空間スコープの関数宣言の戻り値型
- 名前空間スコープの変数宣言の変数の型
- クラスメンバ宣言に指定する型名
    - クラススコープに直接現れる型名指定のほぼすべて
- メンバ関数の引数の型（デフォルト引数を持たない場合のみ）
- ラムダ式の引数の型（デフォルト引数を持たない場合のみ）
- `requires`式の引数の型（デフォルト引数を持たない場合のみ）
- 非型テンプレート引数の型

```cpp
template<class T>
T::R f();       // ok、名前空間スコープ関数宣言の戻り値型

template<typename T>
T::V var = {};  // ok、名前空間スコープ変数宣言の型

template<class T>
struct S {
  using Ptr = PtrTraits<T>::Ptr;  // ok、usingエイリアスの右辺

  T::R f(T::P p) {                // ok、クラススコープ
    return static_cast<T::R>(p);  // ok、static_cast
  }

  auto g() -> S<T*>::Ptr;         // ok、後置戻り値型

  operator T::type() const {      // ok、型変換演算子の型名
    return {};
  };
};

template<typename T,
         T::type NP,        // ok、NTTPの型名
         class U = T::type  // ok、テンプレートパラメータのデフォルト引数
        >
void g();

template<typename T>
void f() {
  T::X x{};     // ng、関数スコープ
  void h(T::X); // ng、関数スコープ
}
```

ここでokで示しているところは、C++17までは`typename`が必要でした。

## クラス型の非型テンプレート引数

- P1907R1 Inconsistencies with non-type template parameters  
  (https://wg21.link/p1907r1)

C++17まで、非型テンプレートパラメータ（以下NTTPと省略）となれる型は整数型やポインタ型をはじめとする組み込み型の一部に限られていました。ユーザー定義のクラス型は当然として、浮動小数点数型もNTTPとして扱う事ができませんでした。

C++20からは、クラス型や浮動小数点数型をNTTPとして扱えるようにNTTPとなれる型の制限が見直されます。C++20よりNTTPとなれる型は*structural type*と呼ばれる型のグループで、それは次のいずれかに該当するものです

- スカラ型
- 左辺値参照型
- リテラル型
    - 全ての基底クラスおよび非静的メンバ変数は`public`かつ`mutable`ではなく
    - 全ての基底クラスおよび非静的メンバ変数の型は*structural type*である

浮動小数点数型は1つ目のスカラ型に含まれており、クラス型は3つ目の条件を全て満たすものがNTTPとして利用可能です。なお、C++20のリテラル型の要件は（いずれかの）コンストラクタとデストラクタが定数式で実行可能である型です。ただし、参照/ポインタ型あるいはそれを内包するクラス型では一時オブジェクトへの参照やポインタを保持する事が禁止されています。

```cpp
template<double F>
double f1(double d) {
  return F * d;
}

template<std::size_t N>
struct fixed_str {
  const char str[N];
};

template<fixed_str<5> fstr>
void f2() {
  std::cout << fstr.str << std::endl;
}

int main() {
  // 共にok
  f1<3.14>(2.0);
  f2<fixed_str<5>{"test"}>();
}
```

また、NTTPはC++17から`auto`による簡易構文が利用可能で、クラス型ではクラステンプレートのテンプレート引数推論（CTAD）を利用する事ができます。

```cpp
// autoパラメータの制約
template<std::floating_point auto F>
double f1(double d) {
  return F * d;
}

template<std::size_t N>
struct fixed_str {
  char str[N];

  constexpr fixed_str(const char(&arr)[N]) {
    std::ranges::copy_n(arr, N, str);
  }
};

template<std::size_t N>
fixed_str(const char(&)[N]) -> fixed_str<N>;


template<fixed_str fstr>
void f2() {
  std::cout << fstr.str << std::endl;
}

int main() {
  // 共にok
  f1<3.14>(2.0);
  f2<"test fix string">();

  // ng
  f1<1>(0.0);
}
```

NTTPは`T n = arg`のように初期化されるため（例えば、`f2<"test">`と書くと`fixed_str fstr = "test"`のように初期化されている）、集成体初期化が効かずコンストラクタが必要になり、`fixed_str`の場合は文字数を推論するために推論補助が必要となります。

## 集成体テンプレートの実引数からのテンプレート引数推論

- P1021R4 Filling holes in Class Template Argument Deduction  
  (https://wg21.link/p1021r4)
- P1816R0 Wording for class template argument deduction for aggregates (https://wg21.link/p1816r0)

C++17で導入されたクラステンプレートのテンプレート引数推論（CTAD）は非常に便利な機能ですが、集成体に対してはそのままでは使用できませんでした。

```cpp
template<typename T>
struct vec3 {
  T x, y, z;
};

int main() {
  std::vector vec = {1, 2, 3, 4, 5};  // ok、std::vector<int>

  std::pair p = {1, 1.0}; // ok、std::pair<int, double>

  vec3 v = {1, 2, 3}; // ng
}
```

渡された引数に対応するコンストラクタがある場合は推論が可能ですが、集成体はコンストラクタを持たないことからそのままでは推論できず、推論補助が必要になっていました。

```cpp
template<typename T>
struct vec3 {
  T x, y, z;
};

// これが必要
template<typename T>
vec3(T, T, T) -> vec3<T>;

int main() {
  vec3 v = {1, 2, 3}; // ok
}
```

C++20ではCTADが集成体に合わせて拡張され、集成体初期化時の初期化子と対応するメンバの型からテンプレート引数を推論してくれるようになります。

```cpp
template<typename T>
struct vec3 {
  T x, y, z;
};

int main() {
  vec3 v = {1, 2, 3}; // ok
}
```

あるクラスについての推論補助の仕組みは、初期化子とマッチするコンストラクタおよび見つかった全ての推論補助を関数テンプレートとして抽出しオーバーロード解決によって1つを選んだうえで、コンストラクタの場合はそのテンプレートパラメータ、推論補助の場合はその戻り値型からテンプレート引数を補うものです。C++17までは集成体初期化がそこでは考慮されていなかったため推論補助が必要となっていました。

C++20からは、集成体`C`の初期化時の引数リスト`{x1, ..., xi}`について、対応する`C`の要素`ei`が過不足なくぴったりと存在している場合に、`C`の要素`ei`の型`Ti`によって`C(T1, ..., Ti)`という仮想的なコンストラクタを考慮して同じ手順でテンプレート引数推論を行うようになります。

この時、その集成体のテンプレート引数にメンバとなっているクラステンプレートが依存していると`{}`省略が出来なくなります。

```cpp
template <typename T>
struct S {
  T x;
  T y;
};

template <typename T>
struct C {
  S<T> s; // テンプレートパラメータTに依存している
  T t;
};

C c1 = {1, 2};        // ng
C c2 = {1, 2, 3};     // ng、{}省略できない
C c3 = {{1u, 2u}, 3}; // ok, C<int>

template <typename T>
struct D { 
  S<int> s; // テンプレートな集成体だが、型は確定している
  T t; 
};

D d1 = {1, 2};    // ng、{}省略するなら初期化子は3つ必要
D d2 = {1, 2, 3}; // ok、{}省略可能
```

## エイリアステンプレートのCTAD

- P1021R4 Filling holes in Class Template Argument Deduction  
  (https://wg21.link/p1021r4)
- P1814R0 Wording for Class Template Argument Deduction for Alias Templates (https://wg21.link/p1814r0)

集成体と同様に、しかし異なる理由から、あるクラステンプレートの初期化時にエイリアステンプレートを経由するとCTADは無効になっていました。

```cpp
template<typename A, typename B>
struct my_pair {
  A a;
  B b;

  my_pair(A arg1, B arg2)
    : a(std::move(arg1)), b(std::move(arg2))
  {}
};

template<typename T>
using pair_with_int = my_pair<int, T>;

int main() {
  my_pair p1 = {1, 2};        // ok
  pair_with_int p2 = {1, 2};  // ng
}
```

エイリアステンプレートでは一部の型がすでに確定していたりして、元のクラスの推論補助やコンストラクタを利用して推論した結果とそれをどう擦り合わせるかが問題となってC++17には入らなかったのだと思われます。C++20では、何もしなくてもエイリアステンプレートに対してCTADが効くようになります。

```cpp
int main() {
  my_pair p1 = {1, 2};        // ok
  pair_with_int p2 = {1, 2};  // ok、C++20
}
```

この手順は少し複雑ですが、概ね次のように実行されています

1. エイリアステンプレートと元の型の対応する推論補助の右辺から、テンプレートパラメータの対応を求める
   - `template<typename T> using pair_with_int = my_pair<int, T>`と
   - `template<typename A, typename B> my_pair(A, B) -> my_pair<A, B>`から
   - `A -> int, B -> T`の対応を得る
2. その対応から元の推論補助にエイリアステンプレートのパラメータをフィードバックして置き換える
   - `int -> A, T -> B`へ代入する
   - `template<typename T> my_pair(int, T) -> my_pair<int, T>`
3. 得られた推論補助の右辺と左辺をエイリアステンプレートで逆に置き換えることで、エイリアステンプレートの推論補助を生成する
   - `my_pair<int, T> -> pair_with_int<T>`へ置換
   - `template<typename T> pair_with_int(int, T) -> pair_with_int<int, T>`
4. その推論補助を用いて通常の手順でCTADを行う

この概略では推論補助の存在を求めていますが、実際には推論補助は必要ありません。その場合は実引数とエイリアスに対応するコンストラクタから推論補助相当のものが補われて上記手順が実行されます。ただし、エイリアステンプレートに対して推論補助を書く事ができるようにはなっていません。生成されるものと同じ推論補助を書いてもエラーとなり、あくまでエイリアス元のクラスの推論補助やコンストラクタを利用して推論を行えるようになっただけです。

なお、これが標準の`pmr`コンテナのエイリアスにもたらされるのはC++23からとなります

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4, 5};      // ok、C++17
  std::pmr::vector vec = {1, 2, 3, 4, 5}; // ok、C++23
}
```

## `explicit(bool)`

- P0892R2 `explicit(bool)` (https://wg21.link/p0892r2)

コンストラクタがテンプレートパラメータに応じて`explicit`であるかどうかを切り替えることはよく行われており、`std::pair`などでも見る事ができます。そこでは、テンプレートパラメータの型が暗黙変換可能であるかどうかによって2つのコンストラクタを切り替えるような実装が行われます。

```cpp
template<typename T>
struct wrap1 {
  T value;

  // Tのexplicit性を考慮しない
  template<typename U = T>
  constexpr wrap1(U&& other)
    : value(std::forward<U>(other))
  {}
};

template<typename T>
struct wrap2 {
  T value;

  template<typename U = T, std::enable_if_t<std::is_convertible_v<U, T>, std::nullptr_t> = nullptr>
  constexpr wrap2(U&& other)
    : value(std::forward<U>(other))
  {}

  // Tがexplicitであるときはexplicitにする
  template<typename U = T, std::enable_if_t<!std::is_convertible_v<U, T>, std::nullptr_t> = nullptr>
  explicit constexpr wrap2(U&& other)
    : value(std::forward<U>(other))
  {}
};

struct S1 {
  explicit S1(int) {}
};

struct S2 {
  S2(int) {}
};

int main() {
  wrap1<S1> s1{1};     // ok
  wrap1<S1> s2 = {1};  // ok、ngになってほしい
  wrap1<S1> s3 = 1;    // ok、ngになってほしい
  wrap1<S2> s4{1};     // ok
  wrap1<S2> s5 = {1};  // ok
  wrap1<S2> s6 = 1;    // ok

  wrap2<S1> s7{1};     // ok
  wrap2<S1> s8 = {1};  // ng、explicit性を継承
  wrap2<S1> s9 = 1;    // ng、explicit性を継承
  wrap2<S2> s10{1};    // ok
  wrap2<S2> s11 = {1}; // ok
  wrap2<S2> s12= 1;    // ok
}
```

`explicit`コンストラクタは暗黙変換を禁止するためのコンストラクタなので、他の型で包まれた時でも暗黙変換されたくはありません。しかしそれを実現するにはかなり面倒で難解なコードを書かねばなりませんでした。ちなみに、この要素型の`explicit`性を継承するイディオムのことを*Perfect Initialization*と呼びます。

これを簡易に書くために、C++20ではコンストラクタに付加する`explicit`が`bool`値を取れるようになり、その真偽によってそのコンストラクタの`explicit`性が変化するようになります。

```cpp
template<typename T>
struct wrap {
  T value;

  // Tがexplicitであるときはexplicitにする
  template<typename U = T>
  explicit(!std::is_convertible_v<U, T>) constexpr wrap(U&& other)
    : value(std::forward<U>(other))
  {}
};

int main() {
  wrap<S1> s1{1};     // ok
  wrap<S1> s2 = {1};  // ng、explicit性を継承
  wrap<S1> s3 = 1;    // ng、explicit性を継承
  wrap<S2> s4{1};     // ok
  wrap<S2> s5 = {1};  // ok
  wrap<S2> s6 = 1;    // ok
}
```

`explicit(bool)`の中が`true`である時そのコンストラクタは`explicit`となり、`false`である時は非`explicit`になります。これによって、*Perfect Initialization*のようなことを書くのにコンストラクタを2つ書いてあれこれしなくても良くなり、SFINAEしない事からコンパイル時間の削減も期待できます。

\clearpage

# ラムダ式

## ジェネリックラムダのテンプレート構文

- P0428R2 Familiar template syntax for generic lambdas (https://wg21.link/p0428r2)

C++14でジェネリックラムダが導入され、ラムダ式はテンプレート`operator()`オーバーロードに近い表現力を手に入れました。ただ、テンプレート`operator()`オーバーロードと異なりテンプレートパラメータを明示的に指定するものではないため、少し使いづらいところがありました。

```cpp
// std::vectorの特殊化であるかをチェックする
template <typename T>
constexpr bool is_std_vector = false;
template <typename T>
constexpr bool is_std_vector<std::vector<T>> = true;

auto f = [](auto vector) {
  // 要素型は無視して、std::vectorだけを受け入れたい
  static_assert(is_std_vector<decltype(vector)>::value);  // #1

  // vectorの要素型を取得
  using T = typename decltype(vector)::value_type;  // #2
};
```

#1の箇所では、ジェネリックラムダの引数型を取得する方法がないために、引数を`decltype`して型を取得しています。さらにそこからメンバ型を取得したい場合#2のように書くことになります。この時、引数によっては`std::remove_cvref`のようなものも必要になり、さらに書く事が増え面倒になります。

また、可変長ジェネリックラムダにおいてその引数の完全転送（あるいはムーブ）を行う場合にも遭遇します。

```cpp
auto f = [](auto&&... args) {
  return foo(std::forward<decltype(args)>(args)...);
};
```

ジェネリックラムダではテンプレートパラメータパックを取得する手段がないので、関数パラメータパックの展開をしながら`decltype`するこのコードが最善でした。

このように、通常の関数テンプレートでは簡単にできる事がジェネリックラムダでは遠回りをしなければならず、ジェネリックラムダの構文は柔軟さを欠いていました。C++20では、ジェネリックラムダにテンプレート構文が導入され、テンプレートパラメータを明示的に書く事ができるようになり、これらの問題を解決する事ができるようになります。

先ほどの例は次のように書く事ができるようになります

```cpp
auto f1 = []<typename T>(const T& vector) {
  // 要素型は無視して、std::vectorだけを受け入れたい
  static_assert(is_std_vector<T>::value);

  // vectorの要素型を取得
  using V = typename T::value_type;
};

auto f2 = []<typename... Args>(Args&&... args) {
  return foo(std::forward<Args>(args)...);
};
```

ラムダ導入子（`[]`）の直後の`<>`内で通常の関数テンプレートと同様のテンプレートパラメータリストを書く事ができるようになります。このテンプレート構文は通常の関数テンプレートで`template<typename T, ...>`のように書くところの`<>`部分だけを書いているようなもので、同じ書き方をする事ができます。これによって、ジェネリックラムダは必要ならテンプレートパラメータ名を取得する事ができるようになり、それができない事で起きていた問題を解決できるようになります。

なお、このテンプレート構文は`auto`引数宣言と同居する事ができます。

```cpp
auto f = []<typename T>(T t, auto n) {
  // ...
};
```

## `[=, this]`と`this`の暗黙キャプチャの非推奨化

- P0409R2 Allow lambda capture `[=, this]` (https://wg21.link/p0409r2)
- P0806R2 Deprecate implicit capture of this via `[=]` (https://wg21.link/p0806r2)

C++17まではデフォルトコピーキャプチャ（`[=]`）において`this`ポインタの明示的なキャプチャは許可されていませんでした。なぜなら、`[=]`は`this`を暗黙にコピーキャプチャしているので重複指定となるためです。一方、`[&, this]`のようにデフォルト参照+`this`の明示キャプチャは当初から可能でした。これは`this`を参照キャプチャする事ができない（意味がない）ため明示キャプチャに意味があったからです。そして、デフォルトキャプチャ（`[=]`or`[&]`）はそのラムダ式本体内でクラスメンバが参照されているときに暗黙的に`this`をコピーキャプチャしていました（後述しますが、この挙動はかなり罠です）。

C++17で`this`の値キャプチャ（`*this`のコピーキャプチャ）が許可されたことで、`this`を明示的にキャプチャする事が望ましい場合があります。

```cpp
struct X {
  int x_;

  bool coin();

  std::function<void()> f(int n);
};

std::function<void()> X::f(int n) {
  return coin() ? [=]() { g(n, x_); }         // thisの暗黙キャプチャ
                : [=, *this]() { h(n, x_); }; // thisの値キャプチャ
}
```

このように、`[=]`と`[=, *this]`の両方がコード中にいくつか存在している場合その2つの構文の違いが忘れられがちになるため、キャプチャの種類を明示することで可読性を向上させたい事がありました。

C++20より、可読性と一貫性の向上のために`[=, this]`のように明示的な`this`のコピーキャプチャが許可されます。

```cpp
std::function<void()> X::f(int n) {
  return coin() ? [=,  this]() { g(n, x_); }  // thisの明示キャプチャ
                : [=, *this]() { h(n, x_); }; // thisの値キャプチャ
}

struct S {
  void f(int i);
};

void S::f(int i) {
  [&, this, i]{ };   // ok
  [=, *this]{ };     // ok、C++17
  [=, this]{ };      // ok、C++20から[=]と同じ意味
  [this, *this]{ };  // ng、thisが2回現れている
}
```

そして逆に`[=]`による`this`の暗黙キャプチャは非推奨とされ、`this`をコピーキャプチャする場合は常に`[=, this]`と書く事が推奨されるようになります。これは、あるクラススコープで`[=]`によってデフォルトコピーキャプチャをしながらラムダ式内部でそのクラスのメンバ変数を使用している時、そのクラスのメンバ変数はコピーキャプチャではなく暗黙にコピーキャプチャされた`this`ポインタを介した参照キャプチャになっており、構文の意図（コピーキャプチャする）と実際に起こる事（クラスメンバは実質参照キャプチャになる）が一致していなかったためです。

```cpp
struct X {
  int x;

  void foo(int n) {
    auto f = [=]() { x = n; };         // C++20より非推奨、このxはthis->xであり、xをコピーしていない
    auto g = [=, this]() { x = n; };   // C++20より推奨
  }
};
```

## ステートレスラムダのクロージャ型のデフォルトコンストラクタと代入演算子の定義

- P0624R2 Default constructible and assignable stateless lambdas  
  (https://wg21.link/p0624r2)

ラムダ式によって生成される関数オブジェクトは型を持ち、その型の事をラムダ式のクロージャ型と呼びます。C++17まで、ラムダ式のクロージャ型はコピー/ムーブコンストラクタとデストラクタを持つことは規定されていましたが、他のメンバは定義されていませんでした。たとえばデフォルト構築可能ではなく、コピー代入が行えません。

しかしそのために、テンプレートパラメータで関数オブジェクトの型を受け取って内部動作をカスタムするようなクラスにラムダ式を渡そうとすると、一度オブジェクトに受けてから`decltype`してテンプレートパラメータに指定しコンストラクタに渡す、という余分なコードを書かねばなりませんでした。そのようなクラスにはたとえば、`std::set/map`をはじめとする連想コンテナ（比較をカスタマイズ可能）や、`std::unique_ptr`をはじめとするスマートポインタ型（デリータをカスタイマイズ可能）があります。

```cpp
// ラムダ式を一度変数に受けて
auto greater = [](auto lhs, auto rhs){ return lhs > rhs;};
// decltypeしてクロージャ型を取得し
using greater_t = decltype(greater);
// クロージャ型を指定しラムダをコンストラクタに渡す
std::set<int, greater_t> set{greater};

// unique_ptrのデリータをカスタムするときも同様
auto custom_deleter = [](auto h) { close_handle(h); };
std::unique_ptr<handle_t, decltype(custom_deleter)> handle{&h, custom_deleter};
```

この様なクラス型は一般的に、受け取った関数オブジェクトをデフォルト構築することで保持しようとするため、通常はこの様にカスタムの関数オブジェクトを渡す必要はありません。しかし、クロージャ型がデフォルト構築可能ではないためラムダ式のオブジェクトをコンストラクタから渡さなければなりません。仮にクロージャ型がデフォルト構築可能であったとしても、構築したカスタムクラスのオブジェクトを代入しようとすると、クロージャ型に代入演算子が無い事からエラーが発生します。

C++20からは、ステートレス（キャプチャをしていない）ラムダ式のクロージャ型はデフォルトコンストラクタとコピー代入演算子を持つようになるため、これらの問題は解決されます。

```cpp
auto greater = [](auto lhs, auto rhs){ return lhs > rhs;};
// コンストラクタに渡さなくてもよくなる
std::set<int, decltype(greater)> set{};

auto custom_deleter = [](auto h) { close_handle(h); };
// コンストラクタに渡さなくてもよくなる
std::unique_ptr<handle_t, decltype(custom_deleter)> handle{&h};
```

C++20からのラムダ式のクロージャ型は、キャプチャをしていない場合そのデフォルトコンストラクタとコピー代入演算子は`default`定義され、キャプチャをしている場合は`delete`定義されるようになります。

## 評価されない文脈におけるラムダ式

- P0315R4 Wording for lambdas in unevaluated contexts (https://wg21.link/p0315r4)

C++17まで、ラムダ式は評価されない文脈（*unevaluated context*）に現れる事が禁止されていました。評価されない文脈とは`decltype, sizeof, noexcept, typeid`のオペランド（およびC++20からは`requires`式とコンセプト定義内）のことで、そこに書かれている式は評価（実行）されることはありません。

この制限はラムダ式によってSFINAEするのを禁止するために課されていましたが、その制限は後で別の形に書き直されたため、それを禁止する必要がなくなりました。そのため、C++20からは評価されない文脈にラムダ式を書けるようになります。

先ほどの「ステートレスラムダのクロージャ型のデフォルトコンストラクタと代入演算子の定義」で挙げたサンプルコードはさらに改善され次のようになります

```cpp
// 直接decltypeできるようになる
std::set<int, decltype([](auto lhs, auto rhs){ return lhs > rhs;})> set{};
std::unique_ptr<handle_t, decltype([](auto h) { close_handle(h); })> handle{&h};
```

この様に、ステートレスラムダのクロージャ型がデフォルト構築可能かつ代入可能となったことと組み合わせると、テンプレートパラメータで関数オブジェクト型を受けるタイプのクラスのカスタムをワンライナーで簡易に書くことができるようになります。

ただし、ラムダ式でSFINAEするのは相変わらず禁止されているため、ラムダ式のオブジェクトおよびそのクロージャ型が外部リンケージを持つ関数などのシグネチャの一部となることはできません。とはいえこれは多くの場合検出しづらいためエラーとならず、未定義動作につながります。

## 初期化キャプチャにおけるパック展開

- P0780R2 Allow pack expansion in lambda init-capture (https://wg21.link/p0780r2)

ラムダ式におけるキャプチャ時、パラメータパックを展開してキャプチャすることは出来たのですが、パック展開して初期化キャプチャすることは出来ませんでした。

```cpp
template<class F, class... Args>
auto delay_invoke1(F f, Args... args) {
  // パラメータパックの各要素をコピーキャプチャする、ok
  return [f, args...]() -> decltype(auto) {
      return std::invoke(f, args...);
  };
}

template<class F, class... Args>
auto delay_invoke2(F f, Args... args) {
  // より効率的にムーブする、ng
  return [f=std::move(f), ...args=std::move(args)]() -> decltype(auto) {
      return std::invoke(f, args...);
  };
}
```

初期化キャプチャにおけるパック展開は明示的に禁止されていたため、`delay_invoke2`のラムダの宣言はエラーとなります。同等の事をやろうとすると、`std::tuple`に受けてキャプチャし`std::apply`で取り出す、のような少し複雑なコードを書くことになります。

C++14当初の仕様では、初期化キャプチャがラムダ式のクロージャ型にその名前のメンバを追加していたため、初期化キャプチャでパック展開するとクラスのメンバとしてパラメータパックが導入されることになりました。しかし、それは可変長テンプレート以外の文脈でパラメータパックが出現することになり、それによってある式にパラメータパックが含まれているかを可変長テンプレートの外のあらゆる所でチェックしなければならなくなるなど、実装に多大な影響を与えます。それは殆ど実装不可能であったため、初期化キャプチャでのパック展開は許可されていませんでした。

その問題はコア言語のIssueとして解決され、初期化キャプチャはクロージャ型に名前付きメンバを導入しないようになりましたが、初期化キャプチャでパック展開できないという制限だけはそのまま残されていました。従って、その制限をする理由は既に取り除かれているため、初期化キャプチャでのパラメータパック展開が許可されました。

```cpp
template<class F, class... Args>
auto delay_invoke2(F f, Args... args) {
  // より効率的にムーブする、C++20からok
  return [f=std::move(f), ...args=std::move(args)]() -> decltype(auto) {
      return std::invoke(f, args...);
  };
}
```

ラムダ式のキャプチャ`[]`の中でパラメータパック`args`を`...`を前置することによって初期化キャプチャパックを導入します。初期化キャプチャパックは初期化キャプチャであるので初期化子が必要で必然的に同じパラメータパックを用いることになりますが、そちらには`...`による展開は必要ありません（`...args=args`で1つのパック展開）。初期化キャプチャパックはラムダ式本体内で使用可能で、それを再度展開するのは通常のパック展開と同じようにします。

```cpp
void g(auto... args) { 
  // ...
}

auto f(auto... args) {
  // パラメータパック展開による初期化キャプチャパックの宣言
  return [...args = std::move(args)] {
    return g(args...);  // 初期化キャプチャパックの展開
  }
}
```

初期化キャプチャはラムダ式のクロージャ型に名前付きメンバを導入しないため、初期化キャプチャパックには、それが宣言されたラムダ式の本体の外からアクセスする方法はありません。

## 構造化束縛した変数のキャプチャ

- P1091R3 Extending structured bindings to be more like variable declarations (https://wg21.link/p1091r3)
- P1381R1 Reference capture of structured bindings (https://wg21.link/p1381r1)

C++17まで、構造化束縛の各変数はラムダ式でキャプチャできるかどうかは不透明でした。構造化束縛宣言で指定される各変数名は実のところ変数ではなかったため、その扱いは通常の変数とは少し異なっていたからです。このことはコンパイラの実装に配慮したものでしたが、構造化束縛をラムダ式でキャプチャする事を妨げる技術的な理由が無かったため、C++20からは明示的に許可されるようになります。

```cpp
struct Foo {
  int a; 
};

int main() {
  auto [a] = Foo{};

  auto f1 = [a] { return a; };  // ok、aをコピーキャプチャする
  auto f2 = [=] { return a; };  // ok、aをコピーキャプチャする
  auto f3 = [&a] { return a; }; // ok、aを参照キャプチャする
  auto f4 = [&]  { return a; }; // ok、aを参照キャプチャする
}
```

GCCでは構造化束縛のキャプチャが早い段階から可能だったようです。

ただし、ビットフィールドを構造化束縛した変数は参照キャプチャする事ができません。

```cpp
struct Foo {
  int a : 1;
  int b;
};

int main() {
  auto [a, b] = Foo();

  // ビットフィールドに対応する変数は常に参照キャプチャ禁止
  auto f1 = [&] { return a; };      // ng、C++17から
  auto f2 = [&a = a] { return a; }; // ng、C++17から
  auto f3 = [&a] { return a; };     // ng、C++17から

  // 非ビットフィールドに対応する変数はok
  auto g1 = [&] { return b; };      // ok、C++20から
  auto g2 = [&b = b] { return b; }; // ok、C++17から
  auto g3 = [&b] { return b; };     // ok、C++20から
}
```

ビットフィールドは特殊なメンバ変数であり、型がなくそのアドレスを取得する事ができないためビットフィールドへの参照を作成する事ができません。構造化束縛もそれに従います。

\clearpage

# 構造化束縛

## `static/thread_local`の許可

- P1091R3 Extending structured bindings to be more like variable declarations (https://wg21.link/p1091r3)

構造化束縛宣言は普通の変数宣言とは異なるもので、追加の指定はCV修飾くらいしか行えませんでした。構造化束縛をさらに普通の変数宣言に近づけるべくこの制限が緩和され、`static/thread_local`の2つの記憶域クラス指定が行えるようになります。

```cpp
auto f() -> std::pair<int, int>;

// static変数として構造化束縛
// 内部リンケージを持ち、プログラムの実行中生存する
static auto [s1, s2] = f();

// thread_local変数として構造化束縛
// スレッドごとに固有となり、各スレッドにおいてその実行中のみ生存する
thread_local auto [t1, t2] = f();
```

許可されたのはこの二つのみで、`inline`や`constexpr`などは許可されず、属性指定も行えません。

## 非公開メンバへのアクセス（DR）

- P0969R0 Allow structured bindings to accessible members  
  (https://wg21.link/p0969r0)

C++17で導入された構造化束縛では基本的にクラスの`public`メンバにしかアクセスできません。しかしこのことは他の宣言との一貫性がありませんでした。他の宣言ではクラスのメンバにアクセスできるかどうかはそれが`public`であるかどうかではなく、そのコンテキストからそのメンバがアクセス可能であるかに依存します。それは例えば`friend`関数で見る事ができます

```cpp
struct A {
  friend void foo();  // frined宣言、#1
private:
  int i;
};

// #1の定義、Aの非公開メンバにアクセス可能
void foo() {
  A a;

  auto x = a.i; // ok
  auto [y] = a; // ng
}
```

`x, y`の2つの宣言は同じスコープで同じメンバ変数へ同じオブジェクト経由でアクセスしようとしますが、構造化束縛だけはエラーとなっていました。また、よりシンプルには、クラス内部からすら構造化束縛ではそのメンバにアクセスできません。

```cpp
class C {
  int i;

  void foo(const C& other) {
    auto [x] = other; // ng、Cのメンバ変数は非公開
  }
};
```

この制限には特段の理由がなく一貫しておらず混乱の元だったので、構造化束縛におけるクラスメンバへのアクセスルールは他の宣言と同等になるようになります。具体的には、あるクラス`C`のオブジェクト`o`に対して構造化束縛しようとするコンテキストで、`C`の直接または基底クラスのメンバ`m`に対して`o.m`というアクセスが可能であれば、構造化束縛は`m`にアクセスする事ができるようになります。それによって、上記2つの例のngケースは全てコンパイルが通るようになり、他の所でも同様のアクセス制限が無くなります。

なお、これはC++17に対する欠陥報告であるため、変更を適用済みのコンパイラではC++17モードでも修正された挙動となります。

## 呼び出す`get`メンバ関数の考慮の変更（DR）

- P0961R1 Relaxing the structured bindings customization point finding rules (https://wg21.link/p0961r1)

tuple-likeな型に対する構造化束縛は、まずメンバ`get()`関数を探し、それがなければADLで非メンバ`get()`を探しに行きます。この時、メンバで`get()`を持っていればそれを優先して使用しようとするため、構造化束縛の意図とは異なる`get()`メンバ関数を持つ型ではコンパイルエラーとなっていました。例えば、`std::shared_ptr`などスマートポインタ型は生ポインタを得るための`get()`を持ちます。スマートポインタ型を構造化束縛で使用することはできませんが、そこから派生したクラスで構造化束縛にアダプトしようとするとその問題に遭遇することになります。

tuple-likeな型に対する構造化束縛で使用する`get()`関数は実際には非型テンプレートパラメータをとる`get<I>()`の形式のものです（`I`はインデックス数値）。そのため、tuple-likeな型に対して最初にメンバ`get()`を探す際に考慮する関数は非型テンプレートパラメータをとるもののみとし、それがないときに非メンバ`get()`を探しに行く、というように探索の手順が修正されます。それによって、メンバ`get()`関数があったとしても、それが非型テンプレートパラメータを取るものでなければ構造化束縛で使用すべき関数であるとはみなされず無視されるようになります。

```cpp
// privateとはいえメンバget()を継承している
struct X : private std::shared_ptr<int> {
  std::string fun_payload;
};

// Xに対して使って欲しいget()、#1
template<int N>
std::string& get(X& x) {
  if constexpr(N == 0) return x.fun_payload;
}

// タプルインターフェースの実装
namespace std {
  template<>
  class tuple_size<X> : public std::integral_constant<int, 1> {};

  template<>
  class tuple_element<0, X> {
  public:
    using type = std::string;
  };
}

int main() {
  X x;
  auto& [y] = x;  // C++17ではng、メンバget()が見つかり優先されてしまう
                  // C++20ではok、メンバget()はテンプレートではないため無視される
}
```

この`X`は`std::shared_ptr<int>`から継承した`get()`メンバ関数をプライベートとはいえ持っています。C++17の構造化束縛では`X`のメンバ`get()`があればとにかくそれを優先してしようとしますが、`X`のそれはテンプレートではなくアクセスもできないためコンパイルエラーとなります。この変更後では同じコードのままでも、`X`のメンバ`get()`は非型テンプレートパラメータを取るものではないため無視され、ADLによって非メンバ`get()`を探しに行き、`#1`のフリー`get()`が発見され意図通りに構造化束縛されます。

これはC++17に対する欠陥報告です。

\clearpage

# ユニコード文字型

## `char8_t`

- P0482R6 char8_t: A type for UTF-8 characters and strings  
  (https://wg21.link/p0482r6)

C++11にてUTF-16/UTF-32文字を表すための`char16_t/char32_t`型が導入されましたが、UTF-8に対応する文字型は導入されませんでした。それは`char`で表現すれば十分であるとのことで、C++11にてUTF-8文字列リテラル（`u8`）が導入されたときもその結果の型は`const char*`でした。

`char`の値は実装定義の実行時エンコーディングであってそのエンコーディングはUTF-8であるとは限らず、UTF-8と1対1対応する文字型が存在しない事によってオーバーロードなど型によって文字コード判別を行うインターフェース設計が妨げられており、文字コード指定のためにユーザーがエンコーディングを明示的に指定するインターフェースを採用せざるを得なくなっていました。それは、`std::filesystem::path`のコンストラクタと代入演算子で見ることができ、明示的にUTF-8文字列を受け取るために`std::filesystem::u8path()`というファクトリ関数を別途用意しています。

また、`char`は符号付か符号なしかが指定されていない（少なくとも8bit幅であることは保証されている）一方、UTF-8は非Asciiの1文字を複数バイトで表現し、非Ascii文字の各バイトの値は[128, 255]の範囲の値となり、`char`の8bit目が使用されます。しかし、`char`が符号付の場合8bit目は符号ビットである可能性があり、それによってUTF-8文字を格納した`char`値とUTF-8文字のコードポイント値の比較が実装定義となる問題もあります。

```cpp
bool is_utf8_multibyte_code_unit(char c) {
  return c >= 0x80;
}
```

この関数はUTF-8文字`c`がAscii範囲外の文字であるかを判定する関数ですが、`char`が符号付の場合はこの結果は非Ascii文字に対して常に`false`になります。`0x80`は`signed char`の表現可能な範囲を超えているため`int`の値として読まれ、左辺の`c`は整数昇格によって`int`に変換されてから比較されますが、`c`の値は非Ascii文字である場合は負の値として比較され、`0x80`は正の値であるため`false`となります。

さらに、`char`の配列あるいは`char*`は他の型のエイリアスとなることが許可されており、コンパイラの最適化の対象外とされることがあります。UTF-8文字列を表現する専用の型があることで、UTF-8文字列の処理に対する更なる最適化が可能になる可能性があります。

C++17策定時点でも、UTF-8はWebの世界での事実上の標準文字コードとなっており、C++のプログラムもWebとのやり取りをする機会が増加し、また今後増加していく事が想定されるため、これらの問題は時間の経過とともに深刻さを増していきます。そのため、C++20ではUTF-8エンコーディングと一対一対応する文字型である`char8_t`を導入し、これがC++11で導入されていたかのように各種仕様を変更します。主な変更は以下の点です

1. 新しい文字型`char8_t`の追加
    - `unsigned char`とほぼ同じ性質を持つが、`char, (unsigned|signed) char`とは型として区別され、エイリアスルールで特別扱いされない
2. UTF-8文字列リテラル（`u8"str..."`）の型を`const char*`から`const char8_t*`へ変更
3. UTF-8文字リテラル（`u8'c'`）の型を`char`から`char8_t`へ変更
4. ユーザー定義リテラルの`char8_t`対応

特に2つ目と3つ目の変更はC++11に対する破壊的変更となります。

```cpp
template<typename>
struct ct;

template<>
struct ct<char> {
  using type = char;
};

void f(const char* s);


int main(){
  const auto* u8s = u8"text";   // u8sの型はC++17まではconst char *、C++20からはconst char8_t *
  const char* ps = u8s;         // C++17まではok、C++20からはng

  auto u8c = u8'c';             // u8cの型はC++17まではchar、C++20からはchar8_t
  char* pc = &u8c;              // C++17まではok、C++20からはng

  std::string s = u8"text";     // C++17まではok、C++20からはng

  f(u8"text");                  // C++17まではok、C++20からはng

  ct<decltype(u8'c')>::type x;  // C++17まではok、C++20からはng
}
```

## `char16_t/char32_t`

- P1041R4 Make `char16_t/char32_t` string literals be UTF-16/32  
  (https://wg21.link/p1041r4)

C++11でUTF-16/UTF-32文字を表す型として`char16_t/char32_t`とそのリテラルを得るための`u/U`プリフィックスが導入されましたが、実際の所それらの型の値及びリテラルの結果がUTF-16/UTF-32の文字列となるかどうかは規定されていませんでした。

とはいえ規格の文面は`char16_t/char32_t`のエンコードが明らかにユニコードであることをほのめかしており、`char16_t`のエンコードとしてサロゲートペアを認めていたり、`char32_t`の文字は1文字が複数の`char32_t`値に跨がらないことを指定していて、またどちらの文字型もユニコード文字のすべての範囲を表現できるように指定されています。

これらの特徴を持つエンコーディングはUTF-16/UTF-32以外に無く、全ての実装は`char16_t/char32_t`のエンコーディングとしてUTF-16/UTF-32を使用していたため、C++標準としても`char16_t/char32_t`のエンコーディングを明確にUTF-16/UTF-32であると規定することになりました。

この変更は実際の慣行を標準に適用しただけなのでプログラマには影響はありませんが、`char16_t/char32_t`を使用するときにエンコーディングを気にする必要がなくなり、`__STDC_UTF_16__/__STDC_UTF_32__`の様なマクロを使用する必要もなくなるなど、余分なことを考えなくてよくなります。

\clearpage

# 属性

## `[[no_unique_address]]`

- P0840R2 Language support for empty objects (https://wg21.link/p0840r2)

C++では任意のオブジェクトは必ず一意のアドレスを持ち、一切のメンバ変数が無い空のクラスであったとしてもそれは同様で、それによってクラスのサイズは必ず1以上になります。これは、`new`によってクラス型のメモリ領域を動的確保したときでも、異なる`new`式の呼び出しが異なるアドレスを返すことを保証するための要請です。

```cPP
struct empty {};

static_assert(0 < sizeof(empty)); // 常に成り立つ

int main() {
  empty e1, e2;

  assert(&e1 != &e2); // 常に成り立つ
}
```

とはいえ、この様な空のクラスをメンバとして持つ時にはそのために余分なサイズを割り当ててほしくはありません。そのために、*Empty Base Optimization*（EBO）という最適化テクニックがありました。クラスの継承時に基底クラスのサイズがゼロならそのレイアウトを派生クラスと一致させることで基底のサイズが0のクラス型に余分な領域を割り当てないことが許可されており、EBOはそれを利用したものです。

```cpp
struct empty {};

struct S : empty {
  int n;
};

static_assert(sizeof(S) == sizeof(int));  // ok
```

（MSVCなど、一部のコンパイラではEBOを働かせるために追加の設定が必要なことがあります）

とはいえこのテクテニックはコードが複雑になりがちで、継承元の空クラスのメンバが派生クラスのスコープに漏れるなど、デメリットもありました。

C++20より、これを簡単に行うために`[[no_unique_address]]`という属性が追加されます。これをメンバ変数に指定してしておくことで、そのメンバのサイズが0なら余分な領域を取らないように最適化されます。

```cpp
struct empty {};

struct S {
  [[no_unique_address]] empty e;
  int n;
};

static_assert(sizeof(S) == sizeof(int));  // ok
```

EBOによるサイズ削減テクニックはテンプレートパラメータ型をメンバとして保持する必要がある場合によく行われ、標準コンテナのアロケータ型で特に目にすることができます。

```cpp
template<typename Key, typename Value,
         typename Hash, typename Pred, typename Allocator>
class hash_map {
  // それぞれサイズが0なら余分な領域を取らない
  [[no_unique_address]] Hash hasher;
  [[no_unique_address]] Pred pred;
  [[no_unique_address]] Allocator alloc;
  Bucket *buckets;

  // ...
};
```

ただし、コンパイラは不明な（未実装の）属性を無視するため、`[[no_unique_address]]`の対応の有無、あるいはコンパイラに対するバージョン指定の有無でこの結果は静かに変化します。クラスレイアウトが変化するためそれはABIの破損を招き、ODR違反を起こします（それは診断不用の未定義動作のため、検出は困難）。この問題はMSVCのABI互換性保証を破損するため、MSVCでは`[[no_unique_address]]`による最適化を全てのC++バージョンにおいて当面無効化しています（代わりに`[[msvc::no_unique_address]]`が提供されてはいます）。

## `[[likely]]/[[unlikely]]`

- P0479R5 Proposed wording for likely and unlikely attributes  
  (https://wg21.link/p0479r5)

条件分岐（`if, switch`など）を書くとき、コードを書いている人から見るとその分岐の一方がどれくらいの頻度で実行されるのか、あるいはその条件が`true`となる可能性が高いのか低いのか、が良く見えている事はよくあります。しかし、その情報は必ずしもコンパイラには伝わらず、コンパイラはそのようなメタな情報に基づく条件分岐の最適化を行う事が出来ません。

条件分岐にそのようなプログラマの認識を伝え最適化を促進するためにそのための手段を実装が用意していることがあります（GCC/clangの`__builtin_expect`など）。しかし、そのような手段は移植性が無く、コンパイラによっては代替手段が無い場合もありました。

`[[likely]]/[[unlikely]]`属性はそれら実装の用意する条件分岐にプログラマの認識を伝えるための手段を標準化するもので、`[[likely]]`によってその分岐の分岐確率が高いことを、`[[unlikely]]`によってその分岐の分岐確率が低いことをコンパイラに囁きます。

```cpp
void f(int n) {
  // n == 0はほぼ来ない事を知っている
  if (n == 0) [[unlikely]] {
    // ...
  } else {
    // ...
  }
}

enum class category {
  success, fail, done
};

void g(category cat) {
  // ほぼ成功する事を知っていて、doneは殆ど無い事も知っている
  switch (cat) {
    [[likely]]
    case category::success :
      // ...
    case category::fail : 
      // ...
    [[unlikely]]
    case category::done :
      // ...
  }
}
```

とはいえ現代のCPUの分岐予測の性能はかなり高く、あるいはそもそもの分岐確率によっては、これらを適切に指定したとしても速くなるとは限りません。

## `[[nodiscard("msg")]]`

- P1301R4 `[[nodiscard("should have a reason")]]` (https://wg21.link/p1301r4)

C++17から導入された`[[nodiscard]]`は、主に関数に付加してその戻り値が捨てられた場合に警告を表示します。しかし、そこにその理由を表示しておく方法がありませんでした。

C++20から`[[nodiscard]]`は引数として文字列リテラルを取れるようになり、なぜそれを捨ててはいけないのかを表示することができるようになります。そして、そのメッセージはコンパイラの警告にも表示されます。

```cpp
struct data_holder {
private:
  std::unique_ptr<char[], data_deleter> ptr;
  std::vector<int> indices;

public:

  // 戻り値を使用しないとメモリリークする
  [[nodiscard("Possible memory leak.")]] 
  char* release() { 
    indices.clear();
    return ptr.release();
  }

  void clear() {
    indices.clear();
    ptr.reset(nullptr);
  }

  // 戻り値を使用しないとき、clear()と間違えている可能性がある
  [[nodiscard("Did you mean 'clear'?")]] 
  bool empty() const {
    return ptr != nullptr && !indices.empty();
  }
};

int main () {
  data_holder dh = /* ... */;

  // 重大な間違い、警告される
  dh.release();
  // 軽微な間違い、警告される
  dh.empty();

  return 0;
}
```

`[[nodiscard]]`はそれだけでもかなり有用性が高いですが、メッセージを付加できるようになった事でより的確に間違いを指摘することができるようになります。

## コンストラクタに対する`[[nodiscard]]`（DR）

- P1771R1 `[[nodiscard]]` for constructors (https://wg21.link/p1771r1)

`[[nodiscard]]`は関数もしくはクラスの宣言に付与し、その戻り値やオブジェクトが使用されずに破棄されてはならない事を示し、それが起きた場合に警告を表示します。しかし、C++17での導入時点ではコンストラクタに対して付加することを考慮しておらず、規定として曖昧になっていました。

コンストラクタ引数からのテンプレートパラメータ推論（CTAD）ではファクトリ関数を省略できクラス名をさも関数であるかのように書くことができるようになるため、変数を定義するのではなくコンストラクタを呼び出しただけ、の様な間違いが起こりやすくなることが想定されるためコンストラクタに対する`[[nodiscard]]`はCTADが導入されるC++17以降特に重要であり、それを明示的に許可することになりました。

特定のコンストラクタに`[[nodiscard]]`を付加し、そのコンストラクタから構築されたオブジェクトが使用されずに破棄された場合に警告を表示します。

```cpp
class unique_file {
  std::ifstream file_;
public:
  // リソース確保しないコンストラクタ #1
  unique_file() = default;

  // リソース確保するコンストラクタ #2
  [[nodiscard]]
  explicit unique_file(const std::string& filename)
    : file_(filename) {}
};

int main() {
  // どちらも変数を定義していない
  unique_file{};        // #1の呼び出し、警告なし
  unique_file{"a.txt"}; // #2の呼び出し、警告される
}
```

なお、これはC++17に対する欠陥報告であり、実装済みのコンパイラではC++17でコンパイルしたときにも有効になります。

\clearpage

# 集成体

## Designated Intilization

- P0329R4 Designated Initialization Wording (https://wg21.link/p0329r4)

*Designated Initialization*（指示付初期化）とは、集成体初期化時にその集成体のメンバ名を指定する形で初期化する構文の事です。C言語ではC99から可能だったものでしたが、C++20よりC++にも導入されます。

```cpp
struct point3d {
  double X, Y, Z;
};

int main() {
  // この二つの初期化構文は同じ意味・効果となる
  point3d v1 = { 2.0, 3.0, 1.0 };
  point3d v2 = { .X = 2.0, .Y = 3.0, .Z = 1.0 }; // C++20から
}
```

このような名前指定（`.X`や`.Y`）の事を指示子（*designator*）と呼びます。指示子を指定できること以外は普通の集成体初期化と同じ効果（初期化式の評価順の規定や縮小変換の禁止など）を持ちます。

ただしいくつか制約があり、ほとんど普通の集成体初期化でメンバ名を指定できるようになったくらいのことしかできません。

1. 指示子の順番はメンバの宣言順
2. 指示子を書くならば全てに指示子による初期化が必要
      - 指示子のあるなしが混在できない
3. 指示子はネストできない

```cpp
struct nest {
  int n;
  point3d p;
};

int main() {
  // 1. 指示子の順番はメンバの順番通りでなければならない
  point3d v1 = { .X = 1.0, .Z = 2.0, .Y = 1.0 }; // ng
  point3d v2 = { .X = 1.0, .Y = 1.0, .Z = 2.0 }; // ok

  // 2. 指示子はあるかないかのどちらか
  point3d v3 = { 1.0, .Y = 2.0, 3.0 };       // ng
  point3d v4 = { .X = 1.0, .Y = 2.0, 3.0 };  // ng

  // 3. ネストした集成体に対しては{}を使用する
  nest n1{ .n = 1, .p.X = 1.0, .p.Y = 1.0, .p.Z = 1.0 };    // ng
  nest n2{ .n = 1, .p = { .X = 1.0, .Y = 1.0, .Z = 1.0 } }; // ok
}
```

この3番目の例のように指示子による初期化では`{}`初期化を使用できます。また、順番が入れ替わらない限りは指示子を省略することもでき、その場合はデフォルトメンバ初期化があればそれによって、無ければ`{}`を指定したかの様に初期化されます。

```cpp
int main() {
  // {}を使った初期化ができる
  point3d v1 = { .X = {1.0}, .Y{1.0}, .Z{} };

  // 省略した要素は{}（この場合は0.0）で初期化される
  point3d v2 = { .X = 1.0, .Y = 1.0 };
  point3d v3 = { .X = 1.0 };

  // 順番通りであれば要素の初期化子は省略できる
  point3d v4 = { .X = 1.0, .Z = 1.0 };
  point3d v5 = { .Y = 1.0 };
}
```

このような指示付初期化は初期化するメンバ名を明確にすることによって初期化指定の間違いを発見しやすくする効果があり（間違えるとコンパイルエラーになる）、大量のメンバを持つ構造体の初期化を読みやすく書くことができます。また、共用体の初期化時には、初期化するメンバを指定して初期化することができます（これは以前は出来なかったことです）。

```cpp
union U {
  int n;
  double d;
};

int main() {
  U u1 = {10};          // nを10で初期化
  U u2 = { .n = 10 };   // 同上
  U u3 = { .d = 3.14 }; // dを3.14で初期化

  U u4 = { .n = 1, .d = 2.72 }; // ng、初期化子は1つでなければならない
}
```

## `()`による集成体初期化

- P0960R3 Allow initializing aggregates from a parenthesized list of values (https://wg21.link/p0960r3)

集成体初期化は`{}`によって行われるものでしたが、C++20より`()`でも行えるようになります。ただし、変数名に直接続く形しか許可されず、間に`=`が入る形は許可されません。

```cpp
struct aggregate {
  int n;
  std::string str;
};

int main() {
  aggregate a1{10, "braces"};
  aggregate a2(10, "parentheses");

  // これはng
  aggregate a3 = (10, "ng");
}
```

ただし、いくつか`{}`による集成体初期化と異なるところがあります。

1. 縮小変換が許可される
2. ネストする波かっこが省略できない
3. ネストする場合に丸かっこを使えない
4. 指示付初期化できない
5. 空の`()`を使えない

```cpp
struct narrow {
  int n;
  float f;
};

struct wrap {
  narrow n;
  int m;
};


int main() {
  unsigned int n = 10;
  long double d = 1.0;

  // 1. 縮小変換の許可
  narrow n1{n, d}; // ng
  narrow n2(n, d); // ok

  // 2. 波かっこ省略できない
  wrap w1{10, 1.0f, 20};    // ok
  wrap w2(10, 1.0f, 20);    // ng
  wrap w3({10, 1.0f}, 20);  // ok

  // 3. ネストして丸かっこを使えない
  wrap w4((10, 1.0f), 20);  // ng

  // 4. 指示付初期化できない
  narrow n3(.n = 10, .f = 1.0f);        // ng
  wrap w5({ .n = 10, .f = 1.0f}, 20);   // ok

  // 5. 空にすると関数宣言とあいまいになる
  narrow n4{};  // ok、変数宣言
  narrow n5();  // ok、ただしこれは関数宣言
}
```

これらの制約の大部分は`()`による集成体初期化がコンストラクタ呼び出しの意味論を踏襲するように設計されていることから来ています。このような`()`による集成体初期化は集成体をより通常のクラスに近づけるもので、主に、標準コンテナなどが備える`emplace`関数で集成体を初期化できなかった問題を解決するものです。

```cpp
int main() {
  std::vector<aggregate> v{};

  v.emplace_back(10, "emplace");            // ng、C++17まで
  v.emplace_back(aggregate{10, "emplace"}); // ok、ムーブ構築
}
```

`emplace`系関数はコンストラクタの引数を受け取ってその内部でコンストラクタを呼び出して構築を行うもので、（コンテナ等に対する）直接構築などとも呼ばれます。その実装は共通して、内部で構築する領域へ`placement new`することで構築します。

```cpp
// 単純なemplace関数の実装例
template<typename... Args>
T& emplace(Args&&... args) {
  // 構築したい領域をstorageとする
  // ここでT()を使っているために集成体初期化が行われない
  T* p = new(&storage) T(std::forward<Args>(args)...);

  // こうすると集成体初期化は行われるが、多くのクラス型で問題が起こる
  T* p = new(&storage) T{std::forward<Args>(args)...};

  return *p;
}
```

ここでの`new`による構築時に`T()`としていることによって、`T`が集成体である場合に集成体初期化が行われません（コピー/ムーブコンストラクタは呼ばれる）。そこを`{}`に変えてやれば集成体初期化は行われますが、`{}`と`()`の性質の違いから別の問題が発生します。`()`による集成体初期化はこのような場合にわざわざ集成体にコンストラクタを書いたり、ムーブ構築に切り替えたりしなくてもよくするために導入されました。このようなものにはほかにも、`std::make_shared()`や`std::make_unique()`、`std::make_from_tuple()`などがあります。

実際のところ、`()`による集成体初期化が必要となるのはこのようなライブラリの内部においてだけであり、普通に使う時は常に`{}`を使うべきです。

## ユーザー宣言コンストラクタの禁止

- P1008R1 Prohibit aggregates with user-declared constructors  
  (https://wg21.link/p1008r1)

C++11より集成体型では`default/delete`なコンストラクタを宣言することはできていましたが、C++20からは集成体型では一切のコンストラクタ宣言を行えなくなります。

これは、`default/delete`なコンストラクタの意図と集成体初期化がかみ合っていなかったための変更で、C++11以降のコードに対する破壊的変更となります。

まず、非集成体型においてデフォルト構築を禁止するためにコンストラクタの`delete`宣言を行っている場合に、同時に集成体の要件を満たすがために集成体初期化が可能になってしまうという問題がありました。

```cpp
struct delete_defctor {
  delete_defctor() = delete;
};

struct delete_init_int {
  delete_defctor() = delete;
  delete_defctor(int) = delete; // intから構築してほしくない

  int n = 10;
};

int main() {
  delete_defctor x;   // ng、デフォルトコンストラクタは削除されている
  delete_defctor x{}; // ok、集成体初期化

  delete_init_int x(3); // ng、コンストラクタは削除されている
  delete_init_int x{3}; // ok、集成体初期化
}
```

また、`default/delete`宣言の位置によって集成体であるか無いかが変化してしまっていました。コンストラクタ（というかメンバ関数全般）はその定義をクラス外で行うことができます。そして、`default/delete`宣言もクラス外で行うことができます。

```cpp
// C++17までは集成体
struct aggregate {
  aggregate() = default;

  int n;
};

// 集成体ではない
struct not_aggregate {
  not_aggregate();

  int n;
};

not_aggregate::not_aggregate() = default;


int main() {
  aggregate x{10};      // ok
  not_aggregate y{10};  // ng
}
```

これらの非自明な挙動を修正するために、集成体型では一切のコンストラクタ宣言が禁止されました。

\clearpage

# 範囲`for`

## 初期化式の指定

- P0614R1 Range-based `for` statements with initializer (https://wg21.link/p0614r1)

C++17にて`if, switch`文において初期化式を指定できるようになりましたが、範囲`for`文はその対象ではありませんでした。

```cpp
auto foo() -> std::vector<int>;
void bar(int, std::size_t);

int main() {
  // 範囲forでカウンタを使用したい
  {
    std::size_t i = 0;
    for (const auto& x : foo()) {
      bar(x, i);
      ++i;
    }
  }
}
```

範囲`for`でカウンタを使用したいなど、`for`文の中でだけ追加の変数を使用したい時はその`for`の前で宣言しておく必要がありました。しかし、そうするとその変数のスコープは使用する`for`よりも広くなってしまうため、気にする場合はそれらをセットでブロックで囲う必要がありました。

C++20では範囲`for`でも初期化式を取れるようになり、`for (init; decl : expr) {...}`のような文法で`init`のところに初期化式を指定します。

```cpp
int main() {
  for (std::size_t i = 0; const auto& x : foo()) {
    bar(x, i);
    ++i;
  }
}
```

この初期化式で宣言された変数のスコープはその`for`文のスコープと一致します。

これは主に、範囲`for`で気をつけなければならない厄介なダングリング参照の問題を回避する1つの方法として導入されました。

```cpp
struct S {
  std::vector<int> vec;

  // メンバへの参照を返す
  auto get_vec() -> std::vector<int>& {
    return this->vec;
  }
};

auto f() -> S;

int main() {
  // 未定義動作
  for (const auto& x : f().get_vec()) {
    // ループ内ではf()の戻り値の寿命が尽きており
    // そのメンバvecへの参照はダングリングとなっている
    ...
  }

  // ok
  for (auto tmp = f(); const auto& x : tmp.get_vec()) {
    // tmpとしてループの間f()の戻り値が生存する
    // tmp.get_vec()の戻り値はダングリングとならない
    ...
  }
}
```

これは範囲`for`がマクロのように通常の`for`ループによる処理に展開される事から生じている罠で、`f()`の戻り値からそのメンバを参照として取り出すような場合やメソッドチェーンなどで*prvalue*が間に生成される場合にワンライナーで書こうとするとこのような未定義動作に陥ります。初期化式で一時オブジェクトを受けておく事でこの問題の影響を低減できます。

## カスタマイゼーションポイント探索の変更（DR）

- P0962R1 Relaxing the range-for loop customization point finding rules (https://wg21.link/p0962r1)

C++17までの範囲`for`では、ループ対象のオブジェクト型がメンバとして`begin/end`のどちらかを持っている場合、メンバ関数の`begin()/end()`を使用します。その時、メンバとして`begin/end`のどちらかだけしか持たない場合（すなわち、イテレータを取得するのとは異なる意味のメンバを持つ時）でも、どちらか片方が見つかってしまうとメンバ関数の`begin()/end()`を使用しようとしてエラーになっていました。

```cpp
struct X : std::stringstream {
  // some other stuff here
};

std::istream_iterator<char> begin(X& x) {
  return std::istream_iterator<char>(x);
}

std::istream_iterator<char> end(X& x) {
  return std::istream_iterator<char>();
}

int main() {
  X x;

  // P0962R1適用前はコンパイルエラー
  for (auto&& i : x) {
    // ...     
  }
}
```

`std::stringstream`は`std::ios_base`から継承した`seekdir`の`end`静的メンバ変数（ファイルストリームのシーク方向を指定するもの）を持っています。それとの衝突を避けるため、`begin()/end()`関数は名前空間スコープに非メンバ関数として定義されています。範囲`for`の探索ルールはメンバ`begin/end`の存在を確認しますがそれが関数かどうかはチェックしないため、この`X`に対してはメンバ`end`だけが見つかるのでメンバ関数の`begin()/end()`を使用しようとし、当然利用できないのでコンパイルエラーとなります。これは、標準のストリームクラスを継承したラッパ型において同じことが起こります。

このように、イテレータを提供するわけではない`begin/end`メンバ（関数ですらないかもしれない）を持つがユーザーがそれを変更できないような型がある時、ADLで発見される事を期待して名前空間スコープに`begin()/end()`を用意していても範囲`for`はそれを使用できずエラーとなってしまっていました。

この挙動は修正され、メンバとして`begin/end`の両方が見つかった場合にのみメンバ関数の`begin()/end()`を使用し、片方だけしか見つからない場合はメンバ関数の使用を諦めADLによって`begin()/end()`を探すようになります。これによって、ユーザーはより柔軟に範囲`for`にアダプトした型を書くことができるようになります。なお、これはC++11に対する欠陥報告（DR）であるため、この変更を適用済みのコンパイラではC++11モードでもこの修正は適用されます。

\clearpage

# その他

## Deprecating `volatile`

- P1152R4 Deprecating `volatile` (https://wg21.link/p1152r4)

`volatile`は理解が難しいことからその効果について誤解されがちで、正しく使用されない場合が多くあります。C++20より`volatile`の本来の効果に対して間違っている、あるいは間違いの元となる用法が非推奨とされます。削除されるわけではないのでコンパイルエラーにはなりませんが、おそらく警告が発せられるようになります。

非推奨となるコア言語の機能は以下のものです

1. `volatile`値に対する複合代入演算子（算術型・ポインタ型のみ）
    - `+= -=`などの形の演算子
2. `volatile`値に対するインクリメント／デクリメント演算子（算術型・ポインタ型のみ）
3. 間に`volatile`値がある場合の連鎖した代入演算子（非クラス型のみ）
    - `a = b = c`のような代入、`b`が`volatile`である場合に非推奨（`a, c`はどちらでもいい）
4. 関数引数のトップレベル`volatile`修飾
    - `volatile T*, volatile T&`はok
5. 関数戻り値型のトップレベル`volatile`修飾
    - `volatile T*, volatile T&`はok
6. 構造化束縛宣言の`volatile`修飾
    - `volatile auto [a, b] = f();`の形

それぞれ込み入った事情があるのでここでは深入りしません。詳細は、cpprefjpの「ほとんどの`volatile`を非推奨化」のページを参照されるといいかと思います。

重要なことは、`volatile`の効果が、`volatile`とマークされたメモリ領域に対する一度のアクセスが各バイトに対して正確に一度だけアクセスし、そのような`volatile`アクセスの順序がコード上の順序と一致することを保証するものであることです。そして、`volatile`はC++メモリモデルの一部ではないため、複数スレッドからのアクセス時にそのアクセス順について何ら保証を得ることはできません。

## ビットフィールドのデフォルトメンバ初期化

- P0683R1 Default member initializers for bit-fields (https://wg21.link/p0683r1)

通常のメンバ変数に対するデフォルトメンバ初期化はC++11から可能になっていましたが、ビットフィールドに対しては許可されていませんでした。

メンバ変数に対するデフォルトメンバ初期化の利便性をビットフィールドにももたらすべく、これができるようになります。

```cpp
struct S {
  int n1 = 10;    // ok
  int n2{10};     // ok
  int b1 : 5 = 0; // C++20からok
  int b2 : 6 {0}; // C++20からok
};
```

ただし、ビットフィールドの`:`の後の幅の指定は文法上定数式を指定可能なため任意の式が来る可能性があり、後方互換を保つためにデフォルトメンバ初期化のつもりの構文でもそうパースされない場合があります。その場合は、幅を指定する式を`()`でくくったうえでデフォルトメンバ初期化を書く必要があります。

```cpp
int a;
const int b = 0;

struct S {
  // 曖昧となる例
  int y1 : true ? 8 : a = 42;      // ok、初期化されていない
  int y2 : true ? 8 : b = 42;      // ng、b(const int)に代入できない
  int z1 : 1 || new int { 0 };     // ok、初期化されていない

  // ()でくくって曖昧さを解消する
  int y3 : (true ? 8 : b) = 42;    // ok、42で初期化
  int z2 : (1 || new int) { 0 };   // ok、0で初期化
};
```

## `using enum`

- P1099R5 Using Enum (https://wg21.link/p1099r5)

C++11で導入された`enum class`は列挙値に対する名前空間みたいなものだと思うことができます。ただし、名前空間とは異なり列挙値を参照するにはその`enum class`名を省略することができません。`switch`文など同じ`enum class`の列挙値を何度も参照する場合にはこれは煩雑です。そのため、名前空間同様に`enum class`名を`using`することで省略できるようになります。

```cpp
// スコープ付き列挙型
enum class rgba_color_channel { 
  red, green, blue, alpha
};

// C++17まで、列挙型名を省略できない
std::string_view to_string(rgba_color_channel channel) {
  switch (channel) {
    case rgba_color_channel::red:   return "red";
    case rgba_color_channel::green: return "green";
    case rgba_color_channel::blue:  return "blue";
    case rgba_color_channel::alpha: return "alpha";
  }
}

// C++20、using enum
std::string_view to_string(rgba_color_channel channel) {
  switch (channel) {
    using enum rgba_color_channel;  // using namespaceと似た働きをする

    case red:   return "red";
    case green: return "green";
    case blue:  return "blue";
    case alpha: return "alpha";
  }
}
```

これは通常の（スコープなし）列挙型に対しても使用可能です。それは、クラス（構造体）の中で入れ子定義されている列挙型をクラス外部で可視にするために使用できます。また、特定の列挙値だけを`using`することもできます（この時は`using enum`ではありません）。

```cpp
struct foo {
  enum bar {
    A, B, C
  };
};

enum class num {
  one, two, three
};

int main() {
  using enum foo::bar;

  auto en = A;  // ok

  // 特定の列挙値のusing
  using num::one;

  auto n = one; // ok
}
```

## 入れ子`inline`名前空間定義の簡易化

- P1094R2 Nested Inline Namespaces (https://wg21.link/p1094r2)

C++17より、ネストした名前空間定義を`A::B::C`のように`::`で繋げて簡略化することができます。しかし、`inline`名前空間はその対象ではなく、ネストした名前空間定義に`inline`名前空間が挟まると、その定義を分割しなければなりませんでした。

```cpp
// ほんとはこう書きたいけど
// namespace A::B::C { ... }
namespace A {
  // inline名前空間が間にあるので個別定義せざるを得ない
  inline namespace B {
    namespace C {
      int f(int n);
    }
  }
}
```

この制限は不便だったため、ネストした名前空間の簡易定義構文を拡張し、各段の名前空間名の前に`inline`キーワードを入れることでその名前空間を`inline`名前空間としてネストさせることができるようになります。

```cpp
namespace A::inline B::C {
  int f(int n);
}
```

この場合、`B`だけが`inline`名前空間となり、`A, C`は通常の名前空間となります。

## `__VA_OPT__`

- P0306R4 Comma omission and comma deletion (https://wg21.link/p0306r4)

可変引数マクロを実現する`__VA_ARGS__`は与えられた引数がゼロの場合には空で置換され、場合によっては可変引数が来ることを想定して置いておいたカンマが余ってしまうというようなことがありました。

```cpp
#define F(...) f(0, __VA_ARGS__)

F();  // f(0, );
```

この場合、プリプロセス後にコンパイルエラーとなります。これは地味に厄介な問題で、回避しようとすると複雑なことをしなければなりません、

C++20より、`__VA_OPT__(...)`を使用することでそのような問題を回避できます。

```cpp
#define F(...)           f(0 __VA_OPT__(,) __VA_ARGS__)

F(a, b, c); // f(0, a, b, c);
F();        // f(0);


#define SDEF(sname, ...) S sname __VA_OPT__(= { __VA_ARGS__ })

SDEF(foo);        // S foo;
SDEF(bar, 1, 2);  // S bar = {1, 2};
```

`__VA_OPT__(tokens...)`は可変引数マクロ（引数を受け取るマクロのうち、`...`を含むもの）の定義部分で使用でき、`__VA_ARGS__`の数（すなわち実引数の数）が0の時には空で置換され、1以上の時にかっこの中の`tokens...`へ置換されます。これによって、引数の数に応じてカンマを置く置かないなど柔軟な置換ができるようになります。

## 添字演算子内カンマの非推奨化

- P1161R3 Deprecate uses of the comma operator in subscripting expressions (https://wg21.link/p1161r3)

C++では添字演算子（`[]`）は複数の引数をとることはできませんが、カンマ演算子（`,`）オーバーロードによって擬似的にそのようなことを行えます。これは一部のテンプレートEDSLライブラリなどにおいて活用されていました（実際にはオープンソースコードベースの調査ではそのようなコードは見つからなかったようです）。

C++20からは将来の添字演算子の可変長引数化に向けて、添字演算子にカンマ区切りで複数の引数を渡す事が非推奨とされます。同じように書きたい場合、`()`でそのようなカンマ区切りの引数リストを囲う事が推奨されます。

```cpp
template<typename A, typename I>
void f(A arr, I x, I y) {
  arr[x, y];    // C++20から非推奨
  arr[(x, y)];  // ok
}
```

非推奨化といってもコンパイルエラーとなるわけではありませんが、警告は出るようになるでしょう。

これは、`std::mdspan`（多次元配列版`std::span`）で可変長`[]`オーバーロードを提供するための変更の第一歩です。

### C++23 Multidimensional subscript operator

- P2128R6 Multidimensional subscript operator (https://wg21.link/p2128r6)

この変更に引き続いて、C++23では可変長引数添字演算子のオーバーロードが許可されます。それによって、`[]`の扱いは関数呼び出し演算子`()`と全く同一になります。

```cpp
int main() {
  int buffer[2*3*4]{};
  auto s = std::mdspan<int, std::extents<2, 3, 4>>(buffer);
  s[1, 1, 1] = 42;  // ok、C++23以降
}
```

## トリビアルな型のオブジェクトを暗黙的に構築する

- P0593R6 Implicit creation of objects for low-level object manipulation (https://wg21.link/p0593r6)

C++にはオブジェクトの生存期間（*lifetime*）という概念があり、オブジェクトは作成された時にその生存期間が開始されます。生存期間外のオブジェクトの操作は未定義動作となり、それによってC言語では問題のないプログラムがC++では未定義動作となることがあります。

```cpp
struct X { int a, b; };

X* make_x() {
  X *p = (X*)malloc(sizeof(struct X));
  p->a = 1;
  p->b = 2;
  return p;
}
```

このコードはCでは問題ありませんがC++では未定義動作となります。なぜなら、`malloc`によってメモリを確保した後、その領域に`X`のオブジェクトを作成していないからです。C++では次の場合にオブジェクトはその定義に従って作成されます

- `new`式
- 共用体のアクティブメンバ切り替え時
- 一時オブジェクトが作成される時

上記プログラムはこれらのことを何もしていないため、`p`に確保されたメモリ領域には`X`のオブジェクトが作成されていません。従って、`X`の定義に従ったオブジェクト操作は未定義動作となります。

この問題は、バイト列が与えられた時にその領域をある型のオブジェクトとして参照できるのか？という問題に帰結されます。これはファイルやネットワークからデータを読みだした後で頻出するパターンでもあります。多くの場合、そこでは単にポインタの読替えが行われます。

```cpp
void process(Stream *stream) {
  unique_ptr<char[]> buffer = stream->read();

  if (buffer[0] == FOO) {
    process_foo(reinterpret_cast<Foo*>(buffer.get())); // #1
  } else {
    process_bar(reinterpret_cast<Bar*>(buffer.get())); // #2
  }
}
```

このコードもまた未定義動作を起こします。`stream->read()`では`Foo, Bar`の両方のオブジェクトが作成されることはないので、どちらかが作成されていたとしても`if`文の片方のパスは未定義動作になります。そして多くの場合、このような場合に明示的にオブジェクトを構築するコードが書かれる事は稀でしょう。

とはいえ、これらのコードはCで同等のことをした時には問題がなく、当然動作すると仮定されていることです。そのため、このような場合に暗黙的にオブジェクトを作成する事でこれらのコードが未定義動作を含まず意図通りに動作することが保証されるようになります。

とはいえそれは全ての型について行えることではなく、バイト列からオブジェクトを作成することなく復元できるような型はごく単純な型であるはずです。C++20では次のカテゴリの型がその対象となります

- スカラ型
- 集成体型
  - 任意の要素型の配列型
  - 任意メンバの集成体クラス
- トリビアルに破棄可能かつトリビアルに構築可能なクラス型

これらに該当する型は*implicit-lifetime types*と呼ばれ区別され、一部の操作において明示的にオブジェクトを作成しなくても暗黙的にオブジェクトが作成されることによってバイト配列の読替えにおいて未定義動作を起こさなくなります。*implicit-lifetime types*は次の時にそのストレージ上にオブジェクトが暗黙的に作成されます

- `char, unsigned char, std::byte`の配列が作成された時
- `malloc, calloc, realloc`及び`operator new, operator new[]`の呼び出し後
- `memcpy, memmove`
- `std::bit_cast`

これに加えて`mmmap`(POSIX)や`VirtualAlloc`(WinAPI)などの実装定義のものが含まれます。ただし、`reinterpret_cast`はいかなる場合でもそのような動作をしません（つまり2つ目の例は相変わらず未定義動作となります）。

\clearpage

# とてもマイナーな変更

## 関数テンプレートに明示的に型指定した場合にADLで見つからない問題を修正

- P0846R0 ADL and Function Templates that are not Visible  
  (https://wg21.link/p0846r0)

関数テンプレートに明示的にテンプレートパラメータを与えて、なおかつADLによって発見されることを期待して呼び出した時、コンパイルエラーとなっていました。

```cpp
namespace N {
  struct A{};

  template <typename T>
  T func(const A&) { return T(); }
}

void f() {
  N::A a;
  func<int>(a); // ng
}
```

なぜかというと、テンプレートパラメータを指定する山かっこの`<`が比較演算子として扱われてしまうためです。そのため、上記のコードは`func`という何かと`int`を比較しようとしている（`func < int`）とみなされ、`func`は無い上に型名との比較は出来ないためコンパイルエラーとなります。このエラーメッセージは分かり辛く、`N::func`のように名前空間をきちんと指定すると起きなくなる（名前空間を指定すると正しく関数名だと分かるようになる）など非常に微妙な問題です。

C++20からは、ある名前について、修飾名・非修飾名探索で何も見つからないか関数が1つ以上見つかっており、その後に`<`が続いている場合に、その名前を関数テンプレート名であるとして特別扱いし、ADLが発動するようにします。それによって、上記のコードは`func`が関数テンプレート名だと認識されるためADLによって意図通りに動作するようになります。

これによる恩恵は、`std::get`をADL経由で呼び出そうとしたときに感じられるかもしれません。

```cpp
int main() {
  std::pair<int, double> p{1, 3.14};
  auto n = get<0>(p); // C++20よりok
}
```

なお、この変更によって関数を意図的に`<`の左側の引数として関数を受け取るようにオーバーロードされた`operator<`の呼び出しがコンパイルエラーとなるようになります。しかし、そのような例はあまりにも病的であるのでこの変更によるメリットの方が大きいと判断されました。

## 要素数不明の配列型への変換  

- P0388R4 Permit conversions to arrays of unknown bound  
  (https://wg21.link/p0388r4)

C++17まで、要素数が確定している配列型から要素数が不定の配列型への変換はできませんでした。この制限は特に理由があったものではないようで、撤廃されます。

要素数不明の配列型というのは、たとえば`extern`で宣言され別の翻訳単位に実体のある配列型などで存在しえます。そして、そのような配列型（のポインタ/参照）を引数に取る関数を宣言することもでき、そのような関数の実装は要素数を仮定した処理となっているはずです。その場合にそこに要素数が確定している配列を渡せることは自然であり、プログラマがその事前条件を満たすように注意すれば特に危険もありません。そのような用途で使用される可能性があり、有用性もあるために許可されるに至ったのだと思われます。

```cpp
extern int arr[];

// 要素数不明の配列への参照を引数に取る関数
void f(int (&)[]);

int main() {
  int arr2[2] = {1, 2};

  f(arr);   // ok
  f(arr2);  // C++20からok
}
```

オーバーロード解決における順序付けでは要素数が一致する候補が最優先され、要素数不明の配列型への変換とその選択は要素数が一致する候補が無い場合に行われます。また、要素型の変換が行われる場合は要素数が一致していなくても変換が起こらない候補が最優先されます。

```cpp
void f(int (&&)[]);     // #1
void f(float (&&)[]);   // #2
void f(int (&&)[2]);    // #3

int main() {
  f({ 1 });         // #1が選択される
  f({ 1.f, 2.f });  // #2が選択される
  f({ 1, 2 });      // #3が選択される
}
```

## 特殊化のアクセスチェック

- P0692R1 Access Checking on Specializations (https://wg21.link/p0692r1)

クラス内で宣言されているローカルクラスがプライベートメンバである時、テンプレートの文脈からそれを参照する際の仕様と実装の非一貫性が長いこと存在していました。

```cpp
template<class T>
struct trait;

class C1 {
  class impl; // クラス内ローカルクラス
};

// C++標準としては許可されない(implがprivateのため)
// しかし実際にはほぼ全てのコンパイラでエラーにならない
template<>
struct trait<C1::impl>;
```

プライベートローカルクラス型によるテンプレートの特殊化は、そのクラス外からは許可されません。しかし実際には、多くのコンパイラがそれを許可していました。ここで、このローカルクラス（`impl`）をローカルクラステンプレートにすると、コンパイラ間でも挙動に差異が生じます。

```cpp
template<class T>
struct trait;

class C2 {
  template<class U>
  struct impl;
};

// C++標準としては許可されない(implがprivateのため)
// ng : clang, icc
// ok : gcc, msvc
template<class U>
struct trait<C2::impl<U>>;
```

プライベートローカルクラステンプレートを用いて特殊化した上でクラス外から参照しようとすると、clangやiccでは許可されないのに対して、gcc,msvcでは許可されます。

C++20では、標準でプライベートローカルクラス（テンプレート）のテンプレートの文脈からの参照が許可され、標準と実装の差異及び実装間の差異がプログラマにとって望ましい形で解消されます。それによって、上記2種類のサンプルはどちらも標準仕様に則った正しいコードとなります。

```cpp
// C++20より、どちらもok

template<>
struct trait<C1::impl>;

template<class U>
struct trait<C2::impl<U>>;
```

これらのことは標準のいくつかの提案を実装しようとした時に問題となり、C++20での例として`<ranges>`の各種`view`のイテレータ型があります。`view`のイテレータ型はほとんどが`view`クラスのプライベートローカルクラステンプレートとして定義されており、この変更がなければそれらの型に対して`std::iterator_traits`をはじめとする`trait`テンプレートを適用する事が合法的かつポータブルに行えませんでした。

## `default`コピーコンストラクタの`const`ミスマッチを`delete`するようにする

- P0641R2 Resolving Core Issue #1331 (const mismatch with defaulted copy constructor) (https://wg21.link/P0641R2)

コピーコンストラクタは通常クラス`C`に対して`const C&`の引数を持ちますが、実のところ意外と柔軟で`const`が無くても良かったりします。すると、他の型のオブジェクトをメンバとして持つクラス型では、それら及び自身のコピーコンストラクタの間で受け取る引数型のミスマッチが発生することになります。

```cpp
struct MyType {
  MyType() = default;
  MyType(MyType&);  // constがない
};

template <typename T>
struct Wrapper {
  Wrapper(const Wrapper&) = default;  // constあり

  T t;
};

Wrapper<MyType> var;  // ng
```

この場合、`Wrapeer<MyType>`のコピーコンストラクタ引数と`MyType`のコピーコンストラクタ引数とで`const`ミスマッチが発生しており、それによって`Wrapeer<MyType>`の`default`コピーコンストラクタが定義できなくなっています。コピーコンストラクタを使用していなくてもこれはエラーとなってしまいます。

なぜこうなるかというと、あるクラス`T`の`default`コピーコンストラクタはそれを明示的に宣言しないときに暗黙に定義されているコピーコンストラクタと同じ引数を持つ必要があり、メンバ型のコピーコンストラクタ引数が`const`無しである場合それに引きずられて`T`の暗黙のコピーコンストラクタは`T(T&);`の様に宣言されるためです。

`Wrapeer<MyType>`ではまさにこれが起きており、`Wrapper(Wrapper&) = default;`と宣言するとコンパイルエラーは起こらなくなります。

C++20では、このような場合にコピーコンストラクタは暗黙`delete`されるようになります。従って、コピーコンストラクタを使用しようとするまでエラーは出なくなります。これはコピー代入演算子等他の特殊メンバ関数にも適用されますが、代入演算子が非参照パラメータを持つ場合や戻り値型がおかしい場合などは引き続きその場でエラーとなります。

この問題が起こりうるラッパー型には`std::tuple`や`std::pair`があり、それらの使用時に非常にまれに恩恵を感じられるかもしれません。とはいえ、コピーコンストラクタは`const T&`を取るようにすべきでしょう。

## `const`修飾されたメンバポインタの制限を修正

- P0704R1 Fixing const-qualified pointers to members (https://wg21.link/P0704R1)

メンバ関数に参照修飾を行うと、そのオブジェクトの値カテゴリに応じて関数を呼び分けることができるようになります。これはC++11から可能になったのですが、メンバポインタまでその考慮が行き届いておらず、メンバポインタを介した呼び出しが意図通りにならない事がありました。

```cpp
struct X {
  void foo() const&;
};

X{}.foo();        // ok
(X{}.*&X::foo)(); // ng
```

この2つのコードは本質的に同じことをしているはずです。右辺値（*prvalue*）は`const`左辺値参照で束縛できるため、右辺値オブジェクトから`const &`修飾されたメンバ関数が呼び出せるのは自然な事です。しかし、`.*`演算子を使用したメンバポインタ呼び出しの仕様では、「`.*`を用いて右辺値オブジェクトに対してメンバ関数ポインタ呼び出しを行う場合、そのメンバ関数が`&`修飾されている場合呼び出しできない」のように指定されており、`const &`が考慮されていませんでした。

この変更によってそれが修正され、上記のコードはどちらも呼び出し可能となります。

これは、`std::invoke`の使用時に恩恵を感じることがあるかもしれません。`std::invoke`は統一的な関数呼び出しを行うものですが、その呼び出し方法の1つにメンバポインタからの呼び出しがあります。

## destroying operator delete

- P0722R3 Efficient sized delete for variable sized classes (https://wg21.link/p0722r3)

`delete`式は指定されたポインタの参照先にあるオブジェクトを破棄（デストラクタ呼び出し）してから`operator delete`によってそのメモリ領域を解放します。あるクラスについて`delete`演算子をオーバーロードしている場合、そのユーザー定義`delete`が呼ばれた時には対応するオブジェクトは破棄済みであり、クラスのメンバに安全にアクセスすることはできません。

```cpp
struct S {
  std::string str;

  void operator delete(void* p) {   // #1
    // 未定義動作！
    const S& self = *static_cast<S*>(p);
    std::cout << self->str;

    ::operator delete(p);
  }
};

int main() {
  S* p = new S{};

  // Sのデストラクタ呼び出しの後#1が呼び出される
  delete p;
}
```

このような場合にオーバーロード`delete`からクラスメンバへアクセスしたいユースケースがあり、それを許可するために`operator delete`に新しい形式が追加されます。追加される形式は、2つ目の引数に`std::destroying_delete_t`を取るもので、その形式でオーバーロードしておくと`delete`式はデストラクタ呼び出しを省略した上でユーザー定義`operator delete`を呼び出すようになります。

```cpp
struct S {
  std::string str;

  void operator delete(void* p, std::destroying_delete_t) {  // #1
    // ok、安全に参照できる
    // まだデストラクタは呼ばれていない
    const S& self = *static_cast<S*>(p);
    std::cout << self.str;

    // デストラクタ呼び出しをしなければならない
    std::destroy(self);
    ::operator delete(p);
  }
};

int main() {
  S* p = new S{};

  // Sのデストラクタは呼ばれずに#1が呼び出される
  delete p;
}
```

`std::destroying_delete_t`はタグ型でありその値に意味はありません。この形式のオーバーロードのことを*destroying operator delete*と呼びます。

*destroying operator delete*を呼び出すために特殊なことをする必要はなく`delete`式がそれを判断してくれます。クラス（ここでは`S`）が仮想デストラクタを持たない場合は`delete`式の効果は単純にデストラクタ呼び出しをスキップして*destroying operator delete*を呼び出します。クラスが仮想デストラクタを持つ場合、`delete`式に指定されたポインタの動的型（実際の最派生オブジェクト型）に定義された*destroying operator delete*を呼び出します（これは通常の仮想関数呼び出しと同じことをします）。なお、`delete`式が*destroying operator delete*を考慮するかどうかはコンパイル時に判定することができるため、これによってそのほかの`delete`式にオーバーヘッドが追加される事はありません。

*destroying operator delete*は例えば、`operator new`とともにオーバーロードする事であるクラスのヒープへの構築・破棄の状況をログとして出力したり、可変サイズオブジェクトの安全な実装などに活用できます。また、次のようなディスパッチを行うこともできます。

```cpp
struct base {
  // destroying operator delete宣言 #1
  void operator delete(base* ptr, std::destroying_delete_t);
};

struct derived : base {
  int n;
};

// #1に対応する定義
void base::operator delete(base* ptr, std::destroying_delete_t) {
  derived* pd = static_cast<derived*>(ptr);
  // derivedのデストラクタを呼び出し、その領域を解放する
  std::destroy(*pd);
  ::operator delete(pd);
}

int main() {
  base* p = new derived{};
  // baseには仮想デストラクタが無いが、正しく破棄と解放が行われる
  delete p;
}
```

\clearpage

# Defect Report

## `new`式における配列要素数の推論

- Array size deduction in new-expressions (https://wg21.link/p1009r2)

配列の要素数について、動的ではない（`new`によらない）配列を`{}`によって初期化する際にはその要素数が初期化子の数から推定されます。しかし、同じことを`new`によって確保される配列に対して行うと要素数が推定できずにコンパイルエラーとなっていました。

```cpp
int a[] = {1, 2, 3};          // ok、要素数3
int* p = new int[]{1, 2, 3};  // ng、要素数が推定できない
```

配列初期化時の要素数推定はCのころからあったものですが、`new`式の仕様は少し複雑でそことは分かたれており、その考慮が`new`式にまで及んでいなかったようです。

これは単に見落とされていたもので、何か障害があったわけではありませんでした。多くのC++プログラマにとって`new`式における初期化は通常の変数の初期化と同じルールに従うことが自然でありこの非一貫性は直感的ではないため、修正することで初期化に関するルールがシンプルかつ教えやすくなります。とはいえ、`new`式で配列を書こうとすることは稀ではあるでしょう。

この提案は以前の全てのバージョンに対するDRであり、`clang`は早期にこれを実装していたようです。

## ポインタ型から`bool`への変換を縮小変換とする

- P1957R2 Converting from `T*` to `bool` should be considered narrowing (re: US 212) (https://wg21.link/p1957r2)

C++17で導入された`std::variant`には当初、ポインタから`bool`への暗黙変換によって意図しない構築がなされるバグがありました。

```cpp
std::variant<std::string, bool> x = "abc";  // boolを保持する
```

その後暗黙の`bool`変換を検出して弾くようにしたこと（P0608R3）でこのバグは解消されたのですが、今度は`bool`に変換可能なユーザー定義型が`bool`を選択して構築出来ないバグが導入されました。

```cpp
std::bitset<4> b("0101");
std::variant<bool, int> v = b[1]; // intを保持する
```

P0608R3による最初の修正時には他の問題の解決も含めていたため、代入時に縮小変換が起きる場合は構築できないようにされました。しかし、`bool`への暗黙変換の一部（特に`char* -> bool`への変換）を縮小変換として扱う事はライブラリレベルでは不可能だったため、`bool`への変換は全て禁止されていました。これによって2つ目のバグが導入されたわけです。

この解決のために、ポインタ型から`bool`への変換を縮小変換であるとコア言語で規定したうえで、`std::variant`の代入演算子の制約から`bool`への暗黙変換禁止を取り除きます。これによって、上記2つのコードはどちらも意図通りになることになります。

```cpp
std::variant<std::string, bool> x = "abc";  // std::stringを保持する

std::bitset<4> b("0101");
std::variant<bool, int> v = b[1]; // boolを保持する
```

この影響は`std::variant`だけではなく言語全体に波及し、ともすれば後方互換性を破壊する可能性があります。

```cpp
struct A {
  bool b;
};

A f(int* p) {
  return {p}; // ng、{}初期化では縮小変換は許可されない
}
```

この様な変換は多くの場合にバグである可能性が高いことから、コードの破損よりもそのような変換をコンパイル時に検出できるメリットの方が大きいと判断されたようです。なお、これはC++17へのDRです。

## 暗黙のムーブ対象の拡大

- P1825R0 Merged wording for P0527R1 and P1155R3 (https://wg21.link/P1825R0)
- P0527R1 Implicitly move from rvalue references in return statements (https://wg21.link/P0527R1)
- P1155R3 More implicit moves (https://wg21.link/P1155R3)

暗黙のムーブとは、ある関数から値をコピーして返す場合に暗黙的にムーブを行うことでコピーを回避する最適化の事です。C++17で規定された値のコピー省略保証（RVO）と似ていますが異なるもので、RVOは`return`されるオブジェクトを呼び出し側で直接構築することでコピーやムーブを完全に省くものです。

(N)RVO及び暗黙のムーブと呼ばれる最適化はC++11から許可されました。それが起こると関数からの`return`に際するコピーが省略され、コピーが一切できないような型のオブジェクトを関数から値で返すことができます。

```cpp
struct Widget {
  Widget(Widget&&);
};

Widget one(Widget w) {
  return w;  // 暗黙ムーブ、C++11から
}

struct RRefTaker {
  RRefTaker(Widget&&);
};

RRefTaker two(Widget w) {
  return w;  // 暗黙ムーブされて構築、C++11(CWG1579)
}
```

これらはC++11の時点で許可されていた暗黙のムーブによる最適化の一例です。関数ローカルの変数（値で宣言された関数引数）を自動でムーブして戻り値を返すことができます。

C++17までは、暗黙のムーブ対象は関数ローカルのオブジェクト（参照やポインタでない）だけでした。C++20からは、右辺値参照でバインドされたオブジェクトが暗黙のムーブ対象になるようになります。

```cpp
RRefTaker three(Widget&& w) {
  return w;  // 暗黙ムーブ、C++20(P0527)
}
```

さらに、暗黙のムーブが起こるコンテキストが拡張され、`throw`式やコンストラクタによらない暗黙変換が起こる場合にも暗黙のムーブが行われるようになります。

```cpp
void four(Widget w) {
  throw w;  // 暗黙ムーブ、C++20(P1155)
}

struct From {
  From(Widget const &);
  From(Widget&&);
};

struct To {
  operator Widget() const &;
  operator Widget() &&;
};

From five() {
  Widget w;
  return w;  // 暗黙ムーブ（コンストラクタによる変換）、C++11
}

Widget six() {
  To t;
  return t;  // 暗黙ムーブ（変換演算子による変換）、C++20(P1155)
}

struct Fowl {
  Fowl(Widget); // 値で受け取るコンストラクタ
};

Fowl seven() {
  Widget w;
  return w;  // 暗黙ムーブ、C++20(P1155)
}

// DerivedはBaseを公開継承しているとき
Base eight() {
  Derived result;
  return result;  // 暗黙ムーブ（基底クラスへの変換）、C++20(P1155)
}
```

暗黙のムーブが可能なローカル変数を`return`する時、その戻り値型を構築するためのオーバーロード解決において、対象変数を右辺値（*xvalue*）としてオーバーロード解決を行い、それによって選択されたコンストラクタ（変換関数）を選択することによって暗黙のムーブが実行されるようになります。

ただし、これら暗黙のムーブは値のコピー省略保証とは異なり必ず実行されるものではなく、あくまでコピー省略による最適化を明示的に許可するものであり、実装によっては行われない可能性があります。

## 異なる例外指定がなされた`default`メンバ関数の許可

- P1286R2 Contra CWG DR1778 (https://wg21.link/P1286R2)

`std::atomic<T>`はデフォルトコンストラクタを持ち、それは常に`noexcept`指定されています。

```cpp
template<typename T>
struct atomic {
  atomic() noexcept = default;

  // ...
};
```

このため、`T`のデフォルトコンストラクタが例外を投げうる（`noexcept`指定が無い）場合に要素型のデフォルトコンストラクタの`noexcept`指定とのミスマッチが発生し、デフォルトコンストラクタを使用するとコンパイルエラーとなっていました。

```cpp
struct S {
  S() noexcept(false) {}
};

int main() {
  std::atomic<S> s{}; // ng
}
```

これは、`default`宣言する特殊関数の例外指定はその宣言がないときに暗黙に定義される特殊関数のものと同じにならないといけないというルールによります（少し前で、コピーコンストラクタについても同じ制限がありました）。

ユーザーが明示的に指定した例外指定はそれが意図する例外指定なのでそれを受け入れるべきであり、この制限は取り除かれることになりました。これによって上記コードはコンパイルエラーとならなくなります。異なっている例外指定については上書きしているもの（`std::atomic`のもの）が優先され、`noexcept`指定されている特殊関数内でされていない関数が例外を投げた場合、プログラムは`std::terminate`によって停止します。

また、この問題及び解決はデフォルトコンストラクタのみではなく`default`指定可能な特殊メンバ関数全てに共通することです。

これはDRですが特にバージョン指定されておらず、この問題の原因となる変更が入ったのがC++14からなので、おそらくC++14に向けてのDRです。

## 不完全型を用いた宣言の抽象クラスのチェックを遅延する

- P0929R2 Checking for abstract class types (https://wg21.link/p0929r2)

抽象クラス（*abstract class*）とは、メンバ関数に純粋仮想関数が一つでもあるようなクラス型の事です。

```cpp
class abst {
  // メンバ変数があってもいい
  int m;

public:

  // コンストラクタがあってもいい
  abst(int n) : m(n) {}

  virtual ~abst() = default;

  // 1つでも純粋仮想関数があれば抽象クラス
  virtual void f() = 0;
};
```

抽象クラスは未定義の純粋仮想関数を含んでいるため、オブジェクトを作成することができません。抽象クラスのオブジェクトは、その派生クラスのオブジェクトの一部としてしか存在できず、抽象クラス型はポインタとして使用することになるでしょう。

しかし、不完全型を使用可能な他の文脈では不完全型として抽象クラス型が使用される可能性があり、そのように使用された場合に後から定義を追加すると、以前はエラーの出なかった宣言が遅れてコンパイルエラーを引きおこしていました。

```cpp
struct S;

S f();   // #1、関数宣言における不完全型の使用、ok

// 数百行の後・・・

// 抽象クラスの定義、#1を遡ってエラーにする
struct S { virtual void f() = 0; };
```

このほかにも、配列の宣言などにおいても同様の事が発生します

この問題の解決のため、関数や配列などの宣言の時点では抽象クラスが使用されてもそこではその抽象性をチェックせず、その関数や配列が実際に定義されたとき（あるいは関数が呼び出されたとき）に初めてチェックされコンパイルエラーが発生するようになります。

これによる恩恵はおそらく、テンプレートメタプログラミングの文脈でテンプレート実引数として抽象型が使用されたときに不要なエラーが起こらなくなるという形で得られるでしょう。

\clearpage

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- cppmap(https://cppmap.github.io/ : ライセンスはCC0 パブリックドメイン)
- yohhoyの日記(https://yohhoy.hatenadiary.jp/)
- C++20のコルーチン for アプリケーション - Qiita  
  (https://qiita.com/Fuyutsubaki/items/a4c9921587ce53d95e55)
- C++コルーチン拡張メモ - Qiita  
  (https://qiita.com/yohhoy/items/aeb3c01d02d0f640c067)
