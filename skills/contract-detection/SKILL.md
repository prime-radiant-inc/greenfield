---
name: contract-detection
description: Layer 1 skill for parsing machine-readable API contracts. OpenAPI/Swagger, GraphQL, Protobuf/gRPC, and JSON Schema detection, extraction, and behavioral claim generation. Loaded by the analyzer agent during Layer 1.
---

# Contract Detection Methodology

Extract behavioral intelligence from machine-readable API contracts. These are formal, published definitions of system interfaces -- the strongest possible specification source. A contract file is an explicit promise about what the system accepts and returns.

## When to Use This Mode

Contract detection activates when:
- The target repository or documentation contains API specification files
- The discovery inventory identifies machine-readable contract files
- Other modes discover machine-readable contracts during analysis

This mode runs independently of all other intelligence sources. All output is **PUBLIC** -- machine-readable contracts are published definitions intended for external consumption. Output goes to `workspace/public/contracts/`.

## Why Contracts Are the Strongest Source

Machine-readable contracts are unique among intelligence sources because they are:
- **Formal** -- they use standardized schemas with unambiguous semantics
- **Published** -- they are intended for external consumers to rely on
- **Machine-verifiable** -- they can be validated against implementations automatically
- **Versioned** -- they explicitly track breaking changes through version fields

A single OpenAPI specification can contain more behavioral intelligence than the entire official documentation site, because every endpoint, parameter, response schema, and error code is defined with machine precision.

## OpenAPI / Swagger

### Detection

```bash
# Find OpenAPI/Swagger files
find . -maxdepth 5 -type f \( \
  -name "openapi.*" -o -name "swagger.*" -o \
  -name "api-spec.*" -o -name "api-docs.*" \
  \) \( -name "*.json" -o -name "*.yaml" -o -name "*.yml" \) 2>/dev/null

# Check for OpenAPI version markers in YAML/JSON files
grep -rl '"openapi":\|openapi:' --include="*.json" --include="*.yaml" --include="*.yml" . 2>/dev/null | head -20
grep -rl '"swagger":\|swagger:' --include="*.json" --include="*.yaml" --include="*.yml" . 2>/dev/null | head -20

# Check for hosted spec endpoints (common locations)
# /api-docs, /swagger.json, /openapi.json, /v2/api-docs, /v3/api-docs
```

### Extraction

For each OpenAPI/Swagger specification, extract:

#### Endpoints

| Field | What It Tells You |
|-------|-------------------|
| `paths` | Every endpoint the API exposes |
| HTTP method | The operation type (GET=read, POST=create, PUT=replace, PATCH=update, DELETE=remove) |
| `operationId` | The canonical name for the operation |
| `summary` / `description` | Behavioral description of what the endpoint does |
| `tags` | Logical grouping of endpoints |

#### Parameters

| Field | What It Tells You |
|-------|-------------------|
| `parameters` (path, query, header, cookie) | Required inputs and their types |
| `required` | Whether the parameter is mandatory |
| `schema` with `enum` | Allowed values (behavioral constraint) |
| `schema` with `minimum` / `maximum` | Value range (behavioral constraint) |
| `schema` with `pattern` | Validation regex (behavioral constraint) |
| `schema` with `default` | Default value when omitted |

#### Request/Response Schemas

| Field | What It Tells You |
|-------|-------------------|
| `requestBody` | What the endpoint accepts (content type, schema) |
| `responses` | Every possible response code and its schema |
| `responses.4xx` | Client error conditions and their structure |
| `responses.5xx` | Server error conditions |
| `components/schemas` | Shared data models with field types, constraints, and relationships |

#### Authentication

| Field | What It Tells You |
|-------|-------------------|
| `securityDefinitions` / `components/securitySchemes` | Auth methods (API key, OAuth2, Bearer, Basic) |
| `security` (global or per-operation) | Which endpoints require which auth |

### Output Format

Write to `workspace/public/contracts/openapi-summary.md`:

