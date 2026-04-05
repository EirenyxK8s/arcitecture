# Chapter 3E: Practical Implementation — The REST API

---

## 3.9 The REST API — eirenyx-api

### 3.9.1 Purpose and Role in the System

The `eirenyx-api` component is a stateless Go HTTP server that acts as the bridge between the Eirenyx web dashboard and
the Kubernetes API. Its purpose is to expose the three Eirenyx CRDs (`Tool`, `Policy`, `PolicyReport`) as a JSON REST
interface that any HTTP client can consume, without requiring direct access to the Kubernetes API server or knowledge of
the Kubernetes resource model.

There are several concrete reasons why a dedicated API component is needed rather than pointing the UI directly at the
Kubernetes API:

**1. Authentication boundary.** The Kubernetes API server uses certificate-based or token-based authentication. Exposing
it directly to a browser-based UI would require either embedding cluster credentials in the browser (a serious security
risk) or implementing complex OAuth/OIDC flows. The `eirenyx-api` runs inside the cluster as a Kubernetes pod with a
`ServiceAccount` that carries the necessary RBAC permissions. The browser communicates with `eirenyx-api` using ordinary
HTTP; the API server authenticates with the cluster on the browser's behalf.

**2. Schema translation.** Kubernetes resource objects carry significant metadata that is irrelevant or confusing for a
dashboard consumer: `managedFields`, `resourceVersion`, `creationTimestamp`, internal status conditions. The API layer
translates Kubernetes objects to clean DTOs that contain only the fields the UI needs.

**3. Error normalisation.** The Kubernetes API returns errors in its own format, with Kubernetes-specific status codes
and reason strings. The `eirenyx-api` maps these to standard HTTP status codes and a consistent JSON error envelope that
the UI's error handling layer can process uniformly.

**4. Abstraction stability.** If the Eirenyx CRD schema changes, the REST API can absorb the change without requiring
updates to the UI. The DTO mapping layer acts as a decoupling boundary between the internal data model and the external
API contract.

### 3.9.2 Why Go and chi

The `eirenyx-api` is implemented in Go for the same reasons as the operator: Go's standard library HTTP stack is
production-grade, the controller-runtime `client.Client` that the operator uses is reusable in the API without any
additional dependencies, and the single-binary deployment model matches well with containerised Kubernetes workloads.

The HTTP router is **chi** (`github.com/go-chi/chi/v5`). Chi was chosen over alternatives for several reasons:

- It is built on Go's `net/http` standard library, meaning any middleware or handler written for `net/http` is
  compatible without adaptation.
- It provides clean URL parameter extraction (`chi.URLParam(r, "namespace")`) without reflection or code generation.
- It has a composable, functional middleware model that makes it straightforward to apply different middleware sets to
  different route groups.
- It has zero non-standard-library dependencies in its core.

The `chi/middleware` package provides production-ready middleware components (`RequestID`, `RealIP`, `Logger`,
`Recoverer`) that would otherwise require custom implementation.

### 3.9.3 The Startup Model — Controller Manager + HTTP Server

An important architectural detail of `eirenyx-api` is that it is scaffolded as a **Kubebuilder controller manager**, not
as a plain HTTP server. The `cmd/main.go` entry point starts a controller-runtime Manager, registers health probes, and
then starts the chi HTTP server as a goroutine alongside the manager.

```go
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme:                 scheme,
    Metrics:                metricsServerOptions,
    HealthProbeBindAddress: probeAddr,           // :8081
    LeaderElection:         enableLeaderElection,
    LeaderElectionID:       "04ab6b3a.eirenyx-api",
})

apiRouter := mvc.NewRouter(mgr.GetClient())
apiServer := &http.Server{
    Addr:    httpAddr,  // :8888
    Handler: apiRouter,
}
go func() {
    if err := apiServer.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
        setupLog.Error(err, "HTTP server error")
        os.Exit(1)
    }
}()

// Graceful shutdown on SIGTERM/SIGINT
go func() {
    <-ctx.Done()
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    _ = apiServer.Shutdown(shutdownCtx)
}()

mgr.Start(ctx)
```

This design has a key advantage: the HTTP server receives the controller manager's `client.Client` — the same caching
client backed by informers that the operator reconcilers use. This client is not a raw Kubernetes API client; it has a
local in-memory cache (populated by list-and-watch informers) that means many `Get` and `List` calls are served from
cache without hitting the Kubernetes API server. The `eirenyx-api` benefits from this caching automatically, with no
additional configuration.

