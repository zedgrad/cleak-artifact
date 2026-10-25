# LLM prompt catalogue (leak-inspector-tui)

> This document collects every prompt and instruction text the system sends
> to an LLM on the `leak-inspector-tui` path, which is the native
> tool-calling orchestration path that actually runs: system prompts, user
> messages, tool descriptions (under native tool-calling each tool's
> `description` also goes into the schema sent to the model), and the *notices*
> injected into the conversation. The commentary is ours; the prompts
> themselves are reproduced verbatim as they appear in the source. Every entry
> cites `file:line`. The source code is the authority, so if a prompt
> changes, update this file too.
>
> Scope: the TUI path only. The web path (`control-plane`, the JSON-action
> orchestrator) has been removed from the system and is no longer documented
> here.

---

## 0. Overview: the four-tier pipeline and where the LLM is called

The TUI path is not an agent free to call any tool. It is a tiered
multi-agent workflow, defined in
`apps/leak-inspector-tui/src/orchestrator/workflowInvestigation.ts`.

The LLM is called in two places: (1) the host-side POLICY tier that runs
BEFORE the pipeline (two one-shot prompts, the allocator profiler and the
strategist, see §0.5), and (2) the four-tier investigation (Stage A, Stage
B when needed, Stage D). Stage C and evidence merging are deterministic, with
no LLM. Every POLICY prompt is one-shot at temperature 0, its output is
verified (grep/clamp) and cached, and it is skipped in the benchmark
(the manifest is frozen), so evaluation stays deterministic.

| Tier | Job | LLM? | Prompt |
|---|---|---|---|
| A · Static fan-out | several static sub-agents, each given one group of candidates, running static tools to gather evidence; they do not return a verdict | yes | `staticSubAgentSystemPrompt` |
| B · Dynamic | with a known `buildCommand` → deterministic recipe, NO LLM (`buildTarget → lsanRun`); otherwise → one dynamic worker (LLM) builds and runs the sanitizer | sometimes | `dynamicWorkerSystemPrompt` |
| C · Synthesize | merge static context with dynamic evidence, stamp coverage | no | n/a |
| D · Hybrid judge | heuristic (deterministic) for every bundle; the LLM judge only for borderline bundles, batched (≤12 per call); consensus (k samples, batched per round) optional | yes | `llmJudge` BATCH_SYSTEM_PROMPT |

The defining properties:

- **Hard tool partitioning:** static sub-agents receive only static tools, the
  dynamic worker only dynamic ones, so the LLM cannot cross from static to
  dynamic or back. Stages A and B run concurrently (`Promise.all`,
  `workflowInvestigation.ts:282`).
- **Evidence is captured AUTOMATICALLY:** static context and dynamic findings are
  recorded by code (`withStaticContextCapture` / `withDynamicEvidenceCapture`);
  the model does not write them itself.
- Verdicts are written by code, not through an LLM tool call. A model-call or
  parse failure at any tier falls back to the heuristic verdict, so the
  system is never worse than its deterministic core.
- The `toolSelect` flag (`opts.toolSelect`, default `true`): when `false`,
  Stage A skips the agentic sub-agent and runs deterministic enrichment only
  (`scanController`'s `enrich` stage), and Stage B runs the deterministic recipe
  if one is available or skips, with no LLM worker fallback. Stage D still runs
  the LLM judge on borderline bundles, independently of `toolSelect`.
- The `dynamicAlreadyRan` flag (`ctx.dynamicAlreadyRan`): when dynamic-only
  discovery has already run (static=false, dynamic selective/aggressive), Stage B
  is skipped because the dynamic evidence is already attached to the bundles.

Provider dispatch (`packages/agent-core/src/providers/index.ts:23-31`):
`provider === 'anthropic'` → `callAnthropic`, everything else (`local` /
`openai` / `openai-compat`) → `callOpenAiChat`. The system prompt is delivered
differently per provider: Anthropic takes it as the top-level `system` parameter
(`providers/anthropic.ts:29`), while OpenAI-compatible endpoints receive it as
the first message, `{role:'system', content}` (`providers/normalize.ts:41`).

---

## 0.5. The POLICY tier (host-side, one-shot, BEFORE investigation)

Two per-project discovery prompts. Both follow the same shape: *gather host-side
→ one-shot `callModel` at temperature 0 → lenient Zod parse (like `parseVerdict`)
→ verify → cache under `<repo>/.cleak/`*. They are resolved in
`surfaces/headless.ts` and skipped when allocators are supplied explicitly
(as in the evaluation), which is what keeps evaluation deterministic.

### 0.5.1. Allocator profiler: `allocatorProfileSystemPrompt`

- **File:** `apps/leak-inspector-tui/src/domain/allocatorProfiler.ts:90-103`

Reads the headers (in full) and the sources (truncated), then lists the
project's custom allocation and deallocation APIs (factories such as
`*_new`/`*_create`/`*_dup`, parsers and printers that return owned memory,
`#define ALLOC` macros, smart-pointer helpers like `make_unique`) together with
`ownershipNotes` (ownership conventions: transfer, borrow, refcount, pool,
"Delete skips const", and so on). The output is JSON
`{allocators, deallocators, reallocators, ownershipNotes, confidence, explanation}`.
Verification: a name is kept only if it is a valid identifier, is not libc,
and actually appears in the source that was shown to the model
(`verifyNames`, the anti-hallucination check). Accuracy is measured by
`scripts/validate-allocator-profile.ts` (P/R/F1 against ground truth). The result
feeds `extraAllocators`/`extraDeallocators` and `ownershipNotes`.

