# WordPress / Zibll 分类筛选与排序补充笔记

适用场景：站点是 WordPress / Zibll 主题，前台顶部导航把影视分类做成 taxonomy category 菜单，用户希望 OmniBox `category()` 支持“各分类筛选 + 排序”。

## 这次会话落下的有效经验

### 1. 子分类 slug 优先从顶部菜单真实链接提取
不要先猜筛选值。像这类站点，前台顶部菜单里经常已经给出真实 taxonomy 路由：

- 电影：`/category/film/domestic`、`/category/film/euramerican`、`/category/film/japankorean`、`/category/film/other`
- 电视剧：`/category/show/domestic-show`、`/category/show/euramerican-show`、`/category/show/japankorean-show`、`/category/show/other-show`
- 动漫：`/category/animation/domestic-animation`、`/category/animation/euramerican-animation`、`/category/animation/japankorean-animation`、`/category/animation/other-animation`
- 综艺：`/category/reality/domestic-reality`、`/category/reality/other-reality`
- 纪录片：`/category/documentary/domestic-documentary`、`/category/documentary/other-documentary`
- 音乐：只有主分类 `music`，未必有二级筛选

做法建议：
1. 先抓首页 HTML；
2. 解析 `.nav .menu-item-type-taxonomy` / `.sub-menu a[href*="/category/"]`；
3. 用真实 href 反推出 `categoryId + subclass slug`；
4. `FILTERS` 默认优先和前台菜单保持一致。

### 2. 分类页就算没显式筛选 UI，也可能支持 `orderby`
这类站点首页常见排序链接：

- `?orderby=modified`
- `?orderby=views`
- `?orderby=like`
- `?orderby=comment_count`

即使分类页 HTML 里没再单独渲染“排序”按钮，也值得直接实测：

- `/category/film?orderby=views`
- `/category/film?orderby=like`
- `/category/film?orderby=comment_count`

如果前几条结果顺序确实变化，就可以把这些值接到 OmniBox `filters` 里。

### 3. 更稳妥的 `category()` 设计
推荐拆成三步：

1. `params.filters || params.extend || params.ext` 统一取值；
2. `subclass` / `orderby` 都做白名单校验；
3. URL 按真实 taxonomy 路由拼接：
   - 无子分类：`/category/{main}`
   - 有子分类：`/category/{main}/{sub}`
   - 翻页：`/page/{n}`
   - 排序：追加 `?orderby=...`

例子：

- `/category/show/domestic-show?orderby=views`
- `/category/animation/japankorean-animation/page/2?orderby=like`

### 4. 日志建议
至少打印：

- `category`
- `subclass`
- `orderby`
- `page`
- `path`
- `final url`
- `list count`
- `pagecount`

这样用户贴一轮日志，就能立刻判断：
- 宿主有没有把筛选值传进来
- 脚本有没有命中真实 taxonomy slug
- 排序参数有没有真正进入请求 URL

## 适合沉淀成代码辅助函数的点

- `normalizeFilterPayload(raw)`：兼容 `filters / extend / ext`
- `resolveCategoryState(categoryId, params)`：白名单归一化 `subclass + orderby`
- `buildCategoryUrl(categoryPath, page, orderby)`：统一拼 taxonomy 路径 + 分页 + 排序
- `parseCategoryPageCount(html, categoryPath, page)`：分页正则要按完整路径匹配，不只是主分类 slug

## 本次实测确认

光鸭·臻影社这类站点已实测支持以下排序值：

- `modified`
- `views`
- `like`
- `comment_count`

验证方法：直接请求分类页并对比前几条文章链接顺序是否变化。