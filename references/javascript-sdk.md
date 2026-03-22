# JavaScript SDK | OmniBox

OmniBox JavaScript SDK 提供请求、日志、环境变量、爬虫源数据、网盘、刮削、弹幕、解析站、观看记录等能力。

## 引入方式

```javascript
const OmniBox = require("omnibox_sdk");
const runner = require("spider_runner");

module.exports = { home, category, detail, search, play };
runner.run(module.exports);
```

### 说明
- SDK 通过运行器注入的 `NODE_PATH` 自动解析
- handler 推荐统一签名：`(params, context)`
- `context.from` 表示调用端：`web / tvbox / uz / catvod / emby`
- 无需手写 stdin / stdout 协议，统一由 `runner.run()` 处理

---

## 1. OmniBox.request(url, options)

### 作用
发送 HTTP 请求，用于访问第三方网站、API、详情页、搜索接口等。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `url` | `string` | 是 | 请求地址 |
| `options` | `object` | 否 | 请求配置对象 |
| `options.method` | `string` | 否 | HTTP 方法，默认 `GET` |
| `options.headers` | `object` | 否 | 请求头 |
| `options.body` | `string \| object` | 否 | 请求体 |

### 出参
返回一个对象：

| 字段 | 类型 | 说明 |
|---|---|---|
| `statusCode` | `number` | HTTP 状态码 |
| `headers` | `object` | 响应头 |
| `body` | `string` | 响应体，通常需要手动 `JSON.parse` |

### 示例

```javascript
const response = await OmniBox.request("https://api.example.com/data", {
  method: "GET",
  headers: {
    "User-Agent": "Mozilla/5.0",
    Referer: "https://example.com/",
  },
});

if (response.statusCode !== 200) {
  throw new Error(`HTTP ${response.statusCode}: ${response.body}`);
}

const data = JSON.parse(response.body || "{}");
```

### 注意事项
- `body` 默认是字符串，不是自动解析后的 JSON
- 推荐统一封装成 `requestApi()`，避免每个 handler 重复写状态码判断和解析逻辑

---

## 2. OmniBox.log(level, message)

### 作用
输出日志，便于在后台调试脚本执行过程。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `level` | `string` | 是 | 日志级别：`info` / `warn` / `error` |
| `message` | `string` | 是 | 日志内容 |

### 出参
- `Promise<void>`

### 示例

```javascript
await OmniBox.log("info", "开始获取首页数据");
await OmniBox.log("warn", "某字段为空，走兜底逻辑");
await OmniBox.log("error", `请求失败: ${error.message}`);
```

### 建议
- `home/category/search/detail/play` 每个阶段都打日志
- 日志里尽量带关键参数和数量信息

---

## 3. OmniBox.getEnv(name)

### 作用
读取环境变量。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `name` | `string` | 是 | 环境变量名 |

### 出参
- `Promise<string>`

### 示例

```javascript
const apiKey = await OmniBox.getEnv("API_KEY");
```

### 推荐写法
文档推荐优先直接使用：

```javascript
const apiKey = process.env.API_KEY || "";
```

### 说明
- `process.env` 性能更好
- `getEnv()` 更适合你明确想走 SDK 统一接口时使用

---

## 4. OmniBox.getSourceFavoriteTags()

### 作用
获取当前爬虫源的收藏标签列表。

### 入参
- 无显式参数
- 依赖 `context.sourceId` 由运行器注入

### 出参
- `Promise<string[]>`

### 示例

```javascript
const tags = await OmniBox.getSourceFavoriteTags();
```

### 适用场景
- 根据当前源的标签决定首页推荐内容
- 做收藏/标签分组展示

---

## 5. OmniBox.getSourceCategoryData(categoryType, page, pageSize)

### 作用
获取当前爬虫源的分类数据，如历史、收藏、标签等。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `categoryType` | `string` | 是 | 分类类型：`history` / `favorite` / `tag` |
| `page` | `number` | 否 | 页码，默认 `1` |
| `pageSize` | `number` | 否 | 每页数量，默认 `20` |

### 出参
返回对象：

| 字段 | 类型 | 说明 |
|---|---|---|
| `list` | `array` | 数据列表 |
| `total` | `number` | 总数量 |
| `pageCount` | `number` | 总页数 |

### 示例

```javascript
const history = await OmniBox.getSourceCategoryData("history", 1, 20);
const favorite = await OmniBox.getSourceCategoryData("favorite", 1, 20);
```

### 注意
- 该 API 依赖 `context.sourceId`
- 更适合“我的历史 / 我的收藏 / 标签页”这类源内数据场景，不是第三方站点抓取 API

---

## 6. OmniBox.getDriveFileList(shareURL, pdirFid = "0")

