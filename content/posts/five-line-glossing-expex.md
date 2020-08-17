---
title: "Latex: Five Line Gloss for Linguistics Typesetting"
date: 2019-01-16T00:49:17-04:00
author: Ze Xuan Ong
feature_image: /five-line-glossing-expex/feature.png
draft: false
tags: ["language"]
summary: "I'm not great with LaTex, but I need a five-line gloss for the conlang I'm building for Minecraft (for a class). Using the `expex` package, we can bootstrap a five-line gloss from the tools it provides out of the box."
---

In the midst of building my conlang for a class, I've ended up with a disparate collection of Word documents for various sections of my language. I figured it'd be a good time to compile everything into a nice LaTex document from hereon for future-proofing purposes (and of course, it looks much nicer to read).

## IPA Symbols

I started off using the TIPA package. It's quite convenient with the reference chart once you get the hang of it. However, it started becoming unwieldly halfway, in particular I found it difficult to get `\tipa` to work with glossing.

I have since changed to Unicode on XeLaTex. I think 

If you need more reasons to switch over to the dark side, [here](https://tex.stackexchange.com/questions/224164/typesetting-phonetic-symbols-unicode-or-tipa) is a good list of reasons why you should. To quote Jason Zentz, `[ˌɛkspləˈneɪʃən]` is much more readable (wait for it) than `\textipa{[""Ekspl@"neIS@n]}`.

## Glossing

I started with `gb4e` and `cgloss4e`. The issue here is that `gb4e` is limited to only 3 line glosses, including the header, and my class needs a 5 line gloss.
There did not appear to be a straightforward workaround to the problem.

Instead I switched to `expex`. `expex` provides approximately 3 gloss line classes, but they can be re-used to give more lines if necessary. For instance:

```
\ex
    \begingl
    \gla dʊro pa jʊma je jʊkmʊha {}//
    \glb dʊro pa jʊma je jʊk- mʊha//
    \glb 3\tss{RD}p.male NOM 1\tss{ST}p ACC HUM-HUM see//
    \glc He {} me {} {} see//
    \glft `He sees me'//
    \endgl
\xe
```

This gives a rather nice five line gloss:

{{< figure src="/five-line-glossing-expex/fiveline.png" caption="A nice five-line gloss! Gosh, why did we need five lines." >}}

It is also possible to modify the styles of `\gla`, `\glb`, and `\glc` respectively with the `expex` package. The full documentation can be found [here](http://mirrors.sorengard.com/ctan/macros/generic/expex/expex-doc.pdf).

Note: `\tss{}` is a shorthand for `\textsuperscript{}`. Typing less is always great
Note2: The `\ex` environment aligns words vertically using whitespaces. Empty 'word' spaces need to be highlighted using `{}`, which I guess is reasonable.

## Fonts

I'm rendering my work on ShareLatex. To use Unicode symbols, we have to compile using XeLaTex instead of the default LaTex. Then, a Unicode supporting font needs to be used as well. In this case, I'm using [Charis SIL](https://software.sil.org/charis/). To get the fonts to work, upload the downloaded fonts to the project directory, and make some adjustments to the settings in the header of the document:

```
\usepackage{expex}
\usepackage{fontspec}
\setmainfont[
    BoldFont={CharisSIL-B.ttf},
    ItalicFont={CharisSIL-I.ttf},
    BoldItalicFont={CharisSIL-BI.ttf}
]{CharisSIL-R.ttf}
```








