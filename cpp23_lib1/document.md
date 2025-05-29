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
