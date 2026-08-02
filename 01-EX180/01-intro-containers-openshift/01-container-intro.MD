# Introduction to Containers

## 1. What is a Container?

A **container** is a lightweight, standalone, executable package of software that includes everything needed to run an application:

- Application code
- Runtime (e.g., language interpreter/VM like Python, JVM, Node.js)
- System libraries and dependencies
- Configuration files

Containers run as **isolated processes on the host operating system**, sharing the host's kernel but keeping their own isolated view of the filesystem, network, and processes. This isolation is achieved using Linux kernel features:

- **Namespaces** — isolate what a process can *see* (its own PID list, network interfaces, mount points, hostname, users).
- **cgroups (control groups)** — limit what a process can *use* (CPU, memory, disk I/O, network bandwidth).
- **Union/Overlay filesystems** — allow multiple layers of a filesystem to be stacked and presented as a single filesystem (this is how images are built in layers).

> **Analogy:** A container is like a shipping container in logistics — a standard box that holds goods (your app + dependencies) and can be moved/run on any ship, truck, or crane (any host) without worrying about what's inside or how the transport works internally.

> **Tool used in this course (DO180/EX180):** We use **Podman** as the container engine, not Docker. Podman provides the same `docker`-style CLI experience (in fact `podman` commands are nearly identical to `docker` commands), but with a different, more secure architecture — see [Section 8](#8-podman-vs-docker-why-podman).

---

## 2. Image vs Container (Instance)

This is one of the most important distinctions to understand.

| | **Image** | **Container** |
|---|---|---|
| Definition | A read-only **template/blueprint** built from a set of instructions (Containerfile) | A **running (or stopped) instance** created from an image |
| State | Static, immutable, stored on disk/registry | Dynamic — has a running process, can change at runtime |
| Analogy | Like a **class** in OOP, or a **.iso installer**, or a **recipe** | Like an **object/instance** of that class, or the **cooked dish** from the recipe |
| Storage | Composed of stacked read-only **layers** | Adds a thin **writable layer** on top of the image layers |
| Reusability | One image can be used to spin up **many containers** | Each container is an independent instance with its own writable layer, network, and process |

**Command relationship (using Podman):**

```bash
# Build an image from a Containerfile
podman build -t myapp:1.0 .

# Create and run a container FROM that image
podman run -d --name myapp-instance myapp:1.0

# You can run many containers from the same image
podman run -d --name myapp-instance-2 myapp:1.0
podman run -d --name myapp-instance-3 myapp:1.0

# List images vs list running containers
podman images
podman ps
```

So: **Image = blueprint (on disk)**, **Container = live instance (in memory/running as a process)**.

---

## 3. Containers vs Virtual Machines (VMs)

| Aspect | **Virtual Machine** | **Container** |
|---|---|---|
| Virtualization level | Hardware-level virtualization via a **Hypervisor** (e.g., KVM, VMware, Hyper-V) | OS-level virtualization — shares the **host kernel** |
| OS | Each VM runs its **own full guest OS** (kernel + userland) | Containers share the **host OS kernel**; only userland/libraries are packaged |
| Size | Heavy — typically **GBs** in size | Lightweight — typically **MBs** in size |
| Boot time | Slow — **minutes** (boots a full OS) | Fast — **seconds or milliseconds** (just starts a process) |
| Resource overhead | High — each VM needs allocated CPU/RAM/disk for its own OS | Low — no duplicate OS overhead, more efficient resource usage |
| Isolation | Strong isolation (separate kernel per VM) — more secure by default | Process-level isolation (shared kernel) — slightly weaker isolation boundary than VMs |
| Density | Fewer VMs per host due to overhead | Many more containers per host possible |
| Portability | Less portable — tied to hypervisor/hardware specifics | Highly portable — "build once, run anywhere" (any host with a container runtime) |
| Use case | Running multiple **different OS types** (e.g., Windows + Linux) on one host | Running many instances of an app efficiently, microservices, CI/CD |

### Visual difference in the stack

```
Virtual Machines                     Containers
------------------                   ------------------
| App A | | App B |                 | App A | | App B |
| Bin/Libs| | Bin/Libs|              | Bin/Libs| | Bin/Libs|
| Guest OS| | Guest OS|              |--------- Podman / Container Engine ---------|
------------------                   |----------------- Host OS -------------------|
|     Hypervisor     |               |-------------- Infrastructure ---------------|
|----- Host OS ------|
|--- Infrastructure --|
```

- VMs virtualize the **hardware** — each guest thinks it has its own machine.
- Containers virtualize the **operating system** — each container thinks it has its own OS, but they all share the same kernel underneath.

---

## 4. Why do we need Containers? (Motivation)

Before containers, the common problems were:

1. **"Works on my machine" problem** — an app runs fine on a developer's laptop but fails in production due to different library versions, OS, or configuration.
2. **Dependency conflicts** — App A needs Python 2.7, App B needs Python 3.10, both on the same server.
3. **Slow, heavy VMs** — spinning up a full VM just to test/run a small app wastes resources and time.
4. **Inconsistent environments** — Dev, QA, and Production environments configured slightly differently, causing bugs that "only happen in prod".

Containers solve this by **packaging the application with everything it needs**, so it behaves identically regardless of where it runs (laptop, test server, cloud, on-prem).

---

## 5. Benefits of Containers

- **Portability** — "Build once, run anywhere." Runs the same on a developer laptop, CI server, on-prem datacenter, or any cloud.
- **Consistency** — Eliminates environment drift between Dev/Test/Prod ("works on my machine" is solved).
- **Lightweight & Fast** — No guest OS overhead; containers start in seconds/milliseconds and use fewer resources than VMs.
- **Efficient resource utilization** — More containers can run per host than VMs, since they share the kernel.
- **Isolation** — Each container has its own filesystem, process space, and network namespace, preventing app A from interfering with app B.
- **Scalability** — Easy to spin up/tear down many identical containers quickly (great fit for microservices & auto-scaling).
- **Version control for infrastructure** — Containerfiles/images can be versioned, tagged, and rolled back just like source code.
- **Faster CI/CD pipelines** — Build an image once, test it, and deploy the exact same artifact to production.
- **Ecosystem & orchestration** — Works seamlessly with orchestrators like **Kubernetes/OpenShift** for automated scaling, healing, and deployment.

---

## 6. Key Jargon Explained

### Immutable vs Mutable

- **Immutable image**: Once a container image is built, it **does not change**. If you need to update the app, you build a **new image** (e.g., `myapp:1.1`) rather than modifying the existing one. This guarantees that the exact same artifact tested in QA is what runs in Production.
- **Mutable container**: A *running* container has a thin **writable layer** on top of the immutable image layers. Any changes made while it runs (new files, logs, temp data) live in this writable layer and are **lost when the container is deleted**, unless persisted via **volumes**.

> Best practice: Treat containers as **cattle, not pets** — don't manually patch a running container; instead, rebuild the image and replace the container. This is the foundation of **immutable infrastructure**.

### Layers

Images are built from a series of **read-only layers**, each corresponding to an instruction in the Containerfile (`FROM`, `RUN`, `COPY`, etc.). Layers are cached and shared across images, which saves disk space and speeds up builds.

### Container Engine / Runtime

Software responsible for creating, running, and managing containers. In this course we use **Podman** (daemonless, rootless-capable). Other examples: **CRI-O**, **containerd** (used internally by Kubernetes/OpenShift for running containers).

### Registry

A centralized storage/distribution service for container images (e.g., **registry.redhat.io**, **Quay.io**, **registry.access.redhat.com**). You `push` images to a registry and `pull` them from any other host using Podman:

```bash
podman pull registry.access.redhat.com/ubi9/ubi
podman push myapp:1.0 quay.io/myuser/myapp:1.0
```

### Containerfile (a.k.a. Dockerfile)

A text file with step-by-step instructions describing how to build an image (base image, dependencies, files to copy, commands to run, ports to expose, default startup command). Podman calls this file **Containerfile** (the name `Dockerfile` still works too, since the syntax is compatible).

### Namespaces

Linux kernel feature that gives each container its own isolated view of: process IDs (PID), network interfaces (NET), mount points (MNT), hostname (UTS), inter-process communication (IPC), and users (USER).

### cgroups (Control Groups)

Linux kernel feature that limits and accounts for resource usage (CPU, memory, disk I/O) per container, preventing one container from starving others on the same host.

### Union/Overlay Filesystem

A filesystem mechanism that stacks multiple read-only image layers plus one writable container layer, presenting them all as a single unified filesystem to the running process.

### Volumes

A mechanism to persist data **outside** the container's writable layer (e.g., on the host filesystem or a network storage) so that data survives even after the container is deleted — since container storage itself is ephemeral by default.

```bash
podman run -d -v /host/data:/container/data:Z myapp:1.0
```

### Stateless vs Stateful

- **Stateless container**: Doesn't need to retain data between restarts (e.g., a web frontend). Easy to scale/replace.
- **Stateful container**: Needs persistent data (e.g., a database). Requires volumes to avoid data loss.

### Tag

A label attached to an image to identify a specific version (e.g., `myapp:1.0`, `ubi9:latest`). Helps with version control of images.

### Rootless Containers

A Podman-specific strength: containers can be run **entirely as a normal, unprivileged user**, without any daemon running as root. This significantly reduces the security attack surface compared to a root-owned daemon managing all containers.

### Orchestration

Automated management of many containers across multiple hosts — scheduling, scaling, self-healing, load balancing (e.g., **Kubernetes**, **OpenShift**).

---

## 7. Quick Summary

- A **container** = an isolated, running process that packages an app + its dependencies, sharing the host OS kernel.
- An **image** = the immutable, read-only template used to create containers.
- A **container** = a running/stopped **instance** of an image (with an added writable layer).
- Compared to **VMs**, containers are lighter, faster, and more portable, but share the kernel (slightly less isolation) instead of virtualizing entire hardware/OS.
- Containers exist to solve **environment inconsistency**, **dependency conflicts**, and **inefficient resource usage**, enabling fast, portable, and scalable application delivery — which is exactly why platforms like **OpenShift** (built on Kubernetes) exist to orchestrate them at scale.

---

## 8. Podman vs Docker (Why Podman?)

Since this content targets the **DO180/EX180** (and related **DO188/EX188**) exams, it's important to know *why* Red Hat's tooling uses **Podman** instead of Docker:

| Aspect | **Docker** | **Podman** |
|---|---|---|
| Architecture | Client-server: a long-running **background daemon** (`dockerd`) runs as **root** and manages all containers | **Daemonless**: each `podman` command runs as a standalone process, no persistent background service required |
| Root privileges | Daemon typically runs as **root**, so anything talking to it effectively has root-level access | Supports **rootless containers** — a normal user can build/run containers without root, improving security |
| CLI compatibility | `docker <command>` | `podman <command>` — CLI syntax is intentionally almost identical to Docker's, so most `docker` commands work by just typing `podman` instead |
| systemd integration | Not designed around systemd | Integrates well with **systemd** — can generate unit files (`podman generate systemd`, or Quadlet) to manage containers as services |
| Pods | No native concept of a "pod" | Supports **pods** — a group of one or more containers sharing network/storage namespaces, same model used by **Kubernetes/OpenShift** |
| Single point of failure | If the daemon crashes, **all** containers/management is affected | No central daemon, so no single point of failure for container management |

> **Key takeaway for the exam:** Podman is a drop-in, daemonless, rootless-capable replacement for Docker's CLI. Because Podman's pod concept mirrors Kubernetes Pods, it's a natural fit for learning concepts that carry over directly to **OpenShift**.
