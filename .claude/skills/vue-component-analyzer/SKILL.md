---
name: vue-component-analyzer
description: Analyzes Vue 3 component structure and suggests optimizations for performance and code reuse. Use this skill when reviewing Vue components for performance issues, duplicated logic, or refactoring opportunities.
---

# Vue Component Analyzer

This skill provides a structured approach for analyzing Vue 3 components in the Factory Inventory Management System, identifying performance bottlenecks, and recommending code reuse improvements.

## When to Use

- Reviewing a Vue component for performance issues
- Looking for duplicated patterns across components
- Evaluating whether to extract composables or shared components
- Before refactoring frontend code

## Analysis Process

### Step 1: Component Structure Audit

For each `.vue` file, evaluate:

**Script Section (`<script setup>`)**
- Are reactive references (`ref`, `reactive`) used appropriately?
- Are expensive computations wrapped in `computed()` instead of methods called in templates?
- Are side effects properly handled in `watch` / `watchEffect` with cleanup?
- Are API calls deduplicated (not re-fetched when data hasn't changed)?

**Template Section**
- Are `v-for` loops using stable, unique `:key` bindings (never index)?
- Are there inline complex expressions that should be computed properties?
- Are large lists rendered without virtualization?
- Are conditional blocks (`v-if` / `v-show`) used correctly? (`v-show` for frequent toggles, `v-if` for rare)

**Style Section**
- Are styles scoped to avoid leaking?
- Is there duplicated CSS across components that could be shared?

### Step 2: Cross-Component Pattern Detection

Scan all views and components for repeated patterns:

```bash
# Find duplicated loading/error state patterns
grep -l "loading.value = true" client/src/views/*.vue client/src/components/*.vue

# Find duplicated API call patterns
grep -l "api\." client/src/views/*.vue

# Find duplicated filter logic
grep -l "useFilters" client/src/views/*.vue

# Find repeated computed property patterns
grep -c "computed(" client/src/views/*.vue | sort -t: -k2 -rn

# Find duplicated template patterns (modals, tables, cards)
grep -l "class=\"modal" client/src/components/*.vue client/src/views/*.vue

# Find inline styles or repeated CSS blocks
grep -rn "style=" client/src/views/*.vue client/src/components/*.vue
```

### Step 3: Performance Checks

#### Reactivity Issues
```javascript
// BAD: Unnecessary reactivity on static data
const columns = ref(['Name', 'SKU', 'Quantity'])  // Never changes

// GOOD: Plain constant
const columns = ['Name', 'SKU', 'Quantity']
```

#### Missing Computed Caching
```javascript
// BAD: Recalculates on every render
const getTotal = () => items.value.reduce((s, i) => s + i.price, 0)
// Used in template as {{ getTotal() }}

// GOOD: Cached, only recalculates when items change
const total = computed(() => items.value.reduce((s, i) => s + i.price, 0))
// Used in template as {{ total }}
```

#### Expensive Watch Without Throttle
```javascript
// BAD: Fires API call on every keystroke
watch(searchQuery, async (val) => {
  const res = await api.search(val)
  results.value = res.data
})

// GOOD: Debounced
import { watchDebounced } from '@vueuse/core'  // or manual debounce
watchDebounced(searchQuery, async (val) => {
  const res = await api.search(val)
  results.value = res.data
}, { debounce: 300 })
```

#### Unnecessary Re-renders from Object/Array Props
```javascript
// BAD: Creates new array reference each render
<ChildComponent :items="items.filter(i => i.active)" />

// GOOD: Stable computed reference
const activeItems = computed(() => items.value.filter(i => i.active))
<ChildComponent :items="activeItems" />
```

### Step 4: Code Reuse Opportunities

#### Extract Composable When
- 2+ components share the same reactive state + logic pattern
- A component has complex logic that obscures the template relationship

**Composable template:**
```javascript
// client/src/composables/useDataLoader.js
import { ref } from 'vue'

export function useDataLoader(apiFn) {
  const data = ref([])
  const loading = ref(false)
  const error = ref(null)

  const load = async (params = {}) => {
    loading.value = true
    error.value = null
    try {
      const response = await apiFn(params)
      data.value = response.data
    } catch (err) {
      error.value = 'Failed to load data'
      console.error(err)
    } finally {
      loading.value = false
    }
  }

  return { data, loading, error, load }
}
```

#### Extract Shared Component When
- 2+ components render the same UI structure with different data
- Modal, table, card, or chart patterns are duplicated

**Common candidates in this project:**
- Modal components (BacklogDetailModal, CostDetailModal, etc.) may share layout
- Loading/error state display blocks
- Data table rendering with sort/filter
- Chart SVG patterns

### Step 5: Report Format

Structure findings as:

```
## Performance Issues
| File | Line | Issue | Impact | Fix |
|------|------|-------|--------|-----|
| Dashboard.vue | 45 | Method call in template | Re-runs every render | Convert to computed |

## Code Reuse Opportunities
| Pattern | Files | Recommendation |
|---------|-------|----------------|
| Loading/error state | Dashboard, Orders, Inventory | Extract useDataLoader composable |

## Quick Wins (low effort, high impact)
1. ...

## Larger Refactors (consider for future)
1. ...
```

## Project-Specific Checks

### Filter System
- Verify views use `useFilters` composable instead of local filter state
- Check that `getCurrentFilters()` is called consistently before API requests
- Ensure inventory views do NOT apply month filters (no time dimension)

### Data Flow
- API responses should be stored in `ref()`, derived data in `computed()`
- Avoid storing filtered data separately when it can be computed from raw data
- Check that watchers on filters trigger data reload appropriately

### Modal Components
- Modals should accept data via props and emit close events
- Check for prop mutation (common bug)
- Verify modals don't fetch their own data if the parent already has it

### Chart Components
- SVG charts should use computed properties for coordinates/dimensions
- Verify chart data is derived, not duplicated from source data
- Check that chart re-renders are driven by reactive data changes, not manual DOM manipulation

## Running the Analysis

```bash
# Quick scan: find largest components (likely candidates for splitting)
wc -l client/src/views/*.vue client/src/components/*.vue | sort -rn | head -20

# Find components without scoped styles
grep -L "scoped" client/src/views/*.vue client/src/components/*.vue

# Find potential prop mutations
grep -n "props\." client/src/components/*.vue | grep -v "// "

# Find v-for without proper keys
grep -B1 "v-for" client/src/views/*.vue client/src/components/*.vue | grep -v ":key"

# Count reactive primitives per file (complexity indicator)
for f in client/src/views/*.vue; do
  echo "$(grep -c 'ref(' "$f") refs - $(basename "$f")"
done | sort -rn
```
