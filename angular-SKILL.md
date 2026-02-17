---
name: angular
description: Use this skill whenever the user wants to create, build, debug, or work with Angular applications. Triggers include any mention of 'Angular', 'ng', '@angular', 'Angular CLI', 'standalone components', 'signals', 'signal forms', 'zoneless', 'Angular Material', 'Angular CDK', 'Angular SSR', 'Angular routing', or requests involving component creation, service injection, reactive forms, template syntax, change detection, lazy loading, interceptors, guards, resolvers, or Angular-specific testing with Vitest or Jasmine. Also use when working with Angular schematics, Angular library creation, Angular workspace configuration (angular.json), or migrating between Angular versions. Do NOT use for AngularJS (v1.x), React, Vue, Svelte, or other non-Angular frontend frameworks.
---

# Angular Skill

This skill guides the creation of professional, performant, and maintainable Angular applications following the latest Angular 21 patterns and best practices. It covers standalone components, signals-based reactivity, zoneless change detection, modern testing with Vitest, and the full Angular CLI toolchain.

The user provides application requirements: component creation, routing, forms, services, HTTP communication, state management, SSR, testing, or full project scaffolding. They may include context about their Angular version, backend API, or design system.

---

## Target Version & Requirements

- **Angular 21** (released November 2025) — the latest stable version
- **Node.js v20+** (LTS) required
- **TypeScript 5.8+** required
- **Package Manager:** npm (default) or yarn/pnpm
- **Build System:** esbuild + Vite (default since Angular 17+)
- **Test Runner:** Vitest (default for new projects in v21), Jasmine/Karma still supported

### Key Angular 21 Features

| Feature | Status | Notes |
|---------|--------|-------|
| Standalone Components | **Stable** (default) | No NgModules needed for new projects |
| Signals (`signal`, `computed`, `effect`) | **Stable** | Core reactivity primitive |
| `linkedSignal`, signal-based inputs/queries | **Stable** | Full signal ecosystem |
| Signal Forms (`@angular/forms/signals`) | **Experimental** | New reactive forms API built on Signals |
| Zoneless Change Detection | **Stable** (default for new apps) | `zone.js` no longer included by default |
| Vitest | **Stable** (default test runner) | Replaces Karma/Jasmine for new projects |
| Angular Aria (`@angular/aria`) | **Developer Preview** | Headless accessible UI components |
| Angular MCP Server | **Stable** | AI-assisted development via CLI |
| Route-level Render Mode | **Stable** | Granular SSR/SSG/CSR per route |
| Incremental Hydration | **Stable** | Powered by `@defer` blocks |
| Tailwind CSS | **Built-in schematic** | `ng add tailwindcss` or configured at `ng new` |

---

## Environment Setup

```bash
# Check Node.js version (v20+ required)
node --version

# Install Angular CLI globally
npm install -g @angular/cli@latest

# Verify CLI version
ng version

# Create a new Angular 21 project (zoneless by default, Vitest by default)
ng new my-app --style=scss --routing

# Serve the application
cd my-app
ng serve

# Build for production
ng build
```

### Project Scaffolding Options

```bash
# Full options for new project
ng new my-app \
  --style=scss \
  --routing \
  --ssr \
  --prefix=app \
  --strict

# Create without SSR
ng new my-app --style=scss --routing --ssr=false

# Add Tailwind CSS to existing project
ng add tailwindcss
```

---

## Project Structure — Feature-Based Organization

