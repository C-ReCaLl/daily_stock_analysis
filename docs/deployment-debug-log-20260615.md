# DSA 部署调试记录（2026-06-15）

记录本次 DSA（Daily Stock Analysis）从本地最短启动到 GitHub Actions 自动化定时运行的完整调试过程，便于后续回溯与排错。

时间：2026-06-15  
时区：Asia/Shanghai  
仓库：C-ReCaLl/daily_stock_analysis  
推送通道：企业微信群机器人

## 1. 目标

- 按官方文档「最短启动」流程在本地把项目跑起来。
- 通过企业微信机器人接收每日分析推送。
- 让任务在工作日定时自动运行：08:30 左右收到盘前大盘/新闻检查，16:00 左右收到收盘完整分析。
- 长期改用 GitHub Actions 运行，避免依赖本地环境。

## 2. 本地最短启动

参照文档 `https://dsa.zhulinsen.tech/docs/local/`，执行以下步骤：

```bash
git clone https://github.com/C-ReCaLl/daily_stock_analysis.git
cd daily_stock_analysis
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
python main.py --webui-only
```

WebUI 默认监听 `http://127.0.0.1:8000`。

### 2.1 兼容性问题：FastAPI 路由报错

直接安装 `requirements.txt` 后启动 WebUI，进程长期不监听 8000 端口。  
排查发现是 `FastAPI 0.137` 对空路径路由更严格，导致项目部分接口注册失败。

修复方式：

```bash
pip install 'fastapi==0.115.14'
```

降级后 WebUI 正常启动。

## 3. .env 配置

本地 `.env` 中实际用到的关键变量：

| 变量 | 用途 |
| --- | --- |
| `STOCK_LIST` | 自选股列表，A 股代码用逗号分隔 |
| `ANSPIRE_API_KEYS` | AI 大模型 + 联网搜索一体的 API Key |
| `WECHAT_WEBHOOK_URL` | 企业微信群机器人 Webhook 地址 |
| `TUSHARE_TOKEN` | Tushare Pro 行情数据接口 Token |

`WECHAT_WEBHOOK_URL` 写法：

```env
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=机器人key
```

写入后用项目自带诊断验证：

```bash
python main.py --check-notify
```

输出确认 `已配置 1 个通知渠道：企业微信`，并通过一次 markdown 测试推送验证链路：

```text
HTTP 200
errcode: 0
errmsg: ok
```

## 4. 完整分析首跑表现

`python main.py` 实际跑完一次完整分析的耗时受外部行情源稳定性影响很大。  
本次首跑日志中出现的常见问题：

```text
Eastmoney 历史K线接口超时
EfinanceFetcher TimeoutError
AkshareFetcher 切换备用源
PytdxFetcher 连接失败
Baostock 登录失败
YFinance RateLimit
```

项目按优先级在 5 个数据源之间自动切换，但单源超时窗口为 30–60 秒，多只股票连续超时会显著拖慢整体耗时。

7 只 A 股自选股的整体耗时大致结论：

| 情况 | 预计耗时 |
| --- | --- |
| 行情源正常 | 8–15 分钟 |
| 部分行情源超时 | 20–40 分钟 |
| 多源连续超时 | 40 分钟以上 |

由于历史 K 线源未成功返回，第一份推送中带有「技术数据缺失，仅基于新闻和实时行情分析」提示，AI 输出整体偏「观望」。  
为此追加配置 `TUSHARE_TOKEN`，用更稳定的 Pro 接口补充日线、交易日历和基础财务数据。

## 5. TRAE 自动任务（已暂停）

为了快速验证「按交易日定时推送」的体感，先在 TRAE 自动任务里建了两个定时：

| 任务 | Cron（北京时间） | 命令 |
| --- | --- | --- |
| DSA 盘前大盘新闻检查 | `15 8 * * 1-5` | `.venv/bin/python main.py --market-review` |
| DSA 收盘完整分析 | `20 15 * * 1-5` | `.venv/bin/python main.py` |

实际触发时收到报错：

