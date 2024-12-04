# 解析文件目录，映射路由

在现代前端框架中，如 `UmiJS`、`Next.js` 和 `Gatsby`，"约定大于配置"（`Convention over Configuration`）是一种常见的设计理念。这种设计策略旨在通过默认的目录结构和文件命名约定来自动化繁琐的配置步骤，使开发者能够更专注于业务逻辑，提高开发效率。

`Gatsby` 是一个基于 `React` 的静态站点生成器，在 `Gatsby` 中利用约定大于配置来自动化页面创建，任何放在 `src/pages` 目录下的文件都会自动被作为一个页面。Gatsby 会根据文件名和目录路径自动生成相应的路由。例如，`src/pages/index.js` 将被用作站点的主页（即 / 路由），而 `src/pages/about.js` 将与 `/about` 路由对应。

`Next.js` 类似于 `Gatsby`，`Next.js` 也使用 `pages` 目录来自动化页面和路由的创建。所有放在 pages 目录中的文件都会被处理成页面，路径与文件结构对应。例如，`pages/contact.js` 对应 `/contact` 路径。`Next.js` 还支持动态路由，比如 `pages/products/[id].js` 对应于 `/products/123` 的路径。

`UmiJS` 是一个面向企业级应用的 `React` 框架。它也使用了类似的约定，默认情况下会将 `pages` 目录中的文件映射到路由。支持文件系统路由的生成，这使得路由声明更加方便和直观。

## 如何做到目录解析

假如现在存在这么一个目录结构：

```
src/
└── pages/
    ├── home/
    │   └── index.tsx
    ├── about/
    │   └── index.tsx
    └── blog/
        └── index.tsx
```

在这个目录结构中：

- `src/` 是项目的源代码目录。
- `pages/` 目录用于存放不同页面的代码。
- `home/`、`about/` 和 `blog/` 是各个页面的子目录，每个目录下用一个 `index.tsx` 文件表示对应的页面入口。

### require.context

`require.context` 是 `Webpack` 提供的功能，可以按照指定的匹配规则，动态加载一个目录中的所有模块。

**用法**
`require.context` 的基本用法如下：

```js
const context = require.context(directory, useSubdirectories, regExp);
```

- `directory` ：模块文件的目录路径，应该是相对于调用者文件的相对路径。
- `useSubdirectories` ：是否检索子目录 (`true` 或 `false`)。
- `regExp` ：匹配文件的正则表达式，指定要引入的文件类型。

由于 `Webpack` 是在编译时解析文件的，不能使用在运行时计算的值作为 `require.context` 的参数。确保所有用于参数的变量都使用字面量。

```js
// 使用字面量
const context = require.context('./src/components', true, /\.js$/); // ✅

// 动态变量
const dynamicPath = getDynamicPath();
const context = require.context(dynamicPath, true, /\.js$/); // ❌

// 显式定义常量
const DIRECTORY = './src/components';
const USE_SUBDIRECTORIES = true;
const REGEXP = /\.js$/;

const context = require.context(DIRECTORY, USE_SUBDIRECTORIES, REGEXP); // ❌
```

以下是一个如何使用 `require.context` 来获取每个模块的默认导出的示例：

```js
const _ = require("lodash");

const importModules = () => {
  const result = {};
  // NOTE - require.context 仅在构建时工作，它依赖 Webpack 的静态分析，因此不能使用动态路径。路径应该是字面常量，而不是变量或函数结果
  const ctx = require.context("../src/pages", true, /\.js$/);
  // ctx.keys() 返回当前目录下所有文件的相对路径，比如 /src/pages 目录下的 ./a/b/c1.js
  ctx.keys().forEach((key) => {
    // 解析路径，比如 ./src/pages/a/b/c1.js
    const resolvedPath = ctx.resolve(key);
    // 获取模块的默认导出，比如 function () { return 'c1';}
    const module = ctx(key);
    // 通过正则匹配路径，获取路径中的目录名，比如 ['src', 'pages', 'a', 'b', 'c1']
    const matches = resolvedPath.match(/[^\.\/]+/g).slice(0, -1);
    const original = {};
    let current = original;
    matches.forEach((match, index) => {
      if (index === matches.length - 1) {
        current[match] = module;
      } else {
        current = current[match] = current[match] || {};
      }
    });
    _.merge(result, original);
  });

  return result;
};
```

