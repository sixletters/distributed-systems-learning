---
layout: page
title: Networking & ML Networking
permalink: /networking/
---

The path to roles like OpenAI Core Network Engineering or Anthropic ML Networking. These are systems-level positions — you are close to the hardware, writing C/C++/Rust, and your bugs cause training jobs to stall or clusters to drop packets. This section documents the path from "knows TCP/IP" to "writes RDMA applications and implements collective algorithms."

The JDs these notes target:
- **OpenAI Core Network Engineering** — host networking, datacenter fabrics, WAN, RDMA/RoCE/InfiniBand, performance engineering, C++/Python/Go
- **Anthropic ML Networking** — kernel networking (XDP/eBPF/io_uring), user-space networking (DPDK/RDMA), collective algorithms, Rust, SmartNICs, PCIe

---

## Phase 0 — Networking Fundamentals (4 weeks)

Before any of the advanced material, these must be solid. Not "I've heard of TCP" — actually solid.

| Topic | Why it matters |
|-------|---------------|
| OSI model, layers 1–7 | The vocabulary of every networking conversation; JDs explicitly call out "OSI layers 2–4" |
| Ethernet, ARP, MAC addressing (Layer 2) | Datacenter fabrics operate at L2; RDMA over Ethernet starts here |
| IP routing, subnets, CIDR, ICMP (Layer 3) | Foundation for all WAN and datacenter routing work |
| TCP: connection lifecycle, congestion control, flow control, segment structure | The protocol you will be tuning and replacing in ML networking |
| UDP: why it exists, when it wins | RDMA transports, some HPC protocols, congestion control work |
| DNS, HTTP/1.1, HTTP/2 (conceptually) | API serving context; understand the protocol stack you're debugging |
| Linux networking tools: `tcpdump`, `wireshark`, `ss`, `netstat`, `ip`, `ping`, `traceroute`, `iperf3` | Diagnostic tools you use daily as a network engineer |

### Resources

