---
name: wechat-toolkit
description: "微信公众号工具包 — 集成文章搜索、下载、洗稿改写、发布四大功能。当用户需要搜索/下载/改写/发布微信公众号文章时使用。"
category: social-media
metadata:
  installed: true
  server_ip: "171.37.44.131"
  credential_location: "/config/.hermes/home/.config/wenyan-md/credential.json"
  wenyan_version: "2.0.8"
  server_ip_history:
    - "171.36.17.27"  # original, now stale
    - "171.37.44.131" # current (2026-05-25)
---

# 📦 微信公众号工具包 (wechat-toolkit)

集成四大功能模块：**搜索 → 下载 → 洗稿 → 发布**，覆盖公众号内容创作全流程。

> **适配说明**：本技能从 OpenClaw 移植到 Hermes Agent。主要变更：
> - `{baseDir}` → `/config/.hermes/skills/social-media/wechat-toolkit/`
> - `wenyan-cli` 是 npm 全局包（非 OpenClaw 专用），已验证可用（v2.0.8）
> - 下载模块需 `puppeteer-core` + Chromium（已验证 `/usr/bin/chromium`）
> - 凭证存储在 `/config/.hermes/home/.config/wenyan-md/credential.json`
> - 当前服务器 IP：`171.36.17.27`（已加入白名单）

---

## 安装笔记

### 依赖安装

```bash
# search 模块：cheerio
cd /config/.hermes/skills/social-media/wechat-toolkit/scripts/search
npm install cheerio

# downloader 模块：puppeteer-core
cd /config/.hermes/skills/social-media/wechat-toolkit/scripts/downloader
npm install
```

### Chromium 配置

下载模块依赖 puppeteer-core + Chromium。调用前设置环境变量：

```bash
export PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium
```

### Node.js

要求 ≥ 18，已验证支持 v22。

---

## 模块一览

| 模块 | 功能 | 状态 | 触发词示例 |
|------|------|------|-----------|
| 🔍 搜索 | 按关键词搜索公众号文章 | ✅ 可用 | "搜XX的公众号文章" |
| 📰 下载 | 下载文章内容/图片/视频 | ✅ 可用 | "下载这篇公众号文章" |
| ✍️ 洗稿 | AI去痕迹+原创改写 | ✅ 可用 | "帮我洗稿/改写这篇文章" |
| 📱 发布 | 发布Markdown到草稿箱 | ✅ 可用 | "发布到公众号" |

---

# 🔍 模块一：文章搜索

通过搜狗微信搜索获取公众号文章列表，支持抓取正文。

## 首次安装依赖

```bash
npm install -g cheerio
```

## 使用方法

```bash
# 基础搜索
node /config/.hermes/skills/social-media/wechat-toolkit/scripts/search/search_wechat.js "关键词"

# 指定数量
node /config/.hermes/skills/social-media/wechat-toolkit/scripts/search/search_wechat.js "关键词" -n 15

# 保存到文件
node /config/.hermes/skills/social-media/wechat-toolkit/scripts/search/search_wechat.js "关键词" -n 20 -o result.json

# 解析真实链接
node /config/.hermes/skills/social-media/wechat-toolkit/scripts/search/search_wechat.js "关键词" -n 5 -r

# 抓取文章正文（自动启用 -r）
node /config/.hermes/skills/social-media/wechat-toolkit/scripts/search/search_wechat.js "关键词" -n 5 -c
```

### 参数说明
- `query`：搜索关键词（必填）
- `-n, --num`：返回数量（默认 10，最大 50）
- `-o, --output`：输出 JSON 文件路径
- `-r, --resolve-url`：解析微信文章真实链接
- `-c, --fetch-content`：抓取文章正文内容（自动启用 -r）

### 输出字段
- 文章标题、地址、概要、发布时间、来源公众号
- `content`：正文内容（使用 -c 时）
- `word_count`：字数统计（使用 -c 时）

---

# 📰 模块二：文章下载

输入公众号文章链接，自动下载内容（Markdown+HTML）、配图和视频。

## 首次安装依赖

```bash
cd /config/.hermes/skills/social-media/wechat-toolkit/scripts/downloader && npm install
```

## 下载前：确认保存位置

```bash
# 查看当前配置
node /config/.hermes/skills/social-media/wechat-toolkit/scripts/downloader/download.js --show-config

# 设置默认下载路径（仅需一次）
node /config/.hermes/skills/social-media/wechat-toolkit/scripts/downloader/download.js --set-output ~/Downloads/wechat-articles
```

