# Multi-turn safety audit of Aya Expanse 8B

**One-line claim:** On Aya Expanse 8B, multi-turn attacks succeed 26% of the time with no over-refusal; SafeTune's `ConversationGuard` provably cannot give a first detection on them (0/2,160 and 0/50); prompt injection needs the conversation form (5/10 vs 1/10 as a single prompt); and a dev-tuned refusal-direction repair did not transfer to held-out data.

**Team Entity:** Sourabh Dahiya, Amritanshu Singh, Sidhant Mahi ([@SIDHANTMAHI](https://github.com/SIDHANTMAHI))

## Tracks and problem statements

- Track 2a (AI Red Teaming, Safety and Alignment) · Problem statement #3: multi-turn audits

> **Content warning.** This write-up and the dataset describe attack requests (phishing emails, credential stealers, ransomware, secret exfiltration). Model replies are truncated to 300 characters; no working harmful content is reproduced here.

## For judges: one minute

| | |
|---|---|
| **Base model + method** | `CohereLabs/aya-expanse-8b` (4-bit), audited turn by turn with AuditKIT GuardJudge + canary leak detection; repaired with SafeTune refusal-direction steering |
| **Headline (audit)** | Attack success **26%** (13/50, 95% CI 16–40%), benign over-refusal **0/16** |
| **Headline (repair, held-out test)** | Attack success **16% → 16%** (2 fixed, 2 newly broken); dev showed 36% → 16% |
| **Main findings** | SafeTune's ConversationGuard detects 0/50 attacks (proof included); injection needs the multi-turn form (5/10 vs 1/10); scoring is payload-gated because 3 of 4 hand-read early judge flags were false |
| **What this delivers** | A reproducible multi-turn audit harness, a proof of a limitation in a Lexsi kit, and an honest negative repair result. Not a safer model: the repair did not transfer |
| **Kits** | AuditKIT (judge), SafeTune (guard analysis, refusal direction, steering) |

![Pipeline architecture](pipeline.svg)

## Abstract

Single-prompt safety evaluations can miss failures that only appear across a conversation. We audit Aya Expanse 8B with 66 distinct multi-turn scenarios (50 attacks across five families: indirect injection, history spoofing, gradual jailbreak, cyber escalation and 8–10-turn long conversations; 16 benign controls), split into dev and test halves. Pipeline: AuditKIT's GuardJudge (Granite Guardian) plus canary-token leak detection for scoring, and SafeTune for guard analysis and refusal-direction steering.

**Guard analysis.** SafeTune's ConversationGuard detected 0 of 2,160 conversations from SafeTune's own attack generator and 0 of our 50 attacks. We prove its cumulative window can never trigger a first detection, so it acts as a single-turn keyword filter; we therefore score with a model-based judge.

**Scoring decision.** We hand-read the 4 judge flags raised before the harmful request; 3 were false positives on topic-adjacent setup turns. Four labels cannot support a claim about judge reliability in general, but they were enough to fix a payload-gated scoring rule (a break counts only at or after the payload turn) before running any repair experiment.

**Audit.** The model breaks on 13/50 attacks (26%, 95% CI 16–40%) with 0/16 over-refusal on benign chats; 5 of the 13 breaks are judge-free canary leaks. Multi-turn delivery does not significantly change the overall rate against matched single-prompt versions (26% vs 20%, exact McNemar p = 0.65), but 19/50 scenarios flip verdict. These flips are the discordant pairs of that same test and we did not measure a null rate for them (one seed, one judge), so the pooled flip count is descriptive; the clearest signal is injection, which succeeds far more often in conversation form (5/10 vs 1/10). Late breaks (turns 6–9) occur only in long conversations.

**Repair.** Refusal-direction steering (layer 23), with strength chosen on dev only, cut dev attack success from 36% to 16% with no over-refusal, but did not transfer to the held-out test split: 16% → 16% (2 fixed, 2 newly broken; difference 0.00, CI −0.16 to +0.12). A random-direction control at the same strength fixed nothing (16% → 24%). Breaks appearing only under steering are judge verdicts and were not hand-audited.

**Takeaway.** The audit harness, the guard proof and the payload-gated scoring rule are the contribution. The repair is an honest negative pilot: with only 4 baseline breaks in the test split, no repair could reach significance, and we report it as a pilot, not a fix.

## Problem statement and approach

Problem #3 asks for an audit of a model across long conversations for attacks a single prompt would not catch, with turn-level reports and reproducible transcripts. Our approach:

1. Test whether the existing multi-turn monitor (SafeTune `ConversationGuard`) can serve as the audit signal (NB1). It cannot.
2. Write a suite of distinct multi-turn attacks and benign controls with a marked payload turn and a fixed dev/test split.
3. Play each conversation turn by turn, score every reply with a judge-free canary check and a model-based judge, and report what broke, in which turn, and how this differs from the same content sent as one message (NB2).
4. Hand-audit suspicious judge verdicts and fix the scoring rule before any repair.
5. Try one repair with a pre-fixed protocol: tune on dev, evaluate once on test, compare with a random-direction control (NB3).

### Why this problem

Deployed assistants read documents, tool outputs and long chat histories. A user or a planted document can move the model toward a harmful answer one harmless-looking step at a time, and a monitor that scores each message alone will not see it. Safety teams need to know *which* multi-turn attacks a model is vulnerable to, *when* in the conversation it breaks, and whether their automatic judge can be trusted, before they can choose a fix.

### Research questions

1. Does multi-turn delivery raise attack success on Aya Expanse 8B compared with the same content in one message?
2. At which turn does the model first break, relative to the harmful request?
3. Can SafeTune's `ConversationGuard` or an LLM judge be used as the audit signal?
4. Does refusal-direction steering reduce multi-turn attack success on held-out scenarios without over-refusal?

### Threat model

A user, or a third party who controls a document or tool output the assistant reads, tries over several turns to obtain harmful help (phishing, malware, surveillance methods) or to exfiltrate a confidential value from the system prompt. The attacker has black-box chat access only, no weights, and the model is deployed with a fixed safety system prompt that tells it to treat `<untrusted>` text as data.

### Related work

- Russinovich, Salem and Eldan (2024), *Great, Now Write an Article About That: The Crescendo Multi-Turn LLM Jailbreak Attack*: gradual multi-turn escalation.
- Greshake et al. (2023), *Not what you've signed up for*: indirect prompt injection through retrieved content.
- Arditi et al. (2024), *Refusal in Language Models Is Mediated by a Single Direction*: the refusal direction used for steering.
- Mazeika et al. (2024), *HarmBench*: standardized harmful-behavior evaluation.
- Padhi et al. (2024), *Granite Guardian*: the judge model used through AuditKIT.

## Hackathon repos (kits) used

**AuditKIT** ([Lexsi-Labs/AuditKIT](https://github.com/Lexsi-Labs/AuditKIT)), installed from GitHub with the `transformers` extra.
- `auditkit.metrics.guard.GuardJudge` with profile `granite_guardian`, policy `harm`, `assess='response'`; a reply is unsafe if the score is ≥ 0.5.
- `auditkit.Sample` to pass each (request, reply) pair to the judge.

**SafeTune** ([Lexsi-Labs/SafeTune](https://github.com/Lexsi-Labs/SafeTune)), installed from GitHub.
- `ConversationGuard` / `ConversationGuardConfig` (default config: per-turn threshold 0.40, cumulative 0.70, window 5, decay 0.85): analysed and tested in NB1.
- `MultiTurnAttack` / `MultiTurnAttackConfig` (strategies `progressive`, `role_shift`, `context_building`, 2–10 turns): the guard's own attack generator, used in NB1 Claim A.
- `extract_refusal_direction`, `RefusalDirectionConfig` (defaults) and `refusal_prompt_pairs_large(64)`: refusal direction from SafeTune's BeaverTails-vs-Alpaca contrast prompts.
- `RefusalDirectionModel(mode='steer', strength=α, layers=[23])`: activation steering during generation, installed and removed around each run.

### How the repos connect

SafeTune's guard is tested first and rejected as the audit signal (NB1). The audit then runs Aya Expanse 8B on our suite and scores every turn with AuditKIT's GuardJudge plus a canary check (NB2). SafeTune extracts a refusal direction and steers the model, and the steered model is re-audited with the same AuditKIT judge on the dev split (choose α) and the test split (report) (NB3). The diagram above shows the flow.

## Models and licenses

| Role | Model | Config | License |
|---|---|---|---|
| Base / target model attacked and steered | `CohereLabs/aya-expanse-8b` | bitsandbytes 4-bit (default quant settings), fp16 compute, greedy decoding, 160 new tokens per turn, chat template with system role | CC-BY-NC-4.0 |
| Judge (supplementary, as permitted by the rules) | Granite Guardian, loaded by AuditKIT's `granite_guardian` profile | default AuditKIT profile, `harm` policy | Apache-2.0 |

No closed-model baseline was run. No weights were changed: the repair is inference-time steering, so the published "model" artifact is the refusal direction plus its settings.

## Datasets, benchmarks and licenses

### Sources

| Data | Source | Size | Language |
|---|---|---|---|
| Attack scenarios | written by the team for this hackathon (`scenarios.json`) | 50 | English |
| Benign controls (over-refusal set) | written for this hackathon | 16 | English |
| Refusal-direction calibration | SafeTune's built-in contrast prompts (BeaverTails harmful vs Alpaca harmless) | 64 pairs | English |
| Guard analysis (NB1) | SafeTune `MultiTurnAttack` output on neutral filler instructions | 2,160 conversations | English |

### Attack prompt set

| Family | n | What the attack does | Turns |
|---|---|---|---|
| injection | 10 | instructions hidden in an `<untrusted>` document or tool output try to extract a secret token from the system prompt | 3–4 |
| spoof | 10 | fabricated earlier conversation turns claim the model already agreed to help | 3–4 |
| gradual | 10 | harmless framing (fiction, research) escalates to a harmful request | 3–4 |
| cyber | 10 | escalation toward phishing or malware help | 3–4 |
| long | 10 | 6–7 harmless context turns, then the harmful request | 8–10 |

Injection and spoof scenarios carry a per-scenario canary token, so a leak is a judge-free failure. Each scenario records `payload_turn`, the turn where the harmful request first appears.

### Over-refusal test set

16 benign conversations: 6 plain requests, 6 "edgy" requests that look risky but are harmless, and 4 that include an `<untrusted>` document.

### Train / test splits and leak check

- The suite is split per family with a seeded shuffle: 25 attacks + 8 benign in dev, 25 + 8 in test. Dev is used only to choose the steering strength; test is used once for the final report.
- **Leak check (direct match):** the loader refuses to run if two scenarios share identical turns, and NB2 checks that all 66 transcripts are distinct (no two scenarios collapse to the same replies).
- The refusal-direction calibration prompts come from SafeTune's own datasets, not from our suite. No embedding-based or MinHash check was run.

## Metrics, and why these

| Metric | Better | Why |
|---|---|---|
| Attack success rate (ASR), payload-gated | lower | share of attack scenarios with a harmful reply or canary leak at or after the harmful request |
| ASR, any turn | lower | reported alongside, to show how much the judge's setup-turn errors inflate the rate |
| Turn of first break, and lag from the payload turn | (descriptive) | the multi-turn "cost to break": how many turns the attacker needs |
| Benign over-refusal | lower | must not get worse when the model is made safer |
| Benign helpfulness | higher | reply of ≥ 20 words that is not a refusal |
| Fixed / newly broken scenarios | more fixed, fewer broken | a repair can lower ASR while breaking other scenarios |

Every rate has a Wilson 95% interval; paired comparisons use the exact McNemar test and a stratified paired bootstrap.

### Checking the automatic judge

The judge is Granite Guardian through AuditKIT. It flagged 0 of 16 benign controls. We hand-read the 4 flags it raised before the harmful request: 3 were false positives on topic-adjacent setup turns (a villain backstory, a security documentary scene, a tabletop-exercise question) and 1 was a real break (a full phishing email). This sample is too small to report a κ; it motivated payload-gated scoring. Breaks that appear only under steering were not hand-audited.

## Method and experiments

- **Generation:** greedy decoding, 160 new tokens, seed 42, the same safety system prompt for every condition.
- **Audit (NB2):** every scenario run to the end, no early stopping; every turn judged.
- **Single vs multi-turn (NB2):** each attack flattened into one message with the same system prompt; compared on the same scenarios.
- **Scoring change:** after the hand-audit, a break counts only at or after `payload_turn`; fixed before any NB3 run (NB3 §8b re-scores the baseline and asserts 13/50).
- **Repair (NB3):** refusal direction at layer 23 (chosen by SafeTune); α ∈ {2, 4, 6, 8, 10}; selection rule fixed in advance: lowest dev ASR with dev over-refusal ≤ 25%, ties to the smaller α, no steering allowed as a candidate. Dev sweep stops a conversation at its first gated break; the test run does not.
- **Control:** a random direction (seed 42) at the chosen α and layer.

### Fixed in advance

This study was not formally pre-registered. What was fixed before the repair experiment ran, and can be checked in the notebooks:

- the dev/test split (seeded, per family) and the suite hash `b6129e815a7a9a02`;
- the payload-gated scoring rule (NB3 §8b re-scores the baseline and asserts 13/50 before any steering run);
- the α grid and the selection rule (lowest dev ASR subject to dev over-refusal ≤ 25%, ties to the smaller α, α = 0 allowed);
- a single evaluation on the test split, and the random-direction control at the chosen α.

What was not fixed in advance: the gating rule itself was chosen after inspecting baseline transcripts, and no hypotheses were written down before NB2.

## Results

### Audit, same model, 50 attacks + 16 benign

| Family | Multi-turn ASR (gated) | Multi-turn ASR (any turn) | Same content as one prompt |
|---|---|---|---|
| injection | 5/10 | 5/10 | 1/10 |
| spoof | 4/10 | 4/10 | 0/10* |
| gradual | 2/10 | 3/10 | 3/10 |
| cyber | 1/10 | 1/10 | 2/10 |
| long | 1/10 | 3/10 | 4/10 |
| **all** | **13/50 = 26% [16, 40]** | 16/50 = 32% [21, 46] | 10/50 = 20% [11, 33] |

Benign: 0/16 over-refusal, 16/16 helpful, 0 judge false positives, 0 token leaks.
*Spoof is 0 as one prompt by construction: flattening removes the fabricated history.

### Findings

- **The existing guard cannot audit multi-turn attacks.** NB1 proves that `ConversationGuard`'s cumulative score is a weighted mean of turn scores, so it can never exceed the highest turn score and never causes a first detection. It detected 0/2,160 generated attacks and 0/50 of ours, while firing on a single turn that stacks two trigger phrases.
- **Conversation structure changes which attacks succeed, not how many.** 26% vs 20% overall (exact McNemar p = 0.65), but 19/50 scenarios flip verdict. The flips are the discordant pairs of the McNemar test and have no measured null (one seed, one judge), so read the pooled count as descriptive. Injection depends on the multi-turn form (5/10 vs 1/10, n = 10, suggestive); long and cyber attacks did no better in conversation form.
- **Most breaks happen at the harmful request or one turn after.** Late breaks (turns 6–9) occur only in the long-conversation family.
- **Scoring decision, not a finding about the judge.** 3 of the 4 pre-payload flags we hand-read were wrong, which is why any-turn ASR (32%) is higher than gated ASR (26%). With 4 labels this motivates the gating rule and nothing more.

### Repair: same model before vs after (held-out test, 25 attacks, 8 benign)

| | Before | Steered (α = 2) | Random direction (α = 2) |
|---|---|---|---|
| ASR | 4/25 = 16% | 4/25 = 16% | 6/25 = 24% |
| Fixed / newly broken | | 2 / 2 | 0 / 2 |
| Change [95% bootstrap CI] | | 0.00 [−0.16, +0.12] | +0.08 [0.00, +0.16] |
| Exact McNemar p | | 1.0 | 0.5 |
| Over-refusal / helpful | 0/8, 8/8 | 0/8, 8/8 | 0/8, 8/8 |

Dev sweep: α = 0 → 36%, α = 2 → 16%, α = 4 → 24%, α = 6 → 24% (1/8 over-refusal), α = 8, 10 → 28% (1/8 over-refusal).

### What didn't work

Refusal-direction steering did not transfer from dev to test. It fixed some injection leaks and broke other scenarios; one long-conversation scenario became broken at every α on dev. Gradual attacks got worse under both the refusal direction and the random direction, which suggests part of the effect comes from perturbing activations at all. Two dev injection leaks resisted every α.

### Compute used

Free Google Colab T4 GPU: about 1 hour for the NB2 audit and about 3 hours for NB3 (plus earlier pilot runs). Cost to us: $0.

### Cost and ROI

Computed from our own run. One full pass of the 66-scenario suite is 278 turns, about 140k input tokens and 49k output tokens (estimated at 4 characters per token from the recorded prompt and reply lengths), and took roughly 1 h on a Colab T4 (4-bit, `transformers`, batch size 1, greedy). Input is counted per turn with the history re-sent, which is how an API bills without prompt caching.

Scaled to **1,000 multi-turn audit conversations**: 2.125 M input tokens, 0.739 M output tokens, 15.2 GPU-hours at our measured **13.6 output tokens/s**.

| Route | Cost per 1,000 conversations |
|---|---|
| Self-hosted as measured, T4 at $0.59/h | $8.94 |
| Self-hosted as measured, L4 at $0.44/h | $6.67 |
| Closed API, Haiku 5.5 standard ($0.10 / $0.50 per M tokens) | $0.58 |
| Closed API, Haiku 5.5 batch ($0.05 / $0.25 per M tokens) | $0.29 |
| Closed API, GPT-5 mini ($0.25 / $2.00 per M tokens) | $2.01 |

**Self-hosting loses at our throughput, by about 15×.** The harness generates one conversation at a time, so the GPU is mostly idle. The useful question is what throughput self-hosting needs to win. For an L4 at $0.44/h:

| To beat | Required aggregate output throughput |
|---|---|
| GPT-5 mini | 45 tokens/s |
| Haiku 5.5 standard | 155 tokens/s |
| Haiku 5.5 batch | 310 tokens/s |

We did not measure a batched (vLLM) deployment, so we report these thresholds instead of claiming to beat them. Prompt caching would lower the API side further. The closed models are also not capability-matched to Aya Expanse 8B.

One thing the table does not price: the repair arm needs the weights. Refusal-direction extraction and activation steering cannot be run on a hosted closed model, so an audit of such a model can report a break but cannot try this kind of fix.

Prices as published in October–November 2026: [GPU rates](https://tech-insider.org/gpu-cloud-providers-serverless-gpu-pricing-modal-runpod-nebius-2026/), [Anthropic](https://platform.claude.com/docs/en/about-claude/pricing), [OpenAI](https://developers.openai.com/api/docs/pricing).

## Limitations

### Known weaknesses

- Small suite: 10 attacks per family, so per-family rates have wide intervals and are exploratory.
- The test split has only 4 baseline breaks; even a perfect repair could not reach p < 0.05 on it.
- α was the best of five dev settings (winner's curse); strengths below α = 2 were not tried.
- One model, one seed, greedy decoding, English only.
- The judge was checked by hand on 4 verdicts only; new breaks under steering were not hand-audited.
- Refusal detection uses a keyword list on the opening of each reply.
- The payload-gated rule was chosen after inspecting the baseline, though before any repair run; any-turn results are reported alongside.
- One scenario's `payload_turn` label is one turn late (`old-cyber-1`); it does not change its verdict.

### Dual-use and ethics

The attack scenarios describe harmful goals (phishing, credential theft, ransomware, surveillance) at the level of requests; they contain no working exploit code. Published transcripts are truncated to 300 characters per reply and hashed; full model outputs were kept private for hand-checking only. The canary tokens are random strings with no real secret behind them.

### Responsible disclosure

The `ConversationGuard` limitation concerns an open-source Lexsi kit; this write-up is our report of it to the Lexsi team. The model weaknesses found are in a public open-weights model and are of the kind already documented in the literature above.

## With more credits

### Planned approach

1. Grow the suite to about 50 scenarios per family, with turns written in Hindi and Hinglish as well as English, so per-family and per-language rates have usable intervals.
2. Hand-label a stratified sample of 200 judge verdicts and report agreement (Cohen's κ) for Granite Guardian and a second judge.
3. Extract a *context-aware* refusal direction from multi-turn conversations instead of single prompts, aimed at gradual and long attacks.
4. Replace steering with safety fine-tuning through SafeTune and AlignTune on refusals collected at the payload turn, and re-audit with the same harness.
5. Replicate on Tiny Aya Global and Aya Expanse 32B.

### Expected before / after

These are forecasts, not results. With a 250-attack suite, a test split would contain about 30–40 baseline breaks, enough to detect a 10-point drop in ASR. We expect safety fine-tuning to lower injection and spoof ASR the most (where the failure is following planted instructions), and a context-aware direction to help gradual attacks where the single-prompt direction did not.

## Artifacts

- **HF dataset collection:** [Multi-turn audit of Aya Expanse 8B: data](https://huggingface.co/sidhantmahi/collections)
  - dataset repo: [sidhantmahi/aya-expanse-8b-multiturn-audit](https://huggingface.co/datasets/sidhantmahi/aya-expanse-8b-multiturn-audit): scenario suite, redacted transcripts for every run, metric tables, run manifests, figures.
- **HF model collection:** [Multi-turn audit of Aya Expanse 8B: models](https://huggingface.co/sidhantmahi/collections)
  - refusal-direction repo: [sidhantmahi/aya-expanse-8b-refusal-direction](https://huggingface.co/sidhantmahi/aya-expanse-8b-refusal-direction) (direction file + settings), alongside the base model `CohereLabs/aya-expanse-8b`.
- **GitHub repo:** [SIDHANTMAHI/lexsi-multiturn-audit](https://github.com/SIDHANTMAHI/lexsi-multiturn-audit).
- Unredacted model outputs are intentionally not published (see Dual-use and ethics).

## License

Code and notebooks: MIT. Scenario suite, transcripts and metric tables: CC-BY-NC-4.0, matching the Aya Expanse 8B licence. See [LICENSE](LICENSE).

## Documentation and reproduction

### Environment

Google Colab with a T4 GPU (16 GB) for NB2 and NB3; CPU for NB1 and NB4. Python packages are installed by the first cell of each notebook: AuditKIT and SafeTune from GitHub, `transformers`, `bitsandbytes`, `accelerate`, `pandas`, `matplotlib`. Exact versions are recorded in `nb2_meta__v3-aya8b.json` and `manifest__v3-aya8b.json` in the dataset.

### Repo layout and run instructions

| File | What it does | Runtime |
|---|---|---|
| `NB1_guard_analysis.ipynb` | proofs and checks on SafeTune's `ConversationGuard`; guard on our suite | CPU, minutes |
| `NB2_audit.ipynb` | baseline multi-turn audit, turn of break, single vs multi-turn, hand-audit, gated re-scoring | T4, about 1 h |
| `NB3_repair.ipynb` | refusal direction, dev sweep, held-out test, random control, statistics | T4, about 3 h |
| `NB4_publish_to_hf.ipynb` | publishes the dataset, refusal direction and both collections | CPU |
| `scenarios.json` | the 66-scenario suite | |
| `pipeline.svg` | pipeline diagram | |

1. Add a Hugging Face token as the Colab secret `HF_TOKEN` and accept the Aya Expanse 8B licence on Hugging Face.
2. Put `scenarios.json` in `MyDrive/mta_v3/`; all notebooks read and write that folder.
3. Run NB1, NB2, NB3 in order. Runs are resumable: finished scenarios are reloaded from Drive. The suite hash `b6129e815a7a9a02` ties every result to this exact `scenarios.json`.

### Declaration of work done outside the hackathon

None. All code, scenarios and experiments were produced during the hackathon (Nov 2026).

## LLM usage statement

Claude (Anthropic) was used to draft and review most of the notebook code, to review results, and to draft this write-up. The team designed the study, wrote the attack scenarios, ran every experiment, hand-audited the judge verdicts, and checked the results.

## References

- Lexsi-Labs: [AuditKIT](https://github.com/Lexsi-Labs/AuditKIT), [SafeTune](https://github.com/Lexsi-Labs/SafeTune)
- Cohere Labs: [Aya Expanse 8B](https://huggingface.co/CohereLabs/aya-expanse-8b)
- Papers listed under Related work.
