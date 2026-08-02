# Container Core Concepts — How a Container Actually Works

This note goes one level deeper than the intro doc. It explains **what happens under the hood** when you run a container: isolation (namespaces), resource limits (cgroups), networking, storage, and the different **container runtimes** (runC, crun, CRI-O, Podman, Docker) and how they relate to the **OCI** standard.

---

## 1. The Big Picture — What "running a container" really means

A container is **NOT a mini virtual machine**. It is just a **normal Linux process** that the kernel has restricted using a few features:

| Kernel feature | What it does | Simple analogy |
|---|---|---|
| **Namespaces** | Controls what the process can **see** | Blinders/curtains — process thinks it's alone |
| **cgroups** | Controls what the process can **use** (CPU/RAM/etc.) | A budget/ration card |
| **chroot / pivot_root + OverlayFS** | Controls what **filesystem** the process sees | Giving it its own private room instead of the whole house |
| **Capabilities / SELinux / seccomp** | Controls what the process is **allowed to do** (syscalls, root actions) | Rule book of do's and don'ts |

So: **Container = a regular Linux process + namespaces (isolation) + cgroups (limits) + a private root filesystem (image layers)**. There is no separate kernel, no boot process — that's why containers start almost instantly.

---

## 2. Namespaces — "What can I see?"

Namespaces give a process its own **isolated view** of a particular kind of system resource. Each container typically gets its own set of these:

| Namespace | Isolates | In simple English |
|---|---|---|
| **PID** | Process IDs | Inside the container, your app looks like PID 1, even though on the host it might be PID 4821. Container can't see host's other processes. |
| **NET** | Network stack (interfaces, IP, ports, routes) | Container gets its own virtual network interface + IP address, separate from the host. |
| **MNT** | Mount points / filesystem view | Container sees only its own root filesystem, not the host's real `/`. |
| **UTS** | Hostname & domain name | Container can have its own hostname (e.g., `my-app`) different from the host. |
| **IPC** | Inter-process communication (shared memory, semaphores) | Container's processes can't accidentally talk to host processes via shared memory. |
| **USER** | User & group IDs | A process can be "root" (UID 0) *inside* the container, but be mapped to an unprivileged, non-root UID on the *host* — this is the trick behind **rootless containers**. |

> **Key exam point:** Namespaces are why a container "thinks" it has its own machine — its own PID 1, its own hostname, its own network — when in reality all containers share the same host kernel.

---

## 3. cgroups (Control Groups) — "How much can I use?"

While namespaces control *visibility*, **cgroups** control and **limit resource usage** so that one container cannot starve others or the host.

Cgroups can limit/measure:

- **CPU** — e.g., limit a container to 0.5 CPU core.
- **Memory** — e.g., cap a container at 512 MB RAM (and kill it — OOM kill — if it exceeds this).
- **Block I/O** — e.g., limit disk read/write throughput.
- **PIDs** — limit the number of processes/threads a container can spawn (protects against fork-bombs).
- **Network bandwidth** (via additional tooling).

Example with Podman:

```bash
# Limit container to half a CPU core and 512MB RAM
podman run -d --cpus=0.5 --memory=512m myapp:1.0
```

Behind the scenes, Podman writes these limits into `/sys/fs/cgroup/...` files, and the Linux kernel scheduler enforces them.

> **Namespaces vs cgroups — the one-line difference:**
> - **Namespaces** = "What can I *see*?" (isolation)
> - **cgroups** = "How much can I *use*?" (limits)

---

## 4. Container Networking — How containers talk to the world

Every container gets its own **network namespace**, meaning its own virtual network interface (`eth0`), IP address, and routing table — separate from the host and from other containers, by default.

### How connectivity is provided

1. **Bridge networking (default)**
   - The container engine (e.g., Podman) creates a virtual bridge (like a virtual switch) on the host, e.g., `podman0` / `cni-podman0`.
   - Each container gets a **virtual ethernet pair (veth)**: one end lives inside the container's network namespace (`eth0`), the other end is plugged into the bridge on the host.
   - Containers on the same bridge can reach each other via their internal IPs; the bridge routes traffic out to the host's real network via NAT (Network Address Translation).

2. **Port mapping / publishing**
   - Since the container has its own private IP, the outside world can't reach it directly.
   - You **publish/map a host port to a container port** so external traffic can reach the container:

   ```bash
   # Map host port 8080 -> container port 80
   podman run -d -p 8080:80 nginx
   ```

   - Now, a request to `http://<host-ip>:8080` gets forwarded (via iptables/nftables NAT rules) into the container's port 80.

