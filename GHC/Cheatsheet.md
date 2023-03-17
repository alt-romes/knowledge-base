#ghc 

### Development

Matthew suggested the flavour is `default+no_profiled_libs+omit_pragmas` is a good default
```
./hadrian/build -j --freeze1 --flavour=default+no_profiled_libs+omit_pragmas
```

### Building the user guide

The documentation is built with sphinx, called by hadrian:
```
./hadrian/build -j docs
```

But could be called directly with:
```
sphinx-build docs/users_guide/ <output_dir>
```

### Packages

Reading an interface file
```
ghc --show-iface Interface/File.hi
```

Inspecting a package database
```
ghc-pkg list --package-db=_build/stage0/inplace/package.conf.d
```

* To display something related to the "true" names of the unit-ids that ??? re-run the previous ghc command with `-v4`.