The `scheme` registered at startup includes both the standard Kubernetes types (`clientgoscheme`) and the Eirenyx CRD
types (`eirenyxv1alpha1`). Without both being registered, the `client.Client` cannot deserialise Kubernetes native types
or Eirenyx custom types respectively.

Graceful shutdown is implemented via a `context.WithTimeout` that gives the HTTP server up to 10 seconds to finish
in-flight requests when the pod receives a `SIGTERM` signal, ensuring that Kubernetes rolling updates do not abruptly
terminate active HTTP sessions.

### 3.9.4 Cluster Deployment

The `eirenyx-api` is deployed as a Kubernetes `Deployment` with a single replica. The Deployment manifest, managed by
Kustomize, defines the full operational configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: controller-manager
  namespace: eirenyx-system
spec:
  replicas: 1
  template:
    spec:
      serviceAccountName: controller-manager
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: manager
          command: [ "/manager" ]
          args:
            - --leader-elect
            - --health-probe-bind-address=:8081
            - --http-bind-address=:8888
          image: controller:latest
          ports:
            - containerPort: 8888
              name: http
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ "ALL" ]
          resources:
            limits: { cpu: 500m, memory: 128Mi }
            requests: { cpu: 10m,  memory: 64Mi }
          livenessProbe:
            httpGet: { path: /healthz, port: 8888 }
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /readyz, port: 8888 }
            initialDelaySeconds: 5
            periodSeconds: 10
```

Several security hardening choices are notable:

- **`runAsNonRoot: true`** — The container runs as UID 65532 (the `nonroot` user from the distroless base image), not as
  root. This limits the blast radius of a container escape.
- **`readOnlyRootFilesystem: true`** — The container cannot write to its own filesystem. The binary has no need to write
  to disk, so this is a zero-cost hardening.
- **`allowPrivilegeEscalation: false`** — The process cannot gain more privileges than it starts with, even if a
  vulnerability allows arbitrary code execution.
- **`capabilities.drop: ["ALL"]`** — All Linux capabilities are dropped. The API server needs no special kernel
  capabilities.

The container image is built with a two-stage Dockerfile:

```dockerfile
FROM golang:1.25 AS builder
WORKDIR /workspace
COPY go.mod go.sum .
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -o manager cmd/main.go

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /workspace/manager .
USER 65532:65532
ENTRYPOINT ["/manager"]
```

The use of `gcr.io/distroless/static:nonroot` as the runtime base image is significant: distroless images contain no
shell, no package manager, no `/tmp` directory, and no utilities. The attack surface is reduced to the Go binary itself.
If an attacker gains code execution inside the container, they have no shell to launch, no tools to use, and no network
utilities to exfiltrate data with.

A Kubernetes `Service` of type `ClusterIP` exposes the API server within the cluster:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: http-service
  namespace: eirenyx-system
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: eirenyx-api
  ports:
    - name: http
      port: 80
      targetPort: 8888
```

External access is provided through a Kubernetes `Ingress` or a load balancer service — the choice depends on the
cluster environment. In development, port-forwarding (`kubectl port-forward svc/http-service 8888:80`) provides local
access.

### 3.9.5 RBAC Permissions

The ServiceAccount associated with the `eirenyx-api` pod holds a `ClusterRole` with specific permissions covering the
resources the API server needs to read and write:

```yaml
rules:
  - apiGroups: [ "eirenyx.eirenyx" ]
    resources: [ "tools", "policies" ]
    verbs: [ "create", "delete", "get", "list", "patch", "update", "watch" ]
  - apiGroups: [ "eirenyx.eirenyx" ]
    resources: [ "policyreports" ]
    verbs: [ "get", "list", "watch" ]
  - apiGroups: [ "eirenyx.eirenyx" ]
    resources: [ "tools/status", "policies/status", "policyreports/status" ]
    verbs: [ "get", "patch", "update" ]
  - apiGroups: [ "batch" ]
    resources: [ "jobs" ]
    verbs: [ "create", "delete", "get", "list", "patch", "update", "watch" ]
```

