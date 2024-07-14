---
title: C++20 落穂拾い
author: onihusube
date: 2024/08/12
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

本書は既発行の『C++20 コア言語機能』『C++20 ranges』『C++20 ライブラリ機能1, 2』などで時間の制約などの都合で扱っていなかったトピックについて解説するものです。

## 注意など

本書はC++20をベースとして記述されています。そのため、C++17までに導入されている機能については特に導入バージョンについて触れず、知っているものとして詳しい解説なども行いません。また、C++20の他の機能に関しても、すでに以前の書籍で説明済みのものは知っているものとして使用します。そちらの説明は以前発行の書籍（『C++20 コア言語機能』『C++20 ranges』『C++20 ライブラリ機能1, 2』）もしくはcpprefjpなどの解説サイトを参照してください。

## サンプルコードのお約束

- そこで主題となっているライブラリ機能のためのヘッダのみを明示的にインクルードし、他のヘッダのインクルードは省略します
- 主題と関係ないところを`...`で省略していることがあります
- 行の末尾コメントで`ok/ng`と表示することで、それがコンパイル可能かどうかを示しています。`// ok`はコンパイルが通り未定義動作がないこと、`// ng`はコンパイルエラーとなる事をそれぞれ表しています
- 本文中でメンバ関数を表記する際、`.menber_func()`のように先頭に`.`を付加して区別しています

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

# コンセプトの半順序、もう少し

コア言語機能本のコンセプトの章では、コンセプトによるオーバーロード解決時の順序付けについて`&&`のもののみを説明して`||`の場合を省略していました。ここでは`||`が入ってくる場合を少しだけ覗いてみます。

## `||`だけからなる制約式における順序付け

`&&`だけからなる場合の制約の順序は、`&&`で繋がれている制約を集めて来た時、片方の関数の制約の集合がもう片方の関数の制約の集合に完全に包含されている場合に包含している方がより制約されているとして、より優先的に選択されていました。

```cpp
template<std::copyable T>
void f(T);  // (1) copyableのみ

template<std::copyable T>
  requires std::convertible_to<T, int>
void f(T);  // (2) copyable + convertible_to

int main() {
  int n = 10;

  f(n);  // (2)が呼ばれる
}
```

`||`の場合も同様にコンセプトの包含関係を考えることで順序をとらえることができますが、その考え方は`&&`とは逆に制約の包含関係が成り立っている場合は包含されている方がより優先順位が高くなります。

```cpp
template<typename T>
  requires std::integral<T>
void f(T);   // (1) integralのみ
template<typename T>
  requires std::integral<T> || std::floating_point<T>
void f(T);   // (2) integral + floating_point

int main() {
  f(10);  // (1)が呼ばれる
  f(1.0); // (2)が呼ばれる
}
```

2つの関数オーバーロードAとBの間の制約の包含関係は、`||`でつながれている原子制約について、一方の関数（A）の全ての原子制約が他方の関数（B）の制約に全て現れていて、かつ他方（B）はそれに加えてさらに制約されている（`||`によって）時、A ⊂ Bの制約の包含関係が成立します（ここまでは`&&`と同じ）。しかし、`||`の場合は包含されている方がより制約されているとしてA > B（AがBより順位が高い）の順序付けがなされます。そして、どちらのオーバーロードもコンセプト以外の部分での順序付けが同等となる場合、より制約されている方のオーバーロードが優先的に選択されます。

上記の例では`f(10)`の呼び出しに対して、(1)も(2)も`std::integral<T>`コンセプトを満たすためどちらも制約を満たすことになりオーバーロード候補として残ります。その際、(2)は`integral<T>`に加えて`floating_point<T>`によっても制約がなされており、同じ`T`に対して(1)の制約は(2)の制約によって完全に包含されており、その包含関係とは逆に(1)のほうが(2)よりも優先順位が高くなります。

ただし、上記`f(1.0)`のように片方のオーバーロードの制約を満たさないことでオーバーロード解決が発生しない場合は制約の包含関係は関係なく残った1つの候補が選択されます。これは`||`に対する直感通りの結果でしょう。

この`||`の場合の制約の包含関係は、`||`で繋がれた中に満たされていない（`false`となっている）コンセプトがあるかどうかは関係がありません。`||`の場合は1つでも満たしていれば全体の制約が満たされるため、それによってオーバーロード解決が発生する場合のコンセプトの包含関係は制約の全体によって考慮されます。例えば次のように`||`の両辺の制約を満たしている場合でも、より制約の少ない方が優先的に選択されます。

```cpp
template<typename T>
  requires std::integral<T>
void f(T);   // (1)
template<typename T>
  requires std::integral<T> || std::same_as<T, int>
void f(T);   // (2)

int main() {
  int n = 10;
  f(n);  // (1)が呼ばれる
}
```

別の言い方をすると、`||`だけが使用されて制約されている2つの関数オーバーロードそれぞれの原子制約を要素とする2つの集合があった時、その間に成立する包含関係（片方は他方の真部分集合）を逆転させたものがコンセプトの包含関係となります。そして、そのような包含関係が成り立たない時は順序付け不可能となり、2つの関数オーバーロードの呼び出しは曖昧となります。

```cpp
template<typename T>
concept A = ...;

template<typename T>
concept B = ...;

template<typename T>
concept C = ...;

template<typename T>
concept D = ...;

template<typename T>
  requires A<T>
void f(T); // (1)

template<typename T>
  requires A<T> || B<T>
void f(T); // (2)

template<typename T>
  requires A<T> || B<T> || C<T>
void f(T); // (3)

template<typename T>
  requires A<T> || B<T> || D<T>
void f(T); // (4)
```

この時、各`f()`の制約の包含関係は(1) ⊂ (2) ⊂ (3)となり、(1) > (2) > (3)の順で優先順位が付きます。また、(1) ⊂ (2) ⊂ (4)の包含関係も成立しているため、(1) > (2) > (4)の優先順位も付きます。 一方、(3)と(4)の制約の間にはお互いに包含関係が成立しない（`C`は(4)の制約に現れず、`D`は(3)の制約に現れない）ため順序が付きません。したがって、(3)と(4)の間ではオーバーロード解決が曖昧となります。

この例のような`f<T>()`の呼び出しにおいて、`T`が`A, B, C`の制約をすべて満たす場合は(1)のオーバーロードが呼ばれます（このとき`D`を満たしていても呼び出しは曖昧になりません）。同様に、`T`が`A, B, D`の制約をすべて満たす場合も(1)のオーバーロードが呼ばれます（`C`を満たしていても）。`T`が`B`だけを満たす場合は(2)が呼ばれ、`T`が`C`だけを満たす場合は(3)が呼ばれ、`T`が`D`だけを満たす場合は(4)が呼ばれます。

一方、`T`が`C`と`D`だけを満たす場合は(3)と(4)の間に順序がつかないため呼び出しは曖昧となりエラーになります。

## `||`と`&&`が混在する場合の順序付け

コンセプトが`&&`だけ、あるいは`||`だけによって構成されている場合は比較的簡単にその順序を理解することができます。しかし、次のような場合はどうでしょう？

```cpp
template<typename T>
concept A = ...;

template<typename T>
concept B = ...;

template<typename T>
concept C = ...;

template<typename T>
concept D = ...;

template<typename T>
  requires ((A<T> && B<T>) || C<T>) && D<T>
void f(); // (1)

template<typename T>
  requires A<T> || D<T>
void f(); // (2)
```

全てのコンセプトを満たすような`T`の入力に対して、どちらのオーバーロードが呼ばれるか分かるでしょうか？これまでのような制約の包含関係の延長でこれを理解するのは無理そうです。これはコンパイラにとっても同様なため、標準ではこのような複雑な制約の間でも順序判定が可能な方法を指定しています。

2つのオーバーロード間の制約を比較する時、1つ目のオーバーロードの制約全体を`P`、2つ目のオーバーロードの制約全体を`Q`として、`P`が`Q`より制約されているかどうかを判定する手順は次のようになります

1. `P`を選言標準形、`Q`を連言標準形に変換する
    - 選言標準形は、`()`で囲まれた`&&`の制約を`||`で接続した形の制約
    - 連言標準形は、`()`で囲まれた`||`の制約を`&&`で接続した形の制約
2. `P`の選言標準形の各項（部分式）`Pi`と`Q`の連言標準形の各項`Qj`を比較する
    - `Pi`は必ず`&&`だけからなる制約となっている
    - `Qj`は必ず`||`だけからなる制約となっている
3. それぞれの`Pi`と`Qj`の比較においては、さらにそれの各項`Pia`と`Qjb`を比較し、`Pia`と`Qjb`が同じコンセプトによる制約になっている組が1つでもあれば、`Pi`は`Qj`よりも制約されている（`Pi > Qj`）となる
4. `P`の選言標準形の全ての項`Pi`が`Q`の連言標準形の全ての項`Qj`よりも制約されている場合に、`P`は`Q`よりも制約されている（`P > Q`）となる

