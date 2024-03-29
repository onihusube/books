---
title: C++20 ranges
author: onihusube
date: 2021/07/11
geometry:
  width: 188mm
  height: 263mm
coverimage: cover.jpg
backcoverimage: backcover.jpg
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

本書はC++20より追加されたRangeライブラリの解説を行うものです。Rangeライブラリの裾野は広く、`<ranges>`ヘッダ以外のところにも多数のものが追加されていますが、基本的に`<ranges>`ヘッダのものに焦点を当てています。そのため、`<ranges>`ヘッダの外にあるものは深入りせずに簡単な紹介に留めています。特に、`<iterator>`と`<concepts>`はRangeライブラリと深く関わっておりC++20で追加/改修されていますが、詳細には踏み込まないため不明瞭となる部分があるかもしれません、ご了承ください。

Rangeライブラリのベースとなっているコンセプト機能については必要になったタイミングで適宜説明を入れていますが、サンプルコードの不足など不十分であるかもしれません。適宜cpprefjpのコンセプトページを参照されると理解が深まるかと思います。

- C++20 コンセプト - cpprefjp  
  https://cpprefjp.github.io/lang/cpp20/concepts.html

また、本書はRangeライブラリの実体やその設計思想などを理解しようと試みるものであり、ライブラリリファレンスではありません。そのため、必ずしも使用するために十分な情報が記載されていないかもしれません。

## お約束など

範囲（*range* : レンジ）とは配列などの様に何らかの要素の列となっているものを指します。この本では、範囲の事をシーケンスとも呼んでいます。

Rangeライブラリのものは`std::ranges`名前空間にありますが、文中では必要な場合を除いて基本的に省略します。ただし、Rangeライブラリ以外のものは`std::`を基本的には省略しません。サンプルコードにおいては、各種ヘッダのインクルードを省略した上で`using namespace std::ranges;`をしているかのように記述しています。

\clearpage

# コンセプト

C++20よりコンセプトが言語機能に正式に組み込まれました。これによってテンプレートパラメータに制約をかけたり、関数テンプレートに優先順を付ける事がC++17以前と比較してより簡単に行えるようになりました。それとともに、ライブラリにおける処理や操作の対象となる”もの・概念”をコンセプトによって明確に定義し、そのコンセプトを中心にジェネリックに組み上げていくコンセプトベースのライブラリデザインの時代が幕を開けています。

Rangeライブラリの元となったRange-v3ライブラリはコンセプトをエミュレートする事でC++17の時代からコンセプトベースで設計されていたライブラリで、それを受けてRangeライブラリも当初からコンセプトベースで設計されており、ライブラリの中心となる概念をコンセプトによって定義しています。

コンセプトベースでしっかりと設計されたライブラリはまだあまりありませんが、C++23に導入予定のExecutorライブラリもコンセプトをベースとした非常にジェネリックなライブラリとして設計されています。

## `range`

Rangeライブラリの中心概念である __範囲（*range*）__ を定義するのが`range`コンセプトです。コンセプトによって、C++のコードとして次のように書かれます。

```cpp
template<class T>
concept range =
  requires(T& t) {
    ranges::begin(t); // equality-preservingが要求されることがある
    ranges::end(t);
  };
```

コンセプトはテンプレートであり、テンプレートパラメータ部分を除くと`concept 名前 = 制約式...;`のような形式で記述します。コンセプトを複数の制約式で構成する場合は、それらを論理積（`&&`）や論理和（`||`）で繋げて構成します。コンセプトに具体的な型を与えて実体化すると、制約式1つ1つは`true/false`のどちらかを返し、全体として`true`となるときにそのコンセプトはその型について満たされます。そして、実体化されたコンセプトは`bool`型の定数式となります。

`range`コンセプトの制約式に現れている`requires(引数){式...}`は`requires`式と呼ばれる制約式で、そこに記述された式が全てコンパイルエラーを起こさない（使用可能である）場合に`true`を返すものです。`range`コンセプトは1つの`requires`式だけから構成されています。その意味は単純で、型`T`のオブジェクト`t`に対して`std::ranges::begin(t)/std::ranges::end(t)`がどちらも呼び出し可能である事です。`std::ranges::begin()/std::ranges::end()`はそれぞれ範囲の先頭イテレータと終端イテレータ（番兵）を取得するもので、`range`コンセプトを満たす型は`std::ranges::begin()/std::ranges::end()`が指定する手段によって範囲の先頭と終端を取得できることを表しています（`std::ranges::begin()/std::ranges::end()`は後程詳しく見ていきます）。

以降しばらく、`std::ranges::begin()/std::ranges::end()`を`begin/end`と書きます。

そして、`range`コンセプトにはこれに加えて次のような要件が文章で指定されます。

- `[begin(t), end(t))`が範囲を示す
- `begin(t), end(t)`の計算量は償却定数であり、範囲を変更しない
- `begin(t)`が`std::forward_iterator`コンセプトのモデルであるならば、`begin(t)`は等しさを保持する（*equality-preserving*）

標準ライブラリのコンセプトの多くは、このような文章によって追加の要求がなされています。そのような要求の多くは、そのコンセプトを満たしていれば当然満足されているはずですが、コンセプト機能によってコードとしてチェックできるものではない事が指定されています。このような要件のことを __意味論的な要件（*semantic requirements*）__ と呼び、コンセプト定義に直接現れている制約（式）を __構文的な要件（*syntactic requirements*）__ と呼びます。コンセプトは構文的な要件と意味論的な要件の両方が揃ってコンセプトであり、コンセプトを利用するときは構文的な部分だけでなく意味論的な部分にも注意を払う必要があります。

ある型`T`が、コンセプト`C`の構文的な要件と意味論的な要件の両方を全て満たしている時、「型`T`はコンセプト`C`のモデルである」と言います。本書で単にコンセプトを満たす等のように言っているときは、モデルとなっているという意味だと思ってください。

さて話を戻して、`range`コンセプトの意味論的な要件を見てみましょう。

> `[begin(t), end(t))`が範囲を示す

これは、同じオブジェクトから取得したイテレータと番兵は、同じ範囲についてのイテレータと番兵であれという意味です。ここでの範囲とは、配列やコンテナなどの要素の列となっているものを指します。例えば、`begin`と`end`がそれぞれ異なる配列のイテレータを返すような実装になっている場合、`range`コンセプトを構文的には満たしますがモデルとはなりません。

> `begin(t), end(t)`の計算量は償却定数であり、範囲を変更しない

ここでの計算量とはシーケンス`t`の要素数に基づくものです。すなわち、`t`の要素数が幾つであれ`begin/end`の呼び出しは一定時間で完了しなければなりません。償却とあるのは、繰り返し呼んだ時に結果的に一定時間となれば良いという意味で、例えば最初の呼び出しだけ何か処理をして結果をキャッシュしておいて、2回目以降の呼び出しはキャッシュから直ぐに返す、という実装が許されます。

範囲を変更しないとはそのままで、`t`の表す範囲を`begin/end`の呼び出しに伴って何か変更（要素を消したり、並び替えたり）してはいけません。

> `begin(t)`が`std::forward_iterator`コンセプトのモデルであるならば、`begin(t)`は等しさを保持する（*equality-preserving*）

`std::forward_iterator`コンセプトは名前の通り、従来*forward iterator*と呼ばれていた種類のイテレータを定義するコンセプトで、C++20から`<iterator>`にて定義されています。イテレータカテゴリには継承関係があり、コンセプトにもそれに応じた包含関係があります。*forward iterator*より強いイテレータは同時に`std::forward_iterator`コンセプトのモデルでもあります。

ある式が等しさを保持する（*equality-preserving*である）とは、ある式に同じ値を入力するといつも同じ結果が得られる事を言います。簡単に言えば、式の内部で状態を持ったりグローバルな変数に依存したりしていない、という事です。同じオブジェクトではなく「同じ値」であることに注意してください、同じオブジェクトを渡してもその値が変更されているならば同じ結果を返す必要はありません。

この意味論的な要件はつまり、`begin(t)`から得られるイテレータが*forward iterator*（あるいはより強いイテレータ）であるならば、`begin(t)`は同じ`t`に対して同じイテレータを返すべし、という事です。

この要件は何を示しているのでしょうか？そのヒントは、イテレータが*forward iterator*であるというところにあります。

*forward iterator*にはマルチパス保証が課せられています。マルチパス保証があるイテレータは、参照するシーケンスを複数のイテレータから走査することができます。すなわち、イテレータの操作によってシーケンスの状態が変更されることがありません。  
*forward iterator*ではないイテレータ（*input(output) iterator*）にはそれがありません。つまり、*input(output) iterator*であるイテレータはその操作（`++, *`など）によって参照するシーケンスの状態が変更されることがあります。そして、そのようなイテレータでは`begin()`によるイテレータ取得時に何かしてからイテレータを得る事があります。つまり、`begin()`の呼び出しが等しさを保持できない場合がありえます。

この要件はそのような特殊なイテレータを認めつつ、普通のイテレータに対しては当然の要求をするものです。このようなイテレータには、例えば`istream_iterator/ostream_iterator`があります。

見てみると、これらの意味論的な要件は実行時にすらチェックが難しいものばかりですが、範囲というものを考えると確かにそれが期待される要求である事がわかると思います。このように、コンセプトはコードで書く構文的な要件と文章で書く意味論的な要件の2本柱から構成され、それによってそれまで何かふわっとしていた概念というものを明確に定義します。

また別の見方をすると、コンセプト定義に現われる構文的な要件によってある概念にまつわる操作や性質を（構文的に）規定し、意味論的な要件によってそれらの操作や性質の意味を（意味論として）定め、その2つをもって1つの概念を構文・意味論の両方向から定義している、と見ることができます。

### コンセプトの暗黙の意味論的な要件

コンセプトには、全てのコンセプトで共通して要求される3つの暗黙の意味論的な制約があります。

- 等しさの保持（*equality-preserving*）
    - 同じ値の入力に対して同じ結果を返す
- 安定（*stable*）
    - 変更されていない同じオブジェクトの入力に対して同じ結果を返す
- 暗黙の式のバリエーション（implicit expression variations）
    - `requires`式において、`const T&`の引数を受ける制約式は期待される全ての値カテゴリの引数を受け入れる必要がある

安定は等しさの保持とセットであり、この二つでもってその式が状態を持たず直接の引数と結果以外に副作用を及ぼさないことを定義します。暗黙の式のバリエーションはほとんど気にする必要はないので、ここでは深く触れませんし他のところでも言及されないはずです。ただ、型がコンセプトのモデルとなるためにはこの3つの要件を全て満たしている必要があります。

等しさの保持（および安定）に関しては、「等しさを保持することを要求しない（*not required to be equality-preserving*）」のように指定される場合があり、この時は満たしている必要はありません。先ほどの`range`コンセプトでは条件付きで満たさなくても良いという指定でした。

### Rangeライブラリ

Rangeライブラリの"Range"とは何か？といえば、この`range`コンセプトを満たす（モデルとなる）あらゆる型の事です。それは`range`コンセプトが定義するように、何かしらの範囲（シーケンス）を表す型であり、`ranges::begin()/ranges::end()`という2つの操作によってその範囲の先頭要素を指すイテレータと終端を表す番兵のペアを取得することができます。

Rangeライブラリはこの*range*という対象について様々なサポートを与えるもので、任意の範囲とそれにまつわる各種操作をより便利に、安全に、簡潔に、そして一様に扱う事を可能にするジェネリックなライブラリです。

## `borrowed_range`

`borrowed_range`は別のオブジェクトが所有するシーケンスを参照しているだけの`range`を定義するコンセプトです。

```cpp
template<class T>
concept borrowed_range =
  range<T> &&
  (std::is_lvalue_reference_v<T> || enable_borrowed_range<remove_cvref_t<T>>);
```

このコンセプトは3つの制約式から構成されています。この様に、コンセプトを構成する制約式としては`requires`式以外にもコンセプトそのものや、`bool`を返す任意の定数式を用いることができます。

そして、`borrowed_range`はその定義中に`range`コンセプトを（`&&`によって繋ぐ形で）含んでいます。この場合`borrowed_range`は`range`コンセプトを完全に包含しており、この包含関係によってコンセプトの半順序が決定され、`borrowed_range`コンセプトによる制約は`range`コンセプトによる制約よりも強く順序付け（制約）されます（あえて記号で書くならば、`range` ⊂ `borrowed_range`、右側ほど強い）。2つのコンセプトの間に順序が付けられるとき、それらのコンセプトを用いてオーバーロードされている関数の間に呼び出しの優先順位を付けることができ、曖昧にならなくなります。ただし、制約式同士が`||`で繋がれている場合はこの限りではなく、ルールはより複雑になります・・・

```cpp
// コンセプトを使用する時は、typename(class)のところにコンセプトを指定する
template<range R>
void f(R&& r);  // (1)

template<borrowed_range R>
void f(R&& r);  // (2)

int main() {
  std::vector v = {...};
  f(v); // (2)が呼ばれる
        // R == std::vector<T>&
}
```

`borrowed_range`コンセプトの意味論的な要件は

- 型`T`のオブジェクト`t`がある時、`t`から取得したイテレータの有効期間が`t`の生存期間（*lifetime*）に関連づけられていない

`borrowed_range`の役割の大部分はこの意味論要件が定義しています。

`borrowed_range`のモデルとなるような型`T`は、関数の引数として受け取る時に参照（`T&`）ではなく値（`T`）で受け取り、その引数から取り出したイテレータをそのまま戻り値として返す事を安全に行う事ができます。

定義に現れているように範囲を示す型`R`に対する左辺値参照（`R&`）がそうですが、そのほかにも例えば`std::string_view`のようなものが該当します。

```cpp
auto strview(std::string_view sv) {
  // 文字列のイテレータはsvの寿命とは無関係に有効

  return sv.begin();  // OK、有効なイテレータを返す
}

auto str_ref(std::string& sr) {
  // 文字列のイテレータは参照srの寿命とは無関係に有効

  return sr.begin();  // OK、有効なイテレータを返す
}

auto str(std::string s) {
  // 文字列のイテレータはsの寿命とリンクしている

  return s.begin();  // NG、ダングリングイテレータを返す
}
```

つまり、`borrowed_range`のモデルとなるような型`T`は範囲を参照しているが所有しておらず、そのオブジェクトの寿命が尽きたとしても元の範囲にはなんの影響も与えないという事です。このようなものは一般的に*View*とも呼ばれます。

`borrowed_range`の役割の大部分は意味論要件にあり、構文的には`range`コンセプトを満たすだけの型でも満たせてしまうため、`borrowed_range`はデフォルトでは全ての型に対して無効化されています。アダプトするためにはオプトインで有効化する必要があり、そのスイッチは定義中に現れている`enable_borrowed_range`によって行います。

```cpp
namespace std::ranges {
  // デフォルトの定義
  template<class>
  inline constexpr bool enable_borrowed_range = false;
}


// 自作viewクラス
struct my_view { ... };

// 例えばこのように有効化（変数テンプレートの明示的特殊化）
inline constexpr bool std::ranges::enable_borrowed_range<my_view> = true;
```

`enable_borrowed_range`はRangeライブラリの一部の型と`std::string_view`、`std::span`に対してこのような特殊化が用意されており、それらの型は何もせずとも`borrowed_range`のモデルとなることができます。

## `sized_range`

`sized_range`は範囲の要素数（長さ）が一定時間で求められる`range`を定義するコンセプトです。

```cpp
template<class T>
concept sized_range =
  range<T> &&
  requires(T& t) { ranges::size(t); };
```

意味論要件は、`T`から参照を取り除いた型のオブジェクト`t`について

- `ranges::size(t)`の計算量は償却定数であり`t`を変更せず、結果は`ranges::distance(t)`と等しくなる
- `iterator_t<T>`が`std::forward_iterator`のモデルであるとき、`ranges::size(t)`は`ranges::begin(t)`の評価と関係なく呼び出し可能

1つ目の要件は意味が分かると思います。範囲の長さと範囲の距離は同じものを別の名前で呼んでいるだけです。`ranges::distance()`というのは`std::distance()`のリファイン版で、`range`オブジェクトを直接受け取ることができるものです。

> `iterator_t<T>`が`std::forward_iterator`のモデルであるとき、`ranges::size(t)`は`ranges::begin(t)`の評価と関係なく呼び出し可能

前半は`T`から`ranges::begin()`で取れるイテレータが*forward iterator*であるとき、という事ですが、後半は何を言っているのでしょうか・・・

`range`コンセプトを思い返してみると、その`begin`の呼び出しは償却定数であることが求められており、例えば最初の`begin`の呼び出し時に何かをしてそれをキャッシュし2回目以降はキャッシュから返すような実装が許される、という事を言っていました。ここでの何かをするというのは殆どの場合シーケンスを生成する事だと思われます。その場合、そのクラスの表現する範囲というものは`begin`を最初に呼び出すまでは生成されておらず、そのクラスが`sized_range`であったとしても`begin`の呼び出し前にはシーケンスの長さを求めることは出来ないわけです。

この2つ目の要件は、そのような`range`はイテレータが*input(output) iterator*であるときにのみ許可されることを意味しています。そして、そのような`range`は後々紹介するRangeライブラリが備える各種`view`の中に見ることができます。

この`sized_range`コンセプトはオプトアウトすることができます。そのメカニズムは`enable_borrowd_range`と同様です。

```cpp
namespace std::ranges {
  // デフォルトの定義
  template<class>
  inline constexpr bool disable_sized_range = false;
}


// 自作の何かrange
struct non_sized_range;

// 例えばこのように無効化（変数テンプレートの明示的特殊化）
inline constexpr bool std::ranges::disable_sized_range<non_sized_range> = true;
```

この様にしてオプトアウトした型に対しては`ranges::size`が呼び出されなくなり、`sized_range`コンセプトを無効化できます。

`ranges::size`は様々な手段でその引数から長さを取得しようとします。結果として、`sized_range`コンセプトを構文的には満たしても意味論的な要件まで満たせない型が出て来てしまいます。これは、そのような場合に`sized_range`を無効化するためのものです。  

例えば、`std::forward_list`のようにサイズを求めることは（イテレータを用いて）出来るけれど計算量が償却定数にならないような実装になっている場合に活用できます（ただ、`std::forward_list`そのものは`ranges::size`でサイズを求められません）。

## `common_range`

`common_range`はその先頭イテレータと終端を指すイテレータ（番兵）とが同じ型となる`range`を定義するコンセプトです。

```cpp
template<class T>
  concept common_range =
    range<T> && same_as<iterator_t<T>, sentinel_t<T>>;
```

意味論要件はなく、見たままです。`iterator_t<T>, sentinel_t<T>`というのは、`range`型`T`からイテレータ型（`ranges::begin()`の戻り値型）と番兵型（`ranges::end()`の戻り値型）を取得するためのエイリアステンプレートです。

C++17まではイテレータに対してその範囲の終端を指すのは同じ型のイテレータだったので、当たり前のことを言っているように思えるかもしれません。

しかしC++20からはむしろ逆で、`ranges::begin`で得られる先頭イテレータと`ranges::end`で得られる終端イテレータは別の型となっている事が前提とされるようになっています。そして、終端を指すものの事を番兵（*Sentinel*）と呼びます。”もの”と言っているように、番兵はイテレータとして扱える必要すらありません。C++17の世界ではあらゆるものが`common_range`でしたが、C++20の世界ではそうではなく、`common_range`は定義通りに単なる`range`よりも厳しい要求となっているのです。

なお、範囲`for`はC++17の時に`common_range`ではない`range`に対しても動作するように改修されています。

## `input_range`

`input_range`はそのイテレータが*input iterator*であるような`range`を定義するコンセプトです。

```cpp
template<class T>
  concept input_range =
    range<T> && input_iterator<iterator_t<T>>;
```

意味論的な要件は指定されていません。

`input_iterator`コンセプトはその名の通り*input iterator*と呼ばれてきたものを定義するコンセプトです。

```cpp
template<class I>
  concept input_iterator =
    input_or_output_iterator<I> &&
    indirectly_readable<I> &&
    requires { typename ITER_CONCEPT(I); } &&
    derived_from<ITER_CONCEPT(I), input_iterator_tag>;
```

ここにも、意味論的な要件は指定されていません。

各種コンセプトを深掘りしていくとページ数がいくらあっても足りないので簡単な紹介に留めます。

`std::input_or_output_iterator`コンセプトは前置/後置インクリメントや間接参照、ムーブが可能であるなど、イテレータとしての基本的な要件を指定するものです。`std::indirectly_readable`コンセプトは`operator*`による間接参照によって値を読み取る事ができる事を表すものです。

`ITER_CONCEPT(I)`というのは説明のためのエイリアステンプレートのようなもので、イテレータ型`I`から最も適切な*iterator category*を取得してくるものです。そして、`derived_from`コンセプトでそれが`std::input_iterator_tag`から派生しているかをチェックしています。継承関係も含めてチェックすることでより強いイテレータを含めて判定しており、また将来的なイテレータカテゴリの拡張に備えています。

C++17までの*input iterator*と意味は同じなのですが、求められる事が若干変化しているため、C++17とC++20の*input iterator*には相互に互換性がありません。

## `output_range`

`output_range`はそのイテレータが*output iterator*であるような`range`を定義するコンセプトです。

```cpp
template<class T>
  concept output_range =
    range<T> && output_iterator<iterator_t<T>>;
```

意味論的な要件は指定されていません。

`output_iterator`コンセプトはその名の通り*output iterator*を定義するコンセプトです。

```cpp
template<class I, class T>
  concept output_iterator =
    input_or_output_iterator<I> &&
    indirectly_writable<I, T> &&
    requires(I i, T&& t) {
      *i++ = std::forward<T>(t);  // 等しさを保持することを要求しない
    };
```

- 型`T`のオブジェクト`t`（値カテゴリは任意）、間接参照可能な型`I`のオブジェクト`i`について、`*i++ = v;`は次の式と等価
  ```cpp
  *i = E;
  ++i;
  ```

`std::indirectly_readable`コンセプトは間接参照（`*`）による値の読み取りを定義するコンセプトです。

`std::output_iterator`の意味論的な要件は何を言っているのか一見わかりませんが、要はイテレータへの出力と進行の操作を1行で書いても分けて書いても同じ結果となる事を要求しています。

C++17までの*output iterator*と意味は同じですが求められる事が若干厳しくなっており、C++17までの*output iterator*はこのコンセプトを満たす事ができない場合があります。逆にC++20*output iterator*はC++17*output iterator*に対して後方互換性があります。

## `forward_range`

`forward_range`はそのイテレータが*forward iterator*であるような`range`を定義するコンセプトです。

```cpp
template<class T>
  concept forward_range =
    input_range<T> && forward_iterator<iterator_t<T>>;
```

意味論的な要件は指定されていません。

`range`ではなく`input_range`を用いているのは、イテレータカテゴリ間の関係性を表現するためです。すなわち、`forward_range`は`input_range`でもあり、より強く制約されています。

`forward_iterator`コンセプトはその名の通り*forward iterator*を定義するコンセプトです。

```cpp
template<class I>
  concept forward_iterator =
    input_iterator<I> &&
    derived_from<ITER_CONCEPT(I), forward_iterator_tag> &&
    incrementable<I> &&
    sentinel_for<I, I>;
```

- `forward_iterator`の`==`は、同じ範囲を参照しているイテレータの全てと（異なる型も含めて）比較可能である
    - イテレータの型が同じである場合、デフォルト構築されたイテレータとの比較が可能である
    - そのようなデフォルト構築されたイテレータ同士の比較は常に`true`となる（同じ空の範囲の終端を指しているかのように振る舞う）
- 範囲`[i, s)`を参照する`forward_iterator`から取得された`[i, s)`への参照やポインタは、`[i, s)`が範囲として有効である限り有効であり続ける
- マルチパス保証

`incrementable`はインクリメント（`++`）によるイテレータの進行を定義するコンセプトで、同時にコピー/ムーブ構築と代入、デフォルト構築を要求します。`sentinel_for`はイテレータ型に対する番兵型を定義するコンセプトで、`sentinel_for<I1, I2>`は型`I1`がイテレータ型`I2`の番兵型であることを定義します。すなわち、`forward_iterator`である`I`は自分自身との比較が可能でなければなりません。

ただし、これは`forward_range`ならば`common_range`であることを意味せず、`forward_range`である様な範囲の終端イテレータは`I`と異なっていても構いません。`forward_range`における*forward*性はあくまでそのイテレータ型だけに対して定義されています。

意味論要件は少し難解ですが、デフォルト構築されたイテレータが他のイテレータと常に比較可能でありそのイテレータに対して終端を示すことや、イテレータの参照する要素の有効性は範囲の生存期間に従う（イテレータの操作と無関係になる）など、普通のイテレータに期待される振る舞いを定義しています。

3つ目のマルチパス保証とはコードで書くと次のような要件です。

- `a == b`ならば`++a == ++b`
- `((void)[](X x){++x;}(a), *a)`は`*a`と等しい

特に2つ目は何言ってるのか分かり辛いですが、「同じ範囲を指すイテレータは同じ範囲を同じ順番でイテレートする」「イテレータをコピーしてから何かをしても、コピー元のイテレータに影響はない」という事を言っています。これは*input iterator*にはない`forward_iterator`の最大の特徴です。

マルチパス保証があることで、ある範囲を別々のイテレータから複数回走査するような、マルチパスなアルゴリズムをイテレータを用いて書くことができます。

C++17までの*forward iterator*と意味は同じですが求められる事が若干厳しくなっており、C++17までの*forward iterator*はこのコンセプトを満たす事ができない場合があります。逆にC++20*forward iterator*はC++17*forward iterator*に対してほぼ後方互換性があります。

## `bidirectional_range`

`bidirectional_range`はそのイテレータが*bidirectional iterator*であるような`range`を定義するコンセプトです。

```cpp
template<class T>
  concept bidirectional_range =
    forward_range<T> && bidirectional_iterator<iterator_t<T>>;
```

意味論的な要件は指定されていません。

これも定義から明らかなように、`bidirectional_range`は`forward_range`でもあり、より強く制約されています。

`bidirectional_iterator`コンセプトはその名の通り*bidirectional iterator*を定義するコンセプトです。

```cpp
template<class I>
  concept bidirectional_iterator =
    forward_iterator<I> &&
    derived_from<ITER_CONCEPT(I), bidirectional_iterator_tag> &&
    requires(I i) {
      { --i } -> same_as<I&>;
      { i-- } -> same_as<I>;
    };
```

- 任意の`bidirectional_iterator`を`r`とすると、`r`は`++q == r`となるようなイテレータ`q`が存在する場合にのみデクリメント（`--`）可能
- 型`I`の等しい2つのイテレータ`a, b`について
    - `a, b`がデクリメント可能ならば、次の4つの条件を全て満たす
        - `addressof(--a) == addressof(a)`
        - `bool(a-- == b)`
        - `a--, --b`の評価の後でも、`bool(a == b)`は`true`となる
        - `bool(++(--a) == b)`
    - `a, b`がインクリメント可能ならば、`bool(--(++a) == b)`

ここに来ると、構文的要件は見たままで分かりやすいです。デクリメント（`--`）という操作が可能であることを要求しています。

それれだけだとデクリメントが何をするものなのか不明瞭ですので、意味論要件によってデクリメントによるイテレータの後退という操作を定義しています。一つ前の要素が無いと後退できないとか、デクリメントしてもイテレータそのもののアドレスが変わるわけではないとか、デクリメントしてインクリメントすれば元に戻る、など*bidirectional iterator*に期待される振る舞いを定義しています。

これも*forward iterator*と同様意味は同じですが求められる事が若干厳しくなっており、C++17までの*bidirectional iterator*はこのコンセプトを満たす事ができない場合があります。逆にC++20*bidirectional iterator*はC++17*bidirectional iterator*に対してほぼ後方互換性があります。

`bidirectional_range`および`bidirectional_iterator`の*bidirectional*性はあくまでそのイテレータ型に対して定義されていることに注意してください。`bidirectional_range`である様な型`R`の終端イテレータ（番兵）に対しては何も規定しいないため、`bidirectional_range`だからと言って`common_range`であるわけではありません。

### 戻り値型の制約

ここでは、`requires`式の新しい構文が出現しています。

```cpp
requires(I i) {
  { --i } -> same_as<I&>;
  { i-- } -> same_as<I>;
};
```

`requires`式内部の`{ expr } -> C;`のような構文で、これは`expr`に指定される式が利用可能であることをまずチェックし、利用可能であるならばその戻り値型がコンセプト`C`を満たすかをチェックしています。その際特徴的なのは、`expr`の結果型を`C`に対して自動的に補ってくれることで、`->`の後ろのコンセプト`C`は実際には`C<decltype((expr))>`のように実体化されます。

ここで現れている`same_as<T, U>`コンセプトは`T, U`が共に同じ型であることを定義するコンセプトで、本来は2引数のコンセプトですが引数は1つしか指定されていません。この場合、その第1引数に補完が行われ、例えば1つ目の制約式では`same_as<decltype((--i)), I&>`のようになっています。`decltype((expr))`は`expr`の結果型を式として参照するために`()`を重ねており、式の値カテゴリを適切に反映した型を取得します。

このような制約式の構文を、戻り値型の制約と呼びます。

戻り値型の制約で行われるようなコンセプト制約の略記法は他のところでもほぼ同様に行われることがあるため、多くのコンセプトは第1引数にチェックしたい型を取るように定義されていることが多いです。例えば、変換可能性を定義する`convertible_to<From, To>`コンセプトは`{ --i } -> convertible_to<int>;`のように、`From -> To`への変換を制約します。このことを意識すると、コンセプトを書くときにより使いやすく書くことができるでしょう。

## `random_access_range`

`random_access_range`はそのイテレータが*random access iterator*であるような`range`を定義するコンセプトです。

```cpp
template<class T>
  concept random_access_range =
    bidirectional_range<T> && random_access_iterator<iterator_t<T>>;
```

意味論要件はなく、構文的には見たままです。

`random_access_iterator`コンセプトはその名の通り*random access iterator*を定義するコンセプトです。

```cpp
template<class I>
  concept random_access_iterator =
    bidirectional_iterator<I> &&
    derived_from<ITER_CONCEPT(I), random_access_iterator_tag> &&
    totally_ordered<I> &&
    sized_sentinel_for<I, I> &&
    requires(I i, const I j, const iter_difference_t<I> n) {
      { i += n } -> same_as<I&>;
      { j +  n } -> same_as<I>;
      { n +  j } -> same_as<I>;
      { i -= n } -> same_as<I&>;
      { j -  n } -> same_as<I>;
      {  j[n]  } -> same_as<iter_reference_t<I>>;
    };
```

`iter_difference_t<I>`の示す型`D`、`D`の値`n`、型`I`の有効なイテレータ`a`と`++a`を`n`回適用したイテレータ`b`について

- `(a += n)`は`b`と等値（*equal*）
- `addressof(a += n)`は`addressof(a)`と等値
    - `+=`は`*this`を返す
- `(a + n)`は`(a += n)`と等値
- `D`の2つの正の値`x, y`について`(a + D(x + y))`が有効ならば、`(a + D(x + y))`は`((a + x) + y)`と等値
- `(a + D(0))`は`a`と等値
- `(a + D(n - 1))`が有効ならば、`(a + n) `は`[](I c){ return ++c; }(a + D(n - 1))`と等値
- `(b += D(-n))`は`a`と等値
- `(b -= n)`は`a`と等値
- `addressof(b -= n)`は`addressof(b)`と等値
    - `-=`は`*this`を返す
- `(b - n)`は`(b -= n)`と等値
- `b`が間接参照可能ならば、`a[n]`は有効であり`*b`と等値
- `bool(a <= b) == true`

`totally_ordered`は全順序による比較を、`sized_sentinel_for`は`sentinel_for`かつイテレータと番兵の引き算によって範囲の長さを求められることをそれぞれ定義しているコンセプトです。`requires`式内では*random access iterator*に通常期待される整数値との足し引きによる進行操作を定義しています。

そして、新しく加わったそれらの進行操作がどういう意味を持つのか、どう振る舞うのかを意味論要件が定義します。`iter_difference_t<I>`というのはイテレータ型の距離型（`defference_type`）を取得するエイリアステンプレートで、`+=`による進行は同じ数インクリメントしたのと同じであるとか、`+`と`+=`は進行という意味では同じであるとか、同様の事が`- -=`とデクリメントの間にも言えるだとか、イテレータの大小比較はイテレータの参照位置に基づくということなどを言っています。

