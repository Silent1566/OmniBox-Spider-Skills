# MacCMS / myui 详情页线路兼容笔记

## 背景
在 `歪比巴卜.js` 调试中出现：`detail({ videoId: "18377" })` 能返回详情元数据（标题、封面、年份、导演、演员、简介等），但 OmniBox 前端详情页不显示任何播放线路。

## 关键发现
1. 宿主调用 `detail` 时传的是 `videoId`，不是传统 `id / ids / vod_id`。
   - 如果脚本只解析 `id/ids/vod_id`，会出现日志 `params={"videoId":"18377"}` 但 `total items=0`。
   - 修复：`detail()` 必须兼容 `params.videoId` 与 `params.video_id`。

2. 目标站详情页存在新版播放 DOM：
   - 线路名在：`module-tab-item tab-item` 的 `data-dropdown-value`
   - 剧集列表在：`module-play-list-content module-play-list-base`
   - 播放链接形态：`/vplay/{vodId}-{sid}-{nid}.html`

3. 该类站点需要同时返回宿主旧链路字段：
   - `vod_play_from`: 多线路名用 `$$$` 分隔
   - `vod_play_url`: 同线路内 `集名$播放URL` 用 `#` 分隔，不同线路用 `$$$` 分隔

## 最小修复模式
```js
let ids = [];
if (Array.isArray(params.id)) ids = params.id;
else if (Array.isArray(params.ids)) ids = params.ids;
else if (params.ids) ids = String(params.ids).split(',').map(s => s.trim()).filter(Boolean);
else if (params.id) ids = String(params.id).split(',').map(s => s.trim()).filter(Boolean);
else if (params.vod_id) ids = String(params.vod_id).split(',').map(s => s.trim()).filter(Boolean);
else if (params.videoId) ids = String(params.videoId).split(',').map(s => s.trim()).filter(Boolean);
else if (params.video_id) ids = String(params.video_id).split(',').map(s => s.trim()).filter(Boolean);
```

## 验证要点
- `node --check 影视/采集/xxx.js` 必须通过。
- 关键路径验证：
  - `detail({ videoId: '18377' })` 返回 `list.length === 1`
  - `list[0].vod_play_from` 非空，如：`推荐$$$蓝光4K$$$蓝光A...`
  - `list[0].vod_play_url` 非空，形如：`第01集$https://.../vplay/...#第02集$...`

## 调试日志建议
`detail()` 至少输出：
- 入参：`[wbbb][detail] params=...`
- 请求 ID：`fetch detail for ...`
- 解析结果：`tabs=<count> groups=<count> playFrom=<names>`
- 返回数量：`total items=...`

如果详情元数据有但没线路，优先检查 `groups.length`、`vod_play_from`、`vod_play_url`，而不是继续改标题/简介解析。
