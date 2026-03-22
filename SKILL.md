---
name: omnibox-spider
description: |-
  Develop, write, debug, and improve OmniBox spider sources (爬虫源). Use when the user asks to write an OmniBox spider script (JS or Python), create a new spider source, fix or debug an existing OmniBox source, convert a third-party video site API into an OmniBox spider, or ask questions about OmniBox spider development (API, SDK, annotations, return formats, etc.). Also triggers on phrases like 写爬虫, OmniBox 爬虫, 爬虫源, spider source, OmniBox 脚本, 帮我写采集站脚本.
---

# OmniBox Spider Skill

> skill_meta
> - last_updated: 2026-03-22 00:33 Asia/Shanghai
> - source_docs: https://omnibox-doc.pages.dev/spider-development/introduction.html
> - source_scope: introduction + getting-started + script-annotation-attributes + api-reference + javascript-sdk + python-sdk
> - sync_note: 每次改动后如需分发，应重新打包 skill 文件

用于开发、编写、调试和改进 OmniBox 爬虫源。优先遵循最新官方文档：
- https://omnibox-doc.pages.dev/spider-development/

## 先做什么

收到 OmniBox 爬虫相关任务时：
1. 先判断是 **JS** 还是 **Python**
2. 再判断是：
   - 新写采集站脚本
   - 修 bug / 对接新站
   - 网盘源 / 推送源
   - 只问 API / SDK / 返回格式
3. 按需读取 references 中对应页面，不要一上来把所有文档都塞进上下文

## 核心规则

### 1) handler 签名统一
所有 handler 优先写成：
- `(params, context)`

不要依赖全局变量读取调用上下文。

### 2) 5 个方法按需实现
| 方法 | 用途 |
|---|---|
| `home` | 首页分类和推荐 |
| `category` | 分类分页列表 |
| `detail` | 视频详情 |
| `search` | 搜索 |
| `play` | 播放地址 |

- 推送型脚本：通常只实现 `detail` + `play`
- 常规影视源：建议实现全部 5 个

### 3) 端差异逻辑统一看 `context.from`
`context.from` 可能是：
- `web`（默认）
- `tvbox`
- `uz`
- `catvod`
- `emby`

如果网页端、TV 端、UZ 端行为不同，统一从这里分支。

### 4) `play` 优先返回推荐格式
优先：
```js
{
  urls: [{ name, url }],
  flag,
  header,
  parse,
  danmaku
}
```

说明：
- `parse=0`：直链
- `parse=1`：需要客户端嗅探，**仅 ok影视 app 有效**
- 其他客户端通常会忽略 `parse=1`

### 5) 错误必须安全返回
- JS：`try/catch`
- Python：`try/except`
- 出错时返回空结构，不要直接崩

### 6) 必须打日志
至少记录：
- 入口参数
- 请求 URL / 关键分支
- 返回数量
- 失败原因

## 常见开发策略

### 普通采集站
1. 先封装请求函数
2. 把站点字段映射为 OmniBox 标准字段
3. `detail` 中构建 `vod_play_sources`
4. `play` 返回直链或解析信息

### 推送源
- 只做 `detail` + `play`
- `detail` 负责把外部链接整理为线路和剧集
- `play` 负责最终播放地址

### 网盘源
优先考虑这些 SDK：
- `getDriveFileList`
- `getDriveVideoPlayInfo`
- `getDriveInfoByShareURL`
- `processScraping` / `getScrapeMetadata`
- `getDanmakuByFileName`

## references 读取指引

### 只想快速上手 / 新建脚本
读：
- `references/getting-started.md`
- `references/js-template.md` 或 `references/py-template.md`

### 想确认返回结构 / handler 规范
读：
- `references/api-reference.md`

### 想确认脚本头注释属性
读：
- `references/script-annotation-attributes.md`

### 写 JavaScript 爬虫
读：
- `references/javascript-sdk.md`
- `references/js-template.md`
- `references/sdk-api.md`

### 写 Python 爬虫
读：
- `references/python-sdk.md`
- `references/py-template.md`
- `references/sdk-api.md`

### 需要理解整体能力与上下文
读：
- `references/introduction.md`

## 重要提醒

- 环境变量：
  - JS 优先 `process.env.KEY`
  - Python 优先 `os.environ.get("KEY")`
- HTTP 请求返回后，通常要手动解析 JSON
- 文档里明确了 `context` 是 Runner 注入，不要自己发明额外调用方式
- `vod_tag: "folder"` 是目录项，不是播放项
- UZ 端有 `search: 1` 这种专用字段，必要时才加

## 交付风格

当用户让你“写 OmniBox 爬虫”：
- 默认直接给出可运行脚本
- 标明需要的环境变量
- 标明注释属性
- 如果站点存在不确定字段，明确写 TODO / fallback
- 必要时附带调试建议，但不要长篇空谈
