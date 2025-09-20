# ES / CSS / DOM

## 1) 事件循环与任务队列：宏任务/微任务的执行顺序；浏览器与 Node 的差异？

**核心模型**
- **执行栈（call stack）**：同步代码入栈→出栈。
- **任务（task）**：来自不同**任务源**（定时器、网络、UI 事件等）的**宏任务**队列；一次事件循环迭代只取**一个**宏任务执行。
- **微任务（microtask）**：`Promise.then/catch/finally`、`queueMicrotask`、`MutationObserver`。**每次宏任务结束、执行栈清空**后会进入**微任务检查点**：持续清空微任务队列直到为空。
- **渲染时机（浏览器）**：清空微任务后，若样式/布局/合成有脏标记且到达**渲染机会**，执行一帧渲染。连续大量微任务会**推迟渲染**，造成白屏或掉帧。

**浏览器顺序（抽象）**
```
同步脚本 → 微任务清空 → 渲染（可选） → 下一个宏任务（如定时器回调） → … 循环
```

**Node.js 顺序与差异**
- 基于 **libuv** 的阶段循环：`timers → pending → idle/prepare → poll → check → close`。
- **`setTimeout/Interval`** 在 `timers` 阶段；**`setImmediate`** 在 `check` 阶段。
- Node 也有微任务队列（V8 的 PromiseJobs），**每个阶段的回调执行完**都会清空微任务。
- **`process.nextTick`** 队列优先级**高于**微任务：每个回调后先清空 `nextTick`，再清空微任务；过度使用会“饿死”I/O。

**常见结论**
- 浏览器/Node 都是：**宏任务回调 → 清空微任务 →（浏览器可能渲染/Node 进入下一阶段）**。
- **`nextTick > microtask(then)`**（Node）；**`then > setTimeout`**（浏览器）。
- 大量微任务会推迟绘制；需要“让出主线程”时用 `setTimeout/MessageChannel`。

---

## 2) Promise 与 async/await 的语义：错误传播、并发控制（all/allSettled/race/any）与取消（AbortController）如何设计？

**状态机与调度**
- Promise 一生只会从 `pending` → `fulfilled/rejected`，**不可逆**。
- `then`/`catch` 回调放入**微任务**；返回值 `x` 通过**“Promise 解析过程”**（展开 thenable）决定下游状态。
- `await p` 等价于在当前函数暂停并在 `p.then` 的微任务中恢复；`await` 的 reject 等价 **`throw`**。

**错误传播**
- 链式传播遵循“就近捕获”原则：任一环 `throw` 或 `reject` 都走最近的 `catch`。
- 未捕获拒绝：浏览器触发 `unhandledrejection`；Node 触发 `unhandledRejection`。生产应统一监听上报。

**并发原语的本质**
- `Promise.all`：**全要型**；任意一个 reject → 直接 reject（fail-fast）；成功结果保持**输入顺序**。
- `Promise.allSettled`：**全收集**；永不 reject，返回 `[{status, value|reason}]`。
- `Promise.race`：**首个定型**即返回（无论成败）。
- `Promise.any`：**首个 fulfill** 即返回；全失败抛 `AggregateError`。

**取消机制（协作式）**
- Promise 设计上**不可取消**；通过 **`AbortController`/`AbortSignal`** 传递取消信号：
  - 原生支持：`fetch`、Web Streams、部分 Node 核心 API。
  - 自定义异步应订阅 `signal.aborted`，尽早停止副作用，抛/返回 `AbortError`。
- 组合：`AbortSignal.timeout(ms)`（Node/WHATWG）或自建“超时控制器”实现请求超时与并发**背压**。

**并发限制（背压示例思想）**
- 维护“在飞集合”与 `Promise.race(inflight)`；超过 **concurrency** 时等待最早完成的 promise 再发下一个，直至全部完成。

---

## 3) 模块体系：ESM vs CommonJS 的差异、tree-shaking 生效条件、动态 import() 与跨环境打包要点？

**ESM（ECMAScript Modules）**
- **静态依赖图**：顶层 `import` 可在构建期解析。
- **活绑定（live binding）**：导入的是变量视图，导出方更新导入方可见。
- **`<script type="module">`**/TLA（Top-Level `await`）：模块初始化可以异步。
- **并行加载**：浏览器可并行加载多个模块，避免“同步阻塞”。

**CJS（CommonJS）**
- 运行时 `require`，导出是对象属性（非变量绑定）；同步加载；`module.exports`/`exports` 同引用（误用会断开）。
- **互操作陷阱**：  
  - `require(esm)` 得到 **命名空间对象**；  
  - `import cjs` 的默认导出取决于打包器/`__esModule` 约定，易出现 **Dual Package Hazard**（同包被 CJS/ESM 双载）。

**Tree-Shaking 生效条件**
- 上游与入口都为 **ESM**（能静态分析导入）。
- 无顶层**副作用**或在 `package.json` 中正确标注 `"sideEffects": false`（或列出有副作用文件白名单）。
- 压缩/摇树器开启 DCE（Terser/esbuild/SWC）并能识别 `/*#__PURE__*/`。
- 避免动态形态阻断：`require()` 分支、`eval`、带副作用的 getter/setter。

**动态 `import()` 与分包边界**
```js
const mod = await import('./feature.js'); // 返回 Promise，天然形成分片边界
```
- 有利于按路由/交互延迟加载；但**完全动态路径**（`import(`./${name}.js`)`）会让打包器无法静态切片，需通过枚举或虚拟模块路由表。

