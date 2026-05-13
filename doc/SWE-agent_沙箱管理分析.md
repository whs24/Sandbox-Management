# SWE-agent 沙箱管理分析

## 结论

SWE-agent 的沙箱管理集中在 `SWEEnv` 与 `swerex.deployment` 抽象上。它默认使用 Docker deployment，启动后创建持久 bash session，Agent 的动作通过 runtime API 在 session 内执行；同时提供文件读写、独立命令执行、repo 初始化/reset、工具安装、命令超时、连续 timeout 退出和总执行时间限制。

SWE-agent 很适合“代码修改型 Agent”的执行环境。对自动工业工程问题求解科研机器，它可以借鉴其持久 shell session、工具协议、repo reset 和动作超时机制；但它不是面向科研实验和工业求解的完整沙箱，Docker 细节主要在外部 `swerex` 依赖中，仓库内缺少工业实验资源策略和指标/artifact 协议。

## 核心代码位置

- `repos/SWE-agent/sweagent/environment/swe_env.py`
- `repos/SWE-agent/sweagent/environment/repo.py`
- `repos/SWE-agent/sweagent/tools/tools.py`
- `repos/SWE-agent/sweagent/agent/agents.py`

## EnvironmentConfig 和 Deployment

`swe_env.py` 中的 `EnvironmentConfig` 默认使用 Docker deployment：

```python
deployment = DockerDeploymentConfig(
    image="python:3.11",
    python_standalone_dir="/root",
)
```

`SWEEnv.from_config()` 会调用 `get_deployment(config.deployment)`，实际部署后端由 `swerex.deployment` 提供。

这是一种优秀的抽象：SWE-agent 不直接把自己绑定死在 Docker 命令上，而是通过 deployment runtime 操作环境。对我们的系统来说，也可以把 Docker、Kubernetes、SSH、Slurm、SandboxFusion 都做成 deployment。

## 启动流程：部署环境和持久 bash session

`SWEEnv.start()` 会初始化 deployment，然后 reset 仓库，并执行 post-startup commands。

核心启动逻辑是：

```python
self.deployment.start()
self.deployment.runtime.create_session(
    CreateBashSessionRequest(
        startup_source=["/root/.bashrc"],
        startup_timeout=10,
    )
)
```

之后 Agent 的大多数 shell 动作都会进入这个持久 bash session。持久 session 的好处是：

- cwd 可以延续；
- shell 环境变量可以延续；
- 激活 virtualenv/conda 后可以复用；
- 工具安装后可持续可见；
- Agent 体验接近真实终端。

对工业工程科研机器来说，这适合“Agent 多轮调试模型代码”的过程。

## communicate：动作执行和超时

`SWEEnv.communicate(input, timeout=25, check=...)` 把命令包装为 `BashAction`：

```python
self.deployment.runtime.run_in_session(
    BashAction(
        command=input,
        timeout=timeout,
        check=check,
    )
)
```

如果 exit code 非 0 且 check 为 true，会抛出异常。它还支持 interrupt session：

```python
self.deployment.runtime.run_in_session(BashInterruptAction())
```

这种设计很适合 Agent 工具执行：每个动作有独立 timeout，必要时可中断当前 shell。

## 文件读写和独立命令

除了持久 session，SWEEnv 还提供：

- `read_file`
- `write_file`
- `execute_command`

这些通过 runtime API 访问环境。区别是：

- `communicate` 用于 Agent 的交互式 shell；
- `execute_command` 用于独立命令；
- file API 用于直接读写文件。

我们的工业工程系统可以采用同样分层：Agent 调试时用 session，系统采集结果时用 file API，健康检查/环境初始化时用 independent command。

## Repo 管理

`environment/repo.py` 负责把 repo 放入沙箱。

`LocalRepoConfig.copy` 会要求本地 repo 是干净的 git repo，然后上传到沙箱路径，并执行 chown：

```python
upload(local_repo, f"/{repo_name}")
chown root ...
```

`GithubRepoConfig.copy` 支持 shallow fetch/checkout，并有 clone timeout。

`SWEEnv._reset_repository()` 会执行 git reset/clean 等命令，将仓库恢复到指定状态。

这对工业工程科研机器很有价值。每个候选方案应从同一个 baseline 开始，Agent 修改代码后产生 diff，实验结束后可 reset 或创建新工作区，保证可复现。

## 工具安装和工具协议

`tools/tools.py` 定义 `ToolConfig`：

```python
execution_timeout = 30
install_timeout = 300
total_execution_timeout = 1800
max_consecutive_execution_timeouts = 3
```

`ToolHandler.install/reset` 会把工具 bundle 上传到环境中，设置 PATH，并运行 install/reset commands。

这说明 SWE-agent 把“Agent 能用什么工具”和“工具如何安装到 sandbox”显式管理。对我们的系统，可扩展为：

- 求解器工具；
- 仿真器工具；
- 数据校验工具；
- 可视化工具；
- benchmark runner；
- report generator。

每类工具应有安装脚本、reset 脚本、timeout 和权限策略。

## Agent 动作执行限制

`agent/agents.py` 中 `_execute_action` 会：

- 检查工具是否允许该动作；
- 使用 `execution_timeout` 执行命令；
- 发生 `CommandTimeoutError` 时中断 session；
- 记录连续 timeout 次数；
- 超过 `max_consecutive_execution_timeouts` 后退出；
- 跟踪 total execution time，超过总预算后退出。

这比单个命令 timeout 更完整。工业工程 Agent 可能会连续跑很慢的求解命令，系统需要识别“Agent 已经卡住”而不是无限等待。

## 安全边界分析

SWE-agent 的隔离依赖 deployment，默认 Docker，但实际 Docker 启动细节在外部 `swerex` 中，不在仓库内直接可见。仓库内能确认的能力是：

- deployment 抽象；
- 持久 bash session；
- 命令 timeout；
- session interrupt；
- repo reset；
- 工具 allow/disallow；
- 文件 API；
- 总执行时间控制。

仓库内没有直接体现：

- Docker 内存/CPU/GPU 限制；
- 网络策略；
- seccomp/AppArmor/rootless；
- 只读数据挂载；
- 工业实验指标协议；
- 求解器许可证策略。

## 对自动工业工程科研机器的适用性

适用度：中高。

适合借鉴：

- deployment/runtime 抽象；
- 持久 bash session；
- 动作级 timeout；
- session interrupt；
- 连续 timeout 熔断；
- 总执行时间预算；
- repo copy/reset；
- 工具安装/reset 协议。

需要补充：

- 实验级 Docker/Kubernetes 资源策略；
- 工业工程 metrics/artifacts；
- 数据和许可证挂载；
- 求解器中断后的 incumbent solution 收集；
- 网络策略；
- 多候选方案并行调度。

推荐定位：SWE-agent 适合作为“代码编辑与调试型 Agent 沙箱接口”参考。我们可以借鉴它的 session 和工具协议，但实验运行和资源隔离应采用更接近 AutoResearchClaw/RD-Agent 的设计。

