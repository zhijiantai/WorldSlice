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

### Example: Traceability — Which parts come from the World Slice?

Good AI reasoning requires distinguishing **WS-grounded facts** from **model-internal knowledge**. Ask for this explicitly:

```
I have the following World Slice:

<paste ws-init.yaml>

Question: Why does start_kernel call mm_core_init() BEFORE sched_init()?

For each claim in your answer, mark:
  [WS] = Directly stated in the World Slice
  [WS→INFER] = Deduced from WS facts by causal chaining
  [KNOWLEDGE] = From your training data, NOT in the WS
```

Expected answer shape (not exact):

```
[WS] start_kernel order: mm_core_init() → sched_init()
[WS] mm_core_init() builds the page allocator (struct page, buddy)
[WS→INFER] The scheduler needs struct page to allocate task_struct.
  If sched_init ran first, kmem_cache_alloc would dereference
  uninitialised memory — instant triple fault.
[KNOWLEDGE] The boot CPU stack is hand-wired in arch/x86 before
  C code runs. The WS doesn't cover arch/, but that's outside scope
  because Linux 7.2 boot convention is canonical.
```

This forces the AI to admit **what it knows from the WS** vs **what it fills in from general kernel knowledge**.

### Example: When WS is insufficient

```
I have ws-locking.yaml and ws-rcu.yaml:

<paste ws-locking.yaml>
<paste ws-rcu.yaml>

Question: Is spin_lock() starvation-free under PREEMPT_RT?
Clearly state which parts of your answer rely on:
  [WS] — stated in the model
  [WS→INFER] — chained from model facts
  [KNOWLEDGE] — from training data, NOT in either WS
```

If the AI relies heavily on [KNOWLEDGE], the WS is **too thin** and should be expanded.

### Example: Expansion request

Submit when the WS cannot answer your question:

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
