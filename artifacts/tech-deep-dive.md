# Tech Deep Dive — paste

A comprehensive Q&A covering the most-asked technical questions about the architecture, detection system, and design decisions behind **paste** — a zero-dependency, single-file smart clipboard web app.

---

## Q: Why a single HTML file instead of a component framework like React or Vue?

The single-file constraint was a deliberate product decision, not a technical limitation.

**Portability.** A single `.html` file can be opened directly from a filesystem, hosted on any static server, dropped into a GitHub Pages repo, or shared as an email attachment. There is no build step, no `node_modules`, no bundler configuration, no deployment pipeline. The total surface area of "things that can go wrong" is essentially zero.

**Longevity.** React 18 will not run your React 14 code without migration. Vue 3 is not compatible with Vue 2 components. Framework code rots. Vanilla HTML/CSS/JS from 2005 still runs in Chrome 2024. A file with no framework dependency is a file that works in ten years without touching it.

**Appropriate complexity.** The app has one screen, one input, one output, and a fixed set of action types. It is not a dashboard. It does not manage complex shared state across a component tree. Reaching for React here would be the engineering equivalent of driving a forklift to carry a backpack.

**Performance.** Zero JavaScript parsing overhead beyond the app's own script (~3 KB). No virtual DOM reconciliation. No hydration cost. The browser just runs the code.

**Auditability.** Security-conscious users can `View Source` and read the entire application in under 15 minutes. No webpack hash filenames, no source maps required, no obfuscated bundles. What you see is what runs.

The trade-off is real: no hot module replacement, no component isolation, no type safety at the framework layer. For an app of this scope, those trade-offs are worth it.

---

## Q: How does the detection pipeline work under the hood?

The pipeline is a **ordered strategy chain** — an array of discrete detectors evaluated top-to-bottom with first-match-wins semantics. Conceptually:

```
detect(input)
  → strip whitespace
  → if empty → return null
  → try phone
  → try email
  → try URL
  → try color
  → try GPS coords
  → try street address
  → try date
  → try credit card (Luhn)
  → try code snippet
  → try airport code
  → fallback → plain text
```

Each detector is a focused unit: it tests the trimmed input against a specific signature (regex, structural check, or algorithmic validation) and either returns a typed result object or falls through to the next detector.

The result object has a consistent shape:

```js
{
  type: 'phone' | 'email' | 'url' | 'color' | 'coords'
       | 'address' | 'date' | 'card' | 'code' | 'airport' | 'text',
  value: string,   // original input
  clean: string,   // normalized/cleaned version
  // type-specific extras, e.g.:
  lat: number,     // coords only
  lng: number,     // coords only
  iso: string,     // date only — ISO 8601 form
}
```

This normalized result object is the contract between the detection layer and the rendering layer. The renderer never knows what regex matched — it only knows the type and the cleaned value.

**Why ordered, not parallel?** Some inputs are ambiguous. `4532015112830366` is 16 digits — it could theoretically match a "number" pattern, but we want it to land on credit card. By placing the phone detector before the card detector, and restricting phone to 7–15 digits, we ensure a 16-digit string routes correctly. Order encodes priority.

**Why first-match-wins, not confidence scoring?** For a synchronous, regex-based pipeline, a confidence model adds complexity without proportional benefit. The regexes are tuned tight enough that false positives are handled by ordering and pre-flight guards rather than weighted scores.

---

## Q: Walk me through the Luhn algorithm implementation for credit card validation.

The Luhn algorithm (also called the "mod 10 algorithm") is a simple checksum formula used to validate credit card numbers, IMEI numbers, and similar identifiers. It does not verify that a card is real or active — it filters out random digit strings and typos.

Here is the implementation from the source:

```js
function luhn(s) {
  const digits = s.replace(/\D/g, '');
  let sum = 0, alt = false;
  for (let i = digits.length - 1; i >= 0; i--) {
    let n = parseInt(digits[i], 10);
    if (alt) { n *= 2; if (n > 9) n -= 9; }
    sum += n;
    alt = !alt;
  }
  return sum % 10 === 0;
}
```

**Step by step:**

1. Strip all non-digit characters. The algorithm operates only on the raw digit sequence.
2. Iterate from **right to left** (from the last digit toward the first).
3. For every **alternating** digit (starting with the second-to-last), double it. If doubling produces a number greater than 9, subtract 9. This is equivalent to summing the two digits of the doubled number (e.g., 14 → 1+4 = 5, which is also 14−9 = 5).
4. Sum all resulting digits.
5. If the total modulo 10 equals 0, the number is valid.

