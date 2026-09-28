# Patient Census Feature - Quick Reference Guide

## Quick Start

### File Structure Template

```bash
mkdir -p src/features/patients/{api,components/__tests__,hooks/__tests__,schemas/__tests__,types}

# Create files in order:
1. src/features/patients/schemas/
   - patient.schema.ts
   - patientFilters.schema.ts
   - patientSearch.schema.ts

2. src/features/patients/types/
   - patient.types.ts

3. src/features/patients/api/
   - patientsApi.ts

4. src/features/patients/hooks/
   - usePatients.ts
   - usePatientDetail.ts
   - usePatientFilters.ts
   - usePatientSearch.ts

5. src/features/patients/components/
   - PatientsPage.tsx
   - PatientTable.tsx
   - PatientFilters.tsx
   - PatientSearch.tsx
   - PatientDetailDrawer.tsx
   - PatientRow.tsx
   - PatientCell.tsx
   - PatientSkeleton.tsx
   - PatientsError.tsx

6. src/features/patients/index.ts

7. src/mocks/data/
   - patients.mock.ts

8. src/mocks/handlers/
   - patients.handlers.ts (update index.ts)
```

---

## Schema Templates

### Patient Schema (Copy-Paste)

```typescript
import { z } from 'zod';

export const patientStatusSchema = z.enum(['critical', 'stable', 'observation', 'recovery', 'pending-discharge']);
export const admissionTypeSchema = z.enum(['emergency', 'urgent', 'scheduled', 'observation']);

export const vitalSignsSchema = z.object({
  heartRate: z.number().int().min(0).max(300),
  bloodPressureSystolic: z.number().int().min(0).max(300),
  bloodPressureDiastolic: z.number().int().min(0).max(200),
  temperature: z.number().min(30).max(45),
  respiratoryRate: z.number().int().min(0).max(100),
  oxygenSaturation: z.number().min(0).max(100),
  timestamp: z.string().datetime(),
});

export const medicationSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  dosage: z.string().min(1),
  frequency: z.enum(['once-daily', 'twice-daily', 'three-times-daily', 'four-times-daily', 'every-6-hours', 'every-8-hours', 'every-12-hours', 'as-needed']),
  startedAt: z.string().datetime(),
  stoppedAt: z.string().datetime().nullable().optional(),
  route: z.enum(['oral', 'iv', 'intramuscular', 'subcutaneous', 'inhalation', 'topical']),
});

export const patientSchema = z.object({
  id: z.string().uuid(),
  mrn: z.string().min(1).max(20),
  firstName: z.string().min(1).max(100),
  lastName: z.string().min(1).max(100),
  dateOfBirth: z.string().date(),
  age: z.number().int().min(0).max(150),
  gender: z.enum(['male', 'female', 'other', 'not-specified']),
  phone: z.string().min(10).max(20).nullable().optional(),
  email: z.string().email().nullable().optional(),
  admissionDate: z.string().datetime(),
  admissionType: admissionTypeSchema,
  admittingPhysician: z.string().min(1),
  primaryDiagnosis: z.string().min(1),
  departmentId: z.string().uuid(),
  departmentName: z.string().min(1),
  bedNumber: z.string().min(1),
  roomNumber: z.string().min(1).nullable().optional(),
  status: patientStatusSchema,
  vitalSigns: vitalSignsSchema,
  allergies: z.array(z.string()).nullable().optional(),
  medications: z.array(medicationSchema),
  lengthOfStay: z.number().int().min(0),
  expectedDischargeDate: z.string().datetime().nullable().optional(),
  assignedNurses: z.array(z.object({ id: z.string().uuid(), name: z.string().min(1) })),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});

export const patientListSchema = z.array(patientSchema);
export type Patient = z.infer<typeof patientSchema>;
export type PatientList = z.infer<typeof patientListSchema>;
```

### Filters Schema (Copy-Paste)

