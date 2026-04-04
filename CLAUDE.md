# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal blog ("观海听涛" / "指月小筑") built with [Hugo](https://gohugo.io/) using the [LoveIt](https://github.com/lixd/LoveIt) theme. The theme is included as a **git submodule** (not a Hugo module), pointing to a fork at `https://github.com/lixd/LoveIt`. Posts are written in Chinese and cover a wide range of technical topics including cloud-native development, Go, Java, Docker, Kubernetes, gRPC, distributed systems, etcd, Web3D, and LLM/AI.

## Hugo Version

Currently tested with Hugo **v0.100.2 extended** (GitHub Actions uses 0.108.0 extended). Updating Hugo may cause compatibility issues.

## Common Commands

```bash
# Local development server (default: development environment)
hugo server

# Local server in production mode
hugo serve -e production

# Build static site to ./public
hugo --minify

# Initialize/update git submodules (LoveIt theme)
git submodule update --init --recursive
```

## Directory Structure

```
content/posts/          # All blog posts, organized by category subdirectories
  ├── blog/             # General blog topics
  ├── cloudnative/      # Cloud-native development
  ├── distributed/      # Distributed systems
  ├── docker/           # Docker deep-dives
  ├── etcd/             # etcd internals
  ├── git/              # Git usage
  ├── go/               # Go language
  ├── golang/           # Go language (duplicate)
  ├── grpc/             # gRPC & Protobuf
  ├── java/             # Java, Spring, design patterns
  ├── kafka/            # Kafka
  ├── kubernetes/       # Kubernetes
  ├── llm/              # Large language models
  ├── reinforcement-learning/
  ├── web3d/             # Web3D, Babylon.js
  ├── 3d-gaussian/      # 3D Gaussian splatting
  ├── pose-estimation/    # Pose estimation
  ├── claude-code/      # Claude Code source code analysis

themes/LoveIt/          # Git submodule — the Hugo theme (do not modify directly)
archetypes/default.md   # Post template (title, date, draft)
config.toml             # Hugo site configuration
deploy/                 # Server deployment docs (Caddyfile, systemd service)
```

## Content Organization

Posts use the `content/posts/<category>/<slug>.md` path pattern. Categories map to URL paths like `/posts/<category>/<slug>/`.

**Adding a new section (栏目):**
1. Create `content/posts/<new-section>/` directory
2. Create `content/posts/<new-section>/_index.md` with `title` and `description`
3. Add entry in `data/sections.json` with `name`, `title`, `icon`, `description`, and `weight`
4. Start writing posts in that directory

Posts must have frontmatter with `title`, `date`, and `draft` fields. Use `draft: true` while writing.

Posts in a section are filtered by the `Section` (directory path), not by frontmatter `categories`.

## Deployment

Pushes to the `master` branch trigger a GitHub Actions workflow that:
1. Builds the site with `hugo --minify` (Hugo 0.108.0 extended, with Dart Sass)
2. Deploys to GitHub Pages via `peaceiris/actions-gh-pages@v3`
3. Additionally rsyncs to a VPS server (configured via secrets: `DEPLOY_HOST`, `DEPLOY_PORT`, `DEPLOY_USER`, `DEPLOY_KEY`)

Server-side setup uses Caddy 2.5.2 with a systemd service. See `deploy/readme.md` for full VPS deployment steps.

## Theme Submodule Notes

The LoveIt theme is a git submodule, not a Hugo module. Key implications:
- Clone with `git clone --recursive` or run `git submodule update --init --recursive`
- Modifications to the theme should be done in the upstream fork (`github.com/lixd/LoveIt`), not directly in `themes/LoveIt/`
- To add/update/remove the submodule, see `deploy/submodule.md`

## Theme Configuration

Key config sections in `config.toml`:
- `[params.home.techSeries]` — legacy homepage series config (now superseded by `data/sections.yaml`)
- `[params.home.recentPosts]` — recent posts section
- `[markup.highlight]` — code syntax highlighting (`noClasses = false` is required)
- `[params.search]` — uses Lunr search (built-in, no API keys needed)
- Comments are disabled (`[params.page.comment.enable = false]`)

**Homepage栏目导航** reads from `data/sections.json`. Each entry defines a section's `name`, `title`, `icon` (FontAwesome class), `description`, and `weight` (display order).

**Section pages** (`/posts/<section>/`) use `layouts/section/section.html`, which auto-looks up section metadata from `data/sections.json` by section name.

## Git Workflow

1. Write/edit posts in `content/posts/<category>/`
2. Run `hugo server` to preview locally
3. Set `draft: false` in frontmatter when ready to publish
4. Commit and push to `master` — GitHub Actions handles the rest
