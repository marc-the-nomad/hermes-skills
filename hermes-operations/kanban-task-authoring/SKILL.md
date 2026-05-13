---
name: kanban-task-authoring
description: "How task creators (Marc, Jude, Iris) write kanban tasks that workers can act on without round-trips. Title is a verb. Body has context, acceptance criteria, links, workspace, max-runtime. Bad tasks waste workers; good tasks compound the team's leverage."
version: 1.3.0
author: Marc Dion
license: MIT
metadata:
  hermes:
    tags: [kanban, task-authoring, leverage, jude, iris, marc-coo, quality]
    related_skills: [team-aup, kanban-orchestrator, kanban-worker, triage-specifier]
---

# Kanban Task Authoring

The dispatcher claims a task; it spawns a fresh worker session with no chat history. The worker sees ONLY what's in the task body. **The task body is the entire context.** A vague body produces vague output, no matter how good the worker is.

The discipline in one line: **write the task you'd want to receive at 2 AM with no other context.**

This skill is auto-loaded on the task creators (jude, iris). Every other agent reads it on demand if they're authoring tasks for fan-out or sub-task creation.

## When to Use This Skill

Loaded when authoring kanban tasks. The rules apply to:

- **Marc kicks off a task via Telegram or dashboard** → iris/jude shapes it
- **Iris fans out a multi-step request** into per-specialist children
- **Jude provisions a new agent** and creates the kickoff task
- **Specifier decomposes a triage card** (specifier loads `triage-specifier` instead, which references this skill)
- **Any orchestrator** creating child tasks via `kanban_create`

Don't load this for: reading or claiming tasks (workers); commenting on existing tasks; dashboard navigation.

## The Anatomy of a Good Task

A good task has 5 parts. In order:

### 1. Title (1 line, verb-first)

A title is a sentence the dispatcher logs and the dashboard shows in a list. It must answer "what does the assignee need to DO?" — verb first.

| Good | Bad | Why |
|---|---|---|
| "Audit the splunk app for stale detections" | "Splunk app" | "Splunk app" is a topic, not a task. |
| "Implement /healthz endpoint in mp-endpoint" | "Health endpoint thing" | "thing" is the tell. |
| "Review PR #482 in madping-security (security focus)" | "PR review" | Specify which PR + lens. |
| "Rotate sops master key (token leaked in journal)" | "Fix the sops thing" | Action + reason. |

The verb maps the title to a worker behavior. "Audit" / "implement" / "review" / "rotate" / "investigate" / "draft" / "summarize" each cue different work.

### 2. Body — Context (2-4 sentences)

What happened that led to this task? What's the current state? What does Marc / the upstream task want?

Context answers "why this work, why now." Without it, the worker can't make judgment calls and asks back via `kanban_block` — wasting a Marc-loop.

**Bad:** "Look into this."
**Good:** "Argus's hourly triage flagged an unusual notable pattern (5x spike in `process_creation_powershell_encoded` events between 2026-05-07 14:00-16:00 UTC). The spike correlates with a known maintenance window on Thor (kanban task `t_3a91`), but Argus didn't have visibility into the maintenance schedule. Verify whether the spike is benign maintenance noise or a real signal we should tune for."

### 3. Body — Acceptance Criteria (numbered, testable)

What must be true for the worker to call `kanban_complete`? Make it concrete and testable.

**Bad:** "Make it work."
**Good:**
```
1. Splunk SPL produces zero hits over the last 24h test data when the
   maintenance-window suppression filter is active.
2. PR opened in madping-security with the new savedsearch + tests.
3. Athena's review approval present on the PR before complete.
```

If a criterion is "ask Marc," that's a `kanban_block`, not an acceptance criterion.

### 4. Body — Links + workspace

- **Related kanban tasks:** `Parent: t_xyz`, `Sibling: t_abc`. The dispatcher uses `--parent` for promotion gating; pure prose links inform the worker.
- **External references:** PR URLs, Splunk dashboards, doc paths in `team-knowledge/`.
- **Workspace:** declare via the `--workspace` flag. Three kinds:
  - `scratch` — temp dir, discarded on complete
  - `dir:<absolute-path>` — shared dir, persists, multiple workers can touch
  - `worktree` — git worktree (isolation per task)

