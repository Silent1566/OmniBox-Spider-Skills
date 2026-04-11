---
name: omnibox-spider
description: |-
  Develop, write, debug, and improve OmniBox spider sources (爬虫源). Use when the user asks to write an OmniBox spider script (JS or Python), create a new spider source, fix or debug an existing OmniBox source, convert a third-party video site API into an OmniBox spider, or ask questions about OmniBox spider development (API, SDK, annotations, return formats, environment variables, etc.). Also triggers on phrases like 写爬虫, OmniBox 爬虫, 爬虫源, spider source, OmniBox 脚本, 帮我写采集站脚本.
---

# OmniBox Spider Skill

> skill_meta
> - last_updated: 2026-03-27 13:26 Asia/Shanghai
> - source_docs: https://omnibox-doc.pages.dev/spider-development/introduction.html
> - source_scope: introduction + getting-started + script-annotation-attributes + api-reference + sdk + repo env scan
> - sync_note: 每次改动后如需分发，应重新打包 skill 文件

用于开发、编写、调试和改进 OmniBox 爬虫源。优先遵循最新官方文档：
- https://omnibox-doc.pages.dev/spider-development/

## 先做什么

收到 OmniBox 爬虫相关任务时：
1. 先读一次记忆（至少读当天 daily memory；如与历史决策/偏好有关，再补读相关 memory）
2. 再判断是 **JS** 还是 **Python**
3. 再判断是：
   - 新写采集站脚本
   - 修 bug / 对接新站
   - 网盘源 / 推送源
   - 只问 API / SDK / 返回格式
4. 按需读取 references 中对应页面，不要一上来把所有文档都塞进上下文

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

### 7) 环境变量优先复用现有命名（强制）
写新脚本时，先查已有变量，再决定是否新增：
- 普通采集站 API → 优先 `SITE_API`
- 弹幕接口 → 优先 `DANMU_API`
- 网盘优先级 → 优先 `DRIVE_ORDER`
- 网盘类型配置 → 优先 `DRIVE_TYPE_CONFIG`
- 源名称映射 → 优先 `SOURCE_NAMES_CONFIG`
- PanCheck / 盘搜 → 优先 `PANCHECK_*`、`PANSOU_*`
- Emby → 优先 `EMBY_*`
- 小雅 / AList / TVBox → 优先 `XIAOYA_*`、`ALIST_TVBOX_TOKEN`
- 站点地址配置 → 优先沿用已有 `XXX_HOST` 风格

如果确实要新增变量，必须：
1. 命名清晰
2. 与现有变量不冲突
3. 在交付说明里解释为什么不能复用旧变量名
- 解析接口这类可跨源复用的配置，优先抽成全局环境变量（如 `PARSE_APIS`），不要无必要给每个源都造一套 `XXX_PARSE_APIS`。

### 8) 每次改脚本后必须升级 `@version`（强制）
无论是修 bug、加功能、改排序、改刮削、改弹幕、改播放历史、改环境变量逻辑，只要脚本有变化，就必须同步升级 `@version`。

### 8.5) 新建脚本默认必须补 `@downloadURL`（强制）
- 只要是新建并准备落到 OmniBox-Spider 仓库的脚本，默认必须补 `@downloadURL`。
- `@downloadURL` 默认指向仓库主分支最终落地路径的原始文件地址；不要偷懒留空，也不要继续指向工作区调试文件名。
- 若最终仓库文件名不带 `.omnibox`，则 `@downloadURL` 也必须同步使用最终文件名。

默认规则：
- 小修复 / 不改主要行为 → 升 **patch**（如 `1.1.1 -> 1.1.2`）
- 新功能 / 明显增强 / 兼容旧配置 → 升 **minor**（如 `1.1.1 -> 1.2.0`）
- 破坏兼容 / 大改结构 / 需要重配 → 升 **major**（如 `1.1.1 -> 2.0.0`）

以后只要我帮你改 OmniBox 脚本，默认都要：
1. 检查并修改 `@version`
2. 在交付说明中说明为什么升 patch / minor / major

### 9) 每次改脚本后必须做静态检查（强制）
每次修改完成后，至少做一次静态检查，最低要求是脚本在语法/编译层面正常：
- JavaScript → `node --check script.js`
- Python → `python3 -m py_compile script.py`

这一步是交付前必做项。不能只凭肉眼判断“看起来没问题”。

如果条件允许，还应顺手检查：
1. 关键变量是否误删
2. 新逻辑是否插在正确函数中
3. `@version` 是否已同步升级

