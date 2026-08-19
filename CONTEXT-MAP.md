# Available context map

| Source | Location | Authority | Freshness | Load when |
| --- | --- | --- | --- | --- |
| Measured architecture | `architecture/e2b-shape.md` | Canonical (this model) | 2026-08-18 | Any structural question |
| Editing envelope | `editing/knowledge-editing-envelope.md` | Canonical (measured sweep) | 2026-08-18 | Choosing layer/alpha |
| The six defects | `tooling/larql-gemma4-fixes.md` | Canonical (our fixes) | 2026-08-18 | Debugging LARQL on Gemma 4 |
| Verified pipeline | `tooling/pipeline.md` | Canonical (run end to end) | 2026-08-18 | Running an edit |
| Patched LARQL | `github.com/fbremer/larql` branch `gemma4-e2b-fixes` | Canonical (code) | 2026-08-18 | Building the tool |
| Upstream LARQL | `github.com/chrishayuk/larql` | Canonical (upstream) | live | Checking whether a fix landed |
| Base weights | `~/.cache/huggingface/hub/models--google--gemma-4-e2b-it` | Canonical (weights) | 2026-08-18 | Any compile |
| Working 26B vindex | `~/.cache/larql/26b-a4b.vindex` | Canonical (verified working) | 2026-08-18 | Any decoupled-serving run |
| 26B safetensors | `~/.cache/huggingface/hub/models--google--gemma-4-26B-A4B-it` | Canonical (extract base) | 2026-08-18, 48 GB | Re-extracting |
| Our patches | `~/src/larql-fork` branches `gemma4-compile-fixes-v2` (3 commits) and `gguf-gemma4-layer-types` (1) | **Active** — rebased on upstream main, defects re-verified live, before/after recorded | 2026-08-18 | Touching the compile path, or submitting upstream |
| Decoupling results | `decoupling/replication-attempt.md` | Canonical (measured here) | 2026-08-18 | Any distributed-serving question |
| 26B slices (published) | `~/.cache/larql/26b-client`, `26b-server` | Reference (incomplete) | 2026-08-18 | Comparing against our own build |
| Method | `KIT-METHOD.md` | Supporting | 2026-08-10 | Managing this context set |

## Source precedence

Measurement on this machine beats documentation. Documentation beats a video
transcript. A vendor's model card beats inference from behaviour.

## Retrieval triggers

- A number is needed that is not in `architecture/`.
- An edit behaves outside the recorded envelope.
- Upstream may have fixed something we patched.

## Deliberate exclusions

| Source | Why | Reconsider when |
| --- | --- | --- |
| Gemma 3 material | Different architecture; its constants misled this work repeatedly | Never for E2B parameters |
| Decoupled serving | Sends residual streams off-machine; residuals decode to text | A network-boundary policy is decided deliberately |
