---
title: C++23 ranges
author: onihusube
date: 2023/12/31
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

本書はC++23をベースとして記述されています。そのため、C++20までに導入されている機能については特に導入バージョンについて触れません。ただし、一部C++23に関する部分があり、それについてはC++23で変更・追加された機能であることを明示します。

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

# C++20 欠陥の修正

C++20における`<ranges>`、特にRangeアダプタまわりに関してはC++20規格完成後のC++23設計サイクル中にも多数の修正が行われ、その初期の時点ではまだ`<ranges>`の実装が無かったか出荷前だったことから、それらの修正は欠陥報告（*Defect Report* : DR）としてC++20に直接適用されました。

そのような修正がC++23設計サイクルの中盤くらいまで相次いだ結果、C++20の`<ranges>`とはどこまでのことを言うのは分かりづらくなっています。この章では、提案という形でまとまっている大きな修正についてまとめておきます。DRとなる修正は過去のバージョンに遡って適用されるため、ここにまとめられている修正は全てC++23ではなくC++20に対するものです。そのため、これらの修正を適用済みのコンパイラにおいてはC++20モード（`-std=c++20`など）でコンパイルした時にも修正後の挙動となります。ただし、過渡期のコンパイラ（標準ライブラリ実装）では修正が間に合っていない場合もあり、その場合は修正前の挙動となります。

C++20 ranges本でも執筆時点で把握していたものに関しては修正を適用していましたが、執筆後にもいくつかの修正が加えられたため完全ではありません。ここではその区別をせずに修正をまとめているため、一部の修正はC++20 ranges本で適用済みのものもあります。

## いくつかのアダプタの`borrowed_range`への対応

当初のRangeアダプタの一部のものには`enable_borrowed_range`の特殊化が用意されておらず、入力の`view`が`borrowed_range`であってもそれを継承できない場合がありました。それによって、それらのアダプタを適用した後の`view`が一部のRangeアルゴリズム（`borrowed_iterator_t`や`borrowed_subrange_t`を戻り値として使うもの）で使用できなくなっていました。

それを解消すべく、設計の大きな変更なしで`borrowed_range`となれるRangeアダプタについて、入力の`view`が`borrowed_range`ならばそれを継承するように修正されました。対象は次の6つのものです

- `views::take` (`take_view`)
- `views::drop` (`drop_view`)
- `views::drop_while` (`drop_while_view`)
- `views::common` (`common_view`)
- `views::reverse` (`reverse_view`)
- `views::elements` (`elements_view`)

これらのRangeアダプタの結果`view`型に対しては、入力`view`の`borrowed_range`性を受け継ぐように`enable_borrowed_range`の特殊化が用意されます。その実装はおおよそ次のようになります

```cpp{style=cppstddecl}
// take_viewにおける実装例
namespace std::ranges {

  template<view V>
  class take_view;

  // take_viewのenable_borrowed_range特殊化
  // 入力view型Tのenable_borrowed_range特殊化を利用する
  template<class T>
  inline constexpr bool enable_borrowed_range<take_view<T>> = enable_borrowed_range<T>; 
}
```

## `iterator_category`の提供基準の変更

当初のRangeアダプタの一部のものには`iterator_category`（イテレータ型と`iterator_traits`から取得される）を提供しないものがある一方で、`iterator_category`を提供するものもありました。`iterator_category`を提供する場合は入力`view`の`iterator_category`（そのイテレータ型を`I`とすると`iterator_traits<I>::iterator_category`）からそれを取得していたため、それらのアダプタが混ざってパイプで接続されると`iterator_category`を取得できないことからコンパイルエラーとなっていました。

```cpp
int main() {
  std::vector<int> vec = {0, 1, 2, 3, 4};

  auto r = vec | std::views::transform([](int c) { return std::views::single(c);})
               | std::views::join
               | std::views::filter(...); // 当初のC++20仕様ではng
}
```

この例の場合、`views::join`の結果`join_view`のイテレータは`iterator_category`を提供していない一方で、`views::filter`の結果`filter_view`のイテレータは常に`iterator_category`を提供しようとしまずが、`join_view`のイテレータからはそれが取得できないためコンパイルエラーとなっています。

`iterator_category`はC++17までのイテレータとしてイテレータカテゴリを表明するためのものであり、これはC++20イテレータコンセプトからは使用されずC++17のイテレータを扱うコードからのみ使用されます。C++20のイテレータはC++17イテレータとして使用可能である場合にのみ`iterator_category`を用意する（`iterator_traits`から取得できるようにする）ことで後方互換性を確保しています。後方互換性を確保する必要がない（できない）場合、`iterator_category`は用意されません。

このような仕組みが用意されていることから分かるように、C++20のイテレータ（イテレータコンセプトによって定義される）とC++17のイテレータ（名前付き要件によって定義される）の間には同じイテレータカテゴリ間でも互換性がない場合があります。特に、C++20の`input_iterator`はC++17基準だとイテレータとして扱えません。そのため、Rangeアダプタは結果の`view`が`input_range`となる場合には`iterator_category`を提供することは適切ではありません。

これらのことを踏まえつつも`iterator_category`があったり無かったりすることから生じる問題を回避するために、Rangeアダプタ（の結果`view`型）のイテレータ型における`iterator_category`は次のような基準で提供するようにされました

- 自身が`forward_range`とならない場合は`iterator_category`は定義されない
- `forward_range`である場合、自身の性質に応じて適切なカテゴリの`iterator_category`を定義する

`iterator_category`が実際にどのカテゴリになるのかはそのイテレータ型（とアダプタの入力`view`型）によります。

## `views::join`の*prvalue*対応

他の言語等で`flat_map`と呼ばれているもの（`range`の`range`を変換しながら平坦化する）を作成しようとすると、C++20では次のような実装が考えられます

```cpp
template<std::ranges::viewable_range R, std::invocable<std::ranges::range_reference_t<R>> F>
  requires std::ranges::range<std::ranges::range_value_t<R>> and
           std::ranges::range<std::invoke_result_t<F, std::ranges::range_reference_t<R>>>
auto flat_map(R&& r, F&& f) {
  return r | std::views::transform(f) // rの内側Rangeを変換し
           | std::views::join;        // 平坦化する
}
```

やってることはそのままですが、当初これは`f()`の戻り値が*prvalue*の`range`を返す場合にコンパイルエラーとなっていました。

当初の`views::join`が平坦化できたのは次の2つのタイプの`range`の`range`でした

- 左辺値`range`の、`range`
- *prvalue*`view`の、`range`

上記例の`f()`の戻り値が*prvalue*の`range`を返す場合はこのどちらにも当てはまらないためエラーとなります。

`views::join`の平坦化作業においては、まず外側`range`のイテレータを取得してそこから内側`range`のイテレータを取得します。この時、この内側`range`そのものは保持されておらずそのイテレータだけを保持しています。このため、内側`range`が*prvalue*だと内側イテレータの使用が安全ではなくなるのでそれを検出してコンパイルエラーにしていました。

この問題の解決のため、`views::join`は内側の`range`が`view`ではない*prvalue*の`range`の場合にキャッシュを使用して内側`range`を保持するように修正され、上記2つに加えて*prvalue*`range`の`range`を平坦化できるようにされました。これによって、先ほどの`flat_map`はほとんどの場合に機能するようになっています。

ただし、このキャッシュが使用される場合の`views::join`は`input_range`となり、`begin()`の呼び出しは1度しか行えません。また、内側`range`のキャッシュは`views::join`の結果の`view`オブジェクトのコピー/ムーブにおいて伝播しません。

## `view`とイテレータのデフォルト構築可能要求の削除

当初の`view`コンセプトは`std::default_initializable`を包摂しており、`view`の要件としてデフォルト構築可能であることを要求していました。同様に、`std::input_iterator`コンセプト（が包摂している`std::weakly_incrementable`コンセプト）も入力イテレータ（と出力イテレータ）に対してデフォルト構築可能であることを要求していました。

```cpp
// C++20当初のviewコンセプトの定義
template<class T>
concept view =
  range<T> &&
  movable<T> &&
  default_initializable<T> && // これ
  enable_view<T>;

// C++20当初のweakly_incrementableコンセプトの定義
template<class I>
concept weakly_incrementable =
  default_initializable<I> && // これ
  movable<I> &&
  requires(I i) {
    typename iter_difference_t<I>;
    requires is-signed-integer-like<iter_difference_t<I>>;
    { ++i } -> same_as<I&>;
    i++;
  };

// input_or_output_iteratorはweakly_incrementableを包摂する
template<class I>
concept input_or_output_iterator =
  requires(I i) {
    { *i } -> can-reference;
  } &&
  weakly_incrementable<I>; // これ
```

これは`view`/イテレータを実装する際に問題となり、実際にC++20のRangeアダプタの一部の`view`型ではこれを満足するために余計な複雑性を導入していました（`semiregular-box`の使用など）。

この要件は緩和され、`view`及び`input_iterator`/`output_iterator`はデフォルト構築可能である必要が無くなりました。また、これに伴って標準の`view`型やイテレータラッパの要件や実装も修正されています。

```cpp
// 修正後のviewコンセプトの定義
template<class T>
concept view =
  range<T> && 
  movable<T> &&
  enable_view<T>;

// 修正後のweakly_incrementableコンセプトの定義
template<class I>
concept weakly_incrementable =
  movable<I> &&
  requires(I i) {
    typename iter_difference_t<I>;
    requires is-signed-integer-like<iter_difference_t<I>>;
    { ++i } -> same_as<I&>;
    i++;
  };
```

どちらのコンセプトも意味論要件には変更がなく、この修正に関しては機能テストマクロ`__cpp_lib_ranges`が`201911L`から`202106L`にバンプアップされています。

## Rangeアダプタの引数受け取りの修正

当初の仕様では、Rangeアダプタオブジェクトが受け取った追加の引数（非`range`引数）をどのように保持するのかは未規定でした。それによって実装間の差異とダングリング参照の危険性発生させていました。

```cpp
template<class F>
auto filter(F f) {
  // GCC : fを参照で保持する
  // MSVC : fを常にコピーして保持する
  return std::views::filter(f);
}

int main() {
  std::vector<int> v = {1, 2, 3, 4};

  // 状態を持つ述語を渡す
  auto f = filter([i = std::vector{4}] (auto x) { return x == i[0]; });

  // GCCの場合、fにダングリング参照が含まれている
  // MSVCは無問題
  auto x = v | f;
}
```

このため、Rangeアダプタオブジェクトが追加の引数を受け取る際にすることは`std::bind_front`と同等になるように修正されました。どちらのものも、受け取った引数は返されたオブジェクト（Rangeアダプタクロージャオブジェクト）内部にコピーまたはムーブして保存され、参照で保持されることは無くなります。

また、そのように保存された値は、Rangeアダプタクロージャオブジェクトの使用（パイプラインへの投入）時のRangeアダプタクロージャオブジェクトの値カテゴリに従ってコピー/ムーブされて転送されます。

```cpp
// 入力のrangeとする
auto rng = ...;
// コピーコストが重い状態付きの巻数オブジェクトとする
auto f = ...;

rng | std::views::transform(f); // fをコピーして結果のviewにムーブする

auto t = std::views::transform(f);  // fをコピーする
rng | t;            // tから再度fをコピーする
rng | std::move(t); // tからfをムーブする
```

## `views::split`と`views::lazy_split`

当初の`views::split`（`split_view`）は文字列分割に特化したものではなく、より一般的な`range`を`range`によって分割することができるものでした。そのため、主たる用途である文字列分割でむしろ使いづらくなっている部分がありました。

```cpp
int main() {
  using namespace std::string_view_literals;
  using namespace std::views;

  const auto str = "split_view takes a view and a delimiter"sv;

  for (auto strv : str | split(' ')
                       | transform([](auto subrng) {
                           // どちらもできない・・・
                           return std::string_view{subrng};
                           return std::string_view{subrng.begin(), subrng.end()};
                         })
  ) {
    std::cout << strv << '\n';
  }
}
```

この例では、`views::split`によって入力文字列からホワイトスペースを区切りとして切り出した部分文字列を`views::transform`によって`string_view`に変換しようとしています。しかし、当初の`views::split`の分割後の部分範囲（内側`range`）は`forward_range`でしかなく`std::string_view`にはそのまま変換可能ではありません。ただし、遅延評価によって内側`range`の参照する各要素（この例では部分文字列）は入力の`range`のものを参照しておりコピーされたりしてはいません。そのため、それを`std::string_view`で参照しようとするのはとても自然な発想でしょう。

これはまた、この例のように`views::split`の後段の`views::transform`内で文字列を対象とした他の操作（`std::from_chars`や`std::regex`など）を行おうとした時にもそれを妨げます。`std::regex`等標準ライブラリ内で文字列を受け取るところでは少なくとも`bidirectional_iterator`でなければならず、`std::from_chars`のようにより厳しいところではポインタでなければなりません。

しかし当初の`views::split`の出力の内側`range`は最も強くても`forward_range`にしかならず、それを文字列として扱おうとすると非自明で面倒な変換を行わなければなりません。

この問題の解消のため、文字列分割に特化したRangeアダプタとして`views::split`が追加され、当初の`views::split`は`views::lazy_split`（`lazy_split_view`）にリネームされました。`views::split`の出力の内側`range`は入力文字列のイテレータをそのまま利用した`ranges::subrange`となり、これは`contiguous_range`となるためかなり扱いやすくなります。

```cpp
int main() {
  using namespace std::string_view_literals;
  using namespace std::views;

  const auto str = "split_view takes a view and a delimiter"sv;

  for (auto strv : str | split(' ')
                       | transform([](auto subrng) {
                           return std::string_view{subrng.begin(), subrng.end()}; // ok
                         }))
  {
    std::cout << strv << '\n';
  }
}
```

当初の`views::split`の提供していたより汎用的な`range`の分割が行いたい場合は`views::lazy_split`を利用します。

## `istream_view`の修正

当初の`istream_view`は`ranges::basic_istream_view`という`view`クラス型とそれを生成するための`ranges::istream_view`という関数テンプレートとして用意されていました。使用する際は`ranges::istream_view<T>(istrm)`のように`T`に読み込むデータ型、`istrm`に読み込み元の入力ストリームオブジェクトを渡します。

```cpp
int main() {
  // 標準入力にからint値を1つづつ読み込む
  for (int n : std::ranges::istream_view<int>(std::cin)) {
    std::cout << n;
  }
}
```

他のRangeファクトリおよびRangeアダプタが`ranges::xxx_view`に対して`views::xxx`というRangeアダプタ/ファクトリオブジェクトを用意しているのに対して、このAPIは微妙に異なっていたため他の`view`の使用法から類推される使用法が`istream_view`には通用しませんでした。

```cpp
int main() {
  std::istringstream mystream{"0 1 2 3 4"};

  // istream_viewはクラスではなく関数
  std::ranges::istream_view<int> v{mystream}; // ng

  // 関数なので{}を使用できない
  for (int n : std::ranges::istream_view<int>{std::cin}) {  // ng
    std::cout << n;
  }

  // views::istreamはない
  for (int n : std::views::istream<int>(std::cin)) {  // ng
    std::cout << n;
  }
}
```

C++20当初の`istream_view`のAPI構造は次のようになっていました

```cpp{style=cppstddecl}
namespace std::ranges {

  // basic_istream_viewクラス
  template<movable Val, class CharT, class Traits>
    requires default_initializable<Val> && stream-extractable<Val, CharT, Traits>
  class basic_istream_view : public view_interface<basic_istream_view<Val, CharT, Traits>>;

  // ranges::istrem_view関数
  template<class Val, class CharT, class Traits>
  basic_istream_view<Val, CharT, Traits> istream_view(basic_istream<CharT, Traits>& s);
}
```

この`istream_view`のAPI構造を他の`view`と一貫させるために、最終的にAPIは次のように変更されました

```cpp{style=cppstddecl}
namespace std::ranges {

  // basic_istream_viewクラスはそのまま
  template<movable Val, class CharT, class Traits>
    requires default_initializable<Val> && stream-extractable<Val, CharT, Traits>
  class basic_istream_view : public view_interface<basic_istream_view<Val, CharT, Traits>>;

  // charとwchar_tの型エイリアスを追加
  // ranges::istrem_viewは型名になった
  template<class Val> 
  using istream_view = basic_istream_view<Val, char>;

  template<class Val> 
  using wistream_view = basic_istream_view<Val, wchar_t>; 

  namespace views {

    // views::istream<T>を追加
    template<typename T>
    inline constexpr /*unspecified*/ istream = /*unspecified*/;
  }
}
```

これによって、`ranges::istream_view`に対して`views::istream`が存在するという他の`view`と同様の命名規則となり、他の`view`のAPIと一貫した利用法ができるようになりました。

```cpp
int main() {
  std::istringstream mystream{"0 1 2 3 4"};

  std::ranges::istream_view<int> v{mystream}; // ok

  for (int n : std::ranges::istream_view<int>{std::cin}) {  // ok
    std::cout << n;
  }

  for (int n : std::views::istream<int>(std::cin)) {  // ok
    std::cout << n;
  }
}
```

## `view`コンセプトの意味論要件変更と`owning_view`

入力の型を`T`として、当初の`view`コンセプトの意味論要件は次のようなものでした

- `T`のムーブ構築/代入は定数時間（`O(1)`）
- `T`のデストラクトは定数時間
- `T`はコピー不可であるか、`T`のコピー構築/代入は定数時間

これらの要件は`view`が範囲を所有しない軽量な型であることを表現しています。

このうち、一部の`view`型では2つ目のデストラクタ呼び出しが定数時間という要件が満たせない場合があることがわかりました（`std::generator`がそれに該当していました）。そのため、`view`コンセプトの意味論要件が再考され、次のように修正されました

- `T`のムーブ構築は定数時間（`O(1)`）
- `T`のムーブ代入の計算量は、デストラクタとムーブ構築を続けて実行した時の計算量よりも複雑ではない
- `M`個の要素を含む`T`のオブジェクトが`N`回コピー/ムーブされた時、それら`N`個のオブジェクトのデストラクタの計算量は`O(N+M)`
    - ムーブ後オブジェクトの破棄は定数時間
- `T`はコピー構築不可であるか、`T`のコピー構築は定数時間
- `T`はコピー代入不可であるか、`T`のコピー代入の計算量は、デストラクタとコピー構築を続けて実行した時の計算量よりも複雑ではない

複雑になりましたが`view`が軽量の型であることを表現しているのは変化しておらず、主にムーブやコピーした際に残った元のオブジェクトのことを考慮に入れるようになっています。

以前に`view`だったものはこの要件の下でも`view`であり続けますが、この要件の下では`view`は範囲を所有することができるようになっています。ただし、範囲を所有している`view`であってもそれは通常のコンテナのように振る舞うのではなく、コピーはその要素をコピーするものであってはならないことを要求しており、ムーブによって所有している範囲の所有権は別のオブジェクトへ移行しなければなりません。

そして、この`view`コンセプトの変更によって許可された範囲を所有するタイプの`view`として`ranges::owning_view`が追加されました。`owning_view`は渡された右辺値のRangeを所有してその寿命を延長させる`view`で、`views::all`に右辺値のRangeを渡したときに`owning_view`にラップされて返されます。これによって、パイプラインにおいて右辺値のRangeの取り扱いが安全になりました。

```cpp
using namespace std::views;
using namespace std::ranges::view;

// 右辺値の範囲を返す
auto rv() -> std::vector<int>;
// 左辺値の範囲を返す
auto lv() -> std::vector<int>&;

int main() {
  // パイプライン処理の事前組み立て
  auto pipe = drop(2)
            | filter(...)
            | transform(...)
            | take(5);
  
  // rv()の戻り値はowning_viewによって保存されている
  view auto rvpipe = rv() | pipe;
  // lv()の戻り値はref_viewによって保存されている
  view auto lvpipe = lv() | pipe;

  // rv()の戻り値はrvpipe内部で生存している
  for(auto n : rvpipe) {
    ...
  }

  // lv()の戻り値の参照元が生存していれば、安全
  for(auto n : lvpipe) {
    ...
  }
}
```

この例の場合、`lv()`の戻り値の参照元の寿命が十分に長いとすると、どちらのパイプライン処理も安全に実行可能となります。また、`views::all`はパイプラインにおいて自動適用されるため、`owning_view`や`ref_view`の存在をユーザーは気にする必要がなく、おそらく直接使用することはほとんど無いでしょう。

# Rangeアダプタ/ファクトリ

Rangeライブラリの主役であるRangeアダプタ/RangeファクトリはC++23でさらに拡充されており、パイプラインによるより多彩な範囲操作が可能になっています。

## `views::repeat`

`views::repeat`は与えられた値を指定された回数の繰り返しによる範囲を生成する`view`です。これは、Rangeに対して作用するRangeアダプタではなく範囲を生成するRangeファクトリです。

```cpp
int main() {
  for (int n : std::views::repeat(23) | std::views::take(4)) {
    std::cout << std::format("{:d}, ", n);
  }
}
```
```{style=planetext}
23, 23, 23, 23, 
```

`views::repeat(value)`のように呼び出すと、`value`を要素とする無限列が生成されます。`views::repeat(value, N)`のように、2つ目の引数に整数値を指定して要素数を指定することもできます。

```cpp
int main() {
  for (int n : std::views::repeat(23, 4)) {
    std::cout << std::format("{:d}, ", n);
  }
}
```
```{style=planetext}
23, 23, 23, 23, 
```

`views::repeat`そのものはRangeファクトリオブジェクトであり、1つか2つの引数を受けてそれをそのまま転送して`repeat_view`を構築して返します。

### `repeat_view`

`repeat_view`は`views::repeat`の実装詳細であり、`views::repeat`は常にこの`view`型を返します。

```cpp
namespace std::ranges {

  // repeat_viewの宣言例
  template<move_constructible T, semiregular Bound = unreachable_sentinel_t>
    requires (is_object_v<T> && same_as<T, remove_cv_t<T>> &&
              (integer-like-with-usable-difference-type<Bound> ||
               same_as<Bound, unreachable_sentinel_t>))
  class repeat_view : public view_interface<repeat_view<T, Bound>> {
    ...
  };
}
```

要素型である`T`はオブジェクト型でありCV修飾がないことの他には`move_constructible`であることだけが求められています。`Bound`は要素数を抑える型で、デフォルト時は`unreachable_sentinel`が使用され、要素数を指定する場合はほぼ整数型を求めています。

要素数を指定しない場合（`Bound = unreachable_sentinel_t`の場合）には`.end()`から得られる番兵として`unreachable_sentinel`が使用され、`unreachable_sentinel`は任意のイテレータとの比較に常に`true`を返す番兵であるため、イテレータの終端判定を最適化によって除去しやすくなっています。ただし、これによって`.begin()`と`.end()`の型が異なってしまうため、`common_range`ではなくなります（要素数を指定する場合は同じイテレータ型が使用されます）。

要素数が指定されている（`Bound`が`unreachable_sentinel_t`ではない）場合にのみ、`repeat_view`は`sized_range`かつ`common_range`になります。

`repeat_view`はその内部に`T`の値（要素）を保存していて、`repeat_view`のコンストラクタは渡された`T`の左辺値をコピーもしくは`T`の右辺値をムーブすることでそのただ一つの要素を初期化します。

### 動作詳細

`repeat_view`の生成する範囲は、当然のことながら、その構築時にその要素の全てがどこかの領域に敷き詰められて保持されているものではなく、実際には`repeat_view`が保存している同じ1つの値を異なるイテレータが参照することによって構成されています。

そのため、そのイテレータの間接参照で得られるものは`repeat_view`に渡されて内部で保持されている`T`の値の`const`左辺値参照（`const T&`）となります。これによって、`repeat_view`はその要素としてムーブオンリーな型の値からなる範囲を生成することができます。

```cpp
int main() {
  for (int n : std::views::repeat(23, 4)) {
    std::cout << std::format("{:p}, ", &n);
  }
  // 全て同じアドレスが出力される

  std::ranges::repeat_view rv{std::make_unique<int>(23), 4};  // ok

  for (const auto& up : rv) {
    std::cout << std::format("{:p}, ", &up);
  }
  // 全て同じアドレスが出力される
}
```

また、この性質によって`repeat_view`は要素型などに関係なく常に`random_access_range`となります。`contiguous_range`ではないのは、イテレータの進行と要素のメモリ配置が同期しない（動かない）ためです。要素数を指定しない場合でも`repeat_view`は`random_access_range`であるため、その場合の`repeat_view`は進行/後退の両方向について無限列となります。

### `views::take`と`views::drop`の特殊対応

`repeat_view`は内部に格納している値をその範囲の全ての要素としていることと`repeat_view`そのものに要素数を指定可能であることから、指定された個数の要素を切り出す/スキップするような処理はかなり単純化できる可能性があり、実際に`views::take`と`views::drop`ではそれが行われています。

`views::take`の場合は、`views::take(r, n)`のように呼ばれた時に、`r`の素の型`R`が`repeat_view`であれば、次のように`repeat_view`を調整して返します

- `R`が`sized_range`である（要素数が指定されている）場合
    - `r`の内包する値を`value`、`ranges::distance(r)`と`n`の短い方を`l`として
    - `views::repeat(value, l)`を返す
- `repeat_view`の要素数が指定されていない場合
    - `r`の内包する値を`value`として
    - `views::repeat(value, n)`を返す

`views::drop`の場合は、`views::drop(r, n)`のように呼ばれた時に、`r`の素の型`R`が`repeat_view`であれば、次のように`repeat_view`を調整して返します

- `R`が`sized_range`である（要素数が指定されている）場合
    - `r`の内包する値を`value`、`ranges::distance(r)`を`m`、`m`と`n`の短い方を`l`として
    - `views::repeat(value, m - l)`を返す
- `repeat_view`の要素数が指定されていない場合
    - `r`をdecay-copyして返す

どの場合でも、元の`repeat_view`オブジェクト`r`から取り出された値（`value`）は`r`の値カテゴリによって適切にムーブorコピーされます。

```cpp
int main() {
  auto inf_rep = std::views::repeat(23);
  
  // 無限`repeat_view`に対するtake/drop
  auto t1 = inf_rep | std::views::take(10);
  // t1 = std::views::repeat(23, 10) と等価
  auto d1 = inf_rep | std::views::drop(10);
  // d1 = auto(inf_rep) と等価
  
  auto fin_rep = std::views::repeat(23, 10);
  
  // 有限`repeat_view`に対するtake/drop、元の長さに収まる場合
  auto t2 = fin_rep | std::views::take(5);
  // t2 = std::views::repeat(23, 5) と等価
  auto d2 = fin_rep | std::views::drop(5);
  // d2 = std::views::repeat(23, 10 - 5) と等価
  
  // 有限`repeat_view`に対するtake/drop、元の長さを超える場合
  auto t3 = fin_rep | std::views::take(15);
  // t3 = std::views::repeat(23, 10) と等価
  auto d2 = fin_rep | std::views::drop(15);
  // d3 = std::views::repeat(23, 10 - 10) と等価
}
```

### `views::repeat`(`repeat_view`)の諸特性

- `reference` : `const T&`
- `range`カテゴリ : `random_access_range`
- `common_range` : 要素数を指定した（有限長の）場合
- `sized_range` : 要素数を指定した（有限長の）場合
- `const-iterable` : 〇
- `borrowed_range` : ×

これらはそれぞれ次のような意味です

- `reference` : イテレータの関節参照の結果型
    - `ranges::range_reference_t<R>`の型
- `range`カテゴリ : モデルとなる最も強い`range`コンセプト
- `common_range` : `begin()`の戻り値型と`end()`の戻り値型が同じであるか
    - `common_range`コンセプトのモデルとなるか
- `sized_range` : `ranges::size()`によって定数時間で要素数を得られる
    - `sized_range`コンセプトのモデルとなるか
- `const-iterable` : `const`修飾されているときでも`range`となるか（要素型が`const`であることを意味しない）
- `borrowed_range` : 範囲の寿命とそこから取得したイテレータの寿命が切り離されている
    - `borrowed_range`コンセプトのモデルとなるか

〇は常にその性質が有効であることを、×は常に無効であることを、特定の型やコンセプトが指定されている場合は常にそれを示すことを、条件が指定されている場合はその条件を満たす場合にその性質が有効になることを、それぞれ表しています。

　

C++23で追加されたRangeファクトリはこの`views::repeat`のみなので、以降のものは全てRangeアダプタになります。

## `views::as_rvalue`

`views::as_rvalue`は、入力範囲の各要素を`std::move()`した右辺値の要素からなる範囲となる`view`です。

```cpp
int main() {
  std::vector<std::string> strvec = { "move_view", "all_move_view", "as_rvalue_view" };
  
  for (std::string&& rv : strvec | std::views::as_rvalue) {
    std::cout << std::format("{:s}", rv);
  }
}
```
```{style=planetext}
move_view
all_move_view
as_rvalue_view
```