```
my-app/
├── src/
│   ├── app/
│   │   ├── core/                       # Singleton services, guards, interceptors
│   │   │   ├── auth/
│   │   │   │   ├── auth.guard.ts
│   │   │   │   ├── auth.interceptor.ts
│   │   │   │   └── auth.service.ts
│   │   │   ├── http/
│   │   │   │   ├── api.interceptor.ts
│   │   │   │   └── error.interceptor.ts
│   │   │   └── services/
│   │   │       └── notification.service.ts
│   │   ├── shared/                     # Reusable components, directives, pipes
│   │   │   ├── components/
│   │   │   │   ├── button/
│   │   │   │   │   ├── button.component.ts
│   │   │   │   │   ├── button.component.html
│   │   │   │   │   ├── button.component.scss
│   │   │   │   │   └── button.component.spec.ts
│   │   │   │   └── loading-spinner/
│   │   │   ├── directives/
│   │   │   │   └── highlight.directive.ts
│   │   │   ├── pipes/
│   │   │   │   └── truncate.pipe.ts
│   │   │   └── models/
│   │   │       └── api-response.model.ts
│   │   ├── features/                   # Feature-based folders
│   │   │   ├── dashboard/
│   │   │   │   ├── dashboard.component.ts
│   │   │   │   ├── dashboard.component.html
│   │   │   │   ├── dashboard.component.scss
│   │   │   │   ├── dashboard.component.spec.ts
│   │   │   │   └── dashboard.routes.ts
│   │   │   ├── users/
│   │   │   │   ├── user-list/
│   │   │   │   │   ├── user-list.component.ts
│   │   │   │   │   └── user-list.component.html
│   │   │   │   ├── user-detail/
│   │   │   │   ├── services/
│   │   │   │   │   └── user.service.ts
│   │   │   │   ├── models/
│   │   │   │   │   └── user.model.ts
│   │   │   │   └── users.routes.ts
│   │   │   └── products/
│   │   │       ├── product-list/
│   │   │       ├── product-detail/
│   │   │       ├── services/
│   │   │       ├── models/
│   │   │       └── products.routes.ts
│   │   ├── app.component.ts
│   │   ├── app.component.html
│   │   ├── app.component.scss
│   │   ├── app.config.ts               # Application configuration (providers)
│   │   └── app.routes.ts               # Top-level routes
│   ├── environments/
│   │   ├── environment.ts
│   │   └── environment.prod.ts
│   ├── styles/
│   │   ├── _variables.scss
│   │   ├── _mixins.scss
│   │   └── _reset.scss
│   ├── index.html
│   ├── main.ts                         # Bootstrap entry point
│   └── styles.scss
├── angular.json
├── package.json
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.spec.json
└── .postcssrc.json                     # Tailwind CSS config (if using Tailwind)
```

---

## Standalone Components — The Default Pattern

Angular 21 uses standalone components by default. No NgModules are needed.

### Component Template

```typescript
// features/users/user-list/user-list.component.ts
import { Component, inject, OnInit, signal, computed } from '@angular/core';
import { RouterLink } from '@angular/router';
import { UserService } from '../services/user.service';
import { User } from '../models/user.model';
import { LoadingSpinnerComponent } from '../../../shared/components/loading-spinner/loading-spinner.component';
import { TruncatePipe } from '../../../shared/pipes/truncate.pipe';

@Component({
  selector: 'app-user-list',
  imports: [RouterLink, LoadingSpinnerComponent, TruncatePipe],
  templateUrl: './user-list.component.html',
  styleUrl: './user-list.component.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserListComponent implements OnInit {
  private readonly userService = inject(UserService);

  // Signals for reactive state
  readonly users = signal<User[]>([]);
  readonly isLoading = signal(true);
  readonly searchTerm = signal('');
  readonly errorMessage = signal<string | null>(null);

  // Computed signal for filtered users
  readonly filteredUsers = computed(() => {
    const term = this.searchTerm().toLowerCase();
    if (!term) return this.users();
    return this.users().filter(user =>
      user.name.toLowerCase().includes(term) ||
      user.email.toLowerCase().includes(term)
    );
  });

  readonly userCount = computed(() => this.filteredUsers().length);

  ngOnInit(): void {
    this.loadUsers();
  }

  async loadUsers(): Promise<void> {
    this.isLoading.set(true);
    this.errorMessage.set(null);
    try {
      const users = await this.userService.getUsers();
      this.users.set(users);
    } catch (error) {
      this.errorMessage.set('Failed to load users. Please try again.');
    } finally {
      this.isLoading.set(false);
    }
  }

  updateSearch(term: string): void {
    this.searchTerm.set(term);
  }
}
```

### Template with Built-in Control Flow

