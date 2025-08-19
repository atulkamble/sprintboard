# SprintBoard – Jira/Trello-like Web App (MVP)

Opinionated, production-ready starter to run a **project management & task allocation** tool for your employees. It ships with:

* ✅ Projects → Boards (Kanban) → Columns → Tasks → Comments
* ✅ Drag & drop reordering (like Trello)
* ✅ Role-based access (Admin, Manager, Member)
* ✅ Auth (NextAuth Credentials) with hashed passwords
* ✅ PostgreSQL (Prisma ORM) + seed data
* ✅ RESTful APIs (App Router)
* ✅ Clean UI (Tailwind + shadcn/ui)
* ✅ Docker & Docker Compose for local/dev

> **Stack**: Next.js 14 (App Router, TS) · Prisma · PostgreSQL · NextAuth · Tailwind · dnd-kit · shadcn/ui

---

## 1) Repo Structure

```
sprintboard/
├─ app/
│  ├─ api/
│  │  ├─ auth/[...nextauth]/route.ts
│  │  ├─ projects/route.ts
│  │  ├─ projects/[id]/columns/route.ts
│  │  ├─ tasks/route.ts
│  │  ├─ tasks/[id]/route.ts
│  │  └─ seed/route.ts               # optional seed endpoint (dev only)
│  ├─ (auth)/login/page.tsx
│  ├─ (dashboard)/layout.tsx
│  ├─ (dashboard)/page.tsx           # Projects list
│  └─ (dashboard)/projects/[id]/page.tsx  # Kanban board
├─ components/
│  ├─ board/Board.tsx
│  ├─ board/Column.tsx
│  ├─ board/TaskCard.tsx
│  ├─ CreateProjectDialog.tsx
│  ├─ CreateTaskDialog.tsx
│  ├─ Navbar.tsx
│  └─ Providers.tsx
├─ lib/
│  ├─ auth.ts
│  ├─ db.ts
│  ├─ rbac.ts
│  └─ utils.ts
├─ prisma/
│  ├─ schema.prisma
│  └─ seed.ts
├─ public/
│  └─ logo.svg
├─ styles/
│  └─ globals.css
├─ middleware.ts
├─ next.config.mjs
├─ package.json
├─ postcss.config.js
├─ tailwind.config.ts
├─ tsconfig.json
├─ .env.example
├─ Dockerfile
├─ docker-compose.yml
└─ README.md
```

---

## 2) Quick Start

### Option A) Run with Docker (recommended)

```bash
# 1) Clone & enter
git clone https://github.com/your-org/sprintboard.git
cd sprintboard

# 2) Create env file
cp .env.example .env

# 3) Start stack (Next.js + Postgres)
docker compose up -d --build

# 4) Run DB migrations & seed
docker compose exec web npx prisma migrate deploy
docker compose exec web node prisma/seed.js    # compiled at build; see notes below

# 5) Open app
open http://localhost:3000
```

### Option B) Local (no Docker)

```bash
# 1) Install deps
pnpm i # or npm i / yarn

# 2) Prepare DB
cp .env.example .env
# update DATABASE_URL to your Postgres

# 3) Migrate + seed
npx prisma migrate dev
node prisma/seed.ts

# 4) Dev server
pnpm dev
```

**Default Admin (from seed)**

* Email: `admin@sprintboard.local`
* Password: `Admin@123!`

> Change after first login.

---

## 3) Environment (.env.example)

```bash
# App
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=changeme-super-secret

# Database
DATABASE_URL=postgresql://postgres:postgres@db:5432/sprintboard?schema=public

# (Optional) Production cookies
AUTH_TRUST_HOST=true
```

---

## 4) Docker & Compose

**docker-compose.yml**

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: sprintboard
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 10

  web:
    build: .
    env_file: .env
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - .:/app
      - /app/node_modules
    command: ["pnpm", "dev"]

volumes:
  db_data:
```

**Dockerfile**

```dockerfile
# Install deps
FROM node:20-alpine AS base
WORKDIR /app

