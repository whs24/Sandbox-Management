# Agent 沙箱管理实现总览

## 总体结论

Agent 沙箱管理不是简单地“把代码放进 Docker 里跑”。一个可用的 Agent 沙箱系统至少要同时管理五件事：

1. Agent 可以访问什么工具；
2. Agent 生成的代码放在哪里；
3. 代码在哪种隔离环境中运行；
4. 运行时能访问哪些文件、网络和硬件资源；
5. 运行结果如何被结构化采集、审计、复现和清理。

从本次对比的仓库看，比较成熟的实现会把沙箱拆成三层：

- 控制面：`SandboxProvider` / factory / deployment，负责创建、复用、释放沙箱；
- 执行面：Docker、远端 SSH、Kubernetes、SandboxFusion、LocalSandbox 等实际后端；
- 工具面：bash、Python interpreter、read/write file、run project、install tools 等 Agent 可见接口。

对“自动工业工程问题求解科研机器”来说，推荐采用组合式架构：以 AutoResearchClaw 的安全执行协议为核心，以 RD-Agent 的 Docker 资源控制为实验执行器，以 deer-flow/SWE-agent 的 provider、middleware、session 和工具协议做 Agent 控制面，再补充工业工程特有的数据、许可证、求解器和 artifact 管理。

## 1. 为什么 Agent 需要沙箱

工业工程科研 Agent 不是只回答问题，而是会执行动作：

- 生成 Python/Julia/MATLAB/AMPL/OR-Tools/Gurobi 代码；
- 读取数据；
- 安装依赖；
- 调用优化器；
- 跑仿真；
- 启动并行搜索；
- 写结果文件；
- 画图；
- 生成报告；
- 多轮修改代码。

这些动作天然有风险：

- 无限循环或长时间求解耗尽资源；
- 误删工作区或覆盖数据；
- 读取不该读取的文件；
- 通过网络泄露数据；
- 安装恶意包；
- 滥用商业求解器许可证；
- 杀错进程；
- 多任务之间互相污染；
- 实验结果不可复现。

因此，Agent 沙箱的目标不是让 Agent 什么都不能做，而是让它在“可控自由度”内完成研究。

## 2. 沙箱控制面的实现

控制面负责回答：这个 Agent 任务应该使用哪个沙箱？沙箱什么时候创建、复用、释放和销毁？

### 典型接口

deer-flow 的 `SandboxProvider` 是清晰范式：

```python
class SandboxProvider:
    def acquire(self, thread_id: str) -> str: ...
    def get(self, sandbox_id: str) -> Sandbox: ...
    def release(self, sandbox_id: str) -> None: ...
    def reset(self) -> None: ...
```

AutoResearchClaw 的 `create_sandbox(config, workdir)` 是另一种范式：根据配置选择后端。

```python
if mode == "docker":
    return DockerSandbox(...)
if mode == "ssh_remote":
    return SshRemoteSandbox(...)
return ExperimentSandbox(...)
```

SWE-agent 则使用 deployment abstraction：

```python
deployment = get_deployment(config.deployment)
deployment.start()
deployment.runtime.create_session(...)
```

### 推荐设计

我们的系统建议定义：

```python
class SandboxProvider:
    def acquire(task_id, profile) -> SandboxHandle: ...
    def release(handle) -> None: ...
    def destroy(handle) -> None: ...
    def healthcheck(handle) -> Health: ...

class Sandbox:
    def run_project(entry_point, args, timeout, env) -> RunResult: ...
    def run_command(command, timeout) -> CommandResult: ...
    def read_file(path) -> bytes: ...
    def write_file(path, data) -> None: ...
    def list_artifacts() -> list[Artifact]: ...
```

其中 `profile` 应包含：

- 任务类型：排程、路径优化、仿真、预测、实验分析；
- 镜像；
- CPU/内存/GPU；
- 网络策略；
- 数据挂载；
- 许可证挂载；
- 最大运行时间；
- 是否允许安装依赖；
- 是否需要持久 session。

## 3. 执行后端的实现

### 本地后端

本地后端通常只是开发便利，不是安全边界。典型例子：

