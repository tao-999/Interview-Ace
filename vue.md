## 1) 响应式系统（Proxy + effect + scheduler）：依赖收集与触发更新

**核心组件**
- `reactive()`/`ref()`：用 **Proxy**（对象/集合）或 **Ref 包装器**（`.value`）拦截 `get/set`。
- `effect(fn)` → `ReactiveEffect`：运行 `fn` 时建立**依赖图（dep）**。读取时 **track(target,key)**；写入时 **trigger(target,key,type)**。
- **Dep**：`Set<ReactiveEffect>`。同一帧内**去重**。
- **scheduler（调度器）**：把触发的 effect 放入 **job 队列**（microtask），按 `flush` 时机统一执行，减少抖动。

**关键 track/trigger 类型**
- 普通属性：`get/set`
- 结构变化：`add`/`delete`（含 `Map/Set`）
- 迭代：`ITERATE_KEY`（for…in / for…of）与 `MAP_KEY_ITERATE_KEY`（`Map.keys()`）

**执行顺序与去重**
1. 写入触发 `trigger`，把相关 effect 入队（含 `computed.effect`、组件渲染副作用）。
2. **微任务**批量执行，避免级联重复渲染。
3. 组件渲染 effect 的优先级略低于用户 `watch` 的 `flush:'pre'`。

**伪码**
```ts
function track(target,key) {
  if (!activeEffect) return;
  dep = getDep(target,key); dep.add(activeEffect);
}
function trigger(target,key,type) {
  const dep = getDep(target,key);
  for (const eff of dep) queueJob(eff.scheduler ?? eff.run);
}
```

---

## 2) `ref`/`reactive`/`shallowRef`/`shallowReactive`/`markRaw` 对比与坑

- **`reactive(obj)`**：深层 **对象/集合**响应式（嵌套按需懒代理）。**不包装原始值**。
- **`ref(v)`**：对**原始值或对象**统一成 `{ value }`；模板中自动脱 `.value`；对象值会被 `reactive` 化。
- **`shallowRef`**：仅追踪 `.value` **引用变化**，内部对象**不递归**代理。适合频繁整体替换的大对象（如图表实例）。
- **`shallowReactive`**：仅第一层响应式，深层原样；适合读多写少的大表数据外壳。
- **`markRaw(obj)`**：**排除**响应式（跳过代理）。适合第三方实例（Mapbox/Chart/Editor），避免 Proxy 污染与性能损耗。

**坑位**
- `reactive(obj)` **不会**自动解包 `ref` 的嵌套字段（只在顶层才有 **ref unwrapping**）。
- 解构 `reactive` 会**丢失响应性**（应使用 `toRef(s)`）。
- `markRaw` 一旦标记，后续 `reactive` **不会生效**（深冻结这一段）。

---

## 3) `computed` vs `watch` / `watchEffect`

- **`computed(getter|{get,set})`**
  - **惰性** + **缓存**：依赖未变化不重新求值。
  - 适合**派生状态**、可双向（带 `set`）。
- **`watch(source, cb, {immediate,deep,flush})`**
  - 懒跟踪 **特定 source**（函数/Ref/数组），变化时执行 `cb(new,old, onCleanup)`。
  - 适合**副作用**：网络请求、写入本地存储、对外交互。
- **`watchEffect(effect, {flush})`**
  - **自动依赖收集**（首次立即执行），更像 `effect()` 的受控版；不关心旧值。
  - 适合快速绑定副作用；复杂场景改用 `watch`。

**清理时机**
```ts
watch(source, (n,o,onCleanup) => {
  const sub = subscribe(n);
  onCleanup(()=> sub.unsubscribe()); // 下次触发或销毁时调用
}, { flush: 'post' });
```

**选择指南**
- “值是 UI 的函数” → `computed`
- “变化时做事” → `watch`/`watchEffect`
- 多依赖、高频、需缓存 → `computed`；IO/副作用 → `watch`

---

## 4) 组件通信：`props/emits`、`v-model`（多 model、自定义）与类型推断

**基本约定**
- `props` 单向下行；修改请 `emit('update:x')`。
- 事件用 `defineEmits<{ (e:'save', p:User):void }>()` 做**类型化**与**校验**。

**`v-model`**
- 默认等价：`modelValue` + `update:modelValue`
- **多 model**：
```vue
<Child v-model:title="title" v-model:checked="checked" />
// 子组件
const model = defineModel<string>('title');
const checked = defineModel<boolean>('checked');
```
- 可带**修饰符**：`defineModel({ set(v, { modifiers }) { if (modifiers.trim) v=v.trim(); return v; } })`

**透传与 attrs**
- 未声明的 `props`/事件落入 `$attrs`，自动透传到根节点；多根节点组件需 `defineOptions({ inheritAttrs:false })` 并手动绑定。

---

## 5) Composition API vs Options API：取舍与可复用性

**Composition 优势**
- 逻辑以**功能维度**组织（而非生命周期分片）；更易抽成 **composable**。
- **TypeScript 友好**；靠类型推断与泛型构建可重用逻辑。
- 与 `script setup`/`<template>` 更贴合现代工具链（Vite/HMR/按需导入）。

**Options 优势**
- 上手简单；小组件/团队历史包袱下更直观。
- 对初学者，生命周期段落清晰。

**实践建议**
- 公共逻辑 → `useXxx()`（暴露状态与方法，内部可用 `effectScope` 与 `onScopeDispose` 管理资源）。
- 组件 UI 壳 + 组装 → Composition API。
- 团队过渡期：允许共存；新功能与中大型组件优先 Composition。

---

## 6) `<script setup>` 宏：`defineProps/withDefaults/defineEmits/defineExpose/defineSlots/defineModel`

- **编译期宏**，仅在 `<script setup>` 中使用；**不会**出现在运行时。
```vue
<script setup lang="ts" generic="T extends { id: string }">
const props = withDefaults(defineProps<{ list: T[]; color?: string }>(), { color: 'red' });
const emit = defineEmits<{ (e:'select', id:string):void }>();
const model = defineModel<string>(); // 默认 v-model
defineExpose({ focus: () => inputEl.value?.focus() });
const slots = defineSlots<{ item(props:{ row: T }):any }>();
</script>
```
- **类型优势**：`defineProps/defineEmits` 直接从泛型/字面量推断；`defineSlots` 为作用域插槽提供签名。
- **多 `v-model`**：`defineModel<'title'>()` 指定参数名。
- **注意**：宏在顶层调用；条件/循环中调用**无效**。

