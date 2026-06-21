# go-images documentation

**A pure-Go (no cgo) scikit-image-style image-processing library.** Filters,
edge detectors, morphology, geometry and colour transforms over an RGBA raster —
no OpenCV, no libvips, no ImageMagick.

Image processing in most languages means binding a C library. go-images is a
single portable module that cross-compiles to a static binary, accelerated by
**go-asmgen SIMD** on the separable hot operations — so it **beats scikit-image**
on box blur and Sobel. **100% coverage**, validated against scikit-image.

```go
import img "github.com/go-images/images"

src, _ := img.Load("photo.png")
edges := img.SobelMag(img.GaussianBlur(src, 1.0))
img.Save("edges.png", edges)
```

## API surface

| Area | Functions |
| --- | --- |
| I/O | `Load`, `Save`, `Decode`, `Encode`, `ToRGBA` |
| Filters | `GaussianBlur`, `BoxBlur`, `Median`, `Sharpen`, `UnsharpMask` |
| Edges | `Sobel`, `SobelX`, `SobelY`, `SobelMag`, `Prewitt`, `Scharr`, `Laplacian`, `Canny` |
| Morphology | `Erode`, `Dilate`, `Open`, `Close` |
| Geometry | `Resize`, `Rotate90`/`180`/`270`, `Crop`, `FlipHorizontal`, `FlipVertical` |
| Colour | `Grayscale`, `Invert`, `RGBToHSV`, `HSVToRGB`, `OtsuThreshold`, `Threshold`, `AdjustBrightness`, `AdjustContrast` |

## Performance & architectures

The separable hot operations are go-asmgen SIMD-accelerated on amd64 (SSE2),
arm64 (NEON) and s390x (vector); all six 64-bit targets run a portable scalar
core. go-images **beats scikit-image** ~3–12× on box blur and Sobel; the
benchmark page is honest about where it does not.

## Where to go next

- [Roadmap](roadmap.md) — the plan and what ships today.
- [Performance](performance.md) — honest benchmarks versus scikit-image.

Source: [github.com/go-images/images](https://github.com/go-images/images) ·
also the cgo-free image processor behind [go-embedded-ruby](https://github.com/go-embedded-ruby/ruby)'s
`Image` class.
