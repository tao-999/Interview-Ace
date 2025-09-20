# React面试题

# 1) React 18 的并发渲染（Concurrent Rendering）做了什么？调度/可中断渲染/优先级如何工作？

**核心目标**：让渲染**可中断、可恢复、可分段**，从而在繁忙页面中让高优先级交互（输入、动画）更流畅。

**关键点**
- **时间切片（time slicing）**：渲染过程被切成小片段，渲染中可让出主线程给高优先任务（如输入）。
- **优先级调度**：内部以 **Scheduler** + **Fiber lanes** 管理不同优先级任务（如 `transition` 被标记为**非紧急**）。
- **可中断/恢复**：一次渲染未完成时，若有更高优先任务到来，当前渲染被**中断**，待空闲再继续。
- **不改变结果**：并发渲染只改变**调度方式**，不会改变渲染**最终结果**（逻辑仍需写成**纯渲染、无副作用**的组件）。
- **自动批处理（automatic batching）**：React 18 将 Promise/事件/定时器产生的多个 `setState` 自动批处理，减少不必要重渲染。
- **StrictMode “双调用”**（开发模式）：帮助发现副作用不纯的问题；生产环境不会双调用。

**易踩坑**
- 渲染中读取外部可变数据（非 state/props）会因**中断/重做**而出现不一致。
- 依赖 `useEffect` 执行时机进行计算的逻辑要确认不会因**批处理/调度**改变假设。

---

# 2) `useTransition` 与 `useDeferredValue` 的区别与使用场景

**`useTransition()`**
- 用法：`const [startTransition, pending] = useTransition();`
- 作用：把一段**非紧急状态更新**标记为 *transition*，让输入类更新（紧急）先执行，非紧急更新在空闲再渲染。
- 场景：**输入框 + 大列表筛选**。先响应输入，再异步更新大列表；可用 `pending` 做“加载中”。

**`useDeferredValue(value)`**
- 作用：返回 `value` 的**延迟版本**。当 `value` 频繁变化时，下游昂贵组件只在空闲时接收“稳定版”值，降低抖动。
- 场景：**搜索建议组件**消费 `deferredQuery`；输入快速变化不频繁触发昂贵渲染。

**对比**
- 颗粒度：`useTransition` 是**更新发起点**的标注；`useDeferredValue` 是**数据消费端**的延迟。
- 控制力：`useTransition` 可分组多次 `setState`；`useDeferredValue` 只影响传入的那份值。
- 组合：常见模式是输入处用 `startTransition`，展示端再用 `useDeferredValue` 稳定下游。

**注意**
- 两者只“让路”，不会请求/数据层去抖；仍需在数据层设计防抖/合并。

---

# 3) Suspense 在“数据获取”与“代码分割”两类场景的机制差异

**共同点**
- 都以 **Suspense Boundary** 为单位：子树**挂起**时展示 `fallback`，就绪后**无缝揭示**。
- 都支持**流式（streaming）**与**分段揭示（reveal order）**，配合 SSR 时可早出骨架、后续填充。

**代码分割（`React.lazy`）**
- 悬挂来源：动态 `import()` 的模块**未加载**。  
- 机制：`lazy()` 产生一个会在模块加载完成后“解决”的组件；在解决前 Suspense 展示 `fallback`。
- 边界：适用于**资源（JS/CSS）加载**，不关心数据。

**数据获取（Data Fetching Suspense）**
- 悬挂来源：组件读取的数据**Promise 未解决**（React 19 客户端可用 `use(promise)`；RSC/框架也会抛出可挂起的 promise）。
- 机制：读取数据时“抛出”一个 promise，边界捕获并显示 `fallback`；完成后重试渲染。
- 边界：需要**与框架/库约定**的数据读取方式（如 Next.js RSC、`use()`、或 TanStack Query 的 `suspense` 模式）。

**设计要点**
- **粒度**：把昂贵/不确定的数据读取放到**更靠近 UI 的边界**，提升“部分可用性”。  
- **错误边界**：与 Suspense **并列**使用；数据/加载错误由 ErrorBoundary 兜底。  
- **reveal-order**：列表逐项揭示 vs 一次性揭示；结合用户体验选择。

---

# 4) React Server Components（RSC）核心模型与限制

**运行边界**
- **服务器端**执行的 React 组件（不包含事件处理），渲染为**RSC Payload**（一种可序列化的“组件树描述”）。  
- 客户端接收 payload 与**客户端组件（client component）**进行拼装与水合（仅事件与必要逻辑在客户端）。

**协作方式**
- **Server Component**：可以直接做数据请求（数据库、后端 API），把结果作为 props 传给子树；不会把数据获取代码打包到客户端。  
- **Client Component**：通过 `'use client'` 声明；可使用浏览器/DOM API、事件处理、状态等。
- **边界传递**：Server → Client 只能传**可序列化的 props**；不能把函数/DOM 节点当 props 传下去。

**收益**
- **更小的客户端 JS**（把纯渲染/数据读取留在服务端），**更少的水合成本**，更快 TTI。
- **近后端数据**：减少“瀑布请求”，组合/聚合在服务端完成。

**限制与注意**
- Server 组件不能使用浏览器 API/状态 Hook；事件必须在 Client 组件中处理。  
- 仅能传**可序列化**数据；避免把大对象/循环引用塞进 props。  
- 需要框架级支持（如 Next.js App Router），与传统 CSR/SSR 部署不同。

---

# 5) Server Actions（React 19）工作原理、渐进增强与安全边界

**是什么**  
- 标记为 `'use server'` 的 **异步函数**，可被**表单提交/客户端调用**触发，在**服务器**执行副作用（写库、发送邮件等），返回结果供 UI 更新。

**调用途径**
- **表单**：`<form action={serverAction}>`；无需手写 fetch，浏览器提交 → 框架路由 → 执行 action → 返回并**触发重新渲染/重新验证**。  
- **客户端触发**：在 client 组件中导入并调用已声明的 server action（框架将其编译为远程调用）。

**渐进增强**
- 即使 JS 关掉也可通过表单原生提交触发 Action（SSR/RSC 环境处理），JS 打开时获得更顺滑体验（无刷新、局部更新、Suspense）。

**数据一致性**
- 执行后可触发**缓存失效/重新验证**（如 Next.js `revalidatePath`/`revalidateTag`），保证 UI 与服务端一致。

**安全边界**
- **永远不信任客户端**：在 Action 中进行鉴权、权限判断、参数校验；  
- 避免将敏感逻辑放在客户端；Action 的源代码不会下发到浏览器，但**调用入口**会被暴露（需鉴权）。  
- 对写操作设计**幂等**与 CSRF 保护（框架通常内置）。

---

# 6) `use()` 钩子（React 19）在 Server/Client 的行为与限制

**能力**
- `use(promise)`：在组件渲染期间**解包 Promise**；若未就绪则**挂起**当前组件，由最近的 Suspense 接管 `fallback`。  
- `use(context)`：读取 React Context（可替代 `useContext` 的语法形式，尤其对 RSC 更直观）。

**在 Server 端**
- 可以**直接等待**：服务器线程可同步“等待” Promise 完成并返回渲染结果（无需显式 `await`）。  
- 与 RSC 天然协作：配合数据层实现**无瀑布**的数据拼装。

**在 Client 端**
- 只能在**渲染期间**调用（与其他 Hook 一样的规则），不能在事件处理器中调用。  
- 未完成时会**挂起**，交给 Suspense；完成后继续渲染。

**限制与注意**
- 只能在**组件顶层**调用，不可条件/循环调用。  
- 与 Suspense 搭配使用（否则挂起会向上冒泡到最近边界）。  
- `use()` 不是数据获取库，本身不做缓存/重试，需要配合框架或自建缓存策略。

---

# 7) SSR → 流式渲染 → Hydration 的完整链路与 mismatch 规避

**链路**
1. **SSR（服务端渲染）**：服务器输出**可交互前的 HTML**，加快首屏可见。  
2. **流式渲染（Streaming）**：结合 Suspense，服务器先发送外层 shell 与各段 fallback，数据就绪的分片继续流出并填充。  
3. **Hydration（水合）**：客户端加载 JS，将事件绑定到已有 DOM，并**复用** SSR 的 markup；之后进入正常 CSR 更新。

