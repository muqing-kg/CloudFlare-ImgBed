# 自动同步说明

本 fork 自动镜像上游，目标是：**尽量永远不落后、尽量不再静默失败**。

- 上游：https://github.com/MarSeventh/CloudFlare-ImgBed
- 同步分支：`main`
- 主工作流：`.github/workflows/auto-sync-upstream.yml`
- 频率：
  - 每 6 小时（UTC）
  - 每天 01:20（UTC）额外兜底一次
  - Actions 页面可手动 `Run workflow`
- 策略：每次把 `main` **hard reset** 到上游 `main`，再写回本 fork 维护文件
- 冲突处理：不手解冲突，直接以上游为准重建
- 无变化判断：比较 tree，不比较 commit SHA（避免每次空跑都 force-push）

## 维护文件（每次同步后都会恢复）

```text
.github/workflows/auto-sync-upstream.yml
.github/workflows/sync-upstream.yml
AUTO_SYNC.md
```

说明：

- `auto-sync-upstream.yml`：真正执行镜像的工作流
- `sync-upstream.yml`：兼容上游同名入口，但**去掉了定时任务**，手动触发时转调 `auto-sync-upstream`
  - 这样可避免上游自带 Upstream Sync 在 fork 上继续用默认 `GITHUB_TOKEN` 失败、抢写 `main`

## 必需 Secret

名称：`SYNC_UPSTREAM_TOKEN`

为什么必需：

- GitHub 默认 `GITHUB_TOKEN` **不能** 创建/更新 `.github/workflows/*`
- 上游一旦改了 workflow，用默认 token 的同步 push 会被拒绝
- 这正是过去自动同步“看起来有、实际全红”的根因

推荐创建 **Classic PAT**（更省事）：

- 勾选：`repo` + `workflow`
- 不过期（或设很长过期并日历提醒）
- 写入本仓库 Actions Secret：

```bash
gh secret set SYNC_UPSTREAM_TOKEN --repo muqing-kg/CloudFlare-ImgBed
```

也可用 Fine-grained PAT：

- Repository access：仅本仓库
- Permissions：
  - Contents: Read and write
  - Workflows: Read and write
  - Metadata: Read

## 失败告警

- 仓库已开启 Issues
- 同步失败时自动打开/更新 Issue：`[auto-sync] Upstream synchronization failed`
- 下一次同步成功后自动关闭

## 使用注意

- 不要在 `main` 上堆长期私有改动；会被下次同步覆盖
- 私有改动请放独立分支/独立 fork 策略
- 仓库自带的 `docker-publish.yml` / `sync-release.yml` 通常只对上游官方仓有意义
- `deploy-worker.yml` 仍会在 `main` 有新推送时尝试部署（需 Cloudflare secrets）

## 手动恢复

```bash
# 1) 确保 secret 有效
gh secret set SYNC_UPSTREAM_TOKEN --repo muqing-kg/CloudFlare-ImgBed

# 2) 手动跑一次
gh workflow run auto-sync-upstream.yml --repo muqing-kg/CloudFlare-ImgBed --ref main

# 3) 看结果
gh run list --repo muqing-kg/CloudFlare-ImgBed --workflow auto-sync-upstream.yml --limit 5
```
