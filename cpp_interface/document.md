---
title: C++ 標準的インターフェース
author: onihusube
date: 2020/01/05
documentclass: bxjsarticle
---
\clearpage

# 標準的インターフェース

## 標準的インターフェースとは

この本でのインターフェスとは、ある共通の意味を持つようなメンバ関数であるとか、ある目的のために使用される特定のメンバ関数であるとか、そう言ったものです。動的ポリモルフィズムのために継承して利用するインターフェースクラス、という狭い意味のインターフェースではありません。

JavaやC#などのインターフェースクラスを利用した動的ポリモルフィズムでは、あるインターフェースを継承したクラスは必ずそのインターフェースを備えていることが分かり、逆に基底となっているインターフェースクラスを通してそのような限定的なインターフェースだけから異なる複数のクラスを統一的に扱います。これは、インターフェースクラスを継承させることによってクラスはそのインターフェースを備えていることを表明し、また、そのインターフェースクラスから利用することによって特定のインターフェースを備えている型だけを受け入れるように制約をかけている、と見ることが出来ます。

同じことはC++でも可能です。しかし、C++ではそのようなインターフェースクラスによる表明と制約を用いることは少なく、主としてテンプレートによるダックタイピングを多用します。この場合、ある型があるインターフェースを備えていることを表明する手段も、テンプレートな処理において入力される型に必要となるインターフェースを制約する手段もありません。ただし、C++20以降はコンセプトを用いることで後者だけは可能になりました。  
このことは、インターフェースクラスによる動的ポリモルフィズムに対して静的ポリモルフィズムと呼ばれます。

C++標準ライブラリだけを見ても、このように暗黙的に使用されているインターフェースがいくつもあります。本書ではこのことを「標準的インターフェース」と呼んでいます。また、標準ライブラリに新しいものが追加される際にもこれを意識した上でインターフェースが決定されています。

## 標準的インターフェースを利用するメリット

この標準的インターフェースを利用することで次のような利点を得ることが出来ます。

1. 変更に強いプログラムを書くことができる
2. 標準もしくは外部ライブラリで用意されている機構に意識せずともアダプトできる
3. テンプレートな処理において受け入れ可能な型の門戸を広げられる
4. 関数の命名で悩まずにすみ、利用者にとっても自然なインターフェースを選択できる

標準的インターフェースはある程度C++を書いていれば意識せずともなんとなく身についており、そうしたC++プログラマが書いたライブラリは自然と標準的インターフェースを意識したものになります。それを知っておけば、そのようなライブラリをあまり迷わずに使用でき、また、利用されやすいライブラリを書くことができるでしょう。そして、標準的インターフェースに基づいた世界でならばそれらライブラリの相互利用さえも自然に行うことができます。

ただし、何でもかんでもこの標準的インターフェースに合わせれば良いというものではありません。これもまた暗黙的な了解によるのですが、それぞれのインターフェースにはその意味論が確かに存在しています。あるインターフェースを選択した時は、その意味論に沿った処理を行い結果を返さなければなりません。これは、よく演算子オーバーロードで例に挙げられる、「`operator+()`を不適切にオーバーロードすれば`a + b`で引き算やかけ算を行うことができる（もちろん、やらないことが推奨される）」というお話に通じるものがあります。

## 標準的インターフェースへアダプトする、ということ

アダプト（適合）するとは、有るインターフェースを要求する処理や機構などに対して、自作の型などをそのインターフェースを備えることで利用可能にする、というような意味です。  
例えば、自作のコンテナクラスに対してそのイテレータを返す`begin(), end()`メンバ関数を用意しておけば、そのコンテナクラスのオブジェクトで範囲`for`文が利用可能になります（この事を、範囲`for`文にアダプトすると言う事ができます）。このような目的ために用意され要求されるインターフェースのことをカスタマイぜーションポイントと呼んだりします。

標準的インターフェースにアダプトしておくことで、現在利用可能なこうした機構や将来利用可能になるかもしれない機構に対して意識せずにアダプトしておくことが出来ます。これは特に、自作のライブラリを提供する際にその利用のし易さに直結するでしょう。  
例えば、`begin(), end()`メンバ関数とイテレータによる要素列挙処理はC++03より以前からありましたが、範囲`for`文が使用可能になったのはC++11以降です。しかし、従来の列挙処理を利用していた型はなんの変更も必要とせず、そのまま範囲`for`文で利用する事が出来ました。

このことは逆に考えれば、あるインターフェースにアダプトしている型はそのインターフェースだけから統一的に（型の詳細を意識せずに）利用することができる、ということです。すなわち、そのような処理ではインターフェースが利用可能である限り入力となる型を自由に取り替えることできます。そのように記述された処理は入力となる型の変更に対して頑健になるでしょう。

## 本書の内容について

本書ではC++17をベースとしてC++20の内容も踏まえながら、これらインターフェースのなんとなくな部分とその意味論について探訪するものです。ただし、紹介するインターフェースはよく使用されると思われる最小のものであり、全てを紹介してはいません。従って、C++20以降のコンセプトにおいて対応するコンセプトを満たすためにはさらにいくつかの関数等の実装が必要となることがあります。

また、ある言葉や概念をあえてあまり説明せずに登場させていることがあります。紙面の都合もありますが、これは~~C++沼に嵌めるため~~ C++のさらなる学習へのステップアップになればいいなあと思ってのことです。分からないことがあったら調べてみてください。google検索が一番早いと思います。詳しい人に聞くのも良いでしょう。twitterなどで私に聞いてくれても構いません。

本書で紹介するような標準的インターフェースについてを知っておくことで、C++に対する理解を一段と深めることが出来、より良いプログラムを書くことができるようになるでしょう。


\clearpage

# イテレータインターフェース

C++STLの柱でもあり、おそらく最も自然に出会うのがこのイテレータインターフェースではないかと思います。これを備えておくことで、STLが用意するイテレータを利用するアルゴリズムなどをそのまま利用可能になり、範囲`for`文やC++20以降のrangeライブラリなども利用可能になります。

## イテレート可能な型のインターフェース

イテレート可能な型というのは例えば`std::vector`のような、イテレータによってその要素を列挙することのできる型のことです。STLでは、イテレータを介すことでコンテナとアルゴリズムをお互いの実装詳細から分離し、なおかつコンテナからイテレータを引き出す操作は共通化されているため、あるアルゴリズムを使用する際にコンテナの詳細をほとんど意識する必要がなくなるようになっています。

イテレート可能な型というのは標準ライブラリに多数存在するコンテナのようにわかりやすく列挙可能な型だけにとどまらず、何かしらの要素の列を抱えていて、それをある順番で1つづつ走査する、という操作が可能な任意の型を指します。そして、そのような型のイテレータインターフェースとは、その要素列を参照するイテレータの取り出し口となるものです。  
標準ライブラリの変わったところでは`std::filesystem`の`(recursive_)directory_iterator`、`std::regex`の`regex_iterator`や`istream, ostream`に対する入出力ストリームイテレータがあります。特に、列挙する全ての要素は必ずしもイテレータを取り出した時点で決定していなくても構わず、時間軸方向にランダムに到着しても良いのです。純粋に列挙可能かどうかに注目すべきです。

### `begin(), end()`

　

```cpp
//何かイテレート可能な型
struct enumrable {

  iterator begin();
  
  iterator end();
};
```

イテレータインターフェースに必要なものはこの2つの関数だけです。`begin()`は列挙する要素列の先頭を、`end()`はその要素列の終端（一番最後の次）をそれぞれ指すイテレータを返すことが期待されます。それぞれを`first, last`という変数で受けたとすれば、イテレートする要素列の範囲は`[first, last)`という半開区間で表され、このような範囲を走査するようにイテレータを実装します。

なお、`end()`の返す型は異なっていても良く、範囲終端を表す（もしくは判定するための）特別なイテレータ型を返しても構いません。そのようなより一般的な意味での範囲の終端の事を番兵(*Sentinel*)と呼んだりします。

標準ライブラリのすべてのコンテナや文字列型など、多くのイテレート可能な型がこのイテレータインターフェースを持っています。

#### `std::being(), std::end()`

　

標準ライブラリにあるイテレート可能な型はみなイテレータインターフェースをメンバ関数として持っています。しかし、生配列はメンバ関数を持つことができないのでイテレート可能ではありますが、イテレータインターフェースを持ち合わせていません。  
そのために用意されているのが`std`名前空間に定義されているフリー関数版の`std::begin(), std::end()`のペアです。この関数は生配列に対してはポインタ操作によるイテレータを返し、それ以外の型に対してはメンバ関数の`begin(), end()`を呼び出すようになっています。  
このため、利用する側から見た真のイテレータインターフェースはこのフリー関数の方になるでしょう。

例えば次のように利用できます。

```cpp
//任意のイテレート可能な型から[first, last)の範囲のイテレータを引き出す
template<typename Iterable>
auto get_range(Iterable& range_obj) {
  return std::make_pair(std::begin(range_obj), std::end(range_obj));
}
```

このようにしておけば、この`get_range()`は生配列を含めた任意のイテレータインターフェースを持つ型に対して`[first, last)`なイテレータのペアを取得することができます。

#### あるいは、rangeインターフェース

　

ところで、型によってはメンバ関数としての`begin(), end()`が別の意味を持ってしまったり、そもそもメンバ関数を定義することを避けたりといった理由によってあえてフリー関数で`begin(), end()`を実装しているかもしれません。そのような型に対してもイテレータインターフェースを見つけ出すには少し工夫が必要です。

```cpp
template<typename Iterable>
auto get_range(Iterable& range_obj) {
  using std::begin;
  using std::end;

  return std::make_pair(begin(range_obj), end(range_obj));
}
```

