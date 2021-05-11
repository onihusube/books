---
title: C++20 <ranges>
author: onihusube
date: 2021/05/05
geometry:
  width: 188mm
  height: 263mm
#coverimage: cover.jpg
#backcoverimage: backcover.jpg
okuduke:
  revision: 初版
  printing: ねこのしっぽ
---
\clearpage

# はじめに

Rangeライブラリのものは`std::ranges`名前空間にありますが、この本では基本的に省略します。ただし、Rangeライブラリ以外のものは`std::`を基本的には省略しません。

範囲とは配列などの様に要素の列となっているものを指します。この本では、範囲の事をシーケンスとも呼んでいます。

## Rangeライブラリとは

# コンセプト

C++20よりコンセプトが言語機能に正式に組み込まれました。これによってテンプレートパラメータに制約をかけたり、関数テンプレートに優先順を付ける事がC++17以前と比較してより簡単に行えるようになりました。それとともに、ライブラリにおける処理や操作の対象となる”もの・概念”をコンセプトによって明確に定義し、そのコンセプトを中心にプログラムをジェネリックに組み上げていくコンセプトベースのライブラリデザインの時代が幕を開けています。

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

ここに表れている`requires(引数){式...}`は`requires`式と呼ばれる制約式で、そこに記述された式が全てコンパイルエラーを起こさない（使用可能である）場合に`true`を返すものです。コンセプトは複数の制約式から構成され、それらを論理積（`&&`）や論理和（`||`）で繋げて構成します。具体的な型が渡されると、制約式1つ1つは`true/false`のどちらかを返し、全体として`true`となるときにそのコンセプトはその型について満たされます。

`range`コンセプトは1つの`requires`式だけから構成されています。その意味は単純で、型`T`のオブジェクト`t`に対して`std::ranges::begin(t)/std::ranges::end(t)`がどちらも呼び出し可能である事です。`std::ranges::begin()/std::ranges::end()`はそれぞれ範囲の先頭イテレータと終端イテレータ（番兵）を取得するものなので、`range`コンセプトを満たす型は`std::ranges::begin()/std::ranges::end()`が指定する手段によって範囲の先頭と終端を取得できることを表しています。

長いので、以降`std::ranges::begin()/std::ranges::end()`を単に`begin/end`と書くことにします。

そして、これに加えて次のような条件が文章で規定されます。

- `[begin(t), end(t))`が範囲を示す
- `begin(t), end(t)`の計算量は償却定数であり、範囲を変更しない
- `begin(t)`が`std::forward_iterator`コンセプトのモデルであるならば、`begin(t)`は等さを保持する（*equality-preserving*）

標準ライブラリのコンセプトの多くは、このような文章によって追加の要求がなされています。そのような要求の多くは、そのコンセプトを満たしていれば当然満足されているはずですが、コンセプト機能によってコードとしてチェックできるものではない事が指定されています。このような要件のことを __意味論的な要件（*semantic requirements*）__ と呼び、コンセプト定義に直接現れている制約（式）を __構文的な要件（*syntactic requirements*）__ と呼びます。

ある型`T`が、コンセプト`C`の構文的な要件と意味論的な要件の両方を全て満たしている時、「型`T`はコンセプト`C`のモデルである」と言います。

コンセプトは構文的な要件と意味論的な要件の両方が揃ってコンセプトなので、コンセプトを利用するときは構文的な部分だけでなく意味論的な部分にも注意を払う必要があります。

さて話を戻して、`range`コンセプトの意味論的な要件を見てみましょう。

> `[begin(t), end(t))`が範囲を示す

これは、同じオブジェクトから取得したイテレータと番兵は、同じ範囲についてのイテレータと番兵である事という意味です。ここでの範囲とは、配列やコンテナなどの要素の列となっているものを指します。  
例えば、`begin`と`end`がそれぞれ異なる配列のイテレータを返すような実装になっている場合、`range`コンセプトを構文的には満たしますがモデルとはなりません。

> `begin(t), end(t)`の計算量は償却定数であり、範囲を変更しない

ここでの計算量とはシーケンス`t`の要素数に基づくものです。すなわち、`t`の要素数が幾つであれ`begin/end`の呼び出しは一定時間で完了しなければなりません。償却とあるのは、繰り返し呼んだ時に結果的に一定時間となれば良いという意味で、例えば最初の呼び出しだけ何か処理をして結果をキャッシュしておいて、2回目以降の呼び出しはキャッシュから直ぐに返す、という実装が許されます。

範囲を変更しないとはそのままで、`t`の表す範囲を`begin/end`の呼び出しに伴って何か変更（要素を消したり、並び替えたり）してはいけません。

> `begin(t)`が`std::forward_iterator`コンセプトのモデルであるならば、`begin(t)`は等さを保持する（*equality-preserving*）