よっぽど数理論理学に精通しているわけでもなければ、これを見てなるほど！とは思わないと思います。まず1に出てくる選言標準形と連言標準形から聞きなれないでしょう。

選言標準形は制約を分配法則や結合法則によって変換して、`&&`による制約を`||`でつないだ形に変換したものです。同様に、連言標準形は`||`による制約を`&&`でつないだ形に変換したものです。

分配法則や結合法則は論理和と論理積（$\lor, \land$）に成り立つものを$\lor$を`||`に$\land$を`&&`に対応させて使用することができます。

```
// 分配法則
A || (B && C) <=> (A || B) && (A || C)
A && (B || C) <=> (A && B) || (A && C)

// 結合法則
A || (B || C) <=> (A || B) || C
A && (B && C) <=> (A && B) && C
```

これを用いてあらゆる制約は選言標準形と連言標準形の両方に変換することができます。

いきなり論理和や論理積が出てきたことから分かるかもしれませんが、選言標準形や連言標準形という言葉もそちらの数理論理学の文脈から来ています。任意の制約がこの2つの標準形に変換可能であることも論理式において同様のことが成り立つことによって保証されています。

論理式における選言・連言標準形への変換では、次の3ステップによって機械的に変形できることが知られています

1. $\to$と$\iff$を同値変形によって論理和と論理積のみからなる式に変形する
2. ド・モルガンの法則、二重否定除去規則を適用する
3. 分配法則によって、選言標準形の場合`&&`を、連言標準形の場合`||`を`()`の中へ移動する

各ステップにおいてはそれぞれのステップでの操作が適用できなくなるまでそれを繰り返します。

とはいえコンセプトの制約式においては1のステップは関係ありません。2のステップに関しては、後述するように否定（`!`）の存在については注意が必要となり、否定は全てコンセプトの中に入れるようにすると不要になります。結局、3のステップが重要となります。

例として、`(a1 && a2) || (a3 && (a4 || (a5 && a6)))`という制約式を変形すると

選言標準形の場合

```
  (a1 && a2) || (a3 && (a4 || (a5 && a6)))
= (a1 && a2) || ((a3 && a4) || (a3 && (a5 && a6)))
= (a1 && a2) || ((a3 && a4) || (a3 && a5 && a6))
= (a1 && a2) || (a3 && a4) || (a3 && a5 && a6)
```

連言標準形の場合

```
  (a1 && a2) || (a3 && (a4 || (a5 && a6)))
= (a1 && a2) || (a3 && ((a4 || a5) && (a4 || a6)))
= (a1 && a2) || (a3 && (a4 || a5) && (a4 || a6))
= (a1 || a3) && (a1 || a4 || a5) && (a1 || a4 || a6) &&
  (a2 || a3) && (a2 || a4 || a5) && (a2 || a4 || a6)
```

となります。ここでの変形には上記の分配法則と結合法則だけを使用しています。

少し脱線しましたがともかく、同様にして任意の制約式は選言標準形にも連言標準形にも変形することができます。

制約の比較手順に戻って、2番目の手順で使用される選言・連言標準形の各項というのは、選言標準形の場合は`||`で繋がれた各項のことで、連言標準形の場合は`&&`で繋がれた各項の事です。先程の例で計算した選言標準形`(a1 && a2) || (a3 && a4) || (a3 && a5 && a6)`だと、`(a1 && a2), (a3 && a4), (a3 && a5 && a6)`の3つの部分式がそれにあたり、連言標準形`(a1 || a3) && (a1 || a4 || a5) && (a1 || a4 || a6) && (a2 || a3) && (a2 || a4 || a5) && (a2 || a4 || a6)`だと`(a1 || a3), (a1 || a4 || a5), (a1 || a4 || a6), (a2 || a3), (a2 || a4 || a5), (a2 || a4 || a6)`の6つの部分式がそれにあたります。

この部分式をよく見ると、選言標準形の場合は各項の制約式は必ず`&&`だけからなり、連言標準形の場合は各項の制約式は必ず`||`だけからなっています。わざわざ標準形に変換するのはこのことが重要だからです。

これらの各項（部分式）は再帰的に部分式を持ち、それが手順の3番目で使用されている`Pia, Qjb`に当たります。

また、手順の3番目で`Pia`と`Qjb`が同じコンセプトによる制約になっているとは、そのまま引数型まで含めて同じコンセプトによる制約になっている場合をいいます。ただしここで注意なのは、コンセプトによらない通常の`bool`式による制約（例えば型特性`std::is_same_v<...>`や比較演算`sizeof(T) == 4`など）では通常これが成り立たず、一見同じ制約式でも異なる制約とみなされます。

このことはC++20 コア言語機能においても触れていましたが、特に問題なのはコンセプトの否定（例えばコンセプト`C<T>`に対する`!C<T>`）もこれに該当してしまう事です。`!`によるコンセプトの制約式の否定はそれが1つの個別の制約式かつ非コンセプトの制約式となるため、`Pia`と`Qjb`に意味的に同一となるそのような否定コンセプトのペアがあってもそれは同じ制約にはなりません。標準形への変換のところで言及していた注意点とはこのことで、このためにコンセプトの制約式ではド・モルガンの法則も二重否定の除去も（意味的には）成り立ちません。幸いこれは、コンセプトの否定を1つのコンセプトとして切り出してしまうと解決できます。

```cpp
// コンセプトAとBがあるとして

template<typename T>
  requires !A<T>
void f(); // (1)

template<typename T>
  requires !A<T> && B<T>
void f(); // (2)

template<typename T>
concept notA = !A<T>;

template<typename T>
  requires notA<T>
void g(); // (3)

template<typename T>
  requires notA<T> && B<T>
void g(); // (4)


int main() {
  // TはAを満たさないがBを満たすとする
  f<T>(); // ng、呼び出しが曖昧

  g<T>(); // ok、(4)が呼ばれる
}
```

ただし、意味的には成り立たなくても変形は適用可能なので、C++コンパイラは制約式の標準形への変換に際して容赦なくそれを行うでしょう。コンセプトの順序が重要になる場合は、否定（`!`）の扱いには注意する必要があります。

さて、一通り説明を終えたところで最初の例に戻りましょう。

```cpp
template<typename T>
  requires ((A<T> && B<T>) || C<T>) && D<T>
void f(); // (1)

template<typename T>
  requires A<T> || D<T>
void f(); // (2)
```

このオーバーロードの優先順位の判定は、`((A<T> && B<T>) || C<T>) && D<T>`と`A<T> || D<T>`の間で順序が付くかどうか（どちらがより制約されているのか）を先程の手順によって判定すればわかります。

まずは`((A<T> && B<T>) || C<T>) && D<T>`を選言標準形に変換します

```
  ((A<T> && B<T>) || C<T>) && D<T>
= (((A<T> && B<T>) && D<T>) || (C<T> && D<T>))
= (A<T> && B<T> && D<T>) || (C<T> && D<T>)
```

`A<T> || D<T>`は既に連言標準形になっているのでこのまま使用します。

次に、それぞれの部分式を取り出します。

```
Pi = (A<T> && B<T> && D<T>), (C<T> && D<T>)
Qj = (A<T> || D<T>)
```

そしてこの`Pi`と`Qj`の各項同士をすべて比較して、そのすべてに対して一致する部分式が存在するかを調べます。今回比較すべきペアは2つです

```
(A<T> && B<T> && D<T>) : (A<T> || D<T>)
(C<T> && D<T>) : (A<T> || D<T>)
```

まず1つ目のペアの左辺は、`A<T>, B<T>, D<T>`の3つの部分式からなり、右辺は`A<T>, D<T>`の2つの部分式からなります。どちらにも`A<T>`と`D<T>`が含まれているため、左辺の方が右辺よりもより制約されています。

2つ目のペアはそれぞれ、`C<T>, D<T>`の2つと`A<T>, D<T>`の2つの部分式からなります。どちらにも`D<T>`が含まれているため、こちらも左辺の方が右辺よりも制約されています。

```
(A<T> && B<T> && D<T>) > (A<T> || D<T>)
 ^^^^            ~~~~     ^^^^    ~~~~
(C<T> && D<T>) > (A<T> || D<T>)
         ^^^^             ^^^^
```

比較すべきペア全てにおいて`Pi`の方が`Qj`よりも制約されているため、`((A<T> && B<T>) || C<T>) && D<T>`の方が`A<T> || D<T>`よりも制約されていることが分かりました。

コンセプトの順序付けにおいては双方向でこの制約関係を調べる必要があり、なおかつどちらか一方のみがより制約されている場合にのみ順序を付けることができます。そのため、逆方向も同様に調べます。

まずは標準形への変換です。`A<T> || D<T>`はすでに選言標準形なのでこのままでよく、`((A<T> && B<T>) || C<T>) && D<T>`を連言標準形に変換します。

