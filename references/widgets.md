# Scriptable 桌面与锁屏组件实践

先按 [Apple UI 设计规范](apple-ui-design.md) 确定界面目的、信息层级、颜色、文字、控件和验证范围。本文件补充 Scriptable 组件的尺寸、刷新和运行限制。

## 目录

- [先定义组件任务](#先定义组件任务)
- [为每个尺寸单独设计](#为每个尺寸单独设计)
- [安排信息层级](#安排信息层级)
- [处理颜色、文字和布局](#处理颜色文字和布局)
- [处理锁屏组件背景](#处理锁屏组件背景)
- [处理图片清晰度和内存](#处理图片清晰度和内存)
- [处理刷新、网络和缓存](#处理刷新网络和缓存)
- [处理交互和详情页](#处理交互和详情页)
- [提供完整状态](#提供完整状态)
- [组织代码](#组织代码)
- [执行预览矩阵](#执行预览矩阵)
- [审查清单](#审查清单)

## 先定义组件任务

用一句话写出用户看组件时要回答的问题，例如“还剩多少流量”和“数据是否过期”。主指标直接回答第一个问题，状态文字回答第二个问题。

逐项检查准备显示的内容。两项内容表达同一个事实时保留更容易扫读的一项。详情页已有的内容不必全部复制进组件。颜色只能补充含义，不能承担唯一的状态表达。

把组件当作系统按计划生成的静态快照。Scriptable 组件不能提供原生 WidgetKit 的 `Button`、`Toggle` 或 App Intent 交互。需要操作时使用 URL 打开脚本、详情页或目标页面。

## 为每个尺寸单独设计

读取 `config.widgetFamily`，为实际支持的每个值调用独立 builder。不要把同一布局按比例缩放。

| 尺寸 | 内容预算 | 合适结构 |
| --- | --- | --- |
| `small` | 一个主指标、一个状态或上下文 | 主数值加短标签，整个组件使用一个 URL |
| `medium` | 一个主指标、两到四个辅助信息 | 左右分区或上下两层，保留明确阅读顺序 |
| `large` | 一个主任务和有理由存在的详情 | 详情列表、趋势或多个相关指标 |
| `extraLarge` | iPad 上需要同时比较的信息 | 多栏布局；先确认 iPad 实际用途再支持 |
| `accessoryInline` | 一个图片和一段短文字 | “名称 数值”或“状态 数值”；额外元素会被过滤 |
| `accessoryCircular` | 一个短数值、图标或简单进度 | 数字和单位分层；避免完整句子 |
| `accessoryRectangular` | 一个主指标和一到两个短辅助值 | 两到三行或紧凑水平布局 |

`extraLarge` 只在支持的 iPad 上可用。锁屏组件从 iOS 16 起可用。

对不支持的尺寸构建该尺寸能显示的短错误组件。认证缺失、网络失败和数据无效也要按当前 family 渲染，不能把桌面错误布局直接交给锁屏组件。

## 安排信息层级

先放主指标，再放更新状态，最后放解释主指标所需的辅助信息。小组件中每个标签都必须帮助用户解释一个值。

使用以下删除检查。

1. 删除这一行后，用户是否少知道一个不同的事实。
2. 该信息是否适合在组件中快速读取。
3. 点击详情页后才能使用的信息是否占用了组件空间。
4. 同一数值是否同时出现在标题、进度说明和详情行中。

避免只写“更新时间 08:30”。缓存跨天时，这个文本会让旧数据看起来像今天的数据。新鲜数据可显示“08:30 更新”，过期数据应显示“缓存 7月27日 08:30”或可比较的相对时间。

## 处理颜色、文字和布局

桌面组件使用 `Color.dynamic()` 提供浅色和深色值。锁屏组件会受壁纸、系统色调、Always-On 降低亮度和系统单色渲染影响。不要依赖品牌色区分正常、警告和错误。给锁屏圆形或矩形组件评估 `addAccessoryWidgetBackground = true` 的实际效果。

使用系统字体和有限的字号层级。给可能增长的文本设置 `lineLimit` 和 `minimumScaleFactor`，同时限制最坏输入长度。不要依赖很小的缩放因子容纳任意字符串，缩放后仍需在真机可读。

优先使用 stack、弹性 spacer 和内容约束。不要只用 `Device.isPad()` 在两组固定像素宽度中选择。确实需要固定宽度绘制进度条时，集中维护设备或尺寸映射，为未知尺寸提供保守布局，并在不同设备上验证裁切。

复用 `SFSymbol` 和已缩小的图片。避免原尺寸照片、大型背景图、循环创建 `DrawContext` 和保留不再使用的大型 JSON。Scriptable 官方文档明确指出组件进程存在内存限制。

锁屏组件预览只是估计。实际壁纸和色调会改变结果。inline 只保留一个图片和一个文字元素，文案按最窄情况设计。

## 处理锁屏组件背景

先为锁屏组件选择透明或系统自适应背景，不要默认复用桌面组件的自定义背景。`ListWidget` 未设置 `backgroundColor` 时，桌面组件默认使用实色，锁屏组件默认透明。`addAccessoryWidgetBackground = true` 添加随锁屏环境变化的系统自适应背景，默认值为 `false`；它不是桌面背景色、渐变或背景图的替代写法。

按 family 在套用桌面背景前分流，让三种锁屏 builder 提前返回。

```javascript
function createWidget(result, family) {
  if (family === "accessoryInline") {
    return createAccessoryInlineWidget(result)
  }
  if (family === "accessoryCircular") {
    return createAccessoryCircularWidget(result, false)
  }
  if (family === "accessoryRectangular") {
    return createAccessoryRectangularWidget(result, false)
  }

  const widget = new ListWidget()
  applyHomeScreenBackground(widget)
  return buildHomeScreenWidget(widget, result, family)
}

function createAccessoryRectangularWidget(result, useSystemBackground) {
  const widget = new ListWidget()
  widget.addAccessoryWidgetBackground = Boolean(useSystemBackground)
  widget.setPadding(8, 10, 8, 10)
  // 构建只属于 accessoryRectangular 的紧凑内容。
  return widget
}
```

保持以下边界。

- 不对 accessory builder 调用设置桌面 `backgroundColor`、`backgroundImage` 或 `backgroundGradient` 的共享 helper。
- `accessoryInline` 保持无背景，并只安排一个图片和一个短文字。
- 圆形和矩形需要容器时才显式启用 `addAccessoryWidgetBackground`，并在目标锁屏上分别比较 `true` 和 `false`。不要通过设置任意半透明颜色模拟系统背景。
- 第一版优先不设置锁屏文字 `textColor`、图片 `tintColor` 和局部 stack `backgroundColor`，让系统根据壁纸与锁屏色调决定外观。确需品牌色、轨道或状态色时，再逐项加入并验证；状态仍需文字表达。
- 先解析一次 family，再把它显式传给正常、缓存、首次配置、认证失效、离线和数据无效 builder。不要让通用 `createMessageWidget()` 无条件套用桌面背景，否则正常态透明而错误态会突然出现实心矩形。
- `config.runsInWidget` 不能区分桌面和锁屏。真实运行可结合 `config.runsInAccessoryWidget` 与 `config.widgetFamily`；应用内预览或 URL 指定 family 时，以已经解析的 family 参数为准。

分别确定两种有独立容器形态的锁屏 family 内边距。

- `accessoryCircular` 第一版不调用 `setPadding()`，保留 `ListWidget` 的系统默认内边距。共享 helper 曾设置过内边距时，使用 `useDefaultPadding()` 恢复默认值，不要用四个零代替默认值。
- `accessoryRectangular` 可以把 `setPadding(8, 10, 8, 10)` 作为紧凑布局的首版起点；参数依次为 top、leading、bottom、trailing。不要把这组数值复制给圆形或桌面组件。
- 把内边距计入固定进度条、图片和 stack 的宽度预算。文字截断时先删除次要内容、缩短文案或调整布局，不要直接把内边距压到零。
- 透明背景和 `addAccessoryWidgetBackground = true` 的视觉边界不同。切换背景策略后重新检查上下、leading 和 trailing 留白，不要假定同一组数值视觉效果相同。
- 正常、缓存和错误 builder 使用同一 family 的内边距策略。不要让通用错误组件退回桌面的 `setPadding(16, 16, 16, 16)`，造成状态切换时内容跳位或锁屏布局被挤压。

应用内 accessory 预览不会准确复现壁纸和锁屏色调。把透明背景和系统自适应背景分别放到真实锁屏，在浅色、深色、复杂壁纸、不同色调、Clear 外观和 Always-On 下检查。对正常、缓存和每一种错误态执行同一矩阵。

## 处理图片清晰度和内存

Scriptable 官方只说明组件进程存在内存限制，没有承诺固定的图片尺寸上限。社区实测曾出现同一张 PNG 在应用内保持原分辨率、放进实际组件后却被自动缩小到约 500 px 的情况。把 500 px 和失败后尝试的 300 px 当作目标设备上的经验起点，不要写成所有版本和设备都遵守的硬限制。

优先避免让组件加载大图后再裁剪。

1. 最优方案是在服务器或图片生成阶段按日期、内容和组件 family 直接输出最终裁剪图。不要下载包含一周内容的大图集，再让组件每天裁剪其中一块。
2. 无法在服务器处理时，在 `config.runsInApp` 为真时完成解码、裁剪和缩放，把每个 family 需要的最终小图保存到应用与组件都能读取的位置。组件运行时只读取已经处理好的文件。
3. 缓存键包含源图片版本、裁剪区域、family 和目标尺寸。使用写入该路径的同一个 `FileManager` 读取；iCloud 文件在读取前等待下载。
4. 一次只处理一张原图和一个输出，不在组件进程中同时保留原图、图集和多个裁剪结果。

直接读取文件后设置 `backgroundImage` 仍然模糊时，可以尝试 `FileManager.read()`、`Image.fromData()` 和 `DrawContext` 的手动重绘路径。

```javascript
function redrawForWidget(image, maxCanvasLength = 500) {
  if (!image) throw new Error("图片无法解码")

  const screenScale = Math.max(1, Device.screenScale())
  const sourceWidth = image.size.width / screenScale
  const sourceHeight = image.size.height / screenScale
  const scale = Math.min(
    1,
    maxCanvasLength / Math.max(sourceWidth, sourceHeight)
  )
  const width = Math.max(1, Math.round(sourceWidth * scale))
  const height = Math.max(1, Math.round(sourceHeight * scale))

  const context = new DrawContext()
  context.size = new Size(width, height)
  context.respectScreenScale = true
  context.opaque = false
  context.drawImageInRect(image, new Rect(0, 0, width, height))
  return context.getImage()
}
```

`Image.size` 使用像素；`DrawContext.respectScreenScale = true` 会让实际输出尺寸再乘以设备的 2× 或 3× 屏幕缩放。因此示例中的 500 是逻辑画布长边，不是最终文件的严格像素上限。需要严格控制输出像素和内存时，把 `respectScreenScale` 设为 `false` 并直接使用目标像素尺寸；需要 Retina 清晰度时保留 `true`，但把放大后的实际输出计入内存预算。图片不需要透明度时可使用不透明画布。

不要照搬 `newWidth - 1` 之类未经说明的经验偏移。宽高应四舍五入并至少为 1。若 500 的画布在目标设备上仍触发模糊、压缩或内存错误，逐步降到 300，或改用应用内预处理和分图方案。手动重绘仍需先解码原图，不能消除解码原图时的内存峰值。

在应用内预览和真实桌面组件中分别检查同一资源。记录原图、预处理输出和缓存文件的尺寸及字节数，并覆盖冷启动、每个支持的 family、不同屏幕缩放和多次刷新。只有 `presentSmall()` 等应用内预览清晰，不能证明组件进程不会再次压缩。

## 处理刷新、网络和缓存

把 `refreshAfterDate` 视为系统允许再次刷新的最早时间。iOS 或 iPadOS 可以延后刷新，低电量或较少查看组件时更可能延后。

不要声称应用内调用 `Script.setWidget()` 能刷新已经放置的组件。Scriptable 没有公开的 `WidgetCenter` 强制刷新接口。应用内手动刷新应更新缓存并显示最新详情，桌面组件等待系统下一次运行。

设计以下数据路径。

1. 读取并校验缓存，记录缓存版本和 `savedAt`。
2. 在组件时间预算内请求网络。给串行请求分别设置短超时，计算最坏总时长。
3. 网络成功后立即保留结果。缓存写入放在独立 `try` 中，写入失败不能把网络成功改成失败。
4. 网络失败时，根据最大可接受年龄决定是否使用缓存。
5. 返回统一结果，例如 `{ data, source, fetchedAt, error }`。
6. 在所有 family 中显示 `source === "cache"`，圆形空间不足时使用短标记或改变可读文案。

区分认证错误和传输错误。只有 HTTP 401、403 或服务明确的登录失效代码才触发凭据更新。超时、DNS、离线、HTTP 5xx、刷新接口失败和无效 JSON 不应删除或覆盖有效凭据。

让凭据更新经过“输入、规范化、请求验证、保存”四步。验证失败时保留旧钥匙串值。

缓存读取应拒绝以下内容。

- 版本不匹配。
- `savedAt` 缺失、无效或晚于当前时间过多。
- 超过业务允许的最大年龄。
- 主指标缺失、为空、不是有限数或超出允许范围。
- 字段之间的业务约束不成立。

需要多个端点时，确认顺序依赖。刷新端点和读取端点必须串行时，允许刷新失败后继续读取，但要保留刷新失败状态供日志或详情页显示。

## 处理交互和详情页

小型组件和锁屏组件使用 `ListWidget.url` 作为单一点击目标。Scriptable 官方文档只保证中型和大型组件中的文字、日期、图片与 stack 可以使用各自的 `url`。根 `url` 会覆盖组件设置中原有的交互行为。

使用 `URLScheme.forRunningScript()` 生成脚本地址，并编码每个查询参数。把 family、业务对象标识和目标动作作为明确参数传递。不要把 Cookie、令牌和完整个人数据放进 URL。

详情页使用 WebView 时遵守以下规则。

- 删除 `maximum-scale=1` 和 `user-scalable=no`，允许用户缩放。
- 给加载状态和提示区设置 `role="status"` 或 `aria-live`。
- 给进度提供文字值或 ARIA 数值，不能只显示颜色和长度。
- 给模拟按钮的元素提供 `role`、`tabindex`、键盘操作和清楚的可访问名称。
- 使用 `prefers-reduced-motion` 停止非必要动画。
- 远程页面使用 `shouldAllowRequest` 限制主机与协议。自包含页面先在目标设备确认 `loadHTML()` 的内部请求，再决定是否设置过滤。
- 优先使用等待式页面事件。目标 Scriptable 版本出现空白页或长期回调失效时，改用 400 到 750 毫秒的轮询，并在页面关闭后停止。

详情页按钮只保留用户会执行的动作。主卡片已经显示的数值不在详情列表中重复。

## 提供完整状态

每个 family 至少处理以下状态。

| 状态 | 显示要求 |
| --- | --- |
| 正常 | 主指标、数据时间和必要上下文 |
| 使用缓存 | 主指标、清楚的缓存标记和缓存日期 |
| 首次配置 | 当前尺寸可读的配置提示，点击后打开脚本 |
| 认证失效 | 区别于普通离线，点击后进入凭据更新流程 |
| 离线或超时 | 有合格缓存时显示缓存，没有缓存时显示短错误 |
| 数据无效 | 不把缺失字段显示为零，说明数据暂不可用 |
| 零值和极大值 | 进度、单位和文字不溢出 |
| 已到期和今天到期 | 使用按日历日比较的纯函数，覆盖边界测试 |

错误组件也要设置根 URL，让用户能进入修复流程。

## 组织代码

采用以下依赖方向。

```text
运行环境
  -> 设置和凭据
  -> 请求与缓存
  -> 规范化后的领域数据
  -> family builder
  -> Script.setWidget 或应用内预览
```

让 `createWidget(result, family)` 显式接收 family。不要让 builder 隐式读取网络、钥匙串或全局错误状态。

把共享视觉方法限制为文字、间距、图标和简单进度等原子组件。每个 family 保留独立布局函数。删除没有调用的 helper、没有读取的参数和只保存同一个值的配置项。

日期函数接收 `now` 参数，便于测试。按日历日计算“今天到期”和剩余天数，不要用到当天 23:59 的毫秒差再向上取整。

## 执行预览矩阵

在应用内为每个支持的 family 提供显式预览入口，并等待对应方法。

```javascript
const previewByFamily = {
  small: "presentSmall",
  medium: "presentMedium",
  large: "presentLarge",
  extraLarge: "presentExtraLarge",
  accessoryInline: "presentAccessoryInline",
  accessoryCircular: "presentAccessoryCircular",
  accessoryRectangular: "presentAccessoryRectangular"
}

const method = previewByFamily[family]
if (!method) throw new Error(`不支持的组件尺寸：${family}`)
await widget[method]()
```

至少检查以下组合。

- 浅色、深色、系统着色外观和锁屏不同壁纸。
- iPhone 小、中、大组件；支持时检查 iPad 和 extra large。
- 三种锁屏 family 的真机结果、Always-On 和低亮度。
- 最短与最长卡号、单位从 KB 到 GB、0%、1%、99% 和 100%。
- 新鲜数据、跨日缓存、离线、有缓存、无缓存和认证失效。
- 到期前一天、到期当天、到期后一天和设备时区变化。
- 用户从组件点击、从应用直接运行和从 URL Scheme 运行。

## 审查清单

- [ ] 每个支持的 family 有独立布局和独立错误态。
- [ ] 主指标没有被详情行重复。
- [ ] 过期缓存包含日期或明确年龄。
- [ ] `refreshAfterDate` 没有被描述成准时刷新或强制刷新。
- [ ] 网络成功不会因缓存写入失败而丢失。
- [ ] 普通网络错误不会触发凭据覆盖。
- [ ] 新凭据验证成功后才写入钥匙串。
- [ ] 锁屏布局不依赖颜色表达唯一含义。
- [ ] accessory family 在桌面背景 helper 之前分流，正常态和所有错误态使用相同的背景策略。
- [ ] 圆形和矩形的透明背景与 `addAccessoryWidgetBackground` 已在真实锁屏分别比较。
- [ ] 圆形保留系统默认内边距，矩形从独立的紧凑内边距起步；所有状态保持一致。
- [ ] inline 只安排一个图片和一个短文字。
- [ ] 大图按 family 预先裁剪和缩放，真实组件与应用内预览都保持可接受的清晰度。
- [ ] 使用 `DrawContext` 处理图片时已把 `respectScreenScale` 的 2× 或 3× 输出计入尺寸和内存预算。
- [ ] 固定宽度经过设备矩阵验证并有未知尺寸降级。
- [ ] URL 不包含秘密，WebView 导航受主机和协议限制。
- [ ] WebView 允许缩放，状态和进度可被辅助功能读取。
- [ ] 日期边界、零值、极大值和无效输入有纯函数测试。
- [ ] 真机验证覆盖桌面、锁屏、浅色、深色和缓存状态。