`std::forward_iterator`コンセプトは名前の通り、従来*forward iterator*と呼ばれていた種類のイテレータを定義するコンセプトで、C++20から`<iterator>`にて定義されています。イテレータカテゴリには継承関係があり、コンセプトにもそれに応じた包含関係があります。*forward iterator*より強いイテレータは同時に`std::forward_iterator`コンセプトのモデルでもあります。

ある式が等さを保持する（*equality-preserving*である）とは、ある式に同じ値を入力するといつも同じ結果が得られる事を言います。簡単に言えば、式の内部で状態を持ったりグローバルな変数に依存したりしていない、という事です。同じオブジェクトではなく「同じ値」であることに注意してください、同じオブジェクトを渡してもその値が変更されているならば同じ結果を返す必要はありません。

この意味論的な要件はつまり、`begin(t)`から得られるイテレータが*forward iterator*（あるいはより強いイテレータ）であるならば、`begin(t)`は同じ`t`に対して同じイテレータを返すべし、という事です。

この要件は何を示しているのでしょうか？そのヒントは、イテレータが*forward iterator*であるというところにあります。

*forward iterator*にはマルチパス保証が課せられています。マルチパス保証があるイテレータは、参照するシーケンスを複数のイテレータから走査することができます。すなわち、イテレータの操作によってシーケンスの状態が変更されることがありません。  
*forward iterator*ではないイテレータ（*input(output) iterator*）にはそれがありません。つまり、*input(output) iterator*であるイテレータはその操作（`++, *`など）によって参照するシーケンスの状態が変更されることがあります。そして、そのようなイテレータでは`begin()`によるイテレータ取得時に何かしてからイテレータを得る事があります。つまり、`begin()`の呼び出しが等しさを保持できない場合がありえます。

この要件はそのような特殊なイテレータを認めつつ、普通のイテレータに対しては当然の要求をするものです。このようなイテレータには、例えば`istream_iterator/ostream_iterator`があります。

見てみると、これらの意味論的な要件は実行時にすらチェックが不可能なものばかりですが、確かにそれが期待される要求である事がわかると思います。このように、コンセプトはコードで書く構文的な要件と文章で書く意味論的な要件の2本柱から構成され、それによってそれまで何かふわっとしていた概念というものを明確に定義します。

また別の見方をすると、コンセプト定義に現われる構文的な要件によってある概念にまつわる操作や性質を（文字通り構文的に）規定し、意味論的な要件によってそれらの操作や性質の意味を（文字通り意味論として）定め、その2つをもって1つの概念を構文・意味論の両方向から定義している、と見ることができます。

### コンセプトの暗黙の意味論的な要件

コンセプトには、全てのコンセプトで共通して要求される3つの暗黙の意味論的な制約があります。

- 等しさの保持（*equality-preserving*）
    - 同じ値の入力に対して同じ結果を返す
- 安定（*stable*）
    - 変更されていない同じオブジェクトの入力に対して同じ結果を返す
- 暗黙の式のバリエーション（implicit expression variations）
    - `requires`式において、`const T&`の引数を受ける制約式は期待される全ての値カテゴリの引数を受け入れる必要がある

安定は等しさの保持とセットであり、暗黙の式のバリエーションはほとんど気にする必要はないので、ここでは深く触れませんし他のところでも言及されないはずです。ただ、型がコンセプトのモデルとなるためにはこの3つの要件を全て満たしている必要があります。

等しさの保持（および安定）に関しては、「等しさを保持することを要求しない（*not required to be equality-preserving*）」のように指定される場合があり、この時は満たしている必要はありません。先ほどの`range`コンセプトでは条件付きで満たさなくても良いという指定でした。

### Rangeライブラリ

Rangeライブラリの"Range"とは何か？といえば、この`range`コンセプトを満たす（モデルとなる）あらゆる型の事です。それは`range`コンセプトが定義するように、何かしらの範囲（シーケンス）を表す型であり、`ranges::begin/ranges::end`という2つの操作によってその範囲の先頭要素を指すイテレータと終端を表す番兵のペアを取得することができます。

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

そして、`borrowed_range`はその定義中に`range`コンセプトを（`&&`によって繋ぐ形で）含んでいます。この場合`borrowed_range`は`range`コンセプトを完全に包含しており、この包含関係によってコンセプトの半順序が決定され、`borrowed_range`コンセプトによる制約は`range`コンセプトによる制約よりも強く順序付け（制約）されます（あえて記号で書くならば、`range ⊂ borrowed_range`、右側ほど強い）。2つのコンセプトの間に順序が付けられるとき、それらのコンセプトを用いてオーバーロードされている関数の間に呼び出しの優先順位を付けることができ、曖昧にならなくなります。ただし、制約式同士が`||`で繋がれている場合はこの限りではなく、ルールはより複雑になります・・・

