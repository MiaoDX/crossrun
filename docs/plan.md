# crossrun 设计文档 v2.1

> **定位**：开源机器人 policy sim/real 运行与评测的参考设计。
> **状态**：设计阶段。架构已定，代码未起。
> **配套调研**：见 [`research/`](research/)。

**本文档的演进历史**：v1.x 曾以「内部评测体系」定位，v2.0 重构为「开源参考设计」，v2.1 补回执行层（Runner）。演进过程中被推翻的判断，均保留在 §12「显式否决」中，避免重复讨论。

| | v1.x：内部体系 | **v2.x：开源参考设计** |
|---|---|---|
| "被认可"指什么 | 我们的数字可信 | **这套架构被认为是对的做法** |
| 依赖数量 | 工程便利问题 | **依赖最小化本身是设计主张** |
| 能力覆盖 | 越多越好 | **越少越清晰，opinionated 是优点** |
| 许可证 | 商业化边界要算 | **基本不用管**（不做商业化） |
| 交付物 | 能跑的评测 | **可读的设计 + 有说服力的 demo**
---

## 1. 项目定位

**一句话**：在 2026 年这个时间点，重新设计一套机器人 sim eval 的参考架构，并用可复现的 demo 证明它是对的。

**为什么现在做有意义**：很多公司要么还没投入，要么已投入很久、背着历史包袱。从当前时间点重做，可以有更干净的架构。

**交付物优先级**：

1. **可读的架构设计**（别人看懂并借鉴）
2. **有说服力的 demo**（架构主张需要可见证据）
3. 能跑的评测（是前两者的载体，不是目的本身）

**成功标准**：别人 clone 下来能跑通，看完之后觉得"sim eval 就该这么做"。

---

## 2. 设计主张（本项目要论证的观点）

这些是 repo 要传达的核心观点，每一条都应有对应的代码体现和 demo 支撑：

1. **协议优先于实现** —— 定义 `obs → action` 契约，模型和仿真器都是可替换的实现
2. **模型不进 repo** —— 每个模型一个容器；repo 只持有协议、模板、registry
3. **后端可插拔且无需互相兼容** —— 后端只需各自兼容协议
4. **默认路径必须零摩擦** —— 笔记本/单卡可跑；重后端一律 opt-in
5. **数字必须带不确定性** —— 单点成功率是有害的，置信区间是底线
6. **标准协议下的高分不可信** —— 扰动分层是必需品不是加分项
7. **跨后端只比排序，不比绝对值**
8. **模型与环境之间必须有执行层** —— 模型只管推理，环境只管物理，episode 怎么跑由中间层负责
9. **仿真与真机的差异要显式命名，不要抽象掉** —— 差异只有三样：reset 从哪来、success 从哪来、时间怎么走

---

## 3. 已知可跑的基线矩阵

参考设计必须先说清"什么是确定能跑的"。以下为调研确认的现状：

| Benchmark | 后端 | 本体 | 已验证模型 | checkpoint |
|---|---|---|---|---|
| **LIBERO** | robosuite / **MuJoCo** | Franka | π0、π0.5、OpenVLA、SmolVLA、Diffusion Policy | ✅ 公开 |
| RoboDojo | Isaac Sim | ARX X5 / Piper / Piper X | **30 个策略**（经 XPolicyLab，40+ 已集成） | 公开榜 |
| RoboLab | Isaac | — | π0.5、π0-FAST、π0、PaliGemma | — |
| RoboTwin | SAPIEN | Aloha-AgileX 等 | 多个 | ✅ |
| SimplerEnv | SAPIEN | Google Robot / WidowX | RT-1、RT-1-X、Octo | ✅ |
| **Genie Sim** | Isaac | **G2** | GO-2、Pi 系列、GR00T | ✅ **我方已本地验证** |
| Arena | Isaac | 多本体 | GR00T N、π0/π0.5、SmolVLA | ✅ |
| SIMPLE | MuJoCo + Isaac | G1、aloha、franka | 约 10 个（含 WAM） | — |
| **Genesis** | 自有 | — | **未找到 VLA 生态** | ❌ |

