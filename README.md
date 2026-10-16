# FDA 483 报告解读技能 · fda483-report-interpreter

一个可复用的 AI 技能（Skill），提炼自真实生产项目（huaxiang / ThinkPHP）的「FDA 483 检查报告」模块，
用于**查询企业 FDA 483 检查报告列表**，并**对报告 PDF 进行 AI 结构化中文解读**（下载扫描件 → OCR → 大模型分析 → 缓存）。

## 功能

- **报告列表查询**：按企业名查询 FDA 483 检查记录（两层匹配：中文品牌词 + 译英兜底），分页返回。
- **AI 全面解读**：下载 483 报告扫描件 PDF，OCR 提取正文，调用 DeepSeek 输出结构化中文解读
  （检查概况 / 逐条缺陷分析 / 法规依据 21 CFR / 风险等级 / 整改建议），结果入库缓存、可重试。

## 目录结构

```
fda483-report-interpreter/
├── SKILL.md                      # 技能主文件（触发条件 + 核心逻辑）
├── README.md                     # 本文件
└── references/
    ├── brand-keyword.md          # 中英文品牌词提取实现与后缀表
    ├── prompt.md                 # DeepSeek AI 解读完整提示词模板
    ├── config.md                 # 环境变量与配置（脱敏）
    ├── schema.md                 # ysc_fda_483_inspection 表结构建议
    └── ocr.md                    # Python OCR 脚本要点与依赖
```

## 安装

将本目录整体放入你的 Agent / 工具链 的 `skills/` 目录（或 `.twork/skills/`）
下，保持目录名 `fda483-report-interpreter` 即可被技能系统识别加载。

## 环境变量（密钥请务必使用环境变量，禁止硬编码）

| 变量 | 用途 |
|------|------|
| `DEEPSEEK.API_KEY` | DeepSeek 翻译 + 解读 |
| `FDA483.API_KEY` | 483 数据源 API 鉴权（X-Api-Key） |
| `FDA483.PYTHON_BIN` | OCR 脚本 Python 解释器路径 |

详见 `references/config.md`。

## 依赖

- PHP >= 7.3（项目原栈为 ThinkPHP 5.0）
- Python 3 + `PyMuPDF`（fitz）、`tesseract`（eng 语言包）
- DeepSeek / 百度 / Google 翻译 Provider（用于中文企业名译英兜底）

## 安全说明

本技能已对所有密钥做**脱敏处理**（使用占位符与示例域名）。
请勿将包含真实 API Key 的源码与本技能一同发布到公开仓库。

## 许可

按所属项目许可发布。
