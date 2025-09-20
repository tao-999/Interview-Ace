# 前端架构

## 1) 单体前端 vs 微前端：选型、边界与代价

**何时单体**
- 团队 < 20 人、发布节奏一致、域模型集中、共享 UI/状态多。
- 优势：统一技术栈与治理、最低跨团队协调成本、首屏与运行时开销小。
- 边界：代码库规模增长导致构建/测试时长上升；模块隔离与权限边界依赖“组织纪律”。

**何时微前端**
- 多业务线独立路标与发布节奏（周更/日更）、多技术栈并存、需要“先并行后整合”。
- 优势：自治发布、技术多样性、出问题可局部回滚。
- 代价：运行时装配的**性能与一致性成本**（多份运行时、CSS/路由冲突、共享依赖治理）、**平台层**（壳应用、协议、运维）投入、跨应用体验一致性难度上升。

**边界划分**
- 以“领域边界 + 页面路由”切分优于“组件粒度”切分；共享能力（设计系统、鉴权、监控、路由协议、数据 SDK）下沉为平台。
- 统一**发布与回滚**接口、共享依赖版本与安全基线、错误隔离策略（Shadow DOM/iframe/进程内隔离约束）。

---

## 2) 微前端集成方式：Module Federation、iframe、Web Components、Runtime Composition

**Module Federation（MF）**
- 形态：构建期暴露 `remoteEntry`，运行时共享依赖与动态加载。
- 优点：运行时共享（singleton/版本协商）、路由/状态在同文档下无上下文切换。
- 难点：强绑定某类 bundler runtime；需要共享依赖规范（React/Vue 单例）、故障域非完全隔离（CSS/全局污染需额外治理）。
- 适用：大前台应用同技术栈、多团队并行。

**iframe**
- 优点：最强隔离（CSS/JS/安全域）、崩溃不影响宿主。
- 难点：跨帧通信（postMessage/Channel）繁琐、SEO/可访问性/路由与滚动同步差、首屏性能弱。
- 适用：强安全隔离、外部合作方接入、遗留系统包埋。

**Web Components（自定义元素 + Shadow DOM）**
- 优点：样式封装、可被任意框架消费、稳定的 DOM 契约。
- 难点：状态/路由/数据契约需自建；SSR 与水合能力因框架而异。
- 适用：跨框架 UI 契约、设计系统基座。

**Runtime Composition（Import Maps / SystemJS / 动态 import 装配）**
- 优点：与构建器弱耦合、可由平台动态切换版本（灰度/回滚）。
- 难点：**无共享依赖协商**，需要平台强制单例；合规与缓存策略更复杂。
- 适用：门户级聚合、跨仓库异步装配。

---

## 3) BFF（Backend for Frontend）：职责边界与 GraphQL/tRPC 协作

**职责**
- **聚合与裁剪**：面向页面/终端优化接口形态（N→1），去除冗余字段。
- **鉴权与风控**：处理 Token/Session、AB 授权、幂等性与速率限制；隐藏内部服务拓扑。
- **缓存与一致性**：短期缓存（内存/Redis）、ETag/Last-Modified；回源/熔断策略。
- **边缘适配**：设备差异、地域法规（数据驻留）、隐私过滤与脱敏。

**与 GraphQL 协作**
- 方案 A：**GraphQL 网关** 前置，BFF 作为 resolver 层（或子图），把页面语义沉到 schema；使用 Apollo Federation/GraphQL Mesh 聚合 REST/RPC。
- 方案 B：BFF 暴露**少量页面级 REST**，GraphQL 直达后端（复杂度移至前端查询）；不利于权限/风控统一。
- 折衷：GraphQL 提供读、BFF 处理写（事务/幂等/审计）。

**与 tRPC 协作**
- 类型直通 TS（端到端类型安全）；BFF 即 tRPC router，聚合与鉴权在 middleware 中处理。
- 注意：版本演进需避免紧耦合客户端/服务端发布节奏（引入路由版本/特性开关）。

---

## 4) SSR/CSR/SSG/ISR/Edge：差异与选择

