# Typescript

## 1) TypeScript 的“结构类型系统”如何决定两个对象类型的兼容性？与“名义类型”有何不同，哪些场景会导致“意外兼容”？
- 核心原理  
  TypeScript 采用**结构类型**（structural typing）：两个类型是否兼容只取决于**成员形状**是否满足。`A` 可赋值给 `B` 等价于：`A` 至少拥有 `B` 所需的所有属性（且类型兼容），函数还需考虑参数/返回值的型变规则。多数情况下这是**子类型判定 = 成员包含判定**。  
  “名义类型”（如 Java/C#）依赖**显式声明关系**（类名/接口名/继承关系）才算兼容，形状相同也不自动兼容。
- 典型“意外兼容”  
  1) **不同语义相同形状**：如 `UserId = string` 与 `OrderId = string`，结构等同，容易混用。  
  2) **对象字面量额外属性检查**：字面量赋值时会做 **excess property check**；但经由变量再赋值可绕过检查（见下）。
- 示例
  ```ts
  type User = { id: string; name: string };
  type Order = { id: string; name: string }; // 结构同形 → 可互相赋值（结构类型）
  const u: User = { id: 'u1', name: 'A' };
  const o: Order = u; // OK（结构兼容），但语义可能错误
  // 字面量“额外属性检查”
  const x: User = { id: '1', name: 'A', age: 18 }; // ❌ age 多余
  const tmp = { id: '1', name: 'A', age: 18 };
  const y: User = tmp; // ✅ 经变量中转通过（结构相容）
  ```
- 规避策略：**品牌类型**（见第 22 题思路提前）或 `unique symbol`/`opaque` 包装；用 ESLint 规则限制“变量中转绕过”。

---

## 2) `any / unknown / never` 的语义边界分别是什么？在项目中如何限制 `any` 的扩散并合理使用 `unknown`？
- 语义  
  - `any`：**顶层逃生舱**。读写均不做检查，传播到下游使类型丢失。  
  - `unknown`：**安全的顶层类型**。可接收任何值，但**在读使用前必须收窄**；不能直接调用/取属性。  
  - `never`：**底类型**。不可达值（抛错、无限循环、穷尽检查的残余分支）。
- 风险与边界  
  - `any` 会通过泛型/交叉类型**向外扩散**，导致类型系统“短路”。  
  - `unknown` 用在**边界**（外部输入、catch 错误、反序列化结果），配合**类型守卫**或 schema 校验。
- 实践
  ```ts
  // 限制 any：开启 strict、noImplicitAny，并用 ESLint “ban-ts-comment”
  function parse(json: string): unknown { return JSON.parse(json); }
  const v = parse('{"n":1}');
  if (typeof v === 'object' && v && 'n' in v) {
    const n = (v as { n: number }).n; // 先收窄再断言
  }
  // never 用于穷尽检查
  type Shape = { kind: 'c'; r: number } | { kind: 's'; a: number };
  function area(s: Shape) {
    switch (s.kind) {
      case 'c': return Math.PI * s.r ** 2;
      case 's': return s.a * s.a;
      default: const _exhaustive: never = s; return _exhaustive; // 编译期提醒
    }
  }
  ```

---

## 3) 类型推断与字面量收缩：为什么 `const x = 'a'` 与 `let x = 'a'` 推断不同？`as const` 会带来哪些连锁影响？
- 原理  
  - **字面量收窄（literal narrowing）**：`const x = 'a'` 推断为 `'a'`；`let x = 'a'` 可再赋值，推断为 `string`。  
  - 函数实参在**上下文类型**影响下也会收窄（contextual typing）。
- `as const`  
  - 将对象/数组**深度只读 + 字面量化**：字符串/数值变字面量类型，数组变 **readonly tuple**。  
  - 影响：只读限制导致函数需要匹配 `readonly` 参数或做只读到可变的映射。
- 示例
  ```ts
  const a = 'x';      // type 'x'
  let b = 'x';        // type string
  const cfg = { mode: 'dark', size: 12 } as const;
//      ^? { readonly mode: 'dark'; readonly size: 12 }
  type Mode = typeof cfg.mode; // 'dark'
  // 只读 tuple
  const path = ['users', 'detail'] as const; // readonly ['users','detail']
  type Route = typeof path[number]; // 'users' | 'detail'
  ```

---

## 4) 联合类型与交叉类型在可空性/可选属性上的行为差异？`strictNullChecks` 对联合类型的可用性有什么改变？
- 联合（`A | B`）：**成员取交集**可访问（共同成员），访问专属成员需收窄。  
  交叉（`A & B`）：**成员合并**，相同键的类型做**交叉**（可能变 `never`）。
- 可选/可空  
  - 可选属性 `p?: T` 表示“**键可能不存在**”，索引访问 `T['p']` 会得到 `T | undefined`。  
  - “可空”用联合 `T | undefined | null`；与 `strictNullChecks` 关闭时的“宽松”不同，开启后必须显式处理。
- 示例
  ```ts
  type A = { a: number; opt?: string };
  type B = { b: number; opt?: number };
  type U = A | B;
  type I = A & B; // { a: number; b: number; opt?: string & number } → opt 为 never（基本不可用）
  function f(x: U) { x.opt; /* string | number | undefined */ }
  ```
- 结论：交叉用于“**叠加能力**”，存在同名属性时需小心；联合用于“**多形态**”，结合收窄使用。开启 `strictNullChecks` 后，`undefined/null` 不再隐式兼容，能显著降低 NPE。

---

## 5) 类型收窄（控制流分析）如何工作？请对比 `typeof / instanceof / in / 判空` 与“可辨识联合”的配合方式与局限。
- 原理  
  TS 构建控制流图（CFG），在布尔条件、赋值、守卫之后**逐步收窄**变量类型。  