```markdown
## API: {title} v{version}

### Endpoints

| Method | Path | Operation | Auth Required | Description |
|--------|------|-----------|---------------|-------------|
| GET | /users | listUsers | Bearer | List all users with pagination |
| POST | /users | createUser | Bearer | Create a new user |

### Data Models

#### User
| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| id | string (uuid) | yes | read-only | Unique identifier |
| email | string | yes | format: email | User's email address |

### Error Responses
| Code | Meaning | Schema |
|------|---------|--------|
| 400 | Validation error | { message: string, errors: [{field, code}] } |
| 401 | Unauthorized | { message: string } |
| 404 | Not found | { message: string } |
```

## GraphQL

### Detection

```bash
# Find GraphQL schema files
find . -maxdepth 5 -type f \( \
  -name "schema.graphql" -o -name "*.graphqls" -o -name "schema.gql" -o \
  -name "*.graphql" \
  \) 2>/dev/null

# Find GraphQL codegen config (indicates GraphQL usage)
find . -maxdepth 3 -type f \( \
  -name "codegen.*" -o -name ".graphqlrc*" -o -name "apollo.config.*" \
  \) 2>/dev/null

# Check for GraphQL in dependencies
grep -l "graphql\|apollo\|@graphql" package.json requirements.txt Gemfile go.mod 2>/dev/null

# Check for introspection endpoint (if running instance available)
# POST /graphql with { "query": "{ __schema { types { name } } }" }
```

### Extraction

For each GraphQL schema, extract:

#### Queries (Read Operations)

| Element | What It Tells You |
|---------|-------------------|
| Query type fields | Every read operation the API exposes |
| Arguments | Required and optional parameters with types |
| Return types | Shape of the response data |
| Directives (`@deprecated`, `@auth`) | Behavioral modifiers |

#### Mutations (Write Operations)

| Element | What It Tells You |
|---------|-------------------|
| Mutation type fields | Every write operation the API exposes |
| Input types | Shape of the data the operation accepts |
| Return types | What the operation returns after modification |
| Error handling patterns | Union types for success/error returns |

#### Subscriptions (Real-Time Operations)

| Element | What It Tells You |
|---------|-------------------|
| Subscription type fields | Events the client can subscribe to |
| Arguments | Subscription filters |
| Payload types | Shape of the real-time data |

#### Type System

| Element | What It Tells You |
|---------|-------------------|
| Object types | Data entities and their fields |
| Enum types | Allowed values for categorical fields |
| Interface types | Shared behavioral contracts across types |
| Union types | Polymorphic response shapes |
| Input types | Structured input shapes for mutations |
| Custom scalars | Domain-specific value types (DateTime, JSON, URL) |

### Output Format

Write to `workspace/public/contracts/graphql-summary.md`:

```markdown
## GraphQL Schema

### Queries
| Query | Arguments | Returns | Description |
|-------|-----------|---------|-------------|
| users | filter: UserFilter, page: Int | [User!]! | List users with filtering |
| user | id: ID! | User | Get user by ID |

### Mutations
| Mutation | Input | Returns | Description |
|----------|-------|---------|-------------|
| createUser | input: CreateUserInput! | User! | Create a new user |
| deleteUser | id: ID! | Boolean! | Delete user by ID |

### Subscriptions
| Subscription | Arguments | Payload | Description |
|-------------|-----------|---------|-------------|
| userCreated | — | User! | Fires when a new user is created |

### Types
#### User
| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| id | ID | no | Unique identifier |
| email | String | no | Email address |
| role | UserRole | no | Enum: ADMIN, USER, VIEWER |
```

## Protobuf / gRPC

### Detection

```bash
# Find .proto files
find . -maxdepth 5 -type f -name "*.proto" 2>/dev/null

# Check for gRPC in dependencies
grep -l "grpc\|protobuf" package.json requirements.txt Gemfile go.mod Cargo.toml 2>/dev/null

# Find generated gRPC code
find . -maxdepth 5 -type f \( -name "*_grpc.pb.go" -o -name "*_pb2_grpc.py" -o -name "*_grpc.rb" \) 2>/dev/null
```