**跨环境打包要点**
- `package.json` 使用 **`exports` 条件** 明确 `import`/`require`/`browser`/`node`，避免双实现不一致。
- 类型入口：`types` 或 `exports['.'].types`。
- 避免同包双加载：锁定消费者只走一种入口；或以 `exports` 约束解析路径。

---

## 4) 原型与继承：原型链解析、class 的本质、super/this 绑定在继承中的行为？

**原型链**
- `obj.[[Prototype]]` 指向其构造器的 `prototype`；属性查找沿链向上直至 `Object.prototype` 或 `null`。
- `hasOwnProperty` 仅检测自有属性；`in` 会沿原型链。

**`class` 本质**
- 语法糖：构造函数 + 原型方法；方法为**不可枚举**；类体内默认严格模式。
- 静态字段挂在构造器本身（`ClassName.staticField`），原型方法在 `ClassName.prototype`。

**`super` / `this` 细节**
- `super.m()` 查找目标是**原型链上的同名方法**，但其中的 `this` 仍指向**当前实例**。
- 子类构造器中**必须**在使用 `this` 前调用 `super()`（初始化 `this`）。
- `super` 依赖 `[[HomeObject]]`（定义时绑定环境）；把方法解构出来调用会丢失 `super` 语义并报错。
- 类字段中的箭头函数方法“绑定”了实例 `this`（每实例一份）；原型方法共享同一函数对象，更省内存。

---

## 5) this 绑定五规则（默认/隐式/显式/new/箭头函数）及常见踩坑案例？

**五规则与优先级**
1. **`new` 绑定**：`new F()` 创建新对象并将其作为 `this`，优先级最高。
2. **显式绑定**：`fn.call/apply/bind(obj)` 指定 `this`。
3. **隐式绑定**：`obj.fn()` → `this === obj`；**丢失绑定**：`const f = obj.fn; f();` 回落到默认绑定。
4. **默认绑定**：非严格模式下 `this === window/global`；严格模式为 `undefined`。
5. **箭头函数**：无 `this`，**词法捕获外层** `this`，无法用 `call` 修改。

**细节与陷阱**
- `bind` + `new`：`new (fn.bind(obj))()` 时，`this` 指向新实例（`bind` 的 `thisArg` 被忽略），**但**预绑定的实参仍生效。
- 事件监听：`el.addEventListener('click', obj.handle)` 内的 `this` 是元素而非 `obj`；要 `obj.handle = obj.handle.bind(obj)` 或在类中用**箭头实例方法**。
- 解构丢失接收者：`const {slice} = Array.prototype; slice.call(arrayLike)`；直接 `slice(arrayLike)` 会报错或 `this` 错误。

---

## 6) 闭包与内存：闭包导致的泄漏风险、循环引用、WeakMap/WeakSet/WeakRef 适用场景？

**闭包与可达性**
- 闭包将其**词法环境**保持在堆上；只要闭包仍可从**GC 根**（全局对象、事件监听、定时器等）到达，环境就不会回收。
- 现代 GC 能处理**循环引用**；真正导致泄漏的是**意外持有**（全局 Map/数组、未清理监听器/定时器）。

**典型泄漏**
- 缓存结构（如 `Map<string, Node>`）未在节点销毁时清理。
- 将大对象捕获在长寿命回调中（如全局单例事件总线），组件卸载后未移除监听。

**Weak 族适用**
- `WeakMap/WeakSet`：键为对象的弱引用；当**仅**剩弱引用时可被 GC 回收。适合“给对象打私有标签/缓存派生数据”，无需手动清理键。
  ```js
  const meta = new WeakMap();
  function tag(node) { meta.set(node, {/*…*/}); } // node 释放时条目自动消失
  ```
- `WeakRef` + `FinalizationRegistry`：实现**可回收缓存**/软引用；不保证调用时机，**不要**用于业务正确性，只做“尽力缓存”。

**治理策略**
- 统一 `dispose()` 约定：清理计时器、取消订阅、置空大字段。
- DevTools：对比堆快照、查找“站点化增长”对象与持有路径（Retainers）。

---

## 7) 深浅拷贝：structuredClone 与 JSON.parse/stringify 的差异；可转移对象与 MessageChannel 传递大数据？

**`structuredClone`**
- 规范级深拷贝：支持 `Date`、`RegExp`、`Map/Set`、`ArrayBuffer`、`Blob/File`、`URL`、循环引用；**不支持**函数、DOM 节点、`WeakMap/WeakSet`。
- 保留**原型链**与属性描述中的可枚举值（不可拷贝 getter 的执行结果，而是其数据值）。
```js
const cp = structuredClone(obj);
```

**JSON 技术**
- 仅限可 JSON 化类型；会丢失 `undefined`/`Symbol`/函数、`Date` → 字符串、**不能处理循环引用**。适用于“纯数据快照”。

**Transferable（零拷贝所有权转移）**
- 通过 `postMessage(payload, [transferList])` 将 `ArrayBuffer`、`MessagePort`、`ImageBitmap`、`OffscreenCanvas` 的底层所有权**转移**给目标上下文：源侧变为 **detached**，拷贝开销≈0。
```js
const {port1, port2} = new MessageChannel();
port1.postMessage(buf, [buf]); // buf 在 port2 一侧接收，源侧被分离
```

**大数据通道**
- 利用 `MessageChannel` / `Worker` + Transferable 传输大二进制；或使用 `ReadableStream` 分块传输，降低主线程阻塞。

---

## 8) 函数式编程在前端中的应用：不可变、柯里化、管道/组合对可维护性与性能的影响？

**不可变（Immutability）**
- 改变用“新值替换旧值”表达状态变更，有利于**时间旅行/快照比较/并发控制**。
- 代价：频繁复制；缓解手段：**结构共享**（持久化数据结构）或 **Immer** 的 Proxy 录制生成补丁。

