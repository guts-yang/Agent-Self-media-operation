# 麦田（Agent-Self-media-operation）Linux 全功能部署方案

- 适用仓库：`https://github.com/guts-yang/Agent-Self-media-operation.git`
- 本地克隆：`/data/workspace/Agent-Self-media-operation`
- 代码版本：`main` @ `3e3402c`（`docs: README 顶部加官网入口 (#41)`）
- 目标环境：Linux 服务器，覆盖 L1 数据监控、L2 多平台自动分发、L3 转化分析全部功能
- 文档性质：部署 runbook 与配置清单。本文不包含任何真实密钥，所有敏感值均用占位符表示。

---

## 0. 结论摘要

1. **项目本质**：Node.js 编写的个人自媒体中控。通过专用 Chrome profile + Chrome 扩展“麦田发布助手”完成多平台上传页填写、定时、取链接、数据采集；通过 SQLite 持久化；通过本地 Web 大屏和飞书卡片展示状态。
2. **无需 `npm install`**：`package.json` 没有 `dependencies` / `devDependencies` / `scripts`。运行时依赖是 Node.js ≥ 22.5（内置 `node:sqlite`）、Google Chrome，以及可选的 `ffmpeg` / `ffprobe` / `lark-cli` / `cloudflared`。
3. **配置先行**：除 `demo.mjs` 外，所有命令在模块加载阶段就要求仓库根目录存在 `config.json`；缺失时 `src/feishu.mjs` 会直接退出。
4. **进程模型**：`node src/hub.mjs daemon` 会在**同一进程内**启动 Web 服务（默认 `0.0.0.0:8787`），不需要单独再起 web；`node src/hub.mjs web` 是独立调试模式。演示服务 `node src/demo.mjs` 使用 `8788`。
5. **Linux 不是开箱即用**：源码硬编码了 macOS 的 Chrome 路径、`/usr/bin/sips`、`/Volumes`、`launchd`、`osascript`、`lsof`、Homebrew 版 `cloudflared`。正式部署前需要做本文第 3 节的适配。
6. **Chrome 是核心依赖**：项目需要“真实浏览器 + 人工登录态 + CDP 9333”，并非纯无头爬虫。服务器上建议使用 Xvfb + VNC/noVNC 提供图形会话；CDP 必须只监听回环地址。
7. **安全审计**：未发现发布 skill 自动下载执行远程脚本的投毒行为；但安装脚本、Windows 安装器、iPhone 小组件加载器存在需要知悉的风险点，详见第 12 节。
8. **当前工作区尚未具备运行条件**：核对时 `/data/workspace` 环境没有 `node` 命令，仓库自带测试未能在此处执行；正式部署前必须先安装 Node.js ≥ 22.5，再按第 9.1 节运行测试。

---

## 1. 当前功能实现梳理

### 1.1 模块关系

| 模块 | 职责 | 关键依赖/被依赖 |
|---|---|---|
| `src/hub.mjs` | CLI 主入口；登录、体检、计划、发布、同步、采集、复盘、周报、看板、守护循环 | 调用 `db/metrics/board/feishu/followers/driver/site/learn/validate/plugin/checks`；daemon 内 `import('./web.mjs')` |
| `src/web.mjs` | Web 大屏、配置管理 API、小组件 API、飞书会话、Cloudflare Access 验签 | 被 daemon 同进程启动；也通过子进程调用 hub 命令 |
| `src/demo.mjs` | 假数据演示，写 `demo/config.json` + `demo/hub.db`，端口 `8788` | 不碰真实配置 |
| `src/init.mjs` | 交互式安装向导：查环境、生成 `config.json`、装 launchd、打开登录页 | Linux 下需跳过其 launchd/Chrome 检测逻辑 |
| `src/db.mjs` | `node:sqlite` 封装，建表、WAL、状态常量 | 几乎所有模块使用；DB 默认 `data/hub.db` |
| `src/chrome-shim.mjs` | 模拟 `chrome.tabs` / `chrome.debugger`，启动专用 Chrome，连接 CDP WebSocket | `driver.mjs`、`plugin.mjs` 使用；Linux 适配重点 |
| `src/driver.mjs` | 驱动选择：`direct`（加载插件 lib）或 `plugin`（8765 开发桥） | 默认 `direct` |
| `src/plugin.mjs` | 定位插件：`store`（Chrome 扩展目录）或 `local`（`pluginDir`） | 默认 `store`，扩展 ID `eaaipbilkhibicihiollecmbpneikpnj` |
| `src/site.mjs` | 自有站点接口适配与 UTM 归因 | L3 需要 |
| `src/feishu.mjs` | 加载配置、生成飞书卡片、发送通知 | 全局配置源；无 webhook 时通知静默跳过 |
| `src/metrics.mjs` / `src/followers.mjs` | 播放/互动数据与粉丝采集 | L1 核心 |
| `src/review.mjs` | 1/3/7 天复盘、周报调度 | daemon 自动调用 |
| `src/board.mjs` | 飞书多维表格同步 | 依赖外部 `lark-cli` |
| `src/checks.mjs` | 平台侧查重、提交后回读 | X / YouTube / Instagram / 抖音 / 知识站 |
| `src/validate.mjs` | 发布工程校验，含 MP4 faststart 检查 | `hub run` 前调用 |
| `src/make-publish.mjs` | 从发布素材 Markdown 生成 `publish.json` | 发布 skill 调用 |
| `src/access.mjs` | Cloudflare Access JWT 验签 | Web 鉴权链 |
| `src/login.mjs` | 飞书验证码登录，会话写 `data/sessions.json` | Web 鉴权链 |
| `src/tunnel.mjs` | cloudflared 隧道，结果写 `data/tunnel.json` | 可选，systemd 单独托管 |
| `src/config-admin.mjs` | Web UI 改飞书/站点/平台开关并落盘 | 写前生成 `config.json.bak` |

