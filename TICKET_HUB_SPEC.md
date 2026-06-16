# Ticket Hub - 企业工单流转平台 完整规格

## Tech Stack (严格限定)

| Layer | Choice | Notes |
|-------|--------|-------|
| Framework | NestJS 11 | pnpm 包管理 |
| Language | TypeScript 5.7+ | SWC 编译 |
| ORM | Drizzle ORM | drizzle-kit 管理迁移 |
| Database | PostgreSQL 15 | UUID 主键，pgEnum |
| Queue | BullMQ | ioredis 连接 Redis |
| Cache/Queue Backend | Redis 7 | |
| Auth | passport-jwt + bcrypt | |
| WebSocket | Socket.IO via @nestjs/platform-socket.io | |
| Docs | @nestjs/swagger | |
| Validation | class-validator + class-transformer | |
| Test | Jest + supertest, 集成测试连真实 DB | |
| CI | GitHub Actions | |
| Container | Docker multi-stage build | |

---

## 代码风格约定 (全局)

- 所有 NestJS service 注入使用 `private` 不加 `readonly`（如 `private db: Database`）
- 数据库注入 token 来自 `database.module.ts` 导出的 `DATABASE` Symbol
- Database 类型单独从 `src/db/index.ts` 导出（`export type Database = ...`）
- 所有 timestamp 字段加 `.notNull()`
- 所有 DTO 使用 `@nestjs/swagger` 的 `@ApiProperty` / `@ApiPropertyOptional`
- Controller 类级别加 `@ApiTags`, `@ApiBearerAuth`
- 复合唯一键使用 `uniqueIndex` 而非 `primaryKey`（闭包表、user_departments、role_permissions、user_roles 均如此）
- 所有外键加 `{ onDelete: 'cascade' }` 选项（RBAC 表、闭包表、user_departments）
- pgEnum 定义放在 schema.ts 中使用该 enum 的表前面（不集中放顶部）

---

## 目录结构约定

```
src/
├── app.module.ts
├── main.ts
├── health.controller.ts          # 独立 controller, 不放 app.controller
├── common/
│   ├── dto/pagination.dto.ts     # PaginationDto + PaginatedResponse<T> interface
│   ├── filters/global-exception.filter.ts
│   ├── interceptors/data-scope.interceptor.ts
│   └── middleware/request-logger.middleware.ts
├── db/
│   ├── index.ts                  # createDrizzle, export type Database, 不导出 DATABASE
│   ├── database.module.ts        # @Global, export const DATABASE = Symbol('DATABASE')
│   ├── schema.ts
│   └── seed.ts
├── auth/
│   ├── auth.module.ts
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── decorators/permissions.decorator.ts   # RequirePermissions + PERMISSIONS_KEY
│   ├── guards/jwt-auth.guard.ts
│   ├── guards/permission.guard.ts            # 读取 JWT payload 中的 permissions[]
│   ├── strategies/jwt.strategy.ts
│   └── dto/
├── rbac/
│   ├── rbac.module.ts
│   ├── rbac.controller.ts        # CRUD endpoints
│   ├── rbac.service.ts
│   └── dto/
├── department/
│   ├── department.module.ts
│   ├── department.controller.ts
│   ├── department.service.ts
│   └── dto/
├── ticket/
│   ├── ticket.module.ts
│   ├── ticket.controller.ts
│   ├── ticket.service.ts
│   ├── ticket-state-machine.ts   # 纯函数, 非 Injectable
│   ├── dto/
│   └── ticket-state-machine.spec.ts
├── sla/
│   ├── sla.module.ts
│   ├── sla.service.ts
│   ├── sla.processor.ts
│   └── sla.constants.ts          # SLA_QUEUE 常量 + SlaJobData interface
├── notify/
│   ├── notify.module.ts
│   ├── notify.gateway.ts
│   ├── notify.listener.ts
│   └── ticket-transitioned.event.ts   # 事件 class
├── audit/
│   ├── audit.module.ts
│   ├── audit.controller.ts
│   ├── audit.service.ts
│   ├── audit.interceptor.ts
│   └── audit.decorator.ts
└── test/
    ├── setup.ts
    └── test-db.ts
```

---

## Phase 1: Project Init + Database Schema

### 初始化

```bash
nest new ticket-hub --package-manager pnpm
# 安装核心依赖:
# drizzle-orm, pg, @types/pg, drizzle-kit
# @nestjs/config, dotenv
```

