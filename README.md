# Life Note

旅行、生活、开发环境与兴趣学习的个人知识库。内容使用 Markdown 编写，通过 VitePress 生成静态网站，并由 GitHub Actions 自动发布到 GitHub Pages。

站点地址：<https://zhangqian-silk.github.io/life-note/>

## 内容导航

- `travel/`：城市景点、餐厅、路线和旅行贴士。
- `launch/`：操作系统、Shell、编程语言与常用软件配置。
- `daily/`：营养与日常生活笔记。
- `麻将/`：麻将牌效率理论和实战策略。
- `web.md`：实用网站收藏。

## 本地开发

需要 Node.js 20 或更高版本。

```shell
npm ci
npm run docs:dev
```

常用命令：

```shell
npm run docs:dev      # 启动开发服务器
npm run docs:build    # 生成生产站点
npm run docs:preview  # 预览生产构建
npm run check         # 检查内容资源并执行生产构建
```

构建结果位于 `.vitepress/dist/`。

## 添加内容

1. 将 Markdown 文档放入对应分类目录。
2. 图片放入文档同级的 `images/` 目录，并使用相对路径引用。
3. 在 `.vitepress/config.mts` 中补充导航或侧边栏入口。
4. 提交前运行 `npm run check`，确认图片引用和生产构建正常。

仓库中的 `agent.md` 记录了 Markdown 格式约定。

## GitHub Pages 部署

`.github/workflows/deploy-pages.yml` 会在内容推送到 `master` 后自动构建和发布站点。

首次使用时，需要在 GitHub 仓库中打开：

```text
Settings → Pages → Build and deployment → Source → GitHub Actions
```

此后每次推送到 `master` 都会触发新版本部署，也可以在 Actions 页面手动运行工作流。

## License

[MIT](LICENSE)
