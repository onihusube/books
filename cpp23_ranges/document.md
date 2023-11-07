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
    std::cout << std::format("{}, ", n);
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
    std::cout << std::format("{}, ", n);
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
- `const-iterable` : `const`修飾されているときでも`range`となるか
- `borrowed_range` : 範囲の寿命とそこから取得したイテレータの寿命が切り離されている
    - `borrowed_range`コンセプトのモデルとなるか

〇は常にその性質が有効であることを、×は常に無効であることを、特定の型やコンセプトが指定されている場合は常にそれを示すことを、条件が指定されている場合はその条件を満たす場合にその性質が有効になることを、それぞれ表しています。

　

C++23で追加されたRangeファクトリはこの`views::repeat`のみなので、以降のものは全てRangeアダプタになります。

## `views::as_rvalue`
## `views::as_cosnt`
## `views::enumerate`

`views::enumerate`は入力範囲の各要素にそのインデックスを紐付けた要素からなる範囲を生成する`view`です。

```cpp
int main() {
  std::vector vec = {1, 3, 5, 7, 11, 13};

  for (auto [index, value] : vec | std::views::enumerate) {
    std::cout << std::format("{}: {}\n", index, value);
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
    std::cout << std::format("{}: {}\n", index, value);
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
std::vector vec = {1, 3, 5, 7, 11, 13};

// enumerate_view構築時は何もしない
std::ranges::enumerate_view ev{vec};

// イテレータ取得時、内部カウンタ（インデックス）が0に初期化
auto it = vec.begin();

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

  std::cout << std::format("length = {}\n", zip1.size());
  std::cout << std::format("length = {}\n", zip2.size());
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

### 動作詳細

`zip_view`は構築されると入力範囲を`views::all`に通したものを`std::tuple`につめて保持しますが、それ以上のことはしません。実際の`zip`動作はそのイテレータが担っています。

`zip_view`のイテレータは入力範囲の全てのイテレータを保持しており、進行に伴ってはそれら全てのイテレータを同様に進行させます。

`zip_view`のイテレータの間接参照では、保持する全てのイテレータに対して`*i`した結果を直接`std::tuple`にラップしてその`std::tuple`オブジェクトを返します。前述のように、ここでは参照は参照のままラップされます。

```cpp
// 入力範囲の保存or参照のみを行う
auto zip = std::views::zip(...);

// 入力範囲のイテレータを取り出し保存する
auto it = zip.begin();

// 進行は保持するイテレータ全てに同じ操作を適用する
++it;
--it;
it += 3;
it -= 3;

// 保持するイテレータ全ての間接参照結果をtupleにラップして返す
auto tuple = *it;
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
## `views::adjacent`
## `views::adjacent_transform`
## `views::chunk`
## `views::slide`
## `views::chunk_by`
## `views::stride`
## `views::cartesian_product`

## `ranges::range_adaptor_closure`

# Rangeアルゴリズム

## `find_last/find_last_if/find_last_if_not`

## `ranges::starts_with`/`ranges::ends_with`
## `ranges::contains`/`ranges::contains_subrange`

## `ranges::iota`

## `ranges::shift_left`/`ranges::shift_right`

## `ranges::fold`

## 非Rangeアルゴリズムでのコンセプトの使用

# `ranges::to`

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


## Rangeアダプタのムーブオンリータイプへの対応
P2494R2

## P2609R3?

\clearpage

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- Compiler Explorer(https://godbolt.org/)
