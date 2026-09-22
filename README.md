# indexion-kgf

Knowledge Graph Framework (KGF) language specifications for [indexion](https://github.com/trkbt10/indexion).

## Overview

KGF is a unified specification format for describing programming languages, DSLs, and natural languages. Each `.kgf` file defines lexical tokens, grammar rules, semantic actions, and module resolution strategies.

## Installation

```bash
# Clone to the OS data directory (recommended for binary distribution)
git clone https://github.com/trkbt10/indexion-kgf.git ~/.indexion/kgfs

# Or point indexion at a checkout anywhere
export INDEXION_KGFS_DIR=/path/to/indexion-kgf
```

indexion also finds a `kgfs/` directory in the project it is analysing, or in
the current working directory, without any configuration.

### Patching a single spec

The resolved spec set is layered: a base set plus any overlays applied on top.
A spec whose `language:` header matches one already loaded replaces it —
matched by language name, not by filename or directory. To change one spec,
you do not copy this repository; you put your file in the project's overlay
directory:

```
my-project/
├── .indexion/
│   └── kgfs/
│       └── programming/
│           └── rust.kgf   ← replaces the installed rust spec
└── src/
```

The overlay may mirror the layout above or be flat — only the `language:`
header decides what a file replaces. Every spec the overlay does not redefine
keeps coming from the installed set, and an overlay spec may `extends:` a spec
of the base set.

You can also give the chain explicitly. `--specs-dir` is repeatable: the first
occurrence is the base set, each later one an overlay layered on top.

```bash
indexion search "query" src/ --specs-dir=/opt/kgfs --specs-dir=./team-kgfs
```

Passing `--specs-dir` at all replaces the whole chain, so a single value is
still "use exactly this directory as the spec set". On `indexion kgf` the
option is spelled `--kgf-dir`; on `indexion digest`, `--specs`.

### Seeing which file won

```bash
indexion kgf list     # layers resolved, each spec's origin, and what it replaced
indexion kgf check    # validate every spec in the resolved set, overlay included
```

## Supported Languages

### Programming Languages (25)
C, C++, C#, Clojure, Dart, Elixir, Go, Haskell, Java, JavaScript, Julia, Kotlin, Lua, MoonBit, OCaml, PHP, Python, Ruby, Rust, Scala, Swift, TypeScript, Zig

### DSLs (7)
CSS, HTML, Markdown, SQL, SQL-DDL, MoonBit Package (moon.pkg), npm package.json

### Natural Languages (4)
Chinese, English, Japanese, Korean

### Project Files (13)
build.gradle.kts, Cargo.toml, composer.json, .csproj, deno.json, Gemfile, go.mod, moon.mod.json, package.json, Package.swift, pom.xml, pyproject.toml, vcpkg.json

## KGF Specification Format

### Version Header
```
kgf 0.6
language: <language-name>
sources: <file-extensions>
extends: <base-language-name>    # optional
```

### Which spec owns a file

`sources:` lists the extensions or file names a spec claims. When several
general-purpose specs claim the same pattern the last one registered wins and
`indexion kgf check --all` reports the collision; resolve it by giving the
pattern one owner. A spec that must stay loaded but never be auto-detected
(an experiment selected with `--spec`, an overlay picked by a specific
command) simply declares no `sources:`. `detection_scope: overlay` in
`=== features` keeps a spec out of the *competition* for a pattern, but a
sole claimant is still assigned it — omitting `sources:` is the only way to
claim nothing.

### Spec Inheritance (`extends:`)

Many specs describe a dialect of another: a wiki page is Markdown, TSX is
TypeScript. Rather than copying the base spec, declare `extends:` and write
only what differs.

```
kgf 0.6
language: sdd-user-story
sources: .md
extends: markdown

=== features
...markdown's features, plus this spec's own...

=== semantics
...markdown's blocks, plus this spec's own...
```

`extends:` names the base by its `language:`, not by a file path, so the base
may live in any subdirectory of the same spec set. Inheritance is resolved
after the whole directory is loaded, so declaration order and directory layout
do not matter.

**Section-level override.** For each `=== section`, the derived spec's text
wins *whole* if the derived spec declares that section at all; the base's is
used otherwise. Nothing is merged *within* a section:

| In the derived spec | Result |
|---|---|
| section omitted | the base's section is used unchanged |
| section declared with content | the base's section is replaced entirely |
| section declared, empty body | the base's section is replaced with nothing |

A derived `=== grammar` therefore replaces the base grammar rather than adding
rules to it, and a derived `=== semantics` replaces *every* `on` block of the
base. When a derived spec needs the base's blocks plus its own, it repeats the
base's blocks in its own section: the result of a merge can be read straight
off the two files.

`language:` and `sources:` are never inherited. Chains are allowed
(`A extends B extends C`); each section comes from the nearest ancestor that
declares it. A cycle, or an `extends:` naming a spec that is not present, is an
error: loading falls back to the spec's own sections and `indexion kgf check`
reports it. `kgf check` on a derived spec validates the *resolved* spec, and
`kgf list` shows the base in its `Extends` column.

### Sections

A KGF file consists of multiple sections, each starting with `===`:

#### `=== lex` - Lexical Rules

Defines token patterns using regular expressions:

```
SKIP /<pattern>/           # Ignored tokens (whitespace)
TOKEN <Name> /<pattern>/   # Named token
TOKEN <Name> /<pattern>/ strip=<n>  # Strip n chars from value
TOKEN <Name> /<pattern>/ skip_value # Ignore matched value
```

A token's *value* (what `$label` and `_text` carry) is the text of its first
participating capture group when the pattern has one, otherwise the matched
text with surrounding quotes stripped. Use `(?:...)` for grouping that must not
become the value: `/#(include|import)\s*(.*)/` exports `include`, while
`/#(?:include|import)\s*(.*)/` exports the path.

Example:
```
SKIP /[ \t]+/
TOKEN NL /\r?\n/
TOKEN String /"([^"\\]|\\.)*"/
TOKEN Keyword /(if|else|while|for|return)\b/
TOKEN Ident /[a-zA-Z_][a-zA-Z0-9_]*/
TOKEN Number /[0-9]+(\.[0-9]+)?/
```

##### `LAYOUT` — indentation-sensitive languages

`LAYOUT` is a directive rather than a pattern. A regex cannot compare one
line's indentation with the previous line's, so a regex-only lexer cannot tell
a nested block from a sibling one — which in Python, YAML or Haskell is the
whole of the nesting structure. `LAYOUT` declares how the engine recovers it:
a post-lex pass maintains an indentation stack and injects synthetic
INDENT/DEDENT tokens, so a grammar rule can bracket a body exactly, at any
depth.

```
LAYOUT newline=<Kind>[,<Kind>...] indent=<Kind> dedent=<Kind> [comment=<Kind>] [open=<Kind>,...] [close=<Kind>,...]
```

| Field | Required | Meaning |
|-------|----------|---------|
| `newline` | yes | token kind(s) whose text is a line break plus the *next* line's leading whitespace; everything after the break characters is that line's indentation. List several when the spec splits indented from column-0 breaks (`NL_INDENT` / `NL`) |
| `indent` | yes | synthetic kind emitted when a line is indented deeper than the enclosing one |
| `dedent` | yes | synthetic kind emitted once per level a shallower line closes |
| `comment` | no | comment token kind; a line whose only token is a comment carries no layout |
| `open` / `close` | no | bracket kinds; while a bracket is open, layout is suppressed |

```
TOKEN NL_INDENT /\r?\n[ \t]+/
TOKEN NL /\r?\n/
SKIP /[ \t]+/
TOKEN Comment /#[^\r\n]*/
LAYOUT newline=NL_INDENT,NL indent=INDENT dedent=DEDENT comment=Comment open=LPAREN,LBRACKET,LBRACE close=RPAREN,RBRACKET,RBRACE

# in the grammar: a suite is exactly one INDENT...DEDENT pair
FunctionDef -> KW_def id:Ident Params COLON ( SuiteOpen doc:Docstring? BodyItem* DEDENT / SimpleBody )
SuiteOpen   -> ( LineBreak / Comment )* INDENT
```

`indent` and `dedent` are synthetic: they must not have a `TOKEN` rule (no
source text matches them) but they are valid grammar symbols; `kgf check`
enforces both halves, reports a `LAYOUT` field naming an undefined kind, and
includes the synthetic kinds in the unreachable-token check. `kgf tokens`
shows them with an empty text. The pass ignores blank and comment-only lines,
closes every open level at end of input, and tolerates a dedent to a column
on no open level (it emits what it can and adopts the new column, since KGF
analyses files that do not compile). Indentation is compared as the raw
whitespace string's length, so one tab counts as one space.

#### `=== grammar` - PEG Grammar Rules

Defines parsing rules using PEG (Parsing Expression Grammar):

```
Rule -> Pattern1 Pattern2    # Sequence
Rule -> Alt1 / Alt2          # Ordered choice
Rule -> Item*                # Zero or more
Rule -> Item+                # One or more
Rule -> Item?                # Optional
Rule -> &Item                # Positive lookahead
Rule -> !Item                # Negative lookahead
```

Example:
```
Module -> Item*
Item -> FnDecl / StructDecl / Import
FnDecl -> KW_fn Ident LPAREN Params? RPAREN Block
Params -> Param (COMMA Param)*
```

#### `=== attrs` - Token Attributes

Defines semantic attributes for tokens:

```
<TokenName>: <attribute-list>
```

Attributes: `keyword`, `operator`, `literal`, `comment`, `string`, `type`, `identifier`

#### `=== features` - Consumer Metadata

Defines optional metadata that downstream tools can consume without hardcoding format-specific behavior:

```
reference_token_kinds: INLINE_CODE, Key
document_symbol_kinds: Function, Class, Type
coverage_token_kinds: Ident, TEXT, Key
```

Values are comma-separated. For example, `indexion plan reconcile` uses `reference_token_kinds` to decide which token kinds represent explicit symbol references in a document spec, `document_symbol_kinds` to decide which code symbol kinds are documentation targets for a programming spec, and `coverage_token_kinds` to build module-coverage term indexes without command-local text splitting rules.

#### `=== semantics` - Semantic Actions

Defines how matched grammar rules become graph nodes and edges. Each `on`
block runs when the named rule has matched; blocks fire **bottom-up** (a rule's
block runs after every block of its children).

```
on <RuleName> [when <expr>] {
  <statement>*
} [else { <statement>* }]
```

Statements:
- `edge <kind> from <expr> to <expr> [attrs <expr>]` - add an edge (`declares`, `moduleDependsOn`, ...)
- `let <name> = <expr>` - local value for the rest of the block
- `bind ns <expr> name <expr> to <expr>` - bind a name in the innermost scope frame
- `scope push` / `scope pop` - open / close a scope frame (see below)
- `note <type> [payload <expr>]` - attach a note (`module_doc`, ...)
- `module <expr> [file <expr>]` - register a module node
- `for <name> in <expr> { ... }` - iterate an array value

Values:
- `$label` - the value of a label bound in this rule's grammar expression
  (`id:Ident`). For a label bound to a multi-token sub-rule this is the LAST
  token's value; use `$label_text` for the accumulated text.
- `$file`, `$root`, `$language` - built-in context variables (they shadow
  locals of the same name; `kgf check` warns about such a `let`).
- `$scope(ns, name)` - look a binding up, innermost frame first.
- `$resolve(path)`, `$resolveFrom(base, path)`, `$findAncestor(marker)` - module resolution.
- Pure functions: `concat`, `obj`, `cond`, `eq`, `not`, `coalesce`, `trim`,
  `toLower`, `toUpper`, `startsWith`, `containsStr`, `beforeFirst`,
  `afterFirst`, `slice`, `countOccurrences`, `digits`, `add`, `sub`, `mul`,
  `div`, `floor`, `first`, `last`, `stripQuotes`, `regexCaptures`,
  `regexPairs`, `collectLinesAfterPrefixes`, `children`.
  `obj` drops null values, so an unbound label leaves its attribute out.

Example (declarations with their doc, members attached to their type):
```
on FnDecl when $vis {
  edge declares from $file to concat($file, "::", $id) attrs obj("name", $id, "kind", "Function", "doc", trim($doc_text))
}
```

##### Lexical scoping: `scope push` / `scope pop`

`bind` writes into the innermost scope frame and `$scope(ns, name)` searches
frames innermost-first. Without explicit frames every `bind` in a spec shares
one flat frame, so a nested declaration permanently clobbers its parent's
binding: after `class Outer { class Inner { ... } fn m() }`, `current_class`
still points at `Inner` when `m` fires. `scope push` and `scope pop` bracket a
declaration's members in their own frame; `scope pop` on the root frame is a
no-op, so an unbalanced spec degrades to the flat behaviour.

Because blocks fire bottom-up, the frame is opened by a sub-rule that
completes *before* the body (the declaration header) and closed by the
enclosing declaration rule, which fires last:

```
on ClassHeader {
  let sym_id = concat($file, "::", $id)
  edge declares from $file to sym_id attrs obj("name", $id, "kind", "Class")
  scope push
  bind ns "value" name "current_class" to sym_id
}

on MethodDecl when $scope("value", "current_class") {
  let parent = $scope("value", "current_class")
  edge declares from parent to concat(parent, ".", $id) attrs obj("name", $id, "kind", "Method")
}

on ClassDecl {
  scope pop
}
```

Bindings that must outlive the declaration (a `child_decl_sym` read by a later
sibling rule) go *before* the `scope push`. Symbols registered by `attrs def`
also land in the innermost frame: the name binding is lexically scoped, the
symbol node in the graph is permanent. `indexion kgf check` warns when a spec
uses `scope push` but never `scope pop`, or vice versa.

#### `=== resolver` - Module Resolution

Configures how imports are resolved:

```
relative_prefixes: ./, ../       # Relative import markers
exts: .js, .ts                   # File extensions to try
indexes: index                   # Index file names
bare_prefix: npm:                # External package prefix
module_path_style: slash         # slash | dot | coloncol | backslash
resolve:                         # Ordered probe chain (optional)
  - manifest: package.json @ main
  - namespace_map: tsconfig.json @ compilerOptions.paths + compilerOptions.baseUrl
  - index: /index.ts
  - ext: .ts
  - fallback: npm:
```

`module_path_style` names the separator of the language's module paths; the
resolver converts it to `/` and keeps a leading run of `relative_prefixes`
intact (`../a.b` → `../a/b`), so a spec can turn its own relative syntax into
those prefixes in `=== semantics` (python rewrites leading dots this way).

**Namespace mapping.** Import paths rarely map onto the directory tree
one-to-one. `namespace_map: <manifest> @ <field> [+ <base_field>]` reads a
prefix→directory map out of the nearest manifest above the importing file —
composer's `autoload.psr-4`, tsconfig's `compilerOptions.paths` — and rewrites
the module with it before the usual `ext`/`index`/`sibling` probes run. The
longest matching key wins; a key may use a `*` wildcard, and `+ <base_field>`
names a field the values are relative to (tsconfig's `baseUrl`).
`source_roots: <dirs> [@ <markers>]` does the same for layouts fixed by
convention rather than declared, such as `src/main/java`, anchored at the
nearest directory holding one of the markers. When nothing matches, the step
is skipped and the chain continues. `bare_prefix` and `ns_prefix` name modules
that were *not* found, so both apply only after every probe has missed.

### Complete Example

```
kgf 0.6
language: example
sources: .ex

=== lex
SKIP /[ \t]+/
TOKEN NL /\r?\n/
TOKEN Comment /#[^\n]*/
TOKEN String /"([^"\\]|\\.)*"/
KEYWORD KW_def /def\b/
KEYWORD KW_end /end\b/
TOKEN Ident /[a-z_][a-zA-Z0-9_]*/
TOKEN Number /[0-9]+/

=== grammar
Module -> Def*
Def -> KW_def Ident Body KW_end
Body -> Stmt*
Stmt -> Ident / Number / String

=== attrs

=== features
reference_token_kinds: Ident

KW_def: keyword
KW_end: keyword
Ident: identifier
Number: literal
String: string
Comment: comment

=== semantics
Def {
  emit(Function, name: $2.text)
}

=== resolver
relative_prefixes: ./
exts: .ex
indexes: index
bare_prefix:
module_path_style: slash
```

## Directory Structure

```
indexion-kgf/
├── programming/     # General-purpose languages
├── dsl/             # Domain-specific languages
├── natural/         # Natural languages
├── project/         # Build/config files
├── toy/             # Experimental specs
└── universal.kgf    # Mixed-content fallback
```

## Contributing

1. Fork this repository
2. Add your `.kgf` file to the appropriate directory
3. Test with indexion: `indexion explore --format=list your-files/`
4. Submit a pull request

## License

Apache-2.0
