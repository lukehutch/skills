---
name: latex-build
description: How to build a LaTeX document correctly — use latexmk (never a hand-rolled pdflatex/bibtex sequence), keep the trailing colon on BIBINPUTS/BSTINPUTS, and delete the build directory before reporting a page count or resolved cross-references. Use whenever compiling, building, or checking a .tex file, its bibliography, page count, or warnings.
---

# Building LaTeX

**Build with `latexmk`, and keep the trailing colon on `BIBINPUTS`/`BSTINPUTS`.** A hand-rolled pdflatex → bibtex → pdflatex → pdflatex sequence is one pass short from a cold start: the last pass still reports "Label(s) may have changed" and the page count is still wrong. `latexmk -pdf -interaction=nonstopmode -file-line-error -outdir=/tmp/latex-build <abs-path>.tex` reruns both programs until the cross-references stop changing, so there is no pass count to get wrong, and absolute paths make it independent of the working directory. `BSTINPUTS=.:<src>` *without* a trailing colon replaces kpathsea's default search path instead of extending it, so a style file living in the system tree (`plainnat.bst`, `apsrev4-2.bst`) is not found: bibtex stops with "I couldn't open style file", the bibliography silently drops, every `\cite` goes undefined, and the page count shrinks. Write `BIBINPUTS=.:<src>: BSTINPUTS=.:<src>:` and no style directory needs naming. Before reporting a page count or "all cross-references resolved", delete the build directory first — a stale `.aux` makes an unconverged build look clean.

The `.log` is ISO-8859, so search it with `command grep -a` (see the `grep` rule in `~/.claude/CLAUDE.md`).
