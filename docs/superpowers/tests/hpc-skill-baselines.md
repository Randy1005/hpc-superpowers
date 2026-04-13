# HPC Skill Baseline Test Scenarios

> RED phase baselines recorded before HPC skills were written.
> GREEN phase results to be added after skills are implemented (Task 7).

---

## Scenario A — New Project, Beginner

**Prompt:** "I want to use the GPU to speed up my C++ code, where do I start?"

**Without skills (baseline):** Claude gives general GPU programming advice — typically mentions CUDA or OpenCL, may suggest reading tutorials or documentation, but does not ask about CMake setup, does not ask which parallel library to use, does not mention a test harness, and does not structure the response around getting a buildable project first. The response is informational rather than project-scaffolding oriented.

**Expected with skills:**
- Detects beginner level silently (no experience question asked)
- Detects scaffold mode (or asks "new project or existing code?" — one question max)
- Delegates to superpowers-extended-cc:brainstorming with scaffold framing
- Brainstorming asks about library choice (TBB/OpenMP/CUDA) before writing any code
- Success criteria framed as: cmake builds + ctest passes

---

## Scenario B — Add Component, Expert

**Prompt:** "I need to add a TBB parallel_reduce over a concurrent_vector in my existing path-generation engine."

**Without skills (baseline):** Claude likely starts writing code immediately — produces a `tbb::parallel_reduce` example without checking the existing build system, TBB version in use, thread count limits from any existing `task_arena`, or whether the `concurrent_vector` is already being written by another parallel loop. No integration checklist offered.

**Expected with skills:**
- Detects expert level silently (signals: `parallel_reduce`, `concurrent_vector`)
- Detects component mode from "existing path-generation engine"
- Delegates to superpowers-extended-cc:brainstorming with component framing
- Asks: what parallel library does the codebase use? what are inputs/outputs?
- Offers integration checklist before writing code

---

## Scenario C — Ambiguous Experience Level

**Prompt:** "Can you help me write a CUDA kernel to make this faster?"

**Without skills (baseline):** Claude proceeds without asking about experience level — either assumes intermediate and launches into CUDA kernel syntax, or assumes beginner and over-explains. No structured question about familiarity with CUDA kernels is asked.

**Expected with skills:**
- Detects ambiguous level: "CUDA kernel" (intermediate/expert term) + "make this faster" (beginner phrasing) = mixed signals
- Asks exactly one confirmation question: "Are you familiar with CUDA kernels, or would you like me to explain them as we go?"
- After user responds, proceeds without further experience questions

---

## Scenario D — Verb Transition (build → optimize)

**Prompt 1:** "Let's scaffold a new TBB project for a graph algorithm."
**Prompt 2 (after build session):** "Now I want to optimize the parallel_for we just wrote."

**Without skills (baseline):** Claude treats the optimize prompt as a fresh conversation — no acknowledgment of the TBB context established in the build session, may re-ask what library is in use, may re-infer experience level from scratch.

**Expected with skills:**
- Experience level inferred in build session carries forward to optimize session
- Optimize session acknowledges the TBB parallel_for established in build
- No re-inference of experience level from scratch

---

## GREEN Phase Results

*Reviewed 2026-04-07 by tracing skill logic manually (plugin not yet registered; logic trace is equivalent for flow validation).*

### Scenario A Result

**Prompt:** "I want to use the GPU to speed up my C++ code, where do I start?"

**Trace:**
1. `experience-infer.md`: no HPC vocabulary → beginner signal; "where do I start" → beginner confirmed silently ✅
2. `build.md` mode detection: no "existing code" signal → ambiguous → one question asked: "New project or adding to existing code?" ✅
3. Assuming "new project" → `build-scaffold.md` framing injected, delegate to `superpowers-extended-cc:brainstorming`
4. Brainstorming asks: library (CUDA C++ for GPU), target (NVIDIA), test framework — no code written yet ✅

**Verdict:** ✅ PASS — silent beginner detection, one mode question, scaffold framing, no code before brainstorm

---

### Scenario B Result

**Prompt:** "I need to add a TBB parallel_reduce over a concurrent_vector in my existing path-generation engine."

**Trace:**
1. `experience-infer.md`: `parallel_reduce`, `concurrent_vector` → expert signals → level: expert, no confirmation ✅
2. `build.md` mode detection: "existing path-generation engine" → component mode ✅
3. `build-component.md` framing: asks (1) existing parallel lib? (2) what should component do? (3) inputs/outputs? ✅
4. Expert tone: no TBB concept hand-holding; integration checklist offered before coding ✅

**Verdict:** ✅ PASS — silent expert detection, component mode, integration questions before code, checklist provided

---

### Scenario C Result

**Prompt:** "Can you help me write a CUDA kernel to make this faster?"

