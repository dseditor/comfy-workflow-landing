---
name: comfy-workflow-landing
description: Land someone else's ComfyUI workflow on a local machine — a cloud or shared workflow.json that will not run as-is. Use when given a workflow JSON to "make it work here", when a workflow shows red nodes or empty dropdowns, when auditing which models and LoRAs a workflow needs, or when a workflow loads cleanly but errors the moment Queue is pressed. This is a procedure rather than a tool, and no scripts ship with it.
---

# Landing a Workflow

Someone hands you a `workflow.json`. It ran on their machine, or on a cloud service.
Your job is to make it run here — and to be honest about what "run here" can mean.

```
The pipeline running  ≠  reproducing their result
```

Their reference images, their uploaded video, their private fine-tune — some of that is
**unrecoverable**. Say so early. A run that completes with substituted inputs is a
different piece of work, and the user should know that before they judge the output.

---

## Report what it is. Do not rule on whether it is worth doing.

The user asked what this workflow is and what it needs. That is the deliverable.

```
YOURS      what the graph does, what it needs, what is missing, what each fix touches
NOT YOURS  whether it is worth landing
```

**"I would not bother with this one" is not a finding, it is an opinion** — and it rests
on knowledge you do not have. Whether a workflow is worth the trouble depends on what
the user already knows about these techniques, what they are trying to learn, whether
the author is someone they follow, and what else they have running. A user with domain
experience may take one look at your inventory and dismiss it in a second, for reasons
that never appear anywhere in the JSON.

So state the cost plainly — packages, downloads, unrecoverable assets — and stop there.
Listing "do none of it" among the options is right. **Recommending it is not.**

📌 The same rule as the acceptance standard in Phase 6, arriving earlier: you can measure
what something costs; you cannot measure what it is worth to them.

---

## The shape of the job: survey read-only, then one decision

```
SURVEY   Phase 0-3    read the prompt / audit nodes / locate models / check assets
                      change nothing, download nothing, ask nothing yet
  |
  +-- nothing missing ------------------------------->  Phase 5 convert & verify
  |                                                      ZERO questions asked
  |
  +-- gaps found --->  Phase 4  ONE report, ONE decision
                       every gap, what exists where, the options, and the RISK
                       the user picks which to act on - possibly none
                            |
                            +------------------------->  Phase 5 convert & verify
```

**Survey completely before asking anything.** Do not stop at the first missing node to
ask, then stop again at the first missing model. That interrupts the user three times
and each time they see only a fragment. Finish the whole survey, then present everything
at once: seeing all the gaps together, a user may act on two and drop the rest, or
abandon the workflow entirely. They cannot make that judgement one gap at a time.

**The survey is read-only.** No downloads, no installs, no edits to the JSON until the
user has decided. Downloading 20 GB before asking is not helpfulness; it removes their
choice.

**Do not manufacture decisions.** A workflow that lands with zero questions is the best
outcome, not a sign you skipped something. Ask only about gaps the survey actually found.

---

## Phase 0 — Read the prompt first. It is the spec.

**Before touching nodes, read every text widget in the graph.** The prompt tells you
things the graph cannot:

- **How many reference images are actually used.** A graph may wire nine `LoadImage`
  slots while the prompt defines only three roles. The prompt wins.
- **What each image is supposed to be** — identity, scene, antagonist, style. If you
  substitute images without matching these roles, the output is incoherent: it will
  treat a random portrait as "the location".
- **Whether the prompt was even written for this model.** Prompts get carried across
  when a graph is ported. Look for another model's name or house format in the text.
  Then check whether any node rewrites it — **often nothing does**, because the author
  moved the graph and left the prompt layer behind.
- **Whether what the prompt asks for is achievable on this graph.** Compare its stated
  duration, shot count, subject count and aspect ratio against what the graph resolves
  to. A prompt asking for twice the runtime the graph produces will not be split across
  time; it will be compressed into what there is.

🚨 **Widget values on inputs that have a link are stale.** They are what was typed
before the wire was attached; the live value arrives down the wire. Read the link, not
the widget — this is the easiest way to report the wrong duration, the wrong prompt,
the wrong everything.

---

## Phase 1 — Nodes: four layers, in this order

For every node type in the graph, ask in order:

```
① Is it built in?            comfy_extras/ and nodes.py   →  ALWAYS PREFER THIS
② Is it in an installed package?   custom_nodes/
③ Is the installed package behind? the node exists upstream but not in this commit
④ Genuinely absent           needs a new package, or a structural workaround
```

### 🔑 Built-in first. This is the standing rule.

