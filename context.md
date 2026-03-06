# Catan Log — Project Context Document
> Last updated: Sprint 3 complete. For use as AI assistant context on resumption.

---

## Project Overview

A mobile Progressive Web App (PWA) for logging Catan board game sessions. Built iteratively across sprints by a non-developer Product Owner working with an AI coding assistant.

**Live URL:** `https://jvelas11.github.io/catan-log/`
**Repository:** GitHub — `catan-log` repo, main branch
**Primary device:** Android, Chrome browser
**Installed as:** PWA (added to home screen)

---

## Tech Stack

| Concern | Decision | Rationale |
|---|---|---|
| App type | PWA (single HTML file) | No app store, instant deploy, works on Android |
| Framework | Vanilla HTML/CSS/JS | No build step, one file to manage |
| Storage | localStorage | No backend needed for prototype |
| AI integration | Anthropic Claude API (direct from browser) | Image recognition for board analysis |
| Deployment | GitHub Pages | Free, simple, branch-based previews |
| Model | claude-sonnet-4-20250514 | Used for board photo analysis |

**Critical note on API key:** The Anthropic API key is stored in localStorage on the user's device. This is acceptable for a personal prototype but would not be appropriate for a production app (key should live server-side). Users configure it via Settings.

---

## Repository Structure

```
catan-log/
├── index.html        # Entire application — HTML, CSS, JS in one file
├── manifest.json     # PWA manifest
└── icon.png          # App icon (192x192px hexagon design)
```

---

## Data Model

Stored in `localStorage` under key `catan-db` as a JSON object.

```javascript
{
  players: [
    {
      id: string,           // uid() generated
      name: string,
      preferredColor: string, // hex color e.g. '#c0392b'
      createdAt: number     // timestamp
    }
  ],
  games: [
    {
      id: string,
      date: string,         // ISO date 'YYYY-MM-DD'
      photo: string|null,   // base64 data URL
      boardAnalysis: object|null, // result from Claude API (see AI Analysis)
      expansion: string,    // 'base'|'seafarers'|'cities'|'traders'|'explorers'
      notes: string,
      participants: [
        {
          id: string,
          playerId: string,       // → players[].id
          settlements: number,    // 0-4, 1pt each
          cities: number,         // 0-4, 2pts each
          extraPoints: number,    // 0-10, 1pt each
          victoryPoints: number,  // computed at save time (see scoring)
          isWinner: boolean,      // computed at save time
          hadLongestRoad: boolean,
          hadLargestArmy: boolean
        }
      ],
      createdAt: number
    }
  ]
}
```

### Legacy data
Original data was stored under `catan-games` (flat array, player names as strings). A migration function `attemptMigration()` runs on load and converts it to the new format. This has already run successfully for the current user — legacy key can be ignored going forward.

---

## Scoring System

```
victoryPoints = settlements
              + (cities × 2)
              + extraPoints
              + (hadLongestRoad ? 2 : 0)
              + (hadLargestArmy ? 2 : 0)
```

### Business rules
- **Victory points** are always computed, never manually entered
- **Winner** is auto-flagged when score >= 10. Not user-selectable
- **Longest Road / Largest Army** are exclusive — only one player can hold each at a time. Toggling transfers from previous holder. Each adds 2pts
- **11-point hard cap (own card):** plus buttons disable at score >= 11. Minus always enabled for corrections
- **Game-over lock (other cards):** when any player reaches 10+, all other players are capped at 9. Their plus buttons disable when score >= 9. Minus always enabled. Cards remain fully interactive otherwise (not dimmed/disabled entirely)

---

## Score Card Component

The score entry UI (Step 3 of New Game flow) renders one card per player, stacked vertically.

### Key functions
```javascript
scCalcScore(player)          // computes score from participant fields + exclusive badges
scGetWinner()                // returns first player with score >= 10, or null
scToggleExclusive(pid, badge) // transfers longestRoad/largestArmy to player, or removes
scChangeCounter(pid, field, delta) // increments/decrements a counter field
buildParticipantRows()       // main render function — builds all cards into #participant-rows
scRenderBadge(...)           // renders one exclusive badge (road/army)
scRenderCounter(...)         // renders one counter row (settlements/cities/extraPoints)
```

### Card visual states
| State | Treatment |
|---|---|
| Default | Dark surface, subtle gold border |
| Winner | Gold border, gold glow, pulsing winner badge |
| Score at 10+ | Score number turns gold with glow |
| Exclusive badge — inactive | Muted, dark |
| Exclusive badge — road active | Green tint |
| Exclusive badge — army active | Blue tint |
| Exclusive badge — disabled | 28% opacity, pointer-events none |
| Plus button — disabled | 22% opacity, cursor not-allowed |

### Pip tracks
Shown on Settlements (4 pips) and Cities (4 pips). Not shown on Extra Points (max 10, too dense).

### Score breakdown chips
Rendered at card bottom when any value is non-zero. Shows human-readable breakdown e.g. "2 settlements", "1 city ×2", "+2 longest road".

---

## AI Board Analysis

### Flow
1. User taps "Analyse Board with AI" on a saved session detail screen
2. App sends board photo (base64) to Claude API with a structured prompt
3. Claude returns JSON identifying all 19 tiles
4. App renders a hex grid SVG and saves result to `game.boardAnalysis`

