EMBED_JFONT = "YuMincho"
EMBED_MFONT = "Monaco"

ifeq ($(OS),Windows_NT)
    EMBED_JFONT = "yumin.ttf"
    EMBED_MFONT = "consola.ttf"
endif

pdf : document.md
	pandoc --toc --toc-depth 4 -N -V titlepage=true -V documentclass=bxjsarticle -V mainfont:$(EMBED_JFONT) -V CJKmainfont:$(EMBED_JFONT) -V monofont:$(EMBED_MFONT) -V CJKmonofont:$(EMBED_JFONT) --pdf-engine=xelatex --highlight-style=tango --template ../template.tex document.md -o book.pdf

book.tex : document.md ../template.tex
	pandoc --toc --toc-depth 4 -N -V titlepage=true -V documentclass=bxjsarticle -V mainfont:$(EMBED_JFONT) -V CJKmainfont:$(EMBED_JFONT) -V monofont:$(EMBED_MFONT) -V CJKmonofont:$(EMBED_JFONT) -M pdfx1a=true --highlight-style=tango --template ../template.tex document.md -o book.tex

.PHONY: book
book : book.tex
#-V 4はPDFのバージョン（1.XのX）指定、PDF/X-1a:2001はPDF1.3
	xelatex -shell-escape -output-driver="xdvipdfmx -V 3 -z 0" book.tex

.PHONY: clean
clean :
	rm book.* creationdate.* *.xmpi *.xmpdata