---

## 7) 生命周期钩子与 `nextTick`

- **核心挂载链**：`setup()` → `onBeforeMount` → **挂载** → `onMounted` → 更新（`onBeforeUpdate` → `onUpdated`）→ 卸载（`onBeforeUnmount` → `onUnmounted`）
- **路由钩子**（在组件内）：`onBeforeRouteLeave`、`onBeforeRouteUpdate`（需 Vue Router 4）。
- **`nextTick`**：等待**DOM 更新刷新**后回调（微任务队列）。
  - 读取/操作真实 DOM（测量、聚焦）→ 放在 `nextTick` 或 `onUpdated`。
  - 配合 `flush:'post'` 的 `watch` 可把副作用排在 DOM 更新之后。

```ts
import { nextTick, onMounted, onUpdated } from 'vue';
onMounted(async () => {
  await nextTick(); // DOM ready
  input.value?.focus();
});
```

---

## 8) `toRef` / `toRefs` / `customRef`：表单防抖与解构保持响应

- **解构保持响应**：`toRefs(reactiveObj)` 把每个属性变成 `Ref`，避免 `const {a}=state` 造成失联。
- **单字段引用**：`toRef(obj,'field')` 对非响应式对象也可创建可写 Ref（读写回写到原对象）。
- **`customRef`**：自定义 `.value` 的 **track/trigger** 行为，可实现**防抖/节流**等。

**防抖输入示例**
```ts
import { customRef } from 'vue';
function useDebouncedRef<T>(value:T, delay=200) {
  let t: any;
  return customRef<T>((track, trigger) => ({
    get() { track(); return value; },
    set(v) { clearTimeout(t); t = setTimeout(()=>{ value = v; trigger(); }, delay); }
  }));
}
const keyword = useDebouncedRef('');
watch(keyword, v => fetchList(v));
```

---

## 9) `provide/inject` 的响应式与只读约束

- **保持响应式**：直接 `provide('key', reactiveStore)` 或 `provide('key', computedValue)`；`inject` 得到的是**同一个引用**。
- **默认值**：`inject(key, defaultValue)`；或者 `inject(key, () => createStore(), true)` 用工厂延迟创建。
- **只读约束**：父层 `provide(readonly(store))`，子层只能读；或只暴露方法：
```ts
const CounterKey = Symbol('counter');
type CounterCtx = { count: Readonly<Ref<number>>; inc():void; }
provide<CounterCtx>(CounterKey, { count: readonly(count), inc });
const ctx = inject(CounterKey)!; // ctx.count.value 可读
```
- **避免深层耦合**：prefer 组合式 + 显式 props；`provide/inject` 用于“跨层级共享的基础设施”（i18n、主题、表单上下文等）。

---

## 10) `slots` / 作用域插槽：工作原理与 `v-slot` 用法

- **本质**：父将**函数**作为子组件的 `slots` 传入；子在渲染时调用这些函数并传入**插槽 props**。
- **默认/具名插槽**
```vue
<Child>
  <template #default="{ row }"> {{ row.name }} </template>
  <template #footer> 页脚 </template>
</Child>
```
- **语法糖**：`v-slot` / `#name`；**同一元素**不能同时 `v-for` 与 `v-if` 与 `v-slot` 混用到混乱，建议拆开。
- **性能**：插槽是函数调用，尽量在子组件内部做 **key 稳定** 与**最小化重渲染**；大表格场景结合**虚拟滚动**。
- **回退内容**：`<slot name="footer">缺省内容</slot>` 在父未提供时渲染回退。
- **类型化**（TS）
```ts
// 子
const slots = defineSlots<{ default(props:{ row: Row }): any; footer?: () => any }>();
```
---
## 11) 动态组件与缓存：`<component :is="...">`、`<KeepAlive>`（`include/exclude/max`）、`activated/deactivated`

**动态组件**
- `<component :is="CompOrTag">` 动态切换渲染目标；可传**组件对象/异步组件/字符串标签**。
- 切换时默认**卸载旧组件并销毁其状态**，除非被 `<KeepAlive>` 包裹。

**缓存（KeepAlive）**
```vue
<KeepAlive :include="/^Page(Detail|List)$/" :exclude="['HeavyEditor']" :max="10">
  <component :is="activeComp" />
</KeepAlive>
```
- `include/exclude`：按**组件名（`name` 选项）**匹配；支持字符串、数组、正则。
- `max`：LRU 策略；淘汰最久未用。
- 生命周期：首次挂载走 `mounted`；命中缓存再显示时触发 `activated`；被切走时触发 `deactivated`（不销毁）。销毁才触发 `unmounted`。

**实践**
- 缓存**表单页/详情页**等“可回退继续编辑”的路由；对“强状态”的第三方实例（编辑器/地图）要在 `activated`/`deactivated` 内显式**暂停/恢复**资源，防止后台占用。
- 组件名必须稳定，否则 `include/exclude` 失效；**组合式 API**下通过 `defineOptions({ name: 'PageList' })` 指定。

---

## 12) Vue Router 4：动态/嵌套/命名视图，守卫与 `scrollBehavior`

**路由声明**
```ts
const routes: RouteRecordRaw[] = [
  { path: '/user/:id(\\d+)', component: User, props: true },                  // 动态参数 + props 注入
  { path: '/home', components: { default: Home, aside: Aside } },             // 命名视图
  { path: '/admin', component: AdminLayout, children: [
      { path: 'users', component: Users },
      { path: 'roles', component: Roles }
  ]},
];
const router = createRouter({ history: createWebHistory(), routes,
  scrollBehavior(to, from, saved) {
    if (saved) return saved;                        // 返回浏览器记忆的滚动位置
    if (to.hash) return { el: to.hash, behavior: 'smooth' };
    return { top: 0 };
  }
});
```

**守卫顺序（简版）**
- 全局：`beforeEach` → 路由记录 `beforeEnter` → 组件内 `beforeRouteEnter` → 解析完毕 → `afterEach`。
- 组合式（组件内）：见第 13 题。