**常见 mismatch 成因**
- **非确定性输出**：`Date.now()`、随机数、环境差异导致 SSR/CSR 内容不同。  
- **条件渲染依赖 `window`/`navigator`**：SSR 无法访问，CSR 有，导致两端分支不同。  
- **动态 ID**：组件内自己生成 id 不一致；应使用 `useId()`。  
- **数据不一致**：SSR 用旧数据、CSR 立即请求新数据 → 首次水合不同。  
- **CSS 顺序/类名变化**：CSS-in-JS 注入顺序不同造成结构/样式抖动。

**规避策略**
- 把**仅客户端**逻辑放到 `useEffect` 或显式的 Client 组件中；SSR 输出稳定占位。  
- 使用 `useId()` 确保 SSR/CSR 一致 ID；对不可避免的差异使用 `suppressHydrationWarning`（少量、可控）。  
- 数据：通过框架注水（serialize data）将 SSR 数据注入到客户端，**首个渲染复用**该数据；随后再做后台刷新。  
- 代码分割与 Suspense：为异步模块/数据配置恰当边界，避免整页等待。  
- 监控：在开发/预发开启 hydration 日志，CI 跑 `lint` 与端到端对比快照。

**与路由/框架的关系**
- Next.js App Router（RSC + Server Actions + Streaming）已将上述链路工程化；正确使用其数据缓存/重新验证更易避免 mismatch。
# 8) 现代状态管理选型：Context、Redux Toolkit、Zustand、Jotai、TanStack Query

**问题切分（UI 本地态 vs 远端数据态）**
- **本地 UI 态**（开关、选中项、向导步骤、临时表单）：组件内部/Context/轻量库（Zustand、Jotai）。
- **远端数据态**（请求、缓存、重试、失效）：**TanStack Query**（TQ）优先，它不是“全局状态”，而是**服务器状态**专用库。

**Context**
- 适合：低频变化的全局信息（主题、用户、权限、设置）。
- 痛点：大量/频繁更新会引发**提供方下所有消费方重渲染**；可用 context selector 或拆分 Provider 缓解。

**Redux Toolkit（RTK）**
- 亮点：标准化写法（immer/RTK Query）、可预测、社区与工具链成熟（DevTools、时间旅行）。
- 适合：多人协作、大型项目、严格审计；与 RTK Query 可一站式处理服务器状态。
- 注意：要避免“把一切都放进全局 store”，UI 局部态不必全局化。

**Zustand**
- 亮点：极简、无样板、细粒度选择器（`useStore(selector)`），默认浅比较减少重渲染。
- 适合：中小规模 UI 态、游戏/可视化面板等高频交互。
- 注意：自行约束不可变与日志，复杂场景需自搭中间件（persist/devtools/immer）。

**Jotai**
- 亮点：原子化依赖图（像 Recoil），每个 atom 是最小可订阅单元，组合灵活。
- 适合：复杂局部状态依赖、跨组件原子组合；小而美。
- 注意：依赖图复杂度随规模上升，需要良好命名与分层。

**TanStack Query（TQ）**
- 亮点：缓存、失效、重试、并发去重、乐观更新、分页/无限加载，天然与 Suspense/Streaming 协作。
- 适合：绝大多数数据获取场景；与表单/变更配合 `mutations`。
- 注意：不要把纯 UI 态放进 TQ；**无状态变更就不需要 Redux** 的典型项目很多。

**经验法则**
- 先用组件内 state → 提升到 Context → 仍痛就引入 Zustand/Jotai。
- 服务器数据统一交给 TQ 或 RTK Query，明确“失效”和“来源”。

---

# 9) `useEffect` / `useLayoutEffect` / `useInsertionEffect` 使用时机与影响

**useEffect**
- 运行时机：**commit 之后**，异步调度，不阻塞绘制。
- 用途：数据订阅、事件监听、日志、非阻塞副作用。
- 清理：返回清理函数，在下一次依赖变化/卸载时执行。

**useLayoutEffect**
- 运行时机：**DOM 更新后、绘制前**（同步），会阻塞浏览器绘制。
- 用途：**读取布局并同步写回**（测量 DOM、同步滚动、计算位移）。
- 警告：滥用会造成布局抖动和卡顿；优先尝试 `useEffect`，仅在读写布局需要同步时使用。

**useInsertionEffect**
- 运行时机：在布局副作用之前，**用于 CSS-in-JS 插入样式**，保证样式早于布局测量插入。
- 用途：库作者场景；应用层几乎不用。

**通用建议**
- 依赖数组**写全**，与 ESLint 规则对齐；把函数/对象**提取或 memo**，避免无谓触发。
- 避免在 effect 内直接 `setState` 产生**永动重渲染**；条件更新+依赖收敛。

---

# 10) Hooks 的“闭包陷阱”（stale closure）与依赖数组

**现象**
- 事件回调/异步回调中读取到**旧的 state/props**，因为回调捕获了渲染当时的闭包。

**常见修法**
- **把依赖写进 `useEffect`/`useCallback` 的依赖数组**，让回调随依赖更新。
- 对高频回调：  
  - 使用 `useEvent`（提案/框架支持时）或社区等价方案，获得“始终看到最新值”的稳定函数引用；  
  - 或在回调内读取 `ref.current`（由 `useEffect` 同步最新值）。
- 对“只需要最新值，不要重新绑定”的场景：`const latest = useRef(value); latest.current = value;` 在回调中读 `latest.current`。

**反例**
```ts
const [count, setCount] = useState(0);
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, []); // ❌ count 被固定为 0
```

**正例**
```ts
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000); // ✅ 函数式更新
  return () => clearInterval(id);
}, []);
```

**依赖数组策略**
- **函数式更新**可避免把 state 塞进依赖。
- 组件外的常量/稳定引用（如 memoized 函数、不可变配置）不需要列入依赖。
- ESLint 的 `react-hooks/exhaustive-deps` 是**朋友**，对不符合约束的情况**重构**而不是直接忽略。

---

# 11) `useMemo` / `useCallback` 收益与成本；识别过度 memo 的方法

**收益**
- 避免昂贵计算重复执行（`useMemo`），或避免因函数新建导致的子组件不必要重渲染（`useCallback`）。
- 仅当**计算昂贵**或**引用稳定性被依赖**时才值得。

**成本**
- 每次渲染需要对比依赖并维护缓存，有**额外开销**；在轻量计算/结构中可能得不偿失。
- 过度 memo 会复杂化依赖管理，增加 bug 面积。

**何时使用**
- 计算 > O(n) 且 n 不小，或包含复杂对象创建/正则解析/格式化。
- 子组件对 props 做 `memo`/`React.memo`，需要稳定的函数/对象引用。
- 列表项中将**稳定部分**作为 `children` 传入 `memo` 组件，避免重渲染。

**识别过度 memo**
- Profiler 观察 memo 维护成本 > 节省成本；  
- 移除 `useMemo/useCallback` 重新测量，若无明显差异即可删；  
- 对“只在 mount 执行一次”的昂贵初始化，**把结果移出组件**成为模块级常量更有效。

---

# 12) React Compiler 的目标与约束

**目标**
- 通过静态分析让编译器自动推断**哪些值稳定**、哪些地方需要**自动 memo**，减少手写 `useMemo/useCallback`，降低重渲染。

**工作方式（概念）**
- 编译期分析组件依赖图与闭包捕获，自动为稳定表达式生成缓存/比较代码。
- 与并发渲染兼容，不改变语义，只优化**渲染次数与 diff 面积**。

**对代码的约束（可读性 → 可优化）**
- 组件是**纯函数**（不读写外部可变状态）；  
- 对象/数组的创建尽量与依赖绑定，避免隐式全局可变；  
- 避免在 render 阶段做副作用；  
- Hooks 调用满足**顶层一致性**。

**迁移策略**
- 维持“可编译风格”：数据不可变、边界清晰、函数小而纯；  
- 逐步去除不必要的手写 memo，保留**明显昂贵**路径上的手动优化作为兜底。

---

# 13) 表单策略：受控 vs 非受控；与 Server Actions 的融合

