<div align="center">

<img src="docs/assets/cinder-mark.svg" alt="cindervm" width="132" height="132">

# cindervm

**A deterministic bytecode VM for agent execution.**
Verified bytecode, serializable continuations, exact replay.

[![CI](https://img.shields.io/github/actions/workflow/status/ashish/cindervm/ci.yml?branch=main&label=ci&logo=githubactions&logoColor=white&style=flat-square)](https://github.com/ashish/cindervm/actions/workflows/ci.yml)
[![crates.io](https://img.shields.io/crates/v/cindervm?logo=rust&logoColor=white&style=flat-square)](https://crates.io/crates/cindervm)
[![docs.rs](https://img.shields.io/docsrs/cindervm?logo=docsdotrs&logoColor=white&style=flat-square)](https://docs.rs/cindervm)
[![MSRV](https://img.shields.io/badge/msrv-1.78-blue?logo=rust&logoColor=white&style=flat-square)](rust-toolchain.toml)
[![ISA](https://img.shields.io/badge/ISA-cdx%2F4-6f42c1?style=flat-square)](docs/isa.md)
[![deps](https://img.shields.io/badge/dependencies-0-success?style=flat-square)](Cargo.toml)
[![unsafe](https://img.shields.io/badge/unsafe-forbidden-success?logo=rust&logoColor=white&style=flat-square)](src/lib.rs)
[![License](https://img.shields.io/badge/license-Apache--2.0%20OR%20MIT-lightgrey?style=flat-square)](#license)

<img src="docs/assets/demo.svg" alt="cinder run --replay walkthrough" width="860">

<sub>`cinderc` → `verify` → `run` → `replay --seek`. Full transcript in [docs/walkthrough.md](docs/walkthrough.md).</sub>

</div>

---

## Contents

- [What this is](#what-this-is)
- [Why a VM](#why-a-vm)
- [Mental model](#mental-model)
- [Quickstart](#quickstart)
- [The instruction set](#the-instruction-set)
- [The verifier](#the-verifier)
- [Continuations](#continuations)
- [Determinism and replay](#determinism-and-replay)
- [The supervisor](#the-supervisor)
- [Performance](#performance)
- [Building from source](#building-from-source)
- [Repository layout](#repository-layout)
- [Stability](#stability)
- [FAQ](#faq)
- [License](#license)

---

## What this is

An agent run is a long-lived, effectful, resumable computation. Most frameworks
model it as a Python `while` loop with a list of dicts, which means the run's
state lives in interpreter frames you cannot serialize, on a machine you cannot
lose, in an order you cannot reproduce.

`cindervm` models it as **bytecode on a machine designed for it**. Agent
control flow — issuing a tool call, awaiting it, forking speculative branches,
checkpointing, yielding to a scheduler — is expressed in the instruction set
rather than in host-language coroutines. Consequences follow from that one
decision:

| Property | Mechanism |
|---|---|
| A run can be moved between hosts mid-flight | The entire machine state is a value ([`cont.rs`](src/cont.rs)) |
| A crashed run resumes without re-executing effects | Append-only, hash-chained journal ([`journal.rs`](src/journal.rs)) |
| A bug is reproducible from the log alone | The interpreter is a pure function of `(image, journal)` |
| Malformed bytecode cannot corrupt the interpreter | Bytecode is verified before it is admitted ([`verify.rs`](src/verify.rs)) |
| Cost is bounded before a token is spent | Budget reservations are instructions, not middleware |

## Why a VM

The alternative designs and why they were rejected:

**Host coroutines** (`async fn`, Python `async def`). Suspension points are
implicit and the suspended state is a compiler-generated struct with no stable
representation. You cannot write it to disk, you cannot inspect it, and you
cannot resume it in a different process. Fine for I/O concurrency; wrong for
durable execution.

**Durable-execution engines with replay-based recovery.** Recovery works by
re-running the whole program and short-circuiting completed calls from a log.
This makes non-determinism catastrophic: one unlogged `now()` and the replay
diverges from the original. `cindervm` restores state directly instead of
re-deriving it, so divergence is impossible by construction — and every source
of non-determinism is an instruction that reads from the journal.

**Interpreting a graph / AST.** Workable, but the state is a tree of live
objects with sharing and cycles, so serialization becomes a graph-walk with
identity preservation. A flat operand stack over a flat heap of tagged values
serializes as a memcpy of two arrays plus a relocation pass.

The VM approach costs an assembler and a verifier. It buys a state
representation that is *already* a byte string.

---

## Mental model

```
                        ┌──────────────────────────────────────────┐
   .cdx source          │  cinderc                                 │
  ───────────────────►  │   lex ─► parse ─► resolve ─► encode      │
                        │                       │                  │
                        └───────────────────────┼──────────────────┘
                                                ▼
                                        image.cdxb  (verified once,
                                                     hash-sealed)
                                                │
        ┌───────────────────────────────────────┴───────────────────┐
        ▼                                                           ▼
┌───────────────────┐                                    ┌────────────────────┐
│  interp.rs        │   trap(CALL_TOOL, args)            │  supervisor (Go)   │
│                   │ ─────────────────────────────────► │                    │
│  pc, stack, heap  │                                    │  scheduler         │
│  frames, budget   │ ◄───────────────────────────────── │  syscall broker    │
│                   │   journal record #n (sealed)       │  quota ledger      │
└─────────┬─────────┘                                    └────────────────────┘
          │
          │  YIELD_CTX / CHECKPOINT
          ▼
┌───────────────────┐
│  cont.rs          │   snapshot ──► blob ──► object store
│  frozen machine   │   blob ──────► restore  (any host, any time)
└───────────────────┘
```

Three invariants hold everywhere in the codebase, and most of the design falls
out of them:

1. **The interpreter performs no I/O.** It returns a `Trap` describing what it
   needs. The host answers. This is why replay is exact — replacing the host
   with a journal reader is a no-op from the interpreter's perspective.
2. **Every value is copyable and tagged.** No host pointers in VM state, no
   `Rc`, no interior mutability. `Value` is `Copy` and 16 bytes; anything larger
   lives in the heap arena behind a `Handle`.
3. **Verification is a precondition of execution, not a mode.** `Image` cannot
   be constructed except through `verify::admit`. The interpreter therefore
   contains no bounds checks on `pc`, no stack-depth checks, and no operand type
   dispatch failures — the verifier already proved they cannot happen.

---

## Quickstart

```sh
cargo install cindervm            # cinderc, cinder
go install ./cmd/cinderd          # supervisor (optional)
```

A minimal agent. `.cdx` is the textual form of the bytecode — a macro assembler,
not a language.

```asm
        .isa      cdx/4
        .image    "triage"
        .budget   tokens=8000 wall=45s tools=6

        .const    $sys   "You triage bug reports. Reply with severity only."
        .tool     %rank  "llm.complete"   -> str
        .tool     %file  "github.issue.label"

        .fn       main() -> i32
        .maxstack 6
main:
        ldc       $sys
        argv      0                       ; the issue body, from the host
        pack      2
        calltool  %rank                   ; traps; supervisor answers
        await                             ; blocks this VM, not the host thread
        dup
        ldc       "critical"
        eq
        brz       .done                   ; not critical → nothing to do
        checkpoint "pre-label"            ; durable point before a side effect
        ldc       "P0"
        calltool  %file
        await
        drop
.done:
        drop
        ldi       0
        ret
```

```sh
$ cinderc triage.cdx -o triage.cdxb
   compiled  triage.cdxb   264 B   1 fn   14 insns   maxstack 6
   verified  frames=14/14  types=ok  stack=ok  effects=ok    1.9 ms

$ cinder run triage.cdxb --arg "segfault on startup, every launch" --journal run.jl
   [0000]  calltool  llm.complete           ↑ 41 tok
   [0001]  await                            ↓ 3 tok   612 ms   "critical"
   [0002]  checkpoint pre-label             ● 1.4 KB
   [0003]  calltool  github.issue.label     ↑ —
   [0004]  await                            ↓ —      208 ms   ok
   halt 0      wall 0.83 s   tokens 44/8000   tools 2/6
```

Then debug it without touching the network:

```sh
$ cinder replay run.jl --seek 0002 --inspect stack
   stack  [0] str  "critical"    (heap #3, 8 B)
   frame  main  pc=0x1a  depth=1/6
   budget tokens 44  wall 0.61s  tools 1
```

`replay` is not a simulation. It is the same interpreter with the syscall broker
swapped for a journal cursor; a replay that diverges from its journal is a bug
and aborts with `E_DIVERGE` naming the instruction.

<img src="docs/assets/replay.svg" alt="time-travel replay: stepping backward through a journal" width="860">

---

## The instruction set

`cdx/4`. Fixed 4-byte encoding: `[opcode:8][a:8][b:16]`, with a wide prefix
(`0xFF`) promoting `b` to 32 bits for large constant pools. Full reference:
[docs/isa.md](docs/isa.md).

<table>
<tr><th align="left">Class</th><th align="left">Opcodes</th><th align="left">Notes</th></tr>
<tr>
<td valign="top"><b>Stack</b></td>
<td><code>LDC</code> <code>LDI</code> <code>DUP</code> <code>DUPN</code> <code>DROP</code> <code>SWAP</code> <code>ROT</code></td>
<td valign="top">Depth is a static property; see verifier.</td>
</tr>
<tr>
<td valign="top"><b>Data</b></td>
<td><code>PACK</code> <code>UNPACK</code> <code>IDX</code> <code>LEN</code> <code>CAT</code> <code>FMT</code></td>
<td valign="top">Heap-allocating; charged against the arena quota.</td>
</tr>
<tr>
<td valign="top"><b>Arithmetic</b></td>
<td><code>ADD</code> <code>SUB</code> <code>MUL</code> <code>DIV</code> <code>MOD</code> <code>EQ</code> <code>LT</code> <code>NOT</code></td>
<td valign="top">Wrapping i64 only. No floats — floats are not deterministic across hosts.</td>
</tr>
<tr>
<td valign="top"><b>Control</b></td>
<td><code>BR</code> <code>BRZ</code> <code>BRNZ</code> <code>CALL</code> <code>RET</code> <code>TAIL</code> <code>SWITCH</code> <code>TRAP</code></td>
<td valign="top">Targets are function-local; no computed jumps.</td>
</tr>
<tr>
<td valign="top"><b>Effects</b></td>
<td><code>CALLTOOL</code> <code>AWAIT</code> <code>POLL</code> <code>CANCEL</code></td>
<td valign="top">The only instructions that can suspend.</td>
</tr>
<tr>
<td valign="top"><b>Concurrency</b></td>
<td><code>SPAWN</code> <code>JOIN</code> <code>SELECT</code> <code>FORK</code> <code>COMMIT</code> <code>ABORT</code></td>
<td valign="top"><code>FORK</code> is copy-on-write over the heap arena.</td>
</tr>
<tr>
<td valign="top"><b>Durability</b></td>
<td><code>CHECKPOINT</code> <code>YIELD_CTX</code> <code>RESUME</code></td>
<td valign="top">Snapshot boundaries. Verified to occur at empty-pending points.</td>
</tr>
<tr>
<td valign="top"><b>Metering</b></td>
<td><code>RESERVE</code> <code>RELEASE</code> <code>SPEND</code> <code>QUERYQ</code></td>
<td valign="top">Two-phase: reserve before the call, spend on the answer.</td>
</tr>
<tr>
<td valign="top"><b>Context</b></td>
<td><code>CTXPUSH</code> <code>CTXPOP</code> <code>CTXWIN</code> <code>CTXCOST</code></td>
<td valign="top">Conversation context as an addressable ring, not a list you append to.</td>
</tr>
</table>

Deliberate omissions, each of which would break a property above:

- **No floating point.** `x87`/SSE rounding and `fma` contraction differ across
  targets. Determinism outranks convenience; use scaled integers.
- **No indirect branches.** The verifier's fixpoint needs a static CFG.
- **No host pointers, no FFI opcode.** Anything a host could inject would be
  unserializable and untrackable.
- **No unbounded loops without metering.** `BR` to a lower `pc` requires a
  dominating `SPEND` or `RESERVE`; enforced in [`verify.rs`](src/verify.rs) so a
  runaway agent burns budget rather than wall-clock.

---

## The verifier

Structurally a JVM-style verifier: abstract interpretation over the CFG,
iterated to a fixpoint, with merge points requiring the frames to unify. It runs
once at load and its result is cached in the image header alongside a hash of the
code section, so a re-run of the same image skips it.

What it proves, per basic block, for every reachable `pc`:

1. **Stack depth is single-valued.** Every path into a block agrees on depth.
   Disagreement is `E_DEPTH_MERGE`, reported with both predecessor blocks. This
   is what lets the interpreter index the stack without checking.
2. **Operand types unify.** The lattice is
   `⊥ ⊑ {i64, str, bytes, list, handle, pending<T>} ⊑ ⊤`, with `⊤` illegal at any
   use site. Merges compute the join; a join landing on `⊤` is a type error at the
   *merge*, not at the eventual use — which is why the diagnostics point at the
   branch rather than at a random instruction 40 lines later.
3. **No `pending<T>` escapes.** A value produced by `CALLTOOL` is `pending<T>`
   and only `AWAIT`, `POLL`, `CANCEL`, and `SELECT` consume it. Any other use, or
   reaching `RET`/`CHECKPOINT` with a live `pending`, is `E_PENDING_ESCAPE`. This
   single rule is what makes snapshots safe: no snapshot can contain a
   half-issued effect whose completion nobody is waiting for.
4. **`maxstack` is honoured.** The declared bound dominates the computed
   high-water mark. The interpreter allocates exactly `maxstack` slots per frame
   and never grows.
5. **Branch targets are in-range and land on instruction boundaries.** Trivial
   given fixed-width encoding, but checked, because the encoding is not the only
   producer of images.
6. **Metering dominates back-edges.** Every cycle in the CFG contains at least
   one metering instruction, proven with a dominator computation over the loop
   headers.
7. **`FORK`/`COMMIT`/`ABORT` are balanced along all paths,** and the fork depth
   at a merge agrees. Unbalanced is `E_FORK_IMBALANCE`.

Diagnostics carry source spans through the assembler, so a verify failure on
hand-written `.cdx` reads like a compiler error rather than an offset:

```
error[E_PENDING_ESCAPE]: pending value reaches `ret` unawaited
  ┌─ triage.cdx:21:9
  │
14│         calltool  %rank
  │         ───────────────  pending<str> produced here
  ·
21│         ret
  │         ^^^ still live at return; depth 1, slot 0
  │
  = a `pending` must be consumed by await/poll/cancel/select
  = help: insert `await` before `ret`, or `cancel` to discard the effect
```

The [conformance corpus](corpus/) contains 214 images that must be rejected, each
with the expected error code, plus 96 that must be accepted. `cinder-fuzz`
generates structurally-valid-but-semantically-broken bytecode by mutating the
accepted set; the invariant under test is that the interpreter never panics on
anything the verifier admits, and never runs anything it rejects.

---

## Continuations

A snapshot is the machine, flattened:

```
┌──────────────────────────────────────────────────────────────┐
│ magic "CDXC"  ver  flags  image_hash[32]  journal_seq        │  header, 56 B
├──────────────────────────────────────────────────────────────┤
│ frames:  [ pc, fn_id, base, depth, fork_depth ] × n          │  20 B each
├──────────────────────────────────────────────────────────────┤
│ operands: [ tag:u8, payload:u64, _pad ] × m                  │  16 B each
├──────────────────────────────────────────────────────────────┤
│ heap arena: bump-allocated bytes, relocated on restore       │  variable
├──────────────────────────────────────────────────────────────┤
│ context ring: window offsets + interned segment ids          │  variable
├──────────────────────────────────────────────────────────────┤
│ budget ledger: reserved / spent / limits                     │  48 B
├──────────────────────────────────────────────────────────────┤
│ blake3(all of the above)                                     │  32 B
└──────────────────────────────────────────────────────────────┘
```

The design constraints that shaped it:

- **Restore validates before it trusts.** A snapshot is a foreign byte string.
  `cont::restore` checks the hash, the image binding, frame/operand bounds, and
  every heap handle's target extent — then reconstructs. A snapshot from a
  different image is `E_IMAGE_MISMATCH`, never a wild jump.
- **Handles are arena-relative, so relocation is arithmetic.** No pointer
  patching, no identity map, no graph walk.
- **`FORK` is copy-on-write at page granularity** over the arena, so speculative
  branches are cheap until they write. Refcounts live outside the arena so a
  snapshot of a forked machine is still a flat copy. Invariants are asserted in
  `debug_assertions` builds after every arena mutation.
- **Snapshots are content-addressed and diffable.** Consecutive checkpoints in a
  long run share most of their arena, so `cinder snap diff a b` is a chunk-level
  comparison and the object store dedups.

```sh
$ cinder snap inspect run.cdxc
  image     triage      6f2a…c1  (matches)
  frames    1           main pc=0x1a
  operands  1/6         [0] str #3
  arena     1184 B      3 live handles, 0 B slack
  journal   seq 2       last: tool.answer #1
  budget    44/8000 tok   1/6 tools   0.61/45 s
```

---

## Determinism and replay

The interpreter is a pure function:

```
step : (Image, State, Answer?) -> (State, Trap?)
```

Nothing else reaches it. Time, randomness, tool results, spawned-child results,
and even the iteration order of `SELECT` are answers delivered by the host and
recorded in the journal. `NOW` and `RAND` are instructions that trap.

The journal is append-only and hash-chained — record *n* commits to
`blake3(prev_hash ‖ payload)` — so a truncated or edited journal is detectable
rather than silently divergent. Records are sealed *before* the answer reaches
the interpreter, which is the ordering that makes crash recovery correct: a crash
between "effect performed" and "answer recorded" is recoverable because the
record was written first and marked `in-flight`, and recovery reconciles by
querying the broker for that record's idempotency key.

```
┌───── record 41 ────┐ ┌───── record 42 ────┐ ┌───── record 43 ────┐
│ prev  a19f…        │ │ prev  7c02…        │ │ prev  e5b1…        │
│ kind  tool.issue   │ │ kind  tool.answer  │ │ kind  now          │
│ key   idem:9f3a…   │ │ ref   41           │ │ value 1756...      │
│ hash  7c02…        │ │ hash  e5b1…        │ │ hash  3d84…        │
└────────────────────┘ └────────────────────┘ └────────────────────┘
```

This gives three things that are usually mutually exclusive:

| | How |
|---|---|
| **Exact replay** | Same image + same journal ⇒ identical state at every step. Enforced, not hoped for: `--verify-replay` re-hashes state at each record and compares. |
| **Time travel** | `replay --seek N` restores the nearest checkpoint ≤ N and steps forward. Backward stepping is forward stepping from an earlier snapshot; there is no undo log. |
| **Divergence as a first-class error** | If the interpreter asks for something the journal does not have next, that is `E_DIVERGE` with the record index, the expected kind, and the requested kind. It means the image changed or the VM has a bug — the two things you actually want to know. |

```sh
$ cinder replay run.jl --verify-replay
  replaying 5 records against triage.cdxb (6f2a…c1)
  ✓ 5/5 states match recorded digests
  ✓ journal chain intact  (head 3d84…)
```

---

## The supervisor

Written in Go, in [`cmd/cinderd`](cmd/cinderd/) and [`internal/`](internal/). The
split is not decorative: the VM wants a single-threaded, allocation-frugal,
panic-free core, while the supervisor wants goroutines, `context` cancellation,
and an HTTP surface. They meet over a length-prefixed frame protocol on a pipe or
UDS ([docs/protocol.md](docs/protocol.md)), so a compromised or crashing tool
cannot take a VM's address space with it.

```
                     ┌────────────────────────────────────────┐
   HTTP / gRPC       │            cinderd                     │
  ───────────────►   │                                        │
                     │  ┌──────────┐   admission + fair queue │
                     │  │ scheduler│   weighted by tenant     │
                     │  └────┬─────┘                          │
                     │       │ lease                          │
                     │  ┌────▼─────┐   ┌──────────┐           │
                     │  │  broker  │──►│  ledger  │  quotas   │
                     │  └────┬─────┘   └──────────┘           │
                     └───────┼────────────────────────────────┘
                             │ frames (UDS)
                ┌────────────┼────────────┬────────────┐
                ▼            ▼            ▼            ▼
             vm #1        vm #2        vm #3        vm #4      (separate procs)
```

Responsibilities, in the order they matter:

- **Scheduler.** VMs are cooperatively descheduled at `AWAIT`/`YIELD_CTX`. A
  suspended VM costs a snapshot, not a thread, so the concurrency ceiling is
  memory, not goroutines. Fair queueing is deficit round-robin over tenants with
  weights; a single tenant cannot starve others by spawning.
- **Syscall broker.** Owns tool dispatch, retries with jittered backoff,
  idempotency keys, and per-tool circuit breakers. Every dispatch is journalled
  before it leaves and reconciled on return.
- **Ledger.** Two-phase budget. `RESERVE` takes an optimistic lease against the
  tenant's remaining allowance; `SPEND` settles it with the real usage;
  `RELEASE` returns unused reservation. Overspend is refused at reserve time, so
  a run cannot exceed its budget and then apologize.
- **Recovery.** On startup, scans the snapshot store for runs whose lease
  expired, reconciles their in-flight journal records against the broker, and
  reschedules from the last checkpoint.

> [!NOTE]
> `cinderd` binds `127.0.0.1:7749` with **no authentication by default** — it is
> built for a trusted network boundary with the real gateway in front. Before
> exposing it, read [docs/deployment.md](docs/deployment.md#authentication) and
> set `CINDERD_AUTH`. The `/debug/vm` endpoint exposes full machine state,
> including tool arguments, and must not be reachable from outside the host.

---

## Performance

`cargo bench` on the committed corpus; 12-core Zen 4, Linux 6.11, `--release`.
Numbers are p50 with p99 in parentheses. Reproduce with
`cargo bench -- --save-baseline main` — [docs/benchmarks.md](docs/benchmarks.md)
covers the methodology and the outlier handling.

| Operation | Result | Notes |
|---|---|---|
| Dispatch, arithmetic-heavy loop | **41 M insn/s** (38 M) | Computed-goto-shaped `match`; the bottleneck is the store to `sp`. |
| Dispatch, effect-heavy | 9.1 M insn/s | Trap construction dominates. |
| Verify, 4 KiB image | **1.9 ms** (2.4 ms) | Fixpoint converges in ≤3 passes on all corpus images. |
| Snapshot, 64 KiB arena | **112 µs** (140 µs) | Two memcpys and a hash; hash is 70% of it. |
| Restore + validate | 138 µs (171 µs) | Validation is 60% — deliberately not optional. |
| `FORK`, 1 MiB arena | **8 µs** | COW; independent of arena size until first write. |
| Journal append, fsync | 1.1 ms | Group-committed; 84 µs at batch 16. |
| Suspended VM, resident | **2.3 KiB** | vs. ~8 KiB for a parked goroutine, ~40 KiB for a Python task. |

The interpreter's inner loop is the one place in the codebase where clarity was
traded for speed, and it is commented accordingly. Everything else is written to
be read.

---

## Building from source

Requires Rust ≥ 1.78 (pinned in [`rust-toolchain.toml`](rust-toolchain.toml)) and
Go ≥ 1.22. No C toolchain, no system libraries, no build scripts — the Rust crate
has **zero dependencies** outside `core`/`alloc`/`std`, including the hash, the
arena, and the CLI argument parsing.

```sh
git clone https://github.com/ashish/cindervm && cd cindervm

make            # debug build of both halves
make release    # LTO, single codegen unit, symbols stripped
make test       # unit + corpus conformance + Go tests + cross-language e2e
make verify     # clippy -D warnings, rustfmt, go vet, staticcheck
make bench      # criterion, baseline-compared
make fuzz T=60  # 60s of bytecode fuzzing against the verifier/interpreter pair
```

Individually:

```sh
cargo build --release --workspace
cargo test  --all-features
cargo clippy --all-targets -- -D warnings
go build ./cmd/...
go test  ./... -race
```

The [Makefile](Makefile) is the source of truth for what CI runs; the
[workflow](.github/workflows/ci.yml) calls the same targets so a green local
`make verify && make test` means a green CI. The matrix covers
`{linux, macos, windows} × {stable, 1.78, nightly}`, with nightly allowed to fail
and Miri run on the arena and continuation modules only (they are where aliasing
mistakes would hide, even under `#![deny(unsafe_code)]` — `Miri` also catches UB
in the standard library calls we make).

<details>
<summary><b>Cross-compiling for the supervisor's target</b></summary>

The VM binary is the only thing that needs to match the tool-execution host.
Static musl builds are the intended deployment:

```sh
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl --bin cinder
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath -ldflags='-s -w' ./cmd/cinderd
```

Both artifacts are then self-contained; the container is `FROM scratch` plus the
two binaries. See [docs/deployment.md](docs/deployment.md).
</details>

<details>
<summary><b>Regenerating the ISA tables and docs</b></summary>

[`src/isa.rs`](src/isa.rs) is the single definition of the instruction set — the
opcode table, operand shapes, stack effects, and type rules all live in one
`const` array. The assembler's mnemonic table, the disassembler, the verifier's
transfer functions, and [docs/isa.md](docs/isa.md) are all derived from it:

```sh
cargo run --bin cinderc -- --emit-isa-md > docs/isa.md
cargo test  isa::table_is_dense   # every opcode 0..N accounted for, no gaps
```

`make verify` fails if the checked-in `docs/isa.md` differs from the generated
one, so the reference cannot drift from the implementation.
</details>

---

## Repository layout

The Rust core is deliberately flat. Modules are large because the boundaries
follow the machine's structure, not a file-size preference.

```
cindervm/
├── src/
│   ├── lib.rs             crate root, invariant docs, #![deny(unsafe_code)]
│   ├── isa.rs             opcode table, encoding, stack effects, type rules
│   ├── value.rs           tagged 16-byte Value, arena handles, coercions
│   ├── lex.rs             .cdx tokenizer with span tracking
│   ├── asm.rs             parser, symbol resolution, fixups, encoder
│   ├── image.rs           .cdxb container: sections, header, sealing
│   ├── verify.rs          abstract interpreter, type lattice, CFG fixpoint
│   ├── cfg.rs             basic blocks, dominators, loop headers
│   ├── interp.rs          the dispatch loop and instruction semantics
│   ├── frame.rs           call frames, operand stack windows
│   ├── heap.rs            bump arena, COW pages, handle validation
│   ├── cont.rs            snapshot / restore, relocation, validation
│   ├── journal.rs         hash-chained record log, cursor, reconciliation
│   ├── replay.rs          journal-backed host, divergence detection
│   ├── budget.rs          two-phase reservation ledger
│   ├── ctx.rs             context ring, windowing, token accounting
│   ├── trap.rs            the interpreter↔host boundary type
│   ├── wire.rs            frame protocol codec
│   ├── diag.rs            spans, error codes, rendered diagnostics
│   ├── disas.rs           disassembler and the `cinder dis` output
│   └── bin/
│       ├── cinderc.rs     assembler CLI
│       ├── cinder.rs      run / replay / snap / dis
│       └── cinder_fuzz.rs mutation fuzzer
├── cmd/cinderd/           supervisor entrypoint
├── internal/
│   ├── sched/             deficit round-robin, leases
│   ├── broker/            tool dispatch, retries, circuit breakers
│   ├── ledger/            tenant quotas
│   └── wire/              Go side of the frame protocol
├── corpus/                310 conformance images with expected outcomes
├── examples/              runnable .cdx agents
├── docs/                  isa, protocol, determinism, deployment, benchmarks
└── tests/                 integration + cross-language end-to-end
```

Every module is documented at the top with what it guarantees and what it assumes
the caller has already proven. [docs/architecture.md](docs/architecture.md) is
the long form.

---

## Stability

`0.x`. The library API is not stable. Two things are:

- **The `.cdxb` container** is versioned and will be read by future minor
  versions. Images do not need recompilation.
- **The journal format** is append-only and forward-compatible; a journal written
  by `0.4` replays on `0.5`.

The ISA itself is versioned separately (`cdx/4`). Adding opcodes bumps the minor;
changing the meaning of one bumps the ISA version and images declare which they
target. `CHANGELOG.md` tracks both.

---

## FAQ

**Why not just use a durable execution framework?**
They recover by re-executing and short-circuiting from a log, which requires your
code to be deterministic and punishes you subtly when it isn't. `cindervm`
restores state rather than re-deriving it, and makes non-determinism impossible
at the ISA level instead of asking you to be careful.

**Is `.cdx` meant to be written by hand?**
For tests and examples, yes. In practice you generate it — the assembler is a
library, and `asm::Builder` is the intended interface for a higher-level frontend.
Writing one is the obvious next project and deliberately out of scope here.

**Why is the interpreter single-threaded?**
Because `SPAWN` creates a VM, not a thread. Parallelism belongs to the
supervisor, which can place VMs across processes and machines. A multi-threaded
interpreter would make state non-serializable, which is the one thing this design
will not trade.

**How does this handle streaming tool results?**
`POLL` returns `pending` unchanged until the answer is complete; partial chunks
are journalled as `tool.chunk` records and accumulate in the arena. Replay
reproduces chunk boundaries exactly, which turns out to matter for reproducing
bugs in streaming parsers.

**Zero dependencies — really?**
Really, for the core crate. blake3 is ~200 lines, the arena is ~400, arg parsing
is ~150. The tradeoff is deliberate: this is a trust-boundary component, and every
dependency is code you are also trusting. Dev-dependencies (criterion,
proptest) are not so constrained.

---

## License

Apache-2.0 OR MIT, at your option. See [LICENSE-APACHE](LICENSE-APACHE) and
[LICENSE-MIT](LICENSE-MIT).

<div align="center">
<sub>Contributions welcome — read <a href="CONTRIBUTING.md">CONTRIBUTING.md</a> first; the corpus has rules.</sub>
</div>

