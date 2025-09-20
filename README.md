# 面试题清单

> 各 **35 道**、聚焦现代实践的高频面试题列表。

---

## Flutter 高频面试题（35 题）

1) Flutter 渲染管线从 **Widget → Element → RenderObject** 各层分别做什么？如何协同？  
2) 约束-布局-绘制（**constraints/layout/paint**）完整流程是怎样的？`BoxConstraints` 如何影响子组件？  
3) 为什么需要 **Key**？`GlobalKey` / `ValueKey` / `ObjectKey` 的典型使用场景与坑？  
4) `StatelessWidget` vs `StatefulWidget` 的区别与 **State 生命周期**（`initState`/`didChangeDependencies`/`didUpdateWidget`/`dispose`）要点？  
5) `BuildContext` 本质是什么？哪些场景会“拿错 context”（如 `showDialog`、`Scaffold.of`）？  
6) **InheritedWidget / InheritedModel / Provider** 的依赖追踪原理与适用边界？  
7) **Navigator 1.0** 与 **Navigator 2.0（Router API）** 的差异、迁移思路与**深链/状态恢复**实战？  
8) 性能优化：如何减少不必要 **rebuild/repaint**？`RepaintBoundary` 与 `const` 的使用边界？  
9) 动画体系对比：**Implicit / Explicit / AnimatedBuilder / TweenAnimationBuilder / AnimationController** 什么时候选谁？  
10) 手势系统：**HitTest** 流程与 **GestureArena** 冲突裁决机制？  
11) **Isolate/compute** 的适用场景？主 Isolate 的**事件循环/微任务**与帧调度关系？  
12) Flutter 与原生交互：`MethodChannel` / `EventChannel` / `BasicMessageChannel` 的差异与最佳实践？  
13) **PlatformView** 的工作原理、性能影响与常见坑（输入法、滚动嵌套、合成层）？  
14) 图片与缓存：`ImageProvider`、`cacheWidth/height`、预加载策略、**OOM** 排查步骤？  
15) **国际化（i18n）** 与**多时区**：ARB 流程、时间戳一致性与动态更新策略？  
16) 资源与体积优化：**Tree-shaking**、字体子集化、**分包/拆包**、增量更新的实现思路？  
17) 构建与发布：**Flavors** 环境切换、签名、多渠道、CI/CD（Fastlane/GitHub Actions）要点？  
18) **Web/桌面/移动** 的差异：渲染后端、文件系统、窗口管理、剪贴板、权限与路由差异？  
19) 可测试性：**Widget 测试 / Golden 测试 / 集成测试** 的边界、稳定性与 mock/fake/stub 的取舍？  
20) 本地存储与数据层：**Isar/Hive/Sqflite/Drift** 选型维度，离线优先与数据同步策略？  
21) 安全与风控：证书固定（Pinning）、反调试/Root 检测、密钥管理、WebView **JSBridge** 风险？  
22) 内置浏览器/WebView 实现细节：**Hybrid Composition**、Cookie/Storage 一致性、跨域/CSP？  
23) **Skia vs Impeller**、`SchedulerBinding` 帧生成与 **Jank** 排查（Timeline/DevTools）？  
24) **Add-to-App**：将 Flutter 模块嵌入原生 App 的做法与要点？  
25) 多系统共存与切换：在一个 App 中承载“**原生系统 + Flutter 系统**”并隔离？  
26) Dart **空安全（null-safety）**：`late`、`?`、`!`、`required`、`const/final` 的语义与坑？  
27) 状态管理选型：**Riverpod / Bloc(Cubit) / Redux / GetX / MobX** 的思想、粒度、可测试性？  
28) 全链路错误捕获与崩溃上报：`FlutterError.onError`、`runZonedGuarded`、`PlatformDispatcher.onError`、原生崩溃如何统一？  
29) 网络层工程化：`http` vs `dio`、拦截器（鉴权/重试/熔断）、ETag 缓存、弱网/离线策略？  
30) 应用与页面生命周期：`AppLifecycleState`、`WidgetsBindingObserver` 的时序与资源管理？  
31) 推送通知端到端：**FCM/APNs** 接入、Token 刷新、深链与三种前后台场景？  
32) 表单与输入体系：`Form`/`FormField`、`TextEditingController`、`FocusNode`、IME 合成区与节流？  
33) **Sliver** 体系与滚动性能：`CustomScrollView`、`SliverList/Grid/PersistentHeader` 的原理与场景？  
34) 无障碍（**a11y**）与可达性：`Semantics` 树、`MergeSemantics`、TalkBack/VoiceOver 差异与测试？  
35) 自定义与发布插件（Plugin）：**联邦式插件**结构、`MethodChannel` vs **FFI**、多平台兼容策略？

---

##  React 高频面试题（35 题）

