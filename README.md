# 面试题清单

> 聚焦现代实践的高频面试题列表。

---

# 前端 · 计算机网络高频面试题（50 题）

1) **HTTP/1.1 vs HTTP/2 vs HTTP/3（QUIC）**：多路复用、头阻塞（HoLB）、优先级/流量控制与握手差异？  
2) **TCP/UDP/QUIC**：连接建立（3/4 次握手、0-RTT/1-RTT）、丢包/拥塞控制、重传与对前端体验的影响？  
3) **TLS 基础**：证书链、SNI、ALPN、HSTS；浏览器如何校验证书与中间人攻击的防护？  
4) **DNS 与延迟**：A/AAAA/CNAME/NS/TXT 记录、TTL、递归解析；`dns-prefetch`/`preconnect` 的作用与时机？  
5) **缓存体系**：`Cache-Control`（`max-age/s-maxage/public/private/immutable/stale-while-revalidate/stale-if-error`）、`ETag` vs `Last-Modified`、强/协商缓存与 304 流程？  
6) **服务端与 CDN 缓存联动**：`Vary`/`Surrogate-Control`、边缘缓存键（含查询/头部）、多区域回源与回源保护？  
7) **内容协商**：`Accept-*`（`Accept-Encoding/Language/Charset`）、`Content-Encoding`（gzip/br/zstd）、`Content-Type` 与 `Vary` 的正确搭配？  
8) **状态码语义**：2xx/3xx/4xx/5xx 常见与冷门（如 103/204/206/308/412/421/429/451）；重定向链与缓存交互？  
9) **请求幂等与重试**：GET/HEAD/PUT/DELETE 的理论幂等与实践陷阱；Idempotency-Key 的设计？  
10) **分块与流式**：`Transfer-Encoding: chunked`、Fetch/Streams API（`ReadableStream`）的读取与背压（backpressure）？  
11) **大文件与断点续传**：Range 请求（206）、多段并发下载、ETag/If-Range 保证一致性？  
12) **同源策略与 CORS**：预检（OPTIONS）触发条件、`Access-Control-*` 头、凭证模式、`opaque` 响应与缓存？  
13) **跨源隔离**：`COOP/COEP/CORP` 三兄弟、`SharedArrayBuffer` 的启用条件与安全权衡？  
14) **安全基线**：CSP（`default-src`、nonce/hash）、Referrer-Policy、X-Frame-Options/`frame-ancestors`、Mixed Content 与升级策略？  
15) **Cookie 策略**：`SameSite=Lax/Strict/None`、`Secure/HttpOnly`、作用域与优先级、会话粘滞与子域共享？  
16) **认证与会话**：Bearer/JWT vs Cookie、刷新令牌、前端如何安全存储与跨域携带？  
17) **网络优先级**：HTTP/2 旧优先级树 vs HTTP Priority（`Priority`/`priority`）、`<link rel=preload>` 与优先级提示（Priority Hints）区别？  
18) **预加载族**：`preload/prefetch/prerender/preconnect` 的语义、时机与踩坑；103 Early Hints 与 `<link rel=preload>` 协作？  
19) **图片/字体传输策略**：`srcset/sizes`、`Content-DPR`/Client Hints、`font-display`、字体子集化与缓存键设计？  
20) **上传与关闭页**：`fetch({ keepalive: true })` vs `navigator.sendBeacon` 的限制与可靠性取舍？  
21) **取消与超时**：`AbortController`、超时/重试退避（exponential backoff）、幂等配合如何设计？  
22) **实时通信选型**：WebSocket vs SSE vs WebRTC vs WebTransport 的能力、拥塞/防火墙友好度与落地场景？  
23) **HTTP/3 in practice**：QUIC 流/连接 ID、迁移与丢包对比 H2 的体验差异；中间盒与降级路径？  
24) **代理与网络栈**：正向/反向代理、负载均衡（L7/L4）、超时/重试/粘性会话对前端的表现影响？  
25) **边缘计算**：Edge Functions/边缘 SSR 的冷启动、地理亲和、缓存失效与回源策略？  
26) **SWR/ISR 与再验证**：静态+增量（ISR）/`stale-while-revalidate` 的前后端协作与一致性风险？  
27) **请求合并与去重**：浏览器层面的连接复用、H2 多路复用；应用层去重（同 key 请求合并、竞态与最后写赢）？  
28) **GraphQL over HTTP**：缓存难点（POST、GraphQL-GET、Persisted Queries）、批量（`@defer/@stream`）与网络层治理？  
29) **Range/分片上传**：大文件切片、并发/顺序策略、校验（MD5/CRC/ETag）与秒传设计？  
30) **网络错误语义**：DNS 失败、TLS 失败、CORS 拒绝、Service Worker 拦截、`TypeError: Failed to fetch` 的精准归因？  
31) **Service Worker 网络策略**：Cache First/Network First/Stale-While-Revalidate、离线兜底与版本迁移（跳票/清缓存）？  
32) **优雅降级**：弱网/高延迟/丢包环境下的超时、断点续传、占位骨架与渐进式渲染策略？  
33) **请求头/响应头精讲**：`Origin/Referer/Accept/Authorization/If-None-Match/ETag/Date/Age/Expires/Retry-After` 的组合拳？  
34) **重定向与安全**：开放重定向检测、SameSite 与跨站跳转、跨域重定向对凭证/缓存的影响？  
35) **监控与采样**：Resource Timing/Navigation Timing/Long Tasks、`PerformanceObserver`、错误上报与采样策略（流量/地域/设备）？
36) **WebSocket 握手与升级**：HTTP `Upgrade`/`Connection`、`Sec-WebSocket-Key/Accept` 流程；子协议（`Sec-WebSocket-Protocol`）与二进制帧/文本帧的差异？  
37) **WebSocket 可靠性与性能**：心跳/断线重连/指数退避与抖动；背压与大消息分片；`permessage-deflate` 压缩的收益与风险？  
38) **SSE vs HTTP Streaming**：浏览器支持、自动重连、缓存/代理兼容性与典型落地（行情/日志/增量）差异？  
39) **WebRTC DataChannel vs WebSocket**：可靠/不可靠传输、时延/拥塞控制、NAT 穿透能力；何时选用哪一个？  
40) **STUN/TURN/ICE**：候选类型（host/srflx/relay）、打洞流程、优先级/选路；TURN 中继的成本与企业内网限制？  
41) **HTTP/3（QUIC）细节**：0-RTT 重放风险、连接 ID 与连接迁移、路径验证、`Alt-Svc` 协商与中间盒兼容性？  
42) **拥塞控制与体验**：CUBIC vs BBR 的差异；如何通过 RUM（LCP/INP/首帧/卡顿）侧信号定位“网络 vs 渲染”瓶颈？  
43) **HoLB 与多路复用**：H2 的“流级 HoLB”与 H3 的“无传输层 HoLB”；弱网下资源“合并 vs 拆分”的策略选择？  
44) **DNS 现代化**：DoH/DoT、EDNS、DNSSEC 的基本概念与前端可感知影响；IPv6/IPv4 **Happy Eyeballs** 对首包时延的作用？  
45) **MTU/PMTUD 与分片**：路径 MTU 发现、DF 位、黑洞路由的表现；前端资源（大图/视频分块）如何规避分片代价？  
46) **代理与隧道**：HTTP `CONNECT`、反向代理/负载均衡对 WebSocket/SSE 的支持差异；WAF/CDN 场景下的头部透传与超时策略？  
47) **OAuth 2.1 / OIDC 跨域登录**：授权码 + PKCE 流程、`state` 防 CSRF、回跳域安全；前端 token 刷新/存储与跨域限制？  
48) **优先级信号**：`fetchpriority`、`<link rel="preload">`、HTTP/2/3 Priority 的区别与配合；如何避免抢占关键渲染资源？  
49) **103 Early Hints + Preload**：服务器早发送链接提示的价值与坑点；如何与缓存/优先级/预连接协作避免重复下载？  
50) **Reporting API / NEL**：网络错误与策略违规的上报机制、采样与隐私考量；如何与前端埋点/告警联动构建闭环？