- **Computer Networking: A Top-Down Approach — Kurose & Ross** — the canonical university text; read chapters 1–4 in full, chapters 5–6 as needed
- **Beej's Guide to Network Programming** (beej.us/guide/bgnet) — free, hands-on C sockets; the fastest way to make networking real
- [Julia Evans — Networking Zines](https://jvns.ca/networking-zine.pdf) — free, excellent for filling gaps quickly
- [RFC 793 (TCP)](https://tools.ietf.org/html/rfc793) — read the actual spec once; the state machine section is worth your time

### Key Exercises

- Use `tcpdump -i lo port 8080` to capture a full TCP handshake from a local server you write. Identify SYN, SYN-ACK, ACK, data segments, and FIN in the output.
- Write a TCP echo server and client in C using `socket()`, `bind()`, `listen()`, `accept()`, `recv()`, `send()`. No wrappers. This is Beej's Guide chapter 5 — do it.
- Deliberately cause TCP retransmission: introduce packet loss with `tc netem` and watch `tcpdump` capture the retransmit. Then measure throughput degradation with `iperf3`.
- Trace a DNS resolution with `dig +trace` on a real domain. Draw the full resolution chain from root to authoritative server.

---

## Phase 1 — Linux Networking Internals (6 weeks)

The Anthropic JD explicitly lists "TCP/IP stack internals, XDP, eBPF, io_uring, and epoll." This is where you learn how the kernel actually handles packets.

| Topic | Why it matters |
|-------|---------------|
| Kernel receive path: NIC → driver → NAPI → sk_buff → socket receive queue | How a packet physically arrives; prerequisite for understanding DPDK/XDP |
| `sk_buff` (socket buffer) structure | The kernel data structure that represents every packet; you'll see this in every kernel networking discussion |
| NAPI (New API) polling | How modern NICs deliver packets without interrupt storm at high rates |
| Network namespaces, veth pairs, Linux bridges | How containers network; needed for understanding datacenter host networking |
| Traffic control (`tc`): queueing disciplines, `netem`, `HTB` | How to shape traffic, introduce artificial delay/loss for testing |
| `epoll`: edge-triggered vs level-triggered, the C10K problem | The I/O multiplexing primitive that high-performance servers are built on |
| `io_uring`: submission queues, completion queues, async I/O without syscall overhead | The modern replacement for epoll in high-throughput I/O code; explicitly listed in Anthropic JD |
| eBPF: programs, maps, verifier, `bpf()` syscall | The kernel programmability layer used for XDP, tracing, and custom protocol offloads |
| XDP (eXpress Data Path): hooking into the NIC driver before the kernel stack | Kernel-bypass for filtering and forwarding at line rate; listed explicitly in Anthropic JD |
| `perf`, `bpftrace`, `ftrace`: kernel-level profiling | How to find where microseconds are being lost in the network path |

### Resources

- **Linux Kernel Networking — Rami Rosen** — the closest thing to a reference manual for the Linux network stack; read the sk_buff chapter early
- **Understanding Linux Network Internals — Christian Benvenuti** — older but deeper on the receive/transmit path internals
- [eBPF.io](https://ebpf.io/what-is-ebpf/) — start here for eBPF orientation
- [The XDP Tutorial (xdp-tutorial on GitHub)](https://github.com/xdp-project/xdp-tutorial) — hands-on, incremental, the best practical resource for learning XDP
- [io_uring explained — Lord's blog](https://unixism.net/loti/) — "Lord of the io_uring" — free, comprehensive, with exercises
- [BPF Performance Tools — Brendan Gregg](https://www.brendangregg.com/bpfperftools.html) — the definitive bpftrace/BCC reference; Gregg is the expert here
- [Linux Networking Documentation](https://www.kernel.org/doc/html/latest/networking/) — primary source; read the packet scheduler, XDP, and eBPF sections

### Key Exercises

- Trace the receive path of a single UDP packet using `bpftrace` — instrument `kprobe:udp_rcv`, `kprobe:ip_rcv`, and `kprobe:napi_gro_receive`. Watch it arrive in real time.
- Write an XDP program (C + libbpf) that counts packets per source IP and exposes counts via a BPF map. Start from the XDP tutorial `basic01-xdp-pass` lesson and build up.
- Implement an `epoll`-based HTTP server in C — one thread, handle 100 concurrent connections without blocking. Benchmark with `wrk`. Then compare latency vs a blocking-accept server.
- Write a simple io_uring file reader in C that submits 100 read operations in a batch and harvests completions without a syscall per operation. Measure syscall count with `strace -c`.
- Use `tc netem` to simulate 10ms latency + 1% packet loss on a loopback interface. Run `iperf3`, observe throughput collapse. Then tune TCP socket options (`TCP_NODELAY`, socket buffer sizes) and observe improvement.
- Profile the kernel network stack latency using `perf trace` — find where time is spent between NIC interrupt and `recv()` returning in userspace.

---

## Phase 2 — Systems Programming (6 weeks)

You cannot work on kernel networking or write RDMA applications without systems programming skills. The Anthropic JD calls out "comfort with or desire to learn Rust" and "memory management, lock-free data structures, NUMA-aware programming."

### Option A: Rust (recommended for Anthropic-track)

| Topic | Why it matters |
|-------|---------------|
| Ownership, borrowing, lifetimes | The core memory safety model; no GC, no use-after-free |
| `unsafe` Rust: raw pointers, FFI | Required for RDMA libraries and kernel interactions |
| Traits, generics, trait objects | How Rust achieves zero-cost abstraction |
| Async/await in Rust: `tokio`, `async-std` | High-performance async networking without the GIL |
| `std::sync`: `Mutex`, `RwLock`, `Arc`, `AtomicU64` | Shared state in concurrent systems code |
| Lock-free data structures: `crossbeam`, `flume` | High-throughput queues for packet processing pipelines |
| NUMA-aware memory allocation | Pinning memory to the correct NUMA node for NIC DMA performance |

#### Resources

- [The Rust Programming Language (book.rustlang.org)](https://doc.rust-lang.org/book/) — free, official, read chapters 1–15 fully
- **Programming Rust, 2nd ed. — Blandy, Orendorff, Tindall** (O'Reilly) — deeper than the official book on unsafe and concurrency
- [Rustonomicon](https://doc.rust-lang.org/nomicon/) — the guide to unsafe Rust; read when you're writing FFI or raw pointer code
- [Tokio tutorial](https://tokio.rs/tokio/tutorial) — async Rust from the authors of the most-used async runtime

### Option B: C/C++ (OpenAI-track)

| Topic | Why it matters |
|-------|---------------|
| Pointers, pointer arithmetic, `malloc`/`free` | Memory is manual; bugs here cause silent corruption |
| `valgrind`, `AddressSanitizer`, `UndefinedBehaviorSanitizer` | The tools you use to find memory bugs before they hit production |
| `pthreads`, mutexes, condition variables | POSIX threading primitives used throughout systems code |
| Atomic operations: `__atomic_*`, `std::atomic` | Lock-free programming building blocks |
| `mmap`, huge pages, `madvise`, `mlock` | Memory management for RDMA buffers and DPDK |
| Cache-line alignment, false sharing, prefetching | Performance tuning at the hardware level |

#### Resources

- **The C Programming Language — Kernighan & Ritchie** — still the best introduction; read it cover to cover
- **Computer Systems: A Programmer's Perspective — Bryant & O'Hallaron** — chapters on memory hierarchy, linking, and exceptional control flow are the most important
- **C++ Concurrency in Action — Williams** — the definitive reference for `std::atomic`, `std::mutex`, and lock-free patterns in C++

### Key Exercises (language-agnostic)

- Implement a lock-free SPSC (single-producer, single-consumer) ring buffer. Verify it with a concurrent stress test — producer and consumer running in separate threads for 10 seconds. No mutex.
- Write a NUMA-aware memory allocator that allocates memory on the same NUMA node as a given CPU. Use `numa_alloc_onnode()` (C) or a Rust equivalent. Measure access latency vs cross-NUMA allocation with `perf stat`.
- Implement a simple memory pool for fixed-size objects — pre-allocate N objects, hand them out and take them back without calling `malloc`. Benchmark allocation throughput vs `malloc`.
- Reproduce a data race deliberately (two threads incrementing a shared counter without synchronization), detect it with ThreadSanitizer, then fix it with an atomic.

---

## Phase 3 — High-Performance Networking: RDMA, DPDK, Kernel Bypass (8 weeks)

This is the core of both JDs. RDMA is how AI training clusters move tensor data. DPDK is how packet processing happens at line rate without kernel involvement.

| Topic | Why it matters |
|-------|---------------|
| RDMA fundamentals: what it is, the three operations (SEND/RECV, RDMA Read, RDMA Write) | The protocol that moves tensor data in distributed training without involving the CPU |
| InfiniBand architecture: HCAs, switches, subnet manager, GUIDs, LIDs | The physical/logical infrastructure you'd operate at OpenAI |
| RoCE (RDMA over Converged Ethernet): v1 vs v2, PFC, ECN | How RDMA runs over commodity Ethernet; most new AI clusters use this |
| Verbs API: `ibv_post_send`, `ibv_post_recv`, completion queues, memory registration | The C API for writing RDMA applications |
| DPDK: the architecture — poll mode drivers (PMDs), DPDK memory model (`rte_mbuf`) | Kernel-bypass packet processing at line rate |
| DPDK: `rte_ring`, `rte_mempool`, multi-queue NICs, RSS | The data structures that make DPDK fast |
| Congestion control for RDMA: PFC (Priority Flow Control), ECN, DCQCN, TIMELY | How AI training clusters avoid head-of-line blocking during AllReduce |
| Memory registration and DMA: why you must pin memory for RDMA | The constraint that drives memory management design in RDMA applications |
| SR-IOV and virtual functions: how NICs share between VMs and containers | How host networking works in virtualized AI infrastructure |

### Resources

- **RDMA Aware Networks Programming User Manual** (Mellanox/NVIDIA, free PDF) — the primary reference for the verbs API; the ping-pong example is where you start
- [RDMA core library (github.com/linux-rdma/rdma-core)](https://github.com/linux-rdma/rdma-core) — read the examples in `libibverbs/examples/`
- **DPDK documentation** (doc.dpdk.org) — read the programmer's guide, the Getting Started guide, and the sample apps overview
- [DPDK — Getting Started Guide for Linux](https://doc.dpdk.org/guides/linux_gsg/) — set up a DPDK environment on a local machine (even a VM)
- **InfiniBand Architecture Specification (IBTA)** — dense but authoritative; read the chapter on transport services
- [DCQCN paper — "Congestion Control for Large-Scale RDMA Deployments"](https://conferences.sigcomm.org/sigcomm/2015/pdf/papers/p523.pdf) (Zhu et al., SIGCOMM 2015) — how RoCE congestion control actually works
- [RDMA over Commodity Ethernet (RoCE) — Chelsio whitepaper](https://www.chelsio.com/wp-content/uploads/resources/T5-10Gb-Ethernet-Adapters-RoCEv2-and-RDMA-Whitepaper.pdf) — accessible overview of RoCE v2 and PFC

### Key Exercises

- **RDMA ping-pong** — write a basic RDMA send/receive ping-pong benchmark using `libibverbs` directly (not a library wrapper). Measure one-way latency. The RDMA Aware Networks manual example is your starting point — understand every line.
- **RDMA Write benchmark** — replace send/receive with RDMA Write (one-sided operation). Measure throughput as a function of message size. Understand why RDMA Write has lower CPU overhead.
- **DPDK `l2fwd` sample** — build and run the DPDK L2 forwarding sample on a test machine or VM. Understand how `rte_eth_rx_burst()` and `rte_eth_tx_burst()` work. Modify it to count packets per source MAC.
- **Write a DPDK ring benchmark** — create two `rte_ring` queues, run a producer and consumer in separate threads (or lcores), measure enqueue/dequeue throughput at different batch sizes.
- **PFC simulation** — use `tc netem` to simulate a lossless network, then introduce a congestion point. Observe queue buildup. Experiment with ECN marking thresholds. This gives you intuition for why congestion control matters in RDMA deployments.

---

## Phase 4 — ML/AI Networking: Collectives, NCCL, GPU Interconnects (6 weeks)

This is the specialist layer. You need Phase 3 before this makes sense.

| Topic | Why it matters |
|-------|---------------|
| Distributed training overview: data parallelism, model parallelism, pipeline parallelism | Why networking matters — gradient synchronization is a collective operation |
| Collective operations: AllReduce, AllGather, Broadcast, Reduce-Scatter, Barrier | The primitives every distributed training framework uses; you implement and optimize these |
| Ring-AllReduce algorithm | The bandwidth-optimal AllReduce algorithm used by Horovod and others |
| Tree-AllReduce and recursive halving-doubling | Alternative algorithms with different latency/bandwidth tradeoffs |
| NCCL (NVIDIA Collective Communications Library) — architecture, transport selection, tuning | The library that orchestrates GPU-to-GPU communication; you tune this as an ML networking engineer |
| NVLink and NVSwitch: intra-node GPU interconnects | The fabric that connects GPUs within a node; orders of magnitude faster than PCIe |
| PCIe: topology, bandwidth, peer-to-peer access, BAR space | How GPUs, NICs, and CPUs connect; bottleneck when NVLink isn't available |
| SHARP (Scalable Hierarchical Aggregation and Reduction Protocol) | In-network compute for AllReduce on InfiniBand — reduces host CPU involvement |
| MPI (Message Passing Interface): `MPI_Allreduce`, `MPI_Isend`, `MPI_Irecv` | The HPC communication standard; NCCL traces its lineage here |
| Tensor parallelism communication patterns: AllReduce vs Reduce-Scatter + AllGather | How model parallelism changes the collective communication pattern |

### Resources

- **"Scaling Distributed Machine Learning with the Parameter Server"** (Li et al., OSDI 2014) — foundational paper on distributed training communication
- **"Horovod: fast and easy distributed deep learning in TensorFlow"** (Sergeev & Del Balso, arXiv 2018) — describes ring-AllReduce implementation; read the full paper
- **"Massively Scale Your Deep Learning Training with NCCL 2.4"** (NVIDIA blog) — practical NCCL tuning guidance
- [NCCL documentation and tests (github.com/NVIDIA/nccl)](https://github.com/NVIDIA/nccl) — read the README, run `nccl-tests` to understand bandwidth vs latency tradeoffs
- [MPI Tutorial (mpitutorial.com)](https://mpitutorial.com) — free, hands-on, covers all collective operations with code examples
- **"A Communication-Efficient Distributed Deep Learning on Sparse Neural Networks"** — read for context on gradient compression as an alternative
- [NVIDIA Megatron-LM](https://github.com/NVIDIA/Megatron-LM) — read the parallelism strategy in the paper; helps understand 3D parallelism communication patterns
- [GPU Interconnects: NVLink architecture overview (NVIDIA)](https://www.nvidia.com/en-us/data-center/nvlink/) — technical overview

### Key Exercises

- **Implement ring-AllReduce from scratch** in Python (no MPI, no NCCL). Use sockets. Given N processes with a vector each, produce the summed vector on all processes. Verify correctness by comparing to a naive approach. Measure bandwidth utilization as a function of message size and number of processes.
- **Implement tree-AllReduce** and compare its latency vs ring-AllReduce on small messages (< 1KB) and large messages (> 1MB). Write up which wins where and why.
- **MPI collective comparison** — using `mpi4py`, benchmark `MPI_Allreduce`, `MPI_Reduce_scatter`, and `MPI_Allgather` at different message sizes. Plot the results. This is what NCCL is built on.
- **NCCL bandwidth test** — run `nccl-tests/build/all_reduce_perf` between two GPUs (or simulate with CPUs). Measure bus bandwidth vs theoretical bandwidth. Tune the `NCCL_SOCKET_IFNAME` and `NCCL_DEBUG=INFO` environment variables to understand transport selection.
- **PCIe topology analysis** — use `nvidia-smi topo -m` on a multi-GPU machine to map the interconnect topology. Identify which GPU pairs go through the PCIe switch vs NVLink. Write a training loop that explicitly moves tensors between different GPU pairs and measure the bandwidth difference.
- **Simulate a slow AllReduce** — in a distributed PyTorch setup, use `nccl_debug` and `torch.profiler` to identify the AllReduce as a bottleneck. Try `gradient_as_bucket_view=True` and measure the impact on overlap between compute and communication.

---

## Phase 5 — Datacenter & WAN Networking (4 weeks, OpenAI-track focus)

More relevant to the OpenAI JD. Anthropic's role is more focused on the host-level stack, but datacenter topology knowledge helps at both.

| Topic | Why it matters |
|-------|---------------|
| Clos / Fat-tree topology | The dominant topology for AI training cluster fabrics; OpenAI explicitly mentions topology design |
| BGP in the datacenter: eBGP between racks, ECMP load balancing | How modern datacenters route without STP; eBGP is the de-facto routing protocol at scale |
| ECMP (Equal Cost Multi-Path): hashing, flow pinning, link failures | How a single flow gets a consistent path; critical for RDMA which breaks under rebalancing |
| VXLAN and overlay networking | How VMs and containers get L2 adjacency across a routed fabric |
| Network telemetry: INT (In-band Network Telemetry), streaming telemetry with gRPC | Observability at scale — how you detect the congestion that degrades training |
| WAN technologies: MPLS, SD-WAN, inter-DC BGP peering | Context for the global WAN/edge component of the OpenAI JD |
| Failure domain design: blast radius, redundancy, link aggregation (LACP) | Reliability design — how you ensure a single switch failure doesn't halt a 10,000-GPU job |
| Network automation: Ansible/Nornir for network config, YANG/NETCONF/gRPC | How you manage thousands of switches without manual CLI |

### Resources

- **BGP in the Data Center — Lapukhov & Doering** (O'Reilly, free PDF) — the definitive guide to datacenter BGP; written by the Facebook/Meta engineers who pioneered it
- **Fundamentals of Data Center Networking (NANOG talks)** — search NANOG.org for talks on Clos topology and BGP design
- **Network Warrior — Gary Donahue** — practical, operational networking reference
- [Arista Networks Tech Library](https://arista.my.salesforce.com/sfc/p/3R000000YVnI/) — real datacenter switch configuration examples
- **"Jupiter Rising: A Decade of Clos Topologies and Centralized Control in Google's Datacenter Network"** (SIGCOMM 2015) — how Google scaled their datacenter fabric; directly applicable to AI cluster design

### Key Exercises

- **Design a fat-tree topology** — for a 64-node cluster, design the Clos topology. Calculate the number of switches, uplinks, and downlinks. Calculate theoretical bisection bandwidth. Verify your design provides full bisection bandwidth.
- **BGP lab with FRRouting** — spin up 4 VMs, configure eBGP between them with FRRouting. Advertise prefixes, verify route propagation, simulate a link failure and watch BGP reconverge.
- **ECMP flow hashing experiment** — with 4 paths between two endpoints, use `iperf3 -P 10` (10 parallel flows) and `ss` to observe which flows land on which paths. Then observe what happens to an RDMA flow (single flow) — it gets only 1/4 of the bandwidth.
- **Build a simple network topology monitor** — use SNMP or gNMI to poll interface counters from a network device (or a simulated one), store in a time-series DB (Prometheus), and alert when a counter exceeds a threshold.

---

## Full Learning Order

```
Weeks 1–4:   Phase 0 — Networking Fundamentals (Kurose & Ross + Beej's sockets)
Weeks 5–10:  Phase 1 — Linux Kernel Networking (XDP tutorial + io_uring)
Weeks 11–16: Phase 2 — Systems Programming (Rust book OR K&R C + CS:APP)
Weeks 17–24: Phase 3 — RDMA + DPDK (RDMA ping-pong + DPDK l2fwd)
Weeks 25–30: Phase 4 — ML/AI Networking (ring-AllReduce + NCCL)
Weeks 31–34: Phase 5 — Datacenter (BGP lab + fat-tree design)
Ongoing:     Read papers, contribute to open source, instrument real systems
```

---

## Key Papers to Read (in order)

| Paper | Why |
|-------|-----|
| "A TCP/IP Tutorial" — RFC 1180 | The clearest single document explaining TCP/IP; read before any kernel work |
| "Congestion Control for Large-Scale RDMA Deployments" (Zhu et al., SIGCOMM 2015) | DCQCN — the congestion control algorithm used in most RoCE deployments |
| "Horovod: fast and easy distributed deep learning in TensorFlow" (Sergeev & Del Balso, 2018) | How ring-AllReduce is implemented in practice |
| "Jupiter Rising" (Singh et al., SIGCOMM 2015) | Google's Clos datacenter fabric; directly analogous to AI cluster networking |
| "FaSST: Fast, Scalable and Simple Distributed Transactions" (Kalia et al., OSDI 2016) | How RDMA is used to build distributed systems with low latency |
| "Efficient Large-Scale Language Model Training on GPU Clusters Using Megatron-LM" (Narayanan et al., SC 2021) | How 3D parallelism changes communication patterns |
| "SHARP: An Architecture for In-Network Compute" (Graham et al., 2016) | In-network AllReduce — the cutting edge of ML networking hardware |

---

## Projects

These are the projects that signal real depth to a hiring committee. Each one requires implementing something non-trivial at the systems level.

### Project N1: Ring-AllReduce Implementation

Implement ring-AllReduce from scratch using raw POSIX sockets in Python or C.

- N processes, each holding a shard of a gradient vector
- Scatter-reduce phase: each process sends to its right neighbor, accumulates partial sums
- Allgather phase: propagate reduced chunks around the ring
- Verify correctness against a naive all-gather + sum approach
- Benchmark bandwidth efficiency as a function of N and message size
- Compare to tree-based reduce at small message sizes

This is the most direct signal for the Anthropic ML Networking role. If you can implement this correctly and explain the bandwidth vs latency tradeoff in different collective algorithms, you will pass the technical screen.

### Project N2: XDP Packet Filter and Counter

Write an XDP program in C that:
- Attaches to a network interface at the driver hook point
- Counts packets per (src IP, dst port) pair using a BPF hash map
- Drops packets from a configurable blocklist (loaded via a BPF map from userspace)
- Exposes counters to userspace via a BPF map read loop every 100ms

Use libbpf (not BCC) — this forces you to understand the real compilation and loading pipeline.

This demonstrates hands-on Linux kernel programming, which is table-stakes for Anthropic ML Networking.

### Project N3: RDMA Bandwidth Benchmark

Using `libibverbs`, write a client-server program that:
- Establishes an RC (Reliable Connected) queue pair
- Benchmarks RDMA Write throughput as a function of message size (256B → 4MB)
- Benchmarks RDMA Send/Recv latency with a ping-pong test
- Plots throughput (Gb/s) and latency (µs) curves

Compare your results to `perftest` (`ib_write_bw`, `ib_send_lat`). If they don't match, find out why.

### Project N4: Network Performance Monitoring Tool (eBPF)

Build a tool that tracks per-flow network latency using eBPF:
- Instruments `tcp_sendmsg` and `tcp_recvmsg` kprobes
- Computes the time from send to ACK for each TCP flow
- Outputs a histogram of latency per destination IP in real time
- Written in Rust using `aya` (Rust eBPF library) — exercises Phase 2 + Phase 1 together

This is the kind of observability tooling both JDs describe: "build tooling and observability systems for network health, performance analysis."

### Project N5: Simple Congestion Controller

Implement a congestion control algorithm in userspace:
- Build a simple stop-and-wait reliable protocol over UDP
- Implement a basic AIMD (Additive Increase Multiplicative Decrease) congestion controller
- Simulate packet loss and delay with `tc netem`
- Plot throughput and window size over time as you vary loss rate
- Compare to TCP Cubic and TCP BBR by running iperf3 in parallel

This demonstrates algorithmic understanding of congestion control — directly referenced in both JDs.

---

## Technology Checklist (by JD)

### Anthropic ML Networking
- [ ] eBPF programs (write, compile, load, attach)
- [ ] XDP hook programs
- [ ] `io_uring` async I/O
- [ ] `epoll` multiplexing server
- [ ] DPDK: poll mode driver, `rte_mbuf`, `rte_ring`
- [ ] RDMA: verbs API, memory registration, completion queues
- [ ] Ring-AllReduce and tree-AllReduce implementation
- [ ] RPC design (build a simple one over TCP)
- [ ] Rust: ownership, unsafe, async with Tokio
- [ ] PCIe topology (conceptual + `lspci` tooling)

### OpenAI Core Network Engineering
- [ ] Linux networking tools (`tc`, `ip`, `ss`, `perf`)
- [ ] RDMA/RoCE: PFC, ECN, DCQCN
- [ ] InfiniBand: subnet manager, GUIDs, partition keys
- [ ] BGP in the datacenter
- [ ] Clos / fat-tree topology design
- [ ] ECMP and flow pinning
- [ ] Network automation (Ansible + NAPALM or Nornir)
- [ ] Observability: gNMI/SNMP telemetry, Prometheus
- [ ] Distributed systems debugging methodology
- [ ] C++ or Go for systems tooling

---

*Notes and exercises will be added below as I work through this section.*
