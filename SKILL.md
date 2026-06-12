---
name: MY-CCS-weekly-report
description: MY CCS 团队周报生成器。自动抓取业绩数据，逐板块引导用户填写周报内容，生成含业绩数据表、6周趋势图的 HTML 周报，最终可发送飞书群。当用户提到"MY 周报"、"MY weekly report"、"MY CCS 周报"、"填周报"、"写周报"时，必须使用此 skill。
---

# MY CCS Weekly Report

开始时告知用户："开始填写 MY CCS 本周周报，共四个板块，我会逐一引导你完成。"

**脚本路径：** `C:/Users/irisding/.claude/skills/my-ccs-weekly-performance/scripts/`

## 成员列表

| staff_id | 姓名 |
|---|---|
| 16055 | zhenconglim |
| 14927 | jordanliow |
| 14488 | jackshenlee |
| 13331 | emelinlee |
| 13344 | mikilee |
| 13233 | alexfoong |
| 12543 | vaashini |

## API 信息

### 来源A — 跟进量数据

```
GET https://mycm.futuoa.com/api/visitor/overseas-statistics/marketing-work
  ?start_date=YYYYMMDD&end_date=YYYYMMDD
```

Headers：`futu-csrf-token`、`x-csrf-token`（均为 csrfToken 值）、`x-futu-client-lang: en`、`x-requested-with: XMLHttpRequest`、`cookie: {session_cookies}`

字段映射（使用 `calculate_date=0` 汇总行）：

| 报告指标 | API 字段 |
|---|---|
| 总跟进量 | `follow_count` |
| 总有效跟进量 | `effective_follow_count` |
| 总PC（外呼+WA去重）| `intervened_transferred_num` |

团队数据 = 7人 `calculate_date=0` 行加总（不用 `staff_id=0` 汇总行）。转化率指标均来自来源B。

### 来源B — 转化率数据（BI Dashboard）

认证：JWT Cookie (`uIdToken`) + Header (`user-id: aXJpc2Rpbmc=`, `x-dom-id: Z3VhbmJp`)
基础 URL：`https://us.data.futuoa.com/api/card/{card_id}/data?v={v_param}`
请求：POST，Body 含 filters（日期范围、地区=MY、周期粒度=周）

| card_name | card_id | v_param |
|---|---|---|
| `new_leads_personal` | `ra25fdcdd1f5643d49983698` | `pRquLBqVQUoFrxPnDBHsuTGs` |
| `stock_leads_team` | `o5c3a83379fd04d53a50b710` | `UDheTtHHfJiBNrmlLOTYDRpc` |
| `stock_leads_personal` | `bf6f882e04f994ccebf7b408` | `JuTCrXotUvldahcJLLrUEmGl` |

`new_leads_personal` 字段（`stock_leads_personal` 结构相同）：

| fmt_idx | 字段 |
|---|---|
| 1 | 有效跟进率 |
| 2 | 有效跟进转化率 |
| 4 | 分配转化率 |

⚠️ `stock_leads_personal` 日期字段类型为 STRING，脚本自动将 `YYYYMMDD` 转为 `YYYY-MM-DD`。

---

## 工作流程

### 板块一：业绩数据抓取

**Step 1：获取认证信息**（或从上下文读取）

- Cookie（含 `EGG_SESS`、`csrfToken`，来自 mycm.futuoa.com）
- uIdToken（来自 us.data.futuoa.com）
- 日期范围：默认上周 Mon~Sun

**Step 2：调用来源A**

```bash
PYTHONIOENCODING=utf-8 py C:/Users/irisding/.claude/skills/my-ccs-weekly-performance/scripts/fetch_data.py \
  --start {start_YYYYMMDD} --end {end_YYYYMMDD} \
  --cookies "EGG_SESS={egg_sess}; csrfToken={csrf_token}" \
  --csrf "{csrf_token}"
```

脚本自动获取本周与上周数据并计算 WoW 对比。

**Step 2b：Q2 累计 PC**

```bash
PYTHONIOENCODING=utf-8 py C:/Users/irisding/.claude/skills/my-ccs-weekly-performance/scripts/fetch_data.py \
  --start {季度起始_YYYYMMDD} --end {本周末_YYYYMMDD} \
  --cookies "EGG_SESS={egg_sess}; csrfToken={csrf_token}" \
  --csrf "{csrf_token}"
```

Q2 PC = `intervened_transferred_num`（7人 `calculate_date=0` 加总）。季度起始：Q1=0101，Q2=0401，Q3=0701，Q4=1001。

**Step 3：调用来源B（--all 模式）**

