# UC网盘处理指南

## 背景
- LIBVIO 等站点的 **UC网盘** 通常在详情页只提供一个 **合集** 链接（例：`/play/5812602-5-1.html`）。
- 该合集页面内部通过 `player_aaaa` 变量暴露真实的分享链接，例如:
  ```js
  var player_aaaa={
    "url":"https://drive.uc.cn/s/3f44153d59d64",
    "from":"ucpan",
    ...
  }
  ```
- 旧实现只在 `splitNetdiskPanels()` 捕获 `网盘|视频下载` 标题，导致 **UC网盘** 被误归入 `collectSources`，从而没有进入网盘展开逻辑。

## 关键改动
1. **面板提取**
   - `splitNetdiskPanels()` 正则已改为直接匹配 `<h3>…网盘…</h3>` + `<ul>...</ul>`，保证 UC 网盘标题也会被捕获。
2. **UC 兜底解析**
   - 在 `detail()` 中对 `allCollectSources` 里检测 `normalizePanSourceName(...) === "UC网盘"`，单独打开该合集页面。
   - 读取页面源码，提取 `player_aaaa.url`（或页面中出现的 `drive.uc.cn` / `uc.cn/s/` 链接），使用 `loadPanFiles()` 展开文件列表。
   - 为每个文件构造 `playId`，并使用 `buildPanEpisodePlayId()` 将 `shareUrl|fileId` 正确写入 `meta.fid`，确保后续刮削映射、弹幕匹配和播放记录能够命中。
3. **日志**
   - 添加 `detail 开始处理 UC 网盘采集线路`、`detail UC 网盘通过采集线路解析成功`、`detail UC 网盘未能提取有效分享链接` 等日志，便于快速定位问题。
4. **过滤**
   - 在过滤 `collectSources` 时排除已经加入 `netdiskSources`（包括刚解析的 UC），防止重复出现 `合集` 占位。

## 调试步骤
1. **确认面板捕获**
   - 在任意包含 UC 网盘的详情页执行:
     ```bash
     grep -n "UC网盘" -A2 -B2 <detail-html>
     ```
   - 确认 `splitNetdiskPanels()` 输出的面板数组里包含 `UC网盘`。
2. **检查 UC 播放页**
   - 访问对应的合集链接（如 `/play/5812602-5-1.html`），查看源码是否有 `player_aaaa` 并确认 `url` 为 **drive.uc.cn** 或 **uc.cn/s/**。
3. **观察日志**
   - 完整运行 `detail`，在日志中应出现:
     - `detail 开始处理 UC 网盘采集线路`
     - `detail UC 网盘通过采集线路解析成功`（若成功）
   - 若出现 `detail UC 网盘未能提取有效分享链接`，检查是否 `player_aaaa.url` 被遮蔽或页面结构变更。
4. **验证播放**
   - 在返回的 `vod_play_sources` 中找到 `UC网盘`，对应的 `episodes` 应包含实际集数（如 `第01集`、`第02集`），并且每个 `playId` 里 `fid` 为 `shareUrl|fileId`。
   - 调用 `play` 时应能直接走 `loadPanFiles()` 并返回有效 `urls`（直链或 `push://` 形式）。

## 常见问题及排查
| 症状 | 可能原因 | 排查方法 |
|------|----------|----------|
| `detail` 日志没有出现 `UC网盘` | `splitNetdiskPanels` 正则未匹配，或页面标题不含 `网盘`/`下载` | 检查 `panelHtml` 内容，确认 `<h3>` 中的文字。
| `UC网盘` 进入 `collectSources` 但未展开 | `player_aaaa` 中没有 `url` 或被混淆为普通播放页 | 查看播放页源码，确认 `player_aaaa.url` 是否存在且为 `drive.uc.cn` 或 `uc.cn/s/`。
| `play` 返回 `parse:1` 或空 `urls` | `loadPanFiles` 返回空文件列表或 `shareUrl` 未通过 `isPanUrl` 检测 | 手动 `curl` 访问 `shareUrl`，确认返回有效文件列表；若需要密码，确保 `shareUrl` 包含 `?pwd=`。
| 刮削映射未命中 | `meta.fid` 仍为空或格式不符合 `shareUrl|fileId` | 在 `detail` 日志中检查 `episodes` 的 `playId` 是否包含 `|||`，并确认 `fid` 已写入。

## 推荐迁移路径
- **旧版**：仅依赖 `collectSources`，导致 UC 只出现占位合集。
- **新版**：面板捕获 + UC 兜底解析，确保所有 UC 分享都能展开为真实集数。
- 在升级脚本后，运行一次完整的 `detail` + `play` 流程，确保没有 `UC网盘` 相关的 `detail 移除未展开或不支持的网盘线路` 警告。

---

**注意**：如果站点未来改为直接在详情页 `player_aaaa` 中提供完整文件列表，而不再需要二次打开合集页面，可进一步简化此流程，仅在 `detail` 的 `netdiskPanels` 循环里直接检查 `player_aaaa.url` 即可。