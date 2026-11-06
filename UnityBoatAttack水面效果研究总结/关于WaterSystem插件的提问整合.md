# 关于 WaterSystem 插件的提问整合

> 整理自对 `com.verasl.water-system`（Boat Attack Water System, 1.4.0-preview.1, URP 7.4.1）的源码深度阅读问答。
> 完整的程序分析文档见同项目 `Assets/文档/WaterSystem插件程序分析.md`。
> 每个问题包含：代码位置、问题原文、简答（TL;DR）、详细分析。

---

## 目录

| # | 问题 | 涉及文件 |
|---|---|---|
| 1 | `_WaveDataBuffer` 和 `waveData` 是如何被赋值的 | GerstnerWaves.hlsl:83-91, Water.cs |
| 2 | 波浪的"风"计算是怎么回事 | GerstnerWavesJobs.cs:176-182 |
| 3 | 相位计算为什么用 `-Time`，正负不是无所谓吗 | GerstnerWavesJobs.cs:185 |
| 4 | 波高为什么要除以波浪数，这不违背线性叠加物理吗 | GerstnerWavesJobs.cs:192 |
| 5 | GPU 侧为什么要乘 `saturate(amplitude * 10000)`，CPU 侧没有 | GerstnerWaves.hlsl:66 |
| 6 | Hull Shader 的那串属性是什么意思 | WaterTessellation.hlsl:64-72 |
| 7 | 反射相机为什么要关闭遮挡剔除 | PlanarReflections.cs:91 |
| 8 | 为什么用一个四维向量就能表示平面 | PlanarReflections.cs:119-120 |
| 9 | 反射平面为什么一定要有偏移（offset） | PlanarReflections.cs:111-120 |
| 附 | 阅读过程中发现的问题清单 | 多处 |

---

## 问题 1：`_WaveDataBuffer` 和 `waveData` 是如何被赋值的

**位置**：`Shaders/GerstnerWaves.hlsl:83-91`（读取端），`Scripts/Water.cs:238-265`（赋值端）

**问题原文**：这段代码中 `_WaveDataBuffer` 和 `waveData` 是如何被赋值的？

**TL;DR**：两者是同一份波浪数据的两条传输通道，由 `USE_STRUCTURED_BUFFER` 宏在编译期二选一；数据都来自 `Water.cs` 的 `SetWaves()`，运行时不会每帧更新。

### 分析

**分支源头**（HLSL 全局作用域声明，**不在 CBUFFER 内**——这是能被 `Shader.SetGlobal*` 赋值的前提）：

```hlsl
#if defined(USE_STRUCTURED_BUFFER)
StructuredBuffer<Wave> _WaveDataBuffer;
#else
half4 waveData[20]; // 0-9 amplitude/direction/wavelength/omni, 10-19 origin.xy
#endif
```

开关在 `Water.shader` 以 `#pragma multi_compile _ USE_STRUCTURED_BUFFER` 声明，两个变体都会被编译，运行时靠全局关键字选择：

```csharp
if (_useComputeBuffer) {
    Shader.EnableKeyword("USE_STRUCTURED_BUFFER");
    waveBuffer = new ComputeBuffer(10, sizeof(float) * 6);
    waveBuffer.SetData(_waves);                          // 二进制原样上传
    Shader.SetGlobalBuffer(WaveDataBuffer, waveBuffer);
} else {
    Shader.DisableKeyword("USE_STRUCTURED_BUFFER");
    Shader.SetGlobalVectorArray(WaveData, GetWaveData()); // 手工打包成 Vector4[20]
}
```

**路径 A（ComputeBuffer）**：C# `Wave` 结构体（amplitude/direction/wavelength/origin(float2)/onmiDir）与 HLSL `Wave` 逐字段对齐，合计 24 字节 = `sizeof(float)*6`，`SetData` 不做转换、靠内存布局隐式映射。

**路径 B（Vector4 数组）**：`GetWaveData()` 把每个波拆成两条：

```
索引   0..9   ← bank0: (amplitude, direction, wavelength, onmiDir)
索引  10..19  ← bank1: (origin.x, origin.y, 0, 0)
```

