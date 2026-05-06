# AudioBookGenerator

We really live in a society where Aud\*ble gives you 01 audiobook per month and our overlords think this is acceptable. NOT ANYMORE.

## What this is

The **generator** half of an Audible-style listening experience. You give it a book, it produces the files a separate **player / interface** needs to deliver chapter navigation, narrated audio, and live word highlighting on screen (what Audible calls *Immersion Reading* / *Read & Listen*).

This repo builds **only** the generator. The player is a separate concern — see [PLAN.md](./PLAN.md) for the contract a player needs to honor.

## Input

A single book file in one of these formats:

- **`.epub`** — preferred. Open standard, structured XHTML, easy chapter boundaries. Used by Apple Books, Kobo, Google Play Books, public libraries (Libby/OverDrive), and most non-Amazon ereaders.
- **`.mobi`** — Amazon's older Kindle format. Non-DRM MOBI files are supported. (Modern Kindle formats `.azw3` / `.kfx` are DRM-locked and **out of scope** — see below.)
- **`.pdf`** — supported but lower fidelity. Text extraction is heuristic (multi-column layouts, embedded fonts, and scanned-image pages all complicate it) and chapter detection falls back to the document TOC, font-size heuristics, or `Chapter N` regex. Scanned-only PDFs require OCR.

DRM-protected files of any kind are explicitly out of scope. Bring books you legally own, or public-domain works.

## Output

A self-contained bundle that any conforming player can render. The bundle contains four things:

1. **Audio** — narration synthesized by a pluggable TTS backend (or, optionally, audio the user supplies themselves).
2. **Chapter map** — start/end timestamps per chapter so the player can present a chapter list and jump within the audio.
3. **Text** — the source book's text, segmented by chapter, in a reader-friendly form (XHTML for the EPUB output shape).
4. **Timing map** — sentence- and (eventually) word-level alignment between the text and the audio, so the player can highlight the words currently being spoken.

Two output shapes:

- **EPUB 3 + Media Overlays** *(canonical)* — the [open W3C standard](https://www.w3.org/publishing/epub32/epub-mediaoverlays.html) for synced text + audio. Carries everything above in a single `.epub` file. Compatible with accessible readers like Thorium and Colibrio out of the box. Audible's own AAX/AAXC + Whispersync stack is the closed equivalent of this.
- **M4B + JSON sidecar** *(fallback)* — a chaptered audio file plus a small JSON file with the timing / chapter / text data. Easier to consume from a custom-built player.

## What this tool does NOT do

- It does not play audio. The listening UI is a separate project.
- It does not bypass DRM.
- It does not host, distribute, or fetch books — you bring your own input.

## Status

Early scaffolding. Project layout, agent skills, and tooling are in place; the generator pipeline is not yet wired up.

See **[PLAN.md](./PLAN.md)** for the design, format research, phased roadmap, and open questions.
