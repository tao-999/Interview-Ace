# 前端构建工具

---

## 1) Webpack 打包管线：入口解析 → 依赖图构建 → 产物生成；Loader/Plugin 执行顺序与 Tapable Hooks

### 生命周期与数据流
1. **初始化**  
   - 读取配置，创建 `Compiler`，实例化与注册 **plugins**（订阅 Tapable hooks）。
   - 解析 `entry`（可为函数/对象/字符串），确定构建入口点（EntryPoints）。
2. **构建（make 阶段）**  
   - 为每个入口创建 `Compilation`，从入口模块开始递归解析模块依赖（`import/require/url()`）。  
   - 使用 **NormalModuleFactory** 解析模块：  
     - 路径解析（`resolve`，受 alias/extensions/mainFields/exports 等影响）。  
     - 命中 **rules** → 匹配到 **loaders** 链，对模块源进行**从右到左**的转换（见下）。  
   - 产出 **模块图（Module Graph）** 与 **代码块（Chunks）**（按入口/动态导入拆分）。
3. **优化（seal 前后）**  
   - Tree-Shaking（`usedExports`）、Scope Hoisting、`SplitChunksPlugin` 合并公共依赖、确定 `moduleIds/chunkIds`。  
   - 生成 runtime（模块加载器 + manifest）。
4. **生成与输出（emit/processAssets）**  
   - 将每个 chunk 渲染为 assets（JS/CSS/资源），写入输出目录；执行 `assetEmitted` 等钩子。

### Loader 顺序与机制
- **匹配**：`module.rules` 自上而下匹配；每条 rule 的 `use` 数组**从右到左**执行（`use: ['style','css','postcss']` → 先 `postcss`，再 `css`，最后 `style`）。  
- **阶段**：  
  - `enforce: 'pre'`（如 eslint-loader），先于普通 loader；  
  - 普通 loader；  
  - `enforce: 'post'`（如覆盖特殊场景）。  
- **pitching**：如定义了 `loader.pitch`，则**从左到右**执行 pitch；pitch 可短路后续 normal 流程（少见但重要）。
- **纯函数约束**：loader 需尽量保持可复用/可缓存（使用 `cache`、`thread-loader` 时尤其）。

### Plugin 与 Tapable Hooks（常用）
- `compiler.hooks`：  
  - `environment/afterEnvironment` → 环境就绪  
  - `beforeRun/run/watchRun` → 启动前  
  - `normalModuleFactory` → 注册解析/构建钩子  
  - `compilation` → 创建 `Compilation`（注册 `compilation.hooks`）  
  - `make` → 开始构建依赖图  
  - `emit`（v5 中推荐 `processAssets`）→ 产物生成前  
  - `assetEmitted` → 单个文件写出后  
  - `done/failed` → 构建结束
- `compilation.hooks`：  
  - `buildModule/succeedModule/finishModules` → 模块级事件  
  - `seal` → 进入优化与封装  
  - `optimizeModules/Chunks/TreeShaking` 等  
  - `processAssets`（替代早期 `emit`，带有 **stage** 优先级）
- Tapable 类型：`SyncHook`、`AsyncSeriesHook`、`SyncWaterfall`、`AsyncParallelBailHook` 等，决定插件注册函数的调用模型。

---

## 2) HMR 原理对比：Webpack Dev Server / Vite / esbuild；模块边界与状态保留

### 共同点
- 通过 **WebSocket** 通知变更；客户端接受到更新后按**模块边界**应用补丁。  
- 模块需显式接受更新（`import.meta.hot.accept` / `module.hot.accept`），否则回退到整页刷新。

### Webpack Dev Server（WDS）
- **先打包后服务**：内存中维护打包产物；变更触发 **增量编译**，产出 HMR 更新包（hot update chunk + manifest）。  
- 客户端 runtime 比对 hash → 拉取更新 → 调用 `module.hot.accept` 回调。  
- **状态保留**：限定在被 accept 的模块图内；未被 accept 的父链将触发上溯或整页刷新。React 栈常配合 `react-refresh` 做组件级状态保留。

### Vite
- **原生 ESM + 即时转换**：开发态**不打包**，按需编译单文件并以 ESM 提供；变更只影响**该模块及其导入者**。  
- HMR 精准到模块：Vite 追踪 import 图，向受影响的模块发送 `hot update`，并执行 `import.meta.hot.accept`。  
- **状态保留**：与框架配套（`@vitejs/plugin-react-swc` 内置 React Fast Refresh；Svelte/Vue 有各自 HMR 实现），因不需要重建整包，边界更细。

### esbuild
- 官方提供 **增量编译** 与 `--watch`/`serve`，**不内置完整 HMR 语义**；生态通常由上层框架实现（如 Vite 利用 esbuild 仅做转译/预构建）。  
- 纯 esbuild 方案常为**全量/局部重载**或社区 HMR 插件，状态保留能力取决于框架层。

### 模块边界策略
- 组件库/视图层：启用框架级刷新（React Refresh / Vue HMR / Svelte HMR）。  
- 非视图模块（工具函数/样式）：直接 `accept` 自身变更；复杂场景下编写自定义热替换逻辑以迁移状态。

---

## 3) Tree-Shaking 生效条件：ESM、`sideEffects`、`/*#__PURE__*/`、类字段与动态属性影响

### 前置条件
- **静态可分析**：要求 ESM（`import/export`），不可含动态 `require`。  
- **生产模式**：Webpack `mode=production` 启用 `usedExports` 标记并在压缩阶段（Terser/SWC）做 DCE。  
- **不被副作用阻塞**：包/文件声明无副作用或精确标注。

### `package.json` 的 `sideEffects`
- `false`：该包默认 **无副作用**，允许删除未用的导入（**注意** CSS/Polyfills/注册全局的模块不能被错误标注）。  
- `["*.css","./polyfill.js"]`：白名单保留有副作用的文件。  
- 不正确的 `sideEffects=false` 会导致必要代码被摇掉；缺失会导致摇不动。

### `/*#__PURE__*/` 与纯函数
- 对 IIFE/构造表达式标注 **纯净**，如：`/*#__PURE__*/ new Heavy()`、`/*#__PURE__*/ someFactory()`，便于压缩器删除未用结果。  
- Babel/TS 转译后产生的 helper 也常带 PURE 注释，提升 DCE 效果。

### 易阻碍 Tree-Shaking 的写法
- 动态属性/计算属性名：`obj[dynamic()] = ...`、`export * from dynamic()`。  
- 类字段含副作用初始化：`class X { y = sideEffect(); }`；即使未用到 `y`，初始化仍会执行。  
- Getter/Setter 含副作用；顶层 `console.log`/注册全局。  
- TS/Babel 将 ESM **降级为 CJS**（`module: commonjs`）会破坏摇树；应输出为 `esnext` 给 bundler 处理。

---

## 4) 代码分割策略：`import()`、Webpack `SplitChunks`、Rollup/Vite `manualChunks` 与 “vendor 大包” 隐患

### 动态导入（应用层分片）
- `import('./feature')` 基于 **路由/交互** 懒加载，天然形成独立 chunk。  
- 适合：路由页面、重型图表编辑器、富文本、低频模块。

### Webpack `SplitChunksPlugin`（依赖层分片）
- 默认将多入口共享依赖抽取到 `vendors~*.js`。  
- 关键参数：`cacheGroups`（按包名/路径归类）、`minChunks`、`maxInitialRequests`、`maxSize`。  
- **隐患**：把所有三方合并成一个**超大 vendor**，任何小改动导致**整块缓存失效**。  
  - 优化：按领域拆 vendor（如 `react-vendor`、`chart-vendor`）、限制 `maxSize` 强制再切分、结合路由分片。

### Rollup/Vite `manualChunks`
- 通过函数精细化控制：  
  ```ts
  // vite.config.ts
  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules/react')) return 'react-vendor';
          if (id.includes('node_modules/echarts')) return 'chart-vendor';
        }
      }
    }
  }
  ```
- 易与路由懒加载配合，形成**功能域** + **页面级**双层分片。避免过度细分造成请求瀑布（HTTP/2/3 影响较小但仍需平衡）。

---

## 5) 长期缓存（Long-term Caching）：`contenthash/chunkhash`、`runtimeChunk`、稳定 `moduleIds/chunkIds`

### 文件名哈希
- `contenthash`：基于文件内容；**推荐**用于 JS/CSS。  
- `chunkhash`：基于 chunk 内容（包含依赖），变动链更长。  
- 通过文件名哈希 + CDN `immutable` 缓存实现持久缓存。

### runtime 与 manifest 稳定
- `optimization.runtimeChunk: 'single'`：将运行时代码（模块加载器 + 清单）独立成 `runtime.js`，避免其变更污染业务 chunk。  
- `optimization.moduleIds: 'deterministic'` / `chunkIds: 'deterministic'`（Webpack 5）：基于路径/内容的稳定 ID，减少 ID 抖动。  
- 避免使用不稳定的 `named`/`natural` 在大项目中造成轻微改动引发大面积哈希漂移。

