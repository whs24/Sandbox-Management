> [项目地址](https://github.com/karpathy/autoresearch)

# 大前提：这个项目<ins><span style="color: rgb(0, 0, 0);">没有</span></ins>传统意义上的沙箱管理！

> 该项目**没有**传统意义上的容器化沙箱（无 Docker、Firejail、gVisor、K8s、chroot、Linux namespace 隔离等）。其“沙箱”是指训练脚本进程本身 ，即一个普通的 Python 子进程，通过程序逻辑和代理行为规范来约束其边界。
> 
> 如果按照传统“沙箱”的各个维度进行分析，该项目主要有以下特点：

## 1.生命周期

### 创建/初始化

```bash
uv run train.py > run.log 2>&1
```

`uv run` 确保依赖的版本保持正确  
所有的 stdout/stderr 都重定向至 `run.log`，与代理的上下文窗口隔离

### 执行

训练主循环：

- `while True` 循环，每步会执行 `grad_accum_steps` 次前向/反向传播
- 加入时间预算检查

### 日志采集

所有的标准输出都流向 `run.log`  
训练中，每步都在同一行实时输出进度，不污染最终日志

### 产物导出/销毁

- **无模型检查点保存：**训练结束后，模型的权重直接被丢弃
- **产物仅为指标：** `val_bpb`、`peak_vram_mb` 等打印到 stdout，并流向 `run.log`，代理读取之后写入 `results.tsv`
- **进程结束即为销毁**

## 2.隔离与安全

### 文件系统隔离

- **唯一可写文件：**代理在至凌晨就被明确限制只能修改 `train.py`，`prepare.py` 为只读文件，评估函数 `evaluate_bpb` 不可改动
- **数据路径固定：**所有数据/分词器读写在 `~/.cache/autoresearch/`，训练脚本只读不写
- **日志文件：** `run.log` 和 `results.tsv` 在工作目录，`results.tsv` 显式不提交 git
- **无 OS 级文件系统隔离**（无 chroot/namespace/overlay filesystem）

### 依赖/包隔离

- 使用 `uv` 进行虚拟环境管理
- 代理**不允许**安装新包

### 网络隔离

- 该项目**无网络隔离**，训练时不需要网络访问，但没有 OS 级别的网络命名空间限制。
- Flash Attention 内核通过 `kernels` 库按需下载，首次运行需网络。

### 权限最小化

- 通过 `uv run` 在标准用户权限下运行，无 root 提权
- 代理行为约束依赖“指令”而非 OS 强制机制

## 3.沙箱后端对比

该项目**只有一种沙箱后端：本地 Python 子进程**（由 `uv run` 启动）。没有 Docker、Firejail、gVisor、K8s 等多后端设计