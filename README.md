# bidi-nv

The Unicode Bidirectional Algorithm decides the order in which text is
drawn when a paragraph mixes left-to-right and right-to-left scripts.
It is specified in
[Unicode Standard Annex #9](https://www.unicode.org/reports/tr9/), and
this package brings it to novo-lang. The reference implementations are
the Rust crate [`unicode-bidi`](https://docs.rs/unicode-bidi), the
Python package [`python-bidi`](https://pypi.org/project/python-bidi/)
and ICU's `ubidi`. The conformance data is the Unicode Character
Database's `BidiTest.txt` and `BidiCharacterTest.txt`.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What the algorithm is

Text is **stored** in the order it is typed and read, which UAX #9
calls logical order. It is **drawn** left to right across a screen,
which UAX #9 calls visual order. For a paragraph in one script the two
orders are the same. For a paragraph that mixes Hebrew or Arabic with
Latin they are not, and the algorithm is the rule that turns one into
the other.

The algorithm works in **embedding levels**. A level is a number from
0 to 126. Its parity is a direction: an even level reads left to
right, an odd level reads right to left. Its magnitude is the nesting
depth. Every character of a paragraph gets a level, and everything
after that is arithmetic on the numbers.

The rules run in five groups, and UAX #9 numbers each rule.

| Group | Rules | What it decides |
| --- | --- | --- |
| Paragraph | P1–P3 | where a paragraph ends, and its base level |
| Explicit | X1–X10 | the levels the embedding and isolate characters open |
| Weak | W1–W7 | what the numbers, separators and marks resolve to |
| Neutral | N0–N2 | what the punctuation and whitespace resolve to |
| Implicit | I1–I2 | the final level of every character |
| Reordering | L1–L4 | the order one line is drawn in, and mirroring |

Two terms from the specification are used throughout this page. A
**level run** (BD7) is a maximal range of characters that share a
level. An **isolating run sequence** (BD13) is a chain of level runs
joined across isolate characters; rules W1 to W7 and N0 to N2 are
defined over these rather than over level runs.

Two of the groups run on different units. **P1 to I2 run on a
paragraph.** **L1 to L4 run on a line.** A paragraph that wraps has
one set of levels and several lines.

## Install

```
novo pkg add bidi-nv
```

## Example

```novo
use bidipara
use bidireorder

fn main() [io]
    // P1: a document is split into paragraphs before anything else.
    for paragraph in bidipara.split_paragraphs("אבג and abc")
        // P2 and P3: the direction follows the first strong character.
        match bidipara.resolve(paragraph, BidiBaseAuto)
            Err(_) => println("that piece is not one paragraph")
            Ok(p)  =>
                println("paragraph level ${p.level}")
                // L1 and L2: one line at a time. Here the line is the
                // whole paragraph; a wrapped paragraph has several.
                match bidireorder.reorder_line(p, 0, bidipara.char_count(p))
                    Err(_) => println("that range is not a line of this paragraph")
                    Ok(order) =>
                        // `order` holds character indices, leftmost first.
                        for i in order
                            println("draw character ${i}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: bidi-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `bidierror` | The four refusals, a stable code for each, and which of them is a fault in the text. |
| `bidiclass` | The Bidi_Class property, its twenty-three values, the table, and the default an unassigned code point takes. |
| `bidilevel` | The embedding level: its parity, the two depth limits, and the arithmetic X2 to X5c are written in. |
| `bidipara` | P1, P2 and P3, and the resolved paragraph that every later question is answered from. |
| `bidirun` | Level runs (BD7), isolating run sequences (BD13), isolate matching (BD9, BD11) and bracket pairs (BD16). |
| `bidireorder` | L1 and L2 for one line: the reset levels, the visual order, and the map a caret needs. |
| `bidimirror` | L4 and the two mirroring properties: Bidi_Mirrored, and Bidi_Paired_Bracket with its type. |

## How to choose an entry point

**`bidipara.resolve` is the call every program makes.** It answers a
`BidiParagraph`, and every other function in the package takes one.

**`bidipara.base_direction_of` answers P2 and P3 alone.** Use it when
a program has to set an HTML `dir` attribute or choose a text field's
alignment and is not drawing anything yet.

**`bidipara.resolve_at_level` sets the paragraph level explicitly.**
Use it when a higher-level protocol has already decided the level and
it is not 0 or 1.

**`bidireorder.reorder_line` answers the drawing order of one line.**
Use it once per line, after the line breaks are chosen.

**`bidireorder.logical_to_visual` answers where one character is
drawn.** Use it to place a caret, and `reorder_line` to paint.

**`bidireorder.reordered_text` answers a display-ordered string.** Use
it in a terminal that draws left to right and in a conformance
harness. It is the only function in the package that builds a string.

**`bidirun.runs` answers the level runs.** Use it to hand a shaper one
range per direction.

**`bidimirror.mirror_at` answers rule L4 for one character.** Use it
when the font does no mirroring of its own.

## The rules a user needs

1. **A document is split into paragraphs first (P1).** A paragraph
   separator terminates a paragraph and belongs to it.
   `bidipara.split_paragraphs` does the split.
   `bidipara.resolve` refuses text holding a separator anywhere but at
   its end.
2. **The paragraph direction is three-valued (P2, P3).**
   `BidiBaseLtr` and `BidiBaseRtl` set the level. `BidiBaseAuto` reads
   the first strong character, skipping everything between an isolate
   initiator and its matching PDI, and takes level 1 for an R or AL
   and level 0 otherwise.
3. **Levels are resolved on a paragraph and reordering happens on a
   line (L1, L2).** A wrapped paragraph is resolved once and reordered
   once per line. Every function in `bidireorder` takes a resolved
   paragraph and a range within it.
4. **The answer is a level per character, not a string.**
   `BidiParagraph.levels` is the list. `bidireorder.reorder_line`
   answers character indices in drawing order.
5. **A level's parity is its direction.** Even is left to right, odd
   is right to left, at every depth.
6. **Isolates are not embeddings.** LRI, RLI, FSI and PDI (Unicode
   6.3) open a level under X5a, X5b, X5c and X6a, and they keep a
   level and a position. LRE, RLE, LRO, RLO and PDF open a level under
   X2 to X5 and X7, and X9 removes them.
7. **X9 removes six classes and no isolate:** LRE, RLE, LRO, RLO, PDF
   and BN. `bidireorder.visible_indices` is the line without them.
8. **The explicit depth limit is 125 (X1–X8).** Past it an embedding
   or isolate is counted and ignored; the text is still resolved. The
   counts are `BidiParagraph.overflow_count` and
   `BidiParagraph.isolate_overflow_count`, and
   `bidipara.had_overflow` reads both. Overflow is not an error.
9. **A character can resolve to level 126.** I1 raises a number at an
   even level by two and I2 raises a letter at an odd level by one, so
   the deepest level is one more than the deepest embedding.
10. **W1 to W7 run over isolating run sequences, not level runs
    (BD13).** Text inside an isolate is a separate sequence, and the
    weak rules do not read across the boundary.
11. **A European number and an Arabic number are different classes.**
    W2 turns an EN after an AL into an AN; W3 then turns the AL into
    an R; W7 turns an EN after an L into an L.
12. **For the neutral rules a number acts as R (N1).** A neutral
    between a Hebrew letter and a European digit resolves to R.
13. **Bracket pairs are matched on the Bidi_Paired_Bracket property
    (BD16, N0), 63 pairs deep.** Past 63 openings in one isolating run
    sequence, bracket processing stops for the rest of that sequence
    and the remaining brackets are resolved by N1 and N2.
    `bidirun.MAX_PAIRING_DEPTH` is the number. The match is by
    canonical equivalence: U+2329 is equivalent to U+3008 and U+232A
    to U+3009, so an angle bracket written either way pairs with one
    written the other. `bidimirror.canonical_bracket` is that rule.
14. **L1 is defined over the original classes, not the resolved
    ones.** `BidiParagraph.classes` holds them.
    `bidireorder.line_levels` applies L1 and depends on where the line
    ends.
15. **Mirroring is L4, and this package does not apply it.**
    `bidimirror.mirror_at` answers which character to draw;
    substituting the glyph is the caller's. A font with an OpenType
    `rtlm` feature does the mirroring itself.
16. **Bidi_Mirrored and Bidi_Paired_Bracket are different
    properties.** About 360 characters are mirrored; 60 pairs are
    paired brackets. A less-than sign is mirrored and is not a
    bracket.
17. **An unassigned code point does not default to L.**
    `DerivedBidiClass.txt` gives ranges whose unassigned code points
    default to R, to AL, to ET or to BN. `bidiclass.default_class` is
    that rule, and `bidiclass.is_assigned` says whether an answer came
    from the table or from a default.
18. **Indices into a paragraph count characters, not bytes.**
    `bidipara.byte_offset_at` converts one to the other.
    `BidiParagraph.text` is indexed in bytes.
19. **L3 is not performed here.** Reordering a combining mark to stay
    with its base character is a rule about glyphs. The mark carries
    the level of its base.

## The tables

The package carries three generated tables and compiles all of them
into any program that calls it.

| Table | Source | Size |
| --- | --- | --- |
| Bidi_Class, with the unassigned defaults | `DerivedBidiClass.txt` | 6.0 KB |
| Bidi_Mirroring_Glyph, 364 pairs | `BidiMirroring.txt` | 2.2 KB |
| Bidi_Paired_Bracket and its type, 60 pairs | `BidiBrackets.txt` | 0.9 KB |

The total is about 9 KB. The sizes are generated from Unicode 16.0.
`bidiclass.CLASS_TABLE_BYTES`, `bidimirror.MIRROR_TABLE_BYTES` and
`bidimirror.BRACKET_TABLE_BYTES` are the same numbers as constants,
and `bidiclass.UNICODE_VERSION` names the version they came from.
There is no tier to choose and no data handle to pass: every lookup
function takes a code point and nothing else.

## Running on a microcontroller

This package makes no device claim and ships no device probe. Every
function in it performs no input or output, so the modules build for a
microcontroller. The 9 KB of tables described above is the whole data
cost, and it is present whether or not a program calls every module.
A resolved paragraph holds one level and one class per character, so
the working memory grows with the length of the paragraph and not with
the length of the document.

## What is not included

- **Shaping.** Choosing the glyphs for a run of Arabic, joining them,
  and applying ligatures is a font engine's work.
  [libharfbuzz-sys](https://novo-lang.org/packages/libharfbuzz-sys) is
  the binding for one.
- **Line breaking.** Deciding where a paragraph wraps needs UAX #14
  and glyph measurements. This package takes the line breaks as
  arguments.
- **Applying L4.** The package answers which characters mirror and to
  what. Substituting the glyph is the caller's.
- **Performing L3.** See rule 19.
- **Normalisation, grapheme clusters, case mapping and the general
  category.** [unicode-nv](https://novo-lang.org/packages/unicode-nv)
  has those. It does not have any property this package reads.
- **A `dir` attribute parser.** `bidipara.base_named` reads the three
  values; extracting them from markup is an HTML parser's work.
- **Reading the Unicode database files.** A `core` package performs no
  input, and the tables are generated ahead of time.
- **The conformance files.** `BidiTest.txt` and
  `BidiCharacterTest.txt` are some five megabytes together. The test
  suites carry a handful of lines from each, in the upstream format,
  and the reader that parses that format.

## Related packages

- [unicode-nv](https://novo-lang.org/packages/unicode-nv) has
  normalisation, grapheme clusters, display width and case mapping.
  Take it as well as this package: normalisation runs before the
  algorithm, and grapheme clusters are what a caret moves by after
  it. It publishes no Bidi_Class, no Bidi_Mirrored and no
  Bidi_Paired_Bracket, and its own page says so.
- [textwrap-nv](https://novo-lang.org/packages/textwrap-nv) chooses
  the line breaks this package's reordering is applied per line of.
- [ansi-nv](https://novo-lang.org/packages/ansi-nv) and
  [tui-nv](https://novo-lang.org/packages/tui-nv) draw into a
  terminal. A terminal draws left to right, so
  `bidireorder.reordered_text` is the form they take.
- [i18n-nv](https://novo-lang.org/packages/i18n-nv) has the locale
  data. A paragraph direction is a property of the text under P2 and
  P3, and of the locale only when a higher-level protocol says so.

## Tests

```bash
novo test tests/bidipara_tests.nv     # P1, P2, P3, and the refusals
novo test tests/bidiclass_tests.nv    # the property table and its defaults
novo test tests/bidireorder_tests.nv  # L1, L2, and the line range checks
novo test tests/bidicover_tests.nv    # levels, runs, sequences, mirroring
```

The normative sources are UAX #9 for the rules and the Unicode
Character Database's `BidiTest.txt` and `BidiCharacterTest.txt` for the
vectors. Six lines of `BidiCharacterTest.txt` and one block of
`BidiTest.txt` are written out in the suites, unaltered, beside the
readers that parse their two formats.

The suites assert that a text holding two paragraphs is refused rather
than resolved, that `BidiBaseAuto` over Hebrew gives level 1 where
`BidiBaseLtr` gives level 0, that an explicit paragraph level above 125
is refused, that Hebrew and Arabic letters resolve to different classes
although they share a general category, that an unassigned code point
in the Hebrew block defaults to R, that X9 removes the five embedding
characters and no isolate, that a segment separator raised by N1 is put
back by L1, that the visual order of three Hebrew letters is the
reverse of their storage order, that a line range outside the paragraph
is refused and an empty one is not, and that a less-than sign is
mirrored and is not a paired bracket.

The tests compile today and fail at run, each on the
`not implemented: bidi-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `bidierror.BidiError` and the other public types | the types are declared |
| `bidierror.message`, `.code`, `.is_text_fault` | no |
| `bidiclass.class_of`, `.classes`, `.default_class`, `.is_assigned` | no |
| `bidiclass.class_name`, `.class_named` | no |
| `bidiclass.is_strong`, `.is_number`, `.is_neutral_or_isolate` | no |
| `bidiclass.is_isolate_initiator`, `.is_explicit_formatting`, `.is_removed_by_x9` | no |
| `bidilevel.level_direction`, `.is_rtl_level`, `.opposite` | no |
| `bidilevel.next_even_above`, `.next_odd_above`, `.level_fits` | no |
| `bidilevel.direction_name`, `.direction_named` | no |
| `bidipara.resolve`, `.resolve_at_level`, `.split_paragraphs` | no |
| `bidipara.base_direction_of`, `.paragraph_level`, `.direction` | no |
| `bidipara.char_count`, `.level_at`, `.class_at`, `.byte_offset_at` | no |
| `bidipara.had_overflow`, `.base_name`, `.base_named` | no |
| `bidirun.runs`, `.run_count`, `.run_at`, `.run_direction` | no |
| `bidirun.sequences`, `.sequence_containing` | no |
| `bidirun.matching_pdi`, `.matching_isolate`, `.bracket_pairs` | no |
| `bidireorder.line_levels`, `.visible_indices`, `.reorder_line` | no |
| `bidireorder.logical_to_visual`, `.is_identity_order`, `.reordered_text` | no |
| `bidimirror.is_mirrored`, `.mirror_of`, `.mirror_at` | no |
| `bidimirror.bracket_type`, `.paired_bracket`, `.canonical_bracket` | no |
| The Bidi_Class, mirroring and bracket tables | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