| 模式 | 渲染时机 | 优势 | 劣势 | 适用 |
|---|---|---|---|---|
| CSR | 客户端 | 前后端分离、CDN 友好 | 首屏慢、SEO 差 | 纯内网/工具后台 |
| SSR | 请求时服务器 | 首屏快、SEO 佳、个性化 | 服务器成本、冷启动 | 动态内容、登录态强 |
| SSG | 构建时 | 速度与成本最佳 | 内容滞后 | 文档/营销/低变更 |
| ISR | 过期重建 | 兼顾实时与成本 | 一致性难、边缘缓存复杂 | 内容中等变化 |
| Edge SSR | 边缘节点 | 时延低、就近渲染 | 运行时限制/状态 | 地域敏感、轻逻辑 |

**选择策略**
- 登录态强/个性化：SSR/Edge SSR。
- 公共高流量内容：SSG + ISR。
- 交易/强一致：SSR + BFF 聚合 + 细粒度缓存禁用。
- 注意：水合成本（JS 负担）与缓存策略（私有 vs 公共）。

---

## 5) Hydration 方案演进：局部水合、渐进水合、岛屿（Islands）

**局部水合（Partial）**  
仅水合交互组件（其余静态），减少绑定开销。

**渐进水合（Progressive）**  
按可见性/优先级分批水合（IntersectionObserver/空闲时间），保证主内容可交互后再处理次要组件。

**岛屿架构（Islands）**  
SSR 输出“岛”占位，按岛为单位独立水合与加载依赖；不同岛可不同技术栈/优先级。

**实现要点**
- 岛作为**最小交互单元**；路由与全局状态在壳层，岛使用订阅/消息桥接。
- 避免重复运行全局初始化（一次性脚本上移到壳）。
- 支持 **按需 JS** 与 **按岛 CSS**；滚动/尺寸监听在岛内去抖。
- 与 streaming SSR 配合：先输出框架与关键岛，再陆续 flush 次要岛数据。

---

## 6) 路由架构：多入口/嵌套路由/动态路由与权限控制

**模块划分**
- 以“业务域 + 导航结构”划分顶层路由；子域内再按角色/任务分组（聚合 loader、共享布局）。
- **多入口**（MPA 或单应用多入口）：公共壳（设计系统、监控、国际化）+ 各入口独立构建与缓存。

**权限控制**
- **路由守卫**：进入前检查会话与角色（BFF 下发短期令牌 + 路由白/黑名单）。
- **组件粒度授权**：以能力点（permission key）控制组件渲染与交互禁用；在渲染层与数据层双校验。
- **数据域隔离**：同路由下，根据 tenant/project 做数据过滤；缓存 key 包含域上下文避免越权复用。

**动态路由**
- 路由表远端下发（特性开关/灰度）；本地将其“编译”为受控白名单，避免任意注入。
- 结合代码分割：路由级 `import()` + 预加载策略（hover/viewport/高频入口）。

---

## 7) 设计系统与组件库：Design Tokens、主题化与多品牌

**Design Tokens**
- 定义层级：`global`（品牌色/字重）→ `alias`（语义，如 `color.text.muted`）→ `component`（Button 的 padding）。
- 存储与下发：单一源（JSON/Style-Dictionary）编译为 CSS Vars/TS/Android/iOS 多目标产物；发布至私有 npm/registry。

**主题化**
- 运行时：CSS Variables + `data-theme` 切换；支持深浅/高对比模式，避免强制重绘。
- 多品牌：`brand-a.css`/`brand-b.css` 作为覆盖层；在壳应用按租户注入；组件内部仅消费语义 Token。

**组件库治理**
- 版本策略：SemVer + 破坏性变更 ADR；多应用并存时提供适配层与 codemod。
- 质量基线：可访问性（键盘/语义/对比度）、视觉回归（Storybook + 截图）、交互契约（Playwright）。
- 出口形态：ESM + 样式抽取（vanilla-extract/预编译 CSS），保留 Tree-Shaking。

---

## 8) 跨应用状态管理：UI 态 vs 服务器态；避免“全局状态泥沼”

**分类**
- **UI 态**：仅影响视图（弹窗开关、局部筛选）；局部 store/组件内部即可。
- **服务器态**：来自后端、可被其他会话修改；应采用 **请求层缓存**（TanStack Query/SWR/RTKQ），包含失效/重试/去重/乐观更新。

