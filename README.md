# ComFree Doc Local Build

This repository contains the local Jupyter Book source for the ComFree documentation.

## Book Location

The book source lives in `comfree_warp_doc/`.

Key files:

- `comfree_warp_doc/_config.yml`
- `comfree_warp_doc/_toc.yml`
- `comfree_warp_doc/pages/`

## Local Setup

Use your current Conda environment, or create one if needed.

Example:

```bash
conda activate irislab
pip install -r comfree_warp_doc/requirements.txt
```

If your environment does not already have the Jupyter Book v1 CLI, install it explicitly:

```bash
pip install "jupyter-book<1"
```

## Build The Book

From the repository root:

```bash
jupyter-book build comfree_warp_doc
```

The built HTML will be generated under:

```bash
comfree_warp_doc/_build/html
```

## Preview The Book Locally

After building, serve the generated HTML with a simple local web server:

```bash
python -m http.server 8000 --directory comfree_warp_doc/_build/html
```

Then open:

```text
http://127.0.0.1:8000
```

## Clean The Build

Remove generated output with:

```bash
rm -rf comfree_warp_doc/_build
```

## Update Content

- Put page content in `comfree_warp_doc/pages/`
- Update navigation in `comfree_warp_doc/_toc.yml`
- Update metadata and logo in `comfree_warp_doc/_config.yml`
