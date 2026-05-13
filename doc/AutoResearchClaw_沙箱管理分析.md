# AutoResearchClaw 沙箱管理分析

## 结论

AutoResearchClaw 是这些仓库中沙箱管理最完整、最接近“自动工业工程问题求解科研机器”需求的实现之一。它不是只把实验代码丢进 Docker，而是把执行环境抽象成统一协议，并围绕本地、Docker、SSH 远端、Colab Drive、Agentic Docker 等模式做了执行入口校验、项目复制、网络策略、资源限制、依赖安装、指标解析和结果落盘。

如果我们的系统要让 Agent 自动生成工业工程模型、调用优化器、运行仿真实验、整理实验指标，AutoResearchClaw 可以作为主要参考：它的优点是控制面清楚、Docker 沙箱可配置、安全意识较强；需要补强的是工业软件许可证管理、商用求解器隔离、数据权限、CPU/GPU 调度和更严格的网络/文件系统策略。

## 核心代码位置

- `repos/AutoResearchClaw/config.researchclaw.example.yaml`
- `repos/AutoResearchClaw/researchclaw/experiment/factory.py`
- `repos/AutoResearchClaw/researchclaw/experiment/sandbox.py`
- `repos/AutoResearchClaw/researchclaw/experiment/docker_sandbox.py`
- `repos/AutoResearchClaw/researchclaw/experiment/ssh_sandbox.py`
- `repos/AutoResearchClaw/researchclaw/experiment/validator.py`
- `repos/AutoResearchClaw/researchclaw/experiment/agentic_sandbox.py`
- `repos/AutoResearchClaw/researchclaw/pipeline/stage_impls/_execution.py`

## 配置层：用 experiment.mode 选择执行后端

`config.researchclaw.example.yaml` 把实验执行模式显式放在 `experiment.mode` 下，支持 `sandbox`、`docker`、`ssh_remote` 等模式。配置里不仅指定 Python 路径，也包含 Docker 镜像、GPU、内存、共享内存、网络策略、是否自动安装依赖、远端 SSH 参数等。

典型配置形态如下：

```yaml
experiment:
  mode: "sandbox"
  sandbox:
    python_path: "python"
    gpu_required: false
    max_memory_mb: 16384
  docker:
    image: "researchclaw-runtime:latest"
    gpu_enabled: false
    memory_limit_mb: 16384
    network_policy: "setup_only"
    pip_pre_install: []
    auto_install_deps: true
    shm_size_mb: 2048
    keep_containers: false
  ssh_remote:
    host: ""
    user: ""
    port: 22
    use_docker: false
    docker_network_policy: "none"
```

这类配置对科研机器很关键，因为工业工程实验往往需要在不同层级运行：小型模型可以本地跑，大型 MILP/仿真模型需要 Docker 或远端 GPU/CPU 机器，企业内数据可能只能在指定服务器上执行。

## 工厂层：统一创建沙箱

`researchclaw/experiment/factory.py` 是沙箱入口。`create_sandbox(config, workdir)` 根据配置选择后端：

- `docker`：导入 `DockerSandbox`，先检查 Docker daemon，再检查镜像是否存在，失败时回退到本地 `ExperimentSandbox`。
- `ssh_remote`：导入 `SshRemoteSandbox`，检查 host 和 SSH 连通性。
- `colab_drive`：创建 Colab Drive 沙箱并生成 worker 模板。
- 默认 `sandbox`：返回本地 `ExperimentSandbox`。

简化后的逻辑是：

```python
if config.mode == "docker":
    from .docker_sandbox import DockerSandbox
    if not check_docker_available():
        return ExperimentSandbox(...)
    if not ensure_docker_image(config.docker.image):
        return ExperimentSandbox(...)
    return DockerSandbox(...)

if config.mode == "ssh_remote":
    from .ssh_sandbox import SshRemoteSandbox
    return SshRemoteSandbox(...)

return ExperimentSandbox(...)
```

这个工厂层把调用方和具体沙箱实现解耦。对我们的系统来说，可以复用这种模式，把后端扩展为：

- 本地可信开发环境；
- Docker 单机实验环境；
- Kubernetes/Slurm 计算集群；
- 企业内网求解器环境；
- 外部 Python 代码解释器服务。

## 本地沙箱：项目复制、入口校验、超时和指标解析

