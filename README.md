# issue-truth

**AI agents can't see that a GitHub issue is already claimed. Here's why — and a fix.**

A Claude Code skill that forces the agent to fetch the *complete* picture of a GitHub issue/PR (all comments, timeline, cross-referenced PRs, reviews, CI) before saying anything about it, and to attach evidence links to every conclusion.

## The failure mode

Ask an AI agent "is anyone working on this issue?" and it will typically:

1. Fetch the issue page (or run a bare `gh issue view`) — which returns the **body only**, no comments;
2. See zero assignees and zero discussion;
3. Confidently tell you the issue is up for grabs.

But "is this claimed" is almost never answered by the issue itself. The signal lives in three places agents don't look:

- **The timeline** — `cross-referenced` events record every PR that mentions the issue, and `assigned` events record the claim history. Neither appears in `gh issue view` or in scraped HTML.
- **Comments** — a buried "I'll take this" from three weeks ago.
- **Other PRs' bodies** — a `fixes #1141` in a PR you never opened.

### A real case

`omnigent-ai/omnigent#1141` ("Add a harness for Mistral-Vibe"): zero comments, no assignee. Every agent we tried called it a perfect first-PR candidate.

The timeline told a different story: PR **#1154** had already implemented it — and a maintainer had **closed** it with "extra harnesses should be built as plugins, please don't add them to the main repo." So not only was the issue taken; the entire category of contribution had been politely shut down. None of that was visible from the issue page.

If you'd started coding based on the agent's answer, you'd have wasted days on a PR that was pre-rejected by policy.

## The fix

Don't make the agent smarter — make skipping data impossible. The skill hard-codes a fetch checklist (all four must run, in parallel):

```bash
# 1. body + ALL comments + assignees + state
gh issue view N -R OWNER/REPO --json title,body,state,assignees,labels,createdAt,updatedAt,comments

# 2. timeline — the data source everyone misses
gh api repos/OWNER/REPO/issues/N/timeline --paginate

# 3. PRs referencing the issue
gh pr list -R OWNER/REPO --search "N in:body,title" --state all --json number,title,state,author,updatedAt

# 4. site-wide search (forks, other repos)
gh search prs "OWNER/REPO#N" --json number,title,state,repository
```

Plus three rules:

- **Follow cross-referenced PRs into their comments** — the verdict usually lives there (as in the case above), not in the issue.
- **Every conclusion carries an evidence link** (which comment, which event, when), so you can spot-check instead of trusting.
- **Claimability is decided by hard rules, not vibes**: any assignee, any open (or recently closed) linked PR, or any "working on this" comment in the last 30 days ⇒ not available. The report ends with a fetch-coverage list; any failed data path must be disclosed.

It also keeps a snapshot memory (`state.json`) per repo, so "anything new in project X?" diffs against the last-seen state and reports every change — new items *and* updates (new comments, state flips, review/CI changes) — with no subjective filtering, because missed updates are exactly the original complaint.

## Install

Copy the skill into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/gh
curl -o ~/.claude/skills/gh/SKILL.md https://raw.githubusercontent.com/tomsen02/issue-truth/main/skill/SKILL.md
```

Requires the [GitHub CLI](https://cli.github.com/) (`gh`) logged in.

## Use

```
/gh owner/repo#123          # full picture of one issue/PR, evidence-linked
/gh owner/repo 新消息        # what changed since last check (snapshot diff)
```

Or just ask naturally: "is anyone working on owner/repo#123?" — the claimability verdict (🟢/🟡/🔴) is applied on top of the same full fetch.

The skill's prompt is currently written in Chinese; the commands and output structure are language-independent, and Claude will respond in your language. PRs translating it are welcome.

## License

MIT