1) 并发渲染（**Concurrent Rendering**）在 React 18 中做了什么？调度/可中断渲染/优先级如何工作？  
2) `useTransition` 与 `useDeferredValue` 的区别与典型使用场景？  
3) **Suspense** 在数据获取与代码分割两种场景的机制差异？`fallback`/reveal-order 如何设计？  
4) **React Server Components（RSC）** 模型：什么在服务端？如何与客户端组件协作？限制点？  
5) **Server Actions**（React 19）如何工作？表单渐进增强、鉴权与副作用边界？  
6) `use()` 钩子在 Server/Client 的行为与限制？如何与 Suspense 协同？  
7) **SSR → Streaming → Hydration** 的完整链路？常见 mismatch 与规避策略？  
8) 现代状态管理选型：**Context / Redux Toolkit / Zustand / Jotai / TanStack Query** 各自适用场景？  
9) `useEffect` / `useLayoutEffect` / `useInsertionEffect` 何时使用？对布局/样式注入的影响？  
10) Hooks 的“闭包陷阱”与依赖数组：如何系统性避免 **stale closure**？  
11) `useMemo` / `useCallback` 的收益与成本评估：怎样识别“过度 memo”？  
12) **React Compiler** 的目标与约束：如何自动优化重渲染？对代码提出哪些要求？  
13) 表单策略：受控 vs 非受控；如何与 **Server Actions** 的提交流程融合？  
14) 列表虚拟化（react-window/react-virtual）关键参数与避免“行高抖动”的策略？  
15) Reconciliation 与 `key` 的本质：为什么别用索引？分页/拖拽如何保证稳定性？  
16) **Error Boundary** 的捕获边界在哪里？事件、异步、服务端错误如何处理？  
17) 事件系统演进：委托位置变化（17+）、合成事件 vs 原生事件的工程影响？  
18) Context 的性能问题来源？如何用 **selector**、拆分 provider 或 `useSyncExternalStore` 缓解？  
19) `useRef` 与 state 的职责边界；如何进行 DOM 测量并避免布局抖动？  
20) 资源预加载：与 Suspense/数据协作的推荐做法（preload/prefetch/preconnect）？  
21) 代码分割：`React.lazy`/动态 import 的边界？路由级/组件级与错误边界/回退如何配套？  
22) **Web Vitals（LCP/CLS/INP）** 与 React Profiler：如何定位重渲染与调度瓶颈？  
23) 安全性：`dangerouslySetInnerHTML` 的正确用法？如何防御 XSS？  
24) CSS 策略对比：**CSS Modules / Tailwind / vanilla-extract / CSS-in-JS** 的规模化取舍？  
25) **Portal** 与层叠上下文：模态/抽屉的滚动锁定与焦点陷阱如何实现且不破坏可达性？  
26) 并发特性与数据请求：如何避免“网络瀑布”？RSC 缓存与框架级 cache 的配合？  
27) 测试策略：React Testing Library、**MSW**、计时器与端到端（Playwright/Cypress）如何分层？  
28) 复杂流程建模：在 React 中用状态机/状态图（**XState**）的收益与折中？  
29) **TanStack Query**：查询缓存/失效/变更与并发、Suspense 的协作方式？  
30) 客户端存储与水合：`localStorage/sessionStorage` 的读取时机与一致性处理？  
31) `useId` 的用途与 SSR 一致性；何时使用它而非自生成 id？  
32) 渲染大型表格/数据网格：虚拟化、不可变数据与 `areEqual` 协同策略？  
33) 媒体组件的“受控 vs 非受控”设计；`forwardRef` / `useImperativeHandle` 的场景？  
34) 构建与工具链（**Vite**）：别名/环境变量、按路由分包与产物分析基线？  
35) 生产工程基线：Source Map、环境/密钥、性能护栏与灰度回滚策略？
36) **Fiber 架构**：FiberNode 的关键字段（`tag/flags/lanes/alternate`）与双缓冲树（current/workInProgress）如何协作？  
37) **工作循环**：`render` 与 `commit` 两阶段的源码路径；`beginWork` / `completeWork` 分别负责什么？  
38) **优先级模型**：Scheduler 优先级与 **Lane 模型**如何映射？离散/连续事件与 `transition` 分别落在哪些 lanes？  
39) **更新队列**：函数组件 Hook 的更新链表（baseState/baseQueue）与类组件 `setState` 队列的差异？`eager state` 何时触发？  
40) **Hook 原理**：Dispatcher 如何保证“同序调用”？为什么 Hook 不能放在条件/循环里？`mountXxx/updateXxx` 成对实现的意义？  
41) **Context 传播**：Provider→Consumer 的依赖追踪与变更广播；源码如何做选择性订阅以降低重渲染？  
42) **子节点 Diff（Reconciliation）**：首渲染 vs 更新阶段策略；`key` 如何决定复用/移动/删除？  
43) **Flags 与副作用**：`Placement/Update/Deletion/Passive` 等标志在 commit 期如何收集/执行？从 Effect List 到按 flags 扫描的演进？  
44) **Suspense 实现**：`thenable`/`ping` 机制；挂起与重试的调度；`fallback` 与 `revealOrder` 在源码层如何落地？  
45) **并发中断/恢复**：`workLoop` 如何让出控制？何时丢弃 WIP、回退到 current 并重来？  
46) **`useTransition` / `useDeferredValue`**：内部如何标注“非紧急更新”？哪些路径会降级到 transition lanes？  
47) **Hydration 算法**：SSR 标记与客户端 fiber 对齐；选择性/优先级水合如何避免阻塞首屏？mismatch 处理路径？  
48) **事件系统（17+）**：根容器事件委托；离散/连续事件优先级；合成事件与原生事件在源码中的分层与逃逸边界？  
49) **RSC / Flight 协议**：Server 组件产出的 wire format 如何在客户端重建并与 Client 边界拼接？  
50) **渲染器 host config**：`appendChild/commitUpdate` 等 DOM 渲染器接口的职责；不同渲染器（DOM/Native/自定义）如何复用 reconciler？

---

# Vue 3 高频面试题清单（35 题）

1) Vue 3 的响应式系统（Proxy + effect + scheduler）是如何工作的？依赖收集与触发更新的流程？  
2) `ref`、`reactive`、`shallowRef`、`shallowReactive`、`markRaw` 的适用场景与坑点对比？  
3) `computed` 与 `watch` / `watchEffect` 的差异、依赖追踪边界与清理回调的时机？  
4) 组件通信：`props / emits`、`v-model`（多 `model`、自定义参数）、`defineEmits` 的校验与类型推断？  
5) 组合式 API（Composition API）与 `Options API` 的设计取舍？何时拆分为可复用的 **composable**？  
6) `<script setup>` 宏：`defineProps/defineEmits/withDefaults/defineExpose/defineSlots/defineModel` 各自语义与 TS 推断？  
7) 生命周期钩子：`onMounted`/`onUpdated`/`onUnmounted`/`onBeforeRouteLeave` 等分别何时触发？与 `nextTick` 的关系？  
8) `toRef` / `toRefs` / `customRef` 的使用场景（比如表单防抖）与实现思路？  
9) `provide/inject` 如何保持响应式？为 `inject` 提供默认值与只读约束的最佳实践？  
10) 父子组件 `slots` / 作用域插槽（scoped slots）工作原理？`<slot name>` 与 `v-slot` 语法糖使用注意？  
11) 动态组件与缓存：`<component :is="...">`、`<KeepAlive>` 的缓存策略、`include/exclude`、`activated/deactivated` 的使用？  
12) 路由（Vue Router 4）：动态路由、嵌套路由、命名视图、`beforeEach`/`beforeResolve`/`afterEach` 守卫与 `scrollBehavior`？  
13) 组合式路由守卫：`onBeforeRouteLeave/onBeforeRouteUpdate` 的典型场景与注意事项？  
14) 异步组件与代码分割：`defineAsyncComponent` 选项（`timeout`/`suspensible`/`loadingComponent`/`errorComponent`）？  
15) `Suspense` 在 Vue 中的工作机制与异步 `setup()`、`async` 组件的配合？  
16) `Teleport` 的渲染边界与可达性：模态/抽屉/通知如何正确处理焦点与滚动锁？  
17) 自定义指令（Vue 3 生命周期）：`beforeMount/mounted/updated/unmounted` 的典型用途与指令值变更对比？  
18) 性能优化：避免不必要渲染的策略（组件拆分、`v-once`、`memoized` 计算、事件节流/去抖）与 DevTools 性能分析？  
19) 列表渲染：`key` 的语义、为什么不要使用索引作为 `key`、分页/拖拽/虚拟滚动如何保证稳定性？  
20) 表单与受控/非受控：`v-model` 修饰符（`number`/`trim`/`lazy`）、组件自定义 `modelValue` 的双向绑定规范？  
21) 事件与修饰符：`@click.stop`、`@keydown.capture`、`@scroll.passive` 的底层影响与最佳实践？  
22) 条件/列表指令的差异：`v-if` vs `v-show` 的渲染与性能取舍、`v-for` 与 `v-if` 同节点冲突的处理？  
23) 样式隔离：SFC `<style scoped>` 的实现原理（数据属性选择器）、`:deep()`/`:slotted()`/`:global()` 的作用与风险？  
24) 过渡与动画：`<Transition>`/`<TransitionGroup>` 的类名约定、状态钩子、与 `requestAnimationFrame` 的协同？  
25) 批量更新与异步队列：更新调度何时发生？`nextTick` 的使用与 `flush: 'pre' | 'post' | 'sync'` 对 `watch` 的影响？  
26) 错误处理：`errorCaptured` 钩子、`app.config.errorHandler`、Promise 未处理错误的统一上报？  
27) 服务器渲染（SSR）与水合：同构渲染链路、常见 hydration mismatch 成因与规避（非确定性输出/客户端专属逻辑）？  
28) Nuxt 3 基本机制（如适用）：`useAsyncData`/`server routes`/缓存与失效、`<ClientOnly>` 的使用场景？  
29) 状态管理：**Pinia** 的核心概念（`state/getters/actions`）、`storeToRefs`、插件/持久化、与 Vuex 迁移的关键差异？  
30) 大型表格/列表性能：虚拟滚动（Vue Virtual Scroller 等）、不可变数据与行/单元格组件化策略？  
31) TypeScript 最佳实践：为 `props`/`emits`/`slots`/`expose` 提供类型、泛型组件与 Volar 推断常见问题？  
32) 组件设计规范：单向数据流、不变性、`emit` 事件命名（`update:modelValue`）、可组合/可测试的 API 形态？  
33) 测试：Vue Test Utils + Vitest 的渲染/交互测试、`flushPromises`、mock 组合式函数与 Pinia store 的方法？  
34) 构建与工程化：Vite 配置（别名/环境变量 `import.meta.env`）、按路由/组件分包、产物分析与体积预算？  
35) 安全与可达性：`v-html` 的消毒与 XSS 预防、表单 label/aria-*、焦点管理在 Teleport/过渡场景中的要求？

