[[src]](https://github.com/ghc/ghc/tree/master/compiler/typecheck/TcType.hs)

(c) The University of Glasgow 2006
(c) The GRASP/AQUA Project, Glasgow University, 1992-1998

# Types used in the typechecker

This module provides the Type interface for front-end parts of the
compiler.  These parts

        * treat "source types" as opaque:
                newtypes, and predicates are meaningful.
        * look through usage types

The "tc" prefix is for "TypeChecker", because the type checker
is the principal client.


# Types


The type checker divides the generic Type world into the
following more structured beasts:

sigma ::= forall tyvars. phi
        -- A sigma type is a qualified type
        --
        -- Note that even if 'tyvars' is empty, theta
        -- may not be: e.g.   (?x::Int) => Int

        -- Note that 'sigma' is in prenex form:
        -- all the foralls are at the front.
        -- A 'phi' type has no foralls to the right of
        -- an arrow

phi :: theta => rho

rho ::= sigma -> rho
     |  tau

-- A 'tau' type has no quantification anywhere
-- Note that the args of a type constructor must be taus
tau ::= tyvar
     |  tycon tau_1 .. tau_n
     |  tau_1 tau_2
     |  tau_1 -> tau_2

-- In all cases, a (saturated) type synonym application is legal,
-- provided it expands to the required form.

### Note: TcTyVars in the typechecker

The typechecker uses a lot of type variables with special properties,
notably being a unification variable with a mutable reference.  These
use the 'TcTyVar' variant of Var.Var.

However, the type checker and constraint solver can encounter type
variables that use the 'TyVar' variant of Var.Var, for a couple of
reasons:

  - When unifying or flattening under (forall a. ty)

  - When typechecking a class decl, say
       class C (a :: k) where
          foo :: T a -> Int
    We have first kind-check the header; fix k and (a:k) to be
    TyVars, bring 'k' and 'a' into scope, and kind check the
    signature for 'foo'.  In doing so we call solveEqualities to
    solve any kind equalities in foo's signature.  So the solver
    may see free occurrences of 'k'.

    See calls to tcExtendTyVarEnv for other places that ordinary
    TyVars are bought into scope, and hence may show up in the types
    and inds generated by TcHsType.

It's convenient to simply treat these TyVars as skolem constants,
which of course they are.  So

* Var.tcTyVarDetails succeeds on a TyVar, returning
  vanillaSkolemTv, as well as on a TcTyVar.

* tcIsTcTyVar returns True for both TyVar and TcTyVar variants
  of Var.Var.  The "tc" prefix means "a type variable that can be
  encountered by the typechecker".

This is a bit of a change from an earlier era when we remoselessly
insisted on real TcTyVars in the type checker.  But that seems
unnecessary (for skolems, TyVars are fine) and it's now very hard
to guarantee, with the advent of kind equalities.

### Note: Coercion variables in free variable lists

There are several places in the GHC codebase where functions like
tyCoVarsOfType, tyCoVarsOfCt, et al. are used to compute the free type
variables of a type. The "Co" part of these functions' names shouldn't be
dismissed, as it is entirely possible that they will include coercion variables
in addition to type variables! As a result, there are some places in TcType
where we must take care to check that a variable is a _type_ variable (using
isTyVar) before calling tcTyVarDetails--a partial function that is not defined
for coercion variables--on the variable. Failing to do so led to
GHC Trac #12785.


# ExpType: an "expected type" in the type checker


# SyntaxOpType


### Note: TcRhoType

A TcRhoType has no foralls or contexts at the top, or to the right of an arrow
  YES    (forall a. a->a) -> Int
  NO     forall a. a ->  Int
  NO     Eq a => a -> a
  NO     Int -> forall a. a -> Int

# TyVarDetails, MetaDetails, MetaInfo


TyVarDetails gives extra info about type variables, used during type
checking.  It's attached to mutable type variables only.
It's knot-tied back to Var.hs.  There is no reason in principle
why Var.hs shouldn't actually have the definition, but it "belongs" here.

### Note: Signature skolems

A SigTv is a specialised variant of TauTv, with the following invarints:

    * A SigTv can be unified only with a TyVar,
      not with any other type

    * Its MetaDetails, if filled in, will always be another SigTv
      or a SkolemTv

SigTvs are only distinguished to improve error messages.
Consider this

  f :: forall a. [a] -> Int
  f (x::b : xs) = 3

Here 'b' is a lexically scoped type variable, but it turns out to be
the same as the skolem 'a'.  So we make them both SigTvs, which can unify
with each other.

Similarly consider
  data T (a:k1) = MkT (S a)
  data S (b:k2) = MkS (T b)
When doing kind inference on {S,T} we don't want *skolems* for k1,k2,
because they end up unifying; we want those SigTvs again.

SigTvs are used *only* for pattern type signatures.

### Note: TyVars and TcTyVars during type checking

The Var type has constructors TyVar and TcTyVar.  They are used
as follows:

* TcTyVar: used /only/ during type checking.  Should never appear
  afterwards.  May contain a mutable field, in the MetaTv case.

* TyVar: is never seen by the constraint solver, except locally
  inside a type like (forall a. [a] ->[a]), where 'a' is a TyVar.
  We instantiate these with TcTyVars before exposing the type
  to the constraint solver.

I have swithered about the latter invariant, excluding TyVars from the
constraint solver.  It's not strictly essential, and indeed
(historically but still there) Var.tcTyVarDetails returns
vanillaSkolemTv for a TyVar.

But ultimately I want to seeparate Type from TcType, and in that case
we would need to enforce the separation.


# UserTypeCtxt



-- Notes re TySynCtxt
-- We allow type synonyms that aren't types; e.g.  type List = []
--
-- If the RHS mentions tyvars that aren't in scope, we'll
-- quantify over them:
--      e.g.    type T = a->a
-- will become  type T = forall a. a->a
--
-- With gla-exts that's right, but for H98 we should complain.


# Untoucable type variables


### Note: TcLevel and untouchable type variables

* Each unification variable (MetaTv)
  and each Implication
  has a level number (of type TcLevel)

* INVARIANTS.  In a tree of Implications,

    (ImplicInv) The level number of an Implication is
                STRICTLY GREATER THAN that of its parent

    (MetaTvInv) The level number of a unification variable is
                LESS THAN OR EQUAL TO that of its parent
                implication

* A unification variable is *touchable* if its level number
  is EQUAL TO that of its immediate parent implication.

* INVARIANT
    (GivenInv)  The free variables of the ic_given of an
                implication are all untouchable; ie their level
                numbers are LESS THAN the ic_tclvl of the implication

### Note: Skolem escape prevention

We only unify touchable unification variables.  Because of
(MetaTvInv), there can be no occurrences of the variable further out,
so the unification can't cause the skolems to escape. Example:
     data T = forall a. MkT a (a->Int)
     f x (MkT v f) = length [v,x]
We decide (x::alpha), and generate an implication like
      [1]forall a. (a ~ alpha[0])
But we must not unify alpha:=a, because the skolem would escape.

For the cases where we DO want to unify, we rely on floating the
equality.   Example (with same T)
     g x (MkT v f) = x && True
We decide (x::alpha), and generate an implication like
      [1]forall a. (Bool ~ alpha[0])
We do NOT unify directly, bur rather float out (if the constraint
does not mention 'a') to get
      (Bool ~ alpha[0]) /\ [1]forall a.()
and NOW we can unify alpha.

The same idea of only unifying touchables solves another problem.
Suppose we had
   (F Int ~ uf[0])  /\  [1](forall a. C a => F Int ~ beta[1])
In this example, beta is touchable inside the implication. The
first solveSimpleWanteds step leaves 'uf' un-unified. Then we move inside
the implication where a new constraint
       uf  ~  beta
emerges. If we (wrongly) spontaneously solved it to get uf := beta,
the whole implication disappears but when we pop out again we are left with
(F Int ~ uf) which will be unified by our final zonking stage and
uf will get unified *once more* to (F Int).

### Note: TcLevel assignment

We arrange the TcLevels like this

   0   Level for all flatten meta-vars
   1   Top level
   2   First-level implication constraints
   3   Second-level implication constraints
   ...etc...

The flatten meta-vars are all at level 0, just to make them untouchable.


# Finding type family instances


# The "exact" free variables of a type


### Note: Silly type synonym

Consider
  type T a = Int
What are the free tyvars of (T x)?  Empty, of course!
Here's the example that Ralf Laemmel showed me:
  foo :: (forall a. C u a -> C u a) -> u
  mappend :: Monoid u => u -> u -> u

  bar :: Monoid u => u
  bar = foo (\t -> t `mappend` t)
We have to generalise at the arg to f, and we don't
want to capture the constraint (Monad (C u a)) because
it appears to mention a.  Pretty silly, but it was useful to him.

exactTyCoVarsOfType is used by the type checker to figure out exactly
which type variables are mentioned in a type.  It's also used in the
smart-app checking code --- see TcExpr.tcIdApp

On the other hand, consider a *top-level* definition
  f = (\x -> x) :: T a -> T a
If we don't abstract over 'a' it'll get fixed to GHC.Prim.Any, and then
if we have an application like (f "x") we get a confusing error message
involving Any.  So the conclusion is this: when generalising
  - at top level use tyCoVarsOfType
  - in nested bindings use exactTyCoVarsOfType
See Trac #1813 for example.


### Note: anyRewritableTyVar must be role-aware

anyRewritableTyVar is used during kick-out from the inert set,
to decide if, given a new equality (a ~ ty), we should kick out
a constraint C.  Rather than gather free variables and see if 'a'
is among them, we instead pass in a predicate; this is just efficiency.

Moreover, consider
  work item:   [G] a ~R f b
  inert item:  [G] b ~R f a
We use anyRewritableTyVar to decide whether to kick out the inert item,
on the grounds that the work item might rewrite it. Well, 'a' is certainly
free in [G] b ~R f a.  But because the role of a type variable ('f' in
this case) is nominal, the work item can't actually rewrite the inert item.
Moreover, if we were to kick out the inert item the exact same situation
would re-occur and we end up with an infinite loop in which each kicks
out the other (Trac #14363).


# Bound variables in a type


# Type and kind variables in a type


### Note: Dependent type variables

In Haskell type inference we quantify over type variables; but we only
quantify over /kind/ variables when -XPolyKinds is on.  Without -XPolyKinds
we default the kind variables to *.

So, to support this defaulting, and only for that reason, when
collecting the free vars of a type, prior to quantifying, we must keep
the type and kind variables separate.

But what does that mean in a system where kind variables /are/ type
variables? It's a fairly arbitrary distinction based on how the
variables appear:

  - "Kind variables" appear in the kind of some other free variable
     PLUS any free coercion variables

     These are the ones we default to * if -XPolyKinds is off

  - "Type variables" are all free vars that are not kind variables

E.g.  In the type    T k (a::k)
      'k' is a kind variable, because it occurs in the kind of 'a',
          even though it also appears at "top level" of the type
      'a' is a type variable, because it doesn't

We gather these variables using a CandidatesQTvs record:
  DV { dv_kvs: Variables free in the kind of a free type variable
               or of a forall-bound type variable
     , dv_tvs: Variables sytactically free in the type }

So:  dv_kvs            are the kind variables of the type
     (dv_tvs - dv_kvs) are the type variable of the type

Note that

* A variable can occur in both.
      T k (x::k)    The first occurrence of k makes it
                    show up in dv_tvs, the second in dv_kvs

* We include any coercion variables in the "dependent",
  "kind-variable" set because we never quantify over them.

* Both sets are un-ordered, of course.

* The "kind variables" might depend on each other; e.g
     (k1 :: k2), (k2 :: *)
  The "type variables" do not depend on each other; if
  one did, it'd be classified as a kind variable!

### Note: CandidatesQTvs determinism and order

* Determinism: when we quantify over type variables we decide the
  order in which they appear in the final type. Because the order of
  type variables in the type can end up in the interface file and
  affects some optimizations like worker-wrapper, we want this order to
  be deterministic.

### Note: Deterministic UniqFM

* Order: as well as being deterministic, we use an
  accumulating-parameter style for candidateQTyVarsOfType so that we
  add variables one at a time, left to right.  That means we tend to
  produce the variables in left-to-right order.  This is just to make
  it bit more predicatable for the programmer.


# Predicates


# \subsection{Tau, sigma and rho}


# \subsection{Expanding and splitting}


These tcSplit functions are like their non-Tc analogues, but
        *) they do not look through newtypes

However, they are non-monadic and do not follow through mutable type
variables.  It's up to you to make sure this doesn't matter.


# Type equalities


# Predicate types


Deconstructors and tests on predicate types

### Note: Kind polymorphic type classes

    class C f where...   -- C :: forall k. k -> Constraint
    g :: forall (f::*). C f => f -> f

Here the (C f) in the signature is really (C * f), and we
don't want to complain that the * isn't a type variable!


### Note: Expanding superclasses

When we expand superclasses, we use the following algorithm:

expand( so_far, pred ) returns the transitive superclasses of pred,
                               not including pred itself
 1. If pred is not a class constraint, return empty set
       Otherwise pred = C ts
 2. If C is in so_far, return empty set (breaks loops)
 3. Find the immediate superclasses constraints of (C ts)
 4. For each such sc_pred, return (sc_pred : expand( so_far+C, D ss )

Notice that

 * With normal Haskell-98 classes, the loop-detector will never bite,
   so we'll get all the superclasses.

 * Since there is only a finite number of distinct classes, expansion
   must terminate.

 * The loop breaking is a bit conservative. Notably, a tuple class
   could contain many times without threatening termination:
      (Eq a, (Ord a, Ix a))
   And this is try of any class that we can statically guarantee
   as non-recursive (in some sense).  For now, we just make a special
   case for tuples.  Something better would be cool.

See also TcTyDecls.checkClassCycles.

### Note: Inheriting implicit parameters

Consider this:

        f x = (x::Int) + ?y

where f is *not* a top-level binding.
From the RHS of f we'll get the constraint (?y::Int).
There are two types we might infer for f:

        f :: Int -> Int

(so we get ?y from the context of f's definition), or

        f :: (?y::Int) => Int -> Int

At first you might think the first was better, because then
?y behaves like a free variable of the definition, rather than
having to be passed at each call site.  But of course, the WHOLE
IDEA is that ?y should be passed at each call site (that's what
dynamic binding means) so we'd better infer the second.

BOTTOM LINE: when *inferring types* you must quantify over implicit
parameters, *even if* they don't mention the bound type variables.
Reason: because implicit parameters, uniquely, have local instance
declarations. See pickQuantifiablePreds.

### Note: Quantifying over equality constraints

Should we quantify over an equality constraint (s ~ t)?  In general, we don't.
Doing so may simply postpone a type error from the function definition site to
its call site.  (At worst, imagine (Int ~ Bool)).

However, consider this
         forall a. (F [a] ~ Int) => blah
Should we quantify over the (F [a] ~ Int).  Perhaps yes, because at the call
site we will know 'a', and perhaps we have instance  F [Bool] = Int.
So we *do* quantify over a type-family equality where the arguments mention
the quantified variables.

# \subsection{Predicates}


### Note: AppTy and ReprEq

Consider   a ~R# b a
           a ~R# a b

The former is /not/ a definite error; we might instantiate 'b' with Id
   newtype Id a = MkId a
but the latter /is/ a definite error.

On the other hand, with nominal equality, both are definite errors


# \subsection{Transformation of Types to TcTypes}


# \subsection{Misc}


### Note: Visible type application

GHC implements a generalisation of the algorithm described in the
"Visible Type Application" paper (available from
http://www.cis.upenn.edu/~sweirich/publications.html). A key part
of that algorithm is to distinguish user-specified variables from inferred
variables. For example, the following should typecheck:

  f :: forall a b. a -> b -> b
  f = const id

  g = const id

  x = f @Int @Bool 5 False
  y = g 5 @Bool False

The idea is that we wish to allow visible type application when we are
instantiating a specified, fixed variable. In practice, specified, fixed
variables are either written in a type signature (or
annotation), OR are imported from another module. (We could do better here,
for example by doing SCC analysis on parts of a module and considering any
type from outside one's SCC to be fully specified, but this is very confusing to
users. The simple rule above is much more straightforward and predictable.)

So, both of f's quantified variables are specified and may be instantiated.
But g has no type signature, so only id's variable is specified (because id
is imported). We write the type of g as forall {a}. a -> forall b. b -> b.
Note that the a is in braces, meaning it cannot be instantiated with
visible type application.

Tracking specified vs. inferred variables is done conveniently by a field
in TyBinder.




Find the free tycons and classes of a type.  This is used in the front
end of the compiler.


# \subsection[TysWiredIn-ext-type]{External types}


The compiler's foreign function interface supports the passing of a
restricted set of types as arguments and results (the restricting factor
being the )


### Note: Foreign import dynamic

A dynamic stub must be of the form 'FunPtr ft -> ft' where ft is any foreign
type.  Similarly, a wrapper stub must be of the form 'ft -> IO (FunPtr ft)'.

We use isFFIDynTy to check whether a signature is well-formed. For example,
given a (illegal) declaration like:

foreign import ccall "dynamic"
  foo :: FunPtr (CDouble -> IO ()) -> CInt -> IO ()

isFFIDynTy will compare the 'FunPtr' type 'CDouble -> IO ()' with the curried
result type 'CInt -> IO ()', and return False, as they are not equal.


----------------------------------------------
These chaps do the work; they are not exported
----------------------------------------------


### Note: Marshalling void

We don't treat State# (whose PrimRep is VoidRep) as marshalable.
In turn that means you can't write
        foreign import foo :: Int -> State# RealWorld

Reason: the back end falls over with panic "primRepHint:VoidRep";
        and there is no compelling reason to permit it


# The "Paterson size" of a type


### Note: Paterson conditions on PredTypes

### Note: Paterson conditions

However, we can be a bit more refined by looking at which kind of constraint
this actually is. There are two main tricks:

 1. It seems like it should be OK not to count the tuple type constructor
    for a PredType like (Show a, Eq a) :: Constraint, since we don't
    count the "implicit" tuple in the ThetaType itself.

    In fact, the Paterson test just checks *each component* of the top level
    ThetaType against the size bound, one at a time. By analogy, it should be
    OK to return the size of the *largest* tuple component as the size of the
    whole tuple.

 2. Once we get into an implicit parameter or equality we
    can't get back to a class constraint, so it's safe
    to say "size 0".  See Trac #4200.

NB: we don't want to detect PredTypes in sizeType (and then call
sizePred on them), or we might get an infinite loop if that PredType
is irreducible. See Trac #5581.
