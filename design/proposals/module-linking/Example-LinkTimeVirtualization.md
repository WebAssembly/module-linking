# Link-Time Virtualization Example

This document walks through the 
[Link-time Virtualization](Explainer.md#link-time-virtualization)
use case.

We start with a child module that has been written and compiled separately,
without regard to the parent module. It imports `wasi:filesystem` to do file I/O:
```wasm
(module $Child
  (import "wasi:filesystem" "read" (func (param i32 i32 i32) (result i32)))
  (import "wasi:filesystem" "write" (func (param i32 i32 i32) (result i32)))
  (func $play (export "play")
    ...
  )
)
```

We want to write a parent module that reuses the child module, but we don't
want to give it real file operations, but rather some virtual filesystem.
This virtual filesystem can be factored out and reused as a separate module
that imports the "real" `wasi:filesystem` and exports a virtualized
implementation of all the `wasi:filesystem` operations:
```wasm
(module $VirtualFS
  (import "wasi:filesystem" "read" (func (param i32 i32 i32) (result i32)))
  (import "wasi:filesystem" "write" (func (param i32 i32 i32) (result i32)))

  (func (export "read") (param i32 i32 i32) (result i32)
    ...
  )
  (func (export "write") (param i32 i32 i32) (result i32)
    ...
  )
)
```

We can now write the parent module as an adapter module that composes `$Child`
and `$VirtualFS`:
```wasm
(adapter module $Parent
  (type $FSInstance (instance
    (export "read" (func (param i32 i32 i32) (result i32)))
    (export "write" (func (param i32 i32 i32) (result i32)))
  ))
  (import "wasi:filesystem" (instance $real-fs (type $FSInstance)))

  (import "./virtualize.wasm" (module $VirtualFS
    (import "wasi:filesystem" (instance (type $FSInstance)))
    (export (type $FSInstance))
  ))
  (import "./child.wasm" (module $Child
    (import "wasi:filesystem" (instance (type $FSInstance)))
    (export "play" (func))
  ))

  (instance $virt-fs (instantiate $VirtualFS (import "wasi:filesystem" (instance $real-fs))))
  (instance $child (instantiate $Child (import "wasi:filesystem" (instance $virt-fs))))
)
```
This example demonstrates several features of the Module Linking proposal:
* The first line of the module is a [type definition](Explainer.md#type-definitions),
  factoring out an instance type so it can be reused four times below.
* The next line is an [import definition](Explainer.md#import-definitions),
  importing an instance of `wasi:filesystem`. This instance import is logically
  equivalent to two two-level imports in a core module with `wasi:filesystem` as the
  "module" name and `read`/`write` as the "field" name.
* The following two imports import *modules*, which is the other new importable
  type (after instances).
* The final two lines are [instance definitions](Explainer.md#instance-definitions),
  instantiating and wiring up the `VirtualizeFS` and `Child` modules.

Here, we assume that the host understands relative file paths. To remove this host
assumption, a tool could replace the imports definitions in `$Parent` with
[module definitions](Explainer.md#module-definitions):
```wasm
(adapter module $Parent
  (type $FSInstance         ... same as above)
  (import "wasi:filesystem" ... same as above)

  (module $VirtualizeFS     ... copied inline)
  (module $Child            ... copied inline)

  (instance $virt-fs        ... same as above)
  (instance $child          ... same as above)
)
```

In general, a tool can trivially go between nested modules and module imports
without any understanding of their contents which allows bundling tools to
optimize for the deployment target. For example, on the Web, the choice to
import vs. nest modules can be based on the usual bundling considerations of
caching, reuse and minimizing fetches.
