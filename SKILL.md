---
name: omnibox-spider
description: |-
  Develop, write, debug, and improve OmniBox spider sources (爬虫源). Use when the user asks to write an OmniBox spider script (JS or Python), create a new spider source, fix or debug an existing OmniBox source, convert a third-party video site API into an OmniBox spider, or ask questions about OmniBox spider development (API, SDK, annotations, return formats, environment variables, etc.). Also triggers on phrases like 写爬虫, OmniBox 爬虫, 爬虫源, spider source, OmniBox 脚本, 帮我写采集站脚本.
---

# OmniBox Spider Skill

> skill_meta
> - last_updated: 2026-04-06 23:30 Asia/Shanghai
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

> **文件修改纪律**：在处理用户请求时，仅对用户明确指定的目标脚本文件进行修改。如果在同一会话中意外修改了其它脚本，请立即使用 `git checkout -- <other_file>` 恢复原始状态并在回复中说明已回滚。此规则防止误编辑导致 PR 内容混乱，也满足用户对精准、最小化改动的期待。

> **新增**：在本地调试时，如果 `omnibox_sdk` 或 `spider_runner` 模块缺失，脚本会自动使用 fallback logger/runner 防止因模块未找到而直接崩溃。该 fallback 仅用于本地开发或单元测试，生产环境仍需确保依赖完整。

> **示例**（已在 `歪比巴卜.js` 中实现）：
>
> ```js
> let OmniBox;
> try { OmniBox = require('omnibox_sdk'); }
> catch (_) { OmniBox = { log(l,m){ console.log(`[${l}] ${m}`); } };
> let runner;
> try { runner = require('spider_runner'); }
> catch (_) { runner = { run(){ } };
> ```
>
> 该 fallback 保证所有 `OmniBox.log` 调用仍能输出日志，且脚本能够正常加载并执行 `node --check`。

- 记录入口参数、关键请求 URL、返回数量及错误原因。
- 日志等级使用 `info`、`error`，统一前缀 `[wbbb][<method>]`（示例见 `detail`）
- 在 `detail`、`search`、`play` 等关键路径均已加入日志。
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

### 8) 每次改脚本后必须升级 `@version`（强制）
无论是修 bug、加功能、改排序、改刮削、改弹幕、改播放历史、改环境变量逻辑，只要脚本有变化，就必须同步升级 `@version`。

特别注意：
- 不只是“功能逻辑变化”才要升版本；**头注释里会影响分发 / 更新识别 / 下载定位的字段变化** 也要升版本。
- 尤其是改动 `@downloadURL` 时，默认也必须同步 bump `@version`，不要只改注解不改版本号。

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
5. 如果准备从 `openclaw` 发 PR，先确认目标文件已经被 `git add + commit` 到分支里；**工作区里的未跟踪文件不等于分支里已有该改动**

### 12) 并发优化默认策略
- `play()`：可将播放地址链路与元数据/弹幕链路并行，优先保证主播放链路成功。
- `detail()`：默认不要全量 `Promise.all` 打满，优先使用限并发（建议 4）。
- 元数据、弹幕、刮削增强链路失败时，应尽量降级，不阻塞主链路返回。
- 网盘增强链路推荐固定模式：`const metadataPromise = (...)(); const [playResult, metadataResult] = await Promise.allSettled([主播放链路, metadataPromise])`。其中主播放链路如 `OmniBox.getDriveVideoPlayInfo(...)` / 站点 `requestPlayApi(...)`，增强链路里再做 `getScrapeMetadata()`、`videoMappings` 命中、`getDanmakuByFileName()`。
- 播放记录默认不要 `await OmniBox.addPlayHistory(...)` 阻塞启播；改用 `OmniBox.addPlayHistory(payload).then(...).catch(...)`，并在回调中分别打印：添加成功、记录已存在、添加失败。
- 普通采集/API 源如果要同时支持刮削、弹幕和播放记录，默认采用这条组合链路：
  1. `detail()` 先把站点分集一次性整理成 `scrapeCandidates`（如 `{ file_id, file_name }`），只调用一次 `OmniBox.processScraping(videoId, vodName, vodName, scrapeCandidates)`；
  2. 紧接着读取一次 `OmniBox.getScrapeMetadata(videoId)`，把 `scrapeData` 与 `videoMappings` 统一回填到 `vod_name / vod_pic / vod_content / 分集名`，不要在每集上重复刮削；
  3. `play()` 里把“获取播放地址主链路”和“读取刮削元数据 + 生成弹幕匹配文件名 + `getDanmakuByFileName()` 辅链路”并行跑，推荐 `Promise.allSettled([playInfoPromise, metadataPromise])`；
  4. 分集 `playId` 默认优先带上 `vodId / fileId(nid) / episodeName / title` 这类轻量上下文，方便 `play()` 直接命中 `videoMappings`、写播放记录和补日志，同时可按需兼容旧的 `主ID@子ID` 格式。
  5. 如果站点原始 `playId` 本身是完整播放页 URL，而不是简单 `nid` / 子集 ID，推荐把透传元数据拼成 **`真实播放URL|||base64(meta)`**：`meta` 至少带 `fid / vod title / episode title / line name / episode index`。这样 `play()` 可以先拆出真实 URL 走主播放链路，再复用 `meta.fid` 回查刮削映射和写播放记录，不必重新从播放页反推分集身份。
- 这类采集源接刮削映射时，优先先尝试用站点分集自身稳定字段（如 `nid` / `episodeId`）直接作为 `videoFiles[].file_id` 与后续 `videoMappings[].fileId` 的匹配键；如果后续弹幕或回填对不上，先核对 `detail()` 写入的 `file_id` 和 `play()` 里匹配用的键是否完全一致，再决定是否改匹配逻辑。
- 如果采集站没有天然的稳定分集 ID，而是“详情页里只有线路名 + 分集序号 + 播放页 URL”，可退而求其次使用 **`detailUrl#lineName#episodeIndex`** 这类组合键做 `fid/file_id`，同时在 `detail()` 和 `play()` 两端保持完全同构。此类组合键虽然不如站点原生 `nid` 稳定，但足够支撑 `processScraping()`、分集改名、弹幕匹配和播放记录这一整条增强链路；关键不是键长什么样，而是 `scrapeCandidates.file_id`、透传 `meta.fid`、以及 `videoMappings[].fileId` 的预期命中值必须一致。
- 再细一层：如果站点详情页拿到的并不是“完整播放 URL”，而只是 `play/xxx.html` 里的 **短 playId / slug**，也仍然适合走同一套增强链路。更稳妥的做法是：`detail()` 里给每集构造 `rawPlayId|||base64(meta)`，其中 `meta` 至少带 `sid=主视频ID`、`fid=主视频ID#lineName#episodeIndex`、`v=片名`、`e=分集名`；对外返回给宿主的 `vod_play_sources` 只保留 `{name, playId}`，同时额外保留一份内部 `_play_sources_for_scrape`（含 `_fid/_rawName`）供刮削回填使用。这样 `play()` 可以先拆出短 `playId` 走主播放页链路，再复用 `meta.fid/meta.sid` 去命中刮削映射、匹配弹幕和异步写播放记录。

### 13) 优先考虑新版 SDK 的缓存能力
- 官方最新版 SDK 已明确提供：`getCache` / `setCache` / `deleteCache`。
- 对高频、重复、跨请求的解析结果，优先考虑缓存，而不是每次都重新打全链路。
- 尤其适合缓存：
  - 搜索结果
  - `getDriveInfoByShareURL()` 的识别结果
  - `detail()` 中的文件列表 / 递归视频文件结果
  - 同一资源在 **分组页 / 分享列表页 / 详情预热** 会重复命中的上游资源列表接口（例如 `/resources/{type}/{tmdbId}` 这类“先取全量再本地过滤”的接口）
