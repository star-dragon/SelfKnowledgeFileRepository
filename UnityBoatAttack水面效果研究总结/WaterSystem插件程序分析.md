# Boat Attack Water System — 程序分析文档

> 分析对象：`Assets/com.verasl.water-system`（包名 `com.verasl.water-system`，版本 `1.4.0-preview.1`）
> 入口文件：`Scripts/Water.cs`
> 分析日期：2026-10-30

---

## 0. 阅读指引

本文档按「**数据 → 逻辑 → 渲染 → Shader**」四层展开，重点回答三个问题：

1. 插件由哪些模块组成，彼此如何协作（第 2 章）
2. 水面几何与着色到底是怎么算出来的（第 5、9 章）
3. 接入时有哪些硬约束和已知坑（第 11、12 章）

如果你只想快速上手，看第 2 章架构图 + 第 11 章接入步骤即可。

---

## 1. 插件概览

### 1.1 基本信息

| 项目 | 值 |
|---|---|
| 包名 | `com.verasl.water-system` |
| 显示名 | Boat Attack Water System |
| 版本 | `1.4.0-preview.1` |
| Unity 版本 | 2019.4 |
| 渲染管线 | URP（Universal Render Pipeline）`7.4.1` |
| 依赖 | `com.unity.mathematics 1.1.0`、`com.unity.burst 1.2.3`、`com.unity.render-pipelines.universal 7.4.1` |

来源是 Unity 官方示例项目 **Boat Attack** 的水体系统，被拆成独立 UPM 包。定位是「中等规模开放水域」：Gerstner 波 + 深度衰减着色 + 平面反射 + 浮力，不含流体模拟、不含 Screen Space Reflection、不含 Foam 粒子系统（泡沫靠贴图与 shader 混合）。

### 1.2 目录结构

```
com.verasl.water-system/
├── Scripts/                      运行时代码（10 个 .cs）
│   ├── Water.cs                  ★ 主入口 / 单例 / 全局参数下发
│   ├── GerstnerWavesJobs.cs      CPU 侧 Gerstner 波（Job + Burst）
│   ├── BuoyantObject.cs          浮力与体素化
│   ├── LocalToWorldJob.cs        浮力采样点本地→世界坐标转换 Job
│   ├── MainCameraAlign.cs        水面网格跟随相机的量化对齐（备用组件）
│   ├── Data/
│   │   ├── WaterSettingsData.cs  渲染/反射设置 SO
│   │   ├── WaterSurfaceData.cs   水面外观设置 SO + Wave/BasicWaves/FoamSettings
│   │   └── WaterResources.cs     默认贴图/材质/网格 SO
│   └── Rendering/
│       ├── PlanarReflections.cs  平面反射相机
│       └── WaterSystemFeature.cs URP ScriptableRendererFeature（WaterFX + Caustics）
├── Shaders/                      6 个 .shader + 6 个 .hlsl
├── Editor/                       5 个编辑器脚本
├── Meshes/                       Water.fbx / SeaVertDisp.fbx / InfiniteSea.fbx
├── Materials/                    Sea.mat / InfiniteWater.mat / Caustics.mat
├── Textures/                     泡沫、水面法线、焦散序列
└── SeaVisual.prefab              预配置的水面预制体
```

### 1.3 代码规模

| 文件 | 行数 | 职责权重 |
|---|---|---|
| `Water.cs` | 402 | 核心 |
| `BuoyantObject.cs` | 411 | 核心（浮力） |
| `PlanarReflections.cs` | 294 | 核心（反射） |
| `WaterCommon.hlsl` | 301 | 核心（着色主体） |
| `WaterSystemFeature.cs` | 233 | 核心（管线集成） |
| `GerstnerWavesJobs.cs` | 206 | 核心（CPU 波） |
| `WaterSurfaceDataEditor.cs` | ~450 | 编辑器 UI |

---

## 2. 总体架构

### 2.1 分层视图

```
┌─────────────────────────────────────────────────────────────┐
│  数据层  ScriptableObject                                     │
│  WaterSettingsData(渲染/反射)  WaterSurfaceData(外观/波浪)     │
│  WaterResources(默认资源)                                     │
└───────────────────────┬─────────────────────────────────────┘
                        │ 读取
┌───────────────────────▼─────────────────────────────────────┐
│  逻辑层  Water.cs（单例, [ExecuteAlways]）                     │
│  · SetWaves()          波浪生成 + 全局 Shader 参数下发          │
│  · GenerateColorRamp() 吸收/散射/泡沫 Ramp 烘焙成 128×4 贴图    │
│  · CaptureDepthMap()   正交相机烘焙岸线深度图                   │
│  · BeginCameraRendering()  无限水面网格跟随相机绘制              │
└───────────────────────┬─────────────────────────────────────┘
                        │ Shader.SetGlobal*  (全局参数)
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌────────────┐ ┌────────────────────────┐
│ GPU 波浪      │ │ CPU 波浪    │ │ 渲染管线扩展             │
│ GerstnerWave │ │ GerstnerWav│ │ WaterSystemFeature      │
│ .hlsl        │ │ esJobs.cs  │ │  ├ WaterFxPass          │
│ (Structured  │ │ (Burst Job)│ │  └ WaterCausticsPass    │
│  Buffer/数组)│ │  → 浮力采样 │ │ PlanarReflections       │
└──────┬───────┘ └─────┬──────┘ └───────────┬────────────┘
       └───────────────┴────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  Shader 层                                                    │
│  Water.shader / WaterTessellated.shader                       │
│    └ WaterCommon.hlsl (WaveVertexOperations + WaterFragment)  │
│        ├ GerstnerWaves.hlsl   波浪位移与法线                   │
│        ├ WaterLighting.hlsl   菲涅尔/软阴影/反射               │
│        ├ WaterInput.hlsl      全局纹理与常量声明                │
│        └ CommonUtilities.hlsl Remap / HeightToNormal / noise   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 关键设计特点

1. **全局参数驱动**：`Water.cs` 通过 `Shader.SetGlobalTexture / SetGlobalFloat / SetGlobalBuffer / EnableKeyword` 下发几乎所有参数。Shader 不依赖材质面板，材质的作用仅是「挂载 shader + 队列」。这带来一个副作用：**场景里只能有一套水体配置**，多个 `Water` 实例会互相覆盖全局状态。

2. **波浪 CPU/GPU 双实现**：GPU 侧算顶点位移（渲染），CPU 侧用 Burst Job 算同一套 Gerstner 公式（浮力/物理查询）。两侧必须保持一致，但当前版本存在参数不一致（见 5.4）。

3. **无 MeshRenderer 依赖**：`Water.cs` 用 `Graphics.DrawMesh` 直接绘制，不要求挂载 Renderer（见 4.5）。

### 2.3 每帧时序

```
帧开始
 │
 ├─ RenderPipelineManager.beginCameraRendering
 │    ├─ PlanarReflections.ExecutePlanarReflections      （若启用平面反射）
 │    │     ├─ 构建镜像相机 + 斜投影矩阵
 │    │     ├─ UniversalRenderPipeline.RenderSingleCamera  → 反射 RT
 │    │     └─ Shader.SetGlobalTexture(_PlanarReflectionTexture)
 │    └─ Water.BeginCameraRendering
 │          ├─ SetGlobalFloat(_CameraRoll)
 │          ├─ SetGlobalMatrix(_InvViewProjection)
 │          └─ Graphics.DrawMesh × defaultWaterMeshes     （无限水面）
 │
 ├─ URP Pass 队列
 │    ├─ [BeforeRenderingOpaques] WaterFxPass  → _WaterFXMap（半分辨率）
 │    ├─ [AfterRenderingSkybox+1] WaterCausticsPass → 焦散叠加
 │    └─ [Transparent-100]       Water.shader  → 水面主体
 │
 ├─ Update()        BuoyantObject：更新采样点、读取波高
 ├─ FixedUpdate()   BuoyantObject：施加浮力
 ├─ LateUpdate()
 │    ├─ Water.LateUpdate       → GerstnerWavesJobs.UpdateHeights()（调度 Job）
 │    └─ BuoyantObject.LateUpdate → LocalToWorldConversion()（调度坐标转换 Job）
 └─ 帧结束
