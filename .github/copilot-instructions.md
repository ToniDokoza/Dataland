Do not change the indentation unless explicitly asked!

# Dataland – GitHub Copilot Project Instructions

## Project Overview
Dataland is a sustainability data platform for ESG (Environmental, Social, Governance) frameworks.
Companies upload structured data for regulatory frameworks (SFDR, EU Taxonomy, PCAF, VSME, LKSG, …).
The platform is a multi-module Gradle monorepo with a Vue 3 frontend and several Spring Boot backend microservices.

---

## Repository Layout (top-level modules)

| Folder | Purpose |
|---|---|
| `dataland-frontend/` | Vue 3 SPA (Vite) – the user-facing application |
| `dataland-backend/` | Core Spring Boot backend |
| `dataland-specification-service/` | Serves framework/data-point-type JSON specs |
| `dataland-specification-lib/` | Shared Kotlin lib for specification models |
| `dataland-qa-service/` | Quality assurance service |
| `dataland-community-manager/` | Company roles & requests |
| `dataland-data-sourcing-service/` | Data request / sourcing flows |
| `dataland-document-manager/` | Document storage & retrieval |
| `dataland-email-service/` | Email dispatch |
| `dataland-keycloak/` | Keycloak configuration |
| `dataland-e2etests/` | Cypress E2E tests |

---

## Frontend Tech Stack (`dataland-frontend/`)

| Concern | Technology |
|---|---|
| Framework | Vue 3 (both Options API and `<script setup>` Composition API are used) |
| Language | TypeScript (strict) |
| Build tool | Vite |
| UI component library | PrimeVue 4.x (`primevue`, `@primevue/core`, `@primeuix/themes`) |
| CSS utility | PrimeFlex 4 (sunsetted — do NOT use in new code) |
| Icons | PrimeIcons (`pi pi-*`) — Material Icons must NOT be used in new code |
| Routing | Vue Router 5 |
| State | Pinia |
| HTTP client | Axios (auto-authenticated via `ApiClientProvider`) |
| Auth | Keycloak JS adapter, injected as `getKeycloakPromise` |
| E2E testing | Cypress 13 |
| Component testing | Cypress Component Testing |

---

## Frontend Directory Structure

```
src/
  App.vue                   # Root component, sets up auth wrapper
  main.ts                   # App bootstrap
  router/index.ts           # All routes (lazy-loaded page components)
  services/ApiClients.ts    # Central API client factory — ApiClientProvider
  frameworks/               # Per-framework type definitions, upload forms
  components/
    pages/                  # One .vue file per route/page
    general/                # Shared interactive components (tabs, dialogs, …)
    generics/               # Layout primitives: TheHeader, TheContent, TheFooter
    resources/              # Domain components grouped by feature
    forms/                  # Upload/edit form components
    wrapper/                # Auth and other wrappers
  stores/Stores.ts          # Pinia stores
  types/                    # Shared TypeScript interfaces & types
  utils/                    # Pure utility functions
  api-models/               # Frontend-local TypeScript model files
```

---

## Routing Conventions (`router/index.ts`)

- **All page components are lazy-loaded** via dynamic `import()`.
- Each route has `meta.requiresAuthentication: boolean`.
- Routes that should activate a specific tab in the `DatasetsTabMenu` carry `meta.initialTabId: string`.
- URL patterns follow `kebab-case`.

Example route entry:
```ts
{
  path: '/frameworks',
  name: 'Frameworks Explorer',
  component: FrameworksExplorerPage,
  meta: {
    initialTabId: 'frameworks',
    requiresAuthentication: false,
  },
}
```

---

## Global Navigation: `DatasetsTabMenu.vue`

`src/components/general/DatasetsTabMenu.vue` renders the top navigation bar.

### How tabs work
1. An array `tabs: TabInfo[]` holds every tab with `{ id, label, route, isVisible }`.
2. `visibleTabs` computed filters by `isVisible || id === currentTabId`.
3. `currentTabId` is derived from `route.meta.initialTabId`.
4. `onTabChange` calls `router.push(tab.route)`.

