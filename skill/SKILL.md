---
name: gh
description: See GitHub reliably. (1) Fetch the COMPLETE picture of an issue/PR (body + all comments + timeline + linked PRs + reviews/CI). (2) Report what's new in a repo using snapshot memory (both new items and updates, nothing missed). Fixes "AI misses GitHub info and can't see that an issue is already claimed". Usage - /gh owner/repo#123, or /gh owner/repo news. Use when the user wants the full story of an issue/PR, asks "is anyone working on this?", or wants to know what's new in a project.
---

# gh — see GitHub reliably

Two capabilities: **fetch-complete** (dig one issue/PR to the bottom) and **memory** (snapshot diff to report changes). Verdicts and filtering are just processing layered on top — provide them only when asked.

## Capability 1: fetch-complete for a single issue/PR

Mandatory fetch checklist. Never skip a path because "it looks like enough":

```bash
# 1. body + ALL comments + assignees + state (for PRs also reviews/CI)
gh issue view N -R OWNER/REPO --json title,body,state,assignees,labels,createdAt,updatedAt,comments
# PR: gh pr view N --json ...,reviews,statusCheckRollup,mergeable

# 2. timeline — the data source everyone misses: cross-referenced (which PRs mention it), assigned (claim history)
gh api repos/OWNER/REPO/issues/N/timeline --paginate

# 3. PRs referencing the issue (catches what the timeline missed)
gh pr list -R OWNER/REPO --search "N in:body,title" --state all --json number,title,state,author,updatedAt

# 4. site-wide search as a backstop (forks, other repos; skippable for the user's own PR)
gh search prs "OWNER/REPO#N" --json number,title,state,repository
```

When a linked PR is found, **follow it into its comments** (why was it closed/merged, what did the maintainer say) — the verdict usually lives in the linked PR, not in the issue itself.

Output: everything in chronological order (comment key-quote excerpts, reviews, CI, assigns, cross-refs), each with a link, no subjective filtering. End with: one paragraph of background (what this issue/PR is about, where it's stuck) + "action needed from you" (say "none" explicitly if none) + a **fetch-coverage list** (which paths ran, how many items each; any failed path must be disclosed).

## Capability 2: what's new (snapshot memory, not just a timestamp)

State file `~/.claude/skills/gh/state.json`, grouped by repo, keyed by issue/PR number:

```json
{ "owner/repo": { "lastCheck": "ISO time",
    "items": { "1234": { "type": "pr", "title": "...", "state": "open",
      "comments": 3, "lastCommentAt": "...", "assignees": [], "reviewState": "...", "updatedAt": "..." } } } }
```

Flow:
1. Fetch current state: `gh issue list` + `gh pr list`, each with `--state all --json number,title,state,assignees,updatedAt,...`; for large repos narrow with `--search "updated:>lastCheck"` (first run with no watermark: take last 30 days and disclose that cutoff)
2. **Diff against the snapshot item by item**; report both kinds:
   - **New**: not in the snapshot → one-line introduction
   - **Updated**: in the snapshot but changed → say exactly what changed (how many new comments + key quotes, state flips open→closed/merged, assignee/review/CI changes). Never just say "updated".
3. Output grouped by repo → issue/PR, newest first, each with a link; **everything that moved inside the window gets listed, no filtering** — not missing things beats brevity
4. Write the current state back to state.json (update item fields + lastCheck)

When the user wants to go deeper on one item, run capability 1 on it.

## Capability 3: watchlist (explicit, self-cleaning — queries are one-shot, watching is opt-in)

Items in state.json may carry `"watch": true` (+ `"watchedAt"`, `"lastActivityAt"`). Rules:

- **In**: the user's own issues/PRs (author or assignee) are auto-watched; anything else ONLY when the user explicitly says to watch it (e.g. after a 🟢 verdict they say "I'll take this" / "盯着它"). Never auto-watch something merely because it was queried once.
- **Out (self-cleaning)**: when a watched item goes closed/merged, report that final change once, then remove the watch. If a watched item has had no activity for 30 days, ask once "still watching?" — no answer next run means remove.
- **Watch run** (triggered by "check my watchlist" / a scheduled job): run capability 2's snapshot diff over watched items only, PLUS capability 1's fetch on any watched item that moved (so the report has substance: who said what, what's needed from the user). **If nothing changed, output exactly "no updates" and nothing else** — silence is the default; only changes earn words.

## Capability 4: my open work, grouped by project ("盘点" / "what's blocked?")

Inventory everything the user has open across GitHub, with a blocker verdict per item:

```bash
gh search prs   --author=@me --state=open --json repository,number,title,updatedAt,isDraft
gh search issues --author=@me --state=open --json repository,number,title,updatedAt
# also include items where the user is assignee: --assignee=@me
```

Then for each PR run the capability-1 style view (`reviewDecision,statusCheckRollup,mergeable,reviews,comments`) and decide **whose court the ball is in**, by hard rules:

- CI failing, merge conflict, or reviewDecision CHANGES_REQUESTED → **blocked on the user** (say exactly what to do)
- last comment/review is from someone else and unanswered → **blocked on the user** (reply needed)
- user spoke last / review requested but none given → **blocked on them** (note how long; suggest a ping if >7 days)
- for issues: same last-speaker rule; no maintainer response at all → note the age

Output grouped by repo, two buckets first: **needs your action** (with the concrete next step) then **waiting on them** (with wait time). Every line links. End with the fetch-coverage list.

## On-demand processing (give what's asked, don't volunteer)

- **"Is anyone working on this / can I take it?"** → verdict 🟢🟡🔴. Hard rules — ANY of these makes it 🔴: has an assignee; timeline shows a linked PR that is open or was closed within 14 days; any claim-style comment in the last 30 days (working on / I'll take / let me); an open linked PR found via search. Claimed 45+ days ago with no follow-up = 🟡 (politely ask, then take over). Only if NONE apply is it 🟢. **Never output a claimability verdict for the user's own issue/PR.**
- **"Scan this repo for claimable issues"** → rough-filter the issue list by removing anything with an assignee, then run each candidate through the FULL capability-1 fetch before any verdict. Never conclude from the list view alone.

## Discipline

Every conclusion carries an evidence link; when unsure mark 🟡, never round up to 🟢; if a data path failed, say confidence is reduced; any truncation (e.g. last-30-days) must be disclosed.