```

注意 Job 是**延迟一帧**的：`UpdateHeights` 在 LateUpdate 调度，`UpdateSamplePoints` 在下一帧 Update 里 `Complete()` 并取结果。

---

## 3. 数据层（ScriptableObject）

### 3.1 `WaterSettingsData` — 渲染设置

```csharp
public GeometryType waterGeomType;                 // VertexOffset | Tesselation
public ReflectionType refType = PlanarReflection;  // Cubemap | ReflectionProbe | PlanarReflection
public PlanarReflections.PlanarReflectionSettings planarSettings;
public Cubemap cubemapRefType;
public bool isInfinite;
public Vector4 originOffset;
```

`refType` 三个枚举直接映射到 shader 的全局关键字 `_REFLECTION_CUBEMAP` / `_REFLECTION_PROBES` / `_REFLECTION_PLANARREFLECTION`，由 `Water.SetWaves()` 三选一 Enable。

### 3.2 `WaterSurfaceData` — 外观与波浪

```csharp
public float _waterMaxVisibility = 40.0f;   // 最大可视水深 → _MaxDepth
public Gradient _absorptionRamp;            // 吸收颜色梯度
public Gradient _scatterRamp;               // 散射颜色梯度
public List<Wave> _waves;                   // 自定义波浪
public bool _customWaves = false;           // false → 用 BasicWaves 随机生成
public int randomSeed = 3234;
public BasicWaves _basicWaveSettings;       // numWaves/amplitude/direction/wavelength
public FoamSettings _foamSettings;
```

**`Wave` 结构体**（CPU/GPU 共用布局，24 字节）：

```csharp
public float amplitude;   // 波高（米）
public float direction;   // 方向，从 Z+ 起算的角度
public float wavelength;  // 峰到峰距离
public float2 origin;     // 全向波原点
public float onmiDir;     // 0=定向波, 1=全向波
```

24 字节正好对应 `sizeof(float) * 6`，与 `Water.cs` 里 ComputeBuffer 的 stride 一致。

**`FoamSettings`**：`foamType` 0=默认(用内置 ramp) / 1=简单(用 basicFoam 曲线重映射) / 2=自定义(全黑)。另有 `liteFoam` / `mediumFoam` / `denseFoam` 三条曲线。

### 3.3 `WaterResources` — 默认资源

```csharp
public Texture2D defaultFoamRamp;
public Texture2D defaultFoamMap;      // → _FoamMap
public Texture2D defaultSurfaceMap;   // → _SurfaceMap（细节法线/焦散）
public Material defaultSeaMaterial;   // 无限水面绘制用
public Mesh[] defaultWaterMeshes;     // 无限水面网格组
```

通过 `Resources.Load("WaterResources")` 加载，**必须放在 `Resources/` 文件夹下**，否则 `Water.Init()` 里 `resources` 为 null 会 NRE。

---

## 4. 入口：`Water.cs`

### 4.1 生命周期

```csharp
private void OnEnable()                    // Water.cs:65-80
{
    _useComputeBuffer = !computeOverride &&
                        SystemInfo.supportsComputeShaders &&
                        platform != WebGL && platform != Android;
    Init();
    RenderPipelineManager.beginCameraRendering += BeginCameraRendering;
    if (resources == null) resources = Resources.Load("WaterResources");
}
```

**关键点**：

- `[ExecuteAlways]` → 编辑模式下也运行，Inspector 改参数即时生效
- 单例 `Water.Instance` 用 `FindObjectOfType` 惰性查找，**不是**在 `Awake` 里赋值；多线程/时序敏感场景要小心
- `computeOverride = true` 的语义是「**强制不使用** ComputeBuffer」，命名有歧义
- Android 与 WebGL 被显式排除在 ComputeBuffer 之外，走 `Vector4[20]` 数组回退路径

### 4.2 `Init()` — 初始化主流程

```153:176:com.verasl.water-system/Scripts/Water.cs
public void Init()
{
    SetWaves();
    GenerateColorRamp();
    if (bakedDepthTex) Shader.SetGlobalTexture(WaterDepthMap, bakedDepthTex);

    if (!gameObject.TryGetComponent(out _planarReflections))
        _planarReflections = gameObject.AddComponent<PlanarReflections>();
    _planarReflections.hideFlags = HideFlags.HideAndDontSave | HideFlags.HideInInspector;
    _planarReflections.m_settings = settingsData.planarSettings;
    _planarReflections.enabled = settingsData.refType == ReflectionType.PlanarReflection;

    if (Application.platform != RuntimePlatform.WebGLPlayer)
        CaptureDepthMap();
}
```

任何 `WaterEditor.OnInspectorGUI` 检测到 `GUI.changed` 都会重新调用 `Init()`，所以编辑时改参数会**重复烘焙深度图**（有性能开销）。

### 4.3 `SetWaves()` — 波浪生成与参数下发

```192:254:com.verasl.water-system/Scripts/Water.cs
// 1) 波浪数组生成
SetupWaves(surfaceData._customWaves);

// 2) 默认贴图
Shader.SetGlobalTexture(FoamMap, resources.defaultFoamMap);
Shader.SetGlobalTexture(SurfaceMap, resources.defaultSurfaceMap);

// 3) 全局标量
_maxWaveHeight = Σ amplitude / waveCount;    // 注意：这是「平均振幅」不是最大值
Shader.SetGlobalFloat(WaveHeight,   transform.position.y);
Shader.SetGlobalFloat(MaxWaveHeight, _maxWaveHeight);
Shader.SetGlobalFloat(MaxDepth,     surfaceData._waterMaxVisibility);

// 4) 反射关键字三选一
// 5) Shader.SetGlobalInt(WaveCount, _waves.Length);

// 6) GPU 数据通道二选一
if (_useComputeBuffer) { EnableKeyword("USE_STRUCTURED_BUFFER"); waveBuffer = new ComputeBuffer(10, sizeof(float)*6); ... }
else                   { DisableKeyword("USE_STRUCTURED_BUFFER"); Shader.SetGlobalVectorArray(WaveData, GetWaveData()); }

