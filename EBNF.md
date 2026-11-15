# Cooklang EBNF description

Grammar of a `.cook` recipe file, as described in the [specification](README.md). The notation is listed under [Symbols](#symbols).

```ebnf
recipe = [ front matter ], { blank line | section line | note | step } ;

(* YAML (https://yaml.org/spec/1.2.2/), only at the very beginning of the file *)
front matter = fence, new line character, yaml, fence, line end ;
fence        = "---", { white space } ;
yaml         = ? YAML document, none of its lines is a fence ? ;

(* takes one line; the name is plain text, with no ingredients, cookware or timers in it *)
section line = "=", { "=" }, [ section name ], { "=" }, { white space }, line end ;
section name = { character - "=" }- ;

(* notes and steps are paragraphs: lines that follow one another with no blank line between them *)

(* a paragraph that starts with ">"; plain text as well *)
note        = note marker, note text, { line break, [ note marker ], note text }, line end ;
note marker = ">" ;
note text   = { character } ;

(* any other paragraph *)
step = { ingredient | cookware | timer | text item }-, line end ;

(* an ingredient whose name starts with "./" or "../" is a reference to another recipe *)
ingredient           = one word ingredient | multiword ingredient ;
one word ingredient  = "@", [ optional marker ], one word component, [ preparation ] ;
multiword ingredient = "@", [ optional marker ], multiword component, [ preparation ] ;

cookware             = one word cookware | multiword cookware ;
one word cookware    = "#", [ optional marker ], one word component ;
multiword cookware   = "#", [ optional marker ], multiword component ;

(* marks the ingredient or cookware as optional; not part of the name *)
optional marker      = "?" ;

(* follows the ingredient directly, with no white space in between *)
preparation          = "(", { text item - ")" }, ")" ;

timer                = no name timer | one word timer | multiword timer ;
no name timer        = "~", no name component ;
one word timer       = "~", one word component ;
multiword timer      = "~", multiword component ;

no name component    =                            "{", amount, "}" ;
one word component   = word,                    [ "{", [ amount ], "}" ] ;
multiword component  = name start, { name item }, "{", [ amount ], "}" ;

(* a multiword name runs up to the "{" and cannot hold the start of another component *)
name start = character - white space - "{" - "@" - "#" - "~" ;
name item  = name start | white space | line break ;

amount       = { white space }, [ fixed marker ], quantity, [ "%", units ] ;
units        = { text item - "}" } ;

(* keeps the quantity as written when the recipe is scaled *)
fixed marker = "=" ;

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

word      = { character - white space - punctuation character }- ;
text item = character | line break ;

(* a new line inside a paragraph: the line after it is not blank and does not start with "=" *)
line break = soft break | hard break ;
soft break = new line character ;        (* read as a space *)
hard break = "\", new line character ;   (* kept as a line break *)

blank line = { white space }, line end ;
line end   = new line character | ? end of file ? ;

character = ? any character except new line character ? ;

(* https://en.wikipedia.org/wiki/Template:General_Category_(Unicode) *)
new line character    = ? newline characters (U+000A ~ U+000D, U+0085, U+2028, and U+2029), U+000D U+000A counts as one ? ;
white space           = ? Unicode General Category Zs and CHARACTER TABULATION (U+0009) ? ;
punctuation character = ? Unicode General Category P* ? ;

(* comments are not part of the text and can appear anywhere after the front matter *)
line comment  = "--", { character } ;
block comment = "[-", ? any characters except the sequence "-]" ?, "-]" ;
```

# Notes

The grammar alone allows more than one reading of some input. These rules settle it:

* The front matter is split off first, then comments are removed, and the rest is parsed after that. A line that holds only a comment is therefore a blank line.
* Blocks are tried in the order blank line, section line, note, step. A line that starts with `=` is always a block of its own: a section line when it fits the rule, a one-line step when it does not.
* `@`, `#` and `~` start an ingredient, cookware or timer only when the text after them fits one of the rules, and the longest fit is taken: `@ground black pepper{}` is one ingredient, `@salt and @pepper{}` is two. Otherwise the character is plain text, as in `@ example{}`.
* A `?` straight after `@` or `#` is always the optional marker, and a `=` that opens an amount is always the fixed marker.
* White space around a name, section name, quantity or units is not part of it. Braces that hold only white space hold no amount.
* A line that starts with `>>` is the deprecated one-line metadata form, `>> key: value`, and not a note. The front matter replaces it and this grammar does not cover it.

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
