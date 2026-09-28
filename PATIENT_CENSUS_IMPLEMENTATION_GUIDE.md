# Patient Census Feature - Implementation Guide

## Executive Summary

This guide provides a step-by-step implementation path for the Patient Census feature, following CareFlow's established patterns and architecture.

**Total Estimated Time**: 8-12 hours
**Difficulty**: Medium
**Complexity**: High (comprehensive feature with tables, filtering, search, and detail views)

**Key Success Factors**:
1. Follow patterns from Dashboard feature exactly
2. Zod schemas drive all types (single source of truth)
3. TanStack Query handles all server state
4. URL params handle filter persistence
5. MSW provides realistic mock data
6. Accessibility and testing integrated from start

---

## Phase 1: Core Setup (1-2 hours)

### Step 1.1: Create Directory Structure

```bash
cd /Users/carlosfelipecruzpachon/Desktop/Health-project/careflow

# Create directories
mkdir -p src/features/patients/api
mkdir -p src/features/patients/components/__tests__
mkdir -p src/features/patients/hooks/__tests__
mkdir -p src/features/patients/schemas/__tests__
mkdir -p src/features/patients/types

# Create mock directories
mkdir -p src/mocks/data
mkdir -p src/mocks/handlers
```

### Step 1.2: Create Zod Schemas

**File**: `src/features/patients/schemas/patient.schema.ts`

Start with the patient.schema.ts from PATIENT_CENSUS_QUICK_REFERENCE.md. This is the foundation for all types.

**Key Points**:
- Enums for status and admission type
- VitalSigns object with realistic ranges
- Medication object with proper validation
- Complete patient object with all fields
- Keep it comprehensive but realistic

**Testing**: Create basic schema validation tests

```bash
# Create test file
touch src/features/patients/schemas/__tests__/patient.schema.test.ts
```

### Step 1.3: Create Filter & Search Schemas

**Files**:
- `src/features/patients/schemas/patientFilters.schema.ts`
- `src/features/patients/schemas/patientSearch.schema.ts`

Use templates from Quick Reference. These define what parameters the API accepts.

**Key Points**:
- Filters schema maps to URL query parameters
- Search schema defines searchable fields
- Validate page/pageSize ranges
- Provide sensible defaults

### Step 1.4: Create Type Barrel Export

**File**: `src/features/patients/types/patient.types.ts`

```typescript
/**
 * Patient Types
 *
 * All types inferred from Zod schemas.
 */

export type { Patient, PatientList, PatientStatus, AdmissionType, VitalSigns, Medication } from '../schemas/patient.schema';
export type { PatientFilters } from '../schemas/patientFilters.schema';
export type { PatientSearch } from '../schemas/patientSearch.schema';

export interface PatientDetailState {
  isOpen: boolean;
  patientId: string | null;
}

export interface PatientTableState {
  selectedRowIds: Record<string, boolean>;
  columnVisibility: Record<string, boolean>;
}
```

### Step 1.5: Update Query Keys

**File**: `src/lib/constants/queryKeys.ts`

Add to existing queryKeys object:

```typescript
patients: {
  all: ['patients'] as const,
  lists: () => [...queryKeys.patients.all, 'list'] as const,
  list: (filters?: Record<string, unknown>) =>
    [...queryKeys.patients.lists(), filters] as const,
  details: () => [...queryKeys.patients.all, 'detail'] as const,
  detail: (id: string) => [...queryKeys.patients.details(), id] as const,
},
```

---

## Phase 2: API Layer (1-2 hours)

### Step 2.1: Create API Module

**File**: `src/features/patients/api/patientsApi.ts`

Use template from Quick Reference. Key points:

```typescript
interface PatientListResponse {
  data: Patient[];
  pagination: {
    page: number;
    pageSize: number;
    total: number;
    totalPages: number;
    hasNextPage: boolean;
    hasPrevPage: boolean;
  };
  metadata: {
    timestamp: string;
    count: number;
  };
}

export const patientsApi = {
  getPatients: async (filters?: PatientFilters): Promise<PatientListResponse> => {
    // Build params from filters
    // Call apiClient.get('/patients', params)
    // Validate with patientListSchema
    // Return response
  },

  getPatientDetail: async (patientId: string): Promise<Patient> => {
    // Call apiClient.get(`/patients/${patientId}`)
    // Validate with patientSchema
    // Return patient
  },
};
```

**Testing**: Test with mock data and schema validation

