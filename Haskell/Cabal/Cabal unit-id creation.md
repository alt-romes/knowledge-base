#ghc #cabal 

Cabal computes a hash that uniquely identifies a package by version and compilation options/interface abi, which it then passes to ghc compiling the package units with -this-unit-id.

Q: How does cabal compute abi hashes for installed units?
A: It calls ghc with --abi-hash on all exposed modules listed in the interface (See `libraries/Cabal/src/Distribution/Simple/Register.hs`)

Q: How does cabal compute unit-ids for installed units?
A: The hash calculated by `hashPackageHashInputs` in `cabal-install:Distribution.Client.PackageHash`, which takes into consideration some `PackageHashInputs`. The full unit-id is put together by `hashedInstalledPackageId` in the same module, which also depends on some `PackageHashInputs`. The unit-id (`InstalledPackageId` in cabal-install) is different depending on the operating system because of name-length constraints (on windows we use shorter names, and on macos we use names even shorter than on windows).

How can we recreate the unit-id of a package?
* `renderPackageHashInputs` does laboured rendering of the `PackageHashInputs` to a string. This logic is needed to compute the unit-id
* `hashedInstalledPackageId` creates the unit-id (longer or shorter) depending on the platform. This is also needed to recreate a unit-id (perhaps not needed for `ghc` which is already short)
* We need a SHA256 function to hash the rendered package hash inputs
* We need the `PackageHashInputs` as they exist at cabal-install time

The logic of nix-style hashes is in `Distribution.Client.PackageHash` in `cabal-install`