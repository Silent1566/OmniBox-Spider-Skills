# Detail 接口缺失播放线路的排查指南

## 背景
在 OmniBox‑Spider 开发过程中，用户可能会遇到 **detail 返回列表中没有 `vod_play_from` / `vod_play_url` / `vod_play_list`** 的情况。常见原因包括：
1. 前端使用了错误的请求路径（`/spider/...` GET 返回 UI 页面），而不是 JSON‑RPC POST 到 `/api/spider/...`。
2. 浏览器/客户端缓存了旧的响应，导致即使后端已修复，前端仍展示空线路。
3. `detail` 实现中对 **无集数的网盘线路** 直接 `continue`，导致对应线路被过滤掉，进而导致 `sourceNames` 与 `sourceUrls` 不匹配。
4. `vod_play_sources` 未正确转化为 legacy 字段，前端渲染层仍旧依赖 `vod_play_from` / `vod_play_url`。

## 排查步骤
1. **确认请求方式**
   ```bash
   curl -X POST -H 'Content-Type: application/json' \
        -d '{"method":"detail","params":{"videoId":"<ID>"}}' \
        http://127.0.0.1:<gateway_port>/api/spider/<script_name>
   ```
   - 必须使用 **POST**，且路径包含 `/api/spider/`。返回的 JSON 必须包含 `vod_play_sources`、`vod_play_from`、`vod_play_url`。
2. **检查后端返回**
   - 在终端直接查看返回的 `vod_play_from`、`vod_play_url`、`vod_play_list`。如果为空，回到代码确认 `detail` 中是否有 `continue` 过滤了全部网盘线路。
3. **清除前端缓存**
   - 在 OmniBox 客户端 → 设置 → 清除本地缓存，或在浏览器打开 DevTools → Application → Clear storage。
   - 重新打开详情页，确保新的响应被拉取。
4. **验证 legacy 字段**
   - `detail` 必须在返回前执行 `Object.assign(vod, buildLegacyPlayFields(vod.vod_play_sources || []))`，保证 `vod_play_from` / `vod_play_url` 与旧版 UI 兼容。
5. **日志对照**
   - 确认 `detail 完成` 日志中 `sourceCount` 大于 0、`episodeCount` 也大于 0。若日志显示 0，则说明在 “网盘线路解析” 或 “刮削候选” 阶段被过滤。

## 常见坑 (Pitfalls)
- **使用 GET `/spider/...`**：返回的是 OpenClaw 控制页面 HTML，前端会报 *Not Found*。始终使用 `/api/spider/...` 并 POST JSON。
- **缓存未刷新**：即使后端已修复，前端仍会使用旧的 `detail` 数据。务必手动清除或在调试时加入 `?nocache=1` 参数（仅用于本地调试）。
- **网盘线路无实际文件时直接返回占位**：`detail` 中对无文件的网盘线路使用 `continue`，并在 `buildLegacyPlayFields` 中只处理有 `episodes` 的 source，防止产生空 `vod_play_url`。

## 推荐实践
- 在 `detail` 完成后统一调用 `logInfo("detail 完成", {...})`，并记录 `sourceCount`、`episodeCount`、`scraped` 等关键指标。
- 对所有 `detail` 实现加入 `Object.assign(vod, buildLegacyPlayFields(vod.vod_play_sources || []))`，确保向后兼容。
- 在 CI 中加入对 `detail` 返回结构的断言：`assert(result.list[0].vod_play_from && result.list[0].vod_play_url && result.list[0].vod_play_list.length > 0)`。
