# Introduction to Node.js & Databases

Full slide content in Markdown format. Each slide is separated by `---`.

---

## Slide 01 — Title

# Introduction to Node.js & Databases

**Drivers, ORMs, Migrations & Production Patterns**

pg · Prisma · Knex · Sequelize · Mongoose · migrations · transactions

---

## Slide 02 — Agenda

### Foundations
- Database access in Node.js
- Raw SQL drivers (pg, mysql2, better-sqlite3)
- Connection pooling & lifecycle
- Query builders — Knex.js

### ORMs & ODMs
- Prisma — modern type-safe ORM
- Prisma in practice — CRUD & transactions
- Sequelize — classic ORM
- Mongoose & MongoDB

### Data Management
- Database migrations
- Seeding & fixtures
- Transactions & ACID
- Connection management in Express

### Production
- Error handling & retry strategies
- Performance & N+1 queries
- Testing database code
- Choosing an ORM / query builder

---

## Slide 03 — Database Access in Node.js

Node.js has first-class async I/O, making it a natural fit for database work. Every database operation is **non-blocking by default** — no thread-per-connection model.

### Driver Landscape
- **pg** — PostgreSQL
- **mysql2** — MySQL / MariaDB
- **better-sqlite3** — SQLite (sync)
- **mongodb** — MongoDB native driver
- **ioredis** — Redis

### Connection Patterns
- **Single client** — one TCP connection
- **Connection pool** — reusable connections
- **Connection string** — `postgres://user:pass@host/db`
- **SSL/TLS** — encrypted in production

### Async by Default
- Every query returns a **Promise**
- Use `async/await` for clean flow
- `Promise.all()` for parallel queries
- Streams for large result sets

### Abstraction Levels

| Level         | Tool             | Trade-off                      |
|---------------|------------------|--------------------------------|
| Raw SQL       | pg, mysql2       | Full control, no abstraction   |
| Query Builder | Knex.js          | Composable, DB-agnostic SQL    |
| ORM           | Prisma, Sequelize| Models, relations, type safety |
| ODM           | Mongoose         | Schema enforcement for MongoDB |

---

## Slide 04 — Raw SQL Drivers

Raw drivers give you direct access to SQL with minimal abstraction. Every Node.js database library is built on top of these.

### pg — PostgreSQL

```javascript
const { Client } = require('pg');
const client = new Client({
  connectionString: process.env.DATABASE_URL
});
await client.connect();

const { rows } = await client.query(
  'SELECT * FROM users WHERE email = $1',
  ['alice@example.com']
);
console.log(rows[0]);
await client.end();
```

### mysql2 — MySQL

```javascript
const mysql = require('mysql2/promise');
const conn = await mysql.createConnection({
  host: 'localhost',
  user: 'root',
  database: 'myapp'
});

const [rows] = await conn.execute(
  'SELECT * FROM users WHERE email = ?',
  ['alice@example.com']
);
console.log(rows[0]);
await conn.end();
```

### better-sqlite3

```javascript
const Database = require('better-sqlite3');
const db = new Database('app.db');

const stmt = db.prepare(
  'SELECT * FROM users WHERE email = ?'
);
const user = stmt.get('alice@example.com');

const insert = db.transaction((users) => {
  for (const u of users)
    db.prepare('INSERT INTO users (name) VALUES (?)').run(u.name);
});
```

> **Always use parameterised queries.** Never interpolate user input into SQL strings. Use `$1` (pg), `?` (mysql2/sqlite), or named parameters.

---

## Slide 05 — Connection Pooling

Opening a new TCP connection per query is expensive (~20-50ms). A **connection pool** maintains pre-established connections that are reused across requests.

```javascript
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});

// Auto checkout/checkin
const { rows } = await pool.query(
  'SELECT * FROM users WHERE active = $1', [true]
);

// Manual checkout for transactions
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [100, 1]);
  await client.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [100, 2]);
  await client.query('COMMIT');
} catch (e) {
  await client.query('ROLLBACK');
  throw e;
} finally {
  client.release();
}
```

### Pool Config Rules of Thumb
- `max` = CPU cores × 2 + disk spindles
- PostgreSQL default max_connections = **100**
- With 5 app instances: max = 100 / 5 = **20 per pool**
- Set `connectionTimeoutMillis` to fail fast

### Connection Lifecycle
- **Idle** — waiting in pool for checkout
- **Active** — checked out, running queries
- **Expired** — idle timeout reached, destroyed
- **Error** — removed from pool, replaced

---

