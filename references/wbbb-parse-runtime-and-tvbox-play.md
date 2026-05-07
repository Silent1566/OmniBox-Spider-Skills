# 歪比巴卜 / 类似解析站：运行时解密 + TVBox 播放返回实战

适用场景：
- 详情页出现伪线路名，如 `排序`、`选择播放源`
- 播放页通过外部解析站 `/player/?url=...` + `api.php` 获取真实地址
- `api.php` 已返回 `code:200` 和 `type:m3u8/mp4`，但宿主仍报 `sdk sniff on resolved failed`

## 1. 先判定是不是“解析请求已成功，但脚本自己没把真实媒体地址解出来”
若日志出现：
- `[resolve][payload] {"code":200,...,"type":"m3u8"|"mp4"}`
- 但 `[play][resolved] url=` 仍是 `https://.../player/?url=...`

说明问题通常不在请求构造，而在 **payload.url 的解密链**。

## 2. 不要假设 `payload.url` 只需直接 AES 解密
这类站常见真实链路是：
1. 先执行解析页 `setting.js`
2. 运行时暴露：`urlValueurl / keyValue / vkeyValue / ckeyValue`
3. 同时还会定义前端函数：`decrypt()`、`deplay()`
4. `payload.aes_key / payload.aes_iv` 往往还要先过 `deplay()`，才能得到真正可用于 AES-CBC 的 key/iv

更稳妥的默认顺序：
1. 优先直接调用 `runtime.decrypt(payload.url)`
2. 若失败，再尝试：
   - `decodedKey = runtime.deplay(payload.aes_key)`
   - `decodedIv = runtime.deplay(payload.aes_iv)`
   - 再用 `decodedKey/decodedIv` 做 AES-CBC-Pkcs7
3. 最后才回退到简单 `Base64/Utf8 parse -> CryptoJS.AES.decrypt`

## 3. `getWbbbResolveRuntime()` 这类 helper 不要只返回参数，还要返回运行时函数
推荐把 VM sandbox 中的函数一并挂出：
- `decrypt`
- `deplay`

否则即使拿到了 `keyValue/vkeyValue/ckeyValue`，后续也可能仍解不开真实媒体 URL。

## 4. TVBox / 非 app-sniff 宿主的返回策略
若 `resolve` 已拿到真实媒体地址：
- `type` 为 `m3u8/mp4`
- 或 URL 已明显是媒体直链

优先直接返回：
```js
{
  parse: 0,
  jx: 0,
  url: resolved.url,
  playId,
  header,
}
```

不要先强行 `sniffVideo(resolved.url)`；否则宿主可能报：
- `sdk sniff on resolved failed: 嗅探失败: run video_sniffer: exit status 1`

更稳妥链路：
1. resolve 成功且是媒体直链 -> 直接返回
2. resolve 成功但仍像网页/解析页 -> 再尝试 `sniffVideo`
3. sniff 失败 -> 再 fallback `parse:1`

## 5. 识别“伪线路 tab”
若详情页 tab 数量明显多于真实播放组，优先检查是否有 UI 控件混入线路：
- `排序`
- `选择播放源`
- `更多`
- `展开`
- `收起`

处理要点：
- `parseTabs()` 阶段直接过滤
- 之后再与真实 `groups` 对齐构造 `vod_play_sources`
- 同时兼容返回：
  - `vod_play_from`
  - `vod_play_url`
  - `vod_play_list`
  - `vod_play_sources`

## 6. 若解析 payload 已成功，排查顺序要切到“最终返回结构”
不要继续反复怀疑：
- API 路径大小写
- POST headers
- next 参数

当这些都已对上抓包且 `code:200` 时，下一步默认先看：
- `resolve.final.url` 到底是不是媒体直链
- `play()` 是不是仍把解析页误判成直链
- `play()` 是否还在 resolved 分支里继续 sniff 并失败

## 7. 建议补的日志
至少补：
- `[resolve][payload]`：脱敏后的 type/url 信息
- `[resolve][final]`：最终解出的 URL 是否仍是解析页
- `[play][resolved]`：resolved url、type、parsePage
- `[play][fallback]`：最终回退对象

所有 token/key/vkey/ckey/Cookie/真实媒体签名参数都必须脱敏。