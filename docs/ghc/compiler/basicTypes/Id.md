[[src]](https://github.com/ghc/ghc/tree/master/compiler/basicTypes/Id.hs)

(c) The University of Glasgow 2006
(c) The GRASP/AQUA Project, Glasgow University, 1992-1998

# @Ids@: Value and constructor identifiers

# \subsection{Basic Id manipulation}


# \subsection{Simple Id construction}


Absolutely all Ids are made by mkId.  It is just like Var.mkId,
but in addition it pins free-tyvar-info onto the Id's type,
where it can easily be found.

### Note: Free type variables

At one time we cached the free type variables of the type of an Id
at the root of the type in a TyNote.  The idea was to avoid repeating
the free-type-variable calculation.  But it turned out to slow down
the compiler overall. I don't quite know why; perhaps finding free
type variables of an Id isn't all that common whereas applying a
substitution (which changes the free type variables) is more common.
Anyway, we removed it in March 2008.



Make some local @Ids@ for a template @CoreExpr@.  These have bogus
@Uniques@, but that's OK because the templates are supposed to be
instantiated before use.


### Note: Exported LocalIds

We use mkExportedLocalId for things like
 - Dictionary functions (DFunId)
 - Wrapper and matcher Ids for pattern synonyms
 - Default methods for classes
 - Pattern-synonym matcher and builder Ids
 - etc

They marked as "exported" in the sense that they should be kept alive
even if apparently unused in other bindings, and not dropped as dead
code by the occurrence analyser.  (But "exported" here does not mean
"brought into lexical scope by an import declaration". Indeed these
things are always internal Ids that the user never sees.)

It's very important that they are *LocalIds*, not GlobalIds, for lots
of reasons:

 * We want to treat them as free variables for the purpose of
   dependency analysis (e.g. CoreFVs.exprFreeVars).

 * Look them up in the current substitution when we come across
   occurrences of them (in Subst.lookupIdSubst). Lacking this we
   can get an out-of-date unfolding, which can in turn make the
   simplifier go into an infinite loop (Trac #9857)

 * Ensure that for dfuns that the specialiser does not float dict uses
   above their defns, which would prevent good simplifications happening.

 * The strictness analyser treats a occurrence of a GlobalId as
   imported and assumes it contains strictness in its IdInfo, which
   isn't true if the thing is bound in the same module as the
   occurrence.

In CoreTidy we must make all these LocalIds into GlobalIds, so that in
importing modules (in --make mode) we treat them as properly global.
That is what is happening in, say tidy_insts in TidyPgm.

# \subsection{Special Ids}


### Note: Primop wrappers

Currently hasNoBinding claims that PrimOpIds don't have a curried
function definition.  But actually they do, in GHC.PrimopWrappers,
which is auto-generated from prelude/primops.txt.pp.  So actually, hasNoBinding
could return 'False' for PrimOpIds.

But we'd need to add something in CoreToStg to swizzle any unsaturated
applications of GHC.Prim.plusInt# to GHC.PrimopWrappers.plusInt#.

Nota Bene: GHC.PrimopWrappers is needed *regardless*, because it's
used by GHCi, which does not implement primops direct at all.


# Evidence variables


# Join variables


# \subsection{IdInfo stuff}



        ---------------------------------
        -- INLINING
The inline pragma tells us to be very keen to inline this Id, but it's still
OK not to if optimisation is switched off.



        ---------------------------------
        -- ONE-SHOT LAMBDAS


### Note: transferPolyIdInfo

This transfer is used in two places:
        FloatOut (long-distance let-floating)
        SimplUtils.abstractFloats (short-distance let-floating)

Consider the short-distance let-floating:

   f = /\a. let g = rhs in ...

Then if we float thus

   g' = /\a. rhs
   f = /\a. ...[g' a/g]....

we *do not* want to lose g's
  * strictness information
  * arity
  * inline pragma (though that is bit more debatable)
  * occurrence info

Mostly this is just an optimisation, but it's *vital* to
transfer the occurrence info.  Consider

   NonRec { f = /\a. let Rec { g* = ..g.. } in ... }

where the '*' means 'LoopBreaker'.  Then if we float we must get

   Rec { g'* = /\a. ...(g' a)... }
   NonRec { f = /\a. ...[g' a/g]....}

where g' is also marked as LoopBreaker.  If not, terrible things
can happen if we re-simplify the binding (and the Simplifier does
sometimes simplify a term twice); see Trac #4345.

It's not so simple to retain
  * worker info
  * rules
so we simply discard those.  Sooner or later this may bite us.

If we abstract wrt one or more *value* binders, we must modify the
arity and strictness info before transferring it.  E.g.
      f = \x. e
-->
      g' = \y. \x. e
      + substitute (g' y) for g
Notice that g' has an arity one more than the original g