// 7) CPU 侧
if (!GerstnerWavesJobs.Initialized && Application.isPlaying) GerstnerWavesJobs.Init();
```

**`SetupWaves()` 的随机生成逻辑**（`custom == false`）：

```267:298:com.verasl.water-system/Scripts/Water.cs
var r = 1f / numWave;
for (var i = 0; i < numWave; i++)
{
    var p   = Mathf.Lerp(0.5f, 1.5f, i * r);              // 递进系数
    var amp = a * p * Random.Range(0.8f, 1.2f);           // 振幅抖动 ±20%
    var dir = d + Random.Range(-90f, 90f);                // 方向抖动 ±90°
    var len = l * p * Random.Range(0.6f, 1.4f);           // 波长抖动 ±40%
    _waves[i] = new Wave(amp, dir, len, Vector2.zero, false);
    Random.InitState(surfaceData.randomSeed + i + 1);     // 每波换种子
}
Random.state = backupSeed;                                 // 恢复全局随机状态
```

用了「保存/恢复 `Random.state`」的正确做法，不会污染场景其它随机逻辑。

**通道 A：ComputeBuffer（波浪数 ≤ 10）**
- 固定申请 10 个元素的 buffer，`stride = 24`
- `Shader.SetGlobalBuffer(_WaveDataBuffer, waveBuffer)`
- Shader 侧 `StructuredBuffer<Wave> _WaveDataBuffer`

**通道 B：`Vector4[20]` 数组（兼容路径）**

```256:265:com.verasl.water-system/Scripts/Water.cs
var waveData = new Vector4[20];
for (var i = 0; i < _waves.Length; i++)
{
    waveData[i]      = new Vector4(amplitude, direction, wavelength, onmiDir);
    waveData[i + 10] = new Vector4(origin.x, origin.y, 0, 0);
}
```

打包约定：`[0..9]` 存 4 个标量，`[10..19]` 存 origin.xy。Shader 侧对应 `waveData[i]` 与 `waveData[i + 10].xy`。

> ⚠️ **硬上限 10 个波**。超过 10 个时：`GetWaveData()` 中 `waveData[i+10]` 会越界（i≥10），ComputeBuffer 也只装得下 10 个，而 `_WaveCount` 却被设为实际数量，Shader 循环会越界读取。

### 4.4 `GenerateColorRamp()` — 颜色查找表烘焙

生成一张 **128×4** 的 `R8G8B8A8_SRGB` 贴图，把三条曲线打包进一张图，Shader 用不同 V 坐标采样：

| 行 | V 采样值 | 内容 | 来源 |
|---|---|---|---|
| 0 | `0.0` | 吸收（Absorption）色 | `_absorptionRamp.Evaluate()` |
| 1 | `0.375` | 散射（Scattering）色 | `_scatterRamp.Evaluate()` |
| 2 | `0.66` | 泡沫混合色 | `defaultFoamRamp` 或 `basicFoam` 曲线 |
| 3 | — | 未使用 | — |

```57:65:com.verasl.water-system/Shaders/WaterCommon.hlsl
half3 Scattering(half depth)
{
    return SAMPLE_TEXTURE2D(_AbsorptionScatteringRamp, sampler_AbsorptionScatteringRamp, half2(depth, 0.375h)).rgb;
}
half3 Absorption(half depth)
{
    return SAMPLE_TEXTURE2D(_AbsorptionScatteringRamp, sampler_AbsorptionScatteringRamp, half2(depth, 0.0h)).rgb;
}
```

用「一张小贴图 + 不同 V」代替三张独立 LUT，省采样器槽位，是常见做法。

### 4.5 `CaptureDepthMap()` — 岸线深度烘焙

这是**离线预烘焙**步骤，用于让水面在靠近岸边/浅滩时产生正确的深度衰减和泡沫。

```341:397:com.verasl.water-system/Scripts/Water.cs
// 相机：正交、俯视、位于水面高度 + 4
t.position = Vector3.up * (transform.position.y + depthExtra);   // depthExtra = 4.0
t.up = Vector3.forward;                                          // 朝下看
_depthCam.orthographicSize = 250;                                // → 覆盖 500×500 单位
_depthCam.farClipPlane = surfaceData._waterMaxVisibility + depthExtra;
_depthCam.cullingMask = (1 << 10);                               // ★ 硬编码 Layer 10

// RT：1024×1024，24 位深度，Linear
_depthTex = new RenderTexture(1024, 1024, 24, RenderTextureFormat.Depth, RenderTextureReadWrite.Linear);
_depthCam.targetTexture = _depthTex;
_depthCam.Render();
Shader.SetGlobalTexture(WaterDepthMap, _depthTex);
Shader.SetGlobalVector(_VeraslWater_DepthCamParams, new Vector4(t.position.y, 250, 0, 0));
```

**采样端**：

```83:86:com.verasl.water-system/Shaders/WaterCommon.hlsl
float WaterTextureDepth(float3 posWS)
{
    return (1 - SAMPLE_TEXTURE2D_LOD(_WaterDepthMap, sampler_WaterDepthMap_linear_clamp,
            posWS.xz * 0.002 + 0.5, 1).r)
           * (_MaxDepth + _VeraslWater_DepthCamParams.x) - _VeraslWater_DepthCamParams.x;
}
```

`0.002 = 1 / 500`：因为正交范围 500 单位映射到 UV 的 `[0,1]`，`*0.002 + 0.5` 把世界 XZ 变换到深度图 UV。

> ⚠️ **强约束**：
> 1. 海底/地形必须放在 **Layer 10**，否则烘焙出来是空的
> 2. 覆盖范围写死 **500×500**，超出范围的水域深度采样错误（源码里也有 `//hardcoded = 1k area - TODO` 注释）
> 3. 只烘焙一次（`Init` 时），地形在运行时变化不会反映到深度图
> 4. WebGL 平台跳过（代码注释说明 OpenGL 深度有 bug）

若需要运行时不烘焙，可以手动指定 `bakedDepthTex` 字段（预烘焙好的贴图）。

### 4.6 `BeginCameraRendering()` — 无限水面

```107:143:com.verasl.water-system/Scripts/Water.cs
private void BeginCameraRendering(ScriptableRenderContext src, Camera cam)
{
    if (cam.cameraType == CameraType.Preview) return;

    Shader.SetGlobalFloat(CameraRoll, cam.transform.localEulerAngles.z);
    Shader.SetGlobalMatrix(InvViewProjection,
        (GL.GetGPUProjectionMatrix(cam.projectionMatrix, false) * cam.worldToCameraMatrix).inverse);

    const float quantizeValue = 6.25f;
    const float forwards = 10f;
    const float yOffset = -0.25f;

    var newPos = cam.transform.TransformPoint(Vector3.forward * forwards);
    newPos.y = yOffset;
    newPos.x = quantizeValue * (int)(newPos.x / quantizeValue);   // 量化，防抖动
    newPos.z = quantizeValue * (int)(newPos.z / quantizeValue);

    var matrix = Matrix4x4.TRS(newPos + transform.position, Quaternion.identity, transform.localScale);

    foreach (var mesh in resources.defaultWaterMeshes)
        Graphics.DrawMesh(mesh, matrix, resources.defaultSeaMaterial, gameObject.layer, cam, ...);
}
```

**设计意图**：水面网格不铺满整个世界，而是每一帧跟着相机走，实现「无限水面」。**量化（quantizeValue = 6.25）**是关键——不量化的话网格随相机连续移动，波浪因 UV 跟随会产生「游泳」般的抖动感；量化后网格按 6.25 单位跳格，视觉上稳定。

`_CameraRoll` 供 `InfiniteWater.shader` 做屏幕空间 raymarch 时校正相机滚转。

> ⚠️ `newPos` 已是世界坐标（`TransformPoint`），却又 `+ transform.position` 二次偏移，且 `newPos.y` 被硬设为 `-0.25`，把 `Water` 物体的 Y 位置覆盖掉。这是源码可疑处，改水面高度时要注意。

### 4.7 `FragWaveNormals()` — 遗留接口

```csharp
public void FragWaveNormals(bool toggle)
{
    var mat = GetComponent<Renderer>().sharedMaterial;
    mat.EnableKeyword("GERSTNER_WAVES") / DisableKeyword(...);
}
```

> ⚠️ 该方法依赖 `Renderer` 组件，但 `Water` 走的是 `Graphics.DrawMesh` 路线，`SeaVisual.prefab` 根节点未必有 Renderer，且 shader 中并不存在 `GERSTNER_WAVES` 关键字。属于**已失效的遗留代码**，不要调用。

---

## 5. 波浪系统

### 5.1 Gerstner 波公式

Gerstner（Trochoidal）波与正弦波的区别：顶点**同时**做水平位移和垂直位移，因此波峰更尖锐、波谷更平缓，更像真实涌浪。

### 5.2 GPU 路径 — `GerstnerWaves.hlsl`

```27:70:com.verasl.water-system/Shaders/GerstnerWaves.hlsl
WaveStruct GerstnerWave(half2 pos, float waveCountMulti, half amplitude, half direction,
                        half wavelength, half omni, half2 omniPos)
{
    half w      = 6.28318 / wavelength;      // 角波数 k = 2π / λ
    half wSpeed = sqrt(9.8 * w);             // 深水色散关系 ω = √(g·k)
    half peak   = 1.5;                       // ★ 陡峭度
    half qi     = peak / (amplitude * w * _WaveCount);   // 水平位移缩放

    direction = radians(direction);
    half2 dirWaveInput  = half2(sin(direction), cos(direction)) * (1 - omni);
    half2 omniWaveInput = (pos - omniPos) * omni;
    half2 windDir = normalize(dirWaveInput + omniWaveInput);
    half  dir     = dot(windDir, pos - (omniPos * omni));

    half calc    = dir * w + -time * wSpeed;
    half cosCalc = cos(calc);
    half sinCalc = sin(calc);

    wave.xz = qi * amplitude * windDir.xy * cosCalc;         // 水平位移
    wave.y  = sinCalc * amplitude * waveCountMulti;          // 垂直位移

    half wa = w * amplitude;
    half3 n = half3(-(windDir.xy * wa * cosCalc), 1 - (qi * wa * sinCalc));  // 解析法线

    waveOut.position = wave * saturate(amplitude * 10000);   // 振幅≈0 时归零，防 NaN
    waveOut.normal   = n.xzy * waveCountMulti;
    return waveOut;
}
```

