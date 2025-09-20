# 图形渲染

## 1) Canvas 2D / SVG / WebGL / WebGPU：编程模型与选型边界
- **Canvas 2D**：**立即模式**（immediate mode），调用即栅格化；状态机（`save/restore`）+ 当前路径（`beginPath`/`stroke`/`fill`）。适合像素级绘制、位图处理、动态图表。局限：CPU 侧重绘为主，场景大/频繁重绘会吃力。
- **SVG**：**保留模式**（retained mode），DOM 节点即图形，浏览器维护属性→渲染。优点：无损缩放、可 CSS/事件/可访问性；局限：大量节点（>几千）会触发布局/样式开销。
- **WebGL**：OpenGL ES 2.0/3.0 浏览器绑定，**GPU 管线编程**（GLSL 顶点/片元）。适合海量几何、后处理、3D。局限：API 历史负担、资源/状态管理成本、异步调试较难。
- **WebGPU**：现代图形/计算 API，**显式资源绑定**（Bind Group）、更强的**并行/计算着色**（WGSL），更接近 D3D12/Metal/Vulkan。适合前沿渲染/计算、跨平台一致性更好；生态尚在成长。
- **常见选型**  
  - UI/报告/导出：**SVG**（语义 + 无损）或 **Canvas 2D**（像素级图表）。  
  - 海量点/动画：**Canvas 2D + 离屏** 或 **WebGL**。  
  - 3D/后处理/粒子：**WebGL/Three.js**。  
  - 需要计算着色/统一现代管线：**WebGPU**。

---

## 2) 坐标系与 DPR（Retina）：缩放与清晰度
- **逻辑像素 vs 物理像素**：CSS 像素（布局）与设备像素（栅格化）比例为 **`window.devicePixelRatio`**。高 DPR（如 2）若不放大画布，会**模糊**。
- **Canvas 标定**（避免拉伸模糊）
  ```js
  const dpr = Math.ceil(window.devicePixelRatio || 1);
  const canvas = document.querySelector('canvas');
  const rect = canvas.getBoundingClientRect();
  canvas.width  = Math.round(rect.width  * dpr);
  canvas.height = Math.round(rect.height * dpr);
  const ctx = canvas.getContext('2d');
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0); // 或 ctx.scale(dpr, dpr)
  // 之后仍以 CSS 逻辑坐标作图
  ```
- **像素对齐**：绘 1px 线时半像素对齐减少混叠：`ctx.moveTo(x+0.5, y+0.5)`（在 scale=1 场景）；缩放后以几何对齐为准。
- **文字/位图**：  
  - 文本：`textBaseline/Align` 精确对齐，尽量避免小字号非整像素位置。  
  - 位图：准备 2x/3x 素材或用 **向量字体/SDF**；设置 `imageSmoothingQuality='high'`。

---

## 3) 2D/3D 变换矩阵与乘法顺序
- **齐次坐标**：用 3×3（2D）/4×4（3D）矩阵统一平移/旋转/缩放/透视。
- **乘法顺序**（右乘向量列向量记法）：`v_world = M_world * M_local * v_local`。组合时**右边先作用**：`M = T * R * S` 表示先缩放 S，再旋转 R，再平移 T。行/列主序仅是存储布局；**语义以乘法约定为准**。
- **2D 基本矩阵**
  ```
  平移 T(tx,ty) = |1 0 tx|
                  |0 1 ty|
                  |0 0  1|
  旋转 R(θ)     = | cosθ -sinθ 0|
                  | sinθ  cosθ 0|
                  |  0      0  1|
  缩放 S(sx,sy)  = |sx 0 0|
                  |0 sy 0|
                  |0  0 1|
  ```
- **3D 世界-观察-投影**：`clip = P * V * M * pos`；注意左手/右手坐标系（WebGL 右手，depth 默认 0..1）。**列主序 + 列向量**是 GL 常用习惯（many libs 按此）。
- **gimbal lock**：欧拉角易锁死；用**四元数**累乘旋转以保持数值稳定。

---

## 4) 栅格化与抗锯齿：MSAA/FXAA/TAA
- **MSAA**（多重采样抗锯齿）：几何边界处多采样**深度/覆盖**，片元着色仍执行一次，混合边缘。优点几何边缘质量高；代价：**多倍颜色/深度缓冲**，显存/带宽↑；对**透明/非几何 aliasing** 效果一般。
- **FXAA/SMAA**（屏幕空间后处理）：检测亮度边缘**模糊近邻**，成本低、易集成；对高对比度边缘有效，但可能软化细节。
- **TAA**（时间抗锯齿）：帧间**抖动采样（jitter）+ 历史重投影**。几何/着色 aliasing 均有效；代价：**拖影/鬼影**（需速度场/Clamp），对动画 UI 需调参。
- **应用建议**：移动端优先 **FXAA/SMAA**；中高端 PC/主机：**MSAA（如果 forward）+ TAA**；延迟渲染多数采用 **TAA**。

---

## 5) 渲染管线与 WebGPU 新能力
- **顶点着色器（VS）**：对象空间→裁剪空间（`MVP`），输出插值变量（`varying`）。常做：骨骼变换、Morph、顶点动画。
- **片元着色器（FS）**：采样纹理、计算光照、输出颜色/深度。可使用舍入/抖动、写入多渲染目标（MRT）。
- **固定功能阶段**：装配→光栅化→插值→深度/模板/混合。
- **WebGPU 新点**：  
  - **资源绑定**：Bind Group/Bind Group Layout，**显式**而稳定，便于缓存/管线复用。  
  - **渲染/计算 Pass**：图形与计算统一；**计算着色器**可做粒子、后处理、剔除、BVH、图像卷积等。  
  - **更严格的同步与生命周期**：避免隐式状态机陷阱；多队列/并行更高效。  
  - **WGSL**：现代着色语言，语义清晰，跨平台一致性更好。
- **GLSL 例（简化）**
  ```glsl
  // VS
  attribute vec3 pos; attribute vec3 nrm; uniform mat4 MVP, M;
  varying vec3 vN;
  void main() { gl_Position = MVP * vec4(pos,1.0); vN = mat3(M) * nrm; }

  // FS
  precision mediump float;
  varying vec3 vN; uniform vec3 L;
  void main() { float NdotL = max(dot(normalize(vN), normalize(L)), 0.0);
                 gl_FragColor = vec4(vec3(NdotL), 1.0); }
  ```

---

