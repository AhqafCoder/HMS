# HMS Unified App – plan.txt

**Goal:** Build a production-grade, unified web application for Hostel Management with Admin, Warden, and Student roles using Next.js (frontend), Express (backend API), PostgreSQL (DB), Clerk (Auth), and Prisma (ORM). Enforce strict RBAC and tenant scoping (hostel_id) across all layers.

---
## 1) Product Scope
- **Actors/Roles:** Admin, Warden, Student
- **Core Modules:**
  1. Hostel Setup (Admin)
  2. Floor & Room Management (Warden)
  3. Student Onboarding (Warden)
  4. Cleaning Requests (Student → Warden/Staff)
  5. Announcements (Warden/Admin → Students)
  6. Analytics & Reports (Admin/Warden)
  7. Audit Logs (All write actions)

- **Non-Goals (MVP):** Payments, biometric attendance, IoT devices, inventory, laundry, visitors (can be phased later).

---
## 2) High-Level Architecture
- **Frontend:** Next.js 14 app router, TypeScript, Tailwind, shadcn/ui, React Query, Zod.
- **Backend:** Express.js (Node 20+), TypeScript, Zod for validation, Prisma for DB access.
- **Auth:** Clerk (JWT sessions). Store role/hostel bindings in DB and mirror essential role data in Clerk public metadata.
- **DB:** PostgreSQL. Single database, multi-tenant via `hostel_id` FK.
- **Infra:** Dockerized services, Nginx or Vercel for Next.js, Render/Fly/EC2 for Express, managed Postgres (Supabase/RDS/Neon).
- **Caching:** Optional Redis (rate limit, session adjuncts, hot queries).
- **Observability:** Winston logs → Logtail/Datadog; OpenTelemetry optional; Health checks `/healthz`.

---
## 3) Security Model (RBAC + Tenant Scoping)
- **Principles:** Frontend is untrusted. All authorization on server. Every query is scoped by `hostel_id`.
- **JWT:** Clerk short-lived session tokens; require on every API call. Use `Authorization: Bearer <token>`.
- **RBAC:** Roles: `admin`, `warden`, `student`.
- **Binding:** `user_roles` table binds `user_id` → `role` → `hostel_id` (nullable for super-admin). Backend cross-checks Clerk user to DB.
- **Middleware:** `requireRole(role)` + `requireHostelContext()`; join on `hostel_id` for all reads/writes.
- **Hardening:**
  - Enforce 2FA for `admin` & `warden` in Clerk.
  - Rate limit by route + user.
  - Input validation (Zod) + output filtering.
  - Prisma: use transactions, parameterized queries.
  - Room capacity constraints at DB level.
  - Audit logs for all mutations.

**RBAC Matrix (MVP):**
| Capability | Admin | Warden | Student |
|---|---|---|---|
| Create hostel | ✔ | ✖ | ✖ |
| Assign warden | ✔ | ✖ | ✖ |
| Manage floors/rooms (within hostel) | ✔ (any) | ✔ (own hostel) | ✖ |
| Create student accounts | ✖ (optional) | ✔ | ✖ |
| Post announcements | ✔ | ✔ (own hostel) | ✖ |
| View announcements | ✔ | ✔ | ✔ |
| Submit cleaning request | ✖ | ✖ | ✔ (own room) |
| Process/close cleaning request | ✖ | ✔ (own hostel) | ✖ |
| View analytics | ✔ | ✔ (own hostel) | ✖ |

