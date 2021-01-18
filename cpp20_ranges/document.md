---
title: C++20 <ranges>
author: onihusube
date: 2021/05/05
geometry:
  width: 188mm
  height: 263mm
#coverimage: cover.jpg
#backcoverimage: backcover.jpg
okuduke:
  revision: 初版
  printing: ねこのしっぽ
---
\clearpage

# はじめに

## Rangeライブラリとは

# コンセプト

## `range`
## `borrowed_range`
## `sized_range`

## `input_range`
## `output_range`
## `forward_range`
## `bidirectional_range`
## `random_access_range`
## `contiguous_range`
## `common_range`

## `view`
## `viewable_range`

# Rangeライブラリの構成部品

## Rangeアクセス関数

### イテレータ/番兵の取得

### 範囲サイズの取得

### 範囲のポインタ取得

## Range情報取得

### `iterator_t/sentinel_t`
### `range_difference_t`
### `range_size_t`
### `range_value_t`
### `range_reference_t`
### `range_rvalue_reference_t`

## `subrange`
## `view_interface`
## `dangling`

# Rangeファクトリ

## `empty_view`
## `single_view`
## `iota_view`
## `istream_view`

# Rangeアダプタ

## `all_view`
## `filter_view`
## `transform_view`
## `take_view`
## `take_while_view`
## `drop_view`
## `drop_while_view`
## `join_view`
## `split_view`
## `counted_view`
## `common_view`
## `reverse_view`
## `element_view`