**受控表单（controlled）**
- input 值来自 state，`onChange` 更新 state → 组件重渲染。
- 优点：可验证、可回放、严格掌控；  
- 缺点：**高频输入**可能产生性能压力（大量重渲染）。

**非受控表单（uncontrolled）**
- 使用 `defaultValue` 与 `ref` 读取，渲染期间不随输入更新。
- 优点：性能好；缺点：校验/联动需要额外逻辑。

**折中方案**
- 受控外观 + **节流/去抖**；  
- 复杂/大表单使用库（React Hook Form）：**基于 refs/注册机制**，避免整表重渲染；
- 提交时集中校验（zod/yup）+ 字段级错误显示。

**与 Server Actions（React 19）的融合**
- `<form action={action}>`：无 JS 也能提交；JS 可得**局部更新**与 Suspense。
- 验证：客户端**即时检查** + 服务端**权威校验**（在 Action 中再校验，返回错误字段映射）。
- 反冲与回填：Action 成功后**重新验证/失效**相关数据；失败时返回 field errors，客户端聚合显示。

---

# 14) 列表虚拟化关键点（react-window / react-virtual）与避免“行高抖动”

**核心配置**
- 视窗高度（`height`）/项数（`itemCount`）/行高（`itemSize` 或 `estimateSize`）。
- **固定高度**列表：最简单、性能最佳。  
- **可变高度**列表：需要**测量**或**估计 + 缓存**。

**避免行高抖动**
- 提前提供**稳定的估计高度**（保守偏大），首次渲染减少跳动。
- 渲染后**缓存实际高度**（如 `useMeasure`/`ResizeObserver`），下次直接使用；对已测量项**不要丢缓存**。
- 图片/异步内容：限制占位尺寸（固定宽高比或 skeleton），等加载完成再解锁真实高度。
- Sticky 行/表头：与虚拟滚动配合需要**额外容器**或库支持（`innerElementType` 自定义容器）。

**性能细节**
- 行项目组件使用 `memo` + 稳定 `key`；props 传递避免新建对象/函数。
- 批量渲染（overscan）适度增加，减少快速滚动白屏；过大浪费。
- 事件/选择等 UI 状态不放在每一项组件内的本地 state，优先上移或用外部 store（Zustand）以减少重渲染。
- SSR 与虚拟化：首屏可渲染**少量可见项 + 占位**，水合后启用完整虚拟化逻辑。
# 15) Reconciliation 与 `key`：为什么别用索引？分页/拖拽如何保证稳定性

**Diff 的本质**  
- React 按**同层级**比较前后两棵子树；`key` 用于**标识节点身份**。一旦 `key` 变，React 视为**不同节点**：卸载旧 → 挂载新（状态丢失、DOM 复用失败）。

**索引作为 key 的问题**
- **插入/删除/重排**时，后续项的索引全变 → React 复用错误，导致：  
  - 输入框光标跳动/值错位；  
  - 动画/过渡错对象；  
  - 受控组件状态串位。
- 仅当**严格静态**列表（永不重排/增删）才可考虑索引。

**稳定 `key` 的来源**
- 后端 id、业务唯一标识（如 `item.id`）。  
- 组合键（`${type}:${id}`）保证跨类型稳定。

**分页/虚拟滚动**
- 列表跨页时仍使用**全局唯一 id** 做 `key`。  
- 虚拟列表要避免用**可见区内的行号**做 key（随滚动变化），应使用数据 id。

**拖拽重排**
- DnD 时 `key` 不变，**仅顺序变**；事件处理依据 id 更新顺序数组：
```tsx
{items.map(item => <Row key={item.id} data={item} />)}
// onDragEnd: setItems(reorderById(items, srcId, dstId));
```

---

# 16) Error Boundary 的捕获边界；事件、异步、服务端错误如何处理

**Error Boundary 能捕获**
- **渲染阶段**错误（render）  
- **生命周期**（constructor、getDerivedStateFromProps、componentDidMount/Update）  
- **后代**的上述错误

**不能捕获**
- **事件处理器**（onClick 等）：在事件回调里自己 `try/catch` 或上报  
- **异步错误**（`setTimeout`/Promise）  
- **服务端渲染**（SSR）阶段的错误  
- **Error boundary 自身**抛出的错误（需外层再包一层）

**标准写法**
```tsx
class Boundary extends React.Component<{}, {hasError:boolean}> {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  componentDidCatch(err: unknown, info: React.ErrorInfo) {
    report(err, info.componentStack);
  }
  render() { return this.state.hasError ? <Fallback/> : this.props.children; }
}
```

**事件/异步**
```tsx
function Btn() {
  const onClick = () => {
    try { doRisky(); } catch (e) { report(e); }
  };
  useEffect(() => {
    const t = setTimeout(() => { /* try/catch or .catch */ }, 0);
    return () => clearTimeout(t);
  }, []);
  return <button onClick={onClick}>Go</button>;
}
```

**SSR & 路由框架**
- 在**服务端**用框架级 try/catch（如 Next.js 的 error route/`error.js`、`_error`），返回错误页与合适的 HTTP 状态码。  
- 客户端挂载后，仍需 Error Boundary 覆盖**页面局部**。

---

# 17) 事件系统演进：委托位置、合成/原生事件差异与工程影响

**委托位置变化**
- React ≤16：绝大部分事件**委托到 `document`**。  
- React 17+：委托到**根容器**（`root`），便于不同 React 版本共存、微前端并存、Portal 行为更贴近浏览器。

**合成事件 vs 原生事件**
- **合成事件**（SyntheticEvent）：跨浏览器一致的事件包装；**React 17 起不再事件池化**（以往需要 `event.persist()`）。  
- **传播模型**：与原生一致（捕获→目标→冒泡）；`stopPropagation` 在 React 树内生效，对外层非 React 区域不一定拦截。  
- **Passive/捕获**：部分事件（如滚动）使用原生监听，表现与浏览器默认更一致。

**工程影响**
- **与非 React 代码共存**更安全：在根容器内的合成事件不会影响其它容器。  
- **调试**更直观：无需担心事件池化导致异步访问属性为 `null`。  
- **性能**：委托到容器避免跨应用冲突；高频滚动/触控应尽量使用**被动监听**或避免阻塞。

---

# 18) Context 的性能问题与缓解：selector、拆分 provider、`useSyncExternalStore`

**问题根源**
- Provider 的 `value` **引用变化**会让**所有 Consumer 重渲染**（同一 Provider 之下）。  
- 常见误用：`<Provider value={{a,b}}>` 每次 render 新对象；把高频变化塞进单一大 Context。

**缓解策略**
1. **稳定引用**  
   - `useMemo` 包裹 `value`：`value={useMemo(()=>({a,b}),[a,b])}`  
   - 将**不变部分**与**高频部分**拆开不同 Provider。
2. **Selector 选择订阅**  
   - 使用 `use-context-selector` 等库：`useContextSelector(Ctx, s => s.part)` 仅当 `part` 变更时重渲染。  
3. **External Store**  
   - 对高频/全局数据，用 `useSyncExternalStore(subscribe, getSnapshot)` 直连外部 store（Zustand/Redux），避免 Context 扇出。  
4. **按域拆分 Provider**  
   - 主题、用户、权限、布局分别独立 provider；页面树中就近提供，缩小影响面。

**度量**
- React Profiler 看 “Why did this render?”；在 Consumer 处打印依赖字段变化，验证是否必要重渲染。

---

# 19) `useRef` 与 state 的职责边界；DOM 测量与避免布局抖动

**何时用 `useRef`**
- 存放**不会触发重渲染**的可变值：计时器句柄、上一次值、滚动位置、外部实例。  
- 获取 DOM 实例做**非渲染状态**操作：测量尺寸、焦点控制、播放控制等。

**何时用 state**
- 任何需要**驱动 UI**变化的值；需要参与 diff/渲染。

**测量模式**
- **回调 ref**：元素挂载更新即拿到节点，适合一次性测量。  
- **`useLayoutEffect` 测量**：在绘制前读取布局并同步写回，避免闪烁。  
- **ResizeObserver**：处理动态内容尺寸变化，缓存最新测量值，必要时再最小化更新 state。

