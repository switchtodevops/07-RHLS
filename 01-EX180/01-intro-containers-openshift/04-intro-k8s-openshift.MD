# Introduction to Kubernetes and OpenShift (Red Hat Documentation Notes)

> Condensed and cleaned-up notes based on Red Hat's official course documentation.

## 1. Kubernetes Overview

- Kubernetes is an **orchestration service** that simplifies the deployment, management, and scaling of containerized applications.
- It manages complex pools of resources: **CPU, RAM, storage, and networking**.
- Provides **high uptime and fault tolerance**, removing the developer's concern about how their applications use resources.
- **Pod** — the smallest manageable unit in Kubernetes:
  - Represents a single application.
  - Consists of **one or more containers**.
  - Includes storage resources and an **IP address**.

## 2. Kubernetes Features

Kubernetes clusters provide a modern container platform addressing the challenges of running applications at scale, delivering robust infrastructure and ease of management, regardless of deployment size.

| Feature | Description |
|---|---|
| **Service discovery & load balancing** | Assigns a single **DNS entry** to each set of containers, so requesting services only need to know the target's DNS name (not its changing IP/location). Kubernetes then **load-balances** requests across the pool of containers providing that service (e.g., evenly splitting requests across available NGINX pods). |
| **Horizontal scaling** | Applications can scale up/down **manually** or **automatically** (via a configuration set), using the Kubernetes CLI or web UI. |
| **Self-healing** | Uses user-defined **health checks** to monitor pods, restarting/rescheduling them automatically on failure. |
| **Automated rollout** | Gradually rolls out updates to an application's containers while checking their status; automatically **rolls back** to the previous version if something goes wrong. |
| **Secrets and configuration management** | Manages configuration settings and secrets (usernames, passwords, service endpoints, or any private setting) **without rebuilding containers**. |
| **Operators** | Packaged Kubernetes applications that bring the **application's lifecycle knowledge** into the cluster. Operators use the Kubernetes API to update cluster state and react to changes in application state. |

## 3. Red Hat OpenShift Container Platform (RHOCP) Overview

- **RHOCP** is a set of modular components and services built **on top of Kubernetes** container infrastructure.
- Adds production-platform capabilities such as:
  - Remote management
  - Multitenancy
  - Increased security
  - Monitoring and auditing
  - Application lifecycle management
  - Self-service interfaces for developers

## 4. RHOCP Features (added on top of Kubernetes)

| Feature | Description |
|---|---|
| **Developer workflow** | Integrates a built-in **container registry**, **CI/CD (Continuous Integration/Continuous Delivery)** pipelines, and **Source-to-Image (S2I)** — a tool that builds container images directly from source code repositories. |
| **Routes** | Easily **exposes services** to the outside world. |
| **Metrics and logging** | Built-in, self-analyzing **metrics service** and **aggregated logging**. |
| **Unified UI** | Integrated UI and unified tools to manage all platform capabilities. |

## 5. Lifecycle of an Application in RHOCP

1. Define a **pod** and the containers it's composed of (containing the application).
2. The pod is **assigned to a healthy node**.
3. The pod **runs** until its containers exit.
4. The pod and its containers are **removed** from the node.

## 6. Key Terms Recap

| Term | Meaning |
|---|---|
| **Kubernetes** | Orchestration platform for deploying, managing, and scaling containerized applications |
| **Pod** | Smallest deployable unit in Kubernetes; one or more containers + storage + a shared IP |
| **Service discovery / load balancing** | DNS-based routing and traffic distribution across container replicas |
| **Horizontal scaling** | Increasing/decreasing the number of running instances (pods) |
| **Self-healing** | Automatic restart/reschedule of failed pods based on health checks |
| **Automated rollout / rollback** | Controlled, gradual updates with automatic revert on failure |
| **Secrets** | Securely managed sensitive configuration (passwords, tokens, endpoints) |
| **Operators** | Kubernetes-native apps that encode operational/lifecycle knowledge and use the Kubernetes API |
| **RHOCP** | Red Hat OpenShift Container Platform — enterprise platform built on Kubernetes |
| **S2I (Source-to-Image)** | Builds a container image directly from an application's source repository |
| **Routes** | OpenShift resource that exposes a service externally |
| **Node** | A worker machine (VM or physical) in the cluster where pods are scheduled to run |
