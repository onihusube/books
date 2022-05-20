---
title: C++20 ライブラリ機能（α）
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

## サンプルコードのお約束

- そこで主題となっているライブラリ機能のためのヘッダのみを明示的にインクルードし、他のヘッダのインクルードは省略します。
- 主題と関係ないところを`...`で省略していることがあります。

\clearpage

# `<concepts>`
\clearpage
# `<ranges>`

## Rangeコンセプト
## Rangeアダプタ
## Rangeアルゴリズム
\clearpage
# `<format>`
\clearpage
# `<bit>`
\clearpage
# `<numbers>`
\clearpage
# `<span>`
\clearpage
# `<source_location>`
\clearpage
# `<coroutine>`
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
void kernel(std::latch& sync, int id, std::span<const std::byte> input, std::span<int> output) {

  for (auto b : input) {
    // メインの処理（集計など）
    ...
  }

  // 出力前に全カーネル（スレッド）の完了を待機する
  sync.arrive_and_wait(); // 全カーネルに渡る同期ポイント

  // 結果を出力
  output[id] = ...;
}


int main() {
  // 処理対象のデータ、画像とか行列とかの2次元データとする
  // vectorの要素数=行数
  std::vector<std::span<std::byte>> input_data = ...;
  // 結果出力先
  std::vector<int> result = ...;
  // スレッドID
  int id = 0;

  // 処理対象のデータ数=立ち上げるスレッド数でラッチを初期化
  std::latch sync{input_data.size()};
  // 全スレッドの終了を待機するためのラッチ
  std::latch end{input_data.size()};

  // 入力データを行ごとに処理する
  for (auto span : input_data) {
    std::thread{[&sync, id, span, &result, &end]{
      // カーネルの実行
      kernel(sync, id, span, result);

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
void fork_proc(std::barrier<>& sync, std::span<const std::byte> input, std::stop_token st) {

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

`std::barrier`の`.arrive_and_wait()`の呼び出しでは、ラッチの時と同様にカウンタ値を1つ減算してからカウンタが0になるのを待機しますが、カウンタが0になった時はカウンタ値をリセット（コンストラクタで指定された値に戻す）してから待機中のスレッドを再開します。これによって、`std::barrier`は複数回同期に使用することができます。

Fork-Joinモデルのような並行処理を行う場合の同期ポイントでは何か同期が必要な処理を行いたいはずです。そしてそれはどこか1つのスレッドで全スレッドがその同期ポイントに到達してから実行する必要があるでしょう。これを行おうとすると、1つのスレッドを特別扱いするなど少し面倒になります。そこで、`std::barrier`はコンストラクタの第2引数でそのような処理（完了関数）を受けとり、カウンタが0になった時に実行させることができます。

```cpp
#include <barrier>

// 複数のスレッドで呼ばれる処理単位
template<typename CF>
void fork_proc(std::barrier<CF>& sync, std::span<const std::byte> input, std::stop_token st) {

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
void and_then_there_were_none(std::barrier<CF>& sync, int id, std::span<const bool> killed) {
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
  std::latch complete{N};

  // N個のスレッドを起動
  for (int id : std::views::iota(0, N)) {
    std::thread{[&sync, id, &complete, &killed]{
      and_then_there_were_none(sync, id, killed);
      complete.count_down();
    }}.detach();
  }

  // 全スレッド完了待機
  complete.wait();
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

どのように混ざるかはほぼランダムであり、たまに意図通りに出力されることもあります。

これを避けるためには`std::cout`に出力するところで同期を取る必要があります。C++20まではそのためにミューテックスによるロックを使用できましたが、出力1つにつき全スレッド間で同期を取る必要があるなどパフォーマンス面で使いづらく、かといって各スレッドに出力バッファを持っておいてそこに出力しておいてから最後にまとめて`std::cout`に出力、というのも実装が面倒でした。

C++20では、複数スレッドからの同期した標準出力のために`std::osyncstream`を利用できるようになります。`std::osyncstream`は`std::basic_osyncstream<charT, traits, Allocator>`の`char`特殊化のエイリアスであり、他にも`std::wosyncstream`が用意されています。以降は、これらを代表して`std::osyncstream`で説明を行います。

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

`std::osyncstream`はコンストラクタで渡された出力ストリームオブジェクトをラップして、それに対する出力についてグローバルに同期を取ります。しかし、その同期の単位は1度の出力（`<<`）ごとではなく、1つの`std::osyncstream`オブジェクトごとになります。つまり、1つの`std::osyncstream`オブジェクトに対する出力はバッファリングされており、そのデストラクタの実行時にまとめてラップしているストリームに出力しており、グローバルな動機はこの時（デストラクタにおける出力時）に行われています。

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
  std::osyncstream(cout) << "Hello, " << "World!" << '\n';
}

void example_emit() {
  std::osyncstream bout(cout);

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

`std::cout`などの場合はこのように使用したときでも安全（データ競合を起こしてストリームの状態が壊れたりしない）ですが、`std::osyncstream`の場合はそのような保証はなく、このように使用してしまうと`std::osyncstream`の持つバッファの状態が壊れ、未定義動作の世界に突入するでしょう。

\clearpage

# `<chrono>`
\clearpage
## カレンダー
## タイムゾーン
## `std::format`

\clearpage
# `<compare>`
## 比較関数オブジェクト

\clearpage
# `<version>`

`<version>`ヘッダでは、C++標準ライブラリのバージョンに関わる情報を提供します。

\clearpage
# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- C++マルチスレッド一巡り(https://zenn.dev/yohhoy/books/cpp-stdlib-multithreading)
- std::threadデストラクタ動作検討の歴史(https://zenn.dev/yohhoy/scraps/393bce83b4f3f0)
- threadの利用と例外安全（その1）(https://yohhoy.hatenadiary.jp/entry/20120209/p1)
- std::jthread and cooperative cancellation with stop token(https://www.nextptr.com/tutorial/ta1588653702/stdjthread-and-cooperative-cancellation-with-stop-token)