# Patient Census Feature - Architecture Specification

## Overview

This is a complete technical architecture specification for the **Patient Census** feature in CareFlow. The specification includes comprehensive documentation covering design, implementation, testing, and accessibility requirements.

**Status**: Ready for Implementation
**Complexity**: Medium-High
**Estimated Effort**: 8-12 hours
**Technology Stack**: React 19.3.0, TypeScript, Vite 8, TanStack Query 5.104.0, TanStack Table 8.21.3, Zod, MSW 2.15.0

---

## Documentation Structure

This specification is organized into 4 comprehensive documents:

### 1. **PATIENT_CENSUS_ARCHITECTURE.md** (1,923 lines)
**Complete Technical Specification**

The main specification document containing:
- Full file structure and directory layout
- Complete Zod schema definitions (patient, filters, search)
- API endpoint specifications with request/response shapes
- TanStack Query integration and hook signatures
- TanStack Table column definitions and configuration
- Component architecture and hierarchy
- State management strategy (server, URL, local)
- Mock data generation strategy
- Performance optimization guidelines
- Accessibility requirements (WCAG 2.2 AA)
- Testing strategy and coverage goals
- Implementation checklist
- Example API requests/responses

**Use this document for**: Reference implementation, understanding architecture decisions, schema definitions, component responsibilities

### 2. **PATIENT_CENSUS_DIAGRAMS.md** (1,177 lines)
**Visual Architecture & Flow Diagrams**

ASCII diagrams and visual representations including:
- File structure tree with descriptions
- Data flow diagram (user action → API call → response → render)
- Component hierarchy tree showing all components and props
- State management layers (URL, Server, Local)
- Query key hierarchy
- API request flow with MSW interception
- Vital signs color coding system
- Accessibility feature map
- Testing pyramid with coverage targets
- Performance optimization opportunities
- Error handling flow
- Mock data population strategy

**Use this document for**: Understanding visual relationships, data flow, component structure, state flow

### 3. **PATIENT_CENSUS_QUICK_REFERENCE.md** (639 lines)
**Copy-Paste Implementation Templates**

Ready-to-use code templates including:
- File structure setup commands
- Complete copy-paste schemas (patient, filters, search)
- Complete copy-paste hooks (usePatients, usePatientDetail, usePatientFilters, usePatientSearch)
- API module template
- Component props interface patterns
- Common component patterns (status badge, skeleton, error, empty)
- Hook test templates
- Component test templates
- Query keys to update
- Accessibility checklist
- Performance checklist
- Common commands

**Use this document for**: Rapid implementation, code templates, quick lookups, testing patterns

### 4. **PATIENT_CENSUS_IMPLEMENTATION_GUIDE.md** (1,011 lines)
**Step-by-Step Implementation Path**

Detailed implementation guide organized by phase:
- **Phase 1**: Core Setup (schemas, types, query keys)
- **Phase 2**: API Layer (API module, mock data, MSW handlers)
- **Phase 3**: Hooks (usePatients, usePatientDetail, usePatientFilters, usePatientSearch)
- **Phase 4**: Components (PatientsPage, PatientTable, filters, search, drawer)
- **Phase 5**: Testing (unit, component, accessibility, schemas)
- **Phase 6**: Polish & Accessibility (semantic HTML, keyboard nav, ARIA, contrast)
- **Phase 7**: Integration & Routing

Plus:
- Verification checklist
- Common issues and solutions
- Next steps for enhancements
- Files reference
- Key principles to remember
- Estimated timeline (8-12 hours)
- Success criteria

**Use this document for**: Step-by-step implementation, phase planning, checklists, troubleshooting

---

## Architecture Highlights

### Design Principles

1. **Feature-First Organization**
   - All patients-related code in `src/features/patients/`
   - Components own their types, schemas, hooks, API calls
   - Clean feature boundaries

2. **Type Safety via Zod**
   - Zod schemas as single source of truth
   - All types inferred via `z.infer<typeof schema>`
   - Runtime validation at API boundary

3. **Server State with TanStack Query**
   - Automatic caching and refetching
   - Configurable stale times and gc times
   - Network request deduplication

4. **URL-Based Persistence**
   - Filters stored in URL search params
   - Survives page refresh and back button
   - Shareable URLs with state

5. **TanStack Table for Data Display**
   - Sortable columns
   - Row selection
   - Virtualization for large datasets
   - Accessible HTML structure