### 常见坑
- CSS 抽取顺序改变 → `contenthash` 变化：固定 `mini-css-extract-plugin` 的 **order**（或使用 `cssnano` 规范化）。  
- 动态导入路径拼接（`import(`./${x}.js`)`）会影响 chunk 边界与命名，尽量使用明确分支。  
- 辅助：`records`（v4）或持久化缓存记录以稳定 ID（v5 通过 deterministic+cache 基本足够）。

---

## 6) Source Map 选择：开发/生产类型与安全/性能权衡

| 场景 | Webpack 常用 | 特性 | 适用 |
|---|---|---|---|
| 开发快编译 | `eval` / `eval-cheap-module-source-map` | 基于 eval 拼接，映射粗（cheap=无列，无 loader 链） | 极致速度 |
| 开发高质量 | `cheap-module-source-map` / `source-map` | module 表示映射到转换前源码；非 eval | 调试友好 |
| 生产可上传 | `hidden-source-map` | 生成 .map 但不在产物中引用 | 配合错误上报平台 |
| 生产防泄漏 | `nosources-source-map` | 仅保留映射不含源码 | 合规严格场景 |
| 关闭 | `false` | 无 | 极致安全、最小体积 |

注意：
- **CSP** 严格时禁止 `eval`，开发态需改用非 eval 的映射。  
- **隐私**：生产禁止内联/公开 source map；采用 **hidden** 并通过 CI 上传到 Sentry 等。  
- 多阶段转换（TS→Babel→Minify）需确保 **`sourceMaps: true`** 且保留 `sourcesContent`，保证链路可追溯。

---

## 7) 模块解析：alias、extensions、mainFields/exports 条件导出、`browser` 字段

### 解析决策点
1. **alias**：重写路径（如 `@/` → `src/`）。  
2. **extensions**：后缀优先级（`.ts`、`.tsx`、`.mjs`…）。  
3. **package.json fields**：  
   - `exports`（优先）：条件导出，按 **conditions** 选择子路径：`import`/`require`、`browser`、`default` 等。  
   - 旧字段：`module`（ESM 入口）、`main`（CJS 入口）、`browser`（浏览器替代）。  
4. **条件**（Webpack `resolve.conditionNames` / Rollup/Vite 内置）：  
   - 浏览器构建常启用 `browser` 与 `import`；Node 构建启用 `node`/`require`。

### 影响与坑
- 不正确的 `exports` 配置会导致 bundler 找不到子路径导出（`package/subpath`）。需在包内声明相应子路径。  
- `browser` 字段可将 Node 内置替换为空或浏览器实现；谨慎避免引入 `fs/path` 等 Node 专属依赖。  
- Monorepo 中需统一 `conditions` 与 TS paths，避免**运行时入口**与**类型入口**不一致。

---

## 8) CSS 处理链：CSS Modules、PostCSS、Tailwind、`@layer` 与 CSS 代码分割

### CSS Modules
- 通过 `*.module.css` 将类名哈希化，导出为 JS 对象：  
  ```js
  import styles from './x.module.css'
  <div className={styles.btn}/>
  ```
- 与 TS：`declarations.d.ts` 声明 `*.module.css` 模块类型。  
- 注意样式顺序与跨组件复用（避免隐式全局覆盖）。

### PostCSS
- 常见插件：`autoprefixer`、`postcss-nesting`、`cssnano`；在 Webpack 走 `postcss-loader`，在 Vite 由 PostCSS 配置自动接入。  
- 构建时统一浏览器目标（`browserslist`）。

### Tailwind
- JIT 模式扫描模板，按**内容**生成原子类；`@layer base/components/utilities` 定义层级与覆盖策略。  
- 生产需配合 `content` 白名单与 safelist，避免误删动态类。  
- 与 CSS Modules 共存：UI 结构用原子类，复杂局部用 modules 或 `@apply`。

### CSS 代码分割
- Webpack：`mini-css-extract-plugin` 抽取 CSS，按 chunk 输出；配合 `SplitChunks` 聚合公共样式。  
- Vite/Rollup：自动将每个入口/异步 chunk 的 CSS 提取为独立文件；可通过 `build.cssCodeSplit` 控制。  
- 关键 CSS：使用 `critters`/`penthouse` 或服务端内联首屏关键样式，剩余 `preload`/`media` 延迟。

---

## 9) 静态资源管线：Webpack Asset Modules vs 传统 loader；Vite 资源 URL、内联阈值与指纹

### Webpack 5 资源类型
- `asset/resource`：复制到输出目录，返回 URL（取代 `file-loader`）。  
- `asset/inline`：转为 data URI（取代 `url-loader`）。  
- `asset/source`：以字符串方式导入（取代 `raw-loader`）。  
- `asset`：自动在 inline/resource 之间选择，受 `parser.dataUrlCondition.maxSize` 影响。

```js
module.rules = [
  { test: /\.png$/, type: 'asset', parser: { dataUrlCondition: { maxSize: 8 * 1024 } } },
  { test: /\.svg$/, type: 'asset/resource' }
]
```

### Vite 资源处理
- 小于 `build.assetsInlineLimit`（默认 4KB）内联为 base64，其余拷贝并指纹命名。  
- 直接 `import img from './a.png'` 得到 URL；`?raw`/`?url`/`?inline` 控制行为。  
- 字体/图片建议设置 `crossorigin` 与 `preload`（关键字体）以减少 FOUT/FOIT。

### 指纹与缓存
- 产物文件带 `[hash]`；CDN 缓存策略：`public, max-age=31536000, immutable`。  
- 大图/多分辨率图像用 `<picture>`+`srcset`，构建时经 `imagemin/squoosh` 预处理。

---

## 10) 环境变量与常量替换：Webpack `DefinePlugin` vs Vite `import.meta.env`；借助 DCE 剔除分支

### 常量替换
- **Webpack**：  
  ```js
  new webpack.DefinePlugin({
    __BUILD_ENV__: JSON.stringify(process.env.BUILD_ENV),
    'process.env.NODE_ENV': JSON.stringify('production')
  })
  ```
  字面量替换发生在编译期，配合压缩器可做 **条件折叠** 与 **DCE**。  
- **Vite**：  
  - `import.meta.env.MODE/DEV/PROD` 内置；自定义以 `VITE_` 前缀注入（`VITE_API_BASE`）。  
  - 亦可使用 `define: { __FEATURE__: 'true' }` 做额外常量替换。

### 死代码消除示例
```ts
if (import.meta.env.PROD) {
  enableProdMode();
} else {
  // 仅开发态
  startDebugOverlay();
}
```
- 在生产构建中，`import.meta.env.PROD === true`，压缩器可移除 `else` 分支（要求条件为**静态可推导**）。  
- Webpack 中常用 `'process.env.NODE_ENV'` 替换为 `'production'` 触发行内优化（React/Redux 等会根据该常量移除警告代码）。

### 风险与建议
- **必须**将值替换为**字面量**，而非运行时变量。  
- Node 专属变量（`process.env.*`）在浏览器目标需明确替换或用 polyfill（Vite 默认不注入 `process`）。  
- 禁止在客户端打包中直接暴露敏感信息（密钥、token）；仅注入非敏感常量，机密由服务端下发。

---

## 11) Vite 预构建（dep optimization）：为何用 esbuild 预打包；何时失效；如何诊断与强制重建

**目的与原理**
- 将 `node_modules` 中的依赖（尤其是 CJS/混合格式）**统一转成 ESM**、**去除多入口重复**、**合并成更少的模块**，以减少开发态请求数与解析开销。
- 使用 **esbuild** 做极快的打包与转译（不做复杂优化/压缩），产物缓存到 `node_modules/.vite/deps`。

**何时触发/失效**
- 首次 `vite dev`；`package.json`/锁文件变化；`vite.config` 中与解析相关的配置变动（alias、optimizeDeps、plugins 影响解析结果）；依赖版本更新；`NODE_ENV`/`MODE` 等环境影响；Monorepo 中被 workspace 链接的包发生源码变化。
- 手动失效：删除 `node_modules/.vite` 或使用 `--force`。

**诊断与控制**
- 开启调试日志：`VITE_DEBUG=true vite` 或 `vite --debug deps`。
- 指定白/黑名单：
  ```ts
  // vite.config.ts
  export default {
    optimizeDeps: {
      include: ['lodash-es', 'dayjs'],
      exclude: ['some-esm-already', 'link-local-lib'] // 避免被预打包
    }
  }
  ```
- 处理 **外链 workspace 包**：对使用 TS/ESM 源码的本地包，常设置 `optimizeDeps.exclude` 并用 `resolve.alias` 指向源码；或将其构建后再被主项目依赖。
- 强制重建：`vite --force`；删除 `.vite/deps`；或变更 `optimizeDeps.force: true`（Vite 5+）。

---

## 12) Vite 开发态架构：原生 ESM + 即时转换链（Plugin hooks）+ 精准 HMR；与“先打包后服务”的区别

