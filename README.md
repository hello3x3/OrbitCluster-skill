# OrbitCluster agent skill

Agent skill（Codex / Claude Code）for the OrbitCluster user CLI `orbit`. It is mounted as the
`skills/orbitcluster` submodule of [hello3x3/OrbitCluster](https://github.com/hello3x3/OrbitCluster).

## 用户安装与更新

用户不需要克隆本仓库。skill 随门户发布，由 CLI 从用户所在门户下载并校验 SHA256：

```bash
orbit skill install        # 安装到 ~/.codex/skills/orbitcluster、~/.claude/skills/orbitcluster
orbit skill status         # 查看是否有新版
orbit skill update         # 更新；orbit update 也会一并更新
```

CLI 每天首次运行时自动检查，未被本地修改的 skill 会自动更新。

## 发布新版本

1. 修改 `SKILL.md`，并把 `VERSION` 递增（如 `1.1.0` → `1.2.0`），提交并推送本仓库。
2. 在 OrbitCluster 主仓库更新子模块指针并打包：

   ```bash
   git submodule update --remote skills/orbitcluster
   python3 oa/cluster-portal/scripts/bundle_docs.py
   git add skills/orbitcluster oa/cluster-portal/portalapp/static/docs
   git commit -m "chore(skill): bump orbitcluster skill to X.Y.Z"
   ```

3. 部署门户后，`/terminal/skill.json` 发布新版本与 SHA256，用户端 CLI 随即可更新。