---

## Flutter 高频面试题（60 题）

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
36) **Impeller 深入**：与 Skia 的差异、着色器编译/缓存策略、SkSL 预热与消除 shader jank 的实践？  
37) **Flutter Web 渲染后端**：CanvasKit vs HTML（与 Wasm 引擎）的取舍；性能/包体/可用性权衡与典型配置？  
38) **文本与排版系统**：`Paragraph`/`TextPainter` 工作流、断行与双向文本（Bidi）/合字（ligature）/emoji 的处理边界？  
39) **图片解码与内存治理**：`ImageCache` 策略、`painting.imageCache` 调优、`ui.instantiateImageCodec` 异步解码与大图 OOM 处理？  
40) **自绘性能**：`CustomPainter`、`saveLayer`、`PictureRecorder` 对合成层与 raster cache 的影响；如何避免过度绘制？  
41) **运行时着色器**：`FragmentProgram`/`ShaderMask`/`BackdropFilter` 的能力边界与跨平台一致性问题？  
42) **Isolate 并行模型**：`Isolate.run`/`compute` 的适用场景、消息传递开销、`TransferableTypedData` 零拷贝与生命周期管理？  
43) **FFI（dart:ffi）与原生性能**：结构体布局/对齐、回调、内存所有权；相比 `MethodChannel` 的延迟/吞吐差异？  
44) **PlatformView 进阶**：Android `Hybrid Composition` vs `Virtual Display`/`Texture` 的渲染/输入差异；iOS UIView/UIViewController 容器化策略？  
45) **手势系统深潜**：`GestureArena` 冲突裁决、自定义 `GestureRecognizer`、滚动与手势的竞争/要求失败（require-fail）场景？  
46) **自定义 RenderObject**：`RenderBox`/`RenderSliver` 的布局协议、`hitTestChildren`、约束系统与可视区域裁剪？  
47) **桌面平台工程化**：Windows/macOS/Linux 的窗口/菜单/托盘、多显示器 DPI、系统事件差异与插件适配？  
48) **Web 与 JS 互操作**：`package:js`/`dart:js_interop` 的使用、CSP 限制、浏览器 API（如剪贴板/全屏）接入边界？  
49) **稳定性与上报**：`FlutterError`/`PlatformDispatcher.onError`/`runZonedGuarded` 与原生崩溃（Crashpad/dSYM/PDB）串联、日志切片与脱敏？  
50) **构建与混淆**：`--split-debug-info`、`--obfuscate`、符号文件管理与回溯；最小化包体与调试可追溯性的平衡？  
51) **动画进阶**：物理动画（`Spring`/`Simulation`）、多控制器协同、`Ticker` 泄漏排查与 `TickerMode` 的节流作用？  
52) **路由与深链**：Navigator 2.0 + `go_router` 的 URL 映射/守卫/恢复；多实例栈与外部唤起（app links/URL schemes）？  
53) **进程死亡与状态恢复**：Android 后台被杀重建、iOS 内存压力退出后的恢复策略；与持久层/缓存的一致性设计？  
54) **插件联邦化与发布**：多平台分仓/接口封装、版本矩阵 CI、breaking change 治理与迁移文档化？  
55) **后台任务与平台协作**：iOS `BackgroundTasks`/`PushKit`、Android Foreground Service/WorkManager 与 Flutter 侧调度的坑点？  
56) **音视频/设备能力**：相机/麦克风/蓝牙权限模型、帧处理与传输带宽；跨平台编码能力与降级策略？  
57) **安全基线**：反调试/Root/Jailbreak 检测、证书锁定（pinning）、WebView JSBridge 安全与屏幕录制防护局限？  
58) **高级国际化**：动态切换语言/区域、ICU 复数/性别规则、本地化数据热更新与时区统一？  
59) **DevTools & 性能诊断**：Timeline/CPU/Memory Profiler、`flutter trace`、`PerformanceOverlay` 的使用路线与读数解读？  
60) **分发与合规**：iOS TestFlight/审核要点、Android App Bundle/Play Integrity、桌面应用签名/公证/沙箱与自动更新？
---

##  React 高频面试题（50 题）

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

