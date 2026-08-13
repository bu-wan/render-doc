# 05 · NRC 算法与着色器详解

## 1. NRC(Neural Radiance Caching)算法原理

### 1.1 问题背景

路径追踪的每个 bounce 都要做昂贵的光线求交与 BRDF 求值。**辐射缓存**的思想是:把"从某点、某方向看出去的辐射度"用函数近似缓存起来,路径经过缓存点时直接查缓存,避免继续追踪。

NRC(Real-time Neural Radiance Caching, NVIDIA 2021)用一个小型 **MLP(全连接网络)** 作为辐射缓存的函数近似:

```
入射辐射 L_i(x, ω_i) ≈ MLP(编码(x, ω_i, n, 材质属性))
```

其中输入是**手工特征编码**后的命中点信息(位置、出射方向、法线、粗糙度、漫反射/镜面反射色),输出是 RGB 辐射度。

### 1.2 前向渲染公式(本项目实现)

对每个路径点,辐射估计拆成两部分:

```
L ≈ bias(x, ω_i) + factor(x, ω_i) · MLP(x, ω_i)
```

- `bias`:路径追踪已经实际计算出的辐射(不完整,通常只走了很少 bounce);
- `factor`:路径累积的透射率(transmittance × BRDF 累计);
- `MLP`:神经网络的辐射缓存预测。

把路径拆成"已追踪部分 + 缓存补全部分":

```
PathTrace(bounce 1..k): radiance = Σ (accumulate_i · emission_i)
缓存补全:radiance += accumulate_k · MLP(input_k)     // 由 NNInference 完成
```

这正是 `PathTracerPass` 与 `NNInference` 的分工:PathTracer 只追踪前几个 bounce 并**记录**,NNInference 用 MLP 补全剩余部分。

### 1.3 训练目标(自监督)

训练数据来自路径追踪本身:**继续追踪更长的路径**得到"完整"的辐射作为标签。对每个路径点 i:

```
target_i = Σ_{j≥i} (colors[i→j] · lights[j])        // 从 i 点出发的完整辐射
predict_i = MLP(input_i)                              // 网络预测
loss = L2(predict, target)  (加相对亮度归一化)
```

本项目 `ExtendedPathTrace`(训练模式)追踪完整路径,把每个中间点的 `(input_i, target_i)` 打包进 `batch_train_records`,供 `NNGradient` 反向传播。

### 1.4 损失函数(相对亮度 L2)

```glsl
// NNLoadDA3_RelativeL2LuminanceLoss
float predict_luminance = dot(vec3(0.299, 0.587, 0.114), max(predict, vec3(0)));
vec3 d_loss = 2.0 * loss_scale * (predict - target) / (predict_luminance^2 + 0.01);
```

除以预测亮度平方(加小量防除零)即**相对误差加权**,让暗部与亮部同等重要,防止网络偏向学习高亮区域。

### 1.5 特征编码(manual features)

`NRCRecord.glsl` 实现论文的特征编码,把原始输入升维到 64 维 fp16:

| 输入 | 编码方式 | 维度 |
| --- | --- | --- |
| 位置 x,y,z(场景 [-1,1] 归一化) | **频率编码**:每轴 12 个三角波频率(1..2048),`mat3x4` | 3×12=36 |
| 出射方向(球坐标编码 2 维) | **一维 blob 编码**:4 段 quartic 基函数 | 2×4=8 |
| 法线(球坐标编码 2 维) | 同上 blob 编码 | 2×4=8 |
| 粗糙度 | `1-exp(-roughness)` 后再 blob 编码 | 4 |
| 漫反射色 / 镜面反射色 | 直接传入 | 3+3=6 |

**频率编码**(类似 NeRF 的 positional encoding):`_nrc_tri` 是三角波 `2|mod(x-0.5,2)-1|-1`,比 sin 更便宜,高频分量让网络能表达尖锐细节。

