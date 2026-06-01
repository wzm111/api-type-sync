---
name: api-type-sync
description: Frontend-backend API type synchronization assistant. Reads OpenAPI/Swagger definitions to auto-generate/update frontend TypeScript types, detect mismatches between frontend calls and API definitions, generate API client code (axios/fetch/tanstack-query), detect API change diffs, and generate mock data from schemas. Activates on /api-type-sync or when swagger.json / openapi.yaml / openapi.json / api-spec/ directory is detected.
user-invocable: true
---

# API Type Sync / 前后端接口类型同步助手

## Purpose
Bridge the gap between backend API definitions and frontend TypeScript code. Automatically sync types, detect breaking changes, generate API clients, and produce mock data — all driven by OpenAPI/Swagger specifications.

## When to Activate
- User types `/api-type-sync`
- User asks for "sync API types", "generate types from swagger", "check API mismatch"
- `swagger.json`, `openapi.yaml`, `openapi.json`, or `swagger.yaml` detected in project
- `api-spec/`, `openapi/`, `swagger/`, `docs/api/` directories detected
- Backend controller/DTO files detected alongside frontend API call code
- User asks about "API client", "tanstack query generator", "mock API data"

## Slash Command Parameter Mapping / 斜杠命令参数映射

Claude Code slash commands don't accept CLI-style arguments. Parse user intent and route to the appropriate handler. Never tell the user to type bash commands manually.

| User Says / 用户说 | Maps To / 映射为 | Action |
|---|---|---|
| `/api-type-sync` | 扫描 + 全量同步 | 自动发现 spec 文件，生成/更新所有类型和 client |
| `/api-type-sync --generate` / "generate types" | 类型生成 | 从 OpenAPI spec 生成 TypeScript 类型定义 |
| `/api-type-sync --generate --client axios` | 类型 + axios client | 生成类型 + axios-based API client |
| `/api-type-sync --generate --client fetch` | 类型 + fetch client | 生成类型 + fetch-based API client |
| `/api-type-sync --generate --client tanstack` | 类型 + tanstack-query hooks | 生成类型 + TanStack Query hooks |
| `/api-type-sync --check` / "check API mismatch" | 不匹配检查 | 对比前端调用代码与 spec，报告差异 |
| `/api-type-sync --diff` / "show API changes" | 变更 diff | 检测 spec 变更，列出受影响的前端代码 |
| `/api-type-sync --mock` / "generate mock data" | Mock 生成 | 基于 schema 生成 faker/msw mock 数据 |
| `/api-type-sync --mock --format msw` | MSW handlers | 生成 MSW (Mock Service Worker) handlers |
| `/api-type-sync --mock --format faker` | Faker 数据 | 生成纯 faker 假数据工厂函数 |
| `/api-type-sync --all` / "do everything" | 全量工作流 | 生成类型 + client + check + mock |
| `/api-type-sync --watch` / "watch for changes" | 监听模式 | 监听 spec 文件变化，自动重新生成 |
| `/api-type-sync --output ./src/api` | 自定义输出目录 | 指定生成文件的输出路径 |
| `/api-type-sync --spec ./backend/openapi.yaml` | 自定义 spec 路径 | 指定 OpenAPI spec 文件位置 |

**Implementation rule:** When user invokes `/api-type-sync` followed by descriptive text, parse flags and intent, then execute the corresponding workflow via internal handlers. Never expose `bash scripts/xxx.sh` to the user.

## Spec File Discovery / Spec 文件自动发现

Search for OpenAPI/Swagger definitions in this order:

1. **Explicit `--spec` path** — use if provided
2. **Common filenames** (project root and 1 level deep):
   - `swagger.json`, `swagger.yaml`, `swagger.yml`
   - `openapi.json`, `openapi.yaml`, `openapi.yml`
   - `api.json`, `api.yaml`, `api.yml`
3. **Common directories** (recursive, max depth 2):
   - `api-spec/`, `openapi/`, `swagger/`, `docs/api/`, `spec/`, `api-docs/`
4. **Backend framework hints**:
   - NestJS: `src/main.ts` (look for `SwaggerModule.setup`)
   - Spring Boot: `*Controller.java` + `*Dto.java` / `*Request.java`
   - FastAPI: Python files with `@app.get/post` decorators
   - Go: `docs/` from swaggo

