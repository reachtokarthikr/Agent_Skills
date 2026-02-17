---
name: react
description: Use this skill whenever the user wants to create, build, debug, or work with React applications. Triggers include any mention of 'React', 'JSX', 'TSX', 'Next.js', 'Vite', 'React hooks', 'Redux', 'Zustand', 'React Router', 'React Query', 'TanStack', or requests to build single-page applications, dashboards, forms, component libraries, or interactive UIs using React. Also use when refactoring class components to hooks, optimizing rendering performance, writing component tests with React Testing Library, implementing state management, building custom hooks, or setting up React projects with TypeScript. Do NOT use for vanilla JS/HTML projects, Vue, Angular, Svelte, or other non-React frameworks unless the user is migrating to/from React.
---

# React Development Skill

This skill guides the creation of modern, production-grade React applications following community best practices, React team recommendations, and TypeScript-first conventions.

The user provides React requirements: components, pages, applications, hooks, or full project setups. They may include context about styling preferences, state management needs, or deployment targets.

---

## Request Tracking — Two IDs Across Every Layer

Every request carries **two tracking IDs** from UI to API to DB and back:

| ID | Purpose | Who Generates | Where Stored |
|----|---------|---------------|--------------|
| **CorrelationId** | Groups all requests in a user flow/session | **UI** generates once per session | `sessionStorage` + sent via `X-Correlation-Id` header |
| **RequestId** | Identifies one API call | **API** generates per request | Read from `X-Request-Id` response header |

**On unhandled errors**: The UI shows a **friendly error screen** displaying the `RequestId` as "Reference ID" — never stack traces, never technical details. The customer gives this ID to support, who traces it through API logs → DB `log.RequestLog`.

---

## Environment Setup

```bash
node --version
npm --version
```

If not available:
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### Project Scaffolding

Prefer **Vite** for new projects:

```bash
# Vite + React + TypeScript (recommended)
npm create vite@latest my-app -- --template react-ts
cd my-app && npm install

# Next.js (for SSR/SSG/full-stack)
npx create-next-app@latest my-app --typescript --tailwind --eslint --app
```

```bash
# Additional packages for full-stack integration
npm install @tanstack/react-query axios zustand clsx tailwind-merge uuid sonner
npm install -D @types/uuid
```

---

## Project Structure — Feature-Based

```
src/
├── app/                       # App-level setup
│   ├── App.tsx
│   ├── routes.tsx
│   └── providers.tsx          ★ wraps QueryClient + ErrorBoundary + TrackingProvider
├── components/                # Shared UI components
│   ├── ui/                    # Primitives (Button, Input, Modal, DataTable)
│   │   ├── Button.tsx
│   │   ├── DataTable.tsx
│   │   └── index.ts
│   ├── layout/                # Header, Sidebar, Footer
│   └── error/
│       ├── ErrorBoundary.tsx  ★ catches render errors, shows RequestId
│       └── ErrorFallback.tsx  ★ friendly error screen with reference ID
├── features/                  # Feature modules (domain-grouped)
│   ├── users/
│   │   ├── components/        # UserForm, UserList, UserCard
│   │   ├── hooks/             # useUsers, useUserManage
│   │   ├── api.ts             # API calls (manage + get)
│   │   ├── types.ts
│   │   └── index.ts
│   ├── products/
│   └── orders/
├── hooks/                     # Shared custom hooks
│   └── useTracking.ts         ★ provides CorrelationId
├── lib/                       # Utilities, API client, constants
│   ├── api-client.ts          ★ attaches CorrelationId, reads RequestId
│   ├── tracking.ts            ★ CorrelationId generation + storage
│   ├── utils.ts
│   └── constants.ts
├── types/                     # Shared TypeScript types
│   └── common.ts              ★ ApiResponse, ApiError with tracking IDs
└── styles/
```

### Key Principles