### Step 2.2: Create Mock Data Generators

**File**: `src/mocks/data/patients.mock.ts`

Use template from Quick Reference. Create:

1. Helper functions (random, randomFloat, getTimestamp)
2. Name generation (firstNames, lastNames arrays)
3. Vital signs generator
4. Medications generator
5. Single patient generator
6. Batch patient generator (75 patients default)

**Key Points**:
- Generate realistic data (valid UUIDs, dates, ranges)
- Varied vital signs to test status colors
- Mix of critical, stable, observation statuses
- Different departments and bed numbers
- Keep randomness but deterministic when needed

### Step 2.3: Create MSW Handlers

**File**: `src/mocks/handlers/patients.handlers.ts`

Create handlers for:

1. `http.get('/api/patients')` - List with filters/sort/pagination
2. `http.get('/api/patients/:id')` - Detail view

Use template from Quick Reference. Key points:

```typescript
const mockPatients = generateMockPatients(75);

export const patientsHandlers = [
  http.get('/api/patients', async ({ request }) => {
    await delay(400); // Simulate network

    // Parse query params
    // Filter mock data
    // Sort
    // Paginate
    // Return response
  }),

  http.get('/api/patients/:id', async ({ params }) => {
    await delay(300);
    // Find patient or 404
    // Return response
  }),
];
```

### Step 2.4: Register Handlers

**File**: `src/mocks/handlers/index.ts`

```typescript
import { dashboardHandlers } from './dashboard.handlers';
import { patientsHandlers } from './patients.handlers';

export const handlers = [...dashboardHandlers, ...patientsHandlers];
```

**Test**: Run app and verify MSW intercepts requests

---

## Phase 3: Hooks (1-2 hours)

### Step 3.1: Create usePatients Hook

**File**: `src/features/patients/hooks/usePatients.ts`

```typescript
export function usePatients(filters?: PatientFilters) {
  return useQuery({
    queryKey: queryKeys.patients.list(filters),
    queryFn: () => patientsApi.getPatients(filters),
    staleTime: 30000,
    gcTime: 300000,
    refetchOnWindowFocus: true,
    refetchOnMount: true,
  });
}
```

**Testing**: Create hook test verifying:
- Fetches data on mount
- Updates queryKey when filters change
- Handles errors
- Refetches on window focus

### Step 3.2: Create usePatientDetail Hook

**File**: `src/features/patients/hooks/usePatientDetail.ts`

```typescript
export function usePatientDetail(patientId: string | null) {
  return useQuery({
    queryKey: queryKeys.patients.detail(patientId || ''),
    queryFn: () => patientsApi.getPatientDetail(patientId!),
    staleTime: 15000,
    gcTime: 300000,
    enabled: !!patientId,
    refetchOnWindowFocus: true,
  });
}
```

**Testing**: Test:
- Doesn't fetch when patientId is null
- Fetches when patientId provided
- Caches different patients separately

### Step 3.3: Create usePatientFilters Hook

**File**: `src/features/patients/hooks/usePatientFilters.ts`

This hook manages URL state. Key points:

```typescript
export function usePatientFilters() {
  const [searchParams, setSearchParams] = useSearchParams();

  // Parse URL params into filters object
  const filters: PatientFilters = {
    status: searchParams.get('status')?.split(',') || undefined,
    page: parseInt(searchParams.get('page') || '1', 10),
    // ... other filters
  };

  // Function to update filters and URL
  const setFilters = useCallback((newFilters: Partial<PatientFilters>) => {
    const updated = { ...filters, ...newFilters };
    const validated = patientFiltersSchema.parse(updated);

    const params = new URLSearchParams();
    // Build params from validated filters

    setSearchParams(params);
  }, [filters, setSearchParams]);

  return { filters, setFilters, resetFilters };
}
```

**Testing**: Test:
- Reads filters from URL
- Updates URL on setFilters
- Resets all on resetFilters
- Persists across navigation

### Step 3.4: Create usePatientSearch Hook

**File**: `src/features/patients/hooks/usePatientSearch.ts`

Debounced search input. Key points:

```typescript
export function usePatientSearch(onSearch: (query: string) => void, debounceMs = 300) {
  const [searchTerm, setSearchTerm] = useState('');
  const debounceTimerRef = useRef<NodeJS.Timeout>();

  useEffect(() => {
    if (debounceTimerRef.current) clearTimeout(debounceTimerRef.current);

    debounceTimerRef.current = setTimeout(() => {
      onSearch(searchTerm);
    }, debounceMs);

    return () => {
      if (debounceTimerRef.current) clearTimeout(debounceTimerRef.current);
    };
  }, [searchTerm, onSearch, debounceMs]);

  return { searchTerm, handleSearchChange, clearSearch };
}
```

**Testing**: Test debouncing and callback invocation

---

## Phase 4: Components (2-3 hours)

### Step 4.1: Create PatientsPage (Orchestrator)

**File**: `src/features/patients/components/PatientsPage.tsx`

This is the main container. Structure:

```typescript
export function PatientsPage() {
  // Get state from hooks
  const { filters, setFilters, resetFilters } = usePatientFilters();
  const { data, isLoading, error, refetch } = usePatients(filters);
  const { searchTerm, handleSearchChange, clearSearch } = usePatientSearch(
    (query) => setFilters({ ...filters, query, page: 1 }),
  );

  // Local state for drawer
  const [detailDrawer, setDetailDrawer] = useState<PatientDetailState>({
    isOpen: false,
    patientId: null,
  });

  // Conditional rendering
  if (isLoading) return <PatientSkeleton />;
  if (error) return <PatientsError error={error} onRetry={refetch} />;

  return (
    <main className="flex h-full gap-6 p-6">
      {/* Filter sidebar */}
      <PatientFilters
        filters={filters}
        onFiltersChange={setFilters}
        onResetFilters={resetFilters}
      />

      {/* Main content */}
      <div className="flex-1 flex flex-col gap-4">
        {/* Search */}
        <PatientSearch value={searchTerm} onChange={handleSearchChange} />

        {/* Table or empty state */}
        {data?.data.length ? (
          <>
            <PatientTable
              data={data.data}
              onRowClick={(p) => setDetailDrawer({ isOpen: true, patientId: p.id })}
            />
            <PatientPagination />
          </>
        ) : (
          <EmptyState />
        )}
      </div>

      {/* Detail drawer */}
      <PatientDetailDrawer
        isOpen={detailDrawer.isOpen}
        patientId={detailDrawer.patientId}
        onClose={() => setDetailDrawer({ isOpen: false, patientId: null })}
      />
    </main>
  );
}
```

**Key Points**:
- Clean separation of concerns
- Each section independently manageable
- State flows down, events bubble up
- Handles all loading/error/empty states

### Step 4.2: Create PatientTable

**File**: `src/features/patients/components/PatientTable.tsx`

Use TanStack Table. Key steps:

1. Define columns using `createColumnHelper`
2. Create table instance with `useReactTable`
3. Render table HTML structure
4. Handle sorting via header clicks
5. Support row selection

**Column Types**:
- MRN (identifier)
- Name (primary display)
- Age
- Status (badge with color)
- Department
- Bed
- LOS (formatted)
- Vitals summary
- Physician
- Admission type
- Actions (expand button)

**Testing**: Test:
- Renders all columns
- Sorts on header click
- Status colors correct
- Row click callbacks fire

### Step 4.3: Create PatientFilters

**File**: `src/features/patients/components/PatientFilters.tsx`

Create sidebar with controls:

```typescript
export function PatientFilters({
  filters,
  onFiltersChange,
  onResetFilters,
}: PatientFiltersProps) {
  return (
    <aside className="w-64 bg-gray-50 rounded-lg p-4 space-y-4">
      {/* Status Filter */}
      <div>
        <label className="block text-sm font-medium mb-2">Status</label>
        <MultiSelect
          values={filters.status}
          options={['critical', 'stable', 'observation', 'recovery', 'pending-discharge']}
          onChange={(values) => onFiltersChange({ status: values })}
        />
      </div>

      {/* Department Filter */}
      <div>
        <label className="block text-sm font-medium mb-2">Department</label>
        <MultiSelect
          values={filters.departmentIds}
          options={departments} // From API
          onChange={(values) => onFiltersChange({ departmentIds: values })}
        />
      </div>

      {/* Date Range */}
      <div>
        <label className="block text-sm font-medium mb-2">Admitted Date</label>
        <DateRangePicker
          from={filters.admittedAfter}
          to={filters.admittedBefore}
          onChange={(from, to) => onFiltersChange({ admittedAfter: from, admittedBefore: to })}
        />
      </div>

      {/* Critical Vitals */}
      <div className="flex items-center gap-2">
        <input
          type="checkbox"
          checked={filters.criticalVitals || false}
          onChange={(e) => onFiltersChange({ criticalVitals: e.target.checked })}
        />
        <label>Show only critical vitals</label>
      </div>

      {/* Reset */}
      <button
        onClick={onResetFilters}
        className="w-full px-4 py-2 border rounded-md hover:bg-gray-100"
      >
        Reset Filters
      </button>
    </aside>
  );
}
```