このサンプルコードはほぼ意味がなく、これはC++23で一緒に導入された`ranges::to`とともに使用されることを意図しています。

```cpp
int main() {
  std::vector<std::unique_ptr<int>> upvec;
  upvec.emplace_back(std::make_unique<int>(10));
  upvec.emplace_back(std::make_unique<int>(100));
  upvec.emplace_back(std::make_unique<int>(17));
  
  // listに詰め替え
  auto up_list = upvec | std::views::as_rvlaue
                       | std::ranges::to<std::list>;  // ok

  // upvecの要素はunique_ptrの左辺値であるため、これだとコピーしようとしてエラー
  auto up_list = upvec | std::ranges::to<std::list>;  // ng  
}
```

`ranges::to`は`range`から`range`への変換を行うもので、詳しくは後の章で解説しています。

また、同様の事をRangeアルゴリズムでやる場合にも使用できます。

```cpp
int main() {
  std::vector<std::unique_ptr<int>> upvec;
  upvec.emplace_back(std::make_unique<int>(10));
  upvec.emplace_back(std::make_unique<int>(100));
  upvec.emplace_back(std::make_unique<int>(17));
  
  // listに詰め替え
  std::list<std::unique_ptr<int>> uplist;
  std::ranges::copy(upvec | std::views::as_rvalue, std::back_inserter(uplist));
}
```

`views::as_rvalue`そのものはRangeアダプタオブジェクトであり、入力範囲`r`（型`R`とする）を受けて次のどちらかの動作をします

- `r`の右辺値参照型（`range_rvalue_reference_t<R>`）と`r`の参照型（`range_reference_t<R>`）が同じ型である場合 : `views::all(r)`を返す
- そうではない場合 : `r`をそのまま転送して`as_rvalue_view`を構築して返す

`views::as_rvalue`は入力範囲の要素の値カテゴリを見て、それが既に右辺値の場合は何もせず、左辺値である場合にのみ`as_rvalue_view`を返します。これによって、既に右辺値の範囲となっている入力範囲に対して要素ごとに`std::move`をかけていくことを回避しており、特に間接参照結果が*prvalue*になっている範囲に対して使用した場合にコピー省略を妨げないようになっています。

### `as_rvalue_view`

`views::as_rvalue`に左辺値を要素とする範囲を入力する場合は`as_rvalue_view`が返されます。

```cpp
namespace std::ranges {

  // as_rvalue_viewの宣言例
  template<view V>
    requires input_range<V>
  class as_rvalue_view : public view_interface<as_rvalue_view<V>> {
    ...
  };
}
```

`as_rvalue_view`は入力範囲を`views::all``views::all`に通したものを保持しています。

### 動作詳細

`as_rvalue_view`は入力範囲のイテレータを`std::move_iterator`にラップして返すことでその要素の右辺値化を行います。そのため、動作はすべて`std::move_iterator`が定義するところによります。

C++17までの（C++17イテレータとしての）`std::move_iterator<I>`はラップするイテレータ`I`の性質をなるべく再現しようとしていました。しかし、`std::move_iterator<I>`の各要素は対応する`I`のイテレータの要素をムーブしたものであり、同じ要素への2回目以降のアクセスは安全ではないため、`forward_iterator`以上になることは適切ではありません。

そのため、C++20からの（C++20イテレータとしての）`std::move_iterator<I>`は`I`にかかわらず常に`input_iterator`となります。これを受けて、`as_rvalue_view`も常に`input_range`となります。

```cpp
// 入力範囲の保存or参照のみを行う
auto arv = std::views::as_rvalue(input);

// 入力のイテレータをmove_iteratorにラップして返す
auto it = arv.begin();

// 進行は保持するイテレータに同じ操作を適用する
++it;

// 保持するイテレータの間接参照結果をmoveして返す
auto&& rvalue = *it;

// 保持するイテレータ同士の比較が行われる
bool b = it == arv.end();
```

入力の範囲（`input`）が右辺値だったとしても入力範囲は`as_rvalue_view`内部で保存されているため、`as_rvalue_view`から取得されるイテレータは元の`as_rvalue_view`オブジェクトが生存している限り安全に使用できます。これは、`views::as_rvalue`が`std::move_iterator`と`ranges::subrange`によって実装されていない理由でもあります（そのような実装では右辺値範囲の入力がダングリングイテレータを生成します）。

### `as_rvalue_view`(`views::as_rvalue`)の諸特性

入力の範囲を`R`とすると次のようになります

- `reference` : `range_rvalue_reference_t<R>`
- `range`カテゴリ : `input_range`
- `common_range` : `R`が`common_range`の場合
- `sized_range` : `R`が`sized_range`の場合
- `const-iterable` : `R`が`const-iterable`の場合
- `borrowed_range` : `R`が`borrowed_range`の場合

## `views::join_with`

`views::join_with`は指定されたパターンによって接合しながら入力の`range`の`range`を平坦化する`view`です。

```cpp
int main() {
  std::vector<std::string> strvec = {"https:", "", "eel.is", "c++draft", "range.join.with"};

  for (char c : strvec | std::views::join_with('/')) {
    std::cout << std::format("{:c}", c);
  }
}
```
```{style=planetext}
https://eel.is/c++draft/range.join.with
```

`views::join`による平坦化に際して、入力範囲の内側`range`間に指定されたパターンを挿入しながら平坦化を行います。

`views::join_with`への接合パターンの渡し方には2種類あり、1つは上記例のように入力範囲の内側`range`の値型（`value_type`）と同じ型のオブジェクトを1つだけ指定して、その値を挿入しながら`join`を行うものです。もう1つは任意の`forward_range`オブジェクトを渡して、範囲を挿入しながら`join`を行うものです。

```cpp
int main() {
  std::vector<std::list<int>> listvec = {{1, 2}, {3, 4, 5}, {6, 7}, {8}};

  // 1. 入力の内側の要素型の値を渡す
  for (int n : listvec | std::views::join_with(0)) {
    std::cout << std::format("{:d}", n);
  }

  std::forward_list pattern = {0, 0, 0};

  // 2. 任意のforward_rangeを渡す
  for (int n : listvec | std::views::join_with(pattern)) {
    std::cout << std::format("{:d}", n);
  }
}
```
```{style=planetext}
12034506708
12000345000670008
```

一応、接合パターンには空の範囲を渡すこともでき、その場合は`views::join`と同じ動作をしますが多少非効率になります。

動作だけを見れば、`views::join_with(p)`は`views::split(p)`（あるいは`views::lazy_split`）の逆変換となっており、`input | views::split(p) | views::join_with(p)`は元の範囲`input`と同じシーケンスになります。ただし、特に特別扱いはされていないため結果の型は異なります。

`views::join_with`そのものはRangeアダプタオブジェクトであり、2つの引数を受け取ってそれをそのまま転送して`join_with_view`を構築して返します。

### `join_with_view`

`join_with_view`は`views::join_with`の実装詳細であり、`views::join_with`は常にこの`view`型を返します。

```cpp
namespace std::ranges {

  // join_with_viewの宣言例
  template<input_range V, forward_range Pattern>
    requires view<V> && input_range<range_reference_t<V>>
          && view<Pattern>
          && compatible-joinable-ranges<range_reference_t<V>, Pattern>
  class join_with_view {
    ...
  };
}
```

入力範囲`V`およびその内側の`range`は`input_range`であることしか求められていない一方で、パターンの範囲`Pattern`は`forward_range`でなければなりません。これは、パターンは複数回参照されるためマルチパス保証が必要となるためです。

`compatible-joinable-ranges`は入力範囲とパターンの要素型/参照型との間で共通の型（共通参照型）が存在することを要求するものです。

この宣言だけを見ると、接合パターンとしては常に`forward_range`が求められており、値1つを渡すパターンの渡し方ができなさそうに思えます。これは`join_with_view`のテンプレートパラメータ推論補助を工夫することで受け入れるようになっています。

```cpp
// forward_rangeなパターンが渡された時のための推論補助
template<class R, class P>
join_with_view(R&&, P&&) -> join_with_view<views::all_t<R>, views::all_t<P>>;

// 単一の値が渡された時のための推論補助
template<input_range R>
join_with_view(R&&, range_value_t<range_reference_t<R>>)
  -> join_with_view<views::all_t<R>, single_view<range_value_t<range_reference_t<R>>>>;
```

1つ目の推論補助は入力範囲とパターン範囲の両方を`views::all`に通すもので、他のRangeアダプタの`view`型でも多用されているものです。

2つ目の推論補助が`forward_range`ではない値1つを渡されたときに使用されるもので、渡された値の型を`views::single`でラップして`join_with_view`のパターン型を求めています。これによって、パターンは常に`forward_range`な範囲として一貫して扱われています。見てわかるように、この場合には入力範囲の内側`range`の値型に一致する型だけを受け入れます。

`join_with_view`はこのように渡された2つの範囲を内部で保存しており、内側`range`が*prvalue*となる（左辺値ではない）場合はさらに内側`range`もキャッシュしています。内側`range`のキャッシュが行われる場合、`join_with_view`は`input_range`になり、`.begin()`の呼び出しは最初の一回だけが有効になります。

また、外側`range`（`V`）が`forward_range`ではない場合もそのイテレータを内部にキャッシュします。この場合も`join_with_view`は`input_range`になります。

### 動作詳細

`join_with_view`の構築時は入力範囲とパターンを保持し、キャッシュが必要となる場合は各種キャッシュ領域を用意しますが、この時点ではほぼ何もしません。

`join_with_view`のイテレータが取得されるとき、パターンのイテレータと入力範囲の外側イテレータを取得しそこから内側`range`と内側イテレータを取得するとともに、`join_with_view`内の必要なキャッシュを初期化します。その後、内側イテレータ（およびパターンのイテレータ）の終端判定を行いながらまず最初の要素まで管理するイテレータを進行させます。

`join_with_view`内では、外側のイテレータと内側イテレータおよびパターンのイテレータの3つを管理していますが、同時に保持しているのは外側のイテレータと内側イテレータもしくはパターンのイテレータのどちらかのみです。後者は、内側イテレータとパターンのイテレータをそれぞれ`II, PI`とすると`std::variant<PI, II>`によって保持されています。その`std::variant<PI, II>`オブジェクトは、現在位置が内側`range`内にあるときは内側イテレータを、パターン内にあるときはパターンのイテレータを保持するように管理されます。

`join_with_view`のイテレータの進行時は、`std::variant<PI, II>`の現在のイテレータを1つ進めた後、その終端チェックを行います。終端チェック時では、内側イテレータが終端に達しているときは外側イテレータを1つ進めて`std::variant<PI, II>`をパターンのイテレータの先頭で更新し、そのパターンのイテレータの終端チェックを再び行います。パターンのイテレータが終端に達しているときは内側`range`と内側イテレータを更新し、内側イテレータの終端チェックを再び行います。

`join_with_view`が`bidirectional_range`となっている場合、後退（`--`）時はこの逆のことをします。まず外側イテレータが終端に達しているなら内側`range`の一番最後の要素の`end`イテレータを取得し`std::variant<PI, II>`に保存します。その後、`std::variant<PI, II>`のイテレータを取得し、その終端チェックを行いながら空の範囲のスキップとイテレータの切り替えを行います。最後に、`std::variant<PI, II>`の現在のイテレータを1つ後退させます。

イテレータの終端判定においては、外側イテレータが終端に達しているときに終端だと判定されます。これは、上記の進行時に外側イテレータが先に終端へ到達するようになっているためです。


```cpp
// 外側rangeがforwardではなければ、そのイテレータのキャッシュ領域を提供する
// 内側rangeがprvalueとなる場合、内側rangeのキャッシュ領域を提供する
auto jwv = input | std::views::join_with(pattern);

auto it = jwv.begin();

// 現在のイテレータ（内側orパターン）を進行させ、都度終端チェックを行いながら切り替えていく 
// 保持する外側イテレータや内側rangeやそれらのキャッシュもその際に更新される
++it;
// 後退時には、先に終端チェックを行い空の部分を飛ばして切り替えを完了させて
// その後現在のイテレータを後退させる
// この場合キャッシュは一切使用されていない
--it;

// 現在のイテレータ（内側orパターン）の関節参照結果をそのまま返す
auto elem = *it;

// 外側イテレータの比較によってのみ終端が判定され、パターンのイテレータは終端判定に関与しない
// ただし、イテレータそのものの比較には内側イテレータも参加する
bool b = it == jwv.end();
```

パターンの範囲とそのイテレータの管理が追加されてはいますが、基本的には`join_view`と同じような方法で平坦化をおこなっています。

### `join_with_view`(`views::join_with`)の諸特性

入力範囲を`R`、その内側の`range`を`IR`、接合パターンを`P`とすると次のようになります

- `reference` : `common_reference_t<range_reference_t<IR>, range_reference_t<P>>`
- `range`カテゴリ
    - `range_reference_t<R>`が参照型であり、かつ
        - `R`が`bidirectional_range`であり、`IR`と`P`が共に`bidirectional_range`かつ`common_range`の場合 : `bidirectional_range`
        - `R`と`IR`が共に`forward_range`の場合 : `forward_range`
    - それ以外の場合 : `input_range`
- `common_range` : `range_reference_t<R>`が参照型であり、`R`と`IR`が共に`forward_range`かつ`common_range`の場合
- `sized_range` : ×
- `const-iterable` : `R`と`P`が共に`const-iterable`であり、`range_reference_t<const R>`が参照型であり`input_range`である場合
- `borrowed_range` : ×

接合パターンが関わってはきますが、ほぼ`views::join`と同じ性質となります。

「`range_reference_t<R>`が参照型であり」のような要求は内側`range`が左辺値であることを要求しており、そうではなく内側`range`が*prvalue*の場合は`join_with_view`内に内側`range`が都度キャッシュされます。そのためその場合は`input_range`になりその他の性質も制限されます。

## `views::as_cosnt`
## `views::enumerate`

`views::enumerate`は入力範囲の各要素にそのインデックスを紐付けた要素からなる範囲を生成する`view`です。

```cpp
int main() {
  std::vector vec = {1, 3, 5, 7, 11, 13};

  for (auto [index, value] : vec | std::views::enumerate) {
    std::cout << std::format("{:d}: {:d}\n", index, value);
  }
}
```
```{style=planetext}
0: 1
1: 3
2: 5
3: 7
4: 11
5: 13
```

C++11で導入されて以降、範囲`for`文でイテレーション中のカウンタ、すなわち要素のインデックスが欲しいケースは多々ありました。しかしそのための簡易な方法は提供されておらず、手動でカウンタを実行するしかありませんでした。

```cpp
int main() {
  std::vector vec = {1, 3, 5, 7, 11, 13};

  for (auto index = 0u; auto value : vec) {
    std::cout << std::format("{:d}: {:d}\n", index, value);
    ++index;
  }
}
```

このようなコードはカウンタを手動で管理する以上インクリメント忘れや間違ったカウンタの変更などのバグの可能性がつきまといます。C++17以前だと、範囲`for`に初期化子を置けなかったためカウンタのスコープ（例中`index`変数のスコープ）が広くなってしまう問題もありました。そして何より、このような手動カウンタはRangeアダプタと相性が良くありません。

`views::enumerate`はこれらの問題を解決しており、これを使用することで追加の手間なしで範囲のイテレーションにカウンタを導入でき、他のRangeアダプタとも自然に合成できます。

`views::enumerate`そのものはRangeアダプタオブジェクトであり、1つの引数を受け取ってそれを`views::all`で包みながら`enumerate_view`を構築して返します。

### `enumerate_view`

`enumerate_view`は`views::enumerate`の実装詳細であり、`views::enumerate`は常にこの`view`型を返します。

```cpp
namespace std::ranges {

  // enumerate_viewの宣言例
  template<view V>
    requires range-with-movable-references<V>
  class enumerate_view : public view_interface<enumerate_view<V>> {
    ...
  };
}
```

`range-with-movable-references`は入力範囲`V`が`input_range`でありその参照型に対してムーブ構築可能であることを要求しています。

### 動作詳細

`enumerate_view`は入力範囲を保持するだけで動作のほとんどはそのイテレータが担っています。そのため、`enumerate_view`を初期化した段階ではまだ何も行われていません。

`enumerate_view`のイテレータは入力範囲のイテレータ（`current`）と現在のインデックス値（`pos`）を保持しており、`enumerate_view`からイテレータを取得すると最初は`pos`は`0`に初期化されています。`pos`の型は入力の範囲`R`の距離型（`ranges::range_difference_t<R>`）が使用されます。

その後、`enumerate_view`のイテレータのインクリメント等による進行に伴って`current`を進行させますが、同時に移動量を`pos`メンバに加算していくことでインデックス計算を行います。

間接参照では、`pos`の値と`*current`（参照）を`std::tuple`にラップして返します。

```cpp
// enumerate_view構築時は何もしない
auto ev = input | std::views::enumerate;

// イテレータ取得時、内部カウンタ（インデックス）が0に初期化
auto it = ev.begin();

// 進行に伴って、同じだけ内部カウンタも増減する
++it;
--it;
it += 3;
it -= 1;

// 現在のカウンタ値と元の要素をtupleにラップして返す
auto elem = *it;

// 1つ目の要素がインデックス
auto index = std::get<0>(elem);
// 2つ目の要素が元の範囲の要素
auto&& value = std::get<1>(elem);
```

間接参照結果の`std::tuple`は`std::tuple<range_difference_t<R>, range_reference_t<R>>`のようになり、元のイテレータの間接参照結果をそのまま（参照なら参照のまま）保持しています。そのため、間接参照結果の`std::tuple`の2つ目の要素は入力範囲の参照型が参照型なら参照型、*prvalue*なら*prvalue*になります。通常ここで要素のコピーは発生せず、元の参照型が*prvalue*の場合にのみ要素はムーブ構築されます。

イテレータが行う特別なことはこの2点だけで、イテレータの進行とインデックス（整数値）の進行は完全に同期させることができるため、`enumerate_view<R>`は元の範囲`R`の性質をほぼそのまま受け継ぎます。ただし、要素のメモリ位置は連続したものにならないため`contiguous_range`にはなりません。 

### イテレータの`.index()`

`enumerate_view`のイテレータは他のイテレータとは異なった`.index()`メンバ関数を備えています。これは要素を取得することなく現在のインデックス値を取得するもので、要素の生成（元のイテレータの間接参照）が重くなる場合にインデックス値だけが欲しい場合に使用できます。

```cpp
int main() {
  std::vector vec = {1, 3, 5, 7, 11, 13};

  auto heavy_map = [](int n) {
    // 何か重い変換処理
    ...
  };

  auto ev = vec | std::views::transform(heavy_map)
                | std::views::enumerate;

  for (auto it = ev.begin(); it != ev.end(); ++it) {
    auto index = it.index();  // 元のイテレータの間接参照を避け、インデックスのみを取得する

    ...
  }
}
```

間接参照のコストが気にならなくても、インデックス値だけが先に欲しい場合などにも利用できるかもしれません。ただし、この場合はイテレータのループを自分で管理しなければならなくなるので、書き間違いなどに注意が必要です。

### `enumerate_view`(`views::enumerate`)の諸特性

入力範囲を`R`とすると次のようになります。

- `reference` : `std::tuple<range_difference_t<R>, range_reference_t<R>>`
- `range`カテゴリ : `R`のカテゴリに従う（ただし、`contiguous_range`にはならない）
- `common_range` : `R`が`common_range`かつ`sized_range`の場合
- `sized_range` : `R`が`sized_range`の場合
- `const-iterable` : `const R`も`range-with-movable-references`を満たす場合
- `borrowed_range` : `R`が`borrowed_range`の場合

## `views::zip`

`views::zip`は複数の範囲を入力に取り、それらの対応するインデックスの要素を一纏めにした要素からなる1つの範囲を生成する`view`です。

```cpp
int main() {
  std::vector vec = {'a', 'b', 'c', 'd'};
  std::list lst = {1, 2, 3, 4};
  std::deque deq = {"あ", "い", "う", "え"};

  auto zip = std::views::zip(vec, lst, deq);

  for (auto [alpha, num, hira] : zip) {
    std::cout << std::format("({:c}, {:d}, {:s})\n", alpha, num, hira);
  }
}
```
```{style=planetext}
(a, 1, あ)
(b, 2, い)
(c, 3, う)
(d, 4, え)
```

複数の範囲を横並びに一括で扱いたい場合に、`views::zip`に通すことでそれらを1つの範囲として扱うことができます。その際、`zip`の要素型は元の範囲それぞれの要素を参照する`std::tuple`オブジェクトとなり、その`std::tuple`オブジェクト内部での各要素の順番は範囲の入力順に一致します。`zip`の各要素の`std::tuple`オブジェクトは入力の各範囲のイテレータの間接参照結果が参照なら参照を保持することでコピーを回避しており、入力イテレータの間接参照が*prvalue*を返す場合にのみムーブして値を保持します。

```cpp
int main() {
  std::vector vec = {'a', 'b', 'c', 'd'};
  std::list lst = {1, 2, 3, 4};
  std::deque deq = {"あ", "い", "う", "え"};

  auto zip = std::views::zip(vec, lst, deq);

  for (auto tuple : zip) {
    char elem0 = std::get<0>(tuple);  // vecの要素
    int elem1 = std::get<1>(tuple);   // lstの要素
    std::string_view elem2 = std::get<2>(tuple);  // deqの要素
  }

  auto zip2 = std::views::zip(deq, lst, vec);

  for (auto tuple : zip2) {
    std::string_view elem0 = std::get<0>(tuple);  // deqの要素
    int elem1 = std::get<1>(tuple);   // lstの要素
    char elem2 = std::get<2>(tuple);  // vecの要素
  }
}
```

`views::zip`の範囲としての長さは、入力範囲のうち最も短いものに合わせられます。

```cpp
int main() {
  std::vector vec = {'a', 'b', 'c', 'd'};
  std::list lst = {1, 2};
  std::deque deq = {"あ", "い", "う", "え"};

  auto zip1 = std::views::zip(vec, deq);
  auto zip2 = std::views::zip(vec, deq, lst);

  std::cout << std::format("length = {:d}\n", zip1.size());
  std::cout << std::format("length = {:d}\n", zip2.size());
}
```
```{style=planetext}
length = 4
length = 2
```

ただし、`.size()`（及び`ranges::size()`）が使用可能なのは、入力範囲が全て`sized_range`である場合のみです。

`views::zip`そのものはカスタマイゼーションポイントオブジェクトであり、0個以上の引数（`args...`）を受け取ってその数に応じて次のどちらかの動作をします

- 引数なし（`sizeof...(args) == 0`）の場合 : `views​::​empty<std::tuple<>>`
- それ以外の場合 : `args...`内のそれぞれの範囲を`views::all`で包みながら、`zip_view`を構築して返す

`views::zip`はRangeアダプタオブジェクトではないため、パイプ（`|`）を使用した入力はサポートされていません。Rangeファクトリ同様に、パイプラインの先頭で使用することになるでしょう。

```cpp
int main() {
  std::vector vec = {'a', 'b', 'c', 'd'};
  std::list lst = {1, 2, 3, 4};
  std::deque deq = {"あ", "い", "う", "え"};

  auto zip = lst | std::views::zip(vec, deq); // ng
}
```

### `zip_view`

`views::zip`に1つ以上の範囲を入力する場合は`zip_view`が返されます。

```cpp
namespace std::ranges {

  // zip_viewの宣言例
  template<input_range... Views>
    requires (view<Views> && ...) && (sizeof...(Views) > 0)
  class zip_view : public view_interface<zip_view<Views...>> {
    ...
  };
}
```

`zip_view`そのものは入力範囲に`input_range`であることくらいしか要求していないため、`zip`できる範囲とその組み合わせに制限はありません。ただし、`zip_view`のRangeカテゴリは入力範囲のカテゴリのうち最も弱いものになります。

`zip_view`は入力範囲を`views::all`に通したものを`std::tuple`につめて保持しています。

### 動作詳細

`zip_view`のイテレータは入力範囲の全てのイテレータを保持しており、進行に伴ってはそれら全てのイテレータを同様に進行させます。

`zip_view`のイテレータの間接参照では、保持する全てのイテレータに対して`*it`した結果を直接`std::tuple`にラップしてその`std::tuple`オブジェクトを返します。前述のように、ここでは参照は参照のままラップされます。

`zip_view`のイテレータ終端判定は、保持する全てのイテレータの比較が行われ、そのうちいずれか1つが終端に達している場合に`zip_view`のイテレータも終端に到達しているとみなされます。この時、短絡評価を行うかどうかは規定されていませんが禁止されてもいません。

```cpp
// 入力範囲の保存or参照のみを行う
auto zip = std::views::zip(rngs...);

// 全ての入力範囲のイテレータを取り出し保存する
auto it = zip.begin();

// 進行は保持するイテレータ全てに同じ操作を適用する
++it;
--it;
it += 3;
it -= 3;

// 保持するイテレータ全ての間接参照結果をtupleにラップして返す
auto tuple = *it;

// 保持するイテレータ全ての比較が行われる
// おそらく短絡評価は行われる
bool b = it == zip.end();
```

動作自体はそこまで難しいことはしておらず、割とシンプルです。ただし、この様子から明らかなように、`zip_view`に入力する範囲の数を`N`とすると`zip_view`のサイズ及びほぼ全ての操作に対して`O(N)`のコストがかかります。例えば、`zip_view`及びそのイテレータは`N`個の範囲（`views::all`に通したもの）orイテレータを保持しており、`zip_view`のイテレータのインクリメントは`N`個のイテレータをインクリメントし、間接参照は`N`個のイテレータを間接参照します。

### `zip_view`(`views::zip`)の諸特性

少なくとも1つ以上入力があり、入力範囲型の列（パック）を`Rs`とすると次のようになります

- `reference` : `std::tuple<range_reference_t<Rs>...>`
- `range`カテゴリ : `Rs`のカテゴリのうち最も弱いもの（ただし、`contiguous_range`にはならない）
- `common_range` : 次のいずれかの条件を満たす
    - `sizeof...(Rs) == 1`であり、`Rs`が`common_range`の場合
    - `Rs`が全て`bidirectional_range`ではなく、全て`common_range`の場合
    - `Rs`が全て`random_access_range`であり、全て`sized_range`の場合
- `sized_range` : `Rs`が全て`sized_range`の場合
- `const-iterable` : `const Rs`が全て`range`である場合
- `borrowed_range` : `Rs`が全て`borrowed_range`の場合

## `views::zip_transform`

`views::zip_transform`は、複数の範囲とそれらの要素全てを受け取る変換関数を受けて、入力範囲の対応する要素をまとめて変換関数に渡して呼び出した結果からなる1つの範囲を生成する`view`です。

```cpp
int main() {
  std::vector vec = {1, 3, 5, 7};
  std::list lst = {11, 13, 17, 19};

  auto zip_map = std::views::zip_transform(std::multiplies{}, vec, lst);

  for (auto n : zip_map) {
    std::cout << std::format("{:d} ", n);
  }
}
```
```{style=planetext}
11 39 85 133 
```

すなわち、`views::zip_transform(func, rng...)`とは`views::zip(rng...) | views::transform(func)`です。`views::zip`の各要素に対して指定された関数を適用した結果を要素とする範囲を生成します。

そのため、要素に関する部分以外のほとんどの性質は`views::zip`と共通しています。

`views::zip_transform`が`views::zip`と`views::transform`の単なる合成と異なるのは、`views::zip`で`std::tuple`に詰めた要素を`views::transform`で再び取り出す中間過程をスキップすることで、間接参照を効率化するとともに、指定する関数の引数に中間の`std::tuple`が現れてしまうことを回避している（入力範囲の要素型を直接受け取る形で指定できるようにしている）点です。

```cpp
int main() {
  std::vector vec = {1, 3, 5, 7};
  std::list lst = {11, 13, 17, 19};

  auto zip_map1 = std::views::zip_transform(std::multiplies{}, vec, lst); //ok

  auto zip_map2 = std::views::zip(vec, lst)
                | std::views::transform(std::multiplies{}); // ng

  auto zip_map3 = std::views::zip(vec, lst)
                | std::views::transform([mul = std::multiplies{}](auto tuple){
                    const auto [n, m] = tuple;
                    return mul(n, m);
                  }); // ok
}
```

`views::zip_transform`そのものはカスタマイゼーションポイントオブジェクトであり、関数呼び出し可能なもの（`func`）と0個以上の入力範囲（`rng...`）を受け取って次のいずれかの動作をします

- `rng`が空（入力範囲が0個）の場合、`FD = std::decay_t<decltype((func))>`として次のどちらか
    - `FD`が`move_constructible`ではなく`FD&`が`regular_invocable`ではない、もしくは`func`を引数なしで呼び出した結果がオブジェクト型ではない場合 : 呼び出しは不適格（コンパイルエラー）
    - それ以外の場合 : `views::empty<decay_t<invoke_result_t<FD&>>`を構築して返す
