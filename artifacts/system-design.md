# System Design — paste

> A technical design document covering the current architecture and a scaled-product vision.

---

## Section 1 — Current Architecture

### Detection Pipeline (Strategy Pattern)

Each detector is a pure, isolated function with a single responsibility: take a raw string, return a match object or `null`. They are composed into an ordered chain — the first match wins, no ambiguity.

```
detect(raw: string) → DetectionResult | null

DetectionResult {
  type:  string        // "phone" | "email" | "url" | ...
  value: string        // original input
  clean: string        // normalized value (digits only, hex, ISO date, etc.)
  ...extras            // type-specific: lat/lng for coords, iso for dates
}
```

Each detector follows this contract implicitly:

```js
function detectPhone(s)   { /* regex + length guard */ }
function detectEmail(s)   { /* RFC-ish regex */ }
function detectURL(s)     { /* protocol prefix check */ }
function detectColor(s)   { /* hex + rgb() pattern */ }
function detectCoords(s)  { /* lat,lng bounds check */ }
function detectAddress(s) { /* street keyword presence */ }
function detectDate(s)    { /* Date.parse + year/month guard */ }
function detectCard(s)    { /* digit count + Luhn algorithm */ }
function detectCode(s)    { /* keyword prefix + operators */ }
function detectAirport(s) { /* 3 uppercase letters */ }
```

The pipeline runs them in exactly that order, returning immediately on the first truthy result. Order matters — GPS coordinates are checked before street addresses to prevent `37.7749, -122.4194` from matching the address detector's digit heuristics.

---

### UI State Machine

The interface has four discrete states. Transitions are one-directional — no shared mutable state, no component framework needed.

```
         [User opens app]
               │
               ▼
          ┌─────────┐
          │  IDLE   │ ◄─────────────────────────────┐
          └────┬────┘  textarea cleared              │
               │ user types / pastes                 │
               ▼                                     │
        ┌──────────────┐                             │
        │  DETECTING   │  (120ms debounce window)    │
        └──────┬───────┘                             │
               │ detect() returns                    │
       ┌───────┴────────┐                            │
       │                │                            │
       ▼                ▼                            │
  ┌─────────┐    ┌────────────┐                      │
  │  NULL   │    │   RESULT   │ ─────────────────────┘
  │ (text)  │    │  (typed)   │  input cleared
  └────┬────┘    └────────────┘
       │ show plain text actions
       └──────────────────────────────────────────►
```

---

### Observer Pattern — Input → Render

A single event listener drives the entire application. No state variables, no reactive store.

```
[textarea 'input' event]
        │
        ▼
  clearTimeout(timer)
        │
        ▼
  setTimeout(120ms)   ← debounce window
        │
        ▼
   detect(value)      ← pure function, no side effects
        │
        ▼
   render(result)     ← DOM mutations only, idempotent
        │
        ├── badgeEl.textContent = ...
        ├── previewEl.textContent = ...
        ├── actionsEl.innerHTML = ...
        └── resultEl.classList.toggle('visible')
```

---

### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        BROWSER                              │
│                                                             │
│  ┌──────────┐   input    ┌─────────────┐                   │
│  │ textarea │ ─────────► │  Debouncer  │ 120ms             │
│  └──────────┘            └──────┬──────┘                   │
│                                 │                           │
│                                 ▼                           │
│                    ┌────────────────────────┐              │
│                    │   Detection Chain       │              │
│                    │  ┌──────────────────┐  │              │
│                    │  │ detectPhone()    │  │              │
│                    │  │ detectEmail()    │  │              │
│                    │  │ detectURL()      │  │              │
│                    │  │ detectColor()    │  │              │
│                    │  │ detectCoords()   │  │              │
│                    │  │ detectAddress()  │  │              │
│                    │  │ detectDate()     │  │              │
│                    │  │ detectCard()     │  │              │
│                    │  │ detectCode()     │  │              │
│                    │  │ detectAirport()  │  │              │
│                    │  └──────────────────┘  │              │
│                    └────────────┬───────────┘              │
│                                 │ DetectionResult           │
│                                 ▼                           │
│                    ┌────────────────────────┐              │
│                    │   Action Builder        │              │
│                    │   buildActions(det)     │              │
│                    └────────────┬───────────┘              │
│                                 │ ActionCard[]              │
│                                 ▼                           │
│                    ┌────────────────────────┐              │
│                    │      Renderer           │              │
│                    │   render(det, cards)    │              │
│                    └────────────────────────┘              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### Why the Architecture Is Intentionally Stateless