```
  ((A<T> && B<T>) || C<T>) && D<T>
= ((A<T> || C<T>) && (C<T> || B<T>)) && D<T>
= (A<T> || C<T>) && (C<T> || B<T>) && D<T>
```

次に、それぞれの部分式を取り出します。

```
Pi = (A<T>), (D<T>)
Qj = (A<T> || C<T>), (C<T> || B<T>), (D<T>)
```

そしてこの`Pi`と`Qj`の各項同士をすべて比較して、そのすべてに対して一致する部分式が存在するかを調べます。今回比較すべきペアは6つです

```
(A<T>) : (A<T> || C<T>)
(A<T>) : (C<T> || B<T>)
(A<T>) : (D<T>)
(D<T>) : (A<T> || C<T>)
(D<T>) : (C<T> || B<T>)
(D<T>) : (D<T>)
```

あとはそれぞれのペア事に、さらに部分式を取り出して同じものがあるかどうかを調べます。個別にみるのは省略して結果だけを見ると

```
(A<T>) > (A<T> || C<T>)
 ^^^^     ^^^^
(A<T>) ? (C<T> || B<T>)

(A<T>) ? (D<T>)

(D<T>) ? (A<T> || C<T>)

(D<T>) ? (C<T> || B<T>)

(D<T>) > (D<T>)
 ^^^^     ^^^^
```

今度は、`Pi`と`Qj`のペアの中に順序がつかないものが1つ以上含まれているため、`P`は`Q`よりも制約されているとは言えません。

従って、`A<T> || D<T>`は`((A<T> && B<T>) || C<T>) && D<T>`よりも制約されていないことが分かりました。

- `((A<T> && B<T>) || C<T>) && D<T>` > `A<T> || D<T>`
- `A<T> || D<T>` ? `((A<T> && B<T>) || C<T>) && D<T>`

という双方向の順序が得られてかつその順序が一方向に決まっているため、この2つの制約の間には順序付けを行うことができます。

```cpp
template<typename T>
  requires ((A<T> && B<T>) || C<T>) && D<T>
void f(); // (1)

template<typename T>
  requires A<T> || D<T>
void f(); // (2)

int main() {
  // TはA, B, C, D全てのコンセプトを満たすとして
  f<T>(); // (1)が呼ばれる
}
```

この場合、`T`が`D`を満たしていない場合は(2)が呼ばれることになります。また、`A`も`B`も満たさないが`C`と`D`は満たす場合は(1)が呼ばれ、`D`だけを満たす場合には(2)が呼ばれます。

## コンセプト定義の再帰的な展開

前項の説明はコンセプトが与えられていた場合にその表層的な部分だけに注目して説明したものだったので、実は正しくない部分があります。それは、コンセプトの順序を考えるときに、コンセプトの定義の制約式も

これは、最初に制約式を標準形に変換する際に、現れている各種コンセプトを再帰的に展開してから標準形に変換することで行われます。より正しい手順は次のようになります

1. 1つ目のオーバーロードの制約式を原子制約式だけからなるように展開した制約式を`P`、2つ目のオーバーロードの制約式を原子制約式だけからなるように展開した制約式を`Q`とする
    - 制約式に含まれているコンセプトを、その定義によって置換していく
    - ただし、否定（`!`）されているコンセプトは展開せず、それで一つの原子制約式となる
    - `P, Q`にはコンセプトによる制約は直接的には含まれなくなる
2. `P`を選言標準形、`Q`を連言標準形に変換する
    - 選言標準形は、`()`で囲まれた`&&`の制約を`||`で接続した形の制約
    - 連言標準形は、`()`で囲まれた`||`の制約を`&&`で接続した形の制約
3. `P`の選言標準形の各項（部分式）`Pi`と`Q`の連言標準形の各項`Qj`を比較する
    - `Pi`は必ず`&&`だけからなる制約となっている
    - `Qj`は必ず`||`だけからなる制約となっている
4. それぞれの`Pi`と`Qj`の比較においては、さらにそれの各項`Pia`と`Qjb`を比較し、`Pia`と`Qjb`が同じ原子制約式による制約になっている組が1つでもあれば、`Pi`は`Qj`よりも制約されている（`Pi > Qj`）となる
    - コンセプトの展開によって導入されていない原子制約式は、たとえ意味的にも字句的にも同一のものでも異なるものとして扱われる
5. `P`の選言標準形の全ての項`Pi`が`Q`の連言標準形の全ての項`Qj`よりも制約されている場合に、`P`は`Q`よりも制約されている（`P > Q`）となる

これによって、直接使用されているコンセプトだけを見た場合に一見その順序付けができない場合でも、定義のレベルで順序付けが可能であるならば先程の手順によって順序付けがなされます。

```cpp
template<typename T>
  requires std::integral<T>
void f(); // (1)

template<typename T>
  requires std::signed_integral<T>
void f(); // (2)

int main() {
  f<int>(); // (2)が呼ばれる
}
```

`std::signed_integral`コンセプトは、`std::integral`かつ`std::is_signed_v`によって制約されており、`std::signed_integral`は`std::integral`よりも制約されるように定義されています。

厄介な点の1つ目は、`!C<T>`のように否定されて現れているコンセプトはそれ以上展開されず、これで1つの原子制約式として扱われる点です。これが異なるコンセプト定義から来ている場合、手順4のマッチング時に同じ制約だとはみなされません。

厄介な点の2つ目は、コンセプトを通さず直接現れている非コンセプトの原子制約式とは異なり、コンセプトの定義を展開した結果として現れる非コンセプトの原子制約式は、手順4のマッチング時に、同じコンセプトから来ているものは同じ原子制約式であるとみなされます。これによって、`!`を直接使用せずにコンセプトに分離すれば順序付けが可能となるわけです。

```cpp
template<typename T>
concept A = false;

template<typename T>
concept B = !A<T>;

template<typename T>
concept C = true;

template<typename T>
concept D = !A<T> && C<T>;

template<typename T>
  requires B<T>
void f(); // (1)

template<typename T>
  requires D<T>
void f(); // (2)

template<typename T>
concept E = B<T> && C<T>;

template<typename T>
  requires B<T>
void g(); // (3)

template<typename T>
  requires E<T>
void g(); // (4)

int main() {
  f<int>(); // ng、呼び出しが曖昧
  g<int>(); // ok、(4)が呼ばれる
}
```

この例では、`f<T>`の呼び出しにおいてコンセプト`B<T>`と`D<T>`の比較が行われ、展開し標準形に変換すると`!false_(B)`と`!false_(D) && true_(C)`となります。`B<T>` > `D<T>`の判定をする場合、`Pi = !false_(B)`と`Qi = !false_(D), true_(C)`となりそれぞれの部分式（この場合1つだけ）同士の比較をこなうわけですが、`true_(C)`に対応するものが無いので順序は定まりません。逆向き`D<T>` > `B<T>`の判定をする場合、`Pi = !false_(D) && true_(C)`と`Qi = !false_(B)`となりマッチングを行うわけですが、このとき`!false`という原子制約式だけを見るとマッチングが取れますがその由来までみると`!false_(D)`と`!false_(B)`で出所が異なるため同じ制約とはみなされず、結局`D<T>` > `B<T>`方向も順序が付かずコンパイルエラーとなります。

`g<T>`の呼び出しの場合、`B<T>`と`E<T>`の比較を行うわけですが、`B<T> > E<T>`は先程と同様に順序が付きません。逆向き`E<T>` > `B<T>`の判定をする場合、`Pi = !false_(B) && true_(C)`と`Qi = !false_(B)`となりマッチングを行うわけですが、今度は`E`の`Pi`の部分式`!false_(B)`が`B`の`Qi`の部分式`!false_(B)`とマッチするため、`E<T>` > `B<T>`の順序が付きます。

このように、手作業で判定を行おうとする場合は原子制約式の由来による区別に気を付けなければなりません。なお、コンセプトに包まれていない原子制約式は由来を持たず、その場合は字句的意味的にも同じでも常に異なる制約とみなされます（何度も言っていることですが・・・）。

TODO: 展開したうえで順序付けを求める例をもう一つ追加する。例えば前項で計算してた例など。あるいは、適当な論理式に対してそれと順序が付くものを求める例。

## 順序付けが必要ない場合の制約のテクニック

コンセプトによってテンプレートパラメータの制約のために結果的に`&& ||`が混在しているものの、別の制約との間で順序をつけることを意図しない場合はよくあると思われます。この場合に`&& ||`の制約を`requires`式内部に移動することによって、順序の表現を捨てる代わりにコンパイル負荷の軽減を図ることができます。

```cpp
template<typename T>
concept A = /*1つの原子制約式*/;

template<typename T>
concept B = /*1つの原子制約式*/;

template<typename T>
concept C = /*1つの原子制約式*/;

template<typename T>
  requires A<T> && (B<T> || C<T>)
void f();

template<typename T>
  requires A<T> && 
           requires { requires (B<T> || C<T>); }
void g();
```

