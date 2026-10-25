# CLeak: prompts and frozen evaluation configurations

Supplementary material for:

> CLeak: A Hybrid Approach to C/C++ Memory Leak Detection with LLM-Driven
> Policy and Deterministic Execution.
> Dang Dung Le, Trong-Nghia To, The Duy Phan, Van-Hau Pham.
> NSS 2026, Singapore. Springer LNCS.

This repository holds the two things the paper's reproducibility footnote
promises: every LLM prompt the system issues, and the frozen per-configuration
settings behind the nine-baseline ablation. It is documentation rather than a
runnable release; the scanner itself is not published here.

## Contents

| Path | What it is |
|---|---|
| [`PROMPTS.md`](PROMPTS.md) | Every prompt the system sends, by pipeline stage, with the output schema each one must satisfy. |
| [`configs/*.yaml`](configs/) | The nine baseline configurations B1 to B7. Their capability flags are unchanged since the reported sweep; only B7's filename and prose description were revised afterwards. |

### Section map

Prompt texts are reproduced verbatim as sent to the model. The surrounding
commentary is ours.

| Section | Prompt |
|---|---|
| 0 | Overview: the four tiers and where the LLM is called |
| 0.5.1 | Allocator profiler (policy discovery, one call per project) |
| 0.5.2 | Strategist (the bounded planner; one call per case in B6a) |
| 1.1 to 1.4 | Stage A, the static fan-out sub-agent: system prompt, user message, tools, completion nudge |
| 2.1 to 2.6 | Stage B, the dynamic worker, including the deterministic no-LLM branch (2.1) |
| 3.1 to 3.4 | Stage D, the borderline LLM judge: escalation gate, system prompt, user message, parsing |
| 4 | Batch judging and the consensus judge |
| 8.1 to 8.4 | Decision boundaries: label-to-boolean collapse, heuristic judge, judge rubric, consensus |
| 10.1 to 10.8 | Compact restatement of each prompt's original text |

Section 3 holds the nine-rule priority rubric and the five-label JSON schema
described in the paper.

## The nine configurations

Each YAML sets five independent capability flags, which the sweep driver maps
onto engine switches. `runs: 3` marks the configurations whose results the paper
reports as mean plus or minus standard deviation over three repetitions.

| File | Paper ID | Configuration |
|---|---|---|
| `b1-static-only.yaml` | B1 | Static evidence only, heuristic judge |
| `b2-dynamic-only.yaml` | B2 | Dynamic evidence only, heuristic judge |
| `b3-rule-ensemble.yaml` | B3 | Static and dynamic, rule-based ensemble |
| `b4-llm-static.yaml` | B4 | LLM judge over static evidence |
| `b5-llm-dynamic.yaml` | B5 | LLM judge over dynamic evidence |
| `b6-llm-all.yaml` | B6 | LLM judge over static and dynamic evidence |
| `b6a-planner-only.yaml` | B6a | B6 plus the bounded planner |
| `b6b-toolselector-only.yaml` | B6b | B6 plus agentic tool selection |
| `b7-full-adaptive.yaml` | B7 | Planner and tool selection together |

## Runtime settings across the reported sweeps

Each run's stored provenance records the provider, the model name and the
temperature. The remaining settings below live in the scanner's configuration
and are not written into per-run provenance, so they are reported as the
defaults in force rather than as per-run facts.

| Setting | Value | Recorded per run? |
|---|---|---|
| Model (main sweeps) | `deepseek-v4-flash`, via an OpenAI-compatible gateway | yes |
| Model (label-masking ablation) | `deepseek-v4.1-flash`, same gateway | yes |
| Temperature (planner and judge) | 0 | yes |
| Output mode | JSON-constrained | no |
| Heuristic judge bands | 0.7 / 0.4 | no, fixed in source |
| LLM-escalation borderline window | [0.35, 0.7] | no, fixed in source |
| Consensus (when enabled) | K = 3, confidence-weighted, down-weight 0.3, veto at 0.75 | yes |
| Repetitions per LLM-assisted configuration | 3 | yes |

The gateway exposes model aliases rather than provider-side snapshot
identifiers, so the paper reports the two aliases above and does not claim a
pinned snapshot.

## Prompt provenance, and one difference in framing

The nine-rule judge rubric published here is byte-identical to the one in force
at the commit that produced the reported sweep. What changed in the code
afterwards is the surrounding machinery, not the rubric.

That machinery matters in one place. The main sweeps sent one bundle per request
under the single-bundle system prompt (section 3.2). The label-masking ablation
ran later and used the batch system prompt (section 4) with a chunk size of one.
The two ablation legs are therefore comparable with each other, but neither is
directly comparable with the main ablation table. The paper states this.

## Label masking

The ablation in the paper's RQ1 rewrites every identifier containing `bad` or
`good` to an opaque `fn_<n>` token in the text sent to the judge, leaving the
deterministic evidence pipeline untouched. It is gated behind the
`CLEAK_MASK_LABELS` environment variable and is off by default, since the
rewrite is only meaningful on Juliet-style corpora. The masking is partial:
file and directory names naming the CWE remain in the prompt. The paper reports
that caveat alongside the result.
