# PostgreSQL for MERN Developers — Zero to Hero
### (Prisma + Supabase + Node.js integration, MongoDB se compare karke)

> **Note pehle:** Maine tera uploaded "PostgreSQL — The Complete Course" PDF (151 pages) dekha. Usme pure **core SQL/PostgreSQL** cover hai — Database/RDBMS basics, installation, CRUD, data types, constraints, clauses, operators, aggregation, ALTER, relationships, JOINs, Views, Procedures. Yeh sab tujhe already aata hai, isliye maine yahan **repeat nahi kiya**.
>
> Jo **missing** tha (aur ek MERN developer ke liye sabse zyada important hai) wo yeh guide cover karti hai:
> - Node.js se PostgreSQL ko **directly** connect karna (`pg` driver)
> - **Prisma ORM** — Mongoose jaisa hi feel, lekin SQL ke liye
> - **Supabase** — Backend-as-a-Service (Postgres + Auth + Realtime + Storage)
> - Transactions, connection pooling, migrations — production-level cheezein
> - Ek **real MERN-style project** jo tera Task Manager Dashboard jaisa hi hoga, lekin PostgreSQL + Prisma ke saath
> - Har jagah **MongoDB/Mongoose se side-by-side comparison**, kyunki wahi teri existing mental model hai

---

## Table of Contents

1. Why is ORM needed — Mongoose vs Prisma mindset shift
2. Raw Node.js + PostgreSQL (`pg` package) — foundation samajhna zaroori hai
3. Prisma ORM — Zero to Hero
4. Supabase — Zero to Hero
5. Real World Project: **TaskFlow API** (MERN + Postgres + Prisma)
6. MongoDB → PostgreSQL Cheat Sheet (side-by-side)
7. Production Best Practices
8. Final Roadmap / Checklist

---

## 1. Why ORM chahiye — Mindset Shift

Tune MongoDB mein seedha `mongoose.connect()` karke schema define kiya, aur phir `User.find()`, `User.create()` jaise methods use kiye. Postgres mein bhi tu raw SQL query bhej sakta hai (`SELECT * FROM users`), lekin real project mein yeh scale nahi karta kyunki:

- Har query manually string mein likhni padegi → typo-prone, SQL injection risk
- Type safety nahi milegi (TypeScript ke saath toh bilkul nahi)
- Relations (JOIN) manually likhne padenge har baar

Isliye Postgres ke world mein bhi **ORM** (Object Relational Mapper) use hota hai — jaisे Mongoose MongoDB ke upar ek layer hai, waise hi **Prisma** Postgres (ya kisi bhi SQL DB) ke upar ek layer hai.

| Concept | MongoDB world | PostgreSQL world |
|---|---|---|
| Driver (raw) | `mongodb` native driver | `pg` (node-postgres) |
| ORM/ODM | **Mongoose** | **Prisma** (ya Sequelize, TypeORM, Drizzle) |
| Schema definition | `mongoose.Schema` | `schema.prisma` file |
| Query style | `Model.find({ age: { $gt: 18 } })` | `prisma.user.findMany({ where: { age: { gt: 18 } } })` |
| Relations | `.populate()` | `include: {}` |

Bas yeh table dimaag mein rakh, poori guide isi analogy pe based hai.

---

## 2. Raw Node.js + PostgreSQL (`pg` package)

Yeh samajhna zaroori hai kyunki Prisma internally isi cheez ko wrap karta hai. Agar tu yeh nahi samjhega, toh connection pooling/transaction errors debug nahi kar payega.

### Installation

```bash
npm install pg
```

### Basic Connection (Client)

```js
// db.js
const { Client } = require('pg');

const client = new Client({
  user: 'postgres',
  host: 'localhost',
  database: 'taskflow',
  password: 'yourpassword',
  port: 5432,
});

client.connect();

client.query('SELECT NOW()', (err, res) => {
  console.log(err, res.rows);
  client.end();
});
```

Yeh MongoDB ke `mongoose.connect(uri)` jaisa hi kaam hai, lekin single connection open karta hai.

