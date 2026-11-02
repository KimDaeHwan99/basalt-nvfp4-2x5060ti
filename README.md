# Qwen3.8-Flash-Next NVFP4 on 2× RTX 5060 Ti 16 GB + 64 GB RAM (Basalt)

A measured, working configuration for running the **NVFP4** Basalt pack of Qwen3.8-Flash-Next (122 GiB file, 63.3 GiB of experts) on a mid-range dual-GPU desktop. **All experts are served from VRAM and pinned RAM, so the SSD is not used as a tier.** No requests were refused up to 103.7K tokens of context.

Shared as a reference. Your numbers will differ with other hardware, engine commits and workloads.

## Hardware and software

| | |
|---|---|
| CPU | AMD Ryzen 5 9600X |
| RAM | 64 GB DDR5 (61.9 GiB visible to Linux) |
| GPU | 2× NVIDIA GeForce RTX 5060 Ti 16 GB (PCIe, no NVLink), driver 595.91 |
| Disk | NVMe SSD (~7 GB/s) |
| OS | Ubuntu 26.04 LTS (kernel 7.0), headless (SSH only) |
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

## OS changes

| Change | Effect |
|---|---|
| BIOS: Primary Video = PCIE, Integrated Graphics = Disabled | +0.5 GiB RAM (the iGPU's UMA carve-out) |
| Removed the `crashkernel=` kernel argument; disabled kdump-tools | +1.25 GiB RAM |
| Disabled snapd, multipathd, networkd-dispatcher, unattended-upgrades; boot to `multi-user.target` | ~0.1–0.2 GiB RAM |
| `vm.swappiness=10` in `/etc/sysctl.d/99-swappiness.conf` | Less swapping of the engine's memory under load |

About 2.6 GiB of a 64 GB machine is never visible to Linux: firmware/e820 reserve ≈ 1.36 GiB and the kernel memmap ≈ 1.2 GiB.

> Disabling unattended-upgrades means you must apply security updates yourself.

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