# Vue 3 高频面试题清单（50 题）

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
36) **Reactive Props Destructure（3.5）**：`defineProps()` 的**解构仍保持响应式**的机制与边界？与 `toRef/toRefs` 的取舍与迁移注意？  
37) **`useTemplateRef()`（3.5）**：与 `ref="xxx"`/`template ref` 的差异、动态列表场景、类型推断与失效时机？  
38) **`onWatcherCleanup()`（3.5）**：与 `watch` 清理回调的关系；取消网络请求/事件监听/动画的最佳范式？  
39) **`<Teleport defer>`（3.5）**：目标容器延迟出现时的挂载时序、可达性（焦点/阅读顺序）与滚动锁协作？  
40) **`useId()`（3.5）与 SSR 一致性**：ID 生成在 CSR/SSR 的一致性保障、对表单/无障碍/可复用组件的价值？  
41) **受控差异抑制（3.5）**：`data-allow-mismatch` 的适用场景与风险；何时应该改为纯客户端逻辑？  
42) **`defineModel()`（3.4）进阶**：多 `v-model`、修饰符类型化、父子组件双向绑定与受控/非受控切换策略？  
43) **泛型组件（3.3+）**：`<script setup lang="ts">` 的泛型声明方式、`defineProps`/`withDefaults` 与 Volar 推断的坑位？  
44) **`defineSlots()` 类型化**：作用域插槽的参数约束、联合/可选槽设计、编译器对错误用法的提示边界？  
45) **渲染性能微优化**：`v-memo`/`v-once` 与 `shallowRef/markRaw` 的组合策略；何时值得用、何时过度优化？  
46) **Effect 调度与刷新策略**：`watch` 的 `flush: 'pre'|'post'|'sync'` 取舍，如何避免布局抖动与多次无效计算？  
47) **`effectScope` / `onScopeDispose`**：Composable 的资源域管理、嵌套作用域与跨组件清理的一致性设计？  
48) **自定义渲染器**：基于 `@vue/runtime-core` 的 `createRenderer` 如何实现 Canvas/WebGL/终端渲染？VNode 与 Host Config 的职责边界？  
49) **Web Components 集成**：`defineCustomElement` 的能力与限制；Shadow DOM/样式隔离/`::part`/`::slotted` 的协作与陷阱？  
50) **SSR 与“选择性/延迟水合”**：在框架层（如 Nuxt）如何按交互/可见性/空闲时机水合？对 SEO 与可感知性能的影响？  
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
# 前端构建工具高频面试题（50 题）

1) **Webpack 打包管线**：从入口解析、依赖图构建到产物生成的全过程？`loader` 与 `plugin` 的执行顺序与 Tapable Hooks 是怎样的？  
2) **HMR 原理对比**：Webpack Dev Server / Vite / esbuild 的热更新实现差异？模块边界与状态保留如何处理？  
3) **Tree-Shaking 生效条件**：只对 ESM？`sideEffects` 字段、`/*#__PURE__*/` 标注、类字段与动态属性的影响？  
4) **代码分割策略**：`import()`、Webpack `SplitChunks`、Rollup/Vite `manualChunks` 的差异与“vendor 大包”隐患？  
5) **长期缓存（Long-term Caching）**：`contenthash/chunkhash`、`runtimeChunk`、稳定 `moduleIds/chunkIds` 的配置与坑？  
6) **Source Map 选择**：开发/生产常用类型（`eval-cheap-module-source-map`、`hidden-source-map` 等）与**安全/性能**权衡？  
7) **模块解析规则**：别名（alias）、`extensions`、`mainFields`/`exports` 条件导出、`browser` 字段如何影响最终产物？  
8) **CSS 处理链**：CSS Modules、PostCSS、Tailwind、`@layer` 与 **CSS Code Split** 在不同工具中的实现差异？  
9) **静态资源管线**：Webpack Asset Modules vs 经典 `file/url-loader`；Vite 资源 URL、内联阈值与图片/字体指纹策略？  
10) **环境变量与常量替换**：Webpack `DefinePlugin` vs Vite `import.meta.env`，如何借助 DCE（死代码消除）剔除分支？  
11) **Vite 预构建（dep optimization）**：为什么用 esbuild 预打包？何时失效、如何诊断与强制重建？  
12) **Vite 开发态架构**：原生 ESM + 即时转换链（Plugin hooks）+ 精准 HMR；与“先打包后服务”的范式区别？  
13) **Rollup 作为库打包器**：`external`/`peerDependencies`、多格式输出（`esm/cjs/umd/iife`）与 Tree-Shaking 友好度？  
14) **构建性能优化**：Webpack 持久化缓存、并行/多进程、esbuild/swc 辅助；冷启动与增量构建的瓶颈定位？  
15) **Babel vs SWC vs esbuild**：转译与压缩能力差异、插件生态、与 TypeScript 的协作模式（仅转译 vs 类型检查分离）？  
16) **TypeScript 策略**：`tsc --noEmit` 做类型检查，打包走 swc/esbuild；`projectReferences`/`paths` 与编译结果路径映射？  
17) **图片与字体优化**：SVGO、imagemin/Squoosh 管线；KTX2/Basis 压缩纹理在 Web 的接入与构建支持？  
18) **Monorepo 构建**：pnpm/yarn workspaces、Nx/Turborepo 的任务图与缓存；跨包别名/TS references 如何联动打包器？  
19) **SSR/SSG/边缘渲染集成**：Vite SSR、Next（Turbopack/Webpack）、Nuxt（Vite/Rollup）如何区分 server/client 构建与同构注入？  
20) **Workers/Worklets 打包**：`new Worker()`/`AudioWorklet`/`OffscreenCanvas` 的产物拆分、publicPath 与跨域限制？  
21) **微前端方案**：Webpack Module Federation vs Import Maps/SystemJS 的差异、共享依赖与运行时冲突治理？  
22) **CSP 与构建**：`nonce/hash` 内联脚本、去除 `eval`、`wasm-unsafe-eval` 限制下的 Source Map 与 HMR 适配？  
23) **国际化分包**：语言包动态加载、按路由/区域懒加载策略与缓存键（`Accept-Language`/自定义 header）设计？  
24) **兼容与 Polyfill 策略**：`browserslist`、core-js 注入、Vite `@vitejs/plugin-legacy` 的差异与代价？  
25) **CSS Tree-Shaking 与 Critical CSS**：Purge/Content 提取误删风险、动态类名处理、首屏关键 CSS 抽取策略？  
26) **产物分析与体积预算**：`webpack-bundle-analyzer` / `rollup-plugin-visualizer`；CI 中设定预算阈值与失败策略？  
27) **预加载与优先级**：`<link rel="preload/prefetch">`、Priority Hints（`fetchpriority`）与打包分片的协同？  
28) **多页面应用（MPA）构建**：多入口 HTML 生成、共享运行时代码的抽取、公共资源缓存与路由隔离？  
29) **环境分层与配置管理**：Webpack `mode`/`webpack-merge`、Vite `.env.*` 与 `define`/`build.rollupOptions` 的协同？  
30) **供应链与安全**：依赖锁定、SRI（子资源完整性）生成、`npm audit`/SCA、恶意包防护与 CI 里的签名验证？  
31) **确定性构建**：records 文件、`deterministic` ids、哈希稳定性；如何避免“每次构建 chunk 名变动”？  
32) **WASM 集成**：wasm-pack/`vite-plugin-wasm`；同步 vs 异步初始化、`COEP/COOP` 与跨源隔离要求？  
33) **新一代打包器**：Rspack/Turbopack/Farm 的架构思路、与 Webpack/Vite 的兼容性与迁移考量？  
34) **CI/CD 实践**：缓存 `node_modules`/`pnpm store`、Docker 层缓存、产物与 Source Map 上传、工件回滚策略？  
35) **疑难构建排错**：循环依赖、CJS/ESM 互操作（`require` vs `import`）、`Cannot use import statement outside a module` 等典型问题的定位路径？
36) **Parcel**：零配置管线如何工作？缓存与并行打包、内容感知（Auto Detect）和 HMR 的机制；与 Vite/Webpack 的取舍与典型落地场景？  
37) **Turbopack**：增量图/按需编译的核心思路、与 Webpack 的兼容层现状；大体量项目迁移评估与“功能完备度 vs 性能”的权衡？  
38) **Rspack / Farm**：Rust 系 bundler 的架构要点、Loader/Plugin 兼容策略、与 Webpack/Vite 的差异；迁移步骤与已知坑位？  
39) **esbuild 插件与二开**：`onResolve/onLoad` 生命周期、namespace/虚拟模块、缓存键与并发安全；典型插件（别名、MD/CSV 读入）如何实现？  
40) **Rollup/Vite 插件生命周期**：`resolveId/load/transform/generateBundle` 的职责边界；`this.emitFile` 对代码分割与资源指纹的影响？  
41) **Webpack 插件（Tapable）**：`compiler`/`compilation`/`NormalModuleFactory` 关键钩子与 Asset Pipeline；如何在生成阶段安全地修改产物？  
42) **持久化缓存与失效**：Webpack `cache`、Rspack/Vite 依赖预构建缓存的 key 组成（loader 版本、环境变量、查询串、文件 Hash）；定位“缓存未失效/错误命中”的步骤？  
43) **Dev Server 代理与 HTTPS**：开发代理（WebSocket/HTTP/HTTP2）与 CORS 的交互、HMR 跨域；自签名证书/HTTP3 开发约束与常见报错定位？  
44) **多目标产物**：面向 **Node/Edge/Browser** 的三套构建如何并行维护？`platform/target/conditions` 与 polyfill 策略（无 Node 内置、`fetch`/`webcrypto` 限制）？  
45) **包导出策略**：`exports`/`imports`/`types`/`main`/`module` 的组合与 **Dual Package Hazard**；在 bundler 与 Node 下如何做一致性验证？  
46) **库发布最佳实践**：多格式输出（ESM/CJS/IIFE/UMD）、`external`/`peerDependencies`、`sideEffects` 标注与 Tree-Shaking；如何给使用者保留 DCE 空间？  
47) **CSS 工具链**：Lightning CSS vs PostCSS（Autoprefixer/CSSnano）；`@nest`/`@layer`/容器查询 等新特性的编译与兼容；何时用静态抽取（vanilla-extract/Linaria）？  
48) **Source Map 全链路**：多阶段转换（TS→Babel/SWC→Minify）如何保持映射准确？`hidden-source-map` vs external vs inline 的取舍与隐私合规（sourcemap 只上传错误平台）？  
49) **“无打包开发”与图失效**：Vite/Snowpack 的原生 ESM（dev 时不打包）对大仓 HMR 稳定性的影响；模块图失效/循环依赖导致的热更异常如何定位？  
50) **Monorepo 编排与远程缓存**：Nx/Turborepo/Bazel 如何与 bundler 协作实现任务图与增量构建？缓存粒度、工件复用与 CI 下的跨平台缓存策略？  

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

