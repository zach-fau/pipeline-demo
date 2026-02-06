---
description: Generate mapping index files (ai-docs-map.json and ai-service-map.json) from documentation and source code
---

# Generate Maps

You are a mapping file generator. Your job is to create two JSON index files that allow the research agent's sub-agents to quickly locate relevant documentation and source files without scanning the entire codebase.

## Step 1: Read Configuration

Read `ai-config.json` from the current working directory. If it doesn't exist, inform the user to run `/generate-docs` first (which creates the config).

Extract:
- `documentationOutputDir` (default: `./documentation`)
- `serviceMapPath` (default: `./ai-service-map.json`)
- `documentationMapPath` (default: `./ai-docs-map.json`)

## Step 2: Build `ai-docs-map.json`

Scan all YAML files in the documentation directory. For each service that has documentation files, create an entry:

```json
{
  "services": {
    "Auth": {
      "openapi": "documentation/openapi/Auth.yml",
      "internalFunctions": "documentation/internalfunctions/Auth.yml",
      "events": "documentation/events/Auth.yml",
      "description": "Handles user authentication, login, registration, and session management"
    },
    "MenuItems": {
      "openapi": "documentation/openapi/MenuItems.yml",
      "internalFunctions": "documentation/internalfunctions/MenuItems.yml",
      "events": "documentation/events/MenuItems.yml",
      "description": "Manages cafe menu items, categories, and availability"
    }
  },
  "keywords": {
    "login": ["Auth"],
    "register": ["Auth"],
    "menu": ["MenuItems"],
    "order": ["Orders"],
    "payment": ["Orders", "Payments"],
    "firebase": ["Auth", "Firestore"],
    "notification": ["Notifications"]
  }
}
```

To build this:
1. List all `.yml` files in `documentation/openapi/`, `documentation/internalfunctions/`, and `documentation/events/`.
2. Group by service name (the filename without extension).
3. Read each YAML file to extract a description for the service. For the `description` field, synthesize a one-sentence summary from the content of the service's documentation files.
4. Build the `keywords` map by extracting significant terms from:
   - Service names (lowercased)
   - Function names from internalfunctions YAML
   - Endpoint paths from openapi YAML
   - Event names from events YAML
   - Description text

Write the result to the path specified in `documentationMapPath`.

## Step 3: Build `ai-service-map.json`

Scan the actual source code to build a comprehensive index of routes, actions, and source files:

```json
{
  "routes": {
    "GET /api/auth/login": {
      "service": "Auth",
      "action": "auth.login",
      "sourceFile": "src/services/auth.ts",
      "description": "Handle user login"
    },
    "POST /api/orders": {
      "service": "Orders",
      "action": "orders.create",
      "sourceFile": "src/services/orders.ts",
      "description": "Create a new order"
    }
  },
  "actions": {
    "auth.login": {
      "service": "Auth",
      "sourceFile": "src/services/auth.ts",
      "method": "login",
      "line": 25,
      "description": "Validates credentials and returns session token"
    }
  },
  "sourceFiles": {
    "src/services/auth.ts": {
      "service": "Auth",
      "type": "service",
      "exports": ["login", "register", "logout"],
      "dependencies": ["src/services/firebase.ts", "src/utils/hash.ts"]
    },
    "src/components/LoginScreen.tsx": {
      "service": "Auth",
      "type": "component",
      "exports": ["LoginScreen"],
      "dependencies": ["src/services/auth.ts", "src/hooks/useAuth.ts"]
    }
  },
  "screens": {
    "LoginScreen": {
      "service": "Auth",
      "sourceFile": "src/components/LoginScreen.tsx",
      "description": "User login interface"
    }
  }
}
```

To build this:

1. **Routes:** Scan for route definitions — Express `.get()/.post()`, API route files, navigation routes, etc. Record the HTTP method + path, which service it belongs to, and the handler.

2. **Actions:** For each significant function/method in service files, create an action entry with `serviceName.methodName` as the key. Include the file path, method name, line number, and description.

3. **Source files:** For every source file in the project (excluding `node_modules`, `dist`, `build`, `.git`, `documentation/`), record:
   - Which service it belongs to (inferred from directory, imports, or documentation)
   - Its type: `service`, `component`, `hook`, `utility`, `config`, `test`, `type`
   - What it exports
   - What other project files it imports (dependencies)

4. **Screens/Pages:** If the project has UI components (React, Vue, etc.), index screens/pages separately for quick lookup.

Write the result to the path specified in `serviceMapPath`.

## Step 4: Cross-Reference and Validate

After building both maps:

1. Check that every service mentioned in `ai-service-map.json` has corresponding documentation in `ai-docs-map.json`. Report any gaps.
2. Check that every service in `ai-docs-map.json` has source files in `ai-service-map.json`. Report orphaned docs.
3. Look for source files that aren't assigned to any service and flag them.

## Step 5: Report Results

Output a summary:

```
Maps generated successfully:

ai-docs-map.json:
  - N services documented
  - M keywords indexed

ai-service-map.json:
  - R routes indexed
  - A actions indexed
  - F source files indexed
  - S screens indexed

Gaps found:
  - [ServiceName] has docs but no source files mapped (or vice versa)
  - N source files not assigned to any service

Run /research "your question" to query the codebase.
```

## Important Guidelines

- **Be thorough with source files:** Index every meaningful source file. The more complete the map, the better the research agent works.
- **Use relative paths:** All paths should be relative to the project root.
- **Keep descriptions concise:** One sentence per entry. These are for quick scanning, not detailed documentation.
- **Handle large codebases:** If there are hundreds of files, still index them all. The JSON files can be large — that's fine.
- **Don't read node_modules:** Skip `node_modules/`, `dist/`, `build/`, `.git/`, and other generated directories.
- **Overwrite existing maps:** Always regenerate from scratch to ensure accuracy.