### Schema 定义 (src/db/schema.ts, 使用 Drizzle pgTable)

#### users
| Column | Type | Constraint |
|--------|------|-----------|
| id | uuid | PK, default gen_random_uuid() |
| email | varchar(255) | not null, unique |
| password | varchar(255) | not null (bcrypt hash) |
| name | varchar(100) | not null |
| isActive | boolean | default true, not null |
| createdAt | timestamp | default now(), not null |
| updatedAt | timestamp | default now(), not null |

注意: `.notNull().unique()` 顺序（email 字段）

#### departments
| Column | Type | Constraint |
|--------|------|-----------|
| id | uuid | PK |
| name | varchar(100) | not null |
| description | varchar(500) | nullable |
| managerId | uuid | FK → users.id, nullable |
| createdAt | timestamp | default now(), not null |
| updatedAt | timestamp | default now(), not null |

注意: departments 表**没有** `parentId` 字段, 父子关系完全通过闭包表表达。

#### department_closure (闭包表)
| Column | Type | Constraint |
|--------|------|-----------|
| ancestorId | uuid | FK → departments.id (onDelete cascade), not null |
| descendantId | uuid | FK → departments.id (onDelete cascade), not null |
| depth | integer | not null |

约束: `uniqueIndex('dept_closure_unique').on(ancestorId, descendantId)`

#### user_departments
| Column | Type | Constraint |
|--------|------|-----------|
| userId | uuid | FK → users.id (onDelete cascade), not null |
| departmentId | uuid | FK → departments.id (onDelete cascade), not null |
| createdAt | timestamp | default now(), not null |

约束: `uniqueIndex('user_dept_unique').on(userId, departmentId)`

#### roles
| Column | Type | Constraint |
|--------|------|-----------|
| id | uuid | PK |
| name | varchar(50) | not null, unique |
| description | varchar(255) | nullable |
| createdAt | timestamp | default now(), not null |
| updatedAt | timestamp | default now(), not null |

#### permissions
| Column | Type | Constraint |
|--------|------|-----------|
| id | uuid | PK |
| resource | varchar(50) | not null |
| action | varchar(50) | not null |
| description | varchar(255) | nullable |

约束: `uniqueIndex('perm_resource_action_unique').on(resource, action)`

#### role_permissions
| Column | Type | Constraint |
|--------|------|-----------|
| roleId | uuid | FK → roles.id (onDelete cascade), not null |
| permissionId | uuid | FK → permissions.id (onDelete cascade), not null |

约束: `uniqueIndex('role_perm_unique').on(roleId, permissionId)`

#### user_roles
| Column | Type | Constraint |
|--------|------|-----------|
| userId | uuid | FK → users.id (onDelete cascade), not null |
| roleId | uuid | FK → roles.id (onDelete cascade), not null |
| createdAt | timestamp | default now(), not null |

约束: `uniqueIndex('user_role_unique').on(userId, roleId)`

#### tickets

pgEnum 定义在 tickets 表之前:
```typescript
export const ticketStatusEnum = pgEnum('ticket_status', ['OPEN','ASSIGNED','IN_PROGRESS','RESOLVED','CLOSED']);
export const ticketPriorityEnum = pgEnum('ticket_priority', ['LOW','MEDIUM','HIGH','URGENT']);
```

| Column | Type | Constraint |
|--------|------|-----------|
| id | uuid | PK |
| title | varchar(200) | not null |
| description | text | nullable |
| status | ticketStatusEnum | default 'OPEN', not null |
| priority | ticketPriorityEnum | default 'MEDIUM', not null |
| creatorId | uuid | FK → users.id, not null |
| assigneeId | uuid | FK → users.id, nullable |
| departmentId | uuid | FK → departments.id, not null |
| createdAt | timestamp | default now(), not null |
| updatedAt | timestamp | default now(), not null |

注意: tickets 表**没有** `version` 字段。乐观锁通过 conditional UPDATE (WHERE status = currentStatus) 实现，不需要版本号。

#### ticket_logs
| Column | Type | Constraint |
|--------|------|-----------|
| id | uuid | PK |
| ticketId | uuid | FK → tickets.id (onDelete cascade), not null |
| fromStatus | varchar(20) | nullable |
| toStatus | varchar(20) | not null |
| operatorId | uuid | FK → users.id, not null |
| comment | varchar(500) | nullable |
| createdAt | timestamp | default now(), not null |