**Vite 开发态核心**
- **不预先打包应用源码**：浏览器以原生 ESM 按需请求模块，Vite 作为中间服务器对每个请求执行 **即时转换**（TS→JS、SFC→JS、CSS→JS 模块等）。
- **插件链**：基于 Rollup 插件接口扩展，开发态使用 `resolveId`/`load`/`transform` 等处理单个模块请求；SSR/构建沿用 Rollup 实现。
- **模块图与 HMR**：维护依赖图，文件更改时精确计算受影响的导入者集合，通过 `import.meta.hot.accept` 触发模块级更新。

**与 Webpack Dev Server 的差异**
- WDS：**先打包后服务**（内存文件系统），变更触发**增量编译**再下发 HMR 更新包。  
- Vite：**先服务后转换**，按需编译单文件，无需等待整个 bundle 重建；HMR 粒度更细。

**关键钩子（开发态）**
- `configureServer`（注册中间件、HMR 处理）、`resolveId`、`load`、`transform`、`handleHotUpdate`（自定义热边界与失效策略）。

---

## 13) Rollup 作为库打包器：`external/peerDependencies`、多格式输出（esm/cjs/umd/iife）与 Tree-Shaking 友好度

**基本配置**
```ts
// rollup.config.ts
import dts from 'rollup-plugin-dts';
import esbuild from 'rollup-plugin-esbuild';

const external = [
  ...Object.keys(require('./package.json').peerDependencies || {}),
  ...Object.keys(require('./package.json').dependencies || {})
];

export default [
  {
    input: 'src/index.ts',
    external, // 不打包外部依赖
    plugins: [esbuild({ target: 'es2019' })],
    output: [
      { file: 'dist/index.mjs', format: 'esm', sourcemap: true },
      { file: 'dist/index.cjs', format: 'cjs', exports: 'named', sourcemap: true }
    ]
  },
  {
    input: 'src/index.ts',
    plugins: [dts()],
    output: { file: 'dist/index.d.ts', format: 'es' }
  }
];
```

**要点**
- **external**：将 `peerDependencies` 与不希望内联的 `dependencies` 标记为外部，避免重复打包与版本冲突。
- **多格式输出**：`esm`（Tree-Shaking 友好、现代工具首选）、`cjs`（Node/老工具）、`umd/iife`（直接 `<script>` 使用）。
- **Tree-Shaking 友好**：保持纯 ESM 源码与入口，避免顶层副作用；`package.json` 中合理设置 `sideEffects:false`（或精确白名单）。
- **类型产出**：推荐独立 rollup `dts` 步骤或 `tsc --emitDeclarationOnly`。
- **Dual Package Hazard**：同时导出 `module`/`main` 时，`exports` 条目需明确条件，防止同包被加载两份（CJS/ESM 混用）。

---

## 14) 构建性能优化：Webpack 持久化缓存、并行/多进程、esbuild/SWC 辅助；冷启动与增量构建定位

**Webpack 侧**
- **持久化缓存**：`cache: { type: 'filesystem' }`，并设置 `buildDependencies`、`version` 以控制失效；缓存目录放在高速磁盘。
- **并行化**：`thread-loader`（限 CPU 密集型 loader）、`parallel` 选项（如 `terser-webpack-plugin`）；但过度并行会因 I/O/序列化开销适得其反。
- **快速转译/压缩**：使用 `esbuild-loader`/`swc-loader` 替代 `babel-loader`；生产压缩用 `esbuildMinify`/`swcMinify` 代替 `terser` 可显著提速。
- **模块解析**：减少 `resolve.modules` 搜索范围；合理的 `alias` 与 `extensions` 顺序；关闭没必要的 `symlinks`/`fullySpecified`。
- **类型检查分离**：TS 项目使用 `fork-ts-checker-webpack-plugin`，主进程只转译。

**诊断路径**
- 生成 `stats.json`，用 `webpack-bundle-analyzer` 与 `speed-measure` 类工具分析耗时阶段。
- 观察 **冷启动**（首次全量）与 **增量构建**（watch/HMR）区别；瓶颈多在**解析/转译/压缩/图片处理**。

**Vite/Rollup 侧**
- 开发态瓶颈多数在 **依赖预构建** 与 **插件 transform**；减少插件数量与正则匹配范围；合理 `optimizeDeps`。
- 生产构建可用 `esbuild` 压缩（`build.minify: 'esbuild'`），在需要兼容性时退回 `terser`。
- 大型图像/字体在构建前离线处理，避免在打包阶段做重 CPU 任务。

---

## 15) Babel vs SWC vs esbuild：转译/压缩能力、插件生态、与 TypeScript 的协作

| 维度 | Babel | SWC | esbuild |
|---|---|---|---|
| 语言转译 | 最全，Stage 提案与插件生态完善 | 覆盖常用特性，Decorator/TS 等成熟 | 覆盖常用 TS/JS/JSX，特性较保守 |
| 速度 | 慢 | 快 | 很快 |
| 压缩 | `babel-minify`（少用）/配合 `terser` | 内置 `minify`（效果与兼容性较好） | 内置压缩（速度快，体积略大于 Terser） |
| 插件生态 | 海量、成熟 | 相对少，但常用足够 | 插件有限（通过二次打包层如 Vite/Rollup 扩展） |
| TS 协作 | `@babel/preset-typescript` 仅**转译不校验** | 同 | 同 |
| Source Map | 完整 | 完整 | 完整 |
| 适用场景 | 需要**复杂语法/宏**与生态兼容 | 性能优先且语法足够 | 极致性能、工具内部管线 |

**推荐组合**
- **应用**：开发/构建链路用 esbuild/SWC 转译 + `tsc --noEmit` 类型检查；生产压缩用 esbuild/SWC（兼容性高要求用 Terser）。
- **库**：Rollup + esbuild/SWC 转译；若需要 Babel 生态插件（宏、i18n 提取等），保持 Babel 作为“最后一公里”。

---

## 16) TypeScript 策略：`tsc --noEmit` 类型检查，打包走 swc/esbuild；Project References/paths 映射

**分离型方案**
```json
// package.json
{
  "scripts": {
    "typecheck": "tsc --noEmit -p tsconfig.json",
    "build": "rollup -c" // 内部用 esbuild/swc 转译
  }
}
```

**Project References（Monorepo 加速与跨包类型联动）**
```json
// tsconfig.json (root)
{
  "files": [],
  "references": [
    { "path": "packages/core" },
    { "path": "packages/ui" }
  ]
}
```
- 各子包 `tsconfig.json` 启用 `"composite": true`，可生成可缓存的 `.tsbuildinfo` 与声明产物，供其他包增量引用。

**路径映射与打包一致性**
- TS `paths` 与 bundler `alias` 必须**同步**：
  ```json
  // tsconfig.base.json
  { "compilerOptions": { "baseUrl": ".", "paths": { "@core/*": ["packages/core/src/*"] } } }
  ```
  ```ts
  // rollup/vite.config
  resolve: { alias: { '@core': fileURLToPath(new URL('packages/core/src', import.meta.url)) } }
  ```
- 产出声明：`tsc --emitDeclarationOnly` 或 `rollup-plugin-dts`。
- 避免 **const enum 被擦除**：如需保持枚举值，配置 `preserveConstEnums: true` 且确保下游可解析；否则编译期内联即可。

---

## 17) 图片与字体优化：SVGO、imagemin/Squoosh；KTX2/Basis 压缩纹理的接入

**矢量（SVG）**
- 管线：`svgo` 基础优化，保留必要属性与可访问性；图标体系可使用 `@svgr/rollup` 转为组件或 `svg-sprite-loader` 合并为雪碧图。
- 注意：移除内联 `fill`/`stroke` 时请确认样式来源，避免渲染差异。

**位图（JPEG/PNG/WebP/AVIF）**
- 预处理：`squoosh-cli` 或 `imagemin` 离线批量压缩（CI/设计交付前置）。
- 响应式：`<picture>` + `srcset`，构建期生成多尺寸与多格式；首屏关键图 `preload`。

**纹理压缩（WebGL/WebGPU）**
- 使用 **Basis Universal/KTX2**（`.ktx2`）以获得跨 GPU 压缩格式；打包保留 `.ktx2`，运行时用 `KTX2Loader`（three.js）或等价加载器。
- 编码链路：`basisu`/`toktx` 将源图转为 `.ktx2`；在构建中视为静态资源（`asset/resource`）。
- 运行要求：COEP/COOP 或适配器跨源隔离以启用 `WebGL2`/`WebGPU` 的多线程解码（视实现）。

**字体**
- 首选 **WOFF2**，必要时保留 WOFF 回退；对 CJK 采用**子集化**（`pyftsubset`、`glyphhanger`）。
- 预加载关键字体：`<link rel="preload" as="font" type="font/woff2" crossorigin>`；可控 FOUT：`font-display: swap|optional`。
- 变量字体（VF）减少切片数量，但需测试渲染一致性与体积收益。

---

## 18) Monorepo 构建：workspaces 与 Nx/Turborepo；跨包 alias/TS references 与打包器联动

**包管理与链接**
- `pnpm`/`yarn`/`npm` workspaces 提供**软链接**与去重；保证**单一 React/Vue 实例**可通过 `dedupe`/`resolutions`。
- 依赖图编排：`Nx`/`Turborepo` 通过 **任务图** 与 **本地/远程缓存**（Remote Cache）复用构建结果；对相同输入的任务命中缓存直接复用产物。