## Slide 06 — Query Builders — Knex.js

Knex is a **SQL query builder** that supports PostgreSQL, MySQL, SQLite3, and MSSQL. It generates parameterised SQL and provides migrations, seeds, and a chainable API.

```javascript
const knex = require('knex')({
  client: 'pg',
  connection: process.env.DATABASE_URL,
  pool: { min: 2, max: 10 }
});

const users = await knex('users')
  .select('id', 'name', 'email')
  .where('active', true)
  .andWhere('role', 'admin')
  .orderBy('created_at', 'desc')
  .limit(20);

const [newUser] = await knex('users')
  .insert({ name: 'Alice', email: 'alice@dev.io' })
  .returning('*');

await knex('users')
  .where('id', 42)
  .update({ last_login: knex.fn.now() });
```

### Migrations

```javascript
exports.up = function(knex) {
  return knex.schema.createTable('users', (t) => {
    t.increments('id').primary();
    t.string('name').notNullable();
    t.string('email').unique().notNullable();
    t.boolean('active').defaultTo(true);
    t.timestamps(true, true);
  });
};

exports.down = function(knex) {
  return knex.schema.dropTable('users');
};
```

```bash
npx knex migrate:latest
npx knex migrate:rollback
npx knex seed:run
```

---

## Slide 07 — Prisma — Modern ORM

Prisma is a **next-generation ORM** with a declarative schema, auto-generated type-safe client, and powerful migration system.

### schema.prisma

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  posts     Post[]
  profile   Profile?
  createdAt DateTime @default(now())
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int
  tags      Tag[]
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[]
}
```

### Generate & Use

```bash
npm install prisma @prisma/client
npx prisma init
npx prisma generate
npx prisma migrate dev --name init
npx prisma studio
```

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

const user = await prisma.user.findUnique({
  where: { email: 'alice@dev.io' },
  include: { posts: true, profile: true }
});
```

---

## Slide 08 — Prisma in Practice

### CRUD Operations

```typescript
// Create
const user = await prisma.user.create({
  data: { name: 'Alice', email: 'alice@dev.io' }
});

// Read with filtering
const admins = await prisma.user.findMany({
  where: {
    role: 'ADMIN',
    createdAt: { gte: new Date('2024-01-01') }
  },
  orderBy: { name: 'asc' }
});

// Update
const updated = await prisma.user.update({
  where: { id: 1 },
  data: { name: 'Alice Smith' }
});

// Delete
await prisma.user.delete({ where: { id: 1 } });
```

### Pagination

```typescript
// Offset pagination
const page = await prisma.post.findMany({
  skip: 20, take: 10,
  orderBy: { createdAt: 'desc' }
});

// Cursor pagination
const next = await prisma.post.findMany({
  take: 10,
  cursor: { id: lastId },
  skip: 1,
});
```

### Transactions

```typescript
await prisma.$transaction(async (tx) => {
  const sender = await tx.account.update({
    where: { id: 1 },
    data: { balance: { decrement: 100 } }
  });
  if (sender.balance < 0) throw new Error('Insufficient funds');
  await tx.account.update({
    where: { id: 2 },
    data: { balance: { increment: 100 } }
  });
});
```

### Raw SQL Escape Hatch

```typescript
const result = await prisma.$queryRaw`
  SELECT u.name, COUNT(p.id) as post_count
  FROM "User" u
  LEFT JOIN "Post" p ON p."authorId" = u.id
  GROUP BY u.id
  HAVING COUNT(p.id) > ${minPosts}
`;
```

---

## Slide 09 — Sequelize

Sequelize is a **mature, full-featured ORM** for Node.js supporting PostgreSQL, MySQL, MariaDB, SQLite, and MSSQL. It follows the Active Record pattern.

### Model Definition

```javascript
const { Sequelize, DataTypes } = require('sequelize');
const sequelize = new Sequelize(process.env.DATABASE_URL);

const User = sequelize.define('User', {
  name: { type: DataTypes.STRING, allowNull: false },
  email: { type: DataTypes.STRING, unique: true, validate: { isEmail: true } },
  role: { type: DataTypes.ENUM('USER', 'ADMIN'), defaultValue: 'USER' }
}, { paranoid: true });

const Post = sequelize.define('Post', {
  title: DataTypes.STRING,
  content: DataTypes.TEXT,
  published: { type: DataTypes.BOOLEAN, defaultValue: false }
});

User.hasMany(Post, { foreignKey: 'authorId' });
Post.belongsTo(User, { foreignKey: 'authorId' });
```

