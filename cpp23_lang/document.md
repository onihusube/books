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

文中では、基本的には`std::`を省略していませんが、文脈上明らかな場合に一部で`std::`を省略することがあります。また、通常の関数の表記（`func()`）に対してメンバ関数の表記（`.member_func()`）は先頭に`.`を付加して区別します。

## 提案文書

C++標準への機能の追加は提案文書を介して行われます。提案文書はC++標準化委員会のメンバによって提出され、それを3段階くらいのレビューステージを通して吟味し、最終的にC++標準化委員会全体の会議で投票にかけられて次のC++バージョンの機能として採用されます。

提案文書はほぼ全て公開されており、英語ではありますが誰でも読むことができます。ただし、C++11以前の古い時代のものは公開されていないものもあります。現在はほぼ1ヶ月に一度、毎月20日前後に最新の提案文書が公開されています。

提案文書は`PxxxxRn`のような形式の番号によって管理されています（例えば、P0734R0とかP0515R3など）。`Pxxxx`の部分は提案そのものの個別番号で、`Rn`の数字`n`はその提案のリビジョン、すなわち更新回数を表します。提案はレビューとフィードバックのループを繰り返しながらブラッシュアップされ、その都度その変更・改善を反映した新しい版の文書が公開されます。その際はPから始まる番号はそのままで、R以降の数字がインクリメントされます（たまにRは2桁に達することがあります）。

提案文書内およびC++コミュニティにおいて特定の提案文書を指すときには、タイトルの代わりにこの番号が使用されるのが一般的です。現在は使用されていませんが、C++11前後くらいの時期の古い提案（およびC標準化委員会）ではPではなくN始まりの数字が使用されていることがあり、その場合はリビジョン毎に番号が変わります。

本書では各機能紹介の冒頭で直接の提案文書および関連性の高いものをリストで示してあります。紙で読まれている方は申し訳ないですが、おそらくP番号だけで検索をかけても文書にたどりつけると思います（出てこない場合はC++を追加すると確実）。

## 欠陥報告（DR）

欠陥報告（*Defect Report* : DR）とは、仕様に対する欠陥（バグ）の報告に伴う解決のための提案であり、その変更は過去のバージョンに遡って適用されます。

DRとされた問題については一部のコンパイラは早期に実装している可能性があり、実装済みのコンパイラでは古いバージョンの指定時（C++23対応コンパイラに`-std=c++11`を指定するなど）にも変更が適用されてコンパイルされるようになります。ただし、DRがどのバージョンまで遡って適用されるかは指定されていない場合もあり、その場合はその仕様が最初に導入されたバージョンまで遡るようです。

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

