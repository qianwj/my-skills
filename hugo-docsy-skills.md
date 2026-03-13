# Hugo + Docsy 文档站点技能整理

## 1. 项目结构与核心机制

**典型目录结构（以 grpc.io 为例）：**

```
project/
├── content/en/          # 所有 Markdown 内容，目录即 URL 路径
│   ├── blog/
│   ├── docs/
│   ├── about/
│   └── showcase/
├── layouts/             # 覆盖主题的自定义模板
│   ├── _default/
│   ├── partials/        # 可复用局部模板
│   └── shortcodes/      # 自定义短代码
├── themes/docsy/        # Docsy 主题（git submodule）
├── static/              # 静态资源，直接映射到根 URL
├── data/                # Hugo 数据文件
├── config.yaml          # 站点配置（或 config.toml）
└── netlify.toml         # Netlify 部署配置
```

**内容与 URL 映射规则：**
- `content/en/docs/foo/bar.md` → `https://site.com/docs/foo/bar/`
- `content/en/docs/foo/_index.md` → `https://site.com/docs/foo/`（Section 首页）
- `_index.md` 用于 Section 首页，`index.md` 用于 Leaf Bundle（含本地资源的页面）

---

## 2. Docsy 主题继承机制

**主题加载顺序（Hugo 模板查找规则）：**
1. 先查找 `layouts/` 目录（项目级）
2. 再查找 `themes/docsy/layouts/`（主题级）

这意味着在 `layouts/` 中创建同名文件即可覆盖主题模板，无需修改 submodule。

**Docsy 作为 git submodule 管理：**
```bash
# 初始化（安装时自动执行）
git submodule update --init --depth 1

# 更新到最新版本
git submodule update --remote --depth 1
```

**Docsy 自身依赖 Node.js：**
```bash
cd themes/docsy && npm install
```
在根级 `package.json` 的 `prepare` 脚本中通常会自动执行这一步。

---

## 3. 内容前置参数（Front Matter）

**常用字段（YAML 格式）：**

```yaml
---
title: 页面标题（浏览器标题栏 + H1）
linkTitle: 导航栏/侧边栏显示名（比 title 更短时使用）
description: SEO 摘要
weight: 10          # 同级页面排序权重，数字越小越靠前
draft: true         # 草稿，仅 dev/preview 构建时可见
menu:
  main: {weight: 2} # 将此页加入顶部导航栏，weight 控制位置
no_list: true       # Section 首页不自动生成子页面列表
spelling: cSpell:ignore someword  # 单页面拼写检查忽略词
---
```

**博客文章额外字段：**
```yaml
---
date: 2024-01-15
author:
  name: Author Name
  link: https://github.com/author
---
```

---

## 4. Hugo 短代码（Shortcodes）

短代码在 `layouts/shortcodes/` 中定义，以 `.html` 为扩展名，在 Markdown 中以 `{{< >}}` 或 `{{% %}}` 调用。

**调用方式区别：**
- `{{< shortcode >}}` — 内容不经 Markdown 渲染，原样传入
- `{{% shortcode %}}` — 内容先经 Markdown 渲染再传入

**Docsy 内置常用短代码：**

| 短代码 | 用途 |
|--------|------|
| `{{< alert >}}` | 提示/警告框（title、color 参数） |
| `{{< tabpane >}}` | 多 Tab 代码示例（常用于多语言对比） |
| `{{< readfile >}}` | 引入外部文件内容 |
| `{{< youtube id="..." >}}` | 嵌入 YouTube 视频 |
| `{{< imgproc >}}` | 响应式图片处理 |

**自定义短代码示例（`layouts/shortcodes/button.html`）：**
```html
<a class="btn btn-lg btn-primary me-3 mb-4" href="{{ .Get "link" }}">
  {{ .Get "text" }} <i class="fas fa-arrow-alt-circle-right ms-2"></i>
</a>
```
调用：`{{< button link="/docs/" text="Get Started" >}}`

---

## 5. npm Scripts 工作流

**标准命令集（package.json）：**

```json
{
  "scripts": {
    "build": "hugo --cleanDestinationDir -e dev -DFE",
    "build:production": "hugo --cleanDestinationDir --minify",
    "build:preview": "hugo --cleanDestinationDir --minify --baseURL \"${DEPLOY_PRIME_URL:-/}\"",
    "serve": "netlify dev -c \"hugo serve -DFE --minify\"",
    "check-links": "make check-links",
    "check-links:all": "HTMLTEST_ARGS= make check-links",
    "test": "npm run check-links"
  }
}
```

