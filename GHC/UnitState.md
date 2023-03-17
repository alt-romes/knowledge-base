#ghc 

The unit state puts together the package databases and handles things like wired-in unit ids to create a more uniform interface to unit ids in general, which is then used, e.g., for pretty printing unit-ids (see [[Pretty printing unit-ids]])
The unit state is created by `mkUnitState` which states:

```
   Plan.

   There are two main steps for making the package state:

    1. We want to build a single, unified package database based
       on all of the input databases, which upholds the invariant that
       there is only one package per any UnitId and there are no
       dangling dependencies.  We'll do this by merging, and
       then successively filtering out bad dependencies.

       a) Merge all the databases together.
          If an input database defines unit ID that is already in
          the unified database, that package SHADOWS the existing
          package in the current unified database.  Note that
          order is important: packages defined later in the list of
          command line arguments shadow those defined earlier.

       b) Remove all packages with missing dependencies, or
          mutually recursive dependencies.

       b) Remove packages selected by -ignore-package from input database

       c) Remove all packages which depended on packages that are now
          shadowed by an ABI-incompatible package

       d) report (with -v) any packages that were removed by steps 1-3

    2. We want to look at the flags controlling package visibility,
       and build a mapping of what module names are in scope and
       where they live.

       a) on the final, unified database, we apply -trust/-distrust
          flags directly, modifying the database so that the 'trusted'
          field has the correct value.

       b) we use the -package/-hide-package flags to compute a
          visibility map, stating what packages are "exposed" for
          the purposes of computing the module map.
          * if any flag refers to a package which was removed by 1-5, then
            we can give an error message explaining why
          * if -hide-all-packages was not specified, this step also
            hides packages which are superseded by later exposed packages
          * this step is done TWICE if -plugin-package/-hide-all-plugin-packages
            are used

       c) based on the visibility map, we pick wired packages and rewrite
          them to have the expected unitId.

       d) finally, using the visibility map and the package database,
          we build a mapping saying what every in scope module name points to.
```

In its body we have

```haskell
  -- Sort out which packages are wired in. This has to be done last, since
  -- it modifies the unit ids of wired in packages, but when we process
  -- package arguments we need to key against the old versions.
  --
  (pkgs2, wired_map) <- findWiredInUnits logger prec_map pkgs1 vis_map2
```

The function `findWiredInUnits` will find the wired-in units and maps them to their wired-in unit ids, by returning the complete mapping from found unit-ids to the wired-in unit-ids.
* For each of the wired-in packages `wiredInUnitIds` in `GHC.Unit.Types`, find the corresponding wired-in unity in the list of packages previously computed as explained in the plan above.
* The finding is defined by a `match` function called on the wired-in package and on each of the available packes sorted by the "preferred" order.
* The `match` function compares the package name gotten from the `UnitInfo` found in the package database against the exact name of the wired-in package, confirming that the package name/unit-id in the `UnitInfo` of the wired-in packages as they are installed is the same as the wired-in name.
* The mappings can be seen by calling ghc with `-v2`

The field `unitInfoMap` in the `UnitState` maps `UnitId`s to `UnitInfo`s. When compiling `ghc`, the map is populated with:
```
Cabal-3.9.0.0, Cabal-syntax-3.9.0.0, array-0.5.5.0, base,
    binary-0.8.9.1, bytestring-0.11.4.0, containers-0.6.7,
    deepseq-1.4.8.1, directory-1.3.8.1, exceptions-0.10.7,
    filepath-1.4.100.1, ghc, ghc-bignum, ghc-boot-9.7, ghc-boot-th-9.7,
    ghc-heap-9.7, ghc-prim, ghci-9.7, haskeline-0.8.2.1, hpc-0.6.2.0,
    mtl-2.3.1, parsec-3.1.16.1, pretty-1.1.3.6, process-1.6.17.0, rts,
    stm-2.5.1.0, system-cxx-std-lib-1.0, template-haskell,
    terminfo-0.4.1.6, text-2.0.2, time-1.12.2, transformers-0.6.1.0,
    unix-2.8.1.0, xhtml-3000.2.2.1
```

Q: Where does the unitInfoMap find 