### Step 4.4: Create PatientSearch

**File**: `src/features/patients/components/PatientSearch.tsx`

Simple search input with clear button:

```typescript
export function PatientSearch({
  value,
  onChange,
  onClear,
}: PatientSearchProps) {
  return (
    <div className="flex items-center gap-2 bg-white px-4 py-2 rounded-lg border">
      <Search className="w-4 h-4 text-gray-400" />
      <input
        type="text"
        placeholder="Search by name, MRN, diagnosis, or bed..."
        value={value}
        onChange={(e) => onChange(e.target.value)}
        className="flex-1 outline-none"
      />
      {value && (
        <button onClick={onClear} aria-label="Clear search">
          <X className="w-4 h-4 text-gray-400" />
        </button>
      )}
    </div>
  );
}
```

### Step 4.5: Create PatientDetailDrawer

**File**: `src/features/patients/components/PatientDetailDrawer.tsx`

Slide-out panel with patient details:

```typescript
export function PatientDetailDrawer({
  isOpen,
  patientId,
  onClose,
}: PatientDetailDrawerProps) {
  const { data: patient, isLoading } = usePatientDetail(patientId);
  const drawerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (isOpen) {
      drawerRef.current?.focus();
      document.body.style.overflow = 'hidden';
    } else {
      document.body.style.overflow = 'unset';
    }
  }, [isOpen]);

  useEffect(() => {
    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    if (isOpen) document.addEventListener('keydown', handleEscape);
    return () => document.removeEventListener('keydown', handleEscape);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return (
    <>
      {/* Backdrop */}
      <div
        className="fixed inset-0 bg-black bg-opacity-50"
        onClick={onClose}
      />

      {/* Drawer */}
      <div
        ref={drawerRef}
        className="fixed right-0 top-0 bottom-0 w-96 bg-white shadow-xl overflow-y-auto"
        role="complementary"
        aria-label="Patient details"
        tabIndex={-1}
      >
        <div className="p-6 space-y-6">
          {/* Header */}
          <div className="flex items-center justify-between">
            <h2 className="text-xl font-bold">Patient Details</h2>
            <button onClick={onClose} aria-label="Close drawer">
              <X className="w-5 h-5" />
            </button>
          </div>

          {isLoading ? (
            <div>Loading...</div>
          ) : patient ? (
            <>
              {/* Patient info sections */}
              <PatientDemographics patient={patient} />
              <PatientVitals patient={patient} />
              <PatientMedications patient={patient} />
              <PatientCareTeam patient={patient} />
            </>
          ) : (
            <div>Patient not found</div>
          )}

          {/* Actions */}
          <div className="flex gap-2 border-t pt-4">
            <button className="flex-1 px-4 py-2 bg-primary-600 text-white rounded-md">
              Discharge
            </button>
            <button className="flex-1 px-4 py-2 border rounded-md">
              Transfer
            </button>
          </div>
        </div>
      </div>
    </>
  );
}
```

### Step 4.6: Create Supporting Components

Create simple components:

- **PatientRow.tsx**: Wrapper with hover effects
- **PatientCell.tsx**: Individual cell with formatting
- **PatientSkeleton.tsx**: Loading placeholder
- **PatientsError.tsx**: Error state UI
- **PatientPagination.tsx**: Pagination controls

---

## Phase 5: Testing (1-2 hours)

### Step 5.1: Unit Tests for Hooks

Create `src/features/patients/hooks/__tests__/`:

```typescript
// usePatients.test.ts
describe('usePatients', () => {
  it('fetches patients successfully', async () => {
    // Setup
    // Render hook
    // Assert loading state
    // Wait for data
    // Assert data loaded
  });

  it('updates when filters change', async () => {
    // Render with initial filters
    // Change filters
    // Assert refetch called
  });
});
```

Target: 8-10 tests per hook

### Step 5.2: Component Tests

Create `src/features/patients/components/__tests__/`:

```typescript
// PatientsPage.test.tsx
describe('PatientsPage', () => {
  it('renders patient table', async () => {
    // Render component
    // Assert table found
    // Assert rows rendered
  });

  it('opens drawer on row click', async () => {
    // Render
    // Click row
    // Assert drawer opens
  });
});
```

Target: 8-10 tests per major component

### Step 5.3: Accessibility Tests

```typescript
// patients-accessibility.test.tsx
describe('Accessibility', () => {
  it('has no violations', async () => {
    // Render
    // Run axe
    // Assert no violations
  });

  it('supports keyboard navigation', async () => {
    // Test Tab through elements
    // Test Enter/Space on buttons
    // Test Escape on drawer
  });
});
```

### Step 5.4: Schema Tests

```typescript
// patient.schema.test.ts
describe('patient schema', () => {
  it('validates complete patient', () => {
    const data = generateMockPatient();
    expect(() => patientSchema.parse(data)).not.toThrow();
  });

  it('rejects missing fields', () => {
    const invalid = { firstName: 'John' };
    expect(() => patientSchema.parse(invalid)).toThrow();
  });
});
```

---

## Phase 6: Polish & Accessibility (1-2 hours)

### Step 6.1: Semantic HTML Review

Ensure:
- `<main>` wraps main content
- `<section>` wraps major sections
- `<table>` used for tabular data
- `<button>` for all buttons
- `<header>` for headers
- `<nav>` for navigation (if applicable)

### Step 6.2: Keyboard Navigation

Test:
- Tab through all interactive elements
- Enter/Space to activate
- Escape to close drawer
- Arrow keys in selects
- Logical focus order

### Step 6.3: Screen Reader Support

Add:
- `aria-label` on icon buttons
- `aria-labelledby` on sections
- `aria-live="polite"` on dynamic content
- `role="table"` on tables
- `role="complementary"` on drawer

### Step 6.4: Color Contrast

Verify:
- Text ≥4.5:1 against background
- Status badges ≥3:1
- Interactive elements have clear focus
- Not relying on color alone for status

### Step 6.5: Performance Optimization

Add:
- `memo()` to components to prevent re-renders
- `useCallback()` for event handlers
- Virtualization if table >50 rows
- Lazy load detail drawer
- Optimize images

---

## Phase 7: Integration & Routing (30 min)

### Step 7.1: Create Feature Barrel Export

**File**: `src/features/patients/index.ts`

```typescript
export * from './types';
export { patientsApi } from './api/patientsApi';
export {
  usePatients,
  usePatientDetail,
  usePatientFilters,
  usePatientSearch,
} from './hooks';
export { PatientsPage } from './components/PatientsPage';
```

### Step 7.2: Add Route (if not already added)

**File**: `src/router.tsx`

```typescript
import { PatientsPage } from '@/features/patients';

const routes = [
  {
    path: '/patients',
    element: <PatientsPage />,
  },
  // ... other routes
];
```

### Step 7.3: Update Navigation

If needed, add link to patients page in navigation/sidebar

---

## Verification Checklist

### Before Submission

- [ ] All TypeScript compiles without errors
- [ ] Biome linting passes (`npm run lint`)
- [ ] All tests pass (`npm test -- src/features/patients`)
- [ ] No console errors in browser
- [ ] MSW intercepts all requests correctly
- [ ] Loading state works (add delay in handler if needed)
- [ ] Error state works (can test by breaking handler)
- [ ] Empty state shows when no results
- [ ] Filters persist in URL
- [ ] Search debounces correctly (300ms)
- [ ] Detail drawer opens/closes with data
- [ ] Table sorts on header click
- [ ] Pagination works (next/prev/page size)
- [ ] Keyboard navigation works (Tab, Enter, Escape)
- [ ] Screen reader announces content (test with VoiceOver/NVDA)
- [ ] Color contrast acceptable (Lighthouse or WAVE)
- [ ] Responsive on mobile (check with DevTools)
- [ ] Performance acceptable (<3s initial load)
- [ ] No accessibility violations (axe-core)
- [ ] No memory leaks (check DevTools)

---

## Common Issues & Solutions

### Issue: Components re-render too often
**Solution**: Wrap with `memo()`, use `useCallback()` for handlers

### Issue: API calls happen multiple times
**Solution**: Check React strict mode, verify hook dependencies, check TanStack Query queryKey

### Issue: Filters not persisting
**Solution**: Verify useSearchParams is correctly updating URL in usePatientFilters

