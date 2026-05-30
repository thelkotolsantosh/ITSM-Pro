# ITSM-Pro — Enterprise IT Service Management System

> A production-grade, full-stack ITSM platform built with Spring Boot 3, React 18, and SQL Server 2022. Inspired by ServiceNow and Jira Service Management.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                          Browser                                │
│                    React 18 + TypeScript                        │
│           (MUI · Recharts · React Query · Zustand)              │
└──────────────────────┬──────────────────────────────────────────┘
                       │ HTTPS / REST JSON
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Nginx Reverse Proxy                        │
│          /api/* → Spring Boot   /  /* → React SPA               │
└────────────┬──────────────────────────────────────────────────--┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│                 Spring Boot 3 REST API (Java 21)                │
│  Security · JWT · JPA · Quartz · JavaMail · MapStruct · Flyway  │
└────────────────────────────┬───────────────────────────────────┘
                             │ JDBC / Hibernate
                             ▼
┌────────────────────────────────────────────────────────────────┐
│                    SQL Server 2022                              │
│     users · incidents · changes · approvals · sla_tracking     │
└────────────────────────────────────────────────────────────────┘
```

## Features

### Incident Management (ServiceNow-style)
- Auto ticket numbering: `INC0010001`, `INC0010002`...
- Priority matrix: Impact × Urgency → P1/P2/P3/P4
- Status lifecycle: New → Assigned → In Progress → Pending → Resolved → Closed
- Real-time SLA countdown timers (updates every second in the UI)

### SLA Engine
| Priority | SLA Target | Description |
|----------|-----------|-------------|
| P1 | 1 hour | Critical — service down |
| P2 | 4 hours | High — major impact |
| P3 | 8 hours | Medium — degraded service |
| P4 | 24 hours | Low — minor issue |

- Automatic breach detection (Quartz scheduler, every 5 minutes)
- SLA pause/resume when tickets enter Pending status
- AT_RISK status at 70% SLA consumption
- Escalation email on breach

### Change Management (ITIL)
- Types: Standard / Normal / Emergency
- Full CAB approval workflow (multi-approver, staged)
- Lifecycle: Draft → Submitted → CAB Review → Approved → Scheduled → Implemented → Closed
- Implementation plan, rollback plan, test plan fields

### Analytics Dashboard
- KPI cards: open incidents, SLA compliance %, MTTR, pending changes
- Recharts visualizations: area charts, pie charts, bar charts, line charts
- Live data, refreshes every 60 seconds

### Authentication & RBAC
| Role | Permissions |
|------|------------|
| ROLE_ADMIN | Full access, user management |
| ROLE_MANAGER | Approve changes, assign tickets, view all |
| ROLE_USER | Create/view own incidents, update assigned |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Java 21, TypeScript |
| Backend | Spring Boot 3.3.4, Spring Security, Spring Data JPA |
| Frontend | React 18.3.1, Vite, MUI v6 |
| Database | SQL Server 2022, Flyway migrations |
| Auth | JWT (JJWT 0.12.6), BCrypt |
| State | Zustand, React Query |
| Charts | Recharts 2 |
| Email | JavaMailSender + Thymeleaf templates |
| Scheduler | Quartz (SLA breach detection) |
| Docs | SpringDoc OpenAPI / Swagger UI |
| Container | Docker, Docker Compose, Nginx |
| CI/CD | GitHub Actions |

---

## Quick Start

### Option 1: Docker Compose (Recommended)

```bash
# 1. Clone the repository
git clone https://github.com/your-org/enterprise-itsm.git
cd enterprise-itsm

# 2. Copy and configure environment variables
cp .env.example .env
# Edit .env if you want custom passwords/secrets

# 3. Start everything (SQL Server + Backend + Frontend + MailHog)
docker compose up --build

# Wait ~2 minutes for SQL Server + Spring Boot to fully initialize.
# Flyway runs migrations and seeds demo data automatically.
```

Access the app:
| Service | URL |
|---------|-----|
| ITSM-Pro Web UI | http://localhost:3000 |
| Swagger API Docs | http://localhost:8080/swagger-ui.html |
| MailHog (dev email) | http://localhost:8025 |
| SQL Server | localhost:1433 |

### Default Login Credentials

| Username | Password | Role |
|----------|----------|------|
| admin | admin123 | Administrator |
| jsmith | admin123 | Manager |
| mwilson | admin123 | Manager |
| bjohnson | admin123 | User |
| slee | admin123 | User |

---

### Option 2: Local Development (No Docker)

#### Prerequisites
- Java 21 (e.g. `sdk install java 21.0.4-tem`)
- Node.js 20+ (`nvm install 20`)
- SQL Server 2022 (or Developer Edition)
- Maven 3.9+

#### 1. Database Setup
```sql
-- Run in SQL Server Management Studio or sqlcmd
CREATE DATABASE itsm_db;
-- Flyway will create all tables and seed data on first backend startup
```

#### 2. Backend
```bash
cd backend

# Set environment variables (or update application.properties)
export DB_URL="jdbc:sqlserver://localhost:1433;databaseName=itsm_db;encrypt=false;trustServerCertificate=true"
export DB_USERNAME=sa
export DB_PASSWORD=YourStrong@Passw0rd
export JWT_SECRET=bXlTdXBlclNlY3JldEtleUZvckpXVEF1dGhlbnRpY2F0aW9u

# Start the Spring Boot application
mvn spring-boot:run
# API available at: http://localhost:8080
# Swagger UI at:    http://localhost:8080/swagger-ui.html
```

#### 3. Frontend
```bash
cd frontend

# Install dependencies
npm install

# Start the Vite dev server (proxies /api to localhost:8080)
npm run dev
# UI available at: http://localhost:3000
```

---

## Project Structure

```
enterprise-itsm/
├── backend/
│   ├── src/main/java/com/itsm/
│   │   ├── ItsmApplication.java         # Entry point
│   │   ├── config/
│   │   │   └── SecurityConfig.java      # Spring Security + JWT + CORS
│   │   ├── controller/
│   │   │   ├── AuthController.java      # POST /api/auth/login|refresh|logout
│   │   │   ├── IncidentController.java  # CRUD /api/incidents
│   │   │   ├── ChangeController.java    # CRUD /api/changes + approval flow
│   │   │   └── DashboardController.java # GET /api/dashboard
│   │   ├── service/
│   │   │   ├── IncidentService.java     # Business logic, SLA calc, state machine
│   │   │   ├── ChangeManagementService.java  # CAB workflow, approval logic
│   │   │   ├── SlaService.java          # SLA engine + @Scheduled breach checker
│   │   │   ├── NotificationService.java # @Async email via JavaMail + Thymeleaf
│   │   │   ├── AuthService.java         # Login, JWT generation, refresh
│   │   │   ├── DashboardService.java    # Analytics aggregation
│   │   │   └── AuditLogService.java     # Immutable audit trail
│   │   ├── entity/                      # JPA entities (mapped to DB tables)
│   │   ├── repository/                  # Spring Data JPA repositories
│   │   ├── dto/                         # Request/Response DTOs (no entity exposure)
│   │   ├── mapper/                      # MapStruct entity↔DTO mappers
│   │   ├── security/
│   │   │   ├── JwtUtils.java            # Token generation + validation
│   │   │   └── JwtAuthenticationFilter.java  # Per-request JWT check
│   │   └── exception/
│   │       └── GlobalExceptionHandler.java   # Unified JSON error responses
│   ├── src/main/resources/
│   │   ├── application.properties       # All configuration
│   │   ├── db/migration/
│   │   │   ├── V1__Initial_Schema.sql   # Table DDL
│   │   │   └── V2__Seed_Data.sql        # Demo users, incidents, changes
│   │   └── templates/email/             # Thymeleaf HTML email templates
│   ├── pom.xml
│   └── Dockerfile
│
├── frontend/
│   ├── src/
│   │   ├── api/
│   │   │   ├── client.ts               # Axios instance + JWT interceptors + refresh
│   │   │   └── services.ts             # Typed API functions for every endpoint
│   │   ├── store/
│   │   │   └── authStore.ts            # Zustand auth state (user, tokens, login/logout)
│   │   ├── components/common/
│   │   │   ├── AppLayout.tsx           # Sidebar + TopBar shell
│   │   │   └── ProtectedRoute.tsx      # Auth guard wrapper
│   │   └── pages/
│   │       ├── LoginPage.tsx           # Auth form with React Hook Form
│   │       ├── DashboardPage.tsx       # KPI cards + Recharts analytics
│   │       ├── IncidentListPage.tsx    # Filterable/paginated table
│   │       ├── IncidentDetailPage.tsx  # Live SLA timer + status transitions
│   │       ├── CreateIncidentPage.tsx  # New incident form + priority preview
│   │       ├── ChangeListPage.tsx      # Change request table
│   │       ├── ChangeDetailPage.tsx    # CAB approval panel
│   │       ├── CreateChangePage.tsx    # New change form
│   │       ├── ApprovalsPage.tsx       # Manager approvals queue
│   │       └── UserManagementPage.tsx  # Admin user CRUD
│   ├── App.tsx                         # Routing + MUI theme + providers
│   ├── main.tsx                        # React entry point
│   ├── index.html
│   ├── vite.config.ts                  # Dev server + proxy config
│   ├── nginx.conf                      # Production Nginx config
│   └── Dockerfile
│
├── database/
│   └── (Flyway migrations in backend/src/main/resources/db/migration/)
│
├── .github/workflows/
│   ├── backend-ci.yml                  # Maven test + Docker build
│   └── frontend-ci.yml                 # npm lint + build + Docker push
│
├── docker-compose.yml                  # Full stack orchestration
├── .env.example                        # Environment variable template
└── README.md
```

---

## API Reference

Full interactive docs at `http://localhost:8080/swagger-ui.html`

### Authentication
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/login` | Login with username/password |
| POST | `/api/auth/refresh` | Refresh expired access token |
| POST | `/api/auth/logout` | Logout (client clears tokens) |

### Incidents
| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| GET | `/api/incidents` | List (filterable, paginated) | All |
| GET | `/api/incidents/{id}` | Get by ID | All |
| POST | `/api/incidents` | Create incident | All |
| PUT | `/api/incidents/{id}` | Update | Manager+ |
| PATCH | `/api/incidents/{id}/status` | Change status | All |
| PATCH | `/api/incidents/{id}/assign` | Assign technician | Manager+ |
| DELETE | `/api/incidents/{id}` | Cancel | Admin |

### Changes
| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| GET | `/api/changes` | List (filterable, paginated) | All |
| POST | `/api/changes` | Create change | All |
| POST | `/api/changes/{id}/submit` | Submit for CAB | All |
| POST | `/api/changes/{id}/approve` | Record decision | Manager+ |
| PATCH | `/api/changes/{id}/status` | Update status | Manager+ |

### Dashboard
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/dashboard?period=30` | All dashboard metrics |

---

## Security Notes

- JWT access tokens expire in 15 minutes (configurable)
- Refresh tokens expire in 7 days
- Passwords hashed with BCrypt (strength 12)
- SQL Server connection uses encrypted JDBC
- No credentials stored in source code (env variables only)
- Production checklist:
  - [ ] Change `JWT_SECRET` to a new random value
  - [ ] Change `DB_PASSWORD` from default
  - [ ] Enable HTTPS on Nginx
  - [ ] Set `springdoc.api-docs.enabled=false` in prod
  - [ ] Set `spring.jpa.show-sql=false`

---

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Commit: `git commit -m 'feat: add my feature'`
4. Push and open a Pull Request

---

*ITSM-Pro v1.0.0 · Enterprise Edition · Built with Spring Boot 3 + React 18*
