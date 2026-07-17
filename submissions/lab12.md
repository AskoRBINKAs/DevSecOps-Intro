# Lab 12 — BONUS — Submission

## Task 1: Install + Hello-World

### Host environment
- Kernel (host): `Linux maibook 7.1.3-arch1-2 #1 SMP PREEMPT_DYNAMIC Thu, 09 Jul 2026 19:55:55 +0000 x86_64 GNU/Linux`
- KVM accessible: `crw-rw-rw- 1 root kvm 10, 232 июл 14 19:44 /dev/kvm`
- containerd version: `containerd github.com/containerd/containerd/v2 v2.3.2 fff62f14765df376e5fc36f5a8f8e795b5670f61.m`

### Kata installation
- Kata version: `3.32.0`
- containerd config snippet:
```toml
[plugins.'io.containerd.grpc.v1.cri'.containerd.runtimes.kata]
  runtime_type = 'io.containerd.kata.v2'
```

### Kernel inside containers
**runc:**
```
Linux 5d9feea9013a 7.1.3-arch1-2 #1 SMP PREEMPT_DYNAMIC Thu, 09 Jul 2026 19:55:55 +0000 x86_64 Linux
processor       : 0
vendor_id       : AuthenticAMD
cpu family      : 23
```

**kata:**
```
Linux 63bb6db9cf08 6.18.35 #1 SMP Mon Jun 15 12:55:58 UTC 2026 x86_64 Linux
processor       : 0
vendor_id       : AuthenticAMD
cpu family      : 23
```

### Why the kernel differs (Reading 12)
Reading 12 explains that `runc` containers are isolated with namespaces and cgroups but still execute on the host kernel, while Kata starts the workload inside a lightweight VM with its own guest kernel. That kernel boundary matters for runc-CVE classes such as CVE-2024-21626 ("Leaky Vessels") from Lecture 7 slide 14: a breakout that abuses the shared runtime or host-kernel container boundary can expose the host under `runc`, but under Kata the attacker first lands in the micro-VM. Kata does not make the workload invulnerable, but it changes the blast radius from "host kernel and host filesystem" to "guest VM boundary first."

## Task 2: Isolation + Performance

### Isolation: /dev diff
```
1d0
< core
```

### Isolation: capability sets
runc:
```
CapInh:	0000000000000000
CapPrm:	00000000a80425fb
CapEff:	00000000a80425fb
CapBnd:	00000000a80425fb
CapAmb:	0000000000000000
```
kata:
```
CapInh:	0000000000000000
CapPrm:	00000000a80425fb
CapEff:	00000000a80425fb
CapBnd:	00000000a80425fb
CapAmb:	0000000000000000
```

### Startup time (5-run avg)
| Runtime | Avg startup (s) |
|---------|----------------:|
| runc | 0.583 |
| kata | 1.630 |

**Overhead: ~2.8x cold start.**

### I/O throughput (100MB dd)
| Runtime | Throughput |
|---------|-----------|
| runc | 14.0 GB/s |
| kata | 13.1 GB/s |

### Trade-off analysis (3-4 sentences, Reading 12 framing)
Kata is worth the cost when the workload boundary matters more than raw startup latency, especially for multi-tenant SaaS workloads, untrusted customer code, CI runners, or sandboxed plugin execution. Reading 12 frames the security gain as a separate guest kernel and VM boundary, which directly reduces the blast radius of runc/container-runtime escape classes because the attacker must cross the micro-VM boundary before reaching the host. Kata is less compelling for trusted, single-tenant batch jobs or very short-lived functions where cold-start overhead and operational complexity outweigh the isolation benefit. In this run the `/dev` difference was small and capabilities were identical, so the meaningful security improvement is the kernel/VM isolation boundary rather than a visibly different Linux capability set.