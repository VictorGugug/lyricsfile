# AMLL TTML Tool Integration (ForkZar)

This file documents how the AMLL TTML Tool reads and writes `.lyricsfile.yaml` : version header handling, metadata mapping, the fixed vocalist id convention, sections, syllables vs segments, plain variants, and round-trip guarantees.

**Status of this work:** this is a practical, working implementation of proposed extensions to the Lyricsfile format. The `version: '1.1'` signal is an opt-in convention used by this tool to identify files that use extension fields. It is not a ratified Lyricsfile standard. A conformant `version: '1.0'` reader should reject files with an unknown version per spec §8, which is intentional: the extensions have not yet been merged into the spec. The open question of whether `version: '1.1'` is the right signaling approach, or whether all extensions should remain in `x_amll_tool` while keeping `version: '1.0'`, is part of the upstream discussion this fork intends to start.

## Compatibility with the Lyricsfile project

- Files written by the tool carry `lyricsfile: '1.1'` / `version: '1.1'`. A conformant 1.0 reader will reject files with an unknown version per spec §8; that is expected. A 1.0 reader that chooses to be lenient can still parse the primary fields, since all extension fields are optional.
- Files without the `x_amll_tool` block or without any 1.1 fields import cleanly as 1.0, and `1.0`/`1.1` are both accepted on read.
- Duet/harmony/background/third-voice lines, sections, transliteration/translation, ruby `segments`, `syllables`, `plain_transliteration`/`plain_translation`, `duration_ms`/`offset_ms`/`instrumental`, and `reversed_sync_lines` all survive a round-trip (export -> parse -> export is byte-identical, verified by 13 vitest cases).
- Where Lyricsfile 1.0 cannot represent something (multiple artists, custom metadata keys, extra `artists` values), the data is kept in `x_amll_tool.extra_metadata` instead of being lost.
- What was not changed: the format is still YAML with UTF-8, a single document, `version` at top level, timestamps as integers in milliseconds, the basic `lines` and `words` model, and `offset_ms` is preserved without reinterpretation.

## Extended format used by the tool (1.1)

The tool writes Lyricsfile 1.1 documents with format headers and one namespaced extension block:

```yaml
lyricsfile: '1.1'
version: '1.1'
metadata:
  title: Example
  artist: Example
  duration_ms: 243000
  offset_ms: 0
  instrumental: false
  vocalists:
    - id: v1
      name: Lead
      type: person
plain: |
  Hello world
plain_transliteration: |
  hello world
x_amll_tool:
  created_by_discord: some_user
  reversed_sync_lines: [1, 4]
  extra_metadata:
    songwriter:
      - Writer One
    artists:
      - Second Artist
lines:
  - text: Hello world
    start_ms: 0
    end_ms: 1000
    vocalist: [v1]
    translation: Hola mundo
    transliteration: hello world
    words:
      - text: 'Hello '
        start_ms: 0
        end_ms: 500
        transliteration: hello
      - text: world
        start_ms: 500
        end_ms: 1000
        trailing_separator: ' '
        segments:
          - text: 今
            start_ms: 0
            end_ms: 250
            transliteration: kyo
        syllables:
          - text: un
            start_ms: 0
            end_ms: 100
```

## Vocalist conventions

The TTML model has four line flavors; they map to **canonical IDs emitted by AMLL** (not a proposed universal registry) on write:

- plain / lead -> `v1`
- `isDuet` -> `v2`
- `isMiddle` -> `v3`
- `isDuetGroup` (harmony, two voices on the same line) -> `v4`

If the line is also `isBG` (background), a `-bg` suffix is appended (`v1-bg`, `v2-bg`, ...) and `role: background` is set. On read, `line.vocalist` accepts both a single string and an array of strings; foreign files with arbitrary ids like `ada`/`rio` are read and normalized positionally (first distinct id -> `v1`, second -> `v2`, etc.) so the AMLL internal model always works with a fixed set. Display names are mapped positionally to `v1`/`v2` on the next export.

