# OmniBox-Spider-Skills

OmniBox-Spider-Skills 是一个面向 OmniBox 爬虫开发的技能仓库，提供可复用的技能说明与参考文档，帮助开发者更快完成爬虫源的编写、调试与维护。

## 项目定位

- 提供 OmniBox 爬虫开发相关的标准化指导
- 沉淀常见开发模式（普通采集站、推送源、网盘源）
- 统一接口约定与返回结构，降低接入成本

## 主要内容

- `SKILL.md`：技能定义与执行规则（核心入口）
- `references/`：官方文档的结构化参考与模板
  - `introduction.md`：爬虫开发介绍
  - `getting-started.md`：快速开始
  - `api-reference.md`：接口规范
  - `javascript-sdk.md` / `python-sdk.md`：SDK 能力说明
  - `js-template.md` / `py-template.md`：脚本模板示例

## 目录结构

```text
.
|-- README.md
|-- SKILL.md
|-- LICENSE
`-- references/
    |-- introduction.md
    |-- getting-started.md
    |-- api-reference.md
    |-- script-annotation-attributes.md
    |-- javascript-sdk.md
    |-- python-sdk.md
    |-- js-template.md
    `-- py-template.md
```

## 快速开始

1. 克隆仓库到本地：

```bash
git clone https://github.com/Silent1566/OmniBox-Spider-Skills.git
```

2. 阅读核心技能说明：

- `SKILL.md`

3. 按语言选择参考文档：

- JavaScript：`references/javascript-sdk.md`、`references/js-template.md`
- Python：`references/python-sdk.md`、`references/py-template.md`

4. 根据场景实现爬虫 handler：

- 常规影视源：建议实现 `home` / `category` / `detail` / `search` / `play`
- 推送型脚本：通常优先实现 `detail` + `play`

## 使用建议

- 优先采用 `(params, context)` 作为 handler 签名
- 使用 `context.from` 处理不同客户端差异（web / tvbox / uz / catvod / emby）
- 发生异常时返回安全空结构，并记录关键日志（参数、URL、数量、错误原因）

## 贡献指南

欢迎通过 Issue 或 Pull Request 参与改进：

- 补充或修正文档内容
- 增加更多实战模板
- 改进技能规则与最佳实践

提交前建议：

- 保持文档结构清晰、术语统一
- 示例代码可运行、字段命名与 OmniBox 规范一致

## 友情链接

- 官方文档：https://omnibox-doc.pages.dev/
- 第三方源仓库：https://github.com/Silent1566/OmniBox-Spider

## Star History

[<image-card alt="Star History Chart" src="https://api.star-history.com/svg?/Silent1566/OmniBox-Spider-Skills&type=Date" ></image-card>](https://star-history.com/#Silent1566/OmniBox-Spider-Skills&Date)

<!-- 或者更推荐带暗色适配的写法 -->
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?/Silent1566/OmniBox-Spider-Skills&type=Date&theme=dark" />
  <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?/Silent1566/OmniBox-Spider-Skills&type=Date" />
  <img alt="Star History Chart" src="https://api.star-history.com/svg?/Silent1566/OmniBox-Spider-Skills&type=Date" />
</picture>

## 许可证

本项目基于 MIT License 开源，详见 `LICENSE`。