```cpp
// コンセプトを使用する時は、typenameのところにコンセプトを指定する
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

`enable_borrowed_range`はRangeライブラリの一部の*View*と`std::string_view`、`std::span`に対してはこのような特殊化が用意されており、それらの型は`borrowed_range`のモデルとなることができます。

## `sized_range`

`sized_range`は範囲の要素数（長さ）が一定時間で求められる`range`を定義するコンセプトです。

```cpp
template<class T>
concept sized_range =
  range<T> &&
  requires(T& t) { ranges::size(t); };
```

意味論要件は

`T`から参照を取り除いた型のオブジェクト`t`について

- `ranges::size(t)`の計算量は償却定数であり`t`を変更せず、結果は`ranges::distance(t)`と等しくなる
- `iterator_t<T>`が`std::forward_iterator`のモデルであるとき、`ranges::size(t)`は`ranges::begin(t)`の評価と関係なく呼び出し可能

1つ目の要件は意味が分かると思います。範囲の長さと範囲の距離は同じものを別の名前で呼んでいるだけです。

> `iterator_t<T>`が`std::forward_iterator`のモデルであるとき、`ranges::size(t)`は`ranges::begin(t)`の評価と関係なく呼び出し可能

前半は`T`から`begin`で取れるイテレータが*forward iterator*であるとき、という事ですが、後半のこれは何を言っているのでしょうか・・・

`range`コンセプトを思い返してみると、その`begin`の呼び出しは償却定数であることが求められており、例えば最初の`begin`の呼び出し時に何かをしてそれをキャッシュし2回目以降はキャッシュから返すような実装が許される、という事を言っていました。ここでの何かをするというのは殆どの場合シーケンスを生成する事だと思われます。その場合、そのクラスの表現するシーケンスというものは`begin`を最初に呼び出すまでは生成されておらず、そのクラスが`sized_range`であったとしても`begin`の呼び出し前にはシーケンスの長さを求めることは出来ないわけです。

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

`ranges::size`はCPOとして様々な手段でその引数から長さを取得しようとします。結果として、`sized_range`コンセプトを構文的には満たしても意味論的な要件まで満たせない型が出て来てしまいます。これは、そのような場合に`sized_range`を無効化するためのものです。  

例えば、`std::forward_list`のようにサイズを求めることは（イテレータを用いて）出来るけれど計算量が償却定数にならないような実装になっている場合に活用できます（ただ、`std::forward_list`そのものは`ranges::size`でサイズを求められません）。

## `input_range`

`input_range`はそのイテレータが*input iterator*であるような*range*を定義するコンセプトです。

```cpp
template<class T>
  concept input_range =
    range<T> && input_iterator<iterator_t<T>>;
