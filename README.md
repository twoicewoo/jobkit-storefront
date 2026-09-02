# 虚拟定制创业操作系统（eshop）

直接执行入口见 `business/START-HERE.md`。公开销售页：https://twoicewoo.github.io/jobkit-storefront/ 。运行 `npm run business:serve` 可在本机预览。

当前首发产品是**简历投递加速包**：客户提交真实经历和目标岗位，项目自动生成岗位定向简历、可编辑 Word、A4 PDF、求职信、面试准备卡和可打印 HTML。市场研究 CLI 保留为选品与迭代证据层。

```bash
npm run business:demo
npm run business:fulfill -- business/intake/<orderId>.json
npm run business:audit -- business/orders/<orderId>
```

上架文案见 `business/storefront/listing.md`，可直接发送给客户的资料表单见 `business/storefront/order-form.html`，逐单 SOP 与真实盈利门槛见 `business/OPERATIONS.md`。每次履约还会生成可直接发送的 `<orderId>-delivery.zip`。

---


一个只在本机运行的**纯 CLI 数据工具**，负责闲鱼登录、确定性采集、SQLite 历史、证据查询、能力画像、蓝海评分、分析存档与受限导出。宿主 AI Agent（Claude Code / Codex / Qoder 等）负责关键词选择、解释、结论与人类报告——采集完成后 CLI 可**主动调用本机 Agent 做可行性评估**。

- 不依赖 MCP，不依赖 DSH 插件，不开 HTTP 服务。
- 项目内不调用模型、不需要模型 API Key（评估由本机 Agent 完成，可用其自身账号）。
- 每个业务日固定一个不可变关键词计划，目标为 **2000** 个唯一商品，桶配额 `1200 / 500 / 300`。
- 登录或安全验证出现后立即停止后续平台请求，可在人工处理后恢复同一计划。
- listing 仅表示卖家供给或市场信号，不表示已验证成交。
- 会员、账号、充值、激活码、卡密及第三方额度属于不可控供给，不能直接进入测试。

## 环境要求

- macOS
- Node.js 22 或更高版本
- `/Applications/Google Chrome.app/Contents/MacOS/Google Chrome`（可用环境变量 `ESHOP_CHROME_PATH` 指向其他 Chrome 可执行文件）

## 首次安装

```bash
cd <项目根>
npm ci
npm run build
npm run doctor
```

`doctor` 检查 Node、Chrome、SQLite 完整性、本地目录、构建、Profile 锁和最近运行；最后一行输出机器可读 JSON。它不会触发采集。

### 安装 Skill 到本机 AI Agent

```bash
./scripts/install-local-agents.sh
```

该脚本构建 CLI，并把 canonical Skill（`xianyu-virtual-custom-research`）以相对软链接接入 **Claude Code**（`.claude/skills`）、**Qoder**（`.qoder/skills`）和 **Codex**（`~/.codex/skills`，若已安装）。脚本可重复运行，不修改用户级全局宿主配置。可选：`npm link` 后可直接使用 `eshop <command>`。

在任一 Agent 中开始研究时显式调用 Skill：

```text
$xianyu-virtual-custom-research 执行今天的闲鱼虚拟定制市场研究。
```

## CLI 命令

```text
eshop status [--json]                    登录状态 + 最近运行 + 能力画像
eshop login                              打开 Chrome 完成闲鱼登录
eshop doctor                             健康检查
eshop history [--days N] [--from D] [--to D] [--json]
eshop evidence --keyword K [--date D] [--min-price N] [--max-price N] [--limit N] [--cursor C] [--json]
eshop collect --plan plan.json [--date D] [--json]
eshop resume [--date D] [--json]
eshop score --input request.json [--json]
eshop save-analysis --input analysis.json [--evidence-ids a,b] [--json]
eshop profile get|update --input profile.json [--json]
eshop export --format jsonl|csv --basename X [筛选] [--json]
eshop notify [claude|codex|qoder|auto|none] [--date D] [--dry-run] [--report-dir DIR]
eshop mcp                              （保留）以 stdio MCP server 方式运行
```

