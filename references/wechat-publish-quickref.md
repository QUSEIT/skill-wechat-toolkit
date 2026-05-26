# 微信公众号文章发布速查

## 发布命令

```bash
export WECHAT_APP_ID=$(python3 -c "import json; c=json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json')); print(c.get('appId',''))")
export WECHAT_APP_SECRET=$(python3 -c "import json; c=json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json')); print(c.get('appSecret',''))")
wenyan publish -f <文章.md> -t <主题> -h <代码高亮>
```

## 主题

| 主题 | 风格 | 推荐场景 |
|------|------|----------|
| `lapis` | 青金石蓝 | 技术文章 |
| `phycat` | 薄荷绿 | 课程/教育内容 |
| `orangeheart` | 橙色 | 温暖风格 |
| `purple` | 紫色 | 创意内容 |

## 图片规格

| 类型 | 尺寸 | 说明 |
|------|------|------|
| 封面图 | 900×383 px (2.35:1) | 必须裁剪适配 |
| 文内配图 | 宽≤800px | 保持比例 |
| 二维码 | 保持原尺寸 | 文末课程引流 |

### Python 图片处理（需要 pillow）

```bash
uv run --with pillow python3 -c "
from PIL import Image
img = Image.open('/tmp/source.png')

# 封面图裁剪 (2.35:1)
target_w, target_h = 900, 383
ratio = img.size[0]/img.size[1]
target_ratio = target_w/target_h
if ratio > target_ratio:
    new_w = int(img.size[1] * target_ratio)
    left = (img.size[0] - new_w) // 2
    cropped = img.crop((left, 0, left + new_w, img.size[1]))
else:
    new_h = int(img.size[0] / target_ratio)
    top = (img.size[1] - new_h) // 2
    cropped = img.crop((0, top, img.size[0], top + new_h))
cropped.resize((target_w, target_h), Image.LANCZOS).save('/tmp/cover-wechat.png', 'PNG')

# 文内配图 (宽800px)
max_w = 800
r = min(max_w/img.size[0], 1)
new_size = (int(img.size[0]*r), int(img.size[1]*r))
img.resize(new_size, Image.LANCZOS).save('/tmpInner-w800.png', 'PNG')
"
```

## 飞书文件下载

```bash
# lark-cli 下载（注意：是 drive +download，不是 file +download）
cd /tmp && npm install @larksuite/cli
node_modules/.bin/lark-cli drive +download --file-token <TOKEN> --output=./output.png
```

常见错误：
- ❌ `--file` → ✅ `--file-token`
- ❌ 绝对路径 → ✅ 先 cd 到目标目录，或用相对路径

## 文末课程二维码标准格式

```markdown
---

![AI协作编程入门课](本地二维码路径)

如果你曾对Python满怀热忱，却因种种原因没能学会，那么我相信：在AI时代，这门课就是你真正掌握Python的最好机会。
```

二维码飞书文件 Token：`HSvbbTFjnoMytJxhnQncudWNnrg`

## 文章 frontmatter 格式

```yaml
---
title: 文章标题
cover: /tmp/cover-wechat.png
---
```

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| 230001 JSON错误 | shell拼接JSON引号转义问题 | 用 Python json.dumps() 或直接用 @file 语法 |
| 400 invalid media type | 封面图格式问题 | 确保PNG/JPG，不超过5MB |
| 发布成功但图片空白 | 图片路径错误 | 使用本地绝对路径 |
