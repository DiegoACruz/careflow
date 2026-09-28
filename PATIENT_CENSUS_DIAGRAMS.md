# Patient Census Feature - Architecture Diagrams

## 1. File Structure Visualization

```
src/features/patients/
│
├── api/
│   └── patientsApi.ts
│       ├── getPatients(filters) → Promise<PatientListResponse>
│       └── getPatientDetail(id) → Promise<Patient>
│
├── components/
│   ├── __tests__/
│   │   ├── PatientCell.test.tsx
│   │   ├── PatientDetailDrawer.test.tsx
│   │   ├── PatientFilters.test.tsx
│   │   ├── PatientRow.test.tsx
│   │   ├── PatientSearch.test.tsx
│   │   ├── PatientSkeleton.test.tsx
│   │   ├── PatientTable.test.tsx
│   │   ├── PatientsError.test.tsx
│   │   ├── PatientsPage.test.tsx
│   │   ├── patients-accessibility.test.tsx
│   │   └── patients-e2e.test.tsx
│   │
│   ├── PatientsPage.tsx (Orchestrator)
│   │   └── Imports: usePatients, usePatientFilters, usePatientSearch
│   │   └── State: detailDrawer, columnVisibility, rowSelection
│   │
│   ├── PatientTable.tsx (TanStack Table)
│   │   ├── Columns: mrn, name, age, status, department, bed, los, vitals, physician, admission, actions
│   │   ├── Sorting: On column headers
│   │   ├── Row selection: Checkboxes
│   │   └── Virtualization: For >50 rows
│   │
│   ├── PatientRow.tsx
│   │   └── Wrapper with hover effects
│   │
│   ├── PatientCell.tsx
│   │   ├── Status badge component
│   │   ├── Formatted fields
│   │   └── Expandable cells
│   │
│   ├── PatientFilters.tsx
│   │   ├── Status multi-select
│   │   ├── Department multi-select
│   │   ├── Admission type filter
│   │   ├── Date range picker
│   │   ├── Critical vitals toggle
│   │   └── Reset button
│   │
│   ├── PatientSearch.tsx
│   │   ├── Search input
│   │   ├── Debounced onChange
│   │   └── Clear button
│   │
│   ├── PatientDetailDrawer.tsx
│   │   ├── Slide-out panel
│   │   ├── Patient header (name, MRN)
│   │   ├── Demographics section
│   │   ├── Vitals section
│   │   ├── Medications section
│   │   ├── Care team section
│   │   ├── Admission info section
│   │   └── Action buttons
│   │
│   ├── PatientSkeleton.tsx
│   │   └── Loading placeholder
│   │
│   └── PatientsError.tsx
│       └── Error state with retry
│
├── hooks/
│   ├── __tests__/
│   │   ├── usePatientDetail.test.ts
│   │   ├── usePatientFilters.test.ts
│   │   ├── usePatients.test.ts
│   │   └── usePatientSearch.test.ts
│   │
│   ├── usePatients.ts
│   │   ├── Input: filters?: PatientFilters
│   │   └── Output: { data, isLoading, error, refetch }
│   │
│   ├── usePatientDetail.ts
│   │   ├── Input: patientId?: string
│   │   └── Output: { data: Patient, isLoading, error }
│   │
│   ├── usePatientFilters.ts
│   │   ├── Input: (none, reads from URL)
│   │   └── Output: { filters, setFilters, resetFilters }
│   │
│   └── usePatientSearch.ts
│       ├── Input: onSearch, debounceMs
│       └── Output: { searchTerm, handleSearchChange, clearSearch }
│
├── schemas/
│   ├── __tests__/
│   │   ├── patient.schema.test.ts
│   │   ├── patientFilters.schema.test.ts
│   │   └── patientSearch.schema.test.ts
│   │
│   ├── patient.schema.ts
│   │   ├── patientStatusSchema: critical | stable | observation | recovery | pending-discharge
│   │   ├── admissionTypeSchema: emergency | urgent | scheduled | observation
│   │   ├── vitalSignsSchema: { heartRate, BP, temp, RR, O2, timestamp }
│   │   ├── medicationSchema: { id, name, dosage, frequency, route, started, stopped }
│   │   └── patientSchema: (full patient object)
│   │
│   ├── patientFilters.schema.ts
│   │   └── PatientFilters: { status[], departmentIds[], admissionType[], dates, criticalVitals, sort, page, pageSize }
│   │
│   └── patientSearch.schema.ts
│       └── PatientSearch: { query, searchFields, highlightMatches }
│
├── types/
│   └── patient.types.ts
│       ├── Re-exports from schemas
│       ├── Patient, PatientList, PatientStatus, AdmissionType, VitalSigns, Medication
│       ├── PatientFilters, PatientSearch
│       ├── PatientDetailState: { isOpen, patientId }
│       └── PatientTableState: { selectedRowIds, columnVisibility }
│
└── index.ts
    └── Feature barrel export (export * from './types')
```