这就是选中代码里 `waveData[i]` 与 `waveData[i + 10].xy` "跳 10 个索引"写法的由来——用一维数组模拟二维结构，绕开旧平台 StructuredBuffer 支持不足的问题。

**通道选择**在 `OnEnable` 决定：支持 ComputeShader 且非 WebGL/Android 走 A；`computeOverride = true` 强制走 B（命名有歧义：它的语义是"强制不使用 ComputeBuffer"）。

**赋值时机**：只在 `Init()` → `SetWaves()` 里调用，触发点是 `OnEnable` 与编辑器 `GUI.changed`。想动态改波浪必须手动再调 `SetWaves()`。

**硬上限**：两条路径容量都是 10 波。超过 10 个时数组路径 `waveData[i+10]` 越界、ComputeBuffer 溢出，而 `_WaveCount` 仍按实际数量下发，shader 循环越界读。

---

## 问题 2：波浪的"风"计算是怎么回事

**位置**：`Scripts/GerstnerWavesJobs.cs:176-182`

**问题原文**：这段对于风的计算，我没弄明白，你来分析一下。

**TL;DR**：`windDir` 是误导性命名——它算的不是风，而是**波的传播方向单位向量**（波矢方向 k̂），最终目的是把 2D 世界坐标投影成一个标量相位 `dir`。

### 分析

Gerstner 波相位为 \(\theta = \mathbf{k}\cdot(\mathbf{P}-\mathbf{P}_0) - \omega t = |\mathbf{k}|\,[\hat{\mathbf{k}}\cdot(\mathbf{P}-\mathbf{P}_0)] - \omega t\)，对照 185 行 `calc = dir * w - Time * wSpeed` 可知 `dir` 承担 \(\hat{\mathbf{k}}\cdot(\mathbf{P}-\mathbf{P}_0)\)（沿传播方向的距离）。

逐行拆解：

| 行 | 代码 | 作用 |
|---|---|---|
| 176 | `(sin θ, cos θ) * (1 - onmiDir)` | 定向波方向 k̂（θ=0 → +Z，θ=90° → +X），onmiDir=1 时清零 |
| 177 | `(pos - omniPos) * onmiDir` | 全向波径向向量（**未归一化**，长度 = 距离 r），onmiDir=0 时清零 |
| 179-181 | 两次 `+=` 后 `normalize` | 因 `onmiDir` 严格为 0/1，两项永不同时非零 → 等价于 GPU 的一行 `normalize(A+B)` |
| 182 | `dot(windDir, pos - omniPos*onmiDir)` | 投影成标量：定向波得 `k̂·pos`（直线波前）；全向波得 `r`（圆形波前） |

**`onmiDir` 是 branchless 三连开关**（176 乘 `1-omni`、177 乘 `omni`、182 乘 `omniPos*omni`），一处代码覆盖两种波、无分支，代价是要求 `onmiDir` 严格为 0 或 1——手填 0.5 会得到"半定向半全向"的怪异混合（182 行参考原点也被缩放，不是平滑插值）。

**181 行的 normalize 对全向波是功能必需**：若不归一化，`dir = r²`，径向波长会随半径压缩（d(r²)/dr = 2r），涟漪失真。归一化后 `dir = r`，径向波长恒为 λ。同时 `windDir` 还在 190-191 行用作 Gerstner 水平位移方向（顶点沿传播方向往复），单位向量是唯一同时满足两用途的解。

**数值验证**（定向波 90°、pos=(3,7)）：`windDir=(1,0)`，`dir=3`，x=3 的任意 z 点相位相同 → 直线波前。（全向波 origin=(10,10)、pos=(13,14)）：`windDir=(0.6,0.8)`，`dir=5=r` → 圆形波前。

**坑**：
- 全向波在 `pos == omniPos` 时 `normalize(float2(0,0))` → `0 × rsqrt(0)` = NaN → 波高 NaN → Rigidbody 位置 NaN，物体消失。CPU 侧浮力采样是离散点，更易命中。防护：`len2 > 1e-8 ? raw * rsqrt(len2) : fallback`。
- 176 行尾 `// TODO - currently radians` 是残留疑虑：175 行刚做过 `math.radians()`，`sin/cos` 吃弧度是正确的，GPU 43 行同样，无 bug。
- 逐项对照 GPU 44-48 行，数学**完全等价**；CPU/GPU 真正的不一致在 `peak`（0.8 vs 1.5）与法线累加。