このように`using`宣言を追加することによって、ADL（引数依存名前探索）によって`range_obj`の実際の型と同じ名前空間に定義されている`begin(), end()`を探し出してきて利用することができるようになります。もちろん、生配列およびメンバ関数に`begin(), end()`を持っている型に対しても引き続き正しく動作します。

とはいえ、このコードは色々問題があります。C++についてそこそこ詳しくないとこれが何をしているのか分かりませんし、そもそも知らないとこのようなコードの書きようがありません。また、一々このように書くのは面倒です。

そこで、C++20からは`std::ranges`名前空間に新しく`begin(), end()`が追加されます。この`std::ranges::begin(), std::ranges::end()`関数は上記で説明したことをすべて行う便利な関数です。つまり、入力された型が生配列ならポインタによって、メンバ関数のイテレータインターフェースが利用可能ならそれを呼び出し、ADLによってフリー関数の`begin(), end()`が利用可能ならばそれを使って、なんとかしてイテレータを引き出すものです。

C++20からはイテレータインターフェースの利用にあたってはこの関数を使うと良いかと思われます（ちなみに、これは実は関数ではなく関数オブジェクトだったりしますが、それはまた別のお話・・・）。逆に、イテレータインターフェースを用意する場合はメンバ関数で用意しておけばいいでしょう。

これらを利用すると先ほどの`get_range()`はだいぶすっきりします。

```cpp
template<std::ranges::range Iterable>
auto get_range(Iterable& range_obj) {
  return std::make_pair(std::ranges::begin(range_obj), std::ranges::end(range_obj));
}
```

この`std::ranges::begin(), std::ranges::end()`によってイテレータのペア（範囲）を取得可能な型は、`std::ranges::range`コンセプトを満たすことができます。すなわち、rangeライブラリの各種アダプタ・ビューなどで利用可能になります。
このような意味で、イテレータインターフェースは`range`インターフェースと呼ぶことができるかもしれません。


### `::iterator`

　

```cpp
//何かイテレート可能な型
struct enumrable {

  using iterator = /*提供するイテレータの型*/;

  iterator begin();
  
  iterator end();
};
```

イテレート可能な型は通常、そのイテレータの型についての情報を入れ子型として公開します。上記の`enumrable`クラスならば`enumrable::iterator`のように利用されます。`auto`が無い時代は複雑なイテレータの型を受けるためにこの入れ子型を利用していました。

```cpp
//任意のvectorの要素を列挙し標準出力に出力する、C++11より前のコード
template<typename T>
void output_vector(const std::vector<T>& vec) {
  typename std::vector<T>::iterator it = vec.begin(), end = vec.end();
  for (; it != end; ++it) {
    std::cout << *it << std::endl;
  }
}
```

現在ではイテレータ型を受けるためには`auto`を利用すればいいですし、このような典型的な列挙ループは範囲`for`文によってかなり簡易に書くことができます。

そのため、この入れ子型`::iterator`はイテレータインターフェースにとって必須ではありませんが、イテレータの型を取得する標準的な方法として提供しておくことで、`std::iterator_traits`（後述）を介してイテレータ詳細を引き出すための口として利用してもらうことができます。特に理由がなければ一緒に定義しておくといいかと思います。


## イテレータ型のインターフェース

イテレータ型というのは、イテレータそのものの型のことです。せっかくイテレータという概念でアルゴリズムとコンテナを分離しているのにイテレータによって操作が異なってしまうと抽象化の意味がありません。そのため、C++で利用されるイテレータ型のインターフェースは統一されています。

そして、イテレータ型はそのイテレータの制約の強さ（特性）に応じて階層的にカテゴリ分けされており、備えているインターフェースもそれによって段階的に限定されたものとなっています。

### 全イテレータ共通の型特性インターフェース

　

先ほどのイテレート可能な型でちらっと言っていましたが、イテレータ型はそのイテレータに関する情報をその入れ子型から取得することが出来ます。

```cpp
struct iterator {
  using difference_type   = /*イテレータの差分の型*/;
  using value_type        = /*イテレートする要素の型*/;
  using pointer           = /*イテレータする要素のポインタ型*/;
  using reference         = /*イテレータする要素の参照型*/;
  using iterator_category = /*イテレータの種類を表すタグ*/;
};
```

おそらくよく使うのは`value_type`と`iterator_category`ではないかと思います。これらの情報を取得出来るようにすることで、コンパイル時にそのイテレータの特性に合わせた処理の切り替えが行えるようしています。

ただし、イテレータがの典型例でもあるポインタ型はこのような入れ子型を提供できません。自分でテンプレートこちょこちょするコードを書けば取得はできますが面倒なので、標準ライブラリには`std::iterator_traits`という型特性クラスが用意されています。

```cpp
//通常のクラス型はこちらを利用
template <typename Iterator>
struct iterator_traits {
 using difference_type   = typename Iterator::difference_type;
 using value_type        = typename Iterator::value_type;
 using pointer           = typename Iterator::pointer;
 using reference         = typename Iterator::reference;
 using iterator_category = typename Iterator::iterator_category;
};

//ポインタ型はこっちを利用
template <typename T>
struct iterator_traits<T*> {
 using difference_type   = std::ptrdiff_t;
 using value_type        = std::remove_cv_t<T>;
 using pointer           = T*;
 using reference         = T&;
 using iterator_category = std::random_access_iterator_tag;
};
```

この`std::iterator_traits`クラスを通すようにすればポインタ型含めたあらゆるイテレータから共通の操作でイテレータに関する情報を引き出すことが出来ます。この場合でも、自分のイテレータ型に対しては先ほどのように入れ子型としてこれらを定義しておくだけでokです。なお、上級者向けではありますが、イテレータ型に入れ子で定義しなくても（もしくは、したくない場合に）、`std::iterator_traits`クラスをそのイテレータ型に対して明示的特殊化しても構いません。

標準ライブラリにもイテレータ型のこれらの情報によって使用する処理を切り替えるものがいくつかあるので、イテレータを実装する際はこれらの入れ子型を定義しておくべきです。

### forward iterator（input iterator）

　

```cpp
//参照先の要素型をTとするイテレータの一例
template<typename T>
struct input_iterator {
  using difference_type   = std::ptrdiff_t;
  using value_type        = T;
  using pointer           = T*;
  using reference         = T&;
  using iterator_category = std::input_iterator_tag;

  //コピー可能
  input_iterator(const input_iterator&);

  //イテレータを1つ進める
  input_iterator& operator++();
  input_iterator& operator++(int);

  //現在の要素の読み出し
  reference operator*();

  //イテレータの比較
  bool operator==(const input_iterator&);
  bool operator!=(const input_iterator&);
}


template<typename T>
struct forward_iterator : input_iterator<T> {
  using iterator_category = std::forward_iterator_tag;
};
```

*input iterator*（入力イテレータ）はイテレータの要件のなかで最小の（最も制約の弱い）もので、イテレータの基本的な要求を定めたものです。これは最も基本的なイテレータであり、全てのイテレータは少なくとも*input iterator*でなければなりません。すなわち、C++におけるイテレータは全て*input iterator*として扱うことができます。

*forward iterator*（前進イテレータ）は一方向にのみ進む事の出来るイテレータであり、1つづつしか要素を進める事のできないものです。その名の通りイテレータを1度進めたら戻ることが出来ません。これも*input iterator*でもあります。

この2つのイテレータでは以下のような操作が可能です。

```cpp
input_iterator<int> it{}, end{};

//イテレータを1つ進める
++it;
it++;
//要素を参照する
int  v  = *it;
//イテレータの比較
bool eq = it != end;
//イテレータのコピー
input_iterator cp = it;
```

*forward iterator*と*input iterator*には操作上の違いはありません。違いはその意味論にあります。*forward iterator*では*input iterator*の要件に加えて、__マルチパス保証__ という意味論的な制約が要求されます。それは以下のような要求です。

あるイテレータ`I`の値`a, b`について

- `a == b`ならば`++a == ++b`が成り立つ
    - これは、`I`が可変なイテレータ（後述）でも、その出力操作はイテレータに影響を与えないことを意味する
    - また、イテレータ同士の同値性はその参照先の要素の同値性と一致するという事でもある
        - `a == b -> *a == *b`かつ`*a == *b -> a == b`という事
- `((void)[](X x){++x;}(a), *a)`と`*a`は等価な操作となる
    - すなわち、イテレータをコピーして何をしてもコピー元のイテレータには影響がない

これはその名の通り複数の経路、すなわち複数のイテレータから同時に同じ要素列を同じ順序の上でイテレートできるという保証です。*input iterator*でしかないイテレータは、あるイテレータから要素を読み出したり、イテレータを進めたりすると、その対象の（参照する）要素列の状態が変更される事があります。*forward iterator*はそのような事が起きない事を表明しており、普通のイテレータであることの証とも言えます。イテレートするものが特殊でこれを満たせるかが分からなければ。*input iterator*として宣言しておくといいでしょう。

### bidirectional iterator

　

```cpp
template<typename T>
struct bidirectional_iterator : forward_iterator<T> {

  using iterator_category = std::bidirectional_iterator_tag;

  //イテレータを1つ戻す
  bidirectional_iterator& operator--();
  bidirectional_iterator& operator--(int);
}
```

*bidirectional iterator*は*forward iterator*であり、さらにイテレータを1つ戻す操作を行えるものです。ただし、あくまで今の位置から前後に1つづつ動く事ができるだけです。

このイテレータでは、*forward iterator*での操作に加えて以下のような操作が可能です。

```cpp
bidirectional_iterator<int> it{};

//一歩進んで二歩下がる
++it;
--it;
it--;
```