**实践**
- **鉴权/重定向**放在 `beforeEach`，配合**缓存的用户态**与**白名单路由**。
- 大型应用将**路由模块化**并**按需动态添加**（`router.addRoute`）以实现**权限路由**与**包拆分**。

---

## 13) 组合式路由守卫：`onBeforeRouteLeave` / `onBeforeRouteUpdate`

```ts
import { onBeforeRouteLeave, onBeforeRouteUpdate, useRoute } from 'vue-router';

onBeforeRouteLeave((to, from, next) => {
  if (hasUnsavedChanges.value) return next(false); // 阻止离开
  next();
});
onBeforeRouteUpdate((to, from, next) => {
  // 同一路由组件复用（仅 param/query 变），在此刷新数据而非销毁重建
  loadData(to.params.id as string).finally(() => next());
});
```

**要点**
- **复用组件**时（如 `/user/1` → `/user/2`），不会触发卸载，需用 `onBeforeRouteUpdate` 监听参数变更。
- 异步 `next()` 必须**始终终止**（成功/失败）以避免挂起。
- 与 `<KeepAlive>` 共用时：切换回缓存页面不会触发这些守卫；放置在**路由层**或在 `activated` 中自查刷新。

---

## 14) 异步组件与代码分割：`defineAsyncComponent` 选项

```ts
const AsyncChart = defineAsyncComponent({
  loader: () => import('./Chart.vue'),
  loadingComponent: Loading,
  errorComponent: LoadError,
  delay: 200,                // Loading 延迟，避免闪烁
  timeout: 15_000,           // 超时视为错误
  suspensible: true          // 可交给 <Suspense> 处理
});
```

**最佳实践**
- 路由级别用**路由懒加载**；组件级用 `defineAsyncComponent`。
- 加 `/* webpackChunkName: "chart" */` 或 Vite 的 `/* @vite-ignore */` 控制分包与动态导入。
- 为大型第三方库（图表/编辑器）单独切块，并用**可见性/交互**触发加载，降低首屏压力。

---

## 15) `Suspense`：与异步 `setup()` / 异步组件的协作

**机制**
- 子树中任何 `suspensible` 的异步任务（异步组件加载、`setup()` 内 `await` 的 Promise、`<Suspense>` 的 `#default` 槽）尚未就绪时，渲染 `#fallback`。
```vue
<Suspense>
  <template #default>
    <UserPanel /> <!-- setup 内 await: fetch user -->
  </template>
  <template #fallback>
    <SkeletonUser />
  </template>
</Suspense>
```

**要点**
- SSR 时，Suspense 会**等待数据**以输出完整 HTML；CSR 首屏可用 fallback 减少**感知等待**。
- 与 `Transition` 协作时注意**先后顺序**：先 Suspense 再 Transition，避免动画与占位切换抖动。
- 大量并发 `await` 建议用 `Promise.all` 并**将 IO 放在 composable**，便于测试与缓存（如 SWR）。

---

## 16) `Teleport`：渲染边界、焦点与滚动锁

**用途**
- 将子树渲染到**DOM 其他位置**（如 `body`），常用于模态/抽屉/通知：
```vue
<Teleport to="body">
  <Modal v-if="open" @close="open=false" />
</Teleport>
```

**可达性 & 交互**
- **焦点陷阱**：打开时将焦点移入模态首个可聚焦元素，关闭时**返回触发源**；键盘 `Esc` 关闭。
- **滚动锁**：给 `html, body` 添加 `overflow: hidden` 或使用**锁计数**避免嵌套模态相互覆盖。
- **ARIA**：`role="dialog" aria-modal="true"`；标注标题 `aria-labelledby`。

**注意**
- 逻辑仍在原父组件作用域内（事件/状态不变），仅**DOM 位置改变**。
- 3.5 的 `<Teleport defer>` 适配**目标容器延迟出现**（见 3.5 相关题）。

---

## 17) 自定义指令（Vue 3 生命周期）：`beforeMount`/`mounted`/`updated`/`unmounted`

```ts
const vFocus: Directive<HTMLElement, boolean> = {
  mounted(el, binding) { if (binding.value !== false) el.focus(); },
  updated(el, binding) { if (binding.value && !binding.oldValue) el.focus(); }
};
```

**指令值比较**
- `binding.value` / `binding.oldValue` 可用于**边沿触发**；
- 修饰符与参数：`v-xxx:arg.mod1.mod2="val"` → `binding.arg` / `binding.modifiers`。

**典型场景**
- DOM 级行为：聚焦/拖拽/粘贴/点击外部（click-outside）。
- 与组件化的取舍：**能组件就组件**（可复用、可测试），指令适合“轻量 DOM 行为”。

---

## 18) 性能优化：避免不必要渲染、`v-once`/`v-memo`、节流/去抖、DevTools

**渲染层**
- **数据结构不可变**（或浅不可变）：列表/表格更新时**创建新引用**以触发精准更新。
- 组件拆分 + `defineComponent({ inheritAttrs:false })` 减少无关更新。
- **`v-once`** 用于**完全静态**片段；**`v-memo`**（按依赖条件跳过重渲染）：
```vue
<div v-memo="[row.id, row.updatedAt]"> ... </div>
```

**事件层**
- 高频输入/滚动 → **去抖/节流**；优先用 `passive` 监听滚动（`@scroll.passive`）。

**工具**
- Vue DevTools Performance：记录 commit、组件树重渲染热区；
- 浏览器 Performance + `isolate` 某组件的 CPU/内存；按**二分法**定位瓶颈。

---

## 19) 列表渲染：`key` 语义、索引陷阱、虚拟滚动与稳定性

- **`key` 的本质**：辅助 **diff 的稳定定位**，避免错误复用；**不要用索引**（增删/排序会导致状态错乱：输入框、动画、过渡）。
- **分页/拖拽**：保证 `key` 来自**业务唯一标识**（id）；拖拽变更顺序时 `key` 不变，DOM 才能被正确移动。
- **虚拟滚动**：长列表用 `vue-virtual-scroller` 等；注意**行高可预测**或给出估值与占位，防止滚动抖动。
- **不可变数据** + 局部组件化（行/单元格）降低重渲染；对超大表格配合**分片渲染**（`requestIdleCallback`/批量挂载）。

---

## 20) 表单与受控/非受控：`v-model` 修饰符、组件的双向绑定规范