- 收窄方式  
  - `typeof x === 'string'`、`instanceof Ctor`、`'prop' in obj`、`x != null`、`Array.isArray(x)` 等。  
  - **可辨识联合**：使用**字面量标记字段**作判别，switch 后分支自动收窄。
- 局限  
  - 复杂别名/可变引用会**打破收窄**；穿越函数边界需要**类型守卫**（第 6 题）。  
  - 异步/闭包中变量被再次赋值，之前的收窄可能失效。
- 示例
  ```ts
  type Msg = { type: 'text'; body: string } | { type: 'img'; url: string };
  function handle(m: Msg) {
    switch (m.type) {
      case 'text': m.body; break;  // 收窄到 text
      case 'img' : m.url;  break;  // 收窄到 img
    }
  }
  function g(x: unknown) {
    if (typeof x === 'number') { x.toFixed(2); } // 收窄 number
    if (x && typeof x === 'object' && 'id' in x) { (x as any).id; }
  }
  ```

---

## 6) 如何编写用户自定义类型守卫（`x is T`）？在哪些情形会失效（跨函数边界、异步场景、闭包逃逸）？
- 语法  
  `function isT(v: unknown): v is T { return /* runtime check */ }` 返回值为**类型谓词**。仅在**true 分支**内收窄。  
- 常见写法  
  ```ts
  type User = { id: string; name: string };
  function isUser(x: unknown): x is User {
    return !!x && typeof x === 'object'
      && 'id' in x && typeof (x as any).id === 'string'
      && 'name' in x && typeof (x as any).name === 'string';
  }
  // 断言函数：失败时抛错并在后续分支认为收窄成立
  function assertUser(x: unknown): asserts x is User {
    if (!isUser(x)) throw new Error('not User');
  }
  ```
- 失效/注意  
  - **异步**返回不能写 `Promise<... is T>`；只能在 `await` 后再收窄。  
  - **闭包逃逸**：守卫后的变量若在外层被重新赋值，旧的收窄不再可信。  
  - 守卫逻辑**不完备**（如只检查存在性不检查类型）会造成**不安全收窄**。

---

## 7) 函数类型：参数是否“逆变”？`strictFunctionTypes` 打开/关闭的行为差异及对回调类型（如事件处理器）的影响。
- 型变规则  
  对函数类型 `A → R`：**参数应逆变**（接受“更宽”的实参类型），**返回值协变**（返回“更窄”的类型）。  
- TypeScript 行为  
  - `strictFunctionTypes: true` 时，**函数类型的参数按逆变**检查（方法参数保留历史“**双向协变**”例外以兼容 DOM）。  
  - 关闭时，参数近似**双向协变**，可能引入不安全赋值。
- 示例
  ```ts
  type Animal = { name: string };
  type Dog    = Animal & { bark(): void };
  type H<T> = (x: T) => void;

  let a: H<Animal>;
  let d: H<Dog>;
  d = a; // ✅ 逆变：处理 Dog 的地方可用处理 Animal 的函数吗？不安全 → 在 strict 下应 ❌
  a = d; // ✅ 处理 Animal 的地方可用处理 Dog 的函数（更具体参数）？这才是逆变允许的方向
  ```
  在事件处理器等框架回调中常见的“方法位（bivariance）”特殊规则可能放宽检查，建议为公共 API 显式声明**更宽**参数类型避免依赖宽松规则。

---

## 8) `interface` 与 `type` 的能力差异与取舍？声明合并、递归类型、计算属性各在哪类问题更合适？
- 相同点：都能描述对象/函数/索引签名，均可递归。  
- 差异  
  - `interface` 可**声明合并**与 **extends** 多个接口；适合**公共 API**、第三方声明增强。  
  - `type` 是别名：可表示**联合/交叉/条件/映射**等更复杂组合；支持模板字面量、键重映射（通过映射类型）。  
- 建议  
  - **公共稳定结构**优先 `interface`（利于合并/扩展）；  
  - **类型运算/组合**优先 `type`。
- 示例
  ```ts
  interface A { x: number }
  interface A { y: string } // 合并 → { x: number; y: string }

  type B = { x: number } & { y: string }; // 组合，不会“自动合并”重名声明
  type C<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] }; // 计算属性更灵活
  ```

---

## 9) 索引类型体系：`keyof / T[K] / 映射类型` 的组合用法；如何用 `as` 做键重映射（重命名/过滤键）？
- 基元  
  - `keyof T`：属性名联合。  
  - `T[K]`：索引访问类型；若 `K` 是联合，会**分发**成联合。  
  - 映射类型：`{ [K in keyof T]: ... }`，可配合 `readonly`/`?` 的 `+/-` 修饰。
- 键重映射（TS 4.1+）  
  ```ts
  type Getterify<T> = {
    [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K]
  };
  type DropByValue<T, V> = {
    [K in keyof T as T[K] extends V ? never : K]: T[K]
  };
  ```
- 注意  
  - `as never` 可**删除**键；  
  - `string & K` 确保模板字面量能处理 symbol/number 键；  
  - 结合 `-?`/`+readonly` 精准控制可选与只读。

---

