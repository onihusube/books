all : book

book : document.md
	pandoc --toc -N -V geometry:height=154cm -V geometry:height=216cm -V geometry:margin=1in --pdf-engine=lualatex document.md -o book.pdf