```typescript
import { z } from 'zod';
import { patientStatusSchema, admissionTypeSchema } from './patient.schema';

export const patientFiltersSchema = z.object({
  status: z.array(patientStatusSchema).optional(),
  departmentIds: z.array(z.string().uuid()).optional(),
  admissionType: z.array(admissionTypeSchema).optional(),
  admittedAfter: z.string().datetime().optional(),
  admittedBefore: z.string().datetime().optional(),
  criticalVitals: z.boolean().optional(),
  sortBy: z.enum(['name', 'mrn', 'admissionDate', 'status', 'department', 'lengthOfStay', 'updatedAt']).optional().default('updatedAt'),
  sortOrder: z.enum(['asc', 'desc']).optional().default('desc'),
  page: z.number().int().min(1).optional().default(1),
  pageSize: z.number().int().min(1).max(100).optional().default(25),
});

export type PatientFilters = z.infer<typeof patientFiltersSchema>;
```

---

## Hook Templates

### usePatients Hook (Copy-Paste)

```typescript
import { useQuery } from '@tanstack/react-query';
import { queryKeys } from '@/lib/constants/queryKeys';
import { patientsApi } from '../api/patientsApi';
import type { PatientFilters } from '../types/patient.types';

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

### usePatientDetail Hook (Copy-Paste)

```typescript
import { useQuery } from '@tanstack/react-query';
import { queryKeys } from '@/lib/constants/queryKeys';
import { patientsApi } from '../api/patientsApi';

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

### usePatientFilters Hook (Copy-Paste)

```typescript
import { useCallback } from 'react';
import { useSearchParams } from 'react-router-dom';
import { patientFiltersSchema } from '../schemas/patientFilters.schema';
import type { PatientFilters } from '../types/patient.types';

export function usePatientFilters() {
  const [searchParams, setSearchParams] = useSearchParams();

  const filters: PatientFilters = {
    status: searchParams.get('status')?.split(',').filter(Boolean) as any[] || undefined,
    departmentIds: searchParams.get('departmentIds')?.split(',').filter(Boolean) || undefined,
    admissionType: searchParams.get('admissionType')?.split(',').filter(Boolean) as any[] || undefined,
    admittedAfter: searchParams.get('admittedAfter') || undefined,
    admittedBefore: searchParams.get('admittedBefore') || undefined,
    criticalVitals: searchParams.get('criticalVitals') === 'true' || undefined,
    sortBy: (searchParams.get('sortBy') as any) || 'updatedAt',
    sortOrder: (searchParams.get('sortOrder') as any) || 'desc',
    page: parseInt(searchParams.get('page') || '1', 10),
    pageSize: parseInt(searchParams.get('pageSize') || '25', 10),
  };

  const setFilters = useCallback((newFilters: Partial<PatientFilters>) => {
    const updated = { ...filters, ...newFilters };
    const validated = patientFiltersSchema.parse(updated);
    const params = new URLSearchParams();
    if (validated.status?.length) params.set('status', validated.status.join(','));
    if (validated.departmentIds?.length) params.set('departmentIds', validated.departmentIds.join(','));
    if (validated.admissionType?.length) params.set('admissionType', validated.admissionType.join(','));
    if (validated.admittedAfter) params.set('admittedAfter', validated.admittedAfter);
    if (validated.admittedBefore) params.set('admittedBefore', validated.admittedBefore);
    if (validated.criticalVitals) params.set('criticalVitals', 'true');
    if (validated.sortBy !== 'updatedAt') params.set('sortBy', validated.sortBy);
    if (validated.sortOrder !== 'desc') params.set('sortOrder', validated.sortOrder);
    if (validated.page > 1) params.set('page', validated.page.toString());
    if (validated.pageSize !== 25) params.set('pageSize', validated.pageSize.toString());
    setSearchParams(params);
  }, [filters, setSearchParams]);

  const resetFilters = useCallback(() => {
    setSearchParams(new URLSearchParams());
  }, [setSearchParams]);

  return { filters, setFilters, resetFilters };
}
```

---

## Component Architecture

### Component Import Order (PatientsPage)

```typescript
// 1. React & Hooks
import { useState, useCallback } from 'react';
import { usePatients, usePatientDetail, usePatientFilters, usePatientSearch } from '../hooks';

// 2. Components
import { PatientTable } from './PatientTable';
import { PatientFilters } from './PatientFilters';
import { PatientSearch } from './PatientSearch';
import { PatientDetailDrawer } from './PatientDetailDrawer';
import { PatientSkeleton } from './PatientSkeleton';
import { PatientsError } from './PatientsError';

// 3. Types & Utilities
import type { Patient } from '../types/patient.types';

// Component...
```

