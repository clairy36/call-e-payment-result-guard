# CALL-E Payment Result Guard

**Check the evidence behind payment-related call results before they become business records.**

When CALL-E returns a transcript and a structured conclusion, an application still needs to answer two questions: **Does the recipient's speech support the conclusion? If it does, which business state can that conclusion support?** This project provides a shared checker, thin adapters for kept and Recover, and offline tests and transcript replays around those questions.

Delivery version: **v0.2**, a bounded community reference implementation. Source changes are distributed as a [complete code patch](payment-result-guard.patch), alongside project documentation and validation evidence. Default demos require no API key and place no calls or payments.

## Why this exists

A completed call, populated fields, or high extraction confidence does not automatically justify a business-state update. Small differences in payment language can change the appropriate next step:

| Statement or extracted value | Business distinction | Relevant check |
| --- | --- | --- |
| Payment depends on approval | Conditional plan versus unconditional promise | Preserve the condition |
| Amount says 1250, but the recipient denies committing to a date | Structured conclusion conflicts with speech | Check support for the conclusion |
| Amount is `1k` | A numeric-cleaning shortcut can produce a misleading value | Reject unsupported amount notation |
| Recipient reports having paid | A statement is not independent settlement evidence | Keep external verification pending |
| Recipient requests a payment retry | Intent is not a successful charge | Keep the result advisory |

Local synthetic probes against kept showed `1k` being recorded as 100 minor units, and conditional amounts or contradictory quotes entering promise records. With the guard, those specific inputs instead produce an ambiguous-amount, conditional-plan, or contradictory-evidence result without constructing a Promise. See the [baseline](baseline-probes.json) and [after-change outputs](after-probes.json).

These observations establish behavior for particular local inputs. They do not establish incident frequency, real financial losses, or improved collection rates.

## Where it fits with CALL-E

CALL-E owns the SDKs, authentication, APIs, call execution, and provider controls. This contribution sits in the community application's result-consumption path: it checks the relationship between extracted fields and recipient speech, then reports the permitted scope of a business record.

```text
CALL-E result and transcript
          |
Host authentication, object binding, and policy checks
          |
Scenario adapter -> Shared evidence checker
          |
Claim status + Business status + Reasons + Applicable evidence references
          |
Host records a promise, saves advice, or routes the result for review
```

Identity and object binding remain host responsibilities. Scheduling remains host-owned. Actual payment and settlement require independent business systems and authorization. Every core output keeps `execution_authorized=false`.

## Reuse across two applications

| Application | Workflow | Integration point | Business boundary |
| --- | --- | --- | --- |
| **kept / Python** | Accounts-receivable payment promises | After existing binding and policy checks, before Promise construction | Supported statements may be recorded as promises; conditional, contradictory, or missing evidence requires review; no settlement claim |
| **Recover / TypeScript** | Customer intent after a subscription charge problem | After authentication and authoritative lookup, before advice persistence | A local Python process runs the same core; explicit retry intent supports advice, not payment execution |

The shared part is the evidence and state contract. Each host retains its data, ledger, and policies. Recover's existing advisory behavior and `recoveredCents=0` are preserved, not counted as new features.

For developers, this provides reproducible checks across two technology stacks. For operators, reason codes can help explain review cases. This delivery does not add an operations UI or claim a measured reduction in review time. See [PROJECT.md](PROJECT.md) for user needs, value, and tradeoffs.

## What is delivered

- **Shared checker:** evaluates business context, source scope, claims, and transcript turns; reports claim status, business status, and reasons separately.
- **Two thin adapters and minimal host integrations:** reuse kept's SDK simulation and ledger path, and Recover's actual route and Python process path.
- **42 synthetic fixtures:** 26 payment-promise cases and 16 recovery-intent cases covering supported expressions, conditions, contradictions, and error boundaries.
- **Three sanitized scripted-call replays:** preserve fictional utterances and anomalous amounts; distinguish provider completion, target acquisition, and core output.
- **Offline harness and complete patch:** support reproducible inspection without live calls or financial actions.

After applying the patch, the core is under `apps/python/payment-result-guard/`, with host changes under `apps/python/kept/` and `apps/typescript/recover/`. This delivery repository is not a full upstream checkout: apply the patch before running the source.

## Reading the validation results

