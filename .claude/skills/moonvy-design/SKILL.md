---
name: moonvy-design
description: 将月维设计标注链接转为前端代码，使用 Playwright 浏览器自动化提取精确设计数据（图层位置、颜色、字体等）
version: 1.1.0
---

# 月维设计图转代码技能

通过 Playwright 浏览器自动化，从月维设计标注页提取精确设计数据，生成像素级还原的前端代码。

## 激活条件

- 用户提供月维项目链接（moonvy.com/project/...）
- 用户说"转换月维设计"或"月维设计图转代码"
- 用户要求从月维设计图实现页面

## 架构

```
月维链接 → Playwright MCP → 设计 JSON + 快照图片 → 代码生成
```

### 数据流

1. **Playwright MCP** 打开月维标注页
2. **browser_evaluate** 从 `fs.moonvy.com` 提取设计 JSON
3. **下载快照图片** 从 JSON 中的 `snapshot` + `images` 映射获取（比浏览器截图更干净）
4. **下载图片资源** 从 `images` 对象批量下载到本地
5. **代码生成** 使用精确 JSON 数据 + 项目技术栈检测

### 关键数据来源

- **首选**：`fs.moonvy.com/{hash}.json` 中的 JSON 文件（每页约 40-80KB）
- **运行时**：标注页上的 `window.genomeSpec.genome` 对象
- **设计截图**：JSON 中 `pages[0].snapshot` → `images[hash].url`（干净的纯设计图，无月维 UI）
- **图片资源**：JSON 中 `images{}` 对象包含所有图片的 URL 映射
- **切图资源**：编组图层的 `slices` 字段引用 `images` 中的图片（如 banner 切图）
- **浏览器截图**：仅在 JSON 快照不可用时使用 `browser_take_screenshot`

## 输出目录结构

**设计资源**放在 `output/{页面名称}/` 下：

```
output/{页面名称}/
├── design-data.json          # 完整设计 JSON（格式化）
├── design-screenshot.png     # 设计截图（来自 snapshot）
├── diff-*.png                # UI 对比验证截图（临时，可批量清理）
├── calc_spacing.py           # 间距计算脚本（可保留供复检）
└── assets/                   # 图片资源（下载 + 从快照裁剪）
    ├── banner.png            # 切图资源（来自 slices）
    ├── card-1.png            # 裁剪的图层图片
    └── ...
```

**所有生成的文件（包括对比验证截图、Playwright 截图）必须放在 `output/{页面名称}/` 下，禁止放在项目根目录或上级 output 目录。** Playwright `browser_take_screenshot` 的 `filename` 参数必须以 `output/{页面名称}/` 为前缀，例如 `output/详情页/diff-hover.png`。

**生成的代码文件**遵循项目现有目录结构：
- Vue → `src/views/{PageName}.vue`
- React → `src/pages/{PageName}/index.tsx`
- Next.js → `app/{page-name}/page.tsx`
- 无项目 → `output/{页面名称}/{页面名称}.html`

## 月维数据结构（概要）

每个设计页 JSON 包含：
- `pages[]` — 画板/页面对象，含完整图层树
- `images{}` — SHA256 → URL 映射，所有图片资源
- `meta` — 来源（figma/sketch）、设计平台、文件名
- `variables{}` — 设计变量
- `styles{}` — 填充/效果/文字样式定义

### 图片资源获取方式

1. **页面快照**：`pages[0].snapshot` → 在 `images` 中查找 URL → 这是完整设计图的截图
2. **切图资源**：图层 `slices` 字段 → `{max: {id: "hash"}, base: {id: "hash"}}` → 在 `images` 中查找 URL
3. **图层填充图片**：`fills: [{type: "image"}]` → 部分图片没有单独的 URL，需要从快照中截取

### 可用的图层属性

每个图层都有：
- `rect: {x, y, w, h}` — 精确位置和尺寸
- `type: "artboard"|"group"|"text"|"layer"` — 图层类型
- `fills[]` — 颜色/渐变/图片填充，含精确 RGB 值
- `strokes[]` — 边框样式
- `effects[]` — 阴影、模糊
- `borderRadius` — 圆角半径
- `blend: {opacity, visible}` — 不透明度和可见性
- `textbox`（仅文字图层）— 文字内容、字体、字号、字重、行高、颜色、对齐方式
- `slices`（部分编组）— 切图资源引用

## 辅助文件

- `extraction-guide.md` — 详细 JSON 结构和提取脚本
- `tech-detection.md` — 项目技术栈自动检测规则
- `code-gen-strategies.md` — 数据到 CSS 映射及各框架策略

## 登录要求

月维需要认证。技能通过以下方式处理：
1. 导航到 URL
2. 如果重定向到登录页 → 提示用户在 Playwright 浏览器中手动登录
3. 登录后 → 重新导航到设计链接
4. 会话在 Playwright 浏览器实例中保持

## 代码生成关键步骤

在 `code-gen-strategies.md` 中定义了必须按顺序执行的 6 步流程：