- **Co-locate** — keep feature components, hooks, types, API calls together
- **Barrel exports** — `index.ts` controls the public API of each module
- **Flat over nested** — 2–3 directory levels max
- **One component per file** (except tiny helper components)

---

## TypeScript Standards

### tsconfig.json — Strict Mode

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "jsx": "react-jsx",
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

### Component Typing

```tsx
interface UserCardProps {
  user: User;
  onSelect?: (userId: number) => void;
  variant?: 'compact' | 'full';
  className?: string;
  children?: React.ReactNode;
}

export function UserCard({ user, onSelect, variant = 'full', className }: UserCardProps) {
  return (
    <div className={cn('user-card', className)} onClick={() => onSelect?.(user.id)}>
      {variant === 'full' && <Avatar src={user.avatarUrl} />}
      <span>{user.firstName} {user.lastName}</span>
    </div>
  );
}
```

### Type Best Practices

- **`interface` for props**, `type` for unions and utilities
- **No `React.FC`** — use plain functions with typed props
- **Discriminated unions** for state machines and variants
- **Never `any`** — use `unknown` and narrow
- **Export types** consumers need; keep internal types private

---

## Tracking System (UI Side)

### CorrelationId Manager (lib/tracking.ts)

```tsx
import { v4 as uuidv4 } from 'uuid';

const CORRELATION_KEY = 'x-correlation-id';

/** Get or create a CorrelationId for the current session. */
export function getCorrelationId(): string {
  let id = sessionStorage.getItem(CORRELATION_KEY);
  if (!id) {
    id = uuidv4();
    sessionStorage.setItem(CORRELATION_KEY, id);
  }
  return id;
}

/** Reset CorrelationId (e.g., on logout or new workflow). */
export function resetCorrelationId(): string {
  const id = uuidv4();
  sessionStorage.setItem(CORRELATION_KEY, id);
  return id;
}
```

### Tracking Hook (hooks/useTracking.ts)

```tsx
import { useMemo } from 'react';
import { getCorrelationId } from '@/lib/tracking';

export function useTracking() {
  const correlationId = useMemo(() => getCorrelationId(), []);
  return { correlationId };
}
```

---

## API Client — Sends CorrelationId, Reads RequestId

### Shared Types (types/common.ts)

```tsx
/** Standard API response from the .NET backend. Both IDs always present. */
export interface ApiResponse<T> {
  success: boolean;
  data: T;
  message?: string;
  correlationId: string;
  requestId: string;
}

export interface PagedApiResponse<T> extends ApiResponse<T[]> {
  totalRecords: number;
  pageNumber: number;
  pageSize: number;
  totalPages: number;
}

/** Matches PagedResult<T> from .NET for feature-level use. */
export interface PagedResult<T> {
  data: T[];
  totalRecords: number;
  pageNumber: number;
  pageSize: number;
  totalPages: number;
  hasPrevious: boolean;
  hasNext: boolean;
}

/** Custom error class that carries both tracking IDs. */
export class ApiError extends Error {
  public correlationId: string;
  public requestId: string;
  public statusCode: number;

  constructor(statusCode: number, message: string, correlationId: string, requestId: string) {
    super(message);
    this.name = 'ApiError';
    this.statusCode = statusCode;
    this.correlationId = correlationId;
    this.requestId = requestId;
  }
}

/** Base filter that matches _Get SP params. */
export interface FilterRequest {
  searchText?: string;
  isActive?: boolean | null;
  sortColumn?: string;
  sortDirection?: 'ASC' | 'DESC';
  pageNumber?: number;
  pageSize?: number;
}
```

### API Client (lib/api-client.ts)