- 如果分组页和分享列表页都依赖同一份上游资源清单，不要在两个 handler 分支里各打一次真实请求；更稳妥的默认做法是抽一个 `getXxxResourcesCached(mainKey...)`，按主资源键（如 `mediaType + tmdbId`）缓存整份列表，再在本地按 `pan_type / line / source` 过滤复用。这样既能减少重复请求，也能显著降低触发上游 429 的概率。
- 设计缓存时要注意：key 稳定、过期时间合理、失败时安全降级。
- 如果缓存 key 直接拼接完整分享链接、标题、筛选 JSON 或其它长字符串，容易触发宿主/SDK 的 key 长度上限（实战里出现过 **256 字节限制**）。更稳妥的默认做法是：保留可读前缀（如 `source:detail` / `source:pan`），再对长入参做 `md5/sha1` 短哈希，最终形成 **`前缀:短哈希`** 结构，而不是把原始长 URL 全塞进 key。
- 进一步的兜底建议：不要只在“某几个 buildCacheKey() 调用点”做缩短；更稳妥的是在脚本统一缓存包装层（如 `getCache/setCache/deleteCache` 的 helper）里，再做一次 **最终 SDK 入参级别** 的 key 长度检查与哈希归一化。这样即使某条业务链路漏传了原始长 key，也不会把超长 key 直接喂给宿主。
- 这层兜底触发时，建议只打一次轻量诊断日志（例如记录 `originalKeyPreview / originalKeyBytes / normalizedKey`），方便后续定位到底是哪条缓存链路仍在生成超长 key，同时避免刷屏。
- 对同一脚本内部的缓存封装，默认可拆成 `getCachedText()` / `getCachedJson()` 两类：前者缓存 HTML / 文本，后者缓存 JSON 结构；读取缓存失败、JSON 解析失败、写缓存失败时都应只记日志并安全回退，不要让缓存层反过来阻断主链路。
- HTML 请求链路建议统一从一个可选 `ttl` 的 `fetchHtml()` 之类入口接缓存，这样 `home/category/search/detail` 能按页面类型分别给 TTL，而不是每个 handler 各写一套缓存分支。
- 分类筛选页如果需要多轮抓页面来解析 `filters -> 最终 URL`，也值得单独缓存“筛选参数组合 → 最终分类 URL”的结果，避免每次翻页/切筛选都重复走整套筛选解析。
- 对网盘分享这类高成本链路，推荐采用 **进程内 Map + SDK 持久缓存** 双层缓存：Map 负责当前运行期快速命中，SDK 缓存负责跨请求复用；缓存对象可直接保存 `driveInfo + 根目录文件 + 递归展开后的视频文件列表`。
- 播放页缓存默认比详情页更保守：若未确认站点播放页足够稳定，不要一上来给 `play()` 上长 TTL；优先先缓存 `home/category/search/detail` 与网盘文件递归这些稳定收益更高的链路，再视站点行为决定是否给播放解析加短 TTL。
- 如果缓存逻辑已覆盖 `home/category/search/detail`、分类筛选 URL、以及 `loadPanFiles()` 这类网盘递归入口，交付说明里建议明确区分“哪些链路已缓存、TTL 多久、哪些高敏感链路（如 play）暂未缓存及原因”，方便后续排查脏缓存或性能问题。

### 14) `getVideoMediaInfo` 要晚用，不要太早打在搜索阶段
- 官方最新版 SDK 已明确提供 `getVideoMediaInfo(playUrl, headers)`（需完整版）。
- 它更适合在已经拿到真实可播地址后，用于：
  - 获取总时长 / 文件大小
  - 给 `addPlayHistory` 写入 `totalDuration`
  - 在详情页里补媒体信息
- 不建议在搜索阶段大规模调用，否则很容易拖慢首屏返回。

### 16) CF 盾 / FlareSolverr 绕过模式（强制条件：FLARESOLVERR_URL 有值才启用）

当站点被 Cloudflare 盾拦截时，可集成 FlareSolverr 实现自动绕过。**核心原则：仅当 `FLARESOLVERR_URL` 环境变量有配置时才启用该逻辑**，否则不应影响原有请求流程。

### 16.5) 评估或同步 `OmniBox-Spider-Skills` 镜像仓库时，优先比“实际内容差异”，不要只看时间戳

- `OmniBox-Spider-Skills` 属于 **技能镜像/分发仓库**，不保证永远与当前本地在用技能完全同步。
- 当用户问“这个技能仓库要不要升级 / 近期经验有没有补进去”时，优先直接对比：
  - 本地在用技能：`~/.hermes/skills/openclaw-imports/omnibox-spider/SKILL.md`
  - 镜像仓库：如 `/root/.hermes/workspace/OmniBox-Spider-Skills/SKILL.md`
- 判断是否落后时，**优先信内容 diff**，不要只看 `last_updated`、commit 标题或 README 描述；时间新不等于内容新，实战里镜像仓库可能比本地工作技能“更新时间更晚但内容更少”。
- 对比时重点先看 4 类高价值内容是否缺失：
  1. 最近实战新增的通用规则（如缓存 key 规范化、`@downloadURL`/`@version` 联动）
  2. 近期高频调试链路（如刮削/弹幕/播放记录组合链路）
  3. 新增的跨站点能力沉淀（如 FlareSolverr 页面级缓存方案）
  4. `references/` 目录里的专题经验文档是否一并同步
- 如果镜像仓库明显比本地技能“短很多 / 少专题 references / 缺近期规则”，默认结论应是：**值得做一轮系统同步**，而不是只补一两句零散说明。
- 做这类检查时，可把它拆成两步：
  1. 先比较主 `SKILL.md` 的章节与关键小节差异；
  2. 再比较 `references/` 是否缺失本地已在用的专题文档（如网盘、筛选、播放、CF、WordPress 等）。

#### 🔴 重要：只缓存 cookie 然后自建请求重试通常会 403（已证伪的模式）
实战中已多次验证：**FlareSolverr 返回的 `cf_clearance` cookie 往往与请求上下文（UA、headers 顺序、IP、路径）强绑定**。单独拿 cookie 用自己的 `axios`/`fetch` 重试同一 URL，即使补全 UA 和 headers，也常常返回 403。

**结论：不要只缓存 cookie 再自建请求重试。** 必须采用**页面级缓存方案**。

---

#### ✅ 推荐方案：直接使用 FlareSolverr 返回的完整页面 + 页面级缓存

FlareSolverr 的 `solution.response` 已经是绕过 CF 盾后的最终 HTML，**直接返回即可**，不需要再自己请求一次。同时将返回结果按 URL 缓存，后续相同 URL 的请求直接命中缓存。

#### 环境变量命名规范
- 优先定义站点专用变量（如 `XXX_FLARESOLVERR_URL`），再回退通用 `FLARESOLVERR_URL`
- 站点专用变量示例：`WOOG_FLARESOLVERR_URL` / `PPNIX_FLARESOLVERR_URL`
- 页面缓存 key 前缀：`WOOG_CF_PAGE_CACHE_KEY_PREFIX`（默认 `wogg:page:`）
- 页面缓存 TTL：`WOOG_CF_PAGE_CACHE_TTL`（默认 3600 秒）
- cookie 缓存 key：`WOOG_CF_CACHE_KEY` → `wogg:cf_clearance`

#### 核心函数集合（参考 `玩偶.js` v1.2.0+ 模式）
1. **`cookiesArrayToString(cookies)`** — 将 FlareSolverr 返回的 cookies 数组 → `name=value;...` 字符串
2. **`getCachedCfCookie()`** — 优先读环境变量硬编码 `XXX_CF_COOKIE`，否则读 SDK 缓存（只用做请求头附加，不做重试依据）
3. **`setCachedCfCookie(cookie)`** — 写入 SDK 缓存，TTL 默认 6 小时
4. **`fetchCfClearanceWithFlareSolverr(targetUrl)`** — **返回完整响应对象 `{ cookie, body, statusCode, headers, ua }`**
   - payload: `{ cmd: "request.get", url: targetUrl, maxTimeout, session? }`
   - 必须校验 `res.data.status === "ok"` 且存在 `cf_clearance` cookie
   - `body` 直接取 `solution.response`（绕过后的完整 HTML）
   - `statusCode` 取 `solution.status`
   - `headers` 取 `solution.headers`（用于后续可能的 UA/headers 同步）
5. **`ensureCfCookie(forceRefresh, targetUrl)`** — 返回 `{ cookie, flareResult }`，其中 `flareResult` 包含完整响应
   - `XXX_CF_AUTO !== "0"` 时自动触发，设为 `"0"` 可关闭
6. **`getCachedPage(url)`** — 按 URL 读页面缓存：`OmniBox.getCache(prefix + url)`，返回 `{ body, statusCode, headers }`
7. **`setCachedPage(url, body, statusCode, headers)`** — 按 URL 写页面缓存，TTL 通过 `WOOG_CF_PAGE_CACHE_TTL` 控制

#### 集成到 httpRequest 的推荐模式（核心逻辑）
```
检测 CF 盾 → 优先读 getCachedPage(url)
  ├─ 命中 → 直接返回缓存的 { body, statusCode, headers }
  └─ 未命中 → 调 fetchCfClearanceWithFlareSolverr(url)
                ├─ 返回 body → 调 setCachedPage(url, ...) → 直接返回
                └─ 失败 → 记日志，fall through 继续尝试下一个域名
```

代码示例：
```js
if (isBlockedHtml(body) && WOOG_FLARESOLVERR_URL) {
  const cached = await getCachedPage(url);
  if (cached) {
    return { statusCode: cached.statusCode, body: cached.body, headers: cached.headers, _cfSource: "page_cache" };
  }
  const result = await fetchCfClearanceWithFlareSolverr(url);
  if (result && result.body) {
    await setCachedPage(url, result.body, result.statusCode, result.headers);
    return { statusCode: result.statusCode, body: result.body, headers: result.headers, _cfSource: "flaresolverr" };
  }
}
```

