---
description: Generate structured YAML documentation for all services in the current codebase (or a single service if specified)
---

# Generate Documentation

You are a documentation generator. Your job is to analyze the current codebase and produce structured YAML documentation files for each service/module you discover.

## Input

The developer may optionally specify a service name: `$ARGUMENTS`

- If a service name is provided, only generate/regenerate docs for that specific service.
- If no service name is provided, scan the entire codebase and generate docs for all discovered services.

## Step 1: Read or Create `ai-config.json`

Check if `ai-config.json` exists in the current working directory.

- If it exists, read it to get configuration paths.
- If it does NOT exist, create it with this default structure:

```json
{
  "serviceMapping": {
    "targetService": "<inferred-project-name>",
    "basePath": ".",
    "serviceMapPath": "./ai-service-map.json",
    "documentationMapPath": "./ai-docs-map.json",
    "documentationOutputDir": "./documentation"
  }
}
```

Infer `targetService` from the nearest `package.json` name field, the directory name, or any other project identifier you can find.

## Step 2: Discover Services

Scan the codebase to identify distinct services/modules. Use these heuristics:

1. **Directory-based:** Look for directories under `src/services/`, `src/modules/`, `src/api/`, `services/`, `lib/`, `functions/`, or similar patterns. Each directory or major file often represents a service.
2. **Route-based:** Look for route definitions (Express routers, API route files, Next.js API routes, etc.). Group routes by their prefix or file.
3. **Class/module-based:** Look for classes, major exported modules, or files with clear domain boundaries (e.g., `auth.ts`, `ratings.ts`, `orders.ts`).
4. **Config-based:** Check for microservice configs, serverless function definitions, or similar.

For each discovered service, note:
- Service name (PascalCase, e.g., "Auth", "MenuItems", "ZSociety")
- The source files that belong to it
- What it does (brief description)

If `$ARGUMENTS` specifies a single service name, only analyze that service's files.

## Step 3: Generate Documentation Files

For each service, create three YAML files in the documentation output directory (from `ai-config.json`, default `./documentation/`).

Create the directory structure:
```
documentation/
├── openapi/
├── internalfunctions/
└── events/
```

### 3a. OpenAPI spec: `documentation/openapi/[ServiceName].yml`

Only create this file if the service exposes API endpoints (HTTP routes, REST endpoints, GraphQL resolvers, etc.). Skip this file for pure internal/utility services.

```yaml
openapi: "3.0.0"
info:
  title: "[ServiceName] API"
  description: "API endpoints for the [ServiceName] service"
paths:
  /actual/path:
    get:
      summary: "What this endpoint does"
      description: "Detailed description of behavior"
      parameters:
        - name: "paramName"
          in: "query"
          required: false
          schema:
            type: "string"
          description: "What this parameter does"
      requestBody:
        content:
          application/json:
            schema:
              type: "object"
              properties:
                field:
                  type: "string"
                  description: "What this field is"
      responses:
        "200":
          description: "Success response"
          content:
            application/json:
              schema:
                type: "object"
                properties:
                  result:
                    type: "string"
```

Populate paths, parameters, request bodies, and responses by reading the actual source code. Be accurate — include real path parameters, query params, and response shapes.

### 3b. Internal functions: `documentation/internalfunctions/[ServiceName].yml`

Document all significant exported functions, class methods, and helpers:

```yaml
service: "[ServiceName]"
functions:
  - name: "functionName"
    file: "relative/path/to/file.ts"
    line: 42
    description: "What this function does in plain English"
    parameters:
      - name: "param1"
        type: "string"
        description: "What this parameter is"
      - name: "param2"
        type: "number"
        description: "What this parameter is"
    returns:
      type: "Promise<Result>"
      description: "What the return value represents"
    dependencies:
      - "otherFunction"
      - "ExternalService.method"
    sideEffects:
      - "Writes to Firestore collection 'users'"
      - "Sends push notification"
```

Include:
- The actual line number where the function is defined
- Real parameter types from the source code
- Dependencies on other functions or services
- Any side effects (database writes, network calls, event emissions)

Skip trivial getters/setters and simple one-line utilities unless they are part of the public API.

### 3c. Events: `documentation/events/[ServiceName].yml`

Document events, callbacks, listeners, and pub/sub patterns:

```yaml
service: "[ServiceName]"
events:
  emitted:
    - name: "event.name"
      description: "When and why this event fires"
      payload:
        field: "type — description"
      emittedFrom: "relative/path/to/file.ts:functionName"
  consumed:
    - name: "event.name"
      handler: "relative/path/to/handler.ts:handlerFunction"
      description: "What happens when this event is received"
```

This includes:
- Custom events (EventEmitter, pub/sub, message queues)
- State change callbacks (React context updates, Redux actions, Zustand store updates)
- Firebase/Firestore listeners (onSnapshot, onAuthStateChanged, etc.)
- Navigation events
- Any other reactive/event-driven patterns

If the service has no events, still create the file with empty `emitted: []` and `consumed: []` arrays.

## Step 4: Report Results

After generating all files, output a summary:

```
Documentation generated for N services:

- Auth
  - documentation/openapi/Auth.yml
  - documentation/internalfunctions/Auth.yml
  - documentation/events/Auth.yml

- MenuItems
  - documentation/openapi/MenuItems.yml
  - documentation/internalfunctions/MenuItems.yml
  - documentation/events/MenuItems.yml

[etc.]

Run /generate-maps to create index files for the research agent.
```

## Important Guidelines

- **Be accurate:** Read the actual source code. Do not guess or hallucinate endpoints, parameters, or function signatures.
- **Be thorough:** Document all significant services. A service with 1 function is still worth documenting.
- **Use relative paths:** All file paths in the YAML should be relative to the project root.
- **Preserve existing docs:** If documentation files already exist and you're regenerating for a specific service, only overwrite that service's files. Do not touch other services' docs.
- **Handle any language/framework:** This system should work for TypeScript, JavaScript, Python, Go, Rust, Java, etc. Adapt the discovery heuristics to whatever you find.