- deer-flow `LocalSandbox`；
- RD-Agent `LocalEnv`；
- Auto-Deep-Research `LocalEnv`；
- AI-Scientist 的本地 `subprocess.run`。

它们可以提供：

- 快速调试；
- 路径映射；
- 只读挂载；
- 简单 timeout。

但不能承载不可信 Agent 代码。生产环境应默认禁用 Agent 的 host bash。

### Docker 后端

Docker 是最常见的执行后端。成熟实现应包含：

- 每个任务独立容器；
- 工作区 rw 挂载；
- 数据集 ro 挂载；
- 求解器许可证 ro 或受控 env 挂载；
- 非 root 用户；
- 内存限制；
- CPU 限制；
- pids 限制；
- shm 限制；
- GPU device request；
- 默认断网或阶段化联网；
- 容器超时 kill；
- 执行后 remove；
- 日志和 artifact 采集。

AutoResearchClaw 的 DockerSandbox 已具备很多关键点：

```bash
docker run --rm \
  -v staging:/workspace \
  -w /workspace \
  --memory=... \
  --shm-size=... \
  --network none \
  --gpus all
```

RD-Agent 的 DockerEnv 在资源控制和 Docker SDK 使用上更成熟：

```python
client.containers.run(
    image=image,
    command=entry,
    volumes=volumes,
    working_dir=mount_path,
    network=network,
    shm_size=shm_size,
    mem_limit=mem_limit,
    cpu_count=cpu_count,
    **gpu_kwargs,
)
```

### 远端后端

远端后端适合企业真实算力：

- SSH 到专用服务器；
- Kubernetes provisioner；
- Slurm 作业；
- 云沙箱服务；
- SandboxFusion Python interpreter。

deer-flow 的 remote backend 和 AutoResearchClaw 的 SSH sandbox 都体现了这种方向。

我们的系统建议把远端后端分为两类：

- interactive sandbox：用于 Agent 多轮调试；
- batch sandbox：用于长时间求解和大规模实验。

## 4. 工具面的实现

Agent 不应直接获得无限制宿主机权限，而应通过工具面使用沙箱能力。

常见工具包括：

- `bash`：执行 shell 命令；
- `python`：执行短 Python 片段；
- `run_project`：运行完整项目入口；
- `read_file` / `write_file`；
- `grep` / `glob` / `ls`；
- `install_deps`；
- `collect_artifacts`；
- `interrupt`。

SWE-agent 的工具协议值得借鉴：

```python
execution_timeout = 30
install_timeout = 300
total_execution_timeout = 1800
max_consecutive_execution_timeouts = 3
```

deer-flow 的 bash 审计也值得保留：命令在执行前先做风险分类，危险命令 block，中风险命令 warn。

对于工业工程 Agent，建议区分两种执行方式：

- 交互式命令：短 timeout，用于调试、查看文件、运行小测试；
- 实验级运行：长 timeout，必须通过 `run_project`，输出 metrics 和 artifacts。

## 5. 文件系统隔离

文件系统是 Agent 沙箱的核心。

推荐目录结构：

```text
/workspace
  /src            # Agent 可写代码
  /runs           # 实验运行目录
  /artifacts      # 输出图表、模型、解文件
  /logs           # stdout/stderr/solver logs
  /cache          # 任务内缓存
/data             # 只读数据集
/benchmarks       # 只读 benchmark
/licenses         # 只读许可证或受控挂载
/opt/solvers      # 镜像内求解器
```

安全要求：

- Agent 只能写 `/workspace`；
- `/data`、`/benchmarks` 默认只读；
- 入口文件必须是相对路径；
- 禁止 `..`；
- resolve 后仍必须位于 staging/workspace 内；
- symlink 不能逃逸；
- 输出 artifact 必须统一收集。

AutoResearchClaw 的入口校验和 deer-flow 的路径映射都可作为参考。

## 6. 网络策略

网络控制应按阶段而不是简单开关。

推荐策略：

- `none`：完全断网，默认用于正式实验；
- `setup_only`：依赖安装阶段联网，实验阶段断网；
- `pip_only`：只允许包安装；
- `allowlist`：只允许访问内部数据/许可证/包镜像；
- `full`：仅用于可信开发或人工批准任务。

