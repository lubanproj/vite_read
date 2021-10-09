# vite 源码解读 —— 文件编译过程分析

> 本文所有分析针对版本 v1.0.0-rc.13，你可以下载 [vite1.0.0-rc.13](https://github.com/vitejs/vite/tags?after=v2.0.0-alpha.1) 的源码

前面我们介绍了 vite server 的启动过程，那么服务启动后，vite 是如何把浏览器需要的 modules 进行编译构建，并且返回给浏览器呢？

### 一、vite 如何处理 modules

我们按照网页加载的顺序，首先从入口的 html 文件进行分析，在 html 文件中，有下面一段代码：

```html
<body>
  <div id="app"></div>
  <script type="module" src="/src/main.js"></script>
</body>
```

当浏览器检测到 `<script type="module" src="/src/main.js"></script>` 这行代码时，会按照 esm 标准处理，会去分析 /src/main.js 里面 import 的 modules。

我们来看看 main.js 里面的内容

```javascript
import { createApp } from 'vue'
import App from '/App.vue'
createApp(App).mount('#app')
```

此时，假如不出意外，浏览器会执行以下操作：

1. 从 vue 文件中获取 createApp 模块
2. 从 App.vue 文件中获取 App 模块

但是这里有个问题，第 1 步的操作是会出现失败的。这是为什么呢？因为 'vue' 是通过 npm 进行管理的，在 node_modules 目录下，所以浏览器是无法通过 'vue' 这个路径找到相应的文件的。那么怎么办呢？

vite 的解决方案是进行路径重写。vite 会将下面语句

```javascript
import { createApp } from 'vue'
```

重写为：

```javascript
import { createApp } from '/@modules/vue'
```

这个过程是在 serverPluginModuleRewrite 这个插件里面完成的。经过路径重写之后，在下一个 koa middleware 中，通过另一个 moduleResolvePlugin 插件，去正则匹配所有带有 '/@modules' 前缀的路径，然后去找到对应的文件。

### 二、vite 如何编译

vite 最初是为 vue3.x 开发的，所以 vite 文件编译最主要的内容是对 vue 文件进行编译。

在了解 vue 文件是如何编译之前，你需要先了解一下 [SFC 规范](https://vue-loader.vuejs.org/zh/spec.html#%E7%AE%80%E4%BB%8B)

每个 `.vue` 文件包含三种类型的顶级语言块 `<template>`、`<script>` 和 `<style>`，事实上，vite 对 vue 文件的编译也是三种语言块分开编译的。

在我们首次启动 vite 项目的时候，对于 App.vue 文件，可以看到如下的三个请求（基于vite 1.0 版本，2.0 版本有改动）

```javascript
App.vue
App.vue?type=style&index=0
App.vue?type=template
```

三个请求的路径都是 App.vue，但是参数不同，第一个请求没有任何参数，第二个请求 type 为 style，第三个请求 type 为 template。

看到这里你可能会感到好奇，为什么请求一个 App.vue 文件会变成三个请求呢，个人理解是方便 vite 对于不同类型的文件调用不同的组件分开编译。

我们来看看源码的逻辑：

```javascript
export const vuePlugin: ServerPlugin = ({
  root,
  app,
  resolver,
  watcher,
  config
}) => {
  ...
  
  app.use(async (ctx, next) => {
    // ctx.vue is set by other tools like vitepress so that vite knows to treat
    // non .vue files as vue files.
    if (!ctx.path.endsWith('.vue') && !ctx.vue) {
      return next()
    }

    // 解析器解析 SFC 文件
    // upstream plugins could've already read the file
    const descriptor = await parseSFC(root, filePath, ctx.body)
    if (!descriptor) {
      return next()
    }

    if (!query.type) {
	  ...
      
      ctx.type = 'js'
      const { code, map } = await compileSFCMain(
        descriptor,
        filePath,
        publicPath,
        root
      )
      ctx.body = code
      ctx.map = map
      return etagCacheCheck(ctx)
    }

    // 根据不同的 query.type 进行不同逻辑的处理
    if (query.type === 'template') {
      ...
      
      const { code, map } = compileSFCTemplate(
        root,
        templateBlock,
        filePath,
        publicPath,
        descriptor.styles.some((s) => s.scoped),
        bindingMetadata,
        vueSpecifier,
        config
      )
	  ...
    }

    if (query.type === 'style') {
      ...
      const result = await compileSFCStyle(
        root,
        styleBlock,
        index,
        filePath,
        publicPath,
        config
      )
      ...
    }

    if (query.type === 'custom') {
      ...
    }
  })
}
```

整个过程是使用 [@vue/compiler-sfc](https://github.com/vuejs/vue-next/tree/master/packages/compiler-sfc) 组件进行解析，这里的原理比较复杂，感兴趣的可以参考这篇文章，进行深入研究 [vue 中 SFC 文件解析为 SFCDescriptor 的流程.md](https://github.com/lubanproj/articles/blob/main/vite/vue%20%E4%B8%AD%20SFC%20%E6%96%87%E4%BB%B6%E8%A7%A3%E6%9E%90%E4%B8%BA%20SFCDescriptor%20%E7%9A%84%E6%B5%81%E7%A8%8B.md)

### 三、css 文件如何编译

在 esm 机制下，css 文件也是通过 js 的形式加载的。所以 css 文件也存在编译过程。

vite 对 css 文件的处理逻辑全都封装在 serverPluginCss.ts 文件下（vite2.0 在 css.ts 文件下），我们可以把 css 文件分为两类，一类是 SFC 文件中 `<style></style>` 标签包裹的样式文件和普通的 css 样式文件，另一类是其他格式的 css 文件，包括 less|sass|scss|styl|stylus|postcss 等

我们来看看源码逻辑：

```javascript
export const cssPlugin: ServerPlugin = ({ root, app, watcher, resolver }) => {
  app.use(async (ctx, next) => {
    await next()
    // handle .css imports
    if (
      isCSSRequest(ctx.path) &&
      // note ctx.body could be null if upstream set status to 304
      ctx.body
    ) {
      const id = JSON.stringify(hash_sum(ctx.path))
      if (isImportRequest(ctx)) {
        // 发现如果是从 js 文件中导入的 css 样式库，例如 `import('/style.css')`，
        // 则进行 processCss 处理
        const { css, modules } = await processCss(root, ctx)
        ctx.type = 'js'
		// 然后调用 codegenCss 将带有 `?import` 请求的的 css 重写为一个 js module
        // 在 js module 里面，css 属性将链接到真实的 css 文件
        ctx.body = codegenCss(id, css, modules)
      }
    }
  })
    
  ...
}
```

cssPlugin 先检测发过来的请求是不是一个 css 请求，通过文件名后缀是否是 `.css` 来判断。假如是 css 请求时，继续判断是不是从 js 文件导入的 css 样式库，也就是含有 import 标签，例如`import('/style.css')`。假如发现含有 import 标签，则调用 processCss 函数进行处理，processCss 函数处理逻辑如下：

```javascript
async function processCss(root: string, ctx: Context): Promise<ProcessedCSS> {
    // source didn't change (marker added by cachedRead)
    // just use previously cached result
    if (ctx.__notModified && processedCSS.has(ctx.path)) {
      return processedCSS.get(ctx.path)!
    }
	...

    const result = await compileCss(root, ctx.path, {
      id: '',
      source: css,
      filename: filePath,
      scoped: false,
      modules: ctx.path.includes('.module'),
      preprocessLang,
      preprocessOptions: ctx.config.cssPreprocessOptions,
      modulesOptions: ctx.config.cssModuleOptions
    })
	...
	
    processedCSS.set(ctx.path, res)
    return res
  }
```

它主要调用了 compileCss 函数进行编译，在 compileCss 函数中又调用了 `compileStyleAsync` 函数：

```javascript
export async function compileCss(
  root: string,
  publicPath: string,
  {
    source,
    filename,
    scoped,
    modules,
    // @ts-ignore TODO @deprecated
    vars,
    preprocessLang,
    preprocessOptions = {},
    modulesOptions = {}
  }: SFCAsyncStyleCompileOptions,
  isBuild: boolean = false
): Promise<SFCStyleCompileResults | string> {

  ...
    
  return await compileStyleAsync({
    source,
    filename,
    id: `data-v-${id}`,
    scoped,
    modules,
    // @ts-ignore TODO @deprecated
    vars,
    modulesOptions: {
      generateScopedName: `[local]_${id}`,
      localsConvention: 'camelCase',
      ...modulesOptions
    },

    preprocessLang,
    preprocessCustomRequire: (id: string) => require(resolveFrom(root, id)),
    preprocessOptions,

    postcssOptions,
    postcssPlugins
  })
}

```

跟踪 compileStyleAsync 这个函数，发现它的本质是一类 sfcCompiler 组件，如下：

```javascript
import sfcCompiler from '@vue/compiler-sfc'

const { compileStyleAsync } = resolveCompiler(root)
  
export function resolveCompiler(cwd: string): typeof sfcCompiler {
  return require(resolveVue(cwd).compiler)
}
```

从代码中可以看出，sfcCompiler 是 @vue/compile-sfc 组件下的一个模块，所以我们得出结论，vite 对 css 的编译，最终还是通过 @vue/compile-sfc 组件完成的。

之前还提到了对  less|sass|scss|styl|stylus|postcss  文件的支持，其实这也是通过 @vue/compile-sfc 实现的，我们在 compileCss 里面有这段逻辑，先会判断处理的文件类型，然后指定文件类型，调用 @vue/compile-sfc 组件处理。

```javascript
export async function compileCss(
  root: string,
  publicPath: string,
  {
    ...
    preprocessLang,
    preprocessOptions = {},
	...
}: SFCAsyncStyleCompileOptions,
  isBuild: boolean = false
): Promise<SFCStyleCompileResults | string> {
  
  if (preprocessLang) {
    preprocessOptions = preprocessOptions[preprocessLang] || preprocessOptions
    // include node_modules for imports by default
    switch (preprocessLang) {
      case 'scss':
      case 'sass':
        preprocessOptions = {
          includePaths: ['node_modules'],
          ...preprocessOptions
        }
        break
      case 'less':
      case 'stylus':
        preprocessOptions = {
          paths: ['node_modules'],
          ...preprocessOptions
        }
    }
  }
    
    return await compileStyleAsync({
    	...
        preprocessLang,
        preprocessCustomRequire: (id: string) => require(resolveFrom(root, id)),
        preprocessOptions,
        ...
  })
}
```

### 四、其他类型文件的编译

上面我们分析了 vue 、js 、css 文件的编译，除此之外，vite 还支持了其他格式的文件编译，比如 tsx|jsx、json、wsam 等，下面简单列举下

#### 1、tsx|jsx

tsx|jsx 的编译是通过 esbuild 实现的，主要在 serverPluginEsbuild 这个插件下，主要逻辑如下：

```javascript
import {
  tjsxRE,
  transform,
  ...
} from '../esbuildService'

export const esbuildPlugin: ServerPlugin = ({ app, config, resolver }) => {
  const jsxConfig = resolveJsxOptions(config.jsx)

  app.use(async (ctx, next) => {
	...
    
    ctx.type = 'js'
    const src = await readBody(ctx.body)
    const { code, map } = await transform(
      src!,
      resolver.requestToFile(cleanUrl(ctx.url)),
      jsxConfig,
      config.jsx
    )
    ctx.body = code
    if (map) {
      ctx.map = JSON.parse(map)
    }
  })
}
```

可以看到，主要调用了 esbuildService 中的  transform 函数，这个函数调用了 esbuild 的 transfrom 函数，如下：

```javascript
import {
  startService,
  Service,
  TransformOptions,
  Message,
  Loader
} from 'esbuild'

...

const ensureService = async () => {
  if (!_servicePromise) {
    _servicePromise = startService()
  }
  return _servicePromise
}

...

// transform used in server plugins with a more friendly API
export const transform = async (
  src: string,
  request: string,
  options: TransformOptions = {},
  jsxOption?: SharedConfig['jsx']
) => {
  const service = await ensureService()
  ...
  const result = await service.transform(src, options)
  if (result.warnings.length) {
      console.error(`[vite] warnings while transforming ${file} with esbuild:`)
      result.warnings.forEach((m) => printMessage(m, src))
  }

  let code = result.code
  // if transpiling (j|t)sx file, inject the imports for the jsx helper and
  // Fragment.
  if (file.endsWith('x')) {
      if (!jsxOption || jsxOption === 'vue') {
          code +=
              `\nimport { jsx } from '${vueJsxPublicPath}'` +
              `\nimport { Fragment } from 'vue'`
      }
      if (jsxOption === 'preact') {
          code += `\nimport { h, Fragment } from 'preact'`
      }
  }

  return {
      code,
      map: result.map
  }
}
```

要弄懂这个逻辑，你需要去了解 esbuild 的原理以及 api，参考 [esbuild transform 函数](https://esbuild.github.io/api/#transform-api)

#### 2、json 文件

对 json 文件的支持，主要通过 @rollup/pluginutils 组件实现，如下：

```javascript
import { ServerPlugin } from '.'
import { readBody, isImportRequest } from '../utils'
import { dataToEsm } from '@rollup/pluginutils'

export const jsonPlugin: ServerPlugin = ({ app }) => {
  app.use(async (ctx, next) => {
    await next()
    // handle .json imports
    // note ctx.body could be null if upstream set status to 304
    if (ctx.path.endsWith('.json') && isImportRequest(ctx) && ctx.body) {
      ctx.type = 'js'
      ctx.body = dataToEsm(JSON.parse((await readBody(ctx.body))!), {
        namedExports: true,
        preferConst: true
      })
    }
  })
}
```

通过调用了 @rollup/pluginutils 的 [dataToEsm](https://github.com/rollup/plugins/tree/master/packages/pluginutils) 函数，将 json 对象转化成 ES module. 

#### 3、wasm 文件

wasm 格式的文件是 WebAssembly 标准的文件，对 wasm 文件的处理主要在 ServerPluginWasm 文件下，如下：

```javascript
export const wasmPlugin: ServerPlugin = ({ app }) => {
  app.use((ctx, next) => {
    if (ctx.path.endsWith('.wasm') && isImportRequest(ctx)) {
      ctx.type = 'js'
      ctx.body = `export default (opts = {}) => {
        return WebAssembly.instantiateStreaming(fetch(${JSON.stringify(
          ctx.path
        )}), opts)
          .then(obj => obj.instance.exports)
      }`
      return
    }
    return next()
  })
}
```

这段代码非常简单，就是在代码中插入了一段 js 代码，调用了 WebAssembly 的 instantiateStreaming 接口，直接从流式底层源编译和实例化WebAssembly模块。可以参考：[Web Assembly API](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/instantiateStreaming)



### 结语

本文主要将 vite 对各种格式文件的编译过程进行了源码分析，主要是基于 v1.0.0-rc.13 版本，vite 2.0 版本有一些改动。如果有遗漏的地方，欢迎大家补充。
