---
title: "使用 Hugo Stack 和 GitHub Pages/Actions 搭建博客"
description: 
date: 2024-10-17T13:39:46+08:00
slug: how-to-use-hugo
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags: 
    - Hugo
    - Github Actions
    - Github Pages
categories:
    - 经验分享
---

# 使用 Hugo Stack 和 GitHub Pages/Actions 搭建博客

[Hugo](https://gohugo.io) 是一个用 Go 语言编写的静态网站生成器，以速度快、易用性高和灵活性强而著称。Hugo 通过结合 Markdown 文件和模板，能够快速生成静态网页，适用于博客、文档和个人网站等场景。它支持多种主题和插件，能够轻松部署到 GitHub Pages 等平台，非常适合开发者和内容创作者使用。

Hugo 提供了许多主题，我使用的是 [Hugo Stack](https://github.com/CaiJimmy/hugo-theme-stack)。在部署过程中，如果使用 Hugo 官方文档，即使已经取消 draft 标记，也无法显示新建的文章：[Issue](https://github.com/CaiJimmy/hugo-theme-stack)。可以直接使用 [Hugo Stack 的官方模板](https://github.com/CaiJimmy/hugo-theme-stack-starter) 来解决这个问题。

## 使用模板创建仓库

![](post/how-to-use-hugo/imgs/template.png)
创建的仓库名为username.github.io，username为自己的github用户名。

## 将仓库克隆到本地并安装 Hugo 拓展版

### Linux
```bash
sudo apt install hugo
```

### Mac OS
```bash
brew install hugo
```

### Windows
```bash
choco install hugo-extended
```

或者直接[下载二进制](https://github.com/gohugoio/hugo/releases/latest)

## 新增文章并发布

```bash
hugo new content content/posts/my-first-post.md
```

使用此命令创建的文章有 draft 标记，Hugo 在默认情况下并不会发布带有此标记的文章，需要添加 `-D` 或 `--buildDrafts` 参数。

确认发布后，将 draft 标记改为 `false`，提交 commit 至 GitHub 即可。

## GitHub Actions 自动化发布

Hugo Stack 模板仓库内包含 `deploy.yml`、`update-theme.yml` 两个 GitHub Workflow 文件，分别负责自动化发布和更新主题版本，详见 [Hugo Stack GitHub Workflows](https://github.com/CaiJimmy/hugo-theme-stack-starter/tree/master/.github/workflows)。仓库的更新会触发 GitHub Actions 进行自动化发布。

## GitHub Pages 配置

`deploy.yml` 会将博客发布到 `gh-pages` 分支。

![deploy.yml](post/how-to-use-hugo/imgs/deploy-yml.png)

所以需要在仓库设置中更改 Pages 的分支为 `gh-pages`。

![repo-pages-setting](post/how-to-use-hugo/imgs/repo-pages-setting.png)

如果没有 `gh-pages` 选项，可能 GitHub Actions 还在构建中，等待构建完毕即可。

## 自定义域名

发布完毕后，可以通过 `https://username.github.io` 访问，也可以配置自定义域名。

只需要将你的域名 CNAME 指向 `username.github.io`，并在仓库设置中添加 Custom domain，等待 DNS 验证成功后，即可使用自定义的域名访问刚刚发布的博客。

同时也可以配置强制 https 以增强安全性。

![custom-domain](post/how-to-use-hugo/imgs/custom-domain.png)