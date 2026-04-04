# Cosmere Catalogue — Architecture & Development Guide

## Overview

An interactive Brandon Sanderson Cosmere reading tracker and lore encyclopedia with a **progressive spoiler system** that reveals information based on which books the user has marked as read. Built as a React app.

## Current State (Working)

The app is currently a single-file React component (`cosmere-tracker-working.jsx`) at ~73KB. It works in Claude's artifact renderer but is hitting size limits for further development. The move to a proper multi-file React project will allow continued expansion.

### What's Built and Working

1. **📚 Reading List tab** — Book tracker with read/owned toggles, series grouping, page counts, progress bars, suggested reading order
2. **🌍 Worlds tab** — 10 world cards with expandable Shard sections and magic system details, all spoiler-gated
3. **◆ Shards tab (rename to "Star Map")** — Collapsible 16 Shards table, 4 Dawnshards table, interactive SVG star map with clickable planets
4. **🎭 Hoid tab** — Journey timeline with Known Abilities section (progressive reveal), appearances per book
5. **⏳ Timeline tab** — Cosmere events grouped by era with time references

### Critical Environment Constraints (Claude Artifacts)

If continuing to deploy as a Claude artifact (`.jsx` single file), these constraints apply:

- **Only `import { useState } from "react"` works reliably.** `useMemo`, `useContext`, `createContext` cause "returnReact is not defined" errors.
- **No IIFEs returning JSX.** `{(()=>{ return <div/> })()}` breaks the bundler. Extract to named components instead.
- **Use `function(){}` syntax, not arrow functions, for component functions and JSX callbacks.** Arrow functions in data definitions are fine.
- **Use `var s = useState(x); var val = s[0]; var set = s[1];`** instead of destructured `const [val, set] = useState(x)` for maximum compatibility.
- **File size limit:** ~75KB seems to be the practical ceiling. Beyond that, the bundler may silently fail.
- **No template literals in style props** — use string concatenation: `"1px solid " + color + "33"` not `` `1px solid ${color}33` ``

If deploying as a standalone React app (Vite/Next.js), none of these constraints apply.

---

## Architecture

### Centralized Spoiler System

The core of the app. Every piece of lore, every UI element that reveals story information, goes through this system.

```javascript
// Creates spoiler checker functions from current read state
function mkSp(books, auStories, showAll) {
  const readSet = new Set();
  books.forEach(b => b.read && readSet.add(b.id));
  auStories.forEach(a => a.read && readSet.add(a.id));
  return {
    // u(requirements) — returns true if ALL required books are read (or showAll is on)
    u: (req) => showAll || req.every(id => readSet.has(id)),
    // uA(requirements) — returns true if ANY required book is read (for "unlock when first encountered")
    uA: (req) => showAll || req.length === 0 || req.some(id => readSet.has(id)),
    showAll
  };
}
```

The `sp` object is created at the App level and passed to all tab components as a prop. Every gated piece of content calls `sp.u(["book_id", ...])` to check if it should be shown.

#### Spoiler Gate Patterns

```javascript
// Simple gate: show after reading one specific book
{ text: "Sazed becomes Harmony", req: ["hoa"] }

// Multi-book gate: show only after reading ALL listed books  
{ text: "Kelsier is Thaidakar", req: ["sh", "row"] }

// Progressive reveal: base version + upgraded version
{
  name: "Illusion Magic",
  nameAfter: { t: "Lightweaving (Yolish)", r: ["sh"] },
  desc: "Creates images from smoke and light.",
  descAfter: { t: "Original form of illusion magic from Yolen.", r: ["sh"] }
}

// Status that changes: Honor is "Splintered" then becomes "Merged (Retribution)"
{
  status: "Splintered",
  statusAfter: { t: "Merged (Retribution)", r: ["wat"] }
}
```

### Data Schemas

#### Books (`B0` array)
```javascript
{
  id: "twok",           // Unique ID used in spoiler gates
  title: "The Way of Kings",
  series: "Stormlight Archive",
  so: 1,                // Sort order within series
  world: "Roshar",      // Planet name
  type: "Novel",        // Novel | Novella | Short Story | Graphic Novel
  pages: 1007,
  read: true,           // User's read status
  owned: true,          // User's ownership status
  note: "",             // Optional display note
  sn: ""                // Optional spoiler note (⚠ warning shown)
}
```

#### Arcanum Unbounded Stories (`AU0` array)
```javascript
{
  id: "sh",
  title: "Mistborn: Secret History",
  ps: "Mistborn",       // Parent series label
  world: "Scadrial",
  type: "Novella",
  pages: 190,
  read: true,
  note: ""
}
```

