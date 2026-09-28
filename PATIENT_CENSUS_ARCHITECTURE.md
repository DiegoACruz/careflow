# Patient Census Feature - Technical Architecture Specification

## Document Overview

This document provides the complete technical architecture specification for the **Patient Census** feature in CareFlow. It includes file structure, data models, API specifications, component hierarchy, and implementation patterns.

**Feature**: Patient Census - Real-time patient list with filtering, search, and detail views
**Status**: Specification Document
**Date**: 2024
**Tech Stack**: React 19.3.0, TypeScript, Vite 8, TanStack Query 5.104.0, TanStack Table 8.21.3, Zod, MSW 2.15.0

---

## 1. File Structure

Complete directory layout for the Patient Census feature following feature-first organization:

```
src/features/patients/
├── api/
│   └── patientsApi.ts              # API module with endpoint functions
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
│   ├── PatientCell.tsx              # Individual cell component
│   ├── PatientDetailDrawer.tsx       # Slide-out detail view
│   ├── PatientFilters.tsx            # Filter controls
│   ├── PatientRow.tsx                # Table row wrapper
│   ├── PatientSearch.tsx             # Search input component
│   ├── PatientSkeleton.tsx            # Loading skeleton
│   ├── PatientsError.tsx             # Error state
│   ├── PatientsPage.tsx              # Main page orchestrator
│   └── PatientTable.tsx              # TanStack Table component
├── hooks/
│   ├── __tests__/
│   │   ├── usePatientDetail.test.ts
│   │   ├── usePatientFilters.test.ts
│   │   ├── usePatients.test.ts
│   │   └── usePatientSearch.test.ts
│   ├── usePatientDetail.ts            # Single patient query hook
│   ├── usePatientFilters.ts           # Filter state management hook
│   ├── usePatients.ts                 # Patient list query hook
│   └── usePatientSearch.ts            # Search state management hook
├── schemas/
│   ├── __tests__/
│   │   ├── patient.schema.test.ts
│   │   ├── patientFilters.schema.test.ts
│   │   └── patientSearch.schema.test.ts
│   ├── patient.schema.ts              # Patient data validation
│   ├── patientFilters.schema.ts       # Filter validation
│   └── patientSearch.schema.ts        # Search params validation
├── types/
│   └── patient.types.ts               # Re-exports from schemas
└── index.ts                           # Feature barrel export

src/mocks/
├── data/
│   └── patients.mock.ts               # Mock data generators
└── handlers/
    └── patients.handlers.ts           # MSW handlers
```

---

## 2. Data Model (Zod Schemas)

### 2.1 Patient Schema

**File**: `/src/features/patients/schemas/patient.schema.ts`

```typescript
import { z } from 'zod';

/**
 * Patient Status Schema
 *
 * Represents the current medical status of a patient.
 */
export const patientStatusSchema = z.enum([
  'critical',
  'stable',
  'observation',
  'recovery',
  'pending-discharge',
]);

/**
 * Admission Type Schema
 *
 * Categorizes how the patient was admitted.
 */
export const admissionTypeSchema = z.enum([
  'emergency',
  'urgent',
  'scheduled',
  'observation',
]);

/**
 * Patient Vital Signs Schema
 *
 * Core vital measurements for the patient.
 */
export const vitalSignsSchema = z.object({
  heartRate: z.number().int().min(0).max(300), // BPM
  bloodPressureSystolic: z.number().int().min(0).max(300), // mmHg
  bloodPressureDiastolic: z.number().int().min(0).max(200), // mmHg
  temperature: z.number().min(30).max(45), // Celsius
  respiratoryRate: z.number().int().min(0).max(100), // Breaths per minute
  oxygenSaturation: z.number().min(0).max(100), // Percentage
  timestamp: z.string().datetime(),
});

/**
 * Medication Schema
 *
 * Current medication administered to the patient.
 */
export const medicationSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  dosage: z.string().min(1), // e.g., "500mg", "2 units"
  frequency: z.enum([
    'once-daily',
    'twice-daily',
    'three-times-daily',
    'four-times-daily',
    'every-6-hours',
    'every-8-hours',
    'every-12-hours',
    'as-needed',
  ]),
  startedAt: z.string().datetime(),
  stoppedAt: z.string().datetime().nullable().optional(),
  route: z.enum(['oral', 'iv', 'intramuscular', 'subcutaneous', 'inhalation', 'topical']),
});

/**
 * Main Patient Schema
 *
 * Comprehensive patient record including demographics, medical status, and care info.
 */
export const patientSchema = z.object({
  id: z.string().uuid(),
  // Demographics
  mrn: z.string().min(1).max(20), // Medical Record Number
  firstName: z.string().min(1).max(100),
  lastName: z.string().min(1).max(100),
  dateOfBirth: z.string().date(),
  age: z.number().int().min(0).max(150),
  gender: z.enum(['male', 'female', 'other', 'not-specified']),
  phone: z.string().min(10).max(20).nullable().optional(),
  email: z.string().email().nullable().optional(),
  // Admission Info
  admissionDate: z.string().datetime(),
  admissionType,
  admittingPhysician: z.string().min(1),
  primaryDiagnosis: z.string().min(1),
  // Current Location
  departmentId: z.string().uuid(),
  departmentName: z.string().min(1),
  bedNumber: z.string().min(1),
  roomNumber: z.string().min(1).nullable().optional(),
  // Medical Status
  status: patientStatusSchema,
  vitalSigns: vitalSignsSchema,
  allergies: z.array(z.string()).nullable().optional(),
  medications: z.array(medicationSchema),
  // Length of Stay
  lengthOfStay: z.number().int().min(0), // Hours
  expectedDischargeDate: z.string().datetime().nullable().optional(),
  // Care Team
  assignedNurses: z.array(
    z.object({
      id: z.string().uuid(),
      name: z.string().min(1),
    }),
  ),
  // Metadata
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});

export const patientListSchema = z.array(patientSchema);

export type Patient = z.infer<typeof patientSchema>;
export type PatientList = z.infer<typeof patientListSchema>;
export type PatientStatus = z.infer<typeof patientStatusSchema>;
export type AdmissionType = z.infer<typeof admissionTypeSchema>;
export type VitalSigns = z.infer<typeof vitalSignsSchema>;
export type Medication = z.infer<typeof medicationSchema>;
```

### 2.2 Patient Filters Schema

**File**: `/src/features/patients/schemas/patientFilters.schema.ts`