```bash
PYTHONIOENCODING=utf-8 py C:/Users/irisding/.claude/skills/my-ccs-weekly-performance/scripts/fetch_bi_data.py \
  --all --start {start_YYYY-MM-DD} --end {end_YYYY-MM-DD} \
  --uid-token "{uid_token}"
```

一次性抓取全部3个 card（含上周 WoW 对比）。

**Step 4：近6周历史数据**

⚠️ 严禁并行执行（会导致 SSL/TCP 超时 WinError 10060），必须顺序执行，每批最多3个用 `&&` 串联：

```bash
# 分批顺序抓 new_leads_personal：W-5→W-4→W-3（第一批），W-2→W-1（第二批）
# 再依同样方式抓 stock_leads_personal
PYTHONIOENCODING=utf-8 py C:/Users/irisding/.claude/skills/my-ccs-weekly-performance/scripts/fetch_bi_data.py \
  --card new_leads_personal --start {W-5_start} --end {W-5_end} --uid-token "{uid_token}" && \
PYTHONIOENCODING=utf-8 py C:/Users/irisding/.claude/skills/my-ccs-weekly-performance/scripts/fetch_bi_data.py \
  --card new_leads_personal --start {W-4_start} --end {W-4_end} --uid-token "{uid_token}" && \
PYTHONIOENCODING=utf-8 py C:/Users/irisding/.claude/skills/my-ccs-weekly-performance/scripts/fetch_bi_data.py \
  --card new_leads_personal --start {W-3_start} --end {W-3_end} --uid-token "{uid_token}"
```

完成后告知用户："板块一业绩数据已就绪，接下来填写团队运营内容。"

### 板块二：团队运营

依次询问三个子项（每次填完再问下一个）：

1. "📌 **二、团队运营 — 出勤情况**：本周出勤情况（如休假、出差、排班调整等）："
2. "📌 **二、团队运营 — 培训情况**：本周培训情况（培训主题、完成情况等）："
3. "📌 **二、团队运营 — 招聘进展**：本周招聘进展（如面试安排、offer 情况、岗位状态等）："

### 板块三：AI 应用

1. "📌 **三、AI 应用 — 使用情况**：本周 AI 工具使用情况（工具名称、使用场景、效果、开发的 Skill 等）："
2. "📌 **三、AI 应用 — 应用计划**：下阶段 AI 应用计划（待探索的工具、自动化方向等）："

### 板块四：下周计划

"📌 **四、下周计划**：请自由填写下周工作计划："

---

## 汇总：生成 HTML 报告

**文件路径：** `C:/Users/irisding/Downloads/my_ccs_weekly_report_{send_date}.html`

`{send_date}` = **报告生成当天日期**（YYYYMMDD），非周期开始日期也非下周一。

### HTML 报告结构

```
[Header]  MY CCS Weekly Report - Irisding | {日期范围} | Week {N}
[Compare] WoW 对比：总跟进量 / 总有效跟进量 / 总PC / Q2累计PC（共4张卡片）
[Table]   团队 & 个人业绩数据（3组列头）
[Chart]   团队近6周转化率趋势（6条折线）
[Section] 二、团队运营 / 三、AI 应用 / 四、下周计划
[Note]    数据来源说明
```

### 数据表格规范

**3组列头：**

| 组 | 列 |
|---|---|
| 跟进量（G1，colspan=4）| 总跟进量 → 有效跟进量 → 总PC（橙色 `.pc`）→ Q2 PC（紫色 `.q2pc`，`color:#7c3aed`）|
| 新 Leads 转化率（G2）| 有效跟进率 → **分配转化率** → 有效跟进转化率 |
| 存量 Leads 转化率（G3）| 有效跟进率 → **分配转化率** → 有效跟进转化率 |

**Q2 PC 规则：**
- Compare Strip 第4卡片：文字 `Q2 累计 PC`，副文字 `2026-04-01 ~ {MM-DD}`，颜色 `#7c3aed`，无 WoW 箭头
- 团队汇总行：Q2 PC 固定紫色，不参与高亮
- 个人行：最高 `#00c853`（绿）、最低 `#e53e3e`（红），其余紫色

**CSS 列区域：**
- G1（浅蓝）：`td:nth-child(2~5)`，分隔线在 `td:nth-child(5)`
- G2（浅绿）：`td:nth-child(6~8)`，分隔线在 `td:nth-child(8)`
- G3（浅橙）：`td:nth-child(9~11)`

**最高/最低高亮（文字颜色，无背景填充）：** 最高 `#00c853`，最低 `#e53e3e`

**表头黄色背景（仅 TH）：** 总PC `rgba(254,215,79,0.45)`，Q2 PC `rgba(254,215,79,0.38)`；数据行 `<td>` 不加黄色。

### 趋势图规范

6条折线（前3新Leads，后3存量Leads）：

