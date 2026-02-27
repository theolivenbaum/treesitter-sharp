# AGENTS.md

## Overview

This repository is a port of the `gotreesitter` library (pure Go Tree-sitter runtime) to modern .NET 10 C#. The goal is to provide a high-performance, pure C# implementation of Tree-sitter that does not rely on C/C++ bindings (P/Invoke) and is WASM-ready.

## Source Code Explanation

The original Go codebase (located in `reference/`) is structured as follows:

### Core Components

*   **`parser.go`**: Implements the `Parser` struct and the core parsing logic. It uses a Generalized LR (GLR) algorithm to handle ambiguous grammars. Key method is `parseInternal` which manages the parse stack(s).
*   **`tree.go`**: Defines the `Tree` and `Node` structures. `Node` represents a syntax tree node with properties like `Symbol`, `StartByte`, `EndByte`, `Children`, etc. `Tree` holds the root node and references to source code and arenas.
*   **`language.go`**: Defines the `Language` struct, which contains the static data for a grammar (parse tables, lexer states, symbol names). This mirrors the `TSLanguage` struct in the C implementation.
*   **`lexer.go`**: Implements a table-driven DFA lexer (`Lexer`). It consumes source code and produces `Token`s based on `LexState` transitions defined in the `Language`.
*   **`arena.go`**: Implements a slab/arena allocator for `Node`s to minimize Garbage Collection (GC) pressure. This is crucial for performance.
*   **`query.go`**: Implements the Tree-sitter query language (S-expressions) for pattern matching against the syntax tree.
*   **`highlight.go`**: Implements syntax highlighting using queries.
*   **`incremental.go`**: Handles incremental parsing logic, reusing subtrees from a previous parse.

### Helper Components

*   **`grammars/`**: Contains generated Go code for various languages.
*   **`external_vm.go`**: A bytecode interpreter for running external scanners (logic that can't be expressed in the DFA).

## Goals

1.  **Pure C# Implementation**: No native dependencies (no `tree-sitter.dll` or `libtree-sitter.so`).
2.  **Modern .NET 10**: Utilize the latest features of C# and .NET 10.
3.  **High Performance**: Match or exceed the performance of the Go implementation (and ideally the C implementation). use `Span<T>`, `Memory<T>`, `ref struct`, and object pooling where appropriate.
4.  **API Compatibility**: The API should be idiomatic C# but familiar to users of other Tree-sitter bindings.

## Porting Guide

### General Strategy

*   **Structs vs Classes**:
    *   Small, immutable data structures (like `Point`, `Range`, `Symbol`) should be `readonly struct` or `record struct`.
    *   Larger structures or those requiring identity (like `Parser`, `Language`) should be `class`.
    *   `Node` in Go is a struct with pointers. in C# it might be a `struct` wrapper around an internal handle or index into the Arena, or a class if we rely on the GC. Given the Arena design in Go, we should likely replicate the Arena pattern in C# to avoid massive GC allocations for parse nodes.

*   **Slices and Arrays**:
    *   Go `[]T` -> C# `T[]`, `List<T>`, `Span<T>`, or `Memory<T>`.
    *   For the source code, use `ReadOnlyMemory<byte>` or `ReadOnlySpan<byte>` to avoid copying.

*   **Arena Allocation**:
    *   Port `arena.go` using an array-based pool or `System.Buffers.ArrayPool<T>`.

### Step-by-Step Porting Plan

1.  **Core Types**:
    *   Implement `Point`, `Range`, `InputEdit`.
    *   Implement `Symbol`, `StateID`, `FieldID` (likely `ushort` aliases or strong types).

2.  **Language Definition (`language.go`)**:
    *   Implement the `Language` class and related structures (`LexState`, `ParseAction`, etc.).
    *   This is the foundation for loading grammars.

3.  **Lexer (`lexer.go`)**:
    *   Implement the `Lexer` class.
    *   Ensure it correctly handles UTF-8 (Go strings are UTF-8; C# strings are UTF-16, but source is likely UTF-8 bytes). Using `ReadOnlySpan<byte>` for source is recommended.

4.  **Tree and Node (`tree.go`)**:
    *   Implement `Tree` and `Node`.
    *   Implement the Arena allocator (`arena.go`).

5.  **Parser (`parser.go`)**:
    *   Implement `Parser` class.
    *   Port the GLR parsing logic (`parseInternal`).
    *   Implement incremental parsing support.

6.  **Query Engine (`query.go`)**:
    *   Port the query compiler and executor.

7.  **Tests**:
    *   Port the tests from `*_test.go` files to xUnit tests.

### Conventions

*   Use `PascalCase` for public members.
*   Use `camelCase` for private members and parameters.
*   Use `async/await` if IO is involved (though parsing is mostly CPU-bound).
*   Document public APIs using XML documentation comments (`///`).

## Verification

*   Run `dotnet test` often.
*   Verify against the reference Go implementation behavior.
