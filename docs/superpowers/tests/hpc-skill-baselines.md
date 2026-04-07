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

*(To be filled in during Task 7 after all skills are written)*

### Scenario A Result
*(pending)*

### Scenario B Result
*(pending)*

### Scenario C Result
*(pending)*

### Scenario D Result
*(pending)*