**打包器联动**
- Linked 包在 Vite 中默认被视为源码，可能进入**依赖预构建**：根据需要在 `optimizeDeps.exclude` 排除，或在子包内产出编译后的 ESM。
- 一致的 **TS paths 与 bundler alias**，避免运行时与类型解析不一致。
- 跨包 Storybook/Docs：以主站 Vite/Rollup 配置复用别名，防止 duplicate React/Vue。

**类型与产物复用**
- 开启 TS Project References；子包构建输出到 `dist/`，主包依赖 `dist`（生产）与 `src`（开发）两套解析策略，可通过条件 `exports` 控制：
  ```json
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "development": "./src/index.ts" // 仅开发工具识别（需谨慎）
    }
  }
  ```

---

## 19) SSR/SSG/边缘渲染集成：Vite SSR、Next（Turbopack/Webpack）、Nuxt（Vite/Rollup）

**构建面**
- **区分 server/client 入口**：两套构建目标、外部化策略、条件导出（`exports` 条件 `node`/`browser`/`deno`/`worker` 等）。
- **外部化**：服务端构建将 Node 内置与外部依赖 `external`；客户端严格禁止 Node 内置依赖。
- **注入与同构**：SSR 将数据注入 HTML（`serialize/escape`），客户端水合；SSG 在构建期产出静态 HTML + 资源。

**Vite SSR**
- `ssr` 构建：`vite build --ssr src/entry-server.ts`；运行时以 `createServer` 注入中间件渲染。
- `ssr.noExternal`/`ssr.external` 控制外部化；Edge/Workers 目标需 ESM 与无 Node API 依赖。
- 可用 **流式**（`pipe`）输出与 `#await` 渐进 UX（具体由框架层支持）。

**Next**
- Webpack/Turbopack 分别负责生产/开发；自动区分 **server/client** 编译（RSC/RFC 场景更严格）。
- `next/dynamic` 实现客户端代码分割；Edge Runtime 要求 Web 平台 API（无 Node 内置）。

**Nuxt**
- 以 Vite 为开发与构建底座（或 Rollup 模式），`nitro` 输出 server/edge 多 target；`server`/`client` 插件隔离。

**注意**
- Edge/Workers 限制：无 `fs/path` 等 Node 内置、无长连接持久状态；依赖 KV/HTTP/DB over HTTP。
- CSP 与 streaming 结合时需避免内联可执行脚本（nonce/hash）。

---

## 20) Workers/Worklets 打包：`new Worker()`/`AudioWorklet`/`OffscreenCanvas` 的产物拆分、publicPath 与跨域限制

**Web Worker/Shared Worker**
- 现代打包器支持 `new Worker(new URL('./worker.ts', import.meta.url), { type: 'module' })` 语法，将 worker 作为独立 chunk 输出并正确处理 **publicPath**。
- Vite 配置：
  ```ts
  export default {
    worker: {
      format: 'es',
      rollupOptions: {
        output: { manualChunks: {} }
      }
    }
  }
  ```
- 跨域：Worker 脚本必须与主线程同源（或正确的 CORS 头）；`type:'module'` 的 worker 同样受 CORS/模块解析约束。

**AudioWorklet**
- 需使用 `audioWorklet.addModule(url)`，该 `url` 应指向独立 JS 文件；打包器需要将 `worklet` 文件**单独发射**为资源（Vite/Rollup 可用 `?worker`/插件，或 `new URL()`）。
- MIME：确保正确的 `Content-Type: text/javascript`；某些环境中需 `crossorigin`。

**OffscreenCanvas**
- 在 Worker 中使用 Canvas，需要浏览器支持 `OffscreenCanvas`；与 `transferControlToOffscreen()` 协作。
- 高级特性（`WebGL` + 多线程解码）常要求 **跨源隔离**（COOP/COEP）以启用 `SharedArrayBuffer`。

**publicPath 与部署**
- Webpack 使用 `output.publicPath`；Vite 生产使用 `base`。Worker/Worklet 的资源 URL 必须与之保持一致，否则在子路径部署时会 404。
- 资源指纹：Worker 产物同样带 hash；注意缓存控制与更新策略。

**HMR 与调试**
- Worker 参与 HMR 的支持有限；开发时通常全量重载该 worker 脚本。
- Source map 需单独生成并确保可访问；Chrome DevTools 中在 `Workers` 面板调试。

---
## 21) 微前端：Module Federation vs Import Maps/SystemJS；共享依赖与冲突治理

### 模式对比
- **Module Federation（MF）**
  - **构建期**：由 Webpack/Rspack 等生成 `remoteEntry.js`（暴露模块清单与模块加载器）。
  - **运行期**：`__webpack_init_sharing__`/`__webpack_share_scopes__` 在 host/remote 之间做 **共享依赖协商**（`singleton`、`requiredVersion`、`strictVersion`、`eager`）。
  - **优点**：原生支持**运行时共享**与版本仲裁；代码可按“应用域”拆分并动态装配；无需发布时统一打包。
  - **代价**：强绑定特定打包器 runtime；多 runtime 协作（如 Vite + MF）需额外桥接；调试复杂。

- **Import Maps/SystemJS**
  - **Import Maps**：标准化的 **模块映射**（bare specifier → URL），浏览器原生解析；**不提供**版本协商/共享策略。
  - **SystemJS**：运行时加载器，兼容多格式（UMD/AMD/CJS→ESM），可配合 Import Maps 动态挂载。
  - **优点**：与打包器弱耦合、协议层更“标准”；适合以 **Web Components** 为边界的松耦合集成。
  - **代价**：**无共享协商**，需要在平台层统一版本或通过 CDN 唯一化；可能出现多份 React/Vue 并存。

### MF 典型配置
```js
// webpack.config.js (host)
new ModuleFederationPlugin({
  name: 'host',
  remotes: { shop: 'shop@https://cdn.example.com/shop/remoteEntry.js' },
  shared: {
    react: { singleton: true, requiredVersion: '^18.2.0', strictVersion: true },
    'react-dom': { singleton: true, requiredVersion: '^18.2.0' }
  }
});

// webpack.config.js (remote)
new ModuleFederationPlugin({
  name: 'shop',
  filename: 'remoteEntry.js',
  exposes: { './Cart': './src/Cart.tsx' },
  shared: { react: { singleton: true }, 'react-dom': { singleton: true } }
});
```

### Import Maps 统一版本
```html
<script type="importmap">
{
  "imports": {
    "react": "https://cdn.example.com/react@18.2/es2020/react.mjs",
    "react-dom": "https://cdn.example.com/react-dom@18.2/es2020/react-dom.mjs",
    "app-a/": "https://a.example.com/assets/",
    "app-b/": "https://b.example.com/assets/"
  }
}
</script>
```

### 共享依赖与冲突治理
- **单例库（React/Vue/Router）**：MF 用 `shared.singleton`; Import Maps **用同一 URL** 强制唯一。
- **版本冲突**：MF `strictVersion` 直接拒绝不兼容；Import Maps 通过“平台统一映射 + 灰度回滚”控制。
- **CSS/命名冲突**：以 **Shadow DOM/Web Components** 或 **CSS Modules** 隔离；公共 Design Token 通过 Custom Properties 下发。
- **运行时隔离**：不同框架/版本通过 **iframe/Shadow Realm（实验）** 或事件总线隔离交互边界。
- **发布治理**：平台提供 **远端配置中心** 控制 remote 版本与 Import Map，支持“金丝雀/回滚”。

---

## 22) CSP 与构建：nonce/hash 内联、禁用 eval、WASM 与 HMR/Sourcemap 适配

### 策略要点
- **脚本**：`Content-Security-Policy: script-src 'self' 'nonce-<rand>' 'strict-dynamic';`  
  - 生产：禁止 `unsafe-inline` / `unsafe-eval`。  
  - Dev：尽量使用 **非 eval** 源映射与 HMR 注入；必要时为 Dev 环境单独放宽策略。
- **样式**：优选外链/nonce，避免行内 `<style>`；Tailwind 等 JIT 输出为文件。
- **WASM**：部分浏览器要求 `wasm-unsafe-eval`；若策略严格，需改为 **静态 URL + streaming instantiate** 并在评审范围内放开该指令。

### Webpack/Vite 适配
- **Source Map**：生产用 `hidden-source-map` 或 `nosources-source-map`；开发用 `cheap-module-source-map`（非 eval）。  
- **去 eval**：Webpack 禁用 `eval` 系列 devtool；Vite/React 刷新在现代实现下不依赖 eval。  
- **HMR nonce 注入**
  - Webpack：设置 `__webpack_nonce__`（style-loader/script）或用 `HtmlWebpackPlugin`/中间件为注入脚本加 `nonce`。
  - Vite：`transformIndexHtml` 注入 `nonce`；自定义 HMR client 在创建 `<script>` 时带上 `nonce`。

```ts
// vite.config.ts - 注入 CSP nonce
export default {
  plugins: [{
    name: 'csp-nonce',
    transformIndexHtml(html) {
      const nonce = process.env.HTML_NONCE!;
      return html.replace(/<script type="module"/g, `<script nonce="${nonce}" type="module"`);
    }
  }],
  server: { hmr: { overlay: true } }
}
```