**关键构建参数说明：**

| 参数 | 说明 |
|------|------|
| `--cleanDestinationDir` | 构建前清空 `public/` 目录 |
| `-e dev` | 指定环境为 dev（影响条件模板逻辑） |
| `-D` | 包含草稿页（draft: true） |
| `-F` | 包含未来日期页 |
| `-E` | 包含已过期页 |
| `--minify` | 压缩 HTML/CSS/JS 输出 |

**Git submodule 初始化钩子（preinstall/prepare）：**
```json
{
  "preinstall": "git submodule update --init --depth 1",
  "prepare": "git submodule update --init --depth 1 && cd themes/docsy && npm install"
}
```
这样 `npm install` 时会自动拉取 submodule 和安装主题依赖。

---

## 6. config.yaml 配置结构

```yaml
baseURL: https://example.com/
theme: [docsy]

# 多语言配置
languages:
  en:
    title: Site Title
    contentDir: content/en
    languageName: English

# Hugo 功能开关
enableGitInfo: true      # 启用 Git 提交信息（用于"最后修改时间"）
enableRobotsTXT: true    # 自动生成 robots.txt

params:
  # Docsy 主题参数
  github_repo: https://github.com/org/repo   # "编辑此页"链接基准
  github_branch: main

  # 自定义参数（可在模板中以 .Site.Params.xxx 访问）
  versions:
    core: v1.78.1
    go: v1.79.1

  # UI 配置
  ui:
    sidebar_menu_compact: true   # 侧边栏折叠模式
    navbar_logo: true

  # Google Analytics
  googleAnalytics: G-XXXXXXXXXX

  # GTM（Docsy 扩展参数）
  gtmID: GTM-XXXXXXXX
```

**在模板中访问自定义参数：**
```html
{{ .Site.Params.versions.core }}
```

**在 Markdown 内容中访问（通过短代码或模板）：**
```go
{{ site.Params.versions.go }}
```

---

## 7. Netlify 部署配置

**`netlify.toml` 标准结构：**

```toml
[build]
publish = "public"         # Hugo 输出目录
command = "npm run build:preview"  # PR Preview 使用的命令

[context.production]
command = "npm run build:production"   # 合并到 main 后的生产构建

[[redirects]]
from = "https://www.example.com/*"
to   = "https://example.com/:splat"
force = true
```

**三种构建上下文：**

| 上下文 | 触发时机 | 典型命令 |
|--------|---------|---------|
| `production` | 主分支合并 | 最小化构建，不含草稿 |
| `deploy-preview` | PR 创建/更新 | 含草稿，baseURL 用 `$DEPLOY_PRIME_URL` |
| `branch-deploy` | 非主分支推送 | 同 preview |

**关键环境变量（Netlify 自动注入）：**
- `DEPLOY_PRIME_URL` — 当前预览的绝对 URL，用于 `--baseURL` 参数

---

## 8. 链接检查（htmltest）