### Connection Pooling (IMPORTANT — production mein hamesha Pool use karo)

Single `Client` production mein use nahi karte kyunki har request pe naya connection banega — Postgres slow ho jayega. Isliye **Pool** use hota hai — ek group of reusable connections.

```js
// db.js
const { Pool } = require('pg');

const pool = new Pool({ // yey sbb supabase cloud say ayega so check it
  user: process.env.DB_USER,
  host: process.env.DB_HOST,
  database: process.env.DB_NAME,
  password: process.env.DB_PASSWORD,
  port: process.env.DB_PORT,
  max: 20,                    // max connections in pool
  idleTimeoutMillis: 30000,   // idle connection ko kitni der rakho
  connectionTimeoutMillis: 2000,
});

module.exports = pool;
```

```js
// usage.js
const pool = require('./db');

async function getUsers() {
  const result = await pool.query('SELECT * FROM users WHERE age > $1', [18]);
  return result.rows;
}
```

**Important baat:** `$1, $2, $3` placeholders use karo, kabhi bhi string concatenation (`+`) se query mat banao — warna **SQL Injection** ka risk hai. Yeh bilkul waisa hi hai jaise Mongoose mein tu raw string se query nahi banata.

```js
// ❌ GALAT — SQL Injection risk
pool.query(`SELECT * FROM users WHERE email = '${email}'`);

// ✅ SAHI — parameterized query
pool.query('SELECT * FROM users WHERE email = $1', [email]);
```

### Transactions with `pg` (MongoDB session jaisa concept)

MongoDB mein tu `session.startTransaction()` karta hai jab multiple documents ek saath atomically update karne ho. Postgres mein bhi same concept hai:

```js
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('UPDATE accounts SET balance = balance - 100 WHERE id = $1', [1]);
  await client.query('UPDATE accounts SET balance = balance + 100 WHERE id = $1', [2]);
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');
  throw err;
} finally {
  client.release();  // pool mein wapas connection release karna zaroori hai
}
```

Yeh bank transfer ka classic example hai — dono updates ya toh dono honge, ya koi nahi (atomicity), bilkul MongoDB transactions jaisa.

> Ab tu raw `pg` samajh chuka hai. Real projects mein tu isse directly kam use karega — Prisma iske upar hi kaam karta hai, lekin ab tujhe pata hai neeche kya ho raha hai jab connection error ya pool exhausted jaisi problem aaye.

---

## 3. Prisma ORM — Zero to Hero

### 3.1 Prisma kya hai

Prisma teen main parts se bana hai:

1. **Prisma Schema** (`schema.prisma`) — yahan tu apne models (tables) define karta hai — Mongoose schema jaisa
2. **Prisma Migrate** — schema changes ko actual database tables mein convert karta hai
3. **Prisma Client** — auto-generated, type-safe query builder jo tu apne Express routes mein use karega — Mongoose Model jaisa

### 3.2 Installation & Setup

```bash
npm install prisma --save-dev
npm install @prisma/client

npx prisma init
```

Yeh command do cheezein banayega:
- `prisma/schema.prisma` — tera schema file
- `.env` — database connection string ke liye

```env
# .env
DATABASE_URL="postgresql://postgres:yourpassword@localhost:5432/taskflow?schema=public"
```

### 3.3 Schema Define Karna (Mongoose vs Prisma)

**Mongoose (tera existing knowledge):**

```js
const userSchema = new mongoose.Schema({
  name: String,
  email: { type: String, unique: true },
  age: Number,
  createdAt: { type: Date, default: Date.now }
});
```

