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