Math, string, resolution, conditional, primitive — **ComfyUI now ships all of these**.
Many workflows still use third-party equivalents out of habit. Replace them:

```
Float / FloatConstant / JWFloat …   →  PrimitiveFloat     (comfy_extras.nodes_primitive)
ConcatTextOfUtils / TextConcat …    →  StringConcatenate  (comfy_extras.nodes_string)
resolution pickers                  →  ResolutionSelector (comfy_extras.nodes_resolution)
attention backend switches          →  ModelAttentionBackend
lora bypass helpers                 →  LoraLoaderBypassModelOnly
```

Replacing with a built-in removes a dependency permanently. Installing a package to
satisfy one trivial node adds one forever — and it can disturb an environment that is
currently working.

### 🚨 Before reporting a node as missing, check **display_name**

The same node can be registered under different `class_type` across packages or
versions while showing the **same name in the UI**:

```
class_type = PrimitiveFloat   display_name = Float   ← built-in
class_type = JWFloat          display_name = Float   ← a package
workflow says "type": "Float"                        ← a package's old registration
```

Scan `/object_info` for `display_name == <the missing type>`. If a built-in matches,
**it is not missing — it was renamed.** Say that, and swap to the built-in.

⚠️ And scan the *whole* table. `custom_nodes/` is not the whole table — built-ins live
in `comfy_extras/`. Reporting a built-in as missing is the most common false alarm.

### 🚨 Registration styles differ — one grep pattern is not enough

```
old  NODE_CLASS_MAPPINGS = {"Name": Cls}
new  comfy_entrypoint() + class Cls(io.ComfyNode)     ← no mappings dict at all
```

Grepping only for `NODE_CLASS_MAPPINGS` against a package using the new API returns
nothing, and you will conclude "not registered" about nodes that are sitting right
there. Scan class definitions too.

### Flag platform-locked nodes explicitly

```
RH_* prefix        platform-hosted nodes (RunningHub and similar)
encrypted nodes    obfuscated or licence-gated
```

**These cannot be landed.** They only run on that platform. Tell the user plainly which
ones they are, and that the workflow needs structural replacement, not a download.

### Record, do not ask yet

For each node that is not simply present, record which bucket it falls in and what the
options are. **Carry it to Phase 4** - do not stop the survey to ask.

An exact built-in equivalent is the one case you may act on without asking: swapping a
third-party `Float` for the built-in `Float` changes nothing the user would want a say
in. Everything else - updating a package, installing one, restructuring the graph -
goes into the report.

---

## Phase 2 — Models: a three-tier search, then hand back

```
① API            one call per filename, and it settles roughly 80% of them outright
② Web search     only for what the API missed — filenames sitting in unrelated
                 grab-bag repos are invisible to a name-matching API but indexed by
                 search engines, which crawl the file-listing pages
③ Report "neither route found it"   ← NOT "it does not exist"
                 hand back to the human, who knows the community map
```

📌 **This is fast. Do not describe it as expensive.** One API round decides most of the
list; the web round mops up the remainder. A survey of twenty models is minutes, not an
afternoon — and misjudging that cost will make you frame a workable job as a burden.

**Query with the exact filename.** Do not translate it into a description:

```
✅  Some_Model_V2          ❌  some model lora v2
underscores, version numbers and case all preserved
```

🚨 **"My channel didn't find it" ≠ "it doesn't exist".** Saying the second closes the
door on the human's knowledge — and they are often the one who knows the file lives in
someone's two-hundred-file dump, or that a model on one site and a model on another are
the same weights under different names.

### Verify every download by header, not by size

`safetensors` records where its data should end. Compare with the actual file size:

```python
n   = int.from_bytes(f.read(8), 'little')      # header length
hdr = json.loads(f.read(n))
end = max(v['data_offsets'][1] for k, v in hdr.items() if k != '__metadata__')
complete  ⟺  8 + n + end == os.path.getsize(path)
```

This separates "downloaded fully" from "downloaded halfway but looks big" — the second
only fails at load time, with an error that never mentions downloading.

Read `__metadata__` too: a base-model field confirms the file is even for this model.

### Record, do not download yet

Locating a file is survey work. **Fetching it is not** - that is the user's call,
because it costs their bandwidth, their disk and sometimes their money. Record for each
missing model: the exact filename, where it was found (or that neither route found it),
the file size, and the nearest alternatives. **Carry it to Phase 4.**

When you do report it, give the user the facts and the options:

