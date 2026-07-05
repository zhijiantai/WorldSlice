# Linux Kernel — World Slice Guide

## 為什麼需要 World Slice？

Linux 7.2-rc1 有 **63,768 個 .c/.h 檔案、~32M 行原始碼**。沒有人能全部讀完，也沒必要。

傳統做法：每當有一個問題，就 load 整個 kernel source tree → grep 每一行 → 理解上下文 → 回答。過程中浪費 99% 的 token/time 在不相關的程式碼上。

World Slice 的做法：把每個子系統的核心知識壓縮成 **最小可推理世界模型**。

> **一個 500 行的 WS 可以回答該子系統 80% 的設計問題，resolution 到 call stack 層級。**
> 不需要 load 原始碼，不需要 grep。

---

## 怎麼用？

直接複製 WS 檔案內容貼進任何 LLM（ChatGPT、Claude、DeepSeek...），後面接你的問題。

### 範例：問單一子系統

```
我有以下 Linux kernel 的世界模型。

<貼上 ws-init.yaml 內容>

問題：start_kernel 裡 mm_core_init() 和 sched_init() 的執行順序？
為什麼這個順序是必要的？
```

### 範例：跨子系統問題

```
我有以下三個 World Slice：

<貼上 ws-vfs-block.yaml>
<貼上 ws-fs.yaml>
<貼上 ws-net.yaml>

問題：從 write() 到資料寫到 SSD，經過哪些 kernel 子系統？
```

### 範例：當 WS 不夠用時

若問題超出現有 WS 範圍，送出 expansion request：

```
WORLD_SLICE_REQUEST:
  question: "eBPF program 如何 attach 到 kernel function? "
  scope: module-scoped
  strictness: minimal
  target:
    - "kernel/bpf/"
    - "include/linux/bpf.h"
```

---

## World Slice 一覽

| WS | 行數 | 主題 | 能回答的問題 |
|:---|:----:|:-----|:------------|
| **ws-init.yaml** | 528 | Boot flow：start_kernel → PID 1 | PID 1 怎麼誕生的？initcall 順序？ |
| **ws-minimal-linux.yaml** | 654 | 核心：boot + sched + MM + process | 核心四機制？怎麼串的？ |
| **ws-locking.yaml** | 399 | 同步：spinlock → mutex → rwsem | spin_lock vs mutex_lock 何時用哪個？ |
| **ws-rcu.yaml** | 371 | RCU：read-side + grace period | 為什麼 reader 幾乎零成本？ |
| **ws-vfs-block.yaml** | 502 | VFS read：syscall → page cache → bio | read() 到磁碟的完整路徑？ |
| **ws-fs.yaml** | 527 | ext4 write：buffered → journal → writeback | write() 資料何時進 journal？ |
| **ws-net.yaml** | 576 | TCP/IPv4：socket → handshake → data | connect() 到 ESTABLISHED 的狀態機？ |

## 壓縮比

| 子系統 | 原始碼 (檔案數) | WS 行數 | 壓縮比 |
|:-------|:--------------:|:-------:|:------:|
| init (boot flow) | 13 | 528 | 1:~12 |
| kernel + mm | ~840 | 654 | 1:~600 |
| ext4 + JBD2 | ~200 | 527 | 1:~150 |
| TCP/IPv4 | ~300 | 576 | 1:~200 |

## 維護原則

1. **每個 fact 都需要 PROV** — 沒有 source line number = 不該存在
2. **Minimality** — 除非移除會讓下游 AI 無法回答，否則不加
3. **Expansion over modification** — 不夠用就透過 MissingScope 指向新 WS
4. **Technician Caveman style** — 短句、A → B → C 因果鏈

## 現有 WS 狀態

已 built (7 個): ws-minimal-linux, ws-init, ws-locking, ws-rcu, ws-vfs-block, ws-fs, ws-net
Total: 3,557 行

尚未 build (roadmap): arch/x86, block (full), security, crypto, ipc, lib, drivers (device model), include/
