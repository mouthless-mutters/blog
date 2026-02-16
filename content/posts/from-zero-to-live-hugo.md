+++
title = "From Zero to Live: Building a Static Blog with Hugo and GitHub Pages"
date = "2026-02-16T09:00:00-05:00"
draft = false
+++

I decided to set up a static blog from scratch using Hugo and GitHub Pages. The goal is to use this blog as a place to take notes about things that I'm doing so later I can reproduce things if I need to.

Here are the steps I took

## Step 1: Install Hugo (Extended)

I’m on Windows, so I installed the extended version of Hugo using Chocolatey:

    choco install hugo-extended -y

Then verified:

    hugo version

The extended version matters because many themes require SCSS support.

## Step 2: Create a New Hugo Site

From PowerShell:

    hugo new site blog
    cd blog

This created the base structure:

- content/
- layouts/
- static/
- hugo.toml

This created the site locally.

## Step 3: Initialize Git Locally

Before touching GitHub, I initialized Git locally:

    git init
    git add .
    git commit -m "Initial Hugo site"

This ensured everything was version controlled before introducing a remote.

## Step 4: Add a Theme

I added the Ananke theme as a Git submodule:

    git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke themes/ananke

Then updated hugo.toml:

    theme = "ananke"

Now I could run:

    hugo server -D

And see the site at:

    http://localhost:1313

## Step 5: Create a First Post

Hugo generates posts with TOML front matter by default:

    hugo new posts/hello-world.md

Important lesson:

- draft = true means it will only show locally with -D
- Production builds exclude drafts

Setting:

    draft = false

makes it publishable.

## Step 6: Push to GitHub

I created a public repository named "blog" on GitHub (without a README).

Then connected my local repo:

    git remote add origin https://github.com/<username>/blog.git
    git branch -M main
    git push -u origin main

Now the code lived on GitHub.

## Step 7: Deploy with GitHub Pages + Actions

Instead of manually building, I added a GitHub Actions workflow at:

    .github/workflows/hugo.yml

The workflow:

- Checks out the repo (including submodules)
- Installs Hugo extended
- Runs hugo --minify
- Deploys the public/ folder to GitHub Pages

Once pushed, GitHub automatically built and deployed the site.

## Step 8: Fix the Base URL

The site initially deployed without styling.

The problem: the repository is hosted at:

    https://mouthless-mutters.github.io/blog/

That /blog/ subpath matters.

Fix in hugo.toml:

    baseURL = "https://mouthless-mutters.github.io/blog/"

After committing and redeploying, CSS loaded correctly.

## Step 9: Understand Front Matter Formats

Hugo supports:

- TOML (+++)
- YAML (---)
- JSON

If you start with +++, you must use TOML syntax:

    key = "value"

Strings must be quoted.

Mixing YAML-style colons will break the build.

## Final Result

- Local development with hugo server -D
- Production deploy via GitHub Actions
- Static hosting on GitHub Pages
- Fully version controlled
- Zero server maintenance