```tsx
import { getCorrelationId } from './tracking';
import { ApiError } from '@/types/common';

const BASE_URL = import.meta.env.VITE_API_URL || '/api';

async function request<T>(url: string, options?: RequestInit): Promise<T> {
  const correlationId = getCorrelationId();

  const response = await fetch(`${BASE_URL}${url}`, {
    headers: {
      'Content-Type': 'application/json',
      'X-Correlation-Id': correlationId,         // ★ Send CorrelationId to API
      ...options?.headers,
    },
    ...options,
  });

  // ★ Read RequestId from response header (API always sets it)
  const requestId = response.headers.get('X-Request-Id') || 'unknown';

  if (!response.ok) {
    let errorMessage = 'An unexpected error occurred.';
    let errorCorrId = correlationId;
    let errorReqId = requestId;

    try {
      const errorBody = await response.json();
      errorMessage = errorBody.message || errorMessage;
      errorCorrId = errorBody.correlationId || errorCorrId;
      errorReqId = errorBody.requestId || errorReqId;
    } catch {
      // response body not JSON — use defaults
    }

    throw new ApiError(response.status, errorMessage, errorCorrId, errorReqId);
  }

  return response.json();
}

export const apiClient = {
  get: <T>(url: string) => request<T>(url),
  post: <T>(url: string, body: unknown) =>
    request<T>(url, { method: 'POST', body: JSON.stringify(body) }),
  put: <T>(url: string, body: unknown) =>
    request<T>(url, { method: 'PUT', body: JSON.stringify(body) }),
  delete: <T>(url: string) =>
    request<T>(url, { method: 'DELETE' }),
};
```

---

## Error UI — Friendly Screen with Reference ID

### ErrorFallback Component (components/error/ErrorFallback.tsx)

**CRITICAL**: On unhandled exceptions, show a **clean, professional error screen** with the RequestId. Never show stack traces, error objects, or technical details.

```tsx
import { ApiError } from '@/types/common';

interface ErrorFallbackProps {
  error: Error;
  resetError?: () => void;
}

export function ErrorFallback({ error, resetError }: ErrorFallbackProps) {
  const isApiError = error instanceof ApiError;
  const requestId = isApiError ? error.requestId : 'N/A';
  const correlationId = isApiError ? error.correlationId : 'N/A';

  return (
    <div className="flex min-h-[400px] items-center justify-center p-6">
      <div className="w-full max-w-md rounded-2xl border border-gray-200 bg-white p-8 text-center shadow-lg">
        {/* Icon */}
        <div className="mx-auto mb-4 flex h-16 w-16 items-center justify-center rounded-full bg-red-50">
          <svg className="h-8 w-8 text-red-500" fill="none" viewBox="0 0 24 24" strokeWidth={1.5} stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round"
              d="M12 9v3.75m-9.303 3.376c-.866 1.5.217 3.374 1.948 3.374h14.71c1.73 0 2.813-1.874 1.948-3.374L13.949 3.378c-.866-1.5-3.032-1.5-3.898 0L2.697 16.126ZM12 15.75h.007v.008H12v-.008Z" />
          </svg>
        </div>
        {/* Message */}
        <h2 className="mb-2 text-xl font-semibold text-gray-900">Something went wrong</h2>
        <p className="mb-6 text-sm text-gray-500">
          We're sorry for the inconvenience. Please try again or contact support with the reference ID below.
        </p>
        {/* ★ Reference ID — this is what the customer gives to support */}
        <div className="mb-6 rounded-lg bg-gray-50 px-4 py-3">
          <p className="mb-1 text-xs font-medium uppercase tracking-wider text-gray-400">Reference ID</p>
          <p className="select-all font-mono text-sm font-semibold text-gray-700">{requestId}</p>
        </div>
        {/* Actions */}
        <div className="flex flex-col gap-2">
          {resetError && (
            <button onClick={resetError}
              className="rounded-lg bg-blue-600 px-4 py-2.5 text-sm font-medium text-white hover:bg-blue-700">
              Try Again
            </button>
          )}
          <button onClick={() => window.location.reload()}
            className="rounded-lg border border-gray-200 px-4 py-2.5 text-sm font-medium text-gray-600 hover:bg-gray-50">
            Reload Page
          </button>
        </div>
        <p className="mt-4 text-xs text-gray-400">Session: {correlationId}</p>
      </div>
    </div>
  );
}
```

