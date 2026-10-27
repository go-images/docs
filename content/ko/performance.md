---
title: "성능"
weight: 40
---

# 성능 — `go-images/images` 대 scikit-image / SciPy {#performance-go-imagesimages-vs-scikit-image-scipy}

> **기준:** *같은 머신*에서, 현실적인 이미지 크기로, 의미가 일치하는 연산끼리 비교한 솔직한 측정값입니다. 이긴 결과**와** 진 결과를 모두 보여 줍니다. "on n'a pas le droit de se tromper", 가짜 승리는 실제 패배보다 나쁩니다.

## 측정 방법 {#method}

- **하드웨어:** 단일 Tart `debian` VM, `aarch64`(arm64) Linux, 커널 6.12, 4 vCPU. 두 스택을 *같은* VM에서 연달아 실행했습니다.
- **이미지:** 결정적(deterministic)인 `512×512`(및 `1024×1024`) RGB(A) 이미지(xorshift / NumPy PRNG).
- **go-images:** Go 1.26.4, `GOWORK=off`, `go test -bench`, 시간 = ns/op(측정 반복 중 최고 기록). 색상 연산은 RGBA에서 실행하며, 비교 대상 레퍼런스는 세 색상 채널 각각에 일치하는 연산을 실행합니다.
- **scikit-image / SciPy:** Debian 시스템 패키지(`python3-skimage` 0.25, `python3-scipy` 1.15, `python3-numpy` 2.2), `time.perf_counter`, 워밍업 후 20회 중 최고 기록. 여기서 NumPy는 SIMD를 쓰는 단일 스레드 C입니다.
- **일치하는 의미(semantics)** (비교를 공정하게 만드는 부분입니다):
  - 박스 블러 반경 5 ⇔ 채널별 `scipy.ndimage.uniform_filter(size=11, mode="nearest")`.
  - 가우시안 σ=3 ⇔ 채널별 `scipy.ndimage.gaussian_filter(sigma=3, truncate=3,
    mode="nearest")` (이 라이브러리의 반경이 `ceil(3σ)`이므로 truncate는 3입니다).
  - Sobel ⇔ 휘도 평면에 대한 `skimage.filters.sobel`(기울기 크기).
  - Erode/Dilate 반경 2 ⇔ 채널별 `scipy.ndimage.grey_erosion` /
    `grey_dilation(size=(5,5), mode="nearest")`.

분리 가능한 핵심 커널은 이제 **SIMD 내부 루프 + 멀티코어 타일링**으로 실행됩니다.

- **곱셈-누산(axpy) SIMD 커널** — 연속된 float64 평면에 대한 `dst[i] += a*src[i]` — 이 가우시안 분리형 컨볼루션을 구동합니다. 채널을 연속된 평면으로 분리(deinterleave)하므로 커널 탭마다 평평한 axpy 한 번이면 되며, 두 패스 사이의 바이트 반올림 중간값은 그대로여서 수치 출력은 이전 스칼라 코드와 동일합니다.
- **요소별 min/max SIMD 커널** — `dst[i] = min/max(dst[i], src[i])` — 이 분리형 그레이스케일 모폴로지를 구동하며, 마찬가지로 연속된 평면 위에서 동작합니다.
- **멀티코어 타일링**은 픽셀 수가 임계값을 넘으면 서로 독립인 행(가로 패스)과 열(세로 패스)을 `GOMAXPROCS`개의 고루틴에 나누어 처리하고, 임계값 아래에서는 직렬 경로가 실행됩니다. 결과는 워커 수와 무관합니다.

