# 1) HTTP/1.1 vs HTTP/2 vs HTTP/3（QUIC）

**并发与 HoLB（Head-of-Line Blocking）**
- **HTTP/1.1**：单连接**队头阻塞**严重；流水线（pipelining）基本不可用 → 现实中靠多连接并发（6~10 个域名连接）。
- **HTTP/2**：单 TCP 连接内**多路复用**（流 ID + 二进制帧）；应用层不阻塞，但**传输层 HoLB**仍在——任何丢包会阻塞该 TCP 上**所有流**的后续数据。
- **HTTP/3(QUIC)**：基于 UDP 的 QUIC 拥有**流级独立可靠性**，单个流丢包仅阻塞**该流**，**无传输层 HoLB**。

**优先级与流量控制**
- **H2**：曾有**依赖树**优先级（多被忽略或简化）；有**连接级 + 流级**窗口（`WINDOW_UPDATE`）。
- **H3**：基于 **HTTP Priority**（优先级信号 Header/Frame）；底层使用 QUIC 的**连接/流级**流控（接收窗口、最大数据量）。

**握手与加密**
- **H1/H2 over TLS**：`TCP 3WHS` + `TLS(1.2/1.3)`；**ALPN** 协商 `h2`/`http/1.1`。
- **H3**：QUIC 内建 **TLS 1.3**，**1-RTT** 完成握手；会话复用可 **0-RTT**（有重放风险，服务端需幂等保障）。

**头部压缩**
- **H2**：HPACK（静/动态表）。
- **H3**：QPACK（为避免队头阻塞，解耦解码与数据流）。

**实战建议**
- 站点全量 **TLS1.3 + H2/H3**；对慢网/高丢包的列表页或首屏，H3 往往更稳。
- 合理使用 **Priority Hints** 与资源分片，避免“把关键渲染资源让位给次要资源”。

---

# 2) TCP / UDP / QUIC：握手、丢包、拥塞与体验

**建立/关闭**
- **TCP**：`3WHS` 建连、**四次挥手**关闭；可靠、有序、拥塞控制在传输层。
- **UDP**：无连接、无握手；可靠性需应用层自管。
- **QUIC**：基于 UDP，**内建 TLS1.3** 与可靠传输；**1-RTT**（可 0-RTT 恢复），连接迁移（连接 ID）更友好移动网络。

**丢包与重传**
- **TCP**：按字节序号、**单个分片丢失阻塞后续**字节交付（HoLB）。
- **QUIC**：包有独立编号 + **ACK ranges**，**流级重传**互不影响。

**拥塞控制**
- 典型：**CUBIC**、**BBR**；QUIC 复用同类算法，但因在用户态可更快演进。

**前端体验**
- 高丢包环境：**H3/QUIC 更抗抖**；移动切换（蜂窝↔Wi-Fi）时 H3 的**连接迁移**减少重连卡顿。
- 0-RTT 仅用于**幂等/可重放**请求；业务需校验。

---

# 3) TLS 基础：证书链、SNI、ALPN、HSTS 与浏览器校验

**证书链与验证**
- 浏览器构建**信任链**（Leaf → 中间 CA → 根 CA（信任库）），校验：
  - 域名（SAN/ CN）匹配、有效期与撤销状态（OCSP/CRL）。
  - 签名算法安全、密钥长度合规。

**SNI / ALPN**
- **SNI**：在 TLS ClientHello 指定目标主机名（同 IP 部署多证书）。
- **ALPN**：协商应用协议（如 `h2`/`http/1.1`/`h3`）。

**HSTS**
- 告诉浏览器**只用 HTTPS** 访问（含子域可选），防降级攻击；可加入浏览器 **Preload List**。

**中间人防护**
- 强制 HTTPS（HSTS）、证书链校验 + pin 验证（APP 侧更常见）、禁用弱套件/协议（≥ TLS1.2）。

---

# 4) DNS 与延迟：记录、TTL、递归；`dns-prefetch` / `preconnect`

**记录与解析**
- **A/AAAA**（IPv4/IPv6）、**CNAME**（别名）、**NS**（权威服务器）、**TXT**（SPF/校验）。
- **TTL**：缓存寿命；CDN 场景常较短以便灵活调度。
- **递归解析**：浏览器/系统 → 递归 DNS → 权威 DNS，**每步 RTT 都算延迟**。

**预解析/预连接**
- `<link rel="dns-prefetch" href="//img.example.com">`：仅做 **DNS** 解析。
- `<link rel="preconnect" href="https://img.example.com" crossorigin>`：预建 **DNS+TCP+TLS**，首包明显提前。

**实践**
- 对“**首屏一定会用**”的跨域静态源做 `preconnect`；仅可能使用的域名做 `dns-prefetch`，避免浪费 socket/FD。

---

# 5) 缓存体系：`Cache-Control`、ETag/Last-Modified、强/协商缓存与 304

**指令速览**
- `max-age`/`s-maxage`（共享缓存）、`public/private`、`immutable`、`no-store`、`no-cache`（需再验证）、`stale-while-revalidate`、`stale-if-error`。

**强缓存 vs 协商缓存**
- **强缓存**：未过期直接用（200 from disk/memory）；  
- **协商缓存**：带 `If-None-Match`（ETag）或 `If-Modified-Since`（Last-Modified），命中返回 **304**。

**ETag vs Last-Modified**
- **ETag**：内容指纹（强/弱 `W/`）；精确、但生成有成本。  
- **Last-Modified**：粒度到秒，依赖文件时间；可能误判（时钟、构建管线）。

**最佳实践**
- 产物文件名带 **内容哈希** → `Cache-Control: immutable, max-age=31536000`；HTML `no-store`。
- 对非哈希资源启用 **SWR**，感知上“秒开”，后台再验证更新。

---

# 6) 服务端与 CDN 缓存联动：`Vary`/`Surrogate-Control`、缓存键与回源

**缓存键（Cache Key）**
- 由 **URL + 规范化查询 + 关键请求头** 构成。常见加入：`Accept-Encoding`、`Accept-Language`、`User-Agent(移动/桌面)`、A/B 实验标识（谨慎，防爆炸）。

**`Vary` 与 `Surrogate-Control`**
- **Vary**：告知下游缓存**依据哪些请求头区分副本**（如 `Vary: Accept-Encoding, Accept-Language`）。
- **Surrogate-Control**：给**边缘（CDN）**的独立指令，不影响浏览器（如 `max-age=86400, stale-while-revalidate=600`）。

**回源与保护**
- **多区域回源**/回源保护（origin shield）降低“雪崩”；  
- 对相同资源的并发 miss 做**请求折叠/合并**；  
- 失败快速回退到上一个可用版本（`stale-if-error`）。

---

# 7) 内容协商：`Accept-*`、`Content-Encoding`、`Content-Type` 与 `Vary`

**常见协商维度**
- **编码**：`Accept-Encoding: br, gzip, zstd` ↔ `Content-Encoding: br|gzip|zstd`。  
- **语言**：`Accept-Language` ↔ 语言包（同时 `Vary: Accept-Language`）。  
- **类型**：`Accept: text/html, application/json` ↔ `Content-Type`。

