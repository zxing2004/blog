# 个人技术博客（Hugo + GitHub Pages）

一个开箱即用的 Hugo 静态博客，自带主题（无需外部主题/submodule），支持浅色/深色主题、代码高亮、标签分类、文章目录、RSS 订阅，通过 GitHub Actions 自动构建部署到 GitHub Pages。零服务器成本。

## 目录结构

```
blog/
├── .github/workflows/deploy.yml  # 自动部署工作流
├── archetypes/default.md         # 新建文章模板
├── content/
│   ├── about.md                  # 关于页
│   └── posts/                    # 文章目录（Markdown）
├── layouts/                      # 站点自带主题模板
├── static/css/                   # 样式（主样式 + 代码高亮）
├── hugo.toml                     # 站点配置 ⭐ 需修改
└── README.md
```

---

## 一、发布到 GitHub（只需做一次）

> 当前目录已完成 `git init` 并生成了初始提交，你只需依次执行以下命令。

### 1. 修改站点配置

编辑 `hugo.toml`，把以下 `YOUR_USERNAME` 占位符替换为你的 GitHub 用户名：

- `baseURL`：改为 `https://<你的用户名>.github.io/blog/`（若仓库名不是 `blog`，改成对应仓库名）
- `params.social.github`：改为 `https://github.com/<你的用户名>`
- （可选）`title`、`author`、`description` 等站点信息

> 说明：部署工作流会用 Pages 的实际地址覆盖 `baseURL`，所以即使这里暂时不改，线上也能正常生成链接；但建议改好，方便本地预览时链接正确。

### 2. 在 GitHub 上创建仓库

打开 <https://github.com/new>，创建一个**公开（Public）**仓库，名称建议为 `blog`（需与 `baseURL` 中的仓库名一致）。**不要**勾选 README、.gitignore、License 等初始化选项。

### 3. 推送代码

将 `<你的用户名>` 和仓库名替换后执行：

```bash
cd /Users/ricead/WorkBuddy/2026-11-17-17-55-34/blog

# 关联远程仓库
git remote add origin https://github.com/<你的用户名>/blog.git

# 重命名分支为 main 并推送
git branch -M main
git push -u origin main
```

### 4. 开启 GitHub Pages

进入仓库 **Settings → Pages**：

- **Build and deployment → Source**：选择 **GitHub Actions**

推送后，打开仓库的 **Actions** 标签页，等待 `Deploy Hugo site to GitHub Pages` 工作流运行完成（首次约 1–2 分钟）。成功后访问：

```
https://<你的用户名>.github.io/blog/
```

---

## 二、日常写文章

### 新建文章

```bash
hugo new posts/文章标题.md
```

这会基于 `archetypes/default.md` 生成文件，**默认 `draft: true`**。写好内容后把 `draft` 改为 `false` 才会发布。

### 本地预览（需要先安装 Hugo）

```bash
# macOS（如已装 Homebrew）
brew install hugo

# 启动本地服务器，-D 表示同时预览草稿
hugo server -D
```

浏览器打开 <http://localhost:1313/blog/> 即可实时预览。

### 发布

```bash
git add .
git commit -m "docs: 新增文章"
git push
```

每次 push 到 `main` 都会自动触发重新构建与部署。

---

## 三、常见问题

**Q：访问页面是 404？**
确认 Settings → Pages 的 Source 已选为 GitHub Actions，且 Actions 工作流运行成功（绿色对勾）。首次部署可能需要几分钟。

**Q：样式或图片路径不对？**
检查 `baseURL` 是否与仓库名匹配。项目站点（`<用户>.github.io/<仓库>/`）的 baseURL 必须带仓库名和结尾斜杠。

**Q：想用自定义域名？**
在仓库根目录添加 `static/CNAME` 文件，内容为你的域名，并在域名商处配置 CNAME 解析。

**Q：如何更换代码高亮风格？**
修改 `hugo.toml` 中 `markup.highlight.style`，可选 `dracula`、`monokai`、`github-dark` 等（详见 Hugo 文档）。

---

## 四、自定义

- **导航菜单**：`hugo.toml` 中的 `[[menu.main]]`
- **社交链接**：`hugo.toml` 中的 `[params.social]`
- **配色**：`static/css/style.css` 顶部的 CSS 变量
- **首页简介**：`layouts/index.html` 的 `.hero` 区块
- **评论系统**：`hugo.toml` 中 `params.comments` 设为 `true` 后，可在 `single.html` 中接入 Giscus/Disqus

祝写作愉快！