### ErrorBoundary (components/error/ErrorBoundary.tsx)

```tsx
import { Component, type ErrorInfo, type ReactNode } from 'react';
import { ErrorFallback } from './ErrorFallback';

interface Props { children: ReactNode; fallback?: (error: Error, resetError: () => void) => ReactNode; }
interface State { error: Error | null; }

export class ErrorBoundary extends Component<Props, State> {
  state: State = { error: null };
  static getDerivedStateFromError(error: Error): State { return { error }; }
  componentDidCatch(error: Error, info: ErrorInfo) { console.error('[ErrorBoundary]', error, info); }
  resetError = () => this.setState({ error: null });
  render() {
    if (this.state.error) {
      return this.props.fallback
        ? this.props.fallback(this.state.error, this.resetError)
        : <ErrorFallback error={this.state.error} resetError={this.resetError} />;
    }
    return this.props.children;
  }
}
```

---

## Error Display Standards

| Scenario | What User Sees | RequestId Visible? |
|----------|---------------|--------------------|
| **API 4xx** (validation, not found) | Toast notification with message | Yes — in toast subtitle |
| **API 5xx** (server error) | Toast + ErrorFallback card if page-level | Yes — prominent in card |
| **Network error** (offline, timeout) | Toast: "Connection error. Check your network." | No — no API response |
| **React render crash** | ErrorBoundary → ErrorFallback card | Yes — if ApiError, else "N/A" |

---

## Providers with Global Error Handling (app/providers.tsx)

```tsx
import { QueryClient, QueryClientProvider, MutationCache, QueryCache } from '@tanstack/react-query';
import { ErrorBoundary } from '@/components/error/ErrorBoundary';
import { ApiError } from '@/types/common';
import { toast } from 'sonner';

const queryClient = new QueryClient({
  defaultOptions: { queries: { retry: 1, staleTime: 5 * 60 * 1000 } },
  queryCache: new QueryCache({
    onError: (error) => {
      if (error instanceof ApiError) {
        toast.error(error.message, { description: `Reference: ${error.requestId}`, duration: 8000 });
      }
    },
  }),
  mutationCache: new MutationCache({
    onError: (error) => {
      if (error instanceof ApiError) {
        toast.error(error.message, { description: `Reference: ${error.requestId}`, duration: 8000 });
      }
    },
  }),
});

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <ErrorBoundary>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </ErrorBoundary>
  );
}
```

---

## Feature API Layer (features/users/api.ts)

```tsx
import { apiClient } from '@/lib/api-client';
import type { ApiResponse, PagedApiResponse, FilterRequest } from '@/types/common';
import type { User, UserManageRequest } from './types';

const URL = '/users';

export const usersApi = {
  // → EXEC app.Users_Get @Action='GETBYID'
  getById: (id: number) =>
    apiClient.get<ApiResponse<User>>(`${URL}/${id}`),

  // → EXEC app.Users_Get @Action='GETALL'/'GETFILTER'
  getList: (filter?: FilterRequest) => {
    const params = new URLSearchParams();
    if (filter?.searchText) params.set('searchText', filter.searchText);
    if (filter?.isActive !== undefined && filter.isActive !== null)
      params.set('isActive', String(filter.isActive));
    if (filter?.sortColumn) params.set('sortColumn', filter.sortColumn);
    if (filter?.sortDirection) params.set('sortDirection', filter.sortDirection);
    params.set('pageNumber', String(filter?.pageNumber ?? 1));
    params.set('pageSize', String(filter?.pageSize ?? 20));
    return apiClient.get<PagedApiResponse<User>>(`${URL}?${params}`);
  },

  // → EXEC app.Users_Manage @Action='INSERT'
  create: (data: UserManageRequest) =>
    apiClient.post<ApiResponse<User>>(URL, data),

  // → EXEC app.Users_Manage @Action='UPDATE'
  update: (id: number, data: Partial<UserManageRequest>) =>
    apiClient.put<ApiResponse<User>>(`${URL}/${id}`, data),

  // → EXEC app.Users_Manage @Action='DELETE'
  delete: (id: number) =>
    apiClient.delete<ApiResponse<{ deleted: boolean }>>(`${URL}/${id}`),
};
```