아키텍처별 SIMD 지원 범위(`go-fft` / `go-ndarray`가 문서화한 멀티 아키텍처 이야기)는 다음과 같습니다. **amd64 SSE2**, **arm64 NEON**(axpy는 VFMLA를 쓰며, Go 어셈블러에 니모닉이 없는 벡터 double FMIN/FMAX는 원시 명령어 워드로 내보내고 인코딩은 실제 하드웨어에서 검증했습니다), **s390x z/vector**(VFMADB + VFMINDB/VFMAXDB — 빅 엔디언이지만 이 요소별 커널은 레인 순서와 무관하며, 피연산자 순서는 qemu-s390x에서 검증했습니다). **loong64 / ppc64le / riscv64**에서는 Go 어셈블러가 쓸 만한 벡터 double 산술을 제공하지 않으므로(riscv64의 경우 기본 qemu CPU에서 V 확장이 폴트를 일으킵니다), 이들은 **스칼라 내부 루프 + 멀티코어 타일링**을 유지하며, 그래도 멀티코어에서는 이득이 있습니다. 모든 SIMD 커널은 스칼라 오라클(oracle)과 비트 단위로 일치하는지 검증하며(axpy는 GOAMD64=v3 빌드의 오라클이 VFMADD로 융합되므로 엄격한 허용 오차로 검증), 연산 전체는 SciPy와 대조해 검증합니다.

## 결과 — 512×512 (arm64 Linux, 낮을수록 좋음) {#results-512512-arm64-linux-lower-is-better}

| 연산                    | go-images **이전** | go-images **이후** | scikit-image / SciPy | SciPy 대비 (이후) |
|-------------------------|---------------------:|--------------------:|---------------------:|---------------:|
| **박스 블러** (r=5)     |            2,812,064 |             870,237 |           10,999,974 | **12.6× 빠름** |
| **Sobel** (크기)        |              882,498 |             351,701 |            2,748,223 | **7.8× 빠름** |
| **가우시안 블러** (σ=3) |           12,900,987 |           4,259,656 |           16,029,709 | **3.8× 빠름** |
| **Erode** (r=2)         |           13,865,054 |           1,581,921 |           10,559,638 | **6.7× 빠름** |
| **Dilate** (r=2)        |           12,898,691 |           1,482,165 |           11,025,974 | **7.4× 빠름** |

### 1단계 에지 / 필터 추가분 — 512×512 (arm64 Linux, 낮을수록 좋음) {#phase-1-edge-filter-additions-512512-arm64-linux-lower-is-better}

새로 추가된 멀티코어(스칼라 내부 루프) 연산이며, 같은 VM에서 일치하는 scikit-image / SciPy 레퍼런스와 비교해 측정했습니다. 아직 SIMD 내부 루프는 없고, 이득은 멀티코어 분산과 더 저렴한 알고리즘(중앙값의 카운팅 선택)에서 나오는데도 이미 scikit-image를 앞섭니다.

| 연산                    | go-images (ns/op) | scikit-image / SciPy (ns/op) | 속도 향상 |
|-------------------------|------------------:|-----------------------------:|---------:|
| **Scharr** (크기)       |         1,421,773 |                    3,833,147 | **2.7× 빠름** |
| **Canny** (σ=1)         |        10,086,768 |                   27,031,572 | **2.7× 빠름** |
| **Median** (r=2, 5×5)   |        37,565,475 |                  214,597,360 | **5.7× 빠름** |

일치하는 의미: Scharr ⇔ 휘도 평면에 대한 `skimage.filters.scharr`, Canny ⇔ `skimage.feature.canny(sigma=1, low=20, high=40)`, Median ⇔ 채널별 `scipy.ndimage.median_filter(size=5, mode="nearest")`. Prewitt와 SobelMag는 Scharr의 커널을 공유하며, UnsharpMask/Sharpen은 GaussianBlur에 픽셀별 융합 덧셈을 더한 것입니다(위의 가우시안 수치가 적용됩니다). 중앙값이 크게 앞서는 것은 SciPy의 범용 n차원 중앙값 대신 O(window+256) 카운팅 선택을 쓰기 때문이며, 멀티코어 타일링이 그 차이를 더 키웁니다. 에지 계열과 Canny가 공유하는 SIMD Sobel 기울기 커널을 추가하는 것이 다음 벡터화 대상입니다.