```tsx
function Box() {
  const ref = useRef<HTMLDivElement>(null);
  const [rect, setRect] = useState<DOMRect | null>(null);
  useLayoutEffect(() => {
    const el = ref.current!;
    const ro = new ResizeObserver(() => setRect(el.getBoundingClientRect()));
    ro.observe(el);
    return () => ro.disconnect();
  }, []);
  return <div ref={ref} style={{height: 200}}>{rect?.width}</div>;
}
```

**避免抖动**
- 批量读写：先读后写，或使用 `requestAnimationFrame`。  
- 把**测量结果缓存到 ref**，仅当改变足以影响 UI 时再 setState。  
- 动画用 transform/opacity，避免触发布局。

---

# 20) 资源预加载策略：与 Suspense/数据的协作

**HTTP 层**
- 关键路径：  
  - `preconnect` 预连：`<link rel="preconnect" href="https://cdn.example.com" crossorigin>`  
  - `dns-prefetch` 预解析域名  
  - `preload` 高优先：`<link rel="preload" as="script/style/font/image" href="...">`（字体需 `crossorigin`）  
  - `prefetch` 低优先：为**下一步导航**提前拉资源
- 字体：`<link rel="preload" as="font" type="font/woff2" crossorigin>` + `font-display: swap`，减少 FOIT。

**React/Suspense 协作**
- 代码：`React.lazy` + Suspense；对首屏关键组件**提前 `preload`** chunk（框架通常内置）。  
- 数据：在 RSC/Next.js 中，**服务器聚合数据**避免“网络瀑布”；客户端可用 `use()` + 缓存。  
- 图片：首屏大图先占位（宽高比盒）+ 预加载；Next.js `Image priority`。

**避免重复与浪费**
- 预加载需**节制**：过多会阻塞关键资源；对“可能访问”用 `prefetch`。  
- 统一通过框架头部管理（`<Head>`/`app/head.tsx`），避免组件零散重复添加 `<link>`。

---

# 21) 代码分割：`React.lazy`/动态 import 的边界；路由级/组件级与错误边界/回退

**`React.lazy` 基本用法**
```tsx
const Editor = React.lazy(() => import('./Editor')); // 需要 default export
<Suspense fallback={<Spinner/>}>
  <Editor/>
</Suspense>
```
- 若无 default，可 `then` 转换：  
  `lazy(() => import('./Editor').then(m => ({ default: m.Editor })))`

**边界与注意**
- `lazy` 只解决**代码加载**，不解决**数据**；需要配合数据层（Suspense for data 或自家请求）。  
- **必须**被置于某个 Suspense 边界之下；否则加载期无 UI。  
- SSR：需使用框架（如 Next.js）或库（`@loadable/component`）做**服务端可见的 chunk 映射**与流式回退，以避免 hydration 问题。

**路由级分割**
- 路由切片最天然：每条路由一个动态 chunk；配合**预取下一跳**（鼠标悬停/视口内）。  
- 框架（Next.js/Vite + 路由）通常自动按路由/入口拆包。

**组件级分割**
- 体积大/低频使用的组件（富文本编辑器、图表、地图）单独拆。  
- 为这些组件配置**专属 ErrorBoundary** 与回退 UI（“加载失败，点击重试”）。

**工程化**
- 利用打包器提示（Bundle Analyzer）设定**体积预算**与拆分阈值；  
- 对关键路径启用 `preload`，对次要路径用 `prefetch`；  
- 与国际化/多主题等变体协作，避免把**所有语言包**打进首包（动态加载字典）。
# 22) Web Vitals（LCP/CLS/INP）与 React Profiler：定位重渲染与调度瓶颈

**指标速览**
- **LCP**（Largest Contentful Paint）：首屏最大元素出现时间；受服务器/网络/资源体积/阻塞脚本影响。
- **CLS**（Cumulative Layout Shift）：布局抖动；常因未预留尺寸（图片/广告/字体切换）。
- **INP**（Interaction to Next Paint）：交互到下一帧绘制时间；受主线程繁忙（长任务）/渲染开销影响。

**采集与监控**
- 生产上用 `web-vitals` 上报（支持 INP）：
```ts
import { onLCP, onCLS, onINP } from 'web-vitals';
[onLCP, onCLS, onINP].forEach(fn => fn(metric => send(metric)));
```

**React Profiler 用法**
- DevTools Profiler 记录渲染 flamegraph，定位**频繁重渲染**组件与“为什么重新渲染”（Why did this render）。
- 关注：
  - 渲染次数是否异常多（Props 引用不稳/Context 扇出）。
  - 单次渲染耗时是否高（昂贵计算/列表未虚拟化）。
  - 是否存在**同步布局读写**造成的长帧（`useLayoutEffect` 误用）。

**优化对策（与指标对应）**
- **LCP**：SSR/Streaming；关键资源 `preload`；图片响应式与占位盒；使用 RSC 将纯渲染逻辑移到服务器减少首包 JS。
- **CLS**：固定宽高比盒（`aspect-ratio`）；图片/广告按已知尺寸占位；避免首帧字体 FOUT/FOIT（`font-display` + preload）。
- **INP**：将昂贵计算移至 **Web Worker** 或**并发渲染下延**；拆分大更新为 `startTransition` 组；避免在事件中 setState 触发全树更新。

---

# 23) 安全性与 `dangerouslySetInnerHTML`：如何正确使用与防御 XSS

**默认安全**  
- React 默认对插值做 HTML 转义，**不执行**注入的 HTML/脚本。危险在于你**主动**使用 `dangerouslySetInnerHTML`。

**正确使用姿势**
1. 确保内容来自**受信任源**或经过**严格消毒**（如 DOMPurify、sanitize-html 且开启 URL/属性白名单）。
2. 内容为可控子集：白名单标签（`p, a, em, strong, ul, li, img`…）、白名单属性（`href, src, alt`），拒绝 `on*` 事件与 `javascript:` 协议。
3. 链接安全：外链 `rel="noopener noreferrer"`；同站点转内部路由。
4. 配置 **CSP**（Content-Security-Policy）：禁止 `unsafe-inline`/`eval`，使用 nonce/hash；现代站点应配合 **Trusted Types**（Chrome）。
5. 服务端也要过滤持久化内容，避免存储型 XSS。

**示例**
```tsx
import DOMPurify from 'dompurify';
function Rich({ html }: { html: string }) {
  const safe = DOMPurify.sanitize(html, { USE_PROFILES: { html: true } });
  return <div dangerouslySetInnerHTML={{ __html: safe }} />;
}
```

**其他风险点**
- SSR 注水（serialize）时应转义 `</script>` 与 `<!--`。
- URL 组装（开放重定向/协议注入）；文件上传的内容嗅探（`X-Content-Type-Options: nosniff`）。

---

# 24) CSS 策略对比：CSS Modules、Tailwind、vanilla-extract、CSS-in-JS（emotion/styled）

**CSS Modules**
- 优点：类名作用域隔离；零运行时；SSR 简单；与任意打包器兼容。
- 场景：中大型项目基线选择；配合原子化工具（PostCSS 插件）可控。

**Tailwind（原子类）**
- 优点：设计系统落地快、可树摇；减少样式命名认知负担。
- 注意：类名可读性与可维护性需通过抽象（`@apply`/组件化）平衡；与设计系统变量协作（`theme()`）。

**vanilla-extract（编译时 CSS-in-TS）**
- 优点：TypeScript 驱动的**静态** CSS，零运行时；主题与变量强类型。
- 场景：设计系统/多主题/大规模样式工程化。

**CSS-in-JS（emotion/styled-components 等）**
- 优点：动态样式、props 驱动；主题容易；生态完善。
- 注意：**运行时开销**（不过可用 Babel 插件、`@emotion/css` + 预编译降低）；SSR 要处理样式收集与注入顺序；与 React 19 `useInsertionEffect` 友好。

**选型建议**
- 以**零/低运行时**为基线（Modules/vanilla-extract/Tailwind），必要时在**极少数**需要动态样式的组件用 CSS-in-JS。
- 统一变量/设计令牌（Design Tokens）来源，样式方案仅是实现细节。

---

# 25) Portal 与层叠上下文：模态/抽屉的滚动锁定与焦点陷阱

**Portal 基本**
- `createPortal(children, container)` 将子树挂到 `container`（如 `document.body`），不改变 React 树关系但改变 DOM 层级。

