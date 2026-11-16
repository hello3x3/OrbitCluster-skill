---
name: orbitcluster
description: 安装和使用 OrbitCluster 用户终端，查看集群 GPU 余量与自己的申请排队状态、申请资源、连接容器、查看日志和管理个人镜像。适用于用户要求通过 orbit CLI 或 agent 操作 OrbitCluster 的任务，不用于集群管理员配置。
---

# OrbitCluster 用户终端

OrbitCluster 用户命令是 `orbit`，供用户与 agent 调用；没有 TUI，也不提供管理员能力。

## 安装与认证

先检查 `orbit --version`，命令不在 PATH 时检查 `~/.local/bin/orbit`；原生 Windows 默认 `%LOCALAPPDATA%\OrbitCluster\bin\orbit.exe`。客户端独立运行，无需 Python、pip、Git 或 Go。

安装脚本和对应平台压缩包仅从用户提供的门户地址下载。macOS/Linux/WSL 用 `/terminal/install.sh`（Bash），原生 Windows 用 `/terminal/install.ps1`（PowerShell）。门户 `/guide/terminal` 提供命令生成器。不要猜地址/HTTPS 端口，不要从别处 clone 仓库或临时 pip 安装替代包。

macOS/Linux 示例，替换为用户提供的真实地址：

```bash
(
  installer="$(mktemp)" || exit 1
  trap 'rm -f "$installer"' EXIT
  curl --fail --show-error --progress-bar 'http://用户提供的地址/terminal/install.sh' -o "$installer" &&
  bash "$installer" --server 'http://用户提供的地址'
)
```

HTTP 默认支持，适用于用户的实验室内网；HTTP 会明文传输凭证。HTTPS 正常校验证书，内部 CA 使用 `--ca-bundle '/本机证书路径'`，不跳过验证。安装器显示进度并校验 SHA256；下载缺包或失败时报告问题，不回退到 Git/Python 安装。服务器需要部署现有终端 API 和客户端发布包，skill 不代替管理员改配置。

验证码选项需要 CLI 0.5.0+ 和门户终端接口支持，具体先看 `orbit login --help`；主项目 `9172b9d` 已提供该接口，旧部署仍可能需要维护者升级，不回退到密码登录。

登录由用户本人在真实终端执行 `orbit login --username 用户名`：请求发送飞书验证码，再隐藏输入四位验证码。受控分步流程先 `orbit login --username 用户名 --send-code`，收到验证码后用 `--code-stdin` 验证已有验证码；不要把验证码放进命令行或聊天。不要再要求密码；不读取、复制或输出 `auth.json`，不使用浏览器 Cookie。

通过 `orbit whoami --json` 检查身份；已有有效终端凭证继续使用，按服务器返回时间到期，默认七天且不自动续期。验证码登录需要门户提供专用终端接口；若 CLI 报 `otp_not_supported`，让主项目维护者接入接口，不回退到密码、不修改用户密码，也不要声称仅有网页验证码就已完成 CLI 登录。

## 容器环境与存储

- 每个资源是独立的 enroot/pyxis 容器；代码、训练、调试和软件环境都在容器内运行。平台不提供容器内 Docker/Podman 或 Docker socket；需要外部 Docker 镜像时，由管理员用 `enroot import` 转成 `.sqsh`。
- **平台基础镜像已预装 Miniconda，位置是 `/opt/miniconda3`。不要在 home 或挂载存储里重装 Conda。** 新版 SSH 交互登录默认激活 base；旧镜像和非交互命令显式初始化：

```bash
source /opt/miniconda3/etc/profile.d/conda.sh
conda activate base
conda create -p /opt/miniconda3/envs/myenv python=3.11
conda activate /opt/miniconda3/envs/myenv
```

- **Conda 环境建在容器 rootfs 的 `/opt/miniconda3/envs/<环境名>` 下。** 不要放在 `/share/home/<用户名>`、`/data`、`/mnt/nas` 等挂载存储路径；挂载的数据不包含在保存的个人镜像里。基础镜像的包缓存位于 `/tmp/conda-pkgs`；保存前可用 `conda clean -afy` 清理缓存，减少镜像体积。
- 安装完依赖后，用 `orbit images save --instance ID --name NAME --json` 保存个人镜像，并等 `orbit instances wait ID --until saved --timeout 1800 --json` 成功。之后选择该个人镜像创建资源即可复用环境。镜像保存不包含运行中的进程、内存或挂载数据。
- 日常使用普通用户。安装系统包/CUDA toolkit 时，在容器内执行 `unshare -Ur --map-root-user bash`，再运行 `apt-get update` / `apt-get install ...`；`whoami` 显示的 root 仅限用户命名空间，不是宿主机 root，`exit` 返回普通用户。新基础镜像的 `su` 函数也是该命名空间的便捷包装；不要索取 root 密码或依赖传统 setuid `su` / `sudo`。
- 普通用户的 `apt-get update` 可能成功，因为展开的 rootfs 归用户所有；安装包所用的 `dpkg` 仍需上述 root shell。
- `/share/home/<用户名>` 只放代码和小型配置，计入 NFS 配额；大型数据集、checkpoint 和大量结果使用站点提供的存储，数据优先从只读 NAS `/mnt/nas/...` 直接读取，不要整份复制到 home，也不要塞进容器 rootfs。
- 单个个人镜像默认上限 **30 GiB**（管理员可调整）；**到期自动保存默认关闭，到期直接停机**。需要保留环境时提前手动保存，不要指望退出 CLI、停机或重启会保留未保存的环境修改。

