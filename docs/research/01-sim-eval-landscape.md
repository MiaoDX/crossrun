# 机器人仿真评测（Robot Sim Eval）开源生态调研报告

> 调研时间：2026 年 7 月 · 这是 crossrun 设计的起点文档
> ⚠️ 部分判断已被后续证据修正，见 [README](README.md)

## TL;DR

- **当前机器人 sim eval 已分化为四条主线**：①学术界经典 manipulation benchmark（LIBERO、ManiSkill、RoboCasa 等）作为"通用货币"；②各公司自研的一体化"数据+仿真+评测"平台（AgiBot Genie Sim、NVIDIA Isaac Lab-Arena、InternRobotics、RoboTwin）；③Real2Sim 路线（SimplerEnv、3DGS 重建式 RoboGSim/DISCOVERSE）与分布式真机评测（RoboArena、AutoEval）；④world model 作为 evaluator 的新兴路线（1X World Model、GE-Sim、GigaWorld）。
- **最值得关注的 15 个 repo**：huggingface/lerobot、Physical-Intelligence/openpi、NVIDIA isaac-sim/IsaacLab-Arena、NVIDIA/Isaac-GR00T、haosulab/ManiSkill(3)、AgibotTech/genie_sim、RoboTwin-Platform/RoboTwin、RoboDojo-Benchmark/RoboDojo、simpler-env/SimplerEnv、InternRobotics(InternUtopia/InternManip/InternNav)、Genesis-Embodied-AI/Genesis、google-deepmind/mujoco(+Playground)、TATP-233/DISCOVERSE、ARISE-Initiative/robosuite(+RoboCasa)、carlosferrazza/humanoid-bench。
- **核心结论**：没有任何单一 benchmark 能作为策略优劣的最终裁判——仿真评测方差大、易被"记忆过拟合"。LIBERO-PRO 论文（arXiv:2510.03827）明确指出：标准 LIBERO 评测下超过 90% 准确率的模型，在其泛化设定下性能崩溃至 0.0%（OpenVLA 在 LIBERO-Goal 位置扰动下 0.96→0.00，π0 0.94→0.00）。

## Key Findings

1. **一体化平台成为公司标配**：AgiBot Genie Sim 3.0（CES 2026 发布）、NVIDIA Isaac Lab-Arena、上海 AI Lab InternRobotics 三大体系都把"环境生成 → 数据采集 → 标准化评测"打通，并强调 LLM/VLM 驱动的场景与评测配置自动生成。
2. **VLA 策略评测接口正在标准化**：openpi 的 policy server（端口 8000、WebSocket）、InternManip 的 client-server、RoboDojo/XPolicyLab 的统一接口、LeRobot 的 `lerobot-eval` CLI，都采用"模型集成一次、benchmark 集成一次"的解耦范式。
3. **Real2Sim 评测可信度已有量化支撑**：SimplerEnv 用 MMRV（Mean Maximum Rank Violation）和 Pearson 相关系数衡量 sim 排序与真机的一致性；Genie Sim 3.0 技术报告（arXiv:2601.02078）用 π0.5 基线、16 个模型配置得到 sim-real 相关性 R²=0.924、斜率≈1.045——但这些结论多来自平台方自证，需谨慎。
4. **World model 作为 evaluator 从概念走向 benchmark**：1X 提出把 world model 当 evaluator；AgiBot GE-Sim/EWMBench；GigaWorld-1（arXiv:2607.02642）系统性研究"什么样的 world model 是可靠评测器"，分析了 7 个视频 world model、4 种动作表示、超过 32.4 万次模拟策略 rollout。
5. **可复现性问题被正式提出**：LIBERO-PRO、LIBERO-Plus 揭示 VLA 在标准协议下的高分主要来自对动作序列/布局的记忆；随机种子导致的成功率波动可超过 30%。

## A. 各公司/机构自研的 sim eval 平台与 benchmark

### 中国公司/机构

**AgiBot 智元 — Genie Sim / Genie Sim Benchmark**（https://github.com/AgibotTech/genie_sim ，约 1.1k stars，MPL 2.0）：基于 Isaac Sim，核心是 3DGS 场景重建转 USD + LLM 驱动的场景/任务/评测配置自动生成。Genie Sim 3.0 于 CES 2026 发布，覆盖 200+ 任务、100,000+ 评测场景，开放 10,000+ 小时合成数据集，支持 GO-2、Pi 系列、GR00T 系列等主流模型。Benchmark 分五大能力套件：指令遵循、空间理解、操作技能、干扰适应、训练-部署泛化。