**要点**
- 压缩应与 `Vary: Accept-Encoding` 搭配，避免把 br 结果缓存给不支持的客户端。  
- 对移动/桌面模板差异，可自定义协商头（如 `X-Device-Class`）并纳入 `Vary`（注意 cache key 膨胀）。  
- JSON 默认 `Content-Type: application/json; charset=utf-8`，避免 `text/plain` 破坏解析与缓存策略。

---

# 8) 状态码语义（含冷门）与重定向缓存

**冷门但常用**
- **103 Early Hints**：提前下发 `<link rel=preload>`。  
- **204 No Content**：成功但无实体（表单回执）。  
- **206 Partial Content**：配合 `Range`。  
- **308 Permanent Redirect**：类似 301，但**保持方法**（不会把 POST 变 GET）。  
- **412 Precondition Failed**：条件请求（If-Match 等）失败。  
- **421 Misdirected Request**：H2 代理/多主机证书误发。  
- **429 Too Many Requests**：限流（配合 `Retry-After`）。  
- **451 Unavailable for Legal Reasons**：法律原因不可用。

**重定向与缓存**
- **301/308** 默认可缓存（有时长）；**302/303/307** 通常不可缓存除非显式 `Cache-Control`。  
- 浏览器会缓存**域名级 301**，谨慎发布，保留**回滚路径**（临时 302 → 观测 → 再 301）。

---

# 9) 幂等与重试：语义、陷阱与 Idempotency-Key

**HTTP 语义**
- **幂等**：GET/HEAD/PUT/DELETE（理论上）；POST 非幂等。
- **现实陷阱**：PUT/DELETE 的副作用计数/审计日志、外部回调可能使其“看似非幂等”。

**重试设计**
- **网络/5xx** 可重试；**幂等**操作直接退避重试（指数退避 + 抖动）。  
- **非幂等**用 **Idempotency-Key**（客户端生成唯一 key，服务端缓存首次结果，后续同 key 返回相同响应）：
  - Key 需与**请求体哈希**绑定；设置**过期**与**幂等窗口**；在多活/边缘同样共享。

---

# 10) 分块与流式：`Transfer-Encoding: chunked`、Fetch Streams 与背压

**HTTP 层**
- **H1**：未知实体长度时用 `Transfer-Encoding: chunked`；服务器可边生成边发送。  
- **H2/H3**：不再使用 `chunked` 头，改用**帧/数据流**语义，但同样支持**流式**发送。

**前端读取（ReadableStream）**
```js
const res = await fetch('/stream');
const reader = res.body.getReader();
const decoder = new TextDecoder();
let received = '';
for (;;) {
  const { done, value } = await reader.read();   // 背压：只有消费后才继续拉取
  if (done) break;
  received += decoder.decode(value, { stream: true });
  render(received); // 增量渲染
}
```
- **背压（backpressure）**：读取端节流生产端，避免内存爆。`reader.read()` 只有在前一次消费后才推进。
- **边界**：某些代理/平台会**缓冲**响应，导致“假流式”；服务端需关闭缓冲（如 Nginx `proxy_buffering off`）、禁用过度聚合。

**常见场景**
- 服务端**增量渲染**（SSR Streaming / RSC）、日志推送、大模型**逐 token**输出。  
- 客户端配合 **AbortController** 超时与取消；渲染层做**增量追加**而非整块替换，减少重排。

```js
const ctrl = new AbortController();
setTimeout(()=>ctrl.abort(), 15000);
const res = await fetch('/stream', { signal: ctrl.signal });
```
> 结论：流式让**首字节更早**、吞吐更稳定，但要确保链路全程（应用 → 代理 → CDN → 浏览器）不开**缓冲**与**压缩合并**，并设计良好的超时/取消与重连策略。
---
# 11) 大文件与断点续传：Range / 206 / 并发分段 + 一致性校验

**核心机制**
- **请求头**：`Range: bytes=start-end`；服务端命中返回 **206 Partial Content**，并带 `Content-Range: bytes start-end/total`。
- **并发分段**：按块（如 4–16MB）切分，**并行拉取**后按 `start` 排序拼装；或用**多 Range**（`Range: bytes=0-1,10-11,...`，支持差）一次请求。
- **校验一致性**：用 **ETag + If-Range**：
  - `If-Range: "<etag>"`：当且仅当 ETag 未变时返回 206，**变了就回 200 + 全量**，避免错拼。
  - 也可用 `If-Range: <HTTP-date>` 对比 `Last-Modified`。

**要点**
- 断点续传需记录：`url/size/etag/chunkSize/已完成块位图`。
- H2/H3 下并发分段**不必多连接**，同连接多路复用即可。
- 合并前做 **MD5/CRC/ETag 对齐**，失败则**丢弃重来**该分段。

**示例（并发 Range）**
```js
async function rangeFetch(url, size, chunk=8<<20, etag=null) {
  const n = Math.ceil(size / chunk);
  const parts = await Promise.all([...Array(n)].map(async (_,i)=>{
    const start = i*chunk, end = Math.min(size-1, start+chunk-1);
    const res = await fetch(url, { headers: { Range: `bytes=${start}-${end}`, ...(etag && { "If-Range": etag }) }});
    if (res.status !== 206) throw new Error(`Bad status ${res.status}`);
    return new Uint8Array(await res.arrayBuffer());
  }));
  // 拼装
  const out = new Uint8Array(size);
  let o=0; for (const p of parts) { out.set(p, o); o+=p.length; }
  return out;
}
```

---

# 12) 同源策略与 CORS：预检、允许头、凭证、opaque 缓存

**何时触发预检（OPTIONS）**
- 方法非 **简单方法**（非 GET/POST/HEAD）。
- 自定义头或**非简单 Content-Type**（非 `application/x-www-form-urlencoded`/`multipart/form-data`/`text/plain`）。
- `fetch` 显式 `mode:"cors"` 且不满足简单请求条件。

**服务端响应**
```
Access-Control-Allow-Origin: https://app.example.com   // 或 *
Access-Control-Allow-Methods: GET,POST,PUT
Access-Control-Allow-Headers: x-token, content-type
Access-Control-Max-Age: 86400
Vary: Origin, Access-Control-Request-Headers, Access-Control-Request-Method
```

**凭证模式**
- 要携带 Cookie/认证：`fetch(..., { credentials: "include" })` + 服务端 **不得用 `*`**，必须回显具体 Origin，并加
  `Access-Control-Allow-Credentials: true`。
- **opaque 响应**（`mode:"no-cors"`）不可读 body/头，不可可靠缓存细粒度（仅用于像素上报、Beacon 等）。

**常见坑**
- 忘记 `Vary: Origin` → CDN 把某个 Origin 的响应缓存给了别的站。
- 预检被 WAF/代理拦截；或预检路径未放开。

---

# 13) 跨源隔离：COOP / COEP / CORP 与 SharedArrayBuffer

**目标**：开启 `crossOriginIsolated`（`window.crossOriginIsolated === true`）以使用 **SharedArrayBuffer**、高精度计时等。