---
## 4) Data Model (Prisma Schema Outline)
```prisma
model User { // mirrored from Clerk (reference only)
  id           String   @id // clerk_user_id
  email        String   @unique
  createdAt    DateTime @default(now())
  roles        UserRole[]
  students     Student[]
  wardens      Warden[]
}

model Hostel {
  id        String   @id @default(cuid())
  name      String   @unique
  createdBy String
  createdAt DateTime @default(now())
  wardens   Warden[]
  floors    Floor[]
  rooms     Room[]
  students  Student[]
  announces Announcement[]
}

model UserRole {
  id        String  @id @default(cuid())
  userId    String
  role      Role
  hostelId  String?
  user      User    @relation(fields: [userId], references: [id])
  hostel    Hostel? @relation(fields: [hostelId], references: [id])
  @@index([userId, role, hostelId])
}

enum Role { admin warden student }

model Warden {
  id        String  @id @default(cuid())
  userId    String
  hostelId  String
  user      User    @relation(fields: [userId], references: [id])
  hostel    Hostel  @relation(fields: [hostelId], references: [id])
  createdAt DateTime @default(now())
}

model Floor {
  id        String  @id @default(cuid())
  hostelId  String
  number    Int
  createdAt DateTime @default(now())
  hostel    Hostel  @relation(fields: [hostelId], references: [id])
  rooms     Room[]
  @@unique([hostelId, number])
}

model Room {
  id        String  @id @default(cuid())
  floorId   String
  hostelId  String
  number    String
  capacity  Int      @default(2)
  createdAt DateTime @default(now())
  floor     Floor    @relation(fields: [floorId], references: [id])
  hostel    Hostel   @relation(fields: [hostelId], references: [id])
  students  Student[]
  cleanings CleaningRequest[]
  @@unique([hostelId, number])
}

model Student {
  id         String  @id @default(cuid())
  userId     String  // clerk user
  hostelId   String
  roomId     String
  collegeId  String  @unique
  fatherPhone String
  createdAt  DateTime @default(now())
  user       User    @relation(fields: [userId], references: [id])
  hostel     Hostel  @relation(fields: [hostelId], references: [id])
  room       Room    @relation(fields: [roomId], references: [id])
}

model CleaningRequest {
  id          String   @id @default(cuid())
  hostelId    String
  roomId      String
  studentId   String
  status      CleaningStatus @default(PENDING)
  requestedAt DateTime @default(now())
  completedAt DateTime?
  room        Room     @relation(fields: [roomId], references: [id])
  student     Student  @relation(fields: [studentId], references: [id])
  hostel      Hostel   @relation(fields: [hostelId], references: [id])
  @@index([hostelId, status])
}

enum CleaningStatus { PENDING IN_PROGRESS DONE REJECTED }

model Announcement {
  id        String   @id @default(cuid())
  hostelId  String
  title     String
  content   String
  createdBy String // userId
  createdAt DateTime @default(now())
  hostel    Hostel  @relation(fields: [hostelId], references: [id])
}

model AuditLog {
  id        String   @id @default(cuid())
  actorId   String   // userId
  action    String
  target    String?  // entity reference
  metadata  Json?
  hostelId  String?
  createdAt DateTime @default(now())
  @@index([actorId, createdAt])
}
```

**DB Constraints:**
- `Room.capacity >= COUNT(students)` → enforced via application logic + transaction.
- Unique room number per hostel.
- Unique floor number per hostel.
- `student.userId` must have `UserRole(role=student, hostelId)`.

---
## 5) API Design (Express + RBAC)
**Conventions:**
- Base URL: `/api`.
- JSON only. Use 201/200/204 for success, 4xx/5xx for errors.
- All requests must include Clerk JWT in `Authorization` header.
- Use middleware `attachUser()` to map Clerk → DB user + roles.

**Middlewares (TypeScript pseudocode):**
```ts
// attachUser: verify JWT with Clerk and fetch roles
app.use(async (req, res, next) => {
  const token = extractBearer(req.headers.authorization);
  const session = await clerk.verifyToken(token);
  req.user = { id: session.userId };
  req.roles = await prisma.userRole.findMany({ where: { userId: req.user.id } });
  next();
});

const requireRole = (roles: Role[]) => (req, res, next) => {
  const ok = req.roles.some(r => roles.includes(r.role));
  if (!ok) return res.status(403).json({ error: "Forbidden" });
  next();
};

const requireHostel = () => (req, res, next) => {
  // read hostelId from body/params/query and ensure user has binding for it
  const hostelId = req.params.hostelId || req.body.hostelId;
  const ok = req.roles.some(r => r.hostelId === hostelId || r.role === 'admin');
  if (!ok) return res.status(403).json({ error: "Cross-tenant access denied" });
  req.hostelId = hostelId;
  next();
};
```

