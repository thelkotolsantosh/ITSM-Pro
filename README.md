# IT Ticketing System Clone - ServiceNow Style

A full-stack ITSM tool with Incident Management, Change Management, SLA, and Approvals.

## Tech Stack
- **Backend**: Spring Boot 3.3.4, Java 21
- **Frontend**: React 18.3.1
- **Database**: SQL Server 2022
- **Auth**: JWT + Spring Security

## Features
1. **Login system** - JWT based, roles: User, Manager, Admin
2. **Incident Management** - Create INC tickets, auto-numbering INC0010001
3. **Priority assignment** - P1 to P4 with different SLA
4. **SLA timers** - Live countdown + auto breach detection + email
5. **Status workflow** - ServiceNow style state model
6. **Change Management** - CHG tickets with CAB approval flow
7. **Manager approvals** - Changes need approval before implementation
8. **Email notifications** - SMTP on create/assign/breach
9. **Dashboard analytics** - Charts for SLA %, ticket volume, MTTR

## Setup
1. Clone repo: `git clone...`
2. DB: Run `database/schema.sql` in SSMS
3. Backend: `cd backend && mvn spring-boot:run`
4. Frontend: `cd frontend && npm install && npm start`
5. Login: admin/admin123