**要求**
- 页面设置：
  ```
  Cross-Origin-Opener-Policy: same-origin
  Cross-Origin-Embedder-Policy: require-corp       // 或 credentialless
  ```
- 所有**跨源子资源**（脚本、图片、字体、wasm、workers…）必须满足：
  - **CORS 可获取**（`Access-Control-Allow-Origin` 允许）**或**
  - 资源方设置 **`Cross-Origin-Resource-Policy: same-site|same-origin`**（CORP）。
- HTTPS & 无混合内容。

**权衡**
- 会“**隔离浏览上下文组**”，跨站 window 引用会被切断；第三方资源需要改造（加 CORS/CORP），否则加载失败。
- 可用 `COEP: credentialless` 降低第三方要求（不带凭证以匿名方式获取）。

**最小可用 Nginx**
```nginx
add_header Cross-Origin-Opener-Policy same-origin always;
add_header Cross-Origin-Embedder-Policy require-corp always;
```

---

# 14) 安全基线：CSP / Referrer-Policy / frame-ancestors / Mixed Content

**CSP 推荐模板（严格版，按需放宽）**
```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-<RANDOM>' 'strict-dynamic';
  style-src  'self' 'unsafe-inline';           # 或改用 nonce
  img-src    'self' data: https:;
  connect-src 'self' https://api.example.com;
  font-src   'self' https:;
  frame-ancestors 'self';
  upgrade-insecure-requests;
  report-to csp-endpoint; report-uri https://r.example.com/csp;
```
- **`nonce`/`hash`**：允许特定内联脚本，配合 `strict-dynamic` 只信任由受信脚本动态加载的后续脚本。
- **Referrer-Policy**：`strict-origin-when-cross-origin`（常用安全默认）。
- **`frame-ancestors`**（优先级高于 `X-Frame-Options`）：限制谁能嵌你页面，防点击劫持。
- **Mixed Content**：`upgrade-insecure-requests` 自动升级 http→https；必要时清理遗留 http 资源。

---

# 15) Cookie 策略：SameSite / Secure / HttpOnly / 作用域 / 子域共享

**关键属性**
- `SameSite=Lax`（默认）：跨站**顶级导航**会发送，嵌入/子资源不发。
- `SameSite=Strict`：仅同站发送。
- `SameSite=None`：**必须 `Secure`**（仅 HTTPS），用于跨站场景（第三方登录、CDN 域名共享）。
- `HttpOnly`：JS 不可读，防 XSS 窃取。
- `Domain`/`Path`：`Domain=.example.com` 允许子域共享；不设则仅 host 级。
- 前沿：**`Partitioned`**（CHIPS）将第三方 Cookie **按顶级站点分区**，减少跨站跟踪与冲突。
- 前缀：`__Host-`（必须 Secure、无 Domain、Path=/）、`__Secure-`（需 Secure）。

**会话粘滞**
- 负载均衡粘滞可通过 **Set-Cookie**（如 `route=shard-a; Path=/; HttpOnly`）；注意与业务 Cookie 分离。
- 大小限制：单 Cookie ≈ 4KB，总数/总大小浏览器有上限；避免滥用。

---

# 16) 认证与会话：JWT vs Cookie Session、刷新令牌与安全存储

**两派**
- **Cookie-Session**：服务端存会话，前端凭 `Cookie`；优点：**HttpOnly + SameSite** 抗 XSS/CSRF、撤销简单；缺点：跨域需 CORS+credentials。
- **JWT（Bearer）**：前端持 `Authorization: Bearer <token>`；优点：跨域易、边缘可自验证；缺点：**撤销困难**、泄露风险。

**刷新与轮换**
- **短期 Access + 长期 Refresh**；Refresh 放 Cookie（HttpOnly, SameSite=Lax/Strict），Access 放 **内存变量**（非 localStorage）以降低持久泄露。
- **刷新轮换**（Refresh Token Rotation）：每次刷新生成新 Refresh，旧的立即作废，防重放。

**CSRF 与 XSS**
- Cookie 模式：SameSite + **CSRF token**（双提交/自定义头）兜底。
- Bearer 模式：避免放 localStorage；若必须，结合 **CSP + iframe 沙箱** 降风险。

**跨域携带**
```js
fetch('https://api.example.com/data', {
  credentials: 'include',   // Cookie 会带上
  headers: { 'X-CSRF-Token': token }
});
```

---

# 17) 网络优先级：H2 旧优先级 vs HTTP Priority；`fetchpriority` 与 `preload`

**协议层**
- **H2**：依赖树优先级，多数服务器/代理**忽略或简化**。
- **HTTP Priority（H2/H3）**：`priority`（请求头）或 PRIORITY 帧，携带紧急度（urgency）/渐进性（progressive）信号，更易被实现。

**页面层**
- **`fetchpriority`**（HTML 属性）：浏览器层面给关键资源（如首屏图片）“拉高/降低”加载优先级：
  ```html
  <img src="hero.jpg" fetchpriority="high" />
  <link rel="preload" as="image" href="hero.jpg" imagesrcset="..." fetchpriority="high">
  ```
- **`<link rel="preload">`**：**强制下载**且需要 `as` 正确，否则优先级/缓存错配。
- 二者配合：preload 负责“**一定要下**”，`fetchpriority` 决定“**多快下**”。

**实战**
- 首屏关键图/字体设 `fetchpriority="high"`；**非首屏**关键资源禁用高优，以免挤占 CSS/JS/首屏图。

---

# 18) 预加载族：`preload` / `prefetch` / `prerender` / `preconnect` + 103 Early Hints

**语义与时机**
- `preconnect`：提前建 **DNS+TCP+TLS**；首屏必用域名。
- `preload`：**当前导航**一定会用的资源（高优）；需 `as`、`crossorigin`（字体）正确。
- `prefetch`：**下一次导航**可能用的资源（低优、可丢弃）。
- `prerender`：后台渲染下一个页面（域名/权限受限，按浏览器支持谨慎使用）。
- **Early Hints (103)**：服务器在主响应前即下发 `Link: <...>; rel=preload`，抢跑关键资源。

**坑点**
- `preload` 未消费会 **浪费带宽**；相同资源不同 `as` 会**重复下载**。
- 字体要加 `crossorigin` 与 `Vary: Origin`/`Timing-Allow-Origin`。
- 服务端/代理需关闭缓冲，Early Hints 才能真正提前。

**103 示例（响应前）**
```
HTTP/1.1 103 Early Hints
Link: </app.css>; rel=preload; as=style
Link: </app.js>; rel=modulepreload; as=script
```

---

# 19) 图片 / 字体传输：`srcset/sizes`、Client Hints、`font-display`、子集化与缓存键

**响应式图片**
```html
<img
  src="img-800.jpg"
  srcset="img-400.jpg 400w, img-800.jpg 800w, img-1200.jpg 1200w"
  sizes="(max-width:600px) 100vw, 600px"
/>
```
- 浏览器按 `sizes` 与屏宽选择最合适资源；避免“800px 容器下却下了 2K 图”。

**Client Hints**
- 服务端按 `DPR/Width/Viewport-Width` 动态裁图：启用 `Accept-CH: DPR, Width`，并在响应设置
  `Vary: DPR, Width`，必要时回写 `Content-DPR`。
