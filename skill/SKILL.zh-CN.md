---
name: gh
description: 可靠地看 GitHub:①把一个 issue/PR 的信息取全(正文+全部评论+timeline+关联PR+review/CI) ②基于快照记忆报某 repo 的新消息(新增+更新都报,不漏)。解决"AI看GitHub漏信息、看不到认领和关联PR"。用法:/gh owner/repo#123 或 /gh owner/repo 新消息。用户想看issue/PR完整情况、问"有人做吗"、想知道某项目有什么新动态时使用。
---

# gh — 可靠地看 GitHub

两个能力:**取全**(单个 issue/PR 挖到底)和**记忆**(快照对比报变化)。判定、筛选都只是在这两者之上的加工,按用户当下问什么来给。

## 能力一:取全单个 issue/PR

强制取数清单,不许因为"看起来够了"跳过任何一路:

```bash
# 1. 正文 + 全部评论 + assignees + 状态(PR 则再加 reviews/CI)
gh issue view N -R OWNER/REPO --json title,body,state,assignees,labels,createdAt,updatedAt,comments
# PR: gh pr view N --json ...,reviews,statusCheckRollup,mergeable

# 2. timeline —— 最容易被漏的数据源:cross-referenced(哪些PR提到它)、assigned(认领史)
gh api repos/OWNER/REPO/issues/N/timeline --paginate

# 3. 搜关联 PR(补捞 timeline 没记录的)
gh pr list -R OWNER/REPO --search "N in:body,title" --state all --json number,title,state,author,updatedAt

# 4. 全站搜索兜底(fork/其他 repo 的引用;查自己的 PR 时可省)
gh search prs "OWNER/REPO#N" --json number,title,state,repository
```

发现关联 PR 后**追进去看**(为什么关/合、maintainer 说了什么)——结论经常藏在关联 PR 的评论里,不在 issue 本身。

输出:按时间序全量呈现(评论关键句摘录、review、CI、assign、cross-ref),每条带链接,不做主观取舍;末尾:背景一段 + "需要你行动的"(无则明说)+「取数覆盖」清单(哪路跑了/多少条,失败必须明说)。

## 能力二:新消息(快照记忆,不只是时间戳)

状态文件 `~/.claude/skills/gh/state.json`,按 repo 分组、issue/PR 号为键:

```json
{ "owner/repo": { "lastCheck": "ISO时间",
    "items": { "1234": { "type": "pr", "title": "...", "state": "open",
      "comments": 3, "lastCommentAt": "...", "assignees": [], "reviewState": "...", "updatedAt": "..." } } } }
```

流程:
1. 取现状:`gh issue list` + `gh pr list` 各带 `--state all --json number,title,state,assignees,updatedAt,...`;若 repo 太大,用 `--search "updated:>lastCheck"` 缩范围(首次无水位则取近30天,并明说这个截断)
2. **和快照逐项对比**,分两类都要报:
   - **新增**:快照里没有的 → 一句话介绍
   - **更新**:快照里有但变了的 → 说清变了什么(新评论几条+关键句、state 变更 open→closed/merged、assignee/review/CI 变化),不许只说"有更新"
3. 输出按 repo → issue/PR 分组,时间倒序,每条带链接;水位内动过的**一律列出不做筛选**——防漏优先于简洁
4. 查完把现状写回 state.json(条目字段更新 + lastCheck)

用户对某条想深入时,直接走能力一取全。

## 能力三:盯梢名单(显式进入、自动清洁——查询是一次性的,盯梢要明说)

state.json 条目可带 `"watch": true`(+`"watchedAt"`、`"lastActivityAt"`)。规则:

- **进**:用户自己的 issue/PR(作者或 assignee)自动盯;其他条目**必须用户明说**才盯(比如 🟢 判定后说"我要做这个"/"盯着它")。绝不因为查过一次就自动订阅。
- **出(自清洁)**:盯梢条目 closed/merged 后报最后一次变化即移除;连续 30 天无动静的问一次"还盯吗",下次运行仍无答复即移除。
- **盯梢跑批**(用户说"看看盯梢"或定时任务触发):只对 watch 条目跑能力二快照对比,有动静的条目再走能力一取全(报告要有肉:谁说了什么、需要用户做什么)。**没有任何变化就只输出"无更新"三个字**——静默是默认态,有变化才配用字。

## 能力四:盘点我的开放项(按项目分组,判卡点)

把用户名下所有还开着的 PR/issue 扫出来,每条判定球在谁手里:

```bash
gh search prs   --author=@me --state=open --json repository,number,title,updatedAt,isDraft
gh search issues --author=@me --state=open --json repository,number,title,updatedAt
# 加上被 assign 的:--assignee=@me
```

每个 PR 再取 `reviewDecision,statusCheckRollup,mergeable,reviews,comments`,按硬规则判卡点:

- CI 挂 / 有冲突 / CHANGES_REQUESTED → **卡在你**(说清具体要做什么)
- 最后发言是对方且你没回 → **卡在你**(要回复)
- 你最后发言 / 等 review 没人给 → **卡在对方**(标注等了多久;>7天建议礼貌催)
- issue 同理看最后发言人;maintainer 从没回过的标注挂了多久

输出按 repo 分组,先「需要你行动」(带具体下一步)后「等对方」(带等待时长),每条带链接,末尾取数覆盖。

## 按需加工(用户问什么给什么,不问不给)

- **"有人做吗/能认领吗"** → 判定 🟢🟡🔴。硬规则,满足任一即 🔴:有 assignee;timeline 有 open 或近14天关闭的关联 PR;近30天评论有认领表述(working on/I'll take/我来/认领);搜到 open 关联 PR。45天+前认领无后续 = 🟡 可礼貌询问。全不满足才 🟢。**目标是用户自己的 issue/PR 时不出判定**。
- **"扫扫这个 repo 有什么可认领的"** → issue list 粗筛掉有 assignee 的,候选逐个走能力一完整取数再判定,不许只看列表下结论。

## 纪律

结论必须带证据链接;拿不准标灰不许判绿;数据没取到就说可信度下降;有截断(比如只取近30天)必须明说。
