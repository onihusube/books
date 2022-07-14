---
title: C++20 ライブラリ機能 1
author: onihusube
date: 2022/08/14
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

本書は、C++20で導入された新しいライブラリ機能についての紹介と解説を行うものです。コア言語機能と同様にC++20のライブラリ拡張もそこそこ大きな範囲に及んでおり、全般的に使用できるような機能（*vocabulary type* : 語彙型、と呼ばれます）だけではなく特定の用途において有用なものもあるなど、その全体像を理解するには多くの提案や規格文書を読む必要があるとともに、多様な前提知識も必要とします。そのため、ただ変更点を眺めていたりライブラリリファレンスを見ているだけでは、必ずしもその機能の意義や有用性を把握できず、C++20ライブラリの全体像を掴む事ができません。

本書は、C++20の新機能に興味はあるもののついて行けないという人や、ある程度C++20機能を知っているものの深入りできていないという人向けに、C++20で新しく導入されたライブラリ機能の内容や意義についての解説を試みるものです。

この本で取り上げるライブラリ機能は主に、新しく導入されたヘッダを中心とした大きな機能です。既存ライブラリの改善などの小さめの機能は後ほど刊行（予定）のライブラリ機能 2で紹介する予定です。

なお、本書ではC++17までのライブラリ機能に関しては前提知識として説明しません。また、C++20のコア言語機能に関しては拙著『C++20 コア言語機能』をご参照ください。

## サンプルコードのお約束

- そこで主題となっているライブラリ機能のためのヘッダのみを明示的にインクルードし、他のヘッダのインクルードは省略します。
- 主題と関係ないところを`...`で省略していることがあります。

コードブロック中で標準出力をしている時、直後のブロックでその出力例を示していることがあります。例えば

```cpp
int main() {
  std::cout "hello world!";
}
```
```
hello world!
```

\clearpage

# `<concepts>`
\clearpage
# `<ranges>`

## Rangeコンセプト
## Rangeアダプタ
## Rangeアルゴリズム
\clearpage

# `<format>`

`<format>`ヘッダでは、`printf()`のように書式文字列によって書式を指定して値を文字列へ変換する、文字列フォーマット機能を提供します。

この新しい文字列フォーマット機能は、`printf()`とは異なり型安全であり、`iostream`のマニピュレータのようにフォーマット指定がストリームに状態として保存されたりしないなど、これまでC++で使用可能だったフォーマット機能の問題点を改善しています。さらに、任意個数引数の自然な扱いやコンパイル時の書式文字列チェックなど、ユーザビリティも向上させるものです。

この`<format>`の機能は`{fmt}`というライブラリをベースとして導入されており、こちらのライブラリはC++17以前のコンパイラでもほぼ同じように使用できます。

## `std::format()`

基本的には、`std::format()`という関数によって文字列フォーマットを行います。この関数は第1引数に書式文字列（フォーマット文字列）を取り、2つ目以降の引数で文字列化したい任意のオブジェクトを指定します。`std::format()`の戻り値は指定された書式に従って文字列化された出力を保存した`std::string`オブジェクトです。

```cpp
#include <format>

int main() {
  // std::stringにフォーマットして出力
  std::string str = std::format("{} {}! C++{}", "hello", "world", 20);

  // そのまま標準出力へ
  std::cout << str;
}
```
```
hello world! C++20
```

フォーマット文字列の文法は、Pythonの文字列フォーマット（`str.format()`）のものをベースとしたものです。フォーマット文字列中の`{}`は置換フィールドとして扱われ、`std::format()`の2つ目以降に指定された引数がその順番で対応する`{}`を置き換える形で文字列化されていきます。つまり、`{}`は`printf()`の`%d`などに当たりますが、それとは異なり`{}`は基本的に型に依存せず（型の指定をせずとも）置換されます。

とりあえずは素の`{}`を単純に使用することにして、フォーマット文字列の構文詳細は後ほど説明します。

フォーマット文字列を`L"{}"`のように`wchar_t`文字列にすると、`std::format()`はワイド文字列を用いたフォーマットを行うようになり、戻り値は`std::wstring`オブジェクトになります。この場合はフォーマット対象の引数として文字列を渡す際に自動で文字コード変換されたりしないので、渡す文字列はワイド文字列となるようにする必要があります。

```cpp
#include <format>

int main() {
  std::wcout << std::format(L"{} {}! C++{}", L"hello", L"world", 20); // ok
  std::wcout << std::format(L"{} {}! C++{}",  "hello",  "world", 20); // ng
}
```

また、どちらの文字列型の`std::format`に対しても、フォーマット文字列の前に`std::locale`オブジェクトを渡すことでロケール依存のフォーマットを行うことができます。その場合、ロケール依存フォーマットを行いたい引数に対する置換フィールドは`{:L}`のようにしてロケール依存フォーマット指定を明示します。

```cpp
#include <format>

int main() {
  // ドイツ語ロケールによるフォーマット
  std::cout << std::format(std::locale{"german"}, "{:L}", 3.1415);
}
```
```
3,1415
```

（ドイツ語ロケールでは、小数点はピリオドではなくカンマになります）

`std::locale`オブジェクトを明示的に指定しなくてもロケール依存フォーマット（`{:L}`）は有効ですが、このオーバーロードを用いることでグローバルロケールではなく個別のロケールに従ってそれを行わせることができます。

## コンパイル時フォーマット文字列チェック

ここまではほぼ`{}`しか用いていませんが、フォーマット文字列中の置換フィールド`{}`は小さなDSLでありもう少し複雑な構文になっています。それはC++の他の部分の構文と馴染みがあるものではないため、フォーマット文字列は容易に間違える可能性があります。

そのため、`std::format()`ではフォーマット文字列の妥当性がコンパイル時にチェックされ、もし間違えて使用していた場合はコンパイルエラーとして早期に知ることができます。

```cpp
#include <format>

int main() {
  // どちらもコンパイルエラー
  std::cout << std::format("{} {}", 10); // 引数が足りない
  std::cout << std::format("{:l}", 10);  // Lは大文字
}
```

この実装詳細はコア言語機能本をご覧ください。完全にライブラリサイドで実装されているため必ずしもエラーメッセージが優しくない場合が多いですが、間違っていることにはすぐに気づくことができます（本家の`{fmt}`ライブラリではかなりわかりやすいメッセージが出力されるので、いずれ改善される可能性はあります）。

このチェックのために、`std::format()`のフォーマット文字列にはコンパイル時に確定している文字列しか使用できません。`std::string`等、実行時に内容が確定する文字列によってフォーマット文字列を指定したい場合には、`std::vformat()`を使用します。

```cpp
#include <format>

void runtime_fmt(const std::string& fmt, auto&&... args) {
  std::cout << std::vformat(fmt, std::make_format_args(args...));
}

int main() {
  runtime_fmt("C++{}\n", 20);
  runtime_fmt("{} {}", "hello", "world");
}
```
```
C++20
hello world
```

`std::vformat()`の場合、フォーマット文字列の間違いは実行時に`std::format_error`例外が投げられることで知ることができます。

`std::vformat()`は`std::format()`の内部実装関数でもあり、フォーマット対象の引数列を型消去して受け取ります。そのため引数列をそのまま渡すことはできないので、`std::make_format_args()`を介して型消去してから渡す必要があります。それ以外の部分（ワイド文字列とロケールなど）は`std::format()`と同様に使用できます。

## 出力先の変更

`std::format()`の出力先は内部で新規に構築された`std::string`固定であり、既存の文字列に出力したり、あるいは他のコンテナなどに保存することはできません。`std::format_to()`を用いると、出力イテレータによって任意の範囲へ出力することができます。

```cpp
#include <format>

auto out(std::output_iterator<char> auto oi, auto&& v) {
  return std::format_to(oi, "Log : {}\n", v);
}

int main() {
  std::string log;
  auto oi = std::back_inserter(log);

  oi = out(oi, 20);
  oi = out(oi, "logging");
  oi = out(oi, 2.718);

  std::cout << log;
}
```
```
Log : 20
Log : logging
Log : 2.718
```

`std::format_to()`の第1引数には出力先のイテレータ（出力イテレータ）を渡して、2つ目以降の引数は`std::format()`の（非ロケールオーバーロードの）引数と同じです。`std::format_to()`の戻り値は出力した文字数分進めた第1引数に渡されたイテレータで、出力した文字数を`N`とすれば`std::next(oi, N)`を返します。つまり、出力済み文字列の次の位置を指すイテレータが得られます。複数回呼び出すときはこの戻り値のイテレータを次の出力イテレータとして使用していくことで、同じ場所に連続して出力していくことができます（ただし、よく見るとこの例ではそれは意味をなしていません・・・）。

ところでこのような場合、各出力の前に出力先（例では`std::string`）に`.reserve()`があるならそれをしておきたくなると思います。そのためには出力される文字数を知る必要があり、`std::formatted_size()`によってフォーマットされて出力される文字列の長さを取得できます。`std::formatted_size()`の引数は`std::format()`と完全に同じで、戻り値としてその引数列によってフォーマットした際の出力文字列の長さを`std::size_t`で返します。

```cpp
#include <format>

void out(std::ranges::output_range<char> auto& r, auto&& v) {
  // フォーマット文字列
  constexpr std::string_view fmt_str = "Log : {}\n";

  // 出力される文字列長
  std::size_t N = std::formatted_size(fmt_str, v);

  // メモリ確保してから出力
  r.reserve(N);
  std::format_to(std::back_inserter(r), fmt_str, v);
}

int main() {
  std::string log;

  out(log, 20);
  out(log, "logging");
  out(log, 2.718);

  std::cout << log;
}
```

`std::format_to()`を使用する場合には、出力先が固定長のバッファなときなど出力可能なサイズ（最大出力文字数）を制限したくなることがあります。`std::format_to_n()`を用いると、`std::format_to()`による出力時に出力文字数制限をかけることができます。`std::format_to()`に対する`std::format_to_n()`の引数は、第1引数の出力イテレータの次の引数で最大文字数を整数値で受け取り、それ以降の引数は同じです。

```cpp
#include <format>

auto out(std::output_iterator<char> auto oi, int n, auto&& v) {
  // 最大n文字まで出力する
  return std::format_to_n(oi, n, "Log : {}\n", v);
}

int main() {
  // 20文字までしか出力できない
  char buf[20]{};

  auto oi = std::ranges::begin(buf);
  int limit = std::ranges::size(buf) - 1; // \0の分引いておく

  // 出力後のイテレータと出力文字数が得られる
  auto [i1, n1] = out(oi, limit, 20);
  // それを用いてoiとlimitを更新する
  oi = i1;
  limit -= n1;

  auto [i2, n2] = out(oi, limit, "logging");
  oi = i2;
  limit -= n2;  // 負の値になる
  
  auto [i3, n3] = out(oi, limit, 2.718);  // ok、正しく動作する

  std::cout << std::string_view{buf};
}
```
```
Log : 20
Log : logg
```

`std::format_to_n()`の戻り値は`std::format_to()`の出力と同様のイテレータと出力した文字数の2つをメンバとしてもつ集成体（`std::format_to_n_result`型）オブジェクトで、例のように構造化束縛で受け取ることができます。

この例では2回目の`out()`の呼び出しで文字数制限に到達しており、その後残りの出力可能文字数を表す`limit`変数は負の値になりますが、この`limit`を制限値として渡しても`std::format_to_n()`は正しく動作し、内部で負の値は`0`に丸められます。

これらの関数について、`std::format_to()`には`std::format()`と同様に`std::vformat_to()`があり、それら及び`std::format_to_n()`にはワイド文字列とロケールを受け取るオーバーロード（ロケール引数位置はフォーマット文字列の直前）があります。

各関数とオーバーロード有無などのまとめ

|関数名|`v`付関数|ワイド文字列|ロケール|
|---|:-:|:-:|:-:|
|`std::format()`|`std::vformat()`|○|○|
|`std::format_to()`|`std::vformat_to()`|○|○|
|`std::format_to_n()`|×|○|○|
|`std::formatted_size()`|×|○|○|
|`std::vformat()`| - |○|○|
|`std::vformat_to()`| - |○|○|

## 組み込み型のフォーマット文字列の構文

フォーマット文字列は、出力したい文字列をベースとして適宜変数の内容を入れ込みたい部分に置換フィールド（`{}`）を当てていく形の文字列で、置換フィールド以外の部分に制約はありません。フォーマット文字列構文は1つの置換フィールドの中に追加のオプションを指定する際の構文です。先ほど出てきたロケール依存フォーマット指定（`{:L}`）もこの一種です。

可能なオプションの全体像は次のようになっています

```
{ index : fill align sign alt(#) pad(0) width .precision locale(L) type }
```

