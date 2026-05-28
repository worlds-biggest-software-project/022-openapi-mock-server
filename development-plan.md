# OpenAPI Mock Server — Development Plan

> Project #022 · Plan Created: 2026-05-25

---

## Technology Decisions

### Runtime & Language: TypeScript on Node.js

**Rationale:** The primary competitors (Prism, Mockoon CLI) are Node.js-based. The target audience (frontend developers, QA engineers, API testers) overwhelmingly has Node.js in their toolchain. TypeScript provides type safety for the complex OpenAPI schema handling and enables a single codebase to serve as both a CLI tool and an embeddable npm package.

```
Runtime:       Node.js >= 20 (LTS)
Language:      TypeScript 5.5+
Package mgr:   pnpm (workspaces for monorepo)
Build:         tsup (fast ESM/CJS dual builds)
```

### Data Model: Hybrid Relational + JSONB (Suggestion 2, adapted for in-memory)

**Rationale:** Data Model Suggestion 2 (Hybrid JSONB) best fits the MVP target. The CLI/single-process deployment model does not need a 29-table normalized schema (Suggestion 1) or event-sourcing infrastructure (Suggestion 3). However, the in-memory runtime model mirrors Suggestion 2's structure: the parsed spec is stored as a single resolved JSON document, an operation index provides fast request matching, and session state is a flexible JSON object.

For the MVP, all state is in-memory. Persistence (SQLite or PostgreSQL) is introduced in Phase 7 for multi-instance and hosted deployments.

```
MVP storage:   In-memory (Map/object stores)
v1.1 storage:  SQLite (optional file-based persistence)
v2.0 storage:  PostgreSQL with JSONB (hosted/multi-tenant)
```

### HTTP Framework: Fastify

**Rationale:** Fastify outperforms Express by 2-3x on throughput benchmarks, has built-in JSON Schema validation (directly aligned with the OpenAPI domain), supports plugins for CORS/logging/lifecycle hooks, and is TypeScript-first.

```json
{
  "dependencies": {
    "fastify": "^5.x",
    "@fastify/cors": "^10.x",
    "@fastify/swagger": "^9.x"
  }
}
```

### OpenAPI Parsing: @apidevtools/swagger-parser + oas-normalize

**Rationale:** `swagger-parser` handles $ref resolution, validation, and supports OpenAPI 2.x/3.0/3.1. `oas-normalize` normalizes different spec formats (YAML/JSON, 2.x/3.x) into a canonical form before parsing.

```json
{
  "dependencies": {
    "@apidevtools/swagger-parser": "^10.x",
    "oas-normalize": "^11.x",
    "openapi-types": "^12.x"
  }
}
```

### AI Data Generation: Anthropic Claude API (with fallback to local heuristics)

**Rationale:** Claude provides the best structured-output fidelity for generating semantically realistic data from schema descriptions. The architecture uses a tiered approach: (1) heuristic field-name matching for common patterns (email, phone, name, address), (2) cached AI generation for complex schemas, (3) random Faker.js fallback when no API key is configured.

```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "^0.39.x",
    "@faker-js/faker": "^9.x"
  }
}
```

### Testing: Vitest + Supertest

**Rationale:** Vitest is the fastest TypeScript-native test runner, supports ESM natively, and integrates with the pnpm monorepo. Supertest provides HTTP assertion capabilities for integration testing the mock server.

```json
{
  "devDependencies": {
    "vitest": "^3.x",
    "supertest": "^7.x",
    "@vitest/coverage-v8": "^3.x"
  }
}
```

### CLI Framework: Commander.js

**Rationale:** Lightweight, zero-dependency CLI framework. Prism uses a custom CLI; Commander provides the same DX with less maintenance burden.

```json
{
  "dependencies": {
    "commander": "^13.x",
    "chalk": "^5.x",
    "ora": "^8.x"
  }
}
```

---

## Project Structure

```
openapi-mock-server/
├── packages/
│   ├── core/                          # Shared kernel: spec parsing, schema resolution, data generation
│   │   ├── src/
│   │   │   ├── parser/
│   │   │   │   ├── spec-loader.ts          # Load + normalize OpenAPI specs (YAML/JSON, 2.x/3.x)
│   │   │   │   ├── ref-resolver.ts         # Resolve all $ref pointers into inline schemas
│   │   │   │   ├── operation-indexer.ts     # Build operation index from parsed spec
│   │   │   │   └── spec-validator.ts        # Validate spec completeness and correctness
│   │   │   ├── generator/
│   │   │   │   ├── data-generator.ts        # Orchestrator: heuristic → AI → faker pipeline
│   │   │   │   ├── heuristic-generator.ts   # Field-name-aware pattern matching (email, phone, etc.)
│   │   │   │   ├── ai-generator.ts          # Claude API integration for complex schemas
│   │   │   │   ├── faker-generator.ts       # Faker.js fallback for basic type generation
│   │   │   │   ├── schema-walker.ts         # Recursive schema traversal (allOf/oneOf/anyOf)
│   │   │   │   └── generation-cache.ts      # LRU cache keyed by schema hash
│   │   │   ├── validator/
│   │   │   │   ├── request-validator.ts     # Validate inbound requests against spec
│   │   │   │   └── response-validator.ts    # Validate generated responses against spec
│   │   │   ├── errors/
│   │   │   │   ├── rfc7807.ts               # RFC 7807 Problem Details builder
│   │   │   │   └── owasp-templates.ts       # OWASP API Top 10 error templates
│   │   │   ├── types/
│   │   │   │   ├── openapi.ts               # Extended OpenAPI type definitions
│   │   │   │   ├── operation-index.ts       # OperationIndex type and interfaces
│   │   │   │   ├── config.ts                # Configuration schema types
│   │   │   │   └── generation.ts            # Data generation pipeline types
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   ├── server/                        # HTTP mock server (Fastify-based)
│   │   ├── src/
│   │   │   ├── server.ts                    # Fastify instance factory
│   │   │   ├── router.ts                    # Dynamic route registration from operation index
│   │   │   ├── request-matcher.ts           # Match incoming requests to operations
│   │   │   ├── response-builder.ts          # Build mock response from matched operation
│   │   │   ├── middleware/
│   │   │   │   ├── cors.ts                  # CORS configuration
│   │   │   │   ├── request-logger.ts        # Request/response logging middleware
│   │   │   │   ├── latency-simulator.ts     # Artificial latency injection
│   │   │   │   └── error-rate.ts            # Random error injection
│   │   │   ├── session/
│   │   │   │   ├── session-store.ts         # In-memory session state management
│   │   │   │   ├── scenario-engine.ts       # State machine execution for scenarios
│   │   │   │   └── state-mutator.ts         # Apply state mutations on transitions
│   │   │   ├── admin/
│   │   │   │   ├── admin-routes.ts          # Admin API for runtime configuration
│   │   │   │   ├── log-viewer.ts            # Request log query endpoint
│   │   │   │   └── health.ts                # Health check endpoint
│   │   │   ├── proxy/
│   │   │   │   ├── proxy-handler.ts         # Forward requests to upstream API
│   │   │   │   └── drift-detector.ts        # Compare upstream response to spec
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   └── cli/                           # CLI entry point
│       ├── src/
│       │   ├── cli.ts                       # Commander program definition
│       │   ├── commands/
│       │   │   ├── mock.ts                  # `mock` command: start mock server
│       │   │   ├── validate.ts              # `validate` command: validate spec
│       │   │   ├── generate.ts              # `generate` command: generate sample data
│       │   │   └── proxy.ts                 # `proxy` command: start in proxy mode
│       │   ├── output/
│       │   │   ├── formatter.ts             # Console output formatting
│       │   │   └── table.ts                 # Tabular display for operation listing
│       │   └── index.ts
│       ├── bin/
│       │   └── openapi-mock.ts              # Shebang entry point
│       ├── package.json
│       └── tsconfig.json
│
├── specs/                             # Sample OpenAPI specs for testing and demos
│   ├── petstore-3.1.yaml
│   ├── petstore-3.0.json
│   ├── petstore-2.0.yaml
│   ├── complex-ecommerce.yaml
│   └── minimal.yaml
│
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
│
├── .github/
│   └── workflows/
│       ├── ci.yml                           # Lint, test, build on PR
│       └── release.yml                      # Publish to npm on tag
│
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── vitest.config.ts
├── .eslintrc.cjs
├── .prettierrc
├── LICENSE                                  # MIT
└── README.md
```

---

## Phase 1: Project Scaffolding & Spec Parsing

**Goal:** Set up monorepo, parse OpenAPI 2.x/3.0/3.1 specs, and build an operation index.

### What

- Initialize pnpm monorepo with `core`, `server`, and `cli` packages
- Implement spec loader that accepts file path or URL, detects format (YAML/JSON), and normalizes to OpenAPI 3.x
- Resolve all `$ref` pointers into inline schemas using `swagger-parser`
- Build an in-memory operation index: for each path+method, store the resolved operation definition, compiled path regex, extracted path parameters, and available response codes
- Implement CLI skeleton with `mock --spec <path>` command that loads and displays the parsed operation index

### Design

**Spec Loader** — the entry point for all spec processing:

```typescript
// packages/core/src/parser/spec-loader.ts
import SwaggerParser from '@apidevtools/swagger-parser';
import OASNormalize from 'oas-normalize';
import { OpenAPIV3_1 } from 'openapi-types';

export interface LoadedSpec {
  raw: string;
  format: 'yaml' | 'json';
  openapiVersion: string;
  parsed: OpenAPIV3_1.Document;
  checksum: string;
}

export async function loadSpec(source: string): Promise<LoadedSpec> {
  // Normalize: convert Swagger 2.0 → OAS 3.x, resolve format differences
  const normalizer = new OASNormalize(source);
  const normalized = await normalizer.validate(); // throws on invalid spec

  // Dereference: resolve all $ref pointers into inline definitions
  const dereferenced = (await SwaggerParser.dereference(
    normalized as any
  )) as OpenAPIV3_1.Document;

  const raw = await normalizer.stringify();
  const checksum = createHash('sha256').update(raw).digest('hex');

  return {
    raw,
    format: source.endsWith('.json') ? 'json' : 'yaml',
    openapiVersion: dereferenced.openapi ?? '2.0',
    parsed: dereferenced,
    checksum,
  };
}
```