**柯里化/组合**
- 柯里化将多参函数拆为链式一参函数，便于**重用部分参数**与组合。
- 组合/管道将“数据→变换”串接（`pipe(f,g,h)(x)`），简化复杂逻辑，降低偶然复杂度。
- 性能权衡：热路径中避免过度高阶函数/临时闭包；可用**融合循环**（一次遍历完成 map/filter/reduce）或使用“transducer”思想。

**副作用管理**
- 把 I/O/DOM/网络视为副作用，集中在边界层（effects）；纯函数只负责数据变换，便于测试与并行。

---

## 9) 元编程：Proxy/Reflect/Object.defineProperty 的对比与实现响应式/校验/拦截的思路？

**`Object.defineProperty`（ODP）**
- 定义数据/访问器属性，能拦截**读写**单键，但对**新增键/删除键/数组索引变更**无感；需要对每个键都定义 getter/setter（Vue2 模型）。
- 对数组需要**重写原型方法**（`push/splice`）以感知变更。

**`Proxy`**
- 在**对象级别**拦截：`get/set/has/ownKeys/deleteProperty/defineProperty` 等几乎所有基本操作。
- 响应式实现思路：`get` 时依赖收集（track）→ `set` 时派发通知（trigger）；`ownKeys`/`getOwnPropertyDescriptor` 用于遍历/`for..in` 依赖。
- 需要遵守**对象不变量**：例如不可扩展对象的不可配置键必须在 `ownKeys` 中出现；违反会抛错。
- 性能：相较直访慢；热路径避免层层 Proxy 嵌套。

**`Reflect`**
- 与 Proxy trap 一一对应的底层原语；在 handler 内部用 `Reflect.xxx` 调用**默认语义**，避免二次触发 trap 或不一致。
```js
const observed = new Proxy(target, {
  get(t,k,rec) { track(t,k); return Reflect.get(t,k,rec); },
  set(t,k,v,rec){ const old = t[k]; const ok = Reflect.set(t,k,v,rec); if(ok && old!==v) trigger(t,k); return ok; }
});
```

---

## 10) 迭代协议：Iterator/Iterable、Generator、异步迭代器的工作机制与场景？

**同步迭代协议**
- **Iterable**：实现 `obj[Symbol.iterator]()`，返回一个 **Iterator**。
- **Iterator**：实现 `next()`（可选 `return/throw`），返回 `{ value, done }`。
- `for..of`/解构/展开运算符都消费 Iterable，并在中断时调用迭代器的 `return()` 进行**关闭**（释放资源）。

**Generator**
- 语法生成迭代器对象：可 `yield` 多次；`gen.return(val)` 用于提前终止；`gen.throw(err)` 向内部抛错。
- `yield*` 可**委托**到另一个 iterable，拼接迭代。

**异步迭代**
- **AsyncIterable**：实现 `obj[Symbol.asyncIterator]()`，返回有 `next(): Promise` 的迭代器；消费语法 `for await..of`。
- 场景：分页 API、流式网络响应（Streams）、长轮询事件序列。
```js
async function* pages(fetchPage){ let c=null; do{ const r=await fetchPage(c); yield* r.items; c=r.next; } while(c); }
```

---

## 11) 类型与相等：== vs ===、Object.is、NaN/0/-0、typeof/instanceof 与 Symbol.toStringTag？

**比较家族**
- `===`：严格相等，不做类型转换。
- `==`：抽象相等，会触发复杂的**类型转换规则**（如对象先变原始值，再参与数值/字符串比较）。业务中应避免。
  - 典例：`[] == ![]` → `true`（`![] → false → 0`；`[] → '' → 0`）。
- `Object.is`：与 `===` 几乎一致，但区分 `+0`/`-0`，且认为 `NaN` 与 `NaN` 相等。

**`NaN/±0`**
- `NaN !== NaN`；用 `Number.isNaN` 判断。
- `1/+0 === Infinity`，`1/-0 === -Infinity`；数值运算会暴露差异。

**`typeof/instanceof`**
- `typeof null === 'object'` 是历史遗留；除了函数，其余对象 typeof 均为 `"object"`。
- `instanceof` 基于**原型链**；跨 realm（iframe）因 `Ctor.prototype` 不同而失效。可改用**能力检测**或 `Symbol.hasInstance` 自定义。

**`Symbol.toStringTag`**
- 影响 `Object.prototype.toString.call(x)` 的标签，可被伪装；例如自定义迭代器设置 `toStringTag` 为 `"Iterator"`。因此类型守卫优先使用专用 API（`Array.isArray`、`ArrayBuffer.isView`）。

---

## 12) 性能优化：去抖/节流、事件委托、V8 优化陷阱（隐藏类、稀疏数组）、热路径与逃逸分析的影响？

**事件频控**
- **去抖（debounce）**：在触发停止后的等待窗执行一次。适合输入联想。
- **节流（throttle）**：固定间隔最多执行一次。适合滚动/窗口 resize。
- 两者实现要点：保持**同一闭包内的状态**（计时器/上次时间戳），避免创建新函数导致解绑失败。

**事件委托**
- 借助冒泡将大量子元素监听合并到父节点，降低监听器数量与内存开销；通过 `e.target.closest('.item')` 做命中判定。

**V8 优化关键**
- **隐藏类（Hidden Class）与 IC（Inline Cache）**：对象属性初始化顺序一致、字段集合稳定，调用点就能形成**单形/少形态**内联缓存，JIT 内联更激进。
  - 反例：同一函数中处理结构不同的对象 → **多形/超形**（megamorphic）导致去优化。
