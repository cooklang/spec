# Unit system metadata

* Proposal: [0020-unit-system](0020-unit-system.md)
* Authors: [Luka Hummel](https://github.com/LukaHummel)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

This proposal introduces an optional, recipe-level `unit system` metadata key
that declares the source unit system used by a recipe. The metadata gives
clients a portable hint for interpreting units and, when they choose to support
it, converting quantities into a reader's preferred unit system. It does not
require conversion, prescribe conversion tables, or change the units stored in
the recipe.

Discussion thread: [Support for Unit and Serving Size Conversions](https://github.com/cooklang/spec/discussions/142).

## Motivation

[Proposal 0010](0010-servings.md) standardises serving-based quantity scaling,
but Cooklang does not yet specify how clients should interpret a recipe's unit
system or offer unit conversion. As a result, each client must infer the source
system entirely from unit strings. This works for unambiguous units such as `g`,
but names such as `cup`, `pint`, and `fluid ounce` have different definitions in
the British Imperial and US customary systems.

Standardising a universal conversion table in Cooklang would also require the
language to decide matters that belong to clients and users, including regional
unit definitions, rounding, fractions, significant digits, preferred output
units, and display conventions.

The `unit system` metadata key fills the interoperability gap without moving
those policy decisions into the language. Recipe authors can declare the source
system, while clients remain free to use their own conversion data and user
preferences.

## Proposed solution

Add `unit system` as an optional canonical metadata key with one of three
lowercase scalar values:

| Value | Meaning |
| --- | --- |
| `metric` | The recipe is primarily written using the metric system. |
| `imperial` | The recipe is primarily written using the British Imperial system. |
| `us customary` | The recipe is primarily written using the US customary system. |

For example, a US customary recipe can declare:

```cooklang
---
unit system: us customary
servings: 4
---

Mix @flour{2%cups} with @milk{1%cup}.
```

A metric recipe can declare:

```cooklang
---
unit system: metric
servings: 4
---

Mix @flour{500%g} with @milk{250%ml}.
```

An Imperial recipe can distinguish an Imperial pint from a US customary pint:

```cooklang
---
unit system: imperial
---

Add @milk{1%pt}.
```

A client may use this metadata and the units attached to individual quantities
to render either recipe in the user's preferred system. Rendering a converted
quantity does not modify the source `.cook` file.

The metadata describes the recipe's **source** system. It is not a request to
convert the recipe and does not declare the author's preferred output system.
The target system, if any, comes from client configuration or an explicit user
choice.

## Detailed design

### Metadata syntax and values

The key is a scalar value in YAML front matter:

```yaml
---
unit system: metric
---
```

The canonical values are exactly `metric`, `imperial`, and `us customary`,
written in lowercase. The multiword `us customary` value does not need to be
quoted in YAML.

The existing case-insensitive handling of canonical metadata keys applies to the
`unit system` key. Clients may recognise additional spellings, aliases, or
systems as private extensions, but portable recipes cannot rely on those
extensions.

The metadata is optional. A missing value preserves existing behaviour and does
not prevent a client from inferring a system from explicit units. An unknown
value must not make the recipe invalid: clients retain it as metadata but cannot
treat it as a standardised source-system hint.

### Unit interpretation and conversion

A client that supports unit conversion may:

1. Read the `unit system` metadata.
2. Parse each numeric quantity and its explicit unit normally.
3. Resolve ambiguous units using the declared source system together with its
   own locale and unit configuration.
4. Treat an explicit recognised unit as authoritative when it conflicts with
   the recipe-level declaration, allowing recipes to mix systems intentionally.
5. Convert only between compatible physical dimensions, such as mass to mass or
   volume to volume.
6. Apply its own rounding, fraction, significant-digit, and best-fit rules.
7. Preserve the original quantity whenever conversion is unavailable or unsafe.

If a unit name exists in more than one configured system, the `unit system`
value selects the source-side definition. If the unit unambiguously identifies a
different system, the explicit unit takes precedence. For example, `2%oz` in a
recipe marked `unit system: metric` still means ounces.

Unitless quantities, textual quantities, and unknown units are not convertible
through this metadata and remain unchanged. A conversion failure for one
quantity must not prevent the rest of the recipe from being displayed.

### Client-owned policy

Conversion is an optional application feature. Supporting clients may normalise
quantities to a best-fitting unit, such as `1000 g` to `1 kg`, or display a
scaled `8 tbsp` as `1/2 cup`. Neither behaviour is required.

This proposal does not standardise:

* conversion ratios;
* regional definitions of units such as cups;
* rounding, approximation, or fraction rules;
* preferred output units or unit systems;
* the format of a client's conversion configuration; or
* ingredient-specific density data.

In particular, mass-to-volume and volume-to-mass conversions are outside the
scope of this proposal because they require ingredient-specific density or
equivalence data. For example, a client cannot reliably convert a cup of flour
to grams from the `unit system` metadata alone.

Display conversion should not rewrite a recipe's source. A client may provide a
separate, explicit operation that rewrites quantities when requested by the
user.

### Grammar and parser output

No Cooklang grammar change is required. YAML front matter already accepts custom
metadata, and existing parsers can expose `unit system` through their generic
metadata representation.

### Interoperability scenarios

| Recipe input | Client context | Expected behaviour |
| --- | --- | --- |
| `unit system: us customary` and `1%cup` | Metric output requested | The client may convert using the US customary cup definition in its configured unit catalogue. |
| `unit system: imperial` and `1%pt` | Metric output requested | The client may convert using the Imperial pint definition rather than the smaller US customary pint. |
| `unit system: metric` and `500%g` | US customary output requested | The client may convert the mass to ounces or pounds using its preferred output and rounding rules. |
| `unit system: metric` and `2%oz` | Any output system | The explicit recognised unit is authoritative; the recipe is not invalid. |
| No `unit system` metadata | Any output system | Existing behaviour is preserved; a client may still infer the source system from units. |
| `unit system: kitchen` | Any output system | The value is retained as metadata but has no standardised meaning. |
| An unknown or textual unit | Conversion requested | The original quantity and unit remain visible. |
| A unitless quantity | Conversion requested | No unit conversion is attempted. |
| `1%cup` of flour | Mass output requested | No volume-to-mass conversion is implied without separate ingredient-density data. |
| A scaled quantity such as `8%tbsp` | Best-fit display enabled | The client may display `1/2 cup`; unit selection and presentation remain client-owned. |

## Effect on applications which use Cooklang

Applications without unit-conversion support require no changes and continue to
render recipes as written. Supporting applications can read `unit system` from
the existing metadata representation and expose a separate user or application
preference for the target system.

Conversion should be handled independently for each quantity. When conversion
is not possible, the original value remains visible. Structured exports retain
the original metadata and quantities unless the user explicitly requests a
converted export.

### CookCLI (terminal and web-server)

No new CookCLI flag or subcommand is prescribed by this proposal. CookCLI may
use `unit system` if it adds a client-owned unit-conversion preference or an
explicit conversion operation in the future. Existing command output remains
unchanged when conversion is not requested or supported.

### Mobile applications

Mobile applications may offer a unit-system preference or a conversion control
in recipe views. When enabled, the application can use `unit system` to
interpret ambiguous source units and display converted quantities. The original
recipe remains unchanged, and unsupported quantities continue to be displayed
as written.

## Alternatives considered

### Infer the system from unit strings alone

This requires no metadata but cannot reliably distinguish units that share a
name across systems or regions. It also leaves authors without a portable way to
state the recipe's source system.

### Treat `unit system` as the desired output system

This would let the recipe author control presentation rather than describe the
source data. The reader's locale and preferences are a better source for output
selection.

### Use a boolean conversion opt-in

A boolean would indicate permission without providing the source-system context
needed to resolve ambiguous units.

### Support a system per physical dimension

A mapping could declare metric mass and imperial volume separately. Explicit
units already handle intentional mixing, and a structured mapping adds
complexity that is unnecessary for the initial convention.

### Declare preferred target units

Listing target units in recipe metadata would couple recipe data to a particular
presentation policy and would not account for reader preferences or regional
conventions.

### Embed conversion ratios or density tables

Recipe-local conversion data would be verbose, difficult to validate, and easy
for recipes and clients to implement inconsistently. Ingredient densities may
be useful in a separate proposal, but are not required for conversions within a
physical dimension.

### Require alternate quantities in ingredient syntax

Storing both values, for example a volume and mass for every ingredient, shifts
conversion work to recipe authors and risks values becoming inconsistent after
editing or scaling.

## Acknowledgments

Thanks to [RPGillespie6](https://github.com/RPGillespie6),
[fellmann](https://github.com/fellmann),
[SkepticMystic](https://github.com/SkepticMystic), and
[stevenbell](https://github.com/stevenbell) for raising and expanding the unit
conversion use cases in [discussion #142](https://github.com/cooklang/spec/discussions/142).
The earlier [discussion about unit keywords and internationalisation](https://github.com/cooklang/spec/discussions/51)
also helped establish the need for client-owned conversion tables and regional
preferences.
