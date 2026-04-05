# Chapter 4: Conclusion, Limitations, and Future Work

---

## 4.1 Summary of Achieved Results

This thesis set out to address a well-defined gap in the Kubernetes security tooling ecosystem: the absence of an
open-source, Kubernetes-native operator that provides a unified declarative interface for the full lifecycle of security
and resilience tooling — from installation through policy configuration to normalised reporting — without replacing the
underlying tools or requiring proprietary cloud infrastructure.

The six requirements established in section 1.7.2 provide the framework for evaluating what was achieved.

### R1 — Non-invasive integration

Eirenyx manages Trivy, Falco, and LitmusChaos without modifying their source code, their Helm charts, or their internal
CRD schemas. Each tool continues to function independently; the operator adds a management layer on top. Trivy still
produces `VulnerabilityReport` CRDs through its own operator; Falco still runs its kernel-level rules engine; Litmus
still executes experiments through its own chaos-operator. Eirenyx is a consumer and orchestrator of these tools, not a
replacement.

**Status: Achieved.** All three tools are installed and managed through the Helm Go SDK, which interacts with official
upstream charts. No tool-specific binary is included in the operator image.

### R2 — Declarative lifecycle management

The `Tool` CRD and `ToolReconciler` provide fully declarative lifecycle management for all three tools. Creating a
`Tool` resource installs the corresponding Helm chart; setting `spec.enabled: false` uninstalls it; deleting the `Tool`
object triggers an orderly Helm release cleanup via the finalizer pattern. No `kubectl` commands, no Helm CLI
invocations, and no shell scripts are used anywhere in the operator.

**Status: Achieved.** The development principle "no shelling out to CLIs" (documented in the user stories) is fulfilled
throughout the implementation. User Story 1.2 (install tools via Tool CRD) and User Story 1.3 (disable or remove tools)
are fully implemented.

### R3 — Unified policy model

The `Policy` CRD provides a single entry point for configuring all three tools. A Trivy vulnerability scan, a Falco
runtime detection rule, and a Litmus chaos experiment are all expressed as `Policy` resources with a common top-level
structure (`type`, `enabled`, `target`) and tool-specific sub-specifications (`spec.trivy`, `spec.falco`,
`spec.litmus`). User Story 2.1 (unified Policy CRD) and User Story 2.2 (reconcile policy to actions) are fully
implemented.

**Status: Achieved.** The factory pattern and the three `Engine` implementations (`TrivyEngine`, `FalcoEngine`,
`LitmusEngine`) provide the tool-specific execution layer while keeping the `PolicyReconciler` itself entirely
tool-agnostic. A platform engineer configures all three tools through the same interface.

### R4 — Normalised reporting

All three tools produce `PolicyReport` resources with a common schema: a `Phase` progression (
`Pending → Running → Completed`), a `Summary` with a binary `Verdict` (`Pass`/`Fail`), `TotalChecks`, `Passed`, and
`Failed` counts, and a `Details` payload containing tool-specific findings. User Story 4.1 (generate policy report) is
fully implemented. User Story 3.1 (collect tool results) is partially implemented — see Limitations.

**Status: Partially achieved.** Trivy and Falco report handlers produce verdict-bearing reports with full findings data.
The Litmus report handler produces a scheduling confirmation report. Cross-tool result correlation (User Story 3.2) is
identified as a future work item.

### R5 — Kubernetes-native state

No external database, configuration file, or in-memory state is used by any Eirenyx component. All persistent state —
tool installation intent, policy configuration, and scan findings — lives in Custom Resources stored in etcd. The
operator, API, and UI can all be restarted at any time without data loss. The controller-runtime caching client ensures
that repeated reads do not generate API server load.

**Status: Achieved.** This requirement is fulfilled by the three-CRD data model and the exclusive use of the
controller-runtime client throughout all components.

### R6 — GitOps compatibility

Because all Eirenyx configuration is expressed as Kubernetes YAML manifests, the entire security posture of a cluster —
which tools are installed, which policies are active, what images are being scanned, which Falco rules are observed,
which chaos experiments are scheduled — can be version-controlled in a Git repository and applied by GitOps
controllers (Flux, ArgoCD). A pull request to the security configuration repository is the mechanism for changing
security policy, giving security teams full audit visibility over configuration changes.

**Status: Achieved.** All three CRD types are fully declarative and YAML-serialisable. The development principle "
declarative over imperative" is upheld throughout the implementation.

---

## 4.2 Architectural Contributions

Beyond the user story coverage, the implementation makes several architectural contributions that have value
independently of the three specific tools integrated.

**The three-interface abstraction model.** The `ToolService`, `Engine`, and `Handler` interfaces establish a clear,
minimal contract for integrating any security or resilience tool into the Eirenyx framework. The three interfaces
together describe the full lifecycle of a managed tool: installation (`ToolService`), policy execution (`Engine`), and
result collection (`Handler`). A new tool — a network policy scanner, an image signing verifier, an SBOM generator — can
be integrated by implementing these three interfaces and adding three lines to the factory functions. No controller code
changes. This extensibility-by-convention is the primary architectural contribution of the system.

