# Fortnite core-affinity layout — i9-9900 (8C/8T, HT OFF)

Optimized configs for ProcRipper Config Tool v3.0.0, tuned for a CPU where
device interrupts are already pinned to specific cores. Affinity is expressed as
a **hex bitmask** (see below), the form the tool itself emits.

Files in this folder:

| File | Goes into | Purpose |
|------|-----------|---------|
| [`FortniteClient.gcfg`](./FortniteClient.gcfg) | `GAME_PRIORITY.GCFG` | the Fortnite per-thread block (paste over the existing one) |
| [`PROC_PRIORITY.gcfg`](./PROC_PRIORITY.gcfg) | `PROC_PRIORITY.GCFG` | the full system/background config, rebuilt to keep every process off the clean game cores |
| [`PROC_PRIORITY-review.md`](./PROC_PRIORITY-review.md) | — | what changed in the system config and why |

## Affinity bitmask reference

Affinity is a hex bitmask where **bit N = core N** (the tool already uses this —
`0X2` appears in `audiodg`). This is the only way to target the *non-contiguous*
device cores in a single field.

| Cores | Bitmask |
|-------|---------|
| 0 | `0X1` |
| 1 | `0X2` |
| 2 | `0X4` |
| 3 | `0X8` |
| 4 | `0X10` |
| 5 | `0X20` |
| 6 | `0X40` |
| 7 | `0X80` |
| **0,2,4,6** (all device cores) | **`0X55`** |
| **1,3,5,7** (all clean cores) | **`0XAA`** |
| all 8 | `0XFF` |

> Verify once before trusting it: set a single thread to `0X55`, launch, and
> confirm in Task Manager / Process Lasso that it sits on cores 0,2,4,6. If your
> build rejects hex masks, the previous single-core/range forms are the fallback.

## Your hardware / starting point

- i9-9900, **8 physical cores, Hyper-Threading OFF** → logical CPUs `0`–`7`.
- Device IRQ / affinity already pinned by you:

| Core | Device pinned here |
|------|--------------------|
| 0 | Ethernet / NIC |
| 2 | USB |
| 4 | Audio controller |
| 6 | GPU |

**Core 5 is free** (no audio controller after all), so there are now **four
clean cores with no device IRQ load: 1, 3, 5, 7.** That's exactly enough to give
each of the four heavy engine threads its own core.

## The layout

| Core | Device | Fortnite threads pinned here | Why |
|------|--------|------------------------------|-----|
| Core | Device | Mask | Fortnite threads pinned here |
|------|--------|------|------------------------------|
| **0** | Ethernet | `0X1` | `RtcNetworkThread`, `RtcWorkerThread`, `OnlineAsyncTaskThreadMcp`, `ThreadedTickWebSocketThread` (+ background) |
| **1** | — (clean) | `0X2` | **`GameThread` (alone)** |
| **2** | USB | `0X4` | `WindowsRawInputThread` (+ background) |
| **3** | — (clean) | `0X8` | **`RenderThread 0` (alone)** |
| **4** | Audio | `0X10` | audio mixer + XAudio (+ background) |
| **5** | — (clean) | `0X20` | **`RHISubmissionThread` (alone)** |
| **6** | GPU | `0X40` | `RHIInterruptThread` (+ background) |
| **7** | — (clean) | `0X80` | **`RHIThread` (alone)** |

Background/worker pool → `0X55` (cores 0,2,4,6), so it can use any device core
but never a clean game core.

### What goes on core 5

`RHISubmissionThread` is the best resident for the freed core. It's the UE5
parallel GPU command-submission worker — it does genuine per-frame work, so it
benefits from an isolated core, and moving it off the shared GPU core (6) leaves
core 6 with only the (mostly-sleeping) `RHIInterruptThread` plus the GPU IRQ.

That puts the **four heavy pipeline threads — GameThread, RenderThread 0,
RHISubmissionThread, RHIThread — each alone on a clean core (1/3/5/7)**, which is
the cleanest possible result for your chip.

Alternative if you care more about input jitter than GPU-submit isolation: put
`WindowsRawInputThread` on core 5 instead and leave `RHISubmissionThread` on the
GPU core 6. I did **not** do this because the input thread is tiny and gains more
from sitting on the USB IRQ core (2) than from a whole dedicated core.