- `"isDefault": true` → 尚未配置，需询问用户
- `"isDefault": false` → 已配置，告知用户当前路径

## 下载文章

```bash
# 使用默认路径
node /config/.hermes/skills/social-media/wechat-toolkit/scripts/downloader/download.js "<文章URL>"

# 临时指定路径
node /config/.hermes/skills/social-media/wechat-toolkit/scripts/downloader/download.js "<文章URL>" --output <临时目录>

# 跳过图片/视频
node /config/.hermes/skills/social-media/wechat-toolkit/scripts/downloader/download.js "<文章URL>" --no-image
node /config/.hermes/skills/social-media/wechat-toolkit/scripts/downloader/download.js "<文章URL>" --no-video
```

### 输出结构
```
<下载目录>/<文章标题>/
├── content/article.html      # 完整 HTML
├── metadata.json              # 标题、作者、时间等
├── images/                    # 所有配图
└── videos/                    # 所有视频/音频
```

### 前置要求
- Node.js ≥ 18（✅ 已验证：v22）
- Google Chrome / Chromium（✅ 已验证：`/usr/bin/chromium`）
- `npm install`（首次运行前执行）

### ⚠️ Chromium 环境变量

如遇到 Chrome 路径问题，可设置：
```bash
export PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium
```

---

# ✍️ 模块三：文章洗稿与改写

将文章改写为自然、原创的风格，去除 AI 写作痕迹，提升原创度。

## 触发词
- "帮我洗稿这篇文章"
- "改写成原创"
- "降低查重率"
- "去掉 AI 味"

## 洗稿工作流

### 标准洗稿流程

1. **获取原文** — 通过搜索（-c 抓正文）或下载获得原文
2. **分析结构** — 识别文章类型、核心论点、段落层次
3. **深度改写** — 按以下策略执行改写
4. **添加 frontmatter** — 补充 title + cover
5. **保存** — 推送到本地目录

### 改写策略

#### A. 结构重组（降重核心）
- **段落重排**：调整段落顺序，打乱原文结构
- **段落拆合**：将长段拆为短段，或合并碎片段落
- **叙事角度转换**：时间线 ↔ 问题导向 ↔ 对比分析 ↔ 故事引入
- **论据重组**：保留核心论据，改变展开方式

#### B. 语言改写（去 AI 痕迹）
- **删除意义膨胀句**："标志性"、"里程碑"、"深远影响" → 替换为具体事实
- **去虚假权威**："专家认为"、"业内普遍认为" → 写明来源或删除
- **去伪深度动词**："提升能力"、"赋能"、"推动进程" → 改为具体动作
- **去广告语气**："卓越"、"极致体验"、"全方位" → 客观描述
- **去 AI 高频词**：赋能、闭环、生态、抓手、底层逻辑、范式、沉淀、势能
- **去填充短语**：事实上、值得注意的是、总体来说、不难发现
- **去空洞结尾**："未来可期"、"值得期待" → 实际结论或行动项

#### C. 标题改写
为每篇文章生成 3-5 个备选标题，涵盖：
- **疑问型**：用问题引发好奇（"为什么XX还在用这种方法？"）
- **数字型**：用数字增强可信度（"3个被忽略的XX技巧"）
- **悬念型**：制造信息差（"XX的真相，90%的人不知道"）
- **痛点型**：戳中读者痛点（"别再XX了，试试这个方法"）

#### D. 开头改写
将原文开头转换为以下风格之一：
- **故事引入**：用一个小故事或场景切入
- **数据引入**：用震撼数据开头
- **痛点引入**：直击读者困扰
- **反问引入**：抛出反直觉问题

#### E. SEO 优化（可选）
- 在标题和首段自然植入核心关键词
- 在小标题中分布长尾关键词
- 控制关键词密度（2%-5%），保持自然可读

### AI 痕迹识别清单

改写完成后，逐项检查：

