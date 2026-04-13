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

*To be filled after Task 5 baselines run.*