## 6) 光照模型：Lambert/Phong/Blinn-Phong → PBR
- **Lambert**：漫反射 `L_d = k_d * N·L`。简单、能量不守恒（取决于 k_d）。
- **Phong/Blinn-Phong**：镜面项 `L_s ≈ k_s * (R·V)^n` 或 `k_s * (N·H)^n`（H 为半程向量）；高光“假”，并非物理可积；但成本低。
- **PBR（基于物理）**：**Cook–Torrance Microfacet** 模型  
  `f = (D · F · G) / (4 (N·L)(N·V))`  
  - **D**：法线分布函数（GGX/Trowbridge-Reitz）  
  - **F**：Fresnel（Schlick 近似：`F = F0 + (1 - F0)(1 - V·H)^5`）  
  - **G**：几何遮蔽（Smith-GGX）  
  - **能量守恒**：金属（`metallic`）→ 反射高、无漫反；非金属（`dielectric`）→ `F0≈0.04`，保留漫反。
- **IBL**：环境辐照（diffuse irradiance）+ **预滤波**反射（Roughness Mip Chain）+ **BRDF LUT**，支撑实时 PBR。
- **选用**：移动端/卡通/低成本 → Phong/Blinn-Phong；写实/统一材质与美术管线 → **PBR**。

---

## 7) 阴影：Shadow Mapping/PCF/PCSS/CSM
- **Shadow Mapping**：从光源视角渲染深度图 `depth_light`；主摄像机渲染时将片元位置变换到光源裁剪空间，比较深度**被挡即阴影**。  
  问题：**走样（锯齿/游走）**、**彼得潘效应**、**漏光**。
- **Bias**：深度比较加入常量/斜率偏移（`polygonOffset` 或手动偏移）避免自遮挡，但过大产生“漂浮”。
- **PCF**：对阴影贴图邻域多次采样平均，软化边缘；代价：采样数↑。
- **PCSS**：先估计 blocker 深度，再按**半影半径**自适应采样，模拟软阴影；代价更大，需优化（Poisson Disk、分级 PCF）。
- **CSM（级联阴影）**：将摄像机视椎体按深度划分多个区间，每个区间单独用较高分辨率的光源贴图；解决远处分辨率不足。代价：**多次渲染**阴影贴图，分割/拼接与过渡需处理。
- **何时用**：室外阳光 → CSM+PCF；室内点光/聚光 → 阴影立方体贴图/聚光贴图 + PCF；对性能敏感 → 小核 PCF 或接近场仅阴影。

---

## 8) 前向 / 延迟 / Forward+ / Clustered 渲染
- **前向（Forward）**：逐物体着色，着色器内循环光源。简单、透明支持好、MSAA 易用；多光源昂贵（每像素遍历全部/可见光源）。
- **延迟（Deferred）**：先写 G-Buffer（法线/漫反/金属度/粗糙度/深度…），再逐像素累加光照。多光源高效、光体积裁剪好；局限：**透明/混合**难处理、内存带宽大、MSAA 复杂、材质分支受限。
- **Forward+ / Clustered**：将屏幕或视锥分成 Tile/Cluster（3D 划分），预计算每个区块的**可见光源列表**，前向渲染但像素仅遍历局部光源。兼顾透明与多光源，内存/实现复杂度较 Deferred 更低但高于 Forward。
- **建议**：移动端/中小型 → Forward/Forward+；大场景/多动光源 → Clustered/Deferred + 透明单独 Forward 通道。

---

## 9) 颜色管理：sRGB/Linear、HDR、Tone Mapping
- **Gamma 与线性空间**：人眼对亮度**非线性**；贴图/输出通常在 **sRGB**。**着色/混合**必须在**线性空间**计算：  
  `linear = srgb ^ 2.2`（近似），输出前再转回 sRGB。错用会导致**能量错误**（叠加偏暗/偏亮）。
- **工程落地（three.js）**  
  ```js
  renderer.outputColorSpace = THREE.SRGBColorSpace; // 输出转 sRGB
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  texture.colorSpace = THREE.SRGBColorSpace; // 仅“颜色贴图”设 sRGB；法线/粗糙度等保持线性
  ```
- **HDR 与 Tone Mapping**：在高动态范围（浮点帧缓冲）累加光照，最后用 **映射算子**（Reinhard/ACES/Filmic/Hable）压到显示器范围；同时配合**曝光**（`exposure`）。
- **预乘 Alpha 与颜色空间**：预乘应在**线性空间**进行，否则边缘会出现光晕/色偏。

---

## 10) 深度/模板缓冲与 Z-fighting
- **深度缓冲**：保存最靠近摄像机的深度。固定函数测试：`if (z_in ≤ z_buf) pass`，并可写入。深度值为**非线性**（透视除法后映射到 0..1，近端密、远端疏）。
- **精度与 Z-fighting**（两面几何深度太近导致闪烁条纹）
  - 近远平面设置过宽（`zFar/zNear` 太大）→ 近端精度挤压；  
  - 大尺度场景叠片共面。
- **缓解**  
  1) **收紧** `zNear`：尽量大些（如 0.1→0.5），`zFar` 尽量小；  
  2) **Polygon Offset**：`gl.polygonOffset(factor, units)` 或 three.js `material.polygonOffset = true; polygonOffsetFactor/Units`；  
  3) **Reversed Z + 浮点深度**：将深度范围反转（近=1 远=0）并使用 `GL_GREATER` 测试，配合浮点深度纹理，整体精度分布更均衡（WebGL 限制较多，WebGPU 更易实施）；  
  4) **LogDepth**：对巨场景用对数深度映射（需自定义着色器，可能与阴影/后处理交互复杂）；  
  5) **剔除/合并共面**：避免共面叠层，或在几何上做微小偏移。
- **模板缓冲（Stencil）**：像素级掩码，常用于**描边/镜面/门户**等需要“选择性绘制”的效果；注意与深度写入/排序配合避免漏写。

---
## 11) 透明与混合：方程、预乘 Alpha 与“白边/暗边”修复
- **混合方程（默认）**  
  `C_out = C_src * F_src + C_dst * F_dst`，`A_out = A_src * F_srcA + A_dst * F_dstA`。  
  常见设置（非预乘图像）：`glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)`。  
- **预乘 Alpha（premultiplied）**  
  纹理像素已满足 `rgb_premul = rgb * a`。混合应改为：`glBlendFunc(GL_ONE, GL_ONE_MINUS_SRC_ALPHA)`。  
  好处：边缘不易出现“白边/暗边”（来自未乘 alpha 的背景色泄漏），多重合成更稳定。  
