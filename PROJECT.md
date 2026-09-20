# From call conclusions to business state: payment-result evidence checks

**This project adds an evidence check before a CALL-E result becomes a business record: first determine whether recipient speech supports the extracted claim, then determine how far that claim can advance the workflow.**

The delivery consists of a shared Python checker, thin kept and Recover adapters, and an offline validation harness. It targets result consumption in existing phone workflows, with no calls or payments in its default execution path. Version v0.2 is a bounded community reference implementation.

## 1. Why the workflow needs this step

CALL-E lets developers execute phone tasks and retrieve transcripts and structured results through SDKs and APIs. Business systems must then decide what a `completed` status, summary, or `promise_to_pay` field actually permits them to record.

A recipient who will pay only after approval has expressed a conditional plan. A recipient who reports having paid has made a statement that still needs settlement evidence. Permission to retry a charge does not establish recovered revenue.

If a consumer checks only field presence or model confidence, these different statements can become the same business state. Subsequent follow-up, promise tracking, and operator decisions depend on those records. The problem is therefore the evidence and authority behind a record, not just whether a conversation ended.

| Layer | Question | Treatment in this project |
| --- | --- | --- |
| Call execution | Did the call and conversation finish? | Consume CALL-E execution information; do not replace the provider |
| Evidence | What did the recipient say, and does it support the amount, date, and intent? | Apply bounded checks to host-bound transcripts, quotes, and fields |
| Business state | What kind of record can the supported claim justify? | Distinguish recordable promises, review, advice, and external verification |
| Financial result | Was money paid, settled, or recovered? | Require independent business systems and authorization |

Research used a repository snapshot from 2026-09-20. The community README and Roadmap place SDKs, authentication, APIs, call execution, and provider controls upstream with CALL-E. The community supplies Skills, Apps, Plugins, templates, and safety patterns, favoring small, reusable, offline-friendly contributions. This project works within that boundary by showing how an application can consume call results more carefully.

## 2. How the opportunity was identified

The investigation examined payment-related projects and reviewed patterns in Verity, Rebuttal, Kol, and Trunkline. Those projects offered useful approaches, but their domain objects and contracts differed. Their engines were not imported as a generic financial validator.

The concrete entry point came from synthetic probes against kept. With valid business binding and high confidence preserved, the probes changed the amount field or evidence quote and observed whether a promise was recorded.

| Probe | Original local behavior | Behavior with this contribution | Significance |
| --- | --- | --- | --- |
| Amount `1k` | Recorded 100 minor units, or USD 1.00 | No Promise; `AMBIGUOUS_AMOUNT` | Preserve uncertainty for unsupported notation instead of asserting the wrong amount |
| Amount `100 if approved`, with an approval condition in the quote | Recorded an ordinary promise for 10000 minor units | Review; `CONDITIONAL_STATEMENT` | Keep a conditional plan distinct from an unconditional promise |
| Amount 1250, with a quote denying commitment to a payment date | Recorded a promise for 125000 minor units | Review; `CONTRADICTORY_EVIDENCE` | Do not let extracted fields override conflicting recipient speech |

Contributing causes included permissive numeric cleaning and saving evidence text without independently checking its support for the promise. These are reproducible observations for specified local inputs and a specified version. They do not establish that CALL-E frequently generates such inputs online or that real losses occurred. See the [baseline output](baseline-probes.json) and [after-change output](after-probes.json).

Recover supplies a second perspective. It already kept financial effects advisory and maintained `recoveredCents=0`. This contribution preserves that boundary while checking whether recipient speech supports retry advice. Existing advisory and zero-recovery controls are not presented as new functionality.

## 3. Users and practical value

**The direct users are developers consuming CALL-E results.** Mapping a call into invoice, subscription, or follow-up records involves conditions, denials, corrections, missing evidence, and source ambiguity, not merely formatting fields. A shared contract lets two applications reuse evidence checks while retaining their own policies and state management.

