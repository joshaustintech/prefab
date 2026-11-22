# Research: Tera fit for whole-project generation

Ticket: #6 (child of map #1). Researched 2026-11-21.

## Answer

**Yes, Tera is a good fit, as long as we target Tera 2 (current release 2.4.0).** Tera renders one template to one string. It has no idea of a "project", so Prefab's Rust core has to own the walk over the file tree: which files to emit, where they go, and how many copies of each (for example one file per Component). Inside a single file, Tera 2 covers what we need: inheritance, includes, typed **components** (they replace macros), custom filters with typed arguments, `-` whitespace trimming, errors on undefined variables, and Rust-style error reports. It can load templates from embedded strings or from disk.

Four things to design around:

1. **Turn off auto-escaping.** Tera HTML-escapes `.html`, `.htm` and `.xml` files by default. A generated `pom.xml` would be corrupted.
2. **No case-conversion filters ship with Tera.** We register our own.
3. **Whitespace control is manual `-` trimming only.** There is no `trim_blocks` or `lstrip_blocks`.
4. **Tera 2 is a recent rewrite.** 2.0.0 came out 2026-06-26, and most material on the web describes v1.

## Version and maintenance status

- **Tera 2 is released.** 2.0.0 shipped 2026-06-26 after alpha.1 through alpha.9 (2026-02-03 to 2026-06-20). Later releases: 2.1.0 (2026-07-23), 2.1.1, 2.2.0, 2.3.0, and **2.4.0 on 2026-09-11, the current max stable**. [crates.io API `/api/v1/crates/tera`; CHANGELOG.md]
  - The CHANGELOG dates 2.1.0 as "2026-06-23", which is before 2.0.0. crates.io gives 2026-07-23, so the changelog entry is a typo.
- The last v1 release is 1.20.1 (2025-10-30). [CHANGELOG.md]
- v2 is "a rewrite from scratch". [MIGRATION.md]
- The repo `Keats/tera` is active and not archived: last push 2026-11-20, 10 open issues. It is mainly a single-maintainer project (Vincent Prouillet, who also maintains Zola). That is a bus-factor point but not a red flag. [GitHub API]
- The crate uses edition 2024, so it needs Rust 1.85 or newer. Its only default dependency is `serde`. [tera/Cargo.toml; docs "Getting started"]
- Optional features: `glob_fs` (disk loading via walkdir and globset), `preserve_order` (indexmap), `unicode`, and `fast`. [tera/Cargo.toml]
- **Ecosystem caveat:** blog posts, Stack Overflow answers and LLM training data mostly describe v1. v1 syntax (macros, `include ignore missing`, `my_vec.0`, `as_str`, and others) no longer works. Template authors and agents should work from the v2 docs and MIGRATION.md only.

## Template inheritance, includes, macros/components

- **Inheritance** works like Jinja2: `{% extends %}` must come first, blocks can be overridden, `{{ super() }}` renders the parent block, and inheritance can go several levels deep with nested blocks. [docs "Inheritance"]
  - Since 2.2.0, content in a child template that would be silently ignored is an error. 2.2.0 also allows blocks inside if/elif/else again. [CHANGELOG 2.2.0]
  - Fit: a per-Technology base such as `java/Base.java.tera` with blocks for imports and body, refined by version-specific children.
- **Include:** `{% include "x" %}` renders with the current context.
  - **The path must be a static string.** `"partials/" ~ name` is rejected.
  - You **cannot pass a custom context** to an include. Use components for that.
  - `set` inside an included template does not leak back to the caller.
  - `include ignore missing` was removed in v2.
  - [docs "Include"; MIGRATION.md]
  - **Consequence:** a template cannot pick a Technology-specific partial from data. That choice belongs in Rust, or in components, or in `set_fallback_prefixes` (see Loading).
