# 代码生成策略

## 数据到 CSS 映射

### 颜色转换

月维使用 RGB 浮点数（0-255）。转换为 CSS：
```javascript
function rgbToHex(r, g, b) {
  return '#' + [r, g, b].map(v => Math.round(v).toString(16).padStart(2, '0')).join('')
}
// 示例: { r: 51, g: 51, b: 51 } → "#333333"
```

### 填充映射

| 月维填充 | CSS 属性 |
|---------|---------|
| `type: "color"` 且 `color: {r,g,b}` | `background-color: #xxxxxx` |
| `type: "gradient"` 线性渐变 | `background: linear-gradient(direction, color-stops)` |
| `type: "image"` | `background-image: url(...)` 或 `<img>` 标签 |

### 排版映射

| 月维属性 | CSS 属性 |
|---------|---------|
| `textbox.segments[].fontSize` | `font-size: {value}px` |
| `textbox.segments[].fontWeight` | `font-weight: {value}` |
| `textbox.segments[].fontName.family` | `font-family: "{family}"` |
| `textbox.segments[].lineHeight.value` | `line-height: {value}px` |
| `textbox.segments[].letterSpacing.value`（unit: "per"） | `letter-spacing: {value}%` |
| `textbox.segments[].fills[].color` | `color: #xxxxxx` — **必须逐个提取，不同文字图层可能有不同颜色** |
| `textbox.align` | `text-align: left/center/right` |
| `textbox.segments[].textCase: "upper"` | `text-transform: uppercase` |

### 布局映射

| 月维属性 | CSS 方案 |
|---------|---------|
| 图层 `rect` 相对于父级 | 用 flexbox/grid 定位，或绝对定位 |
| 含 `children` 的编组 | 容器 `<div>` 包含子元素 |
| 图层堆叠顺序 | DOM 顺序（第一个子元素 = 底层） |
| `borderRadius` | `border-radius: {value}px` |
| `blend.opacity` | `opacity: {value}` |
| `blend.isClip: true` | `overflow: hidden` |

### 阴影映射

```json
{ "offsetX": 0, "offsetY": 4, "blur": 4, "spread": 0, "color": {"r":0,"g":0,"b":0,"alpha":0.25} }
```
→ `box-shadow: 0px 4px 4px 0px rgba(0, 0, 0, 0.25)`

### 描边映射

```json
{ "fills": [{"color": {"r":234,"g":234,"b":234}}], "w": 1, "align": "inside" }
```
→ `border: 1px solid #eaeaea`

## 代码生成流程（必须按顺序执行）

### Step 1: 全量图层扫描

在生成任何代码之前，先用 Python 脚本从 design-data.json 提取完整图层树，列出所有图层的类型、名称、坐标和属性。

**目的**：确保不遗漏任何可见元素（图标、小装饰、页脚图标等）。

```python
# 提取完整图层树的脚本模板
import json
with open('design-data.json', 'r', encoding='utf-8') as f:
    raw = f.read()
data = json.loads(json.loads(raw))  # 注意：可能需要双重解析

page = data['pages'][0]

def print_layer(layer, depth=0):
    name = layer.get('name', '?')
    ltype = layer.get('type', '?')
    r = layer.get('rect', {})
    x, y, w, h = r.get('x',0), r.get('y',0), r.get('w',0), r.get('h',0)
    fills = layer.get('fills', [])
    fill_info = ''
    for fill in fills:
        if fill.get('type') == 'color':
            c = fill.get('color', {})
            fill_info = f' bg=({c.get("r",0):.0f},{c.get("g",0):.0f},{c.get("b",0):.0f})'
        elif fill.get('type') == 'image':
            fill_info = ' [IMG]'
        elif fill.get('type') == 'gradient':
            fill_info = ' [GRAD]'
    textbox = ''
    if 'textbox' in layer:
        tb = layer['textbox']
        text = tb.get('text', '')[:25]
        segs = tb.get('segments', [])
        fs = segs[0].get('fontSize', 0) if segs else 0
        fw = segs[0].get('fontWeight', 0) if segs else 0
        textbox = f' "{text}" fs={fs} fw={fw}'
    slices = ' [SLICES]' if 'slices' in layer else ''
    print(f'{"  "*depth}{ltype}: {name} ({x},{y},{w},{h}){fill_info}{textbox}{slices}')
    for child in layer.get('children', []):
        print_layer(child, depth+1)

print_layer(page)
```