注意: ticket_logs **没有** `action` 字段。只记录 fromStatus → toStatus。

#### sla_policies

SLA 策略按 **priority** 维度定义（不是按 department）:

| Column | Type | Constraint |
|--------|------|-----------|
| id | uuid | PK |
| priority | ticketPriorityEnum | not null |
| responseMinutes | integer | not null |
| resolveMinutes | integer | not null |

约束: `uniqueIndex('sla_priority_unique').on(priority)`

#### audit_logs

pgEnum 定义在 audit_logs 表之前:
```typescript
export const auditActionEnum = pgEnum('audit_action', ['CREATE','UPDATE','DELETE','LOGIN','LOGOUT','ASSIGN']);
export const auditLevelEnum = pgEnum('audit_level', ['INFO','WARN','CRITICAL']);
```

| Column | Type | Constraint |
|--------|------|-----------|
| id | uuid | PK |
| userId | uuid | FK → users.id, nullable |
| action | auditActionEnum | not null |
| resource | varchar(50) | not null |
| resourceId | varchar(100) | nullable |
| detail | text | nullable |
| ip | varchar(45) | nullable |
| userAgent | varchar(500) | nullable |
| level | auditLevelEnum | default 'INFO', not null |
| createdAt | timestamp | default now(), not null |

注意: audit_logs 不需要索引定义（与 RBAC 不同，不做查询优化）。

### Database Module Pattern

```typescript
// src/db/index.ts
import { Pool } from 'pg';
import { drizzle, NodePgDatabase } from 'drizzle-orm/node-postgres';
import * as schema from './schema';

export function createDrizzle(databaseUrl: string) {
  const pool = new Pool({ connectionString: databaseUrl });
  return drizzle(pool, { schema });
}
export type Database = NodePgDatabase<typeof schema>;
// 注意: DATABASE token 不在这里导出

// src/db/database.module.ts
export const DATABASE = Symbol('DATABASE');
// @Global() module, 用 ConfigService 读取 DATABASE_URL
```

### docker-compose.yml

```yaml
services:
  postgres:
    image: postgres:15-alpine
    container_name: ticket-hub-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: ticket_hub
    ports:
      - "5434:5432"
    volumes:
      - ticket_hub_pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: ticket-hub-redis
    ports:
      - "6381:6379"
    volumes:
      - ticket_hub_redis:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  ticket_hub_pgdata:
  ticket_hub_redis:
```

---

## Phase 2: Auth Module (JWT)

### 依赖
@nestjs/jwt, @nestjs/passport, passport, passport-jwt, bcrypt, @nestjs/config

### 架构决策 (关键)

**权限嵌入 JWT Payload**: AuthService 在签发 token 时查询用户的 permissions 列表并嵌入 JWT payload。
这意味着 PermissionGuard **不需要查数据库**，直接从 `request.user.permissions` 读取。

**JWT Payload 结构**: `{ sub: userId, email, permissions: string[] }`

**返回值**: register/login/refresh 都返回 `{ accessToken, refreshToken }`（两个 token，过期时间不同）

### 实现

**AuthService:**
- `register(dto)`: 查 email 唯一 → bcrypt hash → 插入 users → `generateTokens()`
- `login(dto)`: 查 email → bcrypt.compare → `generateTokens()`
- `refreshToken(userId)`: 查用户存在 → `generateTokens()`
- `private getUserPermissions(userId)`: JOIN userRoles + rolePermissions + permissions → 返回去重的 `resource:action` 数组
- `private generateTokens(userId, email)`: 调用 getUserPermissions → 构建 payload → 分别签发 accessToken (JWT_EXPIRES_IN) 和 refreshToken (JWT_REFRESH_EXPIRES_IN)

```typescript
// generateTokens 的实现模式:
private async generateTokens(userId: string, email: string) {
  const perms = await this.getUserPermissions(userId);
  const payload = { sub: userId, email, permissions: perms };
  const accessToken = this.jwt.sign(payload, { expiresIn: this.config.get('JWT_EXPIRES_IN', '15m') });
  const refreshToken = this.jwt.sign(payload, { expiresIn: this.config.get('JWT_REFRESH_EXPIRES_IN', '7d') });
  return { accessToken, refreshToken };
}
```