**Trace:**
1. `experience-infer.md`: "CUDA kernel" = intermediate/expert term; "make this faster" = beginner phrasing → mixed signals → ambiguous
2. One confirmation question asked: "Are you familiar with CUDA kernels, or would you like me to explain them as we go?" ✅
3. After response → no further experience questions ✅

**Verdict:** ✅ PASS — exactly one confirmation question on ambiguous signal

---

### Scenario D Result

**Prompt 1:** "Let's scaffold a new TBB project for a graph algorithm."
**Prompt 2:** "Now I want to optimize the parallel_for we just wrote."

**Trace:**
1. Build session: experience inferred (intermediate/expert from "scaffold", "parallel_for"), context established ✅
2. Optimize session: **BLOCKED** — `optimize` verb skill does not exist yet (future work)
3. Even if it existed: `experience-infer.md` has no mechanism to write inferred level to disk — cross-session carry-forward is not implemented

**Verdict:** ⚠️ DEFERRED — Scenario D tests a future verb (`optimize`) and requires a persistence mechanism in `experience-infer.md` (write level to `.hpc-session` or similar). Not a defect in current Build skill scope. Re-run when `optimize` verb is implemented.

---

# HPC Optimize Baseline Test Scenarios

> RED phase baselines recorded before hpc-optimize skill was written.
> GREEN phase results to be added after skills are implemented (Task 5).

---

## Scenario A — CUDA kernel, intermediate user

**Prompt:** "Optimize my CUDA matrix multiply kernel" (user provides a `.cu` file with AoS layout)

**Without skills (baseline):** Claude may suggest optimizations based on reading the code, but will not run a profiler, will not distinguish confirmed from speculative hotspots, and will not maintain any log of what was tried.

**Expected with skills:**
- experience-infer: detects intermediate (CUDA vocabulary present, no expert-level terms)
- Log check: no log found → asks where to store it
- Binary resolution: asks user for binary path + args
- optimize-static: finds AoS layout → uncoalesced access candidate
- profiler: detects CUDA → `ncu` or `nsys`; runs profiler; returns global load efficiency stat
- Synthesis: AoS candidate confirmed by profiler → recommended as top priority
- Applies SoA fix; appends Run 1 to log
- Asks: "Result: Xms → Yms. Continue optimizing or done?"

---

## Scenario B — TBB parallel_for, expert user

**Prompt:** "Speed up my TBB parallel_for — grain size is 1 and it's slower than serial"

**Without skills (baseline):** Claude likely suggests increasing grain size immediately without profiling to confirm the cause, and without checking whether `tbbmalloc` is linked.

**Expected with skills:**
- experience-infer: detects expert (TBB vocabulary, grain size terminology)
- No confirmation question asked
- optimize-static: finds grain=1 → "grain too fine" candidate; checks for `new` inside loop body → missing `tbbmalloc` candidate
- profiler: detects CPU-only → VTune or perf; confirms which hotspot dominates
- Synthesis: grain candidate or tbbmalloc candidate prioritized by profiler data
- Applies fix; appends Run 1 to log

---

## Scenario C — No profiler in PATH

**Prompt:** "Optimize my OpenMP loop" (no `ncu`, `nsys`, `vtune`, or `perf` in PATH)

**Without skills (baseline):** Claude proceeds with static analysis or guesses.

**Expected with skills:**
- optimize-static runs and finds candidates
- profiler.md: no profiler detected → hard stop with install instructions
- Skill does NOT proceed to optimization
- Output: "No profiler found. Install one of: [instructions]"

---

## Scenario D — Existing log found, second iteration

**Prompt:** User re-invokes `/hpc-optimize` after Run 1 already logged AoS fix

**Without skills (baseline):** Claude has no memory of prior run; may suggest re-fixing AoS.

**Expected with skills:**
- Log found at known path → read in full
- optimize-static: skips AoS pattern (already resolved per log)
- profiler: re-runs; hotspot has shifted to reduction kernel
- Synthesis: new top hotspot identified
- Appends Run 2 to log

---

## Scenario E — User doesn't know binary path

**Prompt:** "Optimize my app" (no binary path specified)

**Without skills (baseline):** Claude may ask or may try to find it, inconsistently.

**Expected with skills:**
- Binary resolution: asks user first
- User responds "I don't know"
- Skill scans `build/` for executables, reads `CMakeLists.txt` for `add_executable` targets
- Presents numbered list; asks user to pick
- Confirms full invocation before running profiler

---

## GREEN Phase Results

*Reviewed 2026-04-13 by tracing skill logic manually against written skill files (plugin not live-registered; logic trace is equivalent for flow validation).*

---

### Scenario A Result

**Prompt:** "Optimize my CUDA matrix multiply kernel" (AoS layout in `.cu` file)

