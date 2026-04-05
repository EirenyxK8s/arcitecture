# Eirenyx – Development Plan & User Stories

## Epic 0 – Project Bootstrap

### User Story 0.1 – Initialize base Kubernetes operator

**As a platform engineer**,
I want to initialize a Kubernetes operator using Kubebuilder,
so that I have a correct and maintainable foundation for Eirenyx.

#### Steps to achieve

1. Initialize the project with Kubebuilder

   ```bash
   kubebuilder init --domain eirenyx --repo github.com/EirenyxK8s/eirenyx
   ```

2. Verify project structure and manager startup
3. Run the operator locally
4. Commit the initial scaffold

#### Done when

* Operator builds successfully
* Controller manager starts without custom logic
* No CRDs exist yet

---

## Epic 1 – Tool Lifecycle Management

### User Story 1.1 – Define Tool CRD

**As a platform engineer**,
I want a Tool custom resource,
so that Eirenyx can declaratively manage external security and chaos tools.

#### Steps to achieve

1. Generate Tool API using Kubebuilder

   ```bash
   kubebuilder create api --group eirenyx --version v1alpha1 --kind Tool
   ```

2. Define `ToolSpec` (type, enabled, install method, config)
3. Define `ToolStatus` (installed, healthy, version)
4. Add OpenAPI validation
5. Generate manifests

#### Done when

* `Tool` CRD exists
* Validation prevents invalid tool types
* Status subresource is present

---

### User Story 1.2 – Install tools via Tool CRD

**As a platform engineer**,
I want Eirenyx to install Trivy, Falco, or Litmus when a Tool resource is created,
so that tool lifecycle is fully declarative.

#### Steps to achieve

1. Implement `ToolReconciler`
2. Detect `spec.enabled == true`
3. Create namespace if missing
4. Install tool using selected strategy (Helm first)
5. Update Tool status

#### Done when

* Creating a `Tool` installs the corresponding tool
* Status reflects installation success or failure
* Reconciliation is idempotent

---

### User Story 1.3 – Disable or remove tools

**As a platform engineer**,
I want to disable a tool using the Tool CRD,
so that unused components can be removed safely.

#### Steps to achieve

1. Detect `spec.enabled == false`
2. Uninstall tool (or mark as inactive)
3. Update status accordingly

#### Done when

* Tool can be cleanly disabled
* No orphaned resources remain

---

## Epic 2 – Policy Definition & Abstraction

### User Story 2.1 – Define unified Policy CRD

**As a DevOps engineer**,
I want a single Policy CRD,
so that I can configure Trivy, Falco, and Litmus through one interface.

#### Steps to achieve

1. Generate Policy API
2. Define PolicySpec sections:

    * trivy
    * falco
    * litmus
3. Reference required tools
4. Add validation rules

#### Done when

* One Policy CRD configures multiple tools
* Policy fails validation if required Tool is missing

---

### User Story 2.2 – Reconcile Policy to actions

**As a DevOps engineer**,
I want Eirenyx to trigger tools based on policy configuration,
so that security and resilience checks run automatically.

#### Steps to achieve

1. Implement `PolicyReconciler`
2. Verify referenced Tool health
3. Trigger:
    * Trivy scans
    * Falco rule activation / event subscription
    * Litmus chaos experiments
4. Track execution state

#### Done when

* Policy drives tool behavior
* Reconciliation is deterministic
* No direct CLI calls are used

---

## Epic 3 – Result Collection & Correlation

### User Story 3.1 – Collect tool results

**As a security engineer**,
I want Eirenyx to collect outputs from all tools,
so that results are centrally visible.

#### Steps to achieve

1. Watch Trivy result CRDs
2. Consume Falco events
3. Observe Litmus experiment results
4. Normalize data into internal model

#### Done when

* Results are accessible to Eirenyx
* Source tool is preserved per finding

---

### User Story 3.2 – Correlate findings

**As a security engineer**,
I want correlated security and resilience findings,
so that real risks are prioritized.

#### Steps to achieve

1. Define correlation rules
2. Combine:
    * vulnerabilities
    * runtime alerts
    * resilience failures
3. Assign severity

#### Done when

* Correlation logic is documented
* Same workload findings are grouped

---

## Epic 4 – Reporting

### User Story 4.1 – Generate policy report

**As a platform engineer**,
I want a report per policy,
so that cluster state is easy to assess.

#### Steps to achieve

1. Define `EirenyxReport` CRD or Policy status schema
2. Populate report during reconciliation
3. Store references to tool outputs

#### Done when

* Each policy has a final report
* Report is Kubernetes-native
* Report survives controller restarts

---

## Development Principles

* Kubernetes-native first
* No shelling out to CLIs
* Idempotent reconciliation
* Declarative over imperative
* Clear separation of:
    * Tool lifecycle
    * Policy behavior
    * Reporting