**Blob 编码**:quartic CDF 差分的分段平滑基函数:
```glsl
vec4 NRCOneBlob4Encode(float x) {
    vec4 l = vec4(0, .25, .5, .75), r = vec4(.25, .5, .75, 1);
    return _quartic_cdf(r - x, 4) - _quartic_cdf(l - x, 4);  // 4 个重叠的平滑脉冲
}
```

编码结果打包成 8 个 `uvec4`(= 64 个 fp16),即 MLP 的输入层宽度 64。

### 1.6 打包(Packing)设计

因为 MLP 输入/激活都在 **fp16**,且要喂给协作矩阵,所有中间量用 `packHalf2x16` 打包:

- `PackedNRCInput`:4 个 uint32 压缩一个完整输入样本:
  - `primitive_id`
  - `flip_bit_instance_id`(bit31 = 法线是否翻转,低 31 位 = instance_id)
  - `barycentric_2x16U`(packUnorm2x16 重心坐标)
  - `scattered_dir_2x16U`(packUnorm2x16 球坐标编码方向)
- `NRCEvalRecord{dst, packed_input}`:20 字节(dst 4B + PackedNRCInput 16B);dst 编码了"写到屏幕像素"或"写到训练批次区间";
- `NRCTrainRecord{6 float + PackedNRCInput}`:训练标签 + 输入。

> 着色器通过 `UnpackNRCInput`(编译开关 `NRC_SCENE_UNPACK`)从打包输入还原出顶点插值、法线、材质属性——**输入样本只存"三角形 ID + 重心 + 方向"三样东西,其余全部运行时重算**,极大省带宽。

### 1.7 Adam 优化器 + EMA

`nrc_optimize.comp` 实现每权重 Adam:

```glsl
gradient = uGradients[i] / uBatchTrainCount / LOSS_SCALE;   // 批平均 + 损失缩放
entry.moment = β1·m + (1-β1)·g;                              // 一阶矩
entry.moment.y = β2·v + (1-β2)·g²;                          // 二阶矩
h_moment = moment / (1 - β^t);                               // 偏差校正
weight -= lr · m̂ / (√v̂ + ε);
ema_weight = (1-α)/η_t · weight + α·η_{t-1}·ema_weight;      // EMA 更新(论文的梯度平均形式)
```

`optimizer_state`(UBO)维护 `t, β1^t, β2^t, α^t, α^{t-1}`,由 `nrc_train_prepare.comp` 每批递增。EMA 权重写进 `use_weights`,可被 UI 关闭(此时直接拷贝 weight)。

### 1.8 训练概率与批次

- 每帧默认 3% 的像素子组进入训练模式(见 02 文档 7.6);
- 每个训练像素贡献**整条路径的所有中间点**(最多 MAX_BOUNCE-1 个),原子追加进 4 个批次之一;
- 每批最多 16384 个样本,一帧最多 4×16384 样本;
- `NNTrain` 的 4 个批次**串行执行**(批次 b 的输出权重是批次 b+1 的输入),保证优化器状态连续。

## 2. 全融合 MLP(Fully-Fused MLP)

### 2.1 什么是全融合

tiny-cuda-nn 论文(以及本项目)的做法:把整个网络(前向 + 反向)写进**一个 kernel**,所有中间激活都留在**寄存器与共享内存**里,不落显存。对比传统"每层一个 kernel":
- 省掉每层的全局读写(带宽开销);
- 融合 ReLU、梯度、权重更新;
- 但寄存器/共享内存压力大,需要精心安排数据流。

### 2.2 布局与维度(NN_nv.glsl)

网络:输入 64 → 5 个隐藏层 64 → 输出 3。权重总数 = 64×64×5 + 64×3 = 20672。

```glsl
#define WORKGROUP_SIZE 128    // workgroup = 128 线程
#define SUBGROUP_SIZE  16/32  // 由编译时 -DSUBGROUP_SIZE 决定(特化两个版本)
#define SUBGROUP_COUNT (128 / SUBGROUP_SIZE)   // 4 或 8 个子组
#define FP_X 64               // 矩阵宽度(输入/激活宽度)
#define ACT_Y WORKGROUP_SIZE  // 激活高度 = 128(一批 128 个样本)
```

