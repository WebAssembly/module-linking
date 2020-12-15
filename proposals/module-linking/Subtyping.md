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
imports (more discussion about this breaking change is available on
[#7](https://github.com/WebAssembly/module-linking/issues/7) and [below as
well](#breaking-change)). This would make module `$A` an invalid module.

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

## Breaking change?!

This entire interpretation of imports relies on the aforementioned breaking
change, which is to disallow two import directives with the same names. The
rationale for this breaking change is:

* It's expected that in practice different import directives with the same name
  is exceedingly rare. So far the only confirmed cases are in test harnesses
  where you might import `console.log` with two signatures for example. It's
  hoped we can [collect
  data](https://bugzilla.mozilla.org/show_bug.cgi?id=1647791) to back up this
  claim.

* Another hope is that the spec can change to disallow duplicate imports, but
  enginess with backwards-compatibility guarantees could deviate from the spec
  in this regard and allow duplicate import strings in older modules. This way
  engines that can could follow the spec exactly, and if necessary engines
  wouldn't have to break existing content.
