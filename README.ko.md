# Qwen3.8-Flash-Next NVFP4를 RTX 5060 Ti 16GB × 2 + RAM 64GB에서 돌리기 (Basalt)

[English](README.md) | **한국어**

중급 듀얼 GPU 데스크톱에서 Qwen3.8-Flash-Next의 **NVFP4** Basalt 팩(파일 122 GiB, 전문가 63.3 GiB)을 돌리는, 실측으로 확인한 설정입니다. **모든 전문가를 VRAM과 고정(page-locked) RAM에서 처리하므로 SSD를 저장 계층으로 쓰지 않습니다.** 문맥 103.7K 토큰까지 거부된 요청이 없었습니다.

참고용으로 공유합니다. 하드웨어, 엔진 커밋, 작업 형태가 다르면 수치도 달라집니다.

## 하드웨어와 소프트웨어

| | |
|---|---|
| CPU | AMD Ryzen 5 9600X |
| 메인보드 | ASUS ProArt B850-CREATOR WIFI NEO (BIOS 1103) |
| RAM | DDR5 64 GB (KLEVV DDR5-5600 CL46 32 GB × 2를 6400 CL30으로 운용, 리눅스에서 보이는 양 61.9 GiB) |
| GPU | PNY GeForce RTX 5060 Ti OC 16 GB × 2 (각각 PCIe Gen5 x8 CPU 직결, NVLink 없음), 드라이버 595.91.07, CUDA 13.4 |
| 디스크 | Samsung PM9A1 NVMe 1 TB |
| 파워 | 1050 W 80+ 골드 |
| OS | Ubuntu Server 26.04.1 LTS (커널 7.0), 텍스트 모드 전용(`multi-user.target`), 모니터 없이 SSH로만 운용 |
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

## BIOS

적지 않은 항목은 Auto 또는 기본값입니다.

| 영역 | 설정 |
|---|---|
| CPU | PBO Enabled, Curve Optimizer 전체 코어 **Negative 40** |
| 메모리 | DDR5-**6400**, UCLK DIV1 MODE = UCLK=MEMCLK, tCL **30**, tRCD WR 36, tRCD RD 38, tRP 38, tRAS 88, DRAM VDD / VDDQ **1.35 V**, High DRAM Voltage Mode Disabled, Power Down Enable Disabled, Memory Context Restore Disabled |
| 패브릭 / SoC | Ai Overclock Tuner Manual, BCLK 100.00 MHz, FCLK **2133 MHz**, AI Cache Boost Enabled, CPU SOC Voltage Manual **1.25 V** |
| PCIe | Resizable BAR Enabled |
| 그래픽 | Primary Video Device = PCIE, **Integrated Graphics = Disabled** (내장 그래픽용 UMA를 회수해 RAM +0.5 GiB, 모니터 없이 운용) |

> 메모리와 CPU 값은 이 부품에 맞춰 조정한 값입니다. 다른 하드웨어에서는 안정성을 직접 검증한 뒤 쓰세요.

## GPU 조정 (부팅할 때마다 적용)

이 설정은 카드에 저장되지 않습니다. 작은 oneshot systemd 유닛(`gpu-tune.service`, 모델 서비스보다 먼저 실행)이 NVML로 적용합니다. 드라이버에 들어 있는 `libnvidia-ml.so.1`을 Python ctypes로 부르므로 추가 패키지가 필요 없습니다. 알고 있는 카드 UUID 두 개에만 적용하므로, 카드를 바꿔 끼워도 다른 카드에서 검증한 값이 들어가지 않습니다.

| | GPU0 | GPU1 |
|---|---|---|
| 코어 클럭 오프셋 (`SetGpcClkVfOffset`) | **+150 MHz** | **+175 MHz** |
| 메모리 클럭 오프셋 (`SetMemClkVfOffset`) | **+750 MHz** (14001 → 14376 MHz) | **+750 MHz** |
| 코어 클럭 고정 범위 (`SetGpuLockedClocks`) | 210–2827 MHz | 210–2962 MHz |
| 적용 후 최대 클럭 (NVML) | 코어 3360 / 메모리 14376 MHz | 코어 3382 / 메모리 14376 MHz |
| 전력 제한 | 180 W (기본값) | 180 W (기본값) |
| 드라이버 상주 | `nvidia-persistenced`가 드라이버를 올려 둠 (persistence mode 플래그는 Disabled) | 같음 |

