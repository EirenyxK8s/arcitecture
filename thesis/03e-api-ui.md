# Chapter 3E: Practical Implementation — REST API and Web Dashboard

## 3.10 The Web Dashboard — eirenyx-ui

The `eirenyx-ui` component is a single-page application built with **Angular 21**. It provides a browser-based interface
for managing `Tool` and `Policy` resources and inspecting `PolicyReport` findings. The application follows Angular's
modern standalone component architecture, using the Signals API for reactive state management without the complexity of
external state management libraries.

### 3.10.1 Application Structure

The application is bootstrapped without NgModules, using Angular's standalone API throughout. The root configuration
registers only two providers:

```typescript
export const appConfig: ApplicationConfig = {
    providers: [
        provideRouter(routes),
        provideHttpClient(withInterceptors([apiErrorInterceptor])),
    ]
};
```

The `provideHttpClient` call registers the global HTTP interceptor at bootstrap time. This is Angular's functional
interceptor API, which replaces the older class-based `HttpInterceptor` interface and does not require a separate module
declaration.

Routes are defined in a flat configuration using lazy-loaded standalone components:

```
/home                             → HomeComponent
/tools                            → ToolsList
/tools/new                        → ToolForm
/tools/:namespace/:name           → ToolDetail
/tools/:namespace/:name/edit      → ToolForm
/policies                         → PoliciesList
/policies/new                     → PolicyForm
/policies/:namespace/:name        → PolicyDetail
/reports/:namespace/:policy       → ReportsList
/reports/:namespace/:policy/:name → ReportDetail
```

Each route maps directly to one of the API endpoints. The `namespace` and `name` path parameters are extracted in
components using `inject(ActivatedRoute).snapshot.paramMap`.

### 3.10.2 Service Layer

Three injectable services provide the HTTP communication layer, one per resource type. Each service is decorated with
`@Injectable({providedIn: 'root'})`, making it a singleton registered with the root injector — available throughout the
application without explicit module declarations.

The `ToolService` illustrates the common pattern:

```typescript

@Injectable({providedIn: 'root'})
export class ToolService {
    private readonly base = environment.apiUrl;

    constructor(private readonly http: HttpClient) {
    }

    listTools(namespace?: string): Observable<Tool[]> {
        const params = namespace ? new HttpParams().set('namespace', namespace) : undefined;
        return this.http.get<Tool[]>(`${this.base}/tools`, {params});
    }

    getTool(namespace: string, name: string): Observable<Tool> {
        return this.http.get<Tool>(`${this.base}/tools/${namespace}/${name}`);
    }

    createTool(payload: ToolRequest): Observable<Tool> {
        return this.http.post<Tool>(`${this.base}/tools`, payload);
    }

    updateTool(namespace: string, name: string, payload: ToolRequest): Observable<Tool> {
        return this.http.put<Tool>(`${this.base}/tools/${namespace}/${name}`, payload);
    }

    deleteTool(namespace: string, name: string): Observable<void> {
        return this.http.delete<void>(`${this.base}/tools/${namespace}/${name}`);
    }
}
```

All methods return `Observable<T>` with full generic type parameters. The `HttpClient` generic methods (`get<T>`,
`post<T>`, etc.) perform automatic JSON deserialisation and type the response, eliminating manual `JSON.parse` calls and
providing compile-time type safety at the service boundary.

The `PolicyService` extends this pattern with optional filter parameters:

```typescript
listPolicies(namespace ? : string, type ? : string)
:
Observable < Policy[] > {
    let params = new HttpParams();
    if(namespace) params = params.set('namespace', namespace);
    if(type)      params = params.set('type', type);
    return this.http.get<Policy[]>(`${this.base}/policies`, {params});
}
```

The `ReportService` is intentionally read-only — it exposes only `listReports` and `getReport` methods, mirroring the
read-only nature of the API's `/reports` endpoints.

### 3.10.3 The HTTP Interceptor

A single functional interceptor — `apiErrorInterceptor` — handles all HTTP errors globally, preventing individual
components from implementing duplicated error-handling logic:

```typescript
export const apiErrorInterceptor: HttpInterceptorFn = (req, next) => {
    const notify = inject(NotificationService);
    return next(req).pipe(
        catchError((err: HttpErrorResponse) => {
            const message = err.error?.error ?? err.message;
            switch (err.status) {
                case 404:
                    notify.error(`Not found: ${message}`);
                    break;
                case 409:
                    notify.error(`Conflict: ${message}`);
                    break;
                case 422:
                    notify.error(`Validation: ${message}`);
                    break;
                default:
                    notify.error(`Unexpected error: ${message}`);
            }
            return throwError(() => err);
        })
    );
};
```

