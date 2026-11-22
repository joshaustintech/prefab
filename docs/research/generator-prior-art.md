# Prior art: spec-driven generators and safe Regeneration

Research for issue #5, a child of map #1. Researched 2026-11-21 against official docs and source repos. Each claim carries a link to its source. Where a primary source was silent, this note says so instead of filling the gap.

**Question:** How do existing spec-driven generators describe apps, and how do they handle Regeneration over user-edited code? What spec shapes, ownership and ignore markers, Override mechanisms, and state or manifest tracking do they use? What pitfalls do their users report?

## Summary

There are three families of approach:

1. **Never edit output; customize from the outside.** Smithy and the generation gap pattern take this line. Generated code is disposable and is regenerated wholesale. Customization lives in separate classes or in generator plugins.
2. **Overwrite with per-path opt-outs.** OpenAPI Generator, Rails, and Yeoman (with JHipster on top of it) take this line. The generator rewrites files. Users protect paths with an ignore file, with `--skip` or `--force`, or by answering a prompt for each file.
3. **Three-way merge from recorded state.** Copier, cruft, and `jhipster upgrade` take this line. The tool records the spec answers and the template version. It re-renders the old version and the new one, then replays the user's diff on top, and conflicts surface the way git conflicts do.

Prefab's model combines family 1 (Override folders) with family 2 (Owned Path dotfiles). The prior art strongly supports that split. It also points to two mechanics Prefab needs and has not yet specified: a **manifest of generated files**, so stale files can be deleted, and a **recorded generator and template version**.

## Per-tool findings

### JHipster (JDL)