**原生表单**
```vue
<input v-model.trim="name" />
<input type="number" v-model.number="age" />
<input v-model.lazy="keyword" /> <!-- 失焦或回车触发 -->
```

**自定义组件的 `v-model`**
- 规范：接收 `modelValue`，在变更时 `emit('update:modelValue', val)`；多模形见第 4 题。
```vue
<script setup lang="ts">
const model = defineModel<string>({ set: v => v?.trim() ?? '' }); // 修饰符/过滤
</script>
```

**受控 vs 非受控**
- **受控**：值由父组件持有，子组件只发事件 → **可预测**、易记录/回滚；代价是**频繁更新**。
- **非受控**：子组件内部持有状态并在关键时机同步 → 可降低上层重渲染；但需提供**受控切换**能力（`defaultValue` + `modelValue?`）。
- **表单库协作**：提供 `blur/change/input` 事件、`validate()`/`reset()` 方法（`defineExpose`）以便外部驱动校验与提交。

---
## 21) 事件与修饰符：`.stop` / `.prevent` / `.capture` / `.passive` 等的语义与最佳实践

- **绑定等价**：`@event.mod1.mod2="fn"` ≈ `el.addEventListener('event', fn, options)` + 在回调里做 `stopPropagation/ preventDefault`。  
  - `.stop` → `e.stopPropagation()`（阻止冒泡）。  
  - `.prevent` → `e.preventDefault()`（阻止默认行为）。  
  - `.capture` → 以 **捕获阶段**监听（`{ capture:true }`），常用于“先截获再决定是否放过”。  
  - `.once` → `{ once:true }`。  
  - `.passive` → `{ passive:true }`，**承诺不调用 `preventDefault`**，浏览器可优化滚动/触摸。
- **键/系统修饰符**：`@keydown.enter.exact`、`@click.ctrl`，在模板层过滤无关事件。
- **滚动/触摸**：对 `scroll` / `touchstart`，若**不会**调用 `preventDefault`，优先用 `.passive`，减少合成滚动阻塞。**不要同时写 `.passive.prevent`**（冲突）。
- **委托**：高频列表中把监听挂到容器上（事件委托），再用 `e.target` / `closest` 识别目标，降低大量节点监听成本。

---

## 22) 条件/列表指令：`v-if` vs `v-show`；`v-for` 与 `v-if` 冲突

- **`v-if`**：**条件渲染**，会创建/销毁组件与 DOM，首次成本高；适合**不常切换**的块。
- **`v-show`**：只切换 `display: none`，不卸载；适合**频繁切换**的块（Tab/折叠）。
- **`v-for` 同节点 `v-if`**：在 Vue 3 上**不推荐**（会给出警告）。条件判断应：  
  1) 通过 **计算属性先过滤**列表：`visibleItems = items.filter(...)`；  
  2) 或用 `<template v-for="..."><li v-if="...">...</li></template>` 分离语义。  
  这样保证 `key` 稳定、diff 明确，避免状态错乱（输入框、过渡等）。

---

## 23) 样式隔离：`<style scoped>` 原理与 `:deep()` / `:slotted()` / `:global()`

- **隔离原理**：编译器为组件节点添加形如 `data-v-xxxx` 的属性，并把样式改写为  
  `.btn[data-v-xxxx] { ... }`，从而仅作用于本组件 DOM 子树。
- **工具选择**：  
  - `:deep(.class)`：**穿透**到子组件内部（生成后代选择器），谨慎使用，过多会引发样式耦合。  
  - `:slotted(.class)`：仅作用于**插槽内容**根节点（父组件控制传入内容样式）。  
  - `:global(.class)`：跳出作用域，等价全局样式（要有命名约定，防污染）。
- **风险与实践**：  
  - 复杂 `:deep()` 选择器会提升**匹配成本**与**特指度**；建议把可定制样式通过 **CSS 变量**/BEM 暴露。  
  - 使用 `scoped` 时的第三方库覆盖：优先**CSS 变量**或在库组件上提供 `class` 钩子而非全局覆盖。

---

## 24) 过渡与动画：`<Transition>` / `<TransitionGroup>`，类名与 rAF 协同

- **类名约定**（默认 `v-` 前缀，可自定义 `name`）：  
  - 进入：`v-enter-from` → `v-enter-active` → `v-enter-to`  
  - 离开：`v-leave-from` → `v-leave-active` → `v-leave-to`
- **JS 钩子**：`onBeforeEnter/ onEnter/ onAfterEnter/ onBeforeLeave/ onLeave/ onAfterLeave`，JS 控制时应在回调中调用 `done()`。
- **列表过渡**：`<TransitionGroup>` 需要**稳定的 `key`** 才能做移动（FLIP）。  
- **与 rAF 协作**：在过渡开始前用一次 `requestAnimationFrame` 触发布局，让浏览器正确计算 `from→to`，避免跳帧：
  ```ts
  onBeforeEnter(el) { el.style.opacity = '0'; }
  onEnter(el, done) {
    requestAnimationFrame(() => {
      el.style.transition = 'opacity .2s ease';
      el.style.opacity = '1';
      el.addEventListener('transitionend', done, { once:true });
    });
  }
  ```
- **性能**：优先用 **合成属性**（transform/opacity），避免触发布局（width/height/left/top）。

---

## 25) 批量更新与异步队列：`nextTick` 与 `watch.flush`

- **批量机制**：同一“tick”内对响应式数据的多次修改会被**去重合并**，在**微任务**中统一刷新组件。  
- **`nextTick`**：等待当前刷新批次**完成并 DOM 更新**后执行：
  ```ts
  count.value++; await nextTick();  // 读取真实 DOM
  ```
- **`watch` 刷新时机**：  
  - `flush: 'pre'`（默认）：**渲染前**触发，适合**派生状态**或纠正输入。  
  - `flush: 'post'`：**渲染后**触发，适合需要访问**最新 DOM**（测量/滚动）。  
  - `flush: 'sync'`：**立即**触发（危险，可能引发递归/抖动），仅在**精确控制**下使用。
- **避免布局抖动**：读写 DOM 分离（多读→一次写），DOM 依赖放 `'post'` 或 `nextTick`。

---

## 26) 错误处理：`errorCaptured`、`app.config.errorHandler`、未处理 Promise

- **组件边界**：  
  ```ts
  onErrorCaptured((err, instance, info) => {
    report(err, info); return false; // 返回 false 阻止向上冒泡
  });
  ```
