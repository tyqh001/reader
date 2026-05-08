# Reader 前端体验优化计划

## Context

当前 Reader 项目（自托管 Web 阅读服务器）的前端基于 Vue 2.6 + Element UI + Vuex，存在以下问题：

- **阅读体验差**：界面像管理后台而非阅读器，排版粗糙、工具栏笨重、过渡生硬
- **原作者广告**：包含微信公众号二维码（"假装大佬"）和 TG 频道推广链接
- **无付费限制**：经代码审查，项目中不存在实际收费功能。`TocRule.kt` 的 `isVip` 字段是书源抓取规则（用于识别源站的 VIP 章节），这是功能性代码，不应移除

核心目标：**让阅读体验像阅读器，而不是管理后台。**

---

## Phase 1: 移除原作者推广内容 (快速见效)

### 删除文件
- `web/src/components/MPCode.vue` — 微信二维码弹窗，标题"关注公众号【假装大佬】"（51行）
- `web/src/assets/imgs/mpcode.jpg` — 二维码图片资产

### 修改 App.vue
移除 5 处：
1. 模板中 `<MPCode v-model="showMPCodeDialog" />`（第75行）
2. `import MPCode from "./components/MPCode.vue"`（第115行）
3. `components` 注册中的 `MPCode`（第194行）
4. `data()` 中的 `showMPCodeDialog: false`（第225行）
5. `beforeMount()` 中的 `eventBus.$on("showMPCodeDialog", ...)` 监听器（第338-339行）

### 修改 Index.vue
2 处删除：
1. "其它"区块下的两个推广按钮（第409-424行）："关注公众号【假装大佬】" 和 "加入TG频道【假装大佬】"
2. `showMPCode()` 和 `joinTGChannel()` 方法（第2748-2753行）

### 验证
- `yarn build` 通过，无 MPCode 或"假装大佬"引用
- 设置面板不再显示推广按钮

---

## Phase 2: 设计令牌（Design Tokens）基础设施

### 新建文件

**`web/src/styles/tokens.css`** — CSS 自定义属性作为设计令牌：
- 排版令牌：`--font-serif`, `--font-sans`（CJK 优化字体栈）
- 阅读布局令牌：`--reader-font-size`（responsive via clamp）, `--reader-line-height`, `--reader-content-width`
- 间距尺度：`--space-xs` (4px) 到 `--space-2xl` (48px)
- 过渡令牌：`--duration-fast` (150ms), `--duration-normal` (300ms), `--ease-out-expo`
- 日间主题颜色：`--color-body-bg: #f5f0e8`（羊皮纸色）, `--color-content-bg: #faf8f3`, `--color-text-primary: #2c2416`（暖深棕）

**`web/src/styles/themes.css`** — 多主题定义：
- `[data-theme="day"]` — 继承 `:root` 默认（暖羊皮纸色）
- `[data-theme="night"]` — 深蓝灰色调（非纯黑），`--color-text-primary: #c8c3b8`（降低长时间阅读的眼疲劳）
- `[data-theme="sepia"]` — 暖琥珀/护眼色
- `[data-theme="cool"]` — 中性冷灰色
- `@media (prefers-color-scheme: dark)` — 自动切换夜间主题
- `@media (prefers-reduced-motion)` — 尊重无障碍偏好

**`web/src/styles/reading.css`** — 阅读专用样式：
- `.reading-content` 类：`font-family: var(--font-serif)`, `text-rendering: optimizeLegibility`
- `::selection` 使用主题颜色
- `scroll-behavior: smooth`
- 响应式内容宽度：`max-width: var(--reader-content-width)` 替代固定 670px
- 隐藏滚动条（保持现有的 `::-webkit-scrollbar { display: none }` 模式）

### 修改 main.js
- 导入 `tokens.css` 和 `themes.css`

### 修改 App.vue
- `setTheme(isNight)` 方法改用 `document.documentElement.setAttribute('data-theme', ...)`
- **保留**现有 `.night-theme` Stylus 块作为 Element UI 组件的过渡桥接

### 验证
- CSS 自定义属性在 DevTools 中可见
- `data-theme` 属性正确切换
- Element UI 暗色模式组件仍正常渲染

---

## Phase 3: 从 Reader.vue 提取阅读组件

**目标：Reader.vue 从 3837 行缩减到 ~2500 行。**

### 新建 5 个子组件

