EMBED_JFONT = "YuMincho"
EMBED_MFONT = "Monaco"

ifeq ($(OS),Windows_NT)
    EMBED_JFONT = "YuMincho"
	EMBED_MFONT = "Consolas"
endif

all : book

book : document.md
	pandoc --toc -N -V geometry:width=154cm -V geometry:height=216cm -V geometry:margin=1in -V mainfont:$(EMBED_JFONT) -V CJKmainfont:$(EMBED_JFONT) -V monofont:$(EMBED_MFONT) -V CJKmonofont:$(EMBED_JFONT) --pdf-engine=xelatex --template template.tex document.md -o book.pdf