- **No state = no bugs from stale state.** The entire UI is derived from a single `detect()` call.
- **Idempotent rendering.** Calling `render(result)` twice with the same input produces the same DOM.
- **Testable in isolation.** Every detector is a pure function. No mocks needed.
- **Zero re-render overhead.** There is no virtual DOM diffing — the DOM is only written when detection fires.

---

### Module Breakdown

| Module | Responsibility |
|---|---|
| `EXAMPLES` | Static array of chip values, rendered once on load |
| `detect(raw)` | Ordered detection chain, returns first match |
| `luhn(s)` | Pure Luhn checksum validation for credit cards |
| `buildActions(det)` | Maps detection result → array of action card configs |
| `TYPE_META` | Maps type key → display label + icon |
| `render(det)` | Mutates DOM to reflect current detection state |
| `copyText(text)` | Clipboard write with legacy fallback |
| `showToast(msg)` | Fixed toast notification with auto-dismiss |

---

## Section 2 — Scaling to a Product

### REST vs WebSocket for Real-time Detection

For a hosted version, **REST is correct**. Detection is a stateless request-response operation — no persistent connection is needed. WebSocket would add complexity for zero benefit here.

```
POST /api/detect
Content-Type: application/json

{ "input": "+1 (415) 555-0193" }
```

```json
{
  "type": "phone",
  "confidence": 0.97,
  "value": "+1 (415) 555-0193",
  "clean": "14155550193",
  "actions": [
    { "icon": "📞", "title": "Call", "href": "tel:+14155550193" },
    { "icon": "💬", "title": "Send SMS", "href": "sms:+14155550193" },
    { "icon": "📋", "title": "Copy digits", "copy": "14155550193" },
    { "icon": "🔍", "title": "Google search", "href": "https://google.com/search?q=..." }
  ]
}
```

**Confidence score** enables a hybrid model: high-confidence regex hits return instantly from the edge; low-confidence results fall through to an ML tier.

---

### Adding ML as a Fallback Tier

```
Input
  │
  ├──► Regex chain (< 1ms)         → confidence > 0.85 → return
  │
  └──► ML classifier (50–200ms)    → confidence score + type label
           │
           └──► Return best result with confidence
```

An ML fallback handles ambiguous inputs like `"Apple"` (company? fruit? airport IATA?) that regex cannot resolve without more context. A lightweight transformer fine-tuned on labeled clipboard data works well here.

---

### Caching Strategy

```
[User pastes input]
       │
       ▼
  hash(input) → SHA-256 → 8-char prefix key
       │
       ▼
  localStorage.getItem(key)
       │
  ┌────┴──────┐
  │  HIT      │ → return cached DetectionResult (< 0.1ms)
  │  MISS     │ → run detect() → store result with TTL 30min
  └───────────┘
```

Cache invalidation: TTL-based. Clipboard content doesn't change semantically, so 30 minutes is safe.

---

### Multi-user: Clip History + Cross-device Sync

**API endpoints:**
```
POST   /clips          → save a clip (type, value, timestamp)
GET    /clips          → list user's clip history (paginated)
DELETE /clips/:id      → delete a clip
GET    /clips/export   → export as JSON/CSV
```

**Database schema (PostgreSQL):**

```
┌──────────────────────────────────────────┐
│ users                                    │
├──────────────┬───────────────────────────┤
│ id           │ uuid PRIMARY KEY          │
│ email        │ text UNIQUE NOT NULL      │
│ created_at   │ timestamptz DEFAULT now() │
└──────────────┴───────────────────────────┘
          │ 1
          │
          │ n
┌──────────────────────────────────────────┐
│ clips                                    │
├──────────────┬───────────────────────────┤
│ id           │ uuid PRIMARY KEY          │
│ user_id      │ uuid REFERENCES users(id) │
│ type         │ text NOT NULL             │
│ raw_value    │ text NOT NULL             │
│ clean_value  │ text                      │
│ detected_at  │ timestamptz DEFAULT now() │
│ label        │ text                      │  ← user-assigned tag
└──────────────┴───────────────────────────┘
```