**Prisma (`schema.prisma`):**

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  age       Int?
  createdAt DateTime @default(now())
}
```

Notice karo — `@id`, `@unique`, `@default` — yeh Prisma ke **attributes** hain, jaise Mongoose mein tu options object deta tha waise.

### 3.4 Data Types Mapping (Postgres ↔ Prisma ↔ Mongoose)

| Mongoose | Prisma | Postgres actual type |
|---|---|---|
| `String` | `String` | `TEXT` / `VARCHAR` |
| `Number` (int) | `Int` | `INTEGER` |
| `Number` (decimal) | `Float` / `Decimal` | `DOUBLE PRECISION` / `NUMERIC` |
| `Boolean` | `Boolean` | `BOOLEAN` |
| `Date` | `DateTime` | `TIMESTAMP` |
| `[String]` | `String[]` | `TEXT[]` (Postgres array) |
| `mongoose.Schema.Types.Mixed` | `Json` | `JSONB` |
| `ObjectId` (ref) | Relation field (below) | Foreign Key |

### 3.5 Migrations — "Migrate dev" (yeh naya concept hai MongoDB se)

MongoDB **schemaless** hai — tu koi bhi field kabhi bhi add kar sakta hai, DB kuch nahi bolega. Postgres **strict schema** follow karta hai, isliye har schema change ko **migration** ke through apply karna padta ha.

```bash
npx prisma migrate dev --name init
```

Yeh command:
1. `schema.prisma` ko padhega
2. Ek SQL migration file generate karega (`prisma/migrations/xxxx_init/migration.sql`)
3. Us SQL ko actual database pe run karega (tables ban jayenge)
4. Prisma Client ko regenerate karega

Jab bhi tu schema change kare (naya field, naya model, relation), phir se yeh command chalao:

```bash
npx prisma migrate dev --name add_task_priority
```

Production mein deploy karte waqt (`migrate dev` nahi, kyunki wo interactive hai):

```bash
npx prisma migrate deploy
```

### 3.6 Prisma Client — CRUD Operations (Mongoose ke saamne)

```js
// prismaClient.js
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();
module.exports = prisma;
```

| Operation | Mongoose | Prisma |
|---|---|---|
| Create | `User.create({ name: "Anand" })` | `prisma.user.create({ data: { name: "Anand" } })` |
| Find all | `User.find()` | `prisma.user.findMany()` |
| Find one | `User.findById(id)` | `prisma.user.findUnique({ where: { id } })` |
| Find with filter | `User.find({ age: { $gt: 18 } })` | `prisma.user.findMany({ where: { age: { gt: 18 } } })` |
| Update | `User.findByIdAndUpdate(id, data)` | `prisma.user.update({ where: { id }, data })` |
| Delete | `User.findByIdAndDelete(id)` | `prisma.user.delete({ where: { id } })` |
| Count | `User.countDocuments()` | `prisma.user.count()` |

Example — poora CRUD ek Express route mein:

```js
const express = require('express');
const prisma = require('./prismaClient');
const router = express.Router();

// CREATE
router.post('/users', async (req, res) => {
  const user = await prisma.user.create({ data: req.body });
  res.status(201).json(user);
});

// READ ALL
router.get('/users', async (req, res) => {
  const users = await prisma.user.findMany();
  res.json(users);
});

// READ ONE
router.get('/users/:id', async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: Number(req.params.id) },
  });
  if (!user) return res.status(404).json({ message: "Not found" });
  res.json(user);
});

// UPDATE
router.put('/users/:id', async (req, res) => {
  const user = await prisma.user.update({
    where: { id: Number(req.params.id) },
    data: req.body,
  });
  res.json(user);
});

// DELETE
router.delete('/users/:id', async (req, res) => {
  await prisma.user.delete({ where: { id: Number(req.params.id) } });
  res.status(204).send();
});

module.exports = router;
```

Bilkul tere existing Express + Mongoose controllers jaisa structure hai — bas `User.find()` ki jagah `prisma.user.findMany()`.

### 3.7 Relations — `populate()` vs `include`

Yeh sabse important part hai kyunki MongoDB mein relations "soft" hoti hain (`ref` + `populate`), Postgres mein "hard" hoti hain (Foreign Keys — jo tune already PDF mein padha hai).

**One-to-Many example** — ek `User` ke multiple `Task` ho sakte hain:

```prisma
model User {
  id    Int    @id @default(autoincrement())
  name  String
  email String @unique
  tasks Task[]        // ek user ke multiple tasks (virtual field)
}