---

## 2. Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          PatientsPage                               │
│                      (Orchestrator Component)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────┐    ┌──────────────────────────────┐  │
│  │  usePatientFilters()    │    │  usePatientSearch()          │  │
│  ├─────────────────────────┤    ├──────────────────────────────┤  │
│  │ Reads: URL params       │    │ Manages: Search term         │  │
│  │ State: filters          │    │ Debounces: onChange (300ms)  │  │
│  │ Method: setFilters()    │    │ Output: searchTerm           │  │
│  │ Method: resetFilters()  │    │ Method: handleSearchChange() │  │
│  └──────────┬──────────────┘    └──────────────┬───────────────┘  │
│             │                                   │                  │
│             └───────────────────┬───────────────┘                  │
│                                 │                                  │
│                          ┌──────▼──────┐                           │
│                          │  usePatients()                          │
│                          │  with filters                           │
│                          └──────┬───────┘                           │
│                                 │                                  │
└─────────────────────────────────┼──────────────────────────────────┘
                                  │
                    ┌─────────────▼────────────┐
                    │  TanStack Query Hook     │
                    ├─────────────────────────┤
                    │ queryKey: ['patients',  │
                    │           filters]     │
                    │ queryFn:               │
                    │   patientsApi.         │
                    │   getPatients(filters) │
                    │ staleTime: 30s         │
                    │ gcTime: 5m             │
                    └─────────────┬──────────┘
                                  │
                    ┌─────────────▼────────────┐
                    │ patientsApi.getPatients()│
                    ├─────────────────────────┤
                    │ Builds URL params       │
                    │ Calls apiClient.get()   │
                    │ Validates: Zod schema  │
                    │ Returns: PatientList    │
                    └─────────────┬──────────┘
                                  │
                    ┌─────────────▼────────────┐
                    │ apiClient.get()         │
                    ├─────────────────────────┤
                    │ Constructs URL          │
                    │ fetch() request         │
                    │ Returns JSON response   │
                    └─────────────┬──────────┘
                                  │
                    ┌─────────────▼────────────┐
                    │ MSW Interceptor         │
                    ├─────────────────────────┤
                    │ Matches: GET /api/      │
                    │          patients?...  │
                    │ Filters mock data       │
                    │ Applies sorting         │
                    │ Applies pagination      │
                    │ Returns response        │
                    └─────────────┬──────────┘
                                  │
                    ┌─────────────▼────────────┐
                    │ Mock Data Store         │
                    ├─────────────────────────┤
                    │ mockPatients (75)       │
                    │ - Regenerated on init   │
                    │ - Filtered in handler   │
                    └────────────────────────┘