### 作用
读取网盘分享目录下的文件列表。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `shareURL` | `string` | 是 | 网盘分享链接 |
| `pdirFid` | `string` | 否 | 父目录 ID，默认 `"0"` 表示根目录 |

### 出参
返回对象：

| 字段 | 类型 | 说明 |
|---|---|---|
| `files` | `array` | 文件列表 |
| `total` | `number` | 总文件数 |
| `has_more` | `boolean` | 是否还有更多 |

### 示例

```javascript
const fileList = await OmniBox.getDriveFileList("https://pan.example.com/s/xxx", "0");
```

### 适用场景
- 网盘源展开目录
- 获取视频文件清单后生成剧集列表

---

## 7. OmniBox.getDriveVideoPlayInfo(shareURL, fid, flag)

### 作用
获取网盘视频的播放信息。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `shareURL` | `string` | 是 | 网盘分享链接 |
| `fid` | `string` | 是 | 文件 ID |
| `flag` | `string` | 否 | 播放方式标识，如服务端代理、本地代理、直连 |

### 出参
返回对象：

| 字段 | 类型 | 说明 |
|---|---|---|
| `url` | `string` | 播放地址 |
| `header` | `object` | 请求头 |
| `danmaku` | `array` | 弹幕列表 |

### 示例

```javascript
const playInfo = await OmniBox.getDriveVideoPlayInfo("https://pan.example.com/s/xxx", "file_id");
```

### 适用场景
- 网盘源 `play()` 方法
- 从网盘文件直接获取可播地址

---

## 8. OmniBox.getDriveShareInfo(shareURL)

### 作用
获取网盘分享信息。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `shareURL` | `string` | 是 | 网盘分享链接 |

### 出参
- `Promise<object>`

### 示例

```javascript
const shareInfo = await OmniBox.getDriveShareInfo("https://pan.example.com/s/xxx");
```

### 说明
- 官方文档未细展开返回字段，通常用于补充网盘分享元信息

---

## 9. OmniBox.getDriveInfoByShareURL(shareURL)

### 作用
根据分享链接识别网盘类型、显示名称、图标等信息。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `shareURL` | `string` | 是 | 网盘分享链接 |

### 出参
返回对象：

| 字段 | 类型 | 说明 |
|---|---|---|
| `driveType` | `string` | 网盘类型，如 `uc` |
| `displayName` | `string` | 显示名称 |
| `iconPath` | `string` | 图标路径 |
| `iconUrl` | `string` | 图标 URL |

### 示例

```javascript
const info = await OmniBox.getDriveInfoByShareURL("https://pan.example.com/s/xxx");
// { driveType, displayName, iconPath, iconUrl }
```

---

## 10. OmniBox.processScraping(resourceId, keyword, resourceName, videoFiles)

### 作用
执行刮削流程，为资源生成刮削元数据。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `resourceId` | `string` | 是 | 资源唯一标识 |
| `keyword` | `string` | 否 | 搜索关键词 |
| `resourceName` | `string` | 否 | 资源名称 |
| `videoFiles` | `array` | 是 | 视频文件列表 |

### 出参
- `Promise<object>`

### 示例

```javascript
const result = await OmniBox.processScraping(
  "resource_123",
  "凡人修仙传",
  "凡人修仙传",
  videoFiles
);
```

### 经验规则
- `resourceId` 和后续 `getScrapeMetadata(resourceId)` 必须一致
- `videoFiles` 里最好带 `format_type: "video"`

---

## 11. OmniBox.processDriveScraping(shareURL, keyword, resourceName, videoFiles)

### 作用
网盘场景下的刮削别名方法，本质上与 `processScraping()` 类似。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `shareURL` | `string` | 是 | 网盘分享链接，在网盘场景下它就是资源 ID |
| `keyword` | `string` | 否 | 搜索关键词 |
| `resourceName` | `string` | 否 | 资源名称 |
| `videoFiles` | `array` | 是 | 视频文件列表 |

### 出参
- `Promise<object>`

### 示例

```javascript
const result = await OmniBox.processDriveScraping(
  "https://pan.example.com/s/xxx",
  "资源标题",
  "资源标题",
  videoFiles
);
```

---

## 12. OmniBox.getScrapeMetadata(resourceId)

### 作用
获取刮削后的元数据。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `resourceId` | `string` | 是 | 资源唯一标识 |

### 出参
返回对象通常包含：

| 字段 | 类型 | 说明 |
|---|---|---|
| `scrapeData` | `object \| null` | 刮削结果 |
| `videoMappings` | `array \| null` | 文件与 TMDB/剧集映射关系 |

### 示例

```javascript
const metadata = await OmniBox.getScrapeMetadata("resource_123");
```