C++17からは求められる事が若干厳しくなっており、C++17までの*random access iterator*はこのコンセプトを満たす事ができない場合があります。逆にC++20*random access iterator*はC++17*random access iterator*に対してほぼ後方互換性があります。

## `contiguous_range`

`contiguous_range`はそのイテレータが*contiguous iterator*であるような`range`を定義するコンセプトです。

```cpp
template<class T>
  concept contiguous_range =
    random_access_range<T> && contiguous_iterator<iterator_t<T>> &&
    requires(T& t) {
      { ranges::data(t) } -> same_as<add_pointer_t<range_reference_t<T>>>;
    };
```

型`T&`の値`t`について

- `to_address(ranges::begin(t)) == ranges::data(t)`

`contiguous_range`は`random_access_iterator`でもあり、かつそのイテレータが`contiguous_iterator`であることに加えて、`ranges::data(t)`という操作によってその範囲の存在するメモリ領域を指すポインタを取得することができます。そして、そのようなポインタ型はイテレータの要素型と一貫しており、取得されるポインタ値は先頭イテレータのアドレスと一致します。

*random access iterator*までは知っていると思われますが、*contiguous iterator*というのは初めて聞くかもしれません。これはイテレータの指す範囲がメモリ上で連続している場合のイテレータを表すものです（普通の配列におけるポインタなど）。概念だけがC++17で導入され、C++20でコンセプト導入とともに実体を持ちました。

*contiguous iterator*はそのまま`contiguous_iterator`コンセプトが定義しています。

```cpp
template<class I>
  concept contiguous_iterator =
    random_access_iterator<I> &&
    derived_from<ITER_CONCEPT(I), contiguous_iterator_tag> &&
    is_lvalue_reference_v<iter_reference_t<I>> &&
    same_as<iter_value_t<I>, remove_cvref_t<iter_reference_t<I>>> &&
    requires(const I& i) {
      { to_address(i) } -> same_as<add_pointer_t<iter_reference_t<I>>>;
    };
```

`a, b`を間接参照可能なイテレータ、`c`を間接参照不可能なイテレータとして、`b`は`a`から、`c`は`b`からそれぞれ到達可能であるとする。そのような型`I`のイテレータ`a, b, c`と`D = iter_difference_t<I>`について

- `to_address(a) == addressof(*a)`
- `to_address(b) == to_address(a) + D(b - a)`
- `to_address(c) == to_address(a) + D(c - a)`

構文的要件は、間接参照の結果が左辺値参照となることとイテレータの値型と参照型が一貫していることを要求し、新しい操作としてイテレータに対する`to_address()`を定義しています。

意味論要件は`to_address()`による操作の意味を定義していて、`to_address()`によってイテレータから得られるポインタ値はその参照する要素のアドレスであり、そのポインタ値を使った範囲の移動とイテレータによる進行とが一致することを定義します。

`std::addressof()`は引数のオブジェクトのアドレスを取得するものであり、`std::to_address()`はポインタも含めたポインタとみなせるもの（ファンシーポインタと呼ばれる）から格納するポインタ値（アドレス）を取得するものです。逆説的に、`std::to_address()`が適用可能であるという事はそれはほとんどポインタとみなせるものであるので、*contiguous iterator*とはほとんどポインタそのものの事を指しています。

ただし、`std::to_address()`は`pointer_traits`か`operator->`のどちらかを利用して格納するポインタ値を取得するものなので、`operator->`を備えた非ポインタの自作イテレータを作成することは出来ます。とはいえその時でも、*contiguous iterator*はポインタの極薄いラッパとなるでしょう。

ここでも、性質は`range`のイテレータ型だけについて定義されています。結局、`range`のカテゴリが何であれ、それは`common_range`である事とは無関係です。

## `view`

Rangeライブラリの影の主役、*View*と呼ばれるものを定義しているのが`view`コンセプトです。

```cpp
template<class T>
  concept view =
    range<T> &&
    movable<T> &&
    enable_view<T>;
```

- `T`のムーブ構築/代入は定数時間
- `T`のデストラクトは定数時間
- `T`はコピー不可であるか、`T`のコピー構築/代入は定数時間

構文的には、ムーブ構築・代入が可能な`range`です。これだけだと普通のコンテナも当てはまっていますが、`std::vector`が`view`であると思う人は多分居ないでしょう。

意味論要件は、コピーやムーブ、破棄などの操作の計算量が全て定数時間であることを言っており、`view`はその表現する範囲の要素数と無関係にコピーやムーブなどの基本操作が可能であることを意味しています。これはすなわち、`view`となる`range`はその表す範囲を所有せずに参照していることを意味しています。範囲を所有している場合、通常は定数時間での破棄は不可能です。

このように、`view`コンセプトの定義の大部分は意味論要件によってなされており、構文的な制約だけでは通常のコンテナも`view`となってしまいます。そのため、`view`コンセプトはデフォルトでは殆どの型について無効化されており、オプトインでの有効化が必要になります。それは構成する最後の制約式、`enable_view`変数テンプレートによって行います。

```cpp
namespace std::ranges {

  struct view_base {};

  // デフォルトの定義
  template<class T>
  inline constexpr bool enable_view = derived_from<T, view_base>;
}


// 自作viewクラス
struct my_view { ... };

// 例えば、変数テンプレートの明示的特殊化によって有効化
inline constexpr bool std::ranges::enable_view<my_view> = true;
```

この利用法は`enable_borrowed_range`等で見慣れたでしょうか。`enable_view`が`view`コンセプトの最後の一押しとなることで、普通の`range`がむやみやたらに`view`と構文的にみなされてしまう事を防いでいます。

`view_base`というのは空のタグ型で、これを継承しておけば自動的に`view`コンセプトを有効化することができます。標準ライブラリの`view`は全てこの型を継承しており、これはユーザー定義型でも利用できます。ただ、おそらくこの型を直接使う事はほぼ無く、`std::ranges::view_interface`という型を通じて利用することになるでしょう（これは後程紹介します）。

標準ライブラリにある`view`には`std::string_view`や`std::span`があります。これらのものは範囲（文字列orメモリ列）を参照しているだけの`range`で、まさに`view`の代表的な例です。どちらに対しても`enable_view`の特殊化が予め用意されており、`view`コンセプトが有効化されています。

### `view`と`borrowed_range`

`view`と`borrowed_range`はどちらも範囲を所有せず参照しているだけの`range`を表しており、同じことを別の方向から表現しているようにも見えます。

`borrowed_range`は定義にあるように、単に左辺値参照型でもモデルとなることができます。そのことと意味論要件からも`borrowed_range`そのものは何か役割を持つものではなく、`range`のごく薄い参照ラッパであることしか求められていません。一方、`view`はムーブが求められ、さらにコピー、デストラクタ等の操作の計算量が規定されているなど明らかにクラス型を指定しており、`view`自身も`range`としての役割を持つことを意図しています。つまり、`borrowed_range`と比べると厚い参照ラッパに見えます。

`borrowed_range`は一時的に`range`を「借用」してくるだけのものです。`range`を所有したくないけれど元の`range`をそのまま利用したい場合に使用できる型を表します。たとえば関数の引数として`range`を取るときに使用することができるでしょう。

`view`は元となる`range`全体を包み込み別の`range`へと変換するようなクラス型を表します。元の`range`をベースとしつつ隠蔽し、別の特徴をもつ`range`として扱うことができます。この観点では、`std::string_view`や`std::span`は`view`としては少し地味かもしれません。

まだピンとこないかと思いますが、後程紹介するRangeアダプタと呼ばれる`view`群を見て行くと`view`の気持ちを理解できるかと思います。

## `viewable_range`

`viewable_range`コンセプトは、`view`に安全に変換できる`range`を定義するコンセプトです。

```cpp
template<class T>
  concept viewable_range =
    range<T> &&
    (borrowed_range<T> || view<remove_cvref_t<T>>);
```

意味論要件はなく、`viewable_range`コンセプトのキモは後半の`||`で繋がれた2つの制約式にあります。簡単には、この制約式は右辺値の`range`を弾くためにあります。

```cpp
// どちらも、なにかview型であるとする
template<range R>
class unsafe_view { ... };

template<viewable_range R>
class safe_view { ... };

int main() {
  /// 右辺値からの構築

  // 構築できてしまう、v1の利用は未定義動作
  unsafe_view v1{std::vector<int>{1, 2, 3, 4, 5}};
  // コンパイルエラー！
  safe_view v2{std::vector<int>{1, 2, 3, 4, 5}};


  /// 左辺値からの構築
  std::vector vec = {1, 2, 3, 4, 5}; 

  // ともにok、安全に利用できる
  unsafe_view v1{vec};
  safe_view   v2{vec};
}
```

この様に、自身で範囲を所有しているタイプの`range`の右辺値から`view`を構築してしまうと、構築直後に範囲は寿命を迎え、以降未定義動作の世界に突入します。ただし、右辺値の`range`が`view`もしくは`borrowed_range`であるならば、それらの寿命と参照している範囲の寿命は無関係なので、問題なく`view`を構築し利用できます。「`view`に安全に変換できる」とは、こうした意味合いです。

`view`とは`range`を受けて別の`range`へ変換するものであることが、この`viewable_range`の役割からも読み取ることができます。

ただし、構築しようとする`view`の参照しようとしている`borrowed_range`（あるいは`view`）が参照している大本の`range`の寿命が別の所で終了する（している）ような場合は、このコンセプトでもどうしようもありません。

```cpp
auto f() {
  std::vector vec = {1, 2, 3, 4, 5}; 
  
  // 問題なく構築できる
  safe_view v1{vec};

  // 問題なく構築できる、が・・・
  return safe_view{std::move(v1)};
}

int main() {
  // 未定義動作の世界へようこそ！
  auto view = f();
}
```

この様な恣意的な例以外でも、たとえば並列処理において大本の`range`が別のスレッドに所有されていたりする場合などにも起こり得ます。Rustならば型レベルでこのような問題を阻止できるようですが、C++ではこれを防ぐことはできません。コンパイラのコード分析や各種サニタイザなどを用いれば検出できるかもしれませんが・・・

\clearpage

# Rangeアクセス関数

C++17においてコンテナアクセスの共通操作として`std`名前空間の下に追加されていたグローバル関数テンプレートに対応するものが、Rangeライブラリにも`std::ranges`名前空間の下に用意されています。これは一見すると同じものが重複して存在しているように見えてしまいますがそうではなく、コンセプトによって制約され、同じ意味を持ついくつかの操作をさらに統一化している、よりジェネリックで安全かつ簡単にその操作の目的を達成するものです。

ここではそれらのものをRangeアクセス関数として纏めて扱います。Rangeアクセス関数の実態は関数ではなく関数オブジェクトであり、カスタマイゼーションポイントオブジェクト（*Customization Point Object* : CPO）と呼ばれます。カスタマイゼーションポイントオブジェクトはADLを阻害することで意図しない関数呼び出しを防止し、その内部で複雑な判定とディスパッチを行ってその目的を果たす非常にテクニカルなものです。

## 従来のカスタマイゼーションポイント関数の問題点

C++17までにカスタマイゼーションポイントとなっていた関数（例えば`std::begin()/std:::end(), std::swap()`など）にはアダプトして動作をカスタマイズするためのいくつかの方法が用意されており、より柔軟に自分が定義した型を適合できるようになっています。しかしその一方で、それによって使用するときに少し複雑な手順を必要としていました。例えば`std::begin()/std::end()`で見てみると

```cpp
// イテレータのペアを取り出したい
template<typename Range>
auto my_algo(Range&& range_obj) {
  // stdのものをusingしてから
  using std::begin;
  using std::end;

  // 修飾なしで呼び出す
  auto it = begin(range_obj);
  auto end = end(range_obj);
  // ...
}
```

真にジェネリックに書くためにはこのように「`std::begin()/std::end()`を`using`してから、`begin()/end()`を名前空間修飾なしで呼び出す」という風に書くことで、`std`名前空間のもの及び配列には`std::begin()`が、ユーザー定義型に対してはADLで使用可能な`begin()`あるいは`std::begin()`を通してメンバ関数の`begin()`が呼び出されるようになります。しかし、手順1つ間違えただけでこのコードはたちまち汎用性を失います。

C++17まではイテレータペアをジェネリックに取得するにはこうする事がベストでした。しかし、このコードがなぜ必要なのかは難しい問題であり、理解せずに書いていると書き忘れたり消してしまったりしかねません。このようなよく使う操作に罠が潜んでいることは、C++初学者にも優しくありません。

この問題は、標準ライブラリにおいてカスタマイゼーションポイントとして定義されている関数すべてに共通することです。例えば、`std::size`や`std::data`などがあります。

`range`ライブラリで新しく導入されたRangeアクセス関数と呼ばれる関数群は、この問題を解決するために導入されています。つまり、Rangeアクセス関数では先程のような呼び出しは1行で書くことができます。

```cpp
// イテレータのペアを取り出したい
template<typename Range>
auto my_algo(Range&& range_obj) {
  // 先程と同じことを達成し、よりジェネリック、かつ安全！
  auto it = std::ranges::begin(range_obj);
  auto end = std::ranges::end(range_obj);

  ...
}
```

## 従来のカスタマイゼーションポイント関数の問題点2

従来のカスタマイゼーションポイント関数の問題点はもう一つあります。それは、実質的に名前を予約してしまっていることです。

カスタイマイゼーションポイントは標準ライブラリをよりジェネリックにするために不可欠な存在ですが、標準ライブラリはそのカスタマイゼーションポイントの名前（関数名）だけに着目して呼び出しを行うため、同名の全く異なる意味を持つ関数が定義されていると未定義動作に陥ります。特に、ADLが絡むとこれは発見しづらいバグを埋め込む事になるかもしれません。したがって、カスマイゼーションポイントを増やすと言う事は実質的に予約されている名前が増える事になり、ユーザーは注意深く関数名を決めなければならないなど負担を負うことになります。

C++20からのコンセプトはそのような問題を解決します。その呼び出しにおいてコンセプトを用いて対象の型が制約を満たしているかを構文的にチェックするようにし、カスタマイゼーションポイントに不適合な場合はオーバーロード候補から外れるようにする事で、ユーザーがカスタマイゼーションポイントとの名前被りを気にしなくても良くなります。結果的に、標準ライブラリにより多くのカスタマイゼーションポイントを設ける事ができるようにもなります。

しかし、コンセプトによって制約されたC++20カスタマイゼーションポイント関数の下では、先程のC++17までのベストプラクティスコードがむしろ最悪のコードになってしまうのです。

```cpp
namespace std {

  // コンセプトによって制約された新しいbegin()関数
  template<range C>
  constexpr auto begin(C& c) -> decltype(c.begin());  // (1)
}

namespace myns {

  struct my_struct {};

  // イテレータを取得するものではないbegin()関数
  bool begin(my_struct&);  // (2)

  // end()は省略
}


template<typename Container>
void my_algo(Container&& rng) {
  using std::begin;

  // 先頭イテレータを得る、はずが・・・
  auto first = begin(rng);  // my_structに対しては(2)が呼び出される
}

int main() {
  myns::my_struct st{};

  my_algo(st);  // ok、呼び出しは適格
}
```

このように、大本の関数がコンセプトで制約されていたとしてもADL経由で制約を満たさない`begin()`が呼ばれています。別の見方をすれば、コンセプトによる制約を簡単に迂回できてしまっています。これはADLを用いる以上仕方のない事ではあるのですが、この場合にADLで呼び出される関数にまで制約を強制しようとすると少し込み入った実装が必要となります。また、コンセプトによる制約を行うことで、C++17との互換性の問題が発生しえます。結局、ユーザーはカスタマイゼーションポイントの名前をきちんと意識してコードを書かなければならなくなってしまいます。

Rangeアクセス関数はこの問題も同時に解決しています。

```cpp
namespace std {
  // Rangeアクセス関数（オブジェクト）
  inline constexpr begin_t begin{}; // (1)
}

namespace myns {

  struct my_struct {};

  // イテレータを取得するものではないbegin()関数
  bool begin(my_struct&);  // (2)
}


template<typename Container>
void my_algo(Container&& rng) {
  using std::ranges::begin;

  auto first = begin(rng);  // (1)が呼び出されコンパイルエラー
}
```

## カスタマイゼーションポイントオブジェクト（*Customization Point Object* : CPO）

まとめると、問題点は次の3点です。

1. 正しい呼び出しが難しい、煩雑
2. 名前を予約してしまっている
3. コンセプトによる制約を行うことが難しい

カスタマイゼーションポイントオブジェクトはこれらの問題をすべて解決するソリューションであり、Rangeアクセス関数は全てカスタマイゼーションポイントオブジェクトとして実装されます。

1. 正しい呼び出しが難しい、煩雑
      - その内部で最適な関数を呼び出すため、ユーザーは気にしなくていい
2. 名前を予約してしまっている
      - コンセプトによるチェックが行われることで軽減される
3. コンセプトによる制約を行うことが難しい
      - 自身が関数オブジェクトであることによってADLを阻害するため、チェックを回避できない

1番目の問題の解決の詳細は以降で個別に見ていきます。2番目の問題は3番目の問題が解決されることで解決されています。

3番目の解決は少し複雑です。C++における名前探索では修飾名探索と非修飾名探索の2つがまず実行され、そこで名前に対応するものが見つからない場合にADLが実行されます。ADLは関数と思われる名前に対しての名前探索方法で、一番最後に適用されます。ある名前が関数ではないことが分かっているとき、すなわちADLの前に名前に対応する関数ではないものが見つかっているとき、ADLは行われません。

カスタマイゼーションポイントオブジェクトは関数オブジェクトであり、その名前は変数名です。最初の2つの名前探索でCPOが発見されると、ADLは行われなくなります。これによって、CPOの呼び出しはADLによって迂回することができなくなり、その内部のあらゆるチェックは常に機能するようになります。

Rangeアクセス関数はCPOとして、従来のカスタマイゼーションポイント関数とは別に`std::ranges`名前空間に定義されています。従来のカスタマイゼーションポイント関数はそのままに新しく用意されているため互換性の問題もおこらず、利用先を切り替えるだけでCPOの利点を享受することができるようになっています。

## イテレータ/番兵の取得

コンテナなどからその要素の先頭イテレータと終端イテレータを取得する操作は、通常メンバ関数の`.begin()/.end()`で行います。配列も含めてよりジェネリックに扱いたいときは、`std::begin()/std::end()`を用いていました。

Rangeライブラリではこれらの操作に対応するものとして、`std::ranges::begin/std::ranges::end`を利用することができます。

```cpp
namespace std::ranges {
  inline namespace unspecified {
    inline constexpr unspecified begin = unspecified;
    inline constexpr unspecified end = unspecified;
  }
}
```

`unspecified`というのは文字通り名前空間名やクラス名、初期化の方法が未規定という意味です。利用する時はこれらの事に依存しないようにする必要があります。

この`std::ranges::begin/std::ranges::end`はカスタマイゼーションポイントオブジェクトであり、呼び出しは通常の関数と同じように行えます。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4};
  int arr[] = {1, 2, 3, 4};

  // 範囲の先頭イテレータを取得する
  auto vit = std::ranges::begin(vec);
  auto ait = std::ranges::begin(arr);

  // 範囲の終端イテレータ（番兵）を取得する
  auto vend = std::ranges::end(vec);
  auto aend = std::ranges::end(arr);
}
```

これだけだと`std::begin()/std::end()`と何が違うのかわかりませんが、その効果には大きな違いがあります。

型`T`のオブジェクト`r`によって`std::ranges::begin(r)`のように呼び出されたとき、その効果は次のようになります

1. `r`が右辺値であり、かつ`enable_borrowed_range<remove_cv_t<T>> == false`の場合、*ill-formed*
2. `T`が配列型であり、`remove_all_extents_t<T>`が不完全型を示す場合、*ill-formed*
3. `T`が配列型なら、`r + 0`
      - 先頭のポインタを返す
4. `decay-copy(r.begin())`が呼び出し可能であり、その戻り値型が  
   `input_or_output_iterator`コンセプトのモデルとなるなら、`decay-copy(r.begin())`
      - メンバ関数の`begin()`を呼び出す 
5. `T`はクラス型か列挙型であり、`decay-copy(begin(r))`が呼び出し可能であり、その戻り値型が`input_or_output_iterator`コンセプトのモデルとなるなら、`decay-copy(begin(r))`
      - ADLによって`begin()`を呼び出す
      - その際、ADLより前に`std`名前空間にある`std::begin()`を発見しないように名前解決が行われる
6. それ以外の場合、*ill-formed*

`r`が右辺値であるとき、`r`を転送する所では`std::forward`を適切に使用したときと同様の完全転送が行われます。

とても複雑ですが、何か結果を返すのは3, 4, 5番目に該当する場合です。そこでは確実にイテレータを返すようになっており、`std::ranges::begin(r)`の呼び出しが有効であるときの戻り値型は`input_or_output_iterator`コンセプトのモデルとなります（`input_or_output_iterator`は何らかのイテレータとしての最小要件を定義するコンセプトです）。なお、`decay-copy()`というのは参照/CV修飾を落としてコピーを行う操作を表しているもので、実際にこの様な名前の関数が存在するわけではありません。イテレータがムーブオンリーかつ参照で取得された場合、`decay-copy`操作はコンパイルエラーとなります。

そして、同じように`std::ranges::end(r)`と呼び出されたとき、その効果は次のようになります

1. `r`が右辺値であり、かつ`enable_borrowed_range<remove_cv_t<T>> == false`の場合、*ill-formed*
2. `T`が配列型であり、`remove_all_extents_t<T>`が不完全型を示す場合、*ill-formed*
3. `T`が要素数不明の配列型である場合、*ill-formed*
4. `T`が配列型なら、`r + std::extent_v<T>`
      - 終端のポインタを返す
5. `decay-copy(r.end())`が呼び出し可能であり、その戻り値型`E`が`sentinel_for<E, iterator_t<T>>`コンセプトのモデルとなるなら、`decay-copy(r.end())`
      - メンバ関数の`end()`を呼び出す 
6. `T`はクラス型か列挙型であり、`decay-copy(end(r))`が呼び出し可能であり、その戻り値型`E`が`sentinel_for<E, iterator_t<T>>`コンセプトのモデルとなるなら、`decay-copy(end(r))`
      - ADLによって`end()`を呼び出す
      - その際、ADLより前に`std`名前空間にある`std::end()`を発見しないように名前解決が行われる
7. それ以外の場合、*ill-formed*

こちらも`r`が右辺値であるとき、`r`を転送する所では`std::forward`を適切に使用したときと同様の完全転送が行われます。

ほとんど`ranges::begin()`と対になっており、`std::ranges::end(r)`の呼び出しが有効であるときの戻り値型`S`は、`std::ranges::begin(r)`の戻り値型を`I`として`sentinel_for<S, I>`コンセプトのモデルとなります（`sentinel_for<S, I>`はイテレータ型`I`に対する番兵型`S`を定義するコンセプトです）。

このように、`std::ranges::begin/std::ranges::end`はイテレータと番兵を様々な方法で取得してくるものです。その試行範囲は従来の`std::begin/std::end`よりも広く、そしてコンセプトによって確実かつ安全にイテレータと番兵を取得できるようになっています。

以降紹介するRangeアクセス関数もこれと同様、より探索範囲を広くしつつコンセプトによって確実に目的を達成できるように、従来あったものからリファインされています。C++20以降、特に後方互換の問題が無ければ、Rangeアクセス関数を優先的に使用することが推奨されます。

### *expression-equivalent*

規格書では、CPOの効果がある場合にある式と同じ効果であることを*expression-equivalent*という言葉で指定しています。

例えば先程の`ranges::begin`では

> `T`が配列型なら、`r + 0`

は正確には

> `T`が配列型なら、`ranges::begin(r)`は`r + 0`と*expression-equivalent*

のようになります。

「式Aは式Bと*expression-equivalent*」という言葉の意味は、式Aの効果と例外を投げるかどうかおよび定数式で使用可能かどうかは、式Bと同一である、という意味です。

つまり配列型に対する`ranges::begin(r)`の場合は、その効果は`r + 0`であり、呼び出しが`noexcept`かどうかは`noexcept(r + 0)`の真偽で決まり、定数式で使用可能であるかも`r + 0`が定数式で使用可能かどうかで決まる、という事です。なお、配列型の場合は常に`noexcept`であり定数式で使用可能です。

これはむしろ4番目と5番目の場合のように、ユーザーが定義した関数（と型）を呼び出す際に特に重要です。そのような場合の`ranges::begin(r)`は`r`で使用可能な`begin`の呼び出しとイテレータ型のコピーが例外を投げるかどうか、定数式で使用可能かどうか、に準じる形でそれらの事が決まります。

CPOの効果は全て*expression-equivalent*で指定されていますが、本書では*expression-equivalent*という言葉は常に省略しています。そのため、以降CPOを見ていく場合に`constepxr`で使用可能かどうかと例外を投げるかどうかが気になった時は、*expression-equivalent*を思い出して、どんな型について使用したいのかによって処理を追いかけてチェックしてみてください。

### Poison-pillオーバーロード

> その際、ADLより前に`std`名前空間にある`std::begin()`を発見しないように名前解決が行われる

ADLによって関数を探すケースのこの条件は`std::begin`を常に使用しないように読めてしまいますがそうではありません。標準ライブラリのCPOは全て`std`名前空間内にあるため、`begin(r)`という呼び出しは`std::begin`を一番最初に発見してしまいます。それだとユーザー定義の`begin`の利用を妨げることになりかねないため、ADLの前に`std::begin`を隠蔽する候補を含んでおくことで、`std::begin`はADLによって発見されたときのみ使用されることとなります。

簡易な実装で見てみると、次のようになります

```cpp
namespace std {

  // std::begin (1)
  template <class C>
  constexpr auto begin(C& c) -> decltype(c.begin()); 

  namespace ranges {
    namespace begin_impl {

      // Poison-pillオーバーロード (2)
      void begin(auto&) = delete;
      void begin(const auto&) = delete;

      // ranges::begin（簡易）実装クラス
      struct begin_fn {

        // ADLケースのオーバーロード
        template<typename T>
        constexpr auto operator()(T&& r) const noexcept(noexcept(begin(std::forward<T>(r))))
          // 後置requires節による制約
          requires input_or_output_iterator<decay_t<decltype(begin(std::forward<T>(r)))>>
        {
          // 非修飾名探索では(2)が見つかり、(1)を隠蔽する
          // その後、std名前空間に対するADLでは(1)が見つかる
          return begin(std::forward<T>(r));  // (3)
        }

        // 他は省略
      };
    }

    inline namespace cpo_begin {
      // ranges::begin CPO定義
      inline constexpr begin_impl::begin_fn begin{};
    }
  }
}
```

(3)の地点での`begin(r)`の呼び出しでは、修飾名探索と非修飾名探索の後でADLが発動します。修飾名探索は何も見つからないはずで、非修飾名探索では(2)を発見しますが(1)を発見しません。その後ADLによって更なる探索が行われ、`T`が`std::`の型（たとえば`std::vector`）であるときにのみ(1)を発見し、オーバーロード解決によって(1)を使用します。

この時、もし(2)の`delete`宣言が無いと(1)を発見してしまいそちらが呼ばれてしまいます。この様に、上位の名前空間にある同名関数を毒殺するテクニックをPoison-pillオーバーロードと呼びます。

また、この簡易な例ではCPOの実装の一部を垣間見ることができます。関数呼び出し演算子オーバーロードとコンセプトによる制約によって引数型に応じて最適な処理を選択し、*expression-equivalent*は`constexpr`関数と`noexcept(noexcept(...))`によって自動的に判定され、`auto`戻り値型によって`decay_copy`が処理されています。

ここで用いられている後置`requires`節は関数を制約するための方法の一つて、`requires`式を関数宣言の一番最後（例では`noexcept`の後）に置くものです。この方法のメリットは後置戻り値型と同様に関数の仮引数を使用することができることと、クラステンプレートのテンプレートではないメンバ関数に対して制約することができることです。

```cpp
// 後置requiresの単純な形式例
template<typename T>
auto f(T t) const & noexcept -> T requires{/* ... */};
```

後置でも前置でも`requires`節としての扱いは変わらないため、この中で`requires`式を定義することや任意の`bool`型の定数式を使用することなどができます。

### `cbegin/cend`

`begin/end`に対する`cbegin/cend`は、`const`イテレータ/番兵を取得するものです。`const`イテレータではその参照先の要素も`const`となるため変更することができなくなります。

```cpp
namespace std::ranges {
  inline namespace unspecified {
    inline constexpr unspecified cbegin = unspecified;
    inline constexpr unspecified cend = unspecified;
  }
}
```

型`T`のオブジェクト`r`によって`ranges::cbegin(r)`のように呼び出されたとき、その効果は次のようになります

1. `r`が左辺値ならば、`ranges::begin(static_cast<const T&>(r))`
2. それ以外の場合、`ranges::begin(static_cast<const T&&>(r))`

同じように`ranges::cend(r)`のように呼び出されたとき、その効果は次のようになります

1. `r`が左辺値ならば、`ranges::end(static_cast<const T&>(r))`
2. それ以外の場合、`ranges::end(static_cast<const T&&>(r))`

どちらにおいても`r`が右辺値であるとき、`r`を転送する所では`std::forward`を適切に使用したときと同様の完全転送が行われます。

`ranges::cbegin(r)`の呼び出しが有効な時の戻り値型`I`は`input_or_output_iterator`コンセプトのモデルとなり、`ranges::cend(r)`の呼び出しが有効であるときの戻り値型`S`は`sentinel_for<S, I>`コンセプトのモデルとなります。

これは従来の`std::cbegin()/std::cend()`フリー関数がやっていたこととさほど変わりありません。`const`イテレータは`const`なオブジェクトに対する`begin/end`によって取得されます。

#### `c`付き`begin/end`の注意点

`ranges::cbegin/ranges::cend`は受け取った引数を`const`化したうえで、その`const`オブジェクトによる`ranges::begin/ranges::end`によって`const`イテレータを取得しようとします。このため、`const`版の`begin/end`が`const`イテレータを返さない場合、`ranges::cbegin/ranges::cend`は`const`イテレータではなく通常の可変イテレータを返してしまいます。

とはいえ標準ライブラリのコンテナ型にはそのような型はありませんし、コンテナ型では問題にはならないはずです。問題となるのは、型に対する`const`性とその要素の`const`性が同期しないタイプの`range`、`view`と呼ばれる`range`に対してであり、特に`std::span`で問題になります。

まさに、`std::span`に対する`ranges::cbegin/ranges::cend`は、`std::span`の参照する要素列の可変イテレータを返します。

```cpp
std::vector<int> coll{1, 2, 3, 4, 5};
std::span<int> sp{coll.data(), 3};

