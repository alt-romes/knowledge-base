#ghc

Package DBs, unit-ids, Cabal hashes, and wired-in units.

Q: How does cabal compute abi hashes for installed units?
A: It calls ghc with --abi-hash on all exposed modules listed in the interface (See `libraries/Cabal/src/Distribution/Simple/Register.hs`)

See [[Cabal unit-id creation]] on how the unit-ids with hashes are computed

Q: Why do we pass `-this-unit-id package-without-version` to the wired-in packages?
A: Because GHC creates Modules and Names assuming those wired-in packages will have those fixed ids (i.e. the ids without the versions).

See Note [[Note {Wired-in units}]] in GHC.Unit.Type

Q: How do we compute the list of package dependencies in a module?
A: The root of calculating the `ImportAvails` of a module (which are combined and used by `mkDependencies` to create a `Dependencies` value) is `calculateAvails` in `GHC.Rename.Names`.


* `HscEnv` has a current unit-id (`ue_current_unit . hsc_unit_env`)
* The `UnitState` (which can be gotten from a `HscEnv`) has a `wireMap` that maps database unit keys to wired-in unit-ids, i.e.
```
[(base-4.18.0.0, base), (ghc-9.7, ghc),
    (ghc-bignum-1.3, ghc-bignum), (ghc-prim-0.10.0, ghc-prim),
    (rts-1.0.2, rts), (template-haskell-2.20.0.0, template-haskell)] 
```

* The `UnitState` is computed by `mkUnitState`, which clears up most of what it considers with the plan:

The creation of the `UnitState` will define which wired-in packages are recognized (in short, the `wireMap` field of the `UnitState` is populated). See [[Note {Wired-in units}]] that uses it



The cabal-install created unit-id for packages takes a lot of flags and properties into consideration. What exactly do we need to match?

The issue is that when loading plugins we don't check if the `ghc` the plugin was compiled against is the same `ghc` the compiler loading the plugin compiles against. To do this check, we must first have a unique id that is different when the two `ghc`s differ, and is equal when the two `ghc`s are the same, and we can use the ghc's unit-id computed by cabal for that effect. However, we currently compile `ghc` the library with `-this-unit-id ghc`, because `ghc` is a wired-in package and we assume it will have exactly the unit id "`ghc`" in order to hardcode `ghc` names such as `Plugin`. Our idea is to not pass `-this-unit-id ghc` when compiling ghc (or any other wired-in package for that matter), and rather let `cabal` pass `-this-unit-id package-version-hash` as it does for all packages (that don't override it). In this sense, the wired-in Names must use the unit-id cabal will pass when compiling this package (and thus know it ahead of time), such that when we install the package, the wired-in names will refer to the actual unit-id of the package, rather than to non-existing unit-ids.

It seems to be a bit more subtle than that, because the same compiler should be able to 

Would the check be done automatically by cabal? I think not.

