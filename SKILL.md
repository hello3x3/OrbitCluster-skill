---
name: orbitcluster
description: 通过 orbit CLI 查询和申请 OrbitCluster 实验室集群资源、获取 SSH 连接、查看日志、保存个人镜像。用户要求用 orbit 或 agent 操作 OrbitCluster 时使用；不用于集群管理员配置。
---

# OrbitCluster（orbit CLI）

参数以 `orbit --help` 和 `orbit 子命令 --help` 为准。机器读取一律加 `--json`（成功在 stdout，错误在 stderr）。

## 前提

- `orbit --version` 不可用时（也检查 `~/.local/bin/orbit`），让用户按门户 `/guide/terminal` 的命令安装；不要从别处下载或克隆。
- 登录只由用户本人在终端执行：`orbit login --username 用户名`。不要索要密码，不读取 `auth.json`。退出码 3 = 需重新登录（凭证七天，不续期）。

## 常用

```bash
orbit instances list --json        # 我的实例；ssh_command 为可直接执行的连接命令（空表示未就绪）
orbit plans --json                 # 套餐 / 镜像 / 账户 / 节点 / 额度：先查，不要猜
orbit images list --json
orbit accounts --json
orbit instances create --name NAME --plan ID --image NAME --hours N --json
orbit instances wait ID --until ssh-ready --timeout 300 --json
orbit instances logs ID --lines 300 --json
orbit images save --instance ID --name NAME --json
```

`ID` 是门户实例 `id`，不是 Slurm `job_id`。任务名 3–10 个字符，同一用户下不可重复。

## 规则

- 创建、重启、保存镜像是真实集群操作；停机、删除、覆盖镜像需用户明确授权后加 `--yes`。
- 停机不会自动保存环境；要保留修改先保存镜像并等 `--until saved` 成功。
- 写操作超时不要重试，先用 `instances list` / `logs` / `images list` 确认结果。
- 退出码：0 成功，1 业务拒绝或超时，2 参数/配置，3 需登录，4 网络。

## 容器环境与存储

- 每个资源是独立容器；代码、训练和调试都在容器内进行。镜像决定环境，环境修改需保存为个人镜像才会保留。
- 日常使用普通用户。安装系统包或 CUDA toolkit 时，在容器内执行 `unshare -Ur --map-root-user bash`，再运行 `apt-get update`、`apt-get install ...`；`whoami` 显示的 root 仅属于容器用户命名空间，不是宿主机 root，`exit` 返回普通用户。
- enroot 展开的 rootfs 归用户所有，因此普通用户的 `apt-get update` 可能成功，但安装包所用的 `dpkg` 仍需上述 root shell。不要依赖传统 `su` / `sudo` 的 setuid 提权，也不要索要 root 密码；无需另装 sudo。
- 基础镜像已预装 `/opt/miniconda3`，不要在 home 重装。激活与创建环境示例：

```bash
source /opt/miniconda3/etc/profile.d/conda.sh
conda create -p /opt/miniconda3/envs/myenv python=3.11
conda activate /opt/miniconda3/envs/myenv
```

- Conda 环境留在容器内的 `/opt/miniconda3` 下，并通过保存个人镜像保留。`/share/home/<用户名>` 只放代码和小型配置；大型数据集、Conda 环境、checkpoint 和大量结果不要放 home，以免耗尽 NFS 配额、影响节点性能。

## 更新

`orbit update` 更新 CLI 和本 skill；`orbit skill status|install|update` 单独管理 skill。只在用户要求时更新 CLI；`modified` 状态的 skill 只有用户同意才加 `--force`。