```

---

## 3. Component Hierarchy Tree

```
PatientsPage (Main Container)
│
├── PatientFilters (Sidebar/Top)
│   ├── StatusFilter (Multi-select dropdown)
│   │   └── Option items (critical, stable, observation, recovery, pending-discharge)
│   ├── DepartmentFilter (Multi-select dropdown)
│   ├── AdmissionTypeFilter (Multi-select dropdown)
│   ├── DateRangePicker
│   │   ├── From date input
│   │   └── To date input
│   ├── CriticalVitalsToggle (Switch)
│   └── ResetButton
│
├── Header (Search + Results Count)
│   ├── PatientSearch
│   │   ├── Input field
│   │   └── Clear button
│   └── Result count display
│
├── PatientTable (Main Content - Conditional Rendering)
│   ├── Table Header
│   │   └── ColumnHeaders (sortable)
│   │       ├── MRN (sort, filter)
│   │       ├── Name (sort, filter)
│   │       ├── Age
│   │       ├── Status (sort, filter) → Status Badge with color
│   │       ├── Department (sort, filter)
│   │       ├── Bed
│   │       ├── LOS (sort)
│   │       ├── Vitals (sort)
│   │       ├── Physician
│   │       ├── Admission Type (sort, filter)
│   │       └── Actions
│   │
│   └── Table Body
│       └── PatientRow (repeated for each patient)
│           ├── PatientCell (MRN) → with copy button
│           ├── PatientCell (Name + Age)
│           ├── PatientCell (Status badge) → colored
│           ├── PatientCell (Department)
│           ├── PatientCell (Bed)
│           ├── PatientCell (LOS)
│           ├── PatientCell (Vitals summary)
│           ├── PatientCell (Physician)
│           ├── PatientCell (Admission type)
│           └── PatientCell (Action button)
│               └── onClick → open PatientDetailDrawer
│
├── PatientPagination (Bottom)
│   ├── Previous button
│   ├── Page indicator
│   ├── Next button
│   └── Page size selector
│
├── PatientSkeleton (Loading state - Conditional)
│   └── Skeleton rows (5)
│
├── PatientsError (Error state - Conditional)
│   ├── Error icon
│   ├── Error message
│   └── Retry button
│
├── EmptyState (No results - Conditional)
│   └── "No patients found" message
│
└── PatientDetailDrawer (Slide-out panel - Overlay)
    ├── Header
    │   ├── Close button
    │   ├── Patient name + MRN
    │   └── Status badge
    │
    ├── ScrollableContent
    │   ├── Demographics Section
    │   │   ├── Name
    │   │   ├── DOB / Age
    │   │   ├── Gender
    │   │   ├── MRN
    │   │   ├── Phone
    │   │   └── Email
    │   │
    │   ├── Admission Info Section
    │   │   ├── Admission date
    │   │   ├── Admission type
    │   │   ├── Primary diagnosis
    │   │   ├── Admitting physician
    │   │   └── Expected discharge date
    │   │
    │   ├── Location Section
    │   │   ├── Department
    │   │   ├── Bed
    │   │   ├── Room
    │   │   └── Length of stay
    │   │
    │   ├── Vital Signs Section (Real-time)
    │   │   ├── Heart rate
    │   │   ├── Blood pressure
    │   │   ├── Temperature
    │   │   ├── Respiratory rate
    │   │   ├── O2 saturation
    │   │   └── Last updated timestamp
    │   │
    │   ├── Medications Section
    │   │   └── Medication List (expandable)
    │   │       ├── Medication name
    │   │       ├── Dosage
    │   │       ├── Frequency
    │   │       ├── Route
    │   │       ├── Started date
    │   │       └── Stopped date (if applicable)
    │   │
    │   ├── Allergies Section
    │   │   └── Allergy badges
    │   │
    │   └── Care Team Section
    │       └── Assigned nurses list
    │           ├── Nurse name
    │           └── Contact info
    │
    └── Footer
        ├── Discharge button
        ├── Transfer button
        └── Edit button
