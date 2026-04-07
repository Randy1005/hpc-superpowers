---
name: hpc-build-component
description: Use when hpc/build.md has identified existing-codebase mode — sets component-addition framing before invoking brainstorming
---

# HPC Build — Component

**Core principle:** A new parallel component must not break existing tests and must be verifiable in isolation before integration.

## Starting Framing for Brainstorming

Pass this as the initial context to superpowers-extended-cc:brainstorming:

> "We're adding a new parallel component to an existing C++ codebase. The component must: compile in isolation, have its own tests, and integrate without breaking the existing test suite. We do not start writing the component until we know its inputs, outputs, and which parallel library the codebase already uses."

## Clarifying Questions (ask in order, stop when you have enough)

1. **Existing parallel library?** (TBB/OpenMP/CUDA/Taskflow — match the codebase to avoid mixing schedulers)
2. **What should the component do?** (algorithm, pipeline stage, data structure transformation, etc.)
3. **Inputs and outputs?** (types, sizes, ownership — e.g., `const std::vector<float>&` in, `tbb::concurrent_vector<float>` out)

## Integration Checklist

Before declaring the component done:

- [ ] Component compiles in its own translation unit without touching other files
- [ ] No shared mutable state with existing code (or existing mutex/arena covers it)
- [ ] Existing test suite still passes: `ctest --test-dir build --output-on-failure`
- [ ] New component has at least one unit test that passes in isolation
- [ ] Thread count does not exceed any existing `task_arena` limit or `OMP_NUM_THREADS` setting

## Success Criteria

```bash
cmake --build build --parallel                  # exit 0
ctest --test-dir build --output-on-failure      # all existing tests pass
./build/tests/component_tests                   # new component tests pass
```
