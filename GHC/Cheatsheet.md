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


---

### Debugging

```
ghc -fforce-recomp -dlint -g3 -ddump-to-file -ddump-simpl -ddump-stg -debug -ddump-cmm -dsuppress-all
```


```
-dtag-inference-checks
```

Run with
```
./Main +RTS -DS
```
To get sanity checking (debugging sanity of the heap)

After compiling with `-g` and `-debug`, able to run `gdb ./Main` to debug

`-g` enables DWARF debugging information (DWARF compiler information for debuggers)
`-debug` links against the debug runtime system

Users guide: Debugging the compiler.

Use gdb bc we wanna know where exactly it is crashing
```
gdb ./Main
> run
> x/8a $rbp -- look at haskell's stack pointer in the bp register (that's what we use it for)
> disassemble
> x/8a $r14 -- lookup $r14
```

ATAT syntax
```
mov 0xe(%r14),%rax
```
Move from %r14+0xe (where 0xe is the offset) to %rax


---

Pointer tagging:
When we know all haskell objects in the haskell heap are aligned
to the machine's word size (8bytes).
The bottom 3 bits (why??) are always going to be zero.
We can encode the index of the constructor in the tag bits (the zero bits in the pointer)
(Just would have index 2, Nothing would have index 1)
We can immediately know which constructor it is from the pointer tag.

* If we find a zero tagged pointer, we jump to the entry code of the thunk.
* If we find a non-zero tagged pointer, 

We have to be sure to untag the pointer before dereferencing
```
and $0x7,%eax -- getting the tag!
cmp $0x1,%rax -- checking whether the tag is == 1
```

A pointer tag of zero means that the pointer points to a thunk
Bumping pointer, points to the next free block

Garbage collector gives contiguous block of memory, until we hit the heap
pointer limit.

A thunk consists of a header w an info table pointer pointer and a body with the
payload which are the free variables of the thunk. What code do we execute for
the thunk?

Historically, GHC stored this information (the pointer to the function entry
code) in the info table pointer (for the benefit of the garbage collector?)

However, this is a bit innefficient (2 jumps).
The optimization is called Tables Next To Code, which makes the thunk pointer to
the info entry table actually point to the code. Because the info table has a
fixed size, we can subtract from the code pointer to get the info table entry

This means that to jump to a thunk, given its zero-tagged pointer, we simply
call the function at pointer.

Using [`rr`](https://github.com/rr-debugger/rr) project, which replays things. "Time travelling debugger".
We can use `reverse-stepi`.

We do stack checks and heap checks before we do allocations to make sure we have
enough space.

Caches are usually partitioned in instruction caches and 
There's a paper about this info table entry being directly before the code
pointer.

Tag inference is the process that attributes tags to pointers pointing to data
types.
Tag inference imposed a new invariant called tag invariant: if we have a pointer
that we know to be evaluated, then it must be tagged. Previously we allowed
things that were evaluated to carry a tag of 0

Tag inference is a pass on STG which essentially propagates information about
the tag that pointers carry, attached to each identifier what we know about each
tag. It's a static analysis that allows us to eliminate many tag checks.

The tag invariant allows us to assume the unlifted type pointer is tagged with
something because unlifted types are always evaluated (so the tag could never be
zero)

Notes on #23146:

* It looks like it's not handling correctly tag inference on top
    level unlifted types.
* The pointer to UNil should be tagged with 1, since UNil is the first
    constructor which is evaluated (it just is evaluated, and couldn't even not
    be evaluated since it's an unlifted type -- boxed but always evaluated)
* Ben: We probably assume that unlifted constructors are tagged


`lldb` debugger for ARM based machines
