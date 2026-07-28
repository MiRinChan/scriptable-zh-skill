# Scriptable API 参考

<a id="scope"></a>

## 文档范围

本文根据 [Scriptable 官方文档](https://docs.scriptable.app/) 的 sitemap 和全文索引整理。抓取日期为 2026-07-28，sitemap 中 61 个页面的 `lastmod` 均为 2024-11-27。正文覆盖首页说明和 60 个 API 页面。

Scriptable 使用 Apple JavaScriptCore，并记录为支持 ECMAScript 6。脚本不在浏览器中运行，因此不能假定存在 `document`、`window`、DOM、浏览器 `fetch()` 或 Node.js 模块。网络、文件、界面和系统能力应使用本文列出的 Scriptable API。

本文保存官方公开签名，并把限制条件改写为操作规则。目标设备上的 Scriptable 版本可能晚于文档快照。遇到签名或行为差异时，以当前官方页面和设备实测为准。

<a id="contents"></a>

## 锚点目录

- [使用流程](#workflow)
- [运行模型](#runtime-model)
- [按任务选择 API](#api-selection)
- [基础脚本结构](#script-structure)
- [运行环境、输入与输出](#contexts)
- [异步、错误和完成状态](#async-errors)
- [网络请求、数据和缓存](#network-data-cache)
- [文件、iCloud 和钥匙串](#files-keychain)
- [桌面与锁屏组件](#widgets)
- [对话框、表格和系统界面](#interfaces)
- [日历、提醒事项、通讯录、定位和照片](#personal-data)
- [通知、快捷指令和应用间调用](#automation)
- [模块、WebView、绘图和 XML](#advanced)
- [权限与安全](#security)
- [调试与验证](#debugging)
- [已废弃接口](#deprecated)
- [完整 API 索引](#api-reference)
- [覆盖核验](#coverage)

<a id="workflow"></a>

## 使用流程

处理 Scriptable 请求时，按下面的顺序执行。

1. 确定脚本从应用、共享表单、快捷指令、Siri、组件、通知或 URL Scheme 中的哪一种环境运行。
2. 确定输入来自哪个 `args` 属性，输出交给界面、`Script.setWidget()`、`Script.setShortcutOutput()`、通知、文件或其他应用中的哪一个目标。
3. 从[按任务选择 API](#api-selection)和[完整 API 索引](#api-reference)核对类名、方法名、参数和返回类型。
4. 标出所有返回 `Promise` 的调用，并在依赖结果的位置使用 `await`。
5. 检查权限、iCloud 下载状态、组件尺寸、运行环境限制和删除操作。
6. 编写一个入口函数，把数据获取、转换、渲染和副作用分开。
7. 对代码做静态检查，并说明仍需在 iPhone 或 iPad 上验证的权限、系统界面和组件行为。

向人类解释代码时，先说明脚本读取什么、改变什么、在哪里显示结果，再解释关键 API。向 agent 提供任务时，应给出运行环境、预期输入、预期输出、最低系统版本、是否允许网络访问和是否允许修改个人数据。

<a id="runtime-model"></a>

## 运行模型

Scriptable 提供全局类、全局对象和全局函数。`new Request()`、`new ListWidget()` 和 `new Alert()` 创建实例；`Device.systemVersion()`、`Keychain.get()` 和 `Photos.latestPhoto()` 是静态调用；`args`、`config` 和 `module` 是全局对象；`importModule()` 是全局函数。

官方签名使用以下记法。

| 记法 | 含义 |
| --- | --- |
| `bool` | 布尔值 |
| `[Type]` | `Type` 数组 |
| `{string: any}` | 字符串键的对象 |
| `fn()` | 回调函数 |
| `Promise<Type>` | 异步结果，需要 `await` |
| `static method()` | 直接在类上调用 |
| `property: Type` | 实例或全局对象属性 |

组件和扩展进程受系统时间与内存限制。脚本不能控制组件准确刷新时间，也不能假定共享扩展、Siri 和主应用共享临时目录或缓存目录。系统权限请求、邮件编辑器、信息编辑器、相册选择器和文件选择器只能在 Apple 设备上验证。

<a id="api-selection"></a>

## 按任务选择 API

| 任务 | 首选 API |
| --- | --- |
| 发送 HTTP 请求 | `Request` |
| 保存 JSON 或图片 | `FileManager`、`Data`、`Image` |
| 保存令牌或密码 | `Keychain` |
| 读取快捷指令、组件或 URL 参数 | `args` |
| 判断当前运行环境 | `config` |
| 输出快捷指令结果 | `Script.setShortcutOutput()` |
| 构建桌面或锁屏组件 | `ListWidget` 和 `Widget...` 类型 |
| 显示确认、输入或日期选择 | `Alert`、`TextField`、`DatePicker` |
| 显示结构化列表 | `UITable`、`UITableRow`、`UITableCell` |
| 加载网页或执行页面 JavaScript | `WebView` |
| 打开网页或其他应用 | `Safari`、`URLScheme`、`CallbackURL` |
| 读写日历事件或提醒事项 | `Calendar`、`CalendarEvent`、`Reminder` |
| 读写联系人 | `Contact`、`ContactsContainer`、`ContactsGroup` |
| 获取位置或照片 | `Location`、`Photos` |
| 调度本地通知 | `Notification` |
| 绘制图片 | `DrawContext`、`Path`、`Point`、`Rect`、`Size` |
| 拆分代码 | `importModule()`、`module.exports` |
| 解析 XML | `XMLParser` |

<a id="script-structure"></a>

## 基础脚本结构

把副作用放进明确的函数。脚本入口只负责读取运行环境、调用函数、输出结果和完成脚本。

```javascript
async function main() {
  const input = args.shortcutParameter != null
    ? args.shortcutParameter
    : args.widgetParameter
  const result = await buildResult(input)

  if (config.runsInWidget) {
    const widget = await buildWidget(result)
    Script.setWidget(widget)
    return
  }

  if (config.runsInApp) {
    await QuickLook.present(JSON.stringify(result, null, 2))
    return
  }

  Script.setShortcutOutput(result)
}

async function buildResult(input) {
  return {
    input,
    generatedAt: new Date().toISOString()
  }
}

async function buildWidget(result) {
  const widget = new ListWidget()
  const text = result.input != null ? result.input : "No input"
  widget.addText(String(text))
  return widget
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

不要在数据函数中直接显示界面，也不要在渲染函数中读取钥匙串或修改日历。分开这些步骤后，同一份数据逻辑可以供应用预览、快捷指令和组件使用。

<a id="contexts"></a>

## 运行环境、输入与输出

先读取 `config`，再选择输入和输出。

| 环境 | 判断方式 | 常用输入 | 常用输出 |
| --- | --- | --- | --- |
| Scriptable 应用 | `config.runsInApp` | 脚本内配置、对话框 | `QuickLook`、`UITable`、`WebView` |
| 共享表单 | `config.runsInActionExtension` | `args.plainTexts`、`urls`、`fileURLs`、`images` | 系统界面、文件、共享表单 |
| Siri | `config.runsWithSiri` | `args.shortcutParameter` | `Speech`、`Script.setShortcutOutput()` |
| 桌面或锁屏组件 | `config.runsInWidget` | `args.widgetParameter` | `Script.setWidget()` |
| 锁屏组件 | `config.runsInAccessoryWidget` | `args.widgetParameter` | `Script.setWidget()` |
| 通知动作 | `config.runsInNotification` | `args.notification` | 系统界面、文件、网络调用 |
| 主屏幕 | `config.runsFromHomeScreen` | `args.queryParameters` 或脚本配置 | 由脚本决定 |
| URL Scheme | 结合运行方式判断 | `args.queryParameters` | 由脚本决定 |

共享表单可能同时提供多类输入。任务只需要一种类型时，应检查数组长度并拒绝含糊输入。组件参数由用户在组件配置中填写，脚本必须处理 `null`、空字符串和格式错误。

组件尺寸由 `config.widgetFamily` 给出。对每一种实际支持的尺寸写显式分支。未支持的尺寸应显示短错误消息，不能返回空白组件。

<a id="async-errors"></a>

## 异步、错误和完成状态

所有 `Promise` 调用都应等待结果。常见异步调用包括网络读取、系统选择器、位置、日历、提醒事项、通知、WebView 和组件预览。

捕获错误时保留原始错误和调用上下文。日志中应出现失败的操作、URL 的主机名、文件路径或目标 API。不要记录访问令牌、Cookie、联系人内容和完整响应正文。

```javascript
async function withContext(label, operation) {
  try {
    return await operation()
  } catch (error) {
    const detail = error && error.stack ? error.stack : String(error)
    throw new Error(`${label}: ${detail}`)
  }
}

const value = await withContext(
  "读取日历事件",
  () => CalendarEvent.today()
)
```

系统选择器和权限请求可能被用户取消。对返回 `null`、空数组或被拒绝的 Promise 分别处理。删除、发送、调度和持久化操作在重试前应确认当前状态，避免重复创建数据。

<a id="network-data-cache"></a>

## 网络请求、数据和缓存

创建 `Request` 后设置方法、请求头、正文和超时，再选择与响应类型一致的加载方法。

```javascript
async function loadJSON(url, token, serviceName) {
  const request = new Request(url)
  request.method = "GET"
  request.timeoutInterval = 20
  request.headers = {
    Accept: "application/json",
    Authorization: `Bearer ${token}`
  }

  const value = await request.loadJSON()
  if (!request.response) {
    throw new Error(`No HTTP response from ${serviceName}`)
  }
  const status = request.response.statusCode
  if (status < 200 || status >= 300) {
    throw new Error(`HTTP ${status} from ${serviceName}`)
  }
  return value
}
```

JSON 缓存应同时保存数据和写入时间。缓存失败不能覆盖有效结果；网络失败时只有在业务允许的情况下才返回过期缓存。

```javascript
function createJSONCache(filename) {
  const fm = FileManager.local()
  const path = fm.joinPath(fm.cacheDirectory(), filename)

  return {
    read(maxAgeMs) {
      if (!fm.fileExists(path)) return null
      const modified = fm.modificationDate(path)
      if (!modified || Date.now() - modified.getTime() > maxAgeMs) return null
      try {
        return JSON.parse(fm.readString(path))
      } catch (error) {
        return null
      }
    },
    write(value) {
      fm.writeString(path, JSON.stringify(value))
    }
  }
}
```

响应为图片时使用 `loadImage()`；需要原始字节时使用 `load()`；服务返回文本时使用 `loadString()`。只有在服务明确返回 JSON 时使用 `loadJSON()`。

<a id="files-keychain"></a>

## 文件、iCloud 和钥匙串

`FileManager.local()` 和 `FileManager.iCloud()` 指向不同存储。先选定实例，再用该实例生成目录和路径。

```javascript
const fm = FileManager.iCloud()
const path = fm.joinPath(fm.documentsDirectory(), "settings.json")

if (fm.fileExists(path) && !fm.isFileDownloaded(path)) {
  await fm.downloadFileFromiCloud(path)
}

const settings = fm.fileExists(path)
  ? JSON.parse(fm.readString(path))
  : {}
```

用户可编辑的文件放在 `documentsDirectory()`。无需在文件应用中显示的应用数据可以放在 `libraryDirectory()`。可重建的数据放在 `cacheDirectory()`。只在当前执行期间使用的文件放在 `temporaryDirectory()`。

凭据使用 `Keychain`。键不存在时，`Keychain.get()` 会抛出错误。

```javascript
const key = "example.api-token"
if (!Keychain.contains(key)) {
  throw new Error(`缺少钥匙串项目 ${key}`)
}
const token = Keychain.get(key)
```

删除文件前确认目标路径由当前脚本生成，并避免把目录根路径、用户选择的文件夹或未经验证的参数传给 `remove()`。

<a id="widgets"></a>

## 桌面与锁屏组件

组件脚本分为数据、布局和输出三步。数据获取应设置短超时并使用缓存。布局必须处理当前 `widgetFamily`。输出时，在组件环境调用 `Script.setWidget()`，在应用中调用对应预览方法。

```javascript
async function buildStatusWidget(status) {
  const widget = new ListWidget()
  widget.backgroundColor = Color.dynamic(
    new Color("#F2F2F7"),
    new Color("#1C1C1E")
  )
  widget.setPadding(14, 14, 14, 14)

  const title = widget.addText(status.title)
  title.font = Font.semiboldSystemFont(15)
  title.textColor = Color.dynamic(Color.black(), Color.white())

  widget.addSpacer(6)

  const detail = widget.addText(status.detail)
  detail.font = Font.systemFont(12)
  detail.textColor = Color.dynamic(Color.darkGray(), Color.lightGray())
  detail.lineLimit = 3

  widget.refreshAfterDate = new Date(Date.now() + 30 * 60 * 1000)
  return widget
}

const widget = await buildStatusWidget({
  title: "Service",
  detail: "Available"
})

if (config.runsInWidget) {
  Script.setWidget(widget)
} else {
  const family = config.widgetFamily || "medium"
  if (family === "small") await widget.presentSmall()
  else if (family === "large") await widget.presentLarge()
  else await widget.presentMedium()
}
Script.complete()
```

组件刷新时间由系统决定。`refreshAfterDate` 表示系统可以开始刷新组件的最早时间，不保证准时执行。需要及时数据时，显示数据时间，并允许用户点击组件打开脚本刷新。

小型组件只有一个点击目标，应设置 `ListWidget.url`。中型和大型组件可以给文字、日期、图片和 stack 设置各自的 `url`。组件根 `url` 会覆盖组件配置中的交互行为。

锁屏组件应按 `accessoryInline`、`accessoryCircular` 和 `accessoryRectangular` 分别设计。inline 组件只显示一个图片和一个文字，多余元素会被过滤。锁屏预览会受壁纸和系统色调影响。

组件进程内存有限。复用图片，避免原尺寸照片，限制 JSON 大小，不在循环中反复创建 `DrawContext`，并在网络失败时使用小型缓存。

<a id="interfaces"></a>

## 对话框、表格和系统界面

需要确认、少量选择或文本输入时使用 `Alert`。需要日期时使用 `DatePicker`。需要多行结构化信息时使用 `UITable`。需要完整网页交互时使用 `WebView`。

```javascript
async function askForName() {
  const alert = new Alert()
  alert.title = "Create item"
  alert.addTextField("Name", "")
  alert.addCancelAction("Cancel")
  alert.addAction("Create")

  const action = await alert.presentAlert()
  if (action === -1) return null

  const value = alert.textFieldValue(0).trim()
  if (!value) throw new Error("Name is required")
  return value
}
```

危险操作使用 `addDestructiveAction()`，并在按钮标题中写明动作。系统邮件和信息 API 只打开编辑界面，最终发送由用户确认。共享表单、文件选择器、照片选择器和系统编辑器都可能被取消。

表格在显示期间修改行后必须调用 `reload()`。Siri 中显示的表格不支持点击行或按钮，因此脚本不能把必要操作只放在表格回调中。

<a id="personal-data"></a>

## 日历、提醒事项、通讯录、定位和照片

创建事件和提醒事项时，设置所属日历并调用 `save()`。删除日历、事件和提醒事项会修改用户数据，执行前显示目标标题、日期和日历名称。

```javascript
async function createReminder(title, dueDate) {
  const reminder = new Reminder()
  reminder.title = title
  reminder.dueDate = dueDate
  reminder.calendar = await Calendar.defaultForReminders()
  reminder.save()
  return reminder
}
```

通讯录修改采用排队和提交两步。创建用 `Contact.add()`，更新用 `Contact.update()`，删除用 `Contact.delete()`，最后等待 `Contact.persistChanges()`。更新邮箱、电话、地址、社交账号、网址和日期时，先保留仍需存在的旧项目，因为赋值会替换整个数组。

定位和照片在首次调用时请求权限。拒绝权限后，脚本应说明需要到系统设置授权，不能通过循环重试触发新提示。读取最近照片或截图时处理没有匹配项的拒绝结果。

<a id="automation"></a>

## 通知、快捷指令和应用间调用

通知只有在调用 `schedule()` 后才会生效。修改通知标题、正文、时间或动作后，再次调用 `schedule()`。

```javascript
const notification = new Notification()
notification.identifier = "example.daily-status"
notification.title = "Daily status"
notification.body = "Open the report"
notification.setDailyTrigger(9, 0, true)
notification.openURL = URLScheme.forRunningScript()
await notification.schedule()
```

通知标识符应稳定，使脚本可以更新或删除自己创建的通知。调用 `removeAllPending()` 或 `removeAllDelivered()` 会影响其他脚本创建的通知，除非用户明确要求，不要使用。

快捷指令输入从 `args.shortcutParameter` 读取，输出交给 `Script.setShortcutOutput()`。共享表单使用有类型的 `args` 数组。URL Scheme 参数从 `args.queryParameters` 读取。生成运行地址时使用 `URLScheme.forRunningScript()`，自行构造地址时对脚本名和每个查询参数进行 URL 编码。

调用支持 x-callback-url 的应用时，用 `CallbackURL.addParameter()` 添加业务参数。不要自行添加 `x-success`、`x-error` 和 `x-cancel`，`CallbackURL` 会管理这些回调。

<a id="advanced"></a>

## 模块、WebView、绘图和 XML

模块使用 `module.exports` 导出值，使用 `importModule()` 导入。

```javascript
// format.js
module.exports = {
  upper(value) {
    return String(value).toUpperCase()
  }
}

// main.js
const format = importModule("format")
console.log(format.upper("scriptable"))
```

`importModule()` 允许省略 `.js`。传入目录时会查找 `index.js`。循环依赖和隐式全局变量会增加调试难度，应保持依赖方向单一，并把共享状态作为参数传入。

WebView 中执行页面代码时，传入 `true` 会进入回调模式。页面代码必须调用 `completion()`。

```javascript
const view = new WebView()
await view.loadHTML("<main id='value'>42</main>")
const value = await view.evaluateJavaScript(`
  const text = document.getElementById("value").textContent
  completion(text)
`, true)
```

DOM 只存在于 WebView 加载的页面内。Scriptable 主脚本仍然没有 `document`。把主脚本数据插入 HTML 或 JavaScript 时使用 `JSON.stringify()` 编码值，并限制 WebView 可以加载的主机。

`DrawContext` 必须先设置画布尺寸。调用 `addPath()` 后，`fillPath()` 或 `strokePath()` 只处理最近加入的路径。需要绘制多个路径时，每个路径按“加入、填充或描边”的顺序处理。

XMLParser 是事件驱动解析器。`foundCharacters` 可能把一个文本节点分成多次回调。

```javascript
const parser = new XMLParser(xml)
let current = null
let buffer = ""
const values = []

parser.didStartElement = name => {
  current = name
  buffer = ""
}
parser.foundCharacters = text => {
  buffer += text
}
parser.didEndElement = name => {
  if (name === current) values.push({ name, text: buffer })
  current = null
  buffer = ""
}
parser.parseErrorOccurred = error => {
  throw error
}

if (!parser.parse()) throw new Error("XML parser did not start")
```

<a id="security"></a>

## 权限与安全

以下规则适用于生成和审查脚本。

- 把访问令牌、密码和长期密钥放进 `Keychain`，不写进脚本、组件参数、URL 或日志。
- 为 `Request` 设置允许的 HTTPS 主机。不要启用 `allowInsecureRequest`，除非用户明确接受该请求的风险。
- WebView 使用 `shouldAllowRequest` 限制未知主机和未知 URL Scheme。
- 删除文件、照片、日历、事件、提醒事项、联系人、分组和通知前，显示准确目标并取得确认。
- 通讯录、日历、定位和照片数据只读取完成任务所需的字段，不写入日志。
- 文件路径来自用户输入时，先限定到预期目录，并拒绝空路径、目录根路径和上级目录跳转。
- 从 URL Scheme 或共享表单读取的值视为外部输入。验证类型、长度、主机、协议和数值范围。
- 把外部文本插入 HTML 或页面 JavaScript 时进行编码。不要把未验证文本拼接成可执行 JavaScript。

权限由系统管理。脚本第一次访问定位、照片、日历、提醒事项或通讯录时可能出现授权界面。用户拒绝后，应报告具体权限和系统设置路径，不重复执行同一个失败调用。

<a id="debugging"></a>

## 调试与验证

按症状检查真实失败位置。

| 症状 | 检查项 |
| --- | --- |
| `document is not defined` | DOM 代码是否误放在主脚本；只有 WebView 页面内有 DOM |
| `Keychain.get()` 抛错 | 读取前是否调用 `Keychain.contains()` |
| iCloud 文件存在但读取失败 | 是否等待 `downloadFileFromiCloud()` |
| Promise 出现在结果中 | 是否遗漏 `await` |
| 组件空白 | 日志、内存占用、图片尺寸、当前 `widgetFamily`、不支持的 API |
| 组件没有按时刷新 | `refreshAfterDate` 是否被误解为准确计划时间 |
| 组件点击目标无效 | 小型组件是否只设置了根 `url`；根 `url` 是否覆盖其他行为 |
| 快捷指令输入为空 | 是否读取 `args.shortcutParameter`；输入类型是否匹配 |
| URL 参数为空 | 是否读取 `args.queryParameters`；参数是否正确编码 |
| 表格更新不显示 | 增删行后是否调用 `reload()` |
| 联系人修改未生效 | 是否排队变更并等待 `Contact.persistChanges()` |
| 通知修改未生效 | 修改后是否再次调用 `schedule()` |
| XML 文本缺字 | 是否累计多次 `foundCharacters` 回调 |

静态验证至少完成以下检查。

1. 所有 API 名称和大小写与[完整 API 索引](#api-reference)一致。
2. 所有依赖异步结果的调用都使用 `await`。
3. 主脚本没有 DOM、浏览器和 Node.js 专用 API。
4. 代码处理空输入、取消操作、权限拒绝、网络错误、非 2xx 状态和无效 JSON。
5. 删除和批量修改操作有明确范围。
6. 组件覆盖要求支持的全部尺寸，并提供应用内预览路径。
7. 日志不含钥匙串值、认证头、Cookie 和个人数据。

无法在普通 Node.js 或浏览器中完整验证 Scriptable 脚本。语法和纯函数可以在本地检查；权限、系统界面、组件布局、iCloud、通知和个人数据操作必须在目标 iPhone 或 iPad 上运行。

<a id="deprecated"></a>

## 已废弃接口

| 接口 | 状态 | 使用方式 |
| --- | --- | --- |
| `args.length` | 1.3 起废弃 | 读取具体类型数组的 `length` |
| `args.all` | 1.3 起废弃 | 读取 `plainTexts`、`urls`、`fileURLs` 或 `images` |
| `args.siriShortcutArguments` | 1.4 起废弃 | 使用 `args.shortcutParameter` |
| `console.logError()` | 1.3 起废弃 | 使用 `console.error()` |
| `Contact.note` | 1.7.5 起废弃 | 联系人备注不再可用 |
| `Contact.isNoteAvailable` | 1.7.5 起废弃 | 不读取联系人备注 |
| `DrawContext.setFontSize()` | 1.5 起废弃 | 使用 `setFont()` |
| `Notification.current()` | 1.3 起废弃 | 使用 `args.notification` |
| `URLScheme.allParameters()` | 1.3 起废弃 | 使用 `args.queryParameters` |
| `URLScheme.parameter()` | 1.3 起废弃 | 使用 `args.queryParameters[name]` |

<a id="api-reference"></a>

## 完整 API 索引

下面的签名保留官方记法。每个标题链接到对应的官方页面。成员按官方页面顺序排列。

### API 页面目录

- 运行环境、输入与控制
  - [args](#api-args)
  - [config](#api-config)
  - [console](#api-console)
  - [Script](#api-script)
  - [importModule](#api-importmodule)
  - [module](#api-module)
  - [URLScheme](#api-urlscheme)
  - [CallbackURL](#api-callbackurl)
  - [Timer](#api-timer)
  - [UUID](#api-uuid)
- 网络、数据与文件
  - [Request](#api-request)
  - [Data](#api-data)
  - [FileManager](#api-filemanager)
  - [Image](#api-image)
  - [Keychain](#api-keychain)
  - [Pasteboard](#api-pasteboard)
  - [DocumentPicker](#api-documentpicker)
  - [ShareSheet](#api-sharesheet)
  - [QuickLook](#api-quicklook)
- 交互界面与系统界面
  - [Alert](#api-alert)
  - [TextField](#api-textfield)
  - [DatePicker](#api-datepicker)
  - [UITable](#api-uitable)
  - [UITableRow](#api-uitablerow)
  - [UITableCell](#api-uitablecell)
  - [WebView](#api-webview)
  - [Safari](#api-safari)
  - [Mail](#api-mail)
  - [Message](#api-message)
  - [Dictation](#api-dictation)
  - [Speech](#api-speech)
- 桌面与锁屏组件
  - [ListWidget](#api-listwidget)
  - [WidgetStack](#api-widgetstack)
  - [WidgetText](#api-widgettext)
  - [WidgetDate](#api-widgetdate)
  - [WidgetImage](#api-widgetimage)
  - [WidgetSpacer](#api-widgetspacer)
  - [Color](#api-color)
  - [Font](#api-font)
  - [LinearGradient](#api-lineargradient)
  - [SFSymbol](#api-sfsymbol)
- 个人数据与设备能力
  - [Calendar](#api-calendar)
  - [CalendarEvent](#api-calendarevent)
  - [Reminder](#api-reminder)
  - [RecurrenceRule](#api-recurrencerule)
  - [Contact](#api-contact)
  - [ContactsContainer](#api-contactscontainer)
  - [ContactsGroup](#api-contactsgroup)
  - [Location](#api-location)
  - [Photos](#api-photos)
  - [Notification](#api-notification)
  - [Device](#api-device)
- 绘图、几何、格式化与解析
  - [Point](#api-point)
  - [Size](#api-size)
  - [Rect](#api-rect)
  - [Path](#api-path)
  - [DrawContext](#api-drawcontext)
  - [DateFormatter](#api-dateformatter)
  - [RelativeDateTimeFormatter](#api-relativedatetimeformatter)
  - [XMLParser](#api-xmlparser)

<a id="category-args"></a>

## 运行环境、输入与控制

<a id="api-args"></a>

### [args](https://docs.scriptable.app/args/)

`args` 读取共享表单、快捷指令、URL Scheme、组件和通知传入的参数。

使用时遵守以下限制。

- 优先读取有类型的 `plainTexts`、`urls`、`fileURLs`、`images`、`shortcutParameter`、`widgetParameter`、`queryParameters` 和 `notification`。
- `length`、`all` 和 `siriShortcutArguments` 已废弃。

```javascript
length: number
all: [any]
plainTexts: [string]
urls: [string]
fileURLs: [string]
images: [Image]
queryParameters: {string: string}
siriShortcutArguments: {string: string}
shortcutParameter: any
widgetParameter: any
notification: Notification
```

<a id="api-config"></a>

### [config](https://docs.scriptable.app/config/)

`config` 说明脚本当前从应用、共享表单、Siri、组件、通知或主屏幕中的哪一种环境运行。

使用时遵守以下限制。

- `widgetFamily` 可能为 `small`、`medium`、`large`、`extraLarge`、`accessoryRectangular`、`accessoryInline`、`accessoryCircular` 或 `null`。

```javascript
runsInApp: bool
runsInActionExtension: bool
runsWithSiri: bool
runsInWidget: bool
runsInAccessoryWidget: bool
runsInNotification: bool
runsFromHomeScreen: bool
widgetFamily: string
```

<a id="api-console"></a>

### [console](https://docs.scriptable.app/console/)

`console` 向 Scriptable 日志写入普通、警告和错误消息。

```javascript
static log(message: any)
static warn(message: any)
static error(message: any)
static logError(message: any)
```

<a id="api-script"></a>

### [Script](https://docs.scriptable.app/script/)

`Script` 读取脚本名，设置组件或快捷指令输出，并标记脚本完成。

使用时遵守以下限制。

- 组件通过 `Script.setWidget()` 输出，快捷指令通过 `Script.setShortcutOutput()` 或脚本返回值输出。
- 异步工作结束后再调用 `Script.complete()`。

```javascript
static name(): string
static complete()
static setShortcutOutput(value: any)
static setWidget(widget: any)
```

<a id="api-importmodule"></a>

### [importModule](https://docs.scriptable.app/importmodule/)

`importModule()` 加载另一个脚本文件，并返回该文件的 `module.exports`。

使用时遵守以下限制。

- 查找顺序依次为当前文件所在目录、iCloud Scriptable 目录、应用组目录和本地 Scriptable 目录。
- 传入目录时，Scriptable 会查找该目录下的 `index.js`。

```javascript
importModule(name: string)
```

<a id="api-module"></a>

### [module](https://docs.scriptable.app/module/)

`module` 提供当前模块路径和导出值。

使用时遵守以下限制。

- `module.exports` 可以是对象、函数、字符串或其他任意值。

```javascript
filename: string
exports: any
```

<a id="api-urlscheme"></a>

### [URLScheme](https://docs.scriptable.app/urlscheme/)

`URLScheme` 生成打开脚本、打开设置或运行脚本的 `scriptable://` 地址。

使用时遵守以下限制。

- 脚本名必须进行 URL 编码。
- `allParameters()` 和 `parameter()` 已废弃，应读取 `args.queryParameters`。

```javascript
static allParameters(): {string: string}
static parameter(name: string): string
static forOpeningScript(): string
static forOpeningScriptSettings(): string
static forRunningScript(): string
```

<a id="api-callbackurl"></a>

### [CallbackURL](https://docs.scriptable.app/callbackurl/)

`CallbackURL` 调用支持 x-callback-url 的应用，并等待对方回传查询参数。

```javascript
new CallbackURL(baseURL: string)
addParameter(name: string, value: string)
open(): Promise<{string: string}>
getURL(): string
```

<a id="api-timer"></a>

### [Timer](https://docs.scriptable.app/timer/)

`Timer` 在指定秒数后执行回调，并可重复执行。

使用时遵守以下限制。

- 重复计时器必须调用 `invalidate()` 停止，否则会继续触发。

```javascript
timeInterval: number
repeats: bool
new Timer()
schedule(callback: fn())
invalidate()
static schedule(timeInterval: number, repeats: bool, callback: fn()): Timer
```

<a id="api-uuid"></a>

### [UUID](https://docs.scriptable.app/uuid/)

`UUID` 生成通用唯一标识符字符串。

```javascript
static string(): string
```


<a id="category-request"></a>

## 网络、数据与文件

<a id="api-request"></a>

### [Request](https://docs.scriptable.app/request/)

`Request` 发送 HTTP 请求，并把响应读取为二进制、文本、JSON 或图片。

使用时遵守以下限制。

- `body` 当前只支持字符串或 `Data`。
- `timeoutInterval` 默认 60 秒。读取完成后通过 `response.statusCode` 检查 HTTP 状态。
- `allowInsecureRequest` 默认关闭。除非用户明确接受风险，不要启用。
- `onRedirect` 返回 `null` 可停止跳转，该回调只对初始请求生效。

```javascript
url: string
method: string
headers: {string: string}
body: any
timeoutInterval: number
onRedirect: fn(Request) -> Request
response: {string: any}
allowInsecureRequest: bool
new Request(url: string)
load(): Promise<Data>
loadString(): Promise<string>
loadJSON(): Promise<any>
loadImage(): Promise<Image>
addParameterToMultipart(name: string, value: string)
addFileDataToMultipart(data: Data, mimeType: string, name: string, filename: string)
addFileToMultipart(filePath: string, name: string, filename: string)
addImageToMultipart(image: Image, name: string, filename: string)
```

<a id="api-data"></a>

### [Data](https://docs.scriptable.app/data/)

`Data` 在字节、UTF-8 字符串、Base64、文件和图片之间转换。

使用时遵守以下限制。

- 无效 UTF-8 或无效 Base64 输入会返回 `null`，调用方必须检查。

```javascript
static fromString(string: string): Data
static fromFile(filePath: string): Data
static fromBase64String(base64String: string): Data
static fromJPEG(image: Image): Data
static fromPNG(image: Image): Data
static fromBytes(bytes: [number]): Data
toRawString(): string
toBase64String(): string
getBytes(): [number]
```

<a id="api-filemanager"></a>

### [FileManager](https://docs.scriptable.app/filemanager/)

`FileManager` 管理本地或 iCloud 文件、目录、标签、扩展属性和文件书签。

使用时遵守以下限制。

- 本地文件和 iCloud 文件必须由同一个 `FileManager` 实例处理。
- 读取 iCloud 文件前检查 `isFileDownloaded()`，必要时等待 `downloadFileFromiCloud()`。
- `temporaryDirectory()` 和 `cacheDirectory()` 不在应用、共享扩展和 Siri 之间共享。
- 从 Scriptable 设置创建的文件书签只能在应用内使用；Siri 和快捷指令需要由相应的快捷指令动作创建书签。
- `remove()` 删除的文件无法恢复。

```javascript
static local(): FileManager
static iCloud(): FileManager
read(filePath: string): Data
readString(filePath: string): string
readImage(filePath: string): Image
write(filePath: string, content: Data)
writeString(filePath: string, content: string)
writeImage(filePath: string, image: Image)
remove(filePath: string)
move(sourceFilePath: string, destinationFilePath: string)
copy(sourceFilePath: string, destinationFilePath: string)
fileExists(filePath: string): bool
isDirectory(path: string): bool
createDirectory(path: string, intermediateDirectories: bool)
temporaryDirectory(): string
cacheDirectory(): string
documentsDirectory(): string
libraryDirectory(): string
joinPath(lhsPath: string, rhsPath: string): string
allTags(filePath: string): [string]
addTag(filePath: string, tag: string)
removeTag(filePath: string, tag: string)
readExtendedAttribute(filePath: string, name: string): string
writeExtendedAttribute(filePath: string, value: string, name: string)
removeExtendedAttribute(filePath: string, name: string)
allExtendedAttributes(filePath: string): [string]
getUTI(filePath: string): string
listContents(directoryPath: string): [string]
fileName(filePath: string, includeFileExtension: bool): string
fileExtension(filePath: string): string
bookmarkedPath(name: string): string
bookmarkExists(name: string): bool
downloadFileFromiCloud(filePath: string): Promise
isFileStoredIniCloud(filePath: string): bool
isFileDownloaded(filePath: string): bool
creationDate(filePath: string): Date
modificationDate(filePath: string): Date
fileSize(filePath: string): number
allFileBookmarks(): [{string: string}]
```

<a id="api-image"></a>

### [Image](https://docs.scriptable.app/image/)

`Image` 表示图片数据，并可从文件或 `Data` 解码图片。

使用时遵守以下限制。

- 文件或二进制数据不能解码为图片时，`Image.fromFile()` 和 `Image.fromData()` 返回 `null`。

```javascript
size: Size
static fromFile(filePath: string): Image
static fromData(data: Data): Image
```

<a id="api-keychain"></a>

### [Keychain](https://docs.scriptable.app/keychain/)

`Keychain` 以键值形式保存凭据、令牌和其他字符串秘密。

使用时遵守以下限制。

- `Keychain.get()` 在键不存在时抛出错误，应先调用 `Keychain.contains()`。

```javascript
static contains(key: string): bool
static set(key: string, value: string)
static get(key: string): string
static remove(key: string)
```

<a id="api-pasteboard"></a>

### [Pasteboard](https://docs.scriptable.app/pasteboard/)

`Pasteboard` 在系统剪贴板中读写字符串和图片。

```javascript
static copy(string: string)
static paste(): string
static copyString(string: string)
static pasteString(): string
static copyImage(image: Image)
static pasteImage(): Image
```

<a id="api-documentpicker"></a>

### [DocumentPicker](https://docs.scriptable.app/documentpicker/)

`DocumentPicker` 从文件应用选择文件或文件夹，也可导出字符串、图片和二进制数据。

```javascript
static open(types: [string]): Promise<[string]>
static openFile(): Promise<string>
static openFolder(): Promise<string>
static export(path: string): Promise<[string]>
static exportString(content: string, name: string): Promise<[string]>
static exportImage(image: Image, name: string): Promise<[string]>
static exportData(data: Data, name: string): Promise<[string]>
```

<a id="api-sharesheet"></a>

### [ShareSheet](https://docs.scriptable.app/sharesheet/)

`ShareSheet` 把一组项目交给系统共享界面处理。

```javascript
static present(activityItems: [any]): Promise<{string: any}>
```

<a id="api-quicklook"></a>

### [QuickLook](https://docs.scriptable.app/quicklook/)

`QuickLook` 用系统预览界面显示文本、图片或文件。

```javascript
static present(item: any, fullscreen: bool): Promise
```


<a id="category-alert"></a>

## 交互界面与系统界面

<a id="api-alert"></a>

### [Alert](https://docs.scriptable.app/alert/)

`Alert` 配置模态对话框或操作表，并返回用户选择的按钮索引。

使用时遵守以下限制。

- `presentAlert()` 和 `presentSheet()` 返回按钮索引；取消按钮返回 `-1`。
- `presentSheet()` 不支持输入框。iPad 上的 sheet 不显示取消按钮，用户点击 sheet 外部即可取消。

```javascript
title: string
message: string
new Alert()
addAction(title: string)
addDestructiveAction(title: string)
addCancelAction(title: string)
addTextField(placeholder: string, text: string): TextField
addSecureTextField(placeholder: string, text: string): TextField
textFieldValue(index: number): string
present(): Promise<number>
presentAlert(): Promise<number>
presentSheet(): Promise<number>
```

<a id="api-textfield"></a>

### [TextField](https://docs.scriptable.app/textfield/)

`TextField` 配置 `Alert` 中输入框的文字、字体、颜色、键盘、对齐和安全输入。

使用时遵守以下限制。

- `TextField` 由 `Alert.addTextField()` 或 `Alert.addSecureTextField()` 创建，不要直接实例化。

```javascript
text: string
placeholder: string
isSecure: bool
textColor: Color
font: Font
setDefaultKeyboard()
setNumberPadKeyboard()
setDecimalPadKeyboard()
setNumbersAndPunctuationKeyboard()
setPhonePadKeyboard()
setWebSearchKeyboard()
setEmailAddressKeyboard()
setURLKeyboard()
setTwitterKeyboard()
leftAlignText()
centerAlignText()
rightAlignText()
```

<a id="api-datepicker"></a>

### [DatePicker](https://docs.scriptable.app/datepicker/)

`DatePicker` 让用户选择时间、日期、日期时间或倒计时长度。

使用时遵守以下限制。

- `initialDate` 只设置初始值，用户选择的结果由 `pick...()` 返回。
- `minuteInterval` 的范围是 1 到 30，倒计时最长为 86399 秒。

```javascript
minimumDate: Date
maximumDate: Date
countdownDuration: number
minuteInterval: number
initialDate: Date
new DatePicker()
pickTime(): Promise<Date>
pickDate(): Promise<Date>
pickDateAndTime(): Promise<Date>
pickCountdownDuration(): Promise<number>
```

<a id="api-uitable"></a>

### [UITable](https://docs.scriptable.app/uitable/)

`UITable` 纵向显示由 `UITableRow` 和 `UITableCell` 组成的交互表格。

使用时遵守以下限制。

- 表格显示期间增删行后，调用 `reload()` 才会刷新界面。

```javascript
showSeparators: bool
new UITable()
addRow(row: UITableRow)
removeRow(row: UITableRow)
removeAllRows()
reload()
present(fullscreen: bool): Promise
```

<a id="api-uitablerow"></a>

### [UITableRow](https://docs.scriptable.app/uitablerow/)

`UITableRow` 配置表格行的单元格、尺寸、背景和选择回调。

使用时遵守以下限制。

- 表格在 Siri 中显示时，行不能点击。

```javascript
cellSpacing: number
height: number
isHeader: bool
dismissOnSelect: bool
onSelect: fn()
backgroundColor: Color
new UITableRow()
addCell(cell: UITableCell)
addText(title: string, subtitle: string): UITableCell
addImage(image: Image): UITableCell
addImageAtURL(url: string): UITableCell
addButton(title: string): UITableCell
```

<a id="api-uitablecell"></a>

### [UITableCell](https://docs.scriptable.app/uitablecell/)

`UITableCell` 创建表格中的文字、图片或按钮单元格。

使用时遵守以下限制。

- 表格在 Siri 中显示时，按钮不能点击。

```javascript
widthWeight: number
onTap: fn()
dismissOnTap: bool
titleColor: Color
subtitleColor: Color
titleFont: Font
subtitleFont: Font
static text(title: string, subtitle: string): UITableCell
static image(image: Image): UITableCell
static imageAtURL(url: string): UITableCell
static button(title: string): UITableCell
leftAligned()
centerAligned()
rightAligned()
```

<a id="api-webview"></a>

### [WebView](https://docs.scriptable.app/webview/)

`WebView` 加载网页、HTML、文件或 `Request`，执行页面 JavaScript，并控制页面请求。

使用时遵守以下限制。

- `evaluateJavaScript(code, true)` 只有在页面代码调用全局 `completion(value)` 后才会结束。
- `preferredSize` 只在 Siri 快捷指令中生效。
- 扩展进程内存有限。显示大型图片时，WebView 通常比 QuickLook 占用更少的扩展内存。
- `waitForLoad()` 只适合等待由 `evaluateJavaScript()` 引起的新页面加载。

```javascript
shouldAllowRequest: fn(Request) -> bool
new WebView()
static loadHTML(html: string, baseURL: string, preferredSize: Size, fullscreen: bool): Promise
static loadFile(fileURL: string, preferredSize: Size, fullscreen: bool): Promise
static loadURL(url: string, preferredSize: Size, fullscreen: bool): Promise
loadURL(url: string): Promise
loadRequest(request: Request): Promise
loadHTML(html: string, baseURL: string): Promise
loadFile(fileURL: string): Promise
evaluateJavaScript(javaScript: string, useCallback: bool): Promise<any>
getHTML(): Promise<any>
present(fullscreen: bool): Promise
waitForLoad(): Promise<any>
```

<a id="api-safari"></a>

### [Safari](https://docs.scriptable.app/safari/)

`Safari` 在 Scriptable 内或 Safari 中打开网页。

```javascript
static openInApp(url: string, fullscreen: bool): Promise
static open(url: string)
```

<a id="api-mail"></a>

### [Mail](https://docs.scriptable.app/mail/)

`Mail` 打开系统邮件编辑界面，并可预填收件人、主题、正文和附件。

```javascript
toRecipients: [string]
ccRecipients: [string]
bccRecipients: [string]
subject: string
body: string
isBodyHTML: bool
preferredSendingEmailAddress: string
new Mail()
send(): Promise
addImageAttachment(image: Image)
addFileAttachment(filePath: string)
addDataAttachment(data: Data, mimeType: string, filename: string)
```

<a id="api-message"></a>

### [Message](https://docs.scriptable.app/message/)

`Message` 打开系统信息编辑界面，并可预填收件人、正文和附件。

```javascript
recipients: [string]
body: string
new Message()
send(): Promise
addImageAttachment(image: Image)
addFileAttachment(filePath: string)
addDataAttachment(data: Data, uti: string, filename: string)
```

<a id="api-dictation"></a>

### [Dictation](https://docs.scriptable.app/dictation/)

`Dictation` 打开听写界面，并返回识别出的文本。

使用时遵守以下限制。

- 用户需要在显示的界面中手动停止听写。

```javascript
static start(locale: string): Promise<string>
```

<a id="api-speech"></a>

### [Speech](https://docs.scriptable.app/speech/)

`Speech` 朗读文本；脚本由 Siri 快捷指令触发时，由 Siri 朗读。

```javascript
static speak(text: string)
```


<a id="category-listwidget"></a>

## 桌面与锁屏组件

<a id="api-listwidget"></a>

### [ListWidget](https://docs.scriptable.app/listwidget/)

`ListWidget` 创建桌面或锁屏组件，并交给 `Script.setWidget()` 显示。

使用时遵守以下限制。

- `refreshAfterDate` 只规定最早刷新时间，实际刷新时机由 iOS 或 iPadOS 决定。
- 组件存在内存限制。缓存并缩小图片，减少重复网络请求和大型中间数据。
- `url` 会覆盖组件设置中的交互行为。
- 锁屏组件预览只提供估计效果，壁纸和系统色调会改变实际显示。

```javascript
backgroundColor: Color
backgroundImage: Image
backgroundGradient: LinearGradient
addAccessoryWidgetBackground: bool
spacing: number
url: string
refreshAfterDate: Date
new ListWidget()
addText(text: string): WidgetText
addDate(date: Date): WidgetDate
addImage(image: Image): WidgetImage
addSpacer(length: number): WidgetSpacer
addStack(): WidgetStack
setPadding(top: number, leading: number, bottom: number, trailing: number)
useDefaultPadding()
presentSmall(): Promise
presentMedium(): Promise
presentLarge(): Promise
presentExtraLarge(): Promise
presentAccessoryInline(): Promise
presentAccessoryCircular(): Promise
presentAccessoryRectangular(): Promise
```

<a id="api-widgetstack"></a>

### [WidgetStack](https://docs.scriptable.app/widgetstack/)

`WidgetStack` 以水平或垂直方向组织组件元素，并支持嵌套。

使用时遵守以下限制。

- stack 默认水平排列；调用 `layoutVertically()` 可改为垂直排列。
- 宽度或高度小于等于零时，由系统决定该方向的尺寸。
- stack 的点击地址只支持中型和大型组件。

```javascript
backgroundColor: Color
backgroundImage: Image
backgroundGradient: LinearGradient
spacing: number
size: Size
cornerRadius: number
borderWidth: number
borderColor: Color
url: string
addText(text: string): WidgetText
addDate(date: Date): WidgetDate
addImage(image: Image): WidgetImage
addSpacer(length: number): WidgetSpacer
addStack(): WidgetStack
setPadding(top: number, leading: number, bottom: number, trailing: number)
useDefaultPadding()
topAlignContent()
centerAlignContent()
bottomAlignContent()
layoutHorizontally()
layoutVertically()
```

<a id="api-widgettext"></a>

### [WidgetText](https://docs.scriptable.app/widgettext/)

`WidgetText` 配置组件文字的内容、字体、颜色、缩放、阴影、对齐和点击地址。

使用时遵守以下限制。

- 组件元素的 `url` 只支持中型和大型组件。小型组件只能使用 `ListWidget.url`。
- 元素位于 stack 中时，使用 spacer 控制水平位置。

```javascript
text: string
textColor: Color
font: Font
textOpacity: number
lineLimit: number
minimumScaleFactor: number
shadowColor: Color
shadowRadius: number
shadowOffset: Point
url: string
leftAlignText()
centerAlignText()
rightAlignText()
```

<a id="api-widgetdate"></a>

### [WidgetDate](https://docs.scriptable.app/widgetdate/)

`WidgetDate` 在组件中显示会随时间更新的日期文本。

使用时遵守以下限制。

- 组件元素的 `url` 只支持中型和大型组件。小型组件只能使用 `ListWidget.url` 作为单一点击目标。
- 元素位于 stack 中时，文字对齐方法不改变 stack 内布局，应使用 spacer 控制位置。

```javascript
date: Date
textColor: Color
font: Font
textOpacity: number
lineLimit: number
minimumScaleFactor: number
shadowColor: Color
shadowRadius: number
shadowOffset: Point
url: string
leftAlignText()
centerAlignText()
rightAlignText()
applyTimeStyle()
applyDateStyle()
applyRelativeStyle()
applyOffsetStyle()
applyTimerStyle()
```

<a id="api-widgetimage"></a>

### [WidgetImage](https://docs.scriptable.app/widgetimage/)

`WidgetImage` 配置组件图片的尺寸、透明度、边框、圆角、色调和填充方式。

使用时遵守以下限制。

- 组件元素的 `url` 只支持中型和大型组件。小型组件只能使用 `ListWidget.url`。
- `tintColor = null` 保留原图颜色。

```javascript
image: Image
resizable: bool
imageSize: Size
imageOpacity: number
cornerRadius: number
borderWidth: number
borderColor: Color
containerRelativeShape: bool
tintColor: Color
url: string
leftAlignImage()
centerAlignImage()
rightAlignImage()
applyFittingContentMode()
applyFillingContentMode()
```

<a id="api-widgetspacer"></a>

### [WidgetSpacer](https://docs.scriptable.app/widgetspacer/)

`WidgetSpacer` 在组件布局中加入固定或弹性空白。

使用时遵守以下限制。

- `length = null` 表示弹性空白。

```javascript
length: number
```

<a id="api-color"></a>

### [Color](https://docs.scriptable.app/color/)

`Color` 表示带透明度的颜色，并支持随浅色或深色外观变化的动态颜色。

使用时遵守以下限制。

- 组件可用 `Color.dynamic()` 适配浅色和深色外观。
- `DrawContext` 不支持动态颜色，应在绘图前选择一个确定颜色。

```javascript
hex: string
red: number
green: number
blue: number
alpha: number
static black(): Color
static darkGray(): Color
static lightGray(): Color
static white(): Color
static gray(): Color
static red(): Color
static green(): Color
static blue(): Color
static cyan(): Color
static yellow(): Color
static magenta(): Color
static orange(): Color
static purple(): Color
static brown(): Color
static clear(): Color
new Color(hex: string, alpha: number)
static dynamic(lightColor: Color, darkColor: Color): Color
```

<a id="api-font"></a>

### [Font](https://docs.scriptable.app/font/)

`Font` 为文字和组件创建系统字体、等宽字体、圆角字体或指定名称的字体。

使用时遵守以下限制。

- 指定字体名称前确认该字体存在于目标 iOS 或 iPadOS 版本。

```javascript
new Font(name: string, size: number)
static largeTitle(): Font
static title1(): Font
static title2(): Font
static title3(): Font
static headline(): Font
static subheadline(): Font
static body(): Font
static callout(): Font
static footnote(): Font
static caption1(): Font
static caption2(): Font
static systemFont(size: number): Font
static ultraLightSystemFont(size: number): Font
static thinSystemFont(size: number): Font
static lightSystemFont(size: number): Font
static regularSystemFont(size: number): Font
static mediumSystemFont(size: number): Font
static semiboldSystemFont(size: number): Font
static boldSystemFont(size: number): Font
static heavySystemFont(size: number): Font
static blackSystemFont(size: number): Font
static italicSystemFont(size: number): Font
static ultraLightMonospacedSystemFont(size: number): Font
static thinMonospacedSystemFont(size: number): Font
static lightMonospacedSystemFont(size: number): Font
static regularMonospacedSystemFont(size: number): Font
static mediumMonospacedSystemFont(size: number): Font
static semiboldMonospacedSystemFont(size: number): Font
static boldMonospacedSystemFont(size: number): Font
static heavyMonospacedSystemFont(size: number): Font
static blackMonospacedSystemFont(size: number): Font
static ultraLightRoundedSystemFont(size: number): Font
static thinRoundedSystemFont(size: number): Font
static lightRoundedSystemFont(size: number): Font
static regularRoundedSystemFont(size: number): Font
static mediumRoundedSystemFont(size: number): Font
static semiboldRoundedSystemFont(size: number): Font
static boldRoundedSystemFont(size: number): Font
static heavyRoundedSystemFont(size: number): Font
static blackRoundedSystemFont(size: number): Font
```

<a id="api-lineargradient"></a>

### [LinearGradient](https://docs.scriptable.app/lineargradient/)

`LinearGradient` 定义组件使用的颜色、位置和起止点。

```javascript
colors: [Color]
locations: [number]
startPoint: Point
endPoint: Point
new LinearGradient()
```

<a id="api-sfsymbol"></a>

### [SFSymbol](https://docs.scriptable.app/sfsymbol/)

`SFSymbol` 按系统符号名称创建图片，并应用字体或字重。

```javascript
image: Image
static named(symbolName: string): SFSymbol
applyFont(font: Font)
applyUltraLightWeight()
applyThinWeight()
applyLightWeight()
applyRegularWeight()
applyMediumWeight()
applySemiboldWeight()
applyBoldWeight()
applyHeavyWeight()
applyBlackWeight()
```


<a id="category-calendar"></a>

## 个人数据与设备能力

<a id="api-calendar"></a>

### [Calendar](https://docs.scriptable.app/calendar/)

`Calendar` 查找、创建、选择和删除保存日历事件或提醒事项的日历。

使用时遵守以下限制。

- 一个日历只能保存事件或提醒事项中的一种。
- 删除日历无法撤销。修改前检查 `allowsContentModifications`。

```javascript
identifier: string
title: string
isSubscribed: bool
allowsContentModifications: bool
color: Color
supportsAvailability(availability: string): bool
save()
remove()
static forReminders(): Promise<[Calendar]>
static forEvents(): Promise<[Calendar]>
static forRemindersByTitle(title: string): Promise<Calendar>
static forEventsByTitle(title: string): Promise<Calendar>
static createForReminders(title: string): Promise<Calendar>
static findOrCreateForReminders(title: string): Promise<Calendar>
static defaultForReminders(): Promise<Calendar>
static defaultForEvents(): Promise<Calendar>
static presentPicker(allowMultiple: bool): Promise<[Calendar]>
```

<a id="api-calendarevent"></a>

### [CalendarEvent](https://docs.scriptable.app/calendarevent/)

`CalendarEvent` 创建、查询、编辑、保存和删除日历事件。

使用时遵守以下限制。

- 新事件只有在调用 `save()` 后才会写入日历。
- `attendees` 只读，因为 iOS 不开放修改事件参与者的接口。

```javascript
identifier: string
title: string
location: string
notes: string
startDate: Date
endDate: Date
isAllDay: bool
attendees: [any]
availability: string
timeZone: string
calendar: Calendar
new CalendarEvent()
addRecurrenceRule(recurrenceRule: RecurrenceRule)
removeAllRecurrenceRules()
save()
remove()
presentEdit(): Promise<CalendarEvent>
static presentCreate(): Promise<CalendarEvent>
static today(calendars: [Calendar]): Promise<[CalendarEvent]>
static tomorrow(calendars: [Calendar]): Promise<[CalendarEvent]>
static yesterday(calendars: [Calendar]): Promise<[CalendarEvent]>
static thisWeek(calendars: [Calendar]): Promise<[CalendarEvent]>
static nextWeek(calendars: [Calendar]): Promise<[CalendarEvent]>
static lastWeek(calendars: [Calendar]): Promise<[CalendarEvent]>
static between(startDate: Date, endDate: Date, calendars: [Calendar]): Promise<[CalendarEvent]>
```

<a id="api-reminder"></a>

### [Reminder](https://docs.scriptable.app/reminder/)

`Reminder` 创建、查询、保存和删除提醒事项。

使用时遵守以下限制。

- 新提醒事项只有在调用 `save()` 后才会写入日历。
- `completedToday()`、`completedThisWeek()` 和 `completedLastWeek()` 按完成时间筛选，不考虑到期日。
- iOS 对部分查询只返回四年时间范围内的结果。

```javascript
identifier: string
title: string
notes: string
isCompleted: bool
isOverdue: bool
priority: number
dueDate: Date
dueDateIncludesTime: bool
completionDate: Date
creationDate: Date
calendar: Calendar
new Reminder()
addRecurrenceRule(recurrenceRule: RecurrenceRule)
removeAllRecurrenceRules()
save()
remove()
static scheduled(calendars: [Calendar]): Promise<[Reminder]>
static all(calendars: [Calendar]): Promise<[Reminder]>
static allCompleted(calendars: [Calendar]): Promise<[Reminder]>
static allIncomplete(calendars: [Calendar]): Promise<[Reminder]>
static allDueToday(calendars: [Calendar]): Promise<[Reminder]>
static completedDueToday(calendars: [Calendar]): Promise<[Reminder]>
static incompleteDueToday(calendars: [Calendar]): Promise<[Reminder]>
static allDueTomorrow(calendars: [Calendar]): Promise<[Reminder]>
static completedDueTomorrow(calendars: [Calendar]): Promise<[Reminder]>
static incompleteDueTomorrow(calendars: [Calendar]): Promise<[Reminder]>
static allDueYesterday(calendars: [Calendar]): Promise<[Reminder]>
static completedDueYesterday(calendars: [Calendar]): Promise<[Reminder]>
static incompleteDueYesterday(calendars: [Calendar]): Promise<[Reminder]>
static allDueThisWeek(calendars: [Calendar]): Promise<[Reminder]>
static completedDueThisWeek(calendars: [Calendar]): Promise<[Reminder]>
static incompleteDueThisWeek(calendars: [Calendar]): Promise<[Reminder]>
static allDueNextWeek(calendars: [Calendar]): Promise<[Reminder]>
static completedDueNextWeek(calendars: [Calendar]): Promise<[Reminder]>
static incompleteDueNextWeek(calendars: [Calendar]): Promise<[Reminder]>
static allDueLastWeek(calendars: [Calendar]): Promise<[Reminder]>
static completedDueLastWeek(calendars: [Calendar]): Promise<[Reminder]>
static incompleteDueLastWeek(calendars: [Calendar]): Promise<[Reminder]>
static completedToday(calendars: [Calendar]): Promise<[Reminder]>
static completedThisWeek(calendars: [Calendar]): Promise<[Reminder]>
static completedLastWeek(calendars: [Calendar]): Promise<[Reminder]>
static allDueBetween(startDate: Date, endDate: Date, calendars: [Calendar]): Promise<[Reminder]>
static completedDueBetween(startDate: Date, endDate: Date, calendars: [Calendar]): Promise<[Reminder]>
static incompleteDueBetween(startDate: Date, endDate: Date, calendars: [Calendar]): Promise<[Reminder]>
static completedBetween(startDate: Date, endDate: Date, calendars: [Calendar]): Promise<[Reminder]>
```

<a id="api-recurrencerule"></a>

### [RecurrenceRule](https://docs.scriptable.app/recurrencerule/)

`RecurrenceRule` 为事件或提醒事项创建按日、周、月、年重复的规则。

使用时遵守以下限制。

- 复杂规则中的日期和位置使用系统日历语义。生成代码时写出具体日期集合和结束条件。

```javascript
static daily(interval: number): RecurrenceRule
static dailyEndDate(interval: number, endDate: Date): RecurrenceRule
static dailyOccurrenceCount(interval: number, occurrenceCount: number): RecurrenceRule
static weekly(interval: number): RecurrenceRule
static weeklyEndDate(interval: number, endDate: Date): RecurrenceRule
static weeklyOccurrenceCount(interval: number, occurrenceCount: number): RecurrenceRule
static monthly(interval: number): RecurrenceRule
static monthlyEndDate(interval: number, endDate: Date): RecurrenceRule
static monthlyOccurrenceCount(interval: number, occurrenceCount: number): RecurrenceRule
static yearly(interval: number): RecurrenceRule
static yearlyEndDate(interval: number, endDate: Date): RecurrenceRule
static yearlyOccurrenceCount(interval: number, occurrenceCount: number): RecurrenceRule
static complexWeekly(interval: number, daysOfTheWeek: [number], setPositions: [number]): RecurrenceRule
static complexWeeklyEndDate(interval: number, daysOfTheWeek: [number], setPositions: [number], endDate: Date): RecurrenceRule
static complexWeeklyOccurrenceCount(interval: number, daysOfTheWeek: [number], setPositions: [number], occurrenceCount: number): RecurrenceRule
static complexMonthly(interval: number, daysOfTheWeek: [number], daysOfTheMonth: [number], setPositions: [number]): RecurrenceRule
static complexMonthlyEndDate(interval: number, daysOfTheWeek: [number], daysOfTheMonth: [number], setPositions: [number], endDate: Date): RecurrenceRule
static complexMonthlyOccurrenceCount(interval: number, daysOfTheWeek: [number], daysOfTheMonth: [number], setPositions: [number], occurrenceCount: number): RecurrenceRule
static complexYearly(interval: number, daysOfTheWeek: [number], monthsOfTheYear: [number], weeksOfTheYear: [number], daysOfTheYear: [number], setPositions: [number]): RecurrenceRule
static complexYearlyEndDate(interval: number, daysOfTheWeek: [number], monthsOfTheYear: [number], weeksOfTheYear: [number], daysOfTheYear: [number], setPositions: [number], endDate: Date): RecurrenceRule
static complexYearlyOccurrenceCount(interval: number, daysOfTheWeek: [number], monthsOfTheYear: [number], weeksOfTheYear: [number], daysOfTheYear: [number], setPositions: [number], occurrenceCount: number): RecurrenceRule
```

<a id="api-contact"></a>

### [Contact](https://docs.scriptable.app/contact/)

`Contact` 读取、创建、修改和删除通讯录联系人。

使用时遵守以下限制。

- 创建、更新和删除联系人时，先调用对应的排队方法，再调用 `Contact.persistChanges()`。
- 邮箱、电话、地址、社交账号、网址和日期属性在更新时会替换整个数组。
- `note` 和 `isNoteAvailable` 已废弃，联系人备注无法读取。

```javascript
identifier: string
namePrefix: string
givenName: string
middleName: string
familyName: string
nickname: string
birthday: Date
image: Image
emailAddresses: [{string: string}]
phoneNumbers: [{string: string}]
postalAddresses: [{string: string}]
socialProfiles: [{string: string}]
note: string
urlAddresses: [{string: string}]
dates: [{string: any}]
organizationName: string
departmentName: string
jobTitle: string
isNamePrefixAvailable: bool
isGiveNameAvailable: bool
isMiddleNameAvailable: bool
isFamilyNameAvailable: bool
isNicknameAvailable: bool
isBirthdayAvailable: bool
isEmailAddressesAvailable: bool
isPhoneNumbersAvailable: bool
isPostalAddressesAvailable: bool
isSocialProfilesAvailable: bool
isImageAvailable: bool
isNoteAvailable: bool
isURLAddressesAvailable: bool
isOrganizationNameAvailable: bool
isDepartmentNameAvailable: bool
isJobTitleAvailable: bool
isDatesAvailable: bool
new Contact()
static all(containers: [ContactsContainer]): Promise<[Contact]>
static inGroups(groups: [ContactsGroup]): Promise<[Contact]>
static add(contact: Contact, containerIdentifier: string)
static update(contact: Contact)
static delete(contact: Contact)
static persistChanges(): Promise
```

<a id="api-contactscontainer"></a>

### [ContactsContainer](https://docs.scriptable.app/contactscontainer/)

`ContactsContainer` 表示一个联系人账户对应的容器。

```javascript
identifier: string
name: string
static default(): Promise<ContactsContainer>
static all(): Promise<[ContactsContainer]>
static withIdentifier(identifier: string): Promise<ContactsContainer>
```

<a id="api-contactsgroup"></a>

### [ContactsGroup](https://docs.scriptable.app/contactsgroup/)

`ContactsGroup` 管理容器中的联系人分组和成员关系。

使用时遵守以下限制。

- 分组的创建、更新和删除也需要调用 `Contact.persistChanges()` 才会写入通讯录。

```javascript
identifier: string
name: string
new ContactsGroup()
static all(containers: [ContactsContainer]): Promise<[ContactsGroup]>
addMember(contact: Contact)
removeMember(contact: Contact)
static add(group: ContactsGroup, containerIdentifier: string)
static update(group: ContactsGroup)
static delete(group: ContactsGroup)
```

<a id="api-location"></a>

### [Location](https://docs.scriptable.app/location/)

`Location` 获取当前位置、调整精度，并把坐标反向解析为地址。

使用时遵守以下限制。

- 首次调用会请求定位权限。用户拒绝后，调用无法取得位置。
- 精度越高，定位通常需要更多时间和电量。

```javascript
static current(): Promise<{string: number}>
static setAccuracyToBest()
static setAccuracyToTenMeters()
static setAccuracyToHundredMeters()
static setAccuracyToKilometer()
static setAccuracyToThreeKilometers()
static reverseGeocode(latitude: number, longitude: number, locale: string): [{string: any}]
```

<a id="api-photos"></a>

### [Photos](https://docs.scriptable.app/photos/)

`Photos` 从相册或相机取得图片，保存图片，并操作最近的照片或截图。

使用时遵守以下限制。

- 相册访问需要用户授权。用户拒绝后，相关调用会失败，必须到系统设置重新授权。
- 读取最近图片的 Promise 在没有匹配图片时会拒绝。
- 删除最近照片或截图会改变用户数据，执行前取得明确确认。

```javascript
static fromLibrary(): Promise<Image>
static fromCamera(): Promise<Image>
static latestPhoto(): Promise<Image>
static latestPhotos(count: number): Promise<[Image]>
static latestScreenshot(): Promise<Image>
static latestScreenshots(count: number): Promise<[Image]>
static removeLatestPhoto()
static removeLatestPhotos(count: number)
static removeLatestScreenshot()
static removeLatestScreenshots(count: number)
static save(image: Image)
```

<a id="api-notification"></a>

### [Notification](https://docs.scriptable.app/notification/)

`Notification` 创建、调度、修改、查询和删除 Scriptable 通知。

使用时遵守以下限制。

- 新通知必须调用 `schedule()`；修改已存在的通知后也要再次调度。
- `removeAllPending()` 会删除所有脚本的待发送通知，无法撤销。
- `Notification.current()` 已废弃，应读取 `args.notification`。

```javascript
identifier: string
title: string
subtitle: string
body: string
preferredContentHeight: number
badge: number
threadIdentifier: string
userInfo: {string: any}
sound: string
openURL: string
deliveryDate: Date
nextTriggerDate: Date
scriptName: string
actions: {string: string}
static current(): Notification
new Notification()
schedule(): Promise
remove(): Promise
setTriggerDate(date: Date)
setDailyTrigger(hour: number, minute: number, repeats: bool)
setWeeklyTrigger(weekday: number, hour: number, minute: number, repeats: bool)
addAction(title: string, url: string, destructive: bool)
static allPending(): Promise<[Notification]>
static allDelivered(): Promise<[Notification]>
static removeAllPending(): Promise
static removeAllDelivered(): Promise
static removePending(identifiers: [string]): Promise
static removeDelivered(identifiers: [string]): Promise
static resetCurrent()
```

<a id="api-device"></a>

### [Device](https://docs.scriptable.app/device/)

`Device` 读取设备型号、系统、屏幕、方向、电量、语言和音量，并可调整屏幕亮度。

使用时遵守以下限制。

- `Device.isUsingDarkAppearance()` 不支持组件；组件主题应使用 `Color.dynamic()`。
- `screenResolution()` 不考虑设备旋转。

```javascript
static name(): string
static systemName(): string
static systemVersion(): string
static model(): string
static isPhone(): bool
static isPad(): bool
static screenSize(): Size
static screenResolution(): Size
static screenScale(): number
static screenBrightness(): number
static isInPortrait(): bool
static isInPortraitUpsideDown(): bool
static isInLandscapeLeft(): bool
static isInLandscapeRight(): bool
static isFaceUp(): bool
static isFaceDown(): bool
static batteryLevel(): number
static isDischarging(): bool
static isCharging(): bool
static isFullyCharged(): bool
static preferredLanguages(): [string]
static locale(): string
static language(): string
static isUsingDarkAppearance(): bool
static volume(): number
static setScreenBrightness(percentage: number)
```


<a id="category-point"></a>

## 绘图、几何、格式化与解析

<a id="api-point"></a>

### [Point](https://docs.scriptable.app/point/)

`Point` 表示二维坐标。

```javascript
x: number
y: number
new Point(x: number, y: number)
```

<a id="api-size"></a>

### [Size](https://docs.scriptable.app/size/)

`Size` 表示二维宽度和高度。

```javascript
width: number
height: number
new Size(width: number, height: number)
```

<a id="api-rect"></a>

### [Rect](https://docs.scriptable.app/rect/)

`Rect` 表示带原点、宽度和高度的矩形。

```javascript
minX: number
minY: number
maxX: number
maxY: number
x: number
y: number
width: number
height: number
origin: Point
size: Size
new Rect(x: number, y: number, width: number, height: number)
```

<a id="api-path"></a>

### [Path](https://docs.scriptable.app/path/)

`Path` 组合直线、矩形、椭圆和曲线，供 `DrawContext` 填充或描边。

```javascript
new Path()
move(point: Point)
addLine(point: Point)
addRect(rect: Rect)
addEllipse(rect: Rect)
addRoundedRect(rect: Rect, cornerWidth: number, cornerHeight: number)
addCurve(point: Point, control1: Point, control2: Point)
addQuadCurve(point: Point, control: Point)
addLines(points: [Point])
addRects(rects: [Rect])
closeSubpath()
```

<a id="api-drawcontext"></a>

### [DrawContext](https://docs.scriptable.app/drawcontext/)

`DrawContext` 在指定尺寸的画布上绘制图片、路径、图形和文字。

使用时遵守以下限制。

- 绘图前必须设置 `size`。
- `strokePath()` 和 `fillPath()` 只作用于最近一次传给 `addPath()` 的路径。
- `setFontSize()` 已废弃，应使用 `setFont()`。

```javascript
size: Size
respectScreenScale: bool
opaque: bool
new DrawContext()
getImage(): Image
drawImageInRect(image: Image, rect: Rect)
drawImageAtPoint(image: Image, point: Point)
setFillColor(color: Color)
setStrokeColor(color: Color)
setLineWidth(width: number)
fill(rect: Rect)
fillRect(rect: Rect)
fillEllipse(rect: Rect)
stroke(rect: Rect)
strokeRect(rect: Rect)
strokeEllipse(rect: Rect)
addPath(path: Path)
strokePath()
fillPath()
drawText(text: string, pos: Point)
drawTextInRect(text: string, rect: Rect)
setFontSize(size: number)
setFont(font: Font)
setTextColor(color: Color)
setTextAlignedLeft()
setTextAlignedCenter()
setTextAlignedRight()
```

<a id="api-dateformatter"></a>

### [DateFormatter](https://docs.scriptable.app/dateformatter/)

`DateFormatter` 按指定格式和地区在 `Date` 与字符串之间转换。

```javascript
dateFormat: string
locale: string
new DateFormatter()
string(date: Date): string
date(str: string): Date
useNoDateStyle()
useShortDateStyle()
useMediumDateStyle()
useLongDateStyle()
useFullDateStyle()
useNoTimeStyle()
useShortTimeStyle()
useMediumTimeStyle()
useLongTimeStyle()
useFullTimeStyle()
```

<a id="api-relativedatetimeformatter"></a>

### [RelativeDateTimeFormatter](https://docs.scriptable.app/relativedatetimeformatter/)

`RelativeDateTimeFormatter` 把两个日期之间的差值格式化为相对时间文本。

```javascript
locale: string
new RelativeDateTimeFormatter()
string(date: Date, referenceDate: Date): string
useNamedDateTimeStyle()
useNumericDateTimeStyle()
```

<a id="api-xmlparser"></a>

### [XMLParser](https://docs.scriptable.app/xmlparser/)

`XMLParser` 通过开始元素、字符、结束元素和错误回调解析 XML。

使用时遵守以下限制。

- 先设置所需回调，再调用 `parse()`。
- `foundCharacters` 可能针对同一个元素调用多次，解析器必须累计文本片段。

```javascript
didStartDocument: fn()
didEndDocument: fn()
didStartElement: fn(string, {string: string})
didEndElement: fn()
foundCharacters: fn()
parseErrorOccurred: fn()
string: string
new XMLParser(string: string)
parse(): bool
```


<a id="coverage"></a>

## 覆盖核验

本索引覆盖 sitemap 中的 60 个 API 页面和 732 个公开成员条目。生成时还读取了首页的 JavaScript 环境、JavaScript 学习提示和社区说明。API 页面数量、页面标题和成员顺序来自官方全文搜索索引。

核对文档更新时，重新读取以下两个入口。

- [官方 sitemap](https://docs.scriptable.app/sitemap.xml)
- [官方全文搜索索引](https://docs.scriptable.app/search/search_index.json)

若 sitemap 增加页面，或搜索索引出现本文件没有的公开成员，应更新完整 API 索引和相关操作规则。
