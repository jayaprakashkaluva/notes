# Docker Deep Dive: From "Works on My Machine" to Kernel Internals

This document explains what Docker actually is, the problems it solves, how those
problems were solved before Docker existed, and how Docker works internally — down
to the Linux kernel primitives it is built on. It assumes you understand how a
computer works (processes, memory, filesystems, syscalls) but nothing about
containers.

---

## Table of Contents

1. [The Problem Docker Solves](#1-the-problem-docker-solves)
2. [How These Problems Were Solved Before Docker](#2-how-these-problems-were-solved-before-docker)
3. [The Key Insight: Containers Are Just Processes](#3-the-key-insight-containers-are-just-processes)
4. [The Kernel Primitives: Namespaces, cgroups, Capabilities](#4-the-kernel-primitives)
5. [Images and Layered Filesystems](#5-images-and-layered-filesystems)
6. [Docker's Architecture: Client, Daemon, containerd, runc](#6-dockers-architecture)
7. [The Container Lifecycle, Step by Step](#7-the-container-lifecycle-step-by-step)
8. [Dockerfiles and the Build System (BuildKit)](#8-dockerfiles-and-the-build-system)
9. [Networking](#9-networking)
10. [Storage: Volumes, Bind Mounts, tmpfs](#10-storage)
11. [Registries and Image Distribution](#11-registries-and-image-distribution)
12. [Security Model](#12-security-model)
13. [Docker Compose and Multi-Container Apps](#13-docker-compose)
14. [What Docker Is Not: VMs, Kubernetes, and the OCI Ecosystem](#14-what-docker-is-not)
15. [Expert-Level Details, Gotchas, and Best Practices](#15-expert-level-details-gotchas-and-best-practices)
16. [Command Reference](#16-command-reference)

---

## 1. The Problem Docker Solves

Software is never just your code. A running application is:

```
your code
  + a language runtime (Python 3.11, JVM 17, Node 20, glibc 2.38...)
  + libraries (versions matter: numpy 1.x vs 2.x, openssl 1.1 vs 3.0)
  + OS packages (imagemagick, ffmpeg, libpq...)
  + configuration files, environment variables
  + assumptions about file paths, users, locales, timezones
```

Every one of those is a dependency on the *environment*, and environments drift.
This causes a family of related problems:

### Problem 1: "Works on my machine"
Your laptop has Python 3.11 and libssl 3.0. Production has Python 3.9 and
libssl 1.1. The code works for you and crashes in prod. The bug isn't in the
code — it's in the *difference between environments*, which is invisible in
code review and often invisible until runtime.

### Problem 2: Dependency conflicts on shared machines
App A needs Java 8. App B needs Java 17. App C needs a system library that
conflicts with App A's. On one machine, with one shared filesystem and one
package manager, these fight each other. The historical answer was "one app
per server," which leads to Problem 3.

### Problem 3: Server utilization and cost
If you isolate apps by giving each its own physical server, most servers sit
at 5–15% CPU utilization. Enormous waste. Virtual machines improved this
(see §2), but each VM carries a full OS: gigabytes of disk, hundreds of MB of
RAM overhead, and minutes of boot time.

### Problem 4: Deployment is a snowflake ritual
Before containers, deploying meant: SSH in, install packages, edit configs,
copy files, restart services — either by hand or with scripts that had to
handle every possible starting state of the machine. Servers became
"snowflakes": each one subtly unique, impossible to reproduce, terrifying to
touch.

### Problem 5: Dev/prod parity and onboarding
A new developer spends day one (or week one) installing the right versions of
everything. CI runs on a machine configured differently from both. Staging is
"mostly like" prod. Every difference is a place bugs hide.

### Docker's answer, in one sentence

> **Package the application *together with its entire userspace environment*
> into a single, immutable, versioned artifact (an image), and run it as an
> isolated process (a container) that behaves identically on any Linux
> machine.**

The image is built once, tested once, and the *exact same bytes* run in dev,
CI, staging, and prod. The unit of deployment stops being "code + a hopefully
correct machine" and becomes "a self-contained image." That's the whole idea.
Everything else in this document is mechanism.

---

## 2. How These Problems Were Solved Before Docker

Understanding the pre-Docker world explains *why* Docker looks the way it does.
Each earlier solution fixed something and left a gap Docker later filled.

### 2.1 One app per physical server (1990s)
The crudest isolation: buy another server. Perfect isolation, absurd cost,
weeks of lead time to provision hardware, ~10% utilization. Deployment was
manual and undocumented ("ask Dave, he set it up").

### 2.2 Virtual machines (2000s: VMware, Xen, KVM)
A **hypervisor** slices one physical machine into many virtual ones, each
running a *full operating system* on virtualized hardware. This solved
utilization (many VMs per host) and isolation (a kernel panic in one VM
doesn't touch others), and enabled the cloud — EC2 is VMs.

What VMs did *not* solve:
- **Weight.** Every VM duplicates an entire OS: gigabytes on disk, a full
  kernel, init system, and system daemons in memory. Boot takes minutes.
- **The environment problem inside the VM.** A VM is just another machine
  that drifts. You still had to install and configure everything within it.
- **No standard artifact.** VM images (multi-GB disk snapshots) were too
  heavy and too opaque to be the unit of software delivery. You didn't
  `git push` a VM image per commit.

### 2.3 Configuration management (mid-2000s: CFEngine, Puppet, Chef, Ansible)
Instead of configuring servers by hand, describe the desired state in code
("package nginx installed, version X; file /etc/nginx.conf with contents Y")
and let a tool *converge* the machine toward it. This gave reproducibility on
paper and made fleets manageable.

The gaps:
- **Convergence, not immutability.** The tool mutates a live machine from
  whatever state it's currently in. If manual changes, partial runs, or
  version skew put the machine in an unexpected state, convergence could
  fail or half-succeed. Drift was reduced, not eliminated.
- **Time.** Bringing a server to desired state meant downloading and
  installing packages at deploy time — slow, and dependent on external
  repositories being up and unchanged.
- **Still shared.** Apps on the same machine still shared one filesystem and
  one set of system libraries; conflicts remained.

### 2.4 Golden images / immutable infrastructure (early 2010s)
Bake a full VM image (e.g., an AMI) with everything installed; deploy by
replacing machines rather than mutating them. This is philosophically the
closest ancestor to Docker — immutable, versioned artifacts — but the
artifact was a multi-gigabyte disk image that took many minutes to build and
boot. Too heavy to build per commit, per app, per developer.

### 2.5 Language-level isolation (virtualenv, rvm, nvm, fat JARs)
Python virtualenvs, Ruby's rvm/bundler, Node's nvm/node_modules, Java's
"fat JAR" all isolate *language* dependencies. Useful, but they stop at the
language boundary: they don't isolate the interpreter's own dependencies,
system libraries (glibc, libssl, libxml), OS packages, or anything about the
machine. A fat JAR still needs the right JVM; a virtualenv still needs the
right Python and the right C libraries under your wheels.

### 2.6 OS-level isolation before Docker — the direct ancestors
The kernel technology Docker uses **predates Docker**. Docker invented almost
no isolation primitives; it packaged existing ones.

| Year | Technology | What it added |
|------|-----------|---------------|
| 1979 | `chroot` (Unix V7) | Change a process's apparent filesystem root. Isolation of *view of the filesystem* only — trivially escapable, no resource limits. |
| 2000 | FreeBSD **jails** | chroot hardened into real isolation: own hostname, IP, users, root-escape protections. |
| 2004 | Solaris **Zones** | Full OS-level virtualization on Solaris: mature, production-grade, with resource controls. Arguably the best pre-Linux container tech — but tied to Solaris. |
| 2002–2013 | Linux **namespaces** | Kernel feature: give processes private views of global resources (mount points, PIDs, network, hostname, users). Added piecemeal over a decade. |
| 2007 | Linux **cgroups** (from Google) | Kernel feature: meter and limit resource usage (CPU, memory, IO) for groups of processes. Google had been running "containers" internally (Borg) on this for years. |
| 2008 | **LXC** (LinuX Containers) | First userspace tool combining namespaces + cgroups into practical "system containers" — like lightweight VMs you managed by hand. |
| 2011 | Heroku / CloudFoundry | PaaS platforms running customer code in LXC-style containers internally — proving the model at scale, but as a closed platform, not a tool. |

So by 2012 the kernel could already do everything containers need, and LXC
exposed it. Why did nobody care until Docker?

### 2.7 What Docker actually contributed (2013)
Docker (originally an internal tool at dotCloud, a PaaS) added the missing
**developer experience and distribution layer** on top of LXC (later its own
runtime):

1. **The image**: a layered, content-addressed, portable snapshot of a
   filesystem — small enough (via layer sharing) to build per-commit and
   push/pull like code.
2. **The Dockerfile**: a reproducible, version-controllable recipe to build
   an image. Environment-as-code that actually produces an artifact.
3. **The registry** (Docker Hub): `docker push` / `docker pull` — a
   package-manager experience for whole environments. Distribution is what
   made images *useful*.
4. **App-centric ergonomics**: `docker run redis` and you have Redis running
   in seconds, on any machine, with nothing installed but Docker. LXC gave
   you a container; Docker gave you *someone else's software, running,
   in one command*.

The one-line history: **the kernel provided isolation (namespaces + cgroups),
Google proved the model (Borg), LXC made it accessible to experts, and Docker
made it usable by everyone and — crucially — made environments shippable.**

---

## 3. The Key Insight: Containers Are Just Processes

The single most important mental model in this document:

> **A container is not a lightweight VM. A container is a normal Linux
> process (or process tree) that the kernel is lying to.**

When you run `docker run nginx`:
- There is no guest OS. There is no hypervisor. There is no virtual hardware.
- The nginx process runs directly on the host kernel and appears in the
  host's `ps` output (try it: `ps aux | grep nginx` on the host).
- What makes it a "container" is that the kernel gives this process a
  **restricted and private view** of the system:
  - It sees its own filesystem root (the image's files, not the host's `/`).
  - It sees itself as PID 1 with only its children around it.
  - It sees its own network interfaces and IP address.
  - It sees its own hostname.
  - It is capped in how much CPU/memory it may consume.

Isolation is achieved by *namespacing kernel data structures*, not by
emulating a machine. That is why containers start in milliseconds (it's just
`fork`/`exec` plus some namespace setup), why they have near-zero runtime
overhead (no virtualization layer on the syscall path), and why a container
image can be megabytes instead of gigabytes (no kernel, no init system —
just the app and its userspace libraries).

Two corollaries:

- **All containers on a host share the host's kernel.** You cannot run a
  Windows container on a Linux kernel or vice versa. (Docker Desktop on
  Windows/macOS works by running a hidden Linux VM and running containers
  inside it.)
- **Isolation is a kernel feature, so its strength is a kernel property.**
  A kernel exploit escapes all containers on the host. VMs have a stronger
  isolation boundary (hardware-assisted). This is the fundamental
  container-vs-VM security tradeoff (§12).

---

## 4. The Kernel Primitives

Docker's isolation is the sum of three Linux kernel features plus some
security hardening. If you understand these, you understand containers.

### 4.1 Namespaces — "what can this process *see*?"

A namespace wraps a global kernel resource so that a group of processes gets
its own private instance of it. Created via the `clone(2)` / `unshare(2)`
syscalls with `CLONE_NEW*` flags.

| Namespace | Flag | What becomes private |
|-----------|------|----------------------|
| **Mount** (`mnt`) | `CLONE_NEWNS` | The set of mounted filesystems. This is how a container gets its own `/` — the image's filesystem is mounted and the process is pivoted into it (`pivot_root`, a robust `chroot`). |
| **PID** | `CLONE_NEWPID` | Process IDs. The first process in the namespace is PID 1 *inside* it (while having a normal PID on the host). It can't see or signal host processes. |
| **Network** (`net`) | `CLONE_NEWNET` | Interfaces, IPs, routing tables, ports. A fresh netns has only a loopback device; Docker wires it up with virtual ethernet pairs (§9). Two containers can both bind "port 80" without conflict. |
| **UTS** | `CLONE_NEWUTS` | Hostname and domain name. Why your container has its own hostname (by default, the container ID). |
| **IPC** | `CLONE_NEWIPC` | System V IPC and POSIX message queues. |
| **User** | `CLONE_NEWUSER` | UID/GID mappings. Allows "root inside the container" to map to an unprivileged UID on the host — a major hardening feature (not enabled by default in Docker; see §12). |
| **Cgroup** | `CLONE_NEWCGROUP` | The visible cgroup hierarchy. |
| **Time** | `CLONE_NEWTIME` | Boot/monotonic clock offsets (newer kernels). |

You can see a process's namespaces at `/proc/<pid>/ns/` — each is a file
descriptor-like handle. Two processes are "in the same container," roughly,
when those inode numbers match.

**Try it without Docker** — this is most of a container in one command:

```bash
sudo unshare --fork --pid --mount --net --uts --mount-proc \
     chroot /path/to/extracted-image /bin/sh
```

### 4.2 cgroups — "what can this process *use*?"

Control groups (cgroups v2 on modern systems, mounted at
`/sys/fs/cgroup`) organize processes into a hierarchy and attach resource
**controllers** to each group:

- `memory` — cap RAM (`docker run --memory=512m`). Exceeding it triggers the
  OOM killer *for that group only*.
- `cpu` — proportional shares (`--cpu-shares`) or hard quota (`--cpus=1.5`,
  implemented as quota/period: e.g., 150ms of CPU time per 100ms window).
- `io` — throttle block device bandwidth/IOPS.
- `pids` — cap number of processes (fork-bomb protection, `--pids-limit`).

Namespaces control *visibility*; cgroups control *consumption*. A container
is precisely: a process tree in its own set of namespaces, placed in its own
cgroup. Docker creates a cgroup per container (visible under
`/sys/fs/cgroup/system.slice/docker-<id>.scope` or similar).

Note: cgroups also give you accurate per-container resource accounting —
that's what `docker stats` reads.

### 4.3 The security layer

Since the container shares the host kernel, additional kernel features narrow
what a containerized process can *do*:

- **Capabilities.** Linux splits root's power into ~40 capabilities
  (`CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, ...). Docker drops most of them by
  default even for container-root, keeping a small allowlist (e.g.,
  `CAP_NET_BIND_SERVICE` so a server can bind port 80). Add/remove with
  `--cap-add` / `--cap-drop`.
- **Seccomp.** A syscall filter. Docker's default profile blocks ~40+
  dangerous syscalls (e.g., `kexec_load`, `mount`, `reboot`, `ptrace`
  variants) — a huge reduction in kernel attack surface.
- **AppArmor / SELinux.** Mandatory access control profiles constraining
  file and capability access further.
- **no_new_privs** — prevents gaining privileges via setuid binaries.

`docker run --privileged` disables essentially all of the above — the
container gets all capabilities, no seccomp, and access to host devices. It
is effectively root on the host; treat it that way.

---

## 5. Images and Layered Filesystems

### 5.1 What an image actually is

An image is **not** a disk image or a single blob. It is:

1. An ordered list of **layers** — each layer is a tarball of filesystem
   *changes* (files added/modified, plus "whiteout" markers for deletions).
2. A **config JSON** — default command, entrypoint, environment variables,
   working directory, exposed ports, and the layer list.
3. A **manifest** — JSON tying config + layers together, each referenced by
   the **SHA-256 digest of its content** (content-addressable, like git).

Because everything is content-addressed:
- Identical layers are stored and transferred **once**, no matter how many
  images share them. A hundred images `FROM python:3.12-slim` share those
  base layers on disk and over the network.
- Images are **immutable**: change anything and the digests change. A digest
  reference (`python@sha256:abc...`) pins exact bytes forever, while a *tag*
  (`python:3.12`) is a mutable pointer — like a git branch vs a commit hash.

### 5.2 Union/overlay filesystems

At runtime, the layers must appear as one filesystem. Docker uses a **union
filesystem** — on modern Linux, **overlayfs** (storage driver `overlay2`):

```
container's view of "/"        ← merged view
├── upperdir  (read-write)     ← the container layer: all writes go here
└── lowerdir  (read-only)      ← image layers, stacked
```

- Reads fall through the stack: the topmost layer containing the file wins.
- Writes go to the container's private **writable layer** (upperdir).
- Modifying a file from a lower layer triggers **copy-up**
  (copy-on-write): the whole file is copied to the upper layer, then
  modified. (Gotcha: appending 1 byte to a 10GB lower-layer file copies
  10GB first.)
- Deleting a lower-layer file creates a **whiteout** (a character device
  marker) in the upper layer that masks it. The file still exists in the
  lower layer — which is why `RUN rm secret` in a *later* Dockerfile layer
  does **not** remove the secret from the image.

Consequences worth internalizing:
- **Containers are cheap** because starting one creates only an empty
  writable layer over shared read-only layers — no copying.
- **The writable layer is disposable.** `docker rm` deletes it. Anything you
  want to survive the container goes in a volume (§10).
- **Images are the unit of truth.** `docker commit` can snapshot a
  container's writable layer into a new image layer, but this is an
  anti-pattern for real work — Dockerfiles keep builds reproducible.

### 5.3 Image vs container, precisely

| | Image | Container |
|---|---|---|
| Nature | Immutable, layered filesystem + config | A process (tree) + namespaces + cgroup + one writable layer |
| Analogy | Class / git commit / executable file | Instance / checkout / running process |
| Lifecycle | build → push → pull | create → start → stop → rm |
| Count | One image → many containers | Each has its own writable layer and namespaces |

---

## 6. Docker's Architecture

Docker is not one program. The stack, top to bottom:

```
docker CLI          (client: parses commands, talks REST to the daemon)
   │  REST over unix socket /var/run/docker.sock (or TCP)
   ▼
dockerd             (the Docker daemon / "Engine": API server, image
   │                 management, networking, volumes, build orchestration)
   │  gRPC
   ▼
containerd          (container runtime supervisor: lifecycle, image pulls,
   │                 storage/snapshots. A CNCF project, also used by
   │                 Kubernetes directly — without dockerd.)
   ▼
containerd-shim     (one tiny process per container; holds the container's
   │                 stdio and exit status so containerd/dockerd can restart
   │                 without killing containers)
   ▼
runc                (the actual runtime: a short-lived binary that reads an
                     OCI spec JSON, makes the clone/unshare/cgroup/pivot_root
                     syscalls to create the container process, then exits)
```

Key facts:

- **The CLI is thin.** Everything is the daemon's REST API
  (`curl --unix-socket /var/run/docker.sock http://localhost/containers/json`).
  This is also why mounting `docker.sock` into a container grants full
  control of the host (§12).
- **`docker build` context**: the client tars up the build context directory
  and ships it to the daemon — which is why a stray `node_modules` makes
  builds slow, and why `.dockerignore` matters.
- **runc is the reference OCI runtime** — extracted from Docker and donated
  to the Open Container Initiative. Alternatives plug in at this layer:
  `gVisor` (userspace kernel), `Kata` (lightweight VMs per container),
  `crun` (faster, in C).
- **The shim is why** `docker restart` of the daemon doesn't kill your
  running containers (with live-restore enabled): the parent of the
  container process is the shim, not dockerd.
- **Docker Desktop** (Windows/macOS) runs this whole Linux stack inside a
  hidden lightweight VM (WSL2 on Windows) and forwards the socket, ports,
  and file mounts. That's why file-mount IO on Mac/Windows is slower than on
  native Linux — it crosses a VM boundary.

---

## 7. The Container Lifecycle, Step by Step

What actually happens on `docker run -d -p 8080:80 nginx:1.27`:

1. **CLI → daemon**: POST `/containers/create` with the parsed options.
2. **Image resolution**: daemon checks local storage for `nginx:1.27`. If
   missing → resolves the tag at the registry, pulls the **manifest**,
   compares layer digests to what's already local, downloads only missing
   layers (in parallel), verifies each against its SHA-256, and unpacks them
   into overlayfs snapshots.
3. **Container creation**: daemon prepares the container object — generates
   an ID, creates the writable layer, allocates an IP from the bridge
   network's subnet, builds the OCI runtime spec (a JSON describing
   namespaces, cgroup limits, mounts, capabilities, seccomp profile, env,
   command).
4. **Handoff**: dockerd → containerd → spawns a `containerd-shim` → shim
   invokes **runc** with the spec.
5. **runc creates the process**: `clone()` with the `CLONE_NEW*` flags →
   in the child: join/create cgroup, set up the overlay mount, `pivot_root`
   into it, mount `/proc`, `/sys`, `/dev` fresh, drop capabilities, apply
   seccomp, set hostname, then `execve("/docker-entrypoint.sh", ...)`.
   runc then *exits* — the container process is reparented to the shim.
6. **Networking**: a **veth pair** (virtual ethernet cable) is created; one
   end goes into the container's netns as `eth0` with the allocated IP, the
   other attaches to the `docker0` bridge on the host. For `-p 8080:80`,
   the daemon programs an iptables/nftables DNAT rule (and runs
   `docker-proxy` for edge cases) so host:8080 → container-IP:80.
7. **Process 1**: nginx is now PID 1 in its PID namespace, sees the image
   filesystem as `/`, its own `eth0`, and is capped by its cgroup.

Stopping (`docker stop`): daemon sends **SIGTERM** to PID 1, waits 10 seconds
(`--stop-timeout`), then **SIGKILL**. Two classic gotchas:

- If your Dockerfile uses **shell form** (`CMD node server.js` → actually
  `/bin/sh -c "node server.js"`), PID 1 is `sh`, which does not forward
  SIGTERM → your app never gets a graceful shutdown and is SIGKILLed after
  the timeout. Use **exec form**: `CMD ["node", "server.js"]`.
- PID 1 has special signal semantics (default handlers don't apply) and is
  responsible for reaping zombie children. For apps that spawn children, run
  a tiny init: `docker run --init` (injects `tini`).

`docker rm` deletes the writable layer and metadata. The image is untouched.

---

## 8. Dockerfiles and the Build System

### 8.1 The model

A Dockerfile is a script where **each instruction produces a layer** (for
filesystem-changing instructions: `FROM`, `RUN`, `COPY`, `ADD`) or metadata
(`ENV`, `EXPOSE`, `CMD`, ...). The builder (BuildKit, the default since
Docker 23) executes instructions in a DAG, caching each step.

```dockerfile
# syntax=docker/dockerfile:1

########## build stage ##########
FROM golang:1.23 AS build
WORKDIR /src
# Copy dependency manifests FIRST — this layer's cache survives code changes
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

########## runtime stage ##########
FROM gcr.io/distroless/static-debian12
COPY --from=build /out/app /app
USER nonroot
ENTRYPOINT ["/app"]
```

### 8.2 Cache semantics — the thing that makes builds fast or slow

For each instruction, BuildKit computes a cache key from the instruction
itself, the parent layer, and (for `COPY`/`ADD`) the **checksums of the
copied files**. A cache miss invalidates that step **and every step after
it**. Therefore the cardinal rule:

> **Order instructions from least-frequently-changing to
> most-frequently-changing.**

Copy `package.json` + lockfile, install dependencies, *then* copy source.
A code-only change then reuses the (slow) dependency-install layer.

Other cache facts:
- `RUN apt-get update && apt-get install -y foo` must be **one instruction**
  — if `update` were its own cached layer, a later edit to the install line
  would run against a stale package index.
- Each `RUN` is a layer: cleanup must happen in the *same* `RUN`
  (`... && rm -rf /var/lib/apt/lists/*`), because deleting in a later layer
  only adds a whiteout — the bytes remain in the earlier layer.
- BuildKit extras: `RUN --mount=type=cache,target=/root/.cache/pip ...`
  (persistent build cache that doesn't enter the image),
  `RUN --mount=type=secret,...` (secrets available during build, never
  stored in a layer), parallel execution of independent stages.

### 8.3 Multi-stage builds

The pattern above: heavy toolchains (compilers, dev headers) live in a build
stage; the final `FROM` starts from a minimal base and copies in only the
artifacts. This is the standard way to get small, low-attack-surface
production images (a static Go binary on `distroless` can be <20 MB, vs a
1+ GB image if you shipped the golang toolchain).

Base image spectrum, big → small:
`ubuntu` (~78MB) → `debian:slim` (~30MB) → `alpine` (~5MB, but **musl libc**,
not glibc — a real compatibility difference for Python wheels and some
binaries) → `distroless` (no shell, no package manager) → `scratch` (empty;
for static binaries).

### 8.4 ENTRYPOINT vs CMD

- `ENTRYPOINT` = the executable; `CMD` = default arguments to it.
- `docker run image extra args` **replaces CMD**, appends to ENTRYPOINT.
- Both should be exec form (JSON array) — see the PID 1 signal gotcha in §7.
- Convention: `ENTRYPOINT ["myapp"]`, `CMD ["--default-flag"]` makes the
  image behave like a binary.

---

## 9. Networking

Docker networking is built from Linux primitives you can inspect: network
namespaces, veth pairs, bridges, and iptables.

### 9.1 The default: bridge network

```
container A (eth0 172.17.0.2) ─veth─┐
container B (eth0 172.17.0.3) ─veth─┤── docker0 bridge (172.17.0.1) ── iptables NAT ── host eth0 ── world
```

- Each container's `eth0` is one end of a **veth pair**; the other end is a
  port on the `docker0` **bridge** (a virtual L2 switch on the host).
- Outbound traffic is **masqueraded** (SNAT) to the host's IP.
- Inbound requires **port publishing**: `-p 8080:80` installs a DNAT rule
  host:8080 → container:80. Without `-p`, nothing outside the host can reach
  the container.

### 9.2 User-defined networks — what you should actually use

`docker network create mynet` gives you a separate bridge **plus an embedded
DNS server**: containers on it resolve each other **by container name**
(`ping db` just works). The default bridge lacks this. Compose (§13) creates
one per project automatically — which is why services address each other as
`http://api:3000` in Compose files. Containers on different user-defined
networks are isolated from each other at the firewall level — networks
double as segmentation.

### 9.3 Other network drivers

- `host` — no netns: the container shares the host's network stack. No
  isolation, no port mapping needed, no NAT overhead.
- `none` — loopback only.
- `overlay` — multi-host virtual network (VXLAN encapsulation); the basis of
  Swarm/multi-node networking.
- `macvlan` — container gets its own MAC/IP on the physical LAN.

DNS inside a container: Docker mounts a generated `/etc/resolv.conf` and
(on user-defined networks) points it at the embedded DNS at `127.0.0.11`.

---

## 10. Storage

The writable layer dies with the container and is slow (copy-up overhead).
For data, Docker offers three mount types:

| Type | Syntax | Backed by | Use for |
|------|--------|-----------|---------|
| **Volume** | `-v mydata:/var/lib/postgresql/data` | Docker-managed dir under `/var/lib/docker/volumes/` (or a volume plugin: NFS, EBS...) | Databases, any persistent app data. Survives `docker rm`; portable across container replacements; best IO path. |
| **Bind mount** | `-v /host/path:/container/path` or `--mount type=bind,...` | An arbitrary host directory | Dev workflows (mount source code into the container for live reload), host config files. Couples the container to host layout. |
| **tmpfs** | `--tmpfs /scratch` | RAM | Secrets, scratch space that must never touch disk. |

Facts that matter:
- A mount **shadows** whatever the image has at that path — with one twist:
  a *named volume* mounted on a *non-empty image directory* gets
  pre-populated with the image's content on first use; a bind mount never
  does (it shows you the host dir, even if empty).
- File permissions are by numeric UID/GID — the container's UID must match
  the host files' ownership on bind mounts (classic "permission denied" in
  dev setups; on Linux, `-u $(id -u):$(id -g)` is the usual fix).
- Anonymous volumes (a bare `VOLUME /data` in a Dockerfile, or `-v /data`)
  accumulate as orphans; `docker volume prune` cleans them.

---

## 11. Registries and Image Distribution

A **registry** is an HTTP content-addressed blob store speaking the OCI
Distribution API: Docker Hub, GHCR, ECR/GCR/ACR, or self-hosted
(`registry:2`, Harbor).

Image reference anatomy:

```
registry.example.com:5000 / team/app : 1.4.2 @ sha256:8f3e...
└──────── registry ─────┘ └─ repo ─┘ └ tag ┘ └── digest ──┘
```

- No registry → defaults to `docker.io`; no tag → `:latest`.
  **`latest` is nothing special** — just the default tag name. It is *not*
  guaranteed to be the newest version; it's whatever was last pushed to that
  tag. Never deploy by `latest` in production; pin versions, and for supply
  chain integrity, pin digests.
- `push`/`pull` transfer only layers the other side doesn't have (digest
  check first) — this is why pulls of a new app version on an unchanged base
  are fast.
- A tag can point at a **manifest list** (multi-arch): one `python:3.12`
  reference resolves to the right image for amd64 / arm64 automatically.
  `docker buildx build --platform linux/amd64,linux/arm64` builds them.
- Signing/verification: Docker Content Trust (Notary) historically; the
  ecosystem has largely moved to **cosign**/sigstore.

---

## 12. Security Model

Layered summary of what stands between a containerized process and the host:

1. Namespaces (visibility) — §4.1
2. cgroups (resource DoS protection) — §4.2
3. Dropped capabilities, seccomp, AppArmor/SELinux — §4.3
4. Optionally: user namespaces / rootless mode

What an expert must know:

- **Root in the container is root on the host by default** (same UID 0),
  merely *constrained* by the layers above. Defense in depth, not a hard
  boundary. Mitigations: run as non-root (`USER` in Dockerfile), enable
  user-namespace remapping, or use **rootless Docker** (the whole daemon
  runs unprivileged).
- **`-v /var/run/docker.sock:...` = handing over the host.** Anyone with the
  socket can start a privileged container with `/` mounted. The same is true
  of `--privileged`, `--pid=host`, `--net=host`, `--cap-add=SYS_ADMIN`.
- **The kernel is the shared boundary.** Container escape = kernel (or
  runtime) vulnerability. For hostile multi-tenant workloads use gVisor or
  Kata (VM-per-container) at the runc layer, or actual VMs.
- **Image supply chain** is the other half of container security: scan
  images (`docker scout`, Trivy, Grype), use minimal bases (less installed =
  less CVE surface), pin digests, don't bake secrets into layers (they're
  extractable from history even if "deleted" later — §5.2, §8.2; use
  BuildKit secret mounts and runtime env/secret injection instead).
- `docker history <image>` shows every layer's creating command — assume
  anything ever written into a layer is public to whoever can pull the image.

---

## 13. Docker Compose

Real applications are several containers: app + database + cache + proxy.
Compose declares them in one YAML file and manages them as a unit.

```yaml
# compose.yaml
services:
  api:
    build: .
    ports: ["8080:8080"]
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/app
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
volumes:
  pgdata:
```

`docker compose up -d` creates a project network (services resolve each
other by name: the API reaches postgres at host `db`), volumes, and
containers; `docker compose down` tears it down (`-v` to also delete
volumes). `depends_on` with `condition: service_healthy` gates startup on
the dependency's **healthcheck** — plain `depends_on` only orders *start*,
not *readiness*.

Compose is single-host. It is the standard for local dev and fine for
small single-server deployments; multi-host orchestration (scheduling,
self-healing, rolling deploys across a fleet) is Kubernetes' job.

---

## 14. What Docker Is Not

### Docker vs virtual machines

| | Container | VM |
|---|---|---|
| Isolation mechanism | Kernel namespaces/cgroups | Hypervisor + virtual hardware |
| Kernel | Shared with host | Own kernel per VM |
| Startup | Milliseconds | Seconds–minutes |
| Size | MBs (app + userspace libs) | GBs (full OS) |
| Overhead | ~none (native syscalls) | Virtualization layer |
| Security boundary | Softer (shared kernel) | Harder (hardware-assisted) |
| Runs different OS? | No (Linux binaries on Linux kernel) | Yes |

They compose rather than compete: production containers usually run *inside*
VMs (every managed Kubernetes node is a VM).

### Docker vs Kubernetes
Docker builds and runs containers on **one machine**. Kubernetes schedules
and operates containers across a **fleet**: replicas, self-healing, service
discovery, rolling updates, autoscaling. Kubernetes doesn't need Docker — it
talks CRI to containerd or CRI-O directly (the 2020 "Docker deprecation" was
only about removing the dockerd middleman on nodes). **Docker images run on
Kubernetes unchanged**, because both implement the OCI standards.

### The OCI ecosystem
The Open Container Initiative standardizes three things — **image format**,
**runtime behavior** (what runc implements), and **distribution** (registry
API). "Docker" today is one vendor's toolchain over these standards.
Interchangeable pieces: **Podman** (daemonless, rootless-first,
CLI-compatible drop-in), **Buildah**/**kaniko**/**BuildKit** (builders),
**containerd**/**CRI-O** (runtimes), **skopeo** (image copying). Expertise in
the concepts transfers across all of them.

---

## 15. Expert-Level Details, Gotchas, and Best Practices

The condensed list of things that separate "uses Docker" from "understands
Docker":

**Images & builds**
1. Order Dockerfile instructions by change frequency; copy lockfiles and
   install deps before copying source (§8.2).
2. One logical concern per `RUN`; clean up in the same layer you created the
   mess (§8.2).
3. Multi-stage builds for anything compiled; distroless/slim runtime bases.
4. `.dockerignore` always (`.git`, `node_modules`, build output) — context
   upload size and cache correctness both depend on it.
5. Never put secrets in layers, args, or env baked into the image — layer
   history is forever. BuildKit `--mount=type=secret` at build time; env
   vars/secret managers at runtime.
6. Pin base images (at least minor version; digest for prod). `latest` is a
   mutable pointer, not "newest release" (§11).
7. Alpine = musl libc: smaller, but Python wheels/glibc binaries may need
   recompilation or fail subtly. `debian:slim` is the safer default.

**Runtime**
8. Exec-form ENTRYPOINT/CMD, or your app never sees SIGTERM (§7).
9. `--init` (tini) for anything that forks; PID 1 must reap zombies (§7).
10. Run as non-root (`USER`); drop capabilities you don't need.
11. Always set memory limits in production; understand that hitting the
    limit means the cgroup OOM killer, i.e., SIGKILL — apps (especially
    JVMs: use container-aware heap settings, default since JDK 10+) must be
    configured for their cgroup, not the host's RAM.
12. Containers are ephemeral **by design**: logs to stdout/stderr (collected
    by logging drivers, viewed with `docker logs`), data to volumes, config
    via env — this is "twelve-factor," and Docker is built around it.
13. One process/concern per container. Not a hard rule (sidecar helpers are
    fine) but "SSH + cron + app in one container" means you've rebuilt a VM
    badly.

**Operations**
14. `docker system df` / `docker system prune` — build cache, stopped
    containers, and dangling images eat disks; unattended hosts fill up.
15. `docker exec -it <c> sh` for a shell in a running container;
    `docker debug` / ephemeral debug images when the image has no shell
    (distroless).
16. `docker inspect` is the ground truth for any object (IPs, mounts, actual
    config); `docker events` streams the daemon's activity.
17. Restart policies (`--restart unless-stopped`) for single-host services —
    Docker's minimal self-healing.
18. Healthchecks make orchestration meaningful — a running process is not a
    ready service (§13).
19. Mounted-socket and `--privileged` containers are host-root equivalent;
    audit for them (§12).

**Mental models to keep**
20. A container is a process the kernel is lying to (§3).
21. An image is a git-like content-addressed stack of tar diffs (§5).
22. Namespaces = what you see; cgroups = what you get; seccomp/caps = what
    you may do (§4).
23. Docker's real invention was distribution — the shippable environment —
    not isolation (§2.7).

---

## 16. Command Reference

```bash
# Lifecycle
docker run -d --name web -p 8080:80 nginx:1.27   # create + start, detached
docker run -it --rm ubuntu:24.04 bash            # interactive, auto-remove
docker ps -a                                     # list (incl. stopped)
docker stop web && docker rm web                 # graceful stop, remove
docker restart web
docker logs -f --tail 100 web
docker exec -it web sh                           # shell into running container
docker stats                                     # live cgroup resource usage
docker inspect web                               # full JSON state
docker cp web:/etc/nginx/nginx.conf .            # copy files in/out

# Images
docker build -t myapp:1.0 .
docker images
docker history myapp:1.0                         # layer-by-layer provenance
docker tag myapp:1.0 ghcr.io/me/myapp:1.0
docker push ghcr.io/me/myapp:1.0
docker pull redis:7
docker rmi myapp:1.0
docker buildx build --platform linux/amd64,linux/arm64 -t me/app --push .

# Networks & volumes
docker network create mynet
docker run -d --network mynet --name db postgres:16
docker network inspect mynet
docker volume create pgdata
docker volume ls / inspect / prune

# Compose
docker compose up -d --build
docker compose ps / logs -f / exec api sh
docker compose down -v                           # -v: delete volumes too

# Housekeeping
docker system df                                 # disk usage by type
docker system prune -a                           # remove all unused (careful)
docker image prune / container prune / builder prune
```

---

## Closing Summary

Before Docker, we isolated software with whole machines, then VMs (heavy,
still-drifting environments inside), then configuration management (mutating
live machines toward a spec), then golden VM images (immutable but too heavy
to be a per-commit artifact), while the kernel quietly grew namespaces and
cgroups and LXC exposed them to experts. Docker's contribution was to fuse
these into a developer product: the **Dockerfile** (environment as
reproducible code), the **image** (an immutable, layered, content-addressed
artifact cheap enough to build per commit), the **registry** (push/pull
distribution of entire environments), and the **container** (that image
running as an ordinary Linux process behind namespace, cgroup, and seccomp
walls). The result: the exact same bytes run everywhere, machines become
interchangeable, and "works on my machine" stops being an excuse — because
you ship the machine.