If multiple specs found, list them and ask user to pick one (or process all if `--all-specs` implied).

## Workflow Stages / 工作流阶段

### Stage 1: Spec Analysis / 解析 Spec

Parse the discovered OpenAPI/Swagger spec and extract:

- **Paths & Operations**: `GET /users`, `POST /orders/{id}`, etc.
- **Request Parameters**: path params, query params, headers, body schema
- **Response Schemas**: 200, 201, 400, 401, 403, 404, 500 — with full type definitions
- **Components/Definitions**: reusable schemas (User, Order, Pagination, etc.)
- **Enums**: string/number enums with descriptions
- **Security Schemes**: bearer auth, API key, OAuth2

**Validation:**
- Verify spec is valid OpenAPI 2.0/3.0/3.1
- Report missing `operationId` (needed for client generation)
- Report missing response schemas (returns `any`)
- Report circular references (common in complex DTOs)

### Stage 2: Type Generation / 类型生成

Generate TypeScript type definitions from spec schemas:

**Output structure:**
```typescript
// types/api.types.ts 或按 domain 拆分

// 1. Schema Types (from components/schemas)
export interface User {
  id: string;
  name: string;
  email: string;
  role: UserRole;  // enum
  createdAt: string; // ISO date
}

export type UserRole = 'admin' | 'user' | 'guest';

// 2. Request Types (path/query/body parameters)
export interface GetUsersRequest {
  page?: number;
  pageSize?: number;
  role?: UserRole;
}

export interface CreateUserRequest {
  body: {
    name: string;
    email: string;
    role?: UserRole;
  };
}

// 3. Response Types (per status code)
export interface GetUsersResponse {
  200: {
    data: User[];
    pagination: Pagination;
  };
  400: ValidationError;
  401: UnauthorizedError;
}

// 4. Path Parameter Types
export interface GetUserByIdPathParams {
  id: string;
}
```

**Naming conventions:**
- PascalCase for interfaces/types
- Suffix `Request` for request bodies, `Response` for responses
- Suffix `PathParams`, `QueryParams` for parameter objects
- Enum names: PascalCase, values: preserve from spec
- Optional fields: `?` for non-required properties
- Dates: `string` (ISO 8601) by default; add `Date` override comment if known

**File organization strategies (auto-detect or ask):**
- `single`: one `api.types.ts` file
- `by-tag`: `user.types.ts`, `order.types.ts` per OpenAPI tag
- `by-domain`: group by path prefix `/users/` → `user/`

### Stage 3: Client Generation / Client 代码生成

Generate ready-to-use API client code based on `--client` flag.

#### Axios Client
```typescript
// api/client.ts
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';
import type { GetUsersRequest, GetUsersResponse, /* ... */ } from './types';

export class ApiClient {
  private client: AxiosInstance;

  constructor(baseURL: string, config?: AxiosRequestConfig) {
    this.client = axios.create({ baseURL, ...config });
  }

  async getUsers(params?: GetUsersRequest): Promise<GetUsersResponse[200]> {
    const { data } = await this.client.get('/users', { params });
    return data;
  }

  async getUserById(pathParams: GetUserByIdPathParams): Promise<GetUserByIdResponse[200]> {
    const { data } = await this.client.get(`/users/${pathParams.id}`);
    return data;
  }

  async createUser(body: CreateUserRequest['body']): Promise<CreateUserResponse[201]> {
    const { data } = await this.client.post('/users', body);
    return data;
  }
}
```

#### Fetch Client
```typescript
// api/fetch-client.ts
export class FetchApiClient {
  constructor(private baseURL: string) {}

  async getUsers(params?: GetUsersRequest): Promise<GetUsersResponse[200]> {
    const url = new URL('/users', this.baseURL);
    if (params) Object.entries(params).forEach(([k, v]) => v != null && url.searchParams.set(k, String(v)));
    const res = await fetch(url);
    if (!res.ok) throw new ApiError(res.status, await res.text());
    return res.json();
  }
}
```

