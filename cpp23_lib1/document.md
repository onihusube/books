---
title: C++23 ライブラリ機能1
author: onihusube
date: 2025/08/17
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

`<stdfloat>`
`<generator>`


本書はC++23をベースとして記述されています。そのため、C++20までに導入されている機能については特に導入バージョンについて触れず、知っているものとして詳しい解説なども行いません。

文中では、基本的には`std::`を省略していませんが、文脈上明らかな場合に一部で`std::`を省略することがあります。また、通常の関数の表記（`func()`）に対してメンバ関数の表記（`.member_func()`）は先頭に`.`を付加して区別します。

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

# stdモジュール

C++20でモジュールが導入されたのに引き続いて、C++23では標準ライブラリをモジュールとして利用することができます。ここでいうモジュールというのはヘッダユニットの事ではなく、名前付きモジュールとして標準ライブラリを利用できるということです。

```cpp
import std;

int main() {
  std::cout << std::format("Hello {}", "world!");
}
```

標準ライブラリモジュールの名前は`std`という名前で、この例のように`import std;`とすることでインポートできます。`std`モジュールからは全ての`std`名前空間以下の標準ライブラリ機能がエクスポートされており、この一文で全てのC++標準ライブラリ機能にアクセスすることができます。

ただし、Cライブラリのものは`std`モジュールからはエクスポートされていません。

```cpp
import std;

int main() {
  printf("Hello world");  // ng、宣言が見えない
}
```

Cライブラリ由来の機能（`<c~>`なヘッダにあるものや`std`名前空間外のグローバル名前空間で定義されているもの）に関しては、`std.compat`モジュールをインポートすることでアクセスできるようになります。

```cpp
import std.compat;

int main() {
  std::cout << std::format("Hello {}", "world!"); //ok
  printf("Hello world");  // ok
}
```

`std.compat`モジュールからはCのライブラリ機能（C++標準で提供されているサブセット）に加えてC++の標準ライブラリ機能もすべてエクスポートされています。そのため、その名前の構造から受けるイメージに反して、`std`モジュールよりも`std.compat`モジュールの方がより大きなモジュールになっています。このような構造になっているのはC++標準ライブラリの構造由来の特殊なものなので、独自のモジュールライブラリ作成時にこのような名前構造の慣習に必ずしも従わなくても良いかもしれません。

なお、これ以外のモジュール、例えば`std.io`だとか`std.conatiner`などのようなモジュールはC++23では用意されません。利用できるのは`std`と`std.compat`のみです。

## マクロ

ヘッダユニットはマクロをエクスポートすることができますが、名前付きモジュールはマクロをエクスポートできません。`std`/`std.compat`モジュールは名前付きモジュールとして提供されるため、ここからはマクロはエクスポートされません。そのようなものにはたとえば、`assert`や`errno`、`offsetof`などがあります。

```cpp
import std.compat;

void f(int n) {
  assert(n == 0); // ng、<cassert>のインクルード/インポートが必要
}
```

## 標準ライブラリモジュールの特殊な保証

名前付きモジュールに属する宣言はそのモジュールに所有されているため、異なるモジュールに属する同じ名前（名前空間まで含めて）のエンティティは異なるものとして識別されます（厳密には実装定義）。

一方、名前付きモジュールに属していない他のすべての宣言はグローバルモジュールに属すものとして扱われ、グローバルモジュールに所有されます。ここでのエンティティ間の同一性のルールは今日ODRとして知られているルールそのままが適用されます。

そして、ヘッダユニットは1つのモジュールとなるものの、そのエンティティはすべてグローバルモジュールに属します。

すると、`std`モジュールのインポートと標準ライブラリのヘッダユニットのインポートを両方書いてしまった時に、それぞれからエクスポートされるエンティティは異なるものを参照するのではないか？という問題が生じます。

```cpp
import std; // stdモジュールのインポート
import <iostream>;  // ヘッダユニットのインポート

int main() {
  std::cout << "hello?";  // coutの実体はどこに？多重定義エラーになる？
}
```

さらにいえば、通常のヘッダインクルードもこれらと共存可能です。

```cpp
import std;
import <iostream>;
#include <iostream>
```

このようなコードはヘッダのインクルードなどを介すことで容易に出現する可能性があり、もしこれらのものがそれぞれ異なるエンティティを指すことになると、それはODRの問題を引き起こします。

このような問題が起こらないようにするため、標準ライブラリのエンティティは標準ライブラリモジュールのインポート、標準ライブラリヘッダユニットのインポート、標準ライブラリヘッダのインクルード、のいずれの方法によっても同じエンティティを参照することが保証されています。したがって、この3つの方法のいずれを用いていてもその方法に関わらず同じエンティティが利用可能になり、それが翻訳単位を跨いでいたとしても（ABIの保証が実装によって提供されていれば）同じエンティティを利用し、なおかつこの3つの方法が混在していても定義が衝突することはありません。

```cpp
import std;
import <iostream>;
#include <iostream>

int main() {
  std::cout << "hello?";  // ok、coutの実体は一つ
}
```

ちなみに、このような強い保証は`std/std.compat`モジュールに固有の特殊な仕様です。ユーザーが普通に定義する名前付きモジュールとその他の間ではこのような強い保証を提供することは（少なくとも標準の範囲内では）不可能です。

# `<print>`

C++20では`std::format()`の導入によってより使いやすく高パフォーマンスな文字列フォーマットが可能になりましたが、このフォーマット結果文字列を標準出力に出力するためには次のようにする必要があります

```cpp
void example(std::string_view world) {
  std::cout << std::format("Hello {}\n", world);
}
```

これの問題点の一つは、`std::format()`の結果として`std::string`の一時オブジェクトが生成されてしまっていることで、それによって（文字列が長いと）動的メモリ確保と解放が起こります。

別の問題点は、フォーマット済み文字列を出力するために従来のiostreamのAPIを使用していることで、フォーマット済み文字列に対する再フォーマットが考慮されることと、余分な関数呼び出し（`<<`そのもの及びその内部）のオーバーヘッドがかかることです。

C++23からは、フォーマット済み文字列を直接出力するための関数として`std::print()`が導入されます。

```cpp
#include <print>

void example(std::string_view world) {
  std::print("Hello {}\n", world);
}
```