- **应用级**：  
  ```ts
  const app = createApp(App);
  app.config.errorHandler = (err, instance, info) => report(err, info);
  window.addEventListener('unhandledrejection', e => report(e.reason, 'unhandledrejection'));
  ```
- **分类与脱敏**：统一结构化上报（url/route/version/userAgent/stack/extra），过滤 PII；对**频繁错误采样**，致命错误全量。
- **用户可见性**：在路由/页面级加**错误边界**（显示“重试/反馈”），对请求级错误做**幂等重试**与**降级**。

---

## 27) SSR 与水合（Hydration）：链路、mismatch 成因与规避

- **链路**：Server 渲染 **HTML + 状态注水** → 客户端加载同构 bundle → **hydrate** 绑定事件并复用 DOM。  
- **mismatch 常见原因**：  
  - **非确定输出**：`Date.now()/Math.random()`、依赖 `window`/`navigator` 的分支在 SSR 与 CSR 不同。  
  - **时区/本地化**差异（服务器与客户端格式化不同）。  
  - **异步数据**在服务器与客户端时序不一致。
- **规避**：  
  - 把浏览器专属逻辑放 `onMounted`；或在模板用 `v-if="mounted"`（首屏闪动可配 skeleton）。  
  - 所有格式化（时间/货币）使用**统一库与时区**；把最终渲染所需数据**注水**到页面。  
  - 日志监控 `hydration-mismatch` 警告并定位组件；极端场景下使用（3.5）受控差异抑制策略（见 3.5 题）。

---

## 28) Nuxt 3 基本机制：`useAsyncData` / `server routes` / 缓存与 `<ClientOnly>`

- **数据获取**（在服务端/客户端两端运行）：  
  ```ts
  const { data, pending, refresh } = await useAsyncData('post', () => $fetch('/api/post/1'), { server:true, default:()=>null });
  ```
  - 支持键控缓存、失效与 `stale-while-revalidate`。  
- **Server Routes（Nitro）**：在 `server/api/*.ts` 下写后端接口，部署到边缘/Node，享受**同构上下文**与运行时自动树摇。
- **缓存与失效**：`cachedEventHandler` / `defineCachedFunction`；按 `ttl/tags` 失效，或显式 `revalidate`。  
- **`<ClientOnly>`**：包裹**仅在浏览器可用**的组件（依赖 DOM/第三方仅浏览器库），避免 SSR 崩溃与 hydration mismatch。
- **路由与布局**：基于文件系统；`middleware` 支持路由级守卫与权限跳转。

---

## 29) Pinia 状态管理：核心概念、`storeToRefs`、插件/持久化、Vuex 迁移

- **定义与使用**
  ```ts
  export const useCart = defineStore('cart', {
    state: () => ({ items: [] as Item[] }),
    getters: { total: s => s.items.reduce((a,i)=>a+i.price,0) },
    actions: {
      add(i:Item) { this.items.push(i); },
      async checkout() { await api.pay(this.items); this.items = []; }
    }
  });
  const cart = useCart();
  const { items, total } = storeToRefs(cart); // 保持解构后的响应式
  ```
- **特点**：**无 mutations**；action 可异步且可返回值；`storeToRefs` 解决解构失联；**模块化/动态注册**简单。
- **插件与持久化**：通过 Pinia 插件注入 `persist`/`logger`/`undo` 等；持久化用 `pinia-plugin-persistedstate` 或自写（`localStorage/IndexedDB`）。  
- **SSR**：支持注水/脱水（`pinia.state.value`），配合 Nuxt 3 自然工作。
- **从 Vuex 迁移**：  
  - `mutations` → 合并进 `actions`；  
  - `mapState/getters` → `storeToRefs`；  
  - 中间件/插件接入点对齐到 Pinia 插件 API。

---

## 30) 大型表格/列表性能：虚拟滚动、不可变数据、单元格组件化

- **虚拟滚动**：只渲染可视区域与小缓冲（`overscan`）；要求可预测行高或提供估算器；滚动时**避免同步测量**。
- **不可变数据**：增删改返回**新引用**，配合稳定 `key`，减少非目标行重渲染；分页/排序在计算层完成而非模板层 `filter/sort`。
- **行/单元格组件化**：把复杂单元格拆为**纯渲染组件**，用 `props` 传数据，避免在大父组件上反复 diff；对列级渲染器用**函数插槽**。
- **交互优化**：  
  - 事件**委托**到列表容器；  
  - 悬浮/选中状态用 `:class` 切换而非频繁内联样式；  
  - 大表编辑走**批处理**（debounce/commit 阶段统一写入）。  
- **观测与兜底**：首屏渲染与滚动掉帧（`Performance`/DevTools）；低端设备开关**简化模式**（禁用阴影/渐变/复杂 formatter）。

---
## 31) TypeScript 最佳实践：给 `props`/`emits`/`slots`/`expose` 打类型，泛型组件与 Volar 坑位

**`<script setup>` 宏类型化**
```vue
<script setup lang="ts" generic="T extends { id: string; name: string }">
const props = defineProps<{ list: T[]; page?: number }>();
const emit = defineEmits<{ (e:'select', row:T): void; (e:'page-change', p:number): void }>();
defineExpose<{ focus(): void }>();
const slots = defineSlots<{ default(p:{ row:T }): any; empty?: () => any }>();
</script>
```
- `generic` 让 SFC 成为**泛型组件**：调用方 `<MyList<User> :list="users" />`。
- `withDefaults(defineProps<...>(), { page: 1 })` 为可选项提供默认值并保留推断。

**Volar / 推断常见问题**
- 顶层宏**必须**顶层调用；条件/循环中调用会失去推断。
- 复杂 `emits` 建议用**函数签名联合**（见上），避免字典式写法丢失参数类型。
- 插槽类型化后，父组件使用时**获得提示**：`<MyList v-slot="{ row }">...</MyList>` 有 `row: T`。
- 给 `defineExpose` 标注接口，避免在父组件处 `any`。

---

## 32) 组件设计规范：单向数据流、不变性、事件命名与“可组合/可测试”API