**Cross-device sync:** Server-sent events (SSE) push new clips to open sessions. No polling.

---

### Deployment Model

```
User → CDN (Cloudflare) → Edge Function (detection, cache hit)
                        → Origin API (Node.js / Bun)
                                    → PostgreSQL (Supabase / RDS)
                                    → Redis (rate limit counters)
                                    → ML Service (optional, async)
```

`paste.html` is a static asset served from CDN with a long `Cache-Control` header. Detection API runs at the edge for low latency. The ML tier runs as a separate service called only on cache miss + low regex confidence.

---

### Rate Limiting

```
Anon users:    100 requests / hour  (keyed by IP)
Auth users:    2000 requests / hour (keyed by user_id)
Burst allowed: 20 requests / 10 seconds
```

Enforced via Redis sliding window counter at the edge.

---

## Section 3 — Extensibility Design

### Plugin Architecture

A third-party can register a new detector without touching core code:

```ts
interface Detector {
  name: string;
  priority: number;          // lower = runs earlier in the chain
  detect(input: string): DetectionResult | null;
}

interface DetectionResult {
  type: string;
  value: string;
  clean: string;
  meta?: Record<string, unknown>;
}

interface ActionCard {
  icon: string;
  title: string;
  desc: string;
  href?: string;             // opens in new tab
  copy?: string;             // copies to clipboard
  primary?: boolean;         // renders full-width
}

interface ActionBuilder {
  forType: string;
  build(result: DetectionResult): ActionCard[];
}
```

**Registration:**
```js
paste.registerDetector({
  name: 'ipv4',
  priority: 35,              // runs after URL (30), before color (40)
  detect(input) {
    const match = input.match(/^(\d{1,3}\.){3}\d{1,3}$/);
    if (!match) return null;
    return { type: 'ipv4', value: input, clean: input };
  }
});

paste.registerActionBuilder({
  forType: 'ipv4',
  build(result) {
    return [
      { icon: '🌐', title: 'IP Lookup', href: `https://ipinfo.io/${result.clean}`, primary: true },
      { icon: '📋', title: 'Copy', copy: result.clean }
    ];
  }
});
```

---

### Configuration Schema

```json
{
  "version": "1",
  "detectors": {
    "phone":   { "enabled": true,  "priority": 10 },
    "email":   { "enabled": true,  "priority": 20 },
    "url":     { "enabled": true,  "priority": 30 },
    "color":   { "enabled": true,  "priority": 40 },
    "coords":  { "enabled": true,  "priority": 50 },
    "address": { "enabled": true,  "priority": 60 },
    "date":    { "enabled": true,  "priority": 70 },
    "card":    { "enabled": false, "priority": 80 },
    "code":    { "enabled": true,  "priority": 90 },
    "airport": { "enabled": true,  "priority": 100 }
  },
  "debounceMs": 120,
  "theme": "dark"
}
```

---

## Section 4 — Observability

### Metrics to Track

| Metric | Why |
|---|---|
| `detection.latency_ms` | Catch regex performance regressions |
| `detection.type_distribution` | Understand real-world usage patterns |
| `action.click_through_rate` | Which actions are actually useful |
| `detection.null_rate` | How often nothing matches (plain text fallback) |
| `clipboard.paste_count` | Total engagement signal |
| `copy_action.success_rate` | Clipboard API permission failures |

### Instrumentation

```js
// Lightweight, no external SDK
function track(event, props) {
  navigator.sendBeacon('/analytics', JSON.stringify({
    event,
    props,
    ts: Date.now(),
    session: sessionId
  }));
}

// Usage
track('detection.fired', { type: result.type, latencyMs: t1 - t0 });
track('action.clicked', { type: result.type, action: card.title });
```

`sendBeacon` is non-blocking and survives page unload — critical for click-through tracking on action cards that open new tabs.

### Error Boundaries

- **Clipboard API failure:** Fallback to `document.execCommand('copy')` already implemented. If both fail, toast says "copy failed — try manually."
- **Detection throws:** Wrapped in try/catch; falls back to plain text result. Never surfaces errors to the user.
- **Date.parse edge cases:** Guarded with year/month name requirement — prevents `"123"` or `"99"` from matching.
- **Regex catastrophic backtracking:** All regexes are anchored (`^...$`) and bounded. No unbounded `.*` adjacent to `+` or `*`.
