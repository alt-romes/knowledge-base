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