**可达性（a11y）**
- 模态：`role="dialog" aria-modal="true"`；打开时**焦点移动**到对话框，关闭时**焦点回到触发器**。
- **焦点陷阱**：对话框内 `Tab` 循环；使用库（`focus-trap`）或自行实现。
- 背景**不可达**：对根内容加 `aria-hidden="true"` 或 `inert`（浏览器支持时）。

**滚动锁定**
- 打开模态时对 `body` 加 `overflow: hidden` 或使用 padding-right 补偿滚动条宽度；移动端需防止触摸穿透（被动滚动）。
- iOS 下避免页面“橡皮筋”滚动穿透：对遮罩阻断触摸滚动/在内容容器内创建独立滚动上下文。

**层叠上下文（z-index）**
- 避免“z-index 战争”：为弹层系统统一**分层约定**（backdrop → modal → popover → tooltip）。
- 使用新层叠上下文（`position: fixed` 或 `transform` 避免受祖先影响），Portal 至 `body`。

**代码骨架**
```tsx
return createPortal(
  <div role="dialog" aria-modal="true" aria-labelledby="title">
    <h2 id="title">Title</h2>
    <button onClick={onClose}>Close</button>
  </div>,
  document.body
);
```

---

# 26) 并发特性与数据请求：避免“网络瀑布”的系统性方案

**服务器端优先聚合**
- RSC/Next.js App Router：在**Server Component**中靠近数据源发起请求，**并行**获取并聚合，减少客户端级联请求。
- `fetch` 缓存与重用（Next.js）：相同请求自动去重；使用 `cache/revalidate` 策略与标签失效（`revalidateTag`）。

**客户端并发与避抖**
- 在客户端同时发起并行请求（不要 await 链式串行）；对请求结果使用 TanStack Query 去重与缓存。
- 输入驱动的请求使用**去抖**与**取消**（AbortController），避免落后响应覆盖最新输入。

**Suspense 协作**
- 对昂贵数据使用 Suspense 边界，允许页面**部分就绪**；streaming SSR 提前输出骨架。
- `useTransition` 将非紧急渲染延后，保证交互优先。

**预取/预加载**
- 路由级 `prefetch` 下个页面数据与代码；列表悬停预取详情。
- 静态内容（字典/配置）在服务器侧注入或在首屏请求中一并返回。

**失效/一致性**
- 统一失效策略：写操作（mutation）后**标记相关查询失效**并重取；或在 Server Action 中触发路径/标签 revalidate。

---

# 27) 测试策略：React Testing Library、MSW、计时器与端到端

**层次化**
- **单元**：纯函数/Hook（`@testing-library/react-hooks` 已并入主库能力或用自包装）。
- **组件**：React Testing Library（RTL）基于**用户视角**查询（`getByRole`/`getByText`）。
- **集成**：多组件/路由交互，结合 **MSW**（Mock Service Worker）模拟网络。
- **E2E**：Playwright/Cypress 真实浏览器验证关键路径（登录、下单）。

**关键实践**
- 避免快照滥用；仅对**稳定结构/协议**（例如 RSC payload 不适合）。
- 计时器：`vi.useFakeTimers()`/`jest.useFakeTimers()` 与 `act()` 配合推进时间。
- 异步等待：使用 `findBy*`/`waitFor`，避免随意 `setTimeout`。
- 可达性：使用 `getByRole` 并断言 `aria-*`、焦点行为，避免脆弱选择器。
- 覆盖网络：MSW 在**浏览器与 Node**两端一致工作，避免手写 `fetch` mock 的脆弱性。

**RSC/Next.js**
- Server Components 用**集成测试**验证产出与关键可见文本；Action 走**伪后端**或 Node 环境集成跑通。

---

# 28) 复杂流程中的状态机/状态图（XState 等）的收益与折中

**为什么要状态机**
- 复杂 UI 流程（多步表单、支付、播放器、实时协作）存在**并行/子状态**、**明确迁移条件**与**错误/超时**处理；状态机会把隐式分支变成**显式图谱**。
- **可视化/可测试**：模型独立于 UI，流程走查与单元测试自然。

**在 React 中使用**
```ts
import { useMachine } from '@xstate/react';
const machine = createMachine({
  id: 'checkout',
  initial: 'idle',
  states: {
    idle: { on: { SUBMIT: 'loading' } },
    loading: { invoke: { src: submit, onDone: 'success', onError: 'error' } },
    success: { on: { RESET: 'idle' } },
    error:   { on: { RETRY: 'loading' } }
  }
});
function Checkout() {
  const [state, send] = useMachine(machine);
  // 根据 state.value 渲染，不再 if/else 地狱
}
```

**收益**
- **确定性**：所有可能状态与迁移一览无余；  
- **副作用在边上**（`invoke`、`entry/exit`），UI 只消费状态；  
- **并行/分层**：可建模同时进行的子流程（上传+校验）。

**折中**
- 学习与建模成本；小型流程可能显繁重。  
- 与现有全局状态/请求库的边界要清晰（状态机关注**流程**，服务器状态仍交给 TQ/RTK Query）。

**推荐策略**
- 把“**要画图才能解释清楚**”的 UI 流程交给状态机；其余保持 Hooks + 局部 state 的简洁路线。
# 29) TanStack Query（TQ）：查询缓存、失效、变更与并发/Suspense协作

**为什么用 TQ**  
- “服务器状态”有**缓存、失效、过期、并发去重、重试**等需求，不适合用全局 store 粗暴管理。TQ 专精此事，UI 只消费**查询结果与状态**。

**核心概念**
- `queryKey`：查询身份（建议数组结构，如 `['todos', { page }]`）。  
- `staleTime` / `gcTime(cacheTime)`：新鲜期与垃圾回收时间。  
- `select`：对结果投影，减少重渲染。  
- 并发去重：同一 `queryKey` 的并发请求自动合并。  
- 依赖查询：`enabled` 控制何时发起（如等待 token）。

**失效与重取**
```ts
const qc = useQueryClient();
qc.invalidateQueries({ queryKey: ['todos'] }); // 标记过期，触发重取（下次聚焦/网络恢复/手动）
qc.setQueryData(['todo', id], updater); // 乐观更新/本地修正
```

**变更（Mutation）与乐观更新**
```ts
const m = useMutation(updateTodo, {
  onMutate: async (patch) => {
    await qc.cancelQueries({ queryKey: ['todo', patch.id] });
    const prev = qc.getQueryData(['todo', patch.id]);
    qc.setQueryData(['todo', patch.id], (old:any) => ({ ...old, ...patch }));
    return { prev };
  },
  onError: (_e, patch, ctx) => ctx && qc.setQueryData(['todo', patch.id], ctx.prev),
  onSettled: (_d, _e, patch) => qc.invalidateQueries({ queryKey: ['todo', patch.id] })
});
```

**与 Suspense/并发协作**
- TQ 可启用 `suspense: true`，未就绪时**抛出 Promise**交给 `Suspense`；用 `useTransition` 包裹非紧急刷新。  
- Next.js：`dehydrate`（服务端）+ `Hydrate`（客户端）实现**SSR 注水**与无缝水合。

---

# 30) 客户端存储与水合：`localStorage/sessionStorage` 的读取时机与一致性

**SSR/CSR 不一致的根源**
- SSR 阶段没有 `window`/`localStorage`；若在**渲染期间**直接读取，会导致**水合不匹配**或报错。

**安全模式**
- **首渲染不依赖存储**：先用确定的 UI（占位/默认值），在 `useEffect` 里异步读取并 `setState`。
```ts
const [theme, setTheme] = useState<'light'|'dark'>('light');
useEffect(() => {
  const t = localStorage.getItem('theme') as 'light'|'dark'|null;
  if (t) setTheme(t);
}, []);
```
- 如需用**懒初始化**：确保**仅在客户端**执行（框架 `dynamic({ ssr:false })` 或在 effect 中设置）。
- 与 Cookie 区分：**需要 SSR 参与的偏好**（如 A/B、区域）用 Cookie 注入 SSR；**纯客户端偏好**用 Storage。

**同步策略**
- 写入节流与版本化（`v1:...`）；迁移旧 key 时保证幂等。  
- 加密/签名：对敏感数据不要存 Storage；最少也需签名校验与过期控制。