### Adding a new tab
1. Add an entry to the `tabs` array in `DatasetsTabMenu.vue`.
2. Add a route in `router/index.ts` with the matching `meta.initialTabId`.

---

## API Client Pattern (`ApiClients.ts`)

All service clients live in `ApiClientProvider`. To add a new service client:

```ts
// In ApiClients interface
specificationController: SpecificationControllerApi;

// In constructApiClients()
specificationController: this.getClientFactory('/specifications')(SpecificationControllerApi),
```

Usage in components:
```ts
const getKeycloakPromise = inject<() => Promise<Keycloak>>('getKeycloakPromise')!;
const apiClientProvider = new ApiClientProvider(assertDefined(getKeycloakPromise)());
apiClientProvider.apiClients.specificationController.listFrameworkSpecifications();
```

---

## Component Patterns

### Preferred style for new components: `<script setup>` Composition API
```vue
<script setup lang="ts">
import { ref, onMounted, inject } from 'vue';
import type Keycloak from 'keycloak-js';
import { ApiClientProvider } from '@/services/ApiClients.ts';
import { assertDefined } from '@/utils/TypeScriptUtils.ts';

const getKeycloakPromise = inject<() => Promise<Keycloak>>('getKeycloakPromise')!;
const apiClientProvider = new ApiClientProvider(assertDefined(getKeycloakPromise)());
</script>
```

Older page components use `defineComponent` Options API — do not refactor them.

### Layout wrapper
Always wrap page content in `<TheContent>`:
```vue
<template>
  <TheContent>
    <!-- page content -->
  </TheContent>
</template>
```

### PrimeVue usage
Import components individually:
```ts
import InputText from 'primevue/inputtext';
import Panel from 'primevue/panel';
import Tabs from 'primevue/tabs';
import TabList from 'primevue/tablist';
import Tab from 'primevue/tab';
import TabPanels from 'primevue/tabpanels';
import TabPanel from 'primevue/tabpanel';
```

Use PrimeFlex utility classes for layout (`flex`, `col-*`, `p-*`, `m-*`, `gap-*`).
Add `data-test="..."` attributes on key interactive elements for Cypress.

---

## Specification Service

- **Base path**: `/specifications` (proxied by the frontend dev server and nginx)
- **OpenAPI spec**: `dataland-specification-service/specificationServiceOpenApi.json`

### Key endpoints

| Endpoint | Returns |
|---|---|
| `GET /specifications/frameworks` | `SimpleFrameworkSpecification[]` |
| `GET /specifications/frameworks/{id}` | `FrameworkSpecification` |
| `GET /specifications/frameworks/{id}/resolved-schema` | `DataPointBaseTypeResolvedSchema` |
| `GET /specifications/data-point-types/{id}` | `DataPointTypeSpecification` |

### Framework JSON shape
```jsonc
{
  "id": "sfdr",
  "name": "SFDR",
  "businessDefinition": "…",
  "schema": {
    "<category>": {
      "<subcategory>": {
        "<dataPointKey>": "<dataPointTypeId>"
      }
    }
  }
}
```

### DataPointType JSON shape
```jsonc
{
  "id": "extendedDecimalScope1GhgEmissionsInTonnes",
  "name": "Scope 1 GHG emissions",
  "businessDefinition": "…",
  "dataPointBaseTypeId": "extendedDecimal",
  "frameworkOwnership": ["pcaf", "sfdr"],
  "constraints": ["min:0"]
}
```

### Available frameworks in spec service
`eutaxonomy-financials`, `eutaxonomy-non-financials`, `nuclear-and-gas`, `pcaf`, `sfdr`

**No frontend client for the specification service exists yet.** It must be added to `ApiClientProvider`.

---

## Naming Conventions

