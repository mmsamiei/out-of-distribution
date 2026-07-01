# Out of Distribution

A static blog built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.

## Prerequisites

- [Hugo Extended](https://gohugo.io/installation/) (v0.151+ recommended; CI uses 0.151.0)
- Git (for the theme submodule)

On macOS:

```bash
brew install hugo
```

## Setup

Clone the repo and initialize the theme submodule:

```bash
git clone <repo-url>
cd out-of-distribution
git submodule update --init --recursive
```

## Build

Generate the static site into `public/`:

```bash
hugo --gc --minify
```

For a production build with a custom base URL:

```bash
hugo --gc --minify --baseURL "https://your-domain.com/"
```

## Run locally

Start the development server with live reload:

```bash
hugo server
```

Open [http://localhost:1313/](http://localhost:1313/).

To include draft posts (`draft: true` in front matter):

```bash
hugo server -D
```

If port 1313 is already in use by another process, pick a different port:

```bash
hugo server --port 1314
```

## Project layout

- `content/posts/` — blog posts (Markdown)
- `hugo.yaml` — site configuration
- `themes/PaperMod/` — theme (git submodule)
- `public/` — build output (generated; do not edit)

## Deployment

Pushes to `main` build and deploy the site to GitHub Pages via `.github/workflows/hugo.yaml`.
