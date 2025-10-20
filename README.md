# Table of Contents
- [1. Motivation](#1-motivation)
- [2. Overview](#2-overview)
- [3. Frontend Integration (Slang -> Moore)](#3-frontend-integration)
- [4. Object Layout](#4-object-layout)
- [5. Core Moore Operators](#5-core-moore-operators)
- [6. SV-to-Moore Lowering Rules](#6-sv-to-moore-lowering-rules)
- [7. Moore Lowering Example Sketch](#7-moore-lowering-example-sketch)
- [8. Moore -> LLVM Lowering](#8-moore---llvm-lowering)
- [9. Naming and Visibility](#9-naming-and-visibility)
- [10. Object Lifetime and Memory Management](#10-object-lifetime-and-memory-management)
- [11. Static Members and Initialization](#11-static-members-and-initialization)
- [12. Class Tasks and Timing Semantics](#12-class-tasks-and-timing-semantics)
- [13. Dynamic Type Operations](#13-dynamic-type-operations)
- [14. VTable Construction and Inheritance](#14-vtable-construction-and-inheritance)
  - [14.1 VTable Layout and RTTI Integration — Full Example](#141-vtable-layout-and-rtti-integration--full-example)
- [15. Abstract Classes and Pure Virtual Methods](#15-abstract-classes-and-pure-virtual-methods)
- [16. Type Parameters](#16-type-parameters)
- [17. Future Work / Out of Scope](#17-future-work--out-of-scope)
- [18. Summary](#18-summary)
- [19. Additional Notes & Open Items](#19-additional-notes--open-items)

# 1. Motivation

SystemVerilog classes are fundamental for verification-oriented code (e.g., UVM, testbenches).[^8]
Currently, the **Moore dialect** lacks abstractions for classes, handles, or object construction.

The goal is to introduce class support that:

- Integrates cleanly with existing Moore concepts (`moore.read`, `moore.ref`, `moore.call`).
- Lowers predictably to LLVM using heap-allocated `struct` objects and standard vtable dispatch.

The implementation further tries to stick closely to standard C++ lowering conventions, as for example defined in [Itanium C++ ABI](https://itanium-cxx-abi.github.io/cxx-abi/abi.html#member-pointers).
This document defines class support according to *IEEE 1800-2023 § 8 “Classes”*[^8].

Scope of this initial design:

- Non-virtual, virtual, and interface classes[^8.20].
- Single inheritance only[^8.13].
- Virtual and non-virtual methods[^8.14].
- Properties without `rand` or `randc`[^8.22].
- No covergroups or constraint blocks.

Support for randomization, constraints, and covergroups may be added in later extensions.

# 2. Overview

The design extends the Moore dialect with:

1. A **class handle type**: `!moore.class<@C>`.
2. A minimal set of **class operations** for allocation, field/method access, and virtual dispatch.
3. Frontend lowering rules mapping Slang’s `ClassSymbol` into Moore IR.
4. Backend lowering rules mapping Moore IR to LLVM (heap allocation, vtable dispatch, etc.).

This enables lowering of typical SystemVerilog class features used in testbenches and UVM environments,
including virtual functions and single inheritance.

# 3. Frontend Integration

## 3.1 Class Metadata

Introduce a `ClassLowering` structure in `Context`:

- `sym` — associated `ClassSymbol`
- `fields` — ordered list of `(name, type)`
- `methods` — ordered list of `(name, functionLowering, visibility, virtual, static)`
- `base` — optional base class symbol
- `needsVTable` — `true` if any method is virtual or the class inherits from a virtual base

This parallels `FunctionLowering` and is used during class body conversion[^8.6].


# 4. Object Layout

## 4.1 Description

Each `!moore.class<@C>` value represents a heap-allocated, reference-counted structure with a deterministic memory layout.
The first word of every instance is a pointer to its **vtable**, enabling virtual dispatch[^8.14].

Fields are laid out in declaration order (including inherited ones), so the **base class subobject** always begins at offset 0.
This makes upcasting a zero-cost reinterpretation[^8.13].

Derived classes extend the base layout by appending their own fields.
Field references resolve to byte offsets within the contiguous object memory, allowing both field access and superclass calls
to be implemented by direct pointer arithmetic.

This layout maps naturally to LLVM `struct` types, ensuring ABI stability, efficient `super` calls, and predictable runtime behavior.

## 4.2 Example

Below is an example of the memory layout for a simple inheritance chain:

```systemverilog
class Base;
  int a;
  int b;
endclass

class Derived extends Base;
  int c;
endclass
```

```mlir
// Base class
!moore.class<@Base> =
  !llvm.ptr<
    !llvm.struct<(
      !llvm.ptr<!vtable<Base>>,  // vtable pointer
      i32,                       // Base::a
      i32                        // Base::b
    )>
  >

// Derived class (extends Base)
!moore.class<@Derived> =
  !llvm.ptr<
    !llvm.struct<(
      !llvm.ptr<!vtable<Base>>,  // vtable pointer (inherited)
      i32,                       // Base::a
      i32,                       // Base::b
      i32                        // Derived::c
    )>
  >
```

# 5. Core Moore Operators

| Op | Description |
|-----------|----------------|
| `moore.class.new @C(args)` | Allocate and construct a new object [^8.7]. |
| `moore.class.delete %inst` | Destruct and deallocate an object [^8.29] |
| `moore.class.field_ref %inst, @field` | Get a `!moore.ref` to a field [^8.5] |
| `moore.class.method.call @C::method(%inst, args)` | Direct (non-virtual) method call [^8.10] |
| `moore.class.vcall @C::method(%inst, args)` | Virtual method dispatch [^8.14] |
| `moore.class.upcast %inst : !moore.class<@Derived> to !moore.class<@Base>` | Return an upcast version of the instance, e.g. when using `super` [^8.13]|
| `moore.class.downcast %inst : !moore.class<@Base> to !moore.class<@Derived>` | Return a downcast version of the instance, implements `cast` [^8.16] |

# 6. SV-to-Moore Lowering Rules

| SystemVerilog Construct | Lowering |
|--------------------------|----------|
| `new C(args)` | `moore.class.new @C(args)` |
| `super`| `moore.class.upcast %inst : !moore.class<@Derived> to !moore.class<@Base>` |
| `this` | Implicit `%this : !moore.class<@C>` parameter [^8.8] |
| `h.f` | `moore.class.field_ref %inst, @f` (then `moore.read` or `moore.assign`) |
| `h.m(args)` | `moore.class.method.call` or `moore.class.vcall` |
| Class methods | Lowered as `func.func private @C::m(%this, args…)` |
| Constructors / Destructors | `@C$ctor`, `@C$dtor` [^8.7][^8.29]|


# 7. Moore Lowering Example Sketch

## 7.1 Full System Verilog Example Sketch

``` systemverilog
class C;
  int f;

  // Constructor
  function new(int init);
    this.f = init;
  endfunction

  // Non-virtual method
  function int add(int a);
    return f + a;
  endfunction

  // Virtual method
  virtual function int scale(int k);
    return f * k;
  endfunction
endclass

function automatic void top_single(output int y, output int z);
  C h;
  h = new(42);
  y = h.add(5);     // -> 47
  z = h.scale(3);   // -> 126 (virtual call)
endfunction
```

``` mlir

// Constructor: @C$ctor(%this, %init)
func.func private @C$ctor(%this: !moore.class<@C>, %init: !moore.i32) {
  // %fref : ref<i32> to field @C::f inside *this
  %fref = moore.class.field_ref %this, @C::f : <i32>
  moore.blocking_assign %fref, %init : i32
  func.return
}

// Non-virtual method: @C::add(%this, %a) -> i32
func.func private @C::add(%this: !moore.class<@C>, %a: !moore.i32) -> !moore.i32 {
  %fref = moore.class.field_ref %this, @C::f : <i32>
  %fval = moore.read %fref : <i32>
  %sum  = moore.add %fval, %a : i32
  func.return %sum : i32
}

// Virtual method: @C::scale(%this, %k) -> i32
// (still a normal function; dynamic dispatch happens at call sites via vcall)
func.func private @C::scale(%this: !moore.class<@C>, %k: !moore.i32) -> !moore.i32 {
  %fref = moore.class.field_ref %this, @C::f : <i32>
  %fval = moore.read %fref : <i32>
  %prod = moore.mul %fval, %k : i32
  func.return %prod : i32
}


// Lowering of `top_single` to Moore/MLIR (one function scope).
func.func private @top_single(%y: !moore.ref<i32>, %z: !moore.ref<i32>) {
  // h = new(42)
  %c42 = moore.constant 42 : i32
  %h   = moore.class.new @C(%c42) : (!moore.i32) -> !moore.class<@C>

  // y = h.add(5)      -- direct (non-virtual) call
  %c5  = moore.constant 5 : i32
  %r1  = moore.class.call @C::add(%h, %c5)
         : (!moore.class<@C>, !moore.i32) -> !moore.i32
  moore.blocking_assign %y, %r1 : i32

  // z = h.scale(3)    -- virtual dispatch
  %c3  = moore.constant 3 : i32
  %r2  = moore.class.vcall @C::scale(%h, %c3)
         : (!moore.class<@C>, !moore.i32) -> !moore.i32
  moore.blocking_assign %z, %r2 : i32

  func.return
}
```

## 7.2 Virtual Class Lowering

``` systemverilog
virtual class Base;
  pure virtual function int f(int a);
endclass

class D extends Base;
  function new(int seed);
    this.seed = seed;
  endfunction
  function int f(int a); return a + seed; endfunction
  int seed;
endclass

module top(output int y);
  D h = new(1);
  initial y = h.f(41); // 42
endmodule
```

``` mlir
moore.class @Base { abstract }
moore.class @D extends @Base { fields { %seed: !moore.i32 } }

func.func private @D$ctor(%this: !moore.class<@D>, %seed: !moore.i32) {
  %seed.ref = moore.class.field_ref %this, @D::seed : !moore.ref<i32>
  moore.blocking_assign %seed.ref, %seed : i32
  func.return
}

func.func private @D::f(%this: !moore.class<@D>, %a: !moore.i32) -> !moore.i32 {
  %seed.ref = moore.class.field_ref %this, @D::seed : !moore.ref<i32>
  %seed     = moore.read %seed.ref : <i32>
  %r        = moore.add %a, %seed : i32
  func.return %r : i32
}

moore.module @top {
  %c1_32 = moore.constant 1 : i32
  %c41_32 = moore.constant 41 : i32
  %y  = moore.variable : <i32>
  %h  = moore.class.new @D(%c1_i32) : (!moore.i32) -> class<@D>
  %r  = moore.class.vcall @Base::f(%h, %c41_i32) : (!moore.class<@D>, !moore.i32) -> i32
  moore.blocking_assign %y, %r : i32
  moore.output
}
```

# 8. Moore -> LLVM Lowering

## 8.1 Representation

- LLVM struct type:
  `%class.C = type { %vptr*, <fields...> }` (if virtual)
  otherwise `{ <fields...> }`
- Handle type: `!llvm.ptr<!llvm.struct<"class.C">>`

## 8.2 Lowering Rules

| Moore Op | LLVM Lowering |
|-----------|----------------|
| `moore.class.new` | `malloc` + optional ctor call |
| `moore.class.delete` | call dtor + `free` |
| `moore.class.field_ref` | `llvm.getelementptr` |
| `moore.class.method.call` | direct `llvm.call` |
| `moore.class.vcall` | load vptr, GEP to slot, indirect `llvm.call` |
| `moore.class.upcast` | bitcast |
| `moore.class.downcast` | See [The dynamic_cast algorithm](https://itanium-cxx-abi.github.io/cxx-abi/abi.html#dynamic_cast-algorithm) |

# 9. Naming and Visibility

| Entity | Symbol Name |
|---------|--------------|
| Method | `@C::m` |
| Constructor | `@C$ctor` |
| Destructor | `@C$dtor` |
| VTable | `@C$vtable` |

Visibility is controlled per § 8.11 “Visibility of class members”[^8.11].

# 10. Object Lifetime and Memory Management

SystemVerilog class handles are conceptually garbage-collected — there is no user-visible delete [^8.29]. To make this deterministic and interoperable with LLVM, the Moore dialect adopts **reference-counted handles**:

- Every `!moore.class<@C>` handle carries a reference count alongside the object.
- `moore.class.new` initializes the count to `1`.
- Assignment between handles inserts a `moore.class.retain` / `moore.class.release` pair.
- When the count reaches zero, the object’s destructor (if present) runs, then memory is freed.


| Operation | Description |
|------------|-------------|
| `moore.class.retain %h` | Increment refcount |
| `moore.class.release %h` | Decrement refcount and delete on zero |
| `moore.class.null` | Create a null handle |
| `moore.class.isnull %h` | Returns `i1` indicating whether `%h` is null |

For simulator integration, these may lower to runtime hooks such as $moore_new or $moore_release.
moore.class.delete remains an internal operation for explicit destruction (e.g., deterministic testbench teardown) but has no SV source equivalent.

# 11. Static Members and Initialization

Static members follow § 8.17 “Static members”[^8.17].
Static properties and methods of a class are modeled as globals within the class namespace.

- Static fields lower to `moore.variable` at module scope with symbol names like `@C$static::x`.
- Static methods lower to regular `func.func` without a `%this` argument.
- Initialization order follows SV semantics:
  statics are initialized once before the first use of the class (elaborated in elaboration order).

```mlir
class C;
    static int counter = 0;
endclass

%counter = moore.variable 0 : <i32>
```

# 12. Class Tasks and Timing Semantics

Class *tasks* (§ 8.15 “Tasks and functions in classes”) are methods that may contain time controls (`@`, `#`, `wait`, etc.)[^8.15].
They lower to Moore coroutines:

- Each task is represented as a `moore.proc` region (already supported for `always`).
- The implicit `%this` argument is captured in the coroutine environment.
- Virtual tasks dispatch through the same vtable as functions, but return a handle to a process instead of a value.

Example:

```systemverilog
class C;
  task run();
    #10 $display("Done");
  endtask
endclass
```

``` mlir
func.func private @C::run(%this: !moore.class<@C>) -> !moore.proc {
  moore.delay (10ns);
  moore.display "Done";
  func.return;
}
```

# 13. Dynamic Type Operations

SystemVerilog supports `$cast`, `is`, and `typename`[^8.16].
This extension models these as RTTI (runtime type info) queries through the vtable pointer.

| SystemVerilog | Lowering | Description |
|----------------|-----------|-------------|
| `$cast(Dest, Src)` | `moore.class.downcast %src : !moore.class<@Base> to !moore.class<@Derived>` | Returns null if dynamic type check fails |
| `Src is Type` | `moore.class.isa %src, @Type` | Returns `i1` |
| `Src.typename()` | `moore.class.typename %src` | Returns `!moore.string` |

Each vtable stores a type descriptor pointer, enabling O(1) dynamic type checks.

# 14. VTable Construction and Inheritance

Each class with virtual methods defines a vtable symbol[^8.14] containing a single type descriptor and entries for all virtual dispatch methods.

```mlir
moore.vtable @C$vtable {
  type_descriptor = @C$typeinfo
  entries = [
    @C::scale,          // slot 0
    @C::virtualTask     // slot 1
  ]
}
```

where the type_descriptor contains a pointer to the class name, pointer to the inherited class type (null for roots), a bitmask for flags (abstract, interfaces, virtual base, etc.).

``` mlir
%typeinfo.C = type {
  ptr @.typename.C,        ; mangled class name
  ptr @C$base,             ; pointer to base typeinfo (null for roots)
  i32 %flags,              ; bitmask (abstract, interface, virtual base, etc.)
}
```

A derived class copies and overwrites base entries.
* A vtable symbol has type `!llvm.struct<(ptr to funcs...)>`.
* On instantiation, `moore.class.new` writes a pointer to the correct vtable at object offset 0.

During lowering:
* `moore.class.vcall` -> (load vptr -> gep slot -> indirect llvm.call)

# 14.1 VTable Layout and RTTI Integration — Full Example

The following example demonstrates how vtables, RTTI, and inheritance interact in the Moore class model.

## Example Source (SystemVerilog)

```systemverilog
virtual class Base;
  int x;
  function new(int v);
    x = v;
  endfunction

  virtual function int f();
    return x + 1;
  endfunction
endclass

class Derived extends Base;
  function new(int v);
    super.new(v);
  endfunction

  function int f();
    return x + 10;
  endfunction
endclass

function int top();
  Base b;
  Derived d;
  int r1, r2;

  d = new(32);
  b = d;             // upcast
  r1 = b.f();        // virtual call
  if ($cast(Derived'(b), d))  // dynamic cast
    r2 = d.f();
  return r1 + r2;
endfunction
```

## Example Source (Moore Class Dialect)

``` mlir
// Base class metadata
moore.class @Base {
  fields { %x: !moore.i32 }
  methods {
    @Base::f : virtual,
    @Base$ctor : constructor
  }
}

// Derived class metadata
moore.class @Derived extends @Base {
  methods {
    @Derived::f : overrides @Base::f,
    @Derived$ctor : constructor
  }
}

//
// RTTI Records
//
moore.rtti @Base$typeinfo {
  name = "Base",
  base = null,
  flags = #moore.class_flags<virtual>
}

moore.rtti @Derived$typeinfo {
  name = "Derived",
  base = @Base$typeinfo,
  flags = #moore.class_flags<virtual>
}

//
// VTables
//
moore.vtable @Base$vtable {
  type_descriptor = @Base$typeinfo
  entries = [ @Base::f ]
}

moore.vtable @Derived$vtable {
  type_descriptor = @Derived$typeinfo
  entries = [ @Derived::f ] // overrides Base::f in slot 0
}

//
// Method Implementations
//
// Base::f
func.func private @Base::f(%this: !moore.class<@Base>) -> !moore.i32 {
  %xref = moore.class.field_ref %this, @Base::x : <i32>
  %x = moore.read %xref : <i32>
  %add = moore.add %x, (moore.constant 1 : i32) : i32
  func.return %add : i32
}

// Derived::f (overrides)
func.func private @Derived::f(%this: !moore.class<@Derived>) -> !moore.i32 {
  %xref = moore.class.field_ref %this, @Base::x : <i32>
  %x = moore.read %xref : <i32>
  %add = moore.add %x, (moore.constant 10 : i32) : i32
  func.return %add : i32
}

// top() function demonstrating upcast, vcall, and downcast
func.func private @top() -> !moore.i32 {
  %c32 = moore.constant 32 : i32

  // d = new(32)
  %d = moore.class.new @Derived(%c32) : (!moore.i32) -> !moore.class<@Derived>

  // b = d (implicit upcast)
  %b = moore.class.upcast %d : !moore.class<@Derived> to !moore.class<@Base>

  // r1 = b.f()  (virtual dispatch)
  %r1 = moore.class.vcall @Base::f(%b) : (!moore.class<@Base>) -> !moore.i32

  // if ($cast(Derived'(b), d))
  %d2 = moore.class.downcast %b : !moore.class<@Base> to !moore.class<@Derived>
  %isvalid = moore.class.isnull %d2
  %notnull = moore.not %isvalid : i1
  // Branch on successful cast
  cf.cond_br %cond, ^then, ^else

^then:
  %r2 = moore.class.call @Derived::f(%d2)
          : (!moore.class<@Derived>) -> !moore.i32
  %sum_then = moore.add %r1, %r2 : i32
  cf.br ^merge(%sum_then : i32)

^else:
  cf.br ^merge(%r1 : i32)

^merge(%res: i32):
  func.return %res : i32
}
```

# 15. Abstract Classes and Pure Virtual Methods

Virtual methods without implementation (`pure virtual`) mark a class abstract [^8.21].

Lowering:

```mlir
moore.class @Base { abstract, vtable { entry @Base::f = null } }
```

* Instantiating an abstract class emits an elaboration-time error. (Checked by Slang)
* A derived class must provide concrete implementations to fill null slots.
* Attempting to vcall an unimplemented slot results in a runtime assertion.

# 16. Type Parameters

Parameterized classes (e.g. `class C #(type T=int);`) are instantiated per specialization[^8.23]. Each specialization becomes a distinct Moore symbol:

```mlir
moore.class @C<int> { ... }
moore.class @C<real> { ... }
```

The parameter environment is part of the class symbol name for both IR and vtable emission.

# 17. Future Work / Out of Scope

- Randomization (`rand`, `randc`, `constraint` blocks)
- Covergroups and coverage methods
- Mailboxes, semaphores, event handles inside classes
- DPI passing of class handles
- Multiple inheritance

Each can be modeled as an extension layer once the basic class infrastructure and handle semantics are in place.

# 18. Summary

This design introduces class support into Moore with minimal IR surface and clean LLVM lowering:

- Compact: one type and ~7 core ops.
- Consistent: integrates with existing `ref`/`read`/`assign`.
- Extensible: supports virtual dispatch and inheritance later.

It enables import of verification-oriented SV codebases (e.g., UVM) while maintaining clean lowering to LLVM for testbench simulation.

# 19. Additional Notes & Open Items

This section summarizes open design considerations, deferred decisions, and future clarifications needed for a robust class implementation.

## 19.1 Handle semantics and nullability

- Equality (`==`) on `!moore.class<@C>` compares **handle identity**, not object contents.
- Dereferencing (`field_ref`, `vcall`) on a null handle is **undefined**; a verifier pass should diagnose it.
- A canonical `moore.class.null` op produces the null handle constant.

## 19.2 Copy, move, and cloning

- Handle assignment copies the handle and updates the reference count (retain/release).
- No implicit deep copy: cloning is expressed via user-defined `C::copy()` or library helpers.
- Moves can be modeled by direct ownership transfer if refcount elision is implemented.

## 19.3 Reference counting details

- Retain/release ops are inserted at SSA join points, function returns, and variable assignments.
- A future `RCOpt` pass may cancel redundant pairs and perform lifetime analysis.
- Reference cycles are **out of scope**; users must break cycles manually (typically in destructors).
- Runtime hooks (`$moore_new`, `$moore_release`) may replace built-in RC operations for simulators.

## 19.4 Destructors and ordering

- Destructors execute in **derived-to-base** order when reference count reaches zero.
- They are always `nothrow`; failure should issue a diagnostic but not terminate execution.
- Each dtor is lowered as `@C$dtor(%this)` and may call `super` destructors explicitly.

## 19.5 Visibility and access control

- `local` and `protected` visibility rules are enforced by a verifier pass.
- Access across packages (`pkg::C`) requires explicit qualification.
- Nested classes and packages use hierarchical naming: `@pkg::C::m`.

## 19.6 Parameterized classes and specialization

- Parameter environments form part of the symbol identity (e.g. `@C<int,4>`).
- Equivalent specializations share vtables where possible.
- A verifier ensures slot-count compatibility between base and derived specializations.

## 19.7 VTable stability and ABI

- Slot numbering is fixed at IR emission and preserved across lowering stages.
- The verifier ensures overrides match base signatures exactly.
- Interface classes use compact vtables containing only pure-virtual entries.

## 19.8 Runtime type information (RTTI)

- Each vtable points to a type descriptor structure containing:
  - Class name
  - Package path
  - Base class pointer
  - Hash or GUID
  - Attribute flags (abstract, interface, etc.)
- `$cast`, `is`, and `typename` queries read this descriptor; all are O(1) operations.

## 19.9 Super calls

- Canonical lowering:
```mlir
%base = moore.class.upcast %this : !moore.class<@Derived> to !moore.class<@Base>
moore.class.call @Base::m(%base, args…)
```

- Direct base calls may be inlined when verifier can guarantee target uniqueness.

## 19.10 Tasks and timing semantics

- Class tasks are lowered to `moore.proc` coroutines.
- They run in the *active* simulation region unless time controls delay them.
- Virtual tasks share the same vtable slots as virtual functions.

## 19.11 DPI and simulator interop

- Out of scope for now. Future extensions may expose opaque handle structs to DPI,
providing retain/release wrappers for safe crossing.

## 19.12 Optimization and analysis

- `retain`/`release` are side-effecting ops to preserve correct lifetime semantics.
- `RCOpt` may elide redundant pairs or promote short-lived objects to stack allocations.
- Inlining rules: never inline through `vcall`; inline `class.call` when private and visible.
- Escape analysis may permit stack allocation of short-lived objects (e.g., local temporaries).

## 19.13 Verification and diagnostics

Verifier checks include:
- Null handle use.
- Virtual call on abstract or null vtable entry.
- Override signature mismatch.
- Constructor failure to initialize vptr before use.
- Diagnostics should identify class, field, and method names to aid debugging.

## 19.14 ABI alignment and memory layout

- vptr is aligned to pointer width (target ABI).
- Field alignment matches LLVM’s struct ABI rules.
- All inheritance chains produce compatible layouts for safe bitcasting.

## 19.15 Pass pipeline

Recommended lowering pipeline:
Slang Frontend
* Class IR emission
* Moore verifier
* Retain/Release insertion
* RC optimization
* VTable materialization
* Lower field_ref
* Lower class ops
* LLVM conversion

## 19.16 Testing plan

- **Unit:** object layout, upcast/downcast, vtable ordering, pure-virtual enforcement.
- **Refcount:** retain/release balancing, destructor order, null-handle behavior.
- **Integration:** inheritance chains, virtual calls, interface conformance.
- **Performance:** measure vcall latency and RC overhead.

## 19.17 Reserved future hooks

- Randomization (`rand`, `randc`, `constraint`) intrinsics.
- Covergroup integration.
- Weak references to handle cyclic structures safely.
- Object monitors for concurrent access (future extension).

Together, these notes define the broader technical envelope for class support in the Moore dialect—clarifying lifetime, ABI, optimization, and verification concerns beyond the core lowering rules.

# Appendix A — Clause Reference Table

| Feature | IEEE 1800-2023 Clause |
|----------|-----------------------|
| General class semantics | § 8 “Classes” |
| Class declarations | § 8.1 “Class declaration” |
| Handles and this | § 8.8 “this handle” |
| Constructors | § 8.7 “Constructors” |
| Field declarations | § 8.9 “Class properties” |
| Methods and calls | § 8.10 “Methods” |
| Visibility rules | § 8.11 “Visibility of class members” |
| Inheritance and super | § 8.13 “Inheritance and subclasses” |
| Virtual methods | § 8.14 “Virtual methods” |
| Tasks and functions | § 8.15 “Tasks and functions in classes” |
| Casting (`$cast`, `is`, `typename`) | § 8.16 “Casting” |
| Static members | § 8.17 “Static members” |
| Abstract classes | § 8.21 “Abstract classes and pure virtual methods” |
| Parameterized classes | § 8.23 “Parameterized classes” |
| Memory management | § 8.29 “Memory management” |
| Randomization (out of scope) | § 18 “Random stability and constraints” |

[^8]: IEEE 1800-2023 § 8 “Classes”
[^8.5]: IEEE 1800-2023 § 8.5 “Object properties and object parameter data”
[^8.6]: IEEE 1800-2023 § 8.6 “Class members”
[^8.7]: IEEE 1800-2023 § 8.7 “Constructors”
[^8.8]: IEEE 1800-2023 § 8.8 “this handle”
[^8.9]: IEEE 1800-2023 § 8.9 “Class properties”
[^8.10]: IEEE 1800-2023 § 8.10 “Methods”
[^8.11]: IEEE 1800-2023 § 8.11 “Visibility of class members”
[^8.13]: IEEE 1800-2023 § 8.13 “Inheritance and subclasses”
[^8.14]: IEEE 1800-2023 § 8.14 “Virtual methods”
[^8.15]: IEEE 1800-2023 § 8.15 “Tasks and functions in classes”
[^8.16]: IEEE 1800-2023 § 8.16 “Casting”
[^8.17]: IEEE 1800-2023 § 8.17 “Static members”
[^8.20]: IEEE 1800-2023 § 8.20 “Interface classes”
[^8.21]: IEEE 1800-2023 § 8.21 “Abstract classes and pure virtual methods”
[^8.22]: IEEE 1800-2023 § 8.22 “Random variables”
[^8.23]: IEEE 1800-2023 § 8.23 “Parameterized classes”
[^8.29]: IEEE 1800-2023 § 8.29 “Memory management”