**Worked example with `4532015112830366`:**

```
Digits (RTL):  6  6  3  0  3  8  2  1  1  5  1  0  2  3  5  4
Alt flag:      N  Y  N  Y  N  Y  N  Y  N  Y  N  Y  N  Y  N  Y
After double:  6 12  3  0  3 16  2  2  1 10  1  0  2  6  5  8
After sub-9:   6  3  3  0  3  7  2  2  1  1  1  0  2  6  5  8
Sum: 6+3+3+0+3+7+2+2+1+1+1+0+2+6+5+8 = 50 → 50 % 10 = 0 ✓
```

The detector also pre-filters: only strings of 13–19 stripped digits are passed to `luhn()`. This avoids wasting cycles on clearly non-card input and prevents phone numbers (which can be 7–15 digits) from ever reaching the Luhn check.

---

## Q: How does debouncing improve UX here, and why 120ms specifically?

Without debouncing, every keystroke fires `detect()`. For a user typing a URL character by character, you would see the result panel flash through `text → text → text → URL` on every character addition. This is visually disruptive and wastes CPU cycles on intermediate states that are immediately invalidated.

Debouncing postpones the detection call until the user **pauses typing** for a threshold duration:

```js
let timer;
inputEl.addEventListener('input', () => {
  clearTimeout(timer);
  const val = inputEl.value;
  if (!val.trim()) { render(null); return; }
  timer = setTimeout(() => render(detect(val)), 120);
});
```

**Why 120ms?**

- **Below 100ms** feels instantaneous to humans. Responses faster than ~80ms are perceived as immediate.
- **Above 200ms** starts feeling like a noticeable lag, especially on paste events (where the full value is available instantly).
- **120ms** is in the sweet spot: fast enough that a paste operation produces near-instant results (the full value is set in a single `input` event, so the debounce fires after exactly 120ms), but slow enough that rapid keystrokes don't thrash the detection logic.

There is also an important interaction: on paste, the `input` event fires exactly once with the full pasted content. The debounce adds a 120ms delay to that single event, which is acceptable. For typed input, the debounce collapses the burst of keystrokes into a single call when typing pauses — which is when the partial input is most likely to be complete enough for meaningful detection.

The empty-string check runs synchronously before the debounce timer is set. Clearing the field clears the result immediately, with no 120ms delay. This makes the "erase" action feel snappy while the "detect" action feels considered.

---

## Q: How do you prevent false positives — e.g., a phone number matching as plain text, or a date matching a plain number?

The false-positive problem is the hardest part of a multi-type detection system. Each detector needs both a **pattern test** (what it looks like) and a **structural guard** (what it is not).

**Phone vs. credit card vs. plain number:**

```js
const phoneClean = s.replace(/[\s\-().+]/g, '');
if (/^\+?[\d\s\-().]{7,20}$/.test(s) && /\d{7,}/.test(phoneClean) && !/[a-zA-Z]/.test(s)) {
  const digits = s.replace(/\D/g, '');
  if (digits.length >= 7 && digits.length <= 15) {
    return { type: 'phone', value: s, clean: digits };
  }
}
```

The phone detector allows formatting characters (`+`, spaces, hyphens, parens) but **caps out at 15 stripped digits**. A 16-digit credit card number falls through to the Luhn check. A bare integer like `42` has only 2 digits and falls through to plain text.

**Date vs. plain number:**

```js
if ((MONTH_NAMES.test(s) || /\b\d{4}\b/.test(s)) && !isNaN(Date.parse(s))) {
  const d = new Date(s);
  if (d.getFullYear() >= 1000) {
    return { type: 'date', value: s, clean: s, iso: d.toISOString().split('T')[0] };
  }
}
```

The date detector requires either a **month name** (Jan, Feb, etc.) or a **4-digit year** before even attempting `Date.parse()`. This blocks single integers like `2024` from matching as dates by also requiring `Date.parse()` to succeed, and then further requiring the parsed year to be ≥ 1000 (which eliminates timestamps like `0000-01-01` that `Date.parse` sometimes returns for garbage input).

**GPS vs. decimal number:**

