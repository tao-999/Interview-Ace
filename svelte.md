# Svelte 高频面试题
---

## 1) “编译型框架”的本质，与 React/Vue 的运行时范式差异

**本质**：Svelte 在构建时将 `.svelte` 源码编译为**直接操作 DOM 的命令式更新代码**，并静态确定依赖关系与更新顺序，消除虚拟 DOM（VDOM）与运行时 diff 的需要。

**与 React/Vue 的根本区别**
- **更新决策位置**：
  - Svelte：**编译期**静态提取依赖与更新路径 → 运行时只做最小必要的 DOM 变更。
  - React/Vue：保留**运行时**（React Fiber/Hook，Vue Proxy+VDOM），通过 diff/effect 在运行时决定如何更新。
- **运行时代码体积**：
  - Svelte：组件可被树摇，接近零运行时；只打包组件所需的最小更新函数。
  - React/Vue：必须携带核心运行时和调度逻辑。
- **心智模型**：
  - Svelte：更接近“**变量即状态，赋值即更新**”；无需 setState/useState。
  - React/Vue：通过 setState/hooks 或 reactive/ref 与依赖追踪机制来驱动更新。

**简化的编译结果示意**（概念示例）：
```svelte
<!-- Counter.svelte -->
<script>
  let count = 0;
  function inc(){ count += 1 }
</script>
<button on:click={inc}>{count}</button>
```

编译后（伪代码）：
```js
function create_fragment(target) {
  const button = document.createElement('button');
  const text = document.createTextNode(count);
  button.appendChild(text);
  button.addEventListener('click', inc);
  target.appendChild(button);

  return {
    p(ctx) {             // precise update
      if (ctx.count_changed) text.data = ctx.count;
    },
    d() { /* detach */ }
  }
}
```

---

## 2) Svelte 4 的响应式：`$:` 与“赋值触发更新”的原理与局限

**核心机制**
- **赋值触发**：对脚本作用域中的变量进行赋值（`x = ...`、`x += 1`）会将该变量标记为脏，驱动依赖该变量的模板片段和 `$:` 声明重新计算。
- **`$:` 响应式声明**：编译期对 `$:` 右侧表达式进行**依赖收集**，当任一依赖变更时，按拓扑顺序重跑该语句。

示例：
```svelte
<script>
  let count = 0;
  $: doubled = count * 2;      // 依赖 count
  function inc(){ count += 1 } // 赋值触发
</script>

<button on:click={inc}>{doubled}</button>
```

**局限与常见坑**
- **深层变更不触发**：以下不会触发更新，因为引用未变：
  ```js
  obj.items.push(x);     // ❌ 不触发
  map.set(k, v);         // ❌ 不触发
  set.add(v);            // ❌ 不触发
  ```
  需改变引用或显式“触发”：
  ```js
  obj = {...obj};        // ✅ 触发
  arr = arr.slice();     // ✅ 触发
  map = new Map(map);    // ✅ 触发
  set = new Set(set);    // ✅ 触发
  ```
- **`$:` 的两种形态**：
  - 赋值式：`$: a = f(b, c)` → 适合纯计算、无副作用。
  - 语句块：`$: { doSomething(); ... }` → 易夹杂副作用；要防重复与时序问题。
- **避免在 `$:` 中做重型副作用**（如网络请求、测量布局反复触发）；这类逻辑更适合 `onMount/afterUpdate` 或（Svelte 5）`$effect`。
- **模块上下文不响应**：`<script context="module">` 中的代码与值**不参与**组件响应式系统。

---

## 3) Svelte 5（Runes）：`$state / $derived / $effect` 的变化与迁移要点

**目标**：让响应式关系显式化、可深度追踪，减少 “必须改引用才能触发” 的心智负担。

基本用法：
```svelte
<script>
  // 局部启用：<svelte:options runes={true} />
  let count   = $state(0);                    // 状态（对象/数组可深度追踪）
  let doubled = $derived(() => count * 2);    // 只读派生
  $effect(() => { console.log('count:', count) }); // 副作用，依赖自动收集

  function inc(){ count += 1 }
</script>

<button on:click={inc}>{doubled}</button>
```

**关键变化**
- `$state`：对**对象/数组**的原地变更也会触发更新（通过 Proxy 深度追踪），不再必须“换引用”。
- `$derived`：明确表达“只读派生值”，替代大量 `$:` 赋值式计算。
- `$effect`：显式副作用边界，依赖自动收集，语义清晰，替代 `$:` 语句块中的副作用用法。
- **渐进迁移**：可在组件或项目级分别开启 `runes`；逐步将 `$:` 计算改为 `$derived`，副作用改为 `$effect`，需要深追踪的状态改为 `$state`。

**迁移建议（对照表）**
| Svelte 4 | Svelte 5（Runes） |
|---|---|
| `$: a = f(b, c)`（纯计算） | `let a = $derived(() => f(b, c))` |
| `$: { sideEffect(); }` | `$effect(() => { sideEffect(); })` |
| `arr.push(x); arr = arr;` | `let arr = $state([]); arr.push(x);`（无需换引用） |
| 本地状态 `let x = 0` | `let x = $state(0)`（若需深追踪或可变封装） |

> 注：Runes 还引入了与 props 双向绑定、安全赋值等相关符号（如 `$bindable` 等），此处聚焦状态/派生/副作用三件套。

---

## 4) 组件通信：`props`、`createEventDispatcher`、`on:*` 的对比与边界

**Props 传参**
```svelte
<!-- Child.svelte -->
<script>
  export let value = 0;   // 定义 props
</script>
<p>{value}</p>

<!-- Parent.svelte -->
<Child value={42} />
```

**自定义事件（子→父）**
```svelte
<!-- Child.svelte -->
<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher<{ change: number }>();
  export let value = 0;
  function onClick(){ value += 1; dispatch('change', value); }
</script>
<button on:click={onClick}>{value}</button>

<!-- Parent.svelte -->
<Child on:change={(e) => console.log(e.detail)} />
```

**原生事件转发 `on:*`（常用于包装原生元素的“透传”组件）**
```svelte
<!-- InputWrapper.svelte：把所有原生事件从内部 <input> 转发给包装组件 -->
<script>
  export let value = '';
</script>
<input bind:value on:* />
```
- 适用：封装原生控件时避免枚举 `on:input`、`on:blur` 等事件，一个 `on:*` 即可全部转发。
- 边界：`on:*` 是对**内部元素**事件的转发；组件间通信仍应通过 `dispatch` 语义化事件名与负载。

**补充：组件双向绑定**
- Svelte 4：`<Child bind:value />` 需要 Child `export let value` 且**在子组件内直接赋值** `value = newVal` 才能向上同步。
- Svelte 5：更倾向显式能力（如 `$bindable`），以避免子组件随意修改父级数据的隐式副作用。

---

## 5) Stores：`writable/readable/derived` 使用场景、订阅/解除订阅与内存泄漏防护

**API 定位**
- `writable<T>(initial)`：可读可写的全局/跨组件状态。
- `readable<T>(initial, start)`：只读 store，`start(set)` 可建立副作用（轮询、订阅），其返回的函数在**最后一个订阅者取消**时被调用以清理。
- `derived<S, T>(stores, fn, initial?)`：从一个或多个源 store 派生出只读 store。

示例（持久化与解订阅）：
```ts
// store.ts
import { writable, readable, derived } from 'svelte/store';

export const count = writable(0);

// readable：最后一个订阅者取消订阅时，stop 被调用
export const now = readable(Date.now(), (set) => {
  const t = setInterval(() => set(Date.now()), 1000);
  return () => clearInterval(t);     // 清理避免内存泄漏
});

export const doubled = derived(count, ($count) => $count * 2);
```

**组件中订阅的正确姿势**
- 在模板中使用 `$store`（自动订阅、自动解订阅），**无需手写 unsubscribe**。
- 在脚本中手动订阅时，务必在 `onDestroy` 中清理：
  ```svelte
  <script>
    import { onDestroy } from 'svelte';
    import { count } from './store';
    const unsub = count.subscribe(v => console.log(v));
    onDestroy(unsub);  // 防泄漏
  </script>
  ```

**防止泄漏与竞态的要点**
- 所有 `readable` 的 `start` 必须返回清理函数。
- 组件手订阅必须 `onDestroy` 清理。
- 防止派生链路里创建新的订阅引用（容易形成“悬空订阅”），尽量通过 `derived` 聚合，而不是在组件里层层 `subscribe` 再 `set`。

