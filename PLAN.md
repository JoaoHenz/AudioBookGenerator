# AudioBookGenerator — design plan

## Goal

Build the **generator** for an Audible-style listening experience. The listening UI itself is a separate project and is out of scope here; this repo's job ends when the bundle is written to disk.

- **Input**: a single book file in one of `.epub`, `.mobi`, or `.pdf`.
- **Generate**: a bundle containing the audio narration, the original text, chapter boundaries, and a word/sentence-level timing map binding the two.
- **Consumed by** *(separate project)*: a player UI that streams the audio, lets the user jump by chapter, and highlights the spoken text in real time — Audible's "Immersion Reading", but standalone.

## Format research — what Audible actually uses

Audible's stack is closed, but the pieces are well-documented externally:

- **AAX** — DRM-wrapped MPEG-4/AAC audio container. Carries audio, embedded chapter markers, bookmarks, and cover art. Standard download format on macOS/Windows. Tiered bitrate: AAX (~32–64 kbps), AAX+ (~128 kbps), AAX++ (~256 kbps).
- **AAXC** — successor introduced ~2019, now default on mobile (and increasingly on desktop). Stronger DRM, designed around streaming. Public DRM tools cannot decrypt new AAXC files.
- **Whispersync for Voice** — Audible's proprietary sync protocol. Pairs a Kindle ebook with its corresponding audiobook and keeps a single "current position" across both, so you can switch devices/modes without losing your spot.
- **Immersion Reading / "Read & Listen"** — the in-app feature where the narrator's words highlight on screen in real time. Sits on top of Whispersync; the actual text-to-audio mapping is delivered server-side through the Audible/Kindle SDKs and is not a public format.

**Takeaway**: the audio container (AAX/AAXC) is essentially MP4-with-DRM-and-chapters. The word-level highlighting magic is *not* in the audio file — it lives in a separate sync structure that Audible keeps private. We need an open equivalent for that part.

## Open standards we can build on

