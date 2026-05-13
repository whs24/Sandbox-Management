# AI-Researcher 沙箱管理分析

## 结论

AI-Researcher 的沙箱管理主要是“持久 Docker 容器 + TCP 命令服务器”。它会为每个任务创建或复用一个容器，把宿主机工作区挂载进去，然后 Agent 通过本地 TCP 端口向容器内服务发送 shell 命令。

这种模式对交互式科研 Agent 很方便：命令输出可以流式返回，容器里可以保留 conda 环境和中间文件。但从安全角度看，它更像远程 shell 封装，而不是严格沙箱。它使用 root 用户、`--userns=host`、端口暴露、无限制 shell 命令，并缺少内存/CPU/网络限制。对我们的自动工业工程问题求解科研机器，它可以参考“持久环境和流式命令执行”，但不能直接作为安全沙箱使用。

## 核心代码位置

- `repos/AI-Researcher/.env.template`
- `repos/AI-Researcher/research_agent/inno/environment/docker_env.py`
- `repos/AI-Researcher/research_agent/inno/environment/tcp_server.py`
- `repos/AI-Researcher/research_agent/inno/tools/terminal_tools.py`
- `repos/AI-Researcher/run_infer_plan.py`
- `repos/AI-Researcher/run_infer_idea.py`

## 配置层

`.env.template` 中定义了 Docker 执行环境：

```dotenv
BASE_IMAGES=tjbtech1/airesearcher:v1
GPUS='"device=0"'
CONTAINER_NAME=...
WORKPLACE_NAME=...
PORT=...
PLATFORM=linux/amd64
```

这些配置说明 AI-Researcher 的默认执行单元是容器。每个实例通过 container name、workspace name 和端口区分。

在 `run_infer_plan.py` 和 `run_infer_idea.py` 中，容器名会拼接任务实例和模型名：

```python
container_name = args.container_name + "_" + instance_id + "_" + COMPLETION_MODEL.replace("/", "__")
env_config = DockerConfig(...)
code_env = DockerEnv(env_config)
code_env.init_container()
```

这有利于并行任务隔离，但只是命名隔离，不等价于安全隔离。

## DockerEnv：启动和复用容器

`research_agent/inno/environment/docker_env.py` 定义 `DockerConfig` 和 `DockerEnv`。

配置项包括：

- `container_name`
- `workplace_name`
- `communication_port`
- `test_pull_name`
- `task_name`
- `git_clone`
- `setup_package`
- `local_root`

初始化时，宿主机目录会映射到容器：

```python
self.local_workplace = local_root / workplace_name
self.docker_workplace = f"/{workplace_name}"
```

`init_container()` 的主要流程：

1. 检查同名容器是否存在。
2. 创建本地 workspace。
3. 可选解压 setup package。
4. 可选 clone `metachain`。
5. 如果容器已运行则复用。
6. 如果容器停止则启动。
7. 否则执行 `docker run` 创建容器。

核心启动命令包含：

```bash
docker run -d \
  --platform $PLATFORM \
  --userns=host \
  --gpus $GPUS \
  --name $CONTAINER_NAME \
  --user root \
  -v local_workplace:docker_workplace \
  -w docker_workplace \
  -p communication_port:8000 \
  --restart unless-stopped \
  $BASE_IMAGES
```

这里的设计偏向可用性：root 用户、host user namespace、自动重启、端口映射都让容器更容易工作，但也扩大了安全面。

## 容器就绪检查

`wait_for_container_ready()` 会做几类检查：

- `docker inspect` 判断 `.State.Running`；
- 检查端口映射；
- 通过 `run_command('ps aux')` 测试 TCP 服务；
- 检查 `tcp_server.py` 是否存在。

这说明 AI-Researcher 的命令执行并不直接用 `docker exec`，而是依赖容器内的命令服务器。

## TCP 命令服务器

`research_agent/inno/environment/tcp_server.py` 是沙箱执行的实际入口。它监听 `0.0.0.0`，收到命令后启动 shell：

```python
subprocess.Popen(
    f"/bin/bash -c 'source {conda_path}/etc/profile.d/conda.sh && "
    f"conda activate autogpt && cd /{workplace} && {command}'",
    shell=True,
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
)
```

输出会按 JSON 行流式发送：

```json
{"type": "chunk", "content": "..."}
{"type": "final", "status": "success", "result": "..."}
```

这个机制对 Agent 体验很好：可以边执行边观察输出，适合调试实验脚本、安装依赖、运行测试。但它的风险也明显：

- 命令直接拼接进 shell 字符串；
- 没有命令 allowlist；
- 没有路径限制；
- 没有进程/内存/网络限制；
- 服务监听 `0.0.0.0`，且通过宿主机端口暴露。

## Tool 层

`research_agent/inno/tools/terminal_tools.py` 中的工具把 Agent 命令转给 `env.run_command(command)`。典型能力包括：

- `execute_command`
- `run_python`

这种工具设计简单，但 Agent 基本获得了容器内完整 shell。对科研探索方便，对不可信代码则危险。

## 安全边界分析

AI-Researcher 的安全边界主要依赖 Docker 容器，但 Docker 使用方式偏宽松：

- `--user root`：容器内 root；
- `--userns=host`：弱化用户命名空间隔离；
- `--restart unless-stopped`：异常任务可能长期存在；
- `-p host:8000`：命令服务暴露到宿主机端口；
- 未设置 `--memory`、`--cpus`、`--pids-limit`；
- 未设置 `--network none` 或受控网络；
- 未设置只读数据挂载；
- 未设置 seccomp/AppArmor 策略；
- 命令服务无认证。

它更适合“受信任研究者在容器中运行自动化工具”，不适合直接承载不可信 Agent 生成代码。

## 对自动工业工程科研机器的适用性

适用度：中低。

可以借鉴：

- 每个任务一个持久容器；
- 容器内保留 conda 环境；
- TCP/JSON 流式命令输出；
- workspace 挂载；
- 容器就绪检查；
- 按任务和模型命名容器。

不建议直接复用：

- root 容器执行；
- host user namespace；
- 无限制 shell；
- 无网络策略；
- 无资源限制；
- 命令服务端口裸露；
- 没有实验指标采集协议。

如果要改造成可用的工业工程沙箱，建议：

- 改用 `docker exec` 或带认证的本地 Unix socket API；
- 容器内非 root 用户；
- 默认 `--network none`，必要时按域名/阶段放行；
- 增加 `--memory`、`--cpus`、`--pids-limit`、磁盘配额；
- 工作区 rw，数据集和许可证 ro；
- 命令层加入 allowlist 和审计；
- 输出结构升级为 `stdout/stderr/exit_code/metrics/artifacts`。

总体看，AI-Researcher 对“交互式容器环境”有参考价值，但不应作为我们安全沙箱的主方案。