### random access iterator

　

```cpp
template<typename T>
struct random_access_iterator : bidirectional_iterator<T> {

  using difference_type   = std::ptrdiff_t;
  using iterator_category = std::random_access_iterator_tag;

  //イテレータをn進める/戻す
  random_access_iterator& operator+=(difference_type n);
  random_access_iterator& operator-=(difference_type n);

  //n進めた/戻したイテレータを計算する
  random_access_iterator& operator+(difference_type n);
  random_access_iterator& operator-(difference_type n);

  //イテレータ間の距離を計算する
  difference_type operator-(random_access_iterator&) const;
}

//n進めた/戻したイテレータを計算する（逆順の演算子）
template<typename T>
random_access_iterator<T>&
    operator+(typename random_access_iterator<T>::difference_type n,
              const random_access_iterator<T>&);
template<typename T>
random_access_iterator<T>&
    operator-(typename random_access_iterator<T>::difference_type n,
              const random_access_iterator<T>&);
```

*random access iterator*は*bidirectional iterator*であり、さらにイテレータを任意の数進める/戻す事ができるようになったものです。*random access iterator*はイテレート範囲を自由に動き回る事ができます。これの典型的な例は配列に対するポインタがそうです。

このイテレータでは、*bidirectional iterator*での操作に加えて以下のような操作が可能です。

```cpp
random_access_iterator<int> it{};

//イテレータを任意の数進める
it += 3;
it -= 2;

//任意の数進めたイテレータを得る
auto cp = it + 6;
cp = 6  + it
cp = it - 6;
cp = 6  - it

//イテレータ間の差分（イテレータ間距離）をとる
auto distance = cp - it;
```

#### コラム：Hidden Friends イディオム

　

上記*random access iterator*の非メンバの`operator+`のようにメンバ関数として定義する事の出来ない関数や、そもそもメンバ関数を最小にしたいなどの理由から、あるクラスにまつわる関数を非メンバ関数として外部の名前空間スコープに定義することがあります。この時、実質的にそのクラス専用の関数が名前空間スコープに存在することになり、コードの可読性が低下するだけでなく、`using namespace`や暗黙の型変換が絡むと意図しない関数呼び出しによる謎のバグを踏んでしまうかもしれません・・・。

この解決として、クラス定義内で`friend`関数としてそうした関数を定義するのが*Hidden Friendsイディオム*です。このような関数はそのクラスのメンバ関数ではありませんが、名前空間に展開されているフリー関数というわけでもありません。ADLでのみ使用可能となる不思議な関数になります。

```cpp
namespace MyNS {
  template<typename T>
  struct MyVector {

    using iteretor = T*;

    //Hidden Friendsなイテレータインターフェース
    friend iteretor begin(MyVector&);
    friend iteretor end(MyVector&);
  };
}

MyNS::MyVector v = {1, 2, 4};

auto frist1 = v.begin();      //ng
auto frist2 = MyNS::begin(v); //ng
auto frist3 = begin(v);       //ok
```

この不思議な性質によって、名前空間スコープを汚すこともなく、`using namespace`された時などに意図しない関数を呼び出してしまうことを防止することが出来ます。標準ライブラリでも`std::filesystem::path`の非メンバ`operator==(const path&, const path&)`が`using namespace std::filesystem`された時に文字列比較に取って代わるバグが発見され、このイディオムを用いて修正されました。それ以降、クラスのインターフェース、特にオーバーロードされた演算子はこのイディオムを利用するようになり、C++20では規格書に明記されるに至りました。

先ほどの`random_access_iterator`クラスで使ってみると例えば次のようになるでしょう。

```cpp
template<typename T>
struct random_access_iterator : bidirectional_iterator<T> {

  using difference_type   = std::ptrdiff_t;
  using iterator_category = std::random_access_iterator_tag;

  //イテレータをn進める/戻す
  random_access_iterator& operator+=(difference_type n);
  random_access_iterator& operator-=(difference_type n);

  //n進めた/戻したイテレータを計算する
  friend random_access_iterator& 
      operator+(const random_access_iterator&, difference_type n);
  friend random_access_iterator& 
      operator-(const random_access_iterator&, difference_type n);
  friend random_access_iterator& 
      operator+(difference_type n, const random_access_iterator&);
  friend random_access_iterator& 
      operator-(difference_type n, const random_access_iterator&);
}
```

クラス外で定義されていた`operator+`/`-`を*Hidden Friends*にし、対応する逆順の演算子も同様にしました。クラス定義内で定義できることによって記述がスッキリしているメリットがここで確認できるでしょう。  
なお、`friend`であるので非`public`なメンバ全てにアクセスすることができるため、`operator+=`なども含めた全てのメンバ関数を*Hidden Friends*にすることはできます。この塩梅は個人の好みによるかもしれませんが、メンバ関数としてのインターフェースとそうでないものを適切に選り分け、メンバ関数として定義した方がいいものは*Hidden Friends*にはしない方が良いかと思われます。

### output itereator

　

```cpp
//TをOutへ出力するoutput itereator
template<typename T, typename Out>
struct output_iterator : input_iterator<T> {
  using difference_type   = void;
  using value_type        = void;
  using pointer           = void;
  using reference         = void;
  using iterator_category = std::output_iterator_tag;

  //参照先への出力
  Out& operator=(const T& value);
}
```

上記4つのイテレータカテゴリに属するあるイテレータに対して、更に`operator=`による代入が可能であり、それが何らかの出力として（イテレータのコピーではない）意味を持つイテレータは*output itereator*でもあります。*output itereator*の`::iterator_category`は`std::output_iterator_tag`が使われます。

これは、以下のような操作が可能なものです。

```cpp
output_itereator oit{}; //何らかのoutput itereatorとする

oit = 'A'; //イテレータの参照先へ出力
```

この*output itereator*の要件を追加で満たすイテレータは、可変（*mutable*）なイテレータと呼ばれます。

この*output itereator*では、`::iterator_category`以外の入れ子型に`void`を指定して出力専用である事を表明することがあります。そのような場合`++oit`や`*oit`は意味を持たない事があります。また、そのように実装しても構いません。その場合、ある出力先へ順番に出力するという意味論だけを持つ事になります。

### 標準ライブラリのクラス・イテレータアダプタとイテレータカテゴリ
　

標準ライブラリに存在するイテレート可能な型のイテレータ、およびイテレータアダプタがどのようなイテレータとして扱えるのかを表にまとめました。標準コンテナやイテレータアダプタを使う際や自作の型にイテレータインターフェースを定義する際の参考になるかもしれません。

| 型名                             | *random access* | *bidirectional* | *forward* | *input* | *output* |
| -------------------------------- | :-------------: | :-------------: | :-------: | :-----: | :------: |
| `vector`                         | ◎               | ○               | ○         | ○       | △        |
| `array`                          | ◎               | ○               | ○         | ○       | △        |
| `initializer_list`               | ◎               | ○               | ○         | ○       | -        |
| `span`                           | ◎               | ○               | ○         | ○       | △        |
| `string`                         | ○               | ○               | ○         | ○       | ○        |
| `string_view`                    | ◎               | ○               | ○         | ○       | ○        |
| `deque`                          | ○               | ○               | ○         | ○       | △        |
| `list`                           | -               | ○               | ○         | ○       | △        |
| `forward_list`                   | -               | -               | ○         | ○       | △        |
| `set`                            | -               | ○               | ○         | ○       | -        |
| `multiset`                      | -               | ○               | ○         | ○       | -        |
| `map`                            | -               | ○               | ○         | ○       | △        |
| `multimap`                      | -               | ○               | ○         | ○       | △        |
| `unordered_set`                  | -               | -               | ○         | ○       | -        |
| `unordered_multiset`            | -               | -               | ○         | ○       | -        |
| `unordered_map`                  | -               | -               | ○         | ○       | △        |
| `unordered_multimap`            | -               | -               | ○         | ○       | △        |
| `regex_iterator`                 | -               | -               | △         | ○       | -        |
| `(recursive_)directory_iterator` | -               | -               | -         | ○       | -        |
| `back_insert_iterator`           | -               | -               | -         | ○       | ○        |
| `front_insert_iterator`          | -               | -               | -         | ○       | ○        |
| `insert_iterator`                | -               | -               | -         | ○       | ○        |
| `i(o)stream_iterator`            | -               | -               | -         | ○       | ○        |
| `i(o)streambuf_iterator`         | -               | -               | -         | ○       | ○        |

*random access iterator*が◎になっているものは、そのイテレータがさらに*contiguous iterator*（連続イテレータ、メモリ連続性を持つイテレータ）の要件を満たしていることを表しています。その他、△となっている所は常にその要件を満たしているわけではないことを表します。

\clearpage

# コンテナインターフェース

コンテナインターフェースとは、標準ライブラリに定義されている各種コンテナが持ち合わせているインターフェースのことです。その目的によってわかりやすい名前を付けられており、コンテナの間で可能な限り統一が図られています。そのようなインターフェースの中からよく使用されると思われるものを中心に`std::vector`を例として見ていきます。

なお、イテレータインターフェースは基本的に全てのコンテナが持ち合わせていますので省略します。

## 領域のチェック・変更

### `size()`

　
```cpp
template<class T, class Allocator = allocator<T>>
class vector {
  //宣言の例
  std::size_t size() const noexcept;
}
```

`size()`関数はコンテナが現在保持している要素数を返します。

```cpp
std::vector vec = {1, 2, 4, 8, 16};
vec.emplace_back(32);

std::size_t size = vec.size(); // 6
```

