#ghc 

In order to achieve [[Plugin Loading Incompatibity]] checks, we need to give `ghc` and other wired-in packages proper unit-ids (see [[Note {About units}]]).

Currently, we pass an unversioned `-this-unit-id ghc` when building `ghc`, and similarly for other wired-in packages. This is because `ghc` hardcodes names expecting the unit-id of `ghc` to be exactly `ghc`.

We don't want to bake into the compiler the version of the wired-in packages into the compiler (see [[Note {Wired-in units}]]) -- this would make the compiler effectively only able to link wired-in packages it compiles against the wired-in packages the compiler itself is linked to/was built against. I'm not sure we can currently do this, but I don't want to further complicate a future in which it is possible.

The best solution seems to be to have the wired-in unit-ids depend on the value of the -this-unit-id flag, effectively only being able to know the unit-id of the wired-in packages at runtime.
* When we compile a package such as `ghc`, cabal (or hadrian) will pass to `-this-unit-id` their computed unit-id that includes a hash identifying the package.
* When we compile a package that depends on a package such as `ghc`, cabal will pass `-package-id ghc-package-id`.
* The `package-id` is associated with a unit-id, and the information of the available packages by name is stored in `UnitState` (see [[UnitState]]).
* *The wired-in names should depend on the unit-id of the wired-in unit found in UnitState*. That is, to create a wired-in name, we must find the unit-id of the wired-in unit/package which can be done by looking up the package name in `UnitState`, and use the looked up value, which is only known at runtime.

For a first iteration, we should write a global variable with `unsafePerformIO` and `NOINLINE` with the unit-ids of the wired-in packages, just to test how this would work out. Then, do a proper refactor to thread the wired-in unit-id information around

A benefit of this approach is that it paves the way for a `{-# BUILTIN #-}` pragma that would define what exactly is built in (effectively we could move the builtin definitions to a base that uses this pragma)

From the -package-id flags, we need to construct an *unwire map*, which maps the names of the wired-in units to the actual wired-in unit-ids (wired-in names -> wired-in unit-id), and tentatively store it in the [[UnitState]].
To construct the *unwire map*, we must update `findWiredInUnits` in `GHC.Unit.State` which is where we previously constructed a mapping from the found wired-in units to their canonical names (base-1.0 ==> base).

When defining the wired-in unit-ids, we must take into consideration that if we don't depend on the wired-in packages (e.g. when we're building ghc we don't depend on ghc), then we cannot have a unit-id for the wired-in packages we don't depend on?

The built-in ghc names are the plugin module defined in `GHC.Builtin.Names` (`pLUGINS`)

A current issue with the unsafePerformIO workaround is that the wired-in names are needed before the wired-in map is set by `mkUnitState`.

The `loadInterface` function will search for the module interface in the UnitState by calling `findAndReadIface` which in turn calls `findPackageModule`.

The `loadInterface` function takes an `SDoc` documenting the reason of the call. It's very useful in debugging.