- 结合 **CDN 裁剪**（`/img.jpg?w=600&dpr=2`）减少原图回源。

**字体**
- 使用 **woff2** + 子集化（按 unicode-range 切片）；首屏必要字体 **preload**：
  ```html
  <link rel="preload" as="font" href="/fonts/ui-regular.woff2" type="font/woff2" crossorigin>
  ```
- `font-display: swap|optional`：避免 FOIT（空白期）。
- 字体跨域需 `Access-Control-Allow-Origin`，并纳入 `Vary`。

---

# 20) 上传与关闭页：`fetch({ keepalive:true })` vs `navigator.sendBeacon`

**`sendBeacon`**
- 设计用于**页面卸载**时的简短上报：**异步、可靠**，不会阻塞跳转；**仅 POST**，**无响应可读**。
- **体积限制**：实现相关，通常 ~64KB 级；类型常为 `text/plain`/`application/x-www-form-urlencoded`/`Blob`。
```js
navigator.sendBeacon('/metrics', new Blob([JSON.stringify(payload)], { type: 'application/json' }));
```

**`fetch(..., { keepalive:true })`**
- 允许在卸载阶段继续进行请求；**方法不限**、可读响应；**也有体积上限**（同样通常 ~64KB），超限可能被丢弃。
```js
await fetch('/log', { method:'POST', body: data, keepalive: true, headers:{'content-type':'application/json'} });
```

**取舍**
- **埋点/心跳**：`sendBeacon` 更稳，不关心响应。
- **必须拿响应**（如确认票据/会话关闭）：`keepalive fetch`；注意**超时+重试**与**幂等**。
- 不要在 `beforeunload` 里做**同步阻塞**；统一把大包拆小，平时批量发送，卸载时只补尾。

**兜底**
- SW（Service Worker）离线队列：把未发出日志存 IndexedDB，后台再补传；这样卸载期只需很小尾包。
---
# 21) 取消与超时：AbortController、超时/重试（退避）与幂等设计

**取消与超时范式**
- 使用 **`AbortController`** 统一中止：用户离开页面、切换筛选、上一次搜索应立即取消。
- 封装**超时**到控制器（而非 race Promise）便于链路内所有 fetch 感知：
```js
function fetchWithTimeout(input, init = {}, ms = 10_000) {
  const ac = new AbortController();
  const id = setTimeout(() => ac.abort(new DOMException('timeout','TimeoutError')), ms);
  const merged = { ...init, signal: init.signal ? anySignal([init.signal, ac.signal]) : ac.signal };
  return fetch(input, merged).finally(() => clearTimeout(id));
}
// anySignal: 多 signal 合并（可用 npm:anysignal 或自行实现）
```

**重试与退避（Exponential Backoff + Jitter）**
```js
async function retry(fn, {retries=3, base=300, factor=2, jitter=true, isRetryable}={}) {
  let attempt = 0;
  for(;;) {
    try { return await fn(); }
    catch (e) {
      if (++attempt > retries || (isRetryable && !isRetryable(e))) throw e;
      const delay = (base * (factor ** (attempt-1))) * (jitter ? (0.5 + Math.random()/2) : 1);
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
// 用法：只对幂等请求或携带 Idempotency-Key 的操作重试
```

**幂等协作**
- 读操作（GET/HEAD）默认可重试；写操作需：
  - **Idempotency-Key**（幂等窗口 + 请求体哈希绑定）。
  - 服务器对已处理同 key 请求返回相同响应。
- 取消与重试的交互：被 `abort` 的请求**不应**立即重试（通常是用户意图）。

---

# 22) 实时通信选型：WebSocket / SSE / WebRTC / WebTransport

| 能力点 | WebSocket | SSE (EventSource) | WebRTC DataChannel | WebTransport |
|---|---|---|---|---|
| 传输方向 | 双向 | 服务端→客户端 | 双向 | 双向 |
| 可靠性 | 可靠 | 可靠 | 可靠/不可靠可选 | 可靠/不可靠可选 |
| 穿透 | 一般 | 一般 | **ICE + STUN/TURN** | 依赖 H3 |
| 中间盒/代理友好 | 较差（对某些代理） | 较好（HTTP 长连接） | 需要 UDP/端口策略 | 需 H3 支持 |
| 典型场景 | IM/行情/推送 | 日志/增量流 | 音视频配套/低延时对战 | 低延时/UDP 友好新建站 |

**实践建议**
- 仅下行更新（监控/日志/行情）：**SSE** 简单稳健（自带自动重连），代理兼容好。
- 双向文本/二进制：**WebSocket**；注意心跳、压缩（`permessage-deflate`）与**背压**。
- 端到端低延迟 & P2P：**WebRTC**（TURN 成本与隐私合规考虑）。
- H3 可用、长连低延迟诉求：探索 **WebTransport**（仍在推广阶段，需回退方案）。

---

# 23) HTTP/3 实战：流/连接 ID、迁移与丢包体验、兼容与降级

**关键特性**
- **连接 ID (CID)**：与四元组解耦，支持网络切换/地址变更时**无缝迁移**。
- **流级可靠**：单流丢包不阻塞其他流（无传输层 HoLB）。

**体验差异**
- 移动网络切换、WIFI 抖动：H3 **更少重连卡顿**；高丢包时尾延迟更可控。
- 首次连接：H3 可能受 **中间盒**/老旧网络策略影响被阻断；建议 **Alt-Svc** 宣告 + **先 H2、后 H3** 的逐步启用。

**兼容/降级**
- 服务端通过 `Alt-Svc: h3=":443"; ma=86400` 暗示 H3 可用；客户端缓存后尝试走 H3。
- 观测 H3 失败占比（按 ASN/地区），为问题网段提供 **fallback 白名单**（强制 H2）。

---

# 24) 代理与网络栈：正/反向代理、负载均衡（L7/L4）、超时与粘性

**前端可感知影响**
- **多级超时**：浏览器超时、边缘（CDN）超时、反向代理超时、应用超时——**最小者生效**；需统一配置避免“早产超时”。
- **重试位置**：L7 代理可能对**幂等**请求自动重试；非幂等请求务必关闭代理重试或引入幂等键。
- **粘性会话**：通过 Cookie/一致性哈希保证**会话亲和**；跨区多活要小心回话跨区导致冷缓存、延迟上升。

**L4 vs L7**
- **L4（直通）**：性能高，少改写；对内容无感知。
- **L7（HTTP 代理）**：可做路由/压缩/缓存/鉴权/请求合并，但引入缓冲、头大小限制、潜在重写。

---

# 25) 边缘计算：Edge Functions / 边缘 SSR 的冷启动、亲和与缓存策略

**优势**
- **地理就近**：首字节更快，适合 HTML Shell/个性化首屏。
- 降低源站压力：边缘**预取 + 回源保护**。

**挑战**
- **冷启动**：无状态函数冷启几十~上百毫秒；通过**预热/常驻/计划任务**降低抖动。
- **状态与一致性**：边缘 KV/缓存与源站数据库一致性（TTL、tag 失效、事件驱动失效）。
- **依赖限制**：某些 Node API/二进制模块不可用（运行时沙箱）。