### 典型用途
- 在 `detail()` 中把 `videoMappings` 映射到 `episodes`
- 补充剧集名称、简介、首播日期、剧照、评分、时长等

---

## 13. OmniBox.getDriveMetadata(shareURL)

### 作用
获取网盘元数据，是 `getScrapeMetadata()` 的网盘场景别名。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `shareURL` | `string` | 是 | 网盘分享链接 |

### 出参
- `Promise<object>`
- 常见字段：`scrapeData`、`videoMappings`

### 示例

```javascript
const metadata = await OmniBox.getDriveMetadata("https://pan.example.com/s/xxx");
```

---

## 14. OmniBox.sniffVideo(url, headers?)

### 作用
通过加载播放页并拦截网络请求，匹配真实视频地址。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `url` | `string` | 是 | 待解析的播放页 URL |
| `headers` | `object` | 否 | 请求头，如 `Referer` / `User-Agent` / `Cookie` |

### 出参
返回对象：

| 字段 | 类型 | 说明 |
|---|---|---|
| `url` | `string` | 嗅探出来的真实视频地址 |
| `header` | `object` | 播放所需请求头 |

### 示例

```javascript
const result = await OmniBox.sniffVideo("https://example.com/play?id=123", {
  Referer: "https://example.com/",
});
```

### 注意事项
- 需要完整版运行环境（Playwright / Chromium）
- 适合对付解析页、跳转页、前端拼接直链等场景

---

## 15. OmniBox.getDanmakuByFileName(fileName)

### 作用
根据文件名匹配弹幕源。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `fileName` | `string` | 是 | 用于匹配弹幕的文件名 |

### 出参
- `Promise<Array<{ name: string, url: string }>>`

### 示例

```javascript
const danmaku = await OmniBox.getDanmakuByFileName("凡人修仙传.S01E01.mp4");
```

### 适用场景
- 在 `play()` 返回 `danmaku`
- 在网盘源里按文件名自动匹配弹幕

---

## 16. OmniBox.getAnalyzeSites()

### 作用
获取系统中已配置的解析站列表。

### 入参
- 无

### 出参
返回数组，每项结构：

| 字段 | 类型 | 说明 |
|---|---|---|
| `name` | `string` | 解析站名称 |
| `url` | `string` | 接口地址 |
| `type` | `number` | `0=Web`，`1=JSON` |

### 示例

```javascript
const sites = await OmniBox.getAnalyzeSites();
const webSites = sites.filter((s) => s.type === 0);
```

### 适用场景
- 动态构建解析线路
- 在 `play()` 中给用户展示可选解析站

---

## 17. OmniBox.addPlayHistory(historyItem)

### 作用
添加观看记录（若不存在）。

### 入参
`historyItem` 为对象，常见字段：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `vodId` | `string` | 是 | 视频 ID |
| `title` | `string` | 是 | 标题 |
| `episode` | `string` | 是 | 当前剧集标识 |
| `pic` | `string` | 否 | 封面 |
| `episodeName` | `string` | 否 | 剧集名称 |
| `playUrl` | `string` | 否 | 播放地址 |
| `playHeader` | `object` | 否 | 播放请求头 |
| `totalDuration` | `number` | 否 | 总时长（秒） |

### 出参
- `Promise<boolean>`
- 表示是否成功新增记录

### 示例

```javascript
const added = await OmniBox.addPlayHistory({
  vodId: "video_123",
  title: "示例视频",
  episode: "ep_1",
  playUrl: "https://example.com/video.m3u8",
  playHeader: { Referer: "https://example.com/" },
});
```

### 说明
- `sourceId` 由 `context` 自动读取
- 如果你已知时长，可直接传 `totalDuration`

---

## 通用开发建议

### 统一请求封装
```javascript
async function requestApi(params = {}) {
  const url = buildUrl(params);
  const response = await OmniBox.request(url, { method: "GET" });
  if (response.statusCode !== 200) {
    throw new Error(`HTTP ${response.statusCode}: ${response.body}`);
  }
  return JSON.parse(response.body || "{}");
}
```

### 推荐开发习惯
1. 统一封装请求函数
2. handler 内只做：参数读取 → 请求 → 映射 → 返回
3. 所有 handler 用 `try/catch`
4. 关键步骤统一打日志
5. 环境变量优先 `process.env`
6. `detail()` 里构建 `vod_play_sources`
7. `play()` 里优先返回 `urls + flag + header + parse + danmaku`

### 典型空返回

#### home 失败
```javascript
return { class: [], list: [] };
```

#### category / search 失败
```javascript
return { page: 1, pagecount: 0, total: 0, list: [] };
```

#### detail 失败
```javascript
return { list: [] };
```

#### play 失败
```javascript
return { url: "", flag: params.flag || "play", header: {} };
```