これは1つの置換フィールド（`{...}`）を表しています。スペース区切りの各単語はオプション種類（グループ）を表すものであり指定する値そのものではなく、実際の指定時はスペースは入りません。また、`:`は*index*以外のオプションを指定する際に必須の文字であり、`.`は*precision*を指定する際に必須の文字です。*alt(#)*と*pad(0)*と*locale(L)*は指定可能な文字がそれぞれ`#`と`0`と`L`しかないことを表しています。

ここまでそうしてきたように全てのオプションは省略可能であり、これら9個のオプションはそれぞれ自由に組み合わせて使用することができます。ただし、フォーマット対象の型（その置換フィールドに対応する変数の型）によってオプションの振る舞いが変化し、一部のオプションは数値型でのみ有効です。

なお、これらオプションの並び順はこの順序でなくてはなりません。間を省略することはできますが並び順が前後してはなりません。

### index

まず最初の*index*オプションは、その置換フィールドに対応するフォーマット対象引数のインデックスを指定するものです。これによって、`std::format()`に渡す引数と異なる順序で文字列化することができます。

オプションには対応する引数の0始まりのインデックスを正の整数値で指定します。同じインデックスを複数回指定して同じ変数を複数回フォーマットすることもでき、インデックス指定しないことで一部の変数を使用しないようにもできます。

```cpp
#include <format>

int main() {
  std::cout << std::format("{1}{0}", 20, "C++");
  std::cout << std::format("{1}{0}{0}", 20, "C++");
  std::cout << std::format("{1}", 20, "C++");
}
```
```
C++20
C++2020
```

この時、インデックスが実際の引数の数以上の値であってはならず、インデックス指定する場合は全ての置換フィールドにインデックス指定が必要です。

```cpp
#include <format>

int main() {
  // 全てコンパイルエラー
  std::format("{1}{}", 20, "C++");      // 全てにインデックスが必要
  std::format("{2}{1}{0}", 20, "C++");  // 引数は2つ（インデックスは0か1）
}
```

前述のように、このフォーマット指定の間違いはコンパイルエラーとなります。

このインデックス指定以外のオプションがある場合は、`:`に続いて記述していきます（`:`だけでもエラーではありません）。

```cpp
#include <format>

int main() {
  std::format("{1:}{0:L}", 20, "C++");  // ok
}
```

### sign

順番に見ていきたいのですが、*fill*と*align*は*width*との関連性が強いためそちらでまとめて説明することにします。

すると、次に現れるのは*sign*です。これは名前の通り数値型の符号の表示に関するオプションであり、数値型（`bool`除く）でのみ有効です。

指定可能なのは次の3つです

|*sign*|意味|
|---|---|
|`+`|常に符号表示|
|`-`|負の数のみ符号表示（デフォルト）|
|スペース|正の数の符号部分にスペースを挿入|

```cpp
#include <format>

int main() {
  std::cout << std::format("{:-}, {:-}\n", 20, -20);
  std::cout << std::format("{:+}, {:+}\n", 20, -20);
  std::cout << std::format("{: }, {: }\n", 20, -20);
}
```
```
20, -20
+20, -20
 20, -20
```

オプションを指定しない場合のデフォルトは`-`であり、負の数にのみ符号が付加されます。

このオプションは符号なし整数型に対しても有効で、そちらの場合は`-`がつく事がないだけです。

```cpp
#include <format>

int main() {
  std::cout << std::format("{:-}\n", 20u);
  std::cout << std::format("{:+}\n", 20u);
  std::cout << std::format("{: }\n", 20u);
}
```
```
20
+20
 20
```

なお、`-0.0`は負の数として扱われます。

```cpp
#include <format>

int main() {
  std::cout << std::format("{:+}, {:+}\n", +0.0, -0.0);
}
```
```
+0, -0
```

### alt

次に出てくる*alt*は数値型の代替形式表示に関するオプションです。そのため、数値型（`bool`除く）でのみ有効であり、指定可能なのは`#`のみです。

この場合の数値型は整数型と浮動小数点数型の2つに分けられ、整数型の代替形式表示とはその基数によるプリフィックス（16進数の`0x`など）を表示させるもので、浮動小数点数型の代替形式表示は小数点以下が省略可能でも小数点を常に表示させるものです。

```cpp
#include <format>

int main() {
  std::cout << std::format("{0:x}, {0:#x}\n", 0xabcd);
  std::cout << std::format("{0:}, {0:#}\n", 1.0);
}
```
```
abcd, 0xabcd
1, 1.
```

整数型で基数のプリフィックスを表示させるにはまず整数値を何進数で表示させるかを指定する必要があり、これは*type*オプションの役割です。ここでは説明のために先取りして、`{:x}`の指定は整数値を16進数で表示させるものです。他にも2進と8進表示が可能であり、それぞれ`0b, 0`のプリフィックスがつきます（10進表示にはプリフィックスはありません）。

### pad と width

次に出てくるのは*pad*ですがこれは次の*width*と組み合わせて使うものなので、先に*width*を見てみます。

*width*はその名前の通り出力する文字列の幅を指定するオプションです。ここでの文字列幅というのは`std::format()`の出力文字列全体ではなくて、1つの（*width*指定のある）置換フィールドを置き換える文字列の幅です。

*width*に指定可能なのは正の整数値で、この値によって置換フィールドの置換後文字列幅の最小値を指定します。

```cpp
#include <format>

int main() {
  std::cout << std::format("|{:3}|\n", 1);
  std::cout << std::format("|{:3}|\n", 10);
  std::cout << std::format("|{:3}|\n", 100);
  std::cout << std::format("|{:3}|\n", 1000);
}
```
```
|  1|
| 10|
|100|
|1000|
```

ここでの前後の`|`は、出力文字幅の視認性を上げるために付加しているだけです。

出力文字列が指定幅内に収まっている場合、余ったところはホワイトスペースで埋められます。なお、*width*に指定するのは最小幅なので置換後文字列幅がそれを超える場合は無視されます。

数値型（`bool`除く）のフォーマット時には*width*の前に*pad*オプション（`0`）を指定することで、数値の前を0でパディングできます。

```cpp
#include <format>

int main() {
  std::cout << std::format("|{:05}|\n", 1);
  std::cout << std::format("|{:+05}|\n", 1);
  std::cout << std::format("|{:#05x}|\n", 0xf);
  std::cout << std::format("|{:+#05x}|\n", -0xd);
  std::cout << std::format("|{:05}|\n", 3.14);
}
```
```
|00001|
|+0001|
|0x00f|
|-0x0d|
|03.14|
```

このように、*pad*による0埋めは符号や基数プリフィックスの後ろに行われます。

なお、浮動小数点数型の`NaN`や`inf`の値に対しては、*pad*を指定してあっても0埋めは行われません。

```cpp
#include <format>

int main() {
  double inf = 1.0/0.0;
  std::cout << std::format("|{:05}|\n", inf);
}
```
```
|  inf|
```

### fill align と width

*width*を見たところで、最初に飛ばしていた*fill*と*align*に戻ります。この2つのオプションは*width*と一緒に使用して、空いたところの穴埋めと幅寄せの指定を行うものです。

まず、*align*オプションは*width*の幅の中で文字列の寄せを指定するオプションです。

指定可能なのは次の3つです

|*align*|意味|
|---|---|
|`>`|右寄せ（デフォルト）|
|`<`|左寄せ|
|`^`|中央寄せ|

```cpp
#include <format>

int main() {
  std::cout << std::format("|{:>6}|\n", 20);
  std::cout << std::format("|{:<6}|\n", 20);
  std::cout << std::format("|{:^6}|\n", 20);
}
```
```
|    20|
|20    |
|  20  |
```

次に、*fill*オプションは幅指定時に余った部分を埋める文字を指定するオプションです。デフォルトはホワイトスペースが使用されています。

```cpp
#include <format>

int main() {
  // *を穴埋めに使用
  std::cout << std::format("|{:*>6}|\n", 20);
  std::cout << std::format("|{:*<6}|\n", 20);
  std::cout << std::format("|{:*^6}|\n", 20);
}
```
```
|****20|
|20****|
|**20**|
```

*fill*オプションに指定できるのは任意の1文字で絵文字なども指定できますが、単一のユニコード値によって表現可能なものでなければなりません。つまり`char32_t`1つで表現可能な文字ということで、結合文字などによって見た目1文字が複数の文字（ユニコード値）から構成されているような文字を使用できません。`std::format()`の場合、そのような文字を指定するとコンパイルエラーとなります。

*width*による幅の指定や*fill*による穴埋めなどにおける1文字の幅とは、ターミナルなどに表示した時の文字幅とはほぼ無関係です。出力される文字の幅はその文字によって幅1か2としてカウントされ、*fill*に指定した穴埋め文字は常に幅1として穴埋めが行われます。

```cpp
#include <format>

int main() {
  std::cout << std::format("|{:†^6}|\n", "神");
  std::cout << std::format("|{:†^6}|\n", "神神");
  std::cout << std::format("|{:猫^6}|\n", "犬");
}
```
```
|††神††|
|†神神†|
|猫猫犬猫猫|
```

この例はすべて幅6に収まってフォーマットされています。

*fill*には`{`と`}`を除く任意の1文字が指定可能ですが、これは明らかに後続のオプションの指定文字とバッティングしています。その曖昧性を回避するために、*fill*を指定する場合は*align*の指定が必須となっており、*align*が指定されない場合は*fill*も指定されていないものとして扱われます。

```cpp
#include <format>

int main() {
  // これはfillとalignの指定
  std::format("{:^6}\n", 20);   // alignのみ
  std::format("{:+^6}\n", 20);  // +でfill
  std::format("{:0^6}\n", 20);  // 0でfill
  std::format("{:#^6}\n", 20);  // #でfill

  // これは他のオプションの指定
  std::format("{:+6}\n", 20); // sign
  std::format("{:#6}\n", 20); // alt
  std::format("{:06}\n", 20); // pad

  // 曖昧では無い場合はコンパイルエラー
  std::format("{:*6}\n", 20); // ng
  std::format("{:!6}\n", 20); // ng
  std::format("{:a6}\n", 20); // ng
}
```

### precision

*precision*は浮動小数点数型の精度を指定するオプションです。*precision*に指定可能なのは正の整数値のみにで、必ず`.`から始まる必要があります。

この精度とは小数点以下の桁数のことではなく、小数点を除いた数値の桁数の指定です。デフォルト（指定しない場合）は6が使用されています。

```cpp
#include <format>

int main() {
  double v = 3.141592653589793;
  std::cout << std::format("{:.1}\n", v);
  std::cout << std::format("{:.5}\n", v);
  std::cout << std::format("{:.10}\n", v);
}
```
```
3
3.1416
3.141592654
```

数値が指定された桁数に満たない場合は、可能なところまで文字列化されます（スペースや0で埋められたりはしません）。

```cpp
#include <format>

int main() {
  double v = 3.14;
  std::cout << std::format("|{:.5}|\n", v);
}
```
```
|3.14|
```

### width と precision の動的な指定

*width*と*precision*の2つのオプションは必ずしもコンパイル時の文字としての静的な値だけではなく、実行時に指定した任意の値を使用したいことがあるでしょう。そのために`std::vformat()`を使用しても良いかもしれませんが、フォーマット文字列そのものを動的に生成しなければならないため少し面倒です。

そこで、*width*と*precision*の2つのオプションは数値の代わりに置換フィールド`{}`を指定することで、`std::format()`に渡された整数値を用いて指定することができます。

```cpp
#include <format>

int main() {
  // widthの置換フィールドによる指定
  std::cout << std::format("|{:*>{}}|\n", 20, 3);

  // precisionの置換フィールドによる指定
  double v = 3.141592653589793;
  std::cout << std::format("{:.{}}\n", v, 3);

  // 両方
  std::cout << std::format("|{:*>{}.{}}|\n", v, 6, 3);
}
```
```
|*20|
3.14
|**3.14|
```

置換フィールド内でさらに置換フィールドを指定できるのはこれら2つのオプションだけです。その際には、*width*と*precision*に指定された置換フィールドは囲む置換フィールドの次のインデックスから渡された変数を参照します。つまりは、ネストしていること以外は参照先に関しては通常の置換フィールドと同様です。

このネストした置換フィールドには*index*オプションだけが指定できます。この振る舞いも通常の置換フィールドと同様です。

```cpp
#include <format>

int main() {
  double v = 3.141592653589793;
  std::cout << std::format("|{0:*>{2}.{1}}|\n", v, 6, 8);
}
```
```
|*3.14159|
```

注意点として、内部の置換フィールドに*index*オプションを与える場合はそのフォーマット文字列に含まれるすべての置換フィールドに*index*指定が必要です。これも通常の置換フィールドと同様のことです。

```cpp
#include <format>

int main() {
  double v = 3.141592653589793;

  // すべてコンパイルエラー
  std::cout << std::format("|{:*>{2}.{1}}|\n", v, 6, 8);
  std::cout << std::format("|{:*>{2}.{}}|\n", v, 6, 8);
  std::cout << std::format("|{0:*>{2}.{1}}| :{}\n", v, 6, 8);
}
```

### locale

*locale*オプションは、フォーマット対象の値について現在の（あるいは指定された）ロケールの代替表現があればそれを用いてフォーマットを行うオプションです。指定可能なのは`L`だけです。これは`std::format()`のロケール依存フォーマットの説明のところで既に使用していました。

`L`オプションが省略された場合はロケール非依存（常にCロケール）でフォーマットされ、`L`オプションが指定されたときは`std::format()`の第一引数に指定されたロケールかグローバルロケールを使用してフォーマットされます。

```cpp
#include <format>

int main() {
  // グローバルロケール（日本語）
  std::locale::global(std::locale("ja_JP"));
  // ローカルロケール（ドイツ語）
  auto loc = std::locale{"german"};

  // グローバルロケールを使用
  std::cout << std::format("{:L}\n", 1000);
  std::cout << std::format("{:L}\n", 3.1415);

  // ローカルロケールを使用
  std::cout << std::format(loc, "{:L}\n", 1000);
  std::cout << std::format(loc, "{:L}\n", 3.1415);
}
```
```
1,000
3.1415
1.000
3,1415
```

なお、組み込み型のすべてのフォーマットのデフォルトはロケール非依存であり、`std::format()`にロケール引数が与えられている時でも、`L`オプションが指定されていなければロケール依存の出力は行われません。

```cpp
#include <format>

int main() {
  // ドイツ語ロケール
  auto loc = std::locale{"german"};

  std::cout << std::format(loc, "{:L}\n", 3.1415);
  std::cout << std::format(loc, "{}\n", 3.1415);
}
```
```
3,1415
3.1415
```

### type 

最後の*type*オプションはデータの表示の仕方を指定するオプションです。この*type*はフォーマット対象のデータ型とは異なるものです。

*type*オプションは基本的に半角英字1文字で指定します。他のオプションと異なり、*type*オプションは型ごとに可能な指定が異なっていたり意味が微妙に違ったりするので、それぞれの型のグループ毎に個別に見ていくことにします。

### 文字列型のフォーマット

文字列型のフォーマット文字列構文の全体は次のようになります。

```
{ index : fill align width .precision type(s) }
```

*sing*とか*alt*などは文字列型では指定可能ではなく、文字列型の*type*オプションは`s`のみです。

```cpp
#include <format>

using namespace std::literals;

int main() {
  std::cout << std::format("{:s}\n", "string");   // const char*
  std::cout << std::format("{:s}\n", "string"sv); // string_view
  std::cout << std::format("{:s}\n", "string"s);  // string
}
```

ただし、*type*オプションを省略したときでも`s`が指定されているかのように扱われるので、文字列型に対する*type*オプションはほぼ無意味です。しいて言うなら、文字列変数で置換したい置換フィールドに整数型などが間違って指定された場合にコンパイルエラーとすることができます。

```cpp
#include <format>

int main() {
  // ok、どちらも同じ出力結果
  std::format("{:}\n", "string");
  std::format("{:s}\n", "string");

  // ng、数値型にはsが使用できない
  std::format("{:s}\n", 0);
  std::format("{:s}\n", 256);
  std::format("{:s}\n", 1.0);
}
```

ただし、`bool`型では`s`オプションが有効なので、`bool`値が指定されるとエラーになりません。

文字列型における*precision*オプションは、文字列に対してその最大長を指定という意味を持ちます。

```cpp
#include <format>

using namespace std::literals;

int main() {
  auto str = "123456789abcdef"sv;
  std::cout << std::format("{:.1s}\n", str);
  std::cout << std::format("{:.5s}\n", str);
  std::cout << std::format("{:.10s}\n", str);
}
```
```
1
12345
123456789a
```

*width*の指定する値は最小幅であり、*precision*の値は最大幅を指定するものです。そのため、*precision*の値からはみ出る部分の文字は出力されません。なお、ここでの文字幅もターミナルなどに表示した時の文字幅とはほぼ無関係で、*fill/align*の際と同じカウント（特定の文字が幅2、それ以外は幅1として扱われる）で出力幅が決まります。

### 整数型のフォーマット

整数型のフォーマット文字列では次のオプションが使用可能です

```
{ index : fill align sign alt(#) pad(0) width locale(L) type }
```

*type*オプションに指定可能なものは次のものです

|*type*|意味|例|
|---|---|---|
|`d`|10進表示（デフォルト）|`63`|
|`b`|2進表示|`0b1000001`|
|`B`|2進表示（大文字）|`0B1000001`|
|`c`|文字として表示|`?`|
|`o`|8進表示|`077`|
|`x`|16進表示|`0x3f`|
|`X`|16進表示（大文字）|`0X3F`|

ここでの例は`63`という整数値をそれぞれのオプション（`d c`以外は`#`付）でフォーマットした時の出力例を示しています。

`b B o x X`の5つのオプションのデフォルトは基数を示すプリフィックスが付加されません。前述のように、それは`#`（*alt*オプション）と一緒に指定することで出力できます。特に、`B`はプリフィックスの部分にしか影響がありません。

```cpp
#include <format>

int main() {
  int n = 63;
  
  std::cout << std::format( "{:b}\n", n);
  std::cout << std::format( "{:B}\n", n);
  std::cout << std::format( "{:#b}\n", n);
  std::cout << std::format( "{:#B}\n", n);
  std::cout << std::format( "{:o}\n", n);
  std::cout << std::format( "{:#o}\n", n);
  std::cout << std::format( "{:x}\n", n);
  std::cout << std::format( "{:X}\n", n);
  std::cout << std::format( "{:#x}\n", n);
  std::cout << std::format( "{:#X}\n", n);
}
```
```
111111
111111
0b111111
0B111111
77
077
3f
3F
0x3f
0X3F
```

なお、`d c`オプションには代替形式が存在しないので`#`と共に指定するとコンパイルエラーとなります。ただ、`#`単体だとエラーにはならず無視されます。

```cpp
#include <format>

int main() {
  int n = 63;
  
  // 共にコンパイルエラー
  std::cout << std::format( "{:#c}\n", n);
  std::cout << std::format( "{:#d}\n", n);

  // これはok
  std::cout << std::format( "{:#}\n", n);
}
```

`c`オプションは与えられた数値を文字コードの値として扱って対応する文字を表示するオプションで、変換先の文字型にはフォーマット文字列の文字型が使用されます。変換先の文字型で値が表現可能でない（オーバーフローする）場合はコンパイルエラーとなります。

Ascii範囲内のコードはほぼ問題ないと思いますが、同じ値でも使用する文字型及びその文字コードによって出力結果やエラーの有無が異なる場合があります。

```cpp
#include <format>

int main() {
  std::cout  << std::format( "{:c}\n", 63); // charの値として文字化
  std::wcout << std::format(L"{:c}\n", 63); // wchar_tの値として文字化
}
```

`char`の文字コードがAsciiと互換性があり`wchar_t`の文字コードがUTF-16/32である場合（おそらく多くのLinux/Windows環境）、このサンプルコードはどちらも`?`が出力されます。

### 文字型のフォーマット

文字型（`char/wchar_t`）の値は整数値ではなく文字として扱われて出力されます。従って、整数型よりも限定されたオプションが使用可能です。

```
{ index : fill align width locale(L) type }
```

整数型における*type*オプションの`c`の時と同様に、ここでの対象となる文字型というのはフォーマット文字列の文字型のことです。

*type*オプションに指定可能なのは次のものです

|*type*|意味|
|---|---|
|`c`|文字として表示（デフォルト）|
|`d b B o x X`|整数値としてフォーマット|

`d b B o x X`のオプション振る舞いは整数型のものと同様になります。ほぼ整数値の*type*オプションと同じ意味であり、違いはデフォルトの挙動くらいです。

```cpp
#include <format>

int main() {
  const char c = 'A';

  std::cout << std::format( "{:}\n", c);
  std::cout << std::format( "{:c}\n", c);  
}
```
```
A
A
```

デフォルトが`c`なので指定してもしなくても同じなのですが、`c`をあえて指定しておくことで文字型以外をフォーマットしようとした場合にコンパイルエラーとする事ができます。ただし、整数型には無力です。

```cpp
#include <format>

int main() {
  // これはng
  std::format( "{:c}\n", "str");
  std::format( "{:c}\n", true);

  // これはok
  std::format( "{:c}\n", 65);
}
```

この文字型のフォーマットは少し特殊で、*type*に`d b B o x X`を指定すると整数型としてフォーマットされるようになり、使えるオプションも整数型のものと同様になります。

### `bool`型のフォーマット

`bool`型の値もまた、整数型とは区別されてフォーマットされます。使用可能なオプションは文字型と同様です。

```
{ index : fill align width locale(L) type }
```

*type*オプションに指定可能なのは次のものです

|*type*|意味|
|---|---|
|`s`|文字列で表示（デフォルト）|
|`d b B o x X`|整数値としてフォーマット|

`d b B o x X`のオプション振る舞いは整数型のものと同様になります。

```cpp
#include <format>

int main() {
  const bool b = false;

  std::cout << std::format( "{:}\n", b);
  std::cout << std::format( "{:s}\n", b);  
}
```
```
false
false
```

デフォルトが`s`なので指定してもしなくても同じなのですが、`s`をあえて指定しておくことで`bool`型以外をフォーマットしようとした場合にコンパイルエラーとする事ができます。ただし前述のように、`s`の*type*指定は文字列型にも行えるので文字列型に対しては無力です。

```cpp
#include <format>

int main() {
  // これはng
  std::format( "{:s}\n", 0);
  std::format( "{:s}\n", 't');

  // これはok
  std::format( "{:s}\n", "str");
}
```

`bool`型のフォーマットも文字型同様に、*type*に`d b B o x X`を指定すると整数型としてフォーマットされるようになり、使えるオプションも整数型のものと同様になります。

### 浮動小数点数型のフォーマット

浮動小数点数型のフォーマットでは、すべてのオプションが使用できます。

```
{ index : fill align sign alt(#) pad(0) width .precision locale(L) type }
```

*type*オプションに指定可能なのは次のものです

|*type*|意味|例|
|---|---|---|
|`a`|16進表示|`1.921cac083126fp+1`|
|`A`|16進表示（大文字）|`1.921CAC083126FP+1`|
|`e`|指数表示|`3.141500e+00`|
|`E`|指数表示（大文字）|`3.141500E+00`|
|`f F`|固定小数表示|`3.141500`|
|`g`|`f`と`e`のどちらか（デフォルト）|`3.1415`|
|`G`|`F`と`E`のどちらか|`3.1415`|

ここでの例は`3.1415`という値をそれぞれのオプションでフォーマットした時の出力例を示しています。

`E G`オプションの`e g`との違いは基本的に指数表示の時の仮数部と指数部の区切りの`e`が大文字になることですが、`nan, inf`も大文字になります。

`g G`オプションでは、出力文字数が少なくなる方のオプションが自動的に選択されてフォーマットされます。

```cpp
#include <format>

int main() {
  std::cout << std::format( "{:g}\n", 1E5);
  std::cout << std::format( "{:g}\n", 1E6);
}
```
```
100000
1e+06
```

`e`と`f`系のオプションではデフォルト（`g G`）と*precision*の意味が少し異なり、小数点以下の桁数の指定になります。デフォルトおよび`g G`では、*precision*の値は小数点以外の数字部分の桁数の指定になります。

```cpp
#include <format>

int main() {
  double v = 3.14;
  std::cout << std::format( "{:.0e}\n", v);
  std::cout << std::format( "{:.1e}\n", v);
  std::cout << std::format( "{:.3e}\n", v);
  std::cout << std::format( "{:.0f}\n", v);
  std::cout << std::format( "{:.1f}\n", v);
  std::cout << std::format( "{:.3f}\n", v);
}
```
```
3e+00
3.1e+00
3.140e+00
3
3.1
3.140
```

混同しがちですが、*width*は置換後文字列全体の幅の最小値の指定であり、*precision*は浮動小数点数値の精度の指定です。同時に指定した時にもそれぞれそのように振る舞い、必ずしも全体の桁数と小数点以下の桁数の同時指定のようにはなりません。

```cpp
#include <format>

int main() {
  double v = 3.14;
  std::cout << std::format( "|{:5.0f}|\n", v);
  std::cout << std::format( "|{:5.2f}|\n", v);
  std::cout << std::format( "|{:5.4f}|\n", v);
}
```
```
|    3|
| 3.14|
|3.1400|
```

### エスケープ

置換フィールドを構成する2文字`{`と`}`はフォーマット文字列中で特別扱いされており、共に単体で出現してしまうと構文エラーとなります。しかし、フォーマット文字列中で`{`と`}`を使用したいこともあるでしょう。そのために、`{`と`}`にはエスケープシーケンスとして`{{`と`}}`が用意されています。

```cpp
#include <format>

int main() {
  std::cout << std::format( "{{{}, {}, {}, {}}}\n", 1, 2, 3, 4);
  std::cout << std::format( "{{ {:+#x} }}\n", 0xff);
}
```
```
{1, 2, 3, 4}
{ +0xff }
```

この2つ以外の文字はフォーマット文字列中で曖昧にならないので、これら以外はそのまま使用できます。

## `std::formatter`

前節で紹介したのは組み込み型のフォーマット文字列の構文であって、標準ライブラリのほとんどの型やユーザー定義の型などでは使用できません。使用しようとするとコンパイルエラーになります。

```cpp
#include <format>

struct vec3 {
  int elem[3];
};

int main() {
  vec v = {1, 2, 3};
  std::optional<int> opt = 10;

  std::format( "{}", v);    // ng
  std::format( "{}", opt);  // ng
}
```

ユーザー定義の型に対して`std::format()`を使用するためには、`std::format()`にその型のためのフォーマット文字列構文やフォーマット方法を教える必要があるため、デフォルトでは何もできないのです。

どうやってそれを教えてあげるかというと、`std::formatter`というクラステンプレートをフォーマットしたい型に対して特殊化（部分特殊化）して、そのメンバ関数を適切に定義することで行います。

実のところ、前節で紹介した組み込み型のフォーマット文字列構文もこの`std::formatter`の特殊化が組み込み型に対してあらかじめ提供され（ていることが保証され）ており、なおかつその構文やフォーマット方法が規定されているために組み込み型で`std::format()`が使用可能となっています。つまりは、`std::format()`にアダプトする必要があるという観点からは、組み込み型は何ら特別扱いされていません。

`std::formatter<T, CharT>`は`T`に対象の型、`CharT`に文字型（`char/wchar_t`）を取り、次のような宣言になっています。

```cpp
namespace std {
  // std::formatterのシグネチャ
  template<typename T, typename CharT = char>
  struct formatter;
}
```

第2引数の文字型は`std::foramt()`に渡されたフォーマット文字列の文字型に対応していて、`std::foramt()`のフォーマット文字列が`char/wchar_t`文字列の2つしか取らないので、現状は他の文字型を指定しても意味がありません。

この文字型はすなわち`std::foramt()`の出力文字列の文字型でもあるので、`std::formatter<T, CharT>`の特殊化は型`T`の`CharT`文字列に対するフォーマッターを定義します。`CharT`には`char`がデフォルトで指定されているため、`T`に対してだけ特殊化すればとりあえず片方は提供できます。

```cpp
// アダプトしたい型
struct vec3 {
  int elem[3];
};

// formatter特殊化1a、char
template<>
struct std::formatter<vec3, char>;

// formatter特殊化1b、char（文字型省略）
template<>
struct std::formatter<vec3>;

// formatter特殊化2、wchar_t
template<>
struct std::formatter<vec3, wchar_t>;
```

ここでは、この`vec3`型を例に`std::formatter`へのアダプト方法を見ていきます。なお、簡略化のために`char`文字列に対するフォーマッターのみを扱います。

`std::formatter`の特殊化にはデフォルト構築・コピー代入と構築・デストラクト可能・`swap`可能などの性質が要求されます。何かメンバ変数を持つことは可能ですがそれがこれらを損ねないようにする必要があるほか、そのオブジェクトは`std::format()`内部で構築されるのでコンストラクタは定義しない方が良いでしょう。

その上で、`std::formatter`の特殊化はフォーマット文字列のパースとフォーマットのために`parse()`と`format()`の2つのメンバ関数を持っている必要があります。この2つのメンバ関数によって任意の型のためのフォーマット文字列構文とフォーマット方法を定義します。

```cpp
template<>
struct std::formatter<vec3, char> {

  // フォーマット文字列をパースする
  constexpr auto parse(std::format_parse_context& pc);

  // フォーマットを行う
  auto format(const vec3& v, std::format_context& fc);
};
```

`std::format()`内部では、型`T`と文字型`CharT`による特殊化`std::formatter<T, CharT>`のオブジェクト`f`を構築し、まず`f.parse()`によってフォーマット文字列の妥当性検査を行い、パースしたオプションを`f`に保存し、次に`f.format()`に`T`のオブジェクトを渡して呼び出して文字列化を行わせます。この時、それぞれの場合に必要な情報（フォーマット文字列やフォーマット結果出力先など）を渡しているのが`std::format_parse_context`と`std::format_context`です。なお、これは文字型が`char`の場合のもので、`wchar_t`の場合は先頭に`w`が付きます。

`std::format()`では、置換フィールドの*index*オプションを無視すれば置換フィールドとフォーマット対象引数列の対応はその出現順となります。従って、`std::format()`内部からはフォーマット対象引数に対応する置換フィールドを特定する事ができ、型`T`のフォーマッターに対応するフォーマット文字列を正しく渡す事ができます。

`parse()`に渡される`std::format_parse_context`はまさにその情報を保持している範囲（`range`）オブジェクトであり、`T`の引数に対応する1つの置換フィールドを参照しています。`std::format_parse_context`オブジェクトが参照する1つの置換フィールド（`{...}`）の範囲`[begin, end)`の`begin`は`{`もしくは`:`の次の文字、`end`は`}`になります。

*index*オプションは置換フィールドと引数の対応を変えるものであり、これをパースして適切な対応をとるのは`std::format()`の役目です。そのため、`std::format_parse_context`の参照する置換フィールド範囲には*index*オプションとそれに続く`:`は含まれておらず、`:`の次の文字から開始されます。

`std::format_parse_context`の参照範囲と引数対応の様子

![](./img/format_parse_context.png)

`parse()`で処理すべきなのはこの範囲の文字列で、パースを正常に終了した場合は最初の非フォーマット文字の位置、すなわち`}`を指すイテレータを返す必要があります。

まずはとりあえず空のオプション（`{}`）を受け入れるようにします。

```cpp
// vec3のためのパースの実装
// とりあえずオプションなしのみを受理する
constexpr auto parse(std::format_parse_context& pc) {
  // フォーマット文字列範囲を取得
  auto it = pc.begin();
  auto end = pc.end();

  // フォーマット文字列が空の場合
  // it == end or *it == '}' のどちらかである事が保証される
  if (it != end && *it != '}') {
    // どちらでもない場合は空でない
    throw std::format_error{"The vec3 format string has no options."};
  }

  // パース終了地点のイテレータを返す
  return it;
}
```

フォーマット文字列が空であるとき、`pc.begin() == pc.end()`もしくは`*pc.begin() == '}'`のどちらかになることが保証されています（どちらあるいは両方になるかは保証がありません）。なので、フォーマット文字列が空かどうかはこのことをチェックすれば分かります。1つのフォーマット文字列範囲の終端に到達している時に`pc.begin() == pc.end()`とならない実装の場合、エンドイテレータは別のところを指している可能性があるため、`parse()`から返すイテレータには`pc.begin()`から取得して進めたイテレータ（上記例では`it`）を使用するようにした方がよりポータブルになります（実際、clangの実装とMSVCの実装でこの点が異なります）。

コンパイル時フォーマット文字列チェックのために`parse()`は`constexpr`関数である必要がありますが、構文エラーを発生させるには実行時でもコンパイル時でも同様に`throw`式で行えます。この部分をラップしたり工夫することで（コンパイラ毎に調整が必要とはいえ）コンパイル時のフォーマット構文エラーメッセージを見やすくする事ができます。

このフォーマット文字列（とは言っても`{}`のみですが）に対して`vec3`のフォーマットは例えば`{1, 2, 3}`のように出力することにします。`format()`にはその文字列化の実装を記述します。

```cpp
// vec3のためのフォーマットの実装
// vec3{1,2,3}を{1, 2, 3}のように出力
auto format(const vec3& v, std::format_context& fc) {
  // フォーマット結果出力先の出力イテレータ
  auto out = fc.out();

  // outに1文字づつ出力していく
  *out = '{';
  ++out;

  // format_toを用いて組み込み型のフォーマット実装へ委譲する
  out = std::format_to(out, "{:d}, ", v.elem[0]);
  out = std::format_to(out, "{:d}, ", v.elem[1]);
  out = std::format_to(out, "{:d}"  , v.elem[2]);

  *out = '}';
  ++out;

  // 出力完了後のイテレータを返す
  return out;
}
```

`format()`の1つ目の引数には`std::format`に渡されたフォーマット対象の引数への参照が渡されるので、フォーマット対象の値はここから読み出します。2つ目の引数には、フォーマット後の出力先などの情報が渡されており、`fc.out()`によって得られる出力イテレータに対してフォーマット済み文字列を出力することで文字列化を行います。

この`format()`の1つ目の引数は、左辺値を受け取れてそれを変更しないこと、および変換可能な型も受け入れ可能であることが求められます。そのため、`format()`の1つ目の引数型はアダプトする型を`T`とすると、`const T&`もしくは`T`で宣言しておかなければなりません。また、`format()`の出力は、第一引数の値、`fc.lcale()`（使用するロケールオブジェクト）、および`parse()`でパースしたフォーマット文字列以外のものに依存してはなりません。例えばグローバル変数とか外部入力に依存してはなりませんが、フォーマット文字列をパースしてその内容をメンバ変数などに保存した情報は用いても大丈夫です。

自前の型とは言っても、ほとんどの型（クラス）では結局そのメンバには組み込み型が入っていると思います。組み込み型に対しては最初から`std::formatter`が用意されているので`std::format()`等が使用可能なため、フォーマット処理を組み込み型のフォーマットに帰結させてやれば実装をかなり省略できます。

この2つの関数を`std::formatter<vec3, char>`に追加（当然ながら`public`で）してやると、とりあえず最低限のフォーマットが可能になります。

```cpp
#include <format>

struct vec3 {
  int elem[3];
};

int main() {
  vec v = {2, 4, 6};

  std::cout << std::format("{}", v); // ok
}
```
```
{2, 4, 6}
```

ところで、`parse()`の2つ目の引数の型の`std::format_context`は`std::basic_format_context`のエイリアスで、これは2つテンプレートパラメータをとるクラステンプレートです。`std::format_context`はその2つのテンプレートパラメータを埋めたものですが、`parse()`の2つ目の引数は正確には`std::basic_format_context<Out, char>`（`Out`は未規定）の左辺値を受け取れることが求められており、それは必ずしも`std::format_context`と一致しない場合があります。

ここでの例は説明のために`std::format_context`を用いていましたが、ポータビリティのためには`parse()`の第2引数はテンプレートパラメータで受けたほうが良いでしょう。

```cpp
template<>
struct std::formatter<vec3, char> {

  // フォーマット文字列をパースする
  constexpr auto parse(std::format_parse_context& pc);

  // フォーマットを行う
  auto format(const vec3& v, auto& fc);
};
```

以降のサンプルコードはこのようにすることにします。

### オプションを追加する

次に、フォーマット文字列を拡張してオプションを1つ追加してみます。`{:v}`のように指定されたときに、フォーマット後の出力を`(1, 2, 3)`のように変えるようにします。ただし、`{}`の時は先ほどと同じフォーマットをさせたいため、オプションの有無で結果が変わります。すなわちフォーマット文字列パース時にオプションを認識したらそれを保持しておかなければなりません。これは単純にパース結果を`std::formatter`のメンバ変数として保存しておくことで行えます。

まずは`parse()`でそのような構文を受理可能にします。

```cpp
template<>
struct std::formatter<vec3, char> {
  // オプションを保存するメンバ変数
  char opt = ' ';

  // vec3のためのパースの実装
  // {:v}を受理する
  constexpr auto parse(std::format_parse_context& pc) {
    auto it = pc.begin();
    auto end = pc.end();

    // {:v}をチェック
    if (*it == 'v') {
      opt = 'v';

      // vの次'}'へ進める
      ++it;
    }

    // {}が閉じていることをチェック
    if (it != end && *it != '}') {
      throw std::format_error{"invalid format."};
    }

    return it;
  }

  auto format(const vec3& v, auto& fc);
};
```

オプションが指定されているかのチェックは愚直な文字列の検査です。この場合は`v`オプションがあるか無いかだけなので簡単ですが、組み込み型のように複雑なオプションを指定可能としたいのならばフォーマット文字列の文法をきちんと定義する必要があるでしょう。

パースしたオプション指定はメンバ変数`opt`に保存しておきます。

次に、`format()`ではここでパースしたオプションに応じて文字列化処理を変えます。

```cpp
template<>
struct std::formatter<vec3, char> {
  // オプションを保存するメンバ変数
  char opt = ' ';

  constexpr auto parse(std::format_parse_context& pc);

  // フォーマットを行う
  auto format(const vec3& v, auto& fc) const {
    // フォーマット結果出力先の出力イテレータ
    auto out = fc.out();

    // outに1文字づつ出力していく
    if (opt == 'v') {
      *out = '(';
    } else {
      *out = '{';
    }
    ++out;

    // format_toを用いて組み込み型のフォーマット実装へ委譲する
    out = std::format_to(out, "{:d}, ", v.elem[0]);
    out = std::format_to(out, "{:d}, ", v.elem[1]);
    out = std::format_to(out, "{:d}"  , v.elem[2]);

    if (opt == 'v') {
      *out = ')';
    } else {
      *out = '}';
    }
    ++out;

    // 出力完了後のイテレータを返す
    return out;
  }
};
```

パース時に保存しておいたオプション情報（`opt`）によってかっこの種類を変えているだけなので特に難しいところはありません。これによって、`vec3`型のフォーマットにおいて`v`オプションが使用可能となります。

```cpp
#include <format>

int main() {
  vec3 v = {1, 2, 3};
  
  std::cout << std::format("{}\n", v);
  std::cout << std::format("{:v}\n", v);  // ok
}
```
```
{1, 2, 3}
(1, 2, 3)
```

### 既存のフォーマット構文の利用と拡張

`vec3`の要素は`int`固定なので、`int`（整数型）のフォーマットオプションを受けて各要素をそれによってフォーマットするようにしてみます。その際、前項の`v`オプションを指定する場合は一番最後に指定することにします。

```cpp
template<>
struct std::formatter<vec3, char> {
  char opt = ' ';
  // int型のフォーマッターを活用
  std::formatter<int, char> intf;
  
  // vec3のためのパースの実装
  constexpr auto parse(std::format_parse_context& pc) {
    auto it = pc.begin();
    auto end = pc.end();

    // 終端を探す（endは必ずしも置換フィールドの終端ではないため）
    const auto pos = std::string_view{it, end}.find_first_of('}');
    if (pos == std::string_view::npos) {
      // }が閉じてない
      throw std::format_error{"invalid format."};
    }

    // パース対象のオプション文字列全体
    // "{:+v}..."の"+v"
    std::string local_fmt{it, it + pos};

    // vの有無をチェック
    // ローカルフォーマット文字列を構成して'}'で終わるようにする
    if (local_fmt.back() == 'v') {
      opt = 'v';
      local_fmt.back() = '}';
    } else {
      local_fmt.push_back('}');
    }

    // ローカルフォーマット文字列を整数型のフォーマッタにパースさせる
    std::format_parse_context pc2{local_fmt};
    intf.parse(pc2);

    // 終端（最初の'}'）まで進める
    it += pos;

    // {}が閉じていることをチェック
    if (it != end && *it != '}') {
      throw std::format_error{"invalid format."};
    }

    return it;
  }
  
  // vec3のためのフォーマットの実装
  auto format(const vec3& v, auto& fc);
};
```

フォーマット文字列が空であるときは`pc.begin() == pc.end()`もしくは`*pc.begin() == '}'`のどちらかになることからわかるように、`parse()`内でのエンドイテレータ（`pc.end()`）は必ずしも1つの置換フィールドの終端`}`を指していません。規格的にはどうやら未規定であり、むしろ実装としては`std::format()`に渡されたフォーマット文字列の終端を指している場合が多いようです。従って、フォーマット文字列内でのエンドイテレータの位置を仮定した処理は意図通りにならないため、基本的には先頭から順番にパースしていかなければなりません。とはいえ、`parse()`内での先頭イテレータの位置は必ず対応する置換フィールドのオプション文字列先頭なので、そこから1つの置換フィールドの終端を見つけるのは難しくありません。

この例では、オリジナルの`v`オプションはあるとしたらオプションの最後、すなわち`}`の1つ前なので、現在の置換フィールド終端を見つけたら`v`オプションの有無は簡単にチェックできます。`v`オプション以外の部分は整数型のフォーマットをそのまま指定可能にしたいのでその部分は整数型のフォーマッターに委譲したくなります。そのためにこの例では、オプション文字列をローカル`std::string`に切り出して終端を`}`で閉じることで即席フォーマット文字列として整数型のフォーマッタに渡し、その整数型のフォーマッタはメンバ`intf`として保持しておく事でパース結果を保存しています。C++20からは`std::string`が定数式で使用可能となっているため、`std::string`をここで使用しても`parse()`はコンパイル時に実行可能です。

他のフォーマッターの`parse()`を使用する場合に必要となる`std::format_parse_context`は`std::string_view`（`wchar_t`の場合は`std::wstring_view`）をとるコンストラクタが1つだけあるので、文字列を渡して構築する事ができます。`std::format()`内から呼ばれる`std::formatter::parse()`（つまりここで実装している関数）では、フォーマット文字列から構築された`std::format_parse_context`が渡っているわけです。

`format()`では、`parse()`で保存されたオプション情報`opt`と`intf`を活用してフォーマットを行います。とはいえ、`v`オプションのみの時と異なるのは`intf.format()`によって整数型のフォーマッタへ処理を委譲する部分だけです。

```cpp
template<>
struct std::formatter<vec3, char> {
  // オプションを保存するメンバ変数
  char opt = ' ';
  std::formatter<int, char> intf;

  constexpr auto parse(std::format_parse_context& pc);

  // フォーマットを行う
  auto format(const vec3& v, auto& fc) {
    // フォーマット結果出力先の出力イテレータ
    auto out = fc.out();

    // outに1文字づつ出力していく
    if (opt == 'v') {
      *out = '(';
    } else {
      *out = '{';
    }
    ++out;

    // 要素間のデリミタ文字列
    const char delim[3] = ", ";

    // format_parse_contextの出力イテレータを更新
    fc.advance_to(out);
    // 整数型のフォーマッタによってフォーマットしてもらう
    out = intf.format(v.elem[0], fc);
    // デリミタの出力
    out = std::copy_n(delim, 2, out);

    fc.advance_to(out);
    out = intf.format(v.elem[1], fc);
    out = std::copy_n(delim, 2, out);
    
    fc.advance_to(out);
    out = intf.format(v.elem[2], fc);

    if (opt == 'v') {
      *out = ')';
    } else {
      *out = '}';
    }
    ++out;

    // 出力完了後のイテレータを返す
    return out;
  }
};
```

`fc.advance_to()`は渡した出力イテレータによって`std::format_parse_context`の保持する出力イテレータを更新するものです。これによって出力イテレータを現在のものに置き換えることで、整数型のフォーマッタ`intf`に対して現在の`std::format_parse_context`を再利用する事ができます。後は`vec3`の各要素に対して、この`format()`と同じように`intf.format()`を使用すれば`fc`の出力先にフォーマット結果が出力されます。`intf`には`parse()`の実行時にオプションのパース結果が保存されているため、それに従ったフォーマットが行われます。

これによって、`vec3`のフォーマット文字列構文で整数型のフォーマット指定が使用できるようになります。

```cpp
#include <format>

int main() {
  vec3 v = {1, -2, 3};
  
  std::cout << std::format("{:^+#06x}\n", v);   // ok
  std::cout << std::format("{:^+#06xv}\n", v);  // ok
}
```
```
{ +0x1 ,  -0x2 ,  +0x3 }
( +0x1 ,  -0x2 ,  +0x3 )
```

ここでの例のように、独自のフォーマッターを実装するときはいかに組み込み型のフォーマッターを活用するかが肝です。

### フォーマッター実装時のデバッグ

`std::format()`のフォーマット文字列はコンパイル時にその妥当性が検査されます。それには当然`std::formatter::parse()`が使用されているわけですが、そのせいで実装ミスやバグがコンパイルエラーとして報告され、そのエラー原因が分かりづらくなります。その場合、実行時の例外のメッセージの方がわかりやすかったり、デバッガによって処理を追いかけたくなるでしょう。

そのため、独自フォーマッターの実装時には`std::vformat()`を利用する事が推奨されます。`std::vformat()`を使用することでフォーマット文字列のパースを実行時まで遅延する事ができ、`parse()`も`format()`も実行時にしか実行されないので`print`デバッグやデバッガによるデバッグも可能となります。

```cpp
#include <format>

int main() {
  vec3 v = {1, -2, 3};
  
  // 全て実行時にのみ実行される
  // パースエラーなどは例外として報告
  std::vformat("{:^+#06x}\n", std::make_format_args(v));
}
```

### その他の例

ここではより簡易な他の型に対する例を簡単に載せておきます。`std::formatter`特殊化実装の参考になるはずです。

1つ目は`std::optional`です。C++20ではまだこれに対する`std::formatter`は提供されていないため、`std::optional`のフォーマットのためにはフォーマッターを実装しなければなりません。

ここでは、`std::optional<T>`のオブジェクトが空の時は`None`と出力し、値を保持している場合はその値を`T`のフォーマットを利用して出力することにします。

```cpp
// std::optional<T>のためのフォーマッター
template<typename T>
struct std::formatter<std::optional<T>, char> {
  // Tのフォーマッターを利用する
  std::formatter<T, char> tf;

  constexpr auto parse(std::format_parse_context& pc) {
    // パースは丸投げ
    return tf.parse(pc);
  }

  auto format(const std::optional<T>& opt, auto& fc) {
    if (opt) {
      // 有効値の場合はその値を文字列化
      return tf.format(*opt, fc);
    } else {
      // 無効値の場合は"None"と出力
      return std::copy_n("None", 4, fc.out());
    }
  }
};
```

`std::optional<T>`に対して`std::formatter<T, char>`をメンバとして持っておいて、パース時もフォーマット時もそちらに丸投げすることで、実装すべき部分は`std::optional`の有効性をチェックして無効値の時に`None`と出力する部分だけにしています。

```cpp
#include <format>

int main() {
  std::optional<int> opt1{};
  std::optional<int> opt2{-20};
  
  std::cout << std::format("{:+04d}\n", opt1);    
  std::cout << std::format("{:+04d}\n", opt2);
}
```
```
None
-020
```

なお、`std::optional`等の型に対する標準フォーマッターの提供はC++26以降になりそうです・・・

2つ目の例は列挙型の値のフォーマットです。列挙値に対してその列挙値名を出力することにします。

```cpp
// フォーマットしたい列挙型
enum class color { red, green, blue };

// color列挙型に対応する列挙値名
const char* color_names[] = { "red", "green", "blue" };

// color列挙型のためのフォーマッター
template<>
struct std::formatter<color, char> : std::formatter<const char*> {

  // 継承することでparse()は基底のものをそのまま使える

  auto format(color c, format_context& ctx) {
    // 列挙値を整数値にして、対応する名前文字列を引き当て
    auto idx = static_cast<std::underlying_type_t<color>>(c);
    // 基底クラスのformat()へ委譲
    return std::formatter<const char*>::format(color_names[idx], ctx);
  }
};
```

ここでは、文字列（`const char*`）のための`std::formatter`特殊化を公開継承することで、`parse()`をそのまま使用し実装を省略しています。`format()`は引数型が異なるためとりあえず定義する必要があり、そうすると派生クラスで同名関数を定義していることから基底クラスの`format()`が隠蔽されるため、この例のように基底クラスを明示して`format()`を呼び出す必要があります（この構文は静的メンバ関数の呼び出しではありません）。

```cpp
#include <format>

int main() {
  color c1 = color::red;
  color c2 = color::blue;
  
  std::cout << std::format("{:>6s}\n", c1);
  std::cout << std::format("{:>6s}\n", c2);
}
```
```
red
blue
```

2つ目の例のように、組み込み型フォーマッターをコンポジション（メンバとして保持）せずに継承すると必要ない場合に`parse()`の実装を完全に省略できるメリットがありますが、ぱっと見何してるのかわからなくなるのと`format()`での委譲構文が少し煩雑になります。好きなほうで実装してください。

\clearpage

# `<bit>`

`<bit>`では主に、良くあるビット演算のための関数が用意されています。

このヘッダの関数は全て`constexpr`指定されているため定数式で使用可能です。また、`std::bit_cast()`と`std::byteswap()`を除いてすべての関数は符号なし整数型のみを引数に取るように制約されています。

## ビットレベルの再解釈

ビットレベルの再解釈とは、ある型のバイト表現（ビット列）をそのまま別の型のオブジェクトとして扱うことを言います。*type punning*とも呼ばれ、C++では未定義動作を回避してこれを行うことが困難でした（いわゆる*Strict Aliasing rule*に容易に抵触する）。しかし、バイナリI/Oなどその需要は高く、安全にそれを行う機能が求められていたため、C++20からは`std::bit_cast()`という関数によってそれが行えるようになります。

```cpp
namespace std {
  // bit_castの宣言例
  template<typename To, typename From>
  constexpr To bit_cast(const From& from) noexcept;
}
```

`std::bit_cast<To>(from)`のように呼んで、`from`オブジェクトのバイト表現（ビット列）を保ったままたコピーした`To`のオブジェクトを返します。この時、型`From/To`は共に*trivially copyable*である必要があり、両方の型のサイズは一致していなければなりません（違反する場合はコンパイルエラー）。

```cpp
#include <bit>

int main() {
  // double -> uint64_tのバイト表現を保った変換
  constexpr auto r = std::bit_cast<std::uint64_t>(1.0/3.0);
  std::cout << std::format("{:x}", r);
}
```
```
3fd5555555555555
```

定数式で使用する場合は、型`From/To`とそのメンバの型が共用体でもポインタ型でも参照型でもなく、`volatile`修飾もされていない必要がありますが、`std::bit_cast()`はビットレベルの再解釈を定数式で合法的に行う唯一の方法です。

## 2のべき乗整数値に関連する操作

ビット演算における2のべき乗の整数値は、どこか1つのビットだけが1になっているという性質があり、色々便利に使用でき、また使用されます。そのためそれに関連したいくつかの操作が`<bit>`ヘッダでも用意されています。

### `std::has_single_bit()`

`std::has_single_bit()`は入力整数値のビット列が、どこか1つだけ立っている（1になっている）かどうかを調べるものです。これはすなわち、整数値が2のべき乗丁度の値かどうかを判定するものです。

```cpp
#include <bit>

int main() {
  bool b1 = std::has_single_bit(0b00u); // false
  bool b2 = std::has_single_bit(0b01u); // true
  bool b3 = std::has_single_bit(0b10u); // true
  bool b4 = std::has_single_bit(0b11u); // false
}
```

この関数の入力は符号なし整数型のみです（符号付整数型は符号ビットがあるため）。

この関数は提案当初は`ispow2`という名前でした。どういう意味を持つかという観点ではこちらの方が分かりやすいかもしれません。

### 2のべき乗値への丸め

ある整数値を2のべき乗値へ丸める際には、その値より大きいか小さいかで丸め先が2通りあります。それぞれが切り上げ切り捨てに対応しており、切り上げには`std::bit_ceil()`を、切り捨てには`std::bit_floor()`が用意されています。名前からも分かるように、これは2のべき乗値への床関数/天井関数でもあります。

```cpp
#include <bit>

void out(std::unsigned_integral auto n) {
  std::cout << std::format("{:0>4b}\n", n);
}

int main() {
  out(std::bit_ceil( 0b0011u));
  out(std::bit_floor(0b0011u));
  out(std::bit_ceil( 0b1000u));
  out(std::bit_floor(0b1000u));
}
```
```
0100
0010
1000
1000
```

これらの関数は整数値を2のべき乗値へ丸めるので、2のべき乗値の入力に対しては何もしません。

`std::bit_floor()/std::bit_ceil()`のイメージ図

![](./img/bit_floor_ceil.png)

この図では値として1024（$2^{10}$）と2048（$2^{11}$）を使用していますが、これを隣り合う2つの2のべき乗数値に変えても同じようになります。

ビットレベルでは、床関数（`std::bit_floor()`）は元の値の最上位ビット（最も左端で1が立っているビット）だけが立った状態にしている（最上位ビット以下のビットをゼロクリアしている）と見ることもできます。

### 値を表現するために必要なビット幅を求める

ある整数値について、その値を表現するために必要な最小のビット幅を求めるのが`std::bit_width()`です。

```cpp
#include <bit>

int main() {
  std::cout << std::bit_width(0b0001u) << "\n";
  std::cout << std::bit_width(0b0010u) << "\n";
  std::cout << std::bit_width(0b0100u) << "\n";
  std::cout << std::bit_width(0b1000u) << "\n";
}
```
```
1
2
3
4
```

細かい所に目をつむれば、この関数は入力値`n`に対して`floor(log2(n)) + 1`のような計算の結果を返すものです。これはすなわち、`n`の最上位ビットの最下位ビットからの位置（1始まりのインデックス）を求めています。

|整数値|ビット列|`bit_width()`|
|:-:|:-:|:-:|
|0|0000|0|
|1|0001|1|
|2|0010|2|
|3|0011|2|
|4|0100|3|
|5|0101|3|
|6|0110|3|
|7|0111|3|
|8|1000|4|

この関数の提案当初の名前はこのことを反映した`log2p1`という名前でした。

## 循環ビットシフト

通常のビットシフト（`>> <<`）に対する循環ビットシフトとは、シフトした結果範囲外に出たビットをもう片側に回すことで循環させるシフト演算です。循環シフト、巡回シフト、ビットローテーションなどとも呼ばれ、誤り訂正符号や暗号アルゴリズム等の実装においてよく使用されます。

シフトには左シフトと右シフトの2種類があり、循環シフトにおいても`std::rotl()`（左シフト）と`std::rotr()`（右シフト）の2つが用意されています。

```cpp
#include <bit>

void out(std::unsigned_integral auto n) {
  std::cout << std::format("{:0>8b}\n", n);
}

int main() {
  // 右に3ビットシフト
  out(std::rotr(std::uint8_t(0b0000'0101u), 3));
  // 左に3ビットシフト
  out(std::rotl(std::uint8_t(0b1010'0000u), 3));
}
```
```
10100000
00000101
```

`std::rotl()/std::rotr()`は2引数関数で、1つ目の引数に入力整数値、2つ目の引数にシフト量を指定します。この時、シフト量には負の値を指定して逆回転させることもできます。

```cpp
#include <bit>

int main() {
  // 右に-3ビット（左に3ビット）シフト
  out(std::rotr(std::uint8_t(0b1100'0000u), -3));
  // 左に-3ビット（右に3ビット）シフト
  out(std::rotl(std::uint8_t(0b0000'0011u), -3));
}
```
```
00000110
01100000
```

なお、この2つの関数も入力は符号なし整数型オンリーであるため、符号付き整数型において符号ビットも含めた循環シフトを行うことはできません。どうしてもそれを行いたい場合は、`std::bitcast()`によって符号付き整数型から符号なし整数型へバイト表現を保った上で変換し、`std::rotl()/std::rotr()`に通した結果を再度`std::bitcast()`によってバイト表現を保ったまま符号付き整数型へ戻す、のような手順を踏めば行うことができます。

## ビットカウント

ビットカウントとは、あるビット列の立っている（1になっている）ビットの数を数えることです。このための関数が`std::popcount()`で、名前にあるようにこの操作は*popcount*（ポップカウント）と呼ばれます。

```cpp
#include <bit>

int main() {
  std::cout << std::popcount(0b0000'0001u) << "\n";
  std::cout << std::popcount(0b0000'0101u) << "\n";
  std::cout << std::popcount(0b0100'0101u) << "\n";
  std::cout << std::popcount(0b1111'1111u) << "\n";
}
```
```
1
2
3
8
```

これはバイナリ符号間のハミング距離を計算する際によく使用されます。

```cpp
#include <bit>

template<std::unsigned_integral T>
constexpr auto hamming_dist(T c1, T c2) -> T {
  return std::popcount(c1 xor c2);
}

int main() {
  std::cout << hamming_dist(0b0010u, 0b0000u) << "\n";
  std::cout << hamming_dist(0b1111u, 0b0111u) << "\n";
  std::cout << hamming_dist(0b1110u, 0b0111u) << "\n";
  std::cout << hamming_dist(0b1110u, 0b0101u) << "\n";
}
```
```
1
1
2
3
```

## バイトオーダー変換

バイトオーダーとは値のバイト配列をメモリ上に保存する際の順序の事で、エンディアンとも呼ばれます。大別すると、ビッグエンディアンとリトルエンディアンの2つがあります。

エンディアンは値（オブジェクト）をバイト列として読み出すときに問題となるほか、ネットワークバイトオーダーはビッグエンディアンである一方でx86 CPUはリトルエンディアンであることから、ネットワーク越しに送受信したバイト列を読み替える（*punning*する）際に問題となったりします。

リトルエンディアンとビッグエンディアンはバイトスワップという操作によって簡単に相互変換することができますが、これまでのC++にはそれを行うための標準的な方法がありませんでした（一応POSIX規格に`htonl()/ntohl()`という関数はありました）。

C++20からはそのために、`std::byteswap()`が用意されます。この`std::byteswap()`の入力は、符号付きも含めた整数型を使用可能です。

```cpp
#include <bit>

void out(std::integral auto n) {
  std::cout << std::format("{:#X}\n", n);
}

int main() {
  // プログラマが扱うのはビッグエンディアン
  constexpr std::uint32_t n = 0x1234ABCD;

  // リトルエンディアンに変換
  std::uint32_t lite = std::byteswap(n);
  // ビッグエンディアンに変換
  std::uint32_t bige = std::byteswap(lite);

  out(lite);
  out(bige);
}
```
```
0XCDAB3412
0X1234ABCD
```

バイトスワップは、バイト列の真ん中（全体の長さが偶数の場合はバイト境界、奇数の場合は中心バイト）を中心とした点対称のような交換によってバイト列の並べ替えを行います。次の図は`0x1234ABCD`という整数値のビッグエンディアンによるバイト列とリトルエンディアンによるバイト列、そのバイトスワップの様子を描いたものです。

![](./img/byteswap.png)

ややこしい点として、このサンプルコードはx86 CPU等リトルエンディアンのCPUで動かしたとき、想定（コメントの記述）とメモリ上の実際の配置は逆になります。例えば、最初の`n`はメモリ上ではリトルエンディアンで格納されており、それを`byteswap()`するとメモリ上の配置としてはビッグエンディアンに見えるようになりますが、CPUはいつもリトルエンディアンとして読み出すため、その値（のバイト表現）はまた実際のメモリ上の配置とは逆になります。

```cpp
#include <bit>

int main() {
  // 0x1234ABCDをビッグエンディアンで配置
  std::array<std::uint8_t, 4> bytes{0x12, 0x34, 0xAB, 0xCD};

  auto n = std::bit_cast<std::uint32_t>(bytes);
  // メモリ配置は|12|34|AB|CD|
  // 値は0xCDAB3412

  std::uint32_t lite = std::byteswap(n);
  // メモリ配置は|CD|AB|34|12|
  // 値は0x1234ABCD

  std::uint32_t bige = std::byteswap(lite);
  // メモリ配置は|12|34|AB|CD|
  // 値は0xCDAB3412

  out(lite);
  out(bige);
}
```
```
0X1234ABCD
0XCDAB3412
```

すなわち、この関数はメモリ上の配置としてバイトスワップを行うのではなく、読み出した値のバイト表現についてバイトスワップを行います（そしてその上で再度環境のエンディアンによってメモリに格納されます）。とはいえ、リトルエンディアンとビッグエンディアンは相互変換可能であることもあり、多くの場合にこのことを意識する必要はないはずです。

## バイトオーダーの検出

環境のバイトーオーダーはCPUやその設定によって異なる可能性があり、ネットワークデータを扱う場合などにはそれをあらかじめ知っておく必要があります。それは例えば2バイト以上の組み込み型の値がメモリ上にどう配置されるかを調べれば検出可能ではありますが、定数式で実行可能ではないなど問題があります。

`<bit>`ヘッダではその検出のために`std::endian`列挙型が提供されます。

```cpp
// std::endianの定義例
namespace std {
  enum class endian {
    little,
    big,
    native
  };
}
```

`std::endian::little`はリトルエンディアンであることを表す値であり、`std::endian::big`はビッグエンディアンであることを表す値です。`std::endian::native`はその実行環境のバイトオーダーを表す列挙値で、環境のエンディアンに応じて`std::endian::little`か`std::endian::big`のどちらかの値を取ります。

つまり、`std::endian::native`が`std::endian::little`と`std::endian::big`のどちらと等しいかを調べることによって実行環境のバイトオーダーを検出することができます。

```cpp
// 常にビッグエンディアンに変換する関数
auto to_bigendian(std::integral auto n) {
  // std::endian:: を省略
  using enum std::endian;

  if constexpr (native == big) {
    // 環境がビッグエンディアンならそのまま
    return n;
  } else if constexpr (native == little) {
    // 環境がリトルエンディアンなら変換
    return std::byteswap(n);
  } else {
    // 他のエンディアンの場合
    static_assert([]{return false;}(), "not support...");
  }
}
```

なお、おそらく出会うことはないと思われますが、`std::endian::native`が`std::endian::little`と`std::endian::big`のどちらとも等しくない可能性があります。それは、環境のバイトオーダーがミドルエンディアンやPDPエンディアンと呼ばれるエンディアンである場合で、それらはビッグエンディアンとリトルエンディアンを混合させたようなエンディアンになっています。

\clearpage

# `<numbers>`

`<numbers>`ヘッダでは、いくつかの数学定数が事前定義された変数テンプレートで提供されます。

提供されるのは次のものです

|変数名|意味|
|---|---|
|`e`|ネイピア数$e$|
|`log2e`|$e$の2進対数$\log_2{e}$|
|`log10e`|$e$の常用対数$\log_{10}{e}$|
|`pi`|円周率$\pi$|
|`inv_pi`|円周率の逆数$\frac{1}{\pi}$|
|`inv_sqrtp`|円周率の平方根の逆数$\frac{1}{\sqrt \pi}$|
|`ln2`|2の自然対数$\log 2$|
|`ln10`|10の自然対数$\log 10$|
|`sqrt2`|2の平方根$\sqrt 2$|
|`sqrt3`|3の平方根$\sqrt 3$|
|`inv_sqrt3`|3の平方根の逆数$\frac{1}{\sqrt 3}$|
|`egamma`|オイラー定数 $\gamma$|
|`phi`|黄金比$\phi = \frac{1 + \sqrt 5}{2}$|

これらの定数は全て`std::numbers`名前空間にあり、例えば`std::numbers::pi`は次のように定義されています

```cpp
namespace std::numbers {
  // プライマリテンプレート、使用不可
  template <class T>
  inline constexpr T pi_v = ...;

  // 組み込み浮動小数点数型用の特殊化
  template <floating_point T>
  inline constexpr T pi_v<T> = 3.14...;

  // doubleのための変数定義
  inline constexpr double pi = pi_v<double>;
}
```

どの数学定数もこのように定義されているため、使用方法は一貫しています。

```cpp
#include <numbers>

using namespace std::numbers;

int main() {
  // floatの定数の使用
  std::cout << std::format("{:.7f}\n", pi_v<float>);

  // doubleの定数の使用
  std::cout << std::format("{:.15f}\n", pi);
  std::cout << std::format("{:.15f}\n", std::sin(2.0*pi/3.0));
}
```
```
3.1415927
3.141592653589793
0.866025403784439
```

なお、プライマリテンプレートがインスタンス化された場合はコンパイルエラーとなります。その場合、自作や在野の数値型に対しては、これらの変数テンプレートをその型に対して部分特殊化することが許可されています。

## ２の平方根の逆数

$\frac{1}{\sqrt 3}$が`std::numbers::inv_sqrt3`として定義されているのに、$\frac{1}{\sqrt 2}$が定義されていないのは不思議に思われるかもしれません。これは、$\frac{1}{\sqrt 2} = \frac{\sqrt 2}{2}$でありこの計算は丸め誤差を導入せずに（指数部に1足すだけで）行えますが、$\frac{\sqrt 3}{3}$はそうではないためです。

```cpp
#include <numbers>

using namespace std::numbers;

int main() {
  constexpr double inv_sqrt2 = sqrt2 / 2.0;
  
  std::cout << std::format("{:.15f}\n", inv_sqrt2);
  std::cout << std::format("{:.15f}\n", 1.0 / sqrt2);
}
```
```
0.707106781186548
0.707106781186547
```

\clearpage

# `<span>`

`<span>`ヘッダでは、任意のメモリ範囲を所有せずに参照するのに便利な`std::span`クラスを提供します。

C++17まで、任意のメモリ範囲をコピーすることなく参照したいときはポインタと範囲サイズを渡す古式ゆかしいAPIが基本でした。

```cpp
// メモリ範囲をTのシーケンスとして利用する関数
void use_mem_seq(int* ptr, std::size_t len) {
  // メモリ範囲内要素の読み出し
  for (int i = 0; i < len; ++i) {
    int e = ptr[i];
    ...
  }
}
```

あるいはこの2つの引数を1つのクラスにまとめることもできます。しかしどちらにせよ、ポインタを直接扱う必要があり、他のメモリ連続なシーケンス（`std::vector`や配列、`std::string`などなど）からの変換とか各種インターフェース・・・などを考え出すとそこそこ大きな手間となります。ただ、このようなポインタと長さのペアによってメモリ範囲をやり取りするということはかなり頻繁に行われます。

`std::span<T>`はまさにこのための機能であり、ポインタとその長さをラップしたクラスに使いやすいインターフェースを揃えたものです。テンプレートパラメータ`T`には参照先メモリ領域にあるオブジェクトの型を指定します。

```cpp
#include <span>

// メモリ範囲をintのシーケンスとして利用する関数
void use_mem_seq(std::span<int> mem) {
  // メモリ範囲内要素の読み出し
  for (int e : mem) {
    ...
  }
}
```

`std::span`はポインタ1つとそのサイズだけを保持する軽量かつ単純な型であり、*Trivially Copyable*（`memcpy`でコピー可能）であることが規定されています。これらの性質から、基本的には関数に対して値渡しして利用します。`std::span<T>`はメモリ上で連続する`T`のシーケンスを所有せずに参照しており、`std::span`オブジェクトをコピーしても元の`T`のシーケンスはコピーされません。

`std::span`と`std::string_view`はやっていることはほとんど同じですが、文字列はメモリ上で連続して配置されている値以上の意味論を持っているため、`std::string_view`は文字列に特化するために別の型として定義されています。

```cpp
#include <span>

int main() {
  // 7要素の配列（\0含めて7文字）
  const char str[7] = "string";

  // 文字列として参照
  std::string_view sv{str};

  // バイト列として参照
  std::span<const char> sp{str};

  std::cout << sv.size() << "\n"; // 6
  std::cout << sp.size() << "\n"; // 7
}
```

このnull文字の扱いの他にも、インターフェースが文字列に特化（`std::string`互換）されている、文字列は基本的にリードオンリーなどの違いがあります。

## `const`性

`std::span`は参照先を所有していないので、`std::span`に対する`const`指定は意図通りになりません。

```cpp
#include <span>

// 読み込み専用spanのつもりでconstを付加
void use_mem_seq(const std::span<int> mem) {
  for (auto& e : mem) {
    e = 10; // ok、書き換えられる
  }
}
```

`std::span`に対する`const`指定の効果はせいぜい代入ができなくなるくらいのもので、その`const`は参照先の各要素にまで波及しません。別の言い方をすると、`std::span`は参照セマンティクスを持つように設計されています。

読み込み専用の`std::span`を表現するには、`std::span`そのものではなくその要素型に対して`const`を指定します。

```cpp
#include <span>

// 読み込み専用spanを受け取る
void use_mem_seq(std::span<const int> mem) {
  for (auto& e : mem) {
    e = 10; // ng、書き換え不可
  }
}
```

標準ライブラリの他のところ、例えば`std::vector`のように範囲を所有しているクラスにおいては、注意深い実装によってそれ自身に対する`const`が所有している要素にまで及ぶようになっており、このような`const`性の事を深い`const`（*deep const*）と呼びます。これに対して、`std::span`に対する`const`指定のようにそこで切れてしまう`const`性を浅い`const`（*shallow const*）と呼びます。

またこのことは、`std::string_view`(深い`const`を持つ)との違いの一つでもあります。

## 多様な変換

`std::span`は任意のメモリ範囲の参照を手軽にするために、かなり柔軟な変換によって構築することができます。

```cpp
#include <span>

// intのメモリ範囲を参照
void use_mem_seq(std::span<int> mem);

// charのメモリ範囲（単なるバイト列）を参照
void use_char_seq(std::span<const char> mem);

// intのメモリ範囲を読み込み専用で参照
void readonly(std::span<const int> mem);

int main() {
  // std::vector
  std::vector vec = {0, 1, 2, 3};
  use_mem_seq(vec); // ok

  // std::array
  std::array<int, 5> arr = {5, 4, 3, 2, 1};
  use_mem_seq(arr); // ok

  // 生配列
  int rawarr[] = {1, 2, 3};
  use_mem_seq(rawarr);  // ok

  // ポインタとサイズのペア
  std::unique_ptr<int[]> p{new int[4]{}};
  use_mem_seq({p.get(), 4});  // ok
  
  // 文字列
  std::string_view str = "string";
  use_char_seq(str);  // ok

  // 要素型の変換（non const -> const）
  std::span<int> rospan = vec;
  readonly(rospan);   // ok

  // const外しできない
  const std::vector cvec = {0, 1, 2, 3};
  use_mem_seq(cvec);  // ng
  readonly(cvec);     // ok
}
```

このように柔軟な変換を用意している一方で、これを行う変換コンストラクタはかなり複雑な制約によってメモリ連続でないコンテナからの変換や危険な変換（`const`外しやサイズの異なる型への変換など）を許可しないようになっています。

## 静的な要素数

生配列や`std::array`などのようにコンパイル時に要素数が決まっているシーケンスや、あらかじめ入力シーケンスに一定の要素があることが確実に分かっているケースなどでは、`std::span`の要素数をコンパイル時に固定しておきたくなるでしょう。そのために、`std::span<T, N>`のように2つ目のテンプレート引数にサイズを指定して固定長にすることができます。以下、実行時に要素数が決まる`std::span<T>`を動的な`std::span`、固定長の`std::span<T, N>`を静的な`std::span`と呼んで区別します。

```cpp
#include <span>

// int型5個分のメモリを参照するspan
void use_mem_seq(std::span<int, 5> mem);

int main() {
  // std::array
  std::array<int, 5> arr = {5, 4, 3, 2, 1};
  use_mem_seq(arr);     // ok

  // 生配列
  int rawarr[] = {1, 2, 3, 4, 5};
  use_mem_seq(rawarr);  // ok

  // std::vector
  std::vector vec = {0, 1, 2, 3, 4};
  use_mem_seq(vec);     // ng

  std::span<int, 5> sp{vec.data(), vec.size()};
  use_mem_seq(sp);      // ok
}
```

静的な`std::span`では、生配列や`std::array`などのコンパイル時に要素数が決まる範囲からの変換はスムーズに行えますが、`std::vector`のように実行時にサイズが決まる可変長範囲からの変換は直接行うことができす、一度固定長`std::span<T, N>`を明示的に構築する必要があります。ただし、可変長範囲からの固定長`std::span<T, N>`構築時にはその領域に`N`要素が確実に存在していなければなりません。なお、`std::span<T> -> std::span<T, N>`への変換（動的から静的への変換）は制限されてる一方で、`std::span<T, N> -> std::span<T>`への変換（静的から動的への変換）はスムーズに行うことができます。

静的`std::span`には、長さにまつわる実行時計算が削減できる、`std::span`のサイズがポインタ1つ分になる（ことが期待できる）などのメリットがあります。

実装としては、`std::span<T>`と`std::span<T, N>`でクラスの定義は分かれておらず、動的な要素数を指定する場合は`N`に`std::dynamic_extent`という値がデフォルト値として指定されていて、これによってサイズが動的なのか静的なのかを区別しています。

```cpp
// std::spanの宣言例
namespace std {
  inline constexpr size_t dynamic_extent = numeric_limits<size_t>::max();

  template<class T, size_t N = dynamic_extent>
  class span;
}
```

## インターフェース

`std::span`は冒頭の例のようにイテレータインターフェースを備えている（`contiguous_range`コンセプトを満たす）ほか、シーケンスコンテナの備えるインターフェースも利用することができます。なお、`std::span`の備えるインターフェースの時間計算量は全て定数と指定されており、参照する長さによらず一定となります。

```cpp
#include <span>

void span_interface(std::span<int> sp) {
  // 先頭の要素を取得
  int& f = sp.front();

  // 末尾の要素を取得
  int& b = sp.back();

  // 添字アクセス
  // 範囲外参照は未定義動作
  int& s = sp[5];

  // 領域ポインタとサイズの取得
  int* p = sp.data();
  std::size_t len = sp.size();

  // 空かどうか取得（trueで空
  bool e = sp.empty();
}
```

`span`特有のインターフェースの1つに、参照領域のバイト単位の長さを求める`.size_bytes()`があります。

```cpp
#include <span>

void span_interface(std::span<std::int32_t, 4> sp) {
  // 要素数の取得
  std::size_t len = sp.size();
  // 参照先領域サイズの取得
  std::size_t byte_len = sp.size_bytes();

  std::cout << len << "\n";       // 4
  std::cout << byte_len << "\n";  // 16
}
```

これは例えば、`std::span`の参照する領域への、もしくはそこからの`memcpy`時に長さ計算を省略しつつ間違わないようにすることができます。

```cpp
#include <span>

// src -> dstへ全要素分コピーしたい
void span_memcpy(std::span<std::int32_t, 4> dst, std::span<const float, 4> src) {
  // やりがちな間違い（4バイトコピーされる）
  std::memcpy(dst.data(), src.data(), src.size());

  // 正しくバイト単位の長さを指定（16バイトコピーされる）
  std::memcpy(dst.data(), src.data(), src.size_bytes());
}
```

`span`からその部分範囲を取得するために、`.first()`、`.last()`、`.subspan()`の3つの関数が用意されています。

```cpp
#include <span>

int main() {
  std::array<int, 10> arr = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
  std::span<int, 10> sp{arr};
  
  // 先頭からの部分spanを取得する
  // 例 : 先頭から4つの要素を参照するspanを取得
  std::span<int, 4> s1 = sp.first<4>();
  std::span<int> s2 = sp.first(4);
  // {0, 1, 2, 3}

  // 後ろからの部分spanを取得する
  // 例 : 後ろからから4つの要素を参照するspanを取得
  std::span<int, 4> s3 = sp.last<4>();
  std::span<int> s4 = sp.last(4);
  // {6, 7, 8, 9}

  // 任意の位置からの部分spanを取得する
  // 例 : 5番目の位置から4つの要素を参照するspanを取得
  std::span<int, 4> s5 = sp.subspan<5, 4>();
  std::span<int> s6 = sp.subspan(5, 4);
  // {5, 6, 7, 8}
}
```

`.first()`は元の`span`の先頭から指定された長さの範囲を、`.last()`は元の`span`の末尾から指定された分戻った長さの範囲を取得します。`.subspan()`には先頭からのオフセットと長さの2つの引数を渡して、先頭からオフセット値分進んだところから指定された長さの範囲を取得します。

3つの関数にはどれも、引数を非型テンプレートパラメータで受け取るものと関数引数で受け取るものの2種類が用意されています。これらの結果は全て`std::span`となり、非型テンプレートパラメータで要素数とオフセットを指定すると静的`span`が、関数引数で要素数とオフセットを指定すると動的`span`が得られます。上記の例では、静的と動的インターフェース2つの使用例それぞれで同じ部分範囲を取得しています。

なお当然ながら、これら部分範囲を取得するときには元々の範囲からはみ出た範囲を取得しないように注意する必要があります。

### バイト列へのキャスト

前述のように、`std::span`は従来のポインタとサイズを取るインターフェースを置き換えるためのものです。そういうインターフェースが使われていたところでよく見られるのは、バイト配列を受け取ってその配列からデータを読み出す、あるいはその配列へデータを書き込む処理です。そのような処理には例えば、通信やI/O、シリアライズなどがあり、`std::span`はバイト配列の参照として使用される頻度が高くなると予想されます。そのため、`std::span<T>`を`std::span<std::byte>`に変換するための関数、`std::as_bytes()`と`std::as_writable_bytes()`が用意されています。

```cpp
#include <span>

// 何かI/Oへの書き込みを行う関数
void write_io(std::span<const std::byte> data);

// 何かI/Oからの読み込みを行う関数
void read_io(std::span<std::byte> buffer);


int main() {
  // 送信
  std::array<int, 5> data = {1, 2, 3, 4, 5};
  write_io(std::as_bytes(std::span{data}));

  // 受信
  std::array<float, 4> response{};
  read_io(std::as_writable_bytes(std::span{response}));
}
```

どちらの関数も入力には任意の`std::span<T, N>`を取りますが、`std::as_bytes()`は読み込み専用バイト列としての`std::span<const std::byte>`を返し、`std::as_writable_bytes()`は書き込み可能なバイト列としての`std::span<std::byte>`を返します。

これらの関数に動的な`std::span<T>`が入力された場合は返される`span`も動的なものになり、静的な`std::span<T, N>`が入力された場合は返される`span`も静的なものになります。どちらの場合でも、得られた`span`の長さは入力の要素数を`N`として`sizeof(T) * N`となります。

```cpp
#include <span>

void write_io(std::span<const std::byte> data);
void read_io(std::span<std::byte> buffer);

struct send_data {
  std::int32_t n;
  std::int32_t m;
  std::int64_t l;
  const char s[4];
};

struct recieve_data {
  std::uint32_t n;
  float f;
  std::uint16_t m;
};

// 構造体をバイト列へシリアライズする例
// このようなことを行う場合、構造体のTrivially Copyable性に注意！
int main() {
  send_data data{ .n = 10, .m = 11, .l = 12, .s = "tes"};

  write_io(std::as_bytes(std::span<send_data, 1>(&data, 1)));

  recieve_data res{};

  read_io(std::as_writable_bytes(std::span<recieve_data, 1>(&res, 1)));
}
```

\clearpage
# `<source_location>`

`<source_location>`ヘッダでは、従来`__LINE__`や`__FILE__`などによって取得していたソースコード上位置情報を一括で取得する手段を提供するとともに、それをまとめて格納し持ち運ぶことができるようにする構造体`std::source_location`が提供されます。

まず、ソースコード位置情報を取得するには`std::source_location::current()`静的メンバ関数を使用します。

```cpp
#include <source_location>

void f() {
  // この場所のソースコード情報を取得
  auto sl = std::source_location::current();

  std::cout << "\n--- in f() ---\n\n";

  std::cout << sl.line() << "\n"            // 行番号
            << sl.column() << "\n"          // 列番号
            << sl.file_name() << "\n"       // ファイル名
            << sl.function_name() << "\n";  // 関数名
}

int main() {
  auto sl = std::source_location::current();

  std::cout << sl.line() << "\n"
            << sl.column() << "\n"
            << sl.file_name() << "\n"
            << sl.function_name() << "\n";
  
  f();
}
```

`std::source_location::current()`は`consteval`指定されているため、この関数の実行は必ずコンパイル時に完了します。そして、その戻り値として`current()`が呼ばれた場所のソースコード情報を保持した`std::source_location`オブジェクトが得られます。

このプログラムの出力は、例えば次のようになります（Wandbox GCCでの実行結果）。

```
17
42
prog.cc
int main()

--- in f() ---

6
42
prog.cc
void f()
```

`std::source_location`は4つのメンバ関数を持っており、`.line()`は行番号、`.column()`は列番号、`.file_name()`はそのソースファイル名、`.function_name()`は囲む最も内側の関数名、をそれぞれ取得します。`current()`がヘッダファイル内コードで呼び出されている場合、ヘッダファイル内におけるソースコード位置が取得され、`.file_name()`はヘッダファイル名を返します。つまり、`std::source_location`で得られるソースコード位置は`#include`の影響を受けません。

なお、この例の`.column()`は`std::source_location::current()`の`()`の位置が取得されていますが、この値は処理系定義とされているのでコンパイラや環境によって異なる可能性があります。

`std::source_location`オブジェクトはコピーやムーブを自由に行うことができるため、ある場所で取得した情報を別の場所に運ぶことが容易にできます。これは特に、クラスのメンバとして保持するときに便利です。

```cpp
#include <source_location>

class S {
  std::source_location m_loc;

public:

  S(std::source_location loc) : m_loc(loc) {}

  auto get_location() {
    return m_loc;
  }
};

void f(const S& s) const {

  auto sl = s.get_location();

  std::cout << sl.line() << "\n"
            << sl.column() << "\n"
            << sl.file_name() << "\n"
            << sl.function_name() << "\n";
}

int main() {
  S s{std::source_location::current()}; // ok

  f(s); // ok
}
```

この出力は例えば次のようになります

```
27
36
prog.cc
int main()
```

`std::source_location`オブジェクトは軽量でコピーが効率的なオブジェクトであると規定されているため、基本的にはコピーして持ち運ぶことができます。

## デフォルト引数での利用

実際に`std::source_location`を使ってみると、一々`std::source_location::current()`とするのは構文的にかなり重いことに気づくでしょう。また、この取得構文はどこでも同一であるため、`std::source_location`が必要となるところでは自動で取得してほしくもなります。このニーズを満たしてくれるのが関数のデフォルト引数で取得しておくという書き方です。

```cpp
#include <source_location>

// ログ出力関数
void log(std::string_view message, 
         std::source_location sl = std::source_location::current())
{
  std::cout << std::format("{:s}:{:d}:{:d} in {:s} : {:s}", sl.file_name(), sl.line(), sl.column(), sl.function_name(), message);
}

int main() {
  log("test");
}
```

出力例（godbolt MSVC 2019、一部改変）

```
example.cpp:13:3 in main : test
```

このように、デフォルト引数で使用した時でも書かれた場所ではなく呼ばれた場所の位置情報を取得してくれるため、`std::source_location::current()`をほぼ省略しながら利用することができます。そのため、`std::source_location`はもっぱらデフォルト引数で使用されるでしょう。

なお、関数のデフォルト引数は呼び出し時に省略可能という性質から必ず引数列の後ろに来なくてはなりません。この制約のため、可変長テンプレートではこの方法を取れないという問題があります。一応将来のC++に向けて可変長テンプレートの推論を調整してこの問題に対処しようとする動きはありますが、C++26以降になりそうです。

\clearpage
# `<coroutine>`

`<coroutine>`ヘッダは、C++20で言語機能として追加されたコルーチンを利用してコルーチンアプリケーションを実装するために必要な最小の機能を提供するものです。`generator`や`task`（`lazy`）のようなコルーチンアプリケーションはまだここにはなく、C++23以降に順次追加される予定です。

ここでは、型`T`の値のシーケンスを生成するコルーチンアプリケーション、`generator<T>`を作成しながら、これらの基本機能をどのように使用していくのかを見ていきます。

なお、この章の内容はコルーチンの言語仕様へのある程度の理解を前提として書かれています。『C++20 コア言語機能』本をお手元に用意するか、その他C++コルーチン解説を頭に入れてからご覧いただくと理解が捗るかと思います。

```cpp
// これを作る
template<std::movable T>
class generator {
  ...

  auto move_next() -> std::optional<T>;
};

// こうして
generator<int> make_range(int first, int last) {
  for (int i = first; i < last; ++i) {
    co_yield i;
  }
}

int main() {
  // こう使える
  auto coro = make_range(0, 10);

  auto opt = coro.move_next();
  std::cout << *opt << "\n";  // 0

  opt = coro.move_next();
  std::cout << *opt << "\n";  // 1

  opt = coro.move_next();
  std::cout << *opt << "\n";  // 2

  ...

  opt = coro.move_next();
  std::cout << *opt << "\n";  // 9

  opt = coro.move_next();
  
  std::cout << std::boolalpha << bool(opt); // false
}
```


本来はC++的なイテレータインターフェースを実装すべきですが、そちらはそちらで厳密にやりだすと複雑なので、今回はこの`move_next()`のような簡易なインターフェースによって値を取得することにします。

## コルーチントレイトとコルーチンハンドル

コルーチン制御において、コルーチン側での制御を担当するプロミス型は`std::coroutine_traits`によって取得されます。コルーチンの戻り値型を`R`、コルーチン引数型を`Args...`とすると、`std::coroutine_traits<R, Args...>::promise_type`から取得されます。デフォルトの`std::coroutine_traits`（プライマリテンプレート）は`R::promise_type`からそれを取得し、`R`と`Args`についてさらなるカスタマイズが必要となる場合は、`std::coroutine_traits<R, Args...>`を部分特殊化して`::promise_type`を提供することもできます。

```cpp
#include <coroutine>

template<std::movable T>
class generator {

  // promise型
  struct generator_promise {
    ...
  };

  ...

public:
  // ::promise_typeとして公開しておく
  // 今回のgenerator型ではこちらを採用
  using promise_type = generator_promise;
};

// もしくは、部分特殊化によって提供してもいい
template<typename T>
struct std::coroutine_traits<generator<T>, T, T> {
  using promise_type = ...;
};
```

コルーチン制御において、コルーチン呼び出し側で制御を担当するのがコルーチンハンドルで、それは`std::coroutine_handle`というクラスとして用意されています。とはいえ、コルーチンアプリケーションの利用においてはこのコルーチンハンドルを直接扱うことはなく、コルーチンハンドルはコルーチンの戻り値型`R`内部に隠蔽され、コルーチンハンドルは`R`の操作経由で利用されます。

```cpp
namespace std {
  template<class Promise = void>
  struct coroutine_handle;

  template<>
  struct coroutine_handle<void> {
    ...
  };

  template<class Promise>
  struct coroutine_handle {
    ...
  };
}
```

`std::coroutine_handle`は通常テンプレートパラメータにプロミス型を取りますが、場合によってはこの型依存が問題となることもあるので、`std::coroutine_handle<>`として型消去して使用することができます。`std::coroutine_handle<Promise>`から`std::coroutine_handle<>`の変換には暗黙変換が提供されます。

```cpp
#include <coroutine>

template<std::movable T>
class generator {

  struct generator_promise {
    ...
  };

  // コルーチンハンドルをメンバとして保持しておく
  // テンプレートパラメータにはpromise型を指定する
  std::coroutine_handle<generator_promise> m_hcoro;

  ...
};
```

コルーチンハンドルのメンバ関数にはコルーチンの再開と終了やその状態取得のためのものが用意されています。これを利用して呼び出し側からのコルーチン制御を行います。

### コルーチンハンドルとプロミスオブジェクトの初期化

この`generator`とコルーチンハンドル（の実体）、およびプロミス型のオブジェクトはコルーチン内部で生成されています。この生成というのは、コルーチンが言語仕様に従って通常の関数の様に書き換えられた結果として挿入されたコードによって行われています。

そこではまず、`std::coroutine_traits`によって取得されたプロミス型をもちいて、そのオブジェクトがコルーチン引数もしくはデフォルト構築よって構築されます。

次に、プロミス型の`.get_return_object()`によってコルーチンの戻り値型オブジェクト（ここでは`generator`オブジェクト）が取得されており、`.get_return_object()`はプロミス型を定義する際にカスタマイズすることができます。

```cpp
#include <coroutine>

template<std::movable T>
class generator {

  struct generator_promise {

    // コルーチン戻り値取得をカスタマイズする
    auto get_return_object() {
      // コルーチン戻り値（generatorオブジェクト）の生成
      return generator{...};
    }

    ...
  };

  ...
};
```

`std::coroutine_handle`のコンストラクタは無効なコルーチンを生成するためのものしか用意されておらず、有効なコルーチンハンドルは`std::coroutine_handle::from_promise()`によって、プロミス型オブジェクトから構築する必要があります。

`.get_return_object()`内ではプロミスオブジェクトは`*this`によって取得でき、`generator`型ではプロミスオブジェクトもしくはコルーチンハンドルから構築することによってコルーチンハンドルを受け取ります。

```cpp
#include <coroutine>

template<std::movable T>
class generator {
  // 前方宣言
  struct generator_promise;

  // 短縮のためのエイリアス
  using handle = std::coroutine_handle<generator_promise>;

  // プロミス型定義
  struct generator_promise {

    auto get_return_object() {
      // コルーチンハンドルからgeneratorオブジェクトを生成
      return generator{handle::from_promise(*this)};
    }

    ...
  };

  // コルーチンハンドル
  handle m_hcoro;

  // コルーチンハンドルを受け取るコンストラクタ
  explicit generator(handle hcoro)
    : m_hcoro(hcoro) {}

  ...
};
```

`std::coroutine_handle`は無効なコルーチンを生成するためにデフォルトコンストラクタおよび`nullptr`を取るコンストラクタを備えているため、`{}`とか`0`とかから変換して構築することができてしまい、それらから`generator`への変換すら行うことができます。コルーチンハンドルを受け取るコンストラクタは`explicit`にしたうえでプライベートで定義しておくとそのようなバグを踏みづらくなるでしょう。プライベートで定義したときでも、プライベートの内部クラス（ここでは`generator_promise`型）からは参照することができます。

また、ここでは不要なのでしていませんが、プロミスオブジェクトはコンストラクタでコルーチン引数を受け取るようにすることもでき、コルーチン引数をプロミスオブジェクト内に保存しておくことで`.get_return_object()`内でコルーチン引数をコルーチン戻り値型まで伝播させることができます。

### コルーチンの終了

コルーチンは通常の関数とは異なり一回の呼び出しで終了せずに、複数回の呼び出しにわたって状態を保持しています（これをコルーチンステートと呼びます）。コルーチンステートにはコルーチン引数やプロミスオブジェクトやコルーチンのローカル変数が保存されており、これらを適切に開放するにはコルーチンを終了させる必要があります。

コルーチンの終了は、コルーチンの実行が関数終端もしくは`co_return`に到達していて、最終サスペンドポイントで中断していなければ自動で終了しています。

どこかで中断中のコルーチンを終了させるのはコルーチン呼び出し側の責任であり、それはコルーチンハンドルの`.destroy()`によって行うことができます。コルーチンハンドルはコルーチン戻り値型`R`（ここでは`generator`）から操作されるため、通常は`R`のデストラクタで呼び出すようにしておきます。

```cpp
#include <coroutine>

template<std::movable T>
class generator {

  ...

  // コルーチンハンドル
  handle m_hcoro;

  ...

public:

  ...

  ~generator() {
    // コルーチンが有効ならば（終了していなければ）
    if (m_hcoro) {
      // コルーチンを終了させる
      m_hcoro.destroy();
    }
  }
};
```

コルーチンが有効かどうか、つまりコルーチンがまだ終了していない（どこかで中断中）かどうかはコルーチンハンドルの`bool`変換演算子から取得でき、`true`を返したときにまだコルーチンは終了していないことが分かります。従って、この例のようにコルーチンハンドルを`bool`変換して`true`が得られたら`.destroy()`する、のような手順でコルーチンの終了を行います。

コルーチン戻り値型の実装においては、そのデストラクタでこのような典型コードがよく見られるでしょう。また、他の関数からこのコードを呼び出すことでコルーチンを明示的に終了させる実装を行うこともできます。

### コルーチンで生成された値の取得

コルーチン内部で値の生成（`co_yield value;`）が行われると、その引数`value`はプロミス型のメンバ関数`.yield_value()`に渡されます。コルーチンアプリケーション作成者は、この関数によってコルーチン内部からの値の受け取りと呼び出し側への値の返却処理を自由に記述することができます。

```cpp
#include <coroutine>

template<std::movable T>
class generator {

  struct generator_promise {
    // 生成された値を保存
    T value;

    // co_yield時に呼ばれる関数
    auto yield_value(T v) {
      value = std::move(v);
      return ...;
    }

    ...
  };

  // コルーチンハンドル
  handle m_hcoro;

  ...
};

```

この`.yield_value()`は戻り値型として`awaitable`型と呼ばれる型のオブジェクトを返す必要があり、この戻り値型によって`co_yield`における中断を制御します。これについては後述します。

このように、コルーチンが生成した（`co_yield`した）値は（通常）プロミス型のオブジェクトに保存されており、コールチンハンドルからは`.promise()`によってそのプロミスオブジェクトへの参照を取得することができます。

```cpp
#include <coroutine>

template<std::movable T>
class generator {

  struct generator_promise {
    // 生成された値を保存
    T value;
    ...
  };

  // コルーチンハンドル
  handle m_hcoro;

  ...

public:

  // 値の取得をする関数
  auto move_next() -> std::optional<T> {
    
    ...

    // promiseオブジェクトの参照を取得
    auto& promise = m_hcoro.promise();
    // yieldされた値を取得
    return {std::move(promise.value)};
  }
};
```

### コルーチンの再開

コルーチンにおける値の生成ではその都度中断している可能性があり、その場合コルーチンを再開させてコルーチンの実行を進める必要があります。その際、コルーチンの再開前後ではコルーチンの状態をチェックしなければなりません。まずそもそもコルーチンが有効であるか（終了していないか）をチェックし、次に再開に伴って最終サスペンドポイントに到達していないか、をチェックします。

コルーチンが最終サスペンドポイントに到達しているかどうかは`.done()`によってチェックでき、`true`を返したら最終サスペンドポイントに到達していることを表します。コルーチンの再開は`.resume()`によって行い、この呼び出しによってコルーチンは中断地点から再開し、次の中断地点（`co_yield/co_await/co_return`、最終サスペンドポイント）まで処理が進行します。

```cpp
#include <coroutine>

template<std::movable T>
class generator {

  struct generator_promise {
    // 生成された値を保存
    T value;
    ...
  };

  // コルーチンハンドル
  handle m_hcoro;

  ...

public:

  // 値を取得する関数
  auto move_next() -> std::optional<T> {
    // コルーチン有効性をチェック
    if (!m_hcoro) return {};

    // コルーチンを再開
    // 次のco_yieldまで進む
    m_hcoro.resume();

    // 最終サスペンドポイントに到達していないことをチェック
    if (m_hcoro.done()) return {};

    // promiseオブジェクトの参照を取得
    auto& promise = m_hcoro.promise();
    // yieldされた値を取得
    return {std::move(promise.value)};
  }
};
```

今回の`generator`は初期サスペンドポイントで中断するようにしている（この実装はのちほど）ため、初回の`resume()`によって最初の`co_yield`まで進行します。その後最後の`co_yield`で中断されている状態で再開されると最終サスペンドポイントに到達し値を返さず中断します（この実装も後程）。そして、`.move_next()`は値が無い（範囲終端に到達した）場合は無効値を返してそれを通知します。

このため、この例の`.move_next()`においては値の取得前にコルーチン再開を行う必要があり、再開後は最終サスペンドポイントに到達していないことをチェックする必要があります。

このあたりの実装はどういうコルーチンアプリケーションを作るのかによって変化するので、都度きちんと考えなければなりません。最終サスペンドポイント到達やコルーチン終了チェックを怠れば、無限ループや未定義動作に繋がるため慎重な検討が必要です。

## `awaitable`型

コルーチン内部で`co_await`や`co_yield`を呼び出すことでコルーチンを中断して呼び出し側に値を返しつつ制御を戻すことができますが、この際の振る舞いをカスタマイズすることもできます。

コルーチン内部で中断を行う所では、必ずプロミス型のメンバ関数を介したカスタマイズポイントが用意されており、そのメンバ関数が返す型によって中断するかどうか及び中断時/再開時の振る舞いを制御することができます。例えば、先ほど`co_yield`で値を受け取るときに使った`.yield_value()`メンバ関数がその一例です。それらの関数から返され、中断時動作のカスタマイズを担う型のことを総称して`awaitable`型と呼びます。

`<coroutine>`ヘッダでは、中断/再開時に何もせず中断するかしないかのみを制御するとても基本的な2つの`awaitable`型が提供されます。

```cpp
namespace std {
  // 常に中断しないawaitable
  struct suspend_never {
    // コルーチンを中断するかしないかを指定する（trueを返すと中断する）
    constexpr bool await_ready() const noexcept { return true; }
    // 中断時（中断直前）に行う処理を指定
    constexpr void await_suspend(coroutine_handle<>) const noexcept {}
    // 再開時（再開直後）に行う処理を指定
    constexpr void await_resume() const noexcept {}
  };

  // 常に中断するawaitable
  struct suspend_always {
    constexpr bool await_ready() const noexcept { return false; }
    constexpr void await_suspend(coroutine_handle<>) const noexcept {}
    constexpr void await_resume() const noexcept {}
  };
}
```

`suspend_never`は中断可能な地点で常に中断することを指示し、`suspend_always`は中断可能な地点で常に中断しないことを指示します。どちらのクラスも、中断/再開時には何もしません。

`awaitable`型はこの3つのメンバ関数を備えた型として定義することで自作することができます。それぞれの関数は次の役割を持ちます

- `bool await_ready()` : コルーチンを中断するかどうかを`bool`値で指示する
    - `true`を返すと中断させ、`false`を返すと中断させない
- `auto await_suspend()` : コルーチンが中断する直前に、任意の処理を実行させる
    - 引数にコルーチンハンドルを取る
- `auto await_resume()` : コルーチンが再開した直後に、任意の処理を実行させる

`awaitable`を自作する場合、この3つのメンバ関数がこのシグネチャで呼び出し可能であるようなクラスとして定義しておく必要があります。その際、構築のされ方や関数の内部は自由にカスタマイズすることができます。

### Await式

`awaitable`型のオブジェクトはAwait式と呼ばれる式によって評価され、コルーチンの中断制御を行います。Await式は最終的には`co_await o`の形の式として実行され、この一行の式内部でコルーチンを中断し、`awaitable`の3つのメンバ関数によってそのカスタマイズを行います。

`co_await`演算子の直接使用をはじめとして、コルーチン内部ではこのAwait式を呼び出す方法（経路）がいくつか存在しています。それによって、Await式の引数である`o`の取得方法が少し異なります。ここでは、`o`が取得された後、Await式のメインの部分がどのように実行されるかを見ていきます。ここでの`o`は、とりあえず先ほどの2つの`awaitable`型のどちらかのオブジェクトと思っておくと良いかもしれません。

まず、`o`に対して使用可能な`co_await`演算子（`operator co_await`）が探索され、見つかった場合はその演算子を`o`に対して適用しその結果値を`e`として取得します。もし`co_await`演算子が見つからない場合、`e`として`o`をそのまま使用します。`co_await`演算子は単項演算子の一種として扱われていて、メンバのもの（`o.operator co_await()`）と非メンバのもの（`operator co_await(o)`）の2種類が考慮されます。

次に、`e.await_ready()`の戻り値を`bool`値`ready`として取得します。`ready`が`false`の時にコルーチンは中断状態となります。ここでの中断状態とは論理的な状態の変化であり、制御（実行地点）はまだコルーチン内です。

コルーチンが中断状態になった後、そのコルーチンのコルーチンハンドル`h`を用いて`e.await_suspend(h)`を実行し、その戻り値があれば`suspend`として取得します。`suspend`が取得可能である場合、その型は`bool`もしくはコルーチンハンドル`std::coroutine_handle<Z>`のどちらかである必要があります。そして、`suspend`の型によって次のどちらかの処理を実行します

- `suspend`が`bool`の時
    - `suspend == true`ならばコルーチンを中断する
    - `suspend == false`ならばコルーチンを再開する
- `suspend`が`std::coroutine_handle<Z>`の時
    - `suspend.resume()`を実行する
      - 別のコルーチンの再開（対称コルーチン）
- `suspend`がない（`e.await_suspend(h)`の戻り値型が`void`の）時
    - 何もしない

`e.await_suspend(h)`の実行と上記処理が完了した後でもコルーチンが中断状態であるならば、コルーチンは呼び出し元に制御を返し、ここで名実ともにコルーチンは中断されます。なおこの時、コルーチンの実行中のスコープはコルーチンステート内に維持されています。

`ready`が`true`の時は`e.await_suspend(h)`は実行されず、コルーチンは中断しません。その場合、あるいはコルーチンが再開された時、`e.await_resume()`を実行し、戻り値があればそれを返してAwait式の実行は完了します。すなわち、`e.await_resume()`の戻り値型が`void`ではない時、その戻り値がAwait式の実行結果となります。

先ほどの2つの`awaitable`型では、`.await_suspend(), .await_resume()`の2つは何もせず戻り値も返さないためAwait式においても何もせず、`.await_ready()`が`true/false`を返すことで中断するしないだけを制御しています。また、使用可能な`co_await`演算子も定義されていないため、これらの型のオブジェクトは直接Await式の引数となります。

### 初期/最終サスペンドポイント

コード上からは直接観測できないAwait式の実行点が初期サスペンドポイントと最終サスペンドポイントの2つです。この2地点はコルーチン本体の実行前後で、それぞれ1度づつ実行されています。

コルーチンで使用されているプロミス型オブジェクトを`p`とすると、初期サスペンドポイントでは`co_await p.initial_suspend()`の呼び出し、最終サスペンドポイントでは`co_await p.final_suspend()`の呼び出しによってAwait式が実行されています。Await式の引数`o`はプロミス型のメンバ関数`.initial_suspend()`および`.final_suspend()`の戻り値として取得され、`.initial_suspend()`、`.final_suspend()`の2つの関数はプロミス型実装を通して自由にカスタマイズできます。

この2地点でのAwait式は、コルーチン呼び出し直後と終了時にコルーチンを中断するかどうかを制御することが主な目的です。

今回の`generator`型では値の取りこぼしを防ぐために最初の`co_yield`の前で一度中断する必要があり、それを初期サスペンドポイントで行うことにします。また、最後に生成された値を取り出す必要性から最終サスペンドポイントでも同様に中断しておく必要があります。これは、最終サスペンドポイントで中断しない場合はコルーチン終了と共にプロミスオブジェクトを含めたコルーチンステートが破棄されるため、コルーチン戻り値型（`generator`）からのほとんどの操作が未定義動作となることを回避するためです。

```cpp
#include <coroutine>

template<std::movable T>
class generator {

  // promise型
  struct generator_promise {
    
    // 初期サスペンドポイントのカスタマイズ
    auto initial_suspend() {
      return std::suspend_always{}; // 常に中断する
    }

    // 最終サスペンドポイントのカスタマイズ
    auto final_suspend() noexcept {
      return std::suspend_always{}; // 常に中断する
    }

    ...

  };

  ...
};
```

このように、単純に中断するしないの制御には標準で用意されている2つの`awaitable`型を使用できます。この場合中断するしないと中断前後に何もしないということがより明確になります。もう少し細かい制御（条件付きの中断判定や中断再開時の処理指定など）を行うには、ここで自作の`awaitable`型を返すようにすればいいわけです。

### `co_await`

コルーチン内での`co_await`の呼び出しは、一番明示的なAwait式の実行地点です。

`co_await v`のように呼び出された時のAwait式の引数`o`は、プロミス型オブジェクトを`p`として`p.await_transform(v)`が使用可能ならばその戻り値として取得され、`p.await_transform`が見つからない場合は`v`がそのまま`o`として取得されます。

すなわち、ここでは`v`が`awaitable`ではない場合に`p.await_transform()`によってそれを`awaitable`に変換してからAwait式に渡せるようになっています。

今回の`generator`型では、`co_await`を明示的に使用する予定はないので何もする必要はありません。もし、`co_await`でも値を生成したい場合はプロミス型で`.await_transform()`を実装し、そこで受けとった値を保存するようにしておきます。

むしろコルーチンで`co_await`を使ってほしくない場合、プロミス型の`.await_transform()`を`delete`しておくことで、`co_await`利用を禁止することができます。

```cpp
#include <coroutine>

template<std::movable T>
class generator {

  struct generator_promise {

    // co_await禁止
    void await_transform() = delete;

    ...
  };

  // コルーチンハンドル
  handle m_hcoro;

  ...
};


generator<int> f() {
  co_yield 0; // ok
  co_await 1; // ng
}
```

なお、Await式における`co_await o`はこの`co_await`呼び出しとは異なるもので、再帰的に`co_await`呼び出しとして処理されることはありません。つまりは、`co_await`を直接書いたとき以外は`.await_transform()`は考慮されません。

### `co_yield`

コルーチン内での`co_yield v`の呼び出しは、プロミス型オブジェクトを`p`として`co_await p.yield_value(v)`のように書き換えられて実行されます。すなわち、`co_yield`によるAwait式の引数`o`はプロミス型のメンバ関数`.yield_value()`に`co_yield`の引数を渡して呼び出した結果として取得されます。

今回の`generator`型では、`co_yield`による値の生成時にはその都度中断しますが中断前後で他に何もしないので、`.yield_value()`は`std::suspend_always`を返すようにします。

```cpp
#include <coroutine>

template<std::movable T>
class generator {

  struct generator_promise {
    // 生成された値を保存
    T value;

    // co_yield時に呼ばれる関数
    auto yield_value(T v) {
      // 値を保存して中断
      value = std::move(v);
      return std::suspend_always{};
    }

    ...
  };

  // コルーチンハンドル
  handle m_hcoro;

  ...
};
```

前述のように、こうして保存された値はコルーチンハンドルの`.promise()`によって取得できるプロミスオブジェクトの参照経由で、コルーチン戻り値型（コルーチン呼び出し側）から取得することができます。

### `co_return`

`co_return`文そのものでは直接Await式が実行されるわけではありませんが、`co_return`は結果として最終サスペンドポイントに到達するため、そこでAwait式が実行されます。

ただし、`co_return`は最終サスペンドポイントの前にプロミス型で定義された`.return_void()/.return_value()`のどちらかを呼び出します。`co_return expr;`のように呼ばれて`expr`の結果が`void`ではないならば`.return_value()`が呼ばれ、`expr`の結果が`void`もしくは`co_return;`のように呼ばれたときは`.return_void()`が呼ばれます。どちらの場合もその実行後に最終サスペンドポイントに到達して、そこでAwait式が実行されます。

今回の`generator`型では、`co_return`を明示的に使用する予定はないので何もする必要はありません。

## `gengerator`型全景

プロミス型を含めて、これでコルーチンアプリケーション作成のために必要なことはおおよそ揃いました。ここまで小出しにしてきたため全体像が分かりづらかったので、完成した`generator`型の全体をここにまとめておきます。

```cpp
template<std::movable T>
class generator {
  // 前方宣言
  struct generator_promise;

  // 短縮のためのエイリアス
  using handle = std::coroutine_handle<generator_promise>;

  // プロミス型定義
  struct generator_promise {
    // 生成された値を保存
    T value;

    // コルーチン戻り値取得をカスタマイズする
    auto get_return_object() {
      // コルーチンハンドルからgeneratorオブジェクトを生成
      return generator{handle::from_promise(*this)};
    }

    // 初期サスペンドポイントのカスタマイズ
    auto initial_suspend() {
      return std::suspend_always{}; // 常に中断する
    }

    // 最終サスペンドポイントのカスタマイズ
    auto final_suspend() noexcept {
      return std::suspend_always{}; // 常に中断する
    }

    // co_yield時に呼ばれる関数
    auto yield_value(T v) {
      // 値を保存して中断
      value = std::move(v);
      return std::suspend_always{};
    }

    // コルーチン内での例外をハンドルする
    void unhandled_exception() { std::terminate(); }

    // co_await禁止
    void await_transform() = delete;

    // co_return;のカスタマイズ
    //void return_void();

    // co_return expr;のカスタマイズ
    //void return_value();
  };

  // コルーチンハンドル
  handle m_hcoro;

  // コルーチンハンドルを受け取るコンストラクタ
  explicit generator(handle hcoro)
    : m_hcoro(hcoro) {}

public:

  // プロミス型の公開
  using promise_type = generator_promise;

  // 値を取得する関数
  auto move_next() -> std::optional<T> {
    // コルーチン有効性をチェック
    if (!m_hcoro) return {};

    // コルーチンを再開
    // 次のco_yieldまで進む
    m_hcoro.resume();

    // 最終サスペンドポイントに到達していないことをチェック
    if (m_hcoro.done()) return {};

    // promiseオブジェクトの参照を取得
    auto& promise = m_hcoro.promise();
    // yieldされた値を取得
    return {std::move(promise.value)};
  }

  ~generator() {
    // コルーチンが有効ならば（終了していなければ）
    if (m_hcoro) {
      // コルーチンを終了させる
      m_hcoro.destroy();
    }
  }
};
```

この`generator`型は次のように使用します。

```cpp
generator<int> make_range(int first, int last) {
  for (int i = first; i < last; ++i) {
    co_yield i;
  }
}

int main() {
  auto gen = make_range(0, 10);

  std::optional<int> opt = gen.move_next();
  
  while (opt) {
    std::cout << *opt << ", ";
    opt = gen.move_next();
  }
  // 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 
}
```

### コルーチン内例外のハンドリング

完成した`generator`型の中で、一つだけ説明していなかったのがプロミス型の`.unhandled_exception()`です。これはコルーチン本体（ユーザーが書いたコード部分）で投げられる例外をハンドリングするための関数です。

初期サスペンドポイントや最終サスペンドポイント等の暗黙に生成されるコード部分での例外処理はコルーチン呼び出し元の責任ですが、コルーチン本体で発生した例外処理はコルーチン戻り値型作成者がまずハンドリングします。

```cpp
template<std::movable T>
class generator {

  struct generator_promise {
    
    ...

    // コルーチン内での例外をハンドルする
    void unhandled_exception() { std::terminate(); }
    
  };

  ...
};
```

多くの場合はこのように`std::terminate()`を呼び出してプログラムを終了させるので良いと思いますが、ハンドルして処理を継続したかったり、呼び出し側に再スローしたかったりする場合は`.unhandled_exception()`をカスタマイズすることができます。

```cpp
// 別のプロミス型実装
struct sample_promise {
  
  // コルーチン内での例外をハンドルする
  void unhandled_exception() { 
    try {
      // このようにして投げられている例外を取得できる
      std::rethrow_exception(std::current_exception());
    } catch (const std::exception& ex) {
      std::cout << ex.what() << "\n";
    } catch (...) {
      throw;
    }
  }
  
};
```

`unhandled_exception()`が正常に完了した場合、コルーチンは最終サスペンドポイントに到達します。

\clearpage

# `<stop_token>`と`std::jthread`

## `<stop_token>`

`<stop_token>`ヘッダでは、マルチスレッド処理における協調的な処理のキャンセル操作を扱うためのライブラリ機能を提供します。

### `std::stop_source/std::stop_token`

キャンセル操作のためには`std::stop_source/std::stop_token`の2つのクラスを利用し、基本的なキャンセル処理の流れは次のようになります

1. メインスレッドで、`std::stop_source`オブジェクトを初期化
2. メインスレッドで、`std::stop_source::get_token()`から`std::stop_token`を取得
3. 2で作った`std::stop_token`オブジェクトを処理スレッドへ受け渡す
4. 処理スレッド内では、`std::stop_token::stop_requested()`によってキャンセル要求をチェック（`true`が返って来た場合キャンセルされている）
5. 処理のキャンセルは、`std::stop_source::request_stop()`によって行う。

```cpp
#include <stop_token>

int main() {
  // 1. stop_sourceの初期化
  std::stop_source ss{};

  // 2. stop_tokenの取得
  // 3. stop_tokenの受け渡し
  auto f = std::async(std::launch::async, [st = ss.get_token()] {

    // 4. キャンセルされたかをチェック
    while(st.stop_requested() == false) {
      // 別スレッドで行う処理
      ...
    }

  });

  // 5. 処理のキャンセル
  ss.request_stop();
}
```

`std::stop_source/std::stop_token`のアイデアは`future`パターンと同じで、スレッド間の共有状態を用いてキャンセル要求を伝達するものです。そして、キャンセルされる側ではキャンセル要求をチェックしながら処理を実行する必要があります。

`<stop_token>`の提供するマルチスレッド処理のキャンセルとは、スレッドをいきなり中断するようなものではありません（これを非同期キャンセルと呼びます）。スレッドの非同期キャンセルのようなことはよくやりたくなりますが、RAIIと相性が悪くリソースリークの危険性があり、そのスレッドに関連する状態が不定となるなど、本質的に危険なのでやるべきではありません。面倒かもしれませんが協調的なキャンセルを常に行うべきで、そのためにこの`std::stop_source/std::stop_token`を利用することができます。

`std::stop_source/std::stop_token`オブジェクトの関連付けは1対1だけではなく多対多の構成をとることもできます。どちらもコピーとムーブが自由に行えるほか、`std::stop_source::get_token()`は呼び出しごとに同じ共有状態に参加する個別の`stop_token`オブジェクトを返します。`std::stop_source/std::stop_token`オブジェクトをコピーしても1つの共有状態に参加する`std::stop_source/std::stop_token`オブジェクトが増えるだけで、共有状態が別にコピーされるわけではありません。

これによって例えば、1つの`stop_source`から複数のスレッドの`stop_token`に対してキャンセルを行えたり、複数のスレッド（あるいは場所）に置いた`stop_source`からある一つのスレッドの`stop_token`に対してキャンセルをかける、等といったことが可能です。

```cpp
#include <stop_token>

int main() {
  std::stop_source ss{};

  std::thread th_array[5];

  for (auto& th : th_array) {
    // 1つのstop_sourceから複数のstop_tokenを取得する
    th = std::thread{[st = ss.get_token()] {
      while(st.stop_requested() == false) {
        // 別スレッドで行う処理
        ...
      }
    }};
  }

  // 複数のスレッドに対するキャンセル要求
  ss.request_stop();
  
  for (auto& th : th_array) {
    th.join();
  }
}
```

### `std::stop_callback`

`std::stop_source/std::stop_token`によるキャンセル処理において、キャンセル要求が発行されたときに何かしたくなることがあるかもしれません。それには、`std::stop_callback`を利用することができます。

`std::stop_callback`のオブジェクト初期化時に、対象の`stop_token`オブジェクトとコールバックを渡すことでコールバックの登録を行います。

```cpp
#include <stop_token>

int main() {
  
  std::stop_source ss{};
  auto st = ss.get_token();

  // 特定のstop_tokenに対して、コールバックを登録
  std::stop_callback sc{st, [] {
    // A キャンセル時コールバック処理
    ...
  }};

  auto f = std::async(std::launch::async, [st = std::move(st)] {

    while(st.stop_requested() == false) {
      // 別スレッドで行う処理
      ...
    }

    // B キャンセル後処理
    ...
  });

  // C キャンセル要求の発行
  ss.request_stop();  // コールバックはこの中で実行される

  // D
}
```

特定の`stop_token`に対して登録されたコールバックは、その`stop_token`に対してキャンセル要求を発行した所（`ss.request_stop()`内部、それを呼び出したスレッド）で実行されます。キャンセル要求を発行して（`st.stop_requested() == true`となって）からコールバックを実行する順序となり、コールバックと処理のキャンセル後処理は同時に実行される可能性があります。

上記のサンプルコードでは、`C -> A -> D`、`C -> B`の順序で処理が進行しますが、`A -> D`と`B`の処理の実行順序は不定です。

そうでなくとも、`stop_source::request_stop()`はどこから呼ばれるかわからないため、`std::stop_callback`に登録するコールバック処理はデータレースやデッドロック等を起こさないように気を付ける必要があります。

なお、既に停止要求が行われた後の`stop_token`に対してコールスタックを登録すると、その場ですぐ実行されます。

```cpp
// 1つ目のコールバック(A)
std::stop_callback sc{st, [] {
  ...
}};

ss.request_stop();  // Aが実行される

// 2つ目のコールバックの登録、即座に実行される
std::stop_callback sc{st, [] {
  ...
}};

ss.request_stop();  // 何もしない
```

`std::stop_callback`はコールバックの管理にRAIIを利用しており、登録したコールバックの解除はそのデストラクタで行われ、明示的に解除する手段は提供されていません。

## `std::jthread`

`std::jthread`はデストラクタで`join()`する`std::thread`です。これは`<thread>`に配置されます。

```cpp
#include <thread>

int main() {
  {
    std::jthread th{[] {
      // 別スレッドで行う処理
      ...
    }};

    // jthreadオブジェクトの破棄時（スコープを抜けるとき）
    // .join()を呼ぶことでスレッドの終了を待機する
  }
  {
    std::jthread th{[] {
      // 別スレッドで行う処理
      ...
    }};

    // スレッドを手放すこともできる
    th.detach();

    // この場合デストラクタでは何もしない
  }
}
```

`std::thread`は、デストラクタが呼ばれるまでの間に`join()`も`detach()`もされていない場合はデストラクタで`std::terminate()`が呼ばれてプログラムを終了させていました。`std::jthread`では、その場合でも自動で`join()`することでこのような振る舞いを避けるとともに、`join()`の呼び出しを省略することができるようになります。

`std::jthread`はデストラクタ（と後述のキャンセル操作サポート）以外の所では意味論も含めて`std::thread`と同一であり、`std::thread`の上位互換として利用することができます。

### 協調的キャンセル操作のサポート

`jthread`にはさらに、前述の`std::stop_source/std::stop_token`による協調的キャンセル機構のサポートが組み込まれています。

利用するにはまず、`jthread`に渡す処理の第一引数で`std::stop_token`を受け取るようにしたうえで、`std::jthread::get_stop_source(),std::jthread::get_stop_token()`メンバ関数によって`stop_source/stop_token`を取得して利用します。

```cpp
#include <thread>

int main() {
  // 渡す処理の1つ目の引数でstop_tokenを受ける
  // 追加の引数は2つ目以降で受け取る
  std::jthread th{[](std::stop_token st, int arg) {

    while(st.stop_requested() == false) {
      // 別スレッドで行う処理
      ...
    }
  }, 0);

  // jthreadに関連付けられたstop_tokenの取得
  auto st = th.get_stop_token();

  // コールバックの登録
  std::stop_callback sc{st, [] {
    ...
  }};

  // jthreadに関連付けられたstop_sourceの取得
  auto ss = th.get_stop_source();

  // キャンセル要求
  ss.stop_request();
}
```

`jthread`オブジェクトから`.get_stop_source()/.get_stop_token()`で`stop_source/stop_token`を取得した後は、`std::stop_source/std::stop_token`と同じように扱うことができます。

なお、`jthread`のデストラクタでは`join()`する前に関連付けられた`stop_token`に対してキャンセル要求を発行（`stop_request()`）しており、自動`join()`による意図しないフリーズの可能性を低減しています。

```cpp
// std::jthreadのデストラクタ実装例
class jthread {
  // スレッドに渡すstop_tokenを生成するstop_source
  stop_source ss_;

  ...

  ~jthread() {
    // joinもdetachもされていなければ
    if (this->joinable()) {
      // キャンセル要求を送ってから
      ss_.request_stop();
      // joinする
      this->join();
    }
  }
};
```

ただし、`stop_token`によるキャンセルをサポートするのはプログラマの責任です。`jthread`による利益を最大化するには、`jthread`で実行する処理では必ず`stop_token`を受け取りキャンセル要求をハンドルできるようにしておくことが推奨されます。

### 自動`join()`の理由

`jthread`はなぜデストラクタで自動`join()`するのでしょうか？そしてなぜ、わざわざ標準に追加されたのでしょうか？

このことは、`std::thread`のベースとなった`boost::thread`における問題に端を発しています。

```cpp
#include <boost/thread/thread.hpp>

// 例外を投げうる処理
void maybe_throw() noexcept(false);

int sample() {
  int a;

  boost::thread th{[&a] {
    
    ...

    // 参照するローカル変数を書き換える
    a = 10;

    ...
  }};

  maybe_throw();  // 例外を送出した場合

  th.join();
  return a;

  // boost::threadのデストラクタは自動detach()していた
}
```

このコードにおいて、`maybe_throw()`が例外を送出した場合、`th.join()`は呼ばれず`th`のデストラクタが呼ばれることになります。以前の`boost::thread`はデストラクタで`detach()`を呼んでいたためスレッドは切り離されてデストラクタの実行は完了し、この関数の実行は終了します。

この時問題となるのは、別スレッドの処理がローカル変数の参照を持っており、それを書き換えうることです。多くの処理系では、ローカル変数はスタック領域に配置されており、スタック領域は1つのスレッドの実行において使いまわされています。さらに、スタック領域には関数の戻り先アドレス等の情報も置かれています。

そのような処理系では、上記例のように関数が例外で終了した場合に呼び出し元ではその関数のスタック領域がほぼ確実に再利用されており、スレッドが参照するローカル変数（`a`）の領域には別のものが置かれている可能性が非常に高くなります。この領域に何かを書き込めばメモリの状態を破壊することになり、上記例の関数が例外で終了した場合は別スレッドで実行されている処理がまさにそれを行います。

スレッドと呼び出し元の処理の内容やタイミングによって何が書き換えられるのかは予測不可能であり、実行ごとに異なる結果が得られるなど発見が困難なバグの原因となり得ます。

`std::thread`はこの問題を回避（軽減）するために、デストラクタの実行までに`join()/detach()`のどちらかが呼ばれていなければ`std::terminate()`する、という設計を採用することで、この問題が起きた時は常にプログラムが終了するようにしています。

ところで、この問題の回避策としてはもう一つ、デストラクタで自動的に`join()`するという方法があります。`detach()`の時とは異なり、`join()`の場合はその関数が終了する前にスレッドの終了を待機するため、先程のようなメモリ破壊の問題は起こらず、例外安全性を提供することができます。

しかし、例外発生時というのは得てして意図的ではなく、スレッドの終了のための処理が実行済みであるとは限りません。すると、デストラクタによる自動`join()`は謎のフリーズ（スレッドが終了しないため）という別の問題を起こす可能性があります（`try-catch`で全体を囲うだけではこれを回避できません）。そのため、`std::thread`ではその方法は採用されず、メモリ破壊が起こりうる場合はプログラムを終了させるという安全側に倒した設計とされたようです。

デストラクタによる自動`join()`を採用できない理由は、スレッドで実行されている処理を`join()`の前にキャンセルさせることができない、あるいはその方法が無いことにあります。`std::jthread`では、`stop_token`による処理の協調的なキャンセル機構を組み込んでいることで`join()`の前に処理のキャンセルをかけることができ、フリーズの可能性を低減することができます。ただし、処理のキャンセルを行うのはプログラマの責任です。

\clearpage
# `<semaphore>`

セマフォは並行処理における複数スレッド間の同期を取るための仕組みで、おそらく最も単純かつ基本的なものです。

セマフォの実体はリソースの使用可能数を表示するカウンタであり、利用者（スレッド）が1増えるごとにカウンタを1減算していき、カウンタが0になったらリソースが使用可能でないことを表します。セマフォは、このようなカウンタを並行処理における同期のための意味論でラップし、カウンタを隠蔽すると共にインタフェースを制限することで並行処理における同期のために使いやすく（あるいはそれにしか使えないように）したものです。

`<semaphore>`で提供されるセマフォは`std::counting_semaphore<N>`という名前のクラステンプレートで、テンプレートパラメータ`N`にはリソースの使用可能数（当然正の整数）を指定します。

```cpp
#include <semaphore>

int main() {
  // 何か共有リソース
  auto shared_resource = ...;

  // セマフォの初期化、同時に2スレッドが使用可能とする
  std::counting_semaphore<2> cs{2};

  std::jthread th_array[5];
  for (auto& th : th_array) {
    // 複数スレッドが共有リソースを任意のタイミングで使用する
    th = std::jthread{[&shared_resource]{
      ...

      // リソースの使用を開始
      // カウンタを減算する、カウンタが0なら1以上になるまでブロック
      cs.acquire();

      // リソースの使用（読み書き）
      use_resource(shared_resource);

      // リソースの使用を終了
      // カウンタを加算する
      cs.release();

      ...
    }};
  }
}
```

`std::counting_semaphore<N>`の初期化においてはそのコンストラクタでセマフォ内部カウンタの初期値を指定します。これは通常`N`を指定しますが使い方によってはそれよりも小さい数や0を指定することもできます（大きい値や負の値を指定すると未定義動作となります）。

`std::counting_semaphore`のメンバ関数は非常に限定されており、後述の`try~`系インターフェースを除くと`acquire()/release()`の2つしか利用可能ではありません。`.acquire()`はリソースの使用開始前に呼び出してセマフォの値を1減算します。その際、セマフォの値が既に0ならば値が1以上になるまでその場で待機します。`.release()`はリソース使用終了後に呼び出してセマフォの値を1加算します。その際、セマフォの値が0から1になっていたら、`.acquire()`を呼び出して待機している他のスレッドのブロックを解除します。

### `std::binary_semaphore`

複数スレッド間で共有するリソースで同時に2つ以上のスレッドが変更をかけても平気なものというのは稀であり、同期のためにセマフォを採用してリソースの変更を行う場合、その利用可能数（テンプレートパラメータ`N`）は1であることがほとんどでしょう。そのため、`std::counting_semaphore<1>`の別名として`std::binary_semaphore`が用意されています。

```cpp
// counting_semaphoreとbinary_semaphoreの宣言例
namespace std {

  template<ptrdiff_t N>
  class counting_semaphore;

  using binary_semaphore = counting_semaphore<1>;
}
```

`std::binary_semaphore`は`N`の値が事前指定されている以外は`std::counting_semaphore`と全く同じように使用できます。

あるリソースにアクセス可能なスレッドを同時に1つに制限する（排他制御する）という観点からは`std::binary_semaphore`とミューテックス（`std::mutex`など）は同一視することができます。しかし、ミューテックスはロックを特定のスレッドが所有する（ロックを取得したスレッド以外からロック解除できない）のに対して、`std::binary_semaphore`（`std::counting_semaphore`）はスレッドに関係なくどこからでもカウンタ値の減算を行うことができます。セマフォとミューテックスの使い分けに迷ったら、そのような使い方をしたいかどうかを考えてみるといいかもしれません。

## `try_acquire`

`.acquire()`はセマフォ値が0なら0以上になるまでそのスレッドをブロックし待機します。場合によってはそのような振る舞いは望ましくなく、すぐにリターンした上で失敗したことを判別できたほうがいい場合もあります。そのために`try_acquire()/try_acquire_for()/try_acquire_until()`メンバ関数が用意されており、これらを用いれば必要以上の時間スレッドをブロックしてしまうのを回避できます。

```cpp
#include <semaphore>

using namespace std::chrono_literals;

template<typename T>
void try_acquire_example(std::counting_semaphore<3>& cs, T& shared_resource) {
  ...

  // セマフォ値が0ならすぐにリターン
  if (cs.try_acquire() == false) {
    // セマフォ値の減算に失敗（リソース利用権を得られなかった）
    ...
    return;
  }
  // trueを返したとき、セマフォ値の減算に成功（リソース使用権を獲得）

  ...

  // セマフォ値が0なら指定された時間待機
  if (cs.try_acquire_for(2s) == false) {
    // タイムアウトのため、セマフォ値の減算に失敗
    ...
    return;
  }
  // trueを返したとき、セマフォ値の減算に成功

  ...

  // セマフォ値が0なら指定された時間まで待機
  if (cs.try_acquire_until(std::chrono::steady_clock::now() + 10s) == false) {
    // タイムアウトのため、セマフォ値の減算に失敗
    ...
    return;
  }
  // trueを返したとき、セマフォ値の減算に成功

  ...
}
```

`.try_acquire()`はセマフォ値が0なら全く待機せずすぐにリターンし、`.try_acquire_for()`は指定した時間が過ぎるまで、`.try_acquire_until()`は指定した時間になるまでセマフォ値が0以上になるのを待機します。3つの関数は全て`bool`値を返し、`true`となる時はセマフォ値の減算に成功したことを、`false`となる時はセマフォ値の減算に失敗した（指定した期間内に0以上にならなかった）ことを表します。

\clearpage

# `<latch>`と`<barrier>`

`<latch>`（ラッチ）と`<barrier>`（バリア）はどちらも、セマフォと同様に並行処理における複数スレッド間の同期を取るための機能を提供します。

## `std::latch` - 一回の同期

ラッチの実体はある点に到達したスレッド数をカウントするカウンタであり、その点をスレッドが通過するごとにカウンタを減算していき、同期を取るスレッドはラッチのカウンタが0になるまで待機します。事前に指定した数のスレッドがその点を通過しカウンタが0になった時、待機しているスレッドを再開させることで同期を取ります。

C++におけるラッチは`std::latch`というクラスで実装されており、カウンタの初期値（想定する通過スレッド数）をコンストラクタで指定し、その減算は`.count_down()`、0になるまで待機するのは`.wait()`で行います。コンストラクタに与えるカウンタ初期値は当然正の整数でなければなりません。

```cpp
#include <latch>

// 複数スレッド立ち上げ時に、処理の開始を待ち合わせる例
int main() {

  // ラッチオブジェクト初期化、カウント数を指定
  std::latch start{1};

  // スレッドの立ち上げ
  std::jthread th_array[5];
  for (auto& th : th_array) {
    th = std::jthread{[&start]{
      // カウンタが0になるまで待機
      start.wait();

      ...
    }};
  }

  // カウンタを減算（1 -> 0）
  // .wait()によって待機しているスレッドの実行が開始される
  start.count_down();

  ...
}
```

複数スレッド間の同期を取る場合、この例とは逆に終了の同期や処理の進行のあるタイミングを揃えるなどの目的がメインだと思います。その場合、各スレッドではラッチを減算してから待機という手順を踏むため、これを一度で行うことができるメンバ関数`.arrive_and_wait()`が用意されています。

```cpp
#include <latch>

// データ並列処理において、処理の最小単位を担う関数
void kernel(std::latch& sync, int id, 
            std::span<const std::byte> input,
            std::span<int> temp,
            std::span<int> output)
{

  // 並列実行する処理
  for (auto b : input) {
    ...
  }

  // 途中結果を保存
  // idによって出力先が異なるためここで同期は不要
  temp[id] = ...;

  // 全カーネル（スレッド）の途中結果保存完了を待機する
  sync.arrive_and_wait(); // 全カーネルに渡る同期ポイント

  // 他スレッドの途中結果を使用する必要がある処理
  for (auto n : output) {
    ...
  }

  // 最終結果を出力
  output[id] = ...;
}


int main() {
  // 処理対象のデータ、画像とか行列とかの2次元データとする
  // vectorの要素数=行数
  std::vector<std::span<std::byte>> input_data = ...;
  // 結果出力先
  std::vector<int> result = ...;
  // 途中結果保存用バッファ
  std::vector<int> temp(input_data.size(), 0);
  // スレッドID
  int id = 0;

  // 処理対象のデータ数=立ち上げるスレッド数でラッチを初期化
  std::latch sync{input_data.size()};
  // 全スレッドの終了を待機するためのラッチ
  std::latch end{input_data.size()};

  // 入力データを行ごとに処理する
  for (auto span : input_data) {
    std::thread{[&sync, id, span, &temp, &result, &end]{
      // カーネルの実行
      kernel(sync, id, span, temp, result);

      // カーネルの完了通知
      end.count_down();
    }}.detach();
    ++id;
  }

  // 全スレッド処理完了を待機
  end.wait();

  // resultを使用する処理
  ...
}
```

`.arrive_and_wait()`は`.count_down()`してから`.wait()`するだけで他のことはせず、この2つを個別に呼び出した時と同じ効果となります。ただ、2つの処理が1つにまとまっていることで、`.arrive_and_wait()`の呼び出し地点はマルチスレッドコードにおける1つの同期ポイントとして見ることができ、同じラッチオブジェクトを使用するスレッドはその同期ポイントで待ち合わせを行うことを明確化できます。

なお、`std::latch`はカウンタを加算あるいは再設定する機能がない（コピーや`swap`も不可な）ため、`std::latch`による同期は一度だけ使用可能です。

カウンタを使って同期をとるという点はセマフォとラッチで共通しています。しかし、セマフォはカウンタ値が0になったときに待機し1以上になると再開しますが、ラッチはカウンタが0になるまで待機し0になったときに再開します。そして、セマフォは主に共有リソースの利用権管理に使用され、ラッチは複数スレッドの進行管理に使用されます。

## `std::barrier` - 複数回の同期

ラッチによる同期は1つの`std::latch`オブジェクトにつき一度だけしか行えませんが、複数スレッドで継続的に実行されている処理についてある点での同期を何度も行いたい場合もあるでしょう。そのために`std::barrier`が用意されており、`std::barrier`はラッチと同等の同期を複数回行うことができます。これは例えば、Fork-Joinモデルと呼ばれるタイプの並行処理の実装に使用できます。

```cpp
#include <barrier>

// 複数のスレッドで呼ばれる処理単位
void fork_proc(std::barrier<>& sync, 
               std::span<const std::byte> input,
               std::stop_token st)
{
  // キャンセルされるまで行われる連続処理
  // ループごとに全スレッドで同期しつつ実行される
  while(st.stop_requested() == false) {

    // メインの処理
    ...

    // 全スレッドはここで待ち合わせる（1度の処理の完了を同期する）
    // カウンタが0になった時、カウント値をリセットしてから再開する
    sync.arrive_and_wait();
  }
}


int main() {
  // 処理対象のデータ
  std::vector<std::span<std::byte>> input_data = ...;
  
  // 処理対象のデータ数=立ち上げるスレッド数でバリアを初期化
  std::barrier<> sync{input_data.size()};

  // キャンセル用stop_source
  std::stop_source ss;

  // 入力データを行ごとに処理する
  for (auto span : input_data) {
    std::thread{[&sync, span, &ss]{
      fork_proc(sync, span, ss.get_token());
    }}.detach();
  }

  ...

  // 処理の中断
  ss.request_stop();

  // 後処理
  ...
}
```

`std::barrier`の`.arrive_and_wait()`の呼び出しでは、ラッチの時と同様にカウンタ値を1つ減算してからカウンタが0になるのを待機しますが、カウンタが0になった時はカウンタ値をリセット（コンストラクタで指定された値に戻す）してから待機中のスレッドを再開します。これによって、1つの`std::barrier`オブジェクトは複数回同期に使用することができます。

Fork-Joinモデルのような並行処理を行う場合の同期ポイントでは何か同期が必要な処理を行いたいはずです。そしてそれはどこか1つのスレッドで全スレッドがその同期ポイントに到達してから実行する必要があるでしょう。これを行おうとすると、1つのスレッドを特別扱いするなど少し面倒になります。そこで、`std::barrier`はコンストラクタの第2引数でそのような処理（完了関数）を受けとり、カウンタが0になった時に実行させることができます。

```cpp
#include <barrier>

// 複数のスレッドで呼ばれる処理単位
template<typename CF>
void fork_proc(std::barrier<CF>& sync, 
               std::span<const std::byte> input,
               std::stop_token st)
{
  // キャンセルされるまで行われる連続処理
  // ループごとに全スレッドで同期しつつ実行される
  while(st.stop_requested() == false) {

    // メインの処理
    ...

    // 全スレッドはここで待ち合わせる（1度の処理の完了を同期する）
    // カウンタが0になった時、CFの処理を実行してからカウント値をリセットして再開する
    sync.arrive_and_wait();
  }
}


int main() {
  // 処理対象のデータ
  std::vector<std::span<std::byte>> input_data = ...;
  
  // 処理対象のデータ数=立ち上げるスレッド数でバリアを初期化
  // 同時に、同期ポイントで再開直前に実行する完了関数を指定
  std::barrier sync{input_data.size(), [&input_data] {
    // 同期ポイントで再開前に実行する必要のある処理
    // 例えば入力データの更新など
    for (auto span : input_data) {
      ...
    }
  }};

  // キャンセル用stop_source
  std::stop_source ss;

  // 入力データを行ごとに処理する
  for (auto span : input_data) {
    std::thread{[&sync, span, &ss]{
      fork_proc(sync, span, ss.get_token());
    }}.detach();
  }

  ...

  // 処理の中断
  ss.request_stop();

  // 後処理
  ...
}
```

このように`std::barrier`に渡した完了関数は同期ポイント（`.arrive_and_wait()`）の内部で、カウンタが0になったときに実行されます。同じバリアオブジェクトで待機しているスレッドのいずれかで実行され、この処理の完了後にカウンタをリセットして、待機しているスレッドが再開されます。つまり、完了関数は1つのスレッドだけで実行され、カウンタ0に到達->完了関数実行->カウンタのリセット->待機スレッドの再開、の順で実行されます。

`std::barrier`は完了関数を型消去することなく受け取るためにクラステンプレートとなっており、完了関数の型（ファンクタや関数型）をテンプレートパラメータにとっています。完了関数を渡さない（何もしない）場合は`std::barrier<>`のように指定し、完了関数を渡す場合でもクラステンプレートの実引数推定を利用することでテンプレートパラメータの指定を省略できます。なお、完了関数は引数なしで呼び出し可能なように定義されている必要があります。

### `arrive_and_drop()`

場合によっては、同期するスレッドグループの一部を何かしらの条件で早期に終了したい場合があるかもしれません。その場合は`.arrive_and_drop()`を使用することで、同期ポイントに到達したことと処理を停止する（同期するスレッドグループから離脱する）ことだけを通知して待機せずに次の処理に移ることができます。

```cpp
#include <barrier>

using namespace std::chrono_literals;

// 1ループごとに1つづつスレッドを減らしていく例
template<typename CF>
void and_then_there_were_none(std::barrier<CF>& sync, 
                              int id,
                              std::span<const bool> killed)
{
  while (true) {
    std::this_thread::sleep_for(1s);

    if (killed[id] == true) {
      // カウンタを減算するだけで待機しない
      // その際、カウンタの最大値（コンストラクタで指定した数）から1引く
      // ここでカウンタが0になった時も完了関数が実行される
      sync.arrive_and_drop();
      return;
    }

    // カウンタを減算し待機
    sync.arrive_and_wait();
  }
}


int main() {
  // スレッド数
  constexpr int N = 10;

  // スレッド終了フラグ
  int count = 0;
  bool killed[N]{};

  // バリアの初期化
  std::barrier sync{N, [&] {
    // 1度呼ばれるごとに順番にスレッドを終了させる
    killed[count] = true;
    ++count;
  }};

  // 全スレッド完了待機のためのラッチ
  std::latch end{N};

  // N個のスレッドを起動
  for (int id : std::views::iota(0, N)) {
    std::thread{[&sync, id, &end, &killed]{
      and_then_there_were_none(sync, id, killed);
      end.count_down();
    }}.detach();
  }

  // 全スレッド完了待機
  end.wait();
}
```

`.arrive_and_drop()`はカウンタ値の減算を行うとともに、`std::barrier`内部のカウンタ最大値（コンストラクタで渡された初期値であり、カウンタリセット時に指定する値）も1つ減算しておくことで、次のループの処理において同期ポイントに到達すべきスレッド数を1つ減らします。そして、`.arrive_and_drop()`の呼び出しはその場で待機せずすぐに完了します。`.arrive_and_drop()`呼び出し時にカウント値が0になった時でも完了関数は呼び出され、完了関数が終了してからカウント値の減算を行います。

\clearpage

# `<syncstream>`

並行処理において、並列に実行されている各スレッドそれぞれで思い思いに標準出力を行いたいときはよくありますが、C++においてこれは長年悩みの種でした。例えば何も考えずにいつも通りに出力すると、各スレッドからの出力がランダムにまぜこぜになってしまいます。 

```cpp
// スレッド毎に異なる種類の草を生やす
void grow_grass(std::string_view grass) {
  for ([[maybe_unused]] auto i : std::views::iota(0, 20)) {
    std::cout << grass;
  }

  std::cout << "\n";
}

int main() {
  std::jthread t1{[] {grow_grass("w");}};
  std::jthread t2{[] {grow_grass("v");}};
  std::jthread t3{[] {grow_grass("w");}};
  std::jthread t4{[] {grow_grass("W");}};
}
```

この実行結果は例えば次のようになります。

```
wwwwwwwwwwwwwwwwwwww
vvvvvvvvvvvvvvvvvvvv
WwWwwwwwWwWWWWWWWWwWWWWWWwwWWW
wwwwwwwwww
```

どのように混ざるかはほぼランダムであり、たまに意図通りに出力されることもあります。一応標準としては、このように複数のスレッドから同時に`std::cout`に出力を行った時でも、データが消えたり`std::cout`のストリームの状態が壊れたりしないことは保証しています。しかし、その出力順序については何も保証してくれません。

これを避けるためには`std::cout`に出力するところで同期を取る必要があります。C++20まではそのためにミューテックスによるロックを使用できましたが、出力1つにつき全スレッド間で同期を取る必要があるなどパフォーマンス面で使いづらく、かといって各スレッドに出力バッファを持っておいてそこに出力しておいてから最後にまとめて`std::cout`に出力、というのも実装が面倒でした。

C++20では、複数スレッドからの同期した標準出力のために`std::osyncstream`を利用できるようになります。`std::osyncstream`は`std::basic_osyncstream<charT, traits, Allocator>`の`char`特殊化のエイリアスであり、他にも`wchar_t`の特殊化である`std::wosyncstream`が用意されています。以降は、これらを代表して`std::osyncstream`で説明を行います。

```cpp
#include <syncstream>

void grow_grass(std::string_view grass) {

  // 出力したいストリーム（std::ostream）を用いて初期化する
  std::osyncstream out{std::cout};
  
  // このスレッド内でのout経由の出力は他のスレッドの出力と競合しない（混ざることはない）
  for ([[maybe_unused]] auto i : std::views::iota(0, 20)) {
    out << grass;
  }

  out << "\n";
}

// main()は先ほどと同じ
```

この出力は例えば次のようになります。

```
wwwwwwwwwwwwwwwwwwww
vvvvvvvvvvvvvvvvvvvv
WWWWWWWWWWWWWWWWWWWW
wwwwwwwwwwwwwwwwwwww
```

各行（各スレッド間）の順番は相変わらずランダムですが、行内（各スレッド内）の出力が混ざることはありません。

`std::osyncstream`は`std::ostream`でもあるため、`std::cout`と同じように扱って出力操作を行うことができます。

## `std::osyncstream`の振る舞いの詳細

`std::osyncstream`はコンストラクタで渡された出力ストリームオブジェクトをラップして、それに対する出力についてグローバルに同期を取ります。しかし、その同期の単位は1度の出力（`<<`）ごとではなく、1つの`std::osyncstream`オブジェクトごとになります。つまり、1つの`std::osyncstream`オブジェクトに対する出力はバッファリングされており、そのデストラクタの実行時にまとめてラップしているストリームに出力しており、グローバルな同期はこの時（デストラクタにおける出力時）に行われています。

```cpp
// osyncstreamによる同期化出力の例
{
  std::osyncstream bout(std::cout);

  bout << "Hello, ";      // #1
  bout << "World!";       // #2
  bout << std::endl;      // #3 ストリームのフラッシュが記録される
  bout << "and more!\n";  // #4
} // このスコープを抜けるとき、boutに出力された文字列がcoutへ転送され、coutはフラッシュされる。
```

この例の`#1`~`#4`の行では`std::cout`への出力はまだ行われておらず、標準出力には何も出力されていません。`std::endl`によるフラッシュも含めて、全ての出力は`std::osyncstream`の内部バッファに記録されているだけです。`std::osyncstream`オブジェクトが破棄されるとき、そのデストラクタにおいて内部バッファの内容が全て`std::cout`（コンストラクタで渡したストリーム）に転送され、その転送の際にだけ同期がとられます。この同期はグローバルに行われており、`std::osyncstream`の異なるオブジェクトの間でも確実に同期します（同時出力しない）。

これによって、各スレッドから任意のタイミングで出力を行なっているときでも、1度の出力毎に全スレッドで同期を取る必要がなくなり、パフォーマンスへの影響を最小化しています。

場合によってはこのような同期と出力のタイミングを制御したい、あるいはすぐに出力したいということがあるでしょう。その場合は`std::osyncstream`を右辺値で使用するか、`.emit()`メンバ関数を利用することができます。

```cpp
void example_temporary() { 
  // osyncstreamの一時オブジェクトに対して出力すると
  // この行の終わりでデストラクタが呼び出されすぐ出力される、ただしフラッシュはされない
  std::osyncstream{std::cout} << "Hello, " << "World!" << '\n';
}

void example_emit() {
  std::osyncstream bout{std::cout};

  bout << "Hello," << '\n';       // フラッシュされない
  bout.emit();                    // coutに文字列が転送される、フラッシュはされない
  bout << "World!" << std::endl;  // フラッシュを記録、フラッシュはされていない
  bout.emit();                    // coutに文字列が転送され、フラッシュされる
  bout << "Greetings." << '\n';   // フラッシュされない

} // boutの破棄時、残りの文字列がcoutに転送される、フラッシュはされない
```

実際のところ、ラップしているストリームへの出力を担っているのはこの`.emit()`であり、`std::osyncstream`のデストラクタでは`.emit()`を呼び出すことで出力を行なっています。そして、`.emit()`の呼び出しにおいては出力先ストリームのフラッシュを行わないため、厳密に即座に出力するためには`std::osyncstream`オブジェクトに対してあらかじめフラッシュを行なっておく（内部バッファにフラッシュを記録しておく）必要があります。

このような振る舞いのため、1つの`std::osyncstream`オブジェクトを複数のスレッドで共有して出力を行う場合には正しく同期しません。

```cpp
#include <syncstream>

// osyncstreamを複数のスレッドで共有すると正しく同期されない
std::osyncstream out{std::cout};

void grow_grass(std::string_view grass) {

  // outを使用するスレッド間で出力が混ざりうる
  for ([[maybe_unused]] auto i : std::views::iota(0, 20)) {
    out << grass;
  }

  out << "\n";
}
```

`std::cout`などの場合はこのように使用したときでも安全（データ競合を起こしてストリームの状態が壊れたりしない）ですが、`std::osyncstream`の場合はそのような保証はなく、このように使用してしまうと`std::osyncstream`の持つバッファの状態が壊れ、未定義動作の世界に突入するでしょう。すなわち、`std::osyncstream`自体はスレッドセーフではありません。

\clearpage

# `<chrono>`

C++20では、`<chrono>`はカレンダーとタイムゾーンを扱えるように大きく拡張されています。

以下、本文及びサンプルコードでは`std::chrono`を省略します。

## 新しい時計型

時計型とは時計を表現するための型のことで、これまでは`system_clock`、`steady_clock`、`high_resolution_clock`の3つの時計型がありました。ここに次の5つの新しい時計型が追加されます。

- `utc_clock` : UTC（協定世界時）時間を表す
- `tai_clock` : TAI（国際原子時）時間を表す
- `gps_clock` : GPS時間を表す
- `file_clock` : ファイル時間を表す
    - `std::filesystem::file_time_type`のための時計型
- `local_t` : ローカル時間を扱うためのダミーの時計型

C++20からは、`system_clock`の時間はUTC（かつ表現はUNIX時間）である事が規定されました（これは全ての実装がそうだったために標準でも規定することになったものです）。`system_clock`と`utc_clock`の違いはうるう秒のカウントの有無で、`system_clock`ではうるう秒がカウントされていません。

TAIは厳密に定義された秒に従って時間をカウントする最も正確な時計であり、UTCはそれに対して日常生活に適合するようにうるう秒などによって調整された時計です。`tai_clock`と`system_clock`はどちらもうるう秒をカウントしませんが、エポック等が異なるためその時刻も異なります。

`file_clock`は`<filesystem>`においてファイルの作成・更新日時を扱うための時計型で、UTC時間に従う時計ではありますがその詳細は未規定であり、タイムゾーンが異なるなど環境の影響を強く受けると思われます。

`local_t`はローカル時間を扱うための疑似的な時計型で、一切のメンバを持たない空の構造体として定義されています。これは他の時計型とは異なり、厳密には時計型ではありません。

任意の時計型を`Clock`とすると、`local_t`を除く全ての時計型で`Clock::now()`静的メンバ関数からその時計の指す現在の時刻が取得できます。

```cpp
#include <chrono>

template<typename C>
void print_time() {
  // 現在時刻を出力する
  std::cout << C::now() << "\n";
}

int main() {
  print_time<system_clock>();
  print_time<utc_clock>();
  print_time<tai_clock>();
  print_time<gps_clock>();
  print_time<file_clock>();
}
```

`Clock::now()`の返す値は時間軸上の一点を指す`time_point`型の値です。ただし、`time_point`型はクラステンプレートであり、時計型と時間間隔（精度）を表す型（`duration`型）によって特殊化されており、時計型毎に`Clock::now()`の戻り値は異なっています。それらの型にアクセスするために、各時計型の使用する`time_point`型のエイリアスが用意されています。

- `system_clock`
    - `sys_time` : システム時刻を表す`time_point`
      - `sys_seconds` : 秒単位のシステム時刻を表す
      - `sys_days` : 日単位のシステム時刻を表す
- `utc_clock`
    - `utc_time` : UTC時刻を表す`time_point`
      - `utc_seconds` : 秒単位のUTC時刻を表す
- `tai_clock`
    - `tai_time` : TAI時刻を表す`time_point`
      - `tai_seconds` : 秒単位のTAI時刻を表す
- `gps_clock`
    - `gps_time` : GPS時刻を表す`time_point`
      - `gps_seconds` : 秒単位のGPS時刻を表す
- `file_clock`
    - `file_time` : ファイル時刻を表す`time_point`
- `local_t`
    - `local_time` : ローカル時間を表す`time_point`
      - `local_seconds` : 秒単位のローカル時間を表す
      - `local_days` : 日単位のローカル時間を表す

`Clock::now()`の返す型は、時計型`xxx_clock`に対して`xxx_time`と命名されるエイリアステンプレートです（`system_clock`だけは`sys_time`と省略されます）。ただし、これらの`time_point`の精度（`duration`型）は未規定であり、おそらく秒単位よりも高い精度が使用されます。`xxx_seconds`などとなっているエイリアスは命名からわかるように、秒単位（あるいは日単位）の精度を持つ`xxx_time`の特殊化されたエイリアスです。

`local_t`型は、この`local_time`のための時計型としての役割しかなく、`local_time`の値はカレンダー機能などを通して取得できます。これらローカル時間に関しては、後でタイムゾーンのところで詳しく説明することにして、しばらく存在を忘れます。

各時計型の示す時刻というのは時計型の参照する時計によって異なっているため、基本的にはこれらの`time_point`型（エイリアス）の間には直接の互換性がありません。また、同じ時計型についての`time_point`の間でも、`duration`型が異なっていて変換によって端数が出うる場合（精度が低くなる方向の変換）は暗黙変換が提供されません。例えば、分->秒の変換は端数が出ませんが、秒->分の変換は端数が出ます。そのような変換には、`time_point_cast<D>()`によって明示的な変換を行う必要があります。

```cpp
#include <chrono>

// UTC時刻を秒単位で受けたい
void f(utc_seconds time);

int main() {
  // 秒単位よりも精度が高い時（例えばミリ秒単位）
  utc_time t = utc_clock::now();

  f(t); // ng、ミリ秒から秒への変換は端数が出うる
  f(time_point_cast<utc_seconds::duration>(t)); // ok
  f(time_point_cast<seconds>(t));               // ok
}
```

`time_point_cast<D>()`の`D`には変換先の`duration`型を指定する必要があり、`time_point`型及びそのエイリアスからは`::duration`メンバ型としてそれを取得する事ができます。あるいは、`seconds`や`milliseconds`などの`duration`型エイリアスを指定することもできます。

### 時計型間の時刻の変換

時計型の間でその時刻値を変換するには、時計型の静的メンバ関数`to_xxx()/from_xxx()`を利用します。`xxx`には変換先の時計の略称が入り、全ての時計型間で相互の直接変換が提供されているわけではありません。ただし、`steady_clock`と`high_resolution_clock`はこのようなメンバ関数を提供していません。

- `system_clock`
    - `to_time_t()/from_time_t()` : `time_t`型（Cライブラリの時刻表現型）と相互変換可能
- `utc_clock`
    - `to_sys()/from_sys()` : `system_clock`と相互変換可能
- `tai_clock`
    - `to_utc()/from_utc()` : `utc_clock`と相互変換可能
- `gps_clock`
    - `to_utc()/from_utc()` : `utc_clock`と相互変換可能
- `file_clock`
    - 次のどちらか
      - `to_utc()/from_utc()` : `utc_clock`と相互変換可能
      - `to_sys()/from_sys()` : `system_clock`と相互変換可能

`file_clock`がどういう経路で変換可能であるかは実装によって異なる可能性があり、少なくともどちらかの変換は提供されていることしか保証がありません。

![](./img/clock_type_conv.png)

（この画像は覚えなくていいです）

よく見ると`utc_clock`を介せば全ての時計型間で相互変換が可能そうですが、時計型間の変換を統一的かつシンプルに行うには何かしらのラッパが必要そうです。そのため、これら時計型間の変換を統一的に行うための`clock_cast<C>()`が用意されています。これを使っていれば時計型間の変換可能性とその方法などを意識せずに変換する事ができます。

```cpp
#include <chrono>

int main() {
  sys_time st = system_clock::now();
  utc_time ut = utc_clock::now();
  tai_time tt = tai_clock::now();
  gps_time gt = gps_clock::now();
  file_time ft = file_clock::now();

  // UTC -> System Time
  sys_time t1 = clock_cast<system_clock>(ut);

  // GPS Time -> TAI
  tai_time t2 = clock_cast<tai_clock>(gt);

  // File Tiem -> GPS Time
  gps_time t3 = clock_cast<gps_clock>(ft);

  // UTC -> File Tiem
  file_time t4 = clock_cast<file_clock>(ut);

  // TAI -> UTC
  utc_time t5 = clock_cast<utc_clock>(tt);
}
```

`clock_cast<C>()`の`C`には変換先の時計型を指定します。これは時計型を基準とした`time_point`値の変換なので、`clock_cast()`の入力と出力はどちらも`time_point`型の値です。なお、`time_t`は時計型でも`time_point`型でもないため`clock_cast()`で変換することはできず、`system_clock::from_time_t()`によって一度`sys_time`にしておく必要があります。

`clock_cast()`で行う変換は異なる時計型間の時刻型（`time_point`型）の変換であり、それに対して`time_point_cast()`の行う変換は、同じ時計型についての時刻型の間での、精度の変換（特に分解能が落ちる方向の変換）です。どちらも同じ`time_point`型間の変換ではありますが、役割が異なるので使い分けを覚えておく必要があります。

### `time_point/duration`の出力サポート

C++17までは、`<chrono>`に用意されている時間関連の型は標準ストリームに直接出力できず、`duration`の値は`.count()`によって整数値を取得してから出力するしかなく、`time_point`の値は一度`time_t`に変換してから出力するか、`.time_since_epoch()`で`duration`にしてから`.count()`を出力する必要がありました。また、`time_t`を経由しない方法では単なる整数値としてしか出力されず、自分でフォーマットを整える必要がありました。

最初の例でさらっと使っていましたが、C++20からは`time_point/duration`の値を標準ストリームに直接出力する事ができるようになります。しかも、最低限ながらフォーマットされて出力されます。

```cpp
#include <chrono>

// time_point型の出力
template<typename C>
void print_time() {
  std::cout << C::now() << "\n";
}

int main() {
  print_time<system_clock>();
  print_time<utc_clock>();
  print_time<tai_clock>();
  print_time<gps_clock>();
  print_time<file_clock>();
}
```

この出力は例えば次のようになります

```
2022-06-09 15:50:24.1013040
2022-06-09 15:50:24.1055415
2022-06-09 15:51:01.1419987
2022-06-09 15:50:42.1423285
2022-06-09 15:50:24.1426693
```

正確には、`time_point`型に直接`<<`が提供されるのではなく、`sys_time`や`utc_time`などの時計型に関連づけられたエイリアス毎に`<<`が用意されます。とはいえ、`Clock::now()`で取得した結果を出力する分には困ることはないでしょう。

`duration`型については直接`<<`が提供されるため、基本的にはあらゆる`duration`値が出力可能になります。

```cpp
#include <chrono>

int main() {
  std::cout << system_clock::now().time_since_epoch() << "\n";
  std::cout << 1s << "\n";
  std::cout << 1s + 500ms << "\n";
  std::cout << 1000us << "\n";
  std::cout << 10ns << "\n";
}
```

この出力は例えば次のようになります

```
16547482878527625[1/10000000]s
1s
1500ms
1000us
10ns
```

このように、`duration`型の出力ではその精度に応じたサフィックスが付加されます。精度があらかじめ定められた単位（基本的には10^3ごと）で表せない場合、一番上の例のように有理数値で示されます。

### うるう秒の処理

`utc_clock`で取得できる時刻にはうるう秒が含まれています。遠い過去や未来の時刻を計算する場合など、うるう秒が何秒（何回）挿入されているのかを取得したくなる事もあるでしょう。`get_leap_second_info()`によって、`utc_time`の値からうるう秒の秒数を取得できます。

```cpp
#include <chrono>

void now() {
  utc_time ut = utc_clock::now();

  // 戻り値はutがちょうどうるう秒であるか否かとうるう秒の秒数
  auto [is_leap_sec, count] = get_leap_second_info(ut);
  // is_leap_sec : false
  // count : 27 [s]
}

void leap() {
  // UTC時刻で2017年1月1日を取得（本書執筆時点で最後にうるう秒が追加された時）
  sys_days ymd = 2017y/1/1; // ???
  auto ut = clock_cast<utc_clock>(ymd) + 0h + 0min - 1s;

  auto [is_leap_sec, count] = get_leap_second_info(ut);
  // is_leap_sec : true
  // count : 27 [s]
}
```

`get_leap_second_info()`の戻り値が2つのメンバを持つ集成体型で、1つ目のメンバ（`is_leap_sec`）はその時刻がうるう秒であるかどうかを`bool`値で返し、2つ目のメンバはその時刻までに加算されているうるう秒の秒数を（`seconds`型で）返します。1つ目のメンバ（`is_leap_sec`）を用いると閏秒の検出ができるわけですが、これが`true`となるケースは非常に稀です。

`leap()`では本書執筆時点（2022年6月くらい）で最後にうるう秒が追加されたときの時刻を求めて、それを`get_leap_second_info()`に渡しています。2017年1月1日0時0分0秒を2回カウントすることでうるう秒は挿入されており、上記のように時刻を作ると非うるう秒の方（2回目）の2017年1月1日0時0分0秒が得られます。そのため、その1秒前（これも2017年1月1日0時0分0秒）がうるう秒になります。

なお、ここでやっている`2017y/1/1`のような日付の作り方は、次の節の主題であるカレンダー機能を使用しています。

## カレンダー

カレンダー、すなわち任意の日付を表現するために、カレンダーリテラル（年 : `y`, 日 : `d`）と`/`による結合が用意されています。

これらのリテラルは`h, min, s`などの`duration`を生成するリテラルと同様にライブラリ定義のユーザー定義リテラルであり、`std::chrono::chrono_literals`を`using namespace`しておくことによっても使用可能になります。

```cpp
#include <chrono>

int main() {
  // 時間単位としての日、年
  auto d = 7d;    // 7日
  auto y = 2022y; // 2022年

  // 日付
  auto ym = 1179y/9;        // 1179年9月
  auto ymd = 1181y/3/27;    // 1181年3月27日
  auto ymdr = 25d/4/1185;   // 1185年4月25日
  auto ymdr2 = 1/25d/1214;  // 1214年1月25日
}
```

これらのリテラルは整数値に対して作用して、リテラルの結果として得られる値は`y`なら年`d`なら日の単位による時間を表す値であって、まだ日付ではありません。整数`n`に対する`y, d`リテラルの結果は、`n`年や`n`日を表していますが、これは`duration`型（時間間隔を表す型）ではなく、それに素直に変換できるわけでもありません。

日付を作るには、年（と月）と日の値を`/`で結合して表現します。`/`で結合する値はどれかが`y, d`リテラルによる値である必要がありますが、全部がそうである必要はありません。よく使いそうなのは`yyyy/mm/dd`の形式で、この場合は年の値だけ`y`リテラルがあればよいでしょう。それ以外では、`dd/mm/yyyy`や`mm/dd/yyyy`等の順番でも結合することができますが、この場合は順番が曖昧になりやすいのと演算子の結合順序によって意図通りにならない可能性があるので`y`と`d`は両方つけておいた方がよいかもしれません。

```cpp
#include <chrono>

int main() {
  // どこが日でどこが年かはっきりさせておく
  auto ymdr = 25d/4/1185y;   // 1185年4月25日
  auto ymdr2 = 1/25d/1214y;  // 1214年1月25日

  // 意図通りにならない例（コンパイルエラーは出ない
  auto ng1 = 1181/3/27d; // 1181/3 -> int が先に計算される
  // これはコンパイルエラー
  auto ng2 = 25/4/1185y;
}
```

`/`は左結合なので、3つ以上連なっているときは左側のものから計算されます。その場合、一番左側の`/`のオペランドのどちらかが`y, d`リテラルを含むようにしておかないとそれは普通の整数演算となりカレンダー計算とは異なるため、そのような結果と`y, d`リテラルを結合してもおそらく意図しない値になってしまいます。`d`リテラルの場合は`mm/dd`の形の結合をサポートするために整数値との結合ができるのでこうしたエラーが起こりえます。`y`リテラルの場合は`mm/yyyy`のような結合は考慮されていないため、そのような結合は常にコンパイルエラーとなります。

月は時間単位としてしっかりしたものではない（1月の日数が固定ではない）ためか月のリテラルは用意されていませんが、その代わりに月を表すカレンダー定数が用意されています。そして、この値は日/年の値と`/`によって結合することができます。

|定数|意味|
|---|---|
|`January`|1月|
|`February`|2月|
|`March`|3月|
|`April`|4月|
|`May`|5月|
|`June`|6月|
|`July`|7月|
|`August`|8月|
|`September`|9月|
|`October`|10月|
|`November`|11月|
|`December`|12月|

```cpp
#include <chrono>

int main() {
  auto m = January;            // 1月
  auto md = July/18;           // 7月18日
  auto ym = 2022y/August;      // 2022年8月
  auto ymd = 2023y/January/1;  // 2023年1月1日
}
```

注意点として、これら月の定数と年を結合する際には年の値に`y`リテラルが必須です。これは、`dd/mm`の形の結合時に左辺に数字がくる場合に日と年の区別が曖昧になってしまうためで、その混同を防ぐためにコンパイルエラーとして警告されます。この場合、同様に`dd/mm`の`dd`の値にも`d`リテラルが必要です。

```cpp
int main() {
  auto md = July/18;          // 7月18日
  auto dm = 18d/July;         // 7月18日
  auto dm2 = 18/July;         // ng
  auto ym = 2022/August;      // ng
  auto ymd = 2023/January/1;  // ng
}
```

カレンダー定数にはもう一つ、週の曜日を表すものも用意されています。

|定数|意味|
|---|---|
|`Sunday`|日曜日|
|`Monday`|月曜日|
|`Tuesday`|火曜日|
|`Wednesday`|水曜日|
|`Thursday`|木曜日|
|`Friday`|金曜日|
|`Saturday`|土曜日|

これらは直接他の日付と結合できません（1月に同じ曜日が4回はあるため）ので、これ単体で使用することはあまりなさそうです。その代わり、月の`n`回目の曜日の日付を指定して取得する`weekday_indexed`クラスとそれを簡易取得するための`operator[]`が用意されています。

```cpp
#include <chrono>

int main() {
  auto sea = July/weekday_indexed{Monday, 3}; // 7月の第3月曜
  auto keiro = September/Monday[3];           // 9月の第3月曜
  auto sports = 2022y/October/Monday[2];      // 2022年10月の第2月曜
  auto seizin = 2023y/1/Monday[2];            // 2023年1月の第2月曜
}
```

月のカレンダー定数に対しての`[n]`は「ある月の`n`回目のその曜日」を表す`weekday_indexed`型の値を返します。あるいは、`weekday_indexed`型のコンストラクタに曜日定数と`n`を渡して構築することもできます。なお、`n`は`[1, 5]`の範囲の整数値である必要があり、ある月の最初の週の曜日は`n = 1`になります。

`weekday_indexed`型の値はまだ特定の日付を表す値ではなく、月および年と結合してもまだ日付にはなりません。月及び年と結合したうえで`sys_days`（日単位のシステム時刻）に変換することで日付としての値を取得できます。なお、この変換は暗黙変換が用意されています。

```cpp
#include <chrono>

int main() {
  sys_days sea = 2020y/July/weekday_indexed{Monday, 3}; // 7月の第3月曜
  sys_days keiro = 2021y/September/Monday[3];           // 9月の第3月曜
  sys_days sports = 2022y/October/Monday[2];      // 2022年10月の第2月曜
  sys_days seizin = 2023y/1/Monday[2];            // 2023年1月の第2月曜
  
  std::cout << sea << "\n";
  std::cout << keiro << "\n";
  std::cout << sports << "\n";
  std::cout << seizin << "\n";
}
```

この出力は次のようになります

```
2020-07-20
2021-09-20
2022-10-10
2023-01-09
```

注意ですが、`yyyy/mm/Monday[n]`のように結合する時に`yyyy`の値を`y`リテラルを用いない普通の数字にしてしまってもコンパイルエラーは出ませんが、意図しない値になります。これは、月リテラルが存在せず、かつ`weekday_indexed`型の値は月を指定する整数値と結合できるために起きています。やはり、`y, d`リテラルはなるべく省略しないほうが良いでしょう。

```cpp
// 有効ではない値になる
auto seizin = 2023/1/Monday[2]; // 2023/1 -> int が先に計算される
```

### 最後の日を表す定数

2月の最終日やある月の最後の日曜日などの日単位の最後（上限）が必ずしもはっきりしていない日付を指定したいとき、`last`定数を用いると文脈に応じた最後の日という形で指定することができます。

```cpp
#include <chrono>

int main() {
  // 全て12月の最終月曜日を表す
  auto lw1 = December/Monday[last];
  auto lw2 = Monday[last]/December;
  auto lw3 = 12/Monday[last];
  auto lw4 = Monday[last]/12;

  // 全て2022年12月の最終金曜日を表す
  auto pf1 = 2022y/December/Friday[last];
  auto pf2 = 2022y/12/Friday[last];
  auto pf3 = Friday[last]/12/2022y;
  auto pf4 = 12/Friday[last]/2022y;

  // 全て2月の最終日を表す
  auto ld1 = February/last; 
  auto ld2 = last/2;     
  auto ld3 = last/2;
  auto ld4 = 2/last;

  // 全て2024年2月の最終日を表す
  auto ld5 = 2024y/February/last;
  auto ld6 = 2024y/2/last;
  auto ld7 = last/2/2024y;
  auto ld7 = 2/last/2024y;
}
```

この`last`という値は日単位を表す値であるので、日を指定する値（`d`リテラル）として結合を行うことができます。

年単位は現在のところ終わりがなく、月単位の最後は12月で固定なので、`last`によって指定できる最後というのは常に日単位になります。また、最初の日というのは常に`1`によって指定可能であるため、`first`のような定数は用意されていません。

`last`による日付についても`weekday_indexed`と同様に、月及び年と結合したうえで`sys_days`に変換することで日付の値を取得できます。

```cpp
#include <chrono>

int main() {
  std::cout << sys_days{2022y/December/Friday[last]} << "\n";
  std::cout << sys_days{2023y/February/last} << "\n";
  std::cout << sys_days{2024y/February/last} << "\n";
}
```

この出力は次のようになります。

```
2022-12-30
2023-02-28
2024-02-29
```

### カレンダー型

ここまで紹介してきたカレンダー型とカレンダー定数による日付の表現について、その結果の値にのみ着目して型については触れないできました。これらの値は、その生成のされ方によって細かく型付けされており、それによって日付の表現と構文との対応を厳密に管理するとともに、日付形式に沿った結合の制御を行っています。

まず、`y, d`リテラルやカレンダー定数から生成される、単一の期間単位を表す値は次のような型によって表現されています。

|生成方法|型名|
|---|---|
|`d`リテラル|`day`|
|`y`リテラル|`year`|
|曜日定数|`weekday`|
|曜日定数`[n]`|`weekday_indexed`|
|月定数|`month`|  

そして、これらの値及び整数値の間で`/`によって結合したときも、その結合の仕方（引数型）によって細かく異なる中間結果の型が生成されます。

|型名|意味|生成される例|
|---|---|---|
|`weekday_last`|最後の曜日|`Monday[last]`|
|`month_day`|月 + 日|`September/11d`|
|`month_day_last`|月 + 最後の日|`February/last`|
|`month_weekday`|月 + `n`回目の曜日|`October/Monday[2]`|
|`month_weekday_last`|月 + 最後の曜日|`October/Monday[last]`|
|`year_month`|年 + 月|`2021y/11`|

1度の`/`の結合だけでは完全な日付とはなっておらず、これらの中間表現の値が生成されています。欠けている情報とさらに結合させることで完全な日付を表す値になります。

|型名|意味|生成される例|
|---|---|---|
|`year_month_day`|年月日|`1181y/3/27`|
|`year_month_day_last`|年月とその月の最終日|`2022y/12/last`|
|`year_month_weekday`|年月とその月の`n`回目の曜日|`2022y/October/Monday[2]`|
|`year_month_weekday_last`|年月とその月の最後の曜日|`2022y/October/Monday[last]`|

この4つの型は、年月日の単位が全て揃って完成された日付を表す型で、特定の日付の情報を保持しています。そのため、この4つの型は`sys_days`へと暗黙変換することができます（他のカレンダー型はできません）。

```cpp
#include <chrono>

int main() {
  // 日付型 -> sys_days
  sys_days t1 = 1181y/3/27;     // year_month_day
  sys_days t2 = 2022y/12/last;  // year_month_day_last
  sys_days t3 = 2022y/October/Monday[2];    // year_month_weekday
  sys_days t4 = 2022y/October/Monday[last]; // year_month_weekday_last
}
```

また、中間表現あるいは完全な日付を表すカレンダー型では、一貫した操作によってその一部分の値（`/`による結合前のそれぞれの値）の取得を行うことができます。

|関数名|戻り値型|意味|
|---|---|---|
|`.weekday()`|`weekday`|曜日の部分の値を取得|
|`.weekday_indexed()`|`weekday_indexed`|`n`回目の曜日の部分の値を取得|
|`.weekday_last()`|`weekday_last`|月の最後の曜日の部分の値を取得|
|`.month_day_last()`|`month_day_last`|月の最後の日の部分の値を取得|
|`.day()`|`day`|日の部分の値を取得|
|`.month()`|`month`|月の部分の値を取得|
|`.year()`|`year`|年の部分の値を取得|

```cpp
void obs_ymd(year_month_day ymd) {
  year y  = ymd.year(); 
  month m = ymd.month(); 
  day d   = ymd.day();

  weekday w = ymd.weekday();  // ng、曜日情報を持たない
}

void obs_ymdl(year_month_day_last ymd) {
  year y  = ymd.year(); 
  month m = ymd.month(); 
  day d   = ymd.day();
  month_day_last mdl = ymd.month_day_last();

  weekday w = ymd.weekday();  // ng、曜日情報を持たない
}

void obs_ymw(year_month_weekday ymd) {
  year y    = ymd.year(); 
  month m   = ymd.month(); 
  day d     = ymd.day();
  weekday w = ymd.weekday();
  weekday_indexed wi = ymd.weekday_indexed();

  month_day_last mdl = ymd.month_day_last();  // ng、最後の日の情報を持たない
}

void obs_ymwl(year_month_weekday_last ymd) {
  year y    = ymd.year(); 
  month m   = ymd.month(); 
  day d     = ymd.day();
  weekday w = ymd.weekday();
  weekday_last wl = ymd.weekday_last();

  month_day_last mdl = ymd.month_day_last();  // ng、最後の日の情報を持たない
}
```

これらの関数は、カレンダー型の保持する情報によって用意されていない場合もありますが、用意されている場合は同じ名前で取得できます。どのような情報を保持しているか（どの関数が使用可能か）は、その型名から把握することができます。

これらの操作も含めて、カレンダー型とその操作はほぼ`constexpr`対応されているため定数式でも使用可能です。また、動的な日付を表現したい場合はこれらカレンダー型（特に`year, month, day`型）のコンストラクタに月や日を表す整数値を渡して構築することで実行時の値として取得でき、以降は`/`を同じように使用することができます。

### 日付形式と`/`

`/`による日付の表現は複雑に見えますが、基本的には3種類の日付形式に沿った結合しかサポートしていません。

|`/`の結合順|日付形式|
|---|---|
|`yyyy/mm/dd`|年月日|
|`mm/dd/yyyy`|月日年|
|`dd/mm/yyyy`|日月年|

`yyyy`は年単位を表す値、`mm`は月単位を指定する値、`dd`は日単位を表す値です。これ以外の形式（日年月とか年を最後2桁で表示とか）はサポートしておらず、これに該当しない結合は基本的にはできないようになっています。

`yyyy`の部分には整数値もしくは`year`型の値（`y`リテラル）のみが使用可能で、`mm`の部分には整数値もしく`month`型の値（月定数）のみが使用可能です。一方、`dd`の部分の値には整数値と`day`の値（`d`リテラル）の他に`weekday, weekday_indexed`の値（曜日定数と曜日定数`[n]`）及び`weekday_last`の値（曜日定数`[last]`）と`last`も使用することができます。

また、結合に使用する`/`は言語組み込みの演算子ではなく演算子オーバーロードによるものです。従って、`/`は通常と同じく左結合であり、左側と真ん中の引数(年月日なら`yyyy, mm`)の両方が単なる整数値（`int`型）だと使用される`/`は整数演算を行ってしまうため、カレンダーを表現するものではなくなってしまいます。

これは、年が右側に来る形式（月日年と日月年）だと、年の値の左辺に整数値が来ないことがわかる（`month_day`や`month_weekday`とその`last`版だけを受ければいい）ためコンパイルエラーとなりますが、年月日形式では日の値は整数値を左辺に取れる（月日年と日月年の形式では、月の値と結合する必要があることから）ため、コンパイルエラーとなりません。

```cpp
template<typename Day>
  requires std::same_as<Day, day> or
           std::same_as<Day, last_spec> or
           std::same_as<Day, weekday_indexed> or
           std::same_as<Day, weekday_last>
void amb(Day dd) {
  // エラーにはならないが意図通りでない 
  auto date1 = 2022/01/dd;  // ok、有効ではない未規定の値
  auto date2 = 2022/01/13;  // ok、整数演算(155)

  // yyyy/mmとdd/mmの混同を防ぐためにエラーとなる
  auto date3 = 2022/August/1;   // ng
  auto date4 = 1/August/2022y;  // ng
}
```

これらのことから、数値によって年を指定するには常に`y`リテラルによって行うことが推奨されます（それによって、`date1~3`は意図通りになる）。また、日を指定する場合もなるべく`d`リテラルを使ったほうが良いでしょう（それによって、`date4`が意図通りになる）。というか、ほとんどの場合に単なる整数値を指定すべきではないかもしれません。

### 時刻との結合

`year_month_day`などの完全な日付を表す型の値でもその精度は日単位で、時刻の情報を持っていません。カレンダー型は時刻を扱うものではなく、`time_point`や`duration`の値と直接的な結合を行えません。日付に時刻の情報も含めるには`time_point`型の値に変換したうえで、`h, min, s`リテラルなどを用いて時刻を追加する必要があります。

日付を表すカレンダー型から`time_point`へは、前述のように`sys_days`へ変換することができます。`sys_days`の値は`system_clock`のタイムポイント値であり、ここからは`clock_cast`によって`utc_clock`等別の時計型の`time_point`へと変換することができます。

一度`time_point`値を得てしまえば、そこに`+`や`-`によって`duration`値を足し引きすることができます。そしてその結果は必要な精度を指定された`time_point`の値になります。

```cpp
#include <chrono>

int main() {
  // 日付をシステム時刻に変換
  sys_days date = 2022y/August/13d;

  // システム時刻をUTC時刻に変換
  auto t0 = clock_cast<utc_clock>(date);

  // 時間を追加
  auto t1 = t0 + 10h;

  // 分を追加
  auto t2 = t1 + 30min;

  // 秒を追加
  auto t3 = t2 + 30s;
  
  // ミリ秒を追加
  auto t4 = t3 + 150ms;
  
  std::cout << date << "\n";
  std::cout << t0 << "\n";
  std::cout << t1 << "\n";
  std::cout << t2 << "\n";
  std::cout << t3 << "\n";
  std::cout << t4 << "\n";
}
```
出力例
```
2022-08-13
2022-08-13 00:00:00
2022-08-13 10:00:00
2022-08-13 10:30:00
2022-08-13 10:30:30
2022-08-13 10:30:30.150
```

`sys_days`はすでに日単位の`time_point`なので、システム時刻で十分な場合は`clock_cast`の必要はありません。

### 時刻計算とカレンダー計算

C++20からは、日週月年の単位の`duration`型エイリアス、`days, weeks, years, months`が用意されており、これを`time_point`値に足し引きすることで時刻を進める/戻す操作を行えます。ただし、これらの値を生成するためのリテラルは用意されていないため、コンストラクタに数値を渡して構築する必要があります。

```cpp
#include <chrono>

int main() {
  sys_days date = 2022y/August/13d;

  std::cout << date + days{1} << "\n";
  std::cout << date - weeks{2} << "\n";
  std::cout << date + years{5} << "\n";
  std::cout << date + months{6} << "\n";
}
```
出力例
```
2022-08-14
2022-07-30
2027-08-13 05:06:00
2023-02-11 14:54:36
```

ただしこれは時刻計算であり、この年月の扱いはカレンダー的な扱いではなくあくまで時間間隔としてのもので、`years`の精度は364.2425[日]×24[時間]×3600[秒]であり、`months`はそれを12で割った精度を持ちます。したがって、カレンダー型の扱う日付のように暦に沿ったものではなく、年月の1間隔はより小さい単位の端数を含んでいます。

暦の上での日付計算を行う場合は、カレンダー型の値に対してこれらの`duration`値を足し引きすることで行います。このカレンダー計算においては整数値を使用することができず、一部のカレンダー型（主に中間表現の型）ではカレンダー計算を行うことができません。使用できる演算子は`+ - += -=`（全て2項演算）の4つで、型によって足し引きできる値が制限されています。

|カレンダー型＼`duration`型|`days`|`months`|`years`|整数型|
|---|---|---|---|---|
|`day`|⭕️|❌|❌|❌|
|`month`|❌|⭕️|❌|❌|
|`year`|❌|❌|⭕️|❌|
|`year_month`|❌|⭕️|⭕️|❌|
|`year_month_day`|❌|⭕️|⭕️|❌|
|`year_month_day_last`|❌|⭕️|⭕️|❌|
|`year_month_weekday`|❌|⭕️|⭕️|❌|
|`year_month_weekday_last`|❌|⭕️|⭕️|❌|

この表の⭕️になってるところでは、その行要素（カレンダー型）と列要素（`duration`型）の間で`+ - += -=`によるカレンダー計算を行うことができることを表します。ただし、`+= -=`の2つは左辺にカレンダー型、右辺に`duration`型がこなければなりません。

```cpp
#include <chrono>

int main() {
  1d + days{5};          // +5日
  January + months{5};   // +5ヶ月
  2019y + years{4};      // +4年
  2022y/8/1d - years{2}; // -2年

  // 計算できない例
  1d + 1;   // day + int
  1d + 1d;  // day + day
  December + 1;       // month + int
  January + December; // month + month
  2022y + 1;  // year + int
  2022y + 1y; // year + year
  2022y/1/31d + 1;        // year_month_day + int
  2022y/1/31d + days{7};  // year_month_day + days
}
```

カレンダー型`day`に対して`duration`型`days`のように末尾に`s`がつく命名になっており、カレンダー計算に使用できるのは`duration`型の値であって、カレンダー型同士の間で足し引きすることはできません。

また、`day, month, year`の3つの型はインクリメント/デクリメントによっても日付を進める/戻すことができます。

```cpp
#include <chrono>

int main() {
  day   dd = 1d;
  month mm = January;
  year  yy = 2022y;

  ++dd; // 2日
  --mm; // 12月
  ++yy; // 2023年
}
```

このカレンダー計算においては、計算結果が不正な日付となることがあります。そうなった時でも例外を投げたりはせず、カレンダー型は不正な日付の値を保持します。その後、`sys_days`（時刻）への変換の際に暦に従った日付への調整が行われます。

```cpp
#include <chrono>

int main() {
  auto ymd1 = 2021y/1/31d;
  auto ymd2 = ymd1 + months{1};

  std::cout << ymd2 << "\n";
  std::cout << sys_days{ymd2} << "\n";
}
```
出力例
```
2021-02-31 is not a valid date
2021-03-03
```

## タイムゾーン

時間を扱う上で、日付と時刻に続いて重要な要素がタイムゾーンです。タイムゾーンは、ある地域の時間がUTCを基準として何時間進んでいる/遅れているかを表すものです。例えば、日本標準時はUTC+9のタイムゾーンに属しています。

ログ出力などに使用する分にはUTCやシステム時刻で事足りますが、何かしら時刻を扱うアプリケーションを作成する場合は地域のローカル時間を使用するシーンが多くあるでしょう。その場合、UTC時刻とタイムゾーン情報によって地域のローカル時間を計算することができ、C++20 `<chrono>`ライブラリにおいてもタイムゾーンによる地域のローカル時間の取り扱いがサポートされています。

現在の（そのシステムローカルの）タイムゾーン情報を取得するには、`current_zone()`を使用します。この戻り値型は`time_zone`オブジェクトへの`const`ポインタです。

```cpp
#include <chrono>

int main() {
  // ローカルタイムゾーンの情報を取得
  const time_zone* tz = current_zone();

  // タイムゾーン名を出力
  std::cout << tz->name() << "\n";

  auto now = system_clock::now();

  // ローカルタイムゾーンの時刻へ変換
  local_time to_jst = tz->to_local(now);
  // ローカル時刻からシステム時刻へ変換
  sys_time from_local = tz->to_sys(to_jst);

  std::cout << now << "\n";
  std::cout << to_jst << "\n";
  std::cout << from_local << "\n";
}
```
出力例
```
Asia/Tokyo
2022-06-16 16:44:15.8155567
2022-06-17 01:44:15.8155567
2022-06-16 16:44:15.8155567
```

`current_zone()`で得られるのは、そのシステムのタイムゾーン設定を参照しているグローバルな`time_zone`オブジェクトへのポインタで、`time_zone`のオブジェクトは自分で構築することはできません。

`time_zone`オブジェクトからは、`.name()`によってそのタイムゾーンの名前文字列（*地域/地名*の形式）を取得でき、`.to_local()`によってシステム時刻にタイムゾーンを適用したローカル時間へ、`.to_sys()`でその逆の変換を行えます。

現在のシステムローカルのタイムゾーンだけではなく、任意の地域のタイムゾーンを取得するために`locate_zone()`が用意されています。引数にタイムゾーン名（*地域/地名*の形式）を指定し、戻り値は`time_zone`オブジェクトへのポインタです。引数に指定する名前とは、`time_zone::name()`で得られるものと同じものです。

```cpp
#include <chrono>

int main() {
  const time_zone* jst = locate_zone("Asia/Tokyo");       // 日本標準時
  const time_zone* edt = locate_zone("America/New_York"); // 東海岸時間

  auto now = system_clock::now();

  std::cout << now << "\n";
  std::cout << jst->to_local(now) << "\n";
  std::cout << edt->to_local(now) << "\n";
}
```
```
2022-06-16 17:16:00.3730206
2022-06-17 02:16:00.3730206
2022-06-16 13:16:00.3730206
```

タイムゾーンは地域名ではなく標準時名を指定することによっても取得できます。

```cpp
#include <chrono>

int main() {
  const time_zone* jst = locate_zone("JST");  // 日本標準時
  const time_zone* gmt = locate_zone("GMT");  // グリニッジ標準時
  const time_zone* utc = locate_zone("UTC");  // 協定世界時

  auto now = system_clock::now();

  std::cout << now << "\n";
  std::cout << jst->to_local(now) << "\n";
  std::cout << gmt->to_local(now) << "\n";
  std::cout << utc->to_local(now) << "\n";
}
```
```
2022-06-16 17:18:01.2464961
2022-06-17 02:18:01.2464961
2022-06-16 17:18:01.2464961
2022-06-16 17:18:01.2464961
```

これらのタイムゾーン名の表記は、IANA Time Zone Databaseで定義されているものです。

### ローカル時間

ここにきて、これまで見ないふりをしていたローカル時間（`local_time`型）が出てきました。このローカル時間とは、システムローカルあるいは地域のローカルタイムゾーンにおける時間のこと**ではありません**。

`<chrono>`におけるローカル時間とは、時計型に依存せずエポックやその意味論が任意の時間を表すものです。`sys_time`や`tai_time`などが、参照する時計もその値（時刻）の意味もコンパイル時に確定されている時間表現であるのに対して、ローカル時間（`local_time`）はユーザーがその意味を実行時に自由に決めることのできる時間表現であるといえます。

例えば、システム時刻（UTC）にタイムゾーンを適用した結果とはすでにUTCではなく、`time_point`によってそれを表現しようとするとタイムゾーン毎に時計型と`time_point`エイリアスの定義を行わなければならなくなります。そこで、`local_time`を用いることで、システム時刻に任意のタイムゾーンを適用した時刻というものを自由に表すことができ、これは実行時に意味論を決められるローカル時間の有効な活用例になっています。

ダミーの時計型`local_t`はローカル時間のための`time_point`エイリアステンプレートである`local_time`の時計型のテンプレート引数を埋めるためのもので、それ以外の役割はありません。ローカル時間の扱いで使うのは、`local_time`型（エイリアステンプレート）とその精度指定されたエイリアス、`local_seconds`と`local_days`です。

ローカル時間には時計型がなく、時計型の`now()`のような現在時刻を取得する方法はありません。先ほど見たように`time_zone::to_local()`によってシステム時刻から変換して取得するほか、完全な日付を表すカレンダー型（`sys_days`への変換が可能な4つ）から`local_days`へ変換することで取得することができます。ただし、日付型から`local_days`への変換は明示的な変換であり暗黙変換ではありません。

```cpp
#include <chrono>

int main() {
  const time_zone* jst = locate_zone("JST");
  auto now = system_clock::now();

  // システム時刻にタイムゾーンを適用した結果
  local_time to_jst = jst->to_local(now);

  // 日付型からの変換（explicit）
  auto date = local_days{2022y/August/13d};

  // local_daysに時刻を追加
  local_seconds time = date + 16h + 0min + 0s;
}
```

前述のように、ローカル時間（`local_time`）は任意の時間を表現するためのものであり、タイムゾーン変換に特化したものではありません。したがって、システム時刻にタイムゾーンを適用してローカル時間に変換すると、結果の`local_time`値からはタイムゾーンの情報が失われています。

### `zoned_time`

`time_zone::to_local()`はタイムゾーンを適用した結果を簡易に受け取るためのもので、タイムゾーン情報を保持しておくことができません。また、`time_point`型もタイムゾーンを受け取れるようにはなっていません。タイムゾーンを考慮した時刻の表現のためには`time_point`値とタイムゾーン情報のペアが必要で、`<chrono>`ではそのための型として`zoned_time`を用意しています。

`zoned_time`はそのコンストラクタでタイムゾーンの指定あるいは`time_zone`オブジェクトのポインタとシステム時刻（`sys_time`、UTC時刻）を受け取って保持することで、そのタイムゾーンにおける時間軸上の一点（つまり`time_point`）を表現します。タイムゾーンを特に指定しない場合はUTCがデフォルトのタイムゾーンとして使用されます。

```cpp
#include <chrono>

int main() {
  // UTC時刻
  auto now = system_clock::now();

  zoned_time zt1{now};                      // UTCタイムゾーンの時刻
  zoned_time zt2{current_zone(), now};      // その環境のタイムゾーンの時刻
  zoned_time zt3{"Asia/Tokyo", now};        // JSTの時刻
  zoned_time zt3{"America/New_York", now};  // 東海岸時間の時刻
  zoned_time zt3{"GMT", now};               // グリニッジ標準時の時刻
}
```

あるいはローカル時間（`local_time`）から構築することもでき、この場合は指定したタイムゾーンによって（指定したタイムゾーンにおける時刻を示すとみなして）システム時間に変換されて保持されます。

```cpp
#include <chrono>

int main() {
  // 2022年8月13日16時0分0秒、ローカル時刻（時計やタイムゾーンが指定されていない）
  local_time lt = local_days{2022y/August/13d} + 16h + 0min + 0s;

  zoned_time zt1{lt};                      // UTC時刻として変換
  zoned_time zt2{current_zone(), lt};      // その環境のタイムゾーンを用いて変換
  zoned_time zt3{"Asia/Tokyo", lt};        // JST時刻として変換
  zoned_time zt3{"America/New_York", lt};  // 東海岸時間の時刻として変換
  zoned_time zt3{"GMT", lt};               // グリニッジ標準時の時刻として変換
}
```

`zoned_time`オブジェクトは暗黙変換あるいは変換関数（`.get_local_time()`/`.get_sys_time()`）によってローカル時間（`local_time`）とシステム時間（`sys_time`）に変換することができます。

```cpp
#include <chrono>

int main() {
  auto now = system_clock::now();
  zoned_time zt{current_zone(), now};

  // 暗黙変換
  std::cout << sys_time{zt} << "\n";
  std::cout << local_time{zt} << "\n";

  // 変換関数
  std::cout << zt.get_sys_time() << "\n";
  std::cout << zt.get_local_time() << "\n";
}
```
```
2022-06-19 17:51:59.6055270
2022-06-20 02:51:59.6055270
2022-06-19 17:51:59.6055270
2022-06-20 02:51:59.6055270
```

また、C++20からの他の`chrono`関連型と同様に、直接`<<`によってストリームに出力することができます。この場合は、コンストラクタで指定されたタイムゾーンを用いてローカル時間に変換するとともに、タイムゾーン名を添えて出力します。

```cpp
#include <chrono>

int main() {
  // UTC時刻
  auto now = system_clock::now();

  std::cout << zoned_time{now} << "\n";
  std::cout << zoned_time{current_zone(), now} << "\n";
  std::cout << zoned_time{"Asia/Tokyo", now} << "\n";
  std::cout << zoned_time{"America/New_York", now} << "\n";
  std::cout << zoned_time{"GMT", now} << "\n";
}
```
```
2022-06-19 17:52:45.1730584 UTC
2022-06-20 02:52:45.1730584 JST
2022-06-20 02:52:45.1730584 JST
2022-06-19 13:52:45.1730584 GMT-4
2022-06-19 17:52:45.1730584 GMT
```

また、`.get_time_zone()`によって保持している`time_zone`オブジェクトのポインタを取得できます。

```cpp
#include <chrono>

int main() {
  // UTC時刻
  auto now = system_clock::now();

  zoned_time zt1{now};                      // UTCタイムゾーンの時刻
  zoned_time zt2{current_zone(), now};      // その環境のタイムゾーンの時刻

  std::cout << zt1.get_time_zone()->name() << "\n";
  std::cout << zt2.get_time_zone()->name() << "\n";
}
```
```
Etc/UTC
Asia/Tokyo
```

ただし、`zoned_time`は`time_point`とは異なり時刻計算を行うことはできず、ほぼシステム時刻をタイムゾーンとともに持っておいて手軽に出力することしかできません。

### 夏時間

夏時間（サマータイム）を導入する地域では、夏時間の開始時に時計が1時間進められ、夏時間終了時に時計が1時間戻されます。これによって、これらのタイムゾーンではUTCへの変換時に存在しない時刻を指してしまったり、変換先が曖昧になることがあります。

```cpp
#include <chrono>

// 存在しない日付時刻からの変換
void nonexist() noexcept(false) {
  const time_zone* edt = locate_zone("America/New_York"); // 東海岸時間

  // アメリカでは、3月第2週目の日曜2時から夏時間開始
  local_time lt = local_days{2022y/March/Sunday[2]} + 2h + 30min;

  // 夏時間開始日の2時〜3時の間の時刻は存在しない
  sys_time = edt->to_sys(lt);  // nonexistent_local_time例外
  // zoned_timeも同様
  zoned_time zt{edt, lt};      // nonexistent_local_time例外
}

// 曖昧な日付時刻への変換
void ambiguous() noexcept(false) {
  const time_zone* edt = locate_zone("America/New_York"); // 東海岸時間

  // アメリカでは、11月第1週目の日曜2時で夏時間終了
  local_time lt = local_days{2022y/November/Sunday[1]} + 1 h + 30min;

  // 夏時間終了日の1時〜2時の間の時刻は、一意のUTC時刻に定まらない
  sys_time st = edt->to_sys(lt);  // ambiguous_local_time例外
  // zoned_timeも同様
  zoned_time zt{edt, lt};         // ambiguous_local_time例外
}
```

例えばアメリカでは、毎年3月の第2日曜の2時から11月の第2日日曜の2時までの期間が夏時間で、問題となるのはこの開始と終了のタイミングです。

夏時間開始時には、その地域の標準時が1時間進められます。アメリカの場合、毎年3月の第2日曜の1時59分59秒の1秒後は、3時0分0秒になります。従って、夏時間開始日のアメリカのタイムゾーンでは、2時から2時59分59秒の間の時刻が存在しません。しかし、カレンダーと時刻の計算によってそのような時刻をローカル時間上で作り出すことはでき、そのような時刻からUTCへ夏時間のタイムゾーンによって変換しようとすると、存在しない時刻からの変換を行うことになり、この場合は`nonexistent_local_time`例外が投げられます。

夏時間終了時には、その地域の標準時が1時間戻されます。アメリカの場合、毎年11月の第1日曜の1時59分59秒の1秒後は、1時0分0秒になります。従って、夏時間終了日のアメリカのタイムゾーンでは、1時から1時59分59秒が2回繰り返され、その間の時刻は2つ存在します。そのような時刻を指すローカル時間値からUTCへ夏時間のタイムゾーンによって変換しようとすると変換先が2つあり曖昧となるため、この場合は`ambiguous_local_time`例外が投げられます。曖昧な場合とは例えばアメリカニューヨークの場合、EDT（東部夏時間）はUTC-4、EST（東部標準時）はUTC-5なので、11月の第2日曜（夏時間終了日）の1時30分の変換先はUTC 5時30分（夏時間中のEDT 1時30分）とUTC 6時30分（夏時間後のEST 1時30分）の2つとなります。

厄介なことに、夏時間は導入している国によって開始日や開始時間が異なっており、さらにある特定の地域ではUTCとのズレが30分だったりします。タイムゾーンを考慮することによって、UTCの時刻から地域のローカル時刻への変換は（夏時間に関わらず）常に一意的に行えますが、逆の変換はこのような問題が起こります。

ところで、曖昧となる変換（`ambiguous_local_time`例外）は曖昧さを解消できれば例外を投げる必要はないはずで、その解消ために`choose`という列挙型とそれを受け取るインターフェースが用意されています。この列挙型は`choose::earliest`（早い方の時刻を選択）と`choose::latest`（遅い方の時刻を選択）の2つの値が定義されていて、この値を変換時に指定することで変換先を1つに指定することができます。

```cpp
// 曖昧な日付時刻への変換
void ambiguous() noexcept {
  const time_zone* edt = locate_zone("America/New_York"); // 東海岸時間

  // アメリカでは、11月第1週目の日曜2時で夏時間終了
  local_time lt = local_days{2022y/November/Sunday[1]} + 1h + 30min;

  // 変換先を指定して変換、例外は投げられない
  sys_time t1 = edt->to_sys(lt, choose::earliest);
  sys_time t2 = edt->to_sys(lt, choose::latest);

  std::cout << t1 << "\n";
  std::cout << t2 << "\n";

  // zoned_timeも同様
  zoned_time zt1{edt, lt, choose::earliest};
  zoned_time zt2{edt, lt, choose::latest};

  std::cout << zt1.get_sys_time() << "\n";
  std::cout << zt2.get_sys_time() << "\n";
}
```
```
2022-11-06 05:30:00
2022-11-06 06:30:00
2022-11-06 05:30:00
2022-11-06 06:30:00
```

そして、このインターフェースは存在しない時刻からの変換（`nonexistent_local_time`例外）時にも例外を投げずに変換させることができ、この場合は常に夏時刻の開始時（アメリカの場合は3時ちょうど）に変換されるようです。

```cpp
// 存在しない日付時刻からの変換
void nonexist() noexcept {
  const time_zone* edt = locate_zone("America/New_York"); // 東海岸時間

  // アメリカでは、3月第2週目の日曜2時から夏時間開始
  local_time lt = local_days{2022y/March/Sunday[2]} + 2h + 30min;

  // 例外は投げられない（変換先は1つ）
  sys_time t1 = edt->to_sys(lt, choose::earliest);
  sys_time t2 = edt->to_sys(lt, choose::latest);

  std::cout << t1 << "\n";
  std::cout << t2 << "\n";

  // zoned_timeも同様
  zoned_time zt1{edt, lt, choose::earliest};
  zoned_time zt2{edt, lt, choose::latest};

  std::cout << zt1.get_sys_time() << "\n";
  std::cout << zt2.get_sys_time() << "\n";
}
```
```
2022-03-13 07:00:00
2022-03-13 07:00:00
2022-03-13 07:00:00
2022-03-13 07:00:00
```

## `std::format()`

`<chrono>`関連の値はほぼストリーム出力が可能となっているとともに、`std::format()`による文字列化もサポートされています。これは、`<chrono>`関連の型に対して`std::formatter`特殊化をあらかじめ用意する形でサポートされています。`<chrono>`関連の型のフォーマット構文は組み込み型のものをベースに制限と拡張を加えたもので、全体は次のようになっています

```
{ index : fill align width .precision locale(L) chrono-spec }
```

最後の*chrono-spec*オプションが`<chrono>`関連型のフォーマットのための専用構文でであり、それ以外のところはほぼ文字列型のフォーマットと同じ振る舞いをします。ただし、*precision*の指定は浮動小数点数型の表現による`duration`型でのみ有効で、それ以外の型に対してはコンパイルエラーとなります。

組み込み型のフォーマット構文では、置換フィールドを含めたフォーマット文字列全体で指定された引数列がどのように文字列化されるのかを指定していましたが、*chrono-spec*オプションでは1つの時間/カレンダー値（以下、`chorno`値）をどう文字列化するかを1つの置換フィールドに対して指定し、置換フィールド内で1つの`chorno`値に対するフォーマット指令を行います。そこでは、`%Y`や`%H`のようなフォーマット指定と任意の文字列によって`chorno`値に含まれる値を意味的に分割しつつフォーマットする事ができます。

```cpp
#include <chrono>

int main() {
  auto now = system_clock::now();

  std::cout << std::format("{}\n", now);
  std::cout << std::format("{:%Y/%m/%d}\n", now);
  std::cout << std::format("{:%Hh%Mm%Ss}\n", now);
}
```
```
2022-07-14 07:53:02.9982661
2022/07/14
07:53:02.9982661
```

すなわち*chrono-spec*オプションは、（制限された）組み込み型のフォーマット構文DSLの内部で、`chorno`値に対するフォーマットを指定するための別のDSLのようになっており、フォーマット文字列内の置換フィールドの中でさらにフォーマット指定を行うものです（`%Y`などは、置換フィールド内での置換フィールドとなっています）。

なお、*chrono-spec*の指定においては先頭の`:`は必須であり、これは出力文字列に含まれません。また、*chrono-spec*内の`%Y`等の指定以外の場所で使用可能な文字は`% { }`以外の文字です。

### 基本のフォーマット指定

*chrono-spec*で使用可能な`%`から始まるフォーマット指定は、POSIX標準にある関数`strftime()`で用いられるフォーマット文字列をベースとしたものです。そこそこ量が多いため、種類ごとにある程度のグループに分けて見ていくことにします。

1つ目は基本となる値の出力を指定するフォーマット指定です。

|フォーマット指定|意味|出力例|
|---|---|---|
|`%C`|100で切り上げた年（2桁の世紀）|`20`|
|`%Y`|4桁の年|`2022`|
|`%y`|年の後ろ2桁（`%C`からのオフセット）|`22`（=2022年）|
|`%m`|2桁の月|`06`|
|`%d`|2桁の日|`05`|
|`%e`|2桁の日（スペース埋め）|` 5`|
|`%H`|2桁の時（24時間時計）|`23`|
|`%I`|2桁の時（12時間時計）|`11`|
|`%M`|2桁の分|`15`|
|`%S`|2桁の秒（ミリ秒）|`59`, `59.285`|

これらの基本的なフォーマット指定だけを見ていてもわかりますが、大文字と小文字の間違いは大きな意味の違いを招きますので気をつけましょう。

すべての指定において、置換後の数値は10進数値であり出力桁数は固定となっています。出力後文字列が固定桁数に満たない場合は先頭が`0`で埋められます。`%e`はその場合に`0`の代わりにホワイトスペース1つで埋めるもので、`%S`はミリ秒以下の値は小数点以下の値として出力します。

```cpp
#include <chrono>

int main() {
  auto ymd = 2022y/07/05d;

  std::cout << std::format("{:%CC %y}\n", ymd);
  std::cout << std::format("|{:%d|%e}|\n", ymd);
  
  auto hsu = 23h + 7s + 1us;
  
  std::cout << std::format("{:%Hh = %Ih}\n", hsu);
  std::cout << std::format("{:%S}\n", hsu);

  std::cout << std::format("{:%Y}\n", 29537y);
}
```
```
20C 22
|05| 5|
23h = 11h
07.000001
```

また、年月日のオプションの桁数とは最小出力桁数の事でありこれらの桁数を超える値が出力できないわけではありません。一方、時刻出力のオプションは時間間隔ではなく時計に従った時刻を出力するオプションであるため、範囲外の時間を正常に出力できません。

```cpp
#include <chrono>

int main() {
  std::cout << std::format("{:%Y年}\n", 29537y);
  std::cout << std::format("{:%mヵ月}\n", month{36});
  std::cout << std::format("{:%d日}\n", 364d);
  std::cout << std::format("{:%H時間}\n", 48h);
  std::cout << std::format("{:%M分間}\n", 120min);
  std::cout << std::format("{:%S秒間}\n", 3600s);
}
```
```
29537年
36ヵ月
108日
00時間
00分間
00秒間
```

後ろ3つの出力結果はおそらく未規定あるいは実装定義です。

なお注意点ですが、*chrono-spec*オプションは文法的に`%`始まりのオプション指定から始まっていなければならず、他の文字を先頭に入れてしまうとエラーになります

```cpp
std::cout << std::format("{:|%d|%e|}\n", ymd);  // ng
                            ^
```

つまり、他のオプションがない場合、`:`の次の文字は`%`でなくてはなりません（かつ、有効なオプション指定である必要があります）。

### 複合フォーマット指定

時分秒の表記は、ISO 8601のようにある程度決まったパターンがあります。そのため、そのような固定パターンを読み取るための複合的なフォーマット指定が用意されています。

|フォーマット指定|等価な指定|受理する例|
|---|---|---|
|`%F`|`%Y-%m-%d`|`2019-05-29`|
|`%D`|`%m/%d/%y`|`05/29/2019`|
|`%T`|`%H:%M:%S`|`17:44:31`|
|`%R`|`%H:%M`|`17:44`|

### タイムゾーンの指定

文字列化されている日付時刻にはタイムゾーン情報が付加されていることもあるため、タイムゾーン情報を指定するフォーマットも用意されています。

|フォーマット指定|意味|受理する例|
|---|---|---|
|`%Z`|タイムゾーンの略称|`JST`|
|`%z`|UTCからのオフセット|`+0900`|
|`%Ez`, `%OZ`|`%z`の改良コマンド|`-04:30`|

### ロケール依存フォーマット指定

年や日、時分などの基本単位を指定するフォーマット指定では、頭に`O`または`E`を付加することでロケール依存のフォーマット指定になります。ロケール依存フォーマットにおいては、パース時にその時点の（ストリームに指定された）ロケールによって定義されている数値等の代替表現を考慮するようになります。

|フォーマット指定|意味|例（日本語ロケール）|
|---|---|---|
|`%EC`|期間（時代）名|`平成`|
|`%EY`|年|??|
|`%Oy`|年の後ろ2桁|??|
|`%Ey`|`%EC`からのオフセット|`14`|
|`%Om`|月|`12`|
|`%Od`|日（2桁0埋め）|`06`|
|`%Oe`|日（2桁スペース埋め）|` 6`|
|`%OH`|時（24時間時計）|`22`|
|`%OI`|時（12時間時計）|`9`|
|`%OM`|分|`35`|
|`%OS`|秒|`15`|
|`%A`|曜日名|`Sunday`, `日曜日`|
|`%a`|曜日の略称|`Sun`, `日`|
|`%B`|月名|`April`, `4月`|
|`%b`, `%h`|月の略称|`Apr`, `4月`|
|`%p`|午前・午後|`AM`, `PM`|

|複合フォーマット|意味|
|---|---|
|`%c`|日付と時刻|
|`%Ec`|日付と時刻の代替表現|
|`%r`|午前午後表示を含む12時間時計による時刻|
|`%x`|日付|
|`%X`|時刻|
|`%Ex`|日付（`%x`）の代替表現|
|`%EX`|時刻（`%X`）の代替表現|


### 週/曜日番号フォーマット指定

年内の何周目や何日目といった番号で日付を指定することがあるようで、それをパースするためのフォーマットが用意されています。

|フォーマット指定|意味|受理する例|
|---|---|---|
|`%U`|週番号|`15`|
|`%W`|週番号|`15`|
|`%w`|曜日番号|`6`（=土曜日）|
|`%j`|元旦からの日単位オフセット|`364`|
|`%OU`|ロケール依存週番号|`15`|
|`%OW`|ロケール依存週番号|`15`|
|`%Ow`|ロケール依存曜日番号|`6`|

週番号とは1年の最初の週を1としてその年の何週目かを指定するもので、曜日番号とは日曜を0として0〜6の数字でその週の曜日を指定するものです。

`%U`と`%W`の違いは基準となる最初の週をどう取るかにあります。`%U`は年内最初の日曜日を起点としてその週が`01`になり、`%W`は年内最初の月曜日を起点としてその週が`01`になります。どちらの場合もそれより前の週は`00`です。つまりは、1週間の始まりが日曜なのか月曜なのかの違いです。


### ISO 8601ベースフォーマット

ISO 8601は日付と時刻の表記についての国際標準規格で、そこでは前項の週番号などの少し異なる定義があります。そのため、それを読み取るためのフォーマット指定が用意されています。

|フォーマット指定|意味|例|
|---|---|---|
|`%G`|ISOベース年|`2022`|
|`%g`|ISOベース年の末尾2桁|`22`|
|`%V`|ISO週番号|`1`|
|`%u`|ISO曜日番号|`7`（=日曜日）|
|`%Ou`|ロケール依存ISO曜日番号|`7`|
|`%OV`|ロケール依存ISO週番号|`1`|

ISO 8601の週番号（`%V`）は、年内最初の木曜日を起点としてその週が`01`になり、それより前の日は前年の最終週となります（週番号`00`はない）。そして、ISO 8601において週は月曜始まりです。その結果、それより前の金土日曜日がグレゴリオ暦では新年初週であるにもかかわらず前年最終週として扱われたり、年内最初の木曜日と同週の月火水曜日がグレゴリオ暦では前年最終週なのに新年初週として扱われる、といったことが起きます。

ISOベース年（`%G`/`%g`）とはこの週の規定に沿った年であり、ISO曜日番号（`%u`）は月曜日が1で日曜日が7となる、1〜7の数字です。

### その他

日付時刻以外のところに関して、特殊な意味を持つフォーマット指定（と文字）が用意されています。

|フォーマット指定|意味|
|---|---|
|`%%`|`%`のエスケープ|
|`%n`|改行文字|
|`%t`|水平タブ|
|`%q`|`duration`のサフィックス|
|`%Q`|`duration`の数値|


## 日付時刻文字列のパース

文字列で表されている日付時刻をストリームからパースすることができます。パースに当たっては、`parse()`というマニピュレータにフォーマット指定と解析後の保存先を渡して、その結果に対して`>>`します。

```cpp
#include <chrono>

int main() {
  std::stringstream strm{};
  // パース対象文字列
  strm << "2022-04-01T12:01:45";

  // 秒単位システム時刻で受け取る
  sys_seconds st;

  // パース
  strm >> parse("%Y-%m-%dT%H:%M:%S", st);

  // エラーチェック
  if (strm) {
    std::cout << st << "\n";
  }
}
```

ここではサンプルコード表示のために入力ストリームに`std::stringstream`を使用していますが、`std::cin`のほか任意の`istream`を同じように使用することができます。

`parse()`の1つ目の引数にはパースするフォーマット指定文字列を渡し、2つ目の引数には結果を受け取る`time_point`値（の参照）を渡します。結果の受け取りは、`sys_time`等の`time_point`特殊化において行うことができるほか、一部のカレンダー型でも受け取ることができます。

```cpp
#include <chrono>

int main() {
  std::stringstream strm{};
  strm << "2022-04-01T12:01:45";

  // 年月日で受け取る
  year_month_day ymd;
  strm >> parse("%Y-%m-%dT", ymd);  // ok

  if (strm) {
    std::cout << ymd << "\n";
  }
}
```

`parse()`の結果受け取りに使える型の一覧

- `duration`
- `sys_time`
- `utc_time`
- `tai_time`
- `gps_time`
- `file_time`
- `local_time`
- `year`
- `month`
- `day`
- `weekday`
- `month_day`
- `year_month`
- `year_month_day`

注意ですが、ここまで`sys_time`などの`time_point`特殊化をさも完結した型名のように使用していましたが、これら6つは正確には精度（`duration`型）指定が必要なエイリアステンプレートです。従って、上記サンプルのようにデフォルト初期化で書こうとするとテンプレートパラメータが埋まっていないとしてエラーになるでしょう。これまでそこを無視してこられたのは、エイリアステンプレートでのクラステンプレートの実引数推定（C++20言語機能）の恩恵によるものです。

パースのエラーは、入力文字列から指定されたフォーマットで日付時刻を読み取れなかった場合と、日付時刻を読み取れたものの指定された出力先に出力できなかった場合に発生します。エラーが起きた時は入力のストリームに`std::ios_base::iostate::failbit`がセットされるため、ストリームオブジェクトの`opreator bool()`によってエラーの有無をチェックできます。しかし、どうしてエラーが起きたのかは知ることができません・・・

### 基本のフォーマット指定

パースのためのフォーマット指定はPOSIX標準にある関数`strptime()`で用いられるフォーマットをベースとしたものです。これは、先程の`std::format`による出力時フォーマット指定と同様の構文であり、意味もある程度対応しています。

|フォーマット指定|意味|受理する例|
|---|---|---|
|`%C`|世紀|`21`|
|`%Y`|年|`2022`|
|`%y`|年の後ろ2桁（`%C`からのオフセット）|`99`（=1999年）|
|`%m`|月|`06`|
|`%d`, `%e`|日|`28`|
|`%H`|時（24時間時計）|`23`|
|`%I`|時（12時間時計）|`11`|
|`%M`|分|`15`|
|`%S`|秒（ミリ秒）|`59`, `59.285`|

すべての場合において、日付時刻の指定は10進整数のみが考慮されます。

これらの基本的なフォーマット指定だけを見ていてもわかりますが、大文字と小文字の間違いは大きな意味の違いを招きますので気をつけましょう。

`%y`は年の末尾2桁で年を読み取るもので、デフォルトだと`[69, 99]`が`[1969, 1999]`年に、`[00, 68]`が`[2000, 2068]`年にマッピングされます。この時、`%C`によって世紀を指定するとその世紀からのオフセット（経過年）を読み取るようになります。

```cpp
#include <chrono>

int main() {
  std::stringstream strm{};

  // 後ろ2桁による年
  strm << "99";
  year y;
  strm >> parse("%y", y);
  if (strm) std::cout << y << "\n";

  // ストリームクリア
  std::exchange(strm, {});

  // 世紀とそこからの経過年
  strm << "18世紀99年";
  strm >> parse("%C世紀%y年", y);
  if (strm) std::cout << y << "\n";
}
```
MSVC 2019の出力例
```
1999
1899
```

ただどうやら、`%C`と`%y`を同時に指定した時は`%C × 100 + %y`の値になり、直感と1世紀ずれます。すなわち、ここでの世紀とは年の上位2桁の数のことです。

`%S`の指定は整数値による秒のみと、小数点以下を含めたミリ秒単位の両方を受理します。

```cpp
#include <chrono>

int main() {
  std::stringstream strm{};

  // 秒のパース
  strm << "23";
  seconds s;
  strm >> parse("%S", s);
  if (strm) std::cout << s << "\n";

  // ストリームクリア
  std::exchange(strm, {});

  // ミリ秒単位も含めてのパース
  strm << "23.333";
  milliseconds ms;
  strm >> parse("%S", ms);
  if (strm) std::cout << ms << "\n";
}
```
```
23s
23333ms
```

全ての指定において、`N`を先頭に付加することでパース対象の文字数を調整することができます。`N`を指定しない場合のデフォルトは`%Y`（年）は4桁、`%S`（秒）はミリ秒指定の場合は入力次第、それ以外の指定では全て2桁です。

|フォーマット指定|意味|
|---|---|
|`%NC`|`N`桁の世紀|
|`%NY`|`N`桁の年|
|`%Ny`|`N`桁の年の後ろ2桁|
|`%Nm`|`N`桁の月|
|`%Nd`, `%Ne`|`N`桁の日|
|`%NH`|`N`桁の時（24時間時計）|
|`%NI`|`N`桁の時（12時間時計）|
|`%NM`|`N`桁の分|
|`%NS`|`N`桁の秒（ミリ秒）|

`std:format()`のフォーマットオプションにおいては出力桁数が固定あるいは自動伸長するのでこれらの`%N~`のオプションは存在していませんでした。入力の場合はそうはいかないので、明示的に指定するようになっています。

```cpp
#include <chrono>

int main() {
  std::stringstream strm{};
  strm << "025000年005月0023日";

  year_month_day ymd;
  strm >> parse("%6Y年%3m月%4d日", ymd);

  if (strm) std::cout << ymd << "\n";
}
```
```
25000-05-23
```

`N`の数値はそのフォーマットでパースする際に読み込む文字列長の最大値であり、これは基本的に先頭にいくつか`0`がついてる場合にそれを何桁まで許容するかの指定です。例えば`%Ny`を使用する場合、年の下`N`桁から年をパースしてくれるようにはなりません。

### 複合フォーマット指定

時分秒の表記は、ISO 8601のようにある程度決まったパターンがあります。そのため、そのような固定パターンを読み取るための複合的なフォーマット指定が用意されています。

|フォーマット指定|等価な指定|受理する例|
|---|---|---|
|`%F`|`%Y-%m-%d`|`2019-05-29`|
|`%NF`|`%NY-%m-%d`|`2019-05-29`|
|`%D`|`%m/%d/%y`|`05/29/2019`|
|`%T`|`%H:%M:%S`|`17:44:31`|
|`%R`|`%H:%M`|`17:44`|

```cpp
#include <chrono>

int main() {
  std::stringstream strm{};
  strm << "2022-04-01T12:01:45";

  // 秒単位システム時刻で受け取る
  sys_seconds st;
  strm >> parse("%FT%T", st);

  if (strm) std::cout << st << "\n";
}
```
```
2022-04-01 12:01:45
```

### タイムゾーンの指定

文字列化されている日付時刻にはタイムゾーン情報が付加されていることもあるため、タイムゾーン情報を指定するフォーマットも用意されています。

|フォーマット指定|意味|受理する例|
|---|---|---|
|`%Z`|タイムゾーンの略称|`JST`|
|`%z`|UTCからのオフセット|`+0900`|
|`%Ez`, `%Oz`|`%z`の改良コマンド|`+9`, `-04:30`|

`%Z`でパースしたタイムゾーンの略称は`parse()`の追加の引数に`std::string`オブジェクトの参照を渡して受け取る事ができ、`%z`でパースしたUTCからのオフセットは`minutes`オブジェクトの参照を渡して受け取る事ができます。

```cpp
#include <chrono>

int main() {
  std::stringstream strm{};
  strm << "2022-04-01T12:01:45 JST +0900";

  sys_seconds st;
  // タイムゾーン名を受け取る
  std::string tz_name;
  // UTCからのオフセットを受け取る
  minutes offset;

  strm >> parse("%FT%T %Z %z", st, tz_name, offset);

  if (strm) {
    std::cout << st << "\n";
    std::cout << tz_name << "\n";
    std::cout << offset << "\n";
  }
}
```
```
2022-04-01 03:01:45
JST
540min
```

UTCからのオフセット（`%z`）のパースに成功した場合、パースされた時刻からオフセット分引かれて（UTC時刻に直して）から第2引数（`st`、日付時刻受け取り変数）に出力されます。

`%Z`/`%z`の結果は同時に受け取るほか、どちらか片方だけを受け取ることもできます。

```cpp
// タイムゾーン名のみ受け取る
strm >> parse("%FT%T %Z %z"s, st, tz_name);

// オフセットのみ受け取る
strm >> parse("%FT%T %Z %z"s, st, offset);
```

`%Ez`/`%Oz`の改良コマンドは、UTCからのオフセットについて異なるフォーマットをパースするものです。`%z`の場合は`[+|-]hh[mm]`（`[]`内は省略可能とする）の形式の文字列をUTCオフセットとしてパースしますが、`%Ez`/`%Oz`の場合は`[+|-]h[h][:mm]`の形式の文字列をUTCオフセットとしてパースします。

すなわち、改良コマンドの場合は時間は1桁でもよく（先頭の0を省略可）、分の2桁との間には`:`が必須となります。

```cpp
#include <chrono>

int main() {
  std::stringstream strm{};
  // チャタム諸島標準時（CHADT）
  strm << "2022-04-01T12:45:30 CHADT +12:45";

  sys_seconds st;
  // タイムゾーン名を受け取る
  std::string tz_name;
  // UTCからのオフセットを受け取る
  minutes offset;

  strm >> parse("%FT%T %Z %Ez", st, tz_name, offset);

  if (strm) {
    std::cout << st << "\n";
    std::cout << tz_name << "\n";
    std::cout << offset << "\n";
  }
}
```
```
2022-04-01 00:00:30
CHADT
765min
```

### ロケール依存フォーマット指定

年や日、時分などの基本単位を指定するフォーマット指定では、頭に`O`または`E`を付加することでロケール依存のフォーマット指定になります。ロケール依存フォーマットにおいては、パース時にその時点の（ストリームに指定された）ロケールによって定義されている数値等の代替表現を考慮するようになります。

|フォーマット指定|意味|例（日本語ロケール）|
|---|---|---|
|`%EC`|期間（時代）名|`平成`|
|`%EY`|年の代替表現|??|
|`%Oy`|年の後ろ2桁の代替表現|??|
|`%Ey`|`%EC`からのオフセット|`14`|
|`%Om`|月|`12`|
|`%Od`, `%Oe`|日|`6`|
|`%OH`|時（24時間時計）|`22`|
|`%OI`|時（12時間時計）|`9`|
|`%OM`|分|`35`|
|`%OS`|秒|`15`|
|`%A`, `%a`|曜日名|`日`, `日曜日`|
|`%B`, `%b`|月名|`April`, `4月`|
|`%p`|午前・午後|`AM`, `PM`|

`%EY`や`%Om`、`%Od`などではロケール定義の数値の代替表現というものが追加で読み取り可能となります。ただし、これが何を指すのかはよく分かりません。例えば、日本語ロケールにおいて全角数字や漢数字、丸数字などを読み取れるものではないようで、半角アラビア数字以外の何を読み取るためのものなのかは不明です。

```cpp
#include <chrono>

int main() {
  std::stringstream strm{};
  strm.imbue(std::locale{"ja-JP"});

  strm << "2022年6月21日 火曜日 11時11分11秒";

  sys_seconds st;
  strm >> parse("%EY年%B%Od日 %a %OH時%OM分%OS秒", st);

  if (strm) std::cout << st << "\n";
  
  // ストリームクリア
  std::exchange(strm, {});
  strm.imbue(std::locale{"ja-JP"});
  
  // 曜日短縮名前と12時間時計+午後指定
  strm << "2022年6月21日 火 PM 11時11分11秒";
  strm >> parse("%Y年%B%d日 %a %p %OI時%M分%S秒", st);

  if (strm) std::cout << st << "\n";
}
```
```
2022-06-21 11:11:11
2022-06-21 23:11:11
```

曜日指定（`%A`/`%a`）は完全名（金曜日）と短縮名（金）の両方が使用でき、午前午後の指定（`%p`）は12時間時計での時刻（`%I`/`%OI`）と共に使用しないと意味を持ちません。

このロケール依存フォーマット指定についても、組み合わせた複合フォーマット指定が用意されています。

|複合フォーマット|意味|
|---|---|
|`%c`|日付と時刻|
|`%Ec`|日付と時刻の代替表現|
|`%r`|午前午後指定を含む12時間時計による時刻|
|`%x`|日付|
|`%X`|時刻|
|`%Ex`|日付（`%x`）の代替表現|
|`%EX`|時刻（`%X`）の代替表現|

これらロケール依存フォーマット指定においては、使用されるロケールで代替表現が定義されていない場合があり、その場合は対応するロケール非依存フォーマット指定（`O`/`E`を取り除いたもの）と同等の意味になります。その際、`%r`は対応するロケール非依存フォーマットがないためパースに失敗します。

### 週/曜日番号フォーマット指定

年内の何周目や何日目といった番号で日付を指定することがあるようで、それをパースするためのフォーマットが用意されています。

|フォーマット指定|意味|受理する例|
|---|---|---|
|`%U`|週番号|`15`|
|`%W`|週番号|`15`|
|`%w`|曜日番号|`6`（=土曜日）|
|`%j`|元旦からの日単位オフセット|`364`|
|`%NU`|`N`桁の週番号|`15`|
|`%NW`|`N`桁の週番号|`15`|
|`%Nw`|`N`桁の曜日番号|`6`|
|`%Nj`|`N`桁の元旦からの日単位オフセット|`364`|
|`%OU`|ロケール依存週番号|`15`|
|`%OW`|ロケール依存週番号|`15`|
|`%Ow`|ロケール依存曜日番号|`6`|

週番号とは1年の最初の週を1としてその年の何週目かを指定するもので、曜日番号とは日曜を0として0〜6の数字でその週の曜日を指定するものです。

`%U`と`%W`の違いは基準となる最初の週をどう取るかにあります。`%U`は年内最初の日曜日を起点としてその週が`01`になり、`%W`は年内最初の月曜日を起点としてその週が`01`になります。どちらの場合もそれより前の週は`00`です。つまりは、1週間の始まりが日曜なのか月曜なのかの違いです。

年と`%U`/`%W`をパースしただけでは日の単位が定まっていないためまだ完全な日付になっていません。追加で、（その週の）曜日番号をパースすることで完全な日付となります。

```cpp
#include <chrono>

int main() {
  std::stringstream strm{};

  strm << "2018年36週目3";

  year_month_day ymd;

  strm >> parse("%Y年%U週目%w", ymd);
  if (strm) std::cout << ymd << "\n";

  strm.seekg(0);
  
  strm >> parse("%Y年%W週目%w", ymd);
  if (strm) std::cout << ymd << "\n";
  
  // ストリームクリア
  std::exchange(strm, {});
  
  strm << "2022年182日目";

  strm >> parse("%Y年%j日目", ymd);
  if (strm) std::cout << ymd << "\n";
}
```
```
2018-09-12
2018-09-05
2022-07-01
```

2018年は元旦が月曜日なので、`%U`と`%W`で読み取った週番号からの日付は1週間ずれています。

### ISO 8601ベースフォーマット

ISO 8601は日付と時刻の表記についての国際標準規格で、そこでは前項の週番号などの少し異なる定義があります。そのため、それを読み取るためのフォーマット指定が用意されています。

|フォーマット指定|意味|例|
|---|---|---|
|`%G`|ISOベース年|`2022`|
|`%g`|ISOベース年の末尾2桁|`22`|
|`%V`|ISO週番号|`1`|
|`%u`|ISO曜日番号|`7`（=日曜日）|
|`%NG`|`N`桁のISOベース年|`2022`|
|`%Ng`|ISOベース年の末尾`N`桁|`22`|
|`%NV`|`N`桁のISO週番号|`1`|
|`%Nu`|`N`桁のISO曜日番号|`7`|
|`%Ou`|ロケール依存ISO曜日番号|`7`|

ISO 8601の週番号（`%V`）は、年内最初の木曜日を起点としてその週が`01`になり、それより前の日は前年の最終週となります（週番号`00`はない）。そして、ISO 8601において週は月曜始まりです。その結果、それより前の金土日曜日がグレゴリオ暦では新年初週であるにもかかわらず前年最終週として扱われたり、年内最初の木曜日と同週の月火水曜日がグレゴリオ暦では前年最終週なのに新年初週として扱われる、といったことが起きます。

ISOベース年（`%G`/`%g`）とはこの週の規定に沿った年であり、ISO曜日番号（`%u`）は月曜日が1で日曜日が7となる、1〜7の数字です。

```cpp
#include <chrono>

int main() {
  std::stringstream strm{};

  // 2022年最初の土曜日
  strm << "2022-W1-6";

  year_month_day ymd;

  strm >> parse("%G-W%V-%u", ymd);
  if (strm) std::cout << ymd << "\n";
  
  std::exchange(strm, {});

  // 2021年最後の土曜日
  strm << "2021-W52-6";

  strm >> parse("%G-W%V-%u", ymd);
  if (strm) std::cout << ymd << "\n";

  std::exchange(strm, {});
  
  // 2022年最初の土曜日
  strm << "2022年0週目6";

  strm >> parse("%Y年%W週目%w", ymd);
  if (strm) std::cout << ymd << "\n";

  std::exchange(strm, {});
  
  // 2021年最後の土曜日
  strm << "2021年52週目6";

  strm >> parse("%Y年%W週目%w", ymd);
  if (strm) std::cout << ymd << "\n";
}
```
```
2022-01-08
2022-01-01
2022-01-01
2021-12-25
```

### その他

日付時刻以外のところに関して、特殊な意味を持つフォーマット指定（と文字）が用意されています。

|フォーマット指定|意味|
|---|---|
|`%%`|`%`のエスケープ|
|`%n`|1個のホワイトスペース|
|`%t`|0/1個のホワイトスペース|
|` `（ホワイトスペース）|0個以上のホワイトスペース|

実のところ、フォーマット文字列中のホワイトスペースは0以上の任意個数のホワイトスペースを受理する特殊なフォーマット指定となっています。ホワイトスペースの個数を厳密に制御したい場合は`%n`や`%t`を使用すると良いでしょう。

## 落穂

C++20`<chrono>`ライブラリの拡張はここまでで一通り見終わりました。ご覧の通りかなり大規模な拡張となっています。そして、ここまでの解説ではその全てを補い切れておらず、タイムゾーンデータベース関連や`hh_mm_ss`クラス周りなど省略しているところがいくつかあります。

気になる方は、cpprefjpの`<chrono>`のページ(https://cpprefjp.github.io/reference/chrono.html)を眺めて、本書で抜け落ちているところを探してみると良いかもしれません。

\clearpage

# `<compare>`

`<compare>`では、三方比較演算子（`<=>`）の戻り値型となる比較カテゴリ型と三方比較にまつわるコンセプト、およびその周辺のユーティリティが提供されます。

このヘッダは、`<=>`を用いるときには明示的にインクルード（もしくは`import`）しなければなりません。

```cpp
//#include <compare>

int main () {
  int a = 1;
  int b = 2;

  auto cmp = a <=> b; // ng、戻り値型が定義されていない
}
```

変なオーバーロードを除いて、`<=>`による比較の戻り値型は比較カテゴリ型と呼ばれるライブラリ定義の型であり、それがこのヘッダに定義されています。`<=>`を使用した時に自動でインクルードされたりはしないので、明示的にインクルードする必要があります。

標準で定義されている比較カテゴリ型は3種類あり、それぞれ数学的な順序関係を表現しています。

|比較カテゴリ型名|対応する順序関係|
|---|---|
|`std::strong_ordering`|全順序|
|`std::weak_ordering`|弱順序|
|`std::partial_ordering`|半順序|

`<=>`による比較の実装においては、返す比較カテゴリ型に対応した順序関係を満たしているようにしておかなければなりません。

比較カテゴリ型では、その静的メンバ定数からその型の値（そのカテゴリにおける比較結果を表す値）を取得することができます。

|静的メンバ定数|意味|使用可能な型|
|---|---|---|
|`less`|`a < b`|3つとも|
|`greater`|`a > b`|3つとも|
|`equivalent`|`a == b`（同値）|3つとも|
|`equal`|`a == b`（等値）|`strong_ordering`のみ|
|`unorderd`|比較不能|`partial_ordering`のみ|

これらの定数からは、`a <=> b`という比較の結果を表す、その名前に応じた値が取得できます。

```cpp
#include <compare>

void ord(std::strong_ordering);   // 1
void ord(std::weak_ordering);     // 2
void ord(std::partial_ordering);  // 3

int main() {
  ord(std::strong_ordering::less);  // 1が呼ばれる
  ord(std::weak_ordering::less);    // 2が呼ばれる
  ord(std::partial_ordering::less); // 3が呼ばれる

  auto er1 = std::weak_ordering::equal;      //ng
  auto er2 = std::strong_ordering::unorderd; //ng
}
```

比較カテゴリ型の値から比較結果を取得するには`0`との比較を行いますが、この操作は必ずしも見て何をしているか分かりやすいものではありません。そこで、`<=>`による比較結果を取得するための名前付き比較関数が用意されています。

```cpp
#include <compare>

template<typename T>
void comp_test(T a, T b) {
  auto cmp = a <=> b;

  std::cout << "== : " << std::is_eq(cmp)   << "\n";
  std::cout << "!= : " << std::is_neq(cmp)  << "\n";
  std::cout << "<  : " << std::is_lt(cmp)   << "\n";
  std::cout << "<= : " << std::is_lteq(cmp) << "\n";
  std::cout << ">  : " << std::is_gt(cmp)   << "\n";
  std::cout << ">= : " << std::is_gteq(cmp) << "\n";
}

int main() {
  std::cout << std::boolalpha;

  comp_test(17, 20);
}
```

この出力はつぎのようになります

```
== : false
!= : true
<  : true
<= : true
>  : false
>= : false
```

これらの名前付き比較関数を用いると、`<=>`による比較の結果から何を（どの比較としての結果を）取得したいかを明確にすることができ、コードの可読性を向上させることができます。

ところで、この例での`comp_test`のように、（関数）テンプレート内でテンプレートパラメータ型の値について三方比較を行う場合、C++20以降はコンセプトによって制約したくなるでしょう。

通常の比較のためには、`<concepts>`ヘッダに同値比較（`std::equality_compareble`）と順序付け比較（`std::totally_ordered`）とその`_with`付きのコンセプトが用意されていました。`<compare>`ヘッダにはこれらと同様に、三方比較のためのコンセプトが用意されます。

```cpp
// <=>による比較が可能な型Tを受けいれる、1
template<std::three_way_comparable T>
void comp_test(T a, T b) {
  auto cmp = a <=> b;

  ...
}

// 異なる型同士で<=>による比較が可能な型T, Uを受けいれる、2
template<typename T, std::three_way_comparable_with<T> U>
void comp_test(T a, U b) {
  auto cmp = a <=> b;

  ...
}

int main() {
  comp_test(17, 20);      // ok、1が呼ばれる
  comp_test(17u, 20ull);  // ok、2が呼ばれる
  comp_test(17, 20.0);    // ok、2が呼ばれる
  comp_test(17.0f, 20.0); // ok、2が呼ばれる
  
  comp_test(17u, 20);          // ng、符号の異なる整数型の比較  
  comp_test(nullptr, nullptr); // ng、比較が定義されていない
}
```

`std::three_way_comparable`は1つの型`T`が`<=>`による比較が可能であることを表し、`std::three_way_comparable_with`は異なる型`T`と`U`の間で`<=>`による比較が可能であることを表します。命名規則は`std::equality_compareble`などと同様になっています。

また、この2つのコンセプトは追加の引数として比較カテゴリ型を取ることができ、`<=>`の比較が特定の比較カテゴリ型を返すことも含めて制約できます。

```cpp
// <=>の比較が全順序の上で行われる型Tを受けいれる
template<std::three_way_comparable<std::strong_ordering> T>
void comp_test(T a, T b) {
  auto cmp = a <=> b;

  ...
}

// <=>の比較が全順序の上で行われる型T, Uを受けいれる
template<typename T, 
         std::three_way_comparable_with<T, std::strong_ordering> U>
void comp_test2(T a, U b) {
  auto cmp = a <=> b;

  ...
}

int main() {
  comp_test(17, 20);     // ok
  comp_test(17.0, 20.0); // ng、結果はpartial_ordering
  
  comp_test2(17u, 20ull);  // ok
  comp_test2(17, 20.0);    // ng、結果はpartial_ordering
}
```

## 三方比較の結果型取得

クラスのデフォルト`<=>`比較の戻り値型など、多数の型が参加する三方比較の結果型（比較カテゴリ型）を推論するのはなかなか難しいものがあります。コンパイラはすべてわかっていますが、それを取得するコードを書くのも少し手間です。

そのため、それを求めるためのメタ関数、`std::common_comparison_category<Ts...>`が用意されています。これは、`Ts...`に任意個数の型を渡すと、それらすべての型が参加する`<=>`の結果の比較カテゴリ型を返してくれるものです。

あるいは、単に1つの型について結果のカテゴリ型を求めたい場合のために、`std::compare_three_way_result<T>`も用意されています。これは`T`のオブジェクト`t, u`の間での`t <=> u`の結果型を返してくれます。

```cpp
#include <comapre>

// partial_ordering
using t1 = std::common_comparison_category_t<int, char, float>;

// strong_ordering
using t2 = std::compare_three_way_result<int>;
```

これらは例えば、別の型`T`の薄いラッパのようなクラステンプレートにおいて、`<=>`のデフォルト実装をフォールバックさせるために次のように活用できます。

```cpp
#include <comapre>

// Tが<=>を使用できない場合、Catにフォールバックする
template<typename T, typename Cat>
using fallback_comp3way_t = std::conditional_t<
        std::three_way_comparable<T>,
          std::compare_three_way_result<T>,
          std::type_identity<Cat>
      >::type;

// フォールバック先のカテゴリ
using category = std::weak_ordering;

template<typename T>
struct wrap {
  T t;

  // <=>を使用可能ならそれを、そうでないなら< ==を使ってdefault実装
  auto operator<=>(const wrap&) const
    -> fallback_comp3way_t<T, category>
      = default;
};

template<typename T1, typename T2, typename T3>
struct triple {
  T1 t1;
  T2 t2;
  T3 t3;

  // <=>を使用可能ならそれを、そうでないなら< ==を使ってdefault実装
  auto operator<=>(const triple&) const
    -> std::common_comparison_category_t<
          fallback_comp3way_t<T1, category>,
          fallback_comp3way_t<T2, category>,
          fallback_comp3way_t<T3, category>
        >
      = default;
};
```

これらに対して、次のような`<=>`演算子を持たない型を与えた時でも、三方比較の`default`実装のフォールバック仕様（戻り値型が指定してあれば`< ==`を用いて実装を試みる）によって`<=>`の自動実装が働きます。

```cpp
// <=>は定義していないが、< ==は定義してあるクラス
struct no_spaceship {
  int n;

  bool operator<(const no_spaceship& that) const noexcept {
    return n < that.n;
  }

  bool operator==(const no_spaceship& that) const noexcept {
    return n == that.n;
  }
};

int main() {
  {
    wrap<no_spaceship> t1 = { {20} }, t2 = { {30} };

    bool b1 = t1 < t2;  // true
    bool b2 = t1 <= t2; // true
    bool b3 = t1 > t2;  // false
    bool b4 = t1 >= t2; // false
  }
  {
    triple<int, double, no_spaceship> t1 = {10, 3.14, {20}}, 
                                      t2 = {10, 3.14, {30}};
                                      
    bool b1 = t1 < t2;  // true
    bool b2 = t1 <= t2; // true
    bool b3 = t1 > t2;  // false
    bool b4 = t1 >= t2; // false
  }
}
```

`<=>`の`default`実装においては、通常はそのクラスのメンバ（及び規定クラス）全てを`<=>`で比較していくような実装が取られますが、もしその際に`<=>`を使用できないメンバがあるとその`default`実装は暗黙`delete`されます。C++17以前に作成された型は全て`<=>`を持っていないので、これは既存のコードをC++20に移行する際に問題となります。

そこで、あるメンバが`<=>`を持っていない場合でもその型が`< ==`による比較を実装していれば、それを用いて`default`実装を行うことができるようになっています。ただしこの時、`default`実装したい`<=>`の戻り値型は`auto`ではなく比較カテゴリ型を明示的に指定しておかなければなりません。

すると、上記の`wrap`や`triple`のようなクラスにおいてはその`<=>`が戻り値型`auto`で`default`実装できるかはテンプレートパラメータの型に依存して変化し、`<=>`の`default`宣言は複数共存できないため、`<=>`を自動実装するには戻り値型を明示指定した`default`宣言を使用したほうが良くなります。しかし、なるべく元の比較カテゴリを維持しながら`<=>`が戻り値型`auto`で定義できない場合にのみフォールバック実装を利用したくなりますが、それを叶えようとすると比較を手書きしなければならなくなります。

上記サンプルコードの`fallback_comp3way_t`のようなものを用意しておくと、1つの`<=>`の`default`宣言だけでその両方の`default`宣言を兼ねることができるようになります。これは、`<=>`が使える場合はその戻り値型を、使えない場合は指定した型`Cat`を`<=>`の`default`宣言の戻り値型として自動的に指定されるようにすることで、なるべく元の比較カテゴリを維持するようにしつつフォールバック実装時は指定したカテゴリを利用するようにしています。また、`<=>`の`default`宣言を維持していることで、`==`も自動定義されています。

`std::common_comparison_category`及び`std::compare_three_way_result`は、このような`<=>`周辺のためのTMPを行う場合に有効活用できます。

## 比較関数オブジェクト

`std::less`など、各種比較演算に関してはそれを行う関数オブジェクト（のクラス）が用意されています。`<=>`に対しても`std:compare_three_way`というクラスが用意されており、これはデフォルト構築可能でステートレスな関数オブジェクトクラスです。

```cpp
int main() {
  std:compare_three_way cmp{};

  auto c1 = cmp(1, 2);      // 1 <=> 2
  auto c2 = cmp(1u, 2ull);  // 1u <=> 2ull
  auto c2 = cmp(1.0f, 2.0); // 1.0 <=> 2.0
}
```

`std::less`等と異なるところは比較対象の型を取るクラステンプレートではないところで、最初からその関数呼び出し演算子が関数テンプレートとして定義されているため、異種比較を効率的に行うことができます。

例えば`std::set`や`std::map`等のように、テンプレートパラメータによって内部で使用する比較をカスタマイズしたい場合に使用できます。

また、`std:compare_three_way`による比較では、ポインタ値の比較において実装定義の狭義全順序の上での比較が可能です（これは`std::less`等でも同様）。素の`<=>`演算子との違いは、無関係なオブジェクト同士など、結果が未定義となる比較においても実装定義とはいえ比較可能である点です。狭義全順序であるとは、多くのプログラマーが思い描くようなメモリ空間のメンタルモデルと（おそらく）一致するような順序付けが行われるという事です。

```cpp
#include <compare>

int main() {
  std:compare_three_way cmp{};

  int a = 1, b = 2;

  auto c1 = &a <=> &b;    // UB
  auto c2 = cmp(&a, &b);  // ok
}
```

また、特定の比較カテゴリの上での三方比較を行うための関数オブジェクト（`std::strong_order, std::weak_order, std::partial_order`）も用意されています。これらの関数オブジェクトは、その名前に対応する比較カテゴリ上で、基本的には`<=>`演算子を用いた比較を行います。

```cpp
#include <compare>

int main() {
  int n1 = 10, n2 = 20;

  auto c1 = std::strong_order(n1, n2);   // strong_ordering::less
  auto c2 = std::weak_order(n1, n2);     // weak_ordering::less
  auto c3 = std::partial_order(n1, n2);  // partial_ordering::less
}
```

これらの関数オブジェクトによる比較においては、対応する比較カテゴリの値が必ず得られ、そのカテゴリの比較が行えない場合はコンパイルエラーとなります。

これらはカスタマイゼーションポイントオブジェクト（CPO）という事前定義関数オブジェクトであり、その比較の実装をカスタマイズすることができます。このカスタマイズに使用する関数は、非修飾ルックアップ（すなわちADL）によって呼び出し可能な同名の関数である必要があり、フリー関数か*Hidden frineds*関数として実装します。

```cpp
#include <compare>

struct S {
  int n1 = 0;
  int n2 = 0;
  int n3 = 0;

  // strong_orderによる比較をカスタムする
  friend auto strong_order(const S& lhs, const S& rhs) -> std::strong_ordering {
    // 宣言順と異なる順序で比較
    if (auto cmp = lhs.n1 <=> rhs.n1; cmp != 0) return cmp;
    if (auto cmp = lhs.n3 <=> rhs.n3; cmp != 0) return cmp;
    return lhs.n2 <=> rhs.n2;
  }
};

int main() {
  S s1{0, 1, 2}, s2{0, 2, 2};

  auto c1 = std::strong_order(s1, s2);   // strong_ordering::less
  auto c2 = std::weak_order(s1, s2);     // weak_ordering::less
  auto c3 = std::partial_order(s1, s2);  // partial_ordering::less
}
```

ある比較カテゴリ型の値はより弱い比較カテゴリ型の値へと暗黙変換することができます。これらのCPOにおいても同様に、より弱い比較においてはより強い比較を通して結果を得ることができます。上記例では、`std::weak_order`と`std::partial_order`による比較は、`std::strong_order`で比較した結果を変換することで行われています。

さらに、`std::strong_order`と`std::weak_order`では、浮動小数点数の比較時にIEEE 754で定義される浮動小数点数の全順序に従った順序づけによって比較を行い、その結果を対応する比較カテゴリで返します。

`std::strong_order`の場合は次のような順序付けが行われます

```
{負のquiet NaN} < {負のsignaling NaN} < {負の数} < -0.0 < +0.0 < {正の数} < {正のsignaling NaN} < {正のquiet NaN}
```

それぞれの集合（`{}`）内でも全ての値の間で順序付けを行うことができ、例えば`NaN`の値はそのペイロード（先頭ビットを除いた仮数部）を整数として扱った時の大小関係によって順序付けけされます。

`std::weak_order`の場合は次のようになります

```
{全ての-NaN} < {-Inf} < {負の正規化数} < {負の非正規化数} < {±0.0} < {正の非正規化数} < {正の正規化数} < {+Inf} < {全ての+NaN}
```

こちらの場合、それぞれの集合内では組み込みの`<=>`による比較によって順序付けされます。もし比較不可能（`unorderd`）となる場合、弱順序上ではそれらは同値として扱われます。例えば、`±NaN`の集合内では全ての要素が同値となります。その一方で、`NaN`とそれ以外のものの間では順序付けが行われます。

```cpp
#include <compare>

int main() {
  std::cout << std::boolalpha;

  // +/-0.0の順序付け
  double z1 = +0.0, z2 = -0.0;

  auto c1 = std::strong_order(z1, z2);   // strong_ordering::less
  auto c2 = std::weak_order(z1, z2);     // weak_ordering::equivalent

  std::cout << std::is_lt(c1) << "\n";
  std::cout << std::is_lt(c2) << "\n";

  // NaNの順序付け
  double n1 = std::numeric_limits<double>::quiet_NaN();
  double n2 = std::numeric_limits<double>::signaling_NaN();

  auto c3 = std::strong_order(n1, n2);   // strong_ordering::less
  auto c4 = std::weak_order(n1, n2);     // weak_ordering::equivalent

  std::cout << std::is_lt(c3) << "\n";
  std::cout << std::is_lt(c4) << "\n";

  // NaNと1.0の順序付け
  auto c5 = std::weak_order(1.0, n2);     // weak_ordering::less
  auto c6 = std::partial_order(1.0, n2);  // partial_ordering::unorderd
  
  std::cout << std::is_lt(c5) << "\n";
  std::cout << std::is_lt(c6) << "\n";
}
```
```
true
false
true
false
true
false
```

浮動小数点数型がIEEE 754のものではない場合は話が変わってくるのですが、おそらくIEEE 754派生なものしか現存していないと思われるのでほぼ同様の比較が可能なはずです（さもなければコンパイルエラーとなるはず）。

最後に、これら3つのCPOにはその名前を`xxx`として`compare_xxx_fallback`のような名前で、`< ==`による比較にフォールバックするCPOが用意されています。これは`<=>`のデフォルト実装時のフォールバック仕様と同様に、`<=>`が使用できない場合に`< ==`による比較によって三方比較を行おうとするものです。

```cpp
#include <compare>

struct no_spaceship {
  int n;

  bool operator<(const no_spaceship& that) const noexcept {
    return n < that.n;
  }

  bool operator==(const no_spaceship& that) const noexcept {
    return n == that.n;
  }
};


int main() {
  no_spaceship ns1{10}, ns2{20};

  auto c1 = std::compare_strong_order_fallback(ns1, ns2);   // strong_ordering::less
  auto c2 = std::compare_weak_order_fallback(ns1, ns2);     // weak_ordering::less
  auto c3 = std::compare_partial_order_fallback(ns1, ns2);  // partial_ordering::less
}
```

これらのものでは、`xxx`のCPOを用いた比較が可能な場合はそれを用いて比較が行われ、不可能な場合にのみフォールバックします。

\clearpage
# `<version>`

`<version>`ヘッダでは、C++標準ライブラリのバージョンに関わる情報を提供します。

\clearpage
# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- 安全で便利なstd::bit_castを使おう - よーる  
  (https://lpha-z.hatenablog.com/entry/2021/12/05/231500)
- C++マルチスレッド一巡り(https://zenn.dev/yohhoy/books/cpp-stdlib-multithreading)
- std::threadデストラクタ動作検討の歴史(https://zenn.dev/yohhoy/scraps/393bce83b4f3f0)
- threadの利用と例外安全（その1）(https://yohhoy.hatenadiary.jp/entry/20120209/p1)
- std::jthread and cooperative cancellation with stop token  
  (https://www.nextptr.com/tutorial/ta1588653702/stdjthread-and-cooperative-cancellation-with-stop-token)
- 可変長テンプレートでもstd::source_locationを使いたい！  
  (https://in-neuro.hatenablog.com/entry/2021/12/15/000033)
- 🕒時刻計算と📅カレンダー計算は違うよ。全然違うよ。  
  (https://qiita.com/yohhoy/items/d1f9004c66cc6ff15774)
- {fmt} Formatting & Printing Library  
  (https://hackingcpp.com/cpp/libs/fmt.html)
- Date and Time - The GNU C Library  
  (https://www.gnu.org/software/libc/manual/html_node/Low_002dLevel-Time-String-Parsing.html)
- ISO 8601 - Wikipedia  
  (https://ja.wikipedia.org/wiki/ISO_8601)