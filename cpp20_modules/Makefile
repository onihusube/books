EMBED_JFONT = "YuMincho"
EMBED_MFONT = "Monaco"

ifeq ($(OS),Windows_NT)
    EMBED_JFONT = "yumin.ttf"
    EMBED_MFONT = "consola.ttf"
endif

all : book

book : document.md
	pandoc --toc --toc-depth 4 -N -V titlepage=true -V documentclass=bxjsarticle -V mainfont:$(EMBED_JFONT) -V CJKmainfont:$(EMBED_JFONT) -V monofont:$(EMBED_MFONT) -V CJKmonofont:$(EMBED_JFONT) --pdf-engine=xelatex --highlight-style=tango --template ../template.tex document.md -o book.pdf