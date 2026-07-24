# 人形 Loco-Manipulation 评测框架穷尽式调研：自建 vs 采用

> 调研时间：2026 年 7 月 · 触发原因是发现 SIMPLE 与当时的自建方案高度重合
> ⚠️ 本文"以 fork SIMPLE 为底座"的核心建议**后来被推翻**（渲染吞吐原因），见 [README](README.md)

## TL;DR

- **不要从零自建。** SIMPLE（arXiv:2606.08278，USC PSI-Lab，MIT）与自建目标在架构层面几乎 1:1 重合——MuJoCo 物理 + Isaac Sim 渲染的双仿真、G1 全身 loco-manipulation、原生 benchmark VLA 与 WAM、LeRobot 数据格式、client-server 策略评测。
- **但一个关键假设需要纠正**：其 README 路线图里「Integrate SONIC whole-body controller」目前仍是**未勾选的 TODO**；仿真中实际接的是 **decoupled WBC**，完整的 GEAR-SONIC universal-token 控制器只在其姊妹项目 Psi0 里、且只用于**真机**。
- **WAM 评测存在两条互不相同的技术路线**：一条是「把 WAM/VLA 当普通策略丢进物理仿真跑成功率」（SIMPLE、Isaac Lab-Arena 走这条，对 WAM/VLA 公平可比），另一条是「用 world model 本身当评测器」（WorldEval、RoboWorld、EWMBench、GigaWorld-1/WMBench，评的是生成视频质量而非任务成功）。两者不可混为一谈。

## Key Findings

**证据强度标注**：强 = 有一手来源直接佐证；中 = 有二手可靠来源或需推断；推测 = 无直接证据。

1. **SIMPLE 是与目标重合度最高的开源项目（强）**。据项目页 psi-lab.ai/SIMPLE 及论文，它"耦合 MuJoCo 的精确接触动力学与 Isaac Sim 的照片级渲染"，并提供"包含 60 个多样化全身任务、50 个室内场景、1000+ 物体资产的大规模环境"。基于 IsaacSim 4.5 + MuJoCo 3.3，支持 franka / aloha 双臂 / dexmate 轮式 / Unitree G1 多本体，约 185 stars / 16 forks。
2. **SIMPLE 是研究级、非成品（强）**。仅 26 个 commit，无 release，文档自述 WIP，依赖栈重（IsaacSim + MuJoCo + CuRobo + uv/Nix/Docker）。
3. **没有任何单一项目能同时覆盖全部诉求**（人形 + 桌面双臂 + VLA + WAM + SONIC + sim/real 一致）。
4. **双仿真（一个引擎算物理 + 另一个做渲染）已是明确成型的趋势（强）**，代表项目包括 SIMPLE（MuJoCo+Isaac Sim）、DISCOVERSE（MuJoCo+3DGS）、RoboGSim（3DGS+Isaac Sim）、SplatSim（3DGS+MuJoCo MJX），以及通用抽象层 RoboVerse/MetaSim。
5. **NVIDIA 官方栈（Isaac Lab-Arena + GR00T-WholeBodyControl/SONIC）天然对齐（强）**，Arena 已内置 `isaaclab_arena_g1` 且原生集成 GR00T / OpenPi / DreamZero 策略；SONIC 的 PyTorch checkpoint 明确支持 Isaac Lab 评测。

## A. 人形 loco-manipulation 评测框架

**SIMPLE（USC PSI-Lab）——核心候选**
- GitHub: physical-superintelligence-lab/SIMPLE，MIT，约 185 stars/16 forks，无 release，文档 WIP
- 双仿真架构：MuJoCo 处理接触物理与高频机器人控制，Isaac Sim 每步同步物理状态并做光追渲染（`--render-hz=50`）。数据生成时可用 `--sim-mode=mujoco_isaac` 同时驱动两个引擎
- **已知局限（论文明确列出）**：① 渲染吞吐低，单 GPU 光追约 **4 fps**；② 全刚体假设，无法仿真布料/绳子/食物等软体
- 策略接入：decoupled client-server。模型推理跑在 server（Psi-0 仓库），SIMPLE client 跑仿真环境。数据与评测环境走 LeRobot 格式
- **SONIC 状态（已核实）**：README 路线图 `- [ ] Integrate SONIC whole-body controller` 未勾选；仿真里用的是 decoupled WBC。完整 GEAR-SONIC 只在 Psi0 仓库、且 news「[2026-06-13] Released SONIC integration for Psi-0」表明是真机流程
- 原生 benchmark 跑了：Psi0、GR00T N1.6、OpenPi π0.5、InternVLA-M1、H-RDT、DreamZero（WAM）、EgoVLA、Diffusion Policy、ACT、Cosmos3（WAM）。每任务分 Level 0/1/2 三级 OOD（视觉+干扰物 / 光照 / 目标位姿扰动），每级 10 次 trial
- 硬件需求：Ubuntu 22.04、CUDA 12.x、Python 3.10、RTX 级 GPU（推荐 4090 16GB+）