- P0847R7 Deducing this(https://wg21.link/P0847R7）

`std::optional`の`value()`関数のように、`this`の状態（`const`/参照修飾）によって適切にメンバ変数を転送する必要がある関数においては、`const`有無と参照修飾で4つのオーバーロードを提供しなければなりません。例えば次のようになり、基本的に楽をする良い方法がありません。

```{style=cppstddecl}
template <typename T>
class optional {
  ...
  
  constexpr T& value() & {
    if (has_value()) {
      return this->m_value;
    }
    throw bad_optional_access();
  }

  constexpr T const& value() const& {
    if (has_value()) {
      return this->m_value;
    }
    throw bad_optional_access();
  }

  constexpr T&& value() && {
    if (has_value()) {
      return move(this->m_value);
    }
    throw bad_optional_access();
  }

  constexpr T const&& value() const&& {
    if (has_value()) {
      return move(this->m_value);
    }
    throw bad_optional_access();
  }
  
  ...
};
```

このようなボイラープレートコードは些細なバグを誘発しやすく、コードの保守性も低下します。一方、これがもしメンバ関数ではなかったとしたら、次のように簡潔な実装を選択できます

```{style=cppstddecl}
template <typename T>
class optional {
  ...
  
  // hidden friendで実装したとすると
  template <typename Opt>
  friend decltype(auto) value(Opt&& o) {
      if (o.has_value()) {
          return forward<Opt>(o).m_value;
      }
      throw bad_optional_access();
  }
  
  ...
};
```

この1つの関数テンプレートでさきほどの4つのメンバ関数と全く同じ動作をさせることができます。ただ、これはメンバ関数ではないので`opt.value()`のように呼び出すことは出来ず、ADLを通して`value(opt)`のように呼ばなければなりません。

C++23ではこのような問題を解決するために、メンバ関数において暗黙の`this`パラメータを明示的に受け取ることのできる構文が導入されます。構文は、非静的メンバ関数の第一引数に`this`による注釈を行うことで`this`パラメータを明示的に取るようにします。

```cpp
struct X {
  // void foo(int i) const & 相当の宣言
  void foo(this const X& self, int i);
};
```

このような関数宣言の事を明示的オブジェクト引数宣言（*explicit object parameter declaration*）と呼び、明示的に取られている`this`引数の事を明示的オブジェクト引数（*explicit object parameter*）と呼びます。

明示的オブジェクト引数宣言によって宣言された関数は通常のメンバ関数と同じように呼び出すことができます。

```cpp
struct D : X { };

void ex(X& x, const D& d) {
  x.foo(42);  // selfにはxがconst X&で渡される

  // アクセスできれば派生クラスからも呼び出し可能
  d.foo(17);  // selfにはdがconst X&で渡される（基底クラスへの変換が行われる）
}
```

明示的オブジェクト引数はテンプレートパラメータとして受け取るようにすることもできます。この場合、呼び出し時に呼び出しに使用したオブジェクトからその型が直接的に推論されます。

```cpp
struct X {
  // thisをテンプレートで受け取る
  template <typename Self>
  void bar(this Self&& self);
};

struct D : X { };

void ex(X& x, const D& d) {
  x.bar();        // SelfはX&に推論され、X::bar<X&>が呼ばれる
  move(x).bar();  // SelfはXに推論され、X::bar<X>が呼ばれる

  d.bar();        // Selfはconst D&に推論され、X::bar<const D&>が呼ばれる

  X& br = d;
  X* bp = &d;
  br.bar();       // SelfはX&に推論され、X::bar<X&>が呼ばれる
  bp->bar();      // SelfはX&に推論され、X::bar<X&>が呼ばれる
}
```

明示的オブジェクト引数がテンプレートパラメータで指定されている場合の呼び出しにおいては、呼び出しに使用したオブジェクトの型（と値カテゴリ）が通常の関数テンプレートと同様に推論されます。これにより、派生クラスから基底クラスの明示的オブジェクト引数を取る関数を呼んだ際は基底クラスへの変換が行われず派生クラス型が直接推論され、逆に派生クラスを基底クラスのポインタ（参照）に入れた状態で呼び出すと派生クラスではなく基底クラス型が推論されます。

これらにより、C++23では最初に例に挙げた`std::optional<T>::value()`の実装は次のように改善されます

```cpp
template <typename T>
class optional {
  ...
  
  template <typename Self>
  constexpr auto&& value(this Self&& self) {
    if (!self.has_value()) {
      throw bad_optional_access();
    }

    return forward<Self>(self).m_value;
  }
  
  ...
};
```

このように、`this`を明示的かつテンプレートで受け取ることで`this`オブジェクトのCVと値カテゴリに応じてメンバを転送するようなコードをかなりと簡潔に書くことができるようになります。

### 値での受け取り

明示的オブジェクト引数は値（参照修飾なし）で受け取ることもできます

```cpp
struct less_than {
  template <typename T, typename U>
  bool operator()(this less_than, T const& lhs, U const& rhs) {
    return lhs < rhs;
  }
};

less_than{}(4, 5);  // ok
```

この場合、`less_than`のオブジェクト（prvalue）はコピー省略によって呼び出し演算子の引数で直接構築されます。`less_than`型の左辺値から呼び出す場合は、`this`引数がコピーされて渡されます。これは状態を持たない関数オブジェクトの暗黙`this`引数の不要なコピーを回避するのに使用できます。

オブジェクトサイズが小さく参照渡しよりも値渡しの方がコストが低いようなクラス型においても同様に、`this`を値で受け取るようにすることでメンバ関数呼び出しのコストを削減することができます。

```cpp
// string_viewのようなクラス型
template <class charT, class traits = char_traits<charT>>
class basic_string_view {
private:
  // オブジェクトサイズは通常8byte
  const_pointer data_;
  size_type size_;
public:
  constexpr const_iterator begin(this basic_string_view self) {
    return self.data_;
  }

  constexpr const_iterator end(this basic_string_view self) {
    return self.data_ + self.size_;
  }

  constexpr size_t size(this basic_string_view self) {
    return self.size_;
  }

  constexpr const_reference operator[](this basic_string_view self, size_type pos) {
    return self.data_[pos];
  }
};
```

ただし、状態を持つクラス型のオブジェクトを値で受け取るとそのメンバ関数内で`this`を変更してもメンバ関数終了後に変更は消えてしまうので注意しましょう。

この動作はまた、メンバ関数を呼び出すとその時点の`this`のコピーを返すような操作に対しても有効です。例えば、コンテナに`.sorted()`メンバ関数（ソート済みのコンテナを返す）を実装する場合などです。

```cpp
struct my_vector : std::vector<int> {
  // thisをコピーしてからソートして返す
  auto sorted(this my_vector self) -> my_vector {
    sort(self.begin(), self.end());
    return self;
  }
};
```

呼び出すオブジェクトが左辺値だと（暗黙に）コピーされますが、右辺値の場合は自動でムーブされて渡されます。

### 明示的オブジェクト引数の変換

明示的オブジェクト引数は`this`引数を明示的に取るものなので、その受ける型名は常識的にはそのクラス型となります。実際そこに渡されるのは呼び出しに使用したオブジェクトそのものではあるのですが、オーバーロード解決においてそこから引数型への変換が考慮されるため、実は自身のクラス型以外の型で受けることができます。

```cpp
struct S {
  int i = 0;

  operator int() const {
    return i;
  }

  int f(this int n) {
    return n;
  }
};

int main() {
  S s{20};
  int n = s.f();  // ok、20
}
```

明示的オブジェクト引数の変換においては暗黙変換が行われるため、このように変換して呼び出しさえできれば明示的オブジェクト引数を所属するクラス型と異なる型にすることができます。

```cpp
struct C {
  int n = -10;

  // 万能変換コンストラクタ
  template <typename T>
  C(T) {};
};

struct B {
  int n = 10;

  // 派生クラスから呼ばれても基底クラスとして受ける
  int f(this const B& self) {
    return self.n;
  }

  // 常にCに変換して受ける
  int g(this C c) {
    return c.n;
  }
};

struct D : B {
  int n = 20;
};

int main() {
  D d{};

  // 派生クラスから基底クラスへの変換が行われる
  int n = d.f(); // ok、10

  // Cの変換コンストラクタによって変換される
  int m = d.g();  // ok、-10
}
```

### 細かい仕様の話

明示的オブジェクト引数（宣言）に対して従来の`this`引数を明示しない関数宣言及び`this`引数の事を暗黙的オブジェクト引数（implicit object parameter）（宣言）と呼びます。

同じ関数名に対して明示的オブジェクト引数宣言と暗黙的オブジェクト引数宣言は混在させることができますが、オーバーロードとして成立している必要があります（同じ意味の宣言がそれぞれの形で宣言されてはならない）。

```cpp
struct X {
  // ng、2つの形式の宣言が衝突している
  void bar() &&;
  void bar(this X&&);
};

struct Y {
  // ok、bar()のオーバーロードは衝突していない
  void bar() &;
  void bar() const&;
  void bar(this Y&&);
};
```

また、`obj.f()`のように呼び出した際に明示的オブジェクト引数宣言と静的メンバ関数宣言が衝突するような宣言（明示的オブジェクト引数のみを取るメンバ関数と引数無しの静的メンバ関数の宣言）も行うことができません。

```cpp
struct X {
  // ng、x.f()の呼び出しが衝突する
  static void f();
  void f(this X const&);

  // ok、呼び出しが衝突しない
  static void g();
  void g(this X const&, int);
};
```

明示的オブジェクト引数宣言の関数は、`static, virtual`で宣言できず、メンバ関数のCV/参照修飾も行えません。

```cpp
struct X {
  // 全てngな宣言例

  static void f1(this X&);

  virtual void f2(this X&);
  
  void f3(this X&) const &;

  void f4(this X&) volatile;
};
```

そして、明示的オブジェクト引数宣言の関数内では`this`を使用できません。メンバアクセスは常に明示的オブジェクト引数を介して行う必要があります。

```cpp
struct X {
  int i = 0;

  void mem_f();

  void f(this X& self) {
    // メンバiを参照したい
    i;        // ng、非静的メンバiを直接参照できない
    this->i;  // ng、ここではthisを使用できない
    self.i;   // ok、非静的メンバ変数iにアクセス


    // メンバ関数mem_f()を呼び出したい
    mem_f();        // ng
    this->mem_f();  // ng
    self.mem_f();   // ok
  }
};
```

これによって、テンプレートパラメータで明示的オブジェクト引数を受けている時にそのメンバ関数を派生クラスからの呼び出した場合、その関数内からどの名前が参照されるかが明確になります。すなわち、渡されたオブジェクトの型でアクセス可能なものにアクセスします。

```cpp
struct X {
  int i = 1;

  template<typename Self>
  int f(this Self& self) {
    return self.i;  // Self::iが参照される
  }
};

struct D : X {
  int i = 10;
};

int main() {
  D d{};
  X& x = d;

  int n1 = d.f(); // 10（D::i
  int n2 = x.f(); // 1 （X::i
}
```

このように、明示的オブジェクト引数宣言の関数はほとんど静的メンバ関数のように動作し、その動作は静的メンバ関数として捉えることができます。とはいえ一応規格的な扱いとしてはまだ非静的メンバ関数です。

例えば、関数ポインタを取った時の動作も非静的メンバ関数とは異なります。次のようなクラスとメンバ変数があるとき

```cpp
struct Y {
  int f(int, int) const&;
  int g(this Y const&, int, int);
};
```

`Y::f`の（メンバ）関数ポインタ`&Y::f`の型は`int(Y::*)(int, int) const&`となりますが、`Y::g`の関数ポインタ`&Y::g`は`int(*)(Y const&, int, int)`になり、単なる関数ポインタになります。そのため、関数ポインタから使用する時に少し使用法が異なります。

```cpp
int main() {
  Y y;
  y.f(1, 2); // ok
  y.g(3, 4); // ok

  auto pf = &Y::f;
  pf(y, 1, 2);              // ng、メンバポインタはこの記法で呼び出せない
  (y.*pf)(1, 2);            // ok、メンバ関数ポインタ呼び出し
  std::invoke(pf, y, 1, 2); // ok

  auto pg = &Y::g;
  pg(y, 3, 4);              // ok、関数ポインタ呼び出し
  (y.*pg)(3, 4);            // ng、pgはメンバ関数ポインタではない
  std::invoke(pg, y, 3, 4); // ok
}
```

とはいえ一方で、静的メンバ関数とも異なる部分があります。

```cpp
struct C {
  void nonstatic_fun();
  
  void explicit_fun(this C c) {
    static_fun(C{});    // ok
    (+static_fun)(C{}); // ok
  }
  
  static void static_fun(C) {
    // 関数の名前の使用方法に関して

    explicit_fun();        // ng、オブジェクト引数が必要
    explicit_fun(C{});     // ng、この形式で呼び出せない

    auto f = explicit_fun; // ng、名前はポインタに変換できない
    (+explicit_fun)(C{});  // ng、同上
    
    C{}.explicit_fun();        // ok
    auto p = explicit_fun;     // ng、名前はポインタに変換できない
    auto q = &explicit_fun;    // ng、クラス名が必要
    auto r = &C::explicit_fun; // ok
    r(C{});                    // ok
  }
  
  // 演算子オーバーロードの可否
  static C operator~();  // ng
  C operator~(this C);   // ok
};

// 関数ポインタの取り方
C c;
int (*a)(C) = &C::explicit_fun; // ok
int (*b)(C) = C::explicit_fun;  // ng、名前はポインタに変換できない

auto x = c.static_fun;     // ok
auto y = c.explicit_fun;   // ng、名前はポインタに変換できない
auto z = c.explicit_fun(); // ok、関数呼び出し
```

通常の非静的メンバ関数（メンバ関数）と明示的オブジェクト引数宣言の関数（明示的`this`）と静的メンバ関数の異なる部分をまとめたのが次の表です

|特性|メンバ関数|明示的`this`|静的メンバ関数|
|---|---|---|---|
|オブジェクト引数が必要/使用可能|Yes|Yes|No|
|暗黙の`this`引数|あり|なし|なし|
|演算子の宣言|可能|可能|一部可能|
|`&C::f`の型|メンバポインタ|関数ポインタ|関数ポインタ|
|`auto f = x.f;`|エラー|エラー|関数ポインタ取得|
|名前を関数ポインタに減衰|不可|不可|可能|
|仮想関数の宣言|可能|不可|不可|

このように、明示的オブジェクト引数宣言の関数は、従来の非静的メンバ関数とも静的メンバ関数とも異なる部分があり、どちらかというと半静的メンバ関数といった趣です。それでもなお一応の扱いは非静的メンバ関数です。

### 名前探索とオーバーロード解決

名前探索に関して、C++17までは`obj.foo()`のように関数呼び出しをした際に探索される候補は、`obj`のクラス型のスコープで宣言された非静的メンバ関数と静的メンバ関数で`foo`という名前を持つ関数です。そして、非静的メンバ関数の場合は第一引数に暗黙のオブジェクト引数があるかのように扱われ、そのCV/参照修飾は関数の修飾によって決定されます。

C++23でもこの探索の基本は変わらず、`foo`という名前の非静的メンバ関数として明示的オブジェクト引数宣言の関数も候補に上がるようになります。

ただし、追加の引数がある場合は明示的オブジェクト引数宣言の関数のオーバーロード解決において引数リストが右にシフトされ、最初の引数に`this`オブジェクトがあてがわれてオーバーロード解決が行われます。また、第一引数の`this`オブジェクト引数のCV/参照修飾はメンバ関数の修飾から決まるのではなく、宣言されたCV/参照修飾によって決定されます。

オーバーロード解決のフェーズもほとんど変更されていませんが、`this`オブジェクトに相当する引数をテンプレートで取ることができるようにされている他、`this`オブジェクトに相当する引数の変換が考慮されるようになっています。

したがって、名前探索とオーバーロード解決に関しては従来のメンバ関数呼び出しと同様に考えることができ、テンプレートの場合も一般の関数テンプレートと同じ推論によって型が決まります。

### ラムダ式

明示的オブジェクト引数はラムダ式においても利用することができ、自身のクロージャ型オブジェクト（ラムダ式自体の`this`）を明示的に取ることができます。

```cpp
// 再帰ラムダの定義
auto fact = [](this auto self, int n) -> int {
  return (n <= 1) ? 1 : n * self(n-1);
};

int n = fact(5);  // ok、120
```

このように、再帰ラムダの定義が非常に簡潔になります。

ラムダ式内部ではキャプチャを参照するのに`this`を使用しないこともあり、明示的オブジェクト引数を取っている場合でもキャプチャを参照するために明示的オブジェクト引数を介する必要はなく、これまでどおりキャプチャ名のみで参照可能です。逆に、明示的オブジェクト引数を介してキャプチャにアクセスすることはできません。

```cpp
int main() {
  int i = 0;

  [=](this auto&& self) {
    ++i;         // ok、キャプチャをインクリメント
    self.i += 1; // ng、キャプチャ名は未規定
  };
}
```

つまり、オブジェクトメンバアクセスに関しては、一般のクラス型における明示的オブジェクト引数宣言の関数と逆になります。

クラス型の明示的オブジェクトパラメータの型は変換を受けることによって比較的自由に記述できましたが、キャプチャを行うラムダ式の場合は自身のクロージャ型、もしくはクロージャ型から派生したクラス型、あるいはそれらの型の参照型でなければなりません。

```cpp
struct C {
  // 万能変換コンストラクタ
  template <typename T>
  C(T);
};

void func(int i) {
  // ok、呼び出し時に自身のクロージャ型に推論される
  int x = [=](this auto&&) { return i; }();
  
  // ng、自身のクロージャ型と関係ない型
  int y = [=](this C) { return i; }();

  // ok、キャプチャしてないので変換できればok
  int z = [](this C) { return 42; }();
}
```

また、明示的オブジェクト引数を使用する場合、そのラムダ式がキャプチャを行っていると`mutable`指定を行えず、ラムダ式の関数呼び出し演算子は`const`修飾されません。

```cpp
int main() {
  int i;

  // ng、明示的オブジェクト引数とmutableは併用できない
  [=](this const auto&) mutable { return ++i; };

  // ok、明示的オブジェクト引数を非constにすればmutableと同等
  auto lm1 = [=](this auto&&) { return ++i; };
  int n = lm1(); // ok、1

  // 関数呼び出しをconstにしたければ
  // 明示的オブジェクト引数をconstにする
  auto lm2 = [=](this const auto&) { return ++i; };
  lm2(); // ng、iを変更できない

  // あるいは、ラムダ式をconstで受ける
  const auto lm3 = [=](this auto&&) { return ++i; };
  lm3(); // ng、iを変更できない
}
```

このサンプルを見ると何となく察せられますが、明示的オブジェクト引数を取る場合のラムダ式本体からのキャプチャの参照も暗黙的ながら明示的オブジェクト引数を介して行われており、なおかつキャプチャ名と実際のクロージャ型のメンバ名が自動で読み替えられてアクセスされています。

### `forward_like`

明示的オブジェクト引数を使用して、そのモチベーション通りに`this`の状態に応じた完全転送を行おうとすると、例えば次のようなコードを書くことになります。

```cpp
template<typename T>
struct wrap {
  T v;

  template<typename Self>
  auto value(this Self&& self) -> decltype(auto) {
    return std::forward<Self>(self).v; // あってる？
  }
};
```

これは`this`の状態（CV修飾および値カテゴリ）に応じてラップしているメンバ変数を完全転送するコードですが、実はこれは全ての場合に正しく動作しません。

詳細は後日発売（予定）の「C++23 ライブラリ機能」に譲りますが、`T`が参照型である場合にCV修飾と値カテゴリを正しく伝播できません。

つまりはここでの`std::forward`は完全なソリューションではなく別の転送関数が必要になります。C++23ではこれを`std::forward_like`というライブラリ機能として利用可能です。

```cpp
template<typename T>
struct wrap {
  T v;

  template<typename Self>
  auto value(this Self&& self) -> decltype(auto) {
    return std::forward_like<Self>(self.v); // 💯
  }
};
```

これにより、明示的オブジェクト引数を利用した場合の真の完全転送が実現できます。

### 応用例

最後に、この明示的オブジェクト引数という機能が新しいC++プログラミングパラダイムを開くものになるかもしれない例をいくつか見てみます。

まず最初はCRT抜きのCRTPです。

CRTPの応用例の一つとして、基底クラスで導出可能な演算子のボイラープレートを定義しておいて、それを継承した派生クラスで必須演算子だけ定義しておくと演算子を自動実装できるユーティリティが知られています。

```cpp
// 後置++を自動実装するCRTP基底クラス
template <typename Derived>
struct add_postfix_increment {
  Derived operator++(int) {
    auto& self = static_cast<Derived&>(*this);

    Derived tmp(self);
    ++self;
    return tmp;
  }
};

struct some_type : add_postfix_increment<some_type> {
  // 前置++だけを定義
  some_type& operator++() { ... }
};
```

このパターンは明示的オブジェクト引数を使うとCRT無しで書くことができるようになります。

```cpp
// 後置++を自動実装する基底クラス
struct add_postfix_increment {
  template <typename Self>
  auto operator++(this Self&& self, int) {
    // 派生クラスから呼ばれると、Selfで派生クラス型を取れる
    auto tmp = self;
    ++self; // 派生クラスで定義された前置++が使用される
    return tmp;
  }
};

struct some_type : add_postfix_increment {
  some_type& operator++() { ... }
};
```

次は再帰ラムダです。階乗を求める例は既に見ていますが、別の例として木構造をトラバースして葉の数をカウントする例を見てみます。

```cpp
struct Leaf { };
struct Node;

using Tree = variant<Leaf, Node*>;

struct Node {
  Tree left;
  Tree right;
};

int num_leaves(Tree const& tree) {
  return visit(overload(
    // Leafに行きつくとこっちが呼ばれる
    [](Leaf const&) { return 1; },
    // Nodeの間はこっちが呼ばれる
    [](this auto const& self, Node* n) -> int {
      // 深さ優先探索を行う
      return visit(self, n->left)
           + visit(self, n->right);
    }
  ), tree);
}
```

`overload`は複数のラムダ式をひとまとめにした呼び出し可能オブジェクトを返すイディオムを実装したものです（`overloaded`とも呼ばれています）。

最後は、SFINAE-friendly callablesを定義する例です。

関数ラッパのオーバーロードを定義する場合、例えば`std::not_fn`（渡された述語関数オブジェクトの結果を反転する関数オブジェクトを返す）を定義しようとすると、典型的には次のような呼び出し可能ラッパを介することになります。

```cpp
template <typename F>
class call_wrapper {
  F f;
public:
  ...
  
  // 実装は省略
  template <typename... Args>
  auto operator()(Args&&... ) &
      -> decltype(!declval<std::invoke_result_t<F&, Args...>>());

  template <typename... Args>
  auto operator()(Args&&... ) const&
      -> decltype(!declval<std::invoke_result_t<F const&, Args...>>());

  // ... same for && and const && ...
};

template <typename F>
auto not_fn(F&& f) {
  return call_wrapper<std::decay_t<F>>{std::forward<F>(f)};
}
```

とてもボイラープレートなのが分かると思います。正確性を増すためには`noexcept`等の考慮も必要です。しかし、それらとは関係ない問題として、このような実装は2つのコーナーケースに対して動作しない事が知られています。

1つは、ラップ対象の`operator()`がSFINAE-friendlyではないために本来機能するはずの呼び出しがエラーになる場合です。

```cpp
struct unfriendly {
  template <typename T>
  auto operator()(T v) {
    static_assert(std::is_same_v<T, int>);
    return v;
  }

  template <typename T>
  auto operator()(T v) const {
    static_assert(std::is_same_v<T, double>);
    return v;
  }
};

unfriendly{}(1);  // ok
not_fn(unfriendly{})(1); // ng
```

非`const`オーバーロードが有効かつ最適な候補ですが、オーバーロード解決の過程で両方のオーバーロードがインスタンス化されることで2つ目のオーバーロードの`static_assert`が起動します。ラップなしの場合、`const`修飾の有無でオーバーロードが決定されるので`const`オーバーロードはインスタンス化されません。

もう一つは、関数呼び出し演算子が`delete`されていることで本来失敗してほしい呼び出しが別の関数呼び出し演算子にフォールバックしてしまう問題です。

```cpp
struct fun {
  template <typename... Args>
  void operator()(Args&&...) = delete;

  template <typename... Args>
  bool operator()(Args&&...) const { return true; }
};

fun{}();  // ng、delete演算子を呼ぼうとする
not_fn(fun{})(); // ok、falseを返す
```

この例では、`not_fn(fun{})`の戻り値を直接呼び出していますがそれは`const`ではないため、`fun`で定義されているのに従って非`const`な呼び出しはできないようにしたいのですが、`call_wrapper`内の`const`オーバーロードが最終的に有効な候補として残ってしまうことで`fun`の`const`オーバーロードにフォールバックしてしまいます。

1つ目の問題はC++20までで解決できないことが分かっており、2つ目の問題は`call_wrapper`にさらに4つの関数呼び出し演算子オーバーロードを追加して、あるCV/参照修飾の候補について削除されたバージョンを追加することで解決できます。

しかし、明示的オブジェクト引数を用いると両方の問題を解決し、なおかつボイラープレートも削減できます。

```cpp
template <typename F>
struct call_wrapper {
  F f;

  template <typename Self, typename... Args>
  auto operator()(this Self&& self, Args&&... args)
    BOOST_HOF_RETURNS(
      !std::invoke(
        std::forward<Self>(self).f,
        std::forward<Args>(args)...))
};

template <typename F>
auto not_fn(F&& f) {
  return call_wrapper<std::decay_t<F>>{std::forward<F>(f)};
}
```

`BOOST_HOF_RETURNS(expr)`は定義を簡略化するためのものでその名の通りBoost.HOFというライブラリで用意されているマクロです。やっていることは、`expr`をコピペして`noexcept`と定義を実装したうえで、`decltype(auto)`戻り値で関数定義を完成させることです。

呼び出しラッパの関数呼び出し演算子オーバーロード層が実質的になくなっていることで、ラップ元のコンテキストで関数呼び出しを解決することができ、それによってラップしたことで起きていた問題が起こらなくなります。

これによって、先程の2つの問題の例はラッパを介さない場合の結果を再現できるようになります。

```cpp
// 正しい動作を得られる
not_fn(unfriendly{})(1); // ok
not_fn(fun{})();         // ng、delete演算子を呼ぼうとする
```

## アクセス制御の異なるメンバ変数のレイアウトを宣言順に規定

- P1847R4 Make declaration order layout mandated  
  (https://wg21.link/P1847R4 )

C++20までの規定では、クラスのアクセス制御（`private, public, protected`）が異なる場合、実装はメンバ変数を並べ替えてメモリに配置することができます。この利点は、型のアライメントの違いによって生じるメンバ変数間のパディングを最小化して構造体サイズを削減することで、メモリ効率を向上させることにあります。

ただし、実際にそれを行う処理系は存在せず、多くのC++プログラマは並べ替えを考慮していないことがほとんどです。この提案は、そのような慣行に従うように規定を修正し、クラスのメンバ変数のメモリレイアウトが常にコード上の宣言順と一致するようにするものです。

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

余談ですが、このクラスレイアウトの並べ替えの許可という仕様はC++11から入っていたものですが、意図して導入したものではなく標準レイアウトクラスという分類を導入する際に誤って混入してしまったもののようです。幸いなことにその仕様を活用する処理系は現れなかったため、この度めでたく消すことができました。

## 添字演算子の多次元サポート

- P2128R6 Multidimensional subscript operator(https://wg21.link/P2128R6 )

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

- P1169R4 static `operator()`(https://wg21.link/P1169R4 )

関数呼び出し演算子をオーバーロードする場合はクラスの非静的メンバ関数として定義する必要がありますが、非静的メンバ関数は暗黙に`this`引数を第一引数に取っています。これは、`callable(args...)`のように呼び出し可能オブジェクトに対して関数呼び出しを行う場合にいつも見えない`this`引数が一つ追加で渡されているということです。

通常これは問題にはなりませんが、関数呼び出し演算子を定義するものの`this`に全く依存しない、ステートレスな呼び出し可能型を定義する場合、この暗黙の`this`引数の存在は避け難いオーバーヘッドとなり得ます。多くの場合最適化（インライン展開）によって削除されますが、最適化しきれない場合にはこのオーバーヘッドから逃れることができません。このことはゼロオーバーヘッド原則にも反しています。

特に、C++20`<ranges>`のCPOやRangeアダプタの多くはステートレスな関数オブジェクトであり、その特性上ラッパ層のオーバーヘッドは可能な限り薄くしておく必要があります。

このオーバーヘッドを除去可能にするために、C++23からは`operator()`のオーバーロードを`static`メンバ関数として定義できるようになります。

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

この`static`な`operator()`は完全に指定された数の引数しか取らず、暗黙の`this`引数はありません。そのため、最適化なしでも追加の引数渡しによるオーバーヘッドは無くなります。ただし当然のことながら、クラスの状態（非静的メンバ変数）を読み取ることはできません。

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

また、前述の明示的オブジェクト引数宣言も`static`と併用できません。

### static `operator[]` 

- P2589R0 static `operator[]`(https://wg21.link/P2589R0 )

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

なお、提案が分かれているのは`operator[]`の複数引数許可と`static operator()`が同時に進行していたためで、`static operator()`が議論されていた段階ではまだ`operatpr[]`はC++20までの仕様のままだったためです。

\clearpage

# 定数式

## if consteval

- P1938R3 `if consteval`(https://wg21.link/P1938R3）

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

C++23では`if (std::is_constant_evaluated())`のこれらの問題を解決するために、この構文糖衣である新しいタイプの`if`文として`if consteval`が追加されます。

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

## `constexpr`関数内で`consteval`関数を呼び出せない問題を軽減

- P2564R3 Generalized wording for partial specializations  
  (https://wg21.link/P2564R3 )

前節の説明の中で、`if consteval`の`true`となるブロックが`consteval`関数内部と同等の扱いを受けることで`consteval`関数を呼び出すことができる、のように説明していました。ここからも分かるように、`consteval`関数の呼び出しはその引数が直接定数ではない場合、かなり制限されます。それによって、関数を受け取って内部で呼び出すようなAPIにおいて、`consteval`関数を渡そうとすると謎のエラーに遭遇する可能性があります。

そのようなAPIの最たる例は`<algorithm>`にあるアルゴリズム関数でしょう。ここでは`std::ranges::all_of`で見てみます。

```cpp
// consteval述語関数
consteval bool is_even(int n) {
  return n % 2 == 0;
}

// 定数式で使用可能な範囲（配列）
constexpr std::array<int, 3> arr = {2, 4, 6};

// 1. consteval関数のアドレス（参照）を取得できない
static_assert(std::ranges::all_of(arr, is_even));

// 2. ラムダ式の引数nが定数式で使用できない
static_assert(std::ranges::all_of(arr, [](int n) { return is_even(n); }));

// 3. ranges::all_of()本体内はconsteval関数を呼び出せるコンテキストではない
static_assert(std::ranges::all_of(arr, [](int n) consteval { return is_even(n); }));

// 4. consteval関数のアドレスを取得できない
static_assert(std::ranges::all_of(arr, +[](int n) consteval { return is_even(n); }));
```

これら4つの例は全て、それぞれ微妙に異なる理由によってコンパイルエラーを起こします（GCC11のC++2aモードなどで確認できます）。

まず、`consteval`関数のアドレスを取得できるのは`consteval`関数本体のコンテキスト（これを即時呼び出しコンテキストと呼びます）のみ、というルールがあり、`static_assert()`の内部は即時呼び出しコンテキストではないため、1と4は関数のアドレスも参照も取得できずにエラーになります。

次に、`constexpr`関数の仮引数は定数式ではないためそれを用いて`consteval`関数を呼び出すことはできません（これは前節でも問題となっていたことでした）。単なる`constexpr`ラムダの仮引数`n`は定数式で使用できないため、2はエラーになります。

最後に、`consteval`関数の引数が定数ではない場合に`consteval`関数を呼び出せるのもまた即時呼び出しコンテキストのみですが、`ranges::all_of`の定義内は単なる`constexpr`関数のコンテキストであるため、その内部で`consteval`関数（`is_even`の呼び出しをラップした`consteval`ラムダ）を呼び出すことができず、3はエラーになります。

3の場合、ラムダが`consteval`になっているのでその本体は即時呼び出しコンテキストとなり引数の使用については問題ないのですが、それを呼び出すことになる`ranges::all_of`の本体内は`consteval`ではないためにエラーになっています。

このことは`consteval`の仕様をよく知らないとなぜエラーになっているのか分からず、解決方法もC++20の範囲ではほぼありません。`static_assert()`ではなく`consteval`関数内で呼び出したりしていると多少緩和されますが、完全ではありません。また、Rangeアルゴリズムの実装側で分岐をするということも、実装が複雑になりコードがかなり冗長になることから現実的ではありませんでした。

そのため、C++23では次の3つのパターンの場合に定数式のコンテキストを自動的に即時呼び出しコンテキストに昇格させることで、この問題が緩和されます

1. `constexpr`関数に即時呼び出しコンテキスト外の`consteval`関数呼び出しが含まれていて、その呼び出しが定数式ではない場合、その`constexpr`関数は暗黙的に`consteval`関数となる
2. 名前を指定する式が`constexpr`関数内の即時呼び出しコンテキスト外で`consteval`関数を指定する場合、そのコンテキストは暗黙的に即時呼び出しコンテキストとなる
3. `std::is_constant_evalueted()`が`true`となるコンテキストは即時呼び出しコンテキストと見做される
    - `static_assert()`や`constexpr if`など

`consteval`関数のポインタ（参照）を取得できなかった1と4の例は、パターン3に該当することでそれが取得できるようになり、なおかつ`ranges::all_of`の本体内の呼び出し式がパターン1に該当することでエラーにならなくなります。

`constexpr`ラムダ式の引数を定数式として使えずエラーになっていた2の例と、`ranges::all_of`の本体内が即時呼び出しコンテキストではないことでエラーになっていた3の例も、`ranges::all_of`の本体内の呼び出し式がパターン1に該当することでエラーにならなくなります。

この提案はC++20への欠陥報告として採択されているため、実装済みのコンパイラではC++20モードでもこの修正が適用されます。

なお、この提案はどうやらC++26に向けた静的リフレクション機能の設計・先行実装作業を行っていた際に遭遇したもののようです。C++26を予定している静的リフレクション機能では、`consteval`関数によって取得したリフレクション情報を加工したりリフレクション情報の`std::vector`をアルゴリズムで処理したりすることができるため、この問題が解消されない場合非常に使いにくいものになっていたでしょう。

## 一部の定数式の文脈での `bool` への縮小変換を許可

- P1401R5 Narrowing contextual conversions to bool  
  (https://wg21.link/P1401R5 )

定数式においては整数型から`bool`型への縮小変換は禁止されており、それが行われるとコンパイルエラーになります（ただし、整数値が`0`か`1`の場合は縮小変換にならないためエラーにはなりません）。特に、`static_assert`や`constexpr if`の条件式においてもこれはエラーになっていました

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

このような条件式における整数型から`bool`への縮小変換は実行時のアサーションや`if`文では普通に行われており、コンパイルエラーも実行時エラーも起こさずに行うことができます。このことは、実行時とコンパイル時で条件式を記述する際に余分に注意すべきことが増えており、一貫性がありませんでした。

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

- P2246R1 Character encoding of diagnostic text(https://wg21.link/P2246R1 )

`static_assert()`には2つ目の引数に診断メッセージを文字列リテラルとして渡して、条件が`false`になった場合にコンパイラのエラーメッセージとしてその文字列を出力させることができます。

C++20までの`static_assert()`の規定では、ソースコードのエンコーディングと実行時のエンコーディングが異なっている場合に実行時エンコーディングで表現可能ではない文字は出力する必要は無い、のように規定されていました。

しかし、診断メッセージの出力は実行時ではなくコンパイル時にコンパイラの実行環境のエンコーディングによって行われ、かつソースコードの文字列のエンコーディングとコンパイル時のコンパイラのメッセージ出力先（ターミナル）のエンコーディングには直接的な関係はありません。すなわち、`static_assert()`のこのような規定はほぼ無意味なものとなっていました。

さらに、`[[deprecated]]`などのコンパイル時に診断メッセージを出力する同様の機能にはそのような規定は備わっていなかったこともあり、`static_assert()`にあったエンコーディングに関する規定は削除されることになりました。

ただし、診断メッセージの文字がコンパイラの実行環境のターミナルのエンコーディングで表示できない場合にどのように出力されるかは、実装品質の問題とされ、どのように出力されるかが指定されるようになるわけではありません。

## 定数式内での非リテラル変数、静的変数・スレッドローカル変数およびgotoとラベルの存在を許可する

- P2242R3 Non-literal variables (and labels and gotos) in constexpr functions(https://wg21.link/P2242R3 )

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

- P2280R4 Using unknown pointers and references in constant expressions(https://wg21.link/P2280R4 )

配列に対する`std::size()`の適用をコンパイル時に行おうとすると、たまに不可思議な現象に出会うことがあります。

```cpp
void check(int const (&param)[3]) {
  int local[] = {1, 2, 3};

  constexpr auto s0 = std::size(local); // ok
  constexpr auto s1 = std::size(param); // ng
}
```

この`s0, s1`はともに`constexpr`ローカル変数であり、その初期化式（ここでは`std::size()`の呼び出し）は定数式でなければなりません。しかし、ここでは変数`local`は定数式として扱われるのに対して、変数`param`は定数式として扱われておらず、それによってエラーになります。

これは、`param`は定数式で使用できない参照の条件に当てはまっているために起こるエラーであり、それは関数引数`param`が初期化されているかどうかが不明であるためです。ただ、`std::size()`の配列（`std::array`含む）についての実装は配列そのものに一切アクセスせず、その型情報からそのサイズを取得します。そのため、この場合参照`param`が初期化されていようがいまいが関係ないはずです。

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

- P2448R2 Relaxing some `constexpr` restrictions(https://wg21.link/P2448R2 )

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

複数の言語バージョンをターゲットとしてコンパイルされるプログラムやライブラリなどでは、このような関数を`constexpr`指定しておくのは困難です。言語機能か標準ライブラリの場合は機能テストマクロが提供されていますが、在野の多くのライブラリはそのようなものを用意しません。機能テストマクロを使って分岐したとしても、機能ごとに細かくどのバージョンから定数式で使用可能になったのかを把握して調整しなければならず、その作業はあまり現実的ではありません。

このようなエラー方針は、定数式で実行可能なものが大きく制限されていたC++11の時代には有用だったのかもしれませんが、より多くのものが定数式で実行可能となった現在（C++23）の環境ではむしろ足枷となっていました。

そのため、C++23からはある`constexpr`関数が仮にすべての入力に対して定数式で実行できなくても、実際に実行されるまではエラーにならないようになります。

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

- P2647R1 Permitting `static constexpr` variables in `constexpr` functions  
  (https://wg21.link/P2647R1 )

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

\clearpage

# テンプレート

## 変数テンプレートの部分特殊化の仕様明確化

- P2096R2 Generalized wording for partial specializations  
  (https://wg21.link/P2096R2 )

変数テンプレートはC++14で導入された機能です。その仕様はクラステンプレートの仕様を踏襲する形で規定されていましたが、そのために変数テンプレートの仕様は分かりづらく、特に部分特殊化についてが曖昧でした。

C++23ではこの仕様が改善され、変数テンプレートの仕様が明確化されるとともに、変数テンプレートにおいても部分特殊化が使用可能であることが明確化されました。

```cpp
// 変数テンプレートによるis_sameメタ関数の実装例

template<typename T, typename U>
inline constexpr bool is_same_v = false;

// T == Uの時の部分特殊化
template<typename T>
inline constexpr bool is_same_v<T, T> = true;
```

ただし、これによって変数テンプレートでできることが増えたわけではありません。変数テンプレートの部分特殊化はC++14から可能です。

## 継承コンストラクタからのクラステンプレート引数の推論

- P2582R1 Wording for class template argument deduction from inherited constructors(https://wg21.link/P2582R1 )

クラステンプレートの実引数推定（CTAD）はC++17で追加され、当初は非集成体のクラステンプレートそのものでしか使用できませんでした。C++20にて集成体テンプレートとエイリアステンプレートに拡大されましたが、継承コンストラクタからのCTADはできないままでした。

C++23では継承コンストラクタからのCTADが行えるようになります。

```cpp
template <typename T>
struct Base {
  Base(T&&);
};

template <typename T>
struct Derived : public Base<T> {
  using Base<T>::Base;  // Baseのコンストラクタを継承
}

Derived d(42); // ok、Derived<int> と推論される
```

CTADの仕組みは、型名（エイリアス含む）とそのコンストラクタからなんとかして関数テンプレートを導出して、その関数テンプレートに実引数を渡してオーバーロード解決して関数テンプレートのテンプレートパラメータを推論し、その結果を元の型名のテンプレートパラメータにフィードバックする、という感じになっています。

継承コンストラクタからのCTADでは、派生クラスに対する基底クラスをエイリアステンプレートのように扱って、エイリアステンプレートからのCTADのアルゴリズムを流用するとともに、継承コンストラクタから生成された推論補助（関数テンプレート）を派生クラスで直接生成されたものよりもオーバーロード順で優先することで行われています。

最初の推論補助（関数テンプレート）の抽出は、クラステンプレート`C`の基底クラスにクラステンプレート`B`が含まれている場合、`C`のテンプレートパラメータを持ち右辺が`B`であるようなエイリアステンプレートを生成し、このエイリアステンプレートを用いて推論補助を取得します。先程の例（`C = Derived`、`B = Base`）だと次のようになります

```cpp
// 生成された、仮想的なエイリアステンプレート
template <typename T>
using D = Base<T>;

// ↑から生成された推論補助
template<typename T>
Derived(T&&) -> Derived<T>;
```

このように生成した推論補助を関数テンプレートとして、コンストラクタ呼び出しに渡されている実引数を渡してオーバーロード解決とテンプレートパラメータ推論を行い、推論した結果（上の例だと`T`）を元のクラステンプレート（`Derived`）にフィードバックして補うことで、テンプレートパラメータが推論されます（ここの後段の手順は全ての場合で共通です）。

基底クラスに推論補助が存在していてコンストラクタ引数がそれに対応する場合、その推論補助がさらに優先的に使用されます。

```cpp
// 上の例の後で、次の推論補助を追加
Base(int) -> Base<char>;

Derived d2(42); // ok、Derived<char> と推論される
```

より複雑な例

```cpp
template <typename T, typename U, typename V>
struct F {
  F(T, U, V);
};

template <typename T, typename U>
struct G : F<U, T, int> {
  // Gのテンプレートパラメータと継承時のパラメータでコンストラクタを継承
  using G::F::F;
}

G g(true, ’a’, 1); // ok、G<char, bool> と推論される
```

この場合、次のように推論補助が生成されています

```cpp
// 生成された、仮想的なエイリアステンプレート
template <typename T, typename U>
using D = F<U, T, int>;

// ↑から生成された推論補助
template<typename T, typename U>
G(T&&, U&&, int) -> G<U, T>;
```

この継承コンストラクタからのCTADにおいては、基底クラスが非テンプレートであったり、派生クラスのテンプレートパラメータと全く無関係だったりすると推論は失敗します。

\clearpage

# 文字・文字列リテラル

## 異なる文字エンコーディングをもつ文字列リテラルの連結を不適格とする

- P2201R1: Mixed string literal concatenation(https://wg21.link/P2201R1 )

ソースコード上の文字列リテラルがトークンとして連続している場合、プリプロセス時に1つの文字列リテラルに連結されます。

```cpp
int main() {
  auto str = "abc" "def" "ghi"; // "abcdefghi"として扱われる
}
```

このとき、そのエンコード指定のプリフィックス（`u8 L u U`）が異なっている場合の動作は実装定義で、規格では条件付きでサポートされるとされており、実際には主要なC++コンパイラはエラーとします。ただし、UTF-8文字列リテラルのその他のリテラルとの連結は明示的に禁止されています。

```cpp
int main() {
  auto str1 = L"abc" "def" U"ghi";   // 実装定義
  auto str2 = L"abc" u8"def" U"ghi"; // u8がある場合はng
}
```

あるマイナーなCコンパイラはこれをサポートしているようですが、主要なC/C++コンパイラではサポートされていません。そして、この挙動を必要とするようなユースケースもどうやら存在していなかったため、C++23では異なるエンコーディング指定を持つ文字列リテラルの連結は禁止され、コンパイルエラーになります。

```cpp
int main() {
  auto str = L"abc" "def" U"ghi";   // ng
}
```

## エスケープシーケンスの区切り

- P2290R3 Delimited escape sequences(https://wg21.link/P2290R3 )

文字列中のエスケープシーケンスには、ユニバーサル文字名（`\uxx... or \Uxx...` : `x`は16進数の数値1つ）、8進エスケープシーケンス（`\onn` : `n`は0~7の数値）、16進エスケープシーケンス（`\xhh...` : `h`は16進数の数値1つ）の3種類があります。8進エスケープシーケンスは3文字制限がありますが、16進エスケープシーケンスには長さの制限はありません。そして、どちらもエスケープシーケンス中に受け付けられない文字が出てきたらそこでエスケープシーケンスを終了するようになっています。これによって、次のような問題が発生します

```cpp
"\17";      // 8進エスケープシーケンス、"0x0f"と等価
"\18";      // 8進エスケープシーケンスと文字、"0x01 8"の2文字
"\xabc";    // 1文字
"\xab" "c"; // 2文字
```

つまりどれも、エスケープシーケンスの終端（あるいは区切り）が明確ではありません。一番最後の例の様な回避策はありますが分かりづらく、この問題をよく知らない人から見ると余計なことをしているようにしか見えません。

また、ユニバーサル文字名は16進数字4桁もしくは8桁のどちらかになりますが、ユニコードのコードポイントの範囲が[0, 0x10FFFF]に制限されているため、有効なコードポイントは5桁以下の16進数字列によって書くことができます。そして、5桁のユニコード文字を書く場合は`\U0001F1F8`のように冗長な0が必要になってしまいます。

このような問題の回避のため、C++23からはこれらのエスケープシーケンスに対応する新しい構文が追加されます。

- ユニバーサル文字名: `\u{x...}`
    - `x`は16進数の数値1つ
- 8進エスケープシーケンス: `\o{n...}`
    - `n`は0~7の数値1つ
- 16進エスケープシーケンス: `\x{h...}`
    - `h`は16進数の数値1つ

どれも現在のエスケープシーケンスの形式に`{}`が続き、`{}`内にエスケープしたい文字列を置くようになっています（`\o`はエスケープシーケンスではないものの予約されていた）。

```cpp
"\o{17}";     // ok、"0x0f"と等価
"\o{1}8";     // ok、"0x01 8"の2文字
"\o{18}";     // ng、8は現れてはいけないのでコンパイルエラー
"\x{abc}";    // ok、1文字
"\x{ab}c";    // ok、2文字
"\u{1F1F8}";  // ok、5桁のユニバーサル文字名
```

## 文字・文字列リテラル中の数値・ユニバーサルキャラクタのエスケープに関する問題解決

- P2029R4 Proposed resolution for core issues 411, 1656, and 2333; numeric and universal character escapes in character and string literals(https://wg21.link/P2029R4 )

この提案は、文字（列）リテラル中での数値エスケープ文字（`'\xc0'`）やユニバーサル文字名（`"\u000A"`）の扱いに関する3つのIssueの解決のためのものです。


- [Core Issue 411: Use of universal-character-name in character versus string literals](http://www.open-std.org/jtc1/sc22/wg21/docs/cwg_active.html#411)
    - ユニバーサル文字名の実行時文字集合へのエンコーディングの規定が矛盾している問題
    - [「文字列リテラル中のユニバーサル文字名は文字リテラルのそれと同じ意味を持つ。文字列リテラルでは、ユニバーサル文字名はマルチバイトエンコーディングのために複数の文字要素に対応することがある」](https://timsong-cpp.github.io/cppwp/n4861/lex.string#13)と規定がある一方で、[「ユニバーサル文字名は単一の`char`にエンコードされ、対応する`char`値がなければ実装定義の値にエンコーディングされる」](https://timsong-cpp.github.io/cppwp/n4861/lex.ccon#8)と規定されており、この2文が矛盾している
    - 例えば実行時文字集合がUTF-8である時、[`U+0153`](https://www.compart.com/en/unicode/U+0153)は文字リテラルとしてどうエンコードされるのかが曖昧（UTF-8だと2文字になる）
- [Core Issue 1656: Encoding of numerically-escaped characters](http://www.open-std.org/jtc1/sc22/wg21/docs/cwg_active.html#1656)
    - 数値エスケープ文字の値を解釈するためのエンコードがソースコードのエンコーディングなのか、実行時エンコーディングなのかが指定されていない問題
    - ソースエンコーディングがLatin-1である時、`u8"\xff"`は[`U+00FF`](https://www.compart.com/en/unicode/U+00FF)としてエンコードされるのか、単に最初のバイトが`0xff`である2バイト文字にエンコードされるのかが曖昧（いくつかの実装は後者のようにエンコードする）
- [Core Issue 2333: Escape sequences in UTF-8 character literals](http://www.open-std.org/jtc1/sc22/wg21/docs/cwg_active.html#2333)
    - UTF-8文字（列）リテラル中での数値エスケープ文字の扱いが不明瞭であり、未定義動作に陥っている問題
    - ただし、主要3実装はUTF-8リテラル中での数値エスケープ文字使用を許可している

それぞれについて次のように振る舞いを明確化しています

- Core Issue 411
    - 単一の文字として表現できないユニバーサル文字名のエンコーディングは文字リテラルと文字列リテラルで異なる振る舞いをする、と規定する
    - 例えば実行時文字集合がUTF-8である時、`'\u0153'`は（サポートされる場合）`int`型の実装定義の値を持つが、`"\u0153"`は`\xC5\x93\x00`の長さ3の文字配列となる
- Core Issue 1656
    - 数値エスケープ文字のエンコードは実行時文字集合であると明確化する
    - 結果、数値エスケープ文字の値はソースファイルの文字コードの値として実行時文字集合の値へ変換されることはない（常に実行時文字集合のコードポイントを表す）
    - `u8"\xff"`は`U+00FF`としてUTF-8エンコードされる
- Core Issue 2333
    - UTF-8リテラル中での数値エスケープ文字使用には価値があるので、明確に振る舞いを規定する

なお、これらの修正は既存のコンパイラの振る舞いをベースに行われたため、ほとんどの場合これによって動作が変わることはないはずです（MSVCは一部影響があるとのこと）。

## ワイド文字1文字に収まらないワイド文字リテラルを禁止する

- P2362R3 Remove non-encodable wide character literals and multicharacter wide character literals(https://wg21.link/P2362R3 )

文字リテラル（`'c'`）には複数の文字を指定することができ、それはワイド文字リテラル（`L'c'`）においても同様です。ワイド文字リテラルではそれに加えて、1文字が1つのコード単位に収まらない文字リテラルを書くことができます。

```cpp
// 全てC++20ではok
wchar_t a = L'🤦‍♀️';  // \U0001f926
wchar_t b = L'ab';  // multi character literal
wchar_t c = L'é́';   // \u0065\u0301
```

上記の`a`は`wchar_t`のサイズが4バイトである環境（Linuxなど）では書いたままになりますが、2バイトの環境（Windowsなど）だと表現しきれないためUTF-16エンコーディングで読み取られた後に、上位か下位の2バイトが最終的な値として取得されます（Windowsは上位2バイトが残る）。

`b`はマルチキャラクタリテラルと呼ばれるもので、どの文字が残るか、あるいはどういう値になるかは実装定義とされます。MSVCでは最初の文字が、GCC/Clangでは最後の文字が残るようです。

`c`は2つのユニコード文字から構成されており、これもマルチキャラクタリテラルの一種です。これは1文字で同じ表現ができる文字がユニコードに存在していますが（`\u00e9`）、`e`と`́́`の2文字を組み合わせて表現することもでき、後者の場合は表示上は1文字ですが、1コード単位ではなく2コード単位の文字列となります。

韓国語やブラーフミー文字などでは、通常使用する文字でも1文字が複数のコードポイントからなるような文字が存在しており、そのような環境でその文字をワイド文字リテラルで表現しようとすることは常にバグです。また、ワイド文字リテラルは常に`wchar_t`の値1つ分しか生成することができないため、複数文字からなるリテラルを有効に解釈することはできず、意味のある値を生成することはどう頑張ってもできません。

そのため、C++23ではこれらの意味のない結果をもたらす複数文字からなるワイド文字リテラルを禁止し、コンパイルエラーにするようにします。

```cpp
wchar_t a = L'🤦‍♀️';  // ng、ただしwchar_tが4バイトならok
wchar_t b = L'ab';  // ng
wchar_t c = L'é́';   // ng
```

ただし、`wchar_t`のサイズが4バイト（UTF32）である環境の上記`a`のケースは適正であるため、引き続き使用可能とされます。

これは一応破壊的変更となりますが、このような複数文字を含むワイド文字リテラルの意味のある解釈は不可能であるため、ユーザーはこれを使用してはいないはずです。実際、オープンソースのコードベースの調査ではコンパイラのテストケースを除いて使用されているコードは発見されなかったようです。

## 名前付文字エスケープ

- P2071R2 Named universal character escapes(https://wg21.link/P2071R2 )

文字・文字列リテラル中で任意のユニコード文字を表すためには、ユニバーサル文字名（`\uxxxx`）を使用できます。これは特に、ソースコードのエンコーディングが非ユニコードの場合でも、ユニコードで表現可能な文字をソースコード上で表すためのポータブルな方法でもあります。

```cpp
// 全て🤔
const char8_t* u8str = u8"\U0001f914";
const char16_t* u16str = u"\U0001f914";
const char32_t u32char = U'\U0001f914';
```

ただし、ほとんどの人はこのユニバーサル文字名の指定を見てもそれがどの文字を表現しているのか分からず、直感的ではありませんでした。

C++23からは、コードポイントの数値ではなく文字の名前で指定するタイプの新しいユニバーサル文字名である、名前付文字エスケープ（Named character escape）を使用できるようになります。構文は、`\N{character_name}`の形でユニバーサル文字名と同じところで使用できます。

```cpp
// 全て🤔
const char8_t* u8str = u8"\N{THINKING FACE}";
const char16_t* u16str = u"\N{THINKING FACE}";
const char32_t u32char = U'\N{THINKING FACE}';
```

ここに指定する`character_name`はユニコード規格で文字（コードポイント）毎に指定されている名前（ユニコード名などと呼ばれている）を指定します。例えば絵文字だとウィキペディアの「[UnicodeのEmojiの一覧](https://ja.wikipedia.org/wiki/Unicode%E3%81%AEEmoji%E3%81%AE%E4%B8%80%E8%A6%A7)」というページの表でユニコードでの名称の列の名前がそれにあたります。

他の文字にも対応する名前があり、名称を知りたい文字を`C`とおくと「C ユニコード」とかでググって出てくるサイトで名称とかユニコード名とか呼ばれているものがそれにあたります。あるいは、ユニコード名のエイリアスと呼ばれる名前と制御文字のユニコード文字データベース名とというものも使用可能です。

この名前の表記はユニコードの指定と厳密に一致している必要があり、大文字が小文字になっていたり、スペースが足りない/余分にあるようなケースはエラーになります（コンパイラは割と柔軟にこの間違いを警告してくれはします）。また、使用可能な文字は英字大文字小文字と数字、スペースにハイフン（マイナス）記号（いずれもASCII範囲のもの）のみで、それ以外の文字が現れていてもエラーになります。

```cpp
// ngな例
const char32_t ng1 = U'\N{THINKING-FACE}';
const char32_t ng2 = U'\N{thinking face}';
const char32_t ng3 = U'\N{THINKING  FACE}';
```

なお、ユニコード名は基本的には全て大文字で構成されているようです。

## `wchar_t`のエンコーディングについての制限の緩和

- P2460R2 Relax requirements on wchar_t to match existing practices  
  (https://wg21.link/P2460R2 )

現在のC++標準における`wchar_t`のエンコーディングに関する規定は、ワイド文字エンコーディングのすべての文字を単一のコード単位としてエンコーディングする必要がある（すなわち、すべての文字が`wchar_t`1つに収まらなければならない）ことを要求しています。例えばWindowsではそれはUTF-16であり、UTF-16はサロゲートペアを持つためその要件に違反していることになります。

さらに、`wchar_t`の値はサポートされているロケールで指定された拡張文字セットの個別の文字を表現できることも要求されているため、実行字文字集号がUTF-8の場合はすべてのユニコード文字を`wchar_t`1つで表現可能であることが求められ、実質的に2バイトエンコーディングを排除しています。

これらのことは実際の実装（主にWindows）に反していたため、C++23では実態を反映する形で規格のこのような制約が修正されます。

この提案による変更は実装ですでに行われていることを標準化するだけなので、プログラマには影響はありません。そして、この提案の修正はC++98に対する欠陥報告であるため、Windowsにおける実装は事後的に合法化されます。

\clearpage

# ラムダ式

## 引数無しのラムダ式でより`()`を省略できるようにする

- P1102R2 Down with ()!(https://wg21.link/P1102R2 )

引数無しのラムダ式はそのかっこ（`()`）を省略することができます。

```cpp
[n] {  // ok
  return n == 10;
};
```

しかし、ラムダ式に対して何らかの指定をしていると途端に省略できなくなります。

```cpp
[n] noexcept {  // ng
  return n == 10;
};

[n] -> bool {  // ng
  return n == 10;
};
```

詳細には、引数無しで`()`を省略しているラムダ式の`[]`の後関数本体開始の`{`までの間に次のいずれかのものが置かれていると`()`を省略できなくなります

- テンプレートパラメータ指定
- `constexpr`
- `mutable`
- `consteval`
- `noexpcet`
- 属性
    - ただしC++20時点では標準属性で指定可能なものはない
- 後置戻り値型
- `requires`節

C++23では、これらのものがある場合でも引数無しラムダ式の`()`を省略できるようになります。これにより、先程のng例は全てそのままエラーにならなくなります。

```cpp
// ok（外側で型名Tが利用可能であるとする
[]<typename U = T> requires std::integral<U> [[nodiscard]] static constexpr noexcept -> bool
{ 
  return true;
};
```

## ラムダ式に対する属性

- P2173R1 Attributes on Lambda-Expressions(https://wg21.link/P2173R1 )

関数に対して指定することで有用な属性は標準属性の中にもいくつかあります。それは当然ラムダ式に対しても同じですが、C++20までのラムダ式には属性を指定できませんでした。より正確には、属性をしていしても関数呼び出し演算子ではなく関数型に対する指定になっていました。

```cpp
// これは関数型に対する指定になる（無意味
[](int n) [[deprecated]] {...};
[] [[deprecated]] {...};
```

（clangは関数型に対する属性指定の有効性を厳密にチェックするため、このコードはエラーになります）

これは意味がなく（C++23でも標準属性で有効なものはない）、関数オブジェクトを手書きした時にはその関数呼び出し演算子に属性を指定できるのに略記法であるラムダ式でも指定できるのが妥当であるとして、C++23ではラムダ式の関数呼び出し演算子に対する属性指定が可能になります。

構文的にはラムダ導入の`[]`すぐ後ろ、引数リストの`()`の前、の位置に置いた属性がラムダ式の関数呼び出し演算子に対する属性指定となります。

```cpp
// 関数呼び出し演算子に対する属性指定
[] [[nodiscard]] (int n)  {...};  // ok
[] [[noreturn]]  (int n)  {...};  // ok

// 両方に指定
[] [[nodiscard]] (int n) [[deprecated]] {...};  // ok（clangはエラー

// これも関数呼び出し演算子に対する属性指定（破壊的変更
[] [[nodiscard]] {...};  // ok
```

引数無しでラムダ式の`()`を省略している場合、属性の指定先が変化します。これは一応破壊的変更となりますが、以前の用例は有効性が乏しく影響は小さいとして受け入れられたようです。

引数無しでラムダ式の`()`を省略している場合に従来のように関数型に対して属性を指定したい場合は、`constexpr`や`mutable`、`noexcept`の指定が存在する場合にのみ、それらの後に指定することができます。

```cpp
// 関数型に対する指定（無意味
[] static [[deprecated]] {...}; // ok
```

ただし前述のように、標準属性の中にはここに指定して意味のあるものは存在しません。

## ラムダ式の後置戻り値型のスコープ変更

- P2036R3 Change scope of lambda trailing-return-type  
  (https://wg21.link/P2036R3 )
- P2579R0 Mitigation strategies for P2036 ”Changing scope for lambda trailing-return-type”(https://wg21.link/P2579R0 )

C++20までのラムダ式には、後置戻り値型の変数のスコープがそれ以外の部分と異なるという問題があります。例えば、次のようなコードはエラーになる場合があります

```cpp
// jがこのスコープに存在していなければエラー
auto counter1 = [j=0]() mutable -> decltype(j) {
  return j++;
};
```

`j`という名前がこのスコープに存在していない場合、後置戻り値型の`decltype(j)`は`j`を見つけられずにエラーになります。

しかしこの場合に`j`がラムダを宣言しているスコープで存在しているとエラーにならなくなります。特に、初期化キャプチャしている`j`と外側の`j`の型が異なっていたりすると、これはとても気づき辛くなります。

```cpp
int j = 0;

// エラーにならない、戻り値型はint
auto counter1 = [j=0.0]() mutable -> decltype(j) {
  return j++;
};
```

C++23からはラムダ式の後置戻り値型の変数のスコープが修正され、後置戻り値型に現れる名前はまずキャプチャしたものを考慮するようになります。これによって、ラムダ式の本体と後置戻り値型指定の双方で現れる名前の扱いが一貫するようになります。

```cpp
int j = 0;

// ok、戻り値型double
auto counter1 = [j=0.0]() mutable -> decltype(j) {
  return j++;
};
```

ただし、後置戻り値型に現れる名前が実際にキャプチャされていない場合でも、キャプチャされた場合の型が取得されます。

```cpp
int i;

// 以前の戻り値型: int&
// この提案以降: const int&
// iはキャプチャされていないがされている場合の型が取得される
auto f = [=](int& j) -> decltype((i)) {
  return j;
};
```

これは、デフォルトキャプチャ（`[=]`or`[&]`）の場合に後置戻り値型で現れる名前が実際にキャプチャされているかはラムダ式の本体を見なければわからず、それを実装に要求するとパーサーの実装が複雑化することが予想されたため、それを回避するための措置です。

後置戻り値型に加えて、同様の名前探索スコープの問題があったのがラムダ式の引数リストです。

```cpp
float x;

[x=1](decltype((x)) y) {
  // yの型は？
  ...
};
```

C++23ではこちらも後置戻り値型と同様に現れる名前はまずキャプチャしたものを考慮するようになります。しかし、こちらの場合は`mutable`指定が問題となります。

ラムダ式の`mutable`指定の有無でキャプチャされた変数の`const`性は変化します。後置戻り値型からはこれが見えていますが、引数リストの時点では見えていません。すると引数で現れる名前について先程と同様にキャプチャされているか分からないという問題に加えて、キャプチャされているとしても`mutable`かどうかが分からないという問題が発生します。C++23ではこの場合、デフォルトキャプチャの場合に引数で現れる名前もキャプチャされたものとして扱われ、なおかつ常に`mutable`（すなわち非`const`）として扱われます。

```cpp
float x;

[x=1](decltype((x)) y) {
  // yの型: int&
  // 戻り値型: int
  return X;
};

[=](decltype((x)) y) {
  decltype((x)) z = x;
  // yの型: float&
  // zの型: const float&
};
```

これはC++11環境でジェネリックラムダを再現した次のようなコードの後方互換のためです

```cpp
// 何かのイテレータ範囲のendの値
auto local_end = ...;

[local_end](decltype(local_end) it) { return it != local_end; };
```

この提案以前、`decltype(local_end)`の`local_end`は外の変数名が参照されていました。この提案ではキャプチャしたものが参照されるようになりますが、ここでは常に非`const`（`mutable`キャプチャ）として扱われることで以前のコードへの影響を緩和しています。

なお、これらの修正は欠陥報告としてC++11に対して適用されています。

\clearpage

# プリプロセッサ

## 文字リテラルの扱いを通常のC++コードと一貫させる

- P2316R2 Consistent character literal encoding (https://wg21.link/P2316R2 )

C++20まで、プリプロセッサにおける文字リテラルの比較とC++コードにおける文字リテラルの比較の結果が一致する保証がありませんでした。例えば、次のようなコードでは同じ環境でも異なった比較結果を規格的には許容します

```cpp
#if 'A' == '\x41'
//...
#endif

if ('A' == 0x41){}
```

現在の仕様では、`#if`の条件式において文字リテラルは対応する数値に変換され処理されますが、文字リテラルをどのように解釈するか（どのエンコーディングで読み取るか）は実装定義であり、C++の式上でのそれと一致するかどうかも実装定義とされます。仮に環境の1バイト文字のエンコーディングがASCII互換であったとしても、プリプロセッサにおいて`'A'`のような文字リテラルを整数として読み取ってもASCIIコードの整数値（`\x41`）に変換されるかは無保証でした。

このような`#if`のコードは、在野のライブラリでは環境の1バイト文字エンコーディングを検出するために使用されています。例えばsqliteでは次のようなコードが使用されています

```cpp
#if 'A' == '\301'
# define SQLITE_EBCDIC 1
#else
# define SQLITE_ASCII 1
#endif
```

このコードは現存するおそらく全てのコンパイラで意図通りに動作していますが、規格的には前述のように保証がありませんでした。C++23からは規定を修正することでこのようなコードが正しく意図通りに動作することを保証します。

## `elifdef` / `elifndef`ディレクティブの追加

- P2334R1 Add support for preprocessing directives elifdef and elifndef(https://wg21.link/P2334R1 )

`#ifdef/#ifndef`は`#if defined(macro_name)/#if !defined(macro_name)`の糖衣構文として随分前から利用可能ですが、`#elif defined(macro_name)/#elif !defined(macro_name)`に対応する糖衣構文はありません。

これは使用可能なディレクティブが一貫していなかったとして、C++23からは`#elif`にも`#ifdef/#ifndef`に対応した糖衣構文`#elifdef/#elifndef`が追加されます。

```cpp
#ifdef M1
...
#elif defined(M2)
...
#elif !defined(M3)
...
#endif

// ↑が↓こう書ける

#ifdef M1
...
#elifdef(M2)
...
#elifndef(M3)
...
#endif
```

これらのプリプロセッシングディレクティブはまた、C23でも利用可能になっています。

## #warning のサポートを追加

- P2437R1 Support for #warning(https://wg21.link/P2437R1 )

プリプロセス時に警告を表示させるディレクティブである`#warning`は非標準ながらもほとんど同じセマンティクスの下で主要なコンパイラで実装されていました。C++23ではこれを標準化することで、合法的にポータブルに使用することができます。

\clearpage

```cpp
// コンセプトの利用可否をチェック
#if __has_include(<concepts>)
  #include <concpets>
#else
  // コンセプトが利用できない場合はSFINAEでエミュレートすることを通知
  #warning Emulate with SFINAE because <concepts> does not exist

  #include <type_traits>
  ...
#endif
```

`#warning`は構文的には`#error`と全く同様に利用できますが、`#warning`は実行されてもコンパイルが停止せず警告が表示されるのみです。

なおこれはC23でも利用可能になっています。

## 汎用的なソースコードのエンコーディングとしてUTF-8をサポート

- P2295R6 Support for UTF-8 as a portable source file encoding  
  (https://wg21.link/P2295R6 )

C++ソースファイルの文字エンコーディングについては規格は単に実装定義としており、どのエンコーディングをサポートするべきともすべきでないとも指定していませんでした。このため、複数のコンパイラ間で1つのソースファイルを共有したい場合に、完全にポータブルなエンコーディングというのは規格的には存在しません。

しかし実際には、主要なコンパイラは全てUTF-8は少なくともサポートしており、ライブラリのソースコードとしてはUTF-8を使用するのが一般的になっていました。

C++23からは、規格に適合するコンパイラはソースファイルの文字エンコーディングとしてUTF-8をサポートしなければならないとされ、UTF-8で書かれたソースファイルのポータビリティが保証されるようになります。

\clearpage

# その他

## decay-copyキャスト

- P0849R8 `auto(x)`: decay-copy in the language(https://wg21.link/P0849R8）

ある値を即席でコピーしたい場合はたまにあります。その場合元の型名で新しく変数を宣言してコピーしたい値で初期化するか、あるいは`auto`変数で同様の事をするのが簡単です。

```cpp
template<typename T>
void f(T v) {
  auto copy{v}; // コピーする
  T copy{v};    // こうでもいい
}
```

しかし、コピー後の値をすぐに他の関数に渡したい場合など、コピー用の変数をわざわざ宣言したくはない場合もあります。その場合、型名が分かっていればその型名でキャストすることで即席コピーを式として記述することができます。

```cpp
void other(auto&&);

template<typename T>
void f(T v) {
  other(T(v));  // コピーして渡す
}
```

しかし、この場合に型名が分かっていなかったらどうすればいいのでしょう。例えばこの`v`のメンバ関数の返す値をコピーしたい場合や、テンプレートパラメータが`T&&`で宣言されていることで`T`が素の型名を表さない場合などが考えられます。

```cpp
void other(auto&&);

template<typename T>
void f(T&& v) {
  // コピーして渡したい、が・・・
  other(T{v});      // Tが参照型の場合がある
  other(v.front()); // front()の戻り値型はTと異なる
}
```

このような場合にはコピー対象の値の型を`decltype()`で取得したうえで、それを`decay`（`remove_cvref`）することで素の型を取得するしかありません。

```cpp
void other(auto&&);

template<typename T>
void f(T&& v) {
  using U1 = std::remove_cvref_t<T>;
  other(U1{v}); // コピーして渡す

  using U2 = std::decay_t<decltype(v.front())>;
  other(U2(v.front())); // コピーして渡す
}
```

C++23ではこの即席コピーを行う新しい形式のキャスト式として、`auto(x)`の形のキャストが利用できるようになります。

```cpp
void other(auto&&);

template<typename T>
void f(T&& v) {
  // 共に、コピーした結果を渡す
  other(auot{v});
  other(auto(v.front()));
}
```

この効果は式の型`T`に対して`T(expr)`の形のキャストと同じです。

`auto(x)`形式のキャストにおいては`auto t(x);`の様に変数を宣言した場合における型`T`が推論され、`auot(x)`の`auto`がその型で置き換えられて`T(x)`のキャストとして実行されています。

この場合の推論で得られるような型（元の型から参照・CV修飾を取り除く+配列型・関数型はポインタ型へ変換）の事をdecayされた型、あるいはこのような型の調整の事をdecayと呼び、先程からやろうとしていた`T(x)`のようなコピーの事をdecay-copyと呼びます。

`auot(x)`のキャスト式はその推論された型`T`に対して`T(x)`と等価であり、これらの式の結果は`T`のprvalueとなります。入力の値`x`が左辺値であればキャスト結果は`x`をコピーしたprvalueとなり、`x`が右辺値の場合はムーブされ、さらに`x`がprvalueならばコピー省略によって`auto`によるキャスト式そのものではコピーもムーブも起こりません。

```cpp
auto f() -> std::string;
auto g() -> const std::string&;

int main() {
  std::vector vec = {1, 2, 3, 4};

  auto v1 = auto(vec);  // ok、コピー
  auto v2 = auto(std::move(vec)); // ok、ムーブ
  auto s1 = auto(f());  // ok、コピー省略
  auto s2 = auto(g());  // ok、コピー
}
```

decay-copyというかdecayにおいては、配列型と関数型は対応するポインタ型へ変換されるため、`auto(x)`も`x`がそのどちらかの型の値の場合は単純なコピーにはならないことに注意が必要です。

```cpp
auto f(int) -> std::size_t;

int main() {
  int a[5]{};

  // 配列型、関数型の場合は対応するポインタ型のprvalueへキャスト
  auto* p1 = auto(a);
  auto* p2 = auto(f);
}
```

なお、`auto(x)`と`auto{x}`はどちらも同じ意味になります。

```cpp
template<typename T>
void f(T x) {
  // この4つは同じ意味
  auto(x);
  T(x);
  auto{x};
  T{x};
}
```

この`auto`キャストが有用なのは、先述のようにコピー対象の素の型`T`が素直に得られず一時変数を作りたくない場合においてです。また、コピー対象の型名が長かったりする場合の短縮構文としても有効です。さらには、少し長くなったとしても`T(x)`の形よりも`auto(x)`の方が専用構文として一貫性が高く、decay-copyをするという意図が明確になります。

```cpp
class very_long_name_my_class {
  ...
};

auto f(const auto&) {
  ...
}

int main() {
  very_long_name_my_class v{};
  int n = 10;
  long double l = 1.0;

  // vをコピーしてfに渡したい場合
  f(very_long_name_my_class(v));  // ok
  f(int(n));          // ok
  f(long double(l));  // ng

  f(auto(v)); // ok
  f(auto(n)); // ok
  f(auto(l)); // ok
}
```

### コンセプトにおける利用について

ある型のdecayされた型を取る場合には`decay_t`や`remove_cvref_t`等のメタ関数を使用することになりますが、これは書いてみると結構長く、ただでさえ複雑になりがちなテンプレートのコードの可読性を低下させます。このような場合に`auto`キャストを使用すると`decltype(auto(x))`でそれが取れるので、（短くはなりませんが）少しだけ見やすくなります。

さらには、このことは型が推論されるコンセプトの文脈においてより有用となります。

例えば、ある関数が`bool`型を返すことをチェックしたい場合で、`bool`そのものでなくてもよい（`const bool`や`bool&`でもよい）ものの`bool`に変換可能な型は弾きたい、という場合に`requires`式の戻り値型制約を使用して正しく記述することはできず、`std::same_as<std::remove_cvref_t<std::invoke_result_t<F>>, bool>;`のような制約式を書く必要があります。

この場合に`auto`キャストを用いるとその意図を簡潔に表現できるようになります。

```cpp
// Fは引数なしで呼び出し可能であり、bool型を返してほしい
template<typename F>
concept returning_bool = 
  std::invocable<F> and
  requires(F& f) {
    {f()} -> std::convertible_to<bool>; // bool型を返すという意味になっていない
    {f()} -> std::same_as<bool>;        // bool&などが弾かれる
    {auto(f())} -> std::same_as<bool>;  // bool+修飾な型まで受け入れる
  };
```

この戻り値型制約（およびコンセプトの第一引数の自動補完が働く他の場所）においては、コンセプトの第一引数には`decltype((expr))`が充てられます。例えば上記の`requires`内1つ目の制約の場合、`std::convertible_to<decltype((f())), bool>`という制約がチェックされますが、`decltype((expr))`で取得していることによって`expr`の値カテゴリの情報が含まれることになります。

この場合にその修飾情報を無視した素の型をチェックするのは意外に難しく（`decay_t`したり`remove_cvref_t`したりする必要がある）、戻り値型制約の利用は諦めざるを得なくなりますが、`auto(x)`を使用することで型の`decay`（`std::remove_cvref_t`）を式としてスマートに記述することができるようになります。

ただし、`auto(x)`によって戻り値のコピーが入る場合があるので若干制約の意味は異なることになるのでその点に注意が必要ではあります。

## size_tリテラル

- P0330R8 Literal Suffix for (signed) size_t(https://wg21.link/P0330R8 )

C++17まで、`unsigned long long`等を得るためのリテラルは存在していましたが、`std::size_t`を直接得るものはありませんでした。C++23からは、新しい整数リテラルのサフィックスとして`z`が追加され、`uz`のように指定することで`std::size_t`型の整数リテラルを使用できるようになります。

```cpp
// size_tが64bit符号なし整数型である場合
std::signed_itegral auto n1 = 20z;   // ok、int64_t相当の型
std::signed_itegral auto n2 = 20Z;   // ok、同上
std::same_as<size_t> auto n3 = 23uz  // ok、size_t
std::same_as<size_t> auto n4 = 20UZ; // ok、同上
```

`z, Z`リテラルは整数にのみ使用可能で、単体で指定すると`std::size_t`に対応する符号付整数型の値を返し、`u, U`とともに指定すると`std::size_t`型の値を返します。なお、`u, U`と`z, Z`の組み合わせは自由であり、大文字小文字や順番の制限はありません。

```cpp
const std::size_t size = ...;

auto clamp1 = std::max(0, std::min(size, 100));     // ng
auto clamp2 = std::max(0uz, std::min(size, 100uz)); // ok
```

`std::max/min`は2つの引数が同じ型であることを要求するため`clamp1`はエラーになります（`std::min(size, 100)`は`int`と`std::size_t`を渡している）が、`uz`リテラルを使用することで整数リテラルの型をすべて`std::size_t`で揃えることができるようになります。

## 暗黙ムーブを簡略化

- P2266R3 Simpler implicit move(https://wg21.link/P2266R3 )

C++11から始まってC++20まで、関数ローカルなものを返す場合の暗黙ムーブが強化されてきており、C++23でも引き続き暗黙ムーブの発動タイミングが拡大されます

```cpp
// 例示用のムーブ可能な型
struct Widget {
  Widget(Widget&&);
};

auto example1(Widget&& w) -> Widget&& {
  // 戻り値型が右辺値参照型の場合のローカル右辺値参照の暗黙ムーブ
  return w;  // ok、C++23から
}

// 右辺値修飾変換演算子を持つ型
struct Jeff {
  operator int&() &&;
};

auto example2(Jeff x) -> int& {
  // 戻り値型が参照型の場合のローカル変数の暗黙ムーブ
  return x;  // ok、C++23から
}
```

C++20までの暗黙ムーブはあくまで戻り値型がオブジェクト型（非参照型）の場合にのみ行われていましたが、C++23ではその制限が取り払われます。そして同時に、暗黙ムーブの仕様がシンプルにまとめられます。

暗黙ムーブではまず、暗黙ムーブ可能なもの（*implicitly movable entity*）を次のように識別します

- 自動記憶域期間の非`volatile`オブジェクト型変数
- 自動記憶域期間の非`volatile`オブジェクト型への右辺値参照

自動記憶域期間というのは要するに関数ローカルスコープのことです。`volatile`変数以外にこの対象に含まれないのは左辺値参照です。

そ次に、`return`文において次の条件を満たす場合に、そのオペランドのid式（変数名を指定する式）はムーブする資格がある（*move-eligible*）とします

- オペランドはid式であり
    - `()`で囲まれていても良い
- そのid式は、その文を囲む最も内側の関数（ラムダ式）の本体内もしくは関数引数宣言内の、暗黙ムーブ可能なものを指定している

id式とは変数名のみからなる式の事で、変数名そのものの事です。

そして最後に、ムーブする資格のあるid式の値カテゴリは*xvalue*となります。

*xvalue*とはすなわち左辺値を`std::move()`した値カテゴリの事で、これによって条件を満たす場合に`return`文のオペランドが自動でムーブされたかのように扱われるわけです。

C++20までの仕様では、`return`文において暗黙ムーブ可能なものがコピーされるときに二段階のオーバーロード解決によってムーブできるならしてできないならコピー、のようにすることで暗黙ムーブを行っていました。このため、コピーが起こらない場合（戻り値が参照型の場合）は対象外となっており、二段階のオーバーロード解決は処理として複雑になるため正しく実装されない（振る舞いがコンパイラによって異なる）などの問題を引き起こしていました。

C++23ではその指定をこのようにかなり単純化することで暗黙ムーブ機会を拡大するとともに、実装の複雑さを低減しています。

### 副作用

ただ、この暗黙ムーブ仕様の単純化はいくつか副作用を伴っています。

C++23の暗黙ムーブ仕様では、非`volatile`ローカル変数をそのまま`return`しようとすれば必ず暗黙ムーブされるわけですが、この時に戻り値型が左辺値参照型だとコンパイルエラーになります。これにより、ローカル変数を左辺値参照として返せなくなります。

```cpp
struct W {};

auto f() -> W& {
  W w{};

  return w; // C++20まではok、C++23からはng
}

// 参照先の型によらない
auto g() -> int& {
  int n = 0;

  return n; // ng、C++23から
}
```

これは暗黙ムーブによって右辺値参照（`T&&`）を左辺値参照（`T&`）として返そうとすることからエラーになっています。ただし、このこととは逆に、右辺値参照を返す関数において関数ローカル変数をそのまま`return`した場合にコンパイルが通るようになってしまっています。

```cpp
auto f() -> W&& {
  W w{};

  return w; // C++20まではng、C++23からはok
}
```

さらに、戻り値型推論の結果がC++20までと変わる場合があります。 

```cpp
decltype(auto) f(int n) {
  return (n);
}
// C++20までは戻り値型はint&
// C++23からは戻り値型はint&&

auto&& g(int n) {
  return n;
}
// C++20までは戻り値型はint&
// C++23からは戻り値型はint&&
```

ムーブする資格のある式の判定では、変数名そのものか変数名を`()`で括った式が対象となります。`decltype(auto)`は変数名を`()`で括った場合はその式の値カテゴリを型情報に含めるため、変数名が左辺値ならば左辺値参照型（`T&`）が推論されます。

しかし、C++23では`return`文のオペランドがムーブする資格のある式ならばその値カテゴリは*xvalue*となるため、`decltype(auto)`の推論結果は右辺値参照型（`T&&`）になります。

同様の理由によって、`auto&&`の場合も`return`文のオペランドがムーブする資格のある式だと右辺値参照型（`T&&`）に推論されます。

型名を`T`（`T&&`は右辺値参照型）として、戻り値型推論とコンパイル可否の変化は次のようになります

|関数宣言と`return`文|C++20|C++23|戻り値|
|---|:-:|:-:|---|
|`decltype(x) f(T x) { return x; }`       |`T`：〇| ― ||
|`decltype((x)) f(T x) { return (x); }`   |`T&`：〇|`T&`：`❌`||
|`decltype(auto) f(T x) { return x; }`    |`T`：〇| ― ||
|`decltype(auto) f(T x) { return (x); }`  |`T&`：〇|`T&&`：〇|ローカル参照|
|`decltype(x) f(T&& x) { return x; }`     |`T&&`：×|`T&&`：`⭕`||
|`decltype((x)) f(T&& x) { return (x); }` |`T&`：〇|`T&`：`❌`||
|`decltype(auto) f(T&& x) { return x; }`  |`T&&`：×|`T&&`：`⭕`||
|`decltype(auto) f(T&& x) { return (x); }`|`T&`：〇|`T&&`：〇||
|`auto&& f(T x) { return x; }`       |`T&`：〇|`T&&`：〇|ローカル参照|
|`auto&& f(T x) { return (x); }`   |`T&`：〇|`T&&`：〇|ローカル参照|
|`auto&& f(T&& x) { return x; }`       |`T&`：〇|`T&&`：〇||
|`auto&& f(T&& x) { return (x); }`   |`T&`：〇|`T&&`：〇||

（関数宣言が見づらいことをお許しください…）

表の中2列の各項目内は、推論される戻り値型:コンパイル可否、のように記述しています。表中の記号の意味は次のようになります

- 〇 : ok
- × : ng
- ― : 変化なし

`auto&`の場合は、`auto&&`のケースで戻り値型指定だけを変えたコードで考えることができて、全てエラーになります（以前はすべてエラーにならなかった）。これは前述のように、`return`文オペランドが暗黙ムーブされて`T&&`になるものの、`auto&`は左辺値参照にしか推論されないためです。

この暗黙ムーブ仕様に関してはもう少し詳しい説明が拙著「C++ return文で起こること」で書かれています。

## コード内容の仮定をコンパイラに伝える`assume`属性

- P1774R8 Portable assumptions(https://wg21.link/P1774R8 )

コードを書くとき、書いているコードについて特定の仮定が成立する事を知っていることがよくあり、そのような情報をコンパイラに伝えることができれば、コンパイラの最適化を促進できる場合があります。そして、主要なC++コンパイラはその手段を提供しています

- Clang : `__builtin_assume(expr)`
- GCC : `if (expr) {} else { __builtin_unreachable(); }`
- MSVC/ICC : `__assume(expr)`

```cpp
int divide_by_32(int x) {
  __builtin_assume(x >= 0); // 仮定を伝える
  return x / 32;
}
```

この例では、コンパイラは通常符号付整数の可能な全ての入力で正しく動作するコードを出力しますが、`__builtin_assume(x >= 0)`によって`x`が負およびゼロのケースを考慮しなくても良いことがわかるため、コンパイラは正の場合のみで正しく動作するコード（5ビット右シフト）を出力します。

上に書いた3実装における構文がバラバラであることから分かるように、このような機能は性能を追求するプログラムで有用だった一方で、コンパイラによって構文もセマンティクスも異なっていたことからポータブルなコードで使用することが困難でした。

C++23からは`[[assume(expr)]]`属性が追加され、これらのコンパイラの独自拡張を使用せずにコードの仮定をコンパイラに伝達することができるようになります。

```cpp
int divide_by_32(int x) {
  [[assume(x >= 0)]]; // 仮定を伝える
  return x / 32;
}
```

`[[assume(expr)]]`属性は空のステートメント（すなわち`;`の前）に対してのみ指定でき、かつ関数内部でのみ使用できます。`[[assume(expr)]]`の`expr`は評価されないオペランドであり副作用を持つ式を指定することもできますが、実行されません。そして、`expr == false`となるような入力に対しては未定義動作となるため、この仮定が満たされるようにするのはプログラマの責任となります。

また、`[[assume(expr)]]`の`expr`に指定可能なのは`conditional-expression`と呼ばれる構文要素であり、ここに指定可能な式とは条件演算子と条件演算子よりも優先順位が高い演算子のみからなる式です。例えば、トップレベルのカンマ演算子や、複合代入演算子、`throw/co_yield`式は使用できません。

```cpp
[[assume(expr1, expr2)]];  // ng
[[assume(x = 1)]];      // ng
[[assume(x += 1)]];     // ng
[[assume(throw x)]];    // ng
[[assume(co_yield x)]]; // ng
```

一応、どうしてもこれらの式を書きたい場合は式全体をもう一つかっこでくくる（`(expr)`）ことで書くことはできます（こうすると、構文的には`primary-expression`と呼ばれる構文要素になり、これは`conditional-expression`に現れることができます）。

最後に、C++のコンパイラは知らない属性を無視することができますが、`[[assume(expr)]]`が無視される場合は完全に無かったかのように振舞うのではなく、`expr`の式に現れているものはコードとして参照されたことにはなります。例えば、`expr`の式内から関数テンプレートが呼ばれていた場合、その関数テンプレートのインスタンス化が行われます。

## 初期化文での型の別名宣言を許可

- P2360R0: Extend init-statement to allow alias-declaration  
  (https://wg21.link/P2360R0 )

C++17で`if switch`、C++20で範囲`for`の構文が拡張され、それぞれの内部に初期化文（*init-statement*）を書けるようになりました。しかし、そこには通常の変数の初期化宣言の他に`typedef`宣言も書くことができますが、なぜか`using`は書けません。

C++23では、`using/typedef`の一貫性を向上させるために、それらの初期化文内で`using`によるエイリアス宣言を書けるようにする提案です。

```cpp
// C++20からok
for (typedef int T; T e : v) { ... }

// C++23からok
for (using T = int; T e : v) { ... }
```

これができなかったのは単に文法の指定漏れだったようで、これはそれを修正するものです。おそらくこう書きたいという事はあまりないでしょう…。

## 範囲for文が範囲初期化子内で生じた一時オブジェクトを延命することを規定

- P2718R0 Wording for P2644R1 Fix for Range-based for Loop  
  (https://wg21.link/P2718R0 )
- P2644R0 Final Fix of Broken Range‐based for Loop,Rev 0  
  (https://wg21.link/P2644R0 )

範囲`for`には長らく、オブジェクト生存期間にまつわる気づきにくいバグが存在していることが知られていました。

```cpp
// どこかで定義されているとして
auto f() -> std::vector<std::string>;

int main() {
  // ok
  for (auto&& str : f()) {
    std::cout << str << '\n';
  }

  // UB 💀
  for (auto&& c : f().at(0)) {
    std::cout << c << ' ';
  }
}
```

`f()`は`std::string`を要素に持つ`std::vector`の値（右辺値）を返す関数です。その戻り値は一時オブジェクトであるので、値として受けるか`auto&&`で受けるなどして寿命を延長する必要があります。範囲`for`文でもそれは行われるので、最初の`for`文は問題ありません。

ところが、2つ目の`for`文は`f()`の戻り値からその要素を引き出しています。ここで問題なのは要素数が不明なことではなく、`f().at()`の戻り値は*lvalue*（`std::string&`）であり、範囲`for`はこの結果のオブジェクトだけを保存してループを廻してくれます。その結果、`f()`の直接の戻り値は`f().at(0)`の後で捨てられ、当然ここから取得した`std::string&`の参照はダングリング参照となります。そして、ダングリング参照のあらゆる利用は未定義動作です。

これを回避するためには、このような関数の直接の戻り値を初期化宣言で受けておくなどしなければなりません。

```cpp
int main() {
  // ok
  for (auto tmp = f(); auto&& c : tmp.at(0)) {
    std::cout << c << ' ';
  }
}
```

この問題は範囲`for`という簡易な構文に覆い隠されていることによってその問題の深刻さをいや増しています。範囲`for`の仕様をよく知らない場合はこの問題に気付くことはできないでしょうし、この問題を知っていても問題の起こる場所が範囲`for`に隠されていることによって気づき辛くなります。

C++23ではこの問題が解決され、範囲`for`のイテレート対象の初期化の式内で作成されたすべての一時オブジェクトの寿命は範囲`for`文の完了（ループ終了）まで延長されるようになります。これにより、以前にUBだったコードはそのままコンパイラの言語バージョンを上げるだけで安全になります。

```cpp
// どこかで定義されているとして
auto f() -> std::vector<std::string>;

int main() {
  // ok
  for (auto&& str : f()) {
    std::cout << str << '\n';
  }

  // ok、C++23から
  for (auto&& c : f().at(0)) {
    std::cout << c << ' ';
  }
}
```

### UBが起きていた理由

範囲`for`文はシンタックスシュガーであり、その実態は通常の`for`文によるコードへ展開される形で実行されます。

例えば、規格においては範囲`for`の構文は次のように規定されています

```
for ( init-statement(opt) for-range-declaration : for-range-initializer ) statement
```

`init-statement`は`for`の初期化宣言で`(opt)`は省略可能であることを表します。

`for-range-declaration`は`for(auto&& v : rng)`の`auto&& v`の部分であり、`for-range-initializer`は`rng`の部分です。残った`statement`は`for`文の本体です。

そして、これは次のように展開されて実行されます

```
{
  init-statement(opt)

  auto &&range = for-range-initializer ;  // イテレート対象オブジェクトの保持
  auto begin = begin-expr ; // std::begin(range)相当
  auto end = end-expr ;     // std::end(range)相当
  for ( ; begin != end; ++begin ) {
    for-range-declaration = * begin ;
    statement
  }
}
```

つまり、イテレータを使った通常の`for`ループに書き換えているわけです。そして、問題は展開後ブロック内の3行目にあります。

```cpp
auto &&range = for-range-initializer ;
```

この式では、`auto&&`で範囲`for`のイテレート対象オブジェクトを受けており、これによって左辺値も右辺値も同じ構文で受けられ、なおかつ右辺値に対しては寿命延長がなされます。ここに先程の`for`文から実際の式をあてはめてみてみましょう。

```cpp
// 1つ目のforから
auto &&range = f() ;  // ok

// 2つ目のforから
auto &&range = f().at(0) ;  // UB 💀
```

2つ目の初期化式の何が問題なのかというと、変数`range`に受けられているのは`f().at(0)`の戻り値（`std::string&`）であって、`f()`の直接の戻り値であり`.at(0)`で取り出した`std::string`の本体を所有するオブジェクト（`std::vector<std::string>`）はどこにも受けられていないからです。

このような一時オブジェクトの寿命はこの初期化文の終わりの`;`で終了し、破棄されます。すなわち、この2つ目の初期化式では`f()`の戻り値の寿命はこの行で尽き、そこから取り出されたすべての参照はダングリング参照となります。

C++23からは、ちょうどここの`for-range-initializer`で生じた一時オブジェクトの寿命が、このような展開後の一番外側のブロックの終了まで延長されることでループの間安全に利用することができるようになっています。

## 拡張浮動小数点数型のサポートの追加

- P1467R9 Extended floating-point types and standard names  
  (https://wg21.link/P1467R9 )

機械学習の分野では、計算の高速化やパラメータ容量の削減などのために32bit浮動小数点数（`float`）よりもさらに幅の小さい浮動小数点数型を使用することが良く行われています。そのような拡張浮動小数点数型が利用できる環境とはNVIDIA GPU（CUDA）が最たる例ですが、CPUでも命令レベルでネイティブサポートされつつあります（AVX512 FP16やARMv8系のFP16サポートなど）。

そのような拡張浮動小数点数型が求められる場所ではC++が使用されることが多いのですが、C++では浮動小数点数型は組み込みの3種類（`float, double, long double`）だけしかサポートされていません。したがって、そのような場所ではクラス型を使って拡張浮動小数点数型を定義することになりますが、これはほぼ言語サポートが無いため実装の手間がかかり、効率化するのに手間がかかります。

そのような需要に対応するために、C++23からは拡張浮動小数点数型が言語とライブラリの両面からサポートされるようになります。ただし、`float`や`double`のようにキーワード付きの型名が利用できるようになるのではなく、`<stdfloat>`ヘッダで型エイリアスとして条件付きで宣言されて利用できるようになります。

```{style=cppstddecl}
// <stdfloat>での拡張浮動小数点数型の宣言例
namespace std {
  #if defined(__STDCPP_FLOAT16_T__)
    using float16_t  = implementation-defined;
  #endif
  #if defined(__STDCPP_FLOAT32_T__)
    using float32_t  = implementation-defined;
  #endif
  #if defined(__STDCPP_FLOAT64_T__)
    using float64_t  = implementation-defined;
  #endif
  #if defined(__STDCPP_FLOAT128_T__)
    using float128_t = implementation-defined;
  #endif
  #if defined(__STDCPP_BFLOAT16_T__)
    using bfloat16_t = implementation-defined;
  #endif
}
```

これらの型名は必ずしも使用できるわけではなく、その実装でサポートされている場合にのみ宣言されています。その環境でこれらの拡張浮動小数点数型がサポートされているかを調べるには`__STDCPP_FLOAT16_T__`等のマクロが定義されているかを調べることで判定できます（命名は異なりますが、これは機能テストマクロです）。そして、サポートされている場合はそれぞれの拡張浮動小数点数値を生成するために`1.1f16`や`3.14f32`のようなリテラルサフィックスを利用できます。

|型名|機能テストマクロ|リテラル|通称|
|---|---|---|---|
|`std::float16_t`|`__STDCPP_FLOAT16_T__`|`f16 F16`|fp16/半精度|
|`std::float32_t`|`__STDCPP_FLOAT32_T__`|`f32 F32`|float|
|`std::float64_t`|`__STDCPP_FLOAT64_T__`|`f64 F64`|double|
|`std::float128_t`|`__STDCPP_FLOAT128_T__`|`f128 F128`|四倍精度|
|`std::bfloat16_t`|`__STDCPP_BFLOAT16_T__`|`bf16 BF16`|bfloat16/bf16|

```cpp
// 全てサポートされているとして 
#include <stdfloat>

int main() {
  std::float16_t fv0 = 1.0;    // ng
  std::float16_t fv1 = 1.0f16; // ok

  std::bfloat16_t fv2 = 2.0;      // ng
  std::bfloat16_t fv3 = 2.0bf16;  // ok
  
  std::float32_t fv4 = 3.0;     // ng
  std::float32_t fv5 = 3.0f32;  // ok
  
  std::float64_t fv6 = 4.0;     // ok
  std::float64_t fv7 = 4.0f64;  // ok
  
  std::float128_t fv8 = 5.0;     // ok
  std::float128_t fv9 = 5.0f128;  // ok
}
```

通常の小数リテラル（`double`値）からの変換は、その精度が劣化しないものに限定されて許可されます。そのため、拡張浮動小数点数型の値の初期化には基本的に対応するリテラルサフィックスを使用すると良いでしょう。これらのリテラルサフィックスはユーザー定義リテラルではなく言語組み込みのものであるため、サポートされていればそのまま使用できます。

さらに、浮動小数点数に関連するライブラリ機能のサポートも追加されます

- `<charconv>`
    - `std::to_chars`/`std::from_chars`に拡張浮動小数点数型を取るオーバーロードを追加
- `<ostream>`/`<istream>`
    - `<<`による出力、`>>`による入力など
- `<cmath>`
    - 全ての数学関数に拡張浮動小数点数型を取るオーバーロードを追加
- `<complex>`
    - `std::complex`での利用許可
- `<atomic>`
    - `std::atomic`/`std::atomic_ref`の要素型としての使用許可
- `<numbers>`
    - 全ての数学定数の特殊化が提供される
- `<concepts>`
    - `std::floating_point`および`std::is_floating_point`で判定可能

拡張浮動小数点数型の実体はビット幅が合っていれば自由というわけではなく、`floatN_t`の浮動小数点数型についてはISO/IEC/IEEE 60559:2020（IEEE754）で定義されている`binaryN`の形式に沿ったものであることが要求されています。`bfloat16_t`についてはIEE754では定義されていませんが、C++の規格においてはその特性が厳密に指定されているため、bfloat16以外の表現にはなりません。

\clearpage

指定されている各浮動小数点数型の特性をまとめると次のようになります

|型名|ビット幅|仮数ビット|指数ビット|指数最大値|
|---|---|---|---|---|
|`std::float16_t`|16|10+1|5|15|
|`std::float32_t`|32|23+1|8|127|
|`std::float64_t`|64|52+1|11|1023|
|`std::float128_t`|128|112+1|15|16383|
|`std::bfloat16_t`|16|7+1|8|127|

左右端の列を除いて数値の単位はビットです。全ての浮動小数点数表現においては先頭ビットは符号ビットであり、仮数部ビット数の+1は所謂ケチ表現の分です。

これを見れば明らかかもしれませんが、多くの環境では`float/double`と`std::float32_t/std::float64_t`は同じものを表現する型になります。しかし、これらは型としては別のものであり`std::float32_t/std::float64_t`は`float/double`のエイリアスとして定義されません。

```cpp
// 別の型なのでオーバーロードが成立する
void f(double);         // (1)
void f(std::float64_t); // (2)

int main() {
  f(1.0);     // ok、(1)が呼ばれる
  f(1.0f64);  // ok、(2)が呼ばれる
}
```

### 変換ランク

拡張浮動小数点数型を含む浮動小数点数型の間には変換ランクというものが定義され、このランクを用いて変換可能性などが定義されます。

変換ランクは浮動小数点数型に対応する浮動小数点数表現が表現可能な値の集合の包含関係によって定義されるものの、標準の浮動小数点数型の表現が規定されていないことなどから非常に抽象的に規定されています。

`double, float`がIEEE754の倍精度と単精度の表現になると仮定し、`long double`の表現を知られている実装で区別しそれ以外を考慮しないとすると、次のような順位になります

\clearpage

| 変換ランク | サブランク | 型 |
|:-:|:-:|---|
|1|1|`std::float128_t`|
|1|2|`long double`（四倍精度）|
|2|-|`long double`（double-double）|
|2|-|`long double`（x87）|
|3|-|`long double`（倍精度）|
|4|1|`std::float64_t`|
|4|2|`double`|
|5|1|`std::float32_t`|
|5|2|`float`|
|6|-|`std::float16_t`|
|6|-|`std::bfloat16_t`|

変換ランクが同じ値でサブランクの指定が無く行が分かれている型は、それらの型で表現可能な値の集合の間で包含関係が成立しないことを表しており、これらの型の間では変換ランクによる順序が付きません。

同じ表に同居しているものの、同じ環境で`long double`が異なる表現を持つ状況はあり得ないため、表内での`long double`同士の比較（順位）には意味がありません。

同じ表現をもつ場合は拡張浮動小数点数型の方が標準の浮動小数点数型よりも変換サブランクが高くなり、このことは通常の算術変換で効いてきます。

### 暗黙変換

一方の型が拡張浮動小数点数型である場合の2つの浮動小数点数型の間の暗黙変換は、変換先の型の変換ランクが変換元の型の変換ランクと同じかより大きい場合に行われます（ここではサブランクは考慮しません）。これにより、暗黙変換はすべてロスレスであり値が失われることはありません。

値が失われる変換、すなわち縮小変換は一方の型が拡張浮動小数点数型である場合は常に暗黙的に行うことができません。拡張浮動小数点数型への/からの縮小変換は明示的な変換が必要です。なお、拡張浮動小数点数型が関与する場合は定数式においても縮小変換の暗黙変換は一律に禁止されます。

```cpp
// 小数リテラルからの変換はfloat64以上のみ可能
std::float128_t fv128 = 1.0; // ok
std::float64_t fv64 = 1.0;   // ok
std::float32_t fv32 = 1.0;   // ng
std::float16_t fv16 = 1.0;   // ng
std::bfloat16_t fvb16 = 1.0; // ng

// 拡張浮動小数点数リテラルからの初期化も同様
double d = 1.0f128; // ng
float f = 1.0f64;   // ng
```
```cpp
// float128_tの値は自身以外に対して縮小変換になる
const std::float128_t fv128 = 1.0f128;

// 全てng
std::float64_t fv64 = fv128;
std::float32_t fv32 = fv128;
std::float16_t fv16 = fv128;
std::bfloat16_t fvb16 = fv128;
double d = fv128;
float f = fv128;

// long doubleの表現がIEEE754四倍精度の場合のみok
long double ld = fv128;
```
```cpp
const std::float64_t fv64 = 1.0f64;

// 共にok
std::float128_t fv128 = fv64;
double d = fv64;

// 全てng
std::float32_t fv32 = fv64;
std::float16_t fv16 = fv64;
std::bfloat16_t fvb16 = fv64;
float f = fv64;
```
```cpp
// float16_tとbfloat_16_tは他の幅の型に暗黙変換可能
const std::float16_t fv16 = 1.0f16;

// 全てok（bfloat16の場合も同様
const std::float128_t fv128 = fv16;
std::float64_t fv64 = fv16;
std::float32_t fv32 = fv16;
double d = fv16;
float f = fv16;
```
```cpp
// bfloat16とfloat16は相互に暗黙変換不可能
std::bfloat16_t fvb16 = 1.0f16; // ng
std::float16_t fv16 = 1.0bf16;  // ng
```

コーナーケースとして、ロスレス変換であっても明示的な変換が必要となる場合があります。これは、2つの標準浮動小数点数型が同じ表現を持っている場合に、それらと同じ表現をもつ拡張浮動小数点数型が存在する場合の変換が該当します。

具体的かつ出会いそうな例は、`double`と`long double`がどちらもIEEE754の倍精度（`binary64`）の表現を持っている場合の`std::float64_t`との間の変換です。先程の変換ランクの表にあるように、`double`と`std::float64_t`は同ランクなのでその値は相互に暗黙変換可能ですが、`long double`は1つランクが高いので`std::float64_t`への変換はロスレスであるにも関わらず明示的変換を必要とします。

```cpp
// long doubleがdoubleと同表現の場合
const double d = 1.0;
const std::float64_t f = 2.0;
const long double ld = 3.0;

// doubleとfloat64_tは相互暗黙変換可能
double v1 = f;          // ok
std::float64_t v2 = d;  // ok

// float64_tからlong doubleは暗黙変換可能
long double v3 = f;     // ok

// long doubleからfloat64_tは暗黙変換不可
std::float64_t v4 = ld; // ng

// 標準浮動小数点数型へは暗黙変換可能
double v5 = ld; // ok
```

他にも、`double`がIEE754単精度（`binary32`）で定義されている環境における`double/float`と`std::float32_t`の間で起こり得ます。

標準の浮動小数点数型は縮小変換が起こる場合でも暗黙変換可能ですがこの動作は修正されず、暗黙変換が制限されるのはあくまで拡張浮動小数点数型が変換に絡む場合のみです。さらに、拡張浮動小数点数型から整数型への縮小変換も制限が無く、暗黙的に行うことができてしまいます。

```cpp
const std::float128_t fp128 = 0.1f128;

int n1 = fp128;           // ok
unsigned char n2 = fp128; // ok
bool b = fp128;           // ok
```

### 通常の算術変換

通常の算術変換（Usual arithmetic conversions）とは、異なる算術型同士の組み込みの二項演算において結果の型を決定するために行われる変換であり、2つのオペランドのうちの片方はこの結果を受けて（暗黙的に）変換されます。

```cpp
float f = 1.0f;
double d = 1.0;

// 結果の型はdouble
// fはdoubleに変換される
std::same_as<double> auto r = d + f;
```

算術型の値同士の二項演算（`a @ b`）において、少なくとも片方のオペランド（実引数、ここでは`a, b`のどちらか）の型が浮動小数点数型（拡張浮動小数点数型を含む）の場合、次のような手順で変換が行われます

- `a, b`が同じ型であれば変換は行われない
- そうではない場合で、もう片方のオペランドの型が浮動小数点数型ではない場合
    - そのオペランドはもう片方と同じ浮動小数点数型に変換される
- そうではない場合（両方とも浮動小数点数型）で
    - 両方のオペランドの型の変換ランクが異なる場合
        - 変換ランクが低い方の型のオペランドが高い方の型に変換される
    - 両方のオペランドの型の変換ランクが等しい場合
        - サブランクが低い方の型のオペランドが高い方の型に変換される
- これら以外の場合はコンパイルエラー

```cpp
// 変換ランクによる決定
const float f = 1.0f;
const std::float16_t fv16 = 2.0f16;
const std::bfloat16_t fvb16 = 3.0bf16;

f + fv16;  // ok、fv16はfloatに変換される
f + fvb16; // ok、fvb16はfloatに変換される
fv16 + fvb16; // ng、相互に変換できない


// サブランクによる決定
const double d = 1.0;
const std::float64_t fv64 = 1.0f64;
const std::float32_t fv32 = 1.0f32;

d + fv64; // ok、dはfloat64_tに変換される
f + fv32: // ok、fはfloat32_tに変換される
```

表現が同じ場合は拡張浮動小数点数型の方が常にサブランクが高くなるため、2つの浮動小数点数型が混在する二項演算における通常の算術変換においては標準の浮動小数点数型は常に拡張浮動小数点数型に変換されます。

### 浮動小数点数型の昇格

可変長引数の関数呼び出しと`float`と`double`の混ざった四則演算等の極めて限定的な状況において、`float`->`double`へ型が暗黙的に昇格されます。これは厳密には変換とは異なる扱いとなっています。

```cpp
// Cの可変長引数では、floatはdoubleに昇格される
std::printf("%f\n", 1.0f);  // 1.0fはdoubleに昇格される
```

これはCのK&Rからの名残であり便利な挙動とはみなされていなかったため、拡張浮動小数点数型には適用されません。拡張浮動小数点数型においては型が変わる場合は全て何かしらの変換が行われ、かつ必要となります。

### オーバーロード順位

拡張浮動小数点数型についてのオーバーロード解決を考えなければいけないシーンとは、異なる算術型・浮動小数点数型を取るオーバーロードが定義されている関数に対して浮動小数点数を渡したとき、どのオーバーロードが選択されるかを考える場合です。

```cpp
void f(std::float32_t);
void f(std::float64_t);

f(std::float16_t(1.0));  // どちらの関数が呼ばれる？
```

特にこのように、渡そうとしている浮動小数点数値と同じ型を取るオーバーロード（ベストマッチするもの）が定義されていない場合に、どちらの関数が選択されるのか、あるいは呼び出しできないのか、を理解しなければなりません。

ベストマッチする候補が存在し一つしかない場合はそれが選択されるので難しいことはありません。問題となるのは、ベストマッチする候補がなく暗黙変換を通してオーバーロードを選択する場合です。

単純化のために、オーバーロードの候補は2つのみでありそれらは引数として算術型の値を1つだけ取るように定義されているものとし、少なくとも片方のオーバーロードは浮動小数点数型を取るようになっているとします。この場合に、引数として渡している浮動小数点数値の型を`FP1`、浮動小数点数型を取るオーバーロードの引数型を`FP2`、もう片方のオーバーロードの引数型（算術型）を`T3`とします。

この時、次の条件を満たす場合に`FP1 -> FP2`の変換が`FP1 -> T3`の変換よりもより優先され（より順位が高くなり）ます

- `FP1`の変換ランクが`FP2`と等しく、かつ
    - `T3`が浮動小数点数型ではない、もしくは
    - `T3`が浮動小数点数型であり
      - `T3`の変換ランクが`FP1`と等しくない、もしくは
      - `FP2`の変換サブランクが`T3`のサブランクよりも大きい

そして、この手順を`FP2`と`T3`を入れ替えて行った時に、`FP1 -> T3`の変換が`FP1 -> FP2`の変換よりも優先されない場合にオーバーロード間に順序が付き、`FP2`を取るオーバーロードが選択されます。もし仮にどちらの変換も互いに対して優先される場合は順位が曖昧となりオーバーロード解決に失敗します。

この手順は、浮動小数点数の値を保持する変換（丸めなどが生じないような変換）を優先し、そのような変換が可能な候補が2つ以上ある場合は、同じ変換ランク内で他の浮動小数点数型への変換を考慮します。変換ランクを下る変換は暗黙変換できないのでそもそも考慮されませんが、変換ランクを上る方向の変換もオーバーロード解決時には順位が下げられます（単なる暗黙変換扱いになり、この手順によって優先されない）。

```cpp
void f(std::float32_t); // (1)
void f(std::float64_t); // (2)
int f(long long);       // (3)

f(std::float16_t(1.0)); // 曖昧、ランクが同じ候補が無く、変換可能な候補が2つある
f(float(2.0));          // (1)が呼ばれる、変換ランクが同じ
f(double(3.0));         // (2)が呼ばれる、有効な候補が1つしかない
f(std::float32_t(4.0)); // (1)が呼ばれる、ベストマッチ


void g(std::float128_t);  // (4)
void g(std::float64_t);   // (5)

g(double(1.0)); // (5)が呼ばれる、変換ランクが同じ
```

前述のように浮動小数点数型から整数型への変換は暗黙的に行うことができる一方で、拡張浮動小数点数型のランクを下る方向の変換は暗黙的に行うことができず、浮動小数点数型のオーバーロード順位付け手順にもそれが反映されてランクを下る方向の変換には順位が付かないようになっています。そのため、整数型と浮動小数点数型の候補があるときに引数の拡張浮動小数点数型の変換ランクが浮動小数点数型の候補の変換ランクよりも高い場合、整数型の候補が選択されます。

```cpp
void f(std::float32_t); // (1)
void f(std::int32_t);   // (2)

f(std::float128_t(1.0));  // (2)が呼ばれる、(1)は明示的変換が必要になるため
f(std::float64_t(1.0));   // (2)が呼ばれる、同上
f(double(1.0));           // (2)が呼ばれる、同上
f(std::float32_t(1.0));   // (1)が呼ばれる、ベストマッチ
f(float(1.0));            // (1)が呼ばれる、変換ランクが同じ
f(std::float16_t(1.0));   // 曖昧、どちらに対しても暗黙変換可能
```

また、変換ランクを上る方向の変換は順位が下げられるものの通常の暗黙変換として扱われてオーバーロード解決自体は成功しますが、あくまで通常の暗黙変換でしかないので整数型のオーバーロードとの間で順位が付きません。とはいえこの場合はエラーになります（上記の最後の例）。

ここまでの説明とサンプルコードでは単純化して1引数かつ候補がせいぜい2つの場合を主に考えてきました。引数が2つ以上ある場合はそれぞれの引数毎にオーバーロード解決を行い、浮動小数点数型については上記のように優先順位が決定されます（最終的な結果は他の引数次第）。オーバーロード候補が3つ以上ある場合、まず2つの組でこの判定を行い、勝った候補と残った候補で再び判定を行い...のようにすることで同様に考えることができます。

### C23拡張浮動小数点数型

C++23とほぼ同時にC23でも拡張浮動小数点数型が追加されています。型名の対応は次のようになります

|C++23|C23|
|---|---|
|`std::float16_t`|`_Float16`|
|`std::float32_t`|`_Float32`|
|`std::float64_t`|`_Float64`|
|`std::float128_t`|`_Float128`|
| | `_FloatN` |
|`std::bfloat16_t`| |

C23の`_FloatN`型は128より大きく32で割り切れる数の`N`ビット幅の拡張浮動小数点数型で、実装オプションとして規定されています。また、Cではbfloat16に対応する型は用意されません。

Cの型名は全てエイリアスではなくキーワードになっています。なお、リテラルサフィックスは同じものが利用できます。

C23とC++23の拡張浮動小数点数型は互換を持つように意識して設計されており、C++の拡張浮動小数点数型については推奨事項としてCの拡張浮動小数点数型との互換性を持つようにすることが付記されています。それでもなお、大きな違いが2つあります

1. 型名
    - Cの拡張浮動小数点数型名はC++では規定されない
2. 暗黙変換
    - Cでは縮小変換は暗黙的に行うことができる
    - C++では暗黙的な縮小変換は許可されない

名前の共通化は少し手間になりますが、そこを乗り越えた後で両方の言語でコンパイルが通るコードは変換や実行時の振る舞いも含めて同じ動作をするため、そこまで大きな違いはないはずです。

型名の差異については、実装はCの型名をC++のエイリアス名の元にするように実装することでC++でもCの型名を利用できるように実装可能であり、実装はそうするはずなのでCの型名が使用できるはず、と提案では述べられています（一応、本書執筆時点で拡張浮動小数点数型を実装済みのGCCはそのように実装しています）。

```cpp
#include <stdfloat>
#include <concepts>

// パスすることが期待される（ただし保証はない）
// gcc（13~15）はパスする
static_assert(std::same_as<std::float16_t, _Float16>);
static_assert(std::same_as<std::float32_t, _Float32>);
static_assert(std::same_as<std::float64_t, _Float64>);
static_assert(std::same_as<std::float128_t, _Float128>);
```

\clearpage

# とてもマイナーな変更

## スコープと名前ルックアップの仕様整理

P1787R6: Declarations and where to find them(https://wg21.link/P1787R6 )

これは、規格書におけるC++の宣言や名前解決に関する規定をクリーンアップするものです。

用語の意味や使用法を一貫させ、既存規定の矛盾点を解消して曖昧さを低減することなどを目的としたものです。これによって、63個のコア言語イシューと21個の潜在的な問題が解決されるとのことです。これは特に、C++20のモジュールにおいての影響の大きさもあったようです。

この提案は何か新機能を提案したり既存の動作を変更したりするものではありませんが、とにかくその影響範囲が幅広いため、変更の量がとても大量です。

## 複合文の末尾へのラベルを許可

- P2324R2 Labels at the end of compound statements (C compatibility)(https://wg21.link/P2324R2 )

複合分（*compound statement*）と呼ばれる構文要素（おおよそブロックのこと）の内部にはラベルを置くことができますが、実はブロックの終端には置くことができませんでした。Cでもこれは同様だったようですがC23で修正されたことで可能になっており、C/C++の互換性向上のためにC++23でも同様に許可することになりました。

```cpp
void foo(void) {
first:  // C/C++共にok
  int x;

second: // C/C++共にok
  x = 1;

last:   // Cはok、C++はng、C++23でok
}
```

この修正はこの例の`last`のようなラベルを置くことができるようにするものです。これによって、ブロック内部ではラベルの位置に関する制限はなくなります。

## 参照するPOSIX規格を更新

- P2227R0 Update normative reference to POSIX(https://wg21.link/P2227R0 )

C++は標準ライブラリ機能の定義の中で現れるPOSIX機能のために、POSIX規格を参照しています。これまでは「ISO/IEC 9945:2003 (POSIX.1-2001 または、The Single UNIX Specification, version 3)」とよばれる2003年頃に策定されたものを参照していました。しかしこの規格は古いもので新しめの標準ライブラリ機能の中にはそこにはないPOSIX機能を参照するものがあったため、C++23では参照を更新し「ISO/IEC/IEEE 9945:2009 (POSIX.1-2008 aka SUSv4)」というより新しい規格を参照するようになります。

ユーザーサイドでは何の影響もない変更ですが、規格的に影響があるのは`errno`周りと`<filesystem>`の2つです。

### `errno`関連

C++11では`errno`に関して「`<cerrno>`の内容はPOSIXの`errno.h`と同じだが各エラー値はマクロとして定義しなければならない」の様に参照され、さらにエラー値のマクロがリストアップされていました。しかし、そのリストにはPOSIX.1-2008と呼ばれるより新しい規格で追加されたもの（`ENOTSUP`、`EOPNOTSUPP`、`ENOTRECOVERABLE`、`EOWNERDEAD`）を含んでいました。

### `<filesystem>`

`<filesystem>`の機能の多くの部分はPOSIXのファイルシステムAPIを使用して定義されています。そしてまず、`filesystem::path`クラスの規定から参照されているPOSIX規格の段落番号が異なっています。

参照されている関数のほとんどはPOSIX.1-2001にも存在するのですが、やはりこちらでも一部の関数（`futimens()`、`fchmodat()`）が存在していません。これによって、これを使用している`filesystem::last_write_time(), filesystem::permissions()`のセマンティクスは未定義になってしまっています。

POSIX規格への参照が更新されることによって、これらの機能が規格的にきちんと定義されることになります。

## 行末スペースを無視するよう規定

- P2223R2 Trimming whitespaces before line splicing  
  (https://wg21.link/P2223R2 )

CとC++では共に、行末にバックスラッシュ（`\`）がある場合にその行と続く行を結合して論理的に1行として扱います。これはプリプロセスより前に処理されるため、マクロを複数行で定義する時などに活用されます。

しかし、バックスラッシュと改行の間にホワイトスペースしかない場合の動作が実装定義とされ、実際にコンパイラ間で挙動が異なっていました。

```cpp
int main() {
  int i = 1  
  // \  
  + 42
  ;
  return i;
}
```

3行目のバックスラッシュの後にはスペースが挿入されています。この時、このバックスラッシュを行結合のマーカーとして扱うかどうかが実装によって異なっています。EDG(ICC),GCC,Clangはホワイトスペース列を除去して行結合を行い、結果1を返します（`+ 42`はコメントアウトされる）。MSVCはバックスラッシュの直後に改行が無いことから行結合とは見なさず、結果43を返します。

前述のように、これはどちらも規格的に正しい振る舞いで実装定義の範疇でしたが、この振る舞いは直感的ではありません。そのため、C++23ではバックスラッシュ後のホワイトスペース列は除去した上で行結合を行うことを規定します。すなわち、MSVCの振る舞いに修正が必要となります（この本の執筆時点ではまだ実装されていません）。

これはMSVC利用者にとっては破壊的変更となります。しかし、この挙動に依存しているコードはバックスラッシュの後の見えない空白を維持し続けているはずですが、それを確実に保証することはできずその有用性も無いため、バグであるとみなされます。例えば、コードフォーマッタは行末の空白を削除する場合が多くあります。

別の例

```cpp
// MSVCのみng
auto str = "\ 
";
```

これも最初の行のバックスラッシュの後に空白が1つあるため、MSVCは行結合を行わずエラーになります。

## ガベージコレクションサポートの削除

- P2186R2 Removing Garbage Collection Support(https://wg21.link/P2186R2 )

C++11ではC++処理系の実装としてガベージコレクション（GC）によるものを許可し、そのサポートのためのライブラリ機能を追加していました。そこから時間が経った現在において、C++によるGCの実装はいくつかある（例えばブラウザのJavascript実装）ものの、C++の実行環境そのものにGCを組み込んだような実装は存在しておらず、C++のGCサポートは一切利用されていませんでした。

さらに、C++11で追加されたそのGCサポートにはいくつも問題点が報告されており、実質的に使用できないものになっていました。利用されておらず役に立たなかったということで、C++23ではそれらの規定およびライブラリ機能を削除します。

ただしこれは、C++実装としてGCサポートを認めないわけではなく、少なくとも現在の仕様とライブラリ機能は役に立たないので削除するものです。将来的には別の形で補われるかもしれません（ただし現在のところそのような動きはありませんが）。

## 文字集合と文字エンコーディングに関する規定の整理

- P2314R0: Character sets and encodings(https://wg21.link/P2314R4 )

この提案は規格書におけるC++ソースコードのコンパイルのかなり早い段階（翻訳フェーズ1~4）においての、文字集合とエンコーディングに関する仕様を整理し、明確化するものです。また、文字列リテラルが使用されるコンテキストごとにどのエンコーディングで解釈すべきかを明確化しています。

この提案は基本的に既存動作を変更しませんが、翻訳フェーズ1においてユニコード文字のユニバーサル文字名への置き換えをしないようにして、全てのユニコード文字が翻訳フェーズの全体にわたって保持されるようになるため、プリプロセッサの文字列化演算子（`#`）の動作が変更されます。

```cpp
#define S(x) # x

// C++20まで
const char * s1 = S(Kﾃｶppe);      // "K\\u00f6ppe"
const char * s2 = S(K\u00f6ppe);  // "K\\u00f6ppe"

// C++23以降
const char * s1 = S(Kﾃｶppe);      // "Kﾃｶppe"
const char * s2 = S(K\u00f6ppe);  // "Kﾃｶppe"
```

ただ、既存のコンパイラでこのC++20の例のような動作をするものは無く、全てこの例の23以降と同じ動作をしていたため、実質的に影響はありません。

\clearpage

# Defect Report

## 無意味なexport宣言の禁止

- P2615R1: Meaningful exports(https://wg21.link/P2615R1 )

C++20のモジュール仕様においては、任意の宣言に対して`export`を最初に指定することでその宣言をそのモジュール（のインターフェース）からエクスポートするように宣言する（エクスポート宣言）ことができます。

ただ、このエクスポート宣言においてはいくつか無意味な宣言を許可していました

```cpp
// 明示的インスタンス化の中のexport宣言
template export void f();

// 明示的インスタンス化のexport宣言
export template void f();

// 部分的特殊化のexport宣言
// 本体の関数テンプレートがexportされているならこちらには不要
export template<> void g(int);

// 部分的特殊化の中のexport宣言
template<> export void g(int);

// 部分的特殊化のexport宣言
// プライマリテンプレートがexportされていれば不要
export template<class T> struct trait<T*>;
```

C++23でこのような無意味なエクスポート宣言がコンパイルエラーになるようにある程度制限されます。具体的には

- 明示的インスタンス化
- 明示的特殊化
- エクスポート宣言そのもの
- テンプレートの部分的特殊化

はエクスポート宣言できなくなります。また、明示的特殊化（部分的特殊化）の宣言および`extern`宣言内でのエクスポート宣言が禁止されます。これにより、先程の例は全てコンパイルエラーになるようになります。

ただし、ブロックのエクスポート宣言においてブロック内にこれらのものが存在していてもエラーにはならず無視されます。これは`export`ブロックの利便性向上のためです。

なお、ほぼついでですが、同時に`extern`宣言においても明示的インスタンス化などを含むことができないようにされています。

これはC++20に対する欠陥報告です。

## 識別子に使用可能な文字の制限

- P1949R7 C++ Identifier Syntax using Unicode Standard Annex 31  
  (https://wg21.link/P1949R7 )

ユニコードには視覚的に表示されない文字がいくつもあります（ゼロ幅空白、異体字セレクタ、結合文字、etc...）。このような文字がソースコード（特にクラス名や変数名）に使用されているとバグの原因にしかなりません。また、ユニコードには見た目が全くあるいはほとんど同じながらも文字（コードポイント）としては異なる文字も存在しており、これもバグの原因となります。

ClangやMSVCはC++20以前からソースコードのエンコーディングにUTF-8を使用でき、ソースコード上でも任意のユニコード文字を使用できていました。GCCもソースコードにUTF-8を使用できることは同じなのですが、バグにより識別子名として使用出来なかったため、C++ソースコードで任意のユニコード文字を使用することはポータブルではありませんでした。

しかし、GCCのそのバグはC++20策定以降に解消されたことでC++ソースコードにおいて任意のユニコード文字を使用することの障壁が低くなりました。それにより、前述の特殊な文字による影響は大きくなっていくことが予想されます。

そのため、C++23では識別子名（クラスや変数の名前）として使用可能な文字を制限することで、そののような問題を解消します。

使用可能な文字セットについてはユニコード規格のUnicode Standard Annex 31（UAX31）というものを参照しており、C++ではそこで規定されているDefault Identifiersと呼ばれるカテゴリを（若干カスタムして）採用しています。

UAX31はプログラミング言語で汎用的に識別子名として使用可能な文字についてまとめたものです。ここにどのような文字が含まれているかは非常に複雑なのですが、制御文字を覗いたASCII文字は少なくとも含まれています。一方、Default Identifiersには絵文字はほとんど含まれていません。従って、C++23においてはC++ソースコード中で何らかの名前に絵文字を使用することができなくなります。

これはC++のすべてのバージョンに対する欠陥報告であるため、これを実装したコンパイラでは以前の言語バージョンにおいてもこの制限が適用されるようになります。ソースコード中でユニコードの特殊な文字を何らかの名前に使用している場合、コンパイラのアップデート後にエラーになる可能性があります。

```cpp
// 以前はok、この提案以降ngな例

class 💩 : public std::exception {};

void f(💩 💩💩) {
  throw 💩💩;
}

void add() {
  const int 😱 = 2;
  const int 😇 = 3;
  const int 🥺 = 5;

  std::cout << 😱 * 😇 * 🥺;
}
```

C++11において規格としてユニコード対応がなされた際、C++のソースコードで使用可能なユニコード文字の範囲としてはUAX31のImmutable Identifier（当時はAlternative Identifier）を参照してユニコードのコードポイント範囲を指定する形で使用可能な文字を制限していました。多くの絵文字はここに入っていたのですが、記号とみなされる文字は除外されていたことから絵文字の中にも使えないものがありました。特に、女性の絵文字は全て使用できませんでした。

```cpp
bool 👷 = true;   // ok、男性建築作業員
bool 👷‍♀ = false;  // ng、女性建築作業員
```

女性の絵文字は、男性の絵文字に対して女性記号`♀`をゼロ幅接合子で結合することで表されており、女性記号はUAX31のImmutable Identifierから除外されていたためこのようなことが起こっていました。この提案の修正以降は、平等にどちらの絵文字も禁止されます。

## 属性指定の重複の許可

- P2156R1: Allow Duplicate Attributes(https://wg21.link/P2156R1 )

C++20まで、一つの属性指定`[[]]`の中で同じ属性が複数回現れることは出来ませんでしたが、属性指定を複数に分割すれば同じ属性が何回重複しても大丈夫でした。

```cpp
// ng、C++20
[[noreturn, carries_dependency, deprecated, noreturn]]
void f();

// ok
[[noreturn]] [[carries_dependency]] [[deprecated]] [[noreturn]]
void g();
```

C++23からはこの制限が無くなり、1つの属性指定内で同じ属性が重複して現れてもエラーにならなくなります。

```cpp
// ok
[[noreturn, carries_dependency, deprecated, noreturn]]
void f();
```

元々のこのような制限は、属性指定を分けて重複可能にすることでマクロによって属性を条件付きで追加していくことをサポートすることを意図したものだったようですが、一方で1つの属性指定の中でそれを行う事はレアケースだと考えられたためこの差が生じていたようです。ただ、これによって各属性それぞれの規定内で重複してはならないことを一々追加しなければならず、標準の規定を無駄に重くしており、しかもその作業を新しい属性を追加するたびに行わなければなりませんでした。

制限しておく必要性の低さとこのような標準化における手間の問題からこの制限は解消されることになりました。

これはC++11への欠陥報告です。

## C++20 機能テストマクロの更新

- P2493R0 Missing feature test macros for C++20 core papers  
  (https://wg21.link/P2493R0 )

この提案は、C++20で追加されたコア言語機能についての機能テストマクロの更新を行うものです。更新されるのは次の2つで

- `__cpp_concepts`: `201907L` -> `202002L`
    - コンセプト、Conditionally Trivial Special Member Functions（P0848R3）
- `__cpp_constexpr`: `201907L` -> `202002L`
    - 定数式における共用体アクティブメンバの切り替えの許可（P1330R0）

どうやらC++20で更新を忘れていたもののようです。従って、これはC++20への欠陥報告です。

## ==演算子からの演算子自動導出の調整

- P2468R2 The Equality Operator You Are Looking For  
  (https://wg21.link/P2468R2 )

C++20における比較演算子の自動導出、特に`operator==`からの`!=`導出に関して、導出された`!=`とユーザ定義の`!=`が衝突してしまうコーナーケースが発見されました。

```cpp
// C++17環境で定義されていた型をC++20でコンパイルしようとしている時
struct S {
  // const修飾を忘れている
  bool operator==(const S&) { return true; }  // (1)
  bool operator!=(const S&) { return false; } // (2)
};

int main() {
  bool b = S{} != S{};  // #1、!=候補が2つあり、曖昧
}
```

このコードを`friend`で書き直して`this`引数を明示的にするとともに、導出される候補も明示すると次のようになります

```cpp
struct S {
  friend bool operator==(S, const S&);  // (1)
  friend bool operator!=(S, const S&);  // (2)

  // (1)から導出される候補
  friend bool operator!=(S, const S&);  // (1)'
};

int main() {
  bool b = S{} != S{};  // #1、(2)と(1)'の間で曖昧
}
```

`#1`の箇所ではユーザー定義`!=`演算子(2)と、(1)から導出された`!=`演算子(1)'の2つが候補として上がります。第一引数（`this`引数）について`S`の右辺値との比較で（変換なしで）ベストマッチするものはなく、`S`の値（非参照）への変換がどちらの候補でも考慮されるものの、どちらのシグネチャも結果的に同じになっているためどちらの候補もベストマッチとなり、オーバーロード解決が曖昧になります。

これは、ユーザー定義の`== !=`演算子へ`const`修飾を忘れているために起こる事です。しかし、実際のコードベースではこのようなうっかりミスが良く検出されたほか、意図的にこれと同種の暗黙変換をトリガーすることでC++17環境でも演算子の自動導出のようなことをしているコードもあり、この影響が思ったよりも大きいことが判明しました。

```cpp
// CRTPで演算子を自動実装する
template <typename T>
struct Base {
  bool operator==(const T&) const;
  bool operator!=(const T&) const;
};

struct Derived : Base<Derived> { };

// 左辺の値が派生クラスから基底クラスへの変換を受けることで==/!=が使用可能になる
bool b = Derived{} == Derived{};
```

この例の`Derived{} == Derived{}`の式においては基底クラス`Base<T>`で定義された`operator==`が使用されますが、直接ベストマッチしない（`this`引数に基底クラスへの変換が必要）ため、定義された`operator==`と引数順をひっくり返して生成された`==`の2つが候補に上がるものの、右辺の値も左辺の値も基底クラスヘの変換が必要であることから2つの候補のオーバーロード解決における順位が同一になり、曖昧になります。この場合、`!=`の定義は関係ありませんが、`!=`が同じように使用されると同様の問題が起こります。

```cpp
template<bool>
struct GenericIterator {
  using ConstIterator = GenericIterator<true>;
  using NonConstIterator = GenericIterator<false>;

  GenericIterator() = default;
  GenericIterator(const NonConstIterator&); // (1)

  bool operator==(ConstIterator) const;
};

using Iterator = GenericIterator<false>;

// 暗黙変換(1)によって==を再利用する
bool b = Iterator{} == Iterator{};
```

この例では、`Iterator`（`GenericIterator<false>`）の比較を暗黙変換コンストラクタ(1)を通すことによって、`ConstIterator`（`GenericIterator<true>`）の比較として実行しています。しかし、一つ前の例と同様にユーザー定義された`operator==`には直接ベストマッチしないため、`bool operator==(ConstIterator) const`とここから導出された逆順の`==`が候補として上がり、`Iterator{} == Iterator{}`の呼び出しにおいては右辺の値も左辺の値も(1)を通して暗黙変換が必要となるため、2つの候補の間で順序が付かずに曖昧になります。

この少し複雑な2例及び最初の例はC++17までは意図通りに動作していたコードであり、おかしなことをしているわけではないため受け入れられなければなりません。そのため、当初のコンパイラの実装ではこのようなケースを特別扱いして、書き換えられていない候補を優先的に採用することでエラーになるのを回避していました。

C++23では、「生成された候補について、その生成元が`operator==`であり、それにマッチする`operator!=`が宣言されている場合（つまり、その`operator==`の名前を`operator!=`に書き換えた時に何かの再宣言となる場合）、生成された候補を取り下げる」というルールを追加します。すなわち、ユーザー定義の`operator==`に対応する`operator!=`がユーザーによって宣言されていると、その`operator==`からの演算子生成はすべて（逆順の`==`も）行われなくなります。

これにより、最初の例と2つ目の例はそのまま曖昧さが解消されます。しかし、3つ目の例（`GenericIterator`の例）は定義されている`operator==`に対応する`!=`演算子を追加することで曖昧さが解消されます（つまり、そのままだと問題が解決されない）。

そして、これはC++20への欠陥報告です。

この修正の後で、C++20における比較演算子の自動導出に関するプログラミングモデルは次のようになります

- 新規に記述するコードの場合
    - `operator==`からの演算子自動導出（`!=`および逆順`==`）を行う場合、`operator==`のみを宣言する
    - `operator==`を演算子自動導出で使用したくない場合、`operator==`に完全に対応する`operator!=`も宣言する
    - `operator<=>`は常に演算子自動導出に使用される
        - 使用したくない場合は`operator<=>`を宣言しない
- C++17から移行する場合
    - 以前の動作を保つには、`operator==`に対応する`operator!=`が宣言されていることを確認する
    - 演算子自動導出を使用しても問題ないことが確認されたら、`operator!=`を削除する

ただし、この修正の後でも壊れるコードは存在しており、提案文書では簡単な例が紹介されています。

## `u8`文字列リテラルの破壊的変更の緩和

- P2513R4 `char8_t` Compatibility and Portability Fix  
  (https://wg21.link/P2513R4 )

C++20で導入された`char8_t`型はUTF-8エンコーディングが保証された文字型であり、`u8`プリフィックスからなる文字/文字列リテラルによって`char8_t`型の文字/文字列を取得することができます。しかし、C++17までは`u8`文字/文字列リテラルは`char`型の文字/文字列を返しており、`char8_t`は破壊的変更を伴っています。

そして、`u8`リテラルによってUTF-8文字を扱ってきた既存コードが無かったはずもなく、この変更の影響は決して小さくはありませんでした。そのため、コンパイラはC++20モードでも`u8`リテラルの動作を変更しないオプションを用意しています。

```cpp
// 全て、C++17まではok、C++20からはng
const char* ptr0 = u8"a";
const char arr0[] = u8"b";
const unsigned char arr1[] = u8"c";
```

さらに、C++20でのこの変更はCとの互換性も壊しています。Cには`char8_t`型はありません（C20まで）が`u8`リテラルは使用可能で、その型は`char[N]`です。すなわち同じ問題がCとC++20の間で起こります。さらに、C23からはCにも`char8_t`が導入されましたが、`char16_t/char32_t`と同じく`char8_t`は`unsigned char`の`typedef`として定義され、`u8`文字列リテラルの戻り値は`char`型の文字列となるものの`char`と`char8_t`の配列どちらにも変換できるように規定されます。すなわち、CはC++の様な非互換を導入しておらず、結局この互換性問題は残り続けます。

```cpp
// 全て、C23以降ok、C++20からはng
extern const char* a = u8"a";
extern const char b[] = u8"b";
extern const unsigned char* c = u8"c";
extern const unsigned char d[] = u8"d";
```

このようなCおよびC++17以前との非互換を緩和するために、`char`もしくは`unsgined char`型の配列初期化時に`u8`文字列リテラルを使用できるようにします。これはCの`u8`文字列リテラルの動作に倣ったもので、`char`/`unsgined char`型の配列初期化時にだけ`u8`文字列リテラルを特別扱いするものです。

ただし、特別扱いがなされるのは配列の初期化時のみで、`const char*`の様なポインタの初期化時の動作は変更されません。また、`signed char`型の配列初期化時の動作も変更されません。

```cpp
// この修正後の動作
const char* ptr0 = u8"a";            // (1)、ng
const unsigned char* ptr1 = u8"b";   // (2)、ng
const signed char* ptr2 = u8"c";     // (3)、ng
const char arr0[] = u8"d";           // (4)、ok
const unsigned char arr1[] = u8"e";  // (5)、ok
const signed char arr2[] = u8"f";    // (6)、ng
```

この例の動作は次のように推移することになります

|例|C++17まで|C++20|この修正|
|:-:|:-:|:-:|:-:|
|(1)|`✅`|`💔`|`💔`|
|(2)|`❌`|`❌`|`❌`|
|(3)|`❌`|`❌`|`❌`|
|(4)|`✅`|`💔`|`✅`|
|(5)|`✅`|`💔`|`✅`|
|(6)|`✅`|`💔`|`💔`|

凡例

- `✅`: ok、コンパイルが通る
- `❌`: C++11からng
- `💔`: C++20からng（破壊的変更

また、`u8`文字列リテラルの結果を`return`していたようなコードはそのままでは壊れたままになるものの、一旦配列初期化を介することで動作するようになります。

```cpp
// C++17までok, C++20でng
constexpr const char* resource_id () {
  return u8"o(*￣▽￣*)o";
}

// この提案後、こう書けばok
constexpr const char* resource_id () {
  const char res_id[] = u8"o(*￣▽￣*)o";
  return res_id;
}
```

この修正後では、C++17以前とC++20、およびCとの互換性を持つように記述されたコードにおいて`u8`文字列リテラルの保持には、`unsigned char[]`型を用いるとポータブルになります。あるいは、C++17以前との互換性を確保したい（Cは気にしない）場合は、`const char[]`型を使用するとポータブルになります。

この修正はC++20への欠陥報告です。

\clearpage

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- Compiler Explorer(https://godbolt.org/)
- 見た目で区別できない変数 | ++C++; // 未確認飛行 C ブログ  
  (https://ufcpp.net/blog/2020/5/variationselectoridentifier/)
- UAX31: Unicode Identifier の話 | ++C++; // 未確認飛行 C ブログ  
  (https://ufcpp.net/blog/2021/2/uax31/)
- メンバ関数の新しい書き方、あるいは Deducing this - Zenn  
  (https://zenn.dev/acd1034/articles/221117-deducing-this)

\clearpage

　
