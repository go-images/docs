---
title: "로드맵"
weight: 30
---

# 구현 계획 — 순수 Go 이미지 처리 (`go-images/images`) {#implementation-plan-pure-go-image-processing-go-imagesimages}

> 목표: Go 표준 라이브러리 위에 구축한, scikit-image 스타일의 **순수 Go(cgo 없음)** 이미지 처리 라이브러리. 네이티브 의존성이 없어 **임베드할 수 있고**, 모든 Go 대상으로 **크로스 컴파일**할 수 있으며, [go-embedded-ruby](https://github.com/go-embedded-ruby/ruby)를 위한 이미지 백엔드로 만들어집니다.

## 1. 왜 순수 Go인가? {#1-why-pure-go}

주류 이미지 스택은 모두 네이티브 C 라이브러리를 필요로 합니다.

- **ruby-vips** → libvips, **RMagick** → ImageMagick.
- cgo 기반 Go 래퍼(`govips`, `go-imagick`)도 같은 공유 객체를 링크합니다.

그 때문에 정적 빌드, 크로스 컴파일, 임베딩이 어렵습니다. 이 라이브러리는 **CGO=0**이므로 `go build` 한 번으로 시스템 패키지 없이 지원하는 어떤 대상에서든 자급자족하는 바이너리가 만들어집니다.

### 솔직한 비교 {#honest-comparison}

순수 Go 이미지 라이브러리는 이미 존재하며, 우리는 그렇지 않은 척하지 않습니다.

- **[bild](https://github.com/anthonynsimon/bild)** — 블러, 에지 검출, 모폴로지, 히스토그램을 갖춘 폭넓은 순수 Go 툴킷. 성숙하고 기능이 풍부합니다.
- **[disintegration/imaging](https://github.com/disintegration/imaging)** — 크기 조정/변환/보정에 집중한, 사랑받는 라이브러리.
- **libvips / ImageMagick** — 기능이 훨씬 많고 **손으로 튜닝한 SIMD**가 있습니다. 오늘날 큰 이미지에서는 훨씬 빠르지만 cgo에 묶여 있습니다.

이 프로젝트가 차별화하려는 지점은 다음과 같습니다.

1. 순수 함수와 작고 안정적인 표면을 갖춘 **깔끔하고 정확하며 100% 커버되는 코어**.
2. 좁은 시그니처 뒤에 **격리된 커널**(`internal/kernels`)로, 공개 API를 건드리지 않고 **[go-asmgen](https://github.com/go-asmgen)으로 생성한 SIMD**로 통째로 교체할 수 있습니다. go-asmgen은 64비트 대상 여섯 가지(amd64, arm64, riscv64, loong64, ppc64le VSX, s390x vector)를 모두 다루는 유일한 Go SIMD 생성기입니다. cgo 없이 libvips/ImageMagick과의 속도 격차를 좁히는 길입니다.
3. go-embedded-ruby의 이미지 백엔드로서의 **임베드 가능성**.

우리는 오늘 libvips/ImageMagick보다 빠르다고도, bild보다 기능이 많다고도 주장하지 **않습니다**. 걸고 있는 것은 cgo 없음 + 임베드 가능 + 여섯 아키텍처에 걸친 SIMD 이야기입니다.

## 2. 아키텍처 {#2-architecture}

```
image.Image
   │  ToRGBA  → *image.RGBA (origin-anchored, tightly packed R,G,B,A)
   ▼
public ops (ops.go, io.go)        ← pure funcs, return a new *image.RGBA
   │  delegate hot loops to
   ▼
internal/kernels                  ← scalar pure-Go pixel loops (today)
   │  same signatures
   ▼
internal/kernels (SIMD, later)    ← go-asmgen kernels per target (Phase 2)
```

RGBA 레이아웃(행 우선, 픽셀당 4바이트)은 의도적인 선택입니다. 벡터 레지스터에 그대로 대응되는 형태이기 때문입니다.

## 3. 로드맵 {#3-roadmap}

### 0단계 — 코어 (완료) {#phase-0-core-done}

- `ToRGBA` 변환 헬퍼.
- 입출력: `Load`, `Save`(확장자에 따라 PNG/JPEG), `Decode`, `Encode`.
- 점 연산: `Grayscale`(Rec. 601 휘도), `Invert`, `AdjustBrightness`, `AdjustContrast`(둘 다 `[0,255]`로 클램프).
- 최근접 이웃과 쌍선형 모드를 지원하는 `Resize`.
- 임의의 홀수 크기 float 커널과 가장자리 클램프 경계를 지원하는 `Convolve`.
- 분리형 컨볼루션으로 구현한 `GaussianBlur`.
- 구문 커버리지 100%. CI는 운영체제 세 가지와 64비트 아키텍처 여섯 가지(네이티브 amd64/arm64, qemu riscv64/loong64/ppc64le/s390x)에서 실행합니다.

### 1단계 — 더 많은 필터와 변환 (진행 중) {#phase-1-more-filters-transforms-in-progress}

- **Sobel 에지 검출 (완료)** — `Sobel`(기울기 크기), `SobelX`/`SobelY`(방향별 응답). 모두 Rec. 601 휘도 위에서 가장자리 클램프 경계를 사용합니다. 휘도 평면은 한 번만 계산하고, 기울기/크기 패스는 펼친(unrolled) 커널 탭과 융합했습니다. 벤치마크는 융합하지 않은 2패스 기준선과 비교하며, 차분(differential) 테스트가 독립적인 단순 레퍼런스에 맞춰 고정합니다.
- **박스 블러 (완료)** — `BoxBlur`, 분리형 실행 합(반경에 대해 픽셀당 O(1)), 두 패스가 정확한 2차원 평균이 되도록 float 중간값을 사용합니다. `scipy.ndimage.uniform_filter`(mode=nearest)와 비트 단위로 일치합니다. 독립적인 단순 오라클과 VM의 SciPy에 대해 검증했습니다.
- **기하 변환 (완료)** — `FlipHorizontal`/`FlipVertical`(numpy.fliplr/flipud), `Rotate90`/`Rotate180`/`Rotate270`(numpy.rot90), `Crop`. 보간 없이 픽셀을 정확하게 재배열합니다.
- **색상 및 임계값 (완료)** — `RGBToHSV`/`HSVToRGB`(바이트 인코딩, 왕복 안정), `OtsuThreshold`(`skimage.filters.threshold_otsu`와 일치), `Threshold`/`Otsu`.
- **에지 연산자 (완료)** — `Prewitt`, `Scharr`, `SobelMag`(휘도 [0,1]에 대한 `sqrt((gx²+gy²)/2)`, 정규화된 분리형 기울기 크기)와 `Laplacian`(이산 `[0,-1,0;-1,4,-1;0,-1,0]`). VM에서 `skimage.filters.{prewitt,scharr,sobel,laplace}`와 비트 단위로 일치합니다(3탭 커널에서는 가장자리 클램프가 skimage의 reflect 경계와 같습니다).
- **Canny (완료)** — `Canny(sigma, low, high)`: 가우시안으로 평활한 휘도 → Sobel 기울기 → 쌍선형 비최대 억제 → 8-연결 히스테리시스로, `skimage.feature.canny`를 그대로 따릅니다. skimage와 내부 영역 일치율이 약 98%(mode=nearest)이며, 나머지 차이는 skimage의 마스킹된 상수 경계 전처리와 이 라이브러리의 더 깔끔한 가장자리 클램프의 차이입니다.
- **Median + unsharp (완료)** — `Median`(256 빈 카운팅 선택으로 구한 정사각형 중앙값, `scipy.ndimage.median_filter` `mode="nearest"`와 비트 단위로 일치)과 `UnsharpMask`/`Sharpen`(`src + amount*(src − blur)`, `skimage.filters.unsharp_mask`의 공식과 반올림 오차 범위 내에서 정확히 일치).
- 양방향(bilateral) 필터, 엠보스.
- 임의 각도 회전, 아핀/워프, 바이큐빅 크기 조정.
- 색상 보정: 감마, 채널 혼합, RGB↔Lab, 히스토그램 평활화.
- 설정 가능한 가장자리 처리(clamp / wrap / reflect / constant).

### 2단계 — 모폴로지와 분석 {#phase-2-morphology-analysis}

- **그레이스케일 모폴로지 (완료)** — `Erode`/`Dilate`(정사각형 요소에 대한 분리형 채널별 지역 min/max, `scipy.ndimage.grey_erosion`/`grey_dilation`과 비트 단위로 일치)와 파생된 `Open`/`Close`. 0/255 이진값. **O(1) van Herk / Gil-Werman** 실행 min/max(픽셀당 비교 세 번, 반경과 무관하게 일정)로 동작하며, 4096²에서 단일 스레드 scikit-image 동등 → 1.02×, 전체 코어에서 4.8–7.8×에 이릅니다.
- 연결 요소, 레이블링, 영역 속성.
- 히스토그램(`kernels.Histogram`이 있음)과 히스토그램 평활화.

### 성능 {#performance}

[성능]({{< relref "performance.md" >}}) 페이지는 go-images와 scikit-image / SciPy / OpenCV의 솔직한 벤치마크를 추적합니다. go-images는 분리 가능한 핵심 연산에서 scikit-image를 앞섭니다(박스 블러 1.8–2.6×, RGB→HSV 약 4.8×, 뒤집기 약 2×, 가우시안은 이제 1.0–1.3×). **모폴로지는 이제 scikit-image와 동등한 수준에 이르렀으며**(erode/dilate는 4096²에서 단일 스레드 동등 → 1.02×, 전체 코어에서 4.8–7.8×) O(1) van Herk / Gil-Werman 재작성 덕분입니다. **Sobel도 이제 동등한 수준에 이르렀습니다**(단일 스레드 1.02–1.06×, 전체 코어 4.2–4.8×). 분기 없는 안쪽 구간 재작성 덕분입니다. 남은 단일 스레드 격차는 순수한 상수 배(작은 반경의 모폴로지, 그리고 픽셀 단위가 아닌 Rotate90/Crop/Otsu 레퍼런스 연산)이며, 이를 좁히고 OpenCV의 O(1)+SIMD 모폴로지에 맞추는 일이 BENCHMARKS.md에서 추적하는 SIMD 작업입니다.

### 3단계 — go-asmgen을 통한 SIMD 커널 {#phase-3-simd-kernels-via-go-asmgen}

- `internal/kernels`의 스칼라 루프를 64비트 Go 대상 여섯 가지 모두에서 SIMD로 교체하되, 스칼라 버전은 폴백과 정확성 오라클로 유지합니다(차분 테스트: SIMD 출력은 스칼라 출력과 같아야 합니다).
- 속도 향상을 측정하고 libvips/ImageMagick과 비교해 추적하는 벤치마크 스위트.

### 4단계 — Ruby 바인딩 {#phase-4-ruby-binding}

- ruby-vips/RMagick을 대체하는 cgo 없는 대안으로서, [go-embedded-ruby](https://github.com/go-embedded-ruby/ruby)를 통해 라이브러리를 노출합니다.

## 4. 품질 기준 (조직 규칙) {#4-quality-bar-org-rules}

- 순수 Go, **CGO=0**, 벤더링 없음 — 모든 대상에서 소스로부터 빌드합니다.
- **구문 커버리지 100%**를 CI에서 강제하며, 테스트 러너의 종료 코드는 커버리지 수치만이 아니라 반드시 0이어야 합니다.
- 저장소 콘텐츠는 영어로만 작성합니다.