`std::print`の引数は`std::format`と全く同じになりますが、フォーマット後文字列を返すのではなく標準出力にそのまま書き出します。これにより、中間の`std::string`が作成されることはなくなり、iostream APIの呼び出しも不要になっています。

`std::print`の出力はその最後に改行が挿入されないため改行したい場合は明示的な改行の指定（`\n`）が必要になりますが、これを自動で行うための`std::println()`も用意されています。

```cpp
#include <print>

int main() {
  std::string_view world = "World";

  std::println("Hello {}", world);  // 最後に改行を挿入する
  std::print("Hello {}", world);    // 最後に改行を挿入しない
  std::print("...");
}
```
```
Hello World
Hello World...
```

これにより、前章の`std`モジュールと合わせるとC++23以降のC++においてのHello Worldはこうなります

```cpp
import std;

int main() {
  std::println("Hello world!");
}
```

## フォーマットオプション

`std::print`で使用可能なフォーマットオプション及びその構文は`std::format`と全く同じです。一切の違いはなく、`std::formatter`を共有して同じ仕組みでフォーマットを行います。

そのため、`std::print`でユーザー定義の型を出力可能にする場合、`std::format`と同様に`std::formatter`の特殊化を追加すればよく、それによって`std::print`と`std::format`の両方で同時に使用可能になります。

フォーマットオプションのコンパイル時チェックに関しても`std::format`と同様に行われます。

フォーマットオプションの詳細は、「C++20 ライブラリ機能1」および「C++23 ranges」を参照してください。

## Unicode対応

`std::print`は文字列リテラルのエンコーディングがUTF-8である場合にのみにユニコード対応が規定されており、ユニコード範囲の文字を含む文字列を文字化けすることなく出力することができます。

```cpp
#include <print>

int main() {
  const char* str = "😇🥺🤔🤡";
  std::print("絵文字: {}", str);
}
```
```
絵文字: 😇🥺🤔🤡
```

`std::print`のユニコード対応判定はすなわち、`char`のエンコーディングがUTF-8であるかどうかによって決定されます。その判定方法については例えば、`std::print`の提案文書で次のような実装が示されています

```cpp
consteval bool is_utf8() {
  const unsigned char micro[] = "\u00B5";
  return sizeof(micro) == 3 && micro[0] == 0xC2 && micro[1] == 0xB5;
}

template <typename... Args>
void print(string_view fmt, const Args&... args) {
  if constexpr (is_utf8())
    vprint_unicode(fmt, make_format_args(args...));
  else
    vprint_nonunicode(fmt, make_format_args(args...));
}
```

`vprint_unicode()`/`vprint_nonunicode()`は`std::print`の内部実装であり、`std::format`における`vformat`に対応するものです。`vprint_unicode()`は出力先ストリームがユニコードを表示可能な端末を参照している場合、プラットフォームネイティブのユニコードAPIを使用して出力を行うことでユニコードを正しく出力します。`vprint_nonunicode()`はユニコードの考慮を行わずに文字列をそのままストリームに書き込むもので、出力先ストリームのユニコードサポートが無いあるいは不明な場合に使用されます。

最近（2025年頃）のLinux環境の場合はおそらく何もせずとも`std::print`はユニコード対応しています。Windows（MSVC）環境は少し厄介で、デフォルトだと`char`のエンコーディングはANSI互換エンコーディング（日本語だとShift-JIS）となっており非ユニコードです。Visual Studioの`/execution-charset:utf-8`などのUTF-8対応オプションを指定すると、`char`のエンコーディングがUTF-8になり`std::print`もユニコード対応されます。

MSVC STLの実際の実装では、上記例の`is_utf8()`のような関数に該当する実装は`_MSVC_EXECUTION_CHARACTER_SET`マクロがUTF-8（65001）に設定されているかをチェックしています。

ちなみに、`vprint_unicode()`の「出力先ストリームがユニコードを表示可能な端末を参照している場合」という性質の判定方法についても一応指定されていて、POSIXの場合は`isatty(fileno(stream)) != 0`、Windowsの場合は`GetConsoleMode(_get_osfhandle(_fileno(stream)), ...) != 0`の時とされています。さらに、Windowsの場合のネイティブのユニコードAPIとは`WriteConsoleW()`であることも指定されています。

`char`のシーケンスとしてのユニコード文字列は任意の整数値に対応する文字が設定される可能性があるため、その文字範囲には無効なコードポイントが含まれ得ます。`vprint_unicode()`によるユニコード出力の場合、ネイティブのユニコード出力に当たって変換が必要となる場合（典型的にはWindowsのUTF-16への変換）には、無効なコードポイントを`U+FFFD`（`�` REPLACEMENT CHARACTER）に置き換えて出力することが推奨されています（推奨なので従わない実装があるかもしれません）。

## 出力先の指定

`std::print`はその出力先が標準出力ストリーム（`stdout`）に固定されていますが、任意の出力先を明示的に指定して出力先を変更することができます。

```cpp
#include <print>
#include <ostream>  // ostreamへの出力の場合こちらのインクルードが必要

// ファイルポインタへの出力
void example1(std::FILE* out_file_ptr) {
  std::print(out_file_ptr, "output to FILE*");
}

// 出力ストリームへの出力
void example2(std::ostream& out_stream) {
  std::print(out_stream, "output to ostream");
}
```

指定できる出力先は、Cのファイルポインタ（`FILE*`）もしくはC++の出力ストリーム（`std::ostream`）のどちらかです。どちらの場合でも出力先は第一引数に指定し、その場合の`std::print`/`std::println`はフォーマット済み文字列を指定された出力先に対して出力します。

こちらのオーバーロードで最も使用するのは、ファイルに対する出力ではないかと思います。

```cpp
int main() {
  std::ofstream ofs{"./output.cav"};

  std::println(ofs, "seq, time, value");
  std::println(ofs, "{:d}, {:.3f}, {:g}", 0, 0.1, std::numbers::pi);
}
```

あるいは、標準エラー出力ストリームに出力したい場合にも使用できます。

```cpp
int main() {
  std::print(stderr, "error message(FILE*)");
  std::print(std::cerr, "error message(ostream)");
}
```

他にも、`std::stringstream`等のストリームに対しても出力可能です。