The `PolicyReport` resource is intentionally limited to read-only verbs (`get`, `list`, `watch`). The API layer enforces
this: there are no `POST`, `PUT`, or `DELETE` endpoints for reports. Reports are created and managed exclusively by the
operator. This read-only restriction is enforced at both the application level (no routes defined) and the RBAC level (
no write verbs granted), providing defence in depth.

### 3.9.6 MVC Structure and Request Lifecycle

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
      tool.go           — Tool request/response structs and mapping
      policy.go         — Policy response structs
      policy_request.go — Policy create/update request structs
      report.go         — PolicyReport response structs
      base_resource.go  — Shared BaseResource struct
  api_errors/
    error.go            — Typed error sentinel types
```

A request follows a strict four-step path through this structure:

```
HTTP Request
     ↓
Controller — extracts path params, decodes body, runs validation
     ↓
Service — calls Kubernetes API, translates K8s errors to typed errors
     ↓
DTO — maps K8s objects to JSON response shapes
     ↓
HTTP Response
```

The controller layer never touches the Kubernetes API directly. The service layer never touches `http.ResponseWriter`.
The DTO layer never makes API calls. This separation is enforced by the package structure: importing across these
boundaries in the wrong direction would create a circular import that the Go compiler would reject.

### 3.9.7 Router and Middleware

The `NewRouter` function accepts the controller-runtime `client.Client`, instantiates all four service and controller
layers, and wires the complete route tree:

```go
func NewRouter(k8s client.Client) http.Handler {
    r := chi.NewRouter()

    r.Use(middleware.RequestID)    // Unique ID on every request
    r.Use(middleware.RealIP)       // Extract real client IP from X-Forwarded-For
    r.Use(middleware.Logger)       // Structured access logging
    r.Use(middleware.Recoverer)    // Recover from panics → 500

    r.Use(cors.Handler(cors.Options{
        AllowedOrigins: []string{"*"},
        AllowedMethods: []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
        AllowedHeaders: []string{"Accept", "Authorization", "Content-Type"},
        MaxAge:         300,
    }))
    // ...route registration...
}
```

The CORS policy (`AllowedOrigins: ["*"]`) is permissive because authentication is delegated entirely to Kubernetes RBAC.
The API server does not authenticate HTTP clients itself — it operates under the assumption that network-level access to
the pod is already restricted by Kubernetes `NetworkPolicy`. For deployments that require stricter CORS, the allowed
origins can be scoped to the UI's specific hostname via the Kustomize patch mechanism.

The route table maps directly to the CRD hierarchy:

| Method   | Path                                         | Action                                  |
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

### 3.9.8 Error Handling — The Typed Error Model

Error handling is centralised in `response.go`. The service layer translates Kubernetes errors into typed sentinel
errors from the `api_errors` package:

```go
func (s *ToolService) Get(ctx context.Context, namespace, name string) (*dto.Tool, error) {
    tool := &eirenyxv1alpha1.Tool{}
    if err := s.k8s.Get(ctx, types.NamespacedName{Namespace: namespace, Name: name}, tool); err != nil {
        if k8serrors.IsNotFound(err) {
            return nil, api_errors.NewErrNotFound(fmt.Sprintf("tool %s/%s not found", namespace, name))
        }
        return nil, fmt.Errorf("getting tool %s/%s: %w", namespace, name, err)
    }
    return dto.MapToTool(*tool), nil
}
```

The `respondError` function in the controller layer maps these typed errors to HTTP status codes using `errors.As`:

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
    respondJSON(w, statusCode, ErrorResponse{
        Error:   errorType,
        Message: err.Error(),
        Code:    statusCode,
    })
}
```

Using `errors.As` rather than a type switch ensures that wrapped errors are correctly matched, preserving the full error
chain while still producing the correct HTTP response. All error responses share a consistent JSON envelope:

```json
{
  "error": "Not Found",
  "message": "tool default/trivy not found",
  "code": 404
}
```

The Angular UI's HTTP interceptor reads this envelope's `error` field to extract the human-readable message for display
in the notification toast. The consistency of this format is what makes the global interceptor pattern possible — if
different errors had different shapes, each component would need to implement its own error parsing logic.

![Figure 8 — Component Integration and Interface Map](../assets/component-integration.drawio.png)

---

*Previous: [Chapter 3D — Litmus Integration](03d-litmus.md)*
*Next: [Chapter 3F — Web Dashboard](03f-ui.md)*
