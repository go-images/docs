---
title: "ロードマップ"
weight: 30
---

# 実装計画 — 純粋な Go の画像処理（`go-images/images`） {#implementation-plan-pure-go-image-processing-go-imagesimages}

> 目標: scikit-image スタイルの、**純粋な Go（cgo なし）** の画像処理ライブラリを、Go 標準ライブラリの上に構築することです。**組み込み可能**（ネイティブな依存がない）で、あらゆる Go ターゲットに**クロスコンパイル可能**であり、
> [go-embedded-ruby](https://github.com/go-embedded-ruby/ruby) の画像バックエンドとして想定されています。

## 1. なぜ純粋な Go なのか {#1-why-pure-go}

主流の画像スタックは、どれもネイティブの C ライブラリを必要とします。

- **ruby-vips** → libvips、**RMagick** → ImageMagick。
- cgo を使う Go のラッパー（`govips`、`go-imagick`）も、同じ共有オブジェクトをリンクします。

そのため、静的ビルド、クロスコンパイル、組み込みが難しくなります。このライブラリは **CGO=0** なので、`go build` 1 回で、対応するどのターゲットでも、システムパッケージなしで自己完結したバイナリが得られます。

### 正直な比較 {#honest-comparison}

純粋な Go の画像ライブラリはすでに存在しており、私たちはそうではないふりをしません。

- **[bild](https://github.com/anthonynsimon/bild)** — 幅広い純粋な Go のツールキットです（ぼかし、エッジ検出、モルフォロジー、ヒストグラム）。成熟していて機能も豊富です。
- **[disintegration/imaging](https://github.com/disintegration/imaging)** — リサイズ・変換・調整に絞った、評価の高いライブラリです。
- **libvips / ImageMagick** — 機能ははるかに豊富で、**手作業でチューニングされた SIMD** を備えています。現時点では大きな画像でずっと高速ですが、cgo に縛られます。

このプロジェクトが差別化を目指す点は次のとおりです。

1. 純粋関数と小さく安定した API を備えた、**クリーンで正確な、100% カバーされたコア**。
2. **カーネルを分離**して `internal/kernels` に置き、狭いシグネチャの背後に隠しているため、公開 API に触れることなく、**[go-asmgen](https://github.com/go-asmgen) で生成した SIMD** へ丸ごと置き換えられます。go-asmgen は、64 ビットの 6 ターゲット（amd64、arm64、riscv64、loong64、ppc64le VSX、s390x ベクトル）すべてに対応する唯一の Go SIMD ジェネレータです。cgo を使わないまま libvips/ImageMagick との速度差を縮める道筋です。
3. go-embedded-ruby の画像バックエンドとしての**組み込みやすさ**。

現時点で libvips/ImageMagick より高速だとも、bild より機能が豊富だとも主張しません。賭けているのは、cgo 不要であること、組み込めること、そして 6 つのアーキテクチャにわたる SIMD の道筋です。

## 2. アーキテクチャ {#2-architecture}

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

RGBA のレイアウト（行優先、1 ピクセル 4 バイト）は意図的なものです。ベクトルレジスタにそのまま対応する形だからです。

## 3. ロードマップ {#3-roadmap}

### フェーズ 0 — コア（完了） {#phase-0-core-done}

- `ToRGBA` 変換ヘルパー。
- 入出力: `Load`、`Save`（拡張子による PNG/JPEG）、`Decode`、`Encode`。
- 点演算: `Grayscale`（Rec. 601 輝度）、`Invert`、`AdjustBrightness`、`AdjustContrast`（どちらも `[0,255]` にクランプ）。
- 最近傍法とバイリニアのモードを持つ `Resize`。
- 任意の奇数サイズの float カーネルと、端へのクランプによる境界処理を持つ `Convolve`。
- 分離可能な畳み込みとしての `GaussianBlur`。
- ステートメントカバレッジ 100%。CI は 3 つの OS と 64 ビットの 6 アーキテクチャ（ネイティブの amd64/arm64、qemu の riscv64/loong64/ppc64le/s390x）で実行します。

### フェーズ 1 — フィルタと変換の拡充（進行中） {#phase-1-more-filters-transforms-in-progress}

- **Sobel エッジ検出（完了）** — `Sobel`（勾配の大きさ）、`SobelX`/`SobelY`（方向ごとの応答）。いずれも Rec. 601 輝度に対して、端へのクランプによる境界処理で動作します。輝度プレーンは 1 回だけ計算し、勾配/大きさのパスは、展開したカーネルタップと融合しています。ベンチマークで、融合しない 2 パスのベースラインと比較し、差分テストで、独立した素朴なリファレンスと一致することを確認しています。
- **ボックスブラー（完了）** — `BoxBlur`。分離可能な移動和（半径に対して 1 ピクセルあたり O(1)）で、float の中間結果を使うため、2 つのパスは正確な 2 次元の平均になります。`scipy.ndimage.uniform_filter`（mode=nearest）とビット単位で一致します。独立した素朴なオラクルと、VM 上の SciPy に対して検証しています。
- **幾何変換（完了）** — `FlipHorizontal`/`FlipVertical`（numpy.fliplr/flipud）、`Rotate90`/`Rotate180`/`Rotate270`（numpy.rot90）、`Crop`。補間を行わない、正確なピクセルの並べ替えです。
- **色としきい値（完了）** — `RGBToHSV`/`HSVToRGB`（バイトエンコード、往復で安定）、`OtsuThreshold`（`skimage.filters.threshold_otsu` と一致）、`Threshold`/`Otsu`。
- **エッジ演算子（完了）** — `Prewitt`、`Scharr`、`SobelMag`（正規化した分離可能な勾配の大きさで、[0,1] の輝度に対する `sqrt((gx²+gy²)/2)`）と、`Laplacian`（離散の `[0,-1,0;-1,4,-1;0,-1,0]`）。VM 上で `skimage.filters.{prewitt,scharr,sobel,laplace}` とビット単位で一致します（3 タップのカーネルでは、端へのクランプが skimage の反射境界と等しくなります）。
- **Canny（完了）** — `Canny(sigma, low, high)`: ガウスで平滑化した輝度 → Sobel 勾配 → バイリニアの非極大抑制 → 8 近傍連結のヒステリシスという流れで、`skimage.feature.canny` を模しています。内部領域での skimage との一致は約 98%（mode=nearest）です。残りの差は、skimage のマスク付き定数境界の前処理と、このライブラリのより素直な端へのクランプの違いによるものです。
- **Median + unsharp（完了）** — `Median`（256 ビンの計数による選択を用いる正方形のメディアンで、`scipy.ndimage.median_filter` の `mode="nearest"` とビット単位で一致）と、`UnsharpMask`/`Sharpen`（`src + amount*(src − blur)`。`skimage.filters.unsharp_mask` の式と、丸めの範囲内で正確に一致）。
- バイラテラルフィルタ、エンボス。
- 任意角度の回転、アフィン/ワープ、バイキュービックのリサイズ。
- 色調整: ガンマ、チャンネルミキシング、RGB↔Lab、ヒストグラム平坦化。
- 設定可能な端の処理（clamp / wrap / reflect / constant）。

### フェーズ 2 — モルフォロジーと解析 {#phase-2-morphology-analysis}

- **グレースケールモルフォロジー（完了）** — `Erode`/`Dilate`（正方形の構造要素に対する、チャンネルごとの分離可能な局所最小/最大で、`scipy.ndimage.grey_erosion`/`grey_dilation` とビット単位で一致）と、そこから導かれる `Open`/`Close`。2 値は 0/255 です。**O(1) の van Herk / Gil-Werman** の移動最小/最大（1 ピクセルあたり 3 回の比較で、半径に依存しない）で動作し、4096² のシングルスレッドで scikit-image と同等 → 1.02×、全コアで 4.8–7.8× に達します。
- 連結成分、ラベリング、領域プロパティ。
- ヒストグラム（`kernels.Histogram` はすでにあります）とヒストグラム平坦化。

### 性能 {#performance}

[性能]({{< relref "performance.md" >}})のページでは、go-images と scikit-image / SciPy / OpenCV の正直なベンチマークを追跡しています。go-images は、分離可能な負荷の高い演算で scikit-image を上回っています（ボックスブラー 1.8–2.6×、RGB→HSV 約 4.8×、反転 約 2×、ガウスは現在 1.0–1.3×）。**モルフォロジーは scikit-image と同等に達しました**（erode/dilate は 4096² のシングルスレッドで同等 → 1.02×、全コアで 4.8–7.8×）。これは O(1) の van Herk / Gil-Werman への書き換えによるものです。**Sobel も同等に達しました**（シングルスレッドで 1.02–1.06×、全コアで 4.2–4.8×）。こちらは分岐なしの内側区間への書き換えによるものです。シングルスレッドで残る差は純粋な定数倍です（小さい半径のモルフォロジーと、ピクセルごとではない Rotate90/Crop/Otsu のリファレンス演算）。これを埋めること（そして OpenCV の O(1)+SIMD モルフォロジーに並ぶこと）が、BENCHMARKS.md で追跡している SIMD の作業です。

### フェーズ 3 — go-asmgen による SIMD カーネル {#phase-3-simd-kernels-via-go-asmgen}

- `internal/kernels` のスカラループを、64 ビットの Go 6 ターゲットすべてで SIMD に置き換えます。スカラ版は、フォールバックであり正しさのオラクルでもあるものとして残します（差分テスト: SIMD の出力はスカラの出力と等しくなければなりません）。
- 高速化を測定し、libvips/ImageMagick に対して追跡するためのベンチマークスイート。

### フェーズ 4 — Ruby バインディング {#phase-4-ruby-binding}

- ruby-vips/RMagick に代わる cgo 不要の選択肢として、[go-embedded-ruby](https://github.com/go-embedded-ruby/ruby) を通じてライブラリを公開します。

## 4. 品質基準（組織のルール） {#4-quality-bar-org-rules}

- 純粋な Go、**CGO=0**、vendoring なし — あらゆるターゲットでソースからビルドします。
- **ステートメントカバレッジ 100%** を CI で強制します。テストランナーの終了コードは 0 でなければなりません（カバレッジの数値だけでは足りません）。
- リポジトリの内容は英語のみです。