**坐标系统说明**：月维 JSON 中子图层的坐标可能是画布相对的（需要减去 page.rect.x/y），也可能是页面相对的。通过检查第一个子图层（通常是页面背景 Rectangle）的坐标来判断：如果背景 rect 接近 (0, 0)，则坐标是页面相对的。

### Step 2: 矢量图标识别与裁剪

月维数据中的矢量图标只有填充颜色和尺寸，**没有 SVG path 数据**。必须从快照裁剪。

**图标识别规则**：

满足以下任一条件的图层判定为图标：
1. `type: "artboard"` 且尺寸较小（< 60×60），子图层全部是 `type: "layer"` 且含 `fills`
2. `type: "group"` 且所有子图层是 `type: "layer"`、无 `textbox`，尺寸 < 80×80
3. 图层名称含 "Icon"、"Vector"、"arrow"、"logo" 等关键词

**裁剪方法**：

```python
from PIL import Image

img = Image.open('design-screenshot.png')
ratio = data['pages'][0].get('snapshotRatio', 2)  # 通常是 2x

# 找到图标图层的 rect
x, y, w, h = layer_rect['x'], layer_rect['y'], layer_rect['w'], layer_rect['h']
box = (int(x*ratio), int(y*ratio), int((x+w)*ratio), int((y+h)*ratio))
crop = img.crop(box)
crop.save(f'assets/{safe_name}.png')
```

**SVG 占位符**：如果图标较简单（如搜索放大镜），可尝试用近似 SVG path 生成，但**必须设置明确的 width/height**，避免浏览器默认 300×150 导致溢出：

```html
<!-- 正确：SVG 有尺寸约束 -->
<button class="search-btn">
  <svg width="22" height="22" viewBox="0 0 22 22">...</svg>
</button>

<!-- 错误：无尺寸约束，浏览器默认 300×150 -->
<button class="search-btn">
  <svg viewBox="0 0 22 22">...</svg>
</button>
```

**规则：所有内联 SVG 必须设置 width 和 height 属性，或通过 CSS 约束尺寸。**

### Step 3: 精确间距计算（核心步骤）

**所有间距必须从 JSON rect 坐标精确计算，禁止目测估计。**

#### 计算方法

```python
# 垂直间距
def v_gap(rect1, rect2):
    """两个图层之间的垂直间距"""
    return rect2['y'] - (rect1['y'] + rect1['h'])

# 水平间距
def h_gap(rect1, rect2):
    """两个图层之间的水平间距"""
    return rect2['x'] - (rect1['x'] + rect1['w'])

# 容器内边距
def padding(container_rect, child_rect):
    """容器到子图层的内边距"""
    return {
        'top': child_rect['y'] - container_rect['y'],
        'left': child_rect['x'] - container_rect['x'],
    }
```

#### 必须计算的间距清单

以下间距在每次代码生成时都必须从坐标计算并输出：