## 10) 常用工具类型的原理：`Partial / Required / Readonly / Pick / Omit / Record / NonNullable / ReturnType / Parameters / InstanceType` 可分别如何用条件/映射类型实现？
```ts
type Partial<T>    = { [K in keyof T]?: T[K] };
type Required<T>   = { [K in keyof T]-?: T[K] };
type Readonly<T>   = { readonly [K in keyof T]: T[K] };
type Pick<T, K extends keyof T>  = { [P in K]: T[P] };
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
type Record<K extends keyof any, V> = { [P in K]: V };

type NonNullable<T> = T extends null | undefined ? never : T;

type ReturnType<F extends (...args: any) => any> =
  F extends (...args: any) => infer R ? R : never;

type Parameters<F extends (...args: any) => any> =
  F extends (...args: infer P) => any ? P : never;

type InstanceType<C extends abstract new (...a: any) => any> =
  C extends abstract new (...a: any) => infer R ? R : never;
```
- 相关基石  
  - `Exclude<T, U> = T extends U ? never : T`  
  - `Extract<T, U> = T extends U ? T : never`
- 细节  
  - `Required` 用 `-?` 去掉可选修饰；  
  - `InstanceType` 需兼容抽象构造函数；  
  - `Omit` 通过对 `keyof T` 先 `Exclude` 再 `Pick`。

---
## 11) 泛型参数设计：约束/默认值、`keyof` 约束链与“过度泛型化”风险
- 核心原则  
  1) **先约束，再推断**：`<T extends {...}>` 让推断空间在“合法集合”内进行，减少 `any` 外泄。  
  2) **能从参数推断就别显式写**：避免把简单函数“模板化”到难以阅读。  
  3) **默认类型**用于“无需传参也有合理类型”：`<T = unknown>`、`<K extends keyof T = keyof T>`。

- 典型模式  
  ```ts
  // 从对象读取属性：K 必须来自 T 的键
  function get<T, K extends keyof T>(obj: T, key: K): T[K] { return obj[key]; }

  // 约束链：K 受 T 影响，默认值也依赖 T
  type PickByKeys<T, K extends keyof T = keyof T> = { [P in K]: T[P] };

  // 泛型默认值：不传 T 时回落到 string
  type Box<T = string> = { value: T };
  ```

- 过度泛型化的气味  
  - **类型参数只出现一次**：`function f<T>(x: T) { return 1 }` → `T` 对返回值/约束无用，应移除。  
  - **在实现中立刻断言/分发**：`as any`、大量 `T extends any ? ...` 表明模型过复杂。  
  - **把值域也泛型化**：本应是联合或枚举的值，被做成 `<T extends string>`，失去可读性与报错质量。

---

## 12) 分布式条件类型与 `infer`：工作原理与典型实现
- 分布式条件类型（Distributive Conditional Types）  
  当 **裸类型参数** `T` 出现在 `T extends U ? X : Y` 左侧时，`T` 为联合会被**自动分发**：  
  `A | B` ⇒ `(A extends U ? X : Y) | (B extends U ? X : Y)`。  
  抑制分发：用元组包裹 `[T] extends [U] ? X : Y`。

- `infer` 推断占位  
  在条件类型的 **真分支**里占位并绑定：  
  ```ts
  // Awaited
  type MyAwaited<T> =
    T extends null | undefined ? T :
    T extends Promise<infer U> ? MyAwaited<U> : T;

  // ReturnType / Parameters
  type MyReturnType<F> = F extends (...a: any) => infer R ? R : never;
  type MyParameters<F> = F extends (...a: infer P) => any ? P : never;

  // 数组元素 / 元组尾
  type Elem<T> = T extends (infer U)[] ? U : T;
  type Tail<T extends any[]> = T extends [any, ...infer R] ? R : [];
  ```

- 误用与边界  
  - **意外分发**导致性能与语义问题：`T extends string ? ...` 而 `T` 是大联合 → 爆炸；用 `[T] extends [string]` 抑制。  
  - `infer` **只能用一次**绑定一个名字；多处出现指同一绑定。  
  - 对未解析的泛型 `T` 做复杂条件：结果会延迟到调用点，报错更晚更难读。

---

## 13) 模板字面量类型：生成 API 名与“路由字符串”建模
- 键名变换（生成 `setX`）  
  ```ts
  type Setters<T> = {
    [K in keyof T as `set${Capitalize<string & K>}`]: (v: T[K]) => void
  };

  type Model = { name: string; age: number };
  // { setName(v: string): void; setAge(v: number): void }
  type MSet = Setters<Model>;
  ```

- 路由/路径的类型安全  
  ```ts
  // 连接工具（递归 Join）
  type Join<S extends string[], D extends string> =
    S extends [] ? '' :
    S extends [infer H extends string] ? H :
    S extends [infer H extends string, ...infer R extends string[]]
      ? `${H}${D}${Join<R, D>}` : string;

  type Path<S extends string[]> = `/${Join<S, '/'>}`;

  const p1: Path<['users', ':id']> = '/users/:id';     // ✅
  const p2: Path<['posts','create']> = '/posts/create'; // ✅
  // const bad: Path<['users']> = '/user';              // ❌
  ```

- 注意事项  
  - 对 `symbol/number` 键要 `string & K` 收窄；  
  - 生成的字面量类型过大时影响性能，需要限规模或预定义片段集合。

---

## 14) 变长元组与参数列表：`Head/Tail/Concat/TupleToUnion` 与推断坑
- 基础工具  
  ```ts
  type Head<T extends any[]> = T extends [infer H, ...any[]] ? H : never;
  type Tail<T extends any[]> = T extends [any, ...infer R] ? R : [];
  type Concat<A extends any[], B extends any[]> = [...A, ...B];
  type TupleToUnion<T extends any[]> = T[number];
  ```

- 保留参数的**精确元组**推断  
  ```ts
  // 通过“变长元组参数”保留每个位置的类型
  function bind<A extends any[], R>(fn: (...a: A) => R, ...a: A) {
    return () => fn(...a);
  }
  const f = (a: number, b: 'x' | 'y') => `${a}-${b}`;
  const g = bind(f, 1, 'x'); // 参数元组 A 推断为 [number, 'x']
  ```

