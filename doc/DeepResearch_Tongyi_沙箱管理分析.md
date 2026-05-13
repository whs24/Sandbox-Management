# Tongyi DeepResearch 沙箱管理分析

## 结论

Tongyi DeepResearch 的沙箱逻辑主要体现在 Python Interpreter 工具上：仓库本身不创建本地 Docker 容器，而是通过 `SANDBOX_FUSION_ENDPOINT` 调用外部 SandboxFusion 服务执行 Python 代码。也就是说，它采用“外部代码解释器服务”模式。

这种模式适合小段 Python 计算、数据处理、可视化草稿或研究 Agent 中的临时分析。对我们的自动工业工程问题求解科研机器，它可以作为轻量 Python 解释器参考，但不能覆盖完整工业实验需求：文件系统、商用求解器、长时间优化、数据挂载、artifact 收集和资源调度都需要更重的 Docker/Kubernetes 沙箱。

## 核心代码位置

- `repos/DeepResearch/README.md`
- `repos/DeepResearch/inference/prompt.py`
- `repos/DeepResearch/inference/tool_python.py`

## README 中的沙箱配置

README 提到环境变量：

```dotenv
SANDBOX_FUSION_ENDPOINT=...
```

该变量指向 Python interpreter sandbox endpoint。DeepResearch 不在仓库内管理容器生命周期，而是假设外部服务已经提供了安全执行能力。

这种架构的责任划分很清楚：

- Agent 框架负责生成代码和调用工具；
- SandboxFusion 负责执行代码；
- 本仓库只做请求封装和结果整理。

## Prompt 层：定义 PythonInterpreter 工具

`inference/prompt.py` 中定义了 PythonInterpreter 工具，要求模型把代码放在 `<code>` 标签里。Agent 看到的是一个“能运行 Python 的工具”，而不是 shell。

这比直接给 shell 安全得多，因为权限边界更窄：Agent 不能自然地执行 `rm`、`curl`、`docker` 等 shell 命令，除非解释器服务允许 Python 侧间接调用。

## tool_python.py：调用外部 sandbox 服务

`inference/tool_python.py` 是执行入口。它从环境变量读取 endpoint：

```python
PYTHON_INTERPRETER_ENDPOINT = os.getenv("SANDBOX_FUSION_ENDPOINT")
```

然后通过 sandbox_fusion SDK 调用：

```python
run_code(
    RunCodeRequest(
        code=code,
        language="python",
        run_timeout=timeout,
    ),
    max_attempts=1,
    client_timeout=timeout,
    endpoint=endpoint,
)
```

执行结果会整理 stdout/stderr。如果 `execution_time >= timeout - 1`，工具会把它视为接近 timeout 并返回超时提示。

## 代码提取和超时

`PythonInterpreter._call()` 会从参数中解析代码，支持：

- structured params；
- triple backticks；
- `<code>` 标签；
- fallback 的代码提取。

默认 timeout 约为 20 秒。这个设计明显面向短代码片段，而不是长时间实验。

对工业工程系统来说，这种短 timeout 工具有价值。例如：

- 快速计算一个公式；
- 验证一个小规模 LP/MIP 建模片段；
- 清洗一小段表格；
- 生成示意图；
- 分析日志片段。

但它不适合运行小时级排程求解、仿真优化或大规模 benchmark。

## 安全边界分析

DeepResearch 仓库内的安全边界很薄，因为实际边界在 SandboxFusion 服务中。仓库本身提供：

- Python-only 工具接口；
- endpoint 配置；
- run timeout；
- stdout/stderr 整理。

仓库本身不提供：

- 容器镜像管理；
- 文件系统挂载策略；
- 网络策略；
- CPU/内存/GPU 限制配置；
- 多任务工作区；
- artifact 收集；
- Docker/Kubernetes 生命周期；
- 代码静态审计。

它的安全性取决于外部 SandboxFusion 的实现和部署策略。

## 对自动工业工程科研机器的适用性

适用度：中，作为轻量解释器；低，作为完整实验沙箱。

可以借鉴：

- 把代码执行封装为窄工具，而不是暴露 shell；
- 外部 sandbox service 形态；
- endpoint 多实例配置；
- 短 timeout；
- stdout/stderr 标准化返回。

不适合直接承担：

- 长时间工业优化实验；
- 大文件/多文件项目执行；
- 商业求解器调用；
- 数据集和许可证挂载；
- 多轮 Agent 工作区；
- 可复现实验 artifact 管理。

推荐定位：

- 作为“轻量 Python 计算工具”接入我们的 Agent；
- 用于方案草算、参数估计、小规模验证；
- 不用于生产级实验执行；
- 大实验仍交给 Docker/Kubernetes sandbox。

总结：Tongyi DeepResearch 的沙箱实现思路是服务化 Python interpreter，边界窄、调用简单。它可以补充我们的工具体系，但不能替代工业工程科研机器的主沙箱。