この`f()`と`g()`制約は同じ制約を表現しています。ただし、1つ目の制約は`(A<T>), (A<T> && B<T>), (A<T> && C<T>), (B<T> || C<T>)`の制約を持つオーバーロードに対して優先され（順序が付く）ますが、`g()`は`A<T>`の制約以外に対した順序が付きません。これは、`requires { requires (B<T> || C<T>); }`で1つの原子制約式となるためです。

ただし、これによってコンセプトを構成する原子制約式が単純化されるため、2つめのほうが相対的にコンパイルへの負荷が軽くなります。これは、制約の順序付けの際に考慮すべき原子制約式を減少させ、標準形への変換や比較の負荷を軽減できるためです。

ただし、これは`requires`式内部の制約式は順序に影響しないということでもあり、このように書き換えることで順序付けを行えなくなります。順序付けを考慮しない場合や、順序付けに参加する制約式を減らしたい場合などに検討してみるといいかもしれません。

# `<=>`演算子の更なる自動生成

`<=>`演算子をデフォルト定義することによって、残りの比較演算子が自動で実装されるようになり、C++20からは比較演算子の実装が著しく簡単になります。しかし、そのためには基底クラス及びメンバ型のすべてが`<=> ==`を定義している必要があります。`==`はともかく、`<=>`はC++20で追加された新しい演算子であり、C++17以前で定義されたクラス型で備えているものはありません。そのため、C++17以前のクラスに対して`<=>`演算子を追加しない場合、`<=>`演算子のデフォルト実装は利用できないことになります。

```cpp
#include <compare>

// C++17以前から使用されてきた型、従来の比較演算子は実装されている
struct old_type {
  int n = 10;

  // 共に実装は省略
  bool operator< (const old_type&) const;
  bool operator==(const old_type&) const;
};


// C++20環境で定義された新しい型
struct new_type {
  int m = 10;
  old_type l = {20}; // <=>を持たない
  int n = 30;

  // old_typeは<=>を持たないため、実装不可、暗黙delete
  auto operator<=>(const new_type&) const = default;

  // ==があるため実装可能（明示的な宣言は実は不要）
  bool operator== (const new_type&) const = default;
};

int main() {
  new_type n1{}, n2 = {20, {30}, 40};

  auto comp = n1 <=> n2;  // ng
  bool eq   = n1 == n2;   // ok
}
```

このような例は珍しいものではなく、既存のコードベースをC++20環境へ移行した後に良く遭遇する光景となるでしょう。また、この`old_type`のような型が変更できない場所（外部ライブラリや他人の管轄コードベースなど）に存在していると`<=>`を追加することもできません。

このような場合でも`<=>`演算子をなるべく書かなくて済むようにするために、`< ==`の2つの演算子を使用して`<=>`を自動実装する仕組みが用意されています。

```cpp
#include <compare>

struct new_type {
  int m = 10;
  old_type l = {20};  // <=>を備えていない
  int n = 30;

  // 指定した戻り値型とold_typeの持つ比較演算子を用いて実装してもらう
  std::strong_ordering operator<=>(const new_type&) const = default;
};

int main() {
  new_type n1{}, n2 = {20, {30}, 40};

  auto comp = n1 <=> n2;  // ok
  bool eq   = n1 == n2;   // ok
}
```

このように、戻り値の比較カテゴリ型を明示的に指定したうえで`default`指定することで、`<=>`演算子が利用可能ではない型がある場合は`< ==`を用いて比較を実装するようになります。

この場合の`default`実装を明示的に書くと次のようになります

```cpp
#include <compare>

struct new_type {
  int m = 10;
  old_type l = {20};
  int n = 30;

  //std::strong_ordering operator<=>(const new_type&) const = default;
  std::strong_ordering operator<=>(const new_type&) const = default {
    // <=>が使える場合はそれを使用
    if (auto comp = static_cast<std::strong_ordering>(m <=> that.m); comp != 0) return comp;

    // <=>が使えない場合は< ==から比較を実装
    std::strong_ordering comp = (l == that.l) ? std::strong_ordering::equal :　
                                (l <  that.l) ? std::strong_ordering::less
                                              : std::strong_ordering::greater;
    if (comp != 0) return comp;

    return static_cast<std::strong_ordering>(n <=> that.n);
  }
};
```

比較しようとするメンバ（基底クラスサブオブジェクト）を`a, b`、指定された戻り値型を`R`としたときに、このように戻り値型を明示的に指定する場合の比較の実装は`R`によって次のようになります

```cpp
// a <=> bが使用可能な場合
static_cast<R>(a <=> b);

// Rがstd::strong_orderingであり
// a < b, a == bが共に使用可能な場合
a == b ? std::strong_ordering::equal : 
a < b  ? std::strong_ordering::less
       : std::strong_ordering::greater;

// Rがstd::weak_orderingであり
// a < b, a == bが共に使用可能な場合
a == b ? std::weak_ordering::equivalent :
a < b  ? std::weak_ordering::less :
       : std::weak_ordering::greater;

// Rがstd::partial_orderingであり
// a < b, a == bが共に使用可能な場合
a == b ? std::partial_ordering::equivalent : 
a < b  ? std::partial_ordering::less : 
b < a  ? std::partial_ordering::greater
       : std::partial_ordering::unordered;
```

実装内では、この結果（必ず比較カテゴリ型のいずれかになる）を受け取って、それが`0`と等しくなければそれを返して終了、`0`と等しければ次へ、のような実装を行います（この部分は戻り値型を`auto`にする場合と同じ）。

このように戻り値型の指定を明示的に行った場合にのみ`< ==`を使用した実装を行っているのは、`partial_ordering`の`unordered`の判定および`weak_ordering`の`equivalent`を正しくハンドルするためです。戻り値型を指定しない場合、これらの値は全て`strong_ordering::equal`として扱われてしまうため、適切な比較にならない可能性があります。

## クラステンプレートにおける適応的な比較演算子自動生成

より汎用的というなら、戻り値型を常に指定して`<=>`を`default`実装する方がより広い状況で`<=>`を実装することができます。ただし、戻り値型を明示的に指定してしまうと、各メンバ・基底クラスの比較がその戻り値型に明示的に変換可能ではない場合にエラーになってしまいます。また、`auto`で本来導出される比較カテゴリ型とは異なる最適ではないカテゴリになってしまう可能性もあります。

```cpp
#include <compare>

template<typename T>
struct wrap {
  T v;

  // Tの<=>の結果がweak_orderingに変換できない場合にエラーになる
  std::weak_ordering operator<=>(const wrap&) const = default;
};

int main() {
  wrap<double> w1{0.0}, w2{1.0}; // ok。ここではエラーにならない

  bool cmp = w1 < w2; // ng、<=>はdeleteされている
}
```

`double`の`<=>`の比較結果は`std::partial_ordering`であり、`std::weak_ordering`は`std::partial_ordering`に変換できないため比較演算子を使用しようとするとエラーになります。

つまり、`<=>`が使用できる場合は戻り値`auto`の宣言を、使用できない場合は戻り値型を指定した宣言を使ってほしいのですが、そのようなことはできません。

```cpp
#include <compare>

template<typename T>
struct wrap {
  T v;

  // この2つの宣言を同居できない
  auto operator<=>(const wrap&) const = default;
  std::weak_ordering operator<=>(const wrap&) const = default;
};
```

この`wrap<T>`のように、メンバとして持つ型がテンプレートパラメータによって指定されているような場合、この問題は悩ましいものになるでしょう。

このような場合のために、`<compare>`に用意されている機能を使用して1つの宣言でこの要求を叶えるソリューションを考えることができます。

```cpp
#include <compare>

// Tが<=>を使用できない場合、Catにフォールバックする
template<typename T, typename Cat>
using fallback_comp3way_t = std::conditional_t<
        std::three_way_comparable<T>,
          std::compare_three_way_result<T>,
          std::type_identity<Cat>
      >::type;
```

`std::three_way_comparable<T>`は`T`が`<=>`によって比較可能であることを定義するコンセプトです、それによって`<=>`の利用可否を判定して、利用できる場合は`std::compare_three_way_result<T>`によってその結果型を取得して返し、できない場合はフォールバック先の比較カテゴリ型`Cat`を返します。

結局、`fallback_comp3way_t<T, Cat>`は、`T`が`<=>`を利用可能であればその結果型、利用できなければ`Cat`になります。

これを使って先程の`wrap`を書き換えると

```cpp
#include <compare>

template<typename T>
struct wrap {
  T t;

  // <=>を使用可能ならそれを、そうでないなら< ==を使ってdefault実装
  auto operator<=>(const wrap&) const
    -> fallback_comp3way_t<T, std::weak_ordering>
      = default;
};

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
    wrap<double> w1{0.0}, w2{1.0};

    // 全て利用可能
    bool b1 = w1 < w2;  // true
    bool b2 = w1 <= w2; // true
    bool b3 = w1 > w2;  // false
    bool b4 = w1 >= w2; // false
  }
  { 
    wrap<no_spaceship> w1{0}, w2{1};

    // 全て利用可能
    bool b1 = w1 < w2;  // true
    bool b2 = w1 <= w2; // true
    bool b3 = w1 > w2;  // false
    bool b4 = w1 >= w2; // false
  }
}
```