```

---

## 4. State Management Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                     STATE MANAGEMENT LAYERS                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ LAYER 1: URL STATE (Persistence Layer)                      │
│ ────────────────────────────────────────────────────────────│
│  ?status=critical,stable&departmentIds=uuid-1&page=2        │
│  &sortBy=admissionDate&sortOrder=asc                         │
│                                                              │
│  usePatientFilters() Hook                                    │
│  ├── Reads: URL search params                               │
│  ├── State: filters (derived from URL)                      │
│  ├── Method: setFilters() → updates URL                     │
│  └── Method: resetFilters() → clears URL                    │
│                                                              │
│  Persists across:                                            │
│  ✓ Page refresh (F5)                                        │
│  ✓ Back button navigation                                   │
│  ✓ Browser history                                          │
│  ✓ Shareable URLs                                           │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ LAYER 2: SERVER STATE (Data Caching Layer)                  │
│ ────────────────────────────────────────────────────────────│
│                                                              │
│  TanStack Query Cache                                        │
│  ├── queryKey: ['patients', { filters }]                    │
│  ├── queryFn: patientsApi.getPatients()                     │
│  ├── staleTime: 30 seconds                                  │
│  ├── gcTime (cacheTime): 5 minutes                          │
│  ├── refetchOnWindowFocus: true                             │
│  └── refetchInterval: 30 seconds                            │
│                                                              │
│  Manages:                                                    │
│  ✓ Network requests deduplication                           │
│  ✓ Automatic refetching                                     │
│  ✓ Background updates                                       │
│  ✓ Error retry logic                                        │
│  ✓ Loading state                                            │
│                                                              │
│  usePatients() Hook                                          │
│  ├── Input: filters                                         │
│  └── Output: { data, isLoading, error, refetch }            │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ LAYER 3: LOCAL UI STATE (Component-Level State)             │
│ ────────────────────────────────────────────────────────────│
│                                                              │
│  useState() Hooks (in components)                            │
│                                                              │
│  PatientsPage:                                               │
│  ├── detailDrawer: { isOpen, patientId }                    │
│  │   └── Lifecycle: Open on row click → Close on Escape     │
│  │       or button click                                    │
│  │                                                          │
│  ├── columnVisibility: { [columnId]: boolean }              │
│  │   └── User preferences for column display                │
│  │                                                          │
│  └── rowSelection: { [rowId]: boolean }                     │
│      └── Selected rows (for bulk operations)                │
│                                                              │
│  PatientSearch:                                              │
│  ├── searchTerm: string                                     │
│  │   └── With debouncing (300ms delay)                      │
│  │                                                          │
│  └── Callback: onSearch(query) → setFilters()              │
│                                                              │
│  PatientDetailDrawer:                                        │
│  └── Controlled by parent (detailDrawer state)              │
│      └── Focus management on open/close                     │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│ STATE FLOW EXAMPLE: Changing Sort Order                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. User clicks "MRN" column header                          │
│     │                                                        │
│  2. PatientTable component detects header click             │
│     │                                                        │
│  3. TanStack Table updates sorting state                    │
│     │                                                        │
│  4. Component calls onSortingChange()                        │
│     │                                                        │
│  5. PatientsPage.setFilters({ sortBy: 'mrn', ... })        │
│     │                                                        │
│  6. usePatientFilters updates URL:                           │
│     ?sortBy=mrn&sortOrder=asc (or desc)                     │
│     │                                                        │
│  7. usePatients() query key changes                          │
│     │                                                        │
│  8. TanStack Query invalidates and refetches                │
│     │                                                        │
│  9. New request sent to API/MSW handler                      │
│     │                                                        │
│  10. MSW sorts data by MRN field                             │
│     │                                                        │
│  11. Response cached in TanStack Query                       │
│     │                                                        │
│  12. PatientTable re-renders with sorted data               │
│     │                                                        │
│  13. URL persists → shareable & browser history ready       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Query Key Hierarchy

```
queryKeys
└── patients
    ├── all: ['patients']
    │   └── Invalidate all patient queries
    │
    ├── lists()
    │   └── ['patients', 'list']
    │       └── Invalidate all list queries
    │
    ├── list(filters)
    │   ├── ['patients', 'list', { status: ['critical'], page: 1, pageSize: 25 }]
    │   ├── ['patients', 'list', { status: ['stable'], page: 1 }]
    │   ├── ['patients', 'list', { departmentIds: ['uuid-1'], page: 2 }]
    │   └── Each filter combination has its own cache entry
    │       → Switching between "Critical" and "Stable" keeps both cached
    │       → Selecting previous page loads from cache immediately
    │
    ├── details()
    │   └── ['patients', 'detail']
    │       └── Invalidate all detail queries
    │
    ├── detail(id)
    │   ├── ['patients', 'detail', 'uuid-patient-1']
    │   ├── ['patients', 'detail', 'uuid-patient-2']
    │   └── Each patient detail cached separately
    │       → Opening detail doesn't invalidate list
    │       → Switching between patients uses cached data
    │
    └── search(query)
        ├── ['patients', 'search', 'sarah']
        ├── ['patients', 'search', 'MRN123']
        └── Optional: Separate cache for search results