```python
# === 使用完整图层树坐标，逐一计算 ===

# 1. 页面各区块的垂直间距（必须逐对计算所有相邻区块！）
#    header底 → banner顶、banner底 → 内容区顶、内容底 → footer顶
#    !! 常见遗漏：最后一个卡片底 → footer顶 !!

# 2. 每个 card/容器的四边 padding（必须全部四边！）
#    padding-top = first_child.y - container.y
#    padding-bottom = (container.y + container.h) - (last_child.y + last_child.h)
#    padding-left = first_child.x - container.x
#    padding-right = (container.x + container.w) - (rightmost_child.x + rightmost_child.w)
#    !! 常见遗漏：padding-bottom 未计算 !!

# 3. Flex/Grid 容器中子元素间距
#    导航项间距、按钮间距、卡片内图文间距、分页项间距

# 4. 绝对定位元素的位置
#    导航下划线的 bottom 值（= underline.y - (text.y + text.h))
```

#### 间距计算脚本模板

```python
# 计算并输出所有关键间距
# !! 此模板必须覆盖以下三类间距，缺一不可 !!

spacings = {}

# ========== 第一类：相邻区块间距 ==========
# 列出所有主要区块（header、各卡片、footer 等），逐对计算间距
sections = [
    ('header', header_rect),
    ('card_top', card_top_rect),
    ('card_bottom', card_bottom_rect),
    ('footer', footer_rect),
]
for i in range(1, len(sections)):
    prev_name, prev = sections[i-1]
    curr_name, curr = sections[i]
    gap = curr['y'] - (prev['y'] + prev['h'])
    spacings['%s_to_%s' % (prev_name, curr_name)] = gap
    print("  %s -> %s: %dpx" % (prev_name, curr_name, gap))

# ========== 第二类：容器四边 padding ==========
# 对每个卡片/容器，找出其内部的第一个和最后一个子元素
# 计算全部四边 padding
def calc_container_padding(name, container, first_child, last_child_bottom, last_child_right=None):
    """计算并输出容器的四边 padding"""
    p_top = first_child['y'] - container['y']
    p_left = first_child['x'] - container['x']
    p_bottom = (container['y'] + container['h']) - (last_child_bottom['y'] + last_child_bottom['h'])
    print("  %s padding: top=%d, left=%d, bottom=%d" % (name, p_top, p_left, p_bottom))
    if last_child_right:
        p_right = (container['x'] + container['w']) - (last_child_right['x'] + last_child_right['w'])
        print("  %s padding-right: %d" % (name, p_right))
    return {'top': p_top, 'left': p_left, 'bottom': p_bottom}

# 示例：
# calc_container_padding('card_top', card_top_rect, breadcrumb_rect, auth_btn_rect)
# calc_container_padding('card_bottom', card_bottom_rect, section_title_rect, detail_image_rect)

# ========== 第三类：容器内子元素间距 ==========
# （原有的 flex/grid 子元素间距计算保持不变）

# --- 页面区块间距 ---
header_h = 80  # header rect height
banner_y, banner_h = 80, 240
spacings['banner_bottom'] = banner_y + banner_h  # 320

# --- 内容区间距 ---
section_title_y = 350  # 从图层树获取
spacings['content_padding_top'] = section_title_y - spacings['banner_bottom']

title_h = 45  # section title rect height
divider_y = 405
spacings['title_to_divider'] = divider_y - (section_title_y + title_h)

first_card_y = 453
spacings['divider_to_cards'] = first_card_y - divider_y

# --- Grid 行间距 ---
row_ys = [453, 652, 851, 1050]  # 每行的 y 坐标
card_h = 151
spacings['grid_row_gap'] = row_ys[1] - (row_ys[0] + card_h)

# --- 分页间距 ---
last_card_bottom = row_ys[-1] + card_h
pagination_y = 1278
spacings['cards_to_pagination'] = pagination_y - last_card_bottom

pagination_h = 23
footer_y = 1409
spacings['pagination_to_footer'] = footer_y - (pagination_y + pagination_h)

# --- 导航间距 ---
nav_items = [(407, 32), (499, 64), (623, 64), (747, 64), (871, 64)]
spacings['nav_gap'] = nav_items[1][0] - (nav_items[0][0] + nav_items[0][1])

# --- 导航下划线位置 ---
underline_y = 59
nav_text_bottom = 29.49 + 24  # nav_text.y + line_height
spacings['nav_underline_bottom'] = -(underline_y - nav_text_bottom)

print("=== SPACING VALUES (from coordinates) ===")
for k, v in spacings.items():
    print("  %s: %dpx" % (k, v))
```

