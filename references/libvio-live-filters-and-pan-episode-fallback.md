# LIBVIO 实战教训

## 1. 动态筛选必须以官网 HTML 为准

- LIBVIO 的分类筛选不是固定模板，`/type/2.html`、`/show/...` 的真实筛选项要从当前 HTML 里解析。
- 不要用截图里的展示名反推官网结构，也不要凭印象把地区写成“中国大陆 / 中国台湾 / 中国香港”。
- 官网原文优先：例如电视剧页的地区会直接写成“大陆 / 台湾 / 香港”，筛选值要跟页面原文对齐。
- 如果官网 HTML 与宿主截图不一致，优先信官网 HTML，再结合宿主日志确认是否是前端展示名做了二次包装。

### 电视剧页实测样例

- 按类型：国产剧、港台剧、日韩剧、海外剧
- 按地区：美国、韩国、英国、日本、大陆、台湾、德国、哥伦比亚、意大利、西班牙、丹麦、挪威、法国、香港、泰国、其它
- 按年份：2026 到 2011
- 页面里的“更新至第XX集/已完结”通常在 `pic-text`，不要拿它冒充筛选项。

## 2. 网盘分集名要有备注兜底

- 详情页里的网盘文件名有时不带“第XX集”，这时不要只保留文件原名。
- 如果详情页备注区或卡片备注里有“第XX集/已完结/更新至第XX集”，可以作为 episode 名称兜底。
- 优先级建议：
  1. 文件名本身含集数
  2. 详情页备注含集数
  3. 再回退到原文件名

## 4. 分页末页判断不要用泛匹配

- LIBVIO 的 `/show/...` 筛选页可能没有真实“下一页”，但页面其它区域、筛选链接或旧逻辑里的字符串匹配仍可能让 `pagecount` 跟着当前页虚增，表现为“电视剧-大陆第 7/8 页仍能继续下一页”。
- 不要用 `html.includes('下一页')`、`html.includes(`>${page + 1}<`)` 或 `html.includes(`-${page + 1}---`)` 这类全页泛匹配判断是否还有下一页。
- 更稳妥的规则是：只认真实分页 `<a>` 中指向 `page + 1` 的 href（例如复用 `findCategoryPageHref(html, page + 1)`），找不到就让 `pagecount` 停在当前页或返回空页。
- `resolveCategoryPageUrl()` 找不到目标页时必须返回空字符串，不要复用当前页 URL；否则前端请求更大页码时会反复拿到末页内容。
- 调试时用官网实页确认：末页若只剩少量列表且没有 `<a>下一页</a>`，日志里的 `pagecount` 不应继续增长到 7/8/9。

## 5. 网盘线路“合集”不是分集

- LIBVIO 详情页网盘区可能只给一个播放页/分享入口，按钮文案常是“合集”。这只是入口名，不是可播放分集名。
- 解析网盘时不能把“合集”直接生成成一集；必须进入播放页解析 `player_aaaa`/真实分享链接，再通过 `loadPanFiles()` / 网盘 SDK 展开具体视频文件。
- 只有拿到具体 `fileId + fileName` 的视频文件列表时才生成网盘 episodes；否则应跳过该网盘源，并打日志说明 `sourceName/href/label`，方便继续定位是播放页解析失败还是 SDK 未展开目录。
- 播放页链接匹配不要过窄：不要只接受 `/play/数字-数字-数字.html`，应兼容新版 `/play/任意slug.html` 这类格式，否则会导致网盘播放页根本没进入解析。

### UC 网盘特殊处理

