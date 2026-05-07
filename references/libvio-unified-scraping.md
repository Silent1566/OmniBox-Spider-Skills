# LIBVIO 统一刮削与多线路回填经验

适用场景：OmniBox 采集源详情页同时有普通采集线路与多个网盘线路（百度、夸克、UC 等），用户反馈“有些线路刮削了、有些没有”、分集仍显示原始文件名或“合集”。

## 关键原则

1. **网盘展开先于刮削**
   - 先把所有网盘入口（含 `UC网盘`、`视频下载 (百度)`、`夸克网盘`）通过播放页 `player_aaaa.url` 或页面裸分享链接展开成真实 `fileId/fileName`。
   - “合集”只是入口名，不是分集；没有展开出真实文件时该网盘线路直接排除。

2. **只调用一次 `processScraping()`**
   - 不要按线路、按网盘类型、按分集重复调用刮削。
   - 参考 `影视/网盘/木偶.js` 的模式：先构造全局候选数组，再统一调用一次：
     ```js
     await OmniBox.processScraping(videoId, keyword, keyword, scrapeCandidates);
     const metadata = await OmniBox.getScrapeMetadata(videoId);
     ```

3. **全局候选要覆盖所有需要刮削的线路**
   - 候选来源应包括：普通采集线路 + 已成功展开的所有网盘线路。
   - 网盘候选至少传：`fid`、`file_id`、`file_name`、`name`、`format_type: "video"`。
   - 网盘 `_fid` 默认使用 `{shareUrl}|{fileId}`，并确保 `playId` meta 里的 `fid` 完全一致。

4. **刮削后再统一遍历回填**
   - 对最终 `vod.vod_play_sources` 的每条线路、每个 episode 遍历。
   - 从 `playId` meta 或 `_fid` 取匹配键，查 `videoMappings[].fileId`。
   - 命中后统一更新：`ep.name`、`meta.e`、`meta.s`、`meta.n`、`meta.fid`，再重写 `playId`。
   - 不要只处理当前选中线路，也不要只处理网盘或只处理普通采集线路。

5. **避免中间桶导致漏回填**
   - `_play_sources_for_scrape` / `scrapeSourceBuckets` 可以作为内部候选来源，但最终回填必须面向最终展示的 `vod_play_sources`。
   - 如果某些线路仍显示原始文件名，优先检查：候选里是否包含该线路、候选 `file_id` 是否等于最终 episode meta.fid、回填遍历是否覆盖最终展示线路。
   - 重构旧脚本时，禁止让“旧合并/排序/返回逻辑”和“新统一刮削逻辑”并存。常见坏补丁是保留两套 `mergedPlaySourcesMap`、`vod_play_sources`、`sortedNetdiskSources` 或两个 `return result`，会直接触发重复声明或不可达旧逻辑。
   - 若补丁后 `node --check` 报 `Identifier ... has already been declared`，先读取统一刮削块前后约 200 行，清理旧块，而不是继续追加新代码。

## 建议日志

- `detail 线路统计`：`collectSourceCount / netdiskSourceCount / scrapeSourceCount / scrapeSources`
- `detail 刮削候选`：`count + fid=>file_name preview`
- `detail 刮削元数据`：`mappingCount + mappingPreview`
- `detail 分集未命中刮削映射`：`sourceName / fid / episodeName / mappingPreview`
- `detail 应用刮削分集名`：`sourceName / from / to / fid`

## 排查顺序

1. 图片/宿主 UI 显示某线路仍是原始文件名时，先看该线路是否进入 `scrapeCandidates`。
2. 若进入候选但未命中，核对 `scrapeCandidates.file_id`、episode meta.fid、`videoMappings[].fileId` 三者是否完全一致。
3. 若只某些线路回填，检查回填循环是否只遍历了某个中间 bucket，而不是最终 `vod_play_sources`。
4. 若还出现 `合集`，检查网盘失败占位是否被 `extractPlaylistSources()` 当普通采集线路送入刮削/展示。