**AgiBot — GE-Sim / GE-Sim-V2 / Genie-Envisioner**：Genie Envisioner 是"世界基础模型"平台；GE-Sim 是其中的 world simulator，GE-Sim-V2 基于 Cosmos2。附带 **EWMBench**（Embodied World Model Benchmark），评测视频式 world model 的场景、运动、语义质量。

**北京人形机器人创新中心 X-Humanoid**：联合北京大学推出 **RoboMIND**（5 类场景、96 类物体、6 类操作、479 项任务；数据来源含 31,005 条 Franka、9,686 条"天工"人形、8,030 条 AgileX 双臂、6,911 条 UR-5e 轨迹）；同时开源运动控制框架 Tien Kung-Lab、铰接资产 ArtVIP、训练工具链。通用具身智能平台"慧思开物"实现"一脑多机、一脑多能"。

**上海 AI Lab / InternRobotics**：InternUtopia（原 GRUtopia，约 1.2k stars，MIT）是基于 Isaac Sim 的城市级具身仿真平台。InternManip（MIT）是"训练+评测"一体化操作套件，统一数据格式/模型接口/评测协议，采用 client-server 部署。InternNav 基于 PyTorch + Habitat + Isaac Sim，支持 VLN-CE、点/图/轨迹目标导航。

**Galbot 银河通用**：以 SDK/本体为主。其仿真评测代表作是 **GraspVLA**（Galbot + 北大 EPIC 实验室，CoRL 2025），全部在仿真中用 SynGrasp-1B（十亿帧合成抓取数据 + 域随机化）训练。

**Galaxea 星海图**：开源 GalaxeaVLA（G0/G0.5 双系统 VLA）与 Galaxea Open-World Dataset（500+ 小时真机移动操作，LeRobot v2.1 格式）。评测上依赖 LIBERO + SimplerEnv，无独立自研仿真器。

**ByteDance Seed — GR-3**（arXiv:2507.15493）：GR-3 VLA 的评测**以真机为主**（ByteMini 双臂移动机器人、99 物体、861 次实验），并非以仿真 benchmark 为主的工作。

**Lightwheel 光轮智能**：与 NVIDIA 共同开发 Isaac Lab-Arena，开源 **LW-BenchHub**（268 任务 = 130 Lightwheel-LIBERO-Tasks + 138 Lightwheel-RoboCasa-Tasks）。其基准研究显示在 8× RTX 6000D 上用 GR00T N1.5 跑 1,024/2,048/4,096 并行环境评测，相较传统 Robosuite/MuJoCo 栈可提速达约 13.5×。

### 海外公司

**NVIDIA — Isaac Lab-Arena / Isaac Lab / Isaac-GR00T**：Isaac Lab-Arena（与 Lightwheel 共研，pre-alpha 开源）是可组合、GPU 加速的策略评测框架，与 HF LeRobot Environment Hub 集成。据 NVIDIA 官方技术博客，在 8×RTX 6000D 上用 Isaac GR00T N1.5、每任务 4096 个环境变体跑 10 个 RoboCasa 任务，并行评测仅需 0.76 小时，比顺序评测（34.9 小时）快 40 倍。Isaac-GR00T 到 N1.7 支持 UNITREE_G1，含 PolicyServer 推理接口。

**Google DeepMind — MuJoCo / MJX / MuJoCo Playground**：MuJoCo（Apache 2.0）是通用物理引擎；MuJoCo Playground 基于 MJX，支持四足、人形、灵巧手、机械臂，强调从 state 和 pixel 输入的零样本 sim-to-real。

**Physical Intelligence — openpi**（约 11.8k stars）：开源 π0、π0-FAST、π0.5，含 LIBERO 和 ALOHA sim 评测。评测采用 policy server（`serve_policy.py`，端口 8000）+ Dockerized LIBERO 工作流。配置中含 roboarena_config，与 RoboArena 打通。

