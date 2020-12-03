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
  i*:vec(import) e*:vec(exporttype)             ->        {imports i*, exports e*}

instancetype ::= e*:vec(exporttype)             ->        {exports e*}

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
  be a DAG and forbids cyclic types. TODO: with type imports we may need cyclic
  types, so this validation will likely change in some form.

## Import Section updates

Updates to
[`importdesc`](https://webassembly.github.io/spec/core/binary/modules.html#binary-importdesc)

```
# note that in MVP wasm this encoding specifies a zero-length field name, but
# the following `0xff` byte is not a valid `importdesc` prefix, so this encoding
# is invalid in MVP wasm
import ::=
    ...
    mod:name 0x00 0xff d:importdesc             ->    {module mod, desc d}

importdesc ::=
    ...
    0x05 x:typeidx                              ->    module x
    0x06 x:typeidx                              ->    instance x
```

**Validation**

* the `func x` production ensures the type `x` is indeed a function type
* the `module x` production ensures the type `x` is indeed a module type
* the `instance x` production ensures the type `x` is indeed an instance type

## Module section (14)

A new module section is added

```
modulesec ::=  t*:section_14(vec(typeidx))           ->        t*
```

**Validation**

* Each type index `t` points to a module type (e.g. not a function type)
* This defines the locally defined set of modules, adding to the module index
  space.

## Instance section (15)

A new section defining local instances

```
instancesec ::=  i*:section_15(vec(instancedef))     ->    i*

instancedef ::= 0x00 m:moduleidx e*:vec(exportdesc)   ->    {instantiate m, imports e*}
```

This defines instances in the module, appending to the instance index space.
Each instance definition declares the module it's instantiating as well as the
items used to instantiate that instance. Note that the leading 0x00 is intended
to allow different forms of instantiation in the future if added. This is also a
placeholder value for now since if an `instantiate` instruction will be added in
the future we'll likely want this binary value to match that.

**Validation**

* The module index `m` must be in bounds.
* Indices of items referred to by `exportdesc` are all in-bounds. Can only refer
  to imports/previous aliases, since only those sections can precede.
* The `e*` list is validated against the module type's declared list
  of [imports pairwise and in-order](Explainer.md#module-imports-and-nested-instances).
  The type of each item must be a subtype of the expected type listed in the
  module's type.

**Execution notes**

* The actual module being instantiated does not need to list imports in the
  exact same order as its type declaration. The `e*` has names based on the
  local module type's declaration.
* Be sure to read up on [subtyping](./Subtyping.md) to ensure that instances
  with a single name can be used to match imports with a two-level namespace.

## Alias section (16)

A new module section is added

```
aliassec ::=  a*:section_16(vec(alias))     ->        a*

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

* Aliased instance indexes are all in bounds. Remember "in bounds" here means it
  can't refer to instances defined after the `alias` item.
* Aliased instance export indices are in bounds relative to the instance's
  *locally-declared* (via module or instance type) list of exports.
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

**Execution notes**

* Note for child aliases that while the export is referred to by index it's
  actually loaded from the specified instance by name. The name loaded
  corresponds to the `i`th export's name in the locally defined type.

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

## Module Code Section (17)

```
modulecodesec ::= m*:section_17(vec(modulecode))       ->    m*

modulecode ::= size:u32 mod:module                      ->    mod, size = ||mod||
```

Note that this is intentionally a recursive production where `module` refers to
the top-level
[`module`](https://webassembly.github.io/spec/core/binary/modules.html#binary-module)

**Validation**

* Module definitions must match their module type exactly, no subtyping (or
  maybe subtyping, see WebAssembly/module-linking#9).
* Modules themselves validate recursively.
* The module code section must have the same number of modules as the count of
  all local module sections.
* Each submodule is validated with the parent's context at the time of
  declaration. For example the set of types and modules the current module
  has defined are available for aliasing in the submodule, but only those
  defined before the corresponding module's type declaration.
