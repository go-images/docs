---
title: "Home"
weight: 10
---

# go-images documentation {#go-images-documentation}

**A pure-Go (no cgo) scikit-image-style image-processing library.** Filters,
edge detectors, morphology, geometry and colour transforms over an RGBA raster —
no OpenCV, no libvips, no ImageMagick.

Image processing in most languages means binding a C library. go-images is a
single portable module that cross-compiles to a static binary, accelerated by
**go-asmgen SIMD** on the separable hot operations — so it **beats scikit-image**
on box blur and Sobel. **100% coverage**, validated against scikit-image.

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

## API surface {#api-surface}

| Area | Functions |
| --- | --- |
| I/O | `Load`, `Save`, `Decode`, `Encode`, `ToRGBA` |
| Filters | `GaussianBlur`, `BoxBlur`, `Median`, `Sharpen`, `UnsharpMask` |
| Edges | `Sobel`, `SobelX`, `SobelY`, `SobelMag`, `Prewitt`, `Scharr`, `Laplacian`, `Canny` |
| Morphology | `Erode`, `Dilate`, `Open`, `Close` |
| Geometry | `Resize`, `Rotate90`/`180`/`270`, `Rotate` (any angle), `Warp`, `Crop`, `FlipHorizontal`, `FlipVertical`, `ExifTranspose` |
| Colour | `Grayscale`, `Invert`, `RGBToHSV`, `HSVToRGB`, `OtsuThreshold`, `Threshold`, `AdjustBrightness`, `AdjustContrast` |
| Scanned pages | `SkewAngle`, `Deskew` |

### Straightening a scanned page {#deskew}

`SkewAngle` says how far a page is tilted and `Deskew` turns it back, in
`Rotate`'s own convention — a page produced by `Rotate(img, a, …)` measures
`a`. The method is **Baird's projection profile** (*The skew angle of printed
documents*, Proc. SPSE 1987): the page is **sheared** rather than rotated,
which keeps the scan-line counts exact where resampling would blur the very
line structure being measured, and the angle taken is the one that maximises
the variance of the marked-pixel count per scan line. `SkewCriterion` selects
Postl's (1986) adjacent-difference score instead; on a page of text the two
agree to the search step.

Judged by tesseract, which knows nothing of this code: a page it reads at
0.968 similarity straight and 0.895 tilted by 7° reads back at **0.968** once
deskewed.

It **refuses** rather than guesses. A page with nothing on it, or with less ink
than a single word, reports `ErrSkewBlank`; a page whose best score sits at the
edge of the search — an overscanned frame with no text lines has no peak at all
— reports `ErrSkewAtLimit` and carries the bound with it, instead of returning
a boundary as though it were a measurement.

⛔ `MaxPixels` bounds the work *and* the measurement. The projection has to
resolve the text lines, so below about two samples per line pitch the angle is
simply wrong — the documentation on the field carries the measured table, for
two page sizes, because the collapse is not predicted by that ratio alone.

## Performance & architectures {#performance-architectures}

The separable hot operations are go-asmgen SIMD-accelerated on amd64 (SSE2),
arm64 (NEON) and s390x (vector); all six 64-bit targets run a portable scalar
core. go-images **beats scikit-image** on box blur (1.8–2.6× single-thread, up
to 10.3× on all cores) and now matches or beats it on Sobel (1.02–1.06×
single-thread, up to 4.8× on all cores); the benchmark page is honest about
where it does not.

## Where to go next {#where-to-go-next}

- [Modules]({{< relref "modules.md" >}}) — the seven modules of this organisation: the library,
  `depth`, the Photoshop reader, the three forks of Go's own codecs, and the
  JPEG 2000 codec.
- [Roadmap]({{< relref "roadmap.md" >}}) — the plan and what ships today.
- [Performance]({{< relref "performance.md" >}}) — honest benchmarks versus scikit-image.

Source: [github.com/go-images/images](https://github.com/go-images/images) ·
also the cgo-free image processor behind [go-embedded-ruby](https://github.com/go-embedded-ruby/ruby)'s
`Image` class.
