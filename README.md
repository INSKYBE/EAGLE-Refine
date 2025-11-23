# EAGLE-Refine
# EAGLE-Refine

## 第一阶段：双驱动逻辑仿真 (Level 0 & Level 1)

核心任务： 将语义转化为结构化真值 (Geometric Ground Truth)。

硬件分配： CPU (物理计算/Blender) + GPU (LLM/MDM 推理)。

### **Step 0: 智能规划与配置生成 (The Brain)**

执行者： LLM (GPT-4 / DeepSeek)

任务： 充当“编剧”和“导演”，定义异常情节以及物理约束规则。

我们不再写死代码，而是让 LLM 输出 **结构化 JSON 配置**。

- **输入 Prompt 指令：**

  > "生成 50 个异常场景描述。对于每个场景，判断其类型（物理意外/主观意图），并定义该动作成立必须满足的几何约束（如接触、朝向、避让）。"

- **输出示例 (JSON):**

  JSON

  ```
  [
    {
      "id": "case_001",
      "prompt": "A person slipping backwards on an icy patch.",
      "driver_type": "PASSIVE_PHYSICS", // 路由标志
      "perturbation": {"target": "friction", "value": "low"}
    },
    {
      "id": "case_002",
      "prompt": "A thief crouching to force a car door open.",
      "driver_type": "ACTIVE_INTENT", // 路由标志
      "constraints": { // 通用约束协议
        "type": "TOUCH",
        "subject_bone": "RightHand",
        "target_object": "CarHandle"
      }
    }
  ]
  ```

### **Step 1: 任务分发 (The Router)**

执行者： Python 脚本 (Task Dispatcher)

逻辑： 读取 JSON 中的 driver_type 字段。

- `PASSIVE_PHYSICS` $\rightarrow$ 进入 **驱动 A**。
- `ACTIVE_INTENT` $\rightarrow$ 进入 **驱动 B**。

### **Step 2 - 驱动 A: 被动物理异常仿真 (Physics Driver)**

适用场景： 滑倒、绊倒、坠落、碰撞。

核心机制： 临界点搜索闭环 (Critical Point Search Loop)。

工具： Blender Rigid Body (刚体物理)。

#### **详细执行流程：**

1. **环境初始化：** Blender 加载 SMPL 人体，根据 Prompt 加载简单几何体（如地面、障碍物）。
2. **正常动作回放：** 0-30 帧播放标准 Mixamo 走路动画（Kinematic 模式）。
3. **物理切换 (Ragdoll Switch):** 第 31 帧，将骨骼切换为 **Dynamic (刚体模式)**，接管控制权。
4. **闭环运行 (The Loop):**
   - **尝试 1:** 注入扰动（如摩擦力 $\mu=0.5$）。
   - **检测:** 检查 Agent 盆骨高度 $H$。
     - 如果 $H > 0.8m$ (没倒) $\rightarrow$ **反馈:** 降低摩擦力 $\mu$，重置场景，重跑。
     - 如果 $H < 0.5m$ (倒了) $\rightarrow$ **Success**。
5. **输出:** 记录下这一段真实的摔倒骨架序列。

### **Step 3 - 驱动 B: 主动意图异常仿真 (Intent Driver)**

适用场景： 偷窃、潜伏、交互、破坏。

核心机制： LLM指导生成 + 通用约束求解 + 语义校验。

工具： MDM (Motion Model) + Blender Python。

#### **详细执行流程：**

##### **3.1 动作生成 (Generation)**

- 调用 **MDM (Motion Diffusion Model)**，输入 Prompt，生成 `.npy` 格式的 3D 骨架序列。

##### **3.2 逆向对齐与几何闭环 (Universal Constraint Solver)**

- **核心思想：** 不写死具体场景脚本，只写一个通用的 `solve_constraint` 函数。
- **执行：** 解析 Step 0 中的 `constraints` JSON。
  - 如果类型是 `TOUCH` (接触):
    - 脚本计算骨骼与目标物体的距离 $D$。
    - **微闭环 (Geometry Loop):**
      - 如果 $D > Threshold$: 调用 Blender 的 **IK (反向动力学)** 系统，将骨骼末端牵引至目标物体表面。
      - 同时，将目标物体（如车门模型）移动到适配骨架的位置（逆向布局）。
    - **交互响应触发 (Interaction Triggers):**    - **逻辑：** 当 `subject_bone` 与 `target_object` 接触且 IK 牵引力大于阈值时，激活物体的预设物理状态。    - **执行：** 触发 `constraint.reaction` (如：门从 `Closed` 状态切换为 `Open` 动画，或窗户模型替换为 `Shattered` 碎片模型)。确保“拉门”不仅仅是手放在门把手上，而是门真的开了。

