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

### 🔴 Next Priority: Star Map System Overhaul

**Rename tab from "◆ Shards" to "🗺 Star Map"**

Add full planetary system data to each star map entry:

#### Rosharan System
- Star: unnamed
- Planets: **Ashyn** (origin of humans, surge-diseases), **Roshar** (3 moons: Salas, Nomon, Mishim), **Braize** (Damnation, Odium confined here), + 10 gas giants
- Perpendicularities: Honor's (mobile, Shattered Plains area) [oath], Cultivation's (Horneater Peaks) [wor]
- Shards: Honor [twok], Cultivation [wor], Odium on Braize [twok] → Retribution [wat]

#### Scadrian System
- Star: unnamed (yellow)
- Planets: **Scadrial**, + 2 gas giants (Aagal Nod, Aagal Uch), + 2 dwarf planets
- Perpendicularities: Well of Ascension (Preservation's) [woa], Pits of Hathsin (Ruin's) [tfe]
- Shards: Preservation [tfe], Ruin [tfe] → Harmony [hoa]

#### Selish System
- Star: Mashe (yellow)
- Planets: **Donne** (barren), **Sel**, + 2 gas giants (Ky, Ralen), + 1 dwarf planet
- Perpendicularity: Pool near Elantris (sapphire liquid) [elan]
- Shards: Devotion [elan], Dominion [elan] — both Splintered, power in Cognitive Realm [sh]