なお実は、`std::print(args...)`のデフォルト（出力先指定なしオーバーロード）は、`std::print(stdout, args...)`のように定義されています。このため、ユニコード対応に関しては`FILE*`/`std::ostream`どちらを取るものでも同じようにサポートされます（詳しくは前節）。

## フラッシュ

文字列リテラルのエンコーディングがUTF-8であり、ストリームがユニコード対応端末に接続されている場合（すなわち、`vprint_unicode()`を使用してユニコード対応出力が行われる場合）、`std::print`は出力の書き込み前に出力先ストリームをフラッシュします。

これは、`std::print`がネイティブのユニコードAPIを使用して書き込む場合は出力先ストリームの持つストリームバッファをバイパスして書き込みを行うことによって、従来のストリームを使用する出力機能とその出力順が同期しない可能性を無くすための措置です。

すなわち、次のような出力が

```cpp
printf("first");
std::print("second");
```

当然次のような順番で出力されること

```
first
second
```

を保証するためです。

`printf()`というか`stdout`は通常1行分のバッファリングを行うため、明示的なフラッシュあるいは`\n`の出力が無い場合、直ぐにはストリームに書き込まれない場合があります。一方で、`std::print`がネイティブのユニコードAPIを使用して書き込む場合はストリームバッファを経由しない可能性があるためすぐにストリームに書き込まれ、それによってこれら2つの出力機能のコード上での実行順と出力結果の順番は一致しない可能性があります。

ただし、Windowsの`stdout`は常にバッファリングを行わないため問題が起こらず、他のプラットフォームではネイティブのユニコードAPIを使用してもストリームバッファをバイパスしないため実際にはほとんどの場合にこのような問題は起こらないようです。

そのためこのような問題は理論的なものと思われますが、それが起こらないことを保証するために、`std::print`/`std::println`は書き込み前に出力先ストリームをフラッシュします。ただしこれが行われるのは、「**文字列リテラルのエンコーディングがUTF-8**」であり、「**出力先のストリームがユニコード対応端末に接続されている**」場合、のみです。例えば、前段の条件を満たしていてもファイルの書き込みでは行われないと思われます。

問題は起こらないことが分かっていてもフラッシュは行われるため、これは`std::print()`の出力パフォーマンスに影響を及ぼす可能性があります。オプトアウトの手段は用意されていないのですが、`vprint_nonunicode()`にはフラッシュの規定が無いためこちらを使えばフラッシュはされないはずです（ただし、ユニコードの出力保証はありませんし、使用方法も若干異なります）。

逆に、フラッシュが行われない場合の`std::print()`で明示的にフラッシュを行いたい場合にも`std::print()`でそれを行う方法は用意されていません。この場合は出力先ストリームを直接フラッシュするしかなく、標準的な方法としては`std::fflush()`（`FILE*`用）/`std::flush`（ioマニピュレータ）が使用できます。

## 標準ライブラリの対応

組み込み型の`std::formatter`特殊化はC++20の時点から用意されていましたが、標準ライブラリ型についてはほぼ用意されていませんでした。C++23ではそのサポートは若干改善され、次のものが`std::format`/`std::print`可能になっています

- 組み込み型
    - 整数型
    - 浮動小数点数型
    - `bool`
    - `char/wchar_t`
    - ポインタ型
      - `std::nullptr_t`
- 文字列
    - 文字型配列
    - 文字型ポインタ
    - `std::string`/`std::string_view`
- `<chrono>`関連型
- `<stacktrace>`
    - `std::basic_stacktrace`
    - `std::stacktrace_entry`
- `range`
    - 全てのコンテナおよびコンテナアダプタ
    - 任意の`view`型
- `std::tuple`/`std::pair`
- `std::thread::id`
    - `std::this_thread::get_id()`もしくは`std::thread::get_id()`の戻り値型

これらのものは、何もせずとも`std::print()`/`std::println()`で使用可能です。

# `<expected>`

`<expected>`ヘッダでは、エラーハンドリングのためのライブラリ型である`std::expected`が提供されます。

`std::expected`はクラステンプレートであり`std::expected<T, E>`のように使用して、`T`に正常値の型を、`E`にエラー値の型を指定します。`std::expected<T, E>`のオブジェクトは、その状態に応じて`T`か`E`のどちらか片方の値をだけを保持しており、どちらを保持しているかによってエラー状態かどうかの分岐を行うことができます。

```cpp
// エラーコード列挙型
enum class myerrc {
  ...
};

// エラーが起こるかもしれない処理
auto maybe_error() -> std::expected<double, myerrc>;

int main() {
  // 処理を実行
  auto res = maybe_error();

  // エラーチェック
  if (res) {
    // ✔ 正常時のパス
    ...
  } else {
    // ❌ エラー時のパス
    ...
  }
}
```

エラーハンドリングメカニズムに求められる特性を次のように定義すると

- 可視性
    - コードレビューの全体を通してエラーケースが表示されていること
    - エラーが隠されているとデバッグが困難になる
- 情報
    - エラーの詳細（発生個所・原因・解決方法等の情報）を含むことができる
- クリーンコード
    - エラーの処理は、コードの別のレイヤでできるだけ目立たないように行われる
    - コードを読むだけで、例外的な状態が存在することに気付ける
- 非侵入的
    - エラーの伝達チャネルが、正常な結果のための伝達チャネルを占有しない
    - エラーは可能な限り分離されているのが望ましい
      - 例: 関数戻り値チャネルはエラー専用であってはならない

`std::expected`と既存のメカニズム（例外・エラーコード戻り値）と比較は次のようになります

|特性＼方法|`std::expected`|例外|エラーコード|
|---|---|---|---|
|可視性|`⭕`|`❌`|`⭕`|
|情報|`⭕`|`⭕`|`❌`|
|クリーンコード|`⭕`|`🔺`|`⭕`|
|非侵入的|`⭕`|`⭕`|`❌`|

`std::expected`はそれぞれ次のように優れています

- 可視性
    - 戻り値型が`std::expected<T, E>`であるため、ユーザーはエラーケースを無視できない
- 情報
    - エラー値として任意の情報を載せられる
- クリーンコード
    - モナドインターフェースによって、エラー処理を別のコードレイヤに委譲できる
- 非侵入的
    - 関数戻り値チャネルを正常値とエラー値で共有する

