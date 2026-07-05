# Linux Kernel — World Slice Guide

## Why World Slice? ｜ 為什麼需要 World Slice？

Linux 7.2-rc1 has **63,768 .c/.h files, ~32M lines of code**.
No one can read it all. No one should have to.

Linux 7.2-rc1 有 **63,768 個 .c/.h 檔案、約 3200 萬行原始碼**。沒有人能全部讀完，也沒必要。

Traditional approach: load the entire kernel source → grep every line → answer.
99% of tokens/time wasted on irrelevant code.

傳統做法：每次有問題就 load 整個 kernel source → grep 每一行 → 回答。
浪費 99% 的 token/time 在不相關的程式碼上。

**World Slice approach**: compress each subsystem into a minimal, reason-able world model.

**World Slice 的做法**：把每個子系統的核心知識壓縮成**最小可推理世界模型**。

> **A 500-line WS answers ~80% of design questions about that subsystem, at call-stack resolution.**
>
> **一個 500 行的 WS 可以回答該子系統約 80% 的設計問題，解析度到 call stack 層級。**
>
> No source code needed. No grep. Just copy, paste, ask.
>
> 不需要 load 原始碼，不需要 grep。複製、貼上、提問即可。

---

## How to Use ｜ 怎麼用？

Copy a WS file into any LLM (ChatGPT, Claude, DeepSeek...), then ask your question.

直接複製 WS 檔案內容貼進任何 LLM，後面接你的問題。

### Example: Single subsystem ｜ 範例：單一子系統

```
I have the following Linux kernel world model.

<paste ws-init.yaml>

Question: What is the execution order of mm_core_init()
and sched_init() inside start_kernel?
Why is this order necessary?
```

### Example: Cross-subsystem ｜ 範例：跨子系統

```
I have three World Slices:

<paste ws-vfs-block.yaml>
<paste ws-fs.yaml>
<paste ws-net.yaml>

Question: From write() to data landing on SSD,
which kernel subsystems does it pass through?
```

### Example: When WS is not enough ｜ 範例：當 WS 不夠用時

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

## World Slice Inventory ｜ 一覽

| WS | Lines 行數 | Topic 主題 | Questions it answers 能回答的問題 |
|:---|:----------:|:-----------|:---------------------------------|
| **ws-init.yaml** | 528 | Boot: start_kernel → PID 1 | How is PID 1 born? initcall order? / PID 1 怎麼誕生的？initcall 順序？ |
| **ws-minimal-linux.yaml** | 654 | Core: boot + sched + MM + process | How do the 4 core mechanisms connect? / 核心四機制怎麼串的？ |
| **ws-locking.yaml** | 399 | Sync: spinlock → mutex → rwsem | spin_lock vs mutex_lock — when to use which? / 何時用哪個？ |
| **ws-rcu.yaml** | 371 | RCU: read-side + grace period | Why is the reader almost free? / 為什麼 reader 幾乎零成本？ |
| **ws-vfs-block.yaml** | 502 | VFS read: syscall → page cache → bio | Full path from read() to disk? / read() 到磁碟的完整路徑？ |
| **ws-fs.yaml** | 527 | ext4 write: buffered → journal → writeback | When does write() data enter the journal? / 資料何時進 journal？ |
| **ws-net.yaml** | 576 | TCP/IPv4: socket → handshake → data | State machine from connect() to ESTABLISHED? / connect() 的狀態機？ |

## Compression Ratio ｜ 壓縮比

| Subsystem 子系統 | Source files 原始碼 (檔案數) | WS lines WS 行數 | Ratio 壓縮比 |
|:-----------------|:--------------------------:|:-----------------:|:------------:|
| init (boot flow) | 13 | 528 | 1:~12 |
| kernel + mm | ~840 | 654 | 1:~600 |
| ext4 + JBD2 | ~200 | 527 | 1:~150 |
| TCP/IPv4 | ~300 | 576 | 1:~200 |

## Design Principles ｜ 維護原則

1. **Every fact needs PROV** — no source line number = doesn't belong
   **每個 fact 都需要 PROV** — 沒有 source line number = 不該存在
2. **Minimality** — don't add unless removal breaks downstream AI reasoning
   **Minimality** — 除非移除會讓下游 AI 無法回答，否則不加
3. **Expansion over modification** — too narrow? point MissingScope to a new WS
   **Expansion over modification** — 不夠用就透過 MissingScope 指向新 WS
4. **Technician Caveman style** — short sentences, A → B → C causal chains
   **短句、因果鏈**

## Status ｜ 現有 WS 狀態

**Built (7 completed) 已建立 (7 個):**
ws-minimal-linux, ws-init, ws-locking, ws-rcu, ws-vfs-block, ws-fs, ws-net
Total: 3,557 lines 行

**Not yet built (roadmap) 尚未建立:**
arch/x86, block (full), security, crypto, ipc, lib, drivers (device model), include/