**策略**
- HTML **Streaming SSR**：先吐出骨架 + 关键 CSS，数据块就绪后增量推送。
- 缓存键：`(url, Vary: device/lang/ab)`；结合 **stale-while-revalidate** 与**Tag purge** 做细粒度失效。

---

# 26) SWR / ISR 与再验证：前后端协作与一致性风险

**术语**
- **SWR (stale-while-revalidate)**：先用旧的，后台再拉新。
- **ISR (Incremental Static Regeneration)**：静态页**按需再生**，到期/触发后在后台重算。

**协作要点**
- HTTP 侧：`Cache-Control: max-age=0, s-maxage=600, stale-while-revalidate=30` 搭配 CDN；浏览器端可本地 SWR。
- 应用侧（如 Next.js）：`revalidate: 600` 或 **tag**/**path** 失效；**按路由**控制再生。

**一致性风险**
- 多副本不一致窗口（读到旧数据）。缓解：关键交易页禁用 SWR；列表可 SWR，详情实时。
- 用户特定个性化内容应**避开共享缓存**；以 cookie/令牌区分或走私有缓存。

---

# 27) 请求合并与去重：连接复用之外的应用层治理

**目标**：减少**同 key**的并发请求、避开竞态“后到覆盖先到”。

**前端 in-flight 合并**
```js
const inflight = new Map(); // key: URL+body hash
async function dedupFetch(reqKey, doFetch) {
  if (inflight.has(reqKey)) return inflight.get(reqKey);
  const p = doFetch().finally(() => inflight.delete(reqKey));
  inflight.set(reqKey, p);
  return p;
}
```

**最后写赢与竞态**
- 更新 API 返回 **版本号/etag**；客户端提交时带上 `If-Match` 或版本字段，冲突返回 **409**，提示合并而非盲覆盖。

**服务端合并**
- 对同 key 的回源 MISS **折叠**；GraphQL/REST 层做 **批量加载**（DataLoader）。

---

# 28) GraphQL over HTTP：缓存、Persisted Queries、`@defer/@stream` 与治理

**缓存难点**
- 默认 **POST** + 复杂 `body` → 难以作为缓存键。解法：
  - **`GET` 查询**（把 query hash/变量放查询串，注意长度限制与隐私）。
  - **APQ / Persisted Query**：先上传 query→获得 hash；后续只发 hash + variables（GET 可缓存）。
  - Vary by `Authorization`/`Accept-Language` 等，防止污染。

**批量与增量**
- **`@defer/@stream`**：以 **`multipart/mixed`** 分块返回增量结果（需代理支持直通与不缓冲）。
- **batching**：多个查询打包一请求（Apollo batch link），减少队头与连接压力。

**治理**
- **复杂度限额**（depth/field cost）与 **速率限制**；  
- **字段级缓存**与**数据加载器**（N+1 抑制）；  
- 错误可观察性：operation name + trace id。

---

# 29) Range/分片上传：切片策略、并发/顺序、校验与“秒传”

**切片与并发**
- 典型 4–16MB 每片；**并发 3–8** 视网络而定；失败片重试（退避）。
- 两种协议：
  - **多段 PUT（S3 Multipart）**：每片有 `ETag`，最后 `CompleteMultipartUpload` 合并；整体 ETag 通常与片 ETag 有关。
  - **Content-Range**：`PUT /upload` + `Content-Range: bytes start-end/total`；服务端维护已收索引。

**校验与秒传**
- 片级 **MD5/CRC** 校验；完成后返回总文件哈希供客户端核对。
- **秒传**：先发 `hash+size` 探测，若服务器已有则直接返回“已存在”；否则返回**续传断点**。

**断点恢复**
- 记录上传 **uploadId/已完成片索引**；网络断开后从下一个未完成片接续。

---

# 30) 网络错误语义：DNS/TLS/CORS/SW/Fetch 错误的精准归因

**常见来源**
- **DNS 失败**：`ERR_NAME_NOT_RESOLVED`（域名不可解）；可观测 `PerformanceResourceTiming` 的 `domainLookupEnd - domainLookupStart`。
- **TLS 失败**：证书失效/主机名不匹配/链断裂 → `ERR_SSL_PROTOCOL_ERROR` 等。
- **CORS 拒绝**：请求发出但 **响应被浏览器拦截**，fetch 抛 `TypeError: Failed to fetch`（看不到状态码）；开发者工具 Network 面板显示 *blocked by CORS*。
- **Service Worker 拦截**：脚本异常或缓存 miss 未处理 → 返回 `opaque`/`error`；需在 SW `fetch` 事件里捕获并回退。
- **中间盒/代理**：注入或压制头部、缓冲导致 **假流式**、响应被替换。

**诊断清单**
- 打印 `error.name/message/cause`；抓 `FetchEvent`/`unhandledrejection`；在服务器开启 **NEL/Reporting API** 收集网络错误。
- 采集 **Resource Timing**：DNS/TCP/TLS/TTFB/下载阶段指标 + `transferSize/encodedBodySize`，定位段位。
- 区分**发起失败** vs **响应被拦截**：发起失败通常无任何 Network 记录；被拦截有记录但被标记阻止。

```js
window.addEventListener('unhandledrejection', e => {
  console.log('unhandled', e.reason?.name, e.reason?.message);
});

// NEL/Reporting 需服务端响应头：
// Report-To: {"group":"default","max_age":10886400,"endpoints":[{"url":"https://r.example.com/nel"}]}
// NEL: {"report_to":"default","max_age":10886400,"success_fraction":0,"failure_fraction":0.05}
```

> 归因的关键是**分层观测**：浏览器（Timing/错误事件）→ SW 日志 → 边缘/CDN 日志 → 源站日志，串起同一请求的 trace-id 才能闭环定位。
---
# 31) Service Worker 网络策略：Cache First / Network First / SWR / 离线兜底与版本迁移

**典型策略**
- **Cache First**（读多写少、CDN 命中高的静态资源）
  ```js
  self.addEventListener('fetch', e => {
    if (e.request.destination === 'image' || e.request.url.endsWith('.woff2')) {
      e.respondWith((async () => {
        const c = await caches.open('assets-v3');
        const hit = await c.match(e.request);
        if (hit) return hit;
        const res = await fetch(e.request, { cache: 'no-store' });
        // 仅缓存 200/opaque-with-cors 之类可复用的响应
        if (res.ok) c.put(e.request, res.clone());
        return res;
      })());
    }
  });
  ```
- **Network First**（动态接口、需要最新数据）
  ```js
  e.respondWith((async () => {
    try {
      const res = await fetch(e.request, { cache: 'no-store' });
      const c = await caches.open('api-v1'); c.put(e.request, res.clone());
      return res;
    } catch {
      const c = await caches.open('api-v1'); return (await c.match(e.request)) || new Response('offline', { status: 503 });
    }
  })());
  ```
- **Stale-While-Revalidate（SWR）**（列表/非强一致页面）  
  首先返回缓存，后台拉新更新缓存，下一次刷新命中新版本。
- **离线兜底**：统一 `offline.html` 与占位图；路由级兜底到“只读缓存视图”。

**版本迁移（跳票/清缓存）**
```js
self.addEventListener('install', e => { self.skipWaiting(); }); // 如需“强更”
self.addEventListener('activate', e => {
  e.waitUntil((async () => {
    const keep = new Set(['assets-v3','api-v1']);
    for (const k of await caches.keys()) if (!keep.has(k)) await caches.delete(k);
    await self.clients.claim();
  })());
});
```
> 关键：**避免缓存中毒**（只缓存可校验的 200），对接口缓存加上**短 TTL+ETag** 并设置回退策略。

---

# 32) 优雅降级：弱网/高延迟/丢包场景的超时、断点续传、骨架与渐进式渲染

**连接与请求级**
- **自适应超时/退避**：基于 RTT/LCP 历史设超时与重试次数；弱网下减少并发拉取数量。
- **分块/断点续传**：上传/下载按 4–16MB 切片，失败片重试；下载 `Range` + `If-Range`；上传记录 `uploadId`。
- **降清晰度/延迟加载**：图片 `srcset/sizes` + LQIP/BlurHash；视频首片短、码率自适应。

**渲染级**
- **骨架/占位**：SSR/Streaming 优先输出骨架与关键 CSS；客户端增量填充。
- **数据分级**：先出“列表壳 + 关键摘要”，次要字段延后加载。
- **离线提示**：`navigator.onLine` + SW 兜底，给用户明确重试入口。

**观测与开关**
- 动态 Feature Flag（弱网禁用昂贵动画/模块）；RUM 侧上报 **TTFB/LCP/错误率** 驱动降级策略。

---

# 33) 请求头/响应头精讲与组合拳

**请求侧**
- `Origin`：CORS 校验基准；**与 `Referer` 不同**（后者还含路径、可被策略裁剪）。
- `Accept` / `Accept-Language` / `Accept-Encoding`：触发**内容协商**；响应需 `Vary` 对应头。
- `Authorization`：带有 **Bearer/Basic 等**；影响缓存（通常不可共享缓存）。
- 条件请求：`If-None-Match`（ETag）、`If-Modified-Since`（Last-Modified）、`If-Match`（写入幂等等前提）

**响应侧**
- `ETag / Last-Modified` + `Cache-Control`（`max-age`/`s-maxage`/`stale-*`）
- `Date / Age / Expires`：**校准缓存年龄**；代理可能补写 `Age`。
- `Retry-After`：配合 429/503 指示重试时间（秒或 HTTP-date）。
- `Content-Type` + `charset`：避免 JSON 被当作 `text/plain`；字体/wasm 必须正确 `type` 以启用缓存与 CORP/CSP。
- `Timing-Allow-Origin`：允许跨域曝光 RT 指标（Resource Timing）。

**组合示例（共享缓存友好）**
```
Cache-Control: public, s-maxage=600, max-age=60, stale-while-revalidate=30
ETag: "v3-a1b2c3"
Vary: Accept-Encoding, Accept-Language
```

---

# 34) 重定向与安全：开放重定向、SameSite 与跨站跳转、凭证与缓存

**开放重定向防护**
- 后端跳转参数如 `?next=` 必须**白名单**或 **仅允许同站相对路径**：
  ```ts
  const next = new URL(req.query.next || '/', 'https://app.example.com');
  if (next.origin !== 'https://app.example.com') return res.status(400).end(); // 拒绝跳外站
  ```
- 对第三方回跳（OAuth）在**创建会话前**校验 `state` 与来源。

**跨站跳转与 Cookie**
- 跨站 3xx 链路会涉及 `SameSite` 规则：需要第三方 Cookie 的流程（支付/登录）→ `SameSite=None; Secure`。
- 注意 **浏览器缓存 301/308** 的持久性；对可回滚的链路先用 302 观察。

**缓存交互**
- 带凭证/受权限保护的 3xx 通常不可缓存；显式 `Cache-Control: no-store`。
- OIDC 授权回调页面应短时缓存或不缓存，避免泄露授权码。

---

# 35) 监控与采样：Resource/Navigation Timing、Long Tasks、`PerformanceObserver` 与采样策略

**关键 API**
```js
// 导航/资源
new PerformanceObserver((list)=>{ for (const e of list.getEntries()) console.log(e.name, e.startTime, e.duration); })
  .observe({ type: 'resource', buffered: true });
// Web Vitals（LCP/CLS/INP 可用 web-vitals 库）
new PerformanceObserver((list)=>{ /* longtask, paint, largest-contentful-paint */ })
  .observe({ type: 'longtask', buffered: true });
