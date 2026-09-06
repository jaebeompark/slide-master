> See [`visual-review.md`](./visual-review.md) §6 for the per-page review dispatch that specializes this contract.

# Subagent Delegation Reference Manual

Delegation contract for both PPT Master route families: which work runs in a spawned subagent instead of the main agent, what each delegate receives, and what it returns.

**Hard rule — mechanics only.** A direct-PPTX route reuses §2–§4 the way it reuses shared scripts. Its own skill still owns every gate, and no §1 row belonging to the other family transfers with them.

**Trigger**: the main agent reads this file once, at the first delegable step of a run. Main-pipeline Steps 2 / 3 / 5 / 7, the `topic-research`, `verify-charts`, `verify-pptx-export`, `visual-review` workflows, and [`ppt-template-fill`](../../ppt-template-fill/SKILL.md) Step 7 dispatch their §1 row through §3.

---

## 1. Delegation Matrix

**Default — delegate every row below whose §5 threshold is met.** The main agent keeps only the decisions listed in the third column.

| Owner step | Delegate does | Main agent keeps | Return artifact |
|---|---|---|---|
| [`topic-research`](../workflows/topic-research.md) §2–3 | Web search, page fetch, fact extraction, research-document composition, image download | The §1 scope clarifier, the final document read, the hand-off checkpoint | `projects/<topic_slug>.md` + `projects/<topic_slug>/` |
| SKILL.md Step 2 | Read the imported `sources/` content files in full, write a fact + pointer index | The conversion and import commands, and every literal value that reaches a slide — Step 4 opens the pointed-at source section for those | `<project>/analysis/delegates/source_digest.md` |
| SKILL.md Step 3 | Structured-template preflight over every SVG root / slot in the workspace; template-index listing | The path trigger, the disambiguation question, the install file mapping | `<project>/analysis/delegates/template_preflight.json` |
| SKILL.md Step 4 §VII | Read all 76 `charts_index.json` summaries, match each page, write the read-audit table with verbatim summary quotes | The outline, every other confirmation field, and accepting the matches | `<project>/analysis/delegates/viz_match.md` |
| SKILL.md Step 4 §VIII | Read all 72 `image-layout-patterns.md` techniques, assign a `Layout pattern` per image row, satisfy the `#38`–`#46` image-as-canvas coverage rule | The §VIII plan, the rows themselves, and accepting the assignments | `<project>/analysis/delegates/layout_match.md` |
| SKILL.md Step 5 | Load `image-generator.md` / `image-searcher.md`, run the acquisition ladder, slice sheets, re-run `analyze_images.py` | The §VIII plan, the Step 4 manifests, the confirmed image path, any user-facing switch offer | Updated `images/image_prompts.json` / `images/image_sources.json` + `<project>/analysis/delegates/image_acquisition.md` |
| SKILL.md Step 7 | Run `verify_deck.py`, read `_pptx_render/<stem>-grid.png`, open suspect pages at full resolution | The fix decisions, re-export, the completion report to the user | `<project>/analysis/delegates/deck_verify.md` |