"이전" 열은 스칼라 단일 스레드 커널이고, "이후"는 SIMD + 4코어입니다. **이전의 두 패배가 이제 모두 승리입니다.** 가우시안은 0.92×(1.08× 느림)에서 **3.8× 빠름**으로, Erode/Dilate는 약 0.9×에서 **6.7×/7.4× 빠름**으로 바뀌었습니다. 이미 앞서던 박스 블러와 Sobel은 격차가 더 벌어졌습니다(멀티코어 타일링만으로 4코어에서 대략 세 배가 됩니다).

## 결과 — 1024×1024 (arm64 Linux, 낮을수록 좋음) {#results-10241024-arm64-linux-lower-is-better}

| 연산                    | go-images (ns/op) | scikit-image / SciPy (ns/op) | 속도 향상 |
|-------------------------|------------------:|-----------------------------:|---------:|
| **박스 블러** (r=5)     |         7,548,773 |                   29,963,031 | **4.0× 빠름** |
| **Sobel** (크기)        |         6,003,498 |                   11,327,476 | **1.9× 빠름** |
| **가우시안 블러** (σ=3) |        11,451,846 |                   46,418,949 | **4.1× 빠름** |
| **Erode** (r=2)         |         7,896,837 |                   59,335,849 | **7.5× 빠름** |
| **Dilate** (r=2)        |         7,893,893 |                   61,166,651 | **7.7× 빠름** |

1024²에서는 scipy가 큰 중간 배열을 만들어 내는 연산(모폴로지, 가우시안)에서 격차가 커집니다. Sobel은 픽셀당 산술이 적고 이득이 대부분 멀티코어 분산에서 나오므로(이 크기에서는 채널별 Python 오버헤드가 없는 단일 SciPy 에지 필터) 1.9×로 좁아집니다.

### 구성 요소별 속도 향상 (512², 이전 → 이후) {#per-component-speed-up-512-before-after}

- **가우시안 — 스칼라 바닥 대비 3.0× 빠름**(이제 SciPy보다도 앞섭니다): axpy SIMD 커널이 조밀한 곱셈-덧셈을 벡터화하고 두 패스가 여러 코어에 분산됩니다. 세로 패스는 연속된 행 전체에 대한 순수 axpy이고, 가로 패스는 안쪽 열을 벡터화하며 경계에서 클램프되는 몇 개의 열은 스칼라로 처리합니다.
- **Erode/Dilate — 스칼라 바닥 대비 약 8.7× 빠름:** 윈도우별 실행 min/max가 평면 위의 연속 SIMD MIN/MAX와 멀티코어 타일링으로 바뀌었고, 행별 스크래치(scratch)는 내부 루프 밖으로 끌어냈습니다(할당이 op당 약 6,200회에서 71회로 줄었습니다).

