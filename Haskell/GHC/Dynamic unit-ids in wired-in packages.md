#ghc 

In order to achieve [[Plugin Loading Incompatibity]] checks, we need to give `ghc` and other wired-in packages proper unit-ids (see [[Note {About units}]]).

Currently, we pass an unversioned `-this-unit-id ghc` when building `ghc`, and similarly for other wired-in packages. This is because `ghc` hardcodes names expecting the unit-id of `ghc` to be exactly `ghc`.

We don't want to bake into the compiler the version of the wired-in packages into the compiler (see [[Note {Wired-in units}]]) -- this would make the compiler effectively only able to link wired-in packages it compiles against the wired-in packages the compiler itself is linked to/was built against. I'm not sure we can currently do this, but I don't want to further complicate a future in which it is possible.

The best solution seems to be to have the wired-in unit-ids depend on the value of the -this-unit-id flag, effectively only being able to know the unit-id of the wired-in packages at runtime.
* When we compile a package such as `ghc`, cabal (or hadrian) will pass to `-this-unit-id` their computed unit-id that includes a hash identifying the package.
* When we compile a package that depends on a package such as `ghc`, cabal will pass `-package-id ghc-package-id`.
* The `package-id` is associated with a unit-id, and the information of the available packages by name is stored in `UnitState` (see [[UnitState]]).
* *The wired-in names should depend on the unit-id of the wired-in unit found in UnitState*. That is, to create a wired-in name, we must find the unit-id of the wired-in unit/package which can be done through `UnitState`, and use that value, which is only known at runtime.

For a first iteration, we should write a global variable with `unsafePerformIO` and `NOINLINE` with the unit-ids of the wired-in packages, just to test how this would work out. Then, do a proper refactor to thread the wired-in unit-id information around

A benefit of this approach is that it paves the way for a `{-# BUILTIN #-}` pragma that would define what exactly is built in (effectively we could move the builtin definitions to a base that uses this pragma)

From the -package-id flags, we need to construct an *unwire map*, which maps the names of the wired-in units to the actual wired-in unit-ids (wired-in names -> wired-in unit-id), and tentatively store it in the [[UnitState]].
To construct the *unwire map*, we must update `findWiredInUnits` in `GHC.Unit.State` which is where we previously constructed a mapping from the found wired-in units to their canonical names (base-1.0 ==> base).

When defining the wired-in unit-ids, we must take into consideration that if we don't depend on the wired-in packages (e.g. when we're building ghc we don't depend on ghc), then we cannot have a unit-id for the wired-in packages we don't depend on. That's fine (?), since if we don't depend on those packages we won't link to them and therefore whatever the unit-id of the wired-in names we wouldn't be able to use them. In practice, we don't use the wired-in names of the packages we don't depend on, so it doesn't matter whether they've got an actual unit-id when we don't need them.
No, it isn't fine. Wired-in names seem to be used regardless of whether the package is specifically specified with -package-id, however, the package is still listed in the `UnitState` store, which also takes into consideration implicit package databases and the wired-in things I guess.
When exactly do we not have access to the wired-in packages in the database (besides perhaps when building them?)
