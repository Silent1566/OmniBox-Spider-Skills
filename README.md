# OmniBox-Spider-Skills

OmniBox-Spider-Skills 是一个面向 OmniBox 爬虫开发的技能仓库，提供可复用的技能说明、规则约束与结构化参考文档，帮助开发者更快完成爬虫源的编写、调试、回归检查与持续维护。

## 项目定位

- 提供 OmniBox 爬虫开发相关的标准化指导
- 沉淀常见开发模式（普通采集站、推送源、网盘源）
- 统一接口约定、返回结构与交付规范，降低接入成本
- 补充版本管理、环境变量复用、校验与经验沉淀，减少重复踩坑

## 主要内容

- `SKILL.md`：技能定义与执行规则（核心入口）
- `references/`：官方文档的结构化参考、模板与补充约束
  - `introduction.md`：爬虫开发介绍
  - `getting-started.md`：快速开始
  - `api-reference.md`：接口规范
  - `script-annotation-attributes.md`：脚本注释属性说明
  - `javascript-sdk.md` / `python-sdk.md`：SDK 能力说明
  - `sdk-api.md`：SDK API 汇总参考
  - `js-template.md` / `py-template.md`：脚本模板示例
  - `environment-variables.md` / `environment-variables-cheatsheet.md`：环境变量约定与速查
  - `versioning.md`：脚本版本升级规则
  - `validation.md`：修改后的静态检查与回归检查要求
  - `lessons-learned.md` / `omnibox-spider-lessons.md`：实战经验与问题复盘
  - `ds-template-notes.md`：模板与开发补充说明

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
|-- sdk-api.md
|-- js-template.md
|-- py-template.md
|-- environment-variables.md
|-- environment-variables-cheatsheet.md
|-- versioning.md
|-- validation.md
|-- lessons-learned.md
|-- omnibox-spider-lessons.md
    `-- ds-template-notes.md
```

## 快速开始

1. 克隆仓库到本地：

```bash
git clone https://github.com/Silent1566/OmniBox-Spider-Skills.git
```

2. 阅读核心技能说明与最新规则：

- `SKILL.md`

3. 按语言选择参考文档：

- JavaScript：`references/javascript-sdk.md`、`references/js-template.md`、`references/sdk-api.md`
- Python：`references/python-sdk.md`、`references/py-template.md`、`references/sdk-api.md`

4. 根据场景实现爬虫 handler：

- 常规影视源：建议实现 `home` / `category` / `detail` / `search` / `play`
- 推送型脚本：通常优先实现 `detail` + `play`
- 网盘源：优先结合相关 SDK 能力与刮削链路实现

## 当前规则重点

- handler 签名默认统一为 `(params, context)`
- 端差异逻辑优先通过 `context.from` 区分（`web` / `tvbox` / `uz` / `catvod` / `emby`）
- `play` 优先返回 `{ urls, flag, header, parse, danmaku }` 结构
- 修改脚本后必须同步升级 `@version`，并按语义化版本说明升级原因
- 修改脚本后至少完成一次静态检查：JavaScript 使用 `node --check`，Python 使用 `python3 -m py_compile`
- 交付前需做关键路径回归检查，至少确认 `detail()`、`vod_play_sources`、`play()` 返回结构正常

## 使用建议

- 优先采用 `(params, context)` 作为 handler 签名
- 使用 `context.from` 处理不同客户端差异（web / tvbox / uz / catvod / emby）
- 发生异常时返回安全空结构，并记录关键日志（参数、URL、数量、错误原因）
- 新增环境变量前先复用仓库既有命名，例如 `SITE_API`、`DANMU_API`、`DRIVE_ORDER`、`SOURCE_NAMES_CONFIG`
- 修改逻辑时默认采用最小改动策略，避免顺手重构无关代码

## 推荐阅读路径

- 新建脚本：`references/getting-started.md` + `references/js-template.md` 或 `references/py-template.md`
- 查返回结构：`references/api-reference.md`
- 查脚本头注释：`references/script-annotation-attributes.md`
- 查环境变量：`references/environment-variables.md`、`references/environment-variables-cheatsheet.md`
- 判断版本号升级：`references/versioning.md`
- 做修改后校验：`references/validation.md`
- 复盘实战经验：`references/lessons-learned.md`、`references/omnibox-spider-lessons.md`

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

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?/Silent1566/OmniBox-Spider-Skills&type=Date&theme=dark" />
  <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?/Silent1566/OmniBox-Spider-Skills&type=Date" />
  <img alt="Star History Chart" src="https://api.star-history.com/svg?/Silent1566/OmniBox-Spider-Skills&type=Date" />
</picture>

## 许可证

本项目基于 MIT License 开源，详见 `LICENSE`。