**JwtStrategy** (passport-jwt):
- 从 Bearer token 解析
- validate 返回整个 payload: `{ id: payload.sub, email: payload.email, permissions: payload.permissions }`

**PermissionGuard (src/auth/guards/permission.guard.ts):**
- **不查数据库**, 直接读 `request.user.permissions`
- 如果 user 没有 permissions 属性 → throw ForbiddenException
- 如果权限不满足 → throw ForbiddenException('Insufficient permissions')

```typescript
// PermissionGuard 的实现模式:
canActivate(context): boolean {
  const required = this.reflector.getAllAndOverride<string[]>(PERMISSIONS_KEY, [...]);
  if (!required || !required.length) return true;
  const { user } = context.switchToHttp().getRequest();
  if (!user?.permissions) throw new ForbiddenException();
  const hasAll = required.every(perm => user.permissions.includes(perm));
  if (!hasAll) throw new ForbiddenException('Insufficient permissions');
  return true;
}
```

**RequirePermissions 装饰器 (src/auth/decorators/permissions.decorator.ts):**
```typescript
export const PERMISSIONS_KEY = 'permissions';
export const RequirePermissions = (...permissions: string[]) =>
  SetMetadata(PERMISSIONS_KEY, permissions);
```

**DTOs:**
- RegisterDto: email (@IsEmail), password (@MinLength(6)), name (@IsString) — 都带 @ApiProperty
- LoginDto: email, password — 都带 @ApiProperty

### 环境变量
```
JWT_SECRET=your-secret
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
```

---

## Phase 3: RBAC + Department

### RBAC Module (src/rbac/)

**RbacService:**
- createRole(dto: CreateRoleDto) → 插入 roles
- findAllRoles() → 查所有角色
- findAllPermissions() → 查所有权限
- assignPermissionsToRole(roleId, permissionIds[]) → 批量插入 role_permissions
- assignRolesToUser(userId, roleIds[]) → 批量插入 user_roles

**RbacController (src/rbac/rbac.controller.ts):**
- `@ApiTags('RBAC')`, `@ApiBearerAuth()`, `@UseGuards(JwtAuthGuard)`
- `POST /rbac/roles` → createRole, `@Audit('CREATE', 'role')`
- `GET /rbac/roles` → findAllRoles
- `GET /rbac/permissions` → findAllPermissions
- `POST /rbac/roles/assign-permissions` → assignPermissions, `@Audit('ASSIGN', 'role', 'WARN')`
- `POST /rbac/users/assign-roles` → assignRoles, `@Audit('ASSIGN', 'user', 'WARN')`

**RBAC DTOs:**
```typescript
// create-role.dto.ts
export class CreateRoleDto {
  @ApiProperty({ description: 'Role name', example: 'agent' })
  @IsString() @MaxLength(50)
  name: string;

  @ApiPropertyOptional({ description: 'Role description', example: 'Handles tickets' })
  @IsOptional() @IsString() @MaxLength(255)
  description?: string;
}

// assign-permissions.dto.ts
export class AssignPermissionsDto {
  @ApiProperty({ description: 'Target role UUID' })
  @IsUUID()
  roleId: string;

  @ApiProperty({ description: 'Permission UUIDs to assign', type: [String] })
  @IsArray() @IsUUID('4', { each: true })
  permissionIds: string[];
}

// assign-role.dto.ts
export class AssignRoleDto {
  @ApiProperty({ description: 'Target user UUID' })
  @IsUUID()
  userId: string;

  @ApiProperty({ description: 'Role UUIDs to assign', type: [String] })
  @IsArray() @IsUUID('4', { each: true })
  roleIds: string[];
}
```

### Department Module (src/department/)

**DepartmentService:**
- `create(dto)`: 事务内执行: 验证 parentId 存在 → 插入 department → 维护闭包表 (自身 depth=0 + 复制 parent 所有祖先关系 depth+1)
- `findAll(pagination: PaginationDto)`: 返回 PaginatedResponse, orderBy createdAt
- `getTree()`: 查所有 departments + 查闭包表 depth=1 关系 → 在内存中组装树 → 返回 roots[]
- `findOne(id)`: 查不到抛 NotFoundException
- `update(id, dto)`: findOne 确认存在 → update + set updatedAt
- `move(id, newParentId)`: 事务内执行:
  1. 校验 id !== newParentId
  2. 检测环路 (newParent 是否为 id 的后代)
  3. 获取子树所有后代 IDs
  4. 删除外部祖先 → 子树的链接
  5. 获取 newParent 的所有祖先
  6. Cross product: 新祖先 × 子树内部关系 → 插入新行
