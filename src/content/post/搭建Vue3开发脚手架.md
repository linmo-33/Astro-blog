---
title: 搭建Vue3开发脚手架
description: Vue3 + Typescript + Pinia + Vite + Tailwind CSS + Element Plus
publishDate: 2025-02-21
tags:
  - 前端
ogImage: /social-card.avif
---
##  1.Node.js安装

Vite需要 Node.js 版本 14.18+，16或更高版本。

Tailwind CSS 需要 Node.js 12.13.0 或更高版本。

可使用 `node -v`命令查看当前node版本，如果不符合要求请先升级Nodejs。

##  2.创建vue3工程

```bash
npm create vue@latest
或
yarn create vue@latest
或
pnpm create vue@latest
```

项目创建成功后执行以下命令安装npm依赖。

```bash
npm install --registry=https://registry.npmmirror.com 
或
yarn install
或
pnpm install
```

依赖安装完成后，执行以下命令可运行代码。

```bash
npm run dev
或
yarn dev
或
pnpm run dev
```

## 3.集成Pinia

如果项目创建过程中已选择了`pinia`特性则可跳过该步骤，如果没有，则需要手动安装`pinia`并创建自定义Store。

```bash
perl 代码解读复制代码npm install --registry=https://registry.npmmirror.com pinia@2.0.33
或
yarn add pinia@2.0.33
或
pnpm install pinia@2.0.33
```

### 修改main.ts

将src/main.ts修改为以下内容：

```javascript
import './assets/main.css'

import { createApp } from 'vue'
import { createPinia } from 'pinia'

import App from './App.vue'
import router from './router'

const app = createApp(App)

app.use(createPinia())
app.use(router)

app.mount('#app')
```

###  创建一个store

```javascript
// stores/counter.ts
import { ref, computed } from 'vue'
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)
  const doubleCount = computed(() => count.value * 2)
  function increment() {
    count.value++
  }

  return { count, doubleCount, increment }
})
```

#### 在组件中使用store

```vue
<script setup lang="ts">
import TheWelcome from '../components/TheWelcome.vue'
import { useCounterStore } from '@/stores/counter'

const counter = useCounterStore();
counter.count++
// 自动补全！ 
counter.$patch({ count: counter.count + 1 })
// 或使用 action 代替
counter.increment()
</script>

<template>
  <main>
    <!-- 直接从 store 中访问 state -->
    <div>Current Count: {{ counter.count }}</div>
    <TheWelcome />
  </main>
</template>
```

##4. 集成Tailwind CSS

Tailwind CSS 需要 Node.js 12.13.0 或更高版本。对于大多数实际项目，建议将 Tailwind 作为 PostCSS 插件安装，本文使用的也是该方式。

### 4.1 安装postcss、sass、autoprefixer和tailwindcss以及相关依赖

- Sass 是一款强化 CSS 的辅助工具，它在 CSS 语法的基础上增加了变量 (variables)、嵌套 (nested rules)、混合 (mixins)、导入 (inline imports) 等高级功能，这些拓展令 CSS 更加强大与优雅。使用 Sass 以及 Sass 的样式库（如 Compass）有助于更好地组织管理样式文件，以及更高效地开发项目。
- autoprefixer是一款自动管理浏览器前缀的插件，它可以解析CSS文件并且添加浏览器前缀到CSS内容里，使用Can I Use（caniuse网站）的数据来决定哪些前缀是需要的。把autoprefixe添加到资源构建工具（例如Grunt）后，可以完全忘记有关CSS前缀的东西，只需按照最新的W3C规范来正常书写CSS即可。如果项目需要支持旧版浏览器，可修改browsers参数设置 。

执行以下命令安装依赖：

```bash
npm install --registry=https://registry.npmmirror.com --save-dev autoprefixer postcss postcss-comment postcss-html postcss-import postcss-scss sass sass-loader tailwindcss 

yarn add --save-dev autoprefixer postcss postcss-comment postcss-html postcss-import postcss-scss sass sass-loader tailwindcss 

pnpm install --save-dev autoprefixer postcss postcss-comment postcss-html postcss-import postcss-scss sass sass-loader tailwindcss 
```

