# AHR999 Optimized 自动化定投系统

> **免责声明**：本项目仅供学习研究使用，不构成任何投资建议。加密货币投资存在极高风险，请在充分了解风险后自行决策。使用前请参考 `ahr999_backtest.html` 查看历史回测数据。

基于 AHR999 指标与幂律增长模型的开源自动化定投工具，支持 OKX 交易所，可一键部署到 GitHub Actions，**无需服务器、无需长期开机**。

---

## 目录

- [项目原理](#项目原理)
- [核心功能](#核心功能)
- [快速上手（GitHub Actions 部署）](#快速上手github-actions-部署)
- [详细配置说明](#详细配置说明)
- [两种运行模式](#两种运行模式)
- [邮件报告解读](#邮件报告解读)
- [本地运行](#本地运行)
- [常见问题](#常见问题)
- [文件结构](#文件结构)

---

## 项目原理

### AHR999 指标是什么？

AHR999 是一个衡量比特币当前价格相对于"合理估值"贵或便宜的比率指标，由数字货币爱好者 ahr999 提出。

**计算公式：**

```
AHR999 = (当前价格 / 200日均成本) × (当前价格 / 幂律增长估值)
```

**信号解读：**

| AHR999 值 | 市场含义 | 本系统动作 |
|-----------|---------|-----------|
| < 0.45 | 极度低估，历史抄底区间 | **2x 加倍买入** |
| 0.45 ~ 1.20 | 合理定投区间 | **1x 正常买入** |
| > 1.20 | 高估区间，风险偏高 | **停止买入** |

> SOL 的上沿阈值为 1.50，因其历史数据较短，中位数更高。

### 幂律增长模型

每种资产使用独立拟合的幂律斜率，反映该资产历史上的增长曲线：

| 资产 | 斜率 | 创世日期 | 拟合 R² | 数据年限 |
|------|------|---------|---------|---------|
| BTC | 4.7777 | 2009-01-03 | 0.78 | 15年 |
| ETH | 1.9872 | 2015-07-30 | 0.58 | 10年 |
| SOL | 1.4446 | 2020-03-16 | 0.53 | 5.5年 |

> R² 越低代表模型可信度越低，系统会在邮件中提示。ETH 和 SOL 的 R² 较低，建议结合其他信号交叉验证。

### 防落刀过滤器

价格处于底部区间 ≠ 安全买入时机。本系统会额外检测近期价格动量，避免在自由落体时重仓。

| 动量状态 | 判定条件 | 执行倍数 |
|---------|---------|---------|
| STABLE（平稳） | 7日跌幅 < 阈值 | **1.0x**（正常执行）|
| STABILIZING（趋稳） | 急跌后已从低点反弹 ≥ 5% | **0.75x**（适度减仓）|
| FALLING（急跌） | 7日跌 > 15% 且未反弹 | **0.40x**（轻仓参与）|

> FALLING 状态保留 0.4x 而非 0：底部往往就诞生于持续下跌中，完全缺席会错失机会。

---

## 核心功能

### 信号计算引擎（`ahr999.py`）

- 从本地 CSV 读取历史 4H K 线，重采样为日线
- 计算 200 日几何均线成本（`gmean`）
- 计算幂律估值（`10^(slope × log10(days) + intercept)`）
- 输出 AHR999 值、历史分位、动量状态、综合评分（0~100）

### 自动 K 线更新（`DataUpdater`）

- 每次运行前自动连接 OKX 公开行情接口
- 检测本地 CSV 末尾时间戳与当前时间的差值
- 分页拉取所有已收盘的新 4H K 线（最多 2000 根/币种）
- 追加写入本地 CSV，并通过 GitHub Actions 自动提交回仓库
- **无需付费 API**，OKX 行情接口完全免费

### 预算管理（`Budget`）

支持两种模式，详见[详细配置说明](#详细配置说明)。

### 资金分配算法（`allocate`）

- 按各币种的 `final_mult`（AHR999 倍数 × 动量倍数）作为权重
- 应用各币种单次最大权重上限（BTC 60%，ETH/SOL 各 50%）
- 3 轮迭代归一化，确保权重收敛且总和为 1

---

## 快速上手（GitHub Actions 部署）

### 前置准备

- GitHub 账号（免费）
- OKX 账号 + API Key（仅信号邮件模式可不需要）
- 一个邮箱（用于接收信号报告）

---

### 第一步：Fork 仓库

1. 打开本项目的 GitHub 页面
2. 点击右上角 **Fork**
3. 选择你自己的账号，点击 **Create fork**
4. Fork 完成后，你的账号下会出现一个同名仓库

> 建议将仓库设为 **Private（私有）**，因为仓库中包含你的 K 线数据文件。

---

### 第二步：获取邮箱授权码

根据你使用的邮箱选择对应教程：

<details>
<summary><b>Gmail（推荐）</b></summary>

1. 登录 Google 账号，访问 [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
2. 需要先开启两步验证（如未开启，按提示操作）
3. 在"选择应用"中选择"邮件"，"选择设备"选择"其他"，输入名称如 `AHR999`
4. 点击"生成"，复制生成的 16 位密码（如 `abcd efgh ijkl mnop`）
5. **去掉空格**后填入 `SMTP_PASSWORD`（即 `abcdefghijklmnop`）

```
SMTP_HOST:     smtp.gmail.com
SMTP_PORT:     587
SMTP_USER:     your@gmail.com
SMTP_PASSWORD: abcdefghijklmnop
```
</details>

<details>
<summary><b>QQ 邮箱</b></summary>

1. 登录 QQ 邮箱网页版 → 设置 → 账户
2. 找到「POP3/IMAP/SMTP/Exchange/CardDAV/CalDAV 服务」
3. 开启「SMTP 服务」，按提示发短信验证
4. 获得授权码（16 位），填入 `SMTP_PASSWORD`

```
SMTP_HOST:     smtp.qq.com
SMTP_PORT:     465
SMTP_USER:     your@qq.com
SMTP_PASSWORD: xxxxxxxxxxxxxxxx
```
</details>

<details>
<summary><b>163 邮箱</b></summary>

1. 登录 163 邮箱网页版 → 设置 → POP3/SMTP/IMAP
2. 开启「SMTP 服务」，设置客户端授权密码
3. 填入以下配置：

```
SMTP_HOST:     smtp.163.com
SMTP_PORT:     465
SMTP_USER:     your@163.com
SMTP_PASSWORD: xxxxxxxx
```
</details>

---

### 第三步：获取 OKX API Key

> 如果你只需要**信号邮件提示**（手动定投），可跳过此步，API Key 留空也能正常运行。

1. 登录 [OKX 官网](https://www.okx.com)
2. 右上角头像 → **API** → **创建 V5 API Key**
3. 类型选择「交易」，填写备注名（如 `AHR999`）
4. **权限只勾选「交易」，不要勾选「提币」**
5. 完成验证后记录三个值：API Key、Secret Key、Passphrase

---

### 第四步：配置 GitHub Secrets

1. 进入你 Fork 的仓库
2. 点击顶部 **Settings** → 左侧 **Secrets and variables** → **Actions**
3. 点击 **New repository secret**，逐一添加以下密钥：

| Secret 名称 | 填写内容 | 是否必填 |
|------------|---------|---------|
| `SMTP_HOST` | 邮件服务器（如 `smtp.gmail.com`）| ✅ 必填 |
| `SMTP_PORT` | 端口（`587` 或 `465`）| ✅ 必填 |
| `SMTP_USER` | 发件邮箱地址 | ✅ 必填 |
| `SMTP_PASSWORD` | 邮箱授权码 | ✅ 必填 |
| `EMAIL_TO` | 收件邮箱（可与发件相同）| ✅ 必填 |
| `OKX_API_KEY` | OKX API Key | 仅自动交易时必填 |
| `OKX_API_SECRET` | OKX Secret Key | 仅自动交易时必填 |
| `OKX_API_PASSPHRASE` | OKX Passphrase | 仅自动交易时必填 |

#不填写交易所api无法获取最新k线数据#

---

### 第五步：配置定投策略

打开仓库中的 `.github/workflows/auto_invest.yml`，找到 `env` 部分按需修改：

```yaml
env:
  BUDGET_MODE: MONTHLY       # MONTHLY = 月度预算自动均摊 | FIXED = 每次固定金额
  BUDGET_AMOUNT: '700'       # MONTHLY: 月度上限 | FIXED: 单次金额（USDT）
  RUN_INTERVAL_DAYS: '7'     # 执行间隔，必须与 cron 保持一致
  SIMULATED: 'true'          # true = 模拟盘 | false = 真实交易
```

---

### 第六步：手动测试

1. 进入仓库，点击顶部 **Actions**
2. 左侧找到 **AHR999 Auto Invest**
3. 点击右侧 **Run workflow** → **Run workflow**
4. 等待约 1~2 分钟，确认运行结果为绿色 ✅
5. 检查邮箱，应收到一封信号报告

**收到邮件 = 配置成功。** 之后每周一 09:00 北京时间自动运行，无需任何操作。

---

## 详细配置说明

### MONTHLY 模式（推荐）

每月设定总预算上限，系统自动按执行间隔均匀分配，月度超支自动停止，月末结余自动补足。

**示例：月预算 $700，每周执行**
- 每月约 4.3 次（30 ÷ 7）
- 每次目标 ≈ $163
- 月末仅剩 $100 时，系统自动将本次调整为 $100，不浪费额度

```yaml
BUDGET_MODE: MONTHLY
BUDGET_AMOUNT: '700'
RUN_INTERVAL_DAYS: '7'
```

### FIXED 模式

每次执行固定金额，不做月度追踪，完全由你控制投入总量。

```yaml
BUDGET_MODE: FIXED
BUDGET_AMOUNT: '175'
RUN_INTERVAL_DAYS: '7'
```

### 执行频率对照表

修改 `cron` 和 `RUN_INTERVAL_DAYS` **必须同步**，否则预算计算出错。

| 频率 | cron 表达式 | RUN_INTERVAL_DAYS |
|------|------------|------------------|
| 每周一 09:00 CST（默认）| `0 1 * * 1` | `7` |
| 每天 09:00 CST | `0 1 * * *` | `1` |
| 每两周（周一）| `0 1 * * 1/2` | `14` |
| 每月 1 日 | `0 1 1 * *` | `30` |

---

## 两种运行模式

### 模式 A：信号邮件 + 手动定投

系统发送信号报告，你手动登录 OKX 按建议金额下单。**零风险，推荐新手。**

在 `auto_invest.yml` 中设置：
```yaml
run: python ahr999_dca.py --notify
```
无需配置 OKX API Secrets。

### 模式 B：全自动交易

系统完成数据更新、信号计算、资金分配、下单全流程。

在 `auto_invest.yml` 中设置：
```yaml
SIMULATED: 'false'
# ...
run: python ahr999_dca.py
```

> 建议先以 `SIMULATED: 'true'` 运行几周确认逻辑正确后，再切换为真实交易。

---

## 邮件报告解读

```
AHR999 定投信号  2026-02-24
策略: $700/月  周投
================================================

当前信号
------------------------------------------------
  BTC   价格  67,582.95 USDT  AHR999 0.4769  定投区  STABLE    建议 1.00x
  ETH   价格   1,963.95 USDT  AHR999 0.3358  极低估  STABLE    建议 2.00x
  SOL   价格      84.33 USDT  AHR999 0.2738  极低估  STABLE    建议 2.00x

本次分配建议（$163.33 USDT）
------------------------------------------------
  BTC     $ 32.67 USDT  权重 20%  倍数 1.00x
  ETH     $ 65.33 USDT  权重 40%  倍数 2.00x
  SOL     $ 65.33 USDT  权重 40%  倍数 2.00x
  合计    $163.33 USDT

月预算 $700  已花 $163.33  剩余 $536.67
```

| 字段 | 含义 |
|------|------|
| `AHR999 0.4769` | 当前估值比率，越低越便宜 |
| `极低估 / 定投区 / 观望区` | 当前价格所处区间 |
| `STABLE / STABILIZING / FALLING` | 近期价格动量状态 |
| `建议 X.XXx` | AHR999 区间倍数 × 动量倍数 |
| `权重 XX%` | 本次预算中该币种的分配比例 |

---

## 本地运行

### 安装依赖

```bash
pip install pandas numpy scipy requests
```

### 设置环境变量

```bash
export OKX_API_KEY="your_api_key"
export OKX_API_SECRET="your_api_secret"
export OKX_API_PASSPHRASE="your_passphrase"
export SMTP_HOST="smtp.gmail.com"
export SMTP_PORT="587"
export SMTP_USER="your@gmail.com"
export SMTP_PASSWORD="your_app_password"
export EMAIL_TO="your@gmail.com"
export BUDGET_MODE="MONTHLY"
export BUDGET_AMOUNT="700"
export RUN_INTERVAL_DAYS="7"
export SIMULATED="true"
```

### 命令列表

```bash
python3 ahr999_dca.py --update          # 仅更新 K 线数据
python3 ahr999_dca.py --notify          # 发送信号邮件（不交易）
python3 ahr999_dca.py --dry-run         # 模拟完整流程（不下单）
python3 ahr999_dca.py                   # 真实执行一次
python3 ahr999_dca.py --daemon          # 守护进程（按间隔自动执行）
python3 ahr999_dca.py --notify-daemon   # 守护进程（仅发邮件）
python3 ahr999_dca.py --history         # 查看全部执行历史
python3 ahr999_dca.py --history 2026-03 # 查看指定月份
```

### crontab 配置

```bash
crontab -e

# 每周一 09:00 发邮件（手动定投）
0 9 * * 1  cd /path/to/project && python3 ahr999_dca.py --notify >> notify.log 2>&1

# 每周一 09:00 自动交易
0 9 * * 1  cd /path/to/project && python3 ahr999_dca.py >> dca.log 2>&1
```

---

## 常见问题

**Q：Actions 显示绿色但没收到邮件？**

1. 检查 `SMTP_PASSWORD` 填的是授权码，不是登录密码
2. Gmail 需先开启两步验证才能生成应用专用密码
3. 在 Actions 日志中搜索 `email failed`，根据错误信息排查

**Q：Actions 报错 `all analysis failed`？**

通常是 CSV 时间格式混用问题。确认使用的是最新版 `ahr999.py`（已包含 `format='mixed'` 兼容修复）。

**Q：如何只收邮件、不自动交易？**

将 `auto_invest.yml` 中的运行命令改为 `python ahr999_dca.py --notify`，无需填写 OKX API Secrets。

**Q：MONTHLY 模式会超支吗？**

不会。系统每次从 `dca_log.json` 读取当月已消费金额，月度耗尽后自动跳过下单，邮件中显示「本次停止定投」。

**Q：GitHub Actions 免费额度够用吗？**

每次运行约 1~2 分钟，免费账户每月有 2000 分钟额度。每周执行一次 = 每月约 8 分钟，完全免费。

**Q：CSV 文件会不会越来越大？**

三个币种每年合计新增约 1.5MB，GitHub 完全可以承受。

---

## 文件结构

```
├── ahr999.py                    # AHR999 信号计算引擎
├── ahr999_dca.py                # 主程序（预算、交易、邮件）
├── requirements.txt             # Python 依赖
├── ahr999_backtest.html         # 历史回测报告（使用前请阅读）
├── btc_4h_data_2018_to_2025.csv # BTC 历史 K 线（每次运行自动更新）
├── eth_4h_data_2017_to_2025.csv # ETH 历史 K 线
├── sol_4h_data_2020_to_2025.csv # SOL 历史 K 线
├── dca_log.json                 # 执行记录（自动生成）
└── .github/workflows/
    └── auto_invest.yml          # GitHub Actions 工作流
```

---

## License

MIT License — 自由使用、修改、分发，请保留原始版权声明。

**再次提醒：本项目仅供学习研究，不构成投资建议。市场有风险，投资需谨慎。**
