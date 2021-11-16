# Shared-Everything C-family Dynamic Linking Example

This document walks through the 
[Shared-Everything C-family Dynamic Linking](Explainer.md#shared-everything-c-family-dynamic-linking)
use case. One high-level point to make before starting is that the scheme
outlined below avoids the need for any sort of runtime loader (like `ld.so`).
Rather, Module Linking proposal is used in such a way that the wasm engine
itself ends up serving this purpose. Due to the declarative linking structure
of Module Linking, this allows the wasm engine to more aggressively optimize
across modules.

## `libc`

As with native dynamic linking, shared-everything dynamic linking requires
conventions followed by the toolchains producing the participating modules.
In this convention, `libc` is special in that, conventionally, an
implementation (like [wasi-libc]) is bundled with a compiler (like clang) to
produce an SDK (like [wasi-sdk]).

For our hypothetical dynamic linking convention, we'll have `libc` define and
export a linear memory which will be imported by the rest of the program, along
with `malloc`, `free`, etc. Thus, `$Libc` is roughly shaped:
```wasm
(module $Libc
  (memory (export "memory") 1)
  (func (export "malloc") (param i32) (result i32) ...impl)
  ...
)
```

The SDK also contains standard library headers which contain declarations
compatible with `$Libc`. To implement them, we first need some helper macros,
which we'll conveniently define in `stddef.h`:
```c
/* stddef.h */
#define WASM_IMPORT(module,name) __attribute__((import_module(#module), import_name(#name))) name
#define WASM_EXPORT(name) __attribute__((export_name(#name))) name
```

These macros use the clang-specific attributes [`import_module`], [`import_name`]
and [`export_name`] to ensure that the annotated C functions produce the
correct wasm import or export definitions. Other compilers would use their own
magic syntax to achieve the same effect. Using these macros, we can declare
`malloc` in `stdlib.h`:
```c
/* stdlib.h */
#include <stddef.h>
#define LIBC(name) WASM_IMPORT(libc, name)
void* LIBC(malloc)(size_t n);
```

With these annotations, C programs that include this header will be compiled
to contain an [import definition](Explainer.md#import-definitions):
```wasm
(import "libc" "malloc" (func (param i32) (result i32)))
```

## `libzip`

With `libc`, we have a functional SDK that we can now use to create the shared
library `libzip`. Its C definition has the rough shape:
```c
/* libzip.h */
#include <stddef.h>
#define LIBZIP(name) WASM_IMPORT(libzip, name)
void* LIBZIP(zip)(void* in, size_t in_size, size_t* out_size);
```
```c
/* libzip.c */
#include <stdlib.h>
void* WASM_EXPORT(zip)(void* in, size_t in_size, size_t* out_size) {
  ...
  void *p = malloc(n);
  ...
}
```

Note that `libzip.h` annotates the `zip` declaration with an *import* attribute
so that client modules generate proper wasm *import definitions* while `libzip.c`
annotates the `zip` definition with an *export* attribute so this function generates
a proper wasm *export definition*. Compiling with `clang -shared libzip.c`
produces a `$Libzip` module shaped like:
```wasm
(module $Libzip
  (import "libc" "memory" (memory 1))
  (import "libc" "malloc" (func (param i32) (result i32)))
  (func (export "zip") (param i32 i32 i32) (result i32)
    ...
  )
)
```

## `zipper`

Using `libc` and `libzip`, we can now implement the `zipper` component:
```c
/* zipper.c */
#include <stdlib.h>
#include "libzip.h"
int main(int argc, char* argv[]) {
  ...
  void *in = malloc(n);
  ...
  void *out = zip(in, n, &out_size);
  ...
}
```

When compiled with `clang zipper.c`, we get a `$Zipper` module:
```wasm
(adapter module $Zipper
  (type $LibcInstance (instance
    (export "memory" (memory 1))
    (export "malloc" (func (param i32) (result i32)))
  ))
  (type $LibZipInstance (instance
    (export "zip" (func (param i32 i32 i32) (result i32)))
  ))

  (import "libc" (module $Libc
    (export $LibcInstance)
  ))
  (import "libzip" (module $Libzip
    (import "libc" (instance $LibcInstance))
    (export $LibZipInstance)
  ))

  (module $Core
    (import "libc" "memory" (memory 1))
    (import "libc" "malloc" (func (param i32) (result i32)))
    (import "libzip" "zip" (func (param i32 i32 i32) (result i32)))
    ...
  )

  (instance $libc (instantiate $Libc))
  (instance $libzip (instantiate $Libzip
    (import "libc" (instance $libc))
  ))
  (instance $core (instantiate $Core
    (import "libc" (instance $libc))
    (import "libzip" (instance $libzip))
  ))
)
```

Note that `$Zipper` imports `$Libc` and `$Libzip` as *modules*, not *instances*,
performing the instantiation itself. Thus, every instance of `$Zipper` will get
a fresh, encapsulated instance of `$Libc` and `$Libzip`, even when these modules
are shared with other components (as will be done below).

The duplication of function signatures in `$Core` is an unfortunate consequence
of layering: since Core WebAssembly is not modified by the Component Model,
there is no way for `$Core` to [alias](Explainer.md#alias-definitions) the
outer type definitions of `$Zipper`.


## `libimg`

Next we create a shared library `libimg` that depends on `libzip` by
importing `libzip.h` from `libimg.c`:
```c
/* libimg.h */
#include <stddef.h>
#define LIBIMG(name) WASM_IMPORT(libimg, name)
void* LIBIMG(compress)(void* in, size_t in_size, size_t* out_size);
```
```c
/* libimg.c */
#include <stdlib.h>
#include "libzip.h"
void* WASM_EXPORT(compress)(void* in, size_t in_size, size_t* out_size) {
  ...
  void *out = zip(in, in_size, &out_size);
  ...
}
```

Compiling with `clang -shared libimg.c` produces a `$Libimg` module:
```wat
(module $Libimg
  (import "libc" "memory" (memory 1))
  (import "libc" "malloc" (func (param i32) (result i32)))
  (import "libzip" "zip" (func (param i32 i32 i32) (result i32)))
  (func (export "compress") (param i32 i32 i32) (result i32)
    ...
  )
)
```

## `imgmgk`

The `imgmgk` component is symmetric to the `zipper` component; the main
difference in the output is the addition of the `libzip` dependency:
```wasm
(adapter module $Imgmgk
  (import "libc" (module $Libc ...))
  (import "libzip" (module $Libzip ...))
  (import "libimg" (module $Libimg ...))

  (instance $libc (instantiate $Libc))
  (instance $libzip (instantiate $Libzip
    (import "libc" (instance $libc))
  ))
  (instance $libimg (instantiate $Libimg
    (import "libc" (instance $libc))
    (import "libimg" (instance $libzip))
  ))
)
```

## `app`

Finally, we can create the `app` component by composing the `zipper` and `imgmgk`
components. Let's say we want to bundle all the dependencies to create a single
file with no external dependencies; a bundler could produce the following:
```wasm
(adapter module $App
  (module $Libc   ... copied inline)
  (module $Libzip ... copied inline)
  (module $Libimg ... copied inline)
  (module $Zipper ... copied inline)
  (module $Imgmgk ... copied inline)
  (module $Core
    ... whatever app does
  )

  (instance $zipper (instantiate $Zipper
    (import "libc" (module $Libc))
    (import "libzip" (module $Libzip))
  ))
  (instance $imgmgk (instantiate $Imgmgk
    (import "libc" (module $Libc))
    (import "libzip" (module $Libzip))
    (import "libimg" (module $Libimg))
  ))
  (instance $core (instantiate $Core
    (import "zipper" (instance $zipper))
    (import "imgmgk" (instance $imgmgk))
  ))
)
```

Note that, while the `$Libc` and `$Libzip` modules are shared between `$Zipper`
and `$Imgmgk`, `$zipper` and `$imgmk` each get private instances of `$Libc` and
`$Libzip` and thus get private, encapsulated linear memories.

Alternatively, a Web-targeted bundler could choose to take advantage of HTTP
caching by using relative URLs that are fetched and cached by the browser via
[ESM-integration]:
```wasm
(adapter module $App
  (import "./libc.wasm" (module $Libc ...))
  (import "./libzip.wasm" (module $Libzip ...))
  (import "./libimg.wasm" (module $Libimg ...))
  (import "./zipper.wasm" (module $Zipper ...))
  (import "./imgmgk.wasm" (module $Imgmgk ...))
  ...
)
```
Importantly, by propagating dependencies as module imports, bundler tools and
developers have a range of options at build time.


## Versioning

Now what happens when `libc`, `libzip` or `libimg` are independently updated?
Using the terminology of [semver], let's consider two cases:
* Major version change: the API may have changed and so type-checking of
  instance imports may fail and we'll need to recompile at least some modules.
* Minor/Patch version change: the API may have grown, but only backwards
  compatibly, meaning that existing dependents should be able to keep working
  unmodified. However, the change may have a bug or expose a bug in client code,
  so we need to be able to control the rollout of the change.

To address these cases, we need to enrich our dynamic linking convention by
associating semantic versions with dependencies. While this could be done as
metadata embedded in a wasm module [custom section], we already have a string in
the module which serves the purpose of communicating dependency information to
the toolchain: the import string. Thus, a lightweight convention could be for
the public header of a shared library to simply include the semantic version in
the declarations. This is similar to the [soname] convention in ELF.

For example, following this convention, `stdlib.h` and `libzip.h` would look like:
```c
/* stdlib.h */
#include <stddef.h>
#define LIBC(name) WASM_IMPORT(libc-1.0.0, name)
void* LIBC(malloc)(size_t n);
```
```c
/* libzip.h */
#include <stddef.h>
#define LIBZIP(name) WASM_IMPORT(libzip-3.4.5, name)
void* LIBZIP(zip)(void* in, size_t in_size, size_t* out_size);
```

When `zipper.c` is first compiled, it will include these versions in its imports:
```wasm
(adapter module $Zipper
  (type $LibcInstance (instance
    (memory (export "memory") 1)
    (func (export "malloc") (param i32) (result i32))
  ))

  (import "libc-1.0.0" (module $Libc
    (export $LibcInstance)
  ))
  (import "libzip-3.4.5" (module $Libzip
    (import "libc-1.0.0" (instance $LibcInstance))
    (export "zip" (func (param i32 i32 i32) (result i32)))
  ))

  (instance $libc (instantiate $Libc))
  (instance $libzip (instantiate $Libzip (import "libc-1.0.0" (instance $libc))))
  ...
)
```

Now let's say `libc` finally adds `free` in new minor version 1.1.0. Let's also
say `zipper` is recompiled before `libzip`, so that `libzip` is still importing
`libc-1.0.0`, without `free`. This will produce a new `$Zipper` module which
mentions two different version numbers and instance types of `libc`:

```wasm
(adapter module $Zipper
  (import "libc-1.1.0" (module $Libc
    (memory (export "memory") 1)
    (func (export "malloc") (param i32) (result i32))
    (func (export "free") (param i32))
  ))
  (import "libzip-3.4.5" (module $Libzip
    (import "libc-1.0.0" (instance
      (memory (export "memory") 1)
      (func (export "malloc") (param i32) (result i32))
    ))
    (export "zip" (func (param i32 i32 i32) (result i32)))
  ))

  (instance $libc (instantiate $Libc))
  (instance $libzip (instantiate $Libzip (import "libc-1.0.0" (instance $libc))))
  ...
)
```
As you might hope, this module still validates without recompiling `libzip`
because `instantiate` performs a subtype check between the supplied
`libc-1.1.0` module and the expected `libc-1.0.0` module type, and subtyping
allows superfluous exports to be ignored.

Now let's say there is a major `libc` version change and the instance type does
change incompatibly. If `libzip` is compiled with an older major version than
`zipper`, then the `instantiate` validation check will fail and it
will be necessary to either upgrade `libzip` to the newer `libc` version
(creating a new major version of `libzip` in the process) or downgrade the
compilation of `zipper` to match `libzip`. When it comes to compiling
(shared-everything) programs with multiple libraries, there is no free lunch:
there has to be a single version of `libc` shared by all other libraries.

The situation is different, however, when bundling distinct *components*, as we
saw with `$App` above. In this case, since there is no shared memory, each
program can have its own major version of `libc`. In the case of minor/patch
version changes, the bundler can implicitly upgrade programs with older
versions to match programs with newer versions (to share more code/memory) or
not.


## Cyclic Dependencies

If cyclic dependencies are necessary, such cycles can be broken by:
* identifying a [spanning] DAG over the module dependency graph;
* keeping the calls along the spanning DAG's edges as normal function imports
  and direct calls (as shown above); and
* converting calls along "back edges" into indirect calls (`call_indirect`) of
  an import const global `i32` index.

For example, a cycle between modules `$A` and `$B` could be broken by arbitrarily
saying that `$B` gets to directly import `$A` and then routing `$A`'s imports
through a shared mutable `funcref` table via `call_indirect`:
```wat
(module $A
  ;; A imports B.bar indirectly via table+index
  (import "linkage" "table" (table funcref))
  (import "linkage" "bar-index" (global $bar-index mut i32))

  (type $FooType (func))
  (func $some_use
    (call_indirect $FooType (global.get $bar-index))
  )

  ;; A exports A.foo directly to B
  (func (export "foo")
    ...
  )
)
```
```wat
(module $B
  ;; B directly imports A.foo
  (import "a" "foo" (func $a_foo))  ;; B gets to directly import A

  ;; B indirectly exports B.bar to A
  (func $bar ...)
  (import "linkage" "table" (table $ftbl funcref))
  (import "linkage" "bar-index" (global $bar-index mut i32))
  (elem (table $ftbl) (offset (i32.const 0)) $bar)
  (func $start (global.set $bar-index (i32.const 0)))
  (start $start)
)
```
Lastly, a toolchain can link these together into a whole program by emitting
a wrapper adapter module that supplies both `$A` and `$B` with a shared
function table and `bar-index` mutable global.
```wat
(adapter module $Program
  (import "./A.wasm" (module $A ...))
  (import "./B.wasm" (module $B ...))
  (module $Linkage
    (global (export "bar-index") mut i32)
    (table (export "table") funcref 1)
  )
  (instance $linkage (instantiate $Linkage))
  (instance $a (instantiate $A
    (import "linkage" (instance $linkage))
  ))
  (instance $b (instantiate $B
    (import "a" (instance $a))
    (import "linkage" (instance $linkage))
  ))
)
```


## Function Pointer Identity

To ensure C function pointer identity across shared libraries, for each exported
function, a shared library will need to export both the `func` definition and an
`i32` `global` containing that `func`'s index in the global `funcref` table.

Because a shared library can't know the absolute offset in the global `funcref`
table for all of its exported functions, the table slots' offsets must be
dynamic. One way this could be achieved is by the shared library calling into a
`ftalloc` export of `libc` (analogous to `malloc`, but for allocating from the
global `funcref` table) from the shared library's `start` function. Elements could
then be written into the table (using [bulk memory operations]) at the allocated
offset and their indices written into the exported `i32`s.

(In theory, more efficient schemes are possible when the main program has more
static knowledge of its shared libraries.)


## Linear-memory stack pointer

To implement address-taken local variables, varargs, and other corner cases,
wasm compilers maintain a stack in linear memory that is maintained in
lock-step with the native WebAssembly stack. The pointer to the top of this
linear-memory stack is usually maintained in a single global `i32` variable that
must be shared by all linked instances. Following the above linking scheme, a
natural way to achieve this is for `libc` to export the `i32` global as part of
its interface and for all shared libraries and the main module to import this
global.


## Runtime Dynamic Linking

The general case of runtime dynamic linking in the style of `dlopen`, where an
*a priori unknown* module is linked into the program at runtime, is not possible
to do purely within wasm with this proposal. Additional host-provided APIs are
required for:
* compiling files or bytes into a module;
* reading the import strings of a module;
* dynamically instantiating a module given a list of import values; and
* dynamically extracting the exports of an instance.

Such APIs could be standardized as part of [WASI]. Moreover, the [JS API]
possesses all the above capabilities allowing the WASI APIs to be prototyped and
implemented in the browser.

Assuming such APIs are standardized, though, an important goal is that a single
library compiled with `-shared` should be able to be dynamically linked both at
load-time, using the pure-wasm strategy outlined in this example, and at
runtime, using APIs; the library shouldn't need to be preemptively compiled for
only one kind of dynamic linking.



[Semver]: https://semver.org
[wasi-libc]: https://github.com/WebAssembly/wasi-libc
[wasi-sdk]: https://github.com/WebAssembly/wasi-sdk
[`import_module`]: https://clang.llvm.org/docs/AttributeReference.html#import-module
[`import_name`]: https://clang.llvm.org/docs/AttributeReference.html#import-name
[`export_name`]: https://clang.llvm.org/docs/AttributeReference.html#export-name
[ESM-integration]: https://github.com/WebAssembly/esm-integration
[Custom Section]: https://webassembly.github.io/spec/core/binary/modules.html#binary-customsec
[Spanning]: https://en.wikipedia.org/wiki/Spanning_tree
[soname]: https://en.wikipedia.org/wiki/Soname
[Bulk Memory Operations]: https://github.com/WebAssembly/bulk-memory-operations/
[WASI]: https://github.com/webassembly/wasi
[JS API]: https://webassembly.github.io/spec/js-api/index.html
[Dependency Hell]: https://en.wikipedia.org/wiki/Dependency_hell