1. **What exactly is missing** — full filename, where it is referenced, what it does
2. **The nearest things that do exist** — with the differences named. One suffix apart
   is a real question: is the suffix a re-quantisation (must match) or just the author's
   naming (safe to substitute)?
3. **What changes if substituted** — "the merge range is what determines the result, so
   a different quantisation of the same range should land in the same place" is a
   judgement the *user* is qualified to make, not you
4. **Never silently rename a file to match.** Storing `…V1` under the name `…V2` because
   the workflow asks for V2 plants a fault nobody can trace later. Keep the real name
   and repoint the workflow instead

⚠️ Some nodes need a **separate weight in a specific subdirectory** that nothing in the
graph advertises. **Node tooltips and error strings usually state the download URL and
the exact folder.** Read them before guessing.

---

## Phase 3 — Assets and dead wiring

Cloud workflows carry the author's uploads as hash filenames. **These are
unrecoverable.**

🚨 **Do not check assets against `/object_info`.** Those dropdowns are populated by the
frontend from the input directory and are not in the static schema — a cloud video will
sail straight through a checker that only reads `object_info`. **Check the filesystem
directly.**

⚠️ `widgets_values` has **two shapes**: a list for most nodes, a **dict** for some video
helper families. A checker doing `if not isinstance(w, list): continue` silently skips
every dict-shaped node.

### Record, do not delete yet

Cloud graphs routinely carry nodes that are **not connected to anything**, or connected
but never given a value. List them and **carry them to Phase 4**.

When reported, recommend **mute (`mode: 2`)** over delete - it keeps the author's
structure visible while stopping it from blocking the run, and deleting loses
information about how the author intended the graph to scale. But the choice is theirs.

---

## Phase 4 - One report, one decision

The survey is done and **nothing has been changed yet**. Put everything in front of the
user at once, and let them choose what to act on.

### Report every gap in the same shape

```
what is missing      exact name, where the workflow references it, what it does
is it out there      found at <source>, size N GB  /  neither search route found it
what is here now     already local / behind by N commits / not present at all
options              A, B, C - with what each costs
risk to your setup   <- the column people actually decide on
```

### Rank the options by what they touch

Not all fixes are the same size of decision. Say which is which:

| Risk | What it touches | Examples |
|---|---|---|
| **None** | nothing existing | swap to a built-in equivalent - edit the workflow copy - mute dead nodes |
| **Low** | disk and time only | download a model or LoRA: adds files, changes nothing that already runs |
| **Medium** | adds a dependency | install a new custom-node package: may conflict, needs maintaining forever |
| **HIGH** | **changes what already works** | update an installed package - update the host application itself - replace a model other workflows share |

**The high-risk row deserves its own sentence in the report.** If the user has pipelines
currently running on that package, an update can silently change their output. Say so
explicitly, and say what would have to be re-verified afterwards.

### Let them answer piece by piece

Present the gaps as a list they can answer selectively, not as one all-or-nothing
question. A realistic answer is "download those two, skip the package update, I will
find the third one myself" - and the procedure must be able to continue from there.

Include the option nobody lists: **do none of it**. Put it on the menu with the same
neutrality as the others — a user who decides a workflow is not worth the packages it
needs has used this phase correctly, and the phase succeeded.

⚠️ **List it; do not argue for it.** The moment you write "I suggest we skip this one",
you have moved from surveying to deciding, and you are deciding with far less context
than they have.

### Offer an HTML copy of the report

A survey report is a wide table with four or five columns per gap. **In a terminal it
wraps into mush, and on a phone it is unreadable.** When the report has more than a
couple of gaps — or whenever the user is working from a CLI or a phone — also write it
as a single HTML file and give them the path.

**What the file must be:**

```
one file, no external anything     no CDN, no fonts, no network - it has to open offline
responsive, phone first             the user is often checking this away from the desk
one card per gap                    not one giant table; cards stack on a narrow screen
risk shown as colour AND text       colour alone fails for colour-blind users and for print
options as radio buttons            including "skip this one" as an explicit choice
```

**And the part that makes it worth building:** a button that turns the selections into
a line of text the user can paste straight back into the chat.

```
[ Copy my decisions ]  ->  "download: A, C   skip: B   update package: no   I will find: D"
```

Without that, you have solved reading and left them typing a paragraph on a phone
keyboard. The HTML has to close the loop, not just open it.

**Sketch — adapt, do not copy verbatim:**