| # | 系列 | 颜色 |
|---|---|---|
| 1 | 新 有效跟进率 | #1456F0 |
| 2 | 新 分配转化率 | #ef4444 |
| 3 | 新 有效跟进转化率 | #10b981 |
| 4 | 存量 有效跟进率 | #f59e0b |
| 5 | 存量 分配转化率 | #14b8a6 |
| 6 | 存量 有效跟进转化率 | #8b5cf6 |

节点标注数值（datalabels，font-size: 8.5px），底部图例可点击隐藏/显示。

---

## 发布 GitHub Pages + 发送飞书卡片（可选）

HTML 文件生成后，询问用户："HTML 周报已生成，是否同步推送到 GitHub Pages 并发送飞书通知？"

### 第一步：推送到 GitHub Pages

```bash
cp C:/Users/irisding/Downloads/my_ccs_weekly_report_{send_date}.html \
   C:/Users/irisding/MY-CCS-weekly-report/my_ccs_weekly_report_{send_date}.html

cd C:/Users/irisding/MY-CCS-weekly-report
git add my_ccs_weekly_report_{send_date}.html index.html
git commit -m "add weekly report {send_date}"
git push origin main
```

更新 index.html 重定向：`<meta http-equiv="refresh" content="0; url=my_ccs_weekly_report_{send_date}.html">`

**同步更新 history.html（必须执行）：** 在 `<div class="list">` 顶部插入新条目，加 `badge-latest`，移除上一条的 `badge-latest`：

```html
<a class="list-item" href="my_ccs_weekly_report_{send_date}.html">
  <div class="item-left">
    <div class="item-icon">📊</div>
    <div>
      <div class="item-title">{Mon DD} ~ {Mon DD}, 2026 <span class="badge-latest">最新</span></div>
      <div class="item-sub">Week {N} · 发送日 {YYYY-MM-DD}</div>
    </div>
  </div>
  <span class="item-arrow">›</span>
</a>
```

```bash
git add history.html && git commit -m "update history page for {send_date}" && git push origin main
```

GitHub Pages URL：`https://irisding001.github.io/MY-CCS-Weekly-report-/`

⚠️ 若 `git push` 失败（non-fast-forward），先 `git pull --rebase origin main` 再 push。

### 第二步：发送飞书交互卡片

```bash
lark-cli im +messages-send \
  --user-id ou_65c43706b75eeb31b763b24bd6b39d31 \
  --msg-type interactive \
  --content '{"config":{"wide_screen_mode":true},"header":{"title":{"tag":"plain_text","content":"📊 MY CCS Weekly Report 已更新 | {MM-DD} ~ {MM-DD}"},"template":"blue"},"elements":[{"tag":"action","actions":[{"tag":"button","text":{"tag":"plain_text","content":"点击查看完整报告"},"type":"primary","url":"https://irisding001.github.io/MY-CCS-Weekly-report-/my_ccs_weekly_report_{send_date}.html"},{"tag":"button","text":{"tag":"plain_text","content":"查看历史报告"},"type":"default","url":"https://irisding001.github.io/MY-CCS-Weekly-report-/history.html"}]}]}' \
  --as bot --profile cli_aa9ebc6861e55bc1
```

`open_id: ou_65c43706b75eeb31b763b24bd6b39d31`，`profile: cli_aa9ebc6861e55bc1`（MY CCS app）

⚠️ 必须用 v1 卡片格式（`config`+`header`+`elements`），不能用 schema v2（v2 不支持 `action` tag，报错 200861）。

---

## 注意事项

- Cookie 有过期时间：若 API 返回 401/403，提示用户从浏览器刷新 Cookie
- `uIdToken` 是 session-based，重新登录后可能失效
- Windows 优先用 `py`；需设置 `PYTHONIOENCODING=utf-8` 避免 emoji 乱码
- 来源A 日期格式：`YYYYMMDD`；来源B `--all` 模式：`YYYY-MM-DD`
- 历史数据某些行可能为 None（特殊周），parse 时需判断 `if cell is not None`
- 每次只问一个子项，用户可随时说"跳过"留空

## 启动时自动设置定时提醒

每次此 skill 被调用时，**必须**用 CronCreate 注册以下定时任务（若已存在可跳过）：

```
cron: "0 20 * * 0"（每周日 20:00），recurring: true
prompt: 使用 lark-cli 向 ou_65c43706b75eeb31b763b24bd6b39d31 发送私信：
  ⏰ 明天要生成 MY CCS 周报，请提前准备好：
  1. EGG_SESS + csrfToken（来自 mycm.futuoa.com）
  2. uIdToken（来自 us.data.futuoa.com）
```

该任务为 session-only，7天后自动过期，每次启动 skill 时重新注册。
