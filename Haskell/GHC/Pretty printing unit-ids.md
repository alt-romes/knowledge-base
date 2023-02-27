#ghc 

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

The `GHC.Unit` module has a handful of notes quite relevant to these endeavours. See the [[Note {About units}]]

Ultimately we have:

```haskell
-- | Pretty-print a UnitId for the user.
--
-- Cabal packages may contain several components (programs, libraries, etc.).
-- As far as GHC is concerned, installed package components ("units") are
-- identified by an opaque UnitId string provided by Cabal. As the string
-- contains a hash, we don't want to display it to users so GHC queries the
-- database to retrieve some infos about the original source package (name,
-- version, component name).
--
-- Instead we want to display: packagename-version[:componentname]
--
-- Component name is only displayed if it isn't the default library
--
-- To do this we need to query a unit database.
pprUnitIdForUser :: UnitState -> UnitId -> SDoc
pprUnitIdForUser state uid@(UnitId fs) =
   case lookupUnitPprInfo state uid of
      Nothing -> ftext fs -- we didn't find the unit at all
      Just i  -> ppr i
```
