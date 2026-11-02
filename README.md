# Qwen3.8-Flash-Next NVFP4 on 2× RTX 5060 Ti 16 GB + 64 GB RAM (Basalt)

**English** | [한국어](README.ko.md)

A measured, working configuration for running the **NVFP4** Basalt pack of Qwen3.8-Flash-Next (122 GiB file, 63.3 GiB of experts) on a mid-range dual-GPU desktop. **All experts are served from VRAM and pinned RAM, so the SSD is not used as a tier.** No requests were refused up to 103.7K tokens of context.

Shared as a reference. Your numbers will differ with other hardware, engine commits and workloads.

## Hardware and software

| | |
|---|---|
| CPU | AMD Ryzen 5 9600X |
| Board | ASUS ProArt B850-CREATOR WIFI NEO (BIOS 1103) |
| RAM | 64 GB DDR5 (2× 32 GB KLEVV DDR5-5600 CL46, run at 6400 CL30; 61.9 GiB visible to Linux) |
| GPU | 2× PNY GeForce RTX 5060 Ti OC 16 GB (PCIe Gen5 x8 each, CPU lanes, no NVLink), driver 595.91.07, CUDA 13.4 |
| Disk | Samsung PM9A1 NVMe 1 TB |
| PSU | 1050 W 80+ Gold |
| OS | Ubuntu Server 26.04.1 LTS (kernel 7.0), text mode only (`multi-user.target`), no monitor, SSH only |
| Engine | Basalt 0.2.0 (`v0.1.0-119-ga04edb96`, 2026-11-01) |
| Model | `Qwen3.8-FN-NVFP4` pack, **`.basalt` format v2** |
| Workload | one user, long agentic sessions (30K–100K tokens), tool calls, code, 1280×800 screenshots, thinking on |

## `basalt-serve.json`

```json
{
 "server": {
  "model": "/path/to/Qwen3.8-FN-NVFP4.v2.basalt",
  "gpu": "0,1",
  "port": 8080,
  "host": "127.0.0.1",
  "vision": "dynamic",
  "alias": "qwen3.8-flash-next",
  "idle-unload": 1800
 },
 "engine": {
  "expert-cache": "auto",
  "prefill": "auto",
  "kv": "int8",
  "resident-experts": true,
  "max-context": 131072,
  "peer-exclusive": true,
  "peer-reserve-mib": 128,
  "vram-reserve-mib": 256,
  "prompt-cache": 2,
  "adapt-swaps": 48,
  "peer-prefill-rows": 0
 }
}
```

| Setting | Why |
|---|---|
| `resident-experts: true` | Experts that are not on a GPU stay page-locked in RAM, so there are no SSD reads for them. |
| `peer-exclusive: true` | Experts held by GPU1 get no RAM copy. This saves about 15 GiB of RAM and is what makes NVFP4 fit in 64 GB. |
| `max-context: 131072` | Elastic KV borrows fewer expert slots than at 262144. |
| `kv: int8` | Default quality/size trade-off; left as is. |
| `vision: dynamic` | No standing VRAM for the vision tower. |
| `prompt-cache: 2` | Fewer checkpoints than the default 6. In real agent use, prompt reuse was still about 88%. |
| `vram-reserve-mib: 256`, `peer-reserve-mib: 128` | Minimums. GPUs end up 98–99% full. |
| `peer-prefill-rows: 0` | **Required with `peer-reserve-mib: 128`.** Without it, a cold first request whose first chunk is ≤ 5120 tokens makes GPU1 run MMQ prefill. That needs about 83 MiB more and fails with a CUDA OOM. |
| `adapt-swaps: 48` | Halves the RAM kept free for adaptive swaps (1266 → 633 MiB). The speed difference is within run noise. |

Engine startup with this config:
- Peer experts: 5907
- Experts in RAM: 48.68 GiB
- RAM free after load: about 5.0 GiB (guard 3.1 GiB)
- Every lendable slot keeps a RAM copy
- Free VRAM: 193 MiB on GPU0, 19 MiB on GPU1

