---
title: 在使用 Hexo NexT 主题时如何无侵入式地添加友链页面
date: 2024-12-10 18:04:53
tags: [hexo, next theme, git submodule]
---

在静态博客系统 `Hexo` 和它的第三方主题 `NexT` 被广泛使用多年的情况下，网上添加友链的相关教程并不少。然而由于管理、使用和部署 `Hexo` 博客的方式有所不同，用户可能并不想或不方便侵入式地在**主题目录**中修改或添加文件。本文介绍了如何在通过 `Git` `submodule` 管理和使用 `NexT` 主题的情况下，**无侵入式地添加并定制友链页面**。

<!-- more -->

前置补充说明：

- `Hexo` 的版本为 `7.3.0`
- `NexT` 的版本为 `8.19.1`
- 使用 `Git` `submodule` 追踪和使用官方 `NexT` 主题，无侵入具体指不修改子模块（即主题目录）中的文件

---------------------------------------------------

在使用 `NexT` 主题的 `Hexo` 博客中添加友链有两种方式：

1. 通过配置文件在**首页**侧边栏添加友链
2. 通过自定义页面的方式添加**单独**的友链页面

### 在首页添加友链

使用 `NexT` 主题的 `Hexo` 博客支持通过编辑配置文件的方式在首页添加友链，方法如下：

1. 在 `Hexo` 博客项目的**根目录**下添加 `NexT` 主题的用户配置文件 `_config.next.yml` 文件，用于覆盖主题的默认配置。这样可以避免直接修改主题目录中的默认配置文件。
2. 在 `_config.next.yml` 文件中添加配置：
```yaml
links:
  LINUX DO: https://linux.do
  Moralok: https://www.moralok.com
  Google: https://google.com
```

通过以上两个简单的步骤，就实现了在首页的侧边栏中显示友链，具体效果如下：

<div style="width:20%;margin:auto">{% asset_img "Snipaste_2024-12-11_22-32-25.png" NexT主题侧边栏显示友链 %}</div>

这种方式的优点是简单快捷；缺点是随着友链数量的增加，首页版面的美观度可能有所下降，同时有一点缺少可定制性。

### 添加单独的友链页面

#### 创建友链页面

尽管第三方主题 `NexT` 并没有原生支持生成友链页面，就像生成归档页面和标签页面那样。但我们仍然可以通过自定义 `Hexo` 页面的方式添加并定制一个友链页面。

1. 在 `Hexo` 博客项目的**根目录**下运行以下命令，创建一个新的页面（假设命名为 `links`）：
```bash
hexo new page "links"
```
2. 上述命令会在 `source` 目录下创建一个 `links` 文件夹，并在其中创建一个 `index.md` 文件。（在不同版本的 `Hexo` 中，文件路径有所不同）
3. 通过编辑这个 `index.md` 文件，就可以定制 `url` `/links` 对应的友链页面，比如以无序列表的方式展示友链：
```markdown
- [LINUX DO](https://linux.do)：新的理想型社区
- [Moralok](https://www.moralok.com)：Moralok 的个人博客
- [Google](https://google.com)：谷歌搜索
```

相比于在首页添加友链，这种创建单独友链页面的方式，增加了可定制性，既可以展示更多的信息，也可以添加更多的样式。实际上，大多数使用这种方式的人或多或少都会进一步定制页面的样式，从而增加博客的美观程度。

#### 定制友链页面

因为 `Hexo` 面向的用户群体非常广大，且自身具备非常高的可定制性，各个领域的用户根据自身不同的认知可能采用不同的方式修改定制自己的博客，比如：

1. 直接修改主题目录中的源文件。
2. 通过用户配置文件覆盖主题级别的默认设置。
3. 通过博客配置文件修改博客级别的默认设置。

需要注意的是，无论是 `Hexo` 还是 `NexT`，都允许并**建议通过配置文件覆盖默认设置的方式定制博客**。这样可以减少对源文件的“污染”，便于 `Hexo` 或 `NexT` 的更新升级。但是在现存大多数的教程中，涉及页面和样式的修改往往都需要在主题目录中修改或增加文件。
笔者认同在第三方主题的基础上进一步定制博客难免需要修改主题的源代码，但是在通过 `Git` `submodule` 管理和使用 `NexT` 主题的情况下，暂时完全不想因为添加友链这一事项专门 `fork` 并维护一个私有版本的主题。那么如何在不侵入子模块（即主题目录）的情况下，定制一个稍微美观的友链页面呢？

注意到 `markdown` 不仅支持内嵌 `HTML`，还支持内嵌 `CSS`，修改 `index.md` 文件：

