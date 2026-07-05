# Linux Kernel — World Slice Guide

## Why World Slice?

Linux 7.2-rc1 has **63,768 .c/.h files, ~32M lines of code**.
No one can read it all. No one should have to.

Traditional approach: load the entire kernel source → grep every line → answer.
99% of tokens/time wasted on irrelevant code.

**World Slice approach**: compress each subsystem into a minimal, reason-able world model.

> A 500-line WS answers ~80% of design questions about that subsystem, at call-stack resolution.
> No source code needed. No grep. Just copy, paste, ask.

---

## How to Use

Copy a WS file into any LLM (ChatGPT, Claude, DeepSeek...), then ask your question.

### Example: Single subsystem

```
I have the following Linux kernel world model.

<paste ws-init.yaml>

Question: What is the execution order of mm_core_init()
and sched_init() inside start_kernel?
Why is this order necessary?
```

### Example: Cross-subsystem

```
I have three World Slices:

<paste ws-vfs-block.yaml>
<paste ws-fs.yaml>
<paste ws-net.yaml>

Question: From write() to data landing on SSD,
which kernel subsystems does it pass through?
```

### Example: When WS is not enough

Submit an expansion request:

```
WORLD_SLICE_REQUEST:
  question: "How does an eBPF program attach to a kernel function?"
  scope: module-scoped
  strictness: minimal
  target:
    - "kernel/bpf/"
    - "include/linux/bpf.h"
```

---

## World Slice Inventory

| WS | Lines | Topic | Questions it answers |
|:---|:-----:|:------|:---------------------|
| **ws-init.yaml** | 528 | Boot: start_kernel → PID 1 | How is PID 1 born? initcall order? |
| **ws-minimal-linux.yaml** | 654 | Core: boot + sched + MM + process | How do the 4 core mechanisms connect? |
| **ws-locking.yaml** | 399 | Sync: spinlock → mutex → rwsem | spin_lock vs mutex_lock — when to use which? |
| **ws-rcu.yaml** | 371 | RCU: read-side + grace period | Why is the reader almost free? |
| **ws-vfs-block.yaml** | 502 | VFS read: syscall → page cache → bio | Full path from read() to disk? |
| **ws-fs.yaml** | 527 | ext4 write: buffered → journal → writeback | When does write() data enter the journal? |
| **ws-net.yaml** | 576 | TCP/IPv4: socket → handshake → data | State machine from connect() to ESTABLISHED? |

## Compression Ratio

| Subsystem | Source files | WS lines | Ratio |
|:----------|:-----------:|:--------:|:-----:|
| init (boot flow) | 13 | 528 | 1:~12 |
| kernel + mm | ~840 | 654 | 1:~600 |
| ext4 + JBD2 | ~200 | 527 | 1:~150 |
| TCP/IPv4 | ~300 | 576 | 1:~200 |

## Design Principles

1. **Every fact needs PROV** — no source line number = doesn't belong
2. **Minimality** — don't add unless removal breaks downstream AI reasoning
3. **Expansion over modification** — too narrow? point MissingScope to a new WS
4. **Technician Caveman style** — short sentences, A → B → C causal chains

## Status

**Built (7 completed):** ws-minimal-linux, ws-init, ws-locking, ws-rcu, ws-vfs-block, ws-fs, ws-net
Total: 3,557 lines

**Not yet built (roadmap):** arch/x86, block (full), security, crypto, ipc, lib, drivers (device model), include/
