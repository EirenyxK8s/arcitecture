# Eirenyx – Unified Policy Operator for Kubernetes

> Inspired by **Eirene**, the Greek goddess of peace and order, Eirenyx brings clarity, control, and stability to
> cloud-native environments.

Eirenyx is a Kubernetes-native security and resilience platform composed of three components:

| Component     | Language         | Role                                     |
|---------------|------------------|------------------------------------------|
| `eirenyx`     | Go (Kubebuilder) | Kubernetes Operator — core engine        |
| `eirenyx-api` | Go               | REST/gRPC API exposing operator state    |
| `eirenyx-ui`  | Angular          | Web dashboard for visibility and control |

---

## Purpose

Modern Kubernetes environments require three complementary security concerns to be addressed simultaneously:

- **Pre-runtime security** — are vulnerabilities present before workloads run?
- **Runtime security** — is anything anomalous happening while workloads execute?
- **Resilience** — do workloads recover correctly when failures occur?

Each of these is typically addressed by a separate, independently operated tool. Eirenyx unifies them under a single
declarative interface, removes the operational burden of managing heterogeneous tooling, and surfaces correlated
findings through a consolidated report model.

---

## Integrated Tools

### Trivy – Vulnerability & Configuration Scanner

Trivy scans container images, Kubernetes workload manifests, and infrastructure configurations for known CVEs and
misconfigurations.

- Detects CVEs in container images (pre-runtime)
- Performs Kubernetes resource misconfiguration checks
- Generates Software Bill of Materials (SBOM)
- Operates against workloads before they become active

**Role in Eirenyx:** pre-runtime security validation layer

**Example:** Identify critical CVEs in images used by active Deployments.

---

### Falco – Runtime Security & Behavioral Detection

Falco is a Kubernetes-native runtime security engine that monitors system calls and Kubernetes audit events in real time
using declarative rule definitions.

- Detects anomalous or malicious container behavior
- Monitors syscall activity at the kernel level
- Continuously observes container and node execution
- Focused on intrusion detection during live workload execution

**Role in Eirenyx:** runtime visibility and intrusion detection layer

**Example:** Detect a container spawning a shell or attempting privilege escalation.

> Note: Falco is chosen over admission-focused policy engines (e.g. OPA/Gatekeeper) deliberately. Admission controllers
> validate what _may_ be deployed; Falco observes what _actually happens_ at runtime. For operational security monitoring,
> runtime detection provides stronger alignment with intrusion detection principles.

---

### Litmus – Chaos Engineering Framework

Litmus is a Kubernetes-native chaos engineering framework that validates workload resilience by injecting controlled
failure conditions.

- Simulates pod failures, network delays, resource stress
- Tests application recovery behavior and fault tolerance
- Validates that SLOs hold under adverse conditions

**Role in Eirenyx:** resilience verification layer

**Example:** Verify that an application recovers automatically after random pod deletion.

---

## Security Lifecycle Alignment

| Phase       | Tool   | Purpose                                    |
|-------------|--------|--------------------------------------------|
| Pre-runtime | Trivy  | Vulnerability and configuration scanning   |
| Runtime     | Falco  | Behavioral monitoring and threat detection |
| Resilience  | Litmus | Fault injection and recovery validation    |

These tools cover complementary, non-overlapping phases of the security lifecycle.

---

## Architecture

```
                        ┌──────────────────────────────────────────────────┐
                        │                  K8s Cluster                     │
                        │                                                  │
                        │   ┌─────────────────┐                            │
User ──CRUD CRDs──────▶ │   │ Eirenyx Operator │                           │
                        │   └────────┬────────┘                            │
                        │            │                                     │
                        │     ┌──────┴───────┐                             │
                        │     │ Eirenyx Policy│ (CRD)                      │
                        │     └──┬───┬───┬───┘                             │
                        │        │   │   │                                 │
                        │     ┌──▼─┐ ┌▼──┐ ┌──▼──┐                         │
                        │     │Lit-│ │Tri│ │Falco│  (managed tools)        │
                        │     │mus │ │vy │ │     │                         │
                        │     └──┬─┘ └─┬─┘ └──┬──┘                         │
                        │        │     │       │                           │
                        │  ┌─────▼┐ ┌──▼──┐ ┌─▼──────┐                     │
                        │  │Litmus│ │Trivy│ │ Falco  │  (reports)          │
                        │  │Report│ │Reprt│ │ Report │                     │
                        │  └──────┘ └──┬──┘ └────────┘                     │
                        │              │          │                        │
                        │         ┌────▼──────────▼─────┐                  │
                        │         │   Eirenyx Report    │  (CRD)           │
                        │         └─────────────────────┘                  │
                        └──────────────────────────────────────────────────┘
                                          │
                                    eirenyx-api
                                    (REST/gRPC)
                                          │
                                    eirenyx-ui
                                    (Angular SPA)
```

---

## Component: `eirenyx` — Kubernetes Operator

Built with **Kubebuilder** (controller-runtime). The operator is the core engine of the platform.

### CRD Model

#### `Tool` CRD

Manages the lifecycle of integrated security tools within the cluster.

```yaml
apiVersion: eirenyx/v1alpha1
kind: Tool
metadata:
  name: trivy
spec:
  type: trivy        # trivy | falco | litmus
  enabled: true
  installMethod: helm
status:
  installed: true
  healthy: true
  version: "0.49.1"
```

- The `ToolReconciler` watches `Tool` resources
- When `spec.enabled: true`, the operator installs the tool (via Helm by default)
- When `spec.enabled: false`, the tool is cleanly uninstalled with no orphaned resources
- Status reflects installation success, health, and version

