# Scriptable WebView 主界面最佳实践

## 目录

- [选择页面类型](#选择页面类型)
- [保持单向数据流](#保持单向数据流)
- [构建安全的 HTML](#构建安全的-html)
- [限制导航和资源请求](#限制导航和资源请求)
- [使用等待式事件桥](#使用等待式事件桥)
- [管理页面生命周期](#管理页面生命周期)
- [安排主界面内容](#安排主界面内容)
- [适配尺寸和外观](#适配尺寸和外观)
- [提供可访问界面](#提供可访问界面)
- [处理加载和错误状态](#处理加载和错误状态)
- [控制性能和复杂度](#控制性能和复杂度)
- [执行验证矩阵](#执行验证矩阵)
- [审查清单](#审查清单)

## 选择页面类型

先确定 WebView 展示哪一种内容。

| 页面类型 | 加载方式 | 请求规则 |
| --- | --- | --- |
| 脚本生成的自包含主界面 | `loadHTML()` | 默认不访问网络，只允许页面运行所需的内部 URL |
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

自包含页面可以加入内容安全策略。根据页面实际资源调整允许项。

```html
<meta
  http-equiv="Content-Security-Policy"
  content="default-src 'none'; img-src data:; style-src 'unsafe-inline'; script-src 'unsafe-inline'; base-uri 'none'; form-action 'none'"
>
```

内容安全策略不能替代输出编码和 `shouldAllowRequest`。三项规则处理的失败位置不同。

## 限制导航和资源请求

Scriptable 的 `WebView.shouldAllowRequest` 默认允许所有请求。自包含页面只允许实际需要的内部 URL。

```javascript
function allowSelfContainedRequest(request) {
  const url = request && request.url ? String(request.url) : ""
  return url === "" ||
    url === "about:blank" ||
    url.indexOf("data:") === 0 ||
    url.indexOf("blob:") === 0
}

const webView = new WebView()
webView.shouldAllowRequest = allowSelfContainedRequest
```

远程页面应解析并比较协议和主机。页面需要 CDN、认证跳转或图片主机时，逐个加入允许列表，并在真机检查被拒绝请求。不要为了修复缺失资源而允许所有 HTTP、HTTPS 或自定义 Scheme。

`loadRequest()` 会使用设置的方法、正文和请求头，但不会调用该 `Request` 的 `onRedirect`。需要限制 WebView 后续请求时，仍要设置 `shouldAllowRequest`。

凭据只发送给预期主机。远程页面跳转到未允许主机时停止导航，并向用户显示可以理解的错误。

## 使用等待式事件桥

Scriptable 官方接口支持 `evaluateJavaScript(code, true)`。该 Promise 会等待页面调用全局 `completion(value)`。使用一次等待处理一个 action，避免每隔几百毫秒查询 DOM 或全局变量。

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

用 `env(safe-area-inset-top)` 和 `env(safe-area-inset-bottom)` 处理全屏安全区。主体设置最大宽度，并让窄屏使用完整可用宽度。

通过 `color-scheme: light dark` 和 `prefers-color-scheme` 提供浅色与深色值。相邻表面必须保持可见差异。深色页面背景和次级按钮不能使用同一个颜色值。

使用系统字体和相对稳定的字号层级。长卡号、日期、错误信息和本地化文本需要换行或安全截断。不要禁止横屏，也不要依赖固定屏幕宽度。

支持 `prefers-reduced-motion`，停止非必要的旋转、位移和缩放动画。加载状态仍应保留文字。

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

- 使用事件桥等待点击，不运行常驻短轮询。
- 把网络和文件工作留在主脚本中，页面不重复请求相同数据。
- 重载整页前判断局部 DOM 更新是否足够。结构和数据同时变化时，整页重载更容易保持一致。
- 避免大型 Base64 图片、未使用的 CSS、重复图标和长响应对象。
- 用 CSS 动画实现短视觉反馈，并遵守 reduced motion。
- 删除未调用的页面函数、没有读取的状态和为了一个值建立的抽象。

## 执行验证矩阵

至少验证以下组合。

- 浅色、深色、较大文字和页面缩放。
- 窄屏、宽屏、竖屏、横屏和安全区。
- VoiceOver 阅读顺序、按钮名称、进度值和状态消息。
- 键盘 Tab、Enter、空格、可见焦点和关闭页面。
- 正常数据、长文本、零值、无效值、缓存数据和跨日缓存。
- 刷新成功、刷新失败、连续点击、页面重载和操作中关闭页面。
- 未知 action、被拒绝导航、远程跳转和资源主机缺失。
- 内容安全策略启用后的样式、脚本和图片。

普通浏览器可以检查 HTML、CSS、DOM 事件和基本可访问性。`WebView`、`completion()`、`shouldAllowRequest`、系统关闭行为和 Scriptable 权限必须在目标 iPhone 或 iPad 上验证。

## 审查清单

- [ ] 自包含主界面和远程网页使用不同 WebView。
- [ ] 页面只接收显示所需字段，不包含秘密。
- [ ] HTML 文本与属性经过编码，页面 JavaScript 参数经过 JSON 编码。
- [ ] 自包含页面限制资源请求并设置合适的内容安全策略。
- [ ] 远程页面限制协议、主机、资源和跳转。
- [ ] action 使用固定集合，主脚本再次校验。
- [ ] 事件桥使用等待式 callback，没有短周期轮询。
- [ ] 页面关闭、桥接错误和页面重载有明确处理。
- [ ] 主数值、数据时间和主要动作位于第一屏。
- [ ] 同一事实没有在主卡片和详情列表中重复。
- [ ] viewport 允许缩放，布局处理安全区和横竖屏。
- [ ] 浅色与深色中的页面、卡片和按钮边界可见。
- [ ] 原生按钮、进度语义、状态消息、焦点和 reduced motion 已处理。
- [ ] 连续点击、动作失败和操作中关闭页面经过真机验证。