⚠️ **不要使用 `_cfRetried` / `_cfCookie` 标志做 cookie 重试 fallback** — 已被证实在多数站点无效。

#### 必须添加的诊断日志
- FlareSolverr 返回结果：`status, cookies数量, cookie预览(前80字符), ua, body长度, message, 执行耗时`
- 命中页面缓存：`[cf] 命中页面缓存: {url}, body长度=...`
- FlareSolverr 返回页面：`[cf] FlareSolverr 已直接返回页面内容: {url}, status=..., body长度=...`
- 写入缓存：`[cf] 已缓存页面: {url}, TTL=...`
- 绕过失败：`[cf] FlareSolverr 绕过失败: ...`

#### 多域名容灾 + CF 绕过的组合逻辑
- 先尝试当前域名
- 如果被 CF 盾拦截 → 查页面缓存
  - 命中 → 返回，不再尝试其他域名
  - 未命中 → FlareSolverr 获取 → 缓存后返回
- 如果 FlareSolverr 也失败 → `requestWithFailover` 继续尝试下一个域名
- 注意：FlareSolverr 失败后不要阻塞容灾流程（`catch` 中只记日志）

#### 关键 Pitfall
- **不要只缓存 cookie 再自建请求重试**：FlareSolverr 返回的 `cf_clearance` 往往与 UA、请求头、IP、路径强绑定，单独使用会 403。必须缓存 FlareSolverr 返回的完整页面内容。
- **`FLARESOLVERR_URL` 为空时必须零成本**：所有 CF 相关的 await/缓存读取都不应发生，否则会拖慢无 CF 盾站点的请求。
- **页面缓存 key 用完整 URL**：不要用固定 key，必须按 URL 分区缓存，避免不同页面返回相同内容。注意 URL 可能很长，必要时加前缀后哈希。
- **页面 TTL 不宜过长**：默认 3600 秒（1 小时），静态列表页可适当延长，动态搜索结果/筛选页应缩短或跳过缓存。
- **缓存失败不阻塞主路径**：`setCachedPage` 异常时只记日志，不要影响正常返回。
- **不要写死 `cf_clearance` 的 domain 检查**：不同站点的 `cf_clearance` 域不同。FlareSolverr 返回的 cookie 中查找 `cf_clearance` 时，不要加域名过滤，除非明确是针对单一站点的脚本。

参考实现：`影视/网盘/玩偶.js`（v1.2.0+；页面缓存方案）、`影视/采集/PPnix.js`（v1.0；旧 cookie 重试方案，仅供参考）

---

## 15) 调试日志与本地运行时兜底

当用户反馈“详情页没有数据，也没有相关日志”时，不要只补业务解析逻辑；优先确认脚本是否在入口阶段就因为运行时依赖缺失而崩掉。常见本地/调试环境问题是 `require('omnibox_sdk')` 或 `require('spider_runner')` 找不到，导致 handler 根本没被调用，用户侧表现为无日志或空数据。

推荐处理方式：
- 每个 handler 入口默认记录 `params`。
- 每个网络请求前记录关键 URL / ID。
- 每个列表/详情解析后记录返回数量。
- `catch` 中必须记录中文错误日志，并安全返回空结构。
- 若需要本地直接 `node require('./脚本.js')` 调试，可为 SDK/runner 加最小 fallback，避免模块缺失直接中断：

```js
let OmniBox;
try {
  OmniBox = require('omnibox_sdk');
} catch (_) {
  OmniBox = {
    log(level, message) {
      console.log(`[${level}] ${message}`);
    },
  };
}

let runner;
try {
  runner = require('spider_runner');
} catch (_) {
  runner = { run() {} };
}
```

注意：fallback 只用于调试与提升可观测性；生产环境仍应优先使用正式 `omnibox_sdk` / `spider_runner`。如果加了日志或 fallback，也属于脚本行为改动，仍要同步 bump `@version`、执行 `node --check`，并提交/推送到 PR 分支。

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
- `references/wordpress-taxonomy-category-filters.md`

### 需要读取这次沉淀的站点实战教训
读：
- `references/libvio-live-filters-and-pan-episode-fallback.md`
- `references/detail-missing-playlines.md`（detail 返回无线路 / 前端无线路展示时优先读取）
- `references/wbbb-parse-runtime-and-tvbox-play.md`（解析站 `api.php` 已 200 但 TVBox 仍 sniff 失败、或 resolved URL 仍停留在 `/player/?url=...` 时优先读取）

### 需要避免把其它宿主格式误提交成 OmniBox 源
读：
- `references/om-format-correction.md`

并先做 3 个判断：
1. 这是 **OmniBox 五方法 spider**，还是 **drpy-node `var rule`**
2. 当前卡点是 **建源 / 结构解析 / 播放链 / 仓库交付** 哪一层
3. 这次目标是先做 **最小可用版本**，还是已经进入 **上传前收尾**

## 重要提醒

- 官方最新文档在爬虫开发部分已收拢为：`introduction / getting-started / script-annotation-attributes / api-reference / sdk`，后续同步时优先按这 5 个页面校准。
- 环境变量：
  - JS 优先 `process.env.KEY`
  - Python 优先 `os.environ.get("KEY")`
