---
title: "Hugo + GitHub Pages + GitHub Actions 搭建个人博客"
date: 2026-09-16
lastmod: 2026-09-16
draft: false
description: "使用 Hugo、PaperMod 与 GitHub Actions 构建并自动部署静态技术博客的完整流程。"
categories:
  - DevOps
tags:
  - Hugo
  - GitHub
  - GitHub Actions
  - CI-CD
featured: true
---

> 环境：Debian 13（trixie）
>  本地工具：Hugo Extended
>  主题：PaperMod
>  远程仓库：GitHub
>  自动化：GitHub Actions
>  托管：GitHub Pages

------

## 1. 整体架构

这套方案的核心思路是：

```text
Markdown
   ↓
Hugo
   ↓
HTML / CSS / JS
   ↓
Git
   ↓
GitHub Repository
   ↓
GitHub Actions
   ↓
Hugo Build
   ↓
Artifact
   ↓
GitHub Pages
   ↓
Internet
```

各组件职责：

- **Markdown**：博客内容源文件。
- **Hugo**：静态网站生成器，将 Markdown + 主题 + 配置组合成 HTML/CSS/JS。
- **Git**：本地版本控制。
- **GitHub**：保存博客源码。
- **GitHub Actions**：自动执行构建和部署。
- **GitHub Pages**：托管最终生成的静态网页。

最终目标是：

```bash
vim content/posts/xxx.md
git add .
git commit -m "content: add new article"
git push
```

然后自动完成：

```text
push
  ↓
GitHub Actions
  ↓
hugo --minify
  ↓
生成 public/
  ↓
部署到 GitHub Pages
```

------

## 2. 安装 Hugo

最开始通过 Debian 官方仓库安装：

```bash
sudo apt install hugo
```

查看版本：

```bash
hugo version
```

最初输出：

```text
hugo v0.131.0+extended linux/amd64
```

说明安装的是 Extended 版本。

### Extended 是什么

Hugo 有普通版和 Extended 版。

Extended 版本额外支持部分 SCSS/Sass 等资源处理能力，很多主题会要求 Extended。

因此看到：

```text
+extended
```

是正常且推荐的。

------

## 3. 创建 Hugo Site

创建博客目录：

```bash
cd ~/ComputerStudy/Blog
hugo new site my-blog
cd my-blog
```

目录结构：

```text
.
├── archetypes
│   └── default.md
├── assets
├── content
├── data
├── hugo.toml
├── i18n
├── layouts
├── static
└── themes
```

主要目录含义：

```text
content/      博客 Markdown 内容
themes/       Hugo 主题
static/       图片等静态文件
layouts/      页面模板
assets/       CSS/JS 等资源
hugo.toml     Hugo 主配置文件
```

------

## 4. 安装 PaperMod 主题

初始化 Git：

```bash
git init
```

使用 Git Submodule 安装 PaperMod：

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

此时目录类似：

```text
themes/
└── PaperMod/
```

### 为什么使用 Git Submodule

博客本身是一个 Git 仓库，而 PaperMod 自己也是一个 Git 仓库。

所以结构本质上是：

```text
my-blog Git Repository
│
├── content/
├── hugo.toml
└── themes/
    └── PaperMod   ← 另一个 Git 仓库
```

Submodule 的作用是：

- 主仓库不直接保存 PaperMod 的全部历史。
- 主仓库只记录 PaperMod 仓库地址和指定 commit。
- CI/CD 拉取代码时，需要同时拉取 Submodule。

查看：

```bash
git submodule status
```

------

## 5. Hugo 版本兼容问题

最初启动：

```bash
hugo server -D
```

出现错误：

```text
WARN Module "PaperMod" is not compatible with this Hugo version:
Min 0.146.0
```

以及：

```text
ERROR => hugo v0.146.0 or greater is required for hugo-PaperMod to build
```

当时本机版本：

```text
Hugo 0.131.0
```

PaperMod 要求：

```text
Hugo >= 0.146.0
```

### 原因

Debian Stable/正式仓库中的软件版本通常更注重稳定性，不一定跟随上游最新版。

