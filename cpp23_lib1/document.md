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

`std::map`や`std::set`などの（順序付き）連想コンテナはノードベースのコンテナであり、各要素はノードという単位でメモリ上に散らばっています。各ノードは`Key`とそれに対応する`Value`に加えて、子のノードへのポインタを2つ保持しています。これらノードを適切な順番でリンクすることで二分探索木（おそらくほとんどの場合赤黒木）を構成しています。

連想コンテナはその特性上`Key`による要素探索によく使用されますが、それぞれの要素（=ノード）がメモリ上に散らばっているためキャッシュ効率が悪く（キャッシュに載りづらい）、検索パフォーマンスが良くないという問題があります。場合によっては、`std::vector`に要素を配置して先頭から線形探索した方が検索が早くなることすらあります。

`std::flat_map`/`std::flat_set`は、このような課題に対応するために新しく追加された連想コンテナです。これらは、`std::vector`のキャッシュ効率の高さと`std::map`の探索の効率性（log計算量）を同時に併せ持っています。

```cpp
#include <flat_map>

int main() {
  std::flat_map<int, std::string_view> map = {
    {1, "one"},
    {18, "eighteen"},
    {6, "six"},
    {3, "three"}
  };

  std::string_view one = map[0];  // "one"

  bool b = map.contains(5) ;  // false
}
```

`std::flat_map`/`std::flat_set`は連想コンテナであり、基本的な使用方法やインターフェースなどは`std::map`/`std::set`に準じています。しかし、内部構造は大きく異なっており、検索の効率性の向上に重点が置かれています。

## 内部構造

`std::flat_map`を代表として、これらの`flat`な連想コンテナの内部構造を見ていきます。

`std::flat_map`をはじめとする`flat`な連想コンテナの基本的な構造は、Key-Valueペアのソート済み配列です。

例えば`{1, "one"}, {3, "three"}, {5, "five"}...`のような要素列（数字をキーとする）がある時、内部のソート済み配列は次のようになります

```
Key-Value: [{1: "one"}, {3: "three"}, {5: "five"}...]
```

このようなソート済みの配列に対しては二分探索によって`O(log N)`の計算量で検索を行うことができます。しかもそれぞれの要素はノードによって独立しているのではなく配列要素としてメモリ上で連続しているため、キャッシュ効率を向上させることができ、検索をより高速化できます。

ただし、このためにはソート済みであることを常に維持しなければならず、しかも配列（メモリ上で連続している）であるため、挿入時には挿入位置の引き当てだけではなくそれより後ろの要素をずらす必要があります。さらに、配列サイズが足りなければそのキャパシティを拡大した配列にデータを移さなければなりません。このため、挿入の計算量は`O(N)`になります。

削除の場合は配列の再確保は不要であるものの、要素をずらす必要はあるため、計算量は同様に`O(N)`になります。

`std::map`のようなKey-Valueペアによる連想コンテナの場合、検索はキーに対して行われ、値は見つかったキーに対して引き当てられます。すると、検索時にコンテナ要素のキーとキーの間に値が入っているのはキャッシュ効率の観点からは邪魔です。そこで、キーの配列と値の配列を分けてやることで、検索時のキャッシュ効率を最大化することができます。

先程の例をベースにすると次のような構造になります

``` 
Key: [1, 3, 5]
Value: ["one", "three", "five"]
```

キーと値は配列のインデックスで対応するため、キーを発見したらその値の保存場所はすぐに特定されます。

このキーと値の2つの配列上にソート済み配列を構築して保持するのが、`std::flat_map`の内部構造となります。2つの内部配列は通常`std::vector`によって管理されており、コンストラクタ-挿入-削除を通してその要素数と位置（ソート済み位置）が維持され続けることによって、ソート済み配列による連想コンテナ機能を提供します。

なお、`std::flat_set`は`std::flat_map`に対して`value`の配列を無くしたものです。こちらはよりソート済み配列そのものです。

### `multi`

`std::map`/`std::set`には対応して`std::multimap`/`std::multiset`があります。これらはキーの重複を許す（1つのキーに対して複数の値が対応しても良い）連想コンテナですが、`std::flat_map`/`std::flat_set`にも同様に`std::flat_multimap`/`std::flat_multiset`が用意されています。

`std::flat_multimap`/`std::flat_multiset`の`std::flat_map`/`std::flat_set`に対する内部構造は全く同じで、キーの重複を許すように格納されるのみです。すなわち、同じキーに対する複数の値は異なる要素として連続した領域に格納されています。

```cpp
#include <flat_map>

int main() {
  std::flat_multimap<int, std::string_view> map = {
    {1, "one"},
    {18, "eighteen"},
    {6, "six"}
  };
  
  map.insert({1, "1"});
  map.insert({1, "un"});

  auto len = map.size();  // 5
}
```