#### `Policy` CRD

The unified policy interface. A single `Policy` resource configures behavior across all three tools.

```yaml
apiVersion: eirenyx/v1alpha1
kind: Policy
metadata:
  name: workload-security
spec:
  targetNamespace: production
  trivy:
    severity: [ CRITICAL, HIGH ]
    scanOnSchedule: "0 2 * * *"
  falco:
    rulesets: [ default, privileged-access ]
  litmus:
    experiment: pod-delete
    targetLabel: app=payments
```

- The `PolicyReconciler` reads the policy and drives each tool's behavior declaratively
- No CLI calls are made — all integration is API or CRD-based
- Reconciliation is idempotent and deterministic

#### `EirenyxReport` CRD

A structured, consolidated report produced per policy.

- Aggregates findings from Trivy, Falco, and Litmus reports
- Survives controller restarts (persisted as a K8s object)
- Includes source tool per finding, severity, and correlation metadata
- Correlated findings group vulnerabilities + runtime alerts + resilience failures for the same workload

### Operator Design Principles

- **Kubernetes-native first** — all state is expressed as Kubernetes objects
- **No shelling out** — tool interaction is entirely through APIs and CRDs
- **Idempotent reconciliation** — applying the same resource twice has no side effects
- **Declarative over imperative** — users express desired state; the operator drives toward it
- **Lifecycle management** — finalizers ensure clean removal of managed resources
- **Clear separation of concerns:**
    - `ToolReconciler` — tool installation and health
    - `PolicyReconciler` — policy evaluation and tool orchestration
    - Report generation — structured output collection and correlation

### Operator Reconciliation Flow

```
[Policy created/updated]
        │
        ▼
PolicyReconciler
        │
        ├── Verify referenced Tools are healthy
        │
        ├── Trigger Trivy scan (via CRD / API)
        ├── Apply Falco rules (via CRD)
        ├── Schedule Litmus experiment (via CRD)
        │
        ├── Watch for tool-specific reports
        │
        └── Correlate findings → write EirenyxReport
```

---

## Component: `eirenyx-api`

A backend service running inside the cluster that exposes operator state to external consumers.

**Responsibilities:**

- Serve cluster data (policies, reports, tool status) over REST or gRPC
- Translate Kubernetes CRD state into API responses
- Provide authentication and access control for external consumers
- Act as the integration point between the Kubernetes control plane and the UI

**Connection to the cluster:**

- Runs as a Deployment inside the cluster with RBAC permissions to read Eirenyx CRDs
- Watches `EirenyxReport`, `Policy`, and `Tool` objects via the Kubernetes API server
- Exposed via a `Service` (ClusterIP internally, or Ingress for external access)

---

## Component: `eirenyx-ui`

An Angular Single Page Application providing a web dashboard for platform and security engineers.

**Responsibilities:**

- Visualize active policies and their scope across namespaces
- Display tool installation and health status (`Tool` CRD status)
- Surface correlated security findings from `EirenyxReport` objects
- Allow CRUD operations on `Policy` and `Tool` resources via the API
- Provide drill-down views per tool (Trivy CVEs, Falco alerts, Litmus experiment results)

**Connection in K8s:**

- Served as a static build from an nginx container, deployed as a Kubernetes `Deployment`
- Communicates exclusively with `eirenyx-api` over HTTP/gRPC
- Does not interact with the Kubernetes API server directly
- Exposed via an `Ingress` resource (or LoadBalancer `Service`)

---

## End-to-End Flow

```
[Platform Engineer]
        │
        │  Creates / updates Policy via UI or kubectl
        ▼
[eirenyx-ui]  ──HTTP──▶  [eirenyx-api]  ──K8s API──▶  [Policy CRD]
                                                              │
                                                     [Eirenyx Operator]
                                                              │
                                          ┌───────────────────┼───────────────────┐
                                          ▼                   ▼                   ▼
                                       [Trivy]             [Falco]            [Litmus]
                                          │                   │                   │
                                     [Trivy Report]   [Falco Report]    [Litmus Report]
                                          └───────────────────┼───────────────────┘
                                                              ▼
                                                    [EirenyxReport CRD]
                                                              │
                                              [eirenyx-api]  ◀──K8s watch
                                                              │
                                              [eirenyx-ui]   ◀──HTTP poll/stream
                                                              │
                                         [Security Engineer views findings]
```

---

## Kubernetes Deployment Topology

| Resource                             | Component                   | Notes                                                 |
|--------------------------------------|-----------------------------|-------------------------------------------------------|
| `Deployment`                         | eirenyx operator            | Runs controller-manager, single replica               |
| `Deployment`                         | eirenyx-api                 | REST/gRPC backend, scalable                           |
| `Deployment`                         | eirenyx-ui                  | nginx static SPA, scalable                            |
| `CRD`                                | Tool, Policy, EirenyxReport | Cluster-scoped or namespace-scoped                    |
| `ClusterRole` / `ClusterRoleBinding` | eirenyx operator            | Read/write access to managed CRDs and tool namespaces |
| `Role` / `RoleBinding`               | eirenyx-api                 | Read access to Eirenyx CRDs                           |
| `Service`                            | eirenyx-api                 | ClusterIP, consumed by UI and Ingress                 |
| `Service`                            | eirenyx-ui                  | ClusterIP behind Ingress                              |
| `Ingress`                            | eirenyx-ui + eirenyx-api    | External access point                                 |

---

## Architecture Diagram

![Architecture Diagram](assets/arcitecture.drawio.png)
