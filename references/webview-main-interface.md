# Scriptable WebView 主界面最佳实践

先按 [Apple UI 设计规范](apple-ui-design.md) 确定界面目的、信息层级、文字、颜色、控件、状态和验证范围。本文件补充 WebView 的数据、安全和生命周期边界。

## 目录

- [选择页面类型](#选择页面类型)
- [保持单向数据流](#保持单向数据流)
- [构建安全的 HTML](#构建安全的-html)
- [限制导航和资源请求](#限制导航和资源请求)
- [选择兼容的事件桥](#选择兼容的事件桥)
- [管理页面生命周期](#管理页面生命周期)
- [安排主界面内容](#安排主界面内容)
- [适配尺寸和外观](#适配尺寸和外观)
- [处理全屏安全区和宿主控件](#处理全屏安全区和宿主控件)
- [提供可访问界面](#提供可访问界面)
- [统一使用 iOS 式加载指示器](#统一使用-ios-式加载指示器)
- [处理加载和错误状态](#处理加载和错误状态)
- [控制性能和复杂度](#控制性能和复杂度)
- [执行验证矩阵](#执行验证矩阵)
- [审查清单](#审查清单)

## 选择页面类型

先确定 WebView 展示哪一种内容。

| 页面类型 | 加载方式 | 请求规则 |
| --- | --- | --- |
| 脚本生成的自包含主界面 | `loadHTML()` | 不加载外部资源；请求过滤需先验证内部主文档 URL |
| 随脚本分发的 HTML、CSS 和图片 | `loadFile()` | 只允许文件目录内的资源和明确外部主机 |
| 已知远程页面 | `loadURL()` 或 `loadRequest()` | 只允许业务需要的协议、主机和跳转 |
| 大型静态图片或文档 | `loadFile()` 或 `loadHTML()` | 限制文件来源和大小 |

把自包含主界面和远程运营商页面放在不同的 WebView 实例中。主界面不需要远程资源时，不给它设置 HTTP 基地址。

Scriptable 的 `preferredSize` 只在 Siri 或快捷指令中使用，应用内主界面不能依赖它确定布局。

## 保持单向数据流

按以下顺序组织代码。

```text
规范化后的业务数据
  -> 只含显示字段的 view model
  -> HTML 字符串
  -> WebView 展示
  -> 页面发送 action
  -> 主脚本执行副作用
  -> 生成新的 view model 并重载页面
```

页面只负责显示、收集点击和更新短暂的视觉状态。网络请求、钥匙串、文件写入、日历修改和通知调度留在主脚本中。

把发送给页面的数据缩减为显示所需字段。不要把 Cookie、令牌、完整响应、调试对象或页面不使用的个人数据写入 DOM。

为 action 定义固定集合，例如 `refresh`、`openPortal` 和 `updateCredential`。主脚本收到 action 后再次校验，不执行页面传来的任意函数名、URL 或 JavaScript。

## 构建安全的 HTML

外部文本进入 HTML 文本节点或属性前进行 HTML 编码。

```javascript
function escapeHtml(value) {
  return String(value)
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#39;")
}
```

外部值进入 `evaluateJavaScript()` 的代码时使用 `JSON.stringify()`。不要把外部值直接拼进引号、事件处理器、CSS 或 HTML 标签名。

需要把 JSON 放进内联 `<script>` 时，同时转义小于号，防止数据中的 `</script>` 提前结束脚本。

```javascript
function jsonForInlineScript(value) {
  return JSON.stringify(value).replace(/</g, "\\u003c")
}
```

自包含页面可以加入内容安全策略。先确认不带策略的最小页面能在目标 Scriptable 版本中展示，再根据页面实际资源逐项增加允许项。

```html
<meta
  http-equiv="Content-Security-Policy"
  content="default-src 'none'; img-src data:; style-src 'unsafe-inline'; script-src 'unsafe-inline'; base-uri 'none'; form-action 'none'"
>
```

内容安全策略不能替代输出编码。启用后需要验证首次展示、页面重载、内联样式、内联脚本和图片。出现空白页时先移除策略确认基线，再逐项定位不兼容的指令。

## 限制导航和资源请求

Scriptable 的 `WebView.shouldAllowRequest` 默认允许所有请求。远程页面必须按协议和主机设置允许列表。

```javascript
function allowPortalRequest(request) {
  const url = request && request.url ? String(request.url) : ""
  return url.indexOf("https://portal.example.com/") === 0
}

const webView = new WebView()
webView.shouldAllowRequest = allowPortalRequest
```

自包含页面没有链接、表单和外部资源时，可以不设置 `shouldAllowRequest`。需要设置时，先在目标设备记录 `loadHTML()` 实际产生的内部主文档 URL，再加入规则。不要假定所有版本都使用完全相同的 `about:blank`、`data:` 或内部 Scheme；拦截主文档会直接产生空白页。

远程页面需要 CDN、认证跳转或图片主机时，逐个加入允许列表，并在真机检查被拒绝请求。不要为了修复缺失资源而允许所有 HTTP、HTTPS 或自定义 Scheme。

`loadRequest()` 会使用设置的方法、正文和请求头，但不会调用该 `Request` 的 `onRedirect`。需要限制 WebView 后续请求时，仍要设置 `shouldAllowRequest`。

凭据只发送给预期主机。远程页面跳转到未允许主机时停止导航，并向用户显示可以理解的错误。

## 选择兼容的事件桥

Scriptable 官方接口支持 `evaluateJavaScript(code, true)`。该 Promise 会等待页面调用全局 `completion(value)`。先在目标 Scriptable 版本验证首次展示、页面重载、连续操作和关闭页面。验证通过后，使用一次等待处理一个 action。

页面端保存一个 action 队列和一个等待中的 resolver。

```javascript
window.__scriptableActions = []
window.__scriptableResolve = null

function sendAction(action) {
  const resolve = window.__scriptableResolve
  if (typeof resolve === "function") {
    window.__scriptableResolve = null
    resolve(action)
    return
  }
  if (window.__scriptableActions.length === 0) {
    window.__scriptableActions.push(action)
  }
}

document.querySelectorAll("[data-action]").forEach((element) => {
  element.addEventListener("click", () => sendAction(element.dataset.action))
})
```

主脚本一次只注册一个等待。

```javascript
async function waitForAction(webView) {
  return await webView.evaluateJavaScript(`
    (() => {
      const queue = window.__scriptableActions
      if (!Array.isArray(queue)) {
        completion(null)
        return
      }
      if (queue.length > 0) {
        completion(queue.shift())
        return
      }
      window.__scriptableResolve = function(action) {
        window.__scriptableResolve = null
        completion(action)
      }
    })()
  `, true)
}
```

收到值后只接受固定 action。

```javascript
function normalizeAction(value) {
  const action = String(value || "")
  const allowed = ["refresh", "openPortal", "updateCredential"]
  return allowed.indexOf(action) >= 0 ? action : null
}
```

页面重载会销毁旧的队列和 resolver。主脚本在重载完成后重新调用 `waitForAction()`。目标 Scriptable 版本仍需真机验证关闭页面时 Promise 的结束行为。

如果等待期间出现空白页、页面重载后失效或关闭页面后 Promise 不结束，改用页面队列和低频轮询。轮询间隔使用 400 到 750 毫秒，并与 `present()` 竞争，页面关闭后停止。

```javascript
async function takeAction(webView) {
  return await webView.evaluateJavaScript(`
    (() => {
      const queue = window.__scriptableActions
      if (!Array.isArray(queue) || queue.length === 0) return null
      return queue.shift()
    })()
  `, false)
}

function sleep(milliseconds) {
  return new Promise((resolve) => {
    Timer.schedule(milliseconds, false, resolve)
  })
}

const presentation = webView.present(true).then(
  () => ({ type: "closed", error: null }),
  (error) => ({ type: "closed", error })
)

while (true) {
  const event = await Promise.race([
    sleep(500).then(() => ({ type: "tick" })),
    presentation
  ])
  if (event.type === "closed") break

  let action = null
  try {
    action = normalizeAction(await takeAction(webView))
  } catch (error) {
    continue
  }
  if (action) await handleAction(action)
}
```

不要同时保留等待 resolver 和轮询。页面队列最多保留一个未处理动作，动作执行期间禁用对应控件。

## 管理页面生命周期

同时等待页面关闭和页面 action。把拒绝转换成带类型的结果，避免未处理的 Promise。

```javascript
const presentation = webView.present(true).then(
  () => ({ type: "closed", error: null }),
  (error) => ({ type: "closed", error })
)

while (true) {
  const event = await Promise.race([
    waitForAction(webView).then(
      (action) => ({ type: "action", action }),
      (error) => ({ type: "bridgeError", error })
    ),
    presentation
  ])

  if (event.type === "closed") break
  if (event.type === "bridgeError") {
    console.warn(String(event.error))
    await presentation
    break
  }

  const action = normalizeAction(event.action)
  if (!action) continue
  await handleAction(action)
}
```

一个 action 处理完成前禁用会重复提交的控件。刷新成功后先保存状态，再重载 HTML。失败时恢复控件并显示错误。页面关闭后停止注册新的事件等待。

不要调用 `waitForLoad()` 等待一个没有开始的导航。官方文档说明这种 Promise 不会结束。

## 安排主界面内容

用一句话写出页面的主任务。流量详情页的主任务可以是“查看剩余流量并在需要时刷新”。主数值、数据时间和刷新动作应在第一屏内找到。

逐项执行删除检查。

1. 删除该元素后，用户是否少知道一个不同的事实。
2. 该信息是否已经在同一屏出现。
3. 该动作是否会被用户执行。
4. 标题是否说明内容，按钮是否说明结果。

主卡片已经显示“已用流量”时，详情列表不再重复该行。数据来自缓存时，页面直接写“缓存数据”和缓存日期。不要只改变颜色。

页面只保留一个明确的一级标题。分区使用有层级的标题，并通过 `aria-labelledby` 关联内容。按钮使用动词和对象，例如“刷新数据”“打开运营商网页”“更新 Cookie”。

## 适配尺寸和外观

使用以下 viewport，保留用户缩放。

```html
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
```

`viewport-fit=cover` 会让页面铺到屏幕边缘，页面必须再用 `env(safe-area-inset-*)` 保护重要内容。主体设置最大宽度，并让窄屏使用完整可用宽度。

通过 `color-scheme: light dark` 和 `prefers-color-scheme` 提供浅色与深色值。相邻表面必须保持可见差异。深色页面背景和次级按钮不能使用同一个颜色值。

使用系统字体和相对稳定的字号层级。长卡号、日期、错误信息和本地化文本需要换行或安全截断。不要禁止横屏，也不要依赖固定屏幕宽度。

支持 `prefers-reduced-motion`，停止非必要的旋转、位移和缩放动画。加载状态仍应保留文字。

## 处理全屏安全区和宿主控件

把 WebView 顶部避让分成三个独立部分。

1. **设备安全区**：刘海、Dynamic Island、圆角和状态栏，由 `env(safe-area-inset-top)` 表示。
2. **Scriptable 宿主控件**：全屏 WebView 上方的 Close、Share 等原生控件。它们不属于页面 DOM，不能用 `z-index` 移动，也不能假定已经包含在 CSS 安全区变量内。
3. **页面视觉留白**：标题与遮挡边界之间仍需保留的普通间距。

[WebKit 的说明](https://webkit.org/blog/7929/designing-websites-for-iphone-x/)把 `safe-area-inset-*` 定义为设备屏幕安全区。[Scriptable 的 `present(fullscreen)` 文档](https://docs.scriptable.app/webview/#-present)只说明 `true` 会全屏展示，没有提供 Close、Share 控件的 frame、高度或 CSS inset。因此，不要把某个宿主控件高度写成适用于所有设备和 Scriptable 版本的常量。

`max(18px, env(safe-area-inset-top))` 只是在“普通页边距”和“设备安全区”中取较大值。当宿主控件也覆盖页面时，这几层需要相加，而不是继续取最大值。自包含全屏页面可以从以下结构开始：

```html
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<body class="scriptable-fullscreen">
  <main class="page">...</main>
</body>
```

```css
:root {
  --page-inline: 16px;
  --page-top-gap: 12px;
  --page-bottom: 30px;
  --scriptable-host-top: 0px;
}

*,
*::before,
*::after {
  box-sizing: border-box;
}

body.scriptable-fullscreen {
  /*
   * 56px 只是保守的初始校准值，不是 Scriptable API 保证值。
   * 必须根据目标设备、系统版本和 Scriptable 版本的真机截图调整。
   */
  --scriptable-host-top: 56px;
}

html,
body {
  margin: 0;
  min-height: 100%;
}

body {
  --protected-top: calc(
    env(safe-area-inset-top, 0px)
    + var(--scriptable-host-top)
  );
  padding: 18px 16px 30px;
  padding-top: calc(var(--protected-top) + var(--page-top-gap));
  padding-right: max(env(safe-area-inset-right, 0px), var(--page-inline));
  padding-bottom: max(env(safe-area-inset-bottom, 0px), var(--page-bottom));
  padding-left: max(env(safe-area-inset-left, 0px), var(--page-inline));
}

.page {
  width: min(100%, 400px);
  margin-inline: auto;
}

.sticky-header {
  position: sticky;
  top: var(--protected-top);
}
```

只在实际调用 `present(true)` 且宿主控件覆盖内容的页面启用 `scriptable-fullscreen`。非全屏展示或已经由容器留出顶部空间时把 `--scriptable-host-top` 保持为 `0px`，避免无意义的大块空白。若实测不同设备需要不同值，优先选择一个能覆盖支持范围的保守值；只有确有稳定依据时才按媒体查询分档。

页面首屏避让正确后，还要单独处理会脱离普通文档流的元素。

- `position: fixed` 或 `position: sticky` 的顶部栏也要把设备安全区和宿主控件避让加入 `top`；只给 `body` 加 padding 不能阻止它滚动后贴到 Close 或 Share 下方。
- 底部悬浮按钮、Toast 和操作栏使用 `max(普通间距, env(safe-area-inset-bottom))`，避免压住 Home 指示条。
- 全屏加载遮罩可以覆盖背景，但遮罩内的关闭、取消和错误操作仍应位于受保护区域。
- Close 和 Share 位于原生宿主层时，DOM、`visualViewport` 和 CSS 都不能可靠测出它们的实际 frame。计算样式只能辅助排查，最终以真机截图和点击测试为准。

自包含页面直接修改生成的 HTML/CSS。自己控制的远程页面应在页面源代码中处理安全区。第三方远程页面不要盲目注入通用 `body` padding，这可能破坏它自己的固定导航；先验证 `present(false)` 是否更合适，确需注入时要在每次导航和重载后重新验证。

## 提供可访问界面

优先使用原生 HTML 元素。使用 `<button>` 后，浏览器会提供点击、键盘操作、禁用状态和角色。只有没有合适元素时才使用 `role` 和 `tabindex` 模拟控件。

进度条至少提供名称、最小值、最大值和当前值。

```html
<div
  role="progressbar"
  aria-label="某物的使用比例"
  aria-valuemin="0"
  aria-valuemax="100"
  aria-valuenow="37"
></div>
```

加载状态使用 `role="status"`、`aria-live="polite"` 和可见文字。需要立即通知的错误使用 `role="alert"`。页面忙碌时在主区域设置 `aria-busy="true"`。

切换显示隐私信息的按钮更新 `aria-pressed` 和可访问名称，例如“显示完整卡号”和“隐藏完整卡号”。

为键盘焦点提供明显的 `:focus-visible` 样式。不要设置 `maximum-scale=1` 或 `user-scalable=no`。W3C WCAG 要求支持文字缩放、回流、键盘操作、可见焦点、名称与角色以及状态消息。

## 统一使用 iOS 式加载指示器

同一 WebView 只保留一种不确定进度指示器。需要接近 iOS 活动指示器的外观时，统一复用八段式 `.ios-spinner`；不要再混用边框圆环 `.spinner`、GIF 或另一套 keyframes。

每个出现位置可以有自己的 spinner 元素，但必须共用同一份结构和 CSS。spinner 仅作装饰，使用 `aria-hidden="true"`；按钮文字或 `role="status"` 的可见文字负责说明“正在刷新”“正在验证”等具体动作。

```html
<span class="ios-spinner" aria-hidden="true">
  <span></span><span></span><span></span><span></span>
  <span></span><span></span><span></span><span></span>
</span>
```

```css
.ios-spinner {
  display: none;
  position: relative;
  width: 18px;
  height: 18px;
  flex: none;
}

.ios-spinner span {
  position: absolute;
  left: 8px;
  top: 1px;
  width: 2px;
  height: 5px;
  border-radius: 999px;
  background: currentColor;
  transform-origin: 1px 8px;
  opacity: 0.15;
  animation: ios-spinner-fade 0.8s linear infinite;
}

.ios-spinner span:nth-child(1) { transform: rotate(0deg);   animation-delay: -0.7s; }
.ios-spinner span:nth-child(2) { transform: rotate(45deg);  animation-delay: -0.6s; }
.ios-spinner span:nth-child(3) { transform: rotate(90deg);  animation-delay: -0.5s; }
.ios-spinner span:nth-child(4) { transform: rotate(135deg); animation-delay: -0.4s; }
.ios-spinner span:nth-child(5) { transform: rotate(180deg); animation-delay: -0.3s; }
.ios-spinner span:nth-child(6) { transform: rotate(225deg); animation-delay: -0.2s; }
.ios-spinner span:nth-child(7) { transform: rotate(270deg); animation-delay: -0.1s; }
.ios-spinner span:nth-child(8) { transform: rotate(315deg); animation-delay: 0s; }

@keyframes ios-spinner-fade {
  0% { opacity: 1; }
  100% { opacity: 0.15; }
}

.refreshing .ios-spinner,
.busy-layer.show .ios-spinner {
  display: inline-block;
}
```

默认隐藏 spinner，并为每个实际状态写出显式显示选择器。只切换 `.busy-layer.show` 而没有 `.busy-layer.show .ios-spinner` 时，祖先虽然出现，spinner 仍会保持 `display: none`。修改状态类名时同步检查 spinner 的选择器、可见文字、`aria-busy` 和 `aria-hidden`。

使用 `currentColor` 让八段颜色跟随按钮或 busy 卡片文字，不再维护另一套浅色、深色 spinner 颜色。启用 `prefers-reduced-motion` 时可以停止动画，但必须保留“正在……”文字；不能靠旋转本身表达加载状态。

## 处理加载和错误状态

页面至少处理以下状态。

| 状态 | 页面行为 |
| --- | --- |
| 初始加载 | 首次 HTML 已有可读内容，不显示无法结束的空骨架 |
| 单个动作执行中 | 禁用相关按钮，按钮内显示动作文字 |
| 全页动作执行中 | 显示带文字的忙碌层，设置 `aria-busy` |
| 动作失败 | 恢复按钮，显示具体错误，不丢失现有数据 |
| 使用缓存 | 标出缓存并显示可比较的日期 |
| 页面关闭 | 停止事件桥和后续页面更新 |
| 页面重载 | 重新创建页面端状态并注册新的等待 |

错误文字说明失败的动作和用户可以采取的下一步。不要把完整响应、Cookie、认证头或内部堆栈显示在页面中。

## 控制性能和复杂度

- 目标版本支持时使用等待式事件桥。兼容轮询使用 400 到 750 毫秒间隔，并随页面关闭停止。
- 把网络和文件工作留在主脚本中，页面不重复请求相同数据。
- 重载整页前判断局部 DOM 更新是否足够。结构和数据同时变化时，整页重载更容易保持一致。
- 避免大型 Base64 图片、未使用的 CSS、重复图标和长响应对象。
- 用 CSS 动画实现短视觉反馈，并遵守 reduced motion。
- 删除未调用的页面函数、没有读取的状态和为了一个值建立的抽象。

## 执行验证矩阵

至少验证以下组合。

- 浅色、深色、较大文字和页面缩放。
- 窄屏、宽屏、竖屏、横屏和安全区。
- `present(true)` 与实际采用的非全屏模式；首次打开、滚动到顶部和旋转后，Close、Share、一级标题及首个操作均不重叠。
- 有刘海或 Dynamic Island 的 iPhone、无刘海设备以及支持范围内的 iPad；宿主控件校准值不能只来自浏览器模拟。
- VoiceOver 阅读顺序、按钮名称、进度值和状态消息。
- 键盘 Tab、Enter、空格、可见焦点和关闭页面。
- 正常数据、长文本、零值、无效值、缓存数据和跨日缓存。
- 刷新成功、刷新失败、连续点击、页面重载和操作中关闭页面。
- 未知 action、被拒绝导航、远程跳转和资源主机缺失。
- 内容安全策略和请求过滤分别启用后的首次展示、重载、样式、脚本和图片。

普通浏览器可以检查 HTML、CSS、DOM 事件和基本可访问性。`WebView`、`completion()`、`shouldAllowRequest`、系统关闭行为和 Scriptable 权限必须在目标 iPhone 或 iPad 上验证。

## 审查清单

- [ ] 自包含主界面和远程网页使用不同 WebView。
- [ ] 页面只接收显示所需字段，不包含秘密。
- [ ] HTML 文本与属性经过编码，页面 JavaScript 参数经过 JSON 编码。
- [ ] 自包含页面没有外部导航；启用请求过滤或内容安全策略时已通过真机首次展示和重载验证。
- [ ] 远程页面限制协议、主机、资源和跳转。
- [ ] action 使用固定集合，主脚本再次校验。
- [ ] 事件桥已在目标版本验证；兼容轮询具有停止条件且间隔不低于 400 毫秒。
- [ ] 页面关闭、桥接错误和页面重载有明确处理。
- [ ] 主数值、数据时间和主要动作位于第一屏。
- [ ] 同一事实没有在主卡片和详情列表中重复。
- [ ] viewport 允许缩放，布局处理安全区和横竖屏。
- [ ] 全屏页面分别叠加设备安全区、宿主控件避让和页面留白，没有把 `safe-area-inset-top` 当作 Close、Share 的高度。
- [ ] 顶部 fixed/sticky 元素不会在滚动后进入宿主控件区域；底部悬浮元素避开 Home 指示条。
- [ ] 宿主控件避让值已在目标 Scriptable 版本和真机上校准，不被描述为通用常量。
- [ ] 浅色与深色中的页面、卡片和按钮边界可见。
- [ ] 原生按钮、进度语义、状态消息、焦点和 reduced motion 已处理。
- [ ] 页面只定义一种 spinner；每个 loading/busy 状态都有匹配的 `.ios-spinner` 显示选择器和可见状态文字。
- [ ] 连续点击、动作失败和操作中关闭页面经过真机验证。