**附带发现**：`pos = Position[i].xz` 来自 `BuoyantObject._samplePoints`。Physical 模式由 `LocalToWorldJob` 写入世界坐标；但 **NonPhysical / NonPhysicalVoxel 模式下 `LocalToWorldConversion()` 直接 return，`_samplePoints` 从未被写入，恒为 (0,0,0)**——所有浮力物体都在世界原点采样同一个波高。

---

## 问题 3：相位计算为什么用 `-Time`

**位置**：`Scripts/GerstnerWavesJobs.cs:185`：`var calc = dir * w + -Time * wSpeed;`

**问题原文**：这里为什么用的是 -Time，正负应该无所谓才是。

**TL;DR**：正负号恰恰最有所谓——它单独决定波的**传播方向**。`-Time` 让波朝 `direction` 字段配置的方向传播，并与 GPU 侧的 `-time` 严格配对。

### 分析

行波等相位面 \(\theta = \mathbf{k}\cdot\mathbf{x} \mp \omega t = C\) 解出 \(\partial x/\partial t = \pm\omega/k\)：`-ωt` 沿 \(+\hat{\mathbf{k}}\)，`+ωt` 沿 \(-\hat{\mathbf{k}}\)。波峰位置 \(\text{dir}_{peak}(t) = (\pi/2 \pm \omega t)/w\)。

**数值演示**（λ=10m，w=0.628，ω=2.48，相速度 ≈3.95 m/s）：

| 时间 | `-Time` 波峰位置 | `+Time` 波峰位置 |
|---|---|---|
| t=0 | +2.50 m | +2.50 m（相同） |
| t=1s | +6.45 m | −1.45 m |
| t=2s | +10.4 m | −5.40 m |

t=0 两种波形完全相同——这正是"正负无所谓"错觉的来源；区别只体现在时间演化上。

**cos 偶函数陷阱**：\(\cos(-\theta)=\cos\theta\) 只对整个自变量成立；这里变号的只是 calc 的一个组成部分（\(A-B \to A+B\)），\(-\text{calc} = -A+B \neq A+B\)，偶函数性质帮不上忙。

**精确等价关系**：翻转时间符号 ≈ 把所有波的 `direction` 转 180°（位移场差 π 相位）。所以符号不是冗余自由度，它与 `direction` 二者取一地锁定传播方向。

**系统耦合**：`Wave.direction` 注释即 "the wave travels in"（WaterSurfaceData.cs:29）；GPU 51 行同为 `-time`，只改 CPU 一侧会让浮力采样与画面波浪反相错拍。`+Time` 得到的仍是物理合法解（时间反演对称），问题不是"算错"而是"方向反了 + 可能失去同步"。

**何时真的无所谓**：仅当时间冻结——`STATIC_EVERYTHING`（CPU 98-102 行，t=0）或 `_STATIC_SHADER`（GPU 30-34 行），即截图/烘焙模式。

---

## 问题 4：波高为什么要除以波浪数

**位置**：`Scripts/GerstnerWavesJobs.cs:192`：`wavePos.y += sinCalc * amplitude * waveCountMulti;`（`waveCountMulti = 1/N`）

**问题原文**：按照物理规律，波之间的高度不是互相叠加吗？直接相加就好了，除以波浪数不就和真实的物理模拟相违背了吗？

**TL;DR**：物理直觉正确，线性叠加就是直接相加；这里的除法不是物理建模而是**参数归一化**——它把 `amplitude` 的语义从"单波物理振幅"重定义为"合成波高的均分预算"。物理公式没变，变的是用户输入到物理振幅的映射。

### 分析

\[
\sum_i \frac{a_i}{N}\sin\theta_i = \frac{1}{N}\sum_i a_i \sin\theta_i
\]

代码版本就是物理版本整体除以 N，波形形状（波峰、波谷、干涉图样）完全不变，只是尺度缩到 1/N。

**三个环环相扣的工程理由**：