| # | 检查项 | 处理方式 |
|---|--------|---------|
| 1 | 意义膨胀句 | 替换为具体事实 |
| 2 | 虚假权威引用 | 写明来源或删除 |
| 3 | 伪深度动词 | 改为具体动作 |
| 4 | 广告语气 | 客观描述 |
| 5 | 模板段落（挑战→机遇→展望） | 删除模板，保留结论 |
| 6 | AI 高频词密集出现 | 替换为日常用语 |
| 7 | 负向并列滥用（不仅…而且…） | 直接表达 |
| 8 | 三段式强拆 | 保留重点，删填充项 |
| 9 | 同义词机械轮换 | 同一概念固定用词 |
| 10 | 破折号滥用 | 改为句号或逗号 |
| 11 | 加粗强调滥用 | 去掉不必要强调 |
| 12 | 列表模板（**X：**…） | 合并为自然段 |
| 13 | 概念堆砌标题 | 改为口语化标题 |
| 14 | Emoji 泛滥 | 除非指定风格，默认删除 |
| 15 | 聊天语残留 | 删除 |
| 16 | 知识截止声明 | 删除 |
| 17 | 过度讨好语气 | 客观回应 |
| 18 | 填充短语 | 删除 |
| 19 | 过度模糊（可能会、或许） | 改为条件判断 |
| 20 | 空洞结尾 | 改为实际结论 |
| 21 | 假区间表达（从…到…） | 列举具体事实 |

### 输出格式

1. **改写后全文**（Markdown 格式，含 frontmatter）
2. **备选标题**（3-5 个）
3. **修改说明**（可选，简要列出主要改动）

### 判断标准

改写成功的标志：
- ✅ 读起来像真人写的，能直接朗读不拗口
- ✅ 没有空洞句、模板段、"像 AI" 的气味
- ✅ 信息密度高，每句话都有具体内容
- ✅ 结构与原文明显不同
- ✅ 保持原文核心信息完整性

---

# 📱 模块四：文章发布

> ✅ **状态**：`wenyan-cli` 已安装（v2.0.8），凭证已配置，可正常发布。

## 凭证存储位置

`/config/.hermes/home/.config/wenyan-md/credential.json`

## 配置凭证

### 方式一：交互式设置（推荐）

```bash
wenyan credential --set
# 按提示输入 AppID 和 AppSecret
```

### 方式二：通过环境变量

```bash
# 方式A：直接设置环境变量（当前会话有效）
export WECHAT_APP_ID=your_app_id
export WECHAT_APP_SECRET=your_app_secret

# 方式B：写入 .env 文件（持久化）
echo 'WECHAT_APP_ID=your_app_id' >> ~/.env
echo 'WECHAT_APP_SECRET=your_app_secret' >> ~/.env

# 运行时指定 env 文件
wenyan publish -f article.md --env-file ~/.env
```

### 方式三：通过 --app-id 参数

```bash
wenyan publish -f article.md --app-id your_app_id
# 会提示输入 AppSecret
```

## 发布文章

### 先预览再发布（推荐）

```bash
# 预览 HTML（不发布，只转换，测试用）
wenyan render -f /path/to/article.md -t phycat -h solarized-light
```

### 发布到草稿箱

```bash
# 标准发布（AIPY 默认使用 phycat 绿色主题）
wenyan publish -f /path/to/article.md -t phycat -h solarized-light

# 使用其他主题
wenyan publish -f /path/to/article.md -t lapis -h solarized-light
```

> ⚠️ **AIPY 发布默认 phycat（绿色主题）** — 不要用 lapis，除非用户明确指定。
>
> ⚠️ **先 render 再 publish** — render 只生成 HTML 不发稿，可反复调试样式，确认后再 publish。
>
> ⚠️ **摘要设置**：wenyan-cli 无 `--description` 参数。**解法**：在 frontmatter 中使用 `description` 字段，生成规则见下方专节。

## 文内配图插入时机与位置

wenyan-cli 不支持指定位置插入图片，**解法**：在 Markdown 中用 `![描述](本地路径)` 在对应章节位置直接嵌入。

**典型位置分配原则**：
- 封面图（cover）：frontmatter `cover:` 字段指定
- 文内图：插入到对应章节的**段落之后、章节标题后的第一段结尾**
- 文末二维码：正文最后、签名/版权信息之前

**图片尺寸处理**：
- 文内配图建议宽度：800px（保持比例）
- 使用 Pillow 缩放：`img.resize((800, int(800*h/w)), LANCZOS).save('output.png')`

## 完整发布流程（飞书文档 → 微信公众号）

**第一步：下载所有素材**
```bash
# 飞书文件下载（lark-cli）
cd /tmp && node_modules/.bin/lark-cli drive +download \
  --file-token <FILE_TOKEN> --output=./output.png --overwrite

# 安装 lark-cli（如未安装）
cd /tmp && npm install @larksuite/cli
```

