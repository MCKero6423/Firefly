# 迁移与上游同步说明

本仓库从 [Firefly](https://github.com/CuteLeaf/Firefly) 派生，内容与配置由原 **fuwari** 博客迁移而来。

本文档记录 **同步 Firefly 上游时需要人工核对的部分**，避免改动被覆盖或合并冲突。

---

## 一、同步上游时零冲突的部分（改配置即可）

以下改动都落在 Firefly 设计好的配置位上，**同步上游不会冲突**：

| 文件 | 内容 |
|---|---|
| `src/config/siteConfig.ts` | 站点信息、主题色、favicon、页面开关、图片格式/质量 |
| `src/config/profileConfig.ts` | 头像、昵称、社交链接 |
| `src/config/navBarConfig.ts` | 导航栏（含「隐私政策」入口、状态/统计/开往外链） |
| `src/config/friendsConfig.ts` | 友链数据 |
| `src/config/sponsorConfig.ts` | 打赏方式 + `runningCosts`（运行成本） |
| `src/config/analyticsConfig.ts` | GA / Umami（**ID 留空**，见下方 ConsentScripts） |
| `src/config/commentConfig.ts` | Twikoo（自托管 jsUrl） |
| `src/config/musicConfig.ts` | 音乐播放器（当前关闭） |
| `src/config/backgroundWallpaper.ts` | 壁纸（桌面单图 / 移动空数组） |
| `src/config/fontConfig.ts` | 字体（本地化到 `public/assets/fonts`） |
| `src/config/FooterConfig.html` | 页脚：萌ICP备 + 隐私政策 + Cookie 首选项 |
| `astro.config.mjs` | astro-consent 配置、swup preload、图片格式 |
| `src/content/**` | 文章、`spec/about.md`、`spec/privacy.md`、`spec/friends.mdx` |

---

## 二、独立新增的文件（不冲突，但需自行维护）

| 文件 | 用途 |
|---|---|
| `src/styles/overrides.css` | **所有对 Firefly 样式的定制集中于此**（对比度修复、Cookie 主题化、移动端渐变） |
| `src/components/features/ConsentScripts.astro` | Cookie 同意后加载 GA / Umami |
| `src/components/features/AiSummary.astro` | 洪墨 AI 摘要（同意后加载） |
| `src/components/features/RunningCosts.astro` | 运行成本组件 |
| `src/plugins/rehype-cloudflare-image.mjs` | `img.mckero.com` → `/cdn-cgi/image/` + srcset |
| `src/pages/privacy.astro` | 隐私政策独立页 |
| `src/cookiebanner/styles.css` | astro-consent 生成的样式（主题化部分已移到 overrides.css） |
| `public/assets/fonts/` | 本地化字体（Zen Maru Gothic / Inter / JetBrains Mono） |
| `public/twikoo.min.js` | 自托管 Twikoo |

---

## 三、⚠️ 改动了 Firefly 上游文件的补丁（同步时必须重放）

这些改动落在上游文件里，`git merge` 上游时**可能冲突**，需逐个核对：

### 1. `src/layouts/WallpaperSection.astro`
- **首张壁纸直接渲染进 DOM**（不再放 `<template>`），带 `fetchpriority="high"` + `fadeIn={false}`
  - 原因：`<template>` 内的图片不发起请求，必须等轮播 JS 克隆，导致 LCP 发现延迟 ~560ms
- **widths 档位调整**：桌面 `[1280,1440,1600,1920]`、移动 `[640,720]`
  - 原因：避免 1335px 视口被迫下载 1920w；移动端去掉 828w（object-cover 全屏背景堆像素无意义）

### 2. `src/layouts/Layout.astro`
- **特效条件渲染**：`SakuraEffect` / `WavesEffect` 关闭时不再渲染组件
  - 原因：原代码无条件渲染，配置关闭后脚本仍被下载执行
- 引入 `src/styles/overrides.css`（实际在 `main.css` 里 import）

### 3. `src/components/layout/PostCard.astro`
- 底部标签补齐 `material-symbols:tag-rounded` 图标
  - 原因：与旧 fuwari 博客的标签样式对齐

### 4. `src/utils/layout-utils.ts`
- `mobile` 显式配置为空数组时**不回退到桌面图**
  - 原因：原逻辑 `mobile.length > 0 ? mobile : desktop`，无法表达"移动端不要壁纸"

### 5. `src/styles/main.css`
- 在末尾 `@import './overrides.css';`
  - **这是让所有定制生效的唯一入口，务必保留**

### 6. `src/layouts/MainGridLayout.astro`
- 移除了 `cookiebanner/styles.css` 的重复导入（Layout 已导入）

---

## 四、已知并已回退的尝试（勿重复踩坑）

| 尝试 | 结果 |
|---|---|
| `build.inlineStylesheets: "always"` | ❌ 首页 HTML 605KB，**TBT 0 → 110ms**，得不偿失 |
| preload 横幅字体 | ❌ Zen Maru Gothic 每字重独立文件，preload 带出 4 个共 44.7KB，抢 LCP 带宽 |
| 把 `@plugin '@tailwindcss/typography'` 移到 `markdown.css` | ❌ Tailwind v4 的 `@plugin` 在 `@reference` 文件里不产出规则，文章页排版全丢 |
| 精简 i18n 语言表 | ❌ 单语言表本身 19KB，非多语言浪费 |

---

## 五、第三方依赖的版本敏感点

- **astro-consent v2.1.1**：`ConsentScripts.astro` 里的 `syncModal()` 与 `localize()` 依赖该版本的具体 DOM 结构
  （`#astro-consent-modal`、`.cb-toggle[data-key]`、`.cb-row`、`.cb-container`）。
  升级该库时**必须复核**，否则弹窗勾选同步与中文文案会失效。

---

## 六、当前性能基线

| 场景 | 分数 |
|---|---|
| 桌面（PageSpeed） | **100** |
| 移动（Moto G / 4G 模拟） | **90–94** |

移动端瓶颈：3 个 CSS 共 39.6KB（gzip）在 4G 下需 ~1.5s，为物理带宽限制。
TBT 0ms、CLS ≤0.003 均为满分。