- UC 网盘的真实分享链接既可能出现在 `player_aaaa.url`（实测示例：`from: "ucpan"`, `url: "https://drive.uc.cn/s/..."`），也可能只在源码或脚本片段里暴露 `drive.uc.cn` / `uc.cn/s/` 分享 URL；不要只处理“没有 `player_aaaa`”的分支。
- 网盘解析应先尝试解析 `player_aaaa`：JSON 成功后对 `player.url` 做 `decodePlayerUrl(player.url, player.encrypt)` + `normalizeShareUrl()`，只要 `from` 含 `uc/pan` 或 `shareUrl` 命中 `isPanUrl()`，就应直接进入 `loadPanFiles()`。
- 当 `player_aaaa` 不存在、JSON 解析失败、或解出的 URL 不是有效网盘分享时，再扫描播放页 HTML 中的 UC 链接作为兜底。
- 如果详情页里 `UC网盘` 段是标准 `stui-vodlist__head + stui-content__playlist` 结构（如 `<h3>UC网盘</h3><a href="/play/...">合集</a>`），`splitNetdiskPanels()` 必须能把这类 head+ul 成对提取出来；不要只依赖 `playlist-panel netdisk-panel` 或过窄的 “视频下载” 正则。
- 如果详情日志里只有最终 `scrapeSources` 出现 `UC网盘:1`，但没有 `开始处理网盘线路: UC网盘`，说明 UC 没进入 `splitNetdiskPanels()` 网盘解析入口，而是被 `extractPlaylistSources()` 当成普通采集线路了；此时应从 `allCollectSources` 里按线路名补充一个网盘解析兜底，或扩展 `splitNetdiskPanels()` 识别该 DOM 结构，而不是继续改 UC 的 `player_aaaa` 解析。
- 从 `allCollectSources` 兜底解析 UC 时，同样要打开真实播放页并优先读 `player_aaaa.url`，不能只扫描裸露链接；实测 LIBVIO UC 播放页存在 `player_aaaa={..., "url":"https:\/\/drive.uc.cn\/s\/...", "from":"ucpan"}`。
- 成功路径要把实际使用的 `shareUrl` 保存到后续 `playId` meta 中；不能在生成分集时再次从空的 `playerJson` 反推 shareUrl，否则会得到空 shareUrl，导致 `play()` 网盘直取失败。
- 关键诊断日志至少区分：`开始处理网盘线路(UC网盘)`、`播放页 player_aaaa 解析结果`、`UC 页面提取候选链接数`、`UC 页面提取解析成功/无文件/未能提取有效分享链接`、`UC collect 兜底解析成功/失败`。
- 解析成功或确认失败的 UC 占位线路都不能继续进入刮削候选；否则会把“合集”误刮削为 `S1E0` 并在 UI 里残留一条不可播的 UC 线路。
## 6. 网盘展开成功后要移除占位线路并控制请求量

- LIBVIO 详情页同一批播放入口可能同时被普通采集线路解析和网盘面板解析命中，表现为界面出现重复线路：例如 `HD7播放` 同时出现在第一排与第二排，或 `视频下载（夸克）/视频下载（百度）` 这类未展开占位与 `夸克网盘-服务端代理/直连` 并存。
- 默认策略：先解析所有站内采集线路为 `allCollectSources`，再解析网盘源；若某个网盘源已成功展开出真实 `episodes`，应按规范化线路名过滤掉对应的普通占位线路，只保留展开成功后的网盘线路。
- 最终输出 `vod_play_sources` 前应按线路名合并、按 `playId` 去重，并过滤 `episodes.length === 0` 的线路，避免宿主 UI 显示多余空线路或重复线路。
- 同一个详情页中，同一个 `/play/xxx.html` 可能被多个 DOM 片段重复扫到；网盘展开播放页前应维护 `processedHrefs`/等价集合，确保同一播放页只请求一次，降低进入详情页时连续请求导致 429 的概率。
- 若用户反馈“网盘正确解析了但还有多余未解析路线”，优先检查：`extractPlaylistSources()` 是否把网盘下载按钮当普通播放线路、网盘成功线路是否反向过滤了占位线路、最终合并去重是否在 `buildLegacyPlayFields()` 之前完成。


## 8. 只把“网盘/下载”线路当网盘解析，未支持网盘要双向过滤

- LIBVIO 详情页里 `HD7播放 / 4K蓝光 / HD10播放` 等普通播放线路可能也位于与下载区相似的 DOM 结构中；不要把所有播放线路都逐个请求 `/play/...` 再尝试当网盘解析，否则会导致详情页请求暴涨，甚至触发 429。
- 网盘解析入口应先按线路标题或按钮文本筛选，只处理明确包含 `网盘` 或 `下载` 字样的线路；普通采集线路只走 `extractPlaylistSources()`，不要进入 `loadPanFiles()` 链路。
- 如果某类网盘暂未支持或解析失败（本次为 UC 网盘），不能只在网盘展开分支 `continue`；还必须从普通采集线路 `collectSources` 中同步过滤掉对应占位线路。否则它会以 `详情页#UC网盘#0` 这类普通采集 fid 进入 `processScraping()`，把“合集”刮削成 `S1E0`，并在 UI 里残留未解析线路。
- 过滤顺序建议：`allCollectSources = extractPlaylistSources()` → 解析受支持网盘得到 `netdiskSources` → 用 `parsedNetdiskSourceNames` 移除已展开占位 → 用 unsupported/source-name 黑名单（如 `/UC/i`）移除不支持网盘 → 再构造 `scrapeSourceBuckets`。这样对外展示、刮削候选、播放记录三条链路保持一致。
- 用户贴日志时，如果出现 `fid=...#UC网盘#0`、`from=合集 -> to=S1E0`，优先判断是“不支持网盘没从采集线路中过滤”，而不是刮削算法本身出错。