因此：

```text
Debian 仓库版本
≠
Hugo 上游最新版
```

### 解决方法

卸载 Debian 仓库版本：

```bash
sudo apt remove hugo
```

从 Hugo 官方 GitHub Releases 下载 Extended `.deb`：

```text
hugo_extended_xxx_linux-amd64.deb
```

安装：

```bash
sudo apt install ./hugo_extended_xxx_linux-amd64.deb
```

再次检查：

```bash
hugo version
```

确认：

```text
Hugo >= 0.146.0
+extended
```

升级后 PaperMod 正常加载。

------

## 6. 配置 hugo.toml

最开始配置：

```toml
baseURL = 'https://example.org/'
languageCode = 'en-us'
title = 'My New Hugo Site'
theme = 'PaperMod'
```

因为 GitHub 仓库实际为：

```text
git@github.com:xiake707/my-blog.git
```

所以 GitHub Pages 地址为：

```text
https://xiake707.github.io/my-blog/
```

最终应配置：

```toml
baseURL = 'https://xiake707.github.io/my-blog/'
languageCode = 'zh-cn'
title = 'My New Hugo Site'
theme = 'PaperMod'
```

### 一个实际踩过的坑

曾错误写成：

```toml
baseURL = '[https://xiake707.github.io/my-blog/](https://xiake707.github.io/my-blog/)'
```

这是 Markdown 链接语法，不是合法的 URL 配置。

正确写法必须是纯 URL：

```toml
baseURL = 'https://xiake707.github.io/my-blog/'
```

------

## 7. 创建第一篇文章

创建：

```bash
hugo new content posts/hello-hugo.md
```

编辑：

```bash
vim content/posts/hello-hugo.md
```

示例：

```markdown
+++
date = '2026-09-16T...'
draft = true
title = 'Hello Hugo'
+++

# Hello Hugo

这是我的第一篇 Hugo 博客。

## 我的博客主要记录

- Linux
- Go
- Docker
- Kubernetes
- DevOps
- SRE
```

------

## 8. 本地启动 Hugo

开发模式：

```bash
hugo server -D
```

其中：

```text
-D
=
--buildDrafts
```

表示构建草稿。

本地访问：

```text
http://localhost:1313/
```

Hugo 支持自动检测文件变化。

例如：

```bash
vim content/posts/hello-hugo.md
```

保存后 Hugo 会自动重建页面。

本地开发链路：

```text
Markdown
   ↓
Hugo
   ↓
PaperMod
   ↓
HTML / CSS / JS
   ↓
localhost:1313
```

------

## 9. 配置 .gitignore

创建：

```bash
vim .gitignore
```

内容：

```gitignore
# Hugo generated files
/public/
/resources/_gen/

# Hugo build lock
.hugo_build.lock

# OS / Editor
.DS_Store
*.swp
*.swo
```

### 为什么忽略 public/

`public/` 是 Hugo 的构建产物。

它和程序编译出来的二进制文件类似。

源码：

```text
content/
hugo.toml
themes/
```

构建产物：

```text
public/
```

因为后续 GitHub Actions 会自动执行 Hugo Build，所以不需要把 `public/` 提交到 Git 仓库。

------

## 10. 第一次 Git Commit

查看状态：

```bash
git status
```

添加：

```bash
git add .
```

再次检查：

```bash
git status
```

提交：

```bash
git commit -m "feat: initialize Hugo blog with PaperMod"
```

查看：

```bash
git log --oneline
```

------

## 11. 创建 GitHub Repository

GitHub 创建空仓库：

```text
my-blog
```

建议：

```text
Public
```

不要提前创建：

```text
README
.gitignore
License
```

否则 GitHub 会先生成一个远程 commit，容易造成：

```text
本地 Git 历史
+
远程 Git 历史
=
历史分叉
```

------

## 12. 连接远程 GitHub

添加远程仓库：

```bash
git remote add origin git@github.com:xiake707/my-blog.git
```

查看：

```bash
git remote -v
```

确认：

```bash
git remote get-url origin
```

输出：

```text
git@github.com:xiake707/my-blog.git
```

