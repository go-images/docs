---
title: "路线图"
weight: 30
---

# 实施计划：纯 Go 图像处理（`go-images/images`） {#implementation-plan-pure-go-image-processing-go-imagesimages}

> 目标：一个 scikit-image 风格的**纯 Go（无 cgo）**图像处理库，
> 构建在 Go 标准库之上。它**可嵌入**（没有
> 原生依赖），并且可以**交叉编译**到所有 Go 目标平台，同时也是
> 面向 [go-embedded-ruby](https://github.com/go-embedded-ruby/ruby) 的图像后端。

## 1. 为什么选择纯 Go？ {#1-why-pure-go}

主流的图像处理软件栈都需要一个原生 C 库：

- **ruby-vips** → libvips，**RMagick** → ImageMagick。
- 基于 cgo 的 Go 封装（`govips`、`go-imagick`）链接的是同样的共享
  对象。

这使得静态构建、交叉编译和嵌入都很痛苦。本库是
**CGO=0**，因此一次 `go build` 就能为任何受支持的目标平台生成自包含的二进制文件，
无需任何系统软件包。

### 如实对比 {#honest-comparison}

纯 Go 的图像库早已存在，我们并不假装它们不存在：

- **[bild](https://github.com/anthonynsimon/bild)**：一个覆盖面很广的纯 Go 工具包
  （模糊、边缘检测、形态学、直方图）。成熟且功能丰富。
- **[disintegration/imaging](https://github.com/disintegration/imaging)**：一个
  专注而广受喜爱的缩放/变换/调整库。
- **libvips / ImageMagick**：功能多得多，并且有**手工调优的 SIMD**；目前
  在大图像上快得多，但受限于 cgo。

本项目希望在以下方面形成差异：

1. **干净、正确、100% 覆盖的核心**，采用纯函数和小而
   稳定的接口。
2. **内核被隔离**在 `internal/kernels` 中，签名精简，因此
   可以整体替换为**用 [go-asmgen](https://github.com/go-asmgen) 生成的 SIMD**
   （唯一覆盖全部六个 64 位目标平台的 Go SIMD 生成器：amd64、arm64、riscv64、loong64、ppc64le
   VSX、s390x 向量指令），而无需改动公共 API。这是在保持无 cgo 的同时
   缩小与 libvips/ImageMagick 速度差距的路径。
3. **可嵌入性**，作为 go-embedded-ruby 的图像后端。

我们**不**声称目前比 libvips/ImageMagick 更快，也不声称比 bild 功能更多。
我们的赌注是：无 cgo + 可嵌入 + 横跨六种架构的 SIMD 方案。

## 2. 架构 {#2-architecture}

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

RGBA 布局（行主序，每像素 4 字节）是刻意选择的：它恰好是
能映射到向量寄存器的形状。

## 3. 路线图 {#3-roadmap}

### 第 0 阶段：核心（已完成） {#phase-0-core-done}

- `ToRGBA` 转换辅助函数。
- I/O：`Load`、`Save`（按扩展名选择 PNG/JPEG）、`Decode`、`Encode`。
- 点运算：`Grayscale`（Rec. 601 亮度）、`Invert`、`AdjustBrightness`、
  `AdjustContrast`（二者都钳位到 `[0,255]`）。
- `Resize`，支持最近邻和双线性两种模式。
- `Convolve`，支持任意奇数尺寸的浮点核和钳位到边缘的边界处理。
- `GaussianBlur`，实现为可分离卷积。
- 100% 语句覆盖率；CI 覆盖三种操作系统和六种 64 位架构（原生
  amd64/arm64，qemu 运行 riscv64/loong64/ppc64le/s390x）。

### 第 1 阶段：更多滤波与变换（进行中） {#phase-1-more-filters-transforms-in-progress}

- **Sobel 边缘检测（已完成）**：`Sobel`（梯度幅值）、
  `SobelX`/`SobelY`（方向响应），均基于 Rec. 601 亮度，
  边界钳位到边缘。亮度平面只计算一次，
  梯度/幅值这一趟与展开的核抽头融合；基准测试
  将其与未融合的两趟基线做对比，并有差分测试把它与
  一个独立的朴素参照对齐。
- **方框模糊（已完成）**：`BoxBlur`，可分离的滑动求和（对半径而言每像素 O(1)），
  使用浮点中间结果，使两趟处理组合后恰好等于二维均值；
  与 `scipy.ndimage.uniform_filter`（mode=nearest）逐位一致。已对照
  一个独立的朴素参照以及虚拟机上的 SciPy 验证。
- **几何变换（已完成）**：`FlipHorizontal`/`FlipVertical`
  （numpy.fliplr/flipud）、`Rotate90`/`Rotate180`/`Rotate270`（numpy.rot90）以及
  `Crop`。精确的像素重排，不做插值。
- **颜色与阈值（已完成）**：`RGBToHSV`/`HSVToRGB`（字节编码，
  往返稳定）、`OtsuThreshold`（与 `skimage.filters.threshold_otsu` 一致）
  以及 `Threshold`/`Otsu`。
- **边缘算子（已完成）**：`Prewitt`、`Scharr` 和 `SobelMag`（归一化的
  可分离梯度幅值，在 [0,1] 的亮度上取 `sqrt((gx²+gy²)/2)`），以及
  `Laplacian`（离散的 `[0,-1,0;-1,4,-1;0,-1,0]`）。在虚拟机上与
  `skimage.filters.{prewitt,scharr,sobel,laplace}` 逐位一致（3 抽头
  内核使钳位到边缘等同于 skimage 的反射边界）。
- **Canny（已完成）**：`Canny(sigma, low, high)`：经高斯平滑的亮度 →
  Sobel 梯度 → 双线性非极大值抑制 → 8 连通滞后阈值处理，
  对应 `skimage.feature.canny`。内部区域与 skimage 约 98% 一致
  （mode=nearest）；剩余差异来自 skimage 带掩码的常量边界预处理
  与本库更干净的钳位到边缘做法之间的区别。
- **中值 + 反锐化（已完成）**：`Median`（通过 256 区间计数
  选择实现的方形中值，与 `scipy.ndimage.median_filter` 的 `mode="nearest"` 逐位一致）
  以及 `UnsharpMask`/`Sharpen`（`src + amount*(src − blur)`，在舍入误差内
  与 `skimage.filters.unsharp_mask` 的公式完全一致）。
- 双边滤波；浮雕。
- 任意角度旋转、仿射/扭曲；双三次缩放。
- 颜色调整：伽马、通道混合、RGB↔Lab、直方图均衡化。
- 可配置的边缘处理（钳位 / 环绕 / 反射 / 常量）。

### 第 2 阶段：形态学与分析 {#phase-2-morphology-analysis}

- **灰度形态学（已完成）**：`Erode`/`Dilate`（在方形结构元素上
  逐通道做可分离的局部 min/max，与 `scipy.ndimage.grey_erosion`/
  `grey_dilation` 逐位一致）以及派生的 `Open`/`Close`。二值图像采用 0/255。
  采用 **O(1) 的 van Herk / Gil-Werman** 滑动 min/max
  （每像素三次比较，与半径无关），在 4096² 下单线程达到
  scikit-image 的持平 → 1.02×，使用全部核心时为 4.8–7.8×。
- 连通分量、标记、区域属性。
- 直方图（`kernels.Histogram` 已存在）和直方图均衡化。

### 性能 {#performance}

[性能]({{< relref "performance.md" >}})页面记录了如实的
go-images 对比 scikit-image / SciPy / OpenCV 的基准。go-images 在
可分离的热点运算上胜过 scikit-image（方框模糊 1.8–2.6×，RGB→HSV 约 4.8×，翻转
约 2×，高斯现为 1.0–1.3×）。**形态学现已达到 scikit-image 的持平水平**
（腐蚀/膨胀在 4096² 下单线程持平 → 1.02×，使用全部核心时为 4.8–7.8×），
这得益于 O(1) 的 van Herk / Gil-Werman 改写；**Sobel 现在也已达到持平**
（单线程 1.02–1.06×，使用全部核心时为 4.2–4.8×），这得益于无分支
内部区段的改写。剩余的单线程差距纯粹是常数
因子（小半径形态学，以及非逐像素的 Rotate90/Crop/Otsu
参照运算）；消除它（并赶上 OpenCV 的 O(1)+SIMD 形态学）是
BENCHMARKS.md 中记录的 SIMD 工作。

### 第 3 阶段：通过 go-asmgen 实现 SIMD 内核 {#phase-3-simd-kernels-via-go-asmgen}

- 把 `internal/kernels` 中的标量循环替换为覆盖全部六个 64 位 Go
  目标平台的 SIMD，同时保留标量版本作为回退和正确性
  参照（差分测试：SIMD 输出必须等于标量输出）。
- 建立基准测试套件，度量提速并对照 libvips/ImageMagick 持续跟踪。

### 第 4 阶段：Ruby 绑定 {#phase-4-ruby-binding}

- 通过
  [go-embedded-ruby](https://github.com/go-embedded-ruby/ruby) 暴露本库，作为
  ruby-vips/RMagick 的无 cgo 替代方案。

## 4. 质量标准（组织规则） {#4-quality-bar-org-rules}

- 纯 Go，**CGO=0**，不使用 vendoring：在每个目标平台上从源码构建。
- **100% 语句覆盖率**，由 CI 强制执行；测试运行器的退出码必须
  为 0（而不仅仅是覆盖率数字）。
- 仓库内容仅使用英文。
