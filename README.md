# plugin-moments

Halo 2 瞬间插件，已经内置前台渲染页面，不再强依赖主题提供 `moments.html` 或 `moment.html`。

## 功能

- 瞬间列表页 `/moments`
- 瞬间详情页 `/moments/{name}`
- 标签筛选与分页
- 图片、视频、音频、外链卡片渲染
- 点赞、评论、RSS
- 插件模板兜底，主题模板可覆盖

## 关键文件

- `src/main/resources/templates/moments.html`
- `src/main/resources/templates/moment.html`
- `src/main/java/run/halo/moments/MomentRouter.java`
- `.github/workflows/ci.yaml`
- `.github/workflows/cd.yaml`
- `build.gradle`

## 构建

插件版本支持通过环境变量注入：

```gradle
version System.getenv("PLUGIN_VERSION") ?: "1.0.0"
```

GitHub Actions 会自动构建并生成新的版本号：

```text
1.0.${GITHUB_RUN_ID}
```

## 说明

- 当前项目目录：`C:\Users\Administrator\Desktop\halo-plugin-Minimalist`
- 当前保留的是插件运行、后台管理、前台模板、GitHub 打包所需的最小核心文件
- 如果主题后续自带 `moments.html` 或 `moment.html`，Halo 会优先使用主题模板
