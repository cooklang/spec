
# CookLang EBNF description (WIP)

```ebnf
recipe = { comments | metadata | step } ;


comments = "/", "/", { all characters } ; (* comments will be changed soon *)
metadata = ">", ">", multiword, ":", { all characters } ;
step = { direction item }, "\n" ;


direction item = multiword | ingredient | cookware | timer ;
ingredient = "@", ( singleword, [ "{", { white space }, [ amount ], { white space }, "}" ])
                | ( singleword, multiword, "{", { white space }, [ amount ], { white space }, "}" ) ;
cookware   = "#", ( singleword, [ "{", { white space }, [ quantity ], { white space }, "}" ])
                | ( singleword, multiword, "{", { white space }, [ quantity ], { white space }, "}" ) ;
timer      = "~", ( "{", { white space }, [ amount ], { white space }, "}" )
                | ( singleword, [ "{", { white space }, [ amount ], { white space }, "}" ])
                | ( singleword, multiword, "{", { white space }, [ amount ], { white space }, "}" ) ;                              


amount = quantity | ( quantity, { white space }, "%", { white space }, units ); (* '%' probably will be changed soon to '*' *)
quantity = number | multiword ;
units = multiword ;


multiword = { singleword | white space } ; (* TODO define better, should include symbols *)
singleword = { alphabetic character | digit } ;


number = integer | fractional | decimal ;
fractional = integer, { white space }, "/", { white space }, integer ;
decimal = integer, ".", integer ;
integer = zero | ( digit non-zero, { digit } ) ;
digit = zero | digit non-zero ;
digit non-zero = "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;
zero = "0" ;


alphabetic character = "A" | "B" | "C" | "D" | "E" | "F" | "G"
                     | "H" | "I" | "J" | "K" | "L" | "M" | "N"
                     | "O" | "P" | "Q" | "R" | "S" | "T" | "U"
                     | "V" | "W" | "X" | "Y" | "Z" 
                     | "a" | "b" | "c" | "d" | "e" | "f" | "g"
                     | "h" | "i" | "j" | "k" | "l" | "m" | "n"
                     | "o" | "p" | "q" | "r" | "s" | "t" | "u"
                     | "v" | "w" | "x" | "y" | "z" ;
special character = ">" | "@" | "#" | "~" | ":" | "{" | "}" | "%" ;
symbols = ? characters other than special character ?; (* TODO define better *)
white space = ? white space characters ? ;
all characters = ? all visible characters ? ;
```