**Hugging Face — LeRobot**（约 25.2k stars，v0.6.0）：把第三方仿真器（LIBERO、Meta-World、RoboTwin 2.0、RoboCasa365 等）包装成 Gymnasium 接口，用统一 `lerobot-eval` CLI 评测。v0.6.0 新增 6 个仿真 benchmark、world model 策略、reward models API，并推出 LIBERO-plus（约 10,000 个扰动变体）。

**Meta — Habitat / PARTNR**：照片级室内导航与 rearrangement 平台，被 InternNav 等作为导航评测后端。

**1X — World Model 作为 evaluator**：1X World Model Challenge（Compression / Sampling / Evaluation 三阶段），把"给定 N 个策略、在 world model 内评测并给出排序"作为终极目标。1X 官方博客强调 world model 解决真机评测不可复现的痛点——同样的模型权重可能因环境背景或光照的细微变化，在数天内出现性能急剧退化。

## B. 学术界经典与新兴 manipulation benchmark

- **LIBERO**：VLA 评测的"通用货币"，四套件 LIBERO-Spatial/Object/Goal/Long，各 10 任务、500 演示。
- **LIBERO-PRO / LIBERO-Plus**：鲁棒性扩展。LIBERO-PRO（arXiv:2510.03827）沿物体/初始状态/指令/环境四维扰动，OpenVLA 在 LIBERO-Goal 位置扰动下由 0.96→0.00，π0 由 0.94→0.00，π0.5 由 0.97→0.38（唯一非零）。LIBERO-Plus 用 L1-L5 难度分层、约 56K 鲁棒性场景评测十个 VLA。
- **ManiSkill / ManiSkill3**（约 3.1k stars，RSS 2025）：基于 SAPIEN 的 GPU 并行仿真与 benchmark，复杂 ReplicaCAD 场景可达 2000+ FPS。SimplerEnv 的 GPU 版即整合到 ManiSkill3。
- **robosuite / RoboCasa / RoboCasa365**：robosuite（MuJoCo）是模块化操作仿真框架；RoboCasa 是大规模家庭厨房 benchmark（120 场景、25 原子任务 + 75 LLM 生成复合任务、100k+ 轨迹）。RoboCasa365 扩展到 365 厨房任务、2,500 程序生成厨房。
- **Genesis**（约 30k stars，Apache 2.0）：统一多物理求解器（刚体/MPM/SPH/FEM/PBD/Stable Fluid）+ 光追渲染。2026 年由 Genesis AI 官方支持，更名 Genesis World 1.0。
- **其它经典**：Meta-World、RLBench、CALVIN、VIMA-Bench、ARNOLD、The Colosseum、VLABench、GenManip、EmbodiedBench、MimicGen/DexMimicGen、GenSim/RoboGen、BEHAVIOR-1K/OmniGibson。

## C. Real2Sim / Sim2Real 评测

- **SimplerEnv / SIMPLER**（约 950 stars，CoRL 2024）：real-to-sim 评估 VLA。两种设置：Visual Matching 与 Variant Aggregation。支持 Google Robot 与 WidowX+Bridge。用 MMRV 与 Pearson 相关系数量化 sim 与真机排序一致性。
- **RoboArena**（arXiv:2506.18123，CoRL 2025）：分布式**真机**评测，众包 + 双盲成对比较，通过 7 所大学对 7 个通才策略的去中心化评测、共 612 次成对比较，声称比集中式评测更准确地排序通用策略。
- **AutoEval**（Berkeley）：真机策略的自主评测（自动重置场景/运行/判定）。
- **3DGS/NeRF 重建式仿真评测**：RoboGSim（Megvii，Real2Sim2Real）、DISCOVERSE（约 476 stars，IROS 2025，清华 AIR 等，3DGS + MuJoCo，650 FPS RGB-D）、SplatSim、real2sim-eval。

## D. 人形 / 全身控制 / 导航方向

- **HumanoidBench**（RSS 2024）：MuJoCo，Unitree H1 + Shadow Hands，101 DoF，27-31 任务，支持 MJX 域随机化与多模态。
- **LocoMuJoCo**：模仿学习为主的运动 benchmark，12 人形 + 4 四足、4 生物力学人体模型，含 22,000+ 动作捕捉数据集。
- **SIMPLE**：耦合 MuJoCo 接触动力学 + Isaac Sim 照片级渲染，60 全身任务、50 室内场景、1000+ 资产。

## E. 底层仿真引擎与渲染栈对比

