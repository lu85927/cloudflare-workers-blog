# Zine · 纸本杂志主题

给 `cloudflare-workers-blog`（Worker + KV + mustache）用的前端主题，适配**带主题切换 / 单页路由的那版 index.js**。
视觉方向：纸本独立杂志 —— 宋体大刊头、等宽小字元信息、单色朱红、分隔线代替卡片、文章自动编号、双栏错位。

## 设计取向（与现有 5 套主题的差别）

| 维度 | argon / aero / JustNews / meadow / pure | zine |
|---|---|---|
| 布局 | 主栏 + 右侧边栏 | 无侧栏，单栏通排；组件移到页脚 colophon |
| 容器 | 卡片、圆角、阴影 | 无框、无圆角、只用 1px 分隔线与留白 |
| 标题 | 无衬线 | 宋体（Songti / Noto Serif SC），刊头超大字号 |
| 图像 | 原色、hover 放大 | 轻微去色（grayscale .55），hover 回色 |
| 编号 | 无 | CSS counter 自动编号 01…，首篇为特稿 |
| 目录 | 侧栏列表 | 顶部横排带序号 + 页脚附录版块 |
| 深色模式 | 深浅切换 | 同样支持（纸黑 + 米白字 + 暖橙点缀） |

## 文件

```
themes/zine/
├── theme.json          【必需】后台"外观设置"读它显示中文名与简介
├── index.html          首页：刊头 / 特稿 / 双栏错位列表 / 分页 / 页脚
├── article.html        文章页：头图 / 编号标题 / 作者 / 首行缩进正文 / 上下篇 / 版权声明
├── page.html           单页模板：/page/<slug>，变量与 worker 的 renderPage 完全对应
└── static/css/main.css 全部样式（含深色模式、动效、响应式）
```

## 与新版 index.js 的对接点

| worker 行为 | 主题侧要求 | 本主题状态 |
|---|---|---|
| `themeURL = THEME_BASE + 当前主题 + "/"`，取 `index.html` | 目录名即主题 id，只允许 `[A-Za-z0-9_-]` | 目录名 `zine`，符合 |
| 后台列出主题时读 `themes/<id>/theme.json` | 需要 `theme.json`，含 `name` / `desc` | 已提供 |
| 后台模板 `admin/index`、`admin/edit` 取不到时**自动回退到 argon** | 主题可以不做后台 | 未做，依赖回退（可用） |
| `/page/<slug>` → `renderPage()` → 取 `page.html` | 模板用 `pageTitle / pageContent / pageCreateDate / pageUpdateDate / description` | 已全部对应 |
| 新 OPT 字段 `author_name` / `copyright_notice` / `favicon` / `logo_url` / `custom_css` | 有则渲染、无则整块跳过 | 已接入 |
| `OPT.user / password / cacheToken / cacheZoneId` 在渲染前被清空 | 模板不能依赖这些字段 | 未使用 |

`OPT.theme_color`、`OPT.logo`（png）本主题没有使用：设计自带纸色配色，地址栏颜色固定为纸色/纸黑；刊头用的是矢量 `logo_url`。

## 启用方式

1. 把 `themes/zine` 推到仓库（`lu85927/cloudflare-workers-blog`）。
2. 进后台 `/admin/theme`（外观设置）→ 点「刷新主题列表」→ 卡片里出现 **Zine 纸本杂志** → 切换。
   切换会写 KV `SYSTEM_THEME` 并 purge 全站缓存，前台立刻生效。
3. 也可以直接设 KV：`SYSTEM_THEME = zine`。

## 上线前建议改一处

模板里的 CSS 地址是 `@master`：

```
https://cdn.jsdelivr.net/gh/lu85927/cloudflare-workers-blog@master/themes/zine/static/css/main.css
```

首次推送后换成具体 commit sha（`@26ddf35…` 这种），否则 jsDelivr 会按 master 缓存旧 CSS，改完样式看不到效果。三个模板各改一次。

## 已知限制 / 需要注意

1. **`?theme=` 与缓存键冲突（worker 侧问题，主题无法修）**
   worker 的缓存键是 `siteDomain/段1/段2/段3`，**不含 query**，但 `?theme=xxx` 会改变输出。
   于是任何人对首页发一次 `/?theme=pure`，这份纯主题渲染结果就会写进 `/` 的缓存，其他人跟着看到错误主题，直到下次 purge。
   同理 `?pageSize=` 也会。建议把 `CURTHEME` 拼进缓存键：
   ```js
   const M = "https://" + OPT.siteDomain + "/" + _ + "/" + b + "/" + L + "?t=" + CURTHEME;
   ```
   单页路由的 `cacheKey` 也建议带上 `CURTHEME`。
2. **后台不跟随主题**。即使做了 zine 的 admin 模板，`/admin` 的页面骨架也是 argon 的版式（回退来源），这是 worker 的设计，不是主题能改的。
3. **`page.html` 依赖单页数据**。`/page/<slug>` 只有在 KV 里存在 `PAGE_<slug>` 时才渲染，否则返回 `html404`；别名里带数字的 slug（如 `2024`）会被当成翻页页码，避开这种命名。
4. **`favicon.ico` 走不通**。worker 对 `/favicon.ico` 直接返回 404，浏览器实际取的是模板里的 `OPT.favicon`，已接。
5. 没有 JS 也能正常阅读：入场动画只在 `html.zine-anim` 存在时生效，脚本失败时内容照常可见。

## 本地预览

```powershell
node tools/render-zine-preview.mjs
# 输出 preview/index.html、article.html、page.html、index-dark.html
```

预览页里的 CSS 会替换成 `./assets/main.css`，其余与线上渲染完全一致；脚本会检查模板是否存在未解析的 `{{` 标记。