**核心约定**
- **单向数据流**：只读 `props` + `emit('update:x')` 回传；避免在子组件直接改 `props`。
- **不变性**：列表/对象更新**返回新引用**，利于 diff & 性能追踪。
- **事件命名**：`update:modelValue`（配合 `v-model`）、`change/confirm/cancel` 等领域语义；不要把业务语义埋在 `click` 里。

**可组合/可测试**
- UI 组件薄、逻辑下沉到 **composable（`useXxx`）**：
```ts
// usePagination.ts
export function usePagination(totalRef: Ref<number>) {
  const page = ref(1); const size = ref(20);
  const pages = computed(() => Math.ceil(totalRef.value / size.value));
  function next(){ if(page.value < pages.value) page.value++; }
  return { page, size, pages, next };
}
```
- 组件只组装状态与模板；**测试时**可直接对 composable 做单元测试。

---

## 33) 测试：VTU + Vitest 的渲染/交互、`flushPromises`、mock 组合式与 Pinia

```ts
// Button.spec.ts
import { mount, flushPromises } from '@vue/test-utils';
import { describe, it, expect, vi } from 'vitest';
import Button from './Button.vue';

it('emits click', async () => {
  const wrapper = mount(Button, { props: { label: 'OK' } });
  await wrapper.find('button').trigger('click');
  expect(wrapper.emitted('click')).toBeTruthy();
});

it('async flow uses flushPromises', async () => {
  const wrapper = mount(Button);
  await wrapper.find('button').trigger('click'); // 内部发起 Promise
  await flushPromises(); // 等待微任务链
  expect(wrapper.text()).toContain('Done');
});
```

**Mock 组合式函数**
```ts
vi.mock('@/composables/useFetch', () => ({
  useFetch: vi.fn(() => ({ data: ref({ ok: true }), pending: ref(false) }))
}));
```

**Pinia 测试**
```ts
import { createTestingPinia } from '@pinia/testing';
const wrapper = mount(App, { global: { plugins: [createTestingPinia({ createSpy: vi.fn })] } });
// store.actions 会被 spy，state 可直接设值
```

**端到端（可选）**：Playwright/Cypress 做**交互/网络**分层；配合 **MSW**（Mock Service Worker）稳定网络。

---

## 34) 构建与工程化（Vite）：别名/环境变量、按路由分包、产物分析与体积预算

**Vite 配置要点**
```ts
// vite.config.ts
export default defineConfig({
  resolve: { alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) } },
  define: { __BUILD_TIME__: JSON.stringify(new Date().toISOString()) },
  build: {
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vue: ['vue','vue-router','pinia'],
          vendor: ['axios','lodash-es']
        }
      }
    },
    chunkSizeWarningLimit: 800
  }
});
```
- 环境变量：`.env.production` → `import.meta.env.VITE_API_BASE`（仅 `VITE_` 前缀会注入）。
- **路由懒加载**：`component: () => import('./views/Detail.vue')`。
- 产物分析：`rollup-plugin-visualizer` 或 Vite `--profile`，建立**体积预算**（每包/总包上限），CI 超限报警。
- 预构建优化：`optimizeDeps.include/exclude` 缩短冷启动；大库拆 **独立 chunk** 便于浏览器缓存。

---

## 35) 安全与可达性：`v-html`、XSS、防护；表单/ARIA；Teleport/过渡中的焦点

**XSS 基线**
- **默认插值已转义**；只有 `v-html` 需要谨慎。对外部 HTML 必须**先消毒**（如 DOMPurify）：
```vue
<div v-html="sanitizedHtml" />
```
```ts
import DOMPurify from 'dompurify';
const sanitizedHtml = computed(() => DOMPurify.sanitize(rawHtml.value));
```
- 禁止把用户输入拼到内联事件/URL（`javascript:`）。

**可达性**
- 表单：`<label for="id">` 对应输入的 `id`；提供 `aria-invalid`、`aria-describedby` 指向错误文本。
- 组件可聚焦元素用 `tabindex` 控制顺序；键盘操作（Esc/Enter/Space）与语义角色（`role="button"`）一致。

**模态/Teleport/过渡**
- `role="dialog" aria-modal="true"`，开启时**聚焦陷阱**，关闭时把焦点还给触发源。
- 过渡期间**不要**改变可聚焦元素数量导致焦点丢失；必要时在 `onAfterEnter` 再聚焦。

---

## 36) Reactive Props Destructure（Vue 3.5）：响应式解构与边界

**3.5 新增**：对 `defineProps()` 的**直接解构**保持响应式（编译期改写为 `toRef`）。
```ts
const { page, pageSize } = defineProps<{ page: number; pageSize: number }>(); // page/pageSize 是 Ref
watch(page, v => ...);
```
**边界与取舍**
- **只作用于 props 顶层字段**；**嵌套对象**仍需 `toRef(props, 'user')` 或在子层使用 `toRef`。
- `rest` 解构（`const { a, ...rest } = props`）里，`rest` 是 **普通对象**，**不保持响应**。
- 迁移旧代码：已有 `toRefs(props)` 可逐步替换为解构；复杂场景保持显式 `toRef` 可读性更强。

---

## 37) `useTemplateRef()`（Vue 3.5）：模板引用的类型安全与列表场景

**目的**：在 `<script setup>` 中以**类型安全**方式声明模板 `ref`，避免 `as unknown as`。
```ts
// 组件/元素引用
const modal = useTemplateRef<InstanceType<typeof Modal>>('modal');
const box = useTemplateRef<HTMLDivElement>('box');
onMounted(() => modal.value?.open());
```
```vue
<Modal ref="modal" />
<div ref="box" />
```
**列表场景**
- 在 `v-for` 中使用同名 `ref`，得到**按渲染顺序的数组**引用：
```ts
const itemsRef = useTemplateRef<HTMLLIElement[]>('items');
```
```vue
<li v-for="i in list" :key="i.id" ref="items">{{ i.name }}</li>
```
- **失效时机**：卸载时置 `null`（或空数组）；访问需在 `onMounted/nextTick` 之后。  
- 与旧法兼容：仍可 `const el = ref<HTMLElement|null>(null); ref="el"`，`useTemplateRef` 主要解决**类型推断**与**可读性**。

---

## 38) `onWatcherCleanup()`（Vue 3.5）：更干净的副作用清理范式

