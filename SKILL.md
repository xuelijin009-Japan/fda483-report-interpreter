---
name: fda483-report-interpreter
description: 基于 huaxiang 项目的 FDA 483 检查报告查询与 AI 解读逻辑提炼而成的可复用技能。当用户需要：查询某企业的 FDA 483 检查报告列表、对 483 报告进行 AI 结构化中文解读（下载扫描件 PDF → OCR → 大模型分析 → 缓存）、构建合规风险分析功能，或涉及 FDA FOIA Inspection、GMP 合规、483 观察项（Observation）分析时，加载本技能。覆盖企业名中英文匹配、API 同步、PDF OCR、DeepSeek 解读提示词等完整链路。
---

# FDA 483 报告解读技能

本技能提炼自真实生产项目（huaxiang / ThinkPHP）中「FDA 483 检查报告」模块的核心逻辑，
用于**查询企业 FDA 483 检查报告列表**并**对报告进行 AI 结构化中文解读**。

> ⚠️ 安全红线：本技能涉及外部数据 API 与 AI API 的密钥。**禁止**在代码、文档、提交产物中
> 硬编码真实密钥。所有密钥必须从环境变量 / 配置文件读取（见 `references/config.md`）。

---

## 何时使用

- 用户要「查询某企业的 FDA 483 报告」「查 483 检查记录」「FDA 合规查询」。
- 用户要「解读 483 报告」「把 483 翻译成中文解读」「483 风险分析」「GMP 缺陷分析」。
- 用户要构建类似的「监管检查报告 AI 解读」能力（FDA 483 / EMA / NMPA 现场检查等）。
- 用户提到 483、Observation、FEI Number、FOIA Inspection、Form 483 等术语。

---

## 核心能力（两大功能）

### 功能一：报告列表查询（按企业名匹配）

入口：`getFda483List(企业名, page, limit)`

1. **数据同步**：首次查询该企业的记录数为 0 时，全量同步（最多 5 页）；否则增量同步（1 页）。
   数据源 API 不支持按企业名过滤，需**全量拉取后本地筛选**，按 `remote_id` 去重入库。
   - 本地表：`ysc_fda_483_inspection`
   - 入库字段：remote_id, fei_number, record_type, record_date, company_name,
     establishment_type, province_or_state, country, attachment, attachment_url, query_enterprise
2. **两层关键词匹配**（FDA 数据源企业名多为英文，而用户输入多为中文）：
   - 第一层（中文）：用 `buildChineseKeywords()` —— 完整名称 + 提取的中文品牌核心词
     （剥离「药业/制药/医药/生物科技/有限公司」等后缀，见 `references/brand-keyword.md`）。
   - 第二层（英文兜底）：第一层无结果时，调用翻译 Provider 把中文名/品牌词译英，
     并抽取英文品牌核心词（剥离 Pharmaceutical / Co., Ltd. / Inc. 等后缀），再匹配。
   - 关键词用 `company_name LIKE '%kw%'` 多关键词 OR 组合，按 `record_date` 倒序分页。

### 功能二：AI 全面解读（下载 → OCR → 解读 → 缓存）

入口：`interpretFda483(记录 id)`

流程（单条记录可能耗时 1–3 分钟，需 `set_time_limit(600)`）：

```
1. 缓存命中：interpretation_status=1 且 interpretation 非空 → 直接返回缓存，不重复调用 AI
2. 下载附件 PDF → runtime/fda483/ 本地缓存（非浏览器 UA 才能正常下载 FDA 官网，见注意事项）
3. OCR 提取文本（Python 脚本，扫描件无文本层）→ 缓存 .txt
4. OCR 文本 < 50 字符 → 抛错（空白扫描件）
5. 调用 DeepSeek 解读（截断到 45K 字符，控制 token）
6. 结果入库（interpretation / interpretation_time / status=1）；失败则 status=2 + error
```

AI 解读提示词（`deepSeekInterpretFda483`）要求输出**结构化中文 Markdown**，包含 6 个部分：
检查概况、检查发现概览、逐条缺陷分析（含法规依据 21 CFR 210/211、风险等级）、
风险综合评估、整改建议（短/长期）、OCR 误差标注。完整提示词模板见 `references/prompt.md`。

---

## 关键实现要点

### 企业名品牌词提取（中/英）

- 中文：`extractBrandKeyword()` 用后缀替换表（药业/制药/医药/生物科技/有限公司…）逐条剥离，
  剩余 < 2 字则取原名前 4 字兜底。
- 英文：`extractEnglishBrandKeyword()` 剥离 Pharmaceutical / Biopharmaceutical / Bio-Tech /
  Co., Ltd. / Corporation / Inc. / Limited 等后缀，最多 3 轮嵌套剥离 + 修剪标点。

### 翻译 Provider 链路（优先级降级）

`getFdaTranslator()`：DeepSeek 优先 → 百度翻译 → Google 免费翻译。
实现见 `app/service/` 下 `DeepSeekTranslateProvider`、`BaiduTranslateProvider`、`GoogleFreeTranslator`，
统一实现 `TranslationProvider` 接口的 `translate(text, from, to)`。

### OCR 依赖

`data/fda483/fda483_ocr.py`：PyMuPDF(fitz) 渲染页面为 PNG → tesseract(eng, psm 3) 逐页 OCR。
环境变量 `FDA483.PYTHON_BIN` 指定 Python 解释器路径（默认 Windows 便携路径），
`--max-pages 40` 限制页数。提取文本 < 50 字符视为失败（退出码 2）。

### 缓存与失败处理

- PDF 与 OCR 文本文件按路径缓存，非空则复用，避免重复下载/识别。
- 解读结果按记录入库并标记 status；失败记录 status=2 + error 字段（限长 480 字），允许前端重试。

---

## 注意事项（踩坑点）

1. **密钥绝不硬编码**：原项目 `syncFda483FromApi()` 曾硬编码 API Key —— 已在本技能脱敏，
   实际部署从 `.env` / 配置读取（`FDA483.API_KEY` 风格）。
2. **FDA 官网下载 UA**：必须用**非浏览器 UA**（如 `FDA483-Fetcher/1.0 (crhms data-service)`），
   伪装浏览器 UA 反而会被重定向到滥用检测页（404）。
3. **OCR 文本质量控制**：扫描件 OCR 误差大，提示词需明确要求「结合上下文合理推断并标注（OCR文本可能有误差）」。
4. **token 截断**：DeepSeek 上下文 64K，OCR 文本截断到 ~45K 字符保留余量。
5. **耗时操作超时**：下载+OCR+AI 全流程需放宽 PHP 执行时间到 600 秒。
6. **中文企业名查不到英文源**：务必保留「译英兜底」第二层，否则大量中国企业查不到 FDA 英文记录。

---

## 参考文件

- `references/brand-keyword.md` —— 中英文品牌词提取实现与后缀表
- `references/prompt.md` —— DeepSeek AI 解读完整提示词模板
- `references/config.md` —— 环境变量与配置（脱敏），含 API 同步入参结构
- `references/schema.md` —— `ysc_fda_483_inspection` 表结构建议
- `references/ocr.md` —— Python OCR 脚本要点与依赖