**第二步：图片预处理**
```bash
# 封面图裁剪（2.35:1）
uv run --with pillow python3 -c "
from PIL import Image
img = Image.open('/tmp/cover.png')
tw, th = 900, 383
r = img.size[0]/img.size[1]
tr = tw/th
if r > tr:
    new_w = int(img.size[1] * tr)
    c = img.crop(((img.size[0]-new_w)//2, 0, (img.size[0]+new_w)//2, img.size[1]))
else:
    new_h = int(img.size[0] / tr)
    c = img.crop((0, (img.size[1]-new_h)//2, img.size[0], (img.size[1]+new_h)//2))
c.resize((tw, th), Image.LANCZOS).save('/tmp/cover-wechat.png', 'PNG')
"

# 文内图缩放（800px宽）
uv run --with pillow python3 -c "
from PIL import Image
img = Image.open('/tmp/image.png')
w = min(800, img.size[0])
r = w / img.size[0]
img.resize((w, int(img.size[1]*r)), Image.LANCZOS).save('/tmp/image-w800.png', 'PNG')
"
```

**第三步：组装 Markdown**
- frontmatter：`title`（必填）、`cover`（封面图本地路径）、`description`（摘要，发布后手动填入后台）
- 内容区顶部：一般不放重复大标题（公众号会自动显示标题）
- 文内配图：用 `![描述](本地路径)` 嵌入
- 文末：二维码 + 课程推荐语

**第四步：预览 + 发布**
```bash
# 先预览
wenyan render -f /tmp/article.md -t phycat -h solarized-light

# 确认无误后发布
export WECHAT_APP_ID=$(python3 -c "import json; print(json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json')).get('appId',''))")
export WECHAT_APP_SECRET=$(python3 -c "import json; print(json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json')).get('appSecret',''))")
wenyan publish -f /tmp/article.md -t phycat -h solarized-light
```

**第五步：微信后台确认**
- 登录 mp.we.weixin.qq.com
- 内容与标题 → 确认摘要是否需要手动粘贴（wenyan 不传摘要字段）
- 确认封面图、文内图、二维码显示正确
- 手动发布

## 主题选项

| 主题 | 说明 |
|------|------|
| `default` | 内置默认主题 |
| `lapis` | 青金石主题（蓝色调，推荐技术文章） |
| `phycat` | 薄荷绿主题（mint-green，清晰层次分明） |
| `orangeheart` | 橙色调主题 |
| `purple` | 紫色调主题 |
| `rainbow` | 彩色主题 |
| `maize` | 浅黄主题 |
| `pie` | sspai风格，现代锐利 |

## 封面图规格与裁剪

**公众号封面图要求**：
- 尺寸：900×383 像素（2.35:1 宽屏比例）
- 格式：PNG/JPG，不超过 5MB
- 路径：使用本地绝对路径

**封面图裁剪脚本**（使用 Pillow）：

```bash
uv run --with pillow python3 -c "
from PIL import Image
img = Image.open('/path/to/source.png')
target_w, target_h = 900, 383
current_ratio = img.size[0]/img.size[1]
target_ratio = target_w/target_h
if current_ratio > target_ratio:
    new_w = int(img.size[1] * target_ratio)
    left = (img.size[0] - new_w) // 2
    cropped = img.crop((left, 0, left + new_w, img.size[1]))
else:
    new_h = int(img.size[0] / target_ratio)
    top = (img.size[1] - new_h) // 2
    cropped = img.crop((0, top, img.size[0], top + new_h))
cropped = cropped.resize((target_w, target_h), Image.LANCZOS)
cropped.save('/path/to/cover-wechat.png', 'PNG')
print(f'Saved: {cropped.size}')
"
```

## 发布命令环境变量注入

从凭证文件读取并注入环境变量（无需手动设置）：

```bash
export WECHAT_APP_ID=$(python3 -c "import json; print(json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json')).get('appId',''))")
export WECHAT_APP_SECRET=$(python3 -c "import json; print(json.load(open('/config/.hermes/home/.config/wenyan-md/credential.json')).get('appSecret',''))")
wenyan publish -f /path/to/article.md -t lapis -h solarized-light
```

## 发布成功判断

成功输出示例：
```
发布成功，Media ID: x8t4d4GktbO7KNjgj7SgKeqzKopmssWUpnoKiTdBpMR2ulZqwk4h6Uh4h1kEeUTD
```

**注意事项**：
- wenyan 只发布到**草稿箱**，不直接发布
- 需登录微信公众号后台确认并手动发布
- frontmatter 必须包含 `title` 字段（`cover` 字段可选）

## ⚠️ 重要前提

**IP 白名单**：微信公众号后台必须将当前服务器 IP 添加到白名单。

