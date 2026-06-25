# `AstBuilder` migration codemod

Tooling to migrate code from the old `AstBuilder` (methods on the `AstBuilder` struct,
e.g. `self.ast.null_literal(span)`) to the new builder methods defined directly on AST types
(e.g. `NullLiteral::new(span, self)`).

See <https://github.com/oxc-project/oxc/issues/23043> for background.

## How it works

1. `AstBuilderGenerator` (in `tasks/ast_tools`) emits [`generated/mappings.json`](generated/mappings.json),
   mapping each old `AstBuilder` method name to the equivalent new method on the AST type, e.g.
   `null_literal` -> `NullLiteral::new`, `alloc_null_literal` -> `NullLiteral::boxed`,
   `statement_expression` -> `Statement::new_expression_statement`. Generating it from the same code
   that builds the method names means it captures the name de-duplication and reserved-word quirks
   exactly. Regenerate with `just ast`.
2. [`custom_mappings.json`](custom_mappings.json) adds, by hand, the methods that are written by hand
   rather than emitted by codegen (e.g. `void_0` -> `Expression::new_void_0`).
3. [`generate_rules.mts`](generate_rules.mts) merges the two and writes
   [`generated/rules.yml`](generated/rules.yml) - one [ast-grep](https://ast-grep.github.io) rule per
   method. Run with `node tasks/ast_builder_migration/generate_rules.mts`.
4. Apply the rules to a crate (or any path) with:

   ```sh
   ast-grep scan --rule tasks/ast_builder_migration/generated/rules.yml --update-all crates/oxc_parser
   ```

Each rule rewrites a call on the old builder to a call to the new method, appending the accessor
(the base of the `<accessor>.ast` receiver) as the final argument:

```rs
self.ast.null_literal(span)             // -> NullLiteral::new(span, self)
p.ast.alloc_object_expression(span, x)  // -> ObjectExpression::boxed(span, x, p)
self.ast.number_0()                     // -> Expression::new_number_0(self)
```

The accessor (`self` / `p`) must implement `GetAstBuilder`.

## Reuse by downstream consumers (e.g. Rolldown)

The mapping and script are tool-agnostic; only the generated `rules.yml` is Oxc-specific.
A consumer whose builder is reached by a path other than `<accessor>.ast` should adjust the `pattern` / `fix`
templates in `generate_rules.mts` and regenerate.
