# Assignment 6 — PokéDex Discovery Dashboard

## 1. Selected API
**PokeAPI v2** — https://pokeapi.co/docs/v2  
Free public REST API, no authentication required, with endpoints to list and detail Pokémon.

### Endpoints Used
| Endpoint | Purpose |
|---|---|
| `GET /pokemon?limit=300` | Initial list of names + URLs |
| `GET /pokemon/{id}` | Details: sprites, stats, types, abilities |
| `GET /pokemon-species/{id}` | Flavor text description |

### Inspected JSON Keys
```
sprites.other.official-artwork.front_default   → HD artwork image
sprites.front_default                          → fallback sprite
types[].type.name                              → primary/secondary type
stats[].base_stat + stats[].stat.name         → base statistics
abilities[].ability.name + abilities[].is_hidden
height, weight, base_experience
flavor_text_entries[].flavor_text (lang: es/en)
```

---

## 2. General Architecture

### Nearly Empty HTML
The `<body>` contains only 4 empty divs (`#header`, `#app`, `#grid`, `#modal-overlay`). All UI is built at runtime using `document.createElement()`.

### Data Flow
```
init()
 └─ fetch /pokemon?limit=300    → state.allPokemon[]
     └─ applyFilters()           → state.filtered[]
         └─ loadNextPage()       → fetch /pokemon/{id} × 24 (Promise.all)
             └─ buildCard(data)  → DOM node inserted into #grid
```

---

## 3. Performance Technical Decisions

### Why `transform` and `opacity` (not `margin`/`display`)
- `transform` and `opacity` are **composited properties**: the browser delegates them to the GPU without recalculating the layout (reflow) or repainting the document (repaint).
- Animating `margin`, `height`, or `top` forces a full **reflow** on every frame → visible jank.
- Technique used on cards: `opacity: 0 → 1` + `translateY(18px) → 0` + `scale(0.96 → 1)`.

### Hybrid Hide Mechanism (modal)
```css
/* CLOSED */
opacity: 0;
visibility: hidden;   /* ← element does NOT receive pointer events */

/* OPEN */
opacity: 1;
visibility: visible;
```
Using only `opacity: 0` would leave the overlay invisible but **intercepting all page clicks**. Adding `visibility: hidden` removes it from the event tree without causing a layout shift.

### Stats Bar Animation
The bars animate `width` inside a container with `overflow: hidden`. The browser only needs to repaint the inner bar (contained), making it a cheap operation compared to animating the element directly in the document flow.

### `Promise.all` for batches of 24 requests
Instead of 24 sequential fetches (slow), all are launched in parallel and wait for the slowest in the batch. Typical time: ~400ms instead of ~4–8s.

---

## 4. Implemented Patterns

| Requirement | Implementation |
|---|---|
| Async fetch with async/await | `async function apiFetch(url)` with try/catch and visual error messages |
| 100% dynamic DOM | `buildCard()`, `showModal()`, `buildHeader()` — zero hardcoded HTML |
| GPU-accelerated animations | `transform + opacity` with `cubic-bezier(0.34, 1.56, 0.64, 1)` (spring easing) |
| Hybrid modal hide | `opacity + visibility` on `#modal-overlay`, `scale + translateY` on `#modal` |
| Search/filter | In-memory filter over `state.allPokemon` without re-fetching |
| Pagination | "Load more" with batches of 24, cached in state object |
| Accessibility | `role="button"`, `tabindex`, `aria-label`, `prefers-reduced-motion` |
| Responsive layout | CSS Grid `auto-fill minmax(160px, 1fr)` + media queries |

---

## 5. API Documentation Citation
- **PokeAPI Docs**: https://pokeapi.co/docs/v2
- **MDN — Using Fetch**: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch
- **MDN — CSS transform / opacity**: https://developer.mozilla.org/en-US/docs/Web/CSS/will-change
- **Web.dev — Rendering Performance**: https://web.dev/rendering-performance/
