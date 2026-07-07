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

### Annotation tag reference

Every claim in the AI's answer must be tagged with one of:

| Tag | Meaning |
|:----|:--------|
| `[WS(file:L100-L200)]` | Directly stated in the World Slice, with provenance |
| `[WS→INFER]` | Deduced from WS facts by causal chaining |
| `[KNOWLEDGE]` | From the AI's training data, NOT in any WS |

If the AI relies heavily on `[KNOWLEDGE]`, the WS is **too thin** and should be expanded.

### Example: Single subsystem

```
I have the following Linux kernel world model.

<paste ws-init.yaml>

Question: What is the execution order of mm_core_init()
and sched_init() inside start_kernel?
Why is this order necessary?

For each claim in your answer, tag it with:
  [WS(file:L100-L200)] — stated in the WS with provenance
  [WS→INFER] — deduced from WS facts by causal chaining
  [KNOWLEDGE] — from training data, NOT in any WS
```

Expected answer shape (not exact):

```
[WS(ws-init.yaml:L20-L30)]  start_kernel order: mm_core_init() → sched_init()
[WS(ws-init.yaml:L40-L50)]  mm_core_init() builds the page allocator
[WS→INFER]  The scheduler needs struct page to allocate task_struct.
  If sched_init ran first, kmem_cache_alloc would dereference
  uninitialised memory — instant triple fault.
[KNOWLEDGE] The boot CPU stack is hand-wired in arch/x86 before
  C code runs. The WS doesn't cover arch/, but that's outside scope
  because Linux 7.2 boot convention is canonical.
```

### Example: Cross-subsystem

```
I have three World Slices:

<paste ws-vfs-block.yaml>
<paste ws-fs.yaml>
<paste ws-net.yaml>

Question: From write() to data landing on SSD,
which kernel subsystems does it pass through?

For each claim in your answer, tag it with:
  [WS(file:L100-L200)] — stated in the WS with provenance
  [WS→INFER] — deduced from WS facts by causal chaining
  [KNOWLEDGE] — from training data, NOT in any WS
```

Expected answer shape (not exact):

```
[WS(ws-vfs-block.yaml:L30-L40)]  write() → sys_write → vfs_write →
  file->f_op->write_iter → ext4_write_iter
[WS(ws-fs.yaml:L50-L60)]  ext4_write_iter → ext4_buffered_write →
  generic_perform_write → block_write_begin → grab_cache_page_write_begin
[WS(ws-vfs-block.yaml:L80-L90)]  __block_write_begin → submit_bh →
  submit_bio → blk_mq_submit_bio → NVMe driver
[WS→INFER]  The page cache (address_space) sits between VFS and block
  layer. Dirty pages are written back later by flusher threads, not
  during write() syscall.
[KNOWLEDGE]  NVMe driver interrupt path uses MSI-X per-IO-queue.
  Implementation detail not captured in any WS.
```

### Example: When WS is insufficient

```
I have ws-locking.yaml and ws-rcu.yaml:

<paste ws-locking.yaml>
<paste ws-rcu.yaml>

Question: Is spin_lock() starvation-free under PREEMPT_RT?

For each claim in your answer, tag it with:
  [WS(file:L100-L200)] — stated in the WS with provenance
  [WS→INFER] — deduced from WS facts by causal chaining
  [KNOWLEDGE] — from training data, NOT in either WS
```

### Example: Expansion request

Submit when no WS can answer your question:

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
| **ws-net-unexpected.yaml** | 534 | TCP/IPv4: unexpected segments per state | What happens when unexpected SYN/ACK/FIN/RST/DATA arrives in any TCP state? |

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

**Built (8 completed):** ws-minimal-linux, ws-init, ws-locking, ws-rcu, ws-vfs-block, ws-fs, ws-net, ws-net-unexpected
Total: 4,091 lines

**Not yet built (roadmap):** arch/x86, block (full), security, crypto, ipc, lib, drivers (device model), include/