```text
You are a C/C++ memory-management API analyst. Given header/source excerpts from ONE project, identify that project's CUSTOM allocation and deallocation functions — the wrappers/replacements for malloc/free: factory constructors and duplicators (`*_new`, `*_create*`, `*_alloc*`, `*_dup`, `*_clone`, `*_copy` that return owned memory) and their matching releases (`*_free`, `*_delete`, `*_destroy`, `*_release`, `*_unref`, `*_close`, pool/arena destructors) — plus any ownership conventions a leak checker must know.
Respond with a JSON object ONLY (no prose), in this exact shape:
{"allocators":["..."],"deallocators":["..."],"reallocators":["..."],"ownershipNotes":["..."],"confidence":0.0-1.0,"explanation":"..."}
Rules:
- Be EXHAUSTIVE on allocators: list EVERY function that returns newly-owned memory — ALL constructors/factories (`*_New*`, `*_Create*`), duplicators (`*_Duplicate`, `*_dup`, `*_clone`, `*_copy`), and parsers/printers/serializers that return an owned buffer (`*_Parse*`, `*_Print*`). Do not stop at a few examples — include the whole family even if it is long.
- Use EXACT function names as they appear in the code.
- Include INTERNAL/static helpers too (e.g. a static `cJSON_malloc`/`*_New_Item` wrapper, a private `*_strdup`), not only the public API — leaks often flow through them.
- Include MACRO allocators/deallocators: a `#define MY_ALLOC(n) malloc(n)` (or `#define FREE_OBJ(p) ...`) is an allocator/deallocator named `MY_ALLOC`/`FREE_OBJ` — list the macro name.
- Do NOT include plain libc malloc/calloc/realloc/free/strdup — the engine already knows those. Only the project's CUSTOM names.
- allocators/reallocators RETURN newly-owned heap memory the caller is responsible for; deallocators FREE or consume it.
- ownershipNotes: short, project-specific rules (transfer vs borrow, refcounting, pool/arena "free the pool, not each object", "X skips items flagged Y", a constructor that steals its argument). Empty array if none.
- Precision over recall: if unsure a name is an allocator/deallocator, OMIT it.
```

### 0.5.2. Strategist: `strategistSystemPrompt`

- **File:** `apps/leak-inspector-tui/src/domain/strategist.ts:72-80`

Reads repository metadata (file count, share of C++, whether a build system is
present, smart-pointer density) plus a summary of the allocator profile, and
chooses `{runDynamic, judge: single|consensus, staticDepth: shallow|full}`. In
v0 only `runDynamic` is wired (it skips the dynamic stage when the project
cannot be built). A deterministic rule-based fallback takes over whenever the
model call fails. Opt-in via `--strategy auto`.

```text
You are the STRATEGIST for a C/C++ memory-leak analyzer. Given a project's metadata + memory-API profile, choose an analysis plan by SELECTING among the engine's existing deterministic capabilities — you do not invent analysis. Respond with a JSON object ONLY:
{"runDynamic": true|false, "judge": "single"|"consensus", "staticDepth": "shallow"|"full", "rationale": "..."}
Guidance:
- runDynamic: run sanitizer (LeakSanitizer) dynamic analysis ONLY if the project is plausibly buildable (a build system is present) AND dynamic coverage would help. If there is NO build system, set false — building is impossible, so skip the expensive dynamic stage (no recall lost).
- judge: "consensus" (slower, more robust) for projects whose ownership is subtle — heavy smart-pointer / refcounting / C++; else "single".
- staticDepth: "shallow" (function summaries only) for tiny or trivial projects; "full" (path constraints + ownership + interprocedural) for larger or control-flow-heavy ones.
Be decisive; prefer cheaper plans when they lose no recall.
```

---

## 1. Stage A: Static sub-agent

Each group of candidates (grouped by *file affinity*, `staticGroupSize=4`)
spawns one sub-agent with a small context and a restricted set of static tools.
The sub-agent runs until every candidate in its group has static context, then
calls `${DONE_STATIC}`.

### 1.1. System prompt: `staticSubAgentSystemPrompt(repoPath)`

- **File:** `apps/leak-inspector-tui/src/domain/subAgentPrompts.ts:39-54`
- **Interpolated variables:** `${repoPath}`.

```text
You are a STATIC-ANALYSIS evidence-gathering sub-agent for C/C++ memory leaks.
You do NOT record verdicts — your only job is to RUN the static tools so the system can collect structured evidence for each candidate, then call `${DONE_STATIC}`.

For EACH candidate in your list, gather evidence:
- `functionSummary` (filePath, functionName) — alloc/free balance + leaky exit paths.
- `pathConstraints` (filePath, lineNumber of the allocation) — feasible leaking paths.
- `astScan` (filePath) — structural patterns + early returns.
- `ownershipConventions` (filePath) — ownership-transfer / missing-free conventions.
- `read_file` to inspect the source and, for interprocedural cases (a function returning an allocation), follow the caller.

Efficiency: you MAY call several of these read-only tools in a SINGLE turn — they run in parallel. The repository root is ${repoPath}.
When you have run the static tools for EVERY candidate in your list, call `${DONE_STATIC}`. Do NOT reply with prose — only tool calls advance the work.
```

### 1.2. User message: `staticSubAgentUserMessage(bundles)`

- **File:** `subAgentPrompts.ts:56-63` (one line per candidate, via
  `candidateList`, `:28-35`).

```text
Gather static evidence for these ${N} candidate allocation site(s):
- ${bundleId} — ${function}() at ${file}:${line} (${allocation_type})
…

Run the static tools for each, then call ${DONE_STATIC}.
```

### 1.3. Tools exposed to the model (Stage A)

`workflowInvestigation.ts:199-203`. For each tool's description see
[§5](#5-tool-descriptions-sent-to-the-model).

- The five content-capable static tools: `candidateScan`, `astScan`,
  `functionSummary`, `pathConstraints`, `ownershipConventions`. These are all
  the static tools exposed to the model (filtered by `CONTENT_CAPABLE_TOOLS`,
  `mcpToolPlan.ts:103-109`). The other six static tools (`indexFiles`,
  `callGraph`, `interproceduralFlow`, `ownershipSummary`, `scanBuildRun`,
  `scanBuildGetReport`) are excluded because they need a shared filesystem
  mount; leaving them out keeps the analyzer stateless and remotely deployable.
- `read_file` (see [§5](#read_file-and-the-done-tools)).
- `done_static`: the terminal tool that ends the loop.

### 1.4. Completion nudge (injected as a user message when the model stops early)

- **File:** `workflowInvestigation.ts:211-216` (the `checkCompletion` guard,
  called from `agent-core/loop.ts:168-177`, at most `maxStopNudges` times).

```text
You stopped, but ${N} candidate(s) have NO static evidence yet: ${ids}. Run functionSummary/pathConstraints/astScan/ownershipConventions for them, then call ${DONE_STATIC}. Only tool calls advance the work.
```

---

## 2. Stage B: Dynamic worker

Runs concurrently with Stage A. It is enabled only when `dynamicMode ≠ off`
and a `dynamicClient` is available (`workflowInvestigation.ts:127`). There
are two branches.

### 2.1. The deterministic branch (NO LLM): `runDeterministicDynamic`

- **File:** `apps/leak-inspector-tui/src/domain/dynamicEvidence.ts:234-274`.
- When the case already carries a `buildCommand` (for instance Juliet's
  convention, `make CC=clang CXX=clang++`), Stage B runs the fixed recipe
  `buildTarget(buildCommand) → lsanRun(a.out)` with no LLM prompt, so the run,
  and therefore the coverage and the verdict, is reproducible. It falls back
  to the LLM worker only when the build fails or no `buildCommand` exists.
- **Notice:** `Stage B · dynamic evidence: deterministic recipe (buildTarget → lsanRun, no LLM)`
  (`:236`); on failure: `Stage B · deterministic recipe produced no run — falling back to the LLM worker` (`:251`).
- With `toolSelect=false` the deterministic branch skips the LLM worker entirely:
  `Stage B · deterministic recipe produced no run — tool_selector off, skipping LLM worker` (`:248-249`).

### 2.2. System prompt: `dynamicWorkerSystemPrompt(repoPath, buildCommand?)`

- **File:** `subAgentPrompts.ts:67-78`
- **Interpolated variables:** `${repoPath}`; the line
  `A hint build command was provided: …` appears only when a `buildCommand`
  exists.

```text
You are a DYNAMIC-ANALYSIS sub-agent for C/C++ memory leaks. Build the project ONCE with a sanitizer, run it under a sanitizer, then call `${DONE_DYNAMIC}`.

1. `read_file` the Makefile / CMakeLists.txt / build script under ${repoPath} to learn how it builds. A hint build command was provided: `${buildCommand}`.
2. `buildTarget` (projectPath=${repoPath}, buildCommand = a clang command with sanitizer flags). Prefer LeakSanitizer (`-fsanitize=leak -g -O0`) — it reports at exit and never aborts mid-run.
3. Run the binary with `lsanRun` (or `asanRun` / `valgrindMemcheck`).