この例の場合、内部配列は例えば次のようになっています

```
Key:   [1, 1, 1, 6, 18]
Value: ["one", "1", "un", "six", "eighteen"]
```

`std::flat_multiset`はここから`value`の配列を無くしたものになり、重複要素の格納方法は同様になります。

## 特徴

### 不変条件

4種類の`std::flat_xxx`なコンテナはいずれも次のような不変条件を持っています

- キーと値の個数が常に等しい
    - キー配列と値の配列の要素数は常に等しい
- キーは比較オブジェクトによって昇順にソートされている
- 値の配列の先頭からのオフセット`off`の場所にある値は、キー配列において先頭から`off`の場所にあるキーに関連付けられている

これらはクラスの不変条件であり、コンテナに対するあらゆる操作の前後で保たれるものです。この不変条件は強い例外安全性の保証によって例外送出時でも保たれます（このため、例外送出が起きた後でコンテナが空になる場合があります。

### 計算量

`std::map`と比較した`std::flat_map`の計算量は次のようになります

|操作|`flat_map`|`map`|備考|
|---|---|---|---|
|構築|O(N log N)|O(N log N)|未ソートの場合、Nは入力要素数|
|構築|O(N)|O(N)|ソート済みの場合、Nは入力要素数|
|挿入|O(N)|O(log N)|Nは自身の要素数|
|削除|O(N)|O(log N)|Nは自身の要素数|
|探索|O(log N)|O(log N)|Nは自身の要素数|

構築以外は基本的に単一要素に対する操作になります。複数の要素の場合、その要素数に応じた計算量が足される形になります。

この表からも、探索以外の計算量は既存の連想コンテナに劣ることが分かります。これは主に、内部のソート済み配列のソート済み状態を維持する必要性によります。

探索の計算量も`std::map`と同等であるため、計算量の観点からは`std::flat_map`に優位性は何もないわけですが、上述のように現代のプロセッサにおけるキャッシュ周りの事情に最適化されたデータ配置をしていることによって、`std::map`に対して探索時の優位性が生まれています。

なお、`std::flat_set`と`std::set`および`multi`なコンテナの場合の計算量の違いも同様になります。

### 利用シーン

これらのような性質から`flat`なコンテナの最適な利用シーンは、コンテナの更新頻度が低く探索の頻度が高い、場合であり、その差が大きいほどより効果を発揮します。

コンテナの更新（挿入や削除）の操作はコンテナの要素数に対して線形の時間がかかるだけではなく、内部で要素の再配置が発生する（場合によっては追加のメモリ確保も行われる）ため計算量以上に効率が悪く、さらには取得済みの参照（イテレータ）を無効化する危険性があります。

一方で探索はキーの配列を二分探索するだけであり、キーの配列のメモリ局所性が高いことによってキャッシュヒット率が向上し、計算量以上の効率的な探索を行うことができます。

そのため、全ての要素は構築時に決定され構築した後は参照（探索）のみを行う、という使い方が最も効率的であり理想です。

`flat`なコンテナの構築後に要素を追加する必要がある場合でも、要素の追加のステップと参照のステップを分けることができるのであれば、構築は`std::map`で行い、参照は`flat_map`で行うという運用が有効です。

```cpp
#include <flat_map>

int main() {
  // 要素の追加フェース
  std::flat_map<int, std::string_view> build = {
    {1, "one"},
    {18, "eighteen"},
    {6, "six"}
  };
  
  map.insert({2, "two"});

  std::list<std::pair<int, std::string_view>> list{
    {22, "twenty-two"},
    {0, "zero"},
    {7, "seven"}
  };

  map.insert_range(list);

  // 探索フェーズ
  const std::flat_map<int, std::string_view> lookup{
    std::sorted_unique,
    build.begin(),
    build.end()
  };

  auto& elem1 = lookup.at(1);
  auto it = lookup.find(22);
  bool b = lookup.contains(100);

  ...
}
```

これらの使用可能なインターフェースについての詳細は、次の節で見ていきます。

## インターフェース

`std::flat_xxx`なコンテナは対応する`std::xxx`なコンテナとほぼ同じインターフェースを持っているため、使用感はほぼ同じになるはずです。

### テンプレートパラメータ

`flat_map`/`flat_set`のテンプレートパラメータは次のようになっています

```cpp
namespace std {
  template<class Key, class T,
           class Compare = less<Key>,
           class KeyContainer = vector<Key>,
           class MappedContainer = vector<T>>
  class flat_map {
    ...
  };

  template<class Key,
           class Compare = less<Key>,
           class KeyContainer = vector<Key>>
  class flat_set {
    ...
  };
}
```