每个 workgroup 同时处理 **128 个样本**,每个样本是 64 维 fp16 激活。激活矩阵 `A(64×128)` 被切成协作矩阵块:

```glsl
fcoopmatNV<16, gl_ScopeSubgroup, 16, 16> act_coopmats[COOPMAT_X][SUBGROUP_ACT_COOPMAT_Y];
// COOPMAT_X = 64/16 = 4,SUBGROUP_ACT_COOPMAT_Y = 128/16/子组数
```

即每个子组持有激活矩阵的若干 16×16 块(每个线程持有 1/16 元素,协同完成乘法)。

### 2.3 协作矩阵前向(NNForward64_ReLU)

```
对每层:
  1. _nn_load_weight_64(layer):每线程把权重块载入共享内存
  2. 零初始化输出激活块
  3. coopMatLoadNV 把权重 16×16 块载入协作矩阵寄存器
  4. coopMatMulAddNV(w, src_act, dst_act) 完成 16×16 块乘加
  5. 遍历所有块组合,累加出整层输出
  6. 对输出块逐元素 ReLU(max(x,0))
```

**关键布局决策**:

```glsl
WEIGHT_COOPMAT_MAJOR = false; // 权重按 RowMajor 载入
ACT_COOPMAT_MAJOR    = true;  // 激活按 ColumnMajor 载入
```

配合 `MAT64_COOPMAT_ELEMENT(x,y) = x*2 + y*16*8` 的共享内存寻址,让 `coopMatMulAddNV` 的 A、B 矩阵方向正好与 Vulkan 的 NV 协作矩阵 C = A·B 语义匹配,避免转置拷贝。

### 2.4 反向传播(NNBackwardDA* / NNUpdateDW*)

反向传播按"计算图"逆序融合:

```
NNGradient main():
  1. 前向 5 层,保存每层激活 act_coopmats[0..5](全在寄存器/共享内存)
  2. NNLoadDA3_RelativeL2LuminanceLoss:从标签算输出层梯度 dA(转置布局)
  3. NNUpdateDW3: dW = (A·dA^T)^T,第 5 层
  4. NNBackwardDA3_ReLU: dA^T · W^T 反传(带 ReLU mask)
  5. 重复 NNUpdateDW64 / NNBackwardDA64_ReLU 直到第 0 层
```

**ReLU 反传的 NAN-mask 技巧**(`_nn_act_64_relu_mask_t`):
- 前向时 ReLU 后输出 `max(x,0)`;
- 反向时把激活矩阵重载回寄存器,`x>0 ? 0 : NaN`,再与 dA 相乘;
- `coopMatMulAddNV(NaN 块, dA, 0)` 得到 `NaN(失效) | 有效梯度`,最后 `isnan ? 0 : v` 一次指令完成 ReLU 导数掩码与矩阵乘融合。

**梯度跨子组归约**(`NNUpdateDW64`):
1. 每个子组算自己的局部 dW 块,`coopMatStoreNV` 写共享内存;
2. `barrier()` 后,每个线程把 `SUBGROUP_COUNT` 个子组的同位置 dW 累加(读共享内存);
3. 用 `GL_EXT_shader_atomic_float` 的 `atomicAdd(float)` 累加进全局 `uDWeights`(fp32 梯度缓冲)。

> 注意:梯度缓冲是 **fp32**(`float uDWeights[]`),因为多个 workgroup(多个样本批)需要原子累加,且 Adam 需要 fp32 精度;权重本身是 fp16(减少存储与矩阵乘带宽)。

### 2.5 为什么比 tiny-cuda-nn 快

- 单 kernel 融合前向+反向,无中间激活的全局内存往返;
- NV 协作矩阵的 fp16 张量核吞吐 >> 纯 CUDA 核心;
- 梯度用原子浮点累加直接跨 workgroup 归约,省掉二次归约 kernel;
- 共享内存复用:权重块、激活块、dW 临时块共用一个 `SHARED_BUFFER`,容量按需取最大者。