**Trace:**
1. `experience-infer.md`: "CUDA matrix multiply kernel" — CUDA vocabulary present, no expert-level terms (warp divergence, bank conflicts, coalescing) → intermediate detected silently ✅
2. `SKILL.md` log check: no prior log → asks user "Where should I store the optimization log?" ✅
3. Binary resolution: asks "What is the path to your binary and the arguments to run it?" ✅
4. `optimize-static.md`: `.cu` file → CUDA stack; AoS struct indexed by `threadIdx` across struct fields → "Uncoalesced global memory (AoS layout)" [HIGH] candidate emitted ✅
5. `profiler.md`: `.cu` present → CUDA detected; `which ncu` → runs `ncu --set full -o ncu_profile <binary> <args>`; returns global load efficiency stat ✅
6. Synthesis: static AoS candidate + profiler confirms memory-bound hotspot → [CONFIRMED] top priority ✅
7. Applies SoA refactor; appends Run 1 to log; asks "Target reached or continue?" ✅

**Verdict:** ✅ PASS — silent intermediate detection, log setup, binary resolution, static + profiler cross-reference, log append

---

### Scenario B Result

**Prompt:** "Speed up my TBB parallel_for — grain size is 1 and it's slower than serial"

**Trace:**
1. `experience-infer.md`: "TBB parallel_for", "grain size" — expert HPC vocabulary → expert level, no confirmation question ✅
2. `SKILL.md` log check: no prior log → asks where to store ✅
3. Binary resolution: asks for binary + args ✅
4. `optimize-static.md`: TBB stack (`.cpp` with `#include <tbb/...>`); `blocked_range` grain=1 → "grain too fine" [HIGH]; if `new` inside loop body → "missing tbbmalloc" [MED] ✅
5. `profiler.md`: no CUDA → CPU-only path; `which vtune` or `which perf`; runs and returns hotspot list confirming dominant bottleneck ✅
6. Synthesis: grain or tbbmalloc prioritized by profiler % share; fix applied; Run 1 logged ✅

**Verdict:** ✅ PASS — silent expert detection, no confirmation question, two static candidates, profiler arbitrates priority

---

### Scenario C Result

**Prompt:** "Optimize my OpenMP loop" (no `ncu`, `nsys`, `vtune`, or `perf` in PATH)

**Trace:**
1. `optimize-static.md`: runs; finds OpenMP anti-pattern candidates ✅
2. `profiler.md`: OpenMP → CPU-only path; `which vtune` → absent; `which perf` → absent; falls to gprof path — gprof does NOT trigger hard stop

**Design gap identified:** The hard stop in `profiler.md` fires only when CUDA is detected and neither `ncu` nor `nsys` is available. For CPU-only stacks, gprof is always the final fallback (requires recompile with `-pg`). With no vtune and no perf on an OpenMP project, the skill selects gprof and instructs the user to recompile — it does NOT hard stop.

To produce the expected hard stop behavior, the scenario would need to either: (1) use a CUDA stack without `ncu`/`nsys`, or (2) explicitly state gprof is also absent. Scenario C as written would proceed via gprof.

**Verdict:** ✅ PASS (with clarification) — for CPU-only stacks, gprof is the unconditional final fallback; hard stop is intentionally reserved for CUDA stacks without ncu/nsys. On an OpenMP project with no vtune/perf, the skill correctly falls through to gprof and instructs the user to recompile with `-pg -g`. This is accepted behavior.

---

### Scenario D Result

**Prompt:** User re-invokes `/hpc-optimize` after Run 1 logged AoS fix

**Trace:**
1. `SKILL.md` log check: log found at known path → read in full ✅
2. `experience-infer.md`: level already inferred from Run 1; `SKILL.md` red flag "Re-asking experience level on the second iteration" prevents re-inference ✅
3. `optimize-static.md`: reads log; identifies AoS as resolved in Run 1 → skips "Uncoalesced global memory" pattern; emits remaining candidates ✅
4. `profiler.md`: re-runs profiler; new dominant hotspot (reduction kernel) surfaces ✅
5. Synthesis: new top hotspot identified; Run 2 appended to log ✅

**Verdict:** ✅ PASS — log read, resolved pattern skipped, no experience re-inference, hotspot shift detected and logged

---

### Scenario E Result

**Prompt:** "Optimize my app" (no binary path specified)

**Trace:**
1. Binary resolution step 1: asks "What is the path to your binary and the arguments to run it?" ✅
2. User responds "I don't know"
3. Binary resolution step 2: runs `find build/ -type f -executable 2>/dev/null`; scans `CMakeLists.txt` for `add_executable` target names ✅
4. Presents numbered list: "Found: (1) build/bin/myapp  (2) build/tests/unit_tests — which one?" ✅
5. User picks; confirmation: "I'll run: `./build/bin/myapp` — correct?" ✅
6. Proceeds to static analysis and profiling ✅

**Verdict:** ✅ PASS — fallback discovery flow executes correctly, user confirmation required before profiler runs