- `remove(id)`: findOne 确认存在 → delete（无子节点检查，因为 cascade 会处理）

**Move Department DTO:**
```typescript
// move-department.dto.ts
export class MoveDepartmentDto {
  @ApiProperty({ description: 'New parent department UUID' })
  @IsUUID()
  newParentId: string;
}
```

### DataScopeInterceptor (src/common/interceptors/data-scope.interceptor.ts)

**作用**: 在请求上下文中注入 `request.dataScope` — 当前用户可访问的部门 ID 列表。
**逻辑**: 查 userDepartments → 查 departmentClosure 获取所有后代 → 去重后赋值到 request.dataScope

### Seed Data (src/db/seed.ts)

**Permissions (10个):**
ticket:create, ticket:view, ticket:assign, ticket:grab, ticket:resolve, ticket:close, ticket:view_all, sla:config, department:manage, audit:view

**Roles (4个):**
- admin: 全部 10 个权限
- manager: ticket:view, ticket:assign, ticket:view_all, department:manage
- agent: ticket:view, ticket:grab, ticket:resolve, ticket:close
- employee: ticket:create, ticket:view

---

## Phase 4: Ticket State Machine

### 状态机定义 (src/ticket/ticket-state-machine.ts)

**纯函数模块, 不是 Injectable Provider。** 导出 `getTransition()` 和 `getAvailableActions()` 两个函数。

```typescript
export type TicketStatus = 'OPEN' | 'ASSIGNED' | 'IN_PROGRESS' | 'RESOLVED' | 'CLOSED';
export type TicketAction = 'assign' | 'grab' | 'start' | 'resolve' | 'close' | 'reject';

interface Transition {
  to: TicketStatus;
  permission: string;
}

// Record<currentStatus, Record<action, Transition>> 结构:
const TRANSITIONS = {
  OPEN: {
    assign: { to: 'ASSIGNED', permission: 'ticket:assign' },
    grab:   { to: 'IN_PROGRESS', permission: 'ticket:grab' },
  },
  ASSIGNED: {
    start: { to: 'IN_PROGRESS', permission: 'ticket:resolve' },
  },
  IN_PROGRESS: {
    resolve: { to: 'RESOLVED', permission: 'ticket:resolve' },
  },
  RESOLVED: {
    close:  { to: 'CLOSED', permission: 'ticket:close' },
    reject: { to: 'IN_PROGRESS', permission: 'ticket:assign' },
  },
};

export function getTransition(currentStatus, action): Transition | null
export function getAvailableActions(status): TicketAction[]
```

关键差异说明:
- action 叫 `reject`（不是 reopen）
- ASSIGNED → IN_PROGRESS 的 `start` 需要 `ticket:resolve` 权限
- 没有 assignee == currentUser 的校验逻辑（在 service 层处理）

### TicketService

**create(dto, userId):**
- 在事务内: 插入 ticket + 插入 ticket_log (fromStatus: null, toStatus: 'OPEN', comment: 'Ticket created')
- 事务提交后: 调用 `slaService.scheduleChecks(ticket.id, ticket.priority)`

**findAll(pagination, dataScope?):**
- dataScope 来自 DataScopeInterceptor
- 如果有 dataScope → WHERE departmentId IN (dataScope)
- 如果 dataScope 为空数组 → 直接返回空
- 如果 dataScope 为 undefined (有 ticket:view_all) → 不加过滤
- 返回 PaginatedResponse, orderBy createdAt

**findOne(id):** 查单个 ticket

**transition(ticketId, dto, user):**
- `user` 参数包含 `{ id, permissions }` (从 JWT payload 来)
- **grab 操作特殊处理**: 用乐观锁（conditional UPDATE WHERE status='OPEN'），不加行锁
- **其他操作**: 事务内 SELECT FOR UPDATE → validate → UPDATE → insert log
- validate 逻辑:
  1. getTransition(currentStatus, action) → 没有则 BadRequestException
  2. 检查 user.permissions 包含 transition.permission → 否则 ForbiddenException
  3. assign 操作必须提供 assigneeId → 否则 BadRequestException
