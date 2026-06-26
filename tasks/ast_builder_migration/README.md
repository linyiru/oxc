# `AstBuilder` migration codemod

[ast-grep](https://ast-grep.github.io) rules that do two things:

1. **Migrate** code from the old `AstBuilder` (methods on the `AstBuilder` struct,
   e.g. `self.ast.null_literal(span)`) to the new builder methods defined directly on AST types
   (e.g. `NullLiteral::new(span, self)`).
2. **Shorten** the result, collapsing verbose new-builder call shapes into their shorthand
   equivalents (e.g. `Statement::VariableDeclaration(VariableDeclaration::boxed(..))` ->
   `Statement::new_variable_declaration(..)`).

See <https://github.com/oxc-project/oxc/issues/23043> for background.

## How it works

1. `AstBuilderGenerator` (in `tasks/ast_tools`) emits two data files, regenerated with `just ast`:
   - [`generated/mappings.json`](generated/mappings.json) maps each old `AstBuilder` method name to
     the equivalent new method on the AST type, e.g. `null_literal` -> `NullLiteral::new`,
     `alloc_null_literal` -> `NullLiteral::boxed`, `statement_expression` ->
     `Statement::new_expression_statement`.
   - [`generated/shorten_mappings.json`](generated/shorten_mappings.json) holds the structural data
     the shortening rules need: which structs are boxable, each enum variant's inner builder, and the
     enum inheritance graph.

   Generating both from the same code that builds the method names means they capture the name
   de-duplication, reserved-word, and `_with_*` default-field quirks exactly.

2. [`custom_mappings.json`](custom_mappings.json) adds, by hand, the methods that are written by hand
   rather than emitted by codegen (e.g. `void_0` -> `Expression::new_void_0`).
3. [`generate_rules.mts`](generate_rules.mts) combines these and writes
   [`generated/rules.yml`](generated/rules.yml) - the migration rules (one per builder method) plus
   the shortening rules. Run with `node tasks/ast_builder_migration/generate_rules.mts`.
4. Apply the rules to a crate (or any path) with [`run.sh`](run.sh):

   ```sh
   tasks/ast_builder_migration/run.sh crates apps napi --globs '!**/generated/**'
   ```

   All arguments are forwarded to `ast-grep scan`. `ast-grep` applies a single pass per invocation
   and does not re-scan its own output, but the rules cascade (one rule's output is another's input,
   and nested calls collapse one level per pass), so `run.sh` re-runs `ast-grep scan --update-all`
   until a pass makes no changes. Always exclude generated files (`--globs '!**/generated/**'`) - they
   contain the builder method bodies the shortening rules are derived from. Run `just fmt` afterwards
   to tidy formatting.

## Migration rules

Each migration rule rewrites a call on the old builder to a call to the new method, appending the
accessor (the base of the `<accessor>.ast` receiver) as the final argument:

```rs
self.ast.null_literal(span)             // -> NullLiteral::new(span, self)
p.ast.alloc_object_expression(span, x)  // -> ObjectExpression::boxed(span, x, p)
self.ast.number_0()                     // -> Expression::new_number_0(self)
```

The accessor (`self` / `p`) must implement `GetAstBuilder`.

## Shortening rules

These run on already-migrated code, collapsing verbose shapes into the shorthand constructors. They
are purely structural - they never mention `<accessor>.ast` - and come in three kinds:

```rs
// box-collapse: ArenaBox::new_in(T::new(..), x)  ->  T::boxed(..)
ArenaBox::new_in(VariableDeclaration::new(span, kind, decls, declare, b), x)
// -> VariableDeclaration::boxed(span, kind, decls, declare, b)

// variant-wrap: E::Variant(Inner::boxed(..))  ->  E::new_variant(..)
Statement::VariableDeclaration(VariableDeclaration::boxed(span, kind, decls, declare, b))
// -> Statement::new_variable_declaration(span, kind, decls, declare, b)

// inherited-from: Outer::from(Inner::new_x(..))  ->  Outer::new_x(..)
Statement::from(Declaration::new_variable_declaration(span, kind, decls, declare, b))
// -> Statement::new_variable_declaration(span, kind, decls, declare, b)
```

`variant-wrap` also covers inherited variants (e.g. `Statement::VariableDeclaration`, inherited from
`Declaration`). `box-collapse` discards the outer allocator argument (`x` above); in real code it is
always the same value as the inner builder argument, so the rewrite is behaviour-preserving.

## Reuse by downstream consumers (e.g. Rolldown)

Only the generated `rules.yml` is Oxc-specific; the generator and script are not.

- The **shortening** rules are accessor-agnostic (they match only new-builder output shapes), so they
  can be reused unchanged.
- The **migration** rules assume the builder is reached via `<accessor>.ast`. A consumer reaching it
  by another path should adjust the `pattern` / `fix` templates in `generate_rules.mts` and
  regenerate.
- All rules use bare type names (`Expression::new_*`), so code that refers to AST types through a
  module alias (as `oxc_react_compiler` does, via `use oxc_ast::ast as oxc;` -> `oxc::Expression`) is
  not matched. Such code needs prefixed rule variants or a hand-edit.