---
# Svelte 高频面试题清单（35 题）

1) Svelte 的“编译型框架”本质是什么？它与 React/Vue 的运行时范式有何根本区别？  
2) Svelte 4 的响应式语法 `$:`、赋值触发更新（`count += 1`）的原理与局限？  
3) Svelte 5 的 **Runes**（如 `$state/$derived/$effect`）相对 Svelte 4 的响应式声明有何变化与迁移要点？  
4) 组件通信：`props` 传参、事件派发 `createEventDispatcher`、原生事件转发 `on:*` 的对比与边界？  
5) **Stores**：`writable/readable/derived` 的使用场景、订阅/解除订阅模式与内存泄漏防护？  
6) `get` 同步读取 store 的时机与风险？什么时候应避免直接 `get(store)`？  
7) 生命周期：`onMount`、`beforeUpdate`、`afterUpdate`、`onDestroy` 的触发时机与典型用法？  
8) `tick()` 的作用是什么？它如何保证在 DOM 更新后执行，常见的误用场景有哪些？  
9) 条件与列表：`{#if}/{:else if}/{:else}`、`{#each}`、`{#key}` 的最佳实践与性能影响？  
10) 模板指令绑定：`bind:value`、`bind:this`、`bind:group`、`bind:clientWidth` 等双向/单向绑定的工作机制？  
11) 作用域插槽与命名插槽：`<slot>`、`<slot name>`、插槽 props 的传递与类型约束？  
12) 动画与过渡：`transition:` / `animate:` / `motion`（`spring/tweened`）的适用边界与性能注意事项？  
13) 自定义 **Action**（`use:xxx`）的设计：参数更新、清理函数、与浏览器原生交互的标准模式？  
14) 自定义 **Store**：如何实现带副作用/清理/持久化（localStorage）的 store？  
15) 可访问性（a11y）：Svelte 编译器的 a11y 提示能覆盖哪些问题？模态/焦点陷阱/键盘导航如何落地？  
16) 组件样式：`<style>` 作用域隔离的实现机制、`:global()` 的风险、与 Tailwind/CSS Modules 的协作策略？  
17) TypeScript 支持：`svelte-preprocess`/`tsconfig`/`ambient.d.ts` 的关键配置与常见坑？  
18) 服务器端渲染（SSR）与水合（Hydration）在 Svelte/SvelteKit 中的链路是什么？  
19) SvelteKit 路由系统：基于文件的 `+page.svelte/+page.ts/+layout.svelte/+layout.ts` 的职责划分？  
20) 数据获取：`load`（客户端/服务端）与 `+page.server.ts` 的差异、何时选择 server load？  
21) SvelteKit 表单 **Actions**：进程内执行、返回 `fail/redirect`、与渐进增强的关系？  
22) Endpoints/API 路由：`+server.ts` 的 `GET/POST` 处理、`RequestEvent`、Cookies、Headers 与 CORS 策略？  
23) 会话与鉴权：`hooks.server.ts` 的 `handle`、`locals`、Cookies/Session 的安全与多端一致性？  
24) 失效与缓存：SvelteKit 的数据失效（`invalidate`/`invalidateAll`）、预取（`prefetch`）与 HTTP 缓存头协作？  
25) Streaming 与渐进渲染：如何在 SvelteKit 中进行流式响应并与 Suspense-like 体验配合？  
26) 适配器与部署：`@sveltejs/adapter-node/static/vercel/cloudflare` 的取舍与边界（Edge/SSR/静态导出）？  
27) 资源与图像优化：静态资源路径、CDN、`<picture>`/`srcset`、滚动懒加载与预加载策略？  
28) SEO 与元信息：`<svelte:head>`、`+layout.ts` 中设置 `load` 产出的 meta、Open Graph/OGP 的最佳实践？  
29) 国际化（i18n）：在 Svelte/SvelteKit 中组织翻译字典、懒加载语言包、路由前缀与 SSR 同构注意点？  
30) 表单与验证：原生表单 + Actions 的服务端校验流程、zod/valibot 集成与错误回显模式？  
31) 大型列表/表格性能：虚拟滚动（第三方库）与 Svelte 模板的协作、`{#key}` 与不可变数据策略？  
32) 构建与打包：Vite 配置（别名、分包、动态 import）、生产 Source Map 与体积预算基线如何设定？  
33) 测试策略：`@testing-library/svelte`、Vitest、Playwright 的分层测试方法与常见陷阱？  
34) 安全：`{@html}` 的使用边界与 XSS 防护（DOMPurify/Trusted Types），敏感数据在 SSR/客户端的暴露控制？  
35) 迁移与前沿：从 Svelte 3/4 迁移到 Svelte 5（Runes）的步骤、兼容层/宏开关、渐进迁移与团队训练成本评估？

---
# 前端架构高频面试题清单（35 题）