**跨应用/多包**
- 共享的数据访问 SDK（统一重试、错误、鉴权、埋点、节流）；缓存 key 包含用户/租户/过滤条件，避免“越域命中”。
- 避免把服务器态塞进全局 store（容易失效难题）；以 Query 层作为**单一可信源**，UI 只订阅 selector。
- 标签页同步：`BroadcastChannel`/`storage` 事件广播失效与会话变更。

---

## 9) 数据获取层：REST vs GraphQL vs RPC；缓存失效、乐观更新与冲突解决

**形态取舍**
- REST：简单、缓存友好（URL 即 cache key），难以规避 N+1。
- GraphQL：前端自选字段，减少过/欠取；需要服务器**复杂限流与权限**；缓存层（Apollo/urql）管理复杂。
- RPC（tRPC/gRPC-web）：端到端类型与高效；需协商错误语义与版本演进。

**缓存与一致性**
- 缓存键：方法 + URL/变量 + 租户 + 角色 + 语言。
- 失效：基于标签（`depends('app:users')` 类语义）或精确 URL 失效；服务端推送（SSE/WS）驱动客户端 `invalidate`。
- 乐观更新：本地先写/回滚；冲突解决策略：**最后写赢**（简单）或 **版本号/ETag**（安全）/CRDT（协作编辑）。
- 幂等与重试：写操作携带幂等键（Idempotency-Key）；对 5xx/网络错误进行指数退避；对重复提交容忍。

---

## 10) 性能基线与预算：目标制定与“超标即报警”

**指标目标（可按业务微调）**
- 首包 JS（gzip）：≤ 170–200 KB；首屏关键 CSS：≤ 20 KB。
- LCP（p75）：≤ 2.5s（移动 4G）；CLS（p75）：≤ 0.1；INP（p75）：≤ 200 ms。
- TTFB（SSR/Edge）：≤ 200 ms（区域内）。

**预算机制**
- 构建预算：CI 统计入口体积（gzip/br），超阈值直接 fail；记录依赖增量（新增第三方）。
- 运行预算：RUM 上报 Web Vitals（分国家/机型/网络），建立 p50/p75 报表与日常回归。
- 预加载策略检查：对首屏路由注入 `modulepreload/preload`，避免**网络瀑布**；静态资源长缓存（`immutable`）+ 运行时版本号确保命中。
- 质量守护：对大图/字体离线压缩与子集化；路由/组件级代码分割 + `prefetch`；关键交互降级路径（无 JS、慢网）。

---
## 11) 代码分割：路由级/组件级与预加载组合，避免“网络瀑布”

**切分维度**
- 路由级：每个页面作为异步 chunk（`import()`）；首屏仅加载壳 + 当前路由。
- 组件级：低频重组件（图表、富文本、地图）与实验特性按需分片；避免碎片化过多导致请求开销上升。

**预加载协同**
- `preload`：对“肯定马上用”的次级资源（当前路由的依赖）提升优先级。
- `modulepreload`：让浏览器提前解析模块依赖图（Vite/现代框架会自动注入）。
- `prefetch`：空闲时拉取“可能会用”的下个路由；在 hover/viewport 触发。
- `preconnect/dns-prefetch`：跨域资源（CDN/第三方）提前建连。

**反模式与治理**
- 动态路径拼接 `import(\`./${name}.js\`)` → 无法静态分析，改为显式路由表。
- “巨大 vendor 单块” → 用领域拆分（`react-vendor`、`chart-vendor`），限制 `maxSize` 强制切分。
- 懒加载即不可见：对首屏关键组件不要懒；对可见性驱动的组件用 IntersectionObserver 延迟挂载。

---

## 12) 资源与缓存：CDN、HTTP 缓存与 Service Worker 版本治理

**CDN 与命名**
- 所有静态产物指纹化（`[contenthash]`），HTTP 头：`Cache-Control: public, max-age=31536000, immutable`。
- HTML 不缓存或短缓存：`no-store` 或 `max-age=0, must-revalidate`，由服务器/边缘控制版本。

**HTTP 缓存**
- 变动不频繁的 API：`ETag/If-None-Match` 或 `Last-Modified/If-Modified-Since`。
- 公共 vs 私有：用户私有响应加 `private, no-store`；图片/字体走公共长缓存。
- `stale-while-revalidate`：对非关键数据提升感知速度。

