# Native replay: readability review

This branch keeps Python as the only implementation of the model and compiles the same files with Cython. It is based on Curve master `0bb370f02970c2c1056da8a4faffd0fa9be35575`. Type declarations live in `declarations/*.pxd`; the `.py` files stay ordinary Python that runs unchanged without a compiler. It is not a proposal to merge every change as one upstream PR.

## Assessment

The remaining Python changes fall into three groups:
- Replay shortcuts that also make interpreted Python about twice as fast.
- Small caches with explicit keys.
- One custom band container.

The only Cython-specific code left in the `.py` files is an import block that picks the C `isfinite`/`sqrt` in native builds. All other typing, array layout and power semantics live in the declarations and the build profile.

| Change | Readability and compatibility cost | Assessment |
|---|---|---|
| Numeric replay and oracle helpers | `_calculate_loss`, `replay_batch`, `_observe` and a module-level `find_target_price` move existing calculations behind a typed boundary; indexed float64 inputs are less general than arbitrary sequences | Reasonable boundary for native compilation; the main replay is still easy to follow |
| Replay shortcuts | The replay checks the minimum applicable fee and an empty band's raw bounds before searching bands; `_trade_prices` returns the quote with those bounds. About ten extra lines in the replay loop, covered by targeted tests | Exact: a distance fee can only raise the fee, and the bounds are the ones `trade_to_price` already tests. Largest single gain, compiled and interpreted |
| Powers | Plain `**` as upstream. The build profile sets Cython's `cpow` directive so `double ** double` stays a libm `pow()` call rather than complex arithmetic; the earlier `_power()` wrapper is gone | No visible cost. A negative base with a fractional exponent would give NaN natively instead of a complex number; no reachable state produces one |
| Caches | A dimensionless band factor `((A - 1) / A) ** n` cached per slot with its `A`, plus last-value caches for `p_oracle ** 3` and `(A / (A - 1)) ** 2`. The global spot-price cache is removed | Small, keyed by every input, no reset obligations |
| Band storage | `BandBalances` with `.read()` / `.write()` replaces `defaultdict(float)`. Represented bands use presence flags and the lowest/highest band instead of a Python set; the dense buffers are C arrays only in the declarations | Still the largest aesthetic cost; behaves like a dict for the existing API and tests |
| Timestamps | Internally a float, with `NO_TIMESTAMP` (`-inf`) for "no clock yet". `current_timestamp` and `prev_p_oracle_time` are properties that still return `None`; `set_p_oracle` and `dynamic_fee` still accept `None`. `calculate_loss` rejects non-finite candle timestamps up front | Two small properties. An explicit `-inf` timestamp now means "missing" rather than raising; private `_price_oracle_view` takes `NO_TIMESTAMP` instead of `None` |
| Per-instance oracle settings | Configuration defaults move from class attributes to instance fields | Changes class-wide override behavior; unchanged from the earlier review |
| Reset and numeric batch replay | State-reset obligations and an eight-column resolved-task interface | Useful for throughput; the batch contract is still benchmark-specific |

Financial formulas, operation order, fee charging, oracle write timing and recovery accounting are unchanged. `_limit_price_oracle`, `_memory_dt` and `get_p` read as upstream again.

## Measured result

Apple M1 Max, Apple clang 17, five interleaved trials over the 11,520 frozen windows. Totals sum the three per-market medians; see [`native-benchmark/search-results`](../../native-benchmark/search-results/README.md).

| Implementation | One worker | Four workers |
|---|---:|---:|
| Upstream Curve Python | 4.891 s | 1.344 s |
| Existing upstream Cython build | 1.834 s | 0.510 s |
| Fork Python | 2.556 s | 0.709 s |
| Fork native, default profile | 0.0510 s | 0.0162 s |
| Fork native, exact profile | 0.1121 s | 0.0333 s |
| Phil C++ with older accounting | 0.0474 s | 0.0146 s |

The default native build is 36x faster than the existing Cython build and 1.08x slower than Phil with one worker. Interpreted fork Python now takes 48% less time than upstream instead of 10% more. Fork Python matches all 11,520 upstream losses bit for bit. The default native build stays within `5e-14` (maximum `4.01e-14`); the exact profile matches Python bit for bit. Clean builds are byte-identical.

## Build declarations

`declarations/lending_amm.pxd` and `declarations/simulator.pxd` mirror the native declarations used by the build kit. An external build package stages them beside the matching `.py` files, then generates and compiles C++. They are review inputs here, not an automatically enabled build or a second accounting implementation.

The build kit also trains a profile-guided optimization pass on seeded synthetic windows (never the benchmark inputs). Compiler flags stay `-std=c++17 -O3 -g0 -ffp-contract=off -fno-fast-math`. Without `-fno-builtin-pow`, clang may compute a constant square as `x * x`, which differs from Apple's libm `pow(x, 2)` in about 0.13% of inputs. That is the source of the default build's small differences, and the exact profile removes it.
