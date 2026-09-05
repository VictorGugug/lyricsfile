# AMLL TTML Tool Integration (ForkZar)

This directory contains the integration work, documentation, and reference files for supporting the Lyricsfile format (version 1.0 and 1.1 superset) within the AMLL TTML Tool.

## Directory Structure

- examples/ : Candidate demonstration files aligned with CONTRIBUTING.md requirements (100% CC0 1.0 Universal, fictional and copyright-free).
  - complete-rich-demo.lyricsfile.yaml : Demonstration track ("Starlight Horizons" by Nova ft. Orion) exercising sections, multi-vocalists (v1, v2, v4), background vocals (v1-bg), sub-word syllable timing, ruby segments, and trailing separators.
  - complete-rich-demo.ttml : Matching Apple Music-like TTML source project.

- testbeds/ : Real-world converted songs used exclusively for verifying bidirectional parser and writer fidelity. These files are kept in this fork repository for testing purposes and are excluded from upstream pull requests to respect licensing constraints.
  - White ball.lyricsfile.yaml and White ball.ttml : English duet with background vocals and multi-songwriter metadata.
  - Yoru ni Kakeru.lyricsfile.yaml and Yoru ni Kakeru.ttml : Japanese word-by-word synced track with Kanji/Kana timing, Romaji transliteration, and Chinese translation.

- CHANGELOG.md : Detailed technical specification and integration documentation covering version headers, vocalist ID conventions (v1-v4), syllable versus segment distinctions, and the namespaced x_amll_tool extension block.