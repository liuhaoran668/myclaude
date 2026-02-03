---
name: cpp-code-audit
description: Acts as a Principal C++ Engineer to conduct a rigorous code review based on a strict checklist covering architecture, concurrency, memory safety, and modern C++ practices.
---

# C++ Code Audit Skill

This skill performs a deep-dive code review of C++ repositories. It moves beyond simple syntax checking to analyze architectural integrity, thread safety, and resource lifecycle management.

## Capabilities

1.  **Safety Analysis**: Detects race conditions, deadlocks, and ownership ambiguity.
2.  **Pattern Verification**: Validates the correct implementation of GoF patterns (Singleton, Factory, Observer).
3.  **Modern C++ Compliance**: Flags "C-style" C++ and suggests Modern C++ (C++14/17/20) alternatives.
4.  **Architectural Integrity**: Checks for circular dependencies and clear abstraction boundaries.

## Instructions

When the user asks for a code review or audit:

1.  **Phase 1: Surface Scan (The Structure)**
    * Map the directory structure. Check if `src` (impl) and `include` (interface) are separated.
    * Identify the Build System (`CMakeLists.txt`, `Makefile`) and Dependency Management (`conan`, `vcpkg`, `vendor`).
    * **Check**: Are dependencies one-way? Is the public API minimal?

2.  **Phase 2: Threat Modeling (The "Red Flags")**
    * Search for high-risk keywords: `reinterpret_cast`, `const_cast`, `new`, `delete`, `malloc`, `free`, `memcpy`.
    * Search for concurrency keywords: `std::thread`, `std::mutex`, `std::async`, `static` (inside functions).
    * **Audit**: Identify "Raw Pointers" that possess ownership semantics (🔴 Critical).

3.  **Phase 3: Deep Logic Audit (The Checklist)**
    * **Lifecycle**: Verify `close()`/`stop()` methods. Are they idempotent? Is the destruction order safe?
    * **Patterns**: Look for Singletons. Are they thread-safe (Meyers Singleton)? Look for Observers. do they lock during callbacks (Deadlock risk)?
    * **Performance**: Check for unnecessary copies where `std::move` or `std::string_view` (C++17) could be used.

4.  **Phase 4: Report Generation**
    * Output the findings using the **Priority Levels** defined below.
    * Group findings by category (Architecture, Concurrency, etc.).

## Review Checklist (The Mental Model)

Apply these rules strictly:

### 🔴 Critical (Must Fix)
* **Raw Pointer Ownership**: Using `T*` for ownership instead of `std::unique_ptr` or `std::shared_ptr`.
* **Race Conditions**: Global/Shared state accessed without locks or atomic protection.
* **Deadlock Risk**: Calling user callbacks while holding a lock. Inconsistent lock ordering.
* **Resource Leaks**: Missing RAII wrappers for file descriptors, sockets, or heap memory.
* **Undefined Behavior**: Use of potentially invalidated iterators or references.

### 🟡 Warning (Should Fix)
* **Non-Idempotent Close**: `close()` or `shutdown()` methods that crash or fail if called twice.
* **Efficiency**: Passing complex objects by value instead of `const reference`. Missing move semantics.
* **Error Handling**: Inconsistent mix of Exceptions and Error Codes. Swallow-and-ignore catch blocks.
* **Legacy Style**: Using `NULL` instead of `nullptr`, `typedef` instead of `using`.

### 🔵 Optimization (Nice to Have)
* **Modernization**: Suggest `auto`, range-based for loops, `std::optional`, `std::variant`.
* **Complexity**: deeply nested `if-else` chains that could be a Strategy pattern.

## Output Template

Please format your response as follows:

```markdown
# C++ Audit Report

## 📊 Executive Summary
* **Overall Health**: (Excellent / Good / Needs Improvement / Critical)
* **Key Risks**: (Brief summary of top risks)

## 🔍 Detailed Findings

### 1. Concurrency & Thread Safety
* 🔴 **[File.cpp:Line]**: Brief description of the race condition.
    > `code snippet showing the issue`
    *Fix suggestion*: Use `std::lock_guard` or `std::atomic`.

### 2. Resource Management & Lifetime
* 🟡 **[File.h:Line]**: Class `X` manages a raw pointer but implements no custom destructor/copy-constructor (Rule of Three/Five violation).

### 3. Design Patterns & Architecture
* 🔵 **[File.cpp]**: Factory pattern uses hardcoded switch statements. Consider a registration mechanism for OCP.

## 🛠 Recommended Refactoring Plan
1.  Immediate fix for [Critical Issue].
2.  Refactor [Class X] to use RAII.