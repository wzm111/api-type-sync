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
| `/api-type-sync --zod` / "generate zod schemas" | Zod 校验生成 | 从 spec 生成 Zod 运行时校验 schema |
| `/api-type-sync --valibot` / "generate valibot schemas" | Valibot 校验生成 | 从 spec 生成 Valibot 运行时校验 schema |
| `/api-type-sync --pagination` / "add pagination helpers" | 分页抽象 | 生成分页 hooks / composables |
| `/api-type-sync --interceptors` / "add auth interceptors" | 拦截器模板 | 生成统一错误处理、token 刷新拦截器 |
| `/api-type-sync --camel-case` / "convert case" | 命名转换 | snake_case ↔ camelCase 自动转换 |
| `/api-type-sync --file-upload` / "handle file uploads" | 文件上传/下载 | 生成 multipart / blob 处理方法 |
| `/api-type-sync --dead-api` / "find dead APIs" | 死代码检测 | 找出 spec 定义但前端未调用的接口 |
| `/api-type-sync --test` / "generate API tests" | 测试生成 | 生成 vitest/jest API 单元测试骨架 |
| `/api-type-sync --jsdoc` / "add JSDoc comments" | JSDoc 文档 | 为类型添加字段描述注释 |
| `/api-type-sync --all` | 全量工作流 | 生成类型 + client + zod + interceptors + test + mock |

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

---

### Stage 7: Runtime Validation / 运行时校验生成

Compile-time TypeScript types cannot catch runtime data anomalies (e.g., backend returns a string where a number is expected after a deployment). Generate Zod or Valibot schemas from OpenAPI definitions for runtime validation.

#### Zod Schemas
```typescript
// validations/zodSchemas.ts
import { z } from 'zod';

export const UserRoleSchema = z.enum(['admin', 'user', 'guest']);

export const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
  role: UserRoleSchema,
  createdAt: z.string().datetime(),
  avatarUrl: z.string().url().optional(),
});

export const PaginationSchema = z.object({
  page: z.number().int().positive(),
  pageSize: z.number().int().positive().max(100),
  total: z.number().int().nonnegative(),
});

export const GetUsersResponseSchema = z.object({
  data: z.array(UserSchema),
  pagination: PaginationSchema,
});

// Type inference
export type User = z.infer<typeof UserSchema>;
```

**Integration with API client:**
```typescript
// client.ts
import { UserSchema, GetUsersResponseSchema } from './validations/zodSchemas';

async function getUsers() {
  const { data } = await axios.get('/users');
  return GetUsersResponseSchema.parse(data); // throws on mismatch
}
```

#### Valibot Schemas (lightweight alternative)
```typescript
// validations/valibotSchemas.ts
import * as v from 'valibot';

export const UserSchema = v.object({
  id: v.string(),
  name: v.pipe(v.string(), v.minLength(1), v.maxLength(100)),
  email: v.pipe(v.string(), v.email()),
  role: v.picklist(['admin', 'user', 'guest']),
  createdAt: v.isoTimestamp(),
});
```

**Auto-detection:** If `zod` is in `package.json`, default to `--zod`. If `valibot` is present, default to `--valibot`.

---

### Stage 8: Pagination Abstraction / 分页抽象

80% of list endpoints are paginated. Auto-detect pagination patterns from the spec and generate unified pagination helpers.

**Detection patterns:**
| Pattern | Query Params | Response Shape |
|---------|-------------|----------------|
| Offset/Limit | `offset`, `limit` | `{ data, total }` |
| Page/PageSize | `page`, `pageSize`/`perPage` | `{ data, pagination: {page,pageSize,total} }` |
| Cursor-based | `cursor`, `limit` | `{ data, nextCursor, hasMore }` |

