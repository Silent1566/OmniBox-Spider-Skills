# Python SDK | OmniBox

OmniBox Python SDK 提供请求、日志、环境变量、加密、爬虫源数据、网盘、刮削、弹幕、解析站、观看记录等能力。

## 引入方式

```python
# -*- coding: utf-8 -*-
from spider_runner import OmniBox, run

if __name__ == "__main__":
    run({"home": home, "category": category, "detail": detail, "search": search, "play": play})
```

### 说明
- SDK 通过运行器注入的 `PYTHONPATH` 自动解析
- handler 推荐签名：`(params, context)`
- `context` 是 `dict`
- 不需要自己处理 stdin / stdout，统一由 `run()` 负责

---

## 1. OmniBox.request(url, options)

### 作用
发送 HTTP 请求，用于访问第三方 API、详情页、搜索接口等。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `url` | `str` | 是 | 请求 URL |
| `options` | `dict` | 否 | 请求配置 |
| `options.method` | `str` | 否 | HTTP 方法，默认 `GET` |
| `options.headers` | `dict` | 否 | 请求头 |
| `options.body` | `str \| dict` | 否 | 请求体 |

### 出参
返回 dict：

| 字段 | 类型 | 说明 |
|---|---|---|
| `statusCode` | `int` | HTTP 状态码 |
| `headers` | `dict` | 响应头 |
| `body` | `str` | 响应体，需要手动 `json.loads()` |

### 示例

```python
import json

response = await OmniBox.request("https://api.example.com/data", {
    "method": "GET",
    "headers": {"User-Agent": "Mozilla/5.0"},
})

if response.get("statusCode") != 200:
    raise RuntimeError(f"HTTP {response.get('statusCode')}: {response.get('body', '')}")

data = json.loads(response.get("body", "{}"))
```

---

## 2. OmniBox.log(level, message)

### 作用
输出日志，便于后台调试脚本执行过程。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `level` | `str` | 是 | `info` / `warn` / `error` |
| `message` | `str` | 是 | 日志内容 |

### 出参
- `None`

### 示例

```python
await OmniBox.log("info", "开始执行搜索")
await OmniBox.log("warn", "某字段缺失，使用默认值")
await OmniBox.log("error", f"处理失败: {error}")
```

---

## 3. OmniBox.get_env(name)

### 作用
读取环境变量。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `name` | `str` | 是 | 环境变量名 |

### 出参
- `str`

### 示例

```python
api_key = await OmniBox.get_env("API_KEY")
```

### 推荐写法
优先直接用：

```python
import os
api_key = os.environ.get("API_KEY", "")
```

---

## 4. OmniBox.get_source_favorite_tags()

### 作用
获取当前爬虫源的收藏标签列表。

### 入参
- 无显式参数
- 依赖 `context.sourceId`

### 出参
- `list[str]`

### 示例

```python
tags = await OmniBox.get_source_favorite_tags()
```

---

## 5. OmniBox.get_source_category_data(category_type, page=1, page_size=20)

### 作用
获取当前爬虫源的分类数据，如历史、收藏、标签。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `category_type` | `str` | 是 | `history` / `favorite` / `tag` |
| `page` | `int` | 否 | 页码，默认 1 |
| `page_size` | `int` | 否 | 每页数量，默认 20 |

### 出参
返回 dict：

| 字段 | 类型 | 说明 |
|---|---|---|
| `list` | `list` | 数据列表 |
| `total` | `int` | 总数量 |
| `pageCount` | `int` | 总页数 |

### 示例

```python
history = await OmniBox.get_source_category_data("history", 1, 20)
favorite = await OmniBox.get_source_category_data("favorite", 1, 20)
```

---

## 6. OmniBox.aes_encrypt(data, key)

### 作用
AES 加密。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `data` | `str` | 是 | 待加密内容 |
| `key` | `str` | 是 | 密钥 |

### 出参
- `str`

### 示例

```python
encrypted = await OmniBox.aes_encrypt("data", "key")
```

---

## 7. OmniBox.aes_decrypt(encrypted, key)

### 作用
AES 解密。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `encrypted` | `str` | 是 | 密文 |
| `key` | `str` | 是 | 密钥 |

### 出参
- `str`

### 示例

```python
decrypted = await OmniBox.aes_decrypt(encrypted, "key")
```

---

## 8. OmniBox.md5(data)

### 作用
计算 MD5 哈希。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `data` | `str` | 是 | 原始数据 |

### 出参
- `str`

### 示例

```python
hash_value = await OmniBox.md5("data")
```

---

## 9. OmniBox.base64_encode(data)

### 作用
Base64 编码。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `data` | `str` | 是 | 原始字符串 |

### 出参
- `str`

### 示例

```python
encoded = await OmniBox.base64_encode("data")
```

---

## 10. OmniBox.base64_decode(encoded)

### 作用
Base64 解码。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `encoded` | `str` | 是 | Base64 字符串 |

### 出参
- `str`

### 示例

```python
decoded = await OmniBox.base64_decode(encoded)
```

---

## 11. OmniBox.get_drive_file_list(share_url, pdir_fid="0")

### 作用
获取网盘文件列表。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `share_url` | `str` | 是 | 网盘分享链接 |
| `pdir_fid` | `str` | 否 | 父目录 ID，默认 `"0"` |

### 出参
返回 dict：

