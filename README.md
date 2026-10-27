# go-images docs

Documentation site for [go-images](https://github.com/go-images/images), built
with Hugo and the [tannevaled/hextra](https://github.com/tannevaled/hextra)
fork (`themes/hextra`, a submodule pinned at a release tag), and served at
<https://go-images.github.io/docs/>.

## Languages

English is the default and sits at the root of each version. Simplified Chinese
(`/zh-cn/`), Korean (`/ko/`) and Japanese (`/ja/`) are under their code, with
the same file names under `content/<lang>/`; a translated heading keeps the
English one's explicit anchor. The CJK pages rely on
`markup.goldmark.extensions.cjk` (`eastAsianLineBreaks`, `css3draft`, not
`simple`, which glues Korean words together) and on the system CJK font stacks
in `assets/css/custom.css`. A change to an English page is a change to its three
translations too.

## Local preview

```bash
git submodule update --init --recursive   # themes/hextra, pinned at a release tag
hugo server                               # http://localhost:1313
```

Pushing to `main` deploys the `latest` version to the `gh-pages` branch via
GitHub Actions; GitHub Pages serves it under `/docs/`.

BSD-3-Clause © the go-images/docs authors.
