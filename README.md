# `paste`

**Paste anything. Understand everything.**

[![License: MIT](https://img.shields.io/badge/License-MIT-black?style=flat-square&labelColor=black&color=e8ff47)](LICENSE)
[![No Dependencies](https://img.shields.io/badge/dependencies-none-black?style=flat-square&labelColor=black&color=e8ff47)](paste.html)
[![Single File](https://img.shields.io/badge/delivery-single%20file-black?style=flat-square&labelColor=black&color=e8ff47)](paste.html)
[![Works Offline](https://img.shields.io/badge/offline-ready-black?style=flat-square&labelColor=black&color=e8ff47)](paste.html)

---

## What it does

`paste` is a zero-install, single-file web app that detects what you paste and instantly surfaces the most useful thing you can do with it. Drop in a phone number and get click-to-call. Drop in a hex color and see the swatch. Drop in GPS coordinates and jump straight to a map. Ten content types, instant recognition, no server, no tracking, no nonsense.

**Live demo:** [https://pritmon.github.io/smart-paste/paste.html](https://pritmon.github.io/smart-paste/paste.html)

---

## Detected types

| Type | Example Input | Actions |
|------|--------------|---------|
| 📞 Phone | `+1 (415) 555-0132` | Call, Send SMS, WhatsApp, Copy formatted |
| 📧 Email | `hello@example.com` | Compose email, Copy, Search web, Add to contacts |
| 🔗 URL | `https://github.com/pritmon` | Open link, Copy, Archive via Wayback, QR code |
| 🎨 Color | `#e8ff47` / `rgb(232,255,71)` | Preview swatch, Copy HEX, Copy RGB, Copy HSL |
| 📍 GPS | `37.7749, -122.4194` | Open in Google Maps, Apple Maps, Copy, Share |
| 🏠 Address | `1600 Pennsylvania Ave NW, Washington DC` | Google Maps, Street View, Copy, Search web |
| 📅 Date | `April 9, 2026` / `2026-04-09` | Add to Calendar, Days from today, Copy ISO, Search |
| 💳 Credit Card | `4111 1111 1111 1111` | Identify network, Validate (Luhn), Mask, Copy last 4 |
| 💻 Code | `` `const x = () => {}` `` | Copy, Detect language, Open in sandbox, Format |
| ✈️ Airport | `SFO` / `LAX` | Flight search, Airport info, Google Maps, Copy IATA |
| 📋 Plain text | `anything else` | Copy, Word count, Search web, Translate |

---

## How it works

Detection runs top-down through an ordered chain of regex patterns and validators. The **first match wins** — no ambiguity, no second-guessing.

1. **Credit card** is checked with a full Luhn algorithm before pattern matching, not just digit count, so `1234 5678 9012 3456` won't false-positive unless it passes the checksum.
2. **GPS coordinates** are matched before street addresses to avoid treating `40.7128, -74.0060` as a street name.
3. **Airport codes** require an exact 3-letter IATA match against a bundled list of ~250 major airports — not just any 3-letter string.
4. **Colors** match hex (`#fff`, `#e8ff47`), `rgb()`, `rgba()`, `hsl()`, and CSS named colors.
5. **Code snippets** are detected heuristically: presence of brackets, semicolons, indentation, or keywords like `function`, `const`, `def`, `=>`.
6. **URLs** require a valid scheme (`http`, `https`, `ftp`) or a resolvable TLD pattern.
7. Everything else falls through to **plain text**.

---

## Usage

No install. No build step. No dependencies.

```bash
# 1. Download
curl -O https://pritmon.github.io/smart-paste/paste.html

# 2. Open
open paste.html
```

Or just download `paste.html` and double-click it. Works offline after the first load (Google Fonts are the only external request, and they are optional).

---

## Tech

| Layer | Choice |
|-------|--------|
| Markup | HTML5 |
| Styles | CSS3 — custom properties, keyframe animations, grid/flexbox |
| Logic | Vanilla JS (ES2020) — no frameworks, no bundler |
| Fonts | [Syne](https://fonts.google.com/specimen/Syne) (UI) + [DM Mono](https://fonts.google.com/specimen/DM+Mono) (code/data) via Google Fonts |
| Accent | `#e8ff47` electric yellow-green |
| Theme | Dark, always |

Everything — HTML, CSS, and JS — lives in a single `paste.html` file. The total unminified size is under 50 KB.

---

## Development

```bash
git clone https://github.com/pritmon/smart-paste.git
cd smart-paste
open paste.html
```

All styles are in a `<style>` block and all logic is in a `<script>` block inside `paste.html`. There is no build process, no `package.json`, no bundler. Edit the file, reload the browser.

---

## License

[MIT](LICENSE) — Copyright (c) 2026 pritmon