厳密には`auto`のままで定義しているわけではありませんが、`<=>`が利用できる場合の結果型は`auto`で宣言した場合と同じ結果になり、なおかつ利用できない場合でも指定した比較カテゴリ型にフォールバックすることでなるべく自動実装してもらうことができます。

対象となるテンプレートパラメータが複数ある場合は`std::common_comparison_category_t`を組み合わせて使用することでこのソリューションを引き続き使用することができます。

```cpp
#include <compare>

// フォールバック先のカテゴリ
using category = std::weak_ordering;

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

int main() {
  triple<int, double, no_spaceship> t1 = {10, 3.14, {20}}, 
                                    t2 = {10, 3.14, {30}};

  // 全て利用可能        
  bool b1 = t1 < t2;  // true
  bool b2 = t1 <= t2; // true
  bool b3 = t1 > t2;  // false
  bool b4 = t1 >= t2; // false
}
```

実はこの`fallback_comp3way_t`はライブラリ機能1でも紹介していました。しかし、そこでは戻り値型を指定した`<=>`の`default`宣言をほとんど説明していなかったため、その説明と合わせて改めて解説を行いました。

これをさらに発展させて`<`だけから`<=> ==`を定義する、みたいなことも考えることもできます（さらに複雑にはなりますが・・・）。

# `<chrono>`のスキップしたもの

C++20ライブラリ機能1における`<chrono>`の拡張の紹介時には、時間等の都合でスキップした項目がいくつかあります。ライブラリ機能2でも触れられることが無かったため、ここで拾っておきます。

この章では`#include <chrono>`と`using namespace std::chrono;`が各コードブロックの先頭に置かれていることとします。また、本文中でも`chrono`のエンティティについては`std::chrono::`を省略します。

## `hh_mm_ss`

ある`duration`の値（時間間隔の値）を1日内の時間表現の値として取得したいことがままあります。C++20ではそのために、`hh_mm_ss`クラステンプレートが用意されています。

`hh_mm_ss`は、任意精度の`dration`値を時分秒に分解するものです。

```cpp
int main() {
  hh_mm_ss hms1{3661s};
  hh_mm_ss hms2{124min};
  hh_mm_ss hms3{13h};
  hh_mm_ss hms4{123456789ms};

  std::cout << hms1 << '\n';
  std::cout << hms2 << '\n';
  std::cout << hms3 << '\n';
  std::cout << hms4 << '\n';
}
```
```
01:01:01
02:04:00
13:00:00
34:17:36.789
```

時分秒への分割を基本としていますが、秒未満の精度も保存されています。

分割後の値はメンバ関数によって取得できます。

```cpp
int main() {
  hh_mm_ss hms{123456789ms};

  hours   hh = hms.hours();
  minutes mm = hms.minutes();
  seconds ss = hms.seconds();
  milliseconds ms = hms.subseconds();
  
  std::cout << hh << '\n';
  std::cout << mm << '\n';
  std::cout << ss << '\n';
  std::cout << ms << '\n';
}
```
```
34h
17min
36s
789ms
```

これらの関数は全て対応する`duration`型の値（`.hours()`なら`std::chrono::hours`など）として分解結果を返します。ただし、秒未満を表す`.subseconds()`の戻り値型だけは入力の値の精度によって変化します。例えばこの例では入力値の精度が`milliseconds`なのでそれと同じ精度になっていますが、`microseconds`やその他の精度ならそれに準じた型の値を返します。

また、入力値が負の値だった場合でも負の値として分解し、その情報は`.is_negative()`メンバ関数によって取得できます。

```cpp
int main() {
  hh_mm_ss hms{-3661s};

  std::cout << hms << '\n';
  std::cout << std::format("{:s}\n", hms.is_negative());
  std::cout << hms.hours() << '\n';
}
```
```
-01:01:01
true
1h
```

この場合でも、`.hours()`等によって取得できる分解値はどれも負の値に名はなりません。

`hh_mm_ss`の値を元のように1つの`duration`値に戻すこともできます。そのためには、明示的な変換を使用します。

```cpp
int main() {
  hh_mm_ss hms{-123456789ms};

  // 明示的変換によって元に戻す
  milliseconds dr1{hms};        // 変換演算子(explicit)
  auto dr2 = hms.to_duration(); // 変換関数
  
  std::cout << dr1 << '\n';
  std::cout << dr2 << '\n';
}
```
```
-123456789ms
-123456789ms
```

ここまでの例で使用しているように標準出力ストリームへの`<<`を備えている他、`std::format()`対応（`std::formatter`特殊化）も用意されています。使用可能なフォーマットオプションは`<chrono>`のためのオプションのうち時間に関わるものが利用できます。 

```cpp
int main() {
  hh_mm_ss hms{3661s};

  std::cout << std::format("{}\n", hms);
  std::cout << std::format("{:%H時%M分%S秒}\n", hms);
  std::cout << std::format("{:%p %I:%M:%S}\n", hms);
}
```
```
01:01:01
01時01分01秒
AM 01:01:01
```

## 時間単位の`duration`値に対する関数

### 午前・午後の判定

時間単位の`duration`値が午前と午後どちらの時間を表しているのかを判定する関数が用意されます。

```{style=cppstddecl}
namespace std::chrono {
  // 午前かどうかの判定
  constexpr bool is_am(const hours& h) noexcept;

  // 午後かどうかの判定
  constexpr bool is_pm(const hours& h) noexcept;
}
```

これは、名前と宣言から想像できるような動作をします。

```cpp
void print(auto d) {
  std::cout << std::format("{:%Hh} : {:<5} / {}\n", d, is_am(d), is_pm(d));
}

int main() {
  print(0h);
  print(11h);
  print(12h);
  print(23h);
  print(24h);
  print(27h);
}
```
```
00h : true  / false
11h : true  / false
12h : false / true
23h : false / true
24h : false / false
27h : false / false
```

`is_am(d)`は`duration`値`d`について`0h <= d && d <= 11h`の結果を返し、`is_pm(d)`は`d`について`12h <= d && d <= 23h`の結果を返します。そのため、30時間制のような時間の表現はどちらも`false`を返します。

なお、この2つの関数の引数型は`hours`であるため、それに暗黙変換できない精度（分単位以下）をもつ`duration`型の値を渡すとエラーになります。とはいえ、`hours`に暗黙変換できる精度（`days`など）の値を渡しても常に`false`になります。この関数は、`hours`の値のうち`0 ~ 23`の範囲内でしか正しく動作しません。

### 12時間時計/24時間時計における時間値の変換

時間単位の`duration`値の値を、12時間時計における値と24時間時計における値に相互に変換する関数が用意されます。

```{style=cppstddecl}
namespace std::chrono {
  // 24時間時計の時間値を12時間時計の時間値に変換する
  constexpr hours make12(const hours& h) noexcept;

  // 12時間時計の時間値を24時間時計の時間値に変換する
  constexpr hours make24(const hours& h, bool is_pm) noexcept;
}
```

`make12()`は24時間時計における時間の値（`[0, 23]`）を12時間時計における時間の値（`[0, 11]`）に変換するもので、`mak24()`は12時間時計における時間の値（`[1, 12]`）を24時間時計における時間の値（`[0, 23]`）に変換するものです。

`make12()`にはそのまま`0 ~ 23`の範囲内の`hours`の値を渡すだけですが、`make24()`には`1 ~ 12`の値に加えてその値が午後の値であるかの`bool`値を渡す必要があります。

```cpp
int main() {
  std::cout << make12(11h) << '\n';
  std::cout << make12(23h) << '\n';
  std::cout << make24(9h, true) << '\n';
  std::cout << make24(9h, false) << '\n';
}
```
```
11h
11h
21h
9h
```

これらの関数に範囲外の値を渡したときに得られる値はどちらも未規定とされています。また、`is_am()`等と同様にこれらの関数も実質的に`hours`型の値の限られた範囲の値に対してのみ正しく動作します。

## タイムゾーンデータベース

C++20`<chrono>`ライブラリにおけるタイムゾーンの扱いについてはライブラリ機能の1で紹介していました。ここでは、そこで取り扱っていなかったタイムゾーンデータベースに関する部分について補足します。

### タイムゾーンデータベースの取得と利用

タイムゾーンデータベースとは全世界の各地域で使用されているタイムゾーンに関する情報を収集し、記録しているデータベースです。タイムゾーンに関する情報を取得する場合はこのデータベースから取得するのが確実かつ正確です。C++の`<chrono>`のタイムゾーンに関するAPIも同様にこのデータベースから情報を得ており、このデータベースそのものを直接的に扱うラッパも提供しています。

最新のタイムゾーンデータベースを取得するには`get_tzdb()`フリー関数を使用します。