- **素材侧修复**  
  - 导出 PNG/WEBP **预乘**通道或在加载时 `texture.premultiplyAlpha = true`；  
  - 贴图边缘做**扩边（bleed）**，把不透明区域颜色向外扩散；  
  - 对半透明 UI，尽量避免将**浅底色**与半透明像素一并烘焙。  
- **代码侧修复（WebGL/three.js）**
  ```js
  // WebGL
  gl.enable(gl.BLEND);
  // 非预乘图像：
  gl.blendFuncSeparate(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA, gl.ONE, gl.ONE_MINUS_SRC_ALPHA);
  // 预乘图像：
  gl.blendFuncSeparate(gl.ONE, gl.ONE_MINUS_SRC_ALPHA, gl.ONE, gl.ONE_MINUS_SRC_ALPHA);

  // three.js
  const renderer = new THREE.WebGLRenderer({ premultipliedAlpha: true });
  const tex = new THREE.Texture(img);
  tex.premultiplyAlpha = true;           // 素材为预乘
  material.transparent = true;
  material.blending = THREE.NormalBlending;  // three 内部会据 premultiply 选对方程
  ```
- **常见故障定位**  
  1) 观察缩放时边缘是否变色 → 多为**未预乘**或**扩边不足**；  
  2) 叠加多层 UI 产生“发灰” → 次序/混合/写深度冲突；关闭 `depthWrite` 或拆通道绘制。

---

## 12) 帧缓冲对象（FBO）：离屏渲染、MRT 与后处理链
- **离屏渲染**：把颜色/深度附件绑定到 FBO，先渲到纹理，再作为输入做后效（FXAA、Bloom、色调映射）。
  ```js
  // WebGL2 基本 FBO
  const fb = gl.createFramebuffer(); gl.bindFramebuffer(gl.FRAMEBUFFER, fb);
  // 颜色附件（RGBA16F 需 EXT_color_buffer_float）
  const color = gl.createTexture(); gl.bindTexture(gl.TEXTURE_2D, color);
  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA16F, w, h, 0, gl.RGBA, gl.HALF_FLOAT, null);
  gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, color, 0);
  ```
- **MRT（多目标渲染）**：WebGL2 的 `gl.drawBuffers([COLOR_ATTACHMENT0..N])` 同时输出法线/亮度等，便于延迟/后效链。  
- **后处理顺序**（典型）：  
  1) 主渲到 HDR 纹理（RGBA16F）；  
  2) **亮度提取** → 多级模糊（Bloom，半/四分之一分辨率，Ping-Pong）→ 叠加；  
  3) FXAA/SMAA（最后一道或在 ToneMapping 前后比较差异）；  
  4) Tone Mapping + 曝光 + Gamma/sRGB 输出。  
- **three.js**  
  ```js
  const composer = new EffectComposer(renderer);
  composer.addPass(new RenderPass(scene, camera));
  composer.addPass(new UnrealBloomPass(new THREE.Vector2(w,h), 1.0, 0.1, 0.9));
  composer.addPass(new ShaderPass(FXAAShader));
  composer.render();
  ```
- **性能要点**  
  - 复用 RenderTarget，避免每帧分配；  
  - Bloom/模糊用降分辨率；  
  - 若用 TAA/历史帧，不要在链中**改变视口尺寸**；  
  - 移动端尽量少用浮点附件，或者使用 R11F_G11F_B10F。

---

## 13) 相机/投影：矩阵推导、裁剪与 TAA 抖动
- **透视矩阵（GL 约定，NDC z ∈ [-1,1]）**  
  记 `t = tan(fovY/2)`，宽高比 `a`，近远面 `n,f`：
  ```
  | 1/(a t)   0        0               0 |
  |   0     1/t        0               0 |
  |   0       0   (f+n)/(n-f)   (2 f n)/(n-f) |
  |   0       0       -1               0 |
  ```
  正交投影将 x/y/z 线性映射到 [-1,1]。
- **视椎体裁剪（Frustum Culling）**：由 `P*V` 提取 6 平面，物体包围体（球/盒）与平面半空间测试剔除。  
- **TAA 抖动（Jitter）**  
  - 在像素网格上添加**子像素偏移**，把偏移量加进投影矩阵：`P[2][0]+=dx, P[2][1]+=dy`（列主序则相应位置）。  
  - 保存**历史帧**，用**速度向量**或上一帧 `MVP` 做重投影融合；  
  - 配合**clamp**（邻域最大最小）与**Catmull-Rom**重建减少拖影；UI/文本层可**跳过 TAA**以防糊。

---

## 14) 光照：Lambert/Phong/Blinn-Phong → PBR 取舍
- **Lambert**：只算漫反，适合卡通/低成本。  
- **Phong/Blinn-Phong**：添加高光，参数直观（`shininess`）；但**能量不守恒**，观察角变化不真实。  
- **PBR（Cook–Torrance）**：  
  - 以**金属度/粗糙度**为主参数；  
  - 环境光用 **IBL**（prefiltered env + BRDF LUT）；  
  - 统一材质表现，美术与 DCC 工具/glTF 生态兼容。  
- **取舍**：移动端/大量半透明 UI → Blinn-Phong；写实资产/一致管线 → PBR。两者可**混合**：天空盒/反射仍用 IBL，主光走简化模型。

---

## 15) 阴影基础：Shadow Map、PCF/PCSS、彼得潘效应与 bias
- **流程**：光相机渲深度贴图 `D_light` → 主相机绘制时，将片元点变换到光裁剪空间，比较 `z_light` 与 `D_light`。  
- **伪阴影/彼得潘**：过大 bias 让阴影与接触面分离（仿佛漂浮）。  
- **漏光/自遮挡**：bias 太小或法线不一致导致“阴影条纹”。  
- **抗锯齿**  
  - **PCF**：邻域多采样平均；样本可用 Poisson Disk；  
  - **PCSS**：先算 blocker 平均深度，再按**半影**半径自适应 PCF 核。  
- **级联（CSM）**：远处分割为多级，近处高分辨率，过渡使用 blend/覆盖。  
- **three.js 调参**  
  ```js
  light.castShadow = true;
  light.shadow.mapSize.set(2048, 2048);
  light.shadow.bias = -0.0005;      // 常量偏移
  light.shadow.normalBias = 0.5;    // 基于表面法线的偏移，减少自遮挡
  // 调整光相机（定向光用正交相机）包围盒，收紧范围提高 texel 密度：
  light.shadow.camera.left = -20; light.shadow.camera.right = 20;
  light.shadow.camera.top = 20;  light.shadow.camera.bottom = -20;
  ```
  建议：**先收紧相机体积** → 再微调 `bias/normalBias` → 需要软阴影时启用 `PCFSoftShadowMap`。