AutoResearchClaw 的 `network_policy` 已体现这种思路。工业工程系统中还应额外控制：

- 许可证服务器访问；
- 内部包仓库访问；
- 禁止外发企业数据；
- 禁止 Agent 任意下载未知脚本。

## 7. 资源限制

工业工程任务尤其容易耗尽资源。资源策略至少包括：

- wall-clock timeout；
- CPU 核数；
- 内存；
- GPU；
- 共享内存；
- pids；
- 磁盘空间；
- 最大输出日志；
- 最大 artifact 大小；
- 最大连续 timeout 次数；
- 总执行预算。

RD-Agent 的 `mem_limit`、`cpu_count`、`shm_size`，SWE-agent 的连续 timeout 和总执行时间，AutoResearchClaw 的 Docker timeout 清理，都值得组合。

## 8. 代码与命令审计

沙箱不是只靠 Docker。执行前应有轻量审计：

- AST 检查：拦截 `subprocess`、`os.system`、`eval`、`exec`、`socket` 等；
- 命令审计：拦截 `rm -rf /`、`curl | bash`、后台 daemon、权限修改等；
- 导入审计：限制危险包；
- 网络审计：记录访问目标；
- 文件审计：记录写入和删除。

AutoResearchClaw 的 `validator.py` 和 deer-flow 的 `sandbox_audit_middleware.py` 是可复用思路。

注意：黑名单不是强安全边界，只是降低事故概率。真正边界仍应由容器、挂载、网络和资源策略提供。

## 9. 结果采集与复现

科研机器必须把“跑过什么”保存下来。

推荐 `RunResult`：

```json
{
  "run_id": "...",
  "entry_point": "run.py",
  "exit_code": 0,
  "timed_out": false,
  "elapsed_sec": 123.4,
  "metrics": {
    "objective": 12345.6,
    "gap": 0.01,
    "runtime": 120.0
  },
  "artifacts": [
    "solution.json",
    "schedule.csv",
    "gantt.png",
    "solver.log"
  ]
}
```

工业工程特有 metrics 可包括：

- objective；
- lower_bound / upper_bound；
- optimality_gap；
- feasibility；
- constraint_violations；
- solve_time；
- simulation_replications；
- average_waiting_time；
- throughput；
- makespan；
- tardiness；
- energy_cost。

AutoResearchClaw 的 `metrics` 解析和 partial result 思路非常适合：即使超时，也应保留 incumbent solution 和阶段性指标。

## 10. 推荐架构

建议最终实现如下：

```text
Agent
  |
  | tool call
  v
Tool Gateway
  - allowlist
  - audit
  - timeout budget
  |
  v
SandboxProvider
  - acquire(task_id, profile)
  - warm pool
  - healthcheck
  - release/destroy
  |
  v
Execution Backend
  - Docker local
  - Kubernetes/Slurm
  - SSH remote
  - Python interpreter service
  |
  v
Sandbox Runtime
  - /workspace rw
  - /data ro
  - resource limits
  - network policy
  - solver/license policy
  |
  v
Result Collector
  - logs
  - metrics
  - artifacts
  - provenance
```

## 11. 对本项目的落地建议

第一阶段可以实现一个 Docker 主后端：

- 复用 AutoResearchClaw 的 `run_project` 思路；
- 复用 RD-Agent 的 Docker SDK 和资源限制思路；
- 加入 deer-flow 风格 `SandboxProvider`；
- 工具层参考 SWE-agent 的 timeout 和 interrupt；
- 默认 `setup_only` 网络策略；
- 定义工业工程 `metrics.json` 和 `artifacts/` 规范。

第二阶段扩展远端执行：

- Kubernetes provider；
- SSH provider；
- Slurm batch provider；
- 轻量 Python SandboxFusion provider。

第三阶段做生产化：

- 镜像签名和扫描；
- 数据访问审计；
- 商业求解器许可证隔离；
- 任务级成本统计；
- 多租户隔离；
- 失败实验自动诊断；
- 基于历史实验的缓存和复现。

整体上，Agent 沙箱管理的核心不是“限制 Agent”，而是把 Agent 的探索能力放进可审计、可复现、可回收、可扩展的实验边界里。

