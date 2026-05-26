# 飞书文档 → 微信公众号 发布流程

本文档记录从飞书获取素材到发布微信公众号草稿箱的完整流程。

---

## 快速流程概览

```
1. 下载飞书文档内容（lark-cli docs +fetch）
2. 下载飞书图片（lark-cli drive +download）
3. 图片预处理（封面裁剪、文内图缩放、二维码保持原尺寸）
4. 组装 Markdown（含 frontmatter）
5. 预览（wenyan render）
6. 发布（wenyan publish）
7. 微信后台确认并发布
```

---

## 步骤详解

### 1. 获取飞书文档内容

```bash
cd /tmp && node_modules/.bin/lark-cli docs +fetch \
  --doc <DOC_TOKEN> --format pretty 2>&1
```

> ⚠️ 返回为 Markdown 格式，可直接重定向到文件。

### 2. 下载飞书图片

```bash
cd /tmp && node_modules/.bin/lark-cli drive +download \
  --file-token <FILE_TOKEN> --output=./output.png --overwrite
```

### 3. 图片预处理规格

| 类型 | 规格 | 命令 |
|------|------|------|
| 封面图 | 900×383 px（2.35:1） | 裁剪 + 缩放 |
| 文内配图 | 宽≤800px（保持比例） | 等比缩放 |
| 二维码 | 保持原尺寸 | 不处理 |

**封面图裁剪**（Python/Pillow）：
```python
from PIL import Image
img = Image.open('/tmp/cover.png')
tw, th = 900, 383
r = img.size[0]/img.size[1]
tr = tw/th
if r > tr:
    new_w = int(img.size[1] * tr)
    cropped = img.crop(((img.size[0]-new_w)//2, 0, (img.size[0]+new_w)//2, img.size[1]))
else:
    new_h = int(img.size[0] / tr)
    cropped = img.crop((0, (img.size[1]-new_h)//2, img.size[0], (img.size[1]+new_h)//2))
cropped.resize((tw, th), Image.LANCZOS).save('/tmp/cover-wechat.png', 'PNG')
```

### 4. Markdown frontmatter 格式

```yaml
---
title: 文章标题
description: 约120字摘要（发布后需手动填入微信后台摘要框）
cover: /tmp/cover-wechat.png
---
```

> ⚠️ wenyan-cli **不传递 description 到微信后台**，需发布后在网页端手动粘贴。

### 5. 文内配图插入位置原则

- 内容区顶部：一般**不**放重复大标题（公众号会自动显示）
- 配图：插入到对应章节的**第一段之后**
- 文末二维码：正文最后、签名/版权信息之前

### 6. 发布命令

```bash
# 预览（不发布）
wenyan render -f /tmp/article.md -t phycat -h solarized-light

# 发布（AIPY 默认 phycat）
WECHAT_APP_ID=wx367d428ae38d132f WECHAT_APP_SECRET=18d3866f6643444bc38832f5bad5d2e9 \
  wenyan publish -f /tmp/article.md -t phycat -h solarized-light
```

> ⚠️ 使用内联 `KEY=VALUE` 形式传 env vars，不要用 `$(...)` 子shell（Hercules 会话中会报 `unexpected EOF`）。

---

## 迭代修改记录（本次会话）

| 版本 | 变更内容 |
|------|----------|
| v1 | 初始发布 |
| v2 | 换绿色主题（phycat） |
| v3 | 两张内图分别插入「场景切入」「第一课教什么」两节 |
| v4 | 移除内容区重复标题，添加文末二维码 |
| v5 | 添加 description frontmatter 作为摘要方案 |

---

## 主题选择

| 主题 | 色调 | 适用场景 |
|------|------|----------|
| `phycat` | 薄荷绿 | 默认推荐，清晰层次 |
| `lapis` | 青金石蓝 | 技术文章 |
| `orangeheart` | 橙色 | 温暖风格 |
| `purple` | 紫色 | 文艺风格 |

---

## AI协作编程入门课 二维码

- **飞书文件 Token**：`HSvbbTFjnoMytJxhnQncudWNnrg`
- **下载**：`lark-cli drive +download --file-token HSvbbTFjnoMytJxhnQncudWNnrg --output=./qrcode.png`
- **文末文案**：「如果你曾对Python满怀热忱，却因种种原因没能学会，那么我相信：在AI时代，这门课就是你真正掌握Python的最好机会。」
- **插入位置**：正文最后、签名/版权信息之前