- **稀疏数组**：带洞数组或巨大索引会退化为字典模式，访问慢、内存浪费。
- **Number vs 非数值**混用导致元素种类变化，打破**元素种类专一性**（packed → holey → dictionary）。
- **逃逸分析**（JIT 优化的一部分）：可将未逃逸到堆外的对象标量替换、栈上分配；一旦对象**逃逸**（跨闭包/存入数组/返回值），优化机会减少。热路径中宜减少临时对象与闭包。

**渲染相关**
- 将 DOM **读**与**写**分离；合并多次读/写，避免“强制同步布局”（如 `offsetWidth` 后立即写 `style`）。
- 在 `requestAnimationFrame` 驱动下批量进行测量与写入；使用 `transform/opacity` 以走**合成层**，必要时谨慎使用 `will-change`（限定时长，防止内存暴涨）。

```js
// 典型读写分离
requestAnimationFrame(() => {
  const h = el.offsetHeight;     // read
  el.style.transform = `translateY(${h}px)`; // write
});
```
---
## 13) 盒模型与 `box-sizing`、`margin` 合并、BFC（Block Formatting Context）

**盒模型本质**
- **`content-box`（默认）**：`width/height` 仅指内容盒；padding/border 参与可视尺寸，但不计入 `width`。
- **`border-box`**：`width/height` 含内容 + padding + border；计算更直观，能减少“算宽度”错误。
```css
/* 工程基线：统一边框盒，稳定尺寸计算 */
*, *::before, *::after { box-sizing: border-box; }
```

**`margin` 合并机制（collapsing）**
- 只在**垂直方向**的块级格式化上下文（BFC）中发生；兄弟块的相邻外边距会取**最大值**（含正负叠加规则）。
- 父子之间也可能合并：当父没有边框/内边距/建立新 BFC 时，父的上/下边距会与第一个/最后一个子元素合并。
- **如何阻断**：给父加 `border/padding`，或让父/子建立 **BFC**（如 `overflow: hidden|auto|clip`、`display: flow-root`、`contain: layout/paint`）。

**BFC 原理与作用**
- **是谁**：BFC 是块级布局的**独立格式化上下文**，内部盒子的布局不影响外部。
- **如何建**：`float!=none`、`overflow` 非 `visible`、`display: inline-block|table-cell|flow-root`、根元素、`position: fixed`、`contain: layout/paint` 等。
- **能做什么**：
  - 清除浮动（父容器高度被内部浮动撑开）。
  - 阻断 `margin` 合并。
  - 控制文字环绕/自适应布局边界。
```css
.wrapper { overflow: auto; }     /* BFC 包裹浮动 */
.footer  { display: flow-root; } /* 清除浮动 + 阻断合并 */
```

---

## 14) 层叠与优先级、`!important`、`z-index` 与层叠上下文（Stacking Context）

**特指度规则**
- 计算四元组：行内样式 (1,0,0,0) > ID (0,1,0,0) > 类/属性/伪类 (0,0,1,0) > 元素/伪元素 (0,0,0,1)。
- `!important` 只在**同源级别**比较时提升声明优先级，滥用会让维护失控。

**层叠上下文（SC）**
- **触发条件**：根元素、`position`（非 `static`）且 `z-index` 非 `auto`、`opacity<1`、`transform/filter/perspective`、`mix-blend-mode`、`will-change`、`isolation:isolate`、`contain: paint` 等。
- **关键特性**：不同 SC 之间**相互独立**；子元素再高的 `z-index` 也无法越过祖先所在 SC 的层级。
- **典型坑**：元素加了 `transform` 形成新 SC 后，把弹窗/下拉“压住”。解决：移除导致 SC 的属性、提层结构、或在弹窗根上创建更高层 SC。
```css
.header { transform: translateZ(0); } /* 新建 SC，可能挡住 fixed 弹层 */
.modal  { position: fixed; z-index: 1000; } /* 仍可能被 header 的 SC 挡住 */
```

---

## 15) 布局选型：Flex vs Grid 的本质差异与坑点

**算法对比**
- **Flex（单维流）**：沿主轴分配剩余空间，交叉轴按对齐规则布置；适合导航条、按钮组等**一维**场景。
- **Grid（二维网格）**：显式/隐式轨道 + 自动放置；适合看板、卡片栅格等**二维**场景。

**常见陷阱**
- Flex 项默认 `flex-shrink: 1` → 容器变窄时子项会压缩；文本容器需 `min-width: 0` 允许换行，否则受“自动最小内容尺寸”影响导致溢出。
```css
.flex-item { min-width: 0; }   /* 允许文本收缩换行 */
```
- `flex-basis` 与 `width`：当 `flex-basis` 非 `auto` 时优先于 `width`；很多“宽度不生效”其实是被 `flex-basis` 覆盖。
- Grid 轨道被内容撑开：使用 `minmax(0,1fr)` 而非 `1fr`，并给单元设置 `min-width:0`，避免受 `min-content` 约束。
```css
.grid { display:grid; grid-template-columns: minmax(0,1fr) minmax(0,1fr); }
.grid > .cell { min-width: 0; }
```

**选型指引**
- 只需“一排/一列自适应对齐” → Flex。
- 需要“二维分区/对齐与对称栅格” → Grid。
- 混用：外层 Grid 区块化，内层 Grid/Flex 组织内容。

---

## 16) 定位与居中：`position` 语义与通用居中技法

**定位语义**
- `relative`：保留空间，仅视觉偏移。
- `absolute/fixed`：脱离文档流，基于**包含块**定位；`fixed` 参考视口。
- `sticky`：在滚动到阈值前为普通流，越阈值后相对滚动容器粘住；受父层 `overflow` 裁剪与滚动容器影响。

