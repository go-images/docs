# go-images docs

Documentation site for [go-images](https://github.com/go-images/images) — built
with MkDocs Material, versioned with [mike](https://github.com/jimporter/mike),
and served at <https://go-images.github.io/docs/>.

## Local preview

```sh
pip install -r requirements.txt
mkdocs serve
```

Pushing to `main` deploys the `latest` version to the `gh-pages` branch via
GitHub Actions; GitHub Pages serves it under `/docs/`.

BSD-3-Clause © the go-images/docs authors.