- SLA 取消逻辑:
  - 状态变为 ASSIGNED / IN_PROGRESS → cancelResponseCheck
  - 状态变为 RESOLVED / CLOSED → cancelAllChecks
- 事件发射: `this.eventEmitter.emit('ticket.transitioned', new TicketTransitionedEvent(...))`

**grab 操作的乐观锁实现:**
```typescript
// 不加 SELECT FOR UPDATE，直接 conditional UPDATE
const [updated] = await this.db
  .update(tickets)
  .set({ status: 'IN_PROGRESS', assigneeId: user.id, updatedAt: new Date() })
  .where(and(eq(tickets.id, ticketId), eq(tickets.status, 'OPEN')))
  .returning();

if (!updated) {
  // 检查是不存在还是已被抢
  const [ticket] = await this.db.select().from(tickets).where(eq(tickets.id, ticketId));
  if (!ticket) throw new NotFoundException('Ticket not found');
  throw new ConflictException('Ticket already grabbed or assigned');
}
```

**getLogs(ticketId):** findOne 确认存在 → 查 ticket_logs orderBy createdAt

### TicketTransitionedEvent (src/notify/ticket-transitioned.event.ts)

```typescript
export class TicketTransitionedEvent {
  constructor(
    public readonly ticketId: string,
    public readonly title: string,
    public readonly fromStatus: string | null,
    public readonly toStatus: string,
    public readonly operatorId: string,
    public readonly creatorId: string,
    public readonly assigneeId: string | null,
  ) {}
}
```

### DTOs
- CreateTicketDto: title (required), departmentId (required, UUID), description (optional), priority (optional, enum)
- TransitionTicketDto: action (required, enum), assigneeId (optional, UUID), comment (optional)

---

## Phase 5: SLA Monitoring

### 依赖
@nestjs/bullmq, bullmq, ioredis

### SLA 常量 (src/sla/sla.constants.ts)

```typescript
export const SLA_QUEUE = 'sla';

export interface SlaJobData {
  ticketId: string;
  type: 'response' | 'resolve';
  priority: string;
}
```

### SLA Module (src/sla/)

**SlaService:**
- `scheduleChecks(ticketId, priority)`: 按 priority 查 sla_policies → 添加**两个** BullMQ delayed job:
  - job name: `'response'`, delay = responseMinutes * 60 * 1000, jobId = `sla:response:${ticketId}`
  - job name: `'resolve'`, delay = resolveMinutes * 60 * 1000, jobId = `sla:resolve:${ticketId}`
- `cancelResponseCheck(ticketId)`: 移除 jobId `sla:response:${ticketId}`
- `cancelResolveCheck(ticketId)`: 移除 jobId `sla:resolve:${ticketId}`
- `cancelAllChecks(ticketId)`: 调用上面两个

**SlaProcessor (BullMQ Worker):**
- 当 job 到期: 检查 ticket 当前状态 → 如果仍为需要响应的状态 → 标记 breached / 发事件

### 集成点
- TicketService.create() 事务提交后调用 `slaService.scheduleChecks(ticket.id, ticket.priority)`
- TicketService.transition():
  - 新状态为 ASSIGNED/IN_PROGRESS → `cancelResponseCheck`
  - 新状态为 RESOLVED/CLOSED → `cancelAllChecks`

---

## Phase 6: WebSocket Notification

### 依赖
@nestjs/websockets, @nestjs/platform-socket.io, socket.io, @nestjs/event-emitter

### Notify Module (src/notify/)

**NotifyGateway (@WebSocketGateway({ cors: { origin: '*' } })):**
- handleConnection: 从 `handshake.auth.token` 或 `headers.authorization` 提取 token → jwt.verify → 加入 room `user:{userId}`
- 无效 token → throw Error (触发 disconnect)
- handleDisconnect: 仅 debug 日志
- `pushToUser(userId, payload)`: `this.server.to('user:${userId}').emit('notification', payload)`

注意:
- Gateway 对外暴露 `pushToUser` 方法供 Listener 调用
- 事件名统一为 `'notification'`（不是 'ticket:update'）
- payload 格式: `{ type: 'TICKET_STATUS_CHANGED', ticketId, title, toStatus }`

