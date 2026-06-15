# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

大学受験料シミュレーター — a single-page Japanese university entrance exam fee calculator. Helps high school students calculate total exam fees across multiple universities, applying each university's specific discount rules automatically.

## Development

There is no build system, package manager, or test framework. The entire application lives in a single file: `index.html`. To develop:

- Open `index.html` directly in a browser (file:// is fine — no server needed)
- Edit `index.html` and reload the browser

## Architecture

Everything — HTML structure, CSS, JavaScript data, and business logic — is embedded in `index.html`. The file is ~3200 lines organized roughly as:

- **Lines 10–262**: CSS with custom properties (`--accent`, `--bg`, `--text`, etc.)
- **Line 287**: `DB` object — the central data store for all universities
- **Lines ~1628–1655**: Utility functions (`fmt()` for ¥ formatting, `escA()` for HTML-escaping)
- **Lines ~2457–2498**: Central event handler using delegation on `data-action` attributes
- **Lines ~2501–2622**: `calcRows()` — the fee calculation engine
- **Lines ~2733–2862**: Cart operations and navigation (`addToCart`, `removeFromCart`, `backToStep1/2`, etc.)
- **Lines ~2876–3036**: Per-university UI renderers
- **Lines ~3039–3239**: `render()` — master renderer called after every state change

### Data Model (`DB`)

Each university entry in `DB` has:
```js
"大学名": {
  type: "hosei",          // optional; drives which renderer is called
  methods: {
    "入試方式": {
      baseFee: 35000,     // fee for first application
      extraFee: 15000,    // discounted fee for subsequent applications
      nFee / thirdFee,   // optional: special 2nd/3rd-app fees
      note: "...",        // displayed discount rule explanation
      faculties: { "学部名": ["専攻1", "専攻2"] },
      englishFaculties / commonTestFaculties,  // optional subsets
      selectAll,          // boolean: show "select all" button
      hasEnglish / hasCommonTest
    }
  }
}
```

### State Management

Global state is held in plain JS variables:
- `cart`: Array of `{ univ, method, faculties }` — the user's selections
- `cur`: `{ university, method, selected, expandedFaculty }` — current wizard step
- Per-university state objects for universities with complex UI: `hoseiState`, `chuoState`, `rikkyoState`, `nduState`, `dokkyoState`, `kokugakuinState`, `musashiState`, `senshuState`, `toyoState`, `aogakuState`

After any mutation, call `render()` to rebuild the DOM via `innerHTML`.

### UI Pattern

Three-step wizard:
1. **Step 1** — Select university (grouped into tiers: 早慶上理, GMARCH, 四工大, etc.)
2. **Step 2** — Select admission method (一般, 共通テスト, 英語外部, etc.)
3. **Step 3** — Select faculties/departments with live fee preview

Buttons carry `data-action="actionName"` attributes. The single delegated click handler at the top of the JS section reads `e.target.closest('[data-action]')` and routes to the appropriate function.

### Fee Calculation (`calcRows`)

This is the most critical and complex function. It returns an array of `{ label, fee, note }` rows for display. Key behaviors:
- First application uses `baseFee`, subsequent use `extraFee` (or `nFee`/`thirdFee` for universities with 3-tier pricing)
- 20+ university-specific special cases: Shibaura free 4th+ apps, Dokkyo external-exam bundling, Musashi faculty bundling, Toyo same-day bundling, NDU A+N method coupling, etc.
- Discount savings are computed as `(baseFee × count) - actualTotal` and displayed in the cart

### Adding a New University

1. Add an entry to `DB` following the existing schema
2. If the university has non-standard discount rules, add a case to `calcRows()`
3. If the university needs a custom faculty-selection UI, write a `render<Name>UI()` function and add a branch in `render()` that calls it based on `DB[cur.university].type`
4. If new per-university accordion/toggle state is needed, add a state object and its `reset*State()` call in `backToStep1()` and `backToStep2()`

### Conventions

- All user-visible strings are in Japanese
- Prices are integers in yen; use `fmt(n)` for display
- Always HTML-escape dynamic attribute values with `escA(s)` before inserting into `innerHTML`
- `render()` is called after every state change — it rebuilds the full DOM, so avoid holding references to DOM nodes across renders
- University tier groupings in Step 1 are hardcoded in `render()`; update them when adding new universities
