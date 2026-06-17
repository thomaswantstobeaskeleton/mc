# Review of `PROC_PRIORITY.GCFG` vs. the Fortnite core layout

This is the **system/background** process config. The goal of the Fortnite
layout is to keep cores **1, 3, 5, 7 clean** for the game's heavy threads. For
that isolation to actually hold, the OS/background processes in *this* file must
also stay **off** those cores. They don't right now.

Reminder of the core map:

| Core | Role |
|------|------|
| 0 | Ethernet / NIC — also the general "housekeeping" core |
| 1 | **CLEAN — GameThread** |
| 2 | USB |
| 3 | **CLEAN — RenderThread 0** |
| 4 | Audio |
| 5 | **CLEAN — RHISubmissionThread** |
| 6 | GPU |
| 7 | **CLEAN — RHIThread** |

Device cores = `0, 2, 4, 6`. Clean (game) cores = `1, 3, 5, 7`.

## The big problem: this config targets a different core scheme

It was generated for a layout where:

- **cores 0–1** = background / "powersave" dump, and
- **cores 3–5** = DWM "realtime".

You can see it in the trailing fields. On almost every line the pair after
`DisableBoost` is the **park core range** `start,end`:

```
amdsasrv64.dll+0x9D7BC=-15,ALL,True,0,1,-1,0,False,False     <- park 0-1
DWM Master Input Thread=15,GROUP:DWM,False,3,5,-1,2,False,False  <- cores 3-5, prio 15
DWM LPC Port Thread=-15,GROUP:POWERSAVE,True,0,1,-1,1,False,False <- cores 0-1
```

Mapped onto your Fortnite layout, that means:

| This config puts… | …on core | which is your |
|-------------------|----------|---------------|
| all parked background processes (`0,1`) | 1 | **GameThread** |
| DWM realtime threads (`3,5`, prio 15) | 3 | **RenderThread 0** |
| DWM realtime threads (`3,5`, prio 15) | 5 | **RHISubmissionThread** |

So 3 of your 4 isolated cores are being used by the desktop compositor and
system housekeeping. **DWM is the worst case** because several of its threads are
**priority 15** — they will preempt your render pipeline on cores 3 and 5.

## What actually needs fixing (by severity)

### 1. DWM — REQUIRED. It runs priority-15 threads on your clean cores.

Replace the whole `[dwm.exe ...]` block with the corrected one in
[`dwm.gcfg`](./dwm.gcfg). It retargets every DWM thread to **core 0** (off the
clean cores, and off the sensitive audio/GPU cores). Priorities and flags are
unchanged — only the core targeting is fixed. During fullscreen gaming DWM is
mostly idle anyway, so core 0 is plenty.

### 2. Active (non-idle) system service threads — RECOMMENDED.

Most entries in this file are priority `-15` (IDLE) with `ALL` affinity. Those
are **low risk**: at IDLE priority they're preempted by your priority-15 game
threads even if they land on a clean core. You can leave them, *or* tighten them
(see §3).

The ones that are **not** idle can still steal cycles and are worth pinning to
their matching device core:

| Process / thread | Current | Suggested core | Reason |
|------------------|---------|----------------|--------|
| `svchost.exe` → `AudioEndpointBuilder`, `AudioSrv` (prio 1) | AUTO | **4** (audio) | audio service near audio IRQ |
| `svchost.exe` → `Tcpip`, `Dnscache`, `Dhcp`, `dnsrslvr*` (prio 0–1) | AUTO | **0** (NIC) | network service near NIC IRQ |
| `svchost.exe` → `hidserv.dll*` (prio 1) | ALL | **2** (USB) | HID service near USB IRQ |
| `Discord.exe` → `AudioiceThread`, `AudioMixerThread`, `AudioRenderThread` (prio 2) | AUTO | **4** or **0** | voice audio at HIGHEST; keep off clean cores |
| `obs64.exe` (prio 2, `EncoderThread` prio 2) | AUTO | **6** (GPU) or **0** | only matters if streaming; NVENC lives on GPU |
| `audiodg.exe` → `AudioRenderThread` | `0X2` | **4** (audio) | see §4 |

`AUTO` lets the tool choose, so these may already avoid clean cores — but pinning
them is the only way to *guarantee* it.

### 3. Optional: tighten the idle masses off core 1.

If you want the parked background processes to prefer **core 0 only** (instead of
0–1, which includes your GameThread core), change the park pair `,0,1,-1,` to
`,0,0,-1,` on the affected lines. It's a soft hint while the affinity stays
`ALL`, so the effect is small (idle priority already protects you), but it's
harmless and removes core 1 from their preference.

### 4. Bug to check: `audiodg.exe → AudioRenderThread=-15,0X2,…`

This is the only hex-mask affinity in the file (`0X2`). As a Windows affinity
**bitmask**, `0x2` = **core 1** (your GameThread core); read as an index it'd be
core 2 (USB). Either way it's not the audio core, and it's the system audio
render thread — set it to **core 4** (your audio core). At minimum, get it off
core 1.

## Useful discovery: hex affinity masks may be supported

That `0X2` is important beyond the bug: it suggests the tool accepts a **hex
bitmask** in the affinity field. If so, that's the clean way to express your
*non-contiguous* device cores in a single token (which plain ranges and comma
lists can't do):

| Cores | Bitmask |
|-------|---------|
| 0,2,4,6 (all device cores) | `0x55` |
| 1,3,5,7 (all clean cores) | `0xAA` |
| single core N | `1 << N` → 0:`0x1` 1:`0x2` 2:`0x4` 3:`0x8` 4:`0x10` 5:`0x20` 6:`0x40` 7:`0x80` |

**Please test this first** (set one thread to `0x55` and confirm in Task
Manager / Process Lasso that it lands on cores 0,2,4,6). If it works:

- In **this** file, give all background/system processes affinity `0x55` — that
  pins every system process to the device cores and **guarantees** they never
  touch a clean game core, in one clean edit.
- In the **Fortnite** file, you can replace the round-robin `0/2/4/6` background
  pins with a single `0x55` so the pool can migrate across all device cores.

If `0x55` does **not** work, stick with the single-core / range forms used in the
provided files (those are confirmed-safe).

## Syntax check (you asked)

Good news: the thread lines in *this* file are already well-formed — every row is
`Name=Priority,Affinity,DisableBoost,f4,f5,f6,f7,f8,f9` (9 fields), the affinity
is a single token (`ALL`, `GROUP:NAME`, a range, or the `0X2` hex), and there are
**no comma-list affinities**. So this file confirms the syntax model we used for
the Fortnite block. The corrected `dwm.gcfg` keeps the exact same 9-field shape.
