[[src]](https://github.com/ghc/ghc/tree/master/compiler/hsSyn/HsExtension.hs)
### Note: Trees that grow


See https://ghc.haskell.org/trac/ghc/wiki/ImplementingTreesThatGrow

The hsSyn AST is reused across multiple compiler passes. We also have the
Template Haskell AST, and the haskell-src-exts one (outside of GHC)

Supporting multiple passes means the AST has various warts on it to cope with
the specifics for the phases, such as the 'ValBindsOut', 'ConPatOut',
'SigPatOut' etc.

The growable AST will allow each of these variants to be captured explicitly,
such that they only exist in the given compiler pass AST, as selected by the
type parameter to the AST.

In addition it will allow tool writers to define their own extensions to capture
additional information for the tool, in a natural way.

A further goal is to provide a means to harmonise the Template Haskell and
haskell-src-exts ASTs as well.