## CSS（常见面试题版）
13) 盒模型与 `box-sizing`：标准盒/怪异盒、`margin` 合并、BFC 触发与典型应用（清除浮动、两栏自适应）。  
14) 层叠与优先级：特指度、`!important`、`z-index` 与**层叠上下文**的创建时机与常见误判。  
15) 布局选型：Flex 常见模式（居中、等高、三栏）与坑点（`flex-basis`/`min-width`），何时用 Grid、两者边界。  
16) 定位与居中：`position`（relative/absolute/fixed/sticky）与多种水平+垂直居中方案对比。  
17) 单位与尺寸：`px/%/em/rem/vw/vh` 的继承与计算；根字号与移动端适配策略。  
18) 响应式与移动端：`<meta viewport>`、断点规划、流式排版、弹性媒体（`max-width:100%`）。  
19) 选择器与伪类/伪元素：`nth-child` vs `nth-of-type`、`:not()`、`::before/::after`，选择器可维护性与性能。  
20) 图像与图标：`<img>` vs `background-image`、SVG（内联/雪碧）vs Icon Font、多倍图与 `image-set()`。  
21) 动画与过渡性能：`transform/opacity` 优先、`will-change` 边界、合成层与卡顿排查。  
22) CSS 模块化与规范：BEM、预处理器（Sass/LESS）、PostCSS（Autoprefixer）、CSS Modules 的作用域策略。  
23) 字体与稳健性：`@font-face`、字体子集/预加载、`font-display`（swap/optional）、降低 FOUT/FOIT 与 CLS。  
24) CSS 自定义属性（变量）与主题：暗黑模式/多品牌主题切换、运行时覆写变量的组织与降级。

## DOM / BOM（常见面试题版）
25) 事件模型与委托：捕获/目标/冒泡、`stopPropagation`/`stopImmediatePropagation`、`passive` 的影响与使用场景。  
26) 表单与输入：提交前拦截、原生校验 API、自定义校验、输入法合成事件（`compositionstart/update/end`）。  
27) DOM 操作与性能：`DocumentFragment`/模板、批量读写合并、`offsetWidth` 等强制回流触发点与规避。  
28) 渲染流水线：回流（reflow）/重绘（repaint）/合成（composite），避免 layout thrashing 的实践。  
29) 观察者 API：`MutationObserver` / `IntersectionObserver` / `ResizeObserver` 的差异、典型组合（懒加载/无限滚动/自适应容器）。  
30) 剪贴板/拖拽/文件：权限模型、粘贴图片/HTML、拖拽目录、`DataTransfer` 安全与沙箱约束。  
31) 历史与导航：`history.pushState/replaceState`、`scrollRestoration`、单页过渡与 `Navigation API` 的兼容策略。  
32) 存储与同源：`localStorage/sessionStorage/IndexedDB` 的选择；跨标签页同步（`BroadcastChannel`/`storage` 事件）；COOP/COEP 与 `SharedArrayBuffer`。  
33) Web 安全常识：XSS/CSRF/Clickjacking 的前端防护；CSP/Trusted Types；富文本消毒（DOMPurify）的边界。  
34) Service Worker 与缓存：`CacheStorage` 策略、离线优先/网络优先、版本更新（跳过等待/回滚）流程。  
35) 性能与观测：Web Vitals（LCP/CLS/INP）采集、`PerformanceObserver`、Long Tasks 与交互延迟优化路径。