#### TanStack Query + Pagination
```typescript
// hooks/usePaginatedQuery.ts
import { useInfiniteQuery, useQuery } from '@tanstack/react-query';

// Offset/Limit style
export function useUsersInfinite(pageSize = 20) {
  return useInfiniteQuery({
    queryKey: ['users', 'infinite'],
    queryFn: ({ pageParam = 0 }) =>
      apiClient.getUsers({ offset: pageParam, limit: pageSize }),
    getNextPageParam: (lastPage, allPages) =>
      lastPage.data.length < pageSize ? undefined : allPages.length * pageSize,
    initialPageParam: 0,
  });
}

// Cursor-based style
export function useUsersCursor(limit = 20) {
  return useInfiniteQuery({
    queryKey: ['users', 'cursor'],
    queryFn: ({ pageParam }) =>
      apiClient.getUsers({ cursor: pageParam, limit }),
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    initialPageParam: undefined as string | undefined,
  });
}
```

#### Vue Composables (Vue Query)
```typescript
// composables/usePaginatedUsers.ts
import { useInfiniteQuery } from '@tanstack/vue-query';

export function usePaginatedUsers(pageSize = 20) {
  return useInfiniteQuery({
    queryKey: ['users'],
    queryFn: ({ pageParam = 1 }) =>
      apiClient.getUsers({ page: pageParam, pageSize }),
    getNextPageParam: (lastPage, allPages) =>
      lastPage.pagination.total > allPages.length * pageSize
        ? allPages.length + 1
        : undefined,
    initialPageParam: 1,
  });
}
```

---

### Stage 9: Interceptors & Error Handling / 拦截器与错误处理

Generate a base client with common interceptor patterns that every project needs.

```typescript
// client/base-client.ts
import axios, { AxiosError, AxiosInstance, InternalAxiosRequestConfig } from 'axios';

export interface ApiError {
  status: number;
  code: string;
  message: string;
  details?: Record<string, string[]>;
}

export class ApiException extends Error {
  constructor(public error: ApiError) {
    super(error.message);
    this.name = 'ApiException';
  }
}

export function createApiClient(baseURL: string): AxiosInstance {
  const client = axios.create({
    baseURL,
    timeout: 30000,
    headers: { 'Content-Type': 'application/json' },
  });

  // Request interceptor: attach auth token
  client.interceptors.request.use((config: InternalAxiosRequestConfig) => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers.set('Authorization', `Bearer ${token}`);
    }
    return config;
  });

  // Response interceptor: global error handling
  client.interceptors.response.use(
    (response) => response,
    (error: AxiosError<ApiError>) => {
      if (error.response) {
        const { status, data } = error.response;

        switch (status) {
          case 401:
            // Token expired → refresh or redirect to login
            localStorage.removeItem('accessToken');
            window.location.href = '/login';
            break;
          case 403:
            console.error('Permission denied:', data?.message);
            break;
          case 404:
            console.error('Resource not found:', data?.message);
            break;
          case 422:
            console.error('Validation failed:', data?.details);
            break;
          case 500:
            console.error('Server error:', data?.message);
            break;
        }

        return Promise.reject(new ApiException(data ?? { status, code: 'UNKNOWN', message: 'Unknown error' }));
      }

      // Network errors
      if (error.request) {
        return Promise.reject(new ApiException({ status: 0, code: 'NETWORK_ERROR', message: 'Network error' }));
      }

      return Promise.reject(error);
    }
  );

  return client;
}
```

**Optional: Token refresh with retry queue**
```typescript
let isRefreshing = false;
let refreshSubscribers: ((token: string) => void)[] = [];

// On 401, refresh token and retry queued requests
client.interceptors.response.use(
  (res) => res,
  async (error: AxiosError) => {
    const originalRequest = error.config!;
    if (error.response?.status === 401 && !originalRequest.headers['X-Retry']) {
      if (!isRefreshing) {
        isRefreshing = true;
        const newToken = await refreshAccessToken();
        isRefreshing = false;
        refreshSubscribers.forEach((cb) => cb(newToken));
        refreshSubscribers = [];
      }
      return new Promise((resolve) => {
        refreshSubscribers.push((token) => {
          originalRequest.headers.set('Authorization', `Bearer ${token}`);
          originalRequest.headers.set('X-Retry', 'true');
          resolve(client.request(originalRequest));
        });
      });
    }
    return Promise.reject(error);
  }
);
```

---

### Stage 10: Case Conversion / 命名转换