### Querying

```javascript
const users = await User.findAll({
  where: { role: 'ADMIN' },
  include: [{ model: Post, where: { published: true } }],
  order: [['createdAt', 'DESC']],
  limit: 10
});
```

### Hooks

```javascript
User.beforeCreate(async (user) => {
  user.email = user.email.toLowerCase();
});

User.afterDestroy(async (user) => {
  await AuditLog.create({ action: 'USER_DELETED', userId: user.id });
});
```

---

## Slide 10 — Mongoose & MongoDB

Mongoose is the **de-facto ODM** (Object Document Mapper) for MongoDB in Node.js. It adds schema validation, middleware, and relationship management.

### Schema & Model

```javascript
const mongoose = require('mongoose');
await mongoose.connect(process.env.MONGO_URI);

const userSchema = new mongoose.Schema({
  name:  { type: String, required: true, trim: true },
  email: { type: String, unique: true, lowercase: true },
  posts: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Post' }],
  metadata: {
    loginCount: { type: Number, default: 0 },
    lastSeen: Date
  }
}, { timestamps: true });

userSchema.pre('save', function(next) {
  if (this.isModified('password')) {
    this.password = hash(this.password);
  }
  next();
});

const User = mongoose.model('User', userSchema);
```

### Population

```javascript
const user = await User.findById(userId)
  .populate({
    path: 'posts',
    match: { published: true },
    select: 'title createdAt',
    options: { sort: { createdAt: -1 }, limit: 5 }
  });
```

### Aggregation Pipeline

```javascript
const stats = await Post.aggregate([
  { $match: { published: true } },
  { $group: {
      _id: '$authorId',
      totalPosts: { $sum: 1 },
      avgLength: { $avg: { $strLenCP: '$content' } }
  }},
  { $sort: { totalPosts: -1 } },
  { $limit: 10 }
]);
```

---

## Slide 11 — Database Migrations

Migrations are **version control for your database schema**. They ensure every environment has the same structure and changes are applied in order.

### Why Migrations Matter
- Reproducible schema across environments
- Team collaboration without conflicts
- Rollback capability for failed deploys
- Audit trail of schema evolution

### Knex Migrations

```javascript
exports.up = function(knex) {
  return knex.schema.createTable('posts', (t) => {
    t.increments('id').primary();
    t.string('title').notNullable();
    t.text('content');
    t.integer('author_id').references('id').inTable('users').onDelete('CASCADE');
    t.timestamps(true, true);
  });
};

exports.down = function(knex) {
  return knex.schema.dropTable('posts');
};
```

### Prisma Migrate

```bash
npx prisma migrate dev --name add_posts
npx prisma migrate deploy
npx prisma migrate status
```

### Migration Workflow
1. Modify schema / create migration file
2. Test locally with `migrate dev`
3. Commit migration files to git
4. CI/CD runs `migrate deploy` on release
5. Rollback if health checks fail

---

## Slide 12 — Seeding & Fixtures

Seeds populate databases with **test data** for development, testing, and demos. Fixtures provide deterministic data for automated tests.

### Knex Seeds

```javascript
exports.seed = async function(knex) {
  await knex('users').del();
  await knex('users').insert([
    { id: 1, name: 'Alice', email: 'alice@test.io', role: 'ADMIN' },
    { id: 2, name: 'Bob',   email: 'bob@test.io',   role: 'USER' },
    { id: 3, name: 'Carol', email: 'carol@test.io',  role: 'USER' }
  ]);
};
```

### Prisma Seed

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

async function main() {
  const alice = await prisma.user.upsert({
    where: { email: 'alice@test.io' },
    update: {},
    create: { name: 'Alice', email: 'alice@test.io' }
  });
}
main().finally(() => prisma.$disconnect());
```

### Factory Patterns

```typescript
import { faker } from '@faker-js/faker';

export function buildUser(overrides = {}) {
  return {
    name: faker.person.fullName(),
    email: faker.internet.email(),
    role: 'USER',
    ...overrides
  };
}