1) **单体前端 vs 微前端**：在组织规模、发布独立性、跨团队协作下如何选型？各自的边界与代价是什么？  
2) **微前端集成方式**对比：Module Federation、iframe、Web Components、Runtime Composition 的权衡与落地难点？  
3) **BFF（Backend for Frontend）** 的职责边界：如何划分聚合/裁剪/鉴权？与 GraphQL 网关或 tRPC 怎么协作？  
4) **SSR/CSR/SSG/ISR/边缘渲染（Edge）** 的差异与选择策略？对 SEO、性能、成本、动态性各有什么影响？  
5) **Hydration 方案**演进：局部水合、渐进水合、岛屿架构（Islands）适用场景与实现要点？  
6) **路由架构**：多入口/嵌套路由/动态路由在大型应用中的模块划分与权限控制怎么设计？  
7) **设计系统 & 组件库**：设计令牌（Design Tokens）、主题化、多品牌/多产品线如何统一与下发？  
8) **跨应用状态管理**：UI 态 vs 服务器态的划分；在多包/多应用环境如何避免“全局状态泥沼”？  
9) **数据获取层**：REST vs GraphQL vs RPC 的取舍；缓存失效、乐观更新、冲突解决与一致性策略？  
10) **性能基线与预算**：首包体积、LCP/CLS/INP 目标如何制定？工程中如何建立“超标即报警”的机制？  
11) **代码分割策略**：路由级/组件级拆包、预加载（preload/prefetch/preconnect）怎么协同以避免“网络瀑布”？  
12) **资源与缓存**：CDN、HTTP 缓存（ETag/Cache-Control）、Service Worker 的版本治理与灰度发布？  
13) **边缘与多区域**部署：Edge Functions/边缘 SSR 的冷启动、状态一致性与回源策略如何设计？  
14) **可观测性**：RUM（真实用户监控）指标、日志/追踪/错误上报的链路与抽样策略怎么搭？  
15) **错误边界与降级**：前端容灾（降级、回退、占位）与后端熔断/限流如何联动？  
16) **发布策略**：灰度/金丝雀/蓝绿发布怎么在前端（静态资源 + 配置）与 BFF 协同落地？  
17) **CI/CD**：Monorepo 下的增量构建、缓存（Bazel/Turborepo/Nx）、制品管理与多环境配置如何打通？  
18) **依赖治理**：第三方脚本与 SDK 的沙箱与隔离（iframe/Worker）；供应链安全（SCA、签名、SRI）如何执行？  
19) **安全基线**：CSP/Trusted Types、XSS/CSRF、子资源完整性（SRI）与同源策略如何在架构层面落地？  
20) **鉴权与会话**：Token 模式（Cookie + SameSite vs Bearer）、多端单点登录、刷新/撤销策略如何统一？  
21) **权限模型**：RBAC/ABAC 在前端的表达与缓存；路由守卫、组件粒度授权与数据域隔离怎么实现？  
22) **配置与特性开关**：Feature Flag/远端配置中心在多版本并存、A/B 实验、回滚中的作用与风险？  
23) **大型表格/可视化**：虚拟化、数据流、增量渲染、跨页面共享缓存如何设计以保证性能与一致性？  
24) **多包/多应用协调**：Monorepo（PNPM workspaces/Nx）与 Polyrepo 的取舍；版本策略与 API 稳定性如何保证？  
25) **TypeScript 架构**：项目引用（Project References）、公共类型包、API 合同（OpenAPI/GraphQL SDL）如何驱动？  
26) **浏览器兼容与现代化**：多目标构建（modern/legacy）、差分下发、Polyfill 策略如何制定与验证？  
27) **国际化与可达性（i18n/a11y）**：动态语言包加载、RTL 布局、可达性基线与测试如何纳入流水线？  
28) **DOM/样式策略**：CSS Modules/Tailwind/CSS-in-JS/vanilla-extract 的规模化取舍与运行时开销控制？  
29) **前端缓存层**：Query 缓存（TQ/RTKQ/SWR）、持久化、跨标签页同步与过期/失效广播如何实现？  
30) **离线与 PWA**：缓存优先/网络优先策略、后台同步、冲突合并与“强一致 vs 最终一致”的权衡？  
31) **可插拔架构**：插件系统（扩展点/生命周期/权限沙箱）、远程模块热插拔如何保障安全与性能？  
32) **观测与实验平台**：埋点 SDK 规范、隐私合规（GDPR/CCPA）、A/B 归因与统计显著性在前端层的实现？  
33) **SEO 与可分享性**：SSR/边缘渲染、OG 标签、站点地图、国际化路由与规范 URL（canonical）如何统一？  
34) **ADT/ADR**：架构决策记录（Architecture Decision Record）在前端团队落地的流程与模板是什么？  
35) **演进与迁移**：框架/路由/构建工具升级（如 React/Vue/Svelte、Webpack→Vite）如何做“分阶段双栈”与技术债清理？

---
# ES / CSS / DOM 高频面试题清单（35 题）

## ECMAScript（ES）
1) 事件循环与任务队列：宏任务/微任务的执行顺序；浏览器与 Node 的差异？  
2) `Promise` 与 `async/await` 的语义：错误传播、并发控制（`all/allSettled/race/any`）与取消（`AbortController`）如何设计？  
3) 模块体系：ESM vs CommonJS 的差异、`tree-shaking` 生效条件、动态 `import()` 与跨环境打包要点？  
4) 原型与继承：原型链解析、`class` 的本质、`super`/`this` 绑定在继承中的行为？  
5) `this` 绑定五规则（默认/隐式/显式/`new`/箭头函数）及常见踩坑案例？  
6) 闭包与内存：闭包导致的泄漏风险、循环引用、`WeakMap/WeakSet/WeakRef` 适用场景？  
7) 深浅拷贝：`structuredClone` 与 `JSON.parse/stringify` 的差异；可转移对象与 `MessageChannel` 传递大数据？  
8) 函数式编程在前端中的应用：不可变、柯里化、管道/组合对可维护性与性能的影响？  
9) 元编程：`Proxy/Reflect/Object.defineProperty` 的对比与实现响应式/校验/拦截的思路？  
10) 迭代协议：`Iterator/Iterable`、`Generator`、异步迭代器的工作机制与场景？  
11) 类型与相等：`==` vs `===`、`Object.is`、`NaN`/`0`/`-0`、`typeof/instanceof` 与 `Symbol.toStringTag`？  
12) 性能优化：去抖/节流、事件委托、V8 优化陷阱（隐藏类、稀疏数组）、热路径与逃逸分析的影响？

## CSS
13) 层叠与优先级：特指度、`!important`、Cascade Layers（`@layer`）如何落地一套可维护的样式层级？  
14) 布局选型：Flex vs Grid 的核心差异与陷阱（`flex-basis` vs `width`，min-content/auto 轨道）？  
15) `subgrid` 的使用场景与相对嵌套网格/自动放置的优势与限制？  
16) 容器查询（`@container size/style`）与容器单位（`cqw/cqh/cqi/cqb`）如何配合媒体查询？  
17) `:has()` 选择器的能力边界与性能考量；用纯 CSS 实现交互逻辑的套路？  
18) 逻辑属性与国际化：`margin-inline/block`、`writing-mode`、RTL 布局的注意事项？  
19) 响应式图片与比例：`aspect-ratio`、`object-fit`、`srcset/sizes`、`AVIF/WebP` 的取舍？  
20) 动画性能：`transform/opacity` 优先、`will-change`、合成层与 CSS 动画 vs Web Animations API 的权衡？  
21) CSS 自定义属性与级联：主题切换、`color-mix()`、`lab/lch` 颜色空间的实践与兼容？  
22) 现代函数：`clamp()/min()/max()/fit-content()`、流式排版（fluid type）怎么写？  
23) 样式隔离：BEM、CSS Modules、Shadow DOM、`@scope`（如支持）的优缺点与团队规范？  
24) 字体与稳健性：`@font-face`、字体预加载、`font-display`，减少 FOUT/FOIT 与布局抖动（CLS）的策略？

## DOM / BOM
25) 事件模型：捕获/目标/冒泡阶段、`stopPropagation/stopImmediatePropagation`、`passive` 监听的实际影响？  
26) 自定义事件：`CustomEvent/dispatchEvent` 的正确用法；框架合成事件与原生事件的差异与协作？  
27) Shadow DOM & Web Components：开放/封闭 shadow、插槽、样式穿透（`::part/::slotted`）的边界？  
28) 渲染流水线：回流（reflow）/重绘（repaint）/合成（composite）；如何避免强制同步布局（layout thrashing）？  
29) 观察者 API：`MutationObserver` vs `IntersectionObserver` vs `ResizeObserver` 的适用差异与常见组合？  
30) 剪贴板/拖拽/文件 API：权限、粘贴图片/HTML、拖拽目录、DataTransfer 安全与沙箱策略？  
31) 历史与导航：`history.pushState`、`scrollRestoration`、`Navigation API`（如可用）的回退与过渡处理？  
32) 存储与同源：`localStorage/sessionStorage/IndexedDB` 的选择；COOP/COEP 与 `SharedArrayBuffer` 的关系？  
33) Web 安全：XSS/CSRF/Clickjacking 的前端防护；CSP/Trusted Types；富文本消毒（DOMPurify）的边界？  
34) Service Worker：缓存策略（CacheStorage）、离线优先/网络优先、版本更新与回滚的正确姿势？  
35) 性能与观测：Web Vitals（LCP/CLS/INP）采集、`PerformanceObserver`、Long Tasks 与交互延迟优化路线？

