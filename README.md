# books

## ビルド

環境ごとの準備をした上で、対象のフォルダに移動し`make`を実行。同じフォルダにpdfファイルが生成される。

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
