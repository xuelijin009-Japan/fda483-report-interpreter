# 环境变量与配置（脱敏）

> ⚠️ 以下所有密钥/Key 必须从环境变量或 `.env` 读取，**禁止提交真实值到仓库**。
> 原项目曾在 `syncFda483FromApi()` 中硬编码 API Key，已在本技能脱敏处理。

## 需要的环境变量

| 变量名 | 用途 | 示例 |
|--------|------|------|
| `DEEPSEEK.API_KEY` | DeepSeek 翻译 + 解读 | `sk-xxxx`（OpenAI 兼容格式） |
| `BAIDU_TRANSLATE.APPID` | 百度翻译备选（可选） | — |
| `FDA483.API_KEY` | 483 数据源 API 鉴权（X-Api-Key） | 服务端下发，勿入库 |
| `FDA483.PYTHON_BIN` | OCR 脚本 Python 解释器路径 | 见下方默认 |

## 数据源 API（同步用）

- URL：`https://<data-service>/api/v1/fda-foia-inspection-reports`（示例用占位域名）
- 鉴权头：`X-Api-Key: {FDA483.API_KEY}`
- 请求体（分页 POST）：

```json
{
  "page": 1,
  "pageSize": 100,
  "params": { "company_name": "<企业名>" },
  "orderBy": "create_time",
  "orderDirection": "DESC"
}
```

- 响应：`{ "code": 200, "data": { "total": N, "list": [ ... ] } }`
- 注意：API 不支持按企业名过滤，需全量拉取后本地按 `company_name LIKE` 筛选。
  全量同步最多 5 页（500 条），增量同步 1 页；按 `remote_id` 去重。

## PDF 下载

- 非浏览器 UA 才能下载 FDA 官网附件：`FDA483-Fetcher/1.0 (crhms data-service)`
- 本地缓存目录：`runtime/fda483/`，已下载且 > 1KB 则复用。

## OCR Python 路径默认值

```
FDA483.PYTHON_BIN = C:/Users/Administrator/.workbuddy/binaries/python/envs/default/Scripts/python.exe
```

（按实际部署环境覆盖；脚本位于 `data/fda483/fda483_ocr.py`，依赖 `PyMuPDF` + `tesseract`）
