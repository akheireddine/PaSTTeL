# PaSTTeL

**PaSTTeL** (Parallel analysiS framework for Termination and non-Termination of Lasso programs) is a C++17 tool that analyses *lasso programs* — a finite stem followed by an infinite loop — to prove **termination** or **non-termination**. It uses SMT solvers (Z3, CVC5) to synthesise ranking functions and non-termination witnesses, and runs a portfolio of analysis techniques in parallel over multiple CPU cores.

## Repository structure

```
PaSTTeL/
├── Dockerfile               — Docker image definition (Ubuntu 24.04)
├── docker-compose.yml       — Docker Compose configuration
├── Makefile                 — Native build (binary, static library, unit tests)
├── LICENSE                  — AGPL-3.0 license
├── include/                 — Public headers (pasttel.h is the library entry point)
├── src/                     — C++ source code
├── examples/                — JSON lasso trace examples for testing
├── scripts/
│   ├── test_non_regression.py         — Non-regression test suite (JUnit XML output)
│   ├── benchmark_ultimate_vs_pasttel.py — Compare PaSTTeL against Ultimate LassoRanker
│   └── comparison.sh                  — Batch comparison driver
├── benchmarks/
│   ├── C/                   — C benchmark programs (SV-COMP and competition benchmarks)
│   ├── BPL/                 — Boogie benchmark programs
│   └── get_lasso_benchmark.sh — Extract lasso traces with Ultimate Buchi Automizer
└── tools/
    ├── UAutomizer-linux/    — Ultimate Buchi Automizer pre-compiled binary
    └── solvers/             — Pre-downloaded solver archives (Z3, CVC5)
```

## Build

### Option 1 — Docker (recommended)

```bash
docker build -t pasttel .

# Interactive shell:
docker run -it --rm pasttel bash

# Or with Docker Compose:
docker compose up -d
docker compose exec pasttel bash
```

The image installs Z3 and CVC5, builds PaSTTeL, and provides the binary at `./bin/pasttel`.

### Option 2 — Native build

Requirements:

- `g++` with C++17 support (GCC ≥ 9)
- Z3 development headers and shared library (`libz3.so`) — tested with 4.15/4.16
- CVC5 1.3.3 shared build (`libcvc5.so`, `libcvc5parser.so`, and dependencies)
- Boost headers (`libboost-dev`)
- Python 3 (for the test and benchmark scripts)

The solver archives bundled in `tools/solvers/` can be used directly:

```bash
unzip tools/solvers/z3-4.15.4-x64-glibc-2.39.zip -d /path/to/solvers
cd /path/to/solvers/z3-4.15.4-x64-glibc-2.39 && mkdir -p lib && cp bin/lib* lib/ && cd -

unzip tools/solvers/cvc5-Linux-x86_64-shared.zip -d /path/to/solvers
```

Then point the Makefile to the solver install prefixes and build:

```bash
export PASTTEL=/path/to/solvers/z3-4.15.4-x64-glibc-2.39
export CVC5_DIR=/path/to/solvers/cvc5-Linux-x86_64-shared
export LD_LIBRARY_PATH=${PASTTEL}/lib:${CVC5_DIR}/lib:${LD_LIBRARY_PATH}

make -j$(nproc)
# Binary produced at: ./bin/pasttel
```

## Usage

PaSTTeL takes a single JSON lasso trace as input:

```
Usage: ./bin/pasttel [options] <filename>

Options:
  -a <terminate|nonterminate|both>   Set analysis mode (default: both)
  -t <int>                           Time limit in seconds (default: 6000)
  -s <z3|cvc5>                       Set SMT solver (default: z3)
  -q                                 Quiet mode (silence output)
  -v                                 Verbose mode (more output)
  -c <int>                           Number of CPUs (default: 1)
  -nla <overapproximate|underapproximate|none>
                                     Non-linear arithmetic handling (default: overapproximate)
  -mode <linear|nonlinear>           Set analysis mode (default: linear)
  -h, --help                         Show this help message

Examples:
  ./bin/pasttel -a terminate -s z3 -c 4 -t 300 input.json
  ./bin/pasttel -a both -s cvc5 -v input.json
```

The verdict is one of `TERMINATING`, `NON-TERMINATING`, `TIMEOUT`, or `UNKNOWN`. Example run on a bundled trace:

```bash
./bin/pasttel -a both -s z3 -c 1 examples/test_simple_counter.json
```

```
============================================================
                    ANALYSIS REPORT                         
============================================================

--- TERMINATION TECHNIQUES ---
Technique                          Result         Time (s)    Proof
--------------------------------------------------------------------------------
RankingBased(AffineTemplate)       TERMINATING    0.007       2·n + 7

============================================================
OVERALL RESULT: TERMINATING
TOTAL TIME: 0.074 s
TERMINATING TIME: 0.007 s
============================================================

```

## Test suite

```bash
# Unit tests (built alongside the main binary):
make test

# Non-regression suite on the JSON examples (JUnit XML output, used by CI):
python3 scripts/test_non_regression.py
```

## Benchmarks

The `benchmarks/C/` and `benchmarks/BPL/` directories contain termination benchmarks from SV-COMP and other competition suites, organised by category.

Lasso traces are extracted from these programs with **Ultimate Buchi Automizer** (pre-compiled binary in `tools/UAutomizer-linux/`, Java 21 JRE required):

```bash
# Extract lasso traces from the C or BPL benchmarks:
cd benchmarks/
./get_lasso_benchmark.sh C     # or BPL
```

Only lasso programs manipulating integers, booleans and reals are supported by PaSTTeL. The comparison against Ultimate LassoRanker is driven by `scripts/benchmark_ultimate_vs_pasttel.py` (requires Python 3 with `plotly`, `pandas` and `matplotlib` for the plots), which also converts Ultimate `.txt` traces into the JSON format expected by PaSTTeL.

## Using PaSTTeL as a library

PaSTTeL can be built as a static library (`bin/libpasttel.a`) for use in other C++17 projects:

```bash
make lib
```

The entry point is `include/pasttel.h` (`runAnalysis()`). Lasso programs can be loaded from a JSON trace file via `JsonTraceParser::parseToLasso()` (see `include/parser/json_trace_parser.h`), or constructed directly as a `LassoProgram` instance (see `include/lasso_program.h`) to integrate PaSTTeL into a larger verification pipeline without the JSON format.

## License

PaSTTeL is released under the [AGPL-3.0](LICENSE) license.

## Authors

- Anissa Kheireddine — <Anissa.Kheireddine@dowsers.finance>, <Anissa.Kheireddine@lip6.fr> (DOWSERS, LIP6)
- Souheib Baarir — <Souheib.Baarir@lip6.fr> (LIP6)
- Hugo — <Hugo@dowsers.finance> (DOWSERS)