---

# 31) `useId` 的用途、SSR 一致性与可达性场景

**作用**
- 生成**稳定、SSR/CSR 一致**的唯一 id，适合无障碍**关联**与表单**label-for** 场景。

**用法与注意**
```tsx
function Field() {
  const id = useId();           // e.g. :r0:-:r3:
  return (
    <div>
      <label htmlFor={id}>Name</label>
      <input id={id} />
    </div>
  );
}
```
- **不要**把 `useId` 用作列表 `key`（key 需来源于数据身份）。  
- 同一个组件多处需要 id 可 `const base = useId()` 后拼接 `\`${base}-desc\`` 等。  
- 与 SSR：相比自生成随机 id，`useId` 能确保**水合一致**，避免 mismatch。

---

# 32) 渲染大型表格/数据网格：虚拟化、不可变数据与 `areEqual`

**关键策略**
- **窗口化/虚拟化**：`react-window`/`react-virtual`；固定行高最稳，变高需测量+缓存。  
- **不可变数据**：分页/筛选/排序生成**新引用**，便于 `memo` 判等。  
- **行/单元格 memo**：`React.memo(Row, areEqual)`，对 props 做**浅比较/关键字段比较**。
```tsx
const Row = React.memo(({ row, onClick }: {row: RowT; onClick: (id:string)=>void}) => {
  return <div onClick={() => onClick(row.id)}>{row.name}</div>;
}, (prev, next) => prev.row === next.row && prev.onClick === next.onClick);
```
- **稳定引用**：把 `onClick`、`columns` 用 `useCallback/useMemo` 固定；避免在 render 构造临时对象。
- **占位与测量**：图片/异步内容预留尺寸；避免行高抖动导致滚动跳跃。  
- **交互与选择**：把“选中集”等状态放到外部 store（Zustand）或上层，避免每行 setState 触发 N 次重渲染。  
- **SSR**：首屏可仅渲染可见行 + 骨架，水合后启用虚拟化逻辑。

---

# 33) 媒体组件的“受控 vs 非受控”设计；`forwardRef`/`useImperativeHandle`

**选择原则**
- **非受控**（推荐默认）：内部用 `ref` 控制 `<video>`/`<audio>`，父组件仅下发初始属性；交互通过暴露的**命令式 API** 完成。  
- **受控**：将 `currentTime/playing` 等抽象为 state，由父组件驱动；适合需要**同步多个播放器**或可回放调试，但重渲染压力更大。

**API 设计**
```tsx
type PlayerHandle = { play():void; pause():void; seek(s:number):void };
const Player = React.forwardRef<PlayerHandle, { src:string }>((p, ref) => {
  const el = useRef<HTMLVideoElement>(null);
  useImperativeHandle(ref, () => ({
    play:  () => el.current?.play(),
    pause: () => el.current?.pause(),
    seek:  (s) => { if (el.current) el.current.currentTime = s; }
  }), []);
  return <video ref={el} src={p.src} controls />;
});
// 父组件
const ref = useRef<PlayerHandle>(null);
<button onClick={() => ref.current?.seek(30)}>跳到 30s</button>
<Player ref={ref} src="/v.mp4" />
```

**细节**
- 事件去抖（`timeupdate` 高频）；全屏/画中画需浏览器特性分支。  
- 移动端自动播放限制与静音策略；错误恢复与重试。  
- 避免将频繁变化（进度）提升为受控 props，改为**订阅/通知**模型。

---

# 34) 构建与工具链（Vite）：别名/环境变量、分包与产物分析基线

**为何 Vite**
- Dev 使用 **esbuild** 预构建 + 原生 ESM，冷/热启动极快；产物用 **Rollup**，生态成熟。

**常用配置**
```ts
// vite.config.ts
import react from '@vitejs/plugin-react';
export default defineConfig({
  plugins: [react()],
  resolve: { alias: { '@': '/src' } },
  define: { __BUILD_TIME__: JSON.stringify(new Date().toISOString()) },
  build: {
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          react: ['react', 'react-dom'],
          vendor: ['lodash-es', 'dayjs']
        }
      }
    }
  }
});
```
- 环境变量：在 `.env`/`.env.production` 中以 `VITE_` 前缀暴露（`import.meta.env.VITE_API_BASE`）。  
- CSS 与静态资源：开启 `cssCodeSplit`，大图走 CDN 与 `assetInlineLimit` 调整。

**产物分析与基线**
- `rollup-plugin-visualizer` 或 `vite-bundle-visualizer` 输出 treemap；建立**体积预算**（如首包 < 200 KB gzip）。  
- 打开 `sourcemap` 便于错误上报与回溯（生产上可用外链上报而不随包发布）。

**路由级分包与预取**
- 动态 `import()` 按路由切片；对下一跳 `prefetch`（框架内置或自加 `<link rel="prefetch">`）。

**CI**
- 缓存 `node_modules` 与 `~/.pnpm-store`；构建结束产出**构建清单**（chunk 名称/体积）与**阈值报警**。

---

# 35) 生产工程基线：Source Map、环境/密钥、性能护栏与回滚