---

## 6) `get(store)` 同步读取的时机与风险；何时避免

`get`（来自 `svelte/store`）会**同步**读取当前值，**不会建立订阅**，因此不会随后续变化自动更新。

**适用场景**
- 单次同步读取（如执行某个一次性逻辑、初始化计算）。
- SSR/服务端路径中需要即时取值，不希望建立订阅。

**风险与不当用法**
- 在响应式语境（`$:` 或事件回调）中频繁使用 `get` 可能造成**读取的值与 UI 更新时序不一致**，且易忽略后续变化。
- 若需要追踪变化，应通过 `$store` 或显式 `subscribe`/`derived`，而非反复 `get`。
- 与异步流程耦合时（如读取后立刻依赖 DOM），需配合 `tick()` 或生命周期钩子保证时序。

---

## 7) 生命周期：`onMount / beforeUpdate / afterUpdate / onDestroy` 触发时机与典型用法

- `onMount(fn)`：组件首次挂载到 DOM 后执行（仅在浏览器运行，**SSR 不会执行**）。适合发起网络请求、访问 `window/document`、注册订阅等。
- `beforeUpdate(fn)`：状态变更导致的更新**即将**应用到 DOM 之前调用。适合在变更前读取旧 DOM 状态进行对比。
- `afterUpdate(fn)`：一次 DOM 更新**之后**调用。适合进行依赖 DOM 的副作用（如测量尺寸、同步第三方控件）。
- `onDestroy(fn)`：组件销毁前调用。用于清理计时器、事件监听、订阅、第三方实例等。

示例（测量与清理）：
```svelte
<script>
  import { onMount, beforeUpdate, afterUpdate, onDestroy } from 'svelte';
  let el, width;

  onMount(() => {
    const ro = new ResizeObserver(() => width = el.clientWidth);
    ro.observe(el);
    return () => ro.disconnect();  // onDestroy
  });

  beforeUpdate(() => { /* 读取旧状态/DOM */ });
  afterUpdate(() => { /* 依赖新 DOM 的副作用 */ });
</script>

<div bind:this={el}>{width}</div>
```

---

## 8) `tick()` 的作用、时序保证与常见误用

**定义**：`tick()` 返回一个 Promise，在**当前批次的状态变更应用到 DOM 之后**才解析。用于在**下一次微任务**中访问“已经更新”的 DOM 状态。

典型用法：
```svelte
<script>
  import { tick } from 'svelte';
  let open = false;

  async function toggle() {
    open = !open;
    await tick();         // 等待 DOM 更新
    // 这里可以安全地测量/聚焦刚刚渲染出来的元素
    document.getElementById('panel')?.focus();
  }
</script>
```

**常见误用**
- 将 `tick()` 当做“强制刷新”或“延时器”。`tick()` 只是等待**Svelte 已经排队的更新**完成，不会触发新的更新。
- 在循环中多次 `await tick()` 导致不必要的多次布局与回流。建议**批量更新状态**，减少 `tick()` 次数。
- 与 `afterUpdate` 的取舍：`afterUpdate` 适合“每次更新后都做”的副作用；`tick()` 适合**单次操作**（如刚打开面板后聚焦）。

---

## 9) 条件/列表与性能：`{#if}`、`{#each}`、`{#key}` 的最佳实践

**条件渲染**
- `{:else if}` 连写时保持表达式简单；复杂条件可拆至计算属性或 `derived` store。
- 不同分支差异很大时，考虑使用 `{#key someKey}` 让分支切换时**重建**子树，避免残留状态。

**列表渲染**
- 总是提供**稳定且唯一**的 `key`（如后端 id）；避免使用索引作为 key：
  ```svelte
  {#each items as item (item.id)}
    <Row {item}/>
  {/each}
  ```
- 大型列表优先“**虚拟滚动**”库，模板层只负责渲染可视区子集。
- 事件处理函数避免在模板内创建过多临时闭包（可抽函数或使用 `on:click={() => handler(item.id)}` 但注意频次）。
- 在 `{#each}` 中避免重型计算，尽量上移至派生或 memo。

**`{#key}` 的用途**
- 在需要**强制重建**子树的场景（例如切换 tab 时重置内部状态、动画重放）使用：
  ```svelte
  {#key activeTab}
    <TabPanel id={activeTab}/>
  {/key}
  ```

---

## 10) 模板指令绑定：`bind:value`、`bind:this`、`bind:group`、`bind:clientWidth` 等的工作机制与边界

**`bind:value`（表单控件双向）**
```svelte
<script>
  let name = '';
</script>
<input bind:value={name}>
```
- 对原生 `<input>` 等，Svelte 通过**属性赋值 + 对应原生事件**（如 `input`/`change`）完成双向绑定。
- 控制复杂表单时，建议清晰地区分“受控/非受控”组件；复杂校验建议将状态上移到 store 或 Runes `$state`。

**组件级双向绑定**
- Svelte 4：父 `<Child bind:value />` 生效的前提是 **子组件 `export let value` 且子内部会对其直接赋值**，Svelte 会将子对 `value` 的赋值同步回父级。示例：
  ```svelte
  <!-- Child.svelte -->
  <script>
    export let value = '';
    function onInput(e){ value = e.target.value; }
  </script>
  <input {value} on:input={onInput}>

  <!-- Parent.svelte -->
  <Child bind:value={username} />
  ```
- 风险：子组件可以修改父级数据，**数据流向不够显式**。Svelte 5 提供更显式的 opt-in（如 `$bindable`），以限制无意的上行修改。

**`bind:this`（引用 DOM/组件实例）**
```svelte
<script>
  let el;
  let child;
  function focus(){ el?.focus(); }
</script>

<input bind:this={el}>
<Child bind:this={child} />
```
- SSR 中不可用（无 DOM），需在 `onMount` 后使用。
- 组件实例引用适合调用暴露的方法（通过 `export function` 暴露）。

**`bind:group`（单选/多选组）**
```svelte
<script>
  let picked = 'a';
</script>

<input type="radio" bind:group={picked} value="a">
<input type="radio" bind:group={picked} value="b">
```
- 单选：组内共享同一个变量。
- 多选（checkbox）：`group` 绑定到数组，Svelte 维护元素值的增删。

**尺寸与滚动绑定（如 `bind:clientWidth`、`bind:offsetHeight`、`bind:scrollY`）**
```svelte
<script>
  let el, width;
</script>
<div bind:this={el} bind:clientWidth={width} />
```
- 这些绑定会在布局变化时触发更新，**可能造成频繁重新渲染**。对性能敏感场合应：
  - 使用 `ResizeObserver`/`IntersectionObserver`，在回调里做**节流/去抖**；
  - 或封装自定义 `action` 集中调度（见下一题的 `use:` 模式）。

**SSR 与水合边界**
- 任何依赖客户端布局的 `bind:*` 在 SSR 阶段不可用，应在客户端启用后再读取。
- 与 `tick()`/`afterUpdate` 配合，确保读取到的是最新布局状态。

---
```
---

## 11) 作用域插槽与命名插槽：`<slot>`、`<slot name>`、插槽 props 传递与类型约束

### 原理与语法
- **默认插槽**：子组件通过 `<slot />` 暴露占位。父组件可直接写 slot 内容。
- **命名插槽**：子组件 `<slot name="header" />`；父组件使用 `<svelte:fragment slot="header">...</svelte:fragment>`。
- **作用域插槽（slot props）**：子组件在 `<slot>` 上**暴露值**，父组件用 `let:` 接收。

子组件：
```svelte
<!-- Card.svelte -->
<script lang="ts">
  export interface $$Slots {
    default: { user: { id: string; name: string } };
    header?: { title: string };
  }
  const user = { id: 'u1', name: 'Ada' };
  const title = 'Profile';
</script>

<div class="card">
  <header>
    <slot name="header" title={title}>Default Title</slot>
  </header>
  <section>
    <slot user={user} />
  </section>
</div>
```

父组件：
```svelte
<!-- Parent.svelte -->
<Card>
  <svelte:fragment slot="header" let:title>
    <h2>{title}</h2>
  </svelte:fragment>

  <!-- 作用域插槽，接收子暴露的 user -->
  <div let:user>
    <p>{user.name}</p>
  </div>
