# 部署前用户准备手册

本文档用于说明：**在 AI 帮你完成部署之前，你需要提前准备什么**。

部署动作（安装依赖、生成配置、创建服务、接入访问、初始化验证）由 AI 完成；你只需要准备账号、凭据、基础环境和少量素材，并亲自完成平台登录与最终发布确认。

---

## 1. 你需要准备的核心内容一览

| 类别 | 是否必须 | 说明 |
|---|---|---|
| 一台自准备的电脑/服务器 | 必须 | Windows / macOS / Linux 均可；AI 会根据系统完成部署 |
| Node.js ≥ 22.5 或允许 AI 安装 | 必须 | 项目使用内置 `node:sqlite` |
| Google Chrome 或允许 AI 安装 | 必须 | 专用浏览器登录态与发布自动化依赖 Chrome |
| 国内四家创作者账号 | 建议 | B站、抖音、小红书、视频号；用于登录、体检、发布 |
| 飞书群与自定义机器人 | 强烈建议 | 用于通知、验证码登录、日报、复盘 |
| `.env` 配置文件 | 如用飞书则必须 | 提供 `FEISHU_WEBHOOK` 与 `FEISHU_KEYWORD` |
| 发布工程素材 | 使用分发功能时必须 | 成片、封面、发布素材 md |
| 自有站点 / 飞书多维表格 | 可选 | 仅 L3 转化分析或看板同步需要 |

---

## 2. 基础环境准备

### 2.0 两个必须最先完成的下载

请按以下顺序准备，不要颠倒：

1. **先安装 Google Chrome 浏览器**  
   官方下载：<https://www.google.com/chrome/>  
   这是所有操作的前提。麦田依赖专用 Chrome profile、Chrome 扩展和调试端口，不能用 Edge、Firefox 或系统自带浏览器替代。

2. **再安装“麦田发布助手”Chrome 扩展**  
   Chrome 应用商店：<https://chromewebstore.google.com/detail/%E9%BA%A6%E7%94%B0%E5%8F%91%E5%B8%83%E5%8A%A9%E6%89%8B/eaaipbilkhibicihiollecmbpneikpnj?hl=zh-CN&utm_source=ext_sidebar>  
   扩展必须在登录 B站、抖音、小红书、视频号之前安装到**专用 Chrome profile** 中。否则平台体检和发布流程会失败。

正确顺序：

```text
安装 Chrome → 打开专用 Chrome → 安装麦田发布助手扩展 → 登录各平台账号 → 体检 → 发布
```

### 2.1 电脑/服务器

你可以准备以下任意一种环境：

- Windows 10/11
- macOS
- Linux 服务器/虚拟机/容器

建议：

- 至少 20 GB 可用磁盘空间；如果要存放大量成片，建议单独准备数据盘或挂载目录。
- 网络能访问：
  - Chrome Web Store（安装“麦田发布助手”扩展）
  - 各平台创作者后台
  - 飞书 webhook 地址
  - Node.js / 系统包下载源

### 2.2 软件环境

以下项目可由 AI 安装或确认，你只需要知道用途：

| 软件 | 用途 | 版本要求 |
|---|---|---|
| Node.js | 运行麦田主程序 | ≥ 22.5 |
| Google Chrome | 专用浏览器、扩展、平台登录态 | 稳定版 |
| Xvfb / 虚拟显示 | Linux 无图形界面时提供虚拟桌面 | Linux 需要 |
| VNC / noVNC | 远程查看并操作专用 Chrome | Linux 无图形界面时推荐 |
| ffmpeg / ffprobe | 视频时长读取、转码、faststart 处理 | 建议安装 |
| nginx | 如需通过域名访问 Web 或 VNC | 可选 |

> 如果你是 Windows / macOS，一般不需要 Xvfb / noVNC，本地可以直接看到 Chrome 窗口。

---

## 3. 平台账号准备

本项目当前推荐先启用国内四家平台：

| 平台 | 账号要求 | 你需要做什么 |
|---|---|---|
| B站 | 可投稿的创作者账号 | 在专用 Chrome 中登录 B站创作者后台 |
| 抖音 | 抖音创作者账号 | 在专用 Chrome 中登录抖音创作者中心 |
| 小红书 | 小红书创作者/专业号 | 在专用 Chrome 中登录小红书创作服务平台 |
| 视频号 | 有视频号创作者权限的微信账号 | 在专用 Chrome 中用微信扫码登录视频号助手 |

注意事项：

1. **登录任何平台前，必须先在专用 Chrome 中安装“麦田发布助手”扩展**（见 2.0 节）。
2. 账号必须有**创作者后台和视频发布权限**。
3. 四个平台的最终“发布”按钮需要你人工点击，AI 不代为点击发布。
4. 登录态保存在专用 Chrome profile 中，不要清理该 profile。
5. 不要在多个 Chrome profile 里混用同一套自动化登录态，避免混乱。
6. 平台登录通常需要你本人扫码或短信验证，AI 不能代替完成。

---

## 4. 飞书准备

飞书用于：

- 发布开始/完成/异常通知
- 等待人工发布提醒
- 每日数据日报
- 1/3/7 天复盘和周报
- 网页验证码登录

### 4.1 创建飞书自定义机器人

请参考飞书官方文档：

<https://open.feishu.cn/document/client-docs/bot-v3/add-custom-bot>

你需要：

1. 选择一个飞书群。
2. 添加“自定义机器人”。
3. 设置机器人名称，例如：`麦田发布助手`。
4. 设置安全策略。
   - 推荐使用“自定义关键词”。
   - 关键词请记下来，例如：`自媒体`。
