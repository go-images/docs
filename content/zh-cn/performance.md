---
title: "性能"
weight: 40
---

# 性能：`go-images/images` 对比 scikit-image / SciPy {#performance-go-imagesimages-vs-scikit-image-scipy}

> **基准：** 在*同一台机器*上，以真实的图像尺寸，对语义匹配的运算做
> 诚实、同口径的计时。我们既展示胜出的项目，
> **也**展示落后的项目："on n'a pas le droit de se tromper"，虚假的胜利比真实的落后更糟。

## 方法 {#method}

- **硬件：** 一台 Tart `debian` 虚拟机，`aarch64`（arm64）Linux，内核
  6.12，4 个 vCPU。两个软件栈在*同一台*虚拟机上先后运行。
- **图像：** 确定性生成的 `512×512`（以及 `1024×1024`）RGB(A) 图像
  （xorshift / NumPy 伪随机数生成器）。
- **go-images：** Go 1.26.4，`GOWORK=off`，`go test -bench`，时间 = ns/op
  （取各计时迭代中的最佳值）。颜色运算在 RGBA 上运行；对照基准
  则对三个颜色通道分别运行匹配的运算。
- **scikit-image / SciPy：** Debian 系统软件包
  （`python3-skimage` 0.25、`python3-scipy` 1.15、`python3-numpy` 2.2），
  使用 `time.perf_counter`，预热后取 20 次运行中的最佳值。这里的 NumPy 是
  带 SIMD 的单线程 C。
- **匹配的语义**（这是让对比公平的关键）：
  - 方框模糊半径 5 ⇔ 对每个通道执行 `scipy.ndimage.uniform_filter(size=11, mode="nearest")`。
  - 高斯 σ=3 ⇔ 对每个通道执行 `scipy.ndimage.gaussian_filter(sigma=3, truncate=3,
    mode="nearest")`（truncate 取 3，因为我们的半径是 `ceil(3σ)`）。
  - Sobel ⇔ 在亮度平面上执行 `skimage.filters.sobel`（梯度幅值）。
  - 腐蚀/膨胀半径 2 ⇔ 对每个通道执行 `scipy.ndimage.grey_erosion` /
    `grey_dilation(size=(5,5), mode="nearest")`。

热点的可分离内核现在运行**SIMD 内循环 + 多核分块**：

- 一个**乘加（axpy）SIMD 内核**，即在连续的 float64 平面上执行
  `dst[i] += a*src[i]`，驱动高斯可分离卷积。各通道被解交织为连续的平面，
  这样每个内核抽头就是一次扁平的 axpy；两趟处理之间按字节取整的中间结果保持不变，
  因此数值输出与原先的标量代码完全一致。
- 一个**逐元素 min/max SIMD 内核**，即 `dst[i] = min/max(dst[i], src[i])`，
  同样在连续平面上驱动可分离的灰度形态学运算。
- **多核分块**在像素数超过阈值时，把相互独立的行（水平方向处理）和列
  （垂直方向处理）分配给 `GOMAXPROCS` 个 goroutine；低于阈值时则运行串行路径。
  结果与工作线程数无关。

各架构的 SIMD 覆盖情况（`go-fft` / `go-ndarray` 所记录的多架构方案）：
**amd64 SSE2**、**arm64 NEON**（axpy 使用 VFMLA；Go 汇编器没有对应助记符的
双精度向量 FMIN/FMAX，则以原始指令字的形式写出，编码已在真实硬件上验证），
以及 **s390x z/vector**（VFMADB + VFMINDB/VFMAXDB，大端序，但这些逐元素内核
与通道顺序无关；操作数顺序已在 qemu-s390x 下验证）。在 **loong64 /
ppc64le / riscv64** 上，Go 汇编器没有可用的双精度向量运算
（对 riscv64 而言，则是在默认的 qemu CPU 下 V 扩展会出错），因此这些平台
保留**标量内循环加多核分块**，仍然能获得多核带来的提速。
每个 SIMD 内核都已与其标量参照逐位验证（axpy 采用严格的容差，因为 GOAMD64=v3 构建的参照会融合为 VFMADD），
而整个运算也已对照 SciPy 验证。

## 结果：512×512（arm64 Linux，越低越好） {#results-512512-arm64-linux-lower-is-better}