### Component Props Interface Pattern

```typescript
// PatientTable.tsx
interface PatientTableProps {
  data: Patient[] | undefined;
  isLoading: boolean;
  filters?: PatientFilters;
  onFiltersChange: (filters: Partial<PatientFilters>) => void;
  onRowClick: (patient: Patient) => void;
}

export function PatientTable({
  data,
  isLoading,
  filters,
  onFiltersChange,
  onRowClick
}: PatientTableProps) {
  // Component implementation
}
```

---

## API Module Template

```typescript
import { apiClient } from '@/lib/api/client';
import type { Patient, PatientFilters } from '../types/patient.types';
import { patientSchema, patientListSchema } from '../schemas/patient.schema';

interface PatientListResponse {
  data: Patient[];
  pagination: { page: number; pageSize: number; total: number; totalPages: number; hasNextPage: boolean; hasPrevPage: boolean };
  metadata: { timestamp: string; count: number };
}

export const patientsApi = {
  getPatients: async (filters?: PatientFilters): Promise<PatientListResponse> => {
    const params = {
      ...(filters?.status && { status: filters.status.join(',') }),
      ...(filters?.departmentIds && { departmentIds: filters.departmentIds.join(',') }),
      sortBy: filters?.sortBy || 'updatedAt',
      sortOrder: filters?.sortOrder || 'desc',
      page: filters?.page || 1,
      pageSize: filters?.pageSize || 25,
    };
    const response = await apiClient.get<PatientListResponse>('/patients', params);
    const validatedPatients = patientListSchema.parse(response.data);
    return { ...response, data: validatedPatients };
  },

  getPatientDetail: async (patientId: string): Promise<Patient> => {
    const response = await apiClient.get<{ data: Patient }>(`/patients/${patientId}`);
    return patientSchema.parse(response.data);
  },
};
```

---

## Mock Data Template (Essential Function)

```typescript
const random = (min: number, max: number): number =>
  Math.floor(Math.random() * (max - min + 1)) + min;

export function generateMockPatient(): Patient {
  const firstNames = ['Sarah', 'Michael', 'Emily', 'James'];
  const lastNames = ['Johnson', 'Smith', 'Williams', 'Brown'];
  const firstName = firstNames[random(0, firstNames.length - 1)];
  const lastName = lastNames[random(0, lastNames.length - 1)];

  return {
    id: crypto.randomUUID(),
    mrn: `MRN${random(100000, 999999)}`,
    firstName,
    lastName,
    dateOfBirth: new Date(random(1940, 2010), random(0, 11), random(1, 28)).toISOString().split('T')[0],
    age: random(18, 95),
    gender: random(0, 1) ? 'male' : 'female',
    phone: `555-${random(100, 999)}-${random(1000, 9999)}`,
    email: `${firstName.toLowerCase()}.${lastName.toLowerCase()}@example.com`,
    admissionDate: new Date(Date.now() - random(1, 48) * 60 * 60 * 1000).toISOString(),
    admissionType: ['emergency', 'urgent', 'scheduled'][random(0, 2)] as any,
    admittingPhysician: 'Dr. Sarah Chen',
    primaryDiagnosis: ['Pneumonia', 'Heart Failure', 'Sepsis'][random(0, 2)],
    departmentId: crypto.randomUUID(),
    departmentName: ['ICU', 'Emergency', 'Medical-Surgical'][random(0, 2)],
    bedNumber: `BED-${random(1, 50)}`,
    roomNumber: `${random(100, 999)}`,
    status: ['critical', 'stable', 'observation'][random(0, 2)] as any,
    vitalSigns: {
      heartRate: random(50, 120),
      bloodPressureSystolic: random(90, 160),
      bloodPressureDiastolic: random(60, 100),
      temperature: random(365, 392) / 10,
      respiratoryRate: random(12, 25),
      oxygenSaturation: random(92, 100),
      timestamp: new Date().toISOString(),
    },
    allergies: ['Penicillin'],
    medications: [{
      id: crypto.randomUUID(),
      name: 'Lisinopril',
      dosage: '10mg',
      frequency: 'twice-daily',
      startedAt: new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString(),
      stoppedAt: null,
      route: 'oral',
    }],
    lengthOfStay: random(1, 48),
    expectedDischargeDate: new Date(Date.now() + random(1, 7) * 24 * 60 * 60 * 1000).toISOString(),
    assignedNurses: [{ id: crypto.randomUUID(), name: 'Nurse Alice' }],
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString(),
  };
}

export function generateMockPatients(count = 75): Patient[] {
  return Array.from({ length: count }, () => generateMockPatient());
}

export const mockPatients = generateMockPatients(75);
```

