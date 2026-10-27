---
title: "首页"
weight: 10
---

# go-images 文档 {#go-images-documentation}

**一个纯 Go（无 cgo）、scikit-image 风格的图像处理库。** 基于 RGBA 光栅图像提供滤波、
边缘检测、形态学、几何变换和颜色变换，不依赖 OpenCV、libvips 或 ImageMagick。

在大多数语言中，图像处理意味着绑定一个 C 库。go-images 则是单个可移植的模块，
可以交叉编译为静态二进制文件，并通过 **go-asmgen SIMD** 加速可分离的热点运算，
因此在方框模糊（box blur）和 Sobel 上**胜过 scikit-image**。**100% 覆盖率**，并已对照 scikit-image 验证。

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

## API 概览 {#api-surface}

| 领域 | 函数 |
| --- | --- |
| I/O | `Load`, `Save`, `Decode`, `Encode`, `ToRGBA` |
| 滤波 | `GaussianBlur`, `BoxBlur`, `Median`, `Sharpen`, `UnsharpMask` |
| 边缘 | `Sobel`, `SobelX`, `SobelY`, `SobelMag`, `Prewitt`, `Scharr`, `Laplacian`, `Canny` |
| 形态学 | `Erode`, `Dilate`, `Open`, `Close` |
| 几何 | `Resize`, `Rotate90`/`180`/`270`, `Crop`, `FlipHorizontal`, `FlipVertical` |
| 颜色 | `Grayscale`, `Invert`, `RGBToHSV`, `HSVToRGB`, `OtsuThreshold`, `Threshold`, `AdjustBrightness`, `AdjustContrast` |

## 性能与架构 {#performance-architectures}

可分离的热点运算在 amd64（SSE2）、arm64（NEON）和 s390x（向量指令）上由 go-asmgen SIMD 加速；
全部六个 64 位目标平台都运行可移植的标量核心。go-images 在方框模糊上**胜过 scikit-image**
（单线程 1.8–2.6×，使用全部核心时最高 10.3×），在 Sobel 上现已与之持平或更快
（单线程 1.02–1.06×，使用全部核心时最高 4.8×）；基准测试页面如实说明了哪些方面尚未做到。

## 下一步 {#where-to-go-next}

- [模块]({{< relref "modules.md" >}})：本组织的七个模块，包括核心库、
  `depth`、Photoshop 读取器、Go 自带编解码器的三个分支，以及
  JPEG 2000 编解码器。
- [路线图]({{< relref "roadmap.md" >}})：计划，以及目前已经发布的内容。
- [性能]({{< relref "performance.md" >}})：与 scikit-image 的如实基准对比。

源码：[github.com/go-images/images](https://github.com/go-images/images) ·
同时也是 [go-embedded-ruby](https://github.com/go-embedded-ruby/ruby) 的
`Image` 类背后无 cgo 的图像处理器。
