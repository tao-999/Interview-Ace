# React Native

## 1) 新架构总览：Fabric + TurboModules + JSI；告别 Bridge 的本质

**角色与边界**
- **JSI（JavaScript Interface）**：C++ 层的通用 JS 引擎适配层（面向虚拟机而不是“消息总线”）。Native 可将 **HostObject/HostFunction** 暴露为 JS 值，JS 调 Native/Native 调 JS 都是**函数调用语义**，无需 JSON 序列化/拷贝。
- **TurboModules**：以 **Codegen**（从 TypeScript/Flow spec 生成）为核心的模块系统。导出接口 → 生成 C++/ObjC/Java/Kotlin 胶水 → 通过 JSI 直接调用。支持 **sync/async/promise**，类型对齐、性能稳定。
- **Fabric 渲染器**：新的跨平台 UI 渲染管线。JS 侧构建 **Shadow Tree**（跨平台 C++ 结构，布局用 **Yoga**），**commit** 产生 **Mounting 指令**，然后在 **UI/Mounting** 线程原子性地应用到平台原生视图。支持**并发/增量**、**原子提交**、**优先级**。

**旧 Bridge 的问题**
- 旧模型是**异步、按帧批量**的 **Message Queue**（JS ↔ native 都是 JSON payload）：高频小调用被批量化但仍有**序列化/复制/跨线程**成本；不支持同步调用；“大对象过桥”易卡顿。
- 调试/性能：过桥次数与体积成为核心瓶颈（动画、手势、列表测量尤甚）。

**新架构的本质收益**
- **调用是调用**（JSI 的函数调用语义）→ 少序列化、少拷贝、可**零拷贝共享内存**（如绑定 ArrayBuffer）。
- **原子渲染提交**（Fabric commit）→ 视图层更新更一致，避免“半应用”闪烁；更好地支持并发/优先级。
- **类型安全与约束**（TurboModules Codegen）→ 在编译期发现不一致；多语言胶水统一生成。

**代价与迁移**
- 需要为模块/组件**写 spec 并过 codegen**；老模块需迁移（或启用兼容层）。
- 自定义组件要理解 **Props/Events/Commands** 的 codegen 与 **Mounting/测量** 生命周期。
- 对引擎（Hermes）和构建链（buck/gradle/pods）版本耦合更强，CI/CD 与缓存要升级。


---

## 2) Hermes 引擎：启动、内存、字节码（hermesc）与调试取舍

**性能与体积**
- **启动时延（TTI）**：Hermes 支持 **字节码预编译**（`hermesc`），包内直接携带字节码（.hbc），避免运行时解析/编译，**冷启动更快**。
- **内存**：紧凑的 GC（分代 + 小对象优化）与低峰值 RSS，移动端更友好。一般应用可见 **堆和常驻内存下降**（与 JSC 相比）。
- **包体**：Hermes 本体会增加二进制体积，但 JS bundle 变小（字节码），总体需视项目而定。

**调试与 Profiling**
- 具备 Chrome DevTools 协议桥接、**Sampling Profiler**、Heap Snapshot。与 Flipper 集成良好（🡒 Network/Layouts/Performance）。
- 与 JSC 的差异：早期 ES 特性落后，如今主流 ES/Intl 能力已齐（仍以目标 RN 版本 release note 为准）；Hermes **错误堆栈**与**源映射**支持更完善。

**实践要点**
- 生产建议启用 **precompile**：Metro 产出 Hermes bytecode（`--hermes`），禁用 `inlineRequires` 需要权衡（Hermes 已有 lazy 解析优化）。
- **崩溃排查**：启用 **源映射上传**（Sentry/Firebase），Hermes stack 反解；注意字节码与 sourcemap 版本一致。
- **国际化**：若用 Intl/正则特性，确认 Hermes 版本支持（或使用 polyfill 与 `@formatjs/intl-*`）。


---

## 3) RN 渲染流水线：线程协作、Shadow Tree 与批量更新

**线程模型（概念统一，平台具体实现略有差异）**
```
JS Thread (逻辑/状态) 
   └─(reconciler)→ Fabric Scheduler (C++)
         └─ 构建/变更 Shadow Tree（Yoga 布局）
              └─ Commit (原子)
                   └─ Mounting(UI) Thread → 平台视图树 (UIView/View)
Render/Compositor Thread（平台负责合成）
```
- **JS Thread**：React 协调（reconcile）生成元素变更（Fiber），交给 **Fabric Scheduler**。
- **Shadow Tree**：纯 C++ 结构，跨平台；**Yoga** 计算布局（测量/百分比/最小最大约束）。
- **Commit**：对 Shadow Tree 的一次不可分批改，生成 **Mounting 指令**（create/update/delete/insert）。
- **Mounting/UI 线程**：**原子地**应用这批指令，避免中途可见的中间态。
- **批处理与优先级**：setState/更新在同一帧聚合；支持 **interruptible**（打断）与优先级（如手势/动画更高）。

**常见性能坑**
- 在 JS 侧频繁 setState 但**缺少批量**（useTransition/unstable_batchedUpdates）会放大开销。
- 渲染重排来自 **测量回调滥用**（见 Yoga 一节）或在 commit 期间做同步工作。


---

## 4) Yoga 布局深入：`flex-basis`、约束与测量

**`flex-basis` vs `width/height`**
- `flex-basis` 是 **Flex 布局主轴的初始尺寸**，它优先于 `width/height`（当主轴为横向时基于 `width` 概念）。若 `flex-basis:auto`，则回退到 `width/height` 或内容测量。
- 计算顺序：收集基准尺寸 → 分配剩余空间（grow/shrink）→ 应用 **min/max** 进行 **clamp**。

**`min/max` 约束**
- Yoga 在 **最终步骤**对结果做 `min/max` 裁剪。错误理解会导致“明明 grow 了却没变大”。要根据 clamp 后尺寸再做子布局。

**文本基线/行内对齐**
- RN 文本通过 **baseline** 对齐时，需确保字体度量可用；跨平台行高差异（iOS ascender/descender 与 Android baseline 计算不同）导致轻微错位，建议在设计上使用统一行高策略。

**百分比尺寸与测量回调**
- 百分比相对于**父容器的 content box**。父尺寸未定时会引发多次布局。
- 自定义 **测量回调（measure）**（Fabric 组件）必须**纯函数**、无副作用、**幂等**，且时间复杂度要低；否则在布局级联中造成抖动与掉帧。

**实践清单**
- 优先用 `flex-basis` 调主轴尺寸，用 `min/max` 做护栏。
- 避免“测量依赖外部异步状态”的组件；必要时缓存尺寸（memoize）并提供 `getItemLayout` 给列表。


---

## 5) 列表性能：FlatList/SectionList/VirtualizedList/FlashList

**抽象与实现**
- **VirtualizedList**：最底层虚拟化容器；FlatList/SectionList 基于它封装。
- **FlatList**：常用；支持 windowing、回收与分块渲染。
- **SectionList**：分组头部/索引，适合分区数据。
- **FlashList（Shopify）**：重写虚拟化策略，强调**测量可预测**与回收；在重负载场景下更稳定流畅。

**关键参数与调优**
- `keyExtractor`：**稳定唯一 key**，避免无谓重渲染。
- `getItemLayout`：提供 **固定或可预知高度** → 跳转/滚动更准且减少测量/抖动（尤其长列表、首屏更快）。
- `windowSize`/`maxToRenderPerBatch`/`updateCellsBatchingPeriod`：窗口大小与批渲染节奏（权衡内存与滚动空窗）。
- `removeClippedSubviews`：滚动时裁剪不可见子树，Android 效果明显；注意有 **overflow**/动画/绝对定位 子元素时的裁剪 bug。
- `initialNumToRender`/`ListHeaderComponent`：首屏渲染控制；避免巨大 Header 阻塞可视区域，必要时拆分懒加载。
- `memo`/`PureComponent`/`React.memo` + **稳定的 props 引用**（避免箭头函数 inline 变化）。
- 图片/富文本项：使用缩略图占位 + **渐进加载**；避免在 `renderItem` 中同步 decode。