##### **3.3 语义闭环 (Semantic Loop via VLM)**

- **问题：** 防止生成的动作像“做操”而不像“偷窃”。
- **执行：**
  1. **灰模渲染:** Blender 极速渲染出只有几何体和灰人的视频。
  2. **VLM 裁判:** 喂给 Video-LLaVA。
     - *Q:* "Does the person in this simulation look like they are [prompt_action]? Is it realistic?"
  3. **反馈:**
     - **Pass:** 输出骨架。
     - **Fail:** VLM 反馈 "Action too stiff"，回传给 LLM 修改 Prompt (如增加 "hunched back")，重跑 3.1。

### **Step 4: 数据标准化输出 (The Output)**

无论来自驱动 A 还是 B，最终输出格式完全一致，以便对接第二阶段（SVD）。

**对于每个样本，保存：**

1. **Pose Sequence:** OpenPose 格式的 RGB 视频（黑底彩线，标记关节位置）。
2. **Depth Map Sequence:** 16位 灰度深度图视频（标记物体前后关系）。
3. **Normal Map (可选):** 法线图（增加表面几何细节）。
4. **Segmentation Map Sequence :** 语义分割图视频。
   - **逻辑：** 在 Blender 渲染时，给“头部”模型分配纯红色 (RGB: 255, 0, 0)，给“手部/交互物”分配纯蓝色 (RGB: 0, 0, 255)，背景为黑色。这将用于第二阶段的强语义控制。
5. **Meta Data:** 原始 Prompt，以及该样本是“物理驱动”还是“意图驱动”的标签。

------

## 第二阶段：视觉映射与生成 (Visual Mapping & Generation)

### **1. 设计目标**

- **保真 (Fidelity):** 严格遵循第一阶段输出的骨架（动作）和深度图（空间结构），不能乱动。
- **真实 (Realism):** 生成符合 CCTV 监控风格的画质（噪点、广角、夜视感）。
- **一致 (Consistency):** 人物和环境在时间轴上不能闪烁、变形。

### **2. 核心模型选型**

为了在 A100 上获得最佳的可控性和效果，推荐使用以下组合：

- **底座模型:** **AnimateDiff (基于 SD v1.5 或 SDXL)** 或 **SVD-XT (Stable Video Diffusion)**。
  - *推荐理由:* AnimateDiff 的 ControlNet 生态最成熟，对骨架的控制最精准。
- **结构控制:** **ControlNet (v1.1)**。
- **特征控制:** **IP-Adapter (Image Prompt Adapter)**。

### **3. 详细模块拆解**

#### **模块 A: 多模态条件注入器 (Multi-Modal Injector)**

这是本阶段的“入口”。我们需要把第一阶段的各种数据“喂”给生成模型。

1. **结构条件 (Structure Condition):**
   - **输入:** `Pose Sequence` (骨架) + `Depth Map` (深度)。
   - **处理:**
     - **ControlNet-OpenPose:** 权重设为 **1.0**。这是“硬约束”，强制生成的视频里人物动作必须和骨架一模一样。
     - **ControlNet-Depth:** 权重设为 **0.6 - 0.8**。这是“软约束”。
       - *Why?* 深度图只告诉模型“这里有个方块”，我们需要模型发挥想象力把它渲染成“车门”或“柜子”。权重太高会导致生成的车像个水泥块；权重太低会导致车形变。
2. **外观条件 (Appearance Condition):**
   - **输入:** `Reference Image` (参考图)。
   - **处理:** 使用 **IP-Adapter**。
     - *场景:* 输入一张真实停车场的照片。
     - *人物:* 输入一张戴黑色头套、穿卫衣的劫匪照片。
   - *作用:* 解决“文本控制不稳定”的问题。确保生成的每一帧里，劫匪都戴着同一个款式的头套。