- 常见坑  
  - **双向推断受限**：在复杂位置（返回值+参数同时依赖）会失败，需**辅助泛型**或临时变量分步推断。  
  - **交叉展开顺序**固定为左到右，`[...A, ...B]` 与 `[..., ...]` 嵌套过多会拖慢检查。  
  - 与**函数重载**混用时，重载先于泛型匹配，可能得到非预期分支。

---

## 15) `satisfies` 与 `as const`：形状校验与推断宽度的配合
- 语义对比  
  - `as const`：**深度只读 + 字面量收缩**；改变值的类型形状。  
  - `satisfies`：**仅做约束校验**，不改变左侧表达式的**变量类型**，但允许从中**提取更精确的字面量信息**。

- 组合模式  
  ```ts
  // 既要“键和值符合规范”，又要保留变量较宽的可写类型
  const routes = {
    home: '/home',
    user: '/users/:id',
  } satisfies Record<string, `/${string}`>; // 校验形状

  let r = routes.user;   // 推断仍为 string（变量更宽），但在类型层可取到字面量集合
  type RouteNames = keyof typeof routes;    // 'home' | 'user'
  type RoutePaths = typeof routes[RouteNames]; // `/${string}`（可进一步约束）
  ```

- 何时用  
  - 配置对象/表驱动：校验键值合法性，不强制“只读”。  
  - 与 `as const` 同用：需要**既只读又校验**时：`const cfg = {...} as const satisfies Schema;`（顺序不限）。

---

## 16) `readonly` 在属性、数组、元组中的语义与 `DeepReadonly` 边界
- 属性层面  
  ```ts
  type T = { readonly x: number; y: { z: number } };
  const t: T = { x: 1, y: { z: 2 } };
  // t.x = 2; // ❌ 仅限制最外层写入
  t.y.z = 3; // ✅ 嵌套可写（浅只读）
  ```

- 数组/元组  
  - `readonly T[]` / `ReadonlyArray<T>`：不可写（`push/splice` 被禁止），但读操作可用。  
  - `readonly [A, B]`：位置元素只读，结构固定。  
  - 与 `as const` 一起用于**常量表**与**不可变配置**。

- `DeepReadonly`（典型实现与局限）  
  ```ts
  type DeepReadonly<T> =
    T extends (...a: any) => any ? T :
    T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
    T;

  // 局限：对“可变但方法式更新”的数据结构无能；运行时仍可通过类型断言/克隆破坏不变性
  ```

---

## 17) `enum / const enum / 字面量联合`：编译产物与取舍
- 编译与运行时  
  - `enum`（数字/字符串）：**生成运行时对象**（双向映射仅数字枚举）。  
  - `const enum`：**编译期内联**，无运行时代码；需要 TS 编译参与（Babel/ts-node `transpileOnly` 场景会炸）。  
  - **字面量联合**：仅类型层，运行时零成本（常配合对象映射）。

- 取舍建议  
  - 库/跨编译链路：**避免 `const enum`**（易因跳过 TS 编译而崩）。  
  - 业务前端：优先**字面量联合 + 对象表**，可结合 `satisfies` 做**穷尽校验**。
  ```ts
  // 字面量联合 + 对象表
  type Color = 'red' | 'green' | 'blue';
  const ColorLabel = {
    red: '红', green: '绿', blue: '蓝'
  } satisfies Record<Color, string>;
  ```

---

## 18) 模块解析与别名：`baseUrl/paths/types/typeRoots/moduleResolution` 与构建联动
- TS 只在**类型检查**阶段解析路径；打包器需**同步配置别名**，否则运行时报错。
- 关键开关  
  ```json
  {
    "compilerOptions": {
      "baseUrl": ".",
      "paths": { "@/*": ["src/*"] },
      "types": ["node", "jest"],     // 全局类型包白名单
      "typeRoots": ["./types", "./node_modules/@types"],
      "moduleResolution": "bundler"  // TS5：更贴近 ESM 打包器解析
    }
  }
  ```
- 联动示例  
  - **Vite/webpack** 同步别名：`resolve.alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) }`。  
  - **ts-node / vitest**：启用 `tsconfig-paths` 或原生支持，保证运行期可识别 `paths`。  
  - ESM/CJS 混用：结合 `package.json` 的 `exports` 与 `typesVersions`，确保类型与运行时目标一致。

---

## 19) DOM/Node 多目标工程：`lib`、多 `tsconfig` 与条件导出避免全局冲突
- 冲突来源  
  - `dom` 与 `node` 的全局可能重名（如 `Event`）；Node 18+ 的 `fetch`/`WebSocket` 类型来自不同来源，容易混淆。
- 工程策略  
  1) **分目标 tsconfig**：  
     ```
     tsconfig.dom.json   → { "lib": ["ES2022", "DOM"], "types": [] }
     tsconfig.node.json  → { "lib": ["ES2022"], "types": ["node"] }
     ```
  2) **构建两份产物**（`dist/browser`，`dist/node`），并在 `package.json` 用条件导出：  
     ```json
     {
       "exports": {
         ".": {
           "browser": "./dist/browser/index.js",
           "default": "./dist/node/index.js",
           "types": "./dist/index.d.ts"
         }
       }
     }
     ```
  3) **避免全局污染**：只在需要时引入 `@types/node`；测试环境（Jest/Vitest）用 `environment` 控制全局注入。

---