3. **DNS / name resolution between containers**
   - When containers are on the same user-defined network (or the same **pod**), the engine provides a small embedded DNS service so containers can reach each other **by container name** instead of IP (e.g., `curl http://db:5432` instead of a hard-coded IP).

4. **Pods share a network namespace**
   - In Podman (and Kubernetes/OpenShift), a **pod** groups multiple containers to **share the same network namespace**. That means containers in the same pod share the same IP address and can talk to each other over `localhost`, exactly like Kubernetes Pods behave.

   ```bash
   podman pod create --name mypod -p 8080:80
   podman run -d --pod mypod nginx
   podman run -d --pod mypod my-sidecar
   ```

### Networking modes (quick reference)

| Mode | Description |
|---|---|
| **bridge** (default) | Private internal network + NAT out to host network |
| **host** | Container shares the host's network namespace directly (no isolation, no port mapping needed) |
| **none** | No networking at all — fully isolated |
| **pod-shared** | Container joins another container/pod's existing network namespace |

---

## 5. Container Storage — How the filesystem works

### Image layers (read-only)

A container image is built from multiple **read-only layers**, stacked on top of each other, each layer representing one instruction in the Containerfile (`FROM`, `RUN`, `COPY`, ...). Layers are content-addressed (identified by hash) and **cached/shared** across images — if two images share a base layer, it's stored only once on disk.

### The writable container layer

When you run a container from an image, the engine adds one more layer on top: a thin, **writable layer**, unique to that specific container.

- Any file **the container writes, modifies, or deletes** happens in this writable layer only.
- The underlying image layers are **never changed** (this is what makes images immutable).
- This mechanism is called **Copy-on-Write (CoW)**: if a container "modifies" a file that lives in a read-only layer below, the storage driver first copies that file up into the writable layer, then modifies the copy.

### OverlayFS — how layers are combined

The most common storage driver is **OverlayFS**, a Union filesystem that merges multiple directories (layers) into a single unified view:

```
/  (what the container process sees — merged view)
│
├── upperdir  → the writable container layer (read-write)
├── lowerdir  → image layers, stacked, read-only (can be many)
└── workdir   → internal bookkeeping used by overlay driver
```

The kernel presents `upperdir + lowerdir(s)` together as one merged filesystem — reads fall through to the lowest layer that has the file; writes always go to `upperdir`.

### Ephemeral by default — why we need Volumes

- The writable layer is **deleted along with the container** (`podman rm`). Any data inside it is lost.
- For data that must **survive** container restarts/removal (databases, uploaded files, logs), you mount a **volume**: a directory from the host (or a managed storage location) directly into the container, bypassing the layered/CoW filesystem for that path.

```bash
# Bind-mount a host directory into the container
podman run -d -v /host/data:/var/lib/mysql:Z mysql

# Or a named/managed volume
podman volume create dbdata
podman run -d -v dbdata:/var/lib/mysql:Z mysql
```

> The `:Z` suffix tells Podman to relabel the content for SELinux so the container is allowed to access it — very relevant on RHEL, since SELinux is enabled by default.

---

## 6. Container Runtimes — Low-level vs High-level, and the OCI Standard

This is a frequently confused topic — there isn't just "one" container runtime; there are **layers of runtimes**, standardized by the **OCI (Open Container Initiative)**.

### What is "OCI"?

The **Open Container Initiative (OCI)** is an open governance body that defines **standard specifications** so that container images and runtimes from different vendors are all compatible with each other. It defines three main specs:

1. **OCI Image Spec** — the standard format for a container **image** (layers, manifest, config) — so an image built by Podman can be pulled and run by Docker, CRI-O, containerd, etc.
2. **OCI Runtime Spec** — the standard for how a **low-level runtime** should actually create and run a container (given an unpacked root filesystem + a config file describing namespaces/cgroups/etc.).
3. **OCI Distribution Spec** — the standard API for **registries** (how images are pushed/pulled, e.g., the Docker Registry HTTP API v2 that most registries implement).

> **In short: OCI = the rulebook that makes "build with tool X, run with tool Y" possible.**

### Runtime = the software that actually starts/stops a container

A **container runtime** is the software layer responsible for the actual mechanics of creating a container: setting up namespaces, applying cgroup limits, mounting the root filesystem, and then executing the process inside — following the OCI Runtime Spec.

There are two "levels" of runtime:

