# Electron

## 1) 进程架构：Main / Renderer / Preload 的职责与“桥接”关系
- **Main（主进程）**  
  单实例、拥有 Node 与完整操作系统能力。负责生命周期（`app.whenReady()`）、窗口/菜单/托盘的创建与销毁、原生能力（文件系统、协议、系统权限）、自动更新与日志/崩溃上报。对外提供**受控 IPC 服务**（`ipcMain.handle`/`on`）。
- **Renderer（渲染进程）**  
  每个 `BrowserWindow`/`BrowserView` 对应一个或多个渲染进程（进程复用由 Chromium 调度）。承担 UI 与业务逻辑，**默认不应有 Node 能力**。通过**受限 API**与 Main 交互。
- **Preload（预加载脚本）**  
  在目标页面加载前注入，运行于**隔离世界**（`contextIsolation: true`）。它是**唯一被授权的桥**：使用 `contextBridge.exposeInMainWorld` 向页面注入白名单 API，并在内部用 `ipcRenderer` 与 Main 对话；负责参数校验、权限下放与事件解绑。
- **与旧 “Bridge/remote” 的关系**  
  早期常直接在渲染器使用 `remote` 或开启 `nodeIntegration`（相当于把 Main 能力“裸露”到页面），安全边界模糊。现代做法：**禁用 remote & Node 集成** → **Preload 暴露最小 API** → **Main 做能力中心**。这就是“新桥接模型”。

---

## 2) 安全基线：`contextIsolation`/`sandbox`/`nodeIntegration`/CSP 的组合
- **推荐组合（现代默认）**
  ```ts
  const win = new BrowserWindow({
    webPreferences: {
      contextIsolation: true,     // 必开：隔离 page 与 preload 的 global
      sandbox: true,              // 强沙箱：Renderer/Preload 都无 Node 能力
      nodeIntegration: false,     // 页面禁用 Node
      preload: path.join(__dirname, 'preload.js'),
      webSecurity: true,          // 同源/混合内容保护，默认 true
      allowRunningInsecureContent: false,
      enableRemoteModule: false,  // 移除 remote
      devTools: true              // 生产可按环境开关
    }
  })
  ```
- **CSP（内容安全策略）**：在页面层限制脚本来源与内联执行（配合 `trustedTypes` 避免 XSS 注入点）。
  ```html
  <meta http-equiv="Content-Security-Policy"
        content="default-src 'self';
                 script-src 'self';
                 style-src 'self' 'unsafe-inline';
                 img-src 'self' data:;
                 connect-src 'self' https:;
                 object-src 'none';
                 base-uri 'none';
                 frame-ancestors 'none'">
  ```
- **取舍要点**
  - `sandbox:true` 下 **Preload 与页面都没有 Node**；仅能使用 `ipcRenderer/contextBridge` 这类受限 Electron API。任何文件/系统访问都应走 Main。
  - 若必须在 Renderer 使用部分 Node 能力，宁可**专门开一个隔离的窗口/域**，不要为整站点开启 `nodeIntegration`。
  - CSP 要和打包器一致（避免内联脚本/hash 失配），第三方脚本通过**固定源清单**引入。

---

## 3) IPC 安全：`invoke/handle` vs `send/on`，参数校验与最小暴露
- **语义差异**
  - `ipcRenderer.invoke('ch', data)` ↔ `ipcMain.handle('ch', async (e, data)=>result)`：**请求-响应**，单一 handler，异常会被 `Promise` 拒绝（推荐**RPC 语义**）。
  - `ipcRenderer.send('ch', data)` ↔ `ipcMain.on('ch', (e, data)=>{})`：**单向消息**，可能有多个监听，适合事件广播。
  - `sendSync`/`returnValue`：**阻塞进程**，禁用。
- **暴露策略**
  - **禁止**把 `ipcRenderer` 直接暴露到 `window`；只通过 Preload 暴露**具体、窄口的函数**。
  - **白名单信道**：集中常量定义；Main 侧仅注册在白名单内的信道。
  - **输入校验与授权**：在 Preload 或 Main 入口用 schema 校验（zod/ajv），并检查调用上下文（来源 `senderFrame.url` 是否在允许域，是否顶层窗口）。
  ```ts
  // preload.ts
  import { contextBridge, ipcRenderer } from 'electron';
  import { z } from 'zod';
  const ReadFileReq = z.object({ path: z.string().max(4096) });

  contextBridge.exposeInMainWorld('api', {
    readTextFile: async (req) => {
      const ok = ReadFileReq.safeParse(req);
      if (!ok.success) throw new Error('bad request');
      return ipcRenderer.invoke('fs:readText', ok.data)
    }
  })
  // main.ts
  import { ipcMain } from 'electron'
  ipcMain.handle('fs:readText', async (e, { path }) => {
    // 额外校验：只允许工作目录下
    assert(path.startsWith(app.getPath('userData')));
    return await fs.promises.readFile(path, 'utf8');
  })
  ```
  - **输出净化**：去除路径、内网地址等敏感信息，不把系统错误直接回传给页面。
  - **订阅解绑**：页面卸载/窗口关闭时移除侦听，防止“幽灵侦听器”。

---