Execution trace:
```
	loadInterface: ghc-prim
	loadInterface: template-haskell
	loadInterface: base
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: ghc-prim
	loadInterface: template-haskell-2.20.0.0
	computeInterface:find_iface
	  template-haskell-2.20.0.0:Language.Haskell.TH.Syntax
	findPackageModule
	  (template-haskell-2.20.0.0,
	   [Cabal-3.9.0.0, Cabal-syntax-3.9.0.0, array-0.5.5.0, base,
	    binary-0.8.9.1, bytestring-0.11.4.0, containers-0.6.7,
	    deepseq-1.4.8.1, directory-1.3.8.1, exceptions-0.10.7,
	    filepath-1.4.100.1, ghc, ghc-bignum, ghc-boot-9.7, ghc-boot-th-9.7,
	    ghc-heap-9.7, ghc-prim, ghci-9.7, haskeline-0.8.2.1, hpc-0.6.2.0,
	    mtl-2.3.1, parsec-3.1.16.1, pretty-1.1.3.6, process-1.6.17.0, rts,
	    stm-2.5.1.0, system-cxx-std-lib-1.0, template-haskell,
	    terminfo-0.4.1.6, text-2.0.2, time-1.12.2, transformers-0.6.1.0,
	    unix-2.8.1.0, xhtml-3000.2.2.1])
	Failed to load interface for ‘Language.Haskell.TH.Syntax’
	no unit id matching ‘template-haskell-2.20.0.0’ was found
	(This unit ID looks like the source package ID;
	 the real unit ID is ‘template-haskell’)

	------- AND ---------
	loadInterface:
	  (Language.Haskell.TH.Syntax is a family-instance module,
	   template-haskell,
	   Language.Haskell.TH.Syntax)
	loadInterface:
	  (Data.Functor.Product is a family-instance module,
	   base,
	   Data.Functor.Product)
	loadInterface:
	  (Need home interface for wired-in thing Type, ghc-prim, GHC.Types)
	loadInterface:
	  (Need home interface for wired-in thing Type, ghc-prim, GHC.Types)
	loadInterface:
	  (Need home interface for wired-in thing Type, ghc-prim, GHC.Types)
	loadInterface:
	  (Need home interface for wired-in thing Type, ghc-prim, GHC.Types)
	loadInterface:
	  (Need home interface for wired-in thing Type, ghc-prim, GHC.Types)
	loadInterface:
	  (Need home interface for wired-in thing Type, ghc-prim, GHC.Types)
	loadInterface:
	  (Need home interface for wired-in thing Type, ghc-prim, GHC.Types)
	loadInterface:
	  (Need home interface for wired-in thing Type, ghc-prim, GHC.Types)
	loadInterface:
	  (Need home interface for wired-in thing Type, ghc-prim, GHC.Types)
	loadInterface:
	  (Need home interface for wired-in thing Type, ghc-prim, GHC.Types)
	loadInterface:
	  (Need home interface for wired-in thing Type, ghc-prim, GHC.Types)
	loadInterface:
	  (Need home interface for wired-in thing Symbol,
	   ghc-prim,
	   GHC.Types)
	loadInterface:
	  (The name `Lift' is mentioned explicitly,
	   template-haskell-2.20.0.0,
	   Language.Haskell.TH.Syntax)
	computeInterface:find_iface
	  template-haskell-2.20.0.0:Language.Haskell.TH.Syntax
	findPackageModule
	  (template-haskell-2.20.0.0,
	   [Cabal-3.9.0.0, Cabal-syntax-3.9.0.0, array-0.5.5.0, base,
	    binary-0.8.9.1, bytestring-0.11.4.0, containers-0.6.7,
	    deepseq-1.4.8.1, directory-1.3.8.1, exceptions-0.10.7,
	    filepath-1.4.100.1, ghc, ghc-bignum, ghc-boot-9.7, ghc-boot-th-9.7,
	    ghc-heap-9.7, ghc-prim, ghci-9.7, haskeline-0.8.2.1, hpc-0.6.2.0,
	    mtl-2.3.1, parsec-3.1.16.1, pretty-1.1.3.6, process-1.6.17.0, rts,
	    stm-2.5.1.0, system-cxx-std-lib-1.0, template-haskell,
	    terminfo-0.4.1.6, text-2.0.2, time-1.12.2, transformers-0.6.1.0,
	    unix-2.8.1.0, xhtml-3000.2.2.1])
	Failed to load interface for ‘Language.Haskell.TH.Syntax’
	no unit id matching ‘template-haskell-2.20.0.0’ was found
	(This unit ID looks like the source package ID;
	 the real unit ID is ‘template-haskell’)
```

The above error is caused because `Language.Haskell.TH.Syntax` is a wired-in module, and thus uses the wired-in unit-id of `template-haskell` which is mapped to `template-haskell-2.20.0.0`. If we compile `template-haskell` without `-this-unit-id template-haskell`, we no longer get the error.

However, we must ensure we can build the stage0 compiler with these changes, which needs `-this-unit-id template-haskell`. The solution is to first compile stage0 with the dynamic wired-in units code but with `-this-unit-id template-haskell` and then compile stage1 with the same code but removing `-this-unit-id template-haskell`. This will entail that our bootstrap compiler will have to be updated wrt dynamic knowledge of wired-in units before we can build stage0 and stage1 without changing `-this-unit-id` flags.

Steps to build:
```sh
# Compile stage0
./hadrian/build -j --flavour=default+no_profiled_libs+omit_pragmas

# Remove -this-unit-id template-haskell (last line of file)
sed -i '' '$d' libraries/template-haskell/template-haskell.cabal.in

# Compile stage1 with existing stage0
./hadrian/build -j --freeze1 --flavour=default+no_profiled_libs+omit_pragmas
```

The next error is
```
libraries/exceptions/src/Control/Monad/Catch.hs:98:1: error:
    Bad interface file: /Users/romes/ghc-dev/ghc-no-this-unit-id-ghc/_build/stage1/inplace/../libraries/template-haskell/build/Language/Haskell/TH/Syntax.hi
        Something is amiss; requested module  template-haskell:Language.Haskell.TH.Syntax differs from name found in the interface file template-haskell-2.20.0.0:Language.Haskell.TH.Syntax (if these names look the same, try again with -dppr-debug)
```
Indeed, we are reading `template-haskell-2.20.0.0` from the binary interface file, but we are expected `template-haskell` as the unit-id. This is most likely due to our wanted module not yet taking into consideration the wire-map correspondence?

See Notes [[Why we need UnitState for wired-in names]]

To take into consideration: Since I've removed the logic to update wired-in unit ids in dependencies and such, I can no longer have a "partial solution". All wired-in units must be updated simultaneously.

TODO:
[]  Revert removal of update packages logic. It was needed for more than renaming modules.
[-] Create `WiredInUnitId` type for the wired-in unit ids "names", and an IO function that transforms them into actual `UnitId`s. Doesn't need to be IO! Just needs a UnitState
[] Ensure that when `loadInterface` is called on a module, the module's unit id is already the "real" one.
[-] `findAndReadIface` must be changed to account for the new wired-in unit logic.
[ ] Write a note [[Figuring out the wired-in unit ids]] and a smaller one [[Why we need UnitState for wired-in names]]