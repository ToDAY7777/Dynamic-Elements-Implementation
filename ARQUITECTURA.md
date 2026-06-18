# Asignación 6 — PokéDex Discovery Dashboard

## 1. API Seleccionada
**PokeAPI v2** — https://pokeapi.co/docs/v2  
API REST pública, sin autenticación, con endpoints para listar y detallar pokémon.

### Endpoints utilizados
| Endpoint | Uso |
|---|---|
| `GET /pokemon?limit=300` | Lista inicial de nombres + URLs |
| `GET /pokemon/{id}` | Detalles: sprites, stats, tipos, habilidades |
| `GET /pokemon-species/{id}` | Descripción de texto (flavor_text) |

### Claves JSON inspeccionadas
```
sprites.other.official-artwork.front_default   → imagen HD
sprites.front_default                          → fallback
types[].type.name                              → tipo primario/secundario
stats[].base_stat + stats[].stat.name         → estadísticas base
abilities[].ability.name + abilities[].is_hidden
height, weight, base_experience
flavor_text_entries[].flavor_text (lang: es/en)
```

---

## 2. Arquitectura General

### HTML casi vacío
El `<body>` tiene sólo 4 divs vacíos (`#header`, `#app`, `#grid`, `#modal-overlay`). Todo el UI se construye en tiempo de ejecución con `document.createElement()`.

### Flujo de datos
```
init()
 └─ fetch /pokemon?limit=300    → state.allPokemon[]
     └─ applyFilters()           → state.filtered[]
         └─ loadNextPage()       → fetch /pokemon/{id} × 24 (Promise.all)
             └─ buildCard(data)  → DOM insertado en #grid
```

---

## 3. Decisiones Técnicas de Rendimiento

### Por qué `transform` y `opacity` (no `margin`/`display`)
- `transform` y `opacity` son propiedades **compuestas**: el navegador las delega a la GPU sin recalcular el layout (reflow) ni repintar el documento (repaint).  
- Animar `margin`, `height` o `top` fuerza un **reflow** completo en cada frame → jank visible.
- Técnica usada en las tarjetas: `opacity: 0 → 1` + `translateY(18px) → 0` + `scale(0.96 → 1)`.

### Mecanismo de Ocultación Híbrido (modal)
```css
/* CERRADO */
opacity: 0;
visibility: hidden;   /* ← el elemento NO recibe eventos de puntero */

/* ABIERTO */
opacity: 1;
visibility: visible;
```
Usar sólo `opacity: 0` dejaría el overlay invisible pero **interceptando todos los clics** del resto de la página. Añadir `visibility: hidden` lo retira del árbol de eventos sin causar layout shift.

### Animación de barras de estadísticas
Las barras usan `width` dentro de un contenedor con `overflow: hidden`. El navegador sólo necesita repintar la barra interior (contenida), lo que lo convierte en una operación barata comparada con animar el elemento directamente en el flujo.

### `Promise.all` para lote de 24 requests
En lugar de 24 fetches secuenciales (lento), se lanzan todos en paralelo y se espera al más lento del lote. Tiempo típico: ~400 ms en lugar de ~4–8 s.

---

## 4. Patrones Implementados

| Requisito | Implementación |
|---|---|
| Fetch async/await | `async function apiFetch(url)` con try/catch y mensajes de error visuales |
| DOM 100% dinámico | `buildCard()`, `showModal()`, `buildHeader()` — zero HTML hardcodeado |
| Animaciones GPU | `transform + opacity` con `cubic-bezier(0.34, 1.56, 0.64, 1)` (spring) |
| Modal híbrido | `opacity + visibility` en `#modal-overlay`, `scale + translateY` en `#modal` |
| Búsqueda/filtro | Filtro en memoria sobre `state.allPokemon` sin re-fetching |
| Paginación | "Cargar más" con lotes de 24, caché en objeto de estado |
| Accesibilidad | `role="button"`, `tabindex`, `aria-label`, `prefers-reduced-motion` |
| Responsive | CSS Grid `auto-fill minmax(160px, 1fr)` + media queries |

---

## 5. Citación de Documentación
- **PokeAPI Docs**: https://pokeapi.co/docs/v2
- **MDN — Using Fetch**: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch
- **MDN — CSS will-change / transform**: https://developer.mozilla.org/en-US/docs/Web/CSS/will-change
- **Google Web Fundamentals — Rendering Performance**: https://web.dev/rendering-performance/
