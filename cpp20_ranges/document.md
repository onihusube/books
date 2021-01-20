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

Rangeライブラリのものは`std::ranges`名前空間にありますが、この本では基本的に省略します。ただし、Rangeライブラリ以外のものは`std::`を省略しません。

## Rangeライブラリとは

# コンセプト

C++20よりコンセプトが言語機能に正式に組み込まれました。これによってテンプレートパラメータに制約をかけたり、関数テンプレートに優先順を付ける事がC++17以前と比較してより簡単に行えるようになりました。それとともに、ライブラリにおける処理や操作の対象となる”もの・概念”をコンセプトによって明確に定義し、そのコンセプトを中心にプログラムをジェネリックに組み上げていくコンセプトベースのライブラリデザインの時代が幕を開けています。

Rangeライブラリの元となったRange-v3ライブラリはコンセプトをエミュレートする事でC++17の時代からコンセプトベースで設計されていたライブラリで、それを受けてRangeライブラリも当初からコンセプトベースで設計されており、ライブラリの中心となる概念をコンセプトによって表現しています。

コンセプトベースでしっかりと設計されたライブラリはまだあまりありませんが、C++23に導入予定のExecutorライブラリもコンセプトをベースとした非常にジェネリックなライブラリとして設計されています。

## `range`

Rangeライブラリの中心概念である __範囲（*range*）__ を定義するのが`range`コンセプトです。コンセプトによって、C++のコードとして次のように定義されます。

```cpp
template<class T>
concept range =
  requires(T& t) {
    ranges::begin(t); // equality-preservingが要求されることがある
    ranges::end(t);
  };
```

ここに表れている`requires(引数){式...}`は`requires`式と呼ばれる制約式で、そこに記述された式がコンパイルエラーを起こさない場合に`true`を返すものです。コンセプトは複数の制約式から構成され、それらを論理積（`&&`）や論理和（`||`）で繋げて構成します。具体的な型が渡されると、制約式1つ1つは`true/false`のどちらかを返し、全体として`true`となるときにそのコンセプトはその型について満足されます。

`range`コンセプトは1つの`requires`式だけから構成されています。その意味は単純で、型`T`のオブジェクト`t`に対して`std::ranges::begin(t)/std::ranges::end(t)`がどちらも呼び出し可能である事です。`std::ranges::begin()/std::ranges::end()`はそれぞれ範囲の先頭イテレータと終端を表す番兵を取得するものなので、`range`コンセプトを満たす型は`std::ranges::begin()/std::ranges::end()`が指定する手段によって範囲の先頭と終端を取得できることを表しています。

長いので、以降`std::ranges::begin()/std::ranges::end()`を単に`begin/end`と書くことにします。

そして、これに加えて次のような条件が文章で規定されます。

- `[begin(t), end(t))`が範囲を示す
- `begin(t), end(t)`の計算量は償却定数
- `begin(t)`が`std::forward_iterator`コンセプトのモデルであるならば、`begin(t)`は等さを保持する（*equality-preserving*）

標準ライブラリのコンセプトの多くは、このような文章によって追加の要求がなされていることがあります。そのような要求の多くは、そのコンセプトを満たしていれば当然満足されているはずですが、コンセプト機能によってコードとしてチェックできるものではない事が指定されています。このような要件のことを __意味論的な要件（*semantic requirements*）__ と呼び、コンセプト定義に直接現れている制約（式）を __構文的な要件（*syntactic requirements*）__ と呼びます。

コンセプトは構文的な要件と意味論的な要件の両方が揃ってコンセプトです。コンセプトを利用するときは構文的な部分だけでなく意味論的な部分にも注意を払う必要があります。ある型`T`が、コンセプト`C`の構文的な要件と意味論的な要件の両方を全て満足している時、「`T`は`C`のモデルである」と言います。

話を戻して、`range`コンセプトの意味論的な要件を見てみましょう。

> `[begin(t), end(t))`が範囲を示す

これは、同じオブジェクトから取得したイテレータと番兵は、同じ範囲についてのイテレータと番兵である事という意味です。ここでの範囲とは、配列やコンテナなどの要素の列となっているものを指します。  
例えば、`begin`と`end`がそれぞれ異なる配列のイテレータを返すような実装になっている場合、`range`コンセプトを構文的には満たしますがモデルとはなりません。

> `begin(t), end(t)`の計算量は償却定数

ここでの計算量とはシーケンス`t`の要素数に基づくものです。すなわち、`t`の要素数が幾つであれ`begin/end`の呼び出しは一定時間で完了しなければなりません。償却とあるのは、繰り返し読んだ時に結果的に一定時間となれば良いという意味で、例えば最初の呼び出しだけ何か処理をして結果をキャッシュしておいて、2回目以降の呼び出しはキャッシュから直ぐに返す、という実装が許されます。

> `begin(t)`が`std::forward_iterator`コンセプトのモデルであるならば
> `begin(t)`は等さを保持する（*equality-preserving*）

2行に分けて見ていきます。

`std::forward_iterator`コンセプトは名前の通り、従来*forward iterator*と呼ばれていた種類のイテレータを定義するコンセプトで、C++20から`<iterator>`にて定義されています。  

ある式が等さを保持する（*equality-preserving*である）とは、ある式に同じ値を入力するといつも同じ値が得られる事を言います。簡単に言えば、式の内部で状態を持ったりグローバルな変数に依存したりしていない、という事です。

この意味論的な要件はつまり、`begin(t)`から得られるイテレータが*forward iterator*であるならば、`begin(t)`は同じ`t`に対して同じイテレータを返すべし、という事です。

## `borrowed_range`
## `sized_range`

## `input_range`
## `output_range`
## `forward_range`
## `bidirectional_range`
## `random_access_range`
## `contiguous_range`
## `common_range`

## `view`
## `viewable_range`

# Rangeライブラリの構成部品

## Rangeアクセス関数

### イテレータ/番兵の取得

### 範囲サイズの取得

### 範囲のポインタ取得

## Range情報取得

### `iterator_t/sentinel_t`
### `range_difference_t`
### `range_size_t`
### `range_value_t`
### `range_reference_t`
### `range_rvalue_reference_t`

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