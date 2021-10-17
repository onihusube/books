---
title: C++20 言語機能編
author: onihusube
date: 2021/12/30
geometry:
  width: 188mm
  height: 263mm
#coverimage: cover.jpg
#backcoverimage: backcover.jpg
titlecolor:
  color1:
    r: 0.796078
    g: 0.949020
    b: 0.4
    c: 0.25
    m: 0.0
    y: 0.80
    k: 0.0
  color2:
    r: 0.207843
    g: 0.631373
    b: 0.419608
    c: 0.75
    m: 0.0
    y: 0.65
    k: 0.0
okuduke:
  revision: 初版
  printing: ねこのしっぽ
---
\clearpage

# はじめに

# 大きな機能

## `<=>`

### 符号付整数型の内部表現は２の補数であると規定


## コンセプト

### 特殊メンバ関数の条件付きトリビアル定義

## モジュール

## コルーチン

# 定数式

## 仮想関数

## `dynamic_cast/typeid`

## `try-catch`

## 共用体のアクティブメンバ切り替え

## トリビアルなデフォルト初期化

## インラインアセンブリ

## `consteval`

## `constinit`

## 動的メモリ確保

# テンプレート

## `auto`による関数テンプレートの簡易定義

## `typename`の省略

## クラス型の非型テンプレート引数

## 集成体テンプレートのCTAD

## エイリアステンプレートのCTAD

## `explicit(bool)`

# ラムダ式

## 暗黙のラムダキャプチャの簡易化（DR）

## テンプレート構文

## `[=, this]`

## ステートレスラムダ式のクロージャ型

## 評価されない文脈におけるラムダ式

## 初期化キャプチャのパック展開

## 構造化束縛した変数のキャプチャ

# 構造化束縛

## `static/thread_local`の許可

## 非公開メンバへのアクセス（DR）

## 呼び出す`get`メンバ関数の考慮の変更（DR）

# ユニコード文字型

## `char8_t`

## `char16_t/char32_t`

