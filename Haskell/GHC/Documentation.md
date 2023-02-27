#ghc 

### Development

Matthew suggested the flavour is `default+no_profiled_libs+omit_pragmas` is a good default
```
./hadrian/build -j --flavour=default+no_profiled_libs+omit_pragmas
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

