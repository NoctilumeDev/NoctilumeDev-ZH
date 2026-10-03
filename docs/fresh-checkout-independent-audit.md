# 从对话记忆到工程记忆：Fresh Checkout 三阶段独立复验

> 工程实证记录 · 2026-08-25
>
> 公开摘要，不是学术论文，也不宣称仓库已经零缺陷。

## 为什么做这次实验

连续协作中的模型可以继承需求背景、设计取舍、失败记录与作者解释。这种上下文很有价值，但它也可能掩盖另一个更重要的问题：

> 当开发对话全部消失，只剩 GitHub 上公开的代码、README、测试、CI、Release、合同与证据时，一个陌生审查者还能否正确理解、运行和复核这些项目？

[《从工具增益到协同复利》](from-tool-gain-to-collaborative-compounding.pdf)把工程产出描述为人、模型、工作流、协议、上下文、工件和任务耦合的共同函数。这次实验不是重复论文体复盘，而是对其中一条判断做更严格的后续工程复验：有价值的上下文最终应从对话记忆迁移为仓库可以独立读取和执行的工程记忆。

按作者保留的生成过程记录，MiniSpringBoot 的主要前期实现阶段在 Trae 中由 **GLM-5.3** 与 **DeepSeek-V4 Pro** 参与。这个来源信息只用于说明项目曾经历跨工具、跨模型协作，不用于把仓库质量归功或归责于某一个模型，也不是模型能力排名。

## 实验假设

把审查者实际可用的上下文简化为：

```text
C_total
= C_dialogue
+ C_artifact
+ C_execution
```

本次实验主动控制为：

```text
C_dialogue ≈ 0          不提供原开发对话与历史解释
C_artifact = public     只读取公开 GitHub 仓库资产
C_execution = fresh     每一阶段使用独立 fresh checkout
```

因此，实验测量的不是“同一个模型记得多少”，而是“公开仓库自己保存了多少可恢复的工程知识”。

## 三阶段角色隔离

实验串行执行，避免多个高强度任务同时争抢单机资源，也避免角色之间共享未冻结的中间判断。

角色隔离不是预设 Agent 会故意作恶，而是拒绝把优化者的自我报告直接当成外部事实：施工者可以改代码，但不应同时定义评分、保管原始证据并宣布最终完成。与其在提示词里不断枚举“不许修改测试、不许修改报告”，不如从流程上让发现、施工与验收彼此复核。

### A · Fresh Auditor

A 从空目录克隆 `NoctilumeDev` 下 8 个公开仓库，只做审查：

- 不修改仓库；
- 不询问作者设计历史；
- 先识别每个仓库公开声明的身份与成熟度；
- 优先执行仓库自己声明的构建、测试、静态检查和 CI 等价命令；
- 再检查 README 声明、版本坐标、失败路径与公开证据是否一致；
- 只提交带定位证据与复现路径的问题。

A 的报告随后冻结为 `F1`。B 只能处理这份冻结问题集，不能依赖 A 的隐含推理过程。

### B · Fresh Fixer

B 在第二个空目录重新克隆相同仓库，并逐条处理 `F1`：

```text
复现
→ 判断 valid / invalid
→ 修复 valid finding
→ 增加约束或回归验证
→ 提交并推送
→ 远端回读
```

B 没有把“已发现”直接当作“已修复”，四项问题都经过独立复现后才进入修改。

### C · Fresh Re-auditor

C 在第三个空目录只克隆修复后的公开 `HEAD`：

- 不读取 A 的报告；
- 不读取 B 的修复说明；
- 不读取原开发对话；
- 不修改文件；
- 从公开仓库重新判断边界并完整复审。

这一步用新的残余问题集 `F2` 检查修复是否真正进入公共工程资产，也避免“施工者自己证明自己”。

## 审查范围与坐标

8 个仓库全部从 GitHub fresh clone。只有 MiniSpringBoot 与 VeriTrail 因 `F1` 修复改变了坐标，其他 6 个仓库在 A 与 C 之间保持不变。