## 4) Preload 的职责边界与生命周期管理
- **定位**：安全“门面层”。只做**能力封装、参数校验、事件桥接**；不写复杂业务。
- **生命周期**：每个 `webContents` 启动前执行一次；`did-navigate` 不会重载 Preload（除非导航跨进程）。
- **最佳实践**
  1) **版本化 API**：`window.api.v1.readFile()`，为向后兼容留余地。  
  2) **只暴露可序列化数据**（结构化克隆友好），避免把复杂原生对象穿越边界。  
  3) **事件桥接**：在 Preload 聚合底层事件，向页面以 DOM 事件或回调的方式再暴露；并在 `window.addEventListener('unload',…)` 中解绑。
  4) **错误边界**：捕获 `ipcRenderer.invoke` 的异常，转化为统一错误对象。
  ```ts
  // preload.ts
  const listeners = new Set<Function>();
  ipcRenderer.on('job:progress', (_e, p) => listeners.forEach(fn => fn(p)));
  contextBridge.exposeInMainWorld('api', {
    onProgress: (fn: (p:number)=>void) => (listeners.add(fn), () => listeners.delete(fn))
  })
  window.addEventListener('unload', () => {
    ipcRenderer.removeAllListeners('job:progress');
    listeners.clear();
  })
  ```

---

## 5) 渲染进程沙箱化：`sandbox: true` 的影响与“能力补齐”
- **区别**  
  - `nodeIntegration: false`：仅禁用页面里的 Node 全局（`require/fs`）；Preload 仍可用 Node。  
  - `sandbox: true`：**更进一步**，Renderer 与 Preload **都**无 Node 环境；只能使用 `ipcRenderer/contextBridge` 等受限 Electron API。  
- **能力补齐路径**  
  - **一切系统能力走 Main**：文件、网络、剪贴板、打印、通知、托盘、协议……在 Main 封装，通过 Preload 暴露。  
  - **第三方库适配**：依赖 Node 的库在 Renderer 直接用会失败，需：  
    1) 将调用迁移到 Main；或  
    2) 在 Preload 建立“代理 API”；或  
    3) 引入纯 Web 替代（Fetch/File System Access/IDB）。  
  - **调试**：确认 `process.contextIsolated === true`、`process.sandboxed === true`，并在 DevTools Console 验证 `typeof require === 'undefined'`。

---

## 6) 导航/外链安全：阻止任意跳转、控制 `window.open` 与外部浏览
- **阻止不受控导航**
  ```ts
  win.webContents.on('will-navigate', (e, url) => {
    const allow = new URL(url).origin === new URL(win.webContents.getURL()).origin;
    if (!allow) { e.preventDefault(); shell.openExternal(url); }
  })
  ```
- **控制新窗口**
  ```ts
  win.webContents.setWindowOpenHandler(({ url, features, disposition }) => {
    // 白名单域/协议才允许；否则改为外部打开
    const ok = /^https:\/\/(docs|pay)\.example\.com/.test(url);
    return ok ? { action: 'allow', overrideBrowserWindowOptions: { webPreferences: { sandbox: true, contextIsolation: true, nodeIntegration:false } } }
              : (shell.openExternal(url), { action: 'deny' });
  })
  ```
- **外部打开**：始终使用 `shell.openExternal`；**永不**在 `BrowserWindow` 中直接加载不受信任 URL。
- **CSP/Trusted Types**：减少 XSS 导致的恶意导航入口；同时关闭 `allowpopups` 等危险特性。

---

## 7) `BrowserWindow` / `BrowserView` / `WebContents`：差异与多窗体组织
- **概念**
  - `BrowserWindow`：顶级原生窗口容器；拥有一个主 `WebContents`。
  - `BrowserView`：可嵌入窗口的子视图（无系统边框/标题栏），适合多面板/停靠区；可多个 `BrowserView` 叠放与切换。
  - `WebContents`：底层渲染载体（也用于 `<webview>` 标签）。多数 API（导航、调试、打印、session）都在此层。
- **选型**
  - **多标签**：一个 `BrowserWindow` + 多个 `BrowserView`（切换激活视图），或单 `WebContents` + 客户端路由（轻量）；  
  - **多工具面板/浮层**：多个 `BrowserView`；  
  - **独立子应用**：多 `BrowserWindow`。
- **组织/通信**
  - 用 `WeakMap<WebContentsId, WindowState>` 管理窗口状态；跨窗通信：以 Main 为**总线**（广播/定向 `webContents.fromId(id).send()`），避免 Renderer ↔ Renderer 直连。
  ```ts
  const bus = new Map<number, BrowserWindow>();
  const createWin = () => { const w = new BrowserWindow(...); bus.set(w.webContents.id, w); };
  ipcMain.on('broadcast', (e, msg) => {
    for (const [id, w] of bus) if (id !== e.sender.id) w.webContents.send('message', msg)
  })
  ```

---

## 8) 自动更新：渠道、差分、回滚与跨平台取舍
- **方案对比**
  - `electron-updater`（NSIS/dmg/AppImage）：整合发布、通道（stable/beta/dev）、差分（blockmap/签名），DX 佳；  
  - Squirrel（win）/autoUpdater（mac）原生 API：更底层，生态旧；  
  - **MAS**：通过 App Store，Electron 内置自动更新不可用，由商店接管。