- HTTP 请求返回后，通常要手动解析 JSON
- 文档里明确了 `context` 是 Runner 注入，不要自己发明额外调用方式
- `vod_tag: "folder"` 是目录项，不是播放项
- 分组型网盘源里要特别区分“网盘类型分组”和“具体分享项”：前者可用 `vod_tag: "folder"`，后者如果预期点击后直接进入 `detail()` 生成剧集列表，就不要再标成 `folder`；优先改成 `vod_tag: "video"` 或等价的详情项语义，否则宿主很可能把“具体分享”继续当目录，多出一层文件夹页。
- 如果日志已经证明点击某个分享时 `parsePanFolderVodId()` 能解析出 `isShareEntry: true`，且 `detail()` 内也已经拿到了 `videoCount > 0`，那么问题通常不在文件递归，而在列表项语义（如 `vod_tag`）或宿主点击链路。
- 分组型网盘源的详情标题默认不要主动拼 ` [百度网盘] / [夸克] / [XX网盘] ` 这类后缀；`vod_name` 应优先保持原始剧名，必要时只在 `vod_remarks`、线路名或分组名里体现网盘类型。
- 网盘分组下的“具体分享列表”不要全部复用同一个剧名；应优先读取分享自身的真实名称（例如 `fileList.displayName` / `display_name`，其次根目录唯一文件夹/文件名），再回退到剧名。这样同一网盘下多条分享才能被区分，用户也更容易判断哪条是想要的版本。
- 如果官网同一条网盘分享在 HTML 里同时以 `href`、重复 `href`、`data-clipboard-text`、或带 `dn/displayName/filename/title` 参数的形式出现，默认不要直接按原始字符串计数；应先做 **URL 规范化后再去重**。常见做法：保留真正决定分享身份的部分（如百度保留 `pwd`，夸克/UC 去掉伪参数 `dn`），删除只影响展示的名称参数，再按规范化 URL 去重；否则会出现“官网只有 1 条，脚本却显示多条结果”。
- 处理分享名时，优先级建议为：`<a title>` / `<a>` 文本 / 链接参数中的 `dn|displayName|filename|title` → 网盘根目录展示名 → 根目录唯一文件/文件夹名 → `分享N`。同时要过滤明显不是资源名的伪标题，例如 `百度网盘`、`夸克网盘`、`网盘下载`、`baidu`、`quark`，避免页面标题退化成平台名。
- 处理站点“网盘下载”区时，如果同一分享链接会在一行里以标题链接、下载按钮、复制按钮、`href`、`data-clipboard-text` 等多种形式重复出现，不要只做全页扫链接后“先到先得”地记分享名；更稳的默认做法是：**先按下载区单行（如 `li.down-list2`）抽取行级元数据**，优先取该行标题 `<a title>` / 标题文本 / 行文本作为 `nameHint`，再把这一行里出现的所有网盘链接都绑定到同一个 `nameHint`，最后再做 URL 规范化去重。这样能避免真实标题被前后截断的锚文本或平台名按钮文案覆盖。
- 如果同一规范化分享 URL 在多个位置提取到不同候选标题，不要只保留第一个非空值；应对候选标题做“择优覆盖”。默认可优先选择：更长、更完整、包含 `4K/HDR/全集/完结/更新/国语/中字/原盘/高码率` 等资源信息的标题；同时清洗掉尾部时间文案（如“今天/昨天/刚刚”）、`提取码`、多余分隔符，并继续过滤 `百度网盘`、`夸克网盘`、`网盘下载`、`baidu`、`quark` 这类伪标题。
- 分组型网盘源如果已经在缓存构建阶段做过 PanCheck，默认不要再在 `ensureDetailCache()`、分组页、列表页、详情页重复触发同一批 share links 的二次校验。更稳妥的默认策略是：**仅在首次构建 grouped/detail 缓存时做一次 PanCheck 并回写过滤后的 grouped**，后续链路优先复用缓存；只有拿到明确证据表明缓存可能是旧脏数据（如历史缓存未过滤、配置刚开启 PanCheck、或需要强制刷新）时，才再加“进入分组前补过滤”的兜底逻辑。
- 命中刮削缓存但详情改名 / 播放映射仍失败时，优先怀疑 **历史脏缓存** 而不是继续盲改匹配逻辑。尤其是曾经改过 `videoMappings[].fileId` 拼装格式（如误用 `#`，正确应为 `{shareURL}|{fileId}`）后，要先检查缓存里的 `mapping.fileId` 预览，再决定是否重跑刮削。
- 对 OmniBox 网盘刮削链路，`videoMappings[].fileId` 默认按 **`{shareURL}|{fileId}`** 匹配；如果详情页构建、剧集改名、播放时弹幕/播放记录映射都失效，先核对脚本内 `_fid`、播放阶段 `formattedFileId`、以及缓存内 `mapping.fileId` 三者是否完全一致，不要混用 `#`。
- 如果用户明确要求“只要有任一线路就要刮削”，不要把 `detail()` 的刮削输入只限定为站内采集线路。更稳妥的默认做法是：对外 `vod_play_sources` 继续按宿主展示需求返回，但内部额外构造一份 `_play_sources_for_scrape`，把 **采集线路 + 网盘线路** 都整理进去；其中网盘分集要补齐 `_fid/_rawName`，`_fid` 优先复用 **`{shareURL}|{fileId}`**，必要时再透传 `_scrapeMeta`（如 `sid/fid/i`）供后续改名、弹幕和播放记录统一命中。这样即使站内采集线路为空，只要网盘线路存在，`processScraping()` 也仍能继续跑。
- 遇到“detail 成功但完全没有进入刮削”这类问题时，建议把原因拆开打日志，而不是只留一个总括判断。至少区分：`vod 为空`、`无可用于刮削的线路`、`宿主未提供 processScraping/getScrapeMetadata`、`scrapeCandidates 为空`，并补一条线路统计日志（如 `collectSourceCount / netdiskSourceCount / scrapeSourceCount / scrapeSources`）。这样用户贴一轮 detail 日志，就能立刻判断是线路识别问题、宿主能力问题，还是候选构建问题。
- 处理这类映射问题时，建议在 `getMergedMetadataCached()` 和详情映射阶段都补“成功路径 + 预览”日志：至少打印 `mappingCount`、`mappingPreview`、`expectedFileId`、`expectedRawName`。这样用户贴一次日志，就能快速判断是缓存未失效、SDK 返回格式变化，还是脚本自身拼装不一致。
- 对普通采集/API 源接 `processScraping()` 时，如果 `getScrapeMetadata()` 返回的 `videoMappings` 里 `episodeName` 有值但 `fileId` 大量是空的（日志常表现为 `mappingPreview=<empty>=>第X集标题`），优先怀疑**传给 SDK 的 scrape candidate 结构不完整**，不要先继续扩展匹配规则。
- 当用户拿官网/宿主截图要求“线路顺序、线路名展示与官网一致”时，默认不要继续按“画质评分/自定义质量分”重排所有线路。更稳妥的默认做法是：
  1. 先区分“站内/蓝光线路”和“聚合/站外线路”的原始来源；
  2. 优先保留上游接口顺序，只有在用户明确点名某几条线路要固定前置时，才对这几条做**最小范围优先级排序**；
  3. 其余线路保持原顺序，避免为了“看起来更智能”把官网顺序打乱；
  4. 同时补一条 `lineOrder` 之类的日志，直接打印最终输出线路名数组，方便用户拿截图对照。
- 线路展示名如果来自 `site_*` / `site123_*` 这类站外聚合 code，默认不要把这段 code 再拼成括号后缀；更稳妥的规则是：**只有 source code 不以 `site` 开头、且确实能给用户提供辨识价值时，才显示 `名称 (CODE)`**。例如保留 `JD蓝光 (JD4K)`、`NB蓝光 (NBY)`，但 `爱奇艺 (site_xxx)`、`站外-如意 (site_xxx)` 默认只显示名称本身。
- 如果这次是“在旧脚本基础上克隆一个新文件名版本”（例如 `A.js` 复制成 `B.js` 再改），且用户后续要走 Git / PR，默认同时核对 4 件事：
  1. `@name` 是否改成新脚本名；
  2. `@downloadURL` 是否指向最终新文件路径；
  3. `@version` 是否 bump；
  4. Git 提交时是否**只提交新文件**、避免把旧脚本未跟踪副本顺手带进 PR。
