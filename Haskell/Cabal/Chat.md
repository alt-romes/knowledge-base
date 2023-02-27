#ghc #cabal

1.  romes
    Hi 👋 I’m trying to understand if the code in cabal-install to compute a package’s unit-id (hashedInstalledPackageId in Distribution.Client.PackageHash) is too tied in to cabal-install or whether it could be feasibly moved to the Cabal package?
    On my current work on plugin abi incompatibility, everything points to a solution that involves having ghc compute its own unit-id exactly as cabal would. An added benefit is that this path ultimately also enables us to handle wired-in packages without the -this-unit-id package hack. So, right now, it seems like the elegant solution :)
    
    🎉1
    
3.  sclv
    i genuinely don't understand this
    
    like i never followed why the proposal of ghc just emitting a reasonable hash of any sort to its wired in package conf file wasn't enough
    
    ghc and cabal-the-library are logically prior to cabal-install's notion of a store
    
    and the wired in ghc package doesnt live in the store
    
7.  romes
    
    Currently, we might have compiled a plugin with a GHC and can load the same plugin with a different GHC and there are no checks to prevent this.
    
    My immediate goal is to prevent _incompatible_ plugins from being loaded.
    
    I've written a paragraph on my current understanding of this all, but if I'm mistaken here, Matthew is still quite sure, regardless, that we need the unit-id to match cabal's:
    
    To have this check, we must first have a unique id that is different when the two ghcs differ, and is equal when the two ghcs are the same. However, we currently compile ghc the library with -this-unit-id ghc, because ghc is a wired-in package and we assume it will have exactly the unit id "ghc" in order to hardcode ghc names such as Plugin. Our idea is to not pass -this-unit-id ghc when compiling ghc (or any other wired-in package for that matter), and rather let cabal pass -this-unit-id package-version-hash as it does for all packages (that don't override it). In this sense, the wired-in Names must use the unit-id cabal will pass when compiling this package (and thus know it ahead of time), such that when we install the package, the wired-in names will refer to the actual unit-id of the package, rather than to non-existing unit-ids.
    
8.  sclv
    
    ok so here's where i think the confusion is
    
    "In this sense, the wired-in Names must use the unit-id cabal will pass when compiling this package (and thus know it ahead of time), such that when we install the package, the wired-in names will refer to the actual unit-id of the package, rather than to non-existing unit-ids."
    
    all you need is _some_ unique id to pass when compiling the package which is known ahead of time.
    
    and ghc itself can calculate that unitid however it likes!
    
13.  romes
    That bit is the conclusion to assuming that "we'll have cabal pass the -this-unit-id and we will not overwrite it". The contention point is whether we do need/want cabal to pass -this-unit-id or if it's a good solution to overwrite it still but with a hash that will serve to prevent us from loading plugins that are linked to incompatible ghc-the-libraries
    
14.  sclv
    you say "let cabal pass -this-unit-id package-version-hash as it does for all packages" but i don't think this is true?
    only cabal new build does this
    for packages written to the store
    i want to emphasize again ghc _is not written to the store_
    so there's no "overwriting" because this is not a package where cabal installs it to the store. it is in the global packagedb
    you can just pick whatever packageid you want for it, so just like.. pick one that has whatever hash the ghc team desires. done!
    
20.  romes
     I'll review this tomorrow and I'll get back to you. I see what you mean. Yet I have an initial intuition for them needing to be the same and Matthew is really confident on it. I'll get to the bottom of why/or why not we need them to be the same.
    
21.  sclv
    i think matthew is wrong. everything that happens in the ghc compilation pipeline can happen without hashes, and can happen without cabal-install. it is all logically prior to the store.
    keeping that phase distinction is architecturally nice, if possible, imho.
    and certainly if we want to be "agnostic" about how things work then we need it -- tooling can and does exist which is _also_ ignorant of the store, and just works with packagedbs directly, and all that tooling is necessarily ignorant of cabal's hashes as well
    
24.  romes
    Sounds convincing! I'll have to gather my thoughts. Thanks though.
    
26.  sclv
    glad it was potentially helpful, i'm happy people are looking at sorting this stuff out.