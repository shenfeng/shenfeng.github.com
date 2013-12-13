SOURCES    := ${wildcard *.dot}

SVGS := $(SOURCES:.dot=.svg)

%.svg: %.dot Makefile
	dot $< -Tsvg -o$@

all: $(SVGS)

clean:
	rm -vf $(EXES)