```cpp
int main() {
  const auto& db1 = get_tzdb();
  const auto& db2 = get_tzdb();

  assert(&db1 == &db2); // 常にパスする
}
```

戻り値型は`const tzdb&`であり、プログラム内で共通のシングルトンオブジェクトとして存在している`tzdb`オブジェクトへの`const`参照が得られます。

`get_tzdb()`の呼び出しは、そのプログラムの最初の一回のみプログラム中のタイムゾーンデータベースの初期化（すなわち、シングルトン`tzdb`オブジェクトの初期化）を行い、その後初期化したタイムゾーンデータベースを保持する`tzdb`オブジェクトへの参照を返します。2回目以降の呼び出しは、初期化済みの`tzdb`オブジェクトへの参照を返すため、1つのプログラム中の`get_tzdb()`の呼び出しはタイムゾーンデータベースが更新されるまでの間同じオブジェクトへの参照を返します。

返される`tzdb`型は単純な構造体であり、次のように定義されています

```{style=cppstddecl}
namespace std::chrono {
  struct tzdb {
    string                 version;
    vector<time_zone>      zones;
    vector<time_zone_link> links;
    vector<leap_second>    leap_seconds;

    const time_zone* locate_zone(string_view tz_name) const;
    const time_zone* current_zone() const;
  };
}
```

`version`メンバ変数は、タイムゾーンデータベースのバージョン文字列を保持しています。

```cpp
int main() {
  const auto& db = get_tzdb();
 
  std::cout << db.version << '\n';
}
```

GCC14.1の出力

```
2024a
```

MSVC 2022 17.8の出力

```
2021a.27
```

全世界のタイムゾーンの情報を収集している団体は大きく3つあり、OSから利用されている（間接的にC++が利用する）ものはそのうちの2つ、IANA Time Zone Databaseとマイクロソフトの収集するタイムゾーンデーターベースのどちらかです。当然、後者はWindowsのみが利用しています。そのため、Windowsとそれ以外とでは微妙に結果が異なる場合があるかもしれません。ただし、C++の標準ライブラリとしてのタイムゾーンデータベースはIANA Time Zone Databaseに基づいて規定されているため、chronoのAPIを通して得られるタイムゾーンの情報には大きな差はないはずです。

ちなみに、IANA Time Zone DatabaseはGithub上で管理されています

- eggert/tz : https://github.com/eggert/tz

2つのメンバ関数`locate_zone()`と`current_zone()`は同名のフリー関数と同じ結果となります。

```cpp
int main() {
  const auto& db = get_tzdb();

  const time_zone* tz1 = db.current_zone();
  std::cout << tz1->name() << '\n';

  const time_zone* tz2 = db.locate_zone("America/New_York");
  std::cout << tz2->name() << '\n';
}
```
```
Asia/Tokyo
America/New_York
```

より正確には、`std::chrono::locate_zone()`と`std::chrono::current_zone()`は`get_tzdb()`の戻り値に対して対応する関数を呼び出すように実装されています。

`zones`メンバはデータベースに保存されているすべてのタイムゾーンについての情報を保持しており、`links`メンバは各タイムゾーンの略称に関する情報を保持しています。

```cpp
int main() {
  const auto& db = get_tzdb();

  for (auto tz : db.zonez | std::views::take(5)) {
    std::cout << tz.name() << '\n';     // タイムゾーン名
  }
}
```
```
Africa/Abidjan
Africa/Accra
Africa/Addis_Ababa
Africa/Algiers
Africa/Asmera
```

```cpp
int main() {
  const auto& db = get_tzdb();

  for (auto name : db.links | std::views::take(5)) {
    std::cout << name.name() << " : ";  // 略称
    std::cout << name.target() << '\n'; // 略称に対応する元のタイムゾーン名
  }
}
```
```
ACT : Australia/Darwin
AET : Australia/Sydney
AGT : America/Buenos_Aires
ART : Africa/Cairo
AST : America/Anchorage
```

タイムゾーンは全部で350個以上あるので、最初の5個だけを表示しています。略称を持たないタイムゾーンもあるので、`links`の長さは`zonez`の長さよりも短くなります。

タイムゾーンの略称は、`Asia/Tokyo`に対する`JST`や`Etc/UTC`に対する`UTC`などのように、`locate_zone()`で本来のタイムゾーン名の代わりに使用することができるものです。一部のタイムゾーン名は一般に略称で表されることの方が多いかもしれません（日本における`JST`と`UTC`はどちらも本来の名前よりも略称の方が浸透している）。

この結果を見ると分かりますが、タイムゾーンのリスト`zonez`及びタイムゾーン略称のリスト`links`は、予めソート（アルファベットの昇順）されています。

`leap_seconds`メンバはそのデータベースに保存されているうるう秒に関する情報を保持しています。

```cpp
int main() {
  const auto& db = get_tzdb();

  for (auto ls : db.leap_seconds
               | std::views::reverse
               | std::views::take(5))
  {
    std::cout << ls.date() << ": " << ls.value() << '\n';
  }
}
```
```
2017-01-01 00:00:00: 1s
2015-07-01 00:00:00: 1s
2012-07-01 00:00:00: 1s
2009-01-01 00:00:00: 1s
2006-01-01 00:00:00: 1s
```

`leap_seconds`も予めソート（時系列）されているため、うるう秒の挿入履歴を時系列で取得することができます。この例では、最新（2024年6月末時点）の履歴から5件を表示しています。なお、うるう秒は廃止されるらしいので、このリストは今後更新されることはなさそうです。

`tzdb::links`の要素型である`time_zone_link`および`tzdb::leap_seconds`の要素型である`leap_second`は次のように定義されている型です

```{style=cppstddecl}
namespace std::chrono {
  class time_zone_link {
  public:
    time_zone_link(time_zone_link&&)            = default;
    time_zone_link& operator=(time_zone_link&&) = default;

    // 未規定の追加のコンストラクタが定義されうる

    string_view name()   const noexcept;
    string_view target() const noexcept;
  };

  class leap_second {
  public:
    leap_second(const leap_second&)            = default;
    leap_second& operator=(const leap_second&) = default;

    // 未規定の追加のコンストラクタが定義されうる

    constexpr sys_seconds date() const noexcept;
    constexpr seconds value() const noexcept;
  };

  // どちらも、非メンバで比較演算子（== <=> etc）を定義している
}
```

### タイムゾーンデータベースのリストと更新

サマータイムをはじめとして、世界各地域で使用されているタイムゾーンは時とともに変化しています。したがって、ある時点で取得したタイムゾーンデータベースはしばらくたつと古くなってしまいます。短時間で処理を終えて直ぐに終了するプログラムならばそれを気にする必要はありませんが、1月や1年などの長時間稼働し続けるプログラムだとタイムゾーン情報を最新のものに追随したい場合があります。そのため、`chrono`ライブラリでもタイムゾーンデータベースの更新に対応しています。

まず、`tzdb`クラスの`.version`メンバによってローカルタイムゾーンデータベースのバージョンを取得することができ、`remote_version()`フリー関数によってリモートのタイムゾーンデータベースのバージョンを取得することができます。

```cpp
int main() {
  const auto& local_db = get_tzdb();
  std::string remote_ver = remote_version();

  std::cout << local_db.version << '\n';
  std::cout << remote_ver << '\n';
}
```

GCC14.1の出力

```
2024a
2024a
```

MSVC 2022 17.8の出力

```
2021a.27
2021a.27
```

プログラムを開始してすぐの時点ではこの2つのバージョンが違っているということはほとんど無いと思われます。長時間稼働していてリモートのタイムゾーンデータベースが更新された場合、この2つのバージョン文字列が異なるようになります。

そして、`reload_tzdb()`というフリー関数によってタイムゾーンデータベースの更新が行えます。

```cpp
int main() {
  // ローカルのタイムゾーンデータベース
  const auto& db_prev = get_tzdb();

  // タイムゾーンデータベースを更新して新しいものを取得
  const auto& db_latest = reload_tzdb();

  // 更新されていれば異なる
  assert(&db_prev != &db_latest);
}
```

`reload_tzdb()`はローカルのタイムゾーンデータベースを更新したうえで、更新後のタイムゾーンデータベースを返します。無駄打ちしてもしょうがないので、この関数はたいてい次のように呼ばれることになるでしょう

```cpp
void update_tzdv() {
  // リモートのバージョンが更新されていたら、リロードする
  if (get_tzdb().version != remote_version()) {
    reload_tzdb()
  }
}
```

`remote_version()`および`reload_tzdb()`の両関数はおそらくネットワーク接続を必要とします。そのため、オフライン環境ではうまく動かないでしょう。`reload_tzdb()`はその場合おそらく`runtime_error`例外を送出します（明確に規定はありませんが`remote_version()`も同様と思われます）。

タイムゾーンデータベースの更新後に古いデータベースが直ぐに削除されるわけではなく、そのプログラムの実行中はタイムゾーンデータベースのリストに保存されています。`get_tzdb_list()`フリー関数によってそのリストを取得することができます。