---

## MSW Handler Template

```typescript
import { delay, HttpResponse, http } from 'msw';
import { generateMockPatients } from '../data/patients.mock';

const mockPatients = generateMockPatients(75);

export const patientsHandlers = [
  http.get('/api/patients', async ({ request }) => {
    await delay(400);
    const url = new URL(request.url);

    const statusFilter = url.searchParams.get('status')?.split(',') || [];
    const sortBy = url.searchParams.get('sortBy') || 'updatedAt';
    const sortOrder = url.searchParams.get('sortOrder') || 'desc';
    const page = parseInt(url.searchParams.get('page') || '1', 10);
    const pageSize = parseInt(url.searchParams.get('pageSize') || '25', 10);

    let filtered = [...mockPatients];
    if (statusFilter.length) filtered = filtered.filter(p => statusFilter.includes(p.status));

    // Sort...
    const total = filtered.length;
    const paginatedData = filtered.slice((page - 1) * pageSize, page * pageSize);

    return HttpResponse.json({
      data: paginatedData,
      pagination: {
        page,
        pageSize,
        total,
        totalPages: Math.ceil(total / pageSize),
        hasNextPage: page < Math.ceil(total / pageSize),
        hasPrevPage: page > 1,
      },
      metadata: { timestamp: new Date().toISOString(), count: paginatedData.length },
    });
  }),

  http.get('/api/patients/:id', async ({ params }) => {
    await delay(300);
    const patient = mockPatients.find(p => p.id === params.id);
    if (!patient) return HttpResponse.json({ error: 'Not found' }, { status: 404 });
    return HttpResponse.json({ data: patient, metadata: { timestamp: new Date().toISOString(), version: 1 } });
  }),
];
```

---

## Common Component Patterns

### Status Badge Pattern

```typescript
const statusColors = {
  critical: 'bg-red-100 text-red-800 border-red-300',
  stable: 'bg-green-100 text-green-800 border-green-300',
  observation: 'bg-blue-100 text-blue-800 border-blue-300',
  recovery: 'bg-amber-100 text-amber-800 border-amber-300',
  'pending-discharge': 'bg-purple-100 text-purple-800 border-purple-300',
};

<span className={`inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium border ${statusColors[status]}`}>
  {status.replace('-', ' ')}
</span>
```

### Loading Skeleton Pattern

```typescript
<div className="space-y-2">
  {Array.from({ length: 5 }).map((_, i) => (
    <div key={i} className="h-12 bg-gray-200 rounded animate-pulse" />
  ))}
</div>
```

### Error State Pattern

```typescript
<div className="flex flex-col items-center justify-center py-12">
  <AlertCircle className="w-12 h-12 text-red-500 mb-4" />
  <p className="text-gray-900 font-medium mb-2">Failed to load patients</p>
  <p className="text-gray-600 text-sm mb-4">{error.message}</p>
  <button
    onClick={() => refetch()}
    className="px-4 py-2 bg-primary-600 text-white rounded-md hover:bg-primary-700"
  >
    Try Again
  </button>
</div>
```

### Empty State Pattern

```typescript
<div className="flex flex-col items-center justify-center py-12">
  <Users className="w-12 h-12 text-gray-400 mb-4" />
  <p className="text-gray-900 font-medium mb-2">No patients found</p>
  <p className="text-gray-600 text-sm">Try adjusting your filters or search term</p>
</div>
```

