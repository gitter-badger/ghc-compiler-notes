[[src]](https://github.com/ghc/ghc/tree/master/compiler/typecheck/TcRnTypes.hs)

(c) The University of Glasgow 2006-2012
(c) The GRASP Project, Glasgow University, 1992-2002


Various types used during typechecking, please see TcRnMonad as well for
operations on these types. You probably want to import it, instead of this
module.

All the monads exported here are built on top of the same IOEnv monad. The
monad functions like a Reader monad in the way it passes the environment
around. This is done to allow the environment to be manipulated in a stack
like fashion when entering expressions... etc.

For state that is global and should be returned at the end (e.g not part
of the stack mechanism), you should use a TcRef (= IORef) to store them.


# LANGUAGE CPP, ExistentialQuantification, GeneralizedNewtypeDeriving,
             ViewPatterns #

# Standard monad definition for TcRn
    All the combinators for the monad can be found in TcRnMonad


The monad itself has to be defined here, because it is mentioned by ErrCtxt


# The interface environments
              Used when dealing with IfaceDecls


# Desugarer monad


Now the mondo monad magic (yes, @DsM@ is a silly name)---carry around
a @UniqueSupply@ and some annotations, which
presumably include source-file location information:


# Global typechecker environment


### Note: Tracking unused binding and imports

We gather two sorts of usage information

 * tcg_dus (defs/uses)
      Records *defined* Names (local, top-level)
          and *used*    Names (local or imported)

      Used (a) to report "defined but not used"
               (see RnNames.reportUnusedNames)
           (b) to generate version-tracking usage info in interface
               files (see MkIface.mkUsedNames)
   This usage info is mainly gathered by the renamer's
   gathering of free-variables

 * tcg_used_gres
      Used only to report unused import declarations

### Note: GRE filtering

# The local typechecker environment


### Note: The Global-Env/Local-Env story

During type checking, we keep in the tcg_type_env
        * All types and classes
        * All Ids derived from types and classes (constructors, selectors)

At the end of type checking, we zonk the local bindings,
and as we do so we add to the tcg_type_env
        * Locally defined top-level Ids

Why?  Because they are now Ids not TcIds.  This final GlobalEnv is
        a) fed back (via the knot) to typechecking the
           unfoldings of interface signatures
        b) used in the ModDetails of this module


### Note: Given Insts

Because of GADTs, we have to pass inwards the Insts provided by type signatures
and existential contexts. Consider
        data T a where { T1 :: b -> b -> T [b] }
        f :: Eq a => T a -> Bool
        f (T1 x y) = [x]==[y]

The constructor T1 binds an existential variable 'b', and we need Eq [b].
Well, we have it, because Eq a refines to Eq [b], but we can only spot that if we
pass it inwards.



# Node [RunSplice ThLevel]

The 'RunSplice' stage is set when executing a splice, and only when running a
splice. In particular it is not set when the splice is renamed or typechecked.

### Note: Delaying modFinalizers in untyped splices



### Note: Escaping the arrow scope

In arrow notation, a variable bound by a proc (or enclosed let/kappa)
is not in scope to the left of an arrow tail (-<) or the head of (|..|).
For example

        proc x -> (e1 -< e2)

Here, x is not in scope in e1, but it is in scope in e2.  This can get
a bit complicated:

        let x = 3 in
        proc y -> (proc z -> e1) -< e2

Here, x and z are in scope in e1, but y is not.

We implement this by
recording the environment when passing a proc (using newArrowScope),
and returning to that (using escapeArrowScope) on the left of -< and the
head of (|..|).

All this can be dealt with by the *renamer*. But the type checker needs
to be involved too.  Example (arrowfail001)
  class Foo a where foo :: a -> ()
  data Bar = forall a. Foo a => Bar a
  get :: Bar -> ()
  get = proc x -> case x of Bar a -> foo -< a
Here the call of 'foo' gives rise to a (Foo a) constraint that should not
be captured by the pattern match on 'Bar'.  Rather it should join the
constraints from further out.  So we must capture the constraint bag
from further out in the ArrowCtxt that we push inwards.


### Note: Meaning of IdBindingInfo

NotLetBound means that
  the Id is not let-bound (e.g. it is bound in a
  lambda-abstraction or in a case pattern)

ClosedLet means that
   - The Id is let-bound,
   - Any free term variables are also Global or ClosedLet
   - Its type has no free variables (NB: a top-level binding subject
     to the MR might have free vars in its type)
   These ClosedLets can definitely be floated to top level; and we
   may need to do so for static forms.

   Property:   ClosedLet
             is equivalent to
               NonClosedLet emptyNameSet True