```
- **Navigation Timing**：DNS/TCP/TLS/TTFB/DOM 解析时间分解。
- **Resource Timing**：每个资源的状态与大小（`transferSize/encodedBodySize/decodedBodySize`）。
- **Long Tasks**：主线程 >50ms 的任务，归因到第三方脚本。

**采样策略**
- **分层采样**：按流量/地域/设备/渠道设不同采样率（如低端机更高采样）。
- **错误优先**：错误/超时 100% 上报，成功流量 1–5% 采样。
- **隐私**：不采个人数据；URL 只保留路径与必要参数；用**映射表**还原业务含义。

**链路追踪**
- 为每次导航/主请求生成 **trace-id**，回传至服务端/边缘，做到端到端合并分析。

---

# 36) WebSocket 握手与升级：HTTP → WS、子协议与帧

**握手**
```http
GET /chat HTTP/1.1
Host: ws.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: json, protobuf   # (可选)
```
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=  # = base64( SHA1(key + GUID) )
Sec-WebSocket-Protocol: json
```
- 子协议由双方协商（如 `graphql-ws`、`wamp`）。
- 完成 101 后切换到 WS 帧：**文本帧/二进制帧**、`ping/pong`、分片/连续帧。

**代理/HTTPS**
- `wss://` 走 TLS；某些代理不支持 `Upgrade`，需 **反向代理直通**或回退 SSE。

---

# 37) WebSocket 可靠性与性能：心跳、重连、退避、背压与压缩

**心跳与超时**
```js
let alive = true;
ws.on('pong', ()=> alive = true);
setInterval(()=>{ if (!alive) return ws.terminate(); alive = false; ws.ping(); }, 30000);
```
- 浏览器端：定时发送“空消息”或应用心跳，服务端超时关闭。

**重连策略**
- 指数退避 + 抖动；**区分网络断开 vs 服务器拒绝**（代码 1008/1011 等）。
- **消息队列**：断线期间把关键消息入队，重连后按序补发（带 `msgId` 去重）。

**背压**
- 监控 `ws.bufferedAmount` 或 Node 侧 `socket.bufferSize`，**超过阈值暂停生产**，或降采样。

**压缩**
- `permessage-deflate` 能显著省带宽，但会加重 CPU、引入 HOL（压缩上下文）；对**小消息/弱端**慎开或限制上下文。

**其他**
- 单连接承载多订阅通道（应用层 multiplex）；服务端按订阅粒度广播，避免全站风暴。

---

# 38) SSE vs HTTP Streaming：支持度、自动重连、缓存/代理兼容与落地