```html
<!-- user-list.component.html -->
<div class="user-list-container">
  <h1>Users ({{ userCount() }})</h1>

  <input
    type="search"
    placeholder="Search users..."
    [value]="searchTerm()"
    (input)="updateSearch($any($event.target).value)"
  />

  @if (isLoading()) {
    <app-loading-spinner />
  } @else if (errorMessage()) {
    <div class="error-banner">
      <p>{{ errorMessage() }}</p>
      <button (click)="loadUsers()">Retry</button>
    </div>
  } @else {
    @if (filteredUsers().length === 0) {
      <p class="empty-state">No users found.</p>
    } @else {
      <ul class="user-grid">
        @for (user of filteredUsers(); track user.id) {
          <li class="user-card">
            <a [routerLink]="['/users', user.id]">
              <h3>{{ user.name }}</h3>
              <p>{{ user.email | truncate: 30 }}</p>
            </a>
          </li>
        }
      </ul>
    }
  }
</div>
```

### Component Design Rules

- **Always use `ChangeDetectionStrategy.OnPush`** — mandatory for performance, especially with zoneless
- **Always use standalone components** — `standalone: true` is the default in Angular 21; never set `standalone: false` in new code
- **Use `inject()` function** over constructor injection — cleaner, works with inheritance
- **Use Signals for component state** — `signal()`, `computed()`, `effect()` instead of BehaviorSubject/plain properties
- **Use built-in control flow** — `@if`, `@else`, `@for`, `@switch`, `@defer` instead of `*ngIf`, `*ngFor`, `*ngSwitch`
- **Always use `track` in `@for` loops** — required for performance
- **Prefix selectors** with app name — `app-user-list`, not `user-list`
- **One component per file** — no exceptions
- **Co-locate related files** — component, template, styles, and tests together

---

## Signals — Reactivity Model

Angular 21 stabilizes the full Signals API as the core reactivity primitive.

```typescript
import { signal, computed, effect, linkedSignal, untracked } from '@angular/core';

// Writable signal
const count = signal(0);
count.set(5);
count.update(current => current + 1);

// Computed signal (read-only, auto-tracks dependencies)
const doubleCount = computed(() => count() * 2);

// Effect (side effect that runs when dependencies change)
effect(() => {
  console.log(`Count changed to: ${count()}`);
});

// LinkedSignal (signal linked to a source with transformation)
const source = signal('hello');
const linked = linkedSignal({
  source: source,
  computation: (sourceValue) => sourceValue.toUpperCase(),
});
```

### Signal-Based Inputs, Outputs, and Queries

```typescript
import {
  Component, input, output, model,
  viewChild, viewChildren, contentChild, contentChildren
} from '@angular/core';

@Component({
  selector: 'app-user-card',
  template: `
    <div class="card" #cardElement>
      <h3>{{ name() }}</h3>
      <p>{{ email() }}</p>
      @if (isEditable()) {
        <button (click)="onEdit.emit(userId())">Edit</button>
      }
    </div>
  `,
})
export class UserCardComponent {
  // Signal-based inputs (stable in v21)
  readonly name = input.required<string>();
  readonly email = input.required<string>();
  readonly userId = input.required<number>();
  readonly isEditable = input(false);    // Optional with default

  // Signal-based output
  readonly onEdit = output<number>();

  // Two-way binding with model()
  readonly selectedState = model(false);

  // Signal-based queries (stable in v21)
  readonly cardElement = viewChild<ElementRef>('cardElement');
  readonly childComponents = viewChildren(SomeChildComponent);
  readonly projectedContent = contentChild(SomeDirective);
}
```

---

## Services & Dependency Injection