```js
const coordMatch = s.match(/^(-?\d{1,3}\.\d+)\s*,\s*(-?\d{1,3}\.\d+)$/);
if (coordMatch) {
  const lat = parseFloat(coordMatch[1]), lng = parseFloat(coordMatch[2]);
  if (lat >= -90 && lat <= 90 && lng >= -180 && lng <= 180) { ... }
}
```

The GPS detector requires **exactly two decimal numbers separated by a comma** with valid geographic range checks. A single decimal number never matches. A pair of integers (no decimal) never matches.

**Airport vs. random 3-letter string:**

```js
if (/^(IATA:\s*)?([A-Z]{3})$/.test(s.trim())) { ... }
```

This requires exactly 3 uppercase ASCII letters. Lowercase, mixed case, or strings with digits fall through. The trade-off: `THE` and `AND` would technically match. In practice, users pasting a 3-character string almost always intend an airport code. If this were a production system, a lookup table against the ~10,000 valid IATA codes would eliminate this ambiguity entirely.

---

## Q: What are the performance characteristics of running 10 regex checks on every input event?

**Very fast.** The detection pipeline runs in O(1) with respect to input length for most detectors (the regexes are anchored to the full string, so backtracking is minimal), and O(n) only for the Luhn algorithm which iterates the digit array once.

**Concrete numbers:**

- Regex evaluation on a modern V8 engine: ~0.001–0.05ms per regex for typical inputs.
- 10 regex checks in sequence: ~0.01–0.5ms total in the worst case.
- Luhn algorithm on a 19-digit string: ~0.01ms.
- Full `detect()` call end-to-end: **sub-1ms on any modern device.**

The 120ms debounce means the pipeline runs at most ~8 times per second during active typing. For a pipeline that completes in <1ms, this is negligible — the rendering (DOM manipulation) costs orders of magnitude more than the detection.

**Why no memoization?** Because it would be premature optimization. The pipeline is cheap enough that caching results would add memory overhead and cache-invalidation complexity with no measurable user benefit. If you extended the app to use ML-based detection (a network call, or a WASM model with a 50ms inference time), memoization by input hash would become worthwhile.

**Regex complexity notes:**

- All regexes use anchors (`^` and `$`) where applicable. This prevents the engine from finding partial matches and dramatically reduces backtracking.
- No catastrophic backtracking (ReDoS) vulnerabilities exist in the current patterns — they lack the nested quantifiers that cause exponential backtracking.
- `Date.parse()` is the most expensive single operation in the pipeline, but it is guarded by the `MONTH_NAMES` or 4-digit-year pre-check, so it rarely executes on inputs that don't resemble dates.

---

## Q: How does the CSS-only animation system work — no libraries, no JS?

Every animation in the app is driven by CSS `@keyframes` and CSS `transition` declarations. Zero JavaScript is used to animate anything.

**Stagger on page load** — the header, paste zone, and examples chip bar each have staggered `animation-delay` values:

```css
.header     { animation: fadeUp 0.6s ease both; animation-delay: 0.05s; }
.paste-zone { animation: fadeUp 0.6s ease both; animation-delay: 0.15s; }
.examples   { animation: fadeUp 0.6s ease both; animation-delay: 0.25s; }

@keyframes fadeUp {
  from { opacity: 0; transform: translateY(16px); }
  to   { opacity: 1; transform: translateY(0); }
}
```

`animation-fill-mode: both` (implied by `both` in the shorthand) holds the `from` state before the animation begins, which ensures elements don't flash into their final state before the delay elapses.

**Result panel reveal** — JavaScript adds/removes the `.visible` class. The animation is a CSS transition, not a JS-driven animation:

```css
.result {
  opacity: 0;
  transform: translateY(12px);
  transition: opacity 0.3s ease, transform 0.3s ease;
}
.result.visible {
  opacity: 1;
  transform: translateY(0);
}
```

The JS role is purely state management (`classList.add('visible')`). The browser handles interpolation.

**Action card stagger** — each card gets an inline `animation-delay` set by JS based on its index:

```js
el.style.animationDelay = (i * 40) + 'ms';
```

```css
.action-card {
  opacity: 0;
  transform: translateY(8px);
  animation: cardIn 0.3s ease forwards;
}
@keyframes cardIn {
  to { opacity: 1; transform: translateY(0); }
}
```