</Card>
```

### 类型约束
- 使用 `export interface $$Slots` 为各命名插槽声明其可用的 **props 类型**，`svelte-check`/TS 可在父组件的 `let:` 处进行类型校验。
- 父组件的 `<svelte:fragment slot="...">` 中的 `let:xxx` 会获得对应的类型提示。

### 边界与实践
- `slot` 内容属于父组件的作用域，**不能**直接访问子组件的局部变量；必须通过 slot props 显式暴露。
- 命名插槽可设置**后备内容**（写在子 `<slot>` 的内部），当父未提供时回退。
- 复杂插槽树建议抽离为子组件，避免在父模板中堆叠大量 `let:` 逻辑。

---

## 12) 动画与过渡：`transition:` / `animate:` / `svelte/motion`（`spring`/`tweened`）

### 三类能力
1. **过渡（Transitions）**：元素**进入/离开**时的过渡效果，指令 `in:`、`out:`、`transition:`（进出统一）。
   ```svelte
   <script>
     import { fly, fade, scale } from 'svelte/transition';
     let visible = false;
   </script>

   {#if visible}
     <div transition:fly={{ y: 16, duration: 150 }} />
   {/if}
   ```
2. **列表动画（`animate:flip`）**：同一列表内**重排**时的 FLIP 动画。
   ```svelte
   <script>
     import { flip } from 'svelte/animate';
     let items = [1,2,3];
   </script>

   <ul>
     {#each items as id (id)}
       <li animate:flip>{id}</li>
     {/each}
   </ul>
   ```
3. **数值插值（`svelte/motion`）**：`tweened`（基于时间）和 `spring`（基于物理弹性），适合**将状态变化平滑化**，并驱动样式或图形。
   ```svelte
   <script>
     import { tweened, spring } from 'svelte/motion';
     const opacity = tweened(0, { duration: 200 });
     const pos = spring({ x: 0, y: 0 }, { stiffness: 0.2, damping: 0.4 });
   </script>

   <div style="opacity: {$opacity}; transform: translate({$pos.x}px, {$pos.y}px);" />
   ```

### 性能与边界
- 大量节点同时过渡可能导致主线程拥堵；避免在长列表上给每个元素都绑定复杂的 `transition:`，必要时改用虚拟列表或仅对**可见区**元素过渡。
- `animate:flip` 依赖初末布局的测量；频繁强制回流需谨慎。确保列表使用 **稳定 key**。
- 尽量在 CSS 层使用 `transform`/`opacity`，避免触发重排。
- SSR 环境不会执行过渡；需要在客户端进行（配合 `onMount` 控制初始状态）。

---

## 13) 自定义 Action（`use:xxx`）设计：参数更新、清理函数、与原生交互模式

### 基本签名
```ts
type Action<P> = (node: HTMLElement, parameters?: P) => {
  update?: (newParams: P) => void;
  destroy?: () => void;
};
```

### 示例：点击外部关闭
```svelte
<!-- useClickOutside.ts -->
export function clickOutside(node: HTMLElement, cb: (e:MouseEvent)=>void) {
  function onDocClick(e: MouseEvent) {
    if (!node.contains(e.target as Node)) cb(e);
  }
  document.addEventListener('click', onDocClick, { capture: true });
  return {
    destroy() {
      document.removeEventListener('click', onDocClick, { capture: true } as any);
    }
  };
}
```

使用：
```svelte
<script lang="ts">
  import { clickOutside } from './useClickOutside';
  let open = true;
  function close(){ open = false; }
</script>

{#if open}
  <div use:clickOutside={close}>...</div>
{/if}
```

### 示例：`ResizeObserver`（带参数与 update）
```ts
// useResize.ts
export function resize(node: HTMLElement, onSize: (cr: DOMRectReadOnly)=>void) {
  const ro = new ResizeObserver(() => onSize(node.getBoundingClientRect()));
  ro.observe(node);
  return {
    update(fn: typeof onSize) { onSize = fn; },   // 参数热更新
    destroy() { ro.disconnect(); }
  };
}
```

### 设计要点
- **清理函数必须可靠**（返回的 `destroy()` 释放事件、Observer、定时器）。
- **参数更新**通过返回对象的 `update()` 实现，避免卸载/重装 action 导致闪烁或状态丢失。
- 事件监听优先 `{ passive: true }`（非阻塞滚动）与 `{ capture: true }`（必要时）。
- SSR 安全：Action 在 SSR 不执行；内部若访问 `window/document`，无需额外判断，但避免在模块顶层做副作用。

---

## 14) 自定义 Store：带副作用/清理/持久化（`localStorage`）

### 可读写 store 封装（带持久化与跨标签页同步）
```ts
// persistStore.ts
import { writable, type Writable } from 'svelte/store';

export function persistStore<T>(key: string, initial: T, options?: {
  serialize?: (v:T)=>string; deserialize?: (s:string)=>T; version?: number;
}) {
  const ser = options?.serialize ?? (v => JSON.stringify({ v, __v: options?.version ?? 1 }));
  const de  = options?.deserialize ?? (s => {
    const o = JSON.parse(s); return o.v;
  });

  let startValue = initial;
  if (typeof localStorage !== 'undefined') {
    const raw = localStorage.getItem(key);
    if (raw) try { startValue = de(raw); } catch {}
  }

  const s: Writable<T> = writable(startValue, (set) => {
    // 跨标签页同步
    function onStorage(e: StorageEvent) {
      if (e.key === key && e.newValue) set(de(e.newValue));
    }
    window?.addEventListener?.('storage', onStorage);
    return () => window?.removeEventListener?.('storage', onStorage);
  });

  // 持久化
  s.subscribe(v => {
    try { localStorage.setItem(key, ser(v)); } catch {}
  });

  return s;
}
```

使用：
```ts
// store.ts
export const settings = persistStore('app:settings', { theme: 'light', lang: 'zh' });
```

### 要点
- `writable` 第二参数 `start(set)` 返回清理函数，可在**最后一个订阅者取消**时释放资源（轮询、WebSocket 等）。
- 序列化建议带版本号与降级策略。
- SSR 期间 `localStorage` 不存在，读取需置于浏览器环境（如 `onMount`）或加存在性判断。

---

## 15) 可访问性（a11y）：编译器提示范围与模态对焦、键盘导航落地

### 编译器 a11y 提示（常见）
- `<img>` 缺少 `alt`。
- `<a>` 缺少 `href` 或不合法的 `href`。
- 可交互元素缺少可达的可点击/键盘事件或可聚焦属性（如 `role="button"` 时需要处理 `keydown`）。
- ARIA 属性与角色不匹配。
- 头阶层级跳跃、列表语义错误等（部分通过插件或 lint 规则提示）。

### 模态/对焦陷阱（Focus Trap）规范落地
- 模态根元素：`role="dialog"` 或 `role="alertdialog"`，`aria-modal="true"`。
- 打开模态时：
  - 记录当前 `document.activeElement`，聚焦到模态内第一个可聚焦元素。
  - 限制焦点循环在模态内部（拦截 `Tab`/`Shift+Tab`）。
  - 背景内容 `inert` 或 `aria-hidden="true"`（根据浏览器与设计选择其一）。
  - 禁止页面滚动（为 `body` 加上 overflow hidden，或添加滚动锁）。
- 关闭时：恢复滚动、恢复先前焦点。

示例（最小可用 focus trap）：
```svelte
<script>
  import { onMount, onDestroy } from 'svelte';
  export let open = false;
  let dialog: HTMLElement | null = null;
  let lastFocused: Element | null = null;

  function firstFocusable(root: HTMLElement){
    return root.querySelector<HTMLElement>(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
  }

  function onKey(e: KeyboardEvent){
    if(e.key !== 'Tab' || !dialog) return;
    const nodes = Array.from(
      dialog.querySelectorAll<HTMLElement>('button,[href],input,select,textarea,[tabindex]:not([tabindex="-1"])')
    );
    if(nodes.length === 0) return;
    const first = nodes[0], last = nodes[nodes.length-1];
    if(e.shiftKey && document.activeElement === first){ last.focus(); e.preventDefault(); }
    else if(!e.shiftKey && document.activeElement === last){ first.focus(); e.preventDefault(); }
  }

  onMount(() => {
    function onOpen(){
      lastFocused = document.activeElement;
      document.addEventListener('keydown', onKey);
      document.body.style.overflow = 'hidden';
      setTimeout(() => firstFocusable(dialog!)?.focus());
    }
    function onClose(){
      document.removeEventListener('keydown', onKey);
      document.body.style.overflow = '';
      (lastFocused as HTMLElement | null)?.focus?.();
    }
    $: open ? onOpen() : onClose();
    return () => !open && onClose();
  });
</script>

{#if open}
  <div class="mask" aria-hidden="true" />
  <div class="dialog" role="dialog" aria-modal="true" bind:this={dialog}>
    <slot />
  </div>
{/if}
```

### 键盘导航
- 所有交互控件可通过键盘操作；自定义可点击元素需要 `tabindex="0"`、`role` 与 `keydown` 处理 `Enter`/`Space`。
- 可见的聚焦样式，避免 `outline: none` 无替代。

---

## 16) 组件样式隔离：`<style>` 作用域机制、`:global()` 风险、与 Tailwind/CSS Modules 协作

### 作用域机制
- Svelte 编译器为组件内元素与样式添加类似 `class="svelte-abc123"` 的**散列选择器**实现**样式隔离**。
- 组件 `<style>` 中的选择器默认仅作用于该组件模板生成的 DOM。

### `:global()` 风险
- `:global(.btn)` 会跳出作用域隔离，污染全局，命名冲突风险高。建议将全局样式集中在 `src/app.css`、布局级组件或受控的 CSS 模块内，避免在任意子组件随意使用 `:global`。
- 必须使用 `:global` 时，尽量限定**作用域窄**的选择器（例如父容器类名前缀）。

### 与 Tailwind 协作
- 推荐在 SvelteKit 项目级引入 `tailwind` 与 `postcss`，全局样式在 `src/app.css`。
- 组件内部以**原子类**为主，复杂状态使用 `class:` 指令或 `@apply` 于全局层封装。
- 避免在 `<style>` 写大量覆盖 Tailwind 的规则；共存时优先**约定式命名**与设计令牌（CSS 变量）以降低特异性战争。

### 与 CSS Modules 协作
- 可以在普通 TS/JS 组件中使用 CSS Modules；在 `.svelte` 中主要靠作用域隔离即可，不强制引入 Modules。
- 若需要复用复杂样式片段，考虑抽成 Svelte 组件或制定全局 Design Tokens。

---

## 17) TypeScript 支持：`svelte-preprocess`、`tsconfig`、`ambient.d.ts` 关键配置与常见坑

### 基础配置
- SvelteKit 中已集成 TS 支持；非 Kit 项目建议使用 `svelte-preprocess` 以启用 `<script lang="ts">`、PostCSS/SCSS 等。
- `tsconfig.json` 关键项：
  ```json
  {
    "compilerOptions": {
      "moduleResolution": "bundler",
      "types": ["svelte"],
      "baseUrl": ".",
      "paths": { "$lib/*": ["src/lib/*"] },
      "strict": true,
      "jsx": "preserve",
      "esModuleInterop": true,
      "skipLibCheck": true
    }
  }
  ```
- `ambient.d.ts`（或 `app.d.ts`）中声明资源模块类型与全局类型：
  ```ts
  // src/app.d.ts
  /// <reference types="@sveltejs/kit" />
  declare module '*.svg' { const url: string; export default url; }
  declare module '*.svg?component' { const c: import('svelte').ComponentType; export default c; }
  ```

### 常见坑
- 忘记在 `tsconfig` 的 `types` 中包含 `svelte`，导致 `svelte` 类型缺失。
- Vite 的别名（如 `$lib`）未在 TS `paths` 同步配置，导致编辑器报错但构建可过。
- DOM 类型与 Node.js 类型混用时需显式引入 `@types/node` 或限定目标运行时。
- `svelte-check` 未配置脚本导致类型问题延迟暴露；建议在 CI 执行 `svelte-check --fail-on-warnings`。

---

## 18) SSR 与水合（Hydration）在 Svelte/SvelteKit 中的链路

### 执行链（首屏）
1. 客户端发起请求到 SvelteKit。
2. `hooks.server.ts` 中的 `handle` 包装请求（可注入会话、国际化、鉴权等）。
3. 按路由层级自外而内依次执行 `+layout.server.ts` → `+layout.ts` → `+page.server.ts` → `+page.ts` 的 `load`（见题 20）。
   - `server` 版本仅在服务器执行；`universal` 版本（`.ts` 无 `server` 后缀）**在 SSR 首屏执行一次**，随后在客户端导航时再次执行。
4. 服务器渲染 HTML，将 `load` 产出的数据序列化注入页面。
5. 客户端 JS 加载后，对 `+layout.svelte`、`+page.svelte` 进行**水合**：将事件监听与状态绑定到已有 DOM 上，避免二次创建。

### 水合注意点
- **一致性**：SSR 的输出与客户端首次渲染必须一致。涉及随机数、时间戳、`window` 尺寸等应推迟到 `onMount`。
- **可选择禁用水合的岛**：某些纯静态区域可在客户端跳过逻辑，或使用延迟加载组件。
- **数据传递**：`load` 返回的对象会以 `data` 传入页面/布局组件；保持结构稳定可降低水合复杂度。

---

## 19) SvelteKit 路由系统：`+page.svelte` / `+page.ts` / `+layout.svelte` / `+layout.ts` 职责

### 基本规则
- **基于文件**的路由；文件夹即路径，动态段用 `[id]`，通配段用 `[...rest]`。
- 页面视图入口：`+page.svelte`。
- 数据加载与预处理：
  - `+layout.ts` / `+page.ts`：**universal load**，可在服务器 SSR 首次运行与后续客户端导航时运行。
  - `+layout.server.ts` / `+page.server.ts`：**仅服务器**运行的 `load`，可访问 Cookies、私密后端、密钥等。
- 共享布局：`+layout.svelte` 包裹同级及子路由页面，适合放导航、头尾、国际化 Provider 等。
- 错误与重定向：`+error.svelte` 自定义错误页面；在 `load` 里 `error(...)` 或 `redirect(...)`。

### 典型结构
```
src/routes/
  +layout.svelte            # 全局布局
  +layout.ts                # 全局通用 load（可放国际化字典索引）
  admin/
    +layout.svelte          # 管理后台的二级布局
    +layout.server.ts       # 鉴权与角色校验
    users/
      +page.svelte
      +page.ts              # 通用数据拉取
  [id]/
    +page.svelte
    +page.server.ts         # 根据 id 拉取私有数据
```

---

## 20) 数据获取：`load`（客户端/服务端）与 `+page.server.ts` 的差异、选择策略

### `load` 的四种位置
- `+layout.ts` / `+page.ts`（universal）：SSR 首屏**在服务器执行**，客户端导航时**在浏览器执行**。
- `+layout.server.ts` / `+page.server.ts`（server-only）：**仅服务器执行**，永不在浏览器运行。

### 选择策略
- **需要访问机密、后端私有接口、Cookies/Headers、文件系统**：使用 `server load`（`+page.server.ts` / `+layout.server.ts`）。返回的安全数据会被序列化后仅以必要字段传给客户端。
- **可公开的数据或希望复用同一段逻辑于 SSR 与客户端导航**：使用 universal `load`（`+page.ts` / `+layout.ts`）。
- **表单 Actions**、鉴权重定向等需在进程内执行的场景：优先落在 `server load` 或 `+page.server.ts` 搭配 `fail/redirect`。

### `load` 签名与内部 `fetch`
```ts
// +page.ts
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ fetch, params, url, depends, data }) => {
  depends('app:users');                   // 与失效系统协作（题 25）
  const res = await fetch('/api/users');  // SvelteKit 包装的 fetch（会携带 cookies 到同源后端）
  const users = await res.json();
  return { users };
};
```

```ts
// +page.server.ts
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ locals, cookies, params }) => {
  const session = locals.session;         // 仅 server 可用
  const users = await getPrivateUsers(session.userId);
  return { users };
};
```

### 边界与实践
- `load` 的返回值须可序列化（不建议返回函数、类实例、巨大二进制）；大对象建议分页或懒加载。
- `load` 内的 `fetch` 会自动将 **同源请求**代理至服务器端（SSR 阶段），并带上 Cookies，便于会话一致性。
- 对于**强一致性**要求的页面（如订单、结算），优先 `server load`，将可信的渲染数据在服务器聚合后再下发，避免客户端 Race Condition。
- 与 `prerender` 的关系：静态预渲染页面需避免在 `load` 中访问动态会话信息；或配置路由的 `prerender` 策略并改造数据来源。

---

## 21) SvelteKit 表单 Actions：进程内执行、`fail/redirect`、与渐进增强

### 核心概念
- **进程内执行**：`+page.server.ts` 中的 `actions` 在服务端运行，接收 `RequestEvent`，可直接访问 `cookies/locals` 与私有后端。
- **渐进增强**：原生 `<form method="POST">` 即可提交。在客户端启用后，通过 `$app/forms` 的 `enhance` 将表单提交升级为 **fetch + 局部更新**，保留无 JS 时的可用性。
- **返回约定**：
  - `return { ... }`：成功，返回值合并到页面 `data.form`，可用于显示成功态或回显。
  - `throw redirect(status, location)`：重定向。
  - `return fail(status, data)`：失败，`status` 为 4xx/5xx，`data` 会注入 `data.form`，用于错误回显且保持 HTTP 语义。

### 目录与代码
```svelte
<!-- src/routes/login/+page.svelte -->
<script lang="ts">
  import { enhance } from '$app/forms';
  export let data; // 包含 server load 与 actions 返回的数据（含 data.form）
</script>

<form method="POST" use:enhance>
  <input name="email" value={data?.form?.values?.email || ''}>
  {#if data?.form?.errors?.email}<p class="err">{data.form.errors.email}</p>{/if}
  <input name="password" type="password">
  <button type="submit">Sign in</button>
  {#if data?.form?.message}<p>{data.form.message}</p>{/if}
</form>
```

```ts
// src/routes/login/+page.server.ts
import { fail, redirect, type Actions } from '@sveltejs/kit';

export const actions: Actions = {
  default: async ({ request, locals, cookies }) => {
    const form = await request.formData();
    const email = String(form.get('email') ?? '');
    const pwd   = String(form.get('password') ?? '');

    const errors: Record<string, string> = {};
    if (!/^\S+@\S+$/.test(email)) errors.email = 'invalid email';
    if (pwd.length < 8) errors.password = 'min length 8';

    if (Object.keys(errors).length) {
      return fail(400, { errors, values: { email }, message: 'invalid fields' });
    }

    const session = await locals.auth?.login(email, pwd);
    if (!session) return fail(401, { message: 'wrong credentials', values: { email } });

    cookies.set('sid', session.id, {
      httpOnly: true, secure: true, sameSite: 'lax', path: '/', maxAge: 60 * 60 * 24 * 7
    });

    throw redirect(302, '/dashboard');
  }
};
```

### 设计要点
- **幂等性**：对“重试可能发生”的 Action（网络抖动、用户重复提交），确保后端与 Action 逻辑幂等。
- **可访问性**：失败信息在无 JS 时也能通过 `fail` 回传并展示。
- **安全性**：优先在 **Action（服务端）** 做校验，避免信任客户端。

---

## 22) Endpoints/API 路由：`+server.ts` 的方法处理、`RequestEvent`、Cookies/Headers 与 CORS

### 路由与方法
- 放置于 `src/routes/**/+server.ts`，导出 `GET/POST/PUT/DELETE/OPTIONS/PATCH` 等方法。
- 返回 `Response` 或 `json(data, init?)`。

```ts
// src/routes/api/users/+server.ts
import { json, type RequestHandler } from '@sveltejs/kit';

export const GET: RequestHandler = async ({ url, locals, setHeaders }) => {
  setHeaders({ 'Cache-Control': 'private, max-age=30' });
  const q = url.searchParams.get('q') ?? '';
  const users = await locals.db.findUsers(q);
  return json(users);
};

export const POST: RequestHandler = async ({ request, locals }) => {
  const body = await request.json();
  const user = await locals.db.createUser(body);
  return json(user, { status: 201 });
};
```

### `RequestEvent` 要点
- `request`、`params`、`url`、`locals`、`cookies`、`fetch`、`setHeaders`、`getClientAddress`。
- **同源 fetch 透传**：在 SSR 阶段，`event.fetch('/api/...')` 会携带 Cookies，等价于服务端内部调用，有利于会话一致性。

### Cookies 与 Headers
```ts
cookies.set('sid', 'value', {
  httpOnly: true, secure: true, sameSite: 'lax', path: '/', maxAge: 86400
});
setHeaders({
  ETag: etagString,
  'Cache-Control': 'public, max-age=300, s-maxage=600'
});
```

### CORS 策略
- 若需要跨域，显式处理 `OPTIONS`：
```ts
// src/routes/api/public/+server.ts
import { json, type RequestHandler } from '@sveltejs/kit';

const ALLOW = {
  'Access-Control-Allow-Origin': 'https://example.com',
  'Access-Control-Allow-Methods': 'GET,POST,OPTIONS',
  'Access-Control-Allow-Headers': 'content-type,authorization',
  'Access-Control-Allow-Credentials': 'true'
};

export const OPTIONS: RequestHandler = async () => new Response(null, { headers: ALLOW });
export const GET: RequestHandler = async () => json({ ok: 1 }, { headers: ALLOW });
```
- 对私有接口，优先**同源**访问，避免暴露不必要的跨域能力。

---

## 23) 会话与鉴权：`hooks.server.ts` 的 `handle`、`locals`、Cookies/Session 的安全与多端一致性

### `handle` 建链
```ts
// src/hooks.server.ts
import type { Handle } from '@sveltejs/kit';
import { getSessionById } from '$lib/server/session';

export const handle: Handle = async ({ event, resolve }) => {
  const sid = event.cookies.get('sid');
  if (sid) {
    const session = await getSessionById(sid);
    if (session) event.locals.user = { id: session.userId, roles: session.roles };
  }
  return resolve(event, {
    filterSerializedResponseHeaders: (name) => name === 'content-type' // 避免泄漏敏感头
  });
};
```

- 将鉴权信息放入 `event.locals`，供 `load`、`actions`、`+server.ts` 使用。
- 禁止在客户端暴露完整的会话对象；在 `load` 返回时仅传必要字段。

### Cookie 与 Session 安全
- `httpOnly`、`secure`、`sameSite=lax/strict`、`path=/`、`maxAge`。
- 会话后端可用 Redis 或数据库，提供**可撤销**能力：用户在任意设备登出后，其他设备的会话可被标记失效。
- **刷新策略**：若使用短期访问令牌 + 长期刷新令牌，刷新接口必须只在服务端路径可用，并绑定客户端指纹/旋转刷新令牌以降低重放风险。
- **CSRF**：同源 + `sameSite` 已降低风险。对跨站需要的接口（如第三方嵌入）引入 **Origin/Referer 校验** 或 CSRF Token（Action 中通过隐藏字段或 Cookie 双提交）。

### 多端一致性
- 统一在 `server load` 聚合用户态数据，前端只依赖 `data.user`；变更资料后通过失效机制（题 24）触发客户端刷新。

---

## 24) 失效与缓存：`invalidate`/`invalidateAll`、`prefetch` 与 HTTP 缓存头协作

### 客户端失效 API
- `import { invalidate, invalidateAll } from '$app/navigation'`
  - `invalidate(urlOrPredicate)`：标记与该 URL 匹配的 `load` 数据失效，**下一次**导航或显示时重拉。
  - `invalidateAll()`：失效所有当前页面数据。
- `depends('tag')`：在 `load` 中声明依赖标签，后续服务端可通过 **命名失效** 触发客户端刷新（例如从 SSE/WS 推送变更）。

```ts
// +page.ts
export const load = async ({ fetch, depends }) => {
  depends('app:users'); // 声明依赖
  const users = await (await fetch('/api/users')).json();
  return { users };
};
```

```ts
// 触发端（例如 WebSocket 消息、SSE、管理后台）调用一个端点
// 服务器可通知客户端：/app:users 失效（实现方式依赖你的推送层）
```

### 预取与加速
- 链接上使用 `data-sveltekit-preload-data` 或 `data-sveltekit-preload-code`，或在交互中调用 `prefetch(url)`，可提前加载路由数据与代码拆分块，提升导航性能。
- `prefetch` 不会提交表单，仅预拉取 `load` 数据。

### 与 HTTP 缓存协作
- 后端可设置 `Cache-Control: public, max-age=...`、`ETag`，在 **universal load 的 `fetch`** 上透明复用。如果需要**强制最新**，在 `fetch` 传入 `cache: 'no-store'`。
- 对**用户私有数据**使用 `private, no-store`，避免浏览器或中间层缓存。
- 对**静态资源**使用文件指纹（Vite 构建产物）与 `immutable`。

---

## 25) Streaming 与渐进渲染：流式响应与 “Suspense-like” 体验

### 渐进渲染（模板层 `#await`）
- 在 `+page(.server).ts` 的 `load` 返回 **Promise 字段**，页面可用 `{#await ...}` 先渲染骨架，再在 Promise 解决后渲染最终内容。
```ts
// +page.server.ts
export const load = async () => {
  const slow = new Promise<string>(r => setTimeout(() => r('done'), 800));
  return { slow };
};
```
```svelte
<!-- +page.svelte -->
{#await data.slow}
  <Skeleton/>
{:then text}
  <div>{text}</div>
{/await}
```

### HTML 流（适配器支持）
- 某些适配器（如 `adapter-node`、部分 Edge 方案）支持 **chunked text/html**。SvelteKit 会先输出可确定部分，再陆续 flush 其余段落。
- 适合首屏快速展示框架与已就绪区块，将慢数据放在 `#await` 中延后。

### 数据流（SSE/WebSocket）
- 对“持续更新”数据，推荐建立 **SSE** 端点与客户端订阅：
```ts
// src/routes/api/events/+server.ts
export const GET: RequestHandler = async () => {
  const stream = new ReadableStream({
    start(controller) {
      const send = (ev: any) => controller.enqueue(`data: ${JSON.stringify(ev)}\n\n`);
      const t = setInterval(() => send({ ts: Date.now() }), 1000);
      return () => clearInterval(t);
    }
  });
  return new Response(stream, {
    headers: { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache', Connection: 'keep-alive' }
  });
};
```
- 客户端使用 `EventSource` 或在组件中封装 store，将事件推入 `$state`/`writable` 以驱动 UI。

---

## 26) 适配器与部署：`@sveltejs/adapter-node` / `adapter-static` / `adapter-vercel` / `adapter-cloudflare` 的取舍

| 适配器 | 模式 | 适用场景 | 约束/边界 |
|---|---|---|---|
| `adapter-node` | 长驻 Node 进程 | 自主管理服务器、需要 WebSocket、SSE、长连接、细粒度缓存/中间件 | 需自行运维；可直接使用 Node API、文件系统、流式 HTML |
| `adapter-static` | 纯静态导出（SSG） | 完全静态站点或仅靠客户端数据获取的 SPA | 不支持 SSR/Actions/`+server.ts`；需明确 `prerender` 策略 |
| `adapter-vercel` | Serverless/Edge | Vercel 平台，函数化部署、区域加速、可选 Edge Runtime | Serverless 冷启动、运行时超时限制；Edge 环境无 Node API |
| `adapter-cloudflare` | Workers/Pages | Cloudflare 全球边缘、KV/D1/R2 集成 | Workers 环境基于 Web 标准，无 Node API，需使用可移植依赖 |

选择原则：
- 需要 **WebSocket/SSE/长轮询/批处理**：优先 `adapter-node` 或支持流的 Serverless/Edge 方案。
- 静态内容为主、SEO 要求可通过 SSG 满足：考虑 `adapter-static`，辅以客户端数据拉取。
- 对冷启动敏感、请求量峰谷明显：Serverless 合适，但需注意超时与连接限制。

---

## 27) 资源与图像优化：静态资源、CDN、`<picture>/srcset`、懒加载与预加载

### 静态资源与路径
- `static/` 目录内容通过服务器静态托管，发布路径即 `/文件名`。
- 构建产物采用 **哈希指纹**；可配合 CDN 设置 `Cache-Control: public, max-age=31536000, immutable`。

### 响应式图片
```svelte
<picture>
  <source type="image/avif" srcset="/img/hero-640.avif 640w, /img/hero-1280.avif 1280w" sizes="(max-width: 768px) 100vw, 1280px">
  <source type="image/webp" srcset="/img/hero-640.webp 640w, /img/hero-1280.webp 1280w" sizes="(max-width: 768px) 100vw, 1280px">
  <img src="/img/hero-1280.jpg" alt="Hero" width="1280" height="720" loading="lazy" decoding="async">
</picture>
```
- 使用 `sizes` 与 `srcset`，优先现代格式（AVIF/WebP），保留 JPEG 回退。
- 指定固有尺寸（`width/height`）减少布局抖动（CLS）。

### 懒加载与预加载
- 图片 `loading="lazy"`、`decoding="async"`；对首屏关键图像可 `preload`。
- 路由级提前加载：`data-sveltekit-preload-data` 或 `prefetch()`。
- 非阻塞第三方脚本：`defer`/`async`，必要时使用 **分域** 与 `preconnect`。

### 运行时优化
- 大图/图集列表采用 **IntersectionObserver** 批量懒加载与卸载。
- 若需要处理源图（裁剪、压缩），使用后端或边缘图像服务（CDN 转码）并缓存。

---

## 28) SEO 与元信息：`<svelte:head>`、由 `load` 驱动的动态 meta、OGP 最佳实践

### 基本用法
```svelte
<!-- +page.svelte -->
<script>
  export let data;
</script>

<svelte:head>
  <title>{data.title} – Site</title>
  <meta name="description" content={data.desc}>
  <link rel="canonical" href={data.canonical}>
  <meta property="og:title" content={data.title}>
  <meta property="og:description" content={data.desc}>
  <meta property="og:type" content="article">
  <meta property="og:url" content={data.canonical}>
  <meta property="og:image" content={data.ogImage}>
  <meta name="twitter:card" content="summary_large_image">
</svelte:head>
```

```ts
// +page.server.ts
export const load = async ({ params, url }) => {
  const article = await fetchArticle(params.slug);
  const canonical = new URL(`/posts/${article.slug}`, url.origin).toString();
  return {
    title: article.title,
    desc: article.summary,
    canonical,
    ogImage: article.ogImage
  };
};
```

### 实务要点
- **规范化 URL**：使用 `<link rel="canonical">`；分页/过滤页慎设 canonical。
- **结构化数据**：在 `<svelte:head>` 注入 JSON-LD（`type="application/ld+json"`），确保与可见内容一致。
- **机器人与站点地图**：提供 `/robots.txt` 与 `/sitemap.xml`（可用 `+server.ts` 生成），与 `prerender` 协作输出静态条目。
- **性能指标**：首屏关键 CSS/字体预加载，避免阻塞；减少布局偏移，提升核心 Web 指标。

---

## 29) 国际化（i18n）：字典组织、懒加载、路由前缀与 SSR 同构

### 路由与语言解析
- 推荐使用路径前缀：`/[lang]/...`，`[lang]` 为 `en/zh/...`。
- 在 `+layout.server.ts` 或 `hooks.server.ts` 中解析 `params.lang` 与 `Accept-Language`，确定语言并写入 `locals.locale` 与 Cookie（便于后续客户端导航）。

```ts
// src/routes/[lang]/+layout.server.ts
import type { LayoutServerLoad } from './$types';
import { loadDict } from '$lib/i18n/server';

export const load: LayoutServerLoad = async ({ params, cookies }) => {
  const lang = params.lang ?? cookies.get('lang') ?? 'en';
  const dict = await loadDict(lang); // 服务器读取或远程获取
  return { lang, dict };
};
```

```svelte
<!-- src/routes/[lang]/+layout.svelte -->
<script>
  export let data;
  // 可将 data.dict 放入 context/store，供子页面使用
</script>
<slot />
```

### 懒加载字典（客户端）
- Universal `load` 中按需 `import()` 对应语言包，并缓存至 store。
- 注意 SSR 同构：首屏 SSR 已注入字典，客户端水合时避免重复请求；可依赖 `data.dict`。

### 字典组织与命名
- 拆分为 **按页面/模块** 的命名空间（如 `common.json`, `home.json`, `product.json`），避免单文件过大。
- 文案键名语义化且稳定，避免包含具体 UI 词序；句内变量使用插值函数安全替换。

### 其他要点
- **方向性**：阿语/希伯来语需处理 `dir="rtl"` 与样式镜像。
- **缓存**：为字典设置 CDN 缓存与 `ETag`；变更时使用 `version` 或哈希参数便于失效。

---

## 30) 表单与验证：原生表单 + Actions 服务端校验、Zod/Valibot 集成与错误回显

### 流程与分层
1. UI：原生 `<form>`，字段 `name` 语义清晰；可用 `enhance` 升级为非刷新提交。
2. 服务端：`+page.server.ts` 的 Action 进行**权威校验**，返回 `fail(400, { errors, values })`。
3. UI 回显：基于 `data.form` 渲染错误与保留用户输入。
4. 客户端可选：在提交前进行**即时提示**（不代替服务端校验）。

### Zod 集成示例
```ts
// src/lib/validation.ts
import { z } from 'zod';
export const SignupSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  agree: z.literal('on').optional() // checkbox
});
export type SignupInput = z.infer<typeof SignupSchema>;
```

```ts
// src/routes/signup/+page.server.ts
import { fail, redirect, type Actions } from '@sveltejs/kit';
import { SignupSchema } from '$lib/validation';

export const actions: Actions = {
  default: async ({ request, locals }) => {
    const fd = await request.formData();
    const input = {
      email: String(fd.get('email') ?? ''),
      password: String(fd.get('password') ?? ''),
      agree: fd.get('agree') ? 'on' : undefined
    };
    const parsed = SignupSchema.safeParse(input);
    if (!parsed.success) {
      const fieldErrors: Record<string, string> = {};
      for (const i of parsed.error.issues) {
        const path = i.path.join('.');
        fieldErrors[path] = i.message;
      }
      return fail(400, { errors: fieldErrors, values: input });
    }

    await locals.db.createUser(parsed.data);
    throw redirect(303, '/welcome');
  }
};
```

```svelte
<!-- src/routes/signup/+page.svelte -->
<script lang="ts">
  import { enhance } from '$app/forms';
  export let data;
</script>

<form method="POST" use:enhance>
  <label>Email <input name="email" value={data?.form?.values?.email || ''}></label>
  {#if data?.form?.errors?.email}<p class="err">{data.form.errors.email}</p>{/if}

  <label>Password <input name="password" type="password"></label>
  {#if data?.form?.errors?.password}<p class="err">{data.form.errors.password}</p>{/if}

  <label><input type="checkbox" name="agree"> I agree</label>
  <button type="submit">Create account</button>

  {#if data?.form?.message}<p>{data.form.message}</p>{/if}
</form>
```

### 工程化细节
- **防重复提交**：在客户端根据 `form` 的 `pending` 状态禁用按钮；服务端使用幂等键（如 nonce）抵御重复请求。
- **错误分层**：字段错误（字典）+ 表单级错误（例如 “邮箱已存在”）。
- **文件上传**：Action 使用 `request.formData()` 处理，限制 MIME 与尺寸；与对象存储的直传结合（预签名 URL）。
- **安全**：所有信任边界在服务端；敏感错误信息不要直接返回给客户端。

---

## 31) 大型列表/表格性能：虚拟滚动与 Svelte 模板协作、`{#key}` 与不可变数据策略

### 性能瓶颈来源
- 节点数量过大导致创建/更新/销毁成本高（布局、绘制、事件绑定、过渡动画）。
- 每行绑定大量响应式表达式与副作用（`$:`、`bind:*`、`transition:`）。
- 在 `{#each}` 中**原地变更**数据结构（push/splice）引发难以预测的更新与重排。

### 虚拟滚动协作模式
核心思路是只渲染可视窗口内的若干行（加前后缓冲），其余以**占位高度**填充，滚动时计算可视区索引。

固定行高（最简单）示例：
```svelte
<script>
  export let items = [];        // 大数组
  const rowHeight = 40;
  const overscan = 6;

  let container;
  let scrollTop = 0;
  $: total = items.length * rowHeight;
  $: start = Math.max(0, Math.floor(scrollTop / rowHeight) - overscan);
  $: end   = Math.min(items.length, Math.ceil((scrollTop + viewport) / rowHeight) + overscan);
  $: visible = items.slice(start, end);

  let viewport = 0;
  function onScroll(){ scrollTop = container.scrollTop; }
  function onResize(){ viewport = container.clientHeight; }
</script>

<div bind:this={container} on:scroll={onScroll} use:resizeObserver={onResize}
     style="overflow:auto;height:600px;position:relative;">
  <div style="height:{total}px;position:relative;">
    {#each visible as item, i (item.id)}
      <div style="position:absolute;left:0;right:0;top:{(start+i)*rowHeight}px;height:{rowHeight}px;">
        <Row {item}/>
      </div>
    {/each}
  </div>
</div>

<script>
  // 简易 ResizeObserver action
  export function resizeObserver(node, cb){ const ro = new ResizeObserver(cb); ro.observe(node); return { destroy(){ ro.disconnect(); } }; }
</script>
```

可变行高的做法：
- 行组件内用 `ResizeObserver` 报告真实高度，维护一张 `heights[]` 与 `prefixSum[]` 表，`scrollTop -> index` 用二分查找。工程上更建议采用成熟库（如虚拟列表/无限滚动库）并保证**稳定 key**。

### `{#key}` 与不可变数据
- 列表渲染使用**稳定且唯一**的 key（后端 id）。避免索引作为 key：
  ```svelte
  {#each items as item (item.id)}
    <Row {item}/>
  {/each}
  ```
- 若需要在切换筛选/排序时**强制重建**子树（清空内部状态、重放动画），可在外层使用 `{#key someHash}`：
  ```svelte
  {#key filterKey}
    <VirtualTable {items}/>
  {/key}
  ```
- 更新列表时使用**不可变策略**：替换引用而非原地改动；仅替换发生变化的节点或字段。Svelte 4 中 `arr[i].x = 1` 不会触发，需要 `arr = [...arr]`；Svelte 5 可用 `$state` 深追踪减少这种样板。

### 事件与渲染优化
- 事件**委托**：对行内点击/选择，尽量把 `on:click` 绑定在列表容器上，通过 `event.target`/`dataset` 区分，减少每行闭包与绑定。
- 避免在模板内执行重型计算（排序/过滤/聚合），上移至派生（`derived`/`$derived`）或服务端分页。
- 大型表格的单元格内容尽量用 `transform/opacity` 实现动画，避免触发布局回流。

---

## 32) 构建与打包：Vite 配置（别名、分包、动态 import）、生产 Source Map 与体积预算

### Vite 基本配置
```ts
// vite.config.ts
import { defineConfig } from 'vite';
import { sveltekit } from '@sveltejs/kit/vite';
import path from 'node:path';

export default defineConfig({
  plugins: [sveltekit()],
  resolve: {
    alias: {
      $lib: path.resolve('./src/lib'),
      $assets: path.resolve('./src/assets')
    }
  },
  build: {
    target: 'es2020',
    sourcemap: true,                // 生产可开启 hidden 源图：'hidden'
    cssCodeSplit: true,
    assetsInlineLimit: 0,           // 可选：全部走文件，便于体积分析
    chunkSizeWarningLimit: 600,     // 警告阈值（仅警告，非强制）
    rollupOptions: {
      output: {
        manualChunks: {
          // 基于依赖分包：缓存更稳定
          svelte: ['svelte', 'svelte/transition', 'svelte/animate', 'svelte/motion'],
          vendor: ['lodash-es', 'dayjs']
        }
      }
    },
    minify: 'esbuild'               // 兼容性要求较高时可用 'terser'
  },
  define: {
    __BUILD_TIME__: JSON.stringify(new Date().toISOString())
  },
  optimizeDeps: {
    include: ['lodash-es'],
    exclude: []
  }
});
```

SvelteKit 说明：
- 路由级代码拆分由 Kit 自动处理；全局 vendor 拆分用 `manualChunks` 辅助稳定缓存。
- SSR 构建可用 `ssr.noExternal`/`ssr.external` 控制外部依赖是否打包进 SSR 产物（Workers/Edge 环境往往要求 ESM 且不可使用 Node 内置模块）。

### 动态 import 与预取
- 基于用户交互或视图分段按需加载：
  ```ts
  const { heavyFn } = await import('./heavy.ts');  // 可配合路由懒加载
  ```
- SvelteKit 链接可使用 `data-sveltekit-preload-data` 与 `data-sveltekit-preload-code`，或在逻辑中调用 `prefetch()` 实现**预加载**。

### 生产 Source Map 策略
- `sourcemap: 'hidden'`：构建产物不暴露 `//# sourceMappingURL`，但 CI/监控可上传 sourcemap（如 Sentry），降低源码泄漏风险。
- 通过 CI 限制 sourcemap 访问，仅供错误上报平台。

### 体积预算与可观测性
- 使用 `rollup-plugin-visualizer` 或 `vite-bundle-visualizer` 生成包分析报告。
- 在 CI 编写**预算脚本**，按 gzip/br 压缩后大小设阈值：
  ```bash
  # 伪示例：统计 dist/*.js.gz > 200KB 则失败
  find dist -name "*.js" -exec bash -c 'gzip -c "$1" | wc -c | xargs -I{} echo {} "$1"' _ {} \; \
    | awk '$1>200000{print "oversize:", $2; fail=1} END{exit fail}'
  ```
- 对图片/字体采用指纹与 CDN 长缓存；JS/CSS 体积控制在**首屏 150–200KB gzip**量级为常见经验基线（具体视业务与设备而定）。

---

## 33) 测试策略：`@testing-library/svelte`、Vitest、Playwright 的分层与陷阱

### 分层模型
- **单元/组件测试（Vitest + @testing-library/svelte）**：验证组件渲染与交互、store 派生逻辑、工具函数。
- **集成测试（Vitest + jsdom/happy-dom）**：组件间交互、store 与网络层的组合（可用 MSW mock）。
- **端到端（Playwright）**：路由、表单 Actions、SSR/水合、鉴权流、真实浏览器行为。

### 组件测试示例
```svelte
<!-- Counter.svelte -->
<script>
  export let init = 0;
  let count = init;
  function inc(){ count += 1 }
</script>
<button aria-label="inc" on:click={inc}>{count}</button>
```

```ts
// Counter.test.ts
import { render, fireEvent, screen } from '@testing-library/svelte';
import { describe, it, expect } from 'vitest';
import Counter from './Counter.svelte';

describe('Counter', () => {
  it('increments on click', async () => {
    render(Counter, { init: 1 });
    await fireEvent.click(screen.getByRole('button', { name: 'inc' }));
    expect(screen.getByRole('button', { name: 'inc' }).textContent).toBe('2');
  });
});
```

要点：
- 选择器优先**可访问性**（`getByRole`, `getByLabelText`），避免脆弱的 class/testid。
- 动画/过渡可能影响时序，必要时禁用过渡或使用 `await tick()`/`await Promise.resolve()` 让微任务队列清空。
- 需要 DOM 环境时设置 `test.environment = 'jsdom'`；仅逻辑测试可用 `node` 环境提升性能。

### Store 与异步测试
```ts
import { writable } from 'svelte/store';
import { vi } from 'vitest';

it('store persists', async () => {
  const s = writable(0);
  const spy = vi.fn();
  const unsub = s.subscribe(spy);
  s.set(1); s.update(x=>x+1);
  unsub();
  expect(spy.mock.calls.at(-1)[0]).toBe(2);
});
```
- 定时器/轮询使用 `vi.useFakeTimers()`，并在断言前 `await vi.runAllTimersAsync()`。
- 使用 `msw` 模拟 `fetch`，避免耦合真实网络。

### E2E（Playwright）
- 在 `beforeAll` 启动已构建的 SvelteKit 服务器（生产近似），设置 `baseURL`。
- 登录场景用**测试账户**与干净数据库/沙箱后端；或拦截网络层以模拟服务端响应（仅限非关键路径）。
- 端到端断言也优先**可访问性**选择器；避免 brittle 的视觉快照为主、行为断言为辅。
- 跨路由/重定向/表单 Actions/文件上传是 Playwright 的重点覆盖对象。

常见陷阱：
- 忽视 SSR/水合差异，仅在 jsdom 下跑单测；需要用 E2E 在真实浏览器与服务器上补齐。
- 在组件测试中直接操作内部实现细节（如私有变量），破坏重构自由。
- 并发测试共享全局可变状态（定时器、store 单例、防抖缓存）导致相互污染；使用独立模块工厂或在 `afterEach` 重置。

---

## 34) 安全：`{@html}` 边界与 XSS、防注入及敏感数据暴露控制

### `{@html}` 使用边界
- 仅渲染**可信来源**的 HTML；对用户输入或第三方内容必须**先净化**。
- 最小化允许标签/属性集合，避免事件属性（`on*`）、`javascript:`/`data:` 协议 URL、内联样式中的危险表达式。

使用 DOMPurify（SSR/浏览器均可，Node 端用 isomorphic-dompurify）：
```svelte
<script>
  import DOMPurify from 'isomorphic-dompurify';
  export let raw = '';
  $: safe = DOMPurify.sanitize(raw, {
    ALLOWED_TAGS: ['p','a','strong','em','ul','ol','li','code','pre','img'],
    ALLOWED_ATTR: ['href','target','rel','src','alt']
  });
</script>

<div>{@html safe}</div>
```

Trusted Types（支持的浏览器）：
```js
// 创建只允许 DOMPurify 输出的策略
window.trustedTypes?.createPolicy('dompurify', {
  createHTML: (s) => DOMPurify.sanitize(s, {/*...*/})
});
```
并在 CSP 中启用 `require-trusted-types-for 'script'`。

### URL 与外链
- 对外链接使用 `rel="noopener noreferrer"`；校验 `href` 协议白名单（`http:`,`https:`）。
- 图片/媒体源同样需要协议校验与大小/类型限制。

### SvelteKit 级安全措施
- 环境变量：机密只放 `$env/static/private` / `$env/dynamic/private`；**禁止**从 `load` 返回完整机密或签名。
- 会话 Cookie：`httpOnly`、`secure`、`sameSite=lax|strict`、`path=/`、`maxAge`；刷新令牌仅在服务端流转。
- CSRF：同源 + `sameSite` 已有基础防护；跨站场景使用 **Origin/Referer 校验** 或双提交 Cookie。
- CSP：为页面设置严格 CSP（Script/Style/Img/Font 等白名单），可在 SvelteKit 配置或反向代理层注入。
- 头部隐藏：`handle` 中使用 `filterSerializedResponseHeaders` 避免将敏感头序列化到客户端。
- 错误处理：自定义 `+error.svelte`，避免生产环境泄漏堆栈与内部路径。

---

## 35) 迁移与前沿：从 Svelte 3/4 到 Svelte 5（Runes）的步骤、兼容层、渐进迁移与团队成本

### 迁移路径总览
1. **依赖升级**：升级 `svelte`、`@sveltejs/kit`、`@sveltejs/adapter-*`、`@sveltejs/kit/vite` 与相关插件，确保构建流程无警告。
2. **保持兼容模式**：Svelte 5 默认仍兼容 4 的语义（即不启用 Runes）。先在兼容模式下修正弃用 API、破坏性变更（若有）。
3. **按组件渐进启用 Runes**：在目标组件顶部添加 `<svelte:options runes={true} />`，逐步迁移核心状态与副作用写法。
4. **抽象层同步**：同步更新公用 hooks/store/action/util 的约定，使其同时适配 Runes 与非 Runes 组件。

### 代码迁移映射
| 旧写法（Svelte 4） | 新写法（Svelte 5 / Runes） | 说明 |
|---|---|---|
| `$: a = f(b, c)` | `let a = $derived(() => f(b, c))` | 纯计算迁移到只读派生，依赖自动收集 |
| `$: { sideEffect(x); }` | `$effect(() => { sideEffect(x); })` | 显式副作用边界与依赖 |
| `obj.items.push(v); obj = obj;` | `let obj = $state({ items: [] }); obj.items.push(v);` | 深度追踪，免“换引用”样板 |
| 子组件内部直接赋值触发父级 `bind:value` | `$bindable`（或改为 `dispatch('change', detail)`） | 更显式的上行数据流控制 |
| 组件内局部 `writable` | 仍可用，或改 `$state` | 跨组件共享仍建议 store/context |

### 渐进策略与质量保障
- **优先迁移新功能/边缘组件**，验证团队对 Runes 的掌握与工具链适配；稳定后迁移热点组件。
- **建立 lint 规则/commit 审查**：限制在 Runes 组件中再写 `$:` 语句块副作用；引导到 `$effect`。
- **测试回归**：单元与 E2E 用例保持不变，关注渲染时序差异（`afterUpdate` 与 `$effect` 的触发时机差别）。
- **文档与培训**：一次专门的学习与 code review 模板，统一以下准则：
  - 本地状态优先 `$state`；跨组件共享继续使用 store/context。
  - 派生逻辑统一 `$derived`，避免混杂在模板或 `$:` 中。
  - 副作用集中 `$effect`，并说明清理策略与依赖边界。

### 风险与回退
- 若某些第三方库/生态尚未适配，保持对应组件**不启用 Runes** 即可；Svelte 5 允许混合。
- 出现性能回退时，基于 profiler 对比**更新次数与依赖图**，检查是否误把重型逻辑放入 `$effect` 高频触发路径。

---

