
# CookLang EBNF description

```ebnf
recipe = { comments | metadata | step } ;

(* comments will be changed soon to '--' *)
comments = "/", "/", { all characters } ;
metadata = "^", ">", ">", multiword, ":", { white space }, text_item | number| amount, "\n" ;
step     = "^", { text_item | ingredient | cookware | timer }, "\n" ;


ingredient           = one_word_ingredient | multiword_ingredient ;
one_word_ingredient  = "@", ( word,          [ "{", { white space }, [ amount ], { white space }, "}" ]) ;
multiword_ingredient = "@", ( word, multiword, "{", { white space }, [ amount ], { white space }, "}" ) ;

cookware             = one_word_cookware | multiword_cookware ;
one_word_cookware    = "#", ( word,          [ "{", { white space }, [ quantity ], { white space }, "}" ]) ;
multiword_cookware   = "#", ( word, multiword, "{", { white space }, [ quantity ], { white space }, "}" ) ;

timer                = no_name_timer | one_word_timer | multiword_timer ;
no_name_timer        = "~", (                  "{", { white space }, [ amount ], { white space }, "}" ) ;
one_word_timer       = "~", ( word,          [ "{", { white space }, [ amount ], { white space }, "}" ]) ;
multiword_timer      = "~", ( word, multiword, "{", { white space }, [ amount ], { white space }, "}" ) ;

(* '%' separator will be changed soon to '*' *)
amount   = quantity | ( quantity, { white space }, "%", { white space }, units );
quantity = number | multiword ;
units    = multiword ;


multiword = { word | white space } ;
word      = { alphabetic character | digit } ;
text_item = { all characters }

number         = integer | fractional | decimal ;
fractional     = integer, { white space }, "/", { white space }, integer ;
decimal        = integer, ".", integer ;
integer        = zero | ( non-zero digit, { digit } ) ;
digit          = zero | non-zero digit ;
non-zero digit = "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;
zero           = "0" ;


alphabetic character = "A" | "B" | "C" | "D" | "E" | "F" | "G"
                     | "H" | "I" | "J" | "K" | "L" | "M" | "N"
                     | "O" | "P" | "Q" | "R" | "S" | "T" | "U"
                     | "V" | "W" | "X" | "Y" | "Z"
                     | "a" | "b" | "c" | "d" | "e" | "f" | "g"
                     | "h" | "i" | "j" | "k" | "l" | "m" | "n"
                     | "o" | "p" | "q" | "r" | "s" | "t" | "u"
                     | "v" | "w" | "x" | "y" | "z" ;
(* special characters = ">" | "@" | "#" | "~" | ":" | "{" | "}" | "%" ; *)
white space = ? white space characters ? ;
all characters = ? all visible characters ? ;
```