#### Shards (`SHARDS` array)
```javascript
{
  n: 1,                  // Shard number (1-16)
  name: "Honor",
  intent: "Oaths & bonds",
  planet: "Roshar",
  vessel: "Tanavast",
  status: "Splintered",
  sA: { t: "Merged (Retribution)", r: ["wat"] },  // Optional status upgrade
  rN: ["twok"],          // Req to show NAME
  rP: ["twok"],          // Req to show PLANET
  rV: ["wor"],           // Req to show VESSEL
  rS: ["wor"],           // Req to show STATUS
  notes: [               // Expandable notes, each gated
    { t: "Dalinar briefly held Honor", r: ["wat"] }
  ]
}
```

#### Dawnshards (`DAWNSHARDS` array)
```javascript
{
  n: 1,
  cmd: "Change",         // The Command
  holder: "Rysn Ftori",
  loc: "Roshar (Aimia)",
  rR: ["dawn"],          // Req to REVEAL this Dawnshard exists
  rH: ["dawn"],          // Req to show HOLDER
  rC: ["dawn"],          // Req to show COMMAND name
  notes: [{ t: "...", r: ["dawn"] }]
}
```

#### Hoid Appearances (`HOID` array)
```javascript
{
  book: "twok",          // Book ID (gates visibility)
  alias: "Wit (King's Wit)",
  world: "Roshar",
  color: "#4a9eff",      // World color
  what: "Description of what he does...",
  magic: ["Illusion Magic"]  // Magic systems demonstrated
}
```

#### Hoid Abilities (`HOID_ABILITIES` array)
```javascript
{
  name: "Illusion Magic",
  nameAfter: { t: "Lightweaving (Yolish)", r: ["sh"] },  // Progressive name
  desc: "Base description...",
  descAfter: { t: "Upgraded description...", r: ["sh"] },  // Progressive desc
  color: "#e2e8f0",
  req: ["twok"]          // When to first show this ability
}
```

#### Timeline Events (`TIMELINE` array)
```javascript
{
  era: "Modern",         // Mythic | Ancient | Classical | Modern | Future
  y: 20,                 // Sort order within era
  name: "Gavilar Assassinated",
  desc: "Detailed description...",
  world: "Roshar",
  color: "#4a9eff",
  req: ["twok"],         // Spoiler gate
  big: true              // Optional: larger font for major events
}
```

#### World Lore (`WD` array)
```javascript
{
  name: "Roshar",
  icon: "⛈",
  color: "#4a9eff",
  bg: "#1a2744",
  db: "Base description (always shown).",
  ds: [                  // Progressive description paragraphs
    { r: ["twok"], t: "Text revealed after TWoK..." }
  ],
  sh: [                  // Shards on this world (each expandable)
    {
      name: "Honor",
      r: ["twok"],       // When to show this Shard
      d: "Short description",
      su: [              // Sub-items (expandable within the Shard)
        { t: "Vessel: Tanavast", r: ["wor"], c: "Details..." }
      ]
    }
  ],
  mg: [                  // Magic systems (same structure as shards)
    {
      name: "Surgebinding",
      r: ["twok"],
      d: "Description",
      su: [
        { t: "The Ten Orders", r: ["wor"], c: "List..." }
      ]
    }
  ],
  bk: ["twok", "wor", ...]  // Book IDs set on this world
}
```

---

## Spoiler Gate Reference

This documents WHICH book unlocks WHICH piece of information. This is critical for accuracy — several gates were refined through user testing.

### Roshar Spoiler Gates
| Info | Gated Behind | Reasoning |
|------|-------------|-----------|
| Honor exists | twok | Mentioned as "the Almighty" |
| Honor's name (Tanavast) | wor | Named in epigraphs |
| Honor was Splintered | wor | Revealed in WoR |
| Cultivation exists | wor | Barely mentioned in TWoK |
| Cultivation is a dragon | wat | Revealed in WaT |
| Odium exists | twok | Referenced as enemy |
| Odium's vessel (Rayse) | wor | Named in epigraphs |
| Taravangian takes Odium | row | End of RoW |
| Retribution (Honor+Odium merged) | wat | End of WaT |
| Dalinar held Honor | wat | WaT climax |
| Singers are original inhabitants | oath | Major Oathbringer reveal |
| Nightwatcher = Cultivation's spren | oath | Oathbringer |
| Dalinar's memory loss from Cultivation | oath | Oathbringer |
| Shardplate from lesser spren | row | RoW reveal |
| Living vs Dead Blades | wor | WoR reveal |
| Anti-Investiture | row | RoW Navani discovery |

### Scadrial Spoiler Gates
| Info | Gated Behind | Reasoning |
|------|-------------|-----------|
| Preservation/Ruin exist | tfe | Core to TFE |
| Vessel names (Leras/Ati) | hoa | HoA reveals |
| Vin holds Preservation | hoa | HoA climax |
| Sazed becomes Harmony | hoa | HoA ending |
| Harmony struggling | aol | Era 2 theme |
| Discord possibility | bom | BoM hints |
| Allomancy is of Preservation | hoa | HoA cosmere reveal |
| Basic 8 metals | tfe | Introduced in TFE |
| Higher metals | woa | Introduced in WoA |
| God metals (Lerasium) | hoa | HoA reveal |
| Twinborn | aol | Era 2 concept |

