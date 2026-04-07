# 指月小筑

![Babylon.js 3D Engine](https://assets.babylonjs.com/screenshots/gaussianSplat.jpg)

> 专注于 3D 视觉、WebGL 技术与云原生开发的个人博客
> 
> 本站部分 3D 内容使用 [Babylon.js](https://www.babylonjs.com/) 引擎渲染

![Babylon.js Logo](./static/img/babylonjs-logo.svg)

博客在本地修改及预览完成后 push 到该仓库，由 github action 推送到指定服务器以及 github page 上。





## 切换部署环境

迁移到新服务器具体流程见 [deploy 教程](./deploy/readme.md)



## 切换写作环境

环境配置步骤,写给自己，备忘。

**1）安装 hugo**

到 [github release page](https://github.com/gohugoio/hugo/releases) 下载压缩包，解压并配置环境变量即可。

> 当前使用的是 v0.100.2 extended 版本，更新版本可能出现不兼容情况。

```bash
$ hugo version
hugo v0.100.2-d25cb2943fd94ecf781412aeff9682d5dc62e284+extended windows/amd64 BuildDate=2022-06-08T10:25:57Z VendorInfo=gohugoio
```

**2）clone 此仓库**

```
git@github.com:lixd/lixd.github.io.git
```

使用 submodules 方式，将 themes 合并到了当前仓库，不需要再额外拉取了。

> 具体配置见 .gitmodules 文件

不过 submodule 还是需要手动更新：

```bash
git submodule update --init --recursive
```
如果需要添加、更新或删除 submodule 的话，参考 [submodule 常用命令](./deploy/readme.md)

至此，环境就 ok 了。

## hugo 常用命令

```
# 查看hugo版本号
hugo version
# 本地运行（开发预览）
hugo server
# 本地运行，指定为 production 环境
hugo serve -e production
# 生成 public 文件
hugo
```

## 文章发布流程

Hugo 通过 frontmatter 中的 `draft` 字段控制文章发布状态，所有文章统一存放在 `content/posts/` 下：

1. **新建文章**：使用 `hugo new posts/<section>/<slug>.md` 创建，新文章自动从 `archetypes/default.md` 继承 `draft: true`
2. **写作**：在本地完成文章内容编写，可通过 `hugo server` 实时预览
3. **发布**：将 frontmatter 中的 `draft: true` 改为 `draft: false`
4. **构建**：`hugo` 生成 public 目录并推送到仓库，GitHub Actions 自动部署到 GitHub Pages 和服务器

**草稿不过滤**：`/posts/<section>/` 页面默认显示所有文章（包括草稿），首页栏目卡片中的数字统计已自动排除草稿。发布前请确保 `draft: false`。

**新增专栏**：在 `data/sections.json` 中添加条目，专栏卡片即会出现在首页。`cc-source-review/` 目录下有源码阅读笔记，可在该专栏下新增同名 markdown 文件并链接到 `cc-arch-read/`。