```

---

## 6. API Request Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                 Patient Filters Component                   │
│              (User selects: Status = "Critical")            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
        ┌────────────────────────────────┐
        │ setFilters({ status: ['critical'] }) │
        └────────────────┬─────────────────┘
                         │
                         ▼
         ┌──────────────────────────────┐
         │ usePatientFilters Hook       │
         │ Updates URL search params    │
         │ ?status=critical&page=1      │
         └────────────────┬──────────────┘
                         │
                         ▼
        ┌───────────────────────────────┐
        │ URL Changes Detected by React │
        │ Router                        │
        │ Component re-renders with     │
        │ new search params             │
        └────────────────┬──────────────┘
                         │
                         ▼
      ┌──────────────────────────────────┐
      │ usePatients(filters) Called      │
      │ filters = { status: ['critical'] }│
      └────────────────┬─────────────────┘
                       │
                       ▼
     ┌───────────────────────────────────┐
     │ queryKey Changes:                 │
     │ ['patients', 'list', {            │
     │   status: ['critical']            │
     │ }]                                │
     └────────────────┬──────────────────┘
                      │
                      ▼ (Cache hit or miss?)
     ┌────────────────────────────────┐
     │ if (queryKey in cache &&       │
     │     !isStale) {                │
     │   Return cached data           │
     │ } else {                       │
     │   Fetch new data               │
     │ }                              │
     └────────────────┬───────────────┘
                      │
              ┌───────┴────────┐
              │                │
              ▼                ▼
    (Cache Hit)          (Cache Miss/Stale)
    Return               │
    immediately          ▼
                 ┌──────────────────────────┐
                 │ patientsApi.getPatients()│
                 │ Builds URL params:       │
                 │ /api/patients?           │
                 │ status=critical&         │
                 │ page=1&pageSize=25       │
                 └──────────────┬───────────┘
                                │
                                ▼
                      ┌──────────────────────┐
                      │ apiClient.get()      │
                      │ fetch(url)           │
                      └──────────────┬───────┘
                                     │
                        ┌────────────┴─────────────┐
                        │                          │
                        ▼ (Real API)    ▼ (MSW Mock)
                 ┌────────────────┐  ┌─────────────────────┐
                 │ HTTP Request   │  │ MSW Interceptor     │
                 │ 400ms latency  │  │ Matches pattern:    │
                 └────────────────┘  │ GET /api/patients   │
                        │            │ with query params   │
                        ▼            └────────┬────────────┘
                 ┌────────────────┐           │
                 │ Response JSON  │           ▼
                 └────────────────┘  ┌───────────────────────┐
                                     │ Filter mock data:     │
                                     │ patients where        │
                                     │ status === 'critical' │
                                     │                       │
                                     │ Apply pagination:     │
                                     │ slice(0, 25)          │
                                     │                       │
                                     │ Return response with  │
                                     │ pagination metadata   │
                                     │ 300ms delay           │
                                     └────────┬──────────────┘
                                              │
                        ┌─────────────────────┴─────────────┐
                        │                                   │
                        ▼                                   ▼
         ┌──────────────────────────┐      ┌──────────────────────────┐
         │ Real API Response        │      │ MSW Response             │
         │ 200 OK                   │      │ 200 OK                   │
         │ {data: [...], ...}       │      │ {data: [...], ...}       │
         └──────────────┬───────────┘      └──────────────┬───────────┘
                        │                                 │
                        └─────────────────┬───────────────┘
                                          │
                                          ▼
                        ┌─────────────────────────────────┐
                        │ Parse & Validate with Zod       │
                        │ patientListSchema.parse(data)   │
                        │ ✓ Valid → Continue             │
                        │ ✗ Invalid → Throw error        │
                        └──────────────┬──────────────────┘
                                       │
                                       ▼
                        ┌──────────────────────────────┐
                        │ Store in TanStack Cache      │
                        │ queryKey → response data     │
                        │ Update staleTime timer (30s) │
                        │ Mark as: isLoading = false   │
                        └──────────────┬───────────────┘
                                       │
                                       ▼
                        ┌──────────────────────────────┐
                        │ Component Re-render          │
                        │ with new patient list        │
                        │ PatientTable updated         │
                        │ Status badges colored        │
                        │ Pagination controls shown    │
                        └──────────────────────────────┘
```

---

## 7. Vital Signs Color Coding

```
HEART RATE (BPM)
├── Critical:     <40 or >130 (red background)
├── Warning:      40-60 or 100-130 (amber background)
└── Normal:       60-100 (green background)

BLOOD PRESSURE (mmHg)
├── Critical:     <90/<60 or >180/>120 (red)
├── Warning:      90-120/<60 or 160-180/100-120 (amber)
└── Normal:       120-140/60-90 (green)

TEMPERATURE (°C)
├── Critical:     <36 or >39 (red)
├── Warning:      36-36.5 or 38.5-39 (amber)
└── Normal:       36.5-37.5 (green)

RESPIRATORY RATE (breaths/min)
├── Critical:     <8 or >30 (red)
├── Warning:      8-12 or 25-30 (amber)
└── Normal:       12-20 (green)

OXYGEN SATURATION (%)
├── Critical:     <88% (red)
├── Warning:      88-94% (amber)
└── Normal:       >94% (green)
```

---

## 8. Accessibility Feature Map

