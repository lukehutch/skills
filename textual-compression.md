---
name: textual-compression
description: Encode a text into a compact interlingua that another reader (human or model) can expand back into full prose, and decode such an encoding back into English. Use this skill whenever the user asks to compress, compact, densify, shrink, or "shorthand" a passage, to write a maximally dense summary intended to be re-expanded later, to decode or expand a compressed or symbolic text, or when they invoke /compress-text, /decompress-text, /ultra-compress-text, or /hanzi-compress-text.
---

# Textual Compression

An interlingua for lossy-but-recoverable text compression, and the procedures
for writing and reading it. Four commands, one shared model.

The target is not a summary. A summary discards; an encoding preserves every
claim in a form a cold reader can expand. Success is measured by what a reader
with no access to the original can reconstruct from the encoding alone.

## The underlying model

A decoder reconstructs from three sources, and an encoding is a set of
constraints that narrows what it can produce:

- **R1 grammar** — the decoder supplies fluent English. Never spend tokens on it.
- **R2 the reader's prior** — what the decoder already knows about the subject.
  Proper names, technical terms and canonical doctrines are pointers into it.
- **R3 the rest of the encoding** — each surviving element constrains its
  neighbors, so coverage is worth more than depth on any one point.

Two consequences drive everything below.

**Compressibility is a property of the source, not of the method.** A text
compresses in proportion to how much of it is redundant prose plus how much is
addressable through R2. Measured compression ratios, same method throughout:

| source                          | token ratio |
|---------------------------------|-------------|
| discursive philosophy           | 3.6x        |
| technical explanation (protocol)| 2.1x        |
| narrative history               | 1.3x        |
| pure mathematics                | 1.26x       |
| quantitative science            | 1.13x       |
| theory with defined terms       | 1.04x       |

Ratio falls as the density of irreducible specifics rises. A number, a date, a
proper noun and a defined term are already at minimum description length:
`2.725 K`, `Savitch`, `NP-complete` cannot be shortened without destroying
them. Below about 1.1x the method has stopped compressing and is only
reformatting; say so rather than claiming a compression that did not happen.
**Never quote a compression ratio without naming the domain.**

**Characters, bytes and tokens are three different axes, and the methods do
not rank the same on them.** One encoding of a 6530-character, 1259-token
discursive source, by each method:

| method                 | chars | bytes | tokens | char  | byte  | token |
|------------------------|-------|-------|--------|-------|-------|-------|
| source                 | 6530  | 6530  | 1259   | 1.00x | 1.00x | 1.00x |
| `/compress-text`       | 3428  | 3621  | 849    | 1.90x | 1.80x | 1.48x |
| `/ultra-compress-text` | 2291  | 2307  | 353    | 2.85x | 2.83x | 3.57x |
| `/hanzi-compress-text` | 682   | 1954  | 743    | 9.57x | 3.34x | 1.69x |

Read that table before promising a reduction, because the three columns
disagree about which method wins. Han script is nearly ten times shorter in
characters and worst but one in tokens. It still leads on bytes, by a smaller
margin than the character column suggests, because each Han character costs
three UTF-8 bytes against one for ASCII, which gives back most of the
advantage. The Latin methods track each other closely across all three, since
deleting an English word removes characters, bytes and tokens together.

Ask which axis the user is actually paying for. A display width or a field
limit is counted in characters; a database column, a network frame or a
storage bill is counted in bytes; a model context window is counted in tokens.
Quoting the flattering column and staying quiet about the other two is the
easiest way to mislead with this skill.

**Omission suppresses the prior; it does not fall back on it.** Leaving out a
claim because "the decoder knows it" makes the decoder *less* likely to
produce it than if nothing had been encoded at all — measured at 25 to 55
points of recall per dropped claim. An encoding is read as an exhaustive
inventory. Cover every claim, however briefly. This single rule outweighs
every token saved anywhere else.

## /compress-text — the default

Produce a symbolic-and-prose hybrid: explicit operators for the relations that
are part of the claim, ordinary words for everything else, one line per claim,
indentation for subordination.

### Encoding procedure