## Syntax

Thread lines are **comma-delimited** with a fixed 9-field shape:

```
ThreadName=Priority,Affinity,DisableBoost,-1,-1,-1,0,False,False   (9 fields)
```

The Affinity is a **single token**. With the hex bitmask we no longer need
ranges/GROUPs or the (invalid) comma list `0,2,4,6` — `0X55` is one token that
means cores 0,2,4,6. Every row in both files was validated to keep exactly the
right field count (9 for thread lines, 14 for process headers).

Why the bitmask is better here: your device cores (0,2,4,6) are non-contiguous,
so a range (`4-6`) can't express them and a comma list breaks the row. A bitmask
has neither problem — it's the tool's native affinity form and the cleanest fit
for this layout.

---

# Investigation: why GameThread / RenderThread / input won't hold priority 15

> You asked me to find the cause, not to fix it — so this section is diagnosis
> only. No priorities were changed; the engine threads are still left at `15` in
> the config.

**Short version:** affinity (core pinning) sticks, but the *priority* of those
three specific threads doesn't, because **Unreal Engine sets and re-applies their
priority from inside the process**, and the engine wins over a one-shot external
write. A secondary factor is **Easy Anti-Cheat** stripping handle rights.

### 1. The engine actively re-applies priority on exactly these threads

GameThread, RenderThread, and the input/message path are the threads UE manages
itself, and it re-asserts their priority repeatedly:

- The **GameThread runs the Windows message pump** (`FWindowsPlatformApplicationMisc::PumpMessages`).
  On focus transitions that code path calls `SetThreadPriority(GetCurrentThread(), …)`
  to drop/restore priority. Whatever your tool wrote gets overwritten the next
  time the pump runs.
- The **RenderThread** priority is owned by the engine via the CVar
  `r.RenderThreadPriority` (and `r.RHIThread.Priority` for the RHI thread). It's
  applied at thread creation and re-applied when those CVars/values are touched.
  The render thread is also **destroyed and recreated on device/resolution
  changes** (alt-tab, fullscreen toggle, res change) — the new thread comes up at
  the engine's default priority with a new thread ID, so any external pin is lost.
- The **input thread** is driven off this same engine-managed message/focus path.

By contrast, the **worker-pool and RHI/submission threads are set once at
creation and not re-pumped**, which is why *their* values stick — and why you
see the failure only on game/render/input.

UE caps these at its own configured level too: it never runs GameThread/
RenderThread at `TIME_CRITICAL`. Even when the write lands, the engine pulls them
back to its level (Normal / AboveNormal ≈ base 8–10), so you never observe a
steady `15`.

References: UE `WindowsPlatformApplicationMisc.cpp` `PumpMessages` focus/priority
logic; CVars `r.RenderThreadPriority`, `r.RHIThread.Priority`, `TaskGraph.TaskThreadPriority`.

### 2. Easy Anti-Cheat strips the handle rights (secondary)

Fortnite is protected by **Easy Anti-Cheat** (and BattlEye). EAC's kernel driver
registers `ObRegisterCallbacks`, which runs *before* a handle to the game is
created and **strips access rights** (e.g. `THREAD_SET_INFORMATION`) from
user-mode handles — even for an Administrator. `SetThreadPriority` then fails with
**Access Denied** (the same reason raising Fortnite's priority in Task Manager
gives "Access Denied"). If your tool isn't running with a kernel helper that EAC
tolerates, some priority writes simply never apply.

This is consistent with what you see: affinity changes that *do* take are being
applied by a path EAC permits, while the engine-managed priority writes are the
ones that get reverted/blocked.

### Why affinity still works but priority doesn't

Affinity is set once by the engine at thread creation and isn't continuously
re-asserted on the message-pump/focus path, so an external pin survives. Priority
on game/render/input is on that hot re-application path, so it's repeatedly
overwritten. That's the core asymmetry — and the reason this layout leans on
**affinity** (which holds) rather than priority (which doesn't) to isolate the
important threads.

*(No fix applied, per your request. If you later want to chase it: the engine-side
levers are the CVars above; the anti-cheat side can't be worked around without
defeating EAC, which would risk a ban.)*