| 运算                    | go-images **优化前** | go-images **优化后** | scikit-image / SciPy | 优化后对比 SciPy |
|-------------------------|---------------------:|--------------------:|---------------------:|---------------:|
| **方框模糊**（r=5）      |            2,812,064 |             870,237 |           10,999,974 | **快 12.6×** |
| **Sobel**（幅值）   |              882,498 |             351,701 |            2,748,223 | **快 7.8×** |
| **高斯模糊**（σ=3） |           12,900,987 |           4,259,656 |           16,029,709 | **快 3.8×** |
| **腐蚀**（r=2）         |           13,865,054 |           1,581,921 |           10,559,638 | **快 6.7×** |
| **膨胀**（r=2）        |           12,898,691 |           1,482,165 |           11,025,974 | **快 7.4×** |

### 第 1 阶段新增的边缘 / 滤波运算：512×512（arm64 Linux，越低越好） {#phase-1-edge-filter-additions-512512-arm64-linux-lower-is-better}

这些是较新的多核（标量内循环）运算，在同一台虚拟机上
对照匹配的 scikit-image / SciPy 参照做了基准测试。它们目前还没有 SIMD 内循环，
提速来自多核分发以及更廉价的算法（中值采用计数选择），
即便如此，它们已经胜过 scikit-image：

| 运算                    | go-images (ns/op) | scikit-image / SciPy (ns/op) | 加速比 |
|-------------------------|------------------:|-----------------------------:|---------:|
| **Scharr**（幅值）  |         1,421,773 |                    3,833,147 | **快 2.7×** |
| **Canny**（σ=1）         |        10,086,768 |                   27,031,572 | **快 2.7×** |
| **中值**（r=2，5×5）   |        37,565,475 |                  214,597,360 | **快 5.7×** |

匹配的语义：Scharr ⇔ 在亮度平面上执行 `skimage.filters.scharr`；
Canny ⇔ `skimage.feature.canny(sigma=1, low=20, high=40)`；中值 ⇔
对每个通道执行 `scipy.ndimage.median_filter(size=5, mode="nearest")`。Prewitt 和
SobelMag 与 Scharr 共用内核；UnsharpMask/Sharpen 是 GaussianBlur 加上一个
融合的逐像素加法（适用上文高斯的数字）。中值的大幅领先来自
O(window+256) 的计数选择，对比 SciPy 通用的 n 维中值，再加上
多核分块进一步拉大了差距。加入一个 SIMD Sobel 梯度内核（由边缘
系列和 Canny 共用）是下一个向量化目标。

“优化前”一列是标量单线程内核；“优化后”是 SIMD +
4 核。**此前的两个落后项现在都已胜出：** 高斯从 0.92×（慢 1.08 倍）
变为**快 3.8×**，腐蚀/膨胀从约 0.9× 变为 **快 6.7×/7.4×**。
本来就领先的方框模糊和 Sobel 则进一步扩大了优势（仅多核
分块一项，在 4 核上就使它们大致提速到三倍）。

## 结果：1024×1024（arm64 Linux，越低越好） {#results-10241024-arm64-linux-lower-is-better}

| 运算                    | go-images (ns/op) | scikit-image / SciPy (ns/op) | 加速比 |
|-------------------------|------------------:|-----------------------------:|---------:|
| **方框模糊**（r=5）      |         7,548,773 |                   29,963,031 | **快 4.0×** |
| **Sobel**（幅值）   |         6,003,498 |                   11,327,476 | **快 1.9×** |
| **高斯模糊**（σ=3） |        11,451,846 |                   46,418,949 | **快 4.1×** |
| **腐蚀**（r=2）         |         7,896,837 |                   59,335,849 | **快 7.5×** |
| **膨胀**（r=2）        |         7,893,893 |                   61,166,651 | **快 7.7×** |

在 1024² 下，对于 scipy 要生成大型中间数组的运算（形态学、高斯），领先幅度进一步扩大；
Sobel 则缩小到 1.9×，因为它每个像素的运算量很小，
胜出主要靠多核分发（SciPy 只有一次边缘
滤波，在这个尺寸下没有逐通道的 Python 开销）。

### 各组件的提速（512²，优化前 → 优化后） {#per-component-speed-up-512-before-after}

- **高斯：比标量基线快 3.0×**（现在已领先于 SciPy）：
  axpy SIMD 内核把密集的乘加向量化，两趟处理则分布到各个核心上。
  垂直方向的一趟是纯粹的连续整行 axpy；水平方向的一趟对内部的列做向量化，
  对少数受边界钳位的列则用标量处理。
