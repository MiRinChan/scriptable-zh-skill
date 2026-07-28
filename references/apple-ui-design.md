# Scriptable Apple UI 设计规范

## 目录

- [确定界面目的](#确定界面目的)
- [建立信息和操作层级](#建立信息和操作层级)
- [适配尺寸和阅读方向](#适配尺寸和阅读方向)
- [处理文字和符号](#处理文字和符号)
- [处理颜色和对比度](#处理颜色和对比度)
- [设计控件和反馈](#设计控件和反馈)
- [使用材质](#使用材质)
- [处理加载、空状态和错误](#处理加载空状态和错误)
- [把可访问性纳入设计](#把可访问性纳入设计)
- [编写界面文字](#编写界面文字)
- [区分组件和 WebView](#区分组件和-webview)
- [执行验证](#执行验证)
- [审查清单](#审查清单)
- [Apple 参考](#apple-参考)

本文件把 Apple Human Interface Guidelines 转换为 Scriptable 桌面组件、锁屏组件和 WebView 可以执行的规则。设计结果应符合平台习惯，同时受 Scriptable API、WebView 和目标系统版本的实际能力约束。

不要用固定圆角、固定蓝色、阴影或透明度判断界面是否符合 Apple 设计。先判断信息、操作、状态和适配是否正确，再决定视觉参数。

## 确定界面目的

开始布局前写出以下内容。

```text
用户打开界面时要回答的问题：
用户最常执行的动作：
用户需要知道的数据时间或来源：
失败后用户可以采取的动作：
界面包含的敏感信息：
```

Apple 当前设计原则可以转换为以下检查。

| 原则 | Scriptable 检查 |
| --- | --- |
| Purpose | 主指标和主要动作直接服务于页面任务。 |
| Agency | 用户能看见当前状态，能取消、关闭或从错误中恢复。 |
| Responsibility | 凭据和个人数据不进入组件参数、URL、日志或不必要的 DOM。 |
| Familiarity | 使用系统字体、SF Symbols、原生 HTML 控件和常见操作顺序。 |
| Flexibility | 支持不同组件尺寸、屏幕尺寸、文字大小、外观和输入方式。 |
| Simplicity | 每项内容回答不同问题，每个按钮执行用户会使用的动作。 |
| Craft | 对长文本、边界数据、错误状态和真机表现逐项验证。 |
| Delight | 视觉反馈帮助理解操作结果，装饰不降低内容可读性。 |

## 建立信息和操作层级

先安排内容，再安排外观。

1. 第一层显示页面名称、主指标或当前任务。
2. 第二层显示解释主指标所需的单位、时间、来源或状态。
3. 第三层显示详情和低频操作。
4. 主要操作使用最明显的控件样式。一个视图通常只保留一个主要操作。
5. 破坏性操作不能使用主要操作样式，并提供安全的取消路径。

逐项执行删除检查。

1. 删除后，用户是否少知道一个不同的事实。
2. 同一事实是否已由主数值、进度、标题或详情行表达。
3. 该内容是否适合当前尺寸和使用时长。
4. 该控件是否改变状态、导航或执行任务。
5. 装饰是否帮助区分层级、状态或可操作性。

控件应与内容可以直接区分。不要让普通文字使用与按钮相同的颜色和形状，也不要让内容卡片看起来可以点击但没有动作。

## 适配尺寸和阅读方向

按照当前容器布局，不按照设备名称选择固定像素。

- 组件为每个 `widgetFamily` 使用独立布局。
- WebView 使用流式布局、合理的最大内容宽度和安全区域。
- 主要内容按阅读顺序放在顶部和 leading 一侧。
- CSS 使用逻辑属性时优先考虑 `margin-inline`、`padding-inline` 和 `text-align: start`。
- 长文本允许换行或有意义的截断，数值和单位不能互相覆盖。
- 横屏、分屏和 iPad 宽屏不能只把手机布局等比放大。
- 支持 RTL 时翻转方向相关布局，不翻转照片、播放图标和不表示方向的图形。

全屏 WebView 使用 `env(safe-area-inset-*)` 避开 Dynamic Island、圆角和 Home 指示条。页面不能横向滚动才能读完主要内容。

## 处理文字和符号

使用系统字体建立有限而稳定的层级。WebView 使用 `-apple-system`，Scriptable 组件使用 `Font.systemFont()`、`Font.mediumSystemFont()` 和 `Font.semiboldSystemFont()` 等系统接口。

- 正文优先采用接近系统默认的字号。iOS 和 iPadOS 的 HIG 默认正文为 17 pt，普通界面文字不应低于 11 pt。
- 小字号避免 Light、Thin 和 Ultralight 字重。
- 主数值可以放大，但单位、标签和数据时间仍需可读。
- 不用连续多个字号和字重表达同一层级。
- 文字放大后保留原有层级，主要内容不能比次要内容更早截断。
- 有语义的符号随文字或辅助功能字号一起放大。

优先使用与当前系统一致的 SF Symbols。图标不能替代不明确的按钮名称。图标表达状态时，同时提供文字或可访问名称。

## 处理颜色和对比度

颜色承担语义时保持一致。强调色表示可操作性后，不再用同一颜色装饰不可操作文字。

- 组件使用 `Color.dynamic()` 提供浅色和深色值。
- WebView 使用 `color-scheme` 和 `prefers-color-scheme`。
- 自定义颜色分别检查浅色、深色和增强对比度。
- 状态不能只靠红、黄、绿或进度条长度表达。
- 相邻背景、卡片、按钮和遮罩在两种外观中保持可见边界。
- 普通文字与背景的对比度至少达到 4.5:1；大号文字或粗体至少达到 3:1。
- 锁屏组件还要检查系统着色、Clear 外观、壁纸和 Always-On 降低亮度后的结果。

Scriptable 不能直接取得 UIKit 的全部语义颜色和增强对比度变体。自定义值需要通过文字、形状、间距和真机验证补足这一限制。

## 设计控件和反馈

控件名称描述执行结果，优先使用动词和对象，例如“刷新数据”“打开运营商网页”和“更新 Cookie”。

- WebView 优先使用 `<button>`、`<a>`、`<input>` 和 `<select>` 等原生元素。
- 常用触控目标至少提供约 44 × 44 pt 的可点击区域。
- 自定义按钮提供默认、按下、聚焦、禁用和执行中状态。
- 按钮禁用时仍保留可读名称，并说明正在执行的动作。
- 同一组并列按钮保持尺寸和对齐一致。
- 取消操作使用“取消”。确认操作写出结果，避免只写“是”“否”或含义不清楚的“确定”。
- 错误提示中的下一步使用实际按钮名称。

组件是静态快照。没有原生 WidgetKit 交互能力时，使用明确 URL 打开脚本或详情页，不绘制无法操作的开关和按钮。

## 使用材质

Apple 将 Liquid Glass 用于控件和导航层，将标准材质用于内容层。Scriptable 中遵守以下边界。

- 不给每张内容卡片添加玻璃效果。
- 不把 `backdrop-filter: blur()` 描述为系统 Liquid Glass。
- WebView 只在临时遮罩、浮动操作或导航层确有需要时使用透明和模糊。
- 使用透明或模糊时，为 Reduce Transparency、增强对比度和不支持滤镜的环境提供不透明背景。
- 系统已经控制组件背景、着色或 Clear 外观时，不叠加模拟系统表面的装饰。
- 视觉层级依靠内容顺序、间距、文字和控件状态成立，不能依赖材质效果才能成立。

Liquid Glass 会随系统版本和辅助功能设置改变。Scriptable WebView 无法完整复现 UIKit 或 SwiftUI 的材质行为。

## 处理加载、空状态和错误

首次展示尽快提供可读内容。已有缓存时先显示缓存和数据时间，再执行网络更新。

- 已知进度使用确定进度，未知时长使用不确定进度。
- 加载超过短暂等待时显示动作名称，例如“正在刷新数据”。
- 单项操作只禁用相关控件，全页阻塞操作才使用全页忙碌层。
- 操作失败后保留已有内容，恢复控件，并显示用户可以执行的下一步。
- 空状态说明为什么没有内容以及如何获得内容。
- 缓存状态直接写出“缓存”和可比较的日期。
- 不用只显示旋转动画的空白页。

弹窗只用于需要立即处理的重要信息、确认或输入。普通状态和非阻塞错误放在相关内容附近。

## 把可访问性纳入设计

在建立布局和控件时同时处理可访问性。

- 保留系统文字缩放和 WebView 页面缩放。
- VoiceOver 顺序与视觉阅读顺序一致。
- 每个控件具有名称、角色、值和当前状态。
- 图表和进度说明业务含义和数值，不描述颜色或几何外观。
- 颜色、声音、动画和手势都不能成为唯一的信息或操作方式。
- 自定义手势提供按钮等可见替代方式。
- 使用 `prefers-reduced-motion` 停止非必要动画。
- 自动消失的提示不能承载必须阅读或执行的信息。
- 键盘操作覆盖 Tab、Enter、空格和可见焦点。

## 编写界面文字

先说明当前情况，再说明可执行动作。

- 标题说明内容或结果，不写泛化标题。
- 按钮说明动作结果，不解释实现过程。
- 错误信息说明失败的动作、已保留的状态和下一步。
- 权限或凭据请求说明用途和保存位置。
- 同一对象、状态和动作在组件、WebView、弹窗和日志中使用同一名称。
- 删除不影响含义的形容词、重复标签和按钮说明。

## 区分组件和 WebView

| 设计项 | 桌面与锁屏组件 | WebView |
| --- | --- | --- |
| 主要任务 | 一眼读取一个状态或数值 | 查看详情并执行少量相关动作 |
| 布局 | 按 family 独立构建 | 流式布局并适配安全区域 |
| 交互 | URL 打开脚本或详情页 | 原生 HTML 控件和受限事件桥 |
| 文字缩放 | 由组件布局和系统渲染共同决定 | 保留页面缩放并测试文字放大 |
| 外观 | 检查系统着色、Clear、壁纸和 Always-On | 检查浅色、深色、对比度和 reduced motion |
| 材质 | 服从系统组件表面 | 不用 CSS 声称复现系统材质 |
| 状态 | 快照中直接标记缓存或错误 | 操作中、成功、失败和恢复均有反馈 |

## 执行验证

至少覆盖以下组合。

- iPhone 窄屏、横屏和支持时的 iPad 宽屏。
- 每个支持的桌面和锁屏 `widgetFamily`。
- 浅色、深色、增强对比度、Reduce Transparency 和 Reduce Motion。
- 系统默认文字、较大文字、页面缩放和最长业务文本。
- VoiceOver、键盘和触控。
- 正常、加载、缓存、空数据、离线、认证失效和无效数据。
- 主操作成功、失败、连续触发和操作中关闭页面。
- 锁屏壁纸、系统着色、Clear 外观和 Always-On。

普通浏览器预览只能检查 WebView 的 HTML、CSS 和 DOM。最终结论必须来自目标 iPhone 或 iPad 上的 Scriptable 运行结果。

## 审查清单

- [ ] 页面任务、主指标和主要动作已经写明。
- [ ] 每项内容和控件通过删除检查。
- [ ] 控件与内容可以直接区分。
- [ ] 组件按 family 设计，WebView 不依赖固定设备宽度。
- [ ] 使用系统字体、有限字号层级和语义明确的 SF Symbols。
- [ ] 浅色、深色和增强对比度均可读。
- [ ] 状态不依赖颜色、动画、声音或手势单独表达。
- [ ] 常用触控目标约为 44 × 44 pt 或更大。
- [ ] 按钮名称说明结果，并具有完整交互状态。
- [ ] Liquid Glass 没有被用于内容层或用 CSS 冒充。
- [ ] 加载时已有可读内容，失败后保留已有数据。
- [ ] VoiceOver、文字放大、键盘、Reduce Motion 和 Reduce Transparency 已检查。
- [ ] 真机验证覆盖实际组件外观和 WebView 生命周期。

## Apple 参考

- [Design principles](https://developer.apple.com/design/human-interface-guidelines/design-principles)
- [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines)
- [Layout](https://developer.apple.com/design/human-interface-guidelines/layout)
- [Typography](https://developer.apple.com/design/human-interface-guidelines/typography)
- [Color](https://developer.apple.com/design/human-interface-guidelines/color)
- [Dark Mode](https://developer.apple.com/design/human-interface-guidelines/dark-mode)
- [Buttons](https://developer.apple.com/design/human-interface-guidelines/buttons)
- [Loading](https://developer.apple.com/design/human-interface-guidelines/loading)
- [Alerts](https://developer.apple.com/design/human-interface-guidelines/alerts)
- [Accessibility](https://developer.apple.com/design/human-interface-guidelines/accessibility)
- [Materials](https://developer.apple.com/design/human-interface-guidelines/materials)
- [Widgets](https://developer.apple.com/design/human-interface-guidelines/widgets)
- [Writing](https://developer.apple.com/design/human-interface-guidelines/writing)
