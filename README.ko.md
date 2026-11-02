# Qwen3.8-Flash-Next NVFP4를 RTX 5060 Ti 16GB × 2 + RAM 64GB에서 돌리기 (Basalt)

[English](README.md) | **한국어**

중급 듀얼 GPU 데스크톱에서 Qwen3.8-Flash-Next의 **NVFP4** Basalt 팩(파일 122 GiB, 전문가 63.3 GiB)을 돌리는, 실측으로 확인한 설정입니다. **모든 전문가를 VRAM과 고정(page-locked) RAM에서 처리하므로 SSD를 저장 계층으로 쓰지 않습니다.** 문맥 103.7K 토큰까지 거부된 요청이 없었습니다.

참고용으로 공유합니다. 하드웨어, 엔진 커밋, 작업 형태가 다르면 수치도 달라집니다.

## 하드웨어와 소프트웨어

| | |
|---|---|
| CPU | AMD Ryzen 5 9600X |
| RAM | DDR5 64 GB (리눅스에서 보이는 양 61.9 GiB) |
| GPU | NVIDIA GeForce RTX 5060 Ti 16 GB × 2 (PCIe, NVLink 없음), 드라이버 595.91 |
| 디스크 | NVMe SSD (약 7 GB/s) |
| OS | Ubuntu 26.04 LTS (커널 7.0), 모니터 없이 SSH로만 운용 |
| 엔진 | Basalt 0.2.0 (`v0.1.0-119-ga04edb96`, 2026-11-01) |
| 모델 | `Qwen3.8-FN-NVFP4` 팩, **`.basalt` 형식 v2** |
| 작업 | 사용자 1명, 긴 에이전트 세션(30K~100K 토큰), 도구 호출, 코드, 1280×800 스크린샷, thinking 켬 |

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

| 설정 | 이유 |
|---|---|
| `resident-experts: true` | GPU에 없는 전문가를 RAM에 고정해 두므로 SSD를 읽지 않습니다. |
| `peer-exclusive: true` | GPU1이 가진 전문가는 RAM 사본을 두지 않습니다. RAM이 약 15 GiB 절약되고, 이 덕분에 NVFP4가 64 GB에 들어갑니다. |
| `max-context: 131072` | 262144일 때보다 elastic KV가 전문가 슬롯을 덜 빌려 갑니다. |
| `kv: int8` | 기본 품질/크기 균형이라 그대로 두었습니다. |
| `vision: dynamic` | 비전 모듈이 VRAM을 상시 차지하지 않습니다. |
| `prompt-cache: 2` | 체크포인트 수를 기본 6개에서 줄였습니다. 실제 에이전트 사용에서도 프롬프트 재사용률은 약 88%였습니다. |
| `vram-reserve-mib: 256`, `peer-reserve-mib: 128` | 최소값입니다. GPU 두 장이 98~99% 찹니다. |
| `peer-prefill-rows: 0` | **`peer-reserve-mib: 128`을 쓰려면 꼭 필요합니다.** 이게 없으면, 첫 덩어리가 5120 토큰 이하인 첫 요청에서 GPU1이 MMQ prefill을 돌립니다. 그때 약 83 MiB가 더 필요해 CUDA OOM이 납니다. |
| `adapt-swaps: 48` | 적응형 교체용으로 비워 두는 RAM이 절반이 됩니다(1266 → 633 MiB). 속도 차이는 측정 오차 수준입니다. |

이 설정으로 엔진을 시작했을 때:
- GPU1(peer) 전문가: 5907개
- RAM의 전문가: 48.68 GiB
- 로드 후 RAM 여유: 약 5.0 GiB (안전선 3.1 GiB)
- 빌려줄 수 있는 슬롯 모두 RAM 사본 보유
- 남은 VRAM: GPU0 193 MiB, GPU1 19 MiB

## OS 변경

| 변경 | 효과 |
|---|---|
| BIOS: Primary Video = PCIE, Integrated Graphics = Disabled | RAM +0.5 GiB (내장 그래픽용 UMA 회수) |
| 커널 인자 `crashkernel=` 제거, kdump-tools 끔 | RAM +1.25 GiB |
| snapd, multipathd, networkd-dispatcher, unattended-upgrades 끔, 부팅 대상 `multi-user.target` | RAM 약 0.1~0.2 GiB |
| `/etc/sysctl.d/99-swappiness.conf`에 `vm.swappiness=10` | 부하 중 엔진 메모리가 스왑으로 밀려나는 일 감소 |