The system CAPTURES every finding from your sanitizer runs AUTOMATICALLY and attaches it to the matching candidate — you do NOT record evidence yourself. Your only job is to get a successful sanitizer run.
Build at most ONCE and run each dynamic tool at most once. If a build or sanitizer fails twice, stop and call `${DONE_DYNAMIC}`. When a sanitizer has run, call `${DONE_DYNAMIC}`. Do NOT reply with prose.
```

> This is where "the model learns the build command": the worker reads the
> Makefile or CMake file itself and then chooses its own clang-plus-sanitizer
> `buildTarget` command. The discovered `buildCommand` is only a hint.

### 2.3. User message: `dynamicWorkerUserMessage(bundles)`

- **File:** `subAgentPrompts.ts:80-90` (lists at most 100 candidates).

```text
Run a sanitizer once over the build that covers these ${N} candidate(s):
- ${bundleId} — ${function}() at ${file}:${line} (${allocation_type})
…
… and ${N-100} more.        # only when N > 100

Build once, run a sanitizer (the system captures the findings), then call ${DONE_DYNAMIC}.
```

### 2.4. Tools exposed to the model (Stage B)

`workflowInvestigation.ts:262-266`: the nine dynamic tools (see
[§5](#dynamic-analyzer-nine-tools)) wrapped in `withDynamicEvidenceCapture`, so
findings are captured automatically (`record_evidence` is deliberately
absent from the toolset), plus `read_file` and `done_dynamic`. The worker's
`maxTurns` is `maxTurns + 10`.

### 2.5. Completion nudge

- **File:** `workflowInvestigation.ts:275-278` (prevents the worker stopping
  before one sanitizer run has succeeded).

```text
No successful sanitizer run yet. buildTarget (with a sanitizer flag), then run lsanRun/asanRun/valgrindMemcheck, then call ${DONE_DYNAMIC}. Only tool calls advance the work.
```

### 2.6. Stage B2: Targeted harness worker (opt-in, off by default)

- **Gate:** `workflow.targetedHarness.enabled` (default `false`; enable with
  `cleak config set workflow.targetedHarness.enabled true` or `--harness`) AND a
  `ctx.buildCommand` must exist. It runs only for candidates that static context
  plus Stage B's whole-binary run still leave borderline
  (`needsTargetedDynamic`, `workflowInvestigation.ts:337`), at most
  `maxHarnessesPerScan` of them (default 5), at concurrency
  `harnessCfg.concurrency` (default 2). Borderline bundles are prioritised ahead
  of CONFIRMED_LEAK bundles queued for a double check
  (`workflowInvestigation.ts:341`, flag `verifyConfirmedLeaks`).
- **System prompt:** `harnessWorkerSystemPrompt(repoPath, buildCommand, analyzerProjectPath)`,
  `domain/subAgentPrompts.ts:101-144`.
- **User message:** `harnessWorkerUserMessage(bundle, staticCtx, suggestedClosureFiles)`,
  `domain/subAgentPrompts.ts:151-174`. `suggestedClosureFiles` is a
  deterministic hint from a single `interproceduralFlow` call computed once
  per scan (not an LLM tool call; `workflowInvestigation.ts:349-350,362-380`);
  the worker still decides whether to use it.
- **Tools exposed to the model:** `buildHarness`, `lsanRun`, `asanRun` (wrapped
  in `withDynamicEvidenceCapture`), `read_file`, `done_harness`
  (`workflowInvestigation.ts:382-394`).
- **maxTurns:** `min(cfg.maxTurns, 12)` (`:400`).
- Completion nudge (injected when the model tries to stop before a sanitizer
  run has succeeded, `:402-405`):

```text
No sanitizer run yet for this harness. Call buildHarness, then lsanRun/asanRun on the returned binaryPath, then call done_harness. Only tool calls advance the work.
```

````text
You are a TARGETED-HARNESS sub-agent for ONE C/C++ memory-leak candidate. Static analysis alone was inconclusive for it, and the project's normal execution didn't exercise it either — your job is to write a SMALL driver that calls just the suspicious function, compile it, and run it under a sanitizer.
You do NOT record verdicts — you only get a sanitizer to run against the right code. The system captures findings from your sanitizer run automatically.

The repository root is ${repoPath}. Its build command is: `${buildCommand}`.

Linkage — pick ONE based on the "static linkage" fact given below:
- If the target function is NOT static (external linkage): your harness source should `extern`-declare it (matching the given return type + parameter types) and call it with concrete argument values. Pass the file(s) needed to link it (usually just the target's own file) as `closureFiles`.
- If the target function IS `static` (internal linkage): a separate translation unit cannot link it. Your harness source MUST `#include "<absolute path to the target file>"` so the function compiles into the harness's own translation unit — and you must NOT also list that file in `closureFiles` (it would be defined twice).

Choosing argument values: use the given path-constraint text to pick values that drive execution down the LEAKING branch (e.g. a condition text like `flag == NULL` means pass NULL for that parameter). A best-effort, plausible value beats no attempt — you do not need to be exhaustive.

REQUIRED harness shape — the SAME source is compiled twice (once as a plain single run, once as a libFuzzer binary for a follow-up fuzz pass), so it must never define BOTH `main()` and `LLVMFuzzerTestOneInput` for the same build — linking libFuzzer already provides its own `main()`, and a harness with both is a duplicate-symbol error. Put the actual call in a `static` helper and switch the entry point on the `HARNESS_FUZZ` macro (the system defines it automatically for the fuzz build — you never set it):
```c
/* extern-declare OR #include the target file per the linkage rule above */
static void run_case(/* concrete or data-derived args */) {
    /* the actual call that should reach the leaking branch */
}
#ifdef HARNESS_FUZZ
#include <stdint.h>
#include <stddef.h>
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    /* derive AT LEAST ONE argument/buffer from data/size, then: */
    run_case(/* ... */);
    return 0;
}
#else
int main(void) {
    run_case(/* concrete values chosen from the path constraints */);
    return 0;
}
#endif
```
Follow this skeleton exactly — same `run_case` body called from both branches, only the entry point differs.

Tools:
- `read_file` — inspect the target file (and any header it needs) before writing the harness.
- `buildHarness` (projectPath="${analyzerProjectPath}", buildCommand, harnessSource, targetFile, closureFiles, entryStyle="single") — compiles+links your harness against the real project's own compiler flags. On reason="harness_unresolvable" (see its own description), stop and call done_harness, do not retry. Always call with entryStyle="single" — the system runs the fuzzer build itself later if needed.
- `lsanRun` (binaryPath) or `asanRun` (binaryPath) — run the compiled harness under a sanitizer, using the binaryPath `buildHarness` returned.

Call buildHarness at most ONCE, then run one sanitizer on the result. If buildHarness fails for a reason other than harness_unresolvable, you may fix the harness source and try ONE more time. When a sanitizer has run (or the harness is unresolvable), call `done_harness`. Do NOT reply with prose — only tool calls advance the work.
````

> After the worker comes a deterministic fuzz escalation with NO extra LLM
> turn (`workflowInvestigation.ts:413-443`): if the single-shot run came back
> clean but the bundle is still borderline AND a `libfuzzerRun` tool is
> available, the system rebuilds the existing harness itself with
> `entryStyle="fuzzer"` and runs it for `fuzzBudgetMs` (default 15s), reusing
> exactly the harness the worker wrote without asking the model again.

---

## 3. Stage D: Judge (LLM, borderline bundles only)

