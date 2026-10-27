---
title: "モジュール"
weight: 20
---

# モジュール {#modules}

`go-images` の **7 つのモジュール**です。それぞれ、ソースと自動生成された API リファレンスへのリンクを備えています。

| モジュール | 概要 | リファレンス |
| --- | --- | --- |
| [`images`](https://github.com/go-images/images) | ライブラリ本体です。RGBA ラスタに対するフィルタ、エッジ、モルフォロジー、幾何変換、色変換を、go-asmgen SIMD カーネルとともに提供します。ピクセル演算は、コーデックを含まない [`images/ops`](#images-ops-pixels-without-codecs) にもあります | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/images) |
| [`depth`](https://github.com/go-images/depth) | 1 枚の画像から両目用の 2 枚を生成します。画像そのものの手がかりから深度を推定し、両目の像を合成します | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/depth) |
| [`psd`](https://github.com/go-images/psd) | Photoshop の `.psd`/`.psb` のフラット化された合成画像を、`image.Decode` で読み込めるようにします | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/psd) |
| [`png`](https://github.com/go-images/png) | Go の `image/png` のフォークです。`DecodePartial` が、途中で切れたストリームのうちデコードできた範囲を返します | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/png) |
| [`jpeg`](https://github.com/go-images/jpeg) | Go の `image/jpeg` のフォークです。4 コンポーネント JPEG のクロマを、libjpeg と同じ方法でアップサンプリングします | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/jpeg) |
| [`gif`](https://github.com/go-images/gif) | Go の `image/gif` のフォークで、`png` と同じ `DecodePartial` を備えます | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/gif) |
| [`jpeg2000`](https://github.com/go-images/jpeg2000) | ajroetker/go-jpeg2000 をフォークした、純粋な Go の JPEG 2000 コーデックです。CMYK をインクとして読み込み、タイル分割エンコードを修正しています | [pkg.go.dev](https://pkg.go.dev/github.com/go-images/jpeg2000) |

## images/ops: コーデックなしのピクセル演算 {#images-ops-pixels-without-codecs}

⛔ **状況: main にマージ済みで、次のリリースに含まれます。** マージ済みですがまだタグ付けされていないため、最新のタグで `go get` しても含まれていません。

`github.com/go-images/images/ops` は `images` のサブパッケージで、フィルタ、エッジ、モルフォロジー、幾何変換、色変換といったすべてのピクセル演算を収めています。ルートパッケージ `images` は各名前を同じ名前のまま転送するため、既存のコードはそのままコンパイルできます。入出力である `Load`、`Save`、`Decode`、`Encode`、`DecodePartial` と EXIF は、引き続きルートパッケージが担います。

理由は API ではなくリンクにあります。デコーダは `init` で自身を登録するため、`images` をインポートすると、ファイルを読むかどうかにかかわらず**すべてのデコーダ**がリンクされ、WebAssembly のコードで約 1.7 MB になります。`ops` はコーデックをインポートしません。

```go
import "github.com/go-images/images/ops" // pixels only: no decoder is linked

blurred, err := ops.GaussianBlur(src, 3)
```

すでに保持しているピクセルを処理するだけのプログラムは、`ops` をインポートしてください。ファイルを読み書きするプログラムは、これまでどおり `images` をインポートします。ウィジェットツールキットは、この理由から `Material` と `Thumbnail` を `ops` へ移しました。詳しくは[バイナリサイズ](https://go-widgets.github.io/docs/latest/rendering/toolkit/#binary-size)を参照してください。

## jpeg2000: 修正のために維持しているフォーク {#jpeg2000-a-fork-kept-for-its-fixes}

`jpeg2000` は、JPEG 2000 のコードストリーム（`.j2k`）と JP2 ファイル（`.jp2`）を、C を使わずにデコード・エンコードします。[ajroetker/go-jpeg2000](https://github.com/ajroetker/go-jpeg2000) のフォークで、Apache-2.0 の要件に従い、すべての変更を `NOTICE` に記載しています。go-pdfkit/render と go-gfx/gfx は直接これを require しており、2026-10-18 には、さらに 45 のフリートモジュールがそれらを経由して require していました。

- JP2 ヘッダーが **CMYK** と宣言している 4 コンポーネントのファイルは、`*image.CMYK` にデコードされます。赤・緑・青として読むと黒の版が失われ、poppler との差はすべてのピクセルで 255 レベルでしたが、現在は 2 440 044 サンプルに対して最大 2 です。
- タイル分割エンコードでは、コードブロックとタイルを、デコーダと OpenJPEG が読む位置（ITU-T T.800 B.7）に配置します。また、ブロックがすべて量子化で消えるタイルも、リファレンス実装と同様に受理されます。
- SIMD の依存先は [go-simd/highway](https://go-simd.github.io/docs/latest/repos/highway/) です。利用側が、上流の go-highway のリリースに対して `go mod tidy` を実行できないためです。
- ⛔ 可逆色変換と逆 5/3 ウェーブレットは、SIMD 呼び出しでは**ありません**。amd64 で `GOEXPERIMENT=simd` を指定しない場合、highway は汎用のフォールバックを使い、ベクトル演算のたびにメモリを割り当てます。
  - 4K フレーム 1 枚の色変換は、408.8 ms と 12.45 million 回のメモリ割り当てを要しましたが、単純なループでは 17.9 ms と 6 回でした。arm64 でも、プレーンの出し入れのコピーがカーネルによる節約を上回ったため、ループの方が NEON 経路より速く、14.1 ms 対 29.9 ms でした。
  - ウェーブレットの垂直パスを行全体について、その場で処理するようにしたところ、amd64 での可逆 1920×1080 のデコードは、638 ms と 14.55 million 回の割り当てから、269 ms と 93 thousand 回になりました。これは arm64 と同じ回数で、arm64 のデコードは遅くなっていません（−1.5 %）。

## ⛔ Go 標準コーデックのフォークが 3 つある理由 {#why-three-forks-of-gos-own-codecs}

デコード結果を変えるためではありません。それぞれの違いは `NOTICE` に記載しており、Go 自身のテストスイートは実質的に変更なしで通ります。

`png` と `gif` は `DecodePartial` を追加します。これは、上から数えて完成している行と、デコードを止めたエラーを返します。**まだ届いている途中の画像と、失敗した画像は同じではありません**が、`image.Decode` は設計上すべてか無かのため、その違いを呼び出し側に伝えられません。

`jpeg` が変えるのは 1 点だけです。**4 コンポーネント**の JPEG のクロマを、各サンプルを繰り返すのではなく、libjpeg と同様にサンプルの間を読んでアップサンプリングします。これが、ここでは CMYK のスキャン画像が他より柔らかく仕上がっていた原因でした。

## ⛔ リーダーが自分で登録する理由 {#why-a-reader-registers-itself}

`psd` は**副作用**のためにインポートします。

```go
import _ "github.com/go-images/psd"

m, _, err := image.Decode(f)   // a .psd now decodes like a .png
```

`image.Decode` はレジストリを参照します。そのため、このインポートを加えたプログラムは、他の場所で名前を挙げなくてもそのフォーマットを扱えるようになります。[go-pdfkit](https://go-pdfkit.github.io/) が、PSD 専用のコードなしに、1 行で Photoshop ファイルを PDF のページにできるのはこのためです。

読み込むのは、Photoshop がファイルの末尾に書き込む、フラット化された**合成画像**です。レイヤーを読むには、ブレンドモード、クリッピング、マスク、調整レイヤーを理解する必要があり、コンポジタを再実装することになります。それでも結果は Photoshop の表示とは異なります。合成画像こそが Photoshop の表示そのものです。

⛔ 他のあらゆる PSD リーダーが失敗する読み方が 2 つあり、そのためこのデコーダには専用のテストがあります。**CMYK は反転して格納されており**（255 がインクなしを意味し、書かれたとおりにバイトを扱うと白いページが黒くなります）、**アルファはストレートです**（乗算済みにすると、半透明のピクセルがすべて明るくなりすぎます。不透明なファイルでは目に見えません）。macOS 自身の ImageIO を `sips` 経由で基準にして判定したところ、実際の 1920×1080 のファイルの **2 073 600 ピクセルで完全に一致**しました。