**FlashList 额外要点**
- 建议提供 `estimatedItemSize`；配合 `getItemType` 与稳定 key，滚动预测更准。
- 滑动流畅优先级时，减少跨线程测量/布局（见 Yoga）。


---

## 6) 动画体系：Reanimated vs Animated vs LayoutAnimation；与手势配合

**Animated（旧体系）**
- JS 驱动（每帧 setValue）或 **native driver**（有限属性集：`transform/opacity` 等）；JS 驱动易受 JS 帧阻塞。
- 复杂交互（手势驱动、物理动画）在 JS 驱动下常出现掉帧。

**Reanimated 2/3**
- **Worklets**：JS 函数经 Babel/JSI 编译，在 **UI 线程**运行（无需过桥）。支持手势同步、物理动画、派生值。
- 与 **react-native-gesture-handler** 深度集成：手势事件在 UI 线程处理，动画与手势闭环（`useAnimatedGestureHandler` / `useSharedValue` / `useAnimatedStyle` / `withTiming/spring`）。
- 可用 `runOnJS` 安全回落至 JS 线程（谨慎使用，避免频繁跨线程）。

**LayoutAnimation**
- 平台级布局变更过渡（插入/删除/位置变化），简单场景易用；控制粒度有限，复杂交互不如 Reanimated 灵活。

**选型建议**
- **手势联动/高帧动画**（卡片跟随、可拖拽底部弹层）：**Reanimated**。
- **简单显隐/过渡**：LayoutAnimation 或 Animated（native driver）即可。
- **复杂序列/可中断**：Reanimated（UI 侧状态机）更稳。

**典型片段（Reanimated + Gesture）**
```ts
const tx = useSharedValue(0);

const pan = Gesture.Pan()
  .onChange(e => { tx.value += e.changeX; }) // UI 线程
  .onEnd(() => { tx.value = withSpring(0); });

const style = useAnimatedStyle(() => ({ transform: [{ translateX: tx.value }] }));

return <GestureDetector gesture={pan}>
  <Animated.View style={style} />
</GestureDetector>;
```


---

## 7) 手势系统：竞争/并发、滚动冲突与优先级

**核心概念（RNGH v2+ API）**
- **`simultaneousHandlers`**：允许两个手势**并行识别**（不互斥），如缩放与旋转。
- **`waitFor`（require failure）**：当前手势需**等待**另一手势**失败**后才能激活，解决“滑动 vs 点击/长按”优先级。
- **状态机**：`BEGAN → (ACTIVE) → END/CANCEL/FAIL`，中间可能被其他手势抢占。

**滚动冲突的常见解法**
- 嵌套 `Pan` 与 `ScrollView`：对横向 `Pan` 设置 `waitFor={scrollGesture}`，或在回调中根据角度/阈值决定 `enabled`。
- `Touchable*` 与 `Pan`：点击类手势常需要 `waitFor(pan)`，避免滑动中触发点击。
- 与 `TextInput`：键盘/选择器弹出期间需适当 `enabled=false`。

**工程策略**
- 明确“主导手势”，其余对其 **waitFor**；需要并行的用 **simultaneousHandlers**。
- 复杂场景将“优先级关系”抽象为常量表，统一管理；为每条关系写可视化调试开关（打印状态流转）。


---

## 8) 导航选型：React Navigation / React Native Navigation / Expo Router；链接与恢复

**架构差异**
- **React Navigation**（JS 驱动，栈/标签基于 React 组件树；手势/过渡依赖 Reanimated + RNGH）：灵活、生态丰富；大规模 App 常用。
- **React Native Navigation（Wix）**（**原生导航栈**，屏幕与容器由原生托管）：切换/过渡更原生，内存管理与生命周期更贴近平台；对 React 模型的统一性较弱。
- **Expo Router**：基于 React Navigation 的**文件路由**层，深度集成 Expo（路由即文件系统）；更规范的约定式导航。

**Deep Link / Universal/App Links**
- **配置**：在 RN（React Navigation）中配置 `linking`（`prefixes`, `config` 路由表）；iOS 配置 `Associated Domains`，Android 配 `intent-filters`。
- **场景**：从推送、Web、邮件跳入指定页；统一 schema（`myapp://...`）与 Universal Links（`https://`）。
- **状态恢复**：应用冷启动用 `getInitialURL()`，热启动用 `Linking.addEventListener`；React Navigation 提供 **state persistence**（序列化路由树到 storage）。

**选择建议**
- 需要 **Native 级导航表现**（复杂转场/大容器/现有原生栈接入）→ **RNN**。
- 需要 **跨平台统一、强可定制** → **React Navigation**。
- Expo 体系 + 约定式路由 → **Expo Router**。


---

## 9) 图片与资源：缓存、预取、占位、矢量资产

**缓存/加载**
- RN 内置 `Image` 的缓存行为**平台相关**，策略不可完全配置；可用 **`Image.prefetch(uri)`** 与 `onLoad`/`onProgress`（部分平台/库支持）做体验优化。
- 第三方：**FastImage**（Android 用 OkHttp/磁盘缓存策略、iOS 用 SDWebImage），支持 **优先级、缓存控制、进度**；或 **expo-image**（Expo）。
- **占位/渐进**：先渲染低清缩略图（blurhash/base64），完成后渐显高清；避免 `resizeMode: 'contain'` 导致的大尺寸 GPU 纹理抖动。

**尺寸与解码**
- 服务器下发**多规格**（`srcset` 等价策略），在 JS 侧按 DPR/容器宽度选择；避免把超大原图下发给列表项。
- **内存峰值** = 宽×高×4（RGBA）≈ 解码后的像素内存，注意长图、GIF（逐帧解码）导致的 OOM。

**矢量资产**
- **Android Vector Drawable（.xml）**：适合图标；可染色，可缩放；复杂路径过多会影响绘制。
- **iOS PDF 矢量资产**：Xcode Asset Catalog 中多分辨率自动导出；对细节/文本锐利度好。
- 跨端一致性：SVG（`react-native-svg`）做动态矢量 UI；IconFont 在可访问性与动态颜色上不如 SVG 灵活。

**示例：预取与占位**
```ts
useEffect(() => { Image.prefetch(uri); }, [uri]);

return (
  <View>
    {loaded ? null : <Image source={thumb} blurRadius={8} style={s.img}/>}
    <Image
      source={{ uri, cache: 'force-cache' }} // FastImage 可用优先级/缓存模式
      onLoadEnd={() => setLoaded(true)}
      style={s.img}
    />
  </View>
);
```


---

## 10) 网络层工程化：fetch/axios、上传进度、断点续传、SSL Pinning、H2、代理调试

**fetch vs axios（在 RN 环境）**
- RN 的 `fetch`/`XMLHttpRequest` 由原生模块实现（iOS CFNetwork，Android OkHttp）。  
- **上传/下载进度**：原生 `fetch` **不提供进度回调**；需用 **`XMLHttpRequest`** 或第三方（`react-native-blob-util`）/自写原生模块。
- axios 语法友好、拦截器丰富，但其进度也依赖底层 XHR 支持；在 RN 中常用 **自定义适配器**。

**上传进度（XHR）**
```ts
const xhr = new XMLHttpRequest();
xhr.open('POST', url);
xhr.upload.onprogress = (e) => {
  if (e.lengthComputable) setProgress(e.loaded / e.total);
};
xhr.onload = () => resolve(xhr.response);
xhr.onerror = reject;
xhr.send(formData); // 包含文件流
```

**断点续传**
- 客户端：记录已上传区间（Range/分块编号），失败后续传；服务端配合 **分块合并** 或 **S3 Multipart**。
- Android/OkHttp 原生可控；iOS 侧建议直连对象存储（S3 SDK）或自建分块接口。