Note the use of `forwards` fill mode: without it, the card would snap back to `opacity: 0` when the animation ends.

**Hover micro-interactions** — all hover effects (border color, background, arrow translate, card lift) are pure CSS `transition` rules with no JavaScript involvement at all.

**Logo pulse** — a looping `@keyframes pulse` fades the logo dots between full and low opacity on a 2-second cycle, with the second dot offset by 0.4s.

**Toast** — uses a CSS `transition` on `opacity` and `transform`. JS adds/removes `.show` to toggle between states.

---

## Q: How would you extend this to support a new detection type without breaking existing ones?

The architecture is designed for exactly this. Adding a new detector is a three-step process:

**Step 1: Add the detector to the `detect()` function.**

Insert a new `if` block in the ordered chain at the appropriate priority position:

```js
// 10.5 Bitcoin address
if (/^(1|3)[a-km-zA-HJ-NP-Z1-9]{25,34}$|^bc1[a-z0-9]{39,59}$/.test(s)) {
  return { type: 'bitcoin', value: s, clean: s };
}
```

**Step 2: Add an action builder to `buildActions()`.**

Add a `case` to the switch statement returning the array of action cards for this type:

```js
case 'bitcoin': return [
  { icon: '₿',  title: 'View on Blockchair', desc: 'block explorer',  href: `https://blockchair.com/bitcoin/address/${clean}`, primary: true },
  { icon: '📋', title: 'Copy address',        desc: 'copy to clipboard', copy: clean },
  { icon: '🔍', title: 'Google search',       desc: 'search address',  href: `https://www.google.com/search?q=${enc(clean)}` },
];
```

**Step 3: Add the type metadata.**

```js
const TYPE_META = {
  // ...existing types...
  bitcoin: { label: 'Bitcoin Address', icon: '₿' },
};
```

**That's it.** No existing detectors are touched. No rendering logic changes. The type result object contract is upheld: every detector returns `{ type, value, clean }`. The renderer is type-agnostic — it reads `TYPE_META[det.type]` and calls `buildActions(det)`.

**Ordering consideration:** Place the new detector before detectors it could conflict with. A Bitcoin address is all-alphanumeric and would never match a phone/email/URL/color/coords/address/date/card, but it would match the 3-letter airport check if the address started with uppercase letters — so insert it before the airport detector.

**Example chips:** Optionally add an entry to `EXAMPLES` for discoverability:

```js
{ label: '₿ bitcoin', value: '1A1zP1eP5QGefi2DMPTfTL5SLmv7Divf Na' },
```

---

## Q: What are the security considerations of a clipboard app that evaluates user input?

**Client-side only.** All processing happens in the browser. The input never leaves the device. There is no server, no logging, no analytics, no telemetry. This is the most important security property of the app.

**No `eval()`, no dynamic script execution.** The detection pipeline is pure regex and math. There is no code execution path that takes user input and runs it as code.

**XSS surface.** There are two places where user input is rendered to the DOM:
- `previewEl.textContent = det.value` — uses `textContent`, not `innerHTML`. Safe; no HTML interpretation.
- The color preview: `previewEl.innerHTML = \`<span style="background:${det.clean}"></span>${det.value}\`` — this is a controlled risk. `det.clean` is the regex-matched hex value (`#rrggbb` or `rgb(r,g,b)`), which cannot contain script-injectable content due to the tight regex. `det.value` is inserted as HTML without escaping, which is a minor theoretical XSS vector for color inputs (though the color detector would never match a string containing `<script>`). A hardened version would use `textContent` for all fields and a `createElement`/`setAttribute` approach for the swatch.

**Action card `href` values.** URLs passed into `href` attributes use `encodeURIComponent` for query parameters. The primary URL for the URL detector uses the raw input as an `href` with `target="_blank" rel="noopener noreferrer"`, which is correct. `noopener` prevents the opened page from accessing `window.opener`. `noreferrer` prevents sending the referring URL.

**Clipboard API.** `navigator.clipboard.writeText()` requires the page to be served over HTTPS (or localhost) and for the user to have granted clipboard-write permission. The fallback path uses `document.execCommand('copy')` — deprecated but still widely supported as a fallback. In neither case does the app read from the clipboard; it only writes to it on explicit user action.

**No external data exfiltration.** The Google Fonts stylesheet is the only external resource loaded at startup. Detection results are never sent anywhere.

---