1. **Inventory first.** List every atomic claim in the source before writing
   anything. The inventory is the specification; the encoding is its rendering.
   A claim missing from the inventory will be missing from the output.
2. **Delete grammar, keep content.** Articles, copulas, auxiliaries and
   discourse connectives are all R1. Nouns, names, numbers, technical terms and
   negations are not — a dropped negation inverts a claim and costs more than
   any saving.
3. **Use layout, which is nearly free.** Newlines and indentation cost a token
   or two for a whole document and carry the hierarchy that would otherwise
   need sentences. Position encodes subordination; order encodes argument flow.
4. **Make relations explicit where the relation is the claim.** Entailment,
   non-entailment, contrast, constitution, intersection and exclusion between
   two ideas are content. Write them as operators. Leave merely connective
   relations in words.
5. **Point, don't restate.** Where the prior is genuinely strong, a name does
   the work of a paragraph: `Camus: absurd = <our demand> ∩ <world's silence>`.
   Where the prior is weak or the source's claim is non-standard, spell it out —
   a pointer to something the reader does not have retrieves nothing.
6. **Annotate ambiguity, never precision.** An emoji or a second glyph is worth
   its tokens only where the bare word is ambiguous in its context. Pairing two
   marks with overlapping senses intersects them onto the intended meaning, the
   way a two-character Mandarin compound does. On text whose vocabulary is
   already precise — technical, mathematical, quantitative — emoji measurably
   *hurt*: they invite reinterpretation of terms that had only one reading.
   Section-level markers are the exception and stay useful everywhere.
7. **Mark the source's own hedges.** Contested, disputed and uncertain claims
   must carry their status, or the decoder promotes them to assertions.
8. **Close the loop once.** State the upshot explicitly at the end. Do not
   restate anything else.

### What operators are for

They are **not** cheaper. Measured against a common tokenizer:

| glyph | tok | English | tok |
|-------|-----|---------|-----|
| `⊆`   | 2   | " in "  | 2   |
| `∧`   | 2   | " and " | 2   |
| `≠`   | 2   | " not " | 2   |
| `≔`   | 2   | " is "  | 2   |
| `⟺`   | 3   | " if and only if " | 5 |
| `∀`   | 1   | " every " | 2 |

Only quantifiers and long connectives win, and by one or two tokens.
Compression comes from deleting grammar, not from replacing words with marks.

What operators buy is **agreement between readers**. Keyword prose leaves the
relations between keywords implicit, so independent decoders each invent a
different set; explicit operators state the relation, so they converge.
Measured on the same source, three decodes each: plain keyword prose scored
97.1 with a standard deviation of 2.2; the symbolic hybrid scored 97.9 with a
standard deviation of 0.75. Same mean, a third of the spread. For a text
encoded once and read once that is worth little. For anything meant to be
read by several parties, or archived and expanded later, determinism is the
whole point.

**A symbolic encoding therefore needs more room than a keyword one, not less.**
Squeezing symbols into keyword length removes content, and the content loss
exceeds anything the notation recovers — measured at four points worse than
either pure style. If the budget is tight, choose keyword prose; do not choose
compressed symbols.

### Working glyph set

Any set may be used provided it meets the requirements below. This one is
tested:

`⇒` implies · `⇏` does not entail · `⟺` iff · `∧` and · `∨` or · `¬` not ·
`∃` there exists · `∄` there is no · `∅` absence of · `∩` intersection of ·
`∪` union · `∖` as against, not · `⊂ ⊃` contained in / contains ·
`∈ ∌` member / not a member · `≔` is defined as · `≠` differs from ·
`≻` beats, outranks · `≈` approximately · `∝` proportional to · `←` becomes ·
`↩` reply to the objection above · `|` separates parallel items ·
`⟨…⟩` groups a phrase into one operand

Requirements on any glyph set: each mark has exactly one reading in the
document; no mark collides with the subject matter's own notation (in
mathematics and physics the subject owns `∈`, `⊆`, `∀` — use words there
instead); every mark is pronounceable as an English clause; and the set is
small enough to hold in mind, roughly twenty marks.