### 1.2 命令清单

平台标识统一为：`bilibili` / `xhs` / `douyin` / `channels` / `youtube` / `x` / `instagram` / `zsz`（知识站）。

| 命令 | 用途 | 建议时机 |
|---|---|---|
| `node src/hub.mjs login [--only a,b]` | 打开专用 Chrome 的平台登录页/插件商店页 | 初始化、登录态过期 |
| `node src/hub.mjs doctor [--only a,b] [--quiet] [--notify]` | 体检登录态与上传页，不填写不上传 | 登录后、发布前、daemon 定期 |
| `node src/hub.mjs notify-test` | 发送飞书测试卡片 | 配置 webhook 后 |
| `node src/hub.mjs adopt <目录>` | 登记已发布内容，不上传 | L1 初始化 |
| `node src/hub.mjs plan [目录]` | 登记分发计划 | L2 发布前 |
| `node src/hub.mjs run <目录> [--only x] [--force] [--now]` | 校验、分批填写、自动/人工提交 | L2 核心 |
| `node src/hub.mjs submit <id> <youtube\|x\|instagram>` | 已填好但中控卡住时单独提交一次 | 异常补救 |
| `node src/hub.mjs sync <id> [--only a,b]` | 取链接、写回记录、校验定时偏差 | 发布后 |
| `node src/hub.mjs status [id] [--logs]` | 查看期/平台状态与日志 | 日常排查 |
| `node src/hub.mjs mark <id> <平台> <状态> [备注] [--url]` | 人工修改状态 | 异常补救 |
| `node src/hub.mjs metrics [id] [--skip xhs]` | 采集流量并生成日报 | daemon 每日 / 手动 |
| `node src/hub.mjs followers [--only] [--skip]` | 单独采集粉丝 | daemon / 手动 |
| `node src/hub.mjs review <id> [天数] [--notify]` | 生成单期复盘 | 第 1/3/7 天 |
| `node src/hub.mjs weekly [--notify]` | 生成上周周报 | 每周一 |
| `node src/hub.mjs board` | 同步飞书多维表格 | 按需 |
| `node src/hub.mjs daemon` | 常驻：web + NAS 检查 + sync + 提醒 + 采样 + 体检 + 复盘 | 生产常驻 |
| `node src/hub.mjs web` | 单独启动 Web（调试） | 一般不用，避免与 daemon 抢 8787 |
| `node src/validate.mjs <目录>` | 独立校验发布工程 | 发布前 |
| `node src/make-publish.mjs <素材.md> <目录> [--require 账号名]` | 生成 `publish.json` | 发布工程准备 |

### 1.3 功能级别与依赖

| 级别 | 依赖配置 | 依赖命令 | 外部条件 | 主要风险 |
|---|---|---|---|---|
| L1 看数据 | `config.json`、`web.key`、可选飞书 webhook、Chrome profile、`rootPath` | `login → doctor → adopt → metrics/followers → daemon` | 专用 Chrome 登录各平台；Xvfb/VNC；NAS 可读 | 页面改版、登录过期 |
| L2 分发 | `plugin: store`、扩展、`batches/auto/manual`、发布工程 | `plan → run → sync → submit` | Chrome Web Store 可访问；人工点国内平台发布 | 风控、重复发布、定时被改、上传超时 |
| L3 转化 | `site.*` + UTM 埋点，可选 `board.*` | `board`、站点 API、review 转化段 | 自有站点按 `src/site.mjs` 接口改造；`lark-cli` | 站点接口变更、管理员登录过期 |

---

## 2. 源码核对的关键事实

