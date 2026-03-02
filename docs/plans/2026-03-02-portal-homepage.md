# 门户式首页改造实施计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans 来逐步实现此计划。

**Goal:** 将 Hugo 博客首页从传统文章列表改造为门户式结构，优先展示个人简介和技术系列入口，文章下沉到二级页面。

**Architecture:** 通过覆盖 LoveIt 主题的 `layouts/index.html` 创建自定义首页布局，添加新的 partial 组件实现技术系列卡片网格展示，利用 Hugo 的 taxonomy 系统组织 AI/开发/运维三大技术分类。

**Tech Stack:** Hugo 静态站点生成器、LoveIt 主题模板系统、Go Template 模板语言、HTML/CSS 响应式布局。

---

## 技术分类映射

将现有 24 个分类映射到三大技术系列：

| 系列 | 包含分类 | 封面文章 |
|------|----------|----------|
| **AI** | (待确认，当前可能无专门分类) | 待指定 |
| **开发** | go, java, android, docker, kubernetes, grpc, go-micro, protobuf, web, markdown, git, linux, network, distributed, mongodb, mysql, redis, kafka, elasticsearch, etcd, tracing, nginx | 待指定 |
| **运维** | kubernetes, docker, nginx, tekton, cloudnative, linux, etcd, kafka, elasticsearch, tracing | 待指定 |

> 注意：部分分类同时属于开发和运维，将在两个系列卡片中展示。

---

## 实施任务分解

### Task 1: 创建自定义首页布局文件

**Files:**
- Create: `layouts/index.html`
- Create: `layouts/partials/home/tech-series.html`
- Create: `layouts/partials/home/recent-posts.html`

**Step 1: 复制主题首页模板到项目**

```bash
cd /mnt/d/fe/tornado404.github.io/.worktrees/portal-homepage
cp themes/LoveIt/layouts/index.html layouts/index.html
```

**Step 2: 修改 layouts/index.html**

移除文章列表部分，保留 Profile，添加技术系列和最近更新组件调用：

```html
{{- define "content" -}}
    {{- $params := .Scratch.Get "params" -}}
    {{- $profile := .Site.Params.home.profile -}}
    {{- $posts := .Site.Params.home.posts -}}

    <div class="page home"{{ if ne $posts.enable false }} data-home="posts"{{ end }}>
        {{- /* Profile */ -}}
        {{- if ne $profile.enable false -}}
            {{- partial "home/profile.html" . -}}
        {{- end -}}

        {{- /* Content */ -}}
        {{- if .Content -}}
            <div class="single">
                <div class="content" id="content">
                    {{- dict "Content" .Content "Ruby" $params.ruby "Fraction" $params.fraction "Fontawesome" $params.fontawesome | partial "function/content.html" | safeHTML -}}
                </div>
            </div>
        {{- end -}}

        {{- /* Tech Series Grid - 新增 */ -}}
        {{- if ne .Site.Params.home.techSeries.enable false -}}
            {{- partial "home/tech-series.html" . -}}
        {{- end -}}

        {{- /* Recent Posts - 新增，替代原文章列表 */ -}}
        {{- if ne .Site.Params.home.recentPosts.enable false -}}
            {{- partial "home/recent-posts.html" . -}}
        {{- end -}}
    </div>
{{- end -}}
```

**Step 3: 创建技术系列 partial**

创建 `layouts/partials/home/tech-series.html`：

```html
{{- $series := .Site.Params.home.techSeries.series | default slice -}}
<div class="tech-series-section">
    <h2 class="section-title">技术系列</h2>
    <div class="tech-series-grid">
        {{- range $series -}}
            <a href="{{ .url | relLangURL }}" class="tech-series-card">
                <div class="card-icon">
                    {{- with .icon -}}
                        <i class="{{ . }}"></i>
                    {{- else -}}
                        <i class="fas fa-code"></i>
                    {{- end -}}
                </div>
                <h3 class="card-title">{{ .name }}</h3>
                <p class="card-description">{{ .description }}</p>
                <div class="card-meta">
                    <span class="post-count">{{ .count }} 篇文章</span>
                </div>
            </a>
        {{- end -}}
    </div>
</div>
```

**Step 4: 创建最近更新 partial**

创建 `layouts/partials/home/recent-posts.html`：

```html
{{- $recentCount := .Site.Params.home.recentPosts.count | default 5 -}}
{{- $recentPosts := first $recentCount .Site.RegularPages -}}
<div class="recent-posts-section">
    <h2 class="section-title">最近更新</h2>
    <div class="recent-posts-list">
        {{- range $recentPosts -}}
            <article class="recent-post-item">
                <a href="{{ .RelPermalink }}" class="post-title">
                    {{- .Title | emojify -}}
                </a>
                <div class="post-meta">
                    <span class="post-date">{{ .Date.Format "2006-01-02" }}</span>
                    {{- with .Params.categories -}}
                        <span class="post-category">
                            {{- range $i, $cat := . -}}
                                {{- if $i }}, {{ end -}}
                                {{- $cat -}}
                            {{- end -}}
                        </span>
                    {{- end -}}
                </div>
            </article>
        {{- end -}}
    </div>
</div>
```

