# Docker Desktop Windows 套接字崩溃：无损恢复与停止边界

> **恢复目标不是“让弹窗消失”，而是在不触碰镜像、容器、卷和 WSL 数据盘的前提下，证明 Docker 引擎重新可用。**

本文记录一类 Windows 上的 Docker Desktop 启动故障，以及适用于个人开发机的低风险恢复边界。它不是通用根因声明，也不把一次恢复成功包装成上游缺陷已经根治。

## 一、适用症状

Docker Desktop 启动后报告 `An unexpected error occurred`，错误包含：

```text
listening on unix://...
remove ...sock: The file cannot be accessed by the system.
```

常见路径包括：

```text
%LOCALAPPDATA%\Docker\run\sailor-ingest.sock
%LOCALAPPDATA%\docker-secrets-engine\engine.sock
```

Windows 上这些 0 字节文件可能是 AF_UNIX socket 对应的 NTFS reparse point。Docker Desktop 异常退出后，残留后台进程或内核状态可能让下一次启动无法替换旧 socket，随后进入重复启动、重复失败的循环。

## 二、证据分级

### OBSERVED / 已观察

2026-09-01，在 Docker Desktop 4.88.1（Windows、WSL 2 后端）上观察到：

1. 首次启动因 `sailor-ingest.sock` 无法移除而失败；
2. 完整停止 Docker Desktop 并隔离纯运行时 `run` 目录后，启动继续前进；
3. 第二次启动因 `docker-secrets-engine\engine.sock` 无法移除而失败；
4. 确认该目录只有一个 0 字节 reparse point 后，将目录改名留档并重建空目录；
5. 再次冷启动后，`docker desktop status` 返回 `running`；
6. `docker version` 成功读取服务端，`docker run --rm hello-world` 成功拉取并运行容器。

本机同时保留了更早日期的同类 socket 隔离目录，说明这是复发现象，不应解释为单次项目构建造成。

### EXTERNAL / 外部证据

- Docker 官方故障排查文档把 Restart、Clean up data 与 Reset to factory defaults 区分为不同风险级别；后两者可能丢失设置或数据：<https://docs.docker.com/desktop/troubleshoot-and-support/troubleshoot/>
- Docker 的公开问题记录了同一类 Windows AF_UNIX stale socket、孤儿 `com.docker.backend` 进程和启动循环；该问题在本文记录时仍为 Open：<https://github.com/docker/desktop-feedback/issues/448>

### INFERRED / 已推断

当前证据支持“Docker Desktop 的 Windows socket 生命周期缺陷被异常退出触发”，但不能证明每一次相同文案都只有这一种原因。杀毒软件、文件系统故障、权限策略和版本回归仍需分别排查。

## 三、先确认不是项目故障

这类错误发生在 Docker Desktop 后台服务初始化阶段，早于 Compose 构建或业务容器启动。应先分层：

```text
Docker Desktop 后台尚未启动
→ Docker daemon 不可访问
→ Compose / 项目构建自然无法开始
→ 不能把宿主故障记成项目构建失败
```

最低限度保存：

```powershell
docker desktop status
docker version
wsl --list --verbose

Get-CimInstance Win32_Process |
  Where-Object { $_.Name -match 'docker|com\.docker' } |
  Select-Object ProcessId, ParentProcessId, Name, ExecutablePath, CommandLine
```

日志通常位于：

```text
%LOCALAPPDATA%\Docker\log\host\monitor.log
```

公开证据应脱敏，不直接上传完整本机日志、用户名、代理、令牌或内部路径。

## 四、低风险恢复顺序

### 第 1 步：先正常停止

```powershell
docker desktop stop --timeout 60
```

如果后台已经崩溃且正常停止失败，再使用：

```powershell
docker desktop stop --force --timeout 30
```

随后确认没有残留 Docker Desktop 进程。只关闭窗口不等于后台已经退出。

### 第 2 步：停止 Docker 的 WSL 后端

```powershell
wsl --terminate docker-desktop
```

如果机器还运行其他 WSL 发行版，不要无差别执行 `wsl --shutdown`，除非已经确认影响范围。

### 第 3 步：只读检查 socket 目录

```powershell
$dockerRunDir = Join-Path $env:LOCALAPPDATA 'Docker\run'
$dockerSecretsDir = Join-Path $env:LOCALAPPDATA 'docker-secrets-engine'

Get-ChildItem -LiteralPath $dockerRunDir -Force
Get-ChildItem -LiteralPath $dockerSecretsDir -Force
```