## 3. 各着色器逐一分析

### 3.1 `vbuffer.vert` / `vbuffer.frag` — V-Buffer 预通道

- 顶点着色器:位置 × 实例变换矩阵(`aModelT` 为 mat3x4,`vec4(aPosition,1)*aModelT`),输出 `vInstanceID = gl_InstanceIndex`;
- 片段着色器:输出 `uvec2(primitive_base + gl_PrimitiveID, instance_id)`;
- 每实例 push constant 更新 `primitive_base`(该实例的全局三角形起始 ID,与 TLAS 的 `instanceCustomIndex` 一致);
- 深度测试开启(近 0.01,远 4.0,与路径追踪 T_MAX 一致)。

### 3.2 `path_tracer.comp` — 主路径追踪

**subgroup 协作**:`local_size = 8×8`,整行(64 线程? 不,8×8=64)……

实际:`layout(local_size_x=8, local_size_y=8)`,每 64 线程一组;`subgroupAll(method == ...)` 保证**一个 subgroup 内所有像素选择相同方法/训练状态**,从而 `subgroupElect()` + `subgroupBroadcastFirst()` 让整个子组共享同一个训练决策(避免组内发散,也保证 atomicAdd 模式一致)。

**主流程**(每个像素):
1. 种子:`(coord.y&0xFF)<<8 | (coord.x&0xFF) + uSeed`(256×256 瓦片级复用种子,保证空间噪声独立但可复现);
2. 从 V-Buffer 取 `(primitive_id, instance_id)`;
3. 构造主光线:clip 坐标 → 相机基;`GetVBufferHit` 用**莫勒-特伦博尔算法**直接求交 V-Buffer 三角形(命中必中,只取参数 t);
4. 按方法分派:
   - `METHOD_NONE`:完整路径追踪(所有 bounce 都实际追踪),不写 eval record——作为参照真值;
   - `METHOD_NRC` / `METHOD_CACHE`:追踪到 c_a0 阈值后,记录 eval record 交给 NNInference(见 1.2);
   - 训练模式(概率 3%):`ExtendedPathTrace` 追踪完整路径并生成训练记录。

**路径终止启发式**:`c_a0 = C · dist² / (4π · cosθ)`,累计 `sqrt_a_sum`(各 bounce 的 `√(dist²/(pdf·cosθ))`)直到平方超过 c_a0——这是"继续追踪的期望方差贡献 ≤ 阈值"的近似,即**路径长度由几何/BRDF 自适应决定**(明亮或粗糙处追得短)。

**训练记录生成**(ExtendedPathTrace 后半段):
```
对每个 bounce 点 i:
  target_i = Σ_{j≥i} colors[i→j]·lights[j]   (从后往前累计:lights[i] += colors[i]·lights[i+1])
  写入 NRCTrainRecord{bias=target, factor=colors[i], input}
```
只有 `train_id < NRC_TRAIN_BATCH_SIZE` 才写入(防止批次溢出)。

### 3.3 `nrc_inference.comp` — NRC 推理

- 间接 dispatch:`dispatch_x = ceil(eval_count / 128)`;
- 每个 workgroup 处理 128 个 eval record,`NNLoadInput` 载入打包输入 → 5 层前向 → 输出 RGB;
- 屏幕目标(dst 类型 SCREEN):`color = bias_factor_r.rgb + vec3(bias_factor_r.a, factor_gb) * predict`,写回 `bias_factor_r`(即渲染图后续读取的"最终颜色"图像);
- 训练目标(dst 类型 TRAIN):把预测合入对应批次的 `bias` 字段(变成 `color = bias + factor·predict`),供 NNGradient 当标签用——**同一前向同时服务渲染与训练**;
- 未命中的线程(`gl_GlobalInvocationID.x ≥ eval_count`)用 `NRC_EVAL_INVALID_DST` 哨兵跳过输出,但**仍执行 MLP**(保持 subgroup 一致,避免掩码分支开销)。

### 3.4 `nrc_gradient.comp` — 反向传播(见 2.4)

