# Modules

The **6 modules** of `go-images`. Each links to its source and to its generated
API reference.

| Module | What it is | Reference |
| --- | --- | --- |
| [`images`](https://github.com/go-images/images) | the library: filters, edges, morphology, geometry and colour over an RGBA raster, with go-asmgen SIMD kernels | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/images) |
| [`depth`](https://github.com/go-images/depth) | one picture in, two eyes out: depth from cues in the picture itself, and both eyes synthesised | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/depth) |
| [`psd`](https://github.com/go-images/psd) | Photoshop `.psd`/`.psb`, the flattened composite, for `image.Decode` | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/psd) |
| [`png`](https://github.com/go-images/png) | a fork of Go's `image/png` whose `DecodePartial` says what a truncated stream did decode | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/png) |
| [`jpeg`](https://github.com/go-images/jpeg) | a fork of Go's `image/jpeg` that upsamples a four-component JPEG's chroma the way libjpeg does | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/jpeg) |
| [`gif`](https://github.com/go-images/gif) | a fork of Go's `image/gif`, with the same `DecodePartial` as `png` | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/gif) |

## ⛔ Why three forks of Go's own codecs

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

## ⛔ Why a reader registers itself

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
