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

## 更新

`orbit update` 更新 CLI 和本 skill；`orbit skill status|install|update` 单独管理 skill。只在用户要求时更新 CLI；`modified` 状态的 skill 只有用户同意才加 `--force`。
