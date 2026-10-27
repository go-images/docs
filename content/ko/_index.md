---
title: "홈"
weight: 10
---

# go-images 문서 {#go-images-documentation}

**순수 Go(cgo 없음)로 작성된 scikit-image 스타일의 이미지 처리 라이브러리입니다.** RGBA 래스터(raster) 위에서 필터, 에지 검출기, 모폴로지, 기하 변환, 색상 변환을 제공하며, OpenCV, libvips, ImageMagick은 필요하지 않습니다.

대부분의 언어에서 이미지 처리는 C 라이브러리를 바인딩하는 일을 뜻합니다. go-images는 정적 바이너리로 크로스 컴파일되는 단일 이식형 모듈이며, 분리 가능한(separable) 핵심 연산에는 **go-asmgen SIMD**가 적용되어 박스 블러와 Sobel에서 **scikit-image를 앞섭니다**. **100% 커버리지**를 갖추었고 scikit-image와 대조하여 검증했습니다.

```go
import img "github.com/go-images/images"

src, err := img.Load("photo.png")
if err != nil {
    log.Fatal(err)
}
blurred, err := img.GaussianBlur(src, 1.0)
if err != nil {
    log.Fatal(err)
}
edges := img.SobelMag(blurred)
if err := img.Save("edges.png", edges); err != nil {
    log.Fatal(err)
}
```

## API 개요 {#api-surface}

| 영역 | 함수 |
| --- | --- |
| 입출력 | `Load`, `Save`, `Decode`, `Encode`, `ToRGBA` |
| 필터 | `GaussianBlur`, `BoxBlur`, `Median`, `Sharpen`, `UnsharpMask` |
| 에지 | `Sobel`, `SobelX`, `SobelY`, `SobelMag`, `Prewitt`, `Scharr`, `Laplacian`, `Canny` |
| 모폴로지 | `Erode`, `Dilate`, `Open`, `Close` |
| 기하 | `Resize`, `Rotate90`/`180`/`270`, `Crop`, `FlipHorizontal`, `FlipVertical` |
| 색상 | `Grayscale`, `Invert`, `RGBToHSV`, `HSVToRGB`, `OtsuThreshold`, `Threshold`, `AdjustBrightness`, `AdjustContrast` |

## 성능 및 아키텍처 {#performance-architectures}

분리 가능한 핵심 연산은 amd64(SSE2), arm64(NEON), s390x(벡터)에서 go-asmgen SIMD로 가속되며, 64비트 대상 여섯 가지 모두에서 이식형 스칼라 코어가 동작합니다. go-images는 박스 블러에서 **scikit-image를 앞서며**(단일 스레드 1.8–2.6배, 전체 코어 사용 시 최대 10.3배), Sobel에서도 이제 동등하거나 앞섭니다(단일 스레드 1.02–1.06배, 전체 코어 사용 시 최대 4.8배). 벤치마크 페이지는 그렇지 못한 부분도 솔직하게 밝힙니다.

## 다음으로 읽을 문서 {#where-to-go-next}

- [모듈]({{< relref "modules.md" >}}) — 이 조직의 일곱 개 모듈: 라이브러리, `depth`, Photoshop 리더, Go 자체 코덱의 포크 세 개, 그리고 JPEG 2000 코덱.
- [로드맵]({{< relref "roadmap.md" >}}) — 계획과 현재 제공되는 기능.
- [성능]({{< relref "performance.md" >}}) — scikit-image와의 솔직한 벤치마크 비교.

소스: [github.com/go-images/images](https://github.com/go-images/images) ·
[go-embedded-ruby](https://github.com/go-embedded-ruby/ruby)의 `Image` 클래스 뒤에서 동작하는 cgo 없는 이미지 프로세서이기도 합니다.