`Key, T, Compare`は`std::map`/`std::set`と同じものです、後ろにある`KeyContainer`および`MappedContainer`はそれぞれ`Key`の配列の型と`T`の配列の型を指定します。配列の型はどちらもデフォルトが`std::vector`になっており、多くの場合はここから変更する必要は無いでしょう。

内部コンテナ型は`KeyContainer`/`MappedContainer`を変更することによってカスタマイズすることができ、シーケンスコンテナと呼ばれるタイプのコンテナを使用することができます。`std::vector`の他には例えば`std::dequeue`が使用できます。

`multi`なコンテナは対応する非`multi`なものと同じテンプレートパラメータを持ちます。

### イテレータ

4種類の`std::flat_xxx`なコンテナはいずれもそのイテレータは`random_access_iterator`であり、`range`としては`random_access_range`です。従って、範囲`for`で使用可能であり、`range`アダプタも接続可能です。

また、イテレータに関する性質は次のようになっています

|メンバ型|非`multi`|`multi`|
|---|---|---|
|`iterator`|`random_access_iterator`|同|
|`value_type`|`pair<Key, T>`|`Key`|
|`reference`|`pair<const Key&, T&>`|`Key&`|

明確な規定はありませんが、`flat_map`/`flat_multimap`の場合はおそらく2つの内部配列を`zip_view`で閉じ合わせたイテレータが使用されると思われます。

4種類の`std::flat_xxx`なコンテナはいずれも内部配列はソート済みであり、イテレータは内部配列の先頭から順番に要素を辿っていくものになります。従って、要素のイテレーション順はソート済みの順番になります。

```cpp
#include <flat_map>

int main() {
  std::flat_map<int, std::string_view> map = {
    {1, "one"},
    {18, "eighteen"},
    {6, "six"},
    {3, "three"}
  };

  for (const auto& [key, value] : map) {
    std::println("key: {:<2}, value: {}", key, value);
  }
}
```

この出力は次のようになります

```
key: 1 , value: one
key: 3 , value: three
key: 6 , value: six
key: 18, value: eighteen
```

#### イテレータの無効化

`flat`なコンテナはデフォルトでは内部配列に`std::vector`を使用しており、その内部構造から明らかにイテレータが無効化されるタイミングがかなり多いことがうかがえます。これは通常の連想コンテナがイテレータの無効化に関してかなり安定していることとは対照的です。

```cpp
std::flat_map<int, std::string_view> map = {
  {1, "one"},
  {18, "eighteen"},
  {6, "six"}
};

auto it = map.find(1);

map.insert({3, "three"}); // 全てのイテレータと参照を無効化しうる

auto v = *it; // 💀 この時点でイテレータの使用は安全ではない
```

`flat`なコンテナでは挿入削除いずれの場合も要素の位置が変わりうるため、コンテナに対する変更の後でそれ以前に取得された要素への参照及びイテレータは無効化される可能性があります。

これは`flat`なコンテナの不変条件が原因であるため、通常内部コンテナを変えたとしても変わりません。

### 構築

`std::flat_xxx`なコンテナのコンストラクタは大きく分けると次の5種類が基本となります。

1. デフォルトコンストラクタ
2. 内部配列を初期化するコンストラクタ
3. イテレータペアを受け取るコンストラクタ
4. `range`コンストラクタ
5. `initializer_list`コンストラクタ
6. コピー/ムーブコンストラクタ

```cpp
using flat_map = std::flat_map<int, std::string_view>;
 
// 1. デフォルトコンストラクタ
flat_map map1{};

std::vector<int> key = {3, 1, 5};
std::vector<std::string_view> value = {"three", "one", "five"};
// 2. 内部配列を初期化するコンストラクタ
flat_map map2{key, value};

std::map<int, std::string_view> container = {
  {3, "three"},
  {1, "one"},
  {5, "five"}
};
// 3. イテレータペアを受け取るコンストラクタ
flat_map map3{container.begin(), container.end()};

// 4. rangeコンストラクタ
flat_map map4{std::from_range, container};

// 5. initializer_listコンストラクタ
flat_map map5 = {
  {3, "three"},
  {1, "one"},
  {5, "five"}
};

// 6. コピー/ムーブコンストラクタ
flat_map map6{map5};
flat_map map7{std::move(map5)};
```

デフォルトコンストラクタとコピー/ムーブコンストラクタ以外のコンストラクタでは、構築時に内部配列はソートされます。さらに、`multi`でないコンテナにおいて、2のコンストラクタでは内部配列をソートした後で連続した重複要素列から最初の要素以外を削除し、それ以外のコンストラクタでは、重複要素は先に現れた要素が後で現れる要素に上書きされる形になります。

