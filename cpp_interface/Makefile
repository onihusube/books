all : book

book : document.md
	pandoc --toc -N -V geometry:a5paper -V geometry:margin=1in --pdf-engine=lualatex document.md -o book.pdf