# OmniBox SDK API 速查

这是 JS / Python 的对照速查表，按“作用 / 入参 / 出参 / 示例”整理。

---

## 1. HTTP 请求

### JS
`OmniBox.request(url, options)`

### Python
`OmniBox.request(url, options)`

### 作用
发送 HTTP 请求，访问第三方 API 或页面。

### 入参
| 参数 | 类型 | 说明 |
|---|---|---|
| `url` | string / str | 请求地址 |
| `options.method` | string / str | HTTP 方法 |
| `options.headers` | object / dict | 请求头 |
| `options.body` | string\|object / str\|dict | 请求体 |

### 出参
| 字段 | 类型 | 说明 |
|---|---|---|
| `statusCode` | number / int | HTTP 状态码 |
| `headers` | object / dict | 响应头 |
| `body` | string / str | 响应体 |

### 示例
```javascript
const res = await OmniBox.request(url, { method: "GET", headers: {} });
const data = JSON.parse(res.body || "{}");
```
```python
res = await OmniBox.request(url, {"method": "GET", "headers": {}})
data = json.loads(res.get("body", "{}"))
```

---

## 2. 日志

### JS
`OmniBox.log(level, message)`

### Python
`OmniBox.log(level, message)`

### 作用
记录调试日志。

### 入参
| 参数 | 类型 | 说明 |
|---|---|---|
| `level` | string / str | `info` / `warn` / `error` |
| `message` | string / str | 日志内容 |

### 出参
- 无

### 示例
```javascript
await OmniBox.log("info", "开始执行")
```
```python
await OmniBox.log("info", "开始执行")
```

---

## 3. 环境变量

### JS
- 推荐：`process.env.KEY`
- 可选：`await OmniBox.getEnv("KEY")`

### Python
- 推荐：`os.environ.get("KEY", "")`
- 可选：`await OmniBox.get_env("KEY")`

### 作用
读取环境变量配置。

### 入参
| 参数 | 类型 | 说明 |
|---|---|---|
| `name` | string / str | 环境变量名 |

### 出参
- string / str

---

## 4. 爬虫源标签

### JS
`OmniBox.getSourceFavoriteTags()`

### Python
`OmniBox.get_source_favorite_tags()`

### 作用
读取当前源的收藏标签。

### 入参
- 无（依赖 `context.sourceId`）

### 出参
- `string[]` / `list[str]`

---

## 5. 爬虫源分类数据

### JS
`OmniBox.getSourceCategoryData(type, page, pageSize)`

### Python
`OmniBox.get_source_category_data(type, page, page_size)`

### 作用
读取当前源的历史、收藏、标签分类数据。

### 入参
| 参数 | 说明 |
|---|---|
| `type` | `history` / `favorite` / `tag` |
| `page` | 页码 |
| `pageSize/page_size` | 每页数量 |

### 出参
| 字段 | 说明 |
|---|---|
| `list` | 数据列表 |
| `total` | 总数 |
| `pageCount` | 总页数 |

---

## 6. 网盘文件列表

### JS
`getDriveFileList(shareURL, pdirFid="0")`

### Python
`get_drive_file_list(share_url, pdir_fid="0")`

### 作用
获取网盘文件列表。

### 入参
- 分享链接
- 父目录 ID

### 出参
- `{ files, total, has_more }`

---

## 7. 网盘视频播放信息

### JS
`getDriveVideoPlayInfo(shareURL, fid, flag?)`

### Python
`get_drive_video_play_info(share_url, fid, flag="")`

### 作用
获取网盘视频播放地址与请求头。

### 入参
- 分享链接
- 文件 ID
- 播放方式标识

### 出参
- `{ url, header, danmaku }`

---

## 8. 网盘信息识别

### JS
`getDriveInfoByShareURL(shareURL)`

### Python
`get_drive_info_by_share_url(share_url)`

### 作用
识别网盘类型、名称、图标。

### 出参
- `{ driveType, displayName, iconPath, iconUrl }`

---

## 9. 刮削

### JS
- `processScraping(resourceId, keyword, resourceName, videoFiles)`
- `getScrapeMetadata(resourceId)`

### Python
- `process_scraping(resource_id, keyword, resource_name, video_files)`
- `get_scrape_metadata(resource_id)`

### 作用
执行刮削并读取刮削元数据。

### 入参
- `resourceId/resource_id`
- `keyword`
- `resourceName/resource_name`
- `videoFiles/video_files`

### 出参
- 刮削结果对象
- metadata 通常含 `scrapeData`, `videoMappings`

---

## 10. 视频嗅探

### JS
`sniffVideo(url, headers?)`

### Python
`sniff_video(url, headers=None)`

### 作用
在播放页中捕获真实视频地址。

### 入参
- URL
- 请求头（可选）

### 出参
- `{ url, header }`

### 注意
- 需要完整版运行环境（Playwright / Chromium）

---

## 11. 弹幕

### JS
`getDanmakuByFileName(fileName)`

### Python
`get_danmaku_by_file_name(file_name)`

### 作用
按文件名匹配弹幕源。

### 出参
- `[{ name, url }]`

---

## 12. 解析站

### JS
`getAnalyzeSites()`

### Python
`get_analyze_sites()`

### 作用
获取解析站配置。

### 出参
- `[{ name, url, type }]`
- `type: 0=Web, 1=JSON`

---

## 13. 观看记录

### JS
`addPlayHistory(historyItem)`

### Python
`add_play_history(...)`

### 作用
写入观看记录。

### 常见入参
- 视频 ID
- 标题
- 剧集 ID
- 封面
- 剧集名称
- 播放地址
- 请求头
- 总时长

### 出参
- `boolean / bool`

---

## 14. Python 专有加密工具

### `aes_encrypt(data, key)`
作用：AES 加密

### `aes_decrypt(encrypted, key)`
作用：AES 解密

### `md5(data)`
作用：计算 MD5

### `base64_encode(data)`
作用：Base64 编码

### `base64_decode(encoded)`
作用：Base64 解码

这些在需要签名、加密参数、对接特殊接口时很有用。
