# 单机工程环境全景认知法

> **开工前的第一份交付物，不是代码，而是一张可信的机器事实图。**
>
> 一个人做项目时，开发机往往同时承担工作站、测试机、中间件宿主、浏览器实验场、
> 代理入口和证据归档站。不了解它，就无法判断一次失败究竟来自代码、依赖、资源、
> 网络，还是上一轮留下的现场。

---

## 一、这套方法解决什么问题？

个人项目最容易出现一种错觉：

```text
代码在我的电脑上
→ 我当然知道它怎么运行
→ 出问题时再看
```

实际情况往往是：

- Java、Node.js、Python、Docker 和浏览器来自不同安装路径；
- 环境变量里仍保留旧版本工具链；
- 中间件既可能由 Windows 服务启动，也可能来自 Docker Compose；
- 多个仓库声明了不同的数据库、缓存、消息队列和端口；
- 代理、VPN、WSL、Docker 网桥共同影响同一条网络链路；
- 16GB 内存既要运行 IDE，也要运行容器、JVM、浏览器和自动化工具；
- 上一轮测试留下的进程、端口、容器和工作树，会混入下一轮结论。

所以，开工前真正要回答的是：

> **这台机器现在是什么状态？项目声称需要什么？两者之间还有哪些差距？**

---

## 二、四层环境模型

不要把“环境”只理解成 `PATH`。一台开发机至少有四层事实。

### 1. 物理资源层

- CPU 型号、核心数与虚拟化能力；
- 内存总量、当前余量与提交限制；
- GPU 及驱动；
- 磁盘容量、文件系统、剩余空间与主要数据目录；
- 网络适配器、无线 / 有线链路与电源策略。

### 2. 操作系统层

- Windows 版本、补丁、区域、编码与时区；
- PowerShell、`cmd.exe`、终端与默认 Shell；
- Windows 服务、计划任务、启动项；
- WSL、Hyper-V、Docker Desktop 与虚拟交换机；
- 防火墙、代理、VPN、DNS、路由与动态端口。

### 3. 工具链与中间件层

- Git、JDK、Maven、Node.js、pnpm、Python、Go 等版本与真实可执行路径；
- MySQL、Redis、RabbitMQ、RocketMQ、Nacos、MinIO、OpenSearch 等安装方式；
- 容器镜像、Compose Profile、数据卷、日志目录和监听端口；
- 浏览器及自动化运行时，例如 Chromium / Playwright；
- 环境变量、证书、凭据入口和本地配置覆盖。

### 4. 项目与证据层

- 项目根目录、Git worktree、分支、远程仓库和未提交改动；
- README、构建清单、Compose、脚本与 CI 声明的运行条件；
- 哪些能力可由公开 Runner 复跑，哪些只能由本机实验室证明；
- 当前冻结基线、验证产物、日志和恢复步骤。

四层必须一起看。只看软件版本，不看端口和进程；只看 Docker，不看宿主代理；只看
README，不看真实路径，都会得到一张不完整的地图。

---

## 三、先给事实分级

每条环境结论都标记来源，避免把猜测写成事实。

| 等级 | 含义 | 例子 |
| --- | --- | --- |
| **OBSERVED / 已观察** | 由当前机器命令、文件或运行结果直接证明 | `java -version`、监听端口、Git HEAD |
| **DECLARED / 已声明** | 项目文档或清单要求如此，但尚未在本机验证 | `pom.xml` 指定 JDK 17 |
| **INFERRED / 已推断** | 由多个仓库和配置反推，仍需实测确认 | Compose 与脚本共同指向 Redis 6379 |
| **UNKNOWN / 未知** | 当前证据不足，不能下结论 | 某次代理超时是否由动态端口耗尽导致 |

最危险的不是 `UNKNOWN`，而是把 `INFERRED` 写成 `OBSERVED`。

---

## 四、串行认知流程

完整流程保持串行：

```text
只读采样
→ 阅读仓库声明
→ 反推依赖与端口
→ 对照真实机器
→ 建立资源预算
→ 最小链路验证
→ 记录差异和停止线
→ 才开始修改
```

串行不是为了慢，而是为了保住因果关系。一次同时启动五套中间件、修改代理、更新
JDK 和移动项目目录，最后即使恢复，也很难知道是哪一步起作用。

---

## 五、硬件与系统快照

Windows 可以先做一轮只读采样：

```powershell
Get-CimInstance Win32_ComputerSystem |
  Select-Object Manufacturer, Model, TotalPhysicalMemory

Get-CimInstance Win32_Processor |
  Select-Object Name, NumberOfCores, NumberOfLogicalProcessors, VirtualizationFirmwareEnabled

Get-CimInstance Win32_OperatingSystem |
  Select-Object Caption, Version, BuildNumber, LastBootUpTime,
    FreePhysicalMemory, TotalVisibleMemorySize

Get-Volume |
  Select-Object DriveLetter, FileSystem, SizeRemaining, Size

Get-TimeZone
Get-Culture
```