**HumanoidBench（UC Berkeley）——可参考**
约 599–772 stars，v0.2.0（2026-07-02）新增 G1。MuJoCo，H1+Shadow Hands，27 个任务（12 locomotion + 15 whole-body manipulation）。定位是 **RL 算法 benchmark**，状态观测为主、无原生 VLA/WAM 接入。

**HumanoidMimicGen loco-manipulation benchmark（arXiv:2605.27724）——可参考**
robosuite + MuJoCo，9 个 G1 任务，沿三轴变化（base 运动量 / 交互复杂度 / 时长），二元成功判定。

**LocoMuJoCo、HumanoidVerse、ProtoMotions、AGILE/WBC-AGILE**——均偏 locomotion 训练或 motion imitation，非 loco-manip 任务评测。AGILE 的四阶段工作流（交互式调试 GUI → 训练 → 统一评测 → descriptor 驱动部署）对方法论有参考价值。

**BEHAVIOR Robot Suite / BEHAVIOR-1K（Stanford）——参考价值有限**
本体是 Galaxea R1（轮式双臂 + 4-DoF 躯干，非双足人形）。基线 WB-VIMA 在 5 项家务任务、每策略 15 次随机化 rollout 下，端到端任务成功率比 DP3 高 13 倍、比 RGB-DP 高 21 倍。支撑 NeurIPS 2025 首届 BEHAVIOR Challenge。

## B. 双仿真 / 混合仿真架构

- **趋势判断（中）**：核心动机是"没有单一引擎同时擅长接触物理和照片级渲染"。SIMPLE 的贡献在于把它工程化为生产者-消费者流水线并跑通 zero-shot sim-to-real
- **DISCOVERSE**（arXiv:2507.21981）：MuJoCo 物理 + 3DGS 渲染，主打 Real2Sim2Real
- **RoboGSim**（arXiv:2411.11839）：3DGS 重建 + Isaac Sim 数字孪生 + VLA 闭环评测器
- **SplatSim / SplatMesh**（arXiv:2506.04120）：3DGS 外观 + MuJoCo(MJX) 可微物理，ALOHA 2 双臂验证
- **RoboVerse/MetaSim**（arXiv:2504.18904，约 1.78k stars，Apache-2.0）：不是双仿真的具体实现，而是**跨仿真器抽象层**，显式提供 hybrid simulation（把先进物理引擎和优秀渲染器配对）、cross-simulator、cross-embodiment 三种能力。**接口设计值得直接借鉴**
- **优缺点（中/推测）**：优点是各取所长、渲染多样性支撑感知泛化；缺点是状态同步复杂、渲染吞吐成瓶颈、两套资产维护成本翻倍、调试难度高

## C. 与 GEAR-SONIC / GR00T-WholeBodyControl 集成

- **GR00T-WholeBodyControl（NVlabs）**是官方栈：含 ① Decoupled WBC（下身 RL + 上身 IK，用于 GR00T N1.5/N1.6）；② GEAR-SONIC 系列（42M 参数行为基础模型，100M+ 帧人类动作数据；universal-token 控制器输出 64 维 latent motion token，50 Hz 运行；C++/TensorRT 部署；PyTorch checkpoint 支持 Isaac Lab 评测与续训）；③ MotionBricks
- **GR00T N1.7** 通过 `UNITREE_G1_SONIC` embodiment tag 支持 SONIC。完整 collect→finetune→deploy 工作流仅对 GEAR-SONIC 支持（`UNITREE_G1` tag 兼容 decoupled WBC 但不支持端到端工作流）
- **两条路线（强）**：SIMPLE 仿真走 decoupled WBC 路线；NVIDIA 官方 N1.7 + Isaac Lab 走 SONIC universal-token 路线
- **Arena 是否已接 SONIC**：未找到直接证据（**没有找到证据**）。已确认 Arena 集成了 GR00T/OpenPi/DreamZero 策略，且 SONIC checkpoint 支持 Isaac Lab 评测，但二者是否已打通端到端需实测
- `isaaclab_arena_g1` 内容（强）：G1 humanoid embodiment + 示例，官方文档示例任务为"G1 搬运箱子到托盘"的 loco-manipulation

