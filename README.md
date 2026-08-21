# 香梨椰奶冰糕的甜品屋

这是博客的 Hugo 源码仓库。网页由 GitHub Actions 自动构建并发布，`public` 是临时构建结果，不需要提交。

## 日常写作

创建新文章：

```powershell
hugo new content post/文章名.md
```

预览网站（包含草稿）：

```powershell
hugo server -D
```

写完后，在文章开头把 `draft = true` 改成 `draft = false`，然后提交并推送到 `main` 分支。

## 发布前检查

```powershell
hugo --gc --minify --cleanDestinationDir
```

构建成功后，推送到 GitHub。GitHub Pages 会自动使用 `.github/workflows/hugo.yml` 构建和发布。

首次使用时，需要在 GitHub 仓库的 **Settings → Pages → Build and deployment → Source** 中选择 **GitHub Actions**。

## 文件职责

- `content/`：文章和独立页面。
- `static/`：图片、头像和下载文件。
- `config/`、`hugo.toml`：博客及主题配置。
- `themes/reimu/`：博客主题。
- `public/`：自动生成的发行版，不直接编辑、不提交。