**Service Worker（SW）**
- 策略分层：静态 `cache-first`；数据 `network-first`/`stale-while-revalidate`；离线兜底页。
- 版本治理：SW 文件名不加 hash，内容变更触发 `waiting` → `skipWaiting` + `clientsClaim`；以“**资源清单版本**”驱动旧缓存清理。
- 灰度：在 HTML 注入远端配置（%rollout%），SW 读取后决定拉取哪一版资源清单以实现金丝雀。

---

## 13) 边缘与多区域部署：Edge SSR 冷启动、状态一致性与回源

**冷启动与体积**
- 选择 Web 标准 API（无 Node 内置）、小体积 bundle；冷启动预算（p95）≤ 50ms（平台相关）。
- 预构建依赖、禁用大 JSON/正则；使用 KV/缓存作为热路径数据源。

**状态一致性**
- 读多写少：多区域只读 + 回源写（主区域）；写后用消息总线/CDC 通知边缘失效。
- 读写均衡：按租户/地理做分区；跨区一致性采用 **最终一致** + 并发控制（版本号/乐观锁）。
- 会话：边缘 Cookie 解码 + 轻鉴权，重权限走回源；或使用边缘可共享的 session（Signed JWT + 旋转）。

**回源策略**
- 优先就近缓存命中；MISS 回主区域 API；失败快速降级（静态片段/骨架 + 重试）。

---

## 14) 可观测性：RUM、日志/追踪/错误上报与抽样

**RUM 指标**
- Web Vitals（LCP/CLS/INP/TTFB/FID）分端、分地域、分网络上报；以 p75 作为 SLO。
- 自定义性能事件：首屏渲染、关键交互耗时、资源失败率。

**日志与追踪**
- 统一 Trace 上下文：前端生成 `traceparent`，随请求传到 BFF/后端，打通链路追踪。
- 结构化日志：JSON 行；含用户/租户/版本/路由/实验组。

**错误上报**
- JS 异常、Promise 未处理、资源错误（`error` + `currentTarget`）。
- Source Map 上传（hidden/nosources）；隐私脱敏（PII 屏蔽、URL 白名单）。

**抽样策略**
- 全量采集关键错误；性能/日志按 **动态采样**（错误多则降采、问题少则升采）；灰度阶段提高采样。

---

## 15) 错误边界与降级：前端容灾与后端熔断协同

**前端层**
- 视图级错误边界（React `componentDidCatch`/ErrorBoundary；Vue/Svelte 等等）+ 页面级兜底。
- 数据降级：读缓存/上次成功快照；隐藏非关键功能、展示占位/简版。
- 超时与重试：对幂等 GET 指数退避；可取消请求（AbortController）。

**与后端协同**
- 约定错误语义：可恢复（4xx） vs 不可恢复（5xx）；幂等键用于写操作重放安全。
- 熔断/限流：前端识别特定错误码转为降级 UI；BFF 返回 `Retry-After`、`X-RateLimit-Remaining` 指示节流。

**示例（React 错误边界）**
```tsx
class Boundary extends React.Component<{fallback: React.ReactNode},{e?:Error}> {
  state = { e: undefined };
  static getDerivedStateFromError(e: Error) { return { e }; }
  componentDidCatch(e: Error, info: React.ErrorInfo) { report(e, info); }
  render() { return this.state.e ? this.props.fallback : this.props.children; }
}
```

---

## 16) 发布策略：灰度/金丝雀/蓝绿；前端静态 + BFF 协同

**静态资源**
- 产物不可变（hash 文件）；通过 **远端配置** 或 Import Maps 指针切换版本（%asset_manifest%）。
- 金丝雀：按用户比例/租户/地区分流；失败回滚只需回退指针，不必回滚 CDN 文件。

**BFF 协同**
- BFF 返回当前前端版本指针与实验配置；同时控制 API 的版本路由（v1/v2）。
- 蓝绿：两套环境，DNS/负载均衡切换；客户端读取 `<meta data-env="blue">` 用于观测分群。

**回滚标准**
- 设定自动回滚阈值（错误率/崩溃率/LCP 恶化），触发时切回旧指针并冻结流量。