## 查询与申请

优先使用 `--json`。成功写 stdout，错误写 stderr。先查询实际可用项，不猜套餐、镜像或账户：

```bash
orbit whoami --json
orbit status --json
orbit queue --json
orbit plans --json
orbit images list --json
orbit accounts --json
orbit nodes --json
orbit quota --json
orbit instances list --json
```

申请使用用户需求中的名称、套餐、镜像、账户和时长，具体参数以 `orbit instances create --help` 为准。任务名 3–20 个字符，同一用户下不区分大小写且不能重复；停止任务不会释放名称。创建、重启和保存镜像会产生真实集群操作，应在用户请求授权的范围内执行。

后续操作使用创建结果中的门户实例 **id**，不是 Slurm `job_id`：

```bash
orbit instances wait ID --until ssh-ready --timeout 300 --json
orbit instances connection ID --json
orbit instances logs ID --lines 300 --json
orbit images save --instance ID --name 保存名称 --json
orbit instances wait ID --until saved --timeout 1800 --json
```

把 `ID` 换成实际门户实例 id。SSH 连接使用已登记公钥对应的本机私钥，私钥不上传平台。

### 占用与排队判断（CLI 0.6.0+）

- `orbit status --json` 汇总全体节点和**当前用户**申请：`cluster` 为节点原始数据，`instances` 为自己的资源，`scope: current_user` 指申请列表。两次查询可能有时间差；旧 CLI 没有该命令时分别用 `nodes` 与 `instances list`，不要假定所有节点仅供当前用户使用。
- `orbit nodes --json` 查看 `partitions`、`state`、`avail`、`reason`、`gres_total/used/free`、CPU 和内存余量；`--details` 展开完整字段。卡数是 Slurm 分配量，不是 GPU 利用率。`null`/文本 `—` 是未知，不算作 0；查询失败不能解读为没有占用。
- `orbit queue --json` 只筛选当前用户的 `PENDING` 申请，返回 `instances` 数组；无排队时为空数组。`instances show ID` 查看单个申请当前状态；`instances wait ID --until ssh-ready` 等待分配并启动 SSH，超时不取消申请。
- **当前终端 API 不提供全体用户作业队列、精确名次、等待原因或预计启动时间。** 不按列表顺序推算“前面还有几人”，不把空卡数当作立即调度承诺。需要精确信息时说明该接口限制，由主项目维护者扩展接口；不读取网页 Cookie、索要管理员权限或临时绕过终端 API。
- 排队时结合 `nodes`、`quota`、`accounts` 和申请的套餐/指定节点检查。未分配 GPU 也可能位于离线/维护节点或停用分区，或不满足 CPU/内存及账户限制；`PENDING` 不能直接解释为“卡满了”。先查询状态，不重复创建同一申请。

## 写操作与恢复

- 停机、删除资源/镜像/公钥、强制覆盖镜像等操作在用户已经授权后使用 `--yes`，不要把查询请求当作写操作授权。授权已经明确时无需重复询问。
- 手动停机不会自动保存环境。用户需要保留修改时，先保存镜像并确认成功；保存不包含进程、内存和挂载数据。
- 重启使用原镜像并重新分配 SSH 端口，不恢复未保存的修改。退出 CLI 不会停止任务。
- 写请求超时后先查询任务、镜像、日志，判断是否已成功，不盲目重试创建、重启或镜像导出。
- 退出码 3 表示需登录，让用户在终端重新认证；2 是配置/参数问题，4 是网络问题，1 是业务拒绝或等待超时，0 成功。

个人镜像、显示名与 SSH 公钥的其他命令先读对应的 `--help`。资产页中的数据集和模型仓库目前是前端占位，不能声称已通过 CLI 查询或管理这些资产。

## 更新与卸载

- `orbit skill status --json` 检查安装状态与门户版本；`orbit skill install --agent codex|claude|all` 安装本 skill，`orbit skill update` 更新已安装的副本。
- 登录后和每日首次运行时检查；未安装会提示安装，未被本地修改的已安装 skill 会在每日检查时更新。`orbit config set auto_update false` 可关闭每日检查。含本地修改的副本只有用户同意后才用 `--force` 覆盖，并保留 `.bak` 备份。
- `orbit update --check` 只检查；用户要求更新 CLI 时执行 `orbit update`，同时更新已安装的 skill，保留配置和凭据，不延长认证期限。旧版没有这些命令时重新执行同一门户安装命令升级一次。
- 用户要求卸载时用 `orbit uninstall`，明确要求清除本机配置和登录时再加 `--purge`；已获授权的非交互执行加 `--yes --json`。`orbit logout` 撤销当前服务器令牌；卸载不停止集群任务、不删除服务器资产，默认保留本机配置和已安装的 skill。
