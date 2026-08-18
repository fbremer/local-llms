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