| 字段 | 类型 | 说明 |
|---|---|---|
| `files` | `list` | 文件列表 |
| `total` | `int` | 总数量 |
| `has_more` | `bool` | 是否还有更多 |

### 示例

```python
file_list = await OmniBox.get_drive_file_list("https://pan.example.com/s/xxx", "0")
```

---

## 12. OmniBox.get_drive_video_play_info(share_url, fid, flag="")

### 作用
获取网盘视频播放信息。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `share_url` | `str` | 是 | 网盘分享链接 |
| `fid` | `str` | 是 | 文件 ID |
| `flag` | `str` | 否 | 播放方式标识 |

### 出参
返回 dict：

| 字段 | 类型 | 说明 |
|---|---|---|
| `url` | `str` | 播放地址 |
| `header` | `dict` | 请求头 |
| `danmaku` | `list` | 弹幕列表 |

### 示例

```python
play_info = await OmniBox.get_drive_video_play_info("https://pan.example.com/s/xxx", "file_id")
```

---

## 13. OmniBox.get_drive_info_by_share_url(share_url)

### 作用
根据分享链接识别网盘类型、名称、图标等。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `share_url` | `str` | 是 | 网盘分享链接 |

### 出参
返回 dict：

| 字段 | 类型 | 说明 |
|---|---|---|
| `driveType` | `str` | 网盘类型 |
| `displayName` | `str` | 显示名称 |
| `iconPath` | `str` | 图标路径 |
| `iconUrl` | `str` | 图标地址 |

### 示例

```python
info = await OmniBox.get_drive_info_by_share_url("https://pan.example.com/s/xxx")
```

---

## 14. OmniBox.process_scraping(resource_id, keyword, resource_name, video_files)

### 作用
执行刮削流程，生成元数据。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `resource_id` | `str` | 是 | 资源唯一标识 |
| `keyword` | `str` | 否 | 搜索关键词 |
| `resource_name` | `str` | 否 | 资源名称 |
| `video_files` | `list` | 是 | 视频文件列表 |

### 出参
- `dict`

### 示例

```python
result = await OmniBox.process_scraping("resource_123", "凡人修仙传", "凡人修仙传", video_files)
```

---

## 15. OmniBox.get_scrape_metadata(resource_id)

### 作用
获取刮削后的元数据。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `resource_id` | `str` | 是 | 资源唯一标识 |

### 出参
返回 dict，常见字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `scrapeData` | `dict \| None` | 刮削结果 |
| `videoMappings` | `list \| None` | 文件映射信息 |
| `scrapeType` | `str \| None` | 刮削类型 |

### 示例

```python
metadata = await OmniBox.get_scrape_metadata("resource_123")
```

---

## 16. OmniBox.sniff_video(url, headers=None)

### 作用
加载播放页并捕获真实视频地址。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `url` | `str` | 是 | 待嗅探页面 URL |
| `headers` | `dict` | 否 | 请求头 |

### 出参
返回 dict：

| 字段 | 类型 | 说明 |
|---|---|---|
| `url` | `str` | 嗅探出的真实视频地址 |
| `header` | `dict` | 播放请求头 |

### 示例

```python
result = await OmniBox.sniff_video("https://example.com/play?id=123", {"Referer": "https://example.com/"})
```

### 注意
- 需要完整版运行环境（Playwright / Chromium）

---

## 17. OmniBox.get_danmaku_by_file_name(file_name)

### 作用
按文件名匹配弹幕源。

### 入参

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `file_name` | `str` | 是 | 文件名 |

### 出参
- `list[dict]`
- 每项通常为 `{ name, url }`

### 示例

```python
danmaku = await OmniBox.get_danmaku_by_file_name("凡人修仙传.S01E01.mp4")
```

---

## 18. OmniBox.get_analyze_sites()

### 作用
获取已配置解析站列表。

### 入参
- 无

### 出参
- `list[dict]`
- 每项结构：

| 字段 | 类型 | 说明 |
|---|---|---|
| `name` | `str` | 站点名称 |
| `url` | `str` | 接口地址 |
| `type` | `int` | `0=Web`，`1=JSON` |

### 示例

```python
sites = await OmniBox.get_analyze_sites()
web_sites = [s for s in sites if s.get("type") == 0]
```

---

## 19. OmniBox.add_play_history(...)

### 作用
添加观看记录。

### 入参
常见参数包括：
- `vod_id`
- `title`
- `episode`
- `pic`
- `episode_number`
- `episode_name`
- `play_url`
- `play_header`
- `total_duration`

### 出参
- `bool`
- 表示是否成功新增

### 示例

```python
added = await OmniBox.add_play_history(
    vod_id="video_123",
    title="示例视频",
    episode="ep_1",
    play_url="https://example.com/video.m3u8",
    play_header={"Referer": "https://example.com/"},
)
```

---

## 通用开发建议

### 推荐开发习惯
1. 统一封装请求函数
2. 参数统一 `params.get(...)`
3. 统一用 `response.get("statusCode")`
4. 所有 handler 用 `try/except`
5. 关键步骤统一打日志
6. 环境变量优先 `os.environ.get()`
7. 最后统一 `run({...})`

### 常见空返回

#### home 失败
```python
return {"class": [], "list": []}
```

#### category / search 失败
```python
return {"page": 1, "pagecount": 0, "total": 0, "list": []}
```

#### detail 失败
```python
return {"list": []}
```

#### play 失败
```python
return {"url": "", "flag": params.get("flag", ""), "header": {}}
```
