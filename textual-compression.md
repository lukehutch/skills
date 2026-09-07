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

## Worked example

Original, 113 tokens:

> Herd immunity is often misunderstood as a property of an individual, but it
> is a property of a population. A vaccinated person is protected directly; an
> unvaccinated person in a highly vaccinated population is protected
> indirectly, because the chain of transmission breaks before it reaches them.
> The threshold depends on how contagious the disease is: measles, which is
> extremely transmissible, requires roughly 95 percent coverage, whereas polio
> requires about 80 percent. Falling below the threshold does not produce a
> gradual increase in cases but a sudden return of outbreaks, because
> transmission is exponential.

`/compress-text`, 102 tokens, 1.11x:

```
❓ herd immunity — routinely misread as property of an INDIVIDUAL ∖ of a POPULATION
   vaccinated: protected directly
   unvaccinated ∈ highly vaccinated population: protected INDIRECTLY
      ⇐ chain of transmission breaks before reaching them
📊 threshold ∝ transmissibility: measles (extremely transmissible) ≈95% coverage | polio ≈80%
⚠ coverage < threshold ⇏ gradual rise ⇒ SUDDEN return of outbreaks ⇐ transmission exponential
```

The two relations that are the paragraph's actual claims are the ones carrying
operators: `∖` holds the misconception apart from the correction, and `⇏`
blocks the inference from crossing the threshold to a gradual rise. Everything
else stays in words. Indentation subordinates the two protection cases to the
population claim, and capitals mark the contrasts the source italicizes in
effect.

`/ultra-compress-text`, 59 tokens, 1.92x:

```
❓ herd immunity misread property individual not population vaccinated protected directly
unvaccinated in highly vaccinated population protected indirectly chain transmission breaks
before reaching 📊 threshold depends transmissibility measles extremely transmissible 95
percent coverage polio 80 percent ⚠ below threshold not gradual increase sudden return
outbreaks transmission exponential
```

Note the ratios. On one paragraph the hybrid barely compresses at all, because
layout costs the same whether it organizes six lines or sixty, and a single
paragraph has few function words to delete relative to its content. The hybrid
earns its keep over documents, not paragraphs; the ultra form is the one that
pays at this scale, and it pays by discarding exactly the relational marks that
made the first version unambiguous. A decoder reading the ultra form must
infer for itself that "not population" is a correction rather than a denial,
and that the last clause explains the one before it.

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
