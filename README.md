# md迁移
传统迁移方式会导致技术栈老旧等问题，目前采用md文档 + ai模型方式迁移，md文档迁移经过被人验证96%+以上都是没问题，同时ai会适配最新的技术栈
# Ticket Hub - 企业工单流转平台

NestJS 11 + PostgreSQL + Redis + BullMQ 构建的工单管理系统，支持 RBAC 权限、状态机流转、SLA 监控、实时通知。

## 技术栈

| 层 | 选型 |
|---|---|
| Framework | NestJS 11 (TypeScript 5.7, SWC) |
| ORM | Drizzle ORM |
| Database | PostgreSQL 15 |
| Cache / Queue | Redis 7 (ioredis + BullMQ) |
| Auth | JWT + bcrypt + RBAC |
| WebSocket | Socket.IO |
| Test | Jest + supertest |
| Benchmark | k6 |

## 核心功能

- **RBAC 权限体系** — 角色/权限矩阵，JWT payload 嵌入 permissions，Guard 链鉴权
- **工单状态机** — 5 状态 6 动作，纯函数实现，grab 抢单用乐观锁
- **部门层级** — 闭包表实现 O(1) 子树查询，支持跨层级移动 + 环路检测
- **SLA 监控** — BullMQ 延迟队列，按优先级自动触发超时告警
- **实时通知** — EventEmitter 解耦 + WebSocket 房间推送
- **审计日志** — Decorator + Interceptor AOP 模式，不侵入业务代码

---

## 性能优化实践

### 问题背景

tickets 表 100 万行数据，`GET /tickets` 分页接口在 20 并发下 P95 = 1.9s。

### 诊断过程

```sql
EXPLAIN ANALYZE SELECT count(*) FROM tickets;
-- Parallel Seq Scan → 232ms（全表扫描）

EXPLAIN ANALYZE SELECT * FROM tickets ORDER BY created_at DESC LIMIT 20 OFFSET 5000;
-- Parallel Seq Scan + Sort → 177ms（无索引，内存排序 100 万行取 20 条）
```

### 优化方案

| 优化项 | 原理 | 效果 |
|--------|------|------|
| B-tree 索引 `(created_at DESC)` | 排序从全表扫描变为 Index Scan | 子查询 151ms → 42ms |
| 延迟 JOIN | 先在索引上取 ID（窄行），再回表取完整数据（避免宽行排序） | 200ms → 8.7ms |
| 近似计数 `pg_class.reltuples` | 避免 count(*) 全表扫描，精度误差 < 1% | 232ms → 0.01ms |
| 复合索引 `(department_id, created_at DESC)` | dataScope 过滤 + 排序一次完成 | - |

### 压测对比 (k6, 20 VU, 30s)

```
优化前: list_tickets avg=1.46s   p(90)=1.77s  p(95)=1.9s   QPS=712
优化后: list_tickets avg=16.8ms  p(90)=23ms   p(95)=25ms   QPS=931
```

**P95 提升 76 倍，吞吐量提升 30%。**

---

## Redis 缓存设计

### Cache-Aside 模式

```
GET /tickets/:id
  → 查 Redis（命中直接返回）
  → 未命中 → 查 DB → 写入 Redis (TTL 5min) → 返回
```

### 缓存一致性：延迟双删

```
写操作（transition/grab）后:
  1. 立即删除缓存
  2. 500ms 后再删一次（防止并发读把旧数据写回缓存）
```

为什么需要双删？存在时序问题：

```
线程A: 读 DB（旧值）          → 写缓存（旧值）
线程B:          写 DB（新值） → 删缓存
结果: 缓存里是旧值（脏数据）
```

延迟双删的第二次删除覆盖了这个窗口。

---

## 接口幂等性

### 问题

网络超时 → 客户端自动重试 → 服务端收到重复请求 → 创建重复工单。

### 方案

请求头 `Idempotency-Key` + Redis 原子锁 + 结果缓存：

```
第一次: SET key NX EX 600 → OK → 执行业务 → 存结果 → 返回
第二次: SET key NX EX 600 → nil → 取缓存结果 → 直接返回（不执行业务）
```

### 实现

NestJS Interceptor — 请求前检查/加锁，响应后存结果。挂载到 `POST /tickets` 和 `POST /tickets/:id/transition`。

---

## 并发控制

| 场景 | 方案 | 原因 |
|------|------|------|
| 抢单 (grab) | 乐观锁 `UPDATE WHERE status='OPEN'` | 无锁等待，快速失败，适合高并发抢占 |
| 状态流转 (assign/resolve/close) | 悲观锁 `SELECT FOR UPDATE` | 保证读-校验-写的原子性 |

---

## 快速启动

```bash
# 启动基础设施
docker compose up -d

# 安装依赖
pnpm install

# 数据库迁移
pnpm drizzle-kit push

# 初始化种子数据 (角色/权限)
pnpm db:seed

# 启动开发服务
pnpm start:dev

# Swagger 文档
open http://localhost:3000/api-docs
```

### 压测

```bash
# 插入 100 万测试数据
pnpm db:seed-bulk

# 运行 k6 性能测试
k6 run benchmark/perf-test.js
```

## 环境变量

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