### 5. Body — Max-runtime

Every task has a runtime budget. It bounds how long a worker can hold a claim before the dispatcher reclaims it.

| Task shape | Suggested max-runtime |
|---|---|
| Triage / triage-specify | 5m |
| Quick research / fact-finding | 15m |
| Single-file code change | 30m |
| Multi-file refactor or PR review | 1h |
| Greenfield implementation | 2h |
| Long synthesis (research / report) | 4h |
| **NEVER unlimited.** | — |

Unlimited runtime breaks the dispatcher's crash-detection (TTL never expires → no reclaim possible).

## Concrete commands, not learning tasks

Workers execute. They don't study, internalize, read-and-summarize, or
"learn the codebase." If the task body sounds like homework, the worker
will either:

- Spend tokens generating a summary nobody asked for, or
- Hit the tool_loop_guardrails and exit without doing the actual work.

Audit/diagnostic/verify tasks must include concrete commands. Examples:

| Wrong (learning task) | Right (executable task) |
|---|---|
| "Read the team's incus-ops skill and internalize the rules" | "Run `incus list --format=csv \| awk -F, '$2!=\"RUNNING\"'` and report stopped containers" |
| "Study the kanban-model.md doc" | "Verify `dispatch_in_gateway: true` is set ONLY in iris's config: `for p in /var/lib/hermes/profiles/*/config.yaml; do grep -H dispatch_in_gateway $p; done`" |
| "Audit the team's setup" | "For each profile, confirm SOUL.md hash matches `agents-source/<profile>/SOUL.md`. Report drift." |
| "Check Splunk server status" | "Use the Splunk REST API (via JWT from env) to GET /services/server/info; parse server_name + version" |

If you can't write the concrete commands at authoring time, the task isn't
ready for a worker — escalate to specifier or kanban_block with the
specific question. Don't substitute a homework task for a real one.

When citing KB docs, use absolute paths the worker can read directly:
`/var/lib/hermes/team-knowledge/hermes-kb/<file>.md`. Reference them as
sources of invariants the worker should verify against, never as
required reading.

## Shared-state ordering — strict parent/child for any overlap

**Rule (load-bearing):** If two tasks touch the same shared state, they MUST be linked parent → child to serialize. There is no per-assignee concurrency cap in Hermes — ordering is expressed in the dependency graph, not by hoping the dispatcher will serialize for you.

The kanban graph IS the source of truth for execution order. Concurrent siblings are an explicit declaration that the work is independent. If two siblings collide on a shared file, the workers race, last-writer-wins, and at least one author's work is silently lost.

### What counts as "shared state"

Walk this enumeration before every `kanban_create`. If the task will touch one of these AND another in-flight or queued task touches the same specific target, link them.

| Category | Examples | Why concurrent edits collide |
|---|---|---|
| **Skill files** | `/var/lib/hermes/skills/<cat>/<name>/SKILL.md`<br>`/var/lib/hermes/profiles/<profile>/skills/<cat>/<name>/SKILL.md` | Last-write-wins on the file; one author's edits silently lost. |
| **Agent memory** | `/var/lib/hermes/profiles/<profile>/memories/MEMORY.md`<br>`/var/lib/hermes/agents-source/<profile>/{SOUL,AGENTS}.md` | Same: per-agent file, file-level race. |
| **Git repo branch (no worktree)** | `/var/lib/hermes/repos/madping-security`, `/var/lib/hermes/repos/mp-endpoint`, `team-knowledge/` (sophia) | Two workers checked out on `main` of the same repo race on file edits, race on `git pull --rebase`, race on `git push`; one push fails. |
| **Cron config** | `/var/lib/hermes/profiles/<profile>/cron/jobs.json` | Two workers calling `hermes cron edit` on overlapping job IDs corrupt the JSON. |
| **sops file** | `/etc/hermes-secrets/<profile>.yaml`, age keys, `.sops.yaml` recipient list | Two `sops` re-encrypts on the same file race; one's changes lost. |
| **Live gateway / cron tick** | `systemctl restart hermes-gateway-<profile>` | Two restarts of the same gateway = unpredictable timing; second restart can interrupt the first's init. |
| **Shared `dir:` workspace** | Multiple tasks pointing at the same `--workspace dir:/abs/path` | Direct file-edit race, same as branch case. |
| **External rate-limited API** | Splunk REST, ollama-cloud heavy model, GitHub API tokens | Not a data race but a budget race; N parallel workers consume N× rate, can 429 each other or burn cost. Chain when budget is tight. |
| **Kanban graph mutations** | Two orchestrators concurrently linking/unlinking/archiving the same set of tasks | TOCTOU on the link table. Hold one orchestration thread per fan-out. |