Backend conventions (Java/Spring, Python/FastAPI, Go) typically use `snake_case`; frontend conventions typically use `camelCase`. Auto-generate bidirectional conversion.

**Strategy:**
- `--camel-case` flag generates a wrapper that converts all request/response keys
- Detected automatically if spec field names are snake_case and project uses camelCase

```typescript
// utils/case-converter.ts
import { camelCase, snakeCase } from 'change-case';

export function toCamelCase<T>(obj: unknown): T {
  if (obj === null || typeof obj !== 'object') return obj as T;
  if (Array.isArray(obj)) return obj.map(toCamelCase) as unknown as T;
  const result: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(obj)) {
    result[camelCase(key)] =
      value && typeof value === 'object' ? toCamelCase(value) : value;
  }
  return result as T;
}

export function toSnakeCase<T>(obj: unknown): T {
  if (obj === null || typeof obj !== 'object') return obj as T;
  if (Array.isArray(obj)) return obj.map(toSnakeCase) as unknown as T;
  const result: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(obj)) {
    result[snakeCase(key)] =
      value && typeof value === 'object' ? toSnakeCase(value) : value;
  }
  return result as T;
}
```

**Integrated client with auto-conversion:**
```typescript
// client.ts
import { toCamelCase, toSnakeCase } from './utils/case-converter';

async function getUsers() {
  const { data } = await client.get('/users');
  return toCamelCase<GetUsersResponse[200]>(data);
}

async function createUser(body: CreateUserRequest['body']) {
  const { data } = await client.post('/users', toSnakeCase(body));
  return toCamelCase<CreateUserResponse[201]>(data);
}
```

**Naming convention detection:**
| Heuristic | Action |
|-----------|--------|
| `>60%` of spec field names contain `_` | Auto-suggest `--camel-case` |
| Project has `change-case` or `lodash` | Reuse existing dependency |
| Vue/React files use camelCase props | Strong signal to enable |

---

### Stage 11: File Upload & Download / 文件上传与下载

Handle `multipart/form-data` requests and blob/binary responses that standard JSON clients ignore.

#### Upload with multipart/form-data
```typescript
// api/upload.ts
export async function uploadAvatar(
  userId: string,
  file: File,
  onProgress?: (percent: number) => void
): Promise<UploadAvatarResponse[200]> {
  const formData = new FormData();
  formData.append('file', file);
  formData.append('userId', userId);

  const { data } = await client.post('/users/avatar', formData, {
    headers: { 'Content-Type': 'multipart/form-data' },
    onUploadProgress: (event) => {
      if (event.total && onProgress) {
        onProgress(Math.round((event.loaded * 100) / event.total));
      }
    },
  });
  return data;
}
```

#### Download as Blob
```typescript
export async function downloadReport(orderId: string): Promise<Blob> {
  const response = await client.get(`/orders/${orderId}/report`, {
    responseType: 'blob',
  });
  return response.data;
}

export function downloadBlob(blob: Blob, filename: string) {
  const url = window.URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = filename;
  document.body.appendChild(link);
  link.click();
  link.remove();
  window.URL.revokeObjectURL(url);
}
```

**Spec detection:**
- `contentType: multipart/form-data` → generate upload helper
- `contentType: application/octet-stream` or `format: binary` → generate blob download helper
- File fields with `type: string, format: binary` (OpenAPI 3.x) or `type: file` (OpenAPI 2.x)

---

### Stage 12: Dead API Detection / 死代码检测

Scan the frontend codebase to find API endpoints defined in the spec that are never called. Helps backend teams identify obsolete endpoints.

**Detection method:**
1. Extract all paths from the OpenAPI spec
2. Scan `src/` for string literals matching path patterns
3. Also check for `operationId` references in function calls

**Output format:**
```
## 🪦 Dead API Report

### APIs never called from frontend (X total)
1. `DELETE /users/{id}/sessions` — Last spec update: 2024-03-15
   - Suggest: Confirm with backend if this can be deprecated

2. `GET /orders/{id}/timeline` — Referenced in 0 files
   - Suggest: Remove from spec or implement frontend feature

3. `POST /webhooks/register` — Only called in test files
   - Suggest: Move to internal/admin spec

### Low-usage APIs (called in 1 file)
4. `PATCH /users/{id}/preferences` — Called only in src/pages/Settings.tsx
   - Suggest: Verify if still needed
```