const admin = buildUser({ role: 'ADMIN' });
```

### Seed Best Practices
- **Idempotent** — safe to run multiple times
- **Deterministic** — same output every run for tests
- **Isolated** — seeds don't depend on external APIs
- **Ordered** — respect foreign key constraints
- Separate dev seeds from test fixtures

---

## Slide 13 — Transactions

A transaction groups multiple operations into an **atomic unit**. Either all succeed or all are rolled back.

### ACID Properties

| Property    | Meaning                                       |
|-------------|-----------------------------------------------|
| Atomicity   | All or nothing — partial writes never persist  |
| Consistency | DB moves from one valid state to another       |
| Isolation   | Concurrent txns don't interfere                |
| Durability  | Committed data survives crashes                |

### Prisma Interactive Transactions

```typescript
await prisma.$transaction(async (tx) => {
  const order = await tx.order.create({
    data: { userId: 1, total: 99.99 }
  });
  await tx.inventory.update({
    where: { productId: 42 },
    data: { stock: { decrement: 1 } }
  });
  await tx.payment.create({
    data: { orderId: order.id, amount: 99.99 }
  });
});
```

### Knex Transactions

```javascript
await knex.transaction(async (trx) => {
  const [orderId] = await trx('orders')
    .insert({ user_id: 1, total: 99.99 }).returning('id');
  await trx('inventory').where('product_id', 42).decrement('stock', 1);
  await trx('payments').insert({ order_id: orderId, amount: 99.99 });
});
```

### Isolation Levels

| Level             | Prevents                    |
|-------------------|-----------------------------|
| Read Uncommitted  | Nothing (fastest)           |
| Read Committed    | Dirty reads                 |
| Repeatable Read   | + Non-repeatable reads      |
| Serializable      | + Phantom reads (safest)    |

PostgreSQL default: **Read Committed**. Use Serializable for financial operations.

---

## Slide 14 — Connection Management in Express

### Shared Pool (Recommended)

```javascript
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL, max: 20 });
module.exports = pool;

router.get('/users', async (req, res) => {
  const { rows } = await pool.query('SELECT * FROM users');
  res.json(rows);
});
```

### Dependency Injection

```javascript
function createApp(pool) {
  const app = express();
  app.get('/users', async (req, res) => {
    const { rows } = await pool.query('SELECT * FROM users');
    res.json(rows);
  });
  return app;
}

// Production: createApp(realPool)
// Test:       createApp(mockPool)
```

### Middleware Pattern

```javascript
app.use(async (req, res, next) => {
  req.db = await pool.connect();
  res.on('finish', () => req.db.release());
  next();
});
```

> **Anti-Pattern:** Never create a new pool per request. Pool creation is expensive and you'll exhaust database connections.

### Prisma in Express

```typescript
const prisma = new PrismaClient();
app.get('/users', async (req, res) => {
  res.json(await prisma.user.findMany());
});
// Prisma manages its own connection pool internally.
```

---

## Slide 15 — Error Handling

### Constraint Violations

```typescript
try {
  await prisma.user.create({ data: { email: 'alice@dev.io' } });
} catch (e) {
  if (e.code === 'P2002') {
    return res.status(409).json({ error: `${e.meta.target} already exists` });
  }
  throw e;
}

// pg: error.code === '23505' (unique_violation)
// pg: error.code === '23503' (fk_violation)
```

### Retry Strategy

```javascript
async function withRetry(fn, { retries = 3, delay = 100 } = {}) {
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (err) {
      const isTransient = ['ECONNRESET', 'ETIMEDOUT', '57P01'].includes(err.code);
      if (!isTransient || i === retries - 1) throw err;
      await new Promise(r => setTimeout(r, delay * Math.pow(2, i)));
    }
  }
}
```

### Dead Letter Patterns
- Failed operations → dead letter queue
- Retry with exponential backoff + jitter
- Alert after N failures (PagerDuty, Slack)
- Circuit breaker: stop calling failing DB

### Common PostgreSQL Error Codes

| Code    | Meaning              |
|---------|----------------------|
| `23505` | Unique violation     |
| `23503` | Foreign key violation|
| `42P01` | Table does not exist |
| `40001` | Serialization failure|

---

## Slide 16 — Performance

### The N+1 Problem

```javascript
// BAD: 1 query for users + N queries for posts
const users = await prisma.user.findMany();
for (const user of users) {
  user.posts = await prisma.post.findMany({
    where: { authorId: user.id }
  });
}