**正面证据（生态是健康的）**：RoboLab 将各策略的 RoboLab-120 成功率与 RoboArena 的 Elo 对比，**策略排序被完好保持（Spearman ρ=1.00）**，分数正相关（Pearson r=0.68）。且数字有清晰区分度 —— 颜色扰动下 π0.5 96.7%、π0-FAST 93.3%、π0 仅 6.7%；阴影扰动下分别为 100% / 90% / 0%。

> **结论：模型跑得动，榜出得来，可比性没问题。** 脆弱的只是"精确复现某个 headline 数字"，那是协议问题不是能力问题。

---

## 4. 后端策略：默认轻，重后端 opt-in

参考 repo 的第一体验是「clone → 装 → 跑出一个数字」。按这个标准决定默认值：

```
默认路径（零摩擦，有公开 checkpoint）
└── MuJoCo / LIBERO ────────── pip 可装；π0.5 / OpenVLA / SmolVLA 都在这

opt-in 重后端（各自有不可替代的理由）
├── Isaac Lab-Arena ────────── 仅为 G1 + SONIC 存在
└── Genie Sim ──────────────── 仅为 G2 存在（已验证可跑）

验证性后端
├── MuJoCo Sim2Sim ─────────── 检测跨后端排序稳定性
└── Genesis ────────────────── 评估门（§8），非默认路径
```

**为什么默认不是 Genesis**：它安装轻、厂商中立（CUDA/Metal/Vulkan）、自带路径追踪渲染，长期看很有潜力 —— 但**目前没有任何已验证的 VLA checkpoint 生态**。默认路径上必须有东西能立刻跑起来。

**为什么 mjlab 出局**：其论文明确设计取向是不为视觉策略提供 RGB 渲染，推崇"视觉策略从全状态特权策略蒸馏 + 外部渲染"，且以非跨仿真器为非目标。适合 locomotion RL，不适合 VLA 评测。

**Isaac 的重量只在必须的那一格付出。** 这既是工程判断，也是设计主张 —— 参考设计的价值取决于有多少人能真的跑起来。

---

## 5. 架构

```
┌──────────────────────────────────────────────────────────┐
│  策略提供方层（不进 repo · 统一说 4 操作协议）                │
│                                                          │
│   单个模型容器                    模型库桥接                │
│  ┌────────┐ ┌────────┐ ┌───────┐ ╔══════════════════════╗│
│  │ π0.5   │ │OpenVLA │ │自研/WAM│ ║ XPolicyLab           ║│
│  │        │ │        │ │        │ ║ 策略服务框架，内含 40+ ║│
│  └───┬────┘ └───┬────┘ └───┬────┘ ╚═══════════╤══════════╝│
│      │          │          │                  │           │
│      │  「一个模型」         │      「一批模型的来源」        │
└──────┼──────────┼──────────┼──────────────────┼───────────┘
       └──────────┴────┬─────┴──────────────────┘
                             │ 策略服务协议（ws / msgpack）
                             │ 强 schema + obs key 映射
                             │ + 动作空间标注（joint / EEF / SONIC token）
┌────────────────────────────┴─────────────────────────────┐
│  ⭐ 执行层 Runner（本项目核心 IP · 零仿真器依赖）            │
│  ┌─────────────────────────┬────────────────────────────┐│
│  │ SimRunner               │ RealRunner                 ││
│  │  reset  : env.reset()   │  reset  : AutoEval 重置策略 ││
│  │  success: 状态谓词       │  success: VLM 分类器        ││
│  │  safe   : 恒真          │  safe   : 工作区边界+电机    ││
│  │  time   : 步进/可批并行  │  time   : 墙钟定频·单流      ││
│  └─────────────────────────┴────────────────────────────┘│
│  向上：对策略完全对称  ·  向下：显式吸收 sim/real 差异       │
└────────────────────────────┬─────────────────────────────┘
                             │ 环境适配
   ┌──────────┬──────────┬───┴──────┬──────────┬──────────┐
┌──▼───┐ ┌────▼────┐ ┌───▼────┐ ┌───▼────┐ ┌───▼─────────┐
│MuJoCo│ │ Arena   │ │GenieSim│ │Genesis │ │ 真机硬件      │
│LIBERO│ │ G1+SONIC│ │  G2    │ │(评估门)│ │(LeRobot Robot│
│(默认)│ │ (opt-in)│ │(opt-in)│ │        │ │ + SONIC C++) │
└──────┘ └─────────┘ └────────┘ └────────┘ └─────────────┘
```