### The decision tree

For each new task you create, ask:

```
Does this task touch a shared-state category above?
│
├─ NO  → Parallel sibling is fine. No links needed (other than upstream gates).
│
└─ YES → Is there another non-archived (todo/ready/running/blocked) task that
         touches the SAME specific target?
         │
         ├─ NO  → Parallel sibling fine for now. WARNING: if THIS fan-out
         │       creates multiple siblings that ALL touch the same target,
         │       they collide with each other → CHAIN them.
         │
         └─ YES → CHAIN. Add `parents=[<existing_task_id>]` to the new task.
                  Dispatcher gates until the existing task reaches `done`.
```

### `parents=[]` semantics — what the link enforces

In Hermes, `parents=[A, B]` means: this task cannot be claimed until BOTH A and B are `status=done`. This is exactly the serialization mechanism for shared-state ordering. When parent A reaches `done`, the dispatcher's `recompute_ready` sweep promotes any child whose parents are all done from `todo` to `ready` on the next dispatch tick.

Cost: at most one dispatch interval of added latency per link (currently ~6 min on Vidar). Worth it to avoid corruption.

### Worktree exception (for git-only collisions)

If the only shared state is a git repo branch, `--workspace worktree` gives each task its own isolated working tree of the repo. Multiple workers can edit independently and merge later. **Use worktrees for parallel code work on the same repo.** They eliminate the file-edit race and let you parallelize most multi-PR fan-outs.

Worktrees do NOT help when:
- One task's commit must land before another can run (still CHAIN with parents=[...]).
- The shared state is non-git (skills, memory, cron, sops) — worktree only isolates the working tree, not arbitrary filesystem paths.

### Worked examples

**Independent fan-out — parallel safe:**

```python
# 6 morgan research topics. Each writes to its own per-task scratch.
# No shared skill updates, no shared repo edits, different topics.
research_ids = []
for topic in TOPICS:
    research_ids.append(kanban_create(
        title=f"Research: {topic}",
        assignee="morgan",
        workspace="scratch",        # per-task isolation
        max_runtime_seconds=3600,
        body=f"Research {topic}; write findings to $WORKSPACE/findings.md ..."
    ))
# Synthesis is the umbrella, gated by all 6.
synthesis = kanban_create(
    title="Synthesize morgan research into design",
    assignee="sophia",
    parents=research_ids,          # gated by all 6 done
    workspace="dir:/var/lib/hermes/team-knowledge",
)
```

**Code editing fan-out, same repo — use worktree (parallel safe):**

```python
for name, body in (("healthz", "..."), ("logger", "..."), ("readme", "...")):
    kanban_create(
        title=f"Add {name} change to mp-endpoint",
        assignee="atlas",
        workspace="worktree",       # isolated checkout per task
        body=body,
    )
# Three workers, three worktrees, three PRs — no race.
```

**Skill authoring fan-out, SAME skill file — CHAIN:**

```python
# Three sections to add to ONE skill file. Must serialize.
a = kanban_create(
    title="Add transient-retry section to incident-response skill",
    assignee="cadmus",
    workspace="dir:/var/lib/hermes/skills/hermes-operations/incident-response",
)
b = kanban_create(
    title="Add postmortem template to incident-response skill",
    assignee="cadmus",
    workspace="dir:/var/lib/hermes/skills/hermes-operations/incident-response",
    parents=[a],
)
c = kanban_create(
    title="Add rate-limit handling section to incident-response skill",
    assignee="cadmus",
    workspace="dir:/var/lib/hermes/skills/hermes-operations/incident-response",
    parents=[b],
)
```

**Memory updates across N agents — parallel safe:**