Section markers earn their keep across all domains: `❓` problem, `📖`
definitions, `🏛` historical, `🔬` current state, `⚔` objections, `⚠` caution,
`🛡` counter-argument, `🧪` empirical evidence, `✅` upshot.

### Report with the output

Source tokens, encoded tokens, ratio, the domain, and any claim deliberately
dropped. If a tokenizer for the target model is unavailable, say which one you
used and call it a proxy.

## /decompress-text

Given an encoding, produce full English prose. The reader has no access to the
original and must not be told anything about it beyond subject and rough length.

1. **Read the whole encoding before writing.** Later lines constrain earlier
   ones; the layout carries the argument structure.
2. **Expand every element that carries a claim.** Each such line becomes at
   least one full sentence.
3. **Dissolve every glyph into prose; never name one and never keep one.** The
   output is ordinary English, so no mark from the encoding appears in it and
   no mark is described in it. The two glyph classes dissolve differently:
   - *Relational operators* carry content. Rewrite the relation into the
     grammar of the sentence. `A ⇏ B` becomes "a negative answer to the first
     question does not entail a negative answer to the second" — not "A does
     not-entail B", and never "A, arrow-with-stroke, B".
   - *Section and annotation marks* carry structure or tone, not content. They
     become a paragraph break, a heading, or a transition, and otherwise
     vanish. A section marker for objections becomes the sentence that opens
     the objections; it never becomes a word, and it is never mentioned.
   Two failures to avoid, both of which leave output that looks complete:
   copying a mark through, which loses the claim while keeping the symbol, and
   narrating a mark, which describes the notation instead of saying what it
   meant. Where a mark has more than one reading, choose the one the
   surrounding claim supports and write it as a full clause.
4. **Expand pointers into their content.** A proper name stands for the claim
   the encoder attached to it; state that claim, not the name alone.
5. **Restore hierarchy from layout.** Indentation is subordination: an indented
   line elaborates, qualifies or answers the line above it.
6. **Preserve the encoded modality.** Do not upgrade a marked-uncertain claim
   into an assertion, and do not add hedges to claims encoded flatly.
7. **Add nothing.** Where the encoding is silent, stay silent. Do not
   contribute your own knowledge of the subject; the output must be traceable
   to the encoding.
8. **Match the target length.** Terse encodings invite paraphrase rather than
   expansion; if the draft is short, the cause is under-expanded lines, not
   missing material.

## /ultra-compress-text

Maximum size reduction. Use only when the user asks for smallest possible
output and has accepted the loss.

This drops everything `/compress-text` keeps for determinism: no operators, no
sentence structure, no punctuation beyond section markers. The output is an
ordered bag of content words — nouns, names, numbers, technical terms,
negations — in source order, with emoji as the only section boundaries.

Encoding rules: keep every content word that carries a claim; delete every
function word; delete all punctuation; keep the source's order, because order
is the only remaining signal of what attaches to what; use one emoji per major
section and nothing else.

Measured on discursive philosophy: 1259 source tokens to 353, a ratio of 3.6x,
recall 96.5 percent — within noise of encodings three times its size. This is
the best size-to-fidelity result recorded.

Three warnings, each of which the result depends on:

- **The ratio does not transfer.** 3.6x came from a text of famous names and
  redundant argument. On quantitative or definition-heavy material the same
  method yields near 1.0x, because there are no function words to delete.
  Estimate the ratio from the domain table above before promising anything.
- **Variance is high.** Relations are entirely implicit, so decoders disagree
  about what attaches to what. Expect a spread roughly three times that of the
  symbolic form. Do not use it where two readers must agree.
- **It is unreadable to humans.** It works on a model with a strong prior and
  a decoding instruction. If a person must read the result, use
  `/compress-text` instead.

## /hanzi-compress-text

Encode into Han characters plus logic operators and section emoji, with no
English except proper names and technical terms that have no settled Chinese
form. Use it when the user asks for a Chinese-script encoding, or when the
budget being spent is characters or bytes rather than tokens. It is not the
method to reach for on fidelity or on token count; both sections below say why.

### What it is for

