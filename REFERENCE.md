# Helix Reference

Everything the `helix` compiler (1.0.0-alpha.0) supports: commands, options, targets, `helix.jsonc`, the language, tests, and C interop.
Everything here is taken from the compiler source. Paths point to where each feature is implemented.

---

## 1. Commands

`src/modules/driver/driver.module.hx:713` · usage text in `src/modules/driver/constants/driver.constant.hx`

| Command | What it does |
|---|---|
| `helix build [file.hx]` | Build a program. Without a file, runs the project's `build.hx`. |
| `helix run <file.hx> [-- args]` | Build a program and run it. Everything after `--` goes to the program. |
| `helix test [paths...]` | Build and run the specs (`*.spec.hx`) under the given files and folders. |
| `helix check <file.hx>` | Report errors without building. |
| `helix ir <file.hx>` | Print the optimized LLVM IR. |
| `helix ast <file.hx>` | Print the syntax tree. |
| `helix tokens <file.hx>` | Print the token stream. |
| `helix cc [--target <t>] ...` | Run clang, set up to compile C for a target. |
| `helix clang ...` / `helix ar ...` / `helix dlltool ...` | Run the clang, llvm-ar or llvm-dlltool that ship inside the compiler. |
| `helix translate-c <header.h>` | Write the Helix declarations of a C header. |
| `helix translate-svd <chip.svd>` | Write a Helix module for a microcontroller's registers. |
| `helix chips [filter]` | List the microcontrollers that `--chip` knows. |
| `helix version` / `--version` | Print the version. |
| `helix help` / `--help` / `-h` | Print the usage text. |

`init`, `add`, `remove`, `fetch`, `package` and `run-script` moved to **hpm** (`hpm init`, `hpm add`, `hpm remove`, `hpm fetch`, `hpm pack`, `hpm run`). Running one of them through `helix` points you there.

### Options of `build`, `run`, `check`, `ir`, `ast`

`src/modules/driver/components/options.component.hx`