**Endpoints (MVP):**
- **Admin**
  - `POST /api/admin/hostels` → create hostel { name }
  - `POST /api/admin/hostels/:hostelId/wardens` → assign warden (by clerk user/email)
  - `GET /api/admin/hostels` → list hostels

- **Warden** (scoped)
  - `POST /api/hostels/:hostelId/floors` → create floor
  - `POST /api/hostels/:hostelId/rooms` → create room { floorId, number, capacity }
  - `POST /api/hostels/:hostelId/students` → create student { roomId, collegeId, fatherPhone }
  - `POST /api/hostels/:hostelId/announcements` → create announcement
  - `GET /api/hostels/:hostelId/cleaning-requests` → list & filter
  - `PATCH /api/hostels/:hostelId/cleaning-requests/:id` → update status

- **Student** (scoped)
  - `GET /api/me/room` → my room details + last cleaning
  - `POST /api/hostels/:hostelId/rooms/:roomId/cleaning-requests` → create
  - `GET /api/hostels/:hostelId/announcements` → list

**Errors (JSON):**
```json
{ "error": "Forbidden", "code": "RBAC_403" }
{ "error": "Cross-tenant access denied", "code": "TENANT_403" }
{ "error": "Validation failed", "code": "VAL_400", "details": [...] }
```

---
## 6) Frontend (Next.js App Router)
**Routes:**
- Public: `/login`, `/reset-password` (Clerk components)
- Role Dashboards:
  - `/admin` → Hostels table, Create hostel, Assign wardens
  - `/warden` → Floors/Rooms CRUD, Students CRUD, Announcements, Cleaning queue
  - `/student` → My Room, Request Cleaning, Announcements

**Guarding:**
- Client-side: hide nav by role.
- Server-side: Route handlers call backend APIs with JWT; handle 401/403.
- Use React Query for data fetching, optimistic updates for non-critical.

**UI/UX:**
- Shared layout + design system (shadcn/ui, Tailwind).
- Toasts for success/error; skeleton loaders.
- Tables with filters (hostel, floor, status) and pagination.

---
## 7) Validation & Business Rules
- **Student creation:** requires `collegeId` unique; `room.capacity` not exceeded.
- **Cleaning requests:** cooldown (e.g., max 1 open request per room at a time); status transitions: `PENDING → IN_PROGRESS → DONE/REJECTED`.
- **Announcements:** length limits; HTML sanitized if rich text.
- **Hostel scoping:** all queries must include `hostelId` in where clause.

---
## 8) Security Checklist (Go-Live)
1. Clerk: 2FA for Admin/Warden; session expiry defaults.
2. HTTPS everywhere; HSTS; secure cookies (SameSite=Lax/Strict where applicable).
3. Express: `helmet`, `cors` allowlist, `express-rate-limit` per IP+user.
4. Input validation with Zod on every endpoint.
5. Prisma: least-privileged DB user for app; separate admin migration user.
6. Audit logs for all POST/PATCH/DELETE with actor + payload hash.
7. Error handling: avoid leaking internals; map to standardized error codes.
8. Secrets in env (Clerk keys, DB URL); rotate regularly.
9. Backups: daily automated DB backups; test restore.
10. Monitoring: 500 rate alerts; auth failure spikes; unusual tenant access.

---
## 9) DevOps & Environments
- **Envs:** `local`, `staging`, `prod`.
- **CI/CD:** GitHub Actions → lint, typecheck, test, prisma migrate, deploy.
- **Migrations:** `prisma migrate deploy` gated; seed scripts per env.
- **Feature flags:** simple `features` table or env flags for experimental modules.