---

## 16) Three.js 场景图：矩阵更新、层级合并与视锥裁剪
- **矩阵链**：每个 `Object3D` 拥有 `matrix`（local）与 `matrixWorld`。  
  - 修改 `position/quaternion/scale` 会置 `matrixWorldNeedsUpdate = true`；  
  - `scene.updateMatrixWorld()` 在渲染前自顶向下更新。  
- **合并与实例化**  
  - 静态多物体可 **merge** 成一个 `BufferGeometry` 以减少 draw call；  
  - 大量同形体用 `InstancedMesh`，每实例用 `instanceMatrix/instanceColor`。  
  - 合并代价：失去独立剔除/拾取；可按**分块**合并折中。  
- **视锥裁剪**  
  - 默认 `mesh.frustumCulled = true`，利用 `geometry.boundingSphere/Box` 测试；  
  - 蒙皮网格/动画几何需**动态更新包围体**，否则会被误裁剪。  
- **层/可见性**  
  - `object.visible` 快速关断；`layers` 可为多相机/多通道渲染划分集合。

---

## 17) Three.js 材质：基础→PBR 与颜色空间
- **材质参与光照程度**  
  - `MeshBasicMaterial`：**不参与**光照；  
  - `MeshLambertMaterial`：Lambert 漫反，顶点着色求光（廉价）；  
  - `MeshPhongMaterial`：Blinn-Phong 高光（中等成本）；  
  - `MeshStandardMaterial`：PBR（金属度/粗糙度）；  
  - `MeshPhysicalMaterial`：在 `Standard` 上加入清漆、透光、折射等。  
- **PBR 贴图通道**（glTF 约定）  
  - baseColor（**sRGB**），metallicRoughness（G/B 通道，**Linear**），normal（**Linear**），occlusion（R），emissive（**sRGB**）。  
- **颜色空间与 Gamma**（three r150+）  
  ```js
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  texture.colorSpace = THREE.SRGBColorSpace;          // 仅“颜色贴图”
  normalMap.colorSpace = THREE.NoColorSpace;          // 线性
  material.toneMapped = true;                         // UI 等可设 false
  ```
- **环境/反射**  
  - IBL：`PMREMGenerator` 预滤环境贴图，`envMapIntensity` 控强度；  
  - AO 需要 `uv2`，由 `BufferGeometryUtils.mergeVertices` 后 `setAttribute('uv2', …)` 或在导入时确保第二套 UV。

---

## 18) Three.js 几何与 BufferGeometry：属性、索引、切线与压缩
- **关键属性**  
  - `position`（vec3）、`normal`（vec3）、`uv`（vec2）、`color`、`tangent`、`skinIndex/skinWeight`、`index`（复用顶点）。  
- **切线（tangent）**  
  - 法线贴图需要切线空间（TBN）。优先使用资产自带切线；否则使用**MikkTSpace**计算（three/examples 提供）。  
  - 若没有切线，可退化为**世界/视图空间法线贴图**或在 shader 内近似重建。  
- **合并与去重**  
  ```js
  import { mergeBufferGeometries, mergeVertices } from 'three/examples/jsm/utils/BufferGeometryUtils.js';
  const merged = mergeBufferGeometries([g1, g2, g3], true);
  const dedup = mergeVertices(merged);                 // 生成更紧凑索引
  ```
- **压缩**  
  - **Draco**：网格几何压缩，运行时解码；  
  - **KTX2/BasisU**：纹理压缩（见下一题）；  
  - 合理顺序：**加载→解压→生成切线/二套 UV→合批**。切线计算应在**解压后**进行。

---

## 19) Three.js 纹理加载与缓存：KTX2/采样差异与常见坑
- **加载与缓存**
  ```js
  const loader = new THREE.TextureLoader();           // 普通图片
  const ktx2 = new KTX2Loader().setTranscoderPath('/basis/').detectSupport(renderer); // 压缩纹理
  const tex = await loader.loadAsync('albedo.png');
  tex.colorSpace = THREE.SRGBColorSpace;
  tex.anisotropy = renderer.capabilities.getMaxAnisotropy();
  ```
  three 的 `LoadingManager` 会做去重缓存；可自建 LRU 缓存控制内存峰值。
- **KTX2/BasisU**：一次打包，多平台在运行时转码为 ASTC/BC/ETC。优点：更小**下载体积**与**显存占用**，含 mipmap。  
- **`flipY` 与来源**  
  - HTML 图像默认 `flipY = true`；glTF 导入的纹理通常 `flipY = false`（加载器处理）。若贴图倒置/法线奇怪，先检查 `flipY` 与 UV。  
- **颜色空间**  
  - **只有**颜色贴图（diffuse/baseColor/emissive）设 sRGB；其余（法线/金属粗糙/ao）保持线性。  
- **采样差异/兼容**  
  - NPOT 纹理在 `REPEAT`/`MIPMAP` 下受限（需 POT 或 WebGL2）；  
  - iOS Safari 对浮点纹理、sRGB 采样要求严格，优先 KTX2；  
  - 设置 `generateMipmaps = true`（有完整尺寸）并选择 `LinearMipmapLinearFilter`，对法线贴图保留 `LinearFilter`。

---

## 20) Three.js 阴影与灯光：类型与调优
- **灯光类型**  
  - `DirectionalLight`：平行光，用**正交相机**投影阴影贴图；适合太阳光，可配合级联（CSM 示例）。  
  - `SpotLight`：聚光，用**透视相机**生成圆锥阴影；  
  - `PointLight`：点光，**立方体阴影贴图（六面）**，开销大。  
- **质量与成本**  
  - `shadow.mapSize`：分辨率越高越清晰但 fillrate/显存↑；  
  - `renderer.shadowMap.type`：`PCFSoftShadowMap` 平滑但成本更高；  
  - 多灯投影会**重复渲染**场景阴影通道，成本线性增长。  
- **关键调参**  
  ```js
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;

  light.castShadow = true;
  light.shadow.mapSize.set(2048, 2048);
  light.shadow.bias = -0.0003;        // 减自遮挡条纹
  light.shadow.normalBias = 0.4;      // 依据法线偏移，减少 acne

  // 定向光收紧相机盒以提高 texel 密度
  const cam = light.shadow.camera;    // OrthographicCamera
  cam.left=-30; cam.right=30; cam.top=30; cam.bottom=-30;
  cam.near=0.5; cam.far=200; cam.updateProjectionMatrix();

  // 只让需要的物体投/受阴影
  mesh.castShadow = true; mesh.receiveShadow = true;
  ```