设置主分支：

```bash
git branch -M main
```

第一次推送：

```bash
git push -u origin main
```

此后直接：

```bash
git push
```

即可。

------

## 13. GitHub Pages 设置

进入 GitHub Repository：

```text
Settings
→ Pages
```

在：

```text
Build and deployment
```

中将：

```text
Source
```

设置为：

```text
GitHub Actions
```

### 注意

GitHub 账户里的：

```text
Pages → Add a verified domain
```

是用于绑定自定义域名的，不是当前需要配置的 Pages 部署入口。

------

# 14. 配置 GitHub Actions

创建目录：

```bash
mkdir -p .github/workflows
```

创建：

```bash
vim .github/workflows/hugo.yml
```

Workflow：

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v6
        with:
          submodules: recursive

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true

      - name: Setup Pages
        uses: actions/configure-pages@v5

      - name: Build Hugo site
        run: hugo --minify

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v4
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

------

# 15. GitHub Actions 核心语法理解

## on

```yaml
on:
  push:
    branches:
      - main
```

含义：

```text
main 分支发生 push
        ↓
自动触发 Workflow
```

同时：

```yaml
workflow_dispatch:
```

允许在 GitHub Actions 页面手动执行。

------

## permissions

```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

含义：

```text
contents: read
→ 读取仓库源码

pages: write
→ 部署 GitHub Pages

id-token: write
→ Pages 部署身份认证
```

------

## jobs

该流水线有两个 Job：

```text
build
deploy
```

关系：

```text
push
 ↓
build
 ↓
Artifact
 ↓
deploy
 ↓
GitHub Pages
```

------

## runs-on

```yaml
runs-on: ubuntu-latest
```

表示 Job 运行在 GitHub 提供的 Ubuntu Runner 上。

可以把 Runner 理解为：

```text
临时 CI 执行环境
```

------

## steps

一个 Job 内部包含多个 Step。

例如：

```text
Checkout
↓
Setup Hugo
↓
Setup Pages
↓
Build
↓
Upload Artifact
```

------

## uses

例如：

```yaml
uses: actions/checkout@v6
```

表示调用一个现成的 GitHub Action。

类似于流水线里的可复用插件。

------

## Submodule

因为 PaperMod 使用 Git Submodule，所以：

```yaml
with:
  submodules: recursive
```

非常重要。

否则 GitHub Actions 只会拿到主仓库，而 PaperMod 目录可能只是一个 Git 指针，Hugo 构建时会找不到主题。

------

## Build

```yaml
run: hugo --minify
```

相当于 GitHub Runner 自动帮我们执行：

```bash
hugo --minify
```

生成：

```text
public/
```

------

## Artifact

```yaml
uses: actions/upload-pages-artifact@v4
with:
  path: ./public
```

将构建出来的 `public/` 上传为 GitHub Pages Artifact。

结构：

```text
Hugo Build
   ↓
public/
   ↓
Artifact
   ↓
Deploy
```

------

## needs

```yaml
needs: build
```

表示：

```text
deploy
必须等待
build 成功
```

如果：

```text
build ❌
```

则：

```text
deploy 不会执行
```

------

# 16. 提交 CI/CD 配置

执行：

```bash
git add .
git commit -m "ci: deploy Hugo blog with GitHub Actions"
git push
```

GitHub Actions 自动触发。

最终成功看到：

```text
build ✅
deploy ✅
```

网站地址：

```text
https://xiake707.github.io/my-blog/
```

------

# 17. 第一次线上页面没有文章的问题

虽然：

```text
build ✅
deploy ✅
```

但是访问网页时只有标题，没有博客内容。

原因是：

```toml
draft = true
```

### 为什么本地能看到

本地运行的是：

```bash
hugo server -D
```

`-D` 会构建草稿。

所以：

```text
draft = true
+
hugo server -D
=
本地能看到
```

### 为什么线上看不到

GitHub Actions 使用：

```bash
hugo --minify
```

没有 `-D`。

因此：

```text
draft = true
+
hugo --minify
=
草稿被忽略
```

最终 GitHub Pages 没有该文章。

### 解决

将：

```toml
draft = true
```

修改为：

```toml
draft = false
```

然后先用生产模式验证：

```bash
hugo server
```

如果不加 `-D` 仍能看到文章，说明正式构建没有问题。

提交：

```bash
git add content/posts/hello-hugo.md
git commit -m "content: publish first blog post"
git push
```

GitHub Actions 再次自动：

```text
build
↓
deploy
↓
网站更新
```

------

# 18. 最终完整 CI/CD 流程

以后发布文章的标准流程：

```bash
vim content/posts/k8s-deployment.md
```

写文章后：

```bash
git add .
git commit -m "content: add kubernetes deployment article"
git push
```

后续自动：

```text
Developer
   │
   │ git push
   ▼