```html
<!DOCTYPE html><html lang="zh-Hant"><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<style>
 body{font:16px/1.6 system-ui,sans-serif;margin:0;padding:16px;background:#f6f7f9;color:#1a1a1a}
 .gap{background:#fff;border-radius:10px;padding:14px;margin:0 0 14px;box-shadow:0 1px 3px #0002}
 .risk{display:inline-block;padding:2px 10px;border-radius:99px;font-size:13px;font-weight:600}
 .r-none{background:#eceff1;color:#37474f}   .r-low{background:#e8f5e9;color:#1b5e20}
 .r-med {background:#fff8e1;color:#e65100}   .r-high{background:#ffebee;color:#b71c1c}
 .facts{font-size:14px;color:#555;margin:8px 0}
 .facts b{color:#1a1a1a;font-weight:600}
 label{display:block;padding:7px 0;border-top:1px solid #eee}
 button{width:100%;padding:14px;font-size:17px;border:0;border-radius:10px;
        background:#1a73e8;color:#fff;font-weight:600}
 #out{white-space:pre-wrap;background:#fff;border-radius:10px;padding:12px;margin-top:12px;
      font:14px/1.5 ui-monospace,monospace}
</style>
<h2>Landing report - <span id="wf"></span></h2>
<p class="facts">N gaps found. Nothing has been downloaded or changed yet.</p>

<div class="gap" data-id="unet">
  <div><b>minimax_h3_...b25-49-int8_r.safetensors</b> <span class="risk r-low">Low risk</span></div>
  <div class="facts">
    Referenced by: <b>UNETLoader #4</b><br>
    Out there: <b>not found with this exact suffix</b>; nearest is the same merge range
    without <code>_r</code>, 19.5 GB<br>
    Here now: <b>not present</b>
  </div>
  <label><input type="radio" name="unet" value="download nearest (19.5 GB)"> Download the nearest (19.5 GB)</label>
  <label><input type="radio" name="unet" value="skip" checked> Skip - I will supply it myself</label>
</div>

<button onclick="collect()">Copy my decisions</button>
<div id="out"></div>
<script>
function collect(){
  let lines=[];
  document.querySelectorAll('.gap').forEach(g=>{
    const c=g.querySelector('input:checked');
    lines.push(g.dataset.id+': '+(c?c.value:'(no answer)'));
  });
  const t=lines.join('\n');
  document.getElementById('out').textContent=t;
  navigator.clipboard&&navigator.clipboard.writeText(t);
}
</script>
```

📌 Keep the plain-text summary in the conversation as well. The HTML is for reading
comfortably; the chat is still where the decision gets made and recorded. Never make
opening a file mandatory in order to answer you.

### Then act only on what was agreed

Downloads, installs, updates - only the ones chosen. Verify each download by header (see
Phase 2). If a package was updated, **check that the nodes you already depend on still
exist afterwards**; do not take cumulative-update claims on faith.

---

## Phase 5 - Converting: swapping a node means four things, not two

🚨 The failure everyone hits: change `type` and `widgets_values`, then stop. Links
attach by **slot index**, so nothing breaks visually — no red box, no empty dropdown,
submission returns success — and then **execution** dies, because ComfyUI passes inputs
by **name**:

```
SomeNode.execute() got an unexpected keyword argument 'text2'
```

When swapping a node, do all four:

```
① type                     old → new
② widgets_values           remap positions; two nodes rarely order them the same
③ input slot NAMES         text1→string_a, text2→string_b   ← the one everyone forgets
④ leftover slots           remove unmapped slots carrying no link
                           (never remove a slot that has a link — that changes the graph)
```

Also update `properties["Node name for S&R"]`, and mark built-ins as core.

---

## Phase 5b - Verify in three layers. Each catches what the previous cannot.

```
① Programmatic check   nodes / model filenames / asset files / numeric ranges
② Load in a browser    the real UI, reporting its own missing nodes and empty dropdowns
③ Press Queue          execution
```

**None substitutes for the next.** A checker can pass clean, the browser then find a
cloud asset it missed entirely, and after that is fixed the run still die on a renamed
input slot.

### Layer ②: load it in a real browser

```js
await app.refreshComboInNodes();          // 🚨 files newly added to input/ are NOT in
                                          //    the frontend's cached dropdowns until this
await app.loadGraphData(data, true, true, 'name');
// then ask the UI itself:
//   nodes whose type matches /^Missing/            → missing nodes
//   n.has_errors                                   → error-flagged
//   widget value not in w.options.values           → empty dropdowns
//     (skip n.mode === 2, those are muted on purpose)
```

### Layer ③: press Queue and read the history

Submission returning success only means **accepted**. Poll the history endpoint for
`execution_error` — and make sure you are reading **this** run, not a previous one.