- **通道与检查**
  ```ts
  autoUpdater.channel = process.env.CHANNEL ?? 'stable';
  autoUpdater.checkForUpdates(); // 结合后端 feed，返回可用版本与差分包
  autoUpdater.on('update-downloaded', () => autoUpdater.quitAndInstall());
  ```
- **差分更新**：blockmap/签名校验，显著降低下载体积；后端需按版本产出 **全量 + 差分**。失败回退到全量。
- **灰度发布**：按机器/账号 hash 分桶；Main 侧透出“可用更新/强更”状态，Renderer 决策提示。
- **回滚策略**：保留上一个完整安装包与缓存，安装失败/启动异常触发回滚；首次启动记录“健康心跳”。

---

## 9) 代码签名与发布：macOS 公证、Windows 签名链与常见错误
- **macOS**  
  1) 使用 **Developer ID Application** 证书签名；  
  2) 启用 **Hardened Runtime** 与所需 entitlements（屏幕录制、文件访问等最小化）；  
  3) **notarization** 上传 Apple，完成后 **staple** 到 app/dmg；  
  4) 常见失败：未签子组件（`*.node`、辅助二进制）、entitlements 不匹配、使用了禁用私有权限。
- **Windows**  
  - 代码签名（最好 **EV 证书**，SmartScreen 信誉更快建立）；  
  - 使用 RFC3161 时间戳服务器（`/tr`），保证证书过期后仍然有效；  
  - 常见失败：证书链不全、时钟漂移、签完后又修改了文件（破坏签名）。
- **发布工件**：区分 x64/arm64 与 **Universal（mac）**；产出带符号表（dSYM/PDB）供崩溃解析，敏感路径做脱敏。

---

## 10) 打包工具链：`electron-builder` / `forge` / `packager` 的差异与常见坑
- **定位**
  - `electron-packager`：最基础的打包（无安装器/更新）。  
  - `electron-forge`：脚手架与插件化构建（可以挂 `maker-…` 与 `publisher-…`）。  
  - `electron-builder`：**一站式**（安装器、签名、自动更新、发布），社区最常用。
- **`electron-builder` 关键配置**
  ```json
  {
    "appId": "com.example.app",
    "asar": true,
    "files": ["dist/**", "package.json"],
    "extraResources": [{ "from": "assets/", "to": "assets" }],
    "mac": { "category": "public.app-category.productivity", "hardenedRuntime": true, "entitlements": "entitlements.mac.plist", "target": ["dmg","zip"] },
    "win": { "target": [{ "target": "nsis", "arch": ["x64","arm64"] }], "signAndEditExecutable": true },
    "linux": { "target": ["AppImage","deb"] },
    "publish": [{ "provider": "generic", "url": "https://updates.example.com/downloads" }]
  }
  ```
- **常见坑**
  - **ASAR 与动态加载**：运行时需要解压写入的文件（如字典/插件）应放 `extraResources`，不要打进 asar。  
  - **native 模块**：需用 `electron-rebuild` 或预构建 ABI（`prebuild`），否则运行时报 `MODULE_VERSION` 不匹配。  
  - **源码映射/调试**：生产 Source Map 单独上传到错误平台，不要随安装包分发。  
  - **多平台差异**：Apple Silicon 需单独 `arm64` 或构建 `universal`；Windows 代理/杀软可能拦截更新服务，需回退镜像/签名白名单。  
  - **自动更新**：`publish.url` 要与应用内 `autoUpdater` 的 feed 一致；CDN 需配置 CORS/Range 请求支持（差分）。

---
## 11) 资源与 ASAR：虚拟包系统、优缺点与“敏感资源加固”
- **ASAR 是什么**  
  Electron 的**只读归档格式**（类似 tar），Chromium 会把 `app.asar` 挂载成**虚拟文件系统**。`fs.readFile('…/app.asar/x.json')` 正常工作；但**不是加密/混淆**，可被解包。
- **优点**  
  1) **减少小文件 IO**：系统一次性顺序读取，启动更快；  
  2) **发布结构干净**：单一包体，易签名/公证。  
- **缺点/限制**  
  - 需要**真实路径**的场景不适合：原生模块加载、外部进程读文件、`child_process.spawn` 目标、系统库依赖、`dynamic import()` 某些实现。  
  - 需要**写入/修改**的资源不可放 ASAR（只读）。  
- **解决策略**  
  - `electron-builder`:  
    - `asar: true` + `asarUnpack: ["**/*.node", "dict/**"]` 将匹配项**解包**到旁路目录；  
    - 需要可写/可执行的文件放 `extraResources`（安装时复制到 `resources/`）。  
  - **运行时解压**：确实需要临时落盘时，复制到 `app.getPath('userData')/tmp-…`，用完删除，并对落盘内容做**签名校验**（见下）。
- **敏感资源加固**  
  1) **数字签名/校验**：对关键脚本/模型/策略文件生成独立哈希或签名，运行前用内置公钥校验；  
  2) **最小暴露**：不要把密钥/私钥打进包（使用系统钥匙串：Keychain/DPAPI）；  
  3) **协议白名单**：加载本地资源用自定义 `app://` 协议只指向受控目录（见第 19 题）；  
  4) **防篡改监控**：主进程定期校验关键资源哈希并上报异常。

---

