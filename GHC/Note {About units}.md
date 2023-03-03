#ghc 

```
Note [About units]
~~~~~~~~~~~~~~~~~~
Haskell users are used to manipulating Cabal packages. These packages are
identified by:
   - a package name :: String
   - a package version :: Version
   - (a revision number, when they are registered on Hackage)

Cabal packages may contain several components (libraries, programs,
testsuites). In GHC we are mostly interested in libraries because those are
the components that can be depended upon by other components. Components in a
package are identified by their component name. Historically only one library
component was allowed per package, hence it didn't need a name. For this
reason, component name may be empty for one library component in each
package:
   - a component name :: Maybe String

UnitId
------

Cabal libraries can be compiled in various ways (different compiler options
or Cabal flags, different dependencies, etc.), hence using package name,
package version and component name isn't enough to identify a built library.
We use another identifier called UnitId:

  package name             \
  package version          |                       ________
  component name           | hash of all this ==> | UnitId |
  Cabal flags              |                       --------
  compiler options         |
  dependencies' UnitId     /

Fortunately GHC doesn't have to generate these UnitId: they are provided by
external build tools (e.g. Cabal) with `-this-unit-id` command-line parameter.

UnitIds are important because they are used to generate internal names
(symbols, etc.).

Wired-in units
--------------

Certain libraries (ghc-prim, base, etc.) are known to the compiler and to the
RTS as they provide some basic primitives.  Hence UnitIds of wired-in libraries
are fixed. Instead of letting Cabal choose the UnitId for these libraries, their
.cabal file uses the following stanza to force it to a specific value:

   ghc-options: -this-unit-id ghc-prim    -- taken from ghc-prim.cabal

The RTS also uses entities of wired-in units by directly referring to symbols
such as "base_GHCziIOziException_heapOverflow_closure" where the prefix is
the UnitId of "base" unit.

Unit databases
--------------

Units are stored in databases in order to be reused by other codes:

   UnitKey ---> UnitInfo { exposed modules, package name, package version
                           component name, various file paths,
                           dependencies :: [UnitKey], etc. }

Because of the wired-in units described above, we can't exactly use UnitIds
as UnitKeys in the database: if we did this, we could only have a single unit
(compiled library) in the database for each wired-in library. As we want to
support databases containing several different units for the same wired-in
library, we do this:

   * for non wired-in units:
      * UnitId = UnitKey = Identifier (hash) computed by Cabal

   * for wired-in units:
      * UnitKey = Identifier computed by Cabal (just like for non wired-in units)
      * UnitId  = unit-id specified with -this-unit-id command-line flag

We can expose several units to GHC via the `package-id <unit-key>` command-line
parameter. We must use the UnitKeys of the units so that GHC can find them in
the database.

During unit loading, GHC replaces UnitKeys with UnitIds. It identifies wired
units by their package name (stored in their UnitInfo) and uses wired-in UnitIds
for them.

For example, knowing that "base", "ghc-prim" and "rts" are wired-in units, the
following dependency graph expressed with database UnitKeys will be transformed
into a similar graph expressed with UnitIds:

   UnitKeys
   ~~~~~~~~                      ----------> rts-1.0-hashABC <--
                                 |                             |
                                 |                             |
   foo-2.0-hash123 --> base-4.1-hashXYZ ---> ghc-prim-0.5.3-hashUVW

   UnitIds
   ~~~~~~~               ---------------> rts <--
                         |                      |
                         |                      |
   foo-2.0-hash123 --> base ---------------> ghc-prim


Note that "foo-2.0-hash123" isn't wired-in so its UnitId is the same as its UnitKey.

[...]
```
