# ms-swift: unsafe YAML load in the test runner (CVE-2025-50460)

Case study of an unsafe-deserialization advisory (CWE-502) in
[modelscope/ms-swift](https://github.com/modelscope/ms-swift): the test-runner script parses a user-supplied YAML
config with `yaml.load(..., Loader=yaml.FullLoader)`; the upstream fix replaces it with `yaml.safe_load`.

> **Demo only, not for use.** This repository is a case study built from unmodified snapshots of
> [modelscope/ms-swift](https://github.com/modelscope/ms-swift) at two historical commits. It is not a fork and is not maintained. Do not install, deploy, or
> depend on it: the `baseline` branch contains code affected by the advisories below.

**This branch: `baseline`** (tag `poe`): snapshot of `modelscope/ms-swift` @ [`c982313ab0`](https://github.com/modelscope/ms-swift/commit/c982313ab0be8aefd882a7c2e3cb90e423dda7c5) (2025-07-31), the parent of the upstream fix commit, i.e. the last vulnerable state. The patched snapshot is on [`fix`](https://github.com/ferralon-demo/ms-swift-yaml-load-rce/tree/fix).

## Advisory

| | |
|---|---|
| **CVE** | [CVE-2025-50460](https://nvd.nist.gov/vuln/detail/CVE-2025-50460) |
| **GHSA** | [GHSA-fm6c-f59h-7mmg](https://github.com/advisories/GHSA-fm6c-f59h-7mmg), "MS SWIFT Remote Code Execution via unsafe PyYAML deserialization", CWE-502 |
| **Affected** | `ms-swift` <= 3.6.3, per the GHSA record (which lists no patched version) |
| **Upstream fix** | [modelscope/ms-swift#5174](https://github.com/modelscope/ms-swift/pull/5174), commit [`b3418ed9b0`](https://github.com/modelscope/ms-swift/commit/b3418ed9b050dc079553c275c5ed14cfb2b66cf7), merged 2025-07-31 |

## Ingress and sink at `baseline`

All line numbers are at upstream commit `c982313ab0`.

| | Location |
|---|---|
| **Ingress** | Command line of the test runner: `--run_config <path>` is declared at [`tests/run.py:591-594`](https://github.com/modelscope/ms-swift/blob/c982313ab0be8aefd882a7c2e3cb90e423dda7c5/tests/run.py#L591-L594); the `if __name__ == '__main__':` block ([line 579](https://github.com/modelscope/ms-swift/blob/c982313ab0be8aefd882a7c2e3cb90e423dda7c5/tests/run.py#L579)) calls `run_in_subprocess(args)` at [line 622](https://github.com/modelscope/ms-swift/blob/c982313ab0be8aefd882a7c2e3cb90e423dda7c5/tests/run.py#L622) when `--run_config` is given |
| **Sink** | `yaml.load(f, Loader=yaml.FullLoader)` at [`tests/run.py:422`](https://github.com/modelscope/ms-swift/blob/c982313ab0be8aefd882a7c2e3cb90e423dda7c5/tests/run.py#L422), inside `run_in_subprocess` (defined at [line 387](https://github.com/modelscope/ms-swift/blob/c982313ab0be8aefd882a7c2e3cb90e423dda7c5/tests/run.py#L387)), parsing the file named by `--run_config` |

The sink is a call into a dependency (PyYAML's `yaml.load`), not ms-swift's own code. The vulnerable code is in the
repository's test-runner script, `tests/run.py`, not in the `swift` Python package.

## What the fix changes

At `b3418ed9b0`, line 422 reads `run_config = yaml.safe_load(f)`: the call to `yaml.load` is gone and the ingress
is unchanged. The same commit adds `PyYAML>=5.4` to `requirements/framework.txt`.

## Provenance

This is not a fork: a fresh repository whose code branches hold upstream source snapshots, with no upstream
history and no upstream remote.

| Branch | Tag | Upstream commit | Date | Snapshot commit | Snapshot tree |
|---|---|---|---|---|---|
| `baseline` | `poe` | [`c982313ab0`](https://github.com/modelscope/ms-swift/commit/c982313ab0be8aefd882a7c2e3cb90e423dda7c5) | 2025-07-31 | `ce460f3726024fff10bc8f690256e95203e84833` | `e05badb9bcb3c5e497996b4b41e58673707bc350` |
| `fix` | `pone` | [`b3418ed9b0`](https://github.com/modelscope/ms-swift/commit/b3418ed9b050dc079553c275c5ed14cfb2b66cf7) | 2025-07-31 | `d1cf165b8deaf30f1465c0ba20c3d974be62924c` | `c7d1f77bbf05e0170306e57010b21dbfb3072697` |

Each snapshot commit is the upstream tree at that commit with one change: `.github/workflows/` (4 files) is
removed so that upstream CI does not run here. No other file is added, removed, or edited. The `fix` snapshot's
parent is the `baseline` snapshot, so the diff between the two snapshot commits
([compare](https://github.com/ferralon-demo/ms-swift-yaml-load-rce/compare/ce460f3726024fff10bc8f690256e95203e84833...d1cf165b8deaf30f1465c0ba20c3d974be62924c)) is exactly the upstream fix commit's diff. On each code branch, a second
commit replaces upstream's `README.md` with a branch-specific version of this file; upstream's README is in the snapshot commit.

To check a snapshot against upstream from a clone of this repository:

```sh
git fetch https://github.com/modelscope/ms-swift c982313ab0be8aefd882a7c2e3cb90e423dda7c5
git diff --stat FETCH_HEAD ce460f3726024fff10bc8f690256e95203e84833   # lists only .github/workflows/*
```

## Licence

ms-swift is licensed under the Apache License 2.0. The upstream `LICENSE` is kept verbatim at the root of the
`baseline` and `fix` branches; upstream ships no `NOTICE` file. Two upstream files (`swift/tuners/lora.py`,
`swift/tuners/lora_layers.py`) carry their own MIT header, also unchanged. Changes made for this case study, on
2026-10-07: `.github/workflows/` removed, and this `README.md` replaces upstream's.
