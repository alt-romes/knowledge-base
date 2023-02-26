### Building the user guide

The documentation is built with sphinx, called by hadrian:
```
./hadrian/build -j docs
```

But could be called directly with:
```
sphinx-build docs/users_guide/ <output_dir>
```