for (auto it = std::ranges::cbegin(sp);
          it != std::ranges::cend(sp); ++it)
{
  *it = 42; // コンパイルエラーにならない・・・
}
```

Rangeライブラリで追加された`view`の中には`const`で反復可能ではないために`begin()`の`const`オーバーロードを提供しないものや、場合によっては利用できないものがあり、それらでも同じことが起き得ます。

`std::span`も`view`であり、`view`は単に`range`を参照するだけのもので、`view`オブジェクト自身に対する`const`は参照先の`range`要素の`const`を意味しません。一方、コンテナ型は`range`を所有しているため、自身に対する`const`はすなわち要素の`const`となります。

とはいえこの事は直感的ではなく、気付き辛く使いづらい挙動です。この問題は標準化委員会にも認識されておりいくつか改善のための提案は出ていますが、今の所採択に至った改善案はありません。`view`に対する`const`イテレート時には、少し気を払う必要があります。

### `rbegin/rend`

`begin/end`に対する`rbegin/rend`は、逆イテレータ/番兵を取得するものです。逆イテレータは範囲を逆順で走査するためのイテレータです。

範囲を逆順に走査するためには、引数となる`range`は`bidirectional_range`でなければなりません。

```cpp
namespace std::ranges {
  inline namespace unspecified {
    inline constexpr unspecified rbegin = unspecified;
    inline constexpr unspecified rend = unspecified;
  }
}
```

型`T`のオブジェクト`r`によって`ranges::rbegin(r)`のように呼び出されたとき、その効果は次のいずれかになります

1. `r`が右辺値であり、かつ`enable_borrowed_range<remove_cv_t<T>> == false`の場合、*ill-formed*
2. `T`が配列型であり、`remove_all_extents_t<T>`が不完全型を示す場合、*ill-formed*
3. `decay-copy(r.rbegin())`が呼び出し可能であり、その戻り値型が  
   `input_or_output_iterator`コンセプトのモデルとなるなら、`decay-copy(r.rbegin())`
      - メンバ関数の`rbegin()`を呼び出す 
4. `T`はクラス型か列挙型であり、`decay-copy(rbegin(r))`が呼び出し可能であり、その戻り値型が`input_or_output_iterator`コンセプトのモデルとなるなら、`decay-copy(rbegin(r))`
      - ADLによって`rbegin()`を呼び出す
      - その際、ADLより前に`std`名前空間にある`std::rbegin()`を発見しないように名前解決が行われる
5. `ranges::begin(r)`と`ranges::end(r)`が呼び出し可能であり、その戻り値型が同じ型で`bidirectional_iterator`のモデルであるなら、`make_reverse_iterator(ranges::end(r))`
6. それ以外の場合、*ill-formed*

そして、同じように`ranges::rend(r)`と呼び出されたとき、その効果は次のいずれかになります

1. `r`が右辺値であり、かつ`enable_borrowed_range<remove_cv_t<T>> == false`の場合、*ill-formed*
2. `T`が配列型であり、`remove_all_extents_t<T>`が不完全型を示す場合、*ill-formed*
3. `decay-copy(r.rend())`が呼び出し可能であり、その戻り値型`E`が`sentinel_for<E, decltype(ranges::rbegin(r))>`コンセプトのモデルとなるなら、`decay-copy(r.rend())`
      - メンバ関数の`rend()`を呼び出す 
4. `T`はクラス型か列挙型であり、`decay-copy(rend(r))`が呼び出し可能であり、その戻り値型`E`が`sentinel_for<E, decltype(ranges::rbegin(r)>`コンセプトのモデルとなるなら、`decay-copyr(end(r))`
      - ADLによって`rend()`を呼び出す
      - その際、ADLより前に`std`名前空間にある`std::rend()`を発見しないように名前解決が行われる
5. `ranges::begin(r)`と`ranges::end(r)`が呼び出し可能であり、その戻り値型が同じ型で`bidirectional_iterator`のモデルであるなら、`make_reverse_iterator(ranges::begin(r))`
6. それ以外の場合、*ill-formed*

どちらにおいても`r`が右辺値であるとき、`r`を転送する所では`std::forward`を適切に使用したときと同様の完全転送が行われます。

`ranges::rbegin(r)`の呼び出しが有効な時の戻り値型`I`は`input_or_output_iterator`コンセプトのモデルとなり、`ranges::rend(r)`の呼び出しが有効であるときの戻り値型`S`は`sentinel_for<S, I>`コンセプトのモデルとなります。

基本的には`ranges::begin/ranges::end`とほとんど同じように`rbegin()/rend()`を探してきます。異なるのは配列型のケアを`ranges::begin/ranges::end`に委譲していることと、5番目のケースです。

5番目のケースはメンバに`rbegin/rend`を備えていないような`range`型に対するケアです。標準ライブラリではRangeライブラリにある各種`view`型が該当し、それが`common_range`であり`bidirectional_range`であれば、そのイテレータペアと`std::make_reverse_iterator()`を利用して逆イテレータのペアを取得します。

### `crbegin/crend`

`crbegin/crend`は、`const`逆イテレータ/番兵を取得するものです。`const`逆イテレータは`const`イテレータかつ逆イテレータであるイテレータです。

範囲を逆順に走査するために、引数となる`range`は`bidirectional_range`でなければなりません。

```cpp
namespace std::ranges {
  inline namespace unspecified {
    inline constexpr unspecified crbegin = unspecified;
    inline constexpr unspecified crend = unspecified;
  }
}
```

型`T`のオブジェクト`r`によって`ranges::crbegin(r)`のように呼び出されたとき、その効果は次のいずれかになります

1. `r`が左辺値ならば、`ranges::rbegin(static_cast<const T&>(r))`
2. それ以外の場合、`ranges::rbegin(static_cast<const T&&>(r))`

同じように`ranges::crend(r)`のように呼び出されたとき、その効果は次のいずれかになります

1. `r`が左辺値ならば、`ranges::rend(static_cast<const T&>(r))`
2. それ以外の場合、`ranges::rend(static_cast<const T&&>(r))`

どちらにおいても`r`が右辺値であるとき、`r`を転送する所では`std::forward`を適切に使用したときと同様の完全転送が行われます。

`ranges::crbegin(r)`の呼び出しが有効な時の戻り値型`I`は`input_or_output_iterator`コンセプトのモデルとなり、`ranges::crend(r)`の呼び出しが有効であるときの戻り値型`S`は`sentinel_for<S, I>`コンセプトのモデルとなります。

`ranges::cbegin/ranges::cend`と同様に、`const`化した引数に対して`ranges::rbegin/ranges::rend`を呼び出すことで`const`逆イテレータ/番兵を取得します。したがって、`ranges::cbegin/ranges::cend`同様に`const`イテレータを必ずしも得ることができない場合があります。

## 範囲サイズの取得

`ranges::size`は範囲のサイズ、すなわち長さを取得するものです。`ranges::ssize`はその長さを符号付整数型で取得し、`ranges::empty`は範囲が空かどうかを`bool`値で取得するものです。

```cpp
namespace std::ranges {
  inline namespace unspecified {
    inline constexpr unspecified size = unspecified;
    inline constexpr unspecified ssize = unspecified;
    inline constexpr unspecified empty = unspecified;
  }
}
```

型`T`のオブジェクト`r`によって`ranges::size(r)`のように呼び出されたとき、その効果は次のいずれかになります

1.  `T`が要素数不明の配列型である場合、*ill-formed*
2.  `T`が配列型である場合、`decay-copy(std::extent_v<T>)`
3.  `disable_sized_range<remove_cv_t<T>> == false`かつ`decay-copy(r.size())`が呼び出し可能であり、その戻り値型が整数型である場合、`decay-copy(r.size())`
        - メンバ関数の`size()`を呼び出す
4.  `T`はクラス型か列挙型であり、`disable_sized_range<remove_cv_t<T>> == false`かつ`decay-copy(size(r))`が呼び出し可能であり、その戻り値型が整数型である場合、`decay-copy(size(r))`
      - ADLによって`size()`を呼び出す
      - その際、ADLより前に`std`名前空間にある`std::size()`を発見しないように名前解決が行われる
5. `to-unsigned-like(ranges::end(r) - ranges::begin(r))`が有効な式であり、`ranges::begin(r)`の戻り値型`I`と`ranges::end(r)`の戻り値型`S`が`sized_sentinel_for<S, I>`と`forward_iterator<I>`のモデルとなる場合、`to-unsigned-like(ranges::end(r) - ranges::begin(r))`
6. それ以外の場合、*ill-formed*

基本的には`ranges::begin/ranges::end`と同様になんとか長さを求めようとします。特殊なのは5番目の場合で、これは`size()`を持たないけれどイテレータの引き算によってその距離を求めることができるような範囲を想定しています。

5番目のケースに見ることができるように、`std::size`と同様に`ranges::size`は計算量が定数時間であることを暗に求めています。それは明記されてはいませんが、`sized_range`コンセプトの意味論要件として見ることができ、そこでは少し緩められた償却定数時間という要件が課されています。これは例えば、一度目の`size()`の呼び出しをキャッシュしておいて返すような実装が許されます。

標準ライブラリのコンテナの中では、`forward_list`はそのフットプリント最小化のために長さを償却定数時間で求めることができないため、この関数で長さを得ることができません。その長さを求めたい場合、`std::distance`あるいは`ranges::distance`を用いてイテレータペアからO(N)で求めることになります。

### *integer-like*

`ranges::size(r)`の呼び出しが有効であるとき、その戻り値型は*integer-like*な型となります。実は、上記4,5のケースで呼ばれる`size()`の戻り値型も整数型ではなく*integer-like*な型であればOKです。

*integer-like*な型の定義は複雑ですが、簡単に言えば組み込みの整数型と殆ど同等に扱える型の事を指します。殆どの場合これは`int`型をはじめとする組み込みの整数型となるはずですが、`iota_view`など一部の`view`やユーザー定義の`range`では必ずしも整数型を返すとは限りません。たとえば、適切に実装された多倍長整数型を返すことが許されます。

そして、`to-unsigned-like(i)`は規格書において説明のために用いられている操作で、`i`の型`X`と同じ幅を持つ符号なし整数型に変換する操作です。つまりは5番目のケースは必ず符号なしの整数型で結果が得られるということです。

### `ssize/empty`

`ranges::ssize`は`ranges::size`に対して、結果を必ず符号付整数型で得られるものです。

型`T`のオブジェクト`r`によって`ranges::ssize(r)`のように呼び出されたとき、その効果は次のいずれかになります

1. `range_difference_t<T>`の幅が`ptrdiff_t`未満である場合、`static_cast<ptrdiff_t>(ranges::size(r))`
2. それ以外の場合、`static_cast<range_difference_t<T>>(ranges::size(r))`

多くの場合は`sized_range`な`r`に対して、1つ目の処理によって結果を返すことになるでしょう。

`range_difference_t<T>`というのは`range`型`T`からその距離型（`difference_type`）を取得するためのエイリアステンプレートです。`range_difference_t<T>`が示す型が符号付整数型でなければならないという要請は直接的には見えていませんが、`range_difference_t<T>`はイテレータ間の距離を表す型であり符号付整数型となるため、2つ目のケースでも結果は符号付整数型となります。`range_difference_t<T>`からコンセプトをたどっていくと`weakly_incrementable`コンセプトによって符号付整数型であることが制約されています。

`ranges::empty`は範囲が空かどうか、すなわちその長さが`0`かどうかを`bool`値で得るものです。

型`T`のオブジェクト`r`によって`ranges::empty(r)`のように呼び出されたとき、その効果は次のいずれかになります

1.  `T`が要素数不明の配列型である場合、*ill-formed*
2.  `bool(r.empty())`が呼び出し可能であ場合、`bool(r.empty())`
        - メンバ関数の`empty()`を呼び出す
3.  `ranges::size(r) == 0`が呼び出し可能であ場合、`ranges::size(r) == 0`
4. `bool(ranges::begin(r) == ranges::end(r))`が有効な式であり、`ranges::begin(r)`の戻り値型`I`が`forward_iterator`のモデルとなる場合、`bool(ranges::begin(r) == ranges::end(r))`
5. それ以外の場合、*ill-formed*

3番目や4番目のケースで分かるように、`ranges::empty`は範囲が空（サイズが`0`）の時に`true`を返します。そして、`ranges::empty(r)`の呼び出しが有効である時の戻り値型は`bool`となります。

なお、`begin/end`や`size`などのCPOと異なり、`ranges::empty`はメンバ関数の`empty`は探してくれますが、ADLによって非メンバ`empty`を探しに行きません。自作の型をアダプトする場合などに注意が必要です。


## 範囲の領域のポインタ取得

`ranges::data`は範囲の存在するメモリ上の領域へのポインタを取得するものです。

範囲の存在する領域が飛び飛びだとポインタを取得する意味がないため、引数となる`range`は`contiguous_range`でなければなりません。

```cpp
namespace std::ranges {
  inline namespace unspecified {
    inline constexpr unspecified data = unspecified;
    inline constexpr unspecified cdata = unspecified;
  }
}
```

型`T`のオブジェクト`r`によって`ranges::data(r)`のように呼び出されたとき、その効果は次のいずれかになります

1. `r`が右辺値であり、かつ`enable_borrowed_range<remove_cv_t<T>> == false`の場合、*ill-formed*
2. `T`が配列型であり、`remove_all_extents_t<T>`が不完全型を示す場合、*ill-formed*
3. `decay-copy(r.data())`が呼び出し可能であり、その戻り値型がオブジェクトポインタ型である場合、`decay-copy(r.data())`
      - メンバ関数の`data()`を呼び出す 
4. `ranges::begin(r)`が呼び出し可能であり、その戻り値型が`contiguous_iterator`のモデルであるなら、`to_address(ranges::begin(r))`
5. それ以外の場合、*ill-formed*

4番目のケースでは`to_address`を使用していることによって、`contiguous_iterator`となる非ポインタ型のイテレータラッパからでもアドレスを取得可能です。

`ranges::cdata`は`ranges::data`のポインタを`const`ポインタで取得するもので、その効果は次のようになります

1. `r`が左辺値ならば、`ranges::data(static_cast<const T&>(r))`
2. それ以外の場合、`ranges::data(static_cast<const T&&>(r))`

`ranges::data(r)/ranges::cdata(r)`の呼び出しが有効である時の戻り値型はオブジェクトポインタ型となります。

`ranges:cdata`のやっていることは`cbegin/cend`と全く同じであるため、`const`ポインタが常に得られるかどうかについて同じ問題があります。

```cpp
int main () {
  int arr[5]{};
  std::span s{arr};
  
  auto* ap = std::ranges::cdata(arr);
  // decltype(ap) == const int*;
  auto* sp = std::ranges::cdata(s);
  // decltype(sp) == int*;
}
```

`std::span`がまさにそうですが、自身の`const`性と要素の`const`性が同期しない`range`型では、この関数が有効でも`const`ポインタが得られるとは限りません。

\clearpage

# その他のRange CPO

分類としてはRangeアクセス関数ではありませんが、Rangeライブラリにおいて新しいカスタマイゼーションポイントとして追加されたCPOが他にもあります。

## `ranges::swap`

`ranges::swap`は値の交換を行うものです。これは従来の`std::swap`をリファインしたものです。ちなみにこれは厳密には`<concepts>`ヘッダに配置されています。

```cpp
namespace std::ranges {
  inline namespace unspecified {
    inline constexpr unspecified swap = unspecified;
  }
}
```

型`T, U`のオブジェクト`a, b`によって`ranges::swap(a, b)`のように呼び出されたとき、その効果は次のいずれかになります

1. `T, U`が共にクラス型か列挙型であり、`(void)swap(a, b)`が呼び出し可能であれば、`(void)swap(a, b)`
      - ADLによって`swap()`を呼び出す
      - その際、ADLより前に`std`名前空間にある`std::swap()`を発見しないように名前解決が行われる
2. `T, U`が共に同じ長さの配列型で`a, b`はともに左辺値であり、`ranges::swap(*a, *b)`が呼び出し可能ならば、`(void)ranges::swap_ranges(a, b)`
      - ただし、例外を投げるかどうかは`noexcept(ranges::swap(*a, *b))`による
3. `a, b`は同じ型`T`の左辺値であり、`T`が`move_constructible<T>`と`assignable_from<T&, T>`のモデルである場合、ムーブ代入とムーブ構築を用いた交換操作によって、`a, b`の値を交換する
      - ただし、例外を投げるかどうかは`is_nothrow_move_constructible_v<T> && is_nothrow_move_assignable_v<T>`による
4. それ以外の場合、*ill-formed*

標準ライブラリのコンテナ型などはメンバ`.swap()`を持っており、1のケースによって`std::swap`経由でメンバの`.swap()`が呼び出されます。

2のケースで用いられる`ranges::swap_ranges`は新しいアルゴリズムであり、範囲の各要素を一つづつ`swap`していくものです。そこでは各要素について`ranges::swap`を順番に適用していくことで、配列全体を交換します。

3のケースは最後の手段であり、ムーブ代入・構築を用いた次のような操作によって値を交換します

```cpp
template<typename T>
void swap(T& a, T& b) {
  T t(std::move(b));
  b = std::move(a);
  a = std::move(t);
}
```

この様な操作のためには、型`T`がムーブ構築とムーブ代入が可能である必要があり、`move_constructible`と`assignable_from<T&, T>`コンセプトによってそれを制約しています。

3のケースでは定数式で使用可能かどうかが別に指定されており、次の条件を満たす必要があります

- `T`はリテラル型
- `a = std::move(b), b = std::move(a)`が共に定数式で使用可能
- `T t1(std::move(a)); T t2(std::move(b));`の様な初期化が定数式で使用可能

`ranges::swap(a, b)`の呼び出しが有効であるとき、`a, b`の値を交換し戻り値型は`void`となります。

## `iter_move`

`ranges::iter_move`はイテレータの指す要素をムーブするためのものです。イテレータ`it`に対して`std::move(*it)`の様な事をやってくれます。

```cpp
namespace std::ranges {
  inline namespace unspecified {
    inline constexpr unspecified iter_move = unspecified;
  }
}
```

型`I`のオブジェクト`it`によって`ranges::iter_move(it)`のように呼び出されたとき、その効果は次のいずれかになります

1. `I`はクラス型か列挙型であり、`iter_move(it)`が呼び出し可能である場合、`iter_move(it)`
      - ADLによって`iter_move()`を呼び出す
      - その際、ADLより前に`std::iter_move()`を発見しないように名前解決が行われる
2. `*it`が有効であり、その結果が左辺値である場合、`std::move(*it)`
3. `*it`が有効であり、その結果が右辺値である場合、`*it`
4. それ以外の場合、*ill-formed*

この操作はC++20から追加されたもので、イテレータの中身の要素を`move`しつつ取り出したいときに使用します。イテレータ型に合わせた最適な方法によって要素のムーブを行うため、`std::move(*it)`よりもこちらを使用すべきです。

文字数的には長くなっているし使用するメリットは分かり辛いのですが、2,3番目の効果に現われているように、イテレータの間接参照が右辺値（*prvalue*）を返すときに余計な値カテゴリの変換を行わないようにすることで、コピー省略などの最適化を妨げないようにしています。

1のケースはイテレータラッパなどを作成する際に、ベースのイテレータに直接`iter_move`を適用するためのカスタマイゼーションポイントとして利用することができます。`iter_move`を使う場合は最終的には2,3番目のどちらかを利用することになる筈です。

## `iter_swap`

`ranges::iter_swap`は2つのイテレータの指す要素を交換するものです。イテレータ`i1, i2`に対して`ranges::swap(*i1, *i2)`の様な事をやってくれます。イテレータを介した`swap`操作と見ることもできます。

```cpp
namespace std::ranges {
  inline namespace unspecified {
    inline constexpr unspecified iter_swap = unspecified;
  }
}
```

型`I1, I2`のオブジェクト`i1, i2`によって`ranges::iter_swap(i1, i2)`のように呼び出されたとき、その効果は次のいずれかになります

1. `I1`と`I2`のどちらかがクラス型か列挙型であり、`iter_swap(i1, i2)`が呼び出し可能である場合、`(void)iter_move(it)`
      - ADLによって`iter_swap()`を呼び出す
      - その際、ADLより前に`std::iter_swap()`を発見しないように名前解決が行われる
2. `I1, I2`が共に`indirectly_readable`コンセプトのモデルであり、それぞれの（要素の）参照型が`swappable_with`コンセプトのモデルとなる場合、`ranges::swap(*i1, *i2)`
3. `I1, I2`が`indirectly_movable_storable<T1, T2>`と`indirectly_movable_storable<T2, T1>`のモデルとなる場合、`(void)(*i1 = iter-exchange-move(i2, i1))`
4. それ以外の場合、*ill-formed*

1のケースは`ranges::iter_move`と同様にイテレータラッパなどで内部イテレータに直接作用させるためのカスタマイゼーションポイントです。`ranges::iter_swap`の主たる効果は2,3番目のケースです。

2番目は単純に、`operator*`によって間接参照しつつ`ranges::swap`で`i1`と`i2`参照先の要素を交換します。その際、`i1, i2`が最低限間接参照可能であることと、その参照先のものが交換可能であることを`swappable_with`コンセプトによってチェックします。

3番目は知らないものばかり登場していて複雑です。`indirectly_movable_storable<In, Out>`コンセプトは、`In`の値型（`iter_value_t<In>`）の中間オブジェクトを介した`In`から`OUt`への要素のムーブ操作を定義するコンセプトです。型`In, Out`のオブジェクトをそれぞれ`in, out`とすると次のような操作が可能であることを表します。

```cpp
iter_value_t<In> tmp = std::move(*in);
*out = std::move(tmp);
```

3番目のケースの条件全体は、`T1 -> tmp -> T2`とその逆方向の`T2 -> tmp -> T1`のような要素の移動が可能であることをコンセプトによってチェックします。これは`ranges::swap`で似たような話を聞いたかもしれません。

そして、`iter-exchange-move`とは次のように定義された説明専用の操作です

```cpp
template<class X, class Y>
constexpr iter_value_t<X> iter-exchange-move(X&& x, Y&& y)
  noexcept(noexcept(iter_value_t<X>(iter_move(x))) &&
           noexcept(*x = iter_move(y)))
{
  iter_value_t<X> old_value(iter_move(x));
  *x = iter_move(y);
  return old_value;
}
```

`std::exchange`の挙動に近いことをしており、第二引数の要素を第一引数へ、第一引数の要素を戻り値としてそれぞれムーブします。`*i1 = iter-exchange-move(i2, i1)`の式全体では、`i1 -> i2 -> i1`の様に玉突き的に要素の値が流れます。つまりは、ムーブによって力業で2つのイテレータの要素を入れ替えています。

このように、3番目のケースは`ranges::swap`によって要素の交換が出来ない場合に最後の手段として働くものです。

ところで、`ranges::iter_move`も`ranges::iter_swap`もよく見るとイテレータ型だけでなく、間接参照可能な（`indirectly_readable`な）型に対して使用することができます。

\clearpage

# Range情報取得エイリアス

イテレータで`iterator_traits`を介してイテレータの性質などの情報を取得出来ていたように、`range`でも同じことをしたいことが多々あります。`iterator_traits`でできていたことをはC++20で使いやすく改修されましたが、イテレータ用のものを利用しようとすると、`range`からイテレータを取り出してその型を取り出して...の様な事をすることになります。

それを避けるために、`range`から直接そのイテレータの各種情報を引き出すためのものがエイリアステンプレートとして用意されています。

## `iterator_t/sentinel_t`

`ranges::iterator_t`と`ranges::sentinel_t`は`range`型からそのイテレータ型と番兵型を得るものです。

```cpp
template<class T>
  using iterator_t = decltype(ranges::begin(declval<T&>()));

template<range R>
  using sentinel_t = decltype(ranges::end(declval<R&>()));
```

実装は見たままで、`ranges::begin/ranges::end`を使用して入力の`range`型からイテレータ/番兵を取り出し、`decltype`でその型を取得しています。とはいえこれが無いと同等のものを一々定義するか、これと同じことを書くかすることになるため、これは便利なものです。C++20からは`typename`を省略できるコンテキストが増えているため、あまり難しいことを考えなくてもこれを使用できます。

```cpp
template<range R>
  // イテレータ型に対する制約を書くときに使う
  requires same_as<iterator_t<R>, int*>
void f(R&& r) {
  // イテレータ型が欲しい時に使う
  iterator_t<R> it = ranges::begin(r);
  sentinel_t<R> se = ranges::end(r);
}
```

目ざとい人は`iterator_t`だけ`range`コンセプトで制約されていない事に気付くでしょう。これはバグではなくあえての事で、`range`ではないが`ranges::begin`が適用可能な型が存在しており、そのような型からでもイテレータ型を取得できるように考慮されての事です。特に要素数不明の配列型を想定しているようです。

```cpp
// 要素数不明の配列
extern int arr[];

int main() {
  iterator_t<decltype(arr)> p1;  // ok
  sentinel_t<decltype(arr)> p2;  // ng
}
```

この様な配慮は`ranges::begin`および`ranges::data`においてこの2つだけに行われており、それを受けて`iterator_t`にも表れているものです。その他の`ranges::end`や`ranges::size`ではこの配慮の必要はないため、要素数不明の配列型を明示的に禁止しています。そのため`sentinel_t`では入力の型を`range`コンセプトによって制約することができています。

## `range_difference_t/iter_difference_t`

`ranges::range_difference_t`は`range`型からそのイテレータ間距離を表すための型を得るものです。そのような型はイテレータの`difference_type`と呼ばれます。

```cpp
template<class I>
  using iter_difference_t = /*...*/;

template<range R>
  using range_difference_t = iter_difference_t<iterator_t<R>>;
```

`ranges::range_difference_t`は`std::iter_difference_t`を用いてイテレータ型からその距離を表す型を取得します。`remove_cvref`したイテレータ型を`I`とすると、`std::iter_difference_t`の示す型は次のどちらかです

- `std::iterator_traits<I>`がプライマリテンプレートの特殊化となる場合、`std::incrementable_traits<I>::difference_type`
- それ以外の場合、`std::iterator_traits<I>::difference_type`

`std::iterator_traits<I>`がプライマリテンプレートの特殊化となる場合というのは、`I`が`iterator_traits`を明示的に特殊化していない時を指します。そして、`std::incrementable_traits`はC++20から追加された型特性クラスで、コンセプトを活用してイテレータの`difference_type`を何とか求めてくれるものです。

```cpp
namespace std {
  // プライマリテンプレート
  template<class>
  struct incrementable_traits {};

  // ポインタ型についての特殊化
  template<class T>
    requires is_object_v<T>
  struct incrementable_traits<T*> {
    using difference_type = ptrdiff_t;
  };

  // constを外すための特殊化
  template<class I>
  struct incrementable_traits<const I>
    : incrementable_traits<I> { };

  // difference_typeを定義している型についての特殊化
  template<class T>
    requires requires { typename T::difference_type; }
  struct incrementable_traits<T> {
    using difference_type = typename T::difference_type;
  };

  // difference_typeを定義していないが
  // -によって差分を取ることができる型についての特殊化
  template<class T>
    requires (!requires { typename T::difference_type; } &&
              requires(const T& a, const T& b) { { a - b } -> integral; })
  struct incrementable_traits<T> {
    using difference_type = make_signed_t<decltype(declval<T>() - declval<T>())>;
  };
}
```

要するに、イテレータ型を`I`として次の3つの経路で`difference_type`を探します

- `I::difference_type`
- `I`の`oeprator-`（2項演算）の戻り値型
- `std::incrementable_traits<I>`の明示的/部分特殊化

メンバ型`difference_type`がなかったとしても取得しようとしてくれます。`std::incrementable_traits`は自作のイテレータ型を`std::iter_difference_t`で使用可能とするときにアダプトするために特殊化して使用します。特に、イテレータラッパにおいてベースのイテレータ型の`difference_type`を直接取得できるようにする際に特殊化して利用されるようです。単に`difference_type`が欲しい場合は`range_difference_t/iter_difference_t`を使用した方がよりジェネリックかつ確実に`difference_type`を取得することができます。

直接的には制約されていないのですが、`range_difference_t/iter_difference_t`の示す型は常に符号付整数型になります。コンセプトを辿っていくと、`std::weakly_incrementable`コンセプトで制約されています。自分でイテレータを定義する際も`difference_type`は符号付整数型にしなければなりません。多くの場合`ptrdiff_t`が利用されます。

```cpp
// n番目の要素を取り出す簡易な例
template<ranges::random_access_range R>
auto pick(R& r, ranges::range_difference_t n) {
  auto it = ranges::begin(r);
  it += n;

  return *it;
}
```

この様に、イテレータをいくつ進める、あるいは範囲の何番目の、などの位置に関する情報を数値で受け取りたいときにその引数型として利用することができます。なお、この例では範囲の終端チェックを省いていることに注意が必要です。

## `range_size_t`

`ranges::range_size_t`は`range`型からその長さの型を取得するものです。

```cpp
template<sized_range R>
  using range_size_t = decltype(ranges::size(declval<R&>()));
```

つまりは、`ranges::size`の戻り値型を取得するものです。

当然入力となる`range`型はサイズを求めることができなくてはならないので、`sized_range`コンセプトで制約されています。

```cpp
template<ranges::sized_range R>
void f(R&& r, int n) {
  // この型が符号なし整数型とすると
  auto len = ranges::size(r);
  
  // 安全のため明示的に型を合わせる
  if (ranges::range_size_t<R>(n) < len) {
    // ...
  }
}
```

あまり有用そうな使い方は思いつかないですが、このように`ranges::size`の戻り値が欲しい時に使用できます。ちなみに、C++20よりこの例のような整数型の比較を安全に行ってくれる便利な関数が`<utility>`に追加されています。

```cpp
// n < lenを安全かつ確実に比較する
if (std::cmp_less(n, len)) {
  // ...
}
```

整数型の符号有無で警告が発せられたり、比較が意図通りにならなかったりと困ることは多かったので、この関数はとても便利です。

## `range_value_t/iter_value_t`

`ranges::range_value_t`は`range`型の要素の値型を取得するものです。そのような型はイテレータの`value_type`と呼ばれます。

```cpp
template<class I>
  using iter_value_t = /*...*/;

template<range R>
  using range_value_t = iter_value_t<iterator_t<R>>;
```

`ranges::range_value_t`は`std::iter_value_t`を用いてイテレータ型からその要素の値型を取得します。`remove_cvref`したイテレータ型を`I`とすると、`std::iter_value_t`の示す型は次のどちらかです

- `std::iterator_traits<I>`がプライマリテンプレートの特殊化となる場合、`std::indirectly_readable_traits<I>::value_type`
- それ以外の場合、`std::iterator_traits<I>::value_type`

`std::iter_difference_t`と似たような感じです。`std::indirectly_readable_traits`はC++20から追加された型特性クラスで、コンセプトを活用してイテレータの`value_type`を何とか求めてくれるものです。

```cpp
namespace std {

  // 素の型を取得しvalue_typeという名前に変換する、説明専用型特性
  template<class>
  struct cond-value-type { };

  template<class T>
    requires is_object_v<T>
  struct cond-value-type<T> {
    using value_type = remove_cv_t<T>;
  };


  // プライマリテンプレート
  template<class>
  struct indirectly_readable_traits { };

  // ポインタ型についての特殊化
  template<class T>
  struct indirectly_readable_traits<T*>
    : cond-value-type<T> { };

  // 配列型についての特殊化
  template<class I>
    requires is_array_v<I>
  struct indirectly_readable_traits<I> {
    using value_type = remove_cv_t<remove_extent_t<I>>;
  };

  // constを外すための特殊化
  template<class I>
  struct indirectly_readable_traits<const I>
    : indirectly_readable_traits<I> { };

  // 説明専用コンセプト
  template<class T>
  concept has-member-value-type = requires { typename T::value_type; };
  template<class T>
  concept has-member-element-type = requires { typename T::element_type; };

  // value_typeを定義している型についての特殊化
  template<has-member-value-type T>
  struct indirectly_readable_traits<T>
    : cond-value-type<typename T::value_type> { };

  // element_typeを定義している型についての特殊化
  template<has-member-element-type T>
  struct indirectly_readable_traits<T>
    : cond-value-type<typename T::element_type> { };

  // value_typeとelement_typeを両方定義している型についての特殊化
  template<has-member-value-type T>
    requires has-member-element-type<T> &&
             same_as<remove_cv_t<typename T::element_type>, 
                     remove_cv_t<typename T::value_type>>; }
  struct indirectly_readable_traits<T>
    : cond-value-type<typename T::value_type> { };
}
```

定義はなかなか複雑ですが、イテレータ型を`I`として主次の3つの経路で`value_type`を取得します

- `I::value_type`
- `I::element_type`
- `std::indirectly_readable_traits<I>`の明示的/部分特殊化

メンバ型`value_type`がなかったとしても取得しようとしてくれます。`std::indirectly_readable_traits`は自作のイテレータ型を`std::iter_value_t`で使用可能とするときにアダプトするために特殊化して使用します。単に`value_type`が欲しい場合は`range_value_t/iter_value_t`を使用した方がよりジェネリックかつ確実に`value_type`を取得することができます。

```cpp
template<ranges::range R>
void f(R& r, ranges::range_value_t<R> v) {
  bool b = std::ranges::find_if(r, v);
  // ...
}
```

このように、関数の引数などで`range`の要素の参照型ではなく値型が欲しい時に使用できます。例にある`ranges::find_if`はRangeライブラリと共に追加されたRangeアルゴリズムの一つで`std::find_if`を`range`に対応させたものです。

## `range_reference_t/iter_reference_t`

`ranges::range_reference_t`は`range`の要素を指す参照型を取得します。そのような型はイテレータの`reference_type`（参照型）と呼ばれます。

```cpp
template<dereferenceable I>
  using iter_reference_t = decltype(*declval<I&>());

template<range R>
  using range_reference_t = iter_reference_t<iterator_t<R>>;
