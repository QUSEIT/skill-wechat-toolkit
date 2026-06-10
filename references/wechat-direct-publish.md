# 微信 API 直接发布脚本（绕过 wenyan-cli）

当 `wenyan publish` 不可用时（如 proxy 问题），使用此脚本直接调用微信 API。

## 凭证

```
/config/.hermes/home/.config/wenyan-md/credential.json
```

## 关键经验（2026-05-26）

### 1. 封面图必须转 JPEG，不能用 PNG

微信 `media/upload?type=thumb` API 对 PNG 有额外格式校验，会报 `40006 invalid media size hint`。

```python
from PIL import Image
img = Image.open('/tmp/cover-wechat.png').convert('RGB')
img.save('/tmp/cover-thumb.jpg', 'JPEG', quality=85)
```

### 2. wenyan publish 必须用 subprocess 调用

Hermes Terminal 工具会残留 proxy 环境变量，导致 wenyan-cli 超时。必须用 Python subprocess 在干净环境中调用：

```python
import subprocess, os, json

env = os.environ.copy()
env.pop('http_proxy', None); env.pop('https_proxy', None)
env.pop('HTTP_PROXY', None); env.pop('HTTPS_PROXY', None)
creds = json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json'))
env['WECHAT_APP_ID'] = creds['appId']
env['WECHAT_APP_SECRET'] = creds['appSecret']

result = subprocess.run(
    ['/usr/local/bin/wenyan', 'publish', '-f', '/tmp/article.md', '-t', 'phycat', '-h', 'solarized-light'],
    capture_output=True, text=True, timeout=90, env=env
)
print(result.stdout)
```

### 3. wenyan publish 正确调用方式（2026-06-05 实测可行）

**必须同时满足三个条件**：
1. 清理所有 proxy 环境变量
2. 使用 `-f` flag 传文件路径
3. 用 subprocess 调用（不用 terminal 工具）

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
     '-f', '/tmp/article-wechat.md',   # ← 必须用 -f，不能只用 positional
     '-t', 'phycat', '-h', 'solarized-light'],
    capture_output=True, text=True, timeout=90, env=env
)
print(result.stdout)  # 发布成功，Media ID: ...
```

**❌ 错误做法（均报"未能找到文章标题"）**：
- 文件作为纯 positional argument（无 `-f`）：`wenyan publish /tmp/article.md`
- 通过 terminal 工具调用（即使 unset 代理）
- 使用 `-f` 但通过 terminal 而非 subprocess

### 4. draft/add API 40007 错误

这个错误通常不是 thumb_media_id 无效，而是 token 问题（需要用 getStableAccessToken 或重新获取）。遇到 40007 时：
1. 先确认 access_token 是最新获取的
2. 换用 wenyan publish（它内部会重新获取 token）

## 已知成功的发布流程

1. 用 subprocess + 干净环境调用 `wenyan publish` → 成功
2. 直接 API 调用 thumb 上传（PNG）→ 失败（40006）
3. 直接 API 调用 thumb 上传（JPEG）→ 成功
4. 直接 API 调用 draft/add → 失败（40007，token 问题）

**结论**：优先使用 wenyan publish（subprocess 方式），不要尝试直接 API 创建草稿。

## 图片 CDN URL 映射

飞书图片替换为微信 CDN URL：

| 飞书 file_token | 用途 | 本地路径 |
|----------------|------|---------|
| F8FObs4pMo2LiixsKPXceqgsnMc | 封面图 | /tmp/cover-wechat.png |
| WZL0b8JidoalhQxHP0ucVxk8nbg | 文内插画1 | /tmp/img1-w800.png |
| XepabtZvjoNcR4xs3w6cgDWFnFe | 文内插画2 | /tmp/img2-w800.png |
| HSvbbTFjnoMytJxhnQncudWNnrg | 课程二维码 | /tmp/qr-code.png |

## 课程二维码 Token

```
HSvbbTFjnoMytJxhnQncudWNnrg
```

下载命令：
```bash
cd /tmp && lark-cli drive +download --file-token HSvbbTFjnoMytJxhnQncudWNnrg --output=./qr-code.png --overwrite
```