#### TanStack Query (React Query) Hooks
```typescript
// api/hooks.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { apiClient } from './client';

export function useUsers(params?: GetUsersRequest) {
  return useQuery({
    queryKey: ['users', params],
    queryFn: () => apiClient.getUsers(params),
  });
}

export function useUser(id: string) {
  return useQuery({
    queryKey: ['user', id],
    queryFn: () => apiClient.getUserById({ id }),
    enabled: !!id,
  });
}

export function useCreateUser() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (body: CreateUserRequest['body']) => apiClient.createUser(body),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['users'] }),
  });
}
```

### Stage 4: Mismatch Check / 不匹配检查

Scan frontend code for API calls and compare against the spec:

**What to check:**
1. **Missing fields in request body**: frontend sends `{name, email}` but spec requires `{name, email, role}`
2. **Extra fields**: frontend sends fields not in spec (may cause 400)
3. **Type mismatches**: frontend sends `number` where spec expects `string`
4. **Missing required query params**: frontend omits required `?page`
5. **Wrong HTTP method**: frontend uses `GET` where spec defines `POST`
6. **Path param mismatches**: wrong param name or missing in URL template
7. **Response handling**: frontend destructures fields removed from spec
8. **Missing error handling**: spec defines 401/403 but frontend only handles 200

**Detection patterns:**
```typescript
// Detect these call patterns:
axios.get('/users', { params: {...} })
axios.post('/users', {...body...})
fetch('/api/users')
useQuery({ queryKey: ['users'], queryFn: ... })
useMutation({ mutationFn: (data) => api.createUser(data) })
```

**Output format:**
```
## ⚠️ API Mismatch Report

### 🔴 Breaking Mismatches (Will cause runtime errors)
1. [src/api/user.ts:42] POST /users
   - Missing required body field: `role` (spec: string, enum: ['admin','user'])
   - Current body: { name, email }
   - Fix: Add `role` field

2. [src/pages/UserList.vue:88] GET /users
   - Missing required query param: `pageSize` (spec: required)
   - Current params: { page: 1 }
   - Fix: Add `pageSize: 20`

### 🟡 Type Mismatches (Potential bugs)
3. [src/api/order.ts:15] POST /orders
   - Field `amount`: frontend sends `string`, spec expects `number`
   - Fix: Parse to number before sending

### 🟢 Warnings
4. [src/api/user.ts:55] GET /users/{id}
   - Response field `avatarUrl` removed from spec v2.1.0
   - Frontend still references it — verify if backend still returns it
```

### Stage 5: Change Diff / 变更检测

When spec has changed (compare with previously-generated types or cached spec):

```
## 📋 API Change Diff

### Breaking Changes 🔴
- `POST /users` — body field `phone` removed
- `GET /users/{id}` — response field `profile.bio` type changed: string → object
- `User.role` enum value `'moderator'` removed

### Additions ✅
- New endpoint: `PATCH /users/{id}/status`
- New field: `User.avatarUrl?: string`
- New enum value: `OrderStatus.shipped`

### Modifications 🟡
- `GET /orders` — query param `status` now required (was optional)
- `Order.total` — type: number → string (currency formatting moved to backend)

### Affected Frontend Code
- [src/pages/UserProfile.tsx:23] uses removed field `phone`
- [src/components/OrderCard.tsx:45] assumes `total` is number (may break)
```

### Stage 6: Mock Generation / Mock 数据生成

Generate realistic mock data and request handlers.

#### Faker-based Factory Functions
```typescript
// mocks/factories.ts
import { faker } from '@faker-js/faker';
import type { User, Order } from '../types';

export function createMockUser(override?: Partial<User>): User {
  return {
    id: faker.string.uuid(),
    name: faker.person.fullName(),
    email: faker.internet.email(),
    role: faker.helpers.arrayElement(['admin', 'user', 'guest']),
    createdAt: faker.date.past().toISOString(),
    ...override,
  };
}

export function createMockUserList(count = 10): User[] {
  return Array.from({ length: count }, () => createMockUser());
}
```

#### MSW Handlers
```typescript
// mocks/handlers.ts
import { http, HttpResponse } from 'msw';
import { createMockUser, createMockUserList } from './factories';

export const handlers = [
  http.get('/api/users', ({ request }) => {
    const url = new URL(request.url);
    const page = Number(url.searchParams.get('page')) || 1;
    const pageSize = Number(url.searchParams.get('pageSize')) || 20;
    const users = createMockUserList(pageSize);
    return HttpResponse.json({
      data: users,
      pagination: { page, pageSize, total: 100 },
    });
  }),

  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json(createMockUser({ id: params.id as string }));
  }),

  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json(createMockUser(body as Partial<User>), { status: 201 });
  }),
];
```