```
KEYBOARD NAVIGATION
├── Tab Navigation
│   ├── Filter section → Tab through filters
│   ├── Search box → Focus and type
│   └── Table → Navigate rows (if custom keyboard handler)
│
├── Sort Columns
│   ├── Header cells focusable
│   ├── Enter/Space → Toggle sort
│   └── Aria-sort attribute: ascending | descending | none
│
├── Open Detail
│   ├── Row with keyboard focus
│   ├── Enter key → Open drawer
│   └── Shift+Enter → Open in new context (optional)
│
├── Close Detail Drawer
│   ├── Escape key → Close drawer
│   ├── Focus returns to row
│   └── Close button always available
│
├── Pagination
│   ├── Previous/Next buttons focusable
│   ├── Enter/Space → Navigate pages
│   └── Page size select → Keyboard navigable
│
└── Filter Reset
    ├── Reset button focusable
    ├── Enter/Space → Clear all filters
    └── Confirmation optional (for safety)

SCREEN READER SUPPORT
├── Semantic HTML
│   ├── <main> for main content
│   ├── <nav> for navigation
│   ├── <section> for major sections
│   ├── <header>, <footer>, <article> as appropriate
│   └── <table> with <thead>, <tbody>, proper structure
│
├── ARIA Labels
│   ├── aria-label="View patient details" on icon buttons
│   ├── aria-labelledby="section-heading" on sections
│   ├── aria-describedby="help-text" on complex inputs
│   └── aria-live="polite" on dynamic content
│
├── Table Accessibility
│   ├── <th scope="col"> for column headers
│   ├── <th scope="row"> for row headers (if applicable)
│   ├── aria-sort="ascending|descending|none" on sortable columns
│   ├── Announce sort changes with live region
│   └── Announce row count changes
│
├── Status Badges
│   ├── aria-label for color-coded status
│   │   Example: aria-label="Status: Critical"
│   ├── Not just color-dependent
│   └── Include text representation
│
├── Interactive Elements
│   ├── All buttons have accessible labels
│   ├── Form inputs associated with labels
│   ├── Focus visible indicators (border, outline, background)
│   ├── Focus order logical (left-to-right, top-to-bottom)
│   └── No focus traps (can exit any component)
│
└── Live Regions
    ├── Search results count updated
    ├── Filter changes announced
    ├── Drawer open/close announced
    └── Errors announced immediately

COLOR CONTRAST
├── Text on background: ≥4.5:1
├── UI components: ≥3:1
├── Don't rely on color alone
├── Use patterns + color for status
└── Status badges have text + color

FOCUS MANAGEMENT
├── Drawer opens
│   ├── Focus moves to drawer (trap focus inside)
│   ├── Tab cycles within drawer only
│   └── ESC closes and returns focus to trigger
│
├── Filter dropdown opens
│   ├── Focus on first option
│   ├── Arrow keys navigate options
│   └── Enter selects option
│
└── Error state
    ├── Alert focused when error appears
    ├── Error message announced via live region
    └── Retry button accessible
```

---

## 9. Testing Pyramid

