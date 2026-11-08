# Mobile device architecture

How does a phone receive a message with its main CPU asleep? Why can a tablet edit high-resolution video smoothly yet reload a browser tab when you switch back to it? Why can a laptop with a similar processor finish a long export sooner? The answers involve different parts of one machine. This guide connects the instructions an app executes to the hardware that senses, computes, displays, stores, transmits, and supplies power.

The examples use a contemporary Arm-based Android phone and an iPhone or iPad where the platform details matter. The comparison machines are a battery-powered laptop and a mains-powered desktop tower. A Windows tablet, a fanless Arm laptop, and a compact desktop are useful boundary cases. Each illustrates which behaviors follow from silicon, which follow from the enclosure, and which follow from operating-system policy.

## Contents

| Inside the device | Across the system |
| --- | --- |
| [1. Whole-device map](#1-whole-device-map) | [11. Power, sleep, and heat](#11-power-sleep-and-heat) |
| [2. Contracts between layers](#2-contracts-between-layers) | [12. Firmware, boot, and updates](#12-firmware-boot-and-updates) |
| [3. CPU execution and scheduling](#3-cpu-execution-and-scheduling) | [13. Kernels, runtimes, and app lifetime](#13-kernels-runtimes-and-app-lifetime) |
| [4. Caches and physical memory](#4-caches-and-physical-memory) | [14. Security and virtualization](#14-security-and-virtualization) |
| [5. Address translation and sharing](#5-address-translation-and-sharing) | [15. End-to-end walkthroughs](#15-end-to-end-walkthroughs) |
| [6. Interconnects, DMA, and device I/O](#6-interconnects-dma-and-device-io) | [16. Quantitative reasoning](#16-quantitative-reasoning) |
| [7. Graphics, touch, and display](#7-graphics-touch-and-display) | [17. Phones, tablets, laptops, and desktops](#17-phones-tablets-laptops-and-desktops) |
| [8. Cameras, audio, and accelerators](#8-cameras-audio-and-accelerators) | [18. Diagnosing a real workload](#18-diagnosing-a-real-workload) |
| [9. Flash storage and durability](#9-flash-storage-and-durability) | [19. Further reading](#19-further-reading) |
| [10. Radios, networking, and sensors](#10-radios-networking-and-sensors) | |

## 1. Whole-device map

A **system on a chip** (SoC) puts several computing engines and the connections between them on one chip. The **application processor** runs the main operating system and apps. Other processors can run their own firmware, keep their own time, and continue working while application CPU cores sleep. The battery, antenna, camera sensor, flash storage, and display panel are physical components elsewhere in the device. Even RAM packaged immediately beside or above the SoC is a distinct component.

~~~text
 Apps                 browser, camera, maps, music player
   |  APIs, runtime libraries, messages to system services
   v
 OS                   scheduling, memory, permissions, files, network
   |  kernel drivers submit work and map buffers       ^ completions
   v                                                    |
 +---------------------- application SoC ------------------------+
 | CPU clusters       GPU          media codec       neural unit |
 | private caches     display      camera ISP        audio DSP   |
 |        \              |             |                 /       |
 |            on-chip interconnect and memory controllers        |
 +---------+-----------------+-----------------------+-----------+
           |                 |                       |
        LPDDR RAM      storage controller       peripheral links
    code, pages,       and managed flash        touch, cameras, panel,
    shared buffers     apps and saved data      microphones, USB

 Application processor <-> modem/baseband <-> RF circuits <-> antennas
                       <-> Wi-Fi / Bluetooth
                       <-> always-on controller <-> sensors

 Battery or charger -> power-management circuits -> separate power rails
 Boot ROM and firmware establish trust, initialize hardware, and manage
 selected functions throughout operation.
~~~

The arrows describe interfaces. An image sensor might send pixels over a dedicated camera link to the ISP while a slower control bus carries exposure settings. Both belong to the camera, but a bus trace of exposure commands will never contain the full preview image. Similarly, a request to a modem can be small even when the resulting network transfer is large. To trace an operation, follow both the **control path**, which requests work, and the **data path**, which carries its payload.

| Hardware block | Work it owns | Constraint visible to software |
| --- | --- | --- |
| CPU clusters | Branching code, system calls, coordination | A runnable thread needs a suitable core and time on it. |
| GPU and display controller | Draw pixels, combine layers, scan buffers to the panel | Rendering and presenting have separate queues and deadlines. |
| ISP and codec | Process sensor samples, encode or decode supported media | Image format, resolution, and supported profiles determine feasible paths. |
| DSP and neural accelerator | Repeated signal or tensor operations | Setup, data conversion, and unsupported operations can dominate short jobs. |
| Modem and radio front end | Maintain cellular service and transmit over the air | Signal conditions, airtime, and radio power states affect latency and energy. |
| RAM and persistent storage | Keep active pages accessible and saved files durable | Capacity, bandwidth, and write completion answer different questions. |
| Power-management IC (PMIC) | Convert battery voltage into supply rails and sequence them | A sleeping subsystem may need clocks, power, and firmware restored before use. |

**A desktop analogy.** Picture a tower with a CPU, graphics card, SSD, network adapter, and power supply. A mobile SoC brings many of their functions closer together. Drivers, command queues, memory protection, and completion events still exist. Integration changes the cost of moving bytes and the number of components that share the same power and cooling budget. One implementation with identifiable CPU, GPU, ISP, and modem blocks is Qualcomm's Snapdragon platform. [Qualcomm mobile platform overview](https://www.qualcomm.com/smartphones/products/8-series/snapdragon-8-gen-3-mobile-platform)

## 2. Contracts between layers

Suppose a camera app requests a preview. It needs a stream with a size, pixel format, and lifetime. It should not need to know which voltage brings a particular image sensor out of standby. Those details sit on opposite sides of an interface. An engineer diagnosing a black preview has to find the first interface whose promise failed.

| Interface | What the caller can rely on | Example failure |
| --- | --- | --- |
| Framework API | Services exposed by the operating system, including permission checks and asynchronous results | The user revoked camera permission after an earlier launch. |
| Interprocess communication (IPC) | A way to identify a service, send arguments or handles, and receive a reply | The camera service died while the app still held its handle. |
| Hardware abstraction layer (HAL) | A platform-defined contract implemented by device-specific software | A particular stream combination exceeds the hardware's supported configuration. |
| System call and driver interface | A controlled transition into the kernel to map memory, wait for an event, or submit I/O | A request's buffer is inaccessible or has already been released. |
| Application binary interface (ABI) | Calling convention, register use, executable conventions, and data layout | A native library targets a different platform ABI. |
| Instruction set architecture (ISA) | Instructions, registers, exceptions, and memory-ordering rules available to software | Code executes an optional instruction without checking CPU support. |

**A call can cross several boundaries.** A framework method may run entirely in the calling process, make a system call, or send IPC to a service that itself uses a driver. In Android, Binder can transfer a reference to a service or a file descriptor to an authorized process. A preview request can travel as a small Binder message while image data lives in separately allocated buffers. A successful reply accepting a request can precede both hardware completion and the first visible pixel. [Android platform architecture](https://developer.android.com/guide/platform) · [Binder overview](https://source.android.com/docs/core/architecture/ipc/binder-overview)

**What carries over to a PC?** An Arm phone and an Arm laptop may both execute an integer addition from the same ISA. A compiled application also depends on its executable format, libraries, ABI, operating-system services, and signing policy. A Linux desktop program that opens a video device directly and an Android app that requests the system camera service encounter different permission boundaries. Windows on Arm adds another possibility, translation of supported binaries compiled for a different ISA. [Microsoft's Windows on Arm overview](https://learn.microsoft.com/en-us/windows/arm/overview)

## 3. CPU execution and scheduling

### From instructions to retired work

A high-performance core fetches instructions from the predicted instruction stream, decodes them, and tracks dependencies. Register renaming lets two instructions that happen to use the same architectural register name occupy different physical registers when their values are independent. A scheduler inside the core can issue ready operations to integer, floating-point, vector, or load/store units while an older load waits. Retirement commits architectural results in program order. Implementations differ in width, queue sizes, branch prediction, and the amount of speculation they can support.

~~~text
 predict and fetch -> decode and rename -> ready operations -> execute
                            |                    |                |
                            +----- track dependencies              v
                                                retire in order <-+
~~~

For a pointer-linked list, load B needs the address returned by load A. An aggressive core can do other independent work during the wait, but it cannot discover B's address early through ordinary parallel issue. Traversing a contiguous array gives hardware prefetch and vector instructions more opportunities. A faster clock helps most when the stalled operation is itself shortened by that clock. Time waiting for DRAM or a reply from another process can stay approximately constant in nanoseconds.

### Cores with different capacities

Many mobile chips have cores or clusters tuned for different combinations of speed, leakage, and power. A high-capacity core can shorten the serial part of opening an app. A smaller core may handle light background work using less energy. A scheduler considers which threads are runnable, their priorities, estimated utilization, CPU capacity, and available choices. Placing a task on an already-active cluster can avoid waking another one. Migrating it can leave useful cache entries behind. The decision changes as temperature, load, and permitted frequencies change. [Linux energy-aware scheduling](https://docs.kernel.org/scheduler/sched-energy.html)

An important limit is a **serial dependency**. If a UI thread blocks on a database lock held by a worker, adding idle cores does little until the worker runs and releases the lock. A trace should distinguish a running thread from a runnable thread waiting for CPU time and a blocked thread waiting for a lock, file, or device. All three can produce the same visible pause. Raising the priority of the blocked thread alone cannot make the lock holder finish.

### Privilege and exceptions

In a common Arm application environment, user code executes at EL0 and the OS kernel at EL1. A system-call instruction such as `SVC` requests a synchronous exception. The processor takes the exception on the current core, saves return state as defined by the architecture, and transfers execution to privileged handler code. The kernel checks arguments and permissions before acting. An interrupt is asynchronous to the interrupted instruction stream. A timer, device completion, or wake event can make a thread runnable later. EL2 can host a hypervisor. Firmware roles and available exception levels depend on the implementation. [Arm exception model](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/Exception%20model.pdf)

The same reasoning applies to laptop and desktop CPUs. Modern PC chips can also have different core types and boost frequencies. What often differs is how long the machine can power the fast cores at a high operating point while meeting enclosure temperature limits. Compare the first second of a task with its steady state.

## 4. Caches and physical memory

### Follow a cache miss

Registers hold the operands a core uses immediately. Small private caches hold recently used instruction and data lines. A miss can proceed to another cache, a shared cache if the design has one, or ultimately the memory controller and low-power DRAM (LPDDR). Some SoCs also provide a system-level cache accessible to several agents. “Shared memory” says little about which caches a particular agent uses. Sizes and inclusion rules are properties of the specific chip.

The controller schedules transactions across channels and DRAM banks. Access to an already-open row can reuse work that opening another row would require. Refresh, writes waiting to drain, and requests from other agents compete for service. A streaming GPU workload can therefore slow CPU cache misses without changing CPU frequency. Latency describes the wait for one dependent read. Bandwidth describes the amount of data that many outstanding operations can deliver per second.

### Capacity, traffic, and energy

A 12-megapixel image decoded into four bytes per pixel occupies about 48 MB before row padding. Keeping the original, a working copy, and an undo copy uses about 144 MB. The compressed file on flash might be only a few megabytes. An editor that decodes a gallery of thumbnails at full resolution can exhaust memory while storage space remains plentiful. Decode close to the display size or use tiles when the operation can be performed locally.

Reading that 48 MB buffer once and writing a new 48 MB buffer once generates 96 MB of logical transfers. Repeating the operation ten times creates 960 MB of logical traffic. Caches, compression, and tile-local storage determine how much reaches external RAM. Fusing two passes can save an intermediate write and read. A fused kernel can also increase register pressure or prevent reuse, so count transfers at the level that limits the workload.

### What does unified memory buy?

A CPU and integrated GPU can access buffers backed by the same physical memory when the platform API permits it. This can remove a transfer between host memory and dedicated graphics memory. The allocation still has a format, ownership, lifetime, and synchronization rules. A tiled GPU surface can require conversion before a CPU library can address its rows. The conversion may cost more than the computation the library performs.

A discrete desktop GPU typically has its own high-bandwidth graphics memory. Uploading a scene once and drawing it many times amortizes the transfer. Sending every intermediate image back across the host link can reverse that advantage. Integrated laptop graphics and Apple silicon Macs offer shared-memory arrangements familiar from phones. Memory topology is more predictive here than the name on the enclosure. [Apple on GPU memory and image processing](https://developer.apple.com/videos/play/wwdc2021/10153/)

## 5. Address translation and sharing

### What a virtual address means

Each process gets a virtual address space. The memory management unit (MMU) translates a virtual page number to a physical frame and checks access permissions. A page offset selects the byte inside the frame. A translation lookaside buffer (TLB) caches recent translations. On a TLB miss the hardware may find a valid mapping by walking page tables. A page fault arises when the translation or access cannot complete under the current tables. A TLB miss with a valid mapping usually completes without entering the kernel. [Arm memory management guide](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/LearnTheArchitecture-MemoryManagement-101811_0100_00_en.pdf)

For a 4 KiB page, the offset occupies 12 bits. For a 16 KiB page it occupies 14 bits. A library that rounds a mapping down by clearing only 12 bits can pass an invalid address on a 16 KiB system. Larger pages can increase the memory reachable through a given number of TLB entries, but they also change allocation granularity and alignment requirements. Native Android software should obtain the runtime page size and satisfy the target's binary alignment requirements. [Android page-size support](https://developer.android.com/guide/practices/page-sizes)

### Reclaim and first touch

A process can reserve an address range before it needs physical frames. On the first write to a fresh anonymous page, a fault can cause the kernel to supply a zero-initialized page and install a writable mapping. For a file mapping, a clean page already in the page cache can satisfy a fault. A cold file page can require storage I/O. Both cases resume at the faulting instruction after successful resolution, yet their costs differ substantially.

When memory gets scarce, the system can discard clean file-backed pages and reload them later. Dirty data needs writeback or other backing before it can be discarded while preserving contents. Some Android configurations use compressed RAM swap (zram), spending CPU time to fit more pages into memory. Under sustained pressure Android can reclaim entire lower-priority processes. A retained screenshot in the app switcher can outlive the process that produced it. [Android memory management](https://developer.android.com/topic/performance/memory-management) · [Android low-memory killer daemon](https://source.android.com/docs/core/perf/lmkd)

### Coherence, ordering, and ownership

**Cache coherence** coordinates copies of the same cache line among participating cores. To write a shared line, a core obtains ownership and other copies may be invalidated. Two threads updating different counters that occupy one line can repeatedly exchange ownership. This *false sharing* is possible even when each thread has exclusive use of its own counter. Separating frequently written data can reduce traffic, at the cost of a larger footprint.

**Memory ordering** concerns the visibility of different operations. Suppose a producer writes ordinary payload data and then publishes a ready flag. A single-producer, single-consumer handoff can use a release store to an atomic flag and an acquire load that observes that store. The consumer can then read the initialized payload, provided the producer does not modify it again while the consumer uses it. The language's synchronization contract matters on Arm and x86 alike. Code with an ordinary data race can fail through compiler transformations even on a strongly ordered CPU. [Arm memory model guide](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/Armv8-A%20memory%20model%20guide.pdf)

CPU atomics alone cannot tell an app that a camera or GPU has finished with a frame. A device may still have queued commands and may participate differently in coherence. The driver and graphics APIs define how to wait for completion, make results visible, and hand the buffer to its next owner. This is the boundary behind a preview image that occasionally shows part of an old frame.

## 6. Interconnects, DMA, and device I/O

### Submitting work to hardware

The on-chip interconnect routes transactions among CPU clusters, accelerators, memory controllers, and peripherals. Its traffic includes ordinary memory accesses, coherence messages where supported, and device-register accesses. A display controller must fetch pixels before scanout consumes them. A storage engine can tolerate short delays if a queue absorbs them. Interconnect arbitration and memory-controller policy determine which request is served first under contention. A benchmark of an isolated CPU cannot expose every interaction during video capture.

A typical queued device operation follows this sequence. Actual queue formats and completion mechanisms vary.

1. A client submits work through a service or driver interface. The driver checks arguments, allocates or imports a buffer, and maintains a reference for the operation's lifetime.
2. The driver establishes a **DMA** (direct memory access) mapping, if required. It supplies an address meaningful to the device. A device-side MMU, often called an IOMMU or Arm SMMU, can translate and restrict these addresses. An app's numeric pointer alone is insufficient.
3. The driver records a command or descriptor and makes it visible to the hardware. Required memory barriers or cache maintenance precede publishing the command. It may write a memory-mapped I/O register to notify the engine.
4. The device reads the descriptor, transfers data to or from mapped pages, and records completion. The CPU can execute other work or sleep during the transfer.
5. An interrupt, polled status, or fence signals progress. The driver completes the operation under the API's visibility rules, wakes a waiter if necessary, and eventually releases the mapping.

A CPU page table protects a process against unauthorized CPU accesses. A device's DMA mapping limits what a bus master can access. Both need the correct lifetime. Freeing and reassigning a page while a device can still write to it can corrupt whichever object next gets that page. A well-designed cancellation path first stops or drains outstanding work, then removes mappings and releases memory.

### Sharing a frame across drivers

~~~text
 camera driver fills buffer A ---- completion ----> GPU may read A
 GPU renders buffer B ------------ completion ----> display may scan B
 display finishes using B -------- release -------> producer may reuse B
~~~

Linux's DMA-BUF mechanism lets one driver export an allocation for another driver to import. A file descriptor can represent the allocation. A graphics buffer can also carry metadata about dimensions, format, stride, and layout. The descriptor is a handle, not an invitation to treat every allocation as a linear array. A completion fence represents progress of work using that buffer. Explicitly passing fences can preserve parallelism because a command can wait on the device timeline while the CPU submits later work. [Linux DMA-BUF documentation](https://www.kernel.org/doc/html/latest/driver-api/dma-buf.html) · [Android graphics synchronization](https://source.android.com/docs/core/graphics/sync)

A desktop PCI Express graphics card or NVMe SSD also consumes commands, performs DMA, and reports completion. On a phone the engines often sit behind internal interconnects with fixed board wiring. An external USB camera on a tablet adds another controller, transport protocol, power demand, and potentially another format conversion. The shape of the connector does not establish which transport speed, display mode, or charging capability the product implements.

## 7. Graphics, touch, and display

### From geometry to a frame

The CPU records draw commands and resource references. The GPU processes vertices, rasterizes primitives into covered samples, runs programmable shading work, and writes the resulting color values. A tile-based renderer divides a surface into small regions so that intermediate color, depth, or stencil data can stay in fast local storage while that tile is being processed. An extra render pass that stores the entire intermediate surface and reloads it later can spend considerable energy on memory traffic. Which passes can be combined depends on their dependencies, the GPU, and its API. [Apple's tile-based rendering guide](https://developer.apple.com/documentation/metal/tailor-your-apps-for-apple-gpus-and-tile-based-deferred-rendering)

The system then **composes** visible layers. A display engine may combine video, UI, or cursor layers directly if their formats, scaling, and transformations are supported. Otherwise the GPU can render layers into a composite image. The display controller fetches the chosen buffers and sends pixels to the panel according to the display's timing. A GPU performance counter says little about a frame still waiting in a presentation queue.

### Queues, timing, and input latency

On Android, a producer can dequeue a graphics buffer, render into it, then queue it for a consumer. SurfaceFlinger composes visible surfaces with help from the hardware composer. Acquire and release synchronization prevents one engine from reading unfinished work or overwriting a surface still on screen. Extra buffers absorb scheduling variation and let stages overlap. When a producer continually runs ahead, queued frames can increase the elapsed time between an input event and the first pixel that reflects it. [Android BufferQueue and Gralloc](https://source.android.com/docs/core/graphics/arch-bq-gralloc) · [Android VSync](https://source.android.com/docs/core/graphics/implement-vsync)

~~~text
 touch sampled -> input delivered -> UI state updated -> rendering submitted
          -> GPU completes -> compositor latches -> panel scans -> pixel changes
~~~

At 120 Hz, refresh opportunities are about 8.33 ms apart. A finger can land just after a touch sample. Its event may wait for UI scheduling, and the rendered frame may just miss the compositor's latch time. Even when the GPU spends only 2 ms drawing, the full **touch-to-photon** delay can span multiple refresh intervals. Variable refresh, scan direction, panel response, and touch sampling cadence affect the result. Refresh rate alone cannot give the delay.

A tablet may have a larger physical panel, more pixels, and pen input. Physical size, pixel count, and refresh rate are independent properties. Doubling both pixel dimensions quadruples pixels per frame. A drawing app may predict stylus movement for immediate feedback, then correct that stroke when actual samples arrive. This trades prediction error for responsiveness. External monitors, scaling, and window resizing can change the work even when the app code stays the same.

A laptop also has a compositor, a scanout engine, and presentation queues. A gaming desktop connected to a high-refresh monitor can face an equally strict frame deadline. The mobile-specific tension is often between low latency, display power, and keeping enough GPU capacity available after the enclosure warms up.

## 8. Cameras, audio, and accelerators

### The camera has two paths

The camera sensor converts light into electrical signals, and its readout produces digital samples. Many sensors use a color filter array. The ISP can correct defective pixels, reconstruct color channels through demosaicing, reduce noise, adjust color and tone, and scale images. The platform can also combine several frames with different exposures. Motion between exposures can cause ghosting unless the processing pipeline can align or reject inconsistent samples. The sequence varies with sensor, capture mode, and implementation.

~~~text
 Control    app -> camera service -> vendor HAL -> exposure, focus, streams
 Results    sensor -> ISP -> preview buffer --------------------> display
                          -> full-resolution or video buffer ---> encoder
 Metadata   timestamps, exposure, focus state -> matching capture result
~~~

In Android's request model, a capture request supplies settings and output targets. Multiple requests can be in flight. Metadata and buffers let the client associate observed exposure and timing with the corresponding capture. A control requested now may affect a later frame because the sensor and processing pipeline already contain work. A viewfinder and saved still can have different resolution, processing, or crop. [Android camera HAL request model](https://source.android.com/docs/core/camera/camera3_requests_hal)

Rolling readout means different rows can represent different instants. A fast pan can bend straight scene features even when every row is individually sharp. A shorter exposure helps motion blur but cannot make the readout simultaneous. Similarly, a brighter preview can come from longer exposure, gain, or combining multiple frames. Each has different consequences for movement, noise, latency, and memory use.

### Codecs and audio deadlines

A video encoder exploits spatial similarity within an image and temporal similarity between frames. A decoder reconstructs images from the bitstream and retained reference frames. A fixed-function block can perform these operations efficiently for supported codecs, profiles, resolutions, and bit depths. If an editor asks for an unsupported combination, software may choose another encoder, convert the input, reduce quality, or reject the configuration. A media workload can move from codec-bound to CPU-bound after changing a single export option.

An audio path can include microphone, analog-to-digital conversion, DSP processing, shared buffers, mixing, and output conversion. It repeatedly consumes a small batch of samples before a deadline. An output underrun occurs when the next batch is unavailable when playback needs it. A larger buffer tolerates more scheduling jitter but increases monitoring latency for a musician. At 48,000 sample frames per second, 240 frames correspond to 5 ms of audio. Input buffering, output buffering, processing, and wireless codec delay can all contribute to round-trip latency. [Android audio latency](https://developer.android.com/ndk/guides/audio/audio-latency)

### Choosing an engine for inference

An NPU accelerates supported neural-network operations using parallel arithmetic and local storage. Software prepares tensors, selects supported data types and layouts, and submits work. A model can be split across engines if some operators lack accelerator support. Crossing that boundary may require synchronization, data layout changes, or conversion between quantized integers and floating point. Raw peak operations per second reveal little about the resulting pipeline.

Consider live background blur. The camera supplies a frame, a model estimates a lower-resolution mask, and a graphics pass combines a blurred background with a sharp foreground. End-to-end metrics include mask age, frame cadence, memory traffic, and sustained power. Running inference on every second frame lowers inference load but may leave a stale boundary around a moving hand. A desktop with a discrete GPU may have more compute yet pay for transfers if the capture, model, and compositor each own incompatible allocations. A phone with compatible shared buffers can avoid some of those transfers.

## 9. Flash storage and durability

### From a file name to NAND

A file name belongs to a filesystem. The filesystem maps file offsets to logical blocks and manages metadata. The storage controller presents a logical block interface while its flash translation layer (FTL) maps those logical blocks to physical NAND locations. NAND programs pages and erases larger groups of pages called blocks. Updating a logical sector can write data at a new location and invalidate the old copy. Garbage collection later moves still-live pages and erases blocks for reuse. A 4 KiB application write can therefore cause more than 4 KiB of physical traffic.

Android devices can use UFS (Universal Flash Storage), with eMMC in some designs. UFS has a host controller, command protocol, device controller, and power states. On a typical desktop, the corresponding device is a replaceable NVMe SSD on PCIe. Both can contain NAND and an FTL. UFS, NVMe, and eMMC describe interfaces. Filesystems such as ext4, F2FS, and APFS describe another layer entirely. [Linux UFS driver documentation](https://docs.kernel.org/scsi/ufs.html)

~~~text
 file write -> filesystem and page cache -> block request -> host controller
            -> device firmware and flash mapping -> NAND program / erase
 later read <- cached page, or a block read followed by device completion
~~~

### When has a save actually finished?

A successful buffered write can mean that the OS accepted bytes into RAM. A filesystem transaction, storage cache flush, and device completion may still be pending. For a POSIX-style atomic file replacement, a typical pattern writes a temporary file in the same directory, checks writes, synchronizes the file, renames it over the destination, and synchronizes the containing directory where supported. The rename provides atomic namespace replacement. Synchronization addresses a different question, what will survive sudden power loss. Exact guarantees depend on filesystem, device, and platform API.

A database adds another level of ordering. In write-ahead logging, changes are recorded in a log before a later checkpoint updates the main database. Record and commit ordering, the configured durability mode, and the filesystem's flush semantics determine what a successful transaction promises. A mobile app that defers saving until a termination callback can lose edits when its process is reclaimed. Committing at meaningful user actions bounds the amount of work at risk.

Free space matters because a controller with few reusable blocks may have to copy live pages before accepting another write. Long-tail write latency can rise while headline sequential bandwidth still looks healthy. To investigate a slow save, time the encode step, buffered write, and durable commit separately. On a laptop the same distinction applies, even if its SSD is faster or replaceable.

## 10. Radios, networking, and sensors

### What the modem does while the CPU sleeps

The application processor runs apps, sockets, and much of the operating-system network stack. A cellular modem or **baseband** handles radio protocol processing, including timing, link adaptation, and communication with the network. The radio-frequency (RF) front end contains components such as filters, amplifiers, and antenna switches. The baseband can share a chip package with application processing or be a separate component. A SIM or eSIM holds subscriber credentials used by cellular service. It does not process every byte of application traffic.

The modem and network coordinate opportunities to listen for paging. A registered phone can leave its main CPU asleep during some of these intervals. The baseband handles radio events and can wake the application processor when host processing is needed. On a Wi-Fi-only tablet the cellular branch is absent. Wi-Fi can implement its own power-saving behavior, and internet messages can reach the device over Wi-Fi without a cellular subscription.

### Follow an upload

1. An app writes a request over a secure transport. User-space libraries and the kernel handle distinct portions of encryption, connection state, sockets, routing, and packet queues depending on the transport and platform.
2. The OS routes traffic to Wi-Fi or cellular. A driver and modem interface transfer packets or descriptors into the radio subsystem. An accepted socket write can precede transmission by a substantial interval.
3. On cellular, the device may need to transition to a suitable radio state and obtain transmission resources. Weak signal, retransmissions, and handovers change the airtime required for a fixed payload.
4. The receiving service processes the request and returns an application response. A transport acknowledgment establishes progress at the transport layer. An application acknowledgment can establish that the remote service committed the upload.

Sending small independent requests can keep the radio in a higher-power state between transfers. Batching deferrable work can reduce these transitions and allow longer sleep intervals. An interactive acknowledgment has a different deadline from thumbnail backup. The useful batch size depends on latency requirements, radio conditions, and the energy cost of each wake. Fixed “energy per byte” estimates hide those variables. [Android network access and power](https://developer.android.com/develop/connectivity/network-ops/network-access-optimization)

### Motion, location, and always-on work

An accelerometer measures specific force along its axes. A gyroscope measures angular velocity. A satellite-navigation receiver estimates position and time from received signals. A location service can combine several sources, including Wi-Fi and cellular observations, according to permissions and requested accuracy. The app may receive a fused estimate without manipulating any one receiver directly.

A sensor hub can sample at a steady rate, timestamp events near capture, and store them in a first-in, first-out queue. It can wake the main processor when a batch is due or a significant event occurs. Sampling rate determines how much detail is captured. Delivery interval determines how stale the app's view can be. For a pedometer, seconds of batching can be acceptable. For head tracking, the same delay can make the scene visibly lag motion. Firmware and the OS also have to relate sensor timestamps to the application clock. [Android sensor batching](https://source.android.com/docs/core/interaction/sensors/batching)

A laptop may also contain a sensor hub, Wi-Fi firmware, and a cellular modem. A tower using wired Ethernet is a useful comparison because it removes radio scheduling and signal variation from the path. It still has driver queues, DMA, interrupts, retransmission higher in the stack, and application-level acknowledgment.

## 11. Power, sleep, and heat

### From battery charge to completed work

The battery stores energy. The PMIC and voltage regulators deliver it at the voltages needed by processing, memory, display, and radio circuits. A **power domain** groups circuitry that can be powered or shut down together. A **clock domain** groups circuitry with related clocking. A powered domain can have its clock stopped while retaining state. Switching a domain off can save additional leakage but may require state restoration and a longer wakeup. Retaining an apparently idle peripheral or reference clock can prevent a larger domain from entering its deepest state.

Dynamic switching power is approximately proportional to activity × capacitance × voltage² × frequency. A higher clock can require a higher voltage, increasing power faster than frequency alone suggests. Leakage, memory, display, and radio power add to total draw. **DVFS**, dynamic voltage and frequency scaling, selects among supported operating points. A faster CPU can use less total energy if it finishes sooner and lets the entire system sleep. The same boost can waste energy if the job spends most of its time waiting on memory or a network response.

### Idle states and deadlines

~~~text
 Running -> clock gated -> retention state -> domain powered off
         lower entry cost                   greater potential saving
 Wake    <- resume clocks <- restore state <- power, reset, reinitialize
~~~

This sequence illustrates a spectrum of states. Each domain implements its own choices. A sleep controller weighs entry and exit cost against the likely idle interval. Periodic timers, background polling, or an app holding a wake lock can keep the application processor active even when its CPU utilization looks modest. The display, modem, and sensor hub may each have a different state. Android Doze adds OS policy restricting some background activity and providing opportunities for deferred work. Hardware idle and app scheduling policy therefore operate at different layers. [Android power management with Doze](https://source.android.com/docs/core/power/platform_mgmt)

A concrete example is a podcast with the screen off. The audio path must deliver samples continuously, but the main CPU may be able to fill buffers in batches or use an offloaded decoder. Frequent UI timers that update an invisible progress bar shorten idle intervals while adding no audible value. Changing the buffer size can improve sleep opportunities but increase response time when the listener presses pause.

### Thermal equilibrium and sustained performance

Heat takes time to spread from a chip through its package, frame, and enclosure to the air. The device can temporarily produce heat faster than it can dissipate while temperatures rise. As it approaches internal, battery, or surface-temperature limits, policy can reduce CPU/GPU frequencies, brightness, charging power, or workload quality, depending on the device. A vapor chamber spreads heat over a larger area. A fan and heatsink can also increase heat transfer to air. Neither technique changes the energy available in the battery. [Android thermal mitigation](https://source.android.com/docs/core/power/thermal-mitigation)

A larger tablet has room for more battery and a larger heat-spreading area, but its larger or brighter screen can consume the gain. A laptop plugged into power may sustain higher clocks, yet it still has fan noise, skin-temperature, and cooling limits. A tower has more room for airflow and can draw from mains power continuously. Comparisons should record ambient temperature, charging state, screen brightness, signal conditions, and elapsed load time. A ten-second burst and a twenty-minute export answer different engineering questions.

## 12. Firmware, boot, and updates

### Getting from reset to the lock screen

At reset, the application CPU begins in a platform-defined state. A small boot ROM establishes the first trusted code. An early loader initializes enough clocks, memory, and storage to load later stages. Firmware can train DRAM timing, configure secure memory, initialize a power controller, and start coprocessor firmware. The bootloader chooses an eligible software image, authenticates required components according to policy, and transfers control to the kernel. The kernel establishes memory management and starts user space. System services bring up display, input, networking, and the lock screen. Other processors can have their own boot sequences.

~~~text
 reset -> boot ROM -> authenticated loader stages -> kernel and early userspace
       -> system services -> lock screen -> user unlock -> protected user files
                               |
                           some services can already run
~~~

A **chain of trust** depends on a trusted first verifier and protected verification material. Each verified stage can authenticate the next. Android Verified Boot can also use a hash tree to check protected blocks as they are read. Rollback protection uses version information in protected state to reject disallowed older versions, even when an image bears a valid signature. Authenticity, integrity, and allowed version are three different checks. A corrupted block, a valid older vulnerable image, and a user file encrypted under an unavailable key are three different failures. [Android Verified Boot](https://source.android.com/docs/security/features/verifiedboot) · [Apple system security](https://support.apple.com/guide/security/system-security-overview-sec114e4db04/web)

### Updating a running device

An update changes software that the next boot must be able to verify and use. In an Android A/B scheme, the currently booted system remains a usable starting point while a new target is prepared. Virtual A/B can use snapshots and copy-on-write storage for dynamic partitions. After switching to the updated system and validating the boot, the system can merge snapshot data. The implementation needs enough temporary space and a defined recovery path when the target fails. Updates, fallback policy, and rollback protection must agree about which image is eligible. [Android Virtual A/B updates](https://source.android.com/docs/core/ota/virtual_ab)

A sudden power loss during download, during target construction, and after switching the selected boot target are different test cases. In each case ask which persistent record identifies a complete image, which verifier trusts it, and which code makes the next selection. On a desktop PC, UEFI firmware and an OS boot manager often select a loader from an EFI system partition. Modern PCs also support secure boot and device firmware updates. The mobile platform more often ships a board-specific collection of boot stages, camera firmware, modem firmware, kernel, and vendor components with a coordinated update policy.

## 13. Kernels, runtimes, and app lifetime

Android builds its kernel services on Linux. iOS and iPadOS use XNU, the kernel in Apple's Darwin system. Its Mach facilities include tasks, threads, virtual memory, and message passing. Its BSD facilities include process and filesystem interfaces. These are pieces of one kernel. A Mach task owns resources such as an address space, while a thread is an execution context the scheduler can run. An Apple framework may send a message through a Mach port to a service, just as an Android framework can reach a service through Binder. The message identifies a request or transfers rights to an object. Whether it also copies the payload depends on the interface.

### From an installed app to its first screen

An Android app package can contain resources, DEX bytecode, and native libraries for supported ABIs. The Android Runtime (ART) executes managed code using a combination of interpretation, just-in-time compilation, and ahead-of-time compilation as appropriate to the platform and installed profiles. Native libraries are loaded and linked into the process. The loader maps executable code and data, resolves dependencies, and applies necessary relocations. A mapped file can consume address space immediately while its pages enter physical memory on demand.

On a typical Android launch, system services arrange for an app process, often through the Zygote mechanism. Forking a prepared process lets children share many clean pages, with copy-on-write when a process modifies a shared page. The runtime initializes the application, creates the requested UI, and submits its first frame. Loading many dependencies synchronously can put page faults, disk reads, initialization code, and service calls on the first-frame critical path. [Android platform architecture](https://developer.android.com/guide/platform) · [Android app startup time](https://developer.android.com/topic/performance/vitals/launch-time)

On iOS and iPadOS, native applications typically contain Mach-O executables and use Apple's dynamic loader and system frameworks. Swift and Objective-C code can use automatic reference counting (ARC) to manage object lifetimes. An ownership cycle can keep objects alive after the UI no longer needs them. In a managed runtime, reachable objects can likewise survive garbage collection because a long-lived object still references them. Both models can leak useful memory through a logically stale reference. Neither captures all allocations made on behalf of an app by graphics or media services.

### What the kernel schedules

The kernel schedules threads, maps pages, controls device access, and accounts for resources. A context switch saves execution state so another thread can run. A process boundary can add a change of address space and other bookkeeping. Switching does not require copying the process's memory. If the camera service is waiting for a hardware frame, its thread can block and leave the CPU available to another task.

Priority interacts with dependencies. A high-priority audio thread that takes a mutex owned by a low-priority worker can miss its deadline if medium-priority work delays the owner. This is **priority inversion**. Priority inheritance on an appropriate lock can help the owner run. Avoiding unbounded waits, memory allocation, or blocking storage access on a deadline-sensitive callback removes additional failure modes. Even a very fast processor cannot give a hard latency bound to a callback that waits indefinitely on another service.

### What survives a switch away?

| State or event | What can happen | Consequence for app design |
| --- | --- | --- |
| Visible and interactive | UI work has an immediate presentation deadline. | Keep long or unpredictable work off the UI thread. |
| In the background | Eligible music, navigation, transfers, or scheduled tasks can receive special handling. | Use the OS facility whose lifetime and permissions match the operation. |
| Suspended or cached | Execution can stop while memory or a task record remains. | Elapsed wall time can exceed accumulated CPU time by minutes or hours. |
| Process reclaimed | Volatile heap state disappears. A later launch reconstructs the UI. | Persist essential edits and enough navigation state to restore context. |
| Device rebooted, before first unlock | Some services may start before certain protected files become accessible. | Handle unavailable data explicitly when a service can run during this phase. |

A desktop application commonly continues computing after its window loses focus while the machine is awake. Mobile systems more tightly mediate background time and resource use. For example, a system-managed transfer can continue under platform rules after the UI loses execution time. An ordinary timer in a suspended process cannot promise that behavior. The same distinction matters for laptops when the entire machine sleeps or a power policy limits background activity. [Apple background task strategies](https://developer.apple.com/documentation/backgroundtasks/choosing-background-strategies-for-your-app)

## 14. Security and virtualization

### Which mechanism protects which asset?

| Boundary | Enforcement | What remains to check |
| --- | --- | --- |
| App to app | Separate address spaces, OS identities, private storage, and sandbox policy | An exported service or shared file may intentionally cross the boundary. |
| App to camera or microphone | Platform permissions, service checks, and driver isolation | Access can change between launches or during a session. |
| CPU process to device memory | Driver validation, DMA mappings, and device-side address translation where implemented | A bad mapping or premature buffer reuse can expose other data. |
| Boot image to trusted code | Root of trust, signatures, integrity checks, and rollback policy | Signed code may still contain vulnerabilities. |
| Storage readout to plaintext | Encryption, hardware-bound secrets, and unlock-dependent key availability | Availability differs by file class and device state. |
| Normal-world software to protected keys | A trusted execution environment or dedicated secure subsystem can authorize key operations | Key use and extraction are different privileges. |

On Android, a distinct application UID, Linux isolation, and SELinux policy restrict access to other apps and system services. An app may obtain a narrower capability, such as access to an individual user-selected file, without broad storage access. On Apple platforms the sandbox, code signing, and entitlements participate in deciding which code and services an app can use. A permission dialog addresses a user-facing choice. Kernel and service checks continue enforcing the decision after the dialog closes. [Android system and kernel security](https://source.android.com/docs/security/overview/kernel-security) · [Apple system security](https://support.apple.com/guide/security/system-security-overview-sec114e4db04/web)

### A locked phone still has useful work to do

Android file-based encryption distinguishes device-encrypted storage available during Direct Boot from credential-encrypted storage that becomes available after the user unlocks. Apple Data Protection assigns classes to files, with different rules for access when locked, after first unlock, or while a previously opened file remains in use. A background service that runs before first unlock must choose which data it genuinely needs at that time. Rebooting and locking an already-unlocked phone are different tests. [Android file-based encryption](https://source.android.com/docs/security/features/encryption/file-based) · [Apple Data Protection classes](https://support.apple.com/en-ie/guide/security/secb010e978a/web)

A secure processor can keep sensitive key material inside an isolated component and authorize an operation without returning the underlying key to the app. Apple's Secure Enclave is one such subsystem, with its own boot and protection mechanisms. A hardware-backed key can still be misused if policy authorizes the wrong caller or the caller signs the wrong data. This is why an architecture review should identify both the location of the secret and the authorization for each use. [Apple hardware security overview](https://support.apple.com/guide/security/hardware-security-overview-secf020d1074/web)

### Process isolation, trusted execution, and virtual machines

A conventional app sandbox shares the OS kernel. A virtual machine runs a guest kernel under a hypervisor. On Arm, a guest can translate a process virtual address into a guest physical address, while a second stage translates that address into memory the hypervisor allows the guest to access. Device I/O also requires appropriate isolation. A guest's CPU page tables alone cannot contain arbitrary device DMA.

Android's Virtualization Framework supports protected execution on compatible devices. Its protected-VM model can restrict what the host can access in guest memory. A protected guest still needs an interface for input, output, and storage, and those interfaces define what information can cross the boundary. A trusted execution environment and a dedicated key processor solve narrower problems with different trust relationships. A desktop developer VM, a mobile app process, and hardware-backed key storage therefore have distinct answers to “which privileged component could inspect these bytes?” [Android Virtualization Framework](https://source.android.com/docs/core/virtualization)

## 15. End-to-end walkthroughs

### A. One instruction reads an image pixel

1. A native image routine executes an AArch64 load such as `LDR W0, [X1]`. Assume X1 points to four readable bytes of a CPU-accessible, correctly aligned image and that device work producing the image has completed.
2. The core computes the virtual address. The MMU consults cached translations or walks page tables, checks read access, and identifies the physical location. Cache lookup and parts of translation can overlap depending on the design.
3. If the data is in a nearby cache, the load can complete quickly. A miss requests a line through the interconnect. It can be supplied by another cache or by the memory controller. A display engine reading other lines at the same time can contribute to contention.
4. When the bytes arrive, the core makes the value available to dependent instructions and eventually retires the load. If the access instead raises a recoverable fault, the kernel resolves it and execution retries. An invalid mapping or permission error can terminate the process.
5. Interpreting the value requires knowing byte order, channel order, format, and stride. Successfully loading four bytes establishes that the memory was readable. It does not establish that those bytes represent the intended pixel. A protected or GPU-only surface may require a different API altogether.

**Where time went.** Four bytes of useful data can cause a translation walk, a full cache-line transfer, or even a storage read if the mapped data was absent. A million adjacent loads amortize those costs very differently from a million random loads. The equivalent desktop instruction uses the same conceptual path. On a discrete-GPU desktop, obtaining a CPU-readable copy of the image might dominate everything that follows.

### B. A finger scrolls a page

1. A touch controller samples the sensor and reports contacts. Driver and input services timestamp, normalize, and dispatch events to the appropriate window. Events can accumulate or be coalesced before the app handles them.
2. The UI thread processes the gesture, updates the scroll position, and schedules a frame. If a synchronous database query blocks that thread, the GPU may sit idle despite the visible stutter.
3. Rendering code records commands. The GPU draws changed content into a buffer. Previously rendered tiles or layers can sometimes be reused, reducing work while scrolling.
4. The compositor picks a ready buffer for the next presentation opportunity. If the new frame misses the latch, the previous image can remain for another refresh. The panel scans the selected image over time.
5. Measure input timestamp, UI execution, GPU completion, latch, and presentation. A frame timeline can distinguish an app missing its deadline from system composition running late. The event you want to explain is when the change became visible. [Perfetto FrameTimeline](https://perfetto.dev/docs/data-sources/frametimeline)

### C. A minute of camera video reaches flash

1. The app requests a camera session with a preview target and an encoder target. The service checks access and the HAL validates whether their resolutions, rates, and formats can be produced together.
2. Sensor readout and the ISP create timestamped frames. The preview can be scaled for the screen. The encoder can consume a YUV image through a supported surface, avoiding a round trip through a CPU-managed RGB array.
3. The encoder produces compressed video samples. The audio system captures and encodes sound on its own cadence. The recorder uses timestamps and writes a container that describes how to synchronize and decode the streams.
4. Encoded samples pass through the filesystem to flash. If writes stall and the outstanding buffers fill, the delay propagates upstream. Depending on policy, the recorder may drop frames, reduce quality, or stop with an error. A display that continues previewing does not prove the saved file is complete.
5. Stopping capture drains pending samples and finalizes the container. A playable local file, a durably stored file, and a remotely backed-up file are three completion points. Test interruption near each boundary. A laptop using a USB camera has the same timing problem with additional transport and potentially a camera that already delivers compressed video.

### D. A message arrives with the screen off

1. The device has registered with the network. A modem or Wi-Fi subsystem handles link-level listening while the application processor can spend time in a low-power state. Assume the app uses a platform push service and the message is eligible for delivery.
2. Network activity reaches the device at a listening opportunity. Firmware and system networking bring the necessary host components out of idle. A system push service can maintain a connection on behalf of many apps, avoiding independent polling timers.
3. System policy considers message type, priority, permissions, and device state. The OS may present a notification or grant limited app execution. A silent background update can be deferred. Network delivery and visible presentation are separate events.
4. When the user taps, the OS resumes or launches the app. If the device was rebooted and remains locked, protected message data may still be unavailable. The app must reconcile local state with the service when it gets access.
5. To diagnose “late notification,” record server send time, device receipt if observable, presentation time, and user-open time. Establish whether the device had connectivity and whether the app had notification permission. Restarting the app's UI says little about the modem's earlier state.

### E. An editor returns after memory pressure

1. The user edits an image and switches to a game. The editor persists the edit recipe and file reference. Its decoded bitmap, shader cache, and current undo objects still occupy RAM while its process remains alive.
2. The game allocates textures and render targets. The system reclaims clean pages, compresses eligible pages on some configurations, or selects a lower-priority process for termination. Retaining a reference to an unused bitmap increases the chance that useful work has to be discarded somewhere.
3. The user taps the editor in the switcher. A still-alive process can resume rapidly. A reclaimed process starts again, maps code and frameworks, opens the saved edit state, and decodes the required region.
4. Restoring a thumbnail immediately and decoding a full-size image later can shorten time to first display. Time to usable editing also includes input readiness, full-resolution data availability, and any pending GPU work.
5. On Apple platforms a jetsam event report can identify a memory-related termination. On Android, process-exit evidence and memory-pressure traces can narrow the cause. A normal crash, an intentional user stop, and pressure-driven reclamation require different fixes. [Apple jetsam event reports](https://developer.apple.com/documentation/xcode/identifying-high-memory-use-with-jetsam-event-reports)

## 16. Quantitative reasoning

All numbers below are worked examples with stated assumptions. MB and GB use decimal units. KiB and MiB use powers of two. The workload's logical traffic, measured DRAM traffic, and bytes saved to a file can differ by orders of magnitude.

### Display and camera bandwidth

| Workload | Calculation | Interpretation |
| --- | --- | --- |
| One 2560 × 1600 RGBA8 surface | 2560 × 1600 × 4 = 16,384,000 B, or 15.625 MiB | Three such buffers occupy 46.875 MiB before padding or auxiliary data. |
| Read that surface once per 120 Hz refresh | 16,384,000 × 120 = 1.96608 GB/s | A raw full-frame read rate. Compression, panel self-refresh, scaling, and extra composition passes change external traffic. |
| One 3840 × 2160 8-bit YUV 4:2:0 frame | 8,294,400 × 1.5 = 12,441,600 B | One luma sample per pixel and two chroma samples per 2 × 2 group. Assumes tight packing. |
| One pass over that stream at 30 frames/s | 12,441,600 × 30 = 373.248 MB/s | A write followed by a read doubles the logical traffic to about 746.5 MB/s. |
| Encoded video at 20 Mbit/s for 60 s | 20 × 60 ÷ 8 = 150 MB | Compressed video payload. Container metadata and audio add bytes. |

These figures explain why a camera can be memory-bandwidth constrained while flash writes are modest. Halving the encoded bitrate reduces file size but may leave sensor readout and preview traffic almost unchanged. Switching from 30 to 60 captured frames per second doubles the raw-frame rate at the same format and dimensions. A format that stores each 10-bit component in a 16-bit word needs a fresh calculation.

### Deadline, throughput, and queue depth

At 60 Hz a refresh interval is about 16.67 ms. At 120 Hz it is about 8.33 ms. Suppose UI work takes 5 ms and GPU work takes 6 ms. With independent frames and sufficient buffers, ideal pipelined throughput can approach one frame every 6 ms after fill. A particular frame with a strict CPU-then-GPU dependency still requires at least 11 ms from the start of its UI work to GPU completion. Input sampling, composition, waiting for a refresh, and scanout add latency. Sustainable throughput can therefore meet a refresh rate while touch-to-photon latency exceeds one interval.

Little's law relates average work in flight to throughput and average time in the system. At 60 completed frames/s and 50 ms average residence, the pipeline contains 60 × 0.050 = 3 frames on average. This assumes stable operation and consistent boundaries for counting frames. Adding buffers can prevent stalls, but if the producer simply queues more stale work, throughput stays unchanged while latency rises. Track queue residence time alongside engine execution time.

### Battery life and thermal limits

Suppose a battery has **18 Wh of usable energy** and the whole device draws **3 W at the battery** on average. Ideal runtime is 18 ÷ 3 = 6 hours. At 5 W it is 3.6 hours. A charge rating in ampere-hours needs a voltage to estimate energy. For example, 4.5 Ah × 3.85 V ≈ 17.3 Wh at nominal voltage. Use whole-device draw or include conversion losses when estimating runtime from subsystem power.

For an optional computation, compare a baseline and an optimized path over the same interval. If extra work draws an incremental 2 W for 0.4 s, its incremental energy is 0.8 J. An alternative at 1.2 W for 0.8 s costs 0.96 J. The lower-power version costs more energy in this example. If it changes screen-on time, radio tail duration, or thermal state, repeat the accounting over the complete user operation.

A rough steady-state model uses temperature rise ≈ sustained heat load × effective thermal resistance. With an illustrative 5 °C/W resistance, a sustained 4 W heat load gives a 20 °C rise from ambient at the modeled measurement point. Real devices have multiple heat sources, thermal paths, sensors, changing airflow, and transient heat capacity. A fast initial benchmark reflects stored thermal capacity as well as cooling. Compare steady-state throughput only after traces show that temperature and operating points have settled.

### Choosing an optimization

Suppose inference takes 40 ms and copying or converting its input takes 20 ms. A fourfold inference speedup reduces total latency from 60 ms to 30 ms, a twofold improvement. Eliminating the conversion as well would reduce it to 10 ms, assuming compatible buffers and identical output. This is Amdahl's law applied to a whole pipeline. Verify that the new path actually removes the copy rather than moving it into a driver.

For an arithmetic kernel with 4 operations per byte fetched from DRAM and 20 GB/s of available bandwidth, a simple roofline upper bound is 80 billion operations/s, further capped by compute throughput. Blocking or reusing data can raise operations per byte. Adding faster arithmetic units alone does little if memory is already the limit. The bandwidth available while the camera and display are active can be smaller than a memory benchmark measured in isolation.

## 17. Phones, tablets, laptops, and desktops

| Design question | Phone | Tablet | Laptop | Desktop tower |
| --- | --- | --- | --- | --- |
| Which processors share the workload? | Typically an Arm SoC with integrated graphics, ISP, and media engines | Phone-class or laptop-class SoC, often with pen or external-display support | Arm or x86-64, often integrated graphics, sometimes a discrete GPU | CPU with integrated graphics or a discrete card, sometimes both |
| Where can graphics data live? | Usually system RAM shared by multiple engines | Similar model, with capacity dependent on product | Shared RAM or dedicated graphics memory | Shared RAM on integrated designs, dedicated VRAM common on discrete cards |
| What sets sustained speed? | Small battery, tight skin-temperature limits, usually passive cooling | Larger chassis, display power, often passive cooling | Battery or AC policy, fan capability, enclosure size | System power supply, processor power limits, cooling and airflow |
| How does the device stay connected? | Cellular plus Wi-Fi, often with always-on low-power subsystems | Wi-Fi standard in typical models, cellular optional | Wi-Fi common, cellular optional, Ethernet model-dependent | Wired Ethernet and Wi-Fi common, external cellular possible |
| How do users interact? | Touch, cameras, microphones, orientation and proximity sensors | Touch and pen, detachable keyboard or pointer on many models | Keyboard, trackpad, optional touch or pen | Independent monitors, keyboard, mouse, and peripherals |
| Which work can continue offscreen? | App lifecycle and background policies govern eligible work | Mobile policies on Android/iPadOS, desktop policies on Windows tablets | Desktop OS generally permits background processes while awake | Similar OS behavior, typically fewer battery-driven restrictions |
| Which parts can be changed? | RAM and flash commonly fixed on board | Similar to phones | Varies from soldered assemblies to socketed RAM and replaceable storage | RAM, graphics, storage, and expansion cards commonly replaceable within board and power limits |
| How is the system updated? | OS, board support, and firmware often coordinated by the platform and vendor | Follows installed OS and device family | OS, system firmware, drivers, and peripheral firmware can have separate lifecycles | Similar to laptops, often with a wider mix of individually sourced components |

**Phone versus tablet.** Consider the same drawing app using the same CPU family. A larger canvas may allocate more tiles and larger render targets. A pen can add high-rate input with a tight feedback deadline. Split-screen operation can leave another visible app competing for RAM and GPU time. Compare the number of pixels actually drawn, the live working set, and the active input devices before attributing the difference to processor speed.

**Tablet versus laptop.** A fanless tablet and laptop can use related silicon and draw from similar memory systems. A background export may nevertheless finish differently if the tablet app is suspended after the user changes tasks. If both stay in the foreground, differences in codec support, cooling, and power policy become more visible. A Windows detachable with a keyboard may follow the laptop software model even when held as a tablet.

**Laptop versus tower.** Configure the same workload and document whether it uses integrated or discrete graphics, whether data must cross PCIe, and how much graphics memory it needs. A short export that uses the same fixed-function codec on both machines may take similar time. A long unsupported effect rendered on general GPU units can reveal a large gap. A tower's upgrade paths and larger cooling system matter only when the configured workload can use them.

## 18. Diagnosing a real workload

### Start with an observable boundary

Choose an event pair that matters to the user. “Tap to first editable frame,” “shutter press to durable photo,” and “screen-off energy over one hour” describe different experiments. Record device, OS, build, data set, display mode and brightness, ambient conditions, power source, and network type. Separate cold launch, warm launch, and returning to an already-rendered screen. A cold process can still benefit from a warm filesystem cache. Rebooting the device changes many more variables than restarting one process.

| Symptom | Evidence that distinguishes plausible causes | Follow-up experiment |
| --- | --- | --- |
| Scrolling stutters | UI thread running, runnable, or blocked around a missed frame, plus GPU and compositor completion | Repeat with the same content after replacing synchronous image decoding with cached thumbnails. |
| Video capture stops late in a session | Temperature, thermal state, storage commit latency, free space, buffer occupancy | Hold capture settings constant and vary starting temperature or available storage separately. |
| App reloads when switching back | Process exit reason, memory pressure, decoded images and GPU allocations | Repeat with a smaller decoded working set and restore the same navigation state. |
| Phone drains overnight | Wake intervals, radio activity, scheduled transfers, baseline screen-off draw | Batch the same payload into fewer transfers and compare energy under similar signal conditions. |
| Model benchmark is fast but the feature is slow | Timestamps for input conversion, inference, synchronization, compositing, and display | Keep model output constant and change one buffer handoff. |
| Laptop wins only on long runs | Throughput versus elapsed time, frequency, thermal state, AC/battery policy | Compare the first iteration, a thermally settled interval, and energy per completed job. |

### Read a trace in causal order

For an Android graphics issue, use a system trace with scheduling and frame timeline data when available. Find a late presented frame, identify its app and compositor slices, then inspect the thread or fence on which the late stage waited. If a thread was runnable for 4 ms without executing, inspect load and scheduling. If it was asleep on a fence, follow the producing engine. If it was running, profile the code on that interval. CPU percentages averaged over a second can hide all three cases. [Perfetto FrameTimeline](https://perfetto.dev/docs/data-sources/frametimeline)

For energy, compare the complete task and a baseline under similar conditions. Android's Power Profiler can expose device power rails on supported hardware. Apple provides Power Profiler and Instruments for supported devices and workflows. A whole-device rail may include work by other apps or services. An attached charging cable, profiler overhead, bright screen, or changing network signal can affect the observation. Keep the setup fixed, collect repeated runs, and report measured scope and variability. [Android Studio Power Profiler](https://developer.android.com/studio/profile/power-profiler) · [Apple Power Profiler](https://developer.apple.com/documentation/xcode/measuring-your-app-s-power-use-with-power-profiler)

### Example investigation

Imagine a 120 Hz tablet that scrolls smoothly for a minute and then begins repeating frames. A trace shows its UI thread finishing on time. GPU completion shifts later as the chassis warms. That supports a GPU or memory-path investigation. It does not yet identify the expensive draw. Inspect render passes, intermediate surface writes, and frame size. Repeat with one unnecessary full-screen effect removed at the same resolution and brightness. Compare late frames, GPU duration, and sustained energy over the same interval.

If frame times improve only after lowering brightness, the display's contribution to the shared heat budget deserves investigation. If a cold device also misses frames only during image decode, there may be a second CPU or storage issue. State each result in terms of the change made and the time boundary measured. This keeps a valid finding about thermal steady state from becoming an unsupported claim about every slow frame.

## 19. Further reading

- For the exception, translation, and ordering behavior behind a native instruction, start with Arm's architecture guides. [Exception model](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/Exception%20model.pdf) · [Memory management](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/LearnTheArchitecture-MemoryManagement-101811_0100_00_en.pdf)
- For the Android path from an app to a hardware service, follow a framework call through Binder, the HAL, and the camera request contract. [Android platform architecture](https://developer.android.com/guide/platform) · [Binder](https://source.android.com/docs/core/architecture/ipc/binder-overview) · [Camera HAL](https://source.android.com/docs/core/camera/camera3_requests_hal)
- For the ownership and synchronization of graphics buffers, read the OS and driver views together. [Android graphics architecture](https://source.android.com/docs/core/graphics/arch-bq-gralloc) · [Linux DMA-BUF](https://www.kernel.org/doc/html/latest/driver-api/dma-buf.html)
- For hardware security and data availability after locking or rebooting, compare the platform guides. [Apple hardware security](https://support.apple.com/guide/security/hardware-security-overview-secf020d1074/web) · [Android file-based encryption](https://source.android.com/docs/security/features/encryption/file-based)
- For a comparison with a PC, the companion note in the computer-architecture folder covers desktop and server pipelines, PCIe, storage queues, memory hierarchy, and virtualization. Read its file-read walkthrough alongside the mobile camera-to-flash example.