- 更稳妥的默认候选结构可向仓库内已验证脚本对齐，至少同时传：`fid`、`file_id`、`file_name`、`name`、`format_type: "video"`；只传 `file_id/file_name` 有时会让 SDK 能识别分集名，却不回填映射键，导致详情改名、弹幕匹配、播放记录 episodeNumber 全部失效。
- 遇到这类问题时，日志建议再补两层：1) `detail()` 调 `processScraping()` 前打印 scrapeCandidates 预览（如 `nid=>rawName`）；2) `play()` 若 `matched=false` 但 `mappings.length>0`，打印前 1~2 条原始 mapping JSON sample。这样用户贴一轮宿主日志，就能快速区分是 SDK 映射键没生成，还是脚本匹配键用错。
- 如果 `processScraping()` 返回看似成功（甚至 `result={}`）但 `videoMappings.fileId` 仍为空，不要把 `result={}`` 本身当作失败证据；应以 `getScrapeMetadata()` 的 `videoMappings` 实际结构为准继续判断。
- 对像影巢 / HDHive 这类“资源列表接口 + 详情解锁接口”双阶段网盘源，如果后台日志显示同一条资源被反复请求 `/resources/:type/:tmdb_id` 或 `/resources/unlock`，排查时不要只看限流；应优先检查**是否真的做了结果缓存**，以及 TTL 是否符合业务预期。更稳妥的默认策略是：
  1. `/resources/{type}/{tmdbId}` 这类上游资源列表接口按 `mediaType + tmdbId` 做结果缓存，TTL 默认可直接拉到 **1 天**；
  2. `/resources/unlock` 这类按资源 slug 解锁分享链接的接口，不要只做“一分钟限流计数”，还要按 **`slug` 单独缓存 unlock 成功结果**，TTL 默认可直接拉到 **1 个月**；
  3. `detail()` 不应再直接调用原始 unlock 请求函数，而应统一走 `getXxxUnlockCached(slug)` 之类包装层；
  4. 日志至少区分 `缓存命中 / 已写缓存 / detail 实际来源 cached=true|false`，否则用户只看后台接口日志时很难判断是缓存没做、TTL 太短，还是根本没命中。
- 如果这类源同时访问 TMDB 与自有上游 API（如 HDHive），且用户要求“共用一个上游 HTTP 代理变量”，默认不要只给某一侧单独加 `XXX_PROXY_URL`。更稳妥的做法是：
  1. 优先定义一个共享变量（如 `UPSTREAM_HTTP_PROXY_URL`，可兼容 `HTTP_PROXY_URL` 别名）；
  2. TMDB 请求链路若原先走 `OmniBox.request` 且不便注入代理，可改用 `axios` 并复用统一 `buildAxiosProxyConfig()`；
  3. 站点自有 API（如 HDHive）也优先读取该共享变量，同时兼容旧的单独代理变量（如 `HDHIVE_PROXY_URL`）作为 fallback，减少用户迁移成本；
  4. 日志至少打印 `TMDB 启用共享代理`、`HDHive 启用代理 source=...` 这类来源日志，方便用户判断两条链路是否真的走了同一代理。
- 做这类共享代理改造时，默认优先复用一个统一 helper（如 `getSharedProxyUrl()` + `buildAxiosProxyConfig()`），避免 TMDB 和站点 API 各自手写一套解析逻辑，后续维护时更不容易一边支持认证代理、一边遗漏。
- 处理详情元数据回填时，**简介字段优先级**要格外小心。像 `payload.remark` 常常只是“免费资源/热门/评分榜”之类短标签，不能默认放在 `scrapeData.overview` 前面；更稳妥的默认顺序应是：`scrapeData.overview -> payload.remark -> fallback`。如果用户反馈“演员导演都有了但简介没了”，优先先检查是否被 `remark` 抢先覆盖，并在日志里额外打印 `overviewLen` 之类长度指标，方便快速判断脚本是否真的返回了简介。
- 对这类 detail 元数据增强，如果最终目标是补 `vod_actor / vod_director / vod_area / vod_lang / type_name / vod_content`，不要只依赖 `processScraping()` 返回的基础 `scrapeData`；更稳妥的默认做法是，在已知 `tmdbId + mediaType` 的前提下，再打一遍 `tmdbGet(..., { append_to_response: "credits" })`，把 `credits` 与已有 `scrapeData` 合并后统一回填。否则常见现象是分集改名成功，但演员导演始终为空。

- 如果官网下载区里的资源标题常带营销前缀/后缀，但核心片名本身较短（例如只有 3~4 个字），分享名评分逻辑不要简单地“最短降权后直接丢弃”；更稳妥的是把“纯平台词/纯按钮词”判为无效，同时允许“短片名 + 更丰富资源标签”的组合标题胜出，避免最终又回退成泛化剧名或 `分享N`。
- UZ 端有 `search: 1` 这种专用字段，必要时才加
- OmniBox 宿主对“点击搜索结果后的首击链路”可能是**宿主强制行为**，不要默认以为 `vod_tag: "folder"` 就一定会把搜索结果点进 `category()`。若日志已经证明“搜索结果点击后实际执行的是 `detail(videoId)`”，则应优先判断：宿主是否无视 `detail()` 返回的 folder list、是否根本不会把该首击当成文件夹页。此时排查顺序默认改为：
  1. 先在 `search()` / 搜索结果项上埋日志并观察宿主首击到底调用 `detail` 还是 `category`
  2. 若宿主首击固定走 `detail`，不要继续只改 `detail()` 返回的 `vod_tag/type_id/search` 猜字段；应尽快切换为“搜索结果自身就是根 folder 项 / 特殊 folder id”的方案，尽量把真正的文件夹页放到 `category()` 链路里
  3. `detail()` 在这种场景里更适合只做缓存预热、网盘链接抽取、PanCheck 过滤与日志落证，不要把它当成一定能承载首层 folder UI 的入口
  4. 交付前明确让用户复测并回贴日志，重点核对：`执行方法` 是 `detail` 还是 `category`、传入的 `videoId/categoryId` 是原始剧集 id 还是 folder id、以及是否出现“分类返回剧集网盘分组文件夹页”之类的链路日志
- `@version / @downloadURL / @indexs / @push / @dependencies` 是当前官方明确说明会被后端实际解析的注释属性；不要凭印象扩展“还有别的也会生效”。
- 磁力 / 电驴类详情分集，默认优先返回“短标题 + 简单 playId”结构；除非确有必要，不要在 `episodes[].playId` 里塞大段 JSON 元数据，否则某些端可能出现按钮空白、详情渲染异常或播放取值异常。
- 当同一详情页同时存在“网盘线路 + 站内采集线路 + 磁力链接”时，若用户要求把采集/磁力做成与网盘分组平级，默认采用 **双层 folder** 结构而不是把所有线路直接塞进首层详情：
  1. 根目录返回 `采集`、`磁力`、各网盘类型分组并列项；
  2. `采集` 点击后进入“采集线路列表”，每条线路再进详情；
  3. `磁力` 点击后进入“磁力线路列表”，每条线路再进详情；
  4. 可为这两类目录额外编码 `sourcefolder` / `sourceline` 之类的 folder id，在 `category()` / `detail()` 中分别识别；
  5. `extractPanLinksFromDetail()` 这类缓存构建函数里，除网盘链接外也要顺手抽取 `.py-tabs + .bd ul.player` 的采集线路，以及页面 HTML / `<a>` 中的 `magnet:` 链接并一起写入 grouped 缓存。
- 采集线路做分组缓存时，优先沿用普通采集版脚本的线路命名与去重规则：例如 `normalizeCollectLineName()`、重复线路名自动补序号、`isBlockedLineName()` 过滤掉名称里带“磁力”的站内线路，避免分组版与普通版表现不一致。
- 磁力分组默认以“线路列表 → 详情单线路”形式返回；磁力详情里的分集名与 `playId` 应保持轻量，推荐 `磁力线路N / 磁力资源N` + 原始 `magnet:` 直链，并在 `play()` 最前面特判 `playId.startsWith("magnet:")` 直接 `parse:0` 返回，避免再走网盘/站内播放分支。
- 如果 `grouped` 缓存里同时混入“网盘 share URL 数组”和“采集/磁力线路对象数组”，后续对 grouped 做统计、列表构建、PanCheck 过滤时要显式区分类型：PanCheck / URL 规范化只作用于网盘链接数组，不要把采集/磁力对象误当 URL 去 `normalizeShareUrl()`。
- 当用户明确要求“点击磁力线路后直接到详情页，不要多一层文件夹”时，默认不要只改 `detail()` 展开逻辑；还要同时检查 **列表项语义** 是否正确：
  1. 具体磁力线路项（`sourceline`）应优先标记为 `vod_tag: "video"`，这样宿主才会按详情项打开，而不是继续当文件夹点进下一层；
  2. 如果首层 `磁力` 分组下实际上只有 **1 条磁力线路**，优先直接把这条线路作为首层返回项（`vod_id` 指向 `sourceline`，`type_id: "source_line"`，`vod_tag: "video"`），避免用户先点“磁力”再点“磁力线路1”的冗余一级；
  3. 如果有 **多条磁力线路**，则保留 `磁力` 分组为 `folder`，进入后展示 `磁力线路1/2/3...`，但这些具体线路项仍应是 `video`，点击直达详情；
  4. 交付前优先核对宿主实际 UI 是否仍停在“磁力线路1/2/3...”这一层，避免误以为只改 `detail()` 就足够。
- 学到 drpy-node 体系后，默认把它当作 **另一种输出目标与工作流参考**，不是让 OmniBox 源直接套 `var rule = {}`。能吸收的是：分诊顺序、分阶段交付、播放专项验收红线；不能把宿主 API、全局函数、仓库上传动作原样搬进 OmniBox 运行器。
- 如果用户提供的是其它宿主/其它项目格式的参考源（例如“不夜版”这种 `module.exports = async (app, opt) => { app.get(...); opt.sites.push(meta); }` 的 Express 路由式源），默认只能把它当作 **站点协议/解析逻辑参考**，不能直接复制其导出形态到 `影视/采集/*.js`。真正的 OmniBox 源必须改写为仓库内已验证的 OM Spider 结构：`const OmniBox = require("omnibox_sdk")` + `const runner = require("spider_runner")` + `module.exports = { home, category, detail, search, play }` + `runner.run(module.exports)`（这是当前仓库多数 JS 采集源的落地形态），或等价的 `class XxxSpider extends OmniBox.Spider { ... }` + `runner.run(new XxxSpider())`。优先对齐当前目录里已验证的同类脚本，而不是只按抽象模板重写。
- OmniBox 源方法命名要优先兼容仓库/runner 实际调用：常规影视采集源应实现并导出 `home/category/detail/search/play` 五方法；不要只写成 `getCards/getTracks/getPlayinfo` 或只写 class 方法而不确认 runner 是否会调用。若采用 class 形式，交付前必须对照同仓库示例或 runner 行为确认可运行。
- 交付前要显式 grep/检查目标文件中不再残留 `module.exports = async (app, opt)`、`app.get(meta.api`、`opt.sites.push(meta)` 这类外宿主格式；若仍存在，应判定为“格式错误，不能提交/PR”。
- 如果已经误提交/误开 PR：不要只口头承诺“继续修”。应立即删除多余参考文件、重写目标文件为 OM 格式、执行 `node --check`、grep 外宿主残留、追加 commit 并推送到既有 PR，同时更新 PR 说明。
- 新建源时，默认先追求“最小可用链路”而不是一步做满：优先顺序通常是 `home/category/search/detail` 可用，再补稳定 `play()`，最后才是上传仓库、标签整理、额外增强。
- 对 MacCMS / myui 一类站点，列表封面不要只盯 `img src`：很多卡片真实封面挂在详情链接 `<a>` 的 `data-original` / `data-src` 上，必要时应优先从详情链接取图，再回退 `img`。
- 搜索页若页面同时包含推荐区、热门区、主结果区，不要全页抓详情链接；应优先缩小到主结果容器（例如 `#searchList`）后再解析，否则容易把搜索页其它区块混进结果。
- 有些片库/zanpian 风格站不是 `/vod/detail` + `/vod/play`，而是前台伪静态路由：分类 `/class/{type}.html` / `/class/{type}-{page}.html`，详情 `/intro/{id}.html`，播放 `/dplay/{id}-{sid}-{nid}.html`，搜索 `/query.html?wd=关键词` 与 `/query/page/{page}/wd/{keyword}.html`。新建源前应先探明真实前台路由，不要先按 MacCMS 默认 `/vod/*` 路由硬写。
- 部分前台站（如毒舌03这一类）会挂一层 `cdndefend_js_cookie` 式 JS 防护：首请求可能先返回 `HTTP 850` challenge，再要求后续请求带上按当前 challenge 中 40 位前缀 + 递增数值计算出的 cookie。不要把前缀写死；应优先从当次返回的 challenge HTML（如 `['PREFIX','cdndefend_js_cookie=']`）动态提取前缀，再用 `sha1(prefix + i)` 命中指定字节条件生成 cookie，并以“850 → 提取前缀 → 算 cookie → 重试”作为默认调试路径。
- 这类前台站的真实路由不一定是传统 MacCMS：可能是分类 `/channel/{id}.html`，搜索 `/search?k=关键词&t=某个令牌`，详情 `/detail/{id}.html`，播放 `/play/{vid}-{sid}-{nid}.html`。建源前先以实际前台链接为准，不要先套 `/vodshow` / `/vodsearch`。
- 进一步注意：某些自定义前台会把 `/channel/{id}.html` 仅作为“频道首页 / 推荐页”，真正带筛选和分页的分类库则在 `/show/{tid}-{type}-{area}-{lang}-{year}-{by}-{page}.html`。遇到这类站点时，`category()` 不要只翻 `/channel/{id}.html?page=N`；应优先按前台真实筛选结构拼 `/show/...`，并让 `filters` 的 key / value 与前台按钮保持一致，否则容易出现“分类页能进但筛选不全 / 切筛选无效 / 分页不对”。
- 这类 `/show/...` 路由的 `filters` 默认至少核对：`type`、`area`、`lang`、`year`、`by`，其中 value 要优先使用前台真实编码或文案（如 `中国大陆`、`2010_2019`、`3`），不要擅自简写成猜测值。
- 如果分页来自 `/show/...` 路由，`parsePageCount()` 也要补上对应链接模式，不要只匹配 `/channel/...` 或搜索分页，否则列表能出但翻页会卡死在 1。
- 某些自定义前台搜索参数里的 `t` 不是“由关键词本地计算”的固定签名，而是页面实时下发的隐藏 token（常见于 `<input name="t" value="...">`）。遇到这类站点，`search()` 应先请求首页/搜索页提取真实 `t`，再带 `k + t` 发起搜索；不要先按 base64 关键词或自猜算法硬写，否则很容易出现“请求成功但结果为 0”。
- 如果站点搜索结果页/首页存在懒加载双图结构（如一张 `logo_placeholder*` 占位图 + 一张真实封面 `data-original`），列表/搜索/详情取图时要显式跳过占位图与 `#noneCoverImg`，优先取真实封面，必要时封装统一 `pickImage()` 之类的回退逻辑，避免首页/分类/详情全都只显示占位图。
- 进一步注意：有些前台卡片真实封面不在 `img` 本身，而是挂在包裹它的详情链接 `<a>` 上（例如 `<a data-original>` / `<a data-src>`）。遇到“列表有标题但一直只显示空白图/占位图”时，`pickImage()` 不要只读 `img`，还要同时检查当前节点、`parent('a')` 与 `closest('a')` 的 `data-original / data-src / src`，并优先记录实际命中的来源，避免误以为页面没有图。
- 如果 OmniBox 端筛选按钮已经把值放在 `params.filters`，而脚本只读取 `params.extend` / `params.ext`，会出现“筛选按钮看起来存在，但点击后始终同一结果”的假成功。遇到分类筛选异常时，`category()` 默认同时兼容 `params.filters || params.extend || params.ext`，并把最终 filters 与请求 URL 打日志，优先确认是否真的请求到了带筛选参数的前台路由。
- 做这类前台分类筛选时，默认还要把筛选值先做白名单规范化（至少限制在 `sort/type/area/lang/year` 等当前分类支持字段内），并给 `sort` 提供安全默认值；同时在拼接分类 URL 前对中文筛选值做 `encodeURIComponent`，避免“按钮有值但中文筛选一点击就失效/回空”的隐性编码问题。
- 如果同一站点存在“普通采集版”和“网盘分组版”两个脚本，且用户要求它们的筛选体验保持一致，默认优先把筛选常量与辅助函数抽成同构实现：例如统一 `FULL_TYPE_OPTIONS / FULL_YEAR_OPTIONS / FULL_AREA_OPTIONS / FULL_LANG_OPTIONS / FULL_SORT_OPTIONS`、`buildFilterOptionList()`、`buildCategoryFilters()`，再按分类决定是否裁剪某些维度。这样后续维护时两边更不容易一边加了筛选、一边遗漏。
- 这类筛选对齐改动完成后，除了 `node --check` 语法检查，建议再做一次轻量级关键路径验证：直接调用/模拟 `parseFilters()` 与 `buildCategoryPath()`，至少确认三件事：1) `params.filters / params.extend / params.ext` 都能被兼容；2) 非法 `sort` 会回退默认值；3) 中文筛选值最终进入分类 URL 时已正确编码。这样可以在不跑宿主的情况下，先拦住“按钮存在但 URL 拼错/筛选不生效”的回归。
- 如果筛选按钮看起来已经渲染，但目标脚本仍使用旧的精简 `FILTERS`，而同站点另一个脚本已有更完整筛选定义，默认优先以“同站点已验证版本”为基准对齐，不要重新拍脑袋发明一套更小的选项集。
- 如果用户给了站点前台截图或明确列出了筛选栏文案，`FILTERS` 默认优先按前台可见项完整补齐，不要只留最小子集；至少让宿主可见的 `类型 / 年代 / 地区 / 语言 / 排序` 与网页端保持大体一致，再通过日志继续校准真实后端接受值。
- 对 WordPress / Zibll 这类 taxonomy 分类站，如果前台顶部导航已经挂出了分类子菜单，`category()` 的筛选值应优先直接复用这些真实 taxonomy slug，而不是自猜 `tv/anime` 一类短值。更稳妥的默认做法是：先抓首页导航里的 `/category/{main}/{sub}` 链接，整理成 `CATEGORY_CONFIG + FILTERS`；随后 `category()` 同时兼容 `params.filters || params.extend || params.ext`，对白名单内的 `subclass/orderby` 做归一化，再拼 `/{main}/{sub}` + `/page/{n}` + `?orderby=...`。若首页存在 `?orderby=modified/views/like/comment_count` 这类排序链接，即使分类页没显式渲染排序按钮，也值得直接请求分类页实测顺序是否变化；确认有效后再把这些排序值接给宿主。详见 `references/wordpress-taxonomy-category-filters.md`。
- 若为这类 taxonomy 分类页新增了“完整路径分页”能力，`parseCategoryPageCount()` 的正则不要只匹配主分类 slug；应按完整 `categoryPath`（如 `show/domestic-show`）匹配 `/category/{path}/page/{n}`，否则二级分类翻页常会误判回 1。
- 若为这类自定义前台新增了 `/show/...` 路由拼装辅助函数（如 `buildShowUrl()`），要同步确认它依赖的 `encodeSegment()` / 编码 helper 已定义且在当前文件可见；否则容易出现“分类方法执行成功但日志报 encodeSegment is not defined，列表 silently 回空”的问题。
- 做图片问题排查时，建议在 `home/category/detail` 各埋少量图片专项日志：至少打印 `title`、`href`、最终 `picked` 图片地址，以及各候选来源（`self` / `parentA` / `closestA`）的 `data-original / data-src / src` 摘要。这样用户一发调试终端日志，就能快速判断是没取到图、取到占位图，还是 OmniBox 前端渲染层另有问题。
- 如果某个网盘源 detail 日志已经证明 `processScraping()` 跑通、`videoMappings` 也能命中并完成分集改名，但详情页里的演员/导演/地区/语言等主信息仍然没变化，优先检查**最终返回的 vod 对象是否真的把 `scrapeData` / TMDB detail 元字段映射回去了**，不要只盯着分集映射链路。常见漏项是：只回填了 `vod_name / vod_pic / vod_year / vod_content / vod_douban_score`，却忘了补 `vod_actor / vod_director / vod_area / vod_lang / type_name`。
- 进一步地，如果这类脚本还会在 detail 阶段额外请求 TMDB 详情来补信息，而你又希望拿到演员/导演，默认不要只请求基础详情接口；应优先带上 `append_to_response=credits`，否则很多情况下 `tmdbDetail` 本身并不包含 `credits.cast / credits.crew`，后续即使写了演员导演映射逻辑也会一直是空。
- 处理这类“刮削改名成功但主信息没补齐”的问题时，默认补两条诊断日志：1) `TMDB/刮削补齐成功`（至少记录是否拿到 credits）；2) `元数据回填结果`（直接打印最终回填出的 `title/actor/director/area/lang/type`）。这样用户贴一轮 detail 日志，就能立刻判断是上游没给、脚本没映射，还是宿主没展示。
- 某些自定义前台播放页不会暴露 `player_aaaa`，而是把真实媒体地址放在 `const playSource = { src, type }` 这类前端初始化对象里，并且字段可能是 `\\uXXXX` 转义；`play()` 调试时要先搜 `playSource` / `Player(config)` / `xgplayer` 相关片段，不要只盯着 `player_aaaa`。
- 对这类“官网能播、脚本多数线路不通”的前台站，`play()` 不要只做“解析出一个 URL 就直接 `parse:0` 返回”。更稳妥的默认链路应是：
  1. 如果拿到的是明确媒体直链（`m3u8/mp4`），再 `parse:0` 直返；
  2. 如果拿到的是播放页 / 解析页 / 非媒体网页 URL，优先调用 `OmniBox.sniffVideo(targetUrl, headers)` 做 **SDK 嗅探**；
  3. 如果 SDK 嗅探失败，再回退标准 app 嗅探写法：`{ parse: 1, url: targetUrl, urls: [{ name, url: targetUrl }], header }`；
  4. `header` 至少补 `Referer / Origin / User-Agent`，并把 `resolved url`、`sniff start/result`、`sdk sniff failed -> parse=1 fallback` 打日志。
  这样可以避免把普通播放页误当直链返回，尤其适合“多数线路依赖前端播放器二跳/嗅探”的站。
- 但如果日志已经证明解析站 `api.php` **明确返回 `code:200` + `type:m3u8/mp4`**，且宿主报错发生在 `sdk sniff on resolved failed`，排查顺序要立刻切到“**脚本是否真的解出了最终媒体 URL**”，不要继续盲猜请求头。很多解析站（如歪比巴卜这类）不是简单 `payload.url + aes_key/aes_iv` 直接 AES；应优先复用解析页 `setting.js` 运行时函数（如 `decrypt()` / `deplay()`）还原真实媒体 URL。若 `resolved.url` 仍是 `/player/?url=...` 解析页，就说明解密链没走对；这时应先修正 `resolveWbbbPlayerUrl()` 的运行时解密，再让 `play()` 对已解出的媒体直链直接 `parse:0` 返回，仅对仍像网页的 URL 才尝试 sniff。详见 `references/wbbb-parse-runtime-and-tvbox-play.md`。

- 遇到 WordPress/资源站这类“评论后可见”详情页时，不要只在 HTML 里硬找隐藏区再放弃。更稳妥的默认调试路径是：
  1. 先确认是否存在明确隐藏提示文案（如“此处内容已隐藏，请评论后刷新页面查看”）；
  2. 再从详情页实时提取**真实评论表单**参数，不要只盯 `#comment_post_ID / #comment_parent / #_wpnonce` 单点值；优先先定位 `#commentform` / `#respond form` / 含 `textarea[name="comment"]` 的 form，再整体提取 `input/textarea/select[name]`。
  3. 如果页面里根本没有评论 form，而只有“请登录后发表评论 / 请登录后查看评论内容”，要优先判断：当前请求拿到的仍是未登录态 HTML，或评论表单需前端二次 Ajax 加载；此时不要把登录弹窗里的 `_wpnonce` 误当评论 nonce。
  4. 如果已经拿到真实 `#commentform`，但字段列表仍然只有 `comment/comment_post_ID/comment_parent/_wpnonce` 这类极小集合，不要继续机械地猜更多 hidden input；下一步应直接追站点前端评论 JS（如 `comment.min.js`），确认它是否还依赖**提交按钮元信息**（`form-action`、`ajax-href`、按钮 `name/value`、`data-postid`、`data-nonce` 等）。
  5. 提交评论时优先透传整张评论表单的字段，再覆盖 `comment / comment_post_ID / comment_parent / _wpnonce / action=submit_comment` 这类核心值；如果前端 JS 里对评论正文做了 `htmlEscapes` 一类转义，脚本端也应对齐后再提交。
  6. 若接口返回 `HTTP 400` 且 body 仅为 `"0"`，默认先怀疑“表单字段不完整 / nonce 取错 / 实际未拿到评论 form / 提交按钮元信息缺失”，不要直接当作“评论待审核”。
  7. 若已经确认主题前端默认走 Ajax 评论，但脚本侧 Ajax 仍反复 `400/0`，应增加**标准 WordPress 评论提交流程回退**：优先用表单自身 `action`，否则回退 `wp-comments-post.php`，然后再刷新详情确认是否真正解锁。
  8. 若标准评论回退返回 `status=200` 且 `raw=""`，不要立刻判失败；继续检查响应头里的 `location` / `set-cookie`。如果出现 `unapproved=`、`moderation-hash=`、`#comment-数字`，或站点前端明确只在 `comment_approved > 0` 时解锁，应优先判断为“评论可能已受理但仍待审核”。
  9. 若接口返回“重复评论”之类结果，也不要立刻判失败，应继续刷新详情页确认隐藏内容是否已解锁。
  10. 站点若还有签到 / 点赞 / 帖子动作类 Ajax，不要只发 `action=...`；优先从页面全局脚本提取 `post_action_nonce`，并按需同时提交 `_wpnonce + post_action_nonce`，否则常见表现是 `HTTP 400 code=0 msg=`。
  11. 日志至少区分：未配 cookie、是否找到真实评论 form、form id/action、字段名列表、提交按钮 id/form-action/ajax-href、评论提交结果、是否触发回退、回退响应 location/set-cookie、刷新后是否仍含隐藏提示、是否疑似待审核。
  12. 本类问题的会话级细节可参考 `references/wordpress-zibll-hidden-comments.md`。
- 对这类同时带“每日签到”能力的 WordPress/Zibll 资源站，若用户要求自动签到，默认优先放在 `home()` / `search()` 等高频入口**异步触发**，并使用 SDK `getCache/setCache` 做 24 小时节流；请求体通常只需 `action=user_checkin`。关键点是：签到失败不要阻塞首页/搜索主返回。
- 进一步的默认缓存策略：**自动签到失败也应写入 24 小时缓存**，不要因为接口持续 `400/0`、缺 nonce、或站点风控异常而在每次 `home/search/detail` 都重复打签到接口刷屏。更稳妥的做法是：成功缓存与失败缓存共用 24h TTL，并在日志中明确区分 `命中成功缓存 / 命中失败缓存`，方便用户判断“今天已签过”还是“今天失败后已暂停重试”。
- 这类站点的“站点 cookie”和“网盘/播放授权 token”默认当成两套凭据设计，不要混用。实战里常见情况是：

  8. 反过来，如果刷新后仍然 `hidden=1` 且 `uniqueShares=0`，优先怀疑评论解锁没有真正绑定当前登录态（cookie 不对、缺动态字段、站点要求真实成功评论），而不是先去改播放或网盘解析。
- 这类站点的“站点 cookie”和“网盘/播放授权 token”默认当成两套凭据设计，不要混用。实战里常见情况是：
  - 站点详情解锁依赖 `COOKIE`
  - 分享列表可依赖 share access token
  - 真正的视频/VOD 接口还要求独立 Bearer auth
  其中 **share access token 不能默认当成 VOD Bearer token 使用**；若实测 `get_vod_download_url` 一类接口返回 `401 / code=117 / 无效token`，优先判断是否缺少独立网盘登录 token，而不是继续重试 share token。
- 对网盘分享站点，`play()` 默认采用“尽力而为 + 明确回退”策略：
  1. 先尝试分享态可直接拿到的下载/播放接口；
  2. 若分享态接口受限，再尝试独立网盘授权接口；
  3. 若仍拿不到直链，则回退分享页 URL，并显式返回 `parse: 1` 或页面型兜底，保证源至少可用。
  交付说明里要明确告诉用户：哪些链路已验证可用、哪些因授权不足只能回退页面。

- `play()` 优化时默认采用三段式兜底：先直解析可播地址；若只拿到播放页/解析页而非明确媒体直链，则先尝试 `OmniBox.sniffVideo(...)` 做一次 SDK 嗅探；如果仍失败，再回退到 `parse: 1` 交由播放器嗅探。这样可减少把普通网页误判为直链的情况。

- `play()` 优化时默认采用三段式兜底：先直解析可播地址；若只拿到播放页/解析页而非明确媒体直链，则先尝试 `OmniBox.sniffVideo(...)` 做一次 SDK 嗅探；如果仍失败，再回退到 `parse: 1` 交由播放器嗅探。这样可减少把普通网页误判为直链的情况。
- 播放逻辑若同时包含“直解析 + 媒体信息/播放记录”后置任务，建议先返回主播放结果，再异步写播放记录和媒体时长，避免阻塞启播。

- 对像影巢这类同时依赖“上游站点接口 + TMDB 接口”的网盘源，如果用户要求统一走代理，默认优先采用**公共上游 HTTP 代理变量**，而不是只给某一侧单独配代理。更稳妥的默认策略是：
  1. 新增并优先读取 `UPSTREAM_HTTP_PROXY_URL`（可兼容别名 `HTTP_PROXY_URL`）；
  2. TMDB 请求链路与 HDHive/站点接口链路都优先共用这一个代理变量；
  3. 若脚本历史上已有 `HDHIVE_PROXY_URL` 之类旧变量，可短期保留为向后兼容 fallback，但交付说明里要明确“公共代理优先，旧变量仅兼容”；
  4. 日志至少区分 `TMDB 启用共享代理`、`HDHive 启用代理 source=...`，方便用户快速判断两条链路是否都真的走了同一代理。
- 对这类“原先 TMDB 走 OmniBox.request、站点接口走 axios”的脚本，如果要补共享 HTTP 代理，不要只改配置常量；还要优先确认 **TMDB 请求实现本身是否支持传代理**。若当前实现不支持代理参数，更稳妥的默认做法是把 TMDB 请求改为与站点接口一致的 axios 请求路径，并复用同一个 `buildAxiosProxyConfig()`/共享代理 helper；否则容易出现“变量加了，但 TMDB 实际仍不走代理”的假修复。

- 如果用户已经明确说“调用工具修改 / 直接改 / 别只分析”，该类 OmniBox 修源任务必须在同一轮回复里立即进入 **read/patch/check** 实操链路；不要先回一段分析或承诺后又不动手。最低交付节奏应是：读目标文件 → patch 目标逻辑 → `node --check`/`py_compile` 校验 → 再汇报结果。
- 如果用户明确限定“只修仓库里的脚本 / 参考文件不用管”，默认不要去改上传的 `.txt/.md` 参考稿；只把参考内容当作对照信息，实际 patch 只落到用户点名的仓库文件。
- 对像 LIBVIO 这类前台筛选并非固定 `/type/{id}-{page}.html`、而是依赖页面实时筛选链接的站点，`category()` 默认优先走 **live filter href chaining**：先抓 `/type/{id}.html` 或当前筛选页，解析真实筛选 `<ul class="clearfix">` 中的 href，再逐步跳到 `/show/...` 最终地址；不要想当然硬拼所有筛选位。此时还要同时兼容 `params.filters / params.extend / params.ext / params.filter`，并把 `最终 filters + finalUrl` 打日志。
- 对 LIBVIO 这类站点排查“分类对不上”时，先用实站 HTML 和宿主调试日志核对 **顶部导航展示名、`/type/{id}.html` 标题、筛选行当前选中项、最终请求 path**，再决定是否修改 `CLASS_LIST`。截图里的 UI 名称可能是宿主自定义展示（如“剧集”），官网 HTML 可能显示“电视剧”；不要只凭官网标题或单次 curl 贸然改分类名。若确需改分类名，必须同步 bump `@version` 并跑 `node --check` 后再汇报。
- 排查“网盘线路没有加载具体集数信息”时，不要只解释 `processScraping/getScrapeMetadata` 理论链路；必须实测详情页 DOM 的网盘区结构（如 `playlist-panel netdisk-panel`、`netdisk-item`、`data-clipboard-text`、`href`、`span.netdisk-url` 等）和日志中的 `netdiskSourceCount / scrapeCandidates / mappingPreview`。如果官网 DOM 只暴露分享链接而文件名来自网盘 SDK，优先补 `loadPanFiles/getAllVideoFiles` 诊断日志（文件数、首几个 fileId/fileName、目录递归失败原因），再判断是 DOM 解析失效、网盘 SDK 未返回子文件、还是刮削映射未命中。
- 任何 OmniBox 修源交付前，不能把“已做准备/理论上会工作”当成修复完成；至少要给出实际 patch、版本号变化、静态检查结果，以及一条关键路径实测或明确说明未能实测的阻塞原因。
- 这类前台详情页若用户反馈“详情页没有线路显示”，不要只盯旧的 `playlist-panel`；要优先检查真实 DOM 是否已经切到 `stui-vodlist__head + stui-content__playlist`、`module-tab + module-play-list` 等新版结构，并在返回 `vod_play_sources` 的同时，按需补一份兼容宿主旧展示链路的 `vod_play_from / vod_play_url`，避免“数据其实有了，但某些端不显示线路名”。
- 对 LIBVIO 这类站点，分类分页末页判断不要用 `html.includes('下一页')` 或全页字符串泛匹配；必须只认真实分页 `<a>` 中的下一页 href。找不到目标页时 `resolveCategoryPageUrl()` 返回空，避免复用末页 URL 导致 `pagecount` 随当前页无限增长。详见 `references/libvio-live-filters-and-pan-episode-fallback.md`。
- 对 LIBVIO 网盘区，“合集”通常只是分享入口名，不是分集。必须进入播放页解析真实分享链接并用网盘 SDK 展开具体视频文件；拿不到 `fileId/fileName` 时跳过该网盘源并打诊断日志，不要把“合集”生成成一集。播放页链接匹配应兼容 `/play/任意slug.html`，不要只匹配数字三段式。UC 网盘还要额外注意：若详情统计里出现 `UC网盘:1`，但没有 `开始处理网盘线路: UC网盘` 日志，说明 UC 没进 `splitNetdiskPanels()`，而是被当普通采集线路；应补充从 `allCollectSources` 按线路名兜底解析或扩展网盘面板识别。UC 播放页常在 `player_aaaa.url` 暴露 `https://drive.uc.cn/s/...`，解析时要保存 `resolvedShareUrl` 并用 `{shareUrl}|{fileId}` 写入 `meta.fid`，避免后续播放/刮削链路拿到空分享链接。详见 `references/libvio-live-filters-and-pan-episode-fallback.md` 与 `references/uc-netdisk-handling.md`。
- 对 LIBVIO 详情页同时存在站内采集线路与网盘/下载线路时，网盘解析入口必须先按线路标题/按钮文本做语义过滤：只有含 `网盘` 或 `下载` 的线路才进入“播放页 -> 分享链接 -> 网盘 SDK 展开”链路；`HD7播放`、`4K蓝光` 等普通采集线路绝不能逐集点进当网盘解析，否则会制造重复线路并触发 429。反过来，所有明确的网盘类型（包括 UC/夸克/百度等）都要尝试解析，**不要因为某一种网盘曾失败就硬编码跳过**；每个网盘分享应作为独立线路，解析成功才展示，解析失败才排除。日志至少区分：开始处理网盘线路、跳过非网盘/下载线路、播放页是否拿到 `player_aaaa`、解码出的 `shareUrl`、`isPanUrl` 判断、`loadPanFiles` 返回视频数/预览、解析成功、解析失败排除。若失败后仍出现 `UC网盘:1` / `合集 -> S1E0`，优先检查该网盘占位是否被 `extractPlaylistSources()` 当作普通采集线路送进刮削候选；已成功展开或解析失败的网盘占位都不应再进入 collect scrape bucket。详见 `references/libvio-live-filters-and-pan-episode-fallback.md`。
- 对 LIBVIO/同类多线路详情页优化刮削时，默认采用“**先统一收集候选、只刮削一次、再遍历最终所有线路回填**”的链路，参考 `影视/网盘/木偶.js`：先把普通采集线路 + 已成功展开的百度/夸克/UC 等网盘文件全部整理成全局 `scrapeCandidates`，只调用一次 `OmniBox.processScraping(videoId, keyword, keyword, scrapeCandidates)` 和一次 `getScrapeMetadata(videoId)`，随后遍历最终展示的 `vod_play_sources`，用 episode meta 里的 `fid` / `_fid` 匹配 `videoMappings[].fileId` 并统一更新 `ep.name`、`meta.e/s/n/fid`。不要按线路重复刮削，也不要只回填当前选中的线路或某个中间 bucket。若截图显示某线路仍是原始文件名，优先核对 `scrapeCandidates.file_id`、episode meta.fid、`videoMappings[].fileId` 三者是否完全一致。详见 `references/libvio-unified-scraping.md`。


## Git / PR 操作补充

- `openclaw -> main` 如果已经有一个打开中的 PR，而你又想把另一个修复单独发成独立 PR，不要继续复用同一个 `openclaw -> main`。
- 更稳妥的做法是：
  1. 先在本地 `openclaw` 完成修改并提交
  2. 再从最新 `origin/main` 切一个专用修复分支
  3. `cherry-pick` 只属于这次修复的提交
  4. 用该专用分支发独立 PR
- 某个修复 PR 合并后，如果后续仍以 `openclaw` 作为默认开发分支，下一轮开发前应尽快把最新 `main` 同步回 `openclaw`，避免分支漂移和 cherry-pick 噪音。

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