配列型では`size()`メンバ関数を利用することができないので、フリーの`std::size()`関数を利用します。こちらを使用すると標準コンテナと配列型とで共通の操作によって現在の要素数を取得できるようになります。

```cpp
int arr[] = {1, 2, 3, 5, 7, 11, 13};

std::size_t v_size = std::size(vec); // 6
std::size_t a_size = std::size(arr); // 7
```

この戻り値型は通常符号なし整数型ですが、符号付と符号なしの整数比較は安全ではなくコンパイラによっては警告を発するため、符号付整数型で取得したいことがあります。そのために、C++20からは`std::ssize()`というフリー関数が追加されます。これは各コンテナから`size()`によって取得した要素数を符号付整数型にキャストして返すものです。

```cpp
std::ptrdiff_t v_size = std::ssize(vec); // 6
std::ptrdiff_t a_size = std::ssize(arr); // 7
```

### `empty()`

　
```cpp
template<class T, class Allocator = allocator<T>>
class vector {
  [[nodiscard]]
  bool empty() const noexcept;
}
```

`empty()`関数はコンテナが空かどうかを調べる関数です。すなわち、要素数がゼロ（`container.size() == 0`）かどうかをチェックするのと等価な意味を持ちます。

```cpp
std::vector<int> vec{};

bool empty = vec.empty(); // true
```

これもやはり配列型では使用できないので、フリー関数の`std::empty()`が用意されています。これを使用すると標準コンテナと配列型とで共通の操作によってコンテナが空かどうかを判定できます。ただし、通常は要素数ゼロの配列を宣言できないので配列版は常に`false`を返します（要素数ゼロの配列を渡すとコンパイルエラーになります）。

```cpp
int arr[] = {1};

bool v_empty = std::empty(vec); // true
bool a_empty = std::empty(arr); // false
```

### `resize()`

　
```cpp
template<class T, class Allocator = allocator<T>>
class vector {
  void resize(std::size_t size, T c = {});
}
```

 `resize()`関数はその名の通りコンテナの現在の要素数を指定したものに変更する操作です。もし現在の要素数よりも小さい数字が指定された場合は要素列の末尾から指定した数になるように切り詰められます。逆に現在の要素数よりも大きい数が指定された場合はデフォルト構築された要素がコピーされて補われます。

 ```cpp
std::vector vec = {1, 2, 4, 8, 16};
vec.resize(3);  // {1, 2, 4}
vec.resize(6);  // {1, 2, 4, 0, 0, 0}
```

現在の要素数よりも大きい数が指定された場合に挿入される要素を指定することもできます。

 ```cpp
std::vector vec = {1, 2, 4, 8, 16};
vec.resize(3, 1);  // {1, 2, 4}
vec.resize(6, 1);  // {1, 2, 4, 1, 1, 1}
```


### `reserve()`

　
```cpp
template<class T, class Allocator = allocator<T>>
class vector {
  void reserve(std::size_t n);
}
```

`reserve()`関数はコンテナが追加のメモリ確保を行わなくても追加できる要素数を指定した数`n`に変更する操作です。ただし、指定した数ぴったりになるとは限らず、少なくとも`n`要素をメモリ確保せずに追加できるように内部の領域を拡張します。すでに要素がある場合はその要素のムーブを行いますが、現在の要素数を超えた領域に対しては何もせずにそのままにしておきます。

```cpp
std::vector<int> vec{};

vec.reserve(1000);
auto capa = vec.capacity(); // 1000 (clang9.0.0 & GCC9.2)

for (int i = 0; i < 1000; ++i) {
  vec.emplace_back(i);  //このループ中では追加のメモリ確保は発生しない
}
```

多くの実装では初項`1`、公比`r`の等比数列に従って内部領域が拡張されるようになっているようで（そうする事で償却計算量を*O(1)*にできるため）、`r`には`1.5`や`2`が使われるようです。そのため、`reserve()`をせずに上記のループによる要素追加を実行すると、完了までに少なくとも11回のメモリ確保が発生します。上記のように`reserve()`を予め行っておく事で1回のメモリ確保で済ませておく事ができます。これは意外と見落とされがちですが、大きなボトルネックの解消に繋がることが多々あります。

この`reserve()`インターフェースを備えている標準コンテナでは、`reserve(n)`の呼び出しの後では少なくとも指定した`n`個の要素はメモリ確保をせずに追加できることが保証されています。自作のコンテナに`reserve()`インターフェースを定義する場合はそのような保証を行うべきです。

ただし、この関数には内部領域縮小の機能はなく、必要ありません。現在の内部容量よりも小さい数が指定された場合は何もしません。

### 標準コンテナ対応表

　

上記4つのインターフェースは必ずしも全ての標準コンテナで使用可能ではありません。何で使えて何で使えないのかを一覧にしてみました。

| 型名                  | `size()` | `empty()` | `resize()` | `reserve()` |
| --------------------- | :------: | :-------: | :--------: | :---------: |
| `vector`              | ○        | ○         | ○          | ○           |
| `array`               | ○        | ○         | -          | -           |
| `string`              | ○        | ○         | ○          | ○           |
| `deque`               | ○        | ○         | ○          | -           |
| `list`                | ○        | ○         | ○          | -           |
| `forward_list`        | -        | ○         | ○          | -           |
| `set`                 | ○        | ○         | -          | -           |
| `multiset`           | ○        | ○         | -          | -           |
| `map`                 | ○        | ○         | -          | -           |
| `multimap`           | ○        | ○         | -          | -           |
| `unordered_set`       | ○        | ○         | -          | ○           |
| `unordered_multiset` | ○        | ○         | -          | ○           |
| `unordered_map`       | ○        | ○         | -          | ○           |
| `unordered_multimap` | ○        | ○         | -          | ○           |
| `stack`               | ○        | ○         | -          | -           |
| `queue`               | ○        | ○         | -          | -           |
| `priority_queue`      | ○        | ○         | -          | -           |

`forward_list`が`size()`を持たないのは、そのオブジェクトサイズを最小にするためです。`size()`に必要なメンバ変数を持った場合、その他の事に使用する事は無くオブジェクトサイズを無駄に太らせるだけという判断のようです。`forward_list` で`size()`相当の値を取得するにはイテレータを取得してその距離を求めます。`std::distance()`がその用途に使用できます。

## 要素アクセス

### `operator[]`（`at()`）

　
```cpp
template<class T, class Allocator = allocator<T>>
class vector {
  T& operator[](std::size_t n);
  T& at(std::size_t n);
}
```

`operator[]`は添字演算子と呼ばれるもので、C言語の生配列アクセスで使用されていたものがベースにあります。`c[1]`のように使用し、指定された添字に対応する要素への参照を返します。`std::map, std::unordered_map`がそうであるように、添字は必ずしも数字である必要はありません。コンテナの意味論において、その要素との対応付けがある何らかの値であれば良いでしょう。別の見方をすれば、この演算子は添字の集合からコンテナという要素の集合への写像となるものです。

```cpp
//vectorの場合
std::vector vec = {1, 3, 5, 7, 11, 13};

int p = vec[2]; //読み出し
vec[3] = 17;    //書き込み

//mapの場合
std::map<std::string, int> map{};
map.emplace("one", 1);

int n = map["one"]; //読み出し
map["two"] = 2;     //書き込み
```

`operator[]`を備えているコンテナは必ず`at()`関数も備えており、ほとんど同じ意味論を持ちます。その違いは、指定された添字に対応する要素が存在しない時の対応を誰が行うのかにあります。`operator[]`では、指定された添字に対応する要素が存在しない場合はコンテナの側で対応します。対して、`at()`関数は`std::out_of_range`例外を投げることでユーザーに対応を任せます。コンテナによっては、要素の存在チェックを行う分パフォーマンスが犠牲になります（現在のCPUでは強力な分岐予測のおかげでその影響はほぼないという説もありますが）。

```cpp
//vectorの場合
std::vector vec = {1, 3, 5, 7, 11, 13};

vec[6];     //未定義動作、範囲境界をチェックしない
vec.at(6);  //std::out_of_range例外を投げる、範囲境界をチェックする

//mapの場合
std::map<std::string, int> map{};
map.emplace("one", 1);

int n = map["two"];     //対応する要素をデフォルト構築して返す、n = 0
int m = map.at("two");  //std::out_of_range例外を投げる
```

自前の型に実装する場合、`operator[]`と`at()`関数は片方を備えるならばもう片方も備えておくことが推奨されます。その差については上記の通りにすべきでしょう。

### `data()`

　
```cpp
template<class T, class Allocator = allocator<T>>
class vector {
  T* data();
}
```

`data()`関数はコンテナ内で要素を保持しているメモリ領域へのポインタを得るものです。ただし、使用できるのは`std::vetor`や`std::array`のようにその領域がメモリ連続性を満たしているコンテナだけです。

```cpp
std::vector vec = {1, 3, 5, 7, 9, 11};

auto p = vec.data();

*p        //1
*(p + 1); //3
*(p + 5); //11
```

配列型も連続したメモリ領域に要素を保持していますが、メンバ関数を持てないのでフリー関数の`std::data()`が用意されています。これを使用すると標準コンテナと配列型とで共通の操作によってメモリ領域先頭へのポインタを取得できます。

```cpp
int arr[] = {1, 3, 5, 7, 9, 11};

auto p = std::data(arr);

*p        //1
*(p + 1); //3
*(p + 5); //11
```

メモリ連続性を満たしているとは、コンテナの保持する要素列がメモリ上でもその順番通りに連続して並んでいる事を言います。そにれより、その先頭のポインタからは通常のポインタ操作によって要素をイテレートすることができます。しかし、コンテナの特性によってはこれを満たすことができないこともあるので、そういう場合は定義する必要はありません。

