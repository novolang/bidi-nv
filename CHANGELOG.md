# Changelog

All notable changes to bidi-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `bidipara` — the resolved paragraph, and the decision the whole
  package follows from: `resolve` answers LEVELS, not a display
  string. A function that answered only a string cannot do L1, which
  is defined over the characters' ORIGINAL classes and would have
  thrown them away; cannot do L2 per line, because the levels are
  resolved on the paragraph and the line breaks are chosen afterwards
  by something measuring glyphs; and cannot give a caret position
  back, which is a question about the mapping and not about the
  string. So `BidiParagraph` carries the text, the paragraph level, a
  level per character and the original class per character, and every
  later question is a function of that one value.
- `bidipara.resolve` REFUSES text holding a paragraph separator that
  is not its last character. P1 splits a document into paragraphs
  before any other rule runs, and the reference implementations
  resolve the first paragraph and apply its level to the rest. The
  symptom is a second paragraph that reads backwards, in a document
  the author never checked. `split_paragraphs` is P1 as its own call.
- `BidiBase` has THREE arms. `BidiBaseAuto` is P2 and P3 — the first
  strong character decides, skipping everything between an isolate
  initiator and its matching PDI. An `auto` that quietly meant LTR is
  the bug every naive implementation ships, and it is invisible until
  somebody pastes Arabic into a field whose direction was supposed to
  follow the text. `base_direction_of` is P2 and P3 alone, for the
  caller that has to set a `dir` attribute and is not drawing yet.
  The ANSWER is two-valued, `bidilevel.BidiDirection`, and the
  asymmetry between the question and the answer is deliberate.
- `bidireorder` — every function takes a resolved paragraph and a
  range within it, and none of them takes a `Str`. Resolution is per
  paragraph and reordering is per line, and a caller that resolved
  each visual line on its own would get a different paragraph level
  for any line beginning with a digit. `line_levels` applies L1,
  which is why `BidiParagraph` keeps the original classes; `reorder_line`
  answers a permutation of character indices, which is exactly the
  form `BidiCharacterTest.txt`'s reordering column is written in;
  `logical_to_visual` is its inverse, because a text field needs both
  and inverting a permutation is three lines that are wrong the first
  time. `reordered_text` is the only function in the package that
  builds a string.
- `bidiclass` — the Bidi_Class property, all twenty-three values, and
  the table. The DEFAULT for an unassigned code point is published as
  `default_class`, because it is not L: `DerivedBidiClass.txt` gives
  ranges that default to R, to AL, to ET and to BN, and a table built
  only from the assigned characters answers L for a code point added
  after its Unicode version — putting a left-to-right island in the
  middle of a Hebrew paragraph and moving the text around it.
  `is_assigned` tells an answer from the table apart from an answer
  from a default.
- `bidirun` — level runs (BD7) and isolating run sequences (BD13) as
  two different things, because W1 to W7 and N0 to N2 are defined
  over the second. Before Unicode 6.3 the two coincided; an
  implementation written as if they still did is wrong on any text
  using FSI, which is what a templating system emits around
  interpolated content. Each sequence carries its own `sos` and `eos`,
  because W1 and N1 read them and a sequence without its edges cannot
  be resolved. `bracket_pairs` is BD16 and `MAX_PAIRING_DEPTH` is its
  63-entry stack, published because a caller that meets the limit sees
  parentheses that stopped matching half way through a line and has no
  other way to find out why.
- `bidilevel` — levels are `Int`, not a wrapper. L2 is "from the
  highest level down to the lowest odd level, reverse any contiguous
  sequence at that level or higher", and "or higher" is a comparison
  between numbers that a wrapper would have to be unwrapped for at
  every site. What is published instead is the arithmetic, named, and
  the TWO limits: `MAX_DEPTH` is 125, the deepest explicit embedding,
  and `MAX_LEVEL` is 126, the deepest a character can resolve to,
  because I1 adds two at an even level. A caller that sized a table by
  the first is one short.
