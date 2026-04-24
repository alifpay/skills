# Backend Review Skills

A pack of Claude Code skills focused on reviewing backend services that move money — payment APIs, ledgers, transactional services, integrations, and the data model that holds them together.

Each skill is a focused checklist Claude loads when relevant code appears in a review. They are derived from real production incidents and hard-learned rules, not generic best-practice lists.

## Skills

| Skill | What it reviews |
|---|---|
| [`review-error-classification`](./review-error-classification/SKILL.md) | Error branches in transactional flows. Infra errors (Redis/RabbitMQ/Kafka/DB/HTTP/timeout) must not become final `FAILED` status; unknown outcomes must stay `PROCESSING`; Unknown sub-kinds (`null_status`, `empty_body`, `unparseable`, `unknown_enum`, `unknown_id`, `unexpected_code`, `signature_fail`) each get their own alert; responses to callers must match committed state. |
| [`review-external-api`](./review-external-api/SKILL.md) | Outbound calls to third-party / provider APIs. Idempotency key stability across retries (no random-suffix tricks), response-code classification, schema validation + contract-drift detection, refund/reversal guards, two-step auth+confirm amount validation, reconciliation, versioning. |
| [`review-sync-transactional`](./review-sync-transactional/SKILL.md) | Synchronous money-moving HTTP/gRPC endpoints. Context ownership, timeouts, client-disconnect handling, rate limiting, rollback endpoints, admin-route separation, **API versioning discipline (no breaking changes in the same version)**. |
| [`review-async-transactional`](./review-async-transactional/SKILL.md) | Message-broker-based transactional services (RabbitMQ, Kafka, JetStream). Outbox/Inbox patterns, Publisher Confirms, Consumer Ack ordering, DLQ, poison-message handling, message schema versioning. |
| [`review-transaction-model`](./review-transaction-model/SKILL.md) | Transaction/ledger data model. State machine, allowed transitions, **final-state immutability** (late callbacks must not overwrite final status), **persist before side effects**, timestamps (UTC, immutability), money in minor units, unique constraints, master-connection-only writes. |
| [`review-database-design`](./review-database-design/SKILL.md) | Read/write routing (master vs replica) and index coverage. No replica reads inside transactions; supporting indexes for every hot query; maintenance-window discipline. |
| [`review-api-testing`](./review-api-testing/SKILL.md) | Test and QA coverage for transactional APIs. Functional + security + load + integration + logging + regression. |

Each skill folder contains a single `SKILL.md` with the complete checklist.

## Prerequisites

