# 技术博客

基于 Hugo 构建的个人博客，记录技术学习、实践与思考。

## 本地预览

安装 [Hugo](https://gohugo.io/)，在项目目录运行：

```bash
hugo server -D
```

然后访问 <http://localhost:1313/blog/>。

## 写文章

```bash
hugo new posts/文章标题.md
```

文章默认是草稿，发布前将 front matter 中的 `draft` 改为 `false`。

## 部署

推送到 `main` 分支后，GitHub Actions 会自动构建并部署到 GitHub Pages。