```

意味論的な要件は指定されていません。

`input_iterator`コンセプトはその名の通り*input iterator*を定義するコンセプトです。

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

`std::input_or_output_iterator`コンセプトは前置/後置インクリメントや間接参照、デフォルト構築とムーブが可能であるなど、イテレータとしての基本的な要件を指定するものです。`std::indirectly_readable`コンセプトは間接参照によって値を読み取る事ができる事を表すものです。

`ITER_CONCEPT(I)`というのは説明のためのエイリアステンプレートのようなもので、イテレータ型`I`から最も適切な*iterator category*を取得してくるものです。そして、`derived_from`コンセプトでそれが`std::input_iterator_tag`から派生しているかをチェックしています。継承関係も含めてチェックすることでより強いイテレータを含めて判定しており、また将来的なイテレータカテゴリの拡張に備えています。

C++17までの*input iterator*と意味は同じなのですが、求められる事が若干変化しているため、C++17とC++20の*input iterator*には相互に互換性がありません。

## `output_range`

`output_range`はそのイテレータが*output iterator*であるような*range*を定義するコンセプトです。

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

- 型`T`のオブジェクト`t`（値カテゴリは任意）、間接参照可能な型`I`のオブジェクト`i`）について、`*i++ = v;`は次の式と等価
  ```cpp
  *i = E;
  ++i;
  ```

`std::indirectly_readable`コンセプトは間接参照（`*`）による値の読み取りを定義するコンセプトです。

`std::output_iterator`の意味論的な要件は何を言っているのか一見わかりませんが、要はイテレータへの出力と進行の操作を1行で書いても分けて書いても同じ結果となる事を要求しています。

C++17までの*output iterator*と意味は同じですが求められる事が若干厳しくなっており、C++17までの*output iterator*はこのコンセプトを満たす事ができない場合があります。逆にC++20*output iterator*はC++17*output iterator*に対して後方互換性があります。

## `forward_range`

`forward_range`はそのイテレータが*forward iterator*であるような*range*を定義するコンセプトです。

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

- `forward_iterator`の`==`は、同じ範囲を参照しているイテレータの全てと（異なる型も含めて）比較可能である。
    - イテレータの型が同じである場合、デフォルト構築されたイテレータとの比較が可能である。
    - そのようなデフォルト構築されたイテレータ同士の比較は常に`true`となる（同じ空の範囲の終端を指しているかのように振る舞う）。
- 範囲`[i, s)`を参照する`forward_iterator`から取得された`[i, s)`への参照やポインタは、`[i, s)`が範囲として有効である限り有効であり続ける。
- マルチパス保証。

`incrementable`はインクリメント（`++`）によるイテレータの進行を定義するコンセプトで、同時にコピー/ムーブ構築と代入、デフォルト構築を要求します。`sentinel_for`はイテレータ型に対する番兵型を定義するコンセプトで、`sentinel_for<I1, I2>`は型`I1`がイテレータ型`I2`の番兵であることを表しています。すなわち、`forward_iterator`である`I`は自分自身がその範囲の終端を表現できなければなりません。

意味論要件は少し難解ですが、デフォルト構築されたイテレータが範囲の終端を指すようになり、かつ他のイテレータはそれと常に比較可能であることや、イテレータの参照する要素の有効性は範囲の生存期間に従う（イテレータの操作と無関係になる）など、普通のイテレータに期待される振る舞いを定義しています。

3つ目のマルチパス保証とはコードで書くと次のような要件です。

- `a == b`ならば`++a == ++b`
- `((void)[](X x){++x;}(a), *a)`は`*a`と等しい

特に2つ目は何言ってるのか分かり辛いですが、「同じ範囲を指すイテレータは同じ範囲を同じ順番でイテレートする」「イテレータをコピーしてから何かをしても、コピー元のイテレータに影響はない」という事を言っています。これは*input iterator*にはない`forward_iterator`の最大の特徴です。

マルチパス保証があることで、ある範囲を別々のイテレータから複数回走査するような、マルチパスなアルゴリズムをイテレータを用いて書くことができます。

C++17までの*forward iterator*と意味は同じですが求められる事が若干厳しくなっており、C++17までの*forward iterator*はこのコンセプトを満たす事ができない場合があります。逆にC++20*forward iterator*はC++17*forward iterator*に対してほぼ後方互換性があります。

## `bidirectional_range`

`bidirectional_range`はそのイテレータが*bidirectional iterator*であるような*range*を定義するコンセプトです。

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

## `random_access_range`

`random_access_range`はそのイテレータが*random access iterator*であるような*range*を定義するコンセプトです。

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

`iter_reference_t<I>`の示す型`D`、`D`の値`n`、型`I`の有効なイテレータ`a`と`++a`を`n`回適用したイテレータ`b`について

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

`totally_ordered`は全順序による比較を、`sized_sentinel_for`はイテレータと番兵の引き算によって範囲の長さを求められることをそれぞれ定義しているコンセプトです。`requires`式内では*random access iterator*に通常期待される整数値との足し引きによる進行操作を定義しています。

そして、新しく加わったそれらの進行操作がどういう意味を持つのか、どう振る舞うのかを意味論要件が定義します。`+=`による進行は同じ数インクリメントしたのと同じであるとか、`+`と`+=`は進行という意味では同じであるとか、同様の事が`- -=`とデクリメントの間にも言えるだとか、イテレータの大小比較はイテレータの参照位置に基づくということなどを言っています。

C++17からは求められる事が若干厳しくなっており、C++17までの*random access iterator*はこのコンセプトを満たす事ができない場合があります。逆にC++20*random access iterator*はC++17*random access iterator*に対してほぼ後方互換性があります。

## `contiguous_range`

`contiguous_range`はそのイテレータが*contiguous iterator*であるような*range*を定義するコンセプトです。

```cpp
template<class T>
  concept contiguous_range =
    random_access_range<T> && contiguous_iterator<iterator_t<T>> &&
    requires(T& t) {
      { ranges::data(t) } -> same_as<add_pointer_t<range_reference_t<T>>>;
    };
```

型`T&`の値`t`について
- `to_address(​ranges​::​begin(t)) == ranges​::​data(t)`


`contiguous_range`は`random_access_iterator`でありそのイテレータが`contiguous_iterator`であることに加えて、`ranges::data(t)`という操作によってその範囲の存在するメモリ領域を指すポインタを取得することができます。そして、そのようなポインタ型はイテレータの要素型と一貫しており、取得されるポインタ値は先頭イテレータのアドレスと一致します。

*random access iterator*までは知っていると思われますが、*contiguous iterator*というものを初めて聞く人もいるかもしれません。これはイテレータの指す範囲がメモリ上で連続している場合のイテレータを表すものです（普通の配列におけるポインタなど）。概念だけがC++17で導入され、C++20でコンセプト導入とともに実体を持ちました。

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

## `common_range`

`common_range`はその先頭イテレータと終端を指すイテレータ（番兵）とが同じ型となる`range`を定義するコンセプトです。

```cpp
template<class T>
  concept common_range =
    range<T> && same_as<iterator_t<T>, sentinel_t<T>>;