**SSL Pinning**
- 需原生支持（OkHttp 拦截器/`NSURLSession` 配置）或库（`react-native-ssl-pinning`）。  
- 策略：**证书指纹**（SHA-256 公钥）优于整证书；注意证书轮换与灰度策略。

**HTTP/2 与代理调试**
- Android（OkHttp）/iOS（NSURLSession）普遍支持 H2；长连接/多路复用对移动网络更友好。  
- **调试代理**：Charles/Proxyman/mitmproxy；安装**受信任根证书**；对 **SSL Pinning 的构建变体**关闭或引入调试白名单。  
- **Flipper**：启用 Network 插件查看请求/响应；结合 Hermes Profiler 定位大包序列化/JSON 解析热点。

**容错与重试**
- 幂等请求（GET/PUT）可**指数退避**重试；POST 需幂等键或服务端支持去重。  
- 弱网与前后台切换：结合 **NetInfo** 与 **AppState** 做**暂停/恢复**；超时/取消用 **AbortController** 串好请求生命周期。

```ts
const ctrl = new AbortController();
const id = setTimeout(() => ctrl.abort(), 15_000);
try { await fetch(url, { signal: ctrl.signal }); } finally { clearTimeout(id); }
```

---
## 11) 本地存储：AsyncStorage / MMKV / SQLite/Realm/WatermelonDB；加密与离线同步

**选型要点**
- **AsyncStorage**（基线 KV）：JS 侧 Promise API，容量与性能一般；Android 旧实现基于文件，社区版（`@react-native-async-storage/async-storage`）已优化但**不适合高频/大数据**。
- **MMKV**（KV，C++/mmap）：腾讯开源，内存映射、**极快**读写；支持多实例/加密；适合**配置、会话、轻量 cache**。不支持复杂查询/事务。
- **SQLite**（关系/事务）：结构化查询、索引、事务；适合**列表缓存、离线查找、分页**。常用库 `react-native-sqlite-storage`、`react-native-quick-sqlite`（JSI，快）。
- **Realm**（对象数据库）：零拷贝对象图、变更通知、迁移；JS API 简单；包体较大；**写入线程模型**需遵守。
- **WatermelonDB**（SQLite + 同步协议）：针对**大列表 + 同步**，延迟加载、工作线程写；要搭配服务器同步端。

**常见分层**
- **KV（MMKV）**：登录态、Feature Flag、最近使用。  
- **对象/关系（SQLite/Realm）**：业务数据缓存、增量同步、索引搜索。  
- **临时文件**：大媒体/导出（`react-native-fs`）。

**MMKV 片段**
```ts
import {MMKV} from 'react-native-mmkv';

export const kv = new MMKV({ id: 'app', encryptionKey: __DEV__ ? undefined : 'your-32-chars-key' });

export const session = {
  get token() { return kv.getString('token') ?? ''; },
  set token(v: string) { kv.set('token', v); },
};
```

**SQLite 片段（Quick SQLite + 事务）**
```ts
import {open} from 'react-native-quick-sqlite';
const db = open({ name: 'app.db' });

db.execute('CREATE TABLE IF NOT EXISTS post(id TEXT PRIMARY KEY, title TEXT, ts INTEGER)');
db.transaction(tx => {
  tx.execute('INSERT OR REPLACE INTO post(id,title,ts) VALUES (?,?,?)', [id, title, Date.now()]);
});
```

**离线同步策略**
- **变更日志**：本地记录 delta（upsert/delete + 版本号/时间戳），后台在线时**批量上行**；服务端返回 authoritative 版本与冲突解决（last-write-wins 或域特定合并）。
- **幂等性**：使用 **clientId + opId** 去重；失败重放。
- **安全与加密**：  
  - KV：MMKV 自带 AES；密钥存于 Keychain/Keystore。  
  - DB：`sqlcipher`（SQLite 加密版）或 Realm 自带加密；**注意密钥轮换与热升级**。  
- **备份/清理**：区分 iOS `NSDocumentDirectory`（备份）与 `Caches`（不备份）；遵循用户登出时**安全擦除**。

---

## 12) 原生模块（TurboModule）：TS 声明 + Codegen；调用边界；Swift/Kotlin 注意

**最小闭环**
1) **定义 TS 接口（spec）**  
```ts
// NativeFoo.ts
export interface Spec {
  ping(msg: string): Promise<string>;
  add(a: number, b: number): number; // sync 可选，但注意阻塞
}
export default TurboModuleRegistry.getEnforcing<Spec>('Foo');
```
2) **注册 codegen**：配置 `react-native-codegen` 指向 TS/Flow spec；生成 C++/ObjC/Kotlin 桥代码。
3) **实现 Native**：
- **Android (Kotlin)**
```kotlin
class FooModule(context: ReactApplicationContext): NativeFooSpec(context) {
  override fun getName() = "Foo"
  override fun add(a: Double, b: Double) = a + b
  override fun ping(msg: String, promise: Promise) {
    promise.resolve("pong: $msg")
  }
}
```
- **iOS (Swift)**（使用 Swift 需要 bridging；或用 ObjC）
```swift
@objc(Foo)
class Foo: NSObject, NativeFooSpec {
  func add(_ a: Double, b: Double) -> Double { a + b }
  func ping(_ msg: String, resolve: RCTPromiseResolveBlock, reject: RCTPromiseRejectBlock) {
    resolve("pong: \(msg)")
  }
  static func requiresMainQueueSetup() -> Bool { false }
}
```

**Promise / Callback / Sync 边界**
- **Promise**：首选；不会阻塞 JS 线程；可串接取消（`AbortController`）与超时。
- **Callback**：旧式；尽量避免多次回调与竞态。
- **Sync**：只有**计算量极小**、**读取内存已有值**时使用（如设备常量）；**严禁**做 IO/锁等待，否则卡死 JS。

**实战注意**
- **线程**：在原生侧把耗时工作放到后台线程，回调切回 **主/UI 线程** 或 **JS 调度器**（`CallInvoker`）。
- **类型对齐**：Codegen 强约束（可空、数组、记录）；与 TS 保持一致。
- **错误约定**：Promise 用统一错误码/域；iOS `NSError`，Android `ReactNoCrashSoftExceptions` 记录。

---

## 13) 原生 UI 组件（Fabric Component）：Props/Events/Commands；测量/布局

**概念**
- **Props**：由 codegen 生成 C++ Props 结构，JS 侧用 `<MyView propA={...} />`。
- **Events**：`onChange` 等回 JS 的事件，走 JSI `EventEmitter`。
- **Commands**：由 JS 主动调用原生实例方法（如 `focus()`）。

**流程（概览）**
1) **定义组件 spec（TS/Schema）** → `codegen` 生成 C++/ObjC/Kotlin 模板。
2) **Native 侧实现**：View + ShadowNode/Descriptor（Fabric）；支持测量（Yoga measure）与 Mounting。
3) **JS 侧**：`requireNativeComponent` 或自动生成的组件包装。

**TS Spec（摘）**
```ts
import type {HostComponent, ViewProps} from 'react-native';
type Event = { value: number };

export interface NativeProps extends ViewProps {
  progress?: number;
  onChange?: (e: Event) => void;
}
export default (codegenNativeComponent<NativeProps>('MyProgressView') as HostComponent<NativeProps>);
```

**测量/布局**
- 实现 **`measure`** 时需纯函数，依据传入 props/样式与文本/内容，**不可访问 JS 侧状态**。
- 对于需要内在大小（如文本），提供 **`getNativeMeasurement`** 或在 ShadowNode 里缓存，避免多次昂贵测量。

**Mounting 生命周期**
- **create → update props → attach/detach → delete**。更新阶段生成 **Mounting 指令**，原子应用，避免中间态闪烁。
- **Commands** 通过 `Ref` 下发到具体原生实例（不要在 Commands 内做长时任务）。

**事件**
- 保持**事件去抖/节流**（如滚动/进度）在**原生侧**处理，减少 JS 压力。
- 事件 payload 稳定、有版本；避免传递大对象/bitmap。

