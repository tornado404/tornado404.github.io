# content/posts - Blog Content

**Parent:** ../AGENTS.md

## OVERVIEW
Technical blog posts organized by topic (30+ categories). Markdown files with Hugo frontmatter.

## STRUCTURE
```
content/posts/
├── kubernetes/      # Kubernetes articles
├── go/              # Golang articles
├── docker/          # Docker articles
├── java/            # Java articles
├── mysql/           # MySQL articles
├── redis/           # Redis articles
├── linux/           # Linux articles
├── 草稿/            # Draft posts (Chinese: draft)
└── template.md      # Post template
```

## POST CONVENTIONS

### Frontmatter (required)
```yaml
title: "Post Title"
description: "Short description"
date: 2024-01-01
draft: true           # true = don't build
categories: ["Go"]
tags: ["Go"]
```

### Summary Separator
```markdown
Summary text here

<!--more-->

Rest of article...
```

### Image Paths
```markdown
![image.png](../../../img/topic/category/image.png)
```
- Images in `static/img/`
- Path: `../../..` = 3 levels up from `content/posts/topic/article.md`

## WHERE TO LOOK
| Task | Location |
|------|----------|
| New post | content/posts/[topic]/ |
| Post template | content/posts/template.md |
| Draft posts | content/posts/草稿/ |

## ANTI-PATTERNS
- Do NOT use absolute URLs for images
- Do NOT forget `draft: true` for work-in-progress
- Do NOT put images outside `static/img/`