- それ以外の場合 :`func`と`rng...`をそのまま転送して`zip_transform_view`を構築して返す

`views::zip`同様に`views::zip_transform`もRangeアダプタオブジェクトではないため、`|`の右側に来ることはできません。

### `zip_transform_view`

`views::zip_transform`に1つ以上の範囲を入力する場合は`zip_transform_view`が返されます。

```cpp
namespace std::ranges {

  // zip_transform_viewの宣言例
  template<move_constructible F, input_range... Views>
    requires (view<Views> && ...) && (sizeof...(Views) > 0) && is_object_v<F> &&
              regular_invocable<F&, range_reference_t<Views>...> &&
              can-reference<invoke_result_t<F&, range_reference_t<Views>...>>
  class zip_transform_view : public view_interface<zip_transform_view<F, Views...>> {
    ...
  };
}
```

`zip_view`に対して増えている制約は追加で受けている関数（呼び出し可能オブジェクト）に対してのものです。関数に対しては入力範囲全ての間接参照結果を渡して呼び出し可能である他には`regular_invocable`であることが求めらています。すなわち、渡す関数は副作用を持ってはなりません。また、戻り値は非`void`であること（参照修飾可能であること）が求められています。

`zip_transform_view`は入力範囲を`views::zip`したものと受け取った関数をそれぞれ内部に保存しています。

### 動作詳細

`zip_transform_view`のイテレータは`zip_transform_view`が保持している`zip_view`からイテレータを取得して内部に保持します。進行時はその`zip_view`のイテレータを同時に進行させます。

`zip_transform_view`のイテレータの間接参照時は、内部の`zip_view`のイテレータが保持している入力範囲のイテレータを直接取得して、その間接参照結果を直接`zip_transform_view`が保持している関数に渡して呼び出し、その結果を返します。この呼び出しは渡された変換関数を`func`、入力範囲のイテレータを`iters...`とすると、`std::invoke(func, *iteres...)`のように行われます。間接参照結果は中間変数等を介することなく直接`func`に渡されます。

なお、間接参照における呼び出し結果は直接返されるため中間変数等を介することなく、値カテゴリ等も含めて受け側に完全に透過的になります。そのため、変換関数が*prvalue*を返す場合はコピー省略が行われます。

`zip_transform_view`のイテレータの終端判定は`zip_view`と同じです。

```cpp
// funcとviews::zip(rng...)を保存する
auto zip_map = std::views::zip_transform(func, rng...);

// views::zipのイテレータを取得し保存する
auto it = zip_map.begin();

// 進行は保持するviews::zipのイテレータに同じ操作を適用する
++it;
--it;
it += 3;
it -= 3;

// 入力イテレータ全ての間接参照結果をfuncに渡して呼び出し、その結果を返す
// この時、zip_viewのイテレータの間接参照は行われない
auto result = *it;

// 保持するイテレータ全ての比較が行われる
bool b = it == zip_map.end();
```

内部で`views::zip`を使用していることからも明らかですが、計算量についても`views::zip`と同じになります。

### `zip_transform_view`(`views::zip_transform`)の諸特性

少なくとも1つ以上入力があり、渡す呼び出し可能な型を`F`、入力範囲型の列（パック）を`Rs`とすると次のようになります

- `reference` : `std::invoke_result_t<F&, range_reference_t<Rs>...>`
- `range`カテゴリ : `Rs`について`views::zip`と同じ
- `common_range` : `Rs`について`views::zip`と同じ
- `sized_range` : `Rs`について`views::zip`と同じ
- `const-iterable` : `CRs`を`Rs`の全ての型を`const`化したパックとして、次のどちらも満たす場合
    - `const R`が`range`
    - `regular_invocable<const F&, range_reference_t<CRs>...>`を満たす
- `borrowed_range` : ×

渡した`F`のオブジェクトは`zip_transform_view`内に保存されており、そのイテレータは取得元の`zip_transform_view`オブジェクトを参照して`F`のオブジェクトを取得しています。それにより、`zip_transform_view`のイテレータは取得元の`zip_transform_view`オブジェクトの寿命が尽きるとダングリングイテレータとなるため、`zip_transform_view`は`borrowed_range`にはなりません。それ以外の部分はほぼ`views::zip`と同じになります。

## `views::adjacent`

`views::adjacent`は、`views::adjacent`による範囲上での現在位置に対応する入力範囲の位置から、連続する`N`個分の要素を一纏めにした要素からなる範囲を生成する`view`です。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4, 5, 6};

  for (auto [n1, n2, n3] : vec | std::views::adjacent<3>) {
    std::cout << std::format("({:d}, {:d}, {:d})\n", n1, n2, n3);
  }
}
```
```{style=planetext}
(1, 2, 3)
(2, 3, 4)
(3, 4, 5)
(4, 5, 6)
```

`views::adjacent`による進行自体は入力範囲上を先頭から1つづつ進んでいき、参照範囲`N`内で元の範囲の終端に到達すると`views::adjacent`も終端になります。

一纏めにする個数`N`は`views::adjacent<N>`のように非型テンプレートパラメータで渡し、`views::adjacent<N>`の`M`番目の要素は元の範囲の`M`番目から`(M + N -1)`番目（0-index）の要素を参照します。入力範囲の長さが`N`未満の場合、`views::adjacent<N>`は空の範囲になります。

```cpp
int main() {
  std::vector vec = {1, 2, 3};

  auto adj1 = vec | std::views::adjacent<3>;
  auto adj2 = vec | std::views::adjacent<4>;

  std::cout << std::format("length = {:d}\n", adj1.size());
  std::cout << std::format("length = {:d}\n", adj2.size());
}
```
```{style=planetext}
length = 1
length = 0
```

`views::adjacent<N>`の長さは、入力範囲の長さを`l`とすると`l - std::min(l, N - 1)`になります。ただし、`.size()`（及び`ranges::size()`）が使用可能なのは入力範囲が`sized_range`である場合のみです。

`views::adjacent<N>`の1つの要素は元の範囲の`N`個の要素を参照する`std::tuple`オブジェクトであり、それは入力範囲のイテレータの間接参照結果が参照なら参照を保持することでコピーを回避しており、入力イテレータの間接参照が*prvalue*を返す場合にのみムーブして値を保持します。

`views::adjacent<N>`そのものはRangeアダプタオブジェクトであり、`N`と1つの範囲を受け取って次のどちらかの動作をします

- `N == 0` : `views​::​empty<std::tuple<>>`を返す
- それ以外の場合 : と受けた引数を転送して`adjacent_view<N>`を構築して返す

`N == 2`の場合のより適切な命名のために`views::pairwise`も用意されています。`views::pairwise`は単に`views::adjacent<2>`の別名です。

```cpp
int main() {
  std::string str = "pairwise";

  for (auto [c1, c2] : str | std::views::pairwise) {
    std::cout << std::format("({:c}, {:c})\n", c1, c2);
  }
}
```
```{style=planetext}
(p, a)
(a, i)
(i, r)
(r, w)
(w, i)
(i, s)
(s, e)
```

### `adjacent_view`

`views::adjacent<N>`で`N`が1以上の場合は`adjacent_view`が返されます。

```cpp
namespace std::ranges {

  // adjacent_viewの宣言例
  template<forward_range V, size_t N>
    requires view<V> && (N > 0)
  class adjacent_view : public view_interface<adjacent_view<V, N>> {
    ...
  };
}
```

`adjacent_view`の入力範囲は`forward_range`でなければなりません。入力範囲のイテレータは最大`N`回間接参照されるので、マルチパス保証が必須になるためです。

`adjacent_view`は入力範囲を`views::all`に通してそれを保持しています。

### 動作詳細

`adjacent_view`のイテレータは元の範囲のイテレータを`N`個分保持しています。元の範囲のイテレータ型を`I`とすると`std::array<I, N>`のような形で保存されており、先頭のイテレータ（``）から最後（`.back()`）2め1つづつインクリメントされます（実際には保存方法がこう指定されているわけではありませんが、説明のために以下ではそうなっているものとします）。`adjacent_view`のイテレータの進行時には保持している`N`個のイテレータを同様に進行させます。

`adjacent_view`のイテレータの間接参照時は、そのように保持している`N`個のイテレータに対して`*i`した結果を直接`std::tuple`にラップしてその`std::tuple`オブジェクトを返します。ここで行われることは`zip_view`のイテレータとほぼ同じことで、それと同様に参照は参照のままラップされます。

`adjacent_view`のイテレータの終端判定は、保持する`N`個のイテレータのうち`.back()`のイテレータが終端に到達しているかによって行われます。このため、範囲`for`のようにチェックしながらインクリメントしている限り、保持するイテレータが元の範囲をオーバーランすることはありません。

```cpp
auto adj = rng | std::views::adjacent<N>;

// 元の範囲のイテレータをN個分保存する
auto it = adj.begin();

// 進行は保持するイテレータ全てに同じ操作を適用する
++it;
--it;
it += 3;
it -= 3;

// 保持するイテレータ全ての間接参照結果をtupleにラップして返す
auto tuple = *it;

// 保持するイテレータのうち一番最後のものと比較される
bool b = it == adj.end();
```

なお、イテレータの進行時に`N`個のイテレータを同時に進行させるかは実装によります。例えばインクリメント時に、`.back()`のイテレータをコピーしたものをインクリメントしてから`.front()`のイテレータを捨てるように保持しているイテレータを移動させて`.back()`のイテレータを更新する、ような実装も可能です。ただ、この場合でも`N`回のインクリメントの代わりに`N`回のムーブ代入が行われます。

おそらくですが、イテレータを1つだけ保持して間接参照時にそれを`N`個分コピーして進行させてから間接参照するような実装は、間接参照のコストが大きくなり実装が複雑化するため行われないと思われます（明確に禁止されてはいないようですが）。そのため、`adjacent_view`のイテレータは`O(N)`の空間コストがかかるでしょう。

### `adjacent_view`（`views::adjacent`）の諸特性

指定した隣接数を`N`、入力範囲を`R`とすると次のようになります

- `reference` : `Rn`を`N`個の`R`からなるパックとすると
    - `std::tuple<range_reference_t<Rn>...>`
- `range`カテゴリ : `R`のカテゴリと同じ（ただし、`contiguous_range`にはならない）
- `common_range` : `R`が`common_range`の場合
- `sized_range` : `R`が`sized_range`の場合
- `const-iterable` : `const R`が`range`の場合
- `borrowed_range` : `R`が`borrowed_range`の場合

## `views::adjacent_transform`

`views::adjacent_transform`は`views::adjacent`の各要素に対して指定した変換を適用した結果からなる範囲を生成する`view`です。

```cpp
int main() {
  std::vector vec = {1, 3, 5, 7, 11, 13, 17, 19};

  auto product = [](auto... args) {
    return (args * ...);
  };

  for (int n : vec | std::views::adjacent_transform<3>(product)) {
    std::cout << std::format("{} ", n);
  }
}
```
```{style=planetext}
15 105 385 1001 2431 4199
```

`views::adjacent_transform<N>(func, rng)`は`views::adjacent<N>(rng)`の要素から元の`N`個の要素`func`に渡して呼び出し、その結果を要素として返します。すなわち、`rng | views::adjacent<N> | views::transform(func)`と同じですが、`views::zip`に対する`views::zip_transform`と同様に`std::tuple`に詰めて取り出すという中間過程をスキップすることで間接参照の効率化と変換関数の引数が元の要素型を直接受け取れるようにしています。

そのため、要素に関する部分以外のほとんどの性質は`views::adjacent`と共通しています。

`views::adjacent_transform<N>`そのものはRangeアダプタオブジェクトであり、`N`と1つの範囲と変換関数`func`を受け取って次のどちらかの動作をします

- `N == 0` : `views​::​zip_transform(func)`を返す
- それ以外の場合 : 受けた引数を転送して`adjacent_transform_view<N>`を構築して返す

`views::adjacent`同様に、`N == 2`の場合の別名として`views::pairwise_transform`も用意されています。

```cpp
int main() {
  std::string str = "pairwise";

  auto to_string = [](auto... chars) {
    return std::string{chars...};
  };

  for (auto str : str | std::views::pairwise_transform(to_string)) {
    std::cout << std::format("({:s})\n", str);
  }
}
```
```{style=planetext}
(pa)
(ai)
(ir)
(rw)
(wi)
(is)
(se)
```

### `adjacent_transform_view`

`views::adjacent_transform<N>`で`N`が1以上の場合は`adjacent_transform_view`が返されます。

```cpp
namespace std::ranges {

  // adjacent_transform_viewの宣言例
  template<forward_range V, move_constructible F, size_t N>
    requires view<V> && (N > 0) && is_object_v<F> &&
             regular_invocable<F&, REPEAT(range_reference_t<V>, N)...> &&
             can-reference<invoke_result_t<F&, REPEAT(range_reference_t<V>, N)...>>
  class adjacent_transform_view : public view_interface<adjacent_transform_view<V, F, N>> {
    ...
  };
}
```

`adjacent_view`に対して増えている制約は追加で受けている変換関数（呼び出し可能オブジェクト）に対してのものです。変換関数に対しては入力範囲の間接参照結果を`N`個渡して呼び出し可能である他には`regular_invocable`であることが求めらています。すなわち、渡す関数は副作用を持ってはなりません。また、戻り値は非`void`であること（参照修飾可能であること）が求められています。

`REPEAT(range_reference_t<V>, N)`というものは、`N`個の`range_reference_t<V>`からなる型パックを生成するマクロのようなものです。

`adjacent_transform_view`は入力範囲を`views::adjacent`したものと受け取った変換関数をそれぞれ内部に保存しています。

### 動作詳細

`adjacent_transform_view`のイテレータは`adjacent_transform_view`が保持している`adjacent_view`からイテレータを取得して内部に保持します。進行時はその`adjacent_view`のイテレータを同時に進行させます。

`adjacent_transform_view`のイテレータの間接参照時は、内部の`adjacent_view`のイテレータが保持している入力範囲のイテレータを直接取得して、その間接参照結果を直接`adjacent_transform_view`が保持している関数に渡して呼び出し、その結果を返します。この呼び出しは渡された変換関数を`func`、入力範囲のイテレータを`iters...`とすると、`std::invoke(func, *iteres...)`のように行われます。間接参照結果は中間変数等を介することなく直接`func`に渡されます。

なお、間接参照における呼び出し結果は直接返されるため中間変数等を介することなく、値カテゴリ等も含めて受け側に完全に透過的になります。そのため、変換関数が*prvalue*を返す場合はコピー省略が行われます。

`adjacent_transform_view`のイテレータの終端判定は`adjacent_view`と同じです。

```cpp
// funcとviews::adjacent(rng...)を保存する
auto adj_map = rng | std::views::adjacent_transform<N>(func);

// views::adjacentのイテレータを取得し保存する
auto it = adj_map.begin();

// 進行は保持するviews::adjacentのイテレータに同じ操作を適用する
++it;
--it;
it += 3;
it -= 3;

// 入力イテレータ全ての間接参照結果をfuncに渡して呼び出し、その結果を返す
// この時、adjacent_viewのイテレータの間接参照は行われない
auto tuple = *it;

// 保持するイテレータのうち一番最後のものと比較される
bool b = it == adj_map.end();
```

動作は`zip_transform_view`(`views::zip_transform`)とよく似ているというかほとんど同じことをしています。

### `adjacent_transform_view`（`views::adjacent_transform`）の諸特性

指定した隣接数を`N`、渡す呼び出し可能な型を`F`、入力範囲を`R`とすると次のようになります

- `reference` : `Rn`を`N`個の`R`からなるパックとすると
    - `std::invoke_result_t<F&, range_reference_t<Rn>...>`
- `range`カテゴリ : `R`について`views::adjacent`と同じ
- `common_range` : `R`について`views::adjacent`と同じ
- `sized_range` : `R`について`views::adjacent`と同じ
- `const-iterable` : `CRn`を`N`個の`const R`からなるパックとして、次のどちらも満たす場合
    - `const R`が`range`
    - `regular_invocable<const F&, range_reference_t<CRn>...>`を満たす
- `borrowed_range` : ×

渡した`F`のオブジェクトは`adjacent_transform_view`内に保存されており、そのイテレータは取得元の`adjacent_transform_view`オブジェクトを参照して`F`のオブジェクトを取得しています。それにより、`adjacent_transform_view`のイテレータは取得元の`adjacent_transform_view`オブジェクトの寿命が尽きるとダングリングイテレータとなるため、`adjacent_transform_view`は`borrowed_range`にはなりません。それ以外の部分はほぼ`views::adjacent`と同じになります。

## `views::chunk`

`views::chunk`は入力の範囲の要素を指定された数まとめたチャンクを要素とする範囲を生成する`view`です。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4, 5, 6, 7};

  for (auto chunk : vec | std::views::chunk(3)) {
    std::cout << "[ ";
    for (int n : chunk) {
      std::cout << std::format("{:d} ", n);
    }
    std::cout << "]\n";
  }
}
```
```{style=planetext}
[ 1 2 3 ]
[ 4 5 6 ]
[ 7 ]
```

`views::chunk`は常に`range`の`range`を生成します。

まとめる数（チャンクサイズ）`n`は実行時の値として指定し、`views::chunk`の要素（チャンク）は`n`個の要素からなる`range`となります。ただし、末尾要素で残りの要素が足りない場合（入力範囲の長さが`n`で割り切れない場合の最後のチャンク）はチャンクの長さが`n`未満になります。

`views::chunk`そのものはRangeアダプタオブジェクトであり、入力範囲とチャンクサイズを受け取って、それをそのまま転送して`chunk_view`を構築して返します。

### `chunk_view`

`chunk_view`は`views::chunk`の実装詳細であり、`views::chunk`は常にこの型を返します。

```cpp
namespace std::ranges {

  // chunk_viewの宣言例（プライマリテンプレート）
  template<view V>
    requires input_range<V>
  class chunk_view : public view_interface<chunk_view<V>> {
    ...
  };

  // forward_rangeのためのchunk_view特殊化
  template<view V>
    requires forward_range<V>
  class chunk_view<V> : public view_interface<chunk_view<V>> {
    ...
  };
}
```

`chunk_view`の入力範囲には`input_range`であることしか求められていませんが、入力範囲が`forward_range`（よりも強い）であるかどうかによって、その実装は分岐します。これは、`input_range`をサポートするためのことです。

どちらの場合でも、`chunk_view`は入力範囲を`views::all`に通したものとチャンクサイズを内部に保存しています。

### 動作詳細

前述のように`chunk_view`の動作は入力範囲が`forward_range`であるかによって大きく変化します。

入力範囲が`forward_range`以上であるとき、`chunk_view`のイテレータは入力範囲のイテレータと番兵（`it, se`）を保持しており、イテレータの進行時にはイテレータと番兵とチャンクサイズ`n`によって`ranges::advance(it, n, se)`のように進行させることで`n`進めながら元の範囲上でオーバーランしないようにしています。

間接参照時には、`views::take(ranges::subrange{it, se}, n)`を返します。このため、間接参照結果型は入力範囲によって変動します。

終端判定は現在のイテレータを元の範囲の終端イテレータと比較することによって行われます。

```cpp
// 最初はまだ何もしない
auto chv = rng | std::views::chunk(n);

// 元の範囲の先頭イテレータと番兵を保存する
auto it = chv.begin();

// 進行は元の範囲でオーバーランしないように進行する
++it;
--it;
it += 3;
it -= 3;

// 元の範囲の現在位置からの部分範囲を取得し
// そこからviews::takeによってチャンクサイズ分の範囲を切り出す
auto chunk = *it;

// 元のイテレータ同士の比較になる
bool b = it == chv.end();
```

入力範囲が`input_range`でしかないとき、`chunk_view`は内外`range`がすべて`input_range`になります。この場合の`chunk_view`は入力範囲とチャンクサイズ（`n`）に加えて元の範囲のイテレータ（`it`）をその内部に保持し、さらに最も内側の範囲（各チャンク）の残りの長さを記録するための変数（`remainder`）を保持しています。

外側イテレータ（`chunk_view`のイテレータ）の取得時には、`chunk_view`内部で入力のイテレータを取得し、`remainder`を`n`にセットします。

外側イテレータの間接参照時には、外側イテレータ内部で定義されている独自の`input_range`型を返します。この内側`range`のイテレータは自身の残りの長さを親の`chunk_view`で保持されている`remainder`によって把握しており、進行に伴って元のイテレータを1つ進めるとともにその値を1づつ減じていきます。`remainder`が0になっている時、内側`range`は終端に達しているとみなされます（`operator==`は`remainder == 0`を返す）。その際、進めた後の元のイテレータは終端チェックが行われ、`remainder`が0になるよりも早く元のイテレータの終端に到着した場合は`remainder`が0にセットされます。

内側イテレータの間接参照時は元のイテレータの間接参照結果（`*it`）を直接返します。

外側イテレータ（`chunk_view`のイテレータ）のインクリメント時には、入力範囲の番兵（`se`）と`n`を用いて`ranges::advance(it, n se)`のように元のイテレータを最大`n`進行させ、`remainder`に`n`をセットします。この時も、元の範囲をオーバーランしないようになっています。

外部イテレータの終端判定は元のイテレータを元の範囲の終端イテレータと比較することで行われますが、外側イテレータが意味的に終端に到達するのは最後のチャンクに到達した後でインクリメントされてからであり、単に元のイテレータの終端チェックだけだと最後のチャンクの走査中に終端に到達してしまうため、`remainder != 0`を追加でチェックすることで最後のチャンクを超えてから終端に到達するようになっています（`remainder`はチャンクの終端に到達すると0になり、外側イテレータをインクリメントすると`n`になるため）。

```cpp
// 要素のチャンクの現在サイズを表す変数remainderを提供する
auto chv = rng | std::views::chunk(n);

// chunk_view内部に元の範囲の先頭イテレータを保存し
// remainderをnにセット
auto it = chv.begin();

// 進行は元の範囲でオーバーランしないように進行する
// 進行するたびにremainderをnにセット
++it;

// 独自のinput_rangeを返す
auto chunk = *it;

// 独自定義内側イテレータ
auto inner_it = chunk.begin();

// 内側イテレータ進行時は元の範囲のイテレータを進行させる
// 同時に親のremainderを減じていくことで残りの長さを管理する
// 元の範囲で終端に達したら、remainderを0にする
++inner_it;

// remainder == 0ならチャンク終端
bool ib = inner_it == chunk.end();

// 元のイテレータ同士の比較とremainder != 0をチェック
bool ob = it == chv.end();
```

非常に複雑ですが、内側・外側イテレータは同じ1つのイテレータ（入力範囲のイテレータ）を共有しており、外側の`range`も内側の`range`（各チャンク）もその共有する1つのイテレータによって管理されています。そのイテレータは各チャンクの残りの長さを表す`remainder`とともに`chunk_view`内部に保存されており、内外のイテレータは親の`chunk_view`への参照を持っておくことでそれらを共同利用します。

### `chunk_view`（`views::chunk`）の諸特性

入力範囲を`R`として、まず`R`が`forward_range`の場合は次のようになります

- `reference` : チャンクサイズを`n`とすると
    - `R`の部分範囲`subrng`について、`views::take(subrng, n)`
- `common_range` : `R`が`common_range`かつ`sized_range`の場合
- `sized_range` : `R`が`sized_range`の場合
- `const-iterable` : `const R`が`forward_range`の場合
- `borrowed_range` : `R`が`borrowed_range`の場合

そして、`R`が`input_range`の場合は次のようになります

- `reference` : `input_range`かつ`view`である型
- `range`カテゴリ : `input_range`
- `common_range` : ×
- `sized_range` : `R`が`sized_range`の場合
- `const-iterable` : ×
- `borrowed_range` : ×

## `views::slide`

`views::slide`は指定した幅のウィンドウで入力範囲上を参照し、そのウィンドウに対応する部分範囲を要素とする`view`です。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4, 5, 6};

  for (auto rng : vec | std::views::slide(3)) {
    std::cout << "[ ";
    for (auto n : rng) {
      std::cout << std::format("{:d} ", n);
    }
    std::cout << "]\n";
  }
}
```
```{style=planetext}
[ 1 2 3 ]
[ 2 3 4 ]
[ 3 4 5 ]
[ 4 5 6 ]
```

`views::slide`は実行時に要素数を指定する`views::adjacent`です。また、`views::chunk`は`views::slide`に対して入力範囲上でウィンドウが重ならないように進行するものです。そのため、動作や性質はこの2つのRangeアダプタとよく似ている部分があります。

`views::slide(n)`の長さは、入力範囲の長さを`l`とすると`l - n + 1`（結果が0以下になる場合は0）になります。これは`views::adjacent`と計算が少し異なりますが結果は同じになり、おそらく長さを指定する変数の型の都合によります（`views::slide`の場合は距離型になるため、型変換が不要）。ただし、`.size()`（及び`ranges::size()`）が使用可能なのは入力範囲が`sized_range`である場合のみです。

実行時にウィンドウサイズを指定するため`views::chunk`同様に要素型は`range`型となり、`views::slide`は`range`の`range`となります。

`views::slide`そのものはRangeアダプタオブジェクトであり、入力範囲とウィンドウサイズを受け取ってそれをそのまま転送して`slide_view`を構築して返します。

### `slide_view`

`views::slide`は常に`slide_view`を返します。

```cpp
namespace std::ranges {

  // slide_viewの宣言例
  template<forward_range V>
    requires view<V>
  class slide_view : public view_interface<slide_view<V>> {
    ...
  };
}
```

`slide_view`の入力範囲は`forward_range`でなければなりません。入力範囲のイテレータは複数回間接参照される可能性があるのでマルチパス保証が必須になるためです。これは`views::chunk`とは異なる所で、`views::slide`は`input_range`のサポートを行いません。

`slide_view`は、受け取った入力範囲を`views::all`に通したものと、指定されたウィンドウサイズを内部で保存しています。

### 動作詳細

`slide_view`のイテレータは`adjacent_view`のものとは異なり、入力範囲のイテレータを1つ（最大2つ）だけ保持しています。イテレータの進行時には、保持するすべてのイテレータを同じだけ進めます。

ただし、入力範囲が`random_access_range`でも`sized_range`でもないとき、`slide_view`の`end`イテレータ取得時には要素となる最後の部分範囲の先頭位置を指すイテレータを計算するとともにそれをイテレータ内部に保存します。さらに、`bidirectional_range`でも`common_range`でもないとき、`slide_view`の`begin`イテレータ取得時にも要素となる最初の部分範囲の最後の位置を指すイテレータを計算するとともにそれをイテレータ内部に保存します。

`slide_view`の`begin/end`イテレータが2つのイテレータを保存している場合、最初に計算されたそのイテレータは`slide_view`内部にキャッシュされます。これによって、指定されたウィンドウサイズ`n`によって`begin()/end()`に`O(n)`の時間コストが常にかかってしまうことを回避し、`range`コンセプトの意味論要件である`begin()/end()`の呼び出しが償却定数であるという要件を満たしています。

イテレータの間接参照時には、保持しているイテレータ（`it`）と指定されたウィンドウサイズ（`n`）によって`views::counted(it, n)`を返します。これは、入力範囲によって`span, sunrange, counted_iterator`のいずれかによる部分範囲を返します。

`slide_view`のイテレータが内部に1つしかイテレータを持たない場合（入力範囲が`random_access_range`かつ`sized_range`の場合）、終端判定は保持するイテレータ同士の比較によって行われます。

`slide_view`のイテレータが内部に2つのイテレータを保持する場合、2つ目のイテレータは常に元の範囲上で現在位置からのスライディングウィンドウの終わりの位置を指すようになっており、この2つ目のイテレータを比較することによって終端判定が行われます。この場合、`slide_view`のイテレータの進行時に2つ目のイテレータも同じだけ進行し、2つ目のイテレータが先に入力範囲上の終端に到達することでオーバーランしないようになっています。

```cpp
auto sld = rng | std::views::slide(n);

// 元の範囲のイテレータを保存する
// 必要な場合、slide_viewの最初の要素の最後の位置のイテレータを追加で保存する
auto it = sld.begin();

// 元の範囲のイテレータを保存する
// 必要な場合、slide_viewの最後の要素の最初の位置のイテレータを追加で保存する
auto se = sld.end();

// 進行は保持するイテレータ全てに同じ操作を適用する
++it;
--it;
it += 3;
it -= 3;

// 保持するイテレータをcurrentとすると
// views::counted(current, n)を返す
auto subrng = *it;