**Payment-follow-up operators are downstream beneficiaries.** When a result cannot be recorded directly, machine-readable reasons and applicable evidence references can distinguish unsupported amounts, approval conditions, and conflicting speech. The current delivery provides interfaces and example integrations, not a new operations dashboard. It has not measured review-time savings.

**Community contributors can inspect and reproduce the change.** Default validation needs no real number or API key. A limited patch exposes both before/after behavior for specific inputs and the checks protecting the existing host paths.

| Value area | Current evidence | What it does not establish |
| --- | --- | --- |
| More reliable records for specific inputs | Three synthetic input classes no longer directly create unsupported or incorrect Promise records | Online error-rate reduction or avoided financial losses |
| More explainable handling | Separate claim status, business status, reason codes, and applicable quote references | Measured operator-efficiency improvement |
| Cross-application reuse | Python kept and TypeScript Recover call the same core | Effortless integration with every payment application |
| Easier reproduction | Offline demo, unified harness, fixed fixtures, and an applicable patch | Production deployment or operational readiness |
| Clearer validation interpretation | Call completion, sample acquisition, and regression results are reported separately | General accuracy on natural customer conversations |

## 4. Two applications, one shared problem

### kept: record a promise without claiming settlement

kept follows up on receivables and decides whether to create a Promise for later tracking.

The adapter runs **after existing binding and policy checks, before Promise construction**. It supplies the host's business object, amount context, call result, and transcript to the shared core. A statement meeting the bounded expression and evidence rules can continue through the existing record path. Conditional, contradictory, or incomplete evidence is routed for review. Confidence, date windows, amount policies, and ledger behavior remain host responsibilities.

For example, "I will pay USD 100.00 on October 10, 2026." can support a promise record when source, field, and context requirements also hold. A statement that payment has already occurred still cannot settle the invoice without independent evidence.

Local verification exercises the original SDK, simulated transport, capture function, and temporary ledger. It checks the recording entry point rather than only calling a standalone string checker.

### Recover: intent can support advice, not recovered revenue

Recover follows up after subscription charge problems. A recipient may request a retry, a card update, or a pause, each implying a different next step.

The adapter runs **after authentication and authoritative lookup, before advice persistence**. Node invokes a fixed Python module and submits the extracted decision together with recipient speech. An explicit retry request remains advisory. A short "Yes" is usable within the limited grammar only when the preceding question asks one explicit retry question. An unavailable or malformed checker result must not silently restore unguarded acceptance.

Local checks exercise the actual POST route and Python process, while substituting provider lookup and database boundaries. No real Stripe charge or recovery amount was verified.

The shared capability is the relationship between a conclusion, its supporting speech, and the next permitted record. The contribution does not unify the hosts' ledgers, payment logic, or operator workflows. That choice keeps the reusable component small.

## 5. Interface and deliverables

Input includes business context, host-supplied source binding and attempt scope, structured claims, and transcript turns with speaker and reference identifiers. A source label is not authentication: a CLI caller can forge it, so the checker cannot establish binding on behalf of the host.

Output separates `claim_status` from `business_effect_status` and includes `reason_codes`, applicable `evidence_refs`, and `next_step`. Every result keeps `execution_authorized=false`.

Within the limited grammar, a supported promise may be `recordable`; a retry request remains `advisory_only`; a report of payment remains `pending_external_verification`; uncertain results require review. These states guide the host's consumption of a result, not a new payment-execution service.

The delivery includes the shared core, both adapters, minimal host entry points, 42 synthetic fixtures, three sanitized scripted-call replays, test scripts, and a complete patch. Source is in [payment-result-guard.patch](payment-result-guard.patch); results are in [validation-report.json](validation-report.json) and the test logs. See [RUN.md](RUN.md) for setup.

## 6. Verification and the live-call learning loop

