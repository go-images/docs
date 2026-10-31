---
title: "모듈"
weight: 20
---

# 모듈 {#modules}

`go-images`의 **7개 모듈**입니다. 각 모듈은 소스와 생성된 API 레퍼런스로 연결됩니다.

| 모듈 | 설명 | 레퍼런스 |
| --- | --- | --- |
| [`images`](https://github.com/go-images/images) | 라이브러리 본체: RGBA 래스터 위의 필터, 에지, 모폴로지, 기하, 색상 연산과 go-asmgen SIMD 커널. 픽셀 연산은 코덱이 없는 [`images/ops`](#images-ops-pixels-without-codecs)에도 있습니다 | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/images) |
| [`depth`](https://github.com/go-images/depth) | 사진 한 장을 넣으면 두 눈용 이미지가 나옵니다: 사진 자체의 단서에서 깊이를 추정하고 양쪽 시점을 합성합니다 | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/depth) |
| [`psd`](https://github.com/go-images/psd) | Photoshop `.psd`/`.psb`의 평탄화된 합성본을 `image.Decode`에서 읽습니다 | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/psd) |
| [`png`](https://github.com/go-images/png) | Go의 `image/png` 포크로, `DecodePartial`이 잘린 스트림에서 어디까지 디코딩되었는지 알려 줍니다 | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/png) |
| [`jpeg`](https://github.com/go-images/jpeg) | Go의 `image/jpeg` 포크로, 4성분 JPEG의 색차(chroma)를 libjpeg와 같은 방식으로 업샘플링합니다 | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/jpeg) |
| [`gif`](https://github.com/go-images/gif) | Go의 `image/gif` 포크로, `png`와 같은 `DecodePartial`을 제공합니다 | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/gif) |
| [`jpeg2000`](https://github.com/go-images/jpeg2000) | ajroetker/go-jpeg2000에서 포크한 순수 Go JPEG 2000 코덱: CMYK를 잉크로 읽고, 타일 방식 인코딩을 수정했습니다 | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/jpeg2000) |

## images/ops: 코덱 없는 픽셀 연산 {#images-ops-pixels-without-codecs}

⛔ **상태: main에 있으며 다음 릴리스에 포함됩니다.** 병합은 되었지만 아직 태그가 붙지 않았으므로, 최신 태그에서 `go get`을 하면 포함되어 있지 않습니다.

`github.com/go-images/images/ops`는 `images`의 하위 패키지로, 필터, 에지, 모폴로지, 기하, 색상 등 모든 픽셀 연산을 담고 있습니다. 루트 패키지 `images`는 각 이름을 같은 이름으로 전달(forward)하므로 기존 코드는 그대로 컴파일되며, 입출력인 `Load`, `Save`, `Decode`, `Encode`, `DecodePartial`, EXIF는 계속 루트 패키지에 있습니다.

이유는 API가 아니라 링크(link)에 있습니다. 디코더는 `init`에서 스스로를 등록하므로, `images`를 임포트하면 프로그램이 파일을 읽든 읽지 않든 **모든 디코더**, 즉 WebAssembly 코드 약 1.7 MB가 링크됩니다. `ops`는 어떤 코덱도 임포트하지 않습니다.

```go
import "github.com/go-images/images/ops" // pixels only: no decoder is linked

blurred, err := ops.GaussianBlur(src, 3)
```

이미 가지고 있는 픽셀만 처리하는 프로그램은 `ops`를 임포트해야 합니다. 파일을 읽거나 쓰는 프로그램은 이전처럼 `images`를 임포트합니다. 위젯 툴킷(widget toolkit)도 이러한 이유로 `Material`과 `Thumbnail`을 `ops`로 옮겼습니다. 자세한 내용은 [바이너리 크기](https://go-widgets.github.io/docs/latest/rendering/toolkit/#binary-size)를 참고하십시오.

## jpeg2000: 수정 사항 때문에 유지하는 포크 {#jpeg2000-a-fork-kept-for-its-fixes}

`jpeg2000`은 C 없이 JPEG 2000 코드스트림(`.j2k`)과 JP2 파일(`.jp2`)을 디코딩하고 인코딩합니다. [ajroetker/go-jpeg2000](https://github.com/ajroetker/go-jpeg2000)의 포크이며, Apache-2.0이 요구하는 대로 모든 변경 사항이 `NOTICE`에 나열되어 있습니다. go-pdfkit/render와 go-gfx/gfx가 이를 직접 요구하며, 2026-10-18에는 이들을 거쳐 다른 플릿(fleet) 모듈 45개가 이를 요구했습니다.

- JP2 헤더가 **CMYK**라고 선언한 4성분 파일은 `*image.CMYK`로 디코딩됩니다. 빨강, 초록, 파랑으로 읽으면 검정 판(black plate)이 사라졌습니다. poppler와 비교해 모든 픽셀에서 255단계 차이가 났으나, 지금은 2 440 044개 샘플에서 최대 2 차이입니다.
- 타일 방식 인코딩은 코드 블록과 타일을 디코더와 OpenJPEG가 읽는 위치에 배치하며(ITU-T T.800 B.7), 블록이 모두 양자화로 사라지는 타일은 레퍼런스와 마찬가지로 허용됩니다.
- SIMD 의존성은 [go-simd/highway](https://go-simd.github.io/docs/latest/repos/highway/)입니다. 사용하는 쪽이 업스트림 go-highway의 릴리스에 대해 `go mod tidy`를 실행할 수 없기 때문입니다.
- ⛔ 무손실 색상 변환과 역(inverse) 5/3 웨이블릿은 SIMD 호출이 **아닙니다**. amd64에서 `GOEXPERIMENT=simd` 없이 highway는 벡터 연산마다 메모리를 할당하는 일반 폴백(fallback)을 사용합니다.
  - 4K 프레임 하나의 색상 변환은 408.8 ms에 12.45 million번 할당이 일어났으나, 단순 루프는 17.9 ms에 6번이었습니다. arm64에서도 평면을 안팎으로 복사하는 비용이 커널이 아낀 시간보다 커서, 루프가 NEON 경로보다 빨랐습니다(14.1 ms 대 29.9 ms).
  - 웨이블릿의 수직 패스를 한 행씩 제자리에서 처리하도록 바꾸자, amd64에서 무손실 1920×1080 디코딩이 638 ms와 14.55 million번 할당에서 269 ms와 93 thousand번 할당으로 줄었습니다. 이는 디코딩이 더 느리지 않은(−1.5 %) arm64와 같은 횟수입니다.
- ⛔ **v0.13.8부터 퍼징합니다.** 퍼즈 타깃이 없었는데, 처음 만든 타깃이 몇 초 만에 패닉 3개를 찾았습니다. 모두 `DecodeConfig`에서 도달할 수 있었고, 그중 하나는 14바이트로 go-pdfkit/render로 PDF를 그리는 모든 프로그램을 중단시켰습니다. 이제 헤더를 OpenJPEG와 같은 방식으로 검사하며, 세 입력은 회귀 테스트로 남아 있습니다. `Decode`에는 여전히 **픽셀 수 제한이 없습니다**. 먼저 `DecodeConfig`를 호출하고 감당할 수 없는 크기의 이미지는 거부하세요.

## ⛔ Go 자체 코덱의 포크가 세 개인 이유 {#why-three-forks-of-gos-own-codecs}

디코딩 결과를 바꾸려는 것이 아닙니다. 각 포크의 차이는 `NOTICE`에 기록되어 있으며, Go 자체 테스트 스위트는 실질적으로 변경 없이 통과합니다.

`png`와 `gif`는 `DecodePartial`을 추가합니다. 위에서부터 세어 완전한 행들과, 디코딩을 멈춘 오류를 돌려줍니다. **아직 도착하는 중인 사진과 실패한 사진은 다릅니다.** 그런데 `image.Decode`는 설계상 전부 아니면 전무(all-or-nothing)여서 호출자에게 그 차이를 알려 줄 수 없습니다.

`jpeg`는 한 가지를 바꿉니다. **4성분** JPEG의 색차를 각 샘플을 반복하는 대신, libjpeg처럼 샘플 사이를 읽어 업샘플링합니다. 그 때문에 CMYK 스캔이 다른 곳보다 이곳에서 더 흐릿하게 나왔던 것입니다.

## ⛔ 리더가 스스로 등록하는 이유 {#why-a-reader-registers-itself}

`psd`는 **부수 효과(side effect)** 를 위해 임포트합니다.

```go
import _ "github.com/go-images/psd"

m, _, err := image.Decode(f)   // a .psd now decodes like a .png
```

`image.Decode`는 레지스트리를 참조하므로, 임포트를 추가한 프로그램은 다른 곳에 이름을 적지 않고도 그 형식을 얻습니다. [go-pdfkit](https://go-pdfkit.github.io/)이 PSD 전용 코드 없이 한 줄만으로 Photoshop 파일을 PDF 페이지로 바꾸는 방식이 바로 이것입니다.

읽는 대상은 Photoshop이 파일 끝에 기록하는 평탄화된 **합성본(composite)** 입니다. 레이어를 읽으려면 블렌드 모드, 클리핑, 마스크, 조정 레이어를 이해해야 하는데, 이는 합성기(compositor)를 다시 구현하는 일이며 결과도 Photoshop이 보여 주는 것과는 여전히 다를 것입니다. 합성본이 곧 Photoshop이 보여 주는 것입니다.

⛔ 읽기 방식 두 가지는 다른 모든 PSD 리더를 깨뜨리는 것이어서 별도의 테스트가 있습니다. **CMYK는 반전되어 저장되며**(255가 잉크 없음을 뜻하므로 기록된 바이트를 그대로 읽으면 흰 페이지가 검게 변합니다), **알파는 스트레이트(straight)입니다**(미리 곱하면 반투명 픽셀이 모두 너무 밝아지는데, 불투명 파일에서는 눈에 보이지 않습니다). macOS 자체의 ImageIO를 `sips`로 호출한 결과와 비교하면, 디코더는 실제 1920×1080 파일의 **2 073 600개 픽셀에서 정확히 일치**합니다.