The heuristic scores every bundle first (`heuristicVerdict`,
`workflowInvestigation.ts:299-303`). Only borderline or contradictory
bundles are escalated to the LLM; the rest keep their heuristic verdict.

### 3.1. The escalation gate: which bundles reach the LLM

- `isBorderline`: `apps/leak-inspector-tui/src/domain/llmJudge.ts:654`: a
  verdict of `likely_leak` or `uncertain`, or `confidence ∈ [0.35, 0.7]`.
- `shouldEscalate`: `llmJudge.ts:672`: `isBorderline`, or a
  contradiction between static evidence and the verdict, or between dynamic
  evidence and the verdict (for example flagged as a leak although the dynamic
  run was clean; not flagged although a correlated runtime leak exists; a verdict
  that contradicts `deriveFusion`). This is where consensus is re-enabled exactly
  when it matters most.

### 3.2. System prompt: `SYSTEM_PROMPT` (and `BATCH_SYSTEM_PROMPT`)

- **File:** `llmJudge.ts:51` (`SYSTEM_PROMPT`)
- **Output format:** JSON only (no native tool-calling; `tools: []`).
- Batch judging (the default for every Stage D call, single-shot and
  consensus alike): instead of one LLM call per bundle, N borderline bundles are
  merged into one call (`judgeBundlesBatched`, `llmJudge.ts:482`) that
  returns a JSON ARRAY of N verdicts rather than one object, using
  `BATCH_SYSTEM_PROMPT` (`llmJudge.ts:61`) in place of `SYSTEM_PROMPT`. The two
  prompts share the identical calibration rules (`CALIBRATION_RULES`,
  `llmJudge.ts:34`) and differ only in how they describe the output format (a
  single object versus an N-element array). A batch holds at most
  `JUDGE_BATCH_SIZE = 12` candidates per call (`llmJudge.ts:369`): large enough
  to amortise the system prompt's fixed cost of roughly 960 tokens (measured on
  one libsolv case, 738 candidates meant 738 separate calls before batching
  existed), small enough to avoid "lost in the middle" and to bound the damage if
  one response is truncated or malformed. The motivation: on real projects
  (LAMeD) the cost is driven by the number of calls, not the size of each
  one, so batching attacks the actual cause rather than tuning individual
  prompts.

```text
You are an expert C/C++ memory-leak analyst. Decide whether ONE allocation is a real leak, using the code, static context, and any runtime evidence provided.
Respond with a JSON object ONLY (no prose), in this exact shape:
{"verdict": "confirmed_leak | likely_leak | uncertain | likely_false_positive | false_positive", "confidence": 0.0-1.0, "explanation": "...", "evidence": ["..."]}
Calibrate using the EVIDENCE, in this priority order:
- A runtime leak (sanitizer/valgrind) whose allocation site is LINKED to this candidate is decisive → confirmed_leak (confidence ≥ 0.9). Weight by leak kind: definitely_lost / asan_leak ⇒ decisive; possibly_lost ⇒ weak corroboration; still_reachable ⇒ usually benign, lean false_positive.
- A runtime finding in the SAME FILE but a DIFFERENT site (not linked) is weak — do not treat it as proof for this allocation. still_reachable with no other evidence → false_positive.
- A CLEAN sanitizer/valgrind run is only strong exculpation when it EXERCISED THIS ALLOCATION (the evidence entry is CORRELATED to this candidate). A clean run that merely covered the FILE (a different site, or a run with no correlated entry) is weak — it does NOT clear this allocation.
- UNPAIRED alloc→free at the allocation site is a STRONG leak signal. When the static pairing table marks an allocation 'UNPAIRED' (no free found in this function), do NOT dismiss it with an ownership-transfer narrative (returned / stored in a struct / handed to a callback) UNLESS the code snippet actually shows the pointer being returned, stored, or handed off. When the static pairing is UNPAIRED and the only counter-evidence is an ambiguous clean dynamic run, default to likely_leak — not false_positive.
- Ownership is decisive for false positives: if the allocation is RETURNED to the caller or its pointer is HANDED OFF to a sink/callback/another function (ownership transferred), freeing it is NOT this function's job. When ownership is transferred AND no runtime leak is linked to THIS allocation, answer likely_false_positive or false_positive — do NOT flag it just because you cannot see the free inside this snippet. An UNPAIRED alloc→free with a reachable leak path and NO ownership transfer → confirmed_leak (≥ 0.85).
- PATH-SENSITIVE leak: an allocation freed on the main/success path but NOT on an error or early-return path (e.g. `if (err) return NULL;` or `goto fail;` before the free) IS a leak — confirmed_leak — EVEN IF the value is returned or added to a structure on the success path. Ownership transferring on success does not cover the error path that loses the object. If the static context lists the allocation as freed "on some paths only" (conditional) or names it on a reachable un-freed exit path, treat that as decisive.
- PARAMETER-ownership leak (allocation_type 'parameter_ownership'): when a function frees a pointer PARAMETER on some paths (taking ownership from the caller, e.g. cJSON's `merge_patch` does `cJSON_Delete(target)`) but a reachable branch returns WITHOUT freeing it, that branch leaks the parameter — confirmed_leak. The parameter has no allocation site in the function; judge it by the conditional free + the reachable un-freed exit.
- Freed on all paths / static-global → false_positive (high confidence). Use uncertain only when the evidence is genuinely insufficient.
- Control flow is concrete, not hypothetical: a constant or scaffolding global such as `if(1)`/`if(0)` or `globalReturnsTrue()` does NOT change between two checks in the SAME function — `if(1)` always runs and `if(0)` is dead code. If the buffer is freed under the same condition it was allocated (or in the `else` of a constant `if`), it IS freed. Do NOT call a leak just because the `free()` sits in a different block, behind a constant condition, or after a `break`/in a second loop — trace whether it actually executes.
```

> Changes since the first version of this document (now in sync with the
> code; the earlier text of this section had drifted):
> (1) The CLEAN sanitizer/valgrind bullet was tightened. It used to treat "a
> clean run that EXERCISED this allocation" as strong exculpation; it now
> exculpates only when the evidence entry is CORRELATED to this exact
> candidate (`llmJudge.ts:615`, the "task-5 class-(b) hardening" note), so a
> clean run that merely covered the same FILE at a different site no longer
> counts as having exercised this allocation and cannot wrongly clear an
> UNPAIRED one.
> (2) The UNPAIRED alloc→free bullet is new: a static pairing marked UNPAIRED is
> a STRONG leak signal and must not be dismissed with an ownership-transfer
> narrative unless the snippet actually shows the pointer being returned,
> stored or handed off; when the only counter-evidence is one ambiguous clean
> dynamic run, the default is `likely_leak`, not `false_positive`.
> (3) The PATH-SENSITIVE bullet gained its closing sentence, "Ownership
> transferring on success does not cover the error path that loses the object."
> (4) The PARAMETER-ownership bullet gained "The parameter has no allocation site
> in the function; judge it by the conditional free + the reachable un-freed
> exit."

### 3.3. User message

- **File:** `buildJudgeUserMessage` (`llmJudge.ts:332`, single bundle, used when
  a batch holds only one candidate) and `buildBatchUserMessage`
  (`llmJudge.ts:383`, N candidates, each as a
  `── CANDIDATE i (id: ...) ──` block concatenated together, reusing
  `sourceSnippet` / `summarizeStatic` / `summarizeEvidence` unchanged, so
  batching changes the wire format and not what each candidate shows).
  `${sourceSnippet}` is the whole enclosing function around the allocation,
  with comments stripped so benchmark labels cannot leak; the fallback window
  is ±(6,5) lines (`judge-shared.ts:95`). `${summarizeStatic}` and
  `${summarizeEvidence}` are built from the static context and the dynamic
  evidence.