これらのコンストラクタはいずれも基本的に、それぞれ引数の最後でオプション引数として比較オブジェクトとアロケータを受け取るオーバーロードが用意されています。

```cpp
using flat_map = std::flat_map<int, std::string_view, std::ranges::less, std::pmr::vector<int>, std::pmr::vector<std::string_view>>;

std::ranges::less comp;
std::pmr::polymorphic_allocator<> alloc;

std::pmr::vector<int> key = {3, 1, 5};
std::pmr::vector<std::string_view> value = {"three", "one", "five"};

// 比較オブジェクトを渡す
flat_map map1{comp};
flat_map map2{key, value, comp};

// アロケータを渡す
flat_map map1{alloc};
flat_map map2{key, value, alloc};

// 両方渡す
flat_map map1{comp, alloc};
flat_map map2{key, value, comp, alloc};
```

この2つのオプション引数を同時に指定する場合は、比較オブジェクト->アロケータの順番になります。

アロケータを渡す場合、内部配列はどちらもこの渡されたアロケータを使用するように構築されます。これにはUses-allocator constructionが使用されますが、setではない場合は1つのアロケータでKey/Value2つの要素型を持つコンテナを初期化できる必要があります。上記の例では、`pmr::vector<T>`に対して`polymorphic_allocator<>`はそれを満たしており、内部配列はこの渡されたアロケータを使用しています。

なお、コピー/ムーブコンストラクタには比較オブジェクトを受け取るオーバーロードはありません。これはコピー/ムーブ元から取得するものを使用するためです。

```cpp
using flat_map = std::flat_map<int, std::string_view, std::ranges::less>;

std::ranges::less comp;
flat_map src{...};

// いずれもng
flat_map map1{src, comp};
flat_map map2{std::move(src), comp};
```

さらに、2・3・5のコンストラクタには入力要素列が既にソート済みであることを伝達することのできるコンストラクタが用意されています。これを利用するためには、非`multi`なコンテナの場合は`std::sorted_unique`、`multi`なコンテナの場合は`std::sorted_equivalent`を先頭で渡します。

```cpp
using flat_map = std::flat_map<int, std::string_view>;
using flat_multimap = std::flat_multimap<int, std::string_view>;

std::vector<int> key = {1, 3, 5};
std::vector<std::string_view> value = {"one", "three", "five"};
// 2. 内部配列を初期化するコンストラクタ
flat_map map2{std::sorted_unique, key, value};

// 5. initializer_listコンストラクタ
flat_map map5 = {
  std::sorted_unique, 
  {
    {1, "one"},
    {3, "three"},
    {5, "five"}
  }
};

// multiの場合
flat_multimap mmap2{std::sorted_equivalent, key, value};
flat_multimap mmap5 = {
  std::sorted_equivalent, 
  {
    {1, "one"},
    {3, "three"},
    {5, "five"}
  }
};
```

`std::sorted_unique`/`std::sorted_equivalent`ではないコンストラクタは内部配列のソートを伴うため（非`multi`の場合は追加で重複要素の削除も）、入力要素数`N`とすると（ソートされていない場合）`O(N log N)`の計算量となりますが、`std::sorted_unique`を指定すると`O(N)`の計算量で常に抑えることができます。ただしこの場合、入力がソートされていること（と非`multi`の場合は重複要素が無い事）を保証するのはユーザーの責任となり、ソートされていない入力などは未定義動作となります。

ちなみに、イテレータペアを受け取るコンストラクタはこの`std::sorted_unique`/`std::sorted_equivalent`を使用できるのですが、rangeコンストラクタでは使用できないという違いがあります。

```cpp
using flat_map = std::flat_map<int, std::string_view>;

std::map<int, std::string_view> container = {
  {1, "one"},
  {3, "three"},
  {5, "five"}
};

// 3. イテレータペアを受け取るコンストラクタ
flat_map map3{std::sorted_unique, container.begin(), container.end()};  // ok

// 4. rangeコンストラクタ
flat_map map4{std::sorted_unique, std::from_range, container};  // ng
```

一応ソート済みの入力に対しての計算量は変わりませんが、`std::sorted_unique`を指定する方はそもそもソートを試みることも（非`multi`なコンテナにおいて）重複要素を削除しようとすることもしないため、少しだけ効率的になる可能性があります。

### 追加・挿入

前述のように、`flat`なコンテナにおける挿入操作の計算量はコンテナサイズ`N`に対して線形（`O(N)`）になるので、この節の関数は特に言及がない限り計算量は`O(N)`です。

挿入操作は大きく`insert`系の関数と`emplace`系の関数とに分かれます。前者はコンテナの要素型のオブジェクトを受けてコンテナに追加する操作で、後者は要素型を構築するための引数（典型的にはキーと値を個別に渡す）を受けて要素をコンテナに追加する関数です。