---

## 17) CI/CD：Monorepo 增量、缓存与多环境配置贯通

**增量构建**
- Nx/Turborepo 基于任务图与输入哈希命中缓存；开启远程缓存（CI↔本地互通）。
- TypeScript Project References + `tsc --build`，共享 `.tsbuildinfo`。

**缓存对象**
- 依赖安装缓存（store），构建缓存（dist/产物）、中间缓存（Vite `.vite`/Webpack `cache`）。
- Docker 多阶段：锁文件与 `package.json` 提前 COPY 命中层缓存。

**多环境配置**
- 参数化发布：`ENV/STAGE` 驱动构建时 `define` 与部署目标；Secrets 全在 CI/Secret Manager，禁止硬编码。
- 自动化质量关卡：sast/sca、lint/typecheck、单测、E2E、体积预算、Web Vitals 基线对比。

---

## 18) 依赖治理：第三方脚本/SDK 隔离与供应链安全

**第三方脚本隔离**
- `iframe sandbox` 或 **Worker**（可移除 DOM 权限）；最小权限 CSP（仅允许其域名与必要 API）。
- 延迟加载与优先级：非关键脚本 `defer/async` + `requestIdleCallback`；失败不阻塞首屏。

**供应链安全（SCA/SLSA）**
- 锁定源（`.npmrc`）、冻结 lockfile；CI 用 `npm ci`/`pnpm i --frozen-lockfile`。
- 审计：`npm/pnpm audit` + 第三方 SCA；对高危包阻断。
- SRI：外链 `<script>`/`<link>` 生成 `integrity`。
- 签名与来源：启用 provenance/Sigstore；构建产物与镜像签名。

---

## 19) 安全基线：CSP/Trusted Types、XSS/CSRF、SRI 与同源策略

**CSP**
- 最小化白名单：`script-src 'self' 'nonce-xxx' 'strict-dynamic'`；禁用 `unsafe-inline/eval`；`connect-src` 仅允许需要的域。
- `Trusted Types`：要求通过策略创建可执行字符串，阻断 DOM 注入类 XSS。

**XSS**
- 模板默认转义；对 `innerHTML`/`{@html}` 使用 DOMPurify；URL 协议白名单（http/https/mailto 等）。

**CSRF**
- 同源 Cookie：`SameSite=Lax/Strict`；跨站需求采用 **Origin/Referer 校验** 或双提交 Token。
- 对非幂等操作强制 POST/带 token。

**SRI 与 SOP**
- 外链脚本/样式使用 SRI；结合 `crossorigin`。  
- 同源策略为默认安全边界；跨域仅通过受控 CORS/代理。

---

## 20) 鉴权与会话：Cookie 模式 vs Bearer；SSO、刷新/撤销

**Cookie（Session/JWT in Cookie）**
- 优点：浏览器自动发送；结合 `httpOnly/secure/sameSite` 安全性高。
- 缺点：跨域复杂（需 CORS + `credentials` + 域配置）。
- 适用：同域/子域体系、SSR/边缘 SSR 便捷读取会话。

**Bearer Token（Authorization: Bearer）**
- 优点：跨域/跨端通用；适合 API/移动端。
- 缺点：存储风险（XSS 读取）；需前端妥善保管（内存/短寿命 + 刷新流）。

**刷新与撤销**
- 短寿命访问令牌 + 长寿命刷新令牌（仅服务端通道使用，避免泄露到 JS）。
- 刷新旋转（每次刷新都生成新 RT，旧的立即作废）；异常使用（地理/设备突变）触发全局撤销。
- 设备列表与会话管理：后端可撤销某设备令牌；前端收到 401 时清会话并引导重新登录。

**SSO**
- 同组织多应用：统一身份提供商（OIDC/SAML）；前端通过 `prompt=none` 静默续期或 Token 交换。
- 边缘 SSR：在边缘校验签名 + 解析最小化用户态，敏感数据只在后端聚合。

---
## 21) 权限模型：RBAC/ABAC 的前端表达与缓存；路由守卫、组件授权与数据域隔离