## 20) 函数重载 vs 联合/泛型：表达力、可维护性与实现签名规则
- 规则回顾  
  - **重载**由多条**签名**+一条**实现签名**组成；调用点从上到下挑选匹配的签名。  
  - **实现签名**必须能兼容所有重载分支（通常写最宽的联合/可选参数）。  
  - 仅**重载签名**会出现在调用方的类型提示中。

- 何时选谁  
  - **重载**：不同参数形态返回**不同的返回类型**，并且这些形态**有限且稳定**（如：`string → number`，`number → string`）。  
  - **联合/泛型**：同构 API，返回类型可从参数**条件推断**即可，无需枚举分支。

- 示例对比  
  ```ts
  // 重载
  function get(x: string): number;
  function get(x: number): string;
  function get(x: string | number) { return typeof x === 'string' ? x.length : String(x); }

  // 条件泛型（调用点可能需要 as 帮助，推断不如重载直观）
  function get2<T extends string | number>(x: T): T extends string ? number : string {
    return (null as any);
  }
  ```

- 常见坑  
  - **实现签名暴露**：不要把实现签名导出（应只导出重载签名）。  
  - **顺序敏感**：更具体的重载需排前；否则被宽签名“吞掉”。  
  - **与变长元组结合**：过多重载可被 `...args: [...A]` + 条件类型替代，减少维护成本。

---
## 21) 协变 / 逆变 / 双向协变：为何回调参数会“放宽”？如何在关键边界强制安全
- 概念与 TS 行为  
  - **返回值协变**、**参数逆变**是函数型变的标准规则。  
  - 在 `strictFunctionTypes: true` 下：  
    - **函数类型表达式**的参数按“**逆变**”检查：`(dog) => void` 不能赋给 `(animal) => void`。  
    - **方法语法**（对象/接口中的 `on(e: T): void`）为历史兼容保留了**双向协变（bivariance）**，更宽松，易埋隐患。
- 风险示例  
  ```ts
  type Animal = { name: string };
  type Dog = Animal & { bark(): void };

  type F<T> = (x: T) => void;          // 函数类型表达式：参数逆变
  type M = { on(x: Dog): void };        // 方法位：参数双向协变（宽松）

  let fAnimal: F<Animal>;
  let fDog: F<Dog>;
  // fDog = fAnimal; // ❌ 逆变保护（strict 下拒绝）

  let mDog: M = { on(x: Dog) { x.bark(); } };
  const mAnimalLike: M = mDog; // ✅ 方法位放宽，潜在不安全
  ```
- 强制安全的工程做法  
  1) **用属性签名代替方法签名**：`on: (e: Animal) => void`，恢复逆变检查。  
  2) **显式写更宽的参数类型**（在公共 API 处）：`(e: Animal) => void`。  
  3) **适配层**：把上游窄函数包一层宽函数：`(a: Animal) => isDog(a) && h(a)`。  
  4) **测试**：用“反例赋值”单测验证 API 的逆变约束。

---

## 22) 用品牌 / 不透明类型模拟“名义性”，防止 ID 混用
- 基本模式（`unique symbol` 品牌）  
  ```ts
  declare const UserIdBrand: unique symbol;
  declare const OrderIdBrand: unique symbol;

  export type UserId = string & { readonly [UserIdBrand]: true };
  export type OrderId = string & { readonly [OrderIdBrand]: true };

  export const asUserId = (s: string): UserId => s as UserId;
  export const asOrderId = (s: string): OrderId => s as OrderId;
  // 不可互换
  // const x: UserId = asOrderId('...'); // ❌
  ```
- 跨包扩展  
  - **导出品牌符号**或在公共 `types` 包内集中声明品牌。  
  - 如需多租户/多环境品牌，可把品牌设计成**泛型**：`type Opaque<T, Brand> = T & { readonly __brand: Brand }`。
- 运行时与调试  
  - 品牌在运行时**不存在**；若需要运行时校验，用 schema（zod/json schema）在边界处同时校验与“加品牌”。

---

## 23) 避免“分布式条件类型”陷阱：何时分发？如何抑制
- 何时分发  
  裸类型参数出现在左侧：`T extends U ? X : Y`。当 `T = A | B` ⇒ `(A extends U ? X : Y) | (B extends U ? X : Y)`。  
  ```ts
  type Box<T> = T extends string ? `S:${T}` : number;
  type R = Box<'a' | 'b'>; // 'S:a' | 'S:b'
  ```
- 典型陷阱  
  - 性能爆炸：`T` 是大联合，分发产生巨大联合。  
  - 语义偏差：你想要“整体判断”，结果被分发了。
- 抑制技巧  
  - **元组包裹**：`[T] extends [U] ? X : Y`。  
  - **条件前收敛**：先把 `T` 归约成更窄的联合再进入条件。  
  - **层级切断**：在复杂链路中用 `unknown/any` 隔离，避免继续递归推导。
  ```ts
  type IsString<T> = [T] extends [string] ? true : false; // 对整体判断
  ```

---

## 24) 高级工具类型实现与边界
```ts
// 递归可选
export type DeepPartial<T> =
  T extends (...a: any) => any ? T :
  T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } :
  T;

// 递归必填
export type DeepRequired<T> =
  T extends (...a: any) => any ? T :
  T extends object ? { [K in keyof T]-?: DeepRequired<T[K]> } :
  T;

// 去只读
export type Mutable<T> = { -readonly [K in keyof T]: T[K] };

// 浅合并：右侧覆盖左侧
export type Merge<A, B> = Omit<A, keyof B> & B;

// 联合 → 交叉
export type UnionToIntersection<U> =
  (U extends any ? (x: U) => 0 : never) extends (x: infer I) => 0 ? I : never;

// 联合的“包含键”（Keys of union，分发 key）
export type KeysOfUnion<T> = T extends any ? keyof T : never;
```
- 边界讨论  
  - 递归工具在**深/大对象**上会**拖慢检查**；给出**最大深度**版本或在外层断开递归。  
  - `UnionToIntersection` 依赖函数参数逆变原理，过大联合也会影响性能。  
  - `Merge` 为类型层“并集”，与运行时 `Object.assign` 行为一致并不保证（如 getter/setter）。

