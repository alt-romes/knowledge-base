### Cross compiling GHC on MacOS to Windows

* Install mingw-w64 GNU cross-compilers
```
brew install mingw-w64
```
* Configure to target arch:x86_64, os:w64, abi:mingw32 (on the root of the
    cloned ghc repository)
* Use `llvm-ar` while [!22805](https://gitlab.haskell.org/ghc/ghc/-/issues/22805) is not fixed
* Drop the `if !os(windows) build-depends: unbuildable` from `libraries/Win32`
* Include mingw32 system libraries in paths
```
git clone --recurse-submodules git@gitlab.haskell.org:ghc/ghc.git
mv ghc x86_64-w64-mingw32-ghc
cd x86_64-w64-mingw32-ghc

AR_STAGE0=llvm-ar ./configure --target=x86_64-w64-mingw32

./hadrian/build -j
```

## Cross compiling host aarch64-apple-darwin to target x86_64-apple-darwin