6. **Mock Data via MSW**
   - Realistic synthetic healthcare data
   - No real backend required
   - Supports filtering, sorting, pagination

7. **Accessibility First**
   - Semantic HTML throughout
   - WCAG 2.2 AA compliance
   - Keyboard navigation
   - Screen reader support
   - Color contrast checked

8. **Comprehensive Testing**
   - Unit tests for hooks and utilities
   - Component tests with React Testing Library
   - Accessibility tests with axe-core
   - E2E tests with Playwright (optional)

---

## File Structure Overview

```
src/features/patients/
├── api/
│   └── patientsApi.ts                  # API endpoints
├── components/
│   ├── __tests__/                      # Component tests (10+ files)
│   ├── PatientsPage.tsx                # Main page orchestrator
│   ├── PatientTable.tsx                # TanStack Table wrapper
│   ├── PatientFilters.tsx              # Filter sidebar
│   ├── PatientSearch.tsx               # Search input
│   ├── PatientDetailDrawer.tsx         # Detail view
│   ├── PatientRow.tsx                  # Table row
│   ├── PatientCell.tsx                 # Table cell
│   ├── PatientSkeleton.tsx             # Loading skeleton
│   └── PatientsError.tsx               # Error state
├── hooks/
│   ├── __tests__/                      # Hook tests (4+ files)
│   ├── usePatients.ts                  # List query hook
│   ├── usePatientDetail.ts             # Detail query hook
│   ├── usePatientFilters.ts            # Filter state management
│   └── usePatientSearch.ts             # Search state + debouncing
├── schemas/
│   ├── __tests__/                      # Schema tests (3+ files)
│   ├── patient.schema.ts               # Patient validation
│   ├── patientFilters.schema.ts        # Filters validation
│   └── patientSearch.schema.ts         # Search validation
├── types/
│   └── patient.types.ts                # Type barrel export
└── index.ts                            # Feature barrel export

src/mocks/
├── data/
│   └── patients.mock.ts                # Mock data generators
└── handlers/
    └── patients.handlers.ts            # MSW handlers
```

---

## Key Decisions

### Why These Technologies?

- **TanStack Query**: Industry standard for server state, excellent caching
- **TanStack Table**: Headless table library, highly customizable, no styling dependencies
- **Zod**: Type-safe validation, single source of truth for schemas and types
- **MSW**: API mocking without backend, realistic network delays
- **React Router**: URL state management for filter persistence
- **Tailwind CSS**: Utility-first styling, consistent design tokens

### Why This State Strategy?

- **URL State for Filters**: Persists across refresh/navigation, shareable URLs
- **TanStack Query for Data**: Automatic caching, deduplication, refetching
- **Local State for UI**: Drawer open/close, row selection, column visibility
- **No Zustand Needed**: Keeps dependencies minimal, sufficient for this scope

### Why This Component Structure?

- **PatientsPage as Orchestrator**: Single responsibility, coordinates children
- **Hooks for Logic**: Custom hooks extract business logic, highly testable
- **Presentational Components**: Dumb components focused on rendering
- **API Module**: Clean separation from components, single responsibility

---

## Key Features Specification

### Patient List Display
- Paginated table with 25 items per page (configurable)
- Sortable columns (name, MRN, status, admission date, LOS, etc.)
- Color-coded status badges (critical/red, stable/green, etc.)
- Inline vital signs summary
- Patient count and pagination controls

### Filtering
- Multi-select: Status (critical, stable, observation, recovery, pending-discharge)
- Multi-select: Department
- Multi-select: Admission type (emergency, urgent, scheduled, observation)
- Date range: Admitted after/before
- Toggle: Show only critical vitals
- Reset all filters button
- Filters persist in URL

### Search
- Real-time search across name, MRN, diagnosis, bed number
- Debounced input (300ms delay)
- Search term highlighted in results
- Clear search button
- Combined with filters (filter → search within results)

### Detail Drawer
- Slide-out panel on patient row click
- Displays comprehensive patient information:
  - Demographics (name, age, DOB, gender, contact)
  - Admission info (date, type, physician, diagnosis)
  - Current location (department, bed, room)
  - Vital signs (real-time, with normal/warning/critical indicators)
  - Medications (name, dosage, frequency, route, dates)
  - Allergies (badge display)
  - Care team (assigned nurses)
- Action buttons (discharge, transfer, edit)
- Keyboard navigation (Escape to close)
- Focus management

