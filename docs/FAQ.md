# ITSM-Pro — Frequently Asked Questions (FAQ)

> 25 real questions answered clearly — for developers, sysadmins, and IT managers
> setting up or maintaining the ITSM-Pro platform.

---

## Table of Contents

1. [General & Getting Started](#1-general--getting-started) — Q1–Q5
2. [Authentication & Security](#2-authentication--security) — Q6–Q10
3. [Incidents & SLA](#3-incidents--sla) — Q11–Q15
4. [Change Management & Approvals](#4-change-management--approvals) — Q16–Q18
5. [Database & Migrations](#5-database--migrations) — Q19–Q21
6. [Email Notifications](#6-email-notifications) — Q22–Q23
7. [Deployment & Docker](#7-deployment--docker) — Q24–Q25

---

## 1. General & Getting Started

---

### Q1. What is ITSM-Pro and what problems does it solve?

**ITSM-Pro** is a full-stack, enterprise-grade **IT Service Management** platform that helps IT
departments track, manage, and resolve technology issues systematically. It is inspired by
tools like ServiceNow and Jira Service Management but is fully open-source and self-hosted.

It solves these core problems:

| Problem | How ITSM-Pro Solves It |
|---------|------------------------|
| No central place to report IT issues | Incident ticketing with structured forms and auto-numbering |
| IT teams miss resolution deadlines | Built-in SLA engine with real-time countdown timers and breach alerts |
| Risky changes deployed without review | CAB (Change Advisory Board) approval workflow with multi-approver support |
| No audit trail of who did what | Immutable audit logs on every action |
| No visibility into IT performance | Analytics dashboard with KPI cards and Recharts charts |

---

### Q2. What is the default login for a fresh installation?

On first startup, Flyway migration `V2__Seed_Data.sql` automatically seeds the database with
demo users. The default credentials are:

| Username | Password | Role |
|----------|----------|------|
| `admin` | `admin123` | Administrator (full access) |
| `jsmith` | `admin123` | Manager (approvals, assignments) |
| `mwilson` | `admin123` | Manager |
| `bjohnson` | `admin123` | User (create/view tickets) |
| `slee` | `admin123` | User |

> **Security warning:** Change all default passwords immediately in any non-development environment.
> Use the Admin → User Management page or update directly in the database.

---

### Q3. Can I run ITSM-Pro without Docker?

Yes. Docker is recommended but not required. For local development without Docker:

**Requirements:** Java 21, Maven 3.9+, Node.js 20+, SQL Server 2022 (or Developer Edition)

```bash
# 1. Create the database manually in SQL Server
sqlcmd -S localhost -U sa -P "YourPassword" -Q "CREATE DATABASE itsm_db"

# 2. Start the backend (Flyway runs migrations automatically on startup)
cd backend
export DB_URL="jdbc:sqlserver://localhost:1433;databaseName=itsm_db;encrypt=false;trustServerCertificate=true"
export DB_USERNAME=sa
export DB_PASSWORD=YourStrong@Passw0rd
export JWT_SECRET=bXlTdXBlclNlY3JldEtleUZvckpXVEF1dGhlbnRpY2F0aW9u
mvn spring-boot:run

# 3. Start the frontend (separate terminal)
cd frontend
npm install
npm run dev
```

The app will be at `http://localhost:3000`.

---

### Q4. How long does the first Docker startup take?

Expect **2–4 minutes** on the first run. This is normal because:

1. **SQL Server** takes ~20–30 seconds to fully initialize its system databases
2. **Spring Boot** waits for SQL Server to be healthy (health check loop), then boots (~30 seconds)
3. **Flyway** runs `V1__Initial_Schema.sql`, `V2__Seed_Data.sql`, and `V3__...sql` migrations (~5 seconds)
4. **React/Nginx** starts almost instantly but waits for the backend health check to pass

Subsequent starts (after the first) take ~30–45 seconds because SQL Server data is already initialized.

Monitor progress with: `docker compose logs -f backend`

---

### Q5. What browsers are supported?

ITSM-Pro's React frontend works in all modern browsers. Recommended:

| Browser | Minimum Version | Notes |
|---------|----------------|-------|
| Chrome | 90+ | Best supported, recommended for development |
| Firefox | 88+ | Fully supported |
| Edge | 90+ | Fully supported |
| Safari | 14+ | Supported (test CSS grid layouts) |

Internet Explorer is **not supported**. The app uses modern CSS features (CSS Grid, CSS Custom
Properties) and ES2020 JavaScript that IE cannot execute.

---

## 2. Authentication & Security

---

### Q6. How does JWT authentication work in this project?

ITSM-Pro uses a **stateless JWT (JSON Web Token)** authentication flow:

```
Step 1: POST /api/auth/login  → Server validates username/password with BCrypt
Step 2: Server generates two tokens:
         - Access token:  short-lived (15 minutes), used for API calls
         - Refresh token: long-lived (7 days), used only to get new access tokens
Step 3: Client stores tokens (sessionStorage in this project)
Step 4: Every API request includes:   Authorization: Bearer <access_token>
Step 5: JwtAuthenticationFilter validates the token on every request
Step 6: When access token expires → client calls POST /api/auth/refresh silently
Step 7: New access token issued → original request retried automatically
```

The server never stores sessions. Each request is independently authenticated by the token's
cryptographic signature. This enables horizontal scaling (multiple backend instances) without
a shared session store.

---

### Q7. Why is the JWT secret stored in an environment variable? Can I hardcode it?

**Never hardcode secrets in source code.** If you commit a hardcode secret to Git, it becomes
part of the repository history permanently — even if you delete the file later.

The `JWT_SECRET` environment variable approach means:
- The secret never touches version control
- Different environments (dev, staging, prod) use different secrets
- Rotating the secret only requires restarting the app with a new env variable

**Generate a strong secret:**
```bash
# Option 1: openssl (Linux/Mac/Git Bash on Windows)
openssl rand -base64 32

# Option 2: Python
python3 -c "import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"

# Example output (use YOUR OWN generated value):
# K7hP9mQxLnRzWvYuJbNcFdTsAeGiOkMw3jX6pH8qL0=
```

Set it in `.env`:
```
JWT_SECRET=K7hP9mQxLnRzWvYuJbNcFdTsAeGiOkMw3jX6pH8qL0=
```

---

### Q8. What is BCrypt and why is strength 12 used?

**BCrypt** is a password hashing algorithm designed to be slow by design. Unlike MD5/SHA-1
(which are fast and easily brute-forced), BCrypt makes each comparison expensive.

The **strength (cost factor)** controls how many iterations run: `iterations = 2^strength`

| Strength | Iterations | Time per hash | Security level (2024) |
|----------|-----------|---------------|----------------------|
| 10 | 1,024 | ~100ms | Acceptable |
| **12** | **4,096** | **~250ms** | **Good (our choice)** |
| 14 | 16,384 | ~1,000ms | Strong (slow for UX) |

At strength 12, brute-forcing one account requires testing millions of guesses at 250ms each —
years of compute time for typical passwords. The trade-off: legitimate logins take ~250ms,
which is imperceptible to users.

---

### Q9. What do the three roles (ROLE_USER, ROLE_MANAGER, ROLE_ADMIN) each allow?

| Action | USER | MANAGER | ADMIN |
|--------|------|---------|-------|
| View all incidents | ✅ | ✅ | ✅ |
| Create incidents | ✅ | ✅ | ✅ |
| Update their own incidents' status | ✅ | ✅ | ✅ |
| Update any incident (full edit) | ❌ | ✅ | ✅ |
| Assign incidents to technicians | ❌ | ✅ | ✅ |
| Cancel/delete incidents | ❌ | ❌ | ✅ |
| Create change requests | ✅ | ✅ | ✅ |
| Approve/reject changes (CAB) | ❌ | ✅ | ✅ |
| View approval queue | ❌ | ✅ | ✅ |
| Manage users | ❌ | ❌ | ✅ |
| View admin settings | ❌ | ❌ | ✅ |

---

### Q10. The app shows "401 Unauthorized" for every API call. What is wrong?

This is the most common issue after setup. Check these causes in order:

1. **Token expired and refresh failed** — Log out and log back in. Check browser DevTools
   → Network tab for the `/api/auth/refresh` call and its response.

2. **Wrong JWT_SECRET** — If the backend was restarted with a different secret, all existing
   tokens are invalid (different signature). All users must re-login.

3. **CORS mismatch** — If you changed the frontend URL, update `SecurityConfig.java`'s
   `corsConfigurationSource()` allowed origins list.

4. **Clock skew** — JWT tokens are time-sensitive. If the server clock is wrong by more than
   a few minutes, tokens will appear expired. Run `date` on the server to check.

5. **Missing Authorization header** — Check `client.ts`'s request interceptor is running.
   In DevTools → Network → Request Headers, you should see `Authorization: Bearer eyJ...`

---

## 3. Incidents & SLA

---

### Q11. How is ticket priority calculated automatically?

Priority is calculated using the **ITIL impact × urgency matrix** in `IncidentService.java`.
You never set priority directly — it is derived from two fields you do provide:

- **Impact**: How many users or systems are affected? (HIGH / MEDIUM / LOW)
- **Urgency**: How quickly does the business need the service restored? (HIGH / MEDIUM / LOW)

```
              URGENCY
             HIGH    MEDIUM   LOW
          ┌────────┬────────┬────────┐
     HIGH │  P1    │  P2    │  P3    │
IMPACT    ├────────┼────────┼────────┤
   MEDIUM │  P2    │  P3    │  P4    │
          ├────────┼────────┼────────┤
     LOW  │  P3    │  P4    │  P4    │
          └────────┴────────┴────────┘
```

**Example:** A database server is down (HIGH impact — all users affected) but it's Friday
evening (MEDIUM urgency — can wait until Monday). Priority = **P2**, SLA = 4 hours.

---

### Q12. What are the SLA target times and what happens when they are breached?

| Priority | Meaning | Resolution Target |
|----------|---------|-------------------|
| P1 | Critical — service completely down | **1 hour** |
| P2 | High — major degradation | **4 hours** |
| P3 | Medium — some users affected | **8 hours** |
| P4 | Low — minor issue, workaround exists | **24 hours** |

**When breached:**
1. `SlaService.checkSlaBreaches()` runs every 5 minutes (Quartz `@Scheduled`)
2. `sla_status` column is set to `BREACHED` in the database
3. An escalation email is sent to the assignment group inbox **and** the assigned technician
4. The SLA countdown in the UI turns red and pulses with a "BREACHED" badge
5. The incident appears first in dashboard breach counts

**What does NOT happen:** The incident is not auto-closed. A human must still resolve it.

---

### Q13. Why does the SLA timer show "PAUSED"?

SLA timers automatically pause when a ticket moves to **PENDING** status. This represents
situations where IT is waiting for something outside their control:

- Waiting for a vendor to respond
- Waiting for the user to provide more information
- Waiting for a maintenance window to apply a fix
- Waiting for a third-party system to be available

When the ticket moves back to **IN_PROGRESS**, the SLA timer resumes from where it paused.
The accumulated pause time is stored in `sla_paused_minutes` on the incident record.

In `SlaService.java`, the `sla_paused` flag controls whether the breach checker skips a ticket
during its 5-minute scan cycle.

---

### Q14. Can I change the SLA hours (e.g. make P1 = 30 minutes)?

Yes. Edit `SlaService.java` in the `SLA_HOURS` map:

```java
// In SlaService.java — change any value here
private static final Map<Priority, Long> SLA_HOURS = Map.of(
    Priority.P1,  1L,   // ← change to 0.5 won't work (Long only); use minutes approach
    Priority.P2,  4L,
    Priority.P3,  8L,
    Priority.P4,  24L
);
```

For sub-hour SLAs (e.g. 30 minutes for P1), switch to a minutes-based approach:

```java
// Change calculateDueDate() to use minutes instead of hours
public LocalDateTime calculateDueDate(Priority priority) {
    long minutes = SLA_MINUTES.getOrDefault(priority, 1440L); // 1440 = 24h
    return LocalDateTime.now().plusMinutes(minutes);
}

private static final Map<Priority, Long> SLA_MINUTES = Map.of(
    Priority.P1,  30L,    // 30 minutes
    Priority.P2,  240L,   // 4 hours
    Priority.P3,  480L,   // 8 hours
    Priority.P4,  1440L   // 24 hours
);
```

Also update `SLA_HOURS` references in `updateAtRiskStatus()` to use minutes consistently.

---

### Q15. How does the AT_RISK SLA status work?

**AT_RISK** is a warning state that fires when a ticket has consumed **70% or more** of its SLA
time budget but has not yet breached.

Formula: `(minutes elapsed / total SLA minutes) × 100 ≥ 70%`

Examples:
- P1 (1 hour SLA) → AT_RISK after **42 minutes** have passed
- P2 (4 hour SLA) → AT_RISK after **168 minutes** (2h 48m) have passed
- P3 (8 hour SLA) → AT_RISK after **336 minutes** (5h 36m) have passed

In the UI, AT_RISK tickets show an **amber** countdown timer and an amber "AT RISK" chip.
This gives the assigned technician a heads-up before a full breach occurs.

The threshold (70%) can be changed in `SlaService.updateAtRiskStatus()`:
```java
// Change 0.70 to any percentage you want (e.g. 0.80 for 80%)
if ((double) minutesConsumed / totalSlaMinutes >= 0.70) {
```

---

## 4. Change Management & Approvals

---

### Q16. What is the difference between Standard, Normal, and Emergency change types?

| Type | Definition | CAB Review Required? | Typical Use |
|------|-----------|---------------------|-------------|
| **Standard** | Pre-approved, low-risk, repeatable change | ❌ No — auto-approved | Software installs, routine patching, config presets |
| **Normal** | Requires planning, has moderate-to-high risk | ✅ Yes — full CAB review | Server upgrades, network changes, application deployments |
| **Emergency** | Urgent — abbreviated approval process | ✅ Yes — fast-tracked single approver | Hotfixes for production outages, security patches for critical CVEs |

In ITSM-Pro, all three types go through the same `CAB_REVIEW` status technically, but in
practice your team can configure different approval groups or simply process Emergency changes
with a shorter review window.

---

### Q17. How does the multi-approver CAB workflow function?

When a change is submitted for review (`POST /api/changes/{id}/submit`):

1. The system queries all active users with `ROLE_MANAGER`
2. One `Approval` database record is created per manager with `decision = PENDING`
3. Each manager receives an email with a "Review & Decide" link
4. Managers open the change and click their decision in the Approvals tab

**Decision logic** (in `ChangeManagementService.recordApprovalDecision()`):

```
If ANY approval is REJECTED  → change.status = REJECTED  (immediate, no waiting)
If ALL approvals are APPROVED → change.status = APPROVED  (only when last approver decides)
If some still PENDING         → change.status stays CAB_REVIEW
```

This means a single veto from any CAB member blocks the change — consistent with ITIL's
Change Advisory Board principles where unanimous approval is typically required.

---

### Q18. Can I require only specific people to approve a change, not all managers?

Currently, `ChangeManagementService.submitForReview()` auto-assigns all `ROLE_MANAGER` users
as approvers. To assign specific approvers per change type, modify the service:

```java
// In ChangeManagementService.submitForReview()
// Replace the "find all managers" query with a more targeted one:

List<User> cabMembers;
if (change.getChangeType() == ChangeType.EMERGENCY) {
    // Emergency: only the primary CAB lead (e.g. fixed username)
    cabMembers = userRepository.findByUsername("cab_lead")
        .map(List::of).orElse(List.of());
} else {
    // Normal: all active managers
    cabMembers = userRepository.findByRoleName("ROLE_MANAGER");
}
```

A more flexible approach would be to add a `CabGroup` entity with a many-to-many relationship
to Users, then assign specific CAB members per change type or department in the admin settings.

---

## 5. Database & Migrations

---

### Q19. What is Flyway and why does the project use it instead of Hibernate DDL?

**Flyway** is a database migration tool that manages schema changes through versioned SQL scripts.

**Why not `spring.jpa.hibernate.ddl-auto=create-drop`?**

| Approach | Pros | Cons |
|----------|------|------|
| `ddl-auto=create-drop` | Zero setup | Destroys ALL data on restart. Cannot use in production. |
| `ddl-auto=update` | Auto-updates schema | Unpredictable. Cannot drop columns. No rollback. |
| **Flyway** | Reproducible. Version-controlled. Auditable. | Must write SQL manually |

With Flyway:
- `V1__Initial_Schema.sql` always runs first on a fresh install
- `V2__Seed_Data.sql` seeds demo data exactly once
- New features add `V3__...sql`, `V4__...sql` and so on
- The `flyway_schema_history` table tracks which scripts have run
- If a script was already applied, Flyway skips it — safe for restarts

**Rule:** Never modify a migration file after it has been applied anywhere. Always create a new
versioned file for changes.

---

### Q20. I get "Flyway checksum mismatch" error on startup. How do I fix it?

This error means a migration SQL file was modified **after** Flyway already applied it.
Flyway stores a checksum of each file and detects tampering.

**Common causes:**
- Editing `V1__Initial_Schema.sql` or `V2__Seed_Data.sql` after the first run
- Line ending changes (CRLF vs LF) on Windows
- Extra spaces or BOM characters added by an editor

**Fix options:**

Option A — Development only (repair the checksum):
```bash
mvn flyway:repair -pl backend
# This updates the stored checksum to match the current file
# Only use this in development — never in production with real data
```

Option B — Reset the database (loses all data):
```sql
-- In SQL Server Management Studio
DROP DATABASE itsm_db;
CREATE DATABASE itsm_db;
-- Then restart Spring Boot — Flyway re-runs from V1
```

Option C — Create a new migration to make the change properly:
```sql
-- Instead of editing V1, create V4__Fix_Column_Name.sql
ALTER TABLE incidents ADD COLUMN new_field NVARCHAR(100);
```

---

### Q21. How do I add a new table or column to the database?

Always use a new Flyway migration file. Never edit existing ones.

**Example: Adding an `attachments` table in a future release**

1. Create `backend/src/main/resources/db/migration/V4__Add_Attachments.sql`:

```sql
-- V4__Add_Attachments.sql
-- Purpose: Add file attachment support for incidents
CREATE TABLE attachments (
    id          BIGINT IDENTITY(1,1) PRIMARY KEY,
    incident_id BIGINT NOT NULL REFERENCES incidents(id) ON DELETE CASCADE,
    file_name   NVARCHAR(255) NOT NULL,
    file_size   BIGINT,
    mime_type   NVARCHAR(100),
    storage_path NVARCHAR(500) NOT NULL,
    uploaded_by  BIGINT REFERENCES users(id),
    created_at   DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);
CREATE INDEX idx_attachments_incident ON attachments(incident_id);
```

2. Create the JPA entity `Attachment.java`
3. Add the repository `AttachmentRepository.java`
4. Restart the backend — Flyway applies V4 automatically

---

## 6. Email Notifications

---

### Q22. How do I test emails in development without a real SMTP server?

The project includes **MailHog** in `docker-compose.yml` — a fake SMTP server that captures all
outgoing emails and shows them in a web interface. No emails are actually delivered.

**Access the MailHog UI:** `http://localhost:8025`

All emails sent by the application (incident created, SLA breach alerts, approval requests, etc.)
appear here in real time. You can see the full HTML rendering exactly as recipients would see it.

**MailHog connection settings** (already configured in `application.properties`):
```properties
spring.mail.host=mailhog    # Docker service name
spring.mail.port=1025       # MailHog SMTP port
spring.mail.properties.mail.smtp.auth=false
```

For production, replace MailHog with a real SMTP provider in your `.env` file:
```
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
SMTP_USERNAME=apikey
SMTP_PASSWORD=your-sendgrid-api-key
```

---

### Q23. Some emails are not being received. How do I debug the email pipeline?

Follow this debugging checklist:

**Step 1 — Check the `notifications` database table:**
```sql
SELECT TOP 20 recipient_email, subject, status, error_message, sent_at
FROM notifications
ORDER BY created_at DESC;
```
`status = 'FAILED'` with an `error_message` tells you exactly what went wrong (connection refused,
authentication failed, invalid recipient, etc.).

**Step 2 — Check backend logs for SMTP errors:**
```bash
docker compose logs backend | grep -i "mail\|smtp\|notification"
```

**Step 3 — Verify SMTP settings:**
```bash
# Test SMTP connectivity from inside the backend container
docker compose exec backend curl -v telnet://mailhog:1025
```

**Step 4 — Check that the recipient email is valid** in the `users` table. A malformed email
address in the seed data causes JavaMail to throw `MessagingException`.

**Step 5 — Confirm `@Async` email sending is working.** If the `itsm-async-*` thread pool is
exhausted (more than 50 emails queued), new emails are rejected. Scale up `AsyncConfig.java`'s
`queueCapacity` or `maxPoolSize` for high-volume environments.

---

## 7. Deployment & Docker

---

### Q24. How do I deploy ITSM-Pro to a production server?

**Minimum production server requirements:**
- 2 vCPU, 4 GB RAM, 40 GB SSD
- Ubuntu 22.04 LTS or RHEL 8+ (or any Linux with Docker support)
- Ports 80 and 443 open (for Nginx)
- Port 1433 NOT publicly exposed (SQL Server internal only)

**Quick production deployment:**
```bash
# 1. Clone the repository on your server
git clone https://github.com/your-org/enterprise-itsm.git
cd enterprise-itsm

# 2. Configure production environment
cp .env.example .env
nano .env   # Fill in real DB password, JWT secret, SMTP settings

# 3. Start with production compose
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build

# 4. Verify all containers are healthy
docker compose ps
docker compose logs backend | tail -20
```

**Production checklist:**
- [ ] Changed `JWT_SECRET` from the default
- [ ] Changed `DB_PASSWORD` from `YourStrong@Passw0rd`
- [ ] Swagger UI disabled (`springdoc.api-docs.enabled=false` in `application-prod.properties`)
- [ ] HTTPS configured on Nginx (Let's Encrypt or corporate certificate)
- [ ] `APP_BASE_URL` set to your actual domain for email deep links
- [ ] Database backup scheduled (daily cron job)

---

### Q25. How do I back up and restore the SQL Server database?

**Create a backup:**
```bash
# Run inside the sqlserver container
docker compose exec sqlserver /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U sa -P "$DB_PASSWORD" -No \
  -Q "BACKUP DATABASE itsm_db TO DISK = '/tmp/itsm_backup_$(date +%Y%m%d).bak' WITH COMPRESSION, STATS = 10"

# Copy the backup file out of the container to the host
docker compose cp sqlserver:/tmp/itsm_backup_$(date +%Y%m%d).bak ./backups/
```

**Restore from backup:**
```bash
# Copy the backup into the container
docker compose cp ./backups/itsm_backup_20240115.bak sqlserver:/tmp/

# Restore
docker compose exec sqlserver /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U sa -P "$DB_PASSWORD" -No \
  -Q "RESTORE DATABASE itsm_db FROM DISK = '/tmp/itsm_backup_20240115.bak' WITH REPLACE"
```

**Automate daily backups with cron (Linux):**
```bash
# Add to crontab: crontab -e
0 2 * * * cd /opt/enterprise-itsm && docker compose exec -T sqlserver \
  /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "$DB_PASSWORD" -No \
  -Q "BACKUP DATABASE itsm_db TO DISK='/tmp/itsm_$(date +\%Y\%m\%d).bak' WITH COMPRESSION" \
  && docker compose cp sqlserver:/tmp/itsm_$(date +%Y%m%d).bak ./backups/ 2>&1 | logger -t itsm-backup
```

---

*ITSM-Pro FAQ · v1.0.0 · Last updated: May 2026*