1. **`_MaxWaveHeight` 归一化闭环**：高度除以 N 后，合成波理论上限 \(\sum a_i/N\) 恰好等于 `Water.cs:200-205` 计算的平均振幅。于是 shader 里 `additionalData.z = wave.y / _MaxWaveHeight * 0.5 + 0.5`（WaterCommon.hlsl:117）严格落在 [0,1]，才能安全喂给泡沫阈值和 SSS。不除的话上限是 Σaᵢ = N×_MaxWaveHeight，归一化溢出到 6。

   数值例子（6 波 × 1.5m）：直接相加最坏 9m、`data.z` 最高 6.0 溢出；除以 N 最坏 1.5m、严格 [0,1]。

2. **Gerstner 陡峭度预算**：`qi = peak/(amplitude·w·N)` 使单波陡峭度 = `peak/N ≤ 1`，防止 N 波叠加时波形自交打结（法线翻转、渲染破碎）。注释 "1 is the sharpest peaks" 指的是**整个波场**的陡峭度总预算。这也解释了 CPU/GPU `peak` 不一致（0.8 vs 1.5）为何影响巨大。

3. **调参体验**：波数与浪高解耦——改波数不改变浪高，"波数"成为表面细节丰富度参数。实时水体系统（含 GPGPU 波谱合成）的通行做法。

**物理视角**：真实海浪谱（Pierson-Moskowitz/JONSWAP）里能量有限，每个模式的振幅本就随模式数/频段收窄而下降——"模式越多、单波越小"在趋势方向与物理一致，1/N 是最粗暴的均匀分配。

**代价**：波被稀释，N 越大越趋向噪声面；`_MaxWaveHeight` 是最坏情况上界，随机相位下典型波高仅其 ~0.6 倍（√N 增长），泡沫/SSS 曲线实际工作在 [0, 0.7] 区间。

**若要改成物理叠加**，必须成套改：① 154 行 `waveCountMulti = 1f`；② Water.cs:205 的 `_maxWaveHeight` 改为不除（保持 Σ）；③ 171 行与 GPU 41 行 `qi` 里的 N 同步去掉——否则水平位移与垂直位移比例改变、波形会变且大概率波峰翻卷。

---

## 问题 5：GPU 侧为什么要乘 `saturate(amplitude * 10000)`

**位置**：`Shaders/GerstnerWaves.hlsl:66`：`waveOut.position = wave * saturate(amplitude * 10000);`（CPU 190-192 行无此操作）

**问题原文**：这里为什么又要乘上 saturate(amplitude * 10000)，在 CSharp 端中的实现里压根就没有这个操作。

**TL;DR**：`qi` 公式把 `amplitude` 约掉了，导致振幅归零时**水平推拉依然满幅存在**（"平水面在抖动"）；这个乘法是把退化波整体淡出的零振幅安全阀。CPU 侧不需要，因为它的水平位移是没人读的死数据。

### 分析

**mask 本体**：`saturate` = clamp(0,1)，阈值在 amplitude = 1e-4（0.1mm）。对任何正常振幅恒为 1.0（99.99% 情况下是恒等操作），只在振幅趋零时介入。

**根因——`qi·a` 约掉振幅**：

\[
q_i \cdot a = \frac{peak}{a \cdot w \cdot N} \cdot a = \frac{peak}{w \cdot N}
\]

水平位移振幅与 amplitude 完全无关（λ=10、N=6 时恒为 ≈0.40m），垂直位移振幅 = a/N 与之成正比。于是：

| amplitude | 垂直起伏 | 水平推拉 | mask | 观感 |
|---|---|---|---|---|
| 1.5 | 0.25m | 0.40m | 1.0 | 正常 |
| 0.01 | 0.0017m | **0.40m（不变！）** | 1.0 | 水面几乎平，但顶点水平狂抖 40cm |
| 0 | 0 | 0.40m | 0 | 应全灭 |

由于 `_WaveCount` 是数组长度无法动态减少，只能靠把 amplitude 置 0 来"禁用"某个波——mask 就是让这个操作真正生效。

