# Module Linking Binary Format

This document is intended to be a reference for the current state of the binary
encoding of the module linking proposal. This is kept in sync with [the
explainer](Explainer.md) and that is also recommended reading to understand this
document. Currently this isn't striving to have a spec-level of detail, but
hopefully it should be enough to get implementations off the ground and agreeing
with each other!

## Module-level updates

At a high-level two new index spaces are added: modules and instances. Index
spaces are also updated to be built incrementally as they are defined.

Four new sections are added

* Module section, `modulesec`, declares nested modules whose code comes
  later. Only types are mentioned in this section.
* Module Code section, `modulecodesec`, defines the nested modules.
* Instance section, `instancesec`, declares and defines local instances.
* Alias section, `aliassec`, brings in exports from nested instances or items
  from the parent module into the local index space.

The Type, Import, Module, Instance, and Alias sections can appear in any order
at the beginning of a module. Each item defined in these sections can only refer
to previously defined items. For example instance 4 cannot reference instance 5,
and similarly no instance can refer to the module's own functions. Similarly
instance 4 can only access module 3 if module 3 precedes instance 4 in binary
format order.

## Type Section updates

Updates to
[`typesec`](https://webassembly.github.io/spec/core/binary/modules.html#binary-typesec):

```
typesec ::=  t*:section_1(vec(type))

type ::=
            0x60 ft:functype                    ->        ft
            0x61 mt:moduletype                  ->        mt
            0x62 it:instancetype                ->        it

functype ::= rt_1:resulttype rt_2:resulttype    ->        rt_1 -> rt-2

moduletype ::=
  i*:vec(import) e*:vec(exporttype)             ->        {imports i, exports e}

instancetype ::= e*:vec(exporttype)             ->        {exports e}

exporttype ::= nm:name d:importdesc             ->        {name nm, desc d}
```

referencing:

* [`resulttype`](https://webassembly.github.io/spec/core/binary/types.html#binary-resulttype)
* [`import`](https://webassembly.github.io/spec/core/binary/modules.html#binary-import)
* [`importdesc`](https://webassembly.github.io/spec/core/binary/modules.html#binary-importdesc) -
  although note that this is updated below as well.

**Validation**

* each `importdesc` is valid according to import section
* Types can only reference preceding type definitions. This forces everything to
  be a DAG and forbids cyclic types.

## Import Section updates

Updates to
[`importdesc`](https://webassembly.github.io/spec/core/binary/modules.html#binary-importdesc)

```
# note that `0x01 0xc0` in MVP-wasm specifies a 1-byte module field of the
# string 0xc0, but the 0xc0 byte is not valid utf-8, so this was not a valid
# MVP import
import ::=
    ...
    mod:name  0x01 0xc0 d:importdesc            ->    {module mod, desc d}

importdesc ::=
    ...
    0x05 x:typeidx                              ->    module x
    0x06 x:typeidx                              ->    instance x
```

**Validation**

* the `func x` production ensures the type `x` is indeed a function type
* the `module x` production ensures the type `x` is indeed a module type
* the `instance x` production ensures the type `x` is indeed an instance type

## Module section (100)

A new module section is added

```
modulesec ::=  t*:section_100(vec(typeidx))           ->        t*
```

**Validation**

* Each type index `t` points to a module type (e.g. not a function type)
* This defines the locally defined set of modules, adding to the module index
  space.

## Instance section (101)

A new section defining local instances

```
instancesec ::=  i*:section_101(vec(instancedef))     ->    i*

instancedef ::= 0x00 m:moduleidx e*:vec(exportdesc)   ->    {instantiate m, imports e}
```

This defines instances in the module, appending to the instance index space.
Each instance definition declares the module it's instantiating as well as the
items used to instantiate that instance. Note that the leading 0x00 is intended
to allow different forms of instantiation in the future if added. This is also a
placeholder value for now since if an `instantiate` instruction will be added in
the future we'll likely want this binary value to match that.

**Validation**

* The type index `m` must point to a module type.
* Indices of items referred to by `exportdesc` are all in-bounds. Can only refer
  to imports/previous aliases, since only those sections can precede.
* The precise rules of how `e*` is validated against the module type's declare
  list of imports is being hashed out in
  [#7](https://github.com/WebAssembly/module-linking/issues/7). For now
  conservative order-dependent rules are used where the length of `e*` must be
  the same as the number of imports the module type has. Additionally the type
  of each element of `e*` must be a subtype of the type that it's being matched
  with. Matching happens pairwise with the list of imports on the module type
  for `m`.

## Alias section (102)

A new module section is added

```
aliassec ::=  a*:section_102(vec(alias))     ->        a*

alias ::=
    0x00 i:instanceidx 0x00 e:exportidx      ->       (alias (instance $i) (func $e))
    0x00 i:instanceidx 0x01 e:exportidx      ->       (alias (instance $i) (table $e))
    0x00 i:instanceidx 0x02 e:exportidx      ->       (alias (instance $i) (memory $e))
    0x00 i:instanceidx 0x03 e:exportidx      ->       (alias (instance $i) (global $e))
    0x00 i:instanceidx 0x05 e:exportidx      ->       (alias (instance $i) (module $e))
    0x00 i:instanceidx 0x06 e:exportidx      ->       (alias (instance $i) (instance $e))
    0x01 0x05 m:moduleidx                    ->       (alias parent (module $m))
    0x01 0x07 t:typeidx                      ->       (alias parent (type $t))
```

**Validation**

* Aliased instance indexes are all in bounds
* Aliased instance export indices are in bounds relative to the instance's
  *locally-declared* (via module or instance type) list of exports
* Export indices match the actual type of the export
* Aliases append to the respective index space.
* Parent aliases can only happen in submodules (not the top-level module) and
  the index specifies is the entry, in the parent's raw index space, for that
  item.
* Parent aliases can only refer to preceeding module/type definitions, relative
  to the binary location where the inline module's type was declared. Note that
  when the module code section is defined all of the parent's modules/types are
  available, but inline modules still may not have access to all of these if the
  items were declared after the module's type in the corresponding module
  section.

## Function section

**Validation**

* Type indices must point to function types.

## Export section

update
[`exportdesc`](https://webassembly.github.io/spec/core/binary/modules.html#binary-exportdesc)

```
exportdesc ::=
    ...
    0x05 x:moduleidx                        -> module x
    0x06 x:instanceidx                      -> instance x
```

**Validation**

* Module/instance indexes must be in-bounds.

## Module Code Section (103)

```
modulecodesec ::= m*:section_103(vec(modulecode))       ->    m*

modulecode ::= size:u32 mod:module                      ->    mod, size = ||mod||
```

Note that this is intentionally a recursive production where `module` refers to
the top-level
[`module`](https://webassembly.github.io/spec/core/binary/modules.html#binary-module)

**Validation**

* Module definitions must match their module type exactly, no subtyping (or
  maybe subtyping, see WebAssembly/module-linking#9).
* Modules themselves validate recursively.
* Must have the same number of modules as the count of all local module
  sections.
* Each submodule is validated with a subset of the parent's context, for example
  the set of types and instances the current module has defined are available
  for aliasing in the submodule.

## Subtyping

Subtyping will extend what's currently ["import
matching"](https://webassembly.github.io/spec/core/exec/modules.html#import-matching)

**Instances**

Instance `{exports e1}` is a subtype of `{exports e2}` if and only if:

* Each name in `e1` is present in `e2`
* For each corresponding name `n` in the sets
  * `e1[n]` is a subtype of `e2[n]`

**Instances**

Module `{imports i1, exports e1}` is a subtype of `{imports i2, exports e2}` if and only if:

* Each name in `e1` is present in `e2`
* For each corresponding name `n` in the sets
  * `e1[n]` is a subtype of `e2[n]`
* ... And some condition on imports. For now this is a bit up for debate on
  WebAssembly/module-linking#7
