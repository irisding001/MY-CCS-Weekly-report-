---
name: MY-CCS-weekly-report
description: MY CCS 团队周报生成器。通过对话逐板块引导用户填写周报内容，生成 HTML 格式的完整周报（含业绩数据表、6周趋势图、文字板块），最终保存为 HTML 文件并可发送飞书群。当用户提到"MY 周报"、"MY weekly report"、"MY CCS 周报"、"填周报"、"写周报"时，必须使用此 skill。
---

# MY CCS Weekly Report

引导用户完成四个板块的周报填写，生成 HTML 格式周报文件，最终可发送飞书群消息。

## 工作流程

开始时告知用户："开始填写 MY CCS 本周周报，共四个板块，我会逐一引导你完成。"

### 板块一：团队业绩（自动抓取）

调用 `my-ccs-weekly-performance` skill 自动抓取并生成本周团队业绩数据：

1. 通过 Skill 工具调用 `my-ccs-weekly-performance`，完成：
   - 来源A：跟进量数据（EGG_SESS + csrfToken）
   - 来源B：BI Dashboard 转化率数据（uIdToken，使用 `--all` 模式）
   - 近6周历史趋势数据（用于 HTML 趋势图）
2. 保存所有数据（来源A + BI + 历史6周），用于后续生成 HTML。
3. 向用户确认："板块一业绩数据已就绪，接下来填写团队运营内容。"

### 板块二：团队运营

依次询问以下三个子项，每次填完再问下一个：

1. > "📌 **二、团队运营 — 出勤情况**
   > 本周出勤情况（如休假、出差、排班调整等）："

2. > "📌 **二、团队运营 — 培训情况**
   > 本周培训情况（培训主题、完成情况等）："

3. > "📌 **二、团队运营 — 招聘进展**
   > 本周招聘进展（如面试安排、offer 情况、岗位状态等）："

### 板块三：AI 应用

依次询问以下两个子项：

1. > "📌 **三、AI 应用 — AI 工具使用情况**
   > 本周 AI 工具使用情况（工具名称、使用场景、效果、开发的 Skill 等）："

2. > "📌 **三、AI 应用 — AI 应用计划**
   > 下阶段 AI 应用计划（待探索的工具、自动化方向等）："

### 板块四：下周计划

> "📌 **四、下周计划**
> 请自由填写下周工作计划："

---

## 汇总：生成 HTML 报告

四个板块填写完毕后，生成 HTML 格式周报文件：

**文件路径：** `C:/Users/irisding/Downloads/my_ccs_weekly_report_{send_date}.html`

> `{send_date}` 为**报告生成当天日期**（格式 YYYYMMDD），非周期开始日期也非下周一。例如报告周期 2026-05-25~05-31，生成日 2026-06-01，文件名为 `my_ccs_weekly_report_20260601.html`。

### HTML 报告结构

```
[Header]  MY CCS Weekly Report - Irisding | {日期范围} | Week {N}
[Compare] WoW 对比：总跟进量 / 总有效跟进量 / 总PC / Q2累计PC（共4张卡片）
[Table]   团队 & 个人业绩数据（3组列头，跟进量组含Q2 PC列）
[Chart]   团队近6周转化率趋势（6条折线，节点标注数值）
[Section] 二、团队运营（出勤情况 / 培训情况 / 招聘进展）
[Section] 三、AI 应用（使用情况 / 应用计划）
[Section] 四、下周计划
[Note]    数据来源说明
```

### 数据表格规范

**3组列头：**
| 组 | 列 |
|---|---|
| 跟进量（G1，colspan=4）| 总跟进量 → 有效跟进量 → 总PC（橙色 `.pc`）→ **Q2 PC（紫色 `.q2pc`，`color:#7c3aed`）** |
| 新 Leads 转化率（G2）| 有效跟进率 → **分配转化率** → 有效跟进转化率 |
| 存量 Leads 转化率（G3）| 有效跟进率 → **分配转化率** → 有效跟进转化率 |

**Q2 PC 说明：**
- 数据来源：来源A，额外调用 `fetch_data.py --start {季度起始_YYYYMMDD} --end {本周末_YYYYMMDD}`
- `intervened_transferred_num` 对应季度累计 PC（Q2 = 0401，Q3 = 0701，依此类推）
- Compare Strip 第4卡片文字：`Q2 累计 PC`，副文字：`2026-04-01 ~ {MM-DD}`（本周末），颜色 `#7c3aed`，无WoW箭头
- 团队汇总行：Q2 PC 固定紫色（`#7c3aed`），不参与高亮
- **个人行：各自填入季度累计 PC，并在个人行中做最高/最低高亮（最高 `#00c853` 绿，最低 `#e53e3e` 红），其余个人行保持紫色**

**CSS 列区域（Q2 PC列加入G1后）：**
- G1（浅蓝）：`td:nth-child(2~5)` → G1 分隔线在 `td:nth-child(5)`
- G2（浅绿）：`td:nth-child(6~8)` → G2 分隔线在 `td:nth-child(8)`
- G3（浅橙）：`td:nth-child(9~11)`

**最高/最低高亮（文字颜色，无背景填充）：**
- 最高值：`color: #00c853`（亮绿）
- 最低值：`color: #e53e3e`（红）

**表头黄色背景（⚠️ 仅 TH，不影响列数据底色）：**
- 总PC 列的 `<th>`：`style="background:rgba(254,215,79,0.45);"`
- Q2 PC 列的 `<th>`：`style="background:rgba(254,215,79,0.38);"`
- 数据行 `<td>` 不加任何黄色背景，G1 浅蓝底色保持不变

