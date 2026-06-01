# api-type-sync — 前后端接口类型同步助手

一个 Claude Code Skill，用于自动同步后端 OpenAPI/Swagger 定义与前端 TypeScript 代码。

## 核心能力

| 能力 | 说明 | 参数 |
| ---- | ---- | ---- |
| 📝 **类型生成** | 读取 OpenAPI/Swagger 自动生成 TypeScript 类型定义 | `--generate` |
| 🔍 **不匹配检查** | 扫描前端调用代码，报告与接口定义的差异 | `--check` |
| 🔧 **Client 生成** | 自动生成 axios / fetch / TanStack Query 调用代码 | `--client <type>` |
| 📊 **变更 Diff** | 检测后端接口变更，列出受影响的前端代码位置 | `--diff` |
| 🎭 **Mock 生成** | 基于接口定义生成 faker / MSW mock 数据 | `--mock` |
| 🛡️ **Zod / Valibot 校验** | 从 spec 生成运行时校验 schema | `--zod` / `--valibot` |
| 📄 **分页抽象** | 自动识别分页模式，生成分页 hooks/composables | `--pagination` |
| 🔒 **拦截器模板** | 生成统一错误处理、token 刷新拦截器 | `--interceptors` |
| 🔄 **命名转换** | snake_case ↔ camelCase 自动双向转换 | `--camel-case` |
| 📎 **文件上传/下载** | 生成 multipart / blob 处理方法 | `--file-upload` |
| 🪦 **死代码检测** | 找出 spec 定义但前端未调用的接口 | `--dead-api` |
| 🧪 **测试生成** | 生成 vitest / jest API 单元测试骨架 | `--test` |
| 📖 **JSDoc 文档** | 为类型添加字段描述注释，IDE 悬停可见 | `--jsdoc` |

## 安装

将本目录复制到你的 Claude Code 项目的 `.claude/skills/` 下：

```bash
cp -r api-type-sync /your/project/.claude/skills/
```

## 使用方式

### 基础命令

```text
/api-type-sync                          # 自动发现 spec，全量同步
/api-type-sync --generate               # 仅生成类型
/api-type-sync --generate --client axios    # 生成类型 + axios client
/api-type-sync --generate --client tanstack # 生成类型 + TanStack Query hooks
/api-type-sync --check                  # 检查前端调用与 spec 不匹配
/api-type-sync --diff                   # 显示接口变更及影响范围
/api-type-sync --mock                   # 生成 mock 数据
/api-type-sync --zod                    # 生成 Zod 运行时校验 schema
/api-type-sync --pagination             # 生成分页 hooks / composables
/api-type-sync --interceptors           # 生成拦截器 + 错误处理模板
/api-type-sync --camel-case             # 启用 snake_case ↔ camelCase 转换
/api-type-sync --file-upload            # 生成文件上传/下载处理
/api-type-sync --dead-api               # 检测未调用的死接口
/api-type-sync --test                   # 生成 API 单元测试
/api-type-sync --jsdoc                  # 为类型添加 JSDoc 注释
/api-type-sync --all                    # 执行全部工作流（类型 + client + zod + interceptors + test + mock）
```

### 常用参数

| 参数 | 说明 |
| ---- | ---- |
| `--spec <path>` | 指定 OpenAPI spec 文件路径 |
| `--output <dir>` | 指定输出目录（默认 `./src/api`） |
| `--client <type>` | client 类型：`axios` / `fetch` / `tanstack` |
| `--mock --format msw` | 生成 MSW handlers |
| `--mock --format faker` | 生成 faker 工厂函数 |
| `--zod` | 生成 Zod 校验 schema（项目需已安装 `zod`） |
| `--valibot` | 生成 Valibot 校验 schema（项目需已安装 `valibot`） |
| `--pagination` | 识别分页模式并生成统一分页 hooks |
| `--interceptors` | 生成请求/响应拦截器（401 跳转、token 刷新等） |
| `--camel-case` | 自动转换 snake_case ↔ camelCase |
| `--file-upload` | 生成 multipart FormData / blob 下载方法 |
| `--dead-api` | 扫描前端死接口 |
| `--test` | 生成 vitest/jest 测试用例 |
| `--jsdoc` | 为所有类型和函数生成 JSDoc |
| `--watch` | 监听 spec 文件变化自动重新生成 |

## 触发条件

Skill 会在以下情况自动激活：

- 用户输入 `/api-type-sync`
- 检测到项目中的 `swagger.json`、`openapi.yaml`、`openapi.json`
- 检测到 `api-spec/`、`openapi/`、`swagger/` 等目录
- 用户询问 API 类型同步、swagger 生成、接口检查相关问题

## 输出结构