### Cross-Cosmere Gates
| Info | Gated Behind | Reasoning |
|------|-------------|-----------|
| The Shattering happened | sh | Secret History / broader lore |
| Odium killed Sel's Shards | sh | Secret History context |
| Yolish Lightweaving | sh | Deep cosmere lore |
| Autonomy's avatars | bom | BoM southern Scadrial |
| Autonomy invasion of Scadrial | tlm | Lost Metal plot |
| Kelsier = Cognitive Shadow | sh | Secret History |
| Dawnshard = Command | dawn | Dawnshard novella |
| Dawnshard "Exist" + Sigzil | wat | WaT ending |
| Sigzil gave Dawnshard away | sunlit | Sunlit Man |
| Hoid's Dawnshard passed to Sigzil | wat | WaT ending |

---

## Roadmap

### ✅ Completed

- **Star Map System Overhaul** — Tab renamed to "✦ Star Map". Full planetary system data (stars, moons, perpendicularities, Shards) for 11 systems. Orbital diagrams in PlanetDetail. Shard diamond indicators on star map SVG (colored ◆ per Shard). Worldhopping connection lines.
- **Interactive Roshar Map** — Embedded iframe to roshar.17thshard.com in PlanetDetail (collapsible section).
- **Profiles & Persistence** — Multi-profile localStorage system. Default profile "crave" (undeletable). Profile dropdown in header. All read/owned state persists across sessions.
- **Reading Order** — Dual reading order (community/publication) with toggle switch. Defaults to 4 items with "+show more" to expand.
- **Spoiler Gating** — "16 Shards" count gated behind Dawnshard. Worlds tab descriptions spoiler-gated. Shard table entries gated per book. Star map planets show ??? for unread worlds.
- **Connections Tab** — 🕸 Connections tab with epigraph sets data. Hoid's First Letter (TWoK) has chapter-by-chapter text.
- **Shard Table** — Colored diamond prefix per Shard in table. SHARD_COLORS map for consistent theming.
- **AU Owned Button** — Unread AU stories show owned/need toggle.

### 🔴 Next Priority

- **Hoid Tab Spoiler Gating** — Description text ("Present at the Shattering", "Collecting magic") visible to new users with no books read. Needs gating behind Secret History or appropriate book.
- **Complete Epigraph Text** — Remaining Stormlight epigraph sets need full chapter-by-chapter text matching Coppermind format. Currently have representative samples for most sets except Hoid's First Letter (complete).

### 🟡 Medium Priority

- **Cosmere Easter Eggs Section** — Expand 🕸 Connections tab into a full reference:
  - **Epigraph Connection Map** — Visual diagram showing which epigraph sets connect to which Shards/worlds/characters.
  - **Worldhopper Tracker** — Characters appearing on worlds they're not from (Zahel=Vasher, Azure=Vivenna, Felt, Demoux, etc.). Revealed as you read both source and appearance books.
  - **Cross-book References** — Nightblood in Stormlight, Hoid's stories referencing other worlds, shared magic system connections.
  - **"Connections You May Have Missed"** — Per-book list of Cosmere connections. E.g., "The three men at the Purelake in TWoK interlude are Demoux (Mistborn), Galladon (Elantris), and Baon (White Sand)."
- **Worldhopper Directory** — Characters who appear across multiple books (Demoux, Galladon, Vasher/Zahel, Vivenna/Azure, Khriss, Nazh, Felt). Each with origin world, current location, and appearances. All spoiler-gated.
- **Cosmere Organizations** — Ghostbloods, Seventeenth Shard, the Ire, Night Brigade, the Set. Goals, known members, book appearances.

### 🟢 Lower Priority / Nice to Have

- **Investiture Encyclopedia** — The three Realms (Physical, Cognitive, Spiritual), Realmatic Theory, Connection, Identity, Fortune. How worldhopping works.
- **Upcoming Books tracker** — Horneater (2026), Ghostbloods trilogy (2028-2030), Elantris sequels (2029-2030), Stormlight 6 (2031).
- **Share Mode** — Generate a URL or export that lets someone else see your reading state and the spoiler-appropriate lore.

---

## Architecture Notes

- **Single-file app** — Everything lives in `index.html`. No build step, no dependencies beyond CDN-loaded React/Babel.
- **Reading state** — Stored per-profile in localStorage (`cosmere_catalogue_profiles` key). Default profile "crave" cannot be deleted.
- **Reading order** — Dual mode (community/publication) with toggle switch. Defaults to showing 4 unread books with "+show more".
- **Deployment** — `git push` to main deploys to GitHub Pages automatically.
