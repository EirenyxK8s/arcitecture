# Chapter 3E: Practical Implementation — REST API and Web Dashboard

---

## 3.9 The REST API — eirenyx-api

The `eirenyx-api` component is a stateless Go HTTP server that bridges the Angular web dashboard and the Kubernetes API.
It exposes a JSON REST interface over the Eirenyx CRDs without introducing any persistent state of its own. All data
lives in Kubernetes objects; the API server is purely a translation layer.

The server is built with the **chi** router (`github.com/go-chi/chi/v5`), a lightweight, idiomatic Go HTTP framework
that is composable, middleware-friendly, and produces clean URL parameter extraction. It runs on port 8888, with health
and readiness probes served separately on port 8081.

### 3.9.1 MVC Structure

The internal package layout follows a strict MVC organisation:

```
internal/
  mvc/
    router.go           — wires controllers, middleware, and routes
    controller/
      tool.go           — HTTP handlers for Tool resources
      policy.go         — HTTP handlers for Policy resources
      report.go         — HTTP handlers for PolicyReport resources (read-only)
      health.go         — liveness and readiness probes
      response.go       — shared JSON response helpers and error mapping
    service/
      tool.go           — Kubernetes API interactions for Tools
      policy.go         — Kubernetes API interactions for Policies
      report.go         — Kubernetes API interactions for PolicyReports
      health.go         — cluster connectivity health check
    dto/
      tool.go           — Tool request/response structs and mapping functions
      policy.go         — Policy response structs
      policy_request.go — Policy create/update request structs
      report.go         — PolicyReport response structs
      base_resource.go  — Shared BaseResource struct
  api_errors/
    error.go            — Typed error sentinel types
```

Each layer has a single responsibility. Controllers decode HTTP requests and encode HTTP responses. Services interact
with Kubernetes through the controller-runtime `client.Client`. DTOs define the JSON contract and contain the mapping
logic between Kubernetes types and API response shapes.

### 3.9.2 Router and Middleware

The `NewRouter` function constructs the chi router, registers global middleware, instantiates the four controllers, and
mounts the routes:

```go
func NewRouter(k8s client.Client) http.Handler {
    r := chi.NewRouter()
    r.Use(middleware.RequestID)
    r.Use(middleware.RealIP)
    r.Use(middleware.Logger)
    r.Use(middleware.Recoverer)
    r.Use(cors.Handler(cors.Options{
        AllowedOrigins: []string{"*"},
        AllowedMethods: []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
        AllowedHeaders: []string{"Accept", "Authorization", "Content-Type"},
    }))
    // ... route registration ...
}
```

Four middleware components are applied globally:

- `RequestID` — attaches a unique request identifier to every request, propagated through the context for log
  correlation.
- `RealIP` — extracts the actual client IP from `X-Forwarded-For` or `X-Real-IP` headers, relevant when the API is
  fronted by an Ingress controller.
- `Logger` — structured access logging with method, path, status code, and latency.
- `Recoverer` — catches panics in any handler and returns a `500 Internal Server Error` rather than crashing the
  process.

CORS is configured with a permissive origin policy (`*`) because authentication is delegated entirely to Kubernetes
RBAC. The API server does not perform its own authentication or authorisation; it relies on the ambient service account
identity of the pod it runs in.

The route structure mirrors the CRD hierarchy:

| Method   | Path                                         | Handler                                 |
|----------|----------------------------------------------|-----------------------------------------|
| `GET`    | `/api/tools`                                 | List all tools (optional `?namespace=`) |
| `POST`   | `/api/tools`                                 | Create a tool                           |
| `GET`    | `/api/tools/{namespace}/{name}`              | Get a single tool                       |
| `PUT`    | `/api/tools/{namespace}/{name}`              | Replace tool spec                       |
| `DELETE` | `/api/tools/{namespace}/{name}`              | Delete a tool                           |
| `GET`    | `/api/tools/{namespace}/{toolName}/policies` | List policies owned by a tool           |
| `GET`    | `/api/policies`                              | List all policies                       |
| `POST`   | `/api/policies`                              | Create a policy                         |
| `GET`    | `/api/policies/{namespace}/{name}`           | Get a single policy                     |
| `PUT`    | `/api/policies/{namespace}/{name}`           | Replace policy spec                     |
| `DELETE` | `/api/policies/{namespace}/{name}`           | Delete a policy                         |
| `GET`    | `/api/reports/{namespace}/{policy}`          | List reports for a policy               |
| `GET`    | `/api/reports/{namespace}/{policy}/{name}`   | Get a single report                     |

