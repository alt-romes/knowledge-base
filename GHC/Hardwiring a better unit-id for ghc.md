#ghc 

To have a better unit-id for ghc we will hardcode the unit-id in a build-system-generated file. By importing it, we can know during compilation the hardwired unit-id of ghc and use it for the wired-in module name of ghc (`thisGhcUnitId`). We also need to know the hardwired unit-id of ghc at runtime, and again, we can use the generated file declarations for this.

Plan:
* Create a declaration with a fixed unit-id in either `GHC.Version` or `GHC.Settings.Config` (which one should it be?).
* There are multiple places where we generate these hardcoded files, which ones must we change?
	* `compiler/Setup.hs` creates the `GHC.Settings.Config` file based on the `ghc --info` output, why exactly? It also means we must change the `ghc --info` output to account for the unit-id?
	* `hadrian/src/Rules/Generate.hs` creates both `Version` and `Config` files from the `system.config` file / from the configure system
	* `libraries/ghc-boot/Setup.hs` doesn't actually generate `GHC.Settings.Config`, only generates `GHC.Version` and platform contants
* Change the wired-in unit-id to the configured one
* Ensure ghc is being compiled with `-this-unit-id the-configure-id` rather than the fixed `-this-unit-id ghc`, i.e. delete the latter line from `compiler/ghc.cabal`
* Ensure when a plugin is loaded, the unit-id of the compiler that is loading is the same as the unit-id of the compiler the plugin depends on.

To add a unit-id to the hadrian generated files, we add a setting option to the `Setting` in `src/Oracle/Settings`. Each setting comes from `hadrian/cfg/system.config` which is generated from `hadrian/cfg/system.config.in`.
Second, we change `system.config.in` to include our new option, and, then, we change `configure.ac` to fill it in correctly.

We must also eventually change `pkgIdentifier` in `Hadrian.Haskell.Cabal` to include a hash instead of just the name and the version. `pkgIdentifier` reads the cabal file to determine the name and version. On the contrary, `packageGhcArgs` in `Settings.Builders.GHC` reads a `ghc_version` from the settings which are defined through `system.config` (meaning: if we can find how hadrian sets the `project-version` config setting, we can also set `project-unit-id` and use it later on throughout the build)

The values in `system.config.in` and other `*.in` files (e.g. `@ProjectVersion@`) are set by the `./configure` script (and substituted as defined in `configure.ac`?). Perhaps having the unit-id be in the configuration stage is not appropriate, and hadrian better generate it separately from the configure script.

Instead of extend `Setting` in hadrian, we generate the identifier elsewhere in hadrian and do not get it from `system.config`.

The first step is to successfully compile with any other unit-id, like ghc-version. After that works we can elaborate the unit-id creation in hadrian to something more complicated -- the cabal-install hash.

Here it is: [!10119](https://gitlab.haskell.org/ghc/ghc/-/merge_requests/10119)

TODO: Write a Note about this. Note [ghc the library's unit-id]

Idea for testing: check the build artifact `GHC.Settings.Config` has a unit-id matching the one cabal gives it?