# 技术栈自动检测规则

## 检测优先级（首个匹配生效）

### 1. 检查 package.json（最高优先级）

```
读取 package.json → 解析 dependencies 和 devDependencies
```

| 检测到的依赖 | 框架 | 备注 |
|------------|------|------|
| `next` | React + Next.js | 检查 `next.config.*` 判断 App Router 还是 Pages Router |
| `nuxt` | Vue + Nuxt | 检查 `nuxt.config.*` 判断 Nuxt 2 还是 3 |
| `vue`（无 nuxt） | Vue 3（如有 @vue/composition-api 则为 Vue 2） | 检查 `vue` 版本 |
| `react`（无 next） | React | 检查 vite/webpack/cra |
| `svelte` 或 `@sveltejs/kit` | Svelte / SvelteKit | |
| `@angular/core` | Angular | |
| `solid-js` | SolidJS | |

### 2. 检查配置文件

| 文件 | 框架 |
|------|------|
| `vite.config.ts/js` | Vite 项目 → 检查插件判断 vue/react/svelte |
| `next.config.ts/js/mjs` | Next.js |
| `nuxt.config.ts/js` | Nuxt |
| `vue.config.js` | Vue CLI（旧版 Vue 2） |
| `angular.json` | Angular |
| `svelte.config.js` | SvelteKit |
| `.vuepress/config.*` | VuePress |
| `vitepress/config.*` | VitePress |

### 3. 检查文件扩展名

| 模式 | 框架 |
|------|------|
| 存在 `*.vue` 文件 | Vue |
| 存在 `*.tsx` 或 `*.jsx` 文件 | React（或通用 JSX） |
| 存在 `*.svelte` 文件 | Svelte |
| 存在 `*.component.ts` 文件 | Angular |

### 4. 检测样式方案

| 信号 | 样式 |
|------|------|
| 存在 `tailwind.config.*` | Tailwind CSS |
| 存在 `uno.config.*` | UnoCSS |
| 存在 `*.module.css` 文件 | CSS Modules |
| 依赖中有 `styled-components` | Styled Components |
| 依赖中有 `@emotion/*` | Emotion |
| 存在 `*.scss` 文件 | SCSS/Sass |
| 存在 `*.less` 文件 | Less |

### 5. 检测 UI 组件库

检查 package.json 依赖：

| 包名 | 组件库 |
|------|--------|
| `element-plus` 或 `element-ui` | Element UI |
| `ant-design-vue` 或 `antd` | Ant Design |
| `@mui/material` | Material UI |
| `vuetify` | Vuetify |
| `@chakra-ui/react` | Chakra UI |
| 存在 `components.json` 文件 | shadcn/ui |
| `@headlessui/react` 或 `@headlessui/vue` | Headless UI |
| `naive-ui` | Naive UI |
| `arco-design/vue` 或 `@arco-design/web-react` | Arco Design |
| `vant` | Vant（移动端） |
| `ant-design-mobile` 或 `ant-design-mobile-vue` | Ant Design Mobile |

### 6. 检测组件命名规范

读取项目中 3-5 个现有组件文件：

- **PascalCase.tsx** → React 规范
- **PascalCase.vue** → Vue 规范
- **kebab-case.vue** → Vue 替代规范
- **PascalCase.svelte** → Svelte 规范

同时检查：
- 导出风格：`export default` 还是命名导出
- 组件模式：函数组件还是类组件
- 脚本风格：`<script setup>` 还是 `<script>` + defineComponent

### 7. 未检测到技术栈 → 纯 HTML/CSS

生成简单静态页面：
- 语义化 HTML5 结构
- CSS 自定义属性作为设计 token
- 无框架依赖
- 单文件或最少文件结构

## 检测脚本

```
1. Glob 查找 package.json → 读取并解析
2. Glob 查找配置文件：vite.config.*、next.config.*、nuxt.config.* 等
3. Glob 查找组件文件：**/*.vue、**/*.tsx、**/*.svelte
4. Glob 查找样式文件：**/*.module.css、**/*.scss、**/*.less、tailwind.config.*
5. 读取 3-5 个现有组件文件了解规范
6. 输出技术栈档案
```

## 技术栈档案输出

```typescript
interface TechStack {
  framework: 'vue3' | 'react' | 'next' | 'nuxt' | 'svelte' | 'angular' | 'html'
  language: 'typescript' | 'javascript'
  styling: 'tailwind' | 'unocss' | 'css-modules' | 'styled-components' | 'scss' | 'css' | 'emotion'
  uiLibrary: string | null
  componentPattern: {
    fileNaming: 'pascal' | 'kebab'
    fileExtension: string
    exportStyle: 'default' | 'named'
    scriptStyle: 'setup' | 'options' | 'function'
  }
  projectRoot: string
}
```