## 12) 性能治理：首窗白屏、热路径、节流与泄漏排查
- **首窗白屏**  
  - `show:false` + `'ready-to-show'` 再 `win.show()`；  
  - **预加载/预编译**：将框架与首屏 JS 打到一个 chunk，减少 RTT；  
  - **Splash**：独立 `BrowserWindow` 显示轻量启动画面，主窗就绪后替换；  
  - **协议加速**：本地资源走 `app://`，避免 `file://` 的 CORS/路径问题与慢查找。
- **热路径优化**  
  - 渲染线程：首帧仅做必需渲染，非关键组件 `requestIdleCallback`/路由懒加载；  
  - 主线程：避免同步磁盘 IO；把 CPU 重任务放 `Worker` 或交给主进程子进程。  
  - 关闭后台节流：对后台需持续刷新场景设 `backgroundThrottling:false`，否则维持默认节流以省电。
- **内存与资源释放**  
  - 销毁窗口：`win.destroy()` 后**清空引用**；`webContents` 事件 `'render-process-gone'` 监控异常退出；  
  - 释放媒体/捕获流、移除事件监听（Preload 里维护监听集合并在 `unload` 清空）；  
  - 周期性快照：`process.getProcessMemoryInfo()`、`renderer.getResourceUsage()` 建立基线告警。
- **排查工具**  
  - DevTools Performance/Memory（查 Detached DOM、Listener 泄漏、长任务）；  
  - `contentTracing` 生成 Chrome Trace（见第 18 题）；  
  - `--enable-logging --v=1` 观察导航/崩溃日志。

---

## 13) GPU 与渲染：硬件加速开关、ANGLE 后端与合成层诊断
- **硬件加速何时关闭**  
  - 显卡/驱动黑名单、远程桌面/虚拟机兼容问题、稳定性优先（后台工具类）；  
  - 关闭需在 `app.whenReady()` **之前**调用：`app.disableHardwareAcceleration()`。  
  缺点：所有绘制走 CPU 合成，动画/视频/3D 性能下降明显。
- **ANGLE 后端**（Chromium 图形抽象）  
  - Windows：默认 D3D11；可用环境变量/命令行切到 D3D9/GL；  
  - macOS：Metal；Linux：OpenGL/Vulkan（版本依赖）。问题定位可打印 `gl.getParameter(VENDOR/RENDERER)` 或查看 `chrome://gpu` 等效信息。
- **合成层/过度绘制**  
  - DevTools “Layers/Rendering” 打开**重绘闪烁**与**合成边界**；  
  - 过多独立层会耗显存/带宽，减少 `will-change` 滥用；  
  - 大面积半透明叠加 → 移到低分辨率 FBO 后合成（见 24 题）；  
  - 像素比例高的 4k 屏建议限制 `setPixelRatio` 上限。

---

## 14) 桌面捕获：`desktopCapturer` × `getUserMedia`，权限与平台差异
- **抓取流程**  
  1) `desktopCapturer.getSources({ types:['window','screen'], thumbnailSize })` 列举源；  
  2) 用户选择源 → 用 `navigator.mediaDevices.getUserMedia({ video:{ mandatory:{ chromeMediaSource:'desktop', chromeMediaSourceId: source.id }}, audio:… })` 获取流；  
  3) 在 `<video>` 播放或 `MediaRecorder` 录制。
- **音频**  
  - Windows：可通过 **系统回放环回** 捕获系统音频（驱动允许时）；  
  - macOS：系统音频通常需要**虚拟声卡**（如 BlackHole）；捕麦克风需 `NSMicrophoneUsageDescription`。  
- **屏幕权限**  
  - macOS 10.15+：首次捕获触发**屏幕录制**权限（系统设置→隐私）；权限未授权会返回黑屏；  
  - Windows/Linux 一般无 OS 级弹窗，但 Wayland/Sandbox 组合下可能受限。  
- **安全与 UX**  
  - 自行做**源预览与确认对话**，不要默认采集全部屏幕；  
  - 在 Preload 把捕获 API **参数校验**后再调用，禁用任意 URL 触发采集；  
  - 录制文件落盘到 `userData` 并给出大小/持续时间上限。

---

## 15) 托盘/菜单/任务栏：多平台差异、冲突与图标资源
- **Tray**（Windows/Linux/部分桌面环境）  
  - `new Tray(icon)` + `tray.setContextMenu(Menu.buildFromTemplate([...]))`；  
  - 高 DPI 提供多分辨率图标；监听 `click/right-click/double-click` 行为差异。  
- **macOS**  
  - `app.dock` 控制 Dock 图标与 Badge；系统托盘为 **Status Bar**（`Tray` 也可用），右键菜单不一致；  
  - 标题栏拖拽：无边框窗体用 `-webkit-app-region: drag`，注意**可点击区域**要 `no-drag`。
- **Jump List（Windows）与 Dock Menu（macOS）**  
  - `app.setUserTasks()` / `app.dock.setMenu()` 提供快捷入口；  
  - Windows `setThumbarButtons` 可在任务栏缩略图加操作按钮。  
- **全局热键**  
  - `globalShortcut.register()` 需处理**冲突/失效**（按平台避让系统快捷键）；在 `will-quit` **注销**。  
- **常见坑**：重复创建 Tray（内存泄漏）、透明 PNG 在不同主题下不可见、Linux 桌面环境不支持托盘需降级方案。

---

