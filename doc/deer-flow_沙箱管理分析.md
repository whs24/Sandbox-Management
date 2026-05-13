# deer-flow 沙箱管理分析

## 结论

deer-flow 的沙箱管理优势不在单次实验执行，而在 Agent 框架层：它抽象了 `Sandbox`、`SandboxProvider`、middleware 和工具调用，使 Agent 可以在同一套接口下使用本地沙箱、Docker all-in-one sandbox 或远端 provisioner。它尤其适合长会话、多轮工具调用、多 Agent 流程。

对“自动工业工程问题求解科研机器”来说，deer-flow 更适合作为控制面和会话工作区管理参考，而不是单独作为安全执行层。它的 LocalSandbox 明确不是安全边界；AioSandboxProvider 更接近生产可用，但还需要补足资源限额、网络策略、工业求解器挂载和实验指标协议。

## 核心代码位置

- `repos/deer-flow/backend/packages/harness/deerflow/sandbox/sandbox.py`
- `repos/deer-flow/backend/packages/harness/deerflow/sandbox/sandbox_provider.py`
- `repos/deer-flow/backend/packages/harness/deerflow/sandbox/middleware.py`
- `repos/deer-flow/backend/packages/harness/deerflow/sandbox/tools.py`
- `repos/deer-flow/backend/packages/harness/deerflow/sandbox/local/local_sandbox.py`
- `repos/deer-flow/backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py`
- `repos/deer-flow/backend/packages/harness/deerflow/sandbox/security.py`
- `repos/deer-flow/backend/packages/harness/deerflow/agents/middlewares/sandbox_audit_middleware.py`
- `repos/deer-flow/backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox.py`
- `repos/deer-flow/backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py`
- `repos/deer-flow/backend/packages/harness/deerflow/community/aio_sandbox/local_backend.py`
- `repos/deer-flow/backend/packages/harness/deerflow/community/aio_sandbox/remote_backend.py`
- `repos/deer-flow/backend/docs/CONFIGURATION.md`

## 抽象层：Sandbox 和 SandboxProvider

`sandbox.py` 定义了 Agent 工具能使用的沙箱能力：

```python
class Sandbox(ABC):
    def execute_command(self, command: str) -> CommandResult: ...
    def read_file(self, path: str) -> str: ...
    def write_file(self, path: str, content: str) -> None: ...
    def list_dir(self, path: str) -> list[str]: ...
    def glob(self, pattern: str) -> list[str]: ...
    def grep(self, pattern: str, path: str) -> str: ...
    def update_file(self, path: str, old: str, new: str) -> None: ...
```

`sandbox_provider.py` 则定义沙箱生命周期：

```python
class SandboxProvider(ABC):
    def acquire(self, thread_id: str) -> str: ...
    def get(self, sandbox_id: str) -> Sandbox: ...
    def release(self, sandbox_id: str) -> None: ...
    def reset(self) -> None: ...
```

`get_sandbox_provider()` 会按配置 `config.sandbox.use` 动态加载 provider，并保持单例。这一层设计非常适合我们的系统：工业工程 Agent 可能有多个任务线程，每个线程需要独立工作区、独立数据挂载和独立执行后端。

## Middleware：把沙箱绑定到 Agent 生命周期

`sandbox/middleware.py` 提供 `SandboxMiddleware`。它在 Agent 执行前按 `thread_id` 获取沙箱，并把 `sandbox_id` 放进 runtime state；执行后释放。

代码行为大致是：

```python
sandbox_id = provider.acquire(thread_id)
runtime.state["sandbox"] = {"sandbox_id": sandbox_id}
...
provider.release(sandbox_id)
```

注意：注释中强调会话内可复用，但实际释放语义取决于 provider。LocalSandbox 是单例；AioSandboxProvider 的 release 不是立刻销毁，而是放入 warm pool。这个设计适合 Agent 多轮交互：工具调用之间可以保留文件系统和 shell 状态，但会话结束后仍可回收资源。

## 工具层：Agent 不直接碰宿主机

`sandbox/tools.py` 负责把 Agent 工具映射到沙箱方法。典型工具包括：

- `bash_tool`
- `ls`
- `glob`
- `grep`
- `read_file`
- `write_file`

`ensure_sandbox_initialized(runtime)` 会懒加载沙箱，如果 runtime state 里没有 `sandbox_id`，就根据 `thread_id` 调用 provider 获取。

对于 LocalSandbox，`bash_tool` 会额外检查：

- 是否允许 host bash；
- 命令路径是否合法；
- 虚拟路径是否能映射到本地路径；
- cwd 是否在允许映射内；
- 输出是否需要截断。

这层隔离的价值在于：Agent 只知道“我要读 `/mnt/acp-workspace/foo.py`”，不需要知道宿主机真实路径。

## LocalSandbox：路径映射，不是安全边界

`local/local_sandbox.py` 实现的是路径映射型沙箱。它把容器风格路径映射到本地路径：

```python
@dataclass
class PathMapping:
    container_path: str
    local_path: str
    read_only: bool = False
```

路径解析时使用 `relative_to(local_root)` 防止路径逃逸：

```python
resolved = (local_root / rel).resolve()
resolved.relative_to(local_root)
```

写文件时会检查最具体的映射是否只读：

```python
if self._is_read_only(container_path):
    raise OSError(errno.EROFS, "Read-only file system")
```

命令执行使用系统 shell：

```python
subprocess.run(
    [shell, "-c", command],
    cwd=resolved_cwd,
    timeout=600,
    capture_output=True,
)
```