model Task {
  id        Int      @id @default(autoincrement())
  title     String
  completed Boolean  @default(false)
  userId    Int                        // Foreign Key column
  user      User     @relation(fields: [userId], references: [id])
}
```

**Mongoose mein tu yeh likhta:**

```js
const taskSchema = new mongoose.Schema({
  title: String,
  completed: Boolean,
  user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' }
});
```

**Query karke related data laana:**

```js
// Mongoose
const user = await User.findById(id).populate('tasks');

// Prisma
const user = await prisma.user.findUnique({
  where: { id },
  include: { tasks: true },   // populate() ka replacement
});
```

**Many-to-Many example** — Task aur Tag (ek task ke multiple tags, ek tag multiple tasks pe):

```prisma
model Task {
  id    Int    @id @default(autoincrement())
  title String
  tags  Tag[]  @relation("TaskTags")
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String
  tasks Task[] @relation("TaskTags")
}
```

Prisma khud automatically ek hidden **join table** bana dega (jaise tune PDF ke Phase 6 mein Many-to-Many ke liye manually junction table banayi thi — Prisma yeh kaam khud kar leta hai).

### 3.8 Filtering, Sorting, Pagination

```js
// Mongoose
const tasks = await Task.find({ completed: false })
  .sort({ createdAt: -1 })
  .skip(10)
  .limit(5);

// Prisma
const tasks = await prisma.task.findMany({
  where: { completed: false },
  orderBy: { createdAt: 'desc' },
  skip: 10,
  take: 5,
});
```

Common Prisma filter operators (MongoDB `$` operators jaise):

| MongoDB | Prisma |
|---|---|
| `$gt` | `gt` |
| `$gte` | `gte` |
| `$lt` | `lt` |
| `$in` | `in` |
| `$ne` | `not` |
| `$regex` | `contains` / `startsWith` / `endsWith` |
| `$or` | `OR: [...]` |
| `$and` | `AND: [...]` |

```js
const tasks = await prisma.task.findMany({
  where: {
    OR: [
      { title: { contains: 'urgent', mode: 'insensitive' } },
      { priority: { in: ['high', 'critical'] } },
    ],
  },
});
```

### 3.9 Transactions in Prisma

```js
// MongoDB session transaction ka Prisma equivalent
const [updatedFrom, updatedTo] = await prisma.$transaction([
  prisma.account.update({ where: { id: 1 }, data: { balance: { decrement: 100 } } }),
  prisma.account.update({ where: { id: 2 }, data: { balance: { increment: 100 } } }),
]);
```

Ya interactive transaction (jab beech mein logic/condition check karni ho):

```js
await prisma.$transaction(async (tx) => {
  const account = await tx.account.findUnique({ where: { id: 1 } });
  if (account.balance < 100) throw new Error("Insufficient balance");

  await tx.account.update({ where: { id: 1 }, data: { balance: { decrement: 100 } } });
  await tx.account.update({ where: { id: 2 }, data: { balance: { increment: 100 } } });
});
```

### 3.10 Prisma Studio (Compass jaisa GUI)

```bash
npx prisma studio
```

Yeh browser mein ek GUI khol dega jaha tu apna data visually dekh/edit kar sakta hai — bilkul MongoDB Compass jaisa.

---

## 4. Supabase — Zero to Hero

### 4.1 Supabase kya hai

Supabase ek **Backend-as-a-Service (BaaS)** hai jo underlying mein **pure PostgreSQL** use karta hai, aur upar se yeh extra cheezein deta hai out-of-the-box:

- Hosted PostgreSQL database (managed — tujhe khud install/maintain nahi karna)
- **Auth** (login/signup/OAuth) — Firebase Auth jaisa
- **Realtime subscriptions** — MongoDB Change Streams jaisa concept, live updates
- **Storage** (file uploads) — S3 jaisa
- Auto-generated REST & GraphQL API (seedha tere database ke upar)

Sochो isse aise: **Supabase = Postgres + Firebase ka combination**, aur MERN developer ke liye yeh MongoDB Atlas (managed hosting) ka Postgres version hai — plus extra batteries included.

### 4.2 Project Setup

1. [supabase.com](https://supabase.com) pe account banao
2. "New Project" → naam, password, region choose karo
3. Tujhe milega: `Project URL`, `anon public key`, aur `DATABASE_URL` (Connection string — Postgres)

### 4.3 Do Tarike se Supabase Use Karna

**Option A — Supabase Client Library** (jab frontend se directly DB access chahiye ho, jaise Firebase):

```bash
npm install @supabase/supabase-js
```

```js
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  'https://xxxx.supabase.co',
  'your-anon-key'
);