- **Macros are gone in v2**, replaced by **components**. [MIGRATION.md "Macros are gone"]
  - Components are registered globally, with no imports, and can be namespaced like `java.field`.
  - Parameters are optionally typed (`string`, `bool`, `integer`, `float`, `number`, `array`, `map`) and can have literal defaults.
  - A component is closed by default: an unknown argument is an error unless the component declares `...rest`.
  - Components are hermetic, so they see only their arguments. The exception is `@implicit` parameters (added in 2.4.0), which are looked up in the caller's context, then the render context, then the global context, then the default.
  - A component's body is passed in as `body`.
  - Call syntax is JSX-like: `{{ <java.field name="id" type="Long" /> }}`, or block form `{% <x> %}…{% </x> %}`.
  - Components can be rendered directly from Rust with `Tera::render_component`.
  - [docs "Components"; CHANGELOG 2.4.0]
  - **Typed, closed parameters suit a generator well**: a misspelled argument fails when the templates load, not as silent wrong output.
- **Checks at load time:** "Tera also now checks at compile-time that all functions/tests/filters/components are present and errors otherwise. Make sure to register everything before adding the templates." [MIGRATION.md] This means a whole template pack can be validated at startup or in a test.

## Custom filters/functions (case conversion)

- **There are no built-in case filters** beyond `lower`, `upper`, `capitalize` and `title`. [docs "Built-in filters"] `tera-contrib` doesn't add any either; its features are base64, date, filesize_format, format, json, rand, regex, slug and urlencode. [tera-contrib/README.md]
- **Registering a filter is simple**: `tera.register_filter("snake", |s: &str, _: Kwargs, _: &State| s.to_snake_case())`.
  - The first argument is typed, so a value of the wrong type is reported automatically.
  - Keyword arguments come from `kwargs.must_get::<T>("name")?`.
  - Filters, tests and functions can all read the context through `State`.
  - [docs "Custom filters, tests and functions"; tera.rs `register_filter`]
- **Bounds:** filters, tests and functions must be `Fn(...) + Send + Sync + 'static`. [filters.rs `trait Filter`] Plain `fn`s and closures that capture owned data satisfy this, so it fits our owned-values style without any `Arc` of our own. (Tera wraps them in `Arc` internally.)
- **Results** are any type that converts to `Value`, so returning an owned `String` works. [MIGRATION.md]
- **Prefab needs:** `snake_case` (Ruby files and methods, SQL), `PascalCase` (Java and Ruby classes), `camelCase` (Java members), `SCREAMING_SNAKE` (constants), `kebab-case` (artifact IDs, Kafka topics), and probably pluralize/singularize. Tera has a `pluralize` filter, but it only chooses a suffix from a count. It is not an inflector. [docs "pluralize"]

## Whitespace control (YAML, Ruby)

- **The only mechanism is `{%-`, `-%}`, `{{-`, `-}}`, `{#-` and `-#}`.** Each one removes *all* whitespace, newlines included, on that side of the tag. [docs "Whitespace control"]
  - The lexer's `whitespace_filter` simply calls `trim_start` or `trim_end` on the neighbouring content. [lexer.rs `whitespace_filter`]
  - There are **no `trim_blocks` or `lstrip_blocks` options** like Jinja2 has. Nothing in tera.rs, lexer.rs or the docs mentions them.
- **Effect on YAML and Ruby:**
  - `{%- if x %}` placed on an indented line also eats the indentation and the newline before it.
  - Where indentation matters, put control tags at column 0 and trim on the right only (`-%}`), or build the block and indent it with the `indent` filter.
  - `indent` takes `width` (default 4), an `indentation` character, a `first` flag and a `blank` flag. [docs "indent"; filters.rs `indent`]
- **Mitigation:** add golden-file snapshot tests for each template and a parse check on the generated YAML or Ruby. The Tera repo itself uses `insta` for snapshot tests. [tera/Cargo.toml dev-deps]

## Generating file paths and names from data

