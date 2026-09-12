# hydra-bootstrap — agent guide

> **Read first:** this repo is **documentation only** — no code, no CI, nothing that runs. Its value
> is that every document records *measured* state, so it must never drift into describing what
> someone assumed.
>
> PR titles are `[PET-xx] Title`. Never put AI attribution in commits or PR bodies, but **do**
> prefix every PR comment you write with `**<Agent> (<Model>)**`. See
> [Working agreements](#working-agreements).

## What this repo is

Runbooks and specifications for standing up the pieces Hydra runs on: hosts, the Cluster API
management cluster, and workload clusters.

| Document | What it is | Status |
|---|---|---|
| `docs/phase0-current-cluster.md` | The cluster as it **actually exists**, from live inspection. Both hosts. Includes a Drift table (repos vs reality) and a Corrections table. | **Authoritative** |
| `docs/capi-management-install.md` | How the existing cluster became the CAPI management cluster, and how to undo it (PET-5) | Done, executed |
| `docs/host-libvirt-prep.md` | Preparing the workstation as a libvirt hypervisor alongside bare metal (PET-31, ADR-003) | Done, executed |
| `docs/cluster-bootstrap.md` | How `hydra-wl0` was built end to end by Cluster API with **no manual `kubeadm` run anywhere** (PET-16, PET-17) | Done, executed |
| `docs/rebuild-control-plane.md` | Rebuild spec relocating the control plane to the workstation | **Obsolete — see below** |

### `phase0-current-cluster.md` outranks your own inspection

It is the verified state of both hosts, and several first-pass conclusions recorded in it turned out
wrong and were corrected in place. **Read it rather than re-inspecting, and trust it over anything
remembered.** If you do find it stale, correct the document in the same change — do not just work
around it, and add the correction to its Corrections table so the next reader knows the entry moved.

### `rebuild-control-plane.md` is obsolete — do not execute it

It exists to carry out **ADR-005**, which was **superseded by ADR-006** on 2026-08-26 and never
executed. Its own supersession note says the goal is reached "by the tool rather than by a manual
rebuild" — that tool is the provider, and the work landed under PET-16. The tracking issue (PET-35)
was **cancelled** for the same reason.

Following it would tear down the only cluster Hydra has ever been proven to work on. Leave it in
place as a record, keep the DRAFT / superseded banner intact, and do not resurrect it.

## How to write documents here

- **Record measured state, not intent.** Every value should be traceable to something observed.
  Where the repos and reality disagree, record reality and list the drift.
- **Say what was verified, when, and how** — these documents carry dated headers (`Performed:`,
  `Captured:`, `State verified:`) and target identifiers (hostname, IP, tailnet address, Kubernetes
  version). Keep that shape.
- **Link the Linear issue and any governing ADR** in the header block.
- **Be honest about status.** A document that has never been executed says so at the top, in bold.
  "Written but unrun" and "done, verified on hardware" are different claims and readers act on them
  differently.
- **Correct in place, and say you did.** These documents have been wrong before; a Corrections
  section is more useful than a clean-looking rewrite.
- **Runbooks are executed by a human at a terminal.** State for each step whether it needs root, and
  put the verification command *in* the step. `sudo` on the workstation is password-gated, so an
  agent cannot run those steps — write them for the person who can.
- **Never paste credentials, private keys, or kubeconfig contents** into a document.

## What belongs here versus Notion

This repo holds **runbooks and reproducible specifications** — things someone executes. Notion holds
**architecture, ADRs, design rationale, worklogs, and validation evidence**. A decision recorded only
here is in the wrong place; a runbook recorded only in Notion is too.

When a document here is completed or executed, record the outcome in Linear and on the linked Notion
page as well — see [Working agreement 4](#4-keep-the-record-current-continuously-not-at-the-end).

## Project context

**Hydra** is an open-source on-prem Kubernetes lifecycle platform built on
[Cluster API](https://cluster-api.sigs.k8s.io/), with libvirt/KVM as the first infrastructure
provider and workload-driven node autoscaling via Cluster Autoscaler. It lives on the
[`Petatron`](https://github.com/Petatron) GitHub org across five repositories:

| Repo | Role |
|---|---|
| [`hydra`](https://github.com/Petatron/hydra) | Umbrella / namesake repo. Effectively empty today. |
| [`cluster-api-provider-hydra`](https://github.com/Petatron/cluster-api-provider-hydra) | The Cluster API **infrastructure provider**. Go, kubebuilder. This is where product code lives. |
| [`hydra-bootstrap`](https://github.com/Petatron/hydra-bootstrap) | **Documentation only.** Runbooks for standing up hosts, the CAPI management cluster, and workload clusters. |
| [`hydra-gitops`](https://github.com/Petatron/hydra-gitops) | Argo CD configuration. **A merge to `main` changes live clusters.** |
| [`hydra-infra`](https://github.com/Petatron/hydra-infra) | Terraform for the pre-Hydra, hand-built worker VMs. Legacy path, being superseded by the provider. |

**Where truth lives — three systems, deliberately separated:**

- **Notion** (PETATRON teamspace → *Project Hydra*) — architecture, ADRs, design docs, runbooks,
  worklogs, validation evidence. This is the source of truth for **engineering knowledge**.
- **Linear** (workspace `petatron`, team key `PET`) — issues `PET-5`..`PET-46+`. Source of truth
  for **delivery status**. `PET-1`..`PET-4` are onboarding stubs, not real work.
- **GitHub** — code, PRs, CI. Not a place to record decisions.

Durable findings go in Notion, **not** Linear comments. Every Linear issue must have a matching
Notion page, linked both ways.

## Working agreements

These are instructions, not suggestions.

### 1. PR titles carry the ticket; commit messages do not

```
PR title:  [PET-27] Publish template capacity for scale-from-zero
Commit:    feat: publish template capacity for scale-from-zero
```

The ticket goes in **square brackets at the front** of the PR title. Commit messages stay in
conventional style (`feat:` / `fix:` / `docs:` / `chore:` / `ci:`) with **no** ticket prefix.
`feat: … (PET-27)` is the wrong shape for a PR title — do not copy it.

**Every PR needs a ticket. If there isn't one, stop and ask.** When the work has no Linear (or Jira)
issue, do not open the PR on your own judgment — ask whether to create a ticket first or to open
this one without a prefix, and let the user decide. **Never invent or guess a number:** a wrong
`[PET-xx]` silently attaches the PR to somebody else's work and corrupts the tracking both systems
exist to provide.

### 2. Never put AI attribution in git history

Commit messages, commit trailers, PR titles, and PR bodies must **never** mention Claude, Codex,
Cursor, Copilot, or any AI assistant. Git history records what changed and why, not which tool
typed it. Never emit:

```
Co-Authored-By: Claude <noreply@anthropic.com>
Co-authored-by: Cursor <cursoragent@cursor.com>
🤖 Generated with [Claude Code](https://claude.com/claude-code)
Made with [Cursor](https://cursor.com)
```

Nor phrases like "AI-assisted", "generated by AI", or a model name anywhere in a commit message or
PR description. Write them as the human author would.

**Some tools inject attribution after the fact — always verify.** Writing a clean message is not
enough:

```bash
ATTRIB='^Co-authored-by:|Generated with|Made with \[|AI-assisted|generated by AI|Claude Code|claude\.ai|cursor\.com|noreply@anthropic\.com'
git cat-file -p HEAD | grep -iE "$ATTRIB" && echo DIRTY || echo clean
gh pr view <n> --json body --jq '.body' | grep -iE "$ATTRIB" && echo DIRTY || echo clean
```

Match those patterns, **not** the bare words `claude` or `cursor` — `CLAUDE.md` is a legitimate
filename and a loose grep reports every commit that mentions it as dirty.

`git commit --amend` re-injects the trailer and cannot fix this; rebuild the commit object with
`git commit-tree` instead. If the bad commit was already pushed, **ask before force-pushing**, then
use `--force-with-lease=<branch>:<old-sha>`.

### 3. DO attribute yourself in PR comments

This is the exception to rule 2, and it is required. Multiple agents (and the human maintainer)
review the same PRs; the comment thread has to say who is speaking.

**Prefix every GitHub PR comment, review comment, review summary, and reply you write with your
agent name and model in bold:**

```
**Claude (Opus 5)** — the informer starts on the first cached Get, so declining
the Watch bought nothing here. Read the Secret through mgr.GetAPIReader().
```

```
**Codex (GPT-5)** — agreed, but the RBAC verb also needs narrowing to `get`.
```

Format is `**<Agent> (<Model>)**` followed by an em dash. Use the name a reader would recognise —
`Claude (Opus 5)`, `Codex (GPT-5)`, `Cursor (Composer)`. If you genuinely do not know your model
identity, use `**<Agent>**` alone rather than guessing.

This applies to **comments only** — the conversation surface. It does **not** apply to the PR body,
the PR title, or commit messages, which stay clean under rule 2.

GitHub still attributes the comment to whichever account is authenticated. The prefix says which
agent wrote the text; do **not** claim the displayed GitHub author has changed.

### 4. Keep the record current continuously, not at the end

Whenever a ticket, a test, or a meaningful piece of progress completes, update **both**:

- **Linear** — status, plus a comment stating what was proven and, just as importantly, what was
  **not**.
- **Notion** — the linked page's worklog section and its Doc Status property.

Write down what was *learned* — especially anything that turned out different from what was
assumed — not just what shipped. The goal is that a future session resumes with no lost context.

Every operation performed against real infrastructure gets recorded as it happens, in both.

### 5. Read PR review comments via GraphQL, never REST

The REST endpoint `/pulls/N/comments` **silently omits threads** — it missed all eight of the
maintainer's review comments on one PR. Always use `reviewThreads`:

```bash
gh api graphql -f query='{repository(owner:"Petatron",name:"<repo>"){pullRequest(number:N){
  reviewThreads(last:80){nodes{isResolved path line
    comments(first:1){nodes{author{login} createdAt body}}}}}}}'
```

Filter by `createdAt` to find new threads. Do not slice by index against the REST count.

### 6. Expect several review rounds, and verify every comment

Copilot reviews every PR and has found dozens of real defects — several that would only have
surfaced in production as orphaned VMs or wedged finalizers. **Expect 3–5 review rounds per PR.**

Verify each comment against the actual file before acting on it. Some have been stale test
assumptions rather than code bugs, and at least one was a linter firing on a conversion that was
genuinely required. A review comment is evidence, not a verdict.

### 7. Generic design beats lab convenience

The reference lab (an XPS 13 control plane at home, a workstation at the office, ~30 ms apart) is
one person's setup, **not a requirement Hydra should be shaped around**. Hydra must assume nothing
about that hardware, topology, or site count.

When lab convenience and generic design diverge, **generic wins** — unless the shortcut is provably
free, meaning a config value that can change later, not a code path that would have to be undone.

Architecture decisions carry a **Scope** property in Notion: `Product` binds Hydra for everyone,
`Reference lab` describes only that setup. Do not read a lab ADR as a product constraint.