**四条分层原则**：

> 1. **模型实现层 ≠ 策略服务层 ≠ 执行层 ≠ 环境层**
> 2. **同一个 policy server 进程，sim client 与 real client 都能连，不改任何东西** —— 这是 sim/real 对称性的全部来源
> 3. **后端之间不需要互相兼容，只需各自兼容协议**
> 4. **执行层零仿真器依赖** —— 这是它能同时装在评测机和真机上的唯一原因

### 5.1 执行层：为什么必须存在

模型容器只管推理（`obs → action`），环境只管物理。但"一个 episode 怎么跑"两边都不管 —— 这正是执行层的职责，也是 sim 与 real 的全部差异所在。

**仿真白给三样东西，真机没有**：

| 能力 | SimRunner | RealRunner |
|---|---|---|
| `reset()` | `env.reset()` | AutoEval 重置策略（或人工 + 透明叠加对位） |
| `is_success()` | 环境状态谓词 | 微调 VLM 成功分类器 |
| `is_safe()` | 恒真 | 工作区边界 + 电机状态检查 + 自动重启 |
| 时间语义 | 步进，可批量并行 | 墙钟定频，单流 |

> **AutoEval 的三模块不是独立工作，它就是 RealRunner 的实现。** 参考数据：节省 >99% 人力，24 小时仅需 3 次干预，新任务 cell 搭建 1–3 小时。

**XPolicyLab 的归位（易误读，特此明确）**：

它扮演两个不同的角色，必须分开看：

| 角色 | 含义 | 在架构中的位置 |
|---|---|---|
| **接口设计的来源** | 其四操作（观测更新 / 动作预测 / policy reset / 批量查询）被采纳为**我们协议的契约形态** | 影响协议层的设计 |
| **一批模型的提供方** | 它内含 40+ 已集成策略，可整体接入 | 协议之下，与单个模型容器**并列但不同类** |

**它不是一个模型。** 图中特意用双线框与单个模型容器区分——单个容器提供一个策略，XPolicyLab 提供一批。二者都在协议之下、都说同一套协议，但结构上不是一类东西。

> 为什么不直接采用 XPolicyLab 作为我们的协议层？因为它的标准化字典规定了 pose 格式，能否承载 SONIC 的 78 维 latent token 未经验证（见 §13）；且 Arena 有自己一套 `isaaclab_arena_<policy>` 约定，我们需要同时接两条。自研薄契约在上，两者作为适配在下。

**SONIC 不破坏架构**：动作空间为 78 维（64 维 latent motion token + 7 维左手 + 7 维右手），SONIC 控制器以 50 Hz 解码为全身关节指令。**解码器在环境侧**，故策略接口仍是 `obs → action_vector`，协议层只需标注动作空间类型。

---

## 6. 模型接入：容器优先

**三种架构的扩展上限**：

| 架构 | 代表 | M 上限 | 瓶颈 |
|---|---|---|---|
| 共享 `src/` + 依赖分组 | Psi0 | ~10 | 依赖冲突（其安装命令用 `--index-strategy unsafe-best-match`、`--no-build-isolation`，是求解吃紧的信号） |
| 每模型独立环境 | XPolicyLab | 40+（已验证） | repo 膨胀；**维护者成为守门人** |
| **每模型一个容器** | vla-eval | 基本无上限 | 需 registry（集群通常已有） |