```typescript
import { z } from 'zod';
import { patientStatusSchema, admissionTypeSchema } from './patient.schema';

/**
 * Patient Filters Schema
 *
 * Defines all filterable criteria for patient census queries.
 * Maps to URL search parameters and API query params.
 */
export const patientFiltersSchema = z.object({
  // Status filters
  status: z.array(patientStatusSchema).optional(), // Multiple statuses supported
  // Department/Location filters
  departmentIds: z.array(z.string().uuid()).optional(),
  // Admission filters
  admissionType: z.array(admissionTypeSchema).optional(),
  // Time-based filters
  admittedAfter: z.string().datetime().optional(),
  admittedBefore: z.string().datetime().optional(),
  // Vital sign-based filters
  criticalVitals: z.boolean().optional(), // Flag for abnormal vitals
  // Sorting
  sortBy: z
    .enum([
      'name',
      'mrn',
      'admissionDate',
      'status',
      'department',
      'lengthOfStay',
      'updatedAt',
    ])
    .optional()
    .default('updatedAt'),
  sortOrder: z.enum(['asc', 'desc']).optional().default('desc'),
  // Pagination
  page: z.number().int().min(1).optional().default(1),
  pageSize: z.number().int().min(1).max(100).optional().default(25),
});

export type PatientFilters = z.infer<typeof patientFiltersSchema>;
```

### 2.3 Patient Search Schema

**File**: `/src/features/patients/schemas/patientSearch.schema.ts`

```typescript
import { z } from 'zod';

/**
 * Patient Search Schema
 *
 * Defines searchable fields and query parameters.
 */
export const patientSearchSchema = z.object({
  query: z.string().min(0).max(200).default(''), // Search term
  searchFields: z
    .array(z.enum(['name', 'mrn', 'diagnosis', 'bed']))
    .optional()
    .default(['name', 'mrn']),
  // Highlighting for results (UI hint)
  highlightMatches: z.boolean().optional().default(true),
});

export type PatientSearch = z.infer<typeof patientSearchSchema>;
```

### 2.4 Type Barrel Export

**File**: `/src/features/patients/types/patient.types.ts`

```typescript
/**
 * Patient Types
 *
 * All types are inferred from Zod schemas to maintain single source of truth.
 * Import schemas and use z.infer<typeof schema> for type inference.
 */

export type { Patient, PatientList, PatientStatus, AdmissionType, VitalSigns, Medication } from '../schemas/patient.schema';
export type { PatientFilters } from '../schemas/patientFilters.schema';
export type { PatientSearch } from '../schemas/patientSearch.schema';

/**
 * Patient-specific UI types (not validated by Zod)
 */
export interface PatientDetailState {
  isOpen: boolean;
  patientId: string | null;
}

export interface PatientTableState {
  selectedRowIds: Record<string, boolean>;
  columnVisibility: Record<string, boolean>;
}
```

---

## 3. API Endpoints

All API requests flow through the centralized API client (`src/lib/api/client.ts`). Feature-specific API module handles formatting and validation.

**File**: `/src/features/patients/api/patientsApi.ts`

### 3.1 GET /api/patients - Patient List

**Request**:
```typescript
// Query parameters (URL search params)
interface PatientListRequest {
  // Filters
  status?: string[]; // e.g., "critical,stable"
  departmentIds?: string[]; // e.g., "uuid1,uuid2"
  admissionType?: string[]; // e.g., "emergency,scheduled"
  admittedAfter?: string; // ISO 8601 datetime
  admittedBefore?: string; // ISO 8601 datetime
  criticalVitals?: boolean; // Flag
  // Search
  query?: string; // Free-text search term
  searchFields?: string[]; // "name,mrn,diagnosis"
  // Sorting
  sortBy?: string; // "name" | "mrn" | "admissionDate" etc.
  sortOrder?: 'asc' | 'desc';
  // Pagination
  page?: number; // 1-indexed
  pageSize?: number; // 1-100
}
```

**Response**:
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
    timestamp: string; // ISO 8601
    count: number;
  };
}
```

### 3.2 GET /api/patients/:id - Patient Detail

**Request**:
```
GET /api/patients/{patientId}
```

**Response**:
```typescript
interface PatientDetailResponse {
  data: Patient;
  metadata: {
    timestamp: string; // ISO 8601
    version: number; // For optimistic updates
  };
}
```

### 3.3 API Module Implementation

```typescript
import { apiClient } from '@/lib/api/client';
import type { Patient, PatientFilters } from '../types/patient.types';
import { patientSchema, patientListSchema } from '../schemas/patient.schema';

/**
 * Patients API Module
 *
 * All API calls for the patients feature.
 * Includes runtime validation with Zod schemas.
 */

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
  /**
   * GET /api/patients
   *
   * Fetches paginated list of patients with filters and search.
   *
   * Query params:
   * - status?: string[] - Filter by patient status (critical, stable, etc.)
   * - departmentIds?: string[] - Filter by department
   * - admissionType?: string[] - Filter by admission type
   * - admittedAfter?: string - Filter by admission date (ISO 8601)
   * - admittedBefore?: string - Filter by admission date (ISO 8601)
   * - criticalVitals?: boolean - Show only patients with critical vitals
   * - query?: string - Free-text search
   * - searchFields?: string[] - Fields to search (name, mrn, diagnosis, bed)
   * - sortBy?: string - Sort field (name, mrn, admissionDate, status, department, lengthOfStay, updatedAt)
   * - sortOrder?: 'asc' | 'desc' - Sort direction
   * - page?: number - Page number (1-indexed, default 1)
   * - pageSize?: number - Results per page (1-100, default 25)
   */
  getPatients: async (filters?: PatientFilters): Promise<PatientListResponse> => {
    const params = {
      ...(filters?.status && { status: filters.status.join(',') }),
      ...(filters?.departmentIds && { departmentIds: filters.departmentIds.join(',') }),
      ...(filters?.admissionType && { admissionType: filters.admissionType.join(',') }),
      ...(filters?.admittedAfter && { admittedAfter: filters.admittedAfter }),
      ...(filters?.admittedBefore && { admittedBefore: filters.admittedBefore }),
      ...(filters?.criticalVitals !== undefined && { criticalVitals: filters.criticalVitals }),
      sortBy: filters?.sortBy || 'updatedAt',
      sortOrder: filters?.sortOrder || 'desc',
      page: filters?.page || 1,
      pageSize: filters?.pageSize || 25,
    };

    const response = await apiClient.get<PatientListResponse>('/patients', params);

    // Validate patient data
    const validatedPatients = patientListSchema.parse(response.data);

    return {
      ...response,
      data: validatedPatients,
    };
  },

  /**
   * GET /api/patients/:id
   *
   * Fetches detailed information for a specific patient.
   * Includes full medical history, current medications, and vital signs.
   */
  getPatientDetail: async (patientId: string): Promise<Patient> => {
    const response = await apiClient.get<{ data: Patient }>(`/patients/${patientId}`);
    return patientSchema.parse(response.data);
  },
};
```

---

## 4. TanStack Query Integration

### 4.1 Query Keys Structure

**File**: `/src/lib/constants/queryKeys.ts` (extend existing)

```typescript
export const queryKeys = {
  // ... existing keys ...

  patients: {
    all: ['patients'] as const,
    lists: () => [...queryKeys.patients.all, 'list'] as const,
    list: (filters?: Record<string, unknown>) =>
      [...queryKeys.patients.lists(), filters] as const,
    details: () => [...queryKeys.patients.all, 'detail'] as const,
    detail: (id: string) => [...queryKeys.patients.details(), id] as const,
    search: (query?: string) => [...queryKeys.patients.all, 'search', query] as const,
  },
};
```

### 4.2 Custom Hooks

#### usePatients Hook

**File**: `/src/features/patients/hooks/usePatients.ts`

```typescript
import { useQuery } from '@tanstack/react-query';
import { queryKeys } from '@/lib/constants/queryKeys';
import { patientsApi } from '../api/patientsApi';
import type { PatientFilters } from '../types/patient.types';

