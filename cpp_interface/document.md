---
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
4. 関数の命名で悩まずにすみ、利用者にとっても自然な名前を選択できる

標準的インターフェースはある程度C++を書いていれば意識せずともなんとなく身についており、そうしたC++プログラマが書いたライブラリは自然と標準的インターフェースを意識したものになります。それを知っておけば、そのようなライブラリをあまり迷わずに使用でき、また、利用されやすいライブラリを書くことができるでしょう。そして、標準的インターフェースに基づいた世界でならばそれらライブラリの相互利用さえも自然に行うことができます。

本書は、この「なんとなく」な部分を明示的に探訪したものです。

ただし、何でもかんでもこの標準的インターフェースに合わせれば良いというものではありません。これもまた暗黙的な了解によるのですが、それぞれのインターフェースにはその意味論が確かに存在しています。あるインターフェースを選択した時は、その意味論に沿った処理を行い結果を返さなければなりません。これは、よく演算子オーバーロードで例に挙げられる、「`operator+()`を不適切にオーバーロードすれば`a + b`で引き算を行うことができる（もちろん、やらないことが推奨される）」というお話に通じるものがあります。

本書では、このようなインターフェースの意味論についても説明をしてあるつもりです。

## 標準的インターフェースへアダプトする、ということ

アダプト（適合）するとは、有るインターフェースを要求する処理や機構などに対して、自作の型などをそのインターフェースを備えることで利用可能にする、というような意味です。  
例えば、自作のコンテナクラスに対してそのイテレータを返す`begin(), end()`メンバ関数を用意しておけば、そのコンテナクラスのオブジェクトで範囲`for`文が利用可能になります（この事を、範囲`for`文にアダプトすると言う事ができます）。

標準的インターフェースにアダプトしておくことで、現在利用可能なこうした機構や将来利用可能になるかもしれない機構に対して意識せずにアダプトしておくことが出来ます。これは特に、自作のライブラリを提供する際にその利用のし易さに直結するでしょう。  
例えば、`begin(), end()`メンバ関数とイテレータによる要素列挙処理はC++03より以前からありましたが、範囲`for`文が使用可能になったのはC++11以降です。しかし、従来の列挙処理を利用していた型はなんの変更も必要とせず、そのまま範囲`for`文で利用する事が出来ました。

このことは逆に考えれば、あるインターフェースにアダプトしている型はそのインターフェースだけから統一的に（型の詳細を意識せずに）利用することができる、ということです。すなわち、そのような処理ではインターフェースが利用可能である限り入力となる型を自由に取り替えることできます。そのような処理は入力となる型の変更に対して頑健になるでしょう。

このような標準的インターフェースを知っておけば、C++に対する理解を一段と深めることが出来、より良いプログラムを書くことができるようになるでしょう。

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
template<std::range Iterable>
auto get_range(Iterable& range_obj) {
  return std::make_pair(std::ranges::begin(range_obj), std::ranges::end(range_obj));
}
```

この`std::ranges::begin(), std::ranges::end()`によってイテレータのペア（範囲）を取得可能な型は、`range`コンセプトを満たすことができます。すなわち、rangeライブラリの各種アダプタ・ビューなどで利用可能になります。
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

そして、イテレータ型はそのイテレータの制約の強さ（特性）に応じて階層的に種類が分けられており、備えているインターフェースもその段階的に限定されたものとなっています。

### 全イテレータ共通の型インターフェース

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


### forward iterator

```cpp
//参照先の要素型をTとするイテレータ
template<typename T>
struct forward_iterator {
  using difference_type   = std::ptrdiff_t;
  using value_type        = T;
  using pointer           = T*;
  using reference         = T&;
  using iterator_category = std::forward_iterator_tag;

  //イテレータを1つ進める
  forward_iterator& operator++();
  forward_iterator& operator++(int);

  //現在の要素を参照
  reference operator*();

  //イテレータの比較
  bool operator==(const forward_iterator&);
  bool operator!=(const forward_iterator&);
}
```

*forward iterator*（前進イテレータ）は一方向にのみ進む事の出来るイテレータであり、1つづつしか要素を進める事のできないものです。その名の通り、イテレータを1度進めたら戻ることが出来ません。これは最も基本的なイテレータであり、全てのイテレータは少なくとも*forward iterator*でなければなりません。すなわち、C++におけるイテレータは全て*forward iterator*として扱うことができます。

#### `operator++()`

これはイテレータを1つ進める操作です。インクリメント演算子と呼ばれるものをオーバーロードします。

#### `operator*()`

　

#### `operator!=()`

　

#### コピー可能

　


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

#### `operator--()`

　

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

#### `operator+=()`/`operator-=()`

　

#### `operator+()`/`operator-()`

　



　


--- コラム：Hidden Friends イディオム ---
上記*random access iterator*の非メンバの`operator+`のようにメンバ関数として定義する事の出来ない関数や、そもそもメンバ関数を最小にしたいなどの理由から、あるクラスにまつわる関数を非メンバ関数として外部の名前空間スコープに定義することがあります。この時、実質的にそのクラス専用の関数が名前空間スコープに存在することになり、コードの可読性が低下するだけでなく、`using namespace`や暗黙の型変換が絡むと意図しない関数呼び出しによる謎のバグを踏んでしまうかもしれません・・・。

この解決として、クラス定義内で`friend`関数としてそうした関数を定義するのが*Hidden Friendsイディオム*です。このような関数はそのクラスのメンバ関数ではありませんが、名前空間に展開されているフリー関数というわけでもありません。ADLでのみ使用可能となる不思議な関数になります。

```cpp
namespace MyNS {
  template<typename T>
  struct MyVector {

    using iteretor = /*略*/;

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
なお、`friend`であるので非`public`なメンバに全てアクセスすることができるので、`operator+=`なども含めた全てのメンバ関数を*Hidden Friends*にすることはできます。この塩梅は個人の好みによるかもしれませんが、メンバ関数としてのインターフェースとそうでないものを適切に選り分け、メンバ関数として定義した方がいいものは*Hidden Friends*にはしない方がいいでしょう。


\clearpage

# コンテナインターフェース

# 文字列インターフェース

# タプルインターフェース

タプルインターフェースを備えている型はtuple-likeな型と呼ばれます。`std::tuple`と同じように扱えると言う意味合いです。タプルインターフェースを備えておくことで、`std::apply()`や`std::make_from_tuple()`、そして何より構造化束縛へアダプトすることが出来ます。

## `std::tuple_size`

## `std::tuple_element`

## `std::get()`

# 関数呼び出しインターフェース

# `swap`インターフェース

# メタ関数のインターフェース