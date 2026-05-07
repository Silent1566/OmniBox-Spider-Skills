## 用户反馈：
- 当前代码 `歪比巴卜.js` 仍使用 Express 风格的 `module.exports = async (app, opt) => { app.get(...); opt.sites.push(meta); }`，这不是 OmniBox 官方要求的 OM Spider 结构。
- 用户要求 **OM（OmniBox）爬虫源**，即必须使用 `OmniBox SDK + spider_runner` 的标准模板：
```js
const OmniBox = require("omnibox_sdk");
const runner = require("spider_runner");

class XxxSpider extends OmniBox.Spider {
  async home() { /* ... */ }
  async category({ id, page }) { /* ... */ }
  async detail({ id }) { /* ... */ }
  async search({ page, wd }) { /* ... */ }
  async play({ id }) { /* ... */ }
}

runner.run(new XxxSpider());
```
- 该文件必须实现 `search/getCards/getTracks/getPlayinfo`（或对应的五方法）并返回 OmniBox 标准返回结构。
- 请在新建或重写脚本时删除旧的 `module.exports` 形式，改为上述类继承方式，并确保所有必要的头部注释 (`@name`, `@version`, `@downloadURL` 等) 正确，且 `@version` 已提升。

### 操作建议
1. 删除 `歪比巴卜不夜版.js`（该文件是错误的 Express 版实现）。
2. 将 `歪比巴卜.js` 重写为符合 OM Spider 模板的代码，使用用户提供的站点配置（`key`, `name`, `api`, `host` 等）。
3. 完成后运行 `node --check "影视/采集/歪比巴卜.js"` 验证语法。
4. 提交并推送到新分支，创建 PR，PR 标题和说明使用中文。
5. 在 PR 描述中注明已完成 **OM Spider 格式迁移**，并列出已检查的五个必实现方法。