- **腐蚀/膨胀：比标量基线快约 8.7×：** 逐窗口的滑动
  min/max 改为在各平面上做连续的 SIMD MIN/MAX 加多核分块，
  并且逐行的临时空间被提到内循环之外（内存分配
  从约 6,200 次/op 降到 71 次/op）。

> **更新：O(1) 的 van Herk / Gil-Werman 形态学。** 形态学内循环
> 此后已从 O(radius) 的折叠改写为 **O(1) 的 van Herk /
> Gil-Werman** 滑动 min/max，每个像素三次比较，与半径无关。在
> 单线程 scikit-image 对等基准（Apple M4 Max）上，腐蚀/膨胀
> 在 4096² 下现已达到 scikit-image 的**持平 → 1.02×**（此前为 0.58–0.77×），使用全部核心时为
> **4.8–7.8×**；输出与
> `scipy.ndimage.grey_erosion`/`grey_dilation` 逐字节一致。小半径下
> 剩余的差距纯粹是常数因子（SIMD）。完整的单线程和
> 多线程对等表格及方法见
> [BENCHMARKS.md](https://github.com/go-images/images/blob/main/BENCHMARKS.md)。

> **更新：无分支的 Sobel。** 上文指出的那个落后项，即逐像素重新计算的 Sobel 梯度，
> 此后已经补上：现在每一行拆分为按钳位寻址的边界列和一段
> **无分支的内部区段**，行基址提到循环之外，因此内部区段不再产生任何
> `clampIndex` 调用。输出与朴素写法逐字节一致。在单线程
> scikit-image 对等基准（Apple M4 Max）上，Sobel 现已达到
> scikit-image 的 **1.02–1.06×**（此前为 0.78–0.80×），使用全部核心时为 **4.2–4.8×**。参见
> [BENCHMARKS.md](https://github.com/go-images/images/blob/main/BENCHMARKS.md)
> 的行动项 B。

## 对照 SciPy 的正确性（不只是速度） {#correctness-vs-scipy-not-just-speed}

同一台虚拟机，一张 `64×48` 的确定性图像，把 go-images 的输出与匹配的 SciPy 参照
逐像素比较，是在 SIMD/多核改写**之后**：

| 运算 | max \|Δ\| | mean \|Δ\| | 结论 |
|-----------|----------:|-----------:|---------|
| 方框模糊  |         0 |      0.000 | **完全一致** |
| 腐蚀     |         0 |      0.000 | **完全一致** |
| 膨胀    |         0 |      0.000 | **完全一致** |
| 高斯  |         1 |      0.079 | 在 1 LSB 以内一致（仅为舍入差异） |

SIMD/多核改写在数值上**没有改变任何东西**：方框模糊和
灰度形态学仍然与 SciPy 逐位一致，高斯也仍然只在最后一次钳位的末位舍入上
有差异，与标量实现的数字相同。每个 SIMD 内核还对照其标量
参照（`internal/kernels/simd_test.go`）检验了每一种长度余数，
而整个可分离运算则在串行和并行两个阈值下，对照一个独立的朴素
参照做了检验（`separable_test.go`）。

### 遗留说明（如实） {#residual-notes-honest}

- **内存：** 基于平面的高斯所分配的临时 float64 暂存空间，比旧的原地字节路径更多
  （512² 下约 14.8 MB/op，而旧路径为 2.1 MB），这是连续 SIMD 布局的代价。
  这部分是临时的（每次调用后释放），换来了 3.8× 的提速；用可复用的暂存池来减少它是后续工作。
- **loong64 / ppc64le / riscv64** 运行标量内循环（Go 汇编器中没有可用的
  双精度向量运算 / qemu 的 V 扩展会出错），仅依靠多核
  分块来提速。它们在各架构的 qemu CI 任务中得到验证。

## 复现 {#reproducing}

```sh
# On the arm64 debian VM, with the repo synced to /tmp/goimages:
cd /tmp/goimages
GOWORK=off /usr/local/go/bin/go test -run=^$ \
  -bench 'BoxBlur|GaussianBlur|Sobel$|Erode|Dilate|Scharr|Canny|Median' -benchtime=2s ./...

# scikit-image / SciPy side (see the harness used to produce the table):
python3 skbench.py      # timings
python3 validate.py     # correctness vs scipy
```