#### 必须计算的四边 padding

**关键规则：容器的 padding 必须计算全部四边，不能只计算 top 和 left。**

```python
# 容器四边 padding 计算模板
def container_padding(container, first_child_top, last_child_bottom, first_child_left, last_child_right):
    """计算容器的四边 padding"""
    return {
        'top': first_child_top['y'] - container['y'],
        'bottom': (container['y'] + container['h']) - last_child_bottom['y'] - last_child_bottom['h'],
        'left': first_child_left['x'] - container['x'],
        'right': (container['x'] + container['w']) - last_child_right['x'] - last_child_right['w'],
    }

# 示例：产品卡片 (335, 92, 1250, 578)
card = {'x': 335, 'y': 92, 'w': 1250, 'h': 578}
breadcrumb = {'x': 359, 'y': 108, 'w': 113, 'h': 21}  # 第一个子元素
auth_btn = {'x': 1307, 'y': 589, 'w': 175, 'h': 45}   # 最后一个子元素（最底部）
# padding-top: 108 - 92 = 16 ✓
# padding-bottom: (92+578) - (589+45) = 670 - 634 = 36px  ← 必须计算！
# padding-left: 359 - 335 = 24 ✓
# padding-right: (335+1250) - (1307+175) = 1585 - 1482 = 103px
```

**常见错误：只计算 padding-top 和 padding-left，忽略 padding-bottom 和 padding-right。**
这会导致容器底部/右侧的子元素紧贴容器边缘。

#### 必须计算的区块间距

**区块间距 = 后一个区块的 rect.y - 前一个区块的 rect.y - 前一个区块的 rect.h**

```python
# 区块间距计算模板（必须逐对计算所有相邻区块）
sections = [
    {'name': 'header', 'rect': header_rect},
    {'name': 'card-top', 'rect': card_top_rect},
    {'name': 'card-bottom', 'rect': card_bottom_rect},
    {'name': 'footer', 'rect': footer_rect},
]

for i in range(1, len(sections)):
    prev = sections[i-1]
    curr = sections[i]
    gap = curr['rect']['y'] - (prev['rect']['y'] + prev['rect']['h'])
    print("%s -> %s gap: %dpx" % (prev['name'], curr['name'], gap))
```

**常见错误：只计算容器内部元素间距，忽略容器之间的间距。**
例如只计算了卡片内部 label row gap，但未计算卡片底部到 footer 的 44px 间距。

#### 关键教训

| 错误做法 | 正确做法 |
|---------|---------|
| 目测估计 margin-top 为 30px | 从 rect 坐标计算：`y2 - (y1 + h1)` |
| 凭感觉设置 gap: 20px | 计算相邻元素的间距：`next.x - (prev.x + prev.w)` |
| 导航下划线 bottom 随意设 -18px | 计算：underline_y - (text_y + line_height) |
| 所有按钮间距统一用 gap | 逐对计算每个间距，不同则用 margin-left |
| 只算容器 padding-top/left | 计算全部四边 padding（top/right/bottom/left） |
| 只算容器内子元素间距 | 计算所有相邻区块之间的间距 |
| 忽略卡片底部 padding-bottom | 从容器底边减去最低子元素底边：`(container.y+container.h) - (child.y+child.h)` |

### Step 4: 图层分类与处理策略

对图层树中每个图层进行分类，确定处理方式：