```

`std::iter_reference_t`はとても潔い定義になっています。この通り、`range`のイテレータの間接参照の結果の型がイテレータの参照型です。参照型と呼ばれてはいますが、イテレータの間接参照の結果は左辺値参照である必要はなく、右辺値（*prvalue*）であっても構いません。参照型というのは歴史的な呼び名です。

この定義からも分かるように、`range`あるいはイテレータと呼ばれるものであれば自然にこれらを利用できるはずです。

`std::iter_reference_t`を制約している`dereferenceable`コンセプトは、間接参照の結果型が`void`ではないことを表す説明専用のコンセプトです（`std::dereferenceable`コンセプトは存在しません）。とても簡易な制約なため、`std::iter_reference_t`はスマートポインタ型や`std::optional`などでも利用できます。

```cpp
template<ranges::range R>
void f(R& r, ranges::range_reference_t<R> v) {
  bool b = std::ranges::find_if(r, v);
  // ...

  // 可能なら参照する
  ranges::range_reference_t<R> e = *ranges::begin(r);
}
```

これは比較的よく使いそうな気がします。`range`のイテレータの間接参照の型そのものが使いたい時や、イテレータから要素を取り出すときに受ける型が欲しい時などに使用できます。

## `range_rvalue_reference_t`/\newline `iter_rvalue_reference_t`

`ranges::range_reference_t`は`range`の要素を指す右辺値参照型を取得します。

```cpp
template<dereferenceable I>
    requires requires(I& i) {
      { ranges::iter_move(i) } -> can-reference;
    }
  using iter_rvalue_reference_t = decltype(ranges::iter_move(declval<I&>()));

template<range R>
  using range_rvalue_reference_t = iter_rvalue_reference_t<iterator_t<R>>;
```

従来のイテレータにはこれに対応する型はありませんでした。

`std::iter_rvalue_reference_t`は少し複雑な定義をされていますが、`ranges::iter_move`CPOの戻り値型を取得しており、行われている制約は`ranges::iter_move`CPOが利用可能であるかをチェックしています。`ranges::iter_move`はイテレータ`i`に対して`*i`が左辺値なら`std::move(*i)`を、右辺値なら`*i`をそのまま得るものなので、その型を得るということは要素を参照する右辺値参照型もしくは*prvalue*である要素の素の型を取得することになります。

ここでの制約式は初めて見る構文もあり少し複雑です。`dereferenceable`コンセプトに加えて、`requires`節の中で`requires`式を使用して制約されています。`requires`節では`bool`を返す任意の式を制約式として`&& ||`でつないでいくことしかできず、そのままだと表現力が足りません。`requires`式も1つの式としては`bool`を返す式であり`requires`節の中で使用することができて、`requires`式の中ではより柔軟な制約を行うことができます。そのため、`requires`節と`requires`式を組み合わせて制約を行うことは良く行われます。`requires requires`と並ぶのは違和感がありますが、慣れましょう・・・

`requires`式の中では戻り値型の制約構文で`ranges::iter_move`が使用可能であることと、その戻り値型を`can-reference`コンセプトによってチェックしています。`can-reference`コンセプトは型が`void`ではないことをチェックする説明専用のものです。つまり全体では`ranges::iter_move`が使用可能であり何かしら結果を返すことを制約しています。

これは例えば次のように、`range`の要素を上書きしてムーブ代入したいとき、そのような処理を行う関数の引数型として使用できます。

```cpp
template<ranges::range R>
void replace(R& r, ranges::range_rvalue_reference_t<R> rv) {
  auto it = ranges::begin(r);
  *it = std::move(rv);
  // ...
}
```

\clearpage

# 部分範囲（*subrange*）型

あらゆる範囲を`range`として扱うと言っても、イテレータペアで扱っているところは残り続けますしそっちの方が便利である場合もあり、それをもう一度`range`の形に戻すには何らかのラッパ型が必要となります。また、範囲の一部分を取り出して1つの部分範囲として扱いたいこともよくあります。イテレータペアを介することでどちらの問題も同様のアプローチによる解決が可能となり、Rangeライブラリにはそれを行ってくれる便利な型、`subrange`が用意されています。

## `view_interface`

`view_interface`は`view`を定義するときに必要となる物の共通部分を集めたクラスです。`view`型はこのクラスを継承して利用することが推奨され、`subrange`をはじめとするRangeライブラリの`view`型は基本的にそうなっています。

```cpp
namespace std::ranges {
  template<class D>
    requires is_class_v<D> && same_as<D, remove_cv_t<D>>
  class view_interface : public view_base {
    // ...
  };
}
```

コンセプトが示すのは、`D`がクラス型でありCV修飾されていないという事です。

`view_interface`利用時は、CRTPによって自身の型を渡しつつ継承することで利用します。

```cpp
// 自作のviewを定義するとき、CRTPして継承する
template<range R>
class my_view : view_interface<my_view<R>> {
  // ...
}
```

`view_interface`が`ranges::view_base`を基底に持つことによって、`view_interface`を継承した型は自動的に`ranges::enable_view`が`true`になるようになり、ほかの条件を満たしていれば`view`コンセプトのモデルとなることができます。

これだけではなく、`view_interface<D>`では`range`型`D`（とそのイテレータ）が満たす性質によって、利用可能となるコンテナインターフェースを自動的に有効化してくれます。

```cpp
template<class D>
  requires is_class_v<D> && same_as<D, remove_cv_t<D>>
class view_interface : public view_base {
public:
  constexpr bool empty() requires forward_range<D>;

  constexpr explicit operator bool()
    requires requires { ranges::empty(derived()); };

  constexpr auto data() requires contiguous_iterator<iterator_t<D>>;

  constexpr auto size()
    requires forward_range<D> &&
             sized_sentinel_for<sentinel_t<D>, iterator_t<D>>;

  constexpr decltype(auto) front() requires forward_range<D>;

  constexpr decltype(auto) back() requires bidirectional_range<D> && common_range<D>;

  template<random_access_range R = D>
  constexpr decltype(auto) operator[](range_difference_t<R> n);
};
```

これらの関数は、継承先の`range`型`D`のイテレータだけを用いて実装される様になっているため（`operator bool`を除いて）、`D`を`range`として正しく実装しておけば追加の作業を必要とせずに有効化されます。なお、省略していますが全て`const`オーバーロードも用意されています。

ここでは後置`requires`節によって、クラステンプレートの非テンプレートメンバ関数の制約が行われています。そこでは主に、`D`がどのタイプの`range`なのかによってそれぞれの関数を有効化するか否かが決定されています。

- `empty()`
    - `D`が`forward_range`であるとき
- `operator bool`
    - `D`のオブジェクト`d`に対して、`ranges::empty(d)`が利用可能であるとき
- `data()`
    - `D`イテレータが`contiguous_iterator`であるとき
- `size()`
    - `D`が`forward_range`であり
    - `D`のイテレータと番兵の引き算によってサイズを求められる時
- `front()`
    - `D`が`forward_range`であるとき
- `back()`
    - `D`が`bidirectional_range`であり`common_range`であるとき
- `operator[]`
    - `D`が`random_access_range`であるとき

```cpp
// 自作のview
template<ranges::range R>
class my_view : ranges::view_interface<my_view<R>> {
  // ...
}

int main() {
  int arr[] = {1, 2, 3, 4};
  my_view mv{arr};

  // range型Rがranges::emptyを利用可能なら
  bool(mv);

  // my_viewがforward_rangeなら
  mv.empty();
  mv.front();

  // さらに、 my_viewのイテレータがsized_sentinel_forのモデルであるなら
  mv.size();

  // さらに、bidirectional_rangeかつcommon_rangeなら
  mv.back();

  // random_access_rangeなら
  mv[1];

  // さらに、contiguous_rangeなら
  mv.data();
}
```

このように、`view_interface`は`view`の定義の一部を自動化してくれるもので、自分で`view`を作る場合に利用しないという選択肢はないかと思われます。そして、これらの操作は`D`そのものか`D`のイテレータを用いてほとんど自動的に導出されるようになっているため、`D`および`D`のイテレータを各種`range`/イテレータコンセプトに適合するようにしてあれば、これらの操作を有効化するために特段何かをする必要はないはずです。

## `subrange`

`ranges::subrange`は任意のイテレータペアをラップして、その範囲を示すことのできる`range`型です。`std::span`が連続したメモリ領域のポインタと長さをラップして参照するものであるように、`subrange`は任意の範囲についてのイテレータ2つあるいはイテレータと番兵から、その範囲を参照する`range`を作成します。

その性質から`subrange`は明らかに`view`であり、常に`view`コンセプトのモデルとなります。また、`borrowed_range`でもあるため`enable_borrowed_range`が`true`となるように特殊化されており、常に`borrowed_range`コンセプトのモデルでもあります。

```cpp
// イテレータペアを受け取る旧来のインターフェース
template<typename I>
void old_range_algo(I i1, I i2) {
  // イテレータペア -> range へ変換
  ranges::subrange sr{i1, i2};

  // 範囲forでイテレートできる
  for (const auto& v : sr) {
    // ...
  }

  // Rangeアルゴリズムで使用可能にする
  auto it = ranges::find_if(sr, [](const auto& v) { /*...*/ });

  // Range Adopterで使用可能にする
  for (const auto& v : sr | views::filter( /*...*/ )
                          | views::transform( /*...*/ ))
  {
    // ...
  }
}
```

`subrange`は主に、旧来のイテレータペアによる範囲の取り回しとC++20 Rangeライブラリとの橋渡しをしてくれるものです。上記例の様に、イテレータペアを`range`（`view`）へ変換することで、Rangeライブラリの多くの機能を利用できるようになります。

## `subrange_kind`

`subrange`は使うだけならとても簡単で悩みどころもあまり無いのですが、実体はとても複雑な型になっています。

```cpp
namespace std::ranges {
  // subrangeの宣言
  template<input_or_output_iterator I, sentinel_for<I> S = I, subrange_kind K =
      sized_sentinel_for<S, I> ? subrange_kind::sized : subrange_kind::unsized>
    requires (K == subrange_kind::sized || !sized_sentinel_for<S, I>)
  class subrange : public view_interface<subrange<I, S, K>> {
    // ...
  };
}
```

何書いてあるかわからないですね。コンセプトに慣れていても目を凝らさないと何が何だかわからないと思われます・・・

テンプレートパラメータは、イテレータ型`I`とその番兵型`S`、`subrange_kind`の値`K`の3つが宣言されています。

ここで、`sentinel_for<I> S`の様な制約の形式では`sentinel_for`コンセプトの第一引数の指定が省略され、自動的に補われています。`sentinel_for`は本来2引数のコンセプトであり、ここでは正しくは`sentinel_for<S, I>`というコンセプトがチェックされます。戻り値型の制約などで第一引数が補われていたことがここでも行われているわけです。もしこれが行われない場合、型パラメータの宣言と同時に制約を行うことができず、あとから`requires`式で制約しなければならなくなってしまうため、不便です。

`subrange_kind`は次のように定義されているスコープ付列挙型で、テンプレートパラメータ`K`は非型テンプレートパラメータです。

```cpp
namespace std::ranges {
  enum class subrange_kind : bool { unsized, sized };
}
```

そして、`K`の初期化は次のように行われています。

```cpp
subrange_kind K = sized_sentinel_for<S, I> ? subrange_kind::sized : subrange_kind::unsized;
```

`sized_sentinel_for<S, I>`は`sentinel_for<S, I>`かつ`I, S`のオブジェクト`i, s`によって、`i - s, s - i`によってイテレータ間距離が求められることを定義するコンセプトです。

`K`は入力の型`I, S`が`sized_sentinel_for<S, I>`を満たすとき`subrange_kind::sized`、満たさないとき`subrange_kind::unsized`で初期化されます。`sized_sentinel_for`の意味と列挙値の名前からもわかるかもしれませんが、この`K`は`subrange`が`sized_range`となるかどうかを制御しています。`K == subrange_kind::sized`の時、`subrange<I, S, K>`は`sized_range`コンセプトのモデルになります。

```cpp
requires (K == subrange_kind::sized || !sized_sentinel_for<S, I>)
```

残った`requires`節による制約は、`K`が`subrange_kind::unsized`ならば`I, S`が  
`sized_sentinel_for<S, I>`を満たさない事をチェックしています。というのも、`subrange`に明示的に型と値を指定して特殊化してしまえば`K`の値を`I, S`によらずに指定することができてしまうためです。ただし、`K == subrange_kind::sized`であれば`I, S`はどうなっていても良いことがここから読み取ることができ、これはあえての事です。

## コンストラクタと推論補助

```cpp
// subrangeの簡易化宣言
template<input_or_output_iterator I, sentinel_for<I> S = I, subrange_kind K>
class subrange {
  // 説明専用静的変数
  // sized_sentinel_for<S, I>が満たされないのにKがsizedである時にtrue
  static constexpr bool StoreSize =
      K == subrange_kind::sized && !sized_sentinel_for<S, I>;
      
public:

  // デフォルトコンストラクタ (1)
  subrange() requires default_initializable<I> = default;

  // イテレータペアからのコンストラクタ (2)
  constexpr subrange(convertible-to-non-slicing<I> auto i, S s) requires (!StoreSize);

  // イテレータペアと長さを受け取るコンストラクタ (3)
  constexpr subrange(convertible-to-non-slicing<I> auto i, S s,
                     make-unsigned-like-t<iter_difference_t<I>> n)
    requires (K == subrange_kind::sized);
  
  // rangeからのコンストラクタ (4)
  template<not-same-as<subrange> R>
    requires borrowed_range<R> &&
             convertible-to-non-slicing<iterator_t<R>, I> &&
             convertible_to<sentinel_t<R>, S>
  constexpr subrange(R&& r) requires (!StoreSize || sized_range<R>);

  // rangeと長さを受け取るコンストラクタ (5)
  template<borrowed_range R>
    requires convertible-to-non-slicing<iterator_t<R>, I> &&
             convertible_to<sentinel_t<R>, S>
  constexpr subrange(R&& r, make-unsigned-like-t<iter_difference_t<I>> n)
    requires (K == subrange_kind::sized)
      : subrange{ranges::begin(r), ranges::end(r), n}
  {}
};
```

また難解ですが、`subrange`には全部で5つのコンストラクタがあります。デフォルトコンストラクタを除くと、イテレータペアと`range`オブジェクトを受け取るコンストラクタの2種類があり、別の見方をすると長さ`n`を受け取るか受け取らないかで2種類のコンストラクタがあります。

(1)はデフォルトコンストラクタです。イテレータ型`I`がデフォルト構築可能である場合に使用できます（`S`へのそれは`sentinel_for`コンセプトに含まれています）。

(2)のコンストラクタはおそらく最も使用されるであろう、イテレータペアを受けとるコンストラクタです。後置`requires`節に現れている`StoreSize`は`K == subrange_kind::sized`なのに`I, S`が`sized_sentinel_for<S, I>`を満たさない事を検出するものです。そのような指定は可能であり意図的に許可されていますが、その場合にこのコンストラクタから初期化する事はできません。  
(4)のコンストラクタはその`range`版です。`range`型`R`は`subrange`自身と同じ型ではなく`borrowed_range`であり、`K == subrange_kind::sized`ならば`sized_sentinel_for<S, I>`が満たされているか、`R`が`sized_range`である必要があります。

複雑ですが、(2)(4)のコンストラクタは共に、入力の`range`あるいはイテレータペアから単純な方法によってその長さを求める事ができる場合に使用されるコンストラクタです。そうでは無い場合は`K`の値に関係なく使用可能ではありません。

(3)のコンストラクタは、イテレータペア`[i, s)`に加えてその範囲サイズ`n`を受け取るコンストラクタです。`make-unsigned-like-t`は説明専用の操作で、要するに`n`が負では無い事を表しています。そして、事前条件として`[i, s)`の長さは`n`に等しい事が（文書で）要求されます。以上でも以下でもなく等しくなければならず、従わない場合は未定義動作です。  
(5)のコンストラクタはその`range`版です。`range`型`R`は`borrowed_range`でなくてはなりません。実装は(3)のコンストラクタに入力`range`オブジェクト`r`から取得したイテレータペアを転送するだけです。

(3)(5)のコンストラクタはその後置`requires`節が示す通りに`K == subrange_kind::sized`の時に使用されるコンストラクタです。(2)(4)との違いは、`sized_sentinel_for<S, I>`を満たしていなくても使用可能であるところにあります。これらのコンストラクタから初期化された場合、`subrange`は`n`の値を保持し`.size()`メンバ関数はその`n`を返すようになります。すなわち、これらのコンストラクタでは`sized_range`では無い範囲を`sized_range`となるように変換する事ができます。

```cpp
template<sized_range R>
void f(R&& r) {
  auto l = ranges::size(r); // l == 5
}

int main() {
  std::forward_list fl = {2, 5, 1, 0, 9};

  // どちらも構築はok
  auto sr1 = subrange{fl.begin(), fl.end()};    // (2)を使用
  auto sr2 = subrange{fl.begin(), fl.end(), 5}; // (3)を使用

  /* range版を用いてもいい
  auto sr1 = subrange{fl};    // (4)を使用
  auto sr2 = subrange{fl, 5}; // (5)を使用
  */

  f(sr1); // ng
  f(sr2); // ok
}
```

ただし、(3)(4)から構築された`subrange`のイテレータの進行において`n`を使ったチェックがなされるわけではなく、`n`の値は`subrange`自身が`sized_range`となるためにしか使用されません。更に言うと、`subrange`のイテレータ/番兵は構築に用いたイテレータ/番兵をそのまま返し、専用のイテレータ型を持っていません。また、(3)(4)のコンストラクタから構築されない場合は`n`を保存しておくためのサイズオーバーヘッドはなく、イテレータペアだけをメンバとして持ちます。

各コンストラクタに指定されている`convertible-to-non-slicing`とかのコンセプトは、引数のイテレータ/番兵型がテンプレートパラメータで指定されたイテレータ/番兵型に変換可能である事を指定し、なおかつ要素型を変換する形で変換されてしまう事を防止するものです（主に`T* -> U*`のようなポインタ型の変換を防止しています）。

(4)(5)の`range`オブジェクトから構築するコンストラクタは`borrowed_range`によって制約されていることで、`view`ではない右辺値の`range`からの構築を防止しています。

このような複雑な型とコンストラクタをしているため、`subrange`はクラステンプレートの実引数推論無くして活用する事はできません。当然そのままだと適切に型を推論できそうも無いので、`subrange`にはいくつか推論補助が用意されています。

```cpp
// (2)に対応
template<input_or_output_iterator I, sentinel_for<I> S>
subrange(I, S) -> subrange<I, S>;

// (3)に対応
template<input_or_output_iterator I, sentinel_for<I> S>
subrange(I, S, make-unsigned-like-t<iter_difference_t<I>>) ->
  subrange<I, S, subrange_kind::sized>;

// (4)に対応
template<borrowed_range R>
subrange(R&&) ->
  subrange<iterator_t<R>, sentinel_t<R>,
           (sized_range<R> || sized_sentinel_for<sentinel_t<R>, iterator_t<R>>)
             ? subrange_kind::sized : subrange_kind::unsized>;

// (5)に対応
template<borrowed_range R>
subrange(R&&, make-unsigned-like-t<range_difference_t<R>>) ->
  subrange<iterator_t<R>, sentinel_t<R>, subrange_kind::sized>;
```

基本的にそれぞれのコンストラクタに対応したものが用意されており、コンストラクタに渡された引数型から`subrange`のテンプレートパラメータを適切に埋めようとしている事がわかります。これらの推論補助によってクラステンプレートのパラメータが確定する前にどのコンストラクタが呼ばれているのかを判定する事ができるため、(3)(4)のコンストラクタが呼ばれた時だけサイズ引数`n`を保存するように`subrange`のレイアウトを変更する、みたいな事ができているのです。

推論補助にコンセプトによる制約が行われている事は驚きかもしれません。推論補助はテンプレートの一種であり、扱いとしては後置戻り値型の関数テンプレートに近いものです。したがって、関数テンプレートでできるような制約の指定はほぼ同様に行う事ができます（当然`requires`節を使用することもできます）。それらの制約は推論補助が選択される時に考慮され、制約なしだと推論補助がバッティングしてしまうような場合にも引数型によって適切な推論補助を選択させる事ができます。制約を満たす型が渡されず推論補助が選択されなかった場合はテンプレートパラメータが確定しないためエラーになるでしょう。

このコンセプトで制約された推論補助によって、`subrange`を使用する多くの場合はそのテンプレートパラメータを明示的に指定する必要はありません。というか、指定しようとしない方がいいでしょう・・・

\clearpage

# Rangeアルゴリズム

`<algorithm>`ヘッダに従来からある各種アルゴリズム関数は古くから存在しており、コンセプトやRangeライブラリに合わせた設計にはなっていません。とはいえ削除したりインターフェースを変更したりすると後方互換性を破壊してしまうので今更変更を加えることはできません。

そこで、従来のアルゴリズム関数をリファインしたRangeアルゴリズム関数が`std::ranges`名前空間の下に追加されます。Rangeアルゴリズム関数はRangeライブラリをベースとして設計されており、効果そのものは大きく変わりませんがより利用しやすくなっています。

本書では、`<algorithm>`ヘッダに用意されている各種関数群のことを単に「アルゴリズム」と呼び、`std::ranges`名前空間にある各種アルゴリズムに対応した新規関数群を「Rangeアルゴリズム」と呼びます。

## dangling iterator handling

Rangeアルゴリズムに入る前に、そこで利用されている面白い仕組みを先に見ておきます。

関数の引数などとして`range`を取りまわす時、`range`オブジェクトをコピーして取りまわすなんてことをするはずはなく、`view`や`borrowed_range`の様な範囲を参照する形でやり取りすることになります。その際に問題となるのは、参照先の範囲の寿命が先に尽きて、その範囲への参照あるいはイテレータが無効になってしまう事です。これは未定義動作に繋がり、そのような参照やイテレータの事をダングリング参照/イテレータと呼びます。

Rangeライブラリでは安全のためにダングリングとなる場合を可能な範囲でコンパイル時に検出しようとします。たとえば`viewable_range`コンセプトは入力となる`range`がダングリングとならない事を保証しようとするコンセプトでした。基本的には`viewable_range`同様に、入力となる`range`が右辺値であったり自身の処理の結果ダングリングなものを返しうるときなどにコンパイルエラーとなるようになっています。

Rangeアルゴリズムでも、入力`range`から取得したイテレータ/`subrange`を返す処理において、戻り値の利用が安全でない場合に代わりの型を返す仕組みが用意されています。

### `dangling`

`ranges::dangling`は出力となるイテレータがダングリングとなる場合に代わりに返される型です。

```cpp
namespace std::ranges {
  struct dangling {
    constexpr dangling() noexcept = default;

    template<class... Args>
    constexpr dangling(Args&&...) noexcept { }
  };
}
```

これは、C++20でリファインされるRangeアルゴリズム関数のうち、`range`を受け取りそのイテレータを返すタイプの関数において利用されます。

```cpp
// prvalueなrangeを返す関数
auto f() -> std::vector<int>;

int main() {
  // f()の戻り値はこの行で寿命が尽きる
  auto result1 = ranges::find(f(), 42);
  // decltype(result1) == ranges::dangling
}
```

`ranges::find`は`std::find`と同じことをイテレータ範囲ではなく`range`オブジェクトの指す範囲に対して行います。そのため、入力となる`range`が右辺値であるとき`ranges::find`の処理終了後に返されるイテレータはダングリングイテレータとなります。

しかし、C++20のRangeアルゴリズムはこれをコンパイル時に認識し、結果としてのイテレータを返す代わりに`ranges::dangling`のオブジェクトを返します。`dangling`は当然イテレータとして利用可能なものではないので、これをイテレータとして利用しようとするとコンパイルエラーになり、気付くことができます。

このような入力`range`のイテレータ/`subrange`を返すようなRangeアルゴリズムでは、入力の`range`型が`borrowed_range`のモデルとならない場合に`dangling`を返します。一方、入力の`range`型が`borrowed_range`のモデルとなり、戻り値の利用が安全であるときは通常の期待される結果を返します。

```cpp
// prvalueなrangeを返す関数
auto f() -> std::vector<int>;

int main() {
  auto result1 = ranges::find(f(), 42);
  // decltype(result1) == ranges::dangling

  auto vec = f();

  // 共に入力rangeはborrowed_rangeである
  auto result2 = ranges::find(vec, 42);
  auto result3 = ranges::find(subrange{vec}, 42);
}
```

この場合の`result2/result3`は正しく`std::vector<int>::iterator`が返されます。`subrange`は`enable_borrowed_range`が`true`となるように特殊化されているため、右辺値であっても`borrowed_range`のモデルとなります。

Rangeアルゴリズムにはイテレータペアを受け取るオーバーロードも用意されていますが、この仕組みは`range`を受け取るオーバーロードでしか働かないため、相対的に`range`を受け取るオーバーロードの方が安全に利用できる様になっています。

ただ、イテレータとして利用しようとしなければ、上記`result1`の様な`dangling`の初期化そのものはエラーとなりません。コードによっては実際に使用する所と初期化位置が離れてしまって、コンパイルエラーメッセージがわかりづらくなることもあるでしょう。なるべくその場で気付き、またエラーが起きてほしいものです。とはいえ一々`static_assert`したりイテレータ型を書いたりするのも面倒です。

そこで、コンセプトによる変数の制約を利用すると、返された`dangling`のオブジェクトを利用する前に`dangling`が返されていることに気付くことができます。

```cpp
// prvalueなrangeを返す関数
auto f() -> std::vector<int>;

int main() {
  // コンセプトによる変数の制約
  std::input_or_output_iterator auto result1 = ranges::find(f(), 42);
}
```

コンセプトによる変数の制約はこの様に`auto`で変数を宣言しているときにのみ使用することができ、`auto`の前にコンセプトを指定することによって制約します。この時、`C auto v = ...;`の様に指定されたコンセプト`C`には、`requires`式における戻り値型の制約の時と同様に第一引数が補われ、`C<decltype((v))>`の様な制約がチェックされることになります。

上記例では、`input_or_output_iterator<decltype((result1))>`の様に型を補われたコンセプトによる制約がチェックされています。`input_or_output_iterator`はイテレータとしての最小要件を定めるコンセプトであるため、`dangling`では構文的にすら満たすことができずにその場でコンパイルエラーが発せられます。しかし、戻り値として正しくイテレータが返っている場合はコンセプトによる制約をパスし、エラーにはなりません。

```cpp
// prvalueなrangeを返す関数
auto f() -> std::vector<int>;

int main() {
  // エラー
  std::input_or_output_iterator auto result1 = ranges::find(f(), 42);

  auto vec = f();

  // 共にOK
  std::input_or_output_iterator auto result2 = ranges::find(vec, 42);
  std::input_or_output_iterator auto result3 = ranges::find(subrange{vec}, 42);
}
```

少し変数宣言が長くなってしまうのが欠点ですが、それに見合った恩恵はあるかと思われます。特に、intellisenseのようなリアルタイムのコードチェックが働いている環境だと、より素早く的確に`dangling`が返されていることに気付くことができるでしょう。

### `borrowed_iterator_t/borrowed_subrange_t`

`ranges::borrowed_iterator_t/ranges::borrowed_subrange_t`は`ranges::dangling`の利用を簡易化するエイリアステンプレートです。

```cpp
namespace std::ranges {
  template<range R>
    using borrowed_iterator_t = conditional_t<borrowed_range<R>, iterator_t<R>, dangling>;

  template<range R>
    using borrowed_subrange_t =
      conditional_t<borrowed_range<R>, subrange<iterator_t<R>>, dangling>;
}
```

入力の型`R`が`borrowed_range`であるかによってイテレータ/`subrange`と`dangling`を切り替えます。

これはRangeライブラリのユーザーが直接使用するものではなく、主に`dangling`を返しうるRangeアルゴリズムの実装に使用されるものです。自分で`range`を受け取りイテレータや`subrange`を返すような処理を書くときにも使用できるかもしれません。

`range`を受け取る関数の戻り値型として、例えば次のように使用します。

```cpp
// 入力rangeのイテレータを返す処理
template<range R>
borrowed_iterator_t<R> my_algo_ret_iter(R&& r) {
  auto it = ranges::begin(r);
  
  // ...

  return it;
}

// 入力rangeのsubrangeを返す処理
template<range R>
borrowed_subrange_t<R> my_algo_ret_subr(R&& r) {
  auto it = ranges::begin(r);
  auto end = ranges::end(r);

  // ...

  return subrange{it, end};
}
```

この様に使用するだけで、入力`range`が`borrowed_range`でなければ`dangling`を返しそれ以外の場合は処理結果のイテレータ/`subrange`を返す、という事が出来ます。

関数実装内部で`borrowed_iterator_t/borrowed_subrange_t`が`dangling`を示すかどうかによって何を返すか切り替える必要があるのでは？と思われるかもしれませんが、`dangling`の定義をよく見るとコンストラクタが2つあり、2つ目のコンストラクタは任意の数の引数を受け取って何もせずに`ranges::dangling`を構築しています。この2つ目のブラックホールコンストラクタが呼ばれることによってイテレータ/`subrange`から`dangling`への暗黙変換が可能になっており、戻り値型に`borrowed_iterator_t/borrowed_subrange_t`さえ使っておけば内部の処理ではそのことを何ら気にしなくてもいいようになっているわけです。

`dangling`を返す場合でも内部の処理が通常通り行われている事が気になるかもしれませんが、`dangling`を返している場合というのは通常コンパイル時に気付くはずで、`dangling`を返している処理が実行されることはないはずです。もしコンパイル時に気付かなかったとすれば、それは戻り値を無視しているという事なのでそれはそれでバグでしょう。いずれにせよ、利用側では変数に対するコンセプトなどによって`dangling`が返されていることに素早く気付くようにしておく事が推奨されます。

## Rangeアルゴリズムの従来アルゴリズムとの差異

C++17までのアルゴリズムはイテレータペアを受け取って処理を行うものでしたが、C++20からのRangeアルゴリズムはイテレータペアに加えて`range`を直接受け取ることができます。

例えば`find`アルゴリズムで見てみると、以前は次の2つの宣言がありました

```cpp
namespace std {
  // 基本
  template<class InputIterator, class T>
  constexpr InputIterator find(InputIterator first, InputIterator last,
                               const T& value);

  // Parallel Algorithm
  template<class ExecutionPolicy, class ForwardIterator, class T>
  ForwardIterator find(ExecutionPolicy&& exec, ForwardIterator first, ForwardIterator last,
                       const T& value);
}
```

C++20 Rangeアルゴリズムでは次の様な2つが追加されます

```cpp
namespace std::ranges {
  // イテレータペアを受け取る
  template<input_iterator I, sentinel_for<I> S, class T, class Proj = identity>
    requires indirect_binary_predicate<ranges::equal_to, projected<I, Proj>, const T*>
  constexpr I find(I first, S last, const T& value, Proj proj = {});

  // rangeを受け取る
  template<input_range R, class T, class Proj = identity>
    requires indirect_binary_predicate<ranges::equal_to, projected<iterator_t<R>, Proj>, const T*>
  constexpr borrowed_iterator_t<R>
      find(R&& r, const T& value, Proj proj = {});
}
```

見辛いですね。どこが関数名なのかすら少し迷ってしまいます。説明のために、関数名を含む最低限のシグネチャだけを抜き出して比べてみると

\clearpage

```cpp
// 従来のfind
constexpr InputIterator find(InputIterator first, InputIterator last, const T& value);

// ranges::find 1
constexpr I find(I first, S last, const T& value, Proj proj = {});

// ranges::find 2
constexpr borrowed_iterator_t<R> find(R&& r, const T& value, Proj proj = {});
```

こうしてみると、追加されたのはイテレータペアを受け取るものと`range`を受け取るものの2つであることが分かるでしょうか。引数も`range`を受け取るものはイテレータペアの代わりに`range`オブジェクトを受けており、両方とも最後に謎の`proj`なるものが追加されている以外はそのままであることが分かります。

`proj`は射影（*Projection*）と呼ばれる機能で、後程詳しく説明します。

`ranges::find`の2つの気になる差は戻り値型に`borrowed_iterator_t`を使用しているか否かでしょう。イテレータペアを受ける方は使用していません。これは、入力が右辺値となりダングリングイテレータを返す危険があるのは`range`を受け取る時だけだとみなせるためで、同じ`range`についてのイテレータペアを取得しているという事は、少なくともその`range`オブジェクトは左辺値であるはずだからです。

```cpp
// prvalueなrangeを返す関数
auto f() -> std::vector<int>;

