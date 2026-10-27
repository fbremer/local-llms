# Available context map

| Source | Location | Authority | Freshness | Load when | State |
| --- | --- | --- | --- | --- | --- |
| Measured architecture | `architecture/e2b-shape.md` | Canonical for this model | 2026-08-18 | Any structural question | On demand |
| Editing envelope | `editing/knowledge-editing-envelope.md` | Canonical measured sweep | 2026-08-18 | Choosing a layer or alpha | On demand |
| Edit results | `editing/e2b-results.md` | Canonical measured run | 2026-08-18 | Quoting an installed fact or a control | On demand |
| The six defects | `tooling/larql-gemma4-fixes.md` | Canonical for the defects | 2026-08-18 | How E2B fails in LARQL | On demand |
| Patch status | `patches/README.md` | Canonical for submission status | 2026-08-18 | Applying, quoting, or submitting the patches | On demand |
| Review copies of the patches | `patches/` | Canonical snapshot in this repo | 2026-08-18, against `e96275c8` | Reading the diff | On demand |
| Live patch branches | `~/src/larql-fork` branches `gemma4-compile-fixes-v2` and `gguf-gemma4-layer-types` | Canonical code that was run | 2026-08-18 | Changing the compile path | On demand |
| Verified pipeline | `tooling/pipeline.md` | Canonical end-to-end recipe | 2026-08-18 | Running an edit | On demand |
| Upstream LARQL | `github.com/chrishayuk/larql` | Canonical upstream | Live, last checked here at `e96275c8` | Whether a fix has landed since the check | On demand |
| Base weights | `~/.cache/huggingface/hub/models--google--gemma-4-e2b-it` | Canonical weights | 2026-08-18 | Any compile | On demand |
| Working 26B vindex | `~/.cache/larql/26b-a4b.vindex` | Canonical verified artifact | 2026-08-18 | Any decoupled-serving run | On demand |
| 26B safetensors | `~/.cache/huggingface/hub/models--google--gemma-4-26B-A4B-it` | Canonical extract base | 2026-08-18, 48 GB | Re-extracting | On demand |
| Published 26B slices | `~/.cache/larql/26b-client`, `26b-server` | Incomplete reference | 2026-08-18 | Comparing against the local build | Reference |
| Decoupling results | `decoupling/replication-attempt.md`, `decoupling/throughput-controls.md`, `decoupling/operating-guide.md` | Canonical measurements on this machine | 2026-08-19 | Any serving or throughput question | On demand |
| Decoupling source notes | `sources/decoupled-attention.md` | Supporting | 2026-08-18 | What the demo claimed | On demand |
| Model quality | `evaluation/summarisation-quality.md` | Canonical blind judging | 2026-08-20 | Any ranking, model pick, or summarisation claim | On demand |
| Ollama operating facts | `ollama/operating-facts.md` | Canonical for this machine | 2026-08-19 | Any Ollama measurement or memory problem | On demand |
| Muse-Glimmer-30B | `ollama/muse-glimmer-30b.md` | Canonical measured run notes | 2026-08-19 | Running or re-tuning the 30B | On demand |
| Working muse tags | `local/muse-glimmer:30b-{iq3-text,q2-text,iq3}` | Canonical verified tags | 2026-08-19 | Loading a tag that was shown to run | On demand |
| Decision history | `DECISIONS.md` | Canonical for choices | 2026-10-27 | Why a rule is in force | On demand |
| Method | `KIT-METHOD.md` | Method | 2026-08-10 | Changing this working set | On demand |

## Source precedence

A measurement on this machine beats documentation. Documentation beats a video transcript. A vendor model card beats a guess from behaviour.

The next session may assume only the results and protocols in `CURRENT.md`. An interpretation carried forward there is a hypothesis, with the result that would contradict it, and is not a premise. An Active entry in `DECISIONS.md` does not add further assumptions. The August notes still put a result and an interpretation in one sentence. Split it before citing it.

The 2026-08-20 evaluation beats the five-model ranking of 2026-08-19. `patches/README.md` beats any sentence that says the patches are submitted or ready for a later upstream. If `patches/` and `~/src/larql-fork` disagree, the fork is the code that was run and `patches/` is the review snapshot. Neither has been checked against upstream past `e96275c8` in this repo.

## Retrieval triggers

- A number is needed that is not in `architecture/`.
- An edit behaves outside the recorded envelope.
- A ranking, a "best quant," or a summarisation claim is about to be quoted.
- A patch is about to be applied or described to someone else.
- An Ollama figure looks wrong, or a model may still be resident.
- Upstream may have moved since `e96275c8`.

## Deliberate exclusions

| Source or category | Why excluded | Reconsider when |
| --- | --- | --- |
| Gemma 3 material | Different architecture. Its constants misled this work. | Never, for E2B parameters |
| Sending residuals off this machine | Residuals decode to text | A network-boundary policy is decided |
| The five-model order of 2026-08-19 | Superseded by `evaluation/summarisation-quality.md` | Do not restore it |
| `judging-eve/evicted/` | Named by the evaluation as the replicate outputs. The directory is not in this repo | The files are added, and the map is updated to their real path |

## Known gaps

The evaluation names `judging-eve/evicted/` as the place the replicate outputs were kept, and that directory is not in this repo. The compile patches are not submitted, and this repo has no check of upstream after `e96275c8`. Throughput under the 2026-08-20 judging regime (full offload, `q4_0` KV) was not measured. The harmony stop-list pull request has not been started.
