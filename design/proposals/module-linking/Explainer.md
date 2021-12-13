# Module Linking Explainer

This explainer introduces the Module Linking proposal, as the first proposal
of the [Component Model] specification. Module Linking enables multiple Core
WebAssembly modules to be linked together without relying on imported host
functionality.

1. [Problem](#problem)
2. [Use Cases](#use-cases)
3. [Additional Requirements](#additional-requirements)
4. [Walkthrough](#walkthrough)
   1. [Module Definitions](#module-definitions)
   2. [Instance Definitions](#instance-definitions)
   3. [Import Definitions](#import-definitions)
   4. [Export Definitions](#export-definitions)
   5. [Alias Definitions](#alias-definitions)
   6. [Type Definitions](#type-definitions)
5. [Binary Format Considerations](#binary-format-considerations)
6. [Web Platform Integration](#web-platform-integration)


## Problem

Currently, WebAssembly modules have no way to define how they are to be
instantiated and linked together without relying on host-specific functionality
such as the [JS API]. Consequently, the only portable way to link modules today
is to [statically link] them using toolchain-specific conventions. This leads
to code duplication in production and inhibits cross-language reuse. A number
of [Component Model use cases] require support for more dynamic forms of
linking that enable the sharing of separately-compiled modules written in
different languages.


## Use Cases

To motivate the proposed solution, we consider 2 use cases whose requirements
aren't satisfied by simpler solutions.


### Link-time Virtualization

When using a first-class instantiation API like the JS API's
[`WebAssembly.instantiate()`], the imports of the module-to-be-instantiated
appear as explicit arguments supplied by the caller. This has several useful
properties that should be preserved by a pure-wasm linking solution:

First, it enables applications to easily enforce the [Principle of Least Authority],
passing modules only the imports necessary to do their job. There are currently
[cases][Figma plugins] of Web applications relying on this property to sandbox
untrusted plugins.

Second, it enables client modules to fully or partially [virtualize] the imports
of their dependencies without extraordinary challenge or performance overhead.
For example, if a module imports a set of file operations and a client wants to
reuse this module on a system without builtin file I/O, the client should be
able to virtualize the file operations with an in-memory implementation.
Virtualization also enhances the ability of developers to apply the Principle of
Least Authority, by allowing a client module to not only pass *subsets* of
capabilities, but to also dynamically [attenuate] capabilities. Lastly,
virtualization allows developers to locally test modules intended for later
deployment by [mocking] deployment APIs with local implementations. In general,
if virtualization is well-supported and efficient, software reusability and
composability are increased.

While it is possible to perform virtualization at run-time, e.g., passing
function references for all virtualizable operations, this approach would be
problematic as a basis for a highly-virtualizable ecosystem since it would
require modules to intentionally opt into virtualization by choosing to receive
first-class function references instead of using function imports. In the limit,
to provide maximum flexibility to client code, toolchains would need to avoid
*all* use of imports, effectively annulling a core part of wasm. Because of
their more-static nature, imports are inherently more efficient and optimizable
than first-class function references, so this would also have a negative
performance impact.

Thus, to avoid the problems of run-time virtualization, a wasm linking solution
should enable **link-time virtualization** such that a parent module can
specify all the imports of its dependencies (without any explicit opt-in on the
part of those dependencies), just as with the JS API's [`instantiate`]. Note
that it's possible to achieve run-time virtualization by supplying link-time
function imports that perform dynamic dispatch on their parameters. Link-time
virtualization is thus a fairly general mechanism to enable multiple styles of
virtualization.

As an example, given the static dependency graph on the left (where all 3
modules import a host-supplied instance of `wasi:filesystem`), it should be
possible for `parent.wasm` to generate the linked instance graph on the right,
where the `parent` instance has created a `virtualized` instance and supplied
it to a new `child` instance as an implementation of the `wasi:filesystem`
interface:

<p align="center"><img src="./link-time-virtualization.svg" width="500"></p>

Importantly, the `child` instance has no access to the `wasi:filesystem`
instance imported by the `parent` instance.

A worked example of this use case is given [here](Example-LinkTimeVirtualization.md).


### Shared-Everything C-family Dynamic Linking

"Dynamic Linking" refers to keeping modules separate until load-time so that
common modules can be shared by multiple independent clients. This avoids the
need to [statically link]  (thereby duplicating) the shared modules into each
client. "Shared-Everything" linking refers to the case where the linked modules
share memory and tables, effectively emulating [native dynamic linking].
Lastly, "C-family" refers to the family of languages that can interoperate
through a traditional C ABI (e.g., C, C++, FORTRAN and Rust).

Two modules are dynamically linked by having one module import the functions,
memories, tables and globals exported by another module. One challenge for
dynamic linking is controlling which module instances share a linear memory and
which have separate linear memories. In the [shared-nothing] context of the
Component Model, we want all the core modules used by a single component to
share a single linear memory while we want each distinct component instance to
get a distinct linear memory.

For example, it should be possible to take the static dependency graph of the
application on the left, which is built from two components (`zipper` and
`imgmgk`) and three shared core modules (`libc`, `libzip` and `libimg`), and
create the dynamically-linked instance graph on the right at load-time:

<p align="center"><img src="./shared-everything-dynamic-linking.svg" width="600"></p>

Here, `libc` defines and exports a linear memory that is imported by the other
core instances contained within the same component instance. The whole
application uses the `libc` module three times, sharing *code*, but with each
usage getting a distinct instance and thus not sharing *data*. Module systems
where sharing a module forces sharing a module instance are common in existing
programming languages but break shared-nothing isolation and this use case.

A worked example of this use case is given [here](Example-SharedEverythingDynamicLinking.md).


## Additional Requirements

Additionally, the following high-level Component Model [design choices] are
relevant:

* **No GC dependency**: Since we wish to use this feature to implement dynamic
  linking of C/C++/Rust programs, and since C/C++/Rust programs aim to run on
  hosts that don't contain a GC, this proposal should not require a general
  garbage collection algorithm to implement.

* **No JIT dependency**: Today, using static linking of libraries, wasm
  hosts are able to use a simple, Ahead-of-Time compilation model to generate
  high-performance machine code images that can be immediately instantiated and
  run. Switching to dynamic linking should not hurt hosts' ability to perform
  the same degree of optimization.

* **No global registries at runtime**: A number of existing dependency
  resolution mechanisms with similar use cases as Module Linking depend on a
  shared mutable registry into which, at runtime, modules are registered and
  names are resolved.


## Walkthrough

This proposal bootstraps the Component Model by defining a new "adapter module"
concept. Adapter modules *contain* Core WebAssembly ("core") modules as well as
several new definition kinds that address the use cases listed above. Just like
core modules, adapter modules have abstract syntax with both text- and
binary-format representations. Similarly, adapter modules are meant to be
distributable binaries that can be directly loaded and executed by a supporting
runtime.

(As a spoiler: adapter modules are extended by the [Interface Types] proposal
with the additional features necessary to achieve the [shared-nothing] goals of
the Component Model. The Interface Types proposal also defines a *restricted*
form of adapter modules called **components** which are adapter modules whose
public interfaces (imports and exports) *only* use interface types, thereby
ensuring shared-nothing isolation of their contained shared mutable state.
Ultimately, both definitions&mdash;adapter modules and components&mdash;are
necessary for different scenarios.)

This walkthrough section introduces adapter modules via their text format. The
[next section](#binary-format-considerations) discusses the high-level
considerations for the binary format of adapter modules while a
[sibling document](Binary.md) provides a detailed binary format description.


### Module Definitions

Like core modules, adapter modules are composed of sequences of definitions.
This proposal includes 6 definition kinds:
```
adapter-module ::= (adapter module <id>? <definition>* )
definition     ::= <module>
                 | <instance>
                 | <import>
                 | <export>
                 | <alias>
                 | <type>
module         ::= <adapter-module>
                 | <core:module>
```
From this grammar fragment we can see that the `adapter-module` production is
recursive and thus adapter modules can contain other adapter modules. The
`core:module` production refers to the Core WebAssembly's [`module`]
parse rule. Together, this means that adapter modules can form trees with core
modules appearing only at leaves.

For example, the following adapter module contains one adapter module and two
core modules:
```wasm
(adapter module
  (adapter module $A
    (module $B
      (func (export "one") (result i32) (i32.const 1))
    )
  )
  (module $C
    (func (export "two") (result i32) (i32.const 2))
  )
)
```
Just like other definitions, adapter and core module definitions go into an
[index space] called the *module index space* which can be referenced by other
definitions (in the text format, via symbolic identifiers like `$A`, `$B` and
`$C` above). Notably, adapter and core modules go into the *same* module index
space since the users of the module index space don't distinguish between
adapter and core modules; after definition, both kinds of modules simply have a
*module type* against which uses are checked (more on module types below).

Unlike core modules, the definitions inside an adapter module are inherently
*ordered*: definitions can only reference preceding definitions so that adapter
module definitions are acyclic by nature. This acyclicy reflects the acyclic
nature of Core WebAssembly's [module instantiation], which adapter modules can
perform (as described below).

Lastly, the syntax and semantics of core modules are not modified by adapter
modules; all interaction with core modules happens through imports and exports
and thus, from the Core WebAssembly's point of view, the Component Model is
just another [WebAssembly embedder][core concepts].


### Instance Definitions

Instance definitions are the central feature of the Module Linking proposal,
allowing adapter modules to create and link module instances. Whenever an
adapter module is instantiated, all of its contained instance definitions are
executed, creating new module instances according to the contained
`instance-expr` expressions. The syntax of instance definitions is:
```
instance      ::= (instance <id>? <instance-expr>)
instance-expr ::= (instantiate <moduleidx> (import <name> <def-ref>)*)
                | (export <name> <def-ref>)*
def-ref       ::= (instance <instanceidx>)
                | (module <moduleidx>)
                | (func <funcidx>)
                | (table <tableidx>)
                | (memory <memidx>)
                | (global <globalidx>)
```
As an example, the following `$Parent` adapter module creates a new `$Child`
module instance every time `$Parent` is instantiated:
```wasm
(adapter module $Parent
  (module $Child
    (memory 1)
  )
  (instance $child (instantiate $Child))
  ...
)
```
The fresh `$Child` instance is added to the *instance index space* which other
definitions (in the `...`) can refer to via the identifier `$child`. Without
the `instance` definition, no instance of `$Child` would be created. If there
were a second `instance` definition of `$Child`, two instances would be
created. Thus, this proposal Module Linking separates the concepts of *defining
a module* from *creating an instance* of a module.

A more interesting example uses instance definitions to link two core modules
together:
```wasm
(adapter module
  (module $A
    (func (export "answer") (result i32) (i32.const 42))
  )
  (module $B
    (func (import "the" "answer") (result i32))
  )
  (instance $a (instantiate $A))
  (instance $b (instantiate $B (import "the" (instance $a))))
)
```
Here, the two-level import names of core modules are resolved by using the
first name (`"the"`) as a named parameter supplied by `instantiate` and the
second name (`"answer"`) as the name of an export.

The static validation rules for `instantiate` ensure that:
* every `name` imported by the instantiated module is supplied exactly once by
  an `import` in the list;
* the indices in the `def-ref`s refer to a preceding definitions of matching
  type.

One reason to have explicit instance definitions is to allow code generators
and developers to construct arbitrary DAGs of module instances where individual
modules can be instantiated multiple times in the DAG. For example, the
following adapter module creates two instances of `$Libc` for two non-[PIC]
modules that would otherwise clobber each other if they shared a linear memory:
```wasm
(adapter module
  (module $Libc
    (memory (export "memory") 1)
    (func (export "malloc") (param i32) (result i32) ...)
  )

  (module $A
    (memory (import "libc" "memory") 1)
    (func (import "libc" "malloc") (param i32) (result i32))
    (func (export "run") ...)
  )
  (instance $libcA (instantiate $Libc))
  (instance $a (instantiate $A (import "libc" (instance $libcA))))

  (module $B
    (memory (import "libc" "memory") 1)
    (func (import "libc" "malloc") (param i32) (result i32))
    (func (export "run") ...)
  )
  (instance $libcB (instantiate $Libc))
  (instance $b (instantiate $B (import "libc" (instance $libcB))))
)
```
This adapter module contains 3 child modules but creates 4 module instances at
instantiation-time. This works because the import string `"libc"` in `$A` and
`$B` is not looked up in a global namespace; it's a named parameter supplied
directly by the outer adapter module.

Lastly, `instance-expr` offers an alternative to `instantiate` for creating
instances: instances can be synthesized by "tupling" together existing
definitions. For example, the following adapter module creates an instance that
pairs together two existing instances:
```wasm
(adapter module
  (module $Inner ...)
  (instance $left (instantiate $Inner))
  (instance $right (instantiate $Inner))
  (instance $pair
    (export "left" (instance $left))
    (export "right" (instance $right))
  )
)
```
Technically, this tupling instance definition is just shorthand for an
auxiliary adapter module that reexports all its imports. This shorthand will
become more evidently useful once other definitions are presented (below).


### Import Definitions

Imports in adapter modules serve the same role as in core modules, but with an
expanded set of importable types that allow whole modules and instances to be
imported:
```
import       ::= (import <name> <deftype>)
deftype      ::= <instancetype>
               | <moduletype>
               | <core:functype>
               | <core:tabletype>
               | <core:memtype>
               | <core:globaltype>
instancetype ::= (instance <id>? (export <name> <deftype>)*)
moduletype   ::= (module <id>? (import <name> <deftype>)* (export <name> <deftype>)*)
```
The `core:`-prefixed productions refer to Core WebAssembly's [`functype`],
[`tabletype`], [`memtype`] and [`globaltype`] productions.

Unlike core modules, adapter module imports do not allow duplicate names.
This is necessary to ensure that the named arguments passed by `instantiate`
unambiguously match a single import.

Also unlike core modules, adapter module imports only have a single name
(whereas core imports have two names). The reason for this is that, with the
ability to import instances, the *exports* of an imported instance take on the
role of the second-level name. Thus, always requiring two names would force
extra, unnecessary names to be invented.

As an example, the following adapter module instantiates its contained core
module with imported `libc` module, matching the second-level name `malloc`
against the declared exports of `libc`:
```wasm
(adapter module $M
  (import "libc" (module $Libc
    (export "malloc" (func (param i32) (result i32)))
  ))
  (module $Core
    (func (import "libc" "malloc") (param i32) (result i32))
    ...
  )
  (instance $libc (instantiate $Libc))
  (instance $core (instantiate $Core (import "libc" (instance $libc))))
)
```
Note that, by importing `libc` as a *module*, instead of as an *instance*, `$M`
is able to create a new instance of `libc` with a fresh, private linear memory.
This allows `$M` to share the *code* of `libc` with other adapter modules
without being forced to share linear memory (which could, e.g., lead to one
module's static global data clobbering another's).

Adapter modules can also import whole instances, as a replacement for two-level
imports. For example:
```wasm
(adapter module $N
  (import "wasi:filesystem" (instance $libc
    (export "path_open" (func ... ))
    (export "read" (func ...))
  ))
  ...
)
```
Here, `$N` imports an instance of the `wasi:filesystem` interface, which means
that *someone else* has already created the instance and `$N` cannot do so
itself.

For the same reason that `instantiate` operates uniformly on modules, imports
do not distinguish between core modules and adapter modules; all that matters
is that the (core|adapter) module have the expected import and export types.
That being said, since adapter modules have a larger available set of
importable types (viz., module and instance types and, later, interface types),
certain signatures will only be implementable by adapter modules in practice.

To deal with the two-level imports of core modules, the Component Model assigns
a module type using single-level imports of instances with the second-level
names as exports. For example, the following core module:
```wasm
(module
  (import "one" "foo" (func))
  (import "two" "bar" (func))
  (import "one" "baz" (func))
  ...
)
```
would be assigned the following module type:
```wasm
(module
  (import "one" (instance
    (export "foo" (func))
    (export "baz" (func))
  ))
  (import "two" (instance
    (export "bar" (func))
  ))
)
```
Since module types do not allow duplicate import or export names, not all core
modules can be assigned a module type. Thus, adapter modules do not allow the
nesting or importing of core modules with duplicate imports.

Module and instance types also support subtyping, so that superfluous imports
and exports can be ignored. For example, the following adapter module
validates, with the superfluous import `a` being ignored by `$N` and the
superfluous export `d` being ignored by `$M`.
```wasm
(adapter module
  (adapter module $M
    (import "i" (module
      (import "a" (func))
      (import "b" (func))
      (export "c" (func))
    ))
  )
  (module $N
    (import "b" (func))
    (func (export "c") ...)
    (func (export "d") ...)
  )
  (instance (instantiate $M (import "i" (module $N))))
)
```
Additionally, since imports and exports are matched by name, module and
instance subtyping ignores import/export declaration order.


### Export Definitions

Exports in adapter modules serve the same role as in core modules, but with an
expanded set of exportable types that allow whole modules and instances to be
exported:
```
export ::= (export <name> <def-ref>)
```
(Note: `def-ref` is defined [above](#instance-definitions).)

As an example, the following adapter module implements the WASI filesystem
interface, exporting the instance with the name `wasi:filesystem` to provide a
clear semantic connection between the implementation and the interface.
```wasm
(adapter module
  (module $Core
    (func (export "path_open") ...)
    (func (export "read") ...)
  )
  (instance $core (instantiate $Core))
  (export "wasi:filesystem" (instance $core))
)
```
To show an example of exporting individual functions, memories, tables, etc,
we first need the ability to extract these from core instances via Alias
Definitions.


### Alias Definitions

Alias definitions inject other modules' definitions into the current module's
index spaces. Aliases can target either instance exports or the local
definitions of an outer adapter module:
```
alias        ::= (alias <alias-target>)
alias-target ::= <instanceidx> <name> <alias-name>
               | <outeridx> <idx> <alias-name>
alias-name   ::= (instance <instanceidx>)
               | (module <moduleidx>)
               | (func <funcidx>)
               | (type <typeidx>)
               | (table <tableidx>)
               | (memory <memidx>)
               | (global <globalidx>)
```
As an example of an instance-export alias, the following adapter module
aliases the `f` export of `i` in order to pass it as the `foo` import of `M`:
```wasm
(adapter module
  (import "M" (module $M
    (import "foo" (func))
  ))
  (import "i" (instance $i
    (export "f" (func))
    (export "g" (func))
  ))
  (alias $i "f" (func $f))
  (instance $m (instantiate $M (import "foo" (func $f))))
)
```
As an example of an outer alias, in the following adapter module, `$Inner`
aliases `$Outer`'s `$Libc` module, avoiding the need to manually import it:
```wasm
(adapter module $Outer
  (module $Libc
    ...
  )
  (adapter module $Inner
    (alias $Outer $Libc (module $InnerLibc))
    (instance (instantiate $InnerLibc))
    ...
  )
)
```
Here, `$Outer` and `$Libc` serve as symbolic [de Bruijn indices], ultimately
resolving to a pair of integers: the number of enclosing adapter modules to
skip, and the index of the target definition. In particular, this number can be
0, in which case the outer alias refers to the current module, allowing a
single module to alias its own definitions at multiple indices in the index
space (which can be [useful to tools][Issue-30]).

Adapter module definitions containing outer aliases effectively produce a
module [closure] at instantiation time, including a copy of the outer-aliased
definitions in the module value. Because of the prevalent assumption that
module values are stateless, outer aliases are restricted to only refer to
stateless module and type definitions. (In the future, outer aliases to all
kinds of definitions could be allowed by recording the "statefulness" of the
resulting bound module in the resulting bound module's type.)

Additionally, to maintain the acyclicy of module instantiation, outer aliases
are only allowed to refer to preceding outer definitions. Otherwise, modules
could easily create incoherent (or coherent, but mind-bending) recursive
definitions which would complicate tools and implementations.

With both instance and outer aliases, the alias definition inserts the target
definition into the index space of the `alias-name` (so, e.g.,
`(alias $i "foo" (func $f))` inserts `foo` into the function index space, with
`$f` resolving to the new function index). The target definition kind is
validated to actually match the kind of the index space. After that, the
aliased definition can be used in all the same ways as a normal imported or
internal definition.

For symmetry with [imports][func-import-abbrev], aliases can be written
in an inverted form that puts the definition kind first:
```wasm
(func $f (import "i" "f")) ≡ (import "i" "f" (func $f))  ;; (existing)
(func $g (alias $i "g1"))  ≡ (alias $i "g1" (func $g))   ;; (new)
```

To reduce the syntactic burden in the text format, aliases come with syntactic
sugar for implicitly declaring aliases inline, in the same manner as the Core
WebAssembly's [`typeuse`] sugar.

For instance-export aliases, the inline sugar has the form:
```
(kind <instanceidx> <name>+)
```
where the `<name>+` is a sequence of exports projected from the `<instanceidx>`.
For example, the following snippet using inline alias sugar:
```wasm
(instance $j (instantiate $J (import "f" (func $i "f"))))
(export "x" (func $j "g" "h"))
```
is equivalent to the expanded explicit aliases:
```wasm
(alias $i "f" (func $f_alias))
(instance $j (instantiate $J (import "f" (func $f_alias))))
(alias $j "g" (instance $g_alias))
(alias $g_alias "h" (func $h_alias))
(export "x" (func $h_alias))
```

For outer aliases, the inline sugar is simply the identifier of the outer
definition, resolved using normal lexical scoping rules. For example, the
following adapter module:
```wasm
(adapter module $M
  (module $N ...)
  (adapter module
    (alias $M $N (module $N))
    (instance (instantiate $N))
  )
)
```
can be equivalently written:
```wasm
(adapter module
  (module $N ...)
  (adapter module
    (instance (instantiate $N))
  )
)
```

Using aliases, an adapter module can re-export individual functions:
```wasm
(adapter module
  (module $Core1
    (func (export "foo") ...)
  )
  (instance $core1 (instantiate $Core1))
  (module $Core2
    (func (export "bar") ...)
  )
  (instance $core2 (instantiate $Core2))
  (export "a" (func $core1 "foo"))
  (export "b" (func $core2 "bar"))
)
```
This example uses the inline alias sugar to alias `foo` and `bar`,
exporting them with the names `a` and `b`, resp.

Addititionally, aliases are useful in combination with instance
definitions for being able to synthesize instances from individual
definitions. For example:
```wasm
(adapter module
  (import "x" (instance $x
    (export "foo" (func))
    (export "bar" (func))
  ))
  (instance $y
    (export "a" (func $x "foo"))
    (export "b" (func $x "bar"))
  )
  ...
)
```
Here, the imported instance `x` is transformed into a new instance `y` by
aliasing `x`'s exports and tupling them back together with new names. The new
instance `y` can then be passed to `instantiate` or `export`. In this way,
instances can be easily renamed and recombined via aliases. Due to the
declarativity of Module Linking, all this instance and name manipulation is
resolved at compile time and thus incurs no runtime penalty.


### Type Definitions

Type definitions in adapter modules serve the same role as they do in core
modules, but with an expanded set of definable types that allow module and
instance types. Type definitions allow these compound types to be reused to
avoid duplication.
```
type ::= (type $id <deftype>)
```
(Note: `deftype` is defined [above](#import-definitions).)

For example, using both type and alias definitions, the type of `$Libc`
is deduplicated in the following adapter module:
```wasm
(adapter module
  (type $LibcModule (module
    (export "memory" (memory 1))
    (export "malloc" (func (param i32) (result i32)))
  ))
  (import "Libc" (module $Libc (type $LibcModule)))
  (import "A" (module $A
    (import "Libc" (module $Libc (type $LibcModule)))
    ...
  ))
  (import "B" (module $B
    (import "Libc" (module $Libc (type $LibcModule)))
    ...
  ))
  ...
)
```
Here, `(type ...)` is used to refer to a type definition, inheriting Core
WebAssembly's [`typeuse`] syntax. This adapter module is an example of the
kind of thing that a bundler might produce to link together modules `A` and `B`
sharing common `Libc` code.

Frequently an adapter module will need to define both a module type and an
instance type that is the result of instantiating the module type. To avoid
duplicating the list of exports in both the module and instance types, a
bit of syntactic sugar is added, allowing an instance's exports to be injected
into a module by writing `(export $InstanceType)` (without a `<name>` before
the `$InstanceType`). For example, this sugar could be used to share between
the `Libc` instance and module types:
```wasm
(adapter module
  (type $LibcInstance (instance
    (export "memory" (memory 1))
    (export "malloc" (func (param i32) (result i32)))
  ))
  (type $LibcModule (module
    (export $LibcInstance)
  ))
  ...
)
```
In this example, `$LibcInstance` is the type of already-created instances while
`$LibcModule` is the type of modules that can be instantiated to create
`$LibcInstance`-compatible instances.


## Binary Format Considerations

The basic idea of the binary format for adapter modules is to replicate the
conventions of the core module binary format to the extent possible. At the
top-level, an adapter module has a [preamble][Module Binary] followed by a
sequence of [sections].

To distinguish core modules from adapter modules, the [`version`][Module
Binary] field is split into two `u16` fields: a `version` field and a `layer`
field, where core modules implicitly already have the `layer` set to 0 and
adapter modules would have the `layer` set to 1. Thus the `layer` field would
allow runtimes to load both kinds of modules through a single entry point
(e.g., the existing JS API or ESM-integration), unambiguously determining how
to interpret the bytes by looking at the premable.

Like core modules, different definition kinds are encoded in different binary
section kinds (e.g., "modules" go into a "module section", "instances" go in
an "instance section", etc).

Unlike core modules, adapter module sections can go in any order and there can
be multiple sections of a given kind (e.g., multiple module sections). The
reason for this is to align the binary format order with the definition order,
where any definition can refer back to any preceding definition, so that
acyclicy can be enforced by simply requiring that all referenced indices are
less-than the current definition's index.

When core modules are embedded in adapter modules, this is done by literally
embedding a [core module binary] in-line in the module section. Thus, a
runtime can implement adapter modules in terms of an existing core engine
by recursively invoking the engine's binary decoding logic when a core module
definition is encountered. More generally, just as the Component Model spec is
layered in top of the Core WebAssembly spec, a Component Model implementation
is meant to be layered on top of a Core WebAssembly implementation.

As an example, this adapter module:
```wasm
(adapter module
  (import "a" (instance $a (export "f" (func))))
  (module $M
    (import "a" "f" (func))
    (func (export "f")))
  (instance $m1 (instantiate $M (import "a" (instance $a))))
  (instance $m2 (instantiate $M (import "a" (instance $m1))))
  (alias $m2 "f" (func $f))
  (export "g" (func $f))
)
```
could be encoded with the binary section sequence:
1. Type Section, defining an instance type (for `$a`) and function type (for `f`)
2. Import Section, defining the import `$a`, referencing the instance type in (1)
3. Module Section, defining the module `$M` by embedding the core module binary
   format of `$M` inline
4. Instance Section, defining the instance `$m1`, referencing (3) and (2)
5. Instance Section, defining the instance `$m2`, referencing (3) and (4)
8. Alias Section, adding `f` to the function index space, referencing (5)
9. Export section, exporting `g`, referencing (8)

This repository also contains a [detailed binary format sketch](./Binary.md).


## Web Platform Integration

### JS API

The [JS API] currently provides functions like `WebAssembly.compile()` and
`WebAssembly.compileStreaming()` which take raw bytes from an `ArrayBuffer` or
`Response` object and compile them into core modules that are wrapped in a
`WebAssembly.Module` objects.

To natively support the Component Model, the JS API would be extended to allow
the existing JS API functions (`WebAssembly.compile()` et al) to accept adapter
module binaries. As described [above](#binary-format-considerations), an engine
can determine whether a given binary is a core or adapter module by examining
the `layer` field in the first 8 bytes.

Since adapter modules have the same outward-facing structure as core modules,
the result of loading an adapter module with the JS API would just be a
`WebAssembly.Module` object. One area where the JS API would have to be
extended is in the [*read the imports*] logic, to support single-level imports,
instance imports and module imports. There is a fairly natural exension for all
of these features. For example, the adapter module:
```wasm
;; a.wasm
(adapter module
  (import "one" (func))
  (import "two" (instance
    (export "three" (instance
      (export "four" (module
        (import "five" (func))
        (export "six" (func))
      ))
    ))
  ))
)
```
could be successfully instantiated via:
```js
WebAssembly.instantiateStreaming(fetch('./a.wasm'), {
  one: () => (),
  two: {
    three: {
      four: new WebAssembly.Module(code)
    }
  }
});
```
where `instantiateStreaming` checks that the module created from `code` exports
a function `six` (and *may* import a function `five`).

Additionally, the JS API `WebAssembly.Module.imports()` and `exports()`
functions would need to be extended to include the new instance and module
types in the `kind` fields.

Lastly, considering the new exportable types, a module export would naturally
produce a `WebAssembly.Module` object. For an instance export, the JavaScript
correspondence already established by [ESM-integration] is a [Namespace Object].


### ESM-integration

Like the JS API, [ESM-integration] can be extended to load adapter module
binaries in all the same places where core module binaries can be loaded today,
branching on the `layer` field in the binary format to determine the kind of
module to decode.

As with the JS API, the main question for ESM-integration is how to deal with
all imports having a single string as well as the new instance and module
import/export types. Going through these:

For adapter module imports of module type, we need a fundamentally new way to
request that the ESM loader parse or decode a module without *also*
instantiating that module. Recognizing this same need from JavaScript, there is
already a TC39 proposal called [Import Reflection] that adds the ability to
write, in JavaScript:
```js
import Foo from "./foo.wasm" as "wasm-module";
assert(Foo instanceof WebAssembly.Module);
```
With this extension to JavaScript and the ESM loader, an adapter module import
of module type can be treated like an `as "wasm-module"` import by
ESM-integration.

In all other cases, the (single) string imported by an adapter module import is
first resolved to a [Module Record] using the same process as resolving the
[Module Specifier] of a JavaScript `import`. After this, the handling of the
imported Module Record is determined by the import type:

For imports of instance type, the ESM loader would treat the exports of the
instance type as if they were the [Named Imports] of a JavaScript `import`.
Thus, single-level imports of instance type mirror the behavior of core
two-level imports with existing ESM-integration. Since the exports of an
instance type can themselves be instance types, this process must be performed
recursively.

Otherwise, the import is treated like a JavaScript [Imported Default Binding]
and the Module Record is converted to its default value. This allows, for
example, a single level import of a function:
```wasm
(adapter module
  (import "./foo.js" (func (result i32)))
)
```
to be satisfied by a JavaScript module via ESM-integration:
```js
// foo.js
export default () => 42;
```
Lastly, for exports, ESM-integration would produce the same JavaScript
objects for exports as described above for the JS API.



[Component Model]: ../../high-level
[Component Model use cases]: ../../high-level/UseCases.md
[Design Choices]: ../../high-level/Choices.md
[Shared-Nothing]: ../../high-level/Choices.md

[Interface Types]: https://github.com/WebAssembly/interface-types

[ESM-integration]: https://github.com/WebAssembly/esm-integration
[Namespace Object]: https://tc39.es/ecma262/multipage/reflection.html#sec-module-namespace-objects
[Import Reflection]: https://github.com/tc39-transfer/proposal-import-reflection
[Module Record]: https://tc39.es/ecma262/#sec-abstract-module-records
[Module Specifier]: https://tc39.es/ecma262/multipage/ecmascript-language-scripts-and-modules.html#prod-ModuleSpecifier
[Named Imports]: https://tc39.es/ecma262/multipage/ecmascript-language-scripts-and-modules.html#prod-NamedImports
[Imported Default Binding]: https://tc39.es/ecma262/multipage/ecmascript-language-scripts-and-modules.html#prod-ImportedDefaultBinding

[Core Concepts]: https://webassembly.github.io/spec/core/intro/overview.html#concepts
[`typeuse`]: https://webassembly.github.io/spec/core/text/modules.html#type-uses
[Index Space]: https://webassembly.github.io/spec/core/syntax/modules.html#indices
[`module`]: https://webassembly.github.io/spec/core/text/modules.html#text-module
[`import`]: https://webassembly.github.io/spec/core/text/modules.html#imports
[`functype`]: https://webassembly.github.io/spec/core/text/types.html#function-types
[`tabletype`]: https://webassembly.github.io/spec/core/text/types.html#table-types
[`memtype`]: https://webassembly.github.io/spec/core/text/types.html#memory-types
[`globaltype`]: https://webassembly.github.io/spec/core/text/types.html#global-types
[func-import-abbrev]: https://webassembly.github.io/spec/core/text/modules.html#text-func-abbrev
[Module Binary]: https://webassembly.github.io/spec/core/binary/modules.html#binary-module
[Sections]: https://webassembly.github.io/spec/core/binary/modules.html#sections
[Module Instantiation]: https://webassembly.github.io/spec/core/appendix/embedding.html#mathrm-module-instantiate-xref-exec-runtime-syntax-store-mathit-store-xref-syntax-modules-syntax-module-mathit-module-xref-exec-runtime-syntax-externval-mathit-externval-ast-xref-exec-runtime-syntax-store-mathit-store-xref-exec-runtime-syntax-moduleinst-mathit-moduleinst-xref-appendix-embedding-embed-error-mathit-error
[JS API]: https://webassembly.github.io/spec/js-api/index.html
[`WebAssembly.instantiate()`]: https://webassembly.github.io/spec/js-api/index.html#dom-webassembly-instantiate-moduleobject-importobject
[*read the imports*]: https://webassembly.github.io/spec/js-api/index.html#read-the-imports

[Statically link]: https://en.wikipedia.org/wiki/Static_library
[Native Dynamic Linking]: https://en.wikipedia.org/wiki/Dynamic_loading
[Principle of Least Authority]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
[Virtualize]: https://en.wikipedia.org/wiki/Virtualization
[Mocking]: https://en.wikipedia.org/wiki/Mock_object
[De Bruijn Indices]: https://en.wikipedia.org/wiki/De_Bruijn_index
[Closure]: https://en.wikipedia.org/wiki/Closure_(computer_programming)
[PIC]: https://en.wikipedia.org/wiki/Position-independent_code

[Figma plugins]: https://www.figma.com/blog/an-update-on-plugin-security/
[Attenuate]: http://cap-lore.com/CapTheory/Patterns/Attenuation.html

[Issue-30]: https://github.com/WebAssembly/module-linking/issues/30