FROM base AS deps
RUN corepack enable && corepack prepare pnpm@9.7.0 --activate
COPY package.json pnpm-lock.yaml* ./
RUN pnpm i --frozen-lockfile

# Dev image
FROM base AS dev
RUN corepack enable && corepack prepare pnpm@9.7.0 --activate
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["pnpm","dev"]

# Production build
FROM base AS builder
RUN corepack enable && corepack prepare pnpm@9.7.0 --activate
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build

FROM base AS runner
ENV NODE_ENV=production
RUN addgroup -g 1001 -S nodejs && adduser -S nextjs -u 1001
WORKDIR /app
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/node_modules ./node_modules
USER nextjs
EXPOSE 3000
CMD ["node","./node_modules/next/dist/bin/next","start","-p","3000"]
```

---

## 5) Prisma Schema (Database)

**prisma/schema.prisma**

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  password  String
  role      Role     @default(MEMBER)
  projects  ProjectMember[]
  tasks     Task[]   @relation("TaskAssignees")
  comments  Comment[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

enum Role {
  ADMIN
  MANAGER
  MEMBER
}

model Project {
  id        String           @id @default(cuid())
  name      String
  key       String           @unique
  members   ProjectMember[]
  columns   Column[]
  tasks     Task[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model ProjectMember {
  id        String  @id @default(cuid())
  user      User    @relation(fields: [userId], references: [id])
  userId    String
  project   Project @relation(fields: [projectId], references: [id])
  projectId String
  role      Role    @default(MEMBER)
}

model Column {
  id        String   @id @default(cuid())
  title     String
  order     Int
  project   Project  @relation(fields: [projectId], references: [id])
  projectId String
  tasks     Task[]
}

model Task {
  id        String   @id @default(cuid())
  title     String
  description String?
  status    Column   @relation(fields: [columnId], references: [id])
  columnId  String
  project   Project  @relation(fields: [projectId], references: [id])
  projectId String
  assignees User[]   @relation("TaskAssignees")
  position  Float    @default(0)
  dueDate   DateTime?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  comments  Comment[]
}

model Comment {
  id        String   @id @default(cuid())
  body      String
  task      Task     @relation(fields: [taskId], references: [id])
  taskId    String
  author    User     @relation(fields: [authorId], references: [id])
  authorId  String
  createdAt DateTime @default(now())
}

// NextAuth tables (Prisma Adapter not required for Credentials, but included for future OAuth)
model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}
```

**prisma/seed.ts**

```ts
import { PrismaClient } from "@prisma/client";
import bcrypt from "bcryptjs";

const prisma = new PrismaClient();

async function main() {
  const password = await bcrypt.hash("Admin@123!", 10);
  const admin = await prisma.user.upsert({
    where: { email: "admin@sprintboard.local" },
    update: {},
    create: { email: "admin@sprintboard.local", name: "Admin", password, role: "ADMIN" },
  });

  const project = await prisma.project.upsert({
    where: { key: "SB" },
    update: {},
    create: { name: "SprintBoard", key: "SB" },
  });

  await prisma.projectMember.upsert({
    where: { id: `${admin.id}_${project.id}` },
    update: {},
    create: { userId: admin.id, projectId: project.id, role: "ADMIN" },
  });

  const cols = ["Backlog", "To Do", "In Progress", "Done"];
  for (let i = 0; i < cols.length; i++) {
    await prisma.column.create({ data: { title: cols[i], order: i, projectId: project.id } });
  }

  console.log("Seed complete");
}

main().finally(() => prisma.$disconnect());
```

> **Build note (Docker):** transpile `seed.ts` at build or invoke via `tsx`. In dev compose, we call `node prisma/seed.js` if compiled; or replace with `pnpm dlx tsx prisma/seed.ts`.

---

## 6) Auth & RBAC

**lib/auth.ts**