**Operation Indexer** — builds the fast-lookup index from a parsed spec:

```typescript
// packages/core/src/parser/operation-indexer.ts
import { OpenAPIV3_1 } from 'openapi-types';

export interface IndexedOperation {
  pathPattern: string;       // '/users/{userId}/orders'
  httpMethod: string;        // 'GET'
  operationId?: string;
  summary?: string;
  pathRegex: RegExp;         // /^\/users\/([^/]+)\/orders$/
  pathParams: string[];      // ['userId']
  hasRequestBody: boolean;
  responseCodes: string[];   // ['200', '404']
  operationDef: OpenAPIV3_1.OperationObject;
}

export function buildOperationIndex(
  spec: OpenAPIV3_1.Document
): IndexedOperation[] {
  const index: IndexedOperation[] = [];
  const paths = spec.paths ?? {};

  for (const [pathPattern, pathItem] of Object.entries(paths)) {
    if (!pathItem) continue;

    const methods = ['get','post','put','patch','delete','head','options','trace'] as const;
    for (const method of methods) {
      const operation = pathItem[method];
      if (!operation) continue;

      const pathParams = [...pathPattern.matchAll(/\{(\w+)\}/g)].map(m => m[1]);
      const regexStr = pathPattern.replace(/\{(\w+)\}/g, '([^/]+)');
      const pathRegex = new RegExp(`^${regexStr}$`);

      index.push({
        pathPattern,
        httpMethod: method.toUpperCase(),
        operationId: operation.operationId,
        summary: operation.summary,
        pathRegex,
        pathParams,
        hasRequestBody: !!operation.requestBody,
        responseCodes: Object.keys(operation.responses ?? {}),
        operationDef: operation,
      });
    }
  }

  // Sort by specificity: longer paths first, then literal segments over parameterized
  return index.sort((a, b) => {
    const aLiterals = a.pathPattern.split('/').filter(s => !s.startsWith('{')).length;
    const bLiterals = b.pathPattern.split('/').filter(s => !s.startsWith('{')).length;
    if (b.pathPattern.length !== a.pathPattern.length) {
      return b.pathPattern.length - a.pathPattern.length;
    }
    return bLiterals - aLiterals;
  });
}
```

**CLI skeleton:**

```typescript
// packages/cli/src/commands/mock.ts
import { Command } from 'commander';
import { loadSpec, buildOperationIndex } from '@openapi-mock/core';
import chalk from 'chalk';

export const mockCommand = new Command('mock')
  .description('Start a mock server from an OpenAPI spec')
  .requiredOption('-s, --spec <path>', 'Path to OpenAPI spec file')
  .option('-p, --port <number>', 'Port to listen on', '4010')
  .option('-d, --dynamic', 'Enable dynamic AI data generation', false)
  .option('--no-validate', 'Disable request validation')
  .action(async (options) => {
    const spec = await loadSpec(options.spec);
    const index = buildOperationIndex(spec.parsed);

    console.log(chalk.green(`Loaded ${spec.openapiVersion} spec: ${spec.parsed.info.title}`));
    console.log(chalk.dim(`  ${index.length} operations indexed`));
    console.log();

    for (const op of index) {
      const method = op.httpMethod.padEnd(7);
      console.log(`  ${chalk.cyan(method)} ${op.pathPattern}`);
    }

    // Server start happens in Phase 2
  });
```

### Testing

- **Unit: spec-loader** — Load petstore 3.1 YAML, petstore 3.0 JSON, and petstore 2.0 YAML; verify `parsed` has correct structure, `openapiVersion` is extracted, `checksum` is deterministic
- **Unit: ref-resolver** — Load spec with deep `$ref` chains (A → B → C), circular `$ref`s, and cross-file `$ref`s; verify all are resolved inline
- **Unit: operation-indexer** — Build index from petstore spec; verify correct count of operations, path regex matches real URLs, path params are extracted, sort order puts specific routes before parameterized
- **Integration: CLI mock command** — Run `openapi-mock mock --spec petstore.yaml` and verify it prints the operation list without errors
- **Edge cases:** Empty paths object, spec with only webhooks (no paths), spec with servers containing variables

---

## Phase 2: Basic HTTP Mock Server

**Goal:** Start a Fastify server that responds to requests with structurally valid mock data using Faker.js.

### What

- Create Fastify server factory that registers routes dynamically from the operation index
- Implement request matcher that maps incoming HTTP requests to indexed operations using path regex and method
- Build response builder that generates structurally valid JSON from the operation's response schema using Faker.js
- Implement recursive schema walker that handles `object`, `array`, `string`, `number`, `integer`, `boolean`, enums, and format-specific values (date-time, email, uri, uuid)
- Add CORS middleware enabled by default
- Add request logging middleware that records method, path, matched operation, and response status

### Design

**Schema Walker** — recursively generates data from any JSON Schema node:

```typescript
// packages/core/src/generator/schema-walker.ts
import { OpenAPIV3_1 } from 'openapi-types';
import { faker } from '@faker-js/faker';

export function walkSchema(
  schema: OpenAPIV3_1.SchemaObject,
  depth: number = 0,
  maxDepth: number = 5
): unknown {
  if (depth > maxDepth) return null;

  // Use spec example if available
  if (schema.example !== undefined) return schema.example;

  // Handle enum
  if (schema.enum && schema.enum.length > 0) {
    return faker.helpers.arrayElement(schema.enum);
  }

  // Handle composition
  if (schema.oneOf) {
    const chosen = faker.helpers.arrayElement(schema.oneOf) as OpenAPIV3_1.SchemaObject;
    return walkSchema(chosen, depth + 1, maxDepth);
  }
  if (schema.anyOf) {
    const chosen = faker.helpers.arrayElement(schema.anyOf) as OpenAPIV3_1.SchemaObject;
    return walkSchema(chosen, depth + 1, maxDepth);
  }
  if (schema.allOf) {
    const merged: Record<string, unknown> = {};
    for (const sub of schema.allOf) {
      const result = walkSchema(sub as OpenAPIV3_1.SchemaObject, depth + 1, maxDepth);
      if (typeof result === 'object' && result !== null) {
        Object.assign(merged, result);
      }
    }
    return merged;
  }

  switch (schema.type) {
    case 'object':
      return generateObject(schema, depth, maxDepth);
    case 'array':
      return generateArray(schema, depth, maxDepth);
    case 'string':
      return generateString(schema);
    case 'number':
    case 'integer':
      return generateNumber(schema);
    case 'boolean':
      return faker.datatype.boolean();
    default:
      return null;
  }
}

function generateObject(
  schema: OpenAPIV3_1.SchemaObject,
  depth: number,
  maxDepth: number
): Record<string, unknown> {
  const result: Record<string, unknown> = {};
  const properties = schema.properties ?? {};
  const required = new Set(schema.required ?? []);

  for (const [name, propSchema] of Object.entries(properties)) {
    // Always generate required fields; generate optional fields 70% of the time
    if (required.has(name) || Math.random() < 0.7) {
      result[name] = walkSchema(
        propSchema as OpenAPIV3_1.SchemaObject,
        depth + 1,
        maxDepth
      );
    }
  }
  return result;
}

function generateArray(
  schema: OpenAPIV3_1.SchemaObject,
  depth: number,
  maxDepth: number
): unknown[] {
  const items = schema.items as OpenAPIV3_1.SchemaObject | undefined;
  if (!items) return [];

  const min = schema.minItems ?? 1;
  const max = Math.min(schema.maxItems ?? 3, 10);
  const count = faker.number.int({ min, max });

  return Array.from({ length: count }, () =>
    walkSchema(items, depth + 1, maxDepth)
  );
}

function generateString(schema: OpenAPIV3_1.SchemaObject): string {
  switch (schema.format) {
    case 'date-time': return faker.date.recent().toISOString();
    case 'date':      return faker.date.recent().toISOString().split('T')[0];
    case 'time':      return faker.date.recent().toISOString().split('T')[1];
    case 'email':     return faker.internet.email();
    case 'uri':
    case 'url':       return faker.internet.url();
    case 'uuid':      return faker.string.uuid();
    case 'ipv4':      return faker.internet.ipv4();
    case 'ipv6':      return faker.internet.ipv6();
    case 'hostname':  return faker.internet.domainName();
    default: {
      const minLen = schema.minLength ?? 3;
      const maxLen = schema.maxLength ?? 20;
      if (schema.pattern) {
        return faker.helpers.fromRegExp(schema.pattern);
      }
      return faker.string.alphanumeric({ length: { min: minLen, max: maxLen } });
    }
  }
}

function generateNumber(schema: OpenAPIV3_1.SchemaObject): number {
  const min = (schema.minimum ?? 0) + (schema.exclusiveMinimum ? 1 : 0);
  const max = (schema.maximum ?? 10000) - (schema.exclusiveMaximum ? 1 : 0);
  const value = faker.number.float({ min, max, fractionDigits: 2 });
  return schema.type === 'integer' ? Math.round(value) : value;
}
```

**Request Matcher:**

```typescript
// packages/server/src/request-matcher.ts
import { IndexedOperation } from '@openapi-mock/core';

export interface MatchResult {
  matched: boolean;
  operation?: IndexedOperation;
  pathParams: Record<string, string>;
}

export function matchRequest(
  method: string,
  path: string,
  index: IndexedOperation[]
): MatchResult {
  const upperMethod = method.toUpperCase();

  for (const op of index) {
    if (op.httpMethod !== upperMethod) continue;

    const match = path.match(op.pathRegex);
    if (!match) continue;

    const pathParams: Record<string, string> = {};
    op.pathParams.forEach((param, i) => {
      pathParams[param] = match[i + 1];
    });

    return { matched: true, operation: op, pathParams };
  }

  return { matched: false, pathParams: {} };
}
```

**Server factory:**