### WASM 与跨源隔离
- 启用 `COOP: same-origin` 与 `COEP: require-corp` 以支持 `SharedArrayBuffer` 与部分并发解码；WASM 走 `instantiateStreaming(fetch(url))`。
- 若 CSP 严格拒绝 `wasm-unsafe-eval`，仅能通过 **放宽该指令** 或迁移至 **Worker + streaming** 且经安全审查例外放行。

---

## 23) 国际化分包：动态加载、路由/区域懒加载与缓存键设计

### 加载策略
- **路由维度**：按页面/模块拆分字典（`common`, `home`, `product`），配合动态导入。
- **语言维度**：`[lang]/[namespace].json` 独立文件，便于 CDN 缓存与增量更新。

```ts
// i18n.ts
export async function loadDict(lang: string, ns: string) {
  // Vite/Rollup：构建时保留可枚举的目录（避免完全动态路径）
  return (await import(/* @vite-ignore */ `./locales/${lang}/${ns}.json`)).default;
}
```

### 缓存键与回源
- **HTTP 头**：`Vary: Accept-Language`（若按请求头回退选择），或通过 URL 前缀 `/en/..` 显式区分（更利于 CDN 缓存）。
- **ETag/Cache-Control**：语言文件设 `max-age=86400, stale-while-revalidate=600`；重大版本递增 `?v=ts` 失效。
- **灰度**：Import Maps/配置中心可按区域/租户切换语言包版本。

### 防错与可观测
- 字典键名稳定、避免 UI 词序耦合；**缺词日志**上报与回退到默认语言；对敏感场景（结算）采用 **server 预注入** 数据避免首屏抖动。

---

## 24) 兼容与 Polyfill：browserslist、core-js 注入、`@vitejs/plugin-legacy` 的取舍

### 目标与注入
- **browserslist** 决定转译与 polyfill 目标；构建工具/Autoprefixer/`preset-env` 共享之。
- **Babel preset-env + core-js**
  - `useBuiltIns: 'usage'`：按用量注入（颗粒细，易重复注入）。  
  - `useBuiltIns: 'entry'`：在入口引入 `core-js/stable`、`regenerator-runtime/runtime`，由 preset 裁剪。
- **Vite Legacy**：输出 **现代** 与 **旧版** 两套产物，旧版注入 SystemJS + polyfill 清单，按 UA 自动降级加载。

```ts
// vite.config.ts
import legacy from '@vitejs/plugin-legacy';
export default {
  plugins: [
    legacy({
      targets: ['defaults', 'not IE 11'],
      additionalLegacyPolyfills: ['regenerator-runtime/runtime']
    })
  ]
}
```

### 代价与边界
- 体积与请求数上升（双产物 + polyfill）；运行时 SystemJS 开销。
- 仅在确有旧设备需求时启用；否则通过业务统计与 A/B 验证“去兼容”。

---

## 25) CSS Tree-Shaking 与 Critical CSS：误删风险与首屏策略

### Tree-Shaking（Purge）
- 工具：Tailwind JIT（content 扫描）、PurgeCSS、Lightning CSS 的 `unused`。  
- **误删来源**：动态类名（`text-${color}`）、运行时字符串拼接、第三方组件内联类。  
- **对策**：safelist（正则/枚举）、在 TS 中以常量枚举导出类名、对第三方样式建立白名单。

```js
// tailwind.config.cjs
module.exports = {
  content: ['./src/**/*.{ts,tsx,html}', './node_modules/some-lib/**/*.{js}'],
  safelist: [{ pattern: /(bg|text|border)-(red|blue|green)-(100|500)/ }]
}
```

### Critical CSS
- SSR/构建时抽取**首屏关键 CSS**（`critters`、`penthouse`）并内联，其余样式延迟加载。  
- 约束：内联大小控制在 10–20KB 以内，避免阻塞与缓存命中下降；多路由可按模板级别分片内联。
- 与代码分割协同：路由级 CSS 拆分 + 关键路径内联，避免单一 “global.css” 巨无霸。

---

## 26) 产物分析与体积预算：工具与 CI 阈值

### 分析工具
- **Webpack**：`webpack-bundle-analyzer`、`stats.json` + 可视化。  
- **Rollup/Vite**：`rollup-plugin-visualizer` 生成 `stats.html`；对比构建前后差异。
- **细项**：gzip/br 后大小、模块重复（重复引入不同版本）、长尾大依赖（如 `moment` 全量、`lodash` 按需）。

### CI 预算（示例）
```bash
# 统计 dist 主要入口的 gzip 大小，超阈值失败
find dist/assets -name '*.js' -maxdepth 1 \
  -exec bash -c 'sz=$(gzip -c "$1" | wc -c); echo $sz $1' _ {} \; \
  | sort -nr | tee sizes.txt

# 阈值 230KB（示例）
awk '$1>230000{print "oversize:", $2; fail=1} END{exit fail}' sizes.txt
```
- 按 **入口/路由** 建立不同预算；对新增依赖强制走 PR 审查（列出新增 3rd-party 清单）。

---

## 27) 预加载与优先级：preload/prefetch、`modulepreload`、Priority Hints

### 概念与用法
- **`<link rel="preload">`**：声明**关键资源**（`as=script/style/font/image`），提升下载优先级。
- **`<link rel="modulepreload">`**：预加载 **ESM 依赖图**（Vite/Rollup 自动注入对分片的 modulepreload）。
- **`<link rel="prefetch">`**：空闲时预取**可能**用到的资源（优先级低，可能被丢弃）。
- **Priority Hints**：`<img fetchpriority="high">`/`<link rel="preload" as="image" imagesrcset="..." fetchpriority="high">`。

```html
<link rel="preload" as="style" href="/assets/above-the-fold.css">
<link rel="modulepreload" href="/assets/chunk-abc.js">
<link rel="preload" as="font" href="/assets/inter.woff2" type="font/woff2" crossorigin>
```

### 与分片协同
- 首屏路由：为**路由壳 + 首屏依赖**做 `preload/modulepreload`。  
- 次级路由：在可预判交互（hover/viewport）时调用框架 `prefetch()` 或注入 `prefetch`。  
- 注意避免**重复预加载**与错误 `as` 导致缓存 miss。

---

## 28) 多页面应用（MPA）：多入口、共享运行时与缓存

### 配置
- **Webpack**：多个 `entry` + 多实例 `HtmlWebpackPlugin`，每页注入对应 chunk。  
- **Vite**：
  ```ts
  // vite.config.ts
  export default {
    build: {
      rollupOptions: {
        input: { home: 'src/home.html', admin: 'src/admin.html' }
      }
    }
  }
  ```

### 共享与隔离
- **共享运行时**：抽取公共依赖（如 UI 库）到 **长期缓存** chunk（`SplitChunks`/`manualChunks`）。  
- **路由隔离**：不同页面的业务代码完全分片，互不影响首屏；公共资源（字体/基础样式）走 CDN 指纹缓存。

### HTML 生成与 SEO
- 每页独立 `<head>` 元信息与 `preload` 列表；按页面特征输出 **定制关键 CSS**。

---

## 29) 环境分层与配置管理：Webpack `mode/merge`、Vite `.env.*` 与 `define` 协同

### Webpack
- 基础配置 + 环境差异通过 `webpack-merge` 组合；`mode` 影响默认优化（压缩、tree-shaking）。
- 常量注入：
  ```js
  new DefinePlugin({ __API__: JSON.stringify(process.env.API_BASE) })
  ```

### Vite
- `.env.development` / `.env.production` / `.env.staging`，仅 `VITE_` 前缀注入到客户端。
- 复杂场景可用 `loadEnv(mode, process.cwd(), '')` 读取并做 **类型收敛**；`define` 注入编译期常量以便 DCE。
- **安全边界**：敏感变量只在服务端使用（SSR/构建脚本），客户端仅保留公开参数。

```ts
// vite.config.ts
import { loadEnv, defineConfig } from 'vite';
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), 'VITE_');
  return {
    define: { __BUILD_TARGET__: JSON.stringify(mode) },
    build: { rollupOptions: { /* 按环境分片/压缩 */ } }
  };
});
```

---

## 30) 供应链与安全：锁定、SRI、审计/SCA、恶意包与签名

### 基础控制
- **锁定与可复现**：CI 使用 `npm ci` / `pnpm install --frozen-lockfile`；只允许通过 PR 更新 lock。  
- **私有源与镜像**：限制可解析 registry（`.npmrc`），启用 **scope 白名单**。

### 审计与监控
- `npm audit`/`pnpm audit`、Snyk/Whitesource 等 SCA 工具纳入 CI；对高危/可利用漏洞设阻断阈值。  
- **可疑脚本**：默认禁用 `postinstall`（`--ignore-scripts`），对确需脚本的包做白名单；开启 `npm config set fund false` 避免社交工程诱导。