### Responsive Design
- Desktop: Full layout with sidebar filters
- Tablet: Adjusted column widths, collapsible filters
- Mobile: Stacked layout, drawer becomes full-height modal

### Performance
- Table virtualization for 50+ rows
- Component memoization to prevent re-renders
- Code splitting for lazy-loaded components
- Efficient caching (stale time: 30s, gc time: 5min)
- Debounced search (300ms)

---

## API Specification

### GET /api/patients
List patients with filtering, search, sorting, pagination

**Query Parameters**:
- `status`: Comma-separated list (critical,stable,observation,recovery,pending-discharge)
- `departmentIds`: Comma-separated UUIDs
- `admissionType`: Comma-separated list (emergency,urgent,scheduled,observation)
- `admittedAfter`: ISO 8601 datetime
- `admittedBefore`: ISO 8601 datetime
- `criticalVitals`: boolean
- `query`: Search term (searches name, MRN, diagnosis, bed)
- `sortBy`: Field to sort (name, mrn, admissionDate, status, department, lengthOfStay, updatedAt)
- `sortOrder`: asc or desc (default: desc)
- `page`: Page number (default: 1)
- `pageSize`: Items per page, 1-100 (default: 25)

**Response**: `{ data: Patient[], pagination: {...}, metadata: {...} }`

### GET /api/patients/:id
Get detailed patient information

**Response**: `{ data: Patient, metadata: {...} }`

---

## Zod Schemas

### Patient Schema
Complete patient object with:
- Demographics (name, DOB, age, gender, contact)
- Admission info (date, type, physician, diagnosis)
- Location (department, bed, room)
- Medical status (status enum, vital signs, allergies)
- Medications array
- Care team (assigned nurses)
- Timestamps (createdAt, updatedAt)

### Patient Filters Schema
- status: array of status enums (optional)
- departmentIds: array of UUIDs (optional)
- admissionType: array of admission types (optional)
- admittedAfter: ISO datetime (optional)
- admittedBefore: ISO datetime (optional)
- criticalVitals: boolean (optional)
- sortBy: enum with defaults
- sortOrder: 'asc' | 'desc' (default: 'desc')
- page: integer, min 1 (default: 1)
- pageSize: integer, 1-100 (default: 25)

### Patient Search Schema
- query: string, 0-200 chars (default: '')
- searchFields: array of field names (default: ['name', 'mrn'])
- highlightMatches: boolean (default: true)

---

## Component Responsibilities

| Component | Responsibility | Key Props |
|-----------|---|---|
| **PatientsPage** | Orchestrator, state coordination | None (uses hooks) |
| **PatientTable** | Data display with TanStack Table | data, isLoading, filters, callbacks |
| **PatientFilters** | Filter UI controls | filters, onFiltersChange, onResetFilters |
| **PatientSearch** | Search input with debouncing | value, onChange, onClear |
| **PatientDetailDrawer** | Patient detail view | isOpen, patientId, onClose |
| **PatientRow** | Table row wrapper | row, onClick |
| **PatientCell** | Individual cell formatting | value, status |
| **PatientSkeleton** | Loading placeholder | None |
| **PatientsError** | Error state UI | error, onRetry |

---

## Testing Strategy

### Coverage Targets
- **Hooks**: 80%+ coverage (utility functions, query logic)
- **Components**: 60%+ coverage (rendering, interactions)
- **Schemas**: 100% coverage (validation edge cases)
- **Accessibility**: Manual testing + axe-core automation

### Test Types
1. **Unit Tests** (50+): Hooks, schemas, utilities
2. **Component Tests** (40+): Rendering, interactions, state
3. **Accessibility Tests** (10+): WCAG compliance
4. **E2E Tests** (5-10): Critical user journeys

---

## Accessibility Compliance

### WCAG 2.2 AA Features
- Semantic HTML (`<table>`, `<button>`, `<section>`, etc.)
- ARIA labels and roles for interactive elements
- Keyboard navigation (Tab, Enter, Escape, Arrow keys)
- Focus management (visible focus indicators, logical order)
- Color contrast ≥4.5:1 for text, ≥3:1 for UI
- Screen reader support (proper labels, live regions)
- No focus traps (can exit any component via Tab)

### Keyboard Support
- Tab: Navigate interactive elements
- Enter/Space: Activate buttons, open dropdowns
- Escape: Close drawer, cancel operations
- Arrow Keys: Navigate select/dropdown options, table rows
- Sort Column: Click header or Spacebar on focused header