`std::optional`もこれに近い特性を備えていますが、`std::expected`と異なるところはエラーの場合の値を何も載せられない点です。`std::optional<T>`は実質`std::expected<T, std::nullopt_t>`であり、エラーが起きたことのみしか通知することができません。上記の特性では「情報」のところが`❌`になります。

`std::expected`は`std::optional`の設計を参考にしているため多くの部分が似通っています。例えば、正常値/エラー値は内部に保持する（動的メモリ確保しない）、正常値/エラー値は領域を共有する（共用体を使用して実装）、要素型のトリビアル性の可能な限りの保持、インターフェース、などがあります。

また、`std::expected`はRustでエラーハンドリングのデファクトのメカニズムとなっている`result`型と意味的に同じ型です。そちらと近い形でエラーハンドリングを書けるようになります（ただし、Rustの`?`演算子に対応するものはまだ用意されていません）。

## 使用可能な型の制限

`std::expected<T, E>`の`T`と`E`にはどんな型でも使用可能であるわけではなく制限があり、なおかつ`T`と`E`で少し異なっています。

|型＼パラメータ|`T`|`E`|
|---|---|---|
|オブジェクト型（配列型を除く）|`⭕`|`⭕`|
|配列型|`❌`|`❌`|
|参照型|`❌`|`❌`|
|関数型|`❌`|`❌`|
|`void`|`⭕`|`❌`|
|`in_place_t`/`unexpect_t`|`❌`|`⭕`|
|`unexpected`特殊化|`❌`|`❌`|
|CV修飾|`⭕`|`❌`|

ここでのオブジェクト型は次のいずれかに該当する型です

- 算術型
    - 整数型と浮動小数点数型及び列挙型
- ポインタ型
    - メンバポインタ及び`std::nullptr`を含む
- クラス型
- 共用体型

`T`の方は`void`を使用することができるほか、`const`/`volatile`修飾されている型を使用することができます。`E`の方は`in_place_t`/`unexpect_t`が使用できる点が`T`より寛容な部分ですが、これらはどちらもコンストラクタを呼び分けるためのタグ型でしかないためあまり役には立たなさそうです。

そして、どちらの場合も参照型を使用することはできません。

## 値へのアクセス

`std::expected`を受け取った側ではまず、その状態確認のために`bool`変換演算子あるいは`.has_value()`メンバ関数を使用できます。どちらも、`true`を返す場合は正常であり、`false`を返す場合はエラー状態です。

```cpp
// エラーコード列挙型
enum class myerrc {
  ...
};

// エラーが起こるかもしれない処理
auto maybe_error() -> std::expected<double, myerrc>;

int main() {
  // 処理を実行
  auto res = maybe_error();

  // エラーチェック（has_value()も使用可能
  if (res.has_value()) {
    // ✔ 正常時のパス
    ...
  } else {
    // ❌ エラー時のパス
    ...
  }
}
```

このエラーチェックの後、それぞれのパスで値にアクセスするには専用の関数を使用します。

まず正常値にアクセスするには`std::optional`と同じインターフェース、すなわち`operator*`もしくは`.value()`を使用します。

```cpp
if (res.has_value()) {
  // ✔ 正常時のパス

  // 正常値へのアクセス
  auto& value1 = *res;
  auto& value2 = res.value();

  ...
} else {
  // ❌ エラー時のパス
  ...
}
```

この2つの関数の違いも`std::optional`と同様で、`operator*`は内部で正常値へのアクセスが安全か（エラー状態ではないか）をチェックせずにアクセスを行い、`.value()`は内部でアクセスが安全かチェックを行ったうえで安全ではない場合（`.has_value() == false`）は例外を投げます。

```cpp
if (res.has_value() == false) {
  // 正常値へのアクセス（エラー状態なので正しくない
  auto& value1 = *res;        // 💀 UB
  auto& value2 = res.value(); // ok、例外が送出される
}
```

次に、エラー値にアクセスするためには`.error()`を使用します。

```cpp
if (res.has_value()) {
  // ✔ 正常時のパス
  ...
} else {
  // ❌ エラー時のパス
  
  // エラー値へのアクセス
  auto& err = res.error();
}
```

`.error()`関数は内部でエラー状態かのチェックを行わずにエラー値にアクセスします。従って、エラー状態ではない（`.has_value() == true`）場合は未定義動作になります。なお、エラー値の取得のための`operator*`に相当する物はありません。

```cpp
if (res.has_value() == true) {
  // エラー値へのアクセス（エラー状態ではないので正しくない
  auto& err = res.error();  // 💀 UB
}
```

これは、`std::expected`の性質上エラー値に明示的にアクセスするのはエラーチェックを行っている場合であることがほぼ仮定できるためだと思われます（エラーチェックを行わずに正常値アクセスをすることはあっても、エラーチェックを行わずにエラー値アクセスするケースは稀なはず）。いずれにせよ、`std::expected`が得られている場合は必ずエラーチェックは行いましょう（そうしないと意味がないです）。

### `.value_or()`/`.error_or()`

`std::optional`には`.value_or()`というアクセス関数も用意されており、これはエラー状態を特定の正常値に変換したうえで正常値として値を取得する関数です。

`std::expected`にも同様の`.value_or()`は用意されており、さらにエラー値を主体とするバージョンの`.error_or()`も用意されています。

```cpp
// 処理結果を表すexpectedを取得
std::expected<double, myerrc> res = maybe_error();

// 正常値はそのまま、エラー値はNaNとして取得
double valid_res = res.value_or(std::numeric_limits<double>::quiet_NaN());

// エラー値はそのまま、正常値は特定の列挙値として取得
myerrc error_res = res.error_or(myerrc::done);
```

どちらの関数でも渡された引数は完全転送されて`T`/`E`のコンストラクタで変換されて返されます。ただし、その際の変換は暗黙変換のみが考慮されます。また、`.value_or()`における正常値と`.error_or()`におけるエラー値は基本的にコピーされて返されます。

これらの関数の実装は単純に条件演算子を使用したものと同等であり、手書きしても一行で済みます

