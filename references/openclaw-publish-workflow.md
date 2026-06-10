# OpenClaw 主题公众号发布工作流

> 来源 session：2026-06-08
> 场景：飞书文档（Markdown）+ OpenClaw 风格插图 → 微信公众号草稿箱

## 工作流概览

```
飞书文档 (Markdown) + 飞书插图
    ↓
下载到本地 (lark-cli drive +download)
    ↓
封面图裁剪 (900×383) + JPEG 转换 (避免 thumb 40006)
    ↓
组装 Markdown（frontmatter + 正文 + 文内插图 + 文末信息）
    ↓
wenyan publish (subprocess, OpenClaw CSS)
    ↓
公众号草稿箱
```

## 完整脚本

```python
import subprocess, os, json
from PIL import Image

# === 1. 下载素材 ===
# lark-cli drive +download --file-token <FILE_TOKEN> --output ./file.md --overwrite
# lark-cli drive +download --file-token <IMG_TOKEN> --output ./img.png --overwrite

# === 2. 封面图处理 ===
img = Image.open('illustration.png')
tw, th = 900, 383
r = img.size[0]/img.size[1]
tr = tw/th
if r > tr:
    new_w = int(img.size[1] * tr)
    left = (img.size[0] - new_w) // 2
    cropped = img.crop((left, 0, left + new_w, img.size[1]))
else:
    new_h = int(img.size[0] / tr)
    top = (img.size[1] - new_h) // 2
    cropped = img.crop((0, top, img.size[0], top + new_h))
cropped = cropped.resize((tw, th), Image.LANCZOS)
cropped.save('cover_wechat.png', 'PNG')
# 必须转 JPEG，否则 thumb 上传报 40006
cropped.convert('RGB').save('cover_thumb.jpg', 'JPEG', quality=85)

# === 3. 组装 Markdown ===
# frontmatter 必须包含 title 和 cover（本地绝对路径）
# 文内插图用 ![描述](本地绝对路径) 嵌入

# === 4. 发布（subprocess + 干净环境）===
env = os.environ.copy()
for k in ['http_proxy','https_proxy','HTTP_PROXY','HTTPS_PROXY']:
    env.pop(k, None)

creds = json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json'))
env['WECHAT_APP_ID'] = creds['appId']
env['WECHAT_APP_SECRET'] = creds['appSecret']

result = subprocess.run(
    ['/usr/local/bin/wenyan', 'publish',
     '-f', '/path/to/article.md',
     '-c', '/config/.hermes/skills/social-media/wechat-toolkit/references/openclaw-theme.css',
     '-h', 'solarized-light'],
    capture_output=True, text=True, timeout=90, env=env
)
print(result.stdout)  # 发布成功，Media ID: ...
```

## 关键要点

| 要点 | 说明 |
|------|------|
| **封面图规格** | 900×383 px (2.35:1)，先裁 PNG 再转 JPEG |
| **OpenClaw 主题** | 用 `-c` 指定 CSS 文件，不能用 `-t openclaw` |
| **文件路径** | 必须用 `-f /path`，不能只用 positional argument |
| **环境清理** | 必须清除所有 proxy 环境变量 |
| **调用方式** | 必须用 Python subprocess，不能用 Hermes terminal 工具 |
| **插图嵌入** | Markdown 中用 `![描述](本地绝对路径)` |

## OpenClaw 主题 CSS 路径

```
/config/.hermes/skills/social-media/wechat-toolkit/references/openclaw-theme.css
```

## 摘要写法（AIPY 风格）

从读者钩子出发，而非内容概括：

```yaml
---
title: 文章标题
cover: /path/to/cover.png
description: 上月OpenAI账单218美元，我只改了两行代码配置，没碰任何业务逻辑，账单降到64美元。
---
```

## 常见陷阱

1. **PNG 封面图报 40006** → 必须转 JPEG
2. **wenyan 超时** → 用 subprocess 而非 terminal 工具
3. **"未能找到文章标题"** → 用 `-f` flag 传文件路径
4. **插图不显示** → 使用本地绝对路径，非飞书 URL