/**
 * TanStack Query hook for patient list
 *
 * Fetches paginated list of patients with filters and search.
 * Automatically handles caching, refetching, and error states.
 *
 * Stale time: 30 seconds (moderate real-time updates)
 * Cache time: 5 minutes (keep recent data)
 * Refetch on window focus: true (refresh when tab regains focus)
 */
export function usePatients(filters?: PatientFilters) {
  return useQuery({
    queryKey: queryKeys.patients.list(filters),
    queryFn: () => patientsApi.getPatients(filters),
    staleTime: 30000, // 30 seconds
    gcTime: 300000, // 5 minutes (formerly cacheTime)
    refetchOnWindowFocus: true,
    refetchOnMount: true,
    enabled: true, // Can be disabled conditionally
  });
}
```

#### usePatientDetail Hook

**File**: `/src/features/patients/hooks/usePatientDetail.ts`

```typescript
import { useQuery } from '@tanstack/react-query';
import { queryKeys } from '@/lib/constants/queryKeys';
import { patientsApi } from '../api/patientsApi';

/**
 * TanStack Query hook for patient detail view
 *
 * Fetches comprehensive patient information including medications, vital signs, etc.
 * Enables when patientId is provided (conditional fetching).
 *
 * Stale time: 15 seconds (fresher for detail view)
 * Cache time: 5 minutes
 */
export function usePatientDetail(patientId: string | null) {
  return useQuery({
    queryKey: queryKeys.patients.detail(patientId || ''),
    queryFn: () => patientsApi.getPatientDetail(patientId!),
    staleTime: 15000, // 15 seconds
    gcTime: 300000, // 5 minutes
    enabled: !!patientId, // Only fetch when patientId is provided
    refetchOnWindowFocus: true,
  });
}
```

#### usePatientFilters Hook (State Management)

**File**: `/src/features/patients/hooks/usePatientFilters.ts`

```typescript
import { useCallback } from 'react';
import { useSearchParams } from 'react-router-dom';
import { patientFiltersSchema } from '../schemas/patientFilters.schema';
import type { PatientFilters } from '../types/patient.types';

/**
 * Hook for managing patient filter state via URL search params
 *
 * Persists filter state in URL so:
 * - Users can bookmark/share filtered views
 * - Filters persist on page refresh
 * - Back button works intuitively
 */
export function usePatientFilters() {
  const [searchParams, setSearchParams] = useSearchParams();

  // Parse current filters from URL
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

  // Update filters function
  const setFilters = useCallback(
    (newFilters: Partial<PatientFilters>) => {
      const updatedFilters = { ...filters, ...newFilters };
      const validated = patientFiltersSchema.parse(updatedFilters);

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
    },
    [filters, setSearchParams],
  );

  // Reset filters
  const resetFilters = useCallback(() => {
    setSearchParams(new URLSearchParams());
  }, [setSearchParams]);

  return { filters, setFilters, resetFilters };
}
```

#### usePatientSearch Hook

**File**: `/src/features/patients/hooks/usePatientSearch.ts`

```typescript
import { useState, useCallback, useRef, useEffect } from 'react';

/**
 * Hook for managing patient search state with debouncing
 *
 * Handles search term input and debounces to avoid excessive API calls.
 * Integrates with URL params via usePatientFilters.
 */
export function usePatientSearch(onSearch: (query: string) => void, debounceMs = 300) {
  const [searchTerm, setSearchTerm] = useState('');
  const debounceTimerRef = useRef<NodeJS.Timeout>();

  useEffect(() => {
    // Clear previous timer
    if (debounceTimerRef.current) {
      clearTimeout(debounceTimerRef.current);
    }

    // Set new debounced search
    debounceTimerRef.current = setTimeout(() => {
      onSearch(searchTerm);
    }, debounceMs);

    return () => {
      if (debounceTimerRef.current) {
        clearTimeout(debounceTimerRef.current);
      }
    };
  }, [searchTerm, onSearch, debounceMs]);

  const handleSearchChange = useCallback((value: string) => {
    setSearchTerm(value);
  }, []);

  const clearSearch = useCallback(() => {
    setSearchTerm('');
  }, []);

  return {
    searchTerm,
    handleSearchChange,
    clearSearch,
  };
}
```

---

## 5. TanStack Table Configuration

### 5.1 Column Definitions

**File**: `/src/features/patients/components/PatientTable.tsx` (excerpt)

```typescript
import { createColumnHelper } from '@tanstack/react-table';
import type { Patient } from '../types/patient.types';

const columnHelper = createColumnHelper<Patient>();