| Option | Meaning |
|---|---|
| `-o <file>` | Output file |
| `--mode <mode>` | `debug` (default), `release-safe`, `release-fast`, `release-small` |
| `--target <target>` | Cross-compile (see [Targets](#2-targets)) |
| `--cpu <cpu>` | Tune for a CPU, e.g. `cortex-m4` |
| `--test` | Build the program's tests instead of its `main` |
| `--strip` | Leave out symbols and debug info |
| `--emit obj` | Write the object file instead of an executable (`obj` is the only value) |
| `-l<name>` | Link a library (passed to the linker as `-l<name>`) |
| `--link <arg>` | Pass one argument to the linker unchanged: a `.a`, `.o`, `.dylib`/`.so`, `-L<dir>`, `-rpath`, ... |
| `--pkg <name=path>` | Map an import name to a file or folder |
| `--pkg-in <scope> <name=path>` | The same mapping, seen only by the package in folder `<scope>` |
| `--linker-script <file>` | Linker script for a program without an OS |
| `--chip <name>` | Build for a known microcontroller, e.g. `stm32f411ce` |
| `--flash <origin:size>` | Flash region of a Cortex-M chip, e.g. `0x08000000:512K` |
| `--ram <origin:size>` | RAM region of a Cortex-M chip |
| `-- args...` | Arguments for the program (`run`) |

### Options of `test`

| Option | Meaning |
|---|---|
| `-s <glob>` | Only spec files whose paths match |
| `-f <regex>` | Only tests whose describe and test names match |
| `-b` | Stop after the first failure |
| `-v` | List every test, not only the failures |
| `-t <ms>` | Timeout of each test (default 5000) |
| `--reporter <name>` | `text`, `json`, or `teamcity` |

### Options of `translate-c`

| Option | Meaning |
|---|---|
| `-o <file>` | Output file (stdout without it) |
| `--target <target>` | Translate for another machine |
| `--all` | Translate everything clang saw, including system headers |
| `--dir` | Translate every header under the header's folder (default: the header plus the headers it includes from its own folder) |
| `-I`, `-D`, `-isystem`, `-include`, any other `-…` | Passed to clang |

```bash
helix translate-c zlib.h -I/opt/zlib/include -o src/zlib.hx
helix translate-svd STM32F411.svd -o src/chip.hx
helix chips f411
```

---

## 2. Targets

`src/modules/driver/components/target.component.hx:100`

The format is `arch-os[-abi]`. Without `--target`, or with `native`, the compiler builds for this machine.

| Arch | OS | Resulting triple |
|---|---|---|
| `x86_64`, `aarch64` (`arm64`), `arm64e` | `macos` | `<arch>-apple-macosx11.0` (`arm64e` is macOS only) |
| `x86_64`, `aarch64`, `riscv64` | `linux[-abi]` | `<arch>-unknown-linux-musl` (default ABI musl; `-gnu` also works) |
| `x86_64`, `aarch64` | `windows[-abi]` | `<arch>-w64-windows-gnu` (MinGW) |
| `wasm32` | `wasi` | `wasm32-unknown-wasip1` |
| `wasm32`, `x86_64`, `aarch64`, `riscv64` | `freestanding` | `<arch>-unknown-none-elf` |
| `thumbv6m`, `thumbv7m`, `thumbv7em`, `thumbv8m.main` | `freestanding[-abi]` | `<arch>-none-eabi` (Cortex-M) |

Examples: `aarch64-macos`, `x86_64-linux`, `x86_64-windows`, `wasm32-wasi`, `thumbv7em-freestanding`.

---

## 3. `helix.jsonc`

`src/modules/driver/components/manifest.component.hx` · allowed fields in `src/modules/driver/constants/manifest.constant.hx`

The file may contain comments and trailing commas. **An unknown field is an error.**

Allowed top-level fields: `$schema`, `name`, `version`, `description`, `license`, `author`, `repository`, `homepage`, `keywords`, `helix`, `private`, `exports`, `c`, `module`, `bin`, `scripts`, `files`, `dependencies`, `devDependencies`, `peerDependencies`, `peerDependenciesMeta`, `registries`, `workspaces`, `aliases`, `chip`.

```jsonc
{
    "name": "app",
    "version": "1.0.0",
    "bin": { "app": "src/main.hx" },              // executables: name -> root file
    "scripts": { "dev": "helix run src/main.hx" },
    "dependencies": { "json": "^1.0.0" },          // installed by hpm into helix-modules/
    "aliases": { "@models": "src/models" },        // import "@models/user"
    "exports": { ".": "src/index.hx" },            // what a package exposes

    // C code built into THIS program
    "c": {
        "sources": ["c/glue.c"],
        "flags": ["-DFOO=1"],
        "includeDirs": ["vendor/include"],
        "systemLibraries": ["z", "sqlite3"]        // -> -lz -lsqlite3
    },

    // for a package: its root and the C it brings to whoever depends on it
    "module": {
        "root": "src/index.hx",
        "cSources": ["c/impl.c"], "cFlags": [], "includeDirs": ["c"], "systemLibraries": []
    },

    // microcontroller: a name from `helix chips`, or a full description
    "chip": "stm32f411ce"
    // "chip": { "cpu": "cortex-m4", "fpu": true,
    //           "flash": { "origin": "0x08000000", "size": "512K" },
    //           "ram":   { "origin": "0x20000000", "size": "128K" } }
}
```

`chip.cpu` must be one of: `cortex-m0`, `cortex-m0plus`, `cortex-m3`, `cortex-m4`, `cortex-m7`, `cortex-m33`.

---

## 4. `build.hx` (project builds)

`lib/std/build.hx`

`helix build` with no file runs the project's `build.hx`. Running `helix build <step>` runs a named step.

```hx
import { Build, CSources, ExecutableOptions } from "std/build";

export function build(b: *Build): void | Error {
    const exe = try b.bin("app");                     // the "bin" entry of helix.jsonc
    try exe.addIncludeDir("vendor/include");
    try exe.linkSystemLibrary("sqlite3");
    try exe.addCSources(CSources { files: ["c/glue.c"], flags: ["-O2"] });
    try b.addRunStep(exe);

    const tool = try b.addExecutable(ExecutableOptions { name: "tool", root: "tools/tool.hx", target: "x86_64-linux" });
    try b.install(tool);

    const fast = b.optionBool("fast", "Build fast");   // also optionString / optionInt
    const step = try b.step("gen", "Generate code");
    try step.dependOn(try b.addSystemCommand(["./gen.sh"]));
}
```

Other `Build` methods:
- `addTestStep(TestOptions)`
- `addInstall(artifact, dir)`, `addInstallFile(path, dir)`, `addInstallFolder(path, dir)`
- `addOptions(name)`, which gives an `Options` with `addBool`, `addInt`, `addString` and `add` that `artifact.addOptions(options)` makes importable

`Mode` is the same set of modes as `--mode`.

---

## 5. Language

### 5.1 Lexical

**Keywords:**
`as break case catch comptime const continue default defer dyn else enum errdefer error export extends extern false for from function if import interface let null of readonly return struct switch throw true try type undefined union unreachable unsafe void while`

These are contextual, so they also work as names: `as dyn error extends from of readonly type`.

**Comments:**
- `// line`
- `/* block */`; block comments nest.
- `/** doc */` is kept as a doc comment.

**Literals:**

| Kind | Examples |
|---|---|
| Integer | `42`, `0xFF`, `0o755`, `0b1010`, `1_000_000` (a leading `0` like `012` is an error) |
| Float | `3.14`, `1e9`, `2.5e-3` |
| Char (an integer) | `'a'`, `'\n'`, `'\u{E9}'` |
| String (`string` = `readonly u8[]`, one line) | `"hello\n"` |
| Template (multi-line, only for `Format` params like `println`, `fmt.alloc`) | `` `id=${id}` `` |
| Others | `true` `false` `null` `undefined` `unreachable` |
| Array | `[1, 2, 3]` |
| Struct | `Point { x: 1, y }`, `{ x: 0 }` |
| Enum member | `.Admin` |

**Escapes:** `\n \r \t \0 \\ \" \' \` \$ \xNN \u{N}`.

**Operators:**

| Group | Tokens |
|---|---|
| Arithmetic (overflow-checked) | `+ - * / %` |
| Wrapping | `+% -% *%` |
| Saturating | `+| -| *|` |
| Comparison (don't chain) | `== != < <= > >=` |
| Logical | `&& || !` |
| Bitwise | `& | ^ ~ << >>` |
| Optionals | `?T` `x!` `a ?? b` `a?.b` |
| Pointers | `&x` `*p` |
| Ranges | `a..b` `a..=b` |
| Cast | `x as T` |
| Ternary | `c ? a : b` |
| Statement-only | `i++` `i--` |
| Assignment | `= += -= *= /= %= +%= -%= *%= +|= -|= *|= &= |= ^= <<= >>=` |

- **Precedence, loosest first:** `?:` → `catch` → `??` → `||` → `&&` → comparisons → `|` → `^` → `&` → shifts → `+ -` → `* / %` → `as` → prefix → postfix.
- **Unlike C:** `flags & MASK == 0` means `(flags & MASK) == 0`.
- **`??=`:** it parses but isn't supported yet.
- **`===`:** an error.

### 5.2 Types

| Type | Syntax |
|---|---|
| Primitives | `void bool noreturn usize isize f32 f64`, plus `u1`…`u65535` and `i1`…`i65535` |
| Built-in names | `string` (`readonly u8[]`), `Error` (any error), `Format` (a template) |
| Pointer | `*T` |
| Read-only | `readonly *T`, `readonly T[]` |
| Optional | `?T` |
| Many-item pointer (C pointer) | `T[*]`; C `void*` is `?u8[*]` |
| Slice | `T[]` (`.len`, `.ptr`) |
| Array | `T[N]`, `f32[3][4]`, `u8[comptime n]` |
| Error union | `T \| MyError`, `void \| Error` |
| Function | `(a: i32, b: i32) => bool` |
| C function | `extern "C" (a: i32) => void` |
| Interface value | `dyn Writer`, `dyn Iterator<i32>` |
| Generic | `List<u8>`, `Pair<A, B>` |
| Anonymous struct | `struct { code: u32, shift: bool }` |

`?u8[]` is an optional slice. Prefixes bind looser than suffixes.

### 5.3 Declarations

```hx
// variables
const MAX: u32 = 64;
let count = 0;
let buf: u8[64] = undefined;

// functions
function add(a: i32, b: i32): i32 { return a + b; }
function repeat(comptime n: usize, s: string): void { ... }
export function main(): void | Error { ... }      // main(): void|u8|i32[|Error], or main(init: Init)

// generics
function max<T extends Numeric>(a: T, b: T): T { return a > b ? a : b; }
max<i64>(a, b);   // explicit type args (usually inferred)
struct Stack<T> { items: T[4], len: usize }   // also union Result<T, E> { ... }, interface Iterator<T> { ... }
const s = Stack<i32> { items: [0, 0, 0, 0], len: 0 };

// structs: field defaults, consts, methods (no `function` keyword)
struct Point {
    x: i32 = 0,
    y: i32 = 0,
    origin(): Point { return Point {}; }                          // static
    move(this: *Point, dx: i32): void { this.x += dx; }           // this: T | *T | readonly *T
}

// enums
enum Color: u8 { Red = 1, Green = 2, Blue = 4 }
enum Role { Admin, Member, Guest, const DEFAULT = Role.Member; }

// tagged unions
union Event { click: Point, key: struct { code: u32, shift: bool }, quit }
const e = Event.click(Point { x: 3, y: 4 });

// error sets
error ParseError { Empty, InvalidDigit }

// type aliases
type Count = u32;
type Predicate = (x: i32) => bool;

// interfaces (structural: no impl blocks)
interface Shape { area(this: readonly *Self): f64; }
function show(s: dyn Shape): void { ... }                         // dynamic dispatch
function total<S extends Shape>(items: readonly S[]): f64 { ... } // static dispatch
```

- **Generic constraints:** `Numeric`, `Integer`, `Float`, or an interface, e.g. `<I extends Iterator<T>, T extends Numeric>`.
- **Locals and parameters:** unused ones are errors (prefix the name with `_`), as is a `let` that is never reassigned.

**Imports and exports:**

```hx
import { Point, twice as double } from "./geometry";
import * as fmt from "std/fmt";
import { area } from "@shapes/circle";          // alias from helix.jsonc
import "./exports-to-c";                        // load only (for its extern "C" exports)
export { point as makePoint } from "./geometry";
export * from "./heap";
```

- A folder import resolves to its `index.hx`.
- There are no default imports.

**std modules** (`lib/std`): `build`, `c`, `fmt`, `fs`, `hash`, `heap`, `io`, `json`, `list`, `mmio`, `path`, `process`, `sha256`, `target`, `testing`, `text`.

**Compile time:**

```hx
const TABLE = comptime buildCrcTable();
comptime { if (@sizeOf(Header) != 16) @compileError("Header must be 16 bytes"); }
```

### 5.4 Statements

```hx
if (x < 0) { ... } else if (x == 0) { ... } else { ... }
if (const v = find(items, 5)) { ... }            // optional capture
while (const n = it.next()) { ... }
while (i < 5) { i++; }
for (let j: i32 = 10; j > 0; j -= 2) { ... }
for (const x of items) { ... }
for (const v, i of values) { ... }               // value, index
for (const k of 0..4) { ... }   for (const k of 1..=n) { ... }
for (const x of iter) { ... }                    // anything with next(this: *Self): ?T
outer: for (...) { continue outer; break outer; }
defer free(a, s);   defer { ... }
errdefer cleanup();   errdefer (e) log(e);
unsafe { ... }
comptime { ... }
return x;   throw ParseError.Empty;
```

### 5.5 Expressions

**switch** works as a statement or an expression. Cases use `:` (not `=>`), and switches over enums, unions and `bool` must be exhaustive.

```hx
switch (code) {
    case 200: ...
    case 2, 3: ...
    case 400..=499: ...
    case "GET": ...
    case n if n >= 500: ...
    default: unreachable;
}
switch (event) {
    case Event.click(p): ...
    case .resize(*r): r.w = 0;               // bind by pointer
    case Event.key({ code, shift }): ...
    case .quit: ...
}
const next = switch (role) { case .Admin: .Member; case .Member: .Guest; case .Guest: .Admin; };
```

**Errors:**

```hx
const n = try parse(text);                   // propagate
const port = parsePort(s) catch 8080;        // fallback
parse(s) catch (e) { ... };
parse(s) catch (e) switch (e) { ... };
const v = f() catch return -2;
void try parseDigit(text);                   // discard the result
@errorName(e);
```

**Optionals and leaving from an expression:**

```hx
const name = user?.boss?.name ?? "none";
const v = maybe!;                             // unwrap
const x = find(i) ?? return -1;               // also ?? continue / ?? break / ?? throw E.X
```

**Other expressions:**

```hx
x as i64        Color.Red as u8        &it as dyn Iterator<i32>
a[i]   a[i..j]   a[i..=j]   a[..j]   a[i..]   sum(&array)
const sq = (x: i32): i32 => x * x;            // arrow functions: no capture of locals
const f = Point.sum;                          // methods as values
void doSomething();                           // an unused non-void result is an error
```

**Builtins** (the complete list):

| Builtin | Meaning |
|---|---|
| `@sizeOf(T)`, `@alignOf(T)` | Size and alignment of a type |
| `@truncate(T, x)` | Narrow an integer |
| `@panic(msg)` | Stop at run time |
| `@compileError(msg)` | Compile-time error (only on reachable paths) |
| `@compileLog(...)` | Print at compile time and fail the build |
| `@ptrCast(T, p)` | Reinterpret a pointer (needs `unsafe`) |
| `@ptrFromInt(T, addr)` | Pointer from an address (needs `unsafe`) |
| `@intFromPtr(p)` | Address as `usize` |
| `@enumFromInt(E, n)` | Integer to enum |
| `@volatileLoad(p)`, `@volatileStore(p, v)` | Memory-mapped I/O |
| `@errorName(e)` | Name of an error |

**Built-in functions:** `print`, `println`, `eprint`, `eprintln`.

**`unsafe { }` is required for:**
- `@ptrCast`
- `@ptrFromInt`
- indexing a `T[*]`
- slicing a `T[*]` (which also needs an end: `p[0..len]`)

### 5.6 Safety rules

- **Overflow:** `+ - *` overflow panics at run time and is an error during constant folding. Use `+%` to wrap and `+|` to saturate.
- **No implicit conversions:** mixing `i32` and `u32` is an error; use `as`.
- **Run-time panics:** out-of-bounds index, division by zero, bad shift, bad cast, null unwrap, double free.
- **Compile errors:**
  - returning a pointer to a local
  - writing through `readonly` or `string`
  - using a `?T` field without `?.` or `!`
  - using a failable value before `try`/`catch`
- **Narrowing:** `if (x != null)` and `if (x == null) return;` narrow optionals, and unions narrow too.

### 5.7 Not in the language

`impl`, `trait`, `match`, `class`, `fn`, `pub`, `mut`, attributes, `packed`/`align`, inline assembly, closures that capture locals, operator overloading, nested `function`s, default imports.

These parse but aren't supported yet: `import()`, `??=`, generic enums, generic type aliases, and `comptime` parameters on arrow functions.

---

## 6. Tests

`src/modules/parser/components/tests.component.hx` · run with `helix test`

```hx
describe("parser") {
    let count: i32 = 0;
    beforeAll { ... }   afterAll { ... }   beforeEach { count += 1; }   afterEach { ... }

    test("adds") { expect(1 + 1).toBe(2); }                   // `it(...)` also works
    test("slow", { timeout: 100 }) { ... }
    test.skip("later") { ... }
    test.only("focus") { ... }
    test.todo("write me");
    test.failing("known bug") { ... }
    test.panics("bounds", "index out of bounds") { ... }
}
describe.skip("x") { ... }   describe.only("y") { ... }
```

**Matchers** (all can be negated with `.not`, e.g. `expect(x).not.toBe(1)`):

| Group | Matchers |
|---|---|
| Values | `toBe`, `toEqual`, `toBeTrue`, `toBeFalse`, `toBeNull`, `toBeDefined`, `toBeCloseTo` |
| Comparisons | `toBeLessThan`, `toBeLessThanOrEqual`, `toBeGreaterThan`, `toBeGreaterThanOrEqual` |
| Collections and text | `toContain`, `toContainEqual`, `toInclude`, `toHaveLength`, `toStartWith`, `toEndWith`, `toMatch` |
| Errors | `toThrow`, `toSucceed` |
| Mocks | `toHaveBeenCalled`, `toHaveBeenCalledTimes`, `toHaveBeenCalledWith`, `toHaveBeenLastCalledWith`, `toHaveBeenNthCalledWith` |

**Assertions and mocks:**
- `require(...)` is an assertion that stops the test when it fails.
- `mock(fn[, impl])` replaces a function **for calls from other modules only**; calls inside the function's own module still run the real code. A bare `mock(fn)` returns zero.
- `spyOn(fn)` records **every** call and keeps the real behavior.
- `mock(...)` returns nothing; call the methods on the function itself: `fn.mockReturnValue(v)`, `fn.mockImplementation(f)`, `fn.mockClear()`, `fn.mockReset()`, `fn.mockRestore()`.
- Each test starts with mocks reset.

```hx
import { readPort } from "./config";
import { serve } from "./server";              // serve() calls readPort()

test("mock") {
    readPort.mockReturnValue(80);
    expect(serve("x")).toBe(160);
    expect(readPort).toHaveBeenLastCalledWith("x");
}
```

---

## 7. C interop

### Calling C

**Option 1: translate a header**

```bash
helix translate-c mylib.h -I/opt/mylib/include -o src/mylib.hx
```

```hx
import * as my from "./mylib";
my.mylib_init();
```

The generated file is regenerated, never edited. Put your Helix wrapper in another file.

**Option 2: declare by hand**

```hx
extern "C" function printf(format: readonly u8[*], ...): i32;   // only C functions may be variadic
extern "C" let optind: i32;                                       // C global
export extern "C" struct Pair { a: i32, b: i32 }                  // C layout
```

### Exporting to C

```hx
export extern "C" function helix_add(a: i32, b: i32): i32 { return a + b; }
```

### Compiling C with the program

Either list the files in `helix.jsonc`:

```jsonc
"c": { "sources": ["c/glue.c"], "flags": [], "includeDirs": ["c"], "systemLibraries": ["m"] }
```

…or add them in `build.hx`:

```hx
try exe.addCSources(...);
try exe.addIncludeDir(...);
try exe.linkSystemLibrary(...);
```

### Linking external libraries

```bash
helix build main.hx -lz                                         # from the default search path
helix build main.hx --link -L/opt/mylib/lib -lmylib             # library in your own folder
helix build main.hx --link -rpath --link /opt/mylib/lib         # let a dylib/.so be found at run time
helix build main.hx --link /opt/mylib/lib/libmylib.a            # a file directly (.a, .o, .dylib, .so)
```

- Each `--link` carries exactly one linker argument.
- `helix.jsonc` has no field for library folders, so `-L` and `-rpath` go on the command line.
- **Cross builds:** every `-lX` must exist as `libX.a` in the target's C library folder. Otherwise, build the library from source as a package with `module.cSources`.
- **Shared libraries** work only for builds for this machine.
- **Compiling C yourself:** `helix cc --target <t> -c x.c -o x.o` and `helix ar rcs libx.a x.o` build C objects and archives for any target.

---

## 8. Embedded (Cortex-M)

```bash
helix chips stm32f4
helix build main.hx --chip stm32f411ce
helix build main.hx --target thumbv7em-freestanding --cpu cortex-m4 \
    --flash 0x08000000:512K --ram 0x20000000:128K
helix translate-svd STM32F411.svd -o src/chip.hx       # typed register module
```

- Registers are reached through `std/mmio`, which uses `@volatileLoad` and `@volatileStore`.
- A custom memory layout can be given with `--linker-script <file>`.
