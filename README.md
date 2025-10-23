# Eirenyx Arcitecture

## Eirenyx – Unified policy operator for Kubernetes

**Eirenyx** -> Kubernetes operator built in Go that unifies three tools — **Trivy**, **Kyverno**, and **Litmus** — under a single, declarative interface. It aims to combine **security scanning**, **policy enforcement**, and **resilience testing** to improve **DevOps workflows** and **cluster code quality**.

Inspired by **Eirene**, the Greek goddess of peace and order.

### Features

- **Policy abstraction** over Kyverno, Trivy, and Litmus
- Unified `Policy` CRD for configuring multiple enforcement types
- Operator reconciles `Policy` specs into real actions in the cluster
- Generates structured **cluster reports** with policy violations, vulnerabilities, and test results

### Tools Used

#### Kyverno – Policy Enforcement Engine

- Kubernetes-native policy engine
- Allows validation, mutation, generation of Kubernetes resources
- Declarative, YAML-based rules
- Perfect for enforcing security best practices and cluster governance

**Example Use**: Block images not coming from a trusted registry, or enforce labels on Pods.

#### Trivy – Vulnerability Scanner

- Accurate open-source vulnerability scanner for containers and infrastructure
- Supports scanning Kubernetes workloads (Pods, Deployments, etc.)
- Provides CVE reports, SBOMs, and misconfiguration checks

**Example Use**: Detect known CVEs in container images used in Deployments.

#### Litmus – Chaos Engineering Framework

- Kubernetes-native chaos testing framework
- Simulates failures such as pod-kill, network-delay, CPU/memory hogs
- Helps test workload resilience under failure conditions

**Example Use**: Test if an application gracefully recovers when its pods are randomly deleted.

### Architecture Overview

![Architecture Diagram](assets/arcitecture.drawio.png)
