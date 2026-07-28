---
name: scriptable
description: 编写、审查、调试和解释 Scriptable 的 JavaScript 自动化脚本。覆盖依据 Apple Human Interface Guidelines 设计的桌面与锁屏组件、WebView 主界面与事件桥，以及快捷指令、共享表单、URL Scheme、通知、网络请求、文件、钥匙串、日历、提醒事项、通讯录、定位、照片、表格、绘图和 XML；在设计或审查组件与 WebView、处理刷新与缓存、选择 Scriptable API、核对签名、处理运行环境差异或排查权限与界面问题时使用。
---

# Scriptable 开发

## 先读取对应参考

设计或审查桌面组件、锁屏组件和 WebView 界面时，先完整读取 [references/apple-ui-design.md](references/apple-ui-design.md)，把 Apple HIG 转换为 Scriptable 可以执行和验证的规则。

处理桌面组件、锁屏组件或组件附带的详情页时，同时完整读取 [references/widgets.md](references/widgets.md)。

使用 WebView 构建主界面、详情页、交互式工具或 JavaScript 事件桥时，同时完整读取 [references/webview-main-interface.md](references/webview-main-interface.md)。组件点击后打开 WebView 时，读取以上三份参考。

需要核对类名、成员、参数、返回类型、废弃接口或运行限制时，读取 [references/api-reference.md](references/api-reference.md)。该文件超过 2000 行，先用以下模式定位，再读取相关小节。

```bash
rg -n '^### \[?(ListWidget|WidgetStack|WidgetText|Request|WebView|FileManager|Keychain|Script|config|args)' references/api-reference.md
```

遇到文档和目标设备行为不一致时，核对当前 [Scriptable 官方文档](https://docs.scriptable.app/)，并以目标设备实测为准。

## 按运行环境确定输入和输出

1. 读取 `config`，确定脚本从应用、共享表单、快捷指令、Siri、组件、通知、主屏幕或 URL Scheme 运行。
2. 从对应的 `args` 属性读取输入。处理组件时同时检查 `args.widgetParameter`、`config.widgetFamily` 和 `config.runsInAccessoryWidget`。
3. 明确输出目标。组件交给 `Script.setWidget()`，快捷指令交给 `Script.setShortcutOutput()`，应用内界面等待展示方法结束。
4. 把数据获取、数据规范化、持久化、布局和界面副作用拆成独立函数。
5. 在所有异步工作结束后调用一次 `Script.complete()`。使用顶层 `try`、`catch` 和 `finally` 保证错误保留上下文。

`config.runsInWidget` 对桌面组件和锁屏组件都为真。需要区分锁屏组件时，读取 `config.runsInAccessoryWidget` 或 `config.widgetFamily`。

Scriptable 使用 Apple JavaScriptCore。不要在主脚本中使用 DOM、浏览器 `fetch()`、Node.js 模块或未确认可用的 JavaScript 新语法。DOM 只存在于 `WebView` 加载的页面中。

## 保持数据路径可验证

把主流程组织成以下步骤。

```javascript
async function main() {
  const context = readRunContext()
  const settings = readAndValidateSettings()
  const result = await loadData(settings, context)
  const view = buildOutput(result, context)
  await presentOutput(view, context)
}

try {
  await main()
} catch (error) {
  console.error(error && error.stack ? error.stack : String(error))
  throw error
} finally {
  Script.complete()
}
```

遵守以下边界。

- 让网络函数返回响应或带类型的错误，不在网络函数中显示对话框。
- 让规范化函数拒绝缺失、空字符串、非有限数和越界值，不把无效值静默转成零。
- 让渲染函数只接收已经规范化的数据，不读取钥匙串、不发请求、不写文件。
- 让缓存写入失败只影响缓存。已经成功取得的网络数据仍应返回。
- 只在服务明确返回认证失败时要求更新凭据。超时、离线、服务错误和 JSON 错误应保留原凭据。
- 先验证新凭据，再替换钥匙串中的旧值。不要在日志、组件参数、URL 或错误正文中输出凭据。

## 处理网络、缓存和文件

为每个 `Request` 设置合理的超时，检查 `response.statusCode`，并使用与响应类型一致的 `loadJSON()`、`loadString()`、`loadImage()` 或 `load()`。

优先使用 HTTPS。只有用户接受明文传输风险且服务没有 HTTPS 时，才设置 `allowInsecureRequest`。Cookie、令牌或个人数据通过 HTTP 发送时，必须在审查结果中指出。

缓存至少保存以下字段。

```javascript
{
  version: 1,
  savedAt: "2026-07-28T12:00:00.000Z",
  data: {}
}
```

读取缓存时校验版本、时间、结构和业务字段。区分新鲜缓存、过期缓存和不可用缓存。界面使用过期缓存时，显示“缓存”或数据日期，不能只显示可能与当前日期混淆的时分。

使用创建路径的同一个 `FileManager` 实例读写文件。读取 iCloud 文件前等待下载。把凭据放进 `Keychain`，把可重建数据放进 `cacheDirectory()`，把需要跨清理周期保留的应用数据放进 `libraryDirectory()` 或 `documentsDirectory()`。

## 处理 WebView

先按 [references/webview-main-interface.md](references/webview-main-interface.md) 选择页面类型、数据边界、导航规则和事件桥。

只在 `WebView` 页面内使用 DOM。把外部值放入 HTML 文本和属性前进行 HTML 编码，把外部值放入页面 JavaScript 前使用 `JSON.stringify()`。

远程页面使用 `shouldAllowRequest` 限制主机和 URL Scheme。自包含页面只有在真机确认内部主文档 URL 后才设置请求过滤，不能让过滤规则拦截 `loadHTML()`。页面动作需要回传主脚本时，先验证等待式事件桥在目标 Scriptable 版本中的展示、重载和关闭行为。出现空白页或长期回调失效时，使用有停止条件的低频轮询。

为详情页保留系统文字缩放和页面缩放。给进度、加载状态、错误提示和可点击非按钮元素添加可访问名称、状态和键盘操作。支持 `prefers-reduced-motion`，并在浅色和深色外观中检查对比度。

## 审查时先报告会改变结果的问题

按以下顺序报告发现，并给出函数名、行号、触发条件和后果。

1. 凭据泄露、明文传输、任意导航、破坏性操作和外部输入注入。
2. 错误数据、日期边界、单位误判、认证状态误判和缓存污染。
3. 组件空白、尺寸不支持、刷新语义错误、过期数据未标记和交互失效。
4. 可访问性、文字截断、深浅色或锁屏渲染问题。
5. 重复布局、未使用代码、轮询、重复请求和维护成本。

不要只写“可优化”或“建议重构”。每项说明当前行为、可复现条件、用户能看到的后果和最小修改方向。

## 完成验证

至少完成以下检查。

1. 对 JavaScript 做语法检查。普通 Node.js 只能验证语法和纯函数，不能验证 Scriptable 全局 API。
2. 为纯函数覆盖缺失值、零值、极大值、日期边界、时区、过期缓存和错误响应。
3. 核对所有 `Promise` 调用是否在依赖结果的位置使用 `await`。
4. 核对日志、URL、组件参数和缓存是否包含秘密。
5. 组件任务按 [references/widgets.md](references/widgets.md) 的预览矩阵在 Scriptable 应用和真实桌面或锁屏上验证。
6. 说明仍需在 iPhone 或 iPad 上验证的权限、系统界面、内存、刷新和布局行为。