```
                           ▲
                          ╱ ╲
                         ╱   ╲
                        ╱ E2E ╲         5-10 tests
                       ╱   5%  ╲        • Full user flow
                      ╱─────────╲       • Real browser
                     ╱           ╲      • Critical paths
                    ╱ Integration ╲
                   ╱       25%      ╲    20-30 tests
                  ╱─────────────────╲   • Component + API
                 ╱                   ╲  • MSW mocking
                ╱ Component Tests    ╲  • User interactions
               ╱         35%          ╲ 30-50 tests
              ╱──────────────────────╲  • React Testing
             ╱                        ╲ •  Library
            ╱  Unit Tests: Hooks      ╲ • Isolated logic
           ╱         35%               ╲ 50+ tests
          ╱──────────────────────────╱ • Vitest
         ╱ Schemas │ Utilities       ╱  • Fast
        ╱──────────────────────────╱   • Reliable


PATIENT CENSUS TEST COVERAGE TARGETS
├── Hooks (40+ tests)
│   ├── usePatients (8 tests)
│   │   ├── Fetches patients successfully
│   │   ├── Filters patients by status
│   │   ├── Filters by department
│   │   ├── Handles sorting
│   │   ├── Handles pagination
│   │   ├── Refetches on stale
│   │   ├── Handles errors
│   │   └── Refetches on window focus
│   │
│   ├── usePatientDetail (6 tests)
│   │   ├── Fetches patient detail
│   │   ├── Handles null ID (disabled)
│   │   ├── Refetches on ID change
│   │   ├── Caches patient detail
│   │   ├── Handles error
│   │   └── Refetches stale detail
│   │
│   ├── usePatientFilters (10 tests)
│   │   ├── Reads URL params
│   │   ├── Updates URL on filter change
│   │   ├── Resets all filters
│   │   ├── Persists across navigation
│   │   ├── Handles missing params
│   │   ├── Validates filter values
│   │   ├── Preserves page on filter change
│   │   ├── Resets page on filter change
│   │   ├── Handles invalid sort order
│   │   └── Handles invalid page number
│   │
│   └── usePatientSearch (6 tests)
│       ├── Updates search term
│       ├── Debounces search (300ms)
│       ├── Clears search term
│       ├── Calls onSearch with debounce
│       ├── Cancels previous debounce
│       └── Handles rapid input
│
├── Components (40+ tests)
│   ├── PatientsPage (8 tests)
│   │   ├── Renders patient table
│   │   ├── Shows loading state
│   │   ├── Shows error state with retry
│   │   ├── Shows empty state
│   │   ├── Opens detail drawer on click
│   │   ├── Closes drawer on escape
│   │   ├── Integrates filters
│   │   └── Integrates search
│   │
│   ├── PatientTable (10 tests)
│   │   ├── Renders table with columns
│   │   ├── Sorts on header click
│   │   ├── Displays patient status badges
│   │   ├── Handles empty state
│   │   ├── Loads skeleton on loading
│   │   ├── Shows vitals inline
│   │   ├── Renders action buttons
│   │   ├── Selects/deselects rows
│   │   ├── Responds to prop changes
│   │   └── Handles large datasets (virtualization)
│   │
│   ├── PatientFilters (8 tests)
│   │   ├── Renders all filter controls
│   │   ├── Updates filters on selection
│   │   ├── Shows selected filter values
│   │   ├── Resets all filters
│   │   ├── Handles multi-select
│   │   ├── Filters by date range
│   │   ├── Toggles critical vitals
│   │   └── Closes dropdown on selection
│   │
│   ├── PatientSearch (6 tests)
│   │   ├── Renders search input
│   │   ├── Updates on input change
│   │   ├── Debounces searches
│   │   ├── Clears search term
│   │   ├── Disables submit without term
│   │   └── Shows clear button when active
│   │
│   └── PatientDetailDrawer (8 tests)
│       ├── Renders drawer when open
│       ├── Closes on backdrop click
│       ├── Closes on escape key
│       ├── Shows patient details
│       ├── Loads patient data
│       ├── Shows medications section
│       ├── Shows vitals section
│       └── Manages focus on open/close
│
├── Schemas (15 tests)
│   ├── patient.schema (6 tests)
│   │   ├── Validates complete patient
│   │   ├── Rejects missing required fields
│   │   ├── Validates status enum
│   │   ├── Validates vital signs ranges
│   │   ├── Validates ISO dates
│   │   └── Parses type correctly
│   │
│   ├── patientFilters.schema (5 tests)
│   │   ├── Validates filter object
│   │   ├── Provides defaults
│   │   ├── Validates page number
│   │   ├── Validates pageSize
│   │   └── Rejects invalid sort order
│   │
│   └── patientSearch.schema (4 tests)
│       ├── Validates search query
│       ├── Validates searchFields array
│       ├── Provides defaults
│       └── Rejects long queries
│
├── Accessibility (10+ tests)
│   ├── Keyboard navigation
│   │   ├── Tab through all interactive elements
│   │   ├── Enter/Space activate buttons
│   │   ├── Escape closes drawer
│   │   └── Arrow keys navigate selects
│   │
│   ├── Screen reader
│   │   ├── Semantic HTML structure
│   │   ├── ARIA labels present
│   │   ├── Live regions for updates
│   │   └── Table headers properly marked
│   │
│   ├── Color contrast
│   │   ├── Text vs background ≥4.5:1
│   │   ├── UI elements ≥3:1
│   │   └── Status badges have text + color
│   │
│   ├── Focus management
│   │   ├── Focus visible indicators
│   │   ├── Logical focus order
│   │   └── No focus traps
│   │
│   └── axe-core automated checks
│       └── No WCAG violations
│
├── Integration (10+ tests)
│   ├── List → Detail flow
│   │   ├── Load list
│   │   ├── Click row
│   │   ├── Detail drawer opens
│   │   └── Detail data matches row data
│   │
│   ├── Filter → Sort → Search flow
│   │   ├── Apply filters
│   │   ├── Sort results
│   │   ├── Search within results
│   │   └── All state persists in URL
│   │
│   ├── Pagination flow
│   │   ├── Load first page
│   │   ├── Go to next page
│   │   ├── Change page size
│   │   └── Go to specific page
│   │
│   └── Error recovery
│       ├── Show error
│       ├── Click retry
│       └── Refetch succeeds
│
└── E2E (5-10 tests) [Playwright]
    ├── Load patients page
    ├── Filter by status and verify
    ├── Search for patient
    ├── Open detail drawer
    ├── Sort table by column
    ├── Paginate through results
    ├── Keyboard-only navigation
    └── Mobile responsive (if applicable)
```

---

## 10. Performance Optimization Opportunities