| Repository | A 审计坐标 | C 复审坐标 |
| --- | --- | --- |
| DarkRoomLibrary | [`0a5d4b1`](https://github.com/NoctilumeDev/DarkRoomLibrary/commit/0a5d4b133ef1ea02add86e48b1df77e203b670ca) | 同左 |
| FlowKernel | [`33bca64`](https://github.com/NoctilumeDev/FlowKernel/commit/33bca64f27c8f82d5ba1b8a434b25058583c7797) | 同左 |
| InkNarratives | [`b41272f`](https://github.com/NoctilumeDev/InkNarratives/commit/b41272f35a0679300d09239b4831539b6eb1e0b5) | 同左 |
| MiniSpringBoot | [`aff84d8`](https://github.com/NoctilumeDev/MiniSpringBoot/commit/aff84d8ad9259b3b1375f25b3c929ba05dfce64d) | [`0a6dbbc`](https://github.com/NoctilumeDev/MiniSpringBoot/commit/0a6dbbcc1834b78f25f69f3a2db6b2635c32c0cc) |
| NoctilumeDev | [`587f2e6`](https://github.com/NoctilumeDev/NoctilumeDev/commit/587f2e6d0ef3e0045411494cd5f1bfa4cadec627) | 同左 |
| PlainJournal | [`ce5b5a8`](https://github.com/NoctilumeDev/PlainJournal/commit/ce5b5a805b48b16e22fc042ed966bca5a4fd1882) | 同左 |
| PlainJournalPro | [`100207b`](https://github.com/NoctilumeDev/PlainJournalPro/commit/100207be8ae309b252feeb0b72dc61b1858cf049) | 同左 |
| VeriTrail | [`22b9443`](https://github.com/NoctilumeDev/VeriTrail/commit/22b944308efd7aadadfffda8433c5d8955d7f559) | [`30f6c8a`](https://github.com/NoctilumeDev/VeriTrail/commit/30f6c8a26da3831fc3b1141f7b4b3e3c14e6dafe) |

审查者没有把未承诺的能力当作缺陷：`Planned`、`Prototype`、`Frozen`、`Released` 与教学型实现均按仓库自己的公开合同理解。Docker、MySQL 或平台能力缺失也与代码 finding 分开记录。

## F1 · 首轮独立发现

| Severity | P0 | P1 | P2 | P3 | Total |
| --- | ---: | ---: | ---: | ---: | ---: |
| A findings | 0 | 3 | 0 | 1 | 4 |

四项 finding 均被 B 独立复现为有效问题：

1. **MiniSpringBoot 依赖声明不准确**：README 的“零第三方运行时依赖”没有表达可选 HikariCP 路径与强制传递依赖之间的真实边界。
2. **MiniSpringBoot 嵌套证据字节不稳定**：Windows `autocrlf` 可改变 M10 证据树中的文本字节，破坏公开复验坐标。
3. **MiniSpringBoot 仓库规模指标失配**：README 中给出的统计命令不能复现同一处展示的文件数与行数。
4. **VeriTrail 发布后版本坐标滞后**：`v0.12.0` 已冻结，但后续 Core 修改仍构建为 `0.12.0`，使发布坐标和开发坐标发生混淆。

## B · 修复闭环

MiniSpringBoot 通过提交 [`0a6dbbc`](https://github.com/NoctilumeDev/MiniSpringBoot/commit/0a6dbbcc1834b78f25f69f3a2db6b2635c32c0cc)完成：

- 把依赖声明改为可验证的“零强制传递第三方运行时依赖”；
- 显式说明可选 HikariCP 路径；
- 为嵌套证据目录固定 LF 字节合同；
- 新增仓库公共合同验证脚本与 CI 门禁；
- 统一 README 的统计口径。

VeriTrail 通过提交 [`30f6c8a`](https://github.com/NoctilumeDev/VeriTrail/commit/30f6c8a26da3831fc3b1141f7b4b3e3c14e6dafe)把开发分支坐标推进为 `0.12.1.dev0`，同时保持已发布的 `v0.12.0` 不可变，并新增版本合同验证。

对应的公共 CI 均成功：

- [MiniSpringBoot CI · 32791232611](https://github.com/NoctilumeDev/MiniSpringBoot/actions/runs/32791232611)
- [VeriTrail Public CI · 32793086513](https://github.com/NoctilumeDev/VeriTrail/actions/runs/32793086513)
- [VeriTrail Browser Smoke · 32793086555](https://github.com/NoctilumeDev/VeriTrail/actions/runs/32793086555)

## F2 · 修复后独立复审

| Severity | P0 | P1 | P2 | P3 | Total |
| --- | ---: | ---: | ---: | ---: | ---: |
| C findings | 0 | 0 | 2 | 1 | 3 |

C 没有重新发现任何 `F1` 问题，但从新 `HEAD` 独立发现三项残余维护问题：

1. **DarkRoomLibrary · P3**：后端公开 **line coverage** 证据没有完全对齐。fresh checkout 实测为 `3811 / 5277`（`72.219064%`），公开 Pages 为 `3813 / 5277`（`72.256964%`），文档为 `3812 / 5277`（`72.238%`）。差异很小，但公共证据应使用同一来源与同一取整合同。
2. **MiniSpringBoot · P2**：`TransactionManager` 在恢复 `autoCommit=true` 抛出异常时可能跳过 `close()`，事务失败路径仍有资源释放闭环缺口。
3. **VeriTrail · P2**：在文档声明的完整可选环境中直接运行 E3 接受命令会受到 pywin32 `scripts` 命名空间影响；模块方式 `python -m scripts.entry_layer_e3_acceptance` 可以成功，说明入口文档和执行方式仍需收敛。

三项均由主代理再次独立复现。C 报告中曾把 DarkRoomLibrary 的覆盖指标误称为 instruction coverage；回读原始命令和数字后确认应为 line coverage。这个小偏差本身也说明：独立审计可以减少原开发上下文的偏见，但审计报告仍需要证据回读，不能因为“来自 fresh auditor”就免于复核。

## 可以成立的指标

| Metric | Result | 边界 |
| --- | ---: | --- |
| Discovery Precision | `4 / 4 = 100%` | A 的四项 finding 均被 B 独立复现 |
| F1 Repair Closure | `4 / 4 = 100%` | C 未在新 HEAD 重新发现任何 F1 |
| Residual F2 | `3` | 2 项 P2、1 项 P3；仍是有效维护输入 |
| Artifact Comprehension | 定性通过 | A/C 能自行区分 Planned、Prototype、Frozen 与 Released 边界 |
| Discovery Recall | **不宣称** | 没有完整、外部独立的缺陷金标准，无法计算召回率 |

`F1` 完全闭环不等于仓库零缺陷；C 又发现 `F2`，恰好阻止了“修完首轮问题即宣布完美”的错误结论。没有 P0/P1 也只表示本次范围内没有被验证的高严重度残余，不表示高严重度问题在逻辑上不存在。

## 这次实验说明了什么

### 1. 工程知识已经部分离开对话

fresh auditor 没有作者解释，仍能从 README、状态标识、测试、CI、Release、合同与失败输出中恢复大部分项目边界。这说明连续协作中的一部分知识已经从人的记忆和模型上下文，迁移成了仓库自己的工程记忆。

### 2. 串行角色隔离比“同一对话再看一遍”更有审查价值

A 发现、B 施工、C 复审，三者不共享未冻结思路。A 不接受开发阶段的“已经完成”，B 不接受 A 的“这一定是 bug”，C 也不接受 B 的“已经修好”；每一步都重新接触可执行事实。串行并没有复制组织开销，而是用最小角色分离保留了发现、修复和验收之间的制衡。这与[硅谷中国特色改良版_拖鞋（妥协）版单兵工程法](solo-engineering-method.md)的核心一致：压扁组织，但不删除约束。

### 3. 干净执行环境会暴露本地连续环境看不见的问题

换行字节、版本坐标、可选依赖命名空间、公开覆盖率来源，都不是“业务页面能打开”就会自动暴露的问题。[单机工程环境全景认知法](single-machine-engineering-environment.md)负责看清机器，[单兵工程公共验证闭环法](public-verification-loop.md)负责把本地事实送到公开平台；Fresh Checkout 复验把两者连接起来。

### 4. 这为论文体复盘提供工程证据，但不是科学证明

本次实验支持这样一个工程判断：

> 强模型的连续上下文可以提高实现效率；去掉对话上下文后的 fresh checkout 更适合检验知识是否真正沉淀进工程资产。

但它不是随机对照实验，没有模型能力基线，也没有完整缺陷金标准。因此不能据此宣布某个模型更强、某套工具普遍最优，或仓库已经无缺陷。

## 与“单兵工程三剑客”的关系

```text
单机环境全景认知
    ↓ 解决“在哪台机器、以什么边界执行”
拖鞋（妥协）版单兵工程法
    ↓ 解决“一个人如何保留完整工程职责”
公共验证闭环法
    ↓ 解决“本地事实如何成为公开事实”
Fresh Checkout 三阶段复验
    ↓ 检查“离开原对话后，这些事实是否仍可独立恢复”
```

前三篇是方法，Fresh Checkout 是一次有冻结坐标、独立角色、公开 CI 和残余问题集的工程实证附录。

## 原始工件与止线

原始 A/B/C 报告、执行账本和 findings JSON 保留在离线实验目录；公开仓库只记录可复核摘要、Git 坐标与 CI 坐标，避免把单机绝对路径伪装成公共入口。

关键冻结摘要哈希：

```text
A findings.json
6AC3D7631AEA554B0751AF12217FBB7CE9662146B525A62493A96A6652FA3AB4

C fresh-reaudit-report.md
2F5DFF50DF7194E29E29503648785C9316D37997D0F31EA27A508E232E04FC87

C findings.json
677DD223FD5448165A1D4613824BBEE97685270BAF8A9CA723BBAF0EA2DB4315

C execution-ledger.md
10712F3D8F5A4A0A128C86565CE9406E247933777492529449954981F2D68F1B
```

这次实验的停止线同样明确：

- 不把 precision 写成 recall；
- 不把 F1 闭环写成零缺陷；
- 不把作者提供的模型来源写成精确提交归因；
- 不把工程复验写成学术证明；
- 不让审计角色的文字替代原始命令、坐标与输出。

真正希望保留下来的结论只有一句：

> 对话可以帮助项目成长，但只有进入代码、合同、测试、CI、Release 与可执行证据的知识，才会在对话结束后继续工作。
