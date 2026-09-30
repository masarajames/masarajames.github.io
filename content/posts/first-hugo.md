---
title: "How I Built This Blog"
date: 2026-02-01
description: "Step by step, how I built this blog with Hugo and GitHub Pages. Installing Hugo, adding a theme as a submodule, previewing locally, and letting GitHub Actions deploy every push. Plus the mistakes I hit."
tags: ["hugo", "github-pages", "blog"]
categories: ["tutorial"]
---

## Why Hugo + GitHub Pages?

Hey 👋  
If you’re reading this, my blog actually works — which still feels a little unreal.

I wanted my first post to document *exactly* how I built this site. Not just the idea, but the **actual commands**, mistakes, and workflow that got everything live.

This is the post I wish I had when I started.

---

I wanted a setup that was:

- 💸 **Free**
- ⚡ **Fast**
- 🧠 **Simple**
- 📝 **Markdown-based**
- 🔄 **Fully version controlled**
- 🗄️ **No database to maintain**

Hugo and GitHub Pages checked every box.

---

## The Tech Stack

- **Hugo (extended)** – static site generator written in Go  
- **Theme** – [nomad-tech](https://github.com/m03315/nomad-tech) (dark, techy, easy to customize)  
- **GitHub Pages** – free static hosting  
- **GitHub Actions** – automatic build & deploy  
- **Custom domain** – [masara.site](https://masara.site)

Once everything is wired up, publishing is literally:

```bash
git push
```

## Step 1: Installing Hugo

First, I installed Hugo locally so I could build and preview the site.

⚠️ Don't just run `sudo apt install hugo`. On Debian/Kali that often gives you an **old, non-extended** version, and themes that use SCSS (like mine) won't build.

Grab the **extended** build from the [Hugo releases page](https://github.com/gohugoio/hugo/releases) instead:

```bash
wget https://github.com/gohugoio/hugo/releases/download/v0.167.0/hugo_extended_0.167.0_linux-amd64.deb
sudo dpkg -i hugo_extended_0.167.0_linux-amd64.deb
```

Verify the installation (look for `+extended`):

```bash
hugo version
```

## Step 2: Creating the Site

Creating a new Hugo site is straightforward:

```bash
hugo new site masarajames.github.io
cd masarajames.github.io
```

Initialize Git (important):

```bash
git init
```

Hugo generates this structure:

```text
content/    # Blog posts
layouts/    # Custom templates
static/     # Images, CSS, JS
themes/     # Themes (submodules)
hugo.toml   # Main configuration
```

## Step 3: Adding a Theme (Git Submodule)

Instead of copying theme files, I added the theme as a submodule:

```bash
git submodule add https://github.com/m03315/nomad-tech.git themes/nomad-tech
```

This keeps the theme clean, separate, and updateable. Anything I want to change goes in my own `layouts/` folder, which overrides the theme without touching it.

## Step 4: Configuring Hugo

Most of the configuration lives in `hugo.toml`:

```toml
baseURL = "https://masara.site"
defaultContentLanguage = "en"
theme = "nomad-tech"

[languages.en]
  languageCode = "en-US"
  title = "This is Masara"

[params]
  author = "Masara"
  subtitle = "Experiments, mistakes, and lessons learned."
  description = "I try things. Sometimes they work"
```

Once this was set, the site finally had styling.

## Step 5: Creating My First Post

Hugo generates posts with front matter automatically:

```bash
hugo new posts/my-first-blog-post.md
```

This creates a file like:

```markdown
---
title: "My First Blog Post"
date: 2026-02-01
draft: true
---

I write everything in Markdown — no CMS, no editor lock-in.
```

## Step 6: Previewing Locally

Before deploying anything, I preview locally:

```bash
hugo server -D
```

Then open:

```text
http://localhost:1313
```

The `-D` flag shows draft posts.

## Step 7: Automating Deployment (GitHub Actions)

I wanted zero manual deployment, so I set up GitHub Actions. This is a trimmed version of my `.github/workflows/hugo.yml`:

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.167.0
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb
      - uses: actions/checkout@v4
        with:
          submodules: recursive
      - id: pages
        uses: actions/configure-pages@v5
      - run: hugo --minify --baseURL "${{ steps.pages.outputs.base_url }}/"
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
    steps:
      - uses: actions/deploy-pages@v4
```

In the repo settings, under **Pages → Source**, pick **GitHub Actions**.

Now every push automatically builds and deploys the site.

## Step 8: Publishing

When the post was ready:

```bash
git add .
git commit -m "Add first blog post"
git push origin main
```

GitHub Actions took over from there.

## How It All Works (Behind the Scenes)

- I write Markdown
- Hugo converts it to HTML
- GitHub Actions builds the site
- GitHub Pages serves it

**Markdown → Hugo → GitHub → Live website**

## Issues I Ran Into (And Fixes)

**Theme not showing**  
→ Forgot `theme = "nomad-tech"`, or cloned without `--recurse-submodules`

**SCSS / "TOCSS" build errors**  
→ Needed Hugo **extended**, not the apt version

**Images broken**  
→ Images must live in `static/` and be linked as `/images/name.png`

**404 after deploy**  
→ Wait a minute and check the GitHub Actions logs