```typescript
// packages/server/src/server.ts
import Fastify, { FastifyInstance } from 'fastify';
import cors from '@fastify/cors';
import { IndexedOperation, walkSchema } from '@openapi-mock/core';
import { matchRequest } from './request-matcher.js';

export interface MockServerOptions {
  port: number;
  corsEnabled: boolean;
  validateRequests: boolean;
  latencyMs: number;
}

export async function createMockServer(
  operations: IndexedOperation[],
  options: MockServerOptions
): Promise<FastifyInstance> {
  const app = Fastify({ logger: true });

  if (options.corsEnabled) {
    await app.register(cors, { origin: true });
  }

  // Catch-all route: match every inbound request against the operation index
  app.all('/*', async (request, reply) => {
    const { matched, operation, pathParams } = matchRequest(
      request.method,
      request.url.split('?')[0],  // strip query string
      operations
    );

    if (!matched || !operation) {
      return reply.status(404).send({
        type: 'https://openapi-mock.dev/errors/no-matching-operation',
        title: 'No Matching Operation',
        status: 404,
        detail: `No operation found for ${request.method} ${request.url}`,
      });
    }

    // Select the success response (prefer 200, then 201, then first 2xx)
    const responses = operation.operationDef.responses ?? {};
    const successCode = findSuccessCode(responses);
    const responseDef = responses[successCode];

    if (!responseDef || !('content' in responseDef)) {
      return reply.status(Number(successCode)).send();
    }

    const content = responseDef.content?.['application/json'];
    if (!content?.schema) {
      return reply.status(Number(successCode)).send();
    }

    const body = walkSchema(content.schema as any);

    if (options.latencyMs > 0) {
      await new Promise(r => setTimeout(r, options.latencyMs));
    }

    return reply.status(Number(successCode)).send(body);
  });

  return app;
}

function findSuccessCode(responses: Record<string, any>): string {
  if (responses['200']) return '200';
  if (responses['201']) return '201';
  const twoXX = Object.keys(responses).find(c => c.startsWith('2'));
  return twoXX ?? Object.keys(responses)[0] ?? '200';
}
```

### Testing

- **Unit: schema-walker** — Generate data for object, array, nested object, string formats (email, uuid, date-time), integer with min/max, enum, allOf/oneOf/anyOf; verify structural validity
- **Unit: request-matcher** — Match `/users` GET, `/users/123` GET, `/users/123/orders` POST; verify correct operation matched and path params extracted; verify unmatched paths return `matched: false`
- **Integration: mock server** — Load petstore spec, start server, send `GET /pets`, verify 200 response with array of objects matching Pet schema; send `GET /nonexistent`, verify 404 with RFC 7807 body
- **Integration: content negotiation** — Send request with `Accept: application/xml` to endpoint that only defines `application/json`; verify appropriate response
- **Integration: CORS** — Send preflight OPTIONS request; verify CORS headers present
- **Edge cases:** Operation with no responses defined, operation with only `default` response, deeply nested schema (10+ levels)

---

## Phase 3: Request Validation

**Goal:** Validate incoming requests against the OpenAPI spec and return informative 4xx errors.

### What

- Validate path parameters against their schema (type, format, enum)
- Validate query parameters: required params present, correct types, enum compliance
- Validate request headers: required headers present, correct format
- Validate request body against the operation's requestBody schema (JSON Schema validation)
- Return RFC 7807 Problem Details responses for validation failures with field-level error details
- Make validation configurable: `--validate` flag (default: on), `--strict` for treating warnings as errors

### Design

**Request Validator:**

```typescript
// packages/core/src/validator/request-validator.ts
import Ajv from 'ajv';
import addFormats from 'ajv-formats';
import { OpenAPIV3_1 } from 'openapi-types';
import { IndexedOperation } from '../parser/operation-indexer.js';

const ajv = new Ajv({ allErrors: true, coerceTypes: true });
addFormats(ajv);

export interface ValidationError {
  field: string;       // 'query.limit', 'body.email', 'path.userId'
  message: string;
  code: string;        // 'required', 'type', 'format', 'enum', 'pattern'
  expected?: string;
  received?: string;
}

export interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];
}

export function validateRequest(
  operation: IndexedOperation,
  request: {
    pathParams: Record<string, string>;
    query: Record<string, string | string[]>;
    headers: Record<string, string>;
    body?: unknown;
    contentType?: string;
  }
): ValidationResult {
  const errors: ValidationError[] = [];

  // 1. Validate parameters (path, query, header)
  const params = (operation.operationDef.parameters ?? []) as OpenAPIV3_1.ParameterObject[];

  for (const param of params) {
    const value = getParamValue(param, request);

    if (param.required && value === undefined) {
      errors.push({
        field: `${param.in}.${param.name}`,
        message: `Required ${param.in} parameter '${param.name}' is missing`,
        code: 'required',
      });
      continue;
    }

    if (value !== undefined && param.schema) {
      const paramErrors = validateAgainstSchema(
        value,
        param.schema as OpenAPIV3_1.SchemaObject,
        `${param.in}.${param.name}`
      );
      errors.push(...paramErrors);
    }
  }

  // 2. Validate request body
  const reqBody = operation.operationDef.requestBody as OpenAPIV3_1.RequestBodyObject | undefined;
  if (reqBody) {
    if (reqBody.required && request.body === undefined) {
      errors.push({
        field: 'body',
        message: 'Request body is required',
        code: 'required',
      });
    } else if (request.body !== undefined) {
      const mediaType = request.contentType ?? 'application/json';
      const content = reqBody.content?.[mediaType];
      if (content?.schema) {
        const bodyErrors = validateAgainstSchema(
          request.body,
          content.schema as OpenAPIV3_1.SchemaObject,
          'body'
        );
        errors.push(...bodyErrors);
      }
    }
  }

  return { valid: errors.length === 0, errors };
}

function validateAgainstSchema(
  value: unknown,
  schema: OpenAPIV3_1.SchemaObject,
  fieldPrefix: string
): ValidationError[] {
  const validate = ajv.compile(schema);
  const valid = validate(value);

  if (valid) return [];

  return (validate.errors ?? []).map(err => ({
    field: `${fieldPrefix}${err.instancePath}`.replace(/\//g, '.'),
    message: err.message ?? 'Validation failed',
    code: err.keyword,
    expected: JSON.stringify(err.params),
    received: typeof value === 'object' ? undefined : String(value),
  }));
}
```

**RFC 7807 Error Builder:**

```typescript
// packages/core/src/errors/rfc7807.ts
export interface ProblemDetails {
  type: string;
  title: string;
  status: number;
  detail: string;
  instance?: string;
  errors?: Array<{
    field: string;
    message: string;
    code: string;
  }>;
}

export function createValidationProblem(
  path: string,
  errors: Array<{ field: string; message: string; code: string }>
): ProblemDetails {
  return {
    type: 'https://openapi-mock.dev/errors/request-validation',
    title: 'Request Validation Failed',
    status: 400,
    detail: `${errors.length} validation error(s) found in the request`,
    instance: path,
    errors,
  };
}

export function createNotFoundProblem(method: string, path: string): ProblemDetails {
  return {
    type: 'https://openapi-mock.dev/errors/no-matching-operation',
    title: 'No Matching Operation',
    status: 404,
    detail: `No operation defined for ${method} ${path} in the OpenAPI spec`,
    instance: path,
  };
}
```

### Testing

- **Unit: request-validator** — Validate requests with missing required query params, wrong type path params, body with missing required fields, body with extra fields in strict mode, valid requests pass
- **Unit: RFC 7807 builder** — Verify output matches RFC 7807 structure with correct `type` URI, `status`, and field-level `errors` array
- **Integration: validation flow** — Start mock server with `--validate`; send POST /pets with missing `name` field; verify 400 response with RFC 7807 body listing the missing field; send same request with `--no-validate`; verify 201 response
- **Edge cases:** Request with content-type mismatch (sending XML to JSON endpoint), multipart form data, empty body for required body, deeply nested body validation

---

## Phase 4: Heuristic Semantic Data Generation

**Goal:** Generate semantically realistic data based on field names and schema descriptions without AI API calls.

### What

- Build a heuristic engine that maps field names to Faker.js generators (e.g., `firstName` -> `faker.person.firstName()`, `email` -> `faker.internet.email()`, `price` -> `faker.commerce.price()`)
- Support compound field names: `user.firstName`, `billing_address.zip_code`, `createdAt`
- Use schema `description` field as an additional signal (e.g., description containing "phone" triggers phone number generation)
- Support format overrides via schema `x-mock-*` extension properties
- Integrate into the data generation pipeline: heuristic first, then Faker fallback
- Generate referentially consistent data within a single response (same user ID referenced in multiple places)

### Design

**Heuristic Generator:**

