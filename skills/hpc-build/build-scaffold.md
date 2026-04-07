---
name: hpc-build-scaffold
description: Use when hpc/build.md has identified new-project mode — sets C++ HPC scaffolding framing before invoking brainstorming
---

# HPC Build — Scaffold

**Core principle:** A new C++ HPC project is not done until `cmake --build` and `ctest` both exit 0.

## Starting Framing for Brainstorming

Pass this as the initial context to superpowers-extended-cc:brainstorming:

> "We're scaffolding a new C++ HPC project. Success means: CMake builds cleanly, chosen parallel library is linked and configured, and at least one smoke test passes under `ctest`. We will not write algorithm code until the build system and test harness are green."

## Clarifying Questions (ask in order, stop when you have enough)

1. **Library:** TBB, OpenMP, SYCL, CUDA C++, Taskflow, or multiple?
2. **Target:** CPU-only, NVIDIA GPU, Intel GPU, or mixed?
3. **Test framework:** GoogleTest (default) or Catch2?
4. **CMake version constraint?** (default: 3.28+)

## CMake Snippets

**TBB:**
```cmake
cmake_minimum_required(VERSION 3.28)
project(hpc_app CXX)
set(CMAKE_CXX_STANDARD 20)
find_package(TBB REQUIRED)
add_executable(main src/main.cpp)
target_link_libraries(main TBB::tbb)
enable_testing()
find_package(GTest REQUIRED)
add_executable(tests tests/smoke.cpp)
target_link_libraries(tests GTest::gtest_main TBB::tbb)
add_test(NAME smoke COMMAND tests)
```

**OpenMP:**
```cmake
cmake_minimum_required(VERSION 3.28)
project(hpc_app CXX)
set(CMAKE_CXX_STANDARD 20)
find_package(OpenMP REQUIRED)
add_executable(main src/main.cpp)
target_link_libraries(main OpenMP::OpenMP_CXX)
enable_testing()
find_package(GTest REQUIRED)
add_executable(tests tests/smoke.cpp)
target_link_libraries(tests GTest::gtest_main OpenMP::OpenMP_CXX)
add_test(NAME smoke COMMAND tests)
```

**CUDA C++:**
```cmake
cmake_minimum_required(VERSION 3.28)
project(hpc_app CUDA CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CUDA_STANDARD 20)
enable_language(CUDA)
add_executable(main src/main.cu)
set_target_properties(main PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
enable_testing()
find_package(GTest REQUIRED)
add_executable(tests tests/smoke.cpp)
target_link_libraries(tests GTest::gtest_main)
add_test(NAME smoke COMMAND tests)
```

**Taskflow (header-only):**
```cmake
cmake_minimum_required(VERSION 3.28)
project(hpc_app CXX)
set(CMAKE_CXX_STANDARD 20)
find_package(Taskflow REQUIRED)
add_executable(main src/main.cpp)
target_link_libraries(main Taskflow::Taskflow)
enable_testing()
find_package(GTest REQUIRED)
add_executable(tests tests/smoke.cpp)
target_link_libraries(tests GTest::gtest_main Taskflow::Taskflow)
add_test(NAME smoke COMMAND tests)
```

## Success Criteria

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release  # exit 0
cmake --build build --parallel                   # exit 0
ctest --test-dir build --output-on-failure       # 1/1 tests passed
```

All three must pass before moving to algorithm implementation.