## 16) 通知体系：支持度、交互限制与回传
- **能力判断**：`Notification.isSupported()`；Electron 的 `new Notification()` 走各平台原生通道。  
- **平台差异**  
  - Windows：操作中心，按钮/进度条支持较好；  
  - macOS：通知中心，**交互按钮受限**，自定义输入/富交互不可用；  
  - Linux：取决于通知守护进程，能力不一致。  
- **回传与深链**  
  - 监听 `notification.on('click'|'action')`，在回调中通过 IPC 通知渲染层；  
  - 复杂交互用 **自定义小窗** 代替系统通知。  
- **注意**：不要把敏感信息放在通知正文；静默/打扰级别按平台规范设置（`silent/urgency`）。

---

## 17) 电源与系统事件：`powerSaveBlocker` / `powerMonitor`
- **阻止休眠**  
  ```ts
  const id = powerSaveBlocker.start('prevent-app-suspension'); // 或 'prevent-display-sleep'
  // …需要时…
  powerSaveBlocker.stop(id);
  ```
  仅在必要时启用，结束后务必 `stop`，否则**耗电**。
- **监控系统电源事件**  
  - `powerMonitor.on('suspend'|'resume'|'on-ac'|'on-battery'|'shutdown'|'lock-screen'|'unlock-screen', …)`；  
  - `shutdown` 可 `e.preventDefault()` 做收尾再退出（时间有限）。  
- **实践**：长任务（转码/下载）在电池电量低时提示或自动降速；媒体播放时仅阻止**显示熄灭**而不阻止系统休眠。

---

## 18) 单实例/深链：参数转发、聚焦与跨平台差异
- **单实例**  
  ```ts
  const got = app.requestSingleInstanceLock();
  if (!got) app.quit(); else {
    app.on('second-instance', (_e, argv, _cwd) => {
      focusMainWindow();
      // Windows: 深链/文件路径在 argv；macOS: 见下
      mainWindow.webContents.send('app:argv', argv);
    });
  }
  ```
- **深链（自定义协议）**  
  - Windows：注册 `myapp://`，后续 `argv` 含 URL；  
  - macOS：监听 `app.on('open-url', (e, url) => …)`；应用未运行时系统会拉起后再触发事件。  
- **聚焦**：`win.isMinimized() && win.restore(); win.show(); win.focus();`；多桌面需 `app.focus({ steal: true })`（版本依赖）。  
- **安全**：解析 URL 时**只接受白名单 host/path**，拒绝注入 payload；不要直接把深链参数拼接到文件路径或命令行。

---

## 19) 自定义协议与文件关联：注册、特权与风控
- **协议注册**  
  - 早期在 `app.whenReady()` 前：`protocol.registerSchemesAsPrivileged([{ scheme:'app', privileges:{ secure:true, standard:true } }])`；  
  - 就绪后：`protocol.registerFileProtocol/BufferProtocol/StreamProtocol/HttpProtocol` 处理请求。  
- **设计建议**  
  - 用 `app://` 只服务**内部静态资源**，路由到安装目录或 `asar` 内固定清单；  
  - 严禁把任意磁盘路径映射出去；  
  - 配合严格 **CSP**，禁止内联脚本/第三方域脚本。
- **文件关联**  
  - 安装时注册扩展名（`.abc`）关联；macOS 监听 `open-file`，Windows 从 `argv` 接收路径。  
  - 打开文件前做**白名单/大小/签名**校验，避免 Zip Slip/路径穿越（`path.normalize` + `startsWith(allowRoot)`）。

---

## 20) 文件系统与权限：对话框、安全路径与大文件管线
- **打开/保存对话框**  
  ```ts
  const { canceled, filePaths } = await dialog.showOpenDialog(win, {
    title:'Open data', properties:['openFile','dontAddToRecent'],
    filters:[{ name:'JSON', extensions:['json'] }]
  });
  ```
- **路径安全**  
  - 仅允许访问**受控根目录**（如 `app.getPath('userData')` 或用户选择的单个文件）；  
  - `path.normalize + resolve` 后检查 `startsWith(allowRoot)`；  
  - Renderer 不直接传绝对路径，由 Main 侧握有**权限与真实路径**。
- **大文件读写**  
  - 使用**流式** API（`fs.createReadStream/WriteStream`），配合 `pipeline` 背压；  
  - CPU 密集型解析放 `Worker`/子进程，避免阻塞 UI；  
  - 落盘临时文件 + **原子写**（写到 `*.tmp`，成功后 `rename` 覆盖），防止中断导致损坏。  
- **沙箱下的能力**  
  - `sandbox:true` 时 Renderer 不能直接 `fs`，统一通过预定义 IPC（见第 3、5 题）调用主进程执行文件操作。

---
## 21) 数据持久化：SQLite/LevelDB/IndexedDB 取舍与敏感数据加固
- **方案选型**
  - **IndexedDB**（Renderer）：零依赖、事务简易、与前端生态协同（Dexie）。局限：大表/复杂查询/多窗口并发下易遇到锁冲突。
  - **LevelDB/RocksDB**（Main/原生模块）：键值模型、写入吞吐高；需要 Node-API 模块，打包/ABI 兼容要注意。
  - **SQLite**（Main）：关系查询、索引能力强，生态成熟（better-sqlite3 同步、sqlite3 异步）。大多走原生模块。
  - 经验：中小型离线数据 → IndexedDB；日志/缓存 → LevelDB；报表/查询/迁移 → SQLite。