int main() {
  auto vec = f();

  // イテレータペアを取得するには、左辺値になっているはず
  auto it = ranges::find(vec.begin(), vec.end(), 20);
}
```

そもそもイテレータだけからでは元の`range`の有効性を判定できないというのもありますが、このような制約を考慮すればイテレータペアについては戻り値がダングリングとなる可能性は低いでしょう。もしそうなる場合はそもそも入力のイテレータからおかしいため、そのケアはユーザーの責任となります。なお、`ranges::begin/ranges::end`を使用すると、右辺値の（`view`などではない）`range`からイテレータ/番兵を取得できないためより安全です。

そして、従来のアルゴリズムとRangeアルゴリズムの特徴的な差異はRangeアルゴリズムのテンプレートパラメータは全てコンセプトによって制約されていることです。今度はRangeアルゴリズムのテンプレートパラメータの宣言部だけを見てみましょう。

```cpp
// ranges::find 1
template<input_iterator I, sentinel_for<I> S, class T, class Proj = identity>
  requires indirect_binary_predicate<ranges::equal_to, projected<I, Proj>, const T*>

// ranges::find 2
template<input_range R, class T, class Proj = identity>
  requires indirect_binary_predicate<ranges::equal_to, projected<iterator_t<R>, Proj>, const T*>
```

ここだけ見てみるとなんとなく同じようなことをしていることがうっすらと見えてきます。

ここでも*projection*関連は無視します。

イテレータペアを受け取るものは、それがきちんとイテレータペアとなっていることをコンセプトによって表現しています。`input_iteraotr`（入力イテレータを定義するコンセプト）`I`に対して`S`はその`sentinel_for<I>`でなければなりません。

一方で`range`を受け取る方は、それが`input_range`であることをシンプルに表現しています。

最後に残ったのは`indirect_binary_predicate`というコンセプトです。  
`indirect_binary_predicate<F, I1, I2>`は間接参照可能な型（イテレータやポインタ型）`I1, I2`の参照先の型によって`F`が呼び出し可能であり、その結果が`bool`となることを定義するコンセプトです。これは、`F, I1, I2`のオブジェクトをそれぞれ`pred, i1, i2`とすると、`bool c = pred(*i1, *i2)`の様な呼び出しが可能であることを表しています。

C++STLでは、1つ以上の引数を受け取ってそれについて何かを判定してその結果を`bool`で返す、様な関数の事を述語（*predicate*）と呼んでいます。この場合の`F`は2つの引数を受け取る必要があるため二項述語（*binary predicate*）と呼ばれ、さらにその引数は間接参照（*indirect read*）の結果として与えられる、という事を`indirect_binary_predicate`という名前は表しています。

これは`find`が内部でやることを考えると正当な要求であることが分かります。`ranges::find`は例えば次のように実装できるでしょう

```cpp
// ranges::findの簡易実装（projectionはないものとして）
template<input_iterator I, sentinel_for<I> S, class T>
  requires indirect_binary_predicate<ranges::equal_to, I, const T*>
constexpr I find(I first, S last, const T& value) {
  ranges::equal_to pred{};

  for (; first != last; ++first) {
    if (pred(*first, value) == true) return first;
    //  ^^^^^^^^^^^^^^^^^^^
  }

  // S -> Iの変換可能性は制約されていないので、firstを返す
  return first;
}
```

ここで使用されているように、`ranges::equal_to`は`==`による比較を行うための関数オブジェクトです。結果、`indirect_binary_predicate<ranges::equal_to, I, const T*>`というような制約は、上記のような実装においての`for`の中の判定部分の様な記述が可能であることを表現していることが分かります。（`indirect_binary_predicate`はイテレータ用のものなので、イテレータでも間接参照可能でもない`T`を直接指定することができません。そのため、最後の引数には`const T*`というポインタ型を渡すことで利用可能となるようにしています。）

また、`range`を受け取るオーバーロードはその`range`オブジェクトから取得したイテレータと番兵を、イテレータペアを受け取るオーバーロードに渡すことで実装できます。

```cpp
// rangeを受け取るfindの簡易実装
template<input_range R, class T>
  requires indirect_binary_predicate<ranges::equal_to, iterator_t<R>, const T*>
constexpr borrowed_iterator_t<R> find(R&& r, const T& value) {
  // イテレータペア版に委譲する
  return ranges::find(ranges::begin(r), ranges::end(r), value);
}
```

規格ではこの様な委譲による実装を実質的に規定しているため、実際のRangeアルゴリズム実装もこうなっているはずです。

### コンセプトによって制約された比較関数オブジェクト

先ほど出て来た`ranges::equal_to`はRangeライブラリの一部として導入された比較関数オブジェクトの一つです。従来あった`std::equal_to`をリファインするもので、同様に次の6つが導入されています。

- `ranges::equal_to`
    - `==`
- `ranges::not_equal_to`
    - `!=`
- `ranges::greater`
    - `>`
- `ranges::less`
    - `<`
- `ranges::greater_equal`
    - `>=`
- `ranges::less_equal`
    - `<=`

なお、これらのものは`<functional>`ヘッダに用意されています。

これらは関数オブジェクトではなく型として用意されているため、使用するにはオブジェクトを構築する必要があります。そして、その関数呼び出し演算子（`opreator()`）は2引数を取る関数テンプレートとして定義されており、テンプレートパラメータは行う比較についてきちんと制約されています。

```cpp
int main() {
  // デフォルト構築でok
  ranges::less le{};

  // 関数呼び出しによって比較を行う
  bool c = le(0, 1);
}
```

これらの型は一切のメンバ変数を持たず、全てのコンストラクタは宣言されていません。従って、デフォルト構築可能でありトリビアルコピー可能な型です。

関数オブジェクトではなく型として用意されているのは、比較を行う処理をテンプレートパラメータとして型で受け取るところで使用可能とするためです。例えば連想コンテナ（`std::map`など）がそうですが、関数オブジェクトとして定義されてしまっているとそういう所に直接渡すことができなくなってしまいます。関数テンプレートだと関数オブジェクトで受け取った方が都合がいい場合が多いですが、その場合でも関数オブジェクトのデフォルトとしてテンプレートパラメータのデフォルト引数に指定しておく事ができます。

```cpp
// 型で渡して比較をカスタマイズする
template<typename Key>
using myset = std::set<Key, ranges::greater>;

// 比較型のデフォルトとして利用
template<Comp = ranges::less>
void custom_cmp(Comp cmp = {});

// 関数オブジェクトとして渡す
custom_cmp(ranges::less_equal{});
```

また、これらにはメンバ型として`is_transparent`が定義されており、連想コンテナにおける透過的な操作に対応しています。

## 射影（*Projection*）

射影（プロジェクション）とは、RangeアルゴリズムにみられるRangeライブラリの新機能で、クラス型のオブジェクトからそのメンバ変数の1つを取り出すような操作の事です。

```cpp
// pairのシーケンス
vector<pair<int, double>> vec = { ... };

// pair<int. double>の1つ目のメンバが20なものを探す
auto it = ranges::find(vec, 20, [](const auto& p)  { return p.first; });
```

ここで、`ranges::find`の最後の引数として渡しているラムダ式がプロジェクションです。

```cpp
// rangeを受け取る
template<input_range R, class T, class Proj = identity>
  requires indirect_binary_predicate<ranges::equal_to, projected<iterator_t<R>, Proj>, const T*>
constexpr borrowed_iterator_t<R>
    find(R&& r, const T& value, Proj proj = {});
                             // ^^^^^^^^^^^^^^
```

プロジェクションとは、入力シーケンスの1つの要素からその要素が内包する1つのメンバ変数を取り出す関数呼び出し可能なものの事を言います。ラムダ式はそのもっともわかりやすい例でしょう。

何が嬉しいのかというと、記述すべきことが減ることでバグを減らすことが出来る点にあります。たとえば、従来の`std::sort`で`pair`のシーケンスの様な物をソートしようとすると、カスタム比較を行う関数オブジェクトを渡す必要がありました。

```cpp
// pairのシーケンス
vector<pair<int, double>> vec = { ... };

// pairの1つ目のメンバでソートしたい
sort(vec.begin(), vec.end(), 
     [](const auto& p1, const auto& p2) { return p1.first < p2.first; });
```

ここでやりたい本来の事は`pair`のシーケンスの1つ目のメンバでソートを行う事ですが、実際に指定しているのは2つの`pair`オブジェクトの比較をどう行うか、という事を書かされています。別に比較がカスタマイズしたいわけではなく、`pair`の1つ目のメンバを取り出したいだけなのです。

たとえば次のようにいくつかのミスパターンを想像することができますが、どれもコンパイルエラーは起きず、気付くことは難しいかもしれません。

```cpp
// 比較演算子の指定を間違えた
sort(vec.begin(), vec.end(),
     [](const auto& p1, const auto& p2) { return p1.first > p2.first; });

// 比較するメンバを間違えた
sort(vec.begin(), vec.end(),
     [](const auto& p1, const auto& p2) { return p1.first < p2.second; });

// 比較する変数を間違えた
sort(vec.begin(), vec.end(),
     [](const auto& p1, const auto& p2) { return p1.first < p1.first; });
```

このようなカスタム比較を指定するこれまでのカスタマイズの方法は、特定のメンバ変数の引き当てと比較のカスタマイズという2つの事を同時に指定せざるを得ないために記述量が増え、些細な、しかし気付き辛いバグの原因となってしまっていました。

プロジェクションはこれを解決するための機能です。プロジェクションを導入することによって従来のカスタマイズ方法は「特定のメンバ変数を引き当てる」と「比較のカスタマイズ」という2つのカスタマイズポイントに分割され、それぞれに指定するカスタマイズ処理はそれぞれの事にしか責任を負わなくなります。

```cpp
// pairの1つ目のメンバでソートする
ranges::sort(vec.begin(), vec.end(), {}, [](auto& p) { return p.first; });
```

`ranges::sort`は3番目の引数に比較関数オブジェクト（デフォルトは先程の`ranges::equal_to`のファミリである`ranges::less`）を、最後の引数にプロジェクションを取ります。プロジェクションだけを使用する場合は比較をカスタマイズする必要はなく（`{}`でいい）、特定のメンバ変数を引き当てるための最低限の記述で済みます。仮に両方のカスタマイズが必要だとしても、比較のカスタマイズは殆どの場合`ranges::less`をはじめとする関数オブジェクトを使用すれば済み、プロジェクションの指定はどう比較するかとは常に無関係に記述できます。

なおプロジェクションという名前ですが、集合論において直積集合の元である順序対からその成分の1つを取り出すような写像の事を射影と呼び、Rangeライブラリにおける射影（*Projection*）はそこから来ています。C++における構造体はメンバ変数の型を1つの集合と見た時のそれらの直積集合とみなすことができ、そのオブジェクトとは直積集合の元である順序対に対応しています。そして、構造体のオブジェクトからその変数の1つを取り出すことは、順序対からその成分の1つを取り出す写像（射影）に対応しています。

### メンバ変数ポインタ

プロジェクションの導入によってメンバ変数の1つを取り出すことが書きやすくなったのですが、その記述にはまだ問題があります。

```cpp
// pairの1つ目のメンバでソートする
ranges::sort(vec.begin(), vec.end(), {}, [](auto& p) { return p.first; });
```

ラムダ式の戻り値型は推論されたうえで`decay`されるため、このラムダ式は`p.first`をコピーして返しています。`int`型なら問題ないですが、コピーが重い型だと思わぬオーバーヘッドに繋がります。それを回避するには戻り値型を指定してやればいいのですが、知らなければそれを行えないし、うっかり忘れる事を防ぐ事は難しいでしょう。同様の事は引数型についても言えます。あと、ラムダ式で引き当てるのは簡易ではありますがそれでも記述量が少ないとは言えません。

ところで、プロジェクションは`std::invoke`を用いた関数呼び出しによって実装されています。たとえば先ほどの簡易実装`ranges::find`で書いてみると

```cpp
// ranges::findの簡易実装 rev2
template<input_iterator I, sentinel_for<I> S, class T, class Proj = identity>
  requires indirect_binary_predicate<ranges::equal_to, projected<I, Proj>, const T*>
constexpr I find(I first, S last, const T& value, Proj proj = {}) {
  ranges::equal_to pred{};

  for (; first != last; ++first) {
    // std::invokeを用いて呼び出しを行う
    if (pred(invoke(proj, *first), value) == true) return first;
    //       ^^^^^^^^^^^^^^^^^^^^
  }

  return first;
}
```

`std::invoke`は関数呼び出し可能なものから可能な限り関数呼び出しを行おうとします。そして、その中にはメンバ変数ポインタからの呼び出しが含まれています。これを利用すると、先ほどのソートは次のように書くことができます。

\clearpage

```cpp
// pairの1つ目のメンバでソートする
ranges::sort(vec.begin(), vec.end(), {}, &pair<int, double>::first);
```

これ（`&pair<int, double>::first`）は`std::pair`の1つ目のメンバ変数`first`を指定するメンバ変数ポインタです。`std::invoke`は型`T`のメンバ変数ポインタ`p`と`T`のオブジェクト（の参照）`o`によって`std::invoke(p, o)`のように呼び出されると、`o.*p`の結果を返してくれます。`.*`はメンバポインタ演算子という演算子で、`o`が左辺値ならば`o.*p`の結果は`o`のメンバ変数への左辺値参照を返すため、深く考えなくてもコピーされません。そして、メンバ変数ポインタの書式は構文にバリエーションがなく曖昧さが少ないため、ミスが入り込む余地が減ります。

このように、プロジェクションにメンバ変数ポインタを利用することで、特定のメンバ変数を引き当てるという処理をより簡潔かつ正確に書くことができるようになります。ただしこれは対象のメンバ変数が`public`であるときにしか使えません。

### `identity`

プロジェクションを取るテンプレートパラメータのデフォルト引数には、`identity`という型が指定されています。これはデフォルトのプロジェクションとなる関数オブジェクトの型で、受け取った引数を何もせずそのまま返すものです。

```cpp
namespace std {
  struct identity {

    template<class T>
    constexpr T&& operator()(T&& t) const noexcept {
      // 引数を値カテゴリも含めてそのまま返す
      return std::forward<T>(t);
    }

    using is_transparent = unspecified;
  };
}
```

Rangeアルゴリズムにプロジェクションを何も指定しない場合はこの何もしない関数が射影として使用されており、すなわちイテレータの間接参照の結果をそのまま利用するという事になります。Rangeアルゴリズムにおけるプロジェクションは、入力の範囲1つごとに1つを引数列の最後で受け取ります。`identity`がデフォルトで指定されていることで、プロジェクションを使用しないときはその指定を省略することができます。

### `projected`

先ほど見ていた`ranges::find`の`indirect_binary_predicate`コンセプトによる制約には謎の型？`projected`が出現していました。

```cpp
template<input_range R, class T, class Proj = identity>
  requires indirect_binary_predicate<ranges::equal_to, 
                                     projected<iterator_t<R>, Proj>,
                                  // ^^^^^^^^^
                                     const T*>
```

`indirect_binary_predicate`を含めて`indirect_xxx`というコンセプトがいくつかあるのですが、それらは基本的に`indirectly_readable`な型（間接参照可能な型）で使用するために設計されているため、「イテレータの間接参照結果を射影した結果」を直接用いることができません。`ranges::find`の場合は比較する値として`T`の値を受けますが、それはイテレータでもなくそれを間接参照しても意味はないので直接使えず、`const T*`を代わりに渡すことで解決しています。

`std::projected<I, P>`はプロジェクション`P`によって間接参照可能な型`I`の間接参照結果を射影した結果をもう一度間接参照可能な型に戻すための型です。これによって`indirect_xxx`なイテレータ用のコンセプトで、イテレータに対するプロジェクションの結果を直接利用することができるようになります。

```cpp
namespace std {
  template<indirectly_readable I, indirectly_regular_unary_invocable<I> Proj>
  struct projected {
    using value_type = remove_cvref_t<indirect_result_t<Proj&, I>>;

    indirect_result_t<Proj&, I> operator*() const;  // 定義されない
  };
}
```

`operator*()`の戻り値型として`I`に`Proj`を適用した結果を示し、`std::projected<I, P>`が有効であるときは`indirectly_readable`コンセプトのモデルとなります。要は`ranges::find`における値型`T`で`indirect_xxx`なコンセプトを利用可能とするために`const T*`というポインタ型を渡す、というのと同じことを少し小難しくイテレータとプロジェクションに対してしているだけです。

`indirectly_regular_unary_invocable<Proj, I>`というのは、`Proj`のオブジェクトを`proj`、`I`のオブジェクトを`i`としたときに、`invoke(proj, *i)`の様な呼び出しが可能であることを表すコンセプトです。見たままプロジェクションですね。`indirect_result_t<Proj&, I>`は`invoke(proj, *i)`の結果型を表すエイリアステンプレートで、`Proj&`となっているのは左辺値`proj`から呼び出しを行う事を指定しています。見たままプロジェクションの結果型の事です。

なお、`std::projected`はこの様な目的の型であるためその間接参照操作（`operator*()`）は定義されておらず、実際に呼び出すことは出来ません。これはあくまでコンセプトの文脈で使用するための型レベルのものです。

## Rangeアルゴリズムの変更点

C++20で追加されるRangeアルゴリズムは多岐に渡りますが、どれも`<algorithm>`ヘッダに元々あるものに対応するものだけで新しく追加されるものはありません。そして、従来のアルゴリズムと比較すると次のような変更がなされています。

1. `std::ranges`名前空間に配置される
2. 元の関数1つに対して、イテレータペアを取るものと`range`を取る物の2種類が追加される
    - カスタマイズ述語を取るかとらないかのオーバーロードは統合されている
    - *Paralell Algorithm*に対応するものは無い
3. 受け取る範囲1つ（イテレータペア/`range`）についてプロジェクションを1つ受け取る
    - プロジェクションは引数の最後で受け取る
4. テンプレートパラメータは全てコンセプトによって必要な制約がなされている
5. 一部戻り値型が変更されているものがある
    - `pair<I1, I2>`から同等の集成体型
    - イテレータから追加の情報を持つ集成体型
    - イテレータから`subrange`、など

特に1～4の変更点は全てのものについて言えます。その一端は先ほどの`ranges::find`で垣間見ることができました。

変更されたのは関数のインターフェースの部分だけで、内部でやることは変更されていません。そのため、全体としてこれらの変更がなされていることを知っていれば、Rangeアルゴリズムを使用していくのに支障はなくなるかと思われます。Rangeアルゴリズムの宣言は複雑になってしまっていますが、実際に使用するときは簡単かつ安全に使用できるようになっています。

また、この様な変更はC++20では`<algorithm>`の関数群だけが対象で、`<numeric>`の関数群にはRangeアルゴリズムは追加されていません。これはC++23にて対応するものが追加されるか、あるいはリファインされた新しい関数が追加される予定です。

## ADLの無効化

`std::ranges`名前空間にあるRangeアルゴリズムはADLによって発見されない事が規定されています。例えば次のような場合

```cpp
void foo() {
  using namespace std::ranges;
  std::vector<int> vec{1, 2, 3};

  find(begin(vec), end(vec), 2);  // #1
}
```

#1の箇所で呼び出されるのは常に`ranges::find`であり、`std::find`が呼び出されることはありません。`std::vector`のイテレータは`std`名前空間にあるのでADL経由で`std::find`を発見でき、オーバーロード解決のルールではより特殊化されている方が優先されるため、テンプレートパラメータ1つでイテレータペアを受けている`std::find`が優先されそうに見えますがそうではなく、そもそもADL以前の名前探索で`ranges::find`が発見されているとADLが行われません。

このような事は`ranges::find`に限らずRangeアルゴリズム全てに対して規定されており、RangeアルゴリズムはADL以前に発見されている場合にADLを妨げ無効化する効果を持っています。これは、Rangeアルゴリズムが従来のアルゴリズムをリファインする新しいものであるため、ADLを介して意図せず古い関数が使用されることを防止するための仕組みです。

そして、このような効果は普通の関数テンプレートでは実現できないため、Rangeアルゴリズムは実のところ関数テンプレートではありません。おそらく、関数オブジェクトとして実装されます。

\clearpage

# viewその1 - Rangeファクトリ

C++20でRangeライブラリに導入される`view`には大きくRangeアダプタ（*range adaptors*）とRangeファクトリ（*range factories*）の2つがあり、まずはそのうちの一つであるRangeファクトリと呼ばれる`view`を見ていきます。

Rangeファクトリは極単純な範囲を保持する`view`を生成するための型および操作のカテゴリで、ここの`view`達は単独で動作して新しい範囲を生成するような振る舞いをします。

## `empty_view`

`empty_view`は指定された型`T`の空の範囲を生成する`view`です。

```cpp
int main() {
  
  empty_view<int> ev{};

  for (int n : ev) {
    assert(false);  // 呼ばれない
  }
}
```

この定義はとても単純で、次のようになっています。

```cpp
namespace std::ranges {
  template<class T>
    requires is_object_v<T>
  class empty_view : public view_interface<empty_view<T>> {
  public:
    static constexpr T* begin() noexcept { return nullptr; }
    static constexpr T* end() noexcept { return nullptr; }
    static constexpr T* data() noexcept { return nullptr; }
    static constexpr size_t size() noexcept { return 0; }
    static constexpr bool empty() noexcept { return true; }
  };
}
```

これは他の`view`（特にRangeアダプタ）の実装において、入力によって空の範囲を返す必要がある場合などに使用されているようです。

### `empty_view<T>`の諸特性

`view`の諸特性とは`view`あるいは`range`全般に共通して見出すことのできる性質で、利用するにあたって予め知っておくと便利なものです。しかし、`view`の種類や`view`の元となる`range`によって変化し、場合によってどうなっているか判断するのが難しい事があります。

- `reference` : `T&`
- `range`カテゴリ : `contiguous_range`
- `common_range` : ◯
- `sized_range` : ◯
- `const-iterable` : ◯
- `borrowed_range` : ◯

これらの特性はそれぞれ次のような意味です。

- `reference` : イテレータの間接参照の結果型（`decltype(*i)`）
- `range`カテゴリ : どの`range`コンセプトのモデルとなるか
- `common_range` : `common_range`コンセプトのモデルとなるか
- `sized_range` : `sized_range`コンセプトのモデルとなるか
- `const-iterable` : `const`化したときでも`range`でいられるか
- `borrowed_range` : `borrowed_range`コンセプトのモデルとなるか

`const-iterable`であることは、`range`型`R`に対して`input_range<const R>`の様にコンセプトで表現されます。`range`を`const`にしても依然として`input_range`であるならば、それは`const-iterable`であり、その`range`オブジェクトの変更を伴わずに範囲をイテレートできます。ただし、`const-iterable`であることは、`const`な`range`の要素についての`const`性を必ずしも意味していないことに注意が必要です。特に、範囲を所有しないタイプの`view`型がそれに当たります。

\clearpage

### `views::empty`

`empty_view`に対応する操作を行うためのものが`std::ranges::views::empty`として用意されています。

```cpp
int main() {

  for (int n : views::empty<int>) {
    assert(false);  // 呼ばれない
  }
}
```

この`views::empty`は変数テンプレートであり、型`T`を受け取って`empty_view<T>`を返します。こちらを用いると、空の範囲を取得すると言う操作を少しだけ簡潔に書くことができます。なお、`std::ranges::views`名前空間は`std::views`としてアクセスすることができ、短くなって意味もわかりやすくなるのでそちらを利用するのがおすすめです。

実際のところ*range factories*というのは`std::ranges::views`名前空間にあるこれらの変数または関数オブジェクトの事を言い、こちらの操作が主体であって`view`型はあくまでその実装の一部に過ぎないわけです。そのため多くの場合、対応する`view`型そのものを使用するよりも*range factories*を介して使用した方がジェネリックかつ自然に書くことができるようになっており、利用の際にもこれらの*range factories*を使用する事がほとんどでしょう。

本書では説明のためにこれらのもののことを「Rangeファクトリオブジェクト」と呼ぶことにします。

## `single_view`

`single_view`は指定された要素1つだけからなる範囲を生成する`view`です。

```cpp
int main() {
  single_view sv{20};

  for (int n : sv) {
    std::cout << n; // 20
  }
}
```

これはある値に対して`range`を取るアルゴリズムを再利用したい場合など、単一の値をシーケンスに変換したい場合に使用できるでしょう。

`single_view`は次のように定義されています

```cpp
namespace std::ranges {
  template<copy_constructible T>
    requires is_object_v<T>
  class single_view : public view_interface<single_view<T>> {
    // ...
  };

  template<class T>
    single_view(T) -> single_view<T>;
}
```

要素型`T`はコピー構築可能であり、オブジェクト型でなければなりません。例えば参照型の`single_view`などは作れない事になります（ポインタ型はOK）。

`single_view`のコンストラクタは値をコピーorムーブするためのものが用意されていますが、中には*in place*構築を行うためのコンストラクタがあります。

```cpp
int main() {
  
  // std::stringのコンストラクタを呼び出してもらう
  std::ranges::single_view<std::string> sv(std::in_place, "in place construct", 8);

  for (auto& str : sv) {
    std::cout << str; // in place
  }
}
```

ただし、このコンストラクタを利用しようとするとクラステンプレートの実引数推論を使用する事ができないため、明示的に要素型を指定してあげる必要があります。

### `single_view<T>`の諸特性

- `reference` : `T&`
- `range`カテゴリ : `contiguous_range`
- `common_range` : ◯
- `sized_range` : ◯
- `const-iterable` : ◯
- `borrowed_range` : ×

`single_view`はその唯一つの要素を内部で保持しているため、`borrowed_range`のモデルではありません。しかし、`view`コンセプトのモデルにはなっています。

### `views::single`

`single_view`に対応する操作であるRangeファクトリオブジェクトとして、`views::single`が用意されています。

```cpp
int main() {
  for (int n : std::views::single(20)) {
    std::cout << n; // 1度だけ呼ばれる
  }
}
```

この`views::single`はカスタマイゼーションポイントオブジェクトであり、引数の式`arg`によって`views::single(arg)`のように呼び出された時、`single_view<decay_t<decltype((arg))>>(arg)`を返します。ただし、このCPOは1引数しか受け付けないため*in place*コンストラクタを呼び出すことはできません。

## `iota_view`

`iota_view`は渡された2つの値をそれぞれ始点と終点として、始点から終点にむけて単調増加するシーケンスを作成する`view`です。話を整数型に限定するならば、`init, bound`の2つの値を渡すと`[init, bound)`の範囲で1づつ増加していく数列を生成します。

```cpp
int main() {
  // [1, 10)の範囲の整数列を生成
  iota_view iv{1, 10};

  for (int n : iv) {
    std::cout << n; // 123456789
  }
}
```

また、引数1つだけで構築することもでき、その場合は終端のない無限列を生成します。

```cpp
int main() {
  // [1, ∞)の整数列を生成
  iota_view iv{1};

  for (int n : iv) {
    std::cout << n;     // 1234567891011121314151617181920
    if (n == 20) break; // 何かしら終わらせる条件がないと無限ループ
  }
}
```

基本的には整数列を生成するために使用すると思われますが、`iota_view`の実体はインクリメント可能であり距離を定義できさえすればどんな型の単調増加シーケンスでも作成可能です。

```cpp
namespace std::ranges {
  // iota_viewの定義の例
  template<weakly_incrementable W, semiregular Bound = unreachable_sentinel_t>
    requires weakly-equality-comparable-with<W, Bound> && copyable<W>
  class iota_view : public view_interface<iota_view<W, Bound>> {
    // ...
  };
}
```

型`W, Bound`はそれぞれ生成する範囲の先頭、終端を表す型です。それぞれのオブジェクトを`w, b`とすると、`[w, b)`の範囲のシーケンスを生成します。`W`は`weakly_incrementable`であるので、`++`によるインクリメントが可能でありさえすればどんな型のシーケンスでも生成する事ができます。`Bound`はその上界（番兵）を指定するもので、デフォルトで指定されている`unreachable_sentinel_t`はあらゆる型との比較に常に`true`を返す特殊な番兵型で、別の方法で範囲の終端が指定される場合に用いる事ができるものです。すなわち、1引数で`iota_view`を構築した場合にこれが使用され、`W`の単調増加無限列を表すことになります。

`requires`節の制約は`W, Bound`が`== !=`で相互に比較可能であり、`copyable`コンセプトは名前通り型`W`がコピー構築/代入可能であることを要求しています。

この性質によって例えば、ポインタ型やイテレータ型のシーケンスを作成可能です。

```cpp
int main() {

  int array[] = {2, 4, 6, 8, 10};

  // ポインタのシーケンス
  iota_view iva{ranges::begin(array), ranges::end(array)};

  for (int* p : iva) {
    std::cout << *p;  // 246810
  }

  std::list list = {1, 3, 5, 7, 9}; 

  // listイテレータのシーケンス
  iota_view ivl{ranges::begin(list), ranges::end(list)};

  for (auto it : ivl) {
    std::cout << *it; // 13579
  }
}
```

なお、浮動小数点数型は`weakly_incrementable`のモデルとならないため`iota_view`では使用できません。2つの値の間の距離を整数値で定義できないためです。

### 遅延評価

`iota_view`によって生成されるシーケンスは`iota_view`オブジェクトを構築した時点では生成されておらず、各要素の生成は遅延評価されます。

具体的には、`iota_view`オブジェクトから得られるイテレータのインクリメント（`++i/i++`）のタイミングで1つづつシーケンスの要素が計算されます。

```cpp
int main() {
  iota_view iv{1, 10};
  auto it = ranges::begin(iv);
  // この段階ではまだ何もしてない

  int n1 = *it; // 初項（1）が得られる
  ++it;         // ここで次の項（2）が計算される
  it++;         // ここで次の項（3）が計算される
  int n2 = *it; // 3番目の項（3）が得られる  
}
```

`iota_view`のイテレータと番兵は`iota_view`に渡された値`[w, b)`をそれぞれ保持しており、インクリメントや同値比較の際に利用します。`iota_view`の行うことの大部分はイテレータによって行われているわけです。また、遅延評価されていることによって、終端を指定してもしなくても（無限列生成時でも）同じ処理によってシーケンスを生成しています。

### `iota_view<W, B>`の諸特性

- `reference` : `W`
- `range`カテゴリ
    - `W`が`advanceable` : `random_access_range`
    - `W`が`decrementable` : `bidirectional_range`
    - `W`が`incrementable` : `forward_range`
    - それ以外 : `input_range`
- `common_range` : `W, Bound`が同じ型である場合
- `sized_range` : 次のいずれかの場合
    - `common_range`かつ`random_access_range`となる時
    - `W, Bound`がともに整数型である時
    - `W, Bound`が`sized_sentinel_for<Bound, W>`のモデルとなる時
- `const-iterable` : ◯
- `borrowed_range` : ◯

`std::incrementable`と言うのは`std::weakly_incrementable`より少し強い制約で、自身との等値比較や後置`++`が自身と同じ型を返す事などが追加で求められるものです。`decrementable`はそれに加えてデクリメント操作が可能であるもので、`advanceable`はさらに`+ - += -=`によって進行する事が可能である事を要求します（なお、`incrementable`は標準コンセプトですが他の2つは説明専用のものです）。すなわち、`W`について各種イテレータと同様の操作による値の増加・減少が可能である時に、対応するイテレータと同じカテゴリの`range`になります。

`iota_view`は自身で範囲を所有しているタイプの`range`ですが、`view`および`borrowed_range`のモデルとなるように実装されます。自身で範囲を所有しているにも関わらず`view`および`borrowed_range`の要求を満たす事ができるのは、シーケンスが遅延評価されていることによって可能となっています。

### `views::iota`

`iota_view`に対応する操作であるRangeファクトリオブジェクトとして、`views::iota`が用意されています。

```cpp
int main() {
  for (int n : views::iota(1, 10)) {
    std::cout << n;   // 123456789
  }
  
  std::cout << '\n';

  for (int n : views::iota(1)) {
    std::cout << n;   // 1234567891011121314151617181920
    if (n == 20) break;
  }
}
```

\clearpage

この`views::iota`はカスタマイゼーションポイントオブジェクトであり、引数の式`w`によって`views::iota(w)`のように呼び出された時、あるいは`w, b`によって`views::iota(w, b)`のように呼び出された時、その効果は次のようになります

1. 1引数で呼ばれた場合 : `iota_view(w)`
2. 2引数で呼ばれた場合 : `iota_view(w, b)`

## `istream_view`

`istream_view<T>`は任意の`istream`オブジェクトが示す入力ストリーム上にある`T`の値のシーケンスを生成する`view`です。

```cpp
int main() {
  // 標準入力に"1 2 3 4 5 6"のように入力したとする
  for (int n : istream_view<int>(std::cin)) {
    std::cout << n; // 123456
  }
}
```

要は`istream_iterator`を`view`型として再設計したものです。`iostream`にアダプトしているため、正確には`basic_istream`によって表現される入力ストリームを受け取ることができます。

```cpp
int main() {
  auto iss = std::istringstream{"0 1  2   3     4"};

  for (int n : istream_view<int>(iss)) {
    std::cout << n; // 01234
  }
}
```

### `basic_istream_view`

`istream_view<T>`は操作に対応する関数（Rangeファクトリ）であり、実体は`basic_istream_view`というクラスによって実装されています。

\clearpage

```cpp
namespace std::ranges {
  // istream_view<T>は関数
  template<class Val, class CharT, class Traits>
  basic_istream_view<Val, CharT, Traits> istream_view(basic_istream<CharT, Traits>& s) {
    return basic_istream_view<Val, CharT, Traits>{s};
  }
}
```

型`CharT, Traits`は`basic_istream`のテンプレートパラメータであり、ストリームのオブジェクトがあれば推論することができます。ユーザーが指定すべきなのは、どの入力ストリームを使うのかと、そこから取り出したい値の型を指定することです。

`basic_istream_view`は次のように定義されています。

```cpp
namespace std::ranges {
  template<movable Val, class CharT, class Traits>
    requires default_initializable<Val> &&
             stream-extractable<Val, CharT, Traits>
  class basic_istream_view : public view_interface<basic_istream_view<Val, CharT, Traits>> {
  public:
    constexpr explicit basic_istream_view(basic_istream<CharT, Traits>& stream);