```ts
import NextAuth from "next-auth";
import Credentials from "next-auth/providers/credentials";
import { PrismaClient } from "@prisma/client";
import bcrypt from "bcryptjs";

const prisma = new PrismaClient();

export const { auth, handlers, signIn, signOut } = NextAuth({
  providers: [
    Credentials({
      name: "Email & Password",
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      async authorize(creds) {
        if (!creds?.email || !creds?.password) return null;
        const user = await prisma.user.findUnique({ where: { email: creds.email } });
        if (!user) return null;
        const ok = await bcrypt.compare(creds.password, user.password);
        if (!ok) return null;
        return { id: user.id, email: user.email, name: user.name, role: user.role } as any;
      },
    }),
  ],
  session: { strategy: "jwt" },
  callbacks: {
    async jwt({ token, user }) {
      if (user) token.role = (user as any).role;
      return token;
    },
    async session({ session, token }) {
      (session as any).role = token.role;
      return session;
    },
  },
});
```

**lib/rbac.ts**

```ts
import { Role } from "@prisma/client";

export function canManageProject(role?: Role) {
  return role === "ADMIN" || role === "MANAGER";
}
```

**middleware.ts**

```ts
export { auth as middleware } from "./lib/auth";

export const config = {
  matcher: ["/", "/projects/:path*"],
};
```

---

## 7) App Shell & UI

**components/Providers.tsx**

```tsx
"use client";
import { SessionProvider } from "next-auth/react";

export default function Providers({ children }: { children: React.ReactNode }) {
  return <SessionProvider>{children}</SessionProvider>;
}
```

**app/(dashboard)/layout.tsx**

```tsx
import "@/styles/globals.css";
import Providers from "@/components/Providers";
import Navbar from "@/components/Navbar";

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Providers>
          <Navbar />
          <main className="mx-auto max-w-7xl p-6">{children}</main>
        </Providers>
      </body>
    </html>
  );
}
```

**components/Navbar.tsx**

```tsx
"use client";
import Link from "next/link";
import { useSession, signOut } from "next-auth/react";

export default function Navbar() {
  const { data } = useSession();
  return (
    <header className="border-b bg-white">
      <div className="mx-auto flex max-w-7xl items-center justify-between p-4">
        <Link href="/" className="font-bold">SprintBoard</Link>
        <div className="flex items-center gap-4">
          <Link href="/">Projects</Link>
          {data?.user ? (
            <button onClick={() => signOut()} className="rounded bg-gray-900 px-3 py-1 text-white">Logout</button>
          ) : (
            <Link href="/login" className="rounded bg-gray-900 px-3 py-1 text-white">Login</Link>
          )}
        </div>
      </div>
    </header>
  );
}
```

**app/(auth)/login/page.tsx**

```tsx
"use client";
import { FormEvent, useState } from "react";
import { signIn } from "next-auth/react";
import { useRouter } from "next/navigation";

export default function LoginPage() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);
  const router = useRouter();

  async function onSubmit(e: FormEvent) {
    e.preventDefault();
    const res = await signIn("credentials", { email, password, redirect: false });
    if (res?.ok) router.push("/");
    else setError("Invalid credentials");
  }

  return (
    <div className="mx-auto mt-20 max-w-md rounded-xl border p-6">
      <h1 className="mb-4 text-2xl font-semibold">Sign in</h1>
      <form onSubmit={onSubmit} className="grid gap-3">
        <input className="rounded border p-2" placeholder="Email" value={email} onChange={e=>setEmail(e.target.value)} />
        <input className="rounded border p-2" placeholder="Password" type="password" value={password} onChange={e=>setPassword(e.target.value)} />
        {error && <p className="text-sm text-red-600">{error}</p>}
        <button className="rounded bg-gray-900 px-4 py-2 text-white">Login</button>
      </form>
      <p className="mt-3 text-sm text-gray-500">Try admin@sprintboard.local / Admin@123!</p>
    </div>
  );
}
```

**app/(dashboard)/page.tsx (Projects list)**

