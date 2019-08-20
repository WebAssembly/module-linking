# Module Types for WebAssembly

A proposal to enhance module types to include import/export names and define
text format parse rules, so that module types can be written independently of
module definition.

This proposal is backwards compatible, does not change the `.wasm` binary
format and requires no changes to wasm consumers such as browsers.

## Motivation

As the WebAssembly ecosystem emerges, there is a growing need to be able
to define the interface of a module independently of the current `.wat` text
format, which needs to include the definition of a module as well. Specific
uses cases include [WASI] or package managers that wish to expose and check
public interfaces.

Ideally, the interface of a WebAssembly module can be written in a
language-neutral format that language-specific interfaces can be automatically
derived from so only one interface needs to be written by the author.

## Summary

The current core spec *already* [classifies modules with a type][Module Validation],
specifically with a function type (from imports to exports). However, to describe
a full module interface, the names of imports and exports must be known as well.
Thus, the proposal enriches the existing **module type** by:
1. adding names to imports and exports
2. assigning text format parse rules

For example, the module:

```wasm
(module
  (memory (import "a" "x") 1 2)
  (func (import "a" "y") (param i32))
  (table (export "b") 1 funcref)
  (func $notImportedOrExported (result i64)
    i64.const 0
  )
  (func (export "c") (result f32)
    f32.const 0
  )
)
```

would have a module type that could be written in the text format as:

```wasm
(module
  (memory (import "a" "x") 1 2)
  (func (import "a" "x") (param i32))
  (table (export "b") 1 funcref)
  (func (export "c") (result f32))
)
```

Here, the `module` keyword is used for both the definition and the type,
symmetric to the existing text format convention where, e.g., `func` is used
in both the definition and types of functions.

In WebAssembly there is also the separate concept of a module *instance*,
which is the result of [instantiating] a module with imports. In loader
frameworks like [ESM], where instantiation is performed by the loader,
not by client modules, or for built-in modules like [WASI], where instantiation
is logically performed by the host, the type of module *instances* may be
what is actually needed for the toolchain. Thus, the proposal also defines
**instance type** along with text format parse rules. 

For example, the above module, when instantiated, would have an instance type
that could be written in the text format as:

```wasm
(instance
  (table (export "b") 1 funcref)
  (func (export "c") (result f32))
)
```

Just as the current [text format conventions recommend `.wat`][WAT] as the
extension of a file that contains a module definition, the proposal would
include a new recommendation for text files containing module types or
instance types.

TODO: add a strawman recommendation (`.wmt`+`.wit`? or just share `.wit`?)


[WASI]: https://github.com/webassembly/wasi
[Module Validation]: https://webassembly.github.io/spec/core/valid/modules.html#valid-module
[WAT]: https://webassembly.github.io/spec/core/text/conventions.html
[instantiating]: https://webassembly.github.io/spec/core/appendix/embedding.html#embed-module-instantiate
[ESM]: https://github.com/WebAssembly/esm-integration