### Issue: Detail drawer not focusing
**Solution**: Add `useEffect` to set focus on drawerRef when isOpen changes

### Issue: Search not debouncing
**Solution**: Check debounceMs value and useEffect cleanup in usePatientSearch

### Issue: Table virtualization needed
**Solution**: Add `@tanstack/react-table-windowed-rows` or use `react-window`

### Issue: Tests failing with MSW
**Solution**: Verify handlers are registered in `/src/mocks/handlers/index.ts`

### Issue: TypeScript errors on Patient type
**Solution**: Ensure types are exported from `src/features/patients/types/patient.types.ts`

---

## Next Steps After Implementation

Once basic feature is complete:

1. **Advanced Filtering**
   - Add vitals range filters
   - Add stay duration filters
   - Save filter presets

2. **Bulk Operations**
   - Select multiple patients
   - Bulk discharge/transfer
   - Bulk status updates

3. **Exports**
   - Export to CSV
   - Export to PDF
   - Print list

4. **Real-time Updates**
   - WebSocket integration
   - Auto-refresh patients
   - Live vital signs updates

5. **Advanced Search**
   - Full-text search
   - Search history
   - Saved searches

6. **Analytics**
   - Patient demographics charts
   - Admit/discharge trends
   - Average LOS by department

7. **Integrations**
   - EHR system sync
   - Billing system sync
   - Alert integrations

---

## Files Reference

### Main Documents
- `/PATIENT_CENSUS_ARCHITECTURE.md` - Complete specification
- `/PATIENT_CENSUS_DIAGRAMS.md` - Visual diagrams
- `/PATIENT_CENSUS_QUICK_REFERENCE.md` - Copy-paste templates
- `/PATIENT_CENSUS_IMPLEMENTATION_GUIDE.md` - This file

### To Create
```
src/features/patients/
├── api/patientsApi.ts
├── components/
│   ├── PatientCell.tsx
│   ├── PatientDetailDrawer.tsx
│   ├── PatientFilters.tsx
│   ├── PatientRow.tsx
│   ├── PatientSearch.tsx
│   ├── PatientSkeleton.tsx
│   ├── PatientsError.tsx
│   ├── PatientsPage.tsx
│   ├── PatientTable.tsx
│   └── __tests__/ (10+ test files)
├── hooks/
│   ├── usePatientDetail.ts
│   ├── usePatientFilters.ts
│   ├── usePatients.ts
│   ├── usePatientSearch.ts
│   └── __tests__/ (4+ test files)
├── schemas/
│   ├── patient.schema.ts
│   ├── patientFilters.schema.ts
│   ├── patientSearch.schema.ts
│   └── __tests__/ (3+ test files)
├── types/
│   └── patient.types.ts
└── index.ts

src/mocks/
├── data/patients.mock.ts
└── handlers/patients.handlers.ts
```

---

## Key Principles to Remember

1. **Single Source of Truth**: Zod schemas define everything
2. **URL is State**: Filters live in URL params
3. **Server State Management**: TanStack Query owns data
4. **Clean Component Boundaries**: Orchestrator → Presentational
5. **Accessibility First**: WCAG compliance built-in
6. **Testing Throughout**: Unit → Component → E2E
7. **MSW for All APIs**: No real backend needed
8. **Follow Patterns**: Match Dashboard implementation exactly

---

## Estimated Timeline

| Phase | Task | Time |
|-------|------|------|
| 1 | Setup + Schemas | 1-2h |
| 2 | API + Mock Data | 1-2h |
| 3 | Hooks | 1-2h |
| 4 | Components | 2-3h |
| 5 | Testing | 1-2h |
| 6 | Polish + A11y | 1-2h |
| 7 | Integration | 30m |
| **Total** | | **8-12h** |

---

## Success Criteria

Feature is complete when:
- ✅ TypeScript compiles with no errors
- ✅ Biome linting passes
- ✅ All tests passing (>80% coverage on hooks, >60% on components)
- ✅ Loading, error, empty states all working
- ✅ Responsive design verified (mobile, tablet, desktop)
- ✅ Keyboard navigation tested
- ✅ Accessibility audited (no violations)
- ✅ Visual design matches system
- ✅ API boundaries clean (no data hardcoding)
- ✅ Performance acceptable (<3s load)
- ✅ No new dependencies added without justification
- ✅ Code reviewed and clear

Good luck! Follow the patterns, check the reference documents, and build incrementally.