```
BEFORE OPTIMIZATION
┌──────────────────────────────────────┐
│ React Component Renders              │
│ • Every filter change → All re-render │
│ • Every table sort → All re-render    │
│ • Every pagination → All re-render    │
│ • Every detail open → All re-render   │
└──────────────────────────────────────┘
             ▼
     Performance: POOR
     Initial load: 3s+
     Interactions: Visible delay

AFTER OPTIMIZATION
┌──────────────────────────────────────┐
│ Memoized Components                  │
│ • PatientTable → memo()              │
│ • PatientRow → memo()                │
│ • PatientCell → memo()               │
│ • PatientFilters → memo()            │
└──────────────────────────────────────┘
             ▼
┌──────────────────────────────────────┐
│ Memoized Callbacks                   │
│ • handleFilterChange → useCallback   │
│ • handleRowClick → useCallback       │
│ • handleSort → useCallback           │
└──────────────────────────────────────┘
             ▼
┌──────────────────────────────────────┐
│ Virtualized Table                    │
│ • Only 10 rows rendered (visible)    │
│ • 65 rows in cache (not DOM)         │
│ • Scroll → Swap out rows dynamically │
└──────────────────────────────────────┘
             ▼
┌──────────────────────────────────────┐
│ Code Splitting                       │
│ • Detail drawer → lazy()             │
│ • Filter panel → lazy()              │
│ • Only load when needed              │
└──────────────────────────────────────┘
             ▼
┌──────────────────────────────────────┐
│ Query Caching                        │
│ • Filter toggle → Cache hit (instant)│
│ • Sort toggle → Cache hit (instant)  │
│ • Back button → No refetch needed    │
└──────────────────────────────────────┘
             ▼
     Performance: EXCELLENT
     Initial load: 1.5s
     Interactions: Instant (cached)
     Interactions: <500ms (fresh)
```

---

## 11. Error Handling Flow

```
Error Scenario 1: API Fails
┌─────────────────────────────────┐
│ MSW or Real API throws error    │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ TanStack Query catches error    │
│ Sets: error, isLoading = false  │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ Component conditional render    │
│ if (error) → PatientsError      │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ PatientsError component shows   │
│ • Error message                 │
│ • Retry button                  │
│ • Try again link                │
└──────────────┬──────────────────┘
               ▼ (user clicks Retry)
┌─────────────────────────────────┐
│ refetch() called                │
│ Attempts API call again         │
└──────────────┬──────────────────┘
               ▼
        Success or Retry again


Error Scenario 2: Validation Fails
┌─────────────────────────────────┐
│ API response received           │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ Zod schema validation           │
│ patientSchema.parse(data)       │
└──────────────┬──────────────────┘
               ▼
        Validation Error
               │
        ┌──────┴──────┐
        │             │
        ▼ (invalid)   ▼ (valid)
    Throw Error    Continue
        │
        ▼
┌─────────────────────────────────┐
│ TanStack Query catches error    │
│ Sets: error, isLoading = false  │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ Component shows error state     │
│ "Failed to load patient data"   │
└─────────────────────────────────┘


Error Scenario 3: Network Timeout
┌─────────────────────────────────┐
│ Fetch request exceeds timeout   │
│ Default: 30 seconds             │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ TanStack Query retry logic      │
│ Attempt 1 → Fail               │
│ Wait 1s, Attempt 2 → Fail       │
│ Wait 2s, Attempt 3 → Fail       │
│ Give up, set error              │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ Show error with retry button    │
└─────────────────────────────────┘
```

---

## 12. Mock Data Population Strategy

```
Application Startup
        │
        ▼
┌───────────────────────────┐
│ Import MSW browser setup  │
│ (in main.tsx)             │
└────────────┬──────────────┘
             │
             ▼
┌───────────────────────────┐
│ Worker.start()            │
│ Starts Service Worker     │
└────────────┬──────────────┘
             │
             ▼
┌───────────────────────────┐
│ Import handlers           │
│ (from /mocks/handlers)    │
└────────────┬──────────────┘
             │
             ▼
┌───────────────────────────┐
│ Import mock data gen      │
│ (from /mocks/data)        │
└────────────┬──────────────┘
             │
             ▼
┌───────────────────────────┐
│ generateMockPatients(75)  │
│ Creates 75 realistic      │
│ patient objects with:     │
│ • Random names            │
│ • Realistic vitals        │
│ • Valid UUIDs             │
│ • Realistic diagnoses     │
│ • Medications assigned    │
└────────────┬──────────────┘
             │
             ▼
┌───────────────────────────┐
│ Store in module scope     │
│ const mockPatients = [...] │
└────────────┬──────────────┘
             │
             ▼
User Request: GET /api/patients
        │
        ▼
┌───────────────────────────┐
│ MSW handler matches       │
│ http.get('/api/patients') │
└────────────┬──────────────┘
             │
             ▼
┌───────────────────────────┐
│ Filter mock data:         │
│ • Apply status filter     │
│ • Apply dept filter       │
│ • Apply search query      │
│ • Apply sorting           │
│ • Apply pagination        │
└────────────┬──────────────┘
             │
             ▼
┌───────────────────────────┐
│ Return filtered subset    │
│ in proper response format │
│ with metadata             │
└────────────┬──────────────┘
             │
             ▼
Browser receives response
as if from real API
```

This completes the comprehensive architecture diagrams for the Patient Census feature.

