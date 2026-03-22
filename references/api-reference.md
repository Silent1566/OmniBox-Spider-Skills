# API 参考 | OmniBox

## 方法总览

所有 handler 推荐签名统一为：`(params, context)`

| 方法 | 作用 | params | 返回 |
|---|---|---|---|
| `home` | 首页分类与推荐 | `{}` | `{ class, list, banner?, filters? }` |
| `category` | 分类分页列表 | `{ categoryId, page, filters? }` | `{ page, pagecount, total, list }` |
| `detail` | 视频详情 | `{ videoId }` | `{ list }` |
| `search` | 搜索结果 | `{ keyword, page?, quick? }` | `{ page, pagecount, total, list }` |
| `play` | 播放信息 | `{ playId, flag? }` | `{ urls, flag, header?, parse?, danmaku? }` |

## context

```ts
interface RequestContext {
  baseURL?: string;
  headers?: Record<string, string>;
  sourceId?: string;
  from?: string; // web | tvbox | uz | catvod | emby
}
```

## `home`
返回：首页分类和推荐列表。

### 主要字段
- `class`: `{ type_id, type_name }[]`
- `list`: `VodItem[]`
- `filters?`
- `banner?`

## `category`
### params
- `categoryId` string
- `page` number
- `filters` object

### 返回
- `page`
- `pagecount`
- `total`
- `list`

## `detail`
### params
- `videoId` string

### 返回关键字段
- `vod_content`
- `vod_actor`
- `vod_director`
- `vod_area`
- `vod_year`
- `vod_remarks`
- `vod_douban_score`
- `type_name`
- `vod_play_sources`

### `vod_play_sources`
```ts
interface PlaySource {
  name: string;
  episodes: Episode[];
}

interface Episode {
  name: string;
  playId: string;
  size?: number;
  episodeName?: string;
  episodeOverview?: string;
  episodeAirDate?: string;
  episodeStillPath?: string;
  episodeVoteAverage?: number;
  episodeRuntime?: number;
}
```

### 刮削元数据
可通过 `OmniBox.getScrapeMetadata(resourceId)` 获取 `videoMappings`，把 TMDB 信息映射到 episode。

## `search`
### params
- `keyword`
- `page?`
- `quick?`

### 返回
- `page`
- `pagecount`
- `total`
- `list`

提示：UZ 客户端支持 `search: 1`。

## `play`
### 推荐返回格式
```ts
{
  urls: Array<{ name: string; url: string }>;
  flag: string;
  header?: Record<string, string>;
  danmaku?: Array<{ name: string; url: string }>;
  parse?: number;
}
```

### `parse`
- `0`：直链（m3u8/mp4）
- `1`：需客户端嗅探

注意：`parse=1` 仅 **ok影视 app** 真正支持，其他端通常忽略。

### 兼容格式
也兼容：
- `url: string`
- `url: string[]`
- `url: { values, position }`

## 数据模型

### `VodItem`
```ts
{
  vod_id: string;
  vod_name: string;
  vod_pic: string;
  type_id: string;
  type_name: string;
  vod_remarks?: string;
  vod_year?: string;
  vod_douban_score?: string;
  vod_subtitle?: string;
  vod_tag?: string;
  search?: number;
}
```

#### 特殊字段
- `vod_tag: "folder"`：目录项，点击进入子目录
- `search: 1`：UZ 专用，点击触发搜索

### `BannerItem`
```ts
{
  title: string;
  subtitle?: string;
  backgroundImage: string;
  genre?: string;
  actors?: string;
  description?: string;
}
```

## 经验规则

- 常规影视源建议实现全部 5 个方法
- 推送型源通常只实现 `detail` 和 `play`
- 所有端差异逻辑统一看 `context.from`