```

意味論要件はなく、見たままです。

C++17まではイテレータに対してその範囲の終端を指すのは同じ型のイテレータだったので、当たり前のことを言っているように思えるかもしれません。

しかしC++20からはむしろ逆で、`ranges::begin`で得られる先頭イテレータと`ranges::end`で得られる終端イテレータは別の型となっている事が前提とされるようになっています。そして、終端を指すものの事を番兵（*Sentinel*）と呼びます。”もの”と言っているように、番兵はイテレータとして扱える必要すらありません。

C++17の世界ではあらゆるものが`common_range`でしたが、C++20の世界ではそうではなく、`common_range`は定義通りに単なる`range`よりも厳しい要求となっているのです。

なお、範囲`for`はC++17の時に`common_range`ではない`range`に対しても動作するように改修されています。

## `view`

Rangeライブラリの影の主役、`view`と呼ばれるものを定義しているのが、`view`コンセプトです。

```cpp
template<class T>
  concept view =
    range<T> &&
    movable<T> &&
    default_initializable<T> &&
    enable_view<T>;
```

- `T`のムーブ構築/代入は定数時間
- `T`のデストラクトは定数時間
- `T`はコピー不可であるか、`T`のコピー構築/代入は定数時間

構文的には、ムーブ構築・代入とデフォルト構築が可能な`range`です。これだけだと普通のコンテナも当てはまっていますが、`std::vector`が`view`であると思う人は多分居ないでしょう。

意味論要件は、コピーやムーブ、破棄などの操作の計算量が全て定数時間であることを言っており、`view`はその表現する範囲の要素数と無関係にコピーやムーブなどの基本操作が可能であることを意味しています。これはすなわち、`view`となる`range`はその表す範囲を所有せずに参照していることを意味しています。範囲を所有している場合、定数時間での破棄は不可能です。

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

意味論要件はなく、`viewable_range`コンセプトのキモは後半の`or`で繋がれた2つの制約式にあります。簡単には、この制約式は右辺値の`range`を弾くためにあります。

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

この様な恣意的な例以外でも、たとえば並列処理において大本の`range`が別のスレッドに所有されていたりする場合などにも起こり得ます。Rustならば型レベルでこのような問題を阻止できますが、C++ではこれを防ぐことはできません。コンパイラのコード分析や各種サニタイザなどを用いれば検出できるかもしれませんが・・・

# Rangeライブラリの構成部品

Rangeライブラリのメインではありませんが、Rangeライブラリを構成するために標準ライブラリに導入された各種部品となるものを見てみます。これらのものはイテレータや`range`を扱う時に利用可能なものです。

# Rangeアクセス関数

C++17においてコンテナアクセスの共通操作として`std`名前空間の下に追加されていたグローバル関数テンプレートに対応するものが、Rangeライブラリにも`std::ranges`名前空間の下に用意されています。これは一見すると同じものが重複して存在しているように見えてしまいますがそうではなく、コンセプトによって制約され、同じ意味を持ついくつかの操作をさらに統一化している、よりジェネリックで安全かつ簡単にその操作の目的を達成するものです。

ここではそれらのものをRangeアクセス関数として纏めて扱います。Rangeアクセス関数の実態は関数ではなく関数オブジェクトであり、カスタマイゼーションポイントオブジェクト（*Customization Point Object* : CPO）と呼ばれます。

カスタマイゼーションポイントオブジェクトはADLを阻害することで意図しない関数呼び出しを防止し、その内部で複雑な判定とディスパッチを行ってその目的を果たす非常にテクニカルなものです。

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

  ...
}
```

真にジェネリックに書くためにはこのように「`std::begin()/std::end()`を`using`してから、`begin()/end()`を名前空間修飾なしで呼び出す」という風に書くことで、`std`名前空間のもの及び配列には`std::begin()`が、ユーザー定義型に対してはADLで使用可能な`begin()`あるいは`std::begin()`を通してメンバ関数の`begin()`が呼び出されるようになります。しかし、手順1つ間違えただけでこのコードはたちまち汎用性を失います。

C++17まではイテレータペアをジェネリックに取得するにはこうする事がベストでした。しかし、このコードがなぜ必要なのかは難しい問題であり、理解せずに書いていると書き忘れたり消してしまったりしかねません。このようなよく使う操作に罠が潜んでいることは、C++諸学者にも優しくありません。

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

  // rangeコンセプトを満たす型だけが呼べるように制約してある新しいbegin()関数とする
  template<std::ranges::range C>
  constexpr auto begin(C& c) -> decltype(c.begin());  // (1)
}