```text
ALLOCATION SITE: ${function}() at ${file}:${line} (${allocation_type})

CODE (context around the allocation):
```c
${sourceSnippet}
```

STATIC ANALYSIS CONTEXT:
${summarizeStatic}        # Ownership · alloc→free pairing · feasible leak paths (or a fallback count of allocs/frees/early returns)

DYNAMIC EVIDENCE (${N}):
${summarizeEvidence}      # per finding: tool:kind bytes/blocks @ site [LINKED | same file, different site | CLEAN]; or "(none)"

PROJECT OWNERSHIP CONVENTIONS (respect these — they encode how THIS project manages memory):   # ONLY when present — ownershipNotes discovered by the allocator profiler (§0.5.1)
- <e.g. "cJSON_Delete skips strings flagged cJSON_StringIsConst"; "cJSON_Add*ToObject transfers ownership to the parent">

Return your JSON verdict.
```

### 3.4. Parsing and enrichment

- Single-object parse: `parseVerdict` (`llmJudge.ts:202`): `JSON.parse`,
  falling back to the regex `/\{[\s\S]*\}/`. A result is accepted only if
  `verdict` is one of the five valid labels (`isLeakVerdictString`), and
  `confidence` is clamped to `[0,1]`. A failure yields `null`, which keeps the
  heuristic verdict.
- Batch-array parse: `parseBatchVerdicts` (`llmJudge.ts:278`): expects one
  JSON array of N elements and matches each item back to `expectedIds` via
  its `id` field (by `id` when present, otherwise by position). One malformed
  or missing item costs only that candidate's verdict; the other N-1 are
  processed normally, so a small error cannot spoil a whole batch. There is an
  NDJSON fallback (`extractJsonObjects`, `llmJudge.ts:229`): when the
  response is not a valid `[...]`, the text is scanned by brace depth (ignoring
  braces and commas inside strings) to split out each standalone JSON object,
  whatever the separators or pretty-printing. This came from an observation in
  practice: the `mimo-v2.5` model (via the `opencode-go-2` endpoint) returned
  one JSON object per line instead of an array, despite
  `BATCH_SYSTEM_PROMPT` asking explicitly for one. Without the fallback the
  entire batch fell back to the heuristic even though the model's answer was
  correct and complete. The failure was reproduced from real response data and is
  covered by a test.
- A valid verdict from either parse path is passed through `enrichLeakVerdict`
  (`@cleak/common/analysis/heuristic-judge`) to attach a `rootCause` and a
  source-anchored repair diff.

---

## 4. Batch judging and the consensus judge (optional)

### 4.1. Batch judging (every Stage D run, including `CONSENSUS_N = 1`)

Borderline bundles are no longer judged one at a time. They are grouped into
batches of at most `JUDGE_BATCH_SIZE = 12` (`llmJudge.ts:369`) and sent in one
call (`judgeBundlesBatched`, `llmJudge.ts:482`), parallelised by
`cfg.workflow.judgeConcurrency` (`stageHybridJudge`,
`workflowInvestigation.ts:623-640`). The cache check runs before batching, so
a cache-hit bundle never occupies a slot. The motivation: one real-project case
(libsolv) had 738 borderline candidates, which meant 738 separate LLM calls under
the old baseline; batching brought that to 34 calls, a factor of about 21.7, at
the same accuracy on Juliet CWE-401 (P/R/F1 = 1.000, no regression).

### 4.2. Consensus (enabled when `CONSENSUS_N > 1`): batched by ROUND

Consensus has no prompt of its own. It reuses `BATCH_SYSTEM_PROMPT` from §3.2
verbatim (not the single-bundle `SYSTEM_PROMPT`). It used to call
`judgeBundleWithLlm` N times per bundle (N × bundle-count calls); now
`judgeBundlesConsensusBatched` (`llmJudge.ts:543`, called from `stageHybridJudge`,
`workflowInvestigation.ts:549-600`) samples by ROUND across bundles instead:

- For `round = 0..N-1`, every still-active bundle (not yet locked by early stop)
  is grouped into batches of ≤ `JUDGE_BATCH_SIZE`, one `judgeBundlesBatched` call
  per batch. That is precisely "sample #round for these bundles"; it is not new
  logic, just the §4.1 batch function called again at
  `cfg.consensus.temperature`.
- With `cfg.consensus.earlyStop` enabled, `isDecisionLocked`
  (`consensus-judge.ts:257`, unchanged) is applied per bundle after each round
  to drop it from the active set. This is exactly the stopping condition of
  the old `sampleWithEarlyStop`; the only difference is that many bundles now
  share one call per round instead of each looping on its own.
- At the end, `combineVerdicts` and `judgeHeuristically` (both unchanged, in
  `consensus-judge.ts`) merge the N samples, or fewer if early stop
  locked a bundle sooner, into one `ConsensusVerdict` per bundle. The vote-merging logic (§8.4)
  is entirely unchanged; only the transport of samples differs.

Cost: without early stop the total is roughly
`N × ceil(borderline-bundles / JUDGE_BATCH_SIZE)` calls instead of
`N × borderline-bundles`. Measured on one cJSON case (109 candidates, 73
borderline, `CONSENSUS_N=3`, early stop on per the default config): 26 real calls,
against a non-batched baseline of up to 73×3 = 219 if early stop never locked
anything. All 73 bundles received a `ConsensusVerdict` with real `samples[]`
(between 1 and 3 samples each, depending on when early stop locked), rather than
falling back to the heuristic as they did before the NDJSON fix (§3.4).

---

## 5. Tool descriptions sent to the model

Under native tool-calling, each tool's `description` goes into the schema the
model sees. MCP tools are loaded through `loadMcpTools` → `wrapMcpTool`
(`agent-core/src/mcp/mcpToolAdapter.ts:39-42`), which preserves the remote tool's
`description` and `inputSchema`. The execution policy
(`mcpToolPlan.ts:45-56`): pure query tools (`CONCURRENCY_SAFE`) are read-only,
run in parallel, and time out at 30s; heavy tools (`SERIAL_HEAVY`: build,
sanitizer, scan-build) are read-only, serial, require approval (`ask`),
and time out at 300s.

### Static analyzer (11 tools): `apps/static-analyzer/src/mcp/static-mcp-server.ts:38-127`