- Tera has no file-tree feature; that part is ours. Options:
  - **(a) A per-Technology manifest written as data.** It lists `{template, path_pattern, for_each}`, and each `path_pattern` is rendered with `Tera::render_str(pattern, &ctx, false)`. [tera.rs `render_str`: "cannot use `{% extends %}` and therefore blocks"; error names refer to `__tera_one_off`]
  - **(b) Templated filenames on disk**, for example `src/{{ module | snake }}.rb.tera`. These are fragile: `|` and `"` are not allowed in Windows filenames, and names are hard to read. A placeholder convention such as `__module__` would need our own substitution code.
  - Fan-out such as "one Java file per Component" is a loop in Rust over the spec. Each iteration renders the same template with a different context. Option (a) keeps that fan-out declared as data, which matches the map's rule that "a new version means data and templates, not Rust code".
- **Recommendation (option, not decision):** (a). It also gives Regeneration a single list of the paths Prefab owns.
- **`Tera::get_template_variables(name)`** returns a best-effort set of the top-level variables a template uses. [tera.rs] It could feed a "spec is missing data X" lint. It returns borrowed `&str` values, so we would clone them into owned `String`s.

## Loading: embedded vs disk

- **Embedded:**
  - `tera.add_raw_templates(vec![(name, include_str!(...)), …])` with `name: AsRef<str>` and `content: AsRef<str>`, so owned `String`s work.
  - If a batch fails, it rolls back to the previous set of templates.
  - Templates are validated together, so inheritance has to be complete within what is loaded.
  - [docs "Adding templates"; tera.rs `add_raw_templates`]
- **Disk:**
  - `add_template_file(s)` works without extra features.
  - `load_from_glob("templates/**/*")` needs the `glob_fs` feature; it strips the glob prefix from template names and supports `full_reload()`.
  - [tera.rs; docs]
- **Layering / overrides:**
  - Adding a template under an existing name replaces it. That allows "embedded defaults, then a user's local template dir loaded on top" with no new API.
  - `set_fallback_prefixes(["java25/spring42/", "java25/", "common/"])` makes `render`, `extends` and `include` try the exact name first and then each prefix in order. It must be called before any template is added. [tera.rs `set_fallback_prefixes`]
  - Fallback prefixes are a good fit for version-layered template packs across the Compatibility Matrix, and they partly make up for static-only includes.
- `Tera` is `Clone`. `register_from(&other)` copies filters, functions and tests between instances. [tera.rs] One option is a configured base instance cloned once per Stack.
- **Other settings:**
  - **Turn off auto-escape:** `tera.autoescape_on(vec![])`. The default list is `.html`, `.htm`, `.xml`. [tera.rs field doc; docs "Auto-escaping"]
  - `set_delimiters` is available if a target format conflicts with `{{ }}`, `{% %}` or `{# #}` (for example GitHub Actions `${{ }}`). It must be set before templates are added. Otherwise use `{% raw %}`. [tera.rs `set_delimiters`; docs "Raw"]
  - `global_context()` holds values shared across a render, such as the Prefab version or a generated-code header. [docs "Rendering a template"]

## Error reporting

- **v2 uses rustc-style reports:** a message, then `--> template:line:col` with a caret span, then `note: called from …` frames for includes. [MIGRATION.md "Better error messages"; errors.rs `ReportError`, `generate_report`]
- **Error kinds** are structured: `SyntaxError`, `RenderingError` (both carry a span report), `CircularExtend`, `CircularInclude`, `MissingParent`, `TemplateNotFound`, `ComponentNotFound`, `InvalidArgument`, `MissingArgument`, `OutOfRangeArgument`, `Io`, `Utf8Conversion`, `Msg`. They are matchable via `Error::kind()`, which helps map them to MCP and CLI diagnostics. [errors.rs]
- **Undefined values are strict:**
  - `{{ missing }}` errors, and so does `{{ existing.missing }}`.
  - Only one level of undefined is tolerated in `if`.
  - `a?.b?.c or "default"` handles optional chains.
  - [MIGRATION.md "Behaviour changes"; docs "Optional chaining"]
  - This is the right default for a generator: a missing spec field fails loudly instead of rendering empty.
