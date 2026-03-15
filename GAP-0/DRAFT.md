# GraphQL @mock Directive Specification

This specification defines the `@mock` directive to allow GraphQL clients to
return mocked data for fields or entire operations.

Mock data may be defined for fields and types that do not yet exist in the
schema. This enables backend and client developers to work in parallel — client
developers can start building applications using expected new fields without
waiting for the server to implement the new schema.


```graphql example
query GetBusinessInfo {
  business(id: "123") {
    name
    # this field doesn't exist yet on the server!
    website @mock(value: "https://www.example.com")
  }
}
```

`@mock` can also be used to mock subtrees and whole operations:

```graphql example
# mock just the 'hours' field
query GetBusinessHours {
  business(id: "123") {
    name
    hours @mock(variant: "morning-only") {
      open
      close
    }
  }
}

# mock the entire operation
query GetBusinessRating @mock(variant: "five-star-bakery") {
  business(variant: "123") {
    name
    rating
  }
}
```


Mock data is stored in JSON files and read statically when requests are
executed:

```json example
{
  "morning-only": {
    "data": {
      "open": "8:00am",
      "close": "12:00pm"
    },
    "__appliesTo__": "Business.hours"
  }
}
```

The client transforms the document to remove mocked
_[selections](https://spec.graphql.org/September2025/#Selection)_ before
executing or sending the request to the server. Upon receiving a response from
the server, mock values are merged into the response object before yielding to
the application.

Use of Large Language Models (LLMs) is suggested as a means of creating and
maintaining mock data.

# Directive

```graphql
directive @mock(
  variant: String
  value: String
) on QUERY | MUTATION | SUBSCRIPTION | FIELD
```

## Arguments

`@mock` accepts two mutually exclusive arguments: {"variant"} and {"value"}.
Exactly one of these arguments must be provided. Providing both {"variant"} and
{"value"} in the same `@mock` application is a validation error.

### variant

{"variant"} maps to a *mock variant id*. This uniquely identifies a
*mock value* (stored in the *mock file*) to return for the field or operation
where `@mock` is applied.

### value

{"value"} provides an inline *mock value* directly in the operation document.
When {"value"} is provided, the client uses the supplied string as the mock
response for the annotated field without consulting a *mock file*.

{"value"} may only be applied to fields that resolve to
_[leaf types](https://spec.graphql.org/September2025/#sec-Leaf-Field-Selections)_
(scalars and enums). It must not be applied to fields that return object types
or to operation roots. The client must coerce the string value to the field's
expected scalar type (for example, `"42"` becomes the integer {42} for an
{Int} field, and `"true"` becomes the boolean {true} for a {Boolean} field).

```graphql example
query GetBusinessInfo {
  business(id: "123") {
    name @mock(value: "The Great British Bakery")
    rating @mock(value: "4.5")
    isOpen @mock(value: "true")
    hours @mock(variant: "morning-only") {
      open
      close
    }
  }
}
```

Note: {"value"} is useful for quick, self-contained mocks of scalar fields
where the overhead of creating and maintaining a *mock file* entry is not
warranted.

## Returning Mock Data

If `@mock` is applied to the operation's root field (e.g. {"query"}), the entire
response must be resolved from a *mock file*. The {"value"} argument must not be
used on operation roots.

If `@mock` is applied to non-root fields only, the client must transform the
document to remove any selections which have `@mock` applied before sending the
request to the server. For fields using {"variant"}, mock values are resolved
from a *mock file*. For fields using {"value"}, the mock value is the coerced
inline string. When the server's response is received, the client merges each
*mock value* into the response before yielding to the application.

When `@mock` is applied to a field, the *mock value* must always be included in
the response, regardless of other directives present on the same field. In
particular, `@mock` takes precedence over `@skip` and `@include` — the mock
data is always returned, irrespective of whether the field would otherwise be
skipped or included based on those directives' conditions.

Mock values must **not** be generated dynamically at runtime. Mock values must
be resolved from the *mock file* or from the inline {"value"} argument.

The mechanism for GraphQL clients running in a web browser or mobile client to
read the *mock file* is implementation defined. See:
[Appendix: Reading Mock Files](#sec-Reading-Mock-Files).

# Mock Files

Each operation or fragment that contains one or more `@mock` directives using
the {"variant"} argument must have an associated *mock file*. Operations or
fragments where all `@mock` directives use only the {"value"} argument do not
require a *mock file*.

:: A *mock file* is a JSON file which maps *mock variant id* keys to a
*mock variant*.

:: The *executable target* of a *mock file* is defined to be the name of an
[ExecutableDefinition](https://spec.graphql.org/September2025/#ExecutableDefinition)
(that is, an operation or fragment) in which the mock values may be used.

Note: Mock files are intended to be long-lived and may checked into version
control. This is useful for client developers working on a project over an
extended period of time, and where the client code depends on GraphQL schema
that does not yet exist.

## Mock File Location

The *mock file* must be named `{Name}.json`, where `{Name}` is the name of its 
*executable target*.

TODO: Support multiple files per target - instead of {Name}.json, it could
be a directory of mock files, nested by the operation target - i.e.
{Name}/arbitrary.json

Mock files should be stored in a `__graphql_mocks__` directory adjacent to the
source file containing the *operation target*.

**Example**

For an operation named `GetBusinessInfo` defined in `BusinessDetails.graphql`:

```
.
├── __graphql_mocks__
│   └── GetBusinessInfo.json
└── BusinessDetails.graphql
```

## Mock File Structure

The mock file contains a JSON object which maps *mock variant id* keys to a
*mock variant*.

:: A *mock variant id* is any key in the object that does not start with two
underscores (`__`).

:: A *mock variant* is the object associated with each *mock variant id*.

### Mock Variant Object

A *mock variant* object may contain **only** the following keys:

- {"data"} (required)
- {"errors"}
- {"extensions"}
- {"__appliesTo__"} (required)
- {"__description__"}
- {"__metadata__"}

#### data

:: {"data"} stores the *mock value*. It may be `null` if the field is nullable.

#### errors

{"errors"} may contain an array of error objects conforming to the
[GraphQL error format](https://spec.graphql.org/September2025/#sec-Errors).
This must be merged into the GraphQL operation's response if defined.

TODO: Define merging algorithm and resolution algorithm for conflicts

#### extensions

{"extensions"} may be a key-value mapping of arbitrary data. This must be merged
into the GraphQL operation's response if defined.

TODO: Define merging algorithm and resolution algorithm for conflicts

**___description__**

{"__description__"} may contain a string which describes the *mock value* in
natural language. This value should be used when regenerating the *mock value*.

```json example
{
  "5-star-business: {
    "data": {
      "business": {
        "name": "The Great British Bakery",
        "rating": 5.0
      }
    },
    "__appliesTo__": "Query",
    "__description__": "A delicious bakery with a rating of 5.0"
  },
}
```

#### __appliesTo__

{"__appliesTo__"} must be defined as the resolved
_[schema coordinate](https://spec.graphql.org/September2025/#sec-Schema-Coordinates)_
of the field or root operation type where `@mock` is applied, or may be applied,
with the *mock variant id*.

**Example**

For the following fragment:

```graphql example
fragment FooFields on Foo {
  # location resolves to "Foo.bar"
  bar @mock(variant: "basic-bar")

  # location resolves to "Foo.baz"
  baz @mock(variant: "basic-baz") {
    qux
  }
}
```

This would be a (minimally) vald corresponding *mock file*:

```json example
{
  "basic-bar": {
    "data": "...",
    "__appliesTo": "Foo.bar"
  },
  "basic-baz": {
    "data": { "qux": "..." },
    "__appliesTo": "Foo.baz"
  },
}
```

#### __metadata__

{"__metadata__"} may be a key-value mapping for additional user or application
defined metadata.

## Mock Entry Validation

### Query Changes

As development progresses, the shape of a query may change — fields may be added,
removed, or renamed. When this happens, an existing *mock value* may no longer
match the expected shape. Conforming clients must detect when a *mock value* is
incompatible with its operation and force corrective action.

Mocks must be validated as part of the application test suite.

Note: It is possible to detect if a JSON payload is valid for a given operation
by constructing an in-memory GraphQL server that has no resolvers, and uses the
JSON payload as its {rootValue} - and ensuring no errors are thrown for
execution of the operation against the test server.

GraphQL clients should warn or throw for an invalid *mock value*. Implementers
must detect this, and similarly force corrective action (e.g. by forcing the
user to regenerate or fix the *mock value* in the *mock file*).

### Missing Mock Name

If the requested *mock variant* does not exist in the associated *mock file*
for the containing operation or fragment, the client must raise an error
indicating the available mock ids.

### Invalid Mock Value

If a *mock value* does not conform to the expected shape, client behavior is
implementation-defined. Clients should validate mock values and provide helpful
error messages during development.

### Inline Value Validation

If {"value"} is applied to a field that does not resolve to a leaf type (scalar
or enum), the client must raise an error.

If {"value"} is applied to an operation root, the client must raise an error.

If the {"value"} string cannot be coerced to the field's expected scalar type,
the client must raise an error.

If both {"variant"} and {"value"} are provided in the same `@mock` application,
the client must raise an error.

# Mock File Generation

The mechanism for generating and maintaining mock files is
implementation-defined.

However, this specification is designed to work well with LLM-based coding
agents. Developers using agents can request new mock variants conversationally.

It is recommended that implementers provide a workflow for developers to
generate contextually appropriate mock data based on field names, types,
and surrounding context.

For example, a developer may add a mock with the following prompt:

```example
"add a mock for GetBusinessInfo where the rating field returns an error"
```

**Non-Normative: Suggested Agent Skill**

Implementers of this specification may provide an
[Agent Skill](https://agentskills.io/home) conforming to the Agent Skills
Specification. This allows coding agents to discover and use mock management
capabilities.

The following is a suggested `SKILL.md`. Implementers are welcome to replace or
adapt this prompt to suit their implementation:

[SKILL.md](./SKILL.md)

# Appendix

_This section is non-normative and used for clarification only._

## Dynamic Mock Variant IDs

In theory, mock variant IDs may be derived from operation
_[variables](https://spec.graphql.org/September2025/#sec-Language.Variables)_.
This is neither explicitly disallowed or encouraged.

A use case would be to build an interface to let viewers control which
*mock value* is used. For example, the following GET parameters:

```example
GET /my_page?gql_mock_variants=barMock=some-variant-id;bazMock=some-other-variant-id
```

Could be used as inputs to the following operation:

```graphql example
query Foo($barMock: String, $bazMock: String) {
  bar @mock(variant: $barMock)
  baz @mock(variant: $bazMock)
}
```

Clients must manually coerce the directive argument variables (similar to
[CoerceArgumentValues](https://spec.graphql.org/draft/#CoerceArgumentValues()).

This is currently outside the scope of this specification.

## Reading Mock Files

GraphQL Clients that implement this schema must have access to the data
contained in mock files when requests are executed. Since web browsers and
mobile applications cannot read remote filesystems, a mechanism is required
to make the data contained in mock files available to the client.

This mechanism is implementation defined.

The reference implementation for web serializes the contents of all mock files
into a single JSON object, and is embedded on the webpage in a `<script>` HTML
tag.

## Schema-Aware Clients

Schema-aware clients face additional complexity: operations using the `@mock`
directive may reference types and fields not yet present in the server schema.
Such clients must patch their local schema to include these definitions, or
disable validation for mocked operations. The mechanism for schema patching is
outside the scope of this specification, however contributions addressing this
are welcome for inclusion in a future version of this document.

**Trusted Documents**

TODO: add a section to clarify that while this intended to be used in local dev
(where the documents don't need to be hashed), because mock files can get
checked in, and @mock directives could get checked in, @mock selections need
to not be included in the actual document that gets hashed (i.e. the same
transforming needs to happen at runtime and query compile time)
This could be useful in staging environments or on prod (behind a feature flag). 
