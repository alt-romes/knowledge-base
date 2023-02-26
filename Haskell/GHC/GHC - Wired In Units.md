Package DBs, unit-ids, Cabal hashes, and wired-in units.

Q: How does cabal compute abi hashes for installed units?
A: It calls ghc with --abi-hash on all exposed modules listed in the interface (See `libraries/Cabal/src/Distribution/Simple/Register.hs`)

Q: How does cabal compute unit-ids for installed units?
A: The hash calculated by `hashPackageHashInputs` in `cabal-install:Distribution.Client.PackageHash`, which takes into consideration some `PackageHashInputs`. The full unit-id is put together by `hashedInstalledPackageId` in the same module, which also depends on some `PackageHashInputs`. The unit-id (`InstalledPackageId` in cabal-install) is different depending on the operating system because of name-length constraints (on windows we use shorter names, and on macos we use names even shorter than on windows).

How can we recreate the unit-id of a package?
* `renderPackageHashInputs` does laboured rendering of the `PackageHashInputs` to a string. This logic is needed to compute the unit-id
* `hashedInstalledPackageId` creates the unit-id (longer or shorter) depending on the platform. This is also needed to recreate a unit-id
* We need a SHA256 function to hash the rendered package hash inputs
* We need the `PackageHashInputs` as they exist at cabal-install time

Q: Why do we pass `-this-unit-id package-without-version` to the wired-in packages?
A: Because GHC creates Modules and Names assuming those wired-in packages will have those fixed ids (i.e. the ids without the versions).

See Note [Wired-in units] in GHC.Unit.Type

Q: How do we compute the list of package dependencies in a module?
A: The root of calculating the `ImportAvails` of a module (which are combined and used by `mkDependencies` to create a `Dependencies` value) is `calculateAvails` in `GHC.Rename.Names`.

Q: Why do interface files of wired-in packages have the version number of the package in its module dependencies?
A: The field `imp_direct_dep_mods` from `ImportAvails`, which is computed by `caculateAvails` has the module dependencies.
	* If we print the direct dependencies from the `ImportAvails` returned by `calculateAvails`, or `rnImports`, or even if we print the direct module dependencies when `mkDependencies`  is called, or in `Iface/Make.hs`, when building ghc the library, the unit id of GHC is indeed `ghc` without the version (despite the interface files having ghc-9.x.x as the dependency unit-id)
	* If we print the direct dependencies when writing the binary interface file, we also have `ghc` without the version as the dependency.
	* If we print the direct dependencies of the iface returned by `readBinIface`, we also get an unversioned  `ghc`.
	* Aha! `pprDeps` calls out to `pprWithUnitState` which "Print unit-ids with UnitInfo found in the given UnitState". Basically, `pprWithUnitState` will set the `sdocUnitIdForUser` field in the `SDoc` configuration, which is used to pretty print unit-ids in a more friendly way. The field comment:
 
```haskell
, sdocUnitIdForUser               :: !(FastString -> SDoc)
      -- ^ Used to map UnitIds to more friendly "package-version:component"
      -- strings while pretty-printing.
      --
      -- Use `GHC.Unit.State.pprWithUnitState` to set it. Users should never
      -- have to set it to pretty-print SDocs emitted by GHC, otherwise it's a
      -- bug. It's an internal field used to thread the UnitState so that the
      -- Outputable instance of UnitId can use it.
      --
      -- See Note [Pretty-printing UnitId] in "GHC.Unit" for more details.
      --
      -- Note that we use `FastString` instead of `UnitId` to avoid boring
      -- module inter-dependency issues.
```

When we're pretty printing a `UnitId` (the `Outputable` instance of `UnitId`), we fetch the `sdocUnitIdForUser` field which the last `pprWithUnitState` to set to use a `UnitState` to fetch information from.

The `GHC.Unit` module has a handful of notes quite relevant to these endeavours.
In fact, it's so relevant that while I don't have the notes database to cross reference, here are them:

