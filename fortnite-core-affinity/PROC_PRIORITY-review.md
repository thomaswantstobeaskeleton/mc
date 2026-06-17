# `PROC_PRIORITY.gcfg` — system/background config (bitmask rebuild)

This is the **system/background** process config, rebuilt so every process is
confined to the **device cores (0,2,4,6 = `0X55`)** and can never touch the
**clean game cores (1,3,5,7)**. Drop-in file: [`PROC_PRIORITY.gcfg`](./PROC_PRIORITY.gcfg)
— it replaces your whole `PROC_PRIORITY.GCFG`.

## Why this was needed

The original config was generated for a different core scheme that collided with
the Fortnite layout:

- Background / "powersave" processes were parked on cores **0–1** (`...,0,1,...`
  and `GROUP:POWERSAVE`) — but core **1 is your GameThread**.
- DWM threads, several at **priority 15**, were pinned to cores **3–5**
  (`GROUP:DWM` / `...,3,5,...`) — but cores **3 and 5 are your RenderThread and
  RHISubmission**.

So three of your four isolated game cores were being used by the desktop
compositor and system housekeeping. DWM was the worst case because it ran
priority-15 threads straight onto your render-pipeline cores.

## What changed

A single, uniform transformation across all 89 process blocks:

- **Affinity → `0X55`** (cores 0,2,4,6) on every process header *and* every
  thread line. Previously these were `ALL`, `AUTO`, `GROUP:DWM`, `GROUP:POWERSAVE`,
  ranges, or `0X2`.
- **Park range fields zeroed** (the `start,end` pair after `DisableBoost`/`GpuPriority`
  set to `-1,-1`), since the bitmask now defines the cores directly — matching how
  the tool writes explicit-affinity rows.
- **Everything else left untouched**: per-process/per-thread **priorities**,
  `DisableBoost`, GPU-priority (`none`/`idle`/`below_normal`), the `module=` lines,
  and the `[DisableBoost]` list at the bottom.

Net effect: all system/background work is corralled onto the device cores, off
your game cores — including the DWM priority-15 threads, which was the real
problem.

Validation: 89 headers (14 fields each), 599 thread lines (9 fields each), 62
`module=` lines and 25 `[DisableBoost]` names preserved; no `ALL`/`AUTO`/`GROUP`/
range tokens remain.

## Notes & options

- **Idle vs. confined.** Most of these were already priority `-15` (IDLE), so
  they'd yield to your priority-15 game threads even on a shared core. Confining
  them to `0X55` is the belt-and-suspenders version: they now *physically can't*
  land on a clean core.
- **Kernel/pseudo processes** (`System`, `Idle`, `Registry`, `smss`) were
  transformed for consistency, but affinity on these is often ignored by Windows —
  harmless either way.
- **Finer pinning (optional).** Uniform `0X55` lets every system process roam the
  four device cores. If you'd rather match services to their device IRQ core, use
  these masks instead of `0X55` on the relevant blocks:
  - audio services (`audiodg`, svchost `AudioSrv`/`AudioEndpointBuilder`, Discord
    audio threads) → `0X10` (core 4)
  - network services (svchost `Tcpip`/`Dnscache`/`Dhcp`/`dnsrslvr`) → `0X1` (core 0)
  - HID (svchost `hidserv`) → `0X4` (core 2)
- **`audiodg → AudioRenderThread`** was `0X2` (= core 1, your GameThread) in the
  original — a real misfire. It's now `0X55` (and could be `0X10` for the audio
  core specifically).

See [`README.md`](./README.md) for the bitmask reference table and the Fortnite
side of the layout.