**要点**：

1. **定向波 vs 全向波**：`omni = 0` 时 `windDir` 是固定方向向量（平行波）；`omni = 1` 时 `windDir = normalize(pos - omniPos)`，波从 `origin` 点向外扩散（圆形波，适合爆炸/落水涟漪）。
2. **法线是解析求导**，不是差分，成本低且平滑。
3. `waveCountMulti = 1 / _WaveCount`：多波叠加时均摊振幅，避免波数增多导致总高度爆炸。
4. `saturate(amplitude * 10000)` 是一个「振幅非零掩码」——`amplitude` 极小或为 0 时把整个位移乘成 0。
5. 时间取自 `_Time.y`；`_STATIC_SHADER` 关键字下时间恒为 0（用于烘焙/截图）。

**叠加循环**：

```72:106:com.verasl.water-system/Shaders/GerstnerWaves.hlsl
inline void SampleWaves(float3 position, half opacity, out WaveStruct waveOut)
{
    half waveCountMulti = 1.0 / _WaveCount;
    half3 opacityMask = saturate(half3(3, 3, 1) * opacity);

    UNITY_LOOP
    for (uint i = 0; i < _WaveCount; i++)
    {
        // 从 StructuredBuffer 或 Vector4 数组取 Wave
        WaveStruct wave = GerstnerWave(pos, waveCountMulti, w.amplitude, ...);
        waveOut.position += wave.position;
        waveOut.normal   += wave.normal;
    }
    waveOut.position *= opacityMask;
    waveOut.normal   *= half3(opacity, 1, opacity);
}
```

`opacity` 是浅水衰减因子（见 9.2），让浅水区波浪减弱——水平方向衰减更快（×3），垂直方向保留更多。

### 5.3 CPU 路径 — `GerstnerWavesJobs.cs`

```150:203:com.verasl.water-system/Scripts/GerstnerWavesJobs.cs
public void Execute(int i)
{
    var waveCountMulti = 1f / WaveData.Length;
    for (var wave = 0; wave < WaveData.Length; wave++)
    {
        var w      = 6.28318f / wavelength;
        var wSpeed = math.sqrt(9.8f * w);
        const float peak = 0.8f;                            // ★ 与 GPU 的 1.5 不同！
        var qi = peak / (amplitude * w * WaveData.Length);

        // ... windDir / dir / calc / cosCalc / sinCalc 同 GPU ...

        wavePos.x += qi * amplitude * windDir.x * cosCalc;
        wavePos.z += qi * amplitude * windDir.y * cosCalc;
        wavePos.y += sinCalc * amplitude * waveCountMulti;

        var wa = w * amplitude;
        var norm = new float3(-(windDir.xy * wa * cosCalc), 1 - (qi * wa * sinCalc));
        waveNorm += (norm * waveCountMulti) * amplitude;    // ★ 多乘了 amplitude
    }
    OutPosition[i] = wavePos;
    OutNormal[i]   = math.normalize(waveNorm.xzy);
}
```

**注册表机制**（多物体共享一块 NativeArray）：

```
_positions[4096]  ← 全局固定容量，所有浮力物体的采样点拼在一起
_wavePos[4096]    ← 输出结果
_waveNormal[4096] ← 输出法线
Registry: Dictionary<instanceID, int2(offsetStart, offsetEnd)>
```

- `UpdateSamplePoints(guid, samplePoints)`：首次注册分配区间，之后原地更新
- `GetData(guid, outPos, outNorm)`：按区间切片拷回托管数组
- 容量写死 **4096 个采样点**，超出的物体静默被拒（`if (_positionCount + len >= 4096) return;`）

**调度节奏**：

```csharp
// Water.LateUpdate → UpdateHeights()：调度 Job（_processing = true）
// BuoyantObject.Update → UpdateSamplePoints() → CompleteJobs()（等待上一帧结果）→ GetData()
```

一个典型的生产者-消费者错位设计：LateUpdate 生产，下一帧 Update 消费。

### 5.4 ⚠️ CPU / GPU 一致性问题

两侧公式存在**两处实质差异**，会导致「物体浮在波浪上的位置」与「看到的波浪形状」对不上：

| 项 | GPU (`GerstnerWaves.hlsl`) | CPU (`GerstnerWavesJobs.cs`) | 影响 |
|---|---|---|---|
| `peak` | `1.5` | `0.8` | 水平位移缩放 `qi` 差 ~1.9 倍，波峰陡峭度不同 |
| 法线累加 | `n.xzy * waveCountMulti` | `(norm * waveCountMulti) * amplitude` | CPU 法线额外乘了振幅，且最后 `normalize` |

**修复建议**：把 `peak` 统一为同一常量（建议 `1.5`，视觉更接近原版 Boat Attack），并去掉 CPU 侧多余的 `* amplitude`。

另外 CPU 侧 `Execute` 中的边界判断 `i < OffsetLength.x || i >= OffsetLength.y - OffsetLength.x` 表达式可疑（第二个条件应为 `i >= OffsetLength.y`），但当前 `OffsetLength = (0, 4096)` 且调度数量为 `_positionCount`，实际不会误触发。

---

## 6. 浮力系统 — `BuoyantObject`

### 6.1 四种模式

| 模式 | 体素化 | 物理 | 行为 |
|---|---|---|---|
| `NonPhysical` | 否（单点） | 否 | 直接把物体 Y 设为波高，用 `Slerp` 对齐法线（漂浮物/视觉道具） |
| `NonPhysicalVoxel` | 是 | 否 | **代码为空，未实现** |
| `Physical` | 否（单点） | 是 | 单点施加浮力（小船近似） |
| `PhysicalVoxel` | 是 | 是 | 逐体素施加浮力（推荐，效果最好） |

### 6.2 体素化 `SliceIntoVoxels()`

```249:297:com.verasl.water-system/Scripts/BuoyantObject.cs
// 1) 临时把物体归位到原点/单位缩放，便于遍历
t.SetPositionAndRotation(Vector3.zero, Quaternion.identity);
t.localScale = Vector3.one;

// 2) 用所有子 Collider 求包围盒，按 voxelResolution 向上取整
rawBounds = VoxelBounds();
_voxelBounds.size = RoundVector(rawBounds.size, voxelResolution);

// 3) 三重循环遍历体素中心点，用 Physics.ClosestPoint 判断是否落在碰撞体内
for ix / iy / iz ...
    var p = new Vector3(x, y, z) + _voxelBounds.center;
    inside = colliders.Any(c => PointIsInsideCollider(c, p));   // 距离 < 0.01 视为内部

// 4) 恢复 transform，计算体积与密度
volume  = Mathf.Min(rawVolume, voxelResolution³ × voxelCount);
density = rigidbody.mass / volume;
```

> ⚠️ 这是个 **O(n³)** 的初始化算法，`voxelResolution` 默认 0.51，稍大的船会生成几百到上千体素，且每个点都调 `Physics.ClosestPoint`。只在 `Init()` / `OnEnable` 执行一次，但复杂模型会有明显卡顿；建议对成品预计算或降低体素密度。

### 6.3 浮力计算

```217:235:com.verasl.water-system/Scripts/BuoyantObject.cs
private void BuoyancyForce(Vector3 position, float3 velocity, float waterHeight,
                           ref float submergedAmount, ref DebugDrawing debug)
{
    if (!(position.y - voxelResolution < waterHeight)) return;   // 完全在水面上，跳过

    var k = math.clamp(waterHeight - (position.y - voxelResolution), 0f, 1f);  // 浸没比例
    submergedAmount += k / _voxels.Length;

    var localDampingForce = Dampner * _rb.mass * -velocity;       // 阻尼 = 0.005 × m × (-v)
    var force = localDampingForce + math.sqrt(k) * _localArchimedesForce;
    _rb.AddForceAtPosition(force, position);                      // 作用点 = 体素位置 → 产生扭矩
}
```

