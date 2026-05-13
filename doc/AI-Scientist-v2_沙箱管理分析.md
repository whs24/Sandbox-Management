# AI-Scientist v2 沙箱管理分析

## 结论

AI-Scientist v2 比 v1 多了更明确的代码解释器和进程级监督机制：它会把 Agent 生成的代码写入 `runfile.py`，在子进程里 `exec(compile(...))` 执行，并通过队列收集 stdout/stderr、状态和结果；超时后先中断，再终止或 kill 子进程。

但它仍不是完整沙箱。它没有容器边界、网络隔离、文件系统权限隔离或资源配额。README 也明确建议使用受控 Docker sandbox。对我们的自动工业工程科研机器，AI-Scientist v2 的进程监督机制值得借鉴，但必须放在 Docker/Kubernetes 沙箱内部使用。

## 核心代码位置

- `repos/AI-Scientist-v2/README.md`
- `repos/AI-Scientist-v2/bfts_config.yaml`
- `repos/AI-Scientist-v2/ai_scientist/treesearch/interpreter.py`
- `repos/AI-Scientist-v2/launch_scientist_bfts.py`
- `repos/AI-Scientist-v2/parallel_agent.py`

## README 中的安全定位

README 明确提示：LLM 写出的代码可能安装危险包、访问不受控网络、启动意外进程，因此应在受控沙箱中运行，例如 Docker。

这和 v1 类似：项目知道需要沙箱，但沙箱不是框架内置的一等模块。

## 配置中的执行参数

`bfts_config.yaml` 中有执行相关参数：

```yaml
exec:
  timeout: 3600
  agent_file_name: runfile.py
  format_tb_ipython: false
  copy_data: true
```

它关注的是代码解释器的运行时长、执行文件名和数据复制，而不是 OS 级隔离。

## Interpreter：子进程执行 Agent 代码

`ai_scientist/treesearch/interpreter.py` 是 v2 的核心执行器。它使用 multiprocessing 创建子进程，并在子进程内执行代码。

子进程初始化会：

- 设置环境变量；
- 切换工作目录；
- 修改 `sys.path`；
- 重定向 stdout/stderr 到队列；
- 准备全局作用域。

执行逻辑大致是：

```python
with open(agent_file_name, "w") as f:
    f.write(code)

exec(
    compile(code, agent_file_name, "exec"),
    global_scope,
)
```

父进程通过队列接收事件，并构造 `ExecutionResult`。这比 v1 的 `subprocess.run` 更适合交互式树搜索，因为可以捕获中间输出、维护解释器状态并控制重置。

## 超时和清理

`Interpreter.run()` 会启动子进程，等待 ready，然后持续轮询事件队列。如果超过 timeout，会先尝试发送 `SIGINT`；如果超过 `timeout + 60`，则执行 cleanup/kill。

清理过程包括：

```python
process.terminate()
process.join(timeout=2)
if process.is_alive():
    process.kill()
```

这对我们的系统有参考价值：即使在容器内部，也应区分“温和中断”和“强制杀死”。很多工业求解器收到中断信号时可以输出 incumbent solution 或日志，如果直接 kill 会损失可用结果。

## launch_scientist_bfts.py 的进程清理风险

`launch_scientist_bfts.py` 在运行结束后会清理当前进程的子进程，还会遍历系统进程，匹配命令行中包含 `python`、`torch`、`mp`、`bfts`、`experiment` 的进程并尝试终止。

这在单用户实验机上可能有用，但在共享工业计算环境里风险很大：可能误杀其他任务、其他用户或系统服务。这个逻辑只有在容器内部才相对可接受，因为容器内进程集合属于当前任务。

## parallel_agent.py：并行执行和 timeout

`parallel_agent.py` 使用 `ProcessPoolExecutor` 并给 future 设置 timeout。超时后会 shutdown executor 并清理进程。

这说明 v2 的并发控制在 Python 进程层完成，而不是容器/集群调度层。对工业工程任务，建议把这个并行层放进更外层的资源调度中：

- 每个大实验一个容器；
- 容器内可以开多个 Python worker；
- 容器外由调度器限制总 CPU/内存/GPU。

## 安全边界分析

AI-Scientist v2 的安全能力主要是：

- 子进程隔离；
- stdout/stderr 捕获；
- timeout；
- SIGINT/terminate/kill；
- 执行文件落盘；
- 运行状态队列。

它不提供：

- 容器隔离；
- 网络隔离；
- 内存/CPU 限制；
- 只读数据挂载；
- 路径逃逸检查；
- AST/命令审计；
- 工具权限控制；
- 多租户安全。

因此，它适合作为“容器内解释器”，不适合作为“沙箱边界”。

## 对自动工业工程科研机器的适用性

适用度：中低。

可以借鉴：

- 将 Agent 代码写入固定 `runfile.py`；
- 子进程执行；
- 事件队列捕获 stdout/stderr；
- timeout 后先中断再强杀；
- `ExecutionResult` 结构化返回；
- 进程池并行。

必须改造：

- 所有解释器进程放进 Docker/Kubernetes sandbox；
- 禁止宿主机级全局进程扫描/kill；
- 增加容器资源限制；
- 增加文件系统和网络策略；
- 增加工业实验 artifact 协议；
- 对求解器中断后的可行解进行收集。

推荐定位：AI-Scientist v2 可作为“容器内代码解释器/树搜索执行器”的参考，而不应作为系统级沙箱。真正的安全边界应由外层 Docker/Kubernetes/远端执行器提供。