**SSE（Server-Sent Events）**
```js
const es = new EventSource('/stream?topic=log', { withCredentials: true });
es.onmessage = e => console.log('data:', e.data);
es.addEventListener('custom', e => ...); // event: custom
// 服务端：Content-Type: text/event-stream; Cache-Control: no-cache; X-Accel-Buffering: no;  (Nginx 关闭缓冲)
```
- **自动重连**（`retry:` 指令或浏览器默认 ~3s），支持 `Last-Event-ID`。
- 单向（下行），纯文本格式，代理兼容好。

**HTTP Streaming（Fetch/ReadableStream）**
- 任意 MIME，二进制友好；**需要代理不缓冲**且浏览器支持 Streams。
- 客户端更灵活（分块渲染/解析）。

**落地对比**
- **日志/行情/指标**：SSE 足够稳；  
- **大模型/增量文本**：二者皆可，Streams 更灵活；  
- **需要上行**：WS 或 WebTransport。

---

# 39) WebRTC DataChannel vs WebSocket：可靠性/有序性、NAT 穿透与复杂度

**DataChannel 特性**
- 基于 **SCTP over DTLS over UDP**；可配置  
  - 可靠/不可靠（`maxRetransmits/maxPacketLifeTime`）  
  - 有序/无序（`ordered: false`）
- NAT 穿透：**ICE + STUN/TURN**，可 P2P，延迟低。

**与 WS 对比**
- WS 依赖 TCP、易用、服务器生态成熟；**端点必须经服务器**，穿透难。
- DataChannel 复杂度更高（信令、候选收集/选路、TURN 成本），但能获得**端到端低延迟**与**传输灵活性**。

**选型**
- **对战/协作/白板/文件直传**：DataChannel（必要时设不可靠+无序）  
- **常规双向消息/业务推送**：WebSocket

**最小样例**
```js
const pc = new RTCPeerConnection();
const ch = pc.createDataChannel('chat', { ordered:false, maxRetransmits:0 });
ch.onmessage = e => console.log(e.data);
// 其余需通过信令服务器交换 SDP/ICE
```

---

# 40) STUN / TURN / ICE：候选、打洞流程与企业网络限制

**候选类型**
- `host`：本地网卡地址（局域网可直连）。  
- `srflx`（server-reflexive）：经 **STUN** 发现的公网映射地址。  
- `relay`：经 **TURN** 中继（流量经服务器，成本最高但最稳）。

**ICE 流程（简化）**
1) 双端收集候选（host/srflx/relay），通过**信令**交换。  
2) 建立**候选对**，按优先级连通性检查（STUN binding）。  
3) 选出最佳路径；网络变化时可**重新选路**（ICE restart）。

**要点与成本**
- 企业内网/防火墙常**封 UDP** 或只开放 80/443 → 需 **TURN over TLS（443）**。  
- TURN 流量计费，应按区域就近部署，启用**配额与认证**（长期凭据注册、短期凭据共享）。  
- **Keep-alive**：移动网络易 NAT 失活，需定时 STUN ping；谨慎设置频率以省电。

**实践建议**
- 提供至少 2 个 STUN、1~2 个 TURN（TCP+TLS）节点；  
- 启用 **Trickle ICE** 缩短建立时延；  
- 监控候选类型占比与失败率，识别“强防火墙”网络并回退 WS。
---
# 41) HTTP/3（QUIC）细节：0-RTT、连接 ID/迁移、路径验证、Alt-Svc 与中间盒

**0-RTT 重放风险**
- 0-RTT 仅适合**可重放/幂等**请求（GET、HEAD）；**禁止**用于创建订单、转账等写操作。  
- 服务端需绑定 **会话票据到客户端属性**（IP/ALPN/TLS 参数）并**限制时间窗**；对 0-RTT 请求可要求**重放检测/幂等键**。

**连接 ID / 迁移**
- CID 将连接与四元组解耦，**IP/网络切换**（蜂窝↔Wi-Fi）不中断；  
- 迁移时进行 **Path Validation**（PATH_CHALLENGE/RESPONSE），确认新路径可达再切换为主路径。

**初始包与 PMTU**
- H3 初始 UDP 包（Initial）要求**≥1200B** 以避免分片；若链路 PMTU 太小，可能触发黑洞（见题 45）。

**Alt-Svc 与兼容**
- 以 `Alt-Svc: h3=":443"; ma=86400` 告知客户端 H3 可用；客户端先用 H2，缓存后再试 H3，失败就**回退**。  
- 某些中间盒/防火墙阻断 UDP/QUIC：按 ASN/地区做**灰度启用**与失败回溯，保留 H2 兜底。

---

# 42) 拥塞控制与体验：CUBIC vs BBR；RUM 归因“网络 vs 渲染”

**算法画像**
- **CUBIC**：丢包触发拥塞窗口回退，吞吐随 RTT/丢包敏感；在**移动/高丢包**环境体验波动更大。  
- **BBR**：以**带宽/RTT**估计为目标，维持接近瓶颈速率，弱网更稳，但对**队列管理**与中间盒更敏感。

**前端侧 RUM 归因**
- 采集 **Navigation/Resource Timing**：`connect/TLS/TTFB/contentDownload` 分段可隔离**网络耗时**。  
- **Web Vitals**：  
  - **LCP**：若 LCP 受 `TTFB` 影响大 → 网络瓶颈；若主要耗在 render/layout → 渲染瓶颈。  
  - **INP**：交互延迟与网络弱相关，更多是主线程忙（JS/样式重排）。  
- 对比 **H2 vs H3** 的 LCP/TTFB 分布；弱网机型/地区启用 H3/BBR（服务端）能否改善 **p95/p99**。

**实践**
- 边缘/源站开启 **BBR**（可控范围内）；对静态大资源用 **H3**；  
- 指标面板分层：地区×运营商×网络类型（4G/5G/Wi-Fi）×协议（H2/H3）。

---

# 43) HoLB 与多路复用：弱网下“资源合并 vs 拆分”

**H2（TCP）**
- 单连接多路复用，但**传输层 HoLB**仍在：任一分片丢失，会阻塞该连接上**所有流**的后续数据。  
- 弱网下，**过度切分**（几十个小 JS）会放大丢包影响。

**H3（QUIC）**
- **流级独立**，单流丢包不拖累别的流；更鼓励**合理拆分**关键/次要资源并配合**优先级**。

**策略建议**
- **H2**：适度合并（路由/域内少数 bundle），避免“瀑布小碎流”；关键路径资源用 `preload` 抢占。  
- **H3**：倾向**拆分**（关键 CSS/早期 JS/首屏图独立流），利用 `priority/fetchpriority` 让关键资源更快达。  
- 双协议并存时，保持**中等粒度**：避免一个巨无霸 bundle，也避免几十个 5–10KB 的碎片。

---

# 44) DNS 现代化：DoH/DoT、EDNS、DNSSEC、Happy Eyeballs（v6/v4）

**加密解析**
- **DoT**（DNS over TLS，853 端口）/ **DoH**（DNS over HTTPS，443 端口）提升隐私与抗干扰能力；前端可感知为**首包更稳定**但可能略增时延。