3. **文本条件 (Text Condition):**
   - **输入:** LLM 生成的 Prompt + **风格后缀**。
   - *Prompt:* "A suspicious man crouching near a car..."
   - *Suffix (关键):* "**CCTV footage, security camera, surveillance style, low quality, grainy, fisheye lens, high angle view, night mode.**"
   - *作用:* 强行将画风统一为“安防监控”风格。

#### **模块 B: 去噪生成引擎 (Denoising Generation Engine)**

这是 A100 算力消耗最大的部分。

- **初始化:** 从纯高斯噪声 (Gaussian Noise) 开始。
- **去噪循环 (The Loop):**
  - 在每一个 Time Step ($t=T \to 0$)：
    1. **U-Net 预测:** 模型根据当前的噪声 + ControlNet 的引导 + IP-Adapter 的特征，预测这一步该怎么画。
    2. **Temporal Attention (时序注意力):** AnimateDiff 的核心模块。它会查看“前后几帧”，确保生成的像素在时间上是连贯的（防止衣服颜色变来变去）。
  - **背景冻结 (Background Locking via Mask):**  - **逻辑:** 既然是固定机位 CCTV，背景不应闪烁。利用 Step 4 输出的 `Segmentation Map` 中的“背景区域（黑色）”作为 Mask。  - **执行:** 在每一步去噪后，将第一帧生成的背景 Latent 强制回填到当前帧的背景区域。仅对前景（人物+交互物）进行动态去噪。这能极大消除 AnimateDiff 常见的背景流动伪影。
- **输出:** 潜在空间 (Latent Space) 的视频张量。

#### **模块 C: 像素解码与后处理 (Pixel Decoding & Post-Processing)**

- **VAE Decode:** 将 Latent 张量解码为 RGB 像素视频。
- **Deflicker (去闪烁 - 可选):** 如果生成的视频背景有细微闪烁，可以使用光流法 (Optical Flow) 进行简单的平滑处理。

------

## 第三阶段：裁判引导的对抗修正闭环 

### **1. 核心设计哲学**

本阶段不再仅仅是“质量过滤器”，而是一个**智能优化器 (Intelligent Optimizer)**。采用了 **Solver-Critic (求解者-裁判)** 范式：

- **Solver (生成器):** 负责生成视频假设（Hypothesis）。
- **Critic (裁判组):** 负责发现逻辑谬误，并提供**可执行的修正梯度 (Actionable Gradient)**，指导 Solver 进行迭代。

### **2. 层级化多模态裁判系统 (Hierarchical Multi-Modal Critic System)**

我们将裁判组形式化为三个“专家智能体”，按照计算成本由低到高、逻辑深度由浅入深的顺序运行。

#### **专家 A: 几何一致性裁判 (The Geometric Critic)**

* **职责:** 确保 2D 视频中的人物结构严格遵循 3D 仿真真值，防止肢体崩坏。
* **工具:** **OpenPose / MMPose**。
* **输入:**
  1.  `GT_Pose`: 第一阶段 Blender 输出的标准骨架序列。
  2.  `Gen_Video`: 第二阶段生成的视频。
* **判据:** 计算 `GT` 与 `Pred` 之间的 **PCK (Percentage of Correct Keypoints)**。
  * 若 PCK < 0.8 (关键点漂移严重) $\rightarrow$ 判定为 **结构崩坏 (Structural Collapse)**。

#### **专家 B: 语义与风格裁判 (The Semantic Critic) **

* **职责:** 确保视觉元素符合 CCTV 分布（低画质、遮挡）及特定语义（如蒙面）。
* **策略:** **关键帧快筛 (Keyframe Screening)**。利用 Step 4 的 Meta Data 锁定关键帧。
* **工具:** **Image-VLM** (如 LLaVA-NeXT / GPT-4o-Mini)。
* **Prompt 清单:**
  1.  *Style Check:* "Is the image quality low, grainy, and consistent with a CCTV surveillance camera?"
  2.  *Identity Check:* "Is the person's face visible? (Expected: NO, must be masked)."
* **判据:** 若回答与预期不符 $\rightarrow$ 判定为 **语义冲突 (Semantic Conflict)**。