**居中套路（尺寸未知也适用）**
```css
/* Flex 居中（首选） */
.center { display:flex; justify-content:center; align-items:center; }

/* 绝对居中 + transform */
.abs-center { position:absolute; left:50%; top:50%; transform: translate(-50%,-50%); }

/* Grid 居中（place-* 语义化） */
.grid-center { display:grid; place-items:center; }
```
- 多行文本垂直居中（单行可用 `line-height`；多行用 Flex/Grid）。

---

## 17) 单位与尺寸：`px/%/em/rem/vw/vh` 与移动端根字号/视口

**度量原理**
- `px`：CSS 像素，和物理像素通过 DPR 映射。
- `%`：相对包含块对应属性（宽相对包含块宽，高相对包含块高）。
- `em/rem`：`em` 相对当前元素字体；`rem` 相对根字体（适合做全局比例）。
- `vw/vh`：相对视口；移动端地址栏折叠会改变 `vh`。现代提供 `svh/lvh/dvh`（小/大/动态视口）以提升稳定性。

**根字号策略**
- PC 保持 16px；移动端可 `:root { font-size: 62.5%; } /* 1rem≈10px */` 或直接 16px，配合 `clamp()` 流式调整。
```css
:root { font-size: clamp(14px, 1.6vw, 16px); }
```

---

## 18) 响应式与移动端：`<meta viewport>`、断点规划、流式媒体与安全区域

**视口与安全区域**
```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```
- 齐刘海屏安全区：`padding-top: env(safe-area-inset-top)` 等，避免内容被遮挡。

**断点与策略**
- 移动优先：以内容断点（组件在何宽度换行）定义 `@media (min-width: …)`，而非设备型号。
- 图片/视频：`max-width:100%`、`object-fit: cover`；首屏大图搭配 `content-visibility: auto`、懒加载降压。

**流式排版**
- 使用 `clamp()`/`minmax()`/`calc()` 平滑缩放字体与间距，减少“硬断点”跳变。

---

## 19) 选择器与伪类/伪元素：可维护性与性能边界

**选择器要点**
- `:nth-child(an+b)` 基于兄弟顺序；`:nth-of-type` 仅计同标签。
- `:not()` 提升表达力，但避免过度嵌套造成可读性差。
- `::before/::after` 适合装饰性内容（非语义），减少额外 DOM。

**性能原则**
- 匹配从右到左进行；优先**类选择器**与扁平结构，避免层层后代选择器。
- 低特指度策略：BEM/utility-first，减少“相互加码”的覆盖战。

---

## 20) 图像与图标：语义、密度与格式优化

**选择语义**
- 内容图像 → `<img>`（有 `alt`、可被无障碍读取与 SEO 收录）。
- 纯装饰 → `background-image`（不进入可访问树）。

**响应式与密度**
```html
<img src="cover-1x.jpg"
     srcset="cover-1x.jpg 1x, cover-2x.jpg 2x"
     sizes="(min-width: 800px) 800px, 100vw"
     width="800" height="500" alt="">
```
- 给定 `width/height` 或 `aspect-ratio` 以预留占位，降低 CLS。
- 背景图用 `image-set()` 适配多密度；格式优先 **AVIF/WebP**，并提供回退。

**图标策略**
- **内联 SVG**：可控颜色/尺寸、无模糊；复用用 `<symbol>`/雪碧或组件化封装。

---

## 21) 动画与过渡性能：合成层、`transform/opacity`、`will-change` 的边界

**原则**
- 动画只改变 **compositor 可处理** 的属性：`transform/opacity`；避免触发布局与重绘。
- `will-change` 可提前分层，但会增内存与绘制成本；仅在动画前短期启用，并在结束后移除。
```css
.card { transition: transform .2s ease, opacity .2s ease; }
.card:hover { transform: translateY(-2px); }
```
- 预算：60FPS → 每帧约 16.7ms。JS、样式、布局、绘制任一阶段过长都会掉帧。用 Performance/Rendering 工具定位瓶颈。

---

## 22) CSS 模块化与规范：BEM、预处理、PostCSS、CSS Modules

**BEM 设计**
- 低特指度、语义清晰的命名：`block__element--modifier`；避免深层后代选择器。

**预处理 & PostCSS**
- Sass/LESS 提供变量/嵌套/混入；**限制嵌套层级**，避免输出高特指度。
- PostCSS：`autoprefixer`、`postcss-preset-env` 将新特性降级；在构建期完成兼容处理。

**CSS Modules**
- 通过哈希类名实现**本地作用域**；跨组件变量用 **CSS 自定义属性** 或“设计令牌包”，而不是复制 Sass 变量。
- 组织层次：ITCSS 或 `@layer`（base → components → utilities）减少覆盖战。

---

## 23) 字体与稳健性：`@font-face`、预加载、`font-display`、减少 FOUT/FOIT 与 CLS

**加载控制**
- `font-display: swap|optional`：首屏优先显示回退字体，加载后替换；`optional` 在弱网下可不替换，避免闪变。
- 关键字体 **预加载**：
```html
<link rel="preload" as="font" href="/fonts/inter-400.woff2" type="font/woff2" crossorigin>
```

**稳定排版**
- 使用字体度量覆盖，减少替换时的行高跳变：
```css
@font-face {
  font-family: Inter;
  src: url(/fonts/inter.woff2) format("woff2");
  font-display: swap;
  ascent-override: 90%;
  descent-override: 22%;
  line-gap-override: 0%;
}
```
- 或设置 `font-size-adjust` 改善回退字体的 x-height 匹配。

