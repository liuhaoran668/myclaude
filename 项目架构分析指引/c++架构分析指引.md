---
name: cpp-architecture-analysis
description: Analyzes a C++ codebase to generate a comprehensive architectural design document, focusing on build systems, class hierarchy, memory management models, and functional data flow.
---

# C++ Architecture Analysis Skill

This skill allows Claude to act as a Senior C++ Software Architect. It systematically explores a C++ repository to reverse-engineer its design, identifying key patterns, build configurations, and functional implementations.

## Capabilities

1.  **Build System Profiling**: Analyzes `CMakeLists.txt`, `Makefile`, `conanfile.txt`, or `vcpkg.json` to understand dependencies and targets.
2.  **Interface Analysis**: Prioritizes scanning `include/` directories and `.h/.hpp` files to establish the public API and class hierarchy before diving into implementation.
3.  **Implementation Review**: Examines `src/` or `.cpp` files to understand *how* interfaces are implemented, focusing on RAII usage, concurrency models, and algorithm choices.
4.  **Design Pattern Detection**: Identifies GoF patterns (Factory, Observer, Singleton, etc.) used in the codebase.

## Instructions

When the user asks for an architecture analysis or design document:

1.  **Phase 1: Reconnaissance (The "Superpowers" Approach)**
    * First, read the root directory to identify the build system.
    * Read `CMakeLists.txt` (or equivalent) to map out executables, libraries, and external dependencies.
    * Locate the "entry point" (`main.cpp` or library interface).

2.  **Phase 2: Structural Mapping**
    * Scan the `include/` directory. Create a mental map of the namespaces and primary classes.
    * Identify "God Classes" or central hubs of data flow.

3.  **Phase 3: Deep Dive**
    * For key components identified in Phase 2, read the corresponding `.cpp` implementation.
    * Analyze memory management: Is it using Smart Pointers (`std::shared_ptr`, `std::unique_ptr`) or raw pointers?
    * Analyze Concurrency: usage of `std::thread`, `std::mutex`, or `std::async`.

4.  **Phase 4: Documentation Generation**
    * Generate a Markdown report following the structure defined in the **Output Template** below.

## Output Template

Your output must follow this structure:

### 1. Project Overview
* **Standard**: (e.g., C++14, C++17, C++20)
* **Build System**: (e.g., CMake 3.14+)
* **Dependencies**: (List external libs found in CMake/Conan)

### 2. High-Level Architecture
* **Diagram Description**: (Describe how modules interact using Mermaid diagram syntax if helpful)
* **Core Modules**:
    * **Module A**: Responsibility and key classes.
    * **Module B**: Responsibility and key classes.

### 3. Key Design Decisions
* **Memory Model**: (e.g., "Uses RAII extensively with `std::unique_ptr` for ownership.")
* **Concurrency**: (e.g., "Uses a thread pool pattern for processing requests.")
* **Error Handling**: (e.g., "Exceptions vs. Return Codes/`std::optional`")

### 4. Functional Implementation Details
* **[Functionality Name]**:
    * *Interface*: `Include/MyClass.h`
    * *Implementation Strategy*: Description of the algorithm or logic in `.cpp`.
    * *Critical Path*: Describe the data flow.

## Guidelines

* **Header First**: Always prioritize reading header files (`.h`, `.hpp`) over source files to save context tokens and get a cleaner view of the abstraction.
* **Modern C++ Check**: Explicitly note if legacy C++ (pre-C++11) patterns are mixed with Modern C++ (C++11+).
* **Dependency Graph**: If `CMakeLists.txt` defines multiple libraries, explicitly map out which library depends on which.
* **Safety Audit**: Flag usage of `new`/`delete` (manual memory management) as potential risks unless wrapped in RAII classes.

## Examples

**User:** "Analyze the architecture of this project."
**Claude:** "I will start by checking the `CMakeLists.txt` and the `include` directory to understand the project structure." (Proceeds to execute `ls` and `read_file` commands, then generates the report using the template above.)
