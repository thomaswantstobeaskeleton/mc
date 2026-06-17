# Fortnite core-affinity layout — i9-9900 (8C/8T, HT OFF)

This folder contains an optimized `[FortniteClient-Win64-Shipping.exe ...]`
block for `GAME_PRIORITY.GCFG` (ProcRipper Config Tool v3.0.0 format),
tuned for a CPU where device interrupts are already pinned to specific cores.

Drop-in file: [`FortniteClient.gcfg`](./FortniteClient.gcfg) — paste it over
the existing Fortnite block in your `GAME_PRIORITY.GCFG`.

## Your hardware / starting point

- i9-9900, **8 physical cores, Hyper-Threading OFF** → logical CPUs `0`–`7`.
- Device IRQ / affinity already pinned by you:

| Core | Device pinned here |
|------|--------------------|
| 0 | Ethernet / NIC |
| 2 | USB |
| 4 | Audio controller |
| 5 | Audio controller |
| 6 | GPU |

That leaves cores **1, 3, 7** with *no* device interrupt load — these are the
cores worth dedicating to the heaviest game threads.

## The goal

> Make the threads that actually matter in Fortnite each own a core (or at
> least never fight each other). Background threads may share cores.

In Unreal Engine the frame is driven by a short pipeline of heavy,
latency-critical threads. The three that dominate frametime and should each get
an isolated core are:

- **GameThread** – gameplay/simulation tick (usually the CPU bottleneck).
- **RenderThread 0** – builds the render command stream.
- **RHIThread** – translates/submits to the D3D driver (GPU-facing).

Everything else is either tiny-but-latency-sensitive (input, GPU submit/interrupt,
audio) or genuinely background (worker pools, async loading, media, heartbeats).

## The layout

| Core | Device | Fortnite threads pinned here | Why |
|------|--------|------------------------------|-----|
| **0** | Ethernet | `RtcNetworkThread`, `RtcWorkerThread`, `OnlineAsyncTaskThreadMcp`, `ThreadedTickWebSocketThread` | Network threads sit on the NIC IRQ core → lowest net latency, off the game cores |
| **1** | — (clean) | **`GameThread` (alone)** | Heaviest thread, fully isolated, zero device IRQs |
| **2** | USB | `WindowsRawInputThread` | Input thread on the same core as USB IRQs (mouse/kbd) → lowest input latency |
| **3** | — (clean) | **`RenderThread 0` (alone)** | Second-heaviest thread, fully isolated |
| **4** | Audio | Audio mixer + XAudio + background pool | Audio threads on the audio IRQ cores |
| **5** | Audio | Audio mixer + XAudio + background pool | " |
| **6** | GPU | `RHISubmissionThread`, `RHIInterruptThread` + background pool | GPU-facing tail next to the GPU IRQ; `RHIInterruptThread` mostly sleeps waiting on a GPU fence, so the wake-up is same-core (no cross-core IPI) |
| **7** | — (clean) | **`RHIThread` (alone)** | Third heavy thread, fully isolated |

The "big three" (`GameThread` → 1, `RenderThread 0` → 3, `RHIThread` → 7) each
own an entire clean core and never share with each other or with device IRQs.
The background/worker pool is confined to cores **4-6** so it can never steal a
cycle from cores 1/3/7. At priority `-15` (IDLE) it only runs when audio/GPU
don't need those cores, so it won't disturb sound or GPU submission.

## Why the background pool is `4-6` (and not `0,2,4,5,6`)

The `.GCFG` per-thread lines are **comma-delimited** (`Name=prio,affinity,boost,…`),
so a comma inside the affinity field is parsed as the next field. The tool's own
output only ever uses a **single core** (`1`), a **contiguous range** (`6-7`,
`0-1`), `ALL`, or `GROUP:` for this reason — never `0,2,4,6`.

The only contiguous range that covers several device cores while avoiding the
three reserved cores `1/3/7` is `4-6`. So that's the background pool. If you
have confirmed your build of the tool accepts comma masks in thread lines, you
can widen the pool to `0,2,4,5,6` (every device core) by find/replacing `4-6`
with `0,2,4,5,6` in the file.

## Notes / optional extras

- **Priorities are unchanged** from your config — only affinities were
  re-mapped. The engine threads stay at `15` (TIME_CRITICAL); workers stay
  negative. (Priority boost is a no-op at level 15, so the `DisableBoost` flags
  were left as you had them.)
- **Process affinity stays `ALL`.** On Windows a thread's affinity must be a
  subset of the process affinity, so the process mask has to include cores
  1/3/7 for the per-thread pins to take. Isolation is achieved by pinning the
  critical threads to 1/3/7 and keeping everything else on 0/2/4/5/6.
- **Unlisted threads** still inherit the process mask (`ALL`) and can briefly
  land on 1/3/7, but they're low-priority and get preempted by the priority-15
  engine threads, so they don't meaningfully contend.
- Optional: set the process-level GPU priority to `high` (the `none` field in
  the header) if you want ProcRipper to also bump Fortnite's GPU scheduling.