- `package.json`：无 npm 依赖；`engines.node >= 22.5`；`bin.maitian -> src/hub.mjs`。
- `src/db.mjs:2` 使用 `import { DatabaseSync } from 'node:sqlite'`；DB 路径可用环境变量 `MAITIAN_DB` 覆盖。
- `src/feishu.mjs:3-5` 在 import 阶段读取 `MAITIAN_CONFIG || <ROOT>/config.json`；**无配置则直接退出**。
- `src/hub.mjs:367` daemon 内 `import('./web.mjs')` 并 `startWeb()`；`src/web.mjs:261` 监听 `W.port, '0.0.0.0'`。
- `src/demo.mjs` 写 `demo/config.json`（`web.port=8788, key=demo`）和 `demo/hub.db`。
- `config-admin.mjs` 的 Web UI 只热更新飞书字段、站点字段、平台开关；**手工修改 `config.json` 的大部分字段需要重启 daemon**。
- SQLite 表：`episodes`、`tasks`、`metrics`、`board_records`、`site_daily`、`snapshots`、`site_campaigns`、`samples`、`site_channels`、`meta`、`logs`，另有 `followers`。迁移仅为 `CREATE TABLE IF NOT EXISTS`，无版本号。
- 防重状态：`filled`、`awaiting_user`、`scheduled`、`published` 被视为锁定，见 `src/db.mjs:27`。
- Web 鉴权链：`mt_key` cookie → 飞书会话 → Cloudflare Access JWT → `?key=` 换 cookie → 401；`/api/widget*` 使用 `widgetToken` 且用 `timingSafeEqual` 比较。
- 高权限 API：`/api/integrations*`、`/api/platform-toggle`、`/api/action`（以 `HUB_NO_WEB=1` 子进程调用 hub），均需主鉴权。

---

## 3. Linux 部署前置与源码适配清单

### 3.1 建议的运行时目录

当前克隆目录 `/data/workspace/Agent-Self-media-operation` 适合评估。正式部署建议：

```text
/opt/maitian/                        # 仓库目录（git 管理）
├── config.json                      # 600，不进 git
├── config.json.bak
├── data/                            # 700，运行时数据
│   ├── hub.db
│   ├── hub.db-wal
│   ├── hub.db-shm
│   ├── sessions.json
│   ├── tunnel.json
│   ├── thumbs/
│   └── *.log
├── chrome-profile/                  # 700，含平台登录态，不进 git
├── rootPath/                        # 发布工程根目录，建议单独挂载
├── scripts/
├── src/
├── skills/maitian-publish/
└── DEPLOY_LINUX.md
```

创建专用系统用户（示例）：

```bash
sudo useradd --system --home /opt/maitian --shell /usr/sbin/nologin maitian
sudo mkdir -p /opt/maitian /data/maitian/projects
sudo chown -R maitian:maitian /opt/maitian /data/maitian/projects
sudo chmod 750 /opt/maitian /data/maitian/projects
```

> 注意：Chrome 与 CDP 不建议以 root 运行。以 `maitian` 用户运行 systemd 服务更安全；如需 `--no-sandbox`，应仅在隔离容器且明确知悉风险时启用。

### 3.2 系统与二进制依赖

```bash
# Debian/Ubuntu 示例
sudo apt-get update
sudo apt-get install -y git curl ca-certificates xvfb x11vnc ffmpeg \
  imagemagick cifs-utils sqlite3

# Node.js >= 22.5（按官方 NodeSource 或 nvm 方式安装，这里只给版本校验）
node -v
node -e 'const [a,b]=process.versions.node.split(".").map(Number);process.exit(a>22||(a===22&&b>=5)?0:1)'

# 可选：飞书多维表格同步依赖
# lark-cli 需按你所在组织的安装方式提供，并确保服务用户 PATH 中可找到

# 可选：远程隧道
# cloudflared 安装后记下绝对路径，写入 config.tunnel.bin
```

设置时区（源码大量硬编码 `Asia/Shanghai` 与 `+08:00`）：

```bash
sudo timedatectl set-timezone Asia/Shanghai
# systemd unit 中同时设置 Environment=TZ=Asia/Shanghai
```

### 3.3 必须做的源码适配（正式部署前）

以下改动是 Linux 全功能部署的前置条件。建议在 fork 上以独立 commit 管理，便于 upstream 同步时 rebase。

#### (1) `src/chrome-shim.mjs`：Chrome 路径与 CDP 绑定

当前源码 `BIN` 在非 Windows 平台硬编码：

```js
: '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome';
```

建议改为：

```js
const BIN = process.platform === 'win32'
  ? (['C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe',
      path.join(os.homedir(), 'AppData', 'Local', 'Google', 'Chrome', 'Application', 'chrome.exe')].find(p => fs.existsSync(p)) || 'chrome.exe')
  : process.platform === 'darwin'
    ? '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome'
    : (process.env.MAITIAN_CHROME || '/usr/bin/google-chrome');
```

并在 `launchChrome()` 的 `args` 中增加回环绑定（CDP 无鉴权，禁止公网可达）：

```js
`--remote-debugging-address=127.0.0.1`,
```

容器 root 环境如必须运行，再评估追加 `--no-sandbox`；优先使用非 root 用户 + Xvfb。

#### (2) 图形会话：Xvfb + VNC