## Output Directory Structure / 输出目录结构

```
src/api/                          # 默认输出目录 (可自定义 --output)
├── types/
│   ├── index.ts                  # 统一导出
│   ├── api.types.ts              # 单文件模式
│   └── [domain]/                 # 按 domain 拆分模式
│       ├── user.types.ts
│       └── order.types.ts
├── client.ts                     # API client (axios/fetch)
├── hooks.ts                      # TanStack Query hooks
├── fetch-client.ts               # fetch-based client
├── index.ts                      # 统一导出所有模块
└── mocks/                        # 仅 --mock 时生成
    ├── factories.ts
    ├── handlers.ts
    └── browser.ts                # MSW browser setup
```

## Rules

1. **Always validate the spec first** — invalid OpenAPI produces garbage types
2. **Preserve existing code** — merge new types with existing, don't overwrite custom extensions
3. **Use strict TypeScript** — avoid `any`, use `unknown` with type guards where necessary
4. **Respect `operationId`** — use it for function/hook names; fallback to method+path pattern
5. **Handle all status codes** — generate union response types or separate error types
6. **Circular references** — use `type` instead of `interface` for circularly-referenced schemas
7. **Enum strategy** — prefer `as const` objects over TypeScript enums for tree-shaking
8. **Date handling** — default to `string` (ISO); add `Date` type option via `--dates-as-date`
9. **Nullable fields** — respect OpenAPI `nullable: true` as `type | null`
10. **Optional chaining** — mark fields without `required` as optional (`?`)
11. **Discriminators** — handle `oneOf` + `discriminator` as discriminated unions
12. **Security headers** — include auth header types in client configuration

## Tech Stack Detection / 技术栈检测

| 检测特征 | 推断 Stack | Client 默认 |
|---------|-----------|------------|
| `package.json` 有 `axios` | axios | axios client |
| `package.json` 有 `@tanstack/react-query` | tanstack-query | hooks + axios |
| `package.json` 有 `vue` + `@tanstack/vue-query` | vue-query | vue composables |
| `package.json` 有 `msw` | MSW | msw handlers |
| `package.json` 无 axios 有 fetch | fetch API | fetch client |
| `vite.config.*` + React | Vite + React | 上述规则 |
| `next.config.*` | Next.js | hooks + fetch (优先原生) |
| `nuxt.config.*` | Nuxt/Vue | vue-query + fetch |

## Implementation Strategy

Since this skill operates without external codegen tools (like `openapi-generator`), the implementation should:

1. **Read the spec file** using the Read tool
2. **Parse JSON/YAML** inline or use available parsers
3. **Generate TypeScript** by traversing schemas and paths
4. **Write output files** using the Write/Edit tool
5. **Scan frontend code** using grep/find to locate API calls
6. **Compare and report** mismatches in markdown format

For complex specs, suggest installing dedicated tools:
- `openapi-typescript` (recommended, dev dependency)
- `@hey-api/openapi-ts`
- `orval` (for tanstack-query)

But the skill itself should work out-of-the-box for basic to moderate specs.

## Example Session

### User input
```
/api-type-sync --generate --client tanstack --output ./src/api
```

### Skill workflow
1. Discover `backend/openapi.yaml`
2. Parse schemas: User, Order, Pagination, etc.
3. Generate `./src/api/types/index.ts`
4. Generate `./src/api/client.ts` (axios base)
5. Generate `./src/api/hooks.ts` (TanStack Query)
6. Generate `./src/api/index.ts` (exports)
7. Report:
```
✅ Generated 12 schema types, 8 request types, 8 response types
✅ Generated 8 TanStack Query hooks
⚠️ 3 operations missing operationId (using fallback names)
📁 Output: ./src/api/
```

### User input
```
/api-type-sync --check
```

### Skill workflow
1. Load spec and previously generated types
2. Scan `src/` for API call patterns
3. Compare each call against spec
4. Report mismatches with file:line references