export const patientTableColumns = [
  // Patient ID + Name (primary identifier)
  columnHelper.accessor('mrn', {
    id: 'mrn',
    header: 'MRN',
    size: 100,
    cell: (info) => (
      <code className="text-sm font-mono text-gray-600">{info.getValue()}</code>
    ),
  }),

  columnHelper.accessor((row) => `${row.firstName} ${row.lastName}`, {
    id: 'name',
    header: 'Patient Name',
    size: 180,
    cell: (info) => {
      const name = info.getValue();
      return <span className="font-medium text-gray-900">{name}</span>;
    },
  }),

  // Age/DOB
  columnHelper.accessor('age', {
    id: 'age',
    header: 'Age',
    size: 70,
    cell: (info) => <span className="text-gray-600">{info.getValue()} y/o</span>,
  }),

  // Status (with color coding)
  columnHelper.accessor('status', {
    id: 'status',
    header: 'Status',
    size: 110,
    cell: (info) => {
      const status = info.getValue();
      const colors = {
        critical: 'bg-red-100 text-red-800 border-red-300',
        stable: 'bg-green-100 text-green-800 border-green-300',
        observation: 'bg-blue-100 text-blue-800 border-blue-300',
        recovery: 'bg-amber-100 text-amber-800 border-amber-300',
        'pending-discharge': 'bg-purple-100 text-purple-800 border-purple-300',
      };
      return (
        <span
          className={`inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium border ${colors[status]}`}
        >
          {status.replace('-', ' ')}
        </span>
      );
    },
  }),

  // Location
  columnHelper.accessor('departmentName', {
    id: 'department',
    header: 'Department',
    size: 140,
    cell: (info) => <span className="text-gray-600">{info.getValue()}</span>,
  }),

  columnHelper.accessor('bedNumber', {
    id: 'bed',
    header: 'Bed',
    size: 90,
    cell: (info) => (
      <span className="font-mono text-gray-700">{info.getValue()}</span>
    ),
  }),

  // Length of Stay
  columnHelper.accessor('lengthOfStay', {
    id: 'lengthOfStay',
    header: 'LOS',
    size: 80,
    cell: (info) => {
      const hours = info.getValue();
      const days = Math.floor(hours / 24);
      const remainingHours = hours % 24;
      return (
        <span className="text-gray-600 text-sm">
          {days > 0 ? `${days}d ${remainingHours}h` : `${remainingHours}h`}
        </span>
      );
    },
  }),

  // Vital Signs Summary
  columnHelper.accessor('vitalSigns', {
    id: 'vitals',
    header: 'Vitals',
    size: 150,
    cell: (info) => {
      const vitals = info.getValue();
      const abnormal = false; // Logic to check if vitals are abnormal
      return (
        <span
          className={`text-xs font-mono ${abnormal ? 'text-red-600 font-bold' : 'text-gray-600'}`}
        >
          HR: {vitals.heartRate} | O₂: {vitals.oxygenSaturation}%
        </span>
      );
    },
  }),

  // Physician
  columnHelper.accessor('admittingPhysician', {
    id: 'physician',
    header: 'Physician',
    size: 140,
    cell: (info) => <span className="text-gray-600">{info.getValue()}</span>,
  }),

  // Admission Type
  columnHelper.accessor('admissionType', {
    id: 'admissionType',
    header: 'Admission',
    size: 100,
    cell: (info) => (
      <span className="text-xs font-medium text-gray-600 uppercase tracking-wide">
        {info.getValue()}
      </span>
    ),
  }),

  // Action (expand detail)
  columnHelper.display({
    id: 'actions',
    header: 'Details',
    size: 80,
    cell: ({ row }) => (
      <button
        onClick={() => row.getToggleExpandedHandler()?.()}
        className="inline-flex items-center justify-center w-8 h-8 text-gray-400 hover:text-primary-600 hover:bg-gray-100 rounded-md transition"
        aria-label={`View details for ${row.original.firstName} ${row.original.lastName}`}
      >
        <ChevronRightIcon className="w-4 h-4" />
      </button>
    ),
  }),
];
```

### 5.2 Table Configuration

```typescript
import { useReactTable, getCoreRowModel, getSortedRowModel, getFilteredRowModel, getPaginationRowModel } from '@tanstack/react-table';

