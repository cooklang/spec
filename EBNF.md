
# Cooklang EBNF description

WIP, the EBNF is outdated and doesn't contain latest changes yet.

```ebnf
recipe = { metadata | step | note }- ;

(* not sure how to show that, but two below should start from a new line *)
metadata = ">", ">", { text item - ":" }-, ":", { text item }-, new line character ;
step     = { text item | ingredient | cookware | timer }-, new line character, blank line ;

note marker = ">" ;
note        = { note marker, note text, blank line }-, blank line ;

(* inline formatting in notes; which delimiters open and close emphasis, and
   how they pair, follows CommonMark 0.31.2 and can't be expressed here *)
note text   = { strong | emphasis | link | escape | text item } ;
strong      = ( "**", note text, "**" ) | ( "__", note text, "__" ) ;
emphasis    = ( "*", note text, "*" )   | ( "_", note text, "_" ) ;
link        = "[", note text, "]", "(", destination, [ white space, title ], ")" ;
destination = ( "<", { text item - "<" - ">" }, ">" ) | { text item - white space }- ;
title       = ( '"', { text item - '"' }, '"' )
            | ( "'", { text item - "'" }, "'" )
            | ( "(", { text item - ")" }, ")" ) ;
escape      = "\", text item ;

ingredient           = one word ingredient | multiword ingredient ;
one word ingredient  = "@", [ optional marker ], one word component ;
multiword ingredient = "@", [ optional marker ], multiword component ;

cookware             = one word cookware | multiword cookware ;
one word cookware    = "#", [ optional marker ], one word component ;
multiword cookware   = "#", [ optional marker ], multiword component ;

(* marks the ingredient or cookware as optional; not part of the name *)
optional marker      = "?" ;

timer                = no name timer | one word timer | multiword timer ;
no name timer        = "~", no name component ;
one word timer       = "~", one word component ;
multiword timer      = "~", multiword component ;

no name component    =                             "{", [ amount ], "}" ;
one word component   = word,                     [ "{", [ amount ], "}" ] ;
multiword component  = word, { text item - "{" }-, "{", [ amount ], "}" ;

amount   = quantity | ( quantity, "%", units ) ;
units    = { text item - "}" }- ;

(* tried in order against the whole quantity, surrounding white space trimmed:
   a quantity that matches range is a range, else one that matches number is
   a number, else it is text *)
quantity = range | number | { text item - "%" - "}" }- ;
range    = number, { white space }, range separator, { white space }, number ;

range separator = "-" | "–" | "—" ;  (* U+002D hyphen-minus, U+2013 en dash, U+2014 em dash *)

number         = decimal | mixed number | fraction | integer ;
integer        = "0" | non-zero digit, { digit } ;
decimal        = [ integer ], ".", digit, { digit } ;
fraction       = integer, { white space }, "/", { white space }, integer ;
mixed number   = integer, { white space }-, fraction ;
digit          = "0" | non-zero digit ;
non-zero digit = "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;

word      = { text item - white space - punctuation character }- ;
text item = ? any character except new line character ? ;

(* https://en.wikipedia.org/wiki/Template:General_Category_(Unicode) *)
new line character    = ? newline characters (U+000A ~ U+000D, U+0085, U+2028, and U+2029) ? ;
white space           = ? Unicode General Category Zs and CHARACTER TABULATION (U+0009) ? ;
punctuation character = ? Unicode General Category P* ? ;
blank line = { white space }, new line character ;

comments       = "-", "-", text item, new line character ;
block comments = "[", "-", ? any character except "-" followed by "]" ?, "-", "]" ;
```

# Symbols

|**Usage**|**Notation**
:-----:|:-----:
definition |<code>**=**</code>
concatenation |<code>,</code>
termination |<code>;</code>
alternation |<code>&#124;</code>
optional |<code>[ ... ]</code>
repetition |<code>{ ... }</code>
repetition (at least 1) |<code>{ ... }-</code>
grouping |<code>( ... )</code>
terminal string |<code>" ... "</code>
terminal string |<code>' ... '</code>
comment |<code>(* ... *)</code>
special sequence |<code>? ... ?</code>
exception |<code>-</code>