(NonClosedLet (fvs::RhsNames) (cl::ClosedTypeId)) means that
   - The Id is let-bound

   - The fvs::RhsNames contains the free names of the RHS,
     excluding Global and ClosedLet ones.

### Note: Bindings with closed types

### Note: Grand plan for static forms

### Note: Bindings with closed types: ClosedTypeId

Consider

  f x = let g ys = map not ys
        in ...

Can we generalise 'g' under the OutsideIn algorithm?  Yes,
because all g's free variables are top-level; that is they themselves
have no free type variables, and it is the type variables in the
environment that makes things tricky for OutsideIn generalisation.

Here's the invariant:
   If an Id has ClosedTypeId=True (in its IdBindingInfo), then
   the Id's type is /definitely/ closed (has no free type variables).
   Specifically,
       a) The Id's acutal type is closed (has no free tyvars)
       b) Either the Id has a (closed) user-supplied type signature
          or all its free variables are Global/ClosedLet
             or NonClosedLet with ClosedTypeId=True.
          In particular, none are NotLetBound.

Why is (b) needed?   Consider
    \x. (x :: Int, let y = x+1 in ...)
Initially x::alpha.  If we happen to typecheck the 'let' before the
(x::Int), y's type will have a free tyvar; but if the other way round
it won't.  So we treat any let-bound variable with a free
non-let-bound variable as not ClosedTypeId, regardless of what the
free vars of its type actually are.

But if it has a signature, all is well:
   \x. ...(let { y::Int; y = x+1 } in
           let { v = y+2 } in ...)...
Here the signature on 'v' makes 'y' a ClosedTypeId, so we can
generalise 'v'.

Note that:

  * A top-level binding may not have ClosedTypeId=True, if it suffers
    from the MR

  * A nested binding may be closed (eg 'g' in the example we started
    with). Indeed, that's the point; whether a function is defined at
    top level or nested is orthogonal to the question of whether or
    not it is closed.

  * A binding may be non-closed because it mentions a lexically scoped
    *type variable*  Eg
        f :: forall a. blah
        f x = let g y = ...(y::a)...

Under OutsideIn we are free to generalise an Id all of whose free
variables have ClosedTypeId=True (or imported).  This is an extension
compared to the JFP paper on OutsideIn, which used "top-level" as a
proxy for "closed".  (It's not a good proxy anyway -- the MR can make
a top-level binding with a free type variable.)


# Operations over ImportAvails


# \subsection{Where from}


The @WhereFrom@ type controls where the renamer looks for an interface file


 SOURCE 

 SYSTEM 

 PLUGIN 

# Type signatures


### Note: Complete and partial type signatures

A type signature is partial when it contains one or more wildcards
(= type holes).  The wildcard can either be:
* A (type) wildcard occurring in sig_theta or sig_tau. These are
  stored in sig_wcs.
      f :: Bool -> _
      g :: Eq _a => _a -> _a -> Bool
* Or an extra-constraints wildcard, stored in sig_cts:
      h :: (Num a, _) => a -> a

A type signature is a complete type signature when there are no
wildcards in the type signature, i.e. iff sig_wcs is empty and
sig_extra_cts is Nothing.


### Note: sig_inst_tau may be polymorphic

### Note: Polymorphic methods

### Note: Wildcards in partial signatures

The wildcards in psig_wcs may stand for a type mentioning
the universally-quantified tyvars of psig_ty

E.g.  f :: forall a. _ -> a
      f x = x
We get sig_inst_skols = [a]
       sig_inst_tau   = _22 -> a
       sig_inst_wcs   = [_22]
and _22 in the end is unified with the type 'a'

Moreover the kind of a wildcard in sig_inst_wcs may mention
the universally-quantified tyvars sig_inst_skols
e.g.   f :: t a -> t _
Here we get
   sig_inst_skols = [k:*, (t::k ->*), (a::k)]
   sig_inst_tau   = t a -> t _22
   sig_inst_wcs   = [ _22::k ]


# 

### Note: Hole constraints

CHoleCan constraints are used for two kinds of holes,
distinguished by cc_hole:

  * For holes in expressions (including variables not in scope)
    e.g.   f x = g _ x

  * For holes in type signatures
    e.g.   f :: _ -> _
           f x = [x,True]

### Note: CIrredCan constraints