namespace myns {

  struct my_struct {};

  // イテレータを取得するものではないbegin()関数
  bool begin(my_struct&);  // (2)
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

このように、大本の関数がコンセプトで制約されていたとしてもADL経由で制約を満たさない`begin()`が呼ばれています。別の見方をすれば、コンセプトによる制約を簡単に迂回できてしまっています。これはADLを用いる以上仕方のない事ではあるのですが、この場合にADLで呼び出される関数にまで制約を強制しようとすると少し込み入った実装が必要となります。また、コンセプトによる制約を行うことで、C++17との互換性の問題が発生しえます。

結局、ユーザーはカスタマイゼーションポイントの名前をきちんと意識してコードを書かなければならなくなってしまいます。

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

3番目の解決は少し複雑です。

C++における名前探索では修飾名探索と非修飾名探索の二つがまず実行され、そこで名前に対応するものが見つからない場合にADLが実行されます。ADLは関数と思われる名前に対しての名前探索方法で、一番最後に適用されます。ある名前が関数ではないことが分かっているとき、すなわちADLの前に名前に対応する関数ではないものが見つかっているとき、ADLは行われません。

カスタマイゼーションポイントオブジェクトは関数オブジェクトであり、その名前は変数名です。最初の2つの名前探索でCPOが発見されると、ADLは行われなくなります。これによって、CPOの呼び出しはADLによって迂回することができなくなり、その内部のあらゆるチェックは常に機能するようになります。

Rangeアクセス関数はCPOとして、従来のカスタマイゼーションポイント関数とは別に`std::ranges`名前空間に定義されています。従来のカスタマイゼーションポイント関数はそのままに新しく用意されているため互換性の問題もおこらず、CPOの利点を享受することができるようになっています。

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

