# Life Note GitHub Pages 知识库改造设计

## 目标

将现有以 Markdown 和图片为主体的 `life-note` 仓库改造成可通过 GitHub Pages 访问的内容型知识库。改造后继续以 Markdown 作为唯一内容源，在不引入后端服务的前提下，提供清晰的首页、分类导航、站内搜索、响应式阅读、暗色模式和自动部署。

预期公开地址为：

```text
https://zhangqian-silk.github.io/life-note/
```

## 范围

本次改造包含：

- 使用 VitePress 生成静态站点。
- 为现有内容建立首页、顶部导航和分类侧边栏。
- 启用本地全文搜索、文章目录、更新时间、编辑链接和上下篇导航。
- 添加少量主题样式，使站点具备统一且适合中英文混排的视觉表现。
- 兼容现有 Markdown、相对图片和内嵌 HTML。
- 使用 GitHub Actions 在 `master` 分支更新后部署 GitHub Pages。
- 添加本地开发、构建和验证命令。
- 更新 README，说明内容结构、开发方式和部署方式。

本次改造不包含：

- 后端服务、账号体系、评论系统或数据库。
- 博客时间线、RSS、文章标签系统或内容管理后台。
- 批量重写现有文章内容。
- 批量重命名中文文件或改变现有目录结构。
- 批量有损压缩原始图片。
- 自定义域名配置。

## 技术选型

采用 VitePress 作为静态站点生成器，npm 作为包管理工具，Node.js 24 作为 CI 构建环境。

选择 VitePress 的原因：

- 当前内容已经是 Markdown，不需要转换数据模型。
- 文件系统路由能直接映射现有目录。
- 默认主题已经提供导航、侧边栏、暗色模式和响应式布局。
- 内置本地搜索无需外部搜索服务。
- 可通过少量 CSS 完成品牌化，不需要维护完整前端应用。
- 生成结果是纯静态资源，符合 GitHub Pages 的部署模型。

MkDocs Material 也适合文档站，但会引入 Python 工具链；Docusaurus 更适合包含博客、版本化文档和插件体系的较大型站点。两者对当前规模没有明显收益。

## 仓库结构

保留现有内容位置，在仓库根目录直接运行 VitePress：

```text
life-note/
├── .github/
│   └── workflows/
│       └── deploy-pages.yml
├── .vitepress/
│   ├── config.mts
│   └── theme/
│       ├── index.ts
│       └── custom.css
├── daily/
├── launch/
├── others/
├── travel/
├── 麻将/
├── docs/superpowers/specs/
├── index.md
├── package.json
├── package-lock.json
├── README.md
└── web.md
```

不将内容整体搬入 `docs/`，原因是现有文档大量使用相对图片路径，`launch/Zsh.md` 还直接引用仓库内的配置路径。保留目录可以减少迁移风险和无意义的 Git 变更。

构建缓存、生成目录和依赖目录加入 `.gitignore`，包括：

```text
node_modules/
.vitepress/cache/
.vitepress/dist/
```

## 信息架构

### 顶部导航

顶部导航固定为：

- 首页
- 旅行攻略
- 开发环境
- 日常生活
- 麻将牌效
- 实用网站
- GitHub

### 分类侧边栏

侧边栏按路径切换：

- `/travel/`：万宁、义乌与横店、山西上下篇、江西、洛阳、银川、长江和青岛。
- `/launch/`：Windows、Go、OpenCloudOS、Zsh 和 PotPlayer。
- `/daily/`：营养元素。
- `/麻将/`：牌效率、实战策略和星野 Poteto 系列课程。

`agent.md`、`travel/_prompt.md` 和设计规格文档不进入用户导航。它们可以继续作为仓库维护资料存在。

### 首页

新增根目录 `index.md`，使用 VitePress 首页布局，包含：

1. `Life Note` 标题和个人知识库简介。
2. “开始阅读”和“查看 GitHub”两个主要入口。
3. 旅行攻略、开发环境、日常生活和麻将牌效四张分类卡片。
4. Zsh 环境配置、山西攻略、营养元素和牌效率等精选入口。
5. 简短说明：内容持续更新，具体时效信息应以官方渠道为准。

首页只承担导航和项目说明，不动态计算最近提交，避免增加构建脚本和维护成本。

## 阅读体验

使用 VitePress 默认主题作为基础，并进行有限度定制：

- 采用中性蓝作为强调色，搭配浅色和暗色背景。
- 使用系统中文字体栈，保证不同平台上的加载速度和可读性。
- 正文设置舒适行高和最大阅读宽度。
- 图片自适应正文宽度，保留原始宽高比例和现有居中效果。
- 表格在窄屏上允许横向滚动。
- 代码块、提示框、标题锚点和内联代码沿用默认主题行为。
- 移动端使用默认折叠导航和侧边栏。
- 不添加复杂动画、远程字体和第三方统计脚本。