---
# 大前端算法高频面试题清单（50 题）

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
36) 最长递增子序列 LIS（贪心 + 二分，O(n log n)）  
37) 字符串匹配三板斧：KMP / Z-Algorithm / Rabin–Karp（滚动哈希碰撞处理）  
38) 会议室问题：最少会议室数 / 能否参加所有会议（扫描线 / 小顶堆）  
39) 合并 K 个有序链表/数组（分治归并 / 小顶堆）  
40) 数据流中位数 / 滑动中位数（双堆 + 延迟删除）  
41) 两个有序数组的中位数（对数级二分，O(log(min(m,n)))）  
42) 单词拆分（Word Break，DP + Trie/记忆化优化）  
43) 单词接龙（Word Ladder，BFS / 双向 BFS）  
44) 网格最短路径（可移除 K 个障碍，BFS 状态扩展：row,col,k）  
45) 不同子序列个数（Distinct Subsequences，DP）  
46) 最大子数组和 / 环形最大子数组和（Kadane + 前缀和技巧）  
47) 零钱兑换 I/II：最少硬币数 / 组合数量（完全背包 DP）  
48) 0/1 背包（容量约束下的最大价值，滚动数组优化）  
49) Aho–Corasick 自动机（多模式匹配，失败指针 + Trie）  
50) 线段树 / 树状数组（区间求和/最值的查询与单点/区间更新，对比差分/前缀和的适用边界）

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

1) 协议总览：请系统列出并简述前端必须掌握的以太坊/多链协议与标准（EIP-1193、EIP-1474、EIP-1559、EIP-3085、EIP-3326、EIP-712、EIP-1271、EIP-4361、EIP-2612、EIP-681 等），分别解决什么问题？  
2) 多钱包连接：基于 **EIP-1193** 的 Provider 事件模型（`request`/`accountsChanged`/`chainChanged`）如何落地一个“连接/断开/切换账户与网络”的完整前端状态机？  
3) WalletConnect v2：解释 **Pairing** 与 **Session** 的区别、**CAIP-2/10** 命名空间与权限协商；前端如何做“多链多账号选择”“断线重连”“会话续期”？  
4) 链切换协议：**EIP-3085** `wallet_addEthereumChain` 与 **EIP-3326** `wallet_switchEthereumChain` 的差异；钱包不支持切换时，前端应如何降级与提示？  
5) SSR/移动端适配：SSR 环境与移动端 Deep Link 下，如何检测 Provider、区分注入钱包与 App 跳转（WalletConnect/Universal Link），并保证首屏无报错？  
6) 签名协议矩阵：`eth_sign`/`personal_sign`（EIP-191）与 `eth_signTypedData_*`（EIP-712）的原理差别、域分隔与防重放；前端何时用哪种，如何在 UI 上做“可读意图”提示？  
7) SIWE 登录（EIP-4361）：挑战-响应流程与域/链绑定；多钱包/多链切换时，前端如何维持/失效登录态并与服务端会话联动？  
8) 合约账户签名（EIP-1271）：前端如何在 EOA 与合约账户混合场景下验证签名与展示签名者身份？  
9) 链上支付（1）：基于 **EIP-681**（Payment URI）与二维码的支付请求如何生成与解析？如何设计“金额/链/收款地址/备注”参数并防钓鱼？  
10) 链上支付（2）：**ERC-20/Permit（EIP-2612/Permit2）** 支付链路的前端交互：离线授权 → 代扣 `transferFrom` → 成功回执；如何限制额度/到期并提供“撤销授权”入口？  
11) 交易费用（EIP-1559）：`baseFee/maxFeePerGas/maxPriorityFeePerGas` 的关系；前端如何通过 `eth_feeHistory` 与 `eth_maxPriorityFeePerGas` 做稳健的小费建议与“加速/取消”替换交易？  
12) 交易生命周期：`simulate (eth_call)` → `estimateGas` → `sendTransaction` → `wait` → `confirmations` 的状态机；如何处理 `nonce` 冲突、`replacement underpriced`、超时与 Reorg？  
13) 只读/可写分层：在 viem/ethers 中如何抽象 **provider（读）** 与 **signer（写）**，并用链配置/地址表避免跨链读错？  
14) 批量读取（Multicall）：Multicall2/3 的 `aggregate/tryAggregate` 差异；前端如何分页拆分长列表、处理部分回退并合并结果？  
15) 日志与事件：`eth_getLogs` 的 topic 过滤与分段窗口策略；如何用 `finalized/safe` 区块标签与 `txHash+logIndex` 去重来对抗重组？  
16) 索引取舍：直接链上日志 vs **The Graph/Substreams**；前端如何做“本地缓存 + 回放校准（区块水位线）”并保证幂等？  
17) 代币交互：`decimals/allowance/approve/transferFrom` 的前端边界；如何避免“无限授权”与授权竞态，设计乐观 UI 与回滚？  
18) NFT 元数据：`tokenURI` → IPFS/Arweave 网关解析、CID 校验、占位/缓存/失败回退；如何处理敏感内容遮罩与 NSFW 提示？  
19) ENS 与 CCIP-Read：namehash/反解流程、**CCIP-Read** 链下读取的安全边界；前端如何降级并标注信任来源？  
20) L2 费用与 EIP-4844：L1 数据费 + L2 执行费的组成、Blob 费对展示的影响；前端如何准确预估/对账并解释与主网差异？  
21) 私有交易与 MEV 防护：公开 mempool vs 私有内存池（Flashbots Protect/MEV-Blocker）原理；前端如何一键切换 RPC 并处理“无公共 txhash/确认变慢”的 UX？  
22) 跨链桥 UX：消息桥（Canonical/ZK/Optimistic）与流动性桥差异；前端怎样展示“提交/证明/执行”三阶段进度、费用拆分与失败回退路径？  
23) 多签（Safe）适配：Safe Apps SDK/Tx Service 的工作方式；前端如何兼容“批量交易/阈值签名/队列执行/模拟结果”的可视化？  
24) 账户抽象（EIP-4337）：UserOperation、Bundler、EntryPoint、Paymaster 的交互流程；前端如何解释“代付/限权 Session Key”并在不支持时回退到 EOA？  
25) 自定义错误与 Revert 解码：Solidity Custom Errors 的 ABI 编解码；前端如何把 `CALL_EXCEPTION/data` 转换为用户可读的错误文案与行动建议？  
26) 价格与预言机：Chainlink Aggregator 读取与 stale 检测；前端如何在报价/清算/滑点提示中使用 **TWAP/中位数** 并给出反操纵说明？  
27) 隐私与零知识：前端本地/远端生成证明的取舍、证明大小/时间的反馈；如何在提交/验证失败时给出可诊断信息并建议降级？  
28) 多链地址簿与 ENS：不同链同名地址冲突、EIP-55 校验和、地址-链绑定；如何在输入框做强校验与 ENS 头像渲染？  
29) 速率限制与容错：多 RPC Fallback/加权、超时/指数退避/幂等重试；前端如何把“最终一致”呈现在 UI（重放按钮/错误快照）？  
30) 安全签名 UX：对高风险方法（`approve`、`setApprovalForAll`、`permitAll`、`increaseAllowance`）如何在签名弹窗前给出“明示目的/额度/到期/可撤销”提示？  
31) 交易模拟：Tenderly/Anvil/Foundry Fork 的 `eth_call` 预检如何在前端集成？哪些失败（权限/余额/路径）可以在发送前阻断并解释？  
32) 链上支付风控：电商收款/订阅扣费的前端设计（固定金额/最小收到/截止时间/一次性授权 vs 定期授权）；如何防“换地址/链骗局”与混淆代币？  
33) 代币列表安全：Token List（Uniswap 标准）的签名信任链；前端如何做来源白名单/哈希校验与“未知代币”强提示？  
34) SDK 选型与包体：ethers v5/v6、viem、wagmi/RainbowKit 的体积与 tree-shaking 差异；如何按路由懒加载 ABI/链配置并避免 Node polyfill 膨胀？  
35) 端到端设计题：实现“多钱包连接 + EIP-712 登录 + Multicall 批量余额查询 + Permit 代扣支付 + 1559 费用与私有 RPC 可选 + 事件/确认追踪（重组容错）”，请给出页面状态机与安全提示清单。  