**EDNS(0)**
- 允许更大 UDP 报文与扩展字段；配合 **ECS（EDNS Client Subnet）** 让 CDN 更精准定位（权衡隐私）。

**DNSSEC**
- 为记录提供签名验证，抗篡改；浏览器通常不直接验证，但递归解析器会→整体可靠性更高。

**Happy Eyeballs (HE)**
- 同时尝试 **AAAA + A**，优先更快可达的栈；降低“IPv6 存在但链路差”带来的首包超时。  
- 对跨域关键域名，配合 **`preconnect`** 与较短 **TTL**，在网络切换时更快收敛。

---

# 45) MTU / PMTUD 与分片：黑洞路由与前端规避

**概念**
- **MTU**：链路最大传输单元；**IP 分片**会增加重传成本。  
- **PMTUD**：通过 DF（Don’t Fragment）+ ICMP 反馈发现路径 MTU；若中间设备**丢 ICMP** → **黑洞**（大包一直失败）。

**影响**
- **TLS/H3** 初始握手包过大或 UDP 初始包 > 路径 MTU，会反复超时；H3 要求初始包 ≥1200B，链路过小将失败。  
- 大资源并非直接“IP 层分片”，但**过高的 MSS** 会让每个 TCP 段更易丢失。

**前端/交付侧策略**
- CDN/边缘启用 **PLPMTUD** 或谨慎的 PMTUD；  
- H3 限制初始 **payload**（保守编码、避免过多 token/头部膨胀），必要时降级 H2；  
- 大文件**应用层分块**（S3 分片/Range 下载），避免单流过长占用。

---

# 46) 代理与隧道：CONNECT、WS/SSE 支持、WAF/CDN 头透传与超时

**CONNECT 隧道**
- 浏览器与正向代理：`CONNECT host:443 HTTP/1.1` 建立 TLS 隧道。  
- **WebSocket**：通过反向代理需直通 `Upgrade/Connection` 头；H2 “扩展 CONNECT”（RFC 8441）并非处处可用，建议 **H1 直通**。

**SSE/Streaming 的缓冲**
- 许多代理默认**缓冲**响应 → 假流式；需显式关闭：  
  - Nginx：`proxy_buffering off;`，`X-Accel-Buffering: no`。  
  - CDN：关闭“Smart Routing Buffering”或启用“Stream Pass-through”。

**超时与保活**
- 为 WS/SSE 设置更长 **idle timeout** 与**心跳**；  
- WAF 限制 headers/方法时，给出**白名单**并记录被拒原因（避免“莫名 403”）。

**头部透传**
- 保留 `X-Forwarded-For/Proto`、`Early-Data`（0-RTT）、`Sec-WebSocket-Protocol`；避免代理乱改 `Connection`。

---

# 47) OAuth 2.1 / OIDC：授权码 + PKCE、`state`、回跳安全、前端存储与跨域

**推荐流（SPA/Native）**
- **Authorization Code + PKCE**（无客户端密钥）：  
  1) 客户端生成 `code_verifier`，发送 `code_challenge`。  
  2) 授权后回跳带 `code`，客户端用 `code_verifier` 换取 token。  
- **必须**校验 `state`（防 CSRF）与 `nonce`（OIDC id_token 重放防护）。

**回跳 URI 安全**
- 严格**白名单匹配**（完整 URL 或 scheme），拒绝通配；  
- 回跳时仅携带必要参数，避免泄露敏感信息到历史记录或第三方脚本。

**Token 存储与跨域**
- **Access Token**：建议放**内存**（页面刷新丢失即可刷新）；  
- **Refresh Token**：若需要在浏览器侧，推荐 **HttpOnly + SameSite=Lax/Strict** Cookie，并通过**同站路径**刷新。  
- 三方 Cookie 受限下（Safari/阻止跨站 Cookie），采用 **BFF 模式**（前端只与自家域通信，由 BFF 与 OP 交互）。

**静默续期与登出**
- 不再依赖隐式流/iframe 静默（受第三方 Cookie 限制）；改用**前端→BFF** 刷新或**Token Rotation**。  
- OIDC 前端登出配合 **RP-Initiated Logout**，清理本地状态与服务器会话。

---

# 48) 优先级信号：`fetchpriority`、`<link rel=preload>` 与 H2/H3 Priority

**层次**
- **HTML 层**：`fetchpriority="high|low|auto"`（img/script/link），影响浏览器内部调度。  
- **协议层**：**HTTP Priority**（`Priority`/`priority`）向服务器/代理表达紧急度与渐进性。  
- **显式获取**：`<link rel="preload">` 强制预取，但需 `as` 与 `crossorigin` 正确。

**组合范式**
```html
<link rel="preload" as="style" href="/app.css" />
<img src="/hero.jpg" fetchpriority="high" />
<script type="module" src="/entry.js"></script>
```
- **避免过度高优**：次要首屏图/次要 JS 不要标高优，防止挤占 CSS/关键 JS。  
- 对 H3，协议层优先级与多流特性互补：**细分流 + 正确优先级**优于大包。

---

# 49) 103 Early Hints + Preload：价值与坑点、与缓存/优先级/预连接协作

**价值**
- 在主响应准备期间就通知浏览器**开始拉取关键资源**（CSS/首屏图/模块入口），缩短 **LCP**。  

**实现要点**
- 边缘/源站必须**直通** 103，不缓冲；CDN/代理要支持 *Early Hints*。  
- 重复发送风险：主响应也下发 `<link rel=preload>`；浏览器会去重，但要确保 **`as`/`crossorigin` 一致**，否则可能重复下载。  
- 与 **preconnect** 协作：提前建链 + 预取关键资源更稳。

**示例**
```
HTTP/1.1 103 Early Hints
Link: </app.css>; rel=preload; as=style
Link: </entry.js>; rel=modulepreload; as=script
```

---

# 50) Reporting API / NEL：网络与策略违规上报、采样与闭环

**服务端响应头**
```
Report-To: {"group":"default","max_age":10886400,"endpoints":[{"url":"https://r.example.com/reports"}]}
NEL: {"report_to":"default","max_age":10886400,"success_fraction":0,"failure_fraction":0.05}
Content-Security-Policy: ...; report-to=default; report-uri https://r.example.com/csp
```
- **NEL** 收集网络错误（DNS/TCP/TLS、协议失败）；  
- **Reporting API** 收集 **CSP 违规/崩溃/弃用警告** 等。

**采样与隐私**
- 失败高采样（5–10%），成功 0%；**不含 PII**，URL 做脱敏（去查询私参），遵循数据保留周期。  
- 对应生成 **request_id/trace_id**，在前端埋点/后端日志/边缘日志间贯通。

**闭环**
- 将 NEL/CSP 违规接入告警通道（按 ASN/地区聚合）：  
  - 某 ISP UDP 丢包异常 → 降级 **H3→H2**；  
  - 某广告脚本 CSP 违规激增 → 动态**阻断**或调整白名单。  
- 联动 **RUM/WAF/CDN** 看板，实现 **发现 → 定位 → 策略变更 → 回看** 的可观测闭环。
