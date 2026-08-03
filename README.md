# Contribute to this doc

First of all install `mkdocs`:
```shell
pip install mkdocs-material
```

Edit files and test it locally with:
```shell
mkdocs serve
```

Then open the URL printed in the terminal (default `http://127.0.0.1:8000/`).
Edits to `docs/` or `mkdocs.yml` hot-reload automatically.

## Port already in use

If `http://127.0.0.1:8000/` does not load (for example on Windows where
`wslrelay` may hold port 8000), pick another port with `--dev-addr`:
```shell
mkdocs serve --dev-addr 127.0.0.1:8765
```

## Strict build check

Before pushing, verify the build is clean (errors on broken nav links or
missing pages):
```shell
mkdocs build --strict
```

Enjoy!