| 图层特征 | 分类 | 处理方式 |
|---------|------|---------|
| `type: "text"` 且有 `textbox` | 文字 | 提取文字内容、字体、颜色，生成 HTML 文本 |
| `type: "layer"` 且 `fills: [{type: "image"}]` | 图片图层 | 裁剪快照或用 images URL |
| `type: "group"` 且有 `slices` | 切图编组 | 用切图 URL 作为背景（见切图去重规则） |
| `type: "group"` 无 `slices`，子图层含 `textbox` | 混合编组 | 分别处理背景和文字子图层 |
| `type: "artboard"` 小尺寸，子图层全 `layer` | 图标容器 | 从快照裁剪为 PNG |
| `type: "group"` 小尺寸，子图层全 `layer` 无 `textbox` | 图标编组 | 从快照裁剪为 PNG |
| `type: "layer"` 含 `fills: [{type: "color"}]` | 色块 | CSS background-color |
| `type: "layer"` 含 `fills: [{type: "gradient"}]` | 渐变块 | CSS linear-gradient |

### Step 5: 代码生成与自检

代码生成完成后，执行以下自检：

1. **全量对照**：遍历图层树，检查每个可见图层是否都有对应的 HTML/CSS 实现
2. **间距核对**：对比计算出的间距值与生成的 CSS 值是否一致
3. **SVG 尺寸**：检查所有内联 SVG 是否有明确的 width/height
4. **图标完整性**：检查所有裁剪的图标是否已在 HTML 中引用
5. **切图去重**：检查有 slices 的编组是否避免了文字重复

### Step 6: 交互元素识别与实现

在静态页面完成后，识别并实现交互行为。

#### 交互元素识别规则

从三个信号源识别交互元素：

**信号 1 — HTML 语义元素**（最高优先级）：

| HTML 元素 | 交互类型 |
|-----------|---------|
| `<button>` | 按钮 — 必须有 hover/active 状态 |
| `<a>` | 链接 — 必须有 hover 状态 |
| `<input>` | 输入框 — 必须有 focus 状态 |
| 带有 `cursor: pointer` 的 `<div>/<span>` | 可点击元素 |

**信号 2 — 图层名称关键词**：

| 关键词（中文/英文） | 交互类型 |
|-------------------|---------|
| 按钮、Button、Btn | 按钮交互 |
| 搜索、Search | 搜索交互 |
| 导航、Nav、Tab、标签 | 切换交互 |
| 收藏、Favorite、Heart | 切换状态交互 |
| 注册、登录、Register、Login | 认证交互 |
| 返回顶部、Back-to-top、Top | 滚动交互 |
| 分页、Pagination、Page | 分页交互 |
| 关闭、Close | 关闭交互 |

**信号 3 — 视觉特征**：

| 视觉特征 | 推断交互 |
|---------|---------|
| 圆角矩形背景 + 白色文字 | 按钮 |
| 小图标 + 短文字（如"收藏"） | 图标按钮 |
| 水平排列的等宽色块 + 短文字 | Tab 标签 |
| 固定定位的小方形元素 | 悬浮按钮 |

#### 交互行为实现策略

**A. 自动实现的行为**（明确知道应该做什么）：

| 交互元素 | CSS | JavaScript |
|---------|-----|-----------|
| 通用按钮 hover | `filter: brightness(1.1)` + `transition` | — |
| 通用按钮 active | `transform: scale(0.98)` | — |
| 导航链接 hover | `color: var(--color-primary)` + `transition` | — |
| 搜索框 focus | `box-shadow: 0 0 0 2px var(--color-primary)` | — |
| 返回顶部按钮 | `opacity` 过渡 + `cursor: pointer` | 点击 `scrollTo({top:0,behavior:'smooth'})` + 滚动显隐 |
| 收藏按钮 | `.is-fav` 状态样式 | 点击 toggle `.is-fav` class |
| Tab/分类切换 | `.active` 下划线过渡 | 点击切换 `.active` class |
| 卡片 hover | `box-shadow` 提升 + `transform: translateY(-2px)` | — |