- **Spec shape.** JDL describes "all your applications, deployments, entities and their relationships in a single file" ([JDL intro](https://www.jhipster.tech/jdl/intro)). An `application { config { baseName, applicationType, databaseType, prodDatabaseType, buildTool … } entities … dto * with mapstruct; service * with serviceClass }` block nests stack choices and per-entity options. Application-level options override global ones ([JDL applications](https://www.jhipster.tech/jdl/applications)). Entities are typed fields with validations, e.g. `name String required`, `age Integer min(42)` ([JDL entities](https://www.jhipster.tech/jdl/entities-fields)). The spec mixes *what* (entities) with *how* (stack choices and options), much as Prefab's Stack sits beside the Universal Description Spec.
- **State.** JHipster keeps the configuration of each entity as JSON in `.jhipster/`, and `.yo-rc.json` records `jhipsterVersion` ([creating an entity](https://www.jhipster.tech/creating-an-entity/), [upgrading](https://www.jhipster.tech/upgrading-an-application/)).
- **Regeneration.** Regenerating warns that "all your custom code will be overwritten". `jhipster jdl` regenerates only the entities that changed, and `--force` "will overwrite all your local changes" ([creating an entity](https://www.jhipster.tech/creating-an-entity/)). `jhipster upgrade` keeps an orphan `jhipster_upgrade` branch, regenerates there with `--force`, commits, and merges back into the user's branch, leaving conflicts to the user. The docs say "Don't commit anything on the `jhipster_upgrade` branch" ([upgrading](https://www.jhipster.tech/upgrading-an-application/)). This is a three-way merge that uses git as the engine.
- **Override mechanism.** The official "side-by-side" tip has users subclass the generated classes and win bean resolution with `@Primary`, mark generated repositories `@NoRepositoryBean`, and put custom endpoints under a separate prefix such as `/api/v2`. The tip lists two pros, "easy to find custom code" and overriding generated behavior, and one con, file duplication ([tip 035](https://www.jhipster.tech/tips/035_tip_combine_generation_and_custom_code.html)). The upgrade guide recommends side-by-side because it reduces merge conflicts ([upgrading](https://www.jhipster.tech/upgrading-an-application/)).
- **Pitfalls.** In [discussion #24655](https://github.com/jhipster/generator-jhipster/discussions/24655), users report that customization guidance is scattered and that "for large projects it could be hard to manually click yes yes no" through overwrite prompts. They also ask for "do not touch" markers. A maintainer agrees: "we lack a lot of documentation with regard to sbs customization".

### OpenAPI Generator

- **Spec shape.** The input is an OpenAPI document. Generator configuration lives in a separate config file or a `batch` YAML ([usage](https://openapi-generator.tech/docs/usage/)).
- **Ignore marker.** `.openapi-generator-ignore` has `.gitignore`-like syntax: `#` comments, `/`-anchored paths, `**`, and `!` negation. It "must exist in the root of the output directory". A negation cannot re-include a file under an excluded parent ([customization](https://openapi-generator.tech/docs/customization)). In the source, the file is created from a template only if it does not already exist, so the user owns it after the first generation. The exception is the `openapiGeneratorIgnoreList` option, which overwrites it every run ([DefaultGenerator.java](https://github.com/OpenAPITools/openapi-generator/blob/master/modules/openapi-generator/src/main/java/org/openapitools/codegen/DefaultGenerator.java)).
- **Other flags.** `--skip-overwrite` keeps existing files, `--minimal-update` writes only the files that changed, and `--dry-run` reports changes without making them ([usage](https://openapi-generator.tech/docs/usage/)).
- **Override mechanism.** Users override templates with a `templateDir` holding a file of the same relative name. A user template that matches a built-in supporting file replaces it. Otherwise it is added. Near-miss names produce "duplicated model docs" or "undefined behavior" ([customization](https://openapi-generator.tech/docs/customization)).
- **Manifest.** `.openapi-generator/FILES` lists every generated path, relative to the output directory and sorted. `.openapi-generator/VERSION` records the generator version. Both are written only when `generateMetadata` is on, and only on real runs, not dry runs ([DefaultGenerator.java](https://github.com/OpenAPITools/openapi-generator/blob/master/modules/openapi-generator/src/main/java/org/openapitools/codegen/DefaultGenerator.java)).
- **Pitfall: stale files.** If you rename a model and regenerate, the old file stays. The proposed fix is to persist the file list and delete the old entries on the next run. That proposal is still **open, filed 2019** ([issue #4076](https://github.com/OpenAPITools/openapi-generator/issues/4076)). In practice, users clear the output directory themselves or turn on the Gradle plugin's `cleanupOutput`. A manifest that nothing acts on does not solve the problem.

### Smithy

- **Spec shape.** The model is a protocol-agnostic graph of shapes (services, operations, resources, structures) annotated with traits. `smithy-build.json` defines **projections**: filtered or transformed views of one model, built with transforms such as `includeShapesByTag`. It also defines the **plugins** that run per projection. Output goes to `${outputDirectory}/${projection}/${artifactName}/`, alongside `model/model.json` and build info ([smithy-build.json](https://smithy.io/2.0/guides/smithy-build-json.html)).
- **Ownership.** The guide's principle: "Generated code is valid without the need to manually edit or further transform it" ([building codegen](https://smithy.io/2.0/guides/building-codegen/index.html)). Output is a build artifact, and users never edit it.
- **Override mechanism.** `SmithyIntegration` plugins are discovered via Java SPI and opted into by model traits or by settings. They can `preprocessModel`, decorate the symbol provider (names and file locations), add files in `customize`, and use **code interceptors** that "prepend to the contents, append to the contents, or completely rewrite" named sections the generator exposes. `runBefore`, `runAfter`, and `priority` order them ([making codegen pluggable](https://smithy.io/2.0/guides/building-codegen/making-codegen-pluggable.html)).
- **Lesson.** Projections are a clean analogue for "one spec, many Stacks". Named, interceptable sections are a template-level Override that needs no hand edits.

### Rails generators

- **Override mechanism.** Rails looks in the app's `lib/templates/` before the generator's own templates. `config.generators` swaps or configures the built-in generators ([generators guide](https://guides.rubyonrails.org/generators.html)).
- **Conflicts.** Generators take `--force`, `--skip`, and `--pretend` ([generators guide](https://guides.rubyonrails.org/generators.html)). Thor prompts on each colliding file with `[Ynaqdhm]`: yes, no, all, quit, diff, help, and merge tool ([thor shell/basic.rb](https://github.com/rails/thor/blob/main/lib/thor/shell/basic.rb)).
- **Regeneration.** Rails keeps no recorded state. `bin/rails app:update` re-runs the app generator interactively over config files. Instead of rewriting behavior in place, it emits `new_framework_defaults_X_Y.rb`, which users enable gradually before bumping `config.load_defaults` ([upgrading guide](https://guides.rubyonrails.org/upgrading_ruby_on_rails.html)). Generators are effectively one-shot scaffolds, and Rails relies on convention, not tracking.
- **Lesson.** Versioned defaults that users opt into (`load_defaults`) are a mechanism for a Stack version bump that does not depend on merging.

### Yeoman

- **State.** `.yo-rc.json` marks the project root and stores namespaced generator config, so sub-generators and re-runs can reuse the answers ([storage](https://yeoman.io/authoring/storage.html)).
- **Conflicts.** All writes are staged in a shared in-memory filesystem and committed to disk at the end. Every overwrite of existing content needs validation ([file system](https://yeoman.io/authoring/file-system.html)). The conflicter's choices are overwrite, skip, all, diff, abort, and the experimental reload, edit, and "ignore, do not overwrite and remember". Unchanged files count as `identical` and are skipped. It also supports `bail` (stop on the first conflict), `ignoreWhitespace`, and `dryRun` ([conflicter.ts](https://github.com/yeoman/yeoman-api/blob/main/workspaces/conflicter/src/conflicter.ts)).
- **Ownership marker.** Choosing "ignore and remember" appends `<path> skip` to a **`.yo-resolve`** file. The file holds minimatch patterns, each with an optional status that defaults to `skip`, and allows `!` negation. A `.yo-resolve` applies to its folder and every folder below it, and the nearest file wins ([yo-resolve.ts](https://github.com/yeoman/yeoman-api/blob/main/workspaces/conflicter/src/yo-resolve.ts)). This is very close to Prefab's Owned Path dotfile.

### cookiecutter and cruft

- cookiecutter has no update mechanism. Replay files in `~/.cookiecutter_replay/` hold past answers so you can generate *a new project* from an updated template ([replay](https://cookiecutter.readthedocs.io/en/stable/advanced/replay.html)). The answers live outside the project, so the project cannot be regenerated on its own.
- cruft adds update support. `.cruft.json` in the project records `template`, `commit`, and the variables. `cruft update` shows the changes and applies them on approval. `cruft check` exits 1 when the project is out of date, for use in CI. A `skip` glob list (in `.cruft.json` or `pyproject.toml`) excludes paths from updates ([cruft](https://cruft.github.io/cruft/)).

### Copier

- **State.** Copier writes `.copier-answers.yml` inside the project, holding the answers plus `_commit`, the template git ref. "Never update `.copier-answers.yml` manually" ([updating](https://copier.readthedocs.io/en/stable/updating/), [configuring](https://copier.readthedocs.io/en/stable/configuring/)). Template versions come from git tags, sorted by PEP 440, and the destination must be git-tracked ([configuring](https://copier.readthedocs.io/en/stable/configuring/)).
- **Algorithm** (from [`_main.py`](https://github.com/copier-org/copier/blob/master/copier/_main.py)):
  1. Render the *old* template version with the recorded answers into a temporary directory.
  2. Diff that old render against the user's tree (`git diff-tree`).
  3. Render the new version over the project.
  4. Re-apply the diff with `git apply --reject`.
  5. For each `.rej`, fall back to `git merge-file` with the old render as base. Files that still have conflicts are marked unmerged in the index.
  6. Delete files that existed only in the old render.

  Files the user deleted are *not* regenerated ("Don't regenerate intentionally deleted paths"). `_skip_if_exists` paths are created if missing but never overwritten, and their diffs are excluded.
- **Conflict modes.** `--conflict inline`, the default, writes git-style markers. `--conflict rej` writes `.rej` files. The docs recommend pre-commit hooks that catch leftover markers and `.rej` files ([updating](https://copier.readthedocs.io/en/stable/updating/)).
- **Other mechanisms.** `_exclude` matches destination paths. Versioned `_migrations` run only on update, between `_version_from` and `_version_to`. `copier recopy` is the escape hatch, and it "will override any changes found in your local project". `copier check-update` reports whether a newer template version exists ([configuring](https://copier.readthedocs.io/en/stable/configuring/), [updating](https://copier.readthedocs.io/en/stable/updating/)).
- **Pitfalls.** The docs warn that hand-editing the answers file makes the diff unpredictable, and that `git merge --abort` does not undo a conflicted update ([updating](https://copier.readthedocs.io/en/stable/updating/)). Credentials passed in the template URL end up stored in the answers file ([FAQ](https://copier.readthedocs.io/en/stable/faq/)).

### Generation gap pattern, and marker comments

- Fowler: "Separate generated code from non-generated code by inheritance." "Generated code should never be edited by hand, otherwise you can't safely regenerate it." He credits John Vlissides, whose version has the hand-written class subclass the generated one ([Fowler, Generation Gap](https://martinfowler.com/dslCatalog/generationGap.html)). JHipster's side-by-side tip is this pattern applied with Spring `@Primary`.
- For contrast, Eclipse EMF/JET marks generated members `@generated`. Users "deface" the tag (`@generated NOT`) to claim a member, and JMerge keeps it on regeneration ([JET FAQ: JMerge](https://wiki.eclipse.org/JET_FAQ_What_is_JMerge%3F)). Forum reports note that user regions are lost when the enclosing generated member disappears from the input. Protected regions inside a file are brittle, which is why `GLOSSARY.md` avoids "protected region".

## Lessons for Prefab

1. **Keep the Generated Code / Override boundary at the file or folder level, never inside a file.** Smithy, Fowler, and JHipster side-by-side all converge on this. In-file markers (EMF) and prompts for each file (Rails, Yeoman) are the reported pain points. For Java, this means generation-gap subclasses or `@Primary` beans in Override folders. For Rails, it means reopened classes, concerns, or subclasses loaded from Override folders. *(Feeds #7 and #10.)*
2. **Write a manifest.** Record every generated path, plus a content hash (an addition to what OpenAPI Generator stores) so that hand edits to Generated Code can be detected. Write it to a Prefab-owned dotdir, like `.openapi-generator/FILES`. *Use it to delete stale files.* OpenAPI Generator writes a manifest but has never acted on it, and issue #4076 has been open since 2019.
3. **Record the inputs needed to reproduce the last generation.** Store the Prefab version, the template and Compatibility Matrix version, and the resolved Stack, like Copier's `_commit` and JHipster's `jhipsterVersion`. A three-way merge needs the old render, and reproducing that render needs these inputs.
4. **Owned Path dotfile semantics.** Model them on `.yo-resolve` and `.openapi-generator-ignore`: a dotfile in a folder covers that folder and everything below it, the nearest file wins, and its glob patterns support `!` negation. Note OpenAPI Generator's trap that a negation cannot re-include a file under an excluded parent. Prefab must never overwrite the marker itself.
5. **Treat user deletions as intent.** Copier does not regenerate paths the user deleted. Prefab should decide whether deleting a Generated Code file is an implicit Owned Path or an error.
6. **Pick a conflict policy, and make it non-interactive for MCP.** Prompting per file (Ynaqdh) does not work for an MCP front end. Copier's inline and `.rej` modes, Yeoman's `bail`, and `--dry-run` / `--pretend` all translate into a non-interactive report.
7. **Spec shape.** JDL puts the stack inside the application block. Smithy keeps the model protocol-agnostic and puts build targets (projections and plugins) in a separate file. Smithy's split fits the Universal Description Spec better: a language-neutral spec, with the Stack in a separate section or file. Copier's `_migrations` and Rails' `load_defaults` suggest how spec-schema and Stack version bumps can migrate state.
8. **Extension points in templates.** Smithy's named sections with prepend, append, and replace interceptors suggest how template packages could expose extension points without hand edits. *(Feeds the template-packaging ticket.)*

## Open questions surfaced

- Does Prefab do a three-way merge (the Copier model, which requires git and the old render) or plain overwrite with Override folders and Owned Paths? Plain overwrite is simpler, and the glossary implies it.
- What happens when a user hand-edits a Generated Code file outside any Override folder or Owned Path? Options are to warn using the manifest hash, refuse to run, or overwrite.
- Should a deleted generated file be recreated, or treated as owned?
- Must the project be in a git repository? Copier and `jhipster upgrade` require one.
- How do Override hooks differ by Technology? Java has subclasses and `@Primary` beans. Rails has reopening, concerns, and Zeitwerk load order.