### `front()`/`back()`

　
```cpp
template<class T, class Allocator = allocator<T>>
class vector {
  T& front();
  T& back();
}
```

`front()`/`back()`関数はコンテナの抱える要素列の先頭/末尾の要素への参照を得るものです。この先頭/末尾というのはコンテナの意味論においてのもので、必ずしも内部のメモリ領域上での順番とか追加された順番などである必要はありません。標準ライブラリでは、シーケンスコンテナと呼ばれる種類のコンテナがこれを備えています。  
これと同じことはイテレータを用いても簡単にできるのですが、要素列の先頭や末尾にアクセスしたいことは多々あるため、その近道となるコンビニエンス関数として定義されています。

```cpp
std::vector vec = {1, 3, 5, 7, 11, 13};

int f = vec.front();  //f = 1
int b = vec.back();   //b = 13
```

### 標準コンテナ対応表

　

| 型名                    | `operator[]/at()` | `data()` | `front()` | `back()` |
| --------------------- | :---------------: | :------: | :-------: | :------: |
| `vector`              | ○                 | ○        | ○         | ○        |
| `array`               | ○                 | ○        | ○         | ○        |
| `string`              | ○                 | ○        | ○         | ○        |
| `deque`               | ○                 | -        | ○         | ○        |
| `list`                | -                 | -        | ○         | ○        |
| `forward_list`        | -                 | -        | ○         | -        |
| `set`                 | -                 | -        | -         | -        |
| `multiset`           | -                 | -        | -         | -        |
| `map`                 | ○                 | -        | -         | -        |
| `multimap`           | -                 | -        | -         | -        |
| `unordered_set`       | -                 | -        | -         | -        |
| `unordered_multiset` | -                 | -        | -         | -        |
| `unordered_map`       | ○                 | -        | -         | -        |
| `unordered_multimap` | -                 | -        | -         | -        |
| `stack`               | -                 | -        | `top()`   | -        |
| `queue`               | -                 | -        | ○         | ○        |
| `priority_queue`      | -                 | -        | `top()`   | -        |

`stack`と`priority_queue`は`top()`という別の名前で対応する関数を備えています。

## コンテナの変更

### `push/pop`系インターフェース

`push_xxx()`関数群はその名の通りコンテナに要素を追加する関数です。既に構築済みの要素を受け取り、対応する内部領域にコピー（ムーブ）構築します。

`pop_xxx()`関数群はその名の通りコンテナから要素を削除する関数です。他のプログラミング言語における同じような関数と異なり、この関数は対応する場所の要素を削除をするだけで要素を取り出す機能は持っていません（例外安全への対応のためにこうなっているという噂があります・・・）。


#### `push_back()/push_front()`

　
```cpp
//vectorは全てを持ち合わせていないのでdequeでの宣言例
template<class T, class Allocator = allocator<T>>
class deque {
  void push_front(const T& x);
  void push_back(const T& x);
}
```

*front/back*の名前の通りこれらはそれぞれコンテナの要素列の先頭/末尾に要素を押し込むものです。要素アクセス関数の`front()/back()`と同じく、頻出する操作のためコンビニエンス関数として定義されています。

```cpp
std::deque deq = {3, 5, 7};

deq.push_front(1);  // {1, 3, 5, 7}
deq.push_back(11);  // {1, 3, 5, 7, 11}
```

要素列の先頭や末尾に要素を追加するという性質上、コンテナによってはその操作は非効率になってしまうことがあり得ます。例えば、`std::vector`は`push_front()`すると必ず領域の再確保が入ることになるため、`push_front()`を備えていません。

#### `pop_back()/pop_front()`

　
```cpp
template<class T, class Allocator = allocator<T>>
class deque {
  void pop_front();
  void pop_back();
}
```

こちらも*front/back*の名前の通りそれぞれコンテナの要素列の先頭/末尾の要素を削除するものです。

```cpp
std::deque deq = {1, 3, 5, 7, 11};

deq.pop_front(1);  // {3, 5, 7, 11}
deq.pop_back(11);  // {3, 5, 7}
```

これらも`push_front()`と同じような理由により一部のコンテナしか備えていません

#### 標準コンテナ対応表

　

| 型名                   | `push_back()` | `push_front()`   | `pop_back()`    | `pop_front()` |
| --------------------- | :-----------: | :--------------: | :-------------: | :-------: |
| `vector`              | ○             | -                | -               | -         |
| `array`               | -             | -                | -               | -         |
| `string`              | ○             | -                | -               | -         |
| `deque`               | ○             | ○                | ○               | ○         |
| `list`                | ○             | ○                | ○               | ○         |
| `forward_list`        | -             | ○                | ○               | ○         |
| `set`                 | -             | -                | -               | -         |
| `multiset`           | -             | -                | -               | -         |
| `map`                 | -             | -                | -               | -         |
| `multimap`           | -             | -                | -               | -         |
| `unordered_set`       | -             | -                | -               | -         |
| `unordered_multiset` | -             | -                | -               | -         |
| `unordered_map`       | -             | -                | -               | -         |
| `unordered_multimap` | -             | -                | -               | -         |
| `stack`               | -             | `push()`         | -               | `pop()`   |
| `queue`               | `push()`      | `pop()`          | -               | -         |
| `priority_queue`      | `push()`      | `pop()`          | -               | -         |

### `emplace()`系インターフェース

```cpp
template<class T, class Allocator = allocator<T>>
class deque {
  template<calss... Args>
  iterator emplace(const_iterator position, Args&&... args);
}
```

`emplace()`系関数は`push`系関数と同じくコンテナに要素を追加する関数ですが、受け取る引数が異なります。これらの関数は要素型のコンストラクタ引数を受け取り、指定した位置（`position`）に要素を直接構築します。
`push`系関数と比べると、要素を一度構築する必要が無い分効率的になります（単純な型ならば最適化で同等の処理になるようですが）。構築済みのオブジェクトを渡してもコピー（ムーブ）コンストラクタが呼ばれるため一見`push`系関数と同じように扱うことができます。

```cpp
//何か自作の型
struct myobj {

  myobj() = default;

  myobj(int n, double d, const char* chars)
    : num(n)
    , v(d)
    , str(chars)
  {}

  int num;
  double v;
  std::string str;
};


std::deque<myobj> deq{};

auto it = deq.emplace(deq.begin(), 10, 2.71, "emplace");  //コンテナ先頭に要素を直接構築

myobj mo{};

it = deq.emplace(it, mo);  //コンテナ先頭に要素をコピー構築
```

#### `emplace_front()/emplace_back()`

　
```cpp
template<class T, class Allocator = allocator<T>>
class deque {
  template<calss... Args>
  void emplace_front(Args&&... args);

  template<calss... Args>
  void emplace_back(Args&&... args);
}
```

*front/back*の名前の通りそれぞれコンテナの要素列の先頭/末尾へ`emplace`するものです。これも頻出する操作のためコンビニエンス関数として定義されています。


```cpp
std::deque<myobj> deq{};

deq.emplace(deq.begin(), 10, 2.71, "emplace");

deq.emplace_front(1, 0.0, "front");  //コンテナ先頭に要素を直接構築
deq.emplace_back(20, 3.14, "back");  //コンテナ末尾に要素を直接構築
```

#### `emplace_hint()/try_emplace()`

　
```cpp
//両方を備えているmapでの宣言例
template <
  class Key,
  class T,
  class Compare = less<Key>,
  class Allocator = allocator<pair<const Key, T> >
>
class map {
  template<calss... Args>
  iterator emplace_hint(const_iterator hint, Args&&... args);

  template<calss... Args>
  iterator try_emplace(const_iterator hint, const Key& k, Args&&... args);
}
```

`emplace_hint()`は要素を新しく挿入したい位置を指しているイテレータを渡すことで連想コンテナにおいての要素追加時の計算量を削減しようとするもので、`try_emplace()`は一部の連想コンテナにおいて指定されたキーを持つ要素が既に存在している場合には引数`args...`に一切手を付けない事を保証するものです。共に、標準ライブラリの連想コンテナにおいて提供されています。

連想コンテナは`std::vector`等のシーケンスコンテナと異なり、要素はその比較述語`Compare`に従う何らかの順序によって保持されています。そのため、要素を新しく挿入しようとしてもまずその挿入位置を決めてやる必要があり、位置を決めた後もコンテナ種別によっては要素の重複を許さない場合があります。前者の問題を可能ならば効率化しようとしたものが`emplace_hint()`であり、加えて後者の場合にも対処したものが`try_emplace()`になるわけです。  
すなわち、`emplace_hint()`では重複要素を許さないような連想コンテナに対して重複する要素を`emplace`した場合に渡したコンストラクタ引数`args...`がムーブされる可能性があり、`emplace_hint()`はそのような場合でも何もしない事を保証します。

```cpp
std::map itos = {
  std::make_pair(3, "three"),
  std::make_pair(5, "five"),
  std::make_pair(1, "one")
};

//1番目に要素を挿入しようとする、戻り値は既にある要素のイテレータ
auto it = itos.emplace_hint(itos.begin(), 1, "one");

//2番目に要素を挿入（直接構築）、戻り値は挿入した要素のイテレータ
it = itos.emplace_hint(++it, 2, "two");

//3番目に重複構築しようとする、戻り値は既にある要素のイテレータ
++it;
auto res = itos.try_emplace(it, 3, "three");
//4番目に要素を挿入（直接構築）、戻り値は挿入した要素のイテレータ
++it;
res = itos.try_emplace(it, 4, "four");

// itos = { {1, "one"}, {2, "two"}, {3, "three"}, {4, "four"}, {5, "five"} }
```