## Q: How does offline support work — what happens with no internet after first load?

**After first load:** The app works completely offline. All detection logic, CSS, and rendering is inline in the HTML file. Once the file is in the browser cache, no network requests are needed to use it.

The only external dependency is the Google Fonts stylesheet (`fonts.googleapis.com`). If the browser cannot reach Google Fonts:
- The `@font-face` declarations for Syne and DM Mono fail to load.
- The browser falls back to `sans-serif` (for Syne) and `monospace` (for DM Mono) — the generic font families declared in the CSS stacks.
- All functionality is intact; only the visual typography changes.

**On first load with no internet:** The app will work with system fonts only. Detection, rendering, and all actions function normally.

**Paths to full offline support:**

1. **Self-host the fonts.** Download the Syne and DM Mono WOFF2 files, base64-encode them or serve them as static assets, and replace the Google Fonts `<link>` with `@font-face` declarations pointing to local files. Now there are zero external dependencies.

2. **Add a Service Worker.** Register a service worker that caches the HTML file itself. This enables the app to work even when accessed via URL with no cached response — the service worker intercepts the navigation request and returns the cached file.

```js
// sw.js
self.addEventListener('install', e => {
  e.waitUntil(caches.open('paste-v1').then(c => c.add('/paste.html')));
});
self.addEventListener('fetch', e => {
  e.respondWith(caches.match(e.request).then(r => r || fetch(e.request)));
});
```

3. **PWA manifest.** Add a `<link rel="manifest">` with a `manifest.json` to enable "Add to Home Screen" on mobile — making the app a pinned offline-capable PWA.

---

## Q: How would you write unit tests for the detection functions?

The detection functions are pure functions: given the same input, they always return the same output, with no side effects. This makes them ideal for unit testing.

**Test framework:** Any framework works — Jest, Vitest, or plain `assert`. Extract the detection functions to a module:

```js
// detectors.js
export { detect, luhn };
```

**Test structure:**

```js
// detect.test.js
import { detect, luhn } from './detectors.js';
import { describe, it, expect } from 'vitest';

describe('luhn()', () => {
  it('returns true for a valid Visa number', () => {
    expect(luhn('4532015112830366')).toBe(true);
  });
  it('returns false for an invalid number', () => {
    expect(luhn('4532015112830367')).toBe(false);
  });
  it('handles spaces and dashes in input', () => {
    expect(luhn('4532 0151 1283 0366')).toBe(true);
  });
});

describe('detect() — phone', () => {
  it('detects a US phone with formatting', () => {
    expect(detect('+1 (415) 555-0193')).toMatchObject({ type: 'phone' });
  });
  it('does not match a 16-digit number as phone', () => {
    expect(detect('4532015112830366')).not.toMatchObject({ type: 'phone' });
  });
  it('does not match plain text as phone', () => {
    expect(detect('hello world')).not.toMatchObject({ type: 'phone' });
  });
});

describe('detect() — email', () => {
  it('detects a standard email', () => {
    expect(detect('user@example.com')).toMatchObject({ type: 'email', clean: 'user@example.com' });
  });
  it('does not match strings without @', () => {
    expect(detect('notanemail.com')).not.toMatchObject({ type: 'email' });
  });
});

describe('detect() — color', () => {
  it('detects 6-digit hex color', () => {
    expect(detect('#e8ff47')).toMatchObject({ type: 'color', clean: '#e8ff47' });
  });
  it('expands 3-digit hex to 6-digit', () => {
    expect(detect('#abc')).toMatchObject({ type: 'color', clean: '#aabbcc' });
  });
  it('detects rgb() notation', () => {
    expect(detect('rgb(255, 100, 50)')).toMatchObject({ type: 'color' });
  });
});

describe('detect() — GPS coords', () => {
  it('detects valid lat/lng pair', () => {
    const result = detect('37.7749, -122.4194');
    expect(result).toMatchObject({ type: 'coords', lat: 37.7749, lng: -122.4194 });
  });
  it('rejects out-of-range latitude', () => {
    expect(detect('91.0, 0.0')).not.toMatchObject({ type: 'coords' });
  });
});

describe('detect() — credit card', () => {
  it('detects a valid Visa number', () => {
    expect(detect('4532015112830366')).toMatchObject({ type: 'card' });
  });
  it('rejects a Luhn-invalid number', () => {
    expect(detect('4532015112830367')).not.toMatchObject({ type: 'card' });
  });
});

describe('detect() — ordering', () => {
  it('phone wins over plain text for a formatted phone', () => {
    expect(detect('+44 7911 123456')).toMatchObject({ type: 'phone' });
  });
  it('card wins over phone for a 16-digit Luhn-valid number', () => {
    expect(detect('4532015112830366')).toMatchObject({ type: 'card' });
  });
  it('URL wins over plain text for https:// input', () => {
    expect(detect('https://github.com')).toMatchObject({ type: 'url' });
  });
});
```