1. **全量图层扫描** — 用 Python 提取完整图层树，确保不遗漏任何元素
2. **矢量图标识别与裁剪** — 自动识别图标类图层，从快照裁剪为 PNG
3. **精确间距计算** — 从 rect 坐标数学计算所有 margin/gap/padding，禁止目测估计
4. **图层分类与处理** — 对每个图层确定处理方式（文字/图片/图标/色块/渐变）
5. **代码生成与自检** — 生成代码后全量对照检查
6. **交互元素识别与实现** — 识别按钮/链接/输入等交互元素，添加 hover/click/focus 行为

## 已知限制

1. 月维数据中该设计没有 auto-layout（flexbox）信息 — 需从 rect 位置推断布局
2. 字体族名是原始设计工具名称（如 "Source Han Sans CN"）— 需要 web-safe 回退字体
3. 颜色值是浮点数（51.000001）— 需取整
4. 坐标可能是画布相对或页面相对 — 需通过背景图层坐标判断坐标系类型
5. 文字段落支持单个文字节点内的混合样式
6. 部分图层图片（`fills: [{type: "image"}]`）没有单独的 URL，只能从页面快照中截取
7. 矢量图标没有 SVG path 数据 — 只能从快照裁剪为 PNG 位图
8. 手动关闭 Playwright 浏览器后需通过 `/mcp` 重启 Playwright MCP 服务

## 历史问题与防范

### 间距不准确（已修复）

**原因**：使用目测估计而非坐标计算
**防范**：所有间距必须通过 `rect2.y - (rect1.y + rect1.h)` 或 `rect2.x - (rect1.x + rect1.w)` 精确计算，生成前输出间距计算表

### 图标遗漏（已修复）

**原因**：图层扫描不完整，遗漏了小尺寸图标层
**防范**：Step 1 全量图层扫描确保不遗漏；图标识别规则覆盖 artboard 小容器和 group 小编组

### SVG 溢出（已修复）

**原因**：内联 SVG 未设置 width/height，浏览器默认 300×150
**防范**：所有内联 SVG 必须设置明确的 width/height 属性或 CSS 尺寸约束

### Banner 文字重复（已修复）

**原因**：slices 切图已包含背景+文字，代码又叠加了 HTML 文字
**防范**：切图去重规则 — 有 slices 的编组不生成子图层 HTML

### 文字颜色使用通用值而非精确提取（已修复）

**原因**：用 `--color-text-secondary: #666666` 统一所有卡片文字颜色，没有逐个提取 `textbox.segments[].fills[].color`
**防范**：每个文字图层的颜色必须从 `segments[].fills[].color` 精确提取，不同图层可能有不同颜色（如卡片详情文字是 #909090 灰色，专利费是 #f20303 红色）

### 文字换行与设计不符（已修复）

**原因**：文字区域宽度不足，导致长文本换行。设计中文字区域的 `rect.w` 是文字实际渲染宽度，需确保 CSS 容器宽度与之匹配
**防范**：当文字图层的 `rect.w` 小于容器可用宽度时，使用 `white-space: nowrap; overflow: hidden; text-overflow: ellipsis` 防止意外换行

### 容器 padding 只算了 top/left，遗漏 bottom/right（已修复）

**原因**：间距计算脚本只计算了容器的 padding-top 和 padding-left，没有计算 padding-bottom。代码生成时 `padding-bottom` 使用了不准确的默认值（如 16px），导致卡片底部元素紧贴或间距错误。
**防范**：
1. 对每个卡片/容器，找出其内部最底部的子元素（y+h 最大的那个）
2. 计算 padding-bottom = `(container.y + container.h) - (last_child.y + last_child.h)`
3. padding-right 同理：`(container.x + container.w) - (rightmost_child.x + rightmost_child.w)`
4. 间距计算脚本必须输出四边 padding 值，不能只输出 top 和 left

### 区块间距遗漏（已修复）

**原因**：间距计算脚本计算了卡片内部的子元素间距，但未计算卡片与卡片之间、最后一个卡片与 footer 之间的间距。导致卡片底部紧贴 footer。
**防范**：
1. 列出所有主要区块（header、卡片、footer 等）的完整 rect
2. 逐对计算相邻区块间距：`next_section.y - (prev_section.y + prev_section.h)`
3. 确保最后一个内容区块到 footer 的间距也被计算和输出

### Flex 子元素间距不一致时误用统一 gap（已修复）

**原因**：多个子元素在同一 flex 容器中，但它们的间距并不相同。例如"专利费:8%"和"获取授权许可"之间有 347px 间距，而"获取授权许可"和"收藏"之间只有 28px。代码生成时统一使用了 `gap: 28px`，导致元素挤在一起而非左右分布。
**防范**：
1. 在推断 flex 布局时，必须计算**每对相邻子元素的间距**，而非只算第一对就统一应用
2. 如果间距不一致（如 347px vs 28px），不能使用单一 `gap` 属性
3. 当存在"左-右"分布（几个元素靠左、几个靠右）时，应使用 `justify-content: space-between` 或将右侧元素用子容器包裹
4. 间距计算脚本应输出每对元素的间距，标注是否一致
