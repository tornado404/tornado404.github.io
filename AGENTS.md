# PROJECT KNOWLEDGE BASE

**Generated:** 2026-03-02 11:28:30
**Commit:** 4553a25
**Branch:** master

## OVERVIEW
Hugo-based Chinese technical blog (钟子期's blog). Static site generated from Markdown content, deployed to GitHub Pages + custom server.

## STRUCTURE
```
./
├── content/posts/    # Markdown blog posts (technical tutorials)
├── content/posts/草稿/  # Draft posts (Chinese: 草稿=draft)
├── static/           # Static assets (images)
├── deploy/           # Deployment scripts
├── themes/           # Hugo theme (LoveIt as submodule)
├── config.toml       # Hugo config
└── .github/workflows/  # CI/CD (Hugo build + deploy)
```

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Add new post | content/posts/[topic]/ | Create .md file with frontmatter |
| Edit blog config | config.toml | Hugo configuration |
| Theme settings | themes/ | LoveIt theme submodule |
| CI/CD pipeline | .github/workflows/hugo.yml | GitHub Actions |
| Deployment scripts | deploy/ | Custom server deploy scripts |

## CODE MAP
Static site generator - no runtime code. Key files:

| File | Purpose |
|------|---------|
| config.toml | Hugo config (language, theme, menus, analytics) |
| .github/workflows/hugo.yml | Auto-deploy on push to master |
| content/posts/*/*.md | Blog post markdown files |

## CONVENTIONS
- Posts use Hugo frontmatter (title, date, tags, categories)
- Chinese language: `languageCode = "zh-CN"`
- Submodule for theme: `themes/LoveIt`
- CI triggers on push to master branch only

## ANTI-PATTERNS (THIS PROJECT)
- Do NOT edit built files (public/) - generated automatically
- Do NOT modify theme directly - override via config or assets/
- Never commit large binary files to git

## UNIQUE STYLES
- Topic-organized: content/posts/{topic}/article.md
- Draft posts in `content/posts/草稿/`
- Tags and categories for navigation
- AdSense and analytics enabled

## COMMANDS
```bash
# Local development
hugo server                    # Dev server with hot reload
hugo server -e production      # Production-like build

# Build
hugo              # Generate to public/
hugo --minify     # Minified output

# New post
hugo new posts/my-topic/my-article.md
```

## NOTES
- Theme: LoveIt (Chinese Hugo theme)
- Hugo version: 0.108.0 (extended)
- Deployment: GitHub Pages + custom server (via actions-gh-pages)
- Google Analytics: G-FL23FNP7CM
- AdSense: ca-pub-7693630257897132