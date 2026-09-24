# comfy-workflow-landing

[English](README.md) | [繁體中文](README.zh-Hant.md) | [简体中文](README.zh-Hans.md)

A skill for **landing someone else's ComfyUI workflow on your own machine** — a
`workflow.json` from a cloud service, a Discord share or a tutorial, that will not run
as-is.

It is **a procedure, not a tool**. No scripts ship with it. It tells an agent what to
check, in what order, where the traps are, and — importantly — **which decisions it must
hand back to you rather than make alone**.

## Install

**Paste this to your agent:**

```
Read https://github.com/dseditor/comfy-workflow-landing and install that skill for me,
following the "For the agent reading this" section of the README.
```

### For the agent reading this

You are installing **a single markdown file**. No code runs, no keys are needed, nothing
is downloaded at runtime.

**1. Pick the target directory for the host you are in:**

| Host | Target directory |
|---|---|
| Claude Code | `~/.claude/skills/comfy-workflow-landing/` |
| Codex CLI | `~/.codex/skills/comfy-workflow-landing/` |
| Any other host with a filesystem | wherever that host loads skills from |

**2. Clone into it:**

```bash
git clone --depth 1 https://github.com/dseditor/comfy-workflow-landing.git <target>
```

If `git` is unavailable, download the zip and extract it so that **`SKILL.md` sits at
the top level** of `<target>`, not nested inside another folder.

**3. Verify — do not skip this:**

The host should now list a skill named `comfy-workflow-landing`. If it does not, check
that `<target>/SKILL.md` exists and that its YAML frontmatter (`name:` and
`description:`) is intact. **A skill with broken frontmatter loads as nothing, silently**
— there is no error to tell you it failed.

### No filesystem (web-based assistants)

Nothing to install. Start a project and drop `SKILL.md` into it.

---

## What actually happens after you hand over a workflow

Landing a workflow is **six phases**, and the first four are a survey. What happens next
depends entirely on what the survey finds:

```
SURVEY  (Phase 0-3)
  │
  ├─ everything present ────►  convert → verify → done.   NO questions asked
  │
  └─ something missing ─────►  ⛔️ that one decision comes to you, then it carries on
```

🚨 **The three decision points below are branches, not gates.** If every node is present,
every model is already on your machine and no assets are missing, **the agent should ask
you nothing at all** — it goes straight to verification and runs it. A workflow that
lands with zero questions is the best outcome, not a sign something was skipped.

So read the phases below as *"if this comes up"*, not *"this will happen"*.

### Phase 0 · Read the prompt  ·  *seconds*

The agent reads every text widget before touching anything. You get told:

- how many reference images are genuinely in use (a graph may wire nine slots and use three)
- what role each one plays — identity / scene / antagonist / style
- whether the prompt was written for **this** model, or carried over from another one
- whether the requested duration and shot count are achievable on this graph

📌 This is also where you find out that a 30-second prompt is going to be compressed
into whatever the graph actually produces — **before** you spend GPU time on it.

### Phase 1 · Node audit  ·  *a minute*

Every node type is checked against four layers: **built-in → installed package →
package behind upstream → genuinely absent.** You get a list sorted into those buckets,
with platform-locked nodes (`RH_*`, encrypted) flagged as **cannot be landed at all**.

> ⛔️ **Decision 1 of 3** — if a node needs the installed package to be *updated*, the
> agent stops. Updating can change results in workflows you already depend on, so that
> is not its call to make.

### Phase 2 · Find the models  ·  *minutes to hours, depending on download size*

For each missing model or LoRA: API search → web search → hand back. Everything
downloaded is verified by reading the `safetensors` header, not by file size.

> ⛔️ **Decision 2 of 3** — when both search routes fail, you get the filename, the
> nearest existing candidates, and what differs between them. Whether a one-suffix
> difference is a re-quantisation (must match) or just the author's naming (safe to
> swap) is a judgement you are better qualified to make.

### Phase 3 · Assets and dead wiring  ·  *a minute*

The author's uploaded images and videos arrive as hash filenames and **cannot be
recovered**. You supply substitutes — matched to the roles Phase 0 identified.

> ⛔️ **Decision 3 of 3** — cloud graphs usually carry unconnected nodes and empty
> loaders. The agent asks whether to remove them or mute and keep them, because they
> often show how the author intended the graph to scale.

### Phase 4 · Convert  ·  *seconds*

A landed copy of the JSON is written — nodes swapped, values repointed, substitutes
wired in, dead nodes muted. **Your original file is never modified.** You get a count of
every change made.

### Phase 5 · Verify in three layers  ·  *minutes, plus one generation*

```
① programmatic check   nodes, model filenames, asset files, numeric ranges
② load in a browser    the real UI reports its own missing nodes and empty dropdowns
③ press Queue          execution
```

Each layer finds what the previous one could not. Expect the agent to go back and fix
things twice here — that is the procedure working, not failing.

### Phase 6 · Acceptance  ·  *yours*

