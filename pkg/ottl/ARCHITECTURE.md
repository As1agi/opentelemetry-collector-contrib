# OTTL Internal Architecture & Execution Flow

This document serves as a roadmap for Go developers contributing to the OpenTelemetry Transformation Language (OTTL). It assumes a basic understanding of compiler theory (specifically Abstract Syntax Trees) and focuses on how OTTL implements its execution pipeline across the codebase

The OTTL lifecycle is divided into three distinct phases:
1. **Lexing** (Tokenizing the user's configuration string)
2. **Parsing & AST** (Building the execution tree)
3. **Execution & Evaluation** (Applying the tree to telemetry data)

---

## 1: Lexing

Before OTTL can parse a statement, the raw string (e.g., `set(attributes["foo"], "bar")`) must be broken down into recognizable tokens. OTTL relies on the `alecthomas/participle/v2/lexer` library for this phase

**Where to look:**
* `pkg/ottl/grammar.go`: Contains the struct tags and regex definitions that dictate the basic token vocabulary of OTTL
* `pkg/ottl/lexer_test.go`: Contains the validation tests for the underlying Participle lexer

## 2: Parsing & The AST

Once tokenized, the stream is converted into an Abstract Syntax Tree (AST). This phase enforces the grammar rules of the language

**Where to look:**
* `pkg/ottl/grammar.go`: Defines the Go structs that represent the hierarchy of the AST
* `pkg/ottl/parser.go`: Contains the parser initialization and the core parsing functions

**Key Data Structures:**
* `parsedStatement`: The root of the tree. It dictates that a statement generally consists of an action (an `Editor` or function) and an optional condition (`WhereClause`)
* `Editor` / `Converter`: These structs represent the actual function calls (e.g., `set`, `replace_all`) and their parsed arguments
* `Path`: Represents the dot-notation paths users write to target telemetry fields (e.g., `resource.attributes["host.name"]`)

## 2.5: Resolution & Function Wiring (Bridging AST to Execution)

Between building the AST and evaluating it against actual telemetry data, OTTL must translate parsed text into optimized, executable Go interfaces. This happens during the Collector's initialization phase, ensuring that string parsing and validation do not occur during the high-throughput execution phase

**Where to look:**
* `pkg/ottl/parser.go`: Contains the core `Parser` struct that orchestrates the resolution
* `pkg/ottl/contexts/internal/ctxcommon/parser.go`: Contains the actual implementation of the `PathExpressionParser` logic
* `pkg/ottl/factory.go`: Defines the `Factory[K]` pattern used to construct executable functions
* `pkg/ottl/ottlfuncs/`: Contains the actual OTTL function definitions (e.g., `set`, `replace_all`) and their implementations
* `pkg/ottl/contexts/internal/ctxlog/log.go`: Contains the signal-specific implementation of the `PathGetSetter` for logs
* `pkg/ottl/expression.go`: Defines core execution interfaces such as `GetSetter` and `ExprFunc[K]`

**Key Concepts:**
* **What is a "Path"?** A path is a string in dot-notation used to target a specific field within a telemetry signal (e.g., `resource.attributes["host.name"]`)
* **Path Resolution (`PathExpressionParser` -> `GetSetter`):** When the AST contains a `Path`, the `PathExpressionParser` evaluates it and returns an `ottl.GetSetter[K]`. This runtime interface provides direct memory access (`.Get()` and `.Set()`) to that specific telemetry field, allowing the execution engine to fetch or modify data without re-parsing the string for every log or span
* **Function Wiring (`Factory[K]`):** When the parser encounters a function invocation, it calls upon a `Factory[K]` (where `K` is the `TransformContext`). The factory takes the parsed `Arguments` and "wires" them by calling `CreateFunction()`, returning a native, executable Go function (`ExprFunc[K]`). This bridges the static AST to the dynamic runtime

## 3: Execution & Evaluation

There is no single "Evaluator" engine in OTTL. Instead, execution triggers recursively across the AST nodes

**Where to look:**
* `pkg/ottl/parser.go`: Contains the `Execute()` methods attached to the `Statement` structs, which orchestrate the evaluation of the `WhereClause` and the execution of the main function
* `pkg/ottl/contexts/ottllog/log.go`: Contains the implementation of the log-specific `TransformContext`
* `pkg/ottl/contexts/internal/ctxlog/log.go`: Contains the signal-specific path resolution logic

**Key Concepts:**
* **The `TransformContext`:** To evaluate a statement, the engine needs state. The `TransformContext` interface wraps the actual OpenTelemetry payload (e.g., a `plog.LogRecord` or `pmetric.Metric`) being processed at that exact microsecond
* **Executing the `GetSetter`:** When the statement executes, it does not re-evaluate the path string. Instead, the engine invokes the `.Get()` or `.Set()` methods on the `GetSetter` interface (created during Phase 2.5) to instantly read or modify the telemetry payload