项目通过 `cdp.click` 依赖真实页面交互，不建议用 `--headless` 替代。

```bash
# 手动启动示例（systemd 中应固化）
Xvfb :99 -screen 0 1600x1000x24 -nolisten tcp &
export DISPLAY=:99
x11vnc -display :99 -localhost -rfbport 5900 -forever &
```

用户通过 SSH 隧道访问 VNC：

```bash
ssh -L 5900:127.0.0.1:5900 user@server
# 本地 VNC 客户端连接 127.0.0.1:5900
```

systemd 服务需设置 `Environment=DISPLAY=:99`。

#### (3) `src/web.mjs`：缩略图 `/usr/bin/sips`

`web.mjs` 使用 macOS 的 `sips` 生成缩略图，失败时会回退原图，不阻塞主流程。可选适配为 ImageMagick：

```js
// 将 sips 调用替换为 convert/magick，例如：
execFileSync('convert', [src, '-thumbnail', `${w}x${h}>`, out]);
```

不做也可运行，只是缩略图效果降级。

#### (4) `src/hub.mjs`：NAS 自动重挂

`ensureNas()` 使用 `/Volumes`、`diskutil`、`osascript`。Linux 上建议：

- 用 `/etc/fstab` 或 systemd mount unit 提前挂好 SMB/NFS；
- `config.rootPath` 直接写 Linux 本地路径；
- **删除或置空 `config.nasUrl`**，避免触发 macOS 重挂逻辑。

如必须保留自动重挂，需改写 `ensureNas()` 为 `mount -t cifs` 实现。

#### (5) `src/tunnel.mjs`：cloudflared 路径

源码默认 `CF = config.tunnel?.bin || '/opt/homebrew/bin/cloudflared'`。Linux 上必须显式配置：

```json
"tunnel": { "name": "maitian", "hostname": "maitian.example.com", "bin": "/usr/local/bin/cloudflared" }
```

#### (6) `src/init.mjs`：跳过其 macOS 专属逻辑

`init.mjs` 会检查 `/Applications/Google Chrome.app`，且仅在 `darwin` 安装 launchd。Linux 上建议**不运行向导**，改用本文第 5 节手工生成配置；如仍想用向导提取配置，运行后需复核 `chromeProfile`、`rootPath`、`nasUrl`。

#### (7) 其他可选适配

| 文件 | macOS 假设 | Linux 处理 |
|---|---|---|
| `src/make-publish.mjs:127` | IG 转码提示 `h264_videotoolbox` | 改用 `libx264`（仅提示文本，不影响主流程） |
| `src/bridge.mjs`、`hub.mjs` 长文模式 | `open -g -a`、`caffeinate`、`lsof` | 保持 `driver: direct` 且不用长文模式即可规避 |
| `hub.mjs:241` | 回填 md 路径为 `os.homedir()/doc/<source>` | 确保该目录存在，或改为基于工程目录解析 |
| `plugin.mjs` | 扩展目录在专用 Chrome profile 内 | 通过商店安装即可，路径自动发现 |

---

## 4. 数据、端口与备份范围

### 4.1 端口

| 端口 | 用途 | 暴露建议 |
|---|---|---|
| `8787` | 正式 Web/API（源码监听 `0.0.0.0`） | 仅本机/VPN；公网必须经 TLS + 访问控制 |
| `8788` | Demo Web | 仅本机 |
| `9333` | Chrome CDP（无鉴权） | **仅 127.0.0.1**，禁止公网/容器外可达 |
| `8765` | `driver: plugin` 开发桥 | 仅本机，默认不用 |

### 4.2 运行时产物

- `config.json` / `config.json.bak`
- `data/hub.db` + WAL/SHM
- `data/sessions.json`（飞书登录会话，hash token）
- `data/tunnel.json`（飞书卡片中的远程地址）
- `data/thumbs/`、`data/*.log`
- `chrome-profile/`（平台登录态 + 商店扩展）
- `rootPath/` 下所有发布工程：`publish.json`、`发布记录.json`、`发布档案/`、成片、封面

### 4.3 备份

推荐停服后整体备份，避免 SQLite WAL 不一致：

```bash
sudo systemctl stop maitian
sudo tar --numeric-owner -czpf /backup/maitian-$(date +%F).tar.gz \
  -C /opt/maitian config.json data chrome-profile
sudo systemctl start maitian
```

发布工程目录单独使用 rsync/快照：

```bash
rsync -a --delete /data/maitian/projects/ /backup/maitian-projects/
```

---

## 5. 配置清单

### 5.1 字段总表