**Source Map 策略**
- 生产开启 **hidden**/**external** sourcemap 并上传到错误平台（Sentry 等），包内不暴露。  
- 版本与构建号打到 `release`，保证 crash 可回溯。

**环境与密钥**
- 公共配置走 `VITE_*`；**切勿**把私钥/长 token 填进前端。  
- 通过后端下发**短期签名**或使用 Edge Functions 代理敏感请求。

**性能护栏**
- LCP/CLS/INP 在线采集 + 阈值告警；  
- Bundle 体积阈值（gzip/brotli）在 CI 审查；  
- 关键页面开启**预渲染/SSR/Streaming**；图像/字体 `preload`。

**回滚与灰度**
- 版本号与静态资源 hash 强绑定；CDN 上**多版本共存**；  
- 通过环境开关/远端配置开关危险特性；异常时**快速回滚**到上一稳定构建。

**安全**
- CSP、SRI（子资源完整性）与 `X-Frame-Options`/`COOP/COEP`；  
- 依赖审计（`pnpm audit`/`npm audit` + SCA），锁定版本并定期刷新。

---

# 35) 生产工程基线：Source Map、环境/密钥、性能护栏与灰度回滚

**Source Map 策略**
- 目标：便于**定位线上错误**，同时**不暴露源码**。
- 做法：
  - 构建产出 **hidden/external** sourcemap（包内不内联）；仅上传到错误平台（Sentry 等）。
  - 以 **release 版本号 + commit hash** 作为唯一标识，配合符号表上传。
  - 对第三方包生成 `vendor` map 以便反解。
- Vite 示例：
  ```ts
  // vite.config.ts
  export default { build: { sourcemap: true } } // external map
  ```
- Webpack 示例：
  ```js
  // webpack.prod.js
  module.exports = { devtool: 'hidden-source-map' };
  ```

**环境与密钥**
- 约束：任何**私密信息**（API key、令牌）不得进入前端包。
- 做法：
  - 公开变量走前缀（Vite 的 `VITE_*`）+ 构建时注入；**敏感变量只在后端**或边缘函数上读取。
  - 对“需要签名”的场景使用**短期签名**（STS/STS-like）或代理。
  - 统一 `.env` 规范与**环境矩阵**（dev/staging/prod），CI 通过密钥柜（Secrets Manager）下发。

**性能护栏**
- **体积预算**：对 `initial`/`async` chunk 设置阈值（gzip/br）并在 CI 拦截。
  - Vite（Rollup）示例：使用 bundle 分析插件并自建脚本检查阈值。
- **Web Vitals**：LCP/CLS/INP **在线采集** + 告警；对关键页面设置 SLO，并将“超过阈值的比例”作为发布门禁。
- **图片/字体**：图像转码（AVIF/WebP）、`<link rel="preload">` 关键字体 + `font-display: swap`。
- **缓存与 CDN**：静态资源 `immutable` + 文件名哈希；HTML `no-store`；多区域回源与健康检查。

**灰度与回滚**
- **灰度发布**（canary）：按用户/地域/流量百分比逐步放量；配合远端**特性开关**（Feature Flag）。
- **快速回滚**：CDN 多版本共存，切换 `release` 指针即可；配合 sourcemap 和错误平台做到“版本→错误回溯→回滚”闭环。
- **数据/配置分离**：把不确定的策略放到远端配置，允许线上**热修**而不用重发包。

---

# 36) Fiber 架构：关键字段与双缓冲树

**核心对象：`FiberNode`**
- 重要字段：  
  - `tag`（节点类型：FunctionComponent/HostComponent 等）  
  - `type`（具体组件/DOM 类型）  
  - `key`（复用与重排依据）  
  - `pendingProps` / `memoizedProps`（新/旧 props）  
  - `memoizedState`（Hook 链表等状态）  
  - `flags` / `subtreeFlags`（提交期需要执行的副作用位）  
  - `lanes` / `childLanes`（优先级/待处理更新集合）  
  - `return`/`child`/`sibling`（树结构）  
  - `alternate`（指向另一棵树的对应节点）
  
**双缓冲树（current / workInProgress）**
- **current**：正在屏幕上显示的那棵 Fiber 树。
- **workInProgress（WIP）**：本次渲染要构建/对比的树；渲染完成后**整棵树指针切换**，实现原子替换。
- 好处：可在并发模式下**中断/恢复**构建，不污染 current。

**副作用收集**
- 渲染阶段在每个 Fiber 节点上打 `flags`；提交阶段根据 `flags/subtreeFlags` 扫描执行（插入/更新/删除、布局/被动副作用等）。

---

# 37) 工作循环：`render` / `commit` 两阶段与关键函数

**渲染阶段（可中断）**
- 入口：`workLoopConcurrent`（并发）或 `workLoopSync`（同步）。
- 主要函数：
  - `beginWork(fiber)`：根据新输入（props/state/context）计算子树，返回下一个要处理的子节点。
  - `completeWork(fiber)`：当前节点收尾（生成 Host 节点更新、收集副作用），回到兄弟或父节点。
- 产出：一棵打好 `flags` 的 **WIP 树**。

**提交阶段（不可中断）**
- 分三小步：
  1) **Before Mutation**：调用 `getSnapshotBeforeUpdate` 等。
  2) **Mutation**：根据 `Placement/Update/Deletion` 修改真实宿主环境（DOM）。
  3) **Layout**：`useLayoutEffect`/class 组件 `componentDidMount/Update`；随后调度 `useEffect`（被动）在微/宏任务后执行。
- 结果：`current` ↔ `workInProgress` 交换；屏幕显示新 UI。

---

# 38) 优先级模型：Scheduler 与 Lanes

**两层模型**
1) **Scheduler 优先级**：调度任务粒度（立即、用户、延迟等），决定**何时**切片执行。
2) **Lane（车道）**：渲染优先级与合并规则，决定**先渲染谁**、哪些更新可以批处理。

**常见映射**
- **离散事件**（点击、键盘）：高优先 lane（尽快响应）。
- **连续事件**（滚动、拖动）：次高优先。
- **Transition**（`startTransition` 标记的非紧急更新）：较低优先，允许让路。
- **Idle**：最低优先，空闲再处理。

**合并与饥饿**
- 同一 Fiber 的多次更新会合并到其 `lanes`；渲染会选择**最高优先的 lane 集**处理，避免低优更新饿死会通过**老化**策略提升优先级。

---

# 39) 更新队列：函数组件 Hook vs 类组件

**函数组件（Hook）**
- 每个 FunctionComponent Fiber 在 `memoizedState` 上维护一条**Hook 链表**。
- `useState`/`useReducer` 有各自的**更新环形队列**：保存 `action`（或 reducer + payload）。
- 渲染时从 `baseState + baseQueue` 依次计算新 state；并将**未消费的更新**串回新队列，保证在中断/恢复后不丢失。
- **eager state**：在触发更新时尝试用上次 reducer/状态直接计算新值，若**不变**可**短路**跳过调度。

**类组件**
- `setState` 维护**链式更新**（可能是函数或部分状态对象），渲染时依次 reduce；`this.state` 只在渲染计算后更新。
- 合并策略与批处理由调度层统一处理。

**要点**
- 并发下**同一更新可能被计算多次**（重做），因此 reducer/更新函数必须**纯**且可重入。

---

# 40) Hook 原理：Dispatcher 与“同序调用”规则

**为什么 Hook 要“顶层调用、同序执行”**
- 运行时通过一个**指针**（current Hook）在 Fiber 上的 Hook 链表中**按顺序**取值/写值。
- 如果在条件/循环内调用，会导致**调用序列改变**，与上次渲染的 Hook 链表**对不上**，状态错位。

**Dispatcher 机制**
- 渲染开始时设置当前 `Dispatcher`（mount 或 update 版本）：
  - `mountState`/`updateState`、`mountEffect`/`updateEffect` … 成对实现。
- 每次调用 `useXxx` 都会**创建/读取**对应的 Hook 节点，移动指针到下一个。

**副作用分类**
- `useLayoutEffect`：提交阶段 **layout** 步骤同步执行（阻塞绘制后、浏览器绘制前）。
- `useEffect`：**被动**，在 commit 结束后调度执行（不阻塞首帧）。

---

# 41) Context 传播：依赖跟踪与选择性更新

**读取与订阅**
- `useContext(MyContext)` 在渲染时对当前 Fiber 记录“我依赖了 `MyContext` 的值”，相当于订阅。
- 当 `Provider` 的 `value` 变更时，React 根据订阅关系**标记**受影响的子树需要更新。

**为什么 Context 容易“全树重渲染”**
- Provider 下**所有消费方**默认会重渲染；如果 `value` 引用频繁变、或包含大对象，很容易放大重渲染面。

**缓解策略（源码视角的友好用法）**
- 保持 `value` 的**稳定引用**（`useMemo`）。
- 拆分多个 Provider，按域提供（主题/权限/用户信息分开）。
- 对热点数据**不要强行用 Context**，而是改用外部 store（`useSyncExternalStore` 模式或 Zustand/Redux），由 store 做**选择性订阅**。

---

# 42) 子节点 Diff（Reconciliation）：`key` 的作用与策略

**规则速览**
- **同类型 + 同 key** → 复用节点（更新 props）。
- **同类型 + 不同 key** 或 **不同类型** → 视作新节点（卸载旧、挂载新）。
- 列表更新采用**单次线性**算法：先按位置对比，必要时构建 key→旧节点的映射以支持**移动**。

**实践要点**
- **稳定 key** 来自业务 id；不要使用索引，尤其是**可增删/重排**的列表。
- 分页/虚拟滚动场景，key 应该是**数据 id**，而非“可见区行号”。

**拖拽示例（保持 key 稳定，只变顺序）**
```tsx
{items.map(item => (
  <Row key={item.id} data={item} />
))}
// onDragEnd: setItems(reorder(items, srcId, dstId));
```

---

# 43) Flags 与副作用：收集与提交

**`flags` / `subtreeFlags`**
- 每个 Fiber 在渲染阶段根据变化打上标志位：  
  - **Mutation** 类：`Placement`（插入）、`Update`（属性变更）、`Deletion`（删除）。  
  - **Passive**：`useEffect` 依赖变更/清理。  
  - **Ref**：`Ref` 附着/变更。  
- 父节点 `subtreeFlags` 聚合子树标志，提交阶段以此**剪枝扫描**（相较早期 Effect List 方案）。

**提交阶段顺序**
1) **Before Mutation**：快照（`getSnapshotBeforeUpdate`）。  
2) **Mutation**：根据 `Placement/Update/Deletion` 操作宿主环境（DOM）。  
3) **Layout**：`useLayoutEffect`/类组件布局副作用；随后调度 **Passive** 效果在微/宏任务中执行（不阻塞绘制）。

**关键差异**
- `useLayoutEffect` 同步、可读取布局并立即写回；  
- `useEffect` 异步、不会阻塞绘制，更适合订阅/日志/网络等非布局副作用。

**排错建议**
- 如果出现“闪烁/抖动”，检查是否误用 `useLayoutEffect` 或在 layout 阶段做了昂贵计算。
- 使用 Profiler/开发者工具查看**为什么渲染**与 Fiber flags，定位多余的 `Update/Placement`。

```tsx
// 典型错误：在渲染期间创建不稳定的回调/对象，导致子树每次都是 Update
const handle = () => doSomething(); // ❌ 每次渲染新引用
const handle = useCallback(() => doSomething(), []); // ✅ 保持引用稳定
```
---

# 44) Suspense 实现原理：thenable、挂起与重试、fallback/reveal-order

**挂起（suspend）如何发生？**  
- 在渲染阶段，当组件**读取尚未就绪的数据**（RSC `use()` / 客户端 `use(promise)` / 第三方库以“抛出 thenable”协议暴露）时，组件会**抛出一个 thenable**。  
- 最近的 Suspense 边界捕获该 thenable，把它登记到“**wakeables**”列表，并**订阅其完成**（`then(ping)`）。

**fallback 的展示与避免闪烁**  
- 若边界下的子树挂起且没有可复用的旧 UI，边界渲染 `fallback`。  
- 在 **transition** 场景（`startTransition` 包裹）下，React 会**尽量保留上一次的已完成 UI**（即“粘住旧界面”），避免立即显示 fallback 闪烁，待数据就绪再无缝切换。

**重试与揭示（reveal）**  
- thenable 解决后触发 **ping**，调度对应 lane 的**重试渲染**；边界重新渲染成功即“揭示”真实 UI。  
- `SuspenseList` 提供 **reveal-order**（`together/forwards/backwards`）控制多边界揭示顺序，兼顾感知速度与布局稳定。

**SSR/Streaming 协作**  
- 流式 SSR：服务器先输出外层 shell 与 fallback，数据就绪的边界以**分块**持续写出；客户端**选择性水合**逐步接管。  
- 错误与加载要分离：加载去 Suspense，异常交给 **Error Boundary**。

---

# 45) 并发渲染中的中断与恢复：让路、丢弃与可重入

**为什么能中断？**  
- 渲染阶段是**可分段的**，React 在 `workLoopConcurrent` 中周期性调用 `shouldYield()`（由 Scheduler 决定），以让出主线程给更高优先任务（输入/动画）。

**中断后的恢复**  
- 若无更高优先任务，仅是时间片用尽，会在下一个时间片**继续从上次的 Fiber** 处恢复。  
- 若有**更高优先 lane** 插入（例如点击事件），当前 WIP 可能被**丢弃重做**，以高优先任务为先。  
- 为保证正确性，**render 必须纯且可重入**：同一次更新可能被计算多次。

**实践要点**  
- **不要**在 render 阶段读取/写入外部可变单例（如全局缓存可被并发读写）；必要时把读取推迟到 `useEffect`。  
- 需要抢占式体验的更新放进 `startTransition`，输入响应保持同步优先。  
- 尽量使用 **不可变数据 + 纯函数**，减少中断后的重复工作量。

---

# 46) `useTransition` / `useDeferredValue` 的内部语义与差异

**`useTransition()`**  
- `startTransition(fn)` 将 `fn` 中的更新**标记为 transition lanes**（低于离散/连续事件的优先级）。  
- 这类更新允许**被打断与延后**，React 会尽量**保留上次已完成的 UI**，直到新 UI 完成再无缝切换。  
- `isPending` 来源于内部对**仍在进行的 transition lanes**的跟踪：当这些 lanes 在根上清空后，`isPending` 才变为 `false`。

**`useDeferredValue(value)`**  
- 返回 `value` 的**延迟版本**：当上游 value 频繁变化时，下游以**较低优先**（通常也是 transition lanes）吸收变化，减少昂贵子树的抖动。  
- 实现思路是 Hook 在比较当前 `value` 与上次“已提交的延迟值”后，必要时以较低优先**安排一次更新**，从而“滞后”消费。

**对比**  
- 作用点：`useTransition` 是**更新发起方**的语义；`useDeferredValue` 是**值消费方**的语义。  
- 粒度：`useTransition` 可把**一组更新**降级；`useDeferredValue` 只影响**这一份值**的传递。  
- 组合：输入端用 `startTransition`，展示端用 `useDeferredValue` 稳定昂贵子树，是常见双保险。

---

# 47) Hydration 算法：匹配、选择性水合与 mismatch 处理

**匹配流程**  
- `hydrateRoot(container, element)` 绑定到已有 SSR DOM。  
- 渲染时，DOM 渲染器通过“**声明式匹配**”尝试把 Fiber 对齐到已有节点：  
  - 元素类型/属性匹配 → 复用并打补丁；  
  - 不匹配 → **切换为客户端渲染**该子树（丢弃该处服务端标记）。

**选择性水合**  
- React 在根容器上委托事件；当某个**离散事件**抵达未水合子树时，会**优先水合该边界**，保证交互不被阻塞。  
- 流式 SSR 配合 Suspense：可先水合外层 shell，再按交互/数据就绪逐步水合内部。

**mismatch 处理与规避**  
- 典型原因：随机数/时间、环境差异、客户端专属逻辑。  
- 规避：  
  - 把仅客户端逻辑放到 `useEffect`；  
  - 使用 `useId()` 保证一致 ID；  
  - 对少量可控差异使用 `suppressHydrationWarning`（谨慎）。  
- 一旦 mismatch，React 会在该处放弃复用，转为**客户端重新创建**节点，代价是多一次 DOM 变更与可能的闪烁。

---

# 48) 事件系统（React 17+）：根容器委托、事件优先级与合成事件边界

**委托位置变化**  
- 17 之前多委托在 `document`，17 起**委托在根容器**，便于多版本/多应用并存，Portal 行为更贴近原生。

**事件优先级**  
- React 为事件赋予**调度优先级**（离散 > 连续 > 默认），影响更新归入的 lanes，从而决定是否可被中断/让路。

**合成事件与原生事件**  
- 合成事件（`SyntheticEvent`）提供跨浏览器统一 API；17 起**不再事件池化**，异步访问属性安全。  
- 某些事件（如 `scroll`）走原生监听，**不经过合成层**；`stopPropagation` 仅影响 React 树内传播。

**工程影响**  
- 与非 React 区域共存更安全（微前端/多根容器）。  
- 高频场景建议使用被动监听或避免在事件中做重计算；将昂贵更新迁移到 **transition** 或异步工作。

---

# 49) RSC / Flight 协议：服务器组件负载与客户端拼装

**核心模型**  
- **Server Components**（不含事件/浏览器 API）在服务器执行，直接**靠近数据源取数**并输出 **Flight payload**（一种可序列化的组件树描述 + 模块引用）。  
- 客户端 runtime 解码 payload，并在 **Client Components** 边界处**拼装与水合**，仅对需要交互的部分下发 JS。

**优势**  
- 显著减少**客户端 JS 体积与水合成本**；避免客户端“网络瀑布”，把聚合/拼接放在服务端完成。  
- 与 **Server Actions**/缓存 配合，可实现端到端的数据一致与失效控制。

