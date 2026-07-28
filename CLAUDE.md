# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## What this is

A map of bars in Malmö, Sweden, rated by the Instagram account
[@malmosbarer](https://www.instagram.com/malmosbarer). Visitors see dark-styled
map pins, filter by rating, tap a pin to open a card with the bar's photo,
address and directions link, or open a full list of all bars sorted by rating.

The entire site is **one file**: `index.html` — markup, CSS and JavaScript in a
single document, no build step, no dependencies installed locally, no tests.
That is deliberate. Keep it that way unless the user explicitly asks to split
things up.

## Repository layout

```
index.html    the whole application (HTML + <style> + <script>)
```

There is nothing else. No `package.json`, no bundler, no CI, no lockfiles.
Loading `index.html` in a browser is the application, and it is also what
GitHub Pages serves.

## Running and verifying changes

Open the file directly — `file:///…/index.html` — or serve the directory:

```bash
python3 -m http.server 8000   # then open http://localhost:8000
```

Both work. There is no watch/build/lint/test command; if you are tempted to run
`npm` something, you are in the wrong repo.

Verification is manual. After a change, check in a browser that:

- pins render at the right coordinates and in the rating colour,
- the rating chips (5…1) still hide and show pins,
- clicking a pin opens the card with photo/placeholder, name, address, rating,
- "Alla barer" opens the panel, and clicking a row flies the map there and
  opens the card,
- Escape closes both the panel and the card.

In a sandbox without a browser you can still screenshot the page with headless
Chromium, but only after vendoring Leaflet locally — the CDN `<script>` fails
there, and without `L` the whole script dies and the page renders empty. Copy
`index.html` to a scratch directory, `curl` the pinned Leaflet CSS/JS next to
it, point the tags at the local copies, and append `openSheet(0)` or a
`panel.classList.add('open')` to capture the card and list states.

Note that the map tiles come from external CDNs. In a sandboxed or offline
environment the tiles will not load and the orange `#tileWarn` box appears —
that is expected there and not a bug to "fix". The pins and all interaction
still work without tiles.

## External dependencies (all via CDN, pinned)

| What | Source |
| --- | --- |
| Leaflet 1.9.4 (CSS + JS) | cdnjs.cloudflare.com |
| Fonts: Fraunces, Space Mono, Work Sans | fonts.googleapis.com |
| Map tiles | CARTO dark, then OSM, then OSM France |

Keep Leaflet pinned to an exact version. Do not add new runtime dependencies
without being asked — a second library would undo the point of the single file.

## The data model

All content lives in the `BARS` array near the top of the `<script>` block:

```js
const BARS = [
  {name:'Bar Kiosko', grade:5, addr:'Nobelvägen 73B', lat:55.5894929, lng:13.0153171, img:''},
  …
];
```

- `grade` is an integer **1–5**. Every grade must have an entry in `COLORS`.
- `lat`/`lng` are decimal degrees; bars are in Malmö, so roughly
  `55.58–55.60` / `13.00–13.03`. **Always geocode, never estimate** — coordinates
  guessed from a street name have landed half a kilometre off. Look the bar up
  by name in OpenStreetMap and use the venue node:

  ```bash
  curl -sS -A 'malmos-barer-map/1.0' \
    'https://nominatim.openstreetmap.org/search?format=json&countrycodes=se&city=Malmö&street=Amiralsgatan%2047'
  ```

  Prefer the result whose `name` matches the bar over the bare house-number
  node, and sanity-check that the new pin lands near the others.
- `addr` is the street address only. The card appends `· Malmö` itself, so do
  not include the city.
- `img` is a path next to `index.html` (`'bilder/bar-kiosko.jpg'`) or an
  absolute URL. Empty string means "no photo", and the card renders a
  placeholder with the bar's initials tinted in the grade colour.
- If a bar has been reviewed more than once, the **latest** review wins — there
  is one row per bar, not one per review.

Adding a bar is normally a one-line addition to this array. Keep the array
sorted by grade descending, and keep the columns visually aligned with the
existing padding — it is hand-formatted on purpose.

## Design system

The look is "dark bar interior, brass and paper". Do not introduce new colours
or fonts ad hoc; use the tokens.

CSS custom properties on `:root`:

| Token | Value | Use |
| --- | --- | --- |
| `--ink` | `#11140F` | page and map background |
| `--ink-2` / `--ink-3` | `#1B211A` / `#252C23` | raised surfaces |
| `--paper` | `#E2DAC6` | primary text on dark |
| `--brass` | `#C89B3C` | accents, links, focus ring |
| `--line` | `rgba(226,218,198,.16)` | hairline borders |

Per-grade colour comes from the `COLORS` map in JS and is injected as the local
custom property `--c` on the element that needs it (pin, chip, score badge,
placeholder gradient). Components read `var(--c)` and never hardcode a grade
colour:

```js
const COLORS = {1:'#9B4A3A',2:'#C0703A',3:'#D3A441',4:'#8FA24E',5:'#4FA37F'};
```

Typography: Fraunces 900 for headings and the pin numerals, Space Mono for
small caps-y metadata (chips, buttons, labels), Work Sans for body text.

Ratings are plain numerals everywhere: the `.stamp` circle in the card (a
rotated ring showing the grade over `AV 5`), the `.num` column in the list, and
the numeral inside the map pin. Do not restyle the rating as emoji, stars or
bars — this was tried with bicycle emoji and reverted.

## Code conventions

- Vanilla JS, no framework, no modules, no transpilation. Assume a modern
  browser; `const`/`let`, arrow functions and optional-chaining-era syntax are
  fine.
- Compact style matching what is already there: two-space indent, minimal
  blank lines, CSS declarations packed onto shared lines with `;` separators.
- `document.getElementById(...)` inline at the point of use is the established
  pattern. Do not refactor into a cached element registry or a state container.
- Escape any user/data string that goes into `innerHTML` with the existing
  `esc()` helper. `renderList()` builds HTML strings — new fields interpolated
  there must go through `esc()`. Prefer `textContent` where the code already
  uses it (`openSheet`).
- Keep the accessibility affordances: `aria-pressed` on the filter chips,
  `aria-label` on scores and close buttons, `role="dialog"` on the panel,
  `:focus-visible` outlines, `keyboard:true` on markers, and the
  `prefers-reduced-motion` block at the end of the stylesheet.
- CSS ordering matters in one place: `.c-media [hidden]{display:none}` must stay
  last in that rule group, or the placeholder's `display:grid` beats `hidden`.
  There is a comment saying so — leave it.

## Language

The site, the UI copy, the source comments and the commit messages are all in
**Swedish**. Match that. Write new comments and any new interface text in
Swedish, and write commit messages in Swedish too, following the existing style:
a short imperative subject line, then a body explaining what changed and why.

```
Lägg till Tacopalatset och Ava vinbar, visa betyg som cyklar

Nya barer: Tacopalatset (4/5, Amiralsgatan 47) och Ava vinbar
(3/5, Karlskronaplan 7).
…
```

## Git workflow

- Default branch is `main`; work happens on feature branches and lands via pull
  request (`claude/<slug>` for AI-authored branches).
- Push with `git push -u origin <branch>`.
- Do not commit or push unless asked, and never push straight to `main`.
