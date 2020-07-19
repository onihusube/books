EMBED_JFONT = "YuMincho"
EMBED_MFONT = "Monaco"

ifeq ($(OS),Windows_NT)
    EMBED_JFONT = "Yu Mincho"
    EMBED_MFONT = "Consolas"
endif

COMMON_OPTION = --toc --toc-depth 4 -N -V titlepage=true -V documentclass=bxjsarticle -V mainfont:$(EMBED_JFONT) -V CJKmainfont:$(EMBED_JFONT) -V monofont:$(EMBED_MFONT) -V CJKmonofont:$(EMBED_JFONT) --highlight-style=zenburn --template ../template.tex document.md

#PC表示用PDF生成
pdf : document.md
	pandoc $(COMMON_OPTION) --pdf-engine=xelatex -o book.pdf

book.tex : document.md ../template.tex
	pandoc $(COMMON_OPTION) -M pdfx1a=true -M separate_cover=true -o book.tex

#入稿用PDF生成、texは2回実行しないと目次を生成しない・・・
.PHONY: book
book : book.tex
#-V 3はPDFのバージョン（1.XのX）指定、PDF/X-1a:2001はPDF1.3
	xelatex -shell-escape -output-driver="xdvipdfmx -V 3 -z 0" book.tex

.PHONY: clean
clean :
	rm book.* creationdate.* *.xmpi *.xmpdata