**The generation-based staleness model.** The use of `metadata.generation` as the change signal for `PolicyReport`
invalidation — comparing the policy's current generation against the generation stored in the report's spec — provides a
clean, race-free mechanism for ensuring that scan results always reflect the current policy configuration. This pattern
is reusable in any Kubernetes operator that needs to invalidate derived resources when their source changes.

**The factory-as-single-dispatch pattern.** Concentrating all tool-type branching into three factory functions (
`NewToolService`, `NewPolicyEngine`, `NewReportEngine`) ensures that the controllers remain entirely free of conditional
logic based on tool type. This is a direct application of the open-closed principle at the Kubernetes operator level:
the system is closed to modification in its controller layer and open to extension through new factory cases and
interface implementations.

---

## 4.3 Limitations

### 4.3.1 Litmus Report Handler — Scheduling Confirmation Only

The `LitmusReportHandler` current implementation records which chaos experiments were *scheduled*, not whether they
*passed or failed*. The handler generates a `PolicyReport` with a `Pass` verdict immediately upon confirming that the
`ChaosEngine` CRDs were created, without waiting for or inspecting `ChaosResult` CRDs.

This is a deliberate initial scope decision: establishing that experiments are being run on an auditable, declarative
basis is a meaningful first step. However, the absence of `ChaosResult` interpretation means the Litmus `PolicyReport`
cannot currently produce a `Fail` verdict — a workload that crashes under pod deletion will produce the same `Pass`
report as one that recovers successfully.

This is the most significant functional gap between the current implementation and the fully realised requirements of
User Story 3.1.

### 4.3.2 Cross-Tool Finding Correlation

User Story 3.2 — correlating findings from Trivy, Falco, and Litmus to produce a unified risk assessment of a specific
workload — is not implemented. Each `PolicyReport` is currently scoped to a single tool's findings. A workload that has
a below-threshold Trivy CVE, an active Falco alert, and a failed Litmus experiment cannot currently be identified as
high-risk from within Eirenyx — the three findings exist as separate objects with no programmatic link to each other
beyond the shared namespace.

The section 1.6.2 integration gap scenario — where the combination of signals is more significant than any individual
signal — cannot be automatically detected by the current implementation.

### 4.3.3 Single-Namespace Tool Lookup Convention

The `PolicyReconciler` locates the `Tool` that owns a `Policy` by assuming the tool's name matches its type within the
same namespace: a policy of `spec.type: trivy` expects a `Tool` named `trivy` in the same namespace. This naming
convention is simple and eliminates a foreign-key reference field in the `PolicySpec`, but it means:

- Only one instance of each tool type can be active per namespace.
- Policies cannot reference tools in other namespaces.
- Renaming a `Tool` resource breaks all policies of that type.

This constraint is acceptable in single-cluster, single-team deployments but becomes a limitation in multi-tenant
environments where different teams may require different Trivy configurations in the same namespace.

### 4.3.4 CORS Wildcard Policy

The `eirenyx-api` is configured with `AllowedOrigins: ["*"]` — accepting requests from any origin. This is acceptable
when network-level access to the API pod is controlled by Kubernetes `NetworkPolicy`, but it represents a security
surface that should be tightened in production deployments. The current implementation does not provide a configuration
mechanism for restricting allowed origins without modifying the Kustomize manifests directly.

### 4.3.5 Falco Alert Counting — Observation Window

The `FalcoReportHandler` determines its verdict by counting event occurrences of the observed rule. The current
implementation does not define an explicit observation window — it counts events that are available at the moment the
handler runs. This means the verdict is a snapshot at reconciliation time rather than a count over a defined time
period, which limits its use for trend analysis or baseline deviation detection.

### 4.3.6 Authentication Delegation

The `eirenyx-api` delegates authentication entirely to the Kubernetes RBAC system by relying on the ambient
ServiceAccount identity of its pod. While this is architecturally clean, it means there is currently no mechanism for
multi-user access control at the API level — any process that can reach the API pod over the network has the full
permissions of its ServiceAccount. A production deployment would benefit from OIDC-based authentication forwarded to the
Kubernetes API server's tokenreview endpoint.

---

## 4.4 Future Work

The limitations described above, together with the unimplemented User Story 3.2, form a natural roadmap for future
development.

### 4.4.1 Litmus ChaosResult Integration

The highest-priority extension is completing the Litmus result collection pipeline. The `LitmusReportHandler` should be
extended to:

1. Poll `ChaosResult` CRDs in the target namespace after the `ChaosEngine` moves to a terminal state.
2. Parse the `ChaosResult.Status.ExperimentStatus.Verdict` field (`Pass`/`Fail`/`Stopped`).
3. Aggregate verdicts across all experiments in the policy.
4. Produce a `Fail` verdict if any experiment's steady-state hypothesis was violated.

This would complete User Story 3.1 and make the Litmus `PolicyReport` a genuine resilience assertion rather than a
scheduling record.

