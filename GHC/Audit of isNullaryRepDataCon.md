In the context of [!10165](https://gitlab.haskell.org/ghc/ghc/-/merge_requests/10165)

* Occurrence in `mkCase2` in `GHC.Core.Opt.Simplify.Utils`:

When doing the constant folding of `dataToTag x` as described in `Note [caseRules for dataToTag]`, we must create bindings for the value arguments of any non-nullary data constructor. Here, the definition of `isNullaryRepDataCon` matches the semantics wanted at its call site (coercion arguments ==> is not nullary)., coercions are value arguments for which we must generate a binder when constant folding `dataToTag x`.
That is, that the function `f`
```haskell
data D x where
  A :: D Int
  B :: D ()
  C :: D Bool

f :: D x -> Int
f x = case dataToTag# x of
         0# -> 0
         1# -> 1
         2# -> 2
{-# NOINLINE f #-}
```
should be (and is) optimized to
```haskell
Main.$wf
  = \ (@x_s1HA) (x1_s1HB :: D x_s1HA) ->
      case x1_s1HB of {
        A ipv_sSs -> 0#;
        B ipv_sSu -> 1#;
        C ipv_sSw -> 2#
      }
```


* Occurrence in `isTagged` in `GHC.Stg.InferTags.Rewrite`:

For *imported* data constructor workers (we currently don't consider the wrappers, should we?), `isTagged`, if the data constructor worker is nullary with the current semantics, i.e., there are no value arguments, including no coercion arguments, will return `True` to say the imported data constructor worker is tagged. For data con workers with arguments (including coercion arguments), and for data con wrappers, `isTagged` will look at the `LFInfo` of the data con.

This occurrence of `isNullaryRepDataCon` will have to match the logic for tagging data con workers. If we tag data con workers with coercion arguments (which I think we should), then this occurrence should not only consider data con workers without coercion arguments.
Another question is whether we should tag/consider tagged data con *wrappers*.

* Occurrence in `lookupInfo` in `GHC.Stg.InferTags.Types`:

I'm not sure what this function is doing, but it seems that for nullary data con workers without coercion arguments we will return `TagProper`, while for those with coercion arguments we will again check the `LFInfo`.

* Occurrence in `schemeTopBind` in `GHC.StgToByteCode`:

The function `schemeTopBind` compiles code for the right-hand side of a top-level binding.
Before compiling the RHS through `schemeR`, it considers a special case -- the top level binding being a *nullary* data con worker, with this explanation:
```haskell
-- Special case for the worker of a nullary data con.
-- It'll look like this:        Nil = /\a -> Nil a
-- If we feed it into 'schemeR', we'll get
--      Nil = Nil
-- because 'mkConAppCode' treats nullary constructor applications
-- by just re-using the single top-level definition.  So
-- for the worker itself, we must allocate it directly.
```
I suspect feeding a constructor that just has coercion arguments will result in something similar, but I have yet to inspect the bytecode.
```haskell
data D x where
    D :: D ()
-------
-- D = /\x -> \co::(x ~# ()) -> D x
-- And if we feed it into 'schemeR', won't we get
-- D = D
-- ?
-- Note [Post unarisation invariants] states that DataCon applications don't have void arguments; and indeed in `mkConAppCode` we filter out void args...
```
Meaning in this case, we probably want to consider constructors only with coercions as nullary data cons... which we currently don't.

(As for only considering *workers*, I think that is correct, though I'm not sure how `mkConAppCode` is dealing with *wrappers*).

* Occurrence in `pushAtom` in `GHC.StgToByteCode`:

`pushAtom` pushes an atom onto the stack. In the case of a global variable, we normally use `PUSH_G` to push the var onto the stack, however, `PUSH_G` doesn't tag constructors. Therefore, if the global variable is a data constructor worker, we `PACK` the the constructor -- but we also assert that this data con worker is nullary:
<details>

```
-- PUSH_G doesn't tag constructors. So we use PACK here
-- if we are dealing with nullary constructor.
case isDataConWorkId_maybe var of
  Just con -> do
	massert (isNullaryRepDataCon con)
	return (unitOL (PACK con 0), szb)

  Nothing
	-- see Note [Generating code for top-level string literal bindings]
	| isUnliftedType (idType var) -> do
	  massert (idType var `eqType` addrPrimTy)
	  return (unitOL (PUSH_ADDR (getName var)), szb)

	| otherwise -> do
	  return (unitOL (PUSH_G (getName var)), szb)
```

</details>
I'm not sure why all global data con workers are  seemingly nullary (which is what we assert).
What are the semantics of `isNullaryRepDataCon` here? I don't see why data cons workers only with coercion arguments wouldn't be tagged (as per the post unarisation invariants, all applications are to non-void arguments, so effectively in STG all applications of nullary data con workers, whether they have or not coercion arguments, will be trivially saturated (0 arguments)).

What about data con *wrappers* and non-nullary data cons here?

* Occurrence in `precomputedStaticConInfo_maybe`:

`precomputedStaticConInfo_maybe` checks if a given constructor application
can be replaced with a reference to a existing static closure.

In this function, an application of a data con (worker or wrapper) that takes zero non-void arguments and is nullary (in the sense it takes no coercion arguments either), is replaced with a reference to an existing static closure. However, data cons that take coercion arguments could similarly be replaced with a reference to an existing static closure. This is the object of attention at [#23158](https://gitlab.haskell.org/ghc/ghc/-/issues/23158).

I believe we might want to simply omit this check here.


* Occurrence in `mkLFImported`:

The root of [#23146](https://gitlab.haskell.org/ghc/ghc/-/issues/23146).

In `mkLFImported` we get the `LambdaFormInfo` of an imported `Id`. If the interface has an `LFInfo` we simply return it, otherwise, we make a conservative LFInfo from the type.
In making a conservative LFInfo, if we have a data con *worker* with no arguments whatsoever (including no coercion arguments), then we return `LFCon con`. However, I believe we can be less conservative here, since data con workers (and wrappers too, right?) that are nullary except for coercion arguments (to which we never apply the data con) have a `LFCon con` LFInfo too.

See also the comment https://gitlab.haskell.org/ghc/ghc/-/merge_requests/10165#note_489340