- Unknown filters, functions, tests and components are reported when templates are added, not when they render. [MIGRATION.md]

## Companion crates it pushes us toward (options for maintainer sign-off)

None is required. Under the crate policy, each need below is listed with candidates and a stdlib-only option.

**Case conversion** (needed):
- `heck` 0.5.0: snake, lower and upper camel, Pascal, kebab, shouty snake, train and title case. Acronyms stay together (`XMLHttpRequest` splits as `XML|Http|Request`). The docs page lists no dependencies. Last release 2024-03. Very widely used. [docs.rs/heck; crates.io]
- `convert_case` 0.12.0: released 2026-08 and more configurable (custom boundaries). [crates.io]
- **Stdlib-only:** about 50 lines for word splitting plus joining. We control the acronym rules (Java `HttpClient` vs `HTTPClient`), but we also own the edge cases.

**Embedding a template directory** (only if we embed a whole tree rather than listing files):
- `include_dir` 0.7.4: `include_dir!("$CARGO_MANIFEST_DIR/templates")`, with `get_file` and `contents_utf8`. Glob support is an optional feature. Change tracking in the compiler needs the `nightly` feature. Last release 2024-06. [docs.rs/include_dir; crates.io]
- `rust-embed` 8.14.0: actively released (2026-11). It reads from disk in debug builds and embeds in release builds. [crates.io]
- **Stdlib-only:** a `build.rs` that walks `templates/` and writes a generated `&[(&str, &str)]` of `include_str!`s (plus `cargo:rerun-if-changed`). This needs no dependency and works with `add_raw_templates`.

**Inflection** (pluralize and singularize for Rails conventions, if wanted): no candidate researched yet; see questions below.

**Other optional features:**
- `tera-contrib` 0.3.0: probably not needed. Only `regex_replace` or `json_encode` might be useful. [tera-contrib/README.md]
- `insta` for golden-file tests is a dev-dependency only.
- Tera's own `glob_fs` feature pulls in walkdir and globset. It is a Tera feature, not a new crate in our manifest, but it does add those dependencies.

## Newly surfaced questions

- **Rails inflection:** Rails conventions (`users` table, `User` model) need pluralize and singularize. Should Prefab use an inflector crate, a stdlib table, or require the Universal Description Spec to state plural names explicitly?
- **Path manifest:** declarative per-Technology file manifest vs templated filenames. This touches Regeneration and Owned Paths (the manifest would be the list of owned files).
- **Template layering:** use `set_fallback_prefixes` to layer Technology-version templates (e.g. `java25/spring42/` over `java25/`), or keep one Tera instance per Stack?
- **Output checks:** should generated YAML and Ruby be parsed or validated after rendering to catch whitespace bugs? That probably needs a YAML parser crate, which would require sign-off.

## Sources

- Tera docs (v2): https://keats.github.io/tera/docs/ (source: https://github.com/Keats/tera/blob/master/docs/content/_index.md)
- Migration guide v1 to v2: https://github.com/Keats/tera/blob/master/MIGRATION.md
- Changelog: https://github.com/Keats/tera/blob/master/CHANGELOG.md
- Source: https://github.com/Keats/tera/blob/master/tera/src/tera.rs, `errors.rs`, `filters.rs`, `parsing/lexer.rs`, `tera/Cargo.toml`
- tera-contrib: https://github.com/Keats/tera/tree/master/tera-contrib
- crates.io API: https://crates.io/api/v1/crates/tera (and heck, convert_case, include_dir, rust-embed, tera-contrib)
- docs.rs: https://docs.rs/tera, https://docs.rs/heck/0.5.0/heck/, https://docs.rs/include_dir/0.7.4/include_dir/