## D. World Action Model (WAM) 的评测框架

**关键方法学区分（强）**：WAM 评测有两个完全不同的含义：

1. **把 WAM 当策略评（任务成功率）**——与 VLA 同场竞技。SIMPLE 已这么做（DreamZero、Cosmos3 与 VLA 同表比成功率）。**这是主线**，对 VLA/WAM/传统方案天然公平。当模型输出 latent action 时，接口答案就是 **latent action → 解码器/WBC → 关节指令**，评测层只认最终任务成功
2. **用 world model 当评测器**——评的是生成视频质量而非真实任务成功：
   - **EWMBench**（AgiBotTech，arXiv:2505.09694）：开源，三维度指标（视觉场景一致性、运动正确性、语义对齐），支撑 AgiBot World Challenge 的 WM track
   - **WorldEval / Policy2Vec**（arXiv:2505.19017）：把视频生成模型变成 world simulator，用 policy 的 latent embedding 注入，评策略排序与真实成功率相关性
   - **RoboWorld**（arXiv:2607.01060）：自回归视频 world model + task-progress-aware VLM 打分，与真实评测相关性 Pearson r=0.989、Spearman ρ=0.970
   - **GigaWorld-1 / WMBench**（arXiv:2607.02642）：用真机遥操作数据 + 匹配的策略 rollout 构建
   - **MotionWAM**（arXiv:2606.09215，Mondo Robotics + HKUST）：不是评测框架，是 WAM 模型本身（real-time 人形 loco-manip，G1），可作为**被测对象**

## E. 全身 VLA 相关工作与其评测协议

- **WholeBodyVLA**（arXiv:2512.11047，ICLR 2026）：统一 latent VLA，AgiBot X2 人形上验证，比基线提升 21.3%
- **TrajBooster**（arXiv:2509.11839）：跨本体轨迹中心学习，10 分钟遥操作数据即可
- **WOLF-VLA**（arXiv:2606.25591）：全身人形 locomotion 的 VLA benchmark，承诺开源数据集+checkpoint+仿真评测套件（**需核实是否已放出**）
- **GR00T N1.5/N1.7、Being-H0.7、Humanoid-VLA、LDA-1B、EgoScale**：各用自建协议，**未形成统一共识任务集，也无公开统一 leaderboard**（强）

## F. 跨本体统一评测框架

- **Isaac Lab-Arena（NVIDIA + Lightwheel，Apache-2.0）**：composable「LEGO」式任务组合，GPU 并行 rollout，RTX 渲染，LeRobot 兼容数据集，原生支持 GR00T N1x/π0/SmolVLA/ACT/Diffusion，含 G1/GR1/Galileo 人形。**是目前唯一能同时服务人形 loco-manip 与桌面操作、且工业级维护的开源评测框架**
- **RoboVerse/MetaSim**：跨本体统一接口，可作为"一套 config 多后端"的备选抽象层
- **Genie Sim 3.0（AgiBotTech，MPL-2.0）**：基于 Isaac Sim。据 CES 2026 官方新闻稿，benchmark 覆盖 200+ 任务、100,000+ 仿真场景，并提供 10,000+ 小时合成数据集。**若重度使用 AgiBot G2，Genie Sim 是最贴合的官方评测栈**

## G. 中国生态

- **宇树 Unitree（强）**：`unitree_rl_gym`（BSD-3，Go2/H1/H1_2/G1，Isaac Gym 训练 + MuJoCo sim2sim + 真机部署）、`unitree_rl_mjlab`（基于 mjlab，MuJoCo 后端）。均为 **locomotion RL 训练脚手架，非 loco-manip 评测 benchmark**
- **智元 AgiBot（强）**：Genie Sim 3.0 + EWMBench + AgiBot World 数据集，AgiBot World Challenge 2026 用 Genie Sim Benchmark + EWMBench 作评测底座，线下决赛用 G2 人形
- **北京人形创新中心 X-Humanoid（强）**：RoboMIND（arXiv:2412.13877，107k 演示轨迹、479 任务、96 类物体，四本体）、慧思开物平台、Tien Kung-Lab 运控框架、ArtVIP 高保真仿真资产（被 NVIDIA Isaac Sim 收录）、Labimus（化学实验室人形仿真评测平台）。牵头制定《人形机器人智能化分级》等标准
- **OpenLoong（人形机器人上海公司，开放原子基金会）**：青龙全尺寸开源人形，MPC+WBC 控制框架部署在 MuJoCo，格物仿真平台
- 上海 AI Lab InternRobotics、银河通用、星海图、傅利叶、开普勒、乐聚等：**未检索到成规模的开源人形评测框架**（没有找到证据，不排除存在）

