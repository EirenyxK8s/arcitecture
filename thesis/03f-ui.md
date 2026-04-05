# Chapter 3F: Practical Implementation — The Web Dashboard

---

## 3.10 The Web Dashboard — eirenyx-ui

### 3.10.1 Purpose and Role in the System

The `eirenyx-ui` component is a single-page application (SPA) that provides a browser-based interface for managing the
Eirenyx platform. It exposes the full lifecycle of `Tool`, `Policy`, and `PolicyReport` resources — creation,
inspection, editing, and deletion — through a graphical interface that does not require the user to interact with
`kubectl` or write Kubernetes manifests directly.

The dashboard serves a different audience from the operator and API:

- The **operator and API** are for automation and scripting: CI/CD pipelines, GitOps systems, and tools that interact
  with the Kubernetes API.
- The **dashboard** is for human operators: security engineers who want to see the current state of all policies at a
  glance, investigate a failed report, or quickly create a new scan policy without writing YAML.

The value proposition of the dashboard is not replacing `kubectl` for power users — it is providing immediate
operational visibility for users who should not need to understand the Kubernetes resource model to use Eirenyx. A
security analyst who wants to know "are there any CRITICAL vulnerabilities in any of our images?" should be able to
answer that question by opening the Eirenyx dashboard, without learning about `VulnerabilityReport` CRDs or writing
label selector queries.

### 3.10.2 Technology Choices

The UI is built with **Angular 21**. Angular was chosen over React or Vue for several reasons:

**Opinionated structure at scale.** Angular provides a complete application framework — routing, HTTP client, dependency
injection, forms, state management — in a single, cohesive package. For an enterprise security dashboard, this
opinionation is an advantage: every Angular application follows the same patterns, making it easier to onboard new team
members and maintain consistency as the codebase grows. React's ecosystem flexibility becomes a liability when different
team members make different architectural choices for routing, state management, and data fetching.

**TypeScript-first.** Angular's entire API is designed around TypeScript. The `HttpClient` generic methods (`get<T>`,
`post<T>`) perform automatic JSON deserialisation and type the response at compile time. This provides end-to-end type
safety from the API's DTO layer to the Angular template.

**Signals API.** Angular 17 introduced the Signals API as a stable, synchronous reactive primitive. Angular 21 uses
Signals throughout. Signals replace the need for external state management libraries (Redux, NgRx, Zustand) for the
Eirenyx dashboard's relatively simple state model, while providing fine-grained reactivity that is more predictable than
RxJS-based approaches for component local state.

**Standalone Components.** Angular 21 uses the standalone component architecture throughout, eliminating NgModules. This
reduces boilerplate and makes it easier to understand each component's dependencies by looking at its `imports` array.

### 3.10.3 Application Structure

The application source is organised by domain and layer:

```
src/
  api-error.interceptor.ts          — Global HTTP error handler
  app/
    app.config.ts                   — Bootstrap configuration
    app.routes.ts                   — Route definitions
    core/
      model/                        — TypeScript interfaces (Tool, Policy, Report)
      service/                      — HTTP service layer
    layout/
      side-nav/                     — Navigation sidebar component
      top-bar/                      — Top navigation bar component
    notification/
      notification.ts               — NotificationService (global state)
      notification-toast/           — Toast display component
    pages/
      home/                         — Dashboard home page
      tools/                        — Tool list, detail, and form pages
      policies/                     — Policy list, detail, and form pages
      reports/                      — Report list and detail pages
  environments/
    environment.ts                  — Development configuration
    environment.prod.ts             — Production configuration
```

The bootstrap configuration is minimal and explicit:

```typescript
export const appConfig: ApplicationConfig = {
    providers: [
        provideRouter(routes),
        provideHttpClient(withInterceptors([apiErrorInterceptor])),
    ]
};
```

The `provideRouter` call registers the route configuration. The `provideHttpClient` call registers Angular's
`HttpClient` with the global error interceptor applied. These are the only two root-level providers — the application
has no NgModules, no `AppModule`, and no global state management library. All other services are declared
`providedIn: 'root'` on their own `@Injectable` decorators, making them self-registering singletons.

### 3.10.4 Route Structure and Navigation

Routes are defined in a flat configuration that maps directly to the Kubernetes resource hierarchy:

```typescript
export const routes: Routes = [
    {path: '', redirectTo: 'home', pathMatch: 'full'},
    {path: 'home', component: Home},

    {path: 'tools', component: ToolsList},
    {path: 'tools/new', component: ToolForm},
    {path: 'tools/:namespace/:name', component: ToolDetail},
    {path: 'tools/:namespace/:name/edit', component: ToolForm},

    {path: 'policies', component: PoliciesList},
    {path: 'policies/new', component: PolicyForm},
    {path: 'policies/:namespace/:name', component: PolicyDetail},
    {path: 'policies/:namespace/:name/edit', component: PolicyForm},

    {path: 'reports/:namespace/:policy', component: ReportsList},
    {path: 'reports/:namespace/:policy/:name', component: ReportDetail},
];
```

The route structure mirrors the Kubernetes resource model: a `Tool` is addressed by `namespace` and `name`; a
`PolicyReport` is addressed by `namespace`, owning `policy`, and its own `name`. This correspondence makes deep-linking
natural — a URL like `/reports/production/api-image-scan/report-20260405` directly encodes the Kubernetes resource
coordinates needed to fetch the report.

`namespace` and `name` path parameters are extracted in components using:

```typescript
private get namespace
(): string
{
    return this.route.snapshot.paramMap.get('namespace')!;
}
```

No additional routing framework or URL parsing utility is needed. Angular's `ActivatedRoute` service provides all
parameter access through a well-typed `ParamMap` interface.

### 3.10.5 The Service Layer — HTTP Communication

Three injectable services provide the HTTP communication layer, one per CRD type. Each service is registered with the
root injector (`providedIn: 'root'`), making it a singleton available throughout the application.

**ToolService** provides full CRUD operations:

```typescript

@Injectable({providedIn: 'root'})
export class ToolService {
    private readonly base = environment.apiUrl;  // '/api' in dev, full URL in prod

    constructor(private readonly http: HttpClient) {
    }

    listTools(namespace?: string): Observable<Tool[]> {
        if (namespace) {
            const params = new HttpParams().set('namespace', namespace);
            return this.http.get<Tool[]>(`${this.base}/tools`, {params});
        }
        return this.http.get<Tool[]>(`${this.base}/tools`);
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

All methods return `Observable<T>`. The `HttpClient` generic methods perform automatic JSON deserialisation:
`http.get<Tool[]>(...)` returns an `Observable<Tool[]>` where `Tool[]` is the TypeScript interface matching the API's
DTO shape. This eliminates manual `JSON.parse` calls and provides compile-time type checking at the service boundary.

**PolicyService** extends this with optional filter parameters for namespace and type filtering:

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

**ReportService** is intentionally read-only — it exposes only `listReports` and `getReport` methods:

```typescript
listReports(namespace
:
string, policy
:
string
):
Observable < PolicyReport[] > {
    return this.http.get<PolicyReport[]>(`${this.base}/reports/${namespace}/${policy}`);
}

getReport(namespace
:
string, policy
:
string, name
:
string
):
Observable < PolicyReport > {
    return this.http.get<PolicyReport>(`${this.base}/reports/${namespace}/${policy}/${name}`);
}
```

This mirrors the read-only restriction at the API level. The TypeScript service has no `createReport`, `updateReport`,
or `deleteReport` methods — the absence of these methods makes it impossible for any UI component to attempt a write
operation on reports, even accidentally.

### 3.10.6 The HTTP Interceptor — Global Error Handling

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

The interceptor reads the `error` field from the API's JSON error envelope (`err.error?.error`) and dispatches a
notification to the `NotificationService`. This is the direct consumer of the consistent error format described in the
API chapter. After notifying, it re-throws the original error so that individual components can still react to specific
error cases — for example, navigating back to the list after a `404` response on a detail page.

The interceptor uses Angular's `inject()` function to resolve the `NotificationService` singleton. This is idiomatic in
Angular 21's functional API: the interceptor function runs inside an injection context, so `inject()` resolves services
without constructor injection.

### 3.10.7 Reactive State with Angular Signals

Components manage their local state using Angular's Signals API. Signals are synchronous, reactive values that
automatically propagate changes to dependent template expressions when their value changes. Angular's change detection
tracks signal reads in templates and schedules a re-render when a signal is updated — no manual
`ChangeDetectorRef.markForCheck()` or `Subject.next()` is required.

**ToolDetail** demonstrates the signal-based state model for a detail page:

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
            error: (err) => this.notify.apiError(err, `Failed to load tool "${this.name}".`),
            complete: () => this.loading.set(false),
        });
    }

    delete(): void {
        this.deleting.set(true);
        this.api.deleteTool(this.namespace, this.name).subscribe({
            next: () => {
                this.notify.success(`Tool "${this.name}" deleted.`);
                this.router.navigate(['/tools']);
            },
            error: (err) => {
                this.notify.apiError(err, `Failed to delete tool.`);
                this.deleting.set(false);
            },
        });
    }
}
```