`researchclaw/experiment/sandbox.py` 定义了统一协议和本地执行实现。

核心数据结构是 `SandboxResult`：

```python
@dataclass
class SandboxResult:
    returncode: int
    stdout: str
    stderr: str
    elapsed_sec: float
    metrics: dict[str, float]
    timed_out: bool = False
```

沙箱协议包括两个主要方法：

```python
class SandboxProtocol(Protocol):
    def run(self, code: str, timeout_sec: int) -> SandboxResult: ...
    def run_project(self, project_dir, entry_point, timeout_sec, args=None, env_overrides=None) -> SandboxResult: ...
```

### 入口点安全校验

`validate_entry_point(entry_point)` 拒绝空路径、绝对路径和包含 `..` 的路径。`validate_entry_point_resolved(staging, entry_point)` 会在项目复制到临时目录后再次解析真实路径，防止符号链接逃逸。

```python
if path.is_absolute():
    raise ValueError("entry_point must be relative")
if ".." in path.parts:
    raise ValueError("entry_point must not contain '..'")
```

这对工业工程 Agent 很重要。Agent 生成的 `run.py`、`solve.py`、`experiment.py` 必须限制在任务工作区内，不能通过路径穿越访问宿主机数据、密钥或其他任务目录。

### 代码执行方式

本地 `ExperimentSandbox.run` 会把代码写入临时脚本，然后用受控命令运行：

```python
cmd = [self.python_path, "-u", str(script_path)]
subprocess.run(
    cmd,
    cwd=self.workdir,
    env={"PYTHONUNBUFFERED": "1", ...},
    timeout=timeout_sec,
)
```

`run_project` 会创建 `_project_{counter}` 临时目录，复制项目，注入 harness，执行指定入口，并解析 stdout 中的实验指标。

这不是强安全边界，但非常适合做“执行协议层”：即使后端换成 Docker 或集群，仍然可以沿用 `run_project -> SandboxResult -> metrics` 这条链路。

## Docker 沙箱：资源、网络、依赖和清理

`researchclaw/experiment/docker_sandbox.py` 是最值得借鉴的部分。它延续本地沙箱 API，但把执行放进容器。

### 三阶段执行

Docker 沙箱把项目执行拆成三阶段：

1. Phase 0：安装 `requirements.txt` 或自动检测出来的依赖。
2. Phase 1：运行 `setup.py`。
3. Phase 2：运行实验入口。

这样可以对网络权限做精细控制：依赖安装阶段允许网络，真正实验阶段关闭网络或限制网络。

### docker run 关键参数

`_build_run_command` 组装容器命令，核心能力包括：

```python
cmd = [
    "docker", "run",
    "--name", container_name,
    "--rm",
    "-v", f"{staging}:/workspace",
    "-w", "/workspace",
    f"--memory={memory_limit_mb}m",
    f"--shm-size={shm_size_mb}m",
]
```

再根据配置追加：

- `--network none`：完全断网；
- `--cap-add=NET_ADMIN`：配合 setup 阶段网络策略；
- `--user uid:gid`：POSIX 下避免 root 写文件；
- `--gpus all` 或 `--gpus device=...`；
- 数据集目录只读挂载；
- Hugging Face 缓存只读挂载；
- `HOME=/workspace/.home`；
- `TORCH_HOME=/workspace/.cache/torch`。

这种设计非常适合工业工程科研机器：实验代码可以访问工作区、数据集缓存和必要模型缓存，但默认不能随意读宿主机目录，也不能无限制占用内存。

### 网络策略

配置中的 `network_policy` 支持：

- `none`：全程断网；
- `setup_only`：仅依赖/setup 阶段允许网络；
- `pip_only`：只允许 pip 安装阶段网络；
- `full`：全程联网。

工业工程场景建议默认使用 `setup_only` 或 `none`。真实生产数据、商业求解器许可证、企业知识库不应暴露给 Agent 生成代码的自由联网过程。

### 超时清理

Docker 执行超时时，沙箱会杀掉并移除容器：

```python
except subprocess.TimeoutExpired:
    self._kill_container(container_name)
finally:
    if not keep_containers:
        self._remove_container(container_name)
```

这对自动系统很关键。工业仿真、启发式搜索、强化学习训练都可能挂死或无限循环，超时后必须释放容器、GPU 和文件锁。