---
# TypeScript 高频面试题（35 题）

1) TypeScript 的“结构类型系统”如何决定两个对象类型的兼容性？与“名义类型”有何不同，哪些场景会导致“意外兼容”？  
2) `any / unknown / never` 的语义边界分别是什么？在项目中如何限制 `any` 的扩散并合理使用 `unknown`？  
3) 类型推断与字面量收缩：为什么 `const x = 'a'` 与 `let x = 'a'` 推断不同？`as const` 会带来哪些连锁影响？  
4) 联合类型与交叉类型在可空性/可选属性上的行为差异？`strictNullChecks` 对联合类型的可用性有什么改变？  
5) 类型收窄（控制流分析）如何工作？请对比 `typeof / instanceof / in / 判空` 与“可辨识联合（discriminated union）”的配合方式与局限。  
6) 如何编写**用户自定义类型守卫**（`x is T`）？在哪些情形会失效（跨函数边界、异步场景、闭包逃逸）？  
7) 函数类型：参数是否“逆变”？`strictFunctionTypes` 打开/关闭的行为差异及对回调类型（如事件处理器）的影响。  
8) `interface` 与 `type` 的能力差异与取舍？声明合并、递归类型、计算属性各在哪类问题更合适？  
9) 索引类型体系：`keyof / T[K] / 映射类型` 的组合用法；如何用 `as` 做**键重映射**（重命名/过滤键）？  
10) 常用工具类型的原理：`Partial / Required / Readonly / Pick / Omit / Record / NonNullable / ReturnType / Parameters / InstanceType` 可分别如何用条件/映射类型实现？
11) 泛型参数设计：约束（`extends`）、默认参数、`keyof T` 约束链；如何避免“过度泛型化”导致的可读性下降？  
12) 分布式条件类型与 `infer`：实现 `Awaited / ReturnType / Parameters` 的关键技巧；哪些写法会触发“分布式”？  
13) 模板字面量类型：如何将 `'getName' | 'getAge'` 生成 `'setName' | 'setAge'`？如何用模板类型建模“路由/路径”字符串？  
14) 变长元组/参数列表类型：如何实现 `Head / Tail / Concat / TupleToUnion`？变长参数在重载与推断中的坑点有哪些？  
15) `satisfies` 与 `as const` 的差异：如何**既校验对象字面量形状**又**保留更宽的变量类型**（避免推断“太窄”）？  
16) `readonly` 在属性、数组、元组中的行为；如何实现浅/深只读（`DeepReadonly`）并讨论其局限（如函数类型）？  
17) 枚举对比：`enum / const enum / 字面量联合` 的编译产物与运行时特征；何时应避免 `const enum`？  
18) 模块解析与别名：`baseUrl / paths / typeRoots / types / moduleResolution (node|bundler)` 的差异；对 Vite/webpack/ts-node 的联动影响？  
19) DOM/Node 多目标工程：如何通过 `lib`、多 `tsconfig` 与条件导出同时支持浏览器与 Node，避免全局类型冲突？  
20) 函数重载 vs 联合/泛型：三者的表达力与可维护性对比；实现签名与重载签名的分离规则有哪些坑？
21) 协变/逆变/双向协变：为何 TS 在回调上采用“参数双向协变（bivariance）”的历史兼容策略？如何在关键边界强制逆变安全？  
22) “名义性”建模：如何用**品牌/不透明类型（branded/opaque）**防止 ID 混用？给出可跨包扩展的品牌方案。  
23) 避免“分布式条件陷阱”：为什么 `T extends U ? X : Y` 会自动分发？何时用 `[T] extends [U]` 抑制分发以获得期望结果？  
24) 高级工具类型实现题：`DeepPartial / DeepRequired / Merge / Mutable / UnionToIntersection / InclusiveKeys` 的实现与边界讨论。  
25) 键重映射 + 模板类型：根据接口 `{ getUser():User; getPost():Post }` 自动生成 `setUser / setPost` 的 API 类型与实现约束。  
26) 错误类型安全：`unknown` 错误如何处理？`instanceof` 在跨 realm 下为何失效？如何在 API 边界建模 `Result<E, A>`？  
27) 声明增强与全局污染：何时需要**模块增强/全局声明合并**（为第三方库补类型）？如何避免“污染式增强”失控？  
28) 装饰器生态：TC39 新装饰器 vs TS 旧实验装饰器的语义差异；`emitDecoratorMetadata` 的体积/反射风险与收益评估。  
29) 项目引用（Project References）：如何拆分大型仓库、加速增量构建与发布？`composite / declarationMap` 的正确配置与常见踩坑。  
30) 类型性能治理：如何定位“类型层爆炸”（深递归、复杂分布式条件）？`--diagnostics`、简化工具类型、边界断开（`any` 隔离法）等手段的取舍。
31) `.d.ts` 与声明发布：如何为 JS 包/第三方库编写/补充类型，组织 `typesVersions` 与 `exports` 以兼容 ESM/CJS？  
32) 类型 ↔ 运行时校验：如何将 zod/valibot/io-ts 与 TS 类型连接（`z.infer`），并讨论从 TS 生成 JSON Schema 的可行/不可行边界？  
33) React/Vue 高级建模：在 React 中对 Props、受控组件、`forwardRef`/`memo` 做精确类型；在 Vue 中为 `defineProps/defineEmits/slots` 建模并推断 `v-model`。  
34) 面向库作者的 `declaration / declarationMap / sourceMap` 策略：如何让下游获得精准跳转与调试体验，同时控制包体与隐私泄露？  
35) 面向未来的类型策略：装饰器落地、Records & Tuples、TypedArray 扩展、ESM 解析策略变更等新特性到来时，如何设计**可回退**的类型与发布方案？

