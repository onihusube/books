# 書いた本のMarkdownソースとか

## ビルド

環境ごとの準備をした上で対象のフォルダに移動し、`cover.jpg, backcover.jpg`の2枚の適当な画像を用意してから`make`を実行。同じフォルダにpdfファイルが生成される。

### Windows

- [pandoc](https://pandoc.org/installing.html)
- [Tex](https://www.ms.u-tokyo.ac.jp/~abenori/soft/abtexinst.html)
    - [簡単LaTeXインストールWindows編（2016年4月版）- 情報科学屋さんを目指す人のメモ](https://did2memo.net/2016/04/24/easy-latex-install-windows-10-2016-04/)
    - 環境変数に`インストール先/w32tex/bin64`を追加したほうが良いかもしれない
- [Make](http://gnuwin32.sourceforge.net/packages/make.htm)
    - 必須ではない

### MacOS

- pandoc
   ```bash
   $ brew install pandoc
   ```
- Tex
    ```bash
    $ brew cask install mactex-no-gui
    ```
## フォルダとの対応

- C++標準的インターフェース : `./cpp_interface`
- C++20 Modules : `./cpp20_modules`
- C++ 集成体 : `./cpp_aggregate`
- C++20 ranges : `./cpp20_ranges`
- C++20 コア言語機能 : `./cpp20_lang`
- C++20 ライブラリ機能 1 : `./cpp20_lib`
- C++20 ライブラリ機能 2 : `./cpp20_lib2`
