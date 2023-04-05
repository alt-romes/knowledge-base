type NP :: [UnliftedType] -> UnliftedType
data NP xs where
  UNil :: NP '[]
  (::*) :: x -> NP xs -> NP (x ': xs)


fieldsSam :: NP xs -> NP xs -> Bool
fieldsSam (x' ::* xs) (y' ::* ys) = fieldsSam xs ys
fieldsSam UNil UNil = True


main :: IO ()
main = print (fieldsSam UNil UNil)




We get a wrapper for UNil:


$WUnil :: NP []
$WUNil = UNil @[] (Refl [])




==========


Invariant: At the end of STG (after "rewriting" = CBV-introduction) every call of a function f with CBV argument passes an "evaluated-and-tagged" argument.  


* CBV-introduction pass decides, for every function, which arguments are CBV.
   * InferTags does this, I think.
* Look at GHC.Types.Id.Info, line 189.   Two IdDetails (JoinId and WorkerLikeId) have [CbvMarks].   
   * "WorkerLike" is a terrible name.
   * The [CbvMark] is always empty (and ignored) until after Tidy for ids from the current module.   Nothing to do with Core!  But it must be visible (somehow) to importing modules: it's part of the calling convention.
   * Callers must ensure argument is properly tagged (when CbvMark = Marked) (Extra evals are added by the "STG rewrite" pass)
   * Example:   data T a = MkT !a;  f x y = MkT x.   (MkT is the worker, not the wrapper.)   'f' allocates a MkT and returns it.  But MkT has a Cbv argument, so it must evald and properly tagged. So "STG rewrite" pass rewrites this to
     f x y = case x of x' -> MkT x'


* LFInfo used to be solely part of codegen: StgToCmm. 
   * Just keeps track, for each binder, of what it is bound to.
   * More recently, we seem to be exporting it in the interface file. 
   * We should understand more clearly why and how.  It's basically a good thing: it exposes calling-convention info to importing modules.
      * This is in Note [Conveying CAF-info and LFInfo between modules].  Covers:
         * CAFInfo
         * LambdaFormInfo
To me, CbvMarks belong here too.
      * "We record the CgInfo in the IdInfo of the Id."  Not true directly, but I see lfInfo :: Maybe LambdaFormInfo, and bitfield contains CAFInfo.
      * Reference to updateModDetailsIdInfos is wrong.  No such function. Nor any module GHC.Iface.UpdateIdInfos.  Fix.
      * Question: is LFInfo always serialised (regardless of optimisation level etc) or only sometimes?  If the latter we need a conservative approximation when importing.  This point needs documenting.
      * Answer: No, not always, Andreas said with -O0 we don’t serialize it.
      * But CbvMarks are not optional, and MUST be pushed though interface files.


(https://gitlab.haskell.org/ghc/ghc/-/wikis/commentary/rts/haskell-execution/pointer-tagging)
Pointer tagging is not optional, contrary to what the paper says. We originally planned that it would be: if the GC threw away all the tags, then everything would continue to work albeit more slowly. However, it turned out that in fact we really want to assume tag bits in some places:
* In the continuation of an algebraic case, R1 is assumed tagged
* On entry to a non-top-level function, R1 is assumed tagged


Q1: Does fieldsSam have CBV arguments?
* Simon's bet: yes it is thus marked, because when we compile fieldsSam itself we don't generate a tag=0 case.


fieldsSam x y = case x of { ... ; ... }
Without CBVmarks, fieldSam would check for
* tag=0: eval
* tag=1: jump to first alt
* tag=2: jump to second alt
If fieldSam has CBV arguments, we can omit the first check for tag=0.  We know that the tag is not zero.  And that saves the eval code.


Now think of the call site  (fieldSam $WUnil).  Just thinking of one arg for now.


If it was a Maybe.  (fieldSam Nothing). I think we'd pass an evaluated and properly tagged Nothing.  Yes we do.  We have a statically allocated Nothing_closure in the heap, and we pass its address + 1.   Nothing_closure itself is not evaluated-and-properly-tagged, but we can make EPT pointer from it by adding 1.   "STG rewrite" pass does not add an eval at the call site.


If we had (fieldSam UNil), where UNil is the worker, we'd do the same thing. Check.  The complication is that 
        UNil :: (a~#[]) => NP []


But we have the wrapper (fieldSam $WUnil)


Q2: why didn't we inline the wrapper?  Seems silly not to.


A: Perhaps for sharing


But nothing should go wrong if we don't in inline it.


presumably, then, the "STG rewrite" pass considers $WUnil to be evaluated-and-properly-tagged. (otherwise it would have added an eval).
Simultaneously, fieldsSam expects its argument to be evaluated-and-properly-tagged (due to CbvMarks?).


Q3: Why does "STG rewrite" pass consider $WNil to be evaluated-and-properly tagged at the call site (fieldSam $WUnil)?
* Because STG rewrite considers it to be EPT, it doesn’t add eval+tag and so the pointer is passed untagged to a function whose calling convention requires it to be tagged
* Question: does the worker UNil or wrapper $WUnil have IdInfo . lfInfo field set to Just xx
* Even for imported data types, Worker and Wrapper Ids are built by GHC.Types.Id.Make.mkDataConWorkId and mkDataConRep.  These don't seem to set lfIfno...  Check. It seems the lfInfo field is populated when deserializing from an interface, otherwise is Nothing
* Assuming they have no LFInfo, why does isTagged consider them EPT?
* isTagged considers them EPT because $WUNil has LFInfo LFUnlifted, however, when generating code for the $WUNil at the call site, we don’t tag it bc it is LFUnlifted (see litIdInfo)


Q4: top level unlifted things.  
* Core letrec invariant
* We need top level bindings
data T :: UnlifftedType = A Int | B
x = A 3 – not a valid definition (why not?)
                 y = B 
        These are OK top level bindings.
                x :: Addr# = if blah then "foo"# else "bar"#
* We can have top level unlifted bindings if:
   * Kind is (BoxedRep Unlifted)
   * RHS is a value
   * Check in lint, exploit in some way?
   * See also https://gitlab.haskell.org/ghc/ghc/-/issues/17521
   * This entails always serializing LFInfo (we’d require it)
      * We could perhaps get rid of LFUnknown if we go this route
* In fact, when we add the Worker bindings (in CorePrep)        
        Just = \x. StgConApp Just [x]
        Nothing = StgConApp Nothing []
Then we can do (map Just xs)
For UNil we allocate
        UNil = StgConApp UNil []    -- Top level unlifted binding!
        $WUNil   similarly: top level unlifted








In Stg   expr ::= let  var = rhs in expr
                | K arg1 ... argn       -- StgConApp
        rhs ::= K arg1 ... argn         -- StgRhsCon
                    |  fvs \ args. Expr               -- StgRhsClosure




Suppose g had a CBV argument
Call    g Just
Q: is Just evaluated-and-properly-tagged?
If not we would add an eval   case Just of j -> g j
But that's a bit silly!
Test.


data T a = MkT !a
foo = MkT Just
Do we store an  arity-tagged pointer in MkT?  I hope so.  Check




Idea:
*  in GHC.Stg.InferTags.Rewrite.isTagged, why do we check for dataConWorkId?
* Why not just use the LFInfo, which follows?
* Yes!  But then data con workers would have to HAVE lfInfo set.
* And in         GHC.Types.Id.Make we don't seem to give it lfInfo.  We should!
* For top level unlifted bindings we also require lf info always known and serialized


f :: forall a b. (a ~# b) => a -> b


A call looks like 
* In Core:   f @t1 @t2 co x
* In STG:   StgApp f [co, x]
Apparently for some reason (it's a bit inconsistent with StgApp) we drop the Void args in StgConApp.  StgConApp is guaranteed saturated so we don't need to keep those coercion args.
* In Note [Post-unarisation invariants]
 (3) DataCon applications (StgRhsCon and StgConApp) don't have void arguments.


We could always serialise LambdaFormInfo?
Part of the lambda form info is its calling convention.
LFInfo export is info needed for code generation