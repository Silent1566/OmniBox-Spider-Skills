# WordPress / Zibll 隐藏内容评论解锁排查笔记

适用场景：资源站详情页出现“此处内容已隐藏，请评论后刷新页面查看.”，需要脚本端自动评论解锁隐藏内容。

## 关键现象

- 已带站点 Cookie，请求详情页时能命中隐藏提示。
- 早期只提交 `comment/comment_post_ID/comment_parent/_wpnonce/action=submit_comment` 到 `wp-admin/admin-ajax.php`，返回：
  - `HTTP 400`
  - body 仅为 `"0"`
- 刷新后仍 `locked`，后续分享提取 `uniqueShares=0`。

## 排查结论

1. **先确认抓到的是不是真实评论表单**
   - 优先定位：`#commentform` / `#respond form` / `form:has(textarea[name="comment"])`
   - 如果页面只有“请登录后发表评论 / 请登录后查看评论内容”，说明当前 HTML 仍是未登录态，或评论区需前端二次加载。
   - 不要把登录弹窗里的 `_wpnonce` 误当评论 nonce。

2. **若真实评论表单字段极少，不要继续盲猜 hidden input**
   - 实战里真实 `#commentform` 可能确实只包含：
     - `comment`
     - `comment_post_ID`
     - `comment_parent`
     - `_wpnonce`
   - 这时下一步应分析站点 `comment.min.js` / 主题主 JS，而不是继续猜还有哪些表单字段没带。

3. **Zibll 主题评论可能依赖提交按钮元信息**
   - 重点检查提交按钮 `#commentform #submit` 是否带：
     - `form-action`
     - `ajax-href`
     - `name/value`
     - `data-postid`
     - `data-nonce`
   - 前端常见流程：`serializeObject()` 表单 → `htmlEscapes(comment)` → `n.action = "submit_comment"` → `zib_ajax(button, n, ...)`

4. **评论正文编码要尽量对齐前端**
   - 若前端会先 `htmlEscapes(comment)`，脚本侧也建议先做 HTML 转义再提交，避免主题侧校验差异。

5. **Ajax 失败时增加标准评论接口回退**
   - 若 Ajax 返回 `400/"0"`，增加回退：
     - 优先表单 `action`
     - 否则 `wp-comments-post.php`
   - 回退后一定要重新抓详情页确认是否 `unlocked`。

## 推荐日志

至少输出：

- `hasForm/formId/action/comment_post_ID/comment_parent/nonceLen`
- `fields=...`
- `submitId/submitAction/submitAjax/submitNonceLen`
- Ajax 提交 `url/action/fieldCount`
- Ajax 响应 `status/raw`
- 是否触发标准评论回退
- 回退响应 `status/raw`
- 刷新后隐藏状态 `locked/unlocked`
- 分享提取统计 `anchorCandidates/textCandidates/uniqueShares`

## 自动签到补充

若同站点还支持每日签到：

- 接口常见为：`POST /wp-admin/admin-ajax.php`
- body：`action=user_checkin`
- 建议放在 `home()` / `search()` 高流量入口异步触发
- 使用 SDK 缓存 24 小时节流
- 失败只记日志，不影响主链路

## 敏感信息处理

- 不要在技能或总结中保存 Cookie、nonce、token 实值。
- 如需记录，只记录：是否存在、长度、是否命中、返回状态。