### 10) 每次改脚本后必须做关键路径回归检查（强制）
静态检查通过后，仍需检查关键运行路径是否被破坏。至少确认：
- `detail()` 返回结构正常
- `vod_play_sources` 在有数据时非空
- `play()` 返回 `urls`
- 关键环境变量没有被误删（如 `SOURCE_NAMES_CONFIG`）

### 11) 默认采用“最小改动”策略（强制）
修改 OmniBox 脚本时，默认遵循：
1. 先复制副本再改，尽量不碰源文件
2. 一次只解决一个明确目标
3. 不顺手重构无关逻辑
4. patch 前先确认插入锚点存在

### 12) 并发优化默认策略
- `play()`：可将播放地址链路与元数据/弹幕链路并行，优先保证主播放链路成功。
- `detail()`：默认不要全量 `Promise.all` 打满，优先使用限并发（建议 4）。
- 元数据、弹幕、刮削增强链路失败时，应尽量降级，不阻塞主链路返回。

### 13) 优先考虑新版 SDK 的缓存能力
- 官方最新版 SDK 已明确提供：`getCache` / `setCache` / `deleteCache`。
- 对高频、重复、跨请求的解析结果，优先考虑缓存，而不是每次都重新打全链路。
- 尤其适合缓存：
  - 搜索结果
  - `getDriveInfoByShareURL()` 的识别结果
  - `detail()` 中的文件列表 / 递归视频文件结果
- 设计缓存时要注意：key 稳定、过期时间合理、失败时安全降级。

### 14) `getVideoMediaInfo` 要晚用，不要太早打在搜索阶段
- 官方最新版 SDK 已明确提供 `getVideoMediaInfo(playUrl, headers)`（需完整版）。
- 它更适合在已经拿到真实可播地址后，用于：
  - 获取总时长 / 文件大小
  - 给 `addPlayHistory` 写入 `totalDuration`
  - 在详情页里补媒体信息
- 不建议在搜索阶段大规模调用，否则很容易拖慢首屏返回。

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

### 需要统一查看官方 SDK 能力（推荐）
读：
- `references/sdk-api.md`

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

### 需要复用仓库既有环境变量
读：
- `references/environment-variables.md`
- `references/environment-variables-cheatsheet.md`

### 需要判断这次该升哪个版本号
读：
- `references/versioning.md`

### 需要确认修改后至少语法/编译正常
读：
- `references/validation.md`

### 需要读取实战经验与防回归规则
读：
- `references/lessons-learned.md`

## 重要提醒

- 官方最新文档在爬虫开发部分已收拢为：`introduction / getting-started / script-annotation-attributes / api-reference / sdk`，后续同步时优先按这 5 个页面校准。
- 环境变量：
  - JS 优先 `process.env.KEY`
  - Python 优先 `os.environ.get("KEY")`
- HTTP 请求返回后，通常要手动解析 JSON
- 文档里明确了 `context` 是 Runner 注入，不要自己发明额外调用方式
- `vod_tag: "folder"` 是目录项，不是播放项
- UZ 端有 `search: 1` 这种专用字段，必要时才加
- `@version / @downloadURL / @indexs / @push / @dependencies` 是当前官方明确说明会被后端实际解析的注释属性；不要凭印象扩展“还有别的也会生效”。
- 磁力 / 电驴类详情分集，默认优先返回“短标题 + 简单 playId”结构；除非确有必要，不要在 `episodes[].playId` 里塞大段 JSON 元数据，否则某些端可能出现按钮空白、详情渲染异常或播放取值异常。

## 交付风格

- 默认优先使用中文：日志、代码注释、变更说明、PR 标题/正文、交付说明等，除非上游项目或外部接口明确要求英文。
- 如果必须混用中英，优先保证对人看的说明部分为中文；仅保留必要的英文标识、变量名、接口字段、提交前缀等。
- 使用 GitHub CLI 创建/编辑 PR 时，正文优先写入 `.md` 临时文件后用 `--body-file` 传入，避免把字面 `\n` 误提交成一整段导致排版混乱。
- 涉及 OmniBox 网页端播放问题时，交付说明和 PR 描述要明确区分：是 `playId` / URL 透传问题、`context.from === "web"` 下 header 不可靠问题，还是嗅探/代理兜底问题，不要笼统写成“修复播放”。

当用户让你“写 OmniBox 爬虫”：
- 默认直接给出可运行脚本
- 标明需要的环境变量
- 优先复用现有环境变量命名
- 标明注释属性
- 如果站点存在不确定字段，明确写 TODO / fallback
- 必要时附带调试建议，但不要长篇空谈