---

### Stage 13: Test Generation / 测试代码生成

Generate Vitest / Jest test skeletons from the spec to validate API contracts before integration.

```typescript
// tests/api/users.test.ts
import { describe, it, expect, vi } from 'vitest';
import { apiClient } from '../../src/api/client';
import { UserSchema } from '../../src/api/validations/zodSchemas';

vi.mock('../../src/api/client');

describe('GET /users', () => {
  it('should return paginated user list', async () => {
    const mockResponse = {
      data: [
        { id: 'uuid-1', name: 'Alice', email: 'alice@example.com', role: 'admin', createdAt: '2024-01-01T00:00:00Z' },
      ],
      pagination: { page: 1, pageSize: 20, total: 1 },
    };

    vi.mocked(apiClient.getUsers).mockResolvedValue(mockResponse);

    const result = await apiClient.getUsers({ page: 1, pageSize: 20 });

    expect(result.data).toHaveLength(1);
    expect(UserSchema.safeParse(result.data[0]).success).toBe(true);
  });

  it('should handle 401 unauthorized', async () => {
    vi.mocked(apiClient.getUsers).mockRejectedValue(
      new ApiException({ status: 401, code: 'UNAUTHORIZED', message: 'Token expired' })
    );

    await expect(apiClient.getUsers({})).rejects.toThrow(ApiException);
  });
});

describe('POST /users', () => {
  it('should create user with valid body', async () => {
    const body = { name: 'Bob', email: 'bob@example.com', role: 'user' };
    const mockResponse = { id: 'uuid-2', ...body, createdAt: '2024-06-01T00:00:00Z' };

    vi.mocked(apiClient.createUser).mockResolvedValue(mockResponse);

    const result = await apiClient.createUser(body);
    expect(result.name).toBe('Bob');
  });

  it('should reject invalid email', async () => {
    const body = { name: 'Bob', email: 'not-an-email', role: 'user' };

    vi.mocked(apiClient.createUser).mockRejectedValue(
      new ApiException({ status: 422, code: 'VALIDATION_ERROR', message: 'Invalid email', details: { email: ['Invalid format'] } })
    );

    await expect(apiClient.createUser(body)).rejects.toMatchObject({ error: { status: 422 } });
  });
});
```

**Auto-detect test runner:**
| `package.json` dependency | Generated tests |
|--------------------------|-----------------|
| `vitest` | Vitest |
| `jest` | Jest |
| `playwright` | E2E API tests (optional) |

---

### Stage 14: JSDoc Documentation / JSDoc 文档生成

Enrich generated types with JSDoc comments from the spec's `description` and `example` fields for IDE hover-tooltips.

```typescript
// types/user.types.ts

/**
 * Represents a registered user in the system.
 * @description 系统中的注册用户实体
 */
export interface User {
  /** Unique identifier (UUID v4) */
  id: string;

  /** Display name, 1-100 characters */
  name: string;

  /** Email address used for login and notifications */
  email: string;

  /** User role determining access permissions */
  role: UserRole;

  /** Account creation timestamp (ISO 8601) */
  createdAt: string;

  /** Avatar image URL, optional */
  avatarUrl?: string;
}

/**
 * @enum UserRole
 * @description Defines permission levels within the application
 */
export type UserRole = 'admin' | 'user' | 'guest';

/**
 * Request parameters for fetching paginated user lists
 * @example { page: 1, pageSize: 20, role: 'admin' }
 */
export interface GetUsersRequest {
  /** Page number, starting from 1 */
  page?: number;

  /** Number of items per page, max 100 */
  pageSize?: number;

  /** Filter by user role */
  role?: UserRole;
}
```

**JSDoc enrichment rules:**
- Spec `description` → JSDoc main description
- Spec `example` → `@example` tag
- Spec `minimum`/`maximum` → `@min` / `@max`
- Spec `pattern` (regex) → JSDoc inline hint
- Spec `deprecated: true` → `@deprecated` tag
- Spec `format: email` → `@format email`
- Enum values → JSDoc `@enum` with value descriptions if available