- **排错清单**  
  - 阴影块状/模糊：提高 `mapSize`、收紧相机盒；  
  - 漂浮/缺口：减小 `bias` 的绝对值、增大 `normalBias`；  
  - 性能突刺：检查是否对**静态物体**每帧都在更新阴影贴图（可冻结或降低刷新频率）。

---
## 21) Three.js 动画系统：Clip/Track/Mixer/Action、混合与 root motion
- **对象关系**
  - `AnimationClip`：一段可播放的动画数据，包含若干 `KeyframeTrack`。
  - `KeyframeTrack`：目标绑定 + 时间序列（`VectorKeyframeTrack`、`QuaternionKeyframeTrack`、`NumberKeyframeTrack`、`ColorKeyframeTrack`…）。绑定字符串由 `PropertyBinding` 解析，如：`'Armature.Hips.position'`、`'.morphTargetInfluences[3]'`。
  - `AnimationMixer`：驱动器，管理多个目标的播放状态。
  - `AnimationAction`：对某个 `Clip` 的一次实例化，提供 `play/pause/stop`、`setLoop`、`setEffectiveWeight`、`fadeIn/fadeOut/crossFadeFrom`、`setEffectiveTimeScale` 等。
- **混合与过渡**
  - **叠加**：同一目标的不同 `Action` 可按 **weight** 混合；同一通道（如旋转）默认线性插值，四元数走**球面插值**。
  - **过渡**：`a.crossFadeFrom(b, duration, warp)`；`warp=true` 会拉伸时间轴对齐步态节奏。
  - **Additive**：对“姿势修饰”类动画（抬手、眨眼），先用 `AnimationUtils.makeClipAdditive(clip)` 生成**增量**，再叠加到基态。
- **root motion**
  - 角色骨架常以“Hip/Root”携带位移。两种做法：
    1) **原位播放**：把 `Hip.position` 的增量采样出来，**从骨架里减去**，再把增量施加到 `mesh.position`（角色移动由外层物体驱动）。
    2) **跟随式**：直接使用骨架位移（更还原，但导航/碰撞要同步）。
  - 采样示例（每帧）：
    ```js
    const prev = hip.getWorldPosition(_p0), prevQ = hip.getWorldQuaternion(_q0);
    mixer.update(dt);
    const now = hip.getWorldPosition(_p1), nowQ = hip.getWorldQuaternion(_q1);
    const delta = now.clone().sub(prev); // 帧位移
    character.position.add(delta);       // 外层位移
    hip.position.sub(delta.applyMatrix4(character.matrixWorld.clone().invert())); // 从骨架抵消
    ```
- **实践建议**：将“循环跑步”等设为 **base action**，姿势/表情用 **additive**；停/走/跑用交叉淡入淡出，`warp` 保节拍。

---

## 22) 拾取（Raycaster vs 颜色编码）：坐标换算、InstancedMesh、工程取舍
- **Raycaster（几何级）**
  - 屏幕 → NDC：  
    ```js
    const rect = renderer.domElement.getBoundingClientRect();
    mouse.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const hits = raycaster.intersectObjects(scene.children, true);
    ```
  - `InstancedMesh` 支持：命中项含 `instanceId`，可从 `getMatrixAt(id, m)` 取实例变换。
  - **优点**：无需二次渲染；能拿到精确三角信息/UV/法线。**缺点**：对巨型网格/海量实例，CPU 侧 BVH 重要（`three-mesh-bvh` 可把三角检测降到 `O(log n)`）。
- **颜色编码（offscreen picking）**
  - 离屏 FBO 渲染场景，每物体（或实例）写入唯一 `RGB` id（禁用 `toneMapping/gamma/dithering`，深度测试保留）。读回鼠标像素 → 反解 id。
  - **优点**：对海量小物体更稳定（不依赖 CPU 递归与三角求交）；对**透明**可做特殊通道。**缺点**：需额外渲染一次；MSAA 处理、预乘 Alpha 需注意。
- **建议**：静态大网格（建筑/地形）→ **Raycaster + BVH**；大量图元（节点/标记）→ **颜色拾取** 或 **实例 id 缓冲**。

---

## 23) 后处理管线：EffectComposer、Pass 协议与多分辨率
- **数据流**：`RenderPass(scene,camera)` 产出 `readBuffer` → 若 `Pass.needsSwap=true`，交换至 `writeBuffer`；后续 `ShaderPass` 读取 `tDiffuse`，链路末端输出到默认帧缓冲。
- **自定义 Pass 协议**
  ```js
  class MyPass extends Pass {
    constructor() { super(); this.fsQuad = new FullScreenQuad(myMaterial); }
    render(renderer, write, read/*THREE.WebGLRenderTarget*/, delta/*s*/) {
      this.material.uniforms.tDiffuse.value = read.texture;
      if (this.renderToScreen) renderer.setRenderTarget(null);
      else renderer.setRenderTarget(write), this.needsSwap = true;
      this.fsQuad.render(renderer);
    }
    setSize(w,h){ this.material.uniforms.res.value.set(w,h); }
  }
  ```
- **多分辨率**：昂贵 Pass（Bloom/SSR/SSAO）在**降分辨率**下运行（1/2 或 1/4），再**上采样**合成；用 **Ping-Pong** 纹理避免频繁分配。
- **排序建议**：几何 → 阴影 → 主渲 → 高亮提取/模糊 → SSR/SSAO → TAA/FXAA → ToneMapping → UI 合成。TAA 与 FXAA 顺序按效果取舍。

---

## 24) 性能工程：draw call/状态切换/过度绘制三角权衡
- **draw call**：CPU→GPU 提交的开销。优化手段：
  - **合批**：几何合并（静态）或 `InstancedMesh`（动态同形体）；粒子用 `Points` 或 `InstancedMesh`。
  - **材质去重**：共享 `Material` 与 `Program`；动态参数走 `uniform` 或 `onBeforeCompile` 注入宏，避免生成大量材质变体。