| Tool | `description` (verbatim) | Exposed to the TUI? |
|---|---|---|
| `candidateScan` | *Scan a file for allocation sites (malloc, calloc, realloc, strdup, new). Optionally supply per-project factory allocators / custom deallocators (≈ LAMeD AllocSource/FreeSink) so wrapper-named allocators (e.g. cJSON_Duplicate) become candidates.* | yes |
| `astScan` | *AST-based structural analysis for memory leak patterns* | yes |
| `functionSummary` | *Summarize a function: alloc/free balance, local vars, calls. Optionally supply per-project allocators/deallocators so factory-allocated vars are paired.* | yes |
| `pathConstraints` | *Analyze path constraints and feasible paths around an allocation. Optionally supply per-project allocators/deallocators so factory allocations are tracked on exit paths.* | yes |
| `ownershipConventions` | *Detect ownership-transfer conventions in a file* | yes |
| `indexFiles` | *Index all C/C++ source files recursively from a root path* | no |
| `callGraph` | *Extract call graph edges and nodes. Optionally supply per-project allocators/deallocators so the alloc→free reachability chains track factory allocators.* | no |
| `interproceduralFlow` | *Interprocedural alloc/free flow tracing for a function. Optionally supply per-project allocators/deallocators so the trace tracks factory allocators (cJSON_malloc/_TIFFfree/…) — without them it is blind to non-libc memory APIs.* | no |
| `ownershipSummary` | *Summarize ownership conventions across files* | no |
| `scanBuildRun` | *Run the project-level Clang Static Analyzer (scan-build) over the project build* | no |
| `scanBuildGetReport` | *Retrieve Clang Static Analyzer (scan-build) findings* | no |

> "no" means excluded from the TUI, because the tool needs a shared filesystem
> mount (`CONTENT_CAPABLE_TOOLS`, `mcpToolPlan.ts:103-109`). The `scanBuild*` slot is now
> a self-contained Clang Static Analyzer (scan-build).
>
> Change since the earlier version: five tools, candidateScan, callGraph,
> functionSummary, interproceduralFlow and pathConstraints, gained descriptions
> covering `extraAllocators`/`extraDeallocators` (≈ LAMeD AllocSource/FreeSink)
> so the deterministic engine can recognise per-project allocator wrappers.

### Dynamic analyzer (nine tools): `apps/dynamic-analyzer/src/mcp/dynamic-mcp-server.ts:41-93`

| Tool | `description` (verbatim) |
|---|---|
| `buildTarget` | *Build the project with sanitizer-instrumented compiler flags* |
| `valgrindMemcheck` | *Run Valgrind Memcheck for detailed leak analysis* |
| `valgrindGetReport` | *Retrieve a normalized Valgrind report* |
| `valgrindListFindings` | *Query Valgrind findings with optional filters* |
| `valgrindCompareRuns` | *Compare two Valgrind analysis runs* |
| `asanRun` | *Run the binary under AddressSanitizer for leak detection* |
| `lsanRun` | *Run the binary under LeakSanitizer* |
| `runBinary` | *Run a binary without instrumentation* |
| `listRuns` | *List stored dynamic analysis runs* |

### `read_file` and the done tools

| Tool | `description` (verbatim) | File |
|---|---|---|
| `read_file` | *Read a source file from the repository (path relative to the repo root, or an absolute path inside it). Returns up to 16000 characters.* | `apps/leak-inspector-tui/src/domain/readFileTool.ts:17-43` |
| `done_static` | *Finish static evidence gathering for this group of candidates.* | `workflowInvestigation.ts:202` |
| `done_dynamic` | *Finish dynamic evidence collection.* | `workflowInvestigation.ts:265` |

> `read_file` is the only remaining domain tool. `buildDoneTool` is defined
> in `subAgentPrompts.ts:17-26` (a no-op returning `{done:true}`).

---

## 6. Notices and placeholders (agent-core: injected into the conversation or displayed)

- Compaction notice: `packages/agent-core/src/loop.ts:103`:
  ```text
  Compacted context: pruned ~${tokens} tokens of stale tool output
  ```
- Stop-nudge notice: `loop.ts:174` (when the model stops early while
  `checkCompletion` still has work outstanding):
  ```text
  Agent stopped early — nudging to finish (${stopNudges}/${maxStopNudges})
  ```
- Placeholder replacing an old tool result during compaction,
  `packages/agent-core/src/compaction.ts:66` (prefix `[elided:`, `:17`):
  ```text
  [elided: ${n} chars of stale tool output pruned to save context]
  ```
- Stage notices (progress display, `workflowInvestigation.ts`): `Stage A ·
  static evidence: …` (`:191`), `Stage B · …` (`:225,236,248,251,257`), `Stage C
  · synthesize: …` (`:296`), `Stage D · judge: …` (`:299,309`).

---

## 7. Loop and judge control parameters: `apps/leak-inspector-tui/src/config.ts`

| Parameter | Default | Env | Line |
|---|---|---|---|
| Max turns per sub-agent | `15` (dynamic worker `+10`) | `AGENT_MAX_TURNS` | `:183` |
| Stream idle timeout | `75000` ms | `LLM_IDLE_TIMEOUT_MS` (falls back to `LLM_TIMEOUT_MS`) | `:111` |
| `maxStopNudges` | `3` | n/a | `agent-core/loop.ts` |
| Consensus N | `1` (opt in to 3 or 5 for the ablation) | `CONSENSUS_N` | `:196` |
| Consensus rule | `weighted` (`majority` / `weighted` / `unanimous-to-flag`) | `CONSENSUS_RULE` | `:197` |
| Consensus temperature | `0.7` | `CONSENSUS_TEMPERATURE` | `:199` |
| Consensus concurrency | `3` | `CONSENSUS_CONCURRENCY` | `:200` |
| Judge temperature (single) | `0` | `JUDGE_LLM_TEMPERATURE` | `:118` |
| Static fan-out | `staticConcurrency=3`, `staticGroupSize=4` | `WORKFLOW_STATIC_*` | `:190-191` |
| Judge concurrency | `3` | `WORKFLOW_JUDGE_CONCURRENCY` | `:192` |
| Judge batch size | `12` (not configurable via config or env) | n/a | `llmJudge.ts:369` (`JUDGE_BATCH_SIZE`) |
| Provider | `local` (`openai` / `anthropic` / `openai-compat`) | `LLM_PROVIDER` plus the per-provider key | `:105-166` |
| Compaction threshold tokens | `100000` | `LLM_COMPACT_THRESHOLD_TOKENS` | `:185` |
| Compaction: recent turns kept | `3` | `LLM_COMPACT_KEEP_TURNS` | `:186` |

---

## 8. Decision boundaries

This section collects every place an output, whether from the LLM or the
heuristic, becomes "leak" or "not a leak". It is a summary that points back at
the code.

### 8.1. Collapsing labels to a boolean (the single source of truth)

A site counts as a leak ⇔ verdict ∈ `{confirmed_leak, likely_leak}`:

```ts
// packages/common/src/analysis/judge-shared.ts:33
export const LEAK_POSITIVE_VERDICTS: ReadonlySet<string> = new Set(['confirmed_leak', 'likely_leak']);
```

`uncertain`, `likely_false_positive` and `false_positive` are not flagged.
This set is the binary boundary for every consumer: consensus (`isFlag`,
`consensus-judge.ts:66`) and the evaluation scorer (`isFlagged`,
`evalScoring.ts:79`). There is no confidence cutoff anywhere on the
prediction or scoring path; confidence is used only to weight consensus, to
display severity, and for calibration/ECE.

### 8.2. The heuristic judge (no LLM, and the backstop for every bundle the LLM does not see)

`packages/common/src/analysis/heuristic-judge.ts:244-249` sums signal scores and
bands the total:

```ts
thresholds: VerdictThresholds = JUDGE_VERDICT_THRESHOLDS,  // { confirmed: 0.7, likely: 0.4 }
...
clamped >= thresholds.confirmed ? CONFIRMED_LEAK : clamped >= thresholds.likely ? LIKELY_LEAK : UNCERTAIN
```

The score contributions, in the order they appear in the source
(`heuristic-judge.ts:64-241`):