**B. 占位提示的行为**（功能未明确，用 toast 提示）：

| 交互元素 | 行为 |
|---------|------|
| 注册/登录按钮 | 弹出 toast "功能开发中" |
| 搜索按钮 | 弹出 toast "搜索功能开发中"（或执行搜索） |
| 获取授权/提交等业务按钮 | 弹出 toast "功能开发中" |
| 分页点击 | 弹出 toast "功能开发中" |
| 卡片点击（无明确跳转目标） | 弹出 toast "功能开发中" |

#### CSS 交互模板

```css
/* === 通用按钮交互 === */
button, [class*="btn-"] {
  transition: filter 0.2s, transform 0.1s, box-shadow 0.2s;
  cursor: pointer;
}
button:hover, [class*="btn-"]:hover {
  filter: brightness(1.08);
}
button:active, [class*="btn-"]:active {
  transform: scale(0.98);
  filter: brightness(0.96);
}

/* === 导航链接 === */
.nav-item {
  transition: color 0.2s;
}
.nav-item:hover {
  color: var(--color-primary);
}

/* === 搜索框 focus === */
.search-box:focus-within {
  box-shadow: 0 0 0 2px rgba(54, 144, 253, 0.3);
}

/* === 卡片 hover === */
.plant-card {
  transition: box-shadow 0.2s, transform 0.2s;
}
.plant-card:hover {
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  transform: translateY(-2px);
}

/* === 返回顶部显隐 === */
.back-to-top {
  opacity: 0;
  visibility: hidden;
  transition: opacity 0.3s, visibility 0.3s;
}
.back-to-top.visible {
  opacity: 1;
  visibility: visible;
}
```

#### JavaScript 交互模板

```javascript
(function() {
  // === Toast 提示组件 ===
  function showToast(message, duration) {
    duration = duration || 2000;
    var existing = document.querySelector('.toast-msg');
    if (existing) existing.remove();
    var toast = document.createElement('div');
    toast.className = 'toast-msg';
    toast.textContent = message;
    toast.style.cssText = 'position:fixed;top:50%;left:50%;transform:translate(-50%,-50%);' +
      'background:rgba(0,0,0,0.7);color:#fff;padding:12px 24px;border-radius:6px;' +
      'font-size:14px;z-index:9999;pointer-events:none;animation:toastFadeIn 0.3s ease';
    document.body.appendChild(toast);
    setTimeout(function() {
      toast.style.opacity = '0';
      toast.style.transition = 'opacity 0.3s';
      setTimeout(function() { toast.remove(); }, 300);
    }, duration);
  }

  // === 返回顶部 ===
  var backTop = document.querySelector('.back-to-top');
  if (backTop) {
    window.addEventListener('scroll', function() {
      if (window.scrollY > 300) {
        backTop.classList.add('visible');
      } else {
        backTop.classList.remove('visible');
      }
    });
    backTop.addEventListener('click', function() {
      window.scrollTo({ top: 0, behavior: 'smooth' });
    });
  }

  // === 收藏切换 ===
  var favBtn = document.querySelector('.btn-fav');
  if (favBtn) {
    favBtn.addEventListener('click', function() {
      favBtn.classList.toggle('is-fav');
      var text = favBtn.querySelector('.btn-fav-text');
      if (text) text.textContent = favBtn.classList.contains('is-fav') ? '已收藏' : '收藏';
    });
  }

  // === Tab/分类切换 ===
  document.querySelectorAll('.tab-item').forEach(function(tab) {
    tab.addEventListener('click', function() {
      document.querySelectorAll('.tab-item').forEach(function(t) {
        t.classList.remove('active');
      });
      tab.classList.add('active');
    });
  });

  // === 占位提示按钮 ===
  document.querySelectorAll('.btn-register, .btn-login').forEach(function(btn) {
    btn.addEventListener('click', function() {
      showToast('功能开发中');
    });
  });

  // === 搜索按钮 ===
  document.querySelectorAll('.search-btn, .content-search-btn').forEach(function(btn) {
    btn.addEventListener('click', function() {
      var input = btn.closest('.search-box, .content-search')
        ? btn.closest('.search-box, .content-search').querySelector('input')
        : null;
      if (input && input.value.trim()) {
        showToast('搜索: ' + input.value.trim());
      } else {
        showToast('请输入搜索内容');
      }
    });
  });
})();
```