**CPU 侧不需要的原因**：翻遍整个包，`Heights[i].x/.z` **从未被读取**（BuoyantObject 只消费 `Heights[i].y`，法线是 197-198 行用 cosCalc/sinCalc 独立计算的）。所以 CPU 的 `wavePos.xz` 即使因 amplitude=0 产生垃圾值，`wavePos.y` 依然正常趋于 0，浮力天然正确；而 GPU 的 `wave.position` xyz 全部参与顶点位移（`input.posWS += wave.position`），xz 异常会直接摧毁顶点。

**局限——防不彻底**：
- amplitude 精确为 0：\(q_i = peak/0 = +\infty\)，\(\infty \times 0 =\) NaN，而 `NaN × 0 = NaN`，乘法无法清除 → 顶点照样 NaN 被光栅化器丢弃。
- HLSL `half` 最大 ≈65504，amplitude < ~6e-6（λ=10、N=6）时 qi 溢出为 Inf，`Inf × a` 仍是 Inf，有限 mask 压不住。
- 有效区间仅 amplitude ∈ [6e-6, 1e-4]。作者依赖编辑器滑块 min=0.1 回避 a=0；但 `BasicWaves.amplitude` 在 SO 里填 0 会让全部 6 波 NaN → 海面凭空消失。

**`10000` 的来历**：阈值 0.1mm 的编码（1/1e-4），该尺度在屏幕上完全不可见。

**建议修法**（两侧同时修根因，不在 CPU 补 mask）：

```hlsl
// GerstnerWaves.hlsl:41
half qi = peak / (max(amplitude, 1e-5) * w * _WaveCount);
```

```csharp
// GerstnerWavesJobs.cs:171（为数值一致性，可选）
var qi = peak / (math.max(amplitude, 1e-5f) * w * WaveData.Length);
```

此时 `qi·a` 在 amplitude→0 时自然趋于 0，66 行 mask 可保留作双保险（视觉淡出层）。

---

## 问题 6：Hull Shader 的那串属性是什么意思

**位置**：`Shaders/WaterTessellation.hlsl:64-72`

**问题原文**：这段我看不懂，是什么意思。

**TL;DR**：这是 GPU 细分管线第一站（Hull Shader）的"配置清单 + 直通函数"——声明补丁域为三角形、用 fractional_odd 分区、顺时针输出、细分因子由 `HSConstant` 提供、3 进 3 出；它本身不做变形，变形在下游 Domain 里做。

### 分析

细分管线三阶段：

```
TessellationVertex ──► [Hull Shader] ──► [固定功能 Tessellator] ──► [Domain Shader] ──► WaterFragment
   (原顶点)          可编程，输出            纯硬件，不可编程            可编程，输出
                     控制点 + 细分因子       按因子切出密集顶点          插值 + 波浪位移
```

**五个属性**：

| 属性 | 含义 |
|---|---|
| `[domain("tri")]` | 补丁域是三角形，Tessellator 用三角形插值（可选 quad/isoline） |
| `[partitioning("fractional_odd")]` | 细分因子的取整/过渡方式，见下 |
| `[outputtopology("triangle_cw")]` | 细分产出的三角形顺时针绕序（影响背面剔除） |
| `[patchconstantfunc("HSConstant")]` | 补丁常量函数，每补丁跑一次，输出细分因子（本项目按屏幕空间边长算） |
| `[outputcontrolpoints(3)]` | 每补丁输出 3 个控制点，Hull 函数体被调用 3 次（`SV_OutputControlPointID` = 0/1/2） |

**直通 Hull**：`return Input[uCPID]` 对控制点不做任何变形。Hull 阶段在硬件上是必经之路——固定功能 Tessellator 需要它提供控制点和细分因子（`HSConstant` 里 `TessellationEdgeFactor` 按屏幕空间边长算每条边因子）。真正的变形在 Domain（77-96 行）做：重心插值新顶点后调用与普通版本相同的 `WaveVertexOperations()` 做 Gerstner 位移。

**关键设计**：**细分发生在波浪位移之前**。低模网格顶点间距远大于波长，Gerstner 波采不出来；先细分出密顶点再位移，波浪才立得起来。

**partitioning 对比**：