- Correlated runtime leak, `definitely_lost` / `asan_leak` / `indirectly_lost`: `+0.5`
- Correlated runtime leak, `possibly_lost`: `+0.2`
- Correlated runtime leak, kind unknown: `+0.4`
- Uncorrelated leak (a different site): `skip` (this fixes an earlier `+0.15` bug)
- Structural "high" missing free: `+0.5`
- Structural "medium" matched pattern: `+0.25`
- Unpaired alloc→free: `+0.25`
- Conditional free (freed on some paths only): `+0.15`
- No `free()` in the function: `+0.25`
- A feasible reachable leak path: `+0.2`
- scan-build agrees: `+0.15`
- Path-sensitive leak: `+0.15`
- Ownership not transferred out: `+0.15`
- Ownership transferred out: `−0.25`, or `−0.1` when a correlated leak exists
- Early return: `+0.1`
- Candidate confidence 'high': `+0.1`
- Freed via callee with no correlated runtime leak: returns `likely_false_positive (0.8)` immediately

Three precision gates can override the result:

1. freedViaCallee (`:222-233`): the pointer is handed to a callee that frees
   it → `likely_false_positive (0.8)`, unless a correlated runtime leak exists.
2. Dynamic exculpation (`:259-267`): the dynamic run was clean and there is
   no decisive static evidence → `likely_false_positive (0.8)`.
3. Strong-signal gate (`:274-285`): a "flagged" verdict that lacks a strong
   signal (`correlatedRuntimeLeak || structuralHigh || candidatePair?.status === 'unpaired' || pathSensitiveLeak || hasOwnershipIssue`)
   is downgraded to `uncertain`.

### 8.3. The LLM judge: a rubric, not a numeric threshold

The model picks one of the five labels according to the §3.2 rubric: a
runtime-linked leak ⇒ `confirmed_leak` at confidence ≥ 0.9; ownership transfer,
freed on all paths, or a clean run ⇒ `false_positive`. The `confidence` the model
assigns is not what gates the flag.

### 8.4. Consensus

`combineVerdicts` (`consensus-judge.ts:159-234`):

- The flag follows the rule in force: `majority` = `flagging*2 > n`;
  `weighted` = `flagW/total > 0.5`, where a vote that contradicts decisive
  dynamic evidence is multiplied by `×0.3`; `unanimous-to-flag` =
  `flagging === n`.
- The final label is the modal verdict within the agreeing cluster; a tie
  resolves to the less severe one.
- **Veto:** a strong heuristic exculpation (`false_positive` or
  `likely_false_positive` at confidence ≥ 0.75) vetoes the flag. The override
  can only *remove* a flag, never *add* one.

---

## 9. Summary table of active prompts and instructions

| Tier | Kind | Name / function | File | Lines | Format |
|---|---|---|---|---|---|
| A | System | `staticSubAgentSystemPrompt` | `domain/subAgentPrompts.ts` | 39-54 | tool-calling |
| A | User | `staticSubAgentUserMessage` | `domain/subAgentPrompts.ts` | 56-63 | n/a |
| A | Nudge | static completion | `orchestrator/workflowInvestigation.ts` | 211-216 | n/a |
| B | System | `dynamicWorkerSystemPrompt` | `domain/subAgentPrompts.ts` | 67-78 | tool-calling |
| B | User | `dynamicWorkerUserMessage` | `domain/subAgentPrompts.ts` | 80-90 | n/a |
| B | Nudge | dynamic completion | `orchestrator/workflowInvestigation.ts` | 275-278 | n/a |
| B | (no LLM) | `runDeterministicDynamic` | `domain/dynamicEvidence.ts` | 234-274 | n/a |
| B2 | System | `harnessWorkerSystemPrompt` (opt-in) | `domain/subAgentPrompts.ts` | 101-144 | tool-calling |
| B2 | User | `harnessWorkerUserMessage` | `domain/subAgentPrompts.ts` | 151-174 | n/a |
| B2 | Nudge | harness completion | `orchestrator/workflowInvestigation.ts` | 402-405 | n/a |
| D | System (single bundle) | `SYSTEM_PROMPT` | `domain/llmJudge.ts` | 51-56 | JSON object |
| D | System (batch, every call) | `BATCH_SYSTEM_PROMPT` | `domain/llmJudge.ts` | 61-67 | JSON array |
| D | Caller (single-shot batch) | `judgeBundlesBatched` + `buildBatchUserMessage` | `domain/llmJudge.ts` | 482, 383 | JSON array |
| D | Caller (consensus, round batch) | `judgeBundlesConsensusBatched` | `domain/llmJudge.ts` | 543 | JSON array × N rounds |
| D | Parse | `parseVerdict` / `parseBatchVerdicts` (+ NDJSON fallback) | `domain/llmJudge.ts` | 202, 278, 229 | n/a |
| D | Logic | `isBorderline` / `shouldEscalate` | `domain/llmJudge.ts` | 654, 672 | n/a |
| D | Logic | `combineVerdicts` (consensus) | `common/analysis/consensus-judge.ts` | 159-234 | n/a |
| n/a | Tool desc | done tools and `read_file` | `subAgentPrompts.ts` / `readFileTool.ts` | 17-26 / 17-43 | schema |
| n/a | Tool desc ×11 | static MCP | `static-analyzer/src/mcp/static-mcp-server.ts` | 38-127 | schema |
| n/a | Tool desc ×9 | dynamic MCP | `dynamic-analyzer/src/mcp/dynamic-mcp-server.ts` | 41-93 | schema |
| n/a | Notice | compaction / nudge / elided | `agent-core/src/loop.ts`, `compaction.ts` | 103,174 / 66 | n/a |
| POLICY | System | `allocatorProfileSystemPrompt` | `domain/allocatorProfiler.ts` | 90-103 | JSON only |
| POLICY | System | `strategistSystemPrompt` | `domain/strategist.ts` | 72-80 | JSON only |

---

## 10. Compact summary: the prompts on their own

### 10.1. The STATIC SUB-AGENT prompt (Stage A)

Purpose: the sub-agent gathers static evidence for each candidate.

File: `subAgentPrompts.ts:39-54`

```text
You are a STATIC-ANALYSIS evidence-gathering sub-agent for C/C++ memory leaks.
You do NOT record verdicts — your only job is to RUN the static tools so the system can collect structured evidence for each candidate, then call `done_static`.

For EACH candidate in your list, gather evidence:
- `functionSummary` (filePath, functionName) — alloc/free balance + leaky exit paths.
- `pathConstraints` (filePath, lineNumber of the allocation) — feasible leaking paths.
- `astScan` (filePath) — structural patterns + early returns.
- `ownershipConventions` (filePath) — ownership-transfer / missing-free conventions.
- `read_file` to inspect the source and, for interprocedural cases (a function returning an allocation), follow the caller.

Efficiency: you MAY call several of these read-only tools in a SINGLE turn — they run in parallel. The repository root is ${repoPath}.
When you have run the static tools for EVERY candidate in your list, call `done_static`. Do NOT reply with prose — only tool calls advance the work.
```

### 10.2. The DYNAMIC WORKER prompt (Stage B)

Purpose: the worker builds the project with a sanitizer and runs it;
findings are captured automatically.

File: `subAgentPrompts.ts:67-78`

