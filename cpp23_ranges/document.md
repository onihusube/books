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

# Rangeアダプタ

## `ranges::range_adaptor_closure`

## Rangeアダプタのムーブオンリータイプへの対応

# Rangeアルゴリズム

## `find_last/find_last_if/find_last_if_not`

## `ranges::starts_with`/`ranges::ends_with`
## `ranges::contains`/`ranges::contains_subrange`

## `ranges::iota`

## `ranges::shift_left`/`ranges::shift_right`

## `ranges::fold`

## 非Rangeアルゴリズムでのコンセプトの使用

# `ranges::to`

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

*range-underlying-spec*オプションは要素型に対するオプションを指定するものです。要素型が`formattable`であれば、その型で利用可能なオプションをそのまま指定することができます。*range-underlying-spec*を指定する場合は、`:`から始まっている必要があります。

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

## `std::generator`

## `view`の２引数コンストラクタの`explicit`化

## *stashing iterator*の修正

P2770R0

## P2609R3?

\clearpage

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- Compiler Explorer(https://godbolt.org/)