> Note: the Step 7 delegate is the only check that sees converted pixels. `svg_quality_checker.py` passes a deck whose SVG is correct but whose PPTX text frames re-wrap on export — a defect that exists only after conversion. Do not treat a clean SVG sweep as a substitute for this row.
| [`verify-charts`](../workflows/verify-charts.md) | The whole workflow — page list, calculator runs, coordinate diff | Approving the SVG edits the report proposes | `<project>/analysis/delegates/chart_verify.md` |
| [`verify-pptx-export`](../workflows/verify-pptx-export.md) | The whole workflow — OfficeCLI validate / issues / screenshot passes | Entering the workflow at all (explicit user approval), the repair decisions | `<project>/analysis/delegates/pptx_export_verify.md` |
| [`visual-review`](../workflows/visual-review.md) | Per-page rubric batches | Orchestration and the aggregated brand review | `<project>/.review/<page>.json` (that workflow's own §6 contract wins) |
| [`ppt-template-fill`](../../ppt-template-fill/SKILL.md) 7.1 | Run `validate`, check the read-back table | Accepting or rejecting the verdict | `<project>/validation/delegates/readback_check.md` |
| [`ppt-template-fill`](../../ppt-template-fill/SKILL.md) 7.2 | Run `officecli validate` + `issues --json`, triage each finding into fix / pre-existing / package-defect | **Every `fill_plan.json` rewrite** — the delegate measures and never rewrites copy | `<project>/validation/delegates/officecli_triage.md` |
| [`ppt-template-fill`](../../ppt-template-fill/SKILL.md) 7.3 | Render every page and read the images against the five visual checks | The fix decisions and the re-apply | `<project>/validation/delegates/render_check.md` |

### 1.1 Parallel groups

| Group | Dispatch |
|---|---|
| `topic-research` subtopics | One delegate per subtopic, all in one message, then one composer delegate |
| Step 7 verification | `deck-verify` and `chart-verify` in one message when both apply |
| template-fill 7.1 + 7.2 | One message; 7.3 waits, since a 7.2 rewrite invalidates the render |
| `visual-review` batches | Per [`visual-review.md`](./visual-review.md) §6.1 |

Everything else is a single delegate. Two delegates MUST NOT write the same file.

---

## 2. Forbidden — Work That Never Delegates

| Work | Owner |
|---|---|
| Executor Step 6 SVG page authoring | Main agent — SKILL.md discipline rules 6 / 7 / 9 |
| Any ⛔ BLOCKING confirmation stage | Main agent — Step 3 disambiguation, Step 4 Strategist stage, `topic-research` Step 1 |
| Route selection | Main agent — [`routing.md`](../workflows/routing.md) |
| Writing or editing `design_spec.md` / `spec_lock.md` | Strategist; delegates read them, never write them |
| Writing or editing `svg_output/*.svg` | Main agent, except the `visual-review` atomic fixes its own contract authorizes |
| Talking to the user | Main agent — a delegate that reaches a user decision stops and returns `needs_user` |
| Skipping or re-labeling a mandatory gate | Main agent |

| Inventing a fact, figure, source, or capability | Any delegate that gathers or measures — every claim traces to something it actually read or ran, and "no reliable figure found" is a valid answer |

**Hard rule**: a delegate's prompt states this forbid list explicitly. A delegate never spawns further delegates.

**Hard rule — report what you could not do.** A delegate records unreachable sources, conflicting values, and data it could not find, in both its artifact and its summary. A gap the main agent cannot see is worse than a gap it can route around.

---

## 3. Dispatch Contract

**Hard rule**: every delegate prompt is self-contained. It carries no conversation context and must run correctly in a fresh session.

| Field | Value |
|---|---|
| `subagent_type` | `general-purpose` |
| `model` | Per §3.1 — most rows do not need the session model |
| `description` | 3–5 words naming the step (`Step 5 image acquisition`) |

### 3.1 Model per delegate

**Rule**: two tiers. Use `sonnet` when the delegate's output is checkable against a fixed rule or a number. Leave `model` unset when the delegate must judge a rendered image or weigh sources against each other. A wrong call in the first group surfaces as a failed check; in the second it surfaces as a deck that looks fine and is wrong.

> Note: `sonnet` is the floor by standing preference — do not lower a row to `haiku` for the mechanical checks even though they would likely survive it.

| Delegate | `model` |
|---|---|
| Step 3 template preflight | `sonnet` — fixed structural contract, one verdict per page |
| template-fill 7.1 read-back | `sonnet` — eight fixed checks against extracted text |
| Step 2 source digest | `sonnet` — extraction and indexing; §1 already bars the digest from carrying literal values, which bounds the cost of a miss |
| Step 4 §VII viz matching | `sonnet` — every named key must grep in `charts_index.json` and every quote must match verbatim, so the whole output is checkable |
| Step 4 §VIII layout matching | `sonnet` — every `#id` and its name string must match `image-layout-patterns.md` verbatim |
| Step 5 image acquisition | `sonnet` — procedural ladder plus status bookkeeping |
| `verify-charts` | `sonnet` — numeric diff against calculator output |
| template-fill 7.2 triage | `sonnet` — classification against the §7.2 finding table |
| `topic-research` gather / compose | unset — source criticism and synthesis |
| Step 7 deck verification | unset — reading a render for overflow and font substitution is visual judgment |
| template-fill 7.3 render check | unset — same |
| `verify-pptx-export` | unset — same |
| `visual-review` | unset — that workflow's own contract |

The prompt MUST inline all of:

1. Absolute `<project_path>` and absolute `${SKILL_DIR}`.
2. The exact reference / workflow files to read, by absolute path — never "read the relevant reference".
3. The exact commands to run, fully expanded, with no placeholder left unresolved.
4. The absolute output artifact path from §1.
5. The §2 forbid list plus any step-specific don't-touch list.
6. The §4 return format.

**Forbidden — a prompt that assumes the delegate knows the project, the deck topic, the confirmed style, or which step is running.**

Spawn parallel delegates in **one** assistant message. Cap concurrent delegates at 10.

---

## 4. Return Contract

Each delegate writes its artifact to disk, then returns exactly:

```
status: ok | partial | needs_user | failed
artifacts: <absolute path> [, <absolute path> ...]
summary: <= 150 words
```

**Hard rule — the cap needs teeth.** A bare "<= 150 words" is ignored; measured across a five-delegate run, every delegate overran it. Write the summary line of the prompt as an explicit enumeration of what to report — "report X, Y, and whatever you could not do and why" — plus `HARD CAP — count them`. A delegate given a list to fill stays near the cap; one given only a number does not.

| Status | Main-agent action |
|---|---|
| `ok` | Advance the gate on the summary alone; open the artifact only when a later step needs its detail |
| `partial` | Open the artifact, resolve the named residue, then advance |
| `needs_user` | Ask the user the returned question, then re-dispatch with the answer inlined |
| `failed` | Resolve per [`failure-recovery`](../workflows/failure-recovery.md); do not retry the identical prompt |

**Forbidden — a delegate pasting file contents, command transcripts, or full search results into its report.** That defeats the delegation. Findings go in the artifact; the report points at it.

---

## 5. Skip Conditions

**Default — run inline when the delegate would not save context.** Delegate only when the work would otherwise pull roughly 5K+ tokens into the main context:

| Row | Delegate when | Run inline when |
|---|---|---|
| Step 2 digest | `sources/` **content-type** files total > ~1,500 lines, or ≥3 of them | Short source, or the user pasted the content into chat. Archived images and sidecars in `sources/` are not content files and never count toward the threshold |
| Step 3 preflight | Workspace has ≥5 template SVGs | Fewer — read them directly |
| Step 4 §VII matching | Any deck with visualization pages | An image-less, chart-less deck |
| Step 4 §VIII matching | Any deck with image rows | An image-less deck |
| Step 5 acquisition | Any `ai` / `web` / `slice` row needs the ladder, recovery, or slicing | Every row already terminal from the Step 4 background launch |
| Step 7 verification | `verify_deck.py` is being run, or the contact sheet needs a look | A re-run of a check that already passed unchanged |
| `topic-research` | Always | Never |
| template-fill 7.1–7.3 | Always — 7.3 reads one image per page, the largest single cost in that route | Never |

> Note: the Step 4 early background launch (`image_gen.py` / `image_search.py` as background processes) is not delegation and is unchanged. A Step 5 delegate collects those runs' results.

---

## 6. Host Fallback

`Agent` is a Claude-Code-specific primitive. On a host without it, the main agent runs each §1 row's procedure inline, in the same order, producing the same artifacts. No gate, command, or artifact path changes — only the context saving is lost.
