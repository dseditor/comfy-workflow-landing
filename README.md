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

Landing a workflow is **a survey, then at most one conversation, then a run.**

```
SURVEY  (Phase 0-3)     reads everything, changes nothing, downloads nothing
  |
  +-- nothing missing ------------------------->  convert & verify.  NO questions asked
  |
  +-- gaps found --->  ONE report, ONE decision (Phase 4)
                       every gap, what exists where, the options, the RISK
                       you pick what to act on - possibly none
                            |
                            +------------------>  convert & verify
```

**The survey is read-only and it asks you nothing.** It does not stop at the first
missing node, ask, then stop again at the first missing model. It finishes, then puts
everything in front of you at once — because seeing all the gaps together is what lets
you say "do those two, skip the rest". You cannot make that call one gap at a time.

**Nothing is downloaded or installed before you agree.** A 20 GB model does not start
transferring because an agent decided it would be helpful.

So read the phases below as *"if this comes up"*, not *"this will happen"*. **A workflow
that lands with zero questions asked is the normal good case.**

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

> 📋 Anything not simply present is **recorded and carried to Phase 4** — the survey does
> not stop to ask. The one exception it may act on alone is an exact built-in equivalent,
> because swapping a third-party `Float` for the built-in `Float` changes nothing you
> would want a say in.

### Phase 2 · Locate the models (find, do not fetch)  ·  *minutes*

For each missing model or LoRA: API search → web search → hand back. Everything
downloaded is verified by reading the `safetensors` header, not by file size.

> 📋 **Locating a file is survey work; fetching it is not.** Nothing downloads at this
> stage. You get the filename, the size, where it was found — or that neither route
> found it — and the nearest candidates, all in the Phase 4 report.

### Phase 3 · Assets and dead wiring  ·  *a minute*

The author's uploaded images and videos arrive as hash filenames and **cannot be
recovered**. You supply substitutes — matched to the roles Phase 0 identified.

> 📋 Cloud graphs usually carry unconnected nodes and empty loaders. They are listed,
> not deleted — they often show how the author intended the graph to scale. The
> recommendation will be to mute rather than remove, but it goes in the report.

### Phase 4 · One report, one decision  ·  *only if the survey found gaps*

Everything the survey found arrives at once, in one shape per gap:

```
what is missing      exact name, where it is referenced, what it does
is it out there      found at <source>, size N GB  /  neither route found it
what is here now      already local / behind by N commits / not present
options              A, B, C - with what each costs
risk to your setup   <- the column you actually decide on
```

**Risk is ranked by what it touches, not by how much work it is:**

| Risk | Touches | Examples |
|---|---|---|
| **None** | nothing existing | swap to a built-in - edit the workflow copy - mute dead nodes |
| **Low** | disk and time | download a model: adds files, changes nothing already running |
| **Medium** | adds a dependency | install a new node package: may conflict, needs maintaining |
| **HIGH** | **what already works** | update an installed package - update ComfyUI itself - replace a shared model |

🔴 That last row is the one to read carefully. If you have pipelines running on that
package today, an update can change their output without any error. The report should
say so, and say what would need re-checking afterwards.

You answer **selectively** — "download those two, skip the package update, I will find
the third myself" is a normal answer. So is **"do none of it"**: knowing a workflow needs
20 GB and a risky update is sometimes enough to decide it is not worth landing.

#### Reading it on a phone, or in a terminal

A survey report is four or five columns per gap. In a terminal it wraps into mush; on a
phone it is worse. So the report can also be written as **a single self-contained HTML
file** — no CDN, no network, opens offline, one card per gap, risk shown as both colour
and words.

The part that actually matters: it ends with a **Copy my decisions** button that turns
your selections into one line of text to paste back into the chat.

```
[ Copy my decisions ]  ->  "unet: download nearest (19.5 GB)
                            package: skip the update
                            lora_3: I will find it myself"
```

Without that button the file solves reading and leaves you typing a paragraph on a phone
keyboard. **The plain-text summary stays in the chat either way** — opening a file is
never required in order to answer.

### Phase 5 · Convert  ·  *seconds*

A landed copy of the JSON is written — nodes swapped, values repointed, substitutes
wired in, dead nodes muted. **Your original file is never modified.** You get a count of
every change made.

### Phase 5b · Verify in three layers  ·  *minutes, plus one generation*

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