Each `metadata.vocalists[]` entry may carry a real singer name (`Ariana` etc.) instead of the generic `Lead`/`Duet`/`Middle`/`Harmony` defaults. The writer always emits `type: person` for individual singer ids (`v1`, `v2`, `v3`); `v4` (two voices simultaneously) may be emitted as `type: group` when the vocalist name reflects a combined entity. On read, any `type` value is accepted and preserved.

## Sections, Transliteration, Segments, Syllables, Plain variants

- **Sections:** TTML sections map to `sections[]` with `kind` (enum `intro`, `verse`, `pre-chorus`, `chorus`, `post-chorus`, `bridge`, `refrain`, `instrumental`, `outro`, `other`; unknown -> `other`), optional `label`, `start_ms`, `end_ms` computed from the lines that belong to the section.
- **Transliteration / Translation:** `line.transliteration` <-> `romanLyric`, `line.translation` <-> `translatedLyric`; `word.transliteration` <-> `romanWord`, `word.translation` <-> `translation`; segment and syllable entries may also carry `transliteration`/`translation`.
- **Segments vs Syllables:** `words[].segments[]` is ruby/CJK (`今` -> `kyo`) with optional `transliteration`/`translation`; `words[].syllables[]` is sub-word musical timing (`un`/`til` inside `until`) as proposed in #4. The parser accepts both keys; the writer emits `segments` for ruby and `syllables` for timing so the two concerns stay distinct.
- **Trailing separator:** `words[].trailing_separator` is an explicit separator alternative to the spec's trailing-space-in-`text` (per issue #1). `text: "Hello"` + `trailing_separator: " "` keeps `start_ms`/`end_ms` on the sung characters for correct highlight interpolation; files that use the classic `"Hello "` trailing space continue to work and are preserved.
- **Plain variants:** `plain` is preserved verbatim if the TTML model carries it (so `[Verse]` headers survive); otherwise it is generated as `lines.map(text).join("\n")`. `plain_transliteration` and `plain_translation` are top-level strings parallel to `plain` (per #8).
- **Metadata extras and the x_amll_tool split:** two categories of non-standard data are handled differently:
  - *Format-level extension fields* -- `vocalists`, `sections`, `plain_transliteration`, `plain_translation`, `trailing_separator`, `syllables` -- are written at the standard document level and tagged `version: '1.1'`, because they are candidates for future spec inclusion.
  - *AMLL-private fields* -- `reversed_sync_lines`, `created_by_discord`, and arbitrary metadata keys (`songwriter`, `composer`, extra `artists`) -- go into `x_amll_tool` regardless of version, because they are implementation-specific and should not pollute the standard namespace. Other implementations may use a different `x_<namespace>` block following the same pattern.
  - `duration_ms`, `offset_ms`, `instrumental` are read/written per spec §3.

## Changelog

### Added (2026-09-04) : full community alignment, CC0 demo, real-world samples & parser fixes
- **CC0 Demo (`examples/complete-rich-demo.lyricsfile.yaml` & `examples/complete-rich-demo.ttml`):** 100% original, copyright-free fictional demonstration track ("Starlight Horizons" by Nova & Orion) specifically aligned with `CONTRIBUTING.md` requirements for submission to `tranxuanthang/lyricsfile`. Demonstrates every feature proposed across GitHub issues #1 to #9:
  - Song structure `sections[]` (`intro`, `verse`, `chorus`) with `kind`, `label`, `start_ms`, `end_ms`.
  - Vocalist definitions (`v1` Lead Nova, `v2` Duet Orion, `v4` Group harmony).
  - Background vocal role (`v1-bg`, `role: background`) with automatic parenthesis formatting.
  - Sub-word musical timing with `words[].syllables[]` (e.g. `for-ev-er`).
  - Ruby pronunciation annotations with `words[].segments[]` (Kanji `星` with `transliteration: hoshi`).
  - `trailing_separator: " "` keeping character timestamps precise for karaoke highlight interpolation without embedded spaces.
  - Full `translation` and `transliteration` across lines, words, and top-level block scalars (`plain`, `plain_transliteration`, `plain_translation`).
  - Namespaced vendor extension `x_amll_tool` (`created_by_discord`, `reversed_sync_lines`, `extra_metadata`).
- **Real-World Testbed 1 (`testbeds/White ball.lyricsfile.yaml` & `testbeds/White ball.ttml`):** Real-world English duet with background vocals and multi-songwriter metadata in `x_amll_tool.extra_metadata`. Fixed line 19 timing anomaly where `start_ms` was `0` and `panicked` word was `0/0` (restored to `87087`-`88340` ms).
- **Real-World Testbed 2 (`testbeds/Yoru ni Kakeru.lyricsfile.yaml` & `testbeds/Yoru ni Kakeru.ttml`):** Real-world Japanese word-by-word synced track sourced from AMLL TTML DB (`夜に駆ける` by YOASOBI). Exercises 56 lines with complex kanji/kana timing, romaji transliteration, and Chinese translation.
- **Parser Fixes in `NaeNae-AMLL-TTML-TOOL`:**
  - `ttml-parser.ts`: Fixed DOM namespace resolution for `<meta>` and `<agent>` elements when parent nodes lack default namespace prefixes.
  - `ttml-parser.ts`: Fixed background span detection (`ttm:role="x-bg"`) to use `localName(el).toLowerCase() === "span"` and `getAttr(el, "role")`, ensuring seamless parsing across XHTML/HTML/XML DOM engines (like happy-dom where XHTML element tag names are uppercase `SPAN`).

### Fixed (2026-09-09) : vocalist type consistency
- **Writer:** `v4` (two voices simultaneously) is now emitted with `type: group` instead of `type: person`, matching the semantic intent described in `OPEN_QUESTIONS.md` and aligning `complete-rich-demo.lyricsfile.yaml` with the actual tool output. Individual singer ids (`v1`, `v2`, `v3`) continue to use `type: person`. On read, any `type` value is accepted and preserved.
- **Tests:** Added assertion that `v4` serializes as `type: group` and `v1` as `type: person` in the round-trip test for real vocalist names.

### Added (2026-08-26) : make drafts fully truthful
- **Parser:** Accept `vocalist` as a single string or array; accept `syllables` alias alongside `segments`; preserve `plain`, `plain_transliteration`, `plain_translation`, `duration_ms`, `offset_ms`, `instrumental`; support `word.trailing_separator` and `word.translation`/`segment.translation`; and map foreign vocalist IDs positionally while preserving names.
- **Writer:** Emit `duration_ms`/`offset_ms`/`instrumental`, `plain_transliteration`/`plain_translation`, `word.trailing_separator`, `word.translation`, `segment.transliteration`/`translation`, and distinct `syllables` vs `segments`; handle `instrumental: true` (empty `lines`, `plain: ""`).
- **Types:** Extended `LyricsfileMetadata` with `duration_ms`/`offset_ms`/`instrumental`; `LyricsfileDocument` with `plain_transliteration`/`plain_translation`; `LyricsfileWord` with `translation`/`trailing_separator`/`syllables`; `LyricsfileSegment` with `transliteration`/`translation`; `LyricsfileLine.vocalist` as `string | string[]`; `TTMLLyric` with `plain`/`plainTransliteration`/`plainTranslation`/`durationMs`/`offsetMs`/`instrumental` and `LyricWord` with `translation`/`trailingSeparator`/`segments`.
- **Tests:** 13 vitest cases covering vocalist string/array, syllables alias, plain_transliteration, duration/instrumental, trailing_separator, and segment translation.
- **Docs:** Documented exact specifications and a clear distinction between the CC0 reference showcase and real-world conversion testbeds.

### Added (initial)
- **testbeds/White ball.lyricsfile.yaml** and **testbeds/White ball.ttml**: initial conversion sample with duet, background, and multi-songwriter lines.
- Initial format integration documentation covering `v1`-`v4` + `-bg` vocalist conventions and `x_amll_tool` extension blocks.