// 保持するイテレータ同士の比較になる
// 保持するイテレータが1つの場合、そのイテレータの比較
// 保持するイテレータが2つの場合、2つ目のイテレータの比較
bool b = it == sld.end();
```

`views::counted`の`views::take`との違い及びそのメリットとしては次のものがあります

- 部分範囲の終端イテレータが必要ない
    - 終端計算を省略できる
- 進行等に伴うオーバーラン防止のチェックが行われない
    - 要素の部分範囲の走査が効率的になる

`adjacent_view`のイテレータが`N`個のイテレータを内部に保持しサイズや操作に`O(N)`の計算量がかかるのに対して、`slide_view`のイテレータは多くても2つのイテレータしか保持しません。さらに、その2つのイテレータをうまくやりくりして入力範囲をオーバーランしないようにしているため、`views::counted`を安全に利用しつつ上記メリットを得ることができます。そのため、実行時の動作はともすればこちらの方が効率的になるかもしれません。

### `slide_transform`

要素が`tuple`にラップされる中間地点がなくメリットが薄いためか、`slide_transform`のようなものは用意されていません。それが欲しい場合は`views::transform`を接続することで得られます。

```cpp
auto slide_transform(auto func, int n) {
  return std::views::slide(n) | std::views::transform(func);
}

int main() {
  std::vector vec = {1, 3, 5, 7, 11, 13, 17, 19};

  auto product = []<std::ranges::range R>(R&& rng) {
    std::ranges::range_value_t<R> p = 1;
    for (auto n : rng) {
      p *= n;
    }
    return p;
  };

  for (int n : vec | slide_transform(product, 3)) {
    std::cout << std::format("{} ", n);
  }
}
```
```{style=planetext}
15 105 385 1001 2431 4199 
```

この場合、変換関数の引数型は入力範囲の部分範囲になってしまいますが、間接参照結果は直接得られます。

### `slide_view`（`views::slide`）の諸特性

入力範囲を`R`とすると次のようになります

- `reference` : 入力範囲の現在のイテレータを`it`、ウィンドウサイズを`n`とすると
    - `views::counted(it, n)`
- `range`カテゴリ : `R`のカテゴリと同じ（ただし、`contiguous_range`にはならない）
- `common_range` : `R`について次のどちらか
    - `random_access_range`かつ`sized_range`の場合
    - `common_range`の場合
- `sized_range` : `R`が`sized_range`の場合
- `const-iterable` : `const R`が`random_access_range`かつ`sized_range`の場合
- `borrowed_range` : `R`が`borrowed_range`の場合

`R`が`random_access_range`かつ`sized_range`ではないとき、前述のように`begin/end`イテレータのどちらか片方もしくは両方のイテレータを内部でキャッシュしています。そのため、その場合は`const-iterable`ではありません。

## `views::chunk_by`

`views::chunk_by`は指定された条件を満たす隣接する要素を1つのチャンクとして、それを要素とする`view`です。

```cpp
int main() {
  std::vector vec = {1, 2, 2, 3, 0, 4, 5, 2};

  for (auto chunk : vec | std::views::chunk_by(std::ranges::less{})) {
    std::cout << "[ "
    for (int n : chunk) {
      std::cout << std::format("{:d} ", n);
    }
    std::cout << "]\n"
  }
}
```
```{style=planetext}
[ 1 2 ]
[ 2 3 ]
[ 0 4 5 ]
[ 2 ]
```

`views::chunk_by`に渡す述語は入力範囲の要素を2つ受ける二項述語である必要があり、そこには入力範囲で隣接している2つの要素が渡されます。渡される要素は、入力範囲に`views::pairwise`（あるいは`views::slide(2)`）を適用した結果の要素と同じペアが渡されます。

`views::chunk_by`による1つのチャンクは、述語に隣接する要素を渡して呼び出した結果が`true`である間持続し、`false`となった時にチャンクが区切られます。

上記例の場合の、入力範囲に対する指定した述語`pred`（`ranges::less`）の適用のイメージ

```
[1, 2, 2, 3, 0, 4, 5, 2]
[1, 2]
   [2, 2] <- pred(2, 2) == false
      [2, 3]
         [3, 0] <- pred(3, 0) == false
            [0, 4]
               [4, 5]
                  [5, 2] <- pred(5, 2) == false

[1, 2][2, 3][0, 4, 5][2]
```

`views::chunk_by`そのものはRangeアダプタオブジェクトであり、入力範囲と述語を受け取ってそれをそのまま転送して`chunk_by_view`を構築して返します。

### `chunk_by_view`

`views::chunk_by`は常に`chunk_by_view`を返します。

```cpp
namespace std::ranges {

  // chunk_by_viewの宣言例
  template<forward_range V, indirect_binary_predicate<iterator_t<V>, iterator_t<V>> Pred>
    requires view<V> && is_object_v<Pred>
  class chunk_by_view : public view_interface<chunk_by_view<V, Pred>> {
    ...
  };
}
```

`chunk_by_view`の入力範囲は`forward_range`でなければなりません。入力範囲のイテレータは2回間接参照されるのでマルチパス保証が必須になるためです。

`chunk_by_view`は、受け取った入力範囲を`views::all`に通したものと、指定された述語オブジェクトを内部で保存しています。

### 動作詳細

`chunk_by_view`のイテレータは元の範囲のイテレータを2つ保持しており、そのペア（`current, next`）によって1つのチャンクの部分範囲を表します。イテレータの間接参照では、`ranges::subarange(current, next)`を返します。

`chunk_by_view`のイテレータの取得時には元の範囲の先頭イテレータを取得して`current`を初期化し、渡された条件（`perd`）で先頭から隣接要素をチェックし最初に条件を満たさない要素へのイテレータ（最初のチャンクの終端/次のチャンクの先頭）によって`next`を初期化します。

この際、`begin()`の最初の呼び出しで一度このように初期化されたイテレータをキャッシュし以降の呼び出しでそれを返すようにすることで、`range`コンセプトの意味論要件を満たしています。

`chunk_by_view`のイテレータ進行時には、`current`を`next`で置き換えた後、取得時同様にそこから最初に条件を満たさない要素へのイテレータ（チャンクの終端/次のチャンクの先頭）を探し、それによって`next`を更新します。

取得時及び進行時にチャンク終端を捜索するのは、入力範囲の番兵を`end`とすると、`ranges::adjacent_find(current, end, not_fn(ref(pred)))`によって行われ、`next`はこの結果を`it`とすると`ranges::next(it, 1, end)`によって更新されます。`adjacent_find`は範囲先頭から隣接する要素を2つづつ述語でチェックしていき、条件を満たす最初のペアの1つ目の要素を指すイテレータを返すため、この1つ後ろが`views::chunk_by`の1つのチャンクの終端となります。

終端判定は元のイテレータ同士の比較によって行われます。

```cpp
auto cby = rng | std::views::chunk_by(pred);

// 元の範囲の先頭イテレータを保存（current）
// 最初のチャンクの終端を計算しそのイテレータも保存（next）
auto it = cby.begin();

// 2回目以降は最初の呼び出し時のキャッシュが返される
auto it2 = cby.begin();

// 進行時は2つのイテレータをそれぞれ進める
// current = next
// nextを次のチャンクの終端位置まで進める
++it;

// 後退時は逆
// next = current
// currentを1つ前のチャンクの先頭まで戻す
--it;

// ranges::subrange(current, next)を返す
auto subrng = *it;

// 保持するイテレータ同士の比較になる
// common_rangeではない場合、current == nextによって終端判定
bool b = it == cby.end();
```

### `chunk_by_view`（`views::chunk_by`）の諸特性

入力範囲を`R`とすると次のようになります

- `reference` : `R`のイテレータ型を`I`とすると
    - `ranges::subrange<I, I>`
- `range`カテゴリ :
    - `R`が`bidirectional_range`の場合 : `bidirectional_range`
    - それ以外の場合 : `forward_range`
- `common_range` : `R`が`common_range`の場合
- `sized_range` : ×
- `const-iterable` : ×
- `borrowed_range` : ×

## `views::stride`

`views::stride`は指定された幅によって入力範囲上で等間隔な位置にある要素からなる`view`です。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

  for (int n : vec | std::views::stride(2)) {
    std::cout << std::format("{:d} ", n);
  }
}
```
```{style=planetext}
1 3 5 7 9 
```

指定する値はスキップする要素数ではなくスキップする幅です。`views::stride(n)`はそのイテレータが`m`進む際に、入力範囲上では`m * n`進みます。

```cpp
int main() {
  for (auto n : std::views::iota(0) 
              | std::views::stride(10)
              | std::views::take(5))
  {
    std::cout << std::format("{:d} ", n);
  }
}
```
```{style=planetext}
0 10 20 30 40 
```

```cpp
using namespace std::ranges;

int main() {
  std::vector vec(10, 0);

  fill(vec | views::stride(3), 23);

  for (int n : vec) {
    std::cout << std::format("{:d} ", n);
  }
}
```
```{style=planetext}
23 0 0 23 0 0 23 0 0 23 
```

`views::stride`そのものはRangeアダプタオブジェクトであり、入力範囲とスキップ幅を受け取ってそれをそのまま転送して`stride_view`を構築して返します。

### `stride_view`

`views::stride`は常に`stride_view`を返します。

```cpp
namespace std::ranges {

  // stride_viewの宣言例
  template<input_range V>
    requires view<V>
  class stride_view : public view_interface<stride_view<V>> {
    ...
  };
}
```

`stride_view`は受け取った入力範囲を`views::all`に通したものと、指定されたスキップ幅を内部で保存しています。

### 動作詳細

`stride_view`のイテレータは入力範囲の先頭と終端のイテレータペア（`current, end`）と指定されたスキップ幅及び進行時に進めなかった距離（`stride, missing`）を保存する変数を保持しています。

`stride_view`のイテレータの進行時には保持するイテレータを`ranges::advance(current, stride, end)`のように進めて、この戻り値を`missing`に保存します。`missing`は`current`が終端に到達した時に非ゼロになります。

イテレータの間接参照時は保持するイテレータの間接参照（`*current`）を直接返し、終端判定は`current`の比較によって行われます。

`missing`が使用されるのは`stride_view`のイテレータの後退時で、`ranges::advance(current, missing - stride)`のように後退させて`missing`を0にします。これは、終端から逆に戻る際に進行時と同じ要素を指すようにするためのものです。

```cpp
auto strd = rng | std::views::stride(n);

// 元の範囲のイテレータペアを保存（current, end）
// ストライド値と終端調整値も保持（stride, missing）
auto it = cby.begin();

// 進行時はranges::advance(current, stride, end)のように進める
// そして、この戻り値をmissingに保存
++it;

// 後退時は逆ranges::advance(current, missing - stride)
// そして、missingを0にする
--it;

// ランダムアクセス時もほぼ同様
// ranges::advance(current, stride * d + missing, end)
// ただし、進行時はmissingを正しく更新するために2回に分けて進行する
it += d;
it -= d;

// *currentをそのまま返す
auto elem = *it;

// 保持するイテレータ同士の比較になる
// common_rangeではない場合、current == endによって終端判定
bool b = it == cby.end();
```

`missing`による正しい逆順操作のサポートは特に、`stride_view`が`random_access_range`となるために必要になります。

### 後退と逆転

`views::stride`による範囲は、先頭から辿っても末尾から辿っても同じ要素列を生成するようになっています。そのため、末尾から`views::stride`するのと、`views::stride`を`views::reverse`するのでは結果が異なります。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

  // 先頭からのstride(2)
  for (int n : vec | std::views::stride(2)) {
    std::cout << std::format("{:2d} ", n);
  }
  
  std::cout << '\n';
  
  // stride(2)の逆順
  for (int n : vec | std::views::stride(2) 
                   | std::views::reverse)
  {
    std::cout << std::format("{:2d} ", n);
  }

  std::cout << '\n';

  // 末尾からのstride(2)
  for (int n : vec | std::views::reverse 
                   | std::views::stride(2))
  {
    std::cout << std::format("{:2d} ", n);
  }
}
```
```{style=planetext}
 1  3  5  7  9 
 9  7  5  3  1 
10  8  6  4  2 
```

### `stride_view`（`views::stride`）の諸特性

入力範囲を`R`とすると次のようになります

- `reference` : `range_reference_t<R>`
- `range`カテゴリ : `R`のカテゴリと同じ（ただし、`contiguous_range`にはならない）
- `common_range` : `R`が`common_range`であり
    - `sized_range`かつ`forward_range`である場合
    - そうではないなら、`bidirectional_range`ではない場合
- `sized_range` : `R`が`sized_range`である場合
- `const-iterable` : `const R`が`range`である場合
- `borrowed_range` : `R`が`borrowed_range`である場合

## `views::cartesian_product`

`views::cartesian_product`は複数の範囲を入力に取り、それらの直積となる範囲を表す`view`です。

```cpp
int main() {
  std::vector vec = {'a', 'b', 'c'};
  std::string str = "def";

  for (auto [a, b] : std::views::cartesian_product(vec, str)) {
    std::cout << std::format("({:c}, {:c})\n", a, b);
  }
}
```
```{style=planetext}
(1, a)
(1, b)
(1, c)
(2, a)
(2, b)
(2, c)
(3, a)
(3, b)
(3, c)
```

`views::zip`と異なる点は、各要素は入力範囲のそれぞれの要素の順序対となるところです。`views::cartesian_product`はこの順序対の可能なペアを全て列挙する範囲となります。

例えば多重ループを1つの`for`ループによって実現することができます

```cpp
// assertのためにインクルード
#include <cassert>

auto multi_index(std::integral auto... Ns) {
  using namespace std::views;

  assert(((0 <= Ns) && ...));
  
  return cartesian_product(iota(0, Ns)...);
}

int main() {
  for (auto [x, y, z] : multi_index(3, 3, 3)) {
    std::cout << std::format("({:d}, {:d}, {:d})\n", x, y, z);
  }
}
```
```{style=planetext}
(0, 0, 0)
(0, 0, 1)
    .
    .
    .
(2, 2, 1)
(2, 2, 2)
```

`views::cartesian_product`の長さは入力範囲の長さの総乗となり、この例では27要素になります（そのため、出力が長くなるので途中を省略しています）。

`views::cartesian_product`の要素型は`views::zip`と同様に入力範囲の各要素を参照する`std::tuple`となり、その要素の順番は範囲の入力順に一致します。また、要素の`std::tuple`オブジェクトは入力の各範囲のイテレータの間接参照結果が参照なら参照を保持することでコピーを回避しており、入力イテレータの間接参照が*prvalue*を返す場合にのみムーブして値を保持します。

`views::cartesian_product`そのものはカスタマイゼーションポイントオブジェクトであり、0個以上の引数（`args...`）を受け取ってその数に応じて次のどちらかの動作をします

- 引数なし（`sizeof...(args) == 0`）の場合 : `views​::single(std::tuple())`
- それ以外の場合 : `args...`内のそれぞれの範囲を`views::all`で包みながら、`cartesian_product_view`を構築して返す

`views::cartesian_product`はRangeアダプタオブジェクトではないため、パイプ（`|`）を使用した入力はサポートされていません。Rangeファクトリ同様に、パイプラインの先頭で使用することになるでしょう。

### `cartesian_product_view`

`views::cartesian_product`に1つ以上の範囲を入力する場合は`cartesian_product_view`が返されます。

```cpp
namespace std::ranges {

  // cartesian_product_viewの宣言例
  template<input_range First, forward_range... Vs>
    requires (view<First> && ... && view<Vs>)
  class cartesian_product_view : public view_interface<cartesian_product_view<First, Vs...>> {
    ...
  };
}
```

入力範囲に関して、1つ目の範囲は`input_range`ですが2つ目以降の範囲は`forward_range`であることが求められています。1つ目だけ制約が異なるのは、`views::cartesian_product`の列挙は1つ目の範囲について行われるためで、1つ目の範囲のある要素が参照されるタイミングは連続的になりますが（1つのイテレータが進むだけで良い）、2つ目以降の範囲の要素は異なるタイミングで複数回参照されるため（マルチパス保証が必要になるため）です。

`cartesian_product_view`は入力範囲をそれぞれ`views::all`に通したものを`std::tuple`に詰めて保持しています。

### 動作詳細

`cartesian_product_view`のイテレータは入力範囲の全てのイテレータを保持しています。この時、そのイテレータの並び順は範囲の入力順に一致し、先頭が1つ目の範囲のイテレータ、一番後ろが最後の入力範囲のイテレータとなっています。

`cartesian_product_view`のイテレータの進行時は、次の`next<N>()`のような関数を入力範囲の数を`N`として呼び出すことで進行（インクリメント）を行います

```cpp
// currentは入力イテレータを保持するstd::tupleオブジェクト
// input_rangesは入力範囲を保持するstd::tupleオブジェクト
template<size_t N>
constexpr void next() {
  // N番目のイテレータを取得
  auto& it = std::get<N>(current);
  // 進める
  ++it;

  if constexpr (N > 0) {
    // N番目のイテレータが終端に到達している場合のみ、N-1番目のイテレータを進める
    if (it == ranges::end(std::get<N>(input_ranges))) {
      it = ranges::begin(std::get<N>(input_ranges));
      next<N - 1>();
    }
  }
}
```

すなわち、保持するイテレータの一番最後のものを1つ進めた後、それが終端に到達していたら先頭にリセットして1つ前のイテレータに対して同様の進行とチェックを行っていきます。

後退時はこれとほぼ逆のことをします（この場合、チェックしてからデクリメントする順番）。また、ランダムアクセス（`+= -=`など）が行われる場合は、進行距離`d`に対して定数時間で`d`回インクリメント/デクリメントしたのと同じ結果になるように指定されています。具体的な実装は規定されていませんが、おそらく、`d`を`N`番目の範囲の長さで割ってその剰余の値で`N`番目のイテレータを進行させ、商の値を長さとして`N - 1`番目も同様に進行させ...のような形で実装されると思われます。

`cartesian_product_view`のイテレータの間接参照では、保持する全てのイテレータに対して`*it`した結果を直接`std::tuple`にラップしてその`std::tuple`オブジェクトを返します。前述のように、ここでは参照は参照のままラップされます。

```cpp
// 入力範囲の保存or参照のみを行う
auto crp = std::views::cartesian_product(rngs...);

// 全ての入力範囲のイテレータを取り出し保存する
auto it = crp.begin();

// 進行時は後のイテレータから進めていく
// 進めたイテレータが終端に達した時のみその1つ前のイテレータを進める
++it;
--it;

// ランダムアクセスも基本は同様に処理される
it += 3;
it -= 3;

// 保持するイテレータ全ての間接参照結果をtupleにラップして返す
auto tuple = *it;

// 保持するイテレータ全ての比較が行われる
// おそらく短絡評価は行われる
bool b = it == crp.end();
```

この実装になっているため、`views::cartesian_product`の要素である順序対内の要素の順番は入力範囲の順番と一致し、要素の列挙はより下位の範囲の要素が先に列挙される形になります。

### `cartesian_product_view`(`views::cartesian_product`)の諸特性

少なくとも1つ以上入力があり、入力範囲型の列（パック）を`Rs`、1つ目の範囲を`R0`とすると次のようになります

- `reference` : `std::tuple<range_reference_t<Rs>...>`
- `range`カテゴリ : 次のいずれか
    - 次の全てを満たす場合 : `random_access_range`
      - `Rs`は全て`random_access_range`
      - `R0`を除いた`Rs`はは全て`sized_range`
    - 次の全てを満たす場合 : `bidirectional_range`
      - `Rs`は全て`bidirectional_range`
      - `R0`を除いた`Rs`は全て次ののどちらか
        - `common_range`
        - `random_access_range`かつ`sized_range`
    - `R0`が`forward_range`の場合 : `forward_range`
    - それ以外の場合 : `input_range`
- `common_range` : `R0`が`common_range`の場合
- `sized_range` : `Rs`が全て`sized_range`の場合
- `const-iterable` : `const Rs`が全て`range`である場合
- `borrowed_range` : ×

`random_access_range`となる場合はランダムアクセスを定数時間で処理するために入力範囲の長さがあらかじめわかっている必要があるため、入力範囲は`sized_range`である必要があります。ただこの場合でも、一番最初の範囲の長さは関係ないので、`R0`だけはそれが求められません。

`bidirectional_range`となる場合も後退をサポートするためにはendイテレータからデクリメントできなければなりませんが、`bidirectional_range`であるが`common_range`ではない場合はそれができないため、定数時間で終端の1つ前を求めるために`random_access_range`かつ`sized_range`であることを求めています。この場合もやはり、`R0`だけはそれが必要ないので求められていません。

## `ranges::range_adaptor_closure`

Rangeアダプタは`views::cartesian_product`で打ち止めで、ここからはRangeアダプタに関連する変更を見ていきます。

`ranges::range_adaptor_closure`はRangeアダプタを自作する場合に標準アダプタとの間でパイプライン演算子（`|`）による接続が行えるようにするためのサポートを提供するクラス型です。

```cpp
namespace std::ranges {
  template<class D>
    requires is_class_v<D> && same_as<D, remove_cv_t<D>>
  class range_adaptor_closure { }; 
}
```

`range_adaptor_closure`はCRTPによって継承して使用し、Rangeアダプタの実装詳細の`view`型ではなくRangeアダプタオブジェクトの型で使用するものです。

オリジナルのRangeアダプタである`my_adaptor`を実装しようとしているとします。パイプのための考慮を一切しない場合、そのままだとパイプライン演算子を使用できません

```cpp
using namespace std::ranges;

// 自作のRangeアダプタの実装view型
template<view V>
class my_view : public view_interface<my_view<V>> {
  ...
};

// 自作のRangeアダプタ型
class my_original_adaptor {
  
  template<viewable_range R>
  view auto operator()(R&& rng) const {
    return my_view(std::forward<R>(rng));
  }
};

// Rangeアダプタであるmy_adaptorの定義
inline constexpr my_original_adaptor my_adaptor{};


int main() {
  std::vector vec = {...};

  // パイプでrangeを入力できない
  view auto v1 = vec | my_adaptor; // ng
  view auto v2 = vec | views::take(2) | my_adaptor;  // ng

  // Rangeアダプタの合成もできない
  auto raco1 = views::take(2) | my_adaptor;  // ng
  auto raco2 = my_adaptor | views::take(2);  // ng
}
```

これを有効化するためには`my_original_adaptor`クラスに`operator|`を実装すればいいのですが、1つ目の`range`の入力はともかく、2つ目の他のアダプタとの合成に関してはどうすればいいのかわかりませんし、実際そのための統一的な方法はC++20時点では提供されていません。

`ranges::range_adaptor_closure`はこの問題を解決するために導入されたもので、これをRangeアダプタ実装クラスで継承するだけで上記2つのタイプのパイプサポートを有効化できます。この例では、`my_original_adaptor`で使用します。

```cpp
#include <ranges>
using namespace std::ranges;

// 自作のRangeアダプタの実装view型
template<view V>
class my_view : public view_interface<my_view<V>> {
  ...
};

// 自作のRangeアダプタ型
// CRTPによってrange_adaptor_closureを継承する
class my_original_adaptor : range_adaptor_closure<my_original_adaptor> {

  template<viewable_range R>
  view auto operator()(R&& rng) const {
    return my_view(std::forward<R>(rng));
  }
};

// Rangeアダプタであるmy_adaptorの定義
inline constexpr my_original_adaptor my_adaptor{};


int main() {
  std::vector vec = {...};

  // パイプでrangeを入力できるようになる
  view auto v1 = vec | my_adaptor; // ok
  view auto v2 = vec | views::take(2) | my_adaptor;  // ok

  // Rangeアダプタの合成も有効化される
  auto raco1 = views::take(2) | my_adaptor;  // ok
  auto raco2 = my_adaptor | views::take(2);  // ok
}
```

ただし、`range_adaptor_closure`が行うのはRangeアダプタクロージャ（オブジェクト）と呼ばれるタイプのRangeアダプタに対してパイプライン演算子のサポートを追加することだけです。そうではないものはRangeアダプタ（オブジェクト）と呼ばれ、こちらの実装のためには入力範囲以外の追加の引数を予め受けておいてそれをラップしたRangeアダプタクロージャオブジェクトを生成する、という処理が必要になります。

Rangeアダプタクロージャオブジェクトとは、そのRangeアダプタの動作のために追加の引数を必要とせずあとは`range`を入力されるだけ、というタイプのものです。標準のものだと例えば`views::reverse`や`views::join`等があります。そうではないRangeアダプタ（オブジェクト）は標準には例えば`views::filter`や`views::transform`等があります。

```cpp
#include <ranges>
using namespace std::ranges;

int main() {
  std::vector vec = {...};
  
  // views::reverseはRangeアダプタクロージャオブジェクト
  view auto v1 = vec | views::reverse;

  // 追加の引数が必要なものはRangeアダプタオブジェクト
  view auto v2 = vec | views::take(5);
  view auto v3 = vec | views::filter([](auto v) { ... });
  view auto v4 = vec | views::transform([](auto v) { ... });

  // Rangeアダプタオブジェクトに追加の引数を予め充填したものはRangeアダプタクロージャオブジェクト
  auto raco1 = views::take(5);
  auto raco2 = views::filter([](auto v) { ... });
  auto raco3 = views::transform([](auto v) { ... });

  // パイプライン演算子はRangeアダプタクロージャオブジェクトに対して作用する
  view auto v5 = vec | raco1; // v2と同じ意味
  view auto v6 = vec | raco2; // v3と同じ意味
  view auto v7 = vec | raco3; // v4と同じ意味
  auto raco4 = raco1 | raco2 | raco3;
}
```

### `std::bind_back`

とはいえRangeアダプタのためのサポートはほったらかしかというとそういうことはなく、Rangeアダプタの処理のために`std::bind_back()`が用意されます。`std::bind_back()`は`std::bind_front()`の逆順版のものです。

`std::bind_front()`は、呼び出し可能なものとその引数の先頭部分を受けてそれをラップしたオブジェクトを返し、そのオブジェクトに残りの引数を与えて呼び出すと`bind_front`に渡した引数を先頭にして追加の引数をその後に並べた形で元の呼び出し可能なオブジェクトに渡して呼び出しを行います。

`std::bind_back()`は、呼び出し可能なものとその引数の末尾部分受けてそれをラップしたオブジェクトを返し、そのオブジェクトに残りの引数を与えて呼び出すと追加の引数を先頭にして`bind_back`に渡した引数をその後に並べた形で元の呼び出し可能なオブジェクトに渡して呼び出しを行います。

```cpp
int main() {
  auto print4 = [](auto&&... args) {
    static_assert(sizeof...(args) == 4);

    std::cout << std::format("{}, {}, {}, {}\n", args...);
  };

  auto bf = std::bind_front(print4, 20, 23);
  auto bb = std::bind_back(print4, 20, 23);

  std::cout << "bind_front : ";
  bf('C', "++");
  std::cout << "bind_back  : ";
  bb('C', "++");
}
```
```{style=planetext}
bind_front : 20, 23, C, ++
bind_back  : C, ++, 20, 23
```

これを用いるとRangeアダプタの処理である追加の引数の順番を保った保存の部分を委任することができます。

```cpp
using namespace std::ranges;

// 自作のRangeアダプタクロージャ型
template<typename F>
struct my_closure_adaptor : range_adaptor_closure<my_closure_adaptor<F>> {
  F f;

  template<viewable_range R>
  view auto operator()(R&& input) const {
    // bind_back()でラッピングされたfに引数のrangeを入力しRangeアダプタを実行する
    return f(input);
  }
};

// 自作のRangeアダプタ型（not クロージャ）
class my_original_adaptor {

  ...
  
  template<typename... Args>
  view auto operator()(viewable_range auto&& input, Args&&... args) const {
    // 入力rangeと必要な引数がすべてそろった状態で、Rangeアダプタを実行し結果のviewを返す
    return my_view(std::forward<R>(input, std::forward<Args>(args)...));
  }
  
  // 追加の引数を受けてRangeアダプタクロージャオブジェクトを返す
  template<typename... Args>
  auto operator()(Args&&... args) const {
    // bind_back()で自身と追加の引数をラッピングし、Rangeアダプタクロージャオブジェクトを作成
    return my_closure_adaptor{.f = std::bind_back(*this, std::forward<Args>(args)...)};
  }
};

inline constexpr my_original_adaptor my_adaptor{};