- **状态切换**：材质/纹理切换会刷新 GPU 管线。three 会把 **不透明**对象按**程序/材质/深度从近到远**排序；透明对象**从远到近**。
- **过度绘制（fillrate）**：多层透明/大面积半透明对移动 Tiling GPU 极其昂贵。策略：使用 `alphaTest` 做 **cutout**；或把 UI/雾片等**降分辨率渲染**再合成。
- **顶点 vs 填充率**：移动端通常**填充率瓶颈**更多；桌面端复杂曲面/细分更容易**顶点瓶颈**。用 GPU 时间戳/管线统计确认热点。
- **额外**：为射线/包围体计算引入 `three-mesh-bvh`；LOD（基于距离/屏幕像素覆盖）切换时做**淡入**减少跳变。

---

## 25) 透明渲染与排序：renderOrder/depthTest/depthWrite、双通道与 OIT
- **基本规则**
  - 不透明先渲、从近到远写深度；透明后渲、从远到近，不写深度（`depthWrite=false`）通常更安全。
  - UI/粒子：`depthTest=false`（或 `depthFunc=ALWAYS`）+ `depthWrite=false`，并设置高 `renderOrder`。
- **常见故障**
  - 玻璃互穿/排序错乱：无法稳定拓扑排序 → 方案：**Weighted Blended OIT**（加权平均 OIT）或拆分**双通道**（前后表面各一次）。
  - 透明遮蔽阴影：透明材质应**不接收阴影**，或在阴影通道使用 `alphaTest` 替代混合。
- **工程套路**
  - **Cutout**（`alphaTest`）优先：叶子/栅栏这类“有洞”但边界清晰的对象。
  - **Dual pass**：先 opaque（厚度/折射近似），再 transparent（表面高光/贴花）。
  - **排序钩子**：设置 `object.renderOrder` 手工干预；慎用过多自定义排序以免打破 three 内部规则。

---

## 26) glTF 2.0 管线：结构与 Draco/KTX2 压缩
- **核心结构**
  - `scenes` → `nodes`（层级/变换/相机/skin）→ `meshes`（`primitives[]`）→ `accessors/bufferViews/buffers`（二进制数据）。
  - 材质：PBR Metallic-Roughness；扩展如 `KHR_materials_transmission/clearcoat/ior/volume`。
  - 动画：`samplers`（time/value 插值）+ `channels`（目标路径：`translation/rotation/scale/weights`）。
- **几何压缩：Draco**
  - 通过 `KHR_draco_mesh_compression` 附加属性；构建期用 gltf-pipeline/gltf-transform 开启，运行时 `DRACOLoader`（Worker）解码，传回 `BufferGeometry`。
- **纹理压缩：KTX2/BasisU**
  - 构建期生成 `.ktx2`（含多 mip），运行时 `KTX2Loader` → 转码为 ASTC/BC/ETC。
  - 优先给**彩色贴图** sRGB，法线/金属粗糙线性。
- **运行时落地**
  ```js
  const draco = new DRACOLoader().setDecoderPath('/draco/');
  const ktx2  = new KTX2Loader().setTranscoderPath('/basis/').detectSupport(renderer);
  const gltfLoader = new GLTFLoader().setDRACOLoader(draco).setKTX2Loader(ktx2);
  const { scene, animations } = await gltfLoader.loadAsync('/model.glb');
  ```
- **建议**：先 KTX2 降带宽/显存，再 Draco 减下载体积；对**小网格**（顶点很少）不开 Draco（头部占比高）。

---

## 27) 骨骼蒙皮与 Morph：公式、次序与重定向
- **骨骼蒙皮（线性混合，LBS）**
  - 顶点在绑定时的基准为 `bindPose`。最终位置：  
    `p' = Σ_i w_i * (M_i * B_i^{-1}) * p`  
    其中 `M_i` 为骨骼当前矩阵，`B_i^{-1}` 为逆绑定矩阵。法线使用上式的**3×3 逆转置**（或在 shader 用 `mat3(normalMatrix)`）。
- **Morph Targets（形变）**
  - 顶点位置/法线作为**偏移量**存于多个目标，按 `weights[]` 线性组合：  
    `p' = p + Σ_k w_k * Δp_k`。  
  - **次序**：three 的默认顺序是 **Morph → Skin**（先做形变再随骨骼变换），与 DCC/glTF 一致。
- **Retarget（动画重定向）**
  1) 建立**骨骼映射**（按名字或拓扑）；  
  2) 计算源模型在**绑定姿势**到**目标绑定姿势**的**相对旋转偏置**；  
  3) 每帧把源骨旋转乘以偏置，写入目标；根骨平移/朝向可单独处理（配合 root motion）。  
  - 对非等比例骨骼，需使用**dual quaternion skinning** 或在离线阶段做长度归一化。

---

## 28) 资源生命周期：并发、缓存、引用计数与泄漏定位
- **并发加载与解压**
  - 几何/纹理解压放 **Worker**（DracoLoader/KTX2Loader 默认如此）；限制并发避免主线程卡顿（`LoadingManager.setMaxConnections`）。
- **缓存**
  - three 的 `Cache`/`LoadingManager` 会对同 URL 去重；更细粒度可自建 **LRU**（键为 URL+参数）。
- **释放**
  ```js
  geometry.dispose();             // VAO/VBO 释放
  material.dispose();             // 程序/Uniform 缓存释放
  texture.dispose();              // 纹理/采样器释放
  renderer.renderLists.dispose(); // 可在场景切换时清理渲染队列
  ```
  - 递归释放：遍历 `scene`，对 `isMesh` 的 `geometry/material/texture` 计数-1 到 0 时调用 `dispose`。
- **显存估算**
  - 顶点：`vertexCount * (pos+norm+uv+...) * bytesPerComponent`；  
  - 纹理：`Σ(width*height*bytesPerPixel*(1+1/3 mip))`（压缩纹理按格式 bpp）；  
  - 监控：`renderer.info.memory`（对象/纹理/几何数量），配合 **Spector.js** 捕获帧看实际占用。
- **泄漏排查**：事件订阅未解绑、循环引用导致对象不被 GC、FBO/RT 未释放、频繁创建 `Material` 导致 program 缓存暴涨。

---

## 29) 交互映射：指针事件 → 射线/坐标、拖拽与 pointer capture
- **屏幕 → 世界**
  - 指针坐标 → NDC（见第 22 题）→ `raycaster.setFromCamera` → `intersectObjects`。
  - 拖拽：在按下时记录命中的**拾取平面**（例如与地面的交点），移动时与该平面求交得到位移增量。
- **pointer capture**
  - 在按下命中对象/画布后 `e.target.setPointerCapture(e.pointerId)`，即使指针移出画布也能持续收到事件；  
  - 释放：`releasePointerCapture` 在结束/取消时调用；**多指**需为每根手指管理独立状态。
