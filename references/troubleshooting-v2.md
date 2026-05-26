# 故障排查补充（2026-05-26）

## T10. ❌ wenyan publish 报 fetch failed（但 curl 直连微信 API 正常）

**错误信息：**
```
fetch failed
Error: Failed to connect to server (https://171.37.44.131)
```

**排查步骤：**

```bash
# Step 1: 验证微信 API 本身是否可达
curl -s --max-time 10 "https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=$WECHAT_APP_ID&secret=$WECHAT_APP_SECRET"
# ✅ 正常返回 {"access_token": "..."}  → 微信 API 没问题

# Step 2: 验证 relay 服务器是否在线
curl -s --max-time 5 http://171.37.44.131:5200
# ❌ 超时/拒绝连接  → relay 服务器宕机
```

**根因：** wenyan-cli 发布流程经过中间 relay 服务器（`171.37.44.131:5200`）。该服务器宕机时，wenyan publish 会报 `fetch failed`，但直接 curl 调用微信 API 仍然正常。

**当前状态：** relay 服务器 `171.37.44.131` 疑似离线（2026-05-26 实测连接超时）。

**临时解法：** 所有素材已准备好，等 relay 服务恢复后一键发布：
```bash
# 素材位置（已裁剪预处理完毕）
/tmp/article-创作手记五.md          # 文章全文（含frontmatter）
/tmp/cover-900x383.png             # 封面图（900×383）
/tmp/illu1-800px.png              # 文内配图1
/tmp/illu2-800px.png              # 文内配图2（无人物）
/tmp/qr-wechat.png                # 二维码
```

**恢复后发布命令：**
```bash
export WECHAT_APP_ID=$(python3 -c "import json; print(json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json'))['appId'])")
export WECHAT_APP_SECRET=$(python3 -c "import json; print(json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json'))['appSecret'])")
wenyan publish -f /tmp/article-创作手记五.md -t phycat -h solarized-light
```

**已知 relay 服务器 IP 历史：**
- `171.36.17.27` — 旧，已失效
- `171.37.44.131` — 当前（2026-05-26 疑似离线）

---

## T11. ❌ Pillow LANCZOS 错误

**错误信息：** `unexpected keyword argument 'LANCZOS'`

**原因：** 低版本 Pillow 不支持 `Image.LANCZOS`（需要 Pillow ≥ 10）。

**解决：** 使用 `Image.BICUBIC` 替代：
```python
img.resize((w, h), Image.BICUBIC)  # 替代 Image.LANCZOS
```