### SRI 与 HTML 产物
- **Subresource Integrity**：对外链脚本/样式生成 `integrity=sha384-...`；Webpack：`webpack-subresource-integrity`，Rollup/Vite：`rollup-plugin-sri`。  
- 注意：SRI 要求同源或 CORS 允许；与 `modulepreload`/指纹名一致性。

### 签名与溯源
- **Provenance/签名**：启用 NPM 包供应链签名（如 npm provenance、Sigstore/cosign 在制品仓库层）。  
- **构建工件签名**：Release 产物（zip/tar）在 CI 用 GPG 签名并上传校验和；对 Docker 镜像用 cosign 附加签名与 SBOM（Syft/Grype）。

### 第三方治理
- 引入前做 **体积/许可/更新频率** 评估；建立“高风险依赖清单”（eval、动态加载、隐私读取）。  
- 定期运行 **依赖修剪**（`depcheck`）与多版本合并（`pnpm dedupe`），避免重复注入与幽灵依赖。

---
## 31) 确定性构建：records、deterministic ids、哈希稳定性；避免 chunk 名漂移

**目标**：小改动不应引起大面积 `contenthash` 漂移，从而命中长期缓存。

- **Webpack**
  - `optimization.moduleIds/chunkIds: 'deterministic'`（v5）：基于路径/内容的稳定 ID，替代 v4 的 `records` 主用场景。
  - `output.filename/chunkFilename: '[name].[contenthash:8].js'`：文件名仅随内容变化。
  - `optimization.runtimeChunk: 'single'`：将 runtime/manifest 抽离，避免其波及业务 chunk。
  - `splitChunks.cacheGroups`：按**领域**拆 vendor，减少“一处升级 → 全站 vendor 失效”。
  - `experiments.realContentHash`（v5 默认启用）：避免因模块顺序变更导致 hash 漂移。
  - 仍需 `recordsPath` 时（跨进程/多机复用 ID）：在 v5 仍可用，但优先 deterministic。
- **Rollup/Vite**
  - `output.manualChunks`：稳定 vendor 边界。
  - `output.entryFileNames/chunkFileNames/assetFileNames` 使用 `[hash]` + 目录规范（如 `assets/[name].[hash].[ext]`）。
  - 避免“动态导入路径拼接”导致 chunk 边界不稳定：尽量枚举明确分支。
- **CSS 稳定性**
  - 抽取顺序不同会影响 CSS hash：固定导入顺序；PostCSS/Minifier 归一化；注意“条件导入”造成顺序波动。

---

## 32) WASM 集成：wasm-pack / vite-plugin-wasm；同步/异步初始化；COOP/COEP 与跨源隔离

- **Rust→WASM**
  - `wasm-pack build --target web|bundler` 产出 npm 包；`bundler` 适合 Webpack/Rollup/Vite。
  - Vite：可用内置 `?init` 方案或 `vite-plugin-wasm`/`vite-plugin-rsw`（Rust workspace 友好）。
- **加载形态**
  - **异步初始化**（推荐）：`import init from './pkg/app_bg.wasm?init'; await init();`
  - **同步初始化**：需 `sync WebAssembly` 支持（Webpack `experiments.syncWebAssembly`），在多数现代工具链中已不建议。
  - `instantiateStreaming(fetch(url), imports)` 优先（正确 `Content-Type`），回退到 `arrayBuffer`。
- **线程/并发**
  - WASM 线程/`SharedArrayBuffer` 要求**跨源隔离**：HTTP 响应头 `Cross-Origin-Opener-Policy: same-origin`、`Cross-Origin-Embedder-Policy: require-corp`。
  - 资源需 `Cross-Origin-Resource-Policy: same-site|cross-origin` 配合；第三方源需 `CORP`/`COEP` 兼容。
- **打包细节**
  - Webpack：`asset/resource` 发射 `.wasm`；或 `experiments.asyncWebAssembly` 直接处理导入。
  - Vite：生产环境对 `.wasm` 生成指纹 URL；注意 base/publicPath，确保 Worker/子路径部署可寻址。
- **类型 & DX**
  - 通过 `wasm-bindgen` 生成 TS 声明；对大对象交互使用 `pass-by-reference`（索引/指针）减少拷贝。

---

## 33) 新一代打包器：Rspack / Turbopack / Farm 架构、兼容性与迁移

- **Rspack（Rust）**
  - 设计目标：**Webpack 生态兼容 + 高性能**。Loader/Plugin 与 Webpack **高度对齐**（多数插件可用），Dev/Prod 构建显著提速。
  - 适合：现有 Webpack 项目低摩擦迁移；保留 `splitChunks`、MF 等模式。
  - 迁移：优先替换不兼容插件；核对 `resolve`、`experiments` 差异；验证 HMR/SourceMap 在复杂链路下的一致性。
- **Turbopack（Rust）**
  - 核心：**增量图 + 按需编译**，面向大型多入口与 Next.js 场景；开发态极快，生产特性逐步覆盖。
  - 兼容：提供 Webpack 兼容层但并未完全等价；特定 Loader/Plugin 需替代方案。
  - 评估：在“**功能完备度 vs 性能**”间权衡；落地于 Next 系列时优先跟随官方推荐版本。
- **Farm（Rust）**
  - API 类 Rollup/Vite，强调**插件/loader 双模型**与并行流水线。
  - 适合：Vite/Rollup 模式的应用构建，追求更快的生产打包。
- **通用迁移流程**
  1. 列出插件/loader 清单 → 对照兼容矩阵。
  2. 用“最小闭环”路由/页面先跑通 dev/prod，验证 HMR/代码分割/动态导入。
  3. 对关键能力（MF/SSR/Workers）逐项验收；对未覆盖能力保持旧链路或局部回退。

---

## 34) CI/CD：依赖缓存、Docker 层缓存、产物与 Source Map 上传、回滚

- **依赖缓存**
  - `pnpm store`/`npm cache` 缓存；使用 `--frozen-lockfile|ci`。缓存键包含 lockfile hash + Node 版本 + 包管理器版本。
  - Monorepo：利用 Nx/Turbo **远程缓存**复用构建工件。
- **Docker**
  - 多阶段构建：`deps -> builder -> runtime`；将 lockfile 与 `package.json` 提前 COPY 以命中层缓存。
  - 产物落地 `/app/dist`，镜像只带运行时所需文件；利用 `buildkit` 并行与 `--mount=type=cache`。
- **工件与 Source Map**
  - 生成并**上传**到制品库/错误平台（Sentry 等）；前端产物上传 CDN/对象存储，开启 `immutable` 缓存。
  - 生产仅使用 **hidden/nosources** 源图；CI 保存构建元数据（git sha、构建时间、依赖摘要）。
- **回滚策略**
  - 产物**不可变版本化**（以 `git sha` 命名）；灰度发布（按流量/区域）与健康检查失败自动回滚。
  - Import Maps/MF/配置中心支持**远程切换**依赖块版本，实现原地回滚。

---

## 35) 疑难构建排错：循环依赖、CJS/ESM 互操作、`Cannot use import statement outside a module`

- **循环依赖**
  - 症状：运行时 `undefined`、模块初始化顺序异常、HMR 热更丢状态。
  - 工具：`madge`、Webpack `stats`、Rollup `--bundleConfigAsCjs` + `--verbose`；在 CI 设为**阻断**。
  - 修复：抽出公共常量/类型；延迟引用（函数级 import）、解耦双向依赖。
- **CJS/ESM 互操作**
  - `default` 导入 CJS → 得到 **命名空间对象**（无默认导出）；使用 `import * as pkg` 或 `require()`。
  - TS `esModuleInterop` 仅在编译层面“容忍”，打包产物仍需正确的 interop（Rollup `output.interop: 'compat'`）。
  - Node 运行：确保 `type: 'module'`/`.mjs`，或输出 CJS；避免同包双格式被同时解析（Dual Package Hazard）。
- **“Cannot use import statement outside a module”**
  - 运行环境未启用 ESM（Node/测试框架）；或打包输出为 ESM 却在 CJS 环境加载。
  - 解决：统一目标格式；为测试/Jest 配置 `transform`；Node 中使用 `--experimental-vm-modules`（取决版本）或切换到 `ts-node/tsx` ESM 模式。

---

## 36) Parcel：零配置、缓存与并行、内容感知与 HMR；与 Vite/Webpack 的取舍

- **零配置**：基于文件类型与 `package.json` 字段自动推导 pipeline；默认内置 TS/Babel/PostCSS/Image 等。
- **缓存与并行**：`.parcel-cache` 存储中间结果，内容哈希驱动；多 worker 并行编译。
- **内容感知**：自动按 `browserslist` 决定转译/Polyfill；自动代码分割与动态导入处理。
- **HMR**：内置 runtime，通过 WS 推送更新；CSS/JS 细粒度替换。
- **取舍**
  - 优点：极快入门、低配置成本、内建合理默认。
  - 局限：生态可定制性弱于 Webpack/Vite；高级场景（MF、复杂 SSR、细粒度分片）需要额外工作或不适配。
  - 适用：中小型站点/工具页/静态站 + 轻服务端。

---

## 37) Turbopack：增量图/按需编译；兼容层与迁移评估