GitHub Repository
   │
   │ push event
   ▼
GitHub Actions
   │
   ├── Checkout Repository
   ├── Checkout PaperMod Submodule
   ├── Setup Hugo
   ├── hugo --minify
   └── Upload Artifact
               │
               ▼
             Deploy
               │
               ▼
         GitHub Pages
               │
               ▼
          Internet
```

------

# 19. 这套方案对应的 DevOps 概念

整个博客部署实际上是一个小型 DevOps 项目。

## Source Code

```text
Markdown
hugo.toml
PaperMod reference
GitHub Actions YAML
```

------

## Source Control

```text
Git
GitHub
```

------

## CI

```text
Push
↓
Checkout
↓
Dependencies
↓
Hugo Build
↓
Artifact
```

------

## CD

```text
Artifact
↓
Deploy
↓
GitHub Pages
```

------

## Pipeline as Code

```text
.github/workflows/hugo.yml
```

对应其他 CI/CD 平台：

```text
GitHub Actions
→ .github/workflows/*.yml

GitLab CI
→ .gitlab-ci.yml

Jenkins
→ Jenkinsfile
```

------

# 20. 常见问题总结

## 1. PaperMod 报 Hugo 版本过低

错误：

```text
Min 0.146.0
```

解决：

```text
升级 Hugo Extended
```

不要修改主题源码来绕过版本检查。

------

## 2. found no layout file

可能原因：

```text
主题没有安装
主题没有启用
主题版本不兼容
```

检查：

```bash
cat hugo.toml
ls themes/PaperMod
git submodule status
```

------

## 3. 本地有内容，线上没内容

重点检查：

```text
draft = true
```

本地：

```bash
hugo server -D
```

会显示草稿。

生产：

```bash
hugo
```

不会发布草稿。

------

## 4. PaperMod 在 GitHub Actions 中找不到

检查 Workflow 是否包含：

```yaml
with:
  submodules: recursive
```

------

## 5. GitHub Pages 地址样式丢失

重点检查：

```toml
baseURL
```

Project Pages 通常类似：

```text
https://username.github.io/repository/
```

所以：

```toml
baseURL = 'https://username.github.io/repository/'
```

------

## 6. Actions 出现黄色 Node.js Warning

如果：

```text
build ✅
deploy ✅
```

说明当前流水线仍然成功。

黄色 Warning 不等于 Build Failure。

需要关注的是红色 Error 和 Job Failed。

------

# 21. 最终理解

Hugo 并不是服务器。

它只是：

```text
Static Site Generator
```

负责：

```text
Markdown
+
Theme
+
Config
↓
HTML / CSS / JS
```

GitHub Pages 才是最终提供网站访问能力的托管平台。

因此电脑关机以后：

```text
本地电脑 OFF
```

不会影响：

```text
GitHub Pages
```

上的博客访问。

最终职责划分：

```text
Hugo
= Build

GitHub
= Source Repository

GitHub Actions
= CI/CD

GitHub Pages
= Hosting
```

------

# 22. 一句话总结

整个 Hugo 博客项目最终实现了：

```text
Markdown
→ Git
→ GitHub
→ GitHub Actions
→ Hugo Build
→ Artifact
→ GitHub Pages
→ 自动发布
```

以后写完文章只需要：

```bash
git add .
git commit
git push
```

即可自动完成博客构建与上线。