---
# 大前端算法高频面试题清单（35 题）

1) 两数之和（哈希表）  
2) 三数之和（排序 + 双指针，含去重）  
3) 无重复字符的最长子串（滑动窗口）  
4) 最小覆盖子串（滑动窗口 + 计数）  
5) 最长回文子串（中心扩展 / Manacher 对比）  
6) 最长回文子序列（DP）  
7) 编辑距离（Levenshtein，DP）  
8) 最长公共子序列 LCS（DP）  
9) 乘积数组除自身（前后缀积，O(1) 额外空间）  
10) 旋转数组中的搜索（二分与边界判定）  
11) 在有序数组中查找元素的第一个/最后一个位置（二分边界）  
12) 寻找峰值（二分）  
13) 合并区间（排序 + 扫描线）  
14) 回溯总题型：子集 / 全排列 / 组合总和（剪枝与去重）  
15) 接雨水（双指针 / 单调栈）  
16) 每日温度（单调栈）  
17) 柱状图中最大的矩形（单调栈）  
18) 滑动窗口最大值（单调队列）  
19) K 个一组翻转链表（链表分段处理）  
20) 环形链表检测 & 链表相交（快慢指针 / 相遇点）  
21) 二叉树遍历（层序 / 前中后序，递归与迭代）  
22) 二叉树最近公共祖先 LCA（递归 / 倍增思路对比）  
23) 二叉搜索树第 K 小元素（中序 + 计数 / 迭代）  
24) 二叉树最大路径和（递归返回“贡献值”）  
25) 课程表 / 能否完成所有课程（拓扑排序：Kahn / DFS）  
26) 岛屿问题：数量 / 最大面积 / 封闭岛（DFS/BFS/并查集）  
27) 最短路径：无权图 BFS / 加权图 Dijkstra（优先队列）  
28) Trie 前缀树：插入 / 查询 / 前缀匹配与通配场景  
29) LRU / LFU 缓存设计（哈希 + 双链表 / 频次堆）  
30) 前缀和与差分（含二维场景）  
31) 子数组和为 K 的个数（前缀和 + 哈希计数）  
32) 第 K 大元素（堆 / Quickselect）  
33) Top K 高频元素（小顶堆 / 桶排序）  
34) 位运算技巧：lowbit、异或求唯一数、快速幂取模  
35) 随机算法：Fisher–Yates 洗牌 / 蓄水池抽样 / 打乱数组

---
# React Native 高频面试题清单（35 题）

1) 新架构总览：**Fabric 渲染器 + TurboModules + JSI** 各自角色？与旧 Bridge 模型的本质差异与收益/代价？  
2) **Hermes** 引擎：启动时延、内存、字节码预编译（`hermesc`）、调试/Profiling 对比 JSC 的取舍？  
3) RN 渲染流水线：**JS Thread / UI Thread / Render(Compositor) Thread** 与 **Shadow Tree（Yoga）** 如何协作？批量更新与批处理机制？  
4) **Yoga 布局** 深入：`flex-basis` vs `width`、`min/max` 约束、文本基线、百分比尺寸与测量回调的坑点？  
5) 列表性能：`FlatList/SectionList/VirtualizedList/FlashList` 对比；`keyExtractor`、`getItemLayout`、`windowSize`、`removeClippedSubviews` 的调优策略？  
6) 动画体系对比：**Reanimated（worklets/UI thread）** vs `Animated` vs `LayoutAnimation`；何时用哪个？与 **Gesture Handler** 如何配合？  
7) 手势系统：`react-native-gesture-handler` 的竞争/并发机制（`simultaneous/requireFailure`）、滚动冲突与手势优先级设计？  
8) 导航选型：**React Navigation / React Native Navigation / Expo Router** 的架构差异；**Deep Link/Universal Link/App Links** 与状态恢复？  
9) 图片与资源：`Image` 缓存策略、预取（`prefetch`）、占位/渐进加载、`FastImage` 对比；Android Vector / iOS PDF 资产的差异？  
10) 网络层工程化：`fetch/axios` 选型、上传进度/断点续传、**SSL Pinning**、HTTP/2 与代理调试（Flipper/Charles）？  
11) 本地存储：`AsyncStorage` vs **MMKV** vs SQLite/Realm/WatermelonDB 的选型、加密与离线同步策略？  
12) 原生模块（**TurboModule**）开发：TypeScript 声明 + Codegen 流程；Promise/Callback/Sync 的边界；Swift/Kotlin 实战注意？  
13) 原生 UI 组件（**Fabric Component**）开发：Props/Events/Commands 的 codegen；Mounting/测量/布局生命周期？  
14) **JSI** 能力：HostObject/HostFunction、零拷贝与 `ArrayBuffer` 绑定、生命周期与线程安全问题？  
15) 构建与发布：iOS（CocoaPods/签名/Bitcode 状态/符号表）、Android（Gradle/KTS、ProGuard/R8、App Bundle）；多 ABI/多渠道？  
16) 多环境配置：iOS **Schemes/Configurations**、Android **productFlavors**、`react-native-config` 等环境注入方案？  
17) **OTA**：CodePush / Expo Updates 的限制（审核政策/二进制兼容/原生变更），灰度/回滚/元数据治理？  
18) 性能与监控：**Flipper** 插件体系、Hermes Profiling、JS Heap/内存泄漏定位、掉帧（UI/GPU）诊断与指标上报（Sentry/Firebase）？  
19) 内存与资源管理：图片大图/缓存、事件订阅泄漏、JS 闭包与原生强引用、后台任务造成的隐性驻留？  
20) 安全基线：反编译与混淆（R8/符号裁剪）、Bundle 保护、证书固定、Keychain/Keystore 的安全存储？  
21) 并发与后台：JS 单线程限制、**Headless JS**、`InteractionManager`、WorkManager/Background Fetch/定时器的正确使用与平台差异？  
22) 权限体系：iOS `Info.plist` 权限文案、Android 运行时权限与 **Scoped Storage**；前后台定位/相册/通知的合规弹窗？  
23) 推送通知：APNs/FCM 接入、Token 生命周期、前台/后台/杀进程三态处理、渠道（Android）与点击深链？  
24) 国际化（i18n）与本地化：语言切换、时区/历法/数字格式，RTL 布局与字体回退策略？  
25) 可达性（a11y）：`accessibilityRole/Label/Hint`、焦点管理、TalkBack/VoiceOver 差异、动态字体与对比度支持？  
26) WebView 与内嵌 H5：`react-native-webview` 的安全（JSBridge/混合内容/下载拦截）、文件上传、OAuth 流程与 Cookie 同步？  
27) 重型原生能力：地图/相机/音视频（摄像头帧处理、贴纸/滤镜）、权限与性能；GPU/IO 瓶颈与线程模型？  
28) Monorepo 与模块化：pnpm/yarn workspaces、`react-native-builder-bob`、私有包/原生链接、跨包 TS/别名配置？  
29) **Metro** 打包器：工作机制、缓存/Transformer、`inlineRequires`、RAM Bundles 与 Hermes 字节码产出？  
30) 测试体系：Jest（组件/Hook）、React Native Testing Library、原生模块 mock；E2E **Detox** 的稳定性与并发策略？  
31) TypeScript 与类型安全：项目模板/路径别名、**TurboModule/Component** 的类型生成、在桥接层确保类型一致性？  
32) 跨端与 Web：`react-native-web` 的能力/限制；平台特定文件（`.ios/.android/.native/.web`）与 SSR 协作？  
33) 错误处理：全局 JS 错误（`ErrorUtils`/`setNativeExceptionHandler`）、未捕获 Promise、崩溃页与生产保护？  
34) 应用架构：UI 态 vs 服务器态（Redux Toolkit/Zustand/Recoil/Jotai + TanStack Query）、模块边界、依赖注入与“可测试性优先”？  
35) 升级与迁移：从旧架构到新架构的步骤（启用 Fabric/TurboModules/Hermes）、兼容旧模块、常见坑与回退策略？