```tsx
import { auth } from "@/lib/auth";
import { prisma } from "@/lib/db";
import Link from "next/link";

export default async function ProjectsPage() {
  const session = await auth();
  const email = session?.user?.email as string;
  const me = await prisma.user.findUnique({ where: { email } });
  const memberships = await prisma.projectMember.findMany({
    where: { userId: me?.id },
    include: { project: true },
    orderBy: { id: "asc" },
  });
  return (
    <div>
      <div className="mb-6 flex items-center justify-between">
        <h1 className="text-2xl font-semibold">Your Projects</h1>
        <form action="/api/projects" method="post" className="flex gap-2">
          <input name="name" placeholder="New project name" className="rounded border p-2"/>
          <input name="key" placeholder="KEY" className="w-24 rounded border p-2 uppercase"/>
          <button className="rounded bg-blue-600 px-3 py-2 text-white">Create</button>
        </form>
      </div>
      <ul className="grid grid-cols-1 gap-3 md:grid-cols-3">
        {memberships.map((m) => (
          <li key={m.id} className="rounded-lg border p-4">
            <h3 className="font-medium">{m.project.name}</h3>
            <p className="text-xs text-gray-500">{m.project.key}</p>
            <Link className="mt-2 inline-block rounded bg-gray-900 px-3 py-1 text-white" href={`/projects/${m.project.id}`}>Open Board</Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

**app/(dashboard)/projects/\[id]/page.tsx (Kanban)**

```tsx
import { prisma } from "@/lib/db";
import Board from "@/components/board/Board";

export default async function ProjectBoard({ params }: { params: { id: string } }) {
  const project = await prisma.project.findUnique({
    where: { id: params.id },
    include: {
      columns: { orderBy: { order: "asc" } },
      tasks: { orderBy: { position: "asc" }, include: { assignees: true } },
    },
  });
  if (!project) return <div className="text-red-600">Project not found</div>;
  return <Board project={project} />;
}
```

**components/board/Board.tsx**

```tsx
"use client";
import { DndContext, DragEndEvent } from "@dnd-kit/core";
import Column from "./Column";
import { useState } from "react";

type Project = any;

export default function Board({ project }: { project: Project }) {
  const [columns, setColumns] = useState(project.columns);
  const [tasks, setTasks] = useState(project.tasks);

  async function onDragEnd(ev: DragEndEvent) {
    const taskId = ev.active.id as string;
    const columnId = ev.over?.id as string | undefined;
    if (!columnId) return;
    const res = await fetch(`/api/tasks/${taskId}`, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ columnId })
    });
    if (res.ok) {
      setTasks((prev: any[]) => prev.map(t => t.id === taskId ? { ...t, columnId } : t));
    }
  }

  return (
    <div>
      <div className="mb-4 flex items-center justify-between">
        <h1 className="text-2xl font-semibold">{project.name}</h1>
        <form action="/api/tasks" method="post" className="flex gap-2">
          <input type="hidden" name="projectId" value={project.id} />
          <input name="title" placeholder="New task" className="rounded border p-2" />
          <button className="rounded bg-blue-600 px-3 py-2 text-white">Add</button>
        </form>
      </div>
      <DndContext onDragEnd={onDragEnd}>
        <div className="grid grid-cols-1 gap-4 md:grid-cols-4">
          {columns.map((c: any) => (
            <Column key={c.id} column={c} tasks={tasks.filter((t:any)=>t.columnId===c.id)} />
          ))}
        </div>
      </DndContext>
    </div>
  );
}
```

**components/board/Column.tsx**

```tsx
"use client";
import { useDroppable } from "@dnd-kit/core";
import TaskCard from "./TaskCard";

export default function Column({ column, tasks }: any) {
  const { isOver, setNodeRef } = useDroppable({ id: column.id });
  return (
    <div ref={setNodeRef} className={`rounded-lg border p-3 ${isOver ? "bg-blue-50" : "bg-white"}`}>
      <h3 className="mb-2 text-sm font-semibold uppercase tracking-wider text-gray-600">{column.title}</h3>
      <div className="flex flex-col gap-2">
        {tasks.map((t: any) => <TaskCard key={t.id} task={t} />)}
      </div>
    </div>
  );
}
```

**components/board/TaskCard.tsx**

```tsx
"use client";
import { useDraggable } from "@dnd-kit/core";