| 模式 | 行为 |
|---|---|
| `integer` | 向上取整，LOD 变化跳变明显 |
| `fractional_even` | 过渡区间 [2,4]，factor=1 时仍产生 2 段（长度可为 0） |
| `fractional_odd` | 过渡区间 [1,3]，**factor=1 时恰好 1 段**（等于不细分），远处退回不细分时最平滑 |
| `pow2` | 只允许 2 的幂，已废弃 |

水面是典型的"近细分、远不细分"场景，`fractional_odd` 是此类需求的常见选择。

**发现的 bug**：Domain 里 `UNITY_SETUP_INSTANCE_ID(v); UNITY_TRANSFER_INSTANCE_ID(v, o);`（81-82 行）引用了不存在的标识符 `v`（参数是 `Input`）——从 `WaterVertex`（参数名恰好叫 v）复制代码的笔误。未启用 instancing 时宏展开为空所以能编译；`WaterTessellated.shader:38` 有 `multi_compile_instancing`，instancing 变体可能编译失败。应为 `UNITY_SETUP_INSTANCE_ID(Input)`。

---

## 问题 7：反射相机为什么要关闭遮挡剔除

**位置**：`Scripts/Rendering/PlanarReflections.cs:91`：`dest.useOcclusionCulling = false;`

**问题原文**：为什么反射相机要关闭遮挡剔除。

**TL;DR**：遮挡剔除依赖烘焙的静态可见性数据（PVS），而反射相机是"视锥被平面镜像"的非法视角，剔除系统对它的判断会失效——把本应出现在倒影里的物体错误剔除。宁可多画，不能漏画。

### 分析

**Umbra 工作原理**：烘焙时把静态场景切分成 cell，预计算每个 cell 的潜在可见集；运行时"相机位置 → 定位 cell → 查 PVS → 用遮挡体判定视锥内物体是否被挡"。对正常相机是**保守且正确**的（宁可多画不漏画）。

**对反射相机失效的两个原因**：

1. **位置与视锥脱节**：反射相机 `transform.position` 在水面以下的镜像位置（`ReflectPosition`），PVS 查询用的是"真实站在水下这个点能看到什么"，与镜像视锥实际覆盖的方向毫无关系。
2. **镜像世界的遮挡关系 ≠ 真实几何**：视锥镜像后遮挡体在光线路径上的位置关系整个翻转。剔除系统不理解"视锥被平面翻转"，按真实几何算遮挡 → 结果是**错误的剔除**而非保守剔除。

**后果**：倒影里的物体时隐时现、成块消失（如岸边石头在倒影里没了）；Scene 视图完全复现不出来（Scene 相机是正常视角），极难排查。

**注意**：关闭的只是遮挡剔除，**视锥剔除仍工作**（反射相机的 view/projection 矩阵是构造好的），并非全量渲染。成本由其他手段控制：`cullingMask = m_ReflectLayers`（136 行）、分辨率默认 1/3、LOD Bias 减半、雾关闭。

**权衡**：

| 方案 | 代价 | 风险 |
|---|---|---|
| 开启 | 省一点 GPU | 倒影随机缺物体（错误剔除） |
| 关闭 | 反射多画一些 | 倒影保证完整 |

这是行业惯例：任何"视锥被数学变换扭曲过"的相机（反射、折射、portal、阴影相机）都应默认关闭遮挡剔除。

---

## 问题 8：为什么用一个四维向量就能表示平面

**位置**：`Scripts/Rendering/PlanarReflections.cs:119-120`

**问题原文**：为什么用一个四维量便可以表示平面。

**TL;DR**：`(n, D)` 四维向量就是平面方程 \(Ax+By+Cz+D=0\) 的系数——法向三个 + 距离一个，恰是完备且便于计算的齐次表示。

### 分析

**代码验证**：

```csharp
var d = -Vector3.Dot(normal, pos) - m_settings.m_ClipPlaneOffset;
var reflectionPlane = new Vector4(normal.x, normal.y, normal.z, d);
```

平面上任一点 \(\mathbf{p}_0\) 代入方程得 \(D = -\mathbf{n}\cdot\mathbf{p}_0\)，正是第一项。数值例子（水面 y=2、offset=0.07）：\(d = -2.07\)，即平面被抬到 y=2.07。