- **进程边界**
  - 统一**由 Main 管理数据库句柄**；Renderer 发起受控 IPC 调用（见 3 题）。避免多窗口直接打开同一 DB 句柄导致锁表/崩溃。
- **敏感数据**
  - 使用系统钥匙串：`keytar` 存储令牌/密钥；密文落盘用 **AEAD**（AES-GCM/ChaCha20-Poly1305），密钥仅存钥匙串，**永不硬编码**。
  - 数据加盐哈希（Argon2/scrypt）保存口令派生值；导出备份前做**再加密**与校验（HMAC）。
- **迁移与备份**
  - 设计 `meta` 表维护 `schemaVersion`；每次升级运行**向上迁移**脚本（幂等）。
  - 大文件存储→用**内容寻址**（sha256 目录）+ 引用表，写入走“写临时文件→校验→原子 rename”。

---

## 22) 崩溃与日志：Crashpad、minidump 上报与符号表管理
- **Crashpad 启用**
  ```ts
  import { crashReporter } from 'electron';
  crashReporter.start({
    productName: 'MyApp',
    companyName: 'ExampleCo',
    submitURL: 'https://crash.example.com/minidump',
    uploadToServer: true, compress: true, ignoreSystemCrashHandler: false,
    extra: { channel: process.env.CHANNEL ?? 'stable' }
  });
  ```
  生成 `.dmp`（minidump），Main/Renderer 皆受控。
- **符号表**
  - macOS 产出 `dSYM`，Windows 产出 `PDB`。打包时**上传到符号服务器**（Sentry/Bugsnag/自建）对应版本。
  - CI 中将符号工件与安装包绑定，清理策略按版本保留。
- **日志**
  - `electron-log` 或自建 `winston` 日志管线，分级（info/warn/error）、**环形归档**，避免撑爆磁盘。
  - 采集**崩溃前事件**：未捕获异常（Renderer `window.onerror`/`unhandledrejection`；Main 进程 `process.on('uncaughtException')`），上报时脱敏（路径/IP/token）。
- **隐私与合规**
  - 明确用户同意；minidump 中可能包含内存片段，上传前**Hash/截断路径**，并允许用户关闭上报。

---

## 23) 调试与剖析：DevTools、`--inspect`、Source Map、`contentTracing`
- **Renderer 调试**
  - 常规 DevTools 性能/内存；长任务（>50ms）与掉帧分析。
  - `--enable-precise-memory-info` 获取细粒度内存。
  - Source Map：生产把 `*.map` 上传错误平台，安装包中**不分发**。
- **Main 调试**
  - `--inspect=9229` / `--inspect-brk=9229` 附加到 Node DevTools；或 `electron --remote-debugging-port=9222` 调试所有 WebContents。
- **Chrome Tracing（全局）**
  ```ts
  import { contentTracing } from 'electron';
  await contentTracing.startRecording({included_categories:['-*','blink','v8','devtools.timeline','disabled-by-default-v8.cpu_profiler']});
  // …运行场景…
  const path = await contentTracing.stopRecording();
  // 用 chrome://tracing or Perfetto 打开
  ```
  分析 UI 线程、合成、IO、V8、GPU 时间线，定位瓶颈（过度绘制、同步 IO、垃圾回收抖动）。

---

## 24) 原生模块与 Node-API/JSI：编译、预构建与 ABI 兼容
- **Node-API（N-API）优先**：与 Node/Electron ABI 脱钩，升级更稳。编写 C/C++（`node-addon-api`）或 Rust（napi-rs）。
- **预构建与安装**
  - 使用 `prebuild`/`prebuildify` 产出 `electron-v{ABI}-{platform}-{arch}` 包体；`optionalDependencies` + 按需下载。
  - 备用路径：`electron-rebuild` 在 postinstall 针对当前 Electron 版本重编。
- **ABI 对齐**
  - Electron 每版绑定特定 Node/V8 ABI；**锁定 electron 与原生模块版本矩阵**。CI 覆盖 `win/macos/linux × x64/arm64`。
- **沙箱考虑**
  - `sandbox:true` 下 Renderer 不能直接加载 `.node`，改由 Main 调用或在 Preload 侧以 IPC 提供包装 API。
- **崩溃防护**
  - 原生模块异常会**干掉进程**；对外提供**幂等/边界检查**、避免传错指针；在 Main 侧包裹为**隔离子进程**（worker/utility）更稳。

---

## 25) 窗体形态：无边框/透明/毛玻璃、命中区域与输入
- **无边框与拖拽**
  ```css
  /* 渲染层 */
  .titlebar { -webkit-app-region: drag; }
  .btn { -webkit-app-region: no-drag; }
  ```
  关闭系统标题栏：`new BrowserWindow({ frame:false, titleBarStyle:'hiddenInset' })`（macOS）。
- **透明与阴影**
  - 透明窗体：`transparent:true, backgroundColor:'#00000000'`；Windows 圆角/阴影需配合 `setVibrancy` 或第三方方案。
- **毛玻璃/亚克力**
  - macOS：`win.setVibrancy('sidebar' | 'under-window' | …)` 或 `visualEffectState: 'active'`。
  - Windows 11 亚克力/云母：社区方案（`electron-acrylic-window`），需注意性能与兼容。
