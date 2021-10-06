---
title: C++20 言語機能編
author: onihusube
date: 2021/12/30
geometry:
  width: 188mm
  height: 263mm
#coverimage: cover.jpg
#backcoverimage: backcover.jpg
titlecolor:
  color1:
    r: 0.796078
    g: 0.949020
    b: 0.4
    c: 0.25
    m: 0.0
    y: 0.80
    k: 0.0
  color2:
    r: 0.207843
    g: 0.631373
    b: 0.419608
    c: 0.75
    m: 0.0
    y: 0.65
    k: 0.0
okuduke:
  revision: 初版
  printing: ねこのしっぽ
---
\clearpage

# はじめに

# 新機能

## `<=>`

### 符号付整数型の内部表現は２の補数であると規定


## コンセプト

### 特殊メンバ関数の条件付きトリビアル定義

## モジュール

## コルーチン

# 定数式

## 仮想関数

## `dynamic_cast/typeid`

## `try-catch`

## 共用体のアクティブメンバ切り替え

## トリビアルなデフォルト初期化

## インラインアセンブリ

## `consteval`

## `constinit`

## 動的メモリ確保

# テンプレート

## `auto`による関数テンプレートの簡易定義

## `typename`の省略

## クラス型の非型テンプレート引数

## 集成体テンプレートのCTAD

## エイリアステンプレートのCTAD

## 条件付き`explicit`

# ラムダ式

## 暗黙のラムダキャプチャの簡易化

## テンプレート構文

## `[=, this]`

## ステートレスラムダ式のクロージャ型

## 評価されない文脈におけるラムダ式

## 初期化キャプチャのパック展開

## 構造化束縛した変数のキャプチャ

# 構造化束縛

## `static/thread_local`の許可

## 非公開メンバへのアクセス

## 呼び出す`get`メンバ関数の考慮の変更

# ユニコード文字型

## `char8_t`

## `char16_t/char32_t`

# 属性

## `[[no_unique_address]]`

## `[[likely]]/[[unlikely]]`

## `[[nodiscard("msg")]]`

## コンストラクタに対する`[[nodiscard]]`

# 集成体

## Designated Intilization

## `()`による集成体初期化

## ユーザー宣言コンストラクタの禁止

# 範囲`for`

## 初期化式の指定

## カスタマイゼーションポイント探索の変更

# Deprecating `volatile`

# その他

## 抽象型のチェック

## ビットフィールドのデフォルトメンバ初期化

## `using enum`

## `new`式における配列要素数の推論

## 要素数不明の配列型への変換  

## 入れ子`inline`名前空間定義の簡易化

## `__VA_OPT__`

## destroying operator delete

## 添字演算子内カンマの非推奨化

### C++23 Multidimensional subscript operator

# 謝辞

本書を執筆するに当たっては以下のサイトをとても参照しました。サイト管理者及び編集者・執筆者の方々に厚く御礼申し上げます。

- cpprefjp(https://cpprefjp.github.io/ : ライセンスはCC-BY 3.0)
- cppreference(https://ja.cppreference.com/w/cpp : ライセンスはCC-BY-SA 3.0)
- yohhoyの日記(https://yohhoy.hatenadiary.jp/)

表紙は友人のKさんに書いていただきました。可愛いキノコをありがとうございました！