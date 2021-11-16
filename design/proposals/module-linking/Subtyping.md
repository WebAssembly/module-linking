# Subtyping

Subtyping will extend what's currently ["import
matching"](https://webassembly.github.io/spec/core/exec/modules.html#import-matching).
The main gotcha with subtyping primarily comes around with how two module
types' import lists are related. This will go into more detail below on that.
First though:

## Instance Subtyping

Instance types are a map of exports from the name of exports to the type of the
export. We can generally say that for one map `e1` to be a subtype of another
map `e2` then everything in `e2` must be present in `e1`:

```
∀ name ∈ e2 : e1[name] ≤ e2[name]
---------------------------------
             e1 ≤ e2
```

And since that's all instance types are we can just use that to define subtyping
between instances:

```
            e1 ≤ e2
-------------------------------
  {exports e1} ≤ {exports e2}
```

With some examples:

```wasm
(instance) ≤ (instance)

;; If asked for something that doesn't have exports and provided something that
;; has exports, that's ok.
(instance (export "" (func))) ≤ (instance)

;; Asked for something that exports an instance with no fields, it's ok to
:; provide an instance which exports an instance with exports. When that
;; instance's nested export is loaded we don't think it has anything, so it's ok
;; that it does.
(instance (export "" (instance (export "e" (func)))))
  ≤ (instance (export "" (instance)))
```

## Import Elaboration

modules are trickier than instances, so first this will describe some tweaks to
imports in the current proposal.  It's important to note [that all two-level
imports are now equivalent to instance
imports](./Explainer.md#import-definitions), meaning that all
imports are a name and a type. The proposed method of elaboration is to group
all two-level imports with the same module name next to one another, and then
flatten each group of two-level imports into one one-level import of an
instance with appropriately named and typed exports.

For example this core module:

```wasm
(module $A
  (import "a" "foo" (func))
  (import "b" "bar" (func))
  (import "a" "baz" (func))
)
```

is assigned this type:

```wasm
(module $A
  (import "a" (instance
    (export "foo" (func))
    (export "baz" (func))
  ))
  (import "b" (instance
    (export "bar" (func))
  ))
)
```

However, this module:

```wasm
(module $A
  (import "" "a" (func))
  (import "" "a" (func (result i32)))
)
```

cannot be assigned this type:

```wasm
(module
  (import "" (instance
    (export "a" (func))
    (export "a" (func (result i32))
  ))
)
```

because duplicate exports are not allowed in module types. Thus, the Module
Linking does not allow nesting `$A` nor importing `$A` via a module-typed
import.

Similarly, none of the following core modules can be assigned module types:

```wasm
(module
  (import "" (func))
  (import "" "a" (func))
)

(module
  (import "" (instance))
  (import "" "a" (func))
)

(module
  (import "" (instance))
  (import "" (instance))
)
```

## Module Subtyping

With the single-level import interpretation above, imports are symmetric to
exports: they're a map from name to a type. Module `{imports i1, exports e1}`
is thus a subtype of `{imports i2, exports e2}` if this property holds:

```
          i2 ≤ i1              e1 ≤ e2
---------------------------------------------------
{imports i1, exports e1} ≤ {imports i2, exports e2}
```

Handling of exports is the same as instances, which basically means that it's ok
if `m1` exports more than what `m2` expects. The subtlety with imports is that
the order of checking is reversed, it's ok to import less than what's
expected.

Some examples of valid modules are:

```wasm
(module) ≤ (module)

;; If asked for something that needs an import and provided something that
;; doesn't need any imports, that's ok
(module) ≤ (module (import "" (instance)))

;; Asked for something that imports an instance with a field export, it's ok to
;; provide a module which imports an instance with no exports. When that module
;; is given the instance-with-exports, it just ignores it.
(module (import "" (instance)))
  ≤ (module (import "" (instance (export "e" (func)))))
```

## Instantiation

Within WebAssembly, instantiation is similar to subtyping: the `instantiate`
caller (either from the JS API or an `(instance)` definition) supplies a
sequence of (name, value) pairs and the imports of the module being
instantiated are checked against this list, ignoring order and just matching on
names. As with module import subtyping, the `instantiate` caller may supply
superfluous names that are ignored and duplicate names are not allowed.

While two-level imports are desugared into single-level imports of instances,
no such desugaring is performed by `instantiate` from within WebAssembly:
two-level imports simply do not exist and imports of instances must be passed
instances. This is backwards-compatible since, before this proposal, only the
JS API's `WebAssembly.instantiate` existed, which already behaved in the
desired way.

WebAssembly *embeddings* may however choose to flatten imports of instances
into lists of imports of the instances' fields, preserving backwards
compatibility and avoiding the need to create intermediate instances.
