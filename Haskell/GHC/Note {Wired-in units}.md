#ghc 

How can we remove `-this-unit-id ghc` such that the one passed by cabal is the one effectively used?

Currently, we have a note for this in `GHC.Unit.Types`:
```
Note [Wired-in units]
~~~~~~~~~~~~~~~~~~~~~

Certain packages are known to the compiler, in that we know about certain
entities that reside in these packages, and the compiler needs to
declare static Modules and Names that refer to these packages.  Hence
the wired-in packages can't include version numbers in their package UnitId,
since we don't want to bake the version numbers of these packages into GHC.

So here's the plan.  Wired-in units are still versioned as
normal in the packages database, and you can still have multiple
versions of them installed. To the user, everything looks normal.

However, for each invocation of GHC, only a single instance of each wired-in
package will be recognised (the desired one is selected via
@-package@\/@-hide-package@), and GHC will internally pretend that it has the
*unversioned* 'UnitId', including in .hi files and object file symbols.

Unselected versions of wired-in packages will be ignored, as will any other
package that depends directly or indirectly on it (much as if you
had used @-ignore-package@).

The affected packages are compiled with, e.g., @-this-unit-id base@, so that
the symbols in the object files have the unversioned unit id in their name.

Make sure you change 'GHC.Unit.State.findWiredInUnits' if you add an entry here.
```

The main issue seems to be that a compiler might compile different packages that depend on different versions of wired-in units, and ghc should be able to link against any of those unit's versions but have its wired-in names still "connect" to the unit's, regardless of version.
*We don't want to bake the version numbers of these packages into GHC*.

All of the wired-in units (on top of which wired-in modules are created) are found beneath this Note in `GHC.Unit.Types` (while the wired-in types and names are in `GHC.Builtin.*`)