- **核心理念**：基于内容寻址的**增量依赖图**，文件更改只重编相关节点；按需编译减少冷启动。
- **开发体验**：大仓/多入口下具有显著速度优势；HMR 传播基于精确依赖。
- **兼容层**：提供对 Webpack 语义的兼容，但**非完全等价**；部分 Loader/Plugin 需要专用替代。
- **迁移评估**
  - 现网复杂能力（SSR、RSC、MF、Worker、CSS-in-JS）逐项验证。
  - 优先在**开发态**或局部路由试点；生产构建能力以官方支持矩阵为准。
  - 指标：冷/热延迟、首屏体积、构建稳定性、Source Map 准确性。

---

## 38) Rspack / Farm：架构要点、兼容策略、迁移与坑位

- **Rspack**
  - **Pipeline**：Rust 实现解析/转换/优化，复用 JS 生态 Loader/Plugin 接口（高兼容）。
  - **MF/TS/React/Vue**：官方适配良好；`css` 处理、`mini-css-extract` 等有等价能力。
  - **坑位**：个别不常见插件 Hook 不兼容；Source Map 行列精度差异；watch 边缘变更触发策略需验证。
- **Farm**
  - **并行编译 + Rust AST**；插件 API 贴近 Rollup/Vite，易上手；生产构建速度优势明显。
  - **生态**：在复杂 SSR/边缘目标上需评估；与 Vite 插件互通度有限。
- **迁移步骤**
  1. 核心构建链路先通：TS/CSS/资产/别名/环境常量。
  2. 动态导入、代码分割、长缓存策略复刻。
  3. E2E 基准：HMR、SSR、Workers、MF。逐步替换不兼容插件。

---

## 39) esbuild 插件与二开：`onResolve/onLoad`、namespace/虚拟模块、缓存键与并发安全

- **生命周期**
  - `onResolve({ filter, namespace })`：解析路径 → 返回 `path/namespace`。
  - `onLoad({ filter, namespace })`：加载模块内容 → 返回 `contents/loader`。
- **虚拟模块**
  ```js
  build.onResolve({ filter: /^virtual:md$/ }, () => ({ path: 'virtual:md', namespace: 'md' }));
  build.onLoad({ filter: /.*/, namespace: 'md' }, async () => {
    const code = `export const html = ${JSON.stringify(renderMD())}`;
    return { contents: code, loader: 'js' };
  });
  ```
- **别名示例**
  ```js
  build.onResolve({ filter: /^@lib\// }, args => ({ path: path.join(srcDir, args.path.slice(5)) }));
  ```
- **缓存与并发**
  - 以 `path + query + loader + tsconfig + env` 为键；命中后直接返回。
  - 并发 `onLoad` 中避免共享可变状态；必要时使用 `Map` + `Promise` 去重。
- **常见插件**
  - MD/CSV/SVG 组件化；`import.meta.glob` 扫描；HTTP 导入（开发态）代理；环境常量内联。

---

## 40) Rollup/Vite 插件生命周期：`resolveId/load/transform/generateBundle`；`this.emitFile` 与代码分割/指纹

- **关键 Hook**
  - `resolveId`：决定模块 ID 与外部化（`return { id, external }`）。
  - `load`：返回源码或二进制（`code/map` 或 `source`）。
  - `transform`：对源码做转换（可基于路径/内容）。
  - `moduleParsed`：已解析 AST（分析依赖/收集元信息）。
  - `renderChunk`：针对单 chunk 的最终代码变换（如 banner、polyfill 注入）。
  - `generateBundle`：可创建/修改产物、写入附带文件。
  - `writeBundle`：写盘后（用于上报/后处理）。
- **`this.emitFile`**
  - `type:'asset'`：发射静态资源，返回 `fileName`（可在 `generateBundle` 阶段依据 hash 稳定命名）。
  - `type:'chunk'`：发射一个代码子图（用于虚拟入口/代码分割），`preserveSignature` 控制导出签名。
  - 与指纹/分割关系：通过 `emitFile` + `referenceId` → `this.getFileName(ref)` 获取最终带 hash 的文件名，避免硬编码路径。
- **Vite 细节**
  - 开发态仅用到 `resolveId/load/transform/handleHotUpdate`；生产态走 Rollup 的完整生命周期。
  - 与多插件协作：注意 **plugin order**（`enforce: 'pre'|'post'`）与 **id 命名空间**，避免重复处理或错过时机。
---
## 41) Webpack 插件（Tapable）：compiler/compilation/NormalModuleFactory 关键钩子与 Asset Pipeline；如何在生成阶段安全地修改产物

### 钩子分层
- **顶层 compiler**（单次构建/监听周期全局）
  - `environment/afterEnvironment`：环境准备。
  - `initialize`：插件可注册全局状态。
  - `beforeRun/run/watchRun`：构建启动。
  - `normalModuleFactory` / `contextModuleFactory`：注册模块创建期钩子。
  - `compilation`：创建 `compilation` 时触发（为一次构建注册更细粒度钩子）。
  - `emit`（v5 推荐使用 `compilation.hooks.processAssets`）/`assetEmitted`：写文件阶段。
  - `done/failed`：收尾。
- **compilation**（单次构建的数据平面）
  - 模块级：`buildModule`、`succeedModule`、`finishModules`。
  - 优化期：`optimizeModules/Chunks`、`afterOptimizeAssets`（v5 用 `processAssets` stages）。
  - 产物期：`processAssets`（带 stage：`ADDITIONAL`→`PRE_PROCESS`→`ADDITIONS`→`OPTIMIZE`→`OPTIMIZE_SIZE`→`DEV_TOOLING`→`HASH`→`OPTIMIZE_TRANSIENT`→`PROCESS_ASSETS_STAGE_SUMMARIZE`→`REPORT`）。
- **NormalModuleFactory**
  - `beforeResolve/afterResolve`：可重写解析结果、注入 loader。
  - `createModule`：自定义模块类型（虚拟模块、数据模块）。

### 产物安全修改（v5）
- 使用 `compilation.updateAsset`/`emitAsset`/`deleteAsset` 搭配 `processAssets` 指定 **stage**，避免与其他插件竞态：
  ```js
  compiler.hooks.compilation.tap('MyPlugin', (compilation) => {
    const { RawSource, SourceMapSource } = compiler.webpack.sources;
    compilation.hooks.processAssets.tap(
      { name: 'MyPlugin', stage: compiler.webpack.Compilation.PROCESS_ASSETS_STAGE_OPTIMIZE },
      (assets) => {
        for (const [file, src] of Object.entries(assets)) {
          if (file.endsWith('.js')) {
            const code = src.source().replace('__BUILD__', JSON.stringify(Date.now()));
            compilation.updateAsset(file, new RawSource(code));
          }
        }
      }
    );
  });
  ```
- 若需要保留 Source Map，使用 `SourceMapSource` 并从旧 `Source` 读取 `map()` 合成新映射。
- 不要在 `emit` 之后再改写文件；不要直接写磁盘（交由 Webpack 负责），否则破坏缓存与并发。

---

## 42) 持久化缓存与失效：Webpack cache、Rspack/Vite 预构建缓存 key 组成；定位“缓存未失效/错误命中”

### Webpack 文件系统缓存
```js
cache: {
  type: 'filesystem',
  buildDependencies: {
    config: [__filename],     // 配置变化失效
    tsconfig: ['tsconfig.json']
  },
  name: 'app-web'             // 多目标构建区分缓存目录
}
```
- **命中键**由：loader 版本与 options、环境变量（对 loader 可见的）、源文件内容 hash、`resolve` 配置、`devtool`、`mode` 等共同决定。
- **常见未失效原因**
  - 自研 loader 未将 **options** 纳入缓存键（需 `this.getOptions()`）。
  - 使用环境变量但 loader 未声明依赖（需在 loader 中 `this.addBuildDependency` 或将 env 体现在选项里）。
  - 外部资源（如生成代码的模板文件）变化未被追踪（`addDependency`）。

### Rspack/Vite 依赖缓存
- **Vite `node_modules/.vite`**：key 包含依赖内容、版本、`optimizeDeps` 配置、插件解析影响；变更 alias/条件导出会触发**重建**。
- **排查**：`vite --debug deps`，删除 `.vite` 或 `--force`；对 workspace 包发生源码变更需 `exclude`，避免“缓存旧编译结果”。

### 定位步骤
1. 复现实验：删除缓存目录重建，比较产物差异。
2. 导出 `stats.json` 或 `--profile`，核对每个模块的 `cached` 标记与命中来源。
3. 对不命中的链路分析：loader/插件版本是否稳定、选项是否纯数据、是否引用未声明的外部文件。

---

## 43) Dev Server 代理与 HTTPS：HTTP/WS 代理与 CORS；自签名证书/HTTP3 开发约束与排错

### 代理与 CORS
- 开发代理用于避开浏览器 CORS，**由 dev server 发起**请求同源转发：
  ```ts
  // Vite
  server: {
    proxy: {
      '/api': {
        target: 'https://backend.local',
        changeOrigin: true,
        ws: true,
        rewrite: p => p.replace(/^\/api/, '')
      }
    }
  }
  ```
