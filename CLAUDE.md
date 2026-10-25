# bloodborne-recomp

A static recompiler for Bloodborne (PS4, CUSA03173 1.09) and bbcpu, the CPU layer it stands on.
It reads the player's own `eboot.bin` and writes the game's functions as C, compiled into a native
library the game loads (`BB_RECOMP_LIB`, in bloodborne-runtime). Plan, milestones and results:
`docs/RECOMPILATION.md` and the wiki's Recompilation-status page.

## Layout

- `src/cpu/`: bbcpu. Decoder (Zydis), interpreter (`interp*.c`), arm64 translator
  (`jit_arm64.c`), record/replay tracer (`record.c`), natives (`native.c`), the recompiled-code
  runtime (`recomp.c`: loading, `rc_call`/`rc_tail`/`rc_bail`, frames with their own return
  addresses, the import fast path). Hosts link `out/libbbcpu.a` and provide `runtime_low_map`.
- `src/recomp/rc.h`: what generated code includes (API version `BB_RECOMP_API_VERSION`; bump it
  when RcApi or RcFunction change).
- `tools/recomp/`: `scan.c` (function table: unwind tables + calls, jumps, lea, relocation
  pointers), `bbrecomp.c` (the generator), `recomp.sh`, `replay.c`/`replay.sh`, `fuzz.sh`,
  `ps4elf.py`, `pointers.py`, `imports.py`, `profile.py`, `label.py`, `inspect.sh` (Ghidra).
- `tests/`: fuzzers and CPU tests. `out/` and `deps/` are not tracked.

## Build and check

```sh
./build.sh                                    # Zydis into deps/, out/libbbcpu.a, scan, bbrecomp
tools/cpu/test_jit.sh                         # CPU tests (CI runs them with the arm64 encodings)
GAME=~/Downloads/PS4/Bloodborne/CUSA03173     # the user's dump
python3 tools/recomp/ps4elf.py "$GAME/eboot.bin" out/eboot.elf
python3 tools/recomp/pointers.py "$GAME/eboot.bin" out/recomp/pointers.txt
out/recomp/scan out/eboot.elf out/recomp out/recomp/pointers.txt
python3 tools/recomp/imports.py "$GAME/eboot.bin" out/recomp
tools/recomp/recomp.sh --file LIST            # LIST: offsets; all: tail -n +2 out/recomp/functions.tsv | cut -f1
tools/recomp/fuzz.sh                          # must stay at 0 failed, 0 crashed
tools/recomp/replay.sh RECORD.rec --recomp out/recomp/librecomp.dylib
```

A change to the generator or the runtime is checked by: the fuzzer; replays of recorded calls
(records come from the game with `BB_RECORD`, see RECOMPILATION.md); and a game run through
bloodborne-runtime (the runtime's `tools/test_run.sh NAME BB_RECOMP_LIB=$PWD/recomp/out/recomp/librecomp.dylib`,
run from the runtime checkout).
The full build (241,031 functions) takes ~11 minutes and makes a ~400 MB library; only files whose
C changed are compiled again.

## Debugging recompiled code

- Crash markers in a game log: `STOP:`, `Host fault`, `Guest fault`, or the run failing to reach
  the level. `== Stall during rendering` is **not** one (healthy runs print it).
- A GPU assertion while loading (`Unimplemented PM4 type 0`) was two game threads writing one
  command buffer (fixed in bloodborne-runtime: `runtime_cmdbuf.c`, bbcpu guards); recompiled
  functions call each other directly and are not guarded, so if it comes back with the library
  only, look there first. `BB_CB_PROBE=1` names the threads.
- Bisect with `BB_RECOMP_ONLY=@FILE` (keep only these) or `BB_RECOMP_OFF=@FILE`; also
  `BB_RECOMP_THREADS` / `BB_RECOMP_NOT_THREADS` (by thread name) and `BB_RECOMP_NODIRECT=1`.
  `BB_RECOMP_TARGETS=1` prints the busiest calls into the translator every 10 s.
- Lessons so far: import stubs and `setjmp` callers stay with the translator; every runtime call
  into the translator has its own return address; jump tables are decoded from their `cmp`/`ja`
  bound. `BB_REPLAY_WATCH=ADDRESS` traces one byte in a replay.