    constexpr auto begin() {
      *stream_ >> value_;
      return iterator{*this};
    }

    constexpr default_sentinel_t end() const noexcept;

  private:
    // プライベートメンバは説明専用
    struct iterator;
    basic_istream<CharT, Traits>* stream_;
    Val value_;
  };
}
```

ストリームの値型`Val`は`movable`かつデフォルト構築可能である必要があります。`stream-extractable`は説明専用のコンセプトであり、`<<`による入力操作によって型`Val`の値が得られることを要求しています。番兵型である`default_sentinel_t`は、イテレータ自身で範囲の終端を示す事ができない場合に利用可能な簡易な番兵型です。

この`basic_istream_view`を直接使おうとすると`Val`を必ず指定しなければならない事からクラステンプレートの実引数推論を利用できず、3つのパラメータ全てを指定することになってしまいます。それを回避するために`istream_view<Val>()`が用意されています。

### 遅延評価

`basic_istream_view`は遅延評価されます。`basic_istream_view`によって生成されるシーケンスは`basic_istream_view`オブジェクトを構築した時点では生成されていません。

まず、`basic_istream_view`オブジェクトから`begin()`によってイテレータを取得した時点で最初の要素が計算（ストリームから読み取り）されます（これは先ほどの定義の`begin()`に見ることができます）。そして、インクリメント（`++i/i++`）のタイミングで1つづつ後続の要素が計算されます。

```cpp
int main() {
  // 標準入力に"1 2 3 4 5 6"のように入力したとする

  auto iv = istream_view<int>(std::cin);
  // この段階ではまだ何もしてない

  auto it = ranges::begin(iv); 
  // この段階で最初の要素（1）が読み取られる

  int n1 = *it; // 最初の要素（1）が得られる
  ++it;         // ここで次の要素（2）が読み取られる
  it++;         // ここで次の要素（3）が読み取られる
  int n2 = *it; // 3番目の要素（3）が得られる  
}
```

読み取られた要素は`basic_istream_view`オブジェクトの内部に保存されており、仮に同じ`basic_istream_view`オブジェクトからイテレータを2つ以上取得していても、片方のイテレータの操作はもう片方のイテレータに影響を与えます。すなわち、`basic_istream_view`の生成するシーケンスは一方向性でマルチパス保証がありません。

この事によって、`basic_istream_view`によるシーケンスは通常のシーケンスとは異なりメモリ上に空間的に存在するのではなく、ストリーム上に時間的に存在していることができます。すなわち、`basic_istream_view`を構築したタイミングで入力データが全て到着している必要はなく、任意のタイミングで到着しても構いません。この事は、C#におけるLINQに対するRxの対応と同じです。

シーケンス終端の判定（`==/!=`）はストリーム上にデータが残っているかによって行われ、`std::basic_ios`の`operator bool`が用いられます。入力が無い（終端に到達した）状態でイテレータをインクリメントする事は意味論としては未定義動作に当たりますが、おそらくブロックすることになります。

### `basic_istream_view<V, C, T>`の諸特性

- `reference` : `V&`
- `range`カテゴリ : `input_range`
- `common_range` : ×
- `sized_range` : ×
- `const-iterable` : ×
- `borrowed_range` : ×

番兵型は常に`std::default_sentinel_t`が利用されるため`common_range`にはならず、シーケンスにマルチパス保証がないため常に`input_range`であり、自身の内部に読み取った要素を保存する事から`borrowed_range`や`const-iterable`となれず、当然そのサイズをあらかじめ知ることもできません。

`basic_istream_view`は、そのイテレータをコピーすることもできない`range`型としてほとんど最低限の操作しかできない、少し特殊な`view`です。しかし、この様な扱いの難しいシーケンスであっても`view`という形に落とし込むことができるという例でもあり、プログラム外部環境に依存するなど特殊なシーケンスを`view/range`化する際の参考にすることができます。

\clearpage

# viewその2 - Rangeアダプタ

2つ目の`view`のグループ、Rangeアダプタ（*range adaptors*）と呼ばれる`view`は、他の`range`に対して作用して特定の操作を適用した`view`に変換するものです。Rangeアダプタは`range`から`range`へ操作を適用しつつ変換するものなので、Rangeアダプタの結果にさらにRangeアダプタを適用する形で操作をチェーンさせることができ、その結果もまた`range`（`view`）として得られます。そして、Rangeアダプタによって生成されるシーケンスは可能な限り遅延評価され、何処かのタイミングでシーケンスの全体を計算してそれを所有するようなものではありません（この事は、`view`コンセプトの要請でもあります）。

Rangeファクトリはシーケンスを生成するタイプの`view`なのでRangeアダプタのように他の`range`に作用することはできませんが、Rangeアダプタと比較して見るとRangeファクトリはRangeアダプタによるチェーンの起点となる`range`として利用するものである事がわかります。

## Rangeアダプタオブジェクト

Rangeファクトリに操作であるRangeファクトリオブジェクトがあったように、Rangeアダプタの操作もRangeアダプタオブジェクト（*range adaptor object*）が担います。Rangeアダプタオブジェクトには対応する`view`がありますが、Rangeアダプタとは操作である関数オブジェクトの事を指しており、こちらが主体であって対応する`view`はそれらRangeアダプタの実装の一部でしかありません。そのため多くの場合、対応する`view`型そのものを使用するよりもRangeアダプタオブジェクトを使用した方がジェネリックかつ自然に書くことができるようになっており、利用の際にもこれらのRangeアダプタオブジェクトを使用する事がほとんどでしょう。

Rangeアダプタオブジェクトは`std::ranges::views`名前空間に配置されており、簡易化のために`std::views`からも呼び出す事ができます。

### パイプライン演算子（`|`）と関数呼び出し

Rangeアダプタオブジェクトはカスタマイゼーションポイントオブジェクト（CPO）であり、第1引数に`viewable_range`を受け取り、結果として`view`を返します。中にはさらに追加の引数をとるものもあります。

Rangeアダプタオブジェクトはさらに、パイプライン演算子（`|`）による`range`オブジェクトへの操作の適用もサポートしており、関数呼び出しとパイプライン演算子の両方によって利用する事ができます。そして、それらによる呼び出しはほとんど等価の振る舞いをします。その2つの呼び出しの差は、`range`オブジェクトを第一引数で受け取るかパイプラインで受け取るかだけです。

```cpp
// R, R1, R2を任意のRangeアダプタオブジェクトとする

// 入力のrange(viewable_range)オブジェクト
viewable_range auto vr = views::iota(1);

// この2つの呼び出しは同じViewを返す
view auto v1 = R(vr);
view auto v2 = vr | R ;

// この3つの呼び出しも同じViewを返す、さらにRangeアダプタが増えても同様
view auto v3 = R2(R1(vr));
view auto v4 = vr | R1 | R2;
view auto v5 = vr | (R1 | R2);

// Rangeアダプタが追加の引数を取るときも同様に書く事ができる
// その時でも、この3つは同じViewを生成する
view auto v6 = R(vr, args...);
view auto v7 = R(args...)(vr);
view auto v8 = vr | R(args...)
```

ここでは`views::iota`を例に用いていますが、それが任意の`viewable_range`オブジェクトであっても同じ事が成り立ちます。なお、このパイプライン演算子は新しい演算子ではなく既存のビット論理和演算子をオーバーロードしたものです。

Rangeアダプタは例えば次のように使用する事ができます。

```cpp
auto even = [](int i) { return 0 == i % 2; };
auto square = [](int i) { return i * i; };