#### Nalthian System
- Star: unnamed (yellow)
- Planets: **Nalthis**, gas giant **Farkeeper the Bright**, dwarf planet **Nightstar the Hidden**, + comet belt
- Perpendicularity: Hallandren jungles (Endowment's) [warb]
- Shards: Endowment [warb]

#### Taldain System
- Stars: **AisDa** (blue-white supergiant), **Eye of Ridos** (white dwarf with particulate ring) — binary system
- Planets: **Taldain** (tidally locked, single planet)
- Perpendicularity: Autonomy creates/collapses them dynamically [ws_ex]
- Shards: Autonomy [ws_ex]

#### Threnodite System
- Planets: **Monody**, **Elegy** (moon: Coronach), **Threnody**, gas giant **Purity**
- Perpendicularity: No stable one — only unpredictable, morbid-origin ones [shadows]
- Shards: Ambition — Splintered here but actually died elsewhere [shadows]

#### Drominad System
- Planets: **First of the Sun** through **Seventh of the Sun** (7 planets, 3 inhabited, all water-dominant)
- Perpendicularity: On First of the Sun — mystery, no known Shard [6th]
- Shards: None resident. Autonomy's avatar Patji [bom]

#### UTol System
- Planets: **UTol**, **Komashi** — binary planet system around one sun
- Perpendicularity: Unknown
- Shards: Virtuosity (Splintered voluntarily) [yumi]

#### Lumar System
- Planet: **Lumar** with 12 geostationary moons
- Perpendicularity: Unknown
- Shards: None. Aethers present [tress]

#### Implementation Plan
1. Add `systemData` object to each PLANET entry with planets array, perps array, shards array
2. Update PlanetDetail panel to show system info when a planet is tapped
3. Add perpendicularity indicators (portal icons) on the star map SVG
4. All new data spoiler-gated per the reference table above

### 🟡 Medium Priority

- **Worldhopper Directory** — Characters who appear across multiple books (Demoux, Galladon, Vasher/Zahel, Vivenna/Azure, Khriss, Nazh, Felt). Each with origin world, current location, and appearances. All spoiler-gated.
- **Cosmere Organizations** — Ghostbloods, Seventeenth Shard, the Ire, Night Brigade, the Set. Goals, known members, book appearances.
- **Connection Web** — Interactive diagram showing how books connect (which characters cross over, which events reference each other). Tap a connection line to see what the crossover is.
- **Easter Eggs / "Connections You May Have Missed"** — Per-book list of Cosmere connections. E.g., "The three men at the Purelake in TWoK interlude are Demoux (Mistborn), Galladon (Elantris), and Baon (White Sand)."

### 🟢 Lower Priority / Nice to Have

- **Individual World Maps** — Detailed maps for Roshar (Shattered Plains, Urithiru, Alethkar, etc.), Scadrial (Luthadel, Elendel), Sel (Arelon, Fjordell). Each with key locations spoiler-gated.
- **Investiture Encyclopedia** — The three Realms (Physical, Cognitive, Spiritual), Realmatic Theory, Connection, Identity, Fortune. How worldhopping works.
- **Upcoming Books tracker** — Horneater (2026), Ghostbloods trilogy (2028-2030), Elantris sequels (2029-2030), Stormlight 6 (2031).
- **Persistent Storage** — Save read state to localStorage or the artifact storage API so it persists across sessions.
- **Share Mode** — Generate a URL or export that lets someone else see your reading state and the spoiler-appropriate lore.

---

## User's Current Reading State

### Read ✅
- **Stormlight Archive:** Way of Kings, Words of Radiance, Edgedancer, Oathbringer, Dawnshard, Rhythm of War, Wind and Truth
- **Mistborn Era 1:** The Final Empire, The Well of Ascension, The Hero of Ages
- **Mistborn Era 2:** The Alloy of Law, Shadows of Self, The Bands of Mourning (NOT The Lost Metal)
- **Standalones:** Elantris, Warbreaker
- **Arcanum Unbounded:** The Emperor's Soul, The Hope of Elantris, Mistborn: Secret History, Sixth of the Dusk, Edgedancer
- **The Eleventh Metal:** Unsure

### Not Yet Read ❌
- The Lost Metal (owned)
- Tress of the Emerald Sea (owned)
- Yumi and the Nightmare Painter
- The Sunlit Man
- Isles of the Emberdark
- Allomancer Jak
- Shadows for Silence
- White Sand

### Suggested Reading Order
1. The Lost Metal → 2. Tress → 3. Yumi → 4. Shadows for Silence → 5. Allomancer Jak → 6. The Eleventh Metal → 7. White Sand → 8. The Sunlit Man → 9. Isles of the Emberdark

---

## File Structure (for multi-file project)

```
cosmere-catalogue/
├── src/
│   ├── App.jsx                    # Main app, tab routing, global state
│   ├── spoiler.js                 # mkSp() and spoiler helpers
│   ├── data/
│   │   ├── books.js               # B0, AU0 arrays
│   │   ├── shards.js              # SHARDS, DAWNSHARDS arrays
│   │   ├── worlds.js              # WD array (world lore)
│   │   ├── hoid.js                # HOID, HOID_ABILITIES arrays
│   │   ├── timeline.js            # TIMELINE array
│   │   ├── starmap.js             # PLANETS, CONNS, system data
│   │   └── constants.js           # SC (series colors), WI (world icons), SER
│   ├── components/
│   │   ├── Toggle.jsx             # Show All / Progressive toggle
│   │   ├── Sec.jsx                # Expandable section component
│   │   ├── SubSec.jsx             # Sub-section component
│   │   ├── ProgressArc.jsx        # SVG progress arc
│   │   └── PlanetDetail.jsx       # Star map planet detail panel
│   ├── tabs/
│   │   ├── TabList.jsx            # Reading List tab
│   │   ├── TabWorlds.jsx          # Worlds & Lore tab
│   │   ├── TabStarMap.jsx         # Star Map tab (was Shards)
│   │   ├── TabHoid.jsx            # Hoid's Journey tab
│   │   └── TabTimeline.jsx        # Timeline tab
│   └── cosmere-tracker-working.jsx  # Current monolith (reference)
├── docs/
│   └── ARCHITECTURE.md            # This file
├── package.json
├── vite.config.js
└── index.html
```

## Setup Instructions

```bash
# Create project
npm create vite@latest cosmere-catalogue -- --template react
cd cosmere-catalogue

# Install dependencies
npm install

# Copy the working monolith as reference
# Then split into the file structure above

# Run dev server
npm run dev
```

If you want to keep it as a single-file artifact for Claude.ai, keep working in `cosmere-tracker-working.jsx` but be mindful of the 75KB size ceiling and the syntax constraints listed above.