// Data fetch
const { data, error } = await supabase
  .from('tasks')
  .select('*')
  .eq('completed', false);

// Insert
const { data, error } = await supabase
  .from('tasks')
  .insert([{ title: 'Learn Prisma', completed: false }]);
```

Yeh Firebase Firestore jaisa feel deta hai — MongoDB ke `.find()` jaisa hi, bas syntax alag.

**Option B — Prisma + Supabase (Recommended for MERN backend)** — Supabase sirf **hosted database** ke roop mein use karo, aur queries Prisma se likho (jo tu already seekh chuka hai upar):

```env
# .env — Supabase se milega yeh connection string
DATABASE_URL="postgresql://postgres.xxxx:[password]@aws-0-region.pooler.supabase.com:6543/postgres?pgbouncer=true"
DIRECT_URL="postgresql://postgres.xxxx:[password]@aws-0-region.pooler.supabase.com:5432/postgres"
```

```prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")   // migrations ke liye direct connection
}
```

> **Important gotcha:** Supabase apne Postgres ke aage **PgBouncer** (connection pooler) lagata hai kyunki serverless/Prisma jaise clients bohot saari connections khol sakte hain. Isliye migrations ke liye `DIRECT_URL` (port 5432) aur normal queries ke liye pooled `DATABASE_URL` (port 6543) alag rakhna padta hai — yeh ek common confusion hai jo beginners face karte hain.

### 4.4 Row Level Security (RLS) — Naya concept jo MongoDB mein nahi hai

MongoDB mein tu Express middleware se access control likhta hai (`req.user.id === doc.userId`). Supabase/Postgres mein tu **database level** pe hi security laga sakta hai — isse **Row Level Security (RLS)** kehte hain.

```sql
-- Sirf apna khud ka data dekh sake user
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own tasks"
ON tasks FOR SELECT
USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own tasks"
ON tasks FOR INSERT
WITH CHECK (auth.uid() = user_id);
```

Iska matlab — agar koi frontend se directly Supabase client use kare (Option A), toh bhi database khud check karega ki user sirf apna data access kar raha hai, chahe frontend code compromise ho jaye. Yeh MongoDB mein directly possible nahi hai (waha application-layer security pe hi depend karna padta hai).

> Agar tu Prisma use kar raha hai (Option B — backend se), toh RLS zaroori nahi kyunki access control tera Express middleware already handle kar raha hai. RLS mainly tab critical hai jab frontend directly Supabase client se DB access kare.

### 4.5 Supabase Auth (quick overview)

```js
// Signup
const { data, error } = await supabase.auth.signUp({
  email: 'anand@example.com',
  password: 'strongpassword',
});

// Login
const { data, error } = await supabase.auth.signInWithPassword({
  email: 'anand@example.com',
  password: 'strongpassword',
});
```

Yeh tera JWT-based auth (jo tune RepoLens AI mein banaya) ka ready-made replacement hai — but agar tera apna JWT system already stable hai, toh usko replace karne ki zaroorat nahi, dono valid approaches hain.

### 4.6 Realtime Subscriptions

```js
supabase
  .channel('tasks-channel')
  .on('postgres_changes', { event: 'INSERT', schema: 'public', table: 'tasks' }, (payload) => {
    console.log('New task added!', payload.new);
  })
  .subscribe();