int main() {
  std::vector vec = {...};

  // my_original_adaptorが追加の引数として整数を1つ受け取るとすると
  view auto v1 = vec | my_adaptor(1); // ok
  view auto v2 = vec | views::take(2) | my_adaptor(2);  // ok
  auto raco = my_adaptor(1);  // ok
  view auto v3 = vec | raco;  // ok、v1と同じ意味
}
```

Rangeアダプタの実装においてはおそらく、ほとんどの場合この例のように入力範囲も含めた必要なものをすべて受けてRangeダプタとしての処理を実行する関数呼び出し演算子と、追加の引数だけを受けて対応するRangeアダプタクロージャオブジェクトを作成する関数呼び出し演算子の2つを記述することになると思われます。

おそらく、この`my_closure_adaptor`のより汎用なもの（汎用的なRangeアダプタクロージャオブジェクト型）はRangeアダプタとは別で作成できて、かつこの例のような典型的な実装になるはずです。そのため、自作のRangeアダプタ毎にこのようなものを作る必要は無いでしょう。この部分がC++23で追加されなかったのは、この部分の最適な実装がまだ確立されていないためだったようです。

## Rangeアダプタのムーブオンリータイプへの対応

C++20のRangeアダプタのうち、ユーザー定義の型を追加で受けるものはその受ける型に対して`copy_constructible`であることを要求しており、これによってムーブオンリーな型の利用が妨げられていました。

これはC++20に導入されるよりも前の`view`コンセプトの要件によっていたようで、その要件が緩和された後にもそのままになっていたようです。

そのため、それらのRangeアダプタ（の`view`型）の要件が緩和されます。影響を受けるのは次のものです

- `single_view`
- `transform_view`
- `take_while_view`
- `drop_while_view`

これらの`view`型ではムーブオンリーな型の値を渡して保持させることができるようになり、その場合その`view`もムーブオンリーになります。

# Rangeアルゴリズム

## `find_last`系アルゴリズム

`find_last`系アルゴリズムは`find`系のアルゴリズムに対して逆の探索を行うもので、範囲の末尾から指定された条件によって探索を行います。

追加されるのは次の3つです

- `ranges::find_last`
    - 指定された値を範囲の末尾から検索する
- `ranges::find_last_if`
    - 条件を満たす最後の要素を検索する
- `ranges::find_last_if_not`
    - 条件を満たさない最後の要素を検索する

```cpp
namespace std::ranges {
  template<forward_range R, class T, class Proj = identity>
    requires
      indirect_binary_predicate<ranges::equal_to, projected<iterator_t<R>, Proj>, const T*>
  constexpr borrowed_subrange_t<R>
    find_last(R&& r, const T& value, Proj proj = {});

  template<forward_range R, class Proj = identity,
           indirect_unary_predicate<projected<iterator_t<R>, Proj>> Pred>
  constexpr borrowed_subrange_t<R>
    find_last_if(R&& r, Pred pred, Proj proj = {});

  template<forward_range R, class Proj = identity,
           indirect_unary_predicate<projected<iterator_t<R>, Proj>> Pred>
  constexpr borrowed_subrange_t<R>
    find_last_if_not(R&& r, Pred pred, Proj proj = {});
}
```

ここでは`range`を受け取るものしか示していませんが、イテレータペアを受け取るオーバーロードも用意されています。

使用感は`find`系アルゴリズムと同様になりますが戻り値だけが異なっており、`find_last`系アルゴリズムは戻り値として末尾の情報も含めて返します。すなわち、終端のイテレータを`end`、指定されたものを見つけた場合にその位置を指すイテレータを`it`とすると

- 指定されたものを見つけた場合 : `[it, end)`
- 何も見つからなかった場合 : `[end, end)`

のような`subrange`を戻り値として返します。

```cpp
using namespace std::ranges;

int main() {
  std::vector vec = {1, 2, 3, 4, 5, 6};

  auto is_even = [](int n) { return n % 2 == 1; };

  range auto res1 = find_last(vec, 4);
  range auto res2 = find_last_if(vec, is_even);
  range auto res3 = find_last_if_not(vec, is_even);

  std::cout << std::format("{} : {}\n", res1.front(), res1.size());
  std::cout << std::format("{} : {}\n", res2.front(), res2.size());
  std::cout << std::format("{} : {}\n", res3.front(), res3.size());
  
  // 見つからない場合
  range auto res4 = find_last(vec, 10);

  std::cout << std::format("  : {}\n", res4.size());
}
```
```{style=planetext}
4 : 3
5 : 2
6 : 1
  : 0
```

`subrange`は`.empty()`関数及び`operator bool`を備えているため、それらを用いると簡易に結果の判定を行うことができます。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4, 5, 6};

  range auto res = find_last(vec, 10);

  // 見つかったかどうかを判定
  if (!res.empty()) { ... }
  if (res) { ... }
}
```

`.empty()`は空の（見つからない）場合に`true`を返すのに対して、`operator bool`は空の場合に`false`を返します。少し注意が必要かもしれません。

なお、これらのアルゴリズムは`ranges`版のものしか用意されておらず、`std::find_last`などは利用できません。

## `ranges::starts_with`/`ranges::ends_with`

`ranges::starts_with`及び`ranges::ends_with`はC++20で`std::string/std::string_view`に追加された同名のメンバ関数をベースとして一般化したもので、範囲の先頭（末尾）が指定された範囲で始まっている（終わっている）かを判定するものです。どちらの関数も判定結果は`bool`値で得られます。

```cpp
namespace std::ranges {

  template<input_range R1, input_range R2, class Pred = ranges::equal_to,
           class Proj1 = identity, class Proj2 = identity>
    requires indirectly_comparable<iterator_t<R1>, iterator_t<R2>,
                                   Pred, Proj1, Proj2>
  constexpr bool starts_with(R1&& r1, R2&& r2, Pred pred = {},
                             Proj1 proj1 = {}, Proj2 proj2 = {});

  template<input_range R1, input_range R2, class Pred = ranges::equal_to,
           class Proj1 = identity, class Proj2 = identity>
    requires (forward_range<R1> || sized_range<R1>) &&
             (forward_range<R2> || sized_range<R2>) &&
             indirectly_comparable<iterator_t<R1>, iterator_t<R2>,
                                   Pred, Proj1, Proj2>
  constexpr bool ends_with(R1&& r1, R2&& r2, Pred pred = {},
                           Proj1 proj1 = {}, Proj2 proj2 = {});
}
```

ここでは`range`を受け取るものしか示していませんが、イテレータペアを受け取るオーバーロードも用意されています。また、これらのアルゴリズムは`ranges`版のものしか用意されていません。

```cpp
using namespace std::ranges;

int main() {
  range auto input = views::iota(0, 50);

  bool res1 = starts_with(input, views::iota(0, 10));
  bool res2 = ends_with(input, views::iota(30, 50));

  std::cout << std::format("{:s}\n{:s}\n", res1, res2);

  std::list list = {2, 4, 6};

  bool res3 = starts_with(input, list);
  bool res4 = ends_with(input, list);

  std::cout << std::format("{:s}\n{:s}\n", res3, res4);
}
```
```{style=planetext}
true
true
false
false
```

`std::string`の`.starts_with()/.ends_with()`と異なる点として、比較を行う方法をカスタマイズできるようになっています。

```cpp
using namespace std::ranges;

int main() {
  range auto input = views::iota(0, 50);

  bool res1 = starts_with(input, std::deque{-1, 0, 1, 2, 3}, greater{});
  bool res2 = ends_with(input, std::array{61, 62, 49, 50}, less{});

  std::cout << std::format("{:s}\n{:s}\n", res1, res2);
}
```
```{style=planetext}
true
true
```

```cpp
using namespace std::ranges;
using namespace std::string_view_literals;

constexpr auto to_upper(char8_t c) -> char8_t {
  if (u8'a' <= c && c <= u8'z') {
    constexpr auto diff = u8'A' - u8'a';
    return c + diff;
  }

  return c;
}

constexpr auto cmp_alpha(char8_t l, char8_t r) -> bool {
  return to_upper(l) == to_upper(r);
}

int main() {
  std::u8string str = u8"Meteor Lake";

  bool res1 = starts_with(str, u8"METEOR"sv, cmp_alpha);
  bool res2 = ends_with(str, u8"lake"sv, cmp_alpha);

  std::cout << std::format("{:s}\n{:s}\n", res1, res2);
}
```
```{style=planetext}
true
true
```

さらに、他のRangeアルゴリズム同様に射影もサポートしています。

計算量は、1つ目の入力範囲の要素数を`L`とすると

- `starts_with` : 高々`L`回の述語の適用
- `ends_with` : 2つ目の範囲の要素数を`M`として
    - `L < M` : 何もしない
    - それ以外の場合 : 高々`min(L, M)`回の述語の適用
      - 1つ目の範囲の長さが定数時間で求められない場合、加えて`L`回のイテレータのインクリメント
      - 2つ目の範囲の長さが定数時間で求められない場合、加えて`M`回のイテレータのインクリメント

## `ranges::contains`/`ranges::contains_subrange`

`ranges::contains`及び`ranges::contains_subrange`はC++20で連想コンテナに対して追加され、C++23で`std::string/std::string_view`に追加された同名のメンバ関数をベースとして一般化したもので、範囲に指定された値、あるいは指定された範囲が含まれているかを判定するものです。どちらの関数も判定結果は`bool`値で得られます。

```cpp
namespace std::ranges {

  template<input_range R, class T, class Proj = identity>
    requires indirect_binary_predicate<ranges::equal_to, 
                                       projected<iterator_t<R>, Proj>, 
                                       const T*>
  constexpr bool contains(R&& r, const T& value, Proj proj = {});

  template<forward_range R1, forward_range R2,
           class Pred = ranges::equal_to,
           class Proj1 = identity, class Proj2 = identity>
    requires indirectly_comparable<iterator_t<R1>, iterator_t<R2>, 
                                   Pred, Proj1, Proj2>
  constexpr bool contains_subrange(R1&& r1, R2&& r2,
                                   Pred pred = {}, 
                                   Proj1 proj1 = {}, Proj2 proj2 = {});
}
```

ここでは`range`を受け取るものしか示していませんが、イテレータペアを受け取るオーバーロードも用意されています。また、これらのアルゴリズムも`ranges`版のものしか用意されていません。

```cpp
using namespace std::ranges;

int main() {
  std::vector vec = {6, 2, 7, 5. 1};
  std::array arr = {2, 7};
  
  bool res1 = contains(vec, 5);
  bool res2 = contains_subrange(vec, arr);

  std::cout << std::format("{:s}\n{:s}\n", res1, res2);

  std::list list = {2, 4, 6};

  bool res3 = contains(vec, 0);
  bool res4 = contains_subrange(vec, list);

  std::cout << std::format("{:s}\n{:s}\n", res3, res4);
}
```
```{style=planetext}
true
true
false
false
```

これらのアルゴリズムは`find`や`search`などの既存アルゴリズムと全く同じことをします。その違いは戻り値が直接`bool`で得られるかにあります。`find`や`search`だと、結果がイテレータや`subrange`で得られるため、見つかるかだけが重要で見つかったものにアクセスする必要がない場合にその結果の取得が冗長になり、コードの意図を読み取りづらくします。

```cpp
using namespace std::ranges;

int main() {
  std::vector vec = {6, 2, 7, 5. 1};
  std::array arr = {2, 7};
  
  bool res1 = find(vec, 5) != vec.end();
  bool res2 = !search(vec, arr).empty();
}
```

`contains`及び`contains_subrange`を代わりに使用することで、冗長な記述を排除しながらその意図を明確にすることができます。

連想コンテナ等の同名メンバ関数と異なる点としては、射影がサポートされている点と`contains_subrange`では比較を行う方法をカスタマイズすることができる点があります。

```cpp
using namespace std::ranges;

int main() {
  std::map<std::u8string, int> map = { {u8"１", 1}, {u8"２", 2}, {u8"３", 3} };
  std::array arr = {0, 1};

  // valueを取り出す
  auto proj = [](auto& p) -> decltype(auto) { return std::get<1>(p); };

  bool res1 = contains(map, 2, proj);
  bool res2 = contains_subrange(map, arr, greater{}, proj);

  std::cout << std::format("{:s}\n{:s}\n", res1, res2);
}
```
```{style=planetext}
true
true
```

なお、`std::map`等要素が`std::pair`ないし`std::tuple`となっているものから特定のインデックスの要素だけを取り出したい（この例のような）場合は、射影よりも`views::keys/views::values`（`views::elements`）を使用した方が簡易かもしれません。

計算量は、入力範囲の要素数を`L`とすると

- `contains` : 高々`L`回の比較
- `contains_subrange` : 2つ目の範囲の要素数を`M`とすると
    - 高々`L * M`回の比較もしくは述語の適用

## `ranges::iota`

`ranges::iota`は以前からあった`std::iota`のRangeバージョンです。

```cpp
// <numeric>で定義される
namespace std::ranges {
  
  template<weakly_incrementable T, output_range<const T&> R>
  constexpr iota_result<borrowed_iterator_t<R>, T> iota(R&& r, T value);
}
```

指定された範囲`r`に`value`から始まり1づつ増加（インクリメント）していく単調増加列を書き込むものです。

```cpp
int main() {
  // 0から始まる10要素のシーケンスを作成する。
  std::array<int, 10> ar;
  const auto [it, v] = std::ranges::iota(ar, 0);

  for (int x : ar) {
    std::cout << x;
  }

  std::cout << std::format("\n{:s}\n{:d}", it == ar.end(), v);
}
```
```{style=planetext}
0123456789
true
10
```

戻り値は、書き込んだ範囲の終端位置（入力範囲終端）を指すイテレータと書き込むために計算した値の最終値（書き込んだ値の1つ次）のペアとなる構造体（集成体）です。

`std::iota`に対してはコンセプトによる制約がなされていることや`range`を直接指定できるところなどがメリットであり、`views::iota`に対しては書き込む要素数を事前計算する必要が無くなり（入力範囲の要素数によって自動で決定できるため）効率的になることなどがメリットとなります。

このアルゴリズムは`views::iota`と役割が被っていることもありC++20では導入されませんでしたが、このメリットが相違点であるとして追加されました。

計算量は、入力範囲の要素数を`L`とすると正確に`L`回のインクリメントと代入（コピー）が行われます。

## `ranges::shift_left`/`ranges::shift_right`

`ranges::shift_left`及び`ranges::shift_right`はC++20で追加された同名のアルゴリズムのRange版です。

```cpp
namespace std::ranges {
  
  template<forward_range R>
    requires permutable<iterator_t<R>>
  constexpr borrowed_subrange_t<R> shift_left(R&& r, range_difference_t<R> n);

  template<forward_range R>
    requires permutable<iterator_t<R>>
  constexpr borrowed_subrange_t<R> shift_right(R&& r, range_difference_t<R> n);
}
```

ここでは`range`を受け取るものしか示していませんが、イテレータペアを受け取るオーバーロードも用意されています。どうやら、C++23以降に新しいアルゴリズムを追加する場合は基本的にRange版のみが追加されるようです。

入力範囲`r`の要素を`n`だけ左/右シフト（非循環シフト）させます。元のアルゴリズム同様に、`n`は0以上の整数である必要があります（違反すると未定義動作）。

```cpp
using std::ranges::subrange;

int main() {

  std::vector vec = {1, 2, 3, 4, 5};
  
  // 戻り値はシフト後の終端位置
  range auto res = shift_left(vec, 2);

  for (int n : res) {
    std::cout << std::format("{} ", n);
  }
  
  std::cout << '\n';

  // 戻り値はシフト後の開始位置
  range auto res = shift_right(vec, 2);

  for (int n : res) {
    std::cout << std::format("{} ", n);
  }
}
```
```{style=planetext}
3 4 5 
1 2 3 
```

戻り値はシフト後の範囲を示す`subrange`であり、入力範囲のサイズを`L`とすると次のようになります

- `L == 0` : 空
- `0 <= n < L` : シフト後の範囲（シフトした後に残った有効な要素列）
- `L < n` : 空

空の範囲を返す（`n`が0もしくは入力範囲のサイズよりも大きい）場合はこの関数は何もせず入力範囲は変更されません。

```cpp
using std::ranges::subrange;

int main() {
  std::vector vec = {1, 2, 3, 4, 5};
  
  range auto res1 = shift_left(vec, 7);
  range auto res2 = shift_right(vec, 7);
  range auto res3 = shift_left(vec, 0);
  range auto res4 = shift_right(vec, 0);

  std::cout << std::format("{:s}\n", res1.empty());
  std::cout << std::format("{:s}\n", res2.empty());
  std::cout << std::format("{:s}\n", res3.empty());
  std::cout << std::format("{:s}\n", res4.empty());
}
```
```{style=planetext}
true
true
true
true
```

この2つのアルゴリズムがC++20時点で導入されなかったのは、`ranges::shift_left`の戻り値についての議論が紛糾したためのようです。`ranges::shift_left`の場合、処理の過程で範囲終端を計算する必要がありますが、それは戻り値には必ずしも反映されません。`common_range`ではない範囲の場合はその終端情報が有効活用できる場合があるため、返すべきかが議論されましたが結局返さないことで決着しました。それによって、両アルゴリズムで戻り値の扱いは一貫しています。

計算量は、入力範囲の要素数を`L`とすると、`shift_left`では高々`L - n`回の代入（ムーブ）が行われ、`shift_right`では高々`L - n`回の代入（ムーブ）あるいは`swap`が行われます。

## `fold`アルゴリズム

`fold`アルゴリズムはいわゆる高階関数としての`fold`関数で、`std::accumulate`のRange版に対応するものです。処理としては、初期値から始めて入力の範囲の各要素に各ステップでの結果を次のステップに渡しながら二項演算を逐次適用していき、その結果を返します。

基本形は初期値を指定するもので、その計算順序によって左（*left*）と右（*right*）の2種類があります。

```cpp
namespace std::ranges {

  // 左方向のfold
  template<input_iterator I, sentinel_for<I> S, class T, 
           indirectly-binary-left-foldable<T, I> F>
  constexpr auto fold_left(I first, S last, T init, F op);
  
  template<input_range R, class T,
           indirectly-binary-left-foldable<T, iterator_t<R>> F>
  constexpr auto fold_left(R&& rng, T init, F op);

  // 右方向のfold
  template<bidirectional_iterator I, sentinel_for<I> S, class T,
           indirectly-binary-right-foldable<T, I> F>
  constexpr auto fold_right(I first, S last, T init, F op);
  
  template<bidirectional_range R, class T,
           indirectly-binary-right-foldable<T, iterator_t<R>> F>
  constexpr auto fold_right(R&& rng, T init, F op);
}
```

他のRangeアルゴリズム同様にイテレータペアを受け取るものと`range`を受け取るものの2つが用意されています。ただし、射影はサポートされていません。

この基本形はどちらも`ranges::fold_xxxx(rng, init, op)`のように呼び出し、戻り値は計算結果の値が得られます。`init`も`op`にもデフォルトの指定はないため、都度指定する必要があります。

```cpp
using namespace std::ranges;

int main() {
  range auto rng = views::iota(1, 11);
  const int init = 0;
  auto op = std::plus<>{};
  
  int res1 = fold_left(rng, init, op);
  int res2 = fold_right(rng, init, op);

  std::cout << std::format("{:d}\n", res1);
  std::cout << std::format("{:d}\n", res2);
}
```
```{style=planetext}
55
55
```

戻り値の型は入力範囲`rng`のイテレータを`it`とすると、`decltype(op(init, *it))`の結果の型を`decay`したものになります。つまりは、二項演算`op`の戻り値型から参照修飾を取り除いたものであり、変なことをしていなければ初期値`init`と同じ型になります。

```cpp
using namespace std::ranges;

int main() {
  std::vector<float> rng = { 0.125f, 0.25f, 0.75f };
  const int init = 1;
  auto op = std::plus<>{};
  
  std::same_as<float> auto res1 = fold_left(rng, init, op);
  std::same_as<float> auto res2 = fold_right(rng, init, op);

  std::cout << std::format("{:g}\n", res1);
  std::cout << std::format("{:g}\n", res2);
}
```
```{style=planetext}
2.125
2.125
```

`std::plus`の場合は入力を`a, b`とすると`decltype(a + b)`の結果型が戻り値型になり、これはどちらかが浮動小数点数型ならその型になります。これによって、`std::accumulate`に存在していた初期値の罠は回避されています。

`fold_left`と`fold_right`の違いは入力範囲をどちらから走査するかの違いであり、入力範囲が左->右の順序で1列になっているとすると、それを左側（先頭）から走査して集計していくのが`fold_left`で、右側（末尾）から集計していくのが`fold_right`です。どちらの場合も初期値はそれぞれの走査開始位置の前に添加される形になります。

```{style=planetext}
0 : init
[1, 2, 3, 4, 5] : rng

fold_leftの処理の様子
0  [1, 2, 3, 4, 5]
0 + 1
  1 + 2
     3 + 3
        6 + 4
          10 + 5 -> fold_left(rng, 0, +)

fold_rightの処理の様子
[1, 2, 3, 4, 5] 0
             5 + 0
          4 + 5
       3 + 9
    2 + 12
 1 + 14 -> fold_right(rng, 0, +)  
```

入力範囲`rng`の各要素を`e0, e1, ..., en`とすると、`fold_left`は`op( op(op(init, e0), e1)... , en)`、`fold_right`は`op(e0, op(e1, ...op(en, init) ))`のような計算を行います。

この動作のために、`fold_right`の入力範囲は`bidirectional_range`であることが要求されます。

```cpp
using namespace std::ranges;

int main() {
  range auto rng = views::iota('a', 'g');
  const std::string init = "init";

  // fold_leftのop
  auto op_left = [](std::string acc, char elem) {
    acc += " -> ";
    acc += elem;
    return acc;
  };
  // fold_rightのop
  auto op_right = [](char elem, std::string acc) {
    acc += " -> ";
    acc += elem;
    return acc;
  };
  
  auto res1 = fold_left(rng, init, op_left);
  auto res2 = fold_right(rng, init, op_right);

  std::cout << std::format("{:s}\n", res1);
  std::cout << std::format("{:s}\n", res2);
}
```
```{style=planetext}
init -> a -> b -> c -> d -> e -> f
init -> f -> e -> d -> c -> b -> a
```

適用する二項演算`op`に渡される引数は初期値もしくは累積値と各要素の値の2つですが、その引数順は`fold_left`と`fold_right`で逆になります。1つ目の例のように入力範囲の要素型と初期値の型が同じなら問題にならないのですが、この例のように異なっていると気を付けなければなりません。

`op`に渡されてくる引数はどちらの関数でも、初期値及び累積値はムーブされ各要素については関節参照の結果が直接渡されます。累積値（初期値）を`acc`、`rng`のイテレータを`it`とすると、`fold_left`の場合は初期値及び各要素に対して各ステップで`op(std::move(acc), *it)`のように呼ばれ、`fold_right`の場合は`op(*it, std::move(acc))`のように呼ばれます（ただし、呼び出しには`std::invoke`が使用されます）。そのため、累積値は値で受ければよく、各要素に関しては基本的に`rng`の参照型で受けることになるでしょう。

また、各ステップでの呼び出し結果は直接累積値`acc`に再代入されます（`acc = op(std::move(acc), *it)`のように呼ばれる）。そのため、戻り値を参照で返したりする必要は無く、その場合でもNRVOが期待でき、悪くても暗黙ムーブは実行されます。このためほとんどの場合、`op`におけるパフォーマンスを気にして戻り値型や`return`文を工夫する必要は無いでしょう。

### 制約について

制約に使用されている`indirectly-binary-left-foldable`とか`indirectly-binary-right-foldable`は説明専用のコンセプトです。

```cpp
namespace std::ranges {

  template<class F, class T, class I, class U>
  concept indirectly-binary-left-foldable-impl =
    movable<T> && movable<U> &&
    convertible_to<T, U> && invocable<F&, U, iter_reference_t<I>> &&
    assignable_from<U&, invoke_result_t<F&, U, iter_reference_t<I>>>;

  template<class F, class T, class I>
  concept indirectly-binary-left-foldable =
    copy_constructible<F> && indirectly_readable<I> &&
    invocable<F&, T, iter_reference_t<I>> &&
    convertible_to<invoke_result_t<F&, T, iter_reference_t<I>>,
           decay_t<invoke_result_t<F&, T, iter_reference_t<I>>>> &&
    indirectly-binary-left-foldable-impl<F, T, I,
                    decay_t<invoke_result_t<F&, T, iter_reference_t<I>>>>;

  template<class F, class T, class I>
  concept indirectly-binary-right-foldable =
    indirectly-binary-left-foldable<flipped<F>, T, I>;
}
```

かなり複雑ですが、ざっくりいえば初期値の型`T`とイテレータ型`I`の間接参照結果によって関数`F`が呼び出し可能で、先ほどの2種類の`fold`処理を行うために必要な操作が可能であること、を要求しています。

`indirectly-binary-right-foldable`は`T`と`I`を入れ替えて`F`に渡すようにした上で`indirectly-binary-left-foldable`に制約を委譲しており、`flipped<F>`は二項呼び出し可能な`F`の引数順序を入れ替える説明専用のクラス型です。

`indirectly-binary-left-foldable`の直接的な制約では、`F`に対して`T`と`I`の間接参照によって呼び出し可能であることやその戻り値型が`fold`処理の戻り値型に変換可能であることをチェックし、`indirectly-binary-left-foldable-impl`では主に、`fold`処理の戻り値型かつ処理中に集計結果を保持している変数の型`U`を用いて、`T -> U`の変換可能性や`F`に`T`の代わりに`U`を渡せるか、`U`の左辺値に各ステップの呼び出し結果を代入できるかなどをチェックしています。

これらのコンセプトは構文的なもので、構成しているそれぞれのコンセプトが持つ意味論要件以上の意味論要件を持っていません。`F, T, I`の入力の型について`fold`処理が可能であることを構文的に制約しているものです。

この制約は`std::accmulate`及び想定されていた`ranges::accmulate`と異なる部分を表現してもいて、`ranges::accmulate`が数値アルゴリズムとして数値範囲に対する集計処理を基本としようとしていたのに対して、`fold_left/fold_right`はより汎用的な範囲に対する二項演算の重畳処理を基本とするようになっています。

例えば、`ranges::accmulate`の想定では、`magma`という代数学における亜群を表現するコンセプトによって制約されていましたが、見ての通り`fold_left/fold_right`にはそういうものは無く、`fold_left/fold_right`は関数型言語におけるリストに対する高階関数としての`foldl/foldr`に対応するものです。

### 初期値を省略する

集計処理の場合などは必ずしも初期値を指定する必要が無い場合もあります。そのための変種が用意されており、`fold_left`と`fold_right`に対してそれぞれ`_first`と`_last`のサフィックスが付きます。

```cpp
namespace std::ranges {

  template<input_range R, 
           indirectly-binary-left-foldable<range_value_t<R>, iterator_t<R>> F>
    requires constructible_from<range_value_t<R>, range_reference_t<R>>
  constexpr auto fold_left_first(R&& rng, F op);

  template<bidirectional_range R,
           indirectly-binary-right-foldable<range_value_t<R>, iterator_t<R>> F>
    requires constructible_from<range_value_t<R>, range_reference_t<R>>
  constexpr auto fold_right_last(R&& rng, F op);
}
```

ここでは`range`を受け取るものしか示していませんが、イテレータペアを受け取るオーバーロードも用意されています。

サフィックスが示す通り、こちらのバージョンではそれぞれ範囲の先頭/末尾の要素を初期値として`fold`していきます。初期値の取り方と戻り値型以外は動作に変わりはありません。

`fold_left/fold_right`では`T`で指定されていた初期値の型は、こちらのバージョンでは入力範囲の値型（`range_value_t<R>`）によって取得されており、`R`の参照型から直接構築可能であることを追加で要求しています。

```cpp
using namespace std::ranges;

int main() {
  range auto rng = views::iota(1, 11);
  auto op = std::plus<>{};

  auto res1 = fold_left_first(rng, op);
  auto res2 = fold_right_last(rng, op);

  std::cout << std::format("{:d}\n", res1.value_or(-1));
  std::cout << std::format("{:d}\n", res2.value_or(-1));
}
```
```{style=planetext}
55
55
```

これらのバージョンの戻り値型は、対応する`fold_left`と`fold_right`の戻り値型を`U`とした時の`std::optional<U>`になります（初期値の型は入力範囲の値型`range_value_t<R>`が使用される）。`std::optional`を使うのは入力範囲が空の場合に値を返せないためで、戻り値が無効値を取るのは入力範囲が空の場合のみです。

なお、`fold_left`/`fold_right`において入力範囲が空の場合、初期値が返されます。

```cpp
using namespace std::ranges;

int main() {
  range auto rng = views::iota(1, 1);
  auto op = std::plus<>{};

  auto res1 = fold_left_first(rng, op);
  auto res2 = fold_right_last(rng, op);
  auto res3 = fold_left(rng, -1, op);
  auto res4 = fold_right(rng, -1, op);

  std::cout << std::format("{:d}\n", res1.value_or(-1));
  std::cout << std::format("{:d}\n", res2.value_or(-1));
  std::cout << std::format("{:d}\n", res3);
  std::cout << std::format("{:d}\n", res4);
}
```
```{style=planetext}
-1
-1
-1
-1
```