```python
# Each agent's memory is its OWN file. Different files, no collision.
for agent in ("argus", "cadmus", "theseus", "atlas"):
    kanban_create(
        title=f"{agent}: update memory with handoff lessons",
        assignee=agent,
        workspace="scratch",
        body="Append handoff lessons to $HERMES_HOME/memories/MEMORY.md ..."
    )
```

**Sequential pipeline — A's output feeds B (CHAIN even without file collision):**

```python
implement = kanban_create(
    title="Implement detection X in madping-security",
    assignee="cadmus",
    workspace="worktree",
)
review = kanban_create(
    title="Review PR for detection X",
    assignee="athena",
    parents=[implement],            # athena needs cadmus's PR to exist
)
deploy = kanban_create(
    title="Deploy detection X to Thor Splunk",
    assignee="janus",
    parents=[review],               # janus needs athena's approval
)
```

### Shared-state worksheet (run mentally before every fan-out)

For each pair of siblings (a, b) you're about to create:

| Check | If yes |
|---|---|
| Same skill file? | CHAIN |
| Same memory file (same assignee)? | CHAIN |
| Same repo branch AND not worktree? | CHAIN or switch to worktree |
| Same cron config / sops file? | CHAIN |
| Same gateway restart target? | CHAIN |
| Same `dir:` workspace? | CHAIN |
| Same external API at >2 concurrent? | CONSIDER chaining for budget |
| One depends on the other's output? | CHAIN regardless |

If every pair is "no" across the board → parallel siblings safe. Otherwise → enumerate which pairs need chaining and wire `parents=[...]` on the dependent ones.

## `kanban_create` — the canonical call

```python
kanban_create(
    title="Audit splunk app for stale detections",
    assignee="cadmus",
    body="""
**Context:** Cadmus shipped the madping-security savedsearches in March 2026.
Six months later, a half-dozen of them haven't fired in 90+ days. We need to
know which are legitimately stale (event sources changed) vs broken (SPL
references a dropped field).

**Acceptance:**
1. List every savedsearch with 0 hits in last 90d.
2. For each, classify as: stale-source / broken-spl / never-fired-correctly.
3. Open one PR per `broken-spl` finding with corrected SPL + test data.
4. Comment on this task with the classification table.

**Workspace:** `dir:/var/lib/hermes/repos/madping-security/`
**References:**
- Sibling task t_a91f (athena's tune-rate review)
- splunk-app-conventions skill
""",
    workspace="dir:/var/lib/hermes/repos/madping-security",
    max_runtime_seconds=3600,
    skills=["splunk-app-conventions"],
)
```

## Triage vs Ready

**Triage cards must have NO assignee.** Create with  and omit . The specifier (via iris-cron specifier-triage-poll) processes triage cards and routes them to the right executor. If you set  on a triage card, the dispatcher would claim it and spawn a specifier worker — specifier has no terminal/file/code tools, so the worker exits with rc=0 protocol violation and the work silently fails. Triage = unrouted; specifier = router; never confuse the two.




**Triage cards must have NO assignee.** Create with `--triage` and omit `--assignee`. The specifier (via iris-cron `specifier-triage-poll`) processes triage cards and routes them to the right executor. If you set `assignee=specifier` on a triage card, the dispatcher would claim it and spawn a specifier worker — specifier has no terminal/file/code tools, so the worker exits with `rc=0 protocol violation` and the work silently fails. Triage = unrouted; specifier = router; never confuse the two.

```bash
# CORRECT
hermes -p jude kanban create "<title>" --triage --body "<body>" --workspace dir:/path

# WRONG — never set assignee on a triage card
hermes -p jude kanban create "<title>" --triage --assignee specifier --body "<body>"
```

A task in `triage` is unclaimable — it sits there until specifier (or a human) decomposes / promotes it. Use triage for:

- Rough one-liners ("fix the logging thing") that need expansion
- Cross-specialist asks (specifier decides who gets what)
- "I'm not sure this is even a task yet" exploration

Use `ready` directly when:

- The task is concrete (verb + acceptance criteria + assignee known)
- You're orchestrating a fan-out and assigning per-child
- The work is small enough to skip the specifier step

