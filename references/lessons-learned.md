# OmniBox Spider 实战经验沉淀

## 1. 最小改动原则
- 优先复制副本修改，不动源文件。
- 一次只解决一个明确目标，避免顺手重构。
- patch 前先确认锚点存在，避免错插到 search/catch 等错误位置。

## 2. 静态检查只是最低标准
- JavaScript 修改后必须执行 `node --check`。
- Python 修改后必须执行 `python3 -m py_compile`。
- 语法通过不等于运行正确，关键路径仍要额外核对。

## 3. 关键路径回归检查
改完后至少核对：
- `detail()` 能正常返回 `list: [vodDetail]`
- `vod_play_sources` 在有数据时不为空
- `play()` 能返回 `urls`
- 关键环境变量没有被误删（如 `SOURCE_NAMES_CONFIG`）

## 4. const / let 风险检查
- 可能被重新赋值的变量必须用 `let`，不要误用 `const`。
- 尤其注意：`playSources`、`vodName`、`vodPic` 这类后续可能会改写的变量。

## 5. 排序逻辑规则
- 排序只放在 `detail()` 最终返回前。
- 只改 `playSources` 顺序，不改 `episodes` 结构。
- `DRIVE_ORDER` 只用于详情页线路排序，不用于搜索结果排序。

## 6. 并发策略
- `play()` 可把 `getDriveVideoPlayInfo` 与元数据/弹幕链路并行。
- `detail()` 不要全量 `Promise.all` 打满，默认改为限并发（建议 4）。
- 元数据/弹幕失败不应阻塞播放主链路。

## 7. 交付说明模板
每次交付至少说明：
1. 修改的文件路径
2. 版本号变更（旧 -> 新）
3. 静态检查结果
4. 本次核心改动点

## 8. Web 播放链路优先保 URL 完整性
- 对 OmniBox 网页端（`context.from === "web"`）的播放链路，优先保证 URL 在 `detail -> play -> 前端` 全链路中完整透传。
- 如果播放页 / 直链自带 `&token`、`&time`、`sign` 等查询参数，不要裸传到 `playId`；优先先编码（如 `encodeURIComponent`），到 `play()` 再解码。
- 网页端很多时候不会可靠消费返回的 `header`；因此不要把 `Referer` / `UA` / `Origin` 当成 web 端成功播放的唯一前提。
- 若站点强依赖 header，优先考虑：
  1. 在 `play()` 内提前解出最终可播直链
  2. 或走代理层，而不是假设网页端一定会带你返回的 header

## 9. GitHub PR 正文排版
- 用 `gh pr create` / `gh pr edit` 传正文时，必须使用真实换行，不要把字面 `\n` 当成换行提交。
- 稳妥做法：优先写入临时 `.md` 文件，再用 `--body-file` 传给 GitHub CLI。
- PR 标题、正文、交付说明默认优先中文，除非上游仓库明确偏好英文。
- 如果是修 OmniBox 播放链路问题，PR 描述里要明确写清：问题出现在哪一段链路（如 `playId` 透传 / web header / 嗅探兜底），避免 reviewer 只看到“修播放”这种过于笼统的表述。