| 字段 | 示例/默认 | 敏感 | 说明 |
|---|---|---|---|
| `name` | `麦田` | 否 | 页面与小组件标题 |
| `feishuWebhook` | `<FEISHU_WEBHOOK>` | 是 | 飞书群机器人；空则所有通知静默 |
| `plugin` | `store` | 否 | `store` 或 `local`（后者需 `pluginDir`） |
| `bridge` | `http://127.0.0.1:8765` | 否 | 仅 `driver: plugin` 使用 |
| `driver` | `direct` | 否 | 推荐保持 `direct` |
| `learnLinks` | `false` | 否 | 关闭教程/交流群链接 |
| `chromePort` | `9333` | 否 | CDP 端口 |
| `chromeProfile` | `/opt/maitian/chrome-profile` | 是 | 登录态所在，权限 700 |
| `rootPath` | `/data/maitian/projects` | 否 | 发布工程根目录 |
| `nasUrl` | 删除该字段 | 视情况 | Linux 下建议不用 |
| `batches` | `[["bilibili","douyin"],["xhs","channels"],["zsz","x"],["youtube"],["instagram"]]` | 否 | 分批发布顺序 |
| `auto` | `["zsz","x","youtube","instagram"]` | 否 | 自动提交平台 |
| `manual` | `["bilibili","douyin","xhs","channels"]` | 否 | 填好后人工点发布 |
| `keepAlive` / `keepAliveMinutes` | `["channels"]` / `60` | 否 | 保活平台与间隔 |
| `disabled` | `[]` | 否 | 停用平台，全局过滤 |
| `sampleHours` | `6` | 否 | 新一期前三天加密采样间隔 |
| `xHandle` | `<X_USERNAME>` | 否 | 采 X 粉丝用 |
| `web.port` | `8787` | 否 | Web 端口 |
| `web.key` | `<WEB_KEY>` | 是 | 网页口令；`?key=` 换 HttpOnly cookie |
| `web.login` | `"feishu"` 或删除 | 否 | 飞书验证码登录 |
| `web.widgetToken` | `<WIDGET_TOKEN>` | 是 | 小组件只读口令 |
| `web.owner` / `web.avatar` | 可选 | 否 | 小组件署名/头像 |
| `web.access` | `{team, aud, emails?}` | 否 | Cloudflare Access JWT |
| `tunnel` | `{name, hostname, bin}` | 否 | 固定域名隧道；`bin` 为 Linux 路径 |
| `site.*` | 见模板 | 管理态在 Chrome profile | L3 自有站点 |
| `board.*` | `baseToken` 为敏感 | 是 | 飞书多维表格；依赖 `lark-cli` |

代码中出现但样例未列的字段：`pluginDir`、`extensionId`、`tunnel.bin`、`web.access.emails`。

### 5.2 安全配置模板（无真实密钥）

```json
{
  "name": "麦田",
  "feishuWebhook": "<FEISHU_WEBHOOK>",
  "plugin": "store",
  "driver": "direct",
  "bridge": "http://127.0.0.1:8765",
  "learnLinks": false,
  "chromePort": 9333,
  "chromeProfile": "/opt/maitian/chrome-profile",
  "rootPath": "/data/maitian/projects",
  "batches": [["bilibili", "douyin"], ["xhs", "channels"], ["zsz", "x"], ["youtube"], ["instagram"]],
  "auto": ["zsz", "x", "youtube", "instagram"],
  "manual": ["bilibili", "douyin", "xhs", "channels"],
  "keepAlive": ["channels"],
  "keepAliveMinutes": 60,
  "disabled": [],
  "sampleHours": 6,
  "xHandle": "<X_USERNAME>",
  "web": {
    "port": 8787,
    "key": "<WEB_KEY>",
    "login": "feishu",
    "widgetToken": "<WIDGET_TOKEN>",
    "owner": "<OWNER>",
    "avatar": "data/avatar.jpg"
  },
  "tunnel": {
    "name": "maitian",
    "hostname": "maitian.example.com",
    "bin": "/usr/local/bin/cloudflared"
  },
  "site": {
    "name": "知识站",
    "enabled": true,
    "base": "https://site.example.com",
    "adminPath": "/admin",
    "coursesApi": "/api/courses?admin=1",
    "dashboardApi": "/api/admin/dashboard",
    "coursePath": "/series/{seriesId}?courseId={id}",
    "logo": "data/site-logo.png"
  },
  "board": {
    "url": "<FEISHU_BASE_URL>",
    "baseToken": "<FEISHU_BASE_TOKEN>",
    "tables": {
      "episodes": "tblxxx",
      "tasks": "tblxxx",
      "daily": "tblxxx",
      "site": "tblxxx",
      "channels": "tblxxx"
    }
  }
}
```

生成随机口令：

```bash
openssl rand -base64 24   # 用于 web.key / widgetToken
```

设置权限：

```bash
chmod 600 /opt/maitian/config.json
chmod -R 700 /opt/maitian/data /opt/maitian/chrome-profile
```

> 手工修改配置后必须重启 `maitian.service`；Web UI「系统 → 集成 / 平台」修改的字段除外。

---

## 6. Chrome、扩展与登录态

1. 启动 Xvfb + VNC（见 3.3），确保服务用户的 `DISPLAY` 正确。
2. 完成 `src/chrome-shim.mjs` 的 Linux Chrome 路径与 `--remote-debugging-address=127.0.0.1` 适配。
3. 首次登录：