```typescript
// core/services/user.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';
import { User, CreateUserDto, UpdateUserDto } from '../../features/users/models/user.model';
import { environment } from '../../../environments/environment';

@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/users`;

  async getUsers(): Promise<User[]> {
    return firstValueFrom(
      this.http.get<User[]>(this.baseUrl)
    );
  }

  async getUserById(id: number): Promise<User> {
    return firstValueFrom(
      this.http.get<User>(`${this.baseUrl}/${id}`)
    );
  }

  async createUser(dto: CreateUserDto): Promise<User> {
    return firstValueFrom(
      this.http.post<User>(this.baseUrl, dto)
    );
  }

  async updateUser(id: number, dto: UpdateUserDto): Promise<User> {
    return firstValueFrom(
      this.http.put<User>(`${this.baseUrl}/${id}`, dto)
    );
  }

  async deleteUser(id: number): Promise<void> {
    return firstValueFrom(
      this.http.delete<void>(`${this.baseUrl}/${id}`)
    );
  }
}
```

### Service Design Rules

- **Use `providedIn: 'root'`** for singleton services (tree-shakable)
- **Use `inject()` function** instead of constructor injection
- **Use `firstValueFrom()`** to convert Observable to Promise for simple HTTP calls
- **Keep RxJS Observables** for streams that emit multiple values (WebSockets, real-time, complex async pipelines)
- **Use environment files** for API URLs — never hardcode
- **Create typed DTOs** — separate `CreateDto`, `UpdateDto`, and response models

---

## Routing — Lazy-Loaded Feature Routes

### App Routes (Top Level)

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { authGuard } from './core/auth/auth.guard';

export const routes: Routes = [
  {
    path: '',
    redirectTo: 'dashboard',
    pathMatch: 'full',
  },
  {
    path: 'dashboard',
    loadComponent: () =>
      import('./features/dashboard/dashboard.component').then(m => m.DashboardComponent),
    canActivate: [authGuard],
  },
  {
    path: 'users',
    loadChildren: () =>
      import('./features/users/users.routes').then(m => m.USERS_ROUTES),
    canActivate: [authGuard],
  },
  {
    path: 'products',
    loadChildren: () =>
      import('./features/products/products.routes').then(m => m.PRODUCTS_ROUTES),
    canActivate: [authGuard],
  },
  {
    path: 'login',
    loadComponent: () =>
      import('./features/auth/login/login.component').then(m => m.LoginComponent),
  },
  {
    path: '**',
    loadComponent: () =>
      import('./shared/components/not-found/not-found.component').then(m => m.NotFoundComponent),
  },
];
```

### Feature Routes

```typescript
// features/users/users.routes.ts
import { Routes } from '@angular/router';

export const USERS_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () =>
      import('./user-list/user-list.component').then(m => m.UserListComponent),
  },
  {
    path: ':id',
    loadComponent: () =>
      import('./user-detail/user-detail.component').then(m => m.UserDetailComponent),
  },
  {
    path: ':id/edit',
    loadComponent: () =>
      import('./user-edit/user-edit.component').then(m => m.UserEditComponent),
  },
];
```

### Routing Rules

- **Always lazy-load feature routes** with `loadChildren` or `loadComponent`
- **Use `loadComponent`** for individual routes, `loadChildren` for feature route groups
- **Use functional guards** — `canActivate: [authGuard]` with `CanActivateFn`
- **Use `@defer` blocks** for below-the-fold content within components
- **Export route arrays as constants** — `export const FEATURE_ROUTES: Routes`

---

## Application Configuration — Providers

```typescript
// app.config.ts
import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
import { provideRouter, withComponentInputBinding, withViewTransitions } from '@angular/router';
import { provideHttpClient, withInterceptors, withFetch } from '@angular/common/http';
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';
import { routes } from './app.routes';
import { apiInterceptor } from './core/http/api.interceptor';
import { errorInterceptor } from './core/http/error.interceptor';
import { authInterceptor } from './core/auth/auth.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    // Angular 21: zoneless is default for new apps, but explicit config shown here
    provideZoneChangeDetection({ eventCoalescing: true }),

    // Router with component input binding (route params as inputs)
    provideRouter(
      routes,
      withComponentInputBinding(),
      withViewTransitions(),
    ),

    // HTTP client with functional interceptors and fetch API
    provideHttpClient(
      withFetch(),
      withInterceptors([authInterceptor, apiInterceptor, errorInterceptor]),
    ),

    // Async animations (lazy-loaded)
    provideAnimationsAsync(),
  ],
};
```

### Bootstrap Entry Point

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig)
  .catch(err => console.error(err));