**本 repo 只持有三样东西**：

```
├── protocol/          # obs → action 协议规格 + 强 schema（4 操作契约）
├── runner/            # ⭐ 执行层：SimRunner / RealRunner
│   ├── sim/           #    各后端适配
│   └── real/          #    AutoEval 三模块（reset / success / safe）
├── eval-protocol/     # seed / episode / 统计 / CI（仿真器与 runner 均无关）
├── template/          # 参考容器模板，照做即可接入
└── registry.yaml      # 模型名 → 镜像 digest + 动作空间 + obs 需求
```

> `runner/real/` 与 `runner/sim/` 实现同一组接口。**这是本 repo 最值得被抄的部分** —— 它让"同一模型、同一协议，仿真与真机一致运行"从口号变成代码。

**收益**：M 无上限；贡献者发一个镜像 + 一个 manifest PR 即可接入，不必动本 repo；**镜像 digest 入结果 = 可复现性免费**。

XPolicyLab 作为其中一个容器接入，白拿其 40+ 策略。

---

## 7. 评测协议

**能力维度**：泛化 / 记忆 / 精度 / 长程 / 开放指令 + 干扰鲁棒性。（Genie Sim 与 RoboDojo 各自独立收敛到相似的五维划分，说明已有共识。）

### 7.1 跨后端可比性规则（硬约束）

不同物理引擎下同一任务难度不同 —— 接触模型、摩擦、求解器容差都会改变成功率。

> **只比同后端内的排序，不比跨后端的绝对数字。**

- 同后端内：策略排序有效，绝对成功率有效
- 跨后端：绝对数字**无效**，不得放进同一张排名表
- 跨后端唯一有效比较：同一组策略的**排序是否一致**。不一致本身是重要信号，说明至少一个后端有仿真伪影
- 后端与版本是报告模板的一等字段，不是脚注

这条同时给 MuJoCo Sim2Sim 一个明确用途：**不是为了得到更准的数字，而是检测排序稳定性。**

### 7.2 统计纪律

- 固定并记录 seed、episode 数、**容器镜像 digest**、归一化统计、physics-settling 步数
- **报告置信区间（Clopper-Pearson），不报单点成功率**
- **不得用"CI 重叠"判断"无显著差异"** —— 使用校正的成对检验
- 参考文献的 trial 数往往很小（SONIC 论文每任务 10–20 trial，SIMPLE 每级 10 trial），**不可与大样本结果直接比较**

**功效参考**：90% 成功率、70 次 rollout 时 95% CI 宽 15.4 个百分点；收窄到 ±2 点需约 1030 次。
**协议差距实例**：OpenVLA 官方协议为 3 seed × 每 seed 500 rollout（每任务 50 次）；社区复现常用 `n_episodes=10`，**相差五倍**。

### 7.3 防记忆过拟合

LIBERO-PRO 证明标准协议下 >90% 成功率的模型，在扰动设定下可崩至 0.0%（OpenVLA、π0 在 LIBERO-Goal 位置扰动下 0.96/0.94 → 0.00；π0.5 → 0.38 为唯一非零）。

→ **扰动分层是必需品。标准分与扰动分必须同时报告。** SIMPLE 的三级 OOD 分级（视觉+干扰物 / 光照 / 目标位姿）是可直接借鉴的设计（MIT 协议）。

### 7.4 真机对称

**实现位于执行层（§5.1），不是独立模块。** RealRunner 用 AutoEval 三模块补齐仿真白给的能力；评测协议层对两种 Runner 一视同仁，只消费 `(success, trajectory)`。

低成本借鉴：RoboDojo-RealEval 用目标布局图与实时观测流的**透明叠加**恢复一致初始条件 —— 比学习式 reset 成本更低，可作为 RealRunner 的第一版 reset 实现。

