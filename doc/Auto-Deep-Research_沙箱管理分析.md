# Auto-Deep-Research 沙箱管理分析

## 结论

Auto-Deep-Research 的沙箱模式与 AI-Researcher 相似：它通过 Docker 启动一个持久容器，在容器内运行 TCP 命令服务器，Agent 通过端口发送 shell 命令并接收流式 JSON 输出。同时它还有 LocalEnv，可以直接在宿主机执行命令。

这种实现适合快速搭建深度研究 Agent 的浏览器、文件和命令执行环境，但安全性较弱。它没有强资源限制、网络策略、命令审计或路径隔离，并且会在初始化时从 GitHub 动态下载 `tcp_server.py`，带来供应链风险。对我们的自动工业工程问题求解科研机器，只建议借鉴持久容器和流式执行形式，不建议直接采用。

## 核心代码位置

- `repos/Auto-Deep-Research/autoagent/environment/docker_env.py`
- `repos/Auto-Deep-Research/autoagent/environment/tcp_server.py`
- `repos/Auto-Deep-Research/autoagent/environment/local_env.py`
- `repos/Auto-Deep-Research/autoagent/cli.py`
- `repos/Auto-Deep-Research/README.md`

## DockerEnv：容器加命令服务

`autoagent/environment/docker_env.py` 定义了 `DockerConfig` 和 `DockerEnv`。配置包括：

- `container_name`
- `workplace_name`
- `communication_port`
- `test_pull_name`
- `task_name`
- `conda_path`
- `git_clone`
- `setup_package`
- `local_root`

初始化时会创建本地 workspace，把它挂载到容器中。容器启动命令大致是：

```bash
docker run -d \
  --name $CONTAINER_NAME \
  --user root \
  -v local_workplace:docker_workplace \
  -w docker_workplace \
  -p communication_port:communication_port \
  $BASE_IMAGES \
  /bin/bash -c "python3 /workspace/tcp_server.py --workplace ... --conda_path ... --port ..."
```

和 AI-Researcher 不同的是，Auto-Deep-Research 会直接把 `tcp_server.py` 作为容器启动命令运行。

## 动态下载 tcp_server.py

`init_container()` 会检查本地 workspace 中是否有 `tcp_server.py`。如果没有，会从 GitHub 下载：

```python
requests.get(
    "https://raw.githubusercontent.com/tjb-tech/agent.midware/refs/heads/main/tcp_server.py"
)
```

这对研究原型很方便，但对工业系统是明显风险：

- 执行环境依赖远程 main 分支；
- 缺少 hash 校验；
- 缺少签名验证；
- 如果远端内容变化，实验不可复现；
- 如果网络被劫持或依赖源异常，沙箱入口可能被污染。

我们的系统应把命令服务作为镜像内固定文件，经过版本锁定、镜像扫描和签名发布。

## TCP 命令服务器

`autoagent/environment/tcp_server.py` 与 AI-Researcher 的模式一致。它监听端口，收到命令后执行：

```python
subprocess.Popen(
    f"/bin/bash -c 'source {conda_path}/etc/profile.d/conda.sh && "
    f"conda activate autogpt && cd /{workplace} && {command}'",
    shell=True,
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
)
```

输出以 JSON chunk/final 形式流式返回。

优点：

- Agent 可获得实时输出；
- 容器内环境可持久存在；
- 适合安装依赖、运行脚本、调试任务。

风险：

- 命令字符串直接拼 shell；
- 无认证；
- 无命令 allowlist；
- 无工作区路径限制；
- 无进程树清理；
- 无资源限制。

## LocalEnv：宿主机执行

`autoagent/environment/local_env.py` 直接在宿主机运行 shell 命令：

```python
subprocess.Popen(
    f"/bin/bash -c 'source ... && conda activate browser && cd {cwd} && {command}'",
    shell=True,
)
```

这不是沙箱，只能用于完全可信开发。对工业工程科研机器，不应让 Agent 生成代码进入 LocalEnv。

## CLI：端口分配和工作区

`autoagent/cli.py` 管理端口和 workspace。它使用 `filelock.FileLock(".port_lock")` 选择端口，并创建类似：

```text
workspace_meta_showcase/showcase_{container_name}
```

的本地目录。

这个细节值得借鉴：多 Agent/多任务并行时，端口和工作区需要集中分配，避免冲突。但生产系统最好使用 Docker 网络、Unix socket 或 provider API，而不是裸露 TCP 端口。

## README 中的后续计划

README 提到 “Code Sandboxes: Supporting additional environments like E2B” 属于 TODO。这说明当前沙箱只是基础容器实现，还没有成熟到 E2B/Firecracker/Kubernetes 级别。

## 安全边界分析

主要问题：

- root 容器；
- 无内存/CPU/GPU 限制；
- 无网络策略；
- 命令服务端口暴露；
- 命令服务动态下载；
- shell=True；
- 没有 Docker 清理策略；
- 没有实验指标和 artifact 协议；
- LocalEnv 直接执行宿主机命令。

## 对自动工业工程科研机器的适用性

适用度：低到中。

可以借鉴：

- 持久容器执行环境；
- TCP/JSON 流式输出；
- 工作区挂载；
- 端口锁；
- 和浏览器/研究工具组合的思路。

不建议复用：

- 动态下载命令服务器；
- root shell 服务；
- 无认证端口；
- LocalEnv 执行 Agent 命令；
- 缺少资源和网络隔离。

改造建议：

- 命令服务内置到镜像；
- 使用本地 socket 或受控 sidecar API；
- 容器非 root；
- 默认断网；
- 加入 CPU/内存/磁盘/pids 限制；
- 加入命令审计和路径策略；
- 输出统一为 `exit_code/stdout/stderr/metrics/artifacts`。

总结：Auto-Deep-Research 适合参考 Agent 工具环境组织方式，不适合直接承担工业级沙箱职责。