The Han script packs a clause into a fraction of the characters Latin script
needs, and the gap is large: on the source measured above, 682 characters
against 3428 for the hybrid and 2291 for the ultra form. Reach for it when
the constraint is characters, and when the constraint is bytes it is still the
leader, though by 3.3x rather than 9.6x. Do not reach for it to save tokens;
it is the weakest of the three on that axis.

It also costs fidelity. Budget about seven points of claim recall against the
hybrid, discussed under measured properties below.

### Encoding procedure

Follow the `/compress-text` procedure, with four changes.

1. **Cut the lexicon hard; leave the relations alone.** One character per
   content word wherever a single character carries it: 责 for duty, 财 for
   wealth, 荣 for flourishing, 渺 for insignificance. Drop 的, 之, 是, 而 and
   every other particle that is not doing semantic work. What must not be cut
   is the operator scaffolding — the marks that say which term stands in which
   relation to which. Measured: cutting 40 percent of the characters this way
   cost between one and two points of recall, while an earlier attempt that
   also thinned the connective structure cost more for a smaller saving.
2. **Choose characters that do not collide with grammar.** Prefer 旨 to 目的
   for purpose: it is one character instead of two, and 目的 ends in the
   possessive particle, so the decoder must disambiguate it at every
   occurrence. This consideration outranks which word is more idiomatic.
3. **Keep proper names in full, in their own script.** Hardin, Ostrom, Pauli,
   dharma, eudaimonia. Shortening the nine philosopher names in the measured
   source to one character each saved 15 characters and cost about a point of
   recall, concentrated in the claims attached to those names. A name is a
   pointer into the reader's knowledge and a fragment of it is a weaker
   pointer; there is nothing to gain by abbreviating one.
4. **Do not gloss ambiguous characters with their English term.** This looks
   like the obvious fix for 义 covering meaning, significance and
   righteousness. It was tried and measured: bracketed English anchors on
   seven such terms cost 38 tokens, moved mean recall by less than the noise,
   and nearly doubled the spread. The decoder was not failing to identify
   which English word a character stood for; it was failing to carry the
   relation between terms. Spend on clause structure instead.

### Decoding

`/decompress-text` applies unchanged. One addition: where a character maps to
several English terms the source held apart, decide from the relation it
stands in rather than from the character, and use one English term
consistently for it throughout. Switching between meaning and significance for
the same character across paragraphs is the characteristic failure here.

### Measured properties

Four encodings of the same 1042-word discursive source, decoded three times
each by readers with no access to the original:

| variant                        | chars | tokens | recall | spread |
|--------------------------------|-------|--------|--------|--------|
| modern Mandarin, spaced        | 1058  | 996    | 91.5   | 0.6    |
| classical register             | 848   | 848    | 89.9   | 2.0    |
| dense, names cut to 1 character| 667   | 727    | 89.3   | 0.2    |
| **dense, names in full**       | 682   | 743    | 90.5   | 1.3    |

The last row is the recommended form and the one the rules above describe. It
is 36 percent shorter in characters than the first row and within about a
point of it on recall.

Against the other two methods on the same source, the hybrid scores 97.9 and
the ultra form 96.5. So the Han method gives up roughly seven points of claim
recall for its character advantage. Two things to tell the user before using
it.

- **The loss is real and it reproduced across every variant.** Encoding into
  another language and decoding back adds a second lossy step that
  same-language compression does not have, and the mapping is many-to-one
  going in, so the decoder cannot invert it. The residual failures concentrate
  in claims that require holding several terms apart across a distance. Note
  that the spread is good — decoders agree with each other — but they agree on
  a reconstruction that has lost more, which reads as reliability and is not.
- **It does not save tokens at any scale, and below document scale it costs
  them.** The three worked paragraphs below come out at 0.99x, 0.94x and 0.87x
  in tokens, at or below parity, while running 4 to 5x shorter in characters.
  A short technical paragraph has no redundant prose to remove, so nothing
  offsets the tokenizer's per-character charge.

## Worked examples

Three paragraphs of comparable length from different domains, each encoded
all three ways and measured on all three axes. Token counts come from a common
tokenizer used as a proxy, and every count is of the block exactly as shown:
the line wrapping is part of what is measured, and costs the ultra form a few
tokens it would not pay as a single line. Character and byte counts exclude
the newlines.

