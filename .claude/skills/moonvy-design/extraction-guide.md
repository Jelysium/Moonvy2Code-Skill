# 月维设计数据提取指南

## 概述

月维将设计数据以 JSON 文件形式存储在 `fs.moonvy.com` 上。每个设计页面的完整规格可通过两种方式获取：

1. **fs.moonvy.com 的 JSON 文件**（首选，每页约 40-80KB）
2. **`window.genomeSpec` 全局变量**（标注页上的运行时对象）

## 数据获取方式

### 方式 1：JSON 文件（首选）

月维标注页加载时，会从 `fs.moonvy.com` 获取 JSON 文件。URL 格式为：
```
https://fs.moonvy.com/{hash}.json
```

查找该 URL 的方法——拦截网络请求或查询 `performance.getEntriesByType('resource')`：
```javascript
const entries = performance.getEntriesByType('resource');
const jsonEntry = entries.find(e => e.name.includes('fs.moonvy') && e.name.endsWith('.json'));
```

然后获取并解析：
```javascript
const resp = await fetch(jsonEntry.name);
const designData = await resp.json();
```

### 方式 2：运行时对象

页面暴露 `window.genomeSpec`，包含：
- `genomeSpec.genome` — 完整设计数据（结构与 JSON 文件相同）
- `genomeSpec.rootLayer` — 根图层快捷访问
- `genomeSpec.selection` — 当前选中的图层
- `genomeSpec.codeGen` — 代码生成配置
- `genomeSpec.code` — 当前选中项的生成代码

### 方式 3：月维 API

月维使用 `api.moonvy.com` 接口：
- `GET /auth/user-info` — 用户认证状态
- `GET /project/project-info?projectId={id}` — 项目元数据
- `POST /anynode/get` — 获取节点数据
- `POST /anynode/list` — 列出节点

## 登录处理

月维需要认证。导航到项目 URL 时：
- 如果重定向到 `/auth?tab=login` → 用户需手动登录
- 登录后，导航回原始 URL

## JSON 数据结构

### 根对象

```json
{
  "genomeVer": 1.9,
  "styles": {
    "fillStyles": [],
    "effectStyles": [],
    "textStyles": []
  },
  "variables": {
    "all": {},
    "collections": {}
  },
  "images": {
    "{sha256_hash}": {
      "type": "png",
      "url": "https://fs.moonvy.com/{hash}.png"
    }
  },
  "pages": [PageObject],
  "meta": {
    "from": "figma",
    "fileId": "figma-file-id",
    "designRatio": 1,
    "designPlatform": "iOS",
    "layerId": "layer-id",
    "layerName": "page-name",
    "filename": "project-name",
    "url": "figma-url"
  },
  "tags": []
}
```

### 页面（画板）对象

```json
{
  "id": "109:161",
  "name": "page-name",
  "type": "artboard",
  "rect": { "x": 6302, "y": 0, "w": 1920, "h": 1479 },
  "isFrame": true,
  "borderRadius": 0,
  "borderRadiusSmoothing": 0,
  "blend": {
    "blendMode": "pass",
    "opacity": 1,
    "visible": true,
    "isClip": true
  },
  "fills": [FillObject],
  "annotations": [],
  "varBind": {},
  "snapshot": "image-hash",
  "snapshotRatio": 2,
  "snapshotPreview": "image-hash",
  "children": [LayerObject]
}
```

### 图层类型

#### 文字图层
```json
{
  "id": "109:449",
  "name": "layer-name",
  "type": "text",
  "rect": { "x": 260, "y": 350, "w": 120, "h": 45 },
  "borderRadius": 0,
  "blend": { "blendMode": "pass", "opacity": 1, "visible": true },
  "textbox": {
    "text": "实际文字内容",
    "indent": 0,
    "spacing": 0,
    "align": "left",
    "alignVertical": "center",
    "autoResizeBox": "all",
    "segments": [TextSegment]
  },
  "annotations": [],
  "varBind": {}
}
```

#### 文字段落
```json
{
  "start": 0,
  "end": 4,
  "fontName": { "family": "Source Han Sans CN", "style": "Bold" },
  "fontSize": 30,
  "fontWeight": 700,
  "fills": [{
    "type": "color",
    "opacity": 1,
    "blendMode": "normal",
    "visible": true,
    "color": { "r": 51, "g": 51, "b": 51 }
  }],
  "letterSpacing": { "value": 0, "unit": "per" },
  "lineHeight": { "value": 44, "unit": "px" },
  "textCase": "none",
  "textDecoration": "none"
}
```