| Item | Convention |
|---|---|
| Vue component files | `PascalCase.vue` (e.g. `FrameworksExplorerPage.vue`) |
| TypeScript files | `PascalCase.ts` for classes/services, `camelCase.ts` for utilities |
| CSS/layout classes | Scoped styles only for structural layout (flex/grid/padding/margin); PrimeVue Design-Tokens (CSS vars) for component styling |
| URL routes | `kebab-case` |
| `data-test` attributes | `kebab-case` strings |

---

## Do's and Don'ts

### Do
- Keep each component/file focused on one concern
- Use `<script setup lang="ts">` for new components
- Wrap page content in `<TheContent>`
- Import PrimeVue components individually
- Add `data-test="..."` on interactive elements
- Use `assertDefined()` from `@/utils/TypeScriptUtils.ts` for non-null assertions
- Follow the `ApiClientProvider` factory pattern for new service clients
- Keep all route definitions in `router/index.ts`
- Use **PrimeIcons** (`pi pi-*`) for all icons
- Use **PrimeVue Design-Tokens** (CSS variables) to style PrimeVue components
- Use `<style scoped>` only for structural layout — `display`, `flex`, `grid`, `padding`, `margin`
- Use `zod` for form validation together with PrimeVue form elements

### Don't
- Do NOT change indentation of existing code unless explicitly asked
- Do NOT refactor existing Options API components to Composition API
- Do NOT use **PrimeFlex** utility classes — PrimeFlex is sunsetted
- Do NOT use **Material Icons** — use PrimeIcons only
- Do NOT use **FormKit** — use PrimeVue form elements instead
- Do NOT use `:deep()` — use PrimeVue's **PassThrough API** for per-instance customisation
- Do NOT modify PrimeVue component design via scoped styles
- Do NOT access or override PrimeVue CSS classes at component scope — modify globally via the preset
- Do NOT add new routes outside `router/index.ts`
- Do NOT use `any` types — always type explicitly
- Do NOT call Keycloak directly — always use the injected `getKeycloakPromise`
- Do NOT copy old code as a pattern — assume existing code may not follow current best practices

---

## Frontend Coding Guidelines

### PrimeVue
1. **Read and understand the PrimeVue API.** PrimeVue provides everything you need — find the right component, read its API, check its properties before building anything custom.
2. **Do not build unnecessary component wrappers.**
3. **Use the component's API to modify its behaviour.**
4. **Do not use FormKit.** Use PrimeVue form elements. For validation, use `zod`. See: https://primevue.org/inputtext/#forms
5. **Do not access PrimeVue CSS classes at local scope.** Modify component styling globally via the preset.
6. **Do not use `:deep()`.** For single-component customisation, use PrimeVue's PassThrough API.

### CSS & Styling
1. **Do not use PrimeFlex classes.** PrimeFlex is sunsetted.
2. **Do not use Material Icons.** Use PrimeIcons (`pi pi-*`) exclusively.
3. **Do not use scoped styles for design.** Scoped styles are only admissible for structural layout CSS properties: `display` (`flex`/`grid`), `padding`, `margin`, positioning.
4. **Use PrimeVue Design-Tokens to style PrimeVue components.** Each token maps to a CSS variable — use those CSS variables.
5. **Do not use `:deep()`.** Use PassThrough API instead.
6. **Do not modify PrimeVue components via scoped styles.**
7. **Global CSS classes** (used in multiple components) go in the central SCSS files. **Component-local CSS classes** go in `<style scoped>` of that component.

### TypeScript in Vue
- Use `<script lang="ts">` (or `<script setup lang="ts">`) on all components.
- New components must use the **Composition API** (`<script setup>`).
- Existing Options API components do not need to be migrated.

### URL / Route Naming
- View pages follow REST resource naming: `/companies/:companyId/frameworks/:frameworkName`
- Upload pages add a `/upload` suffix: `/companies/:companyId/frameworks/:frameworkName/upload`

### Framework Data View Page
- Display logic is **centralised** — one rule per field type applies to all frameworks.
- Do **not** add framework-specific display customisations.
- Raw data must be formatted **once** at the start of the processing chain and not manipulated again before reaching the display component.