64 GB 장비에서 약 2.6 GiB는 리눅스에 아예 보이지 않습니다. 펌웨어/e820 예약이 약 1.36 GiB, 커널 메모리 맵이 약 1.2 GiB입니다.

> unattended-upgrades를 끄면 보안 업데이트를 직접 챙겨야 합니다.

## 결과

벤치마크:
1. 600토큰 응답 1회
2. 대화 하나를 약 8.6K 토큰씩 12단계로 키워 103.7K 토큰까지
3. 새 대화에 1280×800 스크린샷 8장

| | IQ3_S (오른쪽 칸과 같은 엔진/설정) | NVFP4, 이전 엔진 | **NVFP4, 이 설정 (Basalt 0.2.0)** |
|---|---|---|---|
| 짧은 응답 출력 속도 | 122.7 tok/s | 104.8 | 108.3 |
| 긴 문맥 출력 속도 (중앙값 / 범위) | 166.8 / 116~185 | 150.7 / 99~168 | **161.6 / 102~181** |
| prefill (중앙값 / 범위) | 2226 / 2082~2334 | 2162 / 2007~2273 | 2173 / 2015~2317 tok/s |
| 이미지 요청 (1280×800) | 1.5~1.7초 | 1.8~2.0초 | 2.1~2.4초 |
| MTP 추측 토큰 수락률 | 77% | 82% | 82% |
| 벤치 중 최저 RAM 여유 | 21.8 GiB | 3.7 GiB | 4.6 GiB |
| 103.7K까지 거부 | 0 | 0 | 0 |

팩 README에 공개된 품질 수치 (Q8_0 기준):

| 팩 | KLD 평균 | top-1 일치 |
|---|---|---|
| IQ3_S | 0.1547 | 87.25% |
| **NVFP4** | **0.0680** | **91.46%** |
| UD-Q4_K_XL | 0.0566 | 92.16% |

실제 에이전트 요청 318건의 평균 출력 속도는 98 tok/s(67~147)였고, RAM 부족으로 거부된 요청은 없었습니다.

## 시도했지만 안 쓴 것

- **UD-Q4_K_XL 팩** (전문가 71.7 GiB). 시작할 때 SSD 계층은 0 GiB입니다. 하지만 빌려줄 수 있는 슬롯 2986개 중 208개만 RAM 사본을 가질 수 있었습니다. 그래서 벤치 한 번에 파일에서 블롭을 약 35K번(≈ 111 GB) 읽었고, 출력은 20~25%, prefill은 약 18% 느렸습니다. NVFP4 대비 품질 이득이 작아 NVFP4를 유지했습니다.
- **효과가 측정되지 않은 것:** THP `always`(±0.2%), GPU persistence mode, 전력 제한(카드가 제한에 닿지 않음), `prefill-cpu-share`, `pool-workers`, speculative/suffix draft, kv `k8v4`.
- **64 GB로는 RAM 여유가 모자라 못 쓰는 것:** 디스크 대화 캐시, idle-sleep.

## 주의할 점

1. **Basalt 0.2.0은 `.basalt` v2만 읽습니다.** v1 팩을 설정해 둔 채 엔진을 업데이트하면 서비스가 계속 재시작합니다. 새 커밋의 변환 도구로 먼저 바꾸고, 설정을 새 파일로 바꾼 뒤 업데이트하세요.
   ```
   python3 tools/migrations/basalt_v1_to_v2.py old.basalt new.v2.basalt
   ```
   numpy가 필요하고, 팩 크기만큼 디스크 여유가 있어야 합니다.
2. **GPU0 전문가의 RAM 사본 약 10 GiB는 누수가 아니라 설계입니다.** `max-context`가 32768보다 크면, elastic KV가 긴 프롬프트에 전문가 캐시의 마지막 슬롯을 빌려줍니다. 그때 그 전문가는 RAM 사본에서 읽습니다.
3. **`peer-exclusive`를 쓰면 GPU1 전용 전문가는 GPU1이 바쁠 때 파일에서 읽습니다.** 이 설정에서는 벤치 한 번에 약 200 MB로 작지만 0은 아닙니다.
4. 같은 긴 문맥 벤치로 한 번에 하나씩 바꿔 가며 측정하세요. "당연히 효과 있을 것" 같은 설정 여러 개가 아무 변화가 없었습니다.