---

# 2D + 3D 图形渲染高频面试题（共 35 题）

1) Canvas 2D 的渲染模型与命令式绘制栈：路径、状态栈（save/restore）、合成（globalCompositeOperation）的原理与适用场景？  
2) 高分屏与像素密度（DPR）：Canvas 逻辑尺寸 vs CSS 尺寸如何标定；像素对齐、Retina 适配与文字/位图清晰度治理？  
3) Canvas 2D 文本与位图：imageSmoothingQuality、drawImage 子矩形/九宫格裁切、getImageData/putImageData 的性能与使用边界？  
4) Canvas 2D 图层与重绘：脏矩形（dirty rect）策略、离屏缓冲（临时 canvas）与合成顺序如何降低重绘成本？  
5) OffscreenCanvas + Worker：把 2D/WebGL 渲染移出主线程的通信、事件回传与降级策略？  
6) WebGL 最小渲染闭环：上下文创建、缓冲（VBO/IBO）、着色器编译/链接、attribute/uniform 绑定与一次 draw 的流水线原理？  
7) WebGL1 vs WebGL2：VAO、Instancing、UBO、MRT、sRGB 的新增能力对工程实践的直接收益与迁移要点？  
8) GLSL 基础与插值：顶点→片元 varyings 的插值规则、精度限定符、分支与循环对性能的影响？  
9) 纹理采样原理：UV 坐标、wrap 模式、min/mag 策略、mipmap 与各级过滤（含各向异性）如何选择？  
10) 深度测试与背面剔除：zNear/zFar 对精度的影响、Z-fighting 成因与缓解（polygonOffset/反向 Z/调近远裁剪面）？  
11) 透明度与混合：混合方程（src/dst factor）、预乘 Alpha 的数学与“白边/暗边”问题如何从素材与代码两侧修复？  
12) 帧缓冲对象（FBO）：离屏渲染、MRT 与后处理链（FXAA、Bloom、色调映射）的基本搭建与性能考量？  
13) 相机与投影：透视/正交矩阵的推导、视椎体裁剪、抖动（TAA/Jitter）与稳定渲染的关系与实现要点？  
14) 光照模型入门：Lambert/Phong/Blinn-Phong 到 PBR（金属度/粗糙度）在实时渲染中的近似与取舍？  
15) 阴影基础：Shadow Mapping 的深度图生成、PCF/PCSS 的软阴影思路、阴影失真/彼得潘效应与 bias 调参方法？  
16) Three.js 场景图：Object3D 的本地/世界矩阵更新、层次合并与 frustumCulling 的执行时机与代价？  
17) Three.js 材质体系：MeshBasic/Lambert/Phong/Standard/Physical 的光照参与度差异；PBR 纹理通道与颜色空间设置（sRGB/Linear）？  
18) Three.js 几何与 BufferGeometry：attributes/index/tangent 的含义与生成；合并几何/压缩（Draco）在何处介入？  
19) Three.js 纹理加载与缓存：TextureLoader/KTX2Loader/CompressedTexture；色彩空间、flipY、各平台采样差异与踩坑？  
20) Three.js 阴影与灯光：Directional/Point/Spot 阴影贴图的开销；shadow.mapSize、bias、normalBias 的调优策略？  
21) Three.js 动画系统：AnimationClip/KeyframeTrack/AnimationMixer 的职责划分；动画混合/过渡与 root motion 的处理？  
22) Three.js 拾取：Raycaster 的射线与坐标系转换、实例化网格（InstancedMesh）拾取与自定义 id/颜色编码的两套方案？  
23) Three.js 后处理：EffectComposer/RenderPass/ShaderPass 的数据流；自定义 Pass 的输入/输出约定与多分辨率渲染？  
24) Three.js 性能工程：draw call 控制（合批/Instancing）、材质/状态切换排序、顶点数 vs 填充率 vs 过度绘制的权衡？  
25) Three.js 透明与排序：renderOrder、depthTest/depthWrite、双通道（opaque+transparent）与半透明对象的常见故障排查？  
26) glTF 2.0 资产管线：节点/蒙皮/动画/相机的表达；Draco 网格压缩与 KTX2/Basis 纹理压缩的打包与运行时解码流程？  
27) 骨骼蒙皮与 Morph Targets：着色器层的变换公式、权重混合与动画重定向（retargeting）的核心步骤？  
28) 资源生命周期治理：并发加载/解压、缓存与引用计数、纹理/几何/材质 dispose、GPU 内存估算与泄漏定位？  
29) 交互事件映射：Canvas 指针事件到 3D 对象交互（hover/select/drag）的射线与坐标变换；pointer capture 的使用边界？  
30) 多视口与 UI 叠加：setViewport/setScissor 的裁剪；在同一画布绘制小地图/PIP/后备缓冲叠加 UI 的实践？  
31) 移动端优化：Tiling GPU 的带宽与填充率约束、分辨率缩放（动态 DPR）、AA 选型（MSAA/FXAA）与降帧策略？  
32) WebGL 兼容与扩展：能力查询与降级（OES_element_index_uint、EXT_color_buffer_float、ANGLE_instanced_arrays）？  
33) 大规模点云/线框与体素：Points/LineSegments 的着色器实现、屏幕空间线条（meshline）与 BVH 加速的基本思路？  
34) 颜色管理与 HDR：three.js 的 outputColorSpace、toneMapping/exposure，线性工作流的重要性与测试校准方法？  
35) WebGPU 入门与迁移：可编程渲染/计算 Pass、WGSL 着色器、资源绑定（bind groups）与从 Three.js/WebGL 迁移的桥接策略与现状？

