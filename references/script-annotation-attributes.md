# 脚本注释属性 | OmniBox

脚本前 50 行内的注释属性用于声明爬虫源元信息。只有部分属性会被后端解析并实际使用。

- JavaScript：行首使用 `//`
- Python：行首使用 `#`

---

## 1. `@version`

### 作用
声明脚本版本号，用于后台“检查更新”和“从远端更新”时进行版本比较。

### 入参格式
- 字符串版本号，建议语义化版本，如：`1.0.0`

### 出参 / 结果
- 无直接返回值
- 后台会读取并与远端脚本版本比较

### 示例
```javascript
// @version: 1.0.0
// @version 1.0.0
```
```python
# @version: 1.0.0
# @version 1.0.0
```

---

## 2. `@downloadURL`

### 作用
声明远程脚本地址，用于后台从远端拉取脚本并检测版本更新。

### 入参格式
- 一个远程 URL

### 出参 / 结果
- 无直接返回值
- 后台会访问该 URL，读取远端脚本内容和其中的 `@version`

### 示例
```javascript
// @downloadURL: https://example.com/script.js
// @downloadURL https://example.com/script.js
```
```python
# @downloadURL: https://example.com/script.py
# @downloadURL https://example.com/script.py
```

### 注意
- 没有 `@downloadURL` 时，后台无法执行远端更新
- 通常与 `@version` 配合使用

---

## 3. `@indexs`

### 作用
声明该爬虫源是否参与聚合搜索。

### 入参格式
- `1`：参与
- `0`：不参与

### 出参 / 结果
- 无直接返回值
- 后台根据该值决定是否把此源纳入站内聚合搜索

### 示例
```javascript
// @indexs 1
// @indexs 0
```
```python
# @indexs 1
# @indexs 0
```

### 默认行为
- 未声明时，默认不参与聚合搜索

---

## 4. `@push`

### 作用
声明该爬虫源是否为推送型爬虫。

### 入参格式
- `1`：推送型爬虫
- `0`：常规爬虫

### 出参 / 结果
- 无直接返回值
- 后台会据此做展示与筛选上的区分

### 示例
```javascript
// @push 1
// @push 0
```
```python
# @push 1
# @push 0
```

### 说明
- 推送型爬虫通常只需要实现 `detail` 和 `play`
- 未声明时默认按常规爬虫处理

---

## 5. `@dependencies`

### 作用
声明脚本需要的外部依赖包（npm 或 pip）。保存脚本时若缺失，系统会尝试自动安装。

### 入参格式
- 单行
- 逗号分隔包名

### 出参 / 结果
- 无直接返回值
- 后台保存脚本时尝试安装依赖

### 示例
```javascript
// @dependencies: axios, cheerio
// @dependencies axios, cheerio
```
```python
# @dependencies: requests, beautifulsoup4
# @dependencies requests, beautifulsoup4
```

### 注意
- 只在脚本前 50 行内生效
- 包名之间建议用逗号分隔，保持简洁清晰

---

## 通用建议

### 放置位置
所有注释属性尽量放在文件最前面，避免超出前 50 行不被识别。

### 常规影视源推荐
至少写：
- `@version`
- 必要时 `@dependencies`

### 可远程分发脚本推荐
加上：
- `@downloadURL`
- `@version`

### 参与统一搜索的源推荐
加上：
- `@indexs 1`

### 推送源推荐
加上：
- `@push 1`