#### **专家 C: 世界模型与物理裁判 (The World Model & Physics Critic)**

* **职责:** 确保视频内容**符合现实世界规律**。这是本框架的核心逻辑大脑。 

* **关注点:** **机理 (Mechanism)**。判断视频“运行逻辑”真不真。 

* **核心能力:**   

  1. **常识与功能性 (Affordance):** 检查物体属性。例如：门把手在右边，门必须向左开；杯子必须放在桌面上而不是悬空。    
  2. **物理定律 (Physical Laws):** 检查重力、碰撞、穿模。例如：手不能穿过闭合的玻璃；人跳起后必须下落。    
  3. **因果逻辑 (Causality):** 检查动作后果。例如：手接触玻璃瞬间，玻璃才应该碎，而不是接触前就碎。 

* **策略:** **基于元数据的思维链推理 (Metadata-Guided CoT)**。利用 Stage 1 Blender 导出的 `constraints` JSON 作为“世界真值”。

* **工具:** **Video-LLM** (如 Video-LLaVA / GPT-4o-Vision)。 

* **CoT Prompt 示例 (针对拉车门场景):**   

  > **Context from Blender:** "Object: Car Door. Hinge: Left. Handle: Right. Action: Pull to Open."   
  >
  > **Question:** "Analyze the video based on real-world physics:    
  >
  > 1. **Affordance Check:** Is the handle visually located on the RIGHT side? Does the door rotate along the LEFT axis?    
  >
  > 2. **Physics Check:** Does the hand firmly touch the handle without passing through it (clipping)?    
  >
  > 3. **Causality Check:** Does the door open *only after* the hand applies force?" 
  >
  >    **判据:** 任意步骤为 NO $\rightarrow$ 判定为 **世界模型幻觉 (World Model Hallucination)**。

### **3. 错误-修正映射机制 (Error-Correction Mapping Mechanism)**

| 裁判反馈 (Critic Feedback)                               | 错误类型 (Error Type)                  | 回溯深度       | 修正策略 (Refinement Strategy)                               |
| :------------------------------------------------------- | :------------------------------------- | :------------- | :----------------------------------------------------------- |
| **"Skeleton mismatch"**                                  | **结构崩坏**                           | **Short Loop** | 提高 `ControlNet-Pose` 权重；降低去噪强度。                  |
| **"Face visible / Not CCTV style"**                      | **语义冲突**                           | **Short Loop** | 增强 Prompt 权重；调整 IP-Adapter 参考图。                   |
| **"Handle position wrong / Gravity fail"**<br>(违背常识) | **世界模型幻觉**<br>(Affordance Error) | **Short Loop** | **加强 `ControlNet-Depth` 权重**。<br>*原理:* 深度图包含了门把手的准确几何凸起，强制模型按深度图生成，就能修正位置错误。 |
| **"Hand clips through door"**<br>(违背物理)              | **世界模型幻觉**<br>(Physics Error)    | **Long Loop**  | 回溯 Blender，调整碰撞体积，重新生成骨架。                   |
| **"Door opens before touch"**<br>(违背因果)              | **世界模型幻觉**<br>(Causality Error)  | **Long Loop**  | 回溯 Blender，调整 **交互触发器 (Interaction Trigger)** 的时间点，确保同步。 |

### **4. 闭环工作流执行 (Workflow Execution)**

1.  **生成 (Generate):** Stage 1 & 2 输出初始视频 $V_0$。
2.  **评估 (Evaluate):** - 先过 **专家 A** (几何)。Fail $\to$ 查表修正。 Pass $\to$ 继续。
    - 再过 **专家 B** (语义)。Fail $\to$ 查表修正。 Pass $\to$ 继续。
    - 最后 **专家 C** (因果)。Fail $\to$ 查表修正。
3.  **修正 (Refine):** 根据映射表调整参数，生成 $V_1$。
4.  **终止 (Terminate):** - 当所有专家通过 $\to$ 入库 (Positive Sample)。
    - 若计数器 > 3 且裁判仍不通过：
      1.  **降级策略 (Soft Fallback):** 简化场景（如去掉复杂的交互动作，改为简单的靠近）。
      2.  **熔断策略 (Circuit Breaker):** 标记为 **Hard Negative (无效样本)** 并丢弃，防止死循环。