CIrredCan constraints are used for constraints that are "stuck"
   - we can't solve them (yet)
   - we can't use them to solve other constraints
   - but they may become soluble if we substitute for some
     of the type variables in the constraint

Example 1:  (c Int), where c :: * -> Constraint.  We can't do anything
            with this yet, but if later c := Num, *then* we can solve it

Example 2:  a ~ b, where a :: *, b :: k, where k is a kind variable
            We don't want to use this to substitute 'b' for 'a', in case
            'k' is subequently unifed with (say) *->*, because then
            we'd have ill-kinded types floating about.  Rather we want
            to defer using the equality altogether until 'k' get resolved.

### Note: Ct/evidence invariant

If  ct :: Ct, then extra fields of 'ct' cache precisely the ctev_pred field
of (cc_ev ct), and is fully rewritten wrt the substitution.   Eg for CDictCan,
   ctev_pred (cc_ev ct) = (cc_class ct) (cc_tyargs ct)
This holds by construction; look at the unique place where CDictCan is
built (in TcCanonical).

### Note: Evidence field of CtEvidence

### Note: Ct kind invariant

CTyEqCan and CFunEqCan both require that the kind of the lhs matches the kind
of the rhs. This is necessary because both constraints are used for substitutions
during solving. If the kinds differed, then the substitution would take a well-kinded
type to an ill-kinded one.



# Simple functions over evidence variables


### Note: Resetting cc_pend_sc

When we discard Derived constraints, in dropDerivedSimples, we must
set the cc_pend_sc flag to True, so that if we re-process this
CDictCan we will re-generate its derived superclasses. Otherwise
we might miss some fundeps.  Trac #13662 showed this up.

### Note: The superclass story

### Note: Dropping derived constraints

In general we discard derived constraints at the end of constraint solving;
see dropDerivedWC.  For example

 * Superclasses: if we have an unsolved [W] (Ord a), we don't want to
   complain about an unsolved [D] (Eq a) as well.

 * If we have [W] a ~ Int, [W] a ~ Bool, improvement will generate
   [D] Int ~ Bool, and we don't want to report that because it's
   incomprehensible. That is why we don't rewrite wanteds with wanteds!

But (tiresomely) we do keep *some* Derived constraints:

 * Type holes are derived constraints, because they have no evidence
   and we want to keep them, so we get the error report

