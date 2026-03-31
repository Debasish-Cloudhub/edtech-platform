# 🎓 EdTech Platform — Complete Documentation

## Table of Contents
1. [Project Overview](#overview)
2. [Architecture](#architecture)
3. [API Reference](#api-reference)
4. [Incentive Calculation Logic](#incentive-logic)
5. [Excel Format Guide](#excel-format)
6. [Railway Deployment](#railway-deployment)
7. [Hostinger Deployment](#hostinger-deployment)
8. [Migration Railway → Hostinger](#migration)
9. [Test Credentials](#test-credentials)

---

## 1. Project Overview {#overview}

A production-ready B2C education consulting platform for BTech & MBA admissions.

**4 portals, 1 shared backend:**

| Subdomain | Portal | Access |
|-----------|--------|--------|
| `student.domain.com` | Student — browse colleges/courses | Public + Student role |
| `sales.domain.com` | CRM — manage leads, incentives | Sales Agent, Counselor |
| `admin.domain.com` | Admin — full control, Excel upload | Admin, Super Admin |
| `expenses.domain.com` | Expense tracking & approval | Finance, Sales |

**Tech Stack:**
- Backend: NestJS + Prisma + PostgreSQL
- Frontend: Next.js 14 (App Router) + Tailwind CSS
- Auth: JWT + RBAC (Role-Based Access Control)
- Crons: @nestjs/schedule (SLA check 8AM, Sulekha sync 6AM)

---

## 2. Architecture {#architecture}

```
┌─────────────────────────────────────────────────────┐
│                    Nginx (reverse proxy)              │
│   student.* │ sales.* │ admin.* │ expenses.*         │
└──────────┬──────────────────────────────────────────┘
           │
    ┌──────▼──────┐         ┌──────────────┐
    │  Next.js    │         │  NestJS API  │
    │  Frontend   │────────▶│  :4000       │
    │  :3000      │         └──────┬───────┘
    └─────────────┘                │
                            ┌──────▼───────┐
                            │  PostgreSQL  │
                            │  (Prisma)    │
                            └──────────────┘
```

**User Roles & Permissions:**

| Role | Leads | Users | Excel | Expenses | Analytics |
|------|-------|-------|-------|----------|-----------|
| SUPER_ADMIN | All | Full CRUD | Upload | Approve | Full |
| ADMIN | All | Full CRUD | Upload | Approve | Full |
| SALES_AGENT | Own only | None | None | Own only | Own |
| COUNSELOR | Own only | None | None | Own only | Own |
| FINANCE_MANAGER | None | None | None | Approve | Financial |
| STUDENT | None | None | None | None | None |

---

## 3. API Reference {#api-reference}

**Base URL:** `http://localhost:4000/api/v1`
**Swagger UI:** `http://localhost:4000/api/docs`
**Auth:** Bearer token in `Authorization` header

### Authentication
```
POST /auth/login         { email, password }  → { access_token, user }
POST /auth/register      { name, email, password, role? }
GET  /auth/me            [JWT required] → user profile
```

### Leads / CRM
```
GET    /leads                     List leads (filtered by role)
POST   /leads                     Create lead
GET    /leads/stats               Lead statistics
GET    /leads/sla-report          SLA breach report (admin)
GET    /leads/incentive-summary   Incentive payout summary
POST   /leads/incentive-preview   { courseId } → preview incentive
GET    /leads/:id                 Lead detail + activities
PUT    /leads/:id                 Update lead (stage, college, course)
PATCH  /leads/:id/note            { note } → add note
PATCH  /leads/:id/assign          { assignedUserId } → reassign (admin)
GET    /leads/:id/sla             SLA status for lead
GET    /leads/:id/activities      Activity log
POST   /leads/recalculate-incentives  Recalc all incentives (admin)
```

### Colleges & Courses (Public)
```
GET  /colleges              List colleges (search, countryId)
GET  /colleges/countries    List all countries
GET  /colleges/:id          College detail + courses
GET  /courses               List courses (search, type, countryId)
GET  /courses/types         Distinct course types
GET  /courses/:id           Course detail
```

### Excel
```
POST /excel/upload          Upload .xlsx → import colleges/courses (admin)
GET  /excel/template        Download template file (admin)
GET  /excel/export-leads    Export leads to Excel
```

### Expenses
```
GET    /expenses                 List expenses
POST   /expenses                 Log expense
PATCH  /expenses/:id/approve     Approve (admin/finance)
PATCH  /expenses/:id/reject      Reject
GET    /expenses/summary/:year   Monthly breakdown
```

### Dashboard & Analytics
```
GET  /dashboard/admin          Admin overview stats
GET  /dashboard/sales          Sales agent dashboard
GET  /dashboard/funnel         Lead conversion funnel
GET  /dashboard/revenue-expense Revenue vs expense by month
GET  /dashboard/config          System config
PUT  /dashboard/config/:key     Update config
```

### Sulekha
```
POST /sulekha/sync             Manual sync trigger (admin)
GET  /sulekha/history          Sync history
```

### Notifications
```
GET   /notifications           My notifications
GET   /notifications/unread-count
PATCH /notifications/:id/read
PATCH /notifications/read-all
```

### Users (Admin)
```
GET   /users                   List users (role filter)
GET   /users/sales-agents      Sales agents for assignment dropdown
POST  /users                   Create user
PUT   /users/:id               Update user
PATCH /users/:id/deactivate    Deactivate user
```

---

## 4. Incentive Calculation Logic {#incentive-logic}

### Priority Order (highest → lowest):

```
Priority 1: Excel — Incentive Amount column (fixed override)
   → Use exact value from spreadsheet
   → Source: EXCEL

Priority 2: Excel — Incentive % column
   → Incentive = (Fees × Incentive%) / 100
   → Source: EXCEL

Priority 3: System default %
   → Configurable via Admin Settings → DEFAULT_INCENTIVE_PERCENT
   → Default: 5%
   → Incentive = (Fees × defaultPercent) / 100
   → Source: CALCULATED
```

### When Incentive is Calculated:
1. **Lead creation** — if college + course selected upfront
2. **Lead update** — whenever college/course changes
3. **Stage → WON** — finalized and locked with feesAtClosure
4. **Excel re-upload** — bulk recalculate all non-WON leads
5. **Manual recalc** — via Admin Settings → Recalculate button

### Example:
```
Course: MBA General Management — IIM Ahmedabad
Fees: ₹25,00,000
Excel Incentive %: 4%
Incentive = 25,00,000 × 4% = ₹1,00,000
```

---

## 5. Excel Format Guide {#excel-format}

Download the template from: Admin → Settings → Download Template

| Column | Header | Required | Description |
|--------|--------|----------|-------------|
| A | College Name | ✅ | Full college name |
| B | Country | ✅ | India / USA / UK / Canada / Germany |
| C | City | ❌ | City name |
| D | Course Name | ✅ | Full course name |
| E | Course Type | ✅ | BTech / MBA / MS / ME / MCA |
| F | Duration | ❌ | "4 years", "2 years" |
| G | Fees | ✅ | Number only (e.g. 800000) |
| H | Currency | ❌ | INR / USD / GBP (default: INR) |
| I | Incentive % | ❌ | e.g. 5 = 5% |
| J | Incentive Amount | ❌ | Fixed override (takes priority over %) |

**Rules:**
- Row 1 is always the header — do not delete
- If both Incentive% and Amount present → Amount wins
- Uploading overwrites existing College+Course combinations
- After upload → all active lead incentives auto-recalculate

---

## 6. Railway Deployment (Testing) {#railway-deployment}

### Prerequisites
- Railway account + CLI: `npm install -g @railway/cli`
- GitHub repo with the project

### Steps

```bash
# 1. Login
railway login

# 2. Create new project
railway init

# 3. Add PostgreSQL database
railway add postgres

# 4. Deploy backend
cd backend
railway up --service backend

# 5. Set backend environment variables (Railway Dashboard)
DATABASE_URL        = (auto-filled by Railway Postgres plugin)
JWT_SECRET          = your-secret-min-32-chars
NODE_ENV            = production
ALLOWED_ORIGINS     = https://your-frontend.railway.app
PORT                = 4000

# 6. Deploy frontend
cd ../frontend
railway up --service frontend

# 7. Set frontend environment variables
NEXT_PUBLIC_API_URL = https://your-backend.railway.app/api/v1
NEXT_PUBLIC_PORTAL  = admin

# 8. Run migrations + seed
railway run --service backend npx prisma migrate deploy
railway run --service backend npx ts-node prisma/seed.ts
```

### Railway Service URLs (auto-generated):
- Backend:  `https://edtech-backend-production.up.railway.app`
- Frontend: `https://edtech-frontend-production.up.railway.app`

---

## 7. Hostinger Deployment (Production) {#hostinger-deployment}

### Hostinger VPS Setup

```bash
# 1. SSH into your Hostinger VPS
ssh root@YOUR_VPS_IP

# 2. Install Docker & Docker Compose
curl -fsSL https://get.docker.com | sh
apt-get install -y docker-compose-plugin

# 3. Clone your repository
git clone https://github.com/yourname/edtech-platform.git
cd edtech-platform

# 4. Create .env file
cp backend/.env.example .env
nano .env  # Fill in your values

# 5. Build and start all services
docker compose up -d --build

# 6. Run migrations
docker compose exec backend npx prisma migrate deploy

# 7. Seed database (first time only)
docker compose exec backend npx ts-node prisma/seed.ts
```

### SSL Certificates (Let's Encrypt)

```bash
# Install certbot
apt-get install -y certbot

# Get certificates for all subdomains
certbot certonly --standalone \
  -d student.yourdomain.com \
  -d sales.yourdomain.com \
  -d admin.yourdomain.com \
  -d expenses.yourdomain.com

# Copy certs to deployment folder
mkdir -p deployment/ssl
cp /etc/letsencrypt/live/student.yourdomain.com/fullchain.pem deployment/ssl/
cp /etc/letsencrypt/live/student.yourdomain.com/privkey.pem deployment/ssl/

# Update nginx.conf — replace "yourdomain.com" with actual domain
sed -i 's/yourdomain.com/YOURACTUAL.COM/g' deployment/nginx.conf

# Start with nginx
docker compose --profile production up -d
```

### DNS Records (Hostinger DNS Manager)

```
Type  Name      Value
A     student   YOUR_VPS_IP
A     sales     YOUR_VPS_IP
A     admin     YOUR_VPS_IP
A     expenses  YOUR_VPS_IP
```

### Auto-renew SSL

```bash
# Add to crontab
crontab -e
# Add this line:
0 3 * * * certbot renew --quiet && docker compose restart nginx
```

---

## 8. Migration Railway → Hostinger {#migration}

```bash
# Step 1: Export DB from Railway
railway run pg_dump $DATABASE_URL > backup.sql

# Step 2: Copy backup to Hostinger
scp backup.sql root@YOUR_VPS_IP:/root/edtech-platform/

# Step 3: Import to Hostinger PostgreSQL
docker compose exec -T db psql -U edtech_user -d edtech_db < backup.sql

# Step 4: Update environment variables
# Change DATABASE_URL, ALLOWED_ORIGINS, API URLs

# Step 5: Update frontend API URL
# NEXT_PUBLIC_API_URL = https://admin.yourdomain.com/api/v1

# Step 6: Restart services
docker compose restart

# Step 7: Verify
curl https://admin.yourdomain.com/api/v1/auth/me
```

**No Railway lock-in:** All config is environment-variable based.
Switch deployments by updating only `.env` values.

---

## 9. Test Credentials {#test-credentials}

| Role | Email | Password |
|------|-------|----------|
| Admin | admin@test.com | Test@1234 |
| Sales Agent | sales@test.com | Test@1234 |
| Student | student@test.com | Test@1234 |
| Finance Manager | finance@test.com | Test@1234 |

---

## 10. SLA Configuration

| Stage | Default SLA | Configurable |
|-------|-------------|--------------|
| INITIATED | 7 days | ✅ via Admin Settings |
| IN_PROGRESS | 15 days | ✅ via Admin Settings |
| WON | — | N/A |
| LOST | — | N/A |

**SLA Cron:** Runs daily at 8:00 AM
- Marks breached leads (`slaBreach = true`)
- Sends notification to assigned user
- Sends daily summary to all admins
- Tracks near-breach (≤2 days remaining)

**Sulekha Cron:** Runs daily at 6:00 AM
- Dedup by: sulekhaId → phone → email
- Auto-assigns to agent with fewest active leads
- Retries up to 3 times on API failure
- Full sync history logged in DB