- **阿基米德力**：`WaterDensity(1000) × |g| × volume / voxelCount`，方向 +Y
- 用 `sqrt(k)` 而非 `k`：浸没初期浮力增长更快，手感更「弹」
- `AddForceAtPosition` 在体素位置施力，多个体素受力不均自然产生俯仰/横滚
- **阻尼**系数 `Dampner = 0.005` 是纯经验值，模拟水的黏滞阻力

### 6.4 阻力更新

```237:242:com.verasl.water-system/Scripts/BuoyantObject.cs
PercentSubmerged = math.lerp(PercentSubmerged, submergedAmount, 0.25f);   // 平滑
_rb.drag        = _baseDrag + _baseDrag * (PercentSubmerged * 10f);
_rb.angularDrag = _baseAngularDrag + PercentSubmerged * 0.5f;
```

浸没越深，线性/角阻力越大，物体在水中运动越「黏」。

### 6.5 渲染同步

`Physical*` 模式下，体素本地坐标 → 世界坐标的转换走 `LocalToWorldJob`（Burst `IJob`），在 `LateUpdate` 调度、`Update`/`FixedUpdate` 里 `Complete()`，避免主线程阻塞。

---

## 7. 渲染管线扩展 — `WaterSystemFeature`

需要在 URP Renderer Data 里手动添加这个 `ScriptableRendererFeature`。它注入两个 Pass。

### 7.1 `WaterFxPass` — 水面交互缓冲

```12:69:com.verasl.water-system/Scripts/Rendering/WaterSystemFeature.cs
// Configure：
cameraTextureDescriptor.depthBufferBits = 0;
cameraTextureDescriptor.width  /= 2;              // 半分辨率
cameraTextureDescriptor.height /= 2;
cmd.GetTemporaryRT(m_WaterFX.id, descriptor, FilterMode.Bilinear);
ConfigureClear(ClearFlag.Color, new Color(0.0f, 0.5f, 0.5f, 0.5f));

// Execute：
var drawSettings = CreateDrawingSettings(new ShaderTagId("WaterFX"), ...);
context.DrawRenderers(cullResults, ref drawSettings, ref m_FilteringSettings);
```

**`_WaterFXMap` 通道打包格式**：

| 通道 | 含义 | 清屏默认值 |
|---|---|---|
| R | 泡沫遮罩（foam mask） | `0.0` |
| G | 法线 .x | `0.5`（即 0，因为 shader 里 `*2-1`） |
| B | 法线 .z | `0.5`（即 0） |
| A | 高度位移（displacement） | `0.5`（即 0） |

**工作方式**：任何使用 `LightMode = "WaterFX"` 的透明物体（船尾迹、落水涟漪贴片）都会在这一 Pass 被绘制到这张半分辨率 RT。水面 shader 采样它来叠加泡沫、扰动法线、产生局部隆起。

**渲染时机**：`RenderPassEvent.BeforeRenderingOpaques` —— 在天空盒和不透明物之前。

**写入端** `WaterFXShader.shader`：

```89:89:com.verasl.water-system/Shaders/WaterFXShader.shader
half4 comp = half4(foamMask, normalWS.xz, disp);
```

`_INVERT_ON` 关键字可整体取反（用于「挖洞」效果，如船体推开水面）。

### 7.2 `WaterCausticsPass` — 焦散

```75:113:com.verasl.water-system/Scripts/Rendering/WaterSystemFeature.cs
// 生成一个 1000×1000 的水平 Quad，跟随相机 XZ，Y 固定为 0
var position = cam.transform.position;
position.y = 0;                                  // TODO：应读取全局水面高度
var matrix = Matrix4x4.TRS(position, Quaternion.identity, Vector3.one);
cmd.DrawMesh(m_mesh, matrix, WaterCausticMaterial, 0, 0);
```

Shader 侧（`Caustics.shader`）流程：

1. 采样场景深度 → `ReconstructWorldPos()` 反推世界坐标
2. 用主光源方向矩阵 `_MainLightDir` 旋转世界坐标 → 得到「光照空间 UV」（焦散从光源方向投影下来）
3. 两次采样焦散贴图（不同频率、不同偏移）相乘叠加：`(A.z * B.z) * 10 + A.z + B.z`
4. 用 `upperMask`（水面以下）/ `lowerMask`（按 `_BlendDistance` 淡出）裁剪
5. 伪色散：`CausticsDriver * half3(A.w * 0.5, B.w * 0.75, B.x) * MainLight.color`

**混合模式**：非调试时 `SrcBlend = 2`（`SrcAlpha`？实际为 `Blend One` 语义）+ 返回 `Caustics + 1.0` —— 即「乘 1 为恒等，大于 1 变亮」的技巧，本质是**加性提亮**。

> ⚠️ `position.y = 0` 硬编码：水面不在 Y=0 时焦散位置不对。源码里有 TODO 注释。

**渲染时机**：
- 正常：`AfterRenderingSkybox + 1`
- 调试：`AfterRenderingPostProcessing`

### 7.3 配置项

```csharp
public class WaterSystemSettings
{
    [Range(0.1f, 1f)] public float causticScale = 0.25f;
    public float causticBlendDistance = 3f;
    public DebugMode debug = DebugMode.Disabled;   // Disabled | WaterEffects | Caustics
}
```

---

## 8. 反射系统

### 8.1 平面反射 — `PlanarReflections.cs`

```
主相机 beginCameraRendering
   └─ UpdateReflectionCamera()
        ├─ 计算反射平面：pos = 水面位置 + m_planeOffset，normal = (0,1,0)
        ├─ reflectionMatrix = Scale(1,-1,1) × CalculateReflectionMatrix(plane)
        ├─ _reflectionCamera.worldToCameraMatrix = 主相机.worldToCameraMatrix × reflection
        ├─ 斜投影：CalculateObliqueMatrix(CameraSpacePlane(...))   ← 自动裁剪水面以下
        └─ cullingMask = m_ReflectLayers
   └─ PlanarReflectionTexture()   创建/复用 RT（默认 1/3 分辨率，HDR）
   └─ 降低质量设置（关雾、LOD Bias ×0.5、GL.invertCulling = true）
   └─ UniversalRenderPipeline.RenderSingleCamera(context, _reflectionCamera)
   └─ Shader.SetGlobalTexture(_PlanarReflectionTexture, RT)
```

**斜投影（Oblique Near Plane）**是这里最优雅的设计：把反射相机的近平面设成水面平面，GPU 光栅化阶段自动裁掉水面以下的几何，省掉一次 shader discard 或二次渲染。

**性能开关**：
- `m_ResolutionMultiplier`：Full / Half / Third（默认）/ Quarter
- `m_Shadows`：反射是否渲染阴影
- `m_ClipPlaneOffset`：默认 0.07
- `m_ReflectLayers`：反射层级（应排除水层自身）

```csharp
// PlanarReflections.cs:103
if (Application.isPlaying && realCamera != Camera.main) return;
```

> ⚠️ 运行时只为主相机做反射；Scene 视图相机、UI 相机不产生反射（编辑模式下无此限制）。

**渲染开销**：每帧一次完整的场景渲染（降分辨率 + 降 LOD）。这是整个水体系统最贵的一环，移动端建议改用 Cubemap / ReflectionProbe。

### 8.2 Shader 侧采样

```80:107:com.verasl.water-system/Shaders/WaterLighting.hlsl
half3 SampleReflections(half3 normalWS, half3 viewDirectionWS, half2 screenUV, half roughness)
{
#if _REFLECTION_CUBEMAP
    reflection = SAMPLE_TEXTURECUBE(_CubemapTexture, ..., reflect(-viewDir, normal));
#elif _REFLECTION_PROBES
    reflection = GlossyEnvironmentReflection(reflect(-viewDir, normal), 0, 1);
#elif _REFLECTION_PLANARREFLECTION
    // 屏幕空间：用屏幕 UV + 法线 xz 偏移采样反射 RT
    half2 reflectionUV = screenUV + normalWS.zx * half2(0.02, 0.15);
    reflection = SAMPLE_TEXTURE2D_LOD(_PlanarReflectionTexture, ..., reflectionUV, 6 * roughness);
#endif
}
```

平面反射分支用的是**屏幕空间 UV + 法线扰动**近似（不是严格的光线追踪反射），偏移量 `(0.02, 0.15)` 是经验值：Z 方向偏移更大，模拟掠射角下拉长的倒影。