**模型与表达**
- **RBAC**（基于角色）：`user.roles ⊇ res.requiredRoles` → 粗粒度、易缓存。
- **ABAC**（基于属性）：`policy(subject, resource, env)` → 细粒度（时间/地域/租户/敏感级别）。
- 前端建议以**声明式能力点**（`perm: 'order.edit'`）为基本单位，角色/属性在 BFF 侧求值后下发**可缓存的权限快照**（含版本与过期）。

**缓存与失效**
- key = `userId|tenant|appVersion|permVersion`；在登录/切租户/策略变更时失效。
- 跨标签页用 `BroadcastChannel('auth')` 广播更新；边缘/SSR 提前注入当前快照避免闪烁。

**路由守卫与组件授权**
```ts
// 路由守卫（以 React Router 为例）
function Guard({need}: {need: string[]}) {
  const perms = usePerms(); // 来自 BFF 的快照或 Query
  if (!need.every(p => perms.has(p))) return <Navigate to="/403" replace/>;
  return <Outlet/>;
}

// 组件级授权
export function IfCan({need, children}:{need:string[], children:React.ReactNode}) {
  const perms = usePerms();
  return need.every(p => perms.has(p)) ? <>{children}</> : null;
}
```

**数据域隔离**
- 缓存 key 带 `tenant/projectId`，避免越权命中。
- 列表/图表查询加**服务端过滤**（后端再验权限），前端只作 UI 级不可见/禁用，不作为最终防线。

---

## 22) 配置与特性开关：远端配置中心、灰度/A-B、回滚与风险

**层次**
- **编译期开关**：`DEFINE` 常量 → DCE；只用于不可动态切换的结构性特性。
- **运行期开关**：远端配置/Flag 平台（百分比、用户集合、租户/地域标签）。

**可靠性与安全**
- **默认关闭/安全降级**：远端失败→本地默认值；重要开关与路由守卫/权限双向约束。
- **一致性**：SSR/边缘渲染需在请求期确定并注入 flag，避免首屏闪烁与 A/B 归因错误。
- **快照与回滚**：Flag 带版本与生效窗口；中心侧支持回滚与冻结。

**实现要点**
```ts
// Flag SDK（简化）
type Rule = { key:string; variant:'on'|'off'|'bucket'; rollout?: number };
export function decide(rule: Rule, ctx:{userId:string}) {
  if (rule.variant==='on') return true;
  if (rule.variant==='off') return false;
  const hash = murmur(ctx.userId + rule.key) % 100;
  return hash < (rule.rollout ?? 0);
}
```
- 统计与实验：埋点在**曝光时刻**记录 variant；做 **SRM 检测**（样本不平衡报警）。

---

## 23) 大型表格/可视化：虚拟化、增量渲染与跨页缓存一致性

**渲染**
- 固定高虚拟化优先（仅渲染可见行 + overscan）；可变高用 `ResizeObserver` 回报高度与前缀和索引。
- 图表采用**数据分层**：视窗数据（UI 层）+ 背景聚合（服务端/Worker 预聚合）。

**数据流与一致性**
- 统一通过 **Query 层**（TQ/RTKQ/SWR），列表分页 key = `endpoint|filters|tenant|cursor`。
- **乐观更新**：行内编辑先写缓存；失败回滚并提示冲突。
- **跨页共享缓存**：打开详情页复用同一 Query 实例；在返回时无需重新拉取。
- **失效广播**：WebSocket/SSE 推送标签式失效（如 `users:*`），客户端 `invalidateQueries(['users'])`。

**性能要点**
- 行内控件用**受控+批量提交**方案；事件委托到 `<tbody>`。
- 图表逐帧（`requestAnimationFrame`）合并更新；大样本下用 WebGL/Canvas 层。

---

## 24) 多包/多应用协调：Monorepo vs Polyrepo；版本策略与 API 稳定性

**Monorepo**
- 优点：共享工具链、跨包重构、原子提交、远程缓存（Nx/Turbo）。
- 风险：根配置复杂、CI 资源消耗；需**任务图 + 缓存**控制爆炸。

**Polyrepo**
- 优点：边界清晰、团队自治。
- 风险：版本地狱、接口漂移；需要契约与发布纪律。