---

## 25) 键重映射 + 模板类型：从 `getX` 生成 `setX`
- 目标：从接口 `{ getUser(): User; getPost(): Post }` 生成  
  `{ setUser(v: User): void; setPost(v: Post): void }`
```ts
type Setters<T> = {
  [K in keyof T as K extends `get${infer S}`
    ? `set${S}` : never]: T[K] extends (...a: any) => infer R ? (v: R) => void : never
};

interface Query {
  getUser(): { id: string };
  getPost(): { id: number };
}
type Mutation = Setters<Query>;
/*
type Mutation = {
  setUser(v: { id: string }): void;
  setPost(v: { id: number }): void;
}
*/
```
- 变体  
  - 只允许 `get` → `set`，其他键丢弃（通过 `never`）。  
  - 若需要 `updateX(partial)`，可将 `R` 包装为 `DeepPartial<R>`。

---

## 26) 错误类型安全：`unknown`、跨 realm 与 `Result<E, A>`
- `unknown` 错误处理范式（`try/catch`）  
  ```ts
  try { ... }
  catch (e: unknown) {
    const msg =
      e instanceof Error ? e.message :
      typeof e === 'string' ? e :
      JSON.stringify(e);
  }
  ```
- 跨 realm 的 `instanceof Error` 失效  
  - 不同 iframe/worker 的 `Error` 构造不同。替代：  
    1) **duck typing**：`e && typeof e === 'object' && 'message' in e`；  
    2) **`Object.prototype.toString.call(e) === "[object Error]"`**（更稳）；  
    3) 统一通过**序列化通道**把错误转为 `{ name, message, stack? }`。
- `Result<E, A>` 边界建模  
  ```ts
  type Ok<A>  = { ok: true;  value: A };
  type Err<E> = { ok: false; error: E };
  type Result<E, A> = Ok<A> | Err<E>;

  const parseJson = (s: string): Result<string, unknown> => {
    try { return { ok: true, value: JSON.parse(s) }; }
    catch (e) { return { ok: false, error: `invalid: ${String(e)}` }; }
  };
  ```
  - 优点：**显式错误通道**，便于模式匹配；  
  - 缺点：与 `throw` 并存时需一致化，建议**业务层统一 Result**，边界（IO/框架）再转换为异常。

---

## 27) 声明增强与全局污染：何时“增强”，如何不失控
- 模块增强（module augmentation）  
  为现有库补类型或扩展接口：
  ```ts
  // axios-aug.d.ts
  import 'axios';
  declare module 'axios' {
    interface AxiosRequestConfig { traceId?: string }
  }
  ```
- 全局增强（global augmentation）  
  ```ts
  // globals.d.ts
  export {}; // 使其成为模块，避免隐式全局
  declare global {
    interface Window { __APP_VERSION__: string }
  }
  ```
- 风险与治理  
  - **污染范围不可控**：仅在确需共享时用全局；否则**导出类型**由使用方显式导入。  
  - 与上游升级**易冲突**：增强应**窄化**（最少字段），并加注释标明来源版本。  
  - 将增强文件放入独立目录（如 `types/`），并在 `tsconfig.json` 中通过 `typeRoots/types` 精准启用。

---

## 28) 装饰器：TS 旧版 vs TC39 新版；`emitDecoratorMetadata` 的取舍
- 语义差异  
  - **TS 旧实验装饰器**（legacy）：`@dec(target, key, descriptor)`，装饰器形如“函数改造 descriptor”；可配合 `reflect-metadata` 通过 `emitDecoratorMetadata` 注入运行时类型信息（`design:type` 等，**近似**而非精准）。  
  - **TC39 新装饰器**（stage 3，TS 5.x 支持）：`@dec` 接收 `value` 与 `context`，返回**替换器/初始化器**；语义更接近标准，不再依赖 descriptor 细节。
- `emitDecoratorMetadata` 评估  
  - **收益**：依赖注入/验证框架（NestJS 等）可直接读取设计期类型元数据。  
  - **成本**：产物**体积增加**；元数据**不精确**（例如泛型、联合被擦除为 `Object`）；易与 Tree-Shaking 冲突。  
  - 建议：库作者**尽量不用**；应用如需 DI，可**按需开启**并评估产物体积与安全（避免暴露内部类型结构）。

---

## 29) 项目引用（Project References）：拆分构建图与发布
- 基本设置  
  - 子包 `tsconfig.json`：`"composite": true, "declaration": true, "declarationMap": true, "outDir": "dist"`  
  - 根 `tsconfig.json`：  
    ```json
    { "files": [], "references": [{ "path": "packages/utils" }, { "path": "packages/app" }] }
    ```
  - 构建：`tsc -b`（build mode 会按**拓扑顺序**编译、缓存增量）。
- 发布与 IDE 体验  
  - 产出 `.d.ts` + `.d.ts.map`，下游可**跳转到源码**（需 `sourceMap: true` 且打包保留映射）。  
  - 与打包器联动：每个引用包**单独打包**，确保 `exports`/`types` 指向自身产物。
- 常见坑  
  - **循环引用**：`tsc -b` 直接报错，需拆边界或抽公共包。  
  - **路径别名 vs references 混用**：内部包使用相对/包名导入，避免 `paths` 把编译输出与源码混淆。  
  - **noEmit**：引用项目必须允许 emit；否则 `-b` 不产物，联动构建失败。