## 远端 SSH 沙箱

`researchclaw/experiment/ssh_sandbox.py` 用 SSH/SCP 把项目上传到远端目录，再在远端裸机或远端 Docker 中运行。裸机模式会尝试使用 `unshare --net` 做网络隔离；Docker 模式则构造远端 `docker run`，包含：

- `-v remote_dir:/workspace`
- `-w /workspace`
- `-e HOME=/workspace/.home`
- `--memory=...`
- `--shm-size=...`
- `--network none`
- GPU 参数

这对我们很有启发：工业工程求解常常依赖专用服务器、许可证服务器或内网数据。SSH 后端可以作为“接入既有计算资源”的桥，但需要额外加上审计、租户隔离和作业队列。

## 代码级安全校验

`researchclaw/experiment/validator.py` 用 AST 对 Agent 生成代码做预执行检查，拦截危险调用、危险 builtin 和危险模块。

规则包括：

```python
DANGEROUS_CALLS = {
    "os.system",
    "os.popen",
    "subprocess.run",
    "subprocess.Popen",
    "shutil.rmtree",
}

DANGEROUS_BUILTINS = {"eval", "exec", "compile", "__import__"}

BANNED_MODULES = {
    "subprocess",
    "socket",
    "urllib",
    "requests",
    "ctypes",
}
```

这不是完整安全模型，因为 Python 代码绕过 AST 黑名单并不困难；但它可以显著减少误伤型危险操作。对我们的科研机器来说，应把它作为第一层“静态风险提示/拦截”，再叠加容器、只读挂载、网络策略和系统调用限制。

## AgenticSandbox：把编码 Agent 放进 Docker

`researchclaw/experiment/agentic_sandbox.py` 用 Docker 承载一个完整编码 Agent 工作区，而不只是运行一段实验脚本。它会启动容器：

```python
docker run -d \
  --name ... \
  -v workspace:/workspace \
  -w /workspace \
  --memory=... \
  --network none \
  --gpus all
```

然后通过 `docker exec` 调用 Claude 或 Codex：

```python
claude -p ... --allowedTools Bash(*) Read Write Edit Glob Grep
codex --quiet --approval-mode full-auto
```

这对应一种更强的模式：不只沙箱化“实验执行”，还沙箱化“Agent 编写、修改、运行实验”的全过程。对我们的自动工业工程机器很有价值，因为 Agent 往往要多轮写模型、跑求解器、修错误、画图、产出报告。

## Pipeline 集成

`researchclaw/pipeline/stage_impls/_execution.py` 把沙箱接入流水线执行阶段。它根据 `experiment.mode` 创建沙箱，运行项目或代码，写入 `run-1.json`、`results.json`，并根据返回值设置：

- `completed`
- `partial`
- `failed`

其中如果超时但已经捕获到指标，会记为 `partial`。这点非常适合科研自动化：很多实验并不是非黑即白，超时前产生的可用指标仍应进入候选方案评估。

## 对自动工业工程科研机器的适用性

适用度：高。

可直接借鉴的部分：

- 统一 `SandboxProtocol`；
- 本地/Docker/远端后端工厂；
- `run_project` 项目级执行；
- 入口路径校验和符号链接逃逸检查；
- Docker 网络策略；
- 内存、GPU、shm 配置；
- 依赖安装阶段和实验运行阶段分离；
- 指标解析和部分结果保存；
- Agentic Docker 工作区。

需要补强的部分：

- 工业优化器许可证挂载和许可证服务器访问策略；
- CPLEX/Gurobi/OR-Tools/仿真软件的镜像治理；
- 数据集、企业数据、客户数据的访问分级；
- Kubernetes/Slurm 作业调度；
- CPU 核数、进程数、磁盘配额、inode 配额；
- 更强的 seccomp/AppArmor/rootless Docker；
- 对求解日志、模型文件、解文件、甘特图、仿真轨迹的结构化采集；
- 静态代码校验从黑名单升级为策略引擎。

整体看，AutoResearchClaw 可以作为我们沙箱层的主要蓝本：它已经把“科研 Agent 生成实验并执行”的关键风险点纳入设计，只需要把科研领域从通用机器学习扩展到工业工程求解、仿真和排程优化。

