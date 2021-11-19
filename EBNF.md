
# Cooklang EBNF description

```ebnf
recipe = { metadata | step } ;

(* not sure how to show that, but two below should start from a new line *)
metadata = ">", ">", multiword, ":", { white space }, text item | number| amount, new line character ;
step     = { text item | ingredient | cookware | timer }-, new line character ;


ingredient           = one word ingredient | multiword ingredient ;
one word ingredient  = "@", ( word,          [ "{", { white space }, [ amount ], { white space }, "}" ] ) ;
multiword ingredient = "@", ( word, multiword, "{", { white space }, [ amount ], { white space }, "}" ) ;

cookware             = one word cookware | multiword cookware ;
one word cookware    = "#", ( word,          [ "{", { white space }, [ quantity ], { white space }, "}" ] ) ;
multiword cookware   = "#", ( word, multiword, "{", { white space }, [ quantity ], { white space }, "}" ) ;

timer                = no name timer | one word timer | multiword timer ;
no name timer        = "~", (                  "{", { white space }, [ amount ], { white space }, "}" ) ;
one word timer       = "~", ( word,          [ "{", { white space }, [ amount ], { white space }, "}" ] ) ;
multiword timer      = "~", ( word, multiword, "{", { white space }, [ amount ], { white space }, "}" ) ;

(* '%' separator will be changed soon to '*' *)
amount   = quantity | ( quantity, { white space }, "%", { white space }, units );
quantity = number | multiword ;
units    = multiword | punctuation character;


multiword = { word | white space }- ;
(* yay, emoji! *)
word      = { alphabetic character | digit | symbol character - cooklang ancillary character }- ;
text item = { alphabetic character | digit | symbol character | punctuation character | white space }- ;

number         = integer | fractional | decimal ;
fractional     = integer, { white space }, "/", { white space }, integer ;
decimal        = integer, ".", integer ;
integer        = zero | ( non-zero digit, { digit } ) ;
digit          = zero | non-zero digit ;
non-zero digit = "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;
zero           = "0" ;

cooklang ancillary character = (* from symbols set *) ">" | "|" | "~" | (* from punctuation set *)  "@" | "#" | ":" | "{" | "}" | "%" ;
(* https://en.wikipedia.org/wiki/Template:General_Category_(Unicode) *)
alphabetic character = ? Unicode General Categories L* and M* ?;
white space = ? Unicode General Category Zs and CHARACTER TABULATION (U+0009) ? ;
new line character = ? newline characters (U+000A ~ U+000D, U+0085, U+2028, and U+2029) ? ;
punctuation character = ? Unicode General Category P* ? ;
symbol character = ? Unicode General Category S* ? ;

comments = "-", "-", ? any character ?, new line character ;
block comments = "[", "-", ? any character ?, "-", "]" ;
```
