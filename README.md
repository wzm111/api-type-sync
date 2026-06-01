# api-type-sync — 前后端接口类型同步助手

一个 Claude Code Skill，用于自动同步后端 OpenAPI/Swagger 定义与前端 TypeScript 代码。

## 核心能力

| 能力 | 说明 |
|------|------|
| 📝 **类型生成** | 读取 OpenAPI/Swagger 自动生成 TypeScript 类型定义 |
| 🔍 **不匹配检查** | 扫描前端调用代码，报告与接口定义的差异 |
| 🔧 **Client 生成** | 自动生成 axios / fetch / TanStack Query 调用代码 |
| 📊 **变更 Diff** | 检测后端接口变更，列出受影响的前端代码位置 |
| 🎭 **Mock 生成** | 基于接口定义生成 faker / MSW mock 数据 |

## 安装

将本目录复制到你的 Claude Code 项目的 `.claude/skills/` 下：

```bash
cp -r api-type-sync /your/project/.claude/skills/
```

## 使用方式

### 基础命令

```
/api-type-sync                          # 自动发现 spec，全量同步
/api-type-sync --generate               # 仅生成类型
/api-type-sync --generate --client axios    # 生成类型 + axios client
/api-type-sync --generate --client tanstack # 生成类型 + TanStack Query hooks
/api-type-sync --check                  # 检查前端调用与 spec 不匹配
/api-type-sync --diff                   # 显示接口变更及影响范围
/api-type-sync --mock                   # 生成 mock 数据
/api-type-sync --all                    # 执行全部工作流
```

### 常用参数

| 参数 | 说明 |
|------|------|
| `--spec <path>` | 指定 OpenAPI spec 文件路径 |
| `--output <dir>` | 指定输出目录（默认 `./src/api`） |
| `--client <type>` | client 类型：`axios` / `fetch` / `tanstack` |
| `--mock --format msw` | 生成 MSW handlers |
| `--mock --format faker` | 生成 faker 工厂函数 |
| `--watch` | 监听 spec 文件变化自动重新生成 |

## 触发条件

Skill 会在以下情况自动激活：
- 用户输入 `/api-type-sync`
- 检测到项目中的 `swagger.json`、`openapi.yaml`、`openapi.json`
- 检测到 `api-spec/`、`openapi/`、`swagger/` 等目录
- 用户询问 API 类型同步、swagger 生成、接口检查相关问题

## 输出结构

```
src/api/
├── types/
│   ├── index.ts              # 统一导出
│   └── [domain].types.ts     # 按 domain 拆分的类型
├── client.ts                 # API client
├── hooks.ts                  # TanStack Query hooks
├── fetch-client.ts           # fetch-based client
├── index.ts                  # 统一导出
└── mocks/
    ├── factories.ts          # faker 工厂
    └── handlers.ts           # MSW handlers
```

## 技术栈适配

Skill 会自动检测项目技术栈并选择默认生成策略：

| 检测特征 | 默认 Client |
|---------|------------|
| 有 `axios` | axios client |
| 有 `@tanstack/react-query` | tanstack hooks + axios |
| 有 `@tanstack/vue-query` | vue-query composables |
| 有 `msw` | msw handlers |
| 无 axios | fetch client |
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

## 注意事项

- 优先验证 spec 有效性，无效 spec 会产生错误类型
- 生成时会合并而非覆盖已有自定义扩展
- 尽量使用 `type` 而非 `interface` 处理循环引用
- 日期字段默认输出为 `string`（ISO 8601），可通过 `--dates-as-date` 切换
- 对于复杂 spec，建议配合 `openapi-typescript` 或 `orval` 等专用工具使用

## License

MIT
