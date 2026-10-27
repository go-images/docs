---
title: "ホーム"
weight: 10
---

# go-images ドキュメント {#go-images-documentation}

**scikit-image スタイルの、純粋な Go（cgo なし）による画像処理ライブラリです。** RGBA ラスタに対するフィルタ、エッジ検出、モルフォロジー、幾何変換、色変換を提供します。OpenCV も libvips も ImageMagick も使いません。

多くの言語では、画像処理といえば C ライブラリのバインディングを意味します。go-images は静的バイナリへクロスコンパイルできる単一のポータブルなモジュールで、分離可能な負荷の高い処理を **go-asmgen SIMD** で高速化しています。そのため、ボックスブラーと Sobel では **scikit-image を上回ります**。**カバレッジは 100%** で、scikit-image との照合による検証も済んでいます。

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

## API の範囲 {#api-surface}

| 分野 | 関数 |
| --- | --- |
| 入出力 | `Load`, `Save`, `Decode`, `Encode`, `ToRGBA` |
| フィルタ | `GaussianBlur`, `BoxBlur`, `Median`, `Sharpen`, `UnsharpMask` |
| エッジ | `Sobel`, `SobelX`, `SobelY`, `SobelMag`, `Prewitt`, `Scharr`, `Laplacian`, `Canny` |
| モルフォロジー | `Erode`, `Dilate`, `Open`, `Close` |
| 幾何変換 | `Resize`, `Rotate90`/`180`/`270`, `Crop`, `FlipHorizontal`, `FlipVertical` |
| 色 | `Grayscale`, `Invert`, `RGBToHSV`, `HSVToRGB`, `OtsuThreshold`, `Threshold`, `AdjustBrightness`, `AdjustContrast` |

## 性能とアーキテクチャ {#performance-architectures}

分離可能な負荷の高い処理は、amd64（SSE2）、arm64（NEON）、s390x（ベクトル）で go-asmgen SIMD により高速化されています。64 ビットの 6 ターゲットすべてでは、ポータブルなスカラコアが動作します。go-images はボックスブラーで **scikit-image を上回り**（シングルスレッドで 1.8–2.6 倍、全コアで最大 10.3 倍）、Sobel でも scikit-image と同等かそれ以上になっています（シングルスレッドで 1.02–1.06 倍、全コアで最大 4.8 倍）。ベンチマークのページでは、上回れていない箇所についても正直に示しています。

## 次に読むページ {#where-to-go-next}

- [モジュール]({{< relref "modules.md" >}}) — この組織の 7 つのモジュールです。ライブラリ本体、`depth`、Photoshop リーダー、Go 標準コーデックの 3 つのフォーク、そして JPEG 2000 コーデックを扱います。
- [ロードマップ]({{< relref "roadmap.md" >}}) — 計画と、現在出荷されているものを示します。
- [性能]({{< relref "performance.md" >}}) — scikit-image との正直なベンチマークです。

ソース: [github.com/go-images/images](https://github.com/go-images/images) ·
[go-embedded-ruby](https://github.com/go-embedded-ruby/ruby) の
`Image` クラスの背後にある、cgo 不要の画像プロセッサでもあります。