Tokens, with the source count first:

| paragraph               | original | /compress-text | /ultra-compress-text | /hanzi-compress-text|
|-------------------------|----------|----------------|----------------------|---------------------|
| tragedy of the commons  | 150      | 147 (1.02x)    | 91 (1.65x)           | 151 (0.99x)         |
| B-tree indexes          | 136      | 127 (1.07x)    | 84 (1.62x)           | 144 (0.94x)         |
| the Chandrasekhar limit | 142      | 124 (1.15x)    | 91 (1.56x)           | 164 (0.87x)         |

Characters:

| paragraph               | original | /compress-text | /ultra-compress-text | /hanzi-compress-text|
|-------------------------|----------|----------------|----------------------|---------------------|
| tragedy of the commons  | 774      | 587 (1.32x)    | 485 (1.60x)          | 146 (5.30x)         |
| B-tree indexes          | 683      | 464 (1.47x)    | 461 (1.48x)          | 139 (4.91x)         |
| the Chandrasekhar limit | 718      | 493 (1.46x)    | 480 (1.50x)          | 174 (4.13x)         |

Bytes, UTF-8:

| paragraph               | original | /compress-text | /ultra-compress-text | /hanzi-compress-text|
|-------------------------|----------|----------------|----------------------|---------------------|
| tragedy of the commons  | 774      | 619 (1.25x)    | 495 (1.56x)          | 394 (1.96x)         |
| B-tree indexes          | 683      | 503 (1.36x)    | 468 (1.46x)          | 405 (1.69x)         |
| the Chandrasekhar limit | 718      | 531 (1.35x)    | 488 (1.47x)          | 433 (1.66x)         |

The Latin encodings carry a few more bytes than characters because of the
operators and emoji, which are multi-byte.

### Discursive argument — the tragedy of the commons

Original, 150 tokens:

> Hardin's 1968 essay argued that a shared pasture is inevitably destroyed: each herder gains the full benefit of adding one more animal while bearing only a fraction of the cost of overgrazing, so every rational herder adds animals until the pasture collapses. Hardin concluded that only privatization or state coercion could prevent this. Ostrom's fieldwork overturned the inevitability. Studying irrigation systems, fisheries and alpine pastures that had been managed communally for centuries, she showed that users routinely devise their own rules, monitor each other, and graduate sanctions against violators. The failure case is not shared ownership as such but open access without governance, which is a different arrangement that Hardin had conflated with the commons.

`/compress-text`, 147 tokens, 1.02x:

```
❓ tragedy of the commons
🏛 Hardin 1968: shared pasture INEVITABLY destroyed
   ⇐ herder gains FULL benefit of +1 animal ∧ bears FRACTION of overgrazing cost
   ⇒ rational herder adds animals until collapse
   ⇒ remedy: privatization ∨ state coercion ONLY
🔬 Ostrom fieldwork ⇒ overturns the INEVITABILITY
   irrigation | fisheries | alpine pastures, communally managed for centuries
   ⇒ users devise own rules ∧ monitor each other ∧ graduate sanctions on violators
⚠ failure case = open access without governance ∖ shared ownership
   ⇒ different arrangement, conflated by Hardin with the commons
```

The operators carry the two relations that are the paragraph's actual content.
`∖` separates open access from shared ownership, which is the whole point of
the correction; without it a reader can take Ostrom to be denying Hardin's
mechanism rather than denying its inevitability. `⇐` marks the herder's
incentive as the reason for the collapse rather than a further consequence of
it. The names do heavy work here: "Hardin 1968" and "Ostrom fieldwork" each
stand for a body of argument a reader can retrieve.

`/ultra-compress-text`, 91 tokens, 1.65x:

```
❓ tragedy commons 🏛 Hardin 1968 shared pasture inevitably destroyed each herder full benefit one
more animal fraction cost overgrazing rational adds until collapse only privatization state coercion
prevent 🔬 Ostrom fieldwork overturned inevitability irrigation fisheries alpine pastures communal
centuries users devise own rules monitor each other graduate sanctions violators ⚠ failure not
shared ownership but open access without governance different arrangement Hardin conflated commons
```