这一步只建立基线，不立即“优化”系统。发现内存紧张、磁盘不足或虚拟化未启用时，先
记录它会影响哪些实验，再决定是否改变机器。

---

## 六、工具链不能只看版本，还要看路径

同一命令可能命中多个安装位置。至少同时看“版本”和“解析到哪里”：

```powershell
Get-Command git, java, mvn, node, pnpm, python, docker -ErrorAction SilentlyContinue |
  Select-Object Name, Source, Version

where.exe git
where.exe java
where.exe node

git --version
java -version
node --version
pnpm --version
docker version
```

环境变量也要检查，但原始输出可能包含用户名、代理地址、访问令牌或私有路径，不能
直接粘进公开 Issue、README 或日志：

```powershell
Get-Item Env:PATH, Env:JAVA_HOME, Env:MAVEN_HOME,
  Env:NODE_OPTIONS, Env:DOCKER_HOST,
  Env:HTTP_PROXY, Env:HTTPS_PROXY, Env:NO_PROXY -ErrorAction SilentlyContinue
```

检查时重点回答：

- `PATH` 中是否同时存在多个 JDK / Node / Python；
- IDE、终端、CI 脚本和桌面工具是否使用同一个可执行文件；
- 项目要求的版本是“已安装”，还是“当前真正命中”；
- 版本管理器、系统安装包和项目 Wrapper 是否互相覆盖；
- 环境变量是否只对当前 Shell 生效，还是已经写入用户 / 系统范围。

---

## 七、从仓库反推中间件

不要靠记忆列中间件。让仓库自己说话。

优先阅读：

```text
README / docs
pom.xml / build.gradle
package.json / lockfile
compose.yaml / Dockerfile
.env.example / application*.yml
tools / scripts
.github/workflows
```

建立一张对照表：

| 能力 | 仓库声明 | 本机实现 | 端口 / 目录 | 启停入口 | 当前状态 |
| --- | --- | --- | --- | --- | --- |
| 数据库 | MySQL 8 | Docker / Windows 服务 | 项目声明端口 | Compose / Service | 未观察 |
| 缓存 | Redis | Docker | 项目声明端口 | Compose Profile | 已停止 |
| 消息队列 | RabbitMQ / RocketMQ | Docker | 项目声明端口 | 专项脚本 | 按需 |
| 服务发现 | Nacos | Docker | 项目声明端口 | Full Lab | 按需 |
| 对象存储 | MinIO | Docker | 项目声明端口 | Compose Profile | 按需 |

一组仓库同时存在时，这张表尤其重要。它可以发现：

- 两个项目争用同一端口；
- 一个项目用系统 MySQL，另一个以为自己连接的是容器 MySQL；
- README 已迁移版本，但本地数据卷仍来自旧镜像；
- CI 安装了基础依赖，却漏了完整测试真正使用的可选组件。

---

## 八、项目位置与 Git 拓扑

“项目都在桌面”只是位置描述，不是 Git 事实。还要确认：

- 每个目录是不是独立仓库；
- 是否存在 worktree；
- 当前分支和上游是谁；
- 本地 HEAD、远程分支和 PR head 是否一致；
- 是否有未跟踪文件、用户草稿或生成物；
- 脚本、IDE 配置、快捷方式、任务计划是否写死旧绝对路径。

只读检查：

```powershell
git status --short --branch
git worktree list
git remote -v
git branch -vv
git log --oneline --decorate -5
```

移动项目目录前，先搜索绝对路径依赖。普通 Git 仓库整体移动通常没有问题；正在使用的
worktree、IDE 工作区、脚本、容器挂载、服务配置和计划任务可能仍指向旧位置。

---

## 九、网络、代理、Docker 与 WSL 必须分层诊断

一次“网络超时”至少可能来自：

```text
目标服务未启动
本地端口未监听
代理进程正常但上游节点失效
DNS 或路由异常
VPN / Docker / WSL 网段冲突
MTU、NAT 或连接跟踪问题
Windows 动态端口耗尽
应用自身超时或重试风暴
```

诊断原则：

1. 先验证本地监听；
2. 再验证直连；
3. 再验证代理；
4. 再进入 Docker / WSL；
5. 一次只改变一个变量；
6. 重启前保存连接、日志和事件现场。