`.insert()`と`.emplace()`がまず基本形となります。

```cpp
using namespace std::literals;

std::flat_map<int, std::string_view> map;

auto [pos, success] = map.insert(std::make_pair(1, "one"sv));
auto [pos, success] = map.emplace(5, "five");
```

連想コンテナ`C<Key, Value>`の要素型（`value_type`）は`std::pair<Key, Value>`となり、`.insert()`はこの要素型オブジェクト（あるいはそれに変換可能な型のオブジェクト）を受けてそれをコンテナに追加し、`.emplace()`はこの要素型のコンストラクタ引数を受けてそこからコンテナに要素を追加しながら構築します。

どちらの関数でも戻り値は同じ意味を持つイテレータと`bool`値のペアになります。`bool`値は要素の挿入に成功した場合にのみ`true`となる一方、イテレータの方は挿入しようとした要素のキーに対応する要素へのイテレータとなります。

```cpp
std::flat_map<int, std::string_view> map = {
  {1, "one"}
};

auto[it1, success1] = map.emplace(5, "five");
assert(success1);         // ✔
assert(it1->first == 5);  // ✔

auto[it2, success2] = map.emplace(1, "one");
assert(!success2);        // ✔
assert(it2->first == 1);  // ✔
```

すなわちイテレータの方は操作の成否に関わらず常に要素を指しています。

これらの挿入操作が失敗する場合というのは非`multi`なコンテナにおいてキーに対応する要素が既に存在している場合です。そのため、`multi`なコンテナにおいてはこれらの関数の戻り値はイテレータのみであり、そのイテレータは新規に挿入された要素へのものになります。

```cpp
std::flat_multimap<int, std::string_view> map = {
  {1, "one"}
};

auto it = map.emplace(1, "1");
assert(it->first == 1);    // ✔
assert(it->second == "1"); // ✔
```

`set`系コンテナの場合`.insert()`の引数はキー型オブジェクトになり、`.emplace()`の引数はキー型コンストラクタ引数となります。

`.emplace()`は上記形式の一種類のみですが、`.insert()`にはいくつかのオーバーロードがあります。

1つは第一引数に挿入位置を取るものですが、この位置のヒントがあっても計算量は変化しないので、通常の`map`等に比べるとこのオーバーロードの恩恵は小さくなります。これはインターフェースの一貫性のために提供されている側面が大きいです。

次は、イテレータ範囲を取るものと初期化子リストを取るものです。

```cpp
std::vector<std::pair<int, std::string_view>> input = {
  {1, "one"},
  {5, "five"},
  {3, "three"}
};

std::flat_map<int, std::string_view> map = {
  {1, "one"}
};

map.insert(input.begin(), input.end());
map.insert({
  {7, "seven"},
  {9, "nine"}
});

std::println("{}", map);
```
```
{1: "one", 3: "three", 5: "five", 7: "seven", 9: "nine"}
```

これらの関数の計算量は、挿入前コンテナサイズを`N`、挿入しようとする要素数を`M`とすると、`O(N + M log M)`となります。

この2種類のオーバーロードにはさらに`sorted_unique`（`sorted_equivalent`）を取るものがあります。こちらの場合、入力要素列が予めソートされている（非`multi`の場合は重複要素がない）ことを仮定することで効率的となり、計算量は`O(N + M)`となります。

```cpp
std::map<int, std::string_view> input = {
  {1, "one"},
  {3, "three"},
  {5, "five"}
};

std::flat_map<int, std::string_view> map = {
  {1, "one"},
  {5, "5"}
};
std::flat_multimap<int, std::string_view> mmap = {
  {1, "one"},
  {5, "5"}
};

map.insert(std::sorted_unique, input.begin(), input.end());
mmap.insert(std::sorted_equivalent, input.begin(), input.end());

std::println("{}", map);
std::println("{}", mmap);
```
```
{1: "one", 3: "three", 5: "5"}
{1: "one", 1: "one", 3: "three", 5: "5", 5: "five"}
```

ただし、コンストラクタ同様に入力要素列の仮定を満たすのは利用者の責任となります。

これら4種類の複数要素を挿入しうる操作は、インプレイスマージを行うために（仮に結果的に挿入する要素が0でも）追加のメモリを確保する可能性があります。すなわち、最初に内部配列を`N+M`長に拡大してそこに入力要素列をコピーしてからソートとマージを行います。

この`.insert()`/`.emplace()`にはいくつかの派生操作があります。

|操作|動作|戻り値|`multi`の場合|
|---|---|---|---|
|`insert_or_assign`|対応するキーの要素があれば上書き|`emplace`/`emplace_hint`と同じ|なし|
|`insert_range`|範囲を挿入する|なし|あり|
|`emplace_hint`|`emplace`と同じ|対応するキーの要素へのイテレータ|あり|
|`try_emplace`|引数への副作用が無い`emplace`|`emplace`/`emplace_hint`と同じ|なし|