```text
You are a DYNAMIC-ANALYSIS sub-agent for C/C++ memory leaks. Build the project ONCE with a sanitizer, run it under a sanitizer, then call `done_dynamic`.

1. `read_file` the Makefile / CMakeLists.txt / build script under ${repoPath} to learn how it builds. A hint build command was provided: `${buildCommand}`.
2. `buildTarget` (projectPath=${repoPath}, buildCommand = a clang command with sanitizer flags). Prefer LeakSanitizer (`-fsanitize=leak -g -O0`) — it reports at exit and never aborts mid-run.
3. Run the binary with `lsanRun` (or `asanRun` / `valgrindMemcheck`).

The system CAPTURES every finding from your sanitizer runs AUTOMATICALLY and attaches it to the matching candidate — you do NOT record evidence yourself. Your only job is to get a successful sanitizer run.
Build at most ONCE and run each dynamic tool at most once. If a build or sanitizer fails twice, stop and call `done_dynamic`. When a sanitizer has run, call `done_dynamic`. Do NOT reply with prose.
```

### 10.3. The LLM JUDGE prompt (Stage D)

Purpose: judge borderline allocations, batched at up to 12 candidates per
call, returning a JSON verdict (an array for a batch, an object when the batch
holds a single candidate).

File: `llmJudge.ts:51` (`SYSTEM_PROMPT`, single bundle) and `llmJudge.ts:61`
(`BATCH_SYSTEM_PROMPT`, used for every real call, including `CONSENSUS_N=1`)

The full text and its changelog are in §3.2. They are kept in one place only, so
this section cannot drift out of sync the way an earlier version of it did.

### 10.4. The ALLOCATOR PROFILER prompt (policy tier)

Purpose: discover the project's custom allocators and deallocators.
One-shot, temperature 0.

File: `allocatorProfiler.ts:90-103`

```text
You are a C/C++ memory-management API analyst. Given header/source excerpts from ONE project, identify that project's CUSTOM allocation and deallocation functions — the wrappers/replacements for malloc/free: factory constructors and duplicators (`*_new`, `*_create*`, `*_alloc*`, `*_dup`, `*_clone`, `*_copy` that return owned memory) and their matching releases (`*_free`, `*_delete`, `*_destroy`, `*_release`, `*_unref`, `*_close`, pool/arena destructors) — plus any ownership conventions a leak checker must know.
Respond with a JSON object ONLY (no prose), in this exact shape:
{"allocators":["..."],"deallocators":["..."],"reallocators":["..."],"ownershipNotes":["..."],"confidence":0.0-1.0,"explanation":"..."}
Rules:
- Be EXHAUSTIVE on allocators: list EVERY function that returns newly-owned memory — ALL constructors/factories (`*_New*`, `*_Create*`), duplicators (`*_Duplicate`, `*_dup`, `*_clone`, `*_copy`), and parsers/printers/serializers that return an owned buffer (`*_Parse*`, `*_Print*`). Do not stop at a few examples — include the whole family even if it is long.
- Use EXACT function names as they appear in the code.
- Include INTERNAL/static helpers too (e.g. a static `cJSON_malloc`/`*_New_Item` wrapper, a private `*_strdup`), not only the public API — leaks often flow through them.
- Include MACRO allocators/deallocators: a `#define MY_ALLOC(n) malloc(n)` (or `#define FREE_OBJ(p) ...`) is an allocator/deallocator named `MY_ALLOC`/`FREE_OBJ` — list the macro name.
- Do NOT include plain libc malloc/calloc/realloc/free/strdup — the engine already knows those. Only the project's CUSTOM names.
- allocators/reallocators RETURN newly-owned heap memory the caller is responsible for; deallocators FREE or consume it.
- ownershipNotes: short, project-specific rules (transfer vs borrow, refcounting, pool/arena "free the pool, not each object", "X skips items flagged Y", a constructor that steals its argument). Empty array if none.
- Precision over recall: if unsure a name is an allocator/deallocator, OMIT it.
```

### 10.5. The STRATEGIST prompt (policy tier)

Purpose: choose an analysis plan (dynamic on or off, judge mode, static
depth). One-shot, temperature 0.

File: `strategist.ts:72-80`

```text
You are the STRATEGIST for a C/C++ memory-leak analyzer. Given a project's metadata + memory-API profile, choose an analysis plan by SELECTING among the engine's existing deterministic capabilities — you do not invent analysis. Respond with a JSON object ONLY:
{"runDynamic": true|false, "judge": "single"|"consensus", "staticDepth": "shallow"|"full", "rationale": "..."}
Guidance:
- runDynamic: run sanitizer (LeakSanitizer) dynamic analysis ONLY if the project is plausibly buildable (a build system is present) AND dynamic coverage would help. If there is NO build system, set false — building is impossible, so skip the expensive dynamic stage (no recall lost).
- judge: "consensus" (slower, more robust) for projects whose ownership is subtle — heavy smart-pointer / refcounting / C++; else "single".
- staticDepth: "shallow" (function summaries only) for tiny or trivial projects; "full" (path constraints + ownership + interprocedural) for larger or control-flow-heavy ones.
Be decisive; prefer cheaper plans when they lose no recall.
```

### 10.6. Static tool descriptions (the five content-capable tools)

| Tool | `description` (verbatim) |
|---|---|
| `candidateScan` | *Scan a file for allocation sites (malloc, calloc, realloc, strdup, new). Optionally supply per-project factory allocators / custom deallocators (≈ LAMeD AllocSource/FreeSink) so wrapper-named allocators (e.g. cJSON_Duplicate) become candidates.* |
| `astScan` | *AST-based structural analysis for memory leak patterns* |
| `functionSummary` | *Summarize a function: alloc/free balance, local vars, calls. Optionally supply per-project allocators/deallocators so factory-allocated vars are paired.* |
| `pathConstraints` | *Analyze path constraints and feasible paths around an allocation. Optionally supply per-project allocators/deallocators so factory allocations are tracked on exit paths.* |
| `ownershipConventions` | *Detect ownership-transfer conventions in a file* |

Filtered by `CONTENT_CAPABLE_TOOLS` (`mcpToolPlan.ts:103-109`). The remaining
static tools (`indexFiles`, `callGraph`, `interproceduralFlow`,
`ownershipSummary`, `scanBuildRun`, `scanBuildGetReport`) are not exposed to the
TUI because they require a shared filesystem mount.

### 10.7. Dynamic tool descriptions (nine tools)

| Tool | `description` (verbatim) |
|---|---|
| `buildTarget` | *Build the project with sanitizer-instrumented compiler flags* |
| `valgrindMemcheck` | *Run Valgrind Memcheck for detailed leak analysis* |
| `valgrindGetReport` | *Retrieve a normalized Valgrind report* |
| `valgrindListFindings` | *Query Valgrind findings with optional filters* |
| `valgrindCompareRuns` | *Compare two Valgrind analysis runs* |
| `asanRun` | *Run the binary under AddressSanitizer for leak detection* |
| `lsanRun` | *Run the binary under LeakSanitizer* |
| `runBinary` | *Run a binary without instrumentation* |
| `listRuns` | *List stored dynamic analysis runs* |

### 10.8. The domain tool (`read_file`) and the done tools

| Tool | `description` (verbatim) |
|---|---|
| `read_file` | *Read a source file from the repository (path relative to the repo root, or an absolute path inside it). Returns up to 16000 characters.* |
| `done_static` | *Finish static evidence gathering for this group of candidates.* |
| `done_dynamic` | *Finish dynamic evidence collection.* |