```text
src/api/
├── types/
│   ├── index.ts                  # 统一导出
│   ├── api.types.ts              # 单文件模式
│   └── [domain]/                 # 按 domain 拆分模式
│       ├── user.types.ts
│       └── order.types.ts
├── client/
│   ├── index.ts                  # 统一导出
│   ├── base-client.ts            # 带拦截器的基础 client
│   ├── axios-client.ts           # axios 实例
│   ├── fetch-client.ts           # fetch 实例
│   └── upload.ts                 # 文件上传/下载处理
├── hooks/
│   ├── index.ts                  # 统一导出
│   ├── use-query.ts              # TanStack Query hooks
│   ├── use-mutation.ts           # Mutation hooks
│   ├── use-paginated-query.ts    # 分页 hooks
│   └── use-infinite-query.ts     # 无限滚动 hooks
├── validations/                  # 运行时校验
│   ├── zod-schemas.ts
│   └── valibot-schemas.ts
├── composables/                  # Vue Query composables (Vue 项目)
│   └── use-api.ts
├── utils/
│   ├── case-converter.ts         # snake_case ↔ camelCase
│   ├── error-handler.ts          # 错误处理工具
│   └── pagination.ts             # 分页工具函数
├── mocks/                        # Mock 数据
│   ├── factories.ts
│   ├── handlers.ts
│   └── browser.ts                # MSW browser setup
├── tests/                        # API 单元测试
│   ├── api/
│   │   ├── users.test.ts
│   │   └── orders.test.ts
│   └── setup.ts
└── index.ts                      # 统一导出所有模块
```

## 技术栈适配

Skill 会自动检测项目技术栈并选择默认生成策略：

| 检测特征 | 默认策略 |
| -------- | -------- |
| 有 `axios` | axios client |
| 有 `@tanstack/react-query` | tanstack hooks + axios |
| 有 `vue` + `@tanstack/vue-query` | vue-query composables |
| 有 `msw` | msw handlers |
| 无 axios | fetch client |
| 有 `zod` | 默认生成 zod schemas |
| 有 `valibot` | 默认生成 valibot schemas |
| 有 `vitest` | vitest 测试 |
| 有 `jest` | jest 测试 |
| 有 `change-case` / `lodash` | 启用 camelCase 转换 |
| Next.js | hooks + fetch |
| Nuxt/Vue | vue-query + fetch |

## 不匹配检查能力

扫描前端代码，检测以下问题：

- 请求体缺少必填字段
- 发送了 spec 中不存在的字段
- 类型不匹配（如发送 `number` 但 spec 要求 `string`）
- 缺少必填查询参数
- HTTP 方法错误
- 路径参数不匹配
- 响应字段被删除但前端仍在使用
- 缺少错误状态码处理

## 分页模式自动识别

| 模式 | 查询参数 | 响应结构 | 生成代码 |
| ---- | -------- | -------- | -------- |
| Offset/Limit | `offset`, `limit` | `{ data, total }` | `useInfiniteQuery` + offset |
| Page/PageSize | `page`, `pageSize` | `{ data, pagination }` | `useInfiniteQuery` + page |
| Cursor | `cursor`, `limit` | `{ data, nextCursor }` | `useInfiniteQuery` + cursor |

## 拦截器能力

生成的拦截器模板包含：

- 自动附加 Authorization Bearer token
- 401 自动跳转登录页
- Token 过期自动刷新 + 请求重试队列
- 全局错误分类处理（403/404/422/500）
- 网络错误统一封装

## Zod / Valibot 运行时校验

不仅生成 TypeScript 编译时类型，还生成运行时校验 schema：

```typescript
import { UserSchema } from './validations/zod-schemas';

// 后端返回的数据直接做运行时校验
const result = UserSchema.safeParse(response.data);
if (!result.success) {
  console.error('Backend returned invalid data:', result.error);
}
```

## 死代码检测

扫描 `src/` 中所有 API 调用，对比 spec 中的 endpoint：

- **Dead**（0 处调用）：建议后端确认是否可以废弃
- **Low usage**（1 处调用）：建议验证是否仍需保留
- 输出包含文件路径、最后更新时间、建议操作

## 测试生成

每个 API endpoint 自动生成：

- 1 个成功场景测试（验证正常响应结构）
- 1 个错误场景测试（验证异常处理）
- 与 Zod schema 联动校验返回数据格式

## 注意事项

- 优先验证 spec 有效性，无效 spec 会产生错误类型
- 生成时会合并而非覆盖已有自定义扩展
- 尽量使用 `type` 而非 `interface` 处理循环引用
- 日期字段默认输出为 `string`（ISO 8601），可通过 `--dates-as-date` 切换
- `--camel-case` 仅在检测到 snake_case 命名时建议启用
- `--pagination` 自动识别分页模式，无需手动指定
- 对于复杂 spec，建议配合 `openapi-typescript` 或 `orval` 等专用工具使用

## License

MIT