## 趋势分析

1. **双仿真架构已从论文走向工程标配（中）**
2. **人形评测标准化仍是空白，谁都想当那个标准（中）**。SIMPLE、Isaac Lab-Arena、Genie Sim 都在争这个位置，但都还没赢
3. **WAM 评测方法学分裂（强）**
4. **NVIDIA 正在用 Isaac Lab-Arena + GR00T-WBC/SONIC 收编整个人形评测-控制栈（强）**

## 总览对比表

| 项目 | 机构 | 仿真后端 | 本体 | 任务数 | VLA | WAM | 接 SONIC | 开源可用 | License |
|---|---|---|---|---|---|---|---|---|---|
| **SIMPLE** | USC PSI-Lab | MuJoCo+Isaac Sim | G1/franka/aloha/dexmate | 60 | ✅ | ✅ | ❌仅 decoupled WBC | 研究级 | MIT |
| **Isaac Lab-Arena** | NVIDIA+Lightwheel | Isaac Sim | G1/GR1/Galileo+桌面 | 可组合 | ✅ | ✅ | ⚠️未证实 | 工业级 | Apache-2.0 |
| **Genie Sim 3.0** | AgiBot | Isaac Sim | G2/人形+桌面 | 200+ | ✅ | 部分 | ❌ | 工业级 | MPL-2.0 |
| **HumanoidBench** | UC Berkeley | MuJoCo | H1/G1+Shadow Hands | 27 | ❌ | ❌ | ❌ | 成熟(RL) | — |
| **GR00T-WBC** | NVlabs | Isaac Lab | G1 | 控制器 | ✅ | — | ✅原生 | 官方 | NV Open Model |
| **RoboVerse/MetaSim** | PKU/UCB | 多后端抽象 | 跨本体 | 大量 | ✅ | ✅ | ❌ | 活跃 | Apache-2.0 |
| **EWMBench** | AgiBot | (WM 评测器) | — | — | — | ✅评生成 | — | 开源 | — |
| **unitree_rl_gym** | Unitree | IsaacGym+MuJoCo | Go2/H1/G1 | locomotion | ❌ | ❌ | ❌ | 成熟 | BSD-3 |

## 当时的建议（部分已被推翻）

### 必须改
1. **纠正「SIMPLE 已内置 SONIC」的假设**。若坚持用 GEAR-SONIC，需明确：要么自己把 SONIC 接进 SIMPLE 仿真回路，要么走 Isaac Lab-Arena + GR00T-WBC 官方链路
2. **把「WAM 评测」拆成两条明确子线**

### 建议改
3. **自建范围应收窄到「集成层 + 任务/判定标准 + 数字被认可的机制」**：
   - **不要自建**：双仿真物理-渲染流水线、G1 loco-manip 任务集与 OOD 分级、LeRobot 数据管线
   - **应自建**：统一 policy server 抽象层；跨栈的成功判定与指标口径统一；自研机器人本体接入
4. ~~**混合方案**：SIMPLE 做 G1 loco-manip（fork 后改造）+ Isaac Lab-Arena 做桌面操作~~ ← **此建议后来被推翻**
5. **借鉴 RoboVerse/MetaSim 的 config 抽象**

### 触发重新决策的阈值
- 若 SIMPLE 渲染吞吐（~4 fps）导致评测周期不可接受 → 退回 Isaac Lab-Arena 单栈 ← **后来正是这条触发了推翻**
- 若 SIMPLE 官方勾选 SONIC 集成 TODO → 大幅提升 SIMPLE 权重
- 若 NVIDIA 官宣 Arena 原生 SONIC 集成 → G1 全部评测收敛到 Arena
- 若团队人力无法同时维护双栈 → 只留 Arena

## Caveats

- **SIMPLE 的实际可运行性未经验证（中）**。未找到公开的"装不上/跑不起来"报告——但这不等于没有坑；其重依赖栈属公认易碎的组合
- **「500 Hz MuJoCo」在 SIMPLE 文档中未逐字确认（中）**：项目页只说 MuJoCo 做"高频机器人控制"、Isaac 渲染 `--render-hz=50`
- **Isaac Lab-Arena 是否已端到端集成 SONIC：没有找到直接证据**
- **WOLF-VLA 的评测套件是否已开源：未核实**
- **中国二线厂商的开源评测工作：没有找到证据**
- arXiv 编号 2606.xxxxx / 2607.xxxxx 对应 2026 年 6–7 月，均为极新预印本，部分结论来自论文自述，未经第三方复现