ここまで紹介した`emplace`系関数は特に顕著ですが、コンテナ間でその関数名が統一されていても関数引数や戻り値までも含めて（このことをシグネチャと言います）統一されてはいない事があります。また、ある関数についていくつかのオーバーロードを備えている事もあります。

| コンテナ種別       | `emplace()`                                                   |
| ------------ | ------------------------------------------------------------- |
| シーケンスコンテナ    | `template<calss... Args>`                                     |
|              | `iterator emplace(const_iterator position, Args&&... args);`  |
| 連想コンテナ       | `template<calss... Args>`                                     |
|              | `iterator emplace_hint(const_iterator hint, Args&&... args);` |
| 重複不可能な連想コンテナ | `template<calss... Args>`                                     |
|              | `pair<iterator, bool> emplace(Args&&... args);`               |
| 重複可能な連想コンテナ  | `template<calss... Args>`                                     |
|              | `iterator emplace(Args&&... args);`                           |
| `std::forward_list`  | `template<calss... Args>`                                     |
|              | `iterator emplace_after(const_iterator position, Args&&... args);`                           |

本書におけるインターフェースへの準拠とはほとんど関数名のみを指しています。その意味論さえ同じであればこのように引数や戻り値に多少の差やオーバーロードがあっても構わないでしょう。もちろん、統一できるのならばその方が良いです。

#### 標準コンテナ対応表

　

| 型名                    | `try_emplace` | `emplace_hint` | `emplace_front` | `emplace_back` | `emplace` |
| --------------------- | :-----------: | :------------: | :-------------: | :------------: | :-------: |
| `vector`              | -             | -              | -               | ○              | ○         |
| `array`               | -             | -              | -               | -              | -         |
| `string`              | -             | -              | -               | -              | -         |
| `deque`               | -             | -              | ○               | ○              | ○         |
| `list`                | -             | -              | ○               | ○              | ○         |
| `forward_list`        | -             | -              | ○               | -              | `emplace_after`|
| `set`                 | -             | ○              | -               | -              | ○         |
| `multiset`           | -             | ○              | -               | -              | ○         |
| `map`                 | ○             | ○              | -               | -              | ○         |
| `multimap`           | -             | ○              | -               | -              | ○         |
| `unordered_set`       | -             | ○              | -               | -              | ○         |
| `unordered_multiset` | -             | ○              | -               | -              | ○         |
| `unordered_map`       | ○             | ○              | -               | -              | ○         |
| `unordered_multimap` | -             | ○              | -               | -              | ○         |
| `stack`               | -             | -              | `emplace`     | -              | -         |
| `queue`               | -             | -              | -               | `emplace`    | -         |
| `priority_queue`      | -             | -              | -               | `emplace`    | -         |

`forward_list`はそのコンテナ特性上指定された要素の一つ前を参照する事にコストがかかるため、`emplace()`は指定された要素の1つ後ろに構築するようになっており、それを明示するために名前が変更されています。

### `insert()`

　

```cpp
template<class T, class Allocator = std::allocator<T>>
class vector {
  iterator insert(const_iterator position, cont T& x);
};
```

`insert()`はその名の通り指定した位置に要素を挿入する関数です。新しい要素は指定した位置に挿入する、つまりは指定した位置にある要素の1つ前に挿入されます。


```cpp
std::vector vec = {1, 3, 5};

auto it = vec.begin();
++it;

it = vec.insert(it, 2);

it += 2;

it = vec.insert(it, 4);

// vec = {1, 2, 3, 4, 5}
```

### `erase()`

　

```cpp
template<class T, class Allocator = std::allocator<T>>
class vector {
  iterator erase(const_iterator position);
  iterator erase(const_iterator first, const_iterator last);
};
```

`erase()`は要素を削除する関数です。削除したい要素はイテレータもしくはイテレータ範囲で指定し、指定された要素をコンテナから完全に削除します。

```cpp
std::vector vec = {1, 2, 3, 4, 5, 6, 7, 8};

auto it = vec.begin();
++it;

//2番目の要素を削除
it = vec.erase(it);
++it;

//3番目以降の要素を全削除
it = vec.erase(it, vec.end());

// vec = {1, 3}
```

#### `std::erase()/std::erase_if()`

　
```cpp
//vectorに対する宣言例
namespace std {
  template<class T, class Allocator, class U>
  void erase(vector<T, Allocator>& c, const U& value);

  template<class T, class Allocator, class Predicate>
  void erase_if(vector<T, Allocator>& c, Predicate pred);
}
```

実際のところ、イテレータを指定してピンポイントに削除したいということはあまりなく、特定の条件に合う要素をまとめて削除したいというケースの方が多かったりします。ところがそれをやろうとすると、コンテナによって削除のベストプラクティスとなる操作が全く異なったものになってしまいます（例えば、`std::vector`の*erase–remove idiom*など）。  
C++20よりそのような操作を統一的に行うために追加されたのが、この`std::erase()/std::erase_if()`フリー関数です。

`std::erase()`関数は値を受け取り、その値と`operator==`によって同値となる要素を削除します。そして、`std::erase_if()`は述語となる関数オブジェクトを受け取り、それによって`true`と判定される要素を削除します。どちらも、各コンテナ毎に最適な操作によって条件に合う要素を削除します。

```cpp
std::vector vec = {1, 2, 3, 4, 5, 6, 7, 8};

std::erase_if(vec, [](auto x){ return x % 2 == 0;});

// vec = {1, 3, 5, 7}
```

残念ながら、これらの関数にはカスタマイゼーションポイントが用意されていないため、ユーザー定義のコンテナに対してもこの関数によって条件付き削除を行うことはできません。ただ、これに倣ってフリー関数の`std::erase()/std::erase_if()`を用意しておくのはありでしょう。その場合、どこかで見たようなコードによって呼び出しを行えばジェネリックに書けるかもしれません。

```cpp
template<typename Container, typename U>
void my_erase(Container& c, const U& value) {
  using std::erase;

  erase(c, value);
}
```

### `clear()`

　

```cpp
template<class T, class Allocator = std::allocator<T>>
class vector {
  void clear();
};
```

`clear()`はコンテナの全ての要素を削除します。別の言い方をすると、コンテナを空にします。

```cpp
std::vector vec = {1, 2, 3, 4, 5, 6, 7, 8};

vec.clear();  //以降要素は無くなる
```

### 標準コンテナ対応表

　

| 型名                    | `insert()`       | `erase()`       | `std::erase_if()` | `clear()` |
| --------------------- | :--------------: | :-------------: | :---------------: | :-------: |
| `vector`              | ○                | ○               | ○                 | ○         |
| `array`               | -                | -               | -                 | -         |
| `string`              | ○                | ○               | ○                 | ○         |
| `deque`               | ○                | ○               | ○                 | ○         |
| `list`                | ○                | ○               | ○                 | ○         |
| `forward_list`        | `insert_after()` | `erase_after()` | ○                 | ○         |
| `set`                 | ○                | ○               | ○                 | ○         |
| `multiset`           | ○                | ○               | ○                 | ○         |
| `map`                 | ○                | ○               | ○                 | ○         |
| `multimap`           | ○                | ○               | ○                 | ○         |
| `unordered_set`       | ○                | ○               | ○                 | ○         |
| `unordered_multiset` | ○                | ○               | ○                 | ○         |
| `unordered_map`       | ○                | ○               | ○                 | ○         |
| `unordered_multimap` | ○                | ○               | ○                 | ○         |
| `stack`               | -                | -               | -                 | -         |
| `queue`               | -                | -               | -                 | -         |
| `priority_queue`      | -                | -               | -                 | -         |

`forward_list`はそのコンテナ特性上指定された要素の一つ前を参照する事にコストがかかるため、`insert()/erase()`は指定された要素の1つ後ろを対象にとるようになっており、それを明示するために名前が変更されています。また、`std::erase()/std::erase_if()`はセットで提供されているため、片方だけを表に書いています。


## 型特性インターフェース

```cpp
template<class T, class Allocator = std::allocator<T>>
class vector {
  using value_type = T;
  using iterator = /*イテレータの型*/;
  using size_type = std::size_t;
};
```

イテレータ型でも入れ子型によってその詳細を型で提供していたように、コンテナにおいても同様のことをしています。上記の3つはそれぞれ、要素の型、イテレータの型、要素数を表す型（`::size()`の戻り値型）を示しています。これらの事はある意味自明であり、型そのものに書いてあったり`decltype`すれば簡単に取り出せたりしますが、これらの情報が欲しくなることはままあるので、簡単な操作によって取り出すことができるように定義されています。

さらに、`std::map`等の連想コンテナの場合は要素としてキーと値のペアを保持するため`value_type`はテンプレートパラメータの`T`や`Key`とは異なったものになります。そのため、連想コンテナでは追加でそれらを取得するための型特性を持ちます。

```cpp
template <
  class Key,
  class T,
  class Compare = less<Key>,
  class Allocator = allocator<pair<const Key, T> >
>
class map {
  using key_type = Key;
  using mapped_type = T;  //map系連想コンテナのみ

  using value_type = std::pair<Key, T>;
  using iterator = /*イテレータの型*/;
  using size_type = std::size_t;
};
```

なお、本当はさらにいくつもの入れ子型が定義されています。ここで紹介しているのはよく使うであろう一部だけです。

## 検索やソートなど

C++の標準コンテナに対する検索やソートなどアルゴリズムの適用は基本的に`<algorithm>`ヘッダにあるフリー関数のものを利用する事で行います。連想コンテナの検索系のアルゴリズムやリスト系コンテナのリスト操作アルゴリズムのように、その特性上メンバとして持っているコンテナもありますが少数派です。`<algorithm>`や`<range>`ヘッダにはさらに多彩なアルゴリズムが定義されており、イテレータインターフェースを備えておけば利用する事ができるようになるので、基本的にはそちらを利用すべきです。