```
Note [About units]
~~~~~~~~~~~~~~~~~~
Haskell users are used to manipulating Cabal packages. These packages are
identified by:
   - a package name :: String
   - a package version :: Version
   - (a revision number, when they are registered on Hackage)

Cabal packages may contain several components (libraries, programs,
testsuites). In GHC we are mostly interested in libraries because those are
the components that can be depended upon by other components. Components in a
package are identified by their component name. Historically only one library
component was allowed per package, hence it didn't need a name. For this
reason, component name may be empty for one library component in each
package:
   - a component name :: Maybe String

UnitId
------

Cabal libraries can be compiled in various ways (different compiler options
or Cabal flags, different dependencies, etc.), hence using package name,
package version and component name isn't enough to identify a built library.
We use another identifier called UnitId:

  package name             \
  package version          |                       ________
  component name           | hash of all this ==> | UnitId |
  Cabal flags              |                       --------
  compiler options         |
  dependencies' UnitId     /

Fortunately GHC doesn't have to generate these UnitId: they are provided by
external build tools (e.g. Cabal) with `-this-unit-id` command-line parameter.

UnitIds are important because they are used to generate internal names
(symbols, etc.).

Wired-in units
--------------

Certain libraries (ghc-prim, base, etc.) are known to the compiler and to the
RTS as they provide some basic primitives.  Hence UnitIds of wired-in libraries
are fixed. Instead of letting Cabal choose the UnitId for these libraries, their
.cabal file uses the following stanza to force it to a specific value:

   ghc-options: -this-unit-id ghc-prim    -- taken from ghc-prim.cabal

The RTS also uses entities of wired-in units by directly referring to symbols
such as "base_GHCziIOziException_heapOverflow_closure" where the prefix is
the UnitId of "base" unit.

Unit databases
--------------

Units are stored in databases in order to be reused by other codes:

   UnitKey ---> UnitInfo { exposed modules, package name, package version
                           component name, various file paths,
                           dependencies :: [UnitKey], etc. }

Because of the wired-in units described above, we can't exactly use UnitIds
as UnitKeys in the database: if we did this, we could only have a single unit
(compiled library) in the database for each wired-in library. As we want to
support databases containing several different units for the same wired-in
library, we do this:

   * for non wired-in units:
      * UnitId = UnitKey = Identifier (hash) computed by Cabal

   * for wired-in units:
      * UnitKey = Identifier computed by Cabal (just like for non wired-in units)
      * UnitId  = unit-id specified with -this-unit-id command-line flag

We can expose several units to GHC via the `package-id <unit-key>` command-line
parameter. We must use the UnitKeys of the units so that GHC can find them in
the database.

During unit loading, GHC replaces UnitKeys with UnitIds. It identifies wired
units by their package name (stored in their UnitInfo) and uses wired-in UnitIds
for them.

For example, knowing that "base", "ghc-prim" and "rts" are wired-in units, the
following dependency graph expressed with database UnitKeys will be transformed
into a similar graph expressed with UnitIds:

   UnitKeys
   ~~~~~~~~                      ----------> rts-1.0-hashABC <--
                                 |                             |
                                 |                             |
   foo-2.0-hash123 --> base-4.1-hashXYZ ---> ghc-prim-0.5.3-hashUVW

   UnitIds
   ~~~~~~~               ---------------> rts <--
                         |                      |
                         |                      |
   foo-2.0-hash123 --> base ---------------> ghc-prim


Note that "foo-2.0-hash123" isn't wired-in so its UnitId is the same as its UnitKey.

[...]
```

The cabal-install created unit-id for packages takes a lot of flags and properties into consideration. What exactly do we need to match?

The issue is that when loading plugins we don't check if the `ghc` the plugin was compiled against is the same `ghc` the compiler loading the plugin compiles against. To do this check, we must first have a unique id that is different when the two `ghc`s differ, and is equal when the two `ghc`s are the same, and we can use the ghc's unit-id computed by cabal for that effect. However, we currently compile `ghc` the library with `-this-unit-id ghc`, because `ghc` is a wired-in package and we assume it will have exactly the unit id "`ghc`" in order to hardcode `ghc` names such as `Plugin`. Our idea is to not pass `-this-unit-id ghc` when compiling ghc (or any other wired-in package for that matter), and rather let `cabal` pass `-this-unit-id package-version-hash` as it does for all packages (that don't override it). In this sense, the wired-in Names must use the unit-id cabal will pass when compiling this package (and thus know it ahead of time), such that when we install the package, the wired-in names will refer to the actual unit-id of the package, rather than to non-existing unit-ids.

Would the check be done automatically by cabal?