The interceptor uses RxJS `catchError` to intercept any HTTP error response in the application's entire request
pipeline. It reads the `error` field from the API's JSON error envelope (`err.error?.error`) and dispatches a
notification to the `NotificationService`. After notifying, it re-throws the original error so that individual
components can still react to specific errors (e.g., redirecting after a 404).

The use of `inject()` inside an `HttpInterceptorFn` is idiomatic in Angular 21 — the function runs in an injection
context, so `inject(NotificationService)` resolves the singleton without constructor injection.

### 3.10.4 Reactive State with Angular Signals

Components manage their local state using Angular's **Signals API**, introduced as stable in Angular 17 and used
throughout the application. Signals are synchronous, reactive values that automatically propagate changes to dependent
computations and templates, without requiring manual change detection triggers or RxJS `Subject` boilerplate.

The `ToolDetail` component demonstrates the signal-based state model:

```typescript

@Component({selector: 'app-tool-detail', standalone: true, ...})
export class ToolDetail implements OnInit {
    tool = signal<Tool | null>(null);
    loading = signal(true);
    deleting = signal(false);

    private readonly api = inject(ToolService);
    private readonly notify = inject(NotificationService);
    private readonly route = inject(ActivatedRoute);
    private readonly router = inject(Router);

    ngOnInit(): void {
        this.api.getTool(this.namespace, this.name).subscribe({
            next: tool => this.tool.set(tool),
            error: err => this.notify.apiError(err, `Failed to load tool.`),
            complete: () => this.loading.set(false),
        });
    }
}
```

Each piece of state is an independent signal. `loading` drives a skeleton/spinner in the template; `deleting` disables
the delete button during an in-flight request; `tool` holds the fetched resource. The template reads these signals
directly — Angular's change detection automatically re-renders any template expression that reads a signal when that
signal's value changes.

The `ToolsList` component shows how signals are updated from observable subscriptions:

```typescript
tools = signal<Tool[]>([]);

private
load()
:
void {
    this.api.listTools().subscribe({
        next: tools => this.tools.set(tools),
        error: err => this.notify.apiError(err, 'Failed to load tools.'),
        complete: () => this.loading.set(false),
    });
}

delete (tool
:
Tool
):
void {
    this.api.deleteTool(tool.namespace, tool.name).subscribe({
        next: () => {
            this.tools.update(all => all.filter(t => t.name !== tool.name));
            this.notify.success(`Tool deleted successfully.`);
        },
    });
}
```

After a successful deletion, `signal.update` applies a transformation function to the current array value, filtering out
the deleted tool. This produces an immediate optimistic UI update without requiring a full list reload.

### 3.10.5 The Notification Service

The `NotificationService` is the application's single source of truth for user-facing messages. It manages a
notifications array as a signal:

```typescript

@Injectable({providedIn: 'root'})
export class NotificationService {
    readonly notifications = signal<Notification[]>([]);

    success(message: string, durationMs = 4000): void {
        this.push('success', message, durationMs);
    }

    error(message: string, durationMs = 6000): void {
        this.push('error', message, durationMs);
    }

    warning(message: string, durationMs = 5000): void {
        this.push('warning', message, durationMs);
    }

    private push(severity: NotificationSeverity, message: string, durationMs: number): void {
        const id = crypto.randomUUID();
        this.notifications.update(n => [...n, {id, severity, message, durationMs}]);
        setTimeout(() => this.dismiss(id), durationMs);
    }

    dismiss(id: string): void {
        this.notifications.update(n => n.filter(x => x.id !== id));
    }
}
```

The `notifications` signal is `readonly` — consumers can only read it, not replace it. Only the service's own methods
mutate it. The `NotificationToast` component, included in the root `App` layout component, reads the signal and renders
a toast stack that automatically reflects additions and dismissals without any additional event plumbing.

Each notification carries a UUID generated by the Web Crypto API (`crypto.randomUUID()`), a severity level, a message
string, and a display duration. The `setTimeout` call inside `push` schedules automatic dismissal, providing a
self-cleaning notification queue without manual management in components.

---

*Previous: [Chapter 3D — Litmus Integration](03d-litmus.md)*
*Next: Chapter 4 — Conclusion and Achieved Results (to be added)*