```cpp
int main() {
  // タイムゾーンデータベースのリストを取得
  auto& db_list = get_tzdb_list();
}
```

返されるのは`tzdb_list`型の`const`参照で、これはシングルトンオブジェクトを参照しています。あるプログラムの実行中最初の`get_tzdb_list()`の呼び出しによってシングルトンのタイムゾーンデータベースのリストは初期化され、リストの先頭にはその時点で最新のタイムゾーンデータベースを保持する`tzdb`型オブジェクトが挿入されます。`get_tzdb_list()`の2回目以降の呼び出しはそのオブジェクトへの参照を返します。

`reload_tzdb()`が呼び出されてタイムゾーンデータベースが更新されると、タイムゾーンデータベースのリストの先頭には更新されたデータベースが挿入され、古いタイムゾーンデータベースはその後ろに並ぶことになります。

`tzdb_list`型は`range`となるオブジェクトではありますが、操作はかなり限定的です。

```cpp
int main() {
  // タイムゾーンデータベースのリストを取得
  auto& db_list = get_tzdb_list();

  // リストの先頭（最新のデータベース）を取得
  const auto& db = db_list.front();

  // リストのイテレート
  for (auto& db : db_list) {
    ...
  }

  // 指定した要素の後ろの要素を削除
  // 戻り値は削除された要素の後ろの要素のイテレータ（もしくはendイテレータ）
  auto it = db_list.erase_after(db_list.begin());
}
```

あとは`cbegin()/cend()`もありますが、これだけの操作しかできません。インターフェース的には`forward_list`が近しく見えますが、異なる型となり内部がどのようなデータ構造を使用するかは規定がありません。最新のデータベースを得る場合は`.front()`を使用すればいいのですが、古いものが欲しい場合はイテレータを用いて使いたいバージョンのデータベースを引き当てる必要があります。

実際のところ、`<chrono>`のタイムゾーンデータベース周りの関数の中核にはこのタイムゾーンデータベースのリストが居ます。`get_tzdb()`は`get_tzdb_list().front()`を返しており、`reload_tzdb()`は取得したリモートタイムゾーンデータベースをこのタイムゾーンデータベースのリストの先頭に挿入しています。また前述のように、フリー関数の`locate_zone()`と`current_zone()`は`get_tzdb()`で得られるデータベースに対して対応するメンバ関数を呼び出します。

このような関係性と、大本のタイムゾーンデータベースのリストがプログラム中で一意なシングルトンオブジェクトであることから、`get_tzdb_list()`の呼び出しはスレッドセーフであり、`reload_tzdb()`の呼び出しは`tzdb_list`の`.front()`と`.erase_after()`の呼び出しに対してスレッドセーフであることが保証されています。

最新のタイムゾーンの情報が欲しいだけならフリー関数の`locate_zone()`と`current_zone()`を使用すればよく、タイムゾーンデータベースそのものを直接触る必要は無いでしょう。長期間稼働するプログラムでも、サマータイムやタイムゾーンの変化をハンドルするような処理を必要としなければ、タイムゾーンデータベースにアクセスする必要はありません。逆に必要となるのはそのようにタイムゾーンの変化に追随する必要がある日付時刻の処理を行っている場合です。さらにタイムゾーンデータベースの履歴を使用する必要があるのは、更新に伴ってタイムゾーンの名前が変化することが問題となる場合だと思われます。

## `sys_info`

タイムゾーンデータベース1つは各地域のタイムゾーンに関する情報を`zones`メンバにリスト（`std::vector`）として保持しており、その要素型は`time_zone`型です。`time_zone`型の操作についてはライブラリ機能1でも触れていましたが、そこでは`.to_sys()/.to_local()`を使用した時刻変換のみを説明していました。

`time_zone`型におけるシステム時刻とは`system_clock`の指す時刻のことで、これのタイムゾーンはUTC（ただしうるう秒カウント無し）となり、`time_zone`型におけるローカル時刻とはその`time_zone`オブジェクトが保持するタイムゾーンに基づく時刻となります（すなわち、UTC時刻に対してタイムゾーンを適用した時刻）。

`.to_sys()/.to_local()`はシステムUTC時刻と`time_zone`オブジェクトが表すタイムゾーンにおける時刻との間の相互変換を行うもので、そのような時刻変換のためにはタイムゾーンに関する詳細な情報が必要です。`time_zone`型が保持するそれらの情報は`.get_info()`によって取得でき、引数にシステム時刻を指定するとシステム時刻->ローカル時刻の変換に使用されるタイムゾーン情報が`sys_info`型のオブジェクトとして得られ、引数にローカル時刻を指定するとローカル時刻->システム時刻の変換に使用されるタイムゾーン情報が`local_info`型のオブジェクトとして得られます。

```{style=cppstddecl}
namespace std::chrono {
  // time_zone型の定義
  class time_zone {
  public:
    time_zone(time_zone&&) = default;
    time_zone& operator=(time_zone&&) = default;

    // 未規定の追加のコンストラクタが定義されうる

    // タイムゾーン名取得
    string_view name() const noexcept;

    // タイムゾーン詳細の取得
    template<class Duration>
    sys_info   get_info(const sys_time<Duration>& st)   const;

    template<class Duration>
    local_info get_info(const local_time<Duration>& tp) const;

    // システム時刻 <-> ローカル時刻の変換
    template<class Duration>
      sys_time<common_type_t<Duration, seconds>>
        to_sys(const local_time<Duration>& tp) const;

    template<class Duration>
      sys_time<common_type_t<Duration, seconds>>
        to_sys(const local_time<Duration>& tp, choose z) const;

    template<class Duration>
      local_time<common_type_t<Duration, seconds>>
        to_local(const sys_time<Duration>& tp) const;
  };
}
```

```cpp
int main() {
  // キリバス（UT+14）のタイムゾーン
  const time_zone* tz = locate_zone("Pacific/Kiritimati");
  const auto now = system_clock::now(); // 2024/07/07のある時刻とする
  sys_info tz_info = tz->get_info(now);
  
  std::cout << "begin: "  << tz_info.begin  << '\n'
            << "end: "    << tz_info.end    << '\n'
            << "offset: " << tz_info.offset << '\n'
            << "save: "   << tz_info.save   << '\n'
            << "abbrev: " << tz_info.abbrev;
}
```
```
begin: 1994-12-31 00:00:00
end: 32767-12-31 00:00:00
offset: 50400s
save: 0min
abbrev: +14
```

`sys_info`は集成体型でありちょうどこの5つのメンバ変数のみを持ちます。これらのメンバの型と意味は次のようになります

|メンバ|型|意味|
|---|---|---|
|`begin`|`sys_seconds`|この`sys_info`の情報が有効である期間の開始時刻。`[begin, end)`の範囲内の時刻に対してのみその`sys_info`は有効|
|`end`|`sys_seconds`|この`sys_info`の情報が有効である期間の終了時刻|
|`offset`|`seconds`|そのタイムゾーンの指定時刻におけるUTCとの差分（秒単位）|
|`save`|`minutes`|夏時間中のタイムゾーンの場合、夏時間外の時間に対して減ずるオフセットの値|
|`abbrev`|`std::string`|タイムゾーン名の略称|

`save`メンバは多くの場合は`0`となり、`0`ではない場合にはこの`sys_info`（あるいは、指定したタイムゾーンにおける指定した時刻）は夏時間内にあります。夏時間内にある場合、`offset - save`の値はその`sys_info`に対応する夏時間ではないタイムゾーンにおけるシステムUTC時刻に変換するためのオフセットとして使用できます。

ただし、この情報は信頼度の高いものではなく、あるタイムゾーンにおける夏時間内のUTC時刻を同じタイムゾーンで夏時間を適用しない場合のUTC時刻に変換するためのオフセットの正確な値は、そのタイムゾーンにおいて`save`メンバが`0`になっているような`sys_info`から取得することが推奨されます。通常それは、夏時間内にある`sys_info`の`[begin, end)`範囲外のUTC時刻を指定して、同じタイムゾーンに対して`.get_info()`を呼ぶことで取得できます。

```cpp
int main() {
  // ニューヨークのタイムゾーン情報を取得
  const time_zone* tz = locate_zone("America/New_York");
  const auto now = system_clock::now(); // 2024/07/07のある時刻とする
  sys_info tz_info = tz->get_info(now);
  
  std::cout << "begin: "  << tz_info.begin  << '\n'
            << "end: "    << tz_info.end    << '\n'
            << "offset: " << tz_info.offset << '\n'
            << "save: "   << tz_info.save   << '\n'
            << "abbrev: " << tz_info.abbrev;
}
```
```
begin: 2024-03-10 07:00:00
end: 2024-11-03 06:00:00
offset: -14400s
save: 60min
abbrev: EDT
```

