---
name: textual-compression
description: Encode a text into a compact interlingua that another reader (human or model) can expand back into full prose, and decode such an encoding back into English. Use this skill whenever the user asks to compress, compact, densify, shrink, or "shorthand" a passage, to write a maximally dense summary intended to be re-expanded later, to decode or expand a compressed or symbolic text, or when they invoke /compress-text, /decompress-text, or /ultra-compress-text.
---

# Textual Compression

An interlingua for lossy-but-recoverable text compression, and the procedures
for writing and reading it. Three commands, one shared model.

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

| source                          | ratio |
|---------------------------------|-------|
| discursive philosophy           | 3.6x  |
| technical explanation (protocol)| 2.1x  |
| narrative history               | 1.3x  |
| pure mathematics                | 1.26x |
| quantitative science            | 1.13x |
| theory with defined terms       | 1.04x |

Ratio falls as the density of irreducible specifics rises. A number, a date, a
proper noun and a defined term are already at minimum description length:
`2.725 K`, `Savitch`, `NP-complete` cannot be shortened without destroying
them. Below about 1.1x the method has stopped compressing and is only
reformatting; say so rather than claiming a compression that did not happen.
**Never quote a compression ratio without naming the domain.**

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

## Worked examples

Three paragraphs of comparable length from different domains, each encoded
both ways. Token counts are from a common tokenizer, used as a proxy.

| paragraph                    | original | /compress-text | /ultra-compress-text |
|------------------------------|----------|----------------|----------------------|
| tragedy of the commons       | 150      | 147 (1.02x)    | 86 (1.74x)           |
| B-tree indexes               | 136      | 125 (1.09x)    | 80 (1.70x)           |
| the Chandrasekhar limit      | 142      | 124 (1.15x)    | 87 (1.63x)           |

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

`/ultra-compress-text`, 86 tokens, 1.74x:

```
❓ tragedy commons 🏛 Hardin 1968 shared pasture inevitably destroyed each herder full benefit one more animal fraction cost overgrazing rational adds until collapse only privatization state coercion prevent 🔬 Ostrom fieldwork overturned inevitability irrigation fisheries alpine pastures communal centuries users devise own rules monitor each other graduate sanctions violators ⚠ failure not shared ownership but open access without governance different arrangement Hardin conflated commons
```

### Technical explanation — B-tree indexes

Original, 136 tokens:

> A database index turns a full table scan into a lookup. B-trees remain the default because they match the hardware: each node is sized to one disk page, so a tree holding a billion rows is only three or four levels deep, and a lookup costs three or four page reads rather than a million. Nodes stay between half full and full, splitting when they overflow and merging when they underflow, which keeps the tree balanced without periodic rebuilds. The leaves are linked in key order, so a range query walks sideways instead of descending repeatedly. The cost is write amplification: every insert may split nodes up the path, so heavily written tables pay for the reads they accelerate.

`/compress-text`, 125 tokens, 1.09x:

```
❓ database index: full table scan → lookup
📊 B-tree default ⇐ matches hardware
   node = 1 disk page ⇒ 10⁹ rows ≈ 3–4 levels ⇒ lookup ≈ 3–4 page reads ∖ a million
   nodes between half full ∧ full: split on overflow | merge on underflow ⇒ balanced, ∄ periodic rebuild
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

`/ultra-compress-text`, 80 tokens, 1.70x:

```
❓ database index full table scan to lookup 📊 B-tree default matches hardware node sized one disk page billion rows three four levels deep lookup three four page reads not million nodes between half full and full split overflow merge underflow balanced without periodic rebuilds leaves linked key order range query walks sideways not descending repeatedly ⚠ cost write amplification insert may split nodes up path heavily written tables pay for reads they accelerate
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

`/ultra-compress-text`, 87 tokens, 1.63x:

```
❓ white dwarf supported electron degeneracy pressure not heat Pauli exclusion 📐 Chandrasekhar 1930 support fails above 1.44 solar masses compressed electrons relativistic pressure grows more slowly with density than gravity demands above limit collapse cannot halt white dwarf densities 🧪 observed white dwarf masses cluster below type Ia supernovae accreting white dwarf approaches characteristic peak brightness uniformity standard candles basis 1998 measurement cosmic acceleration
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