**Step 5: 运行 Hugo 构建验证**

```bash
cd /mnt/d/fe/tornado404.github.io/.worktrees/portal-homepage
hugo --buildDrafts
```

Expected: 构建成功，无模板错误

**Step 6: 提交**

```bash
git add layouts/index.html layouts/partials/home/tech-series.html layouts/partials/home/recent-posts.html
git commit -m "feat: add custom portal-style homepage layout"
```

---

### Task 2: 添加技术系列样式

**Files:**
- Create: `assets/css/custom.css` (或添加到现有 custom CSS)

**Step 1: 检查现有 custom CSS 位置**

```bash
ls -la assets/ 2>/dev/null || echo "No assets directory"
ls -la static/css/ 2>/dev/null || echo "No static/css directory"
```

**Step 2: 创建自定义样式文件**

创建 `assets/css/custom.css`：

```css
/* Tech Series Section */
.tech-series-section {
    padding: 3rem 1.5rem;
    max-width: 1200px;
    margin: 0 auto;
}

.section-title {
    text-align: center;
    font-size: 2rem;
    margin-bottom: 2rem;
    color: var(--body-text-color);
}

.tech-series-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
    gap: 1.5rem;
    margin-bottom: 3rem;
}

.tech-series-card {
    display: block;
    padding: 1.5rem;
    background: var(--card-background);
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
    transition: transform 0.2s ease, box-shadow 0.2s ease;
    text-decoration: none;
    color: inherit;
}

.tech-series-card:hover {
    transform: translateY(-4px);
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12);
}

.card-icon {
    font-size: 2.5rem;
    margin-bottom: 1rem;
    color: var(--global-link-color);
}

.card-title {
    font-size: 1.25rem;
    margin-bottom: 0.5rem;
    color: var(--body-text-color);
}

.card-description {
    font-size: 0.9rem;
    color: var(--secondary-text-color);
    margin-bottom: 1rem;
    line-height: 1.5;
}

.card-meta {
    display: flex;
    justify-content: space-between;
    font-size: 0.85rem;
    color: var(--secondary-text-color);
}

/* Recent Posts Section */
.recent-posts-section {
    padding: 3rem 1.5rem;
    max-width: 800px;
    margin: 0 auto;
    border-top: 1px solid var(--border-color);
}

.recent-posts-list {
    display: flex;
    flex-direction: column;
    gap: 1rem;
}

.recent-post-item {
    padding: 1rem;
    border-radius: 8px;
    transition: background 0.2s ease;
}

.recent-post-item:hover {
    background: var(--card-background);
}

.post-title {
    font-size: 1.1rem;
    color: var(--body-text-color);
    text-decoration: none;
    display: block;
    margin-bottom: 0.5rem;
}

.post-title:hover {
    color: var(--global-link-color);
}

.post-meta {
    font-size: 0.85rem;
    color: var(--secondary-text-color);
}

.post-date {
    margin-right: 1rem;
}

.post-category {
    font-style: italic;
}

/* Dark mode support */
@media (prefers-color-scheme: dark) {
    .tech-series-card {
        background: #1e1e1e;
    }
    
    .tech-series-card:hover {
        box-shadow: 0 8px 24px rgba(0, 0, 0, 0.3);
    }
}
```

**Step 3: 在 config.toml 中引入自定义 CSS**

修改 `config.toml`，添加：

```toml
[params.page.library.css]
customCSS = "css/custom.css"
```

**Step 4: 提交**

```bash
git add assets/css/custom.css config.toml
git commit -m "feat: add tech series and recent posts styles"
```

---

### Task 3: 配置技术系列数据

**Files:**
- Modify: `config.toml`

**Step 1: 在 config.toml 中添加技术系列配置**

在 `[params.home]` 下添加：

```toml
# 技术系列配置
[params.home.techSeries]
enable = true

[[params.home.techSeries.series]]
name = "开发"
url = "/posts/"
description = "Go、Java、Docker、Kubernetes 等开发技术栈"
icon = "fas fa-laptop-code"
count = 0  # Hugo 构建时自动计算

[[params.home.techSeries.series]]
name = "运维"
url = "/posts/"
description = "Kubernetes、Docker、Nginx、Tekton 等运维工具"
icon = "fas fa-server"
count = 0

[[params.home.techSeries.series]]
name = "AI"
url = "/posts/"
description = "人工智能与机器学习相关技术"
icon = "fas fa-brain"
count = 0

# 最近更新配置
[params.home.recentPosts]
enable = true
count = 5

# 禁用原文章列表
[params.home.posts]
enable = false
paginate = 6
```

**Step 2: 创建 Hugo 自定义 Shortcode 或 Data 文件（可选）**

为了更好的数据管理，可以创建 `data/tech-series.yml`：