---
# Electron 高频面试题（35 题）

1) 进程模型总览：Main / Renderer / Preload 的职责边界与生命周期（`app.ready`、`dom-ready`、`did-finish-load`）分别是什么？  
2) `BrowserWindow` 启动链路：`webPreferences` 中 `contextIsolation/sandbox/nodeIntegration` 的含义与默认值变化史？  
3) IPC 原理：`ipcRenderer.send/on` vs `ipcRenderer.invoke/handle` vs `postMessage(MessagePortMain)` 的消息语义与背压/异常传递差异？  
4) Preload 设计：`contextBridge.exposeInMainWorld` 的 API 面向稳定性（版本化、能力最小化）如何落地？如何清理事件/资源避免泄漏？  
5) 基础安全基线：关闭 `nodeIntegration`、启用 `contextIsolation`、配置 CSP，默认应禁用哪些危险能力与协议？  
6) 导航与外链安全：`will-navigate`、`setWindowOpenHandler`、`shell.openExternal` 的正确使用与“任意跳转/钓鱼”防护策略？  
7) WebContents 行为学：导航、进程复用与站点隔离（Site Isolation）对多标签/多域页面的影响？  
8) 多窗体架构：`BrowserWindow` / `BrowserView` / `<webview>` 的差异、嵌套层级与通信/资源回收策略怎么选？  
9) 会话与网络：`session` 分区、Cookie/Cache 隔离、代理设置、`webRequest` 拦截与证书校验（`setCertificateVerifyProc`）原理？  
10) 权限与合规：摄像头/麦克风/屏幕录制在 macOS/Windows/Linux 的权限模型与弹窗时机；如何最小化授权面？  
11) 桌面捕获：`desktopCapturer` 与 `getUserMedia` 的差异、源选择 UI、系统音频捕获与平台限制有哪些？  
12) 文件系统与对话框：`dialog.showOpenDialog`、`app.getPath`、沙箱/路径穿越防护与大文件（流式）读取方案？  
13) 数据持久化：SQLite/LevelDB/IndexedDB 的选型，Renderer 与 Main 的数据读写边界、敏感数据的 Keychain/DPAPI/keytar 加固？  
14) 性能基线：首窗白屏治理（预加载协议、Splash、懒加载）、`backgroundThrottling`、热路径与大对象泄漏定位套路？  
15) GPU/合成：何时需要 `app.disableHardwareAcceleration()`？如何诊断过度绘制/合成层问题与 ANGLE 后端差异？  
16) 内存与资源释放：WebContents 生命周期、窗口/视图销毁、`destroy` vs `close` 行为、`gc()` 与 DevTools Performance 配合排查？  
17) 日志与崩溃：Crashpad 架构、`crashReporter.start`、minidump 上报与符号表（dSYM/PDB）管理、隐私脱敏的流程？  
18) Tracing 与性能剖析：`contentTracing`、Chrome Tracing 时间轴/火焰图读取方法与典型瓶颈图谱？  
19) 自动更新原理：`electron-updater` vs Squirrel vs MAS 的适用边界、差分更新/失败回滚/多通道灰度策略如何设计？  
20) 代码签名与发布：macOS notarization + hardened runtime、Windows 代码签名链与证书选择、常见签名/公证失败原因与排查？  
21) 打包工具链：`electron-builder` / `electron-forge` / `electron-packager` 的差异、常见配置（asar、多平台、nsis/dmg）与坑点？  
22) ASAR 与资源治理：开启 asar 的优缺点、运行时解压策略、对热修复/差分更新/增量下载的影响？  
23) 通知系统：`new Notification` 与各平台原生通知中心差异，交互按钮/深链返回的实现约束？  
24) 托盘/菜单/任务栏：Tray、Dock、Jump List、Global Shortcut 的平台异同与快捷键冲突/焦点管理问题处理？  
25) 深链与单实例：`app.requestSingleInstanceLock`、`setAsDefaultProtocolClient` 的实现细节与跨平台传参/激活窗口策略？  
26) 自定义协议：`protocol.registerSchemesAsPrivileged` 风险评估，`registerFile/Buffer/Stream/HttpProtocol` 的取舍与 CSP 配置？  
27) Web 安全策略进阶：`webSecurity`、`allowRunningInsecureContent`、`trustedTypes`、`contextIsolation` 组合在桌面端的边界？  
28) 打印与 PDF：`webContents.print/printToPDF` 的分页/页眉页脚/背景图/精度控制差异，跨平台字体/版式稳定性？  
29) 原生能力与 Node-API：C++ 模块编译、预构建二进制、Electron 版本与 Node ABI 对齐策略（prebuild/optionalDependencies）？  
30) 多窗口数据同步：Main 为“数据总线”还是使用跨进程存储？广播/请求-响应/共享内存的权衡与一致性设计？  
31) CI/CD 实战：多平台交叉构建、Apple Silicon/Universal 二进制、签名/公证自动化、密钥保管与缓存策略？  
32) 大型应用架构：模块边界、权限分层（Preload API 白名单）、按域分层（内嵌三方页面隔离）与“安全默认关闭”的工程体系？  
33) 可靠性工程：崩溃自愈（重启/会话恢复）、离线可用、主/渲异常隔离、监控指标（崩溃率/启动时延/内存峰值）与告警？  
34) 升级与迁移：Electron 大版本断点（Chrome/Node/ABI）带来的兼容问题、Remote 模块移除的替代与回滚设计？  
35) 端到端设计题：为“多窗体+自动更新+屏幕共享+企业代理”的跨平台 Electron 应用，给出从安全基线到发布/监控的完整技术方案与关键权衡点。

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
