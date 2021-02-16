# Shared-Everything Dynamic Linking Example

This document walks through the 
[Shared-Everything Dynamic Linking](Explainer.md#shared-everything-dynamic-linking)
use case. One high-level point to make before starting is that the scheme
outlined below avoids the need for any sort of runtime loader (like `ld.so`).
Rather, Module Linking proposal is used in such a way that the wasm engine ends
up doing everything.

As with native, shared-everything dynamic linking is rather involved, requiring
conventions followed by the toolchain, shared libraries and programs. We start
with `libc`:

## libc

`libc` is special in that, conventionally, an implementation (like [wasi-libc])
is bundled with a compiler (like clang) to produce an SDK (like [wasi-sdk]).

For our hypothetical dynamic linking convention, we'll have `libc` define and
export a linear memory which will be imported by the rest of the program, along
with `malloc`, `free`, etc. Thus, `$LIBC` is roughly shaped:

```wasm
(module $LIBC
  (memory (export "memory") 1)
  (func (export "malloc") (param i32) (result i32) ...impl)
  ...
)
```

The SDK also contains standard library headers which contain declarations
compatible with `$LIBC`. To implement them, we first need some helper macros,
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

## libzip

With `libc` sketched out, we have a functional SDK we can now use to create the
shared library `libzip`. Its C definition has the rough shape:

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
produces a `$LIBZIP` module:

```wasm
(module $LIBZIP
  (import "libc" (instance $libc
    (export "memory" (memory 1))
    (export "malloc" (func (param i32) (result i32)))
  ))
  (alias $libc "memory" (memory))  ;; default memory b/c index 0
  (func (export "zip") (param i32 i32 i32) (result i32)
    ...
    call (func $libc "malloc")
    ...
  )
)
```

## zipper

Using `libc` and `libzip`, we can now write the `zipper` program:

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

When compiled with `clang zipper.c`, we get a `$ZIPPER` module:

```wasm
(module $ZIPPER
  (type $Libc (instance
    (export "memory" (memory 1))
    (export "malloc" (func (param i32) (result i32)))
  ))
  (type $LibZip (instance
    (export "zip" (func (param i32 i32 i32) (result i32)))
  ))

  (import "libc" (module $LIBC
    (export (type outer $ZIPPER $Libc))
  ))
  (import "libzip" (module $LIBZIP
    (import "libc" (instance (type outer $ZIPPER $Libc)))
    (export (type outer $ZIPPER $LibZip))
  ))

  (instance $libc (instantiate $LIBC))
  (instance $libzip (instantiate $LIBZIP (import "libc" (instance $libc))))

  (alias $libc "memory" (memory))  ;; default memory b/c index 0
  (func $main
    ...
    call (func $libc "malloc")
    ...
    call (func $libzip "zip")
    ...
  )
)
```

Note that, `zipper` imports `libc` and `libzip` as *modules*, not *instances*,
performing the instantiation itself, every time a `zipper` instance is created.
The modules are imported so that, as we'll see next, `zipper` can share modules
with other programs that use the same libraries. However, if a developer wants a
standalone version without any imports, a bundling tool can trivially replace
the module imports with nested modules, producing a bundled `$ZIPPER-BUNDLED`
module:

```wasm
(module $ZIPPER-BUNDLED
  (module $LIBC ...copied inline)
  (module $LIBZIP ...copied inline)
  (instance $libc (instantiate $LIBC))
  (instance $libzip (instantiate $LIBZIP))
  (func $main ...)
)
```

## libimg

Next we create a shared library `libimg` that depends on `libzip`. The C
definition is quite symmetric to `libzip`:

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

Compiling with `clang -shared libimg.c` produces a `$LIBIMG` module:

```wat
(module $LIBIMG
  (import "libc" (instance $libc
    (export "memory" (memory 1))
    (export "malloc" (func (param i32) (result i32)))
  ))
  (import "libzip" (instance $libzip
    (export "zip" (func (param i32 i32 i32) (result i32)))
  ))

  (alias $libc "memory" (memory))  ;; default memory b/c index 0
  (func (export "compress") (param i32 i32 i32) (result i32)
    ...
    call (func $libc "malloc")
    ...
    call (func $libzip "zip")
    ...
  )
)
```

Note that `libimg` imports *both* `libc` and `libzip` as instances. This ensures
that if there are multiple uses of `libzip` in the client program of `libimg`,
that the program only contains one `libzip` instance shared by all. Concretely,
this ensures that, e.g., there is only one copy of the C static data declared by
`libzip` in the program. While this requires all shared libraries to import all
their dependencies, higher-level tooling can automate this process so that no
manual recompilation is necessary; the propagation can happen automatically at
a final build/bundle stage.

## imgmgk

Using `libimg` we can create the `imgmgk` program. `imgmgk.c` is symmetric with
`zipper.c` and, when compiled, produces output symmetric to `$ZIPPER`, the
main difference being the additional transitive dependency (`libzip`).

```wasm
(module $IMGMGK
  (import "libc" (module $LIBC ...))
  (import "libzip" (module $LIBZIP ...))
  (import "libimg" (module $LIBIMG ...))

  (instance $libc (instantiate $LIBC))
  (instance $libzip (instantiate $LIBZIP
    (import "libc" (instance $libc))
  ))
  (instance $libimg (instantiate $LIBIMG
    (import "libc" (instance $libc))
    (import "libimg" (instance $libzip))
  ))

  (func $main
    ...
  )
)
```

## Whole Application

To illustrate sharing, let's say we now bundle together both `zipper` and
`imgmgk` as part of a single deployment artifact. One option the bundler has is
to simply embed all dependencies into a containing bundle module:

```wasm
(module $DEPLOYMENT-BUNDLE
  (module $LIBC ...copied inline)
  (module $LIBZIP ...copied inline)
  (module $LIBIMG ...copied inline)
  (module $ZIPPER ...copied inline)
  (module $IMGMGK ...copied inline)

  (instance (export "zipper") (instantiate $ZIPPER
    (import "libc" (module $LIBC))
    (import "libzip" (module $LIBZIP))
  ))
  (instance (export "imgmgk") (instantiate $IMGMGK
    (import "libc" (module $LIBC))
    (import "libzip" (module $LIBZIP))
    (import "libimg" (module $LIBIMG))
  ))
)
```

Note that, while the `libc` and `libzip` modules are shared, both `$zipper` and
`$imgmk` get their own private, encapsulated instances of `libc` and `libzip`.

Alternatively, a Web-targeted bundler (like webpack) could choose to take
advantage of HTTP caching by using relative URLs that are fetched by the browser
via [ESM-integration]:

```wasm
(module $WEB-BUNDLE
  (import "./libc.wasm" (module $LIBC ...))
  (import "./libzip.wasm" (module $LIBZIP ...))
  (import "./libimg.wasm" (module $LIBIMG ...))
  (import "./zipper.wasm" (module $ZIPPER ...))
  (import "./imgmgk.wasm" (module $IMGMGK ...))

  (instance (export "zipper") (instantiate $ZIPPER
    (import "libc" (module $LIBC))
    (import "libzip" (module $LIBZIP))
  ))
  (instance (export "imgmgk") (instantiate $IMGMGK
    (import "libc" (module $LIBC))
    (import "libzip" (module $LIBZIP))
    (import "libimg" (module $LIBIMG))
  ))
)
```
or some hybrid configuration which nests some modules and module-imports others
in order to balance minimizing requests and maximizing cache hits. Bundlers for
other hosts may similarly exercise host-specific mechanisms for optimizing code
sharing.

Importantly, by propagating dependencies as module imports, where module sharing
affects only performance characteristics, not behavior or correctness (due to
explicit, private instantiation), bundler tools and developers have a range of
options at build time.

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

When `zipper.c` is compiled, it will include these versions in its imports:

```wasm
(module $ZIPPER
  (type $Libc (instance
    (memory (export "memory") 1)
    (func (export "malloc") (param i32) (result i32))
  ))

  (import "libc-1.0.0" (module $LIBC
    (export (type outer $ZIPPER $Libc))
  ))
  (import "libzip-3.4.5" (module $LIBZIP
    (import "libc-1.0.0" (instance (type outer $ZIPPER $Libc)))
    (export "zip" (func (param i32 i32 i32) (result i32)))
  ))

  (instance $libc (instantiate $LIBC))
  (instance $libzip (instantiate $LIBZIP (import "libc-1.0.0" (instance $libc))))
  ...
)
```

Now let's say `libc` finally adds `free` in new minor version 1.1.0. Let's also
say `zipper` is recompiled before `libzip`, so that `libzip` is still importing
`libc-1.0.0`, without `free`. This will produce a new `$ZIPPER` module which
mentions two different version numbers and instance types of `libc`:

```wasm
(module $ZIPPER
  (import "libc-1.1.0" (module $LIBC
    (memory (export "memory") 1)
    (func (export "malloc") (param i32) (result i32))
    (func (export "free") (param i32))
  ))
  (import "libzip-3.4.5" (module $LIBZIP
    (import "libc-1.0.0" (instance
      (memory (export "memory") 1)
      (func (export "malloc") (param i32) (result i32))
    ))
    (export "zip" (func (param i32 i32 i32) (result i32)))
  ))

  (instance $libc (instantiate $LIBC))
  (instance $libzip (instantiate $LIBZIP (import "libc-1.0.0" (instance $libc))))
  ...
)
```

As you might hope, this module still validates without recompiling `libzip`
because `instantiate` performs a subtype check which ignores the import string
`"libc-1.0.0"` (because import operands are supplied positionally) and the
`libc-1.1.0` instance type is indeed a subtype of the `libc-1.0.0` instance
type.

Now let's say there is a major `libc` version change and the instance type does
change incompatibly. If `libzip` is compiled with an older major version than
`zipper`, then the `instance.instantiation` validation check will fail and it
will be necessary to either upgrade `libzip` to the newer `libc` version
(creating a new major version of `libzip` in the process!) or downgrade the
compilation of `zipper` to match `libzip`.

Overall, when it comes to compiling (shared-everything) programs with multiple
libraries, there is no free lunch: there has to be sufficient agreement on the
single version of `libc` and every other shared library.

The situation is different, however, when bundling distinct *programs*, as we
saw with `$DEPLOYMENT-BUNDLE` above. In this case, since there is no sharing
of memory, each program can have its own major version of `libc`. In the case of
minor/patch version changes, the bundler can implicitly upgrade programs with
older versions to match programs with newer versions (to share more code/memory)
or not.

Ultimately, the above versioning scheme may be too simplistic when many
libraries and programs are used within a single application. Instead of simply
embedding and propagating version numbers, semver *range patterns* may need to
be specified for each dependency to provide developers finer-grained control
over how the final version is chosen. Any such scheme would need to be tailored
to a higher-level layer of tooling and should be able to built on the basic
primitives of the Module Linking proposal, similar to the simplistic scheme
shown above.

## Cyclic Dependencies

If cyclic dependencies are necessary, such cycles can be broken by:
* identifying a [spanning] DAG over the module dependency graph;
* keeping the calls along the spanning DAG's edges as normal function imports
  and direct calls (as shown above); and
* converting calls along "back edges" into indirect calls (`call_indirect`) of
  an import const global `i32` index.

For example, a cycle between modules `a` and `b` could be broken by arbitrarily
saying that `b` gets to directly import `a` and then routing `a`'s imports
through the global `funcref` table via `call_indirect`. This is achieved by
having `b` import `a` as a *module* and then, when `b` instantiates `a`, `b`
supplies mutable `i32` globals containing the index of each function `a` wants
to import of `b`'s. For example:
```wat
(module $A
  (type $Libc (instance ...))
  (import "libc" (instance (type $Libc)))
  (import "bar-index" (global $bar-index mut i32))
  (type $FooType (func))
  (func (export "foo")
    (call_indirect $FooType (global.get $bar-index))
  )
)
```
```wat
(module $B
  (type $Libc (instance ...))
  (import "libc" (module $LIBC (export (type outer $B $Libc))))
  (instance $libc (instantiate $LIBC))

  (global $bar-index mut i32)

  (import "a" (module $A
    (import "libc" (instance (type outer $B $Libc)))
    (import "bar-index" (global mut i32))
    (export "foo" (func $foo))
  ))
  (instance $a (instantiate $A
    (import "libc" (instance $libc)
    (import "bar-index" (global $bar-index))
  ))

  ;; indirectly export bar to a:
  (func $bar ...)
  (elem (i32.const 0) $bar)
  (func $start (global.set $bar-index (i32.const 0)))
  (start $start)

  (func $run (export "run")
    call (func $a "foo")
  )
)
```
Here, calling `$run` will successfully call from `b` into `a` back into `b`.


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
