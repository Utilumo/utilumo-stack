# Utilumo Stack

**The libraries and browser APIs behind [Utilumo](https://utilumo.com) — doing real
file work in the browser with no server.**

Utilumo is ~174 browser tools. Converting video, analysing audio, manipulating
PDFs, running SQL. All of it happens in the tab.

This repo documents what that actually takes: which libraries do the heavy
lifting, which parts we wrote ourselves, and where the honest exceptions are.

## The constraint

The site is a **Next.js static export** on Cloudflare Pages. There is no
application backend — not "we choose not to upload your files", but *there is
nothing to upload them to*. Everything below follows from that.

It also explains the batch limits, which are generous because nobody is paying
for bandwidth or queue capacity:

| | Per batch | Total |
|---|---|---|
| Images | 50 files | 500 MB |
| PDFs | 25 files | 500 MB |
| Audio | 10 files | 500 MB |
| Video | 10 files | 1 GB |

No account, no sign-up, no "one file free".

## What we wrote ourselves

| Component | What it does |
|---|---|
| **Audio analysis** | Tempo and key detection: onset-energy envelope, autocorrelation tempo estimation, chroma vectors mapped to MIDI note classes. Runs in a Web Worker so the UI stays responsive. Not a library wrapper. |
| **ZIP writer** | ~150 lines. Hand-rolled CRC-32 table, stored entries, container assembled by hand. Used by 11 tools for batch downloads. No zip dependency. |
| **robots.txt matcher** | Longest-match precedence, wildcards, `$` anchoring, per-user-agent group selection. |
| **CSP parser** | Directive analysis and risk flagging. |
| **Redaction patterns** | Email, URL, phone, IP and card-like number detection for the text redactor. |

## What we use

| Library | Used for |
|---|---|
| [`mediabunny`](https://github.com/Vanilagy/mediabunny) | All audio and video conversion. Reaches WebCodecs internally — we never call `VideoEncoder`/`AudioEncoder` directly. |
| `@mediabunny/mp3-encoder`, `@mediabunny/flac-encoder` | MP3 and FLAC encoding. No browser's WebCodecs implementation offers either, so these bundled WASM encoders are what make local MP3/FLAC output possible at all. |
| [`@ffmpeg/ffmpeg`](https://github.com/ffmpegwasm/ffmpeg.wasm) | Audio fallback only, when a browser's codec support is not enough to read a source file. See the exceptions below. |
| [`pdf-lib`](https://github.com/Hopding/pdf-lib) | PDF merge, split, rotate, compress, metadata. |
| [`pdfjs-dist`](https://github.com/mozilla/pdf.js) | PDF page preview. |
| [`sql.js`](https://github.com/sql-js/sql.js) | SQLite compiled to WASM — the SQL playgrounds run a real engine in the tab. |
| [`jsqr`](https://github.com/cozmo/jsQR) | QR decoding from an image. |
| [`qrcode`](https://github.com/soldair/node-qrcode) | QR generation. |
| [`fflate`](https://github.com/101arrowz/fflate) | Reading compressed container formats in the compliance importer. |
| [`read-excel-file`](https://github.com/catamphetamine/read-excel-file) | XLSX parsing for the DPP data mapper. |
| [`yaml`](https://github.com/eemeli/yaml) | YAML ↔ JSON. |

## Browser APIs

| API | Used for |
|---|---|
| **Canvas 2D** | The entire image pipeline — resize, crop, rotate, flip, re-encode, EXIF stripping, ASCII conversion, pixel diff. 13 modules. |
| **Web Audio** (`AudioContext` / `OfflineAudioContext`) | Decoding and resampling for audio merge, waveform rendering and song analysis. |
| **WebCodecs** | Audio and video encode/decode — reached through MediaBunny rather than called directly. |
| **Web Workers** | Song analysis, so DSP does not block the main thread. |
| **WebAssembly** | SQLite (`sql.js`), the MP3/FLAC encoders, and the ffmpeg fallback. |

Metadata stripping is a side effect of the architecture rather than a feature: an
image re-encoded through a canvas has no EXIF to carry over.

## The honest exceptions

Two things use the network. Neither sends your data to Utilumo:

**Audio conversion fallback.** MP3, WAV and FLAC output encode locally with the
bundled encoders. If a browser cannot *read* a particular source file, an ffmpeg
WASM build is fetched from a CDN and the conversion still runs in your tab. We
measured this: WebKit fails on some FLAC sources where Chromium and Firefox
succeed, so the fallback is real, not theoretical.

**REST Request Tester.** Sends the request you build to the URL you choose. That
is the entire point of the tool. It goes straight to that address, never through
us, and is never proxied or logged.

Every tool page carries its own privacy note. The
[Privacy Policy](https://utilumo.com/privacy-policy) is authoritative.

## The tools this makes possible

Picked for *uncommon*, not popular. A JSON formatter is not interesting; every
tool site has one. These are the ones where running locally is surprising, or
where the tool itself is rare.

### Heavy compute

| Tool | |
|---|---|
| [Song Analyzer](https://utilumo.com/audio/song-analyzer) | BPM, musical key, Camelot notation — the hand-written DSP above |
| [Video Compressor](https://utilumo.com/video/video-compressor) | WebCodecs re-encode in the tab, verified on Chromium, Firefox and WebKit |
| [Split Video](https://utilumo.com/video/split-video) | Equal MP4 segments, downloaded as a ZIP |
| [SQLite Playground](https://utilumo.com/developer/sqlite-playground) | A real SQLite engine — tables, seed rows, queries |
| [Audio Merge](https://utilumo.com/audio/audio-merge) | Concatenate audio files, export one WAV |

### Rare utilities

| Tool | |
|---|---|
| [Invisible Character Inspector](https://utilumo.com/text/invisible-character-inspector) | Hidden Unicode with names, code points, counts, positions |
| [Compress Image to Size](https://utilumo.com/image/compress-image-to-size) | Hits an exact KB/MB target instead of a quality slider |
| [Visual Image Diff](https://utilumo.com/image/visual-image-diff) | Slider, onion overlay, side-by-side, pixel difference |
| [QR Code Reader](https://utilumo.com/creator/qr-code-reader) | Decodes from an image and shows the real hostname before you open it |
| [ICS Calendar Merger](https://utilumo.com/productivity/ics-calendar-merger) | Merge several `.ics` files into one |

### Privacy with a mechanism

| Tool | |
|---|---|
| [Text Redactor](https://utilumo.com/privacy/text-redactor) | Pattern-detects emails, URLs, phones, IPs, card-like numbers |
| [Screenshot Privacy Cleaner](https://utilumo.com/privacy/screenshot-privacy-cleaner) | Capture and redact in one workflow |
| [Remove EXIF](https://utilumo.com/image/remove-exif) | 50 images per batch, no account |

### Standards and security

| Tool | |
|---|---|
| [robots.txt Tester](https://utilumo.com/developer/robots-txt-tester) | Which rule actually matches a URL — Google retired its own tester |
| [CSP Analyzer](https://utilumo.com/developer/csp-analyzer) | Flags risky directives, emits a hardened starter policy |
| [Security Headers Generator](https://utilumo.com/developer/security-headers-generator) | Build a header set without pasting your domain into a scanner |
| [security.txt Generator](https://utilumo.com/developer/security-txt-generator) | RFC 9116, with a [field reference](https://utilumo.com/reference/security-txt-fields) |

### AI-era

| Tool | |
|---|---|
| [Prompt Injection Scanner](https://utilumo.com/ai/prompt-injection-scanner) | Overrides, hidden instructions, disclosure requests, exfiltration signals |
| [llms.txt Generator](https://utilumo.com/developer/llms-txt-generator) | Resources, categories, usage notes, sitemap |
| [JSON to Zod](https://utilumo.com/developer/json-to-zod) | Nested objects, optional fields |

## Open data

[Browser tool privacy snapshot](https://utilumo.com/research/browser-tool-privacy-snapshot)
— dataset, methodology and CSV / JSON downloads. Free to cite and re-use.

---

Utilumo itself is closed source. This repo documents the stack; questions are
welcome in [issues](https://github.com/Utilumo/utilumo-stack/issues).