| 引擎 | 物理 | 并行/吞吐 | 渲染 | 资产生态 | 许可 |
|---|---|---|---|---|---|
| **MuJoCo / MJX / MJWarp** | 接触精度高 | MJX GPU 并行 | 中（需外接渲染） | MJCF、Menagerie | Apache 2.0 |
| **Isaac Sim + PhysX / Isaac Lab** | PhysX，高保真 | GPU 大规模并行 | RTX 光追，最强 | USD 生态最全 | 混合 |
| **ManiSkill3 / SAPIEN** | GPU 并行 | 2000+ FPS（含渲染） | 光栅+光追 | URDF/自有 | MIT/Apache |
| **Genesis / Genesis World** | 多求解器统一 | 宣称 43M FPS（Franka） | Nyx 光追 | URDF/MJCF/USD/GLB | Apache 2.0 |
| **PyBullet / Gazebo / Drake** | CPU 为主，成熟 | 弱并行 | 一般 | URDF/SDF | 开源 |
| **Newton（NVIDIA/DeepMind/Disney）** | 新一代可微 GPU 物理 | GPU | — | — | 开源 |

## F. 评测方法学与工程基础设施

- **评测协议设计**：固定 seed 与初始状态、成功判定、任务分层、episode 数量（LIBERO 常用每任务 500，全套 2,000；RoboTwin 每任务 100 rollouts）、多 seed 多次运行取均值/置信区间。
- **已知问题**：随机种子导致成功率波动可超 30%；仿真评测慢；记忆过拟合；未记录协议（seed、归一化统计、physics-settling 步数常被论文省略）；sim2real gap；平台方自证的相关性结论。
- **统一评测 harness**：vla-eval（arXiv:2603.13966）用 WebSocket+msgpack + Docker 隔离，支持 14 个仿真 benchmark、6 个 model server，episode 分片并行提速达 47×，并把 Docker 镜像 tag/seed/episode 数与结果一起归档以保证可复现。
- **VLA 策略接口**：openpi policy server、InternManip/RoboDojo 的 client-server、LeRobot 的统一 policy 抽象、GR00T PolicyServer；普遍采用 gRPC/WebSocket 远程推理。

## 锚点 repo 深入分析

1. **AgibotTech/GE-Sim-V2**：Genie Envisioner World Simulator 2.0，基于 Cosmos2 的 world-model 仿真器，与 EWMBench 配套。
2. **AgibotTech/genie_sim**：智元一体化"数据+仿真+评测"平台，Isaac Sim + 3DGS + LLM 场景生成，200+ 任务、五能力套件。
3. **Physical-Intelligence/openpi**：π0/π0.5 开源 + LIBERO/ALOHA sim 评测 + policy server 接口。
4. **RoboTwin-Platform/RoboTwin**：双臂操作 benchmark。RoboTwin 2.0（50+ 任务、731 物体/147 类、100k 演示、强域随机化），CVPR 2025 Highlight，有 leaderboard、IsaacLab-Arena 分支。约 2.4k stars，MIT。
5. **RoboDojo-Benchmark/RoboDojo**：统一 sim-and-real benchmark（42 仿真任务 + 18 真机任务、3 本体），仿真跑在 Isaac Sim，五能力维度，配 RoboDojo-RealEval 远程真机 + XPolicyLab。

## 趋势与 gap

1. **统一评测标准仍缺失**，各家都想成为事实标准但都没赢。
2. **real2sim 评测的可信度**多来自平台方自证，缺乏独立第三方大规模验证。
3. **world model 评测兴起**，相关性数字（RoboWorld r=0.989、PolaRiS r=0.98）很高但成熟度存疑。
4. **benchmark 饱和与泛化评测**：标准协议下的高分越来越不可信。

## Caveats

- star 数与更新时间为近似值，数据截至 2026 年 7 月。
- **平台方自证偏差**：Genie Sim "sim-real 差异 <10%、R²=0.924"、RoboGSim/DISCOVERSE 的相关性结论多来自平台方，缺乏独立复现。
- **GR-3 归类**：ByteDance GR-3 的评测以真机为主，并非仿真 benchmark 工作。
- **快速演进领域**：world model evaluator、real-to-sim 翻译、GPU 并行评测均处于高速迭代期，结论可能很快过时。
