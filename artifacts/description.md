# Project Description — paste

> Smart clipboard web app. Paste anything. It knows what to do.

**Live demo:** https://pritmon.github.io/smart-paste/paste.html

---

## The Idea

Every day people copy things — a phone number from a website, an address from an email, a hex color from a design file, a URL from Slack — and then immediately wonder what to do with it. They open a new tab, navigate somewhere, paste it in, and complete the action manually. It's three steps when it should be one.

`paste` collapses that friction. Paste something into the box and before you reach for the mouse, the right buttons are already there. Call this number. Open this link. View this address on a map. Scan this URL for threats. The interface doesn't ask what you want to do — it already knows.

The design goal was a single feeling: *"how did it know."*

---

## What Makes It Different

Most "smart paste" tools are either detection-only (they tell you what something is but don't help you act) or they show generic actions regardless of context (search, translate, copy — the same three buttons for everything).

`paste` does neither. It detects the type first, then surfaces exactly the actions that make sense for that specific thing — and only those actions. A credit card number gets a BIN lookup and HaveIBeenPwned check, not a Google search. A GPS coordinate gets satellite view and navigation, not LinkedIn. A color code gets a palette explorer and a color info page, not a calendar event.

The result is an interface that feels like it was built specifically for whatever you just pasted.

---

## Detection Engine

The detection system runs an ordered chain of 10 specialized functions, each a pure regex or algorithmic check. The first match wins — no scoring, no ambiguity.

| # | Type | Method |
|---|---|---|
| 1 | Phone number | Regex on digit count (7–15), stripped of formatting |
| 2 | Email address | RFC-approximate pattern |
| 3 | URL | Protocol prefix (`http://`, `https://`) |
| 4 | Color code | Hex (`#fff`, `#e8ff47`) or `rgb()` function |
| 5 | GPS coordinates | Decimal `lat, lng` with bounds check (±90, ±180) |
| 6 | Street address | Street keyword presence (street, avenue, road, lane…) |
| 7 | Date | `Date.parse()` + mandatory year or month name to prevent false positives |
| 8 | Credit card | 13–19 digit string + Luhn algorithm checksum |
| 9 | Code snippet | Keyword prefix (`const`, `def`, `function`, `SELECT`…) or `=>` or HTML tags |
| 10 | Airport code | Exactly 3 uppercase letters, optionally prefixed `IATA:` |

The chain is deterministic and fast. On worst-case input (plain text, no match), all 10 checks run in under 1ms. Detection fires 120ms after the user stops typing — fast enough to feel instant, slow enough to not thrash on every keystroke.

---

## Design Philosophy

**Dark theme.** Clipboard tools get used in the middle of work. A dark interface doesn't interrupt focus the way a white one does at 11pm.

**Syne + DM Mono.** Syne (headings, labels, action titles) is geometric and confident — it carries weight without being aggressive. DM Mono (body, textarea, hints) makes content feel like data rather than prose. The combination signals precision.

**Electric yellow-green accent (#e8ff47).** It's the only color in the UI that carries meaning. Everything that matters — the detected value, the badge, the focus ring, hover states — uses it. The constraint makes it feel intentional rather than decorative.

**Action cards over menus.** Menus hide choices until you interact. Cards expose all actions at once, making the interface scannable in under a second. The primary action spans full width, so the most likely action is also the easiest to click.

**No modals, no steps, no settings.** You paste. It detects. You click. That's the entire interaction model.

---

## Zero Dependency Constraint

`paste` has no npm, no build step, no bundler, no framework. One HTML file. One Google Fonts import.

This constraint isn't a limitation — it's a feature:

- **Portable.** The file works identically whether you open it from a USB drive, a file server, GitHub Pages, or a local folder. No configuration, no environment.
- **Durable.** No dependencies means no supply chain, no breaking changes from upstream packages, no security vulnerabilities in `node_modules`. A file you download today will still work in ten years.
- **Trustworthy.** Users can read the entire source in one scroll. There is no obfuscated code, no external requests (except Google Fonts on first load), no telemetry. What you see is what runs.
- **Fast.** Zero JavaScript to download beyond the inline script. No hydration. No framework overhead. The app is interactive in under 100ms on any connection.

---

## Performance

| Metric | Value |
|---|---|
| Total file size | < 50 KB |
| External requests | 1 (Google Fonts, first load only) |
| Time to interactive | < 100ms on any connection |
| Detection latency | < 1ms (worst case, all 10 checks) |
| Debounce window | 120ms |
| Network requests after first load | 0 |
| Dependencies | 0 |

After first load, `paste` works completely offline. The fonts are cached by the browser. Everything else is inline.

---

## What's Next

**1. PWA / installable app**
Add a `manifest.json` and service worker. `paste` is already offline-capable — installing it as a PWA would make it a proper system-level clipboard companion, launchable from the dock or taskbar.

**2. Browser extension**
A Chrome/Firefox extension could intercept clipboard events automatically — no need to manually paste into a box. Right-click on selected text → "paste actions" context menu with all detected actions inline.

**3. ML fallback tier**
Regex handles clean, structured input well. A lightweight ML classifier (fine-tuned on labeled clipboard samples) would handle ambiguous inputs that the ordered chain misses — company names, product names, mixed-format data.

**4. API mode**
Expose the detection engine as a `POST /detect` endpoint. Third-party apps could send arbitrary text and receive structured detection results with actions. Useful for note-taking apps, productivity tools, and anything that handles user-pasted content.

**5. Custom detector plugins**
Let power users define their own detection patterns via a JSON config or a simple JS plugin interface. Team-specific formats — internal ticket IDs, Jira references, internal URL patterns — could all be added without forking the project.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Markup | HTML5 |
| Styling | CSS3 — custom properties, keyframe animations, grid |
| Logic | Vanilla JavaScript (ES6+) — no transpilation |
| Fonts | Google Fonts — Syne (headings), DM Mono (body) |
| Build tool | None |
| Runtime | Any modern browser |
| Hosting | Any static file server, or just double-click |

---

## Try It

**https://pritmon.github.io/smart-paste/paste.html**

Or download `paste.html` and open it in any browser. That's it.