`--json` 输出机器可读 JSON（供 AI Agent 使用）。所有命令在项目根目录运行；开发时用 `tsx src/cli.ts`。

## 每日研究

每日研究按 canonical Skill 执行：读能力画像与近 30 天历史 → 检查登录 → 生成当天不可变 `1200/500/300` 计划（2000 条）→ 采集/恢复 → 复核证据 → `eshop score` 蓝海评分 → `eshop save-analysis` 保存 `agent-analysis/v2` → 输出五段式创业者决策简报。Agent 不复制或修改 Skill 的评分阈值、提示词和硬门槛。

第 1、2 天的候选保持 `RESEARCH`；第 3 天才有资格接受确定性 `PASS` 或 `GRAY` 判断，资格不等于晋级。`BUYER_REQUEST` 是可选增强证据；卖家侧信号只可说明可比市场的相对热度和竞争，不能说明销量、成交、订单或付款意愿。付款意愿在产生行为证据的有界 `MICRO_TEST` 前保持未知。

人类主报告固定为五段：今天的决定、机会排行榜、商业判断、距离下注还差什么、明天只做什么。报告先回答是否花钱和哪个方向更值得继续，再说明客户价值、供给缺口、差异化、赚钱空间和证据缺口。买方帖子只是可选增强，不是每日必做或晋级前提；英文枚举、版本、哈希、原始指标和证据 ID 单独置于最后折叠的“审计详情”。

## 采集后主动通知本机 Agent 评估

```bash
eshop notify auto            # 自动探测 claude / codex / qoder 并调用
eshop notify codex --date 2026-08-22
eshop notify auto --dry-run  # 只打印评估 prompt，不调用
```

`notify` 会：

1. 把登录状态、最近运行、评分卡、能力画像快照到 `.data/reports/<date>-snapshot.json`；
2. 调用本机已安装的 Agent CLI（Claude Code `-p` / Codex `exec` / Qoder `-p`）headless 执行评估；
3. Agent 按 Skill 规则产出可行性判定（`NO_BET` / `RESEARCH` / `MICRO_TEST` / `EXCLUDED`）+ 五段式简报；
4. 评估报告写入 `.data/reports/<date>-eval.md` 并打印摘要。

Agent 在项目根目录运行，可用上述 `eshop` 命令深挖证据后再给结论。

## 每日自动调度

安装 LaunchAgent 后每天 09:00 自动执行：登录检查 → 当天计划采集 → 通知本机 Agent 评估。

```bash
./scripts/install-daily-research.sh install   # 安装（每天 09:00）
./scripts/install-daily-research.sh kick      # 立即触发一次
./scripts/install-daily-research.sh uninstall # 卸载
```

- 未登录/需验证时自动跳过采集并写日志（需人工 `eshop login`），不会自动访问平台；
- 当天计划放在 `.data/plans/YYYY-MM-DD.json`（由 Agent 按 skill 预生成，配额 1200/500/300）；
- 日志：`.data/logs/daily-research.log`。

## 数据位置

```text
.data/monitor.sqlite          SQLite 事实、运行、计划、证据和分析
.data/xianyu-profile/         本地 Chrome Profile
.data/xianyu-profile.lock     跨进程 Profile 锁
.data/exports/                JSONL/CSV 导出
.data/reports/                评估快照与报告
.data/backups/                迁移前内容寻址数据库备份
```

可通过环境变量把运行数据指向隔离目录：`ESHOP_DB_PATH` / `ESHOP_PROFILE_DIR` / `ESHOP_PROFILE_LOCK_PATH` / `ESHOP_EXPORT_DIR` / `ESHOP_CHROME_PATH`。

## 验证

```bash
npm run typecheck
npm run build
npm test
node dist/cli.js doctor
node dist/cli.js status --json
```