- `bidimirror` — L4 answers WHICH character, never a rewritten string.
  A copy of the text in which `(` has silently become `)` is wrong to
  search, wrong to copy back into a document and wrong to hand to a
  shaper whose font mirrors the glyph itself. `mirror_at` applies L4's
  odd-level condition; `is_mirrored` and `paired_bracket` are kept
  apart because Bidi_Mirrored (about 360 characters) and
  Bidi_Paired_Bracket (60 pairs) are different properties read by
  different rules.
- Overflow is a REPORT, not a refusal. X1 to X8 define what happens
  past `MAX_DEPTH`: the embedding or isolate is counted and ignored
  and the text is still resolved. `overflow_count` and
  `isolate_overflow_count` are fields of the answer, counted apart
  because X6a pops them apart. A package that refused the document
  would refuse one every browser renders.
- `bidierror` — four refusals, with `code` stable across releases and
  `is_text_fault` separating the one that says the text was never one
  paragraph from the three that are about a line range or a level the
  caller chose. `BidiError` implements `Error`, which SPEC § 3.4
  requires of a `Result`'s error type; no signature changed.

### Decided

- **unicode-nv is REFUSED as a dependency, and bidi-nv carries its own
  tables.** unicode-nv 0.0.1 publishes `uclass.UniCategory` — the
  GENERAL category — beside display width, grapheme and word breaks,
  normalisation and case mapping. It publishes no Bidi_Class, no
  Bidi_Mirrored, no Bidi_Mirroring_Glyph and no Bidi_Paired_Bracket,
  and its own "What is not included" names UAX #9. So there was
  nothing to depend on.
- And General_Category could not have been derived from even if it
  were free. Hebrew alef and Arabic alef are both `Lo`, one R and one
  AL, and the difference decides W2 and W3. A European digit and an
  Arabic-Indic digit are both `Nd`, one EN and one AN, and the
  difference decides I1. All nine explicit formatting characters and
  the zero-width joiner are `Cf`; the nine have nine distinct
  Bidi_Classes and the joiner is BN.
- The three tables come to about 9 KB — 6.0 KB of Bidi_Class, 2.2 KB
  of mirroring pairs, 0.9 KB of brackets — so they are compiled in
  with no handle to pass and no tier to choose. unicode-nv tiers
  because normalisation alone is 190 KB; 9 KB fits on the devices this
  package is `core` for. The three sizes are published as constants so
  that the number changes visibly when the tables are regenerated.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suites reaches `not implemented:
  bidi-nv.<module>.<fn>`.
- **The tests need a toolchain newer than 0.9.0.** Their conformance
  readers pass `d => str.to_int(d) ?? 0 - 1` to `list.map`, and a `??`
  inside a mapped lambda segfaults under 0.9.0's LLVM back end; the fix
  is on `main` after 0.9.0. The named function that worked around it is
  gone, so the readers read as one expression each. `src/` builds on
  0.9.0 unchanged.
- **The conformance files are not vendored.** `BidiTest.txt` and
  `BidiCharacterTest.txt` are some five megabytes together. Six lines
  of the second and one block of the first are written out in the
  suites in the upstream formats, beside the two readers that parse
  them, because the implementation lane will want exactly those
  readers to drive the rest of the files.
- **A code point cannot be turned back into a string.**
  `BidiCharacterTest.txt` states its vectors as hexadecimal code
  points, and `str.from_char` narrows its argument to one byte and
  answers mojibake for everything above U+007F, silently; nothing else
  in the standard library encodes a code point. Filed as
  stdlib/str-from-char-truncates-a-non-ascii-code-point. The suites
  work around it by parsing the vectors to `[Int]` for the
  class-level assertions and writing the text a vector denotes as a
  literal, and the site names the filing.
- **L3 is not in the surface.** Reordering a combining mark to stay
  with its base character at a right-to-left level is a rule about
  glyphs, and this package never sees one. The mark carries the level
  of its base, which is what a renderer needs in order to perform L3
  itself.