---

## 14) JSI 能力：HostObject/HostFunction、零拷贝、生命周期与并发安全

**核心**
- **HostFunction**：把原生函数暴露给 JS（`jsi::Function`）。  
- **HostObject**：自定义对象，属性访问/方法调用由原生接管（`get/set`）。  
- **ArrayBuffer 零拷贝**：可把原生内存包装为 `ArrayBuffer`/`TypedArray`，避免跨桥复制（注意所有权）。

**ArrayBuffer 绑定示意（C++）**
```cpp
jsi::Object makeArrayBuffer(jsi::Runtime& rt, std::shared_ptr<std::vector<uint8_t>> buf) {
  auto len = buf->size();
  auto data = buf->data();
  auto ab = jsi::ArrayBuffer(rt, (uint8_t*)data, len); // *实现方案依赖 RN 版本*
  // 需要自定义生命周期：当 JS GC 回收时，确保 buf 存活至最后一个引用释放
  return ab;
}
```
> 实际上在不同 RN/Hermes 版本，`ArrayBuffer` 包装 API 有差异；常见做法是实现一个 **HostObject** 持有 `shared_ptr`，`get()` 时返回 `ArrayBuffer`，或提供拷贝/映射选项。

**线程与安全**
- **JSI API 只能在持有对应 Runtime 所属线程/锁时调用**；跨线程访问需排队到 JS 调度器（`CallInvoker`）或使用**同步原语**保护共享内存。
- **生命周期**：HostObject 中不要持有**短生命周期的 JS 值**；如需回调，保存 `jsi::Function` 时要保证在正确线程调用，并捕获异常。
- **异常**：抛 `jsi::JSError` 让 JS 能拿到堆栈；原生异常不可穿越 JSI 边界。

**何时用 JSI**
- 高频、低延迟**计算/编解码**（如 Base64、加解密）；  
- 与原生库共享内存（音视频帧、图像缓冲）；  
- 自定义 JS 宿主能力（PRNG、持久句柄）。  
优先考虑 **安全与维护成本**，能用 TurboModule/异步 API 解决时不必 JSI 化。

---

## 15) 构建与发布：iOS / Android；多 ABI / 多渠道

**iOS**
- **CocoaPods**：锁定 `Podfile.lock`；Hermes/Fabric 版本与 RN 版本耦合，清理 DerivedData 与 `pod deintegrate` 常见。
- **签名**：`CODE_SIGN_STYLE=Automatic`（小团队）或手动证书/Provision；CI 建议使用 **Match/fastlane** 管理。
- **Bitcode**：已被 Apple 废弃（Xcode 14 起），确保项目关闭相关选项。
- **符号表**：上传 **dSYM**（崩溃反解）与 **BCSymbolMaps**（Bitcode 时代）；现代仅 dSYM 即可。
- **切片/切换**：`EXCLUDED_ARCHS` 控制模拟器/真机；按需引入 `use_frameworks! :linkage => :static` 兼容 Swift。

**Android**
- **Gradle KTS** 推荐；开启 **R8** 混淆与资源压缩；AAB（App Bundle）发布。
- **ABI 分包**：`ndk { abiFilters 'armeabi-v7a','arm64-v8a','x86','x86_64' }`；AAB 会按 ABI 分发。
- **ProGuard/R8 规则**：保留 JSI/TurboModule/Reflection 需要的类与注解；Reanimated/Flipper/Hermes 有官方 keep 规则。
- **符号表**：上传 `mapping.txt`；`ndk-stack`/`addr2line` 解析 native 崩溃。

**多渠道/多变体**
- Android `productFlavors` + `buildTypes`；iOS `Schemes` + `Configurations`。产出多包：`dev/stage/prod`、`china/global` 等。

---

## 16) 多环境配置：Schemes/Configurations、productFlavors、配置注入

**iOS**
- 建立 `Debug-Staging/Release-Staging` 等 **Configuration**，配套 **.xcconfig**；  
- `Scheme` 绑定某个 configuration，便于一键切换；  
- 注入变量：  
  - **编译时**：`GCC_PREPROCESSOR_DEFINITIONS` 或 Swift `Active Compilation Conditions`；  
  - **运行时**：`Info.plist` 自定义键 + `Bundle.main.object(forInfoDictionaryKey:)` 读取。

**Android**
- `productFlavors { dev { dimension "env"; applicationIdSuffix ".dev" } }`；  
- `buildConfigField("String","API_BASE","\"https://api.dev\"")` 暴露给 Java/Kotlin；  
- `resValue("string","app_name","MyApp Dev")` 定制资源。

**跨端注入**
- `react-native-config`：读取 `.env.dev`/`.env.prod` 生成原生常量，再暴露 JS；  
- 更严格方案：在 CI 用 **模板 + secrets** 生成 `.json`，原生读取并经 TurboModule 下发，避免把敏感信息放在 JS bundle。

**注意**
- **不要把密钥放 JS**；使用 Keychain/Keystore 或远程配置；  
- 区分 **构建时** 与 **运行时** 配置，避免 OTA 引发不一致。

---

## 17) OTA：CodePush / Expo Updates；灰度、回滚、二进制兼容

**约束**
- App Store/Play 政策：OTA 不得改变应用核心特性/权限，不得引入**新的原生能力**（需要重新上架）。  
- **二进制兼容**：JS 代码需与当前安装的原生二进制**版本匹配**；否则崩溃/行为不一致。

**CodePush**
- 以 “**Deployment**（Staging/Production）” 管理；每个部署有 `rollout %`、`min/max appVersion` 门槛。  
- 支持 **立即/下次启动应用**、**强制更新**、**回滚**。  
- 资源与 bundle 一起下发；注意 **bundle/hash** 与 **sourcemap** 管理。

**Expo Updates / EAS Update**
- 通过 **Runtime Version** 校验**原生兼容性**（推荐不要用单纯 semver）；  
- 支持 **多渠道**、**分支**、**Rollout**；资产托管在 Expo CDN 或自定义；  
- 与 `expo-asset`/`expo-image` 协作，离线包策略更简洁。

**灰度/回滚/元数据**
- 灰度：初始 1–5%，监控崩溃率/关键指标（启动、错误率）→ 扩大。  
- 回滚：保留 **上一版本**包的可用指针；问题出现立刻指向回退，并标记故障版本 blacklist。  
- 元数据：记录 **git sha、构建号、RN/Hermes 版本**；上报随崩溃与事件以便快速定位。

---

## 18) 性能与监控：Flipper、Hermes Profiling、内存与掉帧监控、上报

**工具**
- **Flipper**：Network、Layout、React DevTools、Hermes Debugger、Perf Monitor；可写自定义插件（如业务指标可视化）。
- **Hermes Sampling Profiler**：采样调用栈 + 火焰图，定位 JS 热点；配合 `console.profile()` 边界。
- **Systrace/Trace**：Android `traceview`/`perfetto` 分析 UI/GPU/IO。
- **Xcode Instruments**：Time Profiler、Core Animation（掉帧）、Leaks/Allocations。

**监控指标**
- 启动：Cold start TTI、bundle 加载时间、首屏渲染。  
- 交互：**JS 帧率**、**UI 帧率**、动画卡顿；  
- 资源：**JS Heap**、Native heap/RSS、图片缓存；  
- 稳定性：JS 未捕获异常、Native 崩溃率；  
- 网络：失败率、P50/95/99 延迟、重试率。

**集成上报**
- **Sentry**：上传 source map（Hermes）/dSYM/mapping；`ErrorUtils.setGlobalHandler` 捕获 JS；`setNativeExceptionHandler` 捕获原生；  
- **Firebase Crashlytics/Performance**：崩溃 + 自定义 trace。

**示例：捕获未处理 Promise**
```ts
const orig = globalThis.__unhandledPromiseRejectionHandler;
if (!orig) {
  process.on?.('unhandledRejection', (e: any) => {
    // 归一化 + 上报
  });
}
```

