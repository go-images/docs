---
title: "Modules"
weight: 20
---

# Modules {#modules}

The **7 modules** of `go-images`. Each links to its source and to its generated
API reference.

| Module | What it is | Reference |
| --- | --- | --- |
| [`images`](https://github.com/go-images/images) | the library: filters, edges, morphology, geometry and colour over an RGBA raster, with go-asmgen SIMD kernels; the pixel operations also live in the codec-free [`images/ops`](#images-ops-pixels-without-codecs) | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/images) |
| [`depth`](https://github.com/go-images/depth) | one picture in, two eyes out: depth from cues in the picture itself, and both eyes synthesised | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/depth) |
| [`psd`](https://github.com/go-images/psd) | Photoshop `.psd`/`.psb`, the flattened composite, for `image.Decode` | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/psd) |
| [`png`](https://github.com/go-images/png) | a fork of Go's `image/png` whose `DecodePartial` says what a truncated stream did decode | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/png) |
| [`jpeg`](https://github.com/go-images/jpeg) | a fork of Go's `image/jpeg` that upsamples a four-component JPEG's chroma the way libjpeg does | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/jpeg) |
| [`gif`](https://github.com/go-images/gif) | a fork of Go's `image/gif`, with the same `DecodePartial` as `png` | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/gif) |
| [`jpeg2000`](https://github.com/go-images/jpeg2000) | a pure-Go JPEG 2000 codec, forked from ajroetker/go-jpeg2000: CMYK read as ink, tiled encodes fixed | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/jpeg2000) |

## images/ops: pixels without codecs {#images-ops-pixels-without-codecs}

⛔ **Status: on main, in the next release.** It is merged but not yet tagged,
so `go get` at the latest tag does not have it.

`github.com/go-images/images/ops` is a subpackage of `images` that holds every
pixel operation: filters, edges, morphology, geometry and colour. The root
package `images` forwards each name under the same name, so existing code keeps
compiling, and keeps the I/O: `Load`, `Save`, `Decode`, `Encode`,
`DecodePartial` and EXIF.

The reason is the link, not the API. Decoders register themselves in `init`, so
importing `images` links **every decoder**, about 1.7 MB of WebAssembly code,
whether the program reads a file or not. `ops` imports no codec.

```go
import "github.com/go-images/images/ops" // pixels only: no decoder is linked

blurred, err := ops.GaussianBlur(src, 3)
```

A program that only processes pixels it already holds should import `ops`. A
program that reads or writes files imports `images`, as before. The widget
toolkit moved its `Material` and `Thumbnail` to `ops` for this reason; see
[Binary size](https://go-widgets.github.io/docs/latest/rendering/toolkit/#binary-size).

## images: a size limit on every decode {#images-size-limit}

An image file states its own size, and a compressed format states a large one in
a few bytes: a 1×1 PNG whose header claims 16000×16000 made the standard decoder
allocate about 1 GB before reading a pixel. Since `images` v0.3.0 every decode
reads the header first and refuses an image above `DecodeOptions.MaxPixels`,
**default 40 000 000 pixels** (a 50-megapixel photo is above it), with an error
wrapping `codec.ErrTooLarge` from go-gfx/gfx. The same bomb now allocates 4.5 KB.

```go
img, err := images.DecodeWith(r, images.DecodeOptions{MaxPixels: 120_000_000})
if errors.Is(err, codec.ErrTooLarge) { /* refuse */ }
```

`Decode`, `DecodeBest`, `DecodePartial` and `Load` apply the default;
`DecodeWith` and `DecodePartialWith` take the options.

## jpeg2000: a fork kept for its fixes {#jpeg2000-a-fork-kept-for-its-fixes}

`jpeg2000` decodes and encodes JPEG 2000 codestreams (`.j2k`) and JP2 files
(`.jp2`) with no C. It is a fork of
[ajroetker/go-jpeg2000](https://github.com/ajroetker/go-jpeg2000), and every
change is listed in its `NOTICE`, as Apache-2.0 requires. go-pdfkit/render
and go-gfx/gfx require it directly, and on 2026-10-18 another 45 fleet
modules required it through them.

- A four-component file whose JP2 header declares **CMYK** decodes to an
  `*image.CMYK`. Read as red, green and blue it lost the black plate: 255
  levels from poppler on every pixel, now at most 2 over 2 440 044 samples.
- Tiled encodes place their code blocks and tiles where the decoder and
  OpenJPEG read them (ITU-T T.800 B.7), and a tile whose blocks all quantise
  away is accepted as the reference accepts it.
- The SIMD dependency is [go-simd/highway](https://go-simd.github.io/docs/latest/repos/highway/),
  because consumers cannot `go mod tidy` against upstream go-highway's releases.
- ⛔ The lossless colour transform and the inverse 5/3 wavelet are **not**
  SIMD calls. On amd64, without `GOEXPERIMENT=simd`, highway takes a generic
  fallback that allocates in every vector operation.
  - The colour transform on one 4K frame took 408.8 ms and 12.45 million
    allocations, against 17.9 ms and 6 for a plain loop. On arm64 the loop also
    beat the NEON path, 14.1 ms against 29.9 ms, because copying the planes in
    and out cost more than the kernel saved.
  - With the wavelet's vertical pass done a whole row at a time and in place,
    a lossless 1920×1080 decode on amd64 went from 638 ms and 14.55 million
    allocations to 269 ms and 93 thousand, the same count as arm64, where the
    decode is not slower (−1.5 %).
- ⛔ **Fuzzed since v0.13.8.** It had no fuzz target, and the first one found
  three panics in seconds, all reachable from `DecodeConfig`. One was fourteen
  bytes that crashed any program rendering a PDF through go-pdfkit/render.
  The headers are now checked as OpenJPEG checks them, and the three inputs
  are regression tests. `Decode` still has **no pixel limit**: call
  `DecodeConfig` first and refuse a picture you cannot afford.

## ⛔ Why three forks of Go's own codecs {#why-three-forks-of-gos-own-codecs}

Not to change what they decode. Each carries its difference in a `NOTICE`, and
Go's own test suite passes unaltered in substance.

`png` and `gif` add `DecodePartial`: the rows that are complete, counting from
the top, and the error that stopped the decode. **A picture still arriving is
not the same thing as a picture that failed**, and `image.Decode` cannot tell a
caller the difference — it is all-or-nothing by design.

`jpeg` changes one thing: a **four-component** JPEG's chroma is upsampled by
reading between the samples, the way libjpeg does, rather than by repeating
each one. That is what made CMYK scans come back softer here than everywhere
else.

## ⛔ Why a reader registers itself {#why-a-reader-registers-itself}

`psd` is imported for its **side effect**:

```go
import _ "github.com/go-images/psd"

m, _, err := image.Decode(f)   // a .psd now decodes like a .png
```

`image.Decode` consults a registry, so a program that adds the import gains
the format without naming it anywhere else — which is how
[go-pdfkit](https://go-pdfkit.github.io/) turns a Photoshop file into a PDF
page with one line and no PSD-specific code.

What it reads is the flattened **composite** Photoshop writes at the end of the
file. Reading layers would mean understanding blend modes, clipping, masks and
adjustment layers — re-implementing a compositor — and the result would still
differ from what Photoshop shows. The composite *is* what Photoshop shows.

⛔ Two of its readings are the ones that break every other PSD reader, so they
have tests of their own: **CMYK is stored inverted** (255 means no ink, and
taking the bytes as written turns a white page black) and **alpha is straight**
(premultiplying makes every semi-transparent pixel too bright — invisibly, on
an opaque file). Judged against macOS's own ImageIO through `sips`, the decoder
is **exact on 2 073 600 pixels** of a real 1920×1080 file.
