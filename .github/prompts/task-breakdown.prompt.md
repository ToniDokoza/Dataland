# Task Breakdown: Frameworks Explorer Tab

This document breaks the Frameworks Explorer feature into small, sequential implementation tasks.
Each task touches **one file** or **one concern** only. Complete and verify each task before
starting the next one.

---

## Task 1 — Add TypeScript types for the specification service API responses

**File**: `src/types/SpecificationTypes.ts` *(create)*

Define TypeScript interfaces that match the OpenAPI schemas from
`dataland-specification-service/specificationServiceOpenApi.json`:

```ts
export interface IdWithRef {
  id: string;
  ref: string;
}

export interface SimpleFrameworkSpecification {
  framework: IdWithRef;
  name: string;
}

export interface FrameworkSpecification {
  framework: IdWithRef;
  name: string;
  businessDefinition: string;
  schema: string;                    // raw JSON string — parse before use
  referencedReportJsonPath?: string;
}

// schema parsed shape: Record<string, Record<string, Record<string, string>>>
export type FrameworkSchema = Record<string, Record<string, Record<string, string>>>;

export interface DataPointTypeSpecification {
  dataPointType: IdWithRef;
  name: string;
  businessDefinition: string;
  dataPointBaseType: IdWithRef;
  usedBy: IdWithRef[];
  constraints?: string[];
}
```

**Acceptance**: File compiles with no TypeScript errors. No other files changed.

---

## Task 2 — Add the specification service client to `ApiClientProvider`

**File**: `src/services/ApiClients.ts` *(modify)*

The specification service is served at `/specifications`. Since there is no auto-generated
client yet, create a thin hand-crafted client class **in the same file** (or in a new file
`src/services/SpecificationClient.ts` that is imported).

The client must implement:

```ts
class SpecificationClient {
  constructor(private readonly axiosInstance: AxiosInstance) {}

  async listFrameworks(): Promise<SimpleFrameworkSpecification[]>
  async getFramework(id: string): Promise<FrameworkSpecification>
  async getDataPointType(id: string): Promise<DataPointTypeSpecification>
}
```

Add it to the `ApiClients` interface and `constructApiClients()` factory.

**Acceptance**: `apiClientProvider.apiClients.specificationClient` is accessible.
TypeScript compiles with no errors.

---

## Task 3 — Create `DataPointDetailPanel.vue`

**File**: `src/components/resources/frameworksExplorer/DataPointDetailPanel.vue` *(create)*

A presentational component that receives a `DataPointTypeSpecification | null` prop and renders:
- Placeholder text when prop is `null`.
- Name, business definition, base type ID, constraints list, and "used by" framework badges otherwise.

Props:
```ts
defineProps<{ dataPoint: DataPointTypeSpecification | null }>()
```

Use `data-test="data-point-detail-panel"` on the root element.

**Acceptance**: Component renders in isolation with both `null` and a sample data point object.

---

## Task 4 — Create `FrameworkSchemaTree.vue`

**File**: `src/components/resources/frameworksExplorer/FrameworkSchemaTree.vue` *(create)*

Receives the parsed `FrameworkSchema` and emits `select-data-point` with the selected
`dataPointTypeId: string` when a leaf row is clicked.

Props:
```ts
defineProps<{ schema: FrameworkSchema }>()
defineEmits<{ 'select-data-point': [dataPointTypeId: string] }>()
```

Features:
- A PrimeVue `InputText` search field at the top that filters visible data point keys in real time.
- Categories and subcategories collapsible via a toggle (use PrimeVue `Panel` or manual `v-show`).
- Each data point key row shows the `camelCase` key name as label.
- `data-test` attributes: `schema-search-input`, `schema-tree`, `data-point-item-{dataPointKey}`.

**Acceptance**: Component renders a schema tree, filters by search text, and emits the correct
event on click.

---

## Task 5 — Create `FrameworkSidebar.vue`

**File**: `src/components/resources/frameworksExplorer/FrameworkSidebar.vue` *(create)*

Receives an array of `SimpleFrameworkSpecification` and the currently selected framework ID.
Emits `select-framework` with the `frameworkId: string` when an item is clicked.

Props:
```ts
defineProps<{
  frameworks: SimpleFrameworkSpecification[];
  selectedFrameworkId: string | null;
}>()
defineEmits<{ 'select-framework': [frameworkId: string] }>()
```

Use `data-test="framework-list"` on the list root and `data-test="framework-item-{id}"` per item.
Highlight the active item with a CSS class (e.g. PrimeFlex bg or custom scoped style).

**Acceptance**: Component renders a list, highlights the selected item, and emits on click.

---

## Task 6 — Create `FrameworksExplorerPage.vue`

**File**: `src/components/pages/FrameworksExplorerPage.vue` *(create)*

The main page component. Composes the three child components and orchestrates data loading.

Behavior:
1. On mount, fetch `specificationClient.listFrameworks()` and store in `frameworks` ref.
2. When a framework is selected in the sidebar, fetch
   `specificationClient.getFramework(id)`, parse `spec.schema` with `JSON.parse()`, and store
   in `selectedSchema`.
3. When a data point key is selected in the schema tree, fetch
   `specificationClient.getDataPointType(dataPointTypeId)` and store in `selectedDataPoint`.
4. Show `DatalandProgressSpinner` while any fetch is in-flight.

Layout (PrimeFlex grid, three columns):
```
<TheContent>
  <div class="flex h-full">
    <FrameworkSidebar class="w-3" … />
    <FrameworkSchemaTree class="w-6" … />
    <DataPointDetailPanel class="w-3" … />
  </div>
</TheContent>
```

Uses `<script setup lang="ts">`.

**Acceptance**: Page loads frameworks on mount, renders schema on selection, shows data point
details on leaf click.

---

## Task 7 — Register the route in `router/index.ts`

**File**: `src/router/index.ts` *(modify)*

Add a lazy-loaded route for the Frameworks Explorer page:

```ts
const FrameworksExplorerPage = (): Promise<RouteComponent> =>
  import('@/components/pages/FrameworksExplorerPage.vue');
```

And a route entry:
```ts
{
  path: '/frameworks',
  name: 'Frameworks Explorer',
  component: FrameworksExplorerPage,
  meta: {
    initialTabId: 'frameworks',
    requiresAuthentication: false,
  },
},
```

**Acceptance**: Navigating to `/frameworks` in the browser loads the page without a 404.

---

## Task 8 — Add the FRAMEWORKS tab to `DatasetsTabMenu.vue`

**File**: `src/components/general/DatasetsTabMenu.vue` *(modify)*

Insert the new tab entry into the `tabs` array. Place it logically — e.g. after the COMPANIES
tab:

```ts
{ id: 'frameworks', label: 'FRAMEWORKS', route: '/frameworks', isVisible: true },
```

**Acceptance**: The FRAMEWORKS tab appears in the navigation bar for all users (authenticated
and unauthenticated). Clicking it navigates to `/frameworks` and keeps the tab highlighted.

---

## Suggested Implementation Order

```
Task 1 → Task 2 → Task 3 → Task 4 → Task 5 → Task 6 → Task 7 → Task 8
```

Each task builds on the previous ones. Tasks 3–5 can be developed in parallel if preferred —
they are independent sibling components that are only composed together in Task 6.