```text
你选择的文件夹中没有找到 daily_stock_analysis 项目目录，也没有找到任何 main.py
```

原因：TRAE 自动任务运行时看到的「已选择文件夹」未必是当前会话里克隆项目的父目录，对长期定时任务不够稳定。  
切换到 GitHub Actions 后已将这两个任务全部 `pause`。

## 6. GitHub Actions 自动化（最终方案）

### 6.1 修改的工作流文件

`.github/workflows/00-daily-analysis.yml` 调整内容：

- 定时触发改为北京时间工作日 08:15 与 15:20（对应 UTC `15 0 * * 1-5` 与 `20 7 * * 1-5`）。
- 08:15 自动以 `market-only` 模式运行，目标 08:30 推送大盘/新闻检查。
- 15:20 自动以 `full` 模式运行，目标 16:00 推送收盘完整分析。
- 默认 `timeout-minutes` 从 30 调高到 90，避免外部行情源超时拖累被强行终止。
- `workflow_dispatch` 手动触发的 `mode` 选项不变，保留 `full` / `market-only` / `stocks-only` / `single-stock`。

提交记录：

```text
Update DSA scheduled analysis workflow
```

### 6.2 GitHub Secrets

仓库 `Settings → Secrets and variables → Actions` 中已配置：

```text
STOCK_LIST
ANSPIRE_API_KEYS
WECHAT_WEBHOOK_URL
TUSHARE_TOKEN
```

`workflow` 文件中通过 `${{ secrets.* }}` 读取，运行时仅在 GitHub Runner 内可见。

### 6.3 推送遇到的权限问题

第一次 `git push` 被 GitHub 拒绝：

```text
refusing to allow a Personal Access Token to create or update workflow without workflow scope
```

原因是 PAT 缺少 `workflow` scope。Classic Token 需要勾选 `repo` 与 `workflow`；Fine-grained Token 需要 `Contents: Read and write` 与 `Workflows: Read and write`。  
换用带 `workflow` 权限的新令牌后推送成功。

### 6.4 测试触发

通过 GitHub REST API 触发一次 `workflow_dispatch`：

```text
mode = market-only
force_run = true
```

运行结果：

```text
status: completed
conclusion: success
```

各步骤全部通过：

```text
检出代码 → 设置 Python 环境 → 安装依赖 → 创建必要目录 →
执行股票分析 → 上传分析报告 → 显示运行结果
```

说明 `.github/workflows/00-daily-analysis.yml` 配置在 GitHub Runner 上可用，企业微信机器人推送链路也验证通过。

## 7. 当前状态

| 项 | 状态 |
| --- | --- |
| 本地 WebUI（`python main.py --webui-only`） | 可用，依赖 `fastapi==0.115.14` |
| 本地 `.env` | 已配置 `STOCK_LIST` / `ANSPIRE_API_KEYS` / `WECHAT_WEBHOOK_URL` / `TUSHARE_TOKEN` |
| 企业微信机器人推送 | 已通过 `--check-notify` 与一次手动 markdown 推送验证 |
| TRAE 自动任务 | 两个 DSA 任务已 `pause` |
| GitHub Actions 工作流 | 已推送 `main`，并以 `market-only` 完成一次 `success` 测试 |
| 定时计划 | 工作日北京时间 08:15 与 15:20 自动触发 |

## 8. 后续优化建议

- 把 GitHub PAT 限定为「Fine-grained Token」，仅授权该仓库的 `Contents` / `Workflows` 权限，并定期轮换。
- 任何泄露过的 PAT 立即在 `Settings → Developer settings → Personal access tokens` 中撤销。
- Tushare Token 不要写进 README、Issue、PR 描述，仅放在 GitHub Secrets 与本地 `.env` 中。
- 如需更稳定的实时行情，可在文档中评估 TickFlow 等付费源；本仓库行情源优先级已支持自动切换，新增源时按 `data_provider` 现有接口扩展。
- 若希望盘后报告技术数据更完整，可把 `15 7 * * 1-5` 调整为 `30 7 * * 1-5` 或更晚，给收盘后落库与新闻沉淀更多时间。