**NotifyListener (@Injectable):**
- `@OnEvent('ticket.transitioned')` → 接收 TicketTransitionedEvent
- 构建 payload: `{ type: 'TICKET_STATUS_CHANGED', ticketId, title, toStatus }`
- 推送给 creator (如果不是 operator)
- 推送给 assignee (如果存在且不是 operator)

```typescript
@OnEvent('ticket.transitioned')
handleTransition(event: TicketTransitionedEvent) {
  const payload = { type: 'TICKET_STATUS_CHANGED', ticketId: event.ticketId, title: event.title, toStatus: event.toStatus };
  if (event.creatorId !== event.operatorId) {
    this.gateway.pushToUser(event.creatorId, payload);
  }
  if (event.assigneeId && event.assigneeId !== event.operatorId) {
    this.gateway.pushToUser(event.assigneeId, payload);
  }
}
```

### 架构决策
- EventEmitter2 解耦: TicketModule 不依赖 NotifyModule
- 房间机制: 每个用户一个 room, 服务端决定推给谁

---

## Phase 7: Audit Log (AOP)

### 架构决策
- audit_logs vs ticket_logs: 前者是安全合规 (谁在何时做了什么), 后者是业务流水 (工单状态变更历史)
- 不可变: AuditService 只有 create() 和 findAll(), 无 update/delete
- AOP: 不侵入业务代码, 通过装饰器 + 拦截器实现

### 实现

**@Audit 装饰器 (src/audit/audit.decorator.ts):**
```typescript
export const AUDIT_KEY = 'AUDIT_META';  // 注意常量名是 AUDIT_META

export interface AuditMeta {
  action: 'CREATE' | 'UPDATE' | 'DELETE' | 'LOGIN' | 'LOGOUT' | 'ASSIGN';
  resource: string;
  level?: 'INFO' | 'WARN' | 'CRITICAL';
}

export const Audit = (
  action: AuditMeta['action'],
  resource: string,
  level: AuditMeta['level'] = 'INFO',  // 默认值 'INFO'
) => SetMetadata(AUDIT_KEY, { action, resource, level } as AuditMeta);
```

**AuditInterceptor:**
- 注入 `AuditService`（不直接操作 db）
- 读取 handler 上的 AUDIT_KEY metadata
- 如果有 → 在 response 完成后 (tap operator) → 调用 `auditService.create()`
- resourceId 从 `req.params?.id` 获取
- detail: `JSON.stringify(req.body)`

```typescript
intercept(context, next) {
  const meta = this.reflector.get<AuditMeta>(AUDIT_KEY, context.getHandler());
  if (!meta) return next.handle();
  const req = context.switchToHttp().getRequest();
  return next.handle().pipe(
    tap(() => {
      this.auditService.create({
        userId: req.user?.id,
        action: meta.action,
        resource: meta.resource,
        resourceId: req.params?.id,
        detail: JSON.stringify(req.body),
        ip: req.ip,
        userAgent: req.headers?.['user-agent'],
        level: meta.level,
      });
    }),
  );
}
```

**AuditModule:** @Global(), providers 中注册 AuditInterceptor 为 APP_INTERCEPTOR

**AuditController:** GET /audit-logs, 需要 audit:view 权限, 支持分页 + userId/resource 过滤

### 应用 @Audit 的端点
- auth: register → @Audit('CREATE', 'user'), login → @Audit('LOGIN', 'auth')
- ticket: create → @Audit('CREATE', 'ticket'), transition → @Audit('UPDATE', 'ticket')
- department: create → @Audit('CREATE', 'department'), update/move → @Audit('UPDATE', 'department'), delete → @Audit('DELETE', 'department', 'WARN')
- rbac: createRole → @Audit('CREATE', 'role'), assignPermissions → @Audit('ASSIGN', 'role', 'WARN'), assignRoles → @Audit('ASSIGN', 'user', 'WARN')

---

## Phase 8: Engineering Polish

### 分页 (通用)
```typescript
// src/common/dto/pagination.dto.ts
export class PaginationDto {
  @ApiPropertyOptional({ description: 'Page number', example: 1, default: 1 })
  @IsOptional() @Type(() => Number) @IsInt() @Min(1)
  page?: number = 1;

  @ApiPropertyOptional({ description: 'Items per page', example: 20, default: 20 })
  @IsOptional() @Type(() => Number) @IsInt() @Min(1) @Max(100)
  pageSize?: number = 20;
}

export interface PaginatedResponse<T> {
  data: T[];
  meta: { page: number; pageSize: number; total: number; totalPages: number };
}
```