#### 形状/图片图层
```json
{
  "id": "109:471",
  "name": "Rectangle 33",
  "type": "layer",
  "rect": { "x": 260, "y": 453, "w": 212, "h": 151 },
  "borderRadius": 0,
  "borderRadiusSmoothing": 0,
  "subType": "vector",
  "blend": { "blendMode": "pass", "opacity": 1, "visible": true },
  "fills": [FillObject],
  "strokes": [StrokeObject],
  "effects": [EffectObject],
  "annotations": [],
  "varBind": {}
}
```

#### 编组图层
```json
{
  "id": "109:448",
  "name": "Group 5",
  "type": "group",
  "rect": { "x": 260, "y": 350, "w": 320, "h": 45 },
  "borderRadius": 0,
  "blend": { "blendMode": "pass", "opacity": 1, "visible": true },
  "children": [LayerObject],
  "slices": {
    "max": { "id": "hash", "ratio": 4 },
    "base": { "id": "hash", "ratio": 1 }
  },
  "annotations": [],
  "varBind": {}
}
```

### 填充类型

#### 纯色
```json
{
  "type": "color",
  "opacity": 1,
  "blendMode": "normal",
  "visible": true,
  "color": { "r": 255, "g": 255, "b": 255 }
}
```

#### 渐变
```json
{
  "type": "gradient",
  "opacity": 1,
  "blendMode": "normal",
  "visible": true,
  "gradient": {
    "type": "linear",
    "stops": [
      { "color": { "r": 96, "g": 163, "b": 252, "alpha": 1 }, "position": 0 },
      { "color": { "r": 52, "g": 126, "b": 246, "alpha": 1 }, "position": 1 }
    ],
    "from": { "x": 0.5, "y": 0 },
    "to": { "x": 0.5, "y": 1 }
  }
}
```

#### 图片
```json
{
  "type": "image",
  "opacity": 1,
  "blendMode": "normal",
  "visible": true
}
```

### 描边对象
```json
{
  "fills": [{
    "type": "color",
    "color": { "r": 234, "g": 234, "b": 234 }
  }],
  "w": 1,
  "join": "none",
  "align": "center",
  "cap": "none",
  "miterLimit": 4,
  "dash": []
}
```

### 效果对象（阴影）
```json
{
  "type": "shadow",
  "visible": true,
  "blendMode": "normal",
  "offsetX": 0,
  "offsetY": 4,
  "blur": 4,
  "spread": 0,
  "color": { "r": 0, "g": 0, "b": 0, "alpha": 0.25 }
}
```

## 数据提取脚本

使用 `browser_evaluate` 执行以下脚本提取所有设计数据：

```javascript
async () => {
  // 从 performance 条目中查找 JSON URL
  const entries = performance.getEntriesByType('resource');
  const jsonEntry = entries.find(e =>
    e.name.includes('fs.moonvy') && e.name.endsWith('.json')
  );

  if (!jsonEntry) return { error: 'Design JSON not found' };

  // 获取完整设计数据
  const resp = await fetch(jsonEntry.name);
  const data = await resp.json();

  // 返回结构化数据
  return {
    meta: data.meta,
    pageName: data.pages[0]?.name,
    pageSize: { w: data.pages[0]?.rect.w, h: data.pages[0]?.rect.h },
    layerCount: countLayers(data.pages[0]),
    imageCount: Object.keys(data.images).length,
    fullData: JSON.stringify(data)
  };
}

function countLayers(layer) {
  let count = 1;
  if (layer.children) {
    layer.children.forEach(c => { count += countLayers(c); });
  }
  return count;
}
```

## 图片资源详解

### 获取设计截图（推荐方式）

JSON 中自带页面快照，比浏览器截图更干净（无月维 UI）：

```javascript
// 页面对象中的 snapshot 字段
const snapshotHash = data.pages[0].snapshot;
// 在 images 对象中查找 URL
const screenshotUrl = data.images[snapshotHash].url;
// 下载
// curl -o design-screenshot.png {screenshotUrl}
```

### 图片分类

JSON 中的 `images` 对象包含以下几类图片：