### Feature Types (features/users/types.ts)

```tsx
export interface User {
  id: number;
  firstName: string;
  lastName: string;
  email: string;
  phone?: string;
  isActive: boolean;
  createdBy?: number;
  createdAt: string;
  modifiedBy?: number;
  modifiedAt?: string;
}

export interface UserManageRequest {
  firstName: string;
  lastName: string;
  email: string;
  phone?: string;
  isActive?: boolean;
}

export interface UserFilter extends FilterRequest {}
```

---

## TanStack Query Hooks with Tracking

```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { usersApi } from '../api';
import type { UserFilter, UserManageRequest } from '../types';

const USERS_KEY = ['users'];

export function useUsers(filter?: UserFilter) {
  return useQuery({
    queryKey: [...USERS_KEY, 'list', filter],
    queryFn: () => usersApi.getList(filter),
    placeholderData: (prev) => prev,
    select: (response) => ({
      data: response.data,
      totalRecords: response.totalRecords,
      totalPages: response.totalPages,
      correlationId: response.correlationId,     // ★ available if needed
      requestId: response.requestId,
    }),
  });
}

export function useUser(id: number) {
  return useQuery({
    queryKey: [...USERS_KEY, 'detail', id],
    queryFn: () => usersApi.getById(id),
    enabled: id > 0,
    select: (response) => response.data,
  });
}

export function useUserManage() {
  const queryClient = useQueryClient();
  const invalidate = () => queryClient.invalidateQueries({ queryKey: USERS_KEY });

  const create = useMutation({
    mutationFn: (data: UserManageRequest) => usersApi.create(data),
    onSuccess: invalidate,
    // ★ onError handled globally by MutationCache — shows toast with RequestId
  });

  const update = useMutation({
    mutationFn: ({ id, data }: { id: number; data: Partial<UserManageRequest> }) =>
      usersApi.update(id, data),
    onSuccess: invalidate,
  });

  const remove = useMutation({
    mutationFn: (id: number) => usersApi.delete(id),
    onSuccess: invalidate,
  });

  return { create, update, remove };
}
```

---

## Feature Page Component

```tsx
import { useState } from 'react';
import { useUsers, useUserManage } from '../hooks';
import { ErrorFallback } from '@/components/error/ErrorFallback';
import type { UserFilter } from '../types';

export function UserListPage() {
  const [filter, setFilter] = useState<UserFilter>({
    pageNumber: 1, pageSize: 20, sortColumn: 'Id', sortDirection: 'ASC',
  });

  const { data, isLoading, error } = useUsers(filter);
  const { remove } = useUserManage();

  // ★ If query fails, show error screen with RequestId
  if (error) return <ErrorFallback error={error} resetError={() => window.location.reload()} />;

  const handleSearch = (searchText: string) =>
    setFilter(prev => ({ ...prev, searchText, pageNumber: 1 }));

  const handlePageChange = (pageNumber: number) =>
    setFilter(prev => ({ ...prev, pageNumber }));

  const handleSort = (sortColumn: string) =>
    setFilter(prev => ({
      ...prev, sortColumn,
      sortDirection: prev.sortColumn === sortColumn && prev.sortDirection === 'ASC' ? 'DESC' : 'ASC',
    }));

  const handleDelete = async (id: number) => {
    if (window.confirm('Are you sure?')) {
      await remove.mutateAsync(id);
      // ★ If fails, MutationCache.onError shows toast with RequestId automatically
    }
  };

  return (
    <div>
      <h1>Users</h1>
      <SearchBar onSearch={handleSearch} />
      <DataTable
        data={data?.data ?? []}
        isLoading={isLoading}
        columns={[
          { key: 'firstName', label: 'First Name', sortable: true },
          { key: 'lastName', label: 'Last Name', sortable: true },
          { key: 'email', label: 'Email', sortable: true },
          { key: 'isActive', label: 'Active', render: (val) => val ? 'Yes' : 'No' },
        ]}
        sortColumn={filter.sortColumn}
        sortDirection={filter.sortDirection}
        onSort={handleSort}
        onDelete={handleDelete}
      />
      <Pagination
        totalRecords={data?.totalRecords ?? 0}
        pageNumber={filter.pageNumber ?? 1}
        pageSize={filter.pageSize ?? 20}
        onPageChange={handlePageChange}
      />
    </div>
  );
}
```

