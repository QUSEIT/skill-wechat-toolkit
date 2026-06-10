# 飞书文档 + OpenClaw 封面 → 微信公众号发布实录

> 来源 session：2026-06-10
> 任务：将飞书文档《Claude Code 最佳实践》+ OpenClaw 风格封面插画发布到微信公众号
> 结果：✅ 发布成功，Media ID: x8t4d4GktbO7KNjgj7SgKZEaBGcmw3L6fVB4MGMQddgTRAdSD2k3oiLd7K_UPeO1

## 工作流概览

```
飞书文档 (Markdown) + 飞书封面图 (PNG)
    ↓
lark-cli drive +download（注意：相对路径 + 扩展名不可信）
    ↓
验证文件类型（file 命令）→ 发现"图片"实际是 Markdown
    ↓
封面图裁剪为 900×383 (2.35:1) + JPEG 转换（避免 thumb 40006）
    ↓
组装 Markdown（frontmatter + cover 路径指向 JPEG）
    ↓
wenyan publish（subprocess + 干净环境 + -f flag）
    ↓
公众号草稿箱
```

## 关键步骤详解

### 1. 下载素材

```bash
# 文章正文（URL 以 .png 结尾，但实际是 Markdown）
cd /config/Desktop
lark-cli drive +download --file-token UJfgbqLcZodT8xxW81Xcbf2LnKd --output article.md --overwrite
# → 实际下载的是 Markdown 文本（9932 bytes）

# 封面插画
cd /config/Desktop
lark-cli drive +download --file-token TZtEbhFpuo49IwxVGMkcnxjAnOc --output cover_source.png --overwrite
# → 实际是 PNG 图片（1326092 bytes）
```

**陷阱**：飞书文件 URL 的扩展名不可信。第一个 token 的 URL 以 `.png` 结尾，但实际内容是 Markdown 文本。下载后必须用 `file` 命令验证。

### 2. 封面图预处理

```python
from PIL import Image

img = Image.open('/config/Desktop/cover_source.png')
tw, th = 900, 383
r = img.size[0] / img.size[1]
tr = tw / th

if r > tr:
    new_w = int(img.size[1] * tr)
    left = (img.size[0] - new_w) // 2
    cropped = img.crop((left, 0, left + new_w, img.size[1]))
else:
    new_h = int(img.size[0] / tr)
    top = (img.size[1] - new_h) // 2
    cropped = img.crop((0, top, img.size[0], top + new_h))

cropped = cropped.resize((tw, th), Image.LANCZOS)
cropped.save('/config/Desktop/cover_wechat.png', 'PNG')
# 必须转 JPEG，避免微信 thumb 上传 40006
cropped.convert('RGB').save('/config/Desktop/cover_thumb.jpg', 'JPEG', quality=85)
```

### 3. 组装 Markdown

```markdown
---
title: Claude Code 最佳实践：如何把 AI 编程效率提升 3 倍
cover: /config/Desktop/cover_thumb.jpg
description: 用 Claude Code 三个月，我从"看着它写"变成"告诉它验证"——效率提升的关键不是 prompt 技巧，而是给 AI 一个能跑通的检查。
---

从配置环境到规模化部署，这是我在实际项目中踩坑总结出的完整工作流...
```

**注意**：
- `cover` 指向 JPEG 文件（非 PNG），避免 40006 错误
- `description` 写入 frontmatter，wenyan 会读取并作为摘要
- 内容区顶部不放重复大标题

### 4. 发布（subprocess 方式）

```python
import subprocess, os, json

env = os.environ.copy()
for k in ['http_proxy','https_proxy','HTTP_PROXY','HTTPS_PROXY']:
    env.pop(k, None)

creds = json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json'))
env['WECHAT_APP_ID'] = creds['appId']
env['WECHAT_APP_SECRET'] = creds['appSecret']

result = subprocess.run(
    ['/usr/local/bin/wenyan', 'publish',
     '-f', '/config/Desktop/article_wechat.md',
     '-c', '/config/.hermes/skills/social-media/wechat-toolkit/references/openclaw-theme.css',
     '-h', 'solarized-light'],
    capture_output=True, text=True, timeout=90, env=env
)
print(result.stdout)
# → 发布成功，Media ID: x8t4d4GktbO7KNjgj7SgKZEaBGcmw3L6fVB4MGMQddgTRAdSD2k3oiLd7K_UPeO1
```

**关键**：
- 必须用 `-f` flag 传入文件路径
- 必须用 subprocess（不能用 Hermes terminal 工具）
- 必须清除所有 proxy 环境变量
- OpenClaw 主题用 `-c` 指定 CSS 文件

## 本次新发现

### 1. 飞书文件扩展名陷阱

飞书云盘文件的 URL 扩展名不可信。一个以 `.png` 结尾的 URL 实际返回的是 Markdown 文本。下载后必须验证文件类型。

### 2. 当前服务器 IP

2026-06-10 实测：`171.37.45.170`

### 3. wenyan publish 成功模式确认

`-f` flag + subprocess + 干净环境 = 发布成功。Positional argument（无 `-f`）会报"未能找到文章标题"。

## 相关文件

- 文章源文件：`/config/Desktop/article.md`
- 封面源图：`/config/Desktop/cover_source.png`
- 处理后封面：`/config/Desktop/cover_thumb.jpg`
- 发布用 Markdown：`/config/Desktop/article_wechat.md`