export function PatientTable({ data, isLoading, filters, onFiltersChange }: PatientTableProps) {
  const [sorting, setSorting] = useState<SortingState>([
    {
      id: filters?.sortBy || 'updatedAt',
      desc: filters?.sortOrder === 'desc',
    },
  ]);
  const [columnVisibility, setColumnVisibility] = useState<VisibilityState>({});
  const [rowSelection, setRowSelection] = useState({});

  const table = useReactTable({
    data: data || [],
    columns: patientTableColumns,
    state: {
      sorting,
      columnVisibility,
      rowSelection,
      pagination: {
        pageIndex: (filters?.page || 1) - 1,
        pageSize: filters?.pageSize || 25,
      },
    },
    onSortingChange: setSorting,
    onColumnVisibilityChange: setColumnVisibility,
    onRowSelectionChange: setRowSelection,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    manualPagination: false, // Client-side pagination (or true for server-side)
  });

  return (
    <div className="overflow-x-auto">
      <table className="w-full border-collapse">
        <thead>
          {table.getHeaderGroups().map((headerGroup) => (
            <tr key={headerGroup.id}>
              {headerGroup.headers.map((header) => (
                <th
                  key={header.id}
                  className="px-4 py-3 text-left text-xs font-semibold text-gray-700 bg-gray-50 border-b"
                  style={{ width: header.getSize() }}
                >
                  {header.isPlaceholder ? null : (
                    <div
                      {...{
                        className: header.column.getCanSort() ? 'cursor-pointer select-none' : '',
                        onClick: header.column.getToggleSortingHandler(),
                      }}
                    >
                      {flexRender(header.column.columnDef.header, header.getContext())}
                      {{
                        asc: ' 🔼',
                        desc: ' 🔽',
                      }[header.column.getIsSorted() as string] ?? null}
                    </div>
                  )}
                </th>
              ))}
            </tr>
          ))}
        </thead>
        <tbody>
          {table.getRowModel().rows.map((row) => (
            <tr key={row.id} className="border-b hover:bg-gray-50 transition">
              {row.getVisibleCells().map((cell) => (
                <td
                  key={cell.id}
                  className="px-4 py-3 text-sm"
                  style={{ width: cell.column.getSize() }}
                >
                  {flexRender(cell.column.columnDef.cell, cell.getContext())}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

---

## 6. Component Architecture

### 6.1 Component Hierarchy Diagram

```
PatientsPage (Orchestrator)
├── PatientFilters (Sidebar/Top)
│   ├── Status Filter (Multi-select)
│   ├── Department Filter (Multi-select)
│   ├── Admission Type Filter (Multi-select)
│   ├── Date Range Picker
│   ├── Critical Vitals Toggle
│   └── Reset Button
├── PatientSearch (Search bar)
│   └── Input with debouncing
├── PatientTable (Main content)
│   ├── PatientRow (for each row)
│   │   ├── PatientCell (MRN)
│   │   ├── PatientCell (Name)
│   │   ├── PatientCell (Status badge)
│   │   ├── PatientCell (Department)
│   │   ├── PatientCell (Bed)
│   │   ├── PatientCell (Vitals)
│   │   └── PatientCell (Actions)
│   ├── Pagination Controls
│   └── Results Summary
└── PatientDetailDrawer (Slide-out)
    ├── Header (patient name, MRN)
    ├── Demographics Section
    ├── Vital Signs Section
    ├── Medications Section
    ├── Care Team Section
    ├── Admission Info Section
    └── Footer (action buttons)
```

### 6.2 Component Specifications

#### PatientsPage (Container/Orchestrator)

**File**: `/src/features/patients/components/PatientsPage.tsx`

Responsibilities:
- Fetch patient data using hooks
- Manage filter/search state via URL params
- Coordinate detail drawer state
- Handle loading/error/empty states
- Orchestrate child components

**Key Props**: None (uses hooks exclusively)

**Key State**:
- `detailDrawer: { isOpen: boolean; patientId: string | null }`
- Uses `usePatients()`, `usePatientFilters()`, `usePatientSearch()`

```typescript
export function PatientsPage() {
  // Get filter state from URL
  const { filters, setFilters, resetFilters } = usePatientFilters();

  // Fetch patient data
  const { data, isLoading, error, refetch } = usePatients(filters);

  // Detail drawer state
  const [detailDrawer, setDetailDrawer] = useState<DetailDrawerState>({
    isOpen: false,
    patientId: null,
  });

  // Search handling
  const { searchTerm, handleSearchChange, clearSearch } = usePatientSearch(
    (query) => setFilters({ ...filters, query, page: 1 }),
    300,
  );

  if (isLoading) return <PatientSkeleton />;
  if (error) return <PatientsError error={error} onRetry={refetch} />;

  return (
    <main className="flex h-full gap-6 p-6">
      {/* Filters sidebar */}
      <PatientFilters
        filters={filters}
        onFiltersChange={setFilters}
        onResetFilters={resetFilters}
      />

      <div className="flex-1 flex flex-col gap-4">
        {/* Search and results header */}
        <div className="flex items-center justify-between">
          <div className="flex-1">
            <PatientSearch
              value={searchTerm}
              onChange={handleSearchChange}
              onClear={clearSearch}
            />
          </div>
          <div className="text-sm text-gray-600">
            {data?.pagination.total} patients
          </div>
        </div>

        {/* Patient table */}
        {data?.data.length ? (
          <>
            <PatientTable
              data={data.data}
              filters={filters}
              onFiltersChange={setFilters}
              onRowClick={(patient) =>
                setDetailDrawer({ isOpen: true, patientId: patient.id })
              }
            />
            {/* Pagination */}
            <PatientPagination
              pagination={data.pagination}
              onPageChange={(page) => setFilters({ ...filters, page })}
              onPageSizeChange={(pageSize) =>
                setFilters({ ...filters, pageSize, page: 1 })
              }
            />
          </>
        ) : (
          <EmptyState message="No patients found matching your criteria" />
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

#### PatientTable Component

**File**: `/src/features/patients/components/PatientTable.tsx`

Responsibilities:
- Render TanStack Table
- Handle sorting and row selection
- Display loading state (skeleton rows)
- Accessibility support (table semantics, ARIA)

#### PatientFilters Component

**File**: `/src/features/patients/components/PatientFilters.tsx`

Responsibilities:
- Render filter controls
- Update URL search params via `setFilters`
- Support multi-select dropdowns
- Date range picker

#### PatientSearch Component

**File**: `/src/features/patients/components/PatientSearch.tsx`

Responsibilities:
- Render search input
- Debounce search term
- Clear button functionality
- Search icon

#### PatientDetailDrawer Component

**File**: `/src/features/patients/components/PatientDetailDrawer.tsx`

Responsibilities:
- Slide-out panel for patient details
- Fetch patient data using `usePatientDetail` hook
- Sections: vitals, medications, care team, admission info
- Action buttons (discharge, transfer, etc.)
- Keyboard: Escape to close, Tab through sections
- Accessibility: `role="complementary"`, focus management

#### Supporting Components

- **PatientRow.tsx**: Wrapper for table rows with hover effects
- **PatientCell.tsx**: Individual cell component with status badges, formatted data
- **PatientSkeleton.tsx**: Loading skeleton matching table layout
- **PatientsError.tsx**: Error state with retry button
- **PatientPagination.tsx**: Pagination controls

---

## 7. State Management Strategy

### 7.1 Server State (TanStack Query)

**Management**: Patient data from API
```typescript
const { data, isLoading, error, refetch } = usePatients(filters);
```

**Configuration**:
- Stale time: 30 seconds
- Cache time (gcTime): 5 minutes
- Auto-refetch on window focus
- Refetch on mount

### 7.2 URL State (React Router)

**Management**: Filters, pagination, search persistence
```typescript
const { filters, setFilters, resetFilters } = usePatientFilters();
```

**Persisted to URL**:
- `?status=critical,stable`
- `?departmentIds=uuid1,uuid2`
- `?page=2&pageSize=50`
- `?sortBy=admissionDate&sortOrder=asc`

### 7.3 Local State (useState)

**Management**: UI-only state
```typescript
// Detail drawer
const [detailDrawer, setDetailDrawer] = useState<DetailDrawerState>({
  isOpen: false,
  patientId: null,
});

// Table column visibility
const [columnVisibility, setColumnVisibility] = useState<VisibilityState>({});

// Row selection
const [rowSelection, setRowSelection] = useState({});
```

### 7.4 Search State (Custom Hook)

**Management**: Search term with debouncing
```typescript
const { searchTerm, handleSearchChange, clearSearch } = usePatientSearch(
  (query) => setFilters({ ...filters, query, page: 1 }),
  300, // debounce delay
);
```

**No Zustand needed** - state is adequately managed through URL params and hooks

---

## 8. Mock Data Strategy

### 8.1 Mock Data Generator

**File**: `/src/mocks/data/patients.mock.ts`

```typescript
/**
 * Mock Data Generators for Patients
 *
 * Generates realistic synthetic healthcare data for development and testing.
 * All data is fictional and does not represent real patients or facilities.
 */

import type { Patient, Medication } from '@/features/patients/types/patient.types';

/**
 * Helper functions
 */
const random = (min: number, max: number): number =>
  Math.floor(Math.random() * (max - min + 1)) + min;

const randomFloat = (min: number, max: number, decimals = 1): number =>
  Number((Math.random() * (max - min) + min).toFixed(decimals));

const getTimestamp = (hoursAgo = 0): string => {
  const date = new Date();
  date.setHours(date.getHours() - hoursAgo);
  return date.toISOString();
};

/**
 * Generate realistic first and last names
 */
const firstNames = [
  'Sarah', 'Michael', 'Emily', 'James', 'Jennifer', 'Robert', 'Jessica', 'William',
  'Maria', 'David', 'Linda', 'Richard', 'Patricia', 'Charles', 'Margaret', 'Christopher',
];

const lastNames = [
  'Johnson', 'Smith', 'Williams', 'Brown', 'Jones', 'Garcia', 'Miller', 'Davis',
  'Rodriguez', 'Martinez', 'Hernandez', 'Lopez', 'Gonzalez', 'Wilson', 'Anderson', 'Chen',
];

function generateName(): { firstName: string; lastName: string } {
  return {
    firstName: firstNames[random(0, firstNames.length - 1)],
    lastName: lastNames[random(0, lastNames.length - 1)],
  };
}

/**
 * Generate mock vital signs
 */
function generateVitalSigns() {
  return {
    heartRate: random(50, 120),
    bloodPressureSystolic: random(90, 160),
    bloodPressureDiastolic: random(60, 100),
    temperature: randomFloat(36.5, 39.2),
    respiratoryRate: random(12, 25),
    oxygenSaturation: random(92, 100),
    timestamp: getTimestamp(0),
  };
}

/**
 * Generate mock medications
 */
function generateMedications(): Medication[] {
  const medicationNames = [
    'Lisinopril', 'Metoprolol', 'Atorvastatin', 'Omeprazole', 'Amoxicillin',
    'Ceftriaxone', 'Aspirin', 'Heparin', 'Insulin', 'Metformin',
    'Levothyroxine', 'Furosemide', 'Sertraline', 'Ibuprofen', 'Acetaminophen',
  ];

  return Array.from({ length: random(1, 4) }, (_, i) => ({
    id: crypto.randomUUID(),
    name: medicationNames[random(0, medicationNames.length - 1)],
    dosage: `${random(250, 1000)}mg`,
    frequency: 'twice-daily' as const,
    startedAt: getTimestamp(random(2, 48)),
    stoppedAt: null,
    route: 'oral' as const,
  }));
}

/**
 * Generate a single mock patient
 */
export function generateMockPatient(): Patient {
  const { firstName, lastName } = generateName();
  const mrn = `MRN${random(100000, 999999)}`;
  const admissionDate = new Date();
  admissionDate.setHours(admissionDate.getHours() - random(1, 48));

  const departments = [
    { id: crypto.randomUUID(), name: 'Intensive Care Unit' },
    { id: crypto.randomUUID(), name: 'Emergency Department' },
    { id: crypto.randomUUID(), name: 'Medical-Surgical' },
    { id: crypto.randomUUID(), name: 'Cardiology' },
    { id: crypto.randomUUID(), name: 'Oncology' },
  ];
  const dept = departments[random(0, departments.length - 1)];

  const statuses = ['critical', 'stable', 'observation', 'recovery', 'pending-discharge'] as const;
  const status = statuses[random(0, statuses.length - 1)];

  const admissionTypes = ['emergency', 'urgent', 'scheduled', 'observation'] as const;
  const admissionType = admissionTypes[random(0, admissionTypes.length - 1)];

  const diagnosises = [
    'Pneumonia', 'Heart Failure', 'Acute Myocardial Infarction', 'Sepsis',
    'Stroke', 'Diabetes', 'Hypertension', 'Acute Kidney Injury', 'COPD', 'Asthma',
  ];
  const diagnosis = diagnosises[random(0, diagnosises.length - 1)];

  const physicians = [
    'Dr. Sarah Chen', 'Dr. Michael Johnson', 'Dr. Emily Davis',
    'Dr. James Wilson', 'Dr. Patricia Martinez', 'Dr. Robert Rodriguez',
  ];
  const physician = physicians[random(0, physicians.length - 1)];

  return {
    id: crypto.randomUUID(),
    mrn,
    firstName,
    lastName,
    dateOfBirth: new Date(
      random(1940, 2010),
      random(0, 11),
      random(1, 28),
    )
      .toISOString()
      .split('T')[0],
    age: random(18, 95),
    gender: random(0, 1) ? 'male' : 'female',
    phone: `555-${random(100, 999)}-${random(1000, 9999)}`,
    email: `${firstName.toLowerCase()}.${lastName.toLowerCase()}@example.com`,
    admissionDate: admissionDate.toISOString(),
    admissionType,
    admittingPhysician: physician,
    primaryDiagnosis: diagnosis,
    departmentId: dept.id,
    departmentName: dept.name,
    bedNumber: `${dept.name.split('-')[0]}-${random(1, 50)}`,
    roomNumber: `${random(100, 999)}`,
    status,
    vitalSigns: generateVitalSigns(),
    allergies: random(0, 2) === 0 ? ['Penicillin', 'Sulfa'] : null,
    medications: generateMedications(),
    lengthOfStay: random(1, 48),
    expectedDischargeDate: new Date(Date.now() + random(1, 7) * 24 * 60 * 60 * 1000).toISOString(),
    assignedNurses: [
      { id: crypto.randomUUID(), name: 'Nurse ' + ['Alice', 'Bob', 'Carol', 'David'][random(0, 3)] },
      { id: crypto.randomUUID(), name: 'Nurse ' + ['Eve', 'Frank', 'Grace', 'Henry'][random(0, 3)] },
    ],
    createdAt: admissionDate.toISOString(),
    updatedAt: getTimestamp(random(0, 2)),
  };
}

/**
 * Generate mock patient list
 */
export function generateMockPatients(count = 75): Patient[] {
  return Array.from({ length: count }, () => generateMockPatient());
}

// Pre-generated mock data
export const mockPatients = generateMockPatients(75);
```

### 8.2 MSW Handlers

**File**: `/src/mocks/handlers/patients.handlers.ts`

```typescript
import { delay, HttpResponse, http } from 'msw';
import { generateMockPatients } from '../data/patients.mock';

/**
 * MSW Request Handlers for Patients API
 *
 * Intercepts and mocks all patient-related API requests.
 * Provides realistic synthetic healthcare data with network latency simulation.
 */

// Persistent mock data (regenerated on each page load)
const mockPatients = generateMockPatients(75);

export const patientsHandlers = [
  /**
   * GET /api/patients
   * Fetch patient list with filtering, searching, and pagination
   */
  http.get('/api/patients', async ({ request }) => {
    await delay(400); // Simulate network latency

    const url = new URL(request.url);

    // Parse query parameters
    const statusFilter = url.searchParams.get('status')?.split(',') || [];
    const departmentFilter = url.searchParams.get('departmentIds')?.split(',') || [];
    const admissionTypeFilter = url.searchParams.get('admissionType')?.split(',') || [];
    const query = url.searchParams.get('query') || '';
    const sortBy = url.searchParams.get('sortBy') || 'updatedAt';
    const sortOrder = url.searchParams.get('sortOrder') || 'desc';
    const page = Number.parseInt(url.searchParams.get('page') || '1', 10);
    const pageSize = Number.parseInt(url.searchParams.get('pageSize') || '25', 10);

    // Apply filters
    let filtered = [...mockPatients];

    if (statusFilter.length > 0) {
      filtered = filtered.filter((p) => statusFilter.includes(p.status));
    }

    if (departmentFilter.length > 0) {
      filtered = filtered.filter((p) => departmentFilter.includes(p.departmentId));
    }

    if (admissionTypeFilter.length > 0) {
      filtered = filtered.filter((p) => admissionTypeFilter.includes(p.admissionType));
    }

    if (query.length > 0) {
      const lowerQuery = query.toLowerCase();
      filtered = filtered.filter(
        (p) =>
          p.firstName.toLowerCase().includes(lowerQuery) ||
          p.lastName.toLowerCase().includes(lowerQuery) ||
          p.mrn.toLowerCase().includes(lowerQuery) ||
          p.primaryDiagnosis.toLowerCase().includes(lowerQuery) ||
          p.bedNumber.toLowerCase().includes(lowerQuery),
      );
    }

    // Apply sorting
    filtered.sort((a, b) => {
      let aVal: any;
      let bVal: any;

      switch (sortBy) {
        case 'name':
          aVal = `${a.firstName} ${a.lastName}`;
          bVal = `${b.firstName} ${b.lastName}`;
          break;
        case 'mrn':
          aVal = a.mrn;
          bVal = b.mrn;
          break;
        case 'admissionDate':
          aVal = new Date(a.admissionDate).getTime();
          bVal = new Date(b.admissionDate).getTime();
          break;
        case 'status':
          aVal = a.status;
          bVal = b.status;
          break;
        case 'department':
          aVal = a.departmentName;
          bVal = b.departmentName;
          break;
        case 'lengthOfStay':
          aVal = a.lengthOfStay;
          bVal = b.lengthOfStay;
          break;
        case 'updatedAt':
        default:
          aVal = new Date(a.updatedAt).getTime();
          bVal = new Date(b.updatedAt).getTime();
      }

      if (aVal < bVal) return sortOrder === 'asc' ? -1 : 1;
      if (aVal > bVal) return sortOrder === 'asc' ? 1 : -1;
      return 0;
    });

    // Apply pagination
    const total = filtered.length;
    const totalPages = Math.ceil(total / pageSize);
    const startIndex = (page - 1) * pageSize;
    const paginatedData = filtered.slice(startIndex, startIndex + pageSize);

    return HttpResponse.json({
      data: paginatedData,
      pagination: {
        page,
        pageSize,
        total,
        totalPages,
        hasNextPage: page < totalPages,
        hasPrevPage: page > 1,
      },
      metadata: {
        timestamp: new Date().toISOString(),
        count: paginatedData.length,
      },
    });
  }),

  /**
   * GET /api/patients/:id
   * Fetch detailed information for a specific patient
   */
  http.get('/api/patients/:id', async ({ params }) => {
    await delay(300);

    const { id } = params;
    const patient = mockPatients.find((p) => p.id === id);

    if (!patient) {
      return HttpResponse.json(
        { error: 'Patient not found', code: 'NOT_FOUND' },
        { status: 404 },
      );
    }

    return HttpResponse.json({
      data: patient,
      metadata: {
        timestamp: new Date().toISOString(),
        version: 1,
      },
    });
  }),
];
```

### 8.3 Register Handlers

**File**: `/src/mocks/handlers/index.ts` (update)

```typescript
import { dashboardHandlers } from './dashboard.handlers';
import { patientsHandlers } from './patients.handlers';

export const handlers = [...dashboardHandlers, ...patientsHandlers];
```

---

## 9. Performance Considerations

### 9.1 Table Virtualization

For tables with >50 rows, implement virtualization:

```typescript
// Use TanStack Table with @tanstack/react-table-windowed-rows
// OR use react-window for row virtualization

// Conditional virtualization based on row count
const shouldVirtualize = data && data.length > 50;

return shouldVirtualize ? (
  <VirtualizedPatientTable data={data} />
) : (
  <PatientTable data={data} />
);
```

### 9.2 Memoization Strategy

```typescript
// Memoize heavy computations
const patientTableColumns = useMemo(() => patientTableColumns, []);

// Memoize component to prevent re-renders
export const PatientRow = memo(function PatientRow({ row, onSelect }: Props) {
  // ...
});

// Memoize callbacks
const handleRowClick = useCallback((patient: Patient) => {
  setDetailDrawer({ isOpen: true, patientId: patient.id });
}, []);
```

### 9.3 Code Splitting

```typescript
// Lazy load detail drawer (heavy component)
const PatientDetailDrawer = lazy(
  () => import('./PatientDetailDrawer'),
);

// Lazy load filter panel
const PatientFilters = lazy(() => import('./PatientFilters'));
```

### 9.4 Auto-Refresh Configuration

```typescript
// RefreshInterval: 30 seconds (configurable)
// Pause when tab not visible
// Resume on focus

export function usePatients(filters?: PatientFilters) {
  return useQuery({
    queryKey: queryKeys.patients.list(filters),
    queryFn: () => patientsApi.getPatients(filters),
    staleTime: 25000,
    refetchInterval: 30000,
    refetchIntervalInBackground: false, // Don't refresh hidden tab
    refetchOnWindowFocus: true,
  });
}
```

---

## 10. Accessibility Requirements

### 10.1 Semantic HTML

```typescript
// Use semantic elements
<main className="space-y-6 p-6">
  <header className="flex items-center justify-between">
    <h1 className="text-3xl font-bold">Patient Census</h1>
  </header>

  <section aria-labelledby="filters-heading">
    <h2 id="filters-heading" className="sr-only">Patient Filters</h2>
    <PatientFilters />
  </section>

  <section aria-labelledby="results-heading">
    <h2 id="results-heading" className="sr-only">Patient Results</h2>
    <PatientTable />
  </section>
</main>
```

### 10.2 Keyboard Navigation

- Tab through filters
- Tab through table rows
- Enter to expand row detail
- Escape to close detail drawer
- Arrow keys for sort/filter selection

```typescript
// Handle keyboard events
const handleKeyDown = (e: KeyboardEvent) => {
  if (e.key === 'Escape') {
    setDetailDrawer({ isOpen: false, patientId: null });
  }
};
```

### 10.3 Screen Reader Support

```typescript
// ARIA labels for icon buttons
<button
  aria-label="View patient details"
  className="..."
>
  <ChevronRightIcon className="w-4 h-4" />
</button>

// Live region for updates
<div
  role="status"
  aria-live="polite"
  aria-atomic="true"
  className="sr-only"
>
  {rowCount} patients found
</div>

// Table with accessible headers
<table role="table">
  <thead>
    <tr role="row">
      <th role="columnheader">MRN</th>
      <th role="columnheader">Name</th>
    </tr>
  </thead>
</table>
```

### 10.4 Focus Management

```typescript
// Focus drawer on open
const drawerRef = useRef<HTMLDivElement>(null);

useEffect(() => {
  if (isOpen && drawerRef.current) {
    drawerRef.current.focus();
  }
}, [isOpen]);

// Focus first input in filters
const firstFilterRef = useRef<HTMLInputElement>(null);
// Use in modal/drawer context
```

### 10.5 Color Contrast

Ensure ≥4.5:1 contrast for text:
- Status badges with semantic colors
- Table text in dark gray on white
- Interactive elements with clear focus states

---

## 11. Testing Strategy

### 11.1 Unit Tests (Hooks)

**File**: `/src/features/patients/hooks/__tests__/usePatients.test.ts`

```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { usePatients } from '../usePatients';

describe('usePatients', () => {
  it('fetches patient list', async () => {
    const wrapper = ({ children }: { children: React.ReactNode }) => (
      <QueryClientProvider client={new QueryClient()}>
        {children}
      </QueryClientProvider>
    );

    const { result } = renderHook(() => usePatients(), { wrapper });

    await waitFor(() => {
      expect(result.current.isLoading).toBe(false);
    });

    expect(result.current.data?.data).toHaveLength(25);
  });
});
```

### 11.2 Component Tests

**File**: `/src/features/patients/components/__tests__/PatientsPage.test.tsx`

```typescript
import { render, screen, userEvent } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { PatientsPage } from '../PatientsPage';

describe('PatientsPage', () => {
  it('renders patient list', async () => {
    const client = new QueryClient();

    render(
      <QueryClientProvider client={client}>
        <BrowserRouter>
          <PatientsPage />
        </BrowserRouter>
      </QueryClientProvider>,
    );

    expect(screen.getByText(/patient census/i)).toBeInTheDocument();
    expect(await screen.findByRole('table')).toBeInTheDocument();
  });

  it('opens detail drawer on row click', async () => {
    const user = userEvent.setup();
    // ... render and test
  });
});
```

### 11.3 Accessibility Tests

**File**: `/src/features/patients/components/__tests__/patients-accessibility.test.tsx`

```typescript
import { axe, toHaveNoViolations } from 'jest-axe';

describe('Patient Census Accessibility', () => {
  it('has no accessibility violations', async () => {
    const { container } = render(<PatientsPage />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });

  it('supports keyboard navigation', async () => {
    const user = userEvent.setup();
    // Test Tab, Enter, Escape
  });
});
```

---

## 12. Implementation Checklist

### Phase 1: Core Setup
- [ ] Create directory structure
- [ ] Define Zod schemas
- [ ] Create type exports
- [ ] Implement API module
- [ ] Create mock data generators
- [ ] Register MSW handlers

### Phase 2: Hooks & Queries
- [ ] Implement `usePatients` hook
- [ ] Implement `usePatientDetail` hook
- [ ] Implement `usePatientFilters` hook
- [ ] Implement `usePatientSearch` hook
- [ ] Add query keys to factory

### Phase 3: Components
- [ ] PatientsPage (orchestrator)
- [ ] PatientTable (TanStack Table)
- [ ] PatientFilters (filter UI)
- [ ] PatientSearch (search box)
- [ ] PatientDetailDrawer (detail view)
- [ ] Loading/Error/Empty states

### Phase 4: Polish & Accessibility
- [ ] Keyboard navigation
- [ ] ARIA labels and roles
- [ ] Focus management
- [ ] Color contrast verification
- [ ] Screen reader testing

### Phase 5: Testing
- [ ] Hook tests
- [ ] Component tests
- [ ] Accessibility tests
- [ ] E2E tests (Playwright)

---

## 13. API Request/Response Examples

### Example: GET /api/patients

**Request**:
```
GET /api/patients?status=critical,stable&departmentIds=uuid-1&sortBy=updatedAt&sortOrder=desc&page=1&pageSize=25
```

**Response**:
```json
{
  "data": [
    {
      "id": "uuid-1",
      "mrn": "MRN123456",
      "firstName": "Sarah",
      "lastName": "Johnson",
      "dateOfBirth": "1965-03-15",
      "age": 59,
      "gender": "female",
      "phone": "555-123-4567",
      "email": "sarah.johnson@example.com",
      "admissionDate": "2024-09-26T14:30:00Z",
      "admissionType": "emergency",
      "admittingPhysician": "Dr. Sarah Chen",
      "primaryDiagnosis": "Pneumonia",
      "departmentId": "uuid-dept-1",
      "departmentName": "Intensive Care Unit",
      "bedNumber": "ICU-12",
      "roomNumber": "123",
      "status": "critical",
      "vitalSigns": {
        "heartRate": 115,
        "bloodPressureSystolic": 145,
        "bloodPressureDiastolic": 88,
        "temperature": 38.5,
        "respiratoryRate": 22,
        "oxygenSaturation": 94,
        "timestamp": "2024-09-27T10:00:00Z"
      },
      "allergies": ["Penicillin"],
      "medications": [
        {
          "id": "med-1",
          "name": "Ceftriaxone",
          "dosage": "1000mg",
          "frequency": "twice-daily",
          "startedAt": "2024-09-26T14:30:00Z",
          "stoppedAt": null,
          "route": "iv"
        }
      ],
      "lengthOfStay": 24,
      "expectedDischargeDate": "2024-10-01T10:00:00Z",
      "assignedNurses": [
        {
          "id": "nurse-1",
          "name": "Nurse Alice"
        }
      ],
      "createdAt": "2024-09-26T14:30:00Z",
      "updatedAt": "2024-09-27T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 25,
    "total": 75,
    "totalPages": 3,
    "hasNextPage": true,
    "hasPrevPage": false
  },
  "metadata": {
    "timestamp": "2024-09-27T10:00:00Z",
    "count": 25
  }
}
```

### Example: GET /api/patients/:id

**Request**:
```
GET /api/patients/uuid-1
```

**Response**:
```json
{
  "data": { /* full patient object */ },
  "metadata": {
    "timestamp": "2024-09-27T10:00:00Z",
    "version": 1
  }
}
```

---

## 14. Summary

This specification provides a complete, production-ready architecture for the Patient Census feature following CareFlow's established patterns:

### Key Characteristics:
1. **Feature-first organization** - All artifacts colocated in `src/features/patients/`
2. **Type safety via Zod** - Single source of truth for validation and types
3. **Server state with TanStack Query** - Efficient caching and refetching
4. **URL-based persistence** - Filters survive page refresh and back button
5. **Accessibility-first** - WCAG 2.2 AA compliance built-in
6. **Comprehensive testing** - Unit, component, and E2E strategies
7. **Mock data via MSW** - Realistic healthcare data without real API dependency
8. **Performance optimized** - Virtualization, memoization, code splitting ready
9. **Clean API boundary** - No data hardcoding in components
10. **Semantic HTML & ARIA** - Full keyboard navigation and screen reader support

The architecture scales from 25 to 1000+ patients efficiently and provides a solid foundation for future enhancements like batch operations, export, advanced filtering, and real-time updates via WebSockets.