> **업데이트 — O(1) van Herk / Gil-Werman 모폴로지.** 모폴로지 내부 루프를 O(radius) 폴드에서 **O(1) van Herk / Gil-Werman** 실행 min/max, 즉 픽셀당 비교 세 번(반경과 무관하게 일정)으로 다시 작성했습니다. 단일 스레드 scikit-image 동등성 벤치마크(Apple M4 Max)에서 erode/dilate는 4096²에서 scikit-image의 **동등 → 1.02×**(이전 0.58–0.77×)에 이르고 전체 코어에서는 **4.8–7.8×**입니다. 출력은 `scipy.ndimage.grey_erosion`/`grey_dilation`과 바이트 단위로 동일합니다. 작은 반경에서 남은 차이는 순수한 상수 배(SIMD)입니다. 단일 스레드와 멀티 스레드 동등성 표 전체와 측정 방법은 [BENCHMARKS.md](https://github.com/go-images/images/blob/main/BENCHMARKS.md)를 참고하십시오.

> **업데이트 — 분기 없는 Sobel.** 위에서 짚었던 유일한 패배인 픽셀마다 다시 계산하는 Sobel 기울기가 이후 해소되었습니다. 이제 각 행은 클램프 주소 지정을 쓰는 경계 열과 행 기준 주소를 루프 밖으로 뺀 **분기 없는 안쪽 구간**으로 나뉘어, 안쪽 구간에서는 `clampIndex` 호출이 전혀 없습니다. 출력은 단순한 형태와 바이트 단위로 동일합니다. 단일 스레드 scikit-image 동등성 벤치마크(Apple M4 Max)에서 Sobel은 이제 scikit-image의 **1.02–1.06×**(이전 0.78–0.80×)에 이르고 전체 코어에서는 **4.2–4.8×**입니다. [BENCHMARKS.md](https://github.com/go-images/images/blob/main/BENCHMARKS.md)의 작업 항목 B를 참고하십시오.

## SciPy 대비 정확성 (속도만이 아닙니다) {#correctness-vs-scipy-not-just-speed}

같은 VM에서 `64×48` 결정적 이미지를 사용해, SIMD/멀티코어 재작성 **이후** go-images 출력을 일치하는 SciPy 레퍼런스와 픽셀 단위로 비교했습니다.

| 연산 | 최대 \|Δ\| | 평균 \|Δ\| | 판정 |
|-----------|----------:|-----------:|---------|
| 박스 블러 |         0 |      0.000 | **정확히 일치** |
| Erode     |         0 |      0.000 | **정확히 일치** |
| Dilate    |         0 |      0.000 | **정확히 일치** |
| 가우시안  |         1 |      0.079 | 1 LSB 이내로 일치 (반올림 차이뿐) |

SIMD/멀티코어 재작성은 수치적으로 **아무것도** 바꾸지 않았습니다. 박스 블러와 그레이스케일 모폴로지는 여전히 SciPy와 비트 단위로 일치하고, 가우시안은 여전히 최종 클램프의 마지막 비트 반올림에서만 차이가 나며, 이는 스칼라 구현과 같은 수치입니다. 또한 각 SIMD 커널은 모든 길이 나머지(residue)에 대해 스칼라 오라클(`internal/kernels/simd_test.go`)과 대조하고, 분리형 연산 전체는 직렬 임계값과 병렬 임계값 양쪽에서 독립적인 단순 레퍼런스(`separable_test.go`)와 대조합니다.

### 남은 사항 (솔직하게) {#residual-notes-honest}

- **메모리:** 평면 기반 가우시안은 이전의 제자리 바이트 경로보다 일시적인 float64 스크래치를 더 많이 할당합니다(512²에서 op당 약 14.8 MB 대 2.1 MB). 연속 SIMD 레이아웃의 대가입니다. 호출마다 해제되는 일시적 할당이며 3.8× 속도 향상의 비용으로 치르는 것입니다. 재사용 가능한 스크래치 풀로 줄이는 것은 후속 과제입니다.
- **loong64 / ppc64le / riscv64**는 스칼라 내부 루프를 실행하며(Go 어셈블러에 쓸 만한 벡터 double이 없고 qemu V는 폴트를 일으킵니다), 속도 향상은 멀티코어 타일링에만 의존합니다. 이들은 아키텍처별 qemu CI 작업에서 검증됩니다.

## 재현 방법 {#reproducing}

```sh
# On the arm64 debian VM, with the repo synced to /tmp/goimages:
cd /tmp/goimages
GOWORK=off /usr/local/go/bin/go test -run=^$ \
  -bench 'BoxBlur|GaussianBlur|Sobel$|Erode|Dilate|Scharr|Canny|Median' -benchtime=2s ./...

# scikit-image / SciPy side (see the harness used to produce the table):
python3 skbench.py      # timings
python3 validate.py     # correctness vs scipy
```