---

## React Hooks & Patterns

### Core Hook Guidelines

```tsx
// useState — simple local state
const [count, setCount] = useState(0);
const [user, setUser] = useState<User | null>(null);

// useReducer — complex state with multiple sub-values
const [state, dispatch] = useReducer(formReducer, initialFormState);

// useEffect — side effects with cleanup
useEffect(() => {
  const controller = new AbortController();
  fetchData(controller.signal).then(setData).catch(handleError);
  return () => controller.abort();
}, [url]);

// useMemo — expensive computations only
const sorted = useMemo(() => items.toSorted((a, b) => a.name.localeCompare(b.name)), [items]);

// useCallback — stable references for memoized children
const handleSubmit = useCallback(async (data: FormData) => {
  await submitForm(data);
}, []);
```

### Custom Hook Rules

- **Prefix with `use`** — `useUser`, `useDebounce`, `useLocalStorage`
- **Return stable references** — memoize returned objects/functions
- **Handle cleanup** — cancel requests, unsubscribe, clear timers
- **One concern per hook**
- **Compose** — build complex hooks from simpler ones

---

## Component Patterns

### Composition Over Configuration

```tsx
// Good: Compound components
<Card>
  <Card.Header><Card.Title>Profile</Card.Title></Card.Header>
  <Card.Body><UserDetails user={user} /></Card.Body>
  <Card.Footer><Button onClick={onSave}>Save</Button></Card.Footer>
</Card>

// Avoid: Monolithic prop-heavy components
<Card title="Profile" body={<UserDetails />} footerActions={[...]} />
```

---

## State Management Decision Guide

| Scope                  | Solution                                   |
|------------------------|--------------------------------------------|
| Component-local        | `useState` / `useReducer`                  |
| Shared (few components)| Lift state up + props                      |
| Shared (deep tree)     | React Context + `useReducer`               |
| Server state (API)     | TanStack Query (React Query)               |
| Complex client state   | Zustand (simple) or Redux Toolkit (large)  |
| URL state              | React Router search params                 |
| Form state             | React Hook Form + Zod                      |

### Zustand (Client State)

```tsx
import { create } from 'zustand';

interface AppStore {
  theme: 'light' | 'dark';
  sidebarOpen: boolean;
  toggleTheme: () => void;
  toggleSidebar: () => void;
}

export const useAppStore = create<AppStore>()((set) => ({
  theme: 'light',
  sidebarOpen: true,
  toggleTheme: () => set((s) => ({ theme: s.theme === 'light' ? 'dark' : 'light' })),
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
}));
```

---

## Styling

### Tailwind CSS (Recommended)

```tsx
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) { return twMerge(clsx(inputs)); }

function Button({ variant = 'primary', className, children, ...props }: ButtonProps) {
  return (
    <button
      className={cn(
        'inline-flex items-center justify-center rounded-lg px-4 py-2 text-sm font-medium transition-colors',
        'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2',
        'disabled:pointer-events-none disabled:opacity-50',
        variant === 'primary' && 'bg-blue-600 text-white hover:bg-blue-700',
        variant === 'secondary' && 'bg-gray-100 text-gray-900 hover:bg-gray-200',
        className
      )}
      {...props}
    >{children}</button>
  );
}
```

