# OmniBox Python 爬虫模板（完整注释版）

这个模板适合“普通采集站 / API 站点”接入 OmniBox。结构上分为：
- 脚本头注释属性
- SDK 与 runner 引入
- 请求封装
- 公共字段映射
- 五个标准 handler：`home / category / detail / search / play`

---

## 完整模板

```python
# -*- coding: utf-8 -*-
# @name 爬虫名称
# @version 1.0.0
# @downloadURL https://example.com/spider.py
# @indexs 1
# @dependencies requests,beautifulsoup4

import json
import os
from urllib.parse import urlencode
from spider_runner import OmniBox, run

# 作用：站点 API 基础地址。
# 建议通过 OmniBox 后台环境变量配置，而不是写死在脚本里。
SITE_API = os.environ.get("SITE_API", "")


async def request_api(params=None):
    """
    作用：统一发送站点请求，并处理：
    - 参数拼接
    - 状态码判断
    - JSON 解析

    入参：
        params (dict | None): 查询参数对象

    出参：
        dict: 解析后的 JSON 数据

    异常：
        - SITE_API 未配置
        - HTTP 状态码异常
        - 返回体不是合法 JSON
    """
    if params is None:
        params = {}
    if not SITE_API:
        raise ValueError("请配置 SITE_API 环境变量")

    qs = {k: v for k, v in params.items() if v is not None and v != ""}
    query = urlencode(qs)
    url = f"{SITE_API.rstrip('/')}?{query}" if query else SITE_API.rstrip("/")

    await OmniBox.log("info", f"请求: {url}")
    res = await OmniBox.request(url, {
        "method": "GET",
        "headers": {"User-Agent": "Mozilla/5.0"},
    })

    if res.get("statusCode") != 200:
        raise RuntimeError(f"HTTP {res.get('statusCode')}: {res.get('body', '')}")

    return json.loads(res.get("body", "{}"))


def to_int(v):
    """
    作用：安全地把各种数字类型转换为 int。

    入参：
        v: 任意值，可能是 str / int / float / None

    出参：
        int
    """
    try:
        return int(float(v)) if isinstance(v, str) else int(v)
    except (ValueError, TypeError):
        return 0


def map_vod(v):
    """
    作用：把站点原始视频对象映射为 OmniBox 标准 VodItem。

    入参：
        v (dict): 原始视频对象

    出参：
        dict: 标准视频字段
    """
    return {
        "vod_id": str(v.get("vod_id", "")),
        "vod_name": str(v.get("vod_name", "")),
        "vod_pic": str(v.get("vod_pic", "")),
        "type_id": str(v.get("type_id", "")),
        "type_name": str(v.get("type_name", "")),
        "vod_remarks": str(v.get("vod_remarks", "")),
        "vod_year": str(v.get("vod_year", "")),
        "vod_douban_score": str(v.get("vod_douban_score", "")),
    }


async def home(params, context):
    """
    作用：获取首页数据。

    入参：
        params (dict): 一般为空对象
        context (dict): 请求上下文，常见字段：
            - baseURL
            - headers
            - sourceId
            - from

    出参：
        dict: {
            "class": 分类列表,
            "list": 推荐视频列表
        }
    """
    try:
        from_val = (context or {}).get("from", "web")
        await OmniBox.log("info", f"[home] from={from_val}")

        data = await request_api({"ac": "list", "pg": "1"})
        return {
            "class": data.get("class") or [],
            "list": [map_vod(v) for v in data.get("list") or []],
        }
    except Exception as e:
        await OmniBox.log("error", f"[home] 失败: {e}")
        return {"class": [], "list": []}


async def category(params, context):
    """
    作用：获取分类分页列表。

    入参：
        params (dict):
            - categoryId: 分类 ID
            - page: 页码
            - filters: 筛选条件（可选）
        context (dict): 请求上下文

    出参：
        dict: {
            "page": 当前页,
            "pagecount": 总页数,
            "total": 总数,
            "list": 视频列表
        }
    """
    try:
        category_id = params.get("categoryId", "1")
        page = params.get("page") or 1
        filters = params.get("filters") or {}

        await OmniBox.log("info", f"[category] categoryId={category_id}, page={page}, filters={filters}")
        data = await request_api({"ac": "videolist", "t": category_id, "pg": str(page)})

        return {
            "page": to_int(data.get("page")),
            "pagecount": to_int(data.get("pagecount")),
            "total": to_int(data.get("total")),
            "list": [map_vod(v) for v in data.get("list") or []],
        }
    except Exception as e:
        await OmniBox.log("error", f"[category] 失败: {e}")
        return {"page": 1, "pagecount": 0, "total": 0, "list": []}


async def detail(params, context):
    """
    作用：获取视频详情，并构建结构化播放源。

    入参：
        params (dict):
            - videoId: 视频 ID
        context (dict): 请求上下文

    出参：
        dict: {
            "list": [
                {
                    "vod_id": ...,
                    "vod_name": ...,
                    "vod_play_sources": [...]
                }
            ]
        }
    """
    try:
        video_id = params.get("videoId")
        if not video_id:
            return {"list": []}

        await OmniBox.log("info", f"[detail] videoId={video_id}")
        data = await request_api({"ac": "detail", "ids": video_id})
        result = []

        for v in data.get("list") or []:
            episodes = []
            play_url = v.get("vod_play_url", "")

            # 某些采集站格式：第1集$playId1#第2集$playId2
            if play_url:
                for i, seg in enumerate(play_url.split("#")):
                    parts = seg.strip().split("$")
                    if len(parts) >= 2:
                        episodes.append({
                            "name": parts[0].strip(),
                            "playId": "$".join(parts[1:]).strip()
                        })
                    elif parts[0]:
                        episodes.append({
                            "name": f"第{i+1}集",
                            "playId": parts[0].strip()
                        })

            item = map_vod(v)
            item.update({
                "vod_content": str(v.get("vod_content", "")),
                "vod_actor": str(v.get("vod_actor", "")),
                "vod_director": str(v.get("vod_director", "")),
                "vod_area": str(v.get("vod_area", "")),
            })

            if episodes:
                item["vod_play_sources"] = [{"name": "线路1", "episodes": episodes}]

            result.append(item)

        return {"list": result}
    except Exception as e:
        await OmniBox.log("error", f"[detail] 失败: {e}")
        return {"list": []}


async def search(params, context):
    """
    作用：根据关键词搜索视频。

    入参：
        params (dict):
            - keyword: 搜索词
            - wd: 某些调用端的搜索参数名
            - page: 页码
        context (dict): 请求上下文

    出参：
        dict: {
            "page": 当前页,
            "pagecount": 总页数,
            "total": 总数,
            "list": 搜索结果
        }
    """
    try:
        keyword = (params.get("keyword") or params.get("wd") or "").strip()
        page = params.get("page") or 1

        if not keyword:
            return {"page": 1, "pagecount": 0, "total": 0, "list": []}

        await OmniBox.log("info", f"[search] keyword={keyword}, page={page}")
        data = await request_api({"ac": "list", "wd": keyword, "pg": str(page)})

        return {
            "page": to_int(data.get("page")),
            "pagecount": to_int(data.get("pagecount")),
            "total": to_int(data.get("total")),
            "list": [map_vod(v) for v in data.get("list") or []],
        }
    except Exception as e:
        await OmniBox.log("error", f"[search] 失败: {e}")
        return {"page": 1, "pagecount": 0, "total": 0, "list": []}


async def play(params, context):
    """
    作用：根据 playId 返回播放信息。

    入参：
        params (dict):
            - playId: 播放 ID 或真实地址
            - flag: 播放源标识
        context (dict): 请求上下文

    出参：
        dict: {
            "urls": [{"name": "播放", "url": ...}],
            "flag": ...,
            "header": {},
            "parse": 0/1
        }

    说明：
        - 若 playId 已是 m3u8/mp4，通常 parse=0
        - 若 playId 是中转页，通常 parse=1
        - 但 parse=1 只有 ok影视 app 真正支持
    """
    try:
        play_id = params.get("playId")
        flag = params.get("flag") or "play"
        if not play_id:
            raise ValueError("playId 不能为空")

        parse = 0 if play_id.endswith((".m3u8", ".mp4")) else 1
        await OmniBox.log("info", f"[play] playId={play_id}, flag={flag}, parse={parse}")

        return {
            "urls": [{"name": "播放", "url": play_id}],
            "flag": flag,
            "header": {},
            "parse": parse,
        }
    except Exception as e:
        await OmniBox.log("error", f"[play] 失败: {e}")
        return {"url": "", "flag": params.get("flag", ""), "header": {}}


if __name__ == "__main__":
    # 作用：注册并启动所有 handler
    run({"home": home, "category": category, "detail": detail, "search": search, "play": play})
```

---

## 模板使用说明

### 1. 适用场景
适合：
- 普通 API 采集站
- 字段结构较稳定的影视接口
- 需要快速起步的 Python 版本源

### 2. 常改位置
- `SITE_API`
- `request_api()` 内的认证 / 签名逻辑
- `map_vod()` 字段映射
- `detail()` 中播放链接拆分逻辑
- `play()` 中的直链 / 嗅探 / 请求头处理

### 3. 常见扩展点
- 加 `filters`
- 加 `banner`
- 加 `danmaku`
- 加 `刮削元数据映射`
- 加 `网盘目录展开`
- 加 `加密/签名` 逻辑

### 4. 常见错误
- 忘记 `run({...})`
- 没处理 `statusCode != 200`
- `body` 没 `json.loads()`
- `detail()` 没返回 `list`
- `play()` 返回结构不符合 OmniBox 要求
