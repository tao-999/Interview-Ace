# 面试题清单 · Flutter & React 🚀

> 各 **35 道**、聚焦现代实践的高频面试题列表。

---

## 📱 Flutter 高频面试题（35 题）

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

## ⚛️ React 高频面试题（35 题）

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

---

## 📎 使用建议

- 自测：按题目从“**原理 → 实战 → 边界/坑**”结构写出要点与代码片段。  
- 面试官视角：为每题准备**追问树**（Why/How/Trade-off），并能结合真实项目举例。