Three signals manage the component's entire view state:

- `tool` — the loaded resource, `null` while loading
- `loading` — drives a skeleton/spinner in the template
- `deleting` — disables the delete button during an in-flight request

**ToolsList** shows how signals are updated after mutating operations, with an optimistic UI pattern:

```typescript
tools = signal<Tool[]>([]);

delete (tool
:
Tool
):
void {
    this.api.deleteTool(tool.namespace, tool.name).subscribe({
        next: () => {
            this.tools.update(all => all.filter(t =>
                t.name !== tool.name || t.namespace !== tool.namespace
            ));
            this.notify.success(`Tool "${tool.name}" deleted successfully.`);
        },
    });
}
```

After a successful deletion, `signal.update` applies a filter function to the current array value, removing the deleted
tool. The template re-renders immediately — no full list reload from the server is needed. This provides an instant,
responsive feel without an additional HTTP round trip.

### 3.10.8 The Policy Form — Dynamic Multi-Tool Form

The `PolicyForm` component is the most complex in the application because it must handle three structurally different
policy types within a single form. The form uses Angular's `FormsModule` for two-way data binding and manages the
tool-specific sections as independent signal arrays:

```typescript
export class PolicyForm implements OnInit {
    type = signal<PolicyType>('trivy');

    // Trivy-specific state
    trivyScans = signal<TrivyScanRequest[]>([{name: '', image: '', severity: ''}]);

    // Falco-specific state
    falcoRuleRef = signal('');
    falcoReportCreate = signal(true);

    // Litmus-specific state
    litmusExperiments = signal<LitmusExperiment[]>([{
        name: '', experimentRef: '', targetNamespace: '',
        appInfo: {appNamespace: '', appLabel: '', appKind: ''},
        mode: '', duration: '',
    }]);
}
```

The template shows only the section relevant to the currently selected `type`. When the user selects "trivy", the Trivy
scan targets section is visible; when they select "litmus", the chaos experiments section appears. This dynamic form
behaviour is implemented with Angular's `@if` structural directive reading the `type()` signal value.

The `buildPayload()` method constructs the API request from the form signals:

```typescript
private
buildPayload()
:
PolicyRequest
{
    const t = this.type();
    const base: PolicyRequest = {
        name: this.name(), namespace: this.namespace(),
        type: t, enabled: this.enabled(),
    };

    if (t === 'trivy') {
        base.trivy = {scans: this.trivyScans()};
    } else if (t === 'falco') {
        base.falco = {
            observe: {ruleRef: {name: this.falcoRuleRef()}},
            report: {create: this.falcoReportCreate()},
        };
    } else if (t === 'litmus') {
        base.litmus = {experiments: this.litmusExperiments()};
    }
    return base;
}
```

This mirrors the `PolicySpec` discriminated union in the backend: the `type` field determines which sub-spec is present
in the request payload. The form and the API share the same structural contract, enforced by the TypeScript type
definitions in the `core/model/policy.model.ts` file.

### 3.10.9 The Report Detail — Type-Discriminated Display

The `ReportDetail` component must display three structurally different `Details` payloads depending on the policy type.
It uses type guard functions to parse the opaque `details` field:

```typescript
export class ReportDetail implements OnInit {
    report = signal<PolicyReport | null>(null);

    trivyDetails(r: PolicyReport): TrivyReportDetails | null {
        return r.spec.type === 'trivy' ? asTrivyDetails(r.details) : null;
    }

    falcoDetails(r: PolicyReport): FalcoReportDetails | null {
        return r.spec.type === 'falco' ? asFalcoDetails(r.details) : null;
    }

    litmusDetails(r: PolicyReport): LitmusReportDetails | null {
        return r.spec.type === 'litmus' ? asLitmusDetails(r.details) : null;
    }

    verdictClass(verdict: string): string {
        switch (verdict.toLowerCase()) {
            case 'pass':
                return 'verdict verdict--pass';
            case 'fail':
                return 'verdict verdict--fail';
            default:
                return 'verdict verdict--unknown';
        }
    }
}
```

The `asTrivyDetails`, `asFalcoDetails`, and `asLitmusDetails` functions parse the `details` JSON blob into typed
TypeScript interfaces. In the template, the component renders a Trivy vulnerability table if `trivyDetails()` returns a
value, a Falco event list if `falcoDetails()` returns a value, or a Litmus experiment summary if `litmusDetails()`
returns a value. Only one branch is rendered at a time, matching the exclusive nature of the policy type.

The `verdictClass` and `phaseClass` helper methods return CSS class names for styling the verdict badge and phase
indicator. A `Pass` verdict renders in green; `Fail` in red. The phase progresses visually from grey (`Pending`) through
blue (`Running`) to green (`Completed`) or red (`Failed`). This colour coding provides immediate visual feedback on the
health of the security posture without requiring the user to read text labels.

### 3.10.10 The Notification Service — Global Feedback

The `NotificationService` is the application's single source of truth for user-facing messages. It manages a
notifications array as a signal, making it directly readable by the template without `@Output` events or `EventEmitter`
boilerplate:

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

    info(message: string, durationMs = 4000): void {
        this.push('info', message, durationMs);
    }

    apiError(err: HttpErrorResponse, fallback: string, durationMs = 6000): void {
        const body = err?.error;
        const message: string = body?.message ?? body?.error ?? fallback;
        this.error(message, durationMs);
    }

    dismiss(id: string): void {
        this.notifications.update(all => all.filter(n => n.id !== id));
    }

    private push(severity: NotificationSeverity, message: string, durationMs: number): void {
        const notification: Notification = {
            id: crypto.randomUUID(),
            severity, message, durationMs,
        };
        this.notifications.update(all => [...all, notification]);
        setTimeout(() => this.dismiss(notification.id), durationMs);
    }
}
```

The `notifications` signal is declared `readonly` — consumers can read it but cannot replace it. Only the service's own
methods mutate it. The `NotificationToast` component in the root layout reads this signal and renders a toast stack in
the corner of the screen, automatically updating as notifications are added or dismissed.

The `apiError` helper method reads the structured error body from `HttpErrorResponse`, using the API's consistent error
envelope (`body?.message ?? body?.error`) before falling back to a component-provided default. This is the second
consumption point of the consistent error format (the first being the interceptor).

Each notification carries:

- A UUID generated by the Web Crypto API (`crypto.randomUUID()`) — universally unique, no collision risk
- A severity level controlling the visual style
- A message string
- A display duration in milliseconds

The `setTimeout` inside `push` schedules automatic dismissal, producing a self-cleaning notification queue with no
manual management required from components.

### 3.10.11 Development and Production Configuration

The environment configuration controls the API URL:

```typescript
// environment.ts (development)
export const environment = {
    production: false,
    apiUrl: '/api'
};

// environment.prod.ts (production)
export const environment = {
    production: true,
    apiUrl: 'https://api.eirenyx.internal/api'  // or injected at build time
};
```

In development, `apiUrl: '/api'` routes all HTTP calls through the Angular development server's proxy configuration,
which forwards `/api` requests to `http://localhost:8888`. This avoids CORS issues during local development and matches
the production deployment model where an Ingress controller routes `/api` to the `eirenyx-api` service and `/` to the
`eirenyx-ui` service.

In production, the Angular application is built as a static bundle (`ng build --configuration production`) and served by
an nginx container or any static file server. The `apiUrl` in `environment.prod.ts` points to the API's cluster-internal
URL, which is accessible from within the cluster network where the UI is deployed.

---

*Previous: [Chapter 3E — REST API](03e-api.md)*
*Next: Chapter 4 — Conclusion and Achieved Results (to be added)*