---

## 30) 类型性能治理：定位“类型层爆炸”与降噪手段
- 识别与度量  
  - `tsc --extendedDiagnostics` / `--generateTrace` 定位耗时文件与符号。  
  - 观察**模板字面量**、**深递归条件类型**、**巨大联合分发**的热点。
- 缓解策略  
  1) **缩小输入规模**：在热点工具类型前置 `T extends object ? ... : never`、`[T] extends [U]` 抑制分发。  
  2) **分层断开**：在类型边界放入 `unknown/any`（“**类型绝缘体**”）阻断继续推导。  
  3) **限制递归深度**：为 `Deep*` 系列提供 `MaxDepth` 参数或在对象层级过深时提前返回 `unknown`。  
  4) **静态常量降维**：对大型 `as const` 配置表改用 `satisfies` 仅校验形状，减少字面量联合规模。  
  5) **避免意外分发**：大联合条件统一用 `[T] extends [U]`。  
- 监控与回归  
  - 将 `tsc` 诊断指标纳入 CI（记录耗时/内存）；增量构建时间突增即回看最近工具类型改动。

---
## 31) `.d.ts` 与声明发布：为 JS/第三方库补类型；ESM/CJS/子路径的发布策略
- 生成 `.d.ts`（TS 源码）
  - `tsconfig`：`"declaration": true, "declarationMap": true, "stripInternal": true, "emitDeclarationOnly": true`（如单独跑类型构建）。
  - 用 `/** @internal */` 隐藏内部 API，开启 `stripInternal` 后不出现在 `.d.ts`。
  - 多包/引用：子包 `composite:true`，顶层 `tsc -b` 产出稳定 `.d.ts` 拓扑。

- 为 JS 包补类型
  - **内联 JSDoc**（轻量）：`// @ts-check` + `/** @param {string} x */`；适合纯 JS 小包。
  - **手写 `.d.ts`**（严谨）：放在 `types/`，`package.json` 指向。
    ```ts
    // index.d.ts (ESM 形态)
    export interface Options { base?: string }
    export default function compile(src: string, opts?: Options): Promise<string>;
    ```
    ```ts
    // index.d.ts (CJS 形态)
    declare function compile(src: string, opts?: { base?: string }): Promise<string>;
    export = compile;
    ```

- 第三方库增强（module augmentation / global）
  ```ts
  // axios-extend.d.ts
  import 'axios';
  declare module 'axios' {
    interface AxiosRequestConfig { traceId?: string }
  }
  // 或全局
  export {};
  declare global { interface Window { __APP_VERSION__: string } }
  ```

- `package.json` 发布矩阵（ESM/CJS/类型/子路径）
  ```json
  {
    "type": "module",
    "main": "./dist/index.cjs",
    "module": "./dist/index.js",
    "types": "./dist/index.d.ts",
    "exports": {
      ".": {
        "types": "./dist/index.d.ts",
        "import": "./dist/index.js",
        "require": "./dist/index.cjs"
      },
      "./client": {
        "types": "./dist/client.d.ts",
        "import": "./dist/client.js",
        "require": "./dist/client.cjs"
      }
    },
    "typesVersions": {
      "*": {
        "client": ["dist/client.d.ts"]
      }
    }
  }
  ```
  - `exports` 是**运行时解析**与**类型入口**的真相源；`typesVersions` 是给旧 TS（不识别 `exports` 类型条件）兜底的**路径映射**。
  - 避免 Dual Package Hazard：按条件导出区分 `import/require`，并保持 `.d.ts` 与实现**一一对应**。

- 调试与健康度
  - 对外库：自己构建时 `skipLibCheck:false` 以保证 `.d.ts` 正确；对使用方：建议 `skipLibCheck:true` 加快 CI。
  - `.d.ts.map` + `sourceMap` 让 IDE 能跳回源码；若不想泄露源码，**不要发布 `.map`**（或仅上传到错误平台）。

---

## 32) 类型 ↔ 运行时校验：zod/valibot/io-ts 结合；JSON Schema 互转的边界
- “类型只在编译期，校验需运行时”——推荐**以运行时 schema 为源**，类型由其导出：
  ```ts
  import { z } from 'zod';
  const User = z.object({
    id: z.string().uuid(),
    name: z.string().min(1),
    roles: z.array(z.enum(['admin','user'])).default(['user']),
    createdAt: z.coerce.date() // 运行时从字符串转 Date
  });
  type User = z.infer<typeof User>;
  ```
  - 优势：**单一事实来源**，避免 TS 与运行时分叉；`z.infer` 自动得到类型。
  - 替代：`valibot`（极轻量），`io-ts`（与 fp-ts 一体，代价高）。

- 与 JSON Schema / Ajv 协作
  - `zod-to-json-schema` 可导出 JSON Schema，供后端/表单引擎/Ajv 使用。
  - **不可逆/边界**：TS 高级类型（条件/映射/模板）、品牌、函数、`Map/Set`、交叉细粒度约束，**无法完整转为 JSON Schema**；Date 通常以 `string`（format）表达。

- 从 TS 生成 Schema？
  - `ts-json-schema-generator` 能覆盖多数接口，但对**条件类型/泛型重度**模型易退化，且**丢失运行时副作用**（如 `coerce`）。
  - 结论：**前后端共享模型**优先从 zod/valibot 开始，必要时生成类型与 JSON Schema；“TS → Schema”仅在模型很规整时采用。

- API 边界范式
  ```ts
  // 入参校验
  export function handle(input: unknown) {
    const parsed = User.safeParse(input);
    if (!parsed.success) return { ok:false, error: parsed.error.format() };
    const data: User = parsed.data;
    // ...
    return { ok:true };
  }
  ```