**1. `web/src/components/reading/ReadingToolbar.vue`** (~200行)
- 提取左侧 `.tool-bar`（原第12-136行）
- 用自定义滑出面板替代 Element UI `<el-popover>`
- 面板从左侧滑入，使用 `transform: translateX` + `opacity`（仅合成器属性）
- 面板背景使用 `--color-overlay` + `backdrop-filter: blur(4px)`
- 工具栏在滚动时自动隐藏，点击屏幕时重新出现
- 使用插槽（slots）接收 BookShelf, BookSource, PopCata, ReadSettings 子组件

**2. `web/src/components/reading/ReadingActions.vue`** (~120行)
- 提取右侧浮动操作按钮（原第138-211行）
- 点击时淡入（opacity 过渡），3秒无操作后自动隐藏
- 按钮使用主题强调色的圆形背景
- 桌面端 hover 显示 tooltip，移动端长按显示

**3. `web/src/components/reading/ReadingProgress.vue`** (~80行)
- 提取进度条和阅读进度文本（原第213-225行 + 第263-266行）
- 重新设计为固定在顶部/底部的细线条（2px高）
- hover/点击时展开显示章节名和百分比
- mini-interface 模式显示更突出的滑块

**4. `web/src/components/reading/ReadingContent.vue`** (~250行)
- 提取 `.chapter` 内容区域（原第385-499行）
- 使用令牌驱动的排版
- 章节加载时添加平滑淡入动画
- 滑动模式：使用 CSS `transform: translateX` 过渡翻页
- 内容顶部添加细微渐变，增加视觉深度
- 保持所有内容类型支持：text, epub iframe, CBZ, cartoon, audio

**5. `web/src/components/reading/SpeechBar.vue`** (~150行)
- 提取 TTS/朗读栏（原第285-383行）
- 底部锚定的滑动条，使用 `transform: translateY` 过渡
- 保留所有语音控制功能

### 集成到 Reader.vue
- 导入并注册 5 个新组件
- 用组件标签 + props/events 替换提取的模板部分
- 移除提取的 Stylus 块（约 500+ 行）
- 保持 `<script>` 部分基本完整
- 逐个组件集成，每步验证

### 验证
- 每个滑出面板开关流畅
- 所有工具面板功能正常
- 浮动按钮正确显示/隐藏
- 阅读进度准确更新
- 语音功能正常
- `yarn build` 无错误

---

## Phase 4: 排版与阅读布局优化

### 改进阅读排版
- 默认阅读字体改为宋体/衬线字体 `var(--font-serif)`（CJK 阅读最佳实践）
- 字体大小响应式：`--reader-font-size: clamp(16px, 1rem + 0.5vw, 22px)` 替代固定 18px
- 添加 `text-rendering: optimizeLegibility` 和 `-webkit-font-smoothing: antialiased`
- 阅读宽度响应式：`--reader-content-width: min(720px, 100vw - 32px)` 替代固定 670px
- `Content.vue` 中引用 CSS 变量替代硬编码值

### 添加翻页过渡动画
- 滑动模式：内容容器 CSS `transition: transform var(--duration-normal) var(--ease-out-expo)`
- 内容淡入：章节加载时 opacity 过渡
- 滚动模式：`scroll-behavior: smooth`
- 动画包裹在 `@media (prefers-reduced-motion: no-preference)` 中

### 验证
- 中文字体在所有选项中正确渲染
- 混合 CJK + Latin 文字间距和换行正确
- 内容宽度响应式
- 翻页动画平滑

---

## Phase 5: 主题系统完善

### Reader.vue 内联样式迁移到 CSS 变量
- 将 `bodyTheme`, `readBarTheme` 等计算属性中的内联 `:style` 替换为 CSS 类 + 变量引用
- 示例：不再用 `:style="bodyTheme"`（JS 计算），改用 `class="reader-body"` 配合 `.reader-body { background: var(--color-body-bg); }`
- **保留** `currentThemeConfig` getter 用于背景图片/纹理主题（索引0-5的主题仍有 `url()` 图片）

### 新增主题变体
- `config.js` 中添加"内置暖黄"、"内置冷灰"主题条目
- `ReadSettings.vue` 主题选择器扩展显示新选项
- `themes.css` 确保 `sepia` 和 `cool` 主题有完整令牌定义

### 验证
- 所有 4 个主题正确渲染
- 纹理/图片主题（背景图）仍然正常工作
- 主题切换无闪烁（FOUC）
- 自动主题（跟随系统）正常工作

---

## Phase 6: 触摸、移动端与细节打磨