## Worker-receivability checklist

Before calling `kanban_create`, ask:

- [ ] Title is a verb-first sentence
- [ ] Body's first paragraph explains why this task, why now
- [ ] Acceptance criteria are numbered + testable (no "make it work")
- [ ] Workspace is explicit (or `scratch` if truly disposable)
- [ ] Max-runtime is bounded
- [ ] Skills attached if a worker needs context not in their auto_load
- [ ] If parent/sibling links exist, named in body AND wired via `parents=[]`
- [ ] If the result feeds a downstream task, that task is created (or planned) with `parents=[<this_id>]`
- [ ] No secrets in body (use sops + env render; reference key names only)
- [ ] Shared-state analysis: walked the shared-state worksheet — any pair of new siblings touching the same skill/memory/repo-branch/sops/cron/`dir:` workspace is linked `parents=[...]` not parallel
- [ ] Body contains concrete commands (not "read X" / "internalize Y" / "study Z" instructions). If the task is audit/diagnostic, the commands are spelled out.

## Examples — good vs bad

### Bad (real, sanitized)

```
Title: "thor maintenance"
Body: "Janus, do the thor maintenance window stuff."
Assignee: janus
Workspace: scratch
Max-runtime: (unset)
```

Janus opens this and has to ask: *which* maintenance? When? What scope? Is this monthly or one-off? — `kanban_block` → Marc-loop → 30 minutes lost.

### Good (rewritten)

```
Title: "Run thor monthly maintenance window 2026-05-15 02:00-04:00 UTC"
Body:
  Marc scheduled the May maintenance window. Standard checklist applies
  (incus image refresh, kernel reboot if updates pending, backup verify,
  splunk index optimization).

  Acceptance:
  1. Pre-window: snapshot every container running on Thor.
  2. Run `apt update && apt upgrade && reboot` if kernel updates pending.
  3. Verify all 4 incus containers come back up healthy.
  4. Verify backup-vidar-to-thor.sh ran in the post-window hour.
  5. Comment on this task with the per-container before/after status.

  Workspace: scratch
  References: thor-audit skill, runbook in team-knowledge/docs/thor-monthly.md
Assignee: janus
Workspace: scratch
Max-runtime: 7200
Skills: [thor-audit]
```

Janus reads this once, executes, comments. No Marc-loop.

## Common Pitfalls

- **Vague titles.** "Splunk thing" / "Fix this" / "Look into X" — the worker has to translate before acting. Translate it for them at authoring time.
- **Body without acceptance criteria.** Worker doesn't know what "done" means; either over-delivers (waste) or `kanban_block`s asking what to deliver (Marc-loop).
- **Unlimited max-runtime.** Breaks crash-detection. Always bound it.
- **Workspace `scratch` for shared work.** If two child tasks need to share a dir, declare `dir:<absolute-path>` explicitly.
- **No parent links when there's a real dependency.** Worker B blocks waiting on A's output; without `parents=[A]`, the dispatcher promotes B too early.
- **Secrets in the body.** Token, key, value of an env var. Reference KEY NAMES ONLY; values come from `/run/hermes-secrets/`.
- **One mega-task instead of fan-out.** If a single task body has 8 acceptance criteria touching 4 specialists, it's actually 4 tasks. Decompose.
- **Title ≠ body intent.** "Review PR #482" with a body asking for a refactor proposal. Title commits to one verb; body should match.
- **Tasks for inanimate verbs.** "PR exists" isn't a task — that's an outcome. "Open PR for X" is the task.
- **Learning tasks for execution agents.** "Read and internalize the docs" is homework, not work.
- **Freelancing research instead of routing to morgan.** When a task needs external information (web search, paper, vendor doc, "what's the current best practice for X?"), the worker should NOT do the research themselves — they should `kanban_create(title="research <topic>", assignee="morgan", parents=[<this_task>])` and `kanban_block` until morgan completes. Morgan is the research lane; everyone else stays out. Trivial inline lookups against KB docs the agent already has access to (e.g. argus reading the Splunk-triage skill, theseus citing mp-endpoint-conventions) are NOT research — they're domain reference. If you'd write it as `kanban_create("Study X", ...)`, you actually want `kanban_create("Verify X by running <commands>", ...)`. Workers spawn fresh sessions; they don't have continuity to "remember what they read." Encode the verification in the body.
- **Setting an assignee on a triage card.** Never set `--assignee specifier` (or any assignee) on a `--triage` card. The specifier mechanism is iris's cron sweep — it doesn't rely on assignment. Setting assignee=specifier causes the dispatcher to claim the triage card and spawn a specifier worker, which has no tools and exits silently. Triage cards that have been pre-assigned have also been observed getting archived prematurely by cleanup sweeps. Create triage cards with `--triage` and omit `--assignee` entirely; the specifier will route them on the next 15-minute poll.