---

## Testing Patterns

### Hook Test Template

```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { usePatients } from '../usePatients';

describe('usePatients', () => {
  it('fetches patient list', async () => {
    const client = new QueryClient();
    const wrapper = ({ children }: any) => (
      <QueryClientProvider client={client}>{children}</QueryClientProvider>
    );

    const { result } = renderHook(() => usePatients(), { wrapper });

    expect(result.current.isLoading).toBe(true);

    await waitFor(() => expect(result.current.isLoading).toBe(false));

    expect(result.current.data?.data).toBeDefined();
    expect(Array.isArray(result.current.data?.data)).toBe(true);
  });
});
```

### Component Test Template

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { PatientsPage } from '../PatientsPage';

describe('PatientsPage', () => {
  it('renders and loads patients', async () => {
    render(<PatientsPage />);

    expect(await screen.findByRole('table')).toBeInTheDocument();
  });

  it('opens detail drawer on row click', async () => {
    const user = userEvent.setup();
    render(<PatientsPage />);

    const rows = await screen.findAllByRole('row');
    await user.click(rows[1]); // Click first data row

    expect(screen.getByRole('complementary')).toBeInTheDocument(); // Drawer
  });
});
```

---

## Query Keys to Update

Add to `/src/lib/constants/queryKeys.ts`:

```typescript
patients: {
  all: ['patients'] as const,
  lists: () => [...queryKeys.patients.all, 'list'] as const,
  list: (filters?: Record<string, unknown>) =>
    [...queryKeys.patients.lists(), filters] as const,
  details: () => [...queryKeys.patients.all, 'detail'] as const,
  detail: (id: string) => [...queryKeys.patients.details(), id] as const,
  search: (query?: string) => [...queryKeys.patients.all, 'search', query] as const,
},
```

---

## Accessibility Checklist

- [ ] Semantic HTML (`<main>`, `<section>`, `<table>`, `<button>`)
- [ ] ARIA labels on icon buttons (`aria-label="View details"`)
- [ ] Table headers with `<th scope="col">`
- [ ] Focus indicators visible (`:focus-visible`)
- [ ] Keyboard navigation (Tab, Enter, Escape, Arrow keys)
- [ ] Live regions for dynamic updates (`aria-live="polite"`)
- [ ] Color contrast ≥4.5:1 for text
- [ ] Focus management (drawer opens/closes)
- [ ] No focus traps (can Tab out of any component)
- [ ] Status not indicated by color alone (include text)

---

## Performance Checklist

- [ ] Memoize components: `memo(PatientTable)`
- [ ] Memoize callbacks: `useCallback(handleSort, [])`
- [ ] Virtualize table if >50 rows
- [ ] Lazy load drawer: `lazy(() => import('./PatientDetailDrawer'))`
- [ ] Optimize API caching stale/gc times
- [ ] No unnecessary re-renders (check React DevTools)
- [ ] Bundle size acceptable (<100KB gzipped)
- [ ] Images optimized (WebP, responsive)

---

## Common Commands

```bash
# Add query keys
edit src/lib/constants/queryKeys.ts

# Register MSW handlers
edit src/mocks/handlers/index.ts
# Add: import { patientsHandlers } from './patients.handlers';
# Add: export const handlers = [...dashboardHandlers, ...patientsHandlers];

# Update feature export
edit src/features/patients/index.ts
# Add: export * from './types';

# Run tests
npm test -- src/features/patients

# Check accessibility
npm test -- --testMatch="*accessibility*"

# Bundle analysis
npm run build && npx vite-bundle-analyzer

# Lighthouse audit
npm run build && npm run preview
# Then use Chrome DevTools Lighthouse
```

---

## Dependency Versions (Reference)

```json
{
  "react": "^19.3.0",
  "typescript": "^5.x",
  "@tanstack/react-query": "^5.104.0",
  "@tanstack/react-table": "^8.21.0",
  "zod": "^3.x",
  "msw": "^2.15.0",
  "react-router-dom": "^6.x",
  "tailwindcss": "^4.x",
  "shadcn": "^4.x"
}
```

This guide provides copy-paste templates and common patterns for rapid implementation!

