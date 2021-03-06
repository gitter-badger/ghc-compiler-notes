[[src]](https://github.com/ghc/ghc/tree/master/compiler/typecheck/TcMatches.hs)

(c) The University of Glasgow 2006
(c) The GRASP/AQUA Project, Glasgow University, 1992-1998


TcMatches: Typecheck some @Matches@


# \subsection{tcMatchesFun, tcMatchesCase}


@tcMatchesFun@ typechecks a @[Match]@ list which occurs in a
@FunMonoBind@.  The second argument is the name of the function, which
is used in error messages.  It checks that all the equations have the
same number of arguments before using @tcMatches@ to do the work.

### Note: Polymorphic expected type for tcMatchesFun

### Note: sig_tau may be polymorphic


@tcMatchesCase@ doesn't do the argument-count check because the
parser guarantees that each equation has exactly one argument.


# \subsection{tcMatch}


### Note: Case branches must never infer a non-tau type

Consider

  case ... of
    ... -> \(x :: forall a. a -> a) -> x
    ... -> \y -> y

Should that type-check? The problem is that, if we check the second branch
first, then we'll get a type (b -> b) for the branches, which won't unify
with the polytype in the first branch. If we check the first branch first,
then everything is OK. This order-dependency is terrible. So we want only
proper tau-types in branches (unless a sigma-type is pushed down).
This is what expTypeToType ensures: it replaces an Infer with a fresh
tau-type.

An even trickier case looks like

  f x True  = x undefined
  f x False = x ()

Here, we see that the arguments must also be non-Infer. Thus, we must
use expTypeToType on the output of matchExpectedFunTys, not the input.

But we make a special case for a one-branch case. This is so that

  f = \(x :: forall a. a -> a) -> x

still gets assigned a polytype.


# \subsection{@tcDoStmts@ typechecks a {\em list} of do statements}


# \subsection{tcStmts}


### Note: Treat rebindable syntax first

When typechecking
        do { bar; ... } :: IO ()
we want to typecheck 'bar' in the knowledge that it should be an IO thing,
pushing info from the context into the RHS.  To do this, we check the
rebindable syntax first, and push that information into (tcMonoExprNC rhs).
Otherwise the error shows up when cheking the rebindable syntax, and
the expected/inferred stuff is back to front (see Trac #3613).

### Note: typechecking ApplicativeStmt

join ((\pat1 ... patn -> body) <$> e1 <*> ... <*> en)

fresh type variables:
   pat_ty_1..pat_ty_n
   exp_ty_1..exp_ty_n
   t_1..t_(n-1)

body  :: body_ty
(\pat1 ... patn -> body) :: pat_ty_1 -> ... -> pat_ty_n -> body_ty
pat_i :: pat_ty_i
e_i   :: exp_ty_i
<$>   :: (pat_ty_1 -> ... -> pat_ty_n -> body_ty) -> exp_ty_1 -> t_1
<*>_i :: t_(i-1) -> exp_ty_i -> t_i
join :: tn -> res_ty


### Note: ApplicativeDo and constraints

An applicative-do is supposed to take place in parallel, so
constraints bound in one arm can't possibly be available in another
(Trac #13242).  Our current rule is this (more details and discussion
on the ticket). Consider

   ...stmts...
   ApplicativeStmts [arg1, arg2, ... argN]
   ...more stmts...

where argi :: ApplicativeArg. Each 'argi' itself contains one or more Stmts.
Now, we say that:

* Constraints required by the argi can be solved from
  constraint bound by ...stmts...

* Constraints and existentials bound by the argi are not available
  to solve constraints required either by argj (where i /= j),
  or by ...more stmts....

* Within the stmts of each 'argi' individually, however, constraints bound
  by earlier stmts can be used to solve later ones.

To achieve this, we just typecheck each 'argi' separately, bring all
the variables they bind into scope, and typecheck the thing_inside.

# \subsection{Errors and contexts}


@sameNoOfArgs@ takes a @[RenamedMatch]@ and decides whether the same
number of args are used in each equation.