### 8.3 菲涅尔

```8:11:com.verasl.water-system/Shaders/WaterLighting.hlsl
half CalculateFresnelTerm(half3 normalWS, half3 viewDirectionWS)
{
    return saturate(pow(1.0 - dot(normalWS, viewDirectionWS), 5));
}
```

用的是 5 次幂的 **Schlick 近似（无 F0 项）**，比标准 Schlick 更激进——掠射角几乎全反射，正视几乎无反射。源码注释 `//fresnel TODO - find a better place` 说明作者自己也认为这里不理想。

---

## 9. Shader 层

### 9.1 文件职责

| 文件 | 职责 |
|---|---|
| `Water.shader` | 主入口，`Queue = Transparent-100`，`ZWrite On`，声明所有 keyword |
| `WaterTessellated.shader` | 细分版本入口（Hull/Domain） |
| `WaterCommon.hlsl` | ★ 顶点变换 + 片元着色主体 |
| `GerstnerWaves.hlsl` | 波浪位移与法线 |
| `WaterLighting.hlsl` | 菲涅尔、软阴影、三种反射、GGX 高光 |
| `WaterInput.hlsl` | 全局纹理/常量声明 |
| `CommonUtilities.hlsl` | `Remap` / `HeightToNormal` / value noise |
| `WaterTessellation.hlsl` | 基于屏幕空间边长的自适应细分 |
| `InfiniteWater.shader` | 屏幕空间 raymarch 无限水面（实验性，不完整） |
| `Caustics.shader` | 焦散投影 |
| `WaterFXShader.shader` / `WaterFXFoamOnly.shader` | 交互缓冲写入端 |

### 9.2 顶点阶段 — `WaveVertexOperations()`

```122:175:com.verasl.water-system/Shaders/WaterCommon.hlsl
WaterVertexOutput WaveVertexOperations(WaterVertexOutput input)
{
    input.normal = float3(0, 1, 0);

    // ① 噪声（用于 UV 扰动，打散平铺感）
    input.fogFactorNoise.y = ((noise(posWS.xz * 0.5 + time) + noise(posWS.xz * 1 + time)) * 0.25 - 0.5) + 1;

    // ② 两层细节 UV（不同尺度、不同速度、被噪声扰动）
    input.uv.zw = posWS.xz * 0.1 + time * 0.05 + noise * 0.1;   // 大尺度
    input.uv.xy = posWS.xz * 0.4 - time * 0.1  + noise * 0.2;   // 小尺度

    // ③ 浅水抬升：水越浅，水面被「顶」得越高（防穿插地形）
    half waterDepth = WaterTextureDepth(posWS);
    input.posWS.y += pow(saturate((-waterDepth + 1.5) * 0.4), 2);

    // ④ Gerstner 波（浅水处减弱）
    WaveStruct wave;
    SampleWaves(input.posWS, saturate(waterDepth * 0.1 + 0.05), wave);
    input.normal = wave.normal;
    input.posWS += wave.position;

    // ⑤ 动态位移（来自 _WaterFXMap，船尾迹等）
    half4 waterFX = SAMPLE_TEXTURE2D_LOD(_WaterFXMap, ..., screenUV.xy, 0);
    input.posWS.y += waterFX.w * 2 - 1;

    // ⑥ 变换到裁剪空间、计算屏幕 UV、视方向、雾
    input.clipPos  = TransformWorldToHClip(input.posWS);
    input.shadowCoord = ComputeScreenPos(input.clipPos);
    input.viewDir  = SafeNormalize(_WorldSpaceCameraPos - input.posWS);
    input.fogFactorNoise.x = ComputeFogFactor(input.clipPos.z);
    input.preWaveSP = screenUV.xyz;

    // ⑦ 附加数据打包
    input.additionalData = AdditionalData(input.posWS, wave);

    // ⑧ 远处法线淡出（防远处高光闪烁）
    half distanceBlend = saturate(abs(length((camPos.xz - posWS.xz) * 0.005)) - 0.25);
    input.normal = lerp(input.normal, half3(0, 1, 0), distanceBlend);
}
```

**`additionalData` 打包**：

| 分量 | 内容 |
|---|---|
| x | `length(viewPos / viewPos.z)` — 用于深度重建 |
| y | 相机到该点的距离 |
| z | **归一化波高** `wave.y / _MaxWaveHeight * 0.5 + 0.5` → 泡沫/SSS 用 |
| w | 水平位移量 `wave.x + wave.z` |

### 9.3 片元阶段 — `WaterFragment()`

```197:299:com.verasl.water-system/Shaders/WaterCommon.hlsl
// ① 采样交互缓冲
half4 waterFX = SAMPLE_TEXTURE2D(_WaterFXMap, ..., IN.preWaveSP.xy);

// ② 深度计算
float3 depth = WaterDepth(IN.posWS, IN.additionalData, screenUV.xy);
//    depth.x = 水面到海底的距离（沿视线）
//    depth.y = 水深（绝对值）
half depthMulti = 1 / _MaxDepth;

// ③ 细节法线：两层 detailBump 混合（浅水减弱）+ _BumpScale + waterFX 法线
IN.normal += half3(detailBump.x, 0, detailBump.y) * _BumpScale;
IN.normal += half3(1-waterFX.y, 0.5, 1-waterFX.z) - 0.5;
IN.normal = normalize(IN.normal);

// ④ 折射 UV 扰动（基于法线，深度越深扰动越强）
half2 distortion = DistortionUVs(depth.x, IN.normal);
distortion = screenUV.xy + distortion;
// 扰动后重采样深度，若得到负值（穿到水下）则回退到原始 UV ← 防 artifacts
depth.x = depth.x < 0 ? d : depth.x;

// ⑤ 菲涅尔
half fresnelTerm = CalculateFresnelTerm(IN.normal, IN.viewDir.xyz);

// ⑥ SSS（次表面散射近似）
half3 directLighting = dot(mainLight.direction, half3(0,1,0)) * mainLight.color;
directLighting += saturate(pow(dot(IN.viewDir, -mainLight.direction) * IN.additionalData.z, 3)) * 5 * mainLight.color;
half3 sss = directLighting * shadow + GI;
//   第二项是「逆光透射」：视线方向与光方向接近且波高大时发亮
//   使用 additionalData.z（归一化波高）作为波峰透光强度

// ⑦ 泡沫：三种来源取最大
half waveFoam  = saturate(IN.additionalData.z - 0.75 * 0.5);              // 波峰泡沫
half edgeFoam  = saturate((1 - min(depth.x, depth.y) * 0.5 - 0.25) + depthAdd) * depthEdge;  // 岸边泡沫
half foamBlendMask = max(max(waveFoam, edgeFoam), waterFX.r * 2);          // 交互泡沫
half3 foamBlend = SAMPLE_TEXTURE2D(_AbsorptionScatteringRamp, ..., half2(foamBlendMask, 0.66)).rgb;
half foamMask = saturate(length(foamMap * foamBlend) * 1.5 - 0.1);

// ⑧ 高光（GGX / PBR）
InitializeBRDFData(half3(0,0,0), 0, half3(1,1,1), 0.95, alpha, brdfData);   // 金属度 0，光滑度 0.95
half3 spec = DirectBDRF(brdfData, IN.normal, mainLight.direction, IN.viewDir) * shadow * mainLight.color;
// + _ADDITIONAL_LIGHTS 循环

// ⑨ 散射着色
sss *= Scattering(depth.x * depthMulti);

// ⑩ 反射
half3 reflection = SampleReflections(IN.normal, IN.viewDir.xyz, screenUV.xy, 0.0);

// ⑪ 折射（采样 _CameraOpaqueTexture + 吸收衰减）
half3 refraction = Refraction(distortion, depth.x, depthMulti);

// ⑫ 合成
half3 comp = lerp(lerp(refraction, reflection, fresnelTerm) + sss + spec, foam, foamMask);
comp = MixFog(comp, fogFactor);
```

**合成逻辑解读**：

```
底色 = mix(折射, 反射, 菲涅尔)     ← 掠射角看反射，俯视看水下
     + SSS                        ← 水的体积散射
     + 高光                       ← 太阳在波峰的镜面反射
结果 = mix(底色, 泡沫, foamMask)   ← 最后盖泡沫
```