これは原初のSTLからのコンテナとアルゴリズムの詳細をイテレータを介して分離するというコンセプトによります。C++ではメンバ関数とフリー関数の区別をなくす仕組み（*Uniform Function Call*あるいは*Unified Call Syntax*）が度々検討されており、これが将来可能になればメンバ関数のように`<algorithm>`ヘッダにあるフリー関数のアルゴリズムを使用できるようになるかもしれません。

\clearpage

# タプルインターフェース

タプルインターフェースを備えている型はtuple-likeな型と呼ばれます。`std::tuple`と同じように扱えると言う意味合いです。タプルインターフェースを備えておくことで、`std::apply()`や`std::make_from_tuple()`、そして何より構造化束縛へアダプトすることが出来ます。

以下のような簡単な型をアダプトする事を考えてみます。

```cpp
template<class T1, class T2, class T3>
struct triple {
  T1 v1;
  T2 v2;
  T3 v3;
};

template<class T1, class T2, class T3>
triple(T1&&, T2&&, T3&&)
  -> triple<
       std::remove_cvref_t<T1>,
       std::remove_cvref_t<T2>,
       std::remove_cvref_t<T3>
     >;
```

## `std::tuple_size`

```cpp
// tupleの宣言例
namespace std {

  //プライマリーテンプレート
  template<class T>
  class tuple_size;

  //std::tupleに対する部分特殊化
  template<class... Types>
  class tuple_size<tupla<Types...>> : integral_constant<size_t, sizeof...(Types)> {};
}
```

これはtuple-likeな型の要素数を取得するものです。tuple-likeな型はその要素数は静的に（コンパイル時に）決まらなければならないのでクラステンプレートの静的変数として取得するようになっています。

先ほどの`triple`型では次のようにアダプトします。

```cpp
namespace std {

  template<class T1, class T2, class T3>
  struct tuple_size<triple<T1, T2, T3>> {
    static constexpr size_t value = 3;
  };
}
```

入れ子の静的変数`value`にその要素数を入れておきます。あるいはこれと同じ事を次のようにより簡易に書くこともできます。

```cpp
namespace std {

  template<class T1, class T2, class T3>
  struct tuple_size<triple<T1, T2, T3>> : integral_constant<size_t, 3> {};
}
```

`std::integral_constant`は静的な整数を表す標準ライブラリのクラスです。これの1つ目のパラメータに整数の型、2つ目のパラメータに要素数を与えた上で継承してやる事によって入れ子の静的変数`value`を提供できます。このテクニックはメタ関数転送と呼ばれています。

## `std::tuple_element`

```cpp
// tupleの宣言例
namespace std {
  //プライマリーテンプレート
  template<size_t I, class T>
  class tuple_element;

  //std::tupleに対する部分特殊化
  template<size_t I, class... Types>
  class tuple_element<I, tupla<Types...>> {
    using type = /*tupleのI番目の型*/;
  };
}
```

これはtuple-likeな型の指定された番号の要素型を取得するものです。これもコンパイル時に定まらなければなりません。

`triple`型に対しては少し複雑ですが、例えば以下のようにアダプトします。

```cpp
namespace std {
 
  template<class T1, class T2, class T3>
  struct tuple_element<0, triple<T1, T2, T3>> {
    using type = T1;
  };

  template<class T1, class T2, class T3>
  struct tuple_element<1, triple<T1, T2, T3>> {
    using type = T2;
  };

  template<class T1, class T2, class T3>
  struct tuple_element<2, triple<T1, T2, T3>> {
    using type = T3;
  };
}
```



## `get()`

```cpp
// tupleの宣言例
namespace std {

  template<size_t I, class... Types>
  constexpr tuple_element<I, tuple<Types...>>::type&
    get(tuple<Types...>&) noexcept; 
}
```

これはtuple-likeな型のオブジェクトから指定された番号の要素を取得するものです。

`triple`型に対してはこれも少し複雑ですが、例えば以下のようにアダプトします。

```cpp
template<size_t I, class T1, class T2, class T3>
constexpr std::tuple_element<I, triple<T1, T2, T3>>::type&
  get(triple<T1, T2, T3>& t) noexcept {
    if constexpr (I == 0) {
      return t.v1;
    } else if constexpr (I == 1) {
      return t.v2;
    } else if constexpr (I == 2) {
      return t.v3;
    } else {
      static_assert([]{return false;}, "I must be less than 3");
    }
  }
```

注意ですが、`get()`の場合は先ほどの2つのように`std`名前空間のものをオーバーロードするのではなく、対象の型（今は`triple`）と同じ名前空間にフリー関数として定義しておくか、メンバ関数として定義しておきます。また、*Hidden Friends*にしても良いでしょう。

```cpp
template<class T1, class T2, class T3>
struct triple {
  T1 v1;
  T2 v2;
  T3 v3;

  //こうしてもok
  template<size_t I>
  constexpr std::tuple_element<I, triple<T1, T2, T3>>::type&
    get() noexcept {
      if constexpr (I == 0) {
        return this->v1;
      } else if constexpr (I == 1) {
        return this->v2;
      } else if constexpr (I == 2) {
        return this->v3;
      } else {
        static_assert([]{return false;}, "I must be less than 3");
      }
    }
};
```

これらを完全に理解するには所謂テンプレートメタプログラミングの知識が必要になりますが、なんとかこれらを定義することができれば、自作の型を構造化束縛で使用できるようになります。

```cpp
triple t = {1, 3.1415, "triple"};

auto& [v1, v2, v3] = t;

std::cout << v1 << ", " << v2 << ", " << v3 << std::endl;
// 1, 3.1415, triple
```

#### コラム：構造化束縛へのアダプト

　

構造化束縛に任意の型をアダプトするにはもう一つ方法があります。それは、集成体のようにクラスのメンバ変数を`public`にしておくことです。そのような型のオブジェクトに対する構造化束縛宣言ではコンパイラが特別な計らいをして、非静的`public`変数だけをその宣言順に束縛してくれます。

```cpp
template<class T>
struct my_pair {
  T first;
  T second;
};

template<class T>
my_pair(T&&, T&&) -> my_pair<std::remove_cvref_t<T>>;


int main() {
  my_pair p = {10, 20};

  auto& [n1, n2] = p;
  
  std::cout << n1 << ", " << n2 << std::endl;
  // 10, 20
}
```

単に構造化束縛にアダプトしたいだけならばこちらの方がはるかに簡単に行えます。ところで、お気付きになられた方もいるでしょう。先程の`triple`型はあんな複雑な事をしなくてもそのまま構造化束縛で使用可能だったのです・・・  
カプセル化のためにメンバを隠蔽する場合や、一部のメンバを公開したくない場合などにタプルインターフェースを備えてみると良いでしょう。

\clearpage

# `swap`インターフェース

`swap`インターフェースは唯一つ`swap`という操作のためだけのものです。それはその名の通り、同じ型のオブジェクト同士あるいは互換性のある型のオブジェクト同士の間でその値を交換する操作です。

この`swap`という操作を用いるとコピーコンストラクタなどをより簡単に書く事ができたり、例外安全への対応が書きやすくなったりという恩恵があるので、`swap`はコピーやムーブよりもより基本的な操作だと見る向きもあります。

## ムーブコンストラクタ/代入演算子

```cpp
namespace std {

  template <class T>
  constexpr void swap(T& a, T& b) noexcept(/*略*/) {
    T t = std::move(a);
    a = std::move(b);
    b = std::move(t);
  }
}
```

デフォルトの`std::swap()`はこの様な典型的な交換操作によって`swap`します。従って、これにアダプトするために必要なのはムーブコンストラクタとムーブ代入演算子の2つです（ムーブが提供できない場合はコピーでも大丈夫です）。

```cpp
template<typename T>
struct wrap {
  T t;

  template<typename U=T>
  wrap(U&& u) : t(std::forward<U>(u)) {}

  //ムーブコンストラクタ
  wrap(wrap&&) = default;

  //ムーブ代入演算子
  wrap& operator=(wrap&&) = default;
};

template<typename T>
wrap(T&&) -> wrap<std::remove_cvref_t<T>>;


int main() {
  wrap a = {10};
  wrap b = {35};

  std::swap(a, b);
  // a = {35}, b = {10}
}
```

なお、ムーブに限らず代入演算子やコンストラクタはなるべくならば`default`実装にお任せするのが推奨されます。

## 非メンバ`swap()`

もしも自作のコンテナ等において、典型的なムーブ操作によるものよりも効率的な`swap`を提供出来る等の場合、`swap`操作そのものをカスタマイズすることもできます。そのためには、対象となる型と同じ名前空間で2引数を取る`swap()`というフリー関数を定義します。

```cpp
// 先程のwrap<T>に対するカスタマイズswap()定義
template<typename T>
void swap(wrap<T>& a, wrap<T>& b) {
  // しかし典型的な操作・・・
  wrap<T> t = std::move(a);
  a = std::move(b);
  b = std::move(a);
}
```

メンバ関数として書きたい気持ちもありますが、`swap()`の場合はフリー関数でなければなりません。ただし、ここでも*Hidden Friends*は有効です。

これを呼び出すだけならば単に名前空間名などを付けずに、`swap(a, b)`のように呼び出すだけです。しかし、よりジェネリックに`swap`するためにはどこかで見たようなコードを書かなければなりません。

```cpp
template<typename T, typename U>
void my_swap(T&& t, U&& u) {
  using std::swap;

  swap(std::forward<T>(t), std::forward<U>(u));
}

wrap a = {10};
wrap b = {35};

my_swap(a, b);
// a = {35}, b = {10}
```

