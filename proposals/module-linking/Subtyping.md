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
imports](./Explainer.md#instance-imports-and-aliases), meaning that all
imports are a name and a type. The proposed method of elaboration is to group
all two-level imports with the same module name next to one another, and then
flatten each group of two-level imports into one one-level import of an
instance with appropriately named and typed exports.

For example this:

```wasm
(module $A
  (import "a" "foo" (func))
  (import "b" "bar" (func))
  (import "a" "baz" (func))
)
```

becomes this:

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

A caveat with this, however, is that this module

```wasm
(module $A
  (import "" "a" (func))
  (import "" "a" (func (result i32)))
)
```

cannot be recast as:

```wasm
(module
  (import "" (instance
    (export "a" (func))
    (export "a" (func (result i32))
  ))
)
```

because exports in instances must have unique names. As a result **this proposal
proposes a breaking change** to forbid duplicate import strings between two
imports. This would make module `$A` an invalid module.

Note that this breaking change would also mean that these modules are all
invalid:

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

With the import elaboration above it means that imports are actually the same
thing as exports, a map from name to a type. Module `{imports i1, exports e1}`
is then a subtype of `{imports i2, exports e2}` if this property holds:

```
          i2 ≤ i1              e1 ≤ e2
---------------------------------------------------
{imports i1, exports e1} ≤ {imports i2, exports e2}
```

Handling of exports is the same as instances, which basically means that it's ok
if `m1` exports more than what `m2` expects. The subtelty with imports is that
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

Instantiation primarily occurs via name-based resolution (e.g. in the JS API and
other language embeddings) or position-based resolution (e.g. embedded engines).

It's expected that the original import list of a module is retained to map
positional-based resolution to name-based resolution. With positional-based
resolution imports would need to be provided as-is with the module in question
(no sugar applied where you can supply an instance for a function import). This
enables the engine to transform a list of imports into a map from import name to
an instance with exports.

Name-based resolution wouldn't need to change too too much, it would allow
top-level names to be defined with actual wasm instances or further host-defined
maps of strings. Instance imports could then be satisfied with maps-of-strings
so long as all the strings line up. Note that the `instantiate` instruction is
expected to use named-based instantiation.

In both cases instantiation is intended to become primarily name-based. This
matches the intended behavior of the `instantiate` instruction in a wasm module
which is to name all the provided items according to the declared module type
that's being instantiated. This also enables embedders to work with instances
supplied to satisfy a list of function imports. For example embedders would take
a singular "wasi instance" to satisfy all wasi function imports from a module.