```

Yeh bilkul MongoDB **Change Streams** jaisa concept hai — real-time update aane pe frontend ko notify karna, bina polling ke.

---

## 5. Real World Project: TaskFlow API (MERN + Prisma + Postgres)

Chalo ab tera existing **Task Manager Dashboard** ko lete hain aur backend Postgres+Prisma mein banate hain, taaki concept clear ho jaye end-to-end.

### 5.1 Schema Design

```prisma
model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  password  String
  tasks     Task[]
  createdAt DateTime @default(now())
}

model Task {
  id          Int       @id @default(autoincrement())
  title       String
  description String?
  status      Status    @default(TODO)
  priority    Priority  @default(MEDIUM)
  dueDate     DateTime?
  userId      Int
  user        User      @relation(fields: [userId], references: [id])
  tags        Tag[]     @relation("TaskTags")
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  tasks Task[] @relation("TaskTags")
}

enum Status {
  TODO
  IN_PROGRESS
  DONE
}

enum Priority {
  LOW
  MEDIUM
  HIGH
}
```

Notice `enum` — MongoDB mein tu `{ type: String, enum: ['TODO', 'IN_PROGRESS', 'DONE'] }` likhta tha, Prisma mein yeh first-class citizen hai aur Postgres level pe bhi enforce hota hai.

### 5.2 Migration Run Karna

```bash
npx prisma migrate dev --name taskflow_init
```

### 5.3 Controller — Task with filters, pagination, related tags

```js
// controllers/task.controller.js
const prisma = require('../prismaClient');

exports.getTasks = async (req, res) => {
  const { status, page = 1, limit = 10 } = req.query;
  const userId = req.user.id;   // JWT middleware se aaya (tera existing pattern)

  const tasks = await prisma.task.findMany({
    where: {
      userId,
      ...(status && { status }),
    },
    include: { tags: true },
    orderBy: { createdAt: 'desc' },
    skip: (page - 1) * limit,
    take: Number(limit),
  });

  const total = await prisma.task.count({ where: { userId } });

  res.json({ tasks, total, page: Number(page) });
};

exports.createTask = async (req, res) => {
  const { title, description, priority, dueDate, tagNames = [] } = req.body;
  const userId = req.user.id;

  const task = await prisma.task.create({
    data: {
      title,
      description,
      priority,
      dueDate,
      user: { connect: { id: userId } },
      tags: {
        connectOrCreate: tagNames.map((name) => ({
          where: { name },
          create: { name },
        })),
      },
    },
    include: { tags: true },
  });

  res.status(201).json(task);
};

exports.updateTaskStatus = async (req, res) => {
  const { id } = req.params;
  const { status } = req.body;

  const task = await prisma.task.update({
    where: { id: Number(id) },
    data: { status },
  });

  res.json(task);
};

