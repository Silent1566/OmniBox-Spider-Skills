# WordPress / Zibll 隐藏内容与评论解锁排查

适用场景：OmniBox 源遇到 `此处内容已隐藏，请评论后刷新页面查看.`、`reply-show`、`hidden-box` 这类 WordPress / Zibll 主题隐藏内容。

## 关键结论

1. 不要默认认为“提交评论成功 = 立即解锁”。
   - Zibll 前端评论逻辑会检查返回里的 `t.comment.comment_approved`。
   - 只有 `comment_approved > 0` 时，前端才会刷新页面并显示隐藏内容。
   - 否则前端会提示：`审核后通过后即可查看隐藏内容`。

2. 如果详情页里能拿到 `#commentform`，也不要只盯固定字段名。
   - 先提取整张评论 form 的 `input/textarea/select[name]`。
   - 再额外记录提交按钮元信息：
     - `id`
     - `form-action`
     - `ajax-href`
     - `name/value`
     - `data-postid`
     - `data-nonce`

3. Zibll 评论 Ajax 常走：
   - `#commentform #submit`
   - `serializeObject()`
   - `comment = htmlEscapes(comment)`
   - `action = "submit_comment"`
   - `zib_ajax(...)`

4. 若 Ajax 返回 `HTTP 400` 且 body 只有 `"0"`：
   - 先怀疑 Ajax 链路不适配当前环境；
   - 可回退尝试标准 `wp-comments-post.php`。
   - 但如果用户已经抓到“浏览器里可直接解锁”的真实 `admin-ajax.php` 评论请求，默认优先**逐字段对齐真实请求**，不要继续盲目提交整张 form。
   - 对这类 Zibll / WordPress 主题，实战里更稳的 Ajax 评论体常常只有：`comment`、`comment_post_ID`、`comment_parent`、`_wpnonce`、`action=submit_comment`；额外混入整张表单字段、submit 按钮字段、或把评论正文再做 HTML escape，反而可能触发 `400/0`。

5. 若回退评论：
   - `status=200` 且 `raw=""`，不一定代表失败；
   - 继续检查响应头中的：
     - `location`
     - `set-cookie`
   - 若出现 `unapproved=` / `moderation-hash=` / `#comment-数字`，大概率是“评论已受理但待审核”。

6. 站点自动签到不要只发 `action=user_checkin`。
   - 先从页面全局脚本提取 `post_action_nonce`；
   - 再提交 `action + _wpnonce + post_action_nonce`；
   - 否则常见表现是 `HTTP 400 code=0 msg=`。
   - 若用户要求避免刷接口，**签到失败也应写入 24 小时缓存**；后续日志要能区分 `命中成功缓存` 与 `命中失败缓存`。

7. 如果浏览器里同一条评论 Ajax `curl` 明确可用，但脚本侧仍持续 `400 raw="0"`，优先把问题收敛到 **请求栈差异** 与 **Cookie 完整性**，不要继续盲猜字段。
   - 先把脚本日志里的 `cookieLen` 与浏览器成功 `curl -b '...'` 的实际长度对比；若明显偏短（实战里出现过 `343 vs 540`），优先判断为**宿主实际吃到的 `GUANGYA_COOKIE` 不是浏览器成功请求那份完整 Cookie**。
   - 在这种场景下，`cookieLen` 的变化比继续微调评论 body 更有价值；只有当脚本侧长度接近浏览器成功请求后，再继续怀疑其它因素。
   - 如果怀疑是 Hermes/宿主内置 `request` 与浏览器行为差异导致，可把关键链路（详情 HTML、签到、评论 Ajax、评论回退、上游 JSON 接口）统一切到 `axios`：补 `@dependencies axios`、`const axios = require("axios")`，并用一个共享 `axios.create({ timeout, validateStatus: () => true, responseType: "arraybuffer", maxRedirects: 0 })` + `requestViaAxios()` 封装来替代分散的宿主请求实现。

11. 如果用户已经提供“浏览器里可成功解锁”的完整 `curl`，默认要把**浏览器成功请求的 Cookie 长度/键集合**与脚本运行日志做对比。
   - 若脚本日志里的 `cookieLen` 明显小于浏览器成功请求（本次实战是 `343` vs `540`），优先判断为：**宿主实际读到的 `GUANGYA_COOKIE` 不完整、未刷新、或根本没吃到最新环境变量**。
   - 此时不要继续盲改评论字段、nonce 或 header；先让用户确认宿主实际运行时读到的是不是那条完整 Cookie。
   - 更稳的日志做法是同时打印：`cookieLen`、Cookie key 预览（只打 key，不打 value）、以及当前 UA 预览。

12. 自动签到如果用户明确要求“失败也别重复打接口”，要把**失败态**也写入 24 小时缓存，而不是只缓存成功态；并补一条命中失败缓存日志，方便区分“今日已成功签到”还是“今日失败后已静默跳过”。

## 推荐日志

- 评论表单：`hasForm / formId / action / fields / submitId / submitAction / submitAjax / submitNonceLen`
- Ajax 评论：`url / action / fieldCount / fields / status / raw`
- 标准评论回退：`url / fieldCount / status / location / setCookie / raw`
- 刷新后：`locked|unlocked`
- 若疑似待审核：明确打印“仅在 comment_approved>0 时自动解锁，当前评论可能待审核”。

## 诊断顺序

1. 先判断页面是否仍含 `reply-show` / `hidden-box` / 隐藏提示文案。
2. 再确认拿到的是否是 `#commentform` 而不是登录表单。
3. 先试 Zibll Ajax 评论；失败再试 `wp-comments-post.php` 回退。
4. 如果回退后仍锁定，不要立刻改分享提取；先判断是否待审核。
5. 只有确认页面已真正 `unlocked` 后，再继续排查分享链接提取。
