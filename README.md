# fda483-report-interpreter

> 可复用的 AI 技能（Skill）：基于真实生产项目（huaxiang / ThinkPHP）的「FDA 483 检查报告」模块逻辑提炼而成。
> 用于**查询企业 FDA 483 检查报告列表**，并**对报告 PDF 进行 AI 结构化中文解读**（下载扫描件 → OCR → 大模型分析 → 缓存）。

[![license](https://img.shields.io/badge/license-Custom-blue.svg)](./LICENSE)
[![skill](https://img.shields.io/badge/type-AI%20Skill-green.svg)](./SKILL.md)

---

## 目录

- [这是什么](#这是什么)
- [核心功能](#核心功能)
- [工作流程](#工作流程)
- [目录结构](#目录结构)
- [安装](#安装)
- [快速上手](#快速上手)
- [配置（环境变量）](#配置环境变量)
- [实现要点](#实现要点)
- [安全说明](#安全说明)
- [FAQ](#faq)
- [许可](#许可)

---

## 这是什么

**FDA Form 483**（Inspectional Observations）是美国 FDA 对受监管企业（药品、生物制品、医疗器械等）进行现场检查时，
检查官记录「观察到的、可能违反《联邦食品、药品和化妆品法案》」的缺陷清单。它是企业 GMP 合规与出口注册的
重要风险信号。

本技能把「拿到一份 483 报告 PDF → 用 AI 产出可操作的中文合规解读」这件事，
从生产代码中抽象为一套**可移植、可复用的流程与提示词模板**，供任何 Agent / 工具链调用。

---

## 核心功能

| 功能 | 说明 |
|------|------|
| **报告列表查询** | 按企业名查询 FDA 483 检查记录；中文企业名做「中文品牌词 + 译英兜底」两层匹配，分页返回 |
| **AI 全面解读** | 下载 483 报告扫描件 PDF → OCR 提取正文 → 调用 DeepSeek 输出结构化中文解读 → 结果入库缓存、可重试 |
| **结构化输出** | 解读含 6 个部分：检查概况、检查发现概览、逐条缺陷分析（含 21 CFR 法规依据 + 风险等级）、风险综合评估、整改建议（短/长期）、OCR 误差标注 |

---

## 工作流程

```
┌─────────────────────────────────────────────────────────────┐
│  功能一：报告列表查询                                          │
│  用户输入企业名                                               │
│       │                                                       │
│       ▼                                                       │
│  1. 同步数据（全量/增量拉取 API → 本地按 remote_id 去重入库）  │
│  2. 两层关键词匹配：                                          │
│       · 中文层：完整名 + 中文品牌核心词（剥后缀）              │
│       · 英文兜底：译英 → 英文品牌核心词                       │
│  3. company_name LIKE '%kw%' OR 组合 → 按 record_date 倒序分页│
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  功能二：AI 解读（单条，耗时 1–3 分钟）                        │
│  interpretFda483(id)                                          │
│       │                                                       │
│       ▼                                                       │
│  缓存命中? ──是──▶ 返回已解读内容（不重复调用 AI）             │
│       │否                                                     │
│       ▼                                                       │
│  下载 PDF 附件（非浏览器 UA）→ runtime/fda483/ 缓存           │
│       ▼                                                       │
│  Python OCR（fitz 渲染 + tesseract eng）→ 缓存 .txt           │
│       ▼                                                       │
│  OCR 文本 < 50 字符? ──是──▶ 抛错（空白扫描件）               │
│       ▼                                                       │
│  DeepSeek 解读（截断 45K 字符）→ 入库（status=1）             │
│  失败 → status=2 + error 字段（可重试）                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 目录结构

```
fda483-report-interpreter/
├── SKILL.md                      # 技能主文件（触发条件 + 核心逻辑，Agent 加载入口）
├── README.md                     # 本文件
└── references/
    ├── brand-keyword.md          # 中英文品牌词提取实现与后缀表
    ├── prompt.md                 # DeepSeek AI 解读完整提示词模板
    ├── config.md                 # 环境变量与配置（已脱敏）
    ├── schema.md                 # ysc_fda_483_inspection 表结构建议
    └── ocr.md                    # Python OCR 脚本要点与依赖
```

---

## 安装

将整个 `fda483-report-interpreter/` 目录放入你的 Agent / 工具链的 `skills/` 目录
（或 `.twork/skills/`），保持目录名不变即可被技能系统识别并加载。

```
your-agent/
└── skills/
    └── fda483-report-interpreter/
        ├── SKILL.md
        ├── README.md
        └── references/
```

---

## 快速上手

### 1. 查询企业 483 列表

```php
$list = $service->getFda483List('恒瑞医药', 1, 20);
// 返回：['total' => N, 'list' => [...], 'has_more' => bool]
```

### 2. AI 解读某条记录

```php
$result = $service->interpretFda483($id);
// 成功：['success' => true,  'interpretation' => '...Markdown...']
// 已解读：['success' => true,  'interpreted' => true, 'interpretation' => '...']
// 失败：  ['success' => false, 'message' => '错误原因']
```

### 3. 解读结果示例（Markdown 节选）

```markdown
## 三、逐条缺陷分析

### OBSERVATION 1
- **缺陷描述**：该公司未对无菌灌装线的培养基模拟灌装（APS）进行充分管理……
- **法规依据**：21 CFR 211.113(b)（无菌工艺控制）
- **风险等级**：高 — 直接影响无菌保证水平，可能涉及已放行产品召回

## 五、整改建议
- **短期**：立即暂停相关生产线、启动偏差调查、评估在库产品风险
- **长期**：重建无菌工艺验证体系、完善培养基模拟灌装方案
```

---

## 配置（环境变量）

> ⚠️ 密钥请一律使用环境变量 / 配置文件读取，**禁止硬编码到代码或提交产物中**。

| 变量 | 用途 |
|------|------|
| `DEEPSEEK.API_KEY` | DeepSeek 翻译 + AI 解读（OpenAI 兼容格式） |
| `FDA483.API_KEY` | 483 数据源 API 鉴权（`X-Api-Key`） |
| `FDA483.PYTHON_BIN` | OCR 脚本 Python 解释器路径 |
| `BAIDU_TRANSLATE.APPID` | 百度翻译备选（可选，译英兜底链路） |

详见 [`references/config.md`](./references/config.md)。

---

## 实现要点

- **中英文品牌词提取**：中文剥「药业/制药/医药/生物科技/有限公司」等后缀；英文剥
  `Pharmaceutical / Co., Ltd. / Inc. / Limited` 等，最多 3 轮嵌套剥离。详见
  [`references/brand-keyword.md`](./references/brand-keyword.md)。
- **翻译 Provider 链路**：DeepSeek 优先 → 百度 → Google 免费翻译，统一接口 `translate(text, from, to)`。
- **OCR 依赖**：`PyMuPDF`（fitz）+ `tesseract`（eng），扫描件无文本层必须 OCR。详见
  [`references/ocr.md`](./references/ocr.md)。
- **提示词模板**：系统提示词「资深 GMP 与 FDA 合规专家」+ 结构化用户提示词，要求 6 段式中文解读。
  详见 [`references/prompt.md`](./references/prompt.md)。
- **数据表结构**：`ysc_fda_483_inspection`（含 `interpretation_status` 缓存状态）。详见
  [`references/schema.md`](./references/schema.md)。

---

## 安全说明

- 本技能所有密钥均**已脱敏**（占位符 + 示例域名）。
- **请勿**将包含真实 API Key 的源码与本技能一同发布到公开仓库。
- 发布前请确认 `.env`、配置文件中不含任何真实凭据。

---

## FAQ

**Q：为什么中文企业名查不到 FDA 英文记录？**
A：FDA 数据源企业名多为英文，需走「译英兜底」第二层匹配。确保翻译 Provider 已配置，
且 `extractEnglishBrandKeyword()` 后缀表覆盖该企业命名习惯。

**Q：OCR 提取内容太少 / 空白？**
A：扫描件质量差或 tesseract 英文包缺失。脚本对 < 50 字符判定为失败（退出码 2），可重试或换高 DPI。

**Q：解读很慢怎么办？**
A：下载 + OCR + AI 全流程约 1–3 分钟，需将 PHP 执行时间放宽到 `set_time_limit(600)`，
并对已解读记录做缓存（命中即返回，不重复调用 AI）。

**Q：能用于 NMPA / EMA 现场检查报告吗？**
A：本技能提炼自 FDA 483 流程，迁移到其它监管机构报告时，主要替换数据源 API 与
解读提示词中的法规依据（如把 21 CFR 换成 GMP 附录），其余 OCR + 缓存链路可复用。

---

## 许可

按所属项目许可发布。转载请注明来源。
