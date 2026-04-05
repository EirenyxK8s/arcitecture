# Eirenyx — A Unified Kubernetes Security and Resilience Operator

---

## Table of Contents

1. [Literature Review](01-literature-review.md)
    - 1.1 Kubernetes and the Operator Model
        - 1.1.1 Architecture and Core Principles
        - 1.1.2 Custom Resource Definitions and the Extension API
        - 1.1.3 The Operator Pattern
        - 1.1.4 Kubebuilder and controller-runtime
    - 1.2 Container Security — The Threat Landscape and Tooling Challenges
        - 1.2.1 The Container Security Problem Space
        - 1.2.2 The Operational Fragmentation Problem
        - 1.2.3 Existing Integration Approaches and Their Limitations
    - 1.3 Trivy — Vulnerability and Misconfiguration Scanning
        - 1.3.1 Purpose and Scope
        - 1.3.2 How Trivy Works — Technical Architecture
        - 1.3.3 The trivy-operator
        - 1.3.4 The CVSS Severity Model
    - 1.4 Falco — Runtime Security and Anomaly Detection
        - 1.4.1 Purpose and the Limits of Admission Control
        - 1.4.2 How Falco Works — Kernel-Level Observation
        - 1.4.3 The Falco Rules Engine
        - 1.4.4 Falco Alerting and Output
    - 1.5 LitmusChaos — Chaos Engineering and Resilience Validation
        - 1.5.1 The Chaos Engineering Discipline
        - 1.5.2 LitmusChaos Architecture and CRD Model
        - 1.5.3 How Litmus Integrates with Eirenyx
    - 1.6 How the Three Tools Work Together
        - 1.6.1 Complementary Coverage Model
        - 1.6.2 The Integration Gap
    - 1.7 Justification for a Unified Approach
        - 1.7.1 The Case for a Control Plane
        - 1.7.2 Requirements for an Effective Solution
        - 1.7.3 Eirenyx as a Solution

2. [Architecture and Technologies](02-architecture.md)
    - 2.1 Justification for Architectural Choices
    - 2.2 Go and Kubebuilder for the Operator
    - 2.3 Go and chi for the REST API
    - 2.4 Angular 21 and the Signals API for the UI
    - 2.5 Helm SDK for Tool Lifecycle Management
    - 2.6 Kubernetes CRD Model as the State Store

3. Practical Implementation
    - [3A — CRD Model, Reconcilers, and Unified Interface](03a-operator-crds.md)
        - 3.1 The CRD-Centric Design Philosophy
        - 3.2 The Three-CRD Data Model (Tool, Policy, PolicyReport)
        - 3.3 The Ownership Hierarchy and Cascading Lifecycle
        - 3.4 The Tool Reconciler
        - 3.5 The Policy Reconciler
        - 3.6 The PolicyReport Reconciler
        - 3.7 The Factory Pattern and Core Interfaces
    - [3B — Trivy Integration](03b-trivy.md)
        - 3.6 Purpose and Role in Eirenyx
        - 3.6.1 How Trivy Works in the Cluster
        - 3.6.2 The Trivy Policy Specification
        - 3.6.3 The Trivy Engine — Creating Kubernetes Jobs
        - 3.6.4 The Trivy Report Handler
        - 3.6.5 Why the Report Architecture Matters
    - [3C — Falco Integration](03c-falco.md)
        - 3.7 Purpose and Role in Eirenyx
        - 3.7.1 How Falco Works in the Cluster
        - 3.7.2 The Falco Policy Specification
        - 3.7.3 The Falco Engine — ConfigMap-Based Rule Configuration
        - 3.7.4 The Falco Report Handler
        - 3.7.5 Why Falco Reports Differ from Trivy Reports
    - [3D — Litmus Integration](03d-litmus.md)
        - 3.8 Purpose and Role in Eirenyx
        - 3.8.1 How Litmus Works in the Cluster
        - 3.8.2 The Litmus Policy Specification
        - 3.8.3 The Litmus Engine — Creating ChaosEngine Resources
        - 3.8.4 The Litmus Report Handler
        - 3.8.5 The Role of Litmus in the Overall Security Posture
    - [3E — REST API](03e-api.md)
        - 3.9 Purpose and Role in the System
        - 3.9.1 Why Go and chi
        - 3.9.2 The Startup Model — Controller Manager + HTTP Server
        - 3.9.3 Cluster Deployment and Security Hardening
        - 3.9.4 RBAC Permissions
        - 3.9.5 MVC Structure and Request Lifecycle
        - 3.9.6 Router and Middleware
        - 3.9.7 Error Handling — The Typed Error Model
    - [3F — Web Dashboard](03f-ui.md)
        - 3.10 Purpose and Role in the System
        - 3.10.1 Technology Choices
        - 3.10.2 Application Structure
        - 3.10.3 Route Structure and Navigation
        - 3.10.4 The Service Layer — HTTP Communication
        - 3.10.5 The HTTP Interceptor — Global Error Handling
        - 3.10.6 Reactive State with Angular Signals
        - 3.10.7 The Policy Form — Dynamic Multi-Tool Form
        - 3.10.8 The Report Detail — Type-Discriminated Display
        - 3.10.9 The Notification Service — Global Feedback
        - 3.10.10 Development and Production Configuration

4. [Conclusion, Limitations, and Future Work](04-conclusion.md)
    - 4.1 Summary of Achieved Results
        - R1–R6 requirements evaluation
        - Completed user story coverage table
    - 4.2 Architectural Contributions
    - 4.3 Limitations
        - 4.3.1 Litmus Report Handler — Scheduling Confirmation Only
        - 4.3.2 Cross-Tool Finding Correlation
        - 4.3.3 Single-Namespace Tool Lookup Convention
        - 4.3.4 CORS Wildcard Policy
        - 4.3.5 Falco Alert Counting — Observation Window
        - 4.3.6 Authentication Delegation
    - 4.4 Future Work
        - 4.4.1 Litmus ChaosResult Integration
        - 4.4.2 Cross-Tool Finding Correlation (WorkloadReport CRD)
        - 4.4.3 Scheduled Scanning
        - 4.4.4 Named Tool References
        - 4.4.5 OIDC Authentication at the API Layer
        - 4.4.6 Additional Tool Integrations
        - 4.4.7 Configurable Verdict Thresholds
    - 4.5 Closing Statement

5. References *(to be added)*