// GOOD: eager loading — 1-2 queries total
const users = await prisma.user.findMany({
  include: { posts: true }
});
```

### Query Analysis

```sql
EXPLAIN ANALYZE
SELECT u.*, COUNT(p.id)
FROM users u
LEFT JOIN posts p ON p.author_id = u.id
GROUP BY u.id;
```

### Indexing (Prisma)

```prisma
model Post {
  id       Int    @id @default(autoincrement())
  authorId Int
  status   String
  title    String

  @@index([authorId])
  @@index([status, authorId])
}
```

### Eager Loading Strategies

| Strategy | ORM       | Method                        |
|----------|-----------|-------------------------------|
| Include  | Prisma    | `include: { posts: true }`    |
| Eager    | Sequelize | `include: [Post]`             |
| Populate | Mongoose  | `.populate('posts')`          |
| Join     | Knex      | `.join('posts', ...)`         |

### Connection Pool Tuning
- Monitor `pool.waitingCount` — if high, increase `max`
- Monitor `pool.idleCount` — if high, decrease `max`
- Set `statement_timeout` to kill runaway queries
- Use `pgBouncer` for connection multiplexing at scale

---

## Slide 17 — Testing Database Code

### Transaction Rollback

```javascript
beforeEach(async () => { await pool.query('BEGIN'); });
afterEach(async () => { await pool.query('ROLLBACK'); });
```

### In-Memory SQLite

```javascript
const knex = require('knex')({
  client: 'better-sqlite3',
  connection: { filename: ':memory:' },
  useNullAsDefault: true
});

beforeAll(async () => {
  await knex.migrate.latest();
  await knex.seed.run();
});
afterAll(() => knex.destroy());
```

### Prisma Test Setup

```typescript
beforeEach(async () => {
  const tables = Prisma.dmmf.datamodel.models.map(m => m.dbName || m.name);
  for (const table of tables) {
    await prisma.$executeRawUnsafe(`TRUNCATE TABLE "${table}" CASCADE`);
  }
});
```

### Testing Strategies Compared

| Strategy              | Speed   | Fidelity |
|-----------------------|---------|----------|
| Transaction rollback  | Fast    | High     |
| Truncate between tests| Medium  | High     |
| In-memory SQLite      | Fastest | Lower    |
| Docker per test suite | Slow    | Highest  |

---

## Slide 18 — Choosing an ORM / Query Builder

| Criterion       | pg / mysql2     | Knex       | Prisma        | Sequelize    | Mongoose          |
|-----------------|-----------------|------------|---------------|--------------|-------------------|
| Type Safety     | None            | Basic      | Excellent     | Via TS defs  | Via TS defs       |
| Learning Curve  | Low (just SQL)  | Low-Medium | Medium        | Medium-High  | Medium            |
| Query Flexibility| Full SQL       | High       | Medium        | Medium       | High (aggregation)|
| Migrations      | Manual          | Built-in   | Built-in      | Built-in     | N/A               |
| Performance     | Best            | Near-native| Good          | Overhead     | Depends on queries|
| Relations       | Manual JOINs    | Manual JOINs| Declarative  | Declarative  | Population        |
| Database        | One per driver  | PG/MySQL/SQLite| PG/MySQL/SQLite/Mongo| PG/MySQL/SQLite/MSSQL| MongoDB only |

### Choose Prisma When
- TypeScript project
- Want generated types from schema
- Standard CRUD-heavy application
- New greenfield project

### Choose Knex / Raw When
- Complex reporting queries
- Dynamic query composition
- Maximum performance needed
- Team knows SQL well

### Choose Sequelize / Mongoose When
- Existing project already uses it
- Need runtime model definition
- MongoDB document store (Mongoose)
- Rich hook/middleware system needed

---

## Slide 19 — Summary & Next Steps

### What We Covered
- Raw SQL drivers: pg, mysql2, better-sqlite3
- Connection pooling & lifecycle management
- Query builder: Knex.js
- Modern ORM: Prisma — type-safe, declarative schema
- Classic ORM: Sequelize — Active Record pattern
- ODM: Mongoose — MongoDB schema validation
- Migrations, seeding, transactions, ACID
- Error handling, retries, performance tuning
- Testing strategies: rollback, fixtures, in-memory DB

### Next Steps
- Build a REST API with Prisma + Express
- Set up migrations & seed a dev database
- Write integration tests with transaction rollback
- Profile queries with `EXPLAIN ANALYZE`
- Explore connection pooling with pgBouncer
- Try Drizzle ORM — another modern TypeScript ORM
- Learn about database replication & read replicas
- Explore Redis for caching & session storage

### Essential Reading
- [Prisma Documentation](https://www.prisma.io/docs)
- [Knex.js Guide](https://knexjs.org/guide/)
- [node-postgres Docs](https://node-postgres.com/)

### Key Takeaway
Start with **Prisma** for new TypeScript projects. Use **raw SQL** when you need performance or complex queries. Always use **connection pooling**, **parameterised queries**, and **migrations**.