The `PolicyReport` resource is intentionally **read-only** at the API level — there are no `POST`, `PUT`, or `DELETE`
endpoints for reports. Reports are created and managed exclusively by the operator.

### 3.9.3 Controller Layer

Controllers are thin. Each method follows the same four-step pattern: extract path/query parameters, decode the request
body (for mutating operations), call the service, write the response. The `ToolsController.Create` method illustrates
this clearly:

```go
func (c *ToolsController) Create(w http.ResponseWriter, r *http.Request) {
    var req dto.ToolRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondJSON(w, http.StatusBadRequest, errorBody("invalid request body: "+err.Error()))
        return
    }
    if err := validateToolRequest(req); err != nil {
        respondJSON(w, http.StatusUnprocessableEntity, errorBody(err.Error()))
        return
    }
    tool, err := c.service.Create(r.Context(), req)
    if err != nil {
        respondError(w, err)
        return
    }
    respondJSON(w, http.StatusCreated, tool)
}
```

Input validation (`validateToolRequest`) runs in the controller, not in the service. It checks structural requirements —
non-empty `name` and `namespace`, and a `spec.type` value belonging to the set `{trivy, falco, litmus}` — and returns a
`422 Unprocessable Entity` immediately, before any Kubernetes API call is made. This separates syntactic validation from
semantic errors that require cluster state.

### 3.9.4 Service Layer

Services hold all Kubernetes API interaction logic. They receive plain Go structs from the controllers and return plain
Go structs or typed errors. The `ToolService.Get` method shows the canonical pattern:

```go
func (s *ToolService) Get(ctx context.Context, namespace, name string) (*dto.Tool, error) {
    tool := &eirenyxv1alpha1.Tool{}
    if err := s.k8s.Get(ctx, types.NamespacedName{Namespace: namespace, Name: name}, tool); err != nil {
        if k8serrors.IsNotFound(err) {
            return nil, api_errors.NewErrNotFound(fmt.Sprintf("tool %s/%s not found", namespace, name))
        }
        return nil, fmt.Errorf("getting tool %s/%s: %w", namespace, name, err)
    }
    result := dto.MapToTool(*tool)
    return &result, nil
}
```

Kubernetes `IsNotFound` errors are explicitly translated into `api_errors.ErrNotFound`. This type-based translation is
the bridge between the Kubernetes error model and the HTTP error model. The service never touches `http.ResponseWriter`;
it only returns typed errors that the controller's `respondError` function knows how to map to HTTP status codes.

The `Create` service method includes a pre-flight existence check to produce a semantic `409 Conflict` response, rather
than exposing a raw Kubernetes API conflict error to the caller:

```go
err := s.k8s.Get(ctx, key, existing)
if err == nil {
    return nil, api_errors.NewErrConflict(fmt.Sprintf("tool %s/%s already exists", ...))
}
```

`Delete` is idempotent: if the resource is already gone, the method returns `nil` rather than an error. This matches
HTTP DELETE semantics and avoids spurious errors when multiple delete requests arrive in quick succession.

### 3.9.5 Error Handling

Error handling is centralised in `response.go` through the `respondError` function. It uses `errors.As` to match typed
errors from the `api_errors` package and map them to HTTP status codes:

```go
func respondError(w http.ResponseWriter, err error) {
    switch {
    case errors.As(err, &notFound):
        statusCode = http.StatusNotFound             // 404
    case errors.As(err, &conflict):
        statusCode = http.StatusConflict             // 409
    case errors.As(err, &validation):
        statusCode = http.StatusUnprocessableEntity  // 422
    default:
        statusCode = http.StatusInternalServerError  // 500
    }
    respondJSON(w, statusCode, ErrorResponse{Error: errorType, Message: err.Error(), Code: statusCode})
}
```

The three semantic error types — `ErrNotFound`, `ErrConflict`, `ErrValidation` — are defined in the `api_errors` package
and implement the `error` interface. Using `errors.As` rather than a type switch ensures that wrapped errors are
correctly matched, preserving error chain information while still producing the correct HTTP status.

All error responses share a consistent JSON envelope:

```json
{
  "error": "Not Found",
  "message": "tool default/trivy not found",
  "code": 404
}
```

The Angular UI relies on this consistent structure: the HTTP interceptor reads `err.error?.error` to extract the
human-readable message for display in the notification toast.

---

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