こうすることで、最適な`swap()`関数が適切に呼び出されることになります。

`begin()/end()`と同じくこれはあまりにも分かり辛いので、C++20よりこの様な事をよしなにやってくれる`std::ranges::swap`が導入されます。これを使えば先程の`my_swap()`のような関数は必要なくなり、簡潔にジェネリック`swap`操作を書くことができるようになります。

```cpp
wrap a = {10};
wrap b = {35};

std::ranges::swap(a, b);
// a = {35}, b = {10}
```

\clearpage

# 関数呼び出しインターフェース

クラスに対して関数呼び出しインターフェースを備える事で、関数呼び出し構文によってそのオブジェクトから何かしらの処理を呼び出すことが可能になります。型によっては単なるメンバ関数呼び出しよりも直感的な操作となる場合があり、標準ライブラリの関数などはその動作のカスタマイズのために関数呼び出しインターフェースを備えた型の値を受け入れる構造になっている事があり、そのような所に渡しやすくなります。また、自作のライブラリにおいてもそのような簡易なカスタマイゼーションポイントとして利用する事ができます。

クラスに対して関数呼び出しインターフェースを備えるには、関数呼び出し演算子をオーバーロードします。

```cpp
struct functor {

  template<typename Arg>
  void operator()(const Arg& arg) {
    std::cout << arg << std::endl;  // 引数を出力
  }
};

functor f{};

f(10);  //関数呼び出し
```

関数呼び出し演算子は`operator()`という名前の関数として定義します。その戻り値および引数型、引数の数や処理内容などは完全に自由です。ほとんど通常のメンバ関数と同様に書く事ができます。ただし、クラス外に書くことはできません（従って*Hidden Friends*ではダメです）。そして、その型のオブジェクト名をさも関数名であるかのように書いてその処理を呼び出す事ができます。

このような関数呼び出しインターフェースは、標準ライブラリ（特に`<algorithm>`ヘッダ）の関数で多用されますが、関数のごく一部の動作をカスタマイズするのに利用されます。

```cpp
struct greater_than_10 {

  // 10よりも大きいかを判定する
  template<typename T>
  bool operator()(T arg) {
    return 10 < arg;
  }
};

std::vector vec = {5, 2, 4, 9, 1, 20, 3, 12, 7};

// 10よりも大きい最初の要素を探す
auto it = std::find_if(vec.begin(), vec.end(), greater_than_10{});
// *it = 20
```

この`std::find_if()`は次のように宣言されています。

```cpp
namespace std {
  template<class InputIterator, class Predicate>
  constexpr InputIterator find_if(InputIterator first,
                                  InputIterator last,
                                  Predicate pred);
}
```

この第3引数の`pred`がこの動作をカスタマイズする関数オブジェクトの受け口です。この場合に期待されているのは1引数を受けとり`bool`を返す関数です。このように、ある値がある条件に合っているかどうか判定する関数（オブジェクト）のことを述語（*predicate*）と呼びます。

また、先ほどのようなコードはラムダ式を用いてより簡単に書けます。

```cpp
std::vector vec = {5, 2, 4, 9, 1, 20, 3, 12, 7};

// 10よりも大きい最初の要素を探す
auto it = std::find_if(vec.begin(), vec.end(), [](auto arg){return 10 < arg;});
```

実のところ、ラムダ式は関数呼び出しインターフェースを備える型のオブジェクトをその場に生成しています。その処理はラムダ式の本体に書かれたものが入り、キャプチャした変数はその型のメンバとなります。先ほどの述語型を定義したコードと比べると対応がわかるでしょう。  
多くの場合、1から関数呼び出し可能な型を書くよりもラムダ式を使った方が同じことを簡便に書く事ができます。

## `std::invoke()`

```cpp
namespace std {

  template<class F, class... Args>
  constexpr invoke_result_t<F, Args...> invoke(F&& f, Args&&... args)
    noexcept(is_nothrow_invocable_v<F, Args...>); 
}
```

C++標準ではさらに、メンバポインタ及び`std::reference_wrapper`さえも関数呼び出し可能な型として扱います。そして、それらの呼び出しを統一的に書くために`std::invoke()`という関数を用意しています。例えばこのような呼び出しが可能です。

```cpp
struct S {
  int num;
  std::string str;
};

S s = {11, "eleven"};

// メンバ変数呼び出し、プロジェクションという操作
auto  num = std::invoke(&S::num, &s); // num = 11
auto& str = std::invoke(&S::str, &s); // str = "eleven"
```

この`std::invoke()`を利用すれば関数呼び出しインターフェースの範囲を広げる事ができます。典型的には次のようなコードを書く事で利用します。

```cpp
// あまり意味のない任意関数呼び出し
template<class F, class... Args>
decltype(auto) custom_func(F&& f, Args... args) {
  return std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
}
```

関数呼び出し可能な`f`とその引数`args...`を受けとって`std::invoke`に完全転送します。`f`にはラムダ式を渡しても、関数オブジェクトを渡しても、関数ポインタを渡しても、メンバポインタを渡しても、あるいは`std::reference_wrapper`を渡しても、引数`args...`が適切であれば関数呼び出しという操作を達成できます。

\clearpage

# メタ関数のインターフェース

メタ関数はテンプレートメタプログラミング（TMP）の文脈において関数に相当するものです。ただ、そこには通常の文脈における関数のように決まったものがある訳ではないので、何を関数とするのかには決まりがありません。しかし、その書き方についてはC++コミュニティおよびC++標準に渡って広く合意が形成されています。これはおそらく、STLライクなメタプログラミングライブラリである`boost.MPL`からのものです。

## 型を返すメタ関数

型を返すメタ関数はテンプレートの文脈においての普通の関数です。引数として型を受け取り、その処理結果を型として返します。

```cpp
// 入力の型にポインタ修飾を付加する
template<typename T>
struct add_pointer {
  using type = T*;
};

// メタ関数の呼び出し
using P = add_pointer<int>::type; // P = int*
```

メタ関数は通常クラステンプレートとして定義され、そのテンプレートパラメータに引数型を与え、入れ子型`::type`によってその戻り値を得ます。通常の関数呼び出しと見比べてみると何か見えてくるものがあるかもしれません・・・

```cpp
// メタ関数の呼び出し
using P = add_pointer<int>::type;

// 実行時関数の呼び出し
auto r = func(10);
```

このように、メタ関数名はそのまま関数名に、`<　>`の間のものが引数リストに対応し、`::type`を書くことで関数を呼び出します。このような書き方（および見方）が、ほぼデファクトスタンダードとなっているメタ関数の記法です。

しかし現在ではあえてこう書くことは珍しく、大抵はエイリアステンプレートを代わりに使います。しかし、この場合でも大元のメタ関数の書き方が変わる訳ではありません。

```cpp
// add_pointerのエイリアステンプレート
template<typename T>
using add_pointer_t = add_pointer<T>::type;

// メタ関数の呼び出し
using P = add_pointer_t<int>;
```

型を返すメタ関数のエイリアステンプレートを定義する場合は、元のメタ関数名の後ろに`_t`を付けます。

また、非型テンプレートパラメータによって型ではなく値を受け取る事ができ、引数は多変数にすることもできます。

```cpp
template<typename T, std::size_t N>
struct add_pointer_n_impl {
  using type = add_pointer_n_impl<T*, N - 1>;
};

template<typename T>
struct add_pointer_n_impl<T, 0> {
  using type = T;
};

// 型にN個のポインタを付け足す、2引数メタ関数
template<typename T, std::size_t N>
struct add_pointer_n : add_pointer_n_impl<T, N>::type {};


// メタ関数の呼び出し
using P = add_pointer_n<int, 4>::type; // P = int****
```

ますます関数っぽく見えてきませんか・・・？

## 値を返すメタ関数

値を受け取る関数があるのですから、当然メタ関数は値を返す事ができます。この場合、先ほど`::type`でメタ関数を起動していた所を、`::value`で起動するようにします。

```cpp
// 2つの型が同じかどうかを判定する
template<typename T, typename U>
struct is_same {
  static constexpr bool value = false;
};

template<typename T>
struct is_same<T, T> {
  static constexpr bool value = true;
};

// メタ関数の呼び出し、値を受け取る
constexpr bool b = is_same<int, int>::value; // b = true
```

現代では`std::true_type`などを使う事で定義を簡略化でき、さらに変数テンプレートによって呼び出しも簡略化できます。

```cpp
// 2つの型が同じかどうかを判定する
template<typename T, typename U>
struct is_same : std::false_type {};

template<typename T>
struct is_same<T, T> : std::true_type {};

// is_sameの変数テンプレート
template<typename T,  typename U>
inline constexpr bool is_same_v = is_same<T, U>::value;


// メタ関数の呼び出し、値を受け取る
constexpr bool b = is_same_v<int, int>; // b = true
```

値を返すメタ関数の変数テンプレートを定義する場合は、元のメタ関数名の後ろに`_v`を付けます。

## `<type_traits>`

標準ライブラリではよく利用されるメタ関数を`<type_traits>`ヘッダに定義しています。ここに定義されているメタ関数は全て上記のようなインターフェースに従った呼び出しが可能で、エイリアステンプレートや変数テンプレートの命名規則も同じです。

全てを書いて定義できる訳ではありませんが、このヘッダの実装は一度見てみると面白いかもしれません。

\clearpage

# 謝辞

本書を執筆するに当たっては、cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0に基づく)およびcppmap(https://cppmap.github.io/)をとても参照しました。サイト管理者及び編集者の方々に厚く御礼申し上げます。
