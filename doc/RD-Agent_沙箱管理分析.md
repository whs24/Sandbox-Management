# RD-Agent 沙箱管理分析

## 结论

RD-Agent 的沙箱管理集中在 `rdagent/utils/env.py`，它提供了成熟的环境执行层：本地环境、Conda 环境和 Docker 环境统一成 `Env.run()`，Docker 环境支持镜像准备、卷挂载、超时、缓存、GPU、内存、CPU、共享内存、日志保存和容器清理。

它非常适合作为“实验执行器”参考，尤其适合我们自动工业工程问题求解科研机器中的批量实验、基准评估、模型训练、优化求解和结果复现。但 RD-Agent 本身不是强安全沙箱框架，它缺少 Agent 代码静态审查、命令审计、细粒度网络策略、工作区路径逃逸检查等能力。

## 核心代码位置

- `repos/RD-Agent/rdagent/utils/env.py`
- `repos/RD-Agent/rdagent/utils/workflow/loop.py`
- `repos/RD-Agent/rdagent/scenarios/**`

## Env 抽象：统一执行入口

`rdagent/utils/env.py` 定义了 `EnvConf`、`EnvResult` 和 `Env`。

`EnvResult` 保存执行结果：

```python
@dataclass
class EnvResult:
    stdout: str
    full_stdout: str
    exit_code: int
    running_time: float
```

`Env.run()` 做了几件关键事情：

- 合并环境变量；
- 应用超时包装；
- 支持缓存；
- 支持重试；
- 屏蔽本地真实路径；
- 返回结构化结果。

超时包装形式接近：

```bash
timeout --kill-after=10 {running_timeout_period} {entry}
```

这对工业工程求解非常实用。MILP、仿真、元启发式算法都可能运行很久；执行器必须能统一设置 wall-clock timeout，并在超时后杀掉进程。

## DockerConf：资源和容器配置

`DockerConf` 是 RD-Agent Docker 执行的主要配置对象，包含：

- `image`
- `mount_path`
- `default_entry`
- `extra_volumes`
- `extra_volumes_mode`，默认 `ro`
- `network`，默认 `bridge`
- `shm_size`
- `enable_gpu`
- `mem_limit`
- `cpu_count`
- `running_timeout_period`
- `save_logs`
- `terminal_tail_lines`

这些字段与工业工程科研机器高度相关：

- `mem_limit` 控制求解器/仿真的内存爆炸；
- `cpu_count` 控制并行求解器线程；
- `enable_gpu` 支持深度学习或仿真加速；
- `extra_volumes` 可以挂载数据集、benchmark、许可证文件；
- `network` 控制联网行为；
- `save_logs` 支持可复现审计。

## 镜像准备

`DockerEnv.prepare()` 支持两种方式：

1. 根据 Dockerfile 构建镜像；
2. 从 registry 拉取镜像。

执行器会在运行前准备好镜像。对我们的系统而言，可以把不同工业工程任务对应到不同镜像：

- 排程优化镜像；
- 车辆路径/物流优化镜像；
- 仿真建模镜像；
- Gurobi/CPLEX 商用求解器镜像；
- OR-Tools/open-source 基础镜像；
- 机器学习预测 + 优化联合镜像。

## DockerEnv._run：真正的容器执行

`DockerEnv._run()` 使用 Docker Python SDK 创建并运行容器。核心形态是：

```python
container = client.containers.run(
    image=self.conf.image,
    command=entry,
    volumes=volumes,
    environment=env,
    detach=True,
    working_dir=self.conf.mount_path,
    network=self.conf.network,
    shm_size=self.conf.shm_size,
    mem_limit=self.conf.mem_limit,
    cpu_count=self.conf.cpu_count,
    **gpu_kwargs,
)
```

执行时还会设置常见环境变量：

```python
PYTHONWARNINGS=ignore
TF_CPP_MIN_LOG_LEVEL=2
PYTHONUNBUFFERED=1
TOKENIZERS_PARALLELISM=false
```

日志通过容器 log stream 获取，容器退出后读取 exit code，最后清理容器。

## GPU 支持

`DockerEnv._gpu_kwargs()` 根据 `CUDA_VISIBLE_DEVICES` 设置 Docker device request：

```python
docker.types.DeviceRequest(
    device_ids=device_ids,
    capabilities=[["gpu"]],
)
```

如果没有指定设备，则请求全部 GPU。它还会用 `nvidia-smi` 测试 GPU 是否可用；如果 Docker API 报错，则退回无 GPU。

对工业工程系统而言，这个逻辑可以扩展为：

- GPU 资源调度；
- 每任务 GPU allowlist；
- GPU 显存监控；
- 按任务类型选择 CPU-only 或 GPU 镜像。

## 卷挂载和缓存

RD-Agent 默认把本地工作目录挂载到容器 `mount_path`，并支持 `extra_volumes`。额外挂载默认只读，这一点很好：

```python
extra_volumes_mode = "ro"
```

它还支持缓存执行结果。`EnvConf` 中有 cache hash 逻辑，会根据 workspace 中的 `.py`、`.csv`、`.yaml` 等文件计算缓存键。

这对工业工程实验很有用：相同模型、相同输入数据、相同参数的实验可以跳过重复运行；但要注意随机种子、求解器版本、许可证参数也应进入缓存键。

## 容器清理

`cleanup_container()` 会先 stop，再 remove 容器：

```python
container.stop()
container.remove()
```

如果清理失败会记录 warning。对于自动系统，这能避免大量短任务留下停止容器或占用 GPU/内存。

## Workflow 超时和子进程清理

`rdagent/utils/workflow/loop.py` 里还有工作流级别的 timeout 管理，并提供 `kill_subprocesses()` 清理子进程。它面向整个 workflow step，而不仅是单个 Docker 命令。

对我们的系统，这对应两层超时：

- 单次实验命令 timeout；
- 一个研究阶段或一个 Agent step 的总 timeout。

## 安全边界分析

RD-Agent 的强项是执行稳定性，不是安全策略。主要缺口：

- 没有 AST/命令安全审查；
- 默认 network 是 `bridge`，不是断网；
- Docker 容器是否 root 取决于镜像；
- 没有 seccomp/AppArmor/rootless 明确策略；
- 没有路径入口校验；
- 没有工具调用审计；
- 没有阶段化网络策略，比如仅 setup 阶段联网。

在工业工程科研机器里，应把 RD-Agent 的 DockerEnv 视为“资源受控执行器”，再叠加 AutoResearchClaw/deer-flow 的沙箱工厂、路径策略和 Agent 工具审计。

## 对自动工业工程科研机器的适用性

适用度：高。

适合直接借鉴：

- `Env.run()` 统一执行入口；
- `DockerConf` 参数化资源控制；
- Docker Python SDK 启动容器；
- GPU 检测和 device request；
- 内存、CPU、shm 控制；
- 额外挂载默认只读；
- 执行超时；
- 容器日志流；
- 结果缓存；
- 容器清理。

需要补强：

- 默认断网或按阶段联网；
- 非 root 容器；
- 求解器许可证挂载策略；
- 工业数据读写权限；
- 任务级磁盘配额；
- Agent 代码静态和动态审计；
- 统一 metrics/artifacts 协议。

推荐定位：RD-Agent 可作为我们工业工程实验执行层的骨架。它负责“怎么可靠地跑”，AutoResearchClaw/deer-flow 负责“在哪个安全边界里跑、由谁生命周期管理、如何和 Agent 工具系统对接”。

