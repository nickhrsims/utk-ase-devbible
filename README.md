# Dungeon Master AI

## Version Control

### Standards & Practices

Please consider using the following:

- [Semantic Versioning](https://semver.org/)
- [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
- [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)

- For Python: [PEP8](https://peps.python.org/pep-0008/)
- For JavaScript: TypeScript and clear best practices

### Structuring Remote Repositories and Projects

I _strongly_ advise that the end-user application, server, and optionally the
CDM (see below) all exist as **separate repositories**. These are all logically
independent projects, that should have independent version control. I argue
that grouping all work into a single repository is **problematic** for
dependency management, division of labor, **possible loss of work due to
version control access collisions by teams**, and may **introduce bugs** for a
number of reasons. There is a reason that other notable open source projects
isolate components of a project into independent repositories. We should model
best practices here as well.

## System Boundaries and Independent System Identification

From the project description, here is a possible interpretation of independent
system boundaries.

### Primary Systems

- Resource Server (`uvicorn` Server running `FastAPI` app, with fine-grained
  behavioral model)

- End-User Application (Web Client w/ `react` or `svelte`, with coarse-grain
  behavioral model API)

### Supporting Systems

- AI Model (Public LLM? This area is not my expertise)

- `mongodb` (Data Persistence)

- Possible Supplementary Systems

  - EBook Scraper (Possible Option: `python` + `poppler` + Collection of
    D&D Ebooks) Only if data sets cannot be found that provide better results

  ```text
  note: I happen to have a vast collection of D&D ebooks for reference up to
  v3.5; about 10GiB worth
  ```

### System Automation Artifacts

- Structured AI Support Content (scraping results)

## Foreword: On The Core Data Model

Hereafter called the CDM.

### Explanation

I vote that the CDM should be the first thing we work on as a group, with
complete participation, before dividing labor into other territories.

```text
   All these communication points require a common understanding of how
   to structure the data to be sent.

   This means that efforts to divide labor will result in blocking work
   until questions are answered regarding the shape of communication
   structures.

   Therefore, I believe this is the first thing to be done: the CDM.

        [ELASTICSEARCH]
              ^
              |
              v
[AI] ----> [SERVER] <---> [CLIENT]
              ^
              |
              v
           [MONGODB]

```

With an established model, work can feasibly be done by separate specialized
sub-groups.

### Single Source of Truth

The CDM should begin with a SSOT (Single Source of Truth).

I vote that this source be the Resource Server itself as `pydantic` models. This
allows describing both the shape of the data, as well as the **behavioral
semantics** supported by the CDM.

In this way, the primary consumer of the model (the client app) may take
advantage of either of the technologies:

- OpenAPI v3 JSON Specifications
- JsonScheme v2020-12

`pydantic` supports generating artifacts of both formats from defined models.

In this way, the schemas themselves may be build-time resources on an HTTP API.

JavaScript may use these directly, with runtime validation support via.

- `ajv` (with native support, but weak TypeScript support natively)
- `yup` (with native TypeScript support, but requires 3rd party libs to parse
  JSON schema)

### Life as a library

Later in this document, I advise we pretend that the CDM is a library. Here, I posit the option that it _truly exist as an independent library_.

## Supporting Systems: MongoDB, Eslasticsearch, and Deployment

For the supporting systems, it is STRONGLY advised that ALL of our code support
dependency injection to remove these as runtime dependencies for the majority
of development.

When we need to run these for integration testing, please consider using
docker. I intend to run these containers locally on my system and can assist
team members with configuring these services. I can, on request, write
`Dockerfile`s for the client and server, as well as a `docker-compose.yml` to
aggregate the entire application package. This allows for future work such as
CI/CD simplification, and a better UX for self-hosted target users. It may also
earn us points on the assignment.

## The Resource Server

- `resource_server/src/resource_server/*`
  - `/core` (model layer)
  - `/data` (data layer)
  - `/resources` (resource layer)
  - `/app.py` (application layer)

### On the Nature of RESTful APIs

I vote we put sincere effort into providing a resource based, stateless HTTP
API; i.e. a RESTful HTTP API. We will likely incur slight deviations, which is
fine when necessary, but the primary objective should be to view the server
this way.

### Module Organization & Architectural Advisory

I vote the Resource Server be structured around an architectural pattern _similar_ to MVC, following similar principles.

The specific architectural design I propose describes the organization of
intra-project dependencies.

#### A Note on Names

Concepts named after "jobs" or "roles" are often a sign of weak
design, and lead to ambiguity in team-comprehension. (i.e. words that end in
"-ER" are conceptually flimsy and difficult to communicate). "Roles" describe
was something _does_, but it can be hard to help describe what it _is_.

The layers described here will resemble MVC loosely, but names will not
exclusively be named this way, nor will modules be organized naively around
names like "controller".

#### Model Layer

The CDM is implemented here directly, and should be treated like an independent
library in isolation. This layer (the CDM) should not communicate with other
layers, not be concerned with files, schema conversion, AI, networking/routing,
database storage, or ANYTHING else unrelated to JUST Dungeons & Dragons logical
representation.

The CDM should probably define (at minimum) the following Aggregate Roots
(top-level entities):

- Dungeons & Dragons NPC

- Dungeons & Dragons Story Prompt / Campaign Specification

- Dungeons & Dragons "Flavour" Configuration Object
  (an object to hold settings, such as genre, tone, complexity, etc.)

Modules organized under this layer are free to follow a flat, non-hierarchical
view, so long as the inter-module dependencies are well documented (and can be
described as a unidirectional tree). **TL;DR**: This layer can be organized
however, just **don't write spaghetti code**.

#### Data Layer

Here, a generic [interface](https://docs.python.org/3/library/abc.html)
modelling a
[repository](https://www.martinfowler.com/eaaCatalog/repository.html) should be
defined.

Specific repositories for all aggregate roots would then be implemented in a
flat organization scheme.

Below, I strongly suggest option 1, it will allow for the writing of `null`
mappers to be written. In this way, we can support DI and remove MongoDB as a
testing requirement for other code.

##### On Inter-repository dependency

Here, we have three options:

1. A generic interface for a [data
   mapper](https://www.martinfowler.com/eaaCatalog/dataMapper.html). Where each
   entity type requiring independent storage have an associated mapper.
   Repositories requiring duplicate logic depend on shared mappers.

2. Let repositories depend on one another.

3. Duplicate access code where necessary.

The project is pretty small. It can likely handle some duplication, but these
are reasonable considerations.

#### Resource Layer

This layer provides abstraction over access to conceptual "resources" from the
perspective of the RESTful HTTP API.

Each sub-directory here represents a _version_ of the API. Each API version
must have a grouping
[router](https://fastapi.tiangolo.com/reference/apirouter/) with a prefix
configured to match the version.

Additional sub-routers should be used to group resources under versions.

See [here](https://fastapi.tiangolo.com/tutorial/bigger-applications/)

Here, any [DTOs](https://martinfowler.com/eaaCatalog/dataTransferObject.html),
utilities, and all endpoint-functions would be defined. Support for DI would be
managed here as well.

#### Application Layer

The application layer is truly just a single file instantiating the app.

Here, a specific version-based API router would be imported and registered, and
dependencies would be initialized and configured for passing to the app from
here.

### Project Definition, Tooling, and Dependency Management

The following is advised

- Use `poetry` for managing the python project and dependencies.
- Use `black` for formatting all python modules.
- Use `ruff` and `pyright` for static analysis. These may be configured with
  most editors to run as [LSP
  implementations](https://en.wikipedia.org/wiki/Language_Server_Protocol). I can
  assist with setting this up in editors like `neovim`, and somewhat with
  `emacs`. VSCode should have native support for these tools.

### Testing Strategy

The following is advised

- Use `pytest` as core testing framework.
- Use `TestClient` for FastAPI route-level integration testing.
- Follow test driven development principles.

I believe we should focus on writing tests first for all logic within the
server. These tests should serve as an informal "specification" for what the
code looks like at an interface level.

From the tests, keep writing implementation until the tests are passing.

This strategy may not be reasonable for all portions of code, and 100% coverage
is not feasible with a project on such tight deadlines, but we should strive to
cover at least any code that one might consider _clever_ (though anything
_clever_ should generally be avoided anyway unless it can't).

## Web Application