> 其实笔者自认为缺乏对网站的审美能力，这次折腾的主要目的仅仅是为了在添加友链的同时展示对方网站的 icon，因此只要页面不要太丑就好。以下页面和样式均由 Chat GPT 提供。

```html
<style>
.links-container {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 20px;
  margin-top: 40px;
}

.link-item {
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.link-item:hover {
  transform: translateY(-5px);
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2);
}

.link-card {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 20px;
  text-decoration: none;
  color: #333;
  border-radius: 8px;
}

.link-card:hover {
  background-color: #f5f5f5;
}

.link-icon img {
  width: 80px;
  height: 80px;
  margin-bottom: 15px;
  border-radius: 50%;
  object-fit: cover;
}

.link-info h3 {
  font-size: 18px;
  font-weight: bold;
  margin-bottom: 5px;
}

.link-info p {
  font-size: 14px;
  color: #666;
  text-align: center;
  line-height: 1.5;
}

/* 增加响应式设计 */
@media (max-width: 768px) {
  .links-container {
    grid-template-columns: 1fr;
  }
}
</style>

<div class="links-container">
  <div class="link-item">
    <a href="https://linux.do/?source=moralok_com" target="_blank" class="link-card">
      <div class="link-icon">
        <img src="/images/linux_do.png" alt="LINUX DO" />
      </div>
      <div class="link-info">
        <h3>LINUX DO</h3>
        <p>新的理想型社区</p>
      </div>
    </a>
  </div>
</div>
```

实现的具体效果如下：

<div style="width:70%;margin:auto">{% asset_img "Snipaste_2024-12-11_23-25-42.png" 单独的友链页面 %}</div>

#### 更新导航菜单

在添加单独的友链页面后，往往会在首页导航菜单部分，添加友链页面的入口。在 `_config.next.yml` 文件中的 `menu` 节点下，添加 `links`：

```yaml
# 菜单
menu:
  home: / || fa fa-home
  tags: /tags/ || fa fa-tags
  archives: /archives/ || fa fa-archive
  links: /links/ || fa fa-link
```

表示在侧边栏展示 links 按钮，点击时将跳转到 `url` `/links/`，按钮使用 Font Awesome 图标库的链接图标。

> `fa-link`、`fa-home` 等类的标签是 Font Awesome 图标库中的图标类，分别代表“链接”、“首页”图标。Font Awesome 是一个广泛使用的图标库，提供了大量的矢量图标，适用于各种网页和应用开发。`NexT` 支持通过以上方式在侧边栏的菜单中使用 Font Awesome 图标库，具体可用的标签可在[官网](https://fontawesome.com/)中搜索。

##### 菜单的国际化

笔者博客默认使用的语言为中文，注意到首页侧边栏的菜单中，友链的按钮显示为 links，有点格格不入。这是因为 `NexT` 的默认国际化配置文件中，没有 `links` 的翻译。查看 `theme/next/languages/zh-CN.yml` 文件，`menu` 节点的配置如下：

```yaml
menu:
  home: 首页
  archives: 归档
  categories: 分类
  tags: 标签
  about: 关于
  search: 搜索
  schedule: 日程表
  sitemap: 站点地图
  commonweal: 公益 404
```

网上大多数的教程都是建议直接修改该文件，但是和前面提到的页面定制相似，我们并不希望直接修改主题目录中的国际化配置文件。注意到同级目录中有一个 `theme/next/languages/README.md` 文件，里面提供了官方建议的无侵入式地覆盖默认翻译的方法：

1. 在 `Hexo` 博客项目的 `source/_data` 目录下创建一个 `languages.yml` 文件。
2. 添加如下配置覆盖默认翻译：
```yaml
# language
zh-CN:
  # items
  menu:
    # the translation you perfer
    links: 友链
```

> 事实上，如果不考虑国际化问题，直接在 `_config.next.yml` 文件中的 `menu` 节点下，用中文“友链”代替 links 也是可以的，但是正常情况下完全不建议这样子做。

### 总结和感想

本文主要介绍了在使用 `Git` `submodule` 管理和使用 `Hexo` `NexT` 主题的情况下，如何无侵入式地添加友链页面。在这个过程中，同样秉承着“无侵入”的原则，处理了侧边栏菜单的国际化问题。这个原则的核心思想其实是“配置与代码分离”，将应用程序的代码和配置项分离有很多好处，包括但不限于更好的可维护性、可扩展性、安全性和版本控制。在这次情况中，最直接明显的一个好处就是保证了主题更新升级的便捷性。