**对比**：`watch()` 的第三参回调 `onCleanup` 仅在回调作用域可用；3.5 提供**外部注入**方式。
```ts
import { watch, onWatcherCleanup } from 'vue';

watch(query, async (q) => {
  const ctrl = new AbortController();
  onWatcherCleanup(() => ctrl.abort()); // 组件卸载或下次触发时执行
  await fetch(`/api?kw=${q}`, { signal: ctrl.signal });
});
```
**最佳实践**
- 取消网络请求、移除事件监听、停止动画/定时器，都用 `onWatcherCleanup` 注册。
- 对 `watchEffect` 同样适用；多重异步时确保**每次触发都创建新资源并清理旧资源**。

---

## 39) `<Teleport defer>`（Vue 3.5）：目标容器延迟与时序/可达性/滚动锁

**行为**
- `defer` 使 Teleport **在目标容器出现后再挂载**（或在同一刷新批次延后），避免在 SSR/异步注入容器时的时序问题。
```vue
<Teleport to="#portal" defer>
  <Modal v-if="open" />
</Teleport>
```
**实务要点**
- 目标容器（如布局框架的 `#portal`）动态渲染时不再闪烁/错位。
- 与**焦点管理**配合：在 `onMounted`/`onActivated` 后进行聚焦，确保容器已存在。
- **滚动锁协调**：等 Teleport 完成后再设置 `overflow: hidden`，避免在错误容器上锁滚。

---

## 40) `useId()`（Vue 3.5）与 SSR 一致性：可预测 ID 与可达性

**作用**：生成**在 SSR 与 CSR 一致**的稳定 ID，避免 hydration mismatch。
```ts
const inputId = useId();           // e.g. "v-123-1"
const hintId  = useId();
```
```vue
<label :for="inputId">Name</label>
<input :id="inputId" />
<p :id="hintId">Please input your name</p>
<input aria-describedby="hintId" />
```
**价值**
- 表单/可达性：`label for`、`aria-*` 关系稳定。
- 复用组件：保证同一组件多实例的 ID **不冲突**，且在 SSR/CSR 一致。
- 避免用 `Math.random()`/时间戳等**非确定性** ID。

**注意**
- `useId` 的序号与组件渲染顺序相关；不要在**条件渲染**中混用生成顺序与跨层级消费，必要时把 ID 下沉为 `prop`。

---
## 41) 受控差异抑制（Vue 3.5）：`data-allow-mismatch` 的适用场景与风险

**场景**  
SSR → CSR 水合时，某些节点的内容**天然不一致**（如时间戳、A/B 随机、用户本地化）。3.5 提供受控标记让这些节点**忽略 mismatch**，避免整棵子树强制重建。

**用法（模板）**
```html
<!-- 仅忽略该元素内容的不一致（文本/属性） -->
<span data-allow-mismatch>
  {{ new Date().toLocaleTimeString() }}
</span>
```

**边界与风险**
- 仅**抑制水合阶段的告警与重建**；并不保证后续一致性。  
- 过度使用会掩盖**真实逻辑错误**（如服务器与客户端业务分支不一致）。  
- SEO/可达性：如果搜索引擎抓到的是 SSR 内容、用户看到的是 CSR 改写内容，两者长期不一致会影响可用性指标与可信度。

**建议**
- 优先：让首屏内容**可确定**（把随机/时钟放到 mounted 后）。  
- 次选：把确实不一致的**最小节点**标记 `data-allow-mismatch`。  
- 极端：纯客户端组件（Nuxt 的 `<ClientOnly>`）或延后水合（见第 50 题）。

---

## 42) `defineModel()`（3.4）进阶：多 `v-model`、修饰符类型化、受控/非受控切换

**多 `v-model`**
```vue
<!-- 父 -->
<MyInput v-model="value" v-model:filter.trim="filter" />
```
```ts
// 子
const model = defineModel<string>();                // 等价 v-model
const filter = defineModel<string, { trim?: boolean }>('filter', {
  set(v, { modifiers }) { return modifiers.trim ? v.trim() : v; }
});
```

**类型化要点**
- 泛型参数 `<T, M>()`：`T` 是值类型，`M` 描述修饰符集合（可选键布尔）。  
- 多模型需确保**名称稳定**（`defineModel<'title'>()`），避免与 `props` 冲突。

**受控 vs 非受控**
- 受控：仅依赖 `modelValue`，内部不持久状态。  
- 非受控：内部持状态，暴露 `defaultValue` & `modelValue?`，并在**外部介入时**同步（双轨）。  
- 切换策略：检测 `modelValue` 是否为 `undefined` → 非受控；否则受控。变更时始终 `emit('update:modelValue')`，外部可选择接管或忽略。

---

## 43) 泛型组件（3.3+）：SFC 泛型、`withDefaults` 配合与 Volar 限制

**声明与使用**
```vue
<script setup lang="ts" generic="T extends { id: string }">
const props = withDefaults(defineProps<{ rows: T[]; page?: number }>(), { page: 1 });
const onSelect = defineEmits<{ (e:'select', row:T): void }>();
</script>

<!-- 调用方 -->
<MyTable<typeof User> :rows="users" @select="handle" />
```

**实践要点**
- `generic="T ..."` 只能在 `<script setup>` 顶层；**不能**条件声明。  
- `withDefaults` 不会破坏 `props` 推断（推荐为可选项提供默认值）。  
- Volar 限制：有时**调用点不显式传泛型**将退化为 `any`；为公共组件导出 `Props<T>` 并提供**示例用法**可缓解。  
- 复杂泛型插槽时，结合 `defineSlots`（见第 44 题）把**插槽参数**也类型化。

---

## 44) `defineSlots()` 类型化：作用域参数、联合/可选槽与错误提示边界

**类型化示例**
```ts
const slots = defineSlots<{
  // 必填默认插槽，传入每行数据
  default(props: { row: Row; index: number }): any;
  // 可选“空态”插槽
  empty?: () => any;
  // 联合参数示例
  status?(p: { kind: 'ok'; data: Row } | { kind: 'error'; message: string }): any;
}>();
```
**提示与边界**
- 父组件在 `v-slot` 上能获得**精确参数提示**；参数不匹配会有**编译期类型错误**。  
- 槽**是否提供**仅在运行时可知，因此“必填槽缺失”的问题仍需**开发时约定/断言**。  
- 建议为**可选槽**提供合理回退内容，避免 UI 空洞。

---

