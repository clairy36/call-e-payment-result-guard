# Version history and validation process

Delivery version: v0.2, the replay and coverage supplement. The earlier local package is retained. This delivery version does not change the core API's `schema_version=0.1`.

## Problem addressed by this supplement

`completed` and `task_completed=true` do not establish target-utterance acquisition, evidence support, or permission to advance a business state. All three system-script calls completed, but only P03 returned the full target line. This supplement reports those layers separately and does not present conservative review routing as new semantic coverage.

## Development and observation rounds

| Round | Approach | Observation | Adjustment and evidence status |
| --- | --- | --- | --- |
| v0.1 local development | Shared core, kept/Recover adapters, minimal host paths | Historical 302 tests; 42 synthetic fixtures matched expectations | Original results remain local validation, without live-call evidence at that stage |
| T1 browser task | Prepare P01 in the chat interface | Preparation interrupted; no call ID or transcript obtained; zero backend calls was an assistant diagnostic | Move to the API; preparation is not a completed real call |
| T2 initial API | Fictional payment role-play with the default AI receptionist | Initial schema rejected with HTTP 400; existing kept schema then accepted; actual call lasted about 8.84 seconds and obtained only a greeting | Schema correction enabled submission; target sample remained outstanding |
| T3 default-receiver restart | P01 explicit promise and P02 conditional promise | About 67 and 46.4 seconds; recipient declined payment/approval role-play; P03 not placed | Owner requested consensual reading of fictional system-test lines |
| T4 scripted system test | Ask willingness, then supply a target line; one call per scenario | P01: 59 seconds, target missing. P02: 49.6 seconds, condition retained but amount became 100100. P03: 45.1 seconds, full denial | Actual scripted samples acquired; not natural-customer business outcomes |
| v0.2 local supplement | Sanitize T4 inputs and add independent replay/coverage checks | 307 tests; three observations reproduced; one exact target line acquired and two coverage follow-ups retained | Replay and coverage tooling verified; no additional call in this supplement |

A purchased inbound number was also prepared, but its assistant and number-page states differed and actual reception was not tested. The owner subsequently selected the original default test recipient. That preparation is not counted as a validated call scenario.

## Evidence for the three scenarios

**P01, explicit promise:** the caller spoke the example, but the recipient transcript was `The fictional invoice is test dash pizza. Interrupted.` The provider returned `unclear`; the core returned `unsupported / review_required`. The target is absent, and caller instructions cannot substitute for recipient evidence.

**P02, conditional promise:** the recipient transcript was `If approved, I will pay USD $100,100 on 10/10/2026.` The provider returned `unclear`, `promise_made=no`, and amount `100100`. The condition was retained, but the amount differed from the intended `100.00`. The core returned `unsupported / review_required`. Audio was not reviewed, so the discrepancy cannot yet be attributed to reading or transcription. The original value is retained.

**P03, denial:** the recipient said `I will not commit to paying this invoice.`, matching the target. The provider returned `refused` and `promise_made=no`; the core still returned `unsupported / review_required`. Target acquisition and provider refusal extraction were observed, but a core-supported denial classification is not implemented.

## What changed in v0.2

- Added three sanitized replay inputs preserving utterances and extracted amounts; replaced call IDs and omitted phone numbers, account data, recordings, credentials, and provider record locators.
- Added `scripts/replay.py`, reporting provider completion, exact recipient target matching, extracted-amount consistency, observed condition text, core claim status, and business status.
- Ordinary replay checks reproduction of observed behavior. `--require-targets` returns nonzero when any exact target line is absent. Exact matching is a fixed-script acquisition check, not a general semantic model.
- Added five tests, including caller-versus-recipient attribution, separation of provider completion from sample coverage, and the coverage-mode exit code.
- Integrated replay into the existing harness and updated documentation and the complete patch. Only the complete patch is published here to avoid mixing revisions.

The core amount rules, payment authority, and kept/Recover adapters are unchanged by this supplement. `unclear/refused` exits before deeper evidence checking, so these replays do not establish independent detection of conditions or amount anomalies. kept already rejects these outcomes; that existing protection is not a new benefit.

## Implementation and verification sequence

1. Inspected T4 API results, transcript turns, and core input/output. The anomalies already existed in the inputs and were not "fixed" by rewriting expectations.
2. Worked in an isolated checkout based on local implementation commit `5da61d3`, preserving uncommitted deletions in the original checkout.
3. Added tests first. Initial collection could not import the not-yet-created replay module; after implementation, all 63 core tests passed.
4. Ran the full harness: 63 core, 195 kept, and 49 Recover tests, totaling 307. The three sanitized replays reproduced their recorded outcomes.
5. Confirmed ordinary replay exit code 0 and coverage exit code 1, identifying P01/P02 follow-up work.
6. Ran repository validation and applied the full patch from the baseline in an isolated Git index. The resulting tree matched the expected tree; see [patch-verification.json](patch-verification.json).

The harness logs record this local run. Its original 42 fixtures are synthetic; the three additional inputs are sanitized replays from actual scripted AI-to-AI calls. Recover still uses provider/database substitutes, without real host business writes. The supplement did not rerun build or repository-wide lint; earlier records remain historical results.

## Proposed next call design, not yet live-validated

Keep invoice identifiers in metadata rather than speaking them with the target line. Obtain willingness to join a fictional audio test, read one short line at a time, and wait for the complete reply. If an amount differs, clarify at most once while retaining both the original and corrected turns; never silently replace it. Limit the call to 90 seconds, stop on refusal, and do not automatically redial.

P01 still needs a complete sample; P02 needs amount/audio review. Recover retry intent and actual host integration need separate verification. These script changes are proposed experiments, not measured improvements in call success, accuracy, or conversion.