---

## Performance Metrics

### Targets
- Initial Load: <1.5s (First Contentful Paint)
- Time to Interactive: <3s
- Lighthouse Performance: 90+
- Bundle Size: <100KB gzipped

### Optimization Techniques
- Memoized components prevent unnecessary re-renders
- Virtualized table (>50 rows)
- Code splitting with `React.lazy()`
- Efficient API caching (stale/gc times)
- Debounced search (300ms)
- Image optimization

---

## Implementation Phases

1. **Phase 1**: Setup + Schemas (1-2h)
2. **Phase 2**: API + Mock Data (1-2h)
3. **Phase 3**: Hooks (1-2h)
4. **Phase 4**: Components (2-3h)
5. **Phase 5**: Testing (1-2h)
6. **Phase 6**: Polish + Accessibility (1-2h)
7. **Phase 7**: Integration & Routing (30min)

**Total: 8-12 hours**

---

## Next Steps for Implementation

1. **Read PATIENT_CENSUS_ARCHITECTURE.md** for complete specification
2. **Review PATIENT_CENSUS_DIAGRAMS.md** for visual understanding
3. **Follow PATIENT_CENSUS_IMPLEMENTATION_GUIDE.md** step-by-step
4. **Use PATIENT_CENSUS_QUICK_REFERENCE.md** for code templates
5. **Create files in order** (schemas → types → hooks → components)
6. **Test incrementally** (hook tests before component tests)
7. **Verify accessibility** (keyboard nav, screen reader, color contrast)
8. **Performance audit** (DevTools, Lighthouse)

---

## Success Criteria

Feature is complete when:
- TypeScript compiles with no errors
- Biome linting passes
- Tests passing (80%+ hook coverage, 60%+ component coverage)
- Loading, error, empty states working
- Responsive design verified
- Keyboard navigation tested
- Accessibility audited (no violations)
- Visual design consistent
- API boundaries clean
- Performance acceptable
- Code reviewed

---

## Documents Location

All specification documents are located in:
`/Users/carlosfelipecruzpachon/Desktop/Health-project/`

1. `PATIENT_CENSUS_ARCHITECTURE.md` - Main specification (55KB)
2. `PATIENT_CENSUS_DIAGRAMS.md` - Visual diagrams (56KB)
3. `PATIENT_CENSUS_QUICK_REFERENCE.md` - Code templates (20KB)
4. `PATIENT_CENSUS_IMPLEMENTATION_GUIDE.md` - Step-by-step guide (25KB)
5. `PATIENT_CENSUS_README.md` - This file

**Total**: ~176KB, 4,750+ lines of comprehensive documentation

---

## Quick Start Commands

```bash
# Start implementation
cd /Users/carlosfelipecruzpachcon/Desktop/Health-project/careflow

# Create directory structure
mkdir -p src/features/patients/{api,components/__tests__,hooks/__tests__,schemas/__tests__,types}

# Run tests as you implement
npm test -- src/features/patients

# Run linting
npm run lint

# Build and preview
npm run build && npm run preview

# Check accessibility
npm test -- --testMatch="*accessibility*"
```

---

## Key References

### From Dashboard (Study These)
- `/src/features/dashboard/schemas/` - Schema patterns
- `/src/features/dashboard/hooks/` - Hook patterns
- `/src/features/dashboard/api/dashboardApi.ts` - API module pattern
- `/src/mocks/data/dashboard.mock.ts` - Mock data pattern
- `/src/mocks/handlers/dashboard.handlers.ts` - MSW handler pattern

### CareFlow Guidelines
- `/CLAUDE.md` - Project principles and architecture
- `/src/lib/constants/queryKeys.ts` - Query key management
- `/src/lib/api/client.ts` - Centralized API client
- `/src/components/ui/` - shadcn components available

---

## Support & Questions

If you have questions during implementation:
1. Check relevant sections in PATIENT_CENSUS_ARCHITECTURE.md
2. Review pattern in Dashboard feature
3. Consult CLAUDE.md for project principles
4. Check PATIENT_CENSUS_QUICK_REFERENCE.md for code examples
5. Review error handling in PATIENT_CENSUS_DIAGRAMS.md

---

**Ready to implement? Start with PATIENT_CENSUS_ARCHITECTURE.md!**