- [Claude Code](https://claude.com/claude-code) installed (CLI, desktop app, or IDE extension)
- `git`
- A backend project you want to review (typically in the service's own git repo)

## Install

Claude Code discovers skills in two locations:

- **User-level** (`~/.claude/skills/`) — available in every project on your machine. Best for individual developers.
- **Project-level** (`<repo>/.claude/skills/`) — only available when Claude Code runs in that repo. Best for teams that want the same review rules enforced for everyone working on a specific service; commit `.claude/skills/` to the repo.

Pick one (or both).

### Option A — user-level, symlink (recommended for individuals)

Lets you `git pull` new rules and every skill in every project sees the update immediately.

```sh
git clone https://github.com/<you>/backend-review-skills.git ~/src/backend-review-skills
mkdir -p ~/.claude/skills
for d in ~/src/backend-review-skills/review-*; do
  ln -s "$d" ~/.claude/skills/
done
```

### Option B — project-level, committed (recommended for teams)

Ensures every developer working on the repo gets the same review bar, pinned to a specific revision.

```sh
cd <your-backend-repo>
mkdir -p .claude/skills
git clone https://github.com/<you>/backend-review-skills.git /tmp/backend-review-skills
cp -R /tmp/backend-review-skills/review-* .claude/skills/
git add .claude/skills
git commit -m "add backend review skills"
```

Or as a git submodule so updates pull cleanly:

```sh
cd <your-backend-repo>
git submodule add https://github.com/<you>/backend-review-skills.git .claude/skills-src
mkdir -p .claude/skills
for d in .claude/skills-src/review-*; do
  ln -s "../skills-src/$(basename "$d")" .claude/skills/
done
git add .gitmodules .claude
git commit -m "add backend review skills (submodule)"
```

### Option C — plain copy (throwaway / trying it out)

```sh
git clone https://github.com/<you>/backend-review-skills.git /tmp/backend-review-skills
cp -R /tmp/backend-review-skills/review-* ~/.claude/skills/
```

### Verify the install

Open Claude Code in any project and run:

```sh
ls ~/.claude/skills/    # or .claude/skills/ for project-level
```

You should see the seven `review-*` folders, each containing a `SKILL.md`. In Claude Code, start a new session — the skills will be listed in the available-skills section of the system context, and you can type `/review-` to auto-complete against them.

## How Claude uses these skills

Each `SKILL.md` has a `description:` field in its frontmatter. Claude reads the descriptions of every installed skill on every turn. When your prompt (or the code Claude is reading) matches a skill's description, Claude loads that skill's body and follows its checklist.

There are two ways to trigger a skill:

1. **Auto-trigger** — Claude picks up on keywords in your prompt or in the code. Example: if you ask for a review of a payment handler that imports a Redis client, `review-error-classification` will auto-load.
2. **Explicit invocation** — you name the skill in your prompt. Always works. Useful when you want a specific lens applied.

Both can be combined. Skills cross-reference each other (e.g. `review-external-api` defers to `review-error-classification` for error branches), so you typically want the whole pack installed and let them collaborate.

## Use — example workflows

### Review a whole branch before opening a PR

```
Review every changed file on this branch against all relevant review-* skills.
Report findings per file, grouped by skill, with severity (block / warning / nit).
End with a summary of all block-level findings.
```

Claude will diff the branch against your main branch, load the skills whose descriptions match the touched code, and produce a structured report.

### Review a specific PR (GitHub)

```
Review PR #123 in this repo against review-external-api, review-error-classification,
and review-transaction-model. Post-ready comments only (no nits).
```

With `gh` installed, Claude can fetch the diff via `gh pr view 123 --json files,...` and apply the skills.

### Focused review — only one concern

Useful when you already know the risk area. Examples:

```
Check only for final-state overwrite bugs in the callback handlers under internal/webhooks/.
Use review-transaction-model → Final-state immutability.
```

```
Audit every outbound call in this service for idempotency key stability across retries.
Use review-external-api → Idempotency key stability across retries.
```

```
Check all error-handling branches in the payment package for infrastructure errors
being misclassified as FAILED. Use review-error-classification.
```

### Review a single file

```
Review internal/payments/provider/acme.go using review-external-api.
```

### Pre-deploy sanity check

Before rolling out a version bump to an integrated external API:

```
Audit this branch for the version-bump checklist in review-external-api, and for
schema-drift detection in the response parser. Call out anything missing.
```

### Writing new code with a skill as a guide

The skills also work as authoring checklists, not only review checklists:

```
I need to add a new webhook handler for provider X. Follow review-transaction-model
(Final-state immutability) and review-external-api (Idempotency / refund guards)
as I write it. Draft the handler, then walk me through the checklist.
```

## What to expect from a review

Each skill produces findings in a consistent shape:

- **Location** — file + line, so you can jump straight to it
- **Rule** — which checklist item was violated
- **Why it matters** — the incident class it would cause (e.g. "double-charge on retry", "refund fired on parse error")
- **Suggested fix** — concrete code or behavior change

Skills are calibrated to catch money-loss bugs first. They are less strict on style, formatting, or architecture preferences — those belong in other tooling.

## Relationship to Claude Code's built-in `/review`

The built-in [`/review`](https://docs.claude.com/claude-code) skill is a general-purpose PR reviewer. These skills are additive:

- Use `/review` for broad coverage (typos, obvious bugs, style, general quality).
- Use `review-*` for the transactional-backend-specific rules that `/review` does not encode.

A good combined workflow:

```
Run /review on this PR first, then apply the review-* skills to the payment-handling files.
```

## Update

```sh
cd ~/src/backend-review-skills   # or .claude/skills-src in project-level install
git pull
```

Symlinked installs pick up the update immediately. Copied installs need to re-run the copy step.

For submodule-based project installs:

```sh
cd <repo>
git submodule update --remote --merge .claude/skills-src
git add .claude/skills-src && git commit -m "bump backend review skills"
```

## Uninstall

```sh
# User-level
rm -rf ~/.claude/skills/review-*

# Project-level
rm -rf .claude/skills/review-*   # or remove the symlinks / submodule
```

## Troubleshooting

**Skill does not auto-trigger** — check the `description:` line in the skill's `SKILL.md`. Claude matches against that text; if your code / prompt does not contain the keywords, invoke the skill explicitly by name.

**Skill triggers for irrelevant code** — the description is too broad. Edit the `description:` to narrow the trigger conditions, or invoke only the specific skills you want.

**Multiple skills report the same finding** — expected: the skills cross-reference each other on shared rules (persist-before-side-effects, idempotency, final-state immutability). Treat the duplicated finding as a strong signal.

**Skill not listed in the session** — Claude Code loads skills at session start. Restart the session (or open a new one) after installing.

**Project-level skill not picked up** — confirm Claude Code is running with `<repo>` as the working directory; project-level skills only load when `.claude/skills/` is in the current project tree.

## Source material

The checklists are distilled from an internal backend-engineering playbook plus a set of real production incidents:

- Infrastructure errors (Redis unavailable) misclassified as final `FAILED` → diverging state between services
- Silent provider-side schema change → parser threw → parse-error routed to refund path → customer refunded on a successful transaction
- Late callback overwrote a final status → refund fired on an already-approved transaction
- Retry path appended a random suffix to the transaction ID → double-charge on duplicate delivery

The skills are the actionable form of these lessons. New incidents → new rules, following the contributing guide below.

## Contributing

New skills welcome, especially when backed by a concrete incident. The highest-value rules are the ones that would have prevented a real production bug — not generic style advice.

A good skill:
- has an English `description:` with specific trigger keywords so Claude auto-loads it only when relevant
- leads with *why* (ideally a one-paragraph incident summary)
- lists red-flag patterns a reviewer can grep for
- lists what the code *should* do
- ends with a PR review checklist the reviewer can walk top-to-bottom

Submit changes as a PR against this repo. Include the incident (or class of incident) that motivated the rule — that is what keeps the pack sharp.
