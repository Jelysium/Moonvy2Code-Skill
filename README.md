# Moonvy2Code-Skill

将 [月维 (Moonvy)](https://moonvy.com) 设计标注页自动转为像素级精确前端代码的 Skill。

## 功能

- 从月维设计链接提取完整设计数据（图层位置、颜色、字体、间距等）
- 自动识别并裁剪矢量图标为 PNG
- 基于 rect 坐标精确计算所有间距，生成像素级还原的 HTML/CSS
- 自动为按钮、链接、输入框等元素添加 hover/click/focus 交互行为
- 支持输出多种前端框架代码（Vue、React、纯 HTML 等）

## 使用

提供月维设计标注页链接，描述你想生成的页面即可。例如：

> 帮我把这个月维设计稿转成 HTML 页面：https://moonvy.com/project/xxx

也可以通过命令调用：

```
/moonvy2code <月维设计链接>
```

适用于 Claude Code、Codex 及所有支持 Skill 规范的环境。

## 依赖

- [Playwright MCP](https://github.com/microsoft/playwright-mcp) — 浏览器自动化，提取月维运行时数据和截图
- Python 3 + Pillow — 图标裁剪和间距计算

## 注意事项

- 月维需要登录，首次使用时需在浏览器中手动完成登录
- 矢量图标从快照裁剪为 PNG 位图，如需矢量版本需手动替换为 SVG