```cpp
template<typename T, typename E>
class expected {
  ...

  // value_or()実装例
  template<class U = remove_cv_t<T>>
  constexpr T value_or(U&& v) const & {
    return has_value() ? **this : static_cast<T>(std::forward<U>(v));
  }
  
  // error_or()実装例
  template<class G = E>
  constexpr E error_or(G&& e) const & {
    return has_value() ? std::forward<G>(e) : error();
  }
};
```

それでもなおこの関数を使用する利点は、条件の記述を回避できることと関数名による意図の明確化、そして条件演算子を隠蔽することによる可読性の向上などがあります。

## 状態の指定

ここでの状態とは、`std::expected<T, E>`の正常orエラー状態の事です。`std::expected`はそのオブジェクトの生存期間のある時点で正常状態かエラー状態のどちらかを取り、どちらでもない状態や両方の状態にはなりません。そして、正常状態の場合は`T`のオブジェクトを保持しており、エラー状態の場合は`E`のオブジェクトを保持しています。

### 構築

`std::expected<T, E>`が正常とエラーのどちらの状態を取るかは基本的に構築時に決定されます。すなわち、`std::expected`のコンストラクタの呼び方によって状態が決定されます。

まず、`std::expected`をデフォルト構築すると正常状態となり`T`の値をデフォルト構築して保持します。

```cpp
std::expected<T, E> e{}; // Tをデフォルト構築して保持

assert(e.has_value());  // ✔
```

このため、`T`がデフォルト構築可能ではない場合はデフォルトコンストラクタは利用できません。

明示的に`T`の値を渡すと、それを完全転送して`T`の値を構築して保持し正常状態となります。

```cpp
T t{...};
std::expected<T, E> e{t}; // tをコピーして保持

assert(e.has_value());  // ✔
```

エラー状態で構築するためには`std::unexpect`タグを使用します。

```cpp
std::expected<T, E> e1{std::unexpect};  // Eをデフォルト構築して保持

E e{...};
std::expected<T, E> e2{std::unexpect, e}; // eをコピーして保持

assert(e1.has_value() == false);  // ✔
assert(e2.has_value() == false);  // ✔
```

正常状態で構築する場合は、`std::in_place`を使用することもできます。

```cpp
std::expected<T, E> e{std::in_place, args...}; // argsからTをin_place構築して保持

assert(e.has_value());  // ✔
```

`std::expected<T, E>`はコピーコンストラクタとムーブコンストラクタを持ちますが、いずれも`T`と`E`の両方がコピー構築/ムーブ構築可能でないと利用できません。

```cpp
using non_copy_expected = std::expected<std::unique_ptr<int>, int>;
non_copy_expected src{};

non_copy_expected e1{src};             // ng、Tがコピー不可
non_copy_expected e2{std::move(src)};  // ok
```

`std::expected<U, G>` -> `std::expected<T, E>`の変換コンストラクタも提供されています。このコンストラクタも、`U -> T`と`G -> E`の変換が両方とも有効でないと利用できません。

```cpp
struct no_convert{};

void example(std::expected<float, int> src) {
  std::expected<double, int> e1{src};         // ok
  std::expected<float, std::int64_t> e2{src}; // ok

  std::expected<no_convert, int> e3{src};    // ng
  std::expected<float, no_convert> e3{src};  // ng
}
```

### `std::unexpected`

`std::expected`をエラー状態で構築するためのもう一つの方法として、`std::unexpected`というラッパクラス型が用意されています。エラー値をこの値で包んで`std::expected`のコンストラクタに渡すことでそのエラー値を保持した`expected`オブジェクトが得られます。

```cpp
std::expected<T, E> e1{std::unexpected{}};  // Eをデフォルト構築して保持

E e{...};
std::expected<T, E> e2{std::unexpected{e}}; // eをコピーして保持

assert(e1.has_value() == false);  // ✔
assert(e2.has_value() == false);  // ✔
```

これだと`std::unexpect`タグを使用する場合とほぼ変わりませんが、`std::unexpected`は`expected`を返す関数の`return`文で使用する際に便利です。例えば次のような場合

```cpp
auto f() -> std::expected<T, E> {
  // 何らかの状態に応じてエラーを返す
  if (condition()) {
    return T{...};  // ok
  } else {
    return E{...};  // ng
    return std::expected<T, E>{ std::unexpect, E{...}};  // ok
  }
}
```

正常値である`T`の値は素直にそのまま`return`に渡すことができるのに、エラー値である`E`の値はできません。先ほど見たように`std::unexpect`タグと明示的なコンストラクタ呼び出しが必要になります。これだとエラー値の構築がかなり冗長かつ構文的に重くなり、`std::expected<T, E>`を使用すればするほどこの典型的なコードは散見されるようになります。

そこで`std::unexpected`を使用することでかなりすっきりと書くことができるようになります。

```cpp
auto f() -> std::expected<T, E> {
  // 何らかの状態に応じてエラーを返す
  if (condition()) {
    return T{...};  // ok
  } else {
    return std::unexpected{E{...}};  // ok
  }
}
```

構文的な冗長さが削減されるとともに、`unexpected`という目立つ型名によってエラーケースの`return`文であることが視覚的に分かりやすくなります。

`std::unexpected`を使用する場合、エラー値の構築のために`std::in_place`を使用することができます

```cpp
auto f() -> std::expected<T, E> {
  ...

  return std::unexpected{std::in_place, args...}; // argsからEをin_place構築して保持
}
```

### 代入

構築された後の`expected`オブジェクトの正常/エラー状態を変更するには、代入を使用します。正常値もしくは`std::unexpect`を代入することで、`expected`オブジェクトの状態を正常/エラー状態にして、代入したものを保持させることができます。

```cpp
auto f() -> std::expected<T, E>;

int main() {
  auto result = f();

  // エラー状態として
  assert(result.has_value() == false);

  result = T{...};  // 正常値を代入して正常状態にする

  assert(result.has_value()); // ✔

  result = std::enexpect(E{...}); // エラー値を代入してエラー状態にする

  assert(result.has_value() == false); // ✔
}
```

正常値はそのまま代入できますが、エラー値を代入するには`std::enexpect`でラップしなければなりません。

`expected`オブジェクトに正常値を保持させるもう一つの方法として、`.emplace()`が利用できます。

```cpp
auto f() -> std::expected<T, E>;

int main() {
  auto result = f();

  // エラー状態として
  assert(result.has_value() == false);

  result.emplace(args...);  // argsからTを内部で構築して保持、正常状態となる
}
```

