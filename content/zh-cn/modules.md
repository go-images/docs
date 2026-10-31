---
title: "模块"
weight: 20
---

# 模块 {#modules}

`go-images` 共有 **7 个模块**。每个模块都链接到其源码和自动生成的
API 参考文档。

| 模块 | 简介 | 参考 |
| --- | --- | --- |
| [`images`](https://github.com/go-images/images) | 核心库：基于 RGBA 光栅图像的滤波、边缘检测、形态学、几何变换和颜色变换，带有 go-asmgen SIMD 内核；像素运算同样位于不含编解码器的 [`images/ops`](#images-ops-pixels-without-codecs) 中 | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/images) |
| [`depth`](https://github.com/go-images/depth) | 一张图进，两只眼出：从图像自身的线索估计深度，并合成双眼视图 | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/depth) |
| [`psd`](https://github.com/go-images/psd) | Photoshop 的 `.psd`/`.psb`，读取拼合后的合成图像，供 `image.Decode` 使用 | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/psd) |
| [`png`](https://github.com/go-images/png) | Go `image/png` 的分支，其 `DecodePartial` 会说明被截断的数据流解码出了哪些部分 | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/png) |
| [`jpeg`](https://github.com/go-images/jpeg) | Go `image/jpeg` 的分支，对四分量 JPEG 的色度进行上采样时与 libjpeg 的做法一致 | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/jpeg) |
| [`gif`](https://github.com/go-images/gif) | Go `image/gif` 的分支，提供与 `png` 相同的 `DecodePartial` | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/gif) |
| [`jpeg2000`](https://github.com/go-images/jpeg2000) | 纯 Go 的 JPEG 2000 编解码器，派生自 ajroetker/go-jpeg2000：CMYK 按油墨读取，并修复了分块编码 | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/jpeg2000) |

## images/ops：不含编解码器的像素运算 {#images-ops-pixels-without-codecs}

⛔ **状态：已合并到 main，将随下一个发布推出。** 它已经合并但尚未打标签，
因此在最新标签上执行 `go get` 得不到它。

`github.com/go-images/images/ops` 是 `images` 的子包，包含全部像素运算：滤波、边缘检测、
形态学、几何变换和颜色变换。根包 `images` 以相同的名称转发每一个名称，因此现有代码
仍能照常编译，并继续保留 I/O：`Load`、`Save`、`Decode`、`Encode`、
`DecodePartial` 和 EXIF。

原因在于链接，而不在 API。解码器在 `init` 中自行注册，因此
导入 `images` 会链接**所有解码器**，约 1.7 MB 的 WebAssembly 代码，
无论程序是否读取文件。`ops` 不导入任何编解码器。

```go
import "github.com/go-images/images/ops" // pixels only: no decoder is linked

blurred, err := ops.GaussianBlur(src, 3)
```

只处理手头已有像素的程序应当导入 `ops`。需要读写文件的
程序则像以前一样导入 `images`。控件工具包（widget toolkit）出于这个原因把它的 `Material` 和 `Thumbnail` 迁到了 `ops`，参见
[二进制体积](https://go-widgets.github.io/docs/latest/rendering/toolkit/#binary-size)。

## jpeg2000：因修复而保留的分支 {#jpeg2000-a-fork-kept-for-its-fixes}

`jpeg2000` 无需 C 即可解码和编码 JPEG 2000 码流（`.j2k`）和 JP2 文件
（`.jp2`）。它是
[ajroetker/go-jpeg2000](https://github.com/ajroetker/go-jpeg2000) 的分支，按 Apache-2.0 的要求，
每一处改动都列在它的 `NOTICE` 中。go-pdfkit/render
和 go-gfx/gfx 直接依赖它，而在 2026-10-18，另有 45 个组织内的
模块通过它们间接依赖它。

- JP2 头声明为 **CMYK** 的四分量文件会解码为
  `*image.CMYK`。若按红、绿、蓝读取，就会丢失黑色版：与 poppler 相比，每个像素都相差 255
  级，现在在 2 440 044 个采样上最多相差 2。
- 分块编码把码块和图块放在解码器和 OpenJPEG 读取它们的位置
  （ITU-T T.800 B.7），并且所有码块都被量化为零的图块，也会像参考实现那样被接受。
- SIMD 依赖是 [go-simd/highway](https://go-simd.github.io/docs/latest/repos/highway/)，
  因为使用方无法针对上游 go-highway 的各发布版本执行 `go mod tidy`。
- ⛔ 无损颜色变换和逆 5/3 小波变换**不是**
  SIMD 调用。在 amd64 上，不带 `GOEXPERIMENT=simd` 时，highway 采用一个通用的
  回退实现，每次向量运算都会分配内存。
  - 对一帧 4K 画面做颜色变换耗时 408.8 ms 和 1245 万次内存分配，
    而普通循环只需 17.9 ms 和 6 次。在 arm64 上，普通循环同样
    胜过 NEON 路径，14.1 ms 对 29.9 ms，因为把各个平面复制进出
    的开销超过了内核节省的时间。
  - 把小波变换的垂直方向处理改为整行一次、原地进行后，
    amd64 上 1920×1080 的无损解码从 638 ms 和 1455 万次内存分配降到
    269 ms 和 9.3 万次，与 arm64 的分配次数相同，而 arm64 上的
    解码并没有更慢（−1.5 %）。
- ⛔ **自 v0.13.8 起做模糊测试。** 此前没有任何模糊测试目标，第一个目标
  在几秒内就发现了三处 panic，全都能从 `DecodeConfig` 触发。其中一处只有
  14 个字节，能让任何通过 go-pdfkit/render 渲染 PDF 的程序崩溃。现在头部
  按 OpenJPEG 的方式校验，这三个输入保留为回归测试。`Decode` 仍然**没有
  像素数上限**：请先调用 `DecodeConfig`，拒绝无法承受的图像。

## ⛔ 为什么要分叉 Go 自带的三个编解码器 {#why-three-forks-of-gos-own-codecs}

并不是为了改变它们解码出的结果。每个分支都把差异记录在 `NOTICE` 中，
而 Go 自身的测试套件在实质上原样通过。

`png` 和 `gif` 增加了 `DecodePartial`：返回从顶部起已完整的行，
以及使解码中止的错误。**仍在传输中的图像
与解码失败的图像并不是一回事**，而 `image.Decode` 无法向
调用方说明这种区别，因为它在设计上就是要么全部成功、要么全部失败。

`jpeg` 只改变一件事：**四分量** JPEG 的色度通过
在采样点之间插值来上采样，与 libjpeg 的做法一致，而不是把
每个采样点重复一遍。正是这一点，使这里的 CMYK 扫描图曾比其他任何地方
都更模糊。

## ⛔ 为什么读取器要自行注册 {#why-a-reader-registers-itself}

`psd` 是为了其**副作用**而导入的：

```go
import _ "github.com/go-images/psd"

m, _, err := image.Decode(f)   // a .psd now decodes like a .png
```

`image.Decode` 会查询一个注册表，因此只要程序添加这一行导入，就获得了该格式的支持，
而无需在别处再提及它。[go-pdfkit](https://go-pdfkit.github.io/) 正是这样
只用一行代码、不含任何 PSD 专用代码，就把 Photoshop 文件变成 PDF
页面的。

它读取的是 Photoshop 写在文件末尾的、拼合后的**合成图像**。
读取图层就意味着要理解混合模式、剪贴、蒙版和
调整图层，也就是重新实现一个合成器，而结果仍会
与 Photoshop 显示的不同。合成图像*就是* Photoshop 显示的内容。

⛔ 它的两处读取方式，恰恰是让其他所有 PSD 读取器出错的地方，因此
它们有各自专门的测试：**CMYK 是反相存储的**（255 表示没有油墨，
按字节原样读取会把白色页面变成黑色），以及 **alpha 是直通的（straight）**
（做预乘会使每个半透明像素都偏亮，而在不透明的文件上这一点是看不出来的）。以 macOS 自带的 ImageIO（通过 `sips`）为基准，
该解码器在一个真实 1920×1080 文件的 **2 073 600 个像素上完全一致**。