此外，使用 `require.context` 之前需要确保项目使用了 `Webpack` 进行构建

```js
const path = require('path');

module.exports = {
  mode: 'development',
  target: 'node',
  entry: './webpack/index.js',
  output: {
    filename: 'bundle.js',
    path: path.resolve(__dirname, 'dist'),
  },
  devtool: 'source-map',
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: ['@babel/preset-env']
          }
        }
      }
    ]
  }
};
```

以下是最终得到的解析结果：

![Img](./FILES/Gatsby%20路由解析.md/img-20241202114202.png)

### import.meta.glob

`import.meta.glob` 是 `Vite` 提供的一种强大功能，允许开发者根据匹配的文件路径批量导入模块。这在需要处理一组文件、自动化某些模块导入操作时非常有用，例如加载所有组件、页面等。

**基本用法**
`import.meta.glob` 返回一个对象，键是匹配的文件路径，值是返回 `Promise` 的导入函数（即懒加载）。这种机制能有效帮助管理大规模项目的文件导入

`import.meta.glob` 支持通过第二个参数传递选项对象来控制导入行为。

- 异步和懒加载：默认情况下，`import.meta.glob` 会返回懒加载的动态导入函数，只有在调用该函数时才执行实际的模块加载。
- `eager`：设置 `{ eager: true }` 来立即导入文件，而不是按需加载。

以下是一个如何使用 `import.meta.glob` 来获取每个模块的默认导出的示例：

```js
const importModules = () => {
  const result = {};
  // NOTE - import.meta.glob 仅在构建时工作，它依赖 Vite 的静态分析，因此不能使用动态路径。路径应该是字面常量，而不是变量或函数结果
  // eager：true 选项表示不适用懒加载，而是在构建时立即加载所有模块
  const modules  = import.meta.glob("../src/pages/**/*.js", { eager: true });

  for (const modulePath in modules) {
    const module = modules[modulePath].default;
    const matches = modulePath.match(/[^\.\/]+/g).slice(0, -1);
    const original = {};
    let current = original;
    matches.forEach((match, index) => {
      if (index === matches.length - 1) {
        current[match] = module;
      } else {
        current = current[match] = current[match] || {};
      }
    });
    _.merge(result, original);
  }

  return result;
};
```

我们定义的页面文件采用 `CommonJS` 规范。由于 `Rollup` 默认只支持 `ES6` 模块语法，因此需要使用 `@rollup/plugin-commonjs` 将 `CommonJS` 模块转换为 `ES6` 模块，以便 `Rollup` 能够正确处理和打包。此外，通过设置 `output.format` 为 `cjs`，我们指定打包输出的模块格式为 `CommonJS` 格式，使其可以在 `Node.js` 环境中执行。

```js
import { defineConfig } from "vite";
import commonjs from '@rollup/plugin-commonjs';

export default defineConfig({
  plugins: [
    commonjs()
  ],
  build: {
    outDir: "dist", // 构建输出目录，默认为 'dist'
    assetsDir: "assets", // 静态资源（如 CSS、JS、图片）目录，默认是 'assets'
    sourcemap: true, // 生成 sourcemap 文件以便于调试
    rollupOptions: {
      input: "./vite/index.js", // 设置你的自定义入口文件
      output: {
        entryFileNames: "assets/js/[name].js",
        chunkFileNames: "assets/js/[name]-[hash].js",
        assetFileNames: "assets/[ext]/[name]-[hash].[ext]",
        format: 'cjs'
      },
    },
  },
});
```

以下是最终得到的解析结果：

![Img](./FILES/Gatsby%20路由解析.md/img-20241202111147.png)