```typescript
// packages/core/src/generator/heuristic-generator.ts
import { faker } from '@faker-js/faker';

// Ordered by specificity: more specific patterns first
const FIELD_PATTERNS: Array<{
  pattern: RegExp;
  generator: () => unknown;
  priority: number;
}> = [
  // Identity
  { pattern: /^(id|uuid|guid)$/i,                     generator: () => faker.string.uuid(), priority: 100 },
  { pattern: /(^|_)(user|account|customer)_?id$/i,     generator: () => faker.string.uuid(), priority: 95 },

  // Person names
  { pattern: /^(first_?name|given_?name)$/i,           generator: () => faker.person.firstName(), priority: 90 },
  { pattern: /^(last_?name|family_?name|surname)$/i,   generator: () => faker.person.lastName(), priority: 90 },
  { pattern: /^(full_?name|display_?name|name)$/i,     generator: () => faker.person.fullName(), priority: 85 },
  { pattern: /^(user_?name|username|login)$/i,         generator: () => faker.internet.username(), priority: 85 },

  // Contact
  { pattern: /^(email|email_?address)$/i,              generator: () => faker.internet.email(), priority: 90 },
  { pattern: /^(phone|phone_?number|mobile|tel)$/i,    generator: () => faker.phone.number(), priority: 90 },

  // Address
  { pattern: /^(street|street_?address|address_?line)$/i,  generator: () => faker.location.streetAddress(), priority: 85 },
  { pattern: /^(city|town)$/i,                             generator: () => faker.location.city(), priority: 85 },
  { pattern: /^(state|province|region)$/i,                 generator: () => faker.location.state(), priority: 85 },
  { pattern: /^(country|country_?code)$/i,                 generator: () => faker.location.countryCode(), priority: 85 },
  { pattern: /^(zip|zip_?code|postal_?code|postcode)$/i,  generator: () => faker.location.zipCode(), priority: 85 },
  { pattern: /^(latitude|lat)$/i,                          generator: () => faker.location.latitude(), priority: 80 },
  { pattern: /^(longitude|lng|lon)$/i,                     generator: () => faker.location.longitude(), priority: 80 },

  // Dates & times
  { pattern: /^(created_?at|created_?date|created)$/i,    generator: () => faker.date.past().toISOString(), priority: 85 },
  { pattern: /^(updated_?at|modified_?at|modified)$/i,    generator: () => faker.date.recent().toISOString(), priority: 85 },
  { pattern: /^(deleted_?at)$/i,                           generator: () => null, priority: 85 },
  { pattern: /^(birth_?date|date_?of_?birth|dob)$/i,      generator: () => faker.date.birthdate().toISOString().split('T')[0], priority: 85 },
  { pattern: /^(start_?date|begin_?date|from)$/i,         generator: () => faker.date.past().toISOString(), priority: 80 },
  { pattern: /^(end_?date|expire|expiry|until)$/i,        generator: () => faker.date.future().toISOString(), priority: 80 },

  // Commerce
  { pattern: /^(price|amount|cost|total|subtotal)$/i,     generator: () => parseFloat(faker.commerce.price()), priority: 85 },
  { pattern: /^(currency|currency_?code)$/i,               generator: () => faker.finance.currencyCode(), priority: 85 },
  { pattern: /^(quantity|qty|count)$/i,                    generator: () => faker.number.int({ min: 1, max: 100 }), priority: 80 },
  { pattern: /^(sku|product_?code)$/i,                     generator: () => faker.string.alphanumeric(8).toUpperCase(), priority: 80 },

  // Web
  { pattern: /^(url|link|href|website|homepage)$/i,        generator: () => faker.internet.url(), priority: 85 },
  { pattern: /^(avatar|avatar_?url|profile_?image|photo)$/i, generator: () => faker.image.avatar(), priority: 80 },
  { pattern: /^(ip|ip_?address)$/i,                        generator: () => faker.internet.ipv4(), priority: 80 },
  { pattern: /^(slug)$/i,                                  generator: () => faker.helpers.slugify(faker.lorem.words(3)), priority: 80 },

  // Text
  { pattern: /^(title|subject|headline)$/i,                generator: () => faker.lorem.sentence({ min: 3, max: 8 }), priority: 75 },
  { pattern: /^(description|summary|bio|about)$/i,        generator: () => faker.lorem.paragraph(), priority: 75 },
  { pattern: /^(body|content|text|message)$/i,             generator: () => faker.lorem.paragraphs(2), priority: 70 },
  { pattern: /^(tag|label|category)$/i,                    generator: () => faker.word.noun(), priority: 70 },

  // Status & flags
  { pattern: /^(status|state)$/i,                          generator: () => faker.helpers.arrayElement(['active', 'inactive', 'pending']), priority: 70 },
  { pattern: /^(is_?active|active|enabled)$/i,             generator: () => true, priority: 75 },
  { pattern: /^(is_?deleted|deleted|archived)$/i,          generator: () => false, priority: 75 },
  { pattern: /^(role|type)$/i,                             generator: () => faker.helpers.arrayElement(['admin', 'user', 'moderator']), priority: 65 },
];

export function heuristicGenerate(
  fieldName: string,
  schema: { description?: string; type?: string; format?: string }
): { value: unknown; confidence: number } | null {
  const normalizedName = fieldName.split('.').pop() ?? fieldName;

  for (const { pattern, generator, priority } of FIELD_PATTERNS) {
    if (pattern.test(normalizedName)) {
      return { value: generator(), confidence: priority / 100 };
    }
  }

  // Check description for hints
  if (schema.description) {
    const desc = schema.description.toLowerCase();
    for (const { pattern, generator, priority } of FIELD_PATTERNS) {
      if (pattern.test(desc)) {
        return { value: generator(), confidence: (priority - 20) / 100 };
      }
    }
  }

  return null; // No heuristic match; fall through to next generator
}
```

**Data Generation Orchestrator** — the pipeline that ties heuristic, AI, and faker together:

```typescript
// packages/core/src/generator/data-generator.ts
import { OpenAPIV3_1 } from 'openapi-types';
import { heuristicGenerate } from './heuristic-generator.js';
import { walkSchema } from './schema-walker.js';

export interface GeneratorConfig {
  aiEnabled: boolean;
  heuristicEnabled: boolean;  // default: true
  seed?: number;              // for deterministic output in tests
}

export function generateResponseData(
  schema: OpenAPIV3_1.SchemaObject,
  config: GeneratorConfig,
  fieldName?: string
): unknown {
  // Priority 1: Spec example
  if (schema.example !== undefined) return schema.example;

  // Priority 2: x-mock-value extension
  if ((schema as any)['x-mock-value'] !== undefined) {
    return (schema as any)['x-mock-value'];
  }

  // Priority 3: Heuristic (field-name-aware)
  if (config.heuristicEnabled && fieldName) {
    const heuristic = heuristicGenerate(fieldName, schema);
    if (heuristic && heuristic.confidence > 0.6) {
      return heuristic.value;
    }
  }

  // Priority 4: AI generation (Phase 5)
  // if (config.aiEnabled) { ... }

  // Priority 5: Structural faker fallback
  return walkSchema(schema);
}
```

### Testing

- **Unit: heuristic-generator** — Test all 30+ field patterns with exact and variant names: `email` -> valid email, `firstName` -> realistic first name, `createdAt` -> ISO date, `price` -> positive number
- **Unit: compound names** — Test `billing_address.zip_code`, `user.email`, `shipping.city` correctly decompose and match
- **Unit: description fallback** — Field named `contact_info` with description "The user's primary email address" should generate an email
- **Unit: data-generator pipeline** — Verify priority order: spec example wins over heuristic, heuristic wins over faker, x-mock-value wins over all
- **Integration: realistic responses** — Load a user management spec, generate response for GET /users; verify response contains realistic names, emails, dates rather than random strings
- **Comparison test** — Generate the same schema with heuristics on vs. off; assert heuristic output passes human readability check (email format, name structure, etc.)

---

## Phase 5: AI-Powered Data Generation

**Goal:** Integrate Claude API for generating semantically realistic data for complex schemas that heuristics cannot handle.

### What

- Implement Claude API integration that sends schema definitions with field names and descriptions, receiving structured JSON data
- Build an LRU cache (in-memory, keyed by schema SHA-256 hash) to avoid redundant API calls
- Implement batch generation: pre-generate data for all schemas at spec load time (background task)
- Support configurable AI provider: `--ai-provider claude` (default) with extension points for other providers
- Graceful degradation: if AI API is unavailable or key not set, fall through to heuristic/faker
- Rate limiting and token tracking for cost control

### Design

**AI Generator:**

```typescript
// packages/core/src/generator/ai-generator.ts
import Anthropic from '@anthropic-ai/sdk';
import { createHash } from 'node:crypto';
import { OpenAPIV3_1 } from 'openapi-types';

export interface AIGeneratorConfig {
  apiKey: string;
  model: string;              // default: 'claude-sonnet-4-20250514'
  maxTokensPerRequest: number; // default: 1024
  cacheTTLMs: number;         // default: 3600000 (1 hour)
  maxCacheSize: number;       // default: 1000 entries
}

interface CacheEntry {
  data: unknown;
  createdAt: number;
  hitCount: number;
}

export class AIGenerator {
  private client: Anthropic;
  private cache: Map<string, CacheEntry> = new Map();
  private config: AIGeneratorConfig;

  constructor(config: AIGeneratorConfig) {
    this.config = config;
    this.client = new Anthropic({ apiKey: config.apiKey });
  }

  async generate(
    schema: OpenAPIV3_1.SchemaObject,
    context: {
      operationId?: string;
      path?: string;
      method?: string;
      fieldName?: string;
    }
  ): Promise<unknown> {
    const cacheKey = this.computeCacheKey(schema, context);

    // Check cache
    const cached = this.cache.get(cacheKey);
    if (cached && Date.now() - cached.createdAt < this.config.cacheTTLMs) {
      cached.hitCount++;
      return cached.data;
    }

    const prompt = this.buildPrompt(schema, context);

    try {
      const response = await this.client.messages.create({
        model: this.config.model,
        max_tokens: this.config.maxTokensPerRequest,
        messages: [{ role: 'user', content: prompt }],
      });

      const text = response.content
        .filter(block => block.type === 'text')
        .map(block => block.text)
        .join('');

      // Extract JSON from response (handle markdown code blocks)
      const jsonMatch = text.match(/```json\s*([\s\S]*?)\s*```/) ?? [null, text];
      const data = JSON.parse(jsonMatch[1] ?? text);

      // Cache result
      this.evictIfNeeded();
      this.cache.set(cacheKey, {
        data,
        createdAt: Date.now(),
        hitCount: 0,
      });

      return data;
    } catch (error) {
      // Graceful degradation: return null, caller falls through to faker
      console.warn(`AI generation failed: ${error}. Falling back to faker.`);
      return null;
    }
  }

  private buildPrompt(
    schema: OpenAPIV3_1.SchemaObject,
    context: { operationId?: string; path?: string; method?: string }
  ): string {
    return `Generate a single realistic JSON example for this API response schema.

Context:
- API operation: ${context.method ?? 'GET'} ${context.path ?? '/unknown'}
- Operation ID: ${context.operationId ?? 'unknown'}

Schema:
${JSON.stringify(schema, null, 2)}

