# Native search results — 2026-10-01

Results for `codex/claude-native-search` at `acc3cd86bb0ea1b6292b0a85fc316161ee24e5cd`, built with the default `native` profile (PGO) and the opt-in `native-exact` profile. The starting point was the handoff kit `3784f64` and source-review candidate `3f2f95d`.

Host: Apple M1 Max (MacBookPro18,2), macOS 15.8.1, CPython 3.11.15, Cython 3.2.4, Apple clang 17.0.0 with SDK 26.2 (`../cython/toolchain.json`). Five interleaved trials, 11,520 frozen windows, 518,400 candles. Totals sum the ETH/BTC/LP medians. Compare these numbers only with each other: other hosts, compilers and sessions differ.

## Five backends

| Backend | 1 worker | 4 workers |
|---|---:|---:|
| Curve Python `0bb370f` | 4.891 s | 1.344 s |
| Curve Python, basic Cython build | 1.834 s | 0.510 s |
| Fork Python (interpreted) | 2.556 s | 0.709 s |
| **Fork compiled, `native` profile** | **0.0510 s** | **0.0162 s** |
| Fork compiled, `native-exact` profile | 0.1121 s | 0.0333 s |
| Phil C++ `e9487b9`, older accounting | 0.0474 s | 0.0146 s |

The compiled fork is 96x faster than Curve Python with one worker and is 1.08x slower than Phil (1.11x with four workers). On the LP market it beats Phil: 0.0119 s against 0.0139 s with one worker. The interpreted fork takes 48% less time than Curve Python, because the replay now skips band searches that cannot trade.

## Correctness

- Fork Python matches untouched Curve Python bit for bit in all 11,520 windows.
- `native` stays within the unchanged `5e-14` tolerance (maximum `4.007905118896815e-14`, BTC). Its ordered outputs are bitwise identical to the handoff candidate's (`3f2f95d`).
- `native-exact` matches interpreted Python bit for bit (maximum error 0). It keeps every libm `pow()` call (`-fno-builtin-pow`), which costs about 2.2x.
- Repeats and one/four-worker outputs are bitwise equal within every backend. Phil differs in every window and remains a speed reference only.
- Two further clean builds produced identical generated C++ and extension binaries (`release-reproducibility.json`), including the PGO profile.

## Per commit

Compiled fork only, seven interleaved trials through `bench.py`'s own worker, one run per prepared commit. Each commit was built from a worktree at that commit so its declarations match its source.

| Commit | Change | 1 worker | 4 workers |
|---|---|---:|---:|
| `3f2f95d` | handoff candidate | 0.1836 s | 0.0517 s |
| `89a6352` | native oracle snapshot tuples | 0.1499 s | 0.0428 s |
| `a1e2e61` | raw oracle price as a double | 0.1436 s | 0.0411 s |
| `fb4d0bf` | dense band presence flags instead of a Python set | 0.1339 s | 0.0382 s |
| `284e7ad` | per-band geometry cache keys, no reset loop | 0.1312 s | 0.0377 s |
| `8384d0e` | `_power()` removed; `cpow` directive | 0.1309 s | 0.0376 s |
| `cb57bcc` | upstream oracle-limit body restored | 0.1299 s | 0.0373 s |
| `2784001` | synthetic PGO profile | 0.1165 s | 0.0338 s |
| `8a0ea8f` | timestamps stored as doubles | 0.0913 s | 0.0272 s |
| `0a46cd8` | geometry cache as fixed C arrays | 0.0881 s | 0.0262 s |
| `7f6157a` | opt-in exact profile (default build unchanged) | 0.0879 s | 0.0263 s |
| `6ffdafa` | skip band searches ruled out by the minimum fee (from wavey0x `d7703ab`) | 0.0660 s | 0.0198 s |
| `5803071` | drop the global spot-price cache (after wavey0x `d8162a1`) | 0.0663 s | 0.0201 s |
| `5cdaf72` | empty-band bounds skip no-op exchanges (from wavey0x `f0799eb`) | 0.0546 s | 0.0170 s |
| `c148f3b` | dimensionless band-factor cache (after wavey0x `c087342`) | 0.0523 s | 0.0162 s |
| `acc3cd8` | band balances in fixed C arrays, no Cython decorators | **0.0507 s** | **0.0162 s** |
| wavey0x `1505d10` | his updated candidate, for reference | 0.0988 s | 0.0288 s |

## Files

`summary.json`, `comparison.json`, `timings.json`, `receipt.json` and `release.json` cover the `native` run; `exact/` holds the same files for `native-exact`. `per-commit.json` holds the per-market medians behind the table above.