- 카드마다 따로 검증해서 값이 서로 다릅니다. 코어 클럭 상한은 오버클럭이 아니라 전력 효율을 위한 제한입니다.
- 효과(같은 벤치 1회씩): 출력 +2.1%(118.6 → 121.1 tok/s), 한국어 글 +2.3%, 10K 입력 처리 −1.6%(측정 오차 범위)였습니다. 정답 40/40과 도구 호출 5/5는 그대로였고 GPU 오류(Xid)는 없었습니다. 출력 향상이 메모리 클럭 상승분(+2.7%)과 거의 같은 이유는, 출력 속도가 메모리 대역폭에 좌우되기 때문입니다.
- 전력 제한을 198 W로 올려도 입력 처리가 +0.4%에 그쳐 기각했습니다. 이 작업에서 카드는 최대 약 126 W만 씁니다.

## OS 변경

| 변경 | 효과 |
|---|---|
| 텍스트 모드: 기본 부팅 대상 `multi-user.target` (데스크톱 없음) | RAM 절약, GPU는 모델만 사용 |
| 커널 인자 `crashkernel=` 제거, kdump-tools 끔 | RAM +1.25 GiB |
| snapd(소켓, seeded 포함), multipathd(소켓 mask), networkd-dispatcher, unattended-upgrades, apt-daily 타이머 끔 | RAM 약 0.1~0.2 GiB, 긴 세션 중 백그라운드 작업 없음 |
| `/etc/sysctl.d/99-swappiness.conf`에 `vm.swappiness=10` | 부하 중 엔진 메모리가 스왑으로 밀려나는 일 감소 (스왑 파일 8 GiB는 유지) |
| THP는 `madvise` 그대로 | `always`는 ±0.2%라 효과 없음 |
| CPU 전원 정책은 amd-pstate-epp `powersave` / `balance_performance` 그대로 | `performance`와 차이 없음 |
| Wi-Fi 절전 끔 (`iw dev <장치> set power_save off`, oneshot 유닛) | Wi-Fi에서 SSH·API 지연이 안정적 |
| sshd `ClientAliveInterval 15`, `ClientAliveCountMax 3` | 끊긴 SSH 세션이 전달 포트를 빨리 놓아줌 |
| `net.mptcp.enabled = 0` | 보안 강화 (수정 커널을 설치하기 전까지 CVE-2026-98432 완화) |

64 GB 장비에서 약 2.6 GiB는 리눅스에 아예 보이지 않습니다. 펌웨어/e820 예약이 약 1.36 GiB, 커널 메모리 맵이 약 1.2 GiB입니다.

업데이트: unattended-upgrades 대신 부팅할 때 도는 업데이트 스크립트가 패키지(커널, 드라이버 포함)를 설치하고, Basalt를 최신 `main`으로 다시 빌드한 뒤 자체 시험(로드, 글, 이미지, 도구 호출)을 돌립니다. 자체 시험이 실패하면 이전 엔진 커밋으로 되돌립니다.

## 엔진 조정 참고

- Basalt 자체 측정 도구(`basalt-probe`)가 이 기계에서 더 바꿀 설정이 없다고 판정했습니다. VRAM 여유분을 더 줄이면 엔진이 멈춥니다. 레이어 분할은 보조 GPU 방식, RAM 상주 방식과 함께 쓸 수 없습니다.
- 공식 커뮤니티 벤치마크(`basalt-bench --community`, IQ3_S 팩, RTX 5060 Ti × 2, 64 GB): 입력 처리 16K 2,370 / 64K 2,459 tok/s, 출력 161.3 tok/s (Basalt 저장소 `docs/community/benchmarks.json`에 등재).

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