**掉帧诊断**
- 追踪 **UI/GPU**：长时间主线程阻塞（布局、图片解码）；  
- 追踪 **JS**：长任务 > 50ms（可用 `PerformanceObserver` polyfill 或定时采样）；  
- Reanimated 场景：优先把动画逻辑放 UI 线程，减少 `runOnJS`。

---

## 19) 内存与资源管理：图片、事件泄漏、闭包持有、后台驻留

**典型问题与对策**
- **图片 OOM**：避免加载原图（像素内存 = w*h*4）；服务器多规格 + 缩略；列表场景使用小图占位与渐进；及时清理缓存（FastImage）。
- **事件订阅泄漏**：`addListener`/`DeviceEventEmitter`/`AppState`/`NetInfo` 等订阅在组件卸载时**必须移除**；自定义 hooks 封装成**单入口/返回取消函数**。
- **闭包持有**：长生命周期对象（定时器、全局缓存）捕获大对象/`navigation` 导致保活；明确取消 `setInterval`/动画；`useRef` 避免重复绑定。
- **JS ↔ Native 强引用环**：JSI/HostObject 保存对 Native 的 `shared_ptr`，Native 再持有 JS Function → 循环；使用 **弱引用/生命周期钩子** 断环。
- **后台任务**：长连（WebSocket）/后台下载/定位在后台维持进程；必要时在 `AppState` 切后台暂停，前台恢复。

**排查**
- **Heap Snapshot**：对比快照，寻找增长型节点；  
- **Native 内存**：Android `adb shell dumpsys meminfo`，iOS Xcode Memory Graph；  
- **图片**：打开 GPU overdraw、纹理统计；关注大尺寸 PNG/JPEG/GIF。

---

## 20) 安全基线：反编译与混淆、Bundle 保护、证书固定、秘钥存储

**反编译与混淆**
- Android 开启 **R8**：`minifyEnabled true shrinkResources true`；保留必要类（JSI/TurboModule/Reanimated）。  
- iOS 对 Swift/ObjC **无混淆**，但可通过 **符号隐藏/strip** 减少信息；避免把敏感字符串明文放二进制（做简单变形也仅是“遮羞布”）。

**JS Bundle 保护（有限）**
- Metro 产出 bundle 可被提取；可做**轻度加密/压缩**并在原生层解密缓存，但**安全收益有限**（客户端最终可被提取）。  
- 关键逻辑应在服务器或原生安全域实现。

**证书固定（SSL Pinning）**
- 采用 **SPKI 指纹**（公钥哈希）优先；实现可使用 `react-native-ssl-pinning` 或自定义 OkHttp/NSURLSession 配置；  
- 维护**多指纹**以覆盖证书轮换；对调试构建放行代理（白名单域名或关闭 pinning）。

**Keychain/Keystore 安全存储**
- 使用 **Keychain（iOS）/Android Keystore** 保存**短期 token/加密密钥**；  
- Android 6+ 可绑定 **StrongBox/TEE**；  
- 避免把刷新令牌、私钥写入 AsyncStorage；在内存中尽量只短暂持有明文。

**防篡改与环境检查（“提高成本”非绝对防御）**
- Root/Jailbreak 检测、Hook 框架检测（Frida），签名校验与反调试；  
- 完整性校验：本地资源哈希比对 + 服务器白名单；  
- 敏感接口**后端风控**与**二次校验**（如设备指纹、Token 绑定）。

**崩溃与安全事件响应**
- 对异常上报脱敏；错误日志中避免包含 PII/密钥；  
- 安全事件具备**远端开关**（撤销证书指纹、停用某功能），与 OTA/热修复联动。

---
## 21) 权限体系：iOS Info.plist、Android 运行时权限/Scoped Storage；前后台定位/相册/通知

**原则**
- **声明 + 解释 + 运行时请求**三步走；任何“可能唤起系统 UI”的权限请求必须在**交互之后**触发。
- **最小权限**：按**功能路径**解耦（拍照仅 CAMERA，不要一次性申请存储/定位）。
- **平台差异**：iOS 以 **Info.plist 文案** + `requestAuthorization` 为主；Android 以 **Manifest 声明** + **运行时请求**为主，且 **Android 10+ Scoped Storage** 限制外部存储访问。

**iOS 关键点**
- 在 `Info.plist` 写清楚目的键：`NSCameraUsageDescription`、`NSPhotoLibraryAddUsageDescription`、`NSLocationWhenInUseUsageDescription`、iOS 13+ `NSLocationAlwaysAndWhenInUseUsageDescription`、iOS 12+ `NSUserTrackingUsageDescription`（如果用 IDFA）。
- iOS 13+ 背景定位需要 **蓝条** 行为与“Always”权限路径（先 when-in-use，再跳引导到设置）。
- iOS 16+ **通知权限** 需要显式 `UNUserNotificationCenter.requestAuthorization`；推送 token 只有在授权后才稳定。

**Android 关键点**
- `AndroidManifest.xml` 声明权限；**Android 13+（API 33）通知权限** `POST_NOTIFICATIONS`；**位置**分 `ACCESS_COARSE/FINE`，后台定位 `ACCESS_BACKGROUND_LOCATION` 必须**分步**申请。
- **Scoped Storage（Android 10+）**：不要用旧的 `WRITE_EXTERNAL_STORAGE` 流程；选择 `MediaStore`/`SAF`/`MANAGE_EXTERNAL_STORAGE`（仅特殊场景）。
- **前台/后台定位**：后台定位需要强理由与系统表单；配合 `foreground service` 做持续定位。

**RN 实践（react-native-permissions）**
```ts
import {check, request, PERMISSIONS, RESULTS, openSettings} from 'react-native-permissions';

async function ensureCamera() {
  const perm = Platform.select({
    ios: PERMISSIONS.IOS.CAMERA,
    android: PERMISSIONS.ANDROID.CAMERA,
  })!;
  const st = await check(perm);
  if (st === RESULTS.GRANTED) return true;
  const rs = await request(perm);
  if (rs === RESULTS.BLOCKED) { await openSettings(); return false; }
  return rs === RESULTS.GRANTED;
}
```

**常见误区**
- 一次性请求多个危险权限（通过率极低，且被系统判定为滥用）。
- 背景定位/相册写入权限在新系统**语义变化**未跟进，导致一直被拒。
- 需求只读照片却申请了“写入”或“全盘存储”。

---

## 22) 推送通知：APNs/FCM、Token 生命周期、三态处理、渠道（Android）与点击深链

**链路**
- **iOS(APNs)**：App → `UNUserNotificationCenter.requestAuthorization` → `registerForRemoteNotifications` → `didRegisterForRemoteNotificationsWithDeviceToken` → 送到服务端 → 服务端调 APNs。
- **Android(FCM)**：App 首次启动向 FCM 拿 **registration token**，服务端用 token 发消息。国内无 GMS 需厂商通道或第三方。

**消息类型**
- **通知消息**（系统托盘展示，点击回调） vs **数据消息**（仅负载，App 前台/后台自行处理）。iOS 前台默认不弹需要本地展示。

**三态处理**
- **前台**：拦截并自绘（In-App banner），避免打断体验。
- **后台唤起**：点击通知 → 深链路由，读取附带参数；注意 **冷启动**和 **热启动**路径一致性。
- **被杀进程**：Android 需 **兼容厂商调度**；可配合 `Headless JS` 或启动页参数转交。

**Android 渠道**
```ts
// notifee 示例
await notifee.createChannel({ id: 'chat', name: 'Chat', importance: AndroidImportance.HIGH });
await notifee.displayNotification({ android: { channelId: 'chat' }, title, body });
```

**Token 生命周期**
- Token 可能**轮换**（重装、清除数据、切系统用户），启动与显著事件（登录）时**上报绑定**；退出登录要**解绑**。