この例のように、サマータイムを実施しているタイムゾーンに対してサマータイム中の時刻を与えて`.get_info()`すると、`save != 0min`となっている`sys_info`を取得できます。この場合の`[begin, end)`の範囲は丁度そのタイムゾーンにおける夏時間の期間になっており、`offset`の値はサマータイム中におけるUTCとのオフセットを示しており、`save`の値はその値から通常の（非サマータイムにおける）UTCとのオフセットに変換するための差分となっています（アメリカ東部夏時間EDTはUTC-4、非サマータイムのアメリカ東部標準時ESTはUTC-5、夏時間->通常へのオフセット差分は`-1h = -60min`）。

また、`zoned_time`（`time_point`値とタイムゾーン情報のペア）の`.get_info()`によって、`zoned_time`が保持するタイムゾーン情報を`sys_info`オブジェクトとして取得することができます。

```cpp
int main() {
  // JSTのタイムゾーン情報付きの時刻
  zoned_time zt{"Asia/Tokyo", system_clock::now()};

  // 保持するタイムゾーン情報をsys_infoで取得
  sys_info tz_info = zt.get_info();
  
  std::cout << "begin: "  << tz_info.begin  << '\n'
            << "end: "    << tz_info.end    << '\n'
            << "offset: " << tz_info.offset << '\n'
            << "save: "   << tz_info.save   << '\n'
            << "abbrev: " << tz_info.abbrev;
}
```
```
begin: 1951-09-08 15:00:00
end: 32767-12-31 00:00:00
offset: 32400s
save: 0min
abbrev: JST
```

これは、`zoned_time`が保持するタイムゾーン情報である`time_zone`のポインタを介した`get_info()`に、同じく保持する時刻情報（`sys_time<duration>`の値）を渡した結果を返しています。`zoned_time`の場合はこれによってそれが保持している時刻について適用されるタイムゾーン情報の詳細を取得することができます。

### `local_info`

`local_info`は`time_zone::get_inf()`にローカル時刻を指定したときの戻り値型です。この場合、指定されたローカル時刻はそのタイムゾーンにおける時刻だとして、そのローカル時刻からシステム時刻への変換に使用されるタイムゾーン情報が`local_info`型のオブジェクトとして得られます。

```{style=cppstddecl}
nemespace std::chrono {
  struct local_info {
    static constexpr int unique      = 0;
    static constexpr int nonexistent = 1;
    static constexpr int ambiguous   = 2;

    int result;
    sys_info first;
    sys_info second;
  };
}
```

`sys_info`同様にこちらも集成体型で、3つのメンバ変数しかありません。`result`は変換結果のUTCシステム時刻が一意に定まるかどうかを表し、`local_time`の`static constexpr`メンバ変数として定義されている3つの値のどれかを取ります。この値によって`first, second`の値とその意味が決まります。

- `result == unique` : ローカル時刻からシステム時刻への変換は1つに定まる
    - `first`: その変換に使用されるタイムゾーン情報
    - `second` : ゼロ初期化
- `result == nonexistent` : ローカル時刻からシステム時刻への変換は存在しない
    - そのローカル時刻は本来存在しない時刻
    - `first` : そのローカル時刻の直前の非サマータイムの`sys_info`
    - `second` : そのローカル時刻の直後のサマータイムの`sys_info`
- `result == ambiguous` : ローカル時刻からシステム時刻への変換が2つ存在する
    - `first` : そのローカル時刻の直前のサマータイムの`sys_info`
    - `second` : そのローカル時刻の直後の非サマータイムの`sys_info`

下2つの場合（`result != unique`）を考慮する必要があるのは、サマータイムのためです。ライブラリ機能1でも少し説明していましたが、あるタイムゾーンがサマータイムに入ると、UTCに対してのオフセットが変化します。それによって、サマータイムの入口ではUTCへの変換先が存在しない時刻が表現でき、サマータイムの出口ではUTCへの変換先が2つある時刻が表現できてしまいます。ライブラリ機能1で書いた説明をもう一度繰り返しておきます。

例えばアメリカでは、毎年3月の第2日曜の2時から11月の第2日曜の2時までの期間が夏時間で、問題となるのはこの開始と終了のタイミングです。

夏時間開始時には、その地域の標準時が1時間進められます。アメリカ（ニューヨークとします）の場合、毎年3月の第2日曜の1時59分59秒の1秒後は、3時0分0秒になります。従って、夏時間開始日のアメリカのタイムゾーンでは、2時から2時59分59秒の間の時刻が存在しません。しかし、カレンダーと時刻の計算によってそのような時刻をローカル時間上で作り出すことはでき、そのような時刻からUTCへ夏時間のタイムゾーンによって変換しようとすると、存在しない時刻からの変換を行うことになり、夏時間にあるタイムゾーンに対してこのようなローカルタイム値を渡すと`result == nonexistent`となります。

夏時間終了時には、その地域の標準時が1時間戻されます。アメリカの場合、毎年11月の第1日曜の1時59分59秒の1秒後は、1時0分0秒になります。従って、夏時間終了日のアメリカのタイムゾーンでは、1時から1時59分59秒が2回繰り返され、その間の時刻は2つ存在します。そのような時刻を指すローカル時間値からUTCへ夏時間のタイムゾーンによって変換しようとすると変換先が2つあり曖昧となるため、夏時間にあるタイムゾーンに対してこのようなローカルタイム値を渡すと`result == ambiguous`となります。例えばアメリカニューヨークの場合、EDT（東部夏時間）はUTC-4、EST（東部標準時）はUTC-5なので、11月の第2日曜（夏時間終了日）の1時30分の変換先はUTC 5時30分（夏時間中のEDT 1時30分）とUTC 6時30分（夏時間後のEST 1時30分）の2つとなります。

日本標準時など、サマータイムを考慮する必要がないタイムゾーンの場合、常に`result == unique`を仮定しても良いでしょう（そのようなタイムゾーンだけを扱う場合にこの型に触ることがあるのかはともかく・・・）。

```cpp
void print(const local_info& li) {
  std::cout << "result: " << li.result << '\n'
            << "first: " << li.first << '\n'
            << "second: " << li.second << '\n';
}

int main() {
  // ニューヨークのタイムゾーン情報を取得
  const time_zone* edt = locate_zone("America/New_York");
  
  // アメリカでは、3月第2週目の日曜2時から夏時間開始
  local_time lt1 = local_days{2024y/March/Sunday[2]} + 2h + 30min;
  local_info tz_info1 = edt->get_info(lt1);
  print(tz_info1);

  std::cout << '\n';
  
  // アメリカでは、11月第1週目の日曜2時で夏時間終了
  local_time lt2 = local_days{2024y/November/Sunday[1]} + 1h + 30min;
  local_info tz_info2 = edt->get_info(lt2);
  print(tz_info2);
}
```
```
result: 1
first: [2023-11-05 06:00:00,2024-03-10 07:00:00,-05:00:00,0min,EST]
second: [2024-03-10 07:00:00,2024-11-03 06:00:00,-04:00:00,60min,EDT]

result: 2
first: [2024-03-10 07:00:00,2024-11-03 06:00:00,-04:00:00,60min,EDT]
second: [2024-11-03 06:00:00,2025-03-09 07:00:00,-05:00:00,0min,EST]
```

この例でさらっと使用していますが、`sys_info`と`local_info`はともに`<<`によって`ostream`への出力を行うことができます。といっても、`sys_info`は`[]`に囲まれてメンバの値が出力されるだけで、`local_info`も同様ですが`result`の値によって一部のメンバのみの出力を行うだけです。

`result == 1`の場合の`first`および`result == 2`の場合の`second`はどちらも非サマータイムの`sys_info`（アメリカ東部標準時、EST）となっており、`result == 1`の場合の`second`および`result == 2`の場合の`dirst`はどちらもサマータイムの`sys_info`（アメリカ東部夏時間、EDT）となっています。このように、存在しない/曖昧なローカル時間が`get_info()`に渡された場合、そのようなローカル時間帯を挟む有効なタイムゾーンについての`sys_info`が2つのメンバに登録されており、そのローカル時間帯よりも前のタイムゾーン情報が`first`、後ろが`second`となります。

## `zoned_traits`

# `consteval`コンストラクタによるコンパイル時検査について

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- Compiler Explorer(https://godbolt.org/)
- yohhoyの日記(https://yohhoy.hatenadiary.jp/)
- 制約とコンセプトとオーバーロードと半順序(https://qiita.com/kazatsuyu/items/ea6b8f1c8c7d384505b8)
- IV. 命題論理の意味論（その２）(https://www2.yukawa.kyoto-u.ac.jp/~kanehisa.takasaki/edu/logic/logic4.html)

また、大阪C++読書会において以前に発行したC++20コア言語機能及びC++20ライブラリ機能1を輪読していただき、感想等のフィードバックを頂いております。モチベーションになっています、ありがとうございます！

- [大阪C++読書会 - connpass](https://cpp-osaka.connpass.com/ )
- [大阪C++読書会 - scrapbox](https://scrapbox.io/cpp-osaka/ )
