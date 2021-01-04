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

Three new sections are added

* Module section, `modulesec`, defines nested modules.
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
            0x60 ft:functype                     ->       ft
            0x61 mt:moduletype                   ->       mt
            0x62 it:instancetype                 ->       it

functype     ::= rt_1:resulttype rt_2:resulttype ->       rt_1 -> rt-2
moduletype   ::= mtd*:vec(moduletypedef)         ->       mtd*
instancetype ::= itd*:vec(instancetypedef)       ->       itd*

moduletypedef ::=
            itd                                  ->       itd
            0x02 i:import                        ->       {import i}

instancetypedef ::=
            0x01 t:type                          ->       {type t}
            0x07 et:exporttype                   ->       {export et}
            0x0f a:alias                         ->       {alias a}

exporttype ::= nm:name d:importdesc              ->       {name nm, desc d}
```

Note: the numeric discriminants in `moduletypedef` are chosen to match the
section numbers for the corresponding sections.

Referencing:
* [`resulttype`](https://webassembly.github.io/spec/core/binary/types.html#binary-resulttype)
* [`import`](https://webassembly.github.io/spec/core/binary/modules.html#binary-import)
* [`importdesc`](https://webassembly.github.io/spec/core/binary/modules.html#binary-importdesc) -
  although note that this is updated below as well.

**Validation**

* The `moduletypedef`s of a `moduletype` are validated in a fresh set of index
  spaces (currently, only the type index space is used). Thus, the function
  type of a function export must either be defined inside the `moduletype` or
  aliased. The same goes for `instancetypedef`s and `instancetype`s.
* Types can only reference preceding type definitions. This forces everything to
  be a DAG and forbids cyclic types. TODO: with type imports we may need cyclic
  types, so this validation will likely change in some form.

## Import Section updates

Updates to
[`importdesc`](https://webassembly.github.io/spec/core/binary/modules.html#binary-importdesc)

```
# note that in MVP wasm this encoding specifies a zero-length `name` for the
# second import string, but the following `0xff` byte is not a valid
# `importdesc` prefix, so this encoding is invalid in MVP wasm
import ::=
    ...
    nm:name 0x00 0xff d:importdesc              ->    {name nm, desc d}

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
modulesec ::=  m*:section_14(vec(modulecode))        ->        m*

modulecode ::= size:u32 mod:module                      ->    mod, size = ||mod||
```

Note that this is intentionally a recursive production where `module` refers to
the top-level
[`module`](https://webassembly.github.io/spec/core/binary/modules.html#binary-module)

**Validation**

* This section adds nested modules to the parent module's module index space.
* Each nested `module` is validated with the parent's context (types and module
  index spaces) at the point of the nested `module` in the byte stream.

## Instance section (15)

A new section defining local instances

```
instancesec ::=  i*:section_15(vec(instancedef))     ->    i*

instancedef ::= 0x00 m:moduleidx arg*:vec(export)     ->    {instantiate m, imports arg*}
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
* The `arg*` list is validated against `m`'s imports according to
  [module subtyping](Subtyping.md#instantiation) rules.


## Alias section (16)

A new module section is added

```
aliassec ::=  a*:section_16(vec(alias))     ->        a*

alias ::=
    0x00 i:instanceidx 0x00 nm:name         ->        (alias (func $i "nm"))
    0x00 i:instanceidx 0x01 nm:name         ->        (alias (table $i "nm"))
    0x00 i:instanceidx 0x02 nm:name         ->        (alias (memory $i "nm"))
    0x00 i:instanceidx 0x03 nm:name         ->        (alias (global $i "nm"))
    0x00 i:instanceidx 0x05 nm:name         ->        (alias (module $i "nm"))
    0x00 i:instanceidx 0x06 nm:name         ->        (alias (instance $i "nm"))
    0x01 ct:varu32 0x05 m:moduleidx         ->        (alias (module outer ct m))
    0x01 ct:varu32 0x07 t:typeidx           ->        (alias (type outer ct t))
```

**Validation**

* The instance, module and type indices are all in bounds. Remember that "in bounds"
  means only those definitions preceding this alias definition in binary format
  order. In the case of outer aliases, this means the position of the nested module
  definition in the outer module.
* Aliased instance export names must be found in `i`'s instance type with a
  matching definition kind.
* The `ct` of an outer alias must be *less than* the number of enclosing modules
  which implicitly prevents outer aliases from appearing in top-level modules.

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

