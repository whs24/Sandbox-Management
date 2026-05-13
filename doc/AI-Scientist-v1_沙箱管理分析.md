# AI-Scientist v1 沙箱管理分析

## 结论

AI-Scientist v1 本身没有完整的沙箱管理器。它的核心实验执行逻辑是直接在当前工作目录中用 `subprocess.run(["python", ...])` 运行 LLM 修改后的实验脚本；安全方面主要依赖 README 中的人工提醒和 `experimental/Dockerfile` 提供的外部容器化运行建议。

因此，AI-Scientist v1 对我们的自动工业工程问题求解科研机器更像“科研 Agent 流水线参考”，不是沙箱实现参考。它可以启发自动提出 idea、修改实验、运行实验、生成图表和论文的流程，但必须包在 AutoResearchClaw、RD-Agent 或自研 Docker/Kubernetes 沙箱中运行。

## 核心代码位置

- `repos/AI-Scientist/README.md`
- `repos/AI-Scientist/experimental/Dockerfile`
- `repos/AI-Scientist/ai_scientist/perform_experiments.py`
- `repos/AI-Scientist/launch_scientist.py`

## README 中的安全定位

README 明确警告：AI-Scientist 会执行 LLM 写出的代码，因此应容器化运行，并限制 Web 访问。它给出 `experimental/Dockerfile` 和 `docker run` 示例，把 templates 挂载到容器中。

这说明项目作者意识到了风险，但实现上没有把 Docker 沙箱做成框架内的一等能力。容器化更像部署建议：

```bash
docker run \
  -e OPENAI_API_KEY=... \
  -v $(pwd)/templates:/app/AI-Scientist/templates \
  ...
```

对我们的系统来说，这种“用户自己把整个系统放进容器”的方式不够。我们需要的是：每个 Agent 任务、每次实验、每个候选方案都由系统自动创建、限制、审计和清理沙箱。

## 实验执行：直接 subprocess

`ai_scientist/perform_experiments.py` 是执行实验和绘图的核心文件。`run_experiment()` 会复制实验脚本，然后直接运行 Python：

```python
subprocess.run(
    ["python", "experiment.py", f"--out_dir=run_{run_num}"],
    cwd=cwd,
    stderr=subprocess.PIPE,
    text=True,
    timeout=timeout,
)
```

默认 timeout 为 7200 秒。失败时会删除本次 run 目录，并把错误反馈给代码修改 Agent。超时时也删除 run 目录，并提示需要优化运行时间。

`run_plotting()` 逻辑类似：

```python
subprocess.run(
    ["python", "plot.py"],
    cwd=cwd,
    stderr=subprocess.PIPE,
    text=True,
    timeout=600,
)
```

这是一种“流程控制级”的超时，不是安全隔离。代码仍在当前 Python 环境和当前用户权限下运行。

## 并行和 GPU 分配

`launch_scientist.py` 支持 multiprocessing 并行运行多个 idea，并通过 `CUDA_VISIBLE_DEVICES` 分配 GPU。它的关注点是吞吐量，而不是隔离。

对工业工程机器来说，GPU 分配只是资源调度的一部分。我们还需要：

- CPU 核数限制；
- 内存限制；
- 求解器线程数限制；
- 磁盘配额；
- 任务级超时；
- 容器级清理；
- 宿主机文件隔离。

AI-Scientist v1 没有覆盖这些。

## 安全边界分析

AI-Scientist v1 的执行风险主要包括：

- LLM 可修改并执行任意 Python 文件；
- 运行环境是当前用户环境；
- 没有 Docker API 创建/销毁容器；
- 没有网络策略；
- 没有资源限制；
- 没有路径逃逸检查；
- 没有 AST/命令审计；
- 没有结构化 artifact 采集；
- 依赖外部人工容器化。

README 提醒用户容器化是必要的，但项目代码没有强制执行。

## 对自动工业工程科研机器的适用性

适用度：低，除非外包给外部沙箱。

可以借鉴：

- idea -> experiment -> plotting -> writeup 的科研循环；
- 实验失败后把 stderr 反馈给代码 Agent；
- 单实验 timeout；
- 多 idea 并行；
- GPU 可见性控制。

不适合直接复用：

- 当前环境直接执行实验；
- 仅靠 README 提醒容器化；
- 没有资源和网络隔离；
- 没有路径和文件权限控制；
- 没有执行审计。

推荐改造方式：

1. 保留 AI-Scientist 的科研工作流。
2. 把 `run_experiment()` 和 `run_plotting()` 替换为统一 `Sandbox.run_project()`。
3. 每个 idea 单独创建工作区和 Docker/Kubernetes sandbox。
4. 工作区 rw，数据/模板/许可证 ro。
5. 实验结果按 `metrics.json`、`artifacts/`、`logs/` 标准输出。

总结：AI-Scientist v1 不是沙箱方案，但它很好地说明了为什么科研 Agent 必须有沙箱。对工业工程自动化而言，它只能作为上层科研流程参考。