**体积与子集**
- 子集化（只打包所需字形/权重）；懒加载非首屏字重；禁用伪粗斜：`font-synthesis: none;`。

---

## 24) CSS 自定义属性（变量）与主题：级联/继承、`color-mix()`、新色彩空间（Lab/LCH）与回退

**变量的计算模型**
- 自定义属性参与**级联与继承**，值在**使用点**计算（非预处理器替换）。这使得“运行时主题切换/动态皮肤”成为可能。
```css
:root { --bg: #fff; --fg: #111; --brand: #3b82f6; }
[data-theme="dark"] { --bg:#0b0f19; --fg:#d1d5db; --brand:#60a5fa; }
.app { color: var(--fg); background: var(--bg); }
.btn { background: var(--brand); }
```

**主题切换与首屏一致性**
- SSR/HTML 内联注入首选主题（含媒体查询 `prefers-color-scheme` 的默认值）以避免“闪白/闪黑”；随后允许用户覆盖。
- 主题存储（`localStorage`）与系统偏好合并时，优先用户显式选择。

**`color-mix()` 与色彩空间**
- `color-mix(in lch, var(--brand), white 20%)` 得到更符合人眼感知的混色；LCH/Lab 在部分环境需要回退。
- **回退策略**：
  - 在构建时预计算常用混色并输出常规十六进制；
  - 运行时加 `@supports (color: color(display-p3 1 1 1))` / `@supports (color: lch(50% 0 0))` 分支启用高色域/新空间。
```css
.btn { background: #3b82f6; } /* 回退 */
@supports (color: lch(50% 0 0)) {
  .btn { background: color-mix(in lch, #3b82f6, white 20%); }
}
```

**跨 Shadow DOM 的主题**
- 通过宿主暴露入口：`::part/::theme` 支持有限，常见做法是在 `:host` 上定义并从外部传入变量：
```css
:host { --card-bg: var(--bg); }
.card { background: var(--card-bg); }
```

**性能注意**
- 变量解析很快，但**深层嵌套与频繁改变根变量**会触发样式重计算；局部主题（作用于容器）优于全局根频繁切换。
---
## 25) 事件模型与委托：捕获/目标/冒泡、停止传播与 `passive` 监听

**事件流三阶段**
1. **捕获（capturing）**：`window → document → ... → target`，捕获阶段监听需 `{ capture: true }`。
2. **目标（at target）**：事件到达目标元素。
3. **冒泡（bubbling）**：`target → ... → document → window`。大多数事件支持冒泡（如 `click`），也有不冒泡的（如 `mouseenter`、`focus`）。

**停止传播与默认行为**
- `event.stopPropagation()`：阻断后续**更外层**的监听器（但同一节点上其余监听器仍会被调用）。
- `event.stopImmediatePropagation()`：阻断**当前节点剩余监听器**，并阻止后续冒泡/捕获。
- `event.preventDefault()`：取消默认行为（若该事件可取消，`event.cancelable === true`）。

**`passive` 的影响**
- `{ passive: true }` 表示监听器**不会调用** `preventDefault()`；浏览器可提前滚动，避免触发**滚动阻塞**警告并提升滑动性能（尤其是 `touchstart`/`touchmove`）。
- 若确需阻止滚动（如自实现滑动容器），应确保监听器**非 passive**并仅在必要时调用 `preventDefault()`。

**委托（事件代理）**
- 将大量子元素的监听合并到祖先节点，通过 `event.target.closest('.item')` 判断命中，降低监听数量/内存占用。
- 注意**停止传播**的组件（如第三方库）可能“截断”事件，委托层需兜底（如在捕获阶段监听）。

**Pointer 事件简述**
- 统一鼠标/触控/笔：`pointerdown/up/move/cancel`。可通过 `pointerId` 跟踪多指；搭配 `touch-action` 控制浏览器手势（如 `pan-x|pan-y`）。

---

## 26) 表单与输入：原生校验、拦截与 IME 合成事件

**提交拦截与数据收集**
- 阻止默认提交，使用 `FormData` 收集字段：
```js
form.addEventListener('submit', (e) => {
  e.preventDefault();
  const data = Object.fromEntries(new FormData(form));
  // 发送 data
});
```
- 也可监听 `formdata` 事件（在表单即将提交时触发），便于统一追加字段：
```js
form.addEventListener('formdata', (e) => e.formData.append('token', getToken()));
```

**Constraint Validation API（约束校验）**
- 原生属性：`required`, `min`, `max`, `pattern`, `type=email/url/number` 等。
- 读取/控制状态：`input.checkValidity()`, `input.reportValidity()`, `input.setCustomValidity(msg)`（设置后 `:invalid` 生效，清空设空字符串）。
- 体验建议：失焦或提交时再提示，避免打字阶段打断；对移动端指定 `inputmode`/`autocomplete` 提升软键盘与自动填充体验。

**IME（输入法合成）与事件序**
- 合成输入阶段会触发 `compositionstart → compositionupdate* → compositionend`，期间 `keydown/keyup` 与 `input` 语义不同。
- 在需要“逐字限制”的场景（如提及、代码编辑），应忽略合成中间态：`if (e.isComposing) return;` 或在 `compositionend` 后处理。

**`beforeinput` 与内容可编辑**
- `beforeinput` 可在输入即将发生时获知**意图**（`inputType`），可用来实现富文本限制/撤销队列，比 `keypress` 更可靠。
- `contenteditable` 结合 `beforeinput`、`input`、`selectionchange` 实现轻量编辑器；注意跨浏览器差异与粘贴清洗。

---

## 27) DOM 操作与性能：`DocumentFragment`、模板与读/写批处理