この表で戻り値の列が「`emplace`/`emplace_hint`と同じ」となっているのは、位置のヒントを受け取るオーバーロードがあり、そちらの場合の戻り値は`.emplace_hint()`と同じになる、という意味です。

`.insert_or_assign()`は挿入しようとするキーに対応する要素が存在する場合、値を上書きするものです。戻り値は基本的に`.insert()`と同じですが、`bool`値は代入が発生した場合に`false`となります（新規挿入の場合に`true`となる）。

少し注意点ですが、`.insert_or_assign()`の引数の与え方は`.insert()`とも`.emplace()`とも異なっており、キー型オブジェクトとそれに対応する値（に変換可能な）オブジェクトのちょうど2つを渡します。

```cpp
std::flat_map<int, std::string_view> map = {
  {1, "one"}
};

auto[it, success1] = map.insert_or_assign(1, "1");
assert(success1 == false);  // ✔
assert(it->second == "1");  // ✔
```

`.insert_range()`は`from_range`を取るコンストラクタの`insert`版で、任意の`range`を一括で挿入します。重複する要素があると削除されますが、それが起きたかを知る方法はありません。

`.insert_range()`の計算量は、挿入前のコンテナサイズ`N`と挿入する`range`のサイズ`M`に対して`O(N + M log M)`となります。

`.emplace_hint()`は挿入しようとするキーの挿入位置のヒントを与える`.emplace()`であり、通常の連想コンテナでは挿入時の計算量を抑えるために使用されますが、`flat`なコンテナはいずれも挿入時の計算量が常にコンテナサイズに線形のためその恩恵はあまり大きくありません。インターフェース一貫性のために存在している側面が大きいです。

最後の`.try_emplace()`と`.emplace()`（`.emplace_hint()`）の違いは、引数の渡し方およびムーブして渡している引数に関する保証にあります。`.try_emplace()`の場合は挿入が行われなかった場合にムーブして渡している引数は実際にはムーブされないため呼出し元で継続して安全に使用できますが、`.emplace()`の場合はそのような保証がありません。

```cpp
std::flat_map<int, std::unique_ptr<int[]>> map{};
map.emplace(1, std::make_unique<int[]>(1));

auto v1 = std::make_unique<int[]>(1);

auto [it1, success1] = map.emplace(1, std::move(v1));
assert(success1 == false);  // ✔
assert(bool(v1));           // ❌ おそらくほぼ常に
  
auto v2 = std::make_unique<int[]>(1);

auto [it2, success2] = map.try_emplace(1, std::move(v2));
assert(success2 == false);  // ✔
assert(bool(v2));           // ✔ 保証される
```

重複チェックのためにはキーをまず得る必要がありますが`.emplace()`は要素を直接構築するため、その引数からキーだけを先に得ることができず、先に要素を構築しなければならないため、`.emplace()`にはこのような保証がありません。

そのため、`.try_emplace()`の第一引数はキー型（あるいはそれと比較可能な型）のオブジェクトを渡す必要があります。

これと同様の理由により、`flat_set`には`.try_emplace()`は提供されず、`multi`の場合は重複チェックをする必要が無いため提供されません。そのため、`.try_emplace()`は`flat_map`でのみ利用可能です。

ただし、`flat_set`の場合は`.insert()`のHeterogeneous Overload（キーと比較可能な任意の型を取るもの）に限って`.try_emplace()`と同じ保証があります。ただし、Heterogeneous Overloadを有効化するには少し作業が必要になります（後述）。

### 削除

削除の基本は`.erase()`です。これはキーを指定してそのキーと等しい要素を削除します。

```cpp
std::flat_map<int, std::string_view> map = {
  {1, "one"},
  {3, "three"},
  {5, "five"}
};

map.erase(3);

std::println("{}", map);
```
```
{1: "one", 5: "five"}
```

前述のとおり、この関数の計算量は`O(N)`です。

条件を満たす要素をすべてコンテナから削除するための操作として、他のコンテナ同様に`std::erase_if()`が利用できます。

```cpp
std::flat_map<int, std::string_view> map = {
  {1, "one"},
  {3, "three"},
  {2, "two"},
  {5, "five"}
};

auto even = [](auto pair) {
  return pair.first % 2 == 0;
};

auto n = std::erase_if(map, even);

std::println("n = {} : {}", n, map);
```
```
n = 1 : {1: "one", 3: "three", 5: "five"}
```