## 实操补丁纪律

- 对 `detail()` 中段做大块替换前，先读取包含“线路提取 → 网盘展开 → 合并线路 → 刮削 → 返回 result”的完整连续代码区；避免只看 offset 片段就半截替换。
- 如果旧代码已经提前构造过 `expandedNetdiskSources` 或 `normalizedCollectSources`，统一刮削改造后要么继续复用，要么删除，不要留下无用变量误导后续维护。
- 最终排序变量建议和旧网盘排序变量区分命名，例如 `sortedFinalPlaySources / expandedFinalPlaySources`，避免与前面用于网盘预展开的 `sortedNetdiskSources / expandedNetdiskSources` 重名。
- 应在最终 `result.list[0].vod_play_sources` 指向已回填后的最终线路数组，不能仍指向刮削前的 `expandedNetdiskSources`。
- 回填日志里如果打印分享 URL / fid，必须脱敏 `pwd`、token、cookie 等敏感参数；必要时只打印短 hash 或 `fileId` 摘要。

## 对话流程教训

用户要求“参考某脚本优化、只需要刮削一次后再遍历处理每个线路”时，应直接进入 read/patch/check 实操，不要只给方案。工具调用后必须立刻处理结果并给出下一步或完成结果，不能空回复。

## 本轮事故复盘：空回复、相对路径与过早提交

- 工具调用后绝不能返回空消息。尤其在 Telegram/即时通讯场景，哪怕工具结果只是失败，也必须立刻用一句中文处理结果，例如“路径是相对当前会话的，刚才失败了；我改用仓库绝对路径继续查”。
- 如果用户连续提示“你刚才工具调用后空回复”，不要继续调用工具后沉默；下一条回复必须先承认当前状态、解释上一条工具结果、再继续实操。
- Git/MCP 工具若没有工作目录，先设置仓库根路径；普通文件工具若相对路径失败，立即改用绝对路径，不要重复同一失败参数。
- 提交 / PR 前如果还有已知待清理项（例如 `normalizedCollectSources` 这类未使用变量、未验证的 finalActor/finalDirector 等），先完成清理和 `node --check`，再 commit。不要在“继续清理中”的状态下过早提交。
- PR 链路每一步都要有可见状态：已设置工作目录、当前分支、已检查 diff、已通过静态检查、已提交 hash、已 push、PR 链接；若卡在凭据/权限/已有 PR 冲突，明确说阻塞点。

## 本次新增教训：提交 / PR 与工具结果处理

- 用户说“提交，pr”后，属于 Git / PR 交付链路，不要只阶段性执行 `git add`、`commit` 就空回复；每次工具调用后都必须把结果转成明确中文状态，并继续执行下一步直到 push + PR 完成或遇到需要用户授权/凭据的阻塞。
- 如果 Git/MCP 工具提示“未设置工作目录”，立即设置为仓库根目录（例如 `/root/.openclaw/workspace/OmniBox-Spider`），再重跑状态；不要重复用相对路径导致 `Path not found`。
- 如果提交工具报告 `success: true` 但同时有签名失败警告，按“提交已创建但未签名”处理，并在最终说明中点出；不要把签名警告当作提交失败，也不要静默忽略。
- 对 OmniBox 源做 PR 前建议顺序：`git status` → 检查 diff / 目标文件 → `node --check` → `git add` → 中文 commit → `git push` → 中文 PR 标题/正文。若当前分支是 `openclaw` 且已有打开 PR，应按 Git / PR 技能里的规则判断是否需要新建专用分支。
- 在对话被用户多次提醒“工具调用后空回复”时，这是高优先级流程缺陷：后续同类任务必须采用“工具结果摘要 + 下一步动作”的节奏，即便还没完成，也要明确当前完成了哪一步、下一步正在做什么，并实际继续调用工具。