**Env Vars (.env):**
```
DATABASE_URL=
CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
JWT_AUDIENCE=
JWT_ISSUER=
RATE_LIMIT_WINDOW=60000
RATE_LIMIT_MAX=100
NODE_ENV=production
```

---
## 10) Sample Flows
**Admin creates hostel:**
1) Admin logs in → `/admin`
2) POST `/api/admin/hostels { name }`
3) POST `/api/admin/hostels/:id/wardens { clerkUserId/email }`

**Warden sets up rooms:**
1) POST floors, rooms
2) POST students (room capacity enforced)

**Student requests cleaning:**
1) POST cleaning request (if none open)
2) Warden updates status; student notified

---
## 11) Testing Strategy
- **Unit:** validation schemas, RBAC middleware, service functions
- **Integration:** API endpoints with mocked Clerk; DB test containers
- **E2E:** Playwright/Cypress for role flows
- **Security tests:** unauthorized access attempts, cross-hostel access

---
## 12) Analytics & Reporting (Phase 2)
- Occupancy rate by hostel/floor
- Cleaning SLA (avg time to complete)
- Request volume heatmaps

---
## 13) Future Extensions
- Staff app for cleaners
- Maintenance tickets (non-cleaning)
- Payments & invoicing
- Visitor management, gate pass QR
- Mobile app (React Native/Expo)

---
## 14) Acceptance Criteria (MVP)
- Admin can create hostel and assign at least one warden.
- Warden can create floors, rooms, and onboard students with capacity checks.
- Student can log in and submit a cleaning request; warden can complete it.
- Announcements visible per hostel.
- All write actions appear in audit logs.
- Unauthorized/cross-tenant access attempts return 403.

---
## 15) Definition of Done (Engineering)
- Typesafe APIs; Zod schemas shared client/server.
- 90%+ endpoint coverage in tests for core flows.
- CI green; zero high-severity vulnerabilities in `npm audit`.
- Load test: sustain 200 rps on read-heavy endpoints with p95 < 300ms.
- Docs updated (README + this plan.txt), envs configured, runbooks ready.

---
## 16) Runbooks (Ops)
- **Rotate Clerk keys:** update env, restart API.
- **DB Migration failure:** rollback tag, restore from latest backup, rerun after fix.
- **Incident response:** page on-call, capture logs & audit trail, postmortem within 48h.

---
## 17) File/Folder Skeleton
```
apps/
  web/ (Next.js)
    app/
      admin/*
      warden/*
      student/*
      api/* (route handlers if needed)
    lib/
    components/
    hooks/
  api/ (Express)
    src/
      index.ts
      middlewares/
      routes/
        admin.ts
        warden.ts
        student.ts
      services/
      schemas/
      utils/
    prisma/
      schema.prisma
    .env

packages/
  ui/ (shared components)
  config/

Dockerfiles, docker-compose.yml, Makefile, README.md
```

---
## 18) Quick Start (Local)
1) `docker compose up -d postgres`
2) `cd apps/api && pnpm dev` (runs Express)
3) `cd apps/web && pnpm dev` (runs Next.js)
4) Set Clerk dev keys; run `prisma migrate dev` and seed.

**Seed Script:** create test admin, hostel, warden, student, rooms.

---
## 19) Sample Seed (pseudo)
```ts
const admin = await ensureUser({ email: 'admin@hms.local' });
await addRole(admin.id, 'admin', null);
const hostel = await prisma.hostel.create({ data: { name: 'Alpha Hostel', createdBy: admin.id }});
const warden = await ensureUser({ email: 'warden@hms.local' });
await addRole(warden.id, 'warden', hostel.id);
await prisma.warden.create({ data: { userId: warden.id, hostelId: hostel.id }});
// floors, rooms, students...
```

---
## 20) Notes
- Do not use fatherPhone as a password. Generate onboarding links or OTP, enforce password reset on first login via Clerk.
- Always filter by `hostelId` in services. Consider service-level helpers that require `hostelId` param to avoid mistakes.
- Keep admin-level operations isolated in code (routes/services) for easier future split if needed.