---

## Performance Optimization

- **Don't optimize prematurely** — measure first with React DevTools Profiler
- **Keep state local** to avoid unnecessary re-renders
- **`React.memo`** only when profiling confirms a problem
- **Lazy load routes**:

```tsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./features/dashboard'));

function AppRoutes() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </Suspense>
  );
}
```

- **Virtualize long lists** with `@tanstack/react-virtual`

---

## Testing

### Setup

```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom msw
```

### Component Test

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { UserCard } from './UserCard';

describe('UserCard', () => {
  const mockUser = { id: 1, firstName: 'John', lastName: 'Doe', email: 'john@test.com' };

  it('renders user name', () => {
    render(<UserCard user={mockUser} />);
    expect(screen.getByText('John Doe')).toBeInTheDocument();
  });

  it('calls onSelect when clicked', async () => {
    const user = userEvent.setup();
    const onSelect = vi.fn();
    render(<UserCard user={mockUser} onSelect={onSelect} />);
    await user.click(screen.getByRole('button'));
    expect(onSelect).toHaveBeenCalledWith(1);
  });
});
```

### Test ErrorFallback Shows RequestId

```tsx
describe('ErrorFallback', () => {
  it('displays RequestId from ApiError', () => {
    const error = new ApiError(500, 'Server error', 'corr-123', 'req-456');
    render(<ErrorFallback error={error} />);
    expect(screen.getByText('Something went wrong')).toBeInTheDocument();
    expect(screen.getByText('req-456')).toBeInTheDocument();
    expect(screen.getByText(/Session: corr-123/)).toBeInTheDocument();
  });

  it('shows N/A for non-ApiError', () => {
    render(<ErrorFallback error={new Error('boom')} />);
    expect(screen.getByText('N/A')).toBeInTheDocument();
  });
});
```

### Testing Best Practices

- **Query by role/label** — `getByRole`, `getByLabelText` over `getByTestId`
- **userEvent over fireEvent** for real interaction simulation
- **Test behavior, not implementation**
- **MSW** for realistic API mocking
- **Co-locate tests** — `Component.test.tsx` next to `Component.tsx`

---

## Accessibility

- Semantic HTML (`button`, `nav`, `main`, not `div` with onClick)
- `aria-label` for icon-only buttons
- Keyboard navigation (focus management, tab order)
- `role` only when semantic HTML doesn't cover it

---

## Common Libraries

| Purpose            | Library                          |
|--------------------|----------------------------------|
| Routing            | React Router v6 / TanStack Router |
| Server state       | TanStack Query                   |
| Client state       | Zustand / Jotai / Redux Toolkit  |
| Forms              | React Hook Form + Zod            |
| Styling            | Tailwind CSS / CSS Modules       |
| UI Components      | shadcn/ui / Radix / Headless UI  |
| Tables             | TanStack Table                   |
| Animations         | Framer Motion                    |
| Testing            | Vitest + Testing Library + MSW   |
| Date handling      | date-fns / dayjs                 |
| Icons              | Lucide React                     |
| Toasts             | Sonner / react-hot-toast         |
| UUID               | uuid                             |

---

## Output Delivery

1. **apiClient** must send `X-Correlation-Id` header on every request
2. **apiClient** must read `X-Request-Id` from every response
3. **ApiError** class must carry both `correlationId` and `requestId`
4. **ErrorFallback** must display `requestId` prominently — **never show stack traces**
5. **TanStack Query** global `onError` must show toast with `requestId`
6. **ErrorBoundary** wraps all routes to catch render crashes
7. Write all project files under `/home/claude/`
8. Ensure the project builds: `npm run build` (or `npx tsc --noEmit`)
9. Copy final deliverables to `/mnt/user-data/outputs/`

For **artifact-style outputs** (single-file React components rendered in Claude UI), use default exports, Tailwind utility classes only, and import from the allowed library set (React, lucide-react, recharts, shadcn/ui, etc.).
