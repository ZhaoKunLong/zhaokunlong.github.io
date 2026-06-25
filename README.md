# 小满

> 春粒渐满，夏果新熟· — 个人博客，基于 [Hexo](https://hexo.io/) 构建，使用 [Maupassant](https://github.com/tufu9441/maupassant-hexo) 主题。

在线地址：[https://zhaokunlong.github.io](https://zhaokunlong.github.io)

## 环境要求

- [Node.js](https://nodejs.org/) >= 12
- [Git](https://git-scm.com/)

## 快速开始

```bash
# 克隆仓库
git clone git@github.com:ZhaoKunLong/ZhaoKunLong.github.io.git
cd ZhaoKunLong.github.io

# 安装依赖
npm install

# 本地预览（默认 http://localhost:4000）
npm run server
```

## 添加文章

### 方式一：命令行创建（推荐）

```bash
# 创建新文章
hexo new "文章标题"

# 或者使用 npm script
npx hexo new "文章标题"
```

这会在 `source/_posts/` 下生成 `文章标题.md`，模板来自 `scaffolds/post.md`，格式如下：

```markdown
---
title: {{ title }}
date: {{ date }}
tags:
---
```

你只需在 `---` 下面用 Markdown 写正文即可。

### 方式二：手动创建

直接在 `source/_posts/` 目录下新建 `.md` 文件，加入 front-matter 头部：

```markdown
---
title: 文章标题
date: 2026-06-25 16:49:00
tags: [标签1, 标签2]
categories: 分类名
---
正文内容...
```

### 草稿

草稿放在 `source/_drafts/` 目录下，不会发布。发布时使用：

```bash
hexo publish "文章标题"
```

## 常用命令

| 命令 | 说明 |
|------|------|
| `npm run server` | 启动本地预览服务器 |
| `npm run build` | 生成静态文件到 `public/` |
| `npm run clean` | 清除缓存和生成的文件 |
| `npm run deploy` | 部署到 GitHub Pages |

或使用 npx 直接调用：

```bash
npx hexo server       # 本地预览
npx hexo generate     # 生成静态文件
npx hexo clean        # 清理
npx hexo deploy       # 部署
```

## 发布流程

```bash
# 1. 清理旧的生成文件
npm run clean

# 2. 生成静态文件
npm run build

# 3. 部署到 GitHub Pages
npm run deploy
```

部署目标：`git@github.com:ZhaoKunLong/ZhaoKunLong.github.io.git` 的 `master` 分支。

## 目录结构

```
├── _config.yml          # Hexo 配置文件
├── package.json
├── scaffolds/           # 文章模板
│   ├── draft.md         # 草稿模板
│   ├── page.md          # 页面模板
│   └── post.md          # 文章模板
├── source/              # 源文件
│   ├── _drafts/         # 草稿
│   ├── _posts/          # 文章（Markdown）
│   ├── about/           # 关于页面
│   ├── images/          # 图片资源
│   └── tags/            # 标签页面
├── themes/              # 主题
│   └── maupassant/      # Maupassant 主题
└── public/              # 生成的静态文件（gitignore）
```

## 配置说明

主要配置项在 `_config.yml` 中：

- **站点信息**：标题"小满"，作者 Scott，语言 zh-CN
- **URL**：`https://zhaokunlong.github.io`
- **固定链接格式**：`/年/月/日/标题/`
- **主题**：maupassant
- **部署**：Git 方式推送到 GitHub Pages
- **RSS**：启用 Atom Feed，路径为 `/atom.xml`
