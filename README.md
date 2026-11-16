# OrbitCluster agent skill

面向 Codex / Claude Code 的 OrbitCluster 用户操作说明。版本源为 `VERSION`，入口为 `SKILL.md`。

**本分支为 1.4.0 适配草稿：验证码说明对应 CLI 0.5.0 适配分支，需要主项目维护者先接入终端验证码接口再一起发布。现有门户只有网页验证码时，不能宣称终端已支持验证码登录。**

## 用户安装与更新

CLI 0.4.2+ 从用户配置的门户获取 `/terminal/skill.json` 和 `/terminal/skill.md`，校验大小与 SHA256 后写入 agent 目录；用户不需要访问 GitHub。

```bash
orbit skill status --json
orbit skill install --agent codex   # 或 claude / all；省略时选择已配置的 agent
orbit skill update
orbit update --check
```

登录和每日检查会提醒未安装的用户；每日检查自动更新未被本地修改的副本。CLI 只提示新版，用户执行 `orbit update` 才安装。含本地修改的 skill 需明确同意后使用 `--force`，并保留备份。旧 CLI 没有以上命令时，先重新执行门户安装器。

## 三仓库发布顺序

依赖关系为 `OrbitCluster/terminal` → `OrbitCluster-CLI`，再由 CLI 的 `skills/orbitcluster` → 本仓库。

1. 修改 `SKILL.md` 并递增 `VERSION`，测试、提交并推送本仓库。
2. 在 **OrbitCluster-CLI 仓库根目录**运行 `git submodule update --remote skills/orbitcluster`，提交并推送子模块指针。
3. 在 **OrbitCluster 主仓库根目录**运行 `git submodule update --remote terminal` 和 `git submodule update --init --recursive`，重新 `mvn -f oa/cluster-portal/pom.xml clean verify` 并打包部署门户，提交 `terminal` 指针。

Maven 从两级子模块打包 skill 文档与版本；门户据此生成 skill 清单。仅更新 skill 不要求重新编译 CLI。若 CLI 源码也变更，则必须递增 CLI 版本并重新生成、部署对应安装包。推送 GitHub 本身不会更新已部署门户或用户文件。

## 与主项目文档对齐

每次更新前只读检查 OrbitCluster 主仓库最新 `main` 的 `docs/user-guide.md`、`oa/03-使用手册.md`、`base-cluster/images/README.md` 和两个基础镜像 Dockerfile，再核对独立 CLI 的 `--help` 与实际 API 支持。重点核对认证方式、资源操作规则、镜像保存上限/自动保存，以及预装 Conda 的位置与环境保存方式。

基础镜像预装 `/opt/miniconda3`；环境用显式 prefix 建在容器内的 `/opt/miniconda3/envs`，不能建在 home、`/data` 或 NAS 等挂载存储里。需要长期保留时保存个人镜像。主项目接口尚未支持的能力要列明依赖，不得把文档描述当作已经可用的客户端能力。

本次工作范围为 CLI 与 skill；主仓库保持只读。主项目后端接口与部署引用的变更由其维护者处理。

本次核对的主项目参考提交为 `ee4e1b6`；未改动该主仓库。