  // 範囲の先頭イテレータ（番兵）を取得する
  auto vend = std::ranges::end(vec);
  auto aend = std::ranges::end(arr);
}
```

これだけだと`std::begin()/std::end()`と何が違うのかわかりませんが、その効果には大きな違いがあります。

型`T`のオブジェクト`r`によって`std::ranges::begin(r)`のように呼び出されたとき、その効果は次のいずれかになります

1. `r`が右辺値であり、かつ`enable_borrowed_range<remove_cv_t<T>> == false`の場合、*ill-formed*
2. `T`が配列型であり、`remove_all_extents_t<T>`が不完全型を示す場合、*ill-formed*
3. `T`が配列型なら、`r + 0`
      - 先頭のポインタを返す
4. `decay-copy(r.begin())`が呼び出し可能であり、その戻り値型が`input_or_output_iterator`コンセプトのモデルとなるなら、`decay-copy(r.begin())`
      - メンバ関数の`begin()`を呼び出す 
5. `T`はクラス型か列挙型であり、`decay-copy(begin(r))`が呼び出し可能であり、その戻り値型が`input_or_output_iterator`コンセプトのモデルとなるなら、`decay-copy(begin(r))`
      - ADLによって`begin()`を呼び出す
      - その際のオーバーロード解決は、`std`名前空間にある`std::begin()`を候補に含まないように行われる
6. それ以外の場合、*ill-formed*

`r`が右辺値であるとき、`r`を転送する所では`std::forward`を適切に使用したときと同様の完全転送が行われます。

とても複雑ですが、何か結果を返すのは3, 4, 5番目に該当する場合です。そこでは確実にイテレータを返すようになっており、`std::ranges::begin(r)`の呼び出しが有効であるときの戻り値型は`input_or_output_iterator`コンセプトのモデルとなります（`input_or_output_iterator`は何らかのイテレータとしての最小要件を定義するコンセプトです）。

そして、同じように`std::ranges::end(r)`と呼び出されたとき、その効果は次のいずれかになります

1. `r`が右辺値であり、かつ`enable_borrowed_range<remove_cv_t<T>> == false`の場合、*ill-formed*
2. `T`が配列型であり、`remove_all_extents_t<T>`が不完全型を示す場合、*ill-formed*
3. `T`が要素数不明の配列型である場合、*ill-formed*
4. `T`が配列型なら、`r + std::extent_v<T>`
      - 終端のポインタを返す
5. `decay-copy(r.end())`が呼び出し可能であり、その戻り値型`E`が`sentinel_for<E, iterator_t<T>>`コンセプトのモデルとなるなら、`decay-copy(r.end())`
      - メンバ関数の`end()`を呼び出す 
6. `T`はクラス型か列挙型であり、`decay-copy(end(r))`が呼び出し可能であり、その戻り値型`E`が`sentinel_for<E, iterator_t<T>>`コンセプトのモデルとなるなら、`decay-copy(end(r))`
      - ADLによって`end()`を呼び出す
      - その際のオーバーロード解決は、`std`名前空間にある`std::end()`を候補に含まないように行われる
7. それ以外の場合、*ill-formed*

こちらも`r`が右辺値であるとき、`r`を転送する所では`std::forward`を適切に使用したときと同様の完全転送が行われます。

ほとんど`ranges::begin()`と対になっており、`std::ranges::end(r)`の呼び出しが有効であるときの戻り値型`S`は、`std::ranges::begin(r)`の戻り値型を`I`として`sentinel_for<S, I>`コンセプトのモデルとなります（`sentinel_for<S, I>`はイテレータ型`I`に対する番兵型`S`を定義するコンセプトです）。

このように、`std::ranges::begin/std::ranges::end`はイテレータと番兵を様々な方法で取得してくるものです。その試行範囲は従来の`std::begin/std::end`よりも広く、そしてコンセプトによって確実かつ安全にイテレータと番兵を取得できるようになっています。

以降紹介するRangeアクセス関数もこれと同様、より探索範囲を広くしつつコンセプトによって確実に目的を達成できるように、従来あったものからリファインされています。C++20以降、特に後方互換の問題が無ければ、Rangeアクセス関数を優先的に使用することが推奨されます。

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

型`T`のオブジェクト`r`によって`ranges::cbegin(r)`のように呼び出されたとき、その効果は次のいずれかになります

1. `r`が左辺値ならば、`ranges::begin(static_cast<const T&>(r))`
2. それ以外の場合、`ranges::begin(static_cast<const T&&>(r))`

同じように`ranges::cend(r)`のように呼び出されたとき、その効果は次のいずれかになります

1. `r`が左辺値ならば、`ranges::end(static_cast<const T&>(r))`
2. それ以外の場合、`ranges::end(static_cast<const T&&>(r))`

どちらにおいても`r`が右辺値であるとき、`r`を転送する所では`std::forward`を適切に使用したときと同様の完全転送が行われます。

`ranges::cbegin(r)`の呼び出しが有効であるときの戻り値型`I`は`input_or_output_iterator`コンセプトのモデルとなり、`ranges::cend(r)`の呼び出しが有効であるときの戻り値型`S`は`sentinel_for<S, I>`コンセプトのモデルとなります。

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
3. `decay-copy(r.rbegin())`が呼び出し可能であり、その戻り値型が`input_or_output_iterator`コンセプトのモデルとなるなら、`decay-copy(r.rbegin())`
      - メンバ関数の`rbegin()`を呼び出す 
4. `T`はクラス型か列挙型であり、`decay-copy(rbegin(r))`が呼び出し可能であり、その戻り値型が`input_or_output_iterator`コンセプトのモデルとなるなら、`decay-copy(rbegin(r))`
      - ADLによって`rbegin()`を呼び出す
      - その際のオーバーロード解決は、`std`名前空間にある`std::rbegin()`を候補に含まないように行われる
5. `ranges::begin(r)`と`ranges::end(r)`が呼び出し可能であり、その戻り値型が同じ型で`bidirectional_iterator`のモデルであるなら、`make_reverse_iterator(ranges​::​end(r))`
6. それ以外の場合、*ill-formed*

そして、同じように`ranges::rend(r)`と呼び出されたとき、その効果は次のいずれかになります

1. `r`が右辺値であり、かつ`enable_borrowed_range<remove_cv_t<T>> == false`の場合、*ill-formed*
2. `T`が配列型であり、`remove_all_extents_t<T>`が不完全型を示す場合、*ill-formed*
3. `decay-copy(r.rend())`が呼び出し可能であり、その戻り値型`E`が`sentinel_for<E, decltype(ranges::rbegin(r))>`コンセプトのモデルとなるなら、`decay-copy(r.rend())`
      - メンバ関数の`rend()`を呼び出す 
4. `T`はクラス型か列挙型であり、`decay-copy(rend(r))`が呼び出し可能であり、その戻り値型`E`が`sentinel_for<E, decltype(ranges::rbegin(r)>`コンセプトのモデルとなるなら、`decay-copyr(end(r))`
      - ADLによって`rend()`を呼び出す
      - その際のオーバーロード解決は、`std`名前空間にある`std::rend()`を候補に含まないように行われる
5. `ranges::begin(r)`と`ranges::end(r)`が呼び出し可能であり、その戻り値型が同じ型で`bidirectional_iterator`のモデルであるなら、`make_reverse_iterator(ranges​::begin(r))`
6. それ以外の場合、*ill-formed*

どちらにおいても`r`が右辺値であるとき、`r`を転送する所では`std::forward`を適切に使用したときと同様の完全転送が行われます。

`ranges::rbegin(r)`の呼び出しが有効であるときの戻り値型`I`は`input_or_output_iterator`コンセプトのモデルとなり、`ranges::rend(r)`の呼び出しが有効であるときの戻り値型`S`は`sentinel_for<S, I>`コンセプトのモデルとなります。

基本的には`ranges::begin/ranges::end`とほとんど同じように`rbegin()/rend()`を探してきます。異なるのは配列型のケアを`ranges::begin/ranges::end`に委譲していることと、5番目のケースです。

5番目のケースはメンバに`rbegin/rend`を備えていないような`range`型に対するケアです。標準ライブラリではRangeライブラリにある各種`view`型が該当し、  それが`common_range`であり`bidirectional_range`であれば、そのイテレータペアと`std::make_reverse_iterator()`を利用して逆イテレータのペアを取得します。

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

`ranges::crbegin(r)`の呼び出しが有効であるときの戻り値型`I`は`input_or_output_iterator`コンセプトのモデルとなり、`ranges::crend(r)`の呼び出しが有効であるときの戻り値型`S`は`sentinel_for<S, I>`コンセプトのモデルとなります。

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
      - その際のオーバーロード解決は、`std`名前空間にある`std::size()`を候補に含まないように行われる
5. `to-unsigned-like(ranges::end(r) - ranges::begin(r))`が有効な式であり、`ranges::begin(r)`の戻り値型`I`と`ranges::end(r)`の戻り値型`S`が`sized_sentinel_for<S, I>`と`forward_iterator<I>`のモデルとなる場合、`to-unsigned-like(ranges::end(r) - ranges::begin(r))`
6. それ以外の場合、*ill-formed*

基本的には`ranges::begin/ranges::end`と同様になんとか長さを求めようとします。特殊なのは5番目の場合で、これは`size()`を持たないけれどイテレータの引き算によってその距離を求めることができるような範囲を想定しています。ちょうど、標準ライブラリにある`forward_list`が該当します。

5番目のケースが特にそうですが、`std::size`とは異なり`ranges::size`は計算量が定数時間であることを求めていません。範囲の長さを`N`として`O(N)`、あるいは呼び出される`size()`の実装によってはそれ以外の計算量となることがあり得、また許可されます。

### *integer-like*

`ranges::size(r)`の呼び出しが有効であるとき、その戻り値型は*integer-like*な型となります。実は、上記4,5のケースで呼ばれる`size()`の戻り値型も整数型ではなく*integer-like*な型であればOKです。

*integer-like*な型の定義は複雑ですが、簡単に言えば組み込みの整数型と殆ど同等に扱える型の事を指します。殆どの場合これは`int`型をはじめとする組み込みの整数型となるはずですが、`iota_view`など一部の`view`やユーザー定義の`range`では必ずしも整数型を返すとは限りません。たとえば、多倍長整数を返すことが許されます。

そして、`to-unsigned-like(i)`は規格書において説明のために用いられている操作で、`i`の型`X`と同じ幅を持つ符号なし整数型に変換する操作です。つまりは5番目のケースは必ず符号なしの整数型で結果が得られるということです。

## 範囲の領域のポインタ取得

```cpp
namespace std::ranges {
  inline namespace unspecified {
    inline constexpr unspecified data = unspecified;
    inline constexpr unspecified cdata = unspecified;
  }
}
```

# その他のRange CPO

分類としてはRangeアクセス関数ではありませんが、Rangeライブラリにおいて新しいカスタマイゼーションポイントとして追加されたCPOを見てみます。

## `ranges::swap`

```cpp
namespace std::ranges {
  inline namespace unspecified {
    inline constexpr unspecified swap = unspecified;
  }
}
```

## `ranges::iter_move`

```cpp
namespace std::ranges {
  inline namespace unspecified {
    inline constexpr unspecified iter_move = unspecified;
  }
}
```
## `ranges::iter_swap`

```cpp
namespace std::ranges {
  inline namespace unspecified {
    inline constexpr unspecified iter_swap = unspecified;
  }
}
```

# Range情報取得エイリアス

### `iterator_t/sentinel_t`
### `range_difference_t`
### `range_size_t`
### `range_value_t`
### `range_reference_t`
### `range_rvalue_reference_t`

# その他rangeユーティリティ
## `subrange`
## `view_interface`
## `dangling`

# Rangeファクトリ

## `empty_view`
## `single_view`
## `iota_view`
## `istream_view`

# Rangeアダプタ

## `all_view`
## `filter_view`
## `transform_view`
## `take_view`
## `take_while_view`
## `drop_view`
## `drop_while_view`
## `join_view`
## `split_view`
## `counted_view`
## `common_view`
## `reverse_view`
## `element_view`

# Rangeアルゴリズム

## 基本形
## 射影（*Projection*）
## ADLの無効化