## 45) 渲染性能微优化：`v-memo`/`v-once`、`shallowRef`/`markRaw` 组合策略

**工具选型**
- `v-once`：永不更新，适合**完全静态**片段（纯文案、图标）。  
- `v-memo="[deps...]"`：依赖未变则**跳过整块子树重渲染**，用于高代价区域（表格复杂单元格、富文本渲染）。  
- `shallowRef`：只在**引用变更**时触发（如图表实例、大对象）。  
- `markRaw`：**排除响应式**，减少代理负担（第三方实例）。

**组合示例**
```vue
<div v-memo="[row.id, row.updatedAt]">
  <ChartView :instance="chart" :data="row.series" />
</div>
```
```ts
const chart = shallowRef<Chart|undefined>(undefined);
// 第三方实例
import { createChart } from 'lib';
onMounted(() => (chart.value = markRaw(createChart(canvasEl))));
```

**度量优先**
- 先用 Vue DevTools / Performance 面板定位**热区**，再加点优化。  
- 避免“微优化成瘾”：过多 `v-memo` 会降低可读性并掩盖**数据结构问题**（例如频繁新建对象）。

---

## 46) Effect 调度与刷新策略：`watch.flush` 取舍，避免布局抖动与重复计算

**选择矩阵**
- `flush: 'pre'`（默认）：**渲染前**执行，适合**派生状态**、校正输入。  
- `flush: 'post'`：**渲染后**，适合需要**读取真实 DOM**（测量、滚动）。  
- `flush: 'sync'`：**立即**执行，谨慎；仅用于**需要与事件完全同步**的场景（如阻止连点）。

**反抖与合并**
```ts
const size = ref({ w:0, h:0 });
const onResize = useDebounceFn(() => {
  size.value = measure(el); // 只在稳定后写
}, 100);
watch(() => route.fullPath, onResize, { flush: 'post' });
```

**避免重复**
- 避免在同一依赖链上**多层 watch/computed**互相触发；将昂贵计算**提升为 computed**并在多处复用。  
- DOM 读写分离：**读在一批，写在一批**；必要时 `await nextTick()` 再写。

---

## 47) `effectScope` / `onScopeDispose`：Composable 的资源域与跨组件清理

**封装可释放的逻辑**
```ts
import { effectScope, onScopeDispose } from 'vue';

export function useLiveQuery(q: Ref<string>) {
  const scope = effectScope(true); // detached
  const state = scope.run(() => {
    const data = ref<string[]>([]);
    const stop = watch(q, async v => { data.value = await api.search(v); }, { immediate: true });
    onScopeDispose(() => stop());   // 当 scope 停止时清理
    return { data };
  })!;
  return {
    ...state,
    stop: () => scope.stop()        // 允许外部手动结束
  };
}
```

**实践**
- 在**组件销毁**时，scope 内注册的副作用会自动清理。  
- **嵌套作用域**：在复杂逻辑里拆分子 scope，按需独立 `stop`。  
- 测试可直接 `stop()` 验证**资源释放**，防内存泄漏。

---

## 48) 自定义渲染器：`createRenderer` 的 Host Config 与职责边界

**思想**：VNode 负责**描述**，Host Config 负责**落地**（如何创建元素、插入、设置属性等）。

**最小 Canvas 渲染器（示意）**
```ts
import { createRenderer } from '@vue/runtime-core';

const { render, createApp } = createRenderer({
  createElement: (type) => ({ type, children: [] as any[] }),
  insert: (child, parent, _anchor) => parent.children.push(child),
  remove: (child, parent) => parent.children.splice(parent.children.indexOf(child),1),
  createText: (text) => ({ type: 'text', text }),
  setElementText: (el, text) => (el.text = text),
  parentNode: (node) => node.parent,
  nextSibling: () => null,
  patchProp: (el, key, _prev, next) => (el[key] = next)
});
// 然后在“commit phase”把虚拟树绘制到 Canvas/WebGL/终端
```

**要点**
- **最小必要接口**：`createElement/insert/remove/patchProp/createText/setElementText/parentNode/nextSibling`。  
- 事件/属性映射由 `patchProp` 自行定义（如把 `x/y/fill` 映射为 Canvas 绘制指令）。  
- SSR 需要单独的 **custom SSR renderer**；否则仅 CSR。

---

## 49) Web Components 集成：`defineCustomElement`、Shadow DOM 与样式定制

**从组件导出自定义元素**
```ts
import { defineCustomElement } from 'vue';
import VueButton from './VueButton.ce.vue';
customElements.define('vue-button', defineCustomElement(VueButton));
```
- `*.ce.vue` 中使用常规 SFC 写法；**样式默认进 Shadow DOM**，隔离外部样式。  
- `props` 映射为 **属性/属性观察**；事件以 **`CustomEvent`** 发出（`el.addEventListener('change', e => ...)`）。

**样式定制**
- Shadow DOM 内部可通过 `::part()` 暴露可定制钩子；外部通过 `::part(btn)` 定制。  
- 传入内容经 `slot`，可用 `::slotted()` 选择；注意其**选择器限制**（仅作用于 slotted 根）。

**陷阱**
- SSR 直出 Web Components 受限（需原生支持）；  
- 不同框架的事件/属性大小写规范差异（建议使用 **kebab-case 属性** 与明确的 `emits`）。

---

## 50) SSR 的“选择性/延迟水合”：按交互/可见性/空闲时机激活

**目标**：降低主线程压力、改善 TTI/INP，同时保持 SEO 的 **HTML 可见**。

**策略**
- **按可见性**：在组件可见时水合（IntersectionObserver）。  
- **按交互**：首次交互（`pointerdown/keydown`）才水合按钮/小部件。  
- **按空闲**：`requestIdleCallback`/`scheduler.postTask` 在空闲期激活非关键区块。  
- **岛架构**：仅对“可交互岛”注入客户端代码，其他纯静态。

**Nuxt 示例**
```vue
<!-- 仅客户端渲染（非 SSR）-->
<ClientOnly>
  <HeavyChart />
</ClientOnly>
```
更进一步可用社区/框架的 **lazy-hydrate** 指令（可见/空闲触发）。  
**权衡**：延迟过久会让用户“点了没反应”；需给出**占位与加载反馈**，并对关键交互设置**水合上限**（如 1s 内必须激活）。

---
