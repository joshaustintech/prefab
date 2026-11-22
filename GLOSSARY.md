# Prefab

Prefab generates backend software from a language-neutral description, the way a prefabricated house is assembled from standard parts. The same description can be regenerated against different tech stacks or versions.

## Language

**Universal Description Spec**:
The language-neutral document that describes what an app is, independent of any language, framework, or datastore.
_Avoid_: Manifest, config, blueprint

### Structure

**System**:
A set of Services and the connections between them, such as which Service publishes an event and which Services consume it. Optional: a spec may describe a single Service.
_Avoid_: Platform, landscape, cluster

**Service**:
One complete, runnable backend application described by the Universal Description Spec, such as a microservice or a modular monolith.
_Avoid_: App, project, microservice (as the general term)

**Module**:
A bounded grouping of Components inside one Service, the unit of separation in a modular monolith. Optional: a Service may hold Components directly.
_Avoid_: Package, bounded context, subsystem, Component

**Component**:
A prefabricated part of a Service, such as an entity with its persistence, an HTTP endpoint, an event producer, or an event consumer.
_Avoid_: Module, part, piece, block

**Feature**:
An optional cross-cutting capability that can be switched on per Service or per Component: logging, tracing, unit tests, or smoke tests.
_Avoid_: Add-on, plugin, extra

### Stacks

**Stack**:
The set of versioned Technologies a Service is generated against, one per role: language, framework, datastore, and optionally messaging.
_Avoid_: Tech stack, platform, target

**Technology**:
One versioned choice filling one role in a Stack, such as Java 25, Rails 8.2, Postgres, or Kafka.
_Avoid_: Dependency, tool, backend

**Compatibility Matrix**:
The declared record of which Technology versions may be combined in one Stack, such as which Rails versions run on which Ruby versions.
_Avoid_: Support table, version map

### Regeneration

**Regeneration**:
Producing a Service's code again from its Universal Description Spec, typically against a changed Stack or version, without destroying user-written code.
_Avoid_: Rebuild, re-scaffold

**Generated Code**:
Code Prefab owns and may overwrite on every Regeneration.
_Avoid_: Boilerplate, scaffold

**Override**:
User-written code placed in a location Prefab designates for it, which Prefab never overwrites and which customizes or replaces Generated Code.
_Avoid_: Patch, customization, extension

**Owned Path**:
A folder the user has marked as theirs, which Regeneration leaves untouched even if the result no longer compiles or runs.
_Avoid_: Ignored path, locked folder, protected region