```

---

## HTTP Interceptors — Functional Pattern

```typescript
// core/http/auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from '../auth/auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();

  if (token) {
    const cloned = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`,
      },
    });
    return next(cloned);
  }

  return next(req);
};
```

```typescript
// core/http/error.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { catchError, throwError } from 'rxjs';
import { NotificationService } from '../services/notification.service';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  const notifications = inject(NotificationService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      switch (error.status) {
        case 401:
          router.navigate(['/login']);
          break;
        case 403:
          notifications.error('You do not have permission to perform this action.');
          break;
        case 404:
          notifications.error('The requested resource was not found.');
          break;
        case 500:
          notifications.error('An unexpected server error occurred. Please try again.');
          break;
        default:
          notifications.error('A network error occurred.');
      }
      return throwError(() => error);
    })
  );
};
```

---

## Guards — Functional Pattern

```typescript
// core/auth/auth.guard.ts
import { CanActivateFn } from '@angular/router';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url },
  });
};
```

---

## Forms

### Signal Forms (Experimental — Angular 21)

```typescript
// Angular 21 Signal Forms — new reactive forms API
import { Component, signal } from '@angular/core';
import { form, SignalFormDirective } from '@angular/forms/signals';

interface UserForm {
  name: string;
  email: string;
  age: number | null;
}

@Component({
  selector: 'app-user-form',
  imports: [SignalFormDirective],
  template: `
    <form [signalForm]="userForm">
      <label>
        Name:
        <input type="text" [formField]="userForm.controls.name" />
      </label>

      @if (userForm.controls.name.touched() && userForm.controls.name.invalid()) {
        <span class="error">Name is required</span>
      }

      <label>
        Email:
        <input type="email" [formField]="userForm.controls.email" />
      </label>

      <button type="submit" [disabled]="userForm.invalid()">Save</button>
    </form>
  `,
})
export class UserFormComponent {
  readonly userData = signal<UserForm>({
    name: '',
    email: '',
    age: null,
  });

  readonly userForm = form(this.userData);

  onSubmit(): void {
    if (this.userForm.valid()) {
      console.log('Form value:', this.userData());
    }
  }
}
```

### Reactive Forms (Stable — Classic Pattern)

```typescript
import { Component, inject } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

@Component({
  selector: 'app-user-form',
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <label>
        Name:
        <input formControlName="name" />
      </label>

      @if (form.controls.name.touched && form.controls.name.invalid) {
        <span class="error">Name is required</span>
      }

      <label>
        Email:
        <input formControlName="email" type="email" />
      </label>

      @if (form.controls.email.touched && form.controls.email.errors?.['email']) {
        <span class="error">Invalid email format</span>
      }

      <button type="submit" [disabled]="form.invalid">Save</button>
    </form>
  `,
})
export class UserFormComponent {
  private readonly fb = inject(FormBuilder);

  readonly form = this.fb.nonNullable.group({
    name: ['', [Validators.required, Validators.minLength(2)]],
    email: ['', [Validators.required, Validators.email]],
    age: [null as number | null, [Validators.min(0), Validators.max(150)]],
  });

  onSubmit(): void {
    if (this.form.valid) {
      const value = this.form.getRawValue();
      console.log('Form value:', value);
    }
  }
}
```

### Forms Rules

- **New projects:** Consider Signal Forms for new form logic — they are the future direction
- **Existing projects:** Continue using Reactive Forms — they remain fully supported
- **Always use typed forms** — `fb.nonNullable.group()` for type-safe form controls
- **Never use template-driven forms** in production applications
- **Validate on both client and server** — client validation is for UX only

---

## Testing — Vitest (Default in Angular 21)

```typescript
// user-list.component.spec.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';
import { UserListComponent } from './user-list.component';
import { UserService } from '../services/user.service';

describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  let mockUserService: { getUsers: ReturnType<typeof vi.fn> };

  beforeEach(async () => {
    mockUserService = {
      getUsers: vi.fn().mockResolvedValue([
        { id: 1, name: 'Alice', email: 'alice@example.com' },
        { id: 2, name: 'Bob', email: 'bob@example.com' },
      ]),
    };

    await TestBed.configureTestingModule({
      imports: [UserListComponent],
      providers: [
        { provide: UserService, useValue: mockUserService },
        provideRouter([]),
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should load users on init', async () => {
    fixture.detectChanges();
    await fixture.whenStable();

    expect(mockUserService.getUsers).toHaveBeenCalledOnce();
    expect(component.users().length).toBe(2);
    expect(component.isLoading()).toBe(false);
  });

  it('should filter users by search term', async () => {
    fixture.detectChanges();
    await fixture.whenStable();

    component.updateSearch('alice');
    expect(component.filteredUsers().length).toBe(1);
    expect(component.filteredUsers()[0].name).toBe('Alice');
  });

  it('should handle error when loading users', async () => {
    mockUserService.getUsers.mockRejectedValue(new Error('Network error'));

    fixture.detectChanges();
    await fixture.whenStable();

    expect(component.errorMessage()).toBe('Failed to load users. Please try again.');
    expect(component.isLoading()).toBe(false);
  });
});
```

### Service Test

```typescript
// user.service.spec.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';
import { UserService } from './user.service';

describe('UserService', () => {
  let service: UserService;
  let httpTesting: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    });

    service = TestBed.inject(UserService);
    httpTesting = TestBed.inject(HttpTestingController);
  });

  it('should fetch users', async () => {
    const mockUsers = [{ id: 1, name: 'Alice', email: 'alice@test.com' }];
    const usersPromise = service.getUsers();

    const req = httpTesting.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers);

    const users = await usersPromise;
    expect(users).toEqual(mockUsers);
  });
});
```

### Testing Rules

- **Use Vitest** for new projects — `vi.fn()` for mocks, `vi.spyOn()` for spies
- **Use `TestBed.configureTestingModule`** — same pattern as before, fully compatible
- **Test signals directly** — `component.users()` reads the signal value
- **Use `provideHttpClientTesting()`** — the modern way to mock HTTP
- **Run tests:** `ng test` (Vitest by default in v21)
- **Browser mode available:** Vitest can run in real browsers instead of jsdom

---

## Deferred Loading — `@defer` Blocks

```html
<!-- Lazy load heavy components below the fold -->
@defer (on viewport) {
  <app-heavy-chart [data]="chartData()" />
} @placeholder {
  <div class="chart-placeholder">Chart loading area</div>
} @loading (minimum 300ms) {
  <app-loading-spinner />
} @error {
  <p>Failed to load chart component.</p>
}

<!-- Defer on interaction -->
@defer (on interaction) {
  <app-comments [postId]="postId()" />
} @placeholder {
  <button>Load Comments</button>
}

<!-- Defer on idle -->
@defer (on idle) {
  <app-analytics-widget />
}
```

---

## Models & Interfaces

```typescript
// shared/models/api-response.model.ts
export interface ApiResponse<T> {
  data: T;
  message: string;
  success: boolean;
}

export interface PaginatedResponse<T> {
  data: T[];
  totalRecords: number;
  pageNumber: number;
  pageSize: number;
  totalPages: number;
}

// features/users/models/user.model.ts
export interface User {
  id: number;
  name: string;
  email: string;
  phone?: string;
  isActive: boolean;
  createdAt: string;
  modifiedAt?: string;
}

export interface CreateUserDto {
  name: string;
  email: string;
  phone?: string;
}

export interface UpdateUserDto {
  name?: string;
  email?: string;
  phone?: string;
}
```

### Model Rules

- **Use `interface` for data shapes** — not `class` (interfaces are erased at compile time)
- **Use `class` only when behavior is needed** (methods, constructors)
- **Separate DTOs** — `CreateDto`, `UpdateDto`, response model
- **Use strict types** — avoid `any`; use `unknown` when the type is truly unknown
- **Optional properties** — use `?` for nullable/optional fields
- **Date strings** — use `string` for ISO date strings from API, convert to `Date` only when needed

---

## Environment Configuration

```typescript
// environments/environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api',
  appTitle: 'My App (Dev)',
};

// environments/environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.myapp.com/api',
  appTitle: 'My App',
};
```

---

## Angular CLI — Generator Commands

```bash
# Components
ng generate component features/users/user-list          # Standalone by default
ng generate component shared/components/button           # Shared component
ng generate component features/users/user-list --inline-style --inline-template  # Inline