---

## 8. Demo：一等交付物

**架构主张需要可见的证据。** demo 不是附属品，是这个项目最重要的产出。三个候选，全部只需 LIBERO + MuJoCo + 公开 checkpoint：

### Demo 1：协议对数字的影响有多大（首要）

取同一个官方 checkpoint，系统性变化 episode 数 / seed / 扰动等级，画出成功率的分布。

**为什么有说服力**：官方协议与社区复现相差五倍 episode 数；70 次 rollout 时 CI 宽 15.4 个百分点。这直接演示"没有固定 seed + 足够 episode + CI + 镜像 digest，你连'这是噪声还是配置 bug'都答不了"。

> ⚠️ **框定要求**：这是建设性的方法学演示，**不是"复现危机"唱衰**。生态是健康的，模型跑得动；我们展示的是协议敏感性。

### Demo 2：跨后端排序不一致

同一组策略在 MuJoCo 与 Isaac 上排名不同。直接证明 §7.1 规则的必要性。**未见有人公开做过，原创性最高。**

### Demo 3：90% → 0%

现场演示模型在标准协议下 >90%、加位置扰动后崩到接近 0。LIBERO-PRO 已证实此现象，但**做成人人可复现的 demo，冲击力完全不同**。

---

## 9. 范围与阶段

> **参考设计不追求覆盖度。** 用最小的例子把模式讲清楚，再证明它能扩展。

**最小可演示集**：LIBERO / ALOHA（MuJoCo）× 两种范式（VLA + WAM）。足以演示全部七条设计主张。
**扩展性证明**：G1 + SONIC（Arena）、G2（Genie Sim）后置。

### Phase 0 — 地基

- [ ] **协议 4 操作契约 + 执行层 Runner 接口定稿**（两者一起定，不可只定协议）
- [ ] 容器模板 + registry 设计定稿
- [ ] SimRunner + LIBERO + π0.5 官方 checkpoint 端到端跑通（默认路径必须最先通）
- [ ] `eval-protocol` 落地：seed 管理、镜像 digest 归档、Clopper-Pearson CI
- [ ] **确认 Arena 跟随 Isaac Lab 的 main 还是 develop 分支**（main 已冻结，Newton 重构在 develop）

### Phase 1 — 三个 Demo + 架构验证

- [ ] Demo 1（协议敏感性）、Demo 3（90%→0%）
- [ ] 接入第二、三个模型（OpenVLA、SmolVLA）验证容器模式
- [ ] 接入一个 WAM 验证范式无关性
- [ ] **RealRunner 骨架 + 最小真机链路**（先用透明叠加做人工 reset，AutoEval 三模块后补）
      → 目的：**尽早证明同一 policy server 能同时服务 sim 与 real**，这是核心主张之一，不能拖到最后验证
- [ ] **Genesis 评估门**（见下）
- [ ] Demo 2（跨后端排序）—— 依赖第二个后端就位

**Genesis 评估门（约 1 周，带可测标准）**

| 判据 | 通过标准 | 不通过意味着 |
|---|---|---|
| ① 移植成本 | 一个任务 + 本体移植 ≤5 天 | >5 天说明抽象层已泄漏，这本身是高价值发现 |
| ② 渲染吞吐 | 4090 上实测；验证 Nyx 宣称的 1080p ≤4ms | 远低于宣称值则其核心优势不成立 |
| ③ 抽象层密封性 | `protocol` 与 `eval-protocol` **零修改**复用 | 需要改 = 抽象层设计有问题，必须先修 |

三条全过 → Genesis 可作为轻量后端候选并承担 Demo 2 的第二后端角色。任一条不过 → 归档为观察项。

### Phase 2 — 扩展性证明：G1 + SONIC