注意: `PaginatedResponse` 是 interface 不是 class，导出供 service 使用。

### 限流 (Rate Limiting)
- @nestjs/throttler, ThrottlerModule.forRootAsync
- ThrottlerGuard 注册为 APP_GUARD
- 认证端点 @Throttle({ default: { limit: 5, ttl: 60000 } })
- 测试环境不注册 ThrottlerGuard

### 全局异常过滤器
- GlobalExceptionFilter: catch all → 统一格式 { statusCode, message, timestamp }
- 不暴露堆栈信息

### Health Check (src/health.controller.ts)
```typescript
@ApiTags('Health')
@Controller('health')
export class HealthController {
  constructor(@Inject(DATABASE) private db: Database) {}

  @ApiOperation({ summary: 'Health check' })
  @Get()
  async check() {
    await this.db.execute(sql`SELECT 1`);
    return { status: 'ok' };
  }
}
```
独立文件，不放 app.controller.ts。无需认证。

### Swagger Documentation
- main.ts: DocumentBuilder + addBearerAuth() + setup('api-docs')
- 所有 DTO: @ApiProperty / @ApiPropertyOptional (description, example)
- 所有 Controller: @ApiTags, @ApiOperation, @ApiResponse
- 保护端点: @ApiBearerAuth (类级别)

### Testing
- 集成测试连真实 PostgreSQL (test DB on port 5434, database: ticket_hub_test)
- test helper: src/test/setup.ts, src/test/test-db.ts
- Jest: --runInBand
- E2E: supertest + @nestjs/testing

### CI (GitHub Actions)
```yaml
jobs:
  ci:
    services: postgres + redis
    steps: checkout → pnpm install → apply migrations → lint → build → test → test:e2e

  docker:
    needs: ci
    if: push to main
    steps: checkout → buildx → login ghcr.io → build+push (tags: sha + latest)
```

### Dockerfile (Multi-stage)
```dockerfile
# Stage 1: build
FROM node:20-alpine
pnpm install --frozen-lockfile → pnpm build

# Stage 2: production
FROM node:20-alpine
pnpm install --frozen-lockfile --prod
COPY dist + drizzle migrations
HEALTHCHECK → wget /health
CMD node dist/main
```

---

## 环境变量清单

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5434/ticket_hub
REDIS_HOST=localhost
REDIS_PORT=6381
JWT_SECRET=your-secret-key
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
THROTTLE_TTL=60000
THROTTLE_LIMIT=100
PORT=3000
```

---

## 给 AI 的执行指引

1. **按 Phase 顺序执行**, 每个 Phase 完成后验证编译通过
2. **数据模型严格按表格定义**, 不要自行添加字段（注意: users 有 isActive, departments 没有 parentId, tickets 没有 version, ticket_logs 没有 action 字段, sla_policies 按 priority 不是 department）
3. **状态机是纯函数模块 (不是 Injectable)**，使用 Record 嵌套结构，导出 getTransition + getAvailableActions
4. **状态机 action 叫 `reject` 不是 `reopen`**
5. **JWT payload 必须包含 permissions[]**，PermissionGuard 从 request.user.permissions 读取（不查数据库）
6. **Auth 返回 { accessToken, refreshToken }** 两个 token
7. **SLA 按 priority 维度**（不是 department），schedule 两个 job (response + resolve)
8. **grab 操作用乐观锁**（conditional UPDATE WHERE status='OPEN'）, 其他操作用 SELECT FOR UPDATE
9. **NotifyGateway 暴露 pushToUser 方法**，事件名 'notification'，payload 含 type 字段
10. **AuditInterceptor 注入 AuditService**（不直接操作 db），AUDIT_KEY = 'AUDIT_META'
11. **RBAC 有独立的 Controller** (POST /rbac/roles, GET /rbac/roles, GET /rbac/permissions, POST assign-permissions, POST assign-roles)
12. **所有复合键用 uniqueIndex** 不用 composite primaryKey
13. **所有外键带 onDelete: 'cascade'**（RBAC 相关表、闭包表、user_departments、ticket_logs.ticketId）
14. 每个 Phase 完成后运行 `npx tsc --noEmit` 确认无错误
15. 测试代码和业务代码同步编写
