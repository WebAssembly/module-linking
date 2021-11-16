# Module Linking Binary Format (sketch)

This document presents the binary format of the adapter modules in the
[attribute grammar] style of the Core WebAssembly [text format]. Additionally,
key validation rules are roughly sketched out. For explanations of what the
different AST productions mean, consult the respective sections in the
[explainer](Explainer.md). The overall plan for the binary format is
described in the explainer's [Binary Format Considerations section](Explainer.md#binary-format-considerations).


## Module Definitions

(See [module definitions](Explainer.md#module-definitions) in the explainer.)
```
magic            ::= 0x00 0x61 0x73 0x6D
adapter-version  ::= 0x0a 0x00 0x01 0x00
adapter-preamble ::= <magic> <adapter-version>
adapter-module   ::= <adapter-preamble> s*:<section>* => (adapter module flatten(s*))
section          ::= t*:section_1(vec(<type>))        => t*
                   | i*:section_2(vec(<import>))      => i* 
                   | m*:section_3(vec(<module>))      => m*
                   | i*:section_4(vec(<instance>))    => i*
                   | a*:section_5(vec(<alias>))       => a*
                   | e*:section_6(vec(<export>))      => e*
module           ::= size:u32 m:<core:module>         => m   (if size = ||m||)
                   | size:u32 m:<adapter-module>      => m   (if size = ||m||)
```
Notes:
* The first `u16` of `adapter-version` is the pre-release version, `0xa`. The
  idea is that, before final standardization, this pre-release version will be
  bumped to coordinate spec changes in the prototypes. When the standard is
  finalized, the version will be changed one last time to `0x1`. This mirrors
  what happened during the path to Core WebAssembly 1.0.
* The second `u16` of `adapter-version` is the "module kind" which is `0x1` for
  adapter modules and `0x0` for core modules.
* The `section_X`'s refer to Core WebAssembly's [`section`] parameterized rule.
  The `X` numbers (and all other magic constants in the adapter module binary
  format) do not attempt to be distinct from or align with the core module
  binary format since, over time, this distinction/alignment will be lost as
  both formats may grow independently.
* `core:module` refers to Core WebAssembly's [`module`] rule.

One general note with respect to the binary format and validation is that each
definition is validated in an environment defined containing only the preceding
definitions. Thus, the index spaces are built and validated incrementally as
each definition of the module is decoded and validated. For validation
purposes, the `section` boundaries are not significant; all that matters is the
individual definitions (`type`, `import`, `export`, `module`, `instance` and
`alias`) contained within the sections.


## Instance Definitions

(See [instance definitions](Explainer.md#instance-definitions) in the explainer.)
```
instance      ::= ie:<instance-expr>                      => (instance ie)
instance-expr ::= 0x00 m:<moduleidx> nd*:vec(<named-def>) => (instantiate m (import nd)*)
                | 0x01 nd*:vec(<named-def>)               => (export nd)*
named-def     ::= nm:<name> dr:<def-ref>                  => nm dr
def-ref       ::= 0x00 x:<instanceidx>                    => (instance x)
                | 0x01 x:<moduleidx>                      => (module x)
                | 0x02 x:<funcidx>                        => (func x)
                | 0x03 x:<tableidx>                       => (table x)
                | 0x04 x:<memidx>                         => (memory x)
                | 0x05 x:<globalidx>                      => (global x)
```
Notes:
* The indices in the `def-ref`s are validated according to the respective index
  space. As mentioned above, index spaces are built incrementally as each
  definition is validated.
* The `import` arguments supplied by `instantiate` are validated against the
  module `m` according to [module subtyping](Subtyping.md#instantiation) rules.
* The `def-ref` codes are chosen to keep the shared-nothing kinds contiguous
  (with room for future additions by Interface Types) and separate from the 
  shared-everything core module kinds.
* `name` refers to the Core WebAssembly's [`name`] rule.


## Import Definitions

(See [import definitions](Explainer.md#import-definitions) in the explainer.)
```
import   ::= nm:<name> dt:<def-type> => (import nm dt)
def-type ::= 0x00 x:typeidx          => (instance x)
           | 0x01 x:typeidx          => (module x)
           | 0x02 x:typeidx          => (func x)
           | 0x03 tt:tabletype       => (table tt)
           | 0x04 mt:memtype         => (memory mt)
           | 0x05 gt:globaltype      => (global gt)
```
Notes:
* Unlike the text format, which allows module/instance/function types to be
  written "inline", the binary format requires these compound types be defined
  separately as a type definition, referred to be type index.
* The `nm` are validated to be unique; duplicate import names are not allowed
  in adapter modules.
* The `typeidx` are validated against the type index space to match the
  declared definition kind.


## Export Definitions

(See [export definitions](Explainer.md#export-definitions) in the explainer.)
```
export ::= nm:<name> dr:<def-ref> => (export nm dr)
```
Notes:
* As with import definitions, the `nm` are validated to be unique.
* As with instance definitions, the indices of `dt` are validated to be
  be valid in the associated index space at the point of the export.


## Alias Definitions

(See [alias definitions](Explainer.md#alias-definitions) in the explainer.)
```
alias ::= 0x00 i:<instanceidx> nm:<name> 0x00 => (alias i nm (instance))
        | 0x00 i:<instanceidx> nm:<name> 0x01 => (alias i nm (module))
        | 0x00 i:<instanceidx> nm:<name> 0x02 => (alias i nm (func))
        | 0x00 i:<instanceidx> nm:<name> 0x03 => (alias i nm (table))
        | 0x00 i:<instanceidx> nm:<name> 0x04 => (alias i nm (memory))
        | 0x00 i:<instanceidx> nm:<name> 0x05 => (alias i nm (global))
        | 0x01 ct:<varu32> i:<moduleidx> 0x01 => (alias outer ct i (module))
        | 0x01 ct:<varu32> i:<typeidx>   0x06 => (alias outer ct i (type))
```
Notes:
* For instance-export aliases, `i` is validated to refer to an instance in the
  instance index space that actually exports `nm` with the specified definition
  kind.
* For outer aliases, `ct` is validated to be *less than* the number of enclosing
  modules and `i` is validated to be a valid index in the specified
  definition's index space of the enclosing adapter module indicated by `ct`
  (counting outward, starting with `0` referring to the immediately-enclosing
  adapter module).


## Type Definitions

(See [type definitions](Explainer.md#type-definitions) in the explainer.)
```
type               ::= 0x7f itd*:vec(<instance-type-decl>)        -> (instance itd*)
                     | 0x7e mtd*:vec(<module-type-decl>)          -> (module mtd*)
                     | 0x7d p*:vec(<val-type>) r*:vec(<val-type>) -> (func (param p)* (result r)*)
instance-type-decl ::= 0x01 t:<type>                              -> t
                     | 0x05 a:<alias>                             -> a
                     | 0x06 nm:<name> dt:<def-type>               -> (export nm dt)
module-type-decl   ::= itd:<instance-type-decl>                   -> itd
                     | 0x02 i:<import>                            -> i
val-type           ::= 0x00 vt:<core:valtype>                     -> vt
```
Notes:
* Instance and modules types create a fresh type index spaces that are
  populated and referenced by their contents. E.g., for a module type that
  imports a function, the `import` `module-type-decl` must be preceded by
  either a `type` or `alias` `module-type-decl` that adds the function type to
  the type index space.
* Currently, the only allowed form of `alias` in instance and module types
  is `(alias outer ct li (type))`. In the future, other kinds of aliases
  will be needed and this restriction will be relaxed.
* To avoid redefining the whole binary format of Core WebAssembly's [`valtype`]
  and to allow implementation reuse, the binary format for adapter modules'
  `val-type` embeds `core:valtype`. A `0x00` prefix is used so that future
  additions to `core:valtype` cannot conflict with future additions to
  `val-type` (viz., in Interface Types).
* The normal index space validation rules for adapter modules described above
  ensure that module and instance type definitions are acyclic.



[Attribute Grammar]: https://en.wikipedia.org/wiki/Attribute_grammar
[Text Format]: https://webassembly.github.io/spec/core/text/conventions.html
[`section`]: https://webassembly.github.io/spec/core/binary/modules.html#sections
[`module`]: https://webassembly.github.io/spec/core/binary/modules.html#binary-module
[`name`]: https://webassembly.github.io/spec/core/binary/values.html#binary-name
[`valtype`]: https://webassembly.github.io/spec/core/binary/types.html#value-types