- **输入命中**
  - 点击穿透：`win.setIgnoreMouseEvents(true, { forward:true })`；用于悬浮层/贴边吸附。
  - 注意可访问性：自绘标题栏需补全键盘可达与系统菜单映射。

---

## 26) 打印与导出：`print`/`printToPDF` 跨平台差异与版式稳定
- **打印**
  ```ts
  win.webContents.print({ silent:false, printBackground:true, deviceName:'…' }, (ok)=>{});
  ```
  受系统驱动影响较大；Windows 可选虚拟打印机，macOS 走系统对话框。
- **PDF 导出**
  ```ts
  const pdf = await win.webContents.printToPDF({
    pageSize:'A4', margins:{ top:0, bottom:0, left:0, right:0 },
    printBackground:true, landscape:false, scaleFactor:100, headerFooter:false
  });
  ```
  - 页眉/页脚模板（Chrome 版式），定位误差用 CSS `@page` 与 `margin` 调整。
  - 字体嵌入：将所用字体**打包**并在页面层 `@font-face`；避免系统缺字导致换行变化。
- **稳定性策略**
  - 导出页面单独路由，**禁用动画/懒加载**；使用标准化 CSS（normalize），避免不同平台默认样式差异。

---

## 27) 剪贴板与拖放：`clipboard`、`nativeImage`、跨应用格式与安全
- **剪贴板**
  ```ts
  import { clipboard, nativeImage } from 'electron';
  clipboard.writeText('hello');
  const img = nativeImage.createFromPath('logo.png');
  clipboard.writeImage(img);
  // 自定义格式
  clipboard.write({ text:'x', html:'<b>x</b>' });
  ```
- **拖拽（Renderer→系统）**
  ```ts
  const { ipcRenderer } = require('electron');
  // 用 webContents.startDrag 需走 Main
  // preload 暴露：
  ipcRenderer.invoke('start-drag', { file: '/tmp/a.txt', icon: '/path/drag.png' });
  // main:
  ipcMain.handle('start-drag', (e, {file, icon}) =>
    e.sender.startDrag({ file, icon: nativeImage.createFromPath(icon) }));
  ```
- **拖入（系统→Renderer）**
  - 监听 `drop`，读取 `DataTransfer` 中的文件列表；**不要**直接在 Renderer 打开任意路径，交给 Main 侧**白名单校验**后处理。
- **安全**
  - 防止 **Zip Slip**：`path.normalize` + `startsWith(allowRoot)`。
  - 图片/HTML 粘贴需做**消毒**（DOMPurify），避免内联脚本入侵。

---

## 28) 可访问性（a11y）：屏幕阅读器、语义与高对比度
- **启用与检测**
  - Electron 自动适配系统屏幕阅读器；可检测 `app.isAccessibilitySupportEnabled()`。
  - 使用标准语义元素（`<button>`/`<nav>`），ARIA roles/labels 清晰。
- **键盘可达**
  - 自绘标题栏/菜单须补 `tabIndex` 与键盘导航；确保焦点轮换不被 `-webkit-app-region: drag` 捕获。
- **高对比度/动态字体**
  - 监听 `nativeTheme.shouldUseHighContrastColors`/`prefers-contrast`，切换主题；
  - 响应系统字体缩放（`window.matchMedia('(prefers-reduced-motion)')`、缩放倍率）。
- **减少动画**
  - 遵循 `prefers-reduced-motion`，禁用大面积视差/不必要过渡。

---

## 29) 会话与网络：`session`、Cookie/Cache、代理、`webRequest` 与证书校验
- **分区**
  ```ts
  const ses = session.fromPartition('persist:work'); // 持久化分区
  const wc = new BrowserWindow({ webPreferences: { session: ses }});
  ```
  各分区 Cookie/Cache 隔离；用于多账户/多环境。
- **Cookie/Cache**
  ```ts
  await ses.clearCache();
  await ses.cookies.set({ url:'https://a.com', name:'sid', value:'…', sameSite:'lax', secure:true });
  ```
- **代理**
  ```ts
  await ses.setProxy({ mode:'fixed_servers', proxyRules:'http=host:8080;https=host:8080', proxyBypassRules:'<local>' });
  ```
- **请求拦截**
  ```ts
  ses.webRequest.onBeforeSendHeaders((details, cb)=>{
    details.requestHeaders['X-App-Version']=app.getVersion();
    cb({ requestHeaders: details.requestHeaders });
  });
  ```
- **证书校验**
  ```ts
  app.on('certificate-error', (e, wc, url, error, cert, cb) => {
    e.preventDefault();
    // 严格校验：仅白名单 host + 指纹允许
    const ok = allowHosts.has(new URL(url).host) && cert.fingerprint==='…';
    cb(ok);
  });
  ```
  原则：**最小例外**，拒绝自签证书除非企业内网且有指纹固化。

---

## 30) 生命周期与平台差异：`ready/activate/window-all-closed`、焦点与菜单
- **关键事件**
  - `app.whenReady()`：创建窗口/菜单时机；
  - `app.on('activate')`（macOS）：Dock 点击时若无窗体，**重建主窗**；
  - `app.on('window-all-closed')`：Windows/Linux 直接 `app.quit()`；macOS 常保活，直到显式退出。