**自由度**：3D 平面只有 3 个自由度（法向 2 + 距离 1），4 维表示有一个齐次冗余：\((\mathbf{n},D) \equiv (k\mathbf{n},kD), k\neq0\)。约定 \(|\mathbf{n}|=1\)（这里 `Vector3.up` 恰好单位），\(D\) 即原点到平面的有符号距离。

**为什么愿意用 4 维——下游三种运算**：

1. **点积判定**：点齐次形式 \((x,y,z,1)\)，一次点积判正/在面上/负。
2. **平面的矩阵变换**：点变换用矩阵直接乘，**平面变换必须乘逆转置** \(\mathbf{P}' = (M^{-1})^T\mathbf{P}\)——要求操作数是 4 维列向量。`CameraSpacePlane`（194-201 行）把世界空间平面转到相机空间就是一次标准矩阵乘，也是 `CalculateObliqueMatrix` 要求的输入格式。用"点+法线"（6 个数）表示则每次变换都要拆开重组。
3. **反射矩阵闭式推导**：`CalculateReflectionMatrix` 里的 `1-2nₓ²` 等元素，起点就是平面方程。

**射影几何背景**：齐次坐标下点与平面天然对偶，"点在平面上"= \(\mathbf{P}\cdot\mathbf{q}=0\)，对称可交换。整套图形学管线（裁剪平面、视锥 6 平面、斜投影、阴影体）都建立在此表示上；`Camera.frustumPlanes`、Unity `Plane` 结构内部同样是四分量。

---

## 问题 9：反射平面为什么一定要有偏移

**位置**：`Scripts/Rendering/PlanarReflections.cs:111-120`（`m_planeOffset` 与 `m_ClipPlaneOffset`）

**问题原文**：我发现这个平面计算都会有一个 m_planeOffset 值，为什么反射平面一定要有偏移呢。

**TL;DR**："反射平面"是数学上的理想平面，而水面是不断起伏的曲面，两者永远无法精确对齐；offset 是给这个不可消除的误差留余量。代码里其实是**三个不同的偏移**，各自分工。

### 分析

**核心矛盾**：平面反射算法假设镜面是无限大理想平面，但这个系统的"水面"——① Gerstner 波上下起伏（米级）；② 顶点在 shader 里被二次抬升（浅水抬升 + `_WaterFXMap` 位移）；③ 物体随波浮动。而反射平面是静态的 `y = 常数`。

若裁剪面精确贴住平均水位：斜投影把裁剪面以下的几何从反射纹理裁掉，波谷区域里"恰好低于裁剪面几厘米"的内容（船吃水部分、浮标底部、岸线）会在倒影中被切掉，且切口随波浪**时进时出地闪烁**——平面反射最经典的穿帮。

**三个偏移的分工**：

| 偏移 | 位置 | 作用 |
|---|---|---|
| `m_planeOffset` | `pos = target.position + up * m_planeOffset`（111 行） | 校正 target 的 pivot 不在水面高度的问题（编辑器未暴露的隐藏字段） |
| `m_ClipPlaneOffset`（0.07） | `d = -dot(n,pos) - m_ClipPlaneOffset`（119 行）→ 进 `CalculateReflectionMatrix` | **镜像平面**：围绕抬高 0.07 的平面镜像，倒影整体沿法向平移 2×0.07=14cm，补偿"视觉水面 ≠ 理论水面"的系统性偏差 |
| 硬编码 `0.1f` + 再次叠加 `m_ClipPlaneOffset` | `CameraSpacePlane(cam, pos - up*0.1f, ...)`（133 行） | **斜投影裁剪面**：抬高让波谷区内容留在反射纹理里；多渲染的水下部分无害（会被折射色/水色覆盖） |

**不对称取舍**：offset 太小 → 波谷穿帮、边缘闪烁；太大 → 镜像几何错位（倒影"浮"得离水面太远）、反射纹理透出更多水下内容。编辑器把 `m_ClipPlaneOffset` 滑块限制在 [-0.5, 0.5]（PlanarSettingsDrawer.cs:29），默认 0.07。波高大时需相应调大。

**验证方法**：把 `m_ClipPlaneOffset` 临时设 0，配一个有波幅的波浪配置，盯漂浮物吃水线的倒影——倒影底部会随波浪周期性被"咬掉一口"；恢复 0.07 后消失。

---

## 附录：阅读过程中发现的问题清单

| # | 问题 | 位置 | 影响 |
|---|---|---|---|
| 1 | 波浪数硬上限 10，超出数组越界 / ComputeBuffer 溢出 | Water.cs:242、GetWaveData() | 崩溃或渲染错误 |
| 2 | CPU/GPU `peak` 不一致（0.8 vs 1.5） | GerstnerWavesJobs.cs:170 vs GerstnerWaves.hlsl:40 | 浮力物体与视觉波浪对不上 |
| 3 | CPU 法线多乘 `amplitude` 且做了 normalize | GerstnerWavesJobs.cs:199-202 | CPU 法线方向偏 |
| 4 | 全向波在原点 `normalize(0,0)` → NaN → Rigidbody 位置 NaN | GerstnerWavesJobs.cs:181 / GerstnerWaves.hlsl:47 | 物体消失（CPU 浮力采样更易命中） |
| 5 | `qi` 分母无保护，amplitude=0 → Inf×0=NaN；`saturate` mask 防不住 | GerstnerWaves.hlsl:41、66 | 顶点 NaN、海面消失 |
| 6 | NonPhysical/NonPhysicalVoxel 模式 `_samplePoints` 从未写入，恒为 (0,0,0) | BuoyantObject.cs:209-215 | 所有物体在世界原点采样同一波高 |
| 7 | `NonPhysicalVoxel` 分支为空，模式未实现 | BuoyantObject.cs:138-140 | 该模式无效 |
| 8 | Domain 中引用不存在的 `v`（应为 `Input`） | WaterTessellation.hlsl:81-82 | instancing 变体可能编译失败 |
| 9 | `WaterTessellated.shader` RenderPipeline Tag 仍写 `LightweightPipeline` | WaterTessellated.shader:10 | URP 下可能不生效 |
| 10 | `FragWaveNormals()` 依赖不存在的 Renderer 和关键字 | Water.cs:183-190 | 调用会 NRE |
| 11 | `InfiniteWater.shader` 是未完成的 raymarch 实现 | InfiniteWater.shader:151 | 不可用于生产 |
| 12 | 深度烘焙 Layer 硬编码 `1 << 10`、范围 500×500、分辨率 1024 与 `posWS.xz*0.002` 强耦合 | Water.cs:341-397 | 改一个必须改另一个 |
| 13 | 焦散网格 Y 硬编码 0 | WaterSystemFeature.cs:88 | 水面不在 Y=0 时错误 |
| 14 | 浮力采样点全局容量 4096，超出静默失效 | GerstnerWavesJobs.cs | 物体浮力失效 |
| 15 | CPU/GPU 波形一致性依赖 `qi`/`waveCountMulti`/`peak`/时间项四处同步，任意单点改动都会破坏对齐 | 两文件 | 隐性回归风险 |

---

## 共性方法论小结

这几轮问答反复出现的几类技巧，值得抽出来沉淀：

1. **branchless 二值开关**：`onmiDir` 三处乘法替代 if 分支（问题 2）——SIMD/Burst/GPU 友好，代价是可读性与"严格 0/1"约束。
2. **参数语义重定义**：除以 N 不是改物理公式，而是改"用户输入 → 物理振幅"的映射（问题 4）。
3. **公式变形的副作用追踪**：`qi·a ≡ peak/(w·N)` 这种"约掉"是退化行为的根源（问题 5），看到分数里出现会被其他项抵消的因子要警觉。
4. **单帧等价 ≠ 动态等价**：±Time 在 t=0 波形相同、1 秒后反向（问题 3）；判断符号/参数是否"无所谓"，必须看时间演化。
5. **数据流向决定容错需求**：CPU 的死数据（xz）可以容忍 NaN，GPU 的活数据（顶点位置）必须保护（问题 5）。
6. **齐次表示的价值**：4 维平面/点让"点积判定 + 逆转置变换 + API 对接"三种运算统一（问题 8）。
7. **保守性权衡方向**：渲染管线里凡涉及"镜像/变换后的非法视角"（问题 7）或"裁剪边界与动态几何交界"（问题 9），一律选"宁可多画，不能漏画/缺块"。