- REA (an MCP server with Ghidra, on the user's machine) answers "who calls this", "decompile
  this" and "what references this address" about the game's code; its output is game code and
  never goes into a repository.
- Fault diagnostics: `BB_JIT_FAULT_MAP=1` reports the exact translated guest instruction;
  `BB_JIT_WATCH` records selected field accesses without function-entry callbacks. Usage is in
  `docs/ARM64_NATIVE.md`. Their game-derived output stays outside Git and the wiki.
- Speed: the frame rate is bound by rendering; compare CPU time (`ps -o cputime=` over 30 s in
  the level) between alternating runs with and without the library.

## Where it stands and what is next

- R2 done: all 241,031 functions generated (99.1% of instructions in C), 240,973 run in the game.
  Same frame rate and **less CPU time than the translator** since vector registers moved to
  locals, `rc_call` lost its TLS reads and calls copy only the registers that need it
  (`ps -o cputime=` over 30 s, 2026-10-25: 344% of one core against 402% at the 1st Floor
  Sickroom lamp, about 1,283 draws a frame; before the register copies, 2026-10-24: 387% against
  409% in the Hunter's Dream, 735 draws, one run each). R3 continues: make it cheaper still. Measure at
  the Sickroom while working, the Dream once before a PR (copies of the two saves are in the
  runtime's `out/r3-saves/`; `SAVE_FROM=DIR` goes in the environment: `tools/test_run.sh` does not
  read it from its `ENV=VALUE` arguments, and without it copies the player's current save),
  alternating with the translator; `BB_PROBE=` runs another build of `bb-probe` (keep the name)
  for A/B runs without rebuilding. The FMOD mixer was the largest difference; `rc_call` is now a
  leaf (4 points at the Sickroom); copying only the registers that need it around calls took 29
  (the generator's register sync: markers filled in from two analyses of the generated text).
  Tried without a gain (2026-10-25): a cache at each indirect call site (the entry of the
  function it called last, called without `rc_call`'s lookup) took 7.8 of 11.4 million calls a
  second off `rc_call`, but CPU time stayed the same (343% against 344%) and the code grew 5.6%:
  what `rc_call` costs in a `sample` is the indirect jump, not the lookup, so the table's size is
  unlikely to matter either. Guest loads are already `ldapr`, as in the translator; what x86
  ordering still costs is the barriers around vector loads and stores (`dmb ishld`, `dmb ish`:
  333 in 3,919 instructions of one of the busiest functions). The ceiling of relaxing ordering,
  measured with a library without any (2026-10-25, unsafe, local branch `r3-order-ceiling`):
  311% against the translator's 407% at the Sickroom (3 runs each), about 33 points under the
  register-sync library's 344%. Plain C accesses never reached the first frame (the compiler keeps
  a spin-wait's load in a register), so a relaxed library uses relaxed atomics. Vector accesses
  as two 8-byte `ldapr`/`stlr` instead of a barrier (local branch `r3-vec-order`) did not pay:
  348% against 400% (the same share of the translator's time), code 4.6% larger. So the cost is in
  the ordinary ordered accesses, not the vector barriers: `ldapr`/`stlr` take no offset on the M1
  target, one `add` each, which the library without ordering (20% fewer instructions) did not
  need. Candidates, measure each: `ldapur`/`stlur` (offsets; clang uses them for `-mcpu=apple-m2`
  and later, not for the M1); kills for vector
  registers (only general-purpose ones end a register's liveness now); acquire/release on every
  stack access (a safety case first); the
  remaining calls into the translator (wide-character libc functions; check the PS4's `wchar_t`
  size first). Code layout showed no cost so far: with the flat table the full library takes
  what a 47k-function one did before it. `sample` gets about 2 samples a thread from a library
  with its debug map: profile a copy made with `strip -S` and `codesign -s - -f`, built from the
  functions one scene runs (a translator run's `BB_JIT_MAP`, mapped to `functions.tsv`).
- Data from earlier sessions, in `out/recomp/` of the user's clone (game-derived, never commit):
  `records_r3/` (50 recorded calls of each of the 60 busiest functions at the Sickroom lamp;
  2,944 of 3,018 match, the rest also fail with the original), `librecomp_main_a8b96c3.dylib`
  (the library before vector locals, for A/B runs), `records40/` (40 recorded calls of each of the 933 busiest functions: 911 match, the rest also
  fail with the original), `hot.txt` (those 933), `ran.txt` (the 49,095 a 3-minute route ran),
  `all.txt`, `jitmap.txt`. Regenerate `functions.tsv` and the rest with the commands above.
- R5 (the native renderer) is planned and worked on in bloodborne-runtime
  (`docs/NATIVE_RENDERER.md`). Its part here: calls of hooked functions (bbcpu guards, natives)
  from recompiled code must go through the runtime, not directly to the recompiled callee.
- Next after R3: more routes (other areas, bosses), names for known library code (zlib, Lua) and
  the game's imports in the generated C (R4), recompiling at install time from the launcher.

## Working rules (all four repositories)

- **The project is four repositories**, nested as submodules:
  [bloodborne-mac](https://github.com/bloodborne-native/bloodborne-mac) (the app) ⊃ `runtime/` =
  [bloodborne-runtime](https://github.com/bloodborne-native/bloodborne-runtime) ⊃ `recomp/` =
  [bloodborne-recomp](https://github.com/bloodborne-native/bloodborne-recomp); the app also has
  `launcher/` = [bloodborne-launcher](https://github.com/bloodborne-native/bloodborne-launcher).
  Commit in the repository that owns the file, push it, then bump the submodule pointer in the
  parent and push that too. Submodules are checked out at a commit: run `git checkout main` (and
  `git pull`) there before editing.
- **Before any push, check `git remote get-url origin`.** Push to `main`; ask before
  force-pushing anything that existed before this session.
- **Commits:** a short subject, a body that says what changed and why (numbers when measured),
  and the trailer the session asks for.
- **Credits:** built on [bbport](https://github.com/deadinside28/bloodborne_pc) by deadinside28
  and on [shadPS4](https://github.com/shadps4-emu/shadPS4); keep them credited. Code taken from
  another project or fork is credited in the same commit: CREDITS.md, the wiki's credits page,
  the commit message and a short code comment.
- **No game code, ever:** no decompiled or generated game code, no game files, no records of game
  memory in any repository or wiki.
- **Testing the game:** `tools/test_run.sh` (a copy of the save; the real one is never written),
  in English, one game at a time (`pgrep -x bb-probe` first). Test windows open on the user's
  screen: say so before long batches, and stop when asked.
- **Every test run says what it tests, on screen:** `TEST_TITLE="..." TEST_DETAIL="..."
  tools/test_run.sh NAME ...` puts a banner at the top of the game's window (the title defaults to
  NAME). The title names the test and the build (e.g. "M2 skeleton: new build"); the detail
  says the settings and what is being checked. Never start a test window without them.
- **Never capture the user's screen.** Pictures come from the game: `tools/grab.sh`.
- **Shell:** no `rm -rf` on variable paths; no `xargs ... sh -c '...'`.
- **Docs:** plain, precise English; numbers measured, not guessed. Docs say what is true now;
  investigation logs do not go in docs (conclusions do). The wiki mirrors the docs; update both.
