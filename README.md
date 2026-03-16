# OCaml Interpreter

A two-stage compiler and interpreter for a small functional language, implemented in OCaml. The high-level source language compiles down to a stack-based virtual machine.

---

## Architecture Overview

```
Source (.myml)
     │
     ▼
  [ Parser ]          parse_top_prog
     │
     ▼
  [ Desugarer ]       desugar / desugar_expr
     │   - multi-arg functions → nested lambdas
     │   - let-bindings → function applications
     │   - variable names → UPPER_CASE
     ▼
  [ Translator ]      translate
     │   - lexpr AST → stack_prog (list of commands)
     ▼
  [ Serializer ]      serialize
     │   - stack_prog → text
     ▼
Stack program (.my)
     │
     ▼
  [ Stack Evaluator ] eval_stack_prog
     │
     ▼
  Trace output
```

---

## Source Language (`.myml`)

A simple functional language with:

- **Literals**: integers, booleans (`true`/`false`), unit (`()`)
- **Variables**: lowercase identifiers
- **Arithmetic**: `+`, `-`, `*`, `/`
- **Comparisons**: `<`, `<=`, `>`, `>=`, `=`, `<>`
- **Logic**: `&&`, `||`, `not`
- **Functions**: `fun x -> expr` (multi-argument supported)
- **Application**: `f x y`
- **Let bindings**: `let f x y = expr in expr`
  At top level: `let f x = expr` (no `in`)
- **Conditionals**: `if b then e1 else e2`
- **Tracing**: `trace expr` — prints a value as a side effect, returns `unit`
- **Comments**: `(* ... *)`

### Example

```ocaml
let fib n =
  if n = 0 || n = 1
  then n
  else fib (n - 1) + fib (n - 2)

let _ = trace (fib 10)
```

---

## Stack Machine Language (`.my`)

The target language is a simple stack-based virtual machine. Programs are sequences of commands operating on a stack and an environment of named bindings.

### Commands

| Command | Description |
|---|---|
| `push <const>` | Push a constant (`int`, `true`, `false`, `unit`) onto the stack |
| `swap` | Swap the top two stack values |
| `trace` | Pop top value, append its string representation to the trace output |
| `add` / `sub` / `mul` / `div` | Arithmetic on the top two integers |
| `lt` | Pop two integers, push `true` if second < first |
| `if <prog> else <prog> end` | Branch on top boolean |
| `fun <NAME> begin <prog> end` | Push a closure onto the stack |
| `call` | Call the top closure with the value below it |
| `return` | Return from a function call |
| `assign <NAME>` | Pop top value and bind it to NAME in the environment |
| `lookup <NAME>` | Push the value bound to NAME onto the stack |

Variable names in the stack language are **uppercase** (e.g., `X`, `SQDIST`).

### Example

```
fun SQDIST
begin
  swap assign X swap assign Y
  lookup X lookup X mul
  lookup Y lookup Y mul
  add swap return
end
assign SQDIST

push 2 push 3 lookup SQDIST call
trace
```
Expected output: `13`

---

## File Structure

```
Ocaml_interpreter/
├── interp.ml       # Core implementation: parser combinators, AST types,
│                   #   stack machine, desugarer, translator, serializer
├── base_interp.ml  # Entry point: reads a stack program (.my), runs it,
│                   #   prints the trace
├── compile.ml      # Entry point: reads a source program (.myml),
│                   #   compiles and prints the stack program
└── examples/
    ├── example-00.myml   # K combinator (multi-arg function)
    ├── example-01.myml   # Evaluation order with trace side effects
    ├── example-02.myml   # Type error → panic
    ├── example-03.myml   # Short-circuit AND/OR
    ├── example-04.myml   # let-binding evaluation order
    ├── fib.myml          # Fibonacci (iterative + recursive)
    ├── is-perfect.myml   # Perfect number checker
    ├── taxi-cab.myml     # Taxicab number checker
    ├── simple.my         # Hand-written stack program (DUP, INCR)
    └── sqdist.my         # Hand-written stack program (squared distance)
```

---

## Usage

All entry points use `#use` to load `interp.ml`, so they are run with the OCaml toplevel (`ocaml`).

### Run the compiler (`.myml` → stack program text)

```bash
ocaml compile.ml < examples/fib.myml
```

### Run the stack interpreter (`.my` → trace output)

```bash
ocaml base_interp.ml < examples/simple.my
```

### Compile and run in one pipeline

```bash
ocaml compile.ml < examples/fib.myml | ocaml base_interp.ml
```

---

## Compilation Pipeline Details

### 1. Parsing (`parse_top_prog`)

A hand-rolled recursive descent parser built from parser combinators. A top-level program is a sequence of `let` definitions:

```
let f x y = expr
let g = expr
...
```

Operator precedence (low to high):
1. `||`
2. `&&`
3. `<` `<=` `>` `>=` `=` `<>`
4. `+` `-`
5. `*` `/`
6. `not` unary `-`
7. function application, `trace`

### 2. Desugaring (`desugar` / `desugar_expr`)

Converts the high-level AST (`expr`) into a simpler intermediate form (`lexpr`):

- Multi-argument functions `fun x y z -> e` become nested lambdas: `fun x -> fun y -> fun z -> e`
- `let f x y = v in e` becomes `(fun f -> fun x -> fun y -> e) v` — i.e., a function applied to its definition
- Top-level `let` sequence is folded into a single nested application
- All variable names are uppercased to match the stack machine convention

### 3. Translation (`translate`)

Compiles `lexpr` to a list of stack commands:

| `lexpr` | Stack code |
|---|---|
| `Num n` | `push n` |
| `Bool b` | `push b` |
| `Unit` | `push unit` |
| `Var x` | `lookup X` |
| `Trace e` | `[translate e; trace; push unit]` |
| `App(f, x)` | `[translate x; translate f; call]` |
| `Fun(p, body)` | `fun P begin swap assign P; [body]; swap return end` |
| `Ife(b, l, r)` | `[translate b; if [translate l] else [translate r] end]` |
| `Bop(And, ...)` | Short-circuit: `[b; if [c] else [push false] end]` |
| `Bop(Or, ...)`  | Short-circuit: `[b; if [push true] else [c] end]` |
| `Bop(Gt, x, y)` | Swap operands and use `lt` |
| `Bop(Lte/Gte/Eq/Neq, ...)` | Derived via `Not`, `Lt`, `Gt` combinations |

### 4. Serialization (`serialize`)

Converts the `stack_prog` command list to a human-readable string that the stack interpreter can parse.

---

## Panic

The stack machine enters a **panic** state when it encounters an error:
- Type mismatch (e.g., adding an int and a bool)
- Division by zero
- Undefined variable lookup
- Missing stack values

Once in panic, all remaining commands are discarded and `"panic"` is appended to the trace.