### 趋势图规范

6条折线，Dataset 顺序（前3新、后3存量）：
1. 新 有效跟进率（#1456F0 蓝）
2. 新 分配转化率（#ef4444 红）
3. 新 有效跟进转化率（#10b981 绿）
4. 存量 有效跟进率（#f59e0b 橙）
5. 存量 分配转化率（#14b8a6 青）
6. 存量 有效跟进转化率（#8b5cf6 紫）

- 节点标注数值（datalabels，font-size: 8.5px）
- 底部图例，可点击隐藏/显示

---

## 发布 GitHub Pages + 发送飞书卡片（可选）

HTML 文件生成后，询问用户："HTML 周报已生成，是否同步推送到 GitHub Pages 并发送飞书通知？"

若确认，执行以下两步：

### 第一步：推送到 GitHub Pages

```bash
# 复制文件到 repo
cp C:/Users/irisding/Downloads/my_ccs_weekly_report_{send_date}.html \
   C:/Users/irisding/MY-CCS-weekly-report/my_ccs_weekly_report_{send_date}.html

# 更新 index.html 重定向（指向最新报告）
# 内容：<meta http-equiv="refresh" content="0; url=my_ccs_weekly_report_{send_date}.html">

cd C:/Users/irisding/MY-CCS-weekly-report
git add my_ccs_weekly_report_{send_date}.html index.html
git commit -m "add weekly report {send_date}"
git push origin main
```

**同步更新 history.html（每次发布后必须执行）：**

在 `C:/Users/irisding/MY-CCS-weekly-report/history.html` 顶部 `<div class="list">` 内插入新条目（第一条，并加 `badge-latest`；同时移除上一条的 `badge-latest`）：

```html
<a class="list-item" href="my_ccs_weekly_report_{send_date}.html">
  <div class="item-left">
    <div class="item-icon">📊</div>
    <div>
      <div class="item-title">{Mon DD} ~ {Mon DD}, 2026 <span class="badge-latest">最新</span></div>
      <div class="item-sub">Week {N} &nbsp;·&nbsp; 发送日 {YYYY-MM-DD}</div>
    </div>
  </div>
  <span class="item-arrow">›</span>
</a>
```

```bash
cd C:/Users/irisding/MY-CCS-weekly-report
git add history.html
git commit -m "update history page for {send_date}"
git push origin main
```

**GitHub Pages 配置：**
- Repo 本地路径：`C:/Users/irisding/MY-CCS-weekly-report/`
- GitHub Pages URL：`https://irisding001.github.io/MY-CCS-Weekly-report-/`
- 推送后访问：`https://irisding001.github.io/MY-CCS-Weekly-report-/my_ccs_weekly_report_{send_date}.html`

> ⚠️ 若 `git push` 失败（non-fast-forward），先执行 `git pull --rebase origin main` 再 push。

### 第二步：发送飞书交互卡片

使用 `lark-im` skill（`--as bot --profile cli_aa9ebc6861e55bc1`）发送 interactive card：

```bash
lark-cli im +messages-send \
  --user-id ou_65c43706b75eeb31b763b24bd6b39d31 \
  --msg-type interactive \
  --content '{"config":{"wide_screen_mode":true},"header":{"title":{"tag":"plain_text","content":"📊 MY CCS Weekly Report 已更新 | {MM-DD} ~ {MM-DD}"},"template":"blue"},"elements":[{"tag":"action","actions":[{"tag":"button","text":{"tag":"plain_text","content":"点击查看完整报告"},"type":"primary","url":"https://irisding001.github.io/MY-CCS-Weekly-report-/my_ccs_weekly_report_{send_date}.html"},{"tag":"button","text":{"tag":"plain_text","content":"查看历史报告"},"type":"default","url":"https://irisding001.github.io/MY-CCS-Weekly-report-/history.html"}]}]}' \
  --as bot --profile cli_aa9ebc6861e55bc1
```

**关键参数：**
- `open_id`：`ou_65c43706b75eeb31b763b24bd6b39d31`（Irisding 本人）
- `profile`：`cli_aa9ebc6861e55bc1`（MY CCS app，注意不是 `my-ccs`）
- 卡片格式：**必须用 v1 格式**（含 `config`+`header`+`elements`），**不能用 schema v2**（v2 不支持 `action` tag，报错 200861）
- 按钮 URL 指向具体的带文件名的页面，不用 index.html 根路径

---

## 注意事项

- 每次只问一个子项，等用户回复后再继续。
- 用户可以随时说"跳过"跳过某个子项，留空处理。
- 日期范围默认使用上周 Mon~Sun（周一到周日）。
- HTML 文件参考 US CCS 格式（白底、蓝色渐变 Header、响应式布局，max-width: 1020px）。
- 此 skill 可被其他 skill 嵌入调用，调用时直接从板块一开始执行。

## 启动时自动设置定时提醒

每次此 skill 被调用时，**必须**用 CronCreate 注册以下定时任务（若已存在同类任务可跳过）：

```
每周日 20:00 发送飞书 Cookie 提醒：
- cron: "0 20 * * 0"
- recurring: true
- user-id: ou_65c43706b75eeb31b763b24bd6b39d31
- 消息内容：
  ⏰ 明天要生成 MY CCS 周报，请提前准备好以下认证信息：
  1. EGG_SESS + csrfToken（来自 mycm.futuoa.com）
  2. uIdToken（来自 us.data.futuoa.com）
  Cookie 有效后直接发给 Claude 开始生成即可。
```

> 该任务为 session-only，7天后自动过期，每次打开 skill 时重新注册以保持持续生效。