- **移动端细节**
  - 使用 `pointer-events` 统一鼠标/触摸；禁用浏览器默认手势（`touch-action: none`）避免滚动/双指缩放干扰；  
  - 拖拽与**相机轨迹**要区分手势优先级（点击阈值/长按判定）。

---

## 30) 多视口与 UI 叠加：Viewport/Scissor、最小二值方案
- **多视口**：在同一 `canvas` 上绘制多个相机画面（小地图/PIP）
  ```js
  function renderView(x, y, w, h, camera, scene) {
    renderer.setViewport(x, y, w, h);
    renderer.setScissor(x, y, w, h);
    renderer.setScissorTest(true);
    renderer.render(scene, camera);
  }
  // 主视口
  renderView(0, 0, width, height, cameraMain, scene);
  // 右下角小地图
  const vw = Math.floor(width*0.25), vh = Math.floor(height*0.25);
  renderView(width-vw-8, 8, vw, vh, cameraOrthoMiniMap, sceneMini);
  ```
- **UI 叠加**
  - **HTML/CSS 叠加**：DOM 层做 HUD/菜单，3D Canvas 仅渲染图形（最简单、可达性友好）。  
  - **同画布二次渲染**：先 3D，再切 `depthTest=false` 渲 2D/HUD 场景；或使用 `Sprites`/`CSS2DRenderer/CSS3DRenderer`。
- **注意**
  - 开启 `scissorTest` 可以限制重绘区域减少 overdraw；  
  - 处理 DPR 时 `setViewport/Scissor` 传入**物理像素**尺寸；  
  - 后处理链如果作用于全屏，需要对每个视口单独渲染或将 HUD 合成在链路末端。

---
## 31) 移动端优化：Tiling GPU、带宽/填充率、分辨率与 AA 取舍
- **Tiling GPU 工作原理**  
  大多数移动 GPU（Mali/Adreno/PowerVR）按**瓦片**把屏幕切块，在片上缓存（on-chip）完成光栅+混合，再写回显存。好处是降低外部带宽；**破坏因素**：频繁读回、跨瓦片依赖、过多半透明导致多次写入。
- **三角权衡：draw call × 状态切换 × 过度绘制（fillrate）**  
  1) **draw call**：用**实例化**/几何合并减少 CPU→GPU 提交；  
  2) **状态切换**：合并材质/纹理，排序减少 program/binding 抖动；  
  3) **过度绘制**：移动端最常见瓶颈。减少覆盖层数（UI 雾片/粒子降分辨率渲染后合成），**前向从近到远**绘制不透明体，扩大早深度裁剪收益。
- **动态分辨率 / DPR 钳制**  
  ```js
  const maxDpr = 2; // iOS 高端可到 3，先从 2 做上限
  const dpr = Math.min(Math.ceil(window.devicePixelRatio || 1), maxDpr);
  renderer.setPixelRatio(dpr);
  renderer.setSize(w, h, false);
  ```
  在监控 GPU 时间（ext timer 或帧间滑动均值）超阈值时，**下调渲染分辨率**或**降低后处理质量**。
- **AA 选择**  
  - **FXAA/SMAA**（后处理）成本低，适用于移动端；  
  - **MSAA** 仅对几何边缘有效且耗显存/带宽，慎用；  
  - **TAA** 抗闪烁好，但历史采样 + 重投影成本高，移动端谨慎。
- **纹理压缩 / bpp 治理**  
  构建期输出 **KTX2**（BasisU）一份工件，运行时转码到 ETC2/ASTC/BC，显著降低带宽与显存。
- **避免“瓦片失配”操作**  
  - 大量 `discard`/复杂分支会降低 Early-Z；  
  - 过多半透明、逐像素混合会导致同瓦片多次写回；  
  - 避免 `preserveDrawingBuffer: true`；避免频繁 `readPixels`；**后处理链**用降分辨率 FBO + 复用 RenderTarget。
- **工程开关**  
  - `antialias: false`（配合 FXAA），`powerPreference: 'high-performance'`；  
  - 控制 `shadow.mapSize`、后处理 Pass 次数与分辨率；  
  - 贴图各向异性上限适度：`texture.anisotropy = Math.min(4, maxAniso)`。

---

## 32) WebGL 兼容与扩展：能力探测、降级策略与上下文丢失
- **能力探测（Feature probe）**
  ```js
  const gl = canvas.getContext('webgl2') || canvas.getContext('webgl');
  const has = (name) => !!gl.getExtension(name);
  const isWebGL2 = !!gl.drawBuffers; // 或 'WebGL2RenderingContext' in window
  const floatRT = isWebGL2
    ? has('EXT_color_buffer_float')
    : has('WEBGL_color_buffer_float') && has('OES_texture_float');
  const instancing = isWebGL2 || has('ANGLE_instanced_arrays');
  const uintIndex  = isWebGL2 || has('OES_element_index_uint');
  ```
  WebGL1 常见扩展：`OES_element_index_uint`（32 位索引）、`ANGLE_instanced_arrays`（实例化）、`EXT_sRGB`、`OES_texture_float[_linear]`、`WEBGL_depth_texture`。
- **降级矩阵**  
  - **Instancing**：无扩展时退化为分批 draw；  
  - **浮点渲染目标**：不可用则 HDR→LDR（8-bit）+ 曲线近似；  
  - **sRGB**：无扩展时手动 Gamma（线性 ↔ sRGB）在 shader 里转换；  
  - **VAO**：WebGL1 用 polyfill 或每帧绑定 attribute；  
  - **MRT**：WebGL1 无法原生，拆成多次渲染。
- **上下文丢失/恢复**  
  ```js
  canvas.addEventListener('webglcontextlost', (e)=>{ e.preventDefault(); paused = true; });
  canvas.addEventListener('webglcontextrestored', ()=>{ reinitGLResources(); paused = false; });
  ```
  保持**资源重建路径**（纹理/缓冲/程序），并缓存原始数据或可重载 URL。
- **调试与黑名单差异**  
  - `WEBGL_debug_renderer_info` 可判定厂商/型号（隐私敏感，谨慎使用）；  
  - iOS Safari 对 sRGB/浮点/立方体阴影更严格，优先使用 KTX2 压缩贴图与 WebGL2 路径。

---