| Layer | Evidence | Scope |
| --- | --- | --- |
| Core regression | [63 tests](core.log) | Bounded rules, CLI, and replay checks |
| kept local path | [195 tests](kept.log) | Existing SDK, simulated transport, capture, and temporary ledger |
| Recover local path | [49 tests](recover.log) | Actual POST route and Python process; provider and database substitutes |
| Total | **307 tests**; [machine-readable report](validation-report.json) | Not 307 real calls |
| Patch application | [Base application and tree comparison](patch-verification.json) | Complete patch against the specified upstream version |
| Transcript replay | [Three sanitized observations](scripted-replay.json) | Reuses existing inputs; does not call or rerun speech recognition |

The prior real AI-to-AI scripted-call round produced these coverage observations:

- **P01, explicit promise:** a complete promise utterance is still needed. The caller's example cannot count as recipient evidence.
- **P02, conditional promise:** the recipient said "If approved," but the intended 100.00 became 100100 in the transcript and extraction. Audio review is needed to locate the discrepancy.
- **P03, denial:** the full denial line was obtained and the provider extracted `refused`.

All three calls returned task-completion indicators, despite different sample coverage. Version v0.2 therefore reports regression reproducibility and target acquisition separately. The core routes these `unclear/refused` results to review before deeper evidence checks; these samples do not establish independent condition or amount detection by the core. Script reading is not a general natural-customer accuracy benchmark. See [ITERATIONS.md](ITERATIONS.md) for the sequence and remaining work.

## Getting started

Use **Python 3.11+**; Recover validation also needs **Node 22**. Download [payment-result-guard.patch](payment-result-guard.patch), then use a clean, normal clone of `awesome-phone-call-agents`:

```bash
git switch --detach 707122340774e3d63d5ce87c643695afc50d53cd
python3 scripts/check_branch_name.py --branch feat/payment-result-guard
git switch -c feat/payment-result-guard
git apply --check /absolute/path/payment-result-guard.patch
git apply /absolute/path/payment-result-guard.patch
```

Replace the example patch path with its actual location. Apply the patch once, without overwriting uncommitted work. From the patched upstream repository root:

```bash
python3.11 -m venv ../guard-venv
source ../guard-venv/bin/activate
python -m pip install -e apps/python/payment-result-guard
python -m payment_result_guard --demo
python apps/python/payment-result-guard/scripts/replay.py
```

The demo keeps a synthetic retry request advisory. Ordinary replay reports `regression_passed=true` when the recorded behavior is reproduced, while retaining `targets_complete=false` to identify outstanding sample work.

The optional `--require-targets` mode returns exit code 1 to flag the P01/P02 coverage follow-ups. This is separate from code-regression results. Installing dependencies may use the network; the default demo and replay need no live credentials. Full host setup and verification commands are in [RUN.md](RUN.md).

## Documentation and evidence

| File | Purpose |
| --- | --- |
| [PROJECT.md](PROJECT.md) | Problem evidence, users, value, implementation, and tradeoffs |
| [RUN.md](RUN.md) | Patch application, setup, demo, and full local validation |
| [ITERATIONS.md](ITERATIONS.md) | Development rounds, real scripted calls, and the v0.2 replay supplement |
| [CHECKLIST.md](CHECKLIST.md) | Delivery coverage and artifact mapping |
| [payment-result-guard.patch](payment-result-guard.patch) | Complete source diff for the upstream baseline |
| [validation-report.json](validation-report.json) | Component execution and synthetic fixture details |
| [SHA256SUMS](SHA256SUMS) | Checksums for published files |

## Boundaries and next steps

Automatic acceptance covers limited en-US expressions, numeric USD amounts, and absolute dates. Spoken numbers, other languages, relative dates, complex semantics, and denial classification remain limited. Recover requires a local Python interpreter; Edge/serverless deployment is unverified. Real host writes, payment execution, and production operations are outside the verified scope.

Default execution has no call or payment side effects. Public artifacts omit credentials, real phone numbers, and raw recordings. Prior live tests were explicitly authorized and run sequentially; stopping a query or closing a page does not cancel an already submitted call.

Next steps are to acquire P01's full sample, investigate P02's amount, and evaluate extensions with new regressions before separately validating Recover's live intent scenario. This is a personal delivery repository, with no upstream PR yet. The patch can be applied to a personal fork for subsequent community review.