## 各框架策略

### Vue 3 + Vite

```vue
<script setup lang="ts">
// 动态数据的 Props
interface Props {
  title?: string
}
const props = withDefaults(defineProps<Props>(), {
  title: ''
})
</script>

<template>
  <div class="page-container">
    <!-- 从图层树生成 -->
  </div>
</template>

<style scoped>
.page-container {
  /* 月维数据中的设计 token */
}
</style>
```

### React + Next.js

```tsx
interface PageProps {
  title?: string
}

export default function Page({ title = '' }: PageProps) {
  return (
    <div className="page-container">
      {/* 从图层树生成 */}
    </div>
  )
}
```

### React + Vite（Tailwind）

```tsx
export default function Page() {
  return (
    <div className="w-[1920px] bg-white">
      {/* 月维数据转为 Tailwind 类名 */}
    </div>
  )
}
```

### 纯 HTML

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>页面标题</title>
  <style>
    :root {
      --color-primary: #3690fd;
      --color-text: #333333;
      --color-text-secondary: #666666;
      --color-bg: #ffffff;
      --font-main: "Source Han Sans CN", sans-serif;
    }
    /* 组件样式 */
  </style>
</head>
<body>
  <!-- 从图层树生成的语义化 HTML -->
</body>
</html>
```

## 图层到组件的映射启发式规则

### 从图层名称识别组件

| 图层名称模式 | 组件类型 |
|-------------|---------|
| Header、Navbar、Navigation | `<header>` / `<nav>` |
| Footer、Bottom | `<footer>` |
| Button、Btn | `<button>` |
| Input、Search、TextField | `<input>` |
| Card、Item、Tile | 卡片组件 |
| List、Grid | 列表/网格组件 |
| Logo | `<img>` 或 SVG |
| Banner、Hero、Cover | 横幅/Hero 区域 |
| Tab、Tabs | 标签导航 |
| Modal、Dialog、Popup | 弹窗组件 |
| Pagination、Pager | 分页组件 |
| Icon、Vector | 图标/SVG |

### 布局推断

1. **水平编组**（子元素水平排列）→ `display: flex; flex-direction: row`
2. **垂直编组**（子元素垂直堆叠）→ `display: flex; flex-direction: column`
3. **网格布局**（规则的行列）→ `display: grid` 配合适当间距
4. **重叠图层**→ `position: absolute` 在相对定位容器内
5. **左右分布布局**（部分元素靠左、部分靠右）→ `justify-content: space-between`，或将右侧元素用子容器包裹

**关键：推断 flex 布局的 gap 时，必须计算每对相邻子元素的间距。**
如果间距不一致（例如 347px vs 28px），说明不是统一的 `gap`，而是左右分布布局。

```python
# 检测 flex 子元素间距是否一致
def check_gap_consistency(children_rects):
    """检查水平排列的子元素间距是否一致"""
    gaps = []
    for i in range(1, len(children_rects)):
        gap = children_rects[i]['x'] - (children_rects[i-1]['x'] + children_rects[i-1]['w'])
        gaps.append(gap)

    if len(set(gaps)) == 1:
        # 所有间距一致，可以使用 gap 属性
        return 'uniform', gaps[0]
    else:
        # 间距不一致，需要分析是左右分布还是其他布局
        # 典型模式：小间距+大间距 = 左右分布
        return 'mixed', gaps