> ⚠️ 注意：`_CameraOpaqueTexture` 需要 URP 的 **Opaque Texture** 选项开启，否则折射是黑的。`_CameraDepthTexture` 同理需要 **Depth Texture** 开启。

### 9.4 软阴影（水体专用 SSS 阴影）

```51:74:com.verasl.water-system/Shaders/WaterLighting.hlsl
half SoftShadows(float3 screenUV, float3 positionWS, half3 viewDir, half depth)
{
    half2 jitterUV = screenUV.xy * _ScreenParams.xy * _DitherPattern_TexelSize.xy;
    half depthFrac = depth * (1.0 / SHADOW_ITERATIONS);     // SHADOW_ITERATIONS = 4
    half3 lightOffset = -viewDir * depthFrac;

    for (uint i = 0u; i < 4; ++i)
    {
        jitterUV += frac(half2(_Time.x, -_Time.z));          // 时域抖动
        float3 jitterTexture = SAMPLE_TEXTURE2D(_DitherPattern, ..., jitterUV + i * _ScreenParams.xy).xyz * 2 - 1;
        half3 j = jitterTexture.xzy * depthFrac * i * 0.1;
        float3 lightJitter = (positionWS + j) + (lightOffset * (i + jitterTexture.y));
        shadowAttenuation += SAMPLE_TEXTURE2D_SHADOW(_MainLightShadowmapTexture, ..., TransformWorldToShadowCoord(lightJitter));
    }
    return BEYOND_SHADOW_FAR(...) ? 1.0 : shadowAttenuation * 0.25;
}
```

沿**视线方向向后**做 4 次抖动采样阴影图并平均 → 得到「水体内部散射导致的软阴影」。这是水这种半透明介质特有的处理：硬阴影在水里看起来很假，需要沿深度方向模糊。

依赖 `_DitherPattern` 贴图做抖动，材质上必须指定。

### 9.5 自适应细分 — `WaterTessellation.hlsl`

```28:36:com.verasl.water-system/Shaders/WaterTessellation.hlsl
float TessellationEdgeFactor(float3 p0, float3 p1)
{
    float edgeLength   = distance(p0, p1);
    float3 edgeCenter  = (p0 + p1) * 0.5;
    float viewDistance = distance(edgeCenter, _WorldSpaceCameraPos);
    return edgeLength * _ScreenParams.y / (_TessellationEdgeLength * viewDistance);
}
```

标准的**屏幕空间边长自适应**：目标是把每条边细分到接近 `_TessellationEdgeLength` 像素。近处细分多、远处细分少。需要 `#pragma require tessellation tessHW`（Shader Model 4.6+ / Metal / Vulkan，GLES 不支持）。

Hull 用 `fractional_odd` 分区 + `triangle_cw` 绕序；Domain 里做重心插值后复用同一个 `WaveVertexOperations()`。

### 9.6 全局参数速查表

| Shader 属性 | 类型 | 来源 | 说明 |
|---|---|---|---|
| `_WaveCount` | int | `Water.SetWaves` | 波数量（≤10） |
| `_WaveDataBuffer` | StructuredBuffer | `Water.SetWaves` | ComputeBuffer 路径 |
| `waveData[20]` | Vector4[20] | `Water.GetWaveData` | 数组回退路径 |
| `_WaveHeight` | float | `Water.SetWaves` | 水面世界 Y |
| `_MaxWaveHeight` | float | `Water.SetWaves` | 平均振幅 |
| `_MaxDepth` | float | `_waterMaxVisibility` | 最大可视水深 |
| `_WaterDepthMap` | Texture2D | `CaptureDepthMap` / `bakedDepthTex` | 岸线深度图 |
| `_VeraslWater_DepthCamParams` | Vector4 | `CaptureDepthMap` | (相机Y, 250, 0, 0) |
| `_AbsorptionScatteringRamp` | Texture2D | `GenerateColorRamp` | 128×4 LUT |
| `_FoamMap` / `_SurfaceMap` | Texture2D | `resources` | 泡沫 / 细节法线 |
| `_WaterFXMap` | Texture2D | `WaterFxPass` | 交互缓冲（半分辨率） |
| `_PlanarReflectionTexture` | Texture2D | `PlanarReflections` | 平面反射 RT |
| `_CubemapTexture` | Cubemap | `settingsData.cubemapRefType` | Cubemap 反射 |
| `_CameraRoll` | float | `Water.BeginCameraRendering` | 相机滚转角 |
| `_InvViewProjection` | Matrix4x4 | `Water.BeginCameraRendering` | 逆 VP 矩阵 |
| `_BumpScale` | half | 材质属性 | 细节法线强度 |
| `_CameraDepthTexture` | Texture2D | URP | 需开启 Depth Texture |
| `_CameraOpaqueTexture` | Texture2D | URP | 需开启 Opaque Texture |

**全局 Shader 关键字**：

| 关键字 | 控制方 | 含义 |
|---|---|---|
| `USE_STRUCTURED_BUFFER` | `Water.OnEnable` | 波浪数据用 ComputeBuffer 还是数组 |
| `_REFLECTION_CUBEMAP` / `_REFLECTION_PROBES` / `_REFLECTION_PLANARREFLECTION` | `Water.SetWaves` | 反射模式三选一 |
| `_STATIC_SHADER` | 材质 Toggle | 时间冻结（烘焙/截图用） |
| `_PLANAR_REFLECTION_CAMERA` | `PlanarReflections` | 渲染反射相机期间启用 |
| `_DEBUG_*`（8 种） | 材质 KeywordEnum | 调试可视化 |
| `_ADDITIONAL_LIGHTS` | URP | 附加光 |
| `_MAIN_LIGHT_SHADOWS` | URP | 主光阴影 |

### 9.7 调试模式

材质上的 `_Debug` KeywordEnum，直接输出中间量：

| 值 | 输出 |
|---|---|
| `SSS` | 散射光照 |
| `Refraction` | 折射（水下颜色） |
| `Reflection` | 反射 |
| `Normal` | 法线（RG 编码） |
| `Fresnel` | 菲涅尔项 |
| `WaterEffects` | `_WaterFXMap` 原始值 |
| `Foam` | 泡沫遮罩 |
| `WaterDepth` | `frac(depth)` 深度可视化 |

调试水面异常时，按这个顺序查最快：**WaterDepth → Normal → Fresnel → Refraction/Reflection → Foam**。

---

## 10. 资产清单

### 10.1 网格

| 网格 | 用途 |
|---|---|
| `Water.fbx` | 基础水面网格（含 LOD0/1/2 的 `WaterBlock_50m_*`） |
| `SeaVertDisp.fbx` | 顶点位移水面 |
| `InfiniteSea.fbx` | 无限海面（配合 `defaultWaterMeshes`） |

`SeaVisual.prefab` 结构：根节点 `SeaVisual`（挂 `Water`，Layer 4 = Water）下挂若干 `WaterLodPatch`，每个 patch 含 `WaterBlock_50m_LOD0/1/2` 三层 LOD —— 是一套 **50 米网格块 + 距离 LOD** 的平铺方案，与 `defaultWaterMeshes` 的「跟随相机单块」方案是两条可选路线。

### 10.2 材质

| 材质 | Shader |
|---|---|
| `Sea.mat` | `BoatAttack/Water` |
| `InfiniteWater.mat` | `Unlit/InfiniteWater` |
| `Caustics.mat` | `Hidden/BoatAttack/Caustics` |

### 10.3 贴图

| 贴图 | 用途 |
|---|---|
| `WaterSurface_single.tif` / `WaterSurface_atlas.tif` | 焦散 + 水面法线序列 |
| `SurfaceSequence/`（16 帧 PNG） | 水面动画序列 |
| `WaterFoam.png` | 泡沫贴图（R=厚 / G=中 / B=轻） |
| `DefaultFoamRamp.tif` | 默认泡沫渐变 |
| `PeakMap.tif` | 波峰遮罩 |
| `WaterSurface.asset` | 序列帧播放资产 |

---

## 11. 接入步骤

