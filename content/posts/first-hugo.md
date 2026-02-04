---
title: "How I Built This Blog (Step by Step, With Real Commands)"
date: 2024-01-15
draft: false
tags: ["hugo", "github-pages", "blog"]
categories: ["setup", "tutorial"]
---

Hey 👋  
If you’re reading this, my blog actually works — which still feels a little unreal.

I wanted my first post to document *exactly* how I built this site. Not just the idea, but the **actual commands**, mistakes, and workflow that got everything live.

This is the post I wish I had when I started.

---

## Why Hugo + GitHub Pages?

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

- **Hugo** – static site generator written in Go  
- **Theme** – PaperMod (clean, minimal, dark mode)  
- **GitHub Pages** – free static hosting  
- **GitHub Actions** – automatic build & deploy  

Once everything is wired up, publishing is literally:

```bash
git push
```

## Step 1: Installing Hugo

First, I installed Hugo locally so I could build and preview the site.

```bash
sudo apt install hugo
```

Verify the installation:

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

content/    # Blog posts
layouts/    # Custom templates
static/     # Images, CSS, JS
themes/     # Themes (submodules)
hugo.toml   # Main configuration

## Step 3: Adding a Theme (Git Submodule)

Instead of copying theme files, I added the theme as a submodule:

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

This keeps the theme clean, separate, and updateable.

## Step 4: Configuring Hugo

Most of the configuration lives in hugo.toml:

```bash
baseURL = "https://masarajames.github.io/"
languageCode = "en-us"
title = "James Masara's Digital Garden"
theme = "PaperMod"

[params]
  description = "My corner of the internet"
  defaultTheme = "auto"
  ShowToc = true
  ShowReadingTime = true

  [[params.socialIcons]]
    name = "github"
    url = "https://github.com/masarajames"
```

Once this was set, the site finally had styling.

## Step 5: Creating My First Post

Hugo generates posts with front matter automatically:

hugo new posts/my-first-blog-post.md
This creates a file like:

```bash
---
title: "My First Blog Post"
date: 2024-01-15T10:00:00Z
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

```bash
http://localhost:1313
```

The -D flag shows draft posts.

## Step 7: Automating Deployment (GitHub Actions)

I wanted zero manual deployment, so I set up GitHub Actions.

```bash
.github/workflows/hugo.yml:

name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'

      - run: hugo --minify

      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    steps:
      - uses: actions/deploy-pages@v4
```

Now every push automatically builds and deploys the site.

## Step 8: Publishing
When the post was ready:

```bash 
git add .
git commit -m "Add first blog post"
git push origin main
```

GitHub Actions took over from there.

How It All Works (Behind the Scenes)

- I write Markdown
- Hugo converts it to HTML
- GitHub Actions builds the site
- GitHub Pages serves it
- Markdown → Hugo → GitHub → Live website

## Issues I Ran Into (And Fixes)

Theme not showing
→ Forgot theme = "PaperMod"

Images broken
→ Images must live in static/

404 after deploy
→ Wait a minute and check GitHub Actions logs

If you want next, I can:

- tune this to be **more playful or more technical**
- add **architecture diagrams**
- optimize it for **SEO**
- split it into a **multi-part series**

Just tell me 👍