```bash
cd /opt/maitian
sudo -u maitian env DISPLAY=:99 node src/hub.mjs login
```

4. 在 VNC 中打开专用 Chrome，从 Chrome Web Store 安装扩展：
   `https://chromewebstore.google.com/detail/eaaipbilkhibicihiollecmbpneikpnj`
5. 在同一 profile 中登录需要使用的平台账号。
6. 体检：

```bash
sudo -u maitian env DISPLAY=:99 node src/hub.mjs doctor
```

`plugin.mjs` 会在 `chrome-profile/Default/Extensions/eaaipbilkhibicihiollecmbpneikpnj/<版本>/` 下自动发现扩展，无需手工配置路径。

---

## 7. systemd 常驻服务

### 7.1 主服务 `/etc/systemd/system/maitian.service`

```ini
[Unit]
Description=Maitian - self media operation hub
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=maitian
Group=maitian
WorkingDirectory=/opt/maitian
Environment=NODE_ENV=production
Environment=TZ=Asia/Shanghai
Environment=DISPLAY=:99
ExecStart=/usr/bin/node /opt/maitian/src/hub.mjs daemon
Restart=always
RestartSec=10
TimeoutStopSec=30
LimitNOFILE=65535
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

> daemon 已包含 Web 服务，不要再单独启动 `hub web` 或占用 8787。

### 7.2 Xvfb 服务 `/etc/systemd/system/maitian-xvfb.service`

```ini
[Unit]
Description=Xvfb for Maitian dedicated Chrome
After=network.target

[Service]
Type=simple
User=maitian
Group=maitian
ExecStart=/usr/bin/Xvfb :99 -screen 0 1600x1000x24 -nolisten tcp
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 7.3 可选隧道服务 `/etc/systemd/system/maitian-tunnel.service`

```ini
[Unit]
Description=Maitian cloudflared tunnel
After=maitian.service
Requires=maitian.service

[Service]
Type=simple
User=maitian
Group=maitian
WorkingDirectory=/opt/maitian
Environment=TZ=Asia/Shanghai
ExecStart=/usr/bin/node /opt/maitian/src/tunnel.mjs
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

启用：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now maitian-xvfb.service
sudo systemctl enable --now maitian.service
sudo systemctl enable --now maitian-tunnel.service   # 可选
sudo systemctl status maitian.service
journalctl -u maitian.service -f
```

---

## 8. 远程访问与安全边界

1. **默认只本机访问**：Web 监听 `0.0.0.0:8787`，用防火墙限制来源，或改源码绑定 `127.0.0.1` 后由反向代理转发。
2. **不要暴露 CDP 9333 / bridge 8765**：无鉴权，必须留在回环或内网。
3. **公网 Web 需双层保护**：
   - TLS（cloudflared / nginx + certbot）；
   - `web.login: "feishu"` 验证码登录，或 `web.access` 接 Cloudflare Access。
4. **隧道模式**：
   - 固定域名：`config.tunnel = {name, hostname, bin}`，地址稳定，推荐；
   - 快速隧道：不配 tunnel，地址每次重启变化，并通过飞书卡片推送。
5. **小组件**：`/api/widget*` 只用 `widgetToken` 鉴权；扫码复制时应避免泄露口令。
6. **密钥**：`web.key`、`widgetToken`、`feishuWebhook`、`board.baseToken`、Chrome profile 均视为敏感资产。

---

## 9. 初始化与验证顺序

严格按以下顺序执行，避免上线后才发现登录态或平台页面问题。

### 9.1 基础测试

```bash
cd /opt/maitian
sudo -u maitian node --test tests/*.test.mjs
```

### 9.2 Demo 预览（不碰真实配置）

```bash
sudo -u maitian node src/demo.mjs
# 浏览器打开 http://localhost:8788/?key=demo
# 验证后 Ctrl+C 停止
```

### 9.3 生成配置

- Linux 下建议手工复制 `config.example.json` 并按第 5 节填写；
- 不要直接运行 `init.mjs` 的 launchd/Chrome 检测部分；
- 确认 `config.json` 权限 600。

### 9.4 飞书与平台联通

```bash
sudo -u maitian env DISPLAY=:99 node src/hub.mjs notify-test
sudo -u maitian env DISPLAY=:99 node src/hub.mjs login
sudo -u maitian env DISPLAY=:99 node src/hub.mjs doctor
```

在 VNC 中完成扩展安装与各平台登录。

### 9.5 L1 数据监控验收

```bash
sudo -u maitian env DISPLAY=:99 node src/hub.mjs adopt <已发布工程目录>
sudo -u maitian env DISPLAY=:99 node src/hub.mjs metrics
sudo -u maitian env DISPLAY=:99 node src/hub.mjs followers
sudo -u maitian env DISPLAY=:99 node src/hub.mjs review <id> 7 --notify
sudo -u maitian env DISPLAY=:99 node src/hub.mjs weekly --notify
```