## BIOS

Settings not listed are Auto/default.

| Area | Setting |
|---|---|
| CPU | PBO Enabled, Curve Optimizer All Core **Negative 40** |
| Memory | DDR5-**6400**, UCLK DIV1 MODE = UCLK=MEMCLK, tCL **30**, tRCD WR 36, tRCD RD 38, tRP 38, tRAS 88, DRAM VDD / VDDQ **1.35 V**, High DRAM Voltage Mode Disabled, Power Down Enable Disabled, Memory Context Restore Disabled |
| Fabric / SoC | Ai Overclock Tuner Manual, BCLK 100.00 MHz, FCLK **2133 MHz**, AI Cache Boost Enabled, CPU SOC Voltage Manual **1.25 V** |
| PCIe | Resizable BAR Enabled |
| Graphics | Primary Video Device = PCIE, **Integrated Graphics = Disabled** (+0.5 GiB RAM from the iGPU's UMA carve-out; the box has no monitor) |

> Memory and CPU values are tuned for these exact parts. Validate stability on your own hardware before using them.

## GPU tuning (applied at every boot)

These settings are not stored on the card. A small oneshot systemd unit (`gpu-tune.service`, ordered before the model service) applies them through NVML (`libnvidia-ml.so.1` via Python ctypes, no extra packages). It does this only for the two known card UUIDs, so a replaced card never gets values validated on another one.

| | GPU0 | GPU1 |
|---|---|---|
| Core clock offset (`SetGpcClkVfOffset`) | **+150 MHz** | **+175 MHz** |
| Memory clock offset (`SetMemClkVfOffset`) | **+750 MHz** (14001 → 14376 MHz) | **+750 MHz** |
| Locked core clock range (`SetGpuLockedClocks`) | 210–2827 MHz | 210–2962 MHz |
| Max clocks after apply (NVML) | core 3360 / mem 14376 MHz | core 3382 / mem 14376 MHz |
| Power limit | 180 W (default) | 180 W (default) |
| Persistence | `nvidia-persistenced` keeps the driver loaded (persistence-mode flag stays Disabled) | same |

- Each card was validated separately, which is why the values differ. The core clock cap is for power efficiency, not for overclocking.
- Effect (same benchmark, one run each): decode +2.1% (118.6 → 121.1 tok/s), Korean text +2.3%, 10K prefill −1.6% (noise). Answers stayed 40/40 and tool calls 5/5, with no Xid errors. The decode gain matches the memory clock gain (+2.7%), because decode is bandwidth-bound.
- Raising the power limit to 198 W gave only +0.4% prefill, so it was rejected. Under this workload the cards draw at most about 126 W.

## OS changes

| Change | Effect |
|---|---|
| Text mode: default target `multi-user.target` (no desktop) | Less RAM, nothing on the GPUs but the model |
| Removed the `crashkernel=` kernel argument; disabled kdump-tools | +1.25 GiB RAM |
| Disabled snapd (+socket, seeded), multipathd (socket masked), networkd-dispatcher, unattended-upgrades, apt-daily timers | ~0.1–0.2 GiB RAM, no background jobs during long sessions |
| `vm.swappiness=10` in `/etc/sysctl.d/99-swappiness.conf` | Less swapping of the engine's memory under load (8 GiB swap file kept) |
| THP left at `madvise` | `always` measured ±0.2%, not worth it |
| CPU governor left at amd-pstate-epp `powersave` / `balance_performance` | `performance` measured no difference |
| Wi-Fi power save off (`iw dev <if> set power_save off`, oneshot unit) | Stable SSH/API latency on Wi-Fi |
| sshd `ClientAliveInterval 15`, `ClientAliveCountMax 3` | Dead SSH sessions release their forwarded ports quickly |
| `net.mptcp.enabled = 0` | Security hardening (CVE-2026-98432 mitigation until a fixed kernel is installed) |

About 2.6 GiB of a 64 GB machine is never visible to Linux: firmware/e820 reserve ≈ 1.36 GiB and the kernel memmap ≈ 1.2 GiB.

Updates: instead of unattended-upgrades, a boot-time updater script installs packages (including kernel and driver), rebuilds Basalt from the latest `main`, and runs a self-test (load, text, image, tool call). If the self-test fails, it rolls back to the previous engine commit.

## Engine tuning notes

- Basalt's own probe tool (`basalt-probe`) found nothing left to change on this machine. Shrinking the VRAM reserve further stops the engine. Layer split cannot be combined with the peer-GPU and RAM-resident modes.
- Official community benchmark (`basalt-bench --community`, IQ3_S pack, 2× RTX 5060 Ti, 64 GB): prefill 16K 2,370 / 64K 2,459 tok/s, decode 161.3 tok/s (listed in the Basalt repo `docs/community/benchmarks.json`).

## Results

Benchmark:
1. One 600-token reply.
2. One conversation grown in 12 steps of about 8.6K tokens, up to 103.7K tokens.
3. A new conversation with eight 1280×800 screenshots.

| | IQ3_S (same engine/config as the next column) | NVFP4, older engine | **NVFP4, this config (Basalt 0.2.0)** |
|---|---|---|---|
| Short decode | 122.7 tok/s | 104.8 | 108.3 |
| Long-context decode (median / range) | 166.8 / 116–185 | 150.7 / 99–168 | **161.6 / 102–181** |
| Prefill (median / range) | 2226 / 2082–2334 | 2162 / 2007–2273 | 2173 / 2015–2317 tok/s |
| Image request (1280×800) | 1.5–1.7 s | 1.8–2.0 s | 2.1–2.4 s |
| MTP draft acceptance | 77% | 82% | 82% |
| Min. RAM available during the bench | 21.8 GiB | 3.7 GiB | 4.6 GiB |
| Refusals up to 103.7K | 0 | 0 | 0 |

Published quality from the pack README (vs Q8_0):

| Pack | KLD mean | Same top-1 |
|---|---|---|
| IQ3_S | 0.1547 | 87.25% |
| **NVFP4** | **0.0680** | **91.46%** |
| UD-Q4_K_XL | 0.0566 | 92.16% |

In real use over 318 agent requests, average decode was 98 tok/s (67–147), and no requests were refused for low RAM.

## Tried and not used

- **UD-Q4_K_XL pack** (experts 71.7 GiB). It starts with 0 GiB on the SSD tier. However, only 208 of 2986 lendable slots could keep RAM copies, so one benchmark run made about 35K blob reads (≈ 111 GB) from the file. Decode was 20–25% slower and prefill about 18% slower. The quality gain over NVFP4 is small, so we kept NVFP4.
- **No measurable gain:** THP `always` (±0.2%), GPU persistence mode, power limit (the cards never reached it), `prefill-cpu-share`, `pool-workers`, speculative/suffix draft, kv `k8v4`.
- **Need more RAM headroom than 64 GB leaves:** disk conversation tier, idle-sleep.

## Gotchas

1. **Basalt 0.2.0 reads only `.basalt` v2.** If you update the engine with a v1 pack configured, the service crash-loops. Convert first with the new commit's migration tool, then point the config at the new file and update:
   ```
   python3 tools/migrations/basalt_v1_to_v2.py old.basalt new.v2.basalt
   ```
   The tool needs numpy and free disk space equal to the pack size.
2. **The ~10 GiB RAM copy of GPU0's experts is by design, not a leak.** With `max-context` > 32768, elastic KV lends the expert cache's last slots to long prompts. Those experts are then read from their RAM copies.
3. **With `peer-exclusive`, GPU1-only experts are read from the file when GPU1 is busy.** This was about 200 MB per benchmark run with this config: small, but not zero.
4. Measure one change at a time with the same long-context benchmark. Several "obvious" levers changed nothing.