**批量插入**
- 使用 `DocumentFragment` 或 `<template>`：
```js
const frag = document.createDocumentFragment();
for (const item of data) frag.append(renderItem(item));
list.append(frag);
```
- `<template>` 的 `content` 是文档片段，可克隆后一次性插入。

**避免强制同步布局（layout thrashing）**
- 会触发**同步布局**的读操作示例：`offsetWidth/Height`, `getBoundingClientRect()`, `getComputedStyle()`, 读取 `scrollTop/Client*` 等。
- 原则：**先读后写**，把多次读/写批量化，或使用 `requestAnimationFrame`：
```js
requestAnimationFrame(() => {
  const h = el.offsetHeight;          // read
  el.style.transform = `translateY(${h}px)`; // write
});
```

**CSS contain / content-visibility**
- `contain: layout paint` 或 `content-visibility: auto` 可将容器变成独立布局与绘制上下文（近似“局部 BFC + 局部绘制”），显著降低大列表/长文档的布局成本（配合 `contain-intrinsic-size` 提供占位）。

---

## 28) 渲染流水线：样式→布局→绘制→合成；避免同步布局与昂贵重绘

**流水线阶段**
1. 样式计算（style）
2. 布局（layout）：计算盒子几何。
3. 绘制（paint）：生成图层的位图。
4. 合成（compositing）：图层在合成器线程上组合。

**优化准则**
- 动画/过渡仅改 `transform/opacity` → 走**合成器**，避免 layout/paint。
- 合理使用**分层**（`will-change: transform`、`transform: translateZ(0)`），但注意内存成本与栅格化开销。
- 批量 DOM 更新，降低样式失效与重排次数；大节点树更新考虑**键控重排**（如虚拟 DOM 的 key）。

**强制同步布局的形成**
- 在浏览器尚未完成前一批写操作的样式/布局计算时执行**读取几何** API，会迫使其立即计算（flash-layout）。解决：读写分离、合并读取。

---

## 29) 观察者 API：`MutationObserver` / `IntersectionObserver` / `ResizeObserver`

**MutationObserver（DOM 变化）**
- 监听子树结构/属性/文本变更；适合做**自定义元素内部响应**或对第三方 DOM 注入做兜底。
```js
const mo = new MutationObserver((list) => { /* 批处理变更 */ });
mo.observe(root, { childList: true, subtree: true, attributes: true });
```
- 注意：回调异步（微任务）；避免在回调里再次大量改 DOM 造成抖动。

**IntersectionObserver（可见性/曝光）**
- 监听目标与根交叉区域变化；用于**懒加载**、**曝光上报**、**无限滚动**：
```js
const io = new IntersectionObserver((es) => {
  for (const e of es) if (e.isIntersecting) load(e.target);
}, { root: null, rootMargin: '200px 0px', threshold: [0, 0.5, 1] });
```

**ResizeObserver（尺寸变化）**
- 监听盒尺寸变化（布局后）；适合自适应组件/图表重绘。注意在回调中不要同步设置会再次触发布局的样式，避免循环（必要时使用旗标/`requestAnimationFrame`）。

**组合常见场景**
- 懒加载图片：Intersection 触发加载 → 加载后用 Resize 调整占位。
- 自定义富组件：Mutation 观测插槽变更 + Resize 适配容器。

---

## 30) 剪贴板 / 拖拽 / 文件：权限与安全模型

**Clipboard（异步 API）**
- `navigator.clipboard.writeText/readText` 需 HTTPS 与用户激活（部分浏览器策略更严），可能触发权限请求。
- 复杂富内容：`navigator.clipboard.write([new ClipboardItem({ 'text/html': blob })])`；读取富内容更受限（隐私风险）。

**拖拽（Drag & Drop）**
- 启用拖入：目标容器的 `dragover` 中 `e.preventDefault()`，否则不会触发 `drop`。
- 数据通道：`DataTransfer`（`e.dataTransfer.files` / `getData/setData`）。对**外来 HTML** 落地前需**净化**（DOMPurify）并限制 scheme。
```js
dropZone.addEventListener('drop', (e) => {
  e.preventDefault();
  for (const f of e.dataTransfer.files) upload(f);
});
```

**文件选择与目录**
- `<input type="file" multiple accept="image/*">`；（实验特性）`webkitdirectory` 选择目录并上传。
- 大文件：使用切片（`Blob.slice`）+ 并发上传与校验；或 `ReadableStream`/Service Worker 做分块。

**沙箱与隔离**
- 第三方编辑器/富文本源建议 iframe sandbox 或严格 CSP，避免剪贴板注入脚本。

---

## 31) 历史与导航：`pushState/replaceState`、`popstate`、滚动恢复与 Navigation API

**History API（SPA 基础）**
- `history.pushState(state, '', url)`：改变地址栏不刷新；`replaceState` 替换当前条目。
- 监听返回/前进：`window.onpopstate = (e) => { /* 依据 e.state 渲染 */ }`。
- 滚动恢复：`history.scrollRestoration = 'manual'` 自行管理；或在路由切换时保存/恢复 `scrollTop`。

**页面过渡与缓存**
- `beforeunload` 仅在离开站点时触发；**BFCache**（回退缓存）可使返回极快，需避免在页面挂载时做与 BFCache 冲突的操作（如解绑必要事件、使用 `unload`）。

**Navigation API（新，支持逐步推进）**
- `navigation.navigate(url, { history: 'push' })` + `navigation.addEventListener('navigate', handler)` 可统一拦截导航、过渡与滚动恢复，逐步替代 History + `popstate`。在不支持的浏览器回退到现有路由方案。

---