5. 复制机器人的 webhook 地址，格式类似：

```text
https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### 4.2 准备 `.env`

请准备一个 `.env` 文件，并告知 AI 它的路径。文件中至少包含：

```bash
FEISHU_WEBHOOK=https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
FEISHU_KEYWORD=自媒体
```

说明：

- `FEISHU_WEBHOOK`：飞书自定义机器人地址。
- `FEISHU_KEYWORD`：机器人安全设置里的关键词。
- 如果机器人没有启用关键词限制，`FEISHU_KEYWORD` 可以留空。
- AI 会读取这两个值并写入项目的 `config.json`，但你无需把真实值发到聊天里。
- 请确保 `.env` 权限为仅当前用户可读，例如 Linux/macOS：

```bash
chmod 600 /path/to/.env
```

Windows 用户请确保 `.env` 不被同步到公开仓库或分享给他人。

---

## 5. 发布素材准备

如果你要使用多平台分发功能，需要为每个视频准备一个发布工程目录：

```text
某系列/第01期/
├── 成片.mp4
├── 发布素材.md
├── 封面/
│   ├── cover-3x4.png
│   ├── cover-4x3.png
│   └── cover-16x9.jpg
└── publish.json        # 可由 AI / 发布 skill 生成
```

### 5.1 视频要求

- 必须是 MP4。
- 必须使用 `faststart`，即 `moov` atom 在文件头。
- 建议导出命令：

```bash
ffmpeg -i input.mp4 -c copy -movflags +faststart 成片.mp4
```

### 5.2 封面要求

建议准备：

- 竖版 3:4：小红书、抖音、视频号主页
- 横版 4:3：B站、抖音、视频号分享卡
- 横版 16:9：YouTube / 备用

### 5.3 发布素材 md

发布素材 md 需要包含各平台标题、正文、标签等内容。AI 或发布 skill 会根据它生成 `publish.json`。

国内平台注意：

- 小红书、抖音、视频号不要放外链和导流话术。
- 各平台标签第一个建议放账号名。
- 定时发布请写清楚时间。

---

## 6. 功能级别选择

请提前告诉 AI 你要启用哪个级别：

| 级别 | 说明 | 你需要额外准备 |
|---|---|---|
| L1 只看数据 | 登录平台、采集播放/粉丝数据、看大屏和飞书日报 | 四家平台账号 |
| L2 自动分发 | 生成发布工程、填写平台上传页、定时、人工点发布、取链接 | 发布素材、成片、封面 |
| L3 转化分析 | 看注册、会员、订单、收入 | 自有站点 API、UTM 埋点 |

当前建议：先做 L1 + L2，国内四家平台，暂不启用海外平台和自有站点。

---

## 7. 可选准备

### 7.1 自有站点

如果要做 L3 转化分析，需要准备：

- 站点地址
- 管理后台路径
- 课程列表 API
- 数据汇总 API
- 课程详情页路径规则
- 站点管理后台登录态

接口格式参考项目中的 `src/site.mjs`。

### 7.2 飞书多维表格

如果要同步看板，需要准备：

- 多维表格地址
- base token
- 表格 ID
- `lark-cli` 安装方式

如没有这些，不影响 L1/L2 主流程。

### 7.3 远程访问

如果你希望通过域名访问：

- 准备一个可解析到部署机器的域名；
- 或准备已有反向代理/网关；
- 告诉 AI 域名与访问方式即可。

远程访问不是发布流程的必需项。

---

## 8. 隐私与安全提醒

以下信息都属于敏感资产，不要提交到公开仓库：

- `.env`
- `config.json`
- `data/`
- `chrome-profile/`
- 飞书 webhook
- 网页访问口令 `web.key`
- 小组件口令 `widgetToken`
- 平台登录态
- 发布工程中的成片和封面

建议：

- `.env` 只放本机，权限 600。
- 不要把 webhook 或 token 粘贴到聊天、Issue、截图里。
- 平台登录后不要清理专用 Chrome profile。
- 定期备份 `config.json`、`data/`、`chrome-profile/` 和发布工程目录。

---

## 9. 你需要交给 AI 的最小清单

最简情况下，你只需要准备并告诉 AI：

```text
1. 部署机器系统：Windows / macOS / Linux
2. Google Chrome 安装地址：<https://www.google.com/chrome/>
3. 麦田发布助手扩展：<https://chromewebstore.google.com/detail/%E9%BA%A6%E7%94%B0%E5%8F%91%E5%B8%83%E5%8A%A9%E6%89%8B/eaaipbilkhibicihiollecmbpneikpnj?hl=zh-CN&utm_source=ext_sidebar>
4. 要启用的平台：例如 bilibili,douyin,xhs,channels
5. 功能级别：L1 / L2 / L3
6. .env 路径：其中包含 FEISHU_WEBHOOK 和 FEISHU_KEYWORD
7. 是否需要域名访问：需要 / 不需要
8. 发布工程目录：例如 /path/to/projects
```

之后 AI 会负责：

- 安装依赖
- 生成配置
- 写入飞书 webhook 与关键词
- 初始化专用 Chrome
- 配置访问控制
- 启动常驻服务
- 执行基础验收

你只需要亲自完成：

- 安装 Google Chrome（如未安装）
- 在专用 Chrome 中安装并启用“麦田发布助手”扩展
- 平台扫码登录
- 国内四家发布按钮的最终点击