---

## 33) React/Vue 高级建模：Props/受控、forwardRef/memo、defineProps/Emits/Slots
- React：受控组件与多形态 props
  ```tsx
  // 受控 vs 非受控（互斥联合）
  type Controlled<T>   = { value: T; onChange(v: T): void; defaultValue?: never };
  type Uncontrolled<T> = { defaultValue?: T; onChange?(v: T): void; value?: never };
  type InputProps = (Controlled<string> | Uncontrolled<string>) & { disabled?: boolean };

  function Input(p: InputProps) {
    const [inner, setInner] = React.useState(p.defaultValue ?? '');
    const value = 'value' in p ? p.value : inner;
    const set = (v: string) => ('value' in p ? p.onChange(v) : (setInner(v), p.onChange?.(v)));
    return <input value={value} onChange={e=>set(e.target.value)} disabled={p.disabled} />;
  }
  ```

  - `forwardRef` + 泛型 + `ElementRef`
    ```tsx
    import React, { forwardRef } from 'react';
    type BtnProps<C extends React.ElementType = 'button'> = {
      as?: C;
      loading?: boolean;
    } & React.ComponentPropsWithoutRef<C>;

    const Btn = forwardRef(<C extends React.ElementType = 'button'>(
      { as, loading, ...rest }: BtnProps<C>,
      ref: React.ComponentRef<C>
    ) => {
      const Comp = (as ?? 'button') as C;
      return <Comp ref={ref as any} aria-busy={loading} {...rest} />;
    });
    ```
  - `memo`：若 props 中含函数/对象，给出 `areEqual` 或将关键 props 变为**稳定值**（如 `useMemo` 后的序列化键）。

- Vue 3：`defineProps/defineEmits/slots` 精确类型
  ```ts
  // MyInput.vue <script setup lang="ts">
  const props = defineProps<{
    modelValue: string;
    disabled?: boolean;
  }>();
  const emit = defineEmits<{
    (e: 'update:modelValue', v: string): void;
    (e: 'focus'): void;
  }>();
  // v-model 推断：<MyInput v-model="x" /> -> x: string
  ```

  - 具名插槽与约束
    ```ts
    // 父组件里 TS 能推断 slot props
    // <MyList v-slot:item="{ user }"> {{ user.name }} </MyList>
    ```

  - 组件库约束：导出 `Props` 与 `Emits` 辅助类型，跨文件保持一致；利用 `as const` + `satisfies` 保持配置表既校验又可推断。

---

## 34) 库作者的 `declaration/declarationMap/sourceMap` 策略：可调试与隐私取舍
- 推荐配置（库）
  ```json
  {
    "compilerOptions": {
      "declaration": true,
      "declarationMap": true,
      "sourceMap": true,
      "inlineSources": true,
      "stripInternal": true,
      "removeComments": false
    }
  }
  ```
  - `.d.ts.map` 让 IDE 跳到源码（配合源码发布或 `npm src` 包）。  
  - `inlineSources` 提升可调试性，但会把源码嵌入 `.map`，**注意隐私与包体**。

- 发布与调试选项
  - **公开源码**：发布 `.map` + 源码，DX 最佳；  
  - **半公开**：只发布 `.d.ts`，不发 `.map`（或仅上传到 Sentry 等错误平台）；  
  - **最小暴露**：`emitDeclarationOnly` + rollup-plugin-dts 合并类型为单文件（便于下游）。

- API 面（防泄漏）
  - 用 `/** @internal */`、`/** @deprecated */` 标注；对内部工具不给导出，防止被依赖（破坏 SemVer）。
  - 用“门面”文件集中 re-export，仅暴露稳定 API；`.d.ts` 只从门面导出，避免内部路径泄漏。

- ESM/CJS 调试
  - 条件导出提供 `types` 指向 `.d.ts`；`sourceMappingURL` 指向正确相对路径；在 bundler 中保留 `preserveModules` 以便映射对应。

---

## 35) 面向未来的类型策略：装饰器、Records & Tuples、TypedArray、解析策略演进
- 装饰器（TC39 新版）
  - TS 已支持**提案装饰器**；与旧版语义不兼容。面向公共库：**只支持新版**，对旧版提供编译期错误与迁移说明。
  - 避免强依赖 `emitDecoratorMetadata`（体积/不精确）；若必须用，提供**可配置开关**与**最小反射面**。

- Records & Tuples（不可变 `#[]`/`#{}`）
  - TS 侧尚无正式类型语义；短期可用 `readonly` + `as const` + 深比较工具**模拟不可变**，API 文档明确不可变约束，未来再无缝替换。

- TypedArray/结构化克隆/`ArrayBuffer` 族
  - 类型层采用更窄的接口（如 `ArrayBufferLike`、`TypedArray` 联合）抽象二进制数据；在运行时分支处理平台差异（Node vs Web）。

- 解析与打包生态
  - 采用 `moduleResolution: "bundler"`（TS 5）贴合 ESM 解析；`import type`/`export type` 明确类型通道，避免意外引入运行时依赖。
  - `exports` 条件：`"types"|"import"|"require"|"browser"`；对老 TS 配 `typesVersions` 兜底。
  - **拒用**：`namespace`、`const enum`、全局增强（除非必要）——降低迁移阻力。

- 兼容与回退
  - 提供**多目标产物**与**语义相同类型声明**（不暴露实现细节），把能力探测与降级策略放在**运行时**，让类型层保持稳定。
  - 在 CI 加入 TS 版本矩阵（如 4.9/5.0/最新），发现破坏性变更及时发布 **minor** 并给出 codemod/迁移文档。