- [ ] `sonic-adapter`：78 维动作空间接入，解码器置于环境侧
- [ ] Arena + GR00T N1.7 + `UNITREE_G1_SONIC`
- [ ] 低层控制器可替换，支持失败归因（隔离 VLA 推理能力 vs WBC 稳定性）
- [ ] MuJoCo Sim2Sim 排序校验

### Phase 3 — 扩展性证明：G2

- [ ] **Genie Sim 作为 pinned submodule**，仅用于抽取 G2 资产
- [ ] 抽取脚本产出 → `assets/`（走 HF Hub），**运行时不依赖 Genie Sim**
- [ ] 判据：**别人不拉 submodule 也能跑**
- [ ] Genie Sim 自带 checkpoint 容器化，作为 pipeline 自检的 known-good 基线

### Phase 4 — 对外

- [ ] RealRunner 补齐 AutoEval 三模块（VLM 成功分类器 / reset 策略 / 安全检测）
- [ ] 文档与设计说明（这是主交付物之一）
- [ ] 注册到 LeRobot EnvHub
- [ ] 考虑向 SIMPLE 上游贡献 SONIC 集成（其自列 TODO）

---

## 10. Genie Sim 的三重拆解

已本地验证可运行，G2 的资产、环境、checkpoint 齐备。但作为**开源参考设计**，不能让它成为运行时必要项 —— 否则就在演示错误示范。拆成三个互不相关的东西：

| 用途 | 形式 | 是否运行时依赖 |
|---|---|---|
| G2 资产来源 | pinned submodule + 抽取脚本 → `assets/` | ❌ |
| 场景设计参考 | 读代码 | ❌ |
| 一个 baseline 模型 | 容器化 | ❌（模型层） |

**判据：别人 clone、不拉 submodule，应该照样能跑默认路径。** submodule 只在重新生成资产时需要。

---

## 11. 风险

| 风险 | 等级 | 应对 |
|---|---|---|
| **Newton 迁移**：Isaac Lab 已冻结 main，物理层从 PhysX 转向 MuJoCo-Warp，Arena 将继承此 churn | 高 | Phase 0 确认分支；严格 pin；协议层隔离 |
| **G1 依赖链最长**（Arena + GR00T + WBC + SONIC C++/ZMQ） | 中高 | 后置到 Phase 2；先在 MuJoCo 跑通 SONIC 栈 |
| **Arena pre-alpha，接口无弃用警告即变更** | 中高 | 严格 pin；协议层隔离；升级独立 PR + 回归 |
| **多后端口径漂移** | 中 | §7.1 规则；`eval-protocol` 统一收口 |
| **demo 沦为玩具** | **中高** | 见下 |

> **"没有历史包袱所以更干净"的已知失败模式**：很多参考架构之所以干净，是因为它还没撞过现实；带包袱的项目，包袱往往是撞出来的。
> **唯一防御：demo 必须是真的。** 真模型、真统计、真扰动。展示"玩具任务上跑通了漂亮架构"会和其他几十个废弃框架一样；展示"我们测出了 90%→0% 且你能复现"才立得住。

**安全**：策略服务器**禁止绑定公网**。参考教训：LeRobot async inference 的 gRPC PolicyServer 曾存在未认证 pickle 反序列化 RCE（CVE-2026-25874，CVSS 9.8）。同类风险适用于所有策略服务器与容器端口。

**许可证**（不做商业化，故大幅简化）：Isaac Lab / Arena / Genesis / mjlab 为 Apache 2.0；Genie Sim 代码 MPL 2.0；SIMPLE MIT（**任务设计可合法借鉴**）；GR00T N1.7 起商业许可可用，SONIC 权重适用 NVIDIA Open Model License。开源用途下均无障碍。

---

## 12. 显式否决（勿重复讨论）