```

### 间距计算

使用 rect 位置计算间距：
```javascript
// 如果两个兄弟元素有 rect：
// rect1: {x: 260, y: 461, w: 443, h: 27}
// rect2: {x: 260, y: 660, w: 443, h: 27}
// 间距 = 660 - (461 + 27) = 172px

function calculateGap(rect1, rect2) {
  return rect2.y - (rect1.y + rect1.h)  // 垂直间距
  // 或：rect2.x - (rect1.x + rect1.w)  // 水平间距
}
```

## 未还原元素提示规则

代码生成完成后，必须检查并提示用户哪些元素未能完全还原：

### 必须提示的情况

1. **矢量图标**：月维数据中只有 vector 图层的填充颜色和尺寸，没有 SVG path 数据。需要从快照中裁剪为 PNG 位图。
   → 提示：`图标 {name} 已裁剪为 PNG 位图，如需矢量版本请手动替换为 SVG`

2. **裁剪图标**：从快照裁剪的图标可能在高 DPI 屏幕或缩放时失真。
   → 提示：`以下图标是从设计快照裁剪的位图，可能在缩放时失真：{列表}`

3. **渐变按钮文字**：月维可能未提供渐变文字的精确参数。
   → 提示：`按钮 {name} 的渐变参数基于近似值，请核对`

4. **图片填充图层**：`fills: [{type: "image"}]` 但无单独 URL 的图层只能从快照裁剪。
   → 提示：`{图层名} 的图片是从快照裁剪的，非原始资源`

5. **复杂阴影/效果**：多层阴影、内阴影等复杂效果可能只能近似还原。
   → 提示：`{元素} 的阴影效果为近似还原`

### 输出格式

在代码文件末尾或生成结果中添加：

```
⚠️ 以下元素未能完全还原，可能需要手动完善：
- [图标] Logo 图标 → 已裁剪为 PNG，建议替换为矢量 SVG
- [图标] 页脚右侧图标 → 已裁剪为 PNG
- [图标] 搜索放大镜图标 → 已裁剪为 PNG
```

## 通用规则

1. **中文文字必须原样保留** — 不翻译
2. **月维图片**：使用 `data.images[hash].url` 中的 URL 作为 `<img>` src
3. **字体回退**：始终添加 web-safe 回退字体：`font-family: "Source Han Sans CN", "Microsoft YaHei", sans-serif`
4. **设计 token**：将重复使用的颜色/尺寸提取为 CSS 自定义属性
5. **UI 组件库组件**：如果检测到 UI 组件库，映射到其组件：
   - 按钮形状 → `<el-button>` / `<Button>` / `<a-button>`
   - 输入框 → 组件库输入组件
   - 卡片 → 组件库卡片组件
6. **语义化 HTML**：使用 `<header>`、`<nav>`、`<main>`、`<section>`、`<footer>`、`<article>`
7. **无障碍**：为图片添加 `alt` 属性，为交互元素添加 `aria-label`
8. **SVG 尺寸约束**：所有内联 SVG 必须设置 width/height 属性或 CSS 尺寸约束

## 切图与文字去重规则

**关键**：月维的 `slices` 切图是整个编组的合成渲染，包含背景图和所有子图层（含文字）。

当使用 `slices` 切图作为背景时，**不要**再生成该编组内的文字子图层为 HTML 元素，否则会重复叠加。

判断逻辑：
- 编组有 `slices` → 切图已包含所有子图层内容 → 仅用切图作为背景，跳过子图层的 HTML 生成
- 编组无 `slices` → 需要分别生成背景和子图层内容

```
// Banner 编组示例
Group "新闻资讯" {
  slices: { max: {id: "hash"} },  ← 切图 = 背景图 + 文字 的合成图
  children: [
    Rectangle (背景图),
    Text "News and Information",
    Text "新闻资讯"
  ]
}
// → 只用切图 URL 作为 background，不生成文字 HTML
```