1. **URP Renderer Data** → Add Renderer Feature → `WaterSystemFeature`
2. **URP Asset** → 勾选 **Depth Texture** 和 **Opaque Texture**（缺一不可，否则折射/深度全错）
3. 创建两个 ScriptableObject：
   - `WaterSystem/Settings`（`WaterSettingsData`）
   - `WaterSystem/Surface Data`（`WaterSurfaceData`）
4. 把 `WaterResources` 资产放到任意 **`Resources/`** 目录下
5. 场景中放置 `SeaVisual.prefab`，或新建空物体挂 `Water` 组件并关联上面两个 SO
6. **把海底/地形放到 Layer 10**（否则岸线深度图为空）
7. 选中水面物体 → Inspector 中调整参数（`WaterEditor` 会在 `GUI.changed` 时自动 `Init()`）
8. 需要浮力：给物体挂 `BuoyantObject`，选 `PhysicalVoxel`，调 `voxelResolution`

### 参数调优建议

| 想要的效果 | 调什么 |
|---|---|
| 更汹涌 | 增大 `BasicWaves.amplitude`、`_waves[].amplitude` |
| 更平缓 | 增大 `wavelength`、减小 `amplitude` |
| 水更浑浊 | `_scatterRamp` 提亮、`_waterMaxVisibility` 减小 |
| 水更清澈 | `_absorptionRamp` 压暗、`_waterMaxVisibility` 增大 |
| 泡沫更多 | `FoamSettings` 曲线整体上移、`foamType` 改 1 自定义 |
| 远处闪烁 | 调 `_BumpScale` 减小，或加大 `distanceBlend` 系数 |

---

## 12. 已知问题与注意事项

### 12.1 功能性缺陷

| # | 问题 | 位置 | 影响 |
|---|---|---|---|
| 1 | **波浪数硬上限 10**，超出会越界/截断 | `Water.cs:242`、`GetWaveData()` | 崩溃或渲染错误 |
| 2 | **CPU/GPU `peak` 不一致**（0.8 vs 1.5） | `GerstnerWavesJobs.cs:170` vs `GerstnerWaves.hlsl:40` | 浮力物体与视觉波浪对不上 |
| 3 | CPU 法线多乘了 `amplitude` 且做了 `normalize` | `GerstnerWavesJobs.cs:199-202` | CPU 法线方向偏 |
| 4 | `BuoyantObject.NonPhysicalVoxel` **未实现**（case 为空） | `BuoyantObject.cs:138-140` | 该模式无效 |
| 5 | `FragWaveNormals()` 依赖不存在的 Renderer 和关键字 | `Water.cs:183-190` | 调用会 NRE |
| 6 | `InfiniteWater.shader` 是 **未完成** 的 raymarch 实现（输出 `frac(p_uv)` 调试值） | `InfiniteWater.shader:151` | 不可直接用于生产 |
| 7 | `WaterTessellated.shader` 的 RenderPipeline Tag 仍写 `LightweightPipeline` | `WaterTessellated.shader:10` | URP 下可能不生效 |
| 8 | `WaterSystemFeature.Create()` 中 `DebugMode.WaterEffects` 分支未设 `renderPassEvent` | `WaterSystemFeature.cs:166` | 该调试模式下 Pass 时机为默认 |

### 12.2 硬编码 / 平台限制

| # | 项 | 值 | 说明 |
|---|---|---|---|
| 1 | 深度烘焙 Layer | `1 << 10` | 地形必须在 Layer 10 |
| 2 | 深度烘焙范围 | `orthographicSize = 250`（500×500） | 超出范围无效 |
| 3 | 深度图分辨率 | 1024×1024 | 与 `posWS.xz * 0.002` 强耦合，改一个要改另一个 |
| 4 | 浮力采样点总容量 | 4096 | 所有浮力物体共享，超出静默失效 |
| 5 | 焦散网格 Y | `0` | 水面不在 Y=0 时错误 |
| 6 | 无限水面量化步长 / 前移 / Y | `6.25 / 10 / -0.25` | 写在 `BeginCameraRendering` 里 |
| 7 | ComputeBuffer 禁用平台 | WebGL、Android | 自动回退数组路径 |
| 8 | 深度烘焙禁用平台 | WebGL | OpenGL 深度 bug |

### 12.3 性能注意

1. **平面反射每帧完整重渲场景** —— 最贵的部分。移动端建议切 `Cubemap` 或 `ReflectionProbe`，或把 `m_ResolutionMultiplier` 降到 Quarter。
2. **软阴影 4 次抖动采样** —— 每像素 4 次 shadowmap 采样，等于阴影成本 ×4。
3. **焦散全屏后处理** —— 每像素 2 次贴图采样 + 深度重建，大面积水域下开销明显。
4. **`_WaterFXMap` 半分辨率** —— 已经是优化过的，但 `BeforeRenderingOpaques` 时机意味着即使场景中没有 WaterFX 物体，RT 仍会创建和清屏。
5. **编辑模式下每次 Inspector 改动都重跑 `Init()`** —— 包含一次 1024×1024 深度烘焙，改参数时会卡顿。

### 12.4 使用禁忌

- **场景中不要放多个 `Water` 组件**：全局 Shader 参数会互相覆盖
- **不要在运行时频繁改波浪参数**：`SetWaves()` 会重建 ComputeBuffer（GC + GPU 同步点）
- **不要依赖 `Water.Instance` 在 `Awake` 中可用**：它是惰性 `FindObjectOfType`
- **`BuoyantObject` 的 `voxelResolution` 不要设太小**：O(n³) 初始化，0.1 以下会卡死

---

## 13. 附录：一次典型的调试流程

**症状：水面全是黑的**

1. 检查 URP Asset 的 **Opaque Texture** 是否开启 → 折射采样 `_CameraOpaqueTexture`
2. 检查 **Depth Texture** 是否开启 → 深度计算
3. 材质 `_Debug` 切到 `WaterDepth`，看深度是否正常
4. 若深度全 0 → 检查地形是否在 **Layer 10**，或 `bakedDepthTex` 是否为空

**症状：远处水面闪烁**

- 减小 `_BumpScale`
- 加大 `WaterCommon.hlsl:171` 的 `distanceBlend` 系数（`0.005` 改大 → 更早淡出）
- 开启 `_SHADOWS_SOFT`

**症状：船浮不起来 / 浮的位置不对**

1. 确认 `GerstnerWavesJobs.Initialized`（只在 `Application.isPlaying` 时 Init）
2. 确认 `BuoyancyType` 是 `Physical` / `PhysicalVoxel`（`NonPhysical*` 不施力）
3. 确认 `Rigidbody` 存在且 `mass` 合理（密度 = mass / volume，>1000 会下沉）
4. 位置对不上 → 见 12.1 #2，统一 CPU/GPU 的 `peak`

**症状：平面反射没效果**

1. `settingsData.refType == PlanarReflection`
2. `WaterSystemFeature` 已加入 Renderer Data
3. `m_ReflectLayers` 包含了想要反射的物体
4. 运行时只有 `Camera.main` 会触发反射（Scene 视图不受此限）

---

## 14. 总结

| 维度 | 评价 |
|---|---|
| **架构** | 清晰分层，数据（SO）/ 逻辑（Water 单例）/ 渲染（RendererFeature）/ 着色（HLSL）职责分明 |
| **技术方案** | Gerstner 波 + 屏幕空间折射 + 深度衰减 + 平面反射，是 2019 年前后 URP 上的主流方案 |
| **完成度** | 核心水面渲染完整可用；浮力、焦散、WaterFX 完整；细分和 InfiniteWater 未完成 |
| **性能** | 平面反射 + 4x 软阴影 + 全屏焦散，中高端 PC 可跑，移动端需大幅裁剪 |
| **维护状态** | `1.4.0-preview.1`，含多处 TODO 与硬编码，属「示例项目提取」而非产品级包 |
| **主要风险** | 全局参数单例化、10 波上限、CPU/GPU 波参数不一致、Layer 10 硬约束 |

**如果要在本项目深度使用，建议优先处理**：① CPU/GPU `peak` 统一（浮力正确性）；② 深度烘焙 Layer 与范围改为可配置；③ 波浪上限改为动态；④ 平面反射在目标平台做降级开关。