### Extraction

For each `.proto` file, extract:

#### Services

| Element | What It Tells You |
|---------|-------------------|
| `service` definitions | Logical groupings of RPC methods |
| `rpc` methods | Every callable operation |
| Request/response types | Input and output shapes |
| Streaming modifiers | `stream` on request, response, or both |

#### Messages

| Element | What It Tells You |
|---------|-------------------|
| `message` definitions | Data structures used in requests and responses |
| Field types and numbers | Schema with wire-format compatibility rules |
| `repeated` fields | Array/list fields |
| `oneof` fields | Union types (exactly one field set) |
| `optional` / `required` | Field presence requirements |
| `map` fields | Key-value pair fields |

#### Enums

| Element | What It Tells You |
|---------|-------------------|
| `enum` definitions | Allowed categorical values |
| `option allow_alias` | Whether multiple names map to the same value |

#### Options

| Element | What It Tells You |
|---------|-------------------|
| `option java_package` | Target language packaging |
| `option go_package` | Go module path |
| Custom options | Domain-specific metadata |

### Output Format

Write to `workspace/public/contracts/protobuf-summary.md`:

```markdown
## Protobuf/gRPC Services

### Service: UserService
| RPC | Request | Response | Streaming | Description |
|-----|---------|----------|-----------|-------------|
| GetUser | GetUserRequest | User | none | Get user by ID |
| ListUsers | ListUsersRequest | ListUsersResponse | none | List users with pagination |
| WatchUsers | WatchUsersRequest | User | server-stream | Stream user updates |

### Messages
#### User
| Field | Number | Type | Label | Description |
|-------|--------|------|-------|-------------|
| id | 1 | string | — | Unique identifier |
| email | 2 | string | — | Email address |
| role | 3 | UserRole | — | User role |

### Enums
#### UserRole
| Name | Number |
|------|--------|
| USER_ROLE_UNSPECIFIED | 0 |
| USER_ROLE_ADMIN | 1 |
| USER_ROLE_USER | 2 |
```

## JSON Schema

### Detection

```bash
# Find JSON Schema files
find . -maxdepth 5 -type f -name "*.schema.json" 2>/dev/null
find . -maxdepth 5 -type f -name "*.json" -exec grep -l '"$schema"' {} \; 2>/dev/null | head -20

# Check for JSON Schema in config files
grep -rl '"$schema":\|"\$ref"' --include="*.json" . 2>/dev/null | head -20

# Find JSON Schema in OpenAPI components (already covered above, but note cross-reference)
```

### Extraction

For each JSON Schema, extract:

#### Type Definitions

| Element | What It Tells You |
|---------|-------------------|
| `type` | The data type (object, array, string, number, boolean, null) |
| `properties` | Named fields with their own schemas |
| `required` | Fields that must be present |
| `additionalProperties` | Whether unknown fields are allowed |

#### Validation Rules

| Element | What It Tells You |
|---------|-------------------|
| `minLength` / `maxLength` | String length constraints |
| `minimum` / `maximum` | Numeric range constraints |
| `pattern` | Regex validation for strings |
| `format` | Semantic format (email, uri, date-time, uuid) |
| `enum` | Allowed values |
| `const` | Fixed required value |
| `minItems` / `maxItems` | Array length constraints |
| `uniqueItems` | Whether array elements must be unique |

#### Composition

| Element | What It Tells You |
|---------|-------------------|
| `$ref` | References to shared schema definitions |
| `allOf` | Schema intersection (all must match) |
| `anyOf` | Schema union (at least one must match) |
| `oneOf` | Schema exclusive union (exactly one must match) |
| `not` | Schema negation |
| `if` / `then` / `else` | Conditional validation |

#### Defaults and Examples