**调试**
- iOS：`apns-push-type`、`apns-topic` 正确；开发/生产证书区分；用 APNs Provider 测试。
- Android：检查渠道重要级、公测 ROM 的通知总开关；前台/后台行为差异（O+ 限制）。

---

## 23) 国际化（i18n）与本地化：动态切换、时区/历法/数字格式、RTL 布局与字体回退

**框架选型**
- **`react-i18next`**：成熟生态、懒加载命名空间、hooks 友好。  
- **`formatjs`/`Intl`**：复杂**日期/数字/复数**规则；Hermes 新版本的 Intl 支持更好，老版本需 polyfill。

**组织策略**
- 字典分 **命名空间**（`common`/`home`/`profile`）+ **语言包**；按路由懒加载，首屏只装 `common`。
- 文案规范化：**变量占位**、**复数**、**性别/语序**；避免字符串拼接。

**动态切换**
```ts
i18n
 .use(initReactI18next)
 .init({ lng: initialLng, resources, fallbackLng: 'en', compatibilityJSON: 'v3' });

const change = (lng: string) => i18n.changeLanguage(lng); // 同步刷新文案
```

**本地化细节**
- **时区/历法/数字**：用 `Intl.DateTimeFormat/NumberFormat/RelativeTimeFormat`；避免自己做格式化。
- **RTL**：`I18nManager.allowRTL(true)`；RTL 切换需要**重启**或 **强制重绘**；样式使用 **逻辑属性**（`start/end`）而非 left/right。
- **字体回退**：东亚/阿拉伯脚本需定义**字体族链**；Android 不同 ROM 字体差异大。

**资源加载**
- 语言包存放在 **assets** 或远端配置；OTA 更新字典时注意**版本同步**与**回退**。

---

## 24) 可达性（a11y）：Role/Label/Hint、焦点、TalkBack/VoiceOver、动态字体与对比度

**核心属性**
- `accessible`、`accessibilityRole`（`button`/`header`/`image`/`switch`…）、`accessibilityLabel`、`accessibilityHint`、`accessibilityState`。
- 复杂组件将点击区域包到一个 `accessible` 容器，避免读屏散乱。

**焦点管理**
```ts
import {AccessibilityInfo, findNodeHandle} from 'react-native';

const ref = useRef<View>(null);
const focus = () => {
  const node = findNodeHandle(ref.current);
  node && AccessibilityInfo.setAccessibilityFocus(node);
};
```
- 弹窗/Toast 打开后**移动焦点**，关闭后把焦点**还原**到触发控件。

**动态字体/对比度**
- `allowFontScaling` 默认 true；使用 `StyleSheet` 与 `PixelRatio.getFontScale()` 校验极端缩放。  
- 检查颜色对比度（WCAG ≥ 4.5:1），暗黑模式下注意品牌色替换。

**TalkBack vs VoiceOver 差异**
- 手势读屏差异（双指滚动/探索触摸）；  
- VO 会将 `accessibilityHint` 与状态一起读，TalkBack 更依赖 `contentDescription`。

**测试**
- RN Testing Library + 可达性查询（`getByA11yLabel/Role`）；E2E 用 Detox 配合 `testID` 与**可达性树**验证。

---

## 25) WebView 与内嵌 H5：安全（JSBridge/混合内容/下载拦截）、文件上传、OAuth、Cookie 同步

**安全基线**
- 用 `react-native-webview`；限制 `originWhitelist`，默认 `https://*`；禁用混合内容（Android `mixedContentMode="never"`）。
- **JS 注入**：`injectedJavaScript` 与 `onMessage` 通信，**白名单消息协议**、签名校验与序列化校验；禁止执行任意字符串。
- **下载拦截**：Android 需实现 `onFileDownload`，交由系统下载或自定义存储（注意 Scoped Storage）。
- `allowFileAccess`/`allowUniversalAccessFromFileURLs` 非必要勿开。

**文件上传**
- `webview` 的 `onFileDownload` / `onShouldStartLoadWithRequest` 与 `<input type="file">` 的兼容；iOS 需将 `camera/microphone` 权限写入 `Info.plist`。

**OAuth 流程**
- 建议用 **系统浏览器**（ASWebAuthenticationSession/CustomTabs）避免 Cookie 隔离与 XSS；若必须内嵌，使用**特定回调域**与**深链**，并彻底校验 `state`。

**Cookie 同步**
- `react-native-cookies` 或 WebView 自带 CookieManager；登录态更新后**刷新** WebView；跨子域需要 `Domain=.example.com`。

**示例：消息桥**
```tsx
<WebView
  source={{ uri }}
  onMessage={(e) => {
    const data = safeParse(e.nativeEvent.data); // 只接受 {type, payload} 且校验来源
    if (data.type === 'PAY') startPay(data.payload);
  }}
  injectedJavaScript={`window.ReactNativeWebView.postMessage(JSON.stringify({type:'READY'})); true;`}
/>
```

---

## 26) 重型原生能力：地图/相机/音视频；权限与性能、GPU/IO、线程模型

**地图**
- `react-native-maps` / 原生 MapKit/Google Maps；大量 Marker 用 **聚合** 或 **静态瓦片**；频繁更新坐标用 **batch** 接口。
- 轨迹回放：避免每帧 setState；原生层插值 + `AnimatedRegion`。

**相机/视频**
- `react-native-vision-camera`（JSI + Frame Processor）：图像处理在 **UI/专用线程**，**零拷贝**交付到 JS 侧仅传标量/结果。
- 视频播放：`react-native-video` 或平台原生控件；长视频用硬解码；封面图用解码缓存。

**音频**
- Android `ExoPlayer`、iOS `AVAudioSession`；后台播放与音频焦点（Duck/Stop）；通知控制器。

**性能要点**
- **避免大对象过桥**：图像帧/PCM 不要走 JS；用 JSI 或直接在原生处理。
- **线程**：相机/解码/IO 放工作线程；UI 线程只做渲染与轻逻辑；加锁粒度小心造成掉帧。
- **功耗**：相机/定位/持续解码需在后台暂停；帧处理降帧率。

---

## 27) Monorepo 与模块化：workspaces、builder-bob、原生链接、TS/别名

**目标**
- 共享组件/业务包与原生库，统一 lint/build/test/发布。

**结构**
```
/apps/mobile      // RN 应用
/packages/ui      // 纯 JS/TS 组件
/packages/native  // 原生模块（bob 模板）
/tsconfig.base.json
```

**关键配置**
- **Yarn/PNPM workspaces**：去重依赖；确保 `react`/`react-native` 不被 hoist 到多份。
- **Metro**：允许 workspace 源码直引（symlink）：
```js
// apps/mobile/metro.config.js
const path = require('path');
module.exports = {
  watchFolders: [path.resolve(__dirname, '..', '..')],
  resolver: {
    nodeModulesPaths: [path.resolve(__dirname, 'node_modules')],
    disableHierarchicalLookup: true,
  },
};
```
- **iOS**：子包的 `podspec` 正确声明 `source_files` 与依赖；`Podfile` 用 `use_native_modules!` 自动链接。
- **Android**：`settings.gradle` include 子模块；Gradle 版本统一。
- **TS Project References**：在 `tsconfig.base.json` 定义路径别名并用 `composite` 加速增量编译。

**发布原生包**
- `react-native-builder-bob`：生成 CJS/ESM 与类型；保留 `react-native.config.js` 以支持 autolinking。

---

## 28) Metro 打包器：工作机制、缓存/Transformer、inlineRequires、RAM Bundles、Hermes 字节码

**架构**
- **Delta Bundler**：文件为单位的依赖图 + 增量构建；HMR 通过增量补丁。
- **Transformer**：默认 Babel，可自定义（如处理 SVG/GraphQL/Proto）。
- **缓存**：分 **文件缓存** 与 **变换缓存**（Key 包含 env/transformer 版本/tsconfig）。