### Numeric traps that never raise

Some parameters accept only a legal series (a `step`, or a stride off a minimum). An
illegal value is often **coerced silently rather than rejected**, so the run produces
output that is not the size or length you think it is. Check every numeric widget
against `min`/`max`/`step` in `/object_info` and flag anything off the series.

📌 A well-built workflow computes such values instead of hard-coding them — an
expression node feeding a length is a sign the author knew about the constraint.
**When you see one, read the expression**: it is also the real value, and the widget
downstream of it is stale.

---

## Phase 6 — Acceptance: separate what you can judge from what you cannot

🚨 **Running to completion is not success.** Two different questions get confused here,
and only one of them is yours:

```
① Self-consistency   the prompt asked for X — is X in the output?   ← YOU can check this
② Meeting the goal   what this workflow is SUPPOSED to achieve      ← only the HUMAN has
                                                                       this standard
```

### ① Go back to the prompt and check it line by line

The prompt is the only specification you were given, so it is the only thing you can
hold the output against. Walk it:

```
Subjects and identity   how many characters, which reference supplies each — do they match
Named actions           each action the prompt spells out: did it happen
On-screen text          every quoted string: present, spelled correctly
Aspect and duration     declared values vs what the file actually is
Preservation clauses    things declared unchanged — did they stay unchanged
Prohibitions            things declared absent — did any slip in
```

Report this as **counts, not adjectives**: "the prompt names 5 actions; 3 are
identifiable on screen" is usable. "The action is rich" is not.

⚠️ Expect partial compliance as the norm, not as failure. Video models routinely obey
the *content* of a prompt while ignoring its *ordering* — timecoded shots often do not
play in sequence. State which parts were obeyed and which drifted; do not average them
into a verdict.

### ⛔️ DECISION POINT — you do not have the acceptance standard. Ask for it.

A workflow is usually built to achieve an effect its author never wrote down: a look, a
feel, a kind of motion, a resemblance to some existing piece. **Nothing in the JSON
tells you what "good" is here.** Asking a model to judge it produces confident nonsense.

Before declaring success, ask:

> "What should the result of this workflow look like, roughly? Is there a reference
> piece I can compare against? Which parts are the effect you want, and which
> deviations are acceptable?"

🚨 **Do not substitute a metric for the missing standard.** Numbers you can compute
(sharpness, motion magnitude, frame counts) measure *something*, but the thing the user
cares about often has no metric — and a metric chosen to fill the gap will confidently
point the wrong way. Two failure modes that look identical to a measurement but are
opposite to a viewer:

```
structure falling apart (blur)   vs   structure holding under extreme load (intensity)
```

Both raise "detail" numbers. Only a person can tell you which one you produced.

📌 So the honest shape of a delivery report is: **"it ran; here is what the prompt asked
for and here is what I can see was delivered; I do not have the standard for whether
this is the effect you wanted — take a look."**

---

## Report like this

```
✅ Running          N nodes / M models / A assets in place; Queue completed; output at <path>
📋 Against prompt   it asks for X items; Y are identifiable — itemised, in counts not adjectives
⚠️ Substituted      K things were swapped — it runs, but this is NOT the author's result
                    list each: what was swapped, why, and how it differs
⛔️ Your call        J decisions I am not making for you: <options and their costs>
❓ Beyond my judgement   whether this achieves the intended effect, and whether it is
                    worth landing at all — both are yours, not mine
🚫 Cannot be landed  platform-locked nodes / weights the author never published — say why
```

---

## Traps this procedure was built from

| Trap | Symptom |
|---|---|
| Scanning only `custom_nodes/` | built-in nodes reported as missing |
| Ignoring `display_name` | a renamed registration reported as missing |
| Knowing only `NODE_CLASS_MAPPINGS` | a whole new-API package reported as unregistered |
| Supporting one combo format only | every filename in new-API nodes reported as absent |
| Cross-matching a value against *every* dropdown on a node | a present model file reported as missing (some loaders have two dropdowns) |
| Checking assets via `object_info` | a cloud video missed entirely — only the browser saw it |
| Assuming `widgets_values` is a list | dict-shaped nodes skipped silently |
| Reading a widget that has a link | wrong duration and wrong prompt reported with confidence |
| Swapping type without renaming input slots | loads clean, submits clean, **dies on execute** |

📌 Most of these produce the same symptom: **a list of things that are not actually
missing.** Act on such a list and you will download files you already have, install
packages you do not need, and believe you were thorough. **When a check reports many
absences, suspect the check before you suspect the machine.**
