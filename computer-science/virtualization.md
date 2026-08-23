# Virtualization: From Zero to Expert

A self-contained deep dive into virtualization — the art of making one physical computer behave like many, or making software believe it is running on hardware that doesn't physically exist. It builds from first principles up to the internals that hypervisor engineers, cloud architects, and performance specialists work with daily. It assumes you know the basics of operating systems (processes, virtual memory, kernel/user mode) — see [operating-systems.md](operating-systems.md) if you don't.

---

## Table of Contents

1. [Part 1 — What Virtualization Is](#part-1--what-virtualization-is)
2. [Part 2 — A Brief History](#part-2--a-brief-history)
3. [Part 3 — Hypervisors: Type 1 and Type 2](#part-3--hypervisors-type-1-and-type-2)
4. [Part 4 — The Theory: Popek & Goldberg](#part-4--the-theory-popek--goldberg)
5. [Part 5 — CPU Virtualization](#part-5--cpu-virtualization)
6. [Part 6 — Memory Virtualization](#part-6--memory-virtualization)
7. [Part 7 — I/O Virtualization](#part-7--io-virtualization)
8. [Part 8 — Paravirtualization](#part-8--paravirtualization)
9. [Part 9 — OS-Level Virtualization: Containers](#part-9--os-level-virtualization-containers)
10. [Part 10 — The Middle Ground: MicroVMs and Sandboxes](#part-10--the-middle-ground-microvms-and-sandboxes)
11. [Part 11 — Storage and Network Virtualization](#part-11--storage-and-network-virtualization)
12. [Part 12 — Live Migration](#part-12--live-migration)
13. [Part 13 — Nested Virtualization](#part-13--nested-virtualization)
14. [Part 14 — Virtualization and the Cloud](#part-14--virtualization-and-the-cloud)
15. [Part 15 — Security](#part-15--security)
16. [Part 16 — Performance: How Experts Think](#part-16--performance-how-experts-think)
17. [Part 17 — Hands-On Lab](#part-17--hands-on-lab)
18. [Part 18 — The Road to Expertise](#part-18--the-road-to-expertise)

---

# Part 1 — What Virtualization Is

## 1.1 The one-sentence definition

**Virtualization is the creation of a software layer that presents an interface identical (or nearly identical) to real hardware, so that software written for the real thing runs unmodified on the fake one.**

The software that does this is called a **hypervisor** or **Virtual Machine Monitor (VMM)**. The fake computer it creates is a **virtual machine (VM)** or **guest**. The real computer underneath is the **host**.

```
+--------------------------------------------------------------+
|   VM 1              VM 2              VM 3                   |
|  +-----------+     +-----------+     +-----------+           |
|  | Apps      |     | Apps      |     | Apps      |           |
|  +-----------+     +-----------+     +-----------+           |
|  | Guest OS  |     | Guest OS  |     | Guest OS  |           |
|  | (Linux)   |     | (Windows) |     | (FreeBSD) |           |
|  +-----------+     +-----------+     +-----------+           |
|  | virtual   |     | virtual   |     | virtual   |           |
|  | hardware  |     | hardware  |     | hardware  |           |
|  +-----------+     +-----------+     +-----------+           |
|                                                              |
|  +--------------------------------------------------------+  |
|  |            Hypervisor (VMM)                            |  |
|  +--------------------------------------------------------+  |
|  +--------------------------------------------------------+  |
|  |            Physical hardware (CPU, RAM, disk, NIC)     |  |
|  +--------------------------------------------------------+  |
+--------------------------------------------------------------+
```

Each guest OS believes it owns a whole computer — its own CPUs, its own physical memory starting at address 0, its own disks and network cards. All of it is an illusion maintained by the hypervisor, exactly the way an OS maintains the illusion for each *process* that it owns the whole machine. Virtualization is the same trick, one layer down.

## 1.2 The three properties a VMM must have

Popek and Goldberg (1974, more in Part 4) defined what makes a "real" virtual machine monitor:

1. **Fidelity (equivalence)** — software runs on the VM exactly as it would on real hardware, apart from timing.
2. **Safety (resource control)** — the guest cannot escape the sandbox; the VMM stays in complete control of physical resources.
3. **Performance (efficiency)** — the vast majority of guest instructions execute *directly on the real CPU* with no VMM involvement. This is what separates virtualization from **emulation**.

## 1.3 Virtualization vs. emulation vs. simulation

These words get confused constantly. The distinction is *what runs where*:

| Technique | Guest instructions execute... | Speed | Example |
|---|---|---|---|
| **Virtualization** | Directly on the host CPU (same architecture) | Near-native | KVM, VMware, Hyper-V |
| **Emulation** | Interpreted/translated in software (any architecture) | 5–100× slower | QEMU (pure), game console emulators |
| **Simulation** | Modeled at whatever detail you want (cycles, signals) | Very slow | gem5, cycle-accurate CPU simulators |

Emulation can run ARM code on an x86 machine because every guest instruction is translated in software. Virtualization cannot cross architectures — its whole speed advantage comes from letting the real CPU run guest code natively.

(QEMU straddles the line: alone it's an emulator with dynamic binary translation (TCG); paired with KVM it's a virtualizer that only *emulates devices*, while the CPU runs guest code natively.)

## 1.4 Why bother? The motivations

1. **Server consolidation.** A physical server at 10% utilization is wasted money. Run ten VMs on it.
2. **Isolation.** A crash, compromise, or resource hog in one VM doesn't touch the others (a stronger boundary than processes sharing a kernel).
3. **Heterogeneity.** Linux, Windows, and BSD on the same box simultaneously.
4. **Snapshots and cloning.** A VM's whole state is data — freeze it, copy it, roll it back, template it.
5. **Live migration.** Move a *running* VM between physical hosts with milliseconds of downtime (Part 12).
6. **The cloud.** AWS, Azure, and GCP are, at bottom, planet-scale VM (and container) rental businesses. Virtualization is the enabling technology of cloud computing.
7. **Development and testing.** Reproduce a customer's environment, test an installer on a clean OS, detonate malware safely.

## 1.5 The many meanings of "virtual"

"Virtualization" in the broad sense means *interposing a mapping layer between a consumer of a resource and the resource itself*. Computer science does this everywhere:

- **Virtual memory** — pages map to frames (the OS virtualizes RAM per-process).
- **Virtual machines (hardware)** — this document's subject.
- **Virtual machines (language)** — JVM, CPython's VM, WebAssembly: an *instruction set* that never existed in silicon, always implemented in software. Different beast, same word.
- **Virtual LANs, virtual disks, virtual functions** — the pattern repeats at every layer.

The pattern is always: **add indirection → gain flexibility (sharing, isolation, migration) → pay some overhead → engineer the overhead away.**

---

# Part 2 — A Brief History

Virtualization is old — older than Unix.

- **1960s — IBM invents it.** IBM's CP-40 and then **CP/CMS on the System/360-67** (1967) time-shared a mainframe by giving every user a private *virtual System/360*. This lineage survives today as z/VM. Mainframes have been running production VMs for almost 60 years.
- **1974 — Popek & Goldberg** publish "Formal Requirements for Virtualizable Third Generation Architectures," the theoretical foundation (Part 4).
- **1980s–90s — The dark ages.** Cheap minicomputers and PCs made "one machine per workload" affordable, and the x86 architecture was (famously) *not* virtualizable by the Popek–Goldberg criteria. Virtualization was considered a mainframe curiosity.
- **1998 — VMware.** Stanford researchers (the Disco project, Rosenblum et al.) found a way to virtualize x86 *anyway*, using **binary translation** (Part 5.3). VMware Workstation (1999) and ESX brought virtualization to commodity hardware and ignited the modern industry.
- **2003 — Xen** introduced practical **paravirtualization** (Part 8): modify the guest OS to cooperate, and x86 virtualization gets much faster. Early Amazon EC2 ran on Xen.
- **2005–2006 — Hardware joins in.** Intel **VT-x** and AMD **AMD-V** added CPU modes that made x86 properly virtualizable at last (Part 5.4). Binary translation and paravirtualized CPUs gradually became historical footnotes.
- **2007 — KVM** merged into the Linux kernel: the Linux kernel *itself* becomes a hypervisor. Paired with QEMU for device emulation, it now underpins most of the cloud (GCP, much of AWS's newer fleet via Nitro, OpenStack).
- **2008 — Hyper-V** ships; Microsoft's hypervisor now underlies Azure and Windows features like WSL2 and Virtualization-Based Security.
- **2013 — Docker** popularizes OS-level virtualization (containers), building on Linux namespaces/cgroups (Part 9).
- **2017–present — MicroVMs.** AWS **Firecracker** (powering Lambda/Fargate) shows a VM can boot in ~125 ms with ~5 MB overhead — VMs light enough to compete with containers (Part 10).

The historical arc: **software tricks → guest cooperation → hardware support → commoditization → miniaturization.**

---

# Part 3 — Hypervisors: Type 1 and Type 2

The classic taxonomy (from Goldberg's 1973 thesis) splits hypervisors by *what they run on*.

## 3.1 Type 1 — bare metal

The hypervisor **is** the operating system of the physical machine. It boots first and controls all hardware directly.

```
+---------------------------------------------+
|   VM    |   VM    |   VM    |  management   |
|         |         |         |  VM / console |
+---------------------------------------------+
|         Type 1 hypervisor                   |
+---------------------------------------------+
|         Hardware                            |
+---------------------------------------------+
```

Examples: **VMware ESXi**, **Microsoft Hyper-V**, **Xen**, IBM z/VM and PowerVM.

Traits: smallest attack surface, best and most predictable performance, used in datacenters and clouds. Often paired with a privileged management VM (Xen's *dom0*, Hyper-V's *root partition*) that runs device drivers and admin tooling, keeping the hypervisor core tiny.

## 3.2 Type 2 — hosted

The hypervisor runs as an application (plus kernel modules) **on top of** a conventional OS, which keeps owning the hardware.

```
+---------------------------------------------+
|  Apps   |  Hypervisor app  ->  |  VM | VM | |
+---------------------------------------------+
|         Host OS (Windows/macOS/Linux)       |
+---------------------------------------------+
|         Hardware                            |
+---------------------------------------------+
```

Examples: **VirtualBox**, **VMware Workstation/Fusion**, **Parallels Desktop**, QEMU.

Traits: easy to install, coexists with your desktop, ideal for development — at the cost of extra layers (guest I/O traverses the host OS) and less predictable performance.

## 3.3 The taxonomy is blurry — and that's instructive

**KVM** breaks the dichotomy: it turns the running Linux kernel into a hypervisor. Is that Type 1 (the kernel controls hardware directly) or Type 2 (there's a general-purpose OS involved)? Most people call it Type 1, because guest execution is handled in-kernel with hardware support and there's no separate OS *under* the hypervisor. Hyper-V similarly slides beneath the Windows that installed it, demoting Windows to a guest-like root partition. The lesson: the types describe an architecture spectrum, not two boxes. What actually matters is **where the performance-critical paths run** and **how big the trusted computing base is**.

## 3.4 The anatomy of a modern hypervisor stack (KVM/QEMU as the example)

```
+------------------------------------------------------+
|  Guest user space                                    |
|  Guest kernel                                        |
+---------------------▲--------------------------------+
                      | VM exits / entries (hardware)
+---------------------▼--------------------------------+
| KVM (kernel module): CPU & memory virtualization,    |
|   runs vCPUs via VT-x/AMD-V, manages EPT             |
+---------------------▲--------------------------------+
                      | ioctl(/dev/kvm), exits needing devices
+---------------------▼--------------------------------+
| QEMU (user process, one per VM):                     |
|   device models (disk, NIC, VGA...), virtio backends,|
|   live-migration logic, monitor/management API       |
+------------------------------------------------------+
| Linux host kernel: scheduler treats each vCPU as a   |
|   thread; page cache, network stack, drivers         |
+------------------------------------------------------+
```

Division of labor: **KVM does the fast, hardware-assisted part** (CPU, memory); **QEMU does the messy part** (pretending to be hundreds of possible devices). Each vCPU is just a host thread; the host scheduler schedules VMs like any other workload. Management layers (libvirt, virt-manager, Proxmox, OpenStack) sit on top.

---

# Part 4 — The Theory: Popek & Goldberg

This 1974 paper is short, formal, and still the sharpest lens for understanding *why* CPU virtualization is hard.

## 4.1 Two classifications of instructions

- **Privileged instructions**: instructions that **trap** (cause an exception into the kernel/hypervisor) when executed in user mode, but execute fine in kernel mode. Example: loading the page-table base register.
- **Sensitive instructions**: instructions that either (a) change machine configuration/resources (*control-sensitive* — e.g., changing memory mappings, interrupt state) or (b) **behave differently depending on the privilege level or configuration** (*behavior-sensitive* — e.g., reading a register that reveals what mode you're in).

## 4.2 The theorem

> An architecture can support an efficient VMM (via **trap-and-emulate**) if the set of **sensitive** instructions is a **subset** of the **privileged** instructions.

Why: run the guest OS *deprivileged* (in user mode). All its ordinary instructions run natively at full speed. Any time it tries something sensitive, the instruction traps into the VMM, which **emulates** the effect against the *virtual* machine state and resumes the guest. The guest can't tell the difference (fidelity), can't touch real resources (safety), and only sensitive instructions pay overhead (efficiency).

```
Guest kernel (running deprivileged)
    |
    |  executes normal instruction  -->  runs natively, full speed
    |  executes sensitive instr.    -->  TRAP
    |                                     |
    |                                     v
    |                              VMM handler:
    |                                emulate against virtual state
    |                                (e.g., update *virtual* CR3)
    |<----------------------- resume guest
```

## 4.3 Why classic x86 failed the test

x86 (pre-2005) had **17 sensitive-but-unprivileged instructions**. The famous ones:

- `POPF` in user mode *silently ignores* changes to the interrupt-enable flag instead of trapping. A deprivileged guest kernel thinks it disabled interrupts; nothing happened; no trap ever told the VMM.
- `SMSW`, `SGDT`, `SIDT`, `SLDT` let user-mode code *read* privileged state (revealing real machine configuration to the guest — a fidelity leak).

So trap-and-emulate alone was impossible on x86. The industry's three answers — **binary translation** (VMware), **paravirtualization** (Xen), and **hardware extensions** (VT-x/AMD-V) — are the subject of the next parts. (Note: modern ARMv8 and RISC-V's H-extension were *designed* to be cleanly virtualizable; the lesson was learned.)

---

# Part 5 — CPU Virtualization

The problem: multiple guest OSes each believe they run in kernel mode on their own CPUs. The hypervisor must preserve that belief while keeping real kernel-mode control for itself.

## 5.1 Trap-and-emulate (the ideal)

Described in 4.2. On a Popek–Goldberg-compliant architecture, this is the whole story: deprivilege the guest, trap the sensitive instructions, emulate them against a per-VM **virtual CPU state** structure (virtual registers, virtual interrupt flag, virtual privilege level).

## 5.2 The vCPU abstraction

A **vCPU** is to a physical CPU what a thread is to a process: a schedulable virtual execution context holding register state. The hypervisor time-slices physical cores among vCPUs. Overcommit (more vCPUs than pCPUs) is normal and fine — until it isn't (see 16.3 on co-scheduling and steal time).

## 5.3 Binary translation (VMware's x86 workaround, 1998–~2010)

Since x86's sensitive instructions wouldn't trap, VMware **rewrote the guest's kernel code on the fly**:

1. The VMM reads the next basic block of guest *kernel* code before it runs.
2. Most instructions are copied through unchanged ("identical translation").
3. Sensitive instructions are replaced with calls into the VMM that emulate them.
4. Translated blocks are cached, so hot code is translated once. Guest *user-mode* code needs no translation at all — it runs directly.

This is a JIT compiler for machine code, from x86 to safe-x86. It was an engineering tour de force and often *outperformed* early hardware-assisted virtualization (a cached translation is cheaper than a VM exit). It's obsolete now, but the technique lives on in emulators (QEMU TCG, Rosetta 2) and DBI tools.

## 5.4 Hardware-assisted virtualization: Intel VT-x / AMD-V (2005+)

The hardware fix: add an orthogonal privilege dimension.

- **VMX root mode** — where the hypervisor runs (ring 0–3 all exist here).
- **VMX non-root mode** — where the guest runs. The guest kernel gets a *real ring 0* — but a defanged one: sensitive events in non-root mode cause a **VM exit** back to the hypervisor.

```
            VMLAUNCH / VMRESUME
 Hypervisor ─────────────────────►  Guest
 (VMX root)                         (VMX non-root,
     ▲                               guest has its own rings 0-3)
     │            VM exit
     └──────────────────────────────┘
        (CPUID, I/O access, EPT violation,
         interrupt, HLT, ... — reason recorded)
```

The **VMCS (Virtual Machine Control Structure)** — AMD's equivalent is the **VMCB** — is a per-vCPU in-memory structure the hardware itself reads and writes. It holds:

- **Guest-state area** — the guest's registers, saved/restored automatically on exit/entry.
- **Host-state area** — where to resume the hypervisor on exit.
- **Execution controls** — a bitmap of *which* events cause exits (the hypervisor tunes this: e.g., "don't exit on interrupt-flag changes, do exit on I/O port access").
- **Exit information** — why the last exit happened, so the handler can dispatch quickly.

**The core performance rule of modern virtualization: performance ≈ (number of VM exits) × (cost per exit).** An exit is a heavyweight context switch (state save/restore, TLB/cache effects — hundreds to thousands of cycles). Fifteen years of CPU and hypervisor engineering have been about *eliminating exits*: hardware page tables (Part 6), interrupt virtualization (**APICv/AVIC** — deliver interrupts to guests without exiting), posted interrupts, and paravirtual devices (Parts 7–8).

## 5.5 Timekeeping — the subtle nightmare

Guests can't tell when they weren't running. Naive tick counting makes guest clocks drift badly. Solutions: hardware TSC offsetting/scaling (the CPU adjusts the timestamp counter per-VMCS), and **paravirtual clocks** (kvm-clock, Hyper-V reference TSC) where the hypervisor shares its notion of time with the guest through shared memory. Timekeeping bugs are a classic source of "impossible" distributed-systems failures inside VMs.

---

# Part 6 — Memory Virtualization

## 6.1 Three levels of address

Ordinary virtual memory has two levels; virtualization adds a third:

```
GVA ── guest page tables ──► GPA ── hypervisor mapping ──► HPA
guest virtual                guest "physical"              host physical
(what a guest process sees)  (what the guest OS thinks     (actual RAM)
                              is RAM)
```

The guest OS builds page tables mapping GVA→GPA, believing GPAs are real RAM. The hypervisor maintains the GPA→HPA mapping. But the physical MMU walks only *one* set of page tables. Two solutions:

## 6.2 Shadow page tables (the software way)

The hypervisor maintains, for each guest page table, a **shadow** table mapping **GVA directly to HPA** — the composition of both mappings — and points the real MMU at the shadow.

To keep shadows coherent, the hypervisor **write-protects the guest's page tables**; every guest PTE update traps, and the VMM updates the shadow to match. This works but is brutal for workloads that touch page tables constantly (process creation, `fork`, JIT compilers): each of a guest's routine memory-management operations becomes a trap.

## 6.3 Nested paging: EPT / NPT (the hardware way, 2008+)

Intel **EPT** (Extended Page Tables) and AMD **NPT/RVI** add a *second* set of page tables walked by the MMU itself:

- Guest page tables (GVA→GPA) — owned by the guest, **no traps needed**.
- EPT (GPA→HPA) — owned by the hypervisor.

On a TLB miss, the hardware performs a **2-D page walk**: every step of the guest walk must itself be translated through EPT. A 4-level guest walk × 4-level EPT walk can touch up to ~24 memory locations (vs. 4 natively). The costs and mitigations:

- **Cost**: TLB misses are several times more expensive. **Mitigation**: big TLBs, paging-structure caches, and **huge pages** (2 MB/1 GB) on *both* levels — huge pages matter far more inside VMs than on bare metal.
- **Win**: page-table-heavy workloads run at near-native speed; memory-management VM exits mostly disappear. EPT was the single biggest virtualization performance improvement ever shipped.

VPIDs/ASIDs tag TLB entries per-VM so world switches don't flush the TLB.

## 6.4 Memory overcommit and reclamation

Hosts routinely promise guests more RAM in total than physically exists. Techniques to make that survivable:

1. **Demand paging / thin allocation** — a guest gets host RAM only for pages it actually touches.
2. **Ballooning** — a driver *inside* the guest (the "balloon") allocates and pins guest pages on hypervisor request, then hands them back to the host. Brilliance: the *guest's own OS* decides which pages it can spare (it swaps/drops caches using its own knowledge), instead of the hypervisor guessing blindly. Deflate the balloon to return memory.
3. **Page sharing / deduplication (KSM, TPS)** — scan for identical pages across VMs (twenty Ubuntu VMs share one copy of libc), map them copy-on-write. (Side-channel caveat: dedup timing attacks — Part 15.)
4. **Host-level swap** — the hypervisor pages guest RAM to disk. Last resort: the guest's LRU knowledge is invisible to the host ("double paging" pathology — the guest may swap the very pages the host already swapped, causing disk I/O storms).

---

# Part 7 — I/O Virtualization

CPU and memory virtualization became nearly free; **I/O is where the overhead lives**, and each approach trades performance against flexibility.

## 7.1 Full device emulation

The hypervisor implements, in software, a *real* device — e.g., an Intel e1000 NIC or an IDE disk — bit-compatible with the datasheet. The guest's unmodified stock driver works out of the box.

Cost: the guest driver bangs on virtual device registers; **every register access is a VM exit**. Sending one packet through an emulated e1000 can take thousands of exits' worth of work. Use: installers, ancient OSes, first boot before better drivers load.

## 7.2 Paravirtual devices (virtio et al.)

Stop pretending. Define an *idealized* device interface designed for the virtual world:

- **virtio** (the open standard: virtio-net, virtio-blk, virtio-scsi, vsock, virtio-gpu...), VMware's vmxnet3/pvscsi, Hyper-V's VMBus synthetic devices.
- The guest loads a special driver that knows it's in a VM. Driver and hypervisor communicate through **shared-memory ring buffers (virtqueues)**: the guest posts buffer descriptors into the ring, kicks the host once (one exit — or none, if the host is polling), and the host processes *batches* of requests. Completion comes back via the ring plus an interrupt.

```
guest driver           shared ring (guest RAM)           host backend
  produce ────────►  [desc][desc][desc][desc...]  ────►  consume batch
  one doorbell/batch                                     one interrupt/batch
```

This is the standard fast path for cloud VMs. Refinements: **vhost** moves the backend into the host kernel (skip QEMU); **vhost-user** moves it into a userspace dataplane (DPDK/OVS) for millions of packets/sec.

## 7.3 Device passthrough (VFIO)

Give a VM the *actual PCIe device* — the guest driver programs real hardware directly, with zero hypervisor involvement in the data path. Native performance; used for GPUs, NVMe, high-end NICs.

The enabler is the **IOMMU** (Intel VT-d / AMD-Vi): the device does DMA using *guest* physical addresses, and the IOMMU translates GPA→HPA and **blocks DMA outside the VM's memory**. Without an IOMMU, a passed-through device could DMA anywhere in host RAM — game over for isolation. The IOMMU is to devices what the MMU is to the CPU.

Trade-off: the device is monopolized by one VM, and live migration becomes hard (physical device state can't be snapshotted generically).

## 7.4 SR-IOV — passthrough for many VMs

**Single-Root I/O Virtualization**: the PCIe device *itself* is virtualization-aware. One **Physical Function (PF)** managed by the host spawns many lightweight **Virtual Functions (VFs)** — each a real PCIe function with its own queues, registers, and DMA — passed through to different VMs. The NIC's internal switch demultiplexes traffic in hardware. Near-native performance, dozens of VMs per device.

The endpoint of this trend: **AWS Nitro** — offload *all* of I/O virtualization (network, storage, security) onto dedicated hardware cards, so the host CPU runs a near-invisible sliver of hypervisor and ~100% of the machine is sellable to guests.

## 7.5 GPU virtualization (a zoo of its own)

- **API remoting** — forward OpenGL/DirectX calls to the host (VirtualBox/VMware 3D, virtio-gpu/Venus).
- **Full passthrough** — one GPU, one VM (gamers' and AI trainers' choice).
- **Mediated / partitioned** — SR-IOV-like slicing (NVIDIA vGPU, Intel SR-IOV graphics), or at a higher level, **NVIDIA MIG** partitioning a datacenter GPU into hardware-isolated instances.

---

# Part 8 — Paravirtualization

## 8.1 The idea

Full virtualization asks: "how do we fool an unmodified OS?" Paravirtualization (Xen, 2003) asks: **"what if the guest OS knows and cooperates?"**

Port the guest kernel to a modified architecture interface: instead of executing sensitive instructions (which trap expensively or fail silently), the guest makes explicit **hypercalls** — system calls into the hypervisor ("please update this page-table entry", "please flush these TLB entries", *batched*). One hypercall replaces dozens of traps.

## 8.2 The trade and the verdict

- **Win**: on pre-VT-x hardware, dramatically faster than binary translation for kernel-heavy work; this made Xen and early EC2 possible.
- **Cost**: you must modify the guest kernel — fine for Linux, impossible for closed-source Windows.
- **Verdict of history**: hardware assist (VT-x + EPT) made *CPU and memory* paravirtualization unnecessary. But paravirtualization **won completely at the I/O and services layer**: virtio drivers, balloon drivers, paravirtual clocks, and "enlightenments" (Windows' term for its Hyper-V-aware optimizations) are in every serious guest today. Modern VMs are full-virtualized CPUs with paravirtualized everything-else.

---

# Part 9 — OS-Level Virtualization: Containers

## 9.1 A different axis entirely

VMs virtualize **hardware**; each guest brings its own kernel. Containers virtualize **the operating system**: many isolated userspaces share **one kernel**.

```
   VMs                                Containers
+------+ +------+                  +------+ +------+ +------+
| apps | | apps |                  | apps | | apps | | apps |
| OS   | | OS   |  ← n kernels     +------+-+------+-+------+
+------+ +------+                  |   one shared host kernel |
|  hypervisor  |                   +--------------------------+
+--------------+
```

A container is **not** a VM and not a single kernel object either — it's a *process group* wrapped in several kernel isolation features:

## 9.2 The Linux building blocks

1. **Namespaces** — per-group *views* of kernel resources. Seven-plus kinds: `pid` (own process tree, its init is PID 1), `net` (own interfaces, routes, ports), `mnt` (own filesystem tree), `uts` (hostname), `ipc`, `user` (UID 0 inside maps to unprivileged UID outside — the key to rootless containers), `cgroup`, `time`.
2. **cgroups** — resource *limits and accounting* per group: CPU shares/quotas, memory ceilings (exceed → OOM-kill inside the group), block I/O and network weights, PID counts. Namespaces control what you can *see*; cgroups control what you can *use*.
3. **Layered filesystems (OverlayFS)** — an image is a stack of read-only layers plus one writable layer per container (copy-on-write). This — not isolation — is what Docker really added: **the image as a shippable, layered, content-addressed artifact**.
4. **Hardening extras** — capabilities (fine-grained root privileges, mostly dropped), seccomp (syscall filter), LSMs (SELinux/AppArmor).

Lineage: `chroot` (1979) → FreeBSD jails (2000) → Solaris Zones (2004) → Linux namespaces+cgroups (2006–2013) → Docker (2013) → OCI standards + Kubernetes.

## 9.3 VMs vs. containers — the real comparison

| Dimension | VM | Container |
|---|---|---|
| Isolation boundary | Hardware interface (VMCS/EPT); tiny attack surface | The ~400-syscall kernel interface; one kernel bug can be a full escape |
| Kernel | Own per guest (any OS, any version) | Shared host kernel (Linux containers need a Linux kernel) |
| Startup | Seconds (OS boot) | Milliseconds (it's a `fork`+`exec`) |
| Memory overhead | 100s of MB (guest kernel + duplicated caches) | ~zero beyond the app |
| Density | Tens per host | Hundreds to thousands |
| Migration | Live migration is mature | Checkpoint/restore (CRIU) exists but is niche |

Rule of thumb: **containers are a packaging and density technology; VMs are an isolation technology.** Clouds run *your* containers inside *their* VMs — both layers, each doing its job. (Windows note: Docker on Windows/macOS runs a Linux VM under the hood — WSL2 is itself a lightweight Hyper-V VM.)

---

# Part 10 — The Middle Ground: MicroVMs and Sandboxes

The 2015+ question: containers' speed with VMs' isolation?

- **Firecracker** (AWS, powers Lambda & Fargate) — a minimal VMM in ~50k lines of Rust on KVM. No BIOS, no PCI, no VGA — just virtio-net/block/vsock and a stripped kernel boot: **~125 ms to a running guest, ~5 MB overhead, thousands per host**. The insight: most of a VM's weight was *legacy PC emulation*, not virtualization itself. (Same niche: Intel's Cloud Hypervisor, Apple's Virtualization.framework, QEMU microvm.)
- **Kata Containers** — every container (or pod) transparently gets its own microVM; keeps the Docker/K8s workflow, upgrades the boundary to hardware.
- **gVisor** (Google) — a third design: a *user-space kernel* (in Go) that intercepts the container's syscalls and services them itself, touching the host kernel through a tiny filtered surface. Isolation without hardware virtualization, at the cost of syscall-heavy performance.
- **Unikernels** (MirageOS, OSv) — compile the *application and a library OS into one image* that boots directly on the hypervisor: single address space, no processes, boots in ms, tiny attack surface. Intellectually elegant; niche in practice (debuggability, ecosystem), but their DNA is visible in Firecracker-style minimalism.

---

# Part 11 — Storage and Network Virtualization

## 11.1 Virtual disks

A guest's disk is typically a file (or LVM/Ceph volume) on the host:

- **Formats**: raw (a flat file, fastest), **qcow2** (QEMU: thin-provisioned, snapshots via internal COW, backing-file chains), VMDK, VHDX.
- **Thin provisioning**: a "100 GB" disk occupies only what's written. Overcommit warning: if every guest fills up simultaneously, the host pool exhausts — monitoring is mandatory.
- **Snapshots & backing chains**: freeze a base image read-only, write changes to an overlay. Cloning 100 VMs from one template costs almost nothing until they diverge. Long snapshot chains degrade reads (each miss walks the chain).
- **The full I/O path is layered caching all the way down** (guest page cache → virtio → host page cache → RAID/SSD cache), which is why *write-safety settings matter*: `cache=none` + guest-flushes-honored is the standard for data integrity; `cache=unsafe` benchmarks beautifully and eats databases in power failures.

## 11.2 Virtual networking

- **vSwitch**: the hypervisor runs a software L2 switch (Linux bridge, Open vSwitch, VMware vSwitch); each VM's virtual NIC plugs into a virtual port. Modes: **bridged** (VM appears on the physical LAN), **NAT** (host masquerades), **host-only/internal**.
- **Overlay networks**: in a datacenter, VMs must migrate across racks while keeping their IPs, and tenants need isolated address spaces at a scale VLANs (12-bit IDs = 4096 max) can't offer. Answer: **encapsulation** — **VXLAN**/Geneve wrap tenant L2 frames in UDP over the physical L3 fabric (24-bit IDs = 16M networks). The physical network sees only host-to-host UDP.
- **SDN (Software-Defined Networking)**: a logically centralized controller programs all the vSwitches — tenant topologies, ACLs, load balancing become *software objects* (NSX, OVN, and every cloud VPC). The cloud "VPC" you configure with API calls is exactly this: a virtualized network whose switches are software and whose wires are encapsulation.
- **NFV**: routers, firewalls, and load balancers themselves shipped as VMs/containers instead of appliance boxes — telcos' 5G cores run this way.

---

# Part 12 — Live Migration

Moving a **running** VM between hosts with sub-second interruption — virtualization's signature magic trick, and routine ops in every datacenter (evacuate a host for maintenance, rebalance load).

## 12.1 Pre-copy (the standard algorithm)

1. **Iterative copy**: while the VM keeps running on host A, copy all its RAM to host B. Track pages the guest **dirties** during the copy (via EPT write-protection or hardware dirty logging — e.g., Intel PML). Re-send dirty pages. Repeat: each round should shrink, because each round is faster than the last.
2. **Stop-and-copy**: when the remaining dirty set is tiny (or max rounds hit), pause the VM, send the final dirty pages + vCPU/device state (milliseconds).
3. **Resume on B**; send a gratuitous ARP so the network learns the VM's new location (same IP — overlay networking makes this trivial).

**Failure mode**: a write-heavy guest dirties memory faster than the network copies it — convergence never happens. Countermeasures: **auto-converge** (deliberately throttle the guest's vCPUs to slow dirtying), compression/zero-page detection, or switch strategies:

## 12.2 Post-copy

Move vCPU state *first*, run the VM on B immediately, and fetch each memory page **on demand from A** at first touch (like network-backed demand paging), with background pre-push. Guaranteed convergence and low downtime; the cost is degraded performance during the fill and a hard new failure mode — if A dies mid-migration, the VM's memory is split across two hosts and it's dead. Production systems often start pre-copy and flip to post-copy if convergence stalls.

Related: **snapshot/suspend-resume** (same machinery, state to disk), and **fault-tolerance mirroring** (continuous checkpointing to a shadow VM — COLO, VMware FT).

---

# Part 13 — Nested Virtualization

A hypervisor inside a VM: the guest runs its *own* guests. Why: developing hypervisors, CI for virtualization code, running WSL2/Hyper-V features inside a cloud VM, security products.

Mechanics (L0 = real hypervisor, L1 = guest hypervisor, L2 = its guest): hardware has only one non-root mode, so **L0 must emulate VT-x for L1**. When L1 executes `VMLAUNCH`, that instruction itself traps to L0, which builds a *merged* "shadow VMCS" and runs L2 directly on the hardware. Every exit L2 causes goes to **L0 first**, which decides: handle it, or reflect it to L1 (expensive — an exit becomes several). Same story for page tables: L0 composes ("shadows") L1's EPT. Hardware keeps absorbing pieces of this (VMCS shadowing, Enhanced VMCS); the cloud offers it routinely now, at a real but shrinking performance tax.

---

# Part 14 — Virtualization and the Cloud

The cloud is virtualization industrialized:

- **IaaS = VMs as an API.** An EC2/Azure/GCE "instance" is a VM whose creation, sizing, disks (virtual), and networks (overlay/SDN) are API calls. Multi-tenancy — strangers' workloads on shared silicon — is *only* possible because the hypervisor boundary is strong.
- **The economics**: consolidation ratios, overcommit (CPU heavily, memory carefully), burstable instances (cgroup/scheduler credits), spot instances (your VM is preemptible), all priced against the hypervisor's ability to pack and migrate.
- **The Nitro pattern** (Part 7.4): push virtualization into hardware until the host tax → 0 and "bare-metal instances" (you rent the whole machine, Nitro cards still enforce the platform) become possible.
- **Higher layers re-virtualize**: Kubernetes schedules containers *across* fleets of VMs; serverless (Lambda) runs functions in per-invocation microVMs; "container-native" clouds (Fargate, Cloud Run) hide the VM layer entirely but it's still there.
- **Live migration at fleet scale**: GCP transparently live-migrates customer VMs for host maintenance — thousands of migrations a day nobody notices.

Understand virtualization and cloud pricing sheets, instance-type zoos, and outage postmortems all become legible.

---

# Part 15 — Security

## 15.1 The good: virtualization as a security boundary

The hypervisor interface is *small* (a few dozen exit reasons, a few hypercalls) versus the kernel's syscall surface (~400 calls, thousands of ioctls). Smaller interface → fewer bugs → stronger boundary. Hence: multi-tenant clouds trust it; malware analysts detonate samples in VMs; Qubes OS runs your browser, email, and banking in separate VMs; Windows VBS runs credential storage in a Hyper-V-protected enclave *under* the OS.

## 15.2 The bad: attacks on the virtualization stack

- **VM escape** — guest code exploits a hypervisor/device-model bug to run on the host. Historically, escapes overwhelmingly hit the **device emulation code** (the biggest, crustiest attack surface): VENOM (2015, QEMU's *floppy controller* — present even when unused), Cloudburst (VMware display), many virtio/USB/network-card CVEs. This is exactly why Firecracker exists (Part 10): delete the devices, delete the bugs.
- **Side channels across VMs** — co-resident guests share caches, branch predictors, DRAM:
  - **Spectre/Meltdown/L1TF/MDS (2018–19)** — speculative execution leaks crossing the VM boundary; L1TF could read the L1 cache across VMs, forcing clouds to stop co-scheduling different tenants on SMT siblings, add flush-on-entry mitigations, and eat double-digit performance costs. The largest security event in virtualization's history.
  - Cache-timing (PRIME+PROBE) key extraction, Rowhammer bit-flips across VM boundaries, memory-dedup timing probes (why cross-tenant KSM is off in public clouds).
- **Management plane** — in practice, most real-world cloud breaches hit orchestration APIs, credentials, and images, not the hypervisor.

## 15.3 The frontier: confidential computing — removing the host from the TCB

Classic virtualization: the guest trusts the hypervisor totally (it reads guest RAM at will). **Confidential computing inverts this**: protect the guest *from* the host.

- **AMD SEV-SNP / Intel TDX / ARM CCA**: guest memory is encrypted with keys the hypervisor never sees, held in hardware; integrity protection stops replay/remap tricks; the hypervisor still *schedules* the VM but cannot *read* it.
- **Remote attestation**: the CPU signs a measurement of the launched guest, so you can cryptographically verify *what* is running *on genuine hardware* before sending it secrets.
- Changes the cloud trust model fundamentally: you no longer have to trust the cloud operator's software stack (or its insiders) with your data-in-use. All major clouds now sell confidential VMs.

## 15.4 Detection and anti-detection

VMs are detectable (CPUID hypervisor bit and vendor leaf, device names, timing anomalies) — which malware uses to evade sandboxes, spawning an arms race of hypervisors that hide and malware that probes. Red-pill/blue-pill history: hypervisor-based rootkits (SubVirt, Blue Pill) that slide *under* a running OS.

---

# Part 16 — Performance: How Experts Think

## 16.1 The mental model

**Overhead = exits × cost-per-exit + steal + memory-translation tax + I/O path length.** Modern CPU-bound guest code runs at 97–100% of native. The gaps appear in: transitions (exits), contention (overcommit), TLB pressure (2-D walks), and I/O software paths.

## 16.2 The diagnostic loop

1. **Measure exits first**: `perf kvm stat live` on a KVM host shows exit reasons ranked. A storm of `EXTERNAL_INTERRUPT`/`MSR_WRITE`/`IO_INSTRUCTION` exits points at interrupt or device configuration; `EPT_VIOLATION` storms point at memory.
2. **Check steal time** (`%st` in guest `top`/`vmstat`): the fraction of time vCPUs were runnable but the host ran someone else. High steal = host CPU overcommit — no amount of in-guest tuning fixes it.
3. **Memory**: huge pages on host *and* guest (transparent or explicit); watch for host swapping of guest RAM (catastrophic); NUMA — pin a VM's vCPUs and memory to one node, or expose virtual NUMA to big guests; balloon pressure.
4. **I/O path**: emulated → virtio → vhost → vhost-user/DPDK → SR-IOV/passthrough is the escalation ladder; multi-queue virtio to spread across vCPUs; `cache=none` + native AIO/io_uring for disks.
5. **Topology honesty**: don't give a latency-sensitive VM 32 vCPUs on a busy 16-core host. Gang-scheduling pathologies (a spinlock holder's vCPU is descheduled → all sibling vCPUs spin — "lock-holder preemption") are why hypervisors have pause-loop exiting and why *fewer, honest* vCPUs often beat more.

## 16.3 Benchmarks lie unless

...you run them at realistic *density* (one VM on an idle host tells you nothing about 40), measure *tail* latency not throughput, and match cache/write-barrier settings to production. The classic trap: `cache=unsafe` disk benchmarks "proving" virtualization is fast.

---

# Part 17 — Hands-On Lab

Reading about hypervisors teaches you *about* hypervisors. Running and inspecting them teaches you hypervisors.

## 17.1 Check your hardware (any machine)

```powershell
# Windows: is a hypervisor present / is virtualization enabled?
systeminfo | Select-String "Hyper-V"
Get-ComputerInfo -Property HyperV*
```

```bash
# Linux: CPU virtualization flags (vmx = Intel VT-x, svm = AMD-V)
grep -Eoc '(vmx|svm)' /proc/cpuinfo     # >0 means supported
lscpu | grep -i virtualization
# Am I *inside* a VM right now?
systemd-detect-virt                      # kvm, microsoft, vmware, none...
```

## 17.2 Windows: Hyper-V and WSL2

```powershell
# Enable Hyper-V (admin PowerShell, then reboot)
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All

# Create a VM in five lines
New-VM -Name lab -MemoryStartupBytes 2GB -Generation 2 `
       -NewVHDPath "$env:USERPROFILE\lab.vhdx" -NewVHDSizeBytes 20GB
Set-VMProcessor lab -Count 2
Add-VMDvdDrive -VMName lab -Path C:\isos\ubuntu-24.04-live-server-amd64.iso
Start-VM lab

# Realize WSL2 is a VM: this is a Hyper-V utility VM booting a real Linux kernel
wsl --status
```

## 17.3 Linux: KVM/QEMU from first principles

```bash
sudo apt install qemu-kvm libvirt-daemon-system virtinst   # Debian/Ubuntu
lsmod | grep kvm                    # kvm_intel or kvm_amd loaded?

# A VM with no libvirt, no XML — just QEMU, to see the moving parts:
qemu-img create -f qcow2 disk.qcow2 10G
qemu-system-x86_64 \
  -enable-kvm -cpu host -smp 2 -m 2G \
  -drive file=disk.qcow2,if=virtio,cache=none \
  -netdev user,id=n0 -device virtio-net-pci,netdev=n0 \
  -cdrom ubuntu-24.04-live-server-amd64.iso

# Watch VM exits live while the guest runs (Part 16.2 in action):
sudo perf kvm stat live
```

## 17.4 Build a container from raw syscalls (no Docker)

```bash
# One command: new hostname/PID/mount namespaces + a private proc
sudo unshare --uts --pid --mount --fork bash -c '
  hostname container1
  mount -t proc proc /proc
  hostname; ps aux'          # ps shows ~2 processes; you are PID 1

# Cap it with cgroups v2
sudo mkdir /sys/fs/cgroup/demo
echo "50000 100000" | sudo tee /sys/fs/cgroup/demo/cpu.max   # 0.5 CPU
echo $$ | sudo tee /sys/fs/cgroup/demo/cgroup.procs
```

Then, for real depth: write a mini-container in ~100 lines of Go/C (`clone()` with `CLONE_NEWNS|CLONE_NEWPID|...`, `pivot_root`, exec) — the classic exercise "containers from scratch."

## 17.5 Things worth doing once

- Boot a **Firecracker** microVM from its getting-started guide; time it.
- Live-migrate a VM between two hosts (`virsh migrate --live`); ping it during migration and count lost packets (usually 0–1).
- Read a **VM-exit reason table** (Intel SDM vol. 3C, Appendix C) — skim it once so exit costs stop being abstract.
- Take a qcow2 snapshot chain apart with `qemu-img info --backing-chain`.

---

# Part 18 — The Road to Expertise

## 18.1 The layered summary — one picture

```
 Language VMs (JVM, Wasm)        ← virtual ISAs (different topic, same word)
 ────────────────────────────
 Containers / microVMs           ← virtualize the OS / minimize the VM
 Guest OS                        ← thinks it owns hardware
 Paravirtual drivers (virtio)    ← guest cooperates for fast I/O
 Hypervisor (KVM/Hyper-V/ESXi)   ← trap-and-emulate + device models
 HW assists (VT-x, EPT, IOMMU,   ← make each layer's illusion cheap
             SR-IOV, SEV/TDX)
 Physical machine
```

Every layer is the same move: **indirection for isolation, then hardware/protocol co-design to erase the cost of the indirection.**

## 18.2 What experts actually know

1. The **exit model** cold: what causes VM exits, roughly what they cost, how to observe and reduce them.
2. The **three-address-level memory model** (GVA/GPA/HPA) and why huge pages and NUMA dominate VM memory performance.
3. The **I/O escalation ladder** (emulated → virtio → vhost → SR-IOV/passthrough) and where each belongs.
4. That **isolation strength and density trade off**, and where each point on the spectrum (process → container → gVisor → microVM → VM → physical) is the right answer.
5. That the **management plane, not the hypervisor, is usually what gets breached**, while side channels are what keeps hypervisor engineers up at night.

## 18.3 Reading list

- **Popek & Goldberg**, "Formal Requirements for Virtualizable Third Generation Architectures" (CACM 1974) — short, foundational.
- **Adams & Agesen** (VMware), "A Comparison of Software and Hardware Techniques for x86 Virtualization" (ASPLOS 2006) — binary translation vs. first-gen VT-x, beautifully written.
- **Barham et al.**, "Xen and the Art of Virtualization" (SOSP 2003) — the paravirtualization paper.
- **Bugnion, Nieh, Tsafrir**, *Hardware and Software Support for Virtualization* (Morgan & Claypool) — the best single book on the mechanisms.
- **Agache et al.**, "Firecracker: Lightweight Virtualization for Serverless Applications" (NSDI 2020).
- **Waldspurger**, "Memory Resource Management in VMware ESX Server" (OSDI 2002) — ballooning and page sharing.
- **Intel SDM Volume 3C** (VMX) — the ground truth, surprisingly readable in targeted doses.
- *Arpaci-Dusseau*, **OSTEP**, the VMM appendix — free, gentle, rigorous.

Everything in the modern infrastructure world — clouds, Kubernetes, serverless, confidential computing — stands on the machinery in this document. Learn the illusion, and the whole stack becomes legible.