### 合并触摸/滑动处理
- 整合 Reader.vue 中分散的 touch 事件处理器
- 阅读区域添加 `touch-action: pan-y` 防止水平滚动干扰
- 翻页滑动阈值增加到 50px（减少误翻页）
- 设备支持时添加 `navigator.vibrate(10)` 触觉反馈
- 活动滑动时在边缘显示微妙的视觉翻页指示器

### 移动端优先响应式调整
- 移动端 (< 768px)：全宽阅读布局，无侧边栏
- 移动端工具栏改为底部卡片式
- 调整移动端字体大小：覆盖 `--reader-font-size` 为 `clamp(16px, 4.5vw, 20px)`
- 安全区域适配：`padding-top: env(safe-area-inset-top)`, `padding-bottom: env(safe-area-inset-bottom)`
- 移除移动端的固定 `width: 670px`

### 验证
- 移动端滑动翻页可靠
- 工具栏点击显示，无操作自动隐藏
- 安全区域在刘海屏手机上正确
- 小屏幕字体可读
- 桌面布局不受影响

---

## Phase 7: 最终清理与集成测试

### 替换剩余 iconfont 字符
- Reader.vue 和 ReadSettings.vue 中替换 iconfont Unicode 码为 Element UI 图标

### 全面构建验证
- `yarn build` 零错误零警告
- `yarn lint` 通过
- 手动测试清单：
  - [ ] 所有 7 种阅读模式：文字滚动、文字滑动、epub、cbz、cartoon、audio、mini-interface
  - [ ] 阅读位置保存和恢复
  - [ ] 目录导航、章节切换保持滚动位置
  - [ ] 书源切换保持阅读位置
  - [ ] 所有 4 种主题切换
  - [ ] 自动主题跟随系统
  - [ ] localStorage 配置向后兼容
  - [ ] 搜索、添加到书架流程
  - [ ] 书签、全文搜索、书籍信息

---

## 风险与缓解

| 风险 | 严重性 | 缓解措施 |
|------|--------|----------|
| Reader.vue 提取破坏组件间 `$refs`/`$parent` 调用 | **高** | 逐个提取，提取前审计所有 `$refs` 和 `$parent` 使用，改用显式 props/events |
| CSS 变量迁移导致 Element UI 暗色主题回归 | **中** | 保留现有 `.night-theme` Stylus 块作为桥接 |
| 触摸/滑动修改在特定 iOS/Android 版本失效 | **中** | 真机测试，保留旧逻辑作为参考 |
| 主题图片纹理支持因 CSS 变量迁移而破坏 | **中** | 仅迁移纯色主题属性，图片主题保留现有逻辑 |
| Content.vue render 函数修改破坏 EPUB/CBZ 显示 | **中** | 最小化 Content.vue 修改，排版修改仅在 CSS 层面 |

---

## 文件变更总览

| 文件 | 操作 | 预估行数变化 |
|------|------|-------------|
| `src/components/MPCode.vue` | **删除** | -51 |
| `src/assets/imgs/mpcode.jpg` | **删除** | -- |
| `src/App.vue` | 修改 | -25, +5 |
| `src/views/Index.vue` | 修改 | -25 |
| `src/styles/tokens.css` | **新建** | +80 |
| `src/styles/themes.css` | **新建** | +120 |
| `src/styles/reading.css` | **新建** | +200 |
| `src/main.js` | 修改 | +2 |
| `src/components/reading/ReadingToolbar.vue` | **新建** | +200 |
| `src/components/reading/ReadingActions.vue` | **新建** | +120 |
| `src/components/reading/ReadingProgress.vue` | **新建** | +80 |
| `src/components/reading/ReadingContent.vue` | **新建** | +250 |
| `src/components/reading/SpeechBar.vue` | **新建** | +150 |
| `src/views/Reader.vue` | 重构 | -1400, +300 |
| `src/plugins/config.js` | 修改 | +60 |
| `src/components/ReadSettings.vue` | 修改 | -10, +10 |
| `src/components/Content.vue` | 修改 | -5, +15 |

**净效果：** Reader.vue 从 3837 行降至 ~2500 行（-35%），阅读体验从管理后台风格转变为沉浸式阅读器。

---

## 实施顺序建议

**第一天：** Phase 1（去掉广告）+ Phase 2（令牌基础设施）  
**第二天：** Phase 3（提取组件，重点）  
**第三天：** Phase 4（排版优化）+ Phase 5（主题完善）  
**第四天：** Phase 6（移动端打磨）+ Phase 7（清理测试）