`.emplace()`はエラー値構築のためには利用できません。

代入/`.emplace()`による構築後の状態変更操作は、必ずしも状態変更を伴わなくても利用することができます。すなわち、正常値を保持した状態で別の正常値を代入（`.emplace()`）することができ、エラー値を保持した状態で別のエラー値を代入することができます。

いずれのケースにおいても代入/`.emplace()`の操作においてはまず、現在保持している値（正常値/エラー値）を破棄してから、値の代入/構築を行います。

なお、`std::expected`は空の状態を取らない（常に必ず正常/エラーのどちらかの状態にある）ため、この代入/`.emplace()`操作は`T, E`のコピー/ムーブコンストラクタが`noexcept`ではない場合に利用できないようにされています。特に、`.emplace()`は無条件で`noexcept`指定されています。

## モナドインターフェース

`if`による分岐は人間のミスが入りやすいため、書かなくて済むなら書かないに越したことはありません。条件演算子はその可読性の低さによって殊更にバグを導入しやすい場所となります。`std::expected`のエラー判定のようなかなり固定的なパターンの条件分岐は手書きするのではなく、どこか一箇所に（テストと共に）書いておいてそれを使いまわすことで条件記述ミスのリスクを減らすことができます。

`std::expected`にはその状態に応じて特定の処理を実行させるためのある種のVisitorインターフェースが用意されており、これらのものはモナドインターフェースと呼ばれます。モナドインターフェースを使用すると、コードの表面上から分岐を隠蔽しよりクリーンかつ高機能なエラーハンドリングを記述することができます。

モナドインターフェースはいずれも、`expected`のどちらかの状態の場合に実行したい処理をCallbleの形で受け取り、`expected`がその状態にあれば渡された関数を実行してその戻り値を返します。そして、モナドインターフェースの戻り値型はまた`expected`であることによってそのように渡した処理そのもののエラーもハンドリングすることができるとともに、得られた`expected`オブジェクトに対する継続処理をメソッドチェーンの形で記述していくことができます。

### `.and_then()`

`.and_then()`は、`expected`が正常値を持っている場合に渡された処理を実行し、そうでないなら何もせずにエラー状態をそのまま伝播させるものです。

これは、渡す関数の戻り値型の`std::expected<U, E>`によって`std::expected<T, E> -> std::expected<U, E>`のような変換を行います。この時、`U`は`T`と異なっていても構いません。

```cpp
// ファイルを開く
auto try_open_file(std::filesystem::path file)
  -> std::expected<std::ifstream, myerrc>;

// ファイルの内容を読み込む
auto read_file_content(std::ifstream ifs) -> std::expected<std::string, myerrc>;

void example() {
  // ファイルを開いて内容を出力してから返す
  auto res = try_open_file("file.txt")
                .and_then(read_file_content)
                .and_then([](auto file_str) {
                  std::println("file content: {:s}", file_str);
                  return std::expected<std::string, myerrc>{std::move(file_str)};
                 });

  ...
}
```

`.and_then()`ではこのように、`std::expected`が成功状態の場合に実行するべき処理を指定していきます。処理の指定方法は1つ目のように関数ポインタ（参照）を渡したり、2つ目の様にラムダ式で渡すこともできます。内部的には`std::invoke()`を使用して呼び出しを行っているため、ある程度の柔軟性があります。

`.and_then()`に渡す処理は再び`std::expected`を返さなければなりません。その際、正常値の値型（`std::expected<T, E>`の`T`）を変更することはできますがエラー型（`E`）を変更することはできず、エラー値は元の値がそのまま`.and_then()`の戻り値に伝播されます。例えば上の例では最初の`try_open_file()`が返したエラーは`.and_then()`2つのチェーン実行後の戻り値`res`まで保持されています。

### `.or_else()`

`.or_else()`は、`expected`がエラー値を持っている場合に渡された処理を実行し、そうでないなら何もせずに正常状態をそのまま伝播させるものです。

これは、関数の戻り値型`std::expected<T, G>`によって`std::expected<T, E> -> std::expected<T, G>`のような変換を行います。この時、`G`は`E`と異なっていても構いません。

```cpp
// ファイルを開く
auto try_open_file(std::filesystem::path file)
  -> std::expected<std::ifstream, myerrc>;

// ファイルオープンエラーをハンドリング、
auto open_error(myerrc err)
  -> std::expected<std::ifstream, std::string>;

void example() {
  // ファイルを開いてエラーを処理する
  auto res = try_open_file("file.txt")
                .or_else(open_error)
                .or_else([](auto error_str) {
                  std::println("file error: {:s}", error_str);
                  return std::expected<std::ifstream, std::monostate>{std::unexpect};
                 });

  ...
}
```

この関数も内部的には`std::invoke()`を使用して呼び出しを行っているため、ラムダ式などを渡すことができます。

`.or_else()`に渡す処理も再び`std::expected`を返さなければなりません。その際、エラー型（`E`）を変更することはできますが正常値の値型（`T`）を変更することはできず、正常値は元の値がそのまま`.or_else()`の戻り値に伝播されます。例えば上の例では最初の`try_open_file()`が成功した場合、返された`ifstream`オブジェクトは`.or_else()`2つのチェーン実行後の戻り値`res`まで保持されています。

この例では2つ目の`.or_else()`でエラーハンドリングを完了したとみなして、以降のエラー値がスルー可能であることを表すために`std::monostate`をエラー型として使用しています。`std::monostate`はオブジェクトとして扱える`void`のような型であり、`E`に`void`が使用できず`.or_else()`に渡す関数も`void`を返せないために使用しています。

### `.transform()`

`.transform()`は`expected`が正常値を持っている場合に渡された処理を実行し、そうでないなら何もせずにエラー状態をそのまま伝播させるものです。

これは、渡す関数の戻り値型の`U`によって`std::expected<T, E> -> std::expected<U, E>`のような変換を行います。この時、`U`は`T`と異なっていても構いません。

```cpp
// ファイルを開く
auto try_open_file(std::filesystem::path file)
  -> std::expected<std::ifstream, myerrc>;

// ファイルの内容を読み込む
auto read_file_content(std::ifstream ifs) -> std::string;

void example() {
  // ファイルを開いて内容を出力してから返す
  auto res = try_open_file("file.txt")
                .transform(read_file_content)
                .transform([](auto file_str) {
                  std::println("file content: {:s}", file_str);
                  return file_str;
                 });

  ...
}
```