int main() {
  for (auto v : views::iota(1) | views::drop(4)
                               | views::filter(even)
                               | views::transform(square)
                               | views::take(5))
  {
    std::cout << v << ", ";
    // 36, 64, 100, 144, 196,
  }
```

各`view`が何をしているのかはこれから説明していきますが、パイプラインスタイルによる記述とその名前からどのような順序で何をしているかがかなり読み取りやすくなっているのがわかるかと思います。

## `ref_view` (All View)

`ref_view`は他の`range`を参照するだけの`view`です。

```cpp
int main() {
  std::vector vec = {1, 3, 5, 7, 9, 11};

  for (int n : ref_view(vec)) {
    std::cout << n; // 1357911
  }
}
```

右辺値でさえなければ任意の`range`オブジェクトを受けることができて、その`range`を単に参照するだけの`view`となります。

ぱっと見ると何に使うのか不明ですが、これは`std::vector`などのコピーが気軽にできない`range`の取り回しを改善するために利用できます。そのような`range`の軽量な参照ラッパとなることで、コピーされるかを気にしなくてよくなるなど可搬性が向上します。役割としては、通常のオブジェクトの参照に対する`std::reference_wrapper`に対応しています。例えば、`std::async`の追加の引数として渡すときなどのように*decay copy*されてしまう所に渡す場合に活用できるでしょう。

```cpp

int main() {
  std::vector<int> vec = {1, 2, 3};

  // vectorがコピーされる
  auto f1 = std::async(std::launch::async, [](auto&&) {}, vec); 
  
  // ref_viewがコピーされる
  auto f2 = std::async(std::launch::async, [](auto&&) {}, ref_view{vec});  
}
```

また、ほかの`view`、特にRangeアダプタの実装にも利用されます。`view`を作ろうとするとムーブ構築/代入が求められますが、`range`オブジェクトへの参照を直接持つと代入演算子が定義できなくなり、ポインタを利用すると`nullptr`を気にしなければなりません。`view`コンセプトの定義を思い出すと、すべての`view`はムーブ構築/代入が可能であり、`ref_view`もまたそれに従います。そのため、`ref_view`（正確には`views::all`）を用いて`range`を受け取り保持するようにすることでその辺りの考慮の必要性がなくなり、`view`の実装を幾分か楽にすることができます。

`ref_view`の定義は次のようになっています。

```cpp
namespace std::ranges {
  template<range R>
    requires is_object_v<R>
  class ref_view : public view_interface<ref_view<R>> {
  private:
    R* r_;  // 説明専用メンバ
  public:

    template<different-from<ref_view> T>
      requires /* 右辺値を弾く制約 */
    constexpr ref_view(T&& t);

    constexpr R& base() const { return *r_; }

    constexpr iterator_t<R> begin() const { return ranges::begin(*r_); }
    constexpr sentinel_t<R> end() const { return ranges::end(*r_); }

    constexpr bool empty() const
      requires requires { ranges::empty(*r_); }
    { return ranges::empty(*r_); }

    constexpr auto size() const requires sized_range<R>
    { return ranges::size(*r_); }

    constexpr auto data() const requires contiguous_range<R>
    { return ranges::data(*r_); }
  };

  template<class R>
    ref_view(R&) -> ref_view<R>;
}
```

入力の型`R`は`range`オブジェクトであればよく、コンストラクタの制約によって`ref_view`自身と右辺値の`range`オブジェクトを弾きます。そして、`ref_view`自体は対象の`range`オブジェクトへのポインタを保持し、その`range`のイテレータをそのまま利用します。`ref_view`のコンストラクタは他の`range`を受けるもの1つだけしかないので、`ref_view`が空（上記例では`r_`メンバ変数が`nullptr`）になることはありません。

### `ref_view<R>`の諸特性

- `reference` : `range_reference_t<R>`
- `range`カテゴリ : `R`のカテゴリに従う
- `common_range` : `R`が`common_range`の場合
- `sized_range` : `R`が`sized_range`の場合
- `const-iterable` : 〇
- `borrowed_range` : 〇

`ref_view`は元の`range`の極々薄いラッパでしかないので、元の`range`の性質をほとんどそのまま受け継ぎます。`ref_view`そのものは常に`const-iterable`ですが、それは参照先の`range`を`const`でイテレートするというわけではありません。それは先ほどの定義例にも見ることができます（メンバをポインタで持つことで`const`性伝播が妨げられている）。他の多くのRangeアダプタの`view`がこの`ref_view`を使用しているため、同様の問題がそれらにも波及しています。

### `views::all`

`ref_view`に対応する操作であるRangeアダプタオブジェクトが`views::all`です。

```cpp
int main() {
  std::vector vec = {1, 3, 5, 7, 9, 11};

  for (int n : views::all(vec)) {
    std::cout << n; // 1357911
  }

  std::cout << '\n';

  // パイプラインスタイル
  for (int n : vec | views::all) {
    std::cout << n; // 1357911
  }
}
```

`views::all`はカスタマイゼーションポイントオブジェクトであり、引数の式`r`によって`views::all(r)`のように呼び出されたとき、その効果は次のようになります

1. `decay_t<decltype(r)>`が`view`コンセプトを満たす（`r`が`view`である）ならば、`decay-copy(r)`
2. `ref_view{r}`が構築可能ならば、`ref_view{r}`
3. それ以外の場合、`subrange{r}`

`views::all`は`ref_view`だけを生成するわけではないですが、結果の型を区別しなければ実質的に`ref_view`相当の`view`を得ることができます。このため、`views::all`による`view`を*All View*と呼びます。

特に注記してはいませんが、Rangeアダプタオブジェクトの第一引数に来る`range`オブジェクトは`viewable_range`でなくてはなりません。したがって、第一引数に右辺値の`range`（`view`や`borrowed_range`ではなく）を渡すとコンパイルエラーとなります。以降も特に注記しませんが、これはすべてのRangeアダプタオブジェクトに共通することです。

先ほど少し触れましたが、この`views::all`は他のRangeアダプタの実装に良く用いられ、その型を参照したいことがよくあります。そのため、`views::all_t`というエイリアステンプレートによって`views::all`の戻り値型を簡単に取得することができるようになっています。

```cpp
namespace std::ranges::views {
  template<viewable_range R>
    using all_t = decltype(all(declval<R>()));
}
```

これは例えば次のように利用されます。

```cpp
template<input_range V>
class my_view {
  V base_;

public:

  // コンストラクタ引数で暗黙変換する
  myview(V base)
    : base_(td::move(base))
  {}

  // ...
};

// 推論補助でall_tを使用する
template<typename R>
my_view(R&&) -> my_view<views::all_t<R>>;
```

`ref_view`にせよ`subrange`にせよその型さえ求めてしまえば暗黙変換が可能なので、推論補助で`views::all_t`を使用して*All View*の型を求めてそれを利用する事で、コンストラクタなどで`views::all`を呼び出す必要がなくなります。後で見て行くように、他の`view`もほぼ全てこの様に実装されています。

## `filter_view`

`filter_view`は受け取った述語に従って元の範囲の要素を選別したシーケンスを生成する`view`です。

```cpp
int main() {
  // 奇数をフィルターする（偶数のみ取り出す）
  filter_view fv{views::iota(1, 10), [](int n) { return n % 2 == 0; }};
  
  for (int n : fv) {
    std::cout << n; // 2468
  }
}
```

元の範囲の各要素に対して渡した述語オブジェクトが`true`を返す要素だけを残します。

```cpp
namespace std::ranges {
  // filter_viewの定義の例    
  template<input_range V, indirect_unary_predicate<iterator_t<V>> Pred>
    requires view<V> && is_object_v<Pred>
  class filter_view : public view_interface<filter_view<V, Pred>> {
    // ...
  };

  template<class R, class Pred>
    filter_view(R&&, Pred) -> filter_view<views::all_t<R>, Pred>;
}
```

`filter_view`の入力となる`range`は`input_range`でなければなりません。述語型`Pred`はオブジェクト型であるように制約されていますが、これは関数ポインタを排除するものではありません。`std::indirect_unary_predicate`は`std::indirect_binary_predicate`の単項述語版です。対応する実装例を思い出すと、`filter_view`がどのように実装されているかが分かるかもしれません。

### 遅延評価

`filter_view`によるシーケンス生成は遅延評価されます。構築時に最初の要素だけが計算され、残りの要素はイテレータのインクリメントのタイミングで計算されます。

\clearpage

```cpp
filter_view fv{views::iota(1, 10), [](int n) { return n % 2 == 0; }};

auto it = ranges::begin(fv);
// イテレータ取得時に条件を満たす最初の要素が探索され、1番目の要素が計算される
// そしてその探索結果はキャッシュされる

++it;
// 次に条件を満たす要素が探索され、2番目の要素が計算される

int n = *it; // 2番目の要素（4）が得られる
```

この例では`iota_view`もまた遅延評価されているので、一連のシーケンス全体が遅延評価によって生成されています。

`fv.begin()`の呼び出し時に元の範囲の先頭から条件を満たす最初の要素を探索してその位置のイテレータを返すのですが、ただし毎回それをすると`range`コンセプトの意味論要件である「`ranges::begin()`の呼び出しが償却定数」というのを満たすことができなくなるため、最初に求めたイテレータは`filter_view`内部にキャッシュされ、`fv.begin()`の2回目以降の呼び出しではキャッシュされたイテレータを返します。

### `filter_view<R, P>`の諸特性

- `reference` : `range_reference_t<R>`
- `range`カテゴリ
    - `R`が`bidirectional_range` : `bidirectional_range`
    - `R`が`forward_range` : `forward_range`
    - それ以外 : `input_range`
- `common_range` : `R`が`common_range`の場合
- `sized_range` : ×
- `const-iterable` : ×
- `borrowed_range` : ×

`filter_view`が`borrowed_range`ではないのは、指定される述語`P`のオブジェクトを`filter_view`内部に保持しており、取得したイテレータはそれを参照しているためです。従って、親の`filter_view`オブジェクトの寿命が尽きるとそこから取得されたイテレータはすべて無効になります。`sized_range`ではないのは終端となる要素が条件`P`によって指定されているために定数時間でサイズを求められないためです。

`const-iterable`ではないのは`.begin()`の呼び出しでキャッシュしているためです。標準ライブラリの`const`メンバ関数はスレッドセーフであることを表明しており、`filter_view`が`const-iterable`であるためには`.begin()`の呼び出しに追加の同期が必要になりますが、それはオーバーヘッドになるため行われません。そのため、`filter_view`は`.begin()`を`const`にすることができないため、`const-iterable`ではありません。

### `views::filter`

`filter_view`に対応するRangeアダプタオブジェクトが`views::filter`です。

```cpp
int main() {
  auto even = [](int n) { return n % 2 == 0; };

  for (int n : views::filter(views::iota(1, 10), even)) {
    std::cout << n;
  }

  std::cout << '\n';

  // パイプラインスタイル
  for (int n : views::iota(1, 10) | views::filter(even)) {
    std::cout << n;
  }
}
```

`views::filter`はカスタマイゼーションポイントオブジェクトであり、引数の式`r, p`によって`views::filter(r, p)`のように呼び出されたとき、`filter_view(r, p)`を返します。

`views::filter`と`filter_view`は一対一対応していますが、パイプラインで使用可能であるなど`views::filter`を使った方が有利なシーンは多いでしょう。

## `transform_view`

`transform_view`は元となる範囲の要素それぞれに指定された変換を適用したシーケンスを生成する`view`です。

```cpp
int main() {
  // 各要素を2倍する
  transform_view tv{views::iota(1, 5), [](int n) { return n * 2; }};
  
  for (int n : tv) {
    std::cout << n; // 2468
  }
}
```

型`T`のシーケンスと`T -> U`へ変換する関数を受けて、`U`のシーケンスを返すものです。この例では結果も`int`ですが、当然型を変換することもできます。このような操作は他の所だと`map`とか`select`と呼ばれていますが、C++は原初のSTLのころからこの操作の事を一貫して`transform`と呼んでいるため、その流れを受けてここでも`transform`となっています。

```cpp
namespace std::ranges {
  // transform_viewの定義例
  template<input_range V, copy_constructible F>
    requires view<V> && is_object_v<F> &&
             regular_invocable<F&, range_reference_t<V>> &&
             can-reference<invoke_result_t<F&, range_reference_t<V>>>
  class transform_view : public view_interface<transform_view<V, F>> {
    // ...
  };

  template<class R, class F>
    transform_view(R&&, F) -> transform_view<views::all_t<R>, F>;
}
```

入力となる`range`及び変換関数型`F`に求められる基本的な事は`filter_view`と同様ですが、`F`には`copy_constructible`だけが要求されています（`filter_view`の場合は`indirect_unary_predicate`を通して要求されていました）。`regular_invocable`コンセプトは等しさを保持しながらの関数呼び出しが可能であることを定義するコンセプトで、`can-reference`は型が`void`ではないことをチェックする説明専用のコンセプトです。すなわち、`F`は`V`の要素によって呼び出し可能であり、その呼び出しは`F`の内外の状態に依存したり副作用を持ったりしてはならず、`F`の戻り値型は`void`であってはならないという事です。

`regular_invocable`コンセプトは`std::invoke`を用いて定義されており、`transform_view`による変換も`std::invoke`を用いて行われます。つまり、単なる関数オブジェクトだけでなくメンバポインタを渡したりすることもできるわけです。

### 遅延評価

`transform_view`もまた、遅延評価によってシーケンスを生成します。イテレータの`opreator*`による間接参照のタイミングで要素の読み出しと変換の適用が行われます。

\clearpage

```cpp
std::ranges::transform_view tv{std::views::iota(1, 5), [](int n) { return n * 2; }};
// transform_view構築時にはまだ計算されない

auto it = std::ranges::begin(tv);
// イテレータ取得時にも計算されない

++it;
// インクリメント時にも計算されない

// 間接参照時に要素が読みだされ、変換が適用される
int n1 = *it; // n1 == 2
int n2 = *it; // n2 == 2、再計算している

++it;

int n3 = *it; // n3 == 4
```

少し注意点なのですが、この性質上`opreator*`による間接参照のタイミングで毎回変換が行われます。一度計算した値をキャッシュしたりはしません。そのため、`transform_view`による範囲を複数回イテレートする際は変換処理の重さに気を配る必要があるかもしれません。

### `transform_view<R, F>`の諸特性

- `reference` : `invoke_result_t<F&, range_reference_t<R>>`
- `range`カテゴリ
    - `R`が`random_access_range` : `random_access_range`
    - `R`が`bidirectional_range` : `bidirectional_range`
    - `R`が`forward_range` : `forward_range`
    - それ以外 : `input_range`
- `common_range` : `R`が`common_range`の場合
- `sized_range` : `R`が`sized_range`の場合
- `const-iterable` : `R`が`const-iterable`であり、`const F`が呼び出し可能である場合
- `borrowed_range` : ×

`reference`はややこしいですが、`R`の要素型`T`について`F`が`F : T -> U`のような変換を行う場合の`U`になり、これは`remove_cvref`等されない`F`の戻り値型そのままです。また、`range`カテゴリが`R`と異なるのは`R`が`contiguous_range`であるときだけです。

常に`borrowed_range`とならないのは、変換関数`F`のオブジェクトを`transform_view`内に保存しておかなければならないためです。`transform_view`イテレータの間接参照時には、親の`transform_view`オブジェクト内に保存された`F`のオブジェクトを使用しています。

### `views::transform`

`transform_view`に対応するRangeアダプタオブジェクトが`views::transform`です。

```cpp
int main() {
  auto time_2 = [](int n) { return n * 2; };

  for (int n : views::transform(views::iota(1, 5), time_2)) {
    std::cout << n; // 2468
  }

  std::cout << '\n';

  // パイプラインスタイル
  for (int n : views::iota(1, 5) | views::transform(time_2)) {
    std::cout << n; // 2468
  }
}
```

`views::transform`はカスタマイゼーションポイントオブジェクトであり、引数の式`r, f`によって`views::transform(r, f)`のように呼び出されたとき、`transform_view(r, f)`を返します。

## `take_view`

`take_view`は元となるシーケンスの先頭から指定された数だけ要素を取り出したシーケンスを生成する`view`です。

```cpp
int main() {
  // 先頭から5つの要素だけを取り出す
  take_view tv{views::iota(1), 5};
  
  for (int n : tv) {
    std::cout << n; // 12345
  }
}
```

`iota_view`の生成する無限列のように無限に続くシーケンスから決められた数だけを取り出したり、Rangeアルゴリズムにおいて他の操作を適用したシーケンスから（先頭に集められた）最終的な結果を取り出す時などに活用できます。

### オーバーランの防止

意地悪な人はきっと、`take_view`に元となる範囲の長さよりも大きい数を指定したらどうなるの？と思うことでしょう。残念ながら？これは対策されています。

`take_view<R>`のイテレータは元となる範囲`R`の種別によって3つのパターンに分岐します。

1. `R`が`sized_range`であるならば、次のいずれか
   1.  `R`が`random_access_range`なら、`R`の先頭イテレータをそのまま利用
   2. それ以外の場合、`std::counted_iterator`に`R`の先頭イテレータと与えられた長さと`R`の長さの短い方を渡して構築
2. それ以外の場合、`std::counted_iterator`に`R`の先頭イテレータと与えられた長さを渡して構築

`std::counted_iterator`はC++20から追加された*iteretor adaptor*で、与えられたイテレータをラップして指定された長さだけイテレート可能なものに変換します。

これによって、距離が事前に求まる場合はその距離を超えてイテレートされることはありません。そして、`sized_range`ではない`range`に対しても`take_view`の提供する番兵型によって確実にオーバーランしないようにチェックされています。

```cpp
int main() {
  using namespace std::string_view_literals;
  
  // 元の文字列の長さを超えた長さを指定する（上記1.1のケース）
  take_view tv1{"str"sv, 10};
  
  int count = 0;
  
  // 安全、3回しかループしない
  for ([[maybe_unused]] char c : tv1) {
    ++count;
  }
  
  std::cout << "loop : " << count << '\n';  // loop : 3
  
  std::list li = {1, 2, 3, 4, 5};
  
  // 元のリストの長さを超えた長さを指定する（上記1.2のケース）
  take_view tv2{li, 10};
  count = 0;

  // 安全、5回しかループしない
  for ([[maybe_unused]] int n : tv2) {
    ++count;
  }
  
  std::cout << "loop : " << count << '\n';  // loop : 5

  std::forward_list fl = {1, 2, 3, 4, 5};
  
  // 元のリストの長さを超えた長さを指定する（上記2のケース）
  take_view tv3{fl, 10};
  count = 0;

  // 安全、5回しかループしない
  for ([[maybe_unused]] int n : tv3) {
    ++count;
  }
  
  std::cout << "loop : " << count << '\n';  // loop : 5
}
```

なお、`take_view`に負の値を渡すこともできますが、`R`が`random_access_range`の場合以外は未定義動作になります。使い所はあるかもしれませんが基本的には避けた方が良いでしょう。

### 遅延評価

`take_view`もまた、遅延評価によってシーケンスを生成します。ただ、`take_view`は元となる`range`の極薄いラッパーなので、ほとんどの操作はベースにあるイテレータの操作をそのまま呼び出すだけで、特別な事は行ないません。`take_view`が行なっている事はほぼその長さの管理だけです。それは主に`==`による終端チェック時に行われます。また、`std::counted_iterator`が使用される場合はそのためにインクリメントのタイミングで残りの距離の計算（単純なカウンタのデクリメントによる）が行われます。

\clearpage

```cpp
take_view tv{views::iota(1), 5};
// take_vieww構築時には何もしない

auto it = ranges::begin(tv);
// イテレータ取得時には元のシーケンスによって最適なイテレータを返す

++it;
// インクリメントはベースのイテレータをインクリメントする
// counted_iteratorが使用される場合、ここで残りの距離が計算される

int n1 = *it; // n1 == 2
// 間接参照時はベースのイテレータを間接参照するだけ

auto fin = ranges::end(tv);
// 番兵取得時には元のシーケンスによって最適な番兵を返す

it == fin;
// 終端チェック時に与えられた長さと元のシーケンスの長さ、現在の位置に基づいて終端チェックが行われる
```

### `take_view<R>`の諸特性

- `reference` : `range_reference_t<R>`
- `range`カテゴリ : `R`のカテゴリに従う
- `common_range` : `R`が`sized_range`かつ`random_access_range`の場合
- `sized_range` : `R`が`sized_range`の場合
- `const-iterable` : `R`が`const-iterable`の場合
- `borrowed_range` : `R`が`borrowed_range`の場合

`take_view`は元の範囲`R`のごく薄いラッパとなり、元の`range`の性質をほぼそのまま受け継ぎます。

### `views::take`

`take_view`に対応するRangeアダプタオブジェクトが`views::take`です。

```cpp
int main() {
  
  for (int n : views::take(views::iota(1), 5)) {
    std::cout << n;
  }

  // パイプラインスタイル
  for (int n : views::iota(1) | views::take(5)) {
    std::cout << n;
  }
}
```

`views::take`はカスタマイゼーションポイントオブジェクトであり、型`R, D`の引数の式`r, n`によって`views::take(r, n)`のように呼び出されたとき、その効果は次のようになります

1. `R`が`empty_view`の特殊化である場合、`((void) n, decay-copy(r))`
2. `R`が`random_access_range`かつ`sized_range`であり以下のいずれかに該当する場合、`R(ranges::begin(r), ranges::begin(r) + min<D>(ranges::size(r), n))`
      - `R`が`std::span`の特殊化であり、`extent`が`std::dynamic_extent`である
      - `R`が`std::basic_string_view`の特殊化である
      - `R`が`iota_view`の特殊化である
      - `R`が`subrange`の特殊化である
3. それ以外の場合、`take_view(r, n)`

その条件は複雑ですが、`random_access_range`かつ`sized_range`である標準ライブラリのもの（`std::span, std::string_view`など）に対しては、与えられた長さと元の長さのより短い方の長さによって構築し直したその型のオブジェクトを返し、そうではない`r`に対しては`take_view`を返します。

厳密には`take_view`だけが得られる訳ではありませんが、結果の型を区別しなければ実質的に`take_view`と同等の`view`が得られます。

## `take_while_view`

`take_while_view`は元となるシーケンスから指定された条件を満たす連続要素によるシーケンスを生成する`view`です。

\clearpage

```cpp
int main() {
  // 先頭から7未満の要素だけを取り出す
  take_while_view tv{views::iota(1), [](int n) { return n < 7; }};
  
  for (int n : tv) {
    std::cout << n; // 123456
  }
}
```

元となる範囲の先頭（`t`）から条件を満たさない最初の要素（`e`）の一つ前までのシーケンス（`[t, e)`）を生成します。

```cpp
namespace std::ranges {

  // take_while_viewの定義例
  template<view V, class Pred>
    requires input_range<V> && is_object_v<Pred> &&
             indirect_unary_predicate<const Pred, iterator_t<V>>
  class take_while_view : public view_interface<take_while_view<V, Pred>> {
    // ...
  };

  template<class R, class Pred>
    take_while_view(R&&, Pred) -> take_while_view<views::all_t<R>, Pred>;
}
```

入力となる`range`及び述語型`P`に求められる基本的な事は`filter_view`と同様です。

### 遅延評価

`take_while_view`もまた、遅延評価によってシーケンスを生成します。ただ、`take_while_view`は元となる`range`の極々薄いラッパーなので、ほとんどの操作はベースにあるイテレータの操作をそのまま呼び出すだけです。`take_while_view`が行なっている事は範囲終端の管理だけで、`==`による終端チェックのタイミングで現在の要素が条件を満たすか否かをチェックします。また、同時に現在の位置が元のシーケンスの終端に到達しているかもチェックすることでオーバーランを防止します。

\clearpage

```cpp
take_while_view tv{views::iota(1), [](int n) { return n < 7; }};
// take_while_view構築時には何もしない

auto it = ranges::begin(tv);
// イテレータ取得時には元のイテレータをそのまま返す

++it;
// インクリメントは元のイテレータのインクリメント

int n1 = *it; // n1 == 2
// 間接参照も元のイテレータの間接参照

auto fin = ranges::end(tv);
// 受け取った述語オブジェクトを私て番兵を構築する

it == fin;
// 終端チェック時に現在のイテレータの指す要素が与えられた条件を満たしているかをチェック
// 同時に元のシーケンスの終端に到達しているかもチェックする
```

この特性上、`==`による終端チェック時には毎回要素の参照と条件のチェック及び終端チェックが行われます。条件判定に重い処理を渡してしまうと思わぬオーバーヘッドになるかもしれません。

### `take_while_view<R, P>`の諸特性

- `reference` : `range_reference_t<R>`
- `range`カテゴリ : `R`のカテゴリに従う
- `common_range` : ×
- `sized_range` :  ×
- `const-iterable` : `R`が`const-iterable`かつ`const P`と`R`のイテレータが  
    `indirect_unary_predicate`である場合
- `borrowed_range` : ×

`take_while_view`は元の範囲`R`の薄いラッパではあるのですが、範囲の終端が条件`P`で指定されているために`sized`ではなく、番兵との`==`の比較時にそれをチェックすることから番兵型を自前で提供せざるを得ないため`common`でもありません。`borrowed`でもないのは、`filter_view`同様に`P`のオブジェクトを内部に保持しているためです。

### `views::take_while`

`take_while_view`に対応するRangeアダプタオブジェクトが`views::take_while`です。

```cpp
int main() {
  auto less_than_7 = [](int n) { return n < 7; };

  for (int n : views::take_while(views::iota(1), less_than_7)) {
    std::cout << n; // 123456
  }
  
  std::cout << '\n';

  // パイプラインスタイル
  for (int n : views::iota(1) | views::take_while(less_than_7)) {
    std::cout << n; // 123456
  }
}
```

`views::take_while`はカスタマイゼーションポイントオブジェクトであり、引数の式`r, p`によって`views::take_while(r, p)`のように呼び出されたとき、`take_while_view(r, p)`を返します。

## `drop_view`

`drop_view`は元となるシーケンスの先頭から指定された数の要素を取り除いたシーケンスを生成する`view`です。

```cpp
int main() {
  // 先頭から5つの要素を取り除く
  drop_view dv{views::iota(1, 10), 5};
  
  for (int n : dv) {
    std::cout << n; // 6789
  }
}
```

ちょうど`take_view`と逆の働きをするもので、先頭から指定個数の要素をスキップしたところから開始するシーケンスを生成します。

### オーバーラン防止と遅延評価

`take_view`がそうであるように`drop_view`もまた元となるシーケンスの長さを超えることの無いようになっています。

`drop_view`によるシーケンスは遅延評価によって生成され、そのイテレータの取得時（`begin()`の呼び出し時）に元となるシーケンスの先頭イテレータを指定した分進めて返します。その際、元のシーケンス上で終端チェックを行いながら進めることでオーバーランしないようになっています。これはC++20から追加された`ranges::next(it, end, n)`を使用して行われ、元のシーケンスの長さよりも大きい値を指定すると空の範囲が得られます。

```cpp
drop_view dv{views::iota(1, 10), 5};
// drop_view構築時にはまだ何もしない

auto it = ranges::begin(tv);
// イテレータ取得時にスキップ処理が行われる
// 元のシーケンスの先頭イテレータを指定した長さ進めるだけ
// その際終端チェックを同時に行う

++it;
*it;
// その他の操作は元のシーケンスのイテレータそのまま
```

`drop_view<R>`の`begin()`の呼び出しは`R`が`forward_range`であるとき、`begin()`の処理結果はキャッシュされます。これによって`drop_view`の`begin()`の計算量は償却定数となります。ただし、`R`に対して`const R`が`random_access_range`かつ`sized_range`である時、キャッシュは使用されず`begin()`メンバ関数は`const`修飾され、`drop_view`は`const`状態でもイテレート可能となります。なぜなら、`R`が`random_access_range`かつ`sized_range`である場合、その範囲の現在の長さの取得とイテレータの適切な進行操作の両方を`O(1)`で行う事ができるため、キャッシュを使用しなくても`range`コンセプトの要件を満たす事ができるためです。なお、入力が`input_range`であるときは、`begin()`の呼び出しは最初の一度だけ有効です（それ以降は`range`コンセプトのモデルとならなくなります）。

### `drop_view<R>`の諸特性

- `reference` : `range_reference_t<R>`
- `range`カテゴリ : `R`のカテゴリに従う
- `common_range` : `R`が`common_range`の場合
- `sized_range` :  `R`が`sized_range`の場合
- `const-iterable` : `R`が`const-iterable`であり`random_access_range`かつ`sized_range`の場合
- `borrowed_range` : `R`が`borrowed_range`の
`const-iterable`だけは少し複雑になっていますが、`drop_view`はイテレータ取得時に元のイテレータを進めて返すだけなので、元の`range`の性質をほぼそのまま受け継ぎます。

### `views::drop`

`drop_view`に対応するRangeアダプタオブジェクトが`views::drop`です。
  
```cpp
int main() {
  for (int n : views::drop(views::iota(1, 10), 5)) {
    std::cout << n;
  }
  
  std::cout << '\n';

  // パイプラインスタイル
  for (int n : views::iota(1, 10) | views::drop(5)) {
    std::cout << n;
  }
}
```

`views::drop`はカスタマイゼーションポイントオブジェクトであり、型`R, D`の引数の式`r, n`によって`views::drop(r, n)`のように呼び出されたとき、その効果は次のようになります

1. `R`が`empty_view`の特殊化である場合、`((void) n, decay-copy(r))`
2. `R`が`random_access_range`かつ`sized_range`であり以下のいずれかに該当する場合、`R(ranges::begin(r) + min<D>(ranges::size(r), n), ranges::end(r))`
      - `R`が`std::span`の特殊化であり、`extent`が`std::dynamic_extent`である
      - `R`が`std::basic_string_view`の特殊化である
      - `R`が`iota_view`の特殊化である
      - `R`が`subrange`の特殊化である
3. それ以外の場合、`drop_view(r, n)`

条件は複雑ですが、`random_access_range`かつ`sized_range`である標準ライブラリのもの（`std::span, std::string_view`など）に対しては、与えられた長さと元の長さのより短い方の位置から開始するように構築し直したその型のオブジェクトを返し、そうではない`r`に対しては`drop_view`を返します。`views::take`と同様ですね。

厳密には`drop_view`だけを返すわけではありませんが、結果の型を区別しなければ実質的に`drop_view`と同等の`view`が得られます。

## `drop_while_view`

`drop_while_view`は元となるシーケンスの先頭から条件を満たす連続した要素を取り除いたシーケンスを生成する`view`です。

```cpp
int main() {
  // 先頭のホワイトスペースを取り除く
  ranges::drop_while_view dv{"     drop while view", [](char c) { return c == ' '; }};
  
  for (char c : dv) {
    std::cout << c; // drop while view
  }
}
```

`take_view`に対する`take_while_view`の対応と同様に、`drop_view`の条件指定版です。そのため、その性質は`take_while_view`によく似ています。

### オーバーラン防止と遅延評価

`drop_while_view`も`drop_view`と同様の方法によって遅延評価とオーバーランの防止を行なっています。すなわち、`drop_while_view`はそのイテレータの取得時（`begin()`の呼び出し時）に元となるシーケンスの先頭イテレータを条件を満たさない最初の要素まで進めて返します。その際、元のシーケンス上で終端チェックを行いながら進めることでオーバーランしないようになっています。これはC++20から追加された`ranges::find_if_not(base_view, pred)`を使用して行われます。

```cpp
std::ranges::drop_while_view dv{"     drop while view", [](char c) { return c == ' '; }};
// drop_view構築時にはまだ何もしない

auto it = std::ranges::begin(dv);
// イテレータ取得時にスキップ処理が行われる
// 元のシーケンスの先頭イテレータを条件を満たす間進めるだけ
// その際終端チェックを同時に行う

++it;
*it;
// その他の操作は元のシーケンスのイテレータそのまま
```

`drop_while_view`の`begin()`の呼び出しも、元のシーケンスが`forward_range`であるときキャッシュされることが規定されています。これによって`drop_while_view`の`begin()`も計算量は償却定数となります。しかし、キャッシュを使用するために`const`修飾できず、`drop_view`とは異なり条件をチェックする必要があることから元の`range`が何であってもそれを回避することができません。そのため、`drop_while_view`は常に`const-iterable`ではありません。`input_range`に対しては`drop_view`同様に`begin()`の呼び出しは最初の一度だけ有効です。

### `drop_while_view<R, P>`の諸特性

- `reference` : `range_reference_t<R>`
- `range`カテゴリ : `R`のカテゴリに従う
- `common_range` : `R`が`common_range`の場合
- `sized_range` :  ×
- `const-iterable` : ×
- `borrowed_range` : `R`が`borrowed_range`の場合

`drop_while_view`は元となる`range`のイテレータを完全に流用しているため、その性質の多くの部分をもとの`range`から継承します。ただし、前述の理由（キャッシュを使用し終端が条件で指定されること）から`sized`と`const-iterable`にだけはどうしてもなりません。

### `views::drop_while`

`drop_while_view`に対応するRangeアダプタオブジェクトが`views::drop_while`です。
  
```cpp
int main() {

  auto skip_ws = [](char c) { return c == ' '; };

  for (char c : views::drop_while("     drop while view", skip_ws)) {
    std::cout << c;
  }

  // パイプラインスタイル
  for (char c : "     drop while view" | views::drop_while(skip_ws)) {
    std::cout << c;
  }
}
```

`views::drop_while`はカスタマイゼーションポイントオブジェクトであり、引数の式`r, p`によって`views::drop_while(r, p)`のように呼び出されたとき、`drop_while_view(r, p)`を返します。

## `join_view`

`join_view`は`range`の`range`となっているシーケンスを平坦化したシーケンスを生成する`view`です。

```cpp
int main() {
  std::vector<std::vector<int>> vecvec = { {1, 2, 3}, {}, {}, {4}, {5, 6, 7, 8, 9}, {10, 11}, {} };

  join_view jv{vecvec};
  
  for (int n : jv) {
    std::cout << n; // 1234567891011
  }
}
```

すなわち、配列の配列を1つの配列に直列化するような事を行うものです。

この例では`std::vector`の`std::vector`を利用していますが、別に外側と内側の`range`が同じものである必要はありません。`std::list`の`std::vector`とか、`std::deque`の生配列など、`range`の`range`になっていれば`join_view`は平坦化してくれます。

```cpp
int main() {
  std::vector<std::list<int>> veclist = { {1, 2, 3}, {}, {}, {4}, {5, 6, 7, 8, 9}, {10, 11}, {} };

  join_view jv1{veclist};
  
  for (int n : jv1) {
    std::cout << n; // 1234567891011
  }

  std::deque<int> arrdeq[] = { {1, 2, 3}, {}, {}, {4}, {5, 6, 7, 8, 9}, {10, 11}, {} };

  join_view jv2{arrdeq};

  for (int n : jv2) {
    std::cout << n; // 1234567891011
  }
}
```

この様な操作は他の所では`flat`とか`flatten`とも呼ばれています。モナドの文脈だと`join`と呼ばれることがあり、`join_view`という名前はそちらを採用しているようです。

```cpp
namespace std::ranges {

  // join_viewの定義例
  template<input_range V>
    requires view<V> && input_range<range_reference_t<V>>
  class join_view : public view_interface<join_view<V>> {
    // ...
  };

  template<class R>
    explicit join_view(R&&) -> join_view<views::all_t<R>>;
}
```

`join_view`の入力となる`range`は`input_range`であり、その要素型が`input_range`である必要があります。まあ当然ですね。

### 遅延評価

`join_view`によるシーケンスもまた遅延評価によって生成されます。`join_view`の仕事の殆どは元となるシーケンスの内側の`range`同士を接続することにあり、イテレータのインクリメントのタイミングでそれを行います。

`join_view`は元となるシーケンスの内側`range`のイテレータを利用して1つの内側`range`のイテレートを行います。そのイテレータが終端に達した時（1つの内側`range`の終端に達した時）、外側`range`のイテレータを1つ進めてそこから次の内側`range`のイテレータを取得します。ただ、そのままだと内側`range`が空の場合に上手くいかないため、すぐに内側`range`の終端チェックを行い空でない内側`range`が見つかるまで外側`range`をイテレートします。そうして外側イテレータが終端に達したとき、そこが`join_view`の終端となります。

\clearpage

```cpp
std::vector<std::vector<int>> vecvec = { {1, 2, 3}, {}, {}, {4}, {5, 6, 7, 8, 9}, {10, 11}, {} };

join_view jv{vecvec};
auto it = ranges::begin(jv);
// 構築とイテレータ取得時には何もしない

++it;
// インクリメント時に内側イテレータの接続を行う
// 内側イテレータが終端に到達していれば、外側イテレータを進めてそこから内側イテレータを再取得する
// 同時に内側イテレータの終端チェックを行い、空の内側rangeをスキップする

--it;
// デクリメント時はその逆を行う

int n = *it;
// 間接参照は元のシーケンスのイテレータそのまま
```

1つの内側`range`をイテレートしている間は内側イテレータの終端チェックのみが行われますが、その終端に到達した時（2つの内側`range`を接続する時）は外側`range`のイテレータのインクリメントと終端チェックが入るため、少し処理が重くなります。

`join_view`は少し複雑ですが、入力となるシーケンスについて内側`range`と外側`range`を意識すると、`join_view`の仕事は内側`range`同士を接続しているだけであることが見えやすくなるかもしれません。次の図は、`join_view`が`range`の`range`を接続する様子を描いたものです。

![](img/join_view.png)

### `join_view<R>`の諸特性

- `reference` : `range_reference_t<range_reference_t<R>>`
- `range`カテゴリ : 
    - `range_reference_t<R>`が参照型であり（`R`の要素が左辺値であり）
      - 外側（`R`）と内側（`range_reference_t<R>`）`range`が共に`bidirectional_range`である場合 : `bidirectional_range`
      - 外側（`R`）と内側（`range_reference_t<R>`）`range`が共に`forward_range`である場合 : `forward_range`
    - それ以外 : `input_range`
- `common_range` : 次の条件を全て満たす場合
    - `R`が`forward_range`かつ`common_range`
    - 内側`range`（`range_reference_t<R>`）が`forward_range`かつ`common_range`
    - `range_reference_t<R>`が参照型
- `sized_range` :  ×
- `const-iterable` : `R`が`const-iterable`であり`range_reference_t<R>`が参照型である場合
- `borrowed_range` : ×

頻出する「`range_reference_t<R>`が参照型」と言う条件は、外側`range`の要素である内側の`range`オブジェクトが右辺値で帰ってこない事を指定しています。そのような`R`であっても`join`可能ですが、その場合は`join_view`内で内側`range`をキャッシュ（右辺値->左辺値へ変換）するため、性質は大きく制限されます。

### `join_view`とキャッシュ

`join_view<R>`は、内側の`range`（`range_reference_t<R>`）が左辺値の場合はさして問題ないのですが、右辺値として得られる場合はそれをどこかに保持しておかないとダングリングとなってしまいます。そのため、`join_view`は内側`range`が右辺値の場合にそれを内部にキャッシュするようになります。1つの内側`range`をイテレートしている間それを保持しておく事でダングリング化することを防いでおり、そのために追加の保存領域を必要とします。

ただし、キャッシュされるのは常に1つだけであるため（`view`コンセプトの要請による）、外側`range`のイテレータが進行すると1つ前の内側`range`オブジェクトは捨てられてしまいます。そのため、内側の`range`が右辺値で得られる場合は外側および内側の性質がなんであっても`join_view`は`input_range`となり、`begin()`の呼び出しは最初の一度だけ有効となります。また、キャッシュ機構を使用するために`const-iterable`ではなくなります。

```cpp
int main() {
  // n -> [0, n)の数列へ変換する処理
  auto to_rrng = [](int n) -> std::vector<int> {
    auto iota = views::iota(0, n);
    return {iota.begin(), iota.end()};
  };

  // 右辺値rangeを返すview
  // [ [0, 1, 2], [0 ... 3], [0 ... 4], [0 ... 5] ]
  auto seq = views::iota(3, 7) | views::transform(to_rrng);

  // 右辺値rangeのviewの平坦化
  for (int n : seq | join_view{seq}) {
    std::cout << n; // 012012301234012345
  }
}
```

この場合でも遅延評価などは行われており、`++`のタイミングで内側`range`をキャッシュする以外にやることは変化していません。

このキャッシュは伝播しません。すなわち、キャッシュを持つ`join_view`をムーブしたりコピーしてもキャッシュの内容はコピー/ムーブ先に伝わらず、コピー/ムーブ元のキャッシュも消去されます。これは、`join_view`のコピー/ムーブが元となる`range`の生成する`range`（要素）に依存してしまう事を回避するための振る舞いで、それによって`join_view`をムーブしたときにキャッシュ元の`range`の所有権だけが別オブジェクトに移動することでキャッシュの内容がある種のダングリング状態に陥ることを回避しようとするものです。

なお、内側`range`が左辺値の場合はこれらの様なオーバーヘッドはありません。すなわち、内側`range`の性質によってクラスレイアウトごと性質が変化します。

### `views::join`

`join_view`に対応するRangeアダプタオブジェクトが`views::join`です。

```cpp
int main() {
  std::vector<std::vector<int>> vecvec = { {1, 2, 3}, {}, {}, {4}, {5, 6, 7, 8, 9}, {10, 11}, {} };

  for (int n : views::join(vecvec)) {
    std::cout << n; // 1234567891011
  }

  std::cout << '\n';

  // パイプラインスタイル
  for (int n : vecvec | views::join) {
    std::cout << n; // 1234567891011
  }
}
```

`views::join`はカスタマイゼーションポイントオブジェクトであり、引数の式`r`によって`views::join(r)`のように呼び出されたとき、`join_view<views::all_t<decltype((r))>>{r}`を返します。

## `lazy_split_view`

`lazy_split_view`は元のシーケンスから指定されたデリミタを区切りとして切り出した部分範囲のシーケンスを生成する`view`です。

```cpp
int main() {
  // ホワイトスペースをデリミタとして文字列を切り出す
  lazy_split_view sv{"lazy_split_view takes a view and a delimiter", ' '};
  
  // split_viewは切り出した文字列のrangeとなる
  for (auto inner_range : sv) {
    // inner_rangeは分割された文字列1つを表すview
    for (char c : inner_range) {
      std::cout << c;
    }
    std::cout << '\n';
  }
  /*
    lazy_split_view
    takes
    a
    view
    and
    a
    delimiter
  */
}
```

一見すると文字列分割に使いたくなるかもしれません。

`lazy_split_view`は切り出した部分範囲それぞれを要素とするシーケンス（外側`range`）となります。そのため、その要素を参照すると分割された文字列1つを表す別のシーケンス（内側`range`）が得られます。そこから、内側`range`の要素を参照することで切り出した文字列を1文字づつ取り出すことができます。

なお、この例の場合の`inner_range`（内側`range`）は`std::string_view`あるいは類するものではなく、単に`forward_range`である何かです。そのため、この`inner_range`から`string_view`等文字列型に変換するのは少し手間のかかる作業となります。

ややこしいですが、`join_view`のところで外側/内側`range`と言っていたのは`join_view`への入力`range`に対しての話で、ここでの内側/外側は`lazy_split_view`からの出力`range`の話です。

`lazy_split_view`の結果はとても複雑で一見非自明です。たとえば`"abc,12,cdef,3"`という文字列を`,`で分割するとき、`lazy_split_view`の様子は次のようになっています。

![](img/split_view.png)

実際の実装は元のシーケンスのイテレータを可能な限り使いまわすのでもう少し複雑になりますが、概ねこの図のような関係性の`range`の`range`が生成されます。

```cpp
namespace std::ranges {
  // lazy_split_viewの定義例
  template<input_range V, forward_range Pattern>
    requires view<V> && view<Pattern> &&
             indirectly_comparable<iterator_t<V>, iterator_t<Pattern>, ranges::equal_to> &&
             (forward_range<V> || tiny-range<Pattern>)
  class lazy_split_view : public view_interface<lazy_split_view<V, Pattern>> {
    // ...
  };

  template<class R, class P>
    lazy_split_view(R&&, P&&) -> lazy_split_view<views::all_t<R>, views::all_t<P>>;

  template<input_range R>
    lazy_split_view(R&&, range_value_t<R>)
      -> lazy_split_view<views::all_t<R>, single_view<range_value_t<R>>>;
}
```

やたらとややこしいですが、入力となる`range`は`input_range`である必要があり、デリミタ（`Pattern`）は`forward_range`でなくてはなりません。入力`range`のイテレータとデリミタのイテレータは`ranges::equal_to`によって`indirectly_comparable`である必要があります。これは、それぞれのイテレータを間接参照して得られた要素が`ranges::equal_to`で比較可能であることを要求しています。`requires`節3行目の制約は、入力`range`が`forward_range`でないならばデリミタは`tiny-range`でなければならないと言っています。`tiny-range<R>`は`R`が`sized_range`であり、その`size()`が静的メンバ関数かつ`constexpr`呼び出し可能で、そのサイズが1以下であるような`range`を指しています。これは殆ど実質的に`single_view`を指定しており、3行目の制約式は`input_range`を分割する時はデリミタの長さが1以下であることを規定しています。

用意されている推論補助の2つ目のものは、デリミタに非`range`の単一要素を指定した場合に選択され、それを`single_view`で包んで`range`化して`lazy_split_view`のインスタンス化を行います。`requires`節3行目の制約式は、この推論補助を利用して`lazy_split_view`がインスタンス化されることを前提としています。

### 遅延評価

`lazy_split_view`もまた遅延評価によって分割処理と`view`の生成を行います。

`lazy_split_view`の主たる仕事は元のシーケンス上でデリミタと一致する部分を見つけだし、それを区切りに内側`range`を生成する事にあります。それは外側`range`のイテレータのインクリメントと内側`range`のイテレータの終端チェックのタイミングで行われます。

外側`range`のイテレータ（外側イテレータ）は元のシーケンスのイテレータ（元イテレータ）を持っており、インクリメント時にはそのイテレータから出発して最初に出てくるデリミタ部分を探し出し、そのデリミタ部分の終端位置に元イテレータを更新します。インクリメントと共にこれを行う事で、デリミタ探索範囲を限定しています。

内側`range`における終端検出（すなわち文字列分割そのもの）は、内側`range`のイテレータ（内側イテレータ）の終端チェック（`==`）のタイミングで行われます。内側イテレータは自身を生成した外側イテレータとその親の`lazy_split_view`を参照して、外側イテレータの保持する元イテレータの終端チェック→デリミタが空かのチェック→自信の現在位置とデリミタの比較、の順で終端チェックを行います。　

```cpp
lazy_split_view sv{"lazy_split_view takes a view and a delimiter", ' '};
// 構築時点では何もしない

auto outer_it = std::ranges::begin(sv);
// 外側イテレータの取得、特に何もしない

++outer_it;
// 外側イテレータのインクリメント時は
// 元のシーケンス上で次に出現するデリミタ位置を探す
// 内部rangeは"lazy_split_view"->"takes"へ進む


auto inner_range = *outer_it;
// 内側rangeの取得、特に何もしない

auto inner_it = std::ranges::begin(inner_range);
// 内側イテレータの取得、特に何もしない

++inner_it;
// 内側イテレータのインクリメントは
// 元のシーケンスのイテレータのインクリメントとほぼ等価

char c = *inner_it; // a
// 元のシーケンスのイテレータのデリファレンスと等価

bool f = inner_it == std::ranges::end(inner_range); // false
// 内側rangeの終端チェック
// 元のシーケンスの終端と、デリミタが空かどうか、
// 現在の位置でデリミタが出現しているかどうか、を調べる
```

`lazy_split_view`の結果が複雑となる一因は、この遅延評価を行うことによるところがあります。

### `input_range`の分割

先ほど見た定義にも現れていて、少し驚くべきことかもしれませんが、`lazy_split_view`は`input_range`を分割することができます。

```cpp
int main() {
  // 標準入力には「4 0 123 0 1 6 700 0 8 3 0 9 7 0」と入力
  input_range auto in = istream_view<int>(std::cin);

  // 0をデリミタとして分割
  for (auto inner : lazy_split_view{in, 0}) {
    for (auto n : inner) {
      std::cout << n;
    }
    std::cout << '\n';
  }
  /*
    4
    123
    16700
    83
    97
  */
}
```

`input_range`では基本的にイテレータによるマルチパスな走査ができなたいめ、一見すると分割することは出来ないのではないかと思ってしまいます。しかし、デリミタの長さが最大でも1で固定されている事が分かっていて、かつ遅延評価を行う事によって、`input_range`の範囲でもシングルパスの走査によって分割を行う事ができます。その際、入力`range`のイテレータ（`input_iterator`）はコピーして取りまわせないために`lazy_split_view`の内部に保存され、`lazy_split_view`のイテレータはそれを参照して動作するようになります。すなわち、複数のイテレータによる走査時には互いに影響を及ぼし合い、`input_range`の分割時の外側`range`は常に`input_range`となります。

これは外側イテレータと内側イテレータが協調動作することで実現されています。外側イテレータの進行時には次に出てくるデリミタ要素が見つかるまで元イテレータを進めます。その際、まだ文字列を分割しません。内側`range`のイテレータの進行時もそのまま元イテレータを進め、分割しません。内側`range`の終端チェック時、現在の要素がデリミタと一致するかを調べ、一致する場合に終端であると判定され内側`range`は終了します。すなわち、内側`range`の終了によって仮想的に元`range`の分割を行っているわけです。

外側イテレータと内側イテレータは両方とも`lazy_split_view`の内部に保存されている元`range`のイテレータを使用して元`range`の走査を行っており、外側イテレータと内側イテレータが1つの元イテレータを共有しながら進行と終端チェックをそれぞれで行う事で、シングルパスでの分割という作業を実現しています。

```cpp
auto sv = lazy_split_view{istream_view<int>(std::cin), 0};

auto outer_it = ranges::begin(sv);
// 外側イテレータ取得時には特に何もしない

++outer_it;
// 外側イテレータの進行時
// 次に出現するデリミタ要素の次まで元rangeのイテレータを進める

auto outer_end = ranges::end(sv);

bool b = outer_it == outer_end;
// 外側rangeの終端チェック、以下の事をチェック
// 1. 元rangeの終端に到達しているか
// 2. 一番最後の要素がデリミタそのものであるか

auto inner = *outer_it;
// 内側rangeの取得、特に何もしない


auto inner_it = ranges::begin(inner);
// 内側イテレータ取得、特に何もしない

++inner_it;
// 内側イテレータの進行は、元rangeのイテレータを進める

auto inner_end = ranges::end(inner);

bool b2 = inner_it == inner_end;
// 内側rangeの終端チェック時、以下の事をチェック
// 1. 元rangeの終端に到達しているか
// 2. デリミタ長が0ではないか
// 3. 現在位置にデリミタ要素が現れていないか
```

ところがこれをよく見ると、具体的にしていることは先程説明した遅延評価と同じである事が分かります。細かい動作の差はあれど、`lazy_split_view`は入力`range`のカテゴリに関わらず、シングルパスの走査によって分割を行っているわけです。この強力な遅延評価機構はしかし、`lazy_split_view`を複雑にすると共に文字列分割で使用しづらくしています。

### `lazy_split_view<R, P>`の諸特性

`lazy_split_view`は`range`の`range`を生成する`view`であり、外側`range`と内側`range`の2つを意識する必要があります。

まず、外側`range`の性質は次のようになります

- `reference` : `range_reference_t<R>`の`range`
- `range`カテゴリ
    - `R`が`forward_range`の場合 : `forward_range`
    - それ以外の場合 : `input_range`
- `common_range` : `R`が`forward_range`かつ`common_range`の場合
- `sized_range` :  ×
- `const-iterable` : `R`および`const R`が共に`forward_range`である場合
- `borrowed_range` : ×

外側`range`は`R`が`input_range`であるときは`input_range`となり、内部に`R`のイテレータを保持するため`const-iterable`となりません。また、遅延評価を行うためにインクリメント以外でイテレータを進行させることが出来ないため、`forward_range`より強くなりません。

内側`range`は外側`range`の`reference`であり入力`range`（`R`）の部分要素ですが、`subrange`を用いるわけではなく独自の`range`型となります。

- `reference` : `range_reference_t<R>`
- `range`カテゴリ : 外側`range`のカテゴリに従う
- `common_range` : ×
- `sized_range` :  ×
- `const-iterable` : ◯
- `borrowed_range` : ×

内側`range`はどんなにがんばっても`forward_range`にしかならないため、多くの文字列を扱う所（最低でも`bidirectional_range`を要求する）で使用できなくなっています。内側`range`を独自型で定義しなおしているのは遅延評価を実装するためですが、この事が`lazy_split_view`を文字列分割に使用しづらくしています。

### `views::lazy_split`

`lazy_split_view`に対応するRangeアダプタオブジェクトが`views::lazy_split`です。

```cpp
int main() {
  const auto str = std::string_view("split_view takes a view and a delimiter");

  for (auto inner_range : views::lazy_split(str, ' ')) {
    for (char c : inner_range) {
      std::cout << c;
    }
    std::cout << '\n';
  }

  // パイプラインスタイル
  for (auto inner_range : str | views::lazy_split(' ')) {
    for (char c : inner_range) {
      std::cout << c;
    }
    std::cout << '\n';
  }
}
```

`views::lazy_split`はカスタマイゼーションポイントオブジェクトであり、引数の式`r, p`によって`views::lazy_split(r, p)`のように呼び出されたとき、`lazy_split_view(r, p)`を返します。

### 任意のシーケンスによる任意のシーケンスの分割

実のところ、`lazy_split_view`はとてもとてもジェネリックに定義されているため分割対象は文字列に限らず、デリミタもまた文字だけではなく任意の`forward_range`を使用することができます。

たとえば、`std::vector`を`std::list`で分割することができます。

```cpp
int main() {
  // 3つ1が並んでいるところで分割する
  std::vector<int> vec = {1, 2, 4, 4, 1, 1, 1, 10, 23, 67, 9, 1, 1, 1, 1111, 1, 1, 1, 1, 1, 1, 9, 0};
  std::list<int> delimiter = {1, 1, 1};
  
  for (auto inner_range : vec | std::views::lazy_split(delimiter)) {
    for (int n : inner_range) {
      std::cout << n;
    }
    std::cout << '\n';
  }
  /*
    1244
    1023679
    1111
    
    90
  */
}
```

実際このようなジェネリックな分割が必要になる事があるのかはわかりません・・・

遅延評価を頑張ることとこのジェネリックさによって、`lazy_split_view`は文字列で使いづらくなってしまっています。

## `split_view`

`split_view`は文字列分割に特化した`lazy_split_view`です。

```cpp
int main() {
  // ホワイトスペースをデリミタとして文字列を切り出す
  split_view sv{"split_view takes a view and a delimiter", ' '};
  
  // split_viewは切り出した文字列のシーケンスとなる
  // その要素は容易に文字列として扱う事ができる
  // string_viewへの直接変換はC++23から
  for (std::string_view str : sv) {
    std::cout << str << '\n';
  }
  /*
    split_view
    takes
    a
    view
    and
    a
    delimiter
  */
}
```

`split_view`もやはり、分割した`range`のシーケンスとなる`range`を返します。しかし、`lazy_split_view`と異なり内側の`range`は入力`range`のイテレータを利用した`subrange`となるため、文字列であれば内側`range`は`contiguous_range`かつ`common_range`となり、かなり扱いやすくなっています。そして、`string_view`にはC++23より`range`コンストラクタが追加されているので、そのままストレートに変換する事ができます。

```cpp
namespace std::ranges {
  // split_viewの定義例
  template<forward_range V, forward_range Pattern>
    requires view<V> && view<Pattern> &&
             indirectly_comparable<iterator_t<V>, iterator_t<Pattern>, ranges::equal_to>
  class split_view : public view_interface<split_view<V, Pattern>> {
    // ...
  };

  // デリミタがrange（文字列）である場合
  template<class R, class P>
    split_view(R&&, P&&) -> split_view<views::all_t<R>, views::all_t<P>>;

  // デリミタが単一要素（文字）である場合
  template<forward_range R>
    split_view(R&&, range_value_t<R>)
      -> split_view<views::all_t<R>, single_view<range_value_t<R>>>;
}
```

文字列に特化したと言いつつ、実際のところ入力となる`range`は`forward_range`であればよく、デリミタ（`Pattern`）も`forward_range`であればOKです。例えば`std::list`などを、あるいはそれによって分割する事ができるわけです。しかし、`input_range`の分割をサポートしていないことがここからわかり、`lazy_split_view`が行っていたような複雑な遅延評価を行う必要が無くなっています。また、デリミタに単一要素を指定しても`single_view`を用いて`range`化する事で、デリミタを常に`range`として統一的に扱っている事が推論補助から見て取れます。

たとえば、文字列を文字列で分割することができます。

```cpp
using namespace std::literals;

int main() {
  // カンマとホワイトスペースで区切りたい
  // そのままだとナル文字\0が入るのでうまくいかない
  split_view ng{"1, 12434, 5, 0000, 3942", ", "};

  for (std::string_view str : ng) {
    std::cout << str << '\n';
  }
  // 1, 12434, 5, 0000, 3942

  // デリミタ文字列を2文字分の範囲にする
  // string_viewは\0を含めない範囲となる
  split_view ok{"1, 12434, 5, 0000, 3942", ", "sv};

  for (std::string_view str : ok) {
    std::cout << str << '\n';
  }
  /*
    1
    12434
    5
    0000
    3942
  */
}
```

ナル文字`\0`の扱いは少し厄介です。文字列としての終端は最後の`\0`ですが`range`としての終端は`\0`の次になり、`split_view`はデリミタ列を`range`として扱って比較を行うために文字列終端の`\0`を含めてしまいます。例にあるように、`string_view`（あるいは`string`）を通すことでナル文字を除いた部分の文字列による範囲となるようになるため、意図通りになります。

### 遅延評価

`split_view`もまた遅延評価によって分割処理と`view`の生成を行いますが、`lazy_split_view`とは少し異なります。

`split_view`では、`.begin()`による外側イテレータの取得時に最初のデリミタ位置を探索し文字列先頭位置とともに記録しておきます。そして、外側イテレータのインクリメントのタイミングでデリミタ位置と文字列先頭位置の更新を行なっていきます。外側イテレータの間接参照では、現在の文字列先頭位置と次のデリミタ位置のイテレータから切り出す文字列を`subrange`によって返すため、内側`range`は特別なことを何もしません。外側イテレータの終端チェックでは、デリミタ列に一致する文字列が終端に来ているケースを考慮するために少し判定が増えていますが、`lazy_split_view`と比べると単純になっています。

```cpp
split_view sv{"split_view takes a view and a delimiter", ' '};
// 構築時点では何もしない

auto outer_it = ranges::begin(sv);
// 外側イテレータの取得時、最初にデリミタ列に一致する部分範囲を計算
// 外側イテレータは、文字列先頭位置と次に出現するデリミタ位置を
// イテレータで管理している

++outer_it;
// 外側イテレータのインクリメント時、次に出現するデリミタ列を探し更新
// 文字列先頭位置を前のデリミタ位置から求め更新
// 内部rangeは"split_view"->"takes"へ進む

bool b = outer_it == ranges::end(sv); // false
// 外側range終端チェック
// 元のシーケンス上での終端チェックと
// 現在の位置でデリミタが出現しているかどうかを調べる

auto inner_range = *outer_it;
// 内側rangeの取得、subrangeを作って返す
// 現在の文字列先頭位置イテレータ(cur)と
// 次に出現するデリミタ列の先頭イテレータ(next)から、
// subrange{cur, next}を返す

auto inner_it = ranges::begin(inner_range);
// 内側イテレータは元のrangeのイテレータを返す

++inner_it;
// 内側イテレータのインクリメントは、元の範囲のイテレータのインクリメント

char c = *inner_it; // a
// 元のシーケンスのイテレータのデリファレンスと等価
```

`lazy_split_view`の様に過度な一般化と過度な遅延評価を行わない事によって、文字列分割時に結果を文字列に簡単に変換可能である様に実装されています。一方、`begin()`の呼び出しで最初のデリミタ位置の探索を行っているため、`range`コンセプトの意味論要件を満たすために常にキャッシュが使用されます。これによって、`split_view`の外側`range`は`const-iterable`でも`borrowed_range`でもなくなります（とはいえそれは`lazy_split_view`とほぼ同じことです）。

### `split_view<R, P>`の諸特性

`split_view`の外側`range`の性質は次の様になります。

- `reference` : `subrange<iterator_t<R>>`
- `range`カテゴリ : `forward_range`
- `common_range` : `R`が`common_range`の場合
- `sized_range` :  ×
- `const-iterable` : ×
- `borrowed_range` : ×

`input_range`サポートを切ったためカテゴリは常に`forward_range`となります（`forward`より強くならないのは`lazy_split_view`と同様の理由による）。

内側`range`は入力`range`（`R`）のイテレータによる`subrange`であるので、`R`の性質をそのまま受け継ぎます。

- `reference` : `range_reference_t<R>`
- `range`カテゴリ : `R`のカテゴリに従う
- `common_range` : `R`が`common_range`の場合
- `sized_range` :  `R`が`sized_range`の場合
- `const-iterable` : `R`が`const-iterable`の場合
- `borrowed_range` : ◯

内側`range`とそのイテレータが何もしなくなったことによって、内側`range`の性質が大きく改善されています。これによって、文字列分割への適性が大きく向上しています。

\clearpage

### `views::split`

`split_view`に対応するRangeアダプタオブジェクトが`views::split`です。

```cpp
int main() {
  const auto str = std::string_view("split_view takes a view and a delimiter");

  for (auto inner_range : views::split(str, ' ')) {
    for (char c : inner_range) {
      std::cout << c;
    }
    std::cout << '\n';
  }

  // パイプラインスタイル
  for (auto inner_range : str | views::split(' ')) {
    for (char c : inner_range) {
      std::cout << c;
    }
    std::cout << '\n';
  }
}
```

`views::split`はカスタマイゼーションポイントオブジェクトであり、引数の式`r, p`によって`views::split(r, p)`のように呼び出されたとき、`split_view(r, p)`を返します。

## Counted view

*Counted view*は元となるシーケンスの先頭から指定された個数の要素のシーケンスを生成する`view`です。

```cpp
int main() {
  auto iota = views::iota(1);
  
  for (int n : views::counted(ranges::begin(iota), 5)) {
    std::cout << n; // 12345
  }
}
```

この*Counted view*は`couted_view`のようなクラスがあるわけではなく、生成するには`views::counted`を使用します。`views::counted`はカスタマイぜーションポイントオブジェクトであり、引数の式`i, n`によって`views::counted(i, n)`のように呼び出されたとき、その効果は次の様になります（`T = decay_t<decltype((E))>, D = iter_difference_t<T>`として）

1. `n`の型と型`D`が`convertible_to<decltype((n)), D>`のモデルとならない場合、、*ill-formed*
2. `T`が`contiguous_iterator`である場合、`span(to_address(i), static_cast<D>(n))`
3. `T`が`random_access_iterator`である場合、`subrange(i, i + static_cast<D>(n))`
4. それ以外の場合、`subrange(counted_iterator(i, n), default_sentinel)`

イテレータ1つと要素数を受け取って、イテレータに応じて最適な型を選択してサイズ`n`の`range`を返してくれます。`counted_iterator`というのはC++20から追加されたイテレータラッパで、イテレータとカウント数からそのカウント数だけ進行可能なイテレータを作成するものです。`couted_view`はないのですが、`views::counted`には`counted_iterator`が対応しています。

ただ、`views::counted`はRangeアダプタオブジェクトではないのでパイプラインスタイルで使用することはできません。どちらかというとRangeファクトリっぽいものですがそうではなくRangeアダプタでもないのですが、他のカテゴライズもないのでとりあえずRangeアダプタとして扱われている感じがあります。

### `take_view`との差異

*Counted view*はまさに`take_view`と同じことをしてくれますが、次の様な違いがあります。

- `take_view`は`view`を受けるが、*Counted view*はイテレータを受け取る
- *Counted view*はオーバラン防止のためのケアをしない

```cpp
int main() {
  int arr[] = {1, 2, 3};

  // 範囲を飛び越す！
  for (int n : views::counted(std::ranges::begin(arr), 5)) {
    std::cout << n; // 123??
  }
}
```

*Counted view*は単体のイテレータに対して`take_view`相当のものを生成するためのものであり、イテレータ1つではその範囲の終端は分からないのでオーバーランを防ぐことができないのです。

### Counted viewの諸特性

入力のイテレータ型を`I`として

- `reference` : `iter_reference_t<I>`
- `range`カテゴリ : `I`のイテレータカテゴリに対応するカテゴリ
- `common_range` : `I`が`random_access_iterator`の場合
- `sized_range` :  ◯
- `const-iterable` : ◯
- `borrowed_range` : ◯

## `common_view`

`common_view`は元となる`range`を`common_range`に変換する`view`です。

```cpp
int main() {
  // common_rangeではないシーケンス
  auto even_seq = views::iota(1)
    | views::filter([](int n) { return n % 2 == 0; })
    | views::take(10);

  // イテレータ型と終端イテレータ型が合わないためエラー 
  std::vector<int> vec(ranges::begin(even_seq), 
                       ranges::end(even_seq));  // ng
  
  common_view common{even_seq};
  
  std::vector<int> vec(ranges::begin(common), ranges::end(common)); //ok

  for (int n : vec) {
    std::cout << n; // 2468101214161820
  }
}
```

`common_range`とは`begin()/end()`によって取得できるイテレータと終端イテレータの型が同じとなる`range`のことでした。これはC++20以降のイテレータ（Rangeライブラリの`view`などのイテレータ）をC++17以前のイテレータを受け取るものに渡す際に使用できます。

標準コンテナのイテレータペアを受け取るコンストラクタや`<algorithm>`の各種アルゴリズム群など、C++17以前のイテレータを受け取るところでは`begin()/end()`の型が同一である事を前提としています。しかし、C++20以降のイテレータおよび`range`では異なっていることが前提です。特に、各種`view`型の場合は元となる`range`の種別や構築のされ方によって`begin()/end()`の型は細かく変化するので、古いライブラリと組み合わせる際に`common_view`が必要となります。

`<algorithm>`のイテレータアルゴリズム関数群は`std::ranges`名前空間の下にあるRangeアルゴリズムを利用すればC++20以降のイテレータに対してもそのまま使用できるようになっていますが、標準コンテナのイテレータペアを取るコンストラクタや`<numeric>`にあるアルゴリズム関数などでは`common_view`を使用する必要があります。

```cpp
int main() {
  auto even_seq = std::views::iota(1)
    | std::views::filter([](int n) { return n % 2 == 0; })
    | std::views::take(10);

  auto common = std::views::common(even_seq);

  auto less_than_10 = [](int n) { return 10 < n;};
  
  // 古いやつ
  auto it1 = std::find_if(common.begin(), common.end(), less_than_10);
  std::cout << *it1 << '\n';  // 12
  
  // 新しいやつ
  auto it2 = std::ranges::find_if(even_seq.begin(), even_seq.end(), less_than_10);
  std::cout << *it2 << '\n';  // 12
  
  // rangeそのまま
  auto it3 = std::ranges::find_if(even_seq, less_than_10);
  std::cout << *it3 << '\n';  // 12

  // 古いやつ
  auto sum = std::accumulate(common.begin(), common.end(), 0u);
  std::cout << sum << '\n';   // 110
  
  // まだない・・・
  //auto sum2 = std::ranges::accumulate(even_seq.begin(), even_seq.end(), 0u);
  //auto sum3 = std::ranges::accumulate(even_seq, 0u);
}
```

\clearpage

`common_view`は次の様に宣言されています。

```cpp
namespace std::ranges {
  template<view V>
    requires (!common_range<V> && copyable<iterator_t<V>>)
  class common_view : public view_interface<common_view<V>>{
    // ...
  };

  template<class R>
    common_view(R&&) -> common_view<views::all_t<R>>;
}
```

入力となる`range`は`common_range`ではなく、そのイテレータがコピー可能である必要があります。すなわち、すでに`common_range`である`range`に対しては`common_view`を使用できません。

### `common_view<R>`の諸特性

ここでの`R`は常に`common_range`では無い事を意識してください。

- `reference` : `range_reference_t<R>`
- `range`カテゴリ
    - `R`が`random_access_range`かつ`sized_range`の場合 : `R`のカテゴリに従う
    - `R`が`forward_range`の場合 : `forward_range`
    - それ以外の場合 : `input_range`
- `common_range` : ◯
- `sized_range` :  `R`が`sized_range`の場合
- `const-iterable` : `R`が`const-iterable`の場合
- `borrowed_range` : `R`が`borrowed_range`の場合

`R`が`random_access_range`かつ`sized_range`の時はイテレータに対してサイズを`+/+=`すれば、`O(1)`の操作で終端イテレータが同じイテレータ型で得られ、`R`のイテレータをそのまま利用できるため`R`のカテゴリを継承します。

それ以外の場合は`common_iterator`を利用して元のイテレータをラップするため`R`のカテゴリと同じにはなりません。イテレータ型と番兵型が異なっている場合、`R`が`bidirectional`でもその番兵（`sentinel_t<R>`）に対して`--`操作が定義されておらず、その様なイテレータペアをラップした`common_iterator`も`--`操作を行えないため、`forward_iterator`となるのが精一杯です。そのため、`common_iterator`を利用する場合の`common_view`も`forward`にしかなりません。

### `views::common`

`common_view`に対応するRangeアダプタオブジェクトが`views::common`です。

```cpp
int main() {
  auto even_seq = views::iota(1)
    | views::filter([](int n) { return n % 2 == 0; })
    | views::take(10);

  auto common1 = views::common(even_seq);
  
  std::vector<int> vec(ranges::begin(common1), ranges::end(common1)); //ok

  for (int n : vec) {
    std::cout << n; // 2468101214161820
  }
}
```

```cpp
int main() {
  // パイプラインスタイル
  auto even_seq = views::iota(1)
    | views::filter([](int n) { return n % 2 == 0; })
    | views::take(10)
    | views::common;

  std::vector<int> vec(ranges::begin(even_seq), 
                       ranges::end(even_seq));  //ok
  
  for (int n : vec) {
    std::cout << n; // 2468101214161820
  }
}
```

`views::common`はカスタマイゼーションポイントオブジェクトであり、引数の式`r`によって`views::common(r)`のように呼び出されたとき、その効果は次の様になります

1. `decltype((E))`が`common_range`の場合、`views::all(r)`
2. それ以外の場合、`common_view{r}`

結果の型を区別しなければ、あらゆる`range`オブジェクトに対して`common_view`相当のものを得ることができます。特に、`common_view`は`common_range`から構築することができないので、`common_view`が欲しい際は`views::common`を利用するとよりジェネリックです。

## `reverse_view`

`reverse_view`は元となるシーケンスを逆順にしたシーケンスを生成する`view`です。

```cpp
int main() {
  // [10, 30, 50, 70, 90]
  auto seq = views::iota(1, 10)
    | views::filter([](int n) { return n % 2 == 1; })
    | views::transform([](int n) { return n * 10; });
  
  reverse_view rv{seq};
  
  for (int n : rv) {
    std::cout << n; // 9070503010
  }
}
```

`reverse_view`は次のように宣言されています。

```cpp
namespace std::ranges {
  template<view V>
    requires bidirectional_range<V>
  class reverse_view : public view_interface<reverse_view<V>> {
    // ...
  };
}
```

逆順にするという操作の都合上、入力となる`range`は`bidirectional_range`でなければなりません。

### 遅延評価

`reverse_view`は遅延評価によって逆順範囲を生成します。とはいえ、逆順範囲の生成と管理を`reverse_iterator`に丸投げしているので、`reverse_view`の行うことは`begin()`によってイテレータを取得するタイミングで`reverse_iterator`を適切に構築することです。

そこではまず、元の`range`が`common_range`であればその終端イテレータ（`end()`で取得できるイテレータ）を使って`std::reverse_iterator`を構築します。元の`range`が`common_range`ではない場合、元の`range`のイテレータを`ranges::next()`によって終端まで進めて、それによって`reverse_iterator`を構築します。

`reverse_iterator`は自身のインクリメントのタイミングなどで元のイテレータを逆に進める事でイテレータの逆順走査を行なっており、こちらも遅延評価されています。

```cpp
auto seq = views::iota(1, 10)
  | views::filter([](int n) { return n % 2 == 1; })
  | views::transform([](int n) { return n * 10; });

reverse_view rv{seq};
// 構築の時点では何もしない

auto it = ranges::begin(rv);
// イテレータ取得時に元となるシーケンスの終端1つ手前のイテレータからreverse_iteratorを構築して返す
// 元となるシーケンスがcommon_rangeではない場合、時間計算量はO(N)になる

++it;
--it;
int n = *it;
// イテレータはreverse_iterator
```

元の`range`が`common_range`ではない場合、終端イテレータの取得はインクリメントによって行われるため、シーケンスの長さを`N`とした`O(N)`の時間計算量となってしまいます。そしてその場合、`reverse_view`の`begin()`の処理結果はキャッシュされます。これによって`reverse_view`の`begin()`の計算量は償却定数となります。

### `reverse_view<R>`の諸特性

ここでの`R`は少なくとも`bidirectional_range`です。

- `reference` : `range_reference_t<R>`
- `range`カテゴリ
    - `R`が`random_access_range`の場合 : `random_access_range`
    - それ以外の場合 : `bidirectional_range`
- `common_range` : ◯
- `sized_range` :  `R`が`sized_range`の場合
- `const-iterable` : `const R`が`common_range`の場合
- `borrowed_range` : `R`が`borrowed_range`の場合

`reverse_view`のイテレータは`reverse_iterator`を利用しているので、`reverse_view`は常に`common_range`となります。一方、`R`の`contiguous`性は継承されません。また、`R`が`common_range`では無い場合は`begin()`の呼び出しでキャッシュするので`const-iterable`ではありません。そのほかは`R`の性質を継承します。

### `views::reverse`

`reverse_view`に対応するRangeアダプタオブジェクトが`views::reverse`です。

```cpp
int main() {
  auto seq = views::iota(1, 10)
    | views::filter([](int n) { return n % 2 == 1; })
    | views::transform([](int n) { return n * 10; });
  
  for (int n : views::reverse(seq)) {
    std::cout << n; // 9070503010
  }
  
  std::cout << '\n';
  
  // パイプラインスタイル
  for (int n : seq | views::reverse) {
    std::cout << n; // 9070503010
  }
}
```

`views::reverse`はカスタマイゼーションポイントオブジェクトであり、引数の式`r`（その型を`R`とする）によって`views::reverse(r)`のように呼び出されたとき、その効果は次のようになります。

1. `R`が`reverse_view`の特殊化である場合、`r.base()`
2. `R`は`subrange<reverse_iterator<I>, reverse_iterator<I>, K>`の様な型であり
      1. `K == subrange_kind::sized`の場合、`subrange<I, I, K>(r.end().base(), r.begin().base(), r.size())`
      2. それ以外の場合、`subrange<I, I, K>(r.end().base(), r.begin().base())`
3. それ以外の場合、`reverse_view{r}`

ここで頻繁に呼び出されている`.base()`とか`.size()`等のメンバ関数は、Rangeライブラリの`view`及びそのイテレータ型が共通で備えているメンバ関数です。`view.base()`はその`view`の元になっている`range`を取得し、`iter.base()`はそのイテレータがラップしている元のイテレータを取得します。`view.size()`は元の`range`が`sized_range`であれば、その長さを取得します。

この複雑な条件分岐は、すでに逆順になっているシーケンス（`reverse_view`や`reverse_iterator`の`subrange`）に対してはその元になっている`range`及びイテレータを返す事によって、`reverse_view`や`reverse_iterator`が何回も適用される事を回避しています。`reverse`操作をしているのは3番目の効果だけで、1,2番目の効果は`reverse`している範囲を元に戻すものです。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4, 5, 6, 7, 8, 9};

  auto rv = vec | views::reverse  // vector<int> -> reverse_view
                | views::reverse  // reverse_view -> ref_view<vector<int>>
                | views::reverse  // ref_view<vector<int>> -> reverse_view
                | views::reverse; // reverse_view -> ref_view<vector<int>>

  for (int n : rv) {
    std::cout << n; // 123456789
  }
}
```

`view`を構築するときに元の`range`が`views::all`を通ることになるので完全にそのままとはなっていませんが、実質的には`reverse`の`reverse`で元の`range`に戻るようになっています。

## `elements_view`

`elements_view`は*tuple-like*な型の値のシーケンスから、指定された番号の要素を取り出したシーケンスを生成する`view`です。

```cpp
int main() {
  // tupleのシーケンス
  std::vector<std::tuple<int, double, std::string_view>> vec = { 
    {1, 1.0, "one"},
    {2, 2.0, "two"},
    {3, 3.0, "three"}
  };
  
  // 1つ目（int）のtuple要素のシーケンス
  elements_view<views::all_t<decltype((vec))>, 0> ev1{vec};
  
  for (int n : ev1) {
    std::cout << n; //123
  }
  
  // 3つ目（string_view）のtuple要素のシーケンス
  elements_view<views::all_t<decltype((vec))>, 2> ev2{vec};
  
  for (std::string_view sv : ev2) {
    std::cout << sv << ' '; // one two three
  }
}
```

*tuple-like*な型のシーケンスのそれぞれの要素を指定された番号`N`による`get<N>()`で射影し、その値のシーケンスを生成します。*tuple-like*な型であって`std::tuple`でなければならないわけでは無いので、`pair`とか`array`などのシーケンスに対しても使用できます。

`elements_view`には抽出する`tuple`要素の番号を非型テンプレートパラメータとして渡さなければならないため、クラステンプレートの実引数推論を利用できません。そのため、引数として渡す`range`の型を書く必要があります。その際は他の`view`がそうであるように`std::views::all_t`を使うのが便利なのですが、`decltype(())`としないと`views::all`に左辺値として渡せずにコンパイルエラーを起こします（`views::all`は右辺値をリジェクトします）。

これは特に、連想コンテナにおいて便利かと思われます。

```cpp
using namespace std::string_view_literals;

int main() {

  std::map<int, std::string_view> i2s = { 
    {6, "six"sv}, {1, "one"sv}, {0, "zero"sv}, 
    {2, "two"sv}, {3, "three"sv}, {5, "five"sv}, 
    {4, "four"sv}
  };
  
  // keyのシーケンス
  elements_view<views::all_t<decltype((i2s))>, 0> ev1{i2s};
  
  for (int n : ev1) {
    std::cout << n; // 0123456
  }
  
  std::cout << '\n';

  // valueのシーケンス
  elements_view<views::all_t<decltype((i2s))>, 1> ev2{i2s};
  
  for (std::string_view sv : ev2) {
    std::cout << sv << ' '; // zero one two three four five six
  }
}
```

`std::tuple`に`get`を使用する時と同様に、指定する要素番号は`0`始まりでなければなりません。

### 遅延評価

`elements_view`によるシーケンスもまた、遅延評価によって生成されます。とはいえそれは単純に、イテレータの間接参照のタイミングで元のシーケンスの要素に対して`get<N>`を適用して要素を取り出すだけで、`taransform_view`とやっていることは同じなので遅延評価の仕方も同じです。

```cpp
std::vector<std::tuple<int, double, std::string_view>> vec = { 
  {1, 1.0, "one"}, {2, 2.0, "two"}, {3, 3.0, "three"}
};
  
elements_view<views::all_t<decltype((vec))>, 0> ev1{vec};
// elements_view構築時は何もしない

auto it = ranges::begin(ev1);
// イテレータ取得時も何もしない

++it;
--it;
// インクリメント等進行操作はほぼ元のイテレータそのまま

auto&& elem = *it;
// 間接参照時に元のイテレータの間接参照結果に
// get<N>を適用して1要素だけを取り出す
```

`elements_view`が行う殆どの事はそのイテレータの間接参照時に集中しており、そのほかの操作は元のイテレータの薄いラッパとなります。

### `elements_view<R, N>`の諸特性

- `reference` : `tuple_element_t<N, range_reference_t<R>>`
- `range`カテゴリ
    - `R`が`random_access_range` : `random_access_range`
    - `R`が`bidirectional_range` : `bidirectional_range`
    - `R`が`forward_range` : `forward_range`
    - それ以外 : `input_range`
- `common_range` : `R`が`common_range`の場合
- `sized_range` :  `R`が`sized_range`の場合
- `const-iterable` : `R`が`const-iterable`の場合
- `borrowed_range` : `R`が`borrowed_range`の場合

`reference`は確かに`tuple_element_t`によって分かりますが、`std::get`は引数の値カテゴリで細かく変動するため、言うほど推測が簡単なわけではありません。`elements_view`は特殊化された`transform_view`であるので`R`の`contiguous`性を受け継ぎませんが、`transform_view`と異なり`R`がそうであれば`borrowed_range`となることができます。これは、変換関数が固定されているため`view`本体にそれを保存しておく必要が無いためです。

### `views::elements`

`elements_view`に対応するRangeアダプタオブジェクトが`views::elements`です。

```cpp
int main() {
  std::vector<std::tuple<int, double, std::string_view>> vec = { 
    {1, 1.0, "one"}, {2, 2.0, "two"}, {3, 3.0, "three"}
  };
  
  for (int n : views::elements<0>(vec)) {
    std::cout << n; //123
  }

  // パイプラインスタイル
  for (std::string_view sv : vec | views::elements<2>) {
    std::cout << sv << ' '; // one two three
  }
}
```

`views::elements`はカスタマイぜーションポイントオブジェクトであり、数値`N`と引数の式`r`によって`views::elements<N>(r)`のように呼び出されたとき、`elements_view<views::all_t<decltype((E))>, N>{r}`を返します。

`elements_view`は要素番号をテンプレートパラメータで受け取る都合上クラステンプレートの実引数推論が効かないため、テンプレートパラメータに引数`range`の型を書かなければなりませんが、`views::elements`を使うことでそれを省略できます。

### `keys_view/values_view`

`elements_view`は連想コンテナにおいてよく使用される事を想定しているためか、連想コンテナで使う際に便利なエイリアスがあらかじめ用意されています。これを用いると、冒頭の連装コンテナのサンプルは次のように書くことができます。

```cpp
using namespace std::string_view_literals;

int main() {

  std::map<int, std::string_view> i2s = {
    {6, "six"sv}, {1, "one"sv}, {0, "zero"sv}, 
    {2, "two"sv}, {3, "three"sv}, {5, "five"sv}, 
    {4, "four"sv}
  };
  
  // keyのシーケンス
  keys_view ev1{i2s};
  
  for (int n : ev1) {
    std::cout << n; // 0123456
  }
  
  std::cout << '\n';

  // valueのシーケンス
  values_view ev2{i2s};
  
  for (std::string_view sv : ev2) {
    std::cout << sv << ' '; // zero one two three four five six
  }
}
```

`keys_view/values_view`は`elements_view`のエイリアステンプレートで、次のように定義されます。

```cpp
namespace std::ranges {
  template<typename R>
  using keys_view = elements_view<views::all_t<R>, 0>;

  template<typename R>
  using values_view = elements_view<views::all_t<R>, 1>;
}
```

C++20からはエイリアステンプレートの実引数推論が導入されているので、これらを利用すると完全に型を省略できるようになります。

さらに、`keys_view/values_view`に対応するRangeアダプタオブジェクトも用意されています。

\clearpage

```cpp
using namespace std::string_view_literals;

int main() {

  std::map<int, std::string_view> i2s = {
    {6, "six"sv}, {1, "one"sv}, {0, "zero"sv}, 
    {2, "two"sv}, {3, "three"sv}, {5, "five"sv}, 
    {4, "four"sv}
  };
  
  // keyのシーケンスをイテレート
  for (int n : i2s | views::keys) {
    std::cout << n; // 0123456
  }
  
  std::cout << '\n';

  // valueのシーケンスをイテレート
  for (std::string_view sv : i2s | views::values) {
    std::cout << sv << ' '; // zero one two three four five six
  }
}
```

`views::keys/views::values`は`views::elements`の単なる特殊化です。

```cpp
namespace std::ranges::views {

  // views::elements
  template<size_t N>
    inline constexpr /*unspecified*/ elements = /*unspecified*/;

  // view::keys/views::values
  inline constexpr auto keys = elements<0>;
  inline constexpr auto values = elements<1>;
}
```

連想コンテナについては、これらを用いると簡潔に書くことができる様になります。

\clearpage

# おわりに

他のモダンな言語でのシーケンス操作に触れたことがある人には、C++20 Rangeライブラリの機能、特に`view`は物足りなく映ることでしょう。全部で17個しかない`view`の中には、例えば`zip`や`concat`等のよく利用されるものが含まれていませんから・・・

とはいえ、Rangeライブラリはこれで終わりではありません。Rangeライブラリのほとんどの部分は*range-v3*というライブラリでの経験をベースにしています（そもそも、提案者がその作者のEric Nieblerさんだったりします）が、それは巨大なライブラリであり、そのすべてを一度にC++に導入しようとすると膨大な作業が必要となります。ともすれば、十分な議論を尽くすことができず、将来に遺恨を残すことになりかねません。

そのため、C++20ではRangeライブラリの基礎となるコンセプトとユーティリティ、及び基本的な`view`だけに範囲を絞ったうえで提案され、採択されています。C++20のRangeライブラリが物足りないのはあえてのことです。

ちなみに、この本を執筆している2021年6月終盤の時点で既にC++23にいくつかの新機能やバグ修正が採択されており、その中にはRangeライブラリに関わる物がいくつかあります。そして、その全てがバグ修正であり一部はC++20に対する欠陥報告として遡及して適用されています。C++20 Rangeライブラリは*range-v3*の経験をベースに時間をかけて議論されてきたはずですが、それでも決して少なくはない問題を抱えてしまっていたわけです。この事実からも、慎重な議論と検討が必要であることが理解できるかと思います。

C++23に向けては既にいくつかの`view`の提案が出されており、本格的にRangeライブラリの拡張が始まる予定です。そこには`view`だけではなく、`range`に対する*Algorithm*（`accumulate, inner_product`などの様な操作）や*Action*（`range`に直接作用する`sort, copy`などの操作）も含まれています。「P2214R0 A Plan for C++23 Ranges(http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2214r0.html)」に展望が描かれています。

とは言えやはり、C++は長期感使われることを見越した上で機能が安定していることを重視しており、議論は慎重に十分な時間をかけて行われます。そのため一度に全部とはいかず、優先度を付けながら、すこしづつ新しいものを導入していく事になります。

\clearpage

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- C++20 Range Adaptors and Range Factories - Barry Revzin  
  (https://brevzin.github.io/c++/2021/02/28/ranges-reference/)
- C++20のRangeライブラリの強力な機能、プロジェクション - 本の虫 : ライセンスはCC-BY-SA 4.0)  
  (https://ezoeryou.github.io/blog/article/2019-01-22-ranges-projection-ja.html)
- yohhoyの日記(https://yohhoy.hatenadiary.jp/)

表紙は友人のKさんに書いていただきました。可愛いキノコをありがとうございました！

表紙にあるC++のロゴは「Standard C++ Foundation」の正式なものですが、本書および本サークルは「Standard C++ Foundation」からなんらかの支持を得たものではありません。「Standard C++ Foundation」およびそのロゴについては以下のリンクをご参照ください。

- Standard C++ (https://isocpp.org)
- Foundation name and C++ logo (https://isocpp.org/home/terms-of-use)