先ほどの文字範囲を集計して文字列を作る例は、初期値を指定できないことで出力を`std::string`にすることができなくなるため、少し変更が必要になります。

```cpp
using namespace std::ranges;

int main() {
  // 文字範囲ではなく、文字列範囲にする
  range auto rng = views::iota('a', 'g')
                 | views::transform([](char c) {
                     return std::string(1, c);
                   });
  
  // fold_leftのop
  auto op_left = [](std::string acc, std::string&& elem) {
    acc += " -> ";
    acc += elem;
    return acc;
  };
  // fold_rightのop
  auto op_right = [op_left](std::string&& elem, std::string acc) {
    return op_left(std::move(acc), std::move(elem));
  };
  
  auto res1 = fold_left_first(rng, op_left);
  auto res2 = fold_right_last(rng, op_right);

  std::cout << std::format("{:s}\n", res1.value_or(""));
  std::cout << std::format("{:s}\n", res2.value_or(""));
}
```
```{style=planetext}
a -> b -> c -> d -> e -> f
f -> e -> d -> c -> b -> a
```

`indirectly-binary-left-foldable`に現れているように、二項演算`op`は1つ目の引数として初期値（`range_value_t<R>`）を渡しても累積値（戻り値型`decay_t<invoke_result_t<F&, range_value_t<R>, range_reference_t<R>>>`）を渡してもその両方で呼べる必要があります。初期値の型を指定できないこちらのバージョンでは、累積値の型（戻り値型）は入力範囲の要素型と`op`の戻り値型から決定されるため、入力範囲の型から大きく戻り値型を変化させることが難しくなります。

### 計算終了地点のイテレータを返す

`fold_left`系の処理の場合、範囲の終端で処理を終えるため終端イテレータの計算を同時に行なってもいます。そのため、その計算した終端イテレータが欲しい場合があり、そのための変種も用意されています。

ただし、`fold_right`系で得られるのは先頭のイテレータであり、それは`ranges::begin()`で労せずして得られるためこちらのバージョンに対応するものはありません。

```cpp
namespace std::ranges {

  template<input_range R, class T, 
           indirectly-binary-left-foldable<T, iterator_t<R>> F>
  constexpr auto fold_left_with_iter(R&& rng, T init, F op);

  template<input_range R,
           indirectly-binary-left-foldable<range_value_t<R>, iterator_t<R>> F>
    requires constructible_from<range_value_t<R>, range_reference_t<R>>
  constexpr auto fold_left_first_with_iter(R&& rng, F op);
}
```

ここでは`range`を受け取るものしか示していませんが、イテレータペアを受け取るオーバーロードも用意されています。

`fold_left/fold_left_first`に対して`_with_iter`のサフィックスが付いた形になっており、戻り値型以外の部分は元になった関数と同じことを行います。

こちらのバージョンの戻り値型は、対応する`_with_iter`なしの関数の戻り値型を`U`とすると、`iterator_t<R>`と`U`のペアとなる構造体（集成体）を返します。

```cpp
using namespace std::ranges;

int main() {
  range auto rng = views::iota(1, 11);
  const int init = 0;
  auto op = std::plus<>{};

  auto [end1, res1] = fold_left_with_iter(rng, init, op);
  auto [end2, res2] = fold_left_first_with_iter(rng, op);

  std::cout << std::format("{:d}\n", res1);
  std::cout << std::format("{:d}\n", res2.value_or(-1));
  std::cout << std::format("{:s}\n", end1 == rng.end());
  std::cout << std::format("{:s}\n", end2 == rng.end());
}
```
```{style=planetext}
55
55
true
true
```

正確には`ranges::in_value_result<iterator_t<R>, U>`という構造体で、1つ目のメンバ変数`in`にイテレータを保持し、2つ目のメンバ変数`value`に計算結果の値を保持しています。

## 非Rangeアルゴリズムでのコンセプトの使用


# `range` to `range`の変換

C++20の`<ranges>`は既存コンテナへの出力に関してほぼ互換性が無く、Rangeアダプタなどの結果の`range`から直接コンテナを構築したり挿入したりすることができませんでした。

```cpp
using std::ranges::common_range;

int main() {
  auto in = std::views::iota(0)
          | std::views::take(10);

  std::vector<int> vec(in);  // ng
  vec.insert(vec.end(), in); // ng

  // 終端を指定しないviews::iotaはcommon rangeではない
  static_assert(not common_range<decltype(in)>);

  std::vector<int> vec2(in.begin(), in.end());    // ng
  vec2.insert(vec2.end(), in.begin(), in.end());  // ng

　// common rangeにする
  common_range auto inc = in | std::views::common;

  std::vector<int> vec3(in.begin(), in.end());    // ok
  vec3.insert(vec3.end(), in.begin(), in.end());  // ok
}
```

`std::vector`で代表させましたが全てのコンテナでこの事は変わりません。イテレータペアを取るコンストラクタ/関数はあれど`range`を取るものはなく、イテレータペアを取るものもイテレータと番兵が同じ型でなければ使用できません。

## `std::from_range`

コンテナの`range`対応のためにまず、現在の既存コンテナに存在するイテレータペアを取るコンストラクタに対して、`std::from_range`を先頭に2つ目の引数で`range`オブジェクトを受け取る形のコンストラクタが追加されます。

追加のパターンは全コンテナ（`std::array`を除く）で同様で、大まかに例示すると次のようなコンストラクタが追加されます

```cpp
namespace std {

  // 何らかのコンテナ型とする
  template<typename T ...>
  class C {

    ...

    // 既存のイテレータペアを取るコンストラクタ
    template<class InputIterator>
    C(InputIterator first, InputIterator last, ...);

    // 追加されるfrom_rangeコンストラクタ
    template<container-compatible-range<value_type> R>
    C(from_range_t, R&& rng, ...);

    ...
  };
}
```

`std::from_range_t`は単なるタグ型であり`std::from_range`はその事前定義オブジェクトです。イテレータペアを取るコンストラクタは追加の引数を取る場合がありますが（アロケータや比較関数オブジェクトなど）、その追加の引数の渡し方は変化しません。

このコンストラクタによって、任意の`range`（`input_range`）からコンテナを構築することができます。

```cpp
int main() {
  auto in = std::views::iota(0)
          | std::views::take(10);
  
  // どれも[0, 10)の範囲から初期化される
  std::vector vec(std::from_range, in); // ok
  std::list lst(std::from_range, in);   // ok
  std::forward_list flst(std::from_range, in); // ok
  std::set set(std::from_range, in);    // ok
  std::stack stk(std::from_range, in);  // ok

  std::pmr::monotonic_buffer_resource mr{};

  // 追加の引数は入力範囲の後ろに指定
  std::pmr::vector<int> pmrvec(std::from_range, in, &mr); // ok

  // std::stringにもある
  std::string str(std::from_range, std::views::iota('a', 'z')); // ok
}
```

`from_range`コンストラクタでもCTADが利用可能になるように推論補助も追加されているため、このように要素型を自動で補ってもらうこともできます。

これらのコンストラクタの制約に使用されている`container-compatible-range<R, T>`は説明専用のコンセプトで、入力範囲`R`の参照型がコンテナの要素型`T`に変換可能であることを要求するものです。

```cpp
namespace std {

  template<class R, class T>
  concept container-compatible-range =
    ranges::input_range<R> &&
    convertible_to<ranges::range_reference_t<R>, T>;
}
```

つまりは`from_range`コンストラクタでは、入力範囲の要素型からコンテナの要素型に暗黙変換可能であればその入力範囲から構築することができます。

`container-compatible-range<R, T>`の`T`は大抵の場合コンテナ型`C<T>`の`T`に一致しますが、`std::map`等の連想コンテナではそうではありません。その場合は、入力範囲の要素型に注意する必要があります。

```cpp
int main() {
  // 要素型はstd::pair<int, std::string>
  auto in = std::views::iota('0')
          | std::views::transform([](char c) {
              return std::make_pair(int(c), std::string(1, c));
            })
          | std::views::take(10);
  
  // どちらもKey=int、Value=std::string
  std::map map(std::from_range, in);  // ok
  std::unordered_map umap(std::from_range, in); // ok
}
```

`std::map<K, V>`の要素型は`std::pair<const K, V>`であり、`K, V`を推論させようと思ったら`from_range`コンストラクタの入力範囲の要素型は少なくとも`std::pair`である必要があります。推論させないのであれば、入力範囲の要素型は`std::pair<const K, V>`に変換可能であればokです。

```cpp
int main() {
  // 要素型はstd::tuple<size_f, std::string>
  auto in = std::views::iota('0')
          | std::views::transform([](char c) {
              return std::string(1, c);
            })
          | std::views::take(10)
          | std::views::enumerate;
  
  // CTADする場合は入力要素型はstd::pairでなければならない
  std::map map1(std::from_range, in);  // ng
  std::unordered_map umap1(std::from_range, in); // ng

  // CTADしないなら2要素std::tupleでも可
  std::map<std::size_t, std::string> map2(std::from_range, in);  // ok
  std::unordered_map<std::size_t, std::string> umap2(std::from_range, in); // ok
}
```

なお、要素は全てコピーして構築されます。入力範囲が右辺値であっても同様です。ムーブしたい場合は`views::as_rvalue`を適用してから渡すと良いでしょう。

コンテナの`range`コンストラクタが`std::from_range`というタグを取るのは、次のようなコードにおいて構築後の要素型が変化してしまうことを回避するためです

```cpp
std::list<int> l;
std::vector v{l}; // 要素型は何？
```

このコードはC++20でも有効であり、`v`の型は`std::vector<std::list<int>>`になります。`std::from_range`タグを取ることで、この問題を回避しています。

### 計算量について

`from_range`コンストラクタにおける計算量は、入力範囲`rng`の型を`R`、長さを`N`とすると次のように指定されています

- シーケンスコンテナ
    - `std::vector`の場合
      - $O(N)$回の要素コピー（ムーブ）
      - `R`が`forward_range`であるか`sized_range`の場合、メモリ割り当ては1度だけ行われる
        - そうではない場合、$log N$回のメモリ再割り当てが行われる
    - それ以外のコンテナの場合
      - $O(N)$回の要素コピー（ムーブ）
- 連想コンテナ
    - 順序付き連想コンテナ
      - `rng`がソートされている場合$O(N)$
      - そうでないなら$O(N log N)$
    - 非順序連想コンテナ
      - 平均$O(N)$、最悪$O(N^2)$

おおよそ対応するイテレータペアを取るコンストラクタと同じになっています。

ただし、`std::vector`の場合はメモリ確保についての要件が指定されており、これは予め`reserve`してから要素の挿入を行うことを意味しています。

## `_range`系関数

コンテナの`range`対応のために次に、構築済みのコンテナへ要素を挿入する関数の`range`対応版としてサフィックスに`_range`の付くメンバ関数が追加されます。これも`from_range`コンストラクタと同様に、現在イテレータペアを取る挿入系関数のオーバーロードをベースとして、イテレータペアを`range`に置き換えたような関数として追加されます。

追加される関数とそのもとになった関数は次のようになっています

|新しい関数|元の関数|意味|
|---|---|---|
|`insert_range`|`insert`|指定位置の前への範囲挿入|
|`insert_range_after`|`insert_after`|指定位置の後ろへの範囲挿入|
|`append_range`|なし（近いのは`push_back`）|末尾への範囲挿入|
|`prepend_range`|なし（近いのは`push_front`）|先頭への範囲挿入|
|`assign_range`|`assign`|コンテナへの範囲代入|
|`replace_with_range`|`replace`|指定範囲の文字列置換|
|`push_range`|なし（近いのは`push`）|末尾への範囲挿入|

これらの関数は全てのコンテナに対して追加されるわけではなく、それぞれ次のコンテナで追加され使用可能になります

- `insert_range`
    - `std::vector`
    - `std::deque`
    - `std::list`
    - `std::forward_list`
    - `std::basic_string`
    - 全ての連想コンテナ
- `insert_range_after`
    - `std::forward_list`
- `append_range`
    - `std::vector`
    - `std::deque`
    - `std::list`
    - `std::basic_string`
- `prepend_range`
    - `std::deque`
    - `std::list`
    - `std::forward_list`
- `assign_range`
    - `std::vector`
    - `std::deque`
    - `std::list`
    - `std::forward_list`
    - `std::basic_string`
- `replace_with_range`
    - `std::basic_string`
- `push_range`
    - `std::queue`
    - `std::priority_queue`
    - `std::stack`

`std::basic_string`はコンテナではありませんが、`std::vector`と同じ性質を持っているので追加対象になっています。

以下特に明記しませんが、これらの関数では挿入先コンテナの要素型`T`と入力範囲`R`について、`from_range`コンストラクタの時の`container-compatible-range<R, T>`を満たすようになっていなければ挿入できません。

### `insert_range`/`insert_range_after`

`.insert_range()`は`.insert()`同様に、指定した位置（の後ろ）に指定した範囲を挿入するものです。

```cpp
using std::ranges::next;

int main() {
  std::vector vec = {1, 2, 9, 10};

  auto rng = std::views::iota(3, 9);

  auto vit = vec.insert_range(next(vec.begin(), 2), rng); // ok
  // vec : {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
  // *vit : 3
}
```

`.insert_range()`の戻り値は、挿入した位置（挿入した範囲の先頭要素の挿入後コンテナ内での位置）を指すイテレータを返します。`deque`や`list`でも同様ですが、`forward_list`はその特有の事情により少し異なります。

```cpp
using std::ranges::next;

int main() {
  std::vector vec = {1, 2, 9, 10};

  auto rng = std::views::iota(3, 9);

  std::deque deq(std::from_range, vec);
  std::list lst(std::from_range, vec);

  auto dit = deq.insert_range(next(deq.begin(), 2), rng); // ok
  auto lit = lst.insert_range(next(lst.begin(), 2), rng); // ok
  // 戻り値のイテレータ及び挿入後範囲はvectorの場合と同じ

  std::forward_list fwl(std::from_range, vec);
  
  // 指定位置の前に挿入
  auto flit = fwl.insert_range_after(next(fwl.begin()), rng); // ok
  // 挿入後範囲はvectorの場合と同じ
  // *flit : 8
}`
```

`forward_list`の場合、戻り値のイテレータは挿入した要素の最後のものへのイテレータになります。

連想コンテナ系の場合は、元の`.insert()`からして他のコンテナとは少し異なっており、`.insert_range()`もそれに準じています。

```cpp
int main() {
  std::map<char, int> map = {{'1', 1}, {'2', 2}, {'5', 5}};
  std::unordered_map<char, int> ump(std::from_range, map);

  std::vector<std::pair<char, int>> rng = {{'3', 3}, {'4', 4}};

  // 位置指定なし、戻り値なし
  map.insert_range(rng);  // ok
  // map : {'1': 1, '2': 2, '3': 3, '4': 4, '5': 5}

  ump.insert_range(rng);  // ok
  // 挿入後範囲はmapと同じ

  std::set set = {2, 1, 5};
  std::unordered_set ust(std::from_range, set);

  rng2 = rng | std::views::values;

  set.insert_range(rng2); // ok
  ust.insert_range(rng2); // ok
  // set : {1, 2, 3, 4, 5}
  // ustは順不定
}
```

これらの関数の計算量は、元となった`.insert()`のイテレータ範囲をとるオーバーロードと同じになります。

なお、入力範囲が右辺値であっても挿入される要素は全てコピーされています。ムーブしたい場合は`views::as_rvalue`を適用するなどして要素を右辺値にして渡す必要があります。

### `append_range`/`prepend_range`

`.append_range()`/`.prepend_range()`は、それぞれコンテナの終端/先頭（の手前）に指定した範囲を挿入するものです。これらの関数には完全に対応する関数はありませんが、`.push_back()`や`.push_front()`が単一要素の代わりに範囲をまとめて挿入するようになったものです。

```cpp
int main() {
  std::deque deq = {1, 2, 3};

  auto append = std::views::iota(-3, 0);
  auto prepend = std::views::iota(4)
               | std::views::take(3);

  // 戻り値なし
  deq.append_range(append);
  deq.prepend_range(prepend);
  // deq : {-3, -2, -1, 0, 1, 2, 3, 4, 5, 6}
}
```

この2つの関数は動作に伴って挿入しようとする範囲の順序を並べ替えません。特に、`.prepend_range()`において挿入する範囲を逆転させてから挿入したりしません。

`.prepend_range()`が使用できるのは`deque, list, forward_list`だけで、`vector, string`は`.append_range()`のみが使用できます。

```cpp
int main() {
  std::vector vec = {1, 2, 3};
  std::list lst(std::from_range, vec);
  std::forward_list fwl(std::from_range, vec);

  auto append = std::views::iota(-3, 0);
  auto prepend = std::views::iota(4)
               | std::views::take(3);

  vec.append_range(append); // ok
  lst.append_range(append); // ok
  fwd.append_range(append); // ok
  
  vec.prepend_range(append); // ng

  lst.prepend_range(append); // ok
  fwd.prepend_range(append); // ok
}
```

これらの関数の計算量は、挿入される範囲の長さを`N`とすると次のようになります

- `std::vector` : 挿入前のコンテナサイズを`M`とすると
    - メモリ再割り当てが行われた場合 : `N + M`に対して線形
    - メモリ再割り当てが行われない場合 : `N`に対して線形
- それ以外 : `N`に対して線形

なお、入力範囲が右辺値であっても挿入される要素は全てコピーされています。ムーブしたい場合は`views::as_rvalue`を適用するなどして要素を右辺値にして渡す必要があります。

### `assign_range`

`.assign_range()`はコンテナに指定した範囲を代入するものです。元となった`.assgin()`同様に、コンテナの要素を一旦全てクリアしてから、指定された範囲の要素を新しい要素として追加します。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4, 5};
  std::deque deq(std::from_range, vec);
  std::list lst(std::from_range, vec);
  std::forward_list fwl(std::from_range, vec);

  auto asn = std::views::iota(-5, 0)
           | std::views::reverse
           | std::views::take(3);

  // 戻り値なし
  vec.assign_range(asn);  // ok
  // vec : {-1, -2, -3}

  deq.assign_range(asn);  // ok
  lst.assign_range(asn);  // ok
  fwl.assign_range(asn);  // ok
  // 挿入後範囲はvecと同じ
}
```

全ての場合に、コンテナの要素型`T`と代入しようとする`range`型`R`は`assignable_from<T&, ranges::range_reference_t<R>>`のモデルとならなければなりません。

`vector, deque`の場合、この関数の実行後に実行前に取得された終端イテレータ（`.end()`から取得したイテレータ）が無効化されます。

これも、入力範囲が右辺値であっても挿入される要素は全てコピーされています。ムーブしたい場合は`views::as_rvalue`を適用するなどして要素を右辺値にして渡す必要があります。

### `replace_with_range`

`.replace_with_range()`は`std::basic_string`のみに追加される関数で、元となった`.replace()`同様に指定された範囲の文字列を挿入する文字範囲によって置き換えるものです。

```cpp
using std::ranges::next;

int main() {
  std::string str = "abcdefg";
  auto rep = views::repeat('x', 4);

  // 3文字目から2文字文
  auto pos1 = next(str.begin(), 2);
  auto pos2 = next(pos1, 2);

  auto& res = str.replace_with_range(pos1, pos2, rep);
  // str : "abxxxxefg"

  // 戻り値としては*thisが返されている
  assert(&res == &str);

  pos1 = next(str.begin(), 2);
  pos2 = next(pos1, 3);

  // 3文字目から3文字分を"ww"で置換
  str.replace_with_range(pos1, pos2, views::repeat('w', 2));
  // str : "abwwxefg"
}
```

挿入する範囲はイテレータによって指定して、挿入する文字範囲の長さは指定範囲の長さと一致している必要はありません。指定範囲より後ろの文字列は、挿入する長さが指定範囲の長さより短ければ詰められ、長ければ後ろにずらされます。

戻り値は元のオブジェクトへの左辺値参照が得られます。メソッドチェーンのように処理を連結したい場合以外はあまり使い道は無いでしょうか。

### `push_range`

`.push_range()`はコンテナアダプタと呼ばれるコンテナに追加される関数で、指定した範囲の要素をそれぞれのコンテナの意味論のもとで追加していくものです。これには元になった関数はなく、`.push()`が単一要素の代わりに範囲をまとめて追加するようになったものです。

```cpp
int main() {
  std::deque deq = {5, 6, 7};

  std::queue que{deq};
  std::priority_queue pqu{std::less<void>{}, deq};
  std::stack stk{deq}

  auto push = std::views::iota(1, 4);

  // 戻り値なし
  que.push_range(push);
  // que : {5, 6, 7, 1, 2, 3}

  pqu.push_range(push);
  // pqu : {7, 6, 5, 3, 2, 1}

  stk.push_range(push);
  // stk : {3, 2, 1, 7, 6, 5}
}
```

挿入しようとする範囲の要素をその順番通りに`.push()`していったように要素を追加します。処理後の順序は、それぞれのコンテナの意味論に応じた順序になります。

これも、入力範囲が右辺値であっても挿入される要素は全てコピーされています。ムーブしたい場合は`views::as_rvalue`を適用するなどして要素を右辺値にして渡す必要があります。

## `ranges::to`

コンテナが`range`を直接受け取れるようになったのは大きな進歩でありかなり便利ですが、パイプラインを使用しているとコンテナの`from_range`コンストラクタを呼び出すのではなくパイプラインでつなげてコンテナへの変換まで行いたくなるでしょう。`ranges::to`はそのためのもので、パイプラインによって入力した`range`を任意のコンテナに変換して返します。

```cpp
int main () {
  auto is_odd = [](auto n) {
    return n % 2 == 1;
  };

  std::same_as<std::vector<int>> 
    auto vec = std::views::iota(0)
             | std::views::filter(is_odd)
             | std::views::stride(3)
             | std::views::take(5)
             | std::ranges::to<std::vector>();  // 直接変換
  
  for (int n : vec) {
    std::cout << std::format("{:d} ", n);
  }
}
```
```{style=planetext}
1 7 13 19 25 
```

同様の記法によって他のコンテナへも変換できます。

```cpp
int main() {
  auto is_odd = ...;

  auto rng = std::views::iota(0)
           | std::views::filter(is_odd)
           | std::views::stride(3)
           | std::views::take(5);

  // 以下すべてokな例
  auto lst = rng 
           | std::ranges::to<std::list>();

  auto deq = rng 
           | std::ranges::to<std::deque>();

  auto fwl = rng 
           | std::ranges::to<std::forward_list>();

  auto que = rng 
           | std::ranges::to<std::queue>();
  
  auto pqu = rng 
           | std::ranges::to<std::priority_queue>();

  auto stk = rng 
           | std::ranges::to<std::stack>();
}
```

`ranges::to<C>()`のようにコンテナ型`C`を指定して呼び出すと、`C`への変換を行うRangeアダプタクロージャオブジェクトを返します。それにはRangeアダプタの場合と同様にパイプによって`range`を入力することができ、`range`を入力するとその要素列をコンテナ`C`へ詰めて返します。

```cpp
int main() {
  namespace ranges = std::ranges;

  auto is_odd = ...;

  auto rng = std::views::iota(0)
           | std::views::filter(is_odd)
           | std::views::stride(3)
           | std::views::take(5);

  // vectorへの変換を行う呼び出し可能オブジェクトを構築
  auto to_vec = ranges::to<std::vector>();

  // Rangeアダプタ同様の方法でrangeを入力できる
  auto vec1 = rng | to_vec; // ok
  auto vec2 = to_vec(rng);  // ok

  // ()で直接渡すこともできる
  auto vec3 = ranges::to<std::vector>(rng); // ok

  // 要素型を指定してもいい
  auto vec4 = rng
            | ranges::to<std::vector<int>>(); // ok

  // 暗黙変換可能なら要素型は入力と異なってもいい
  auto vec5 = rng
            | ranges::to<std::vector<double>>();  // ok
}
```

このような使用法からも分かるかもしれませんが、`ranges::to`は関数テンプレートです。そのため、使用時は最後に`()`が必須です。

```cpp
namespace std::ranges {

  // A. 入力範囲からの変換を行うオーバーロード
  template<class C, input_range R, class... Args>
    requires (!view<C>)
  constexpr C to(R&& rng, Args&&... args);
  
  template<template<class...> class C, input_range R, class... Args>
  constexpr auto to(R&& rng, Args&&... args);

  // B. Rangeアダプタクロージャオブジェクトを生成するオーバーロード
  template<class C, class... Args>
    requires (!view<C>)
  constexpr auto to(Args&&... args);
  
  template<template<class...> class C, class... Args>
  constexpr auto to(Args&&... args);
}
```

全部で4つのオーバーロードがあり、大きく2種類に分けられます。範囲の入力から変換まで全てこなすのは第一引数に`input_range`を受け取る2つのオーバーロード（A）で、残りの2つ（B）はコンテナ型の指定と追加の引数を保存しておいて入力範囲を受けてAを呼び出すRangeアダプタクロージャオブジェクトを生成します。

そのため、Bの戻り値は`range`ですらないRangeアダプタクロージャオブジェクトとなり、Aの戻り値は指定したコンテナ型`C`（の特殊化）のオブジェクト（*prvalue*）となります。

全てのオーバーロードが追加の引数を受けるようになっていますが、これはコンテナ構築時の追加の引数を渡すために使用します。

```cpp
int main () {
  namespace ranges = std::ranges;
  
  std::pmr::monotonic_buffer_resource mr{};
  
  auto rng = std::views::iota(0)
           | std::views::stride(2)
           | std::views::take(5);


  auto vec1 = rng
            | ranges::to<std::pmr::vector<int>>(&mr); // ok

  auto vec2 = ranges::to<std::pmr::vector<int>>(rng, &mr);  // ok

  auto pqu = rng 
           | ranges::to<std::priority_queue>(std::greater<>{});  // ok
}
```

### 要素型の推論

`ranges::to<C>()`の`C`にはコンテナ型を指定しますがそこでは要素型を指定してもしなくても大丈夫になっています。それはテンプレートテンプレートパラメータを取るオーバーロード（ABそれぞれで2つ目のもの）が定義されているためで、そちらのオーバーロードでは変換後の要素型を含めたコンテナ型全体を導出してからAの1つ目のオーバーロードに委譲します。

Bの2つ目のオーバーロードはAの2つ目に処理を委譲するため、要素型推論を行っているのはAの2つ目のオーバーロードです。

```cpp
namespace std::ranges {
  
  template<template<class...> class C, input_range R, class... Args>
  constexpr auto to(R&& rng, Args&&... args);
}
```

このオーバーロードは次のようにAの1つめに処理を委譲します

```cpp
return to<decltype(DEDUCE_EXPR)>(std​::​forward<R>(rng), std​::​forward<Args>(args)...);
```

この時、`ranges::to`のテンプレートパラメータの`decltype(DEDUCE_EXPR)`によって要素型も含めたコンテナ型全体を推論しており、この`DEDUCE_EXPR`は次のように決まります

1. 次の式が有効な場合
    - `C(declval<R>(), declval<Args>()...)`
2. そうではなく、次の式が有効な場合
    - `C(from_range, declval<R>(), declval<Args>()...)`
3. そうではなく、次の式が有効な場合
    - `C(declval<input-iterator>(), declval<input-iterator>(), declval<Args>()...)`
4. それ以外の場合はill-formed

ここでの`C`は要素型を欠いているコンテナ型であり、この3パターンの式はいずれもコンストラクタ呼び出し式です。

この3パターンの式のいずれかを`decltype()`に入れることでコンテナ型を導出しており、つまりは`C`の初期化式におけるCTADを利用して要素型を推論しています。

1つ目の式は直接`range`オブジェクトを渡すタイプのもので、標準のコンテナアダプタに対して内部コンテナを指定する場合などがこの経路で要素型が求められます。

2つ目の式は前節で紹介した`from_range`コンストラクタを呼び出すもので、ほとんどの標準コンテナはこの経路で要素型が求められます。

3つ目の式は従来のイテレータペアを取るコンストラクタを呼び出すものです。使われている`input-iterator`というのは次のように定義される、即席のダミーイテレータ型です。

```cpp
struct input-iterator {
  using iterator_category = input_iterator_tag;
  using value_type = range_value_t<R>;
  using difference_type = ptrdiff_t;
  using pointer = add_pointer_t<range_reference_t<R>>;
  using reference = range_reference_t<R>;

  reference operator*() const;
  pointer operator->() const;

  input-iterator& operator++();
  input-iterator operator++(int);

  bool operator==(const input-iterator&) const;
};
```

これは*Cpp17InputIterator*要件を満たす少なくともC++17の入力イテレータとして（型レベルで）有効なイテレータ型で、入力範囲`R`に対応するC++17入力イテレータを即席でエミュレートするものです。