**Edge case tests to prioritize:**
- Inputs at boundary lengths (6-digit phone, 19-digit card)
- Leading/trailing whitespace (should be normalized by `.trim()`)
- Mixed-case inputs for detectors that are case-sensitive
- Empty string (should return `null`)
- Single characters

---

## Q: How would you handle accessibility — screen readers, keyboard navigation?

The current app has a reasonable baseline but has room for improvement. Here is an audit and remediation plan:

**What works now:**
- The `<textarea>` is a semantic form element — natively focusable and readable by screen readers.
- Example chips are `<button>` elements — keyboard-activatable, announced as "button."
- Action cards that are links use `<a>` with meaningful `title` and `desc` text.
- Action cards that copy use `<button>`.

**What needs improvement:**

**1. Live region for detection results.**

Screen readers do not announce dynamic DOM changes unless a live region is present. Add:

```html
<div aria-live="polite" aria-atomic="true" id="sr-announce" class="sr-only"></div>
```

```css
.sr-only {
  position: absolute; width: 1px; height: 1px;
  padding: 0; margin: -1px; overflow: hidden;
  clip: rect(0,0,0,0); white-space: nowrap; border: 0;
}
```

When a result is detected, update the live region:

```js
document.getElementById('sr-announce').textContent =
  `Detected ${meta.label}: ${det.value}. ${cards.length} actions available.`;
```

**2. `aria-label` on textarea.**

```html
<textarea aria-label="Paste or type content to detect" ...>
```

**3. Focus management.** After pasting and detection, optionally move focus to the first action card so keyboard users can Tab through actions without extra navigation.

**4. Keyboard activation of chips.** Currently handled correctly since chips are `<button>` elements — Enter and Space both activate them.

**5. Color contrast.** The muted color (`#666666` on `#0a0a0a`) has a contrast ratio of ~4.5:1, which passes WCAG AA for normal text (requires 4.5:1) but fails AAA (requires 7:1). Consider bumping muted text to `#888888` for better legibility.

**6. `prefers-reduced-motion`.** Wrap animations in a media query:

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

**7. Action card icons.** Emoji icons should have `aria-hidden="true"` since they are decorative:

```html
<span class="card-icon" aria-hidden="true">📞</span>
```

---

## Q: If you had to add a backend, what would the API contract look like?

The detection logic is currently client-side only, which is intentional. If you needed a backend — for server-side detection, logging, ML enrichment, or multi-device history — here is the minimal API contract:

**Endpoint:** `POST /detect`

**Request:**
```json
{
  "input": "4532015112830366",
  "options": {
    "locale": "en-US",
    "enabled_types": ["phone", "email", "url", "color", "coords", "address", "date", "card", "code", "airport"]
  }
}
```

**Response (success):**
```json
{
  "type": "card",
  "confidence": 0.99,
  "value": "4532015112830366",
  "clean": "4532015112830366",
  "meta": {
    "bin": "453201",
    "scheme": "visa",
    "issuer": "Chase"
  },
  "actions": [
    {
      "id": "bin_lookup",
      "icon": "🔎",
      "title": "BIN lookup",
      "description": "Identify card issuer",
      "href": "https://www.binlist.net/453201",
      "primary": true
    }
  ]
}
```

**Response (no match):**
```json
{
  "type": "text",
  "confidence": 1.0,
  "value": "some plain text",
  "clean": "some plain text",
  "actions": [...]
}
```

**Response (error):**
```json
{
  "error": "input_too_long",
  "message": "Input exceeds 10,000 character limit",
  "max_length": 10000
}
```

