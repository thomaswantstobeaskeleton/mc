# Fortnite core-affinity layout — i9-9900 (8C/8T, HT OFF)

Optimized `[FortniteClient-Win64-Shipping.exe ...]` block for `GAME_PRIORITY.GCFG`
(ProcRipper Config Tool v3.0.0 format), tuned for a CPU where device interrupts
are already pinned to specific cores.

Drop-in file: [`FortniteClient.gcfg`](./FortniteClient.gcfg) — paste it over the
existing Fortnite block in your `GAME_PRIORITY.GCFG`.

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
| **0** | Ethernet | `RtcNetworkThread`, `RtcWorkerThread`, `OnlineAsyncTaskThreadMcp`, `ThreadedTickWebSocketThread` | Network threads on the NIC IRQ core, off the game cores |
| **1** | — (clean) | **`GameThread` (alone)** | Heaviest thread, fully isolated |
| **2** | USB | `WindowsRawInputThread` | Input thread on the USB IRQ core → lowest input latency |
| **3** | — (clean) | **`RenderThread 0` (alone)** | Second-heaviest thread, fully isolated |
| **4** | Audio | audio mixer + XAudio + background pool | Audio threads on the audio IRQ core |
| **5** | — (clean, newly free) | **`RHISubmissionThread` (alone)** | GPU command-submission worker — real per-frame work, now isolated instead of sharing the GPU core |
| **6** | GPU | `RHIInterruptThread` + background pool | Interrupt thread next to the GPU IRQ; it mostly sleeps on a GPU fence so the wake-up is same-core |
| **7** | — (clean) | **`RHIThread` (alone)** | GPU-facing engine thread, fully isolated |

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

## Background pool: `0,2,4,6`

The background/worker pool is pinned to the even (device) cores `0,2,4,6` so it
can never touch the clean cores `1/3/5/7`. At IDLE priority (`-15`) it only runs
when those device cores have nothing else to do, so it won't disturb audio, GPU,
network, or input.

`0,2,4,6` is listed verbatim as a valid affinity in the format's own docs
("Use only even cores"), so it's used directly. Note that `.GCFG` thread lines
are otherwise comma-delimited; **if your specific build rejects a comma list in a
per-thread affinity field, replace `0,2,4,6` on those lines with a single device
core** (e.g. `4` or `6`). Single cores and ranges (`6-7`) are always accepted.

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
