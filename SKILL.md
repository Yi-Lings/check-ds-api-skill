---
name: check-ds-api
description: 通过 playwright-cli 登录 DeepSeek 开放平台，提取用量信息（余额、消费、各模型 Token 消耗、缓存命中率）并保存为 Excel 报告。
allowed-tools: Bash(playwright-cli:*) Bash(python:*) Bash(date:*) Bash(mkdir:*)
---

# Check DeepSeek API Usage

自动化提取 DeepSeek 开放平台用量信息（余额、消费、各模型 Token 明细、缓存命中率），保存为结构化 Excel 报告（4 个工作表）。

## Quick Start

```bash
# 1. 打开浏览器（需已登录）
playwright-cli open --headed --persistent https://platform.deepseek.com/usage

# 2. 获取页面数据确认已登录
playwright-cli snapshot --boxes

# 3. 查找 API 端点
playwright-cli requests

# 4. 提取 Token 用量明细
playwright-cli response-body 12

# 5. 提取消费金额明细
playwright-cli response-body 13

# 6. 提取账户余额
playwright-cli response-body 11

# 7. 生成 Excel 报告
python references/gen_report.py

# 8. 关闭浏览器
playwright-cli close
```

## 完整工作流

### 1. 打开浏览器并登录

```bash
playwright-cli open --headed --persistent https://platform.deepseek.com/usage
```

- `--headed`: 显示浏览器窗口（便于手动登录/确认验证码）
- `--persistent`: 使用持久化 Profile，下次复用登录状态
- 首次需要手动登录；后续重启 session 会自动恢复

### 2. 浏览页面确认数据

```bash
playwright-cli snapshot --boxes
```

观察页面确认显示了余额（充值余额）和月消费概览。

### 3. 发现后端 API 端点

UI 仅显示模型总 Token 数，不显示输入/输出拆分和缓存命中。需要通过请求列表找到真实 API：

```bash
playwright-cli requests
```

关键 API 端点（**注意请求号可能因 session 不同而变化，以实际为准**）：
| # | 端点 | 数据内容 |
|---|---|---|
| 11 | `/api/v0/users/get_user_summary` | 余额、Bonus、Token 估值 |
| 12 | `/api/v0/usage/amount?month=7&year=2026` | 各模型 Token 明细（含缓存命中/未命中） |
| 13 | `/api/v0/usage/cost?month=7&year=2026` | 各模型消费金额明细 |

### 4. 提取 API 响应

```bash
# 账户总览（余额、月消耗）
playwright-cli response-body 11 > responses/11.json

# Token 明细（含每日 breakdown）
playwright-cli response-body 12 > responses/12.json

# 消费金额明细（按模型拆分）
playwright-cli response-body 13 > responses/13.json
```

将三个 JSON 响应保存到 `responses/` 目录，供后续脚本读取。

### 5. 生成 Excel 报告

```bash
python references/gen_report.py
```

输出文件：`reports/ds-<日期时间戳>.xlsx`

包含 4 个工作表：

| Sheet | 内容 |
|-------|------|
| **账户总览** | 充值余额、赠送余额、月消费、Token 总量 |
| **模型用量** | 各模型请求数、缓存命中/未命中/输出 Tokens、缓存命中率 |
| **消费明细** | 按模型拆分缓存命中/未命中/输出费用（CNY） |
| **按日分布** | 当月逐日各模型用量明细 |

### 6. 关闭浏览器

```bash
playwright-cli close
```

## 数据字段说明

### `get_user_summary` (Request #11)

```
normal_wallets[].balance              — 充值余额 (CNY)
bonus_wallets[].balance               — 赠送余额 (CNY)
monthly_costs[].amount                — 当月总消费 (CNY)
monthly_token_usage                   — 当月总 Token 消耗
total_available_token_estimation      — 可用 Token 估值
```

### `usage/amount` (Request #12)

每个模型的 `usage[]` 数组：
| type | 含义 |
|---|---|
| `PROMPT_TOKEN` | Prompt Token 总数（通常=0，因为下面细分） |
| `PROMPT_CACHE_HIT_TOKEN` | 输入缓存命中 Token 数 |
| `PROMPT_CACHE_MISS_TOKEN` | 输入缓存未命中 Token 数 |
| `RESPONSE_TOKEN` | 输出 Token 数 |
| `REQUEST` | API 请求次数 |

`days[]` 包含当月逐日明细。

### `usage/cost` (Request #13)

与 `usage/amount` 结构相同，但值为金额（CNY）。

## 数据释义

```
总输入 Tokens = CACHE_HIT + CACHE_MISS
缓存命中率   = CACHE_HIT / (CACHE_HIT + CACHE_MISS) × 100%

模型总 Tokens = CACHE_HIT + CACHE_MISS + RESPONSE
                 (PROMPT 是 CACHE_HIT + CACHE_MISS 之和)
```

## Known Pitfalls

### ⚠️ 坑 1: UI 数据不完整

页面 Snapshot 只显示模型级别的 **总 Tokens** 和 **总请求数**，不显示：
- 输入/输出 Token 拆分
- 缓存命中/未命中
- 各模型的消费金额

**必须**通过 `requests` + `response-body` 从 API 获取。

### ⚠️ 坑 2: `page.eval(fetch())` 不携带认证

```js
// ❌ 会返回 "Missing Token"
await page.evaluate(async () => {
  const r = await fetch('/api/v0/usage/amount?month=7&year=2026');
  return await r.text();
});
```

`fetch()` 不会自动带上 Cookie 和 `authorization` header。**必须**用 `playwright-cli response-body <id>` 读取已存在的请求。

### ⚠️ 坑 3: 请求号不固定

每次 session 的请求 ID（11/12/13）可能不同。必须先用 `playwright-cli requests` 确认实际 ID。

### ⚠️ 坑 4: 模型名称不可展开

尝试 `click` 模型名称不会展开更多详情——这些是纯展示文本，非可交互元素。

### ⚠️ 坑 5: 时间与延迟

- 所有日期按 **UTC+0** 显示（不是本地时区）
- 数据可能有 **5 分钟延迟**
- 月份选择器中的选项包含未来日期，但无数据的天填充为零

### ⚠️ 坑 6: Authorization Token 时效

`requests` 中看到的 `authorization: Bearer xxx` 来自持久化 Cookie/Storage。如果 token 过期：
- **重新打开浏览器手动登录**
- 使用 `--persistent` 可减少登录频率
- 长期使用可通过 `state-save` / `state-load` 管理

### ⚠️ 坑 7: Month/Year 参数

```bash
MONTH=$(date "+%-m")   # Linux/Mac
MONTH=$(date "+%_m")   # 某些 Unix
YEAR=$(date "+%Y")
```

### ⚠️ 坑 8: Python 版本

必须用 `python`（非 `python3`），Windows Store 的 `python3` 有沙箱限制导致 exit code 49。

## 输出示例

见 [references/gen_report.py](references/gen_report.py) — 实际生成脚本。

## 触发

用户说"查 DeepSeek 用量"、"看 DeepSeek 余额"、"查看 platform.deepseek.com 使用量"、"查 API 消耗"、"检查 DS 用量"、"check_ds" 或任何与 DeepSeek 平台用量查询相关的请求。