间接 dispatch,每 workgroup 处理 128 个训练样本,原子累加梯度。`LOSS_SCALE=1.0` 由 Constant.glsl 提供(损失缩放,与优化器中的除法配对)。

### 3.5 `nrc_train_prepare.comp` — 批次准备

1×1 dispatch:
- 裁剪 `count = min(count, 16384)`,回写;
- 生成 `VkDispatchIndirectCommand{(count+127)/128, 1, 1}`(间接调度梯度核);
- 递增优化器状态:`t++, β1^t·=0.9, β2^t·=0.999, α^{t-1}=α^t, α^t·=0.99`。

### 3.6 `nrc_optimize.comp` — Adam + EMA(见 1.7)

64 线程/workgroup,dispatch `WeightCount/64 = 323` 组。`WRITE_USE_WEIGHTS` 编译开关决定是否同时更新 `use_weights`(最后一个训练批才开,由 NNOptimizer 的两个管线实例体现)。

### 3.7 `nrc_indirect.comp` — 间接调度生成

1×1:从 uniform count 生成 `{(count+127)/128,1,1}` 间接命令,供 NNInference 使用。

### 3.8 `screen.frag` — 色调映射与累积

- 输入:`subpassLoad(uColor)`(NNInference 输出的颜色,作为 input attachment);
- 累积:`color = (color + accumulate·count) / (count+1)`,写回 accumulate(仅当累积开启);
- 色调映射:Hejl 2015 Filmic,white point 3.2,再 gamma 1/2.2。

### 3.9 公共头文件

| 文件 | 内容 |
| --- | --- |
| `Constant.glsl` | M_PI、网络/批次常量、LOSS_SCALE、Adam/EMA 参数 |
| `RNG.glsl` | PCG(输出 RXS-M-XS 32/32)缩减版,`[0,1]` 均匀 |
| `Sample.glsl` | 余弦加权半球采样 + 切空间对齐 `AlignDir` |
| `LambertBRDF.glsl` | 理想漫反射 |
| `CookTorranceBRDF.glsl` | Cook-Torrance 微表面:Beckmann 法线分布、Smith G、Schlick Fresnel;采样 = 镜面(Beckmann 采样)+ 漫反射混合 |
| `Scene.glsl` | 场景缓冲声明 + 顶点/UV/材质取值(含纹理数组特化常量) |
| `NN_nv.glsl` | 全融合 MLP 前向/反向(核心,见第 2 节) |
| `NRCRecord.glsl` | 记录打包/解包、特征编码(见第 1 节) |

### 3.10 Cook-Torrance BRDF 细节

```glsl
// Beckmann 法线分布
D = exp((n·h²-1)/(a²·n·h²)) / (π·a²·(n·h)⁴)
// Walter 几何遮挡 G1(近似 Smith)
// Schlick Fresnel f0=(ior-1)²/(ior+1)²
// 采样:先按 Beckmann 分布采样微表面法线 h → reflect(-v,h) 得入射方向 l
// 以亮度占比 CT_Specular_Prob 混合镜面采样与余弦采样,PDF 用混合形式
```

采样与 PDF 严格对应(保证无偏),`roughness` 下限钳制 0.001,`ior` 下限 1.5(防退化)。

## 4. 数据流与一致性总结

| 阶段 | 生产者 | 消费者 | 数据 | 格式 |
| --- | --- | --- | --- | --- |
| V-Buffer | VBufferPass | PathTracer | (primitive, instance) | R32G32_UINT |
| 渲染目标 | PathTracer | NNInference/Screen | bias+factor | rgba32f/rg32f |
| eval 记录 | PathTracer | NNInference | packed input + dst | 20B/条 |
| 训练记录 | PathTracer(标签)+ NNInference(预测) | NNGradient | bias/factor + input | 40B/条 |
| 梯度 | NNGradient | NNOptimizer | fp32 梯度 | 20672×4B |
| 权重 | NNOptimizer | NNInference | fp16 权重 | 20672×2B |
| 颜色 | NNInference | ScreenPass | RGB | rgba32f |