### API call
```javascript
fetch('https://api.anthropic.com/v1/messages', {
  headers: {
    'x-api-key': localStorage.getItem('catan-api-key'),
    'anthropic-version': '2023-06-01',
    'anthropic-dangerous-direct-browser-access': 'true'
  },
  body: JSON.stringify({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1500,
    messages: [{ role: 'user', content: [image, prompt] }]
  })
})
```

### boardAnalysis object shape
```javascript
{
  tiles: [
    {
      row: number,
      col: number,
      type: 'wood'|'brick'|'wheat'|'sheep'|'ore'|'desert'|'unknown',
      number: number|null,
      confident: boolean
    }
    // 19 tiles, rows of 3-4-5-4-3
  ],
  summary: string,
  overallConfidence: 'high'|'medium'|'low',
  uncertainTiles: number
}
```

### Known issue
Large photos can cause an API error. Fix not yet implemented — solution is to compress the image client-side before sending (canvas resize to max 1024px before base64 encoding).

---

## App Screens

| Screen | ID | Notes |
|---|---|---|
| Home | `screen-home` | Recent sessions list, quick new game CTA |
| New Game | `screen-new` | 4-step flow (photo → players → scores → details) |
| History | `screen-history` | Full session list |
| Session Detail | `screen-detail` | Results, AI analysis, delete |
| Players | `screen-players` | Roster management, tap to edit |
| Stats | `screen-stats` | Per-player win rate, avg VP, specials |
| Settings | `screen-settings` | API key config, data management |

### New Game flow steps
- **Step 0** — Photo capture (camera or gallery). Skippable
- **Step 1** — Date + player selection from roster. Quick-add new players inline
- **Step 2** — Score entry via scorecard component (settlements, cities, extra points, road, army)
- **Step 3** — Expansion selection + free-text notes

---

## Colour Palette (CSS variables)

```css
--bg:       #0f0d08   /* page background */
--surface:  #1c1810   /* card background */
--surface2: #261f14   /* inset elements */
--surface3: #2e2416   /* deeper inset */
--gold:     #c9933a   /* primary accent */
--gold-lt:  #e8b86d   /* lighter gold */
--sand:     #d4b483   /* body text on dark */
--text:     #ede0c4   /* primary text */
--muted:    #7a6a52   /* secondary text */
--red:      #b84c3a
--green:    #4a7c59
--blue:     #3a5f8a
--radius:   14px
```

**Fonts:** Cinzel (headings, labels, numbers) + Crimson Pro (body text). Both loaded from Google Fonts.

---

## Sprints Completed

### Sprint 1 — App Shell
- Single HTML file PWA
- Photo capture (camera + gallery)
- Player name entry (free text)
- Winner selection
- Game history with localStorage persistence
- GitHub Pages deployment + PWA manifest

### Sprint 2 — AI Board Detection
- Anthropic API integration
- Manual "Analyse Board" trigger on session detail
- Hex grid SVG renderer (19 tiles, colour-coded)
- Uncertain tile indicators (dashed border + ⚠ icon)
- Confidence level display
- API key stored in localStorage, configured via Settings
- Loading state with spinner

### Sprint 3 — Game Logging
- **Data model redesign:** Player entity separated from Game. GameParticipant join table
- **Persistent player profiles:** name, preferred colour, created once, reused across games
- **Score card component:** settlements, cities, extra points counters with pip tracks; Longest Road / Largest Army exclusive badges; computed VP; winner auto-detection; game-over locking logic
- **Stats screen:** per-player wins, avg VP, specials count, win rate bar
- **Game details:** expansion field, notes field
- **Legacy data migration:** `attemptMigration()` converts old `catan-games` format. Manual recovery button in Settings
- **Branch-based deployment:** first sprint using GitHub branch workflow

---

## Known Issues / Backlog

| Item | Priority | Notes |
|---|---|---|
| Photo compression before API call | Medium | Large photos cause API error. Fix: canvas resize to max ~1024px |
| History filters | Medium | Filter by player, date, expansion, winner |
| Share session | Low | Export or screenshot a session result |
| Polish pass | Low | CSS vendor prefix warnings (`-webkit-background-clip`, `-webkit-appearance`) |

---

## Development Workflow

- **Editor:** GitHub web VS Code (press `.` on repo page) or desktop VS Code
- **Branching:** create feature branch → deploy branch via GitHub Pages Settings → test on device → merge PR → switch Pages back to main
- **Testing:** no local server available in web editor; deploy branch to GitHub Pages and test on live URL
- **No build step:** edit `index.html` directly, commit, Pages auto-deploys in ~2 minutes

---

## Key Decisions & Rationale

- **Single file app:** simplicity over maintainability. Acceptable for prototype scale
- **No backend:** localStorage only. Means no cross-device sync, but no server costs or auth complexity
- **API key in localStorage:** security trade-off accepted for personal prototype
- **Cities worth 2pts:** correct Catan rule. Settlements = 1pt each
- **Winner triggers at 10pts, not 11:** standard Catan win condition
- **11-point hard cap:** prevents impossible game states while allowing corrections via minus buttons
- **Game-over lock caps others at 9:** once someone wins, other players can record their final scores up to 9 but cannot reach 10
- **Longest Road / Largest Army exclusive:** only one player per badge per game, transfer on tap, remove on re-tap
- **Pip track omitted for Extra Points:** max of 10 is too dense for a pip visual; settlements and cities max at 4 which works well