## 32) 存储与同源：`localStorage/sessionStorage/IndexedDB`、跨页同步与 `SharedArrayBuffer` 前置条件

**存储选型**
- `localStorage`：同步、主线程阻塞、小容量（~5–10MB）、按源隔离；适合轻量首选项。
- `sessionStorage`：同 `localStorage` 但仅当前标签页会话。
- **IndexedDB**：异步、事务型、可存对象/二进制；适合大数据缓存（如离线资源、查询缓存）。

**跨标签页同步**
- `storage` 事件：在其他标签页对 `localStorage` 修改时触发（不在本页触发）。
- `BroadcastChannel`：同源任意标签页/Worker 间**实时通信**，更简洁：
```js
const bc = new BroadcastChannel('auth');
bc.postMessage({ type: 'logout' });
bc.onmessage = (e) => { if (e.data.type === 'logout') signOut(); };
```

**`SharedArrayBuffer` 启用条件（跨源隔离）**
- 必须满足：
  - `Cross-Origin-Opener-Policy: same-origin`
  - `Cross-Origin-Embedder-Policy: require-corp`（或 `credentialless` 策略）
- 所有跨源子资源需要 CORS 或 CORP 头，否则被阻止嵌入。

---

## 33) Web 安全：XSS/CSRF/Clickjacking、CSP/Trusted Types、富文本消毒边界

**XSS（跨站脚本）**
- 输入→输出通道：HTML、属性、URL、JS 上下文各自的**转义规则不同**。
- 前端原则：**默认转义**；对 `innerHTML`/框架的“原样渲染指令”必须经过**白名单消毒**（DOMPurify），并限制 URL 协议（`http/https/mailto/tel`）。

**CSP（内容安全策略）**
- 基线：`script-src 'self' 'nonce-xyz' 'strict-dynamic'`，禁用内联/`eval`；第三方脚本需通过 nonce/哈希授权。
- `frame-ancestors 'none'` 或 `X-Frame-Options: DENY` 防点击劫持（Clickjacking）。

**Trusted Types**
- 将“可执行字符串”收口到受信工厂（`policy.createScriptURL(...)`），阻断 DOM 注入类 XSS。前端与框架需适配（避免内联拼接）。

**CSRF**
- Cookie 模式：`SameSite=Lax/Strict`；对跨站请求校验 `Origin/Referer` 或使用 CSRF token 双提交策略；非幂等操作禁用 GET。

**消毒边界**
- DOMPurify 可去除事件属性、脚本、危险 URL，但**无法**理解业务语义（如“某字段不该出现外链”）；对模板协议、内联样式中的 `url()`、SVG 中的 `xlink:href` 也需注意。

---

## 34) Service Worker：缓存策略、离线/在线与版本治理

**注册与生命周期**
- `register('/sw.js')` → `install`（预缓存）→ `activate`（清理旧缓存）→ 接管 `fetch`。
- **更新模型**：SW 文件内容变更 → 下载新 SW（`installing`）→ 等待旧 SW 释放 → `waiting`。调用 `skipWaiting()` + 客户端 `postMessage({ type: 'SKIP_WAITING' })` 可加速切换。

**典型缓存策略**
- 静态资源：`cache-first`（命中即返回，后台更新）。
- API 数据：`network-first`（离线时回退缓存）或 `stale-while-revalidate`（先返回缓存，再后台刷新）。
- 版本治理：以**资源清单（manifest）hash** 做缓存键；发布回滚只需切换清单指针，旧缓存按版本清理。

**注意事项**
- 避免缓存**带鉴权**的私有数据（或采用分区缓存）；对跨域请求注意 CORS 与 `opaque` 响应不可复用的问题。
- 监控更新：页面监听 `registration.updatefound`、向 SW 发送 `message` 通知用户“可更新”。

---

## 35) 性能与观测：Web Vitals、`PerformanceObserver`、Long Tasks 与交互延迟

**Web Vitals（字段监测）**
- **LCP**（Largest Contentful Paint）：最大内容渲染时间；受服务器/资源加载/渲染影响。
- **CLS**（Cumulative Layout Shift）：布局抖动；受未占位图片/广告/字体替换影响。
- **INP**（Interaction to Next Paint）：交互到下一次渲染的总时延（取代表性分位数），替代 FID。

**采集与上报（示意）**
- 可使用轻量实现或库（如 web-vitals）；原生 `PerformanceObserver` 也可手动观测：
```js
const po = new PerformanceObserver((list) => {
  for (const e of list.getEntries()) {
    // e.name/e.startTime/e.duration
  }
});
po.observe({ type: 'longtask', buffered: true }); // Long Tasks > 50ms
```
- `PerformanceObserver` 可观测 `paint`（FP/FCP）、`largest-contentful-paint`、`layout-shift`、`longtask`、`resource` 等条目。对 INP 需监听 `event`/`first-input` 及后续交互延迟（建议用现成库计算）。

**优化路线**
- **LCP**：关键资源预加载（HTML 内联 `link rel=preload/modulepreload`）、CDN、压缩/子集字体、首屏关键 CSS 内联、图像尺寸匹配与延迟解码。
- **CLS**：为媒体设定尺寸/`aspect-ratio`、懒加载占位、避免异步注入在首屏上方改变布局、字体度量覆盖/`font-display`。
- **INP**：减少主线程阻塞（切分长任务、Web Worker）、避免同步布局、压缩渲染工作量（虚拟列表/合成层）、在事件回调中**快速返回**（延后次要工作到 `requestIdleCallback`/微任务或下一帧）。

**采样与分群**
- 按**国家/网络/设备**分群上报，关注 p75（或更严格分位）；与发布流水线/实验平台打通，出现显著劣化自动告警与回滚。