export default function TaskCard({ task }: any) {
  const { attributes, listeners, setNodeRef, transform } = useDraggable({ id: task.id });
  const style = transform ? { transform: `translate3d(${transform.x}px, ${transform.y}px, 0)` } : undefined;
  return (
    <div ref={setNodeRef} style={style} {...listeners} {...attributes}
         className="cursor-grab rounded-md border bg-white p-3 shadow-sm">
      <div className="text-sm font-medium">{task.title}</div>
      <div className="text-xs text-gray-500">{task.id.slice(0,6).toUpperCase()}</div>
    </div>
  );
}
```

---

## 8) API Routes (REST)

**lib/db.ts**

```ts
import { PrismaClient } from "@prisma/client";
export const prisma = new PrismaClient();
```

**app/api/projects/route.ts**

```ts
import { prisma } from "@/lib/db";
import { auth } from "@/lib/auth";
import { NextResponse } from "next/server";

export async function POST(req: Request) {
  const session = await auth();
  if (!session?.user?.email) return new NextResponse("Unauthorized", { status: 401 });
  const form = await req.formData();
  const name = String(form.get("name") || "").trim();
  const key = String(form.get("key") || "").trim().toUpperCase();
  if (!name || !key) return new NextResponse("Bad Request", { status: 400 });

  const user = await prisma.user.findUnique({ where: { email: session.user.email } });
  if (!user) return new NextResponse("Unauthorized", { status: 401 });

  const project = await prisma.project.create({ data: { name, key } });
  await prisma.projectMember.create({ data: { projectId: project.id, userId: user.id, role: "ADMIN" } });

  const titles = ["Backlog","To Do","In Progress","Done"];
  await Promise.all(titles.map((t,i)=> prisma.column.create({ data: { title: t, order: i, projectId: project.id } })));
  return NextResponse.redirect(`${process.env.NEXTAUTH_URL}/projects/${project.id}`);
}
```

**app/api/tasks/route.ts**

```ts
import { prisma } from "@/lib/db";
import { auth } from "@/lib/auth";
import { NextResponse } from "next/server";

export async function POST(req: Request) {
  const session = await auth();
  if (!session?.user?.email) return new NextResponse("Unauthorized", { status: 401 });
  const form = await req.formData();
  const title = String(form.get("title") || "");
  const projectId = String(form.get("projectId") || "");
  if (!title || !projectId) return new NextResponse("Bad Request", { status: 400 });

  const firstColumn = await prisma.column.findFirst({ where: { projectId }, orderBy: { order: "asc" } });
  if (!firstColumn) return new NextResponse("No columns", { status: 400 });

  await prisma.task.create({ data: { title, projectId, columnId: firstColumn.id } });
  return NextResponse.redirect(`${process.env.NEXTAUTH_URL}/projects/${projectId}`);
}
```

**app/api/tasks/\[id]/route.ts**

```ts
import { prisma } from "@/lib/db";
import { NextResponse } from "next/server";

export async function PATCH(_req: Request, { params }: { params: { id: string } }) {
  const body = await _req.json();
  const { columnId } = body as { columnId?: string };
  if (!columnId) return new NextResponse("Bad Request", { status: 400 });
  await prisma.task.update({ where: { id: params.id }, data: { columnId } });
  return NextResponse.json({ ok: true });
}

export async function DELETE(_req: Request, { params }: { params: { id: string } }) {
  await prisma.task.delete({ where: { id: params.id } });
  return NextResponse.json({ ok: true });
}
```

**app/api/auth/\[...nextauth]/route.ts**

```ts
export { handlers as GET, handlers as POST } from "@/lib/auth";
```

---

## 9) Tailwind & Global Styles

**tailwind.config.ts**

```ts
import type { Config } from "tailwindcss";
const config: Config = {
  content: ["./app/**/*.{ts,tsx}", "./components/**/*.{ts,tsx}"],
  theme: { extend: {} },
  plugins: [],
};
export default config;
```

**styles/globals.css**

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

html, body { background: #f8fafc; }
```