- **Parallel siblings touching shared state.** Two tasks at status `ready` with the same skill file / memory / repo branch / sops file / cron / shared `dir:` workspace as their target. The dispatcher will claim them concurrently and both workers will race. Last writer wins; the other's work is silently lost. Walk the shared-state worksheet (above) for every fan-out, and add `parents=[...]` whenever any pair overlaps. The kanban graph — not a runtime concurrency cap — is how Hermes serializes shared-state work.

## Handoffs between agents — YOU create the next task

**Rule (Marc, 2026-05-12):** if your task's deliverable produces work for another agent, **you create the follow-up task yourself** via `hermes -p <your-profile> kanban create ...`, parented on your task. You do NOT email iris, you do NOT comment "iris please fan out", you do NOT leave it implicit in your handoff doc and hope someone picks it up.

The kanban IS the coordination channel. Anything else is a side channel that breaks the audit trail.

### Why this rule exists

The pattern that triggered the rule (2026-05-12): Theseus was self-blocked with reason "PR #14 needs Athena architectural review before merge." That's a legitimate block — but he didn't create an Athena review task. So the PR sat indefinitely with nobody assigned to actually do the review. Iris caught the gap manually and created the Athena task afterwards.

Every blocked-waiting-on-another-agent task that doesn't have a corresponding **assigned, ready, parent-linked** task for that other agent is a dropped baton. Same applies to:

- "Argus should verify the parser is consuming events" → create the argus task, don't just mention it in your completion comment
- "This change requires a Cadmus props.conf update" → create the cadmus task before you mark yours done
- "Janus needs to restart the service on the fleet" → create the janus task, parent it on yours
- "Marc needs to merge this PR" → that one IS the exception: Marc gets a passive Discord/Telegram page; you don't create a task assigned to Marc (he has no kanban queue).

### How to do the handoff right

1. **Identify the follow-up.** As you near completion, ask: "is the natural next step on someone else's plate?"
2. **Create the task.** Use `hermes -p <your-profile> kanban create` directly. Don't escalate to iris. The CLI is available to every gateway-running profile.
3. **Parent-link it.** `--parent <your-task-id>` means the follow-up only promotes when your task completes.
4. **Body has full context.** The receiving agent has zero history with your work. Write the task body so it stands alone (deliverable, acceptance criteria, links to artifacts you produced, references to commits/PRs).
5. **Comment on your own task** noting which follow-up you created and assigned to whom. That's the audit trail.

### Concrete example (canonical pattern, from 2026-05-11 Argus → Janus chain)

Argus's task said: "Verify nebula:status sourcetype is actually flowing after stats-block removal." Acceptance criteria included: "If count = 0 across all hosts: flag whether to (a) re-enable stats with `type: prometheus` properly (Janus task), (b) keep parser dormant, or (c) something else. If (a): follow-up task created and linked."

When Argus found zero events, he created `[Janus] Restore Nebula stats block on asgard + midgard with type: prometheus` (t_52c39e2a), parent-linked it to his own task, and commented on his task with the new task ID. Janus's gateway picked it up automatically. No iris involvement, no Marc round-trip.

That is the pattern. The skill's "Worker-receivability checklist" already covers this implicitly — but as of 2026-05-12 it is now an **explicit rule**, not an implicit best practice.

### Exception: when iris/jude IS the right author

There are 3 cases where iris or jude (orchestrator profiles) creates the task, not you:

- **The follow-up requires Marc input** (a triage card for a decision). File as a triage card via `--triage --author <you>`; specifier-triage-poll routes it.
- **The follow-up requires cross-agent coordination you can't see** (e.g., you don't know whether Cadmus or Athena should review a Splunk-app change). Open a triage card and let specifier resolve.
- **The follow-up is project-scope (new phase, large milestone)**. Jude scopes phase boundaries.

For anything that's "X happened, Y needs to do Z next" — **you create the Y task, not iris**.

### Anti-pattern (do NOT do)

Bad:
```
Comment on your completed task: "Argus needs to verify the parser now."
(then mark task done)
```
This leaves Argus with no signal. He doesn't poll your completion comments.

Bad:
```
Comment on iris's last task: "@iris please ask Janus to restart nebula"
```
You can ask iris directly via her CLI — but a comment isn't a request. And anyway, iris doesn't need to broker this. You can create the janus task yourself with one command.

Good:
```bash
# In your worker session, before completing your task:
hermes -p <your-profile> kanban create "[Argus] Verify nebula:status flow post-stats-restore" \
  --body "..." \
  --assignee argus \
  --parent <your-task-id> \
  --idempotency-key <yours>-followup-argus-verify \
  --max-runtime 1h
```


## Review tasks are SIBLINGS, not children (v1.3)

**This is the single most common deadlock pattern in the kanban graph.** When you finish work and need a reviewer, the review task is a SIBLING of your task — NOT a child. A child task cannot be claimed until the parent reaches `done`, but your task can't reach `done` until the review completes. That's a deadlock.

### The deadlock (what NOT to do)

```
# ❌ DEADLOCKS: parent cycle
# Theseus self-blocks on t_b39d82a4 for PR review, creates:
kanban_create("Review PR #17" --assignee athena --parent t_b39d82a4)
# → athena's task stuck in 'todo': parent t_b39d82a4 is still 'running'
# → t_b39d82a4 is stuck 'blocked': waiting on review
# → Nobody can move → Iris has to `kanban unlink t_b39d82a4 t_5be16953`
```

**Observed deadlocks (3 in 48 hours):**
- 2026-05-12 t_5be16953 parented on t_b39d82a4 (Theseus PR #17)
- 2026-05-13 t_bbac2e00 parented on t_61481f55 (Theseus PR #23)
- 2026-05-13 t_34f9e571 (Theseus PR #22, almost — caught in task body)

### The correct pattern

```
# ✅ SIBLING: runs immediately
kanban_create("Review PR #N" --assignee athena)
# In your block reason, reference the review task:
kanban_block("review-required: PR #N (athena task t_xxxx)")
```

The mental model: **your task waits on the reviewer; the reviewer's task does NOT wait on you.** The dependency runs the OTHER way — expressed via the block reason that iris reads, not via `parents=[]`.

### When parent IS correct (legitimate children)

Parent-linking is still correct for:
- **Output dependency:** Janus needs Cadmus's PR to exist before deploying (`implement → review → deploy` chain)
- **Shared-state ordering:** Two tasks touching the same skill/memory/sops file
- **Pipeline stages:** Morgan's research feeds Sophia's synthesis

The rule: parent-link when the CHILD's work literally cannot start until the PARENT's output exists. Review tasks CAN start the moment the PR is open — they just need the branch URL, not the parent's completion event.

### Exception — genuine pre-review gates

If your work has a true pre-condition gate (e.g. "athena must review the architecture doc BEFORE I start coding"), that IS a legitimate `parents=[review_task]` — you cannot start until the review finishes. This is rare. Most review tasks follow the sibling pattern.

## How This Composes With Other Skills

- **`team-aup` rule 22 (forward-reference dependencies)** — task chains use `kanban_link` or `parents=[]`, not buried prose.
- **`triage-specifier`** — when a task lands in triage, specifier consumes this skill's rules in reverse: takes a vague body and adds context + acceptance + workspace.
- **`kanban-orchestrator`** (bundled) — fan-out patterns; this skill's rules apply per child task.
- **`kanban-worker`** (bundled, auto-injected) — tells workers how to read tasks; this skill tells authors how to write them. Two halves of the same protocol.

---

*The task body is the worker's whole world. Write it like a memo to a smart stranger arriving at 2 AM. Specificity at authoring time saves N round-trips at execution time.*