**工具：** [htmltest](https://github.com/wjdp/htmltest) — 扫描构建产物 `public/` 目录检查所有链接。

**Makefile 集成模式：**
```makefile
HTMLTEST        ?= htmltest
HTMLTEST_ARGS   ?= --skip-external
HTMLTEST_DIR     = tmp

# 若 PATH 中无 htmltest，自动下载
ifeq (, $(shell which $(HTMLTEST)))
override HTMLTEST = $(HTMLTEST_DIR)/bin/htmltest
endif

check-links:
	$(HTMLTEST) $(HTMLTEST_ARGS)

get-link-checker:
	curl https://htmltest.wjdp.uk | bash -s -- -b $(HTMLTEST_DIR)/bin
```

**`.htmltest.yml` 配置结构：**
```yaml
DirectoryPath: public
ExternalTimeout: 5
IgnoreAltMissing: true
IgnoreDirectoryMissingTrailingSlash: true
IgnoreURLs:
  - ^https?://localhost          # 忽略本地链接
  - ^https://flaky-site\.com     # 忽略已知不稳定外部站点
IgnoreInternalURLs:
  - /some/generated/path         # 忽略生成的内部路径
```

**npm scripts 联动（先构建再检查）：**
```json
{
  "precheck-links": "npm run build",
  "check-links": "make check-links",
  "check-links:all": "HTMLTEST_ARGS= npm run check-links"
}
```

---

## 9. 拼写检查（cSpell）

**`.cspell.json` 结构：**
```json
{
  "version": "0.2",
  "language": "en",
  "words": [
    "grpc", "protobuf", "proto", "gRPC",
    "microservices", "balancer", "Kubernetes"
  ],
  "ignoreRegExpList": [
    "\\{\\{.*?\\}\\}"   // 忽略 Hugo 模板语法
  ],
  "ignorePaths": [
    "themes/", "public/", "node_modules/"
  ]
}
```

**在单个 Markdown 文件中忽略特定词：**
```yaml
---
spelling: cSpell:ignore someSpecificWord anotherWord
---
```

---

## 10. 局部模板（Partials）覆盖

Docsy 主题提供了大量可被项目覆盖的局部模板（Partials）：

**常见覆盖点：**

| 文件路径 | 用途 |
|---------|------|
| `layouts/partials/hooks/head-end.html` | 在 `</head>` 前注入自定义脚本/样式 |
| `layouts/partials/hooks/body-end.html` | 在 `</body>` 前注入（如 GTM noscript） |
| `layouts/partials/navbar.html` | 完全替换顶部导航 |
| `layouts/partials/footer.html` | 完全替换页脚 |
| `layouts/partials/meta.html` | 自定义 `<meta>` 标签 |

**Hooks 机制（推荐方式，最小侵入）：**
```html
<!-- layouts/partials/hooks/head-end.html -->
<!-- GTM Script -->
{{ with .Site.Params.gtmID }}
<script>(function(w,d,s,l,i){...})(window,document,'script','dataLayer','{{ . }}');</script>
{{ end }}
```

---

## 11. 多语言国际化

**`config.yaml` 中配置多语言：**
```yaml
defaultContentLanguageInSubdir: false  # true 则英文也带 /en/ 前缀

languages:
  en:
    contentDir: content/en
    languageName: English
    weight: 1
  zh:
    contentDir: content/zh
    languageName: 中文
    weight: 2
```

**内容目录结构：**
```
content/
├── en/
│   └── docs/introduction.md
└── zh/
    └── docs/introduction.md
```

**注意：** 未翻译的页面会 fallback 到默认语言（通常为英文）。

---

## 12. 版本号集中管理模式

将框架/依赖版本集中定义在 `config.yaml` 的 `params` 中，避免散落在各个文档页面：

```yaml
params:
  versions:
    core: v1.78.1
    go: v1.79.1
    java: v1.79.0
```

**在短代码中引用（`layouts/shortcodes/version.html`）：**
```html
{{ $lang := .Get 0 }}
{{ index .Site.Params.versions $lang }}
```

**在 Markdown 中调用：**
```markdown
当前 Go gRPC 版本：{{< version go >}}
```

**更新版本时只需修改 `config.yaml` 一处**，所有引用该参数的页面自动更新。

---

## 13. 常用调试技巧

**查看 Hugo 内置变量（在模板中）：**
```html
{{ printf "%#v" .Params }}
{{ printf "%#v" .Site.Params }}
```

**强制重新构建（清除缓存）：**
```bash
hugo --cleanDestinationDir --ignoreCache
```

**检查哪个模板被渲染了（--verbose 模式）：**
```bash
hugo --verbose 2>&1 | grep "Render"
```

**本地服务器热更新但不打开浏览器：**
```bash
hugo serve --disableLiveReload=false --navigateToChanged
```

---

## 14. 关键路径速查

| 功能 | 路径 |
|------|------|
| 站点全局配置 | `config.yaml` |
| Netlify 部署 | `netlify.toml` |
| 链接检查配置 | `.htmltest.yml` |
| 拼写检查配置 | `.cspell.json` |
| 自定义短代码 | `layouts/shortcodes/*.html` |
| 主题钩子注入点 | `layouts/partials/hooks/` |
| 顶部导航配置 | `config.yaml` → `params.menu.main` 或各页 Front Matter |
| 所有 Markdown 内容 | `content/` |
| 静态文件（图片等） | `static/` → 映射到根 URL |
| 构建产物 | `public/`（.gitignore 中） |