**版本与契约**
- **SemVer + Changeset**；破坏性变更配 ADR 与迁移文档。
- **Contract Test**：下游用真实构建消费上游 **发布候选**；OpenAPI/GraphQL SDL 作为单一事实源，CI 校验**兼容性**。
- **依赖区间**：应用锁死（`^` 谨慎），库对 peerDeps 声明宿主范围。

---

## 25) TypeScript 架构：Project References、公共类型包、API 合同驱动

**Project References**
```json
// tsconfig.json (root)
{ "files": [], "references": [{ "path": "packages/types" }, { "path": "packages/app" }] }
```
- 子包启用 `"composite": true` 输出声明与 `.tsbuildinfo`，加速增量。

**公共类型包**
- 导出 Design Tokens、PermKeys、API 类型（生成于 `@types/app`），应用只依赖此包。
- 版本随 API 合同演进（SemVer），破坏性变更伴随 codemod。

**合同驱动**
- OpenAPI/GraphQL → 代码生成器产出 **客户端 SDK + TS 类型**；CI 校验是否**非兼容变更**（删除字段/收紧枚举）。

---

## 26) 浏览器兼容与现代化：多目标构建、差分下发、Polyfill 验证

**多目标**
- 现代包（ESM, ES2018+）+ 旧版包（SystemJS/legacy）双轨；按 UA/`nomodule` 差分加载。
- Polyfill：以 **browserslist** 与真实使用分析（`coverage`、RUM）确定最小集。

**验证**
- 端到端矩阵（浏览器×设备×网络）；自动化用 Sauce/BrowserStack；页面脚本注入当前特性快照与错误统计，发现**隐性兼容**问题。

**成本控制**
- Legacy 仅针对**使用占比阈值以上**地区开启；按路由/页面进行差分而非全站。

---

## 27) 国际化与可达性：动态语言包、RTL、a11y 基线与流水线

**i18n**
- 语言包按 **语言×命名空间** 分片懒加载；`hreflang` + 路由前缀 `/en/…`。
- 服务器注入 `lang/dir` 与首屏字典，避免闪烁；客户端哈希版号缓存语言包。

**RTL**
- 使用逻辑属性（`margin-inline-start`）与 CSS 自适应；图标/箭头注意镜像。
- 组件库内置 `dir` 切换与快照测试（LTR⇄RTL）。

**a11y**
- 基线：键盘可达、语义标签、对比度、焦点管理（模态/菜单）、表单关联。  
- 流水线：`axe-core` 静态检查 + E2E 可达性断言；PR 阶段对关键页面跑无障碍门槛。

---

## 28) DOM/样式策略：CSS Modules/Tailwind/CSS-in-JS/vanilla-extract 的规模化取舍

**对比**
- **CSS Modules**：零运行时、作用域清晰；复用与主题化需约定。
- **Tailwind**：原子类，JIT 产物小；复杂组件可读性与约束需规范（`@apply`/`variants`）。
- **CSS-in-JS（runtime）**：灵活动态样式，但**运行时开销**与 SSR 抖动；慎用在大流量页面。
- **静态抽取（vanilla-extract/Linaria）**：零运行时 + 类型安全，适合设计系统。

**策略**
- 站点层面优先 **原子类 + 静态抽取**；对极少数需动态计算的场景（主题色渐变、用户自定义）再落回 runtime。
- 设计系统提供**语义 token**与组件变体 API，避免样式散落。

---

## 29) 前端缓存层：Query 缓存、持久化、跨标签同步与失效广播

**Query 层**
- 默认 `staleTime`、`cacheTime` 分档；Key 含入参/租户/语言。
- **持久化**：`@tanstack/query-persist-client` + IndexedDB；携带 schema 版本，升级时迁移或丢弃。

**跨标签同步**
```ts
const bc = new BroadcastChannel('query');
bc.onmessage = (e) => queryClient.invalidateQueries(e.data.tag);
function announce(tag:string){ bc.postMessage({tag}); }
```

**失效来源**
- 用户操作（写后）、SSE/WS 推送、BFF 返回 `ETag` 变化信号。
- 后端下发**命名失效**（如 `cart:*`）；前端映射到 Query Key 前缀。

---

## 30) 离线与 PWA：缓存/同步/冲突合并与一致性权衡