### 4.4.2 Cross-Tool Finding Correlation

User Story 3.2 requires a correlation model that links findings across tool types to a common workload identity. A
practical implementation would:

- Introduce a `WorkloadReport` CRD that aggregates `PolicyReport` objects sharing the same target workload (identified
  by namespace + label selector).
- Implement a `WorkloadReportReconciler` that computes a composite risk score from the individual tool verdicts,
  weighted by severity.
- Expose the `WorkloadReport` through the REST API and display it on a dedicated dashboard page showing the three-pillar
  security posture of each workload at a glance.

This would address the multi-signal risk scenario from section 1.6.2: a workload with a medium CVE, a Falco alert, and a
failed Litmus test would produce a `High` composite risk even though no individual signal crosses its alert threshold.

### 4.4.3 Scheduled Scanning

The current implementation runs Trivy scans on-demand — when a `Policy` is created or its specification changes. A
useful extension would be a `schedule` field in the `TrivyPolicySpec` (using standard cron syntax) that triggers
periodic re-scans. This is necessary because new CVEs are disclosed continuously: an image that passed a scan on Monday
may fail by Thursday when a new vulnerability is published against one of its dependencies. Scheduled scanning would
bring Eirenyx closer to the trivy-operator's continuous scanning model while preserving the policy-driven control plane
model.

### 4.4.4 Named Tool References

Replacing the implicit naming convention (a policy of `spec.type: trivy` binds to a `Tool` named `trivy`) with an
explicit `spec.toolRef.name` field in the `PolicySpec` would remove the single-namespace-per-tool-type constraint. This
would enable multi-tenant deployments where multiple teams maintain separate tool instances with different
configurations in the same cluster, and policies can explicitly reference the tool instance they intend to use.

### 4.4.5 OIDC Authentication at the API Layer

Extending `eirenyx-api` with OIDC authentication — forwarding the user's bearer token to the Kubernetes API server's
`TokenReview` endpoint — would provide per-user access control without introducing a separate identity provider. This
pattern (used by the Kubernetes dashboard and by tools like Headlamp) allows the API server's existing RBAC to control
which users can create or delete `Tool` and `Policy` resources, rather than granting all access to the API pod's
ServiceAccount.

### 4.4.6 Additional Tool Integrations

The factory pattern and the three-interface model were designed with extensibility in mind. Candidate tools for future
integration include:

| Tool                  | Purpose                                          | Integration type                                                           |
|-----------------------|--------------------------------------------------|----------------------------------------------------------------------------|
| **Kyverno**           | Admission control and policy enforcement         | Engine creates `Policy` objects; Handler reads `PolicyReport`              |
| **Cosign / Sigstore** | Container image signature verification           | Engine triggers signature verification job; Handler reads result           |
| **OWASP ZAP**         | Dynamic application security testing             | Engine creates scan job; Handler reads JSON findings                       |
| **Terrascan**         | Infrastructure-as-code misconfiguration scanning | Engine creates job against manifest repository; Handler reads SARIF output |

Each of these could be integrated by adding a new `ToolType` constant, implementing the three interfaces, and adding
three `case` statements to the factory functions — with no changes to the controller layer.

### 4.4.7 Configurable Verdict Thresholds

The current verdict logic is hardcoded: Trivy fails on CRITICAL/HIGH, Falco fails on any event count > 0. A
`VerdictPolicy` configuration field in each tool's sub-spec would allow platform teams to tune these thresholds — for
example, treating CRITICAL-only as `Fail` in production but CRITICAL/HIGH/MEDIUM as `Fail` in development environments.
This would reduce alert fatigue in less critical environments while maintaining strict posture where it matters.

---

## 4.5 Closing Statement

Eirenyx demonstrates that it is possible to build a unified, open-source control plane for Kubernetes security tooling
that satisfies all core requirements — declarative lifecycle management, unified policy configuration, normalised
reporting, and GitOps compatibility — without replacing or forking the underlying tools, without requiring proprietary
cloud infrastructure, and without introducing an external state store.

The three-CRD data model (`Tool`, `Policy`, `PolicyReport`), the factory pattern, and the three-interface abstraction
together form a framework that is both immediately useful and straightforwardly extensible. The six epics defined at
project inception have been substantially delivered: all tool lifecycle management, policy reconciliation, and reporting
infrastructure is implemented and operational. The remaining work — `ChaosResult` interpretation, cross-tool
correlation, and scheduled scanning — represents genuine depth of the problem domain rather than missing foundations.

The broader contribution is a reference architecture for the pattern of *operator-as-integration-layer*: using the
Kubernetes operator pattern not to manage a single complex application, but to provide a unified declarative interface
over a heterogeneous collection of existing tools. This pattern is applicable beyond the security domain — anywhere that
multiple independently-developed Kubernetes-native tools need to be operated as a coherent system by a team that should
not be required to understand each tool's internal model.

---

*Previous: [Chapter 3F — Web Dashboard](03f-ui.md)*
*Next: Chapter 5 — References (to be added)*
