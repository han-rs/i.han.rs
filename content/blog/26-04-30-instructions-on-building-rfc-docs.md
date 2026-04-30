+++
title = "Instructions on building RFC documents locally"
date = "2026-04-30"
# updated = ""
description = """
仅供参考.
"""

[taxonomies]
tags = ["RFC"]

# [extra]
# toc = false
+++

## Prerequisites

### Python

1. Python 3.x, here we use Python 3.14.1
1. Python virtual environment (optional but recommended)

Using pyenv:

```bash
pyenv install 3.14.1
pyenv virtualenv 3.14.1 rfc
pyenv activate rfc
```

And then install the required packages:

```bash
pip install sortedcontainers
```

### NodeJS

We make use of `aasvg`, just install it globally:

```bash
npm install -g aasvg
```

### IETF toolchains

We need `kramdown-rfc` and `xml2rfc`:

```sh
apt install kramdown-rfc xml2rfc
```

And then run `make` to install other dependencies. This will try to build the document, but it will fail.

## Building the document

Run the following command to build the HTML:

```sh
cat draft-ietf-tls-rfc8446bis.md | lib/trace.sh -v draft-ietf-tls-rfc8446bis.xml -s preprocessor python3 mk-appendix.py | kramdown-rfc --v3 |  lib/trace.sh -v -q draft-ietf-tls-rfc8446bis.xml -s v2v3 xml2rfc -q --rfc-base-url https://www.rfc-editor.org/rfc/ --id-base-url https://datatracker.ietf.org/doc/html/ --allow-local-file-access --cache=./.refcache --v2v3 /dev/stdin -o /dev/stdout > draft-ietf-tls-rfc8446bis.xml && xml2rfc draft-ietf-tls-rfc8446bis.xml --html
```

This is modified from the CI build script, removing step `lib/trace.sh -v -q draft-ietf-tls-rfc8446bis.xml -s venue python3 lib/add-note.py` since it failed locally for me.

If you want a cleaner way, run `make clean` in advance.
