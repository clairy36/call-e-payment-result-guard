# Running and reviewing the contribution

This repository distributes a complete source patch. Apply it to the specified upstream baseline, then install and run the resulting source.

## Apply the patch

Download `payment-result-guard.patch`. In a clean, normal clone of `awesome-phone-call-agents`:

```bash
git switch --detach 707122340774e3d63d5ce87c643695afc50d53cd
python3 scripts/check_branch_name.py --branch feat/payment-result-guard
git switch -c feat/payment-result-guard
git apply --check /absolute/path/payment-result-guard.patch
git apply /absolute/path/payment-result-guard.patch
```

Replace the example path with the downloaded file's location. Do not run these steps over uncommitted work; apply the complete patch only once. The resulting source includes the core, tests, adapters, and documentation. Application to an isolated Git index and comparison of the full resulting tree are recorded in [patch-verification.json](patch-verification.json).

## Quick offline demo

Use Python 3.11+. From the patched upstream repository root:

```bash
python3.11 -m venv ../guard-venv
source ../guard-venv/bin/activate
python -m pip install -e apps/python/payment-result-guard
python -m payment_result_guard --demo
python apps/python/payment-result-guard/scripts/replay.py
python apps/python/payment-result-guard/scripts/replay.py --require-targets
```

Dependency installation may use the network. The default demo and replay need no API key and make no calls or payments. Ordinary replay exits 0 when the observed results are reproduced. Coverage mode exits 1 with `targets_complete=false`, identifying the outstanding P01 promise utterance and P02 amount follow-up. Regression, sample coverage, and business state are recorded separately.

## Both hosts and full local validation

Install Node 22. In the same patched repository and Python environment:

```bash
python -m pip install -e 'apps/python/kept[dev]'
npm --prefix apps/typescript/recover ci
python apps/python/payment-result-guard/scripts/verify.py --output ../guard-report
python scripts/validate_repository.py
```

Current local evidence covers 307 tests: [core](core.log), [kept](kept.log), [Recover](recover.log), and the [full report](validation-report.json). The three sanitized call replays are in [scripted-replay.json](scripted-replay.json).

Host checks use local substitutes at provider or database boundaries. They do not connect to real payment or database services. The latest supplement did not rerun the full build or repository-wide lint; earlier validation recorded pre-existing Recover lint issues. These results are not a claim that every check across the upstream repository is clean.

## Community contribution path

This is a personal delivery repository, with no upstream PR yet. Apply the patch to a feature branch in a personal fork before creating a Draft PR. Do not push the local snapshot-import history. The project explanation and iteration record are directly readable here, without an archive.
