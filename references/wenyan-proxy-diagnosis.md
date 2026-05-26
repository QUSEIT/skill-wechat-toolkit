# wenyan publish fetch failed 诊断记录（2026-05-26）

## 症状

```
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
wenyan publish -f /tmp/article-创作手记五.md -t phycat
# → fetch failed
```

注意：即使完全清除代理环境变量，publish 仍然失败。根因不是系统级代理，而是 wenyan-cli 内部的 ProxyAgent 行为。

## 诊断过程

| 测试 | 结果 | 结论 |
|------|------|------|
| `curl api.weixin.qq.com` | 200 | 微信API直连正常 |
| `unset proxy && wenyan publish` | `fetch failed` | 清除系统代理后仍失败 |
| `curl ifconfig.me` | 171.37.44.131 | 本机公网IP确认 |
| `wenyan serve` 健康检查 | `{"status":"ok"}` | 本地服务正常 |
| `wenyan publish --server http://localhost:3050` | `未提供 AppID` | 本地server路由有效，但AppID传不过去 |
| Python urllib 直连微信API | 成功（token+图片上传均OK） | 微信API完全可达，wenyan-cli 的 fetch 被拦截 |
| `wenyan publish --proxy ""` | 仍 `fetch failed` | `--proxy ""` 参数无效（CLI不识别） |

## 根因

wenyan-cli v2.0.8 使用 `undici`，内部调用 `setGlobalDispatcher(new ProxyAgent())`。ProxyAgent 自动从 `process.env.http_proxy` / `process.env.https_proxy` 读取代理地址。关键问题：**wenyan 不尊重 `--proxy ""` 参数，且没有机制禁用 ProxyAgent**。清除系统代理环境变量对 wenyan 内部的 ProxyAgent 无效，因为 ProxyAgent 可能在 CLI 启动时已初始化。

## 解法

### 方案A（推荐）：Python 脚本直接调用微信 API

绕过 wenyan-cli，直接用 Python urllib 调用微信 API。经验证可行：
- 获取 access_token ✅
- 上传临时 thumb（type=thumb） ✅（PNG 392KB 成功，JPEG 53KB 也成功）
- 上传永久图片素材（type=image，media字段） ✅
- 创建草稿（draft/add）⚠️ 报错 40007（thumb_media_id 格式要求）

**完整发布脚本**：`references/wechat-direct-publish.md`

### 方案B：手动发布（依赖微信后台）

1. 登录 mp.weixin.qq.com → 草稿箱 → 新建草稿
2. 手动上传封面图（900×383px）和文内配图
3. 粘贴文章内容（图片已通过 `references/wechat-direct-publish.md` 中的脚本上传到 CDN）

### 方案C：修复 wenyan-cli 代理问题（未解决）

尝试过但无效：
- `unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY` → 仍 fetch failed
- `wenyan publish --proxy ""` → CLI 不识别 `--proxy` 参数（它只有 `--server` 和 `--api-key`）
- 本地 wenyan serve（3050端口）启动正常，但 publish --server 时 AppID 无法传递

---

## 关键发现（影响 future sessions）

1. **wenyan publish 无法绕过**：没有 `--proxy` 参数可以禁用 ProxyAgent，清除系统代理也无效
2. **微信草稿 API 对 thumb_media_id 有特殊要求**：必须是通过 `media/upload?type=thumb` 上传的临时 media_id，不接受永久素材 media_id（报错 40007 invalid media_id）
3. **临时 thumb 上传有 size 限制**：PNG 392KB 成功，JPEG 53KB 也成功（限制可能是 2MB）
4. **永久图片素材上传**：必须用 `media` 字段（不是 `file` 字段）才能成功（errcode 41005 media data missing）
5. **cover 图建议用 PNG**：PNG 格式上传 thumb 成功率高于 JPEG

## 已知 workaround

获取 access_token 后：
```bash
# 上传临时 thumb（用于 draft/add）
curl -F "file=@/tmp/cover.png;type=image/png" \
  "https://api.weixin.qq.com/cgi-bin/media/upload?access_token=TOKEN&type=thumb"

# 上传永久图片素材（用于后续 img 标签 src）
curl -F "media=@/tmp/cover.png;type=image/png" \
  "https://api.weixin.qq.com/cgi-bin/material/add_material?access_token=TOKEN&type=image"
```

## 相关进程状态

- wenyan serve：proc_0745eef71f4e（port 3050），health OK
- 本机公网IP：171.37.44.131
- 微信公众号 AppID：wx367d428ae38d132f
- 封面图永久 media_id：`x8t4d4GktbO7KNjgj7SgKS5KF_sTHBNuzhj28ZCNH4AnoTqHgk2GSEKIfTcZ6waV`