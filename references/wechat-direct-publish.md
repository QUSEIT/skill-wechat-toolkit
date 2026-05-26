# 微信 API 直接发布脚本（绕过 wenyan-cli）

当 `wenyan publish` 因 `fetch failed` 无法使用时，使用此脚本直接调用微信 API。

## 脚本：publish_wx.py

```python
#!/usr/bin/env python3
"""微信公众号直接发布脚本 - 绕过 wenyan-cli"""
import subprocess, json, re, urllib.request, os
from PIL import Image

CREDS_PATH = '/config/.hermes/home/.config/wenyan-md/credential.json'

def get_access_token():
    creds = json.load(open(CREDS_PATH))
    token_req = subprocess.run(['python3', '-c', (
        f'import urllib.request,json; '
        f'd=json.loads(urllib.request.urlopen(urllib.request.Request('
        f'"https://api.weixin.qq.com/cgi-bin/token'
        f'?grant_type=client_credential'
        f'&appid={creds["appId"]}'
        f'&secret={creds["appSecret"]}")).read()); '
        f'print(d["access_token"])'
    )], capture_output=True, text=True)
    return token_req.stdout.strip()

def upload_perm_material(file_path, access_token):
    """上传永久图片素材（用于 img src 的 CDN URL）"""
    result = subprocess.run([
        'curl', '-s', '--max-time', '30',
        '-F', f'media=@{file_path};type=image/jpeg',
        f'https://api.weixin.qq.com/cgi-bin/material/add_material?access_token={access_token}&type=image'
    ], capture_output=True, text=True)
    resp = json.loads(result.stdout)
    if 'media_id' in resp:
        return resp['media_id'], resp['url']
    raise Exception(f"上传失败: {json.dumps(resp)}")

def upload_temp_thumb(file_path, access_token):
    """上传临时封面图（用于 draft/add 的 thumb_media_id）"""
    result = subprocess.run([
        'curl', '-s', '--max-time', '30',
        '-F', f'file=@{file_path};type=image/png',
        f'https://api.weixin.qq.com/cgi-bin/media/upload?access_token={access_token}&type=thumb'
    ], capture_output=True, text=True)
    resp = json.loads(result.stdout)
    if 'thumb_media_id' in resp:
        return resp['thumb_media_id']
    raise Exception(f"thumb 上传失败: {json.dumps(resp)}")

def md_to_html(text, feishu_to_wx=None):
    """简单 Markdown → HTML 转换"""
    if feishu_to_wx:
        for fid, cdn_url in feishu_to_wx.items():
            text = text.replace(f'https://www.feishu.cn/file/{fid}', cdn_url)
    text = re.sub(r'\*\*(.+?)\*\*', r'<strong>\1</strong>', text)
    text = re.sub(r'```(\w*)\n(.+?)```', r'<pre><code>\2</code></pre>', text, flags=re.DOTALL)
    text = re.sub(r'`(.+?)`', r'<code>\1</code>', text)
    text = re.sub(r'^## (.+)$', r'<h2>\1</h2>', text, flags=re.MULTILINE)
    text = re.sub(r'^> (.+)', r'<blockquote>\1</blockquote>', text, flags=re.MULTILINE)
    text = re.sub(r'\n\n+', '\n', text)
    return text.strip()

def prepare_article(md_path, thumb_media_id, feishu_to_wx=None):
    """读取 md 文件，转换为 article payload"""
    content = open(md_path).read()
    lines = content.split('\n')
    start = next(i for i,l in enumerate(lines) if l.strip() == '---')
    end = next(i for i,l in enumerate(lines) if i > start and l.strip() == '---')
    body = '\n'.join(lines[end+1:])
    title = re.search(r'^# (.+)', content, re.MULTILINE).group(1)
    desc_match = re.search(r'^description:\s*(.+)$', content, re.MULTILINE)
    digest = desc_match.group(1) if desc_match else ''
    return {
        "title": title,
        "author": "爱派AI编程",
        "digest": digest,
        "content": md_to_html(body, feishu_to_wx),
        "content_source_url": "",
        "thumb_media_id": thumb_media_id,
        "need_open_comment": 1,
        "only_fans": 0,
    }

# === 主流程 ===
access_token = get_access_token()
print(f"Token: {access_token[:20]}...")

# 1. 上传所有图片为永久素材，获取 CDN URL
images = {
    'cover': '/tmp/cover-900x383.png',
    'illu1': '/tmp/illu1-800px.png',
    'illu2': '/tmp/illu2-800px.png',
    'qr': '/tmp/qr-wechat.png',
}
media_ids = {}
cdn_urls = {}
for name, path in images.items():
    media_id, cdn_url = upload_perm_material(path, access_token)
    media_ids[name] = media_id
    cdn_urls[name] = cdn_url
    print(f"{name}: {media_id}")

# 2. 上传临时 thumb（草稿 API 必须用这个）
thumb_media_id = upload_temp_thumb('/tmp/cover-900x383.png', access_token)
print(f"Thumb: {thumb_media_id}")

# 3. 构建 feishu→微信 URL 映射
feishu_to_wx = {
    'F8FObs4pMo2LiixsKPXceqgsnMc': cdn_urls['cover'],
    'WZL0b8JidoalhQxHP0ucVxk8nbg': cdn_urls['illu1'],
    'XepabtZvjoNcR4xs3w6cgDWFnFe': cdn_urls['illu2'],
}

# 4. 创建草稿
article = prepare_article('/tmp/article-创作手记五.md', thumb_media_id, feishu_to_wx)
payload = json.dumps({"articles": [article]}, ensure_ascii=False).encode('utf-8')
req = urllib.request.Request(
    f"https://api.weixin.qq.com/cgi-bin/draft/add?access_token={access_token}",
    data=payload, headers={'Content-Type': 'application/json'}
)
result = json.loads(urllib.request.urlopen(req, timeout=30).read())
print(f"Draft: {json.dumps(result)}")
```

## 当前状态（2026-05-26）

- 上传永久素材：✅ 成功（cover/illu1/illu2/qr 全部获得 media_id 和 CDN URL）
- 上传临时 thumb：✅ 成功（PNG 392KB 通过，JPEG 53KB 也通过）
- 创建草稿：❌ 报错 40007（thumb_media_id 格式不对，微信要求必须是临时 media_id）
- 根本原因：微信 draft/add API 只接受通过 `media/upload?type=thumb` 上传的临时 thumb media_id，不接受通过 `material/add_material?type=thumb` 上传的永久 thumb media_id

## 临时解法：手动发布

图片已全部获得 CDN URL，直接在 mp.weixin.qq.com 后台操作：
1. 新建草稿 → 手动上传封面图 → 粘贴正文（图片 src 用 CDN URL）
2. 永久素材 media_id（备用）：
   - cover: `x8t4d4GktbO7KNjgj7SgKS5KF_sTHBNuzhj28ZCNH4AnoTqHgk2GSEKIfTcZ6waV`
   - illu1: `x8t4d4GktbO7KNjgj7SgKYoI7s_XuPuoAjXeFKPq8a76CTCKcT76J2_TRUaEQR5Q`
   - illu2: `x8t4d4GktbO7KNjgj7SgKQBCi8dGLCiGQZSbfnjSJZTOx2Z0EMXzwXO5D6xssORZ`
   - qr: `x8t4d4GktbO7KNjgj7SgKWQwbGlN2h88hfN326C7xarLjR0Gfu2l_4hj8N3V1AK4`