| Element | What It Tells You |
|---------|-------------------|
| `default` | Default value when field is omitted |
| `examples` | Example values (behavioral documentation) |
| `description` | Human-readable behavioral description |

### Output Format

Write to `workspace/public/contracts/json-schema-summary.md`:

```markdown
## JSON Schema: {title}

### Type: CreateUserRequest
| Property | Type | Required | Constraints | Default | Description |
|----------|------|----------|-------------|---------|-------------|
| email | string | yes | format: email | — | User email address |
| name | string | yes | minLength: 1, maxLength: 100 | — | Display name |
| role | string | no | enum: [admin, user, viewer] | "user" | User role |
| tags | array of string | no | maxItems: 10, uniqueItems: true | [] | User tags |

### Validation Rules
- `email` must match email format (RFC 5322)
- `name` must be 1-100 characters
- `role` defaults to "user" when omitted
- `tags` must contain unique strings, maximum 10
```

## Cross-Contract Correlation

When multiple contract types are present (e.g., both OpenAPI and GraphQL, or OpenAPI with JSON Schema references), correlate them:

- Do the endpoint definitions agree on field names and types?
- Are the same data models defined consistently across contracts?
- Do version numbers align?
- Are there endpoints in one contract that are missing from another?

Document correlations and discrepancies in `workspace/public/contracts/cross-contract-notes.md`.

## Provenance Rules

### Source Type

All claims from contract detection use `source=machine-readable-contract`:

```markdown
- The /users endpoint accepts a `role` query parameter with values: admin, user, viewer
  <!-- cite: source=machine-readable-contract, ref=openapi.yaml:/paths/~1users/get/parameters/0, confidence=confirmed, agent=contract-detector -->
```

### Confidence Levels

- **confirmed** -- the behavioral claim is explicitly defined in the contract schema (endpoint exists, field has this type, parameter has this constraint). Machine-readable contracts are formal specifications; their explicit definitions are confirmed by definition.
- **inferred** -- the behavioral claim is derived from contract structure rather than explicit definition (e.g., "this API follows REST conventions" inferred from path patterns)
- **assumed** -- the behavioral claim extrapolates beyond what the contract defines (e.g., "this endpoint probably supports pagination" because similar endpoints do)

### Cite As You Go

Every behavioral claim gets an inline citation immediately after the claim. The `ref` field should be `<file-path>:<json-path-or-line>`.

## Output Structure

```
workspace/public/contracts/
    openapi-summary.md             # OpenAPI/Swagger extraction
    graphql-summary.md             # GraphQL schema extraction
    protobuf-summary.md            # Protobuf/gRPC extraction
    json-schema-summary.md         # JSON Schema extraction
    cross-contract-notes.md        # Correlations and discrepancies across contract types
    raw/                           # Copies of discovered contract files for reference
        openapi.yaml
        schema.graphql
        service.proto
        config.schema.json
```

Note: Output goes to `workspace/public/contracts/` -- not `workspace/raw/`. Machine-readable contracts are published definitions intended for external consumption. They contain no proprietary implementation details.

## Rules

1. **Contracts are PUBLIC** -- machine-readable contracts are published interface definitions. Output goes to `workspace/public/`, not `workspace/raw/`.
2. **Exhaustive extraction** -- extract EVERY endpoint, field, constraint, and error code. Contracts are finite and complete. Do not sample.
3. **Preserve precision** -- contracts define exact types, constraints, and enums. Do not paraphrase "minLength: 1, maxLength: 100" as "limited length." Preserve the exact constraints.
4. **Note versions** -- contracts have version fields. Always record the version. Behavioral claims from contracts are version-specific.
5. **Detect staleness** -- a contract file may be outdated relative to the implementation. Flag contracts that appear unmaintained (old modification dates, version mismatches with the codebase).
6. **Copy originals** -- place copies of discovered contract files in `workspace/public/contracts/raw/` for downstream reference.
7. **Cite as you go** -- every behavioral claim gets an inline `<!-- cite: -->` comment immediately after the claim. Never defer citation to a later step.