## 33) 大规模点云 / 线框 / 体素：可视化与加速结构
- **点云（Points / Point Sprites）**
  - 顶点着色器设定 `gl_PointSize`，片元使用点精灵纹理（圆点/高斯核）模拟体积；  
  - 颜色/强度/分类作为 per-vertex attribute；  
  - **拾取**：颜色编码或基于 BVH 的最近点查询；  
  - **LOD/分块**：八叉树（Octree）或 Potree 格式按距离/屏幕覆盖渐进加载。
- **线条**  
  - WebGL 的 `lineWidth` 基本不可用；改为**屏幕空间四边形**法：将每条段扩展两个三角形，顶点着色器根据相机构建偏移向量（miter/join/cap），或使用 meshline 等实现。  
  - 大规模网络图：**实例化**每段线（属性：端点索引/粗细/颜色），在 VS 里解码端点坐标。
- **体素 / 体渲染**  
  - WebGL2 使用 3D 纹理 + 射线步进；对医学/科学数据可做**早终止**与**空洞跳跃**（空体素跳过）。  
  - 体素地形：**marching cubes** 在顶点着色器/计算阶段生成网格（WebGL 受限，WebGPU 更适合）。  
- **加速结构（Picking/Culling）**  
  - 三角网格：`BVH`（three-mesh-bvh）将三角求交从 O(n) 降到 O(log n)；  
  - 点/线：分块 + 屏幕空间裁剪（Frustum + 局部 AABB）；  
  - CPU 端预构建索引，GPU 端用纹理缓冲（WebGL2 `texelFetch`）读取。

---

## 34) 颜色管理与 HDR（three.js 实战）：线性工作流、Tone Mapping 与校准
- **线性工作流要点**  
  1) **输入**：仅**色彩贴图**（BaseColor/Albedo/Emissive）标记为 sRGB，其余（法线/金属/粗糙/AO）保持 Linear。  
  2) **处理中**：所有光照/混合在 Linear 进行；  
  3) **输出**：统一映射到显示色域（sRGB/Display P3），并做 **Tone Mapping** 与 Gamma。
- **three.js 配置（r150+）**
  ```js
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  renderer.toneMapping = THREE.ACESFilmicToneMapping; // 或 Filmic/Reinhard/None
  renderer.toneMappingExposure = 1.0;

  baseColorTex.colorSpace = THREE.SRGBColorSpace;
  normalMap.colorSpace    = THREE.NoColorSpace;
  material.toneMapped     = true; // UI 贴图可关闭
  ```
- **HDR 渲染链**  
  - 主渲到 **浮点 RT**（`RGBA16F`）→ 后处理（Bloom/SSR/SSAO）→ Tone Mapping → sRGB 输出；  
  - 无浮点 RT 时：用 LDR 替代，但注意高亮溢出与 banding。  
  - 轻微带状（banding）：开启材质 `dithering = true` 或在最终 Pass 做抖动。
- **校准与排错**  
  - 梯度测试纹理：观察是否有不均匀条纹（Gamma 错置/重复编码）；  
  - UI 偶发发灰：检查 UI Pass 是否被 Tone Mapping；  
  - 贴图曝光过亮/过暗：确认 BaseColor 是否错误标 sRGB；  
  - 多浏览器差异：确保 CSS 层无 `filter()` 干扰，Canvas 背景透过 `premultipliedAlpha` 正确配置。

---

## 35) WebGPU 入门与迁移：资源绑定、WGSL、渲染/计算 Pass 与迁移策略
- **核心概念**  
  - `GPUAdapter → GPUDevice`：能力与队列；  
  - **管线**：`GPURenderPipeline` / `GPUComputePipeline`，着色器语言为 **WGSL**；  
  - **资源绑定**：`GPUBindGroupLayout`（描述槽位）+ `GPUBindGroup`（绑定具体 buffer/texture/sampler）；  
  - **命令编码**：`GPUCommandEncoder` → `RenderPassEncoder/ComputePassEncoder` → `queue.submit()`
- **最小渲染闭环（伪代码）**
  ```js
  const adapter = await navigator.gpu.requestAdapter();
  const device = await adapter.requestDevice();
  const ctx = canvas.getContext('webgpu');
  ctx.configure({ device, format: navigator.gpu.getPreferredCanvasFormat() });

  const vs = /* wgsl */`
    @vertex fn main(@location(0) pos: vec2f) -> @builtin(position) vec4f {
      return vec4f(pos, 0.0, 1.0);
    }`;
  const fs = /* wgsl */`
    @fragment fn main() -> @location(0) vec4f { return vec4f(1,0,0,1); }`;

  const pipeline = device.createRenderPipeline({
    layout: 'auto',
    vertex: { module: device.createShaderModule({code:vs}), buffers:[{arrayStride:8, attributes:[{shaderLocation:0, offset:0, format:'float32x2'}]}]},
    fragment:{ module: device.createShaderModule({code:fs}), targets:[{ format: ctx.getCurrentTexture().format }] },
    primitive:{ topology:'triangle-list' }
  });

  const render = () => {
    const tex = ctx.getCurrentTexture();
    const view = tex.createView();
    const encoder = device.createCommandEncoder();
    const pass = encoder.beginRenderPass({ colorAttachments:[{ view, clearValue:{r:0,g:0,b:0,a:1}, loadOp:'clear', storeOp:'store' }] });
    pass.setPipeline(pipeline);
    pass.draw(3); // 画 fullscreen 三角形
    pass.end();
    device.queue.submit([encoder.finish()]);
  };
  render();
  ```
- **迁移策略（从 WebGL/Three）**  
  1) **资产与材质**：沿用 glTF/KTX2 资产，先**复现 PBR 着色**（Metallic-Roughness + IBL）；  
  2) **管线拆分**：把 WebGL 的隐式状态机改为**显式绑定/布局**，将 UBO/纹理归类到 BindGroup 以便复用；  
  3) **计算着色器**：把 culling、粒子更新、后处理卷积、Clustered Light 构建迁至 **Compute Pass**；  
  4) **同步与生命周期**：WebGPU 显式资源过期/重用更安全，编码阶段**批量提交**减少驱动开销；  
  5) **回退**：无 WebGPU 则退回 WebGL2 路径（能力探测 + 双管线封装）。  
- **性能基线**  
  - 合并小 Buffer 为大 Buffer，使用**动态偏移**；  
  - BindGroup/管线复用，减少 `setBindGroup`/`setPipeline` 频率；  
  - 渲染/计算分离提交，利用并行；  
  - 用 GPU 时间戳/Profiler（浏览器 DevTools/WebGPU 捕获）定位瓶颈。

---