当前服务器 IP：`171.37.44.131`（2026-05-25 实测）

> 💡 **IP 可能变化**：如果发布时报 `invalid ip not in whitelist`，立即用 `curl -s ifconfig.me` 获取当前 IP，更新微信后台白名单。旧 IP `171.36.17.27` 已失效。

添加路径：微信公众号后台 → 设置与开发 → 安全中心 → IP白名单 → 添加当前 IP

> 💡 如需确认当前 IP：`curl -s ifconfig.me` 或 `curl -s api.ipify.org`

---

## 📝 摘要生成规则（AIPY 内容体系）

**核心原则：从读者感兴趣的钩子出发，而非从文章内容概括。**

读者刷到公众号文章时，停留时间只有3秒。摘要的任务是——让人停下来想点进来。

### 好的摘要公式

- **痛点共鸣** + **好奇缺口** + **身份认同**（可选）

### 具体做法

1. 识别文章中最能引起目标读者共鸣的一个痛点或惊讶时刻
2. 用一句话概括这个" hook "，不超过120字
3. 避免空泛的"本文介绍了..."、"本课程..."
4. 优先选择：反常识结论、真实失败经历、能解决什么问题的承诺

### 示例对比

❌ 弱摘要（从内容概括）：
> "本文介绍了AI编程中提示词工程的重要性..."

✅ 强摘要（从读者钩子出发）：
> "学Python三个月，背熟了变量和循环，但让AI写代码时还是只会说'帮我写个计算器'——问题出在哪？"

❌ 弱摘要：
> "第二课不教语法，教你怎么和AI说话..."

✅ 强摘要：
> "第二课不教Hello World，教你怎么让AI写出你想的代码。一个温度转换程序，三次失败到成功的对比，告诉你提问质量的差距才是关键。"

---

# ⚠️ wenyan publish fetch failed 故障排查

**症状**：`wenyan publish` 报 `fetch failed`，但直接 `curl api.weixin.qq.com` 返回 200 成功。

**根因**：`wenyan-cli v2.0.8` 内部使用 `undici` 的 `setGlobalDispatcher(ProxyAgent)`，无法禁用。清除系统代理环境变量或设置 `--proxy ""` 均**无效**。

**关键发现**：
- `wenyan publish` 无法绕过（没有 `--proxy` 参数可以禁用 ProxyAgent）
- `wenyan publish --server http://localhost:3050` 可以路由到本地 server，但 AppID 传不过去
- Python urllib 直连微信 API 完全正常（token 获取、图片上传均成功）

**解法**：

### 方案A（推荐）：Python 脚本直接调用微信 API

详见：`references/wechat-direct-publish.md`

覆盖：获取 token → 上传图片到永久素材 → 上传临时 thumb → 替换飞书图片 URL → 创建草稿

### 方案B：手动发布

登录 mp.weixin.qq.com → 草稿箱 → 新建草稿。图片已通过方案A上传到微信 CDN，手动粘贴正文即可。

### 方案C：修复 wenyan-cli（未解决）

`unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY` → 仍 fetch failed。ProxyAgent 在 CLI 启动时已初始化，系统代理变量无法清除。

---

**详细诊断记录**：`references/wenyan-proxy-diagnosis.md`

---

# 完整工作流示例

## 飞书文档 → 微信公众号发布

详见：`references/feishu-wechat-workflow.md`

覆盖场景：飞书文档 + 飞书文件（封面图）→ 微信公众号草稿箱，含封面图裁剪规格。

## 搜索 → 洗稿 → 保存

```
1. 搜索文章：
   node /config/.hermes/skills/social-media/wechat-toolkit/scripts/search/search_wechat.js "AI教程" -n 5 -c

2. 选择目标文章，执行洗稿改写

3. 保存为 Markdown（含 frontmatter）：
   /config/Desktop/wechat-articles/{标题}/article.md
```

## 下载 → 洗稿 → 保存

```
1. 下载文章：
   node /config/.hermes/skills/social-media/wechat-toolkit/scripts/downloader/download.js "https://mp.weixin.qq.com/s/xxx"

2. 读取下载的 HTML/Markdown，执行洗稿改写

3. 保存为 Markdown（含 frontmatter）
```

---

## 注意事项

- 所有工具仅供个人学习使用，请遵守版权法规
- 搜索功能内置防封禁机制（随机UA、请求延迟），请勿高频使用
- 配置文件：下载器 `/config/.hermes/skills/social-media/wechat-toolkit/scripts/downloader/config.json`
