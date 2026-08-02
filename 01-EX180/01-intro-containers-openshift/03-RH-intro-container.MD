# Introduction to Containers (Red Hat Documentation Notes)

> Condensed and cleaned-up notes based on Red Hat's official course documentation.

## 1. What is a Container?

- A container is an **encapsulated process** that includes the runtime dependencies required for the program to run.
- **Application-specific libraries** are packaged inside the container, independent of the host OS libraries.
- Libraries/functions **not specific** to the containerized app are provided by the host **OS and kernel** — this keeps containers compact, and able to start/stop quickly.
- A container engine creates a **union file system** by merging container image layers.
  - Image layers are **immutable**.
  - The engine adds a **writable layer** on top for runtime file modifications.
  - Containers are **ephemeral** by default — the writable layer is removed when the container is removed.
- Containers rely on Linux kernel features:
  - **cgroups (Control Groups)** — resource management (CPU time, memory allocation, etc.).
  - **Namespaces** — isolate processes within containers from each other and from the host system.
- The environment **inside** a container is always Linux-based, regardless of the host OS.
  - On **non-Linux** hosts, these Linux-specific features are virtualized by the container engine implementation.
- **Origins & standards:**
  - Containerization originated from technologies like **chroot** (partial/full environment isolation).
  - It evolved into the **Open Container Initiative (OCI)** — a governance organization defining standards for creating and running containers.
  - Most container engines conform to OCI specs, so developers can confidently build deployable artifacts as **OCI containers**.

## 2. Images versus Instances

- **Container image** — contains effectively **immutable** data defining an application and its libraries.
- **Container instance** — the **running version** of an image, incorporating runtime resources (networking, storage, etc.).
- A single container image can be used to create **many distinct container instances**, potentially across **multiple hosts**.
- The application inside a container is **independent of the host environment**.

## 3. Deployment at Scale

- Both **containers** and **VMs** can work well at various scales.
- Containers require **considerably fewer resources** than VMs, giving performance/resource benefits at larger scale.
- Common large-scale pattern: **run containers inside VMs** — combining the strengths of both technologies.

## 4. Development for Containers

Containerization simplifies development via easier testing/deployment, and by providing stability, security, and flexibility.

### 4.1 Testing and Workflows

- Containers make it easy to **scale**: write and test software locally, then deploy the same application to a cloud server or dedicated cluster with **few or no changes**.
- Especially useful for **microservices** — small, ephemeral containers designed to spin up/down as needed.
- Containers integrate naturally with **CI/CD (Continuous Integration/Continuous Development)** pipelines.
  - **RHOCP (Red Hat OpenShift Container Platform)** offers built-in integration features for CI/CD pipelines and workflows.

### 4.2 Stability

- Container images are a **stable deployment target**.
- Applications often require specific library versions, which can cause dependency issues or specific OS requirements.
- Since libraries are bundled inside the image, developers avoid dependency issues at deployment.
- Bundling libraries removes variability between **testing and production** environments.
  - Example: a container with a specific Python version guarantees the same Python version everywhere it's deployed.

### 4.3 Multi-container Applications

- A **multi-container application** is distributed across several containers.
- Containers can be run:
  - From the **same image** — for **High Availability (HA)** replicas.
  - From **several different images** — e.g., a database container running separately from a web API container.
- Applications can rely on container management software to provide HA replicas and multi-container orchestration, using tools/specs such as:
  - **Podman Pods**
  - The **compose-spec** specification

## 5. Key Terms Recap

| Term | Meaning |
|---|---|
| **Encapsulated process** | A container is fundamentally an isolated process, not a separate machine |
| **Union file system** | Merges multiple image layers into a single filesystem view |
| **Immutable layer** | Read-only image layers that never change |
| **Writable layer** | Per-container layer added at runtime for changes; removed with the container |
| **Ephemeral** | Container state/data is temporary by default, gone once the container is removed |
| **cgroups** | Kernel feature for limiting/managing resources (CPU, memory, etc.) |
| **Namespaces** | Kernel feature for isolating a container's view of the system |
| **chroot** | Early ancestor technology for filesystem-level isolation |
| **OCI** | Open Container Initiative — standards body for container images/runtimes |
| **Container image** | Immutable blueprint defining an app + libraries |
| **Container instance** | A running instance created from a container image |
| **Microservices** | Small, independent, ephemeral containers designed for scaling |
| **CI/CD** | Continuous Integration/Continuous Development pipelines, well-supported by RHOCP |
| **HA (High Availability) replicas** | Multiple instances of the same image running for redundancy |
| **Multi-container application** | An app composed of several cooperating containers |
| **Podman Pods / compose-spec** | Tools/specs for managing multi-container applications |
