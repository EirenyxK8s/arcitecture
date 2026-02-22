# Eirenyx Architecture

## Eirenyx – Unified Policy Operator for Kubernetes

**Eirenyx** is a Kubernetes Operator written in Go that unifies multiple security and resilience tools under a single declarative interface.

It integrates:

* **Trivy** – vulnerability and configuration scanning
* **Falco** – runtime behavioral security detection
* **Litmus** – chaos engineering and resilience validation

The operator abstracts these heterogeneous systems into a unified `Policy` Custom Resource Definition (CRD), enabling centralized security orchestration, automated reconciliation, and structured reporting within a Kubernetes cluster.

Inspired by **Eirene**, the Greek goddess of peace and order, Eirenyx aims to introduce clarity, control, and stability into cloud-native environments.

---

## Core Features

* Unified `Policy` CRD for multi-engine security control
* Engine abstraction layer for Falco, Trivy, and Litmus
* Namespace-aware policy targeting
* Declarative reconciliation into engine-specific artifacts
* Structured `PolicyReport` generation
* Cluster-wide consolidated security and resilience visibility
* Lifecycle management with cleanup and finalizers

---

## Integrated Tools

### Falco – Runtime Security & Behavioral Detection

Falco is a Kubernetes-native runtime security engine that monitors system calls and Kubernetes audit events in real time.

**Key Characteristics**

* Detects anomalous or malicious behavior
* Uses declarative rule definitions
* Focused on detection and response during execution
* Observes container and node activity continuously

Within Eirenyx, Falco serves as the **runtime visibility and intrusion detection layer**, providing insight into what is actually happening inside the cluster.

**Example Use Case**
Detect a container spawning an unexpected shell or attempting privilege escalation.

---

### Trivy – Vulnerability & Configuration Scanner

Trivy is an open-source scanner for container images, Kubernetes workloads, and infrastructure configurations.

**Key Characteristics**

* Detects known CVEs in container images
* Performs misconfiguration checks
* Generates Software Bill of Materials (SBOM)
* Supports Kubernetes resource scanning

Within Eirenyx, Trivy provides **pre-runtime security validation**, identifying vulnerabilities before workloads become active.

**Example Use Case**
Identify critical CVEs in container images used in Deployments.

---

### Litmus – Chaos Engineering Framework

Litmus is a Kubernetes-native chaos engineering framework designed to validate workload resilience under failure conditions.

**Key Characteristics**

* Simulates pod failures, network delays, and resource stress
* Tests application recovery behavior
* Evaluates system fault tolerance

Within Eirenyx, Litmus provides **resilience verification**, ensuring workloads behave predictably under adverse conditions.

**Example Use Case**
Verify that an application automatically recovers after random pod deletion.

---

## Security Lifecycle Alignment

Eirenyx integrates tools that operate across distinct phases of the Kubernetes security lifecycle:

| Phase       | Tool   | Purpose                                    |
| ----------- | ------ | ------------------------------------------ |
| Pre-runtime | Trivy  | Vulnerability and configuration scanning   |
| Runtime     | Falco  | Behavioral monitoring and threat detection |
| Resilience  | Litmus | Fault injection and recovery validation    |

This separation ensures complementary responsibilities without overlap.

---

## Architectural Rationale

The integration of Falco instead of admission-focused policy engines shifts Eirenyx toward runtime observability and active threat detection.

* Admission controllers validate what *may* be deployed.
* Runtime engines observe what *actually occurs* during execution.

For a system focused on cluster stability, security visibility, and automated response, runtime detection provides stronger alignment with intrusion detection principles and operational security monitoring.

---

## Architecture Overview

![Architecture Diagram](assets/arcitecture.drawio.png)