---
# Web3 高频面试题清单（35 题）

1) 账户抽象（AA）：EOA vs 合约账户的根本差异；**EIP-4337** 与“原生 AA”（如 3074/7702 路线）的对比与迁移路径？  
2) 以太坊费模型：**EIP-1559** 的 base fee / priority fee 如何协作？钱包如何做 Gas 估算与小费策略？  
3) **EIP-4844（Proto-Danksharding）**：Blob 数据、Blob Gas、对 L2 成本与数据可用性的影响？  
4) Rollup 体系：**Optimistic vs ZK Rollup** 的证明/挑战窗口/退出时间/安全假设各是什么？  
5) 数据可用性（DA）：以太坊 DA vs 外部 DA（Celestia/EigenDA 等），Rollup 在 DA 上的取舍与风险？  
6) MEV 基础：套利/清算/夹击三大类；从 mempool 到 PBS（Proposer-Builder Separation）的演进与影响？  
7) 订单流与抗 MEV：私有内存池、保护 RPC、批量拍卖、SUAVE/Intent 系统的设计思路与权衡？  
8) 共识与最终性：PoS slot/epoch、检查点与 finalized 的语义；再组织（reorg）与活性/安全性的取舍？  
9) 分布式验证技术（**DVT**）：阈值签名/SSV 的工作原理，如何降低单点与惩罚（slashing）风险？  
10) L2 Sequencer：中心化 Sequencer 的审查与停机风险；去中心化/共享 Sequencer、Based Rollup 的利弊？  
11) 跨链桥安全：多签桥 vs 轻客户端桥 vs ZK/Optimistic 消息桥的信任模型比较与常见攻击面？  
12) 代币标准：**ERC-20/721/1155** 的核心差异；**ERC-2612（Permit）**、**ERC-4626（Vault）** 的典型用法？  
13) 升级方案：Transparent/UUPS 代理的架构差异；存储布局与“Storage Gap”如何避免踩坑？  
14) 常见漏洞：重入、整数溢出、授权滥用、签名重放（`eth_sign`/`personal_sign`）与预防策略？  
15) **EIP-712** Typed Data 签名：域分隔、链 ID、防钓鱼交互与钱包提示设计要点？  
16) SELFDESTRUCT 变更：上海后 **EIP-6780** 对合约销毁/存储清理语义的影响与兼容策略？  
17) 费与存储优化：`calldata` vs `memory`、存储打包、事件日志成本、EIP-3529 退款规则变化的影响？  
18) DeFi 基础：恒定乘积 AMM、集中流动性（Uni v3）与 TWAP 价格的原理与边界？  
19) 借贷与清算：抵押率、清算折扣、拍卖机制（英式/荷兰式）与价格源的抗操纵设计？  
20) 预言机：Chainlink/Redstone/自建喂价的对比；TWAP/Medianizer 与闪电贷操纵防护？  
21) 稳定币设计：法币抵押 vs 加密抵押 vs 算法稳定；铸赎机制、脱锚风险与恢复手段？  
22) 质押与再质押：LST（stETH 等）与 LRT/EigenLayer 的收益与风险传染路径？  
23) NFT 与元数据：ERC-721/1155、**ERC-2981** 版税约定；on-chain vs off-chain 元数据与持久化？  
24) 钱包与密钥：多签（Safe）、**MPC**、AA 钱包（社交恢复、Session Key/Paymaster）各自的取舍？  
25) 隐私与合规：零知识（SNARK/STARK）基础、选择器/Nullifier；混币器与制裁合规的工程边界？  
26) 开发与测试栈：Hardhat/Foundry 对比；主网 fork、模糊测试（fuzz）、属性测试与覆盖率度量？  
27) 静态/形式化分析：Slither/Mythril/Echidna/Certora 的适用场景与“不可变式（invariant）”设计？  
28) 事件与索引：事件 schema 设计、The Graph/Substreams 索引管线、重组与幂等性处理？  
29) 多链开发与运维：Chain ID/Nonce 差异、RPC 可靠性、重试/回滚、部署与权限（Owner/Timelock）治理？  
30) 治理与金库安全：投票模型（Snapshot/链上）、代理/时间锁、防突袭（rage quit/guardian）机制？  
31) 前端与签名交互：EIP-1193 Provider、`eth_requestAccounts`、权限会话/断开、硬件钱包与 WebAuthn 集成？  
32) Intent-centric 设计：用户意图 → 求解器（Solver）竞价 → 清算结算的流程与与传统 swap 请求的差异？  
33) L2 费用模型：L1 安全费用 + L2 执行费用的组成；**EIP-4337** Paymaster/AA Gas 支付体验？  
34) 时间与随机性：`block.timestamp` 偏移风险、VRF/可验证随机源、跨链/跨批次的一致性问题？  
35) 安全事件复盘：著名 DeFi/NFT/桥攻击归因分类（权限误配、预言机操纵、重入、初始化缺陷），如何建立预防清单与应急预案？

---
# TypeScript 高频面试题（35 题）