**策略**
- 静态：`cache-first`；数据：`network-first` + 回退。
- **后台同步**：Background Sync/Periodic Sync；失败队列持久化（IndexedDB），网络恢复后重放（带幂等键）。
- **冲突合并**
  - 简单：时间戳/版本对比（客户端比服务器旧→提示覆盖或合并）。
  - 协同：CRDT/OT（文本/白板）。

**权衡**
- 交易/支付→**强一致**，离线只读/禁写。
- 社交/非关键→**最终一致**，允许短暂分叉。

---

## 31) 可插拔架构：插件系统（扩展点/生命周期/权限沙箱）与远程热插拔

**扩展点模型**
- 事件型：`on(event, handler)`；插槽型：`register(slot, component)`；命令型：`execute('print', args) → result`。
- 生命周期：`init → activate → update → deactivate → dispose`；插件可声明依赖与版本范围。

**安全与隔离**
- **权限清单**（`permissions: ['storage','net:api.example.com']`）；运行前审计与用户授权。
- **沙箱**：iframe/Worker/Realms（实验）；只暴露受控 API；资源上限（CPU/内存/请求速率）。

**远程模块**
- 通过 Import Maps/MF 受控加载；指纹/签名校验；灰度发布与回滚由平台控制。

---

## 32) 观测与实验平台：SDK 规范、隐私合规与 A/B 归因

**SDK 规范**
- 统一 `context`（user/tenant/app/version/locale）与 `sessionId`；事件 schema 版本化。
- 失败重试/脱机队列/批量上报；异常防抖与采样。

**隐私**
- PII 识别与**本地裁剪**（白名单字段）；GDPR 权限与撤回；Do-Not-Track/同意管理（CMP）集成。

**实验**
- 随机化与粘滞（user-based hashing）；**曝光**即记录 variant；SSR 注入 variant 保持一致。
- 统计：设定最小样本、功效、最短实验周期；做 **SRM** 与 **连续监控偏差** 防御；必要时多臂 bandit。

---

## 33) SEO 与可分享性：SSR/边缘渲染、OG、站点地图、国际化与 canonical

**元信息**
```html
<meta name="description" content="...">
<meta property="og:title" content="...">
<meta property="og:description" content="...">
<meta property="og:image" content="...">
<link rel="canonical" href="https://example.com/en/post/slug">
<link rel="alternate" hreflang="en" href="https://example.com/en/post/slug">
<link rel="alternate" hreflang="zh" href="https://example.com/zh/post/slug">
```
- SSR/Edge 渲染首屏元信息；避免客户端再替换造成抓取差异。
- 站点地图（动态 + SSG 列表）与 `robots.txt`；分页与筛选页慎设 canonical。

**国际化**
- 路由前缀 + `hreflang`；避免区域化内容互相竞争索引。

---

## 34) ADT/ADR：架构决策记录的落地流程与模板

**流程**
1. 提案（问题背景/备选方案/权衡/影响范围）。
2. 评审（架构小组 + 业务代表），形成结论与迁移路径。
3. 落地（PoC→试点→全量），记录里程碑与回退策略。
4. 事后评估（指标与教训，是否需要修订）。

**模板（要点）**
- Context（背景与约束）
- Decision（结论）
- Options（A/B/C 与权衡矩阵：成本/风险/性能/可维护）
- Consequences（正/负影响，技术债）
- Rollout（迁移计划、回滚、度量）
- Status（Proposed/Accepted/Deprecated/Replaced）

---

## 35) 演进与迁移：双栈与技术债清理（框架/路由/构建工具）

**总体策略**
- **Strangler**：新功能走新栈；老模块在边界适配（Adapter/Anti-corruption）。
- **双栈共存**：路由网关/壳应用桥接（同域、同会话）；共享设计系统与数据 SDK。

**迁移序**
1. 建**兼容层**（设计系统/数据 SDK/路由协议）。
2. 优先迁移独立模块（低耦合、高收益）。
3. 热点页面与性能瓶颈路由（配合监控评估收益）。
4. 移除旧链路：CI 禁止新代码进入旧栈，冻结旧依赖。

**度量与回退**
- 指标：构建时长、体积、Web Vitals、故障率；每阶段 ADR 附带目标与验收。
- 回退：入口开关/Import Maps 指针；出问题迅速切回旧版本并记录根因。