**Design notes:**
- `confidence` is a float 0–1. Client-side regex gives 0.99 for tight matches; ML fallback tier gives a calibrated probability.
- `meta` is type-specific enrichment (BIN data, timezone info, etc.) that the client cannot compute alone.
- `actions` can be generated server-side for types that require server state (e.g., "you've pasted this phone number before").
- Versioning: use `Accept: application/vnd.paste.v1+json` or a URL prefix `/v1/detect`.

---

## Q: What is the browser compatibility story? Any polyfills needed?

**Supported without polyfills (everything modern):**

| Feature | Support |
|---|---|
| CSS custom properties (`--var`) | Chrome 49+, Firefox 31+, Safari 9.1+ |
| CSS Grid | Chrome 57+, Firefox 52+, Safari 10.1+ |
| `@keyframes` + CSS transitions | Universal |
| `const`, `let`, arrow functions | Chrome 49+, Firefox 44+, Safari 10+ |
| Template literals | Chrome 41+, Firefox 34+, Safari 9+ |
| `Array.forEach`, `classList` | Universal |
| `navigator.clipboard.writeText()` | Chrome 66+, Firefox 63+, Safari 13.1+ |
| `clamp()` in CSS | Chrome 79+, Firefox 75+, Safari 13.1+ |

**The one exception — `navigator.clipboard`:**

The Clipboard API requires HTTPS and a user gesture. The fallback in the code handles non-HTTPS environments (localhost, file://) and older browsers:

```js
function copyText(text) {
  if (navigator.clipboard) {
    navigator.clipboard.writeText(text).then(() => showToast('copied'));
  } else {
    const ta = document.createElement('textarea');
    ta.value = text;
    document.body.appendChild(ta);
    ta.select();
    document.execCommand('copy');
    document.body.removeChild(ta);
    showToast('copied');
  }
}
```

`document.execCommand('copy')` is deprecated but still works in all major browsers as a fallback.

**IE11:** Not supported, not worth supporting. IE11's global market share is below 0.5% and it lacks too many features used here (`const`, template literals, CSS Grid, custom properties) to polyfill economically.

**Bottom line:** The app targets evergreen browsers (Chrome 80+, Firefox 80+, Safari 14+, Edge 80+) with no polyfills required. For the ~99.9% of users on these browsers, it works as-is.

---

## Q: How does `navigator.clipboard.writeText` work and why is there a fallback?

`navigator.clipboard` is the modern **Async Clipboard API**, introduced in 2018 and standardized in the W3C Clipboard API spec.

**How it works:**

```js
navigator.clipboard.writeText(text)
  .then(() => console.log('success'))
  .catch(err => console.error('failed:', err));
```

Internally, the browser:
1. Checks that the document has **focus** (clipboard access is denied if the tab is in the background).
2. Checks that the page is served over **HTTPS** or is `localhost` (the Permissions API requires a secure context).
3. Checks the **clipboard-write** permission — in most browsers this is auto-granted on user gesture; some browsers show a permission prompt.
4. Writes the text to the OS clipboard.

Returns a Promise that resolves on success and rejects if permission is denied, the context is not secure, or the document has lost focus.

**Why is there a fallback?**

Three scenarios where `navigator.clipboard` is unavailable or fails:

1. **Non-secure context (HTTP, file://).** The app can be opened as a local file. `navigator.clipboard` is `undefined` in insecure contexts in Chrome.

2. **Focus loss.** If the user somehow triggers a copy while the tab is not focused (e.g., via a programmatic action), the Promise rejects.

3. **Old browsers.** Safari < 13.1 does not support the Async Clipboard API at all.

The fallback uses the legacy `document.execCommand('copy')` approach:

```js
const ta = document.createElement('textarea');
ta.value = text;
document.body.appendChild(ta);
ta.select();
document.execCommand('copy');
document.body.removeChild(ta);
```

This creates an off-screen textarea, selects its content, triggers a synchronous copy command, and removes the element. It is synchronous (no Promise), works in insecure contexts, and is supported universally — though it is deprecated and browser vendors may remove it in future versions.

A belt-and-suspenders implementation wraps both in try/catch:

```js
async function copyText(text) {
  try {
    await navigator.clipboard.writeText(text);
    showToast('copied');
  } catch {
    try {
      const ta = Object.assign(document.createElement('textarea'), { value: text });
      document.body.appendChild(ta);
      ta.select();
      document.execCommand('copy');
      ta.remove();
      showToast('copied');
    } catch {
      showToast('copy failed — select manually');
    }
  }
}
```