#### A) Low-level (OCI) runtimes — do the actual kernel work

These are small, simple tools that take a prepared root filesystem + a `config.json` and directly make the Linux kernel syscalls (namespaces, cgroups, `chroot`/`pivot_root`, capabilities) to spin the container process up.

| Runtime | Notes |
|---|---|
| **runc** | The original, most widely used reference implementation of the OCI Runtime Spec. Written in Go, originally extracted from Docker. |
| **crun** | A faster, lighter-weight OCI runtime written in C (used by default in RHEL/Podman/CRI-O). Lower memory footprint than runc, supports cgroups v2 well. |
| gVisor / Kata Containers | Alternative low-level runtimes that add extra sandboxing/VM-level isolation for higher security — not covered on the exam, but good to know they exist. |

You almost never call these directly — they're invoked internally by the higher-level tools below.

#### B) High-level (container engine) runtimes/tools — what you actually interact with

These provide the user-facing CLI/API: pulling images from registries, managing storage, building images, managing networking, and then **delegating to a low-level runtime** (like `runc`/`crun`) to actually launch the container process.

| Tool | Type | Notes |
|---|---|---|
| **Docker** | Standalone engine, daemon-based | `dockerd` daemon runs as root; internally uses `containerd` → `runc` to actually launch containers. Great for local single-host use, but not used by RHEL/OpenShift by default. |
| **Podman** | Standalone engine, **daemonless** | Directly manages images/containers/pods/networking itself, then calls `crun` (or `runc`) to launch the container. Rootless-capable. This is the CLI tool used throughout DO180/DO188/EX180/EX188. |
| **CRI-O** | Kubernetes-focused engine | A lightweight container engine built **specifically to implement Kubernetes' CRI (Container Runtime Interface)**. It has no CLI of its own for end users — **Kubernetes/OpenShift's kubelet talks to CRI-O**, and CRI-O then calls `runc`/`crun` under the hood. This is the runtime **OpenShift** uses on its nodes. |
| **containerd** | Engine/daemon | Originally extracted from Docker; used internally by Docker and also directly by some Kubernetes distributions as their CRI implementation (alternative to CRI-O). |

### Putting it all together — the stack

```
 You (CLI / kubelet)
        │
        ▼
 High-level engine:  Podman  /  Docker  /  CRI-O  /  containerd
        │        (pulls images, manages storage/network, builds images)
        ▼
 Low-level OCI runtime:  runc  /  crun
        │        (does the actual namespaces + cgroups + exec syscalls)
        ▼
      Linux Kernel
```

### Where each fits in this course

- **Podman** — what *you* type on the command line to build/run/manage containers locally (DO180/EX180, DO188/EX188).
- **CRI-O** — what actually runs your containers once they're deployed onto an **OpenShift** cluster; OpenShift's `kubelet` talks to CRI-O via the CRI interface instead of talking to Podman or Docker.
- **runc / crun** — the low-level engine underneath both Podman and CRI-O that does the real kernel-level work. Podman and CRI-O both default to **crun** on RHEL.
- **Docker** — not used in this course/on RHEL/OpenShift, but good to understand since it popularized containers and shares the same OCI-compatible image format, so images are interchangeable.

> **Exam-relevant takeaway:** Because everything (Podman, Docker, CRI-O, containerd) agrees on the **OCI Image Spec**, an image you build once with `podman build` can be pushed to a registry and later pulled and run unmodified by CRI-O inside an OpenShift cluster — that's the whole point of the "build once, run anywhere" promise.

---

## 7. Quick Recap

- A container is just an isolated **Linux process**: **namespaces** control what it can see, **cgroups** control what it can use.
- **Networking**: each container gets its own network namespace + virtual interface, connected via a bridge, with **port mapping** used to expose it externally; **pods** let multiple containers share one network namespace.
- **Storage**: images are stacked **read-only layers**; a container adds one **writable layer** on top (Copy-on-Write via **OverlayFS**); use **volumes** for data that must persist beyond the container's life.
- **OCI** is the standard that makes images and runtimes from different vendors interoperable (Image Spec, Runtime Spec, Distribution Spec).
- **Low-level runtimes** (`runc`, `crun`) do the actual kernel work; **high-level engines** (`Podman`, `Docker`, `CRI-O`, `containerd`) provide the user/Kubernetes-facing tooling and call down into a low-level runtime.
- In this course: you use **Podman** on the command line, and **OpenShift** uses **CRI-O** (with **crun**) under the hood to run the exact same OCI images on the cluster.
