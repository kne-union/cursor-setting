---
name: product-official-site
description: 为业务系统生成产品官网首页（landing page），并用真实组件 + mock 数据批量截取中英文产品截图。用户说「做官网」「产品官网」「给 XX 系统也搞一套首页」「官网截图」「宣传截图」「产品截图」「重新截图」时使用。
---

# 产品官网首页 + 真实组件截图

参考实现：micro-hire、ai-interview-flowup 的 `src/components/OfficialSite`。

## 0. 开工前

- 业务仓按 business-development-workflow 在 workplace 副本开发；不启动、不重启、不 kill 用户的 dev server，只请用户启动。
- 用 AskQuestion 定三件事：截图方式（推荐真实组件 + mock）、根路径 `/` 是否改为官网首页、主色（沿用品牌色）。
- 内容约束（默认遵守）：
  - 不用真实数据，人物、公司、数字都虚构；
  - 尽量少暴露管理端（admin）页面和具体操作流程，只展示客户端 / 候选人端能力。

## 1. 页面结构（components/OfficialSite）

- 文件：`index.js`（页面）、`Standalone.js`（独立入口）、`screenshots.js`（截图名清单）、`locale/zh-CN.js` + `locale/en-US.js`（键数一致）、`style.module.scss`。
- 区块顺序：吸顶导航（logo、锚点、中英切换、登录、开始使用）→ 首屏（标题、卖点、浏览器外框截图轮播）→ 产品特点 → 产品介绍（多组截图轮播，带标签）→ 使用流程 → 适用角色 → 可信保障 → CTA → 页脚。
- 截图路径 `publicAsset('/screenshots/{name}-{zh|en}.jpg')`，按当前语言取图；图片 lazy 加载。
- 移动端只用 `@kne/responsive-utils` 的 `resp.mobile-container` 和 `useIsMobile`。

## 2. 路由与启动（bootstrap.js）

- 判定首页：`pathname 去尾斜杠 === basePath` 且不是退出登录回跳（如 `hasLogoutReturn()`）或候选人 `?code=` 入口。
- 首页直接 `root.render(<OfficialHome />)`，跳过 globalInit 和远程组件。
- `App` 改为 `import('./App')` 动态加载，只在非首页分支加载；否则 App 里模块级副作用（如请求 `ipapi.co`）会在首页报错。
- Standalone 自建 BrowserRouter（basename）、antd ConfigProvider、ResponsiveProvider；locale 存本地 state；自己移除 boot mask，补 body 样式。
- 登录 / 开始使用用整页跳转 `window.location.assign(管理端地址)`，不要用 SPA navigate 进入依赖 globalInit 的页面。

## 3. 截图工具（放 /tmp/<proj>-pw，不进仓库）

- `playwright-core` + `chromium.launch({ channel: 'chrome', headless: true })`；视口 1440×900，`deviceScaleFactor: 2`。
- 登录态：伪造无签名 JWT 写入 `localStorage['kne-oidc:oidc-spa:token']`；语言写 `X-User-Locale`。
- `context.route('**/api/**')`：除 `/api/oidc` 外全部按 `mock.js` 返回 `{ code: 0, data }`；`/static/file-url/{id}` 返回纯文本 URL；记录并打印未 mock 的请求（UNMOCKED），直到为 0。
- 素材：`/__mock/*` 路由到本地 `assets/` 目录（头像、logo、封面、人像）。
- `shot.js <locale> <path> <out> [waitMs]`，用环境变量做交互：`CLICK`（`||` 分隔）、`SCROLLTEXT` + `NTH` / `OFFSET`、`SCROLL`（选择器）；隐藏右下角 fixed 浮标（modules-dev 灯泡按钮）。
- `batch.sh` 依次截中英文全部页面，再 `sips -s format jpeg -s formatOptions 80 -Z 2160` 压缩。
- **先输出到 /tmp，全部完成后一次性拷进 `public/screenshots/`**：dev server 监听 public，逐张写入会让用户页面反复刷新。

## 4. mock 数据规则

- `data.js` 分中英两套人设：英文用欧美姓名、`example.com` 邮箱、美国城市 / 学校，人像也用非亚洲面孔；中文用虚构中文名。
- 18 位雪花 ID 用字符串，用 `sid(base, i)` 生成，避免 JS 精度丢失。
- 核对数值单位（百分比字段是 0–100 还是 0–1，权重是否要除 100），截图前逐张看图。
- logo、头像不能留空：
  - 公司 logo 自己用 SVG 设计，再用 Playwright 渲染成 PNG；
  - 人像（管理员头像、候选人摄像头画面、数字人）在用户明确要求后用 GenerateImage 生成，不用网图。
- 报告类大结构按组件读取的字段手写虚构数据，不复用仓库里可能来自真实数据的 mock JSON。

## 5. 难以走到的界面：直接渲染组件 + mock props

面试间、视频房间、设备检测后的页面等依赖 RTC / 摄像头 / 多步检测时，**不要驱动完整业务流程**，直接渲染组件：

1. 打开 modules-dev 组件文档页 `/modules-dev-components/<Component>`。
2. `context.route(/README_md.*\.chunk\.js$/)` 拦截文档 chunk，把目标示例的 `code: "..."` 字符串替换成自己的 mock 代码（用 `JSON.stringify(code)` 写回）；mock props 通过 `addInitScript` 挂到 `window.__SHOT__`。
3. mock 代码里外包一层 `position: fixed; inset: 0` 的容器，并注入样式去掉祖先的 `transform` 和 `container-type`（`.example-driver-runner{transform:none}`、`.core-page-content{container-type:normal}`），否则 fixed 不相对视口。
4. 示例懒渲染：先 `scrollIntoViewIfNeeded` 到示例标题再截图。
5. 语言：既给组件传 `locale` prop，又用 `components-core:Global@SetGlobal` 设置 `globalKey="locale"`，否则子组件仍显示中文。
6. 文档页其他示例会请求接口，把 `/api/**` 统一 mock 掉，避免 401 跳登录。
7. 视频画面用图片：优先用组件的图片 props（如 `background`、`userCamera`）；必须走 `<video>` 时只设 `poster`，不录 webm、不抽视频帧。

若页面能直接进入（不需要多步检测），也可以沿用 micro-hire 做法：真实页面 + 接口 mock + `addInitScript` 注入假 TRTC，用图片代替视频画面。

## 6. 验证

- `home.js`：打开 `/`（中、英、手机宽度），检查 console 错误数、外部请求数、首屏出现时间、所有截图 200。
- 逐张查看截图：logo、头像不为空；英文图里没有中文文案和中国人物；没有 localhost 链接、没有空白区域。
- 汇报时给出 `cd "<WORKPLACE_ROOT>" && npm start` 和访问地址，等用户验收后再按开发流程提交。

## 禁止

- 用旧版 / 废弃 UI 的宣传图冒充当前界面截图。
- 截图里出现真实客户数据、空 logo、默认头像占位。
- 为了截图逐步点击多步设备检测等完整流程（应改为渲染组件）。
- 截图过程中逐张写入 `public/` 触发用户页面反复刷新。
- 自行启动或重启用户的本地服务。

