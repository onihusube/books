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

```cpp
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

これは`view`/イテレータを実装する際に問題となり、実際にC++20のRangeアダプタの一部の`view`型ではこれを満足するために余計な複雑性を導入していました（`semiregular-box`の使用など）。

この要件は緩和され、`view`及び`input_iterator`/`output_iterator`はデフォルト構築可能である必要が無くなりました。また、これに伴って標準の`view`型やイテレータラッパの要件や実装も修正されています。

この修正に関しては機能テストマクロ`__cpp_lib_ranges`が`201911L`から`202106L`にバンプアップされています。

## Rangeアダプタの引数受け取りの修正

当初の仕様では、Rangeアダプタオブジェクトが受け取った追加の引数（非`range`引数）をどのように保持するのかは未規定でした。それによって実装間の差異とダングリング参照の危険性発生させていました。

```cpp
template<class F>
auto filter(F f) {
  // GCC : fを参照で保持する
  // MSVC : fを常にコピーして保持する
  return std::views::filter(f);
}

std::vector<int> v = {1, 2, 3, 4};

// 状態を持つ述語を渡す
auto f = filter([i = std::vector{4}] (auto x) { return x == i[0]; });

// GCCの場合、fにダングリング参照が含まれている
// MSVCは無問題
auto x = v | f;
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

## `split_view`と`lazy_split_view`

## `istream_view`の修正

## `owning_view`


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

## `std::format`の`range`対応

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
