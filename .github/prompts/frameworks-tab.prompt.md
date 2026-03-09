# Feature Prompt: FRAMEWORKS Explorer Tab

## Goal
Add a new **FRAMEWORKS** tab to the main navigation bar (`DatasetsTabMenu.vue`) that allows
users to explore all ESG framework specifications served by the specification service.

---

## Feature Description

The Frameworks Explorer is a read-only reference page. It lets any user (authenticated or not)
browse every framework available in the specification service, inspect its schema tree, search for
data points by name or key, and view the detailed definition of any selected data point type.

---

## Page Layout

The page uses a **three-column layout** inside `<TheContent>`:

```
┌──────────────────┬──────────────────────────────────┬─────────────────────────┐
│  LEFT SIDEBAR    │       CENTER PANEL               │    RIGHT PANEL          │
│                  │                                  │                         │
│  Framework List  │  Schema Tree + Search            │  Data Point Details     │
│                  │                                  │                         │
│  • SFDR          │  🔍 [Search data points...]      │  Name: Scope 1 GHG …    │
│  • EU Taxonomy   │                                  │  Base type: extendedDec │
│    (Financials)  │  ▼ environmental                 │  Definition: …          │
│  • EU Taxonomy   │    ▼ greenhouseGasEmissions       │  Constraints: min:0     │
│    (Non-Fin.)    │       • scope1GhgEmissions  ←──  │  Used by: sfdr, pcaf    │
│  • PCAF          │       • scope2GhgEmissions       │                         │
│  • Nuclear & Gas │    ▼ energyPerformance           │                         │
│                  │       • renewableEnergy…          │                         │
└──────────────────┴──────────────────────────────────┴─────────────────────────┘
```

### Left Sidebar — Framework List
- Renders a list of all frameworks using data from `GET /specifications/frameworks`.
- Each entry shows the framework `name` (e.g. "SFDR").
- Clicking a framework loads its schema into the center panel.
- The active/selected framework is visually highlighted.
- Use PrimeVue `Listbox` or a styled list with cursor-pointer hover state.
- `data-test="framework-list"` on the container, `data-test="framework-item-{id}"` on each item.

### Center Panel — Schema Tree + Search
- Shows the 3-level nested schema (`category > subcategory > dataPointKey`) for the selected framework.
- Uses the schema part of `GET /specifications/frameworks/{id}` (parsed from the JSON string).
- Provides a text search input (`InputText`) that filters data point keys by name in real time.
- Categories and subcategories are collapsible (use PrimeVue `Panel` with toggleable state, or a custom accordion).
- Each data point key is a clickable row. Clicking it selects that data point and populates the right panel.
- Show a loading spinner (`DatalandProgressSpinner`) while fetching.
- `data-test="schema-search-input"` on the search field.
- `data-test="schema-tree"` on the tree container.
- `data-test="data-point-item-{dataPointKey}"` on each leaf row.

### Right Panel — Data Point Details
- Appears when a data point leaf is selected from the schema tree.
- Fetches `GET /specifications/data-point-types/{dataPointTypeId}` on selection.
- Displays:
  - **Name** (`name`)
  - **Business Definition** (`businessDefinition`)
  - **Base Type** (`dataPointBaseTypeId`)
  - **Constraints** (comma-separated list, e.g. `min:0`)
  - **Used by frameworks** (`frameworkOwnership` — shown as pill/badge tags)
- Shows a placeholder message "Select a data point to see its details" when nothing is selected.
- `data-test="data-point-detail-panel"` on the container.

---

## Data Sources

| Data | Endpoint | When |
|---|---|---|
| Framework list | `GET /specifications/frameworks` | On page mount |
| Framework schema | `GET /specifications/frameworks/{id}` | On framework selection |
| Data point details | `GET /specifications/data-point-types/{id}` | On data point selection |

The schema field of `FrameworkSpecification` is a **JSON string** — parse it with `JSON.parse()` to get the nested category/subcategory/key structure.

---

## Route

```ts
{
  path: '/frameworks',
  name: 'Frameworks Explorer',
  component: FrameworksExplorerPage,   // lazy-loaded
  meta: {
    initialTabId: 'frameworks',
    requiresAuthentication: false,
  },
}
```

## Tab Entry (in `DatasetsTabMenu.vue`)

```ts
{ id: 'frameworks', label: 'FRAMEWORKS', route: '/frameworks', isVisible: true },
```

---

## Files to Create / Modify

| File | Action |
|---|---|
| `src/components/pages/FrameworksExplorerPage.vue` | Create — main page component |
| `src/components/resources/frameworksExplorer/FrameworkSidebar.vue` | Create — left sidebar |
| `src/components/resources/frameworksExplorer/FrameworkSchemaTree.vue` | Create — center tree/search |
| `src/components/resources/frameworksExplorer/DataPointDetailPanel.vue` | Create — right details panel |
| `src/services/ApiClients.ts` | Modify — add spec service client |
| `src/router/index.ts` | Modify — add `/frameworks` route |
| `src/components/general/DatasetsTabMenu.vue` | Modify — add frameworks tab |

---

## Constraints & Conventions

- Use `<script setup lang="ts">` for all new components.
- Wrap the page in `<TheContent>`.
- Import PrimeVue components individually.
- Never use `any` — type all API responses explicitly.
- Add `data-test` attributes on all interactive and key display elements.
- Do NOT change indentation of existing files unless adding new code.