```cpp
int main() {
  namespace ranges = std::ranges;
  
  auto rng = std::views::iota(0, 10);
  std::list lst = {1, 2, 3};

  // 1のパターンによる推論
  auto stk = lst | ranges::to<std::stack>();
  // std::stack(lst) -> std::stack<int, std::list<int>>

  // 2のパターンによる推論
  auto vec = rng | ranges::to<std::vector>();
  // std::vector(std::from_range, rng) -> std::vector<int>

  // 3のパターンによる推論
  auto pqu = rng 
           | ranges::to<std::priority_queue>(std::greater<>{});
  // std::priority_queue(input-iterator{}, input-iterator{},
  //                     std::greater<>{})
  //   -> std::priority_queue<int, std::greater<>>
}
```

ここで選択され呼び出されるコンストラクタはあくまで要素型推論のためだけのもので実際の変換に使用されているわけではありませんが、標準のコンテナの場合はこの推論時と変換時で異なるコンストラクタが選択されることは無いでしょう。

### 変換の詳細

結局、`range`からコンテナへの変換を担っているのはAの1つ目のオーバーロードです。

```cpp
namespace std::ranges {

  template<class C, input_range R, class... Args>
    requires (!view<C>)
  constexpr C to(R&& rng, Args&&... args);
}
```

残りのオーバーロードはそれぞれのステップで欠けているものを埋めながら最後はこのオーバーロードに行き着きます。そして、ここではコンテナ型`C`に応じた変換が選択され実行されています。

入力が`input_range`であるのは当然として、追加の制約として変換先の型`C`が`view`ではないことが求められています。`view`は通常範囲を所有しない軽量な型であり、`ranges:to`の変換先としては適切ではありません。

まず大まかに次の3つのいずれかに分岐します

1. `C`が`input_range`ではないか、`range_reference_t<R>`が`range_value_t<C>`へ変換可能（`convertible_to`）な場合
    - 後述
2. そうではなく、`range_reference_t<R>`が`input_range`である場合
    - すぐ下で説明
3. それ以外の場合はill-formed

1の場合はさらに分岐するので後で見ることにして、まず2の場合を見てみます。`range_reference_t<R>`が`input_range`である場合とはつまり、入力範囲が`range`の`range`になっているような場合です。この場合、次のような式の結果を返します

```cpp
to<C>(rng | views::transform([](auto&& elem) {
  return to<range_value_t<C>>(std::forward<decltype(elem)>(elem));
}), std::forward<Args>(args)...);
```

これは、入力範囲の要素の`range`を再帰的に`ranges::to`で処理しています。この場合、出力コンテナ型`C`の要素型もコンテナであるか`ranges::to<range_value_t<C>>`の変換が適用可能な型である必要があります。

```cpp
using namespace std::ranges;

int main() {
  // 文字範囲の範囲からstringのvectorへ変換
  auto vec = "split_view takes a view and a delimiter"
           | views::split(' ')
           | to<std::vector<std::string>>();

  for (auto& str : vec) {
    std::cout << std::format("{:s}\n", str);
  }
}
```
```{style=planetext}
split_view
takes
a
view
and
a
delimiter
```

なお、この場合のネストに制限は特に設けられていません。2つは当然として3つでも4つでも変換は可能です。

1の場合の「`C`が`input_range`ではないか、`range_reference_t<R>`が`range_value_t<C>`へ変換可能な場合」というのに該当する場合、さらに次の条件によって処理が分岐します

1. `constructible_from<C, R, Args...>`である場合
2. `constructible_from<C, from_range_t, R, Args...>`である場合
3. 次の条件をすべて満たす場合
    - `common_range<R>`
    - `iterator_traits<iterator_t<R>>​::​iterator_category`がイテレータカテゴリとして有効である
        - `input_iterator_tag`派生でなければならない
    - `constructible_from<C, iterator_t<R>, sentinel_t<R>, Args...>`
4. 次の条件をすべて満たす場合
    - `constructible_from<C, Args...>`
    - `C`が`.push_back()`か`.insert()`を備えていて入力`R`の要素を挿入可能である
5. それ以外の場合はill-formed

以後、こちらの4パターンの分岐はそれぞれ1-1や1-4の様に参照することにします。

まず1-1の場合、次の式を実行して返します

```cpp
C(std::forward<R>(rng), std::forward<Args>(args)...)
```

これは要素型推論の1の場合に対応し、直接入力の`range`オブジェクト`rng`を転送して`C`を構築するものです。標準のコンテナアダプタなどがここに該当します。

次に1-2の場合、次の式を実行して返します

```cpp
C(std::from_range, std::forward<R>(rng), std::forward<Args>(args)...)
```

これは要素型推論の2の場合に対応し、`from_range`コンストラクタによって`rng`を渡して`C`を構築します。標準のコンテナ型のほとんどがここに該当します。

次に1-3の場合ですが、この一見複雑な条件は要するにC++17までのイテレータペアを取るコンストラクタが利用可能であるかをチェックしているものです。この場合は次の式を実行して返します

```cpp
C(ranges::begin(rng), ranges::end(rng), std::forward<Args>(args)...)
```

これは要素型推論の3の場合に対応し、`common_range`である`rng`の先頭と終端のイテレータを渡して`C`を構築します。

最後の1-4の場合は次のような処理によって`C`を構築して返します

```cpp
C c(std::forward<Args>(args)...);

if constexpr (sized_range<R> && reservable-container<C>) {
  c.reserve(static_cast<range_size_t<C>>(ranges::size(rng)));
}

ranges::copy(rng, container-inserter<range_reference_t<R>>(c));
```

これは1-1から1-3までのいずれにも当てはまらないような型（おそらくコンテナ型）について、メンバ関数の`.insert()`もしくは`.push_back()`が可能であり、`rng`の各要素がそのいずれかによって挿入可能である場合に、それを使用して`rng`から`C`の構築を行うものです。

`if constexpr`の分岐は、`C`が`.reserve()`を備えていて`R`が`sized_range`である場合に`rng`の要素数を用いて予め`.reserve()`しています。

`container-inserter()`は`.insert()`と`.push_back()`の可能な方を使用して挿入するように切り替える説明専用の関数テンプレートで、おおよそ次のようなものです

```cpp
template<class Ref, class Container>
constexpr auto container-inserter(Container& c) {
  if constexpr (requires { c.push_back(declval<Ref>()); })
    return back_inserter(c);
  else
    return inserter(c, c.end());
}
```

1-4に該当するのは、入力範囲が`common_range`ではなく、`C`が`from_range`コンストラクタを持たないような場合だと思われます。なおこの場合でも、要素型の推論はイテレータペアを取るコンストラクタを用いて行われるため、それさえあれば要素型を指定せずに`ranges::to`することは可能です。

```cpp
#include <boost/container/vector.hpp>

int main() {
  namespace ranges = std::ranges;
  
  auto rng = std::views::iota(0, 10);
  std::list lst = {1, 2, 3};

  // 1-1の場合の構築
  auto stk = lst | ranges::to<std::stack>();
  // return std::stack(lst);

  // 1-2の場合の構築
  auto vec = rng | ranges::to<std::vector>();
  // return std::vector(std::from_range, rng);

  // 1-3の場合の構築
  auto pqu = rng 
           | ranges::to<std::priority_queue>(std::greater<>{});
  // return std::priority_queue(ranges::begin(rng)
  //                            ranges::end(rng),
  //                            std::greater<>{});

  // common_rangeではない範囲
  // sized_rangeでもない
  auto nc_rng = std::views::iota(0)
              | std::views::take(10);

  // 1-4の場合の構築
  auto bvec = nc_rng
            | ranges::to<boost::container::vector>(); // ok
  // boost::container::vector<int> c();
  // ranges::copy(rng, back_inserter(c));
  // return c;
}
```

1-4に該当するコンテナはおそらくC++23の標準ライブラリ内にはないため、Boost.Containerの`boost:container::vector`（ver 1.83時点）を持ってきました。

Boostのコンテナを持ってきていることからも分かるように、`ranges::to`のこれらの変換仮定は標準のコンテナに対して制限されておらず、ユーザー定義のコンテナ、もっと言えば`range`を受け取ることのできるような任意の型に対して開かれています。

そのため、`ranges::to<C>()`の`C`にはここまで説明した変換や推論の規則に当てはまる任意の型を渡すことができます。

### 要素をムーブする

`ranges::to`に入力する変換元の`range`の各要素は多くの場合デフォルトでは変換先のコンテナへコピーされています。何もしなくてもムーブされているのは、要素（間接参照結果）が右辺値になってる場合のみです。

```cpp
using namespace std::ranges;

int main() {
  // 文字範囲の範囲からstringのvectorへ変換
  auto vec = "split_view takes a view and a delimiter"
           | views::split(' ')
           | to<std::vector<std::string>>();

  // listへ変換、各要素のstringはコピー構築されている
  auto lst = vec | to<std::list>();


  // 要素が右辺値stringになっている範囲
  auto rng = = "split_view takes a view and a delimiter"
           | views::split(' ')
           | views::transform([](auto rng) {
               return to<std::string>(rng);
             });

  // この場合、dequeの要素のstringはムーブ構築されている
  auto deq = rng | to<std::deque>();
}
```

要素がムーブオンリーな型の場合はこのことはコンパイルエラーとして観測できます。

```cpp
int main() {
  namespace ranges = std::ranges;
  
  std::vector<std::unique_ptr<int>> upvec;
  upvec.emplace_back(std::make_unique<int>(10));
  upvec.emplace_back(std::make_unique<int>(100));
  upvec.emplace_back(std::make_unique<int>(17));
  
  // upvecの要素はunique_ptrの左辺値であるため、これだとコピーしようとしてエラー
  auto up_list = upvec 
               | std::ranges::to<std::list>;  // ng

  // ムーブしてもダメ
  auto up_list = std::move(upvec)
               | std::ranges::to<std::list>;  // ng
}
```

このサンプルコードは`views::as_rvalue`で例示したものと同じであり、これは`views::as_rvalue`によって範囲の各要素を右辺値に変換することで解決できます。すなわち、`ranges::to<C>(rng)`において`rng`の各要素をムーブして`C`を構築したいのならば、`views::as_rvalue`を用いて`rng`を右辺値の範囲に変換しておく必要があります。

```cpp
int main() {
  ...

  // 各要素を右辺値にしてから変換するとok
  auto up_list = upvec 
               | std::views::as_rvlaue
               | std::ranges::to<std::list>;  // ok

}
```

このことは`ranges::to`だけでなく、`from_range`コンストラクタや`_range`系のメンバ関数でも同様です。

# `std::generator`

C++20でコルーチンの言語サポートと、コルーチンユーによる機能作成のための最低限のライブラリ機能が用意されましたが、ユーザーが気軽に利用できるコルーチンアプリケーションは一切用意されていませんでした。

C++23では、その第一弾として範囲をオンデマンドで生成していくためのコルーチンジェネレータクラスである`std::generator`が`<generator>`ヘッダに用意されます。

```cpp
// 新設の<generator>ヘッダに追加される
import <generator>;

auto random_number_sequence() -> std::generator<std::uint64_t> {
  std::mt19937_64 engine(std::random_device{}());

  while (true) {
    co_yield engine();
  }
}

int main() {
  // 乱数を10個出力
  for (auto rnd : random_number_sequence() | std::views::take(10))
  {
    std::cout << rnd << '\n';
  }
}
```

拙著「C++20 ライブラリ機能1」では簡易的ながらもこれと同等のクラスを例として作成していましたが、それよりも洗練されたものが標準ライブラリ機能として利用できるようになります。この例のように、`std::generator`はそれそのものが`range`（常に`input_range`）となるため、範囲`for`文や`<ranges>`の各種アダプタ、Rangeアルゴリズムなどと容易に組み合わせて使用できます

`std::generator`を返す関数内で`co_yield value;`とすると`value`を呼び出し元に返し、この関数（コルーチン）の実行はそこで中断します。コルーチンの中断中はその実行は中断箇所で停止しており、関数の状態（自動変数やスタック）等は保持されています。コルーチンの直接の戻り値である`std::generator`オブジェクトをイテレートするごとに（より正確にはそのイテレータをインクリメントするごとに）、コルーチンの実行が再開され、関数の実行は次の`co_yield`まで進行し、そこでまた値を返して中断します。

値の生成は常に`co_yield`で行う必要があり、`std::generator`によるコルーチンでは`co_await`を使用できません。`std::generator`によるコルーチンはそのコルーチンの末尾に到達するか`co_return`に到達することで終了（中断ではなく）します。

実のところ、`std::generator`を使用しようと思って思いつく単純な例の多くは、RangeファクトリとRangeアダプタを組み合わせることでより効率的に実現可能だったりします。ある処理を`range`とその反復で表現することができる場合で新しいイテレータを作成しないとそれを達成できそうもない場合に、`std::generator`を利用することでイテレータを作成することなくそのような処理を単純な関数（コルーチン）の形で記述して`range`化することができます。

## 生成する値などの型の決定

`std::generator<T>`を利用する際、テンプレートパラメータ`T`には生成したい値の型を指定します。ほとんどの場合はこれは修飾なしの素の方を指定すれば事足りるはずですが、稀に参照を返したい場合もあるかもしれません。そういう場合でも`T`には参照型や`const`参照型を渡すことができます。ただ、その場合に`std::generator<T>`は`T`に応じて`co_yield`に渡せる型（正確には、プロミス型の`.yield_value()`の引数型）や、それらに応じてコルーチン呼び出し側で生成される値の型（イテレータの間接参照の結果型）を調整します。

```cpp
namespace std {
  // std::generatorの宣言例
  template<class Ref, class V = void, class Allocator = void>
  class generator : public ranges::view_interface<generator<Ref, V, Allocator>> {
    ...
  };
}
```

`std::generator`の第一テンプレートパラメータに指定する型を`Ref`、任意の修飾なしの型を`T`、`std::generator`のイテレータの間接参照の結果型を`reference`とすると、3種類の型の関係性は次のようになります

|`Ref`|`co_yield`に渡せる型|`reference`|
|---|---|---|
|`T`|`T&&, const T&`|`T&&`|
|`T&`|`T&`|`T&`|
|`T&&`|`T&&, const T&`|`T&&`|
|`const T&`|`const T&`|`const T&`|

`const T`とか`const T&&`は実用的ではないので省略しています。

`std::generator`はさらに、2つ目のテンプレートパラメータ`V`でそのイテレータの値型（`value_type`）を明示的に指定することができます。`V`が指定されない場合は`std::remove_cv_ref_t<Ref>`がデフォルトの値型として使用されます。

値型`V`が明示的に指定されている場合、`Ref`が修飾なしの`T`の場合のみ先ほどの表は若干変化します

|`Ref`|`co_yield`に渡せる型|`reference`|
|---|---|---|
|`T`|`const T&`|`T`|
|`T&`|`T&`|`T&`|
|`T&&`|`T&&, const T&`|`T&&`|
|`const T&`|`const T&`|`const T&`|

どちらの場合も、表にない組み合わせはコンパイルエラーとなります。

ほとんどの場合、`V`を明示的に指定する必要はありません。指定が必要なのは、イテレータの参照型（`reference`）と値型（`value_type`）の関係性が特殊となり、それが問題となる場合のみです。おそらくそれは、`Ref`としてプロキシ型のようなものを使用し、なおかつ一部のイテレータ/Rangeアルゴリズムを使用しているような場合が該当するでしょう。

例えば、`Ref`として`std::tuple<T&, U&>`を指定して`std::tuple<T&, U&>`のオブジェクトを生成しているとき、その値型も`std::tuple<T&, U&>`となりますが、おそらくこの場合に正しいのは`std::tuple<T, U>`であると思われます。これが問題となる場合、`V`に`std::tuple<T, U>`を明示的に指定することでイテレータの値型を直接指定することができるようになっています。

## `co_yield`から`operator*`までの経路について

`std::generator`を使用したコルーチン内部で`co_yield`してから、呼び出し側のイテレータの間接参照（`operator*`）の結果を受けるまでの間は、コルーチン仕様に隠蔽されているため自明ではありません。そこで何回コピーが起こりうるのかは当然気になってくるでしょう。

`std::generator`は基本的に、`co_yield`に渡された値をコピーもムーブもすることなく、そのポインタだけをプロミス型内部に格納します。先ほどの表の「`co_yield`に渡せる型」は全て参照型であるため、`co_yield expr;`（及び`.yield_value()`）に渡しているオブジェクト（`expr`）が例え右辺値であったとしても、そこの`co_yield`式で中断している間は`expr`のオブジェクトは生存し続けています。その状態で呼び出し側でイテレータの間接参照を通してその値を取得する場合、ポインタを通して取得したその参照を`reference`型にキャストして返すことでコピーを回避します。

ただし、`reference`が右辺値（`T`もしくは`T&&`）である場合に`co_yield`に左辺値を渡すと、`const T&`をとる`.yield_value()`が呼ばれます。この場合、`co_yield expr;`の呼び出しは、`expr`をコピーします。この場合、`co_yield(const T&)`が返す`awaitable`型オブジェクト内部に`expr`はコピーされており、プロミス型はそのオブジェクトへのポインタを保持します。この場合もその後すぐコルーチンを中断させることでこの`awaitable`型オブジェクトは次に再開されるまで生存しており、呼び出し側でイテレータの間接参照を通してその値を取得する場合には同じようにそのオブジェクトへの参照を`reference`にキャストするだけです。そのため、この経路ではコピーが1回だけ発生します。

まとめると

- `Ref`に右辺値（`T`/`T&&`）を指定していて`co_yield`に左辺値を渡している場合
    - `std::generator`の機構内部でコピーが1回発生する
    - `Ref`の素の型`T`は`copy_constructible`である必要がある
- それ以外の場合
    - `co_yield`から`operator*`まででコピーは発生しない
    - `Ref`の素の型`T`はムーブオンリーでもコピー/ムーブ不可でも良い

のようになっています。ただし、`co_yield`に`T`に変換可能な型の値を渡した場合は変換されてから同様の経路（おそらく2つ目の経路）を辿るため、変換に伴う処理は発生します。

なお、ここまでのことはあくまで`std::generator`のイテレータの`operator*`がリターンするところまでの話です。その結果を受ける場所での受け方によっては、そこでコピーが発生する可能性があります。いずれの場合でも参照型で受けておけばコピーは発生せず、非参照型（ちょうど`T`）で受けるとコピーが発生します。

```cpp
auto gen_coro() -> std::generator<...>;

int main() {
  auto gen = gen_coro();
  auto it = gen.begin();

  auto   v1 = *it;  // ok、コピー（ムーブ）される
  auto&  v2 = *it;  // ng、右辺値を受けられない
  auto&& v3 = *it;  // ok、コピーなし、寿命に注意
  const auto& v4 = *it;  // ok、コピーなし、寿命に注意
}
```

とりあえず`auto&&`で受けておけばジェネリックかるゼロコピーになるのですが、それは参照で取得されており右辺値に対しても寿命の延長は行われていません（正確にはある1経路だけは行われます）。そのため、取得後にコルーチンを再開すると、その参照（上記例の`v3, v4`）はダングリング参照となります。とはいえこれを気にする必要があるのは、この例のように`std::generator`のイテレータを直接触ってる場合のみです。範囲`for`で使用する分には、要素を受ける変数宣言は常に`auto&&`にしておいても危険性はありません。

```cpp
auto gen_coro() -> std::generator<...>;

int main() {
  // 常にゼロコピーで受けられ、かつ安全
  for (auto&& elem : gen_ven()) {
    ...
  }
}
```

## アロケータのカスタマイズ

`std::generator`はアロケータをカスタマイズすることができます。この場合のアロケータはコルーチンの最初の呼び出し時にコルーチンステートを動的確保する部分のメモリ確保方法をカスタマイズするためのものです。デフォルトでは`std::allocator<void>`が使用されています。

基本的なカスタマイズ方法は、`std::generator`によるコルーチンの引数の先頭で`std::allocator_arg_t`とアロケータを受け取るようにしておくことです。

```cpp
// アロケータをカスタマイズするコルーチン宣言
template <class Allocator>
auto gen_coro(std::allocator_arg_t, Allocator alloc, auto&&... args) -> std::generator<int>;

int main() {
  std::pmr::synchronized_pool_resource mr{};
  std::pmr::polymorphic_allocator<> alloc{&mr};

  // アロケータはallocator_argとともに引数の先頭で渡す
  gen_coro(std::allocator_arg, std::move(alloc), ...);
}
```

この方法はアロケータを型に出現させることなくアロケータのカスタマイズを行えますが、`std::generator`を使用した関数では`std::allocator_arg_t`とアロケータを受け取るようにしておかなければなりません。オーバーロードを提供すればユーザーの使用負担は減らせるかもしれませんが、少し手間です。

そこで、`std::generator`の3番目の引数にアロケータ型（デフォルト構築可能であること）を指定することで、追加のアロケータ専用引数を省略してカスタマイズすることもできます。

```cpp
// オリジナルのアロケータ（ステートレスであること）
template<typename T>
struct my_stateless_allocator {
  my_stateless_allocator() = default;
  ...
};

// アロケータをカスタムしたstd::generator
template<typename T, typename V = void>
using my_generator = std::generator<T, V, my_stateless_allocator<T>>;

// アロケータをカスタマイズするコルーチン宣言
auto gen_coro(auto&&... args) -> my_generator<int>;

int main() {
  // アロケータを渡す必要がない
  gen_coro(...);
}
```

ただしこの方法は、指定したアロケータ型をデフォルト構築することでアロケータを取得しているため、基本的にステートレスなアロケータ型の場合に使用します。

また、`std::generator`の3番目の引数にアロケータ型を指定した場合でもコルーチン引数先頭でアロケータを渡すことができます。

```cpp
// オリジナルのアロケータ
template<typename T>
struct my_allocator {
  ...
};

template<typename T, typename V = void>
using my_generator = std::generator<T, void, my_allocator<T>>;

// アロケータをカスタマイズするコルーチン宣言
auto gen_coro(auto&&... args) -> my_generator<int>;

int main() {
  std::pmr::synchronized_pool_resource mr{};
  std::pmr::polymorphic_allocator<> alloc{&mr};

  // アロケータはallocator_argとともに引数の先頭で渡す
  gen_coro(std::allocator_arg, alloc, ...);

  // デフォルト構築可能な場合はアロケータを渡さなくても良い
  gen_coro(...);
}
```

最初の方法との違いは`std::generator`の型にアロケータ型が出るか出ないかだけです。

まとめると、`std::generator`の3番目のテンプレート引数を`A`として、`std::generator`のアロケータのカスタマイズ方法には次の4パターンがあります

1. `A`を指定しない（`A = void`）
    1. コルーチン引数でアロケータを受け取る
        - 受け取ったアロケータを使用
        - 受け取ったアロケータはコルーチンステートに保存（コピーされる）
    2. コルーチン引数でアロケータを受け取らない
        - デフォルトアロケータの使用（アロケータカスタマイズしない）
        - アロケータは保存されない（オンデマンドで構築）
2. `A`を指定する
    1. コルーチン引数でアロケータを受け取る
        - 受け取ったアロケータを使用
        - 受け取ったアロケータはコルーチンステートに保存（コピーされる）
    2. コルーチン引数でアロケータを受け取らない
        - `A`をデフォルト構築してアロケータを取得
        - アロケータはステートレスであること
        - アロケータは保存されない（オンデマンドで構築）

アロケータを保存しなくてよい場合、確保されるコルーチンステートの領域はそれに応じて小さくなります。

これらいずれかの方法によって渡したアロケータは、コルーチンの最初の呼び出し時のコルーチン初期化処理中にコルーチンステートのための領域を確保するために使用されますが、その経路は明確ではありません（コード上からは見えない）。

コルーチンステートを保存するための領域が必要な場合は、コルーチン初期化の一番最初、プロミス型のオブジェクトを構築するよりも前に、`operator new`を使用することで領域を確保します。その際、使用する`operator new`はまずプロミス型のコンテキストで検索され、見つからなかったらグローバルのものを使用します。どちらの場合にせよ、見つかった`operator new`は確保する領域のサイズに続いてコルーチンの引数列を渡して呼び出されます。呼び出し（オーバロード解決）が失敗した場合、確保する領域のサイズのみを渡して呼び出します（グローバルの`operator new`が使用される場合は常にこの経路による）。

`std::generator`の場合はそのプロミス型（`std::generator<...>::promise_type`）が`operator new`オーバーロードを3つ定義しており、常にこちらがメモリ確保のために使用されます。3つのうちの1つは確保サイズのみをとるデフォルトのもので、残りの2つはコルーチン引数列も取るものです。どれが使用されたとしても、上記3つの方法のいずれかで渡したアロケータがこれらの`operator new`オーバーロードに適切に伝播されて使用されることで、ユーザーが指定したアロケータによってメモリ確保が行われるようになっています。

オリジナルのコルーチンアプリケーションを自作する場合でも、この`std::generator`がやっていることを参考にしてアロケータサポートを追加することができます。

## `pmr::generator`

C++20以降、`std::pmr::polymorphic_allocator`はC++のライブラリ機能のうちで基本的なものとして扱われており、アロケータカスタマイズをサポートする標準ライブラリ機能は`polymorphic_allocator`のサポートを含むようになっています。それは型エイリアスを定義することで行われており、そのエイリアスは`std::pmr`名前空間に配置されます。

`std::generator`にも、3番目のアロケータパラメータに`polymorphic_allocator`が事前に指定された`std::pmr::generator`が用意されています。

前述のように、この場合は2つの方法でアロケータの動作（使用するメモリリソース）をカスタマイズすることができます。

```cpp
// polymorphic_allocatorを使用するコルーチン宣言
auto gen_coro(auto&&... args) -> std::pmr::generator<int>;

int main() {
  std::pmr::synchronized_pool_resource mr{};
  std::pmr::polymorphic_allocator<> alloc{&mr};

  // アロケータをコルーチンに渡して指定
  gen_coro(std::allocator_arg, alloc, ...);

  // デフォルトリソースを変更
  std::pmr::set_default_resource(&mr);
  gen_coro(...);
}
```

`polymorphic_allocator`はデフォルトコンストラクタで`std::pmr::get_default_resource()`からデフォルトのメモリリソースを取得しますが、このデフォルトは`std::pmr::set_default_resource()`で変更することができます。そのため、アロケータを明示的に渡さず内部でデフォルト構築して使用されている場合でも、アロケータの動作をカスタマイズすることが可能です。

ただし、どちらの場合でも、使用するメモリリソースはそれを渡しているコルーチンの動作期間（戻り値`std::generator`オブジェクトの生存期間）よりも長くなるように管理する必要があります。

## `elements_of`

`std::ranges::elements_of`は、`std::generator`によるコルーチン内で`range`を展開しながら順次生成（`co_yiled`）するためのユーティリティです。

```cpp
auto gen_coro(auto&&... args) -> std::pmr::generator<int> {
  co_yield 1; // ok、1を生成

  std::array<int, 5> arr = {2, 3, 4, 5, 6};

  co_yield arr; // ng、std::array<int, 5>を生成しようとする

  for (int n : arr) {
    co_yield n; // ok
  }
  // 2, 3, 4, 5, 6と生成

  co_yield std::ranges::elements_of(arr); // ok、2, 3, 4, 5, 6と生成
}
```

パイプラインで`range`を`views::join`するのと同じことをコルーチンジェネレータ内で行ってくれます。

`elements_of`を使用しない場合、範囲`for`で展開して順次`co_yiled`していくコードを書かなければなりませんが、`elements_of`を使用することでそれを簡略化することができます。

なお、この例では`std::array`を`elements_of`に渡していますが、`std::array`に限らず任意の`range`を渡すことができます。ただし、`std::generator<T>`に対して`T`に変換可能な型を要素とする`range`でなければなりません。

これだけだと範囲`for`で展開するコードを書いてもさほど変わりはないような気もしますが、`elements_of`が重要なのはコルーチンジェネレータをネストさせた場合においてです。ジェネレータのネストは自身の再帰呼び出しや他のコルーチンジェネレータの呼び出しなどによって発生し、`elements_of`の引数でネストしたコルーチンを呼び出すことで先ほどの`range`の場合と同様にコルーチン起動と要素の展開および順次生成を行えます。

```cpp
// 単純な二分木
struct tree {
  tree* left;
  tree* right;
  int value;
};

// コルーチンジェネレータによるtreeのトラバーサル（深さ優先）
auto dfs_traverse(tree& node) -> std::generator<int> {
  if (node.left) {
    co_yield std::ranges::elements_of(dfs_traverse(*node.left));
  }

  co_yield node.value;
  
  if (node.right) {
    co_yield std::ranges::elements_of(dfs_traverse(*node.right));
  }
}
```

この時、`elements_of`によって展開するとネストしたコルーチンジェネレータを手動で呼び出して要素ごとに`co_yield`するよりも効率的に各要素を生成していくことができます。むしろ、`elements_of`の本領はこちらです。

