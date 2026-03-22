# 快速开始 | OmniBox

本指南用于创建第一个 OmniBox 爬虫源。

---

## 1. 创建爬虫源

### 作用
在 OmniBox 后台创建一个新的自定义视频源脚本。

### 步骤
1. 打开 OmniBox 管理后台
2. 进入“爬虫源管理”页面
3. 点击“新建爬虫源”
4. 填写基础信息：
   - 名称
   - 类型（JavaScript / Python）
   - 描述（可选）

---

## 2. 语言选择

### JavaScript
#### 作用
适合熟悉 Node.js 的开发者。

#### 特点
- Node.js 运行时
- 支持 npm 包
- 使用 `require()`
- 支持 ES6+

### Python
#### 作用
适合熟悉 Python 的开发者。

#### 特点
- Python 3 运行时
- 支持 pip 包
- 推荐配合 `spider_runner.run()`

---

## 3. JavaScript 最小示例

### 作用
演示一个最基础可运行的 JS 爬虫脚本。

### 示例

```javascript
// @name 示例爬虫源
// @version 1.0.0

const OmniBox = require("omnibox_sdk");
const runner = require("spider_runner");

module.exports = { home, category, detail, search, play };
runner.run(module.exports);

async function home(params, context) {
  await OmniBox.log("info", "获取首页数据");
  return {
    class: [
      { type_id: "1", type_name: "电影" },
      { type_id: "2", type_name: "电视剧" },
    ],
    list: [
      {
        vod_id: "1",
        vod_name: "示例视频",
        vod_pic: "https://example.com/pic.jpg",
        type_id: "1",
        type_name: "电影",
      },
    ],
  };
}
```

### 说明
- `home(params, context)`：获取首页数据
- `class`：分类列表
- `list`：推荐视频列表
- `runner.run(module.exports)`：注册并启动所有 handler

---

## 4. Python 最小示例

### 作用
演示一个最基础可运行的 Python 爬虫脚本。

### 示例

```python
# -*- coding: utf-8 -*-
# @name 示例爬虫源
# @version 1.0.0

from spider_runner import OmniBox, run

async def home(params, context):
    await OmniBox.log("info", "获取首页数据")
    return {
        "class": [{"type_id": "1", "type_name": "电影"}],
        "list": [{
            "vod_id": "1",
            "vod_name": "示例视频",
            "vod_pic": "https://example.com/pic.jpg",
            "type_id": "1",
            "type_name": "电影"
        }],
    }

if __name__ == "__main__":
    run({"home": home})
```

### 说明
- `run({"home": home})`：注册 handler
- Python 推荐统一使用 `run()`，不要自己写 stdin 解析

---

## 5. 五个核心 handler

### `home(params, context)`
#### 作用
返回首页分类与推荐列表。

#### 入参
- `params`：通常为空对象
- `context`：请求上下文

#### 出参
```json
{
  "class": [],
  "list": [],
  "banner": [],
  "filters": {}
}
```

### `category(params, context)`
#### 作用
返回指定分类的分页列表。

#### 入参
- `params.categoryId`
- `params.page`
- `params.filters`（可选）

#### 出参
```json
{
  "page": 1,
  "pagecount": 10,
  "total": 100,
  "list": []
}
```

### `detail(params, context)`
#### 作用
返回视频详情与播放源。

#### 入参
- `params.videoId`

#### 出参
```json
{
  "list": [
    {
      "vod_id": "...",
      "vod_name": "...",
      "vod_play_sources": []
    }
  ]
}
```

### `search(params, context)`
#### 作用
根据关键词搜索。

#### 入参
- `params.keyword`
- `params.page`（可选）

#### 出参
```json
{
  "page": 1,
  "pagecount": 1,
  "total": 1,
  "list": []
}
```

### `play(params, context)`
#### 作用
返回播放地址、请求头、弹幕等。

#### 入参
- `params.playId`
- `params.flag`（可选）

#### 出参
推荐格式：
```json
{
  "urls": [{"name": "播放", "url": "https://..."}],
  "flag": "play",
  "header": {},
  "parse": 0,
  "danmaku": []
}
```

---

## 6. 推荐开发顺序

### 作用
降低一次性实现全部功能的复杂度。

### 顺序建议
1. 先写 `home`
2. 再写 `category`
3. 再写 `detail`
4. 再写 `play`
5. 最后补 `search`

---

## 7. 推送型脚本

### 作用
适配直接推送链接、网盘链接、资源链接类场景。

### 最低实现要求
通常只需要：
- `detail`
- `play`

### 说明
这类脚本不一定需要首页、分类、搜索能力。
