Taskcluster-lib-references is responsible for handling the API reference data,
including manifests, API references, exchange references, and JSON schemas.

It exports a class, `References`, that manages reading and writing this data in
several formats, as well as performing transformations and consistency checks.

It also serves as the canonical repository for a few shared schemas and
meta-schemas under `schemas/`.

# Data Formats

## Built Services

This input format corresponds to a directory structure built by
[taskcluster-builder](https://github.com/taskcluster/taskcluster-builder)
during the cluster build process:

```
├── myservice
│   ├── metadata.json
│   ├── references
│   │   ├── events.json
│   │   └── api.json
│   └── schemas
│       └── v1
│           ├── data-description.json
│           └── more-data.json
├── anotherservice
¦   ├── metadata.json
```

This is a subset of the
[taskcluster-lib-docs](https://github.com/taskcluster/taskcluster-lib-docs)
documentation tarball format, and `metadata.json` is defined there.  The
library will in fact load any `.json` files in `references` and interpret them
according to their schema.  So, the `$schema` property of every file in
`references` must point to the relevant schema using a `/`-relative URI such as
`/schemas/common/api-reference-v1.json`.

*NOTE* this library can only read data in this format, not write it.

## URI-Structured

This data structure looks like the `schemas/` and `references/` URIs within a
Taskcluster rootUrl.

```
├── references
│   ├── manifest.json
│   ├── auth
│   │   └── v1
│   │       ├── api.json
│   │       └── exchanges.json
│   └── ...
└── schemas
    ├── auth
    │   └── v1
    │       ├── authenticate-hawk-request.json
    │       └── ...
    ├── common
    │   ├── action-schema-v1.json
    │   ├── api-reference-v0.json
    │   ├── exchanges-reference-v0.json
    │   └── manifest-v2.json
    └── ...
```

## Serializable

The serializable format is similar to the URI-structured format, but as a
JSON-serializable data structure consisting of an array of `{filename,
content}` objects.

```json
[
  {
    "filename": "references/manifest.json",
    "content": {
      "$schema": ...
    }
  }, ...
]
```

# Abstract and Absolute References

A References instance can either be "abstract" or
"absolute". The abstract form contains `/`-relative URIs of the form
`/references/..` and `/schemas/..`, abstract from any specific rootUrl.  These
appear anywhere a link to another object in the references or schemas appears,
including `$schema`, `$id`, and `$ref`.

The absolute form contains URIs that incorporate a specific rootUrl and thus
could actually be fetched with an HTTP GET (in other words, the URIs are URLs).

The library can load and represnt either form, converting between them with
`asAbsolute` and `asAbstract`.  Some operations are only valid on one form.
For example, json-schema requires that `$schema` properties be absolute URLs,
so schema operations on schemas should not be performed on the abstract form.

# API

To create a References instance, use one of the following methods:

```js
const References = require('taskcluster-lib-references');

// Build from a built services format (which is always abstract)
references = References.fromBuiltServices({directory: '/build/directory'});

// Build from a uri-structured format; omit rootUrl when on-disk data is abstract
references = References.fromUriStructured({directory: '/app', rootUrl});

// Build from a serializable data structure; omit rootUrl if data is abstract
references = References.fromSerializable({serializable, rootUrl});
```

To validate the references, call `references.validate()`.
If validation fails, the thrown error will have a `problems` property containing the discovered problems.

To convert between abstract and absolute references, use

```js
const abstract = references.asAbstract();
const absolute = abstract.asAbsolute(rootUrl);
```

Note that both of these operations create a new object; Reference instances are
considered immutable.

To write data back out, use

```js
// write URI-structured data
references.writeUriStructured({directory});

// create a serializable data structure
data = references.makeSerializable();
```

To build an [Ajv](https://github.com/epoberezkin/ajv) instance containing all schemas and metaschemas, call `references.makeAjv()`.
This is only valid on an absolute References instance.
To skip validation, use `makeAjv({skipValidation: true})`.

To get a specific schema, call `references.getSchema($id)`, with an absolute or abstract `$id` as appropriate.
To skip validation, use `getSchema($id, {skipValidation: true})`.

# Validation

Validation occurs automatically in most operations on this class.

Every schema must:
* have a valid `$id` within a Taskcluster deployment (so, a relative path of `/schemas/<something>/<something>.json#`);
* have a valid `$schema`, either a json-schema.org URL or another schema in the references; and
* validate against the schema indicated by `$schema`.

All `$ref` links in schemas must be relative and must refer only to schemas in
the same service (so `v1/more-data.json` may refer to
`../util/cats.json#definitions/jellicle` but not to
`/schemas/anotherservice/v1/cats.json`). As an exception, refs may refer to
`http://json-schema.org` URIs.

Every reference must:
* have a valid `$schema`, defined in the references;
* have a schema that, in turn, hsa metaschema `/schemas/common/metadata-metaschema.json#`;
* validate against the schema indicated by `$schema`.

# File Contents

The contents of the files handled by this library are described in the schemas under `schemas/` in the source repository, or under `/schemas/common` in a Taskcluster deployment.

# Development

The usual `yarn` and `yarn test` will run tests for this library.