主题配置启用：

- 本地全文搜索。
- 二至四级标题目录。
- 最后更新时间。
- 上一篇和下一篇导航。
- 指向 `master` 分支源文件的 GitHub 编辑链接。
- GitHub 社交链接。
- 简体中文界面文案。

## 路由与资源处理

仓库属于 GitHub Pages 项目站点，因此生产环境基础路径设置为：

```ts
base: '/life-note/'
```

本地开发环境仍从根路径提供内容。页面链接优先使用 VitePress 能处理的站内路径，图片继续使用相对于 Markdown 文件的路径。

现有图片分为 Markdown 图片语法和内嵌 HTML `<img>` 两类：

- 构建前检查每个本地图片引用是否存在。
- 保留现有 `width` 设置和居中结构。
- 为缺少相关属性的 HTML 图片补充 `loading="lazy"` 和 `decoding="async"`。
- 启用 Markdown 图片懒加载。
- 第一阶段不更改图片格式；构建后记录站点产物体积，为后续 WebP 优化提供基线。

历史文档中指向旧 `zhangqian-silk/Markdown` 仓库的自引用链接应改为当前 `life-note` 仓库链接，避免读者跳转到旧地址。

## 构建与部署

`package.json` 提供以下命令：

```text
npm run docs:dev      启动本地开发服务器
npm run docs:build    生成生产静态站点
npm run docs:preview  预览生产构建结果
npm run check         执行内容检查和生产构建
```

GitHub Actions 工作流在以下情况触发：

- 向 `master` 分支推送。
- 在 Actions 页面手动执行。

工作流分为 `build` 和 `deploy` 两个任务：

1. 检出完整 Git 历史，以生成准确的最后更新时间。
2. 安装 Node.js 24 并启用 npm 缓存。
3. 使用 `npm ci` 安装锁定依赖。
4. 执行内容检查和 VitePress 生产构建。
5. 上传 `.vitepress/dist` 为 GitHub Pages artifact。
6. 通过 `github-pages` environment 部署 artifact。

工作流只授予 `contents: read`、`pages: write` 和 `id-token: write` 所需权限，并使用并发组避免旧部署覆盖新部署。

GitHub 仓库需要一次性在 `Settings → Pages → Build and deployment` 中将发布源设置为 `GitHub Actions`。该设置不由仓库代码自动修改。

## 内容检查与错误处理

在生产构建前执行只读内容检查：

- 检查 Markdown 中引用的本地图片是否存在。
- 检查配置中列出的导航目标是否存在。
- 将缺失资源视为构建失败，防止发布带有明显 404 的版本。
- 外部网站链接不作为 CI 阻断项，避免网络波动导致部署失败。

VitePress 构建错误、资源缺失或配置错误都会使 GitHub Actions 停止在 build 阶段，不触发 deploy。已上线的上一版本继续可用。

## 验证方案

实现完成后执行：

1. `npm ci`，验证全新环境可重复安装。
2. `npm run check`，验证本地资源和生产构建。
3. 检查构建目录存在入口 `index.html`。
4. 检查构建产物中包含主要分类页面和搜索索引。
5. 使用生产预览检查首页、分类导航、中文路径、暗色模式和移动端布局。
6. 抽查青岛、山西、Zsh、营养元素和麻将课程中的图片、表格、代码块和内嵌 HTML。
7. 确认所有生成 URL 在 `/life-note/` 基础路径下工作。
8. 推送后检查 GitHub Actions 的 build 和 deploy 任务均成功。

## 验收标准

- 首页能够进入所有公开分类。
- 顶部导航和分类侧边栏中列出的全部内容均可访问。
- 分类侧边栏和上下篇导航不存在无效目标。
- 中文搜索能够找到正文内容。
- 文档中的本地图片均能显示，且不会溢出正文区域。
- 桌面端和移动端均可完成导航与阅读。
- 浅色、暗色主题均不存在明显对比度问题。
- `npm run check` 在干净检出环境中通过。
- `master` 更新后能够自动部署到 GitHub Pages。
- README 清楚说明站点地址、本地命令和发布流程。

## 后续优化

以下内容仅在首版上线后按实际需要处理：

- 将体积较大的照片转换为 WebP 或 AVIF，并保留必要的高质量原图。
- 添加站点地图、RSS 或访问统计。
- 使用自定义域名。
- 根据文章数量增长情况自动生成侧边栏。
- 为有明显时效性的旅行和健康内容补充来源及复核日期。