- 对 WebSocket 需要 `ws: true`；上游若要求 `Origin` 校验，配合 `changeOrigin`。
- 若后端需要 Cookie，会话保持需设置 `secure` 和 `sameSite`；代理层应透传 `set-cookie`。

### HTTPS 与证书
- 本地启用 HTTPS：Vite/webpack-dev-server 支持 `server.https: { key, cert }`；使用自签名证书需将根证书导入系统信任。
- 常见报错：
  - `NET::ERR_CERT_AUTHORITY_INVALID`：证书未受信。
  - `Mixed Content`：页面 HTTPS 下代理到 HTTP 资源；需后端启用 HTTPS 或通过反向代理升级。
- HTTP/2/3：H3 需 QUIC 支持（本地较繁琐），一般生产网关开启，开发保持 H2/H1 即可。

---

## 44) 多目标产物：Node/Edge/Browser 三套构建；platform/target/conditions 与 polyfill 策略

### 产物矩阵
- **Browser（现代 ESM）**：`target: 'es2018+'`，不含 Node 内置，`exports: { browser }`。
- **Node（CJS/ESM）**：`target: 'node18+'`，`exports: { import, require }`，`external` 处理依赖。
- **Edge/Workers**：Web 平台 API（`fetch`, `URL`, `crypto.subtle`），**禁止 Node 内置**，产物满足 `module`/`worker` 条件。

### 条件导出（package.json）
```json
"exports": {
  ".": {
    "types": "./dist/index.d.ts",
    "browser": "./dist/browser/index.mjs",
    "edge": "./dist/edge/index.mjs",
    "import": "./dist/node/index.mjs",
    "require": "./dist/node/index.cjs",
    "default": "./dist/browser/index.mjs"
  }
}
```

### Polyfill 策略
- Browser：按 `browserslist` 注入（必要时 Vite Legacy 双轨）。
- Node：不应打包浏览器 polyfill；利用运行时能力。
- Edge：使用 **可移植依赖**（`undici` 在 Node 端，Edge 原生 `fetch`）；避免 `Buffer`/`path` 等 Node-only。

---

## 45) 包导出策略：`exports`/`imports`/`types`/`main`/`module` 与 Dual Package Hazard；一致性验证

### 导出字段
- **现代优先使用 `exports`**，精确声明子路径与条件；`main/module` 为兼容旧工具。
- `types` 指向类型入口；或在 `exports['.'].types` 下细化。
- **子路径导出**：显式列出（`"./utils": "./dist/utils/index.mjs"`），避免“私有路径”被消费者直接 import。

### Dual Package Hazard
- 同时提供 CJS 与 ESM，若解析不一致会导致**同包加载两份**。对策：
  - 通过 `exports` 条件保证 `import`/`require` 指向同一实现（或等价实现）。
  - 在代码中避免状态ful单例跨格式共享。

### 一致性验证
- Node：用 `node --conditions=browser,node --input-type=module` 测试不同条件。
- Bundler：用 Rollup/Vite/webpack 分别打一个 smoketest，验证 Tree-Shaking、sideEffects、生效入口。
- Types：`tsc --noEmit` + `tsd` 测试子路径类型正确性。

---

## 46) 库发布最佳实践：多格式输出、external/peerDependencies、sideEffects 标注与 DCE 空间

- **多格式输出**：`esm`（默认推荐）、`cjs`（Node/老工具）、必要时 `umd/iife`（浏览器直引）。
- **依赖策略**
  - `peerDependencies`：像 `react`、`vue` 等**宿主库**标为 peer，避免重复与版本冲突。
  - `external`：打包时外部化上述依赖，减小包体。
- **Tree-Shaking 友好**
  - 纯 ESM 入口；避免顶层副作用。
  - `package.json: { "sideEffects": false }` 或列举需要保留副作用的文件（如样式）。
- **保持 DCE 空间**
  - 尽量导出**细粒度**函数而非大对象；避免“默认导出整个工具箱”。
  - 对工厂/类实例调用加 `/*#__PURE__*/`，便于压缩器删除未使用实例化表达式。
- **类型与文档**
  - 发布 `d.ts` 与 Source Map（`typesVersions` 指向）；在 README 中标注目标引擎与 Tree-Shaking 支持。

---

## 47) CSS 工具链：Lightning CSS vs PostCSS（Autoprefixer/CSSnano）；新特性编译与静态抽取（vanilla-extract/Linaria）

### Lightning CSS
- 集成转译（嵌套、层、媒体查询范围等）、前缀、压缩，性能优于传统 PostCSS 组合；Parcel/Vite 有集成方案。
- 优点：**一体化**、快；缺点：生态插件少于 PostCSS。

### PostCSS 生态
- `postcss-preset-env`（含 `autoprefixer`、新语法降级），`cssnano` 压缩。
- 需要自定义流程（RTL、设计令牌注入、平台约束）时灵活度更高。

### 新特性
- `@nest`、`@layer`、容器查询（`@container`）等可由上述工具降级；请与目标浏览器矩阵一致。
- `:has()` 的降级成本高，谨慎使用或保留渐进增强。

### 静态抽取
- **vanilla-extract**：TypeScript 中声明样式，构建期抽取为 CSS，具备摇树能力与类型安全。
- **Linaria**：写类 CSS-in-JS，编译期抽出样式；避免运行时开销。
- 适合：追求**零运行时**与**可摇树**样式体系；缺点：心智/模版语法迁移成本。

---

## 48) Source Map 全链路：TS→Babel/SWC→Minify 的映射；`hidden-source-map`/`external`/`inline` 与隐私合规

### 映射传递
- 每一阶段（TS 编译、Babel/SWC 转译、压缩）都需启用 `sourceMaps: true` 并**注入上一阶段的 map**，最终由压缩器合并：
  ```js
  // 伪代码：transform(code, { sourceMaps: true, inputSourceMap })
  ```
- Loader/插件应返回 `{ code, map }`；否则链路断裂导致行列错位。

### 产物形态
- **inline**：`data:application/json;base64` 直接内联，开发可用，生产禁用。
- **external**：生成独立 `.map` 文件，JS 末尾 `//# sourceMappingURL=...` 指向；生产不建议公开发布。
- **hidden-source-map**：生成 map 但不写入引用注释；配合错误上报平台**私下上传**。
- **nosources-source-map**：只包含映射不含源码；兼顾合规但调试体验差。

### 隐私与合规
- 生产仅上传到 Sentry 等平台，**不要**让 `.map` 公开可下载。
- map 中的 `sourcesContent` 可去除（体积/隐私权衡），但错误平台回溯需访问源码仓库。

---

## 49) “无打包开发”与图失效：Vite/Snowpack 原生 ESM 的 HMR 稳定性；模块图失效/循环依赖导致的热更异常定位

- **原生 ESM**：开发态不聚合 bundle，依赖图由浏览器请求驱动；HMR 依赖服务器维护的 **模块图**。
- **图失效场景**
  - 动态导入路径过于动态（`import(`./${name}.ts`)`）导致图不可预测；需要枚举或虚拟模块路由表。
  - 循环依赖改变导出时序，热替换后出现 `undefined`；通过 `madge` 检测并打破环。
  - 虚拟模块/插件未正确实现 `handleHotUpdate`，导致受影响模块未被标记。
- **定位**
  - 开启 `--debug hmr` 日志查看失效传播链。
  - 检查 `import.meta.hot.data` 的状态迁移，必要时在接受方手动 `accept` 指定依赖。
  - 将重型全局状态迁到 **单例 store**，在 HMR 时保留并显式替换 reducer/handlers。

---

## 50) Monorepo 编排与远程缓存：Nx/Turborepo/Bazel 与 bundler 协作；缓存粒度、工件复用与跨平台缓存

- **任务图（DAG）**
  - 基于 `package.json` 依赖与自定义 `project.json` 构建图；`build`, `test`, `lint`, `typecheck` 等形成可并行/串行的任务集。
- **缓存粒度**
  - 任务输入哈希 = 源码 + 配置 + 环境变量 + 工具版本 + lockfile；命中后复用 **产物目录**（如 `dist/`）与日志。
  - 对前端构建，建议把 **图片/字体预处理** 独立为可缓存任务，避免每次构建重复耗时。
- **远程缓存**
  - CI/本地互通：开发机可拉取 CI 产物（节省冷启动），CI 也复用本地缓存；注意**平台差异**（Linux vs macOS）造成二进制不兼容，需按 `os/arch` 隔离缓存命名空间。
- **与 bundler 协作**
  - 输出目录稳定；产物的 hash/manifest 一并缓存；对 Vite 的 `.vite` 与 Webpack 的 `cache` 目录可作为 **二级缓存**，但要确保 key 与环境一致。
- **回滚与可追踪**
  - 将构建产物与 `git sha` 绑定；缓存条目携带 `tsc`/bundler 版本与 `browserslist` 摘要，便于诊断“错命中”。
- **反模式**
  - 将“安装依赖”与“构建”混在同一任务，导致缓存粒度过粗。
  - 在缓存命中后仍执行产物改写（如注入时间戳），破坏可复现性；时间戳应在**运行时**记录或注入到 `__BUILD_ID__` 常量而非真实文件内容。

---
