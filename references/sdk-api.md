# OmniBox SDK API 速查

> 基于最新官方文档同步：
> https://omnibox-doc.pages.dev/spider-development/sdk.html

官方当前将 SDK 说明收拢为统一页面，JavaScript / Python 共享同一套函数名。

## 引入方式

### JavaScript
```javascript
const OmniBox = require("omnibox_sdk");
const runner = require("spider_runner");
module.exports = { home, category, detail, search, play };
runner.run(module.exports);
```

### Python
```python
from spider_runner import OmniBox, run
if __name__ == "__main__":
    run({"home": home, "category": category, "detail": detail, "search": search, "play": play})
```

---

## 基础能力

### `request(url, options)`
作用：
- 发送 HTTP 请求
- 返回 `statusCode / headers / body`

#### JavaScript
```javascript
const response = await OmniBox.request("https://api.example.com/data", {
  method: "GET",
  headers: { "User-Agent": "Mozilla/5.0" },
});
const data = JSON.parse(response.body || "{}");
```

#### Python
```python
import json
response = await OmniBox.request("https://api.example.com/data", {
    "method": "GET",
    "headers": {"User-Agent": "Mozilla/5.0"},
})
data = json.loads(response.get("body", "{}"))
```

### `log(level, message)`
作用：
- 输出日志，便于调试与排错

### `getEnv(name)`
作用：
- 读取环境变量

说明：
- 实战里通常仍推荐直接用：
  - JS：`process.env.KEY`
  - Python：`os.environ.get("KEY")`

---

## 爬虫源数据

### `getSourceFavoriteTags()`
作用：
- 获取当前源的收藏标签列表

### `getSourceCategoryData(categoryType, page, pageSize)`
作用：
- 获取当前源的历史 / 收藏 / 标签分页数据

---

## 网盘能力

### `getDriveFileList(shareURL, pdirFid)`
作用：
- 获取分享目录下的文件列表

### `getDriveVideoPlayInfo(shareURL, fid, flag)`
作用：
- 获取可播放地址

### `getDriveInfoByShareURL(shareURL)`
作用：
- 识别网盘类型与显示名称

### `getDriveShareInfo(shareURL)`
作用：
- 获取分享信息（类型、token 等）

---

## 刮削能力

### `processScraping(videoId, keyword, resourceName, videoFiles)`
作用：
- 执行刮削任务并保存结果

### `getScrapeMetadata(videoId)`
作用：
- 获取刮削后的元数据，如 `scrapeData`、`videoMappings`

---

## 播放辅助

### `sniffVideo(url, headers)`
作用：
- 嗅探页面中的真实视频地址

说明：
- 依赖完整版运行环境
- 如果已知真实播放页，可优先交给它
- 但它也可能受运行环境 bug 影响，不能盲目当唯一方案

### `getVideoMediaInfo(playUrl, headers)` ✅
作用：
- 根据播放地址探测媒体信息
- 返回 ffprobe 原始结果，常见可用字段包括 `format`、`streams`

常见用途：
- 获取总时长 `format.duration`
- 获取文件大小 `format.size`
- 在 `addPlayHistory` 时补 `totalDuration`
- 在详情页里展示时长 / 大小等媒体信息

说明：
- 需要完整版运行环境
- 更适合在已经拿到可播放地址之后调用，不建议在搜索阶段大规模探测

### `getDanmakuByFileName(fileName)`
作用：
- 根据文件名匹配弹幕

### `getAnalyzeSites()`
作用：
- 获取已配置解析站列表

### `addPlayHistory(data)`
作用：
- 记录观看历史
- 官方文档示例中可配合 `getVideoMediaInfo()` 拿到的时长写入 `totalDuration`

---

## 缓存能力 ✅

### `getCache(key)`
作用：
- 读取缓存，未命中返回 `null / None`

### `setCache(key, value, exSeconds)`
作用：
- 写入缓存
- 支持设置过期秒数

### `deleteCache(key)`
作用：
- 删除缓存，返回删除条数

说明：
- 官方描述为“用于跨请求缓存解析结果（语义接近 Redis）”
- 很适合缓存：
  - 搜索结果
  - 网盘信息识别结果
  - 详情页文件列表
  - 递归目录扫描结果

---

## 当前同步后的理解

- 官方当前不再拆成 JS SDK / Python SDK 两个主文档页面，而是统一在 `sdk.html` 里对照说明。
- 技能内部保留 JS / Python 参考文件可以继续用，但默认应优先把 `sdk-api.md` 视为官方同步主入口。