この関数は`.and_then()`とよく似ており、一見すると違いが分からない所があります。`.and_then()`との違いは渡す処理の戻り値型の制限にあり、`.and_then()`に渡す処理が`expected`を返さないといけないのに対して、`.transform()`に渡すものは任意の型を返すようにすることができます。しかし、`.transform()`事態の戻り値型はまた`expected`になります。

これによって、`.transform()`はその名前の通りに`expected`の正常値型を変換するという動作に特化しています。

なお、`.transform()`に渡す処理は`void`戻り値型にすることもできます。この場合、`.transform()`の戻り値型は`expected<void, E>`になります。

```cpp
// ファイルを開く
auto try_open_file(std::filesystem::path file)
  -> std::expected<std::ifstream, myerrc>;

// ファイルの内容を読み込む
auto read_file_content(std::ifstream ifs) -> std::string;

void example() {
  // ファイルを開いて内容を出力する
  try_open_file("file.txt")
    .transform(read_file_content)
    .transform([](auto file_str) -> void {
      std::println("file content: {:s}", file_str);
    });
  ...
}
```

### `.transform_error()`

`.transform_error()`は`expected`がエラー値を持っている場合に渡された処理を実行し、そうでないなら何もせずに正常状態をそのまま伝播させるものです。

これは、関数の戻り値型`G`によって`std::expected<T, E> -> std::expected<T, G>`のような変換を行います。この時、`G`は`E`と異なっていても構いません。

```cpp
// ファイルを開く
auto try_open_file(std::filesystem::path file)
  -> std::expected<std::ifstream, myerrc>;

// ファイルオープンエラーをハンドリング、
auto open_error(myerrc err)
  -> std::expected<std::ifstream, std::string>;

void example() {
  // ファイルを開いてエラーを処理する
  auto res = try_open_file("file.txt")
                .transform_error(open_error)
                .transform_error([](auto error_str) {
                  std::println("file error: {:s}", error_str);
                  return error_str;
                 });

  ...
}
```

この関数も`.or_else()`とよく似ており、その違いは`.transform()`の場合と同様に渡す処理の戻り値型として任意の型を返すようにすることができるところにあります。`.or_else()`に渡す処理の戻り値型は`expected`でなければなりませんが、`.transform_error()`に渡すものは任意の型を返すようにすることができます。

`.transform_error()`もまたその名前の通りに`expected`のエラー値型を変換するという動作に特化しています。

`std::expected`はエラー型に`void`を指定できないため、`.transform_error()`に渡す処理は`void`戻り値型にすることができません。これをやりたい場合は、先程同様に`std::monostate`を使用すると良いでしょう。

```cpp
// ファイルを開く
auto try_open_file(std::filesystem::path file)
  -> std::expected<std::ifstream, myerrc>;

// ファイルオープンエラーをハンドリング、
auto open_error(myerrc err) -> std::string;

void example() {
  // ファイルを開いてエラーを処理する
  try_open_file("file.txt")
    .transform_error(open_error)
    .transform_error([](auto error_str) -> std::monostate {
      std::println("file error: {:s}", error_str);
      return {};
     });

  ...
}
```

### 2種類のインターフェースの違い

`expected`のモナドインターフェースは`and_then/or_else`と`transform/transform_error`の2種類に分けることができます。どちらもよく似た使用方法を持ちよく似た動作をしますが、前者は渡す処理が`expected`を返さなければならないのに対して、後者はその制限がありません。従って、通常使用しやすいのは`transform/transform_error`の方でしょう。

`and_then/or_else`の利点は、渡した処理内部のエラーもしくはエラー復帰を`expected`として返すことができるところにあります。

```cpp
// ファイルを開く
auto try_open_file(std::filesystem::path file)
  -> std::expected<std::ifstream, myerrc>;

// エラー文字列変換
auto to_err_string(myerrc e)
  -> std::expected<std::ifstream, std::string>
{
  if (e == myerrc::special_case) {
    // 特定のエラーでは、予め用意されたファイルを読んで返す
    // 以降正常系へ復帰する
    return std::ifstream{"special_file.txt"};
  }

  ...

  // エラー値を文字列変換して返す
  return { std::unexpect, enum_to_str(e) };
}

// ファイルを読みだす
auto read_file_content(std::ifstream ifs)
  -> std::expected<std::string, std::string>
{
  std::string line;
  std::string content;

  // 読み出し内容の確認などでエラーを検知し報告する
  if (!std::getline(ifs, line)) {
    return { std::unexpect, "strem read error." };
  }
  if (!validation(line)) {
    return { std::unexpect, "invalid file format." };
  }

  ...

  // 読み出し内容を返して正常終了
  return content;
}

void example() {
  // ファイルを開いて内容を読み取って返す
  auto res = try_open_file("file.txt")
                .or_else(to_err_string)
                .and_then(read_file_content);

  ...
}
```

`transform/transform_error`は正常/エラー状態の型を変更することはできますが、その呼び出し前後で`expected`の正常/エラー状態を変更することはできません。そのため、渡した処理の内部でのエラー/復帰を`expected`として返すことはできません。一方で、`and_then/or_else`は渡す処理が直接`expected`を返すため内部でのエラー/復帰を`expected`として返すことができ、その呼び出し前後で正常/エラー型の変更と同時に`expected`の正常/エラー状態を変更することができます。

### メソッドチェーン

これらの4つのモナドインターフェースはいずれも戻り値が`expected`で返ります。そのため、モナドインターフェースの呼び出しはメソッドチェーンの形で連鎖的に使用することができます。

```cpp
// ファイルを開く
auto attempt_to_open(std::filesystem::path file)
  -> std::expected<std::ifstream, myerrc>;

// ファイルオープンエラーをハンドリング、
auto attempt_recovery(myerrc err)
  -> std::expected<std::ifstream, myerrc>;

// ファイルの内容を読み込む
auto extract_content(std::ifstream ifs)
  -> std::string;

// 読み取った内容を検証
auto ensure_valid_content(std::string file_data)
  -> std::expected<std::string, myerrc>;

// 一連のエラーを文字列化
auto render_error(myerrc err) -> std::string;


void example() {
  // ファイルを開いて内容を読み取って返す
  auto res = attempt_to_open("file.txt")
                .or_else(attempt_recovery)
                .transform(extract_content)
                .and_then(ensure_valid_content)
                .transform_error(render_error);

  if (res.has_value() == false) {
    // エラー報告
    std::println("file read error: {:s}", res.error());
    return;
  }

  ...
}
```