Requirements:
- Return ONLY valid JSON matching the schema exactly
- Use semantically realistic values (real-looking names, valid emails, plausible dates)
- Respect all constraints: required fields, enums, formats, min/max values
- Make data internally consistent (e.g., a user's email domain matches their company)
- Do not include any explanation, only the JSON object`;
  }

  private computeCacheKey(
    schema: OpenAPIV3_1.SchemaObject,
    context: Record<string, unknown>
  ): string {
    const input = JSON.stringify({ schema, context });
    return createHash('sha256').update(input).digest('hex');
  }

  private evictIfNeeded(): void {
    if (this.cache.size < this.config.maxCacheSize) return;

    // LRU eviction: remove the entry with oldest createdAt + lowest hitCount
    let oldestKey = '';
    let oldestScore = Infinity;
    for (const [key, entry] of this.cache) {
      const score = entry.createdAt + entry.hitCount * 60000;
      if (score < oldestScore) {
        oldestScore = score;
        oldestKey = key;
      }
    }
    if (oldestKey) this.cache.delete(oldestKey);
  }

  getCacheStats(): { size: number; hitRate: number } {
    let totalHits = 0;
    for (const entry of this.cache.values()) {
      totalHits += entry.hitCount;
    }
    return {
      size: this.cache.size,
      hitRate: this.cache.size > 0 ? totalHits / this.cache.size : 0,
    };
  }
}
```

**Updated Data Generator pipeline (Phase 4 + 5):**

```typescript
// packages/core/src/generator/data-generator.ts (updated)
export async function generateResponseData(
  schema: OpenAPIV3_1.SchemaObject,
  config: GeneratorConfig,
  context: { fieldName?: string; operationId?: string; path?: string; method?: string },
  aiGenerator?: AIGenerator
): Promise<unknown> {
  // Priority 1: Spec example
  if (schema.example !== undefined) return schema.example;

  // Priority 2: x-mock-value extension
  if ((schema as any)['x-mock-value'] !== undefined) {
    return (schema as any)['x-mock-value'];
  }

  // Priority 3: Heuristic (for simple fields)
  if (config.heuristicEnabled && context.fieldName) {
    const heuristic = heuristicGenerate(context.fieldName, schema);
    if (heuristic && heuristic.confidence > 0.6) {
      return heuristic.value;
    }
  }

  // Priority 4: AI generation (for complex objects/arrays)
  if (config.aiEnabled && aiGenerator && isComplexSchema(schema)) {
    const aiResult = await aiGenerator.generate(schema, context);
    if (aiResult !== null) return aiResult;
  }

  // Priority 5: Structural faker fallback
  return walkSchema(schema);
}

function isComplexSchema(schema: OpenAPIV3_1.SchemaObject): boolean {
  if (schema.type === 'object' && schema.properties) {
    return Object.keys(schema.properties).length >= 3;
  }
  if (schema.type === 'array' && schema.items) return true;
  if (schema.allOf || schema.oneOf || schema.anyOf) return true;
  return false;
}
```

### Testing

- **Unit: AI generator** — Mock Claude API; verify prompt includes schema and context; verify JSON parsing from response; verify cache key is deterministic for same input
- **Unit: cache behavior** — Generate same schema twice; verify second call is a cache hit; verify LRU eviction when cache is full; verify TTL expiration
- **Unit: graceful degradation** — Simulate API failure (network error, 429, 500); verify null return and warning logged; verify pipeline falls through to faker
- **Integration: data quality** — Generate data for a complex User schema via AI; verify output has realistic names, valid email format, internally consistent data
- **Integration: batch pre-generation** — Load spec with 20 schemas; verify all are pre-generated on startup; verify subsequent requests use cached data
- **Cost tracking** — Verify token usage is tracked per generation call; verify cache hit ratio is reported in stats

---

## Phase 6: Request Logging & Debug UI

**Goal:** Record all requests with match details and provide admin API endpoints for inspecting mock behavior.

### What

- Implement in-memory request log (ring buffer, configurable max size, default 1000 entries)
- Log: timestamp, method, path, matched operation, match result, validation errors, response status, response source (example/heuristic/ai/faker), duration
- Admin API: `GET /__admin/logs` with filtering (by status, method, path, match result)
- Admin API: `GET /__admin/operations` — list all registered operations with match stats
- Admin API: `GET /__admin/config` — show current configuration
- Admin API: `POST /__admin/reset` — clear logs and reset session state
- Admin API: `GET /__admin/health` — health check with uptime and stats

### Design

**Request Log Store:**

```typescript
// packages/server/src/admin/request-log-store.ts
export interface LogEntry {
  id: string;
  timestamp: string;
  request: {
    method: string;
    path: string;
    headers: Record<string, string>;
    query: Record<string, string>;
    body?: unknown;
  };
  match: {
    result: 'matched' | 'unmatched' | 'validation_error';
    operationId?: string;
    pathPattern?: string;
    validationErrors?: Array<{ field: string; message: string }>;
  };
  response: {
    status: number;
    source: 'spec_example' | 'heuristic' | 'ai_generated' | 'faker' | 'rule_override' | 'error';
    durationMs: number;
  };
}

export class RequestLogStore {
  private entries: LogEntry[] = [];
  private maxSize: number;

  constructor(maxSize: number = 1000) {
    this.maxSize = maxSize;
  }

  add(entry: LogEntry): void {
    this.entries.push(entry);
    if (this.entries.length > this.maxSize) {
      this.entries.shift(); // Ring buffer: drop oldest
    }
  }

  query(filters: {
    method?: string;
    path?: string;
    status?: number;
    matchResult?: string;
    limit?: number;
    offset?: number;
  }): { entries: LogEntry[]; total: number } {
    let filtered = this.entries;

    if (filters.method) {
      filtered = filtered.filter(e => e.request.method === filters.method!.toUpperCase());
    }
    if (filters.path) {
      filtered = filtered.filter(e => e.request.path.includes(filters.path!));
    }
    if (filters.status) {
      filtered = filtered.filter(e => e.response.status === filters.status);
    }
    if (filters.matchResult) {
      filtered = filtered.filter(e => e.match.result === filters.matchResult);
    }

    const total = filtered.length;
    const offset = filters.offset ?? 0;
    const limit = filters.limit ?? 50;

    return {
      entries: filtered.slice(offset, offset + limit).reverse(), // newest first
      total,
    };
  }

  getStats(): {
    total: number;
    matched: number;
    unmatched: number;
    validationErrors: number;
    bySource: Record<string, number>;
  } {
    const stats = {
      total: this.entries.length,
      matched: 0, unmatched: 0, validationErrors: 0,
      bySource: {} as Record<string, number>,
    };

    for (const entry of this.entries) {
      if (entry.match.result === 'matched') stats.matched++;
      else if (entry.match.result === 'unmatched') stats.unmatched++;
      else stats.validationErrors++;

      const source = entry.response.source;
      stats.bySource[source] = (stats.bySource[source] ?? 0) + 1;
    }

    return stats;
  }

  clear(): void {
    this.entries = [];
  }
}
```

**Admin Routes:**

```typescript
// packages/server/src/admin/admin-routes.ts
import { FastifyInstance } from 'fastify';
import { RequestLogStore } from './request-log-store.js';
import { IndexedOperation } from '@openapi-mock/core';

export async function registerAdminRoutes(
  app: FastifyInstance,
  logStore: RequestLogStore,
  operations: IndexedOperation[],
  startTime: number
): Promise<void> {

  app.get('/__admin/health', async () => ({
    status: 'ok',
    uptime: Math.floor((Date.now() - startTime) / 1000),
    operationCount: operations.length,
    logStats: logStore.getStats(),
  }));

  app.get('/__admin/logs', async (request) => {
    const query = request.query as Record<string, string>;
    return logStore.query({
      method: query.method,
      path: query.path,
      status: query.status ? parseInt(query.status) : undefined,
      matchResult: query.matchResult,
      limit: query.limit ? parseInt(query.limit) : undefined,
      offset: query.offset ? parseInt(query.offset) : undefined,
    });
  });

  app.get('/__admin/operations', async () =>
    operations.map(op => ({
      method: op.httpMethod,
      path: op.pathPattern,
      operationId: op.operationId,
      summary: op.summary,
      responseCodes: op.responseCodes,
      hasRequestBody: op.hasRequestBody,
    }))
  );

  app.post('/__admin/reset', async () => {
    logStore.clear();
    return { message: 'Logs cleared and sessions reset' };
  });
}
```

### Testing

- **Unit: RequestLogStore** — Add entries; verify ring buffer evicts oldest when maxSize exceeded; verify query filters (by method, path, status, matchResult) return correct subsets; verify stats aggregation
- **Integration: admin endpoints** — Start mock server; send requests to mock endpoints; query `/__admin/logs` and verify entries appear; query `/__admin/health` and verify uptime/stats; call `/__admin/reset` and verify logs are cleared
- **Integration: response source tracking** — Generate responses from spec example, heuristic, and faker; verify each log entry records the correct `source`
- **Edge cases:** Admin routes do not conflict with spec-defined routes (spec has a `/__admin` path); query with no filters returns all entries; empty log returns empty array

---

## Phase 7: Stateful Session Simulation

**Goal:** Maintain in-memory state across requests to simulate realistic multi-step API workflows.

### What

- Implement session store with token-based session identification (via `X-Mock-Session` header or query param)
- Implement CRUD state management: POST creates resources in session state, GET reads them, PUT/PATCH updates, DELETE removes
- Implement scenario engine for predefined state machines (YAML/JSON definition files)
- Auto-detect CRUD patterns from OpenAPI spec structure (POST /resources, GET /resources/{id}, etc.)
- Session expiration with configurable TTL
- Session inspection via admin API: `GET /__admin/sessions`, `GET /__admin/sessions/:id`

### Design

**Session Store:**

```typescript
// packages/server/src/session/session-store.ts
import { randomBytes } from 'node:crypto';

export interface Session {
  id: string;
  token: string;
  state: Record<string, unknown>;  // e.g., { users: [...], orders: [...] }
  scenarioState?: string;          // current state in state machine
  createdAt: number;
  updatedAt: number;
  expiresAt: number;
  requestCount: number;
}

export class SessionStore {
  private sessions: Map<string, Session> = new Map();
  private ttlMs: number;

  constructor(ttlMs: number = 3600000) { // 1 hour default
    this.ttlMs = ttlMs;
  }

  getOrCreate(token?: string): Session {
    if (token) {
      const existing = this.sessions.get(token);
      if (existing && existing.expiresAt > Date.now()) {
        return existing;
      }
    }

    const session: Session = {
      id: randomBytes(16).toString('hex'),
      token: token ?? randomBytes(24).toString('base64url'),
      state: {},
      createdAt: Date.now(),
      updatedAt: Date.now(),
      expiresAt: Date.now() + this.ttlMs,
      requestCount: 0,
    };

    this.sessions.set(session.token, session);
    return session;
  }

  /**
   * Apply a CRUD mutation to session state.
   * The resourceKey is derived from the path (e.g., '/users' -> 'users').
   */
  applyCrudMutation(
    session: Session,
    method: string,
    resourceKey: string,
    resourceId: string | undefined,
    body: unknown
  ): { mutated: boolean; resource?: unknown } {
    if (!session.state[resourceKey]) {
      session.state[resourceKey] = [];
    }
    const collection = session.state[resourceKey] as any[];

    switch (method) {
      case 'POST': {
        const newResource = {
          id: randomBytes(8).toString('hex'),
          ...(body as Record<string, unknown>),
          createdAt: new Date().toISOString(),
        };
        collection.push(newResource);
        session.updatedAt = Date.now();
        return { mutated: true, resource: newResource };
      }
      case 'GET': {
        if (resourceId) {
          const found = collection.find((r: any) => r.id === resourceId);
          return { mutated: false, resource: found };
        }
        return { mutated: false, resource: collection };
      }
      case 'PUT':
      case 'PATCH': {
        const idx = collection.findIndex((r: any) => r.id === resourceId);
        if (idx >= 0) {
          collection[idx] = method === 'PUT'
            ? { ...body as object, id: resourceId }
            : { ...collection[idx], ...body as object };
          collection[idx].updatedAt = new Date().toISOString();
          session.updatedAt = Date.now();
          return { mutated: true, resource: collection[idx] };
        }
        return { mutated: false };
      }
      case 'DELETE': {
        const deleteIdx = collection.findIndex((r: any) => r.id === resourceId);
        if (deleteIdx >= 0) {
          const deleted = collection.splice(deleteIdx, 1)[0];
          session.updatedAt = Date.now();
          return { mutated: true, resource: deleted };
        }
        return { mutated: false };
      }
      default:
        return { mutated: false };
    }
  }

  cleanup(): number {
    const now = Date.now();
    let removed = 0;
    for (const [token, session] of this.sessions) {
      if (session.expiresAt <= now) {
        this.sessions.delete(token);
        removed++;
      }
    }
    return removed;
  }
}
```

**CRUD Pattern Detector:**

```typescript
// packages/server/src/session/crud-detector.ts
import { IndexedOperation } from '@openapi-mock/core';

export interface CrudGroup {
  resourceKey: string;       // 'users', 'orders'
  collectionPath: string;    // '/users'
  itemPath?: string;         // '/users/{userId}'
  idParam?: string;          // 'userId'
  operations: Map<string, IndexedOperation>;  // method -> operation
}

export function detectCrudGroups(index: IndexedOperation[]): CrudGroup[] {
  const groups: Map<string, CrudGroup> = new Map();

  for (const op of index) {
    // Match pattern: /resources or /resources/{id}
    const collectionMatch = op.pathPattern.match(/^\/(\w+)$/);
    const itemMatch = op.pathPattern.match(/^\/(\w+)\/\{(\w+)\}$/);

    const resourceKey = collectionMatch?.[1] ?? itemMatch?.[1];
    if (!resourceKey) continue;

    if (!groups.has(resourceKey)) {
      groups.set(resourceKey, {
        resourceKey,
        collectionPath: `/${resourceKey}`,
        operations: new Map(),
      });
    }

    const group = groups.get(resourceKey)!;
    group.operations.set(op.httpMethod, op);

    if (itemMatch) {
      group.itemPath = op.pathPattern;
      group.idParam = itemMatch[2];
    }
  }

  // Only return groups that have at least POST + GET (minimum CRUD)
  return [...groups.values()].filter(
    g => g.operations.has('POST') || g.operations.has('GET')
  );
}
```

### Testing

- **Unit: SessionStore** — Create session; verify token; add resources via POST; retrieve via GET; update via PATCH; delete; verify state mutations; verify TTL expiration
- **Unit: CRUD detector** — Given petstore index, detect `pets` CRUD group with POST/GET/PUT/DELETE operations; verify nested paths like `/users/{userId}/orders` are handled
- **Integration: stateful flow** — Start mock server with sessions enabled; POST /users with body; GET /users returns array including posted user; GET /users/{id} returns the specific user; DELETE /users/{id} removes it; subsequent GET returns 404
- **Integration: session isolation** — Two different session tokens; POST resource in session A; verify it does not appear in session B
- **Integration: session headers** — Verify `X-Mock-Session` header in response contains the session token; verify subsequent requests with same token maintain state
- **Edge cases:** POST without body, GET on empty collection, DELETE with non-existent ID, expired session returns fresh state

---

## Phase 8: CLI Polish & Docker

**Goal:** Production-ready CLI with watch mode, Docker image, and npm publishing configuration.

### What

- Implement `--watch` flag: monitor spec file for changes and hot-reload the mock server
- Implement `--dynamic` flag: enable AI data generation (requires `ANTHROPIC_API_KEY` env var)
- Implement `openapi-mock validate <spec>` command: validate spec and report issues without starting server
- Implement `openapi-mock generate <spec> --operation <id>` command: generate and print sample response data
- Create multi-stage Dockerfile (build + runtime) with minimal image size
- Configure npm package publishing: dual ESM/CJS, `bin` entry, peerDependencies
- Add `--json` flag for machine-readable output across all commands
- Signal handling: graceful shutdown on SIGTERM/SIGINT

### Design

**Watch Mode Implementation:**

```typescript
// packages/cli/src/commands/mock.ts (watch mode addition)
import { watch } from 'node:fs';
import { resolve } from 'node:path';

async function startWithWatch(
  specPath: string,
  options: MockOptions
): Promise<void> {
  const absPath = resolve(specPath);
  let server: FastifyInstance | null = null;

  async function reload(): Promise<void> {
    if (server) {
      console.log(chalk.yellow('\n  Spec changed, reloading...'));
      await server.close();
    }

    try {
      const spec = await loadSpec(absPath);
      const index = buildOperationIndex(spec.parsed);
      server = await createMockServer(index, options);
      await server.listen({ port: options.port, host: '0.0.0.0' });
      console.log(chalk.green(`  Mock server restarted on port ${options.port}`));
    } catch (err) {
      console.error(chalk.red(`  Reload failed: ${err}`));
    }
  }

  await reload();

  const watcher = watch(absPath, { persistent: true }, (eventType) => {
    if (eventType === 'change') {
      reload();
    }
  });

  // Graceful shutdown
  const shutdown = async () => {
    console.log(chalk.dim('\n  Shutting down...'));
    watcher.close();
    if (server) await server.close();
    process.exit(0);
  };

  process.on('SIGINT', shutdown);
  process.on('SIGTERM', shutdown);
}
```

**Dockerfile:**

```dockerfile
# docker/Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
RUN corepack enable && corepack prepare pnpm@latest --activate
COPY pnpm-workspace.yaml pnpm-lock.yaml ./
COPY packages/ packages/
RUN pnpm install --frozen-lockfile
RUN pnpm -r build

FROM node:20-alpine AS runtime
WORKDIR /app
RUN corepack enable && corepack prepare pnpm@latest --activate
COPY --from=builder /app/packages/cli/dist ./cli/
COPY --from=builder /app/packages/server/dist ./server/
COPY --from=builder /app/packages/core/dist ./core/
COPY --from=builder /app/node_modules ./node_modules/
COPY --from=builder /app/packages/cli/package.json ./package.json

EXPOSE 4010
ENTRYPOINT ["node", "cli/bin/openapi-mock.js"]
CMD ["mock", "--spec", "/spec/openapi.yaml"]
```

**package.json exports configuration:**

```json
{
  "name": "@openapi-mock/cli",
  "version": "0.1.0",
  "bin": {
    "openapi-mock": "./bin/openapi-mock.js"
  },
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    }
  },
  "engines": {
    "node": ">=20"
  }
}
```

### Testing

- **Integration: watch mode** — Start mock server with `--watch`; modify spec file; verify server restarts and reflects new operations without manual intervention
- **Integration: validate command** — Run `openapi-mock validate valid-spec.yaml`; verify success output; run against invalid spec; verify error report with line numbers
- **Integration: generate command** — Run `openapi-mock generate petstore.yaml --operation listPets`; verify JSON output printed to stdout matching the Pet schema
- **Docker: build and run** — Build Docker image; run with petstore spec mounted as volume; verify mock server responds to requests; verify image size < 200MB
- **npm: pack test** — Run `pnpm pack`; install in a clean project; verify `npx openapi-mock --version` works
- **Signal handling** — Send SIGTERM to running mock server; verify graceful shutdown (no connection errors for in-flight requests)

---

## Phase 9: Proxy Mode & Contract Drift Detection

**Goal:** Forward requests to a real upstream API, compare responses to the spec, and flag divergences.

### What

- Implement proxy mode: `openapi-mock proxy --spec spec.yaml --upstream https://api.example.com`
- Forward all matched requests to the upstream URL, return the upstream response to the client
- Compare upstream response against the spec schema: status code, content type, body structure, field types
- Detect drift types: missing fields, extra fields, type mismatches, enum violations, format violations
- Assign severity levels: breaking (type change, missing required field), warning (extra field, nullable change), info (format variation)
- Report drift events via admin API: `GET /__admin/drift`
- Console output: live-print drift events as they are detected

### Design

**Proxy Handler:**

```typescript
// packages/server/src/proxy/proxy-handler.ts
export interface ProxyConfig {
  upstreamUrl: string;
  driftDetectionEnabled: boolean;
  timeout: number;  // ms
}

export async function proxyRequest(
  request: FastifyRequest,
  operation: IndexedOperation,
  config: ProxyConfig
): Promise<{
  status: number;
  headers: Record<string, string>;
  body: unknown;
  driftEvents: DriftEvent[];
}> {
  const upstreamPath = request.url;
  const upstreamUrl = `${config.upstreamUrl}${upstreamPath}`;

  const upstreamResponse = await fetch(upstreamUrl, {
    method: request.method,
    headers: {
      ...request.headers as Record<string, string>,
      host: new URL(config.upstreamUrl).host,
    },
    body: request.method !== 'GET' && request.method !== 'HEAD'
      ? JSON.stringify(request.body)
      : undefined,
    signal: AbortSignal.timeout(config.timeout),
  });

  const body = await upstreamResponse.json().catch(() => null);
  const headers = Object.fromEntries(upstreamResponse.headers.entries());

  let driftEvents: DriftEvent[] = [];
  if (config.driftDetectionEnabled) {
    driftEvents = detectDrift(operation, {
      status: upstreamResponse.status,
      headers,
      body,
    });
  }

  return {
    status: upstreamResponse.status,
    headers,
    body,
    driftEvents,
  };
}
```

**Drift Detector:**

```typescript
// packages/server/src/proxy/drift-detector.ts
export interface DriftEvent {
  id: string;
  type: DriftType;
  severity: 'breaking' | 'warning' | 'info';
  operationId?: string;
  path: string;
  jsonPath: string;
  message: string;
  expected: unknown;
  actual: unknown;
  timestamp: string;
}

export type DriftType =
  | 'status_code_mismatch'
  | 'missing_required_field'
  | 'extra_field'
  | 'type_mismatch'
  | 'enum_violation'
  | 'format_violation'
  | 'missing_response_definition';

export function detectDrift(
  operation: IndexedOperation,
  response: { status: number; headers: Record<string, string>; body: unknown }
): DriftEvent[] {
  const events: DriftEvent[] = [];
  const statusStr = String(response.status);
  const responseDef = operation.operationDef.responses?.[statusStr]
    ?? operation.operationDef.responses?.['default'];

  // Drift: status code not defined in spec
  if (!operation.responseCodes.includes(statusStr) && !operation.responseCodes.includes('default')) {
    events.push({
      id: randomUUID(),
      type: 'missing_response_definition',
      severity: 'warning',
      operationId: operation.operationId,
      path: `${operation.httpMethod} ${operation.pathPattern}`,
      jsonPath: '$',
      message: `Response status ${response.status} is not defined in the spec`,
      expected: operation.responseCodes,
      actual: response.status,
      timestamp: new Date().toISOString(),
    });
  }

  // Schema-level drift detection
  if (responseDef && 'content' in responseDef) {
    const content = responseDef.content?.['application/json'];
    if (content?.schema && response.body !== null) {
      const schemaDrift = compareToSchema(
        response.body,
        content.schema as OpenAPIV3_1.SchemaObject,
        '$',
        operation
      );
      events.push(...schemaDrift);
    }
  }

  return events;
}

function compareToSchema(
  actual: unknown,
  schema: OpenAPIV3_1.SchemaObject,
  path: string,
  operation: IndexedOperation
): DriftEvent[] {
  const events: DriftEvent[] = [];

  if (schema.type === 'object' && typeof actual === 'object' && actual !== null) {
    const actualObj = actual as Record<string, unknown>;
    const properties = schema.properties ?? {};
    const required = new Set(schema.required ?? []);

    // Check for missing required fields
    for (const field of required) {
      if (!(field in actualObj)) {
        events.push({
          id: randomUUID(),
          type: 'missing_required_field',
          severity: 'breaking',
          operationId: operation.operationId,
          path: `${operation.httpMethod} ${operation.pathPattern}`,
          jsonPath: `${path}.${field}`,
          message: `Required field '${field}' is missing from the response`,
          expected: 'present',
          actual: 'absent',
          timestamp: new Date().toISOString(),
        });
      }
    }

    // Check for extra fields not in schema
    for (const field of Object.keys(actualObj)) {
      if (!(field in properties) && schema.additionalProperties === false) {
        events.push({
          id: randomUUID(),
          type: 'extra_field',
          severity: 'info',
          operationId: operation.operationId,
          path: `${operation.httpMethod} ${operation.pathPattern}`,
          jsonPath: `${path}.${field}`,
          message: `Field '${field}' is not defined in the spec schema`,
          expected: 'absent',
          actual: typeof actualObj[field],
          timestamp: new Date().toISOString(),
        });
      }
    }

    // Recurse into defined properties
    for (const [field, propSchema] of Object.entries(properties)) {
      if (field in actualObj) {
        events.push(...compareToSchema(
          actualObj[field],
          propSchema as OpenAPIV3_1.SchemaObject,
          `${path}.${field}`,
          operation
        ));
      }
    }
  }

  // Type mismatch check
  if (schema.type && actual !== null && actual !== undefined) {
    const actualType = Array.isArray(actual) ? 'array' : typeof actual;
    const expectedType = schema.type === 'integer' ? 'number' : schema.type;
    if (actualType !== expectedType) {
      events.push({
        id: randomUUID(),
        type: 'type_mismatch',
        severity: 'breaking',
        operationId: operation.operationId,
        path: `${operation.httpMethod} ${operation.pathPattern}`,
        jsonPath: path,
        message: `Expected type '${schema.type}' but received '${actualType}'`,
        expected: schema.type,
        actual: actualType,
        timestamp: new Date().toISOString(),
      });
    }
  }

  return events;
}
```

### Testing

- **Unit: drift-detector** — Compare response with missing required field; verify breaking drift event; compare response with extra field; verify info drift event; compare response with wrong type; verify type_mismatch event
- **Unit: severity classification** — Missing required field is breaking; extra field is info; type mismatch is breaking; undocumented status code is warning
- **Integration: proxy mode** — Start mock server in proxy mode with a test upstream (use a second mock); send requests; verify upstream response is returned to client; verify drift events are logged
- **Integration: drift admin API** — Trigger drift events via proxy; query `/__admin/drift`; verify events appear with correct type, severity, and JSON path
- **Edge cases:** Upstream timeout, upstream returns HTML instead of JSON, upstream returns empty body, circular schema references in drift comparison

---

## Phase 10: Spec Gap Detection & Error Generation

**Goal:** Analyze OpenAPI specs for gaps and auto-generate realistic error responses including OWASP API Top 10 scenarios.

### What

- Detect missing response codes: POST without 409 Conflict, auth-protected endpoints without 401/403, endpoints without 429 rate-limiting
- Detect missing schema properties: objects without `id`, collections without pagination, timestamps without formats
- Generate RFC 7807 error responses for all detected gaps
- Implement OWASP API Top 10 error scenario templates (API1:Broken Object Level Auth through API10:Unsafe API Consumption)
- CLI command: `openapi-mock audit <spec>` — print gap report
- Auto-generate error responses when `--errors` flag is enabled

### Design

**Spec Gap Analyzer:**

```typescript
// packages/core/src/analyzer/spec-gap-analyzer.ts
export interface SpecGap {
  type: GapType;
  severity: 'critical' | 'warning' | 'suggestion';
  operationId?: string;
  path: string;
  description: string;
  suggestedFix?: {
    statusCode: string;
    description: string;
    schema: Record<string, unknown>;
  };
  owaspCategory?: string;
}

export type GapType =
  | 'missing_error_response'
  | 'missing_auth_errors'
  | 'missing_rate_limit'
  | 'missing_pagination'
  | 'missing_validation_errors'
  | 'naming_convention'
  | 'incomplete_schema';

export function analyzeSpec(operations: IndexedOperation[]): SpecGap[] {
  const gaps: SpecGap[] = [];

  for (const op of operations) {
    const codes = new Set(op.responseCodes);

    // POST/PUT/PATCH without 400 (validation error)
    if (['POST', 'PUT', 'PATCH'].includes(op.httpMethod) && op.hasRequestBody) {
      if (!codes.has('400') && !codes.has('422')) {
        gaps.push({
          type: 'missing_validation_errors',
          severity: 'warning',
          operationId: op.operationId,
          path: `${op.httpMethod} ${op.pathPattern}`,
          description: 'Operation accepts a request body but does not define a 400 or 422 validation error response',
          suggestedFix: {
            statusCode: '400',
            description: 'Bad Request - validation error',
            schema: RFC_7807_SCHEMA,
          },
          owaspCategory: 'API8:2023',
        });
      }
    }

    // POST creating resources without 409 (conflict)
    if (op.httpMethod === 'POST' && !op.pathPattern.includes('{')) {
      if (!codes.has('409')) {
        gaps.push({
          type: 'missing_error_response',
          severity: 'suggestion',
          operationId: op.operationId,
          path: `${op.httpMethod} ${op.pathPattern}`,
          description: 'POST operation creating resources should define a 409 Conflict response for duplicate detection',
          suggestedFix: {
            statusCode: '409',
            description: 'Conflict - resource already exists',
            schema: RFC_7807_SCHEMA,
          },
        });
      }
    }

    // Operations without 401/403 (auth errors)
    if (!codes.has('401') && !codes.has('403')) {
      gaps.push({
        type: 'missing_auth_errors',
        severity: 'warning',
        operationId: op.operationId,
        path: `${op.httpMethod} ${op.pathPattern}`,
        description: 'Operation does not define 401 (Unauthorized) or 403 (Forbidden) responses',
        owaspCategory: 'API1:2023',
        suggestedFix: {
          statusCode: '401',
          description: 'Unauthorized - authentication required',
          schema: RFC_7807_SCHEMA,
        },
      });
    }

    // No rate limiting response defined across any operation
    if (!codes.has('429')) {
      gaps.push({
        type: 'missing_rate_limit',
        severity: 'suggestion',
        operationId: op.operationId,
        path: `${op.httpMethod} ${op.pathPattern}`,
        description: 'No 429 (Too Many Requests) response defined; consider adding rate limit documentation',
        owaspCategory: 'API4:2023',
      });
    }
  }

  return gaps;
}

const RFC_7807_SCHEMA = {
  type: 'object',
  properties: {
    type: { type: 'string', format: 'uri' },
    title: { type: 'string' },
    status: { type: 'integer' },
    detail: { type: 'string' },
    instance: { type: 'string', format: 'uri' },
  },
  required: ['type', 'title', 'status'],
};
```

### Testing

- **Unit: spec-gap-analyzer** — Analyze petstore spec; verify detection of missing 401/403, missing 400 on POST, missing 429; verify severity levels are correct
- **Unit: OWASP mapping** — Verify each detected gap maps to the correct OWASP API Top 10 category
- **Integration: audit command** — Run `openapi-mock audit petstore.yaml`; verify structured report lists all gaps with suggested fixes
- **Integration: auto-error generation** — Start mock server with `--errors`; send request that would trigger 401; verify RFC 7807 response with correct structure; verify error response matches OWASP scenario
- **Edge cases:** Spec with all error codes already defined (should report zero gaps); spec with only default response; spec with security schemes defined but no 401/403 responses

---

## Phase 11: OpenAPI 3.1 & JSON Schema 2020-12 Full Support

**Goal:** Complete support for OpenAPI 3.1-specific features and the full JSON Schema 2020-12 vocabulary.

### What

- Support `$dynamicRef` and `$dynamicAnchor` for recursive schema patterns
- Support JSON Schema 2020-12 keywords: `prefixItems`, `$comment`, `dependentRequired`, `dependentSchemas`, `if`/`then`/`else`, `unevaluatedProperties`, `unevaluatedItems`
- Support `const` keyword (fixed value)
- Support `contentMediaType` and `contentEncoding` for binary/base64 fields
- Handle `type` as an array (`"type": ["string", "null"]`) per JSON Schema 2020-12
- Support `webhooks` section of OpenAPI 3.1 for callback simulation
- Support `pathItems` references and reuse
- Ensure backward compatibility with OpenAPI 3.0 and 2.0 specs

### Design

**JSON Schema 2020-12 walker additions:**

```typescript
// Additions to packages/core/src/generator/schema-walker.ts

function walkSchemaV2(
  schema: OpenAPIV3_1.SchemaObject,
  depth: number,
  maxDepth: number
): unknown {
  // Handle const
  if ('const' in schema) return (schema as any).const;

  // Handle type as array (e.g., ["string", "null"])
  if (Array.isArray(schema.type)) {
    const nonNullTypes = schema.type.filter(t => t !== 'null');
    if (nonNullTypes.length === 0) return null;
    // Pick a non-null type to generate
    const chosenType = faker.helpers.arrayElement(nonNullTypes);
    return walkSchema({ ...schema, type: chosenType } as any, depth, maxDepth);
  }

  // Handle if/then/else
  if ('if' in schema) {
    const ifSchema = (schema as any).if;
    const thenSchema = (schema as any).then;
    const elseSchema = (schema as any).else;
    // Default to then branch for mock generation
    if (thenSchema) {
      return walkSchema(
        mergeSchemas(schema, thenSchema) as any,
        depth, maxDepth
      );
    }
  }

  // Handle prefixItems (tuple validation)
  if ('prefixItems' in schema && Array.isArray((schema as any).prefixItems)) {
    const prefixItems = (schema as any).prefixItems as OpenAPIV3_1.SchemaObject[];
    const result = prefixItems.map((item, i) =>
      walkSchema(item, depth + 1, maxDepth)
    );
    // If items also defined, add extra items
    if (schema.items && schema.minItems && schema.minItems > prefixItems.length) {
      const extraCount = schema.minItems - prefixItems.length;
      for (let i = 0; i < extraCount; i++) {
        result.push(walkSchema(schema.items as any, depth + 1, maxDepth));
      }
    }
    return result;
  }

  // Handle dependentRequired
  if ('dependentRequired' in schema && schema.type === 'object') {
    const result = generateObject(schema, depth, maxDepth);
    const depReq = (schema as any).dependentRequired as Record<string, string[]>;
    for (const [field, required] of Object.entries(depReq)) {
      if (field in result) {
        for (const reqField of required) {
          if (!(reqField in result) && schema.properties?.[reqField]) {
            result[reqField] = walkSchema(
              schema.properties[reqField] as any,
              depth + 1,
              maxDepth
            );
          }
        }
      }
    }
    return result;
  }

  // Delegate to base walker for standard types
  return walkSchema(schema, depth, maxDepth);
}
```

### Testing

- **Unit: const keyword** — Schema with `{ "const": "USD" }` always generates `"USD"`
- **Unit: type array** — Schema `{ "type": ["string", "null"] }` generates either a string or null
- **Unit: prefixItems** — Tuple schema `{ "prefixItems": [{ "type": "string" }, { "type": "number" }] }` generates `["somestring", 42]`
- **Unit: dependentRequired** — When `creditCard` field is present, `billingAddress` must also be present
- **Unit: if/then/else** — Schema with conditional generates data matching the `then` branch
- **Integration: full 3.1 spec** — Load a comprehensive OAS 3.1 spec using all new keywords; verify mock server generates valid responses
- **Backward compat** — Verify OpenAPI 2.0 and 3.0 specs still load and serve correctly after 3.1 additions

---

## Phase 12: Performance, Hardening & Release

**Goal:** Optimize performance, add comprehensive error handling, and ship v1.0.

### What

- Benchmark: target < 5ms p99 latency for cached responses, < 50ms for heuristic generation
- Connection pooling and keep-alive for proxy mode
- Response body size limits (configurable, default 1MB)
- Spec size limits and memory usage monitoring
- Structured logging with configurable verbosity (--log-level debug/info/warn/error)
- Comprehensive error handling: spec parse failures, AI API failures, upstream timeouts, OOM protection
- Security: sanitize spec-derived values in error messages, prevent path traversal in file loading
- npm publish workflow: semantic versioning, changelog generation, GitHub Releases
- CI/CD: GitHub Actions pipeline with lint, test, build, Docker build, npm publish on tag

### Design

**Performance benchmarking harness:**

```typescript
// packages/server/src/__benchmarks__/response-generation.bench.ts
import { bench, describe } from 'vitest';
import { walkSchema } from '@openapi-mock/core';
import { heuristicGenerate } from '@openapi-mock/core';

const userSchema = {
  type: 'object' as const,
  required: ['id', 'name', 'email'],
  properties: {
    id: { type: 'string' as const, format: 'uuid' },
    name: { type: 'string' as const },
    email: { type: 'string' as const, format: 'email' },
    age: { type: 'integer' as const, minimum: 18, maximum: 120 },
    address: {
      type: 'object' as const,
      properties: {
        street: { type: 'string' as const },
        city: { type: 'string' as const },
        country: { type: 'string' as const },
      },
    },
  },
};

describe('Response Generation', () => {
  bench('faker walkSchema - simple object', () => {
    walkSchema(userSchema);
  });

  bench('heuristic generate - field name matching', () => {
    heuristicGenerate('email', { type: 'string', format: 'email' });
  });

  bench('faker walkSchema - array of 10 objects', () => {
    walkSchema({
      type: 'array',
      items: userSchema,
      minItems: 10,
      maxItems: 10,
    });
  });
});
```

**GitHub Actions CI:**

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm test -- --coverage
      - run: pnpm build

  docker:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v6
        with:
          context: .
          file: docker/Dockerfile
          push: false
          tags: openapi-mock:test
      - name: Smoke test
        run: |
          docker run -d --name mock -p 4010:4010 \
            -v ${{ github.workspace }}/specs:/spec \
            openapi-mock:test mock --spec /spec/petstore-3.1.yaml
          sleep 3
          curl -sf http://localhost:4010/pets | jq .
          docker stop mock
```

### Testing

- **Benchmark: faker generation** — Verify < 1ms p99 for simple object, < 5ms for array of 10 objects
- **Benchmark: request matching** — Verify < 0.1ms per match against index of 100 operations
- **Benchmark: full request cycle** — Verify < 5ms p99 end-to-end for cached/faker responses
- **Load test** — 1000 concurrent requests using `autocannon`; verify zero dropped connections, < 10ms p99
- **Security: path traversal** — Attempt `--spec ../../etc/passwd`; verify rejection
- **Security: error message sanitization** — Spec with XSS payload in description; verify it is not reflected in error responses without encoding
- **Resilience: large spec** — Load spec with 500+ operations; verify startup time < 10s and memory < 256MB
- **Release: npm pack** — Verify package installs cleanly, `openapi-mock --version` works, all bin entries resolve

---

## Dependency Graph

```
Phase 1: Scaffolding & Spec Parsing
    |
    v
Phase 2: Basic HTTP Mock Server --------+
    |                                    |
    v                                    |
Phase 3: Request Validation             |
    |                                    |
    v                                    |
Phase 4: Heuristic Data Generation      |
    |                                    |
    v                                    |
Phase 5: AI Data Generation             |
    |                                    |
    +----+-------------------------------+
         |
         v
Phase 6: Request Logging & Admin API
         |
         v
Phase 7: Stateful Sessions --------+
         |                         |
         v                         |
Phase 8: CLI Polish & Docker       |
         |                         |
         +--------+----------------+
                  |
                  v
Phase 9: Proxy & Drift Detection
                  |
                  v
Phase 10: Spec Gap & Error Gen
                  |
                  v
Phase 11: OAS 3.1 Full Support
                  |
                  v
Phase 12: Performance & Release
```

**Parallelization opportunities:**
- Phases 3 and 4 can be developed in parallel (validation and heuristic generation are independent)
- Phase 6 can start alongside Phase 5 (logging does not depend on AI generation)
- Phases 9 and 10 can be developed in parallel (proxy mode and spec gap analysis are independent)
- Phase 11 can be started anytime after Phase 4 (schema walker extensions)

---

## Definition of Done

### Per-Phase Completion Criteria

Each phase is considered complete when ALL of the following are satisfied:

1. **Code complete** — All tasks in the "What" section are implemented
2. **Tests passing** — All tests described in the "Testing" section are implemented and pass with >= 90% line coverage for new code
3. **Lint clean** — Zero ESLint errors and zero TypeScript compiler errors
4. **Documentation** — Inline JSDoc on all exported functions; README updated with any new CLI flags or configuration options
5. **PR reviewed** — Code reviewed by at least one other developer (or self-review with documented checklist for solo development)
6. **No regressions** — Full test suite passes; no existing tests broken

### v1.0 Release Criteria (after Phase 12)

| Criterion | Target |
|-----------|--------|
| Test coverage (line) | >= 85% across all packages |
| Test coverage (branch) | >= 75% across all packages |
| Benchmark: p99 latency (cached response) | < 5ms |
| Benchmark: p99 latency (heuristic generation) | < 50ms |
| Benchmark: spec load time (500 operations) | < 10s |
| Benchmark: memory (500 operations, idle) | < 256MB |
| Docker image size | < 200MB |
| npm install time (clean) | < 30s |
| OpenAPI 2.0 specs | Load and serve correctly |
| OpenAPI 3.0 specs | Load and serve correctly |
| OpenAPI 3.1 specs | Load and serve correctly with JSON Schema 2020-12 |
| Node.js 20 LTS | All tests pass |
| Node.js 22 LTS | All tests pass |
| CLI --help | All commands documented |
| Zero critical/high CVEs | npm audit clean |
| MIT licence | LICENCE file present; all dependencies MIT/Apache/BSD compatible |