打开 `http://localhost:8787/?key=<WEB_KEY>`，确认“分发计划 / 单期 / 平台 / 转化”页面有数据。

### 9.6 L2 分发验收

```bash
# 1. 准备发布工程：成片.mp4、封面、发布素材.md
# 2. 生成 publish.json
sudo -u maitian node src/make-publish.mjs "<发布素材.md>" "<工程目录>" --require <账号名>

# 3. 校验（必须关注 faststart）
sudo -u maitian node src/validate.mjs "<工程目录>"

# 4. 计划 + 试跑
sudo -u maitian env DISPLAY=:99 node src/hub.mjs plan "<工程目录>"
sudo -u maitian env DISPLAY=:99 node src/hub.mjs run "<工程目录>" --only youtube,x --now

# 5. 国内四家在 VNC 中人工点击发布，再由中控取链接
sudo -u maitian env DISPLAY=:99 node src/hub.mjs sync <id>
```

如自动提交卡住且页面已填好，使用 `submit <id> <平台>`，不要另写脚本点击。

### 9.7 L3 转化验收

- 自有站点实现 `src/site.mjs` 顶部约定的两个只读 API；
- 平台文案链接带 `utm_source`、`utm_medium=video`、`utm_campaign=<主题>-<序号>`；
- 在专用 Chrome 中登录站点管理后台；
- 配置 `board.*` 并提供 `lark-cli`；
- 执行 `node src/hub.mjs board` 与 `node src/hub.mjs review <id>` 验证转化段。

### 9.8 守护进程验收

```bash
sudo systemctl start maitian.service
sudo systemctl status maitian.service
curl -s "http://127.0.0.1:8787/api/overview?key=<WEB_KEY>" | head
```

`/api/overview` 的 `health` 字段应包含 `chrome`、`metricsDay`、`doctorDay`、`platforms`、`nas` 等健康信息。

---

## 10. 升级与回滚

```bash
cd /opt/maitian
sudo -u maitian git fetch --tags
sudo -u maitian git tag pre-upgrade-$(date +%F-%H%M)

# 停服 + 备份
sudo systemctl stop maitian maitian-tunnel
sudo tar --numeric-owner -czpf /backup/maitian-pre-upgrade.tar.gz config.json data chrome-profile

# 升级
sudo -u maitian git pull --ff-only
sudo -u maitian node --test tests/*.test.mjs

# 启动
sudo systemctl start maitian maitian-tunnel
```

回滚：

```bash
sudo systemctl stop maitian maitian-tunnel
sudo -u maitian git checkout pre-upgrade-<timestamp>
sudo tar --numeric-owner -xzpf /backup/maitian-pre-upgrade.tar.gz -C /opt/maitian
sudo systemctl start maitian maitian-tunnel
```

---

## 11. 操作规程与风险预案

1. **国内四家平台最终发布必须人工点击**：B站、抖音、小红书、视频号只自动填写；不要绕过。
2. **同一视频每个平台只上传一次**：失败后先去平台后台确认，再用 `--force` / `mark` / `submit` 补救。
3. **成片必须 faststart**：`validate.mjs` 会拦截 `mdat` 在 `moov` 前的 MP4；导出时加 `-movflags +faststart`。
4. **定时偏差告警**：视频号等平台可能改定时；`sync` 会比对并在偏差 > 60 秒时告警。
5. **登录态保活**：`keepAlive` 平台（如视频号）按 `keepAliveMinutes` 巡检；掉线时 20 分钟内补跑。
6. **停用平台**：用 `disabled` 或 Web「系统 → 平台」关闭，不登录、不体检、不发布、不采粉丝。
7. **Chrome 卡死**：通过 CDP 关闭卡住标签页，或重启 `maitian.service`；profile 不要删除。
8. **不要做互动自动化**：项目红线是不评论、不点赞、不私信、不关注他人账号。
9. **配置变更需重启**：手工改 `config.json` 后重启服务；Web UI 改的字段除外。
10. **上游差异**：本仓库是 fork，README / 安装脚本多处指向 `CzzzzzzJ/mdldm-maitian`。升级前应确认要同步的是哪个版本，不要盲目运行上游一键脚本。

---

## 12. 安全审计摘要

审计对象：`skills/maitian-publish/SKILL.md`、`agent-install.md`、`scripts/`、`scripts/widget/`、`src/init.mjs`、`src/tunnel.mjs`。

### 12.1 结论

- 🔴 **P0 阻断级：0 个**
- ⚠️ **P1 需关注：4 个**
- ✅ **P2 安全/预期行为：其余**

未发现发布 skill 自动执行 `curl | bash`、`irm | iex` 或下载后 `eval`；未发现读取 `~/.ssh` / `.env` 并外发；未发现硬编码真实密钥；未发现 `sudo` / `chmod 777` 等权限提升组合。SKILL.md 的 `description` 无异常堆砌，无占位符 license。