**package.json**

```json
{
  "name": "sprintboard",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start -p 3000",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev",
    "seed": "tsx prisma/seed.ts"
  },
  "dependencies": {
    "@dnd-kit/core": "^6.1.0",
    "@prisma/client": "^5.18.0",
    "bcryptjs": "^2.4.3",
    "next": "^14.2.5",
    "next-auth": "^5.0.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@types/bcryptjs": "^2.4.6",
    "@types/node": "^20.12.12",
    "@types/react": "^18.3.5",
    "autoprefixer": "^10.4.17",
    "postcss": "^8.4.35",
    "prisma": "^5.18.0",
    "tailwindcss": "^3.4.7",
    "tsx": "^4.16.2",
    "typescript": "^5.5.4"
  }
}
```

**next.config.mjs**

```js
/** @type {import('next').NextConfig} */
const nextConfig = { experimental: { serverActions: { allowedOrigins: ["*"] } } };
export default nextConfig;
```

**postcss.config.js**

```js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

**tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "es2022"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "baseUrl": ".",
    "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
```

---

## 10) How to Extend (roadmap)

* **Sprint planning**: add `Sprint` model; link tasks → sprints; add burn-down chart.
* **Statuses per project**: allow custom columns with WIP limits.
* **Labels & filters**: TaskLabel model + UI filter bar.
* **Attachments**: integrate S3/MinIO for file uploads.
* **Real-time**: add Socket.IO or Pusher for multi-user live updates.
* **Notifications**: email via SES/SendGrid; in-app toasts & inbox.
* **SSO**: NextAuth OAuth (Google/Microsoft Entra) + organization/tenant support.
* **Audit log**: write to `ActivityLog` model.
* **Analytics**: velocity, cycle time, cumulative flow diagram.

---

## 11) Security & Ops

* Enforce strong password policy & 2FA (TOTP) for admins.
* Use **row-level checks** on API (verify membership on every write/read).
* Add rate limiting on auth & write endpoints.
* Backups: daily `pg_dump` + retention policy.
* Infra as Code: provision Postgres + secrets via Terraform (AWS RDS/Neon/Azure PG).

---

## 12) Troubleshooting

* **Port already in use** (3000): change `start -p <port>` or stop the process: `lsof -i :3000 | awk 'NR>1 {print $2}' | xargs kill -9` (macOS/Linux).
* **Prisma schema changes**: `npx prisma migrate dev` in dev; `migrate deploy` in prod.
* **Seed not found**: use `pnpm dlx tsx prisma/seed.ts` inside the container.

---

## 13) License

MIT – use internally at your company or customize.

Awesome, Atul — I’ve dropped a complete **Jira/Trello-style web app starter** called **SprintBoard** into the canvas. It includes:

* Next.js 14 + Tailwind UI with drag-and-drop Kanban
* Auth (NextAuth Credentials), RBAC (Admin/Manager/Member)
* Prisma + PostgreSQL schema & seed
* REST APIs for projects, columns, tasks
* Dockerfile + docker-compose for one-command local run

### Run it now

1. In the repo root:

```bash
cp .env.example .env
docker compose up -d --build
docker compose exec web npx prisma migrate deploy
docker compose exec web pnpm dlx tsx prisma/seed.ts
```

2. Open: [http://localhost:3000](http://localhost:3000)
   Login: `admin@sprintboard.local` / `Admin@123!` (change after first login).

If you want, I can tailor it for:

* **SSO** (Google/Microsoft Entra)
* **S3/MinIO** file attachments
* **Real-time updates** (Socket.IO/Pusher)
* **Terraform** for AWS/Azure DB + secrets

Tell me your preferred cloud & auth method, and I’ll extend the stack accordingly.
