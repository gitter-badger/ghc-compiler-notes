[[src]](https://github.com/ghc/ghc/tree/master/compiler/basicTypes/Lexeme.hs)


# Lexical categories


These functions test strings to see if they fit the lexical categories
defined in the Haskell report.

### Note: Classification of generated names


Some names generated for internal use can show up in debugging output,
e.g.  when using -ddump-simpl. These generated names start with a $
but should still be pretty-printed using prefix notation. We make sure
this is the case in isLexVarSym by only classifying a name as a symbol
if all its characters are symbols, not just its first one.




# Detecting valid names for Template Haskell