You get the output, plus a line-by-line comparison against the prompt **in counts, not
adjectives**: "the prompt names 5 actions; 3 are identifiable on screen."

Then the agent stops, because the remaining question is not one it can answer: **is this
the effect the workflow was built for?** Nothing in the JSON says what "good" looks like
here, and a metric picked to fill that gap will point the wrong way with full confidence.
It will ask you for a reference, or simply ask you to look.

---

### What you end up with

```
a landed workflow.json     runs on your machine, your original untouched
a list of substitutions    what is no longer the author's, and how it differs
the models and nodes       verified complete, with sources recorded
an honest report           including the parts the agent could not judge
```

### Roughly how long

For a 67-node workflow with a 15-LoRA chain, the audit and conversion took about
**twenty minutes of back-and-forth**; the bulk of the wall-clock time was a 19.5 GB base
model download and the generation itself. **The three decision points were the only
moments a human was needed.**

---

## Why the procedure is shaped this way

Every step below exists because skipping it produces a specific, recognisable failure.
If you only remember one thing: **most of these failures look like diligence.**

### Read the prompt before touching the graph

The prompt is the only specification you were handed. It states how many reference
images are genuinely used (a graph may wire nine slots and use three), what role each
one plays, and how long the result is supposed to be.

Substituting images without matching their roles does not produce a slightly-off video;
it produces an incoherent one, because the model will faithfully treat a portrait as
"the location".

### Built-in nodes first, always

ComfyUI now ships math, string, resolution, conditional and primitive nodes. Many
workflows still use third-party equivalents out of habit.

Swapping to a built-in **removes a dependency permanently**. Installing a package to
satisfy one trivial node **adds one forever** — and risks disturbing an environment that
currently works. The asymmetry is the whole argument.

### Check `display_name` before declaring a node missing

The same node is registered under different `class_type` across packages and versions
while displaying the same name in the UI. A workflow saying `"type": "Float"` may be
referring to something your install already has under another registration.

Reporting a built-in as missing is the single most common false alarm in this job.

### Search for models in three tiers, then stop and hand back

A hub's search API generally matches **repository names**, not file names. A LoRA
sitting inside someone's two-hundred-file grab-bag repo is invisible to it — while a
web search finds it immediately, because search engines crawl the file-listing pages.

And when both fail, the honest report is **"neither route found it"**, not "it does not
exist". The second sentence sounds like a fact about the world; it is only a fact about
your coverage — and it quietly removes the one person who might know where the file
lives.

### Verify downloads by header, not by size

`safetensors` records where its data should end. Comparing that against the actual file
size distinguishes "complete" from "truncated but large". A truncated file fails only at
load time, with an error that never mentions downloading.

### Ask before deleting dead wiring

Cloud graphs routinely carry unconnected nodes and empty loaders. They are not
necessarily junk — they often show how the author intended the graph to scale. Mute
them (`mode: 2`) rather than delete, and ask the user which they want.

### Swapping a node means four edits, not two

Links attach by **slot index**, so changing only `type` and `widgets_values` leaves a
graph that loads with no red boxes, submits successfully — and dies during execution,
because inputs are passed **by name**:

```
SomeNode.execute() got an unexpected keyword argument 'text2'
```

You must also rename the input slots and drop leftover unlinked ones.

### Verify in three layers

```
① Programmatic check    catches missing nodes and model filenames
② Load in a browser     catches what the checker's assumptions missed
③ Press Queue           catches what neither can see
```

These are not redundant. A checker passes clean; the browser then finds a cloud asset it
never looked for; and after that is fixed, execution still fails on a renamed slot.
Each layer's blind spot is the next layer's first finding.

### Separate "self-consistency" from "meeting the goal"

This is the distinction that matters most, and the easiest to collapse.

```
① The prompt asked for X — is X there?     ← an agent can check this
② Is this the effect the workflow is for?  ← only the user has this standard
```

Nothing in a JSON file says what "good" looks like here. **Do not fill that gap with a
metric.** Sharpness, motion magnitude and frame counts measure *something*, but the
thing the user cares about often has no metric — and a metric picked to fill the gap
will point the wrong way with full confidence.

Consider two outcomes that are identical to a measurement and opposite to a viewer:

```
structure falling apart (blur)   vs   structure holding under load (intensity)
```

Both raise every "detail" number you can compute. Only a person can say which one you
made. So the procedure ends by **asking the user for the acceptance standard** rather
than inventing one.

---

## What the report should end up looking like

```
✅ Running          what is in place, Queue completed, where the output is
📋 Against prompt   itemised, in counts not adjectives
⚠️ Substituted      what was swapped, why, how it differs — this is NOT the author's result
⛔️ Your call        decisions deliberately not made alone
❓ Beyond judgement  whether the effect was achieved — no standard available, please look
🚫 Cannot be landed  platform-locked nodes, unpublished weights — and why
```

## Licence

MIT. See `LICENSE`.
