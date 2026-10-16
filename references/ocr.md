# Python OCR 脚本要点

文件：`data/fda483/fda483_ocr.py`（扫描件 PDF，无文本层，需 OCR）

## 依赖

- `PyMuPDF`（`fitz`）：PDF → PNG 渲染
- `tesseract` + 英文语言包（`eng`）：OCR 引擎，默认路径
  `C:\Program Files\Tesseract-OCR\tesseract.exe`

## 用法

```bash
python fda483_ocr.py <input.pdf> <output.txt> [--dpi 200] [--max-pages 40] [--tesseract <path>]
```

## 流程

1. 用 fitz 逐页 `get_pixmap(dpi=200)` 渲染为 PNG（临时目录）；
2. 每页 `tesseract <img> stdout -l eng --psm 3` 提取英文文本；
3. 文本合并写入 `output.txt`，进度打印到 stderr；
4. 清理临时 PNG。

## 退出码

- `0`：成功
- `1`：异常（stderr 输出 `OCR_ERROR: ...`）
- `2`：提取字符数 < 50（视为空白扫描件/失败）

## 调用约定（PHP 侧）

```php
$cmd = escapeshellarg($python) . ' ' . escapeshellarg($script) . ' '
     . escapeshellarg($pdfPath) . ' ' . escapeshellarg($outTxt) . ' --max-pages 40 2>&1';
exec($cmd, $out, $code);
// $code !== 0 或输出文件不存在 → 抛错
```