### Note: Equalities with incompatible kinds

 * We keep most derived equalities arising from functional dependencies
      - Given/Given interactions (subset of FunDepOrigin1):
        The definitely-insoluble ones reflect unreachable code.

        Others not-definitely-insoluble ones like [D] a ~ Int do not
        reflect unreachable code; indeed if fundeps generated proofs, it'd
        be a useful equality.  See Trac #14763.   So we discard them.

      - Given/Wanted interacGiven or Wanted interacting with an
        instance declaration (FunDepOrigin2)

      - Given/Wanted interactions (FunDepOrigin1); see Trac #9612

      - But for Wanted/Wanted interactions we do /not/ want to report an
        error (Trac #13506).  Consider [W] C Int Int, [W] C Int Bool, with
        a fundep on class C.  We don't want to report an insoluble Int~Bool;
        c.f. "wanteds do not rewrite wanteds".

To distinguish these cases we use the CtOrigin.

NB: we keep *all* derived insolubles under some circumstances:

  * They are looked at by simplifyInfer, to decide whether to
    generalise.  Example: [W] a ~ Int, [W] a ~ Bool
    We get [D] Int ~ Bool, and indeed the constraints are insoluble,
    and we want simplifyInfer to see that, even though we don't
    ultimately want to generate an (inexplicable) error message from it

# CtEvidence
         The "flavor" of a canonical constraint


### Note: Custom type errors in constraints


When GHC reports a type-error about an unsolved-constraint, we check
to see if the constraint contains any custom-type errors, and if so
we report them.  Here are some examples of constraints containing type
errors:

TypeError msg           -- The actual constraint is a type error

TypError msg ~ Int      -- Some type was supposed to be Int, but ended up
                        -- being a type error instead

Eq (TypeError msg)      -- A class constraint is stuck due to a type error

F (TypeError msg) ~ a   -- A type function failed to evaluate due to a type err

It is also possible to have constraints where the type error is nested deeper,
for example see #11990, and also:

Eq (F (TypeError msg))  -- Here the type error is nested under a type-function
                        -- call, which failed to evaluate because of it,
                        -- and so the `Eq` constraint was unsolved.
                        -- This may happen when one function calls another
                        -- and the called function produced a custom type error.


### Note: When superclasses help

### Note: The superclass story

We expand superclasses and iterate only if there is at unsolved wanted
for which expansion of superclasses (e.g. from given constraints)
might actually help. The function superClassesMightHelp tells if
doing this superclass expansion might help solve this constraint.
Note that

  * Superclasses help only for Wanted constraints.  Derived constraints
    are not really "unsolved" and we certainly don't want them to
    trigger superclass expansion. This was a good part of the loop
    in  Trac #11523

  * Even for Wanted constraints, we say "no" for implicit parameters.
    we have [W] ?x::ty, expanding superclasses won't help:
      - Superclasses can't be implicit parameters
      - If we have a [G] ?x:ty2, then we'll have another unsolved
        [D] ty ~ ty2 (from the functional dependency)
        which will trigger superclass expansion.

    It's a bit of a special case, but it's easy to do.  The runtime cost
    is low because the unsolved set is usually empty anyway (errors
    aside), and the first non-imlicit-parameter will terminate the search.

    The special case is worth it (Trac #11480, comment:2) because it
    applies to CallStack constraints, which aren't type errors. If we have
       f :: (C a) => blah
       f x = ...undefined...
    we'll get a CallStack constraint.  If that's the only unsolved
    constraint it'll eventually be solved by defaulting.  So we don't
    want to emit warnings about hitting the simplifier's iteration
    limit.  A CallStack constraint really isn't an unsolved
    constraint; it can always be solved by defaulting.


# 

### Note: Given insolubles

Consider (Trac #14325, comment:)
    class (a~b) => C a b

    foo :: C a c => a -> c
    foo x = x

    hm3 :: C (f b) b => b -> f b
    hm3 x = foo x

In the RHS of hm3, from the [G] C (f b) b we get the insoluble
[G] f b ~# b.  Then we also get an unsolved [W] C b (f b).
Residual implication looks like
    forall b. C (f b) b => [G] f b ~# b
                           [W] C f (f b)

### Note: Given errors

Bottom line: insolubleWC (called in TcSimplify.setImplicationStatus)
             should ignore givens even if they are insoluble.

### Note: Insoluble holes

Hole constraints that ARE NOT treated as truly insoluble:
  a) type holes, arising from PartialTypeSignatures,
  b) "true" expression holes arising from TypedHoles

An "expression hole" or "type hole" constraint isn't really an error
at all; it's a report saying "_ :: Int" here.  But an out-of-scope
variable masquerading as expression holes IS treated as truly
insoluble, so that it trumps other errors during error reporting.
Yuk!

# Implication constraints


### Note: Needed evidence variables

Th ic_need_evs field holds the free vars of ic_binds, and all the
ic_binds in nested implications.

  * Main purpose: if one of the ic_givens is not mentioned in here, it
    is redundant.

  * solveImplication may drop an implication altogether if it has no
    remaining 'wanteds'. But we still track the free vars of its
    evidence binds, even though it has now disappeared.

### Note: Shadowing in a constraint

We assume NO SHADOWING in a constraint.  Specifically
 * The unification variables are all implicitly quantified at top
   level, and are all unique
 * The skolem variables bound in ic_skols are all freah when the
   implication is created.
So we can safely substitute. For example, if we have
   forall a.  a~Int => ...(forall b. ...a...)...
we can push the (a~Int) constraint inwards in the "givens" without
worrying that 'b' might clash.

### Note: Skolems in an implication

The skolems in an implication are not there to perform a skolem escape
check.  That happens because all the environment variables are in the
untouchables, and therefore cannot be unified with anything at all,
let alone the skolems.

Instead, ic_skols is used only when considering floating a constraint
outside the implication in TcSimplify.floatEqualities or
TcSimplify.approximateImplications

### Note: Insoluble constraints

Some of the errors that we get during canonicalization are best
reported when all constraints have been simplified as much as
possible. For instance, assume that during simplification the
following constraints arise:

 [Wanted]   F alpha ~  uf1
 [Wanted]   beta ~ uf1 beta

When canonicalizing the wanted (beta ~ uf1 beta), if we eagerly fail
we will simply see a message:
    'Can't construct the infinite type  beta ~ uf1 beta'
and the user has no idea what the uf1 variable is.

Instead our plan is that we will NOT fail immediately, but:
    (1) Record the "frozen" error in the ic_insols field
    (2) Isolate the offending constraint from the rest of the inerts
    (3) Keep on simplifying/canonicalizing

At the end, we will hopefully have substituted uf1 := F alpha, and we
will be able to report a more informative error:
    'Can't construct the infinite type beta ~ F alpha beta'

Insoluble constraints *do* include Derived constraints. For example,
a functional dependency might give rise to [D] Int ~ Bool, and we must
report that.  If insolubles did not contain Deriveds, reportErrors would
never see it.

# Pretty printing


# CtEvidence


### Note: Evidence field of CtEvidence

During constraint solving we never look at the type of ctev_evar/ctev_dest;
instead we look at the ctev_pred field.  The evtm/evar field
may be un-zonked.

### Note: Bind new Givens immediately

For Givens we make new EvVars and bind them immediately. Two main reasons:
  * Gain sharing.  E.g. suppose we start with g :: C a b, where
       class D a => C a b
       class (E a, F a) => D a
    If we generate all g's superclasses as separate EvTerms we might
    get    selD1 (selC1 g) :: E a
           selD2 (selC1 g) :: F a
           selC1 g :: D a
    which we could do more economically as:
           g1 :: D a = selC1 g
           g2 :: E a = selD1 g1
           g3 :: F a = selD2 g1

  * For *coercion* evidence we *must* bind each given:
      class (a~b) => C a b where ....
      f :: C a b => ....
    Then in f's Givens we have g:(C a b) and the superclass sc(g,0):a~b.
    But that superclass selector can't (yet) appear in a coercion
    (see evTermCoercion), so the easy thing is to bind it to an Id.

So a Given has EvVar inside it rather than (as previously) an EvTerm.

### Note: Given in ctEvCoercion

### Note: Evidence field of CtEvidence



# 

### Note: Constraint flavours

Constraints come in four flavours:

* [G] Given: we have evidence

* [W] Wanted WOnly: we want evidence

* [D] Derived: any solution must satisfy this constraint, but
      we don't need evidence for it.  Examples include:
        - superclasses of [W] class constraints
        - equalities arising from functional dependencies
          or injectivity

* [WD] Wanted WDeriv: a single constraint that represents
                      both [W] and [D]
  We keep them paired as one both for efficiency, and because
  when we have a finite map  F tys -> CFunEqCan, it's inconvenient
  to have two CFunEqCans in the range

The ctev_nosh field of a Wanted distinguishes between [W] and [WD]

Wanted constraints are born as [WD], but are split into [W] and its
"shadow" [D] in TcSMonad.maybeEmitShadow.

### Note: The improvement story and derived shadows

### Note: eqCanRewrite

(eqCanRewrite ct1 ct2) holds if the constraint ct1 (a CTyEqCan of form
tv ~ ty) can be used to rewrite ct2.  It must satisfy the properties of
a can-rewrite relation, see Definition [Can-rewrite relation] in
TcSMonad.

With the solver handling Coercible constraints like equality constraints,
the rewrite conditions must take role into account, never allowing
a representational equality to rewrite a nominal one.

### Note: Wanteds do not rewrite Wanteds

We don't allow Wanteds to rewrite Wanteds, because that can give rise
to very confusing type error messages.  A good example is Trac #8450.
Here's another
   f :: a -> Bool
   f x = ( [x,'c'], [x,True] ) `seq` True
Here we get
  [W] a ~ Char
  [W] a ~ Bool
but we do not want to complain about Bool ~ Char!

### Note: Deriveds do rewrite Deriveds

### Note: The improvement story

However, for now at least I'm only letting (Derived,NomEq) rewrite
(Derived,NomEq) and not doing anything for ReprEq.  If we have
    eqCanRewriteFR (Derived, NomEq) (Derived, _)  = True
then we lose property R2 of Definition [Can-rewrite relation]
in TcSMonad
  R2.  If f1 >= f, and f2 >= f,
       then either f1 >= f2 or f2 >= f1
Consider f1 = (Given, ReprEq)
         f2 = (Derived, NomEq)
          f = (Derived, ReprEq)

I thought maybe we could never get Derived ReprEq constraints, but
we can; straight from the Wanteds during improvement. And from a Derived
ReprEq we could conceivably get a Derived NomEq improvement (by decomposing
a type constructor with Nomninal role), and hence unify.


### Note: funEqCanDischarge

Suppose we have two CFunEqCans with the same LHS:
    (x1:F ts ~ f1) `funEqCanDischarge` (x2:F ts ~ f2)
Can we drop x2 in favour of x1, either unifying
f2 (if it's a flatten meta-var) or adding a new Given
(f1 ~ f2), if x2 is a Given?

Answer: yes if funEqCanDischarge is true.


### Note: eqCanDischarge

Suppose we have two identical CTyEqCan equality constraints
(i.e. both LHS and RHS are the same)
      (x1:a~t) `eqCanDischarge` (xs:a~t)
Can we just drop x2 in favour of x1?

Answer: yes if eqCanDischarge is true.

Note that we do /not/ allow Wanted to discharge Derived.
We must keep both.  Why?  Because the Derived may rewrite
other Deriveds in the model whereas the Wanted cannot.

However a Wanted can certainly discharge an identical Wanted.  So
eqCanDischarge does /not/ define a can-rewrite relation in the
sense of Definition [Can-rewrite relation] in TcSMonad.

We /do/ say that a [W] can discharge a [WD].  In evidence terms it
certainly can, and the /caller/ arranges that the otherwise-lost [D]
is spat out as a new Derived.  

# SubGoalDepth


### Note: SubGoalDepth

The 'SubGoalDepth' takes care of stopping the constraint solver from looping.

The counter starts at zero and increases. It includes dictionary constraints,
equality simplification, and type family reduction. (Why combine these? Because
it's actually quite easy to mistake one for another, in sufficiently involved
scenarios, like ConstraintKinds.)

The flag -fcontext-stack=n (not very well named!) fixes the maximium
level.

* The counter includes the depth of type class instance declarations.  Example:
     [W] d{7} : Eq [Int]
  That is d's dictionary-constraint depth is 7.  If we use the instance
     $dfEqList :: Eq a => Eq [a]
  to simplify it, we get
     d{7} = $dfEqList d'{8}
  where d'{8} : Eq Int, and d' has depth 8.

  For civilised (decidable) instance declarations, each increase of
  depth removes a type constructor from the type, so the depth never
  gets big; i.e. is bounded by the structural depth of the type.

* The counter also increments when resolving
equalities involving type functions. Example:
  Assume we have a wanted at depth 7:
    [W] d{7} : F () ~ a
  If there is a type function equation "F () = Int", this would be rewritten to
    [W] d{8} : Int ~ a
  and remembered as having depth 8.

  Again, without UndecidableInstances, this counter is bounded, but without it
  can resolve things ad infinitum. Hence there is a maximum level.

* Lastly, every time an equality is rewritten, the counter increases. Again,
  rewriting an equality constraint normally makes progress, but it's possible
  the "progress" is just the reduction of an infinitely-reducing type family.
  Hence we need to track the rewrites.

When compiling a program requires a greater depth, then GHC recommends turning
off this check entirely by setting -freduction-depth=0. This is because the
exact number that works is highly variable, and is likely to change even between
minor releases. Because this check is solely to prevent infinite compilation
times, it seems safe to disable it when a user has ascertained that their program
doesn't loop at the type level.



# CtLoc


The 'CtLoc' gives information about where a constraint came from.
This is important for decent error message reporting because
dictionaries don't appear in the original source code.
type will evolve...



# SkolemInfo


### Note: Skolem info for pattern synonyms

For pattern synonym SkolemInfo we have
   SigSkol (PatSynCtxt p) ty _
but the type 'ty' is not very helpful.  The full pattern-synonym type
has the provided and required pieces, which it is inconvenient to
record and display here. So we simply don't display the type at all,
contenting outselves with just the name of the pattern synonym, which
is fine.  We could do more, but it doesn't seem worth it.

### Note: SigSkol SkolemInfo

Suppose we (deeply) skolemise a type
   f :: forall a. a -> forall b. b -> a
Then we'll instantiate [a :-> a', b :-> b'], and with the instantiated
      a' -> b' -> a.
But when, in an error message, we report that "b is a rigid type
variable bound by the type signature for f", we want to show the foralls
in the right place.  So we proceed as follows:

* In SigSkol we record
    - the original signature forall a. a -> forall b. b -> a
    - the instantiation mapping [a :-> a', b :-> b']

* Then when tidying in TcMType.tidySkolemInfo, we first tidy a' to
  whatever it tidies to, say a''; and then we walk over the type
  replacing the binder a by the tidied version a'', to give
       forall a''. a'' -> forall b''. b'' -> a''
  We need to do this under function arrows, to match what deeplySkolemise
  does.

* Typically a'' will have a nice pretty name like "a", but the point is
  that the foral-bound variables of the signature we report line up with
  the instantiated skolems lying  around in other types.

# CtOrigin



Constraint Solver Plugins
-------------------------


# Role annotations
