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

文中では、`std::ranges`もしくは`std::views`から始まるものについては`std::`を省略しています。また、通常の関数の表記（`func()`）に対してメンバ関数の表記（`.member_func()`）は先頭に`.`を付加して区別します。

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

https://wg21.link/P0847R7

## アクセス制御の異なるメンバ変数のレイアウトを宣言順に規定

- P1847R4 Make declaration order layout mandated(https://wg21.link/P1847R4)

C++20までの規定では、クラスのアクセス制御（`private, public, protected`）が異なる場合、実装はデータメンバを並べ替えてメモリに配置することができます。この利点は、型のアライメントの違いによって生じるメンバ変数間のパディングを最小化して構造体サイズを削減することで、メモリ効率を向上させることにあります。

ただし、実際にそれを行う処理系は存在せず、多くのC++プログラマはは並べ替えを考慮していないことがほとんどです。この提案は、そのような慣行に従うように規定を修正し、クラスのデータメンバのメモリレイアウトが常にコード上の宣言順と一致するようにするものです。

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

以下余談です。

このクラスレイアウトの並べ替えの許可という仕様はC++11から入っていたものですが、意図して導入したものではなく、標準レイアウトクラスという分類を導入する際に誤って混入してしまったもののようです。幸いなことにその仕様を活用する処理系は現れなかったため、この度めでたく消すことができました。

## 添字演算子の多次元サポート

- P2128R6 Multidimensional subscript operator(https://wg21.link/P2128R6)

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

と言いますか、この提案のモチベーションの大きな部分は`std::mdspan`でこれを使用できるようにするためでした。

## static operator()

https://wg21.link/P1169R4

# 定数式

## if consteval

https://wg21.link/P1938R3

## 定数式の文脈での bool への縮小変換を許可

https://wg21.link/P1401R5

## 定数式内での非リテラル変数、静的変数・スレッドローカル変数およびgotoとラベルの存在を許可する

https://wg21.link/P2242R3

## 静的な診断メッセージの文字エンコーディング

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2246r1.pdf

## constexpr 関数が定数実行できない場合でも適格とする

https://wg21.link/P2448R2

## constexpr 関数内での static constexpr 変数を許可

https://wg21.link/P2647R1

## constexpr 関数内で consteval 関数を呼び出せない問題を軽減

https://wg21.link/P2564R0

## 定数式における未知の参照の利用の許可

https://wg21.link/P2280R4

# テンプレート

## 変数テンプレートの部分特殊化の仕様明確化

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2096r2.html

## 継承コンストラクタからのクラステンプレート引数の推論

https://wg21.link/P2582R1

# 文字・文字列リテラル

## 異なる文字エンコーディングをもつ文字列リテラルの連結を不適格とする

https://wg21.link/P2201R1

## エスケープシーケンスの区切り	

https://wg21.link/P2290R3

## 文字・文字列リテラル中の数値・ユニバーサルキャラクタのエスケープに関する問題解決

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2029r4.html

## 1ワイド文字に収まらないワイド文字リテラルを禁止する

https://wg21.link/P2362R3

## 名前付きユニバーサルキャラクタ名

https://wg21.link/P2071R2

## wchar_t の制限の緩和

https://wg21.link/P2460R2

# ラムダ式

## ラムダ式で () を省略できる条件を緩和

https://wg21.link/P1102R2

## ラムダ式に対する属性	

https://wg21.link/P2173R1

## ラムダ式の後置戻り値型のスコープ変更

https://wg21.link/P2036R3
https://wg21.link/P2579R0

# プリプロセッサ

## 文字リテラルエンコーディングを一貫させる

https://wg21.link/P2316R2

## elif / elifdef / elifndef のサポートを追加

https://wg21.link/P2334R1

## #warning のサポートを追加	

https://wg21.link/P2437R1

## 汎用的なソースコードのエンコーディングとしてUTF-8をサポート

https://wg21.link/P2295R6

# その他

## decay-copy

https://wg21.link/P0849R8

## size_t リテラルのためのサフィックス

https://wg21.link/P0330R8

## 暗黙ムーブを簡略化

https://wg21.link/P2266R3

## コード内容の仮定をコンパイラに伝えるassume属性

https://wg21.link/P1774R8

## 初期化文での型の別名宣言を許可

https://wg21.link/P2360R0

## 範囲for文が範囲初期化子内で生じた一時オブジェクトを延命することを規定

https://wg21.link/P2718R0

## 拡張不動小数点型のサポート

https://wg21.link/P1467R9

# とてもマイナーな変更

## スコープと名前ルックアップの仕様整理

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p1787r6.html

## 複合文の末尾へのラベルを許可

https://wg21.link/P2324R2

## 参照するPOSIX規格を更新

http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2227r0.html

## 行末スペースを無視するよう規定

https://wg21.link/P2223R2

## GCサポートの削除

https://wg21.link/P2186R2

## P2314R0 Character sets and encodings

https://wg21.link/P2314R4

# Defect Report

## 無意味なexport宣言を禁止する

https://wg21.link/P2615R1

## 識別子に使用可能な文字の制限

https://wg21.link/P1949R7

## 属性指定の重複の許可

https://wg21.link/P2156R1

## P2943R0

https://wg21.link/P2493R0

## ==演算子の導出の調整

https://wg21.link/P2468R2

## char8_t互換性の修正

https://wg21.link/P2513R4

## CWG2518

https://cplusplus.github.io/CWG/issues/2518.html

\clearpage

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- Compiler Explorer(https://godbolt.org/)