Current local validation covers **307 tests: 63 core, 195 kept, and 49 Recover**. The 42 baseline synthetic fixtures comprise 26 promise cases and 16 recovery-intent cases. Additional tests cover CLI behavior, adapters, host paths, and errors; this count is not a count of real calls.

Actual AI-to-AI calls then exposed a separate testing issue: task completion does not establish that the intended sample was obtained.

| Scripted scenario | Recipient behavior | Current interpretation |
| --- | --- | --- |
| P01, explicit promise | Agreed to participate but returned an incomplete invoice phrase | Full promise sample still needed; the caller's example is not recipient evidence |
| P02, conditional promise | Said "If approved," with 100100 in transcript and extraction rather than the intended 100.00 | Conditional language observed; amount discrepancy needs audio investigation |
| P03, denial | Returned "I will not commit to paying this invoice." | Complete denial obtained; provider extracted `refused` |

Version v0.2 adds replay and coverage checks to report provider completion, recipient target acquisition, extracted-amount consistency, and core output separately. Ordinary replay exits 0 when the three observed outcomes are reproduced. Coverage mode exits 1 to identify outstanding target samples. Anomalous amounts are preserved instead of replaced with intended values.

All three core outputs are `unsupported / review_required`. The core exits early for `unclear/refused`, before deeper evidence checks. Consequently, this round does not prove independent condition or amount detection by the core, and P03 does not establish a supported-denial classification. The [iteration record](ITERATIONS.md) preserves these limits and the preceding rounds.

## 7. Tradeoffs and responsibility

The checker covers only bounded en-US expressions, numeric USD amounts, and absolute dates. Spoken numbers, other languages, relative dates, and complex semantics require review. Deterministic rules support reproduction but have limited coverage; this is not a general language-understanding system.

Recover reuses Python through a local subprocess, adding an interpreter dependency. The local Node path is verified; Edge/serverless deployment is not. The contribution does not build a financial platform, operations UI, scheduler, or payment executor, or complete every host recovery and concurrency mechanism.

Default checks need no credentials and make no calls or payments. Previous real calls were explicitly authorized, restricted to test lines, run sequentially, and not automatically redialed. After a query interruption, only the existing call ID was queried. Stopping a query or closing a page does not cancel a submitted call. Public artifacts omit API keys, real phone numbers, raw recordings, and provider record identifiers that locate the account's calls.

The host owns identity, recipient authorization, object binding, and state policy; the host scheduler owns recurrence; CALL-E owns execution; independent systems and authorized people own settlement and actual financial actions.

## 8. AI assistance and human judgment

AI assisted repository research, code reading, comparison of existing tools, implementation, tests, execution, transcript inspection, and documentation. The work followed a traceable sequence: code finding, synthetic reproduction, integration, regression checks, actual scripted calls, and sanitized replay/documentation updates.

The project owner retained the choice of problem, payment-domain scope, cross-application direction, real-call authorization, recipient routing, and script changes. Generated conclusions were checked against source, inputs and outputs, logs, and patches rather than assistant summaries. Work spanned multiple rounds; the full delivery is not described as having been completed within three hours.

## 9. How to decide whether to invest further

First acquire P01's full sample and inspect the audio behind P02's amount discrepancy to distinguish script reading, transcription, extraction, and local consumption. Then use new regression cases to evaluate denial classification and spoken-number support. Expanded coverage should examine both normal acceptance and erroneous acceptance rather than merely making fixed fixtures look ideal.

Separately validate Recover's live retry-intent scenario and controlled host integration. Future measures could include detection of inconsistent records, the proportion of valid results routed to review, operator review time, and integration effort. These are proposed measures, not achieved business outcomes.

Research sources were the fixed-snapshot READMEs, Roadmap, AGENTS.md, community review policy, and kept/Recover source. The upstream projects are CALLE-AI/call-e-integrations and CALLE-AI/awesome-phone-call-agents. The exact patch baseline and reproduction steps are in the run guide.
