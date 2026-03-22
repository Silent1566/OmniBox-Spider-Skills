# OmniBox JavaScript 爬虫模板（完整注释版）

这个模板适合“普通采集站 / API 站点”接入 OmniBox。结构上分为：
- 脚本头注释属性
- SDK 与 runner 引入
- 站点请求封装
- 公共字段映射函数
- 五个标准 handler：`home / category / detail / search / play`

---

## 完整模板

```javascript
// @name 爬虫名称
// @version 1.0.0
// @downloadURL https://example.com/spider.js
// @indexs 1
// @dependencies axios,cheerio

/**
 * 作用：引入 OmniBox SDK 与脚本运行器。
 * - omnibox_sdk：提供请求、日志、网盘、刮削、弹幕等能力
 * - spider_runner：负责从 stdin 读取 method/params/context 并分发到对应 handler
 */
const OmniBox = require("omnibox_sdk");
const runner = require("spider_runner");

/**
 * 作用：导出当前脚本支持的 handler。
 * 注意：
 * - 常规影视源建议实现 5 个方法
 * - 推送型脚本通常只需 detail + play
 */
module.exports = { home, category, detail, search, play };

/**
 * 作用：启动 runner。
 * 行为：
 * - 从标准输入读取 OmniBox 发来的 method / params / context
 * - 自动调用同名 handler(params, context)
 */
runner.run(module.exports);

/**
 * 作用：站点 API 基础地址。
 * 来源：环境变量 SITE_API。
 * 推荐：
 * - 在 OmniBox 后台按源配置环境变量
 * - 不要把真实接口地址硬编码到公开脚本里
 */
const SITE_API = process.env.SITE_API || "";

/**
 * 函数：requestApi
 * 作用：统一发送站点请求，并自动处理：
 * - 参数拼接
 * - 请求头
 * - 状态码判断
 * - JSON 解析
 *
 * 入参：
 * @param {Object} params - 查询参数对象，例如 { ac: "list", pg: "1" }
 *
 * 出参：
 * @returns {Promise<Object>} - 解析后的 JSON 对象
 *
 * 常见异常：
 * - SITE_API 未配置
 * - HTTP 状态码不是 200
 * - body 不是合法 JSON
 */
async function requestApi(params = {}) {
  if (!SITE_API) throw new Error("请配置 SITE_API 环境变量");

  // 过滤空参数，避免把 null / undefined / 空字符串拼进 URL
  const qs = new URLSearchParams(
    Object.fromEntries(Object.entries(params).filter(([, v]) => v != null && v !== ""))
  ).toString();

  // 规范化接口地址，避免重复斜杠
  const url = `${SITE_API.replace(/\/$/, "")}?${qs}`;
  await OmniBox.log("info", `请求: ${url}`);

  const res = await OmniBox.request(url, {
    method: "GET",
    headers: {
      "User-Agent": "Mozilla/5.0",
    },
  });

  if (res.statusCode !== 200) {
    throw new Error(`HTTP ${res.statusCode}: ${res.body}`);
  }

  return JSON.parse(res.body || "{}");
}

/**
 * 函数：mapVod
 * 作用：把第三方站点返回的视频对象映射为 OmniBox 标准 VodItem。
 *
 * 入参：
 * @param {Object} v - 站点原始视频对象
 *
 * 出参：
 * @returns {Object} OmniBox 标准视频项
 *
 * 说明：
 * - 这里只处理最常见字段
 * - 若站点字段名不同，可在这里统一兼容
 */
function mapVod(v) {
  return {
    vod_id: String(v.vod_id || ""),
    vod_name: String(v.vod_name || ""),
    vod_pic: String(v.vod_pic || ""),
    type_id: String(v.type_id || ""),
    type_name: String(v.type_name || ""),
    vod_remarks: String(v.vod_remarks || ""),
    vod_year: String(v.vod_year || ""),
    vod_douban_score: String(v.vod_douban_score || ""),
  };
}

/**
 * 函数：home
 * 作用：获取首页数据。
 *
 * 入参：
 * @param {Object} params - 一般为空对象
 * @param {Object} context - 请求上下文，常见字段：
 *   - context.baseURL
 *   - context.headers
 *   - context.sourceId
 *   - context.from
 *
 * 出参：
 * @returns {Promise<{class: Array, list: Array}>}
 *
 * 返回结构：
 * - class: 分类列表
 * - list: 推荐视频列表
 *
 * 失败兜底：
 * - 返回 { class: [], list: [] }
 */
async function home(params, context) {
  try {
    const from = context?.from || "web";
    await OmniBox.log("info", `[home] from=${from}`);

    const data = await requestApi({ ac: "list", pg: "1" });

    return {
      class: (data.class || []).map(c => ({
        type_id: String(c.type_id),
        type_name: String(c.type_name),
      })),
      list: (data.list || []).map(mapVod),
    };
  } catch (e) {
    await OmniBox.log("error", `[home] 失败: ${e.message}`);
    return { class: [], list: [] };
  }
}

/**
 * 函数：category
 * 作用：获取分类分页列表。
 *
 * 入参：
 * @param {Object} params
 * - params.categoryId {string} 分类 ID
 * - params.page {number} 页码
 * - params.filters {Object} 筛选条件（可选）
 * @param {Object} context - 请求上下文
 *
 * 出参：
 * @returns {Promise<{page:number,pagecount:number,total:number,list:Array}>}
 *
 * 失败兜底：
 * - 返回空分页结构
 */
async function category(params, context) {
  try {
    const { categoryId = "1", page = 1, filters = {} } = params;
    await OmniBox.log("info", `[category] categoryId=${categoryId}, page=${page}, filters=${JSON.stringify(filters)}`);

    const data = await requestApi({ ac: "videolist", t: categoryId, pg: String(page) });

    return {
      page: Number(data.page) || page,
      pagecount: Number(data.pagecount) || 0,
      total: Number(data.total) || 0,
      list: (data.list || []).map(mapVod),
    };
  } catch (e) {
    await OmniBox.log("error", `[category] 失败: ${e.message}`);
    return { page: 1, pagecount: 0, total: 0, list: [] };
  }
}

/**
 * 函数：detail
 * 作用：获取视频详情，并构建播放源结构。
 *
 * 入参：
 * @param {Object} params
 * - params.videoId {string} 视频 ID
 * @param {Object} context - 请求上下文
 *
 * 出参：
 * @returns {Promise<{list:Array}>}
 *
 * 关键点：
 * - OmniBox 详情页通常要求返回 list 数组
 * - 如果站点返回 vod_play_url，需要拆成 episodes
 * - vod_play_sources 是结构化播放线路
 */
async function detail(params, context) {
  try {
    const { videoId } = params;
    if (!videoId) return { list: [] };

    await OmniBox.log("info", `[detail] videoId=${videoId}`);
    const data = await requestApi({ ac: "detail", ids: videoId });

    const list = (data.list || []).map(v => {
      const episodes = [];

      /**
       * 说明：
       * 某些采集站会把播放地址写成：
       * "第1集$playId1#第2集$playId2"
       * 这里需要把它拆成 OmniBox episodes 结构。
       */
      if (v.vod_play_url) {
        v.vod_play_url.split("#").forEach((seg, i) => {
          const parts = seg.trim().split("$");
          episodes.push(
            parts.length >= 2
              ? { name: parts[0].trim(), playId: parts.slice(1).join("$").trim() }
              : { name: `第${i + 1}集`, playId: parts[0].trim() }
          );
        });
      }

      return {
        ...mapVod(v),
        vod_content: String(v.vod_content || ""),
        vod_actor: String(v.vod_actor || ""),
        vod_director: String(v.vod_director || ""),
        vod_area: String(v.vod_area || ""),
        vod_play_sources: episodes.length > 0 ? [{ name: "线路1", episodes }] : undefined,
      };
    });

    return { list };
  } catch (e) {
    await OmniBox.log("error", `[detail] 失败: ${e.message}`);
    return { list: [] };
  }
}

/**
 * 函数：search
 * 作用：根据关键词搜索视频。
 *
 * 入参：
 * @param {Object} params
 * - params.keyword {string} 搜索关键词
 * - params.wd {string} 某些调用端可能使用 wd
 * - params.page {number} 页码
 * @param {Object} context - 请求上下文
 *
 * 出参：
 * @returns {Promise<{page:number,pagecount:number,total:number,list:Array}>}
 */
async function search(params, context) {
  try {
    const keyword = params.keyword || params.wd || "";
    const page = params.page || 1;

    if (!keyword) {
      return { page: 1, pagecount: 0, total: 0, list: [] };
    }

    await OmniBox.log("info", `[search] keyword=${keyword}, page=${page}`);
    const data = await requestApi({ ac: "list", wd: keyword, pg: String(page) });

    return {
      page: Number(data.page) || page,
      pagecount: Number(data.pagecount) || 0,
      total: Number(data.total) || 0,
      list: (data.list || []).map(mapVod),
    };
  } catch (e) {
    await OmniBox.log("error", `[search] 失败: ${e.message}`);
    return { page: 1, pagecount: 0, total: 0, list: [] };
  }
}

/**
 * 函数：play
 * 作用：根据 playId 返回最终播放信息。
 *
 * 入参：
 * @param {Object} params
 * - params.playId {string} 播放 ID 或真实地址
 * - params.flag {string} 播放源标识，默认 play
 * @param {Object} context - 请求上下文
 *
 * 出参：
 * @returns {Promise<{urls:Array,flag:string,header:Object,parse:number}>}
 *
 * 说明：
 * - 若 playId 已经是 m3u8/mp4，可直接视为直链，parse=0
 * - 若 playId 只是中转页 / 解析页，可设 parse=1
 * - 但 parse=1 只有 ok影视 app 真正支持
 */
async function play(params, context) {
  try {
    const { playId, flag = "play" } = params;
    if (!playId) throw new Error("playId 不能为空");

    const parse = /\.(m3u8|mp4)$/i.test(playId) ? 0 : 1;
    await OmniBox.log("info", `[play] playId=${playId}, flag=${flag}, parse=${parse}`);

    return {
      urls: [{ name: "播放", url: playId }],
      flag,
      header: {},
      parse,
    };
  } catch (e) {
    await OmniBox.log("error", `[play] 失败: ${e.message}`);
    return { url: "", flag: params.flag || "play", header: {} };
  }
}
```

---

## 模板使用说明

### 1. 这是最适合起步的“普通 API 采集站模板”
如果目标站点：
- 有分类接口
- 有详情接口
- 有搜索接口
- 有播放地址字段

那可以直接基于这个模板改。

### 2. 你通常要改的地方
- `SITE_API`
- `requestApi()` 的认证 / 参数签名逻辑
- `mapVod()` 的字段映射
- `detail()` 里 `vod_play_url` 的拆分规则
- `play()` 里是否需要嗅探 / 解析站 / 请求头

### 3. 常见扩展点
- 加 `banner`
- 加 `filters`
- 加 `danmaku`
- 加刮削元数据映射
- 加网盘展开逻辑

### 4. 常见错误
- 忘记 `runner.run(module.exports)`
- `requestApi()` 里没处理 403/500
- 把 JSON 字符串当对象直接用了
- `detail()` 没返回 `list`
- `play()` 直接返回错结构