**优化点**
- `inlineRequires: true`：推迟模块初始化（更快冷启动），但会改变初始化时序（副作用模块谨慎）。
- `maxWorkers` 与 `resetCache`；CI 缓存 `~/.metro`。
- **RAM Bundle**（iOS 可加载分块）：现代 Hermes 字节码方案更主流，建议直接产出 `hermes bytecode`。
- **多入口**：大多通过 **多 bundle** 或 **Split by route**（需自定义 resolver）。

**自定义 SVG Transformer（示意）**
```js
// metro.config.js
const {getDefaultConfig} = require('metro-config');
module.exports = (async () => {
  const cfg = await getDefaultConfig(__dirname);
  cfg.transformer.babelTransformerPath = require.resolve('react-native-svg-transformer');
  cfg.resolver.assetExts = cfg.resolver.assetExts.filter(e => e !== 'svg');
  cfg.resolver.sourceExts.push('svg');
  return cfg;
})();
```

---

## 29) 测试体系：Jest/RTL/Mocks；Detox E2E 与稳定性

**单元/组件**
- **Jest**：启用 `@testing-library/react-native`（RTL）做行为驱动测试；**避免脆弱的快照**，关注可见文本、可达性角色、回调触发。
- **Mock 原生**：对 Camera/Geolocation/WebView 提供最小 mock，保证测试不触平台。
```ts
jest.mock('react-native/Libraries/Animated/NativeAnimatedHelper');
jest.mock('@react-native-async-storage/async-storage', () => mockAsyncStorage);
```

**异步/计时**
- 使用 `fakeTimers` 控制 debounce/throttle；`await waitFor(...)` 直到断言满足。
- 网络 mock：`msw` 或 `nock`；确保测试隔离与幂等。

**E2E（Detox）**
- **Idling Resource**：确保网络/动画空闲，否则 flakiness 高。
- CI：Android Emulator 冷启动参数、iOS 使用 `-detoxServer`；设备分辨率固定。
- 断言：尽量用 **可达性选择器/测试 ID**；避免过短超时。

---

## 30) TypeScript 与类型安全：模板/别名、TurboModule/Component 类型生成、桥接一致性

**项目基线**
```json
// tsconfig.json（核心片段）
{
  "compilerOptions": {
    "target": "ES2020",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "paths": {
      "@ui/*": ["../packages/ui/src/*"]
    },
    "types": ["react", "react-native"]
  }
}
```
- 采用 `moduleResolution: "bundler"` 解决 Metro/TS 对 ESM 的解析差异。
- 开启 `strict` 系列（`noImplicitAny/StrictNullChecks`）。

**TurboModule/Component 类型**
- 用 **TS/Flow spec** 作为**单一事实来源**，`react-native-codegen` 生成类型与桥接胶水；JS/TS 与原生声明一致，避免“参数名/可空性漂移”。
- 自定义组件 Props 用 `codegenNativeComponent<NativeProps>()`，生成 HostComponent，避免 `any` 泄漏。

**导航与数据层**
- React Navigation 配置 **参数表**（`ParamList`）并为 `useRoute/useNavigation` 注明泛型，禁止 `as any`。
- API 层结合 **zod/io-ts** 做**运行时校验**，把远端不可信数据转换为**安全类型**。

**落地细节**
- 对“跨包”导入的类型用 **`exports`/`types`** 显式导出；避免隐式 `index.d.ts` 偏差。
- 用 ESLint `@typescript-eslint/consistent-type-imports` 强制类型导入写法；`tsc --noEmit` 独立在 CI 执行。

---
## 31) 跨端与 Web：react-native-web 的能力/限制；平台特定文件（.ios/.android/.native/.web）与 SSR 协作

**能力与映射**
- `react-native-web` 将 RN 组件映射为 DOM + CSS（如 `View→div`、`Text→span`、`Pressable/Touchable→button/div`），保留 RN 的样式模型（`StyleSheet`、无级联、`flex` 默认 `flexDirection: 'column'`）。
- 事件模型抽象为 RN 语义（如 `onPress`），统一跨端代码；`Accessibility` 属性映射到 ARIA。
- 动画：`Animated` 基于 `requestAnimationFrame` 与样式插值；Reanimated 在 Web 侧基于 `@ungap/structured-clone`/`worklet` 的 polyfill 与 DOM 驱动（版本匹配要小心）。
- 手势：`react-native-gesture-handler` 在 Web 走事件代理，有局限（多指、复杂竞争模型不如原生稳定）。

**限制与差异**
- 无原生模块/JSI（需 Web 替身或 `expo-web` 支持）；摄像头、文件系统等需 Web API 替代。
- 布局默认 `flex` 列方向，与浏览器默认不同（容易误判）；`percentage/auto` 与最小/最大约束在 Web 与 Yoga 行为略有差异，避免依赖边界行为。
- 文本测量/换行策略差异（字距、字形回退、换行算法）；`numberOfLines` 通过 CSS line-clamp 模拟，兼容性受限。

**平台特定文件解析**
- Metro/webpack 解析顺序（示例）：`.web.tsx` → `.native.tsx` → `.tsx`（实际顺序视 resolver 配置）。跨端组织：
  - 公共实现：`Button.tsx`
  - 原生实现：`Button.native.tsx`
  - Web 实现：`Button.web.tsx`
  - 避免在公共实现中引用 `react-native` 无法 polyfill 的 API（如 `NativeModules`）。

**SSR 协作（Next/Remix/Expo Router Web）**
- 关键点：**同构**（Node 端渲染初始 HTML，客户端 hydrate）。
- 需配置 **webpack alias** 与 **babel preset**，确保 RN 源码能被转译到浏览器目标；并把 ESM/TS 通过 babel 处理。
- Hydration 一致性：服务端与客户端样式必须一致；`Appearance`/`Dimensions` 等“运行时环境差异”要延迟到 `useEffect` 后应用，避免 SSR/CSR 标记不匹配。

**Next.js 最小配置示例**
```js
// next.config.js
const withTM = require('next-transpile-modules')([
  'react-native',
  'react-native-web',
  'react-native-gesture-handler',
  'react-native-reanimated',
]);
module.exports = withTM({
  webpack: (config) => {
    config.resolve.alias = {
      ...(config.resolve.alias || {}),
      'react-native$': 'react-native-web',
    };
    config.resolve.extensions = ['.web.js', '.web.tsx', ...config.resolve.extensions];
    return config;
  },
});
```
```json
// babel.config.json
{
  "presets": ["next/babel", "module:metro-react-native-babel-preset"],
  "plugins": ["react-native-reanimated/plugin"] // 若使用 Reanimated
}
```

**工程建议**
- 设计上以“**能力最小公倍数**”驱动：优先使用 RN primitives；平台特化通过 `*.web.tsx`/`*.native.tsx` 分流。
- 样式用 RN 的 `StyleSheet`，不要混入全局 CSS；需要全局样式时限定在 Web shell（如 `_app.tsx`）。
- SSR 中避免在 render 阶段读取 `window`/`document`；与 `useSafeAreaInsets` 等依赖设备环境的 hook 需在 CSR 后应用。

---

## 32) 错误处理：全局 JS 错误、未捕获 Promise、原生异常、崩溃页与生产保护

**JS 层**
- 全局异常：`ErrorUtils.setGlobalHandler((error, isFatal) => { ... })` 可捕获未处理同步异常。
- 未捕获 Promise：部分 RN 版本提供 `globalThis.__unhandledPromiseRejectionHandler`，通用做法是在应用入口挂载：
```ts
// setupErrors.ts
ErrorUtils.setGlobalHandler((e: any, fatal?: boolean) => {
  report(e, { fatal });
  showFatalScreenIfNeeded(e, fatal);
});

const orig = (global as any).onunhandledrejection;
(global as any).onunhandledrejection = (evt: PromiseRejectionEvent) => {
  report(evt.reason, { unhandledRejection: true });
  orig?.(evt);
};
```
- 组件级：`ErrorBoundary` 兜底 UI，隔离局部崩溃：
```tsx
class Boundary extends React.Component { state={e:null as any};
  static getDerivedStateFromError(e:any){ return {e}; }
  componentDidCatch(e:any, info:any){ report(e, {info}); }
  render(){ return this.state.e ? <CrashFallback/> : this.props.children; }
}
```

**原生层**
- iOS：`RCTSetFatalHandler` / `RCTSetUncaughtExceptionHandler`，或使用社区库 `react-native-exception-handler` 的 `setNativeExceptionHandler`。  
- Android：设置默认 `Thread.UncaughtExceptionHandler`；注意不要阻塞或在 handler 中做复杂 IO。
- 崩溃符号化：上传 **dSYM（iOS）**、`mapping.txt（Android R8）`，Hermes 需要 **source map** 用于 JS 栈反解。

**生产保护**
- 对“致命错误”显示**降级页**（离线/维护/重启建议），记录用户操作上下文（breadcrumbs）与设备信息。
- API/状态层对 **不可恢复错误** 标记“只读模式”或“重登模式”，避免陷入错误循环。
- 打点去敏：错误日志脱敏（token/PII）；在上报 SDK（Sentry/Firebase）层做过滤。

**易错点**
- 仅用 ErrorBoundary 无法捕获异步/事件回调中的错误（需全局 handler）。  
- `console.error` 被覆盖但未上报；区分 `__DEV__` 与生产行为（开发期可直接红屏，生产期走降级）。

---

## 33) 应用架构：UI 态 vs 服务器态；状态库选型；模块边界与可测试性

**UI 态 vs 服务器态**
- **UI 态**：组件可直接推导或本地派发的状态（可见性、输入、临时选择等）；生命周期短、与路由耦合。  
- **服务器态**：来自远端、需缓存/失效/并发控制的数据（列表、详情、配置）；具备“取数/缓存/重试/失效”语义。

**建议分工**
- **服务器态 → TanStack Query**（或 SWR）：请求去重、缓存、失效、并发、预取、离线。  
- **UI 态 → 轻量库**：Zustand/Jotai/Recoil/Redux Toolkit 皆可，**慎存**服务器态避免“双份真相”。

**TanStack Query 基本骨架**
```ts
const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 30_000, retry: 2, refetchOnReconnect: true }
  }
});
// App 根
<QueryClientProvider client={queryClient}><App/></QueryClientProvider>

// 取数
const { data, isLoading } = useQuery({
  queryKey: ['post', id],
  queryFn: () => api.getPost(id),
});

// 失效
queryClient.invalidateQueries({ queryKey: ['post', id] });
```

**Redux Toolkit 与 DI**
- 事件驱动的 **领域 slice**（`createSlice`）；副作用统一在 **thunk/observer**；依赖（API/BFF）通过工厂函数注入，便于 mock。
```ts
export const makeStore = (deps: { api: Api }) => configureStore({
  reducer: { cart: cartSlice.reducer },
  middleware: (gDM) => gDM({ thunk: { extraArgument: deps }})
});
```

**模块边界**
- 以“功能域/页面”为单位划分包；公共 UI 与工具下沉到 `@ui`/`@shared`。
- API 层的 **DTO→Domain** 转换在数据边界集中处理，UI 只见 Domain 模型，降低耦合。

**可测试性优先**
- State/副作用隔离：hook 与纯函数分层，hook 只拼装状态，核心业务逻辑纯函数可单测。
- 通过 **Contract Test**（对 API mock）固定接口行为；E2E 验少量关键链路。

---

## 34) 升级与迁移：启用 Hermes/Fabric/TurboModules，新旧模块兼容、回退策略

**总体策略**
- **分阶段**：先升级 RN 版本（确保社区依赖兼容）→ 默认 Hermes → 开 `new architecture` 开关（Fabric/TurboModules）。  
- **可回退**：保留开关与 CI 变体，任何阶段故障可一键回到旧架构。

**步骤**
1) **升级 RN 基线**：遵循 RN Upgrade Helper；确保 Reanimated/RNGH/Navigation/Flipper 等关键库版本匹配。
2) **Hermes**：现代 RN 默认启用；确认 Metro 产出 Hermes bytecode + sourcemap 上报；移除 JSC 相关配置。
3) **启用新架构**
   - Android：`gradle.properties` 加 `newArchEnabled=true`；iOS：`RCT_NEW_ARCH_ENABLED=1` + `use_react_native! :new_architecture_enabled => true`。
   - 运行 `yarn react-native codegen` 或构建时自动生成；修复编译错误（C++17、缺失头文件）。
4) **TurboModules 迁移**
   - 为原生模块补充 **TS/Flow spec**，跑 codegen；Promise 优先，禁止长耗时同步。
   - 保持旧 Bridge 模块并行（过渡期），对外导出相同 JS API。
5) **Fabric 组件迁移**
   - 组件 Props/Events/Commands 的 schema → codegen → Native 实现；验证测量/布局。
   - 无法迁移的使用 Paper 包装层或临时保留旧组件。
6) **灰度**：在 QA/预生产渠道开启新架构，监控崩溃率、TTI、掉帧；问题即刻回滚。

**常见坑**
- 第三方库未适配新架构/JSI：临时锁版本或用 fork；提交 issue/PR。
- Reanimated 版本与 Babel 插件不匹配导致启动崩溃；确保 `react-native-reanimated/plugin` 在 babel 最末尾。
- iOS `use_frameworks!` 与静态库链接冲突；Android NDK/ABI 配置导致 C++ 符号冲突。

**回退**
- 关闭 `newArchEnabled`；保留 Hermes。严重问题可临时回 JSC（不推荐长期）。

---

## 35) 跨端与 Web（SSR 深化）：同构陷阱、Hydration、性能与可达性

**同构陷阱**
- **环境差异**：服务端没有 `window`/`document`；使用 `Platform.OS==='web' && typeof window!=='undefined'` 守卫；把依赖浏览器能力的逻辑放到 `useEffect`。
- **初始状态**：SSR 时应注入与客户端一致的初始 store/query 缓存，避免首次 hydrate 触发重复请求：
```tsx
// Next _app.tsx
<QueryClientProvider client={client}>
  <Hydrate state={pageProps.dehydratedState}><App/></Hydrate>
</QueryClientProvider>
```
- **动态尺寸**：`Dimensions` SSR 阶段不可用，首帧用“保守尺寸”，待 CSR 后用实际尺寸替换（避免内容跳动需占位/骨架）。

**Hydration 一致性**
- 不要在 render 阶段读取时间/随机数造成服务端与客户端标记不一致；若必须，`suppressHydrationWarning` 包裹，并尽快在 CSR 修正。
- 条件渲染在 SSR/CSR 保持相同树形；外观主题（深色/浅色）SSR 前就确定（基于 cookie 或 UA），避免切换闪烁。

**性能**
- `next/dynamic` + `ssr:false` 移除仅客户端组件（手势/动画重型组件）出 SSR 阶段压力。
- 关键路由做 **critical CSS** 与资源预加载；图片走 `next/image`（或 `expo-image` Web）以获得懒加载与响应式。
- 列表虚拟化在 Web 侧用 `react-window`/`react-virtualized` 替代 RN List 的 Web 适配（重列表场景更稳）。

**可达性**
- 结合 Web 的 ARIA 能力：`accessibilityRole`→`role`，补齐必要的 `aria-*`；表单元素用原生 `<input>`/`label` 包装 RN 组件，提升可用性。
- 焦点管理：SSR 初次渲染不要强制聚焦；路由切换后将焦点移至页面主标题（无障碍导航）。

**最小整合示例（SSR 注水）**
```ts
// getServerSideProps
const qc = new QueryClient();
await qc.prefetchQuery(['post', id], () => api.getPost(id));
return { props: { dehydratedState: dehydrate(qc) } };
```
```tsx
// 页面组件
const { data } = useQuery({ queryKey: ['post', id], queryFn: () => api.getPost(id) });
```

---