- P1041R4 Make `char16_t/char32_t` string literals be UTF-16/32 (https://wg21.link/p1041r4)

C++11でユニコード文字を表す型として`char16_t/char32_t`とそのリテラルを得るための`u/U`プリフィックスが導入されましたが、実際の所それらの型の値及びリテラルの結果がユニコード文字列（UTF-16/UTF-32）となるかどうかは規定されていませんでした。

とはいえ規格の文面は`char16_t/char32_t`のエンコードが明らかにユニコードであることをほのめかしており、`char16_t`のエンコードとしてサロゲートペアを認めていたり、`char32_t`の文字は1文字が複数の`char32_t`値に跨がらないことを指定していて、またどちらの文字型もユニコード文字のすべての範囲を表現できるように指定されています。

これらの特徴を持つエンコーディングはUTF-16/UTF-32以外に無く、全ての実装は`char16_t/char32_t`のエンコーディングとしてUTF-16/UTF-32を使用していたため、C++標準としても`char16_t/char32_t`のエンコーディングを明確にUTF-16/UTF-32であると規定することになりました。

この変更は実際の慣行を標準に適用しただけなのでプログラマには影響はありませんが、`char16_t/char32_t`を使用するときにエンコーディングを気にする必要がなくなり、`__STDC_UTF_16__/__STDC_UTF_32__`の様なマクロを使用する必要もなくなるなど、余分なことを考えなくてよくなります。

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

EBOによるサイズ削減テクニックは特に、テンプレートパラメータ型をメンバとして保持する必要がある場合に行われ、標準コンテナのアロケータ型で特に目にすることができます。

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

- P0479R5 Proposed wording for likely and unlikely attributes (https://wg21.link/p0479r5)

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
  narrow n5();  // ng、これは関数宣言
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

ここでの`new`による構築時に`T()`としていることによって、`T`が集成体である場合に集成体初期化が行われません（コピー/ムーブコンストラクタは呼ばれる）。そこを`{}`に変えてれば集成体初期化は行われますが、`{}`と`()`の性質の違いから別の問題が発生します。`()`による集成体初期化はこのような場合にわざわざ集成体にコンストラクタを書いたり、ムーブ構築に切り替えたりしなくてもよくするために導入されました。このようなものにはほかにも、`std::make_shared()`や`std::make_unique()`、`std::make_from_tuple()`などがあります。

実際のところ、`()`による集成体初期化が必要となるのはこのようなライブラリの内部においてだけであり、普通に使う時は常に`{}`を使うべきです。

## ユーザー宣言コンストラクタの禁止

- P1008R1 Prohibit aggregates with user-declared constructors (https://wg21.link/p1008r1)

C++11より集成体型では`default/delete`なコンストラクタを宣言することはできていましたが、C++20からは集成体型では一切のコンストラクタ宣言を行えなくなります。

これは、`default/delete`なコンストラクタの意図と集成体初期化がかみ合っていなかったり、`explicit`なコンストラクタが集成体初期化と相性が悪かったための変更で、C++11以降のコードに対する破壊的変更となります。

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

`std::stringstream`は`std::ios_base`から継承した`seekdir`の`end`静的メンバ変数（ファイルストリームのシーク方向を指定するもの）を持っています。その存在はコントロールできないため、`begin()/end()`関数は名前空間スコープに非メンバ関数として定義されています。範囲`for`の探索ルールはメンバ`begin/end`の存在を確認しますがそれが関数かどうかはチェックしないため、この`X`に対してはメンバ`end`だけが見つかるのでメンバ関数の`begin()/end()`を使用しようとし、当然利用できないのでコンパイルエラーとなります。これは、標準のストリームクラスを継承したラッパ型において同じことが起こります。

このように、イテレータを提供するわけではない`begin/end`メンバ（関数ですらないかもしれない）を持つがユーザーがそれを変更できないような型がある時、ADLで発見される事を期待して名前空間スコープに`begin()/end()`を用意していても範囲`for`はそれを使用できずエラーとなってしまっていました。

この挙動は修正され、メンバとして`begin/end`の両方が見つかった場合にのみメンバ関数の`begin()/end()`を使用し、片方だけしか見つからない場合はメンバ関数の使用を諦めADLによって`begin()/end()`を探すようになります。これによって、ユーザーはより柔軟に範囲`for`にアダプトした型を書くことができるようになります。なお、これはC++11に対する欠陥報告（DR）であるため、この変更を適用済みのコンパイラではC++11モードでもこの修正は適用されます。

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
  s[1, 1, 1] = 42;
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

# とてもマイナーな変更

## 関数テンプレートに明示的に型指定した場合にADLで見つからない問題を修正

- P0846R0 ADL and Function Templates that are not Visible (https://wg21.link/p0846r0)

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

- P0388R4 Permit conversions to arrays of unknown bound (https://wg21.link/p0388r4)

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

プライベートローカルクラステンプレートを用いて特殊化した上でクラス外から参照しようとすると、calngやiccでは許可されないのに対して、gcc,msvcでは許可されます。

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

- P0641R2 Resolving Core Issue #1331 (const mismatch with defaulted copy constructor)  
  (https://wg21.link/P0641R2)

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

## const修飾されたメンバポインタの制限を修正

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

# Defact Report

欠陥報告（*Defact Report* : DR）とは、仕様に対する欠陥（バグ）の報告に伴う解決のための提案であり、その変更は過去のバージョンに遡って適用されます。

ここまでにもいくつかDRとなる提案がありました（タイトルの後ろにDRと記載のあるもの）が、ここでは特にカテゴライズされないものを見ていきます。

DRとされた問題については一部のコンパイラは早期に実装している可能性があり、実装済みのコンパイラでは古いバージョンの指定時（C++20対応コンパイラに`-std=c++11`を指定するなど）にも変更が適用されてコンパイルされるようになります。ただし、DRがどのバージョンまで遡って適用されるかは指定されていない場合が多く、謎です・・・

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

これは、`default`宣言する特殊関数の例外指定はその宣言がないときに暗黙に定義される特殊関数のものと同じにならないといけないというルールによります（少し前で、コピーコンストラクタについても似たような制限がありました）。

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

P0929はこの問題を解決するものであり、関数や配列などの宣言の時点では抽象クラスが使用されてもそこではその抽象性をチェックせず、その関数や配列が実際に定義されたとき（あるいは関数が呼び出されたとき）に初めてチェックされコンパイルエラーが発生するようにする、というものです。

これによる恩恵はおそらく、テンプレートメタプログラミングの文脈でテンプレート実引数として抽象型が使用されたときに不要なエラーが起こらなくなるという形で得られるでしょう。

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- cppmap(https://cppmap.github.io/ : ライセンスはCC0 パブリックドメイン)
- yohhoyの日記(https://yohhoy.hatenadiary.jp/)

表紙は友人のKさんに書いていただきました。可愛いキノコをありがとうございました！