### 12.2 P1 明细

1. **安装脚本中的有界破坏性操作**
   - 位置：`scripts/install.sh:17`、`scripts/install-windows.ps1:57`
   - 行为：clone 失败重试时执行 `rm -rf "$R"` / `Remove-Item -Recurse -Force $dest`
   - 判定：作用范围限于用户指定的安装子目录，且由用户主动运行安装脚本触发，不是 skill 自动执行；但仍需避免误设 `MAITIAN_HOME`。
   - 建议：正式部署不要运行一键脚本，使用本文手工 clone + 审计后路径。

2. **Windows 安装器自动全局安装依赖**
   - 位置：`scripts/install-windows.ps1:23,41`
   - 行为：缺 Git/Node 时自动 `winget install`
   - 判定：系统级环境变更，属于 P1 环境风险；Linux 部署不涉及。

3. **小组件加载器下载并执行远端代码**
   - 位置：`scripts/widget/maitian-loader.js:10-24`
   - 行为：每次刷新从 `/api/widget/code` 拉代码，`new AsyncFunction(code)()` 执行；仅校验包含 `Script.complete()`，无签名校验。
   - 判定：这是“自动更新小组件”的设计，代码来源是用户自己的麦田服务且有 `widgetToken` 保护；不属于 skill 自动执行，但属于远程代码执行模式。
   - 建议：确认服务器与 token 安全；如不需要小组件，不要安装该脚本。

4. **上游仓库路径与当前 fork 不一致**
   - 位置：`agent-install.md:49,60`、`scripts/install.sh:3,17`、`scripts/install-windows.ps1:2,55`、`skills/maitian-publish/SKILL.md:8`
   - 行为：文档/脚本统一指向 `CzzzzzzJ/mdldm-maitian`，而本次部署对象是 `guts-yang/Agent-Self-media-operation`。
   - 判定：直接仓库安装的供应链关注项；若 Agent 盲从 `agent-install.md`，会安装上游版本而非当前 fork。
   - 建议：部署与升级始终以当前 fork 的 commit 为准；如确需合并上游，先 diff 再审。

### 12.3 详细检查记录

- **命令执行**：命中 `git clone`、`git pull --ff-only`、`node src/init.mjs`、`node src/hub.mjs doctor`、`spawn(CF, ...)` 等；均需人工/用户触发或属于项目预期功能，未构成自动投毒组合。
- **隐蔽执行**：命中 `2>/dev/null`（`launchctl bootout` 容错）、`git -q`、`Out-Null`；用于抑制预期输出，未与危险操作组合。
- **网络请求**：域名集中于 `github.com`、`raw.githubusercontent.com`、`chromewebstore.google.com`、`nodejs.org`、`git-scm.com`、`open.feishu.cn`、Cloudflare 与 localhost；未发现可疑动态外发。
- **敏感路径**：仅出现 `~/.claude/skills`、`~/.maitian` 类安装目录，未出现 `~/.ssh` / `/etc/shadow` / 云凭证读取。
- **文件操作**：写配置、建目录、软链、缓存小组件代码、卸载 launchd plist；均在预期安装/运行范围内。
- **权限提升**：未发现 `sudo` / `chmod 777`；Windows 一键命令注释含 `ExecutionPolicy Bypass`，需用户手动复制执行。
- **依赖安装**：无 npm/pip 全局安装；Windows 脚本会 `winget install` Git/Node（见 P1-2）。
- **Base64/混淆**：未发现可疑长串编码载荷；正则命中均为普通标识符或中文文本片段。
- **元数据**：`SKILL.md` frontmatter 仅 `name` 与 `description`，长度正常；仓库 `LICENSE` 为 MIT。

---

## 13. 最终验收清单

- [ ] 仓库克隆到目标路径，记录 commit，工作区干净
- [ ] Node ≥ 22.5、`TZ=Asia/Shanghai`、Chrome、Xvfb/VNC 就绪
- [ ] Linux 适配 commit 已应用（Chrome 路径、CDP 回环、cloudflared 路径）
- [ ] `config.json` 已生成，权限 600，`chrome-profile`/`data` 权限 700
- [ ] `node --test tests/*.test.mjs` 通过
- [ ] `node src/demo.mjs` 在 8788 可访问
- [ ] 飞书 `notify-test` 收到卡片
- [ ] 扩展已安装，各平台 `doctor` 通过
- [ ] L1：`adopt` / `metrics` / `followers` / `review` / `weekly` 有数据
- [ ] L2：`validate` → `plan` → `run` → 人工发布 → `sync` 全流程走通
- [ ] L3：站点 API + UTM + `board` 验证通过
- [ ] systemd `maitian.service`（及可选 tunnel/xvfb）已 enable 并自动拉起
- [ ] 8787 有访问控制；9333/8765 不对公网开放
- [ ] 备份、升级、回滚脚本已演练一次