# Services
ng generate service core/services/notification
ng generate service features/users/services/user

# Guards (functional by default)
ng generate guard core/auth/auth --functional

# Interceptors (functional by default)
ng generate interceptor core/http/api

# Pipes
ng generate pipe shared/pipes/truncate

# Directives
ng generate directive shared/directives/highlight

# Libraries (for monorepo)
ng generate library my-shared-lib

# Environments
ng generate environments
```

---

## Performance Best Practices

| Practice | Implementation |
|----------|---------------|
| **OnPush change detection** | Set on every component: `changeDetection: ChangeDetectionStrategy.OnPush` |
| **Zoneless (default in v21)** | New apps are zoneless by default; migrate existing apps with provided schematics |
| **Lazy loading** | All feature routes use `loadChildren` / `loadComponent` |
| **Defer blocks** | Use `@defer (on viewport)` for below-fold content |
| **Track expressions** | Always provide `track` in `@for` loops |
| **Signal-based state** | Use `signal()` / `computed()` instead of mutable properties |
| **Image optimization** | Use `NgOptimizedImage` directive for all `<img>` tags |
| **Bundle analysis** | Run `ng build --stats-json` and use `webpack-bundle-analyzer` or source-map-explorer |
| **Preloading strategy** | Configure `withPreloading(PreloadAllModules)` in router |
| **Tree-shaking** | Use `providedIn: 'root'` for services |

---

## SSR & Hydration (Angular 21)

```typescript
// app.config.server.ts
import { mergeApplicationConfig, ApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/platform-server';
import { provideServerRouting, RenderMode } from '@angular/ssr';
import { appConfig } from './app.config';
import { serverRoutes } from './app.routes.server';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(),
    provideServerRouting(serverRoutes),
  ],
};

export const config = mergeApplicationConfig(appConfig, serverConfig);

// app.routes.server.ts
import { RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  { path: '', renderMode: RenderMode.Prerender },                // Static prerender
  { path: 'dashboard', renderMode: RenderMode.Server },          // SSR
  { path: 'users/:id', renderMode: RenderMode.Server },          // SSR with dynamic params
  { path: 'login', renderMode: RenderMode.Client },              // Client-only
  { path: '**', renderMode: RenderMode.Server },                 // Default SSR
];
```

---

## Key Design Decisions Summary

| Feature | Implementation |
|---------|---------------|
| **Components** | Standalone by default, `OnPush` change detection, `inject()` for DI |
| **Reactivity** | Signals (`signal`, `computed`, `effect`, `linkedSignal`) — stable in v21 |
| **Forms** | Signal Forms (experimental) for new code, Reactive Forms (stable) for existing |
| **Routing** | Lazy-loaded features, functional guards, component input binding |
| **HTTP** | Functional interceptors, `withFetch()`, typed DTOs |
| **Testing** | Vitest (default), `vi.fn()` mocks, `TestBed` compatible |
| **Change detection** | Zoneless by default in v21; `OnPush` on all components |
| **Template syntax** | Built-in control flow (`@if`, `@for`, `@switch`, `@defer`) |
| **Styling** | SCSS by default, Tailwind CSS via built-in schematic |
| **SSR** | Route-level render mode (Prerender / Server / Client) |
| **Accessibility** | Angular Aria (`@angular/aria`) for headless accessible components |
| **Project structure** | Feature-based folders: `core/`, `shared/`, `features/` |

---

## Output Delivery

1. Write all files under `/home/claude/`
2. **All components must be standalone** — no NgModules unless explicitly required
3. **All components must use `OnPush` change detection** 
4. **Use Signals for state** — `signal()`, `computed()`, `effect()`
5. **Use built-in control flow** — `@if`, `@for`, `@switch`, `@defer`
6. **Use `inject()` function** for dependency injection
7. **Use functional interceptors and guards** — not class-based
8. **Lazy-load all feature routes** with `loadComponent` / `loadChildren`
9. **Include TypeScript strict mode** in all `tsconfig.json` files
10. Copy final files to `/mnt/user-data/outputs/`
11. For multi-file deliverables, organize by feature and include a `README.md`
12. Run lint/build checks where possible before delivery