モナドインターフェースのメソッドチェーンを使用すると、途中で必要な`if`による`expected`状態チェックとその後の中身の取り出し、処理の結果によって`expected`を再構築して次に渡す、のようなお決まりの処理を省略することができます。さらに、渡す関数名を工夫することによってエラーを起こしうる処理の流れを宣言的に記述することができます。

## `std::expected<void, E>`

`std::expected<T, E>`の`T`は`void`にすることができます（`E`はできません）。これは、`void`戻り値型関数における例外やエラーコード等によるエラー伝達手段からの移行先として利用できます。

```cpp
// 戻り値なしで例外によってエラー報告する関数
void f1(int n) {
  if (n == 0) {
    throw std::runtime_error{"n must be non zero."};
  }

  ...
}

// ↑をexpectedで書き直す
auto f2(int n) -> std::expected<void, myerrc> {
  if (n == 0) {
    return std::unexpect{myerrc::zero_input};
  }

  ...

  return {};
}
```

`std::expected<void, E>`は通常の`std::expected<T, E>`に対して部分特殊化によって定義されており、これによって正常値に関するインターフェースが`void`型に特化したものに置き換えられています。

例えば`operator*`や`.value()`は利用可能なものの戻り値型が`void`になっており、`.emplace()`は引数を取らなくなっています。そして、`.value_or()`や正常値を渡せるコンストラクタ/代入演算子などは利用できません。

モナドインターフェースは4種類全てが利用できますが、正常値の場合の関数（`.and_then()`/`.transform()`）は渡す関数が引数無しである必要があります。

```cpp
// さっきのf2()
auto f2(int n) -> std::expected<void, myerrc>;

int main() {
  auto res = f2(0);

  // 状態判定方法は共通
  if (res.has_value()) {
    // 正常値の取得関数は使用可能だが戻り値はない
    *res;
    res.value();
  } else {
    // エラー値取得は全く同じ
    auto& err = res.error();
  }

  // 正常時動作系のモナドインターフェースは引数無し
  res.and_then([] {
    ...
  });
  res.transform([] {
    ...
  });

  // エラー時動作系は通常と同じ
  res.or_else([](myerrc& e) {
    ...
  });
}
```

例外送出やエラーコードを返すことによってエラーを報告していた戻り値のない関数をわざわざ`expected<void, E>`で書き換える利点は、モナドインターフェースをはじめとする`expected<void, E>`のエラーハンドリング機能を利用できるようになる点にあります。

## `std::bad_expected_access<E>`

`.value()`によって正常値を取得しようとする場合、`expected`オブジェクトの状態がチェックされエラー状態にあれば例外が送出されます。

```cpp
auto f() -> std::expected<T, E>;

int main() {
  auto res = f();

  auto& v = res.value();  // エラー状態の場合例外が投げられる
}
```

この場合に送出される例外型は`std::bad_expected_access<E>`という型です。この例外型はクラステンプレートであり、テンプレートパラメータとして例外送出元の`expected<T, E>`の`E`を取ります。`std::bad_expected_access<E>`は`std::exception`の派生型でありその他の例外オブジェクトと同様に扱うことができますが、それに加えて`.error()`によってエラー値を取得することができます。

```cpp
auto f() -> std::expected<T, E>;

int main() {
  auto res = f();

  try {
    auto& v = res.value();  // エラー状態の場合例外が投げられる

    ...
  } catch (const std::bad_expected_access<E>& ex) {
    auto& err = ex.error();   // 送出元のexpectedが保持してたエラー値を取得できる

    auto err_str = ex.what(); // std::exceptionのインターフェースも利用可能
    ...
  }
}
```

`std::bad_expected_access<E>`が保持するエラー値は、それを送出する元になった`expected`オブジェクトがその時点で保持しているエラー値をコピー（右辺値の場合はムーブ）したものです。

```cpp
enum class myerrc : int {
  error = -1,
  success = 0,
};

auto f() -> std::expected<int, myerrc> {
  return std::unexpected{myerrc::error};
}

int main() {
  try {
    auto res = f();
    auto v = res.value(); // 例外
  } catch(const std::bad_expected_access<myerrc>& ex) {
    auto e = ex.error();

    assert(e == myerrc::error); // ✔
  }
}
```

`std::bad_expected_access<E>`によって例外を`catch`するためには、送出元の`expected<T, E>`の`E`型を知っている必要があります。`std::expected`を内部的に重層的に扱っている関数の呼び出しだったり、別の翻訳単位にある関数（必ずしもソースコードを見ることができない）などでは、`E`の型を求めるのが簡単ではない場合があります。

その場合に`expected`由来の例外を弁別してキャッチするために`std::bad_expected_access<void>`が利用できます。

```cpp
auto f() -> std::expected<T, E>;

int main() {
  try {
    auto v = f().value();
  } catch (const std::bad_expected_access<E>& ex) {
    // エラー型Eのexpected由来の例外をキャッチ
  } catch (const std::bad_expected_access<void>& ex) {
    // E以外のエラー型をもつexpected由来の例外をキャッチ
  } catch (const std::exception& ex) {
    // expected::value()以外の原因の例外をキャッチ
  }
}
```

`std::bad_expected_access<void>`は`std::bad_expected_access<E>`の基底クラスとなっているため、`std::bad_expected_access<void>`は`E`に関わらず`std::bad_expected_access`全般をキャッチすることができます。

```cpp
namespace std {

  template<>
  class bad_expected_access<void> : public exception {
    ...
  };

  template<class E>
  class bad_expected_access : public bad_expected_access<void> {
    ...
  };
}
```

# `<flat_map>`/`<flat_set>`

# `<mdspan>`

# `<spanstream>`

# `<stacktrace>`

# `<stdatomic.h>`

\clearpage

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- Compiler Explorer(https://godbolt.org/)
