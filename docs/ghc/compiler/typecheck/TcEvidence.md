[[src]](https://github.com/ghc/ghc/tree/master/compiler/typecheck/TcEvidence.hs)
### Note: TcCoercions

| TcCoercions are a hack used by the typechecker. Normally,
Coercions have free variables of type (a ~# b): we call these
CoVars. However, the type checker passes around equality evidence
(boxed up) at type (a ~ b).

An TcCoercion is simply a Coercion whose free variables have may be either
boxed or unboxed. After we are done with typechecking the desugarer finds the
boxed free variables, unboxes them, and creates a resulting real Coercion with
kosher free variables.



# 

# Evidence bindings


### Note: Typeable evidence terms

The EvTypeable data type looks isomorphic to Type, but the EvTerms
inside can be EvIds.  Eg
    f :: forall a. Typeable a => a -> TypeRep
    f x = typeRep (undefined :: Proxy [a])
Here for the (Typeable [a]) dictionary passed to typeRep we make
evidence
    dl :: Typeable [a] = EvTypeable [a]
                            (EvTypeableTyApp (EvTypeableTyCon []) (EvId d))
where
    d :: Typable a
is the lambda-bound dictionary passed into f.

### Note: Coercion evidence terms

A "coercion evidence term" takes one of these forms
   co_tm ::= EvId v           where v :: t1 ~# t2
           | EvCoercion co
           | EvCast co_tm co

We do quite often need to get a TcCoercion from an EvTerm; see
'evTermCoercion'.

### Note: Bind new Givens immediately

### Note: EvBinds/EvTerm

How evidence is created and updated. Bindings for dictionaries,
and coercions and implicit parameters are carried around in TcEvBinds
which during constraint generation and simplification is always of the
form (TcEvBinds ref). After constraint simplification is finished it
will be transformed to t an (EvBinds ev_bag).

Evidence for coercions *SHOULD* be filled in using the TcEvBinds
However, all EvVars that correspond to *wanted* coercion terms in
an EvBind must be mutable variables so that they can be readily
inlined (by zonking) after constraint simplification is finished.

Conclusion: a new wanted coercion variable should be made mutable.
[Notice though that evidence variables that bind coercion terms
 from super classes will be "given" and hence rigid]

### Note: KnownNat & KnownSymbol and EvLit

A part of the type-level literals implementation are the classes
"KnownNat" and "KnownSymbol", which provide a "smart" constructor for
defining singleton values.  Here is the key stuff from GHC.TypeLits

  class KnownNat (n :: Nat) where
    natSing :: SNat n

  newtype SNat (n :: Nat) = SNat Integer

Conceptually, this class has infinitely many instances:

  instance KnownNat 0       where natSing = SNat 0
  instance KnownNat 1       where natSing = SNat 1
  instance KnownNat 2       where natSing = SNat 2
  ...

In practice, we solve `KnownNat` predicates in the type-checker
(see typecheck/TcInteract.hs) because we can't have infinitely many instances.
The evidence (aka "dictionary") for `KnownNat` is of the form `EvLit (EvNum n)`.

We make the following assumptions about dictionaries in GHC:
  1. The "dictionary" for classes with a single method---like `KnownNat`---is
     a newtype for the type of the method, so using a evidence amounts
     to a coercion, and
  2. Newtypes use the same representation as their definition types.

So, the evidence for `KnownNat` is just a value of the representation type,
wrapped in two newtype constructors: one to make it into a `SNat` value,
and another to make it into a `KnownNat` dictionary.

Also note that `natSing` and `SNat` are never actually exposed from the
library---they are just an implementation detail.  Instead, users see
a more convenient function, defined in terms of `natSing`:

  natVal :: KnownNat n => proxy n -> Integer

The reason we don't use this directly in the class is that it is simpler
and more efficient to pass around an integer rather than an entier function,
especially when the `KnowNat` evidence is packaged up in an existential.

The story for kind `Symbol` is analogous:
  * class KnownSymbol
  * newtype SSymbol
  * Evidence: EvLit (EvStr n)

### Note: Overview of implicit CallStacks

(See https://ghc.haskell.org/trac/ghc/wiki/ExplicitCallStack/ImplicitLocations)

The goal of CallStack evidence terms is to reify locations
in the program source as runtime values, without any support
from the RTS. We accomplish this by assigning a special meaning
to constraints of type GHC.Stack.Types.HasCallStack, an alias

  type HasCallStack = (?callStack :: CallStack)

Implicit parameters of type GHC.Stack.Types.CallStack (the name is not
important) are solved in three steps:

1. Occurrences of CallStack IPs are solved directly from the given IP,
   just like a regular IP. For example, the occurrence of `?stk` in

     error :: (?stk :: CallStack) => String -> a
     error s = raise (ErrorCall (s ++ prettyCallStack ?stk))

   will be solved for the `?stk` in `error`s context as before.

2. In a function call, instead of simply passing the given IP, we first
   append the current call-site to it. For example, consider a
   call to the callstack-aware `error` above.

     undefined :: (?stk :: CallStack) => a
     undefined = error "undefined!"

   Here we want to take the given `?stk` and append the current
   call-site, before passing it to `error`. In essence, we want to
   rewrite `error "undefined!"` to

     let ?stk = pushCallStack <error's location> ?stk
     in error "undefined!"

   We achieve this effect by emitting a NEW wanted

     [W] d :: IP "stk" CallStack

   from which we build the evidence term

     EvCsPushCall "error" <error's location> (EvId d)

   that we use to solve the call to `error`. The new wanted `d` will
   then be solved per rule (1), ie as a regular IP.

   (see TcInteract.interactDict)

3. We default any insoluble CallStacks to the empty CallStack. Suppose
   `undefined` did not request a CallStack, ie

     undefinedNoStk :: a
     undefinedNoStk = error "undefined!"

   Under the usual IP rules, the new wanted from rule (2) would be
   insoluble as there's no given IP from which to solve it, so we
   would get an "unbound implicit parameter" error.

   We don't ever want to emit an insoluble CallStack IP, so we add a
   defaulting pass to default any remaining wanted CallStacks to the
   empty CallStack with the evidence term

     EvCsEmpty

   (see TcSimplify.simpl_top and TcSimplify.defaultCallStacks)

This provides a lightweight mechanism for building up call-stacks
explicitly, but is notably limited by the fact that the stack will
stop at the first function whose type does not include a CallStack IP.
For example, using the above definition of `undefined`:

  head :: [a] -> a
  head []    = undefined
  head (x:_) = x

  g = head []

the resulting CallStack will include the call to `undefined` in `head`
and the call to `error` in `undefined`, but *not* the call to `head`
in `g`, because `head` did not explicitly request a CallStack.


Important Details:
- GHC should NEVER report an insoluble CallStack constraint.

- GHC should NEVER infer a CallStack constraint unless one was requested
  with a partial type signature (See TcType.pickQuantifiablePreds).

- A CallStack (defined in GHC.Stack.Types) is a [(String, SrcLoc)],
  where the String is the name of the binder that is used at the
  SrcLoc. SrcLoc is also defined in GHC.Stack.Types and contains the
  package/module/file name, as well as the full source-span. Both
  CallStack and SrcLoc are kept abstract so only GHC can construct new
  values.

- We will automatically solve any wanted CallStack regardless of the
  name of the IP, i.e.

    f = show (?stk :: CallStack)
    g = show (?loc :: CallStack)

  are both valid. However, we will only push new SrcLocs onto existing
  CallStacks when the IP names match, e.g. in

    head :: (?loc :: CallStack) => [a] -> a
    head [] = error (show (?stk :: CallStack))

  the printed CallStack will NOT include head's call-site. This reflects the
  standard scoping rules of implicit-parameters.

- An EvCallStack term desugars to a CoreExpr of type `IP "some str" CallStack`.
  The desugarer will need to unwrap the IP newtype before pushing a new
  call-site onto a given stack (See DsBinds.dsEvCallStack)

- When we emit a new wanted CallStack from rule (2) we set its origin to
  `IPOccOrigin ip_name` instead of the original `OccurrenceOf func`
  (see TcInteract.interactDict).

  This is a bit shady, but is how we ensure that the new wanted is
  solved like a regular IP.



# Pretty printing