###4.2 创建配置文件postcss.config.js和tailwind.config.js

**创建配置文件**

使用命令行可以自动创建postcss.config.js和tailwind.config.js配置文件，也可以手动创建。

```bash
npx tailwindcss init -p
```

**tailwind.config.js**

```js
css 代码解读复制代码/** @type {import('tailwindcss').Config} */
module.exports = {
  darkMode: "class",
  corePlugins: {
    preflight: false
  },
  content: ["./index.html", "./src/**/*.{vue,js,ts,jsx,tsx}"],
  theme: {
    extend: {
      colors: {
      }
    }
  }
};
```

**postcss.config.js**

```js
css 代码解读复制代码export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}
```

###4.3创建并引入tailwind.css

**创建tailwind.css**

在src目录下创建styles目录，在styles目录下创建tailwind.css。

```tailwind.css```文件内容如下：

```less
@tailwind base;
@tailwind components;
@tailwind utilities;
```

**main.ts中引入tailwind.css**

配置完成后需要引入tailwindcss，修改src/main.ts内容如下：

```javascript
import '@/styles/tailwindcss.css';
import './assets/main.css'

import { createApp } from 'vue'
import { createPinia } from 'pinia'

import App from './App.vue'
import router from './router'

const app = createApp(App)

app.use(createPinia())
app.use(router)

app.mount('#app')
```

**在组件中使用tailwindcss**

```xml
<template>
  <main>
    <!-- 直接从 store 中访问 state -->
    <div class="w-full h-[100px] bg-[red] flex justify-center items-center">
      Hello Tailwind CSS
    </div>
    <TheWelcome />
  </main>
</template>
```

以上代码定义了一个宽度100%，高度100px，背景是红色，使用flex布局，垂直方向和水平方向内容都居中的区域，区域中有一个文本元素，显示Hello Tailwind CSS。

##5.Element Plus

本文使用的是Element Plus按需自动引入的方式，此方式可以使编译产物体积更小，运行速度更快。如果需要实现完整导入，请参阅Element Plus官方文档。

###5.1 安装Element Plus

```sql
npm install --registry=https://registry.npmmirror.com element-plus --save
或
yarn add element-plus --save
或
pnpm install element-plus --save
```

###5.2 修改tsconfig.json

在 tsconfig.json 中通过 compilerOptions.type 指定全局组件类型，这样可以配合Volar插件实现代码提示功能。

```json
{
  "files": [],
  "references": [
    {
      "path": "./tsconfig.node.json"
    },
    {
      "path": "./tsconfig.app.json"
    }
  ],
  "compilerOptions": {
    "types": [
      "element-plus/global"
    ]
  }
}
```

### 5.3 安装Element Plus自动导入工具

```bash
npm install --registry=https://registry.npmmirror.com -D unplugin-vue-components unplugin-auto-import
或
yarn add -D unplugin-vue-components unplugin-auto-import
或
pnpm install -D unplugin-vue-components unplugin-auto-import
```

### 5.4 修改vite.config.js

```javascript
import { fileURLToPath, URL } from 'node:url'

import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import vueJsx from '@vitejs/plugin-vue-jsx'
import AutoImport from 'unplugin-auto-import/vite'
import Components from 'unplugin-vue-components/vite'
import { ElementPlusResolver } from 'unplugin-vue-components/resolvers'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [
    vue(),
    vueJsx(),
    AutoImport({
      resolvers: [ElementPlusResolver()],
    }),
    Components({
      resolvers: [ElementPlusResolver()],
    }),
  ],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url))
    }
  }
})
```

### 5.5 使用Element Plus组件

```xml
<template>
  <main>
    <!-- 直接从 store 中访问 state -->
    <div class="w-full h-[100px] bg-[red] flex justify-center items-center">
      Hello Tailwind CSS
    </div>
    <div>
      <el-button type="primary">Element Plus按钮</el-button>
    </div>
    <div>Current Count: {{ counter.count }}</div>
    <TheWelcome />
  </main>
</template>
```

以上代码就是添加了一个Element Plus组件库的按钮。