# 中英文品牌词提取

FDA 数据源（FOIA Inspection）的企业名多为英文，而用户输入多为中文企业名。
需做两层匹配：先中文关键词，再译英兜底。

## 中文品牌核心词提取 `extractBrandKeyword($name)`

逐条替换后缀（长后缀放前面优先匹配），再 trim，剩余不足 2 字取原名前 4 字兜底。

```php
private function extractBrandKeyword(string $name): string
{
    $suffixes = [
        '药业集团股份有限公司', '药业集团有限公司', '药业股份有限公司',
        '制药集团股份有限公司', '制药集团有限公司', '制药股份有限公司',
        '医药集团股份有限公司', '医药集团有限公司', '医药股份有限公司',
        '生物科技有限公司', '生物技术股份有限公司', '生物制药有限公司',
        '制药有限公司', '药业有限责任公司', '制药有限责任公司',
        '医药有限责任公司', '药业公司', '制药公司', '医药公司',
        '集团股份', '集团有限', '有限公司', '股份有限',
    ];
    $keyword = $name;
    foreach ($suffixes as $suffix) {
        $keyword = str_replace($suffix, '', $keyword);
    }
    $keyword = trim($keyword);
    if (mb_strlen($keyword) < 2) {
        return mb_substr($name, 0, 4);
    }
    return $keyword;
}
```

## 英文品牌核心词提取 `extractEnglishBrandKeyword($name)`

剥离英文通用后缀，最多 3 轮嵌套剥离 + 修剪尾部标点。

```php
private function extractEnglishBrandKeyword(string $name): string
{
    $name = trim($name);
    $suffixes = [
        'Pharmaceutical Group Co., Ltd.', 'Pharmaceutical Group Co. Ltd.',
        'Pharmaceutical Co., Ltd.', 'Pharmaceutical Co. Ltd.',
        'Pharmaceutical Co., Limited', 'Pharmaceutical Co.,Ltd.',
        'Pharmaceutical Technology Co., Ltd.', 'Pharmaceutical Technology Co. Ltd.',
        'Pharmaceutical Holdings Co., Ltd.', 'Biopharmaceutical Co., Ltd.',
        'Biopharmaceutical Co., Ltd', 'Bio-Tech Co., Ltd.', 'Bio-Tech Co. Ltd.',
        'Biotech Co., Ltd.', 'Pharmaceuticals Co., Ltd.', 'Pharmaceuticals, Inc.',
        'Pharmaceutical Co., Inc.', 'Pharmaceutical Co.', 'Pharmaceutical Group',
        'Pharmaceutical, Inc.', 'Pharmaceutical Inc.', 'Pharmaceutical',
        'Co., Ltd.', 'Co. Ltd.', 'Co.,Ltd.', 'Co., Limited', 'Co., Inc.', 'Co.,Inc.',
        'Corporation', 'Corp.', 'Corp',
        'Incorporated', 'Inc.', 'Inc',
        'Limited', 'Ltd.', 'Ltd',
        'Company', 'Co.', 'Co',
    ];
    $keyword = $name;
    foreach ($suffixes as $suffix) {
        $suffixLen = strlen($suffix);
        $kwLen     = strlen($keyword);
        if ($suffixLen > 0 && $kwLen > $suffixLen && strcasecmp(substr($keyword, -$suffixLen), $suffix) === 0) {
            $keyword = substr($keyword, 0, $kwLen - $suffixLen);
            $keyword = rtrim($keyword);
            break; // 每次只剥一个，外层再跑一轮处理剩余后缀
        }
    }
    for ($round = 0; $round < 3; $round++) {
        $trimmed = rtrim($keyword, " \t\n\r\0\x0B.,，。");
        if ($trimmed === $keyword) break;
        $keyword = $trimmed;
    }
    return $keyword;
}
```

## 中文 → 英文候选词构建 `buildEnglishCandidates($chineseName)`

1. 翻译**完整企业名** → 取其英文品牌核心词；
2. 翻译**中文品牌核心词**（更稳定、更易命中）→ 取其英文核心词。
3. 任意一步失败仅 `error_log` 不中断，最终返回去重候选列表。