1) 结构类型系统与“名义类型”的差异是什么？如何用“品牌类型（branded type）”模拟名义性？  
2) 类型收窄（Control Flow Analysis）如何工作？`in / typeof / instanceof / 判空` 与可辨识联合的配合？  
3) `any`、`unknown`、`never` 分别代表什么语义？各自的风险与使用边界？  
4) 泛型参数的约束/默认值如何设计？分布式条件类型与 `infer` 推断有哪些典型用法？  
5) 交叉类型与联合类型在可空性/可选属性上如何交互？`Partial<T>` 与可选属性有何差别？  
6) 索引类型体系：`keyof`、索引访问类型、映射类型与键重映射（`as`）如何协作？  
7) 模板字面量类型如何与字符串字面量/枚举配合实现“路径/路由”的类型安全？  
8) 常用工具类型 `Pick/Omit/Record/ReturnType/Parameters/InstanceType/NonNullable/Required/Readonly` 的原理？  
9) 函数类型的协变/逆变：参数双向协变（bivariance）与 `strictFunctionTypes` 的影响？  
10) `this` 类型与 `noImplicitThis` 有什么关系？如何在函数签名中显式声明 `this` 参数？  
11) 函数重载（overload）与联合/泛型的取舍？实现签名与重载签名规则有哪些坑？  
12) `interface` 与 `type` 的差异与取舍？何时需用声明合并、递归类型或计算属性？  
13) 模块解析与别名：`baseUrl/paths`、`types/typeRoots`、`moduleResolution` 的行为差异？  
14) 声明合并（接口/命名空间/模块增强）适用场景与风险点有哪些？  
15) `satisfies` 与 `as const` 各自语义是什么？如何“既校验形状又保留宽类型推断”？  
16) 装饰器现状（实验/提案）与 `emitDecoratorMetadata` 的成本/收益如何权衡？  
17) `readonly` 在属性、数组、元组中的行为如何？不可变模式如何建模？  
18) 变长元组与参数列表类型：`...`、`Concat`、`Tail/Head`、`Tuple → Union` 的实现思路？  
19) 分布式条件类型的“自动分配”陷阱如何避免（`[T] extends [U]` 技巧）？  
20) DOM 类型与 `lib` 选择如何影响全局类型？多目标（Node/浏览器）项目怎么配置？  
21) `declare` / `.d.ts`：如何为 JS/第三方库编写/补充类型声明？  
22) `enum` vs `const enum` vs 字面量联合的编译输出差异与可维护性取舍？  
23) 类/接口在混入模式与多实现中的写法？抽象类与装饰器如何结合？  
24) 用户自定义类型守卫（Type Guard）如何编写与测试？局限点在哪？  
25) Error 的类型安全：`unknown` 错误如何处理？`instanceof` 的跨 realm 问题怎么解？  
26) `tsconfig` 严格模式族（`strictNullChecks` 等）对代码质量与迁移成本的影响？  
27) 项目引用（Project References）如何提升 Monorepo 的构建/发布效率？  
28) 声明文件生成（`.d.ts`）与 `declarationMap` 在库发布中的最佳实践？  
29) 与 React/Vue 集成：JSX/Props/Emit/Slot 如何进行高级类型建模？  
30) 类型 ↔ 运行时校验：zod/valibot/io-ts 与 JSON Schema 互转的思路与边界？  
31) 键重映射 + 模板字面量：如何从 `getX` 自动生成 `setX`/事件名等 API？  
32) 模式匹配类型（`infer` + 递归）：如何提取路由参数、Promise 内部类型、事件负载？  
33) 类型计算的性能瓶颈如何诊断与治理（类型层“爆炸”问题）？  
34) `export type` 与类型擦除（erasure）：如何划清“类型系统 vs 运行时”的边界？  
35) 面对 JS 新提案（装饰器、记录/元组、TypedArray 等），TS 的类型策略与迁移注意点？

---

# 2D + 3D 图形渲染高频面试题（共 35 题）

1) Canvas 2D / SVG / WebGL / WebGPU 各自的编程模型与适用边界如何选型？  
2) 坐标系与像素密度（DPR）：缩放、Retina 与逻辑像素/物理像素如何影响绘制质量？  
3) 2D/3D 变换矩阵：平移/旋转/缩放与齐次坐标；矩阵乘法顺序如何影响结果？  
4) 栅格化与抗锯齿：MSAA/FXAA/TAA 的差异及适用场景？  
5) 渲染管线：顶点/片元着色器的职责是什么？WebGPU 的可编程阶段有何新能力？  
6) 光照模型：Lambert/Phong/Blinn-Phong 与 PBR（基于物理的渲染）各自的优劣？  
7) 阴影实现：Shadow Mapping/PCF/PCSS/CSM 的原理与性能权衡？  
8) 前向渲染 vs 延迟渲染 vs Forward+ / Clustered 渲染的取舍？  
9) 颜色管理：sRGB vs Linear、HDR 与 Tone Mapping；Gamma 校正为何重要？  
10) 深度/模板缓冲如何工作？Z-fighting 的常见原因与缓解办法？  
11) 混合（Blending）与预乘 Alpha 的数学与实际工程注意事项？  
12) 纹理采样：mipmap、min/mag/anisotropic 过滤与纹理坐标越界处理？  
13) 材质系统与 BRDF：金属/非金属、粗糙度、环境反射（IBL）如何实现？  
14) 实例化与批渲染：UBO/SSBO/顶点压缩如何降低 CPU→GPU 往返成本？  
15) 粒子系统：发射器、受力/加速度、GPU 粒子的实现与排序问题？  
16) 后处理：Bloom、Vignette、SSAO/SSR、Motion Blur 的核心思路与顺序安排？  
17) 相机模型：透视/正交、视椎体裁剪、Jitter 抖动与稳定渲染的关系？  
18) 贴图技术：法线/视差/位移贴图、环境/立方体贴图、三平面投影的适用场景？  
19) 资源与格式：glTF/Draco 压缩、KTX2/BasisU 纹理压缩的管线如何搭建？  
20) 骨骼蒙皮与 Morph 目标：动画混合/重定向如何在引擎中落地？  
21) 物理与碰撞：Ammo/Cannon/Oimo/Box2D 的差异；刚体/软体/关节如何抽象？  
22) 拾取（Picking）：颜色编码/射线投射（Raycasting）/BVH 加速的优缺点？  
23) 2D 路径与布尔运算：填充规则（even-odd）、线段连接与自交处理？  
24) 字体渲染：SDF/MSDF 字体原理；国际化排版（shaping/换行）如何处理？  
25) 图像处理：卷积核（边缘检测/模糊/锐化）、WebGPU 计算/多通写法？  
26) 瓦片化与大图分片渲染：地图/海量点渲染的层级与裁剪策略？  
27) 离屏渲染：`OffscreenCanvas`、FBO、多目标渲染（MRT）在性能上的价值？  
28) 时间步长与帧同步：固定/可变时间步、VSync、插值/外推的利弊？  
29) 场景管理：Octree/Quadtree/BVH/LOD 如何降低渲染与碰撞成本？  
30) 引擎选型：Three.js/Babylon.js/PlayCanvas（或自研）在架构/生态/可扩展性上的差异？  
31) WebGL1 vs WebGL2 vs WebGPU：特性差异、迁移路径与兼容策略？  
32) 移动端限制：带宽/填充率/过度绘制管理；Tiling GPU 的特殊优化？  
33) 透明与渲染顺序：排序、双通道技术、深度预通道（Z-prepass）解决哪些问题？  
34) 资源加载与生命周期：并发、解压、缓存、引用计数与内存上限治理？  
35) 调试与分析：Spector.js、WebGPU 捕获、GPU 时间戳与火焰图如何定位瓶颈？

---
# Electron 高频面试题清单（35 题）

