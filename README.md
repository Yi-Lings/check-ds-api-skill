# DeepSeek API Usage Checker

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

通过浏览器自动化登录 [DeepSeek 开放平台](https://platform.deepseek.com/usage)，提取用量信息（余额、消费、各模型 Token 消耗、缓存命中率）并保存为结构化 Excel 报告。

> **本仓库是一个 Claude Agent Skill** — 供 [Claude Code](https://claude.ai/code) 等 AI 助手读取并执行的指令文件，而非传统命令行工具。AI 助手按照 `SKILL.md` 中的步骤驱动 Playwright 完成数据提取与报告生成。

---

## 功能

- 自动打开浏览器并登录 DeepSeek 开放平台（支持持久化 session）
- 拦截后端 API 请求，提取真实用量数据（UI 不展示输入/输出拆分和缓存命中率）
- 生成包含 **4 个工作表** 的 Excel 报告：
  | Sheet | 内容 |
  |-------|------|
  | **账户总览** | 充值余额、赠送余额、月消费、Token 总量 |
  | **模型用量** | 各模型请求数、缓存命中/未命中/输出 Tokens、缓存命中率 |
  | **消费明细** | 按模型拆分缓存命中/未命中/输出费用（CNY） |
  | **按日分布** | 当月逐日各模型用量明细 |

---

## 环境要求

| 依赖 | 版本 | 安装 |
|------|------|------|
| [playwright-cli](https://github.com/anthropics/claude-code/blob/main/docs/skills/playwright-cli.md) | 最新 | Claude Code 内置 |
| Python | ≥ 3.8 | [python.org](https://python.org) |
| openpyxl | ≥ 3.0 | `pip install openpyxl` |
| curl | — | Git Bash 内置 |

> **Windows 注意：** 必须使用 `python`（而非 `python3`）命令，Windows Store 的 `python3` 存在沙箱限制导致 exit code 49。

---

## 使用说明

### 方式一：AI 助手驱动（推荐）

在 Claude Code 中直接说：

```
查 DeepSeek 用量
看 DeepSeek 余额
查看 platform.deepseek.com 使用量
查 API 消耗
```

AI 助手会按 `SKILL.md` 的步骤自动执行完整流程。

### 方式二：手动执行

```bash
# 1. 打开浏览器（首次需手动扫码/登录）
playwright-cli open --headed --persistent https://platform.deepseek.com/usage

# 2. 确认页面已加载（观察余额和月消费概览）
playwright-cli snapshot --boxes

# 3. 查找后端 API 请求号
playwright-cli requests

# 4. 提取三个 API 响应（请求号以实际为准）
playwright-cli response-body 11 > responses/11.json
playwright-cli response-body 12 > responses/12.json
playwright-cli response-body 13 > responses/13.json

# 5. 生成 Excel 报告
python references/gen_report.py

# 6. 关闭浏览器
playwright-cli close
```

输出文件：`reports/ds-<YY-MM-DD-HH-MM>.xlsx`

---

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

| type | 含义 |
|------|------|
| `PROMPT_TOKEN` | Prompt Token 总数（通常=0） |
| `PROMPT_CACHE_HIT_TOKEN` | 输入缓存命中 Token 数 |
| `PROMPT_CACHE_MISS_TOKEN` | 输入缓存未命中 Token 数 |
| `RESPONSE_TOKEN` | 输出 Token 数 |
| `REQUEST` | API 请求次数 |

### `usage/cost` (Request #13)

与 `usage/amount` 结构相同，但值为金额（CNY）。

---

## 注意事项

### ⚠️ API 请求号不固定

每次 session 的请求 ID（如 11/12/13）可能不同。必须先用 `playwright-cli requests` 确认实际 ID。

### ⚠️ `page.eval(fetch())` 不携带认证

```javascript
// ❌ 会返回 "Missing Token"
await page.evaluate(async () => {
  const r = await fetch('/api/v0/usage/amount?month=7&year=2026');
  return await r.text();
});
```

必须用 `playwright-cli response-body <id>` 读取已存在的请求。

### ⚠️ Authorization Token 有时效

`requests` 中看到的 `authorization: Bearer xxx` 来自持久化 Cookie/Storage。Token 过期后重新打开浏览器手动登录即可。

### ⚠️ 时区与延迟

- 所有日期按 **UTC+0** 显示
- 数据可能有 **5 分钟延迟**
- 月份选择器中包含未来日期，但无数据的天填充为零

---

## 输出示例

![Excel Report Preview](docs/screenshot.png)

---

## License

[MIT](LICENSE)
