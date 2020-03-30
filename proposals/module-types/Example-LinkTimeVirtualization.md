# Link-Time Virtualization Example

This document walks through the 
[Link-time Virtualization](Explainer.md#link-time-virtualization)
use case.

We start with a child module that has been written and compiled separately,
without regard to the parent module. It imports `wasi_file` to do file I/O:

```wasm
;; child.wat
(module
  (type $WasiFile (instance
    (export "read" (func $read (param i32 i32 i32) (result i32)))
    (export "write" (func $write (param i32 i32 i32) (result i32)))
  ))
  (import "wasi_file" (instance $wasi-file (type $WasiFile)))
  (func $play (export "play")
    ...
    call $wasi-file.$read
    ...
  )
)
```

We want to write a `parent` module that reuses `child`, but we don't want to
give it real file operations, but rather some file operations we virtualized.
For example, maybe we want an adapter that automatically compresses on write
and decompresses on read. This adapter module can be factored out and reused
as a separate module that imports a "real" `wasi_file` instance and uses it
to implement a new virtualized `wasi_file` instance:

```wasm
;; virtualize.wat
(module
  (import "wasi_file" (instance $wasi-file (type $WasiFile)))

  (func (export "read") (param i32 i32 i32) (result i32)
    ... impl
  )
  (func (export "write") (param i32 i32 i32) (result i32)
    ... impl
  )
)
```

We can now write the parent module by composing `virtualize.wasm` and
`child.wasm` appropriately:

```wasm
;; parent.wat
(module
  (type $WasiFile (instance
    (export "read" (func $read (param i32 i32 i32) (result i32)))
    (export "write" (func $write (param i32 i32 i32) (result i32)))
  ))
  (import "wasi_file" (instance $real-wasi (type $WasiFile)))

  (import "./virtualize.wasm" (module $VIRTUALIZE
    (import "wasi_file" (instance (type $WasiFile)))
    (exports $WasiFile)
  ))
  (import "./child.wasm" (module $CHILD
    (import "wasi_file" (instance (type $WasiFile)))
    (export "play" (func))
  ))

  (instance $virt-wasi (instance.instantiate $VIRTUALIZE (ref.instance $real-file)))
  (instance $child (instance.instantiate $CHILD (ref.instance $virt-wasi)))

  (func (export "work")
    ...
    call $child.$play
    ...
  )
)
```

Here, we assume the host understands relative file paths, but we could also bundle
all 3 files into one compound `parent-bundle.wasm`:

```wasm
;; parent-bundled.wat
(module
  (type $WasiFile     ...same as above)
  (import "wasi_file" ...same as above)

  (module $VIRTUALIZE ... copied inline)
  (module $CHILD      ... copied inline)

  (instance $virt-wasi ...same as above)
  (instance $child     ...same as above)

  (func (export "work") ...same as above)
)
```

In general, a tool can trivially transform between nested modules and module
imports without any understanding of their contents which allows bundling tools
to optimize for the deployment target. For example, on the Web, the choice to
import vs. nest modules can be based on the usual bundling considerations of
caching, reuse and minimizing fetches.