只有满足以下条件时，才进入隔离步骤：

- Docker / `com.docker` 进程已全部退出；
- 目录内容已逐项查看；
- 内容是 0 字节 reparse point 等纯运行时 socket；
- 没有配置、密钥、数据库、镜像、卷或未知普通文件。

任何一项不满足，都停止，不猜测。

### 第 4 步：改名隔离，不强删

单个 reparse point 可能连移动或删除都会返回 Windows 错误 1920。此时不要使用恢复出厂，也不要递归清理 Docker 数据根目录。可以在确认目录只含运行时 socket 后，将父目录改名留档，再重建空目录：

```powershell
$recoveryStamp = Get-Date -Format 'yyyyMMdd-HHmmss'
$dockerRunDir = Join-Path $env:LOCALAPPDATA 'Docker\run'
$dockerSecretsDir = Join-Path $env:LOCALAPPDATA 'docker-secrets-engine'

Rename-Item -LiteralPath $dockerRunDir -NewName "run-stale-$recoveryStamp"
New-Item -ItemType Directory -Path $dockerRunDir

Rename-Item -LiteralPath $dockerSecretsDir -NewName "docker-secrets-engine-stale-$recoveryStamp"
New-Item -ItemType Directory -Path $dockerSecretsDir
```

这一步必须逐目录执行并逐次验证。不要把它扩写成对 `%LOCALAPPDATA%\Docker` 的递归删除脚本。

### 第 5 步：冷启动与三级验收

```powershell
docker desktop start --detach
docker desktop status
docker version
docker run --rm hello-world
```

三级验收分别证明：

1. Desktop 状态机进入 `running`；
2. 客户端能够访问真实 daemon；
3. daemon 能完成镜像拉取、容器创建、运行和自动删除。

只看到 Dashboard 打开，不算恢复成功。

## 五、明确禁止的动作

在没有独立备份和更强证据前，不做：

- `Reset to factory defaults`；
- Clean / Purge data；
- 注销或删除 `docker-desktop` WSL 发行版；
- 删除 Docker 的 WSL 数据盘；
- 批量删除镜像、卷、容器或整个 `%LOCALAPPDATA%\Docker`；
- 同时升级 Docker、修改 WSL、关闭安全软件和清理目录；
- 因 Docker 宿主失败而改动业务仓库代码。

这些动作会破坏因果链，部分动作还会造成不可恢复的数据丢失。

## 六、何时停止

出现以下任一情况，停止本地修复并保留现场：

- socket 目录出现普通文件、配置、密钥或未知数据；
- 完整停机后仍存在无法识别的持有进程；
- WSL 自身报虚拟化、VHDX、发行版注册或文件系统错误；
- 每隔离一处就出现新的非 socket 故障；
- `docker version` 能连接 daemon，但真实容器仍失败；
- 修复需要恢复出厂、重装或删除数据盘。

此时优先生成脱敏诊断、核对 Docker 当前版本的 release notes，并等待上游修复。诊断包可能包含本机信息；上传前应确认内容和传输边界。

## 七、长期边界

用户侧可以降低复发概率，但不能替代上游修复：

- 关机或重启 Windows 前，先让 Docker Desktop 正常停止并确认状态；
- 不连续点击启动入口，不在后台退出过程中重复拉起；
- 保持 Docker Desktop 与 WSL 在受支持版本；
- 升级前阅读 Windows 相关 release notes，升级后做冷启动和真实容器验证；
- 隔离目录先保留，经过后续冷启动验证后再单独决定是否清理；
- 把“宿主恢复成功”和“项目容器构建通过”记录为两条证据。

真正的根治条件是：Docker Desktop 能在异常退出后可靠清理或替换 stale socket，并且不再留下会阻断后续启动的孤儿后台。上游明确修复并经本机重复冷启动验证前，应把它保留为已知环境边界。

## 八、最终箭头

```text
保存错误与日志
→ 区分宿主失败和项目失败
→ 正常停止，必要时强制停止
→ 终止 Docker WSL 后端
→ 只读确认目录内容
→ 改名隔离纯运行时 socket
→ 冷启动
→ status + daemon + 真实容器三级验收
→ 保留上游未知，不伪称根治
```

核心原则：

> **能无损恢复，就不以重置代替诊断；无法证明安全，就在停止线前停下。**