```yaml
series:
  - name: 开发
    url: /posts/
    description: "Go、Java、Docker、Kubernetes 等开发技术栈"
    icon: "fas fa-laptop-code"
    categories:
      - go
      - java
      - docker
      - kubernetes
      
  - name: 运维
    url: /posts/
    description: "Kubernetes、Docker、Nginx、Tekton 等运维工具"
    icon: "fas fa-server"
    categories:
      - kubernetes
      - docker
      - nginx
      - tekton
      
  - name: AI
    url: /posts/
    description: "人工智能与机器学习相关技术"
    icon: "fas fa-brain"
    categories: []
```

**Step 3: 提交**

```bash
git add config.toml
git commit -m "feat: add tech series configuration"
```

---

### Task 4: 创建系列分类页面模板

**Files:**
- Create: `layouts/posts/section.html`
- Create: `layouts/_default/taxonomy.html` (可选)

**Step 1: 创建分类页面模板**

创建 `layouts/posts/section.html`：

```html
{{- define "title" -}}
    {{- .Params.Title | default (T .Section) | default .Section | dict "Some" | T "allSome" }} - {{ .Site.Title -}}
{{- end -}}

{{- define "content" -}}
    <div class="page archive">
        {{- /* Title */ -}}
        <h2 class="single-title animate__animated animate__pulse animate__faster">
            {{- .Params.Title | default (T .Section) | default .Section -}}
        </h2>
        
        {{- /* Series Description */ -}}
        {{- with .Params.description -}}
            <p class="series-description">{{ . }}</p>
        {{- end -}}

        {{- /* Paginate */ -}}
        {{- if .Pages -}}
            {{- $pages := .Pages.GroupByDate "2006" -}}
            {{- with .Site.Params.section.paginate | default .Site.Params.paginate -}}
                {{- $pages = $.Paginate $pages . -}}
            {{- else -}}
                {{- $pages = .Paginate $pages -}}
            {{- end -}}
            {{- range $pages.PageGroups -}}
                <h3 class="group-title">{{ .Key }}</h3>
                {{- range .Pages -}}
                    <article class="archive-item">
                        <a href="{{ .RelPermalink }}" class="archive-item-link">
                            {{- .Title | emojify -}}
                        </a>
                        <span class="archive-item-date">
                            {{- $.Site.Params.section.dateFormat | default "01-02" | .Date.Format -}}
                        </span>
                    </article>
                {{- end -}}
            {{- end -}}
            {{- partial "paginator.html" . -}}
        {{- end -}}
    </div>
{{- end -}}
```

**Step 2: 提交**

```bash
git add layouts/posts/section.html
git commit -m "feat: add custom section page template"
```

---

### Task 5: 本地测试与验证

**Files:**
- N/A (测试任务)

**Step 1: 启动 Hugo 开发服务器**

```bash
cd /mnt/d/fe/tornado404.github.io/.worktrees/portal-homepage
hugo server -e production --buildDrafts
```

**Step 2: 验证以下功能**

1. 首页显示 Profile 个人简介
2. 首页显示技术系列卡片网格（3 个系列）
3. 首页显示最近更新列表（5 篇）
4. 原文章列表已移除
5. 点击系列卡片跳转到对应分类页
6. 响应式布局正常
7. 明暗主题切换正常

**Step 3: 记录测试结果**

```bash
# 记录测试结果
echo "测试结果：" > test-results.md
echo "- [ ] Profile 显示正常" >> test-results.md
echo "- [ ] 技术系列卡片显示正常" >> test-results.md
echo "- [ ] 最近更新显示正常" >> test-results.md
echo "- [ ] 原文章列表已移除" >> test-results.md
echo "- [ ] 响应式布局正常" >> test-results.md
echo "- [ ] 明暗主题正常" >> test-results.md
```

---

### Task 6: 最终代码审查与清理

**Files:**
- N/A (审查任务)

**Step 1: 运行 Hugo 构建**

```bash
cd /mnt/d/fe/tornado404.github.io/.worktrees/portal-homepage
hugo --minify
```

Expected: 构建成功，public/ 目录生成

**Step 2: 检查 git 状态**

```bash
git status
git log --oneline
```

**Step 3: 运行最终审查**

检查以下项目：
- [ ] 所有文件已提交
- [ ] 无临时文件
- [ ] 构建成功
- [ ] 无控制台错误

---

## 验收标准

### 功能验收
- [ ] 首页显示个人简介（Profile）
- [ ] 首页显示 3 个技术系列卡片（AI、开发、运维）
- [ ] 首页显示最近 5 篇更新
- [ ] 原文章列表已移除
- [ ] 点击系列卡片跳转到对应分类页
- [ ] 响应式布局在移动端正常

### 代码质量
- [ ] Hugo 构建无错误
- [ ] 无未提交的更改
- [ ] 提交信息清晰
- [ ] 代码符合 DRY 原则

### 视觉验收
- [ ] 卡片网格布局美观
- [ ] 悬停效果正常
- [ ] 明暗主题适配
- [ ] 字体大小和间距合适

---

## 回滚计划

如需回滚：

```bash
cd /mnt/d/fe/tornado404.github.io
git worktree remove .worktrees/portal-homepage
git branch -D feature/portal-homepage
```

恢复原首页：
```bash
rm -rf layouts/index.html layouts/partials/home/tech-series.html layouts/partials/home/recent-posts.html
```