- **菜单/焦点**
  - macOS 需创建应用级菜单栏（至少包含 `App`/`Edit`）以启用系统快捷键（复制/粘贴）。
  - 焦点转移：新窗口 `show:false` 先加载再 `show()`；防止抢焦点可设 `focusable:false`/`alwaysOnTop:false`。
- **退出顺序**
  - 阻止系统关机/重启：监听 `before-quit`/`session-end` 做收尾，**不要长时间阻塞**。

---

## 31) CI/CD：多平台构建、签名/公证自动化与密钥保护
- **矩阵构建**
  - macOS 构建 mac 包（签名/公证必须在 macOS）；Windows 签名在 Windows 环境；Linux 可互相交叉但签名能力有限。
- **缓存**
  - 缓存 `node_modules`/`~/.cache/electron`/`~/.cache/electron-builder`；原生模块用 `prebuild` 缓存分发。
- **密钥保护**
  - 证书/Apple API Key 放在 CI **密文仓**；签名时解密到临时 Keychain/CertStore，用后删除。
- **自动公证（macOS）**
  - 使用 `appleId/appleIdPassword` 或 App Store Connect API Key；流水线等待公证结果并 `staple`。
- **灰度分发**
  - 将安装包与 `latest.yml`/`appcast` 上传 CDN；分渠道目录隔离；回滚保留 N 个版本并回写 `latest.yml`。

---

## 32) 测试策略：单元/集成/E2E（Spectron 替代）
- **单元（Renderer/Main）**：Jest/Vitest，IPC 层做**契约测试**（mock `ipcMain`/`ipcRenderer`）。
- **集成（Preload + IPC）**：在 Node 环境起 `ipcMain` 处理器，使用 `@electron/remote` 的**替代模拟**或 `electron-mocha`。
- **E2E**
  - **Playwright** 官方支持 Electron：
    ```ts
    const electronApp = await _electron.launch({ args:['.'] });
    const win = await electronApp.firstWindow();
    await win.getByRole('button', { name:'Login' }).click();
    ```
  - **WebdriverIO/Chromedriver** 也可驱动 `app`。
  - 稳定性：**禁用动画**、固定网络/时钟、给 IPC 结果显式等待；测试号/测试服务隔离。
- **可测性设计**
  - Preload 暴露**纯函数风格** API；Main 将副作用集中；为关键流程提供“探针”事件。

---

## 33) Web 安全策略：`webSecurity`、CSP、Trusted Types、隔离域
- **`webSecurity:true`**（默认）保持同源策略与混合内容阻断；绝不要全局关闭。
- **CSP**：严禁内联脚本；仅允许 `script-src 'self'` 与特定 CDN；`object-src 'none'`；`frame-ancestors 'none'`。
- **Trusted Types**：引入 `trustedTypes.createPolicy` 封装 DOM 注入，阻断常见 XSS sink。
- **隔离域**：第三方页面放在**独立 `BrowserView` + 分区 session**，开启 `sandbox:true`；禁用 `nodeIntegration`。
- **下载与执行**：任何可执行内容（脚本/扩展）**不得**在 Renderer 决策；必须经 Main 白名单、签名验证与落盘到受控目录。

---

## 34) 权限与合规：摄像头/麦克风/屏幕录制、MAS Entitlements、隐私条款
- **权限声明**
  - macOS `Info.plist`：`NSCameraUsageDescription`、`NSMicrophoneUsageDescription`、`NSDocumentsFolderUsageDescription`、屏幕录制权限（系统级设置）；  
  - Windows：系统无集中弹窗，但 UWP 能力无关；合规提示靠应用自身。
- **MAS（Mac App Store）**
  - 开启沙箱与必要 entitlements；某些 API（私有/自更新/执行外部二进制）不可用，需要降级实现。
- **隐私与合规**
  - 告知收集项与用途，允许用户关闭遥测；崩溃/日志上报脱敏；敏感数据不写日志。
- **企业合规**
  - 代理/证书固定（pinning）政策、离线模式、数据保留期限；为安全审计提供**配置开关**与导出报表。

---

## 35) 迁移与升级：大版本断点、`remote` 移除、兼容与回滚
- **断点类型**
  - **Chromium 升级**：Web API/行为变化、GPU/ANGLE 后端改变；  
  - **Node/V8 升级**：原生模块 ABI 变动；  
  - **Electron API**：弃用/默认值变化（如 `contextIsolation` 默认开启）。
- **`remote` 模块**
  - 已移除。替代：**Preload + `contextBridge` + `ipcRenderer.invoke`**；若第三方库强依赖，临时采用 `@electron/remote`（谨慎、仅内部工具）。
- **兼容矩阵**
  - 在 CI 跑 **双版本**（当前稳定 + 即将升级），覆盖三平台；原生模块准备预构建产物。
- **回滚与金丝雀**
  - 自动更新配合**分桶灰度**，监控崩溃率/启动失败率；异常自动回滚上一版。安装器保留上一个完整包。
- **迁移步骤**
  1) 升级到目标 Electron；  
  2) 修复弃用 API/默认值（如 `nativeWindowOpen`、安全默认）；  
  3) 原生模块重建/预构建；  
  4) 全量回归 + 金丝雀发布；  
  5) 观察指标达标后扩大放量。

---