**限制与约束**  
- Server 组件不能使用浏览器 API/状态 Hook；只能向下传**可序列化的 props**。  
- Client 组件需以 `'use client'` 显式声明，事件/状态在客户端处理。  
- 需要框架级支持（如 Next.js 的 `react-server-dom-webpack` 管线）。

---

# 50) 渲染器 Host Config：复用调和器的“平台适配层”

**调和器与渲染器分层**  
- React **Reconciler** 负责构建/对比 Fiber 树与调度；具体“如何操作宿主环境”由各渲染器的 **Host Config** 决定。  
- DOM 渲染器（react-dom）、原生渲染器（react-native）等共享调和器，差异在于 Host Config 的实现。

**Host Config 的关键接口（节选）**  
- 创建/更新：`createInstance` / `finalizeInitialChildren` / `prepareUpdate` / `commitUpdate`  
- 树操作：`appendChild` / `insertBefore` / `removeChild`  
- 文本节点：`createTextInstance` / `commitTextUpdate`  
- 效果支持：`supportsMutation` / `supportsPersistence` / `now` / `scheduleTimeout` …

**工程意义**  
- 新平台（Canvas/WebGPU/终端 UI）可通过实现 Host Config 得到 React 的**调和 + 并发能力**。  
- 需保证这些接口是**幂等、可重入**的，以适配并发渲染与中断/恢复；并正确处理**布局/测量**时机，避免闪烁与抖动。

```
::contentReference[oaicite:0]{index=0}