exports.deleteTask = async (req, res) => {
  await prisma.task.delete({ where: { id: Number(req.params.id) } });
  res.status(204).send();
};
```

`connect` aur `connectOrCreate` naye keywords hain — inka Mongoose equivalent nahi hai kyunki Mongoose mein tu seedha ObjectId assign kar deta tha. Prisma mein relation link karne ke explicit tarike hain:

| Prisma keyword | Matlab |
|---|---|
| `connect` | Existing record se link karo (jaise Mongoose mein `task.user = userId`) |
| `create` | Naya related record banao aur link karo (nested create) |
| `connectOrCreate` | Agar record exist karta hai toh link karo, warna naya banao |
| `disconnect` | Relation hatao (delete nahi, sirf link todo) |

### 5.4 Yeh sab Express app mein wire karna — tera existing structure exactly same rahega

```js
// app.js — koi bada change nahi, bas mongoose ki jagah prisma
const express = require('express');
const app = express();
app.use(express.json());
app.use('/api/tasks', require('./routes/task.routes'));
app.listen(5000, () => console.log('Server running'));
```

Router, middleware, JWT auth, error handling — sab kuch **exactly waisa hi rahega** jaisa tu Mongoose mein likhta tha. Sirf model layer (`prisma.task.*` instead of `Task.*`) change hota hai.

---

## 6. MongoDB → PostgreSQL/Prisma Cheat Sheet

| Kaam | MongoDB / Mongoose | PostgreSQL / Prisma |
|---|---|---|
| Connect DB | `mongoose.connect(uri)` | `DATABASE_URL` in `.env`, Prisma auto-connects |
| Define schema | `new mongoose.Schema({...})` | `model X { ... }` in `schema.prisma` |
| Unique field | `{ unique: true }` | `@unique` |
| Auto ID | `_id` (ObjectId, auto) | `id Int @id @default(autoincrement())` or `@default(uuid())` |
| Default value | `{ default: Date.now }` | `@default(now())` |
| Relation | `ref: 'Model'` + `populate()` | `@relation` + `include` |
| Create | `Model.create()` | `prisma.model.create()` |
| Read many | `Model.find()` | `prisma.model.findMany()` |
| Read one | `Model.findById()` | `prisma.model.findUnique()` |
| Update | `findByIdAndUpdate()` | `prisma.model.update()` |
| Delete | `findByIdAndDelete()` | `prisma.model.delete()` |
| Transactions | `session.startTransaction()` | `prisma.$transaction()` |
| Schema flexibility | Schemaless (koi bhi field add karo) | Strict — migration chahiye har change ke liye |
| GUI tool | MongoDB Compass | Prisma Studio / pgAdmin / DBeaver |
| Hosted service | MongoDB Atlas | Supabase / Neon / Railway |
| Indexing | `schema.index({ field: 1 })` | `@@index([field])` (tu already GIN indexing seekh chuka hai) |
| Aggregation pipeline | `Model.aggregate([...])` | Raw SQL (`$queryRaw`) ya `groupBy()` |

---

## 7. Production Best Practices

1. **Connection pooling hamesha use karo** — `Pool` (raw pg) ya Prisma ka built-in pooling. Serverless environments (Vercel) mein PgBouncer zaroori hai warna "too many connections" error aayega.
2. **Migrations ko version control mein commit karo** — `prisma/migrations` folder ko `.gitignore` mat karo, team ke sabhi members same schema history share karte hain.
3. **`.env` ko kabhi commit mat karo** — `DATABASE_URL` mein password hota hai.
4. **Seed data ke liye `prisma/seed.js` banao:**

```js
// prisma/seed.js
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

async function main() {
  await prisma.user.create({
    data: { name: 'Anand', email: 'anand@test.com', password: 'hashed' },
  });
}

main().finally(() => prisma.$disconnect());
```

```bash
npx prisma db seed
```

5. **N+1 query problem se bacho** — jab loop mein query chalao (jaise 100 tasks ke liye 100 separate user queries), Prisma ka `include` ek hi query mein JOIN kar deta hai — isliye `include` use karo, manual loop mein query mat chalao.
6. **Indexing** — tu already GIN indexing aur `EXPLAIN ANALYZE` seekh chuka hai, wahi concept Prisma models mein `@@index([columnName])` se apply hota hai.

---

## 8. Final Roadmap Checklist

- [x] Core SQL & PostgreSQL fundamentals (already done — uploaded PDF)
- [x] JSONB, GIN Indexing, EXPLAIN ANALYZE (already done — pehle ka roadmap)
- [ ] `pg` driver — Pool, parameterized queries, transactions
- [ ] Prisma schema, migrations, Prisma Client CRUD
- [ ] Prisma relations (`include`, `connect`, `connectOrCreate`)
- [ ] Supabase project setup + connection string (pooled vs direct)
- [ ] Row Level Security basics
- [ ] Build a small project — convert **one** existing Mongoose model (Affectra ya Sketch AI se) into Prisma as practice
- [ ] Deploy Prisma migration on Supabase production DB

Agar tu chahe toh main tere **RepoLens AI ya Sketch AI** ke kisi ek Mongoose model ko step-by-step Prisma schema mein convert karke dikha sakta hoon — real practice ke liye wahi sabse best tareeka hoga.
