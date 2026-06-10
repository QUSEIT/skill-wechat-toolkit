# OpenClaw 红色主题 + wechat-toolkit 使用指南

> 本文件补充 wechat-toolkit SKILL.md，记录 2026-06-03 验证通过的 OpenClaw 红色/橘红主题正确用法。

---

## OpenClaw 主题背景

`wenyan theme -l` 内置主题列表中**没有 openclaw**。OpenClaw 是自定义 CSS 主题（橘红→红色渐变），通过独立 CSS 文件提供。

CSS 文件路径：`/config/.hermes/skills/social-media/wechat-toolkit/references/openclaw-theme.css`

---

## 正确用法：`-c` 指定 CSS 文件

```bash
# ❌ 错误 - 会报错 "主题不存在: openclaw"
wenyan publish -f /tmp/article.md -t openclaw

# ✅ 正确 - 用 -c 指定 CSS 文件（可用于 publish 和 render）
wenyan publish -f /tmp/article.md \
  -c /config/.hermes/skills/social-media/wechat-toolkit/references/openclaw-theme.css \
  -h solarized-light

wenyan render -f /tmp/article.md \
  -c /config/.hermes/skills/social-media/wechat-toolkit/references/openclaw-theme.css \
  -h solarized-light
```

支持的 alias（publish.js 源码映射）：
- `openclaw` → openclaw-theme.css
- `橙色` → openclaw-theme.css
- `橘红` → openclaw-theme.css
- `红色` → openclaw-theme.css

但由于 `-t` 参数不支持 alias 扩展，**无论 alias 还是 theme 名都应统一用 `-c` 指定 CSS 路径**。

---

## Python subprocess 调用（含完整凭证注入）

```python
import subprocess, os, json

env = os.environ.copy()
env.pop('http_proxy', None); env.pop('https_proxy', None)
env.pop('HTTP_PROXY', None); env.pop('HTTPS_PROXY', None)

creds = json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json'))
env['WECHAT_APP_ID'] = creds['appId']
env['WECHAT_APP_SECRET'] = creds['appSecret']

result = subprocess.run(
    ['/usr/local/bin/wenyan', 'publish',
     '-f', '/tmp/article.md',
     '-c', '/config/.hermes/skills/social-media/wechat-toolkit/references/openclaw-theme.css',
     '-h', 'solarized-light'],
    capture_output=True, text=True, timeout=90, env=env
)
print(result.stdout)
# 成功输出: "发布成功，Media ID: x8t4d4GktbO..."
```

**注意**：
- 必须从 `credential.json` 读取凭证，不能内联 AppID/AppSecret
- 必须清除代理环境变量（wenyan-cli 内部 undici 有 ProxyAgent，无法通过 env var 禁用）
- 封面图需转为 JPEG（PNG 报 40006 thumb 上传错误）

---

## 本次验证记录（2026-06-03）

| 文章 | 主题 | 命令 | 结果 |
|------|------|------|------|
| 使用 Claude 构建多代理工作流 | openclaw (红色) | wenyan publish -c openclaw-theme.css | ✅ 成功，Media ID: x8t4d4GktbO7KNjgj7SgKYWX6B7zCIiRnnLmqU1S7DNp34FcQ7U0lssUotbyPxAu |