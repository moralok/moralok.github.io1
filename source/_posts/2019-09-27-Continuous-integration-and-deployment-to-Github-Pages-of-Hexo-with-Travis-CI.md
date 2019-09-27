---
title: 使用 Travis CI 持续集成 Hexo 并部署到 Github Pages
date: 2019-09-27 16:06:32
categories:
- Web
tags: 
- Hexo
---

## 背景
本文意在记录笔者在使用 [Hexo](https://hexo.io/) 的过程中的遇到问题后的一些思考。

### 建站
[Hexo](https://hexo.io/)! 的使用，请参考 [Hexo 官方文档](https://hexo.io/docs/)。

### 部署
[Hexo](https://hexo.io/) 的部署，请参考 [如何部署 Hexo](https://hexo.io/zh-cn/docs/deployment.html)。文档中介绍了包扩 Git 在内的多种部署方式。

以上的部署，本质上是将 Hexo 生成的静态文件，即 public 文件夹中的内容传输到某一个地方。以 Git 为例，就是将文件上传到 GitHub 或 Gitlab 等代码仓库。


### 访问站点
一般而言，即使是仅仅为了访问一个静态文件构成的站点，我们也需要一台服务器，安装 Apache 或 Nginx 等 Web 服务器应用。但是，Github 提供了一个服务，称为 GitHub Pages。

GitHub Pages 允许你为自己的每一个仓库创建一个 GitHub Page，通过 `<your-username>.github.io/<project-name>` 访问。如果你创建了一个仓库名为 `<your-username>.github.io`，你将可以通过 `<your-username>.github.io` 访问该仓库构建成的 GitHub Page。

需要注意的是，在仓库的设置中启用 GitHub Pages 时，Source 一般可以选取 gh-pages branch、master branch 和 master branch/docs folder 作为发布源。但是仓库名为 `<your-username>.github.io` 的仓库，不能使用 gh-pages branch 作为发布源。详情参考官方文档[配置 GitHub 页面的发布源](https://help.github.com/cn/articles/configuring-a-publishing-source-for-github-pages)。

这样，GitHub 相当于帮我们免费搭建了服务器和 Web 服务器应用。如果我们发布源中的文件没有问题，那么我们就可以通过给定的域名，访问到我们的站点。

### 使用 Travis CI 持续集成 Hexo
和传统的 Git 项目不同的是，Hexo 的源代码仍然在你的本地电脑中。你上传到代码仓库中的，只是一份生成的静态文件。如果你不满足于在一台固定的电脑上使用 Hexo，如果你想要使用 Git 管理你的 Hexo 项目，如果你不想每一台电脑上都安装 Hexo 相关的依赖，那么使用 Travis CI 持续集成 Hexo，是一个不错的选择。

Hexo 文档 - [将 Hexo 部署到 GitHub Pages](https://hexo.io/zh-cn/docs/github-pages) 提供了相关的指导。坑爹的地方在于，文档简洁明了，并没有对于仓库名为 `<your-username>.github.io` 不能使用 gh-pages branch 作为发布源作出说明。

注意：
1. Travis 对于 GitHub Pages 的部署，默认是部署到 gh-pages branch。
2. 如果你的仓库名为 `<your-username>.github.io`，请将源代码分支设置为 dev 或者其他名字，在 .travis.yml 文件中，将 Travis CI 监听的分支设置为源代码的分支，部署分支设置为 master。设置的说明参考文档 - [GitHub Pages Deployment](https://docs.travis-ci.com/user/deployment/pages/)。