`/hanzi-compress-text`, 151 tokens, 0.99x on tokens, 5.30x on characters:

```
❓公地悲剧
🏛Hardin1968：共牧场必毁
　⇐牧人独得＋1畜全利∧仅担过牧成本一分
　⇒理性牧人增畜⇒崩溃
　⇒药方：唯私有∨国家强制
🔬Ostrom田野⇒推翻其必然
　灌溉｜渔｜高山牧场，共管数百年
　⇒民自订规∧互监∧对违者递级罚
⚠败因＝无治理之开放取用∖共有产权
　⇒二者异，Hardin混为一谈
```

### Technical explanation — B-tree indexes

Original, 136 tokens:

> A database index turns a full table scan into a lookup. B-trees remain the default because they match the hardware: each node is sized to one disk page, so a tree holding a billion rows is only three or four levels deep, and a lookup costs three or four page reads rather than a million. Nodes stay between half full and full, splitting when they overflow and merging when they underflow, which keeps the tree balanced without periodic rebuilds. The leaves are linked in key order, so a range query walks sideways instead of descending repeatedly. The cost is write amplification: every insert may split nodes up the path, so heavily written tables pay for the reads they accelerate.

`/compress-text`, 127 tokens, 1.07x:

```
❓ database index: full table scan → lookup
📊 B-tree default ⇐ matches hardware
   node = 1 disk page ⇒ 10⁹ rows ≈ 3–4 levels ⇒ lookup ≈ 3–4 page reads ∖ a million
   nodes between half full ∧ full: split on overflow | merge on underflow
      ⇒ balanced, ∄ periodic rebuild
   leaves linked in key order ⇒ range query walks SIDEWAYS ∖ re-descending
⚠ cost = write amplification: insert may split nodes up the path
   ⇒ write-heavy tables pay for the reads they accelerate
```

Note what is not compressed. Every quantity survives intact, because a number
is already at its shortest expressible form. What goes is the connective
prose, and the causal chain from page-sized nodes through shallow depth to
cheap lookups becomes a run of `⇒` rather than three subordinate clauses.
`∖` marks the contrast the paragraph turns on, three page reads against a
million.

`/ultra-compress-text`, 84 tokens, 1.62x:

```
❓ database index full table scan to lookup 📊 B-tree default matches hardware node sized one disk
page billion rows three four levels deep lookup three four page reads not million nodes between half
full and full split overflow merge underflow balanced without periodic rebuilds leaves linked key
order range query walks sideways not descending repeatedly ⚠ cost write amplification insert may
split nodes up path heavily written tables pay for reads they accelerate
```

`/hanzi-compress-text`, 144 tokens, 0.94x on tokens, 4.91x on characters:

```
❓B树索引
索引：全表扫描⇒定点查找
🔧仍为默认⇐合硬件：节点＝一磁盘页
　⇒十亿行之树仅3~4层⇒查找＝3~4次页读∖百万
⚖节点半满至全满：溢则裂，欠则并⇒自平衡，无须定期重建
➡叶按键序相连⇒范围查询横走∖反复下降
💸代价＝写放大：每插入或沿路径向上裂
　⇒重写之表，为其所加速之读付费
```

### Quantitative science — the Chandrasekhar limit

Original, 142 tokens:

> A white dwarf is supported not by heat but by electron degeneracy pressure, a consequence of the Pauli exclusion principle. Chandrasekhar showed in 1930 that this support fails above about 1.44 solar masses, because as the star is compressed the electrons become relativistic and the pressure grows more slowly with density than gravity demands. Above that limit collapse cannot be halted at white dwarf densities, which is why the observed white dwarf masses cluster below it and why type Ia supernovae, produced when an accreting white dwarf approaches it, have a characteristic peak brightness. That uniformity is what makes them standard candles, and it is the basis of the 1998 measurement of cosmic acceleration.

`/compress-text`, 124 tokens, 1.15x:

```
❓ white dwarf: supported by electron degeneracy pressure ∖ heat ⇐ Pauli exclusion
📐 Chandrasekhar 1930: support FAILS above ≈1.44 M☉
   ⇐ compression ⇒ electrons relativistic ⇒ pressure grows with density SLOWER than gravity demands
   ⇒ above limit: collapse ∄ halt at white dwarf densities
🧪 ⇒ observed white dwarf masses cluster BELOW it
   ⇒ type Ia SNe (accreting WD → limit) have characteristic peak brightness
   ⇒ uniformity ⇒ standard candles ⇒ basis of 1998 cosmic acceleration measurement
```

This paragraph is a single causal chain ending in an observational payoff, so
almost all of it renders as nested `⇒`. `∖` again carries a correction the
reader would otherwise get wrong, that the support is not thermal. The
subject matter owns some notation of its own here, so `M☉` is kept as the
field's own symbol rather than replaced.

`/ultra-compress-text`, 91 tokens, 1.56x:

```
❓ white dwarf supported electron degeneracy pressure not heat Pauli exclusion 📐 Chandrasekhar 1930
support fails above 1.44 solar masses compressed electrons relativistic pressure grows more slowly
with density than gravity demands above limit collapse cannot halt white dwarf densities 🧪 observed
white dwarf masses cluster below type Ia supernovae accreting white dwarf approaches characteristic
peak brightness uniformity standard candles basis 1998 measurement cosmic acceleration
```

`/hanzi-compress-text`, 164 tokens, 0.87x on tokens, 4.13x on characters:

```
❓Chandrasekhar极限
白矮星撑于电子简并压∖热⇐Pauli不相容
📉Chandrasekhar1930：撑失效于≳1.44太阳质量
　⇐压缩⇒电子相对论化⇒压强随密度之增≺引力所需
⇒超此，坍缩不止于白矮星密度
　⇒①实测白矮星质量皆聚其下
　　②Ia超新星（吸积白矮星趋近该极限时生）有特征峰值亮度
✨此一致性⇒标准烛光⇒1998宇宙加速之测量所本
```

### What these ratios show

At paragraph scale the hybrid barely compresses anything, in any domain, and a
careless encoding of it will come out longer than the source. Layout costs the
same whether it organizes six lines or sixty, and one paragraph has few
function words to delete relative to its content. The hybrid earns its keep
over documents, where the ratios in the domain table above apply; the ultra
form is what pays at this scale.

The domain spread visible at document scale does not appear here — the three
ultra ratios sit within a tenth of each other, and the small differences are
authoring noise rather than a property of the topics. Redundancy and
retrievable background need a document's worth of text before they separate.

The Han-script column behaves differently on each axis, which is the clearest
demonstration in this file of why the axis has to be named. In tokens it is at
or below parity, the worst of the three; in characters it wins by a factor of
four to five, far more than either Latin method manages; in bytes it wins by
less than two. Same three encodings, three different verdicts. At paragraph
length there is no removable prose to change any of this, so what remains is
the arithmetic of the script itself: about a quarter the characters, three
bytes each, and roughly one token per character.

The ultra form pays by discarding exactly the marks that made the hybrid
unambiguous. In the first example its reader must work out unaided that
"not shared ownership but open access" is a correction of Hardin rather than a
rejection of Ostrom, and in the third that the closing clauses are a causal
chain rather than a list.

## Verifying an encoding

Claims about fidelity require a measurement, not an impression.

1. Write a rubric over the source: the atomic claims, each with the alternate
   surface forms that count as recovering it. Confirm the rubric scores the
   source itself at 100 percent before using it — a rubric that fails on its
   own source is measuring its own defects.
2. Decode with a fresh reader that has no access to the source and no
   knowledge of the encoding scheme, using a fixed instruction held constant
   across every comparison. Changing the instruction ends the comparison.
3. Score the decode against the rubric.
4. **Run at least three decodes and report the mean and the spread.** Single
   runs at this length vary by about two points, so any ordering inside a
   two-point band is noise and must not be reported as a result. A finding that
   rests on one run is not a finding.
5. Never write the encoding against the rubric's patterns. Encode from the
   source by judgment; the rubric is the examiner, not the syllabus.