本主题的实战版本见 PlainJournal 的
[《本地开发网络与 Windows 故障边界》](https://github.com/NoctilumeDev/PlainJournal/blob/main/docs/07-local-development-network.md)。
它记录了代理上游、动态端口、Docker/VPN 候选原因、压测停止线和恢复顺序之间应该
如何区分，而不是把时间相近的现象强行归为同一根因。

---

## 十、给单机建立资源预算

一台机器不能因为“理论上装得下”就默认“可以同时跑”。

先列出每一类负载：

```text
IDE / Codex / 终端
Docker Desktop / WSL
数据库 / 缓存 / MQ / 搜索 / 对象存储
多个 JVM
前端开发服务器
浏览器与 Playwright
压测工具
代理与 VPN
```

然后定义三条线：

- **启动线**：启动前至少保留多少内存、磁盘和端口空间；
- **升压线**：达到什么指标才进入下一档并发；
- **停止线**：出现交换、持续 GC、代理异常、网络事件或业务不一致时立即停止。

资源受限时，正确方法通常是：

> **拆 Profile、分批启动、串行验证、及时清理。**

不是让所有中间件常驻，再把随机失败解释为代码不稳定。

---

## 十一、环境变更必须可回退

环境认知阶段默认只读。需要修改时，先记录：

- 原值是什么；
- 为什么改；
- 影响用户范围还是系统范围；
- 哪些项目依赖它；
- 如何验证；
- 如何恢复。

高风险动作包括：

- 修改系统 `PATH`、代理、路由、防火墙和网卡跃点；
- 扩大动态端口范围或改变 TCP 全局参数；
- 清空 Docker 卷、数据库目录或依赖缓存；
- 批量升级 JDK、Node、Python 或中间件；
- 在未确认路径时递归移动、覆盖或删除项目。

没有证据时，不以“重装一遍”代替诊断。

---

## 十二、环境事实表模板

每次大型工程开始前，可以保存一份短快照：

```markdown
# 环境事实快照

## 主机
- OS / Build：
- CPU / RAM / Disk：
- 最近启动时间：

## 工具链
- Git：版本 / 路径
- JDK：版本 / 路径
- Node / pnpm：版本 / 路径
- Python / Go：版本 / 路径
- Docker / WSL：版本 / 状态

## 中间件
- 名称 / 版本 / 启动方式 / 端口 / 数据目录 / 当前状态

## 项目
- 仓库 / 路径 / 分支 / HEAD / 上游 / 工作树状态

## 网络
- 直连 / 代理 / DNS / VPN / Docker 网段 / 动态端口

## 资源预算
- 本轮启动批次：
- 升压线：
- 停止线：
- 清理顺序：

## 未知项
- 仍需实测确认的推断：
```

快照应脱敏。不要记录密码、令牌、私钥、完整内部地址或个人目录清单。

---

## 十三、常见错误

### 错误 1：只运行 `--version`

版本正确不等于路径正确，也不等于 IDE、脚本和 CI 使用同一份工具链。

### 错误 2：只看已安装软件列表

项目真正使用什么，必须从配置、端口、进程和运行结果交叉证明。

### 错误 3：先改系统，再找原因

同时换代理、改路由、重启 Docker、升级依赖，会直接毁掉因果链。

### 错误 4：把单次成功当作环境稳定

冷启动、重复运行、清理后重启和资源升压，可能暴露完全不同的问题。

### 错误 5：把本机路径写进公开文档

公开文档保存可移植规则；机器级路径、凭据和一次性流水留在脱敏证据或本地记录中。

### 错误 6：把“我的电脑能跑”当成公共证明

本机证明和公开 CI 各自回答不同问题。如何把两者闭合，见
[《单兵工程公共验证闭环法》](public-verification-loop.md)。

---

## 十四、最终检查清单

开始正式施工前，至少确认：

- [ ] 主机硬件、系统、磁盘和虚拟化已观察；
- [ ] 工具链版本与真实路径已对照；
- [ ] 关键环境变量已检查并脱敏；
- [ ] 中间件由仓库声明反推，并与本机端口 / 进程核对；
- [ ] 项目路径、分支、worktree、远程和未提交改动已确认；
- [ ] 网络、代理、Docker、WSL 的边界没有混写；
- [ ] 本轮资源批次、升压线、停止线和清理顺序已定义；
- [ ] 所有推断都标注为推断，未知项没有被伪装成结论；
- [ ] 修改环境前已经保存原值和回退方式；
- [ ] 最小真实链路已在当前机器上重建。

---

## 十五、最终箭头

```text
先看机器
→ 再读仓库
→ 用仓库反推依赖
→ 用机器验证声明
→ 给资源和网络划边界
→ 保存差异与未知项
→ 最小链路重建
→ 才进入正式施工
```

这套方法的核心不是“把电脑信息收集得越多越好”，而是：

> **在动手之前，先建立一张足以解释成败、支持回退、约束实验的环境事实图。**

环境事实图建立后，进入[《单兵工程法 V1》](solo-engineering-method.md)施工；系统真实运行中的
故障定位、控制变量、恢复与清理验收，继续见
[《单兵工程法 V2：运行诊断与运维验收篇》](solo-engineering-runtime-diagnostics.md)。
