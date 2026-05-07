# FlareSolverr CF 盾绕过 — 会话级参考

> 来源：2026-04-06 升级《玩偶.js》支持 CF 盾 + 日志增强
> 参考源：`PPnix.js`（单域名）、`玩偶.js`（多域名容灾 + CF 盾）

## 🔴 关键教训：`WOOG_FLARESOLVERR_SESSION` 必须给默认值

**问题**：默认空字符串会导致每次 FlareSolverr 请求都启动一个全新的浏览器实例，cookie 和 UA 无法跨请求复用，CF 绕过几乎必败。

**正确做法**：
```js
// ❌ 错误（空默认）
const WOOG_FLARESOLVERR_SESSION = process.env.WOOG_FLARESOLVERR_SESSION || "";

// ✅ 正确（固定 session 名）
const WOOG_FLARESOLVERR_SESSION = process.env.WOOG_FLARESOLVERR_SESSION || "wogg";
```

FlareSolverr 收到 `session` 参数后会保持同一浏览器上下文，后续同域名请求直接复用已解开的 cookie，成功率大幅提升。

## 生态现状

玩偶系列站点（wogg.xxooo.cf / wogg.333232.xyz / www.wogg.net）中部分域名有 CF 盾保护。
FlareSolverr 能够成功获取 `cf_clearance` cookie，但**获取到 cookie 不代表后续请求一定通过 CF 盾**：
- 如果 UA 不一致，CF 可能仍拦截
- 如果返回的 `cf_clearance` 域不完全匹配，仍可能无效
- 部分站点可能有额外 JS 检查或会话绑定

## 日志流（正常绕过成功）

```
[INFO] 尝试请求域名 1/3: https://wogg.xxooo.cf/, timeout=10000ms
[WARN] [cf] https://wogg.xxooo.cf/ 被CF盾拦截，尝试通过 FlareSolverr 获取 cf_clearance
[INFO] [cf] FlareSolverr 返回结果: status=ok, cookies=2, cookie预览=cf_clearance=abcd..., ua=Mozilla/5..., 执行耗时=1320ms
[INFO] [cf] 已获取 cf_clearance 长度为 68，重试 https://wogg.xxooo.cf/
[INFO] [cf] 携带 cf_clearance 重试后成功: https://wogg.xxooo.cf/, status=200, body长度=44123
[INFO] 域名 https://wogg.xxooo.cf/ 请求成功
```

## 日志流（FlareSolverr 成功但 cookie 仍无效，切到下一个域名）

```
[INFO] 尝试请求域名 1/3: https://wogg.xxooo.cf/, timeout=10000ms
[WARN] [cf] https://wogg.xxooo.cf/ 被CF盾拦截，尝试通过 FlareSolverr 获取 cf_clearance
[INFO] [cf] FlareSolverr 返回结果: status=ok, cookies=2, cookie预览=cf_clearance=abcd..., ua=Mozilla/5..., 执行耗时=1320ms
[INFO] [cf] 已获取 cf_clearance 长度为 68，重试 https://wogg.xxooo.cf/
[WARN] [cf] 携带 cf_clearance 重试后仍被CF盾拦截: https://wogg.xxooo.cf/, status=200, body预览=<!DOCTYPE html>...just a moment...
[WARN] 域名 https://wogg.xxooo.cf 命中风控页,切换下一个域名
[WARN] 域名 https://wogg.333232.xyz 请求失败: getaddrinfo ENOTFOUND
[INFO] 尝试请求域名 3/3: https://www.wogg.net/, timeout=10000ms
[INFO] 域名 https://www.wogg.net/ 请求成功
```

## 已知问题定位方向

| 日志现象 | 原因诊断 | 建议操作 |
|---|---|---|
| FlareSolverr 返回 `status=ok` 但重试后仍被 CF 拦截 | 可能 UA 不匹配，或 `cf_clearance` 域不对 | 同步 `solution.userAgent` 到重试请求的 `User-Agent` 头 |
| FlareSolverr 返回 `status!=ok` 或 非 200 | FlareSolverr 服务异常或被限流 | 检查 FlareSolverr 日志 |
| FlareSolverr 返回的 cookies 数组中无 `cf_clearance` | 目标站点可能不是标准 CF 防护，或 FlareSolverr 未完成验证 | 检查 `res.data.message` 内容 |
| 有 `cf_clearance` 但重试仍 CF 拦截 | 站点使用 CF Under Attack 模式或 JS 挑战（非 cookie 级别） | 需 headless browser 方案 |

## 多域名场景下的 CF 集成原则

1. FlareSolverr 获取 cookie 后只在**同一域名**重试一次
2. 重试失败后**不要影响容灾流程**（继续尝试下一个域名）
3. Cookie 缓存（SDK `getCache/setCache`，默认 6h TTL）跨所有域名复用
4. 因为不同域名的 `cf_clearance` 域不同，下次切到另一个域名时如果该域名也有 CF 盾，会**再次**触发 FlareSolverr
