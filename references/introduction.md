# 爬虫开发介绍 | OmniBox

本文档介绍 OmniBox 爬虫源的基本概念、能力边界、运行方式和推荐开发模型。

---

## 1. 什么是爬虫源

### 作用
爬虫源是 OmniBox 的核心扩展机制，用来把第三方网站、API、网盘、推送链接等数据接入 OmniBox。

### 能力
- 自定义视频源：从任意网站或 API 获取数据
- 灵活扩展：支持 JavaScript / Python
- 统一接口：所有源都遵循统一 handler 规范
- 可视化编辑：可在后台编辑与调试
- 实时调试：可查看日志与执行结果

### 适用场景
- 普通采集站对接
- 网盘资源站对接
- 推送型脚本
- 第三方影视 API 转换
- TVBox / UZ / Emby 等兼容转换

---

## 2. 支持的脚本语言

### JavaScript
#### 作用
适合 Node.js 开发者快速实现站点采集逻辑。

#### 特点
- Node.js 运行时
- 支持 Node 标准库与 npm 包
- 使用 `require()`
- 支持 ES6+

### Python
#### 作用
适合 Python 开发者，尤其是需要更强文本处理、签名、加密、脚本生态时。

#### 特点
- Python 3 运行时
- 支持标准库与第三方库（pip）
- 支持异步编程
- 自动处理中文编码

---

## 3. 五个核心 handler

OmniBox 爬虫源推荐统一实现以下方法：

| 方法 | 作用 | 常见入参 | 常见出参 |
|---|---|---|---|
| `home` | 首页分类与推荐 | `{}` | `{ class, list, banner?, filters? }` |
| `category` | 分类分页列表 | `{ categoryId, page, filters? }` | `{ page, pagecount, total, list }` |
| `detail` | 视频详情 | `{ videoId }` | `{ list }` |
| `search` | 搜索结果 | `{ keyword, page?, quick? }` | `{ page, pagecount, total, list }` |
| `play` | 播放信息 | `{ playId, flag? }` | `{ urls, flag, header?, parse?, danmaku? }` |

### 说明
- 不是所有脚本都必须实现全部 5 个方法
- **推送型脚本**通常只需要 `detail + play`
- **常规影视源**建议实现全部 5 个

---

## 4. 请求上下文 context

### 作用
Runner 在每次调用 handler 时注入上下文，用于获取调用端、请求头、当前源 ID 等环境信息。

### 推荐签名
```javascript
async function home(params, context) {}
```
```python
async def home(params, context):
    pass
```

### 字段说明

| 字段 | 类型 | 作用 |
|---|---|---|
| `baseURL` | string | 当前请求基础 URL，可用于拼绝对链接 |
| `headers` | object | 调用端透传请求头，如 UA / Cookie |
| `sourceId` | string | 当前爬虫源 ID |
| `from` | string | 调用端：`web / tvbox / uz / catvod / emby` |

### 示例
```javascript
const from = context?.from || "web";
if (from === "tvbox") {
  // 只返回 TV 端可识别字段
}
```

### 重要规则
- 必须通过第二个参数 `context` 读取
- 不要从全局变量猜测调用端

---

## 5. SDK 注入方式

### 作用
让脚本能直接调用 OmniBox SDK，无需自己指定路径。

### 注入机制
- JavaScript：通过 `NODE_PATH`
- Python：通过 `PYTHONPATH`

### 示例
```javascript
const OmniBox = require("omnibox_sdk");
```
```python
from spider_runner import OmniBox, run
```

---

## 6. 脚本注释属性

### 作用
在脚本头声明版本、远程地址、依赖、是否参与聚合搜索、是否推送源等元信息。

### 常用项
- `@version`
- `@downloadURL`
- `@indexs`
- `@push`
- `@dependencies`

### 说明
- 仅脚本前 50 行内生效
- 详细规则见 `script-annotation-attributes.md`

---

## 7. SDK 能力概览

### JavaScript / Python 通用方向
- HTTP 请求
- 日志
- 环境变量
- 爬虫源标签与分类数据
- 网盘能力
- 刮削能力
- 视频嗅探
- 弹幕匹配
- 解析站配置
- 观看记录

### 额外说明
- Python 还提供加密相关 API（AES / MD5 / Base64）
- 视频嗅探需要完整版运行环境（Playwright / Chromium）

---

## 8. 推荐开发流程

### 作用
降低第一次写 OmniBox 源的复杂度。

### 步骤
1. 在后台创建爬虫源
2. 选择 JS 或 Python
3. 实现最小 `home` / `detail` / `play`
4. 用后台调试器测试
5. 补全 `category` / `search`
6. 加日志、兜底返回、端兼容逻辑

---

## 9. 常见开发模式

### 普通采集站
- 封装请求函数
- 映射返回字段到 OmniBox 标准字段
- 在 `detail` 中构建 `vod_play_sources`
- 在 `play` 中返回可播信息

### 推送源
- 通常只实现 `detail` 与 `play`
- 适合处理外部链接、资源链接、网盘链接

### 网盘源
- 常用：
  - `getDriveFileList`
  - `getDriveVideoPlayInfo`
  - `getDriveInfoByShareURL`
  - `processScraping`
  - `getScrapeMetadata`
  - `getDanmakuByFileName`

---

## 10. 关键实践建议

### 返回结构要稳定
即使失败，也尽量返回空结构，而不是直接抛裸异常。

### 日志要完整
至少记录：
- 入口参数
- 请求 URL
- 结果数量
- 失败原因

### 优先读取 context.from
所有客户端差异逻辑统一由 `context.from` 控制。

### 环境变量优先直接读取
- JS：`process.env.KEY`
- Python：`os.environ.get("KEY")`

### parse 的兼容性要心里有数
- `parse = 1` 仅 ok影视 app 真正支持
- 其他客户端通常忽略