`std::erase_if()`の第二引数には削除条件を指定する述語オブジェクトを渡し、戻り値からは削除した要素の数が得られます。また、渡す述語にはキーではなくコンテナの要素そのものが渡されるため、非`multi`なコンテナでは`pair<const Key&, const T&>`が渡されます。

`std::erase_if()`の計算量は、要素数`N`に対してちょうど`N`回の述語の適用となります。

### 要素アクセス（検索）

前述のように、`flat`なコンテナにおける検索の計算量はコンテナサイズ`N`に対して`O(log N)`になるので、この節の関数は特に言及がない限り計算量は`O(log N)`になります。。

まず、非`multi`なコンテナ内の特定の要素にアクセスするには`[]`か`.at()`を使用するのが基本です。

```cpp
std::flat_map<int, std::string_view> map = {
  {1, "one"},
  {3, "three"},
  {5, "five"}
};

auto& elem1 = map[1];
auto& elem2 = map.at(3);
```

これらの関数の動作は通常の連想コンテナと全く同じであり、`[]`は指定されたキーに対応する要素が存在しない場合に新しい要素を挿入し、`.at()`は指定されたキーに対応する要素が存在しない場合に例外を投げます。

```cpp
const std::flat_map<int, std::string_view> map = {
  {1, "one"},
  {3, "three"},
  {5, "five"}
};

auto& elem1 = map[1];     // ng、constで使用できない
auto& elem2 = map.at(2);  // 例外を送出する
```

`[]`と`.at()`のこの違いにより、計算量も微妙に異なります。どちらも検索そのものの計算量は`O(log N)`なのですが、`[]`で要素の追加が発生する場合は追加で`O(N)`の計算量がかかります。

存在しないキーに対して例外を送出せず、自動挿入もしない検索方法としては`.find()`が利用できます。こちらは`multi`なコンテナでも利用可能です。

```cpp
const std::flat_map<int, std::string_view> map = {
  {1, "one"},
  {3, "three"},
  {5, "five"}
};

auto it = map.find(1);  // 見つかると該当要素のイテレータが返る

if (it == map.end()) {
  // 見つかる場合
  auto elem = *it;
  ...
} else {
  // 見つからない場合
  ...
}
```

単に要素の存在判定だけを行いたい場合は`.contains()`が利用できます。

```cpp
const std::flat_map<int, std::string_view> map = {
  {1, "one"},
  {3, "three"},
  {5, "five"}
};

bool b1 = map.contains(1);  // true
bool b2 = map.contains(2);  // false
```

また、`.count()`を使用すると指定されたキーに対応する要素がいくつかあるかを取得することができます。

```cpp
const std::flat_map<int, std::string_view> map = {
  {1, "one"},
  {1, "one"},
};
const std::flat_multimap<int, std::string_view> mmap = {
  {1, "one"},
  {1, "one"},
};

std::size_t count1 =  map.count(1); // 1
std::size_t count2 = mmap.count(1); // 2
```

これは`multi`なコンテナの場合のみ2以上になり、非`multi`なコンテナにおいては`.contains()`と同じ意味になります。

最後に、ソート済み範囲に対するアルゴリズムである`.lower_bound()`、`.upper_bound()`および`.equal_range()`をメンバ関数として利用できます。

```cpp
const std::flat_map<int, std::string_view> map = {
 {1, "one"},
 {18, "eighteen"},
 {6, "six"},
 {3, "three"}
};

auto lbit = map.lower_bound(6);
auto ubit = map.upper_bound(6);
auto [lbit2, ubit2] = map.equal_range(6);

std::println("lower_bound: {}", *lbit);
std::println("upper_bound: {}", *ubit);
std::println("equal_range: [{}, {}]", *lbit2, *ubit2);
```
```
lower_bound: (6, "six")
upper_bound: (18, "eighteen")
equal_range: [(6, "six"), (18, "eighteen")]
```

`.lower_bound()`は指定されたキーの値を超えない最大のキーを持つ要素をルックアップし、`.upper_bound()`は指定されたキーの値を超える最小のキーを持つ要素をルックアップします。そして、`.equal_range()`はその2つの要素を同時にルックアップします。

`.count()`同様に、`.equal_range()`は非`multi`なコンテナでは返されるイテレータ範囲に入る要素は最大で1つですが、`multi`なコンテナでは複数の要素が入り得ます。

```cpp
const std::flat_multimap<int, std::string_view> map = {
 {1, "one"},
 {18, "eighteen"},
 {6, "six"},
 {6, "6"},
 {6, "roku"},
 {3, "three"}
};

auto [lbit, ubit] = map.equal_range(6);
std::println("equal_range: {}", std::ranges::subrange(lbit, ubit));
```
```
equal_range: [(6, "six"), (6, "6"), (6, "roku")]
```

