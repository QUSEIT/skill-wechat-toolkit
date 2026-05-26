# 故障排查指南

## 1. ❌ IP 不在白名单

**错误信息：**
```
Error: invalid ip 171.37.44.131 ipv6 ::ffff:171.37.44.131, not in whitelist
```

**原因：** 服务器 IP `171.37.44.131`（2026-05-25 实测）未添加到微信公众号后台白名单。

**当前 IP：`171.37.44.131`**（旧 IP `171.36.17.27` 已失效，如遇报错请先确认当前 IP）

**解决方法：**

登录微信公众号后台 → 设置与开发 → 安全中心 → IP白名单 → 添加 `171.37.44.131`

**验证 IP：**
```bash
curl -s ifconfig.me
```

---

## 2. ❌ wenyan-cli 未安装

**错误信息：**
```
wenyan: command not found
```

**解决方法：**
```bash
npm install -g @wenyan-md/cli
wenyan --version  # 验证安装
```

---

## 3. ❌ Frontmatter 缺失

**错误信息：**
```
Error: 未能找到文章封面
```

**原因：** Markdown 文件缺少必需的 frontmatter（`title` 字段必填）。

**格式要求：**
```markdown
---
title: 文章标题（必填）
cover: /absolute/path/to/cover.jpg（可选）
description: 约120字摘要（发布后手动填入微信后台）
---

# 正文...
```

---

## 4. ❌ 图片下载失败

**错误信息：**
```
下载图片失败 URL: https://mmbiz.qpic.cn/...
```

**原因：** 微信图片 URL 含特殊参数导致下载失败。

**解决方法：** 使用可靠图源（如 `https://picsum.photos/1080/864`）作为测试封面。

---

## 5. ❌ 封面图规格

**公众号封面图要求：**
- 尺寸：900×383 像素（2.35:1 宽屏比例）
- 格式：PNG/JPG，不超过 5MB

**快速裁剪脚本：**
```bash
uv run --with pillow python3 -c "
from PIL import Image
img = Image.open('/path/to/source.png')
w, h = 900, 383
ratio = w/h
curr_ratio = img.size[0]/img.size[1]
if curr_ratio > ratio:
    new_w = int(img.size[1] * ratio)
    cropped = img.crop(((img.size[0]-new_w)//2, 0, (img.size[0]+new_w)//2, img.size[1]))
else:
    new_h = int(img.size[0] / ratio)
    cropped = img.crop((0, (img.size[1]-new_h)//2, img.size[0], img.size[1]))
cropped.save('/path/to/cover-wechat.png', 'PNG')
"
```

---

## 6. 凭证存储位置

Hermes 环境下凭证存储于：
```
/config/.hermes/home/.config/wenyan-md/credential.json
```

格式：
```json
{
  "appId": "wx367d428ae38d132f",
  "appSecret": "18d3866f6643444bc38832f5bad5d2e9"
}
```

**快速读取凭证发布：**
```bash
export WECHAT_APP_ID=$(python3 -c "import json; print(json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json'))['appId'])")
export WECHAT_APP_SECRET=$(python3 -c "import json; print(json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json'))['appSecret'])")
wenyan publish -f /path/to/article.md -t lapis
```

---

## 7. ❌ 40001 invalid credential (token 过期)

**错误信息：**
```
40001: invalid credential, access_token is invalid or not latest
```

**原因：** wenyan-cli 缓存的 `access_token` 过期失效。

**解决方法：** 删除本地 token 缓存后重试：
```bash
rm -f /config/.hermes/home/.config/wenyan-md/token.json
export WECHAT_APP_ID=$(python3 -c "import json; print(json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json'))['appId'])")
export WECHAT_APP_SECRET=$(python3 -c "import json; print(json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json'))['appSecret'])")
wenyan publish -f /path/to/article.md -t phycat -h solarized-light
```

> ⚠️ 每次遇到 40001 都需先删 token.json，wenyan-cli 不会自动刷新。

---

## 8. Node.js 版本

要求 ≥ 18，已验证支持 v22。

---

## 9. ❌ lark-cli 文件下载路径限制

**错误信息：**
```
unsafe file path: --file must be a relative path within the current directory
```

**原因：** lark-cli 的 `--output` 参数只接受相对路径，不接受绝对路径。

**解决方法：** 先 `cd /tmp`，然后用相对路径：
```bash
cd /tmp && /tmp/node_modules/.bin/lark-cli drive +download --file-token <TOKEN> --output ./filename.png --overwrite
```