| 图片类型 | 来源 | 获取方式 |
|---------|------|---------|
| 页面快照 | `pages[0].snapshot` | 直接用哈希在 `images` 中查 URL |
| 页面预览 | `pages[0].snapshotPreview` | 同上，低分辨率版本 |
| 切图资源 | 图层 `slices.max.id` / `slices.base.id` | 用切图哈希在 `images` 中查 URL |
| 图层填充 | `fills: [{type: "image"}]` | **无单独 URL**，仅存在于快照中 |

### 切图资源示例

编组图层的 `slices` 字段引用图片（如 banner 横幅图）：

```json
// Banner 编组中的 slices
{
  "slices": {
    "max": {"id": "3c090a62...", "ratio": 4},
    "base": {"id": "efd48906...", "ratio": 1}
  }
}

// 在 images 对象中查找
"3c090a62...": {"url": "https://fs.moonvy.com/cHEbGFxV...png"}
```

### 图层填充图片的获取方案

部分图层的 `fills` 为 `[{type: "image"}]` 但没有具体的哈希引用。这类图片需要从页面快照中裁剪。

**方案：按 `rect` 坐标从快照裁剪**

快照图片就是画板的渲染结果，图层 `rect` 坐标与快照像素坐标直接对应（因为子图层坐标是画板相对的）。

Python 裁剪脚本（推荐）：

```python
from PIL import Image
import json, os

# 读取设计数据
with open('design-data.json', 'r') as f:
    data = json.load(f)

# 下载的页面快照
img = Image.open('design-screenshot.png')
os.makedirs('assets', exist_ok=True)

# 遍历图层树，裁剪 fills 为 image 类型的图层
def crop_image_layers(layer, images_map):
    # 检查是否为图片填充图层且无单独 URL
    if layer.get('fills'):
        for fill in layer['fills']:
            if fill.get('type') == 'image' and not fill.get('imageRef'):
                rect = layer['rect']
                box = (rect['x'], rect['y'], rect['x'] + rect['w'], rect['y'] + rect['h'])
                crop = img.crop(box)
                name = layer.get('name', layer.get('id', 'unknown'))
                safe_name = name.replace(' ', '_').replace('/', '_')
                crop.save(f'assets/{safe_name}.png')

    # 递归处理子图层
    for child in layer.get('children', []):
        crop_image_layers(child, images_map)

page = data['pages'][0]
crop_image_layers(page, data.get('images', {}))
```

ImageMagick 替代方案：
```bash
magick design-screenshot.png -crop "212x151+260+453" +repage "assets/card-1.png"
```

**注意**：如果快照的 `snapshotRatio` 不为 1（如 2），图片是高分辨率的，裁剪坐标需要乘以该比例。

## 关键注意事项

1. **坐标系统可能有两种**：
   - **页面相对**：子图层坐标直接是页面内的位置（更常见）。例如页面 rect 为 (6302, 0)，但子图层从 (0, 0) 开始。
   - **画布相对**：子图层坐标包含画布偏移，需减去页面 rect 的 x/y。
   - **判断方法**：检查第一个背景图层（通常是覆盖全页的 Rectangle）的 rect。如果接近 (0, 0) 或 (page.x, page.y)，即可确定坐标系类型。

2. **颜色值是浮点数**（如 51.000001）。转换为十六进制前需四舍五入取整。

3. **字体族名** 是原始 Figma 字体名（如 "Source Han Sans CN"）。生成 CSS 时需映射为 web-safe 等效字体。

4. **文字段落** 支持单个文字图层内的混合样式（一个文字块中可以有不同字体/颜色）。

5. **图片** 通过 SHA256 哈希在 `images` 对象中引用。哈希映射到 `fs.moonvy.com` 上的 URL。

6. **本设计没有 auto-layout 数据**。如果存在 Figma Auto Layout 数据，会以 `autoLayout` 属性出现在图层上。

7. **设计截图优先用 JSON 中的 snapshot**。比 `browser_take_screenshot` 更干净，不含月维标注工具界面。

8. **JSON 可能需要双重解析**：提取保存时如果直接 `JSON.stringify(data)`，读取时需要 `json.loads(json.loads(raw))`。建议保存时确保写入的是已解析的对象而非字符串。

9. **矢量图标无法获取 SVG path**：`type: "layer"` 且含 `fills` 的矢量图层只有颜色和尺寸数据，没有 path 路径。只能从快照裁剪为 PNG 位图。