例えば、上記例の`dfs_traverse()`において、ネストしたコルーチンジェネレータを手動展開すると次のようになります

```cpp
// elements_ofを使用しないtreeのトラバーサル
auto dfs_traverse(Tree& tree) -> std::generator<int> {
  if (tree.left) {
    for (int x : dfs_traverse(*tree.left)) {
      co_yield x;
    }
  }

  co_yield tree.value;
  
  if (tree.right) {
    for (int x : dfs_traverse(*tree.right)) {
      co_yield x;
    }
  }
}
```

最初に`dfs_traverse()`を呼び出した側（一番外側）から見ると、`dfs_traverse()`を再開すると各`for`では、外側の`dfs_traverse()`の中断とネストして呼ばれているコルーチン（再帰した`dfs_traverse()`）の再開がループの度に行われます。この例がそうですが、ネストが深くなると一番外側の1回のループでその深さの分だけ中断と再開が連鎖することになります（`N`段ネストしているとき、大元の`std::generator`のイテレータの進行に伴って`N`回中断と再開が再帰的に行われる）。

`elements_of`を使用すると、`co_yield elements_of(...);`の評価で一番外側の`dfs_traverse()`が中断し、ネストして呼ばれているコルーチン（再帰した`dfs_traverse()`）が起動され、以降外側のコルーチンへの再開はこのネストしたコルーチンを直接再開するようになります。これは何段ネストしていたとしても、大元のコルーチンの制御が直接一番深いところのコルーチンを制御するようになります。このため、ネストの深さによらず、一番外側の1回のループで中断と再開は一回だけ発生します（`N`段ネストしているとき、大元の`std::generator`のイテレータの進行に伴って1回だけ中断と再開が行われる）。

また、このようにコルーチンがネストして呼ばれている時に対称転送（*symmetric transfer*）を適切に行うことで、ネストしたコルーチンが相互再起呼び出しを行ってスタック消費量が増えてしまう問題を回避しています。

`elements_of`はこれらの最適化を単一の`std::generator`型のみで実現するためにそのネスト構造を検出するためのマーカーであり、実のところ関数ではなく構造体（集成体）です（そのため、`elements_of{...}`としても使用できます）。

```cpp
namespace std::ranges {

  // elements_ofの実装例
  template<range R, class Allocator = allocator<byte>>
  struct elements_of {
    [[no_unique_address]]
    R range;
    
    [[no_unique_address]]
    Allocator allocator = Allocator();
  };

  // 推論補助
  template<class R, class Allocator = allocator<byte>>
  elements_of(R&&, Allocator = Allocator()) -> elements_of<R&&, Allocator>;
}
```

# その他の関連する変更

## `std::string_view`の`range`コンストラクタ

`std::string_view`に`range`コンストラクタが追加されることによって、文字列の範囲となっているものをより簡易に`std::string_view`に変換できるようになります。ただし、入力の`range`は`contiguous_range`でなければならずいくつかの追加の制約（後述）を満たす必要があります。

```cpp
int main() {
  using namespace std::string_view_literals;
  using namespace std::views;

  const auto str = "split_view takes a view and a delimiter"sv;

  for (auto strv : str | split(' ')
                       | transform([](auto subrng) {
                           // C++20だと例えばこう書いていた
                           return std::string_view{subrng.begin(), subrng.end()};
                           // C++23はそのまま渡せる
                           return std::string_view{subrng}; // ok
                         }))
  {
    std::cout << strv << '\n';
  }
}
```

ただし、この`range`コンストラクタは無条件で`explicit`指定されているため暗黙変換はできず、どこかで明示的変換が必要になります。

```cpp
int main() {
  using namespace std::string_view_literals;
  using namespace std::views;

  const auto str = "split_view takes a view and a delimiter"sv;

  // 暗黙変換は禁止されるため、これはできない
  for (std::string_view strv : str | split(' ')) {
    std::cout << strv << '\n';
  }
}
```

このコンストラクタの宣言及び制約はおおよそ次のようになっています

```cpp
template <class charT, class traits = char_traits<charT>>
class basic_string_view {
  ...

  // 追加されるコンストラクタの制約例
  template <ranges::contiguous_range R>
    requires (not same_as<remove_cvref_t<R>, basic_string_view>) && 
             ranges::sized_range<R> &&
             is_same_v<ranges::range_value_t<R>, charT> &&
             (not is_convertible_v<R, const charT*>) &&
             (not requires(remove_cvref_t<R> d) {
               d.operator ::std::basic_string_view<charT, traits>();
             })
  constexpr explicit basic_string_view(R&& r);

  ...
};
```

複雑ですが、`R`が`std::string_view`そのものや`std::string`等`std::string_view`への変換演算子を備えているもの、文字列ポインタへ変換できるものなどを弾いています。これらは`std::string_view`の他のコンストラクタ及び既存の変換と競合するのでそれを回避するための制約です。残ったものは、文字型の一致と`ranges::size()`、`ranges::data()`を使用可能であるかをチェックしています。

このコンストラクタが受けた`range`オブジェクトを`r`とすると、文字列先頭ポインタを`ranges::data(r)`から、文字列長を`ranges::size(r)`から取得し`std::string_view`を初期化します。すなわち、このコンストラクタは受けた文字範囲全体を文字列として参照する`std::string_view`を構築します。

```cpp
int main() {
  const char str[] = "test";
  std::span<const char> spn{str};

  std::string_view sv1{str};  // length : 4
  std::string_view sv2{spn};  // length : 5
}
```

これは選択されるコンストラクタによって終端`\0`の位置判定が異なることから生じており、このことはこのコンストラクタに`explicit`が付加されている理由でもあります。`char`の配列は単なるバイト配列としても広く利用されているため、その全体を常に文字列として見做すのは正しくない場合があるため、`range`からの変換はプログラマの意図を確認するために`explicit`指定されています。

## `std::format`の`range`対応

C++20で追加された`std::format()`では、組み込み型に対してのみフォーマットサポートが提供されていました。C++23では、一般の`range`に対するサポートが加わることで、標準コンテナをはじめとした各種の`range`オブジェクトを`std::format()`で直接文字列化できるようになります。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4};
  std::map<int, const char*> map = {{1, "1"}, {2, "2"}, {3, "3"}};

  std::cout << std::format("{}\n", vec);
  std::cout << std::format("{}\n", map);
}
```
```{style=planetext}
[1, 2, 3, 4]
{1: "1", 2: "2", 3: "3"}
```

`range`は基本的に`[]`に囲まれてその要素がカンマ区切りで出力されます。この時、標準の連想コンテナだけは`{}`に囲まれ、要素のキーと値が`: `で区切られた上でカンマ区切りで出力されます。`set`系の場合は各要素は単にカンマ区切りで出力されます。

```cpp
int main() {
  std::set<int> set = {4, 3, 10, 5, 9};

  std::cout << std::format("{}\n", set);
}
```
```{style=planetext}
{3, 4, 5, 9, 10}
```

ただし、いずれの場合でも`range`の要素型が`std::format`で出力可能でなければ出力できません。

```cpp
int main() {
  // std::optionalはstd::format非対応
  std::vector<std::optional<int>> vec = {1, std::nullopt, 2};

  std::cout << std::format("{}\n", vec);  // ng
}
```

### `formattable`コンセプト

`std::format`で出力可能な型というのを識別する方法はC++20では提供されていませんでしたが、C++23ではそれは`std::formattable`コンセプトとして提供されるようになります。

```cpp
// 説明専用formattable-withコンセプト
template<class T, class Context,
         class Formatter = typename Context::template formatter_type<remove_const_t<T>>>
concept formattable-with =
  // formatter特殊化はsemiregularであること
  semiregular<Formatter> &&
  requires(Formatter& f, const Formatter& cf, T&& t, Context fc,
           basic_format_parse_context<typename Context::char_type> pc)
  {
    // formatterの2つのメンバ関数の要求
    { f.parse(pc) } -> same_as<typename decltype(pc)::iterator>;
    { cf.format(t, fc) } -> same_as<typename Context::iterator>;
  };

// formattableコンセプト定義
template<class T, class charT>
concept formattable =
  formattable-with<remove_reference_t<T>, basic_format_context<fmt-iter-for<charT>, charT>>;
```

`std::formattable<T, charT>`は、フォーマット対象の型`T`が`charT`でフォーマットできるように`std::formatter<remove_cvref_t<T>, charT>`が用意されている場合に`true`となります。

`fmt-iter-for<charT>`は`output_iterator<const charT&>`のモデルとなる出力イテレータ型です。`formattable-with`の定義内での`Formatter`は`std::formatter`の特殊化であり、`formattable-with`コンセプトはそれに対して`semiregular`であることを求めていたり、`.parse()`と`.format()`がライブラリの想定する形で存在していることを要求しています。

なお、`formattable-with`を介しているのは定義の簡略化のためだと思われ、`T`の修飾を取り除いたり`Formatter`を導出したりといったことをユーザー側に露出させないように行っています。

`std::formattable`コンセプトは、例えば`std::formatter`を特殊化する際に使用します。例えば`std::optional<T>`に対する`std::formatter`を特殊化する際に`T`についてチェックするようにします

```cpp
// std::optional<T>のためのフォーマッター
template<std::formattable<char> T>
struct std::formatter<std::optional<T>, char> {
  // Tのフォーマッターを利用する
  std::formatter<T, char> tf;

  constexpr auto parse(std::format_parse_context& pc) {
    // フォーマット文字列のパース
    ...
  }

  auto format(const std::optional<T>& opt, auto& fc) {
    // 文字列化実装
    ...
  }
};
```

他には、`std::format()`をラップする関数のインターフェースで使用することもできます。

```cpp
// 日付時刻と一緒に値をログ出力する関数
void print_log(std::formattable<char> auto const& v) {
  using namespace std::chrono;
  const auto now = time_point_cast<seconds>(system_clock::now());

  std::clog << std::format("{:%F %T} : {}\n", now, v);
}

int main() {
  int n = 10;
  std::any a = 20;

  print_log(n); // ok
  print_log(a); // ng
}
```

`std::format()`で`range`を出力する場合、その要素型が`formattable`である必要があります。

### `range`フォーマットのオプション

`range`のためのフォーマット文字列は基本的には組み込み型のものと同じなのですが、使用可能なフォーマットオプションが少し異なります。

指定可能なオプションの全体像は次のようになっています

```
{ index : fill align width (n) range-type range-underlying-spec }
```

基本的に全てのオプションが省略可能ですが、並べ替えはできません。

#### index 及び fill align と width

*index*及び*fill align width*は組み込み型の時と同じように動作します。ただし注意点として、これらのオプションはすべて1つの`range`オブジェクトに対しての指定であって、その要素型に対するものではないということです。そのため、幅や寄せの指定は範囲を表す文字列全体に対して作用します。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4};

  // fill align widthの例
  std::cout << std::format("|{:*>16}|\n", vec);
  std::cout << std::format("|{:*<16}|\n", vec);
  std::cout << std::format("|{:*^16}|\n", vec);
}
```
```{style=planetext}
|****[1, 2, 3, 4]|
|[1, 2, 3, 4]****|
|**[1, 2, 3, 4]**|
```

*fill*には`{`、`}`と`:`を除く任意の1文字が指定可能であり、*fill*が指定されている場合は*align*は省略できません。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4};

  std::cout << std::format("|{:*16}|\n", vec);  // ng
}
```

これらのオプションに関わる細かいことは組み込み型のフォーマットオプションと共通していますので、ここでは深入りしません（拙著「C++20 ライブラリ機能 1」で説明されています）。

### n

次の*n*オプション以降は`range`のフォーマット特有のオプションです。

*n*オプションは丁度`n`のみが指定可能で、指定されている場合に範囲文字列の先頭と末尾の囲み文字（`std::vector`の場合`[ ]`）を出力しないようにします。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4};
  
  std::cout << std::format("{:}\n"  , vec);
  std::cout << std::format(" {:n}\n", vec);
}
```
```{style=planetext}
[1, 2, 3, 4]
 1, 2, 3, 4
```

連想コンテナのフォーマットの場合も同様に、囲む`{ }`が出力されなくなります。

### range-type

*range-type*オプションは、`range`出力時の範囲文字列としての表現方法を変更するオプションです。`range`の要素型を`T`として、指定可能なものは次のものです

|*type*|要件|意味|
|:-:|---|---|
|`m`|`T`は`std::pair`か2要素の`std::tuple`|連想コンテナのフォーマットを使用する|
|`s`|`T`は文字型|文字列として出力|
|`?s`|`T`は文字型|デバッグ文字列として出力|

表中の要件とは各オプションが有効となる条件のことで、これが満たされない場合はコンパイルエラーになります。また、要件における文字型とはフォーマット文字列の文字型のことです。

まず、`m`の指定は非連想コンテナの`range`に対して、連想コンテナと同じ形式でフォーマットするものです。

```cpp
int main() {
  std::vector<std::pair<int, const char*>> vec = {{1, "1"}, {2, "2"}, {3, "3"}};
  
  std::cout << std::format("{:}\n", vec);
  std::cout << std::format("{:m}\n", vec);

  // 参考
  std::map<int, const char*> map = {{1, "1"}, {2, "2"}, {3, "3"}};
  std::cout << std::format("{:}\n", map);
}
```
```{style=planetext}
[(1, "1"), (2, "2"), (3, "3")]
{1: "1", 2: "2", 3: "3"}
{1: "1", 2: "2", 3: "3"}
```

そのため、`range`の要素型は`std::pair`の特殊化であるか、2要素の`std::tuple`でなければなりません。

```cpp
int main() {
  std::vector<std::tuple<int, double>> vec1 = {...};
  std::vector<std::array<int, 2>> vec2 = {...};
  std::vector<int> vec3 = {...};
  
  std::cout << std::format("{:m}\n", vec1); // ok、2要素tuple
  std::cout << std::format("{:m}\n", vec2); // ng、2要素tuple-like
  std::cout << std::format("{:m}\n", vec3); // ng、非tuple
}
```

なお、連想コンテナに対して`m`を指定しても出力は変化しません（無視されます）。

`s`及び`?s`オプションはどちらも、文字の範囲を文字列として出力するオプションです。そのため、要素型は文字型（フォーマット文字列の文字型と同じ）である必要があります。

```cpp
int main() {
  std::vector<char> vec = {'h', 'e', 'l', 'l', 'o', '.'};
  
  std::cout << std::format("{:}\n", vec);
  std::cout << std::format("{:s}\n", vec);
  std::cout << std::format("{:?s}\n", vec);
}
```
```{style=planetext}
['h', 'e', 'l', 'l', 'o', '.']
hello.
"hello."
```

どちらも文字列として出力するものですが、`?`が付いていると、その文字列をC++コード上で再現可能な文字列リテラルとして有効な文字列を出力します。このため、特にエスケープシーケンスの扱いが異なります。

```cpp
int main() {
  std::vector<char> vec = {'h', 'e', 'l', '\n', 'l', 'o', '.'};
  
  std::cout << std::format("{:}\n", vec);
  std::cout << std::format("{:s}\n", vec);
  std::cout << std::format("{:?s}\n", vec);
}
```
```{style=planetext}
['h', 'e', 'l', '\n', 'l', 'o', '.']
hel
lo.
"hel\nlo."
```

`?s`で出力される文字列はそれをコピペしてC++ソースコードに貼り付けると、文字列リテラルとして元の文字列を得ることができます。`?`オプションはデバッグ出力オプションとしてC++23で追加されたオプションであり、単体で文字型及び文字列型の*type*オプションでも使用できます。

なお、*range-type*オプションとして`s`もしくは`?s`が指定されている場合、`n`及び後続の*range-underlying-spec*を同時に指定することはできません。

```cpp
int main() {
  std::vector<char> vec = {'h', 'e', 'l', 'l', 'o', '.'};
  
  std::cout << std::format("{:ns}\n", vec);   // ng
  std::cout << std::format("{:?s:c}\n", vec); // ng
}
```

### range-underlying-spec

*range-underlying-spec*オプションは要素型に対するオプションを指定するもので、要素型で利用可能なオプションをそのまま指定することができます。*range-underlying-spec*を指定する場合は、`:`から始まっている必要があります。

```cpp
int main() {
  std::vector vec = {1, 2, 3, 4};

  std::cout << std::format("{::+#x}\n", vec);
  std::cout << std::format("({:n:#06b})\n", vec);
}
```
```{style=planetext}
[+0x1, +0x2, +0x3, +0x4]
(0b0001, 0b0010, 0b0011, 0b0100)
```

*range-underlying-spec*オプションが指定される場合、`{::}`のように置換フィールド内では`:`が複数現れます。この場合、1つ目の`:`は`range`そのものへのオプション指定の開始記号であり、2つ目の`:`はその`range`の要素型へのオプション指定の開始記号となります。このために、`range`型のフォーマットオプションでは*fill*での`:`の使用が禁止されているわけです。

`range`の要素が`range`である場合でも、*range-underlying-spec*オプションで`range`型のフォーマットオプションが使用できます。つまりは、`:`は3つ以上現れる場合もあります。

```cpp
int main() {
  std::vector<std::deque<int>> deqvec = { {1, 2}, {3}, {4, 5}};
  
  std::cout << std::format("{::n:+#x}\n", deqvec);

  std::vector<std::deque<std::array<double, 2>>> advec{ { {1, 2}, {3, 4} }, { {5, 6} } };

  std::cout << std::format("{::::.1e}\n", advec);
}
```
```{style=planetext}
[+0x1, +0x2, +0x3, +0x4, +0x5]
[[[1.0e+00, 2.0e+00], [3.0e+00, 4.0e+00]], [[5.0e+00, 6.0e+00]]]
```

この場合、`{::n:+#x}`の1つ目の`:`は`std::vector<std::deque<int>>`への指定で、2つ目の`:n`は`std::deque<int>`への指定、3つ目の`:+#x`は`int`への指定となっています。以降、間に`range`が1つ増えると`:`も1つ増えます。

これらのオプションは全て必要ないなら省略することができるため、`range`の`range`を出力する際に必ずその個数分の`:`が必要になるわけではありません。ただし、より内側のもの（特に要素型）に対してオプションを指定する場合はその外側のものに対する`:`は省略できなくなります。

```cpp
int main() {
  std::vector<std::deque<std::array<double, 2>>> advec{...};

  std::cout << std::format("{}\n", advec);      // ok、オプションなし
  std::cout << std::format("{::::}\n", advec);  // ok、オプションなし
  std::cout << std::format("{::n}\n", advec);   // ok、dequeに対するオプションのみ
  std::cout << std::format("{:::n}\n", advec);  // ok、arrayに対するオプションのみ

  std::cout << std::format("{:.1e}\n", advec);  // ng、vectorに対するオプションとしては不正
}
```

## 範囲for文における一時オブジェクトの寿命延長

範囲`for`文には当初から、イテレート対象範囲の初期化式内で生成される一時オブジェクトの扱いについての問題がありました。

```cpp
// 右辺値を返す関数
auto f() -> std::vector<std::string>;

int main() {
  // これはok
  for (auto&& str : f()) {
    std::cout << str << '\n';
  }

  // これはUB
  for (auto&& c : f().at(0)) {
    std::cout << c << ' ';
  }
}
```

`f()`は右辺値の`std::vector`を返す関数で、`f().at(0)`はそこから左辺値の`std::string`を取り出しています。この時、`f()`の直接の戻り値である`std::vector`オブジェクトはどこにも保存されず、範囲`for`のループ開始前に破棄されます。すると、そこから取り出した`std::string`への参照はダングリング参照となり、範囲`for`ループは未定義動作に陥ります。

この直接の原因は範囲`for`文の裏側に隠蔽されており、表面からでは問題が分かりづらくなっています。

範囲`for`文はシンタックスシュガーであり、その実体はイテレータを用いた通常の`for`文へ展開される形で実行されます。例えば、規格においては範囲`for`の構文は次のように規定されています

```
for ( init-statement(opt) for-range-declaration : for-range-initializer ) statement
```

`init-statement`は`for`の初期化式で`(opt)`は省略可能であることを表します。`for-range-declaration`は`for(auto&& v : r)`の`auto&& v`の部分で、`for-range-initializer`は`r`の部分、残った`statement`は`for`文の本体です。

そして、これは次のように展開されてコンパイルされます

```cpp
{
	init-statement(opt)

	auto &&range = for-range-initializer ;  // イテレート対象オブジェクトの保持
	auto begin = begin-expr ; // std::ranges::begin(range)相当
	auto end = end-expr ;     // std::ranges::end(range)相当
	for ( ; begin != end; ++begin ) {
		for-range-declaration = * begin ;
		statement
	}
}
```

問題は展開後ブロック内の3行目にあります。

```cpp
auto &&range = for-range-initializer ;
```

先ほどのサンプルから`for-range-initializer`を当てはめてみると

```cpp
// 1つ目のforから
auto &&range = f() ;  // ok

// 2つ目のforから
auto &&range = f().at(0) ;  // UB
```

`auto&&`による変数宣言では初期化式が右辺値ならばその寿命を延長するため、1つ目の場合は`f()`の直接の戻り値はループの間生存しています。しかし、2つ目の場合、`auto&&`が受けているのは`f().at(0)`であり、`f()`の直接の戻り値はどこにも受け止められていません。このため、`f()`の直接の戻り値の寿命はこの初期化式の完了後に終了し、そこから取り出した参照はダングリング参照となります。

この例は恣意的かもしれませんが、`std::optional`で範囲を返し`*/.value()`から取得しようとする場合や、`std::tuple`に含まれる範囲を`std::get`で取り出そうとする場合などにも同じ問題があります。しかもこのことは範囲`for`の構文に隠蔽されているため、問題を知っていてもそれが起きていることに気づくのは困難です。

この問題はC++11発行前から認識されていたものの解決されず、緩和のためにC++20で範囲`for`に初期化式（`init-statement`）を書けるようになりましたが、本質的な問題は解決されていませんでした。C++23にてこの問題は解決され、範囲`for`の初期化式（`for-range-initializer`）内で作成されたすべての一時オブジェクトの寿命は範囲`for`文の完了（ループ終了）まで延長される、と規定されるようになります。

実際にどのように寿命延長が図られるのかは実装定義となりますが、これは同種の問題についてのより一般的な解決策を妨げないようにするためのことで、利用側から見れば範囲`for`の安全性は確実に向上します。

```cpp
// 右辺値を返す関数
auto f() -> std::vector<std::string>;

int main() {
  // C++20まではUB、C++23からはok
  for (auto&& c : f().at(0)) {
    std::cout << c << ' ';
  }
}
```

## `view`の２引数コンストラクタの`explicit`化

C++20の各種Rangeアダプタの`view`型の中には、コンストラクタで2引数以上を受け取るものがあり、そのコンストラクタは`explicit`指定されていませんでした。一方、C++23で追加された新しい`view`型では、2引数以上を受け取るコンストラクタは`explicit`指定がデフォルトになっています。

これによる差異は、コピーリスト初期化と呼ばれる形の初期化（`= {...};`の形の初期化）ができるかどうかの一点のみです。

```cpp
int main() {
  std::vector v = {1, 2, 3, 4};

  // C++20時点
  std::ranges::take_view r1 = {v, 1}; // ok
  std::ranges::filter_view r2 = {v, [](int) { return true; }};  // ok

  // C++23
  std::ranges::chunk_view r3 = {v, 1}; // ng
  std::ranges::chunk_by_view r4 = {v, [](int, int) { return true; }}; // ng
  
  std::ranges::chunk_view r5{v, 1}; // ok
  std::ranges::chunk_by_view r6{v, [](int, int) { return true; }}; // ok
}
```

この場合にコンストラクタに`explicit`が付加されていたとしても、特に何か利益があるわけではありません。しかし、このことはC++20と23の間で一貫していなかったため、最終的に`view`型の2引数以上のコンストラクタには`explicit`を付加することをデフォルトとすることに決定されました。

これによって、C++20の`view`型ではコピーリスト初期化ができなくなります。これは一応は快適変更ではありますが、標準ライブラリの`view`型は全てRangeアダプタ/ファクトリオブジェクトを通して取得することがデフォルトかつ推奨されているため`view`型を直接初期化することは稀であり、また、この変更の影響を受けたとしても`=`を削除する（リスト初期化に変更する）だけで問題を解決できるため、影響は小さいだろうと判断されたようです。

## *stashing iterator*の修正

*stashing iterator*は標準で定義されたものではありませんがイテレータカテゴリの一つで、イテレータが参照するものがそのイテレータ内部に保存されているイテレータの事です。従って、イテレータの間接参照結果が左辺値を返している時、その生存期間は取得元のイテレータの生存期間とリンクしています。

この性質のため*stashing iterator*は`input_iterator`にしかならず、そこから取得したものの扱いには注意が必要です。

Rangeライブラリの`view`型のものを除くと、C++20時点の標準ライブラリの*stashing iterator*としてカテゴライズされるイテレータとしては`std::regex_iterator`と`std::regex_token_iterator`の2つがありました。しかしこれらのイテレータのカテゴリは`forward_iterator`となっており、*stashing iterator*を考慮するとカテゴリ指定が間違っていました。

C++23のRangeアダプタは遅延評価を行うためにイテレータをやりくりしているため、このカテゴリの間違いが深刻な問題を引き起こす場合がありました。特に、`views::join`でそれが顕著でした。

```cpp
using std::ranges;

int main() {
  char const text[] = "Hello";
  std::regex regex{"[a-z]"};  // 小文字アルファベット1文字にマッチング

  // 範囲（sub_match）の範囲（match_results）
  subrange regex_range(std::cregex_iterator(
                          begin(text),
                          end(text),
                          regex
                       ),
                       std::cregex_iterator{}
                      );

    // string_viewの範囲
  auto lower = regex_range
    | views::join  // sub_matchの範囲へと平坦化
    | views::transform([](auto const& sm) {
        // sub_matchオブジェクトが参照する文字列範囲をstring_viewへ変換
        return std::string_view(sm.first, sm.second);
      });

  // elloを1文字づつ改行して出力する（はず
  for (auto const& sv : lower) {
    std::cout << sv << '\n';
  }
}
```

この例は`text`にある文字列から小文字だけを取り出すものです。

`std::cregex_iterator`は1つのマッチングについて（`std::match_results`）を列挙し、`std::match_results`はサブマッチ（`std::sub_match`、`()`による1つのグループを表す）を列挙する`range`です。この例では1つのマッチングにつきサブマッチは1つになりますが、構造としては範囲の範囲になっています。`regex_range`は`ranges::subrange`でそのイテレータペアをラップすることで`range`としてまとめたものです。

範囲の範囲に対して`views::join`によって平坦化を試みるのは自然なことで、これによってサブマッチ（`std::sub_match`）の範囲に変換されます。あとは、`views::transform`によってサブマッチ結果からマッチング文字列（この例では1文字ですが）を取り出して`std::string_view`に変換しています。結局、`lower`は`std::string_view`の範囲になっています。

一見問題なさそうなこのコードは、`views::transform`の処理内部でダングリングイテレータを発生させています。`regex_range`のイテレータは外側（`std::cregex_iterator`）も内側も*stashing iterator*ですが前述のようにイテレータカテゴリは`forward_iterator`となっています。本当に`forward_iterator`であればマルチパス保証によってイテレータのコピーが問題なく行えるため、`views::join`は入力Rangeの内外イテレータをコピーして保持して処理を行おうとします。まだこの時点では問題が無く、`views::transform`がその処理のために内部で`join_view`のイテレータをコピーしてしまうタイミングがあり、そこで`std::cregex_iterator`とそこから取り出されたものとの生存期間が分離し、ダングリングイテレータを発生させます。

この問題は複雑であり、規格の規定や実装のコードを詳しく追わないと何が起きているか分からないものがありますが、問題の主要因は次の2つです

1. `std::regex_iterator`（`std::cregex_iterator`）と`std::regex_token_iterator`のイテレータカテゴリの間違い
2. `join_view`（`join_with_view`も）が*stashing iterator*を正しく扱えていない

この問題の解決のため、それぞれ次のように修正されます

1. `std::regex_iterator`（`std::cregex_iterator`）と`std::regex_token_iterator`の`iterator_concept`を`input_itetaror`として定義
2. `join_view`（`join_with_view`も）は`input_range`の入力時に、外側イテレータを`view`オブジェクト内部にキャッシュする
    - *stashing iterator*を検出できないため、`input_range`全般に対して適用

これによって、上記の例を含めた標準ライブラリでの*stashing iterator*の扱いが改善されます。

## P2609R3?

\clearpage

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- Compiler Explorer(https://godbolt.org/)
- なぜ ranges::accumulate は難しいのか - Zenn(https://zenn.dev/acd1034/articles/221006-why-ranges-accumulate-is-difficult)