`security.py` 明确指出 LocalSandbox 不是安全边界，默认禁止 host bash，除非配置：

```yaml
sandbox:
  allow_host_bash: true
```

这点对我们的系统非常重要：本地沙箱只能用于开发、演示、可信代码或只读分析，不能运行 Agent 生成的工业求解脚本。

## LocalSandboxProvider：受控挂载

`local/local_sandbox_provider.py` 将技能目录和用户配置的 mounts 映射进 LocalSandbox。它会过滤不合理挂载：

- host path 必须是绝对路径；
- container path 必须是绝对路径；
- 不能挂到保留路径下，如 `/mnt/skills`、`/mnt/acp-workspace`、`/mnt/user-data`；
- 支持只读挂载。

这对工业工程系统很有用：企业模板、求解器示例、标准数据集可以只读挂载，Agent 只能把结果写到任务工作区。

## AioSandbox：Docker/远端 all-in-one 运行环境

`community/aio_sandbox/aio_sandbox.py` 把沙箱实现委托给 `agent_sandbox.Sandbox` HTTP 客户端。它提供：

- shell 命令执行；
- 文件读写；
- list/glob/grep；
- base64 update_file；
- 命令串行化锁。

命令执行大致是：

```python
self._client.shell.exec_command(
    command=command,
    no_change_timeout=600,
)
```

因为 AIO 容器只有一个持久 session，deer-flow 用 threading lock 防止并发命令互相污染。如果出现特定 `ErrorObservation`，会尝试新 session 重试。

这种“容器内常驻服务 + HTTP API”的模式很适合工业工程 Agent：可以在容器中保留 Python 环境、求解器、临时文件和会话状态，Agent 通过受控 API 操作它。

## AioSandboxProvider：缓存、预热池和跨进程锁

`community/aio_sandbox/aio_sandbox_provider.py` 是 deer-flow 的生命周期管理重点。它支持：

- 按 `thread_id` 确定性生成 sandbox id；
- 内存缓存；
- warm pool；
- idle timeout；
- 文件锁，防止多进程重复创建同一个容器；
- orphan reconciliation；
- signal handler；
- local Docker backend 和 remote provisioner backend。

核心流程是：

```python
sandbox_id = deterministic_id(thread_id)
if sandbox_id in cache:
    return sandbox_id
if warm_pool:
    reuse_warm_sandbox()
else:
    backend.create(...)
```

释放时不是立即销毁，而是进入 warm pool；超时后 idle checker 再清理。对自动科研机器来说，这能显著减少反复创建工业求解环境的成本，尤其是镜像很大、依赖很重时。

## LocalContainerBackend：本地 Docker 启动

`community/aio_sandbox/local_backend.py` 负责启动本地 Docker 容器。主要参数包括：

```python
docker run \
  --security-opt seccomp=unconfined \
  --rm -d \
  -p 127.0.0.1:host_port:8080 \
  --name deer-flow-sandbox-... \
  -v host:container \
  image
```

它会自动发现运行中的容器，做健康检查，并在重启后重建 provider 状态。

这里的不足是：代码中没有体现强资源限额、严格网络策略、rootless、安全 profile 等能力。对于工业工程系统，Docker backend 应进一步加上：

- `--memory`
- `--cpus`
- `--pids-limit`
- `--network none` 或受控代理网络；
- `--read-only` 根文件系统；
- 指定只读数据挂载；
- seccomp/AppArmor 配置；
- 容器内非 root 用户。

## RemoteBackend：对接 provisioner/Kubernetes

`remote_backend.py` 通过 REST API 调用 provisioner：

- 创建 sandbox；
- 销毁 sandbox；
- 查询已有 sandbox；
- 健康检查。

这给我们的系统提供了一个方向：本地开发用 Docker，生产环境用 Kubernetes/Slurm/企业调度器。Agent 层只依赖 `SandboxProvider`，不关心背后是本机容器还是集群 pod。

## Bash 审计

`agents/middlewares/sandbox_audit_middleware.py` 会审计 bash 工具调用。它通过正则和 `shlex` 分类命令：

- 高风险：直接 block；
- 中风险：warn；
- 低风险：pass。

它还会拒绝：

- 空命令；
- 超长命令；
- 包含 null byte 的命令。

这种中间件适合我们的 Agent 工业求解系统。即使最终执行在容器中，也应该保留工具调用审计，用于复现 Agent 行为、定位危险操作和生成合规报告。

## 对自动工业工程科研机器的适用性

适用度：中高。

适合借鉴：

- `Sandbox` / `SandboxProvider` 双层抽象；
- 按 `thread_id` 管理工作区；
- middleware 自动注入沙箱；
- 工具层统一读写/命令接口；
- LocalSandbox 的路径映射和只读挂载；
- AioSandbox 的常驻容器 API；
- warm pool、idle timeout、orphan reconciliation；
- bash 审计中间件。

主要不足：

- LocalSandbox 不是安全边界；
- Aio Docker backend 缺少显式 CPU/内存/GPU/网络策略；
- 对实验指标、模型文件和工业工程结果没有标准协议；
- 对长时间求解、并行优化、许可证服务器没有专门支持。

推荐定位：把 deer-flow 作为 Agent 沙箱控制面的参考，把 AutoResearchClaw/RD-Agent 的执行限制和指标协议叠加进去。这样可以得到“多轮 Agent 会话 + 安全实验运行 + 可复现实验结果”的组合。