## Output Directory Structure / 输出目录结构

```
src/api/                          # 默认输出目录 (可自定义 --output)
├── types/
│   ├── index.ts                  # 统一导出
│   ├── api.types.ts              # 单文件模式
│   └── [domain]/                 # 按 domain 拆分模式
│       ├── user.types.ts
│       └── order.types.ts
├── client/
│   ├── index.ts                  # 统一导出
│   ├── base-client.ts            # 带拦截器的基础 client (--interceptors)
│   ├── axios-client.ts           # axios 实例
│   ├── fetch-client.ts           # fetch 实例
│   └── upload.ts                 # 文件上传/下载 (--file-upload)
├── hooks/
│   ├── index.ts                  # 统一导出
│   ├── use-query.ts              # TanStack Query hooks (--client tanstack)
│   ├── use-mutation.ts           # Mutation hooks
│   ├── use-paginated-query.ts    # 分页 hooks (--pagination)
│   └── use-infinite-query.ts     # 无限滚动 hooks (--pagination)
├── validations/                  # 运行时校验 (--zod / --valibot)
│   ├── zod-schemas.ts
│   └── valibot-schemas.ts
├── composables/                  # Vue Query composables (Vue 项目)
│   └── use-api.ts
├── utils/
│   ├── case-converter.ts         # snake_case ↔ camelCase (--camel-case)
│   ├── error-handler.ts          # 错误处理工具
│   └── pagination.ts             # 分页工具函数
├── mocks/                        # 仅 --mock 时生成
│   ├── factories.ts
│   ├── handlers.ts
│   └── browser.ts                # MSW browser setup
├── tests/                        # 仅 --test 时生成
│   ├── api/
│   │   ├── users.test.ts
│   │   └── orders.test.ts
│   └── setup.ts                  # vitest/jest setup
└── index.ts                      # 统一导出所有模块
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
13. **Zod/Valibot parity** — validation schema must mirror TypeScript type exactly; never add stricter constraints than the spec implies
14. **Auto-detect pagination** — scan query param names (`page`/`pageSize`, `offset`/`limit`, `cursor`) and response shape to infer pagination pattern
15. **Case conversion opt-in** — only enable `--camel-case` when `>60%` of spec field names contain underscores and frontend uses camelCase
16. **File upload helpers** — generate `FormData` construction and progress callback for every `multipart/form-data` endpoint
17. **Blob download helpers** — generate `responseType: 'blob'` handlers for `application/octet-stream` or `format: binary` responses
18. **Dead API threshold** — flag endpoints called in 0 files as "dead"; called in 1 file as "low usage"
19. **Test coverage per endpoint** — generate at minimum one success test and one error test per operation
20. **JSDoc completeness** — every interface field and every function parameter must have JSDoc when `--jsdoc` is enabled
21. **Environment configs** — generate separate baseURL configs for dev/staging/prod environments
22. **Incremental generation** — when re-running, only regenerate files whose spec schemas have changed (use spec hash or timestamp)

## Tech Stack Detection / 技术栈检测

| 检测特征 | 推断 Stack | Client 默认 |
|---------|-----------|------------|
| `package.json` 有 `axios` | axios | axios client |
| `package.json` 有 `@tanstack/react-query` | tanstack-query | hooks + axios |
| `package.json` 有 `vue` + `@tanstack/vue-query` | vue-query | vue composables |
| `package.json` 有 `msw` | MSW | msw handlers |
| `package.json` 无 axios 有 fetch | fetch API | fetch client |
| `package.json` 有 `zod` | Zod | zod schemas (--zod) |
| `package.json` 有 `valibot` | Valibot | valibot schemas (--valibot) |
| `package.json` 有 `vitest` | Vitest | vitest tests (--test) |
| `package.json` 有 `jest` | Jest | jest tests (--test) |
| `package.json` 有 `change-case` / `lodash` | Case converter | camelCase conversion (--camel-case) |
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
