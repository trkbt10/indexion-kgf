# indexion-kgf

Knowledge Graph Framework (KGF) language specifications for [indexion](https://github.com/trkbt10/indexion).

## Overview

KGF is a unified specification format for describing programming languages, DSLs, and natural languages. Each `.kgf` file defines lexical tokens, grammar rules, semantic actions, and module resolution strategies.

## Installation

```bash
# Clone to user config directory (recommended for binary distribution)
git clone https://github.com/trkbt10/indexion-kgf.git ~/.indexion/kgfs

# Or set environment variable
export INDEXION_KGFS_DIR=/path/to/indexion-kgf
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
```

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

Example:
```
SKIP /[ \t]+/
TOKEN NL /\r?\n/
TOKEN String /"([^"\\]|\\.)*"/
TOKEN Keyword /(if|else|while|for|return)\b/
TOKEN Ident /[a-zA-Z_][a-zA-Z0-9_]*/
TOKEN Number /[0-9]+(\.[0-9]+)?/
```

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

#### `=== semantics` - Semantic Actions

Defines actions to build semantic graphs:

```
<RuleName> {
  <action>
}
```

Actions:
- `emit(<type>, <fields>)` - Create a node
- `link(<relation>, <target>)` - Create an edge
- `scope(<name>)` - Enter a scope
- `end_scope` - Exit scope

Example:
```
FnDecl {
  emit(Function, name: $2.text, params: $4)
  scope($2.text)
}
```

#### `=== resolver` - Module Resolution

Configures how imports are resolved:

```
relative_prefixes: ./, ../       # Relative import markers
exts: .js, .ts                   # File extensions to try
indexes: index                   # Index file names
bare_prefix: npm:                # External package prefix
module_path_style: slash         # Path separator style
```

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
TOKEN KW_def /def\b/
TOKEN KW_end /end\b/
TOKEN Ident /[a-z_][a-zA-Z0-9_]*/
TOKEN Number /[0-9]+/

=== grammar
Module -> Def*
Def -> KW_def Ident Body KW_end
Body -> Stmt*
Stmt -> Ident / Number / String

=== attrs
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
