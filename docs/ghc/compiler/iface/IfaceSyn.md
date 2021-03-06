[[src]](https://github.com/ghc/ghc/tree/master/compiler/iface/IfaceSyn.hs)

(c) The University of Glasgow 2006
(c) The GRASP/AQUA Project, Glasgow University, 1993-1998


# Declarations


### Note: Versioning of instances

See [http://ghc.haskell.org/trac/ghc/wiki/Commentary/Compiler/RecompilationAvoidance#Instances]

# Functions over declarations


# Expressions


### Note: Empty case alternatives

### Note: Empty case alternatives

### Note: Expose recursive functions

For supercompilation we want to put *all* unfoldings in the interface
file, even for functions that are recursive (or big).  So we need to
know when an unfolding belongs to a loop-breaker so that we can refrain
from inlining it (except during supercompilation).

### Note: IdInfo on nested let-bindings

Occasionally we want to preserve IdInfo on nested let bindings. The one
that came up was a NOINLINE pragma on a let-binding inside an INLINE
function.  The user (Duncan Coutts) really wanted the NOINLINE control
to cross the separate compilation boundary.

In general we retain all info that is left by CoreTidy.tidyLetBndr, since
that is what is seen by importing module with --make

# Printing IfaceDecl


### Note: Minimal complete definition

The minimal complete definition should only be included if a complete
class definition is shown. Since the minimal complete definition is
anonymous we can't reuse the same mechanism that is used for the
filtering of method signatures. Instead we just check if anything at all is
filtered and hide it in that case.


### Note: Printing IfaceDecl binders

The binders in an IfaceDecl are just OccNames, so we don't know what module they
come from.  But when we pretty-print a TyThing by converting to an IfaceDecl
(see PprTyThing), the TyThing may come from some other module so we really need
the module qualifier.  We solve this by passing in a pretty-printer for the
binders.

When printing an interface file (--show-iface), we want to print
everything unqualified, so we can just print the OccName directly.


# MINIMAL" <+>
        pprBooleanFormula
          (\_ def -> cparen (isLexSym def) (ppr def)) 0 minDef <+>
        text "#

### Note: Result type of a data family GADT

Consider
   data family T a
   data instance T (p,q) where
      T1 :: T (Int, Maybe c)
      T2 :: T (Bool, q)

The IfaceDecl actually looks like

   data TPr p q where
      T1 :: forall p q. forall c. (p~Int,q~Maybe c) => TPr p q
      T2 :: forall p q. (p~Bool) => TPr p q

To reconstruct the result types for T1 and T2 that we
want to pretty print, we substitute the eq-spec
[p->Int, q->Maybe c] in the arg pattern (p,q) to give
   T (Int, Maybe c)
Remember that in IfaceSyn, the TyCon and DataCon share the same
universal type variables.

----------------------------- Printing IfaceExpr ------------------------------------


" <+> pprWithCommas ppr is
                     <+> text "

# Finding the Names in IfaceSyn


This is used for dependency analysis in MkIface, so that we
fingerprint a declaration before the things that depend on it.  It
is specific to interface-file fingerprinting in the sense that we
don't collect *all* Names: for example, the DFun of an instance is
recorded textually rather than by its fingerprint when
fingerprinting the instance, so DFuns are not dependencies.


### Note: Tracking data constructors

In a case expression
   case e of { C a -> ...; ... }
You might think that we don't need to include the datacon C
in the free names, because its type will probably show up in
the free names of 'e'.  But in rare circumstances this may
not happen.   Here's the one that bit me:

### Note: Lazy deserialization of IfaceId

The use of lazyPut and lazyGet in the IfaceId Binary instance is
purely for performance reasons, to avoid deserializing details about
identifiers that will never be used. It's not involved in tying the
knot in the type checker. It saved ~1% of the total build time of GHC.

When we read an interface file, we extend the PTE, a mapping of Names
to TyThings, with the declarations we have read. The extension of the
PTE is strict in the Names, but not in the TyThings themselves.
LoadIface.loadDecl calculates the list of (Name, TyThing) bindings to
add to the PTE. For an IfaceId, there's just one binding to add; and
the ty, details, and idinfo fields of an IfaceId are used only in the
TyThing. So by reading those fields lazily we may be able to save the
work of ever having to deserialize them (into IfaceType, etc.).

For IfaceData and IfaceClass, loadDecl creates extra implicit bindings
(the constructors and field selectors of the data declaration, or the
methods of the class), whose Names depend on more than just the Name
of the type constructor or class itself. So deserializing them lazily
would be more involved. Similar comments apply to the other
constructors of IfaceDecl with the additional point that they probably
represent a small proportion of all declarations.