| 方案 | 否决理由 |
|---|---|
| 从头自建模拟器 / benchmark | 数年工程量；与社区不可比 |
| fork Arena / Genie Sim / SIMPLE | 上游剧变 → merge 考古；私有分叉损害可比性 |
| **Genie Sim 作为运行时依赖** | 参考设计不能依赖一个做了十件事的大 repo 才能跑；改为 submodule 资产源 |
| SIMPLE 作为底座 | 渲染约 4 fps，与目标统计规模差 2–3 个数量级；SONIC 未集成（其 README 仍是未勾选 TODO）。**任务设计免费借鉴即可** |
| **Genesis 作为默认后端** | 无已验证 VLA checkpoint 生态；默认路径必须有东西能立刻跑 |
| mjlab | 设计上不为视觉策略提供 RGB 渲染 |
| LeIsaac | SO101 中心；面向数据生产而非评测 |
| LeRobot async inference 作为 policy server | 无法满足依赖隔离；缺 policy reset；优化目标与 episodic 评测相反 |
| 双 policy server | 复杂度翻倍 |
| 模型代码放进本 repo | 依赖冲突使 M 上限约 10 |
| 解耦上肢 + 绕开 SONIC | SONIC 论文明确此类动作空间难以实现手脚协调的 loco-manipulation |
| 纯下肢运控评测 | 评测对象是完成操作任务的能力 |
| **把 XPolicyLab 当作与模型并列的一层** | 它是策略服务框架不是模型；这么拍平会丢掉 sim/real 对称性所在的执行层（v2.0 曾犯此错，v2.1 修正） |
| **"复现危机"叙事** | 过度悲观且取证有偏（GitHub issue 是投诉渠道）。生态健康，改为"协议敏感性"框定 |

---

## 13. 待验证事项（诚实标注）

1. **Arena 跟随 Isaac Lab 的哪个分支** —— 直接影响 pin 策略与迁移风险
2. **G1 本体变体、手部构型、相机配置** —— SONIC 需带腿标准 G1；动作空间每手 7 维，指向 7-DoF 手（如 Dex3-1）
3. **`isaaclab_arena_g1/` 与 SONIC 的衔接方式** —— 未找到 Arena 原生集成 SONIC 的证据
4. **协议能否承载 78 维 latent token** —— 需扩展 schema 与否未验证
5. **Genesis Nyx 实测渲染吞吐** —— 宣称 1080p ≤4ms 为厂商自证，评估门实测
6. **Demo 2 的可行性** —— 「跨后端排序不一致」是否真的会出现，未经验证；若排序高度一致，该 demo 的立论需调整（**但这本身也是有价值的结论**）
7. **RealRunner 与 SimRunner 的接口能否真正统一** —— 时间语义差异（步进 vs 墙钟）是否会迫使接口分叉，需在 Phase 1 最小真机链路上验证
8. **新机器人接入的实际工期** —— 无公开量化数字；**此项被物理验证锁死，AI agent 加速有限**

---

## 附：关键链接

| 项目 | 链接 |
|---|---|
| openpi（π0/π0.5 + LIBERO 工作流） | https://github.com/Physical-Intelligence/openpi |
| LeRobot（`lerobot-eval`） | https://github.com/huggingface/lerobot |
| LIBERO | https://github.com/Lifelong-Robot-Learning/LIBERO |
| Isaac Lab-Arena | https://github.com/isaac-sim/IsaacLab-Arena |
| Newton Physics | https://github.com/newton-physics/newton |
| XPolicyLab | https://github.com/XPolicyLab/XPolicyLab |
| RoboDojo | https://github.com/RoboDojo-Benchmark/RoboDojo |
| Isaac-GR00T / GR00T-WBC | https://github.com/NVIDIA/Isaac-GR00T · https://github.com/NVlabs/GR00T-WholeBodyControl |
| Genie Sim（G2 资产 submodule） | https://github.com/AgibotTech/genie_sim |
| SIMPLE（任务设计参考，MIT） | https://github.com/physical-superintelligence-lab/SIMPLE |
| Genesis World（评估门） | https://github.com/Genesis-Embodied-AI/genesis-world |
| AutoEval | https://auto-eval.github.io/ |