- **EPUB 3 Media Overlays** ([W3C spec](https://www.w3.org/publishing/epub32/epub-mediaoverlays.html)) — the de facto open format for "synced audio + text" books. Defines a SMIL 3 subset where each `<par>` element binds:
  - an `<audio src="..." clipBegin="..." clipEnd="..."/>` clip, and
  - a `<text src="content.xhtml#fragment-id"/>` reference into an XHTML content document.
  Granularity is typically sentence/phrase, but word-level is allowed (just more `<par>` elements + more `id`s in the XHTML). Designed to gracefully degrade in readers that don't support it.
- **DAISY 3 / ANSI/NISO Z39.86** — the older accessibility standard that EPUB 3 Media Overlays evolved out of. Same XML + SMIL + audio idea. Worth knowing about for compat with screen-reader tooling, not as our primary format.
- **M4B** — MP4 audio container with native chapter markers and metadata. No DRM, no text sync. Good baseline for "audio + chapters" portability.
- **MP3 + ID3 chapter frames** (CHAP/CTOC) — works, but the chapter UX is rougher and there's no place for synced text.

**Decision (proposed)**: target **EPUB 3 with Media Overlays** as the canonical bundle format. It's the only open standard that natively expresses everything we need (text + audio + chapters + word-level sync) and is supported by accessible readers (Thorium, Colibrio, etc.) out of the box. A simpler "audio-only" export to **M4B with chapters** stays as a fallback for users who just want a portable audio file.

## Forced alignment — getting the timings

This is the core technical problem: given a text and an audio recording (or TTS output), produce a timestamp for every word/sentence so the player can highlight in real time.

Tools to evaluate, ranked by current accuracy (per [Rousso et al., Interspeech 2024](https://www.isca-archive.org/interspeech_2024/rousso24_interspeech.pdf)):

1. **Montreal Forced Aligner (MFA)** — Kaldi-based GMM-HMM. Most accurate word-level timestamps in the comparison. Requires per-language acoustic models. Heavier setup.
2. **WhisperX** — Whisper transcription + wav2vec2 phoneme alignment. Multilingual, easy to install (`pip`), GPU-friendly. Less accurate than MFA but still good enough for read-along; some reports of word-boundary drift on long files.
3. **OpenAI Whisper with `word_timestamps=True`** — built-in, simplest path. Quality is decent; recent research shows attention-based alignment can match wav2vec.
4. **aeneas** — Python, DTW-based, sentence-level only. Mature, lightweight, no GPU needed — good for v0 / EPUB-style sentence highlighting.

**Decision (proposed)**: start with **aeneas** for sentence-level alignment in v0 (cheap, deterministic, no model downloads), then layer on **WhisperX** for word-level highlighting once the pipeline is stable. Keep MFA as a "high-fidelity" optional path for users who want the best.

## TTS — generating the audio

If users bring their own narration, this is moot. But the repo name says *Generator*, so:

- **Cloud APIs** — OpenAI TTS, ElevenLabs, Google Cloud TTS, Azure. Best voice quality, costs scale with book length, network-dependent.
- **Local OSS** — Coqui TTS / XTTSv2, Kokoro, Piper, Bark. Free, private, hardware-bound. XTTSv2 and Kokoro are currently the strongest quality at reasonable speeds on consumer GPUs.

**Decision (proposed)**: pluggable TTS backend behind a thin interface. Ship a local default (Kokoro or Piper for speed) and one cloud adapter (OpenAI) for users who want top quality.

## Input handling

Each accepted format needs a slightly different front end:

- **EPUB** — easiest. Unzip, parse `content.opf` for the spine, read the XHTML files, use the `nav` document or NCX for chapter boundaries. Libraries: `ebooklib`, or just `zipfile` + `lxml`.
- **MOBI** — Amazon's pre-Kindle-Format-X format. Cleanest path is to convert MOBI → EPUB internally (e.g. via Calibre's `ebook-convert` CLI or the `mobi` Python package), then run the EPUB pipeline. Don't try to handle MOBI internals natively.
- **PDF** — the messy one.
  - Text extraction: PyMuPDF (`fitz`) is fastest and handles most files; pdfplumber is better at column-aware extraction; pdfminer.six is a fallback. Pick one as default and document the limitations.
  - Chapter detection: prefer the PDF's outline / TOC bookmarks if present (`fitz.Document.get_toc()`). Fallback heuristics: largest font sizes on a page, `^Chapter \d+` regex, page-break detection.
  - Scanned PDFs (image-only): require OCR via Tesseract (`pytesseract`). Treat as a v2 feature; v0 can refuse and ask the user to OCR upstream.

Everything downstream of input handling sees a normalized internal representation: ordered chapters, each with title + plain-text or XHTML body.

## Architecture sketch

```
┌──────────────────────────────┐
│  Input: .epub / .mobi / .pdf │
└──────────────┬───────────────┘
               │
   ┌───────────▼────────────┐    ┌──────────────────┐
   │  Format-specific       │───▶│ Chapter splitter │
   │  text extractor        │    │ (TOC / heuristic)│
   └────────────────────────┘    └────────┬─────────┘
                                          │ normalized chapters
                                          ▼
                              ┌────────────────────────┐
                              │  TTS (pluggable)       │
                              └────────────┬───────────┘
                                           │ audio per chapter
                                           ▼
                              ┌────────────────────────┐
                              │  Forced aligner        │
                              │  (aeneas → WhisperX)   │
                              └────────────┬───────────┘
                                           │ SMIL / sentence timing
                                           ▼
                              ┌────────────────────────┐
                              │  Bundler               │
                              │  → EPUB 3 + Overlays   │
                              │  → M4B + JSON sidecar  │
                              └────────────┬───────────┘
                                           │
                                           ▼
                              ┌────────────────────────────┐
                              │  Output bundle (on disk)   │  ◀── tool ends here
                              └────────────┬───────────────┘
                                           │
                                           ▼
                              ┌────────────────────────────┐
                              │  External player (separate │
                              │  project — not this repo)  │
                              │  - audio engine            │
                              │  - chapter nav             │
                              │  - word highlighting       │
                              └────────────────────────────┘
```

## Phased roadmap

**v0 — minimum loop, end-to-end.**
- **EPUB input only** (the cleanest of the three; gets us a working pipeline fastest).
- One TTS backend (whichever has the easiest setup).
- aeneas for sentence-level alignment.
- Output: **M4B + JSON sidecar** with `[{chapter, sentence, start, end, text}]`.
- A throwaway reference player (single HTML file) **only as a test harness** — not a deliverable.

**v1 — full input coverage + canonical bundle.**
- Add **MOBI** input (via internal MOBI→EPUB conversion).
- Add **PDF** input (text-based PDFs only; OCR deferred). TOC-based chapter detection with heuristic fallback.
- Real **EPUB 3 + Media Overlays** output as the canonical bundle shape (M4B+JSON kept as fallback).
- Word-level alignment via WhisperX.

**v2 — polish.**
- OCR for scanned PDFs (Tesseract).
- Cloud TTS adapter (OpenAI), voice picker.
- Optional: MFA backend for high-fidelity alignment.
- Optional: bundle-format spec write-up so third parties can build players against it.

## Open questions

- **MOBI handling**: do we ship Calibre's `ebook-convert` as an external dep, or pull a pure-Python MOBI parser into the project? Calibre is heavier but proven; pure-Python is lighter but flakier on edge-case files.
- **PDF chapter detection**: how aggressive should the fallback heuristics be? A bad split is arguably worse than a single-chapter audiobook. Possible escape hatch: let the user pass a TOC override file.
- **Voice cloning**: in scope or out? XTTSv2 supports it, but it has obvious abuse potential. Default off, behind a flag.
- **Bundle format spec**: do we publish a stable schema for the M4B+JSON shape so other people can build their own players, or treat it as internal-only and tell them to consume the EPUB output instead?
- **Ebook DRM**: explicitly out of scope. We work on text the user already has the rights to.

## Sources

- [EPUB Media Overlays 3.2 (W3C)](https://www.w3.org/publishing/epub32/epub-mediaoverlays.html)
- [Audible AAX format docs](https://docs.fileformat.com/audio/aax/)
- [Audible AAXC overview](https://fileinfo.com/extension/aaxc)
- [Audible Whispersync for Voice](https://www.audible.com/ep/wfs)
- [Audible Read & Listen launch coverage (Feb 2026)](https://9to5mac.com/2026/02/18/amazon-adds-read-and-listen-mode-to-audible-app-for-immersion-reading-feature/)
- [DAISY Format — DAISY Consortium](https://daisy.org/activities/standards/daisy/)
- [Rousso et al., "A Comparison of Modern ASR Methods for Forced Alignment", Interspeech 2024](https://www.isca-archive.org/interspeech_2024/rousso24_interspeech.pdf)
- [WhisperX — GitHub](https://github.com/m-bain/whisperx)
- [aeneas — read-along EPUB how-to](https://www.albertopettarin.it/blog/2014/08/02/how-to-create-epub-3-read-aloud-ebooks.html)