1) 进程架构：**Main / Renderer / Preload** 各自职责？与旧 **Bridge** 通信模型的关系是什么？  
2) 安全基线清单：`contextIsolation`、`sandbox`、`nodeIntegration`、CSP 等应该如何组合与取舍？  
3) **IPC 安全**：`ipcRenderer.invoke/handle` 与 `send/on` 的差异？如何做参数校验、最小暴露与防注入？  
4) Preload 脚本的职责边界：`contextBridge.exposeInMainWorld` 暴露 API 的最佳实践与生命周期管理？  
5) 渲染进程沙箱化：开启 `sandbox: true` 后常见能力缺失如何补齐？与 `nodeIntegration: false` 的区别？  
6) 导航与外链安全：`will-navigate`、`setWindowOpenHandler`、`shell.openExternal` 的使用规范？如何防止任意跳转/钓鱼？  
7) **BrowserWindow / BrowserView / WebContents** 的差异与选型？复杂多窗体应用如何组织层级与通信？  
8) 自动更新体系：`electron-updater` vs Squirrel vs MAS 各自适用场景？**差分更新**与多通道灰度如何设计？  
9) 代码签名与发布：macOS **notarization + hardened runtime**、Windows 证书与签名链配置要点？  
10) 打包工具链：`electron-builder`、`electron-forge`、`electron-packager` 的能力对比与常见配置陷阱？  
11) 资源与 ASAR：开启 asar 的优缺点？如何对敏感资源加固并处理运行时解压？  
12) 性能治理：首窗白屏、热路径定位、`backgroundThrottling`、释放 WebContents/内存泄漏排查的套路？  
13) GPU 与渲染：何时需要 `app.disableHardwareAcceleration()`？过度绘制/合成层问题如何诊断？  
14) 桌面捕获：`desktopCapturer`/`getUserMedia` 获取屏幕/窗口时的权限与跨平台差异？  
15) 系统托盘/菜单/任务栏：Tray、Dock、Jump List、Global Shortcut 的平台差异与冲突处理？  
16) 通知体系：`new Notification` 与各平台本地通知中心差异？自定义按钮/交互的可行性与限制？  
17) 电源与系统事件：`powerSaveBlocker`、`powerMonitor` 的使用场景与副作用？  
18) 单实例/深链：`app.requestSingleInstanceLock` 与 `setAsDefaultProtocolClient` 如何实现 Deep Link 唤起？  
19) 自定义协议与文件关联：`protocol.registerSchemesAsPrivileged` 的风险与标准/特权协议的权衡？  
20) 文件系统与权限：`dialog.showOpenDialog`、`app.getPath`、沙箱/权限弹窗与目录穿越防护？  
21) 数据持久化：SQLite/LevelDB/IndexedDB 的取舍？敏感数据如何用 **Keychain/DPAPI/keytar** 加密存储？  
22) 崩溃与日志：Crashpad 集成、minidump 上报、符号表（dSYM/PDB）管理与隐私脱敏？  
23) 调试与剖析：DevTools、性能面板、`--inspect`、Source Map、`contentTracing` 的正确打开方式？  
24) 原生模块与 JSI/Node-API：如何编译/预构建跨平台二进制？Electron 版本与 Node ABI 的兼容策略？  
25) 窗体形态：无边框/透明/毛玻璃（vibrancy）/置顶窗口的实现细节与输入命中区域（`-webkit-app-region: drag`）？  
26) 打印与导出：`webContents.print`/`printToPDF` 的平台差异、分页/页眉页脚/背景图处理？  
27) 剪贴板与拖放：`clipboard`、`nativeImage`、Drag & Drop 跨进程/跨应用的格式适配？  
28) 可访问性（a11y）：屏幕阅读器兼容、语义树、快捷键冲突与系统高对比度模式支持？  
29) 会话与网络：`session` 分区、Cookie/Cache 管理、代理设置、`webRequest` 拦截与证书校验（`setCertificateVerifyProc`）？  
30) 生命周期与平台差异：`ready/activate/window-all-closed` 在 macOS/Windows 的行为差异与菜单焦点坑点？  
31) CI/CD：多平台交叉构建、Apple Silicon/Universal 二进制、签名/公证自动化与密钥保护？  
32) 测试策略：Spectron 退场后的替代方案（Playwright/Chromedriver 驱动）与端到端测试稳定性建设？  
33) Web 安全策略：`allowRunningInsecureContent`、`webSecurity`、CSP、`trustedTypes` 等在桌面端的适配与边界？  
34) 权限与合规：摄像头/麦克风/屏幕录制权限提示、MAS 沙箱 Entitlements、隐私条款与崩溃/埋点合规？  
35) 迁移与升级：Electron 大版本升级的断点（Chrome/Node/ABI 变化）、Remote 模块移除的替代方案与回滚策略？

---

# 实战领域面试题（35 题）

1) 电商商品详情/列表架构如何设计？SKU 选择、价格展示与缓存一致性如何保证？  
2) 收银台/支付集成如何实现多渠道（卡/钱包/转账）与风控、对账与幂等？  
3) 购物车/优惠券/满减引擎如何抽象规则并保持可测试与可回放？  
4) 搜索与推荐在前端如何做结果合并、打点回传与降级回退？  
5) 动态表单引擎：基于 Schema 的渲染/校验/联动与性能优化如何落地？  
6) 富文本编辑器：基于 ProseMirror/Slate 的模型、插件化与粘贴清洗安全？  
7) 协同编辑：OT vs CRDT 的前端实现、冲突解决与离线合并？  
8) IM/聊天室：WS/SSE 的断线重连、已读回执、消息去重与本地队列？  
9) WebRTC 连麦/直播：编解码、带宽自适应（ABR）、TURN 中继与弱网策略？  
10) 音视频播放器：MSE/EME、HLS/DASH、自适应码率与首帧优化？  
11) 地图/GIS：瓦片/矢量瓦片渲染、路径规划与海量点聚合（Cluster）？  
12) BI 大屏与实时可视化：流式数据、增量渲染与权限隔离（租户/数据域）？  
13) 拖拽式低代码：DSL/Runtime/出码策略，如何保证扩展点与沙箱安全？  
14) 工作流/审批流：可视化编排、长事务补偿（SAGA）、幂等与回放？  
15) 日历/排班系统：时区/夏令时、重复事件（RRULE）与提醒的正确建模？  
16) 国际化：多语言/多货币/单位换算，翻译工作流与运行期字典更新？  
17) H5/小程序/公众号一体化：路由、能力差异与组件层抽象如何统一？  
18) 移动端适配：高刷屏/折叠屏/刘海安全区，触控手势与滚动性能优化？  
19) 桌面应用：Electron/Tauri 打包、原生能力桥接与自动更新/回滚？  
20) 浏览器扩展：内容脚本、权限、跨域通信与安全沙箱（CSP/Manifest V3）？  
21) SaaS 多租户：主题/品牌切换、配置隔离、性能与成本分摊怎么做？  
22) 权限系统：RBAC/ABAC/数据级权限在路由/组件/接口层如何协同？  
23) 埋点与 RUM：统一 SDK、隐私最小化、采样/归因与实验平台联动？  
24) PWA：离线缓存策略、消息推送、版本迁移与“离线-在线”一致性？  
25) 文件系统：大文件分片/断点续传/秒传、哈希校验与并发合并？  
26) 打印/导出：PDF/Excel 的模板化、分页/国际化排版与客户端字体处理？  
27) 地理围栏/定位：权限管理、后台定位与隐私合规（GDPR/CCPA）？  
28) 电子签名：签署流程、时间戳服务（TSA）、哈希链与防篡改？  
29) 安全与风控：验证码/设备指纹/反自动化、黑产对抗与监控告警？  
30) 可访问性（a11y）：语义/焦点/对比度/动态字体在复杂组件中的落地？  
31) SEO & SSR：多区域/多语言的可索引性、边缘渲染与缓存策略如何协作？  
32) 微前端治理：跨团队路由/样式/依赖冲突、共享运行时与灰度发布？  
33) 大型表格/网格编辑器：虚拟化、公式/依赖图、撤销/快照与协同？  
34) AI 集成：LLM 助手、向量检索、多模态上传与“前端侧隐私”策略？  
35) 数据隐私与合规：脱敏/加解密/密钥管理、审计追踪与用户数据导出/删除流程？

---

## 使用建议

- 自测：按题目从“**原理 → 实战 → 边界/坑**”结构写出要点与代码片段。  
- 面试官视角：为每题准备**追问树**（Why/How/Trade-off），并能结合真实项目举例。