この3つの検索関数は動作も計算量もフリー関数の同名アルゴリズムと同じですが、キーの指定のみで使用できるので若干簡単に使用できます。

### Heterogeneous Overload

Heterogeneous Overloadとはテンプレートパラメータで指定されたキー型以外の型の値を使用して要素の引き当て（検索・削除）を行うことができる機能です。特に、キー型への暗黙変換を回避してキー型と指定した型の値を直接的に比較するため、より便利かつ効率的になります。

典型的なユースケースとして、`std::string`をキーとするコンテナに対して`std::string_view`や文字列リテラルを使用して値の検索等を行いたい場合があります。

```cpp
std::flat_map<std::string, int> map = {
  {"one", 1},
  {"three", 3},
  {"five", 5}
};

std::string_view key = "five";

auto& elem1 = map.at(key);    // ng string_view -> stringへ暗黙変換不可
auto& elem2 = map.at("one");  // ok、一時オブジェクトが作成される
```

しかし、他の連想コンテナ同様にこの操作はデフォルトで有効になっていません。連想コンテナにおいてこれを有効化するためには比較関数オブジェクト型`Compare`が入れ子型の`is_transparent`を持つようにする必要があります。

`std::less<T>`はこれを持ちませんが、`std::less<void>`はこれを持っているため、3番目のテンプレートパラメータを`std::less<void>`にするとHeterogeneous Overloadが有効化されます。

```cpp
std::flat_map<std::string, int, std::less<void>> map = {
  {"one", 1},
  {"three", 3},
  {"five", 5}
};

std::string_view key = "five";

auto& elem1 = map.at(key);    // ok
auto& elem2 = map.at("one");  // ok、一時オブジェクトなし
```

あるいは`std::ranges::less`なども`is_transparent`を持っています。

なお、Heterogeneous Overload自体はここまで紹介してきたほとんどの操作（追加・削除・検索）においてオーバーロードとして用意されているため、比較型を差し替えればすぐに利用可能になります。

### ノードハンドル

`flat`なコンテナはいずれもノードベースコンテナではないため、ノードハンドルに非対応です。これは連想コンテナと明確に異なる点です。

### 内部配列アクセス

他の連想コンテナだとノードハンドル取り出しに使用されている`.extract()`関数は内部配列を取り出すためのもので、ノードハンドルインターフェースではなく、右辺値参照修飾されています。

```cpp
int main() {
  std::flat_map<int, std::string_view> map{ ... };

  auto conainers = map.extract(); // ng、左辺値から呼び出せない
  auto conainers = std::move(map).extract();  // ok

  conainers.keys;   // キーの配列
  conainers.values; // 値の配列
}
```

`.extract()`の戻り値型は2つの配列を保持する集成体型であり、`keys`/`values`という名前で取り出した配列にアクセスすることができます。集成体型であるため、構造化束縛を利用することもできます。

`.extract()`で取り出した内部配列を別の`flat`なコンテナに渡すには、コンストラクタか`.replace()`を使用します。

```cpp
std::flat_map<int, std::string_view> src{ ... };

auto conainers = std::move(src).extract();

// 内部配列を指定するコンストラクタ
std::flat_map<int, std::string_view> dst {
 std::sorted_unique,
 conainers.keys,
 conainers.values
};

dst.clear();

// 内部配列の置き換え
dst.replace(
  std::move(conainers.keys),
  std::move(conainers.values)
);
```

コンストラクタ渡しの場合はコピーして渡すことができますが、`.replace()`はムーブのみ（右辺値参照引数のみ）を受け付けます。

`.replace()`の動作はその名の通り、現在の内部配列を渡されたもので置き換えます。このため、内部配列を渡すコンストラクタ同様に両方の配列が同じ長さである必要があり、さらにソート済みである必要もあります（これらは事前条件として指定されます）。

左辺値のコンテナから単に内部配列を参照したい場合は、`.keys()`/`.values()`が利用できます。

```cpp
std::flat_map<int, std::string_view> src{ ... };

auto& keys = src.keys();
auto& values = src.values();
```

これらの配列は`flat`コンテナの不変条件を満たした状態で取得できますが、これらの関数の戻り値は`const`参照であるため、内部配列を外から変更することはできません。また、イテレータと異なりこれらの参照は取得元のコンテナの生存期間内で無効化されることは基本的にはありません。

キーと値どちらかだけをソート済みの状態でイテレーションしたい場合に`map | views::keys`（`elements_view`）などを使用するよりも、直接コンテナをイテレーションした方が少しだけ効率的になるでしょう。また、デフォルトの`flat`なコンテナは`contiguous_range`にはなりませんが、こちらの配列は`std::vector`が使用されている場合は`contiguous_range`であるため、ポインタを使用可能になります。

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
