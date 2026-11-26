# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

Pre-implementation. The only source of truth so far is `project-scope.md` (features, data model, out-of-scope list, success criteria). Read it before making design decisions, and update this file once the backend/frontend projects are scaffolded (actual directory layout, commands, conventions).

## Tech Stack

- **Backend:** Spring Boot REST API, Java 21, Maven, Spring Data JPA, PostgreSQL
- **Frontend:** React + TypeScript (Vite), Tailwind CSS, shadcn/ui, native `fetch` (no axios / TanStack Query)
- **Runtime:** local only; the whole app (backend, frontend, database) should start with a single command (e.g., Docker Compose)
- **Undecided:** database migration tool (Flyway / Liquibase / JPA `ddl-auto`) — ask before choosing

## Commands

Standard commands for this stack (verify against the actual project once scaffolded):

- Backend: `./mvnw spring-boot:run`, `./mvnw test`, single test: `./mvnw test -Dtest=ClassName#methodName`
- Frontend: `npm run dev`, `npm run build`, `npm run lint`

## Domain Rules (non-obvious, from project-scope.md)

- **Statuses:** `SAVED`, `APPLIED`, `INTERVIEWING`, `OFFER`, `REJECTED` only. `SAVED` means "planning to apply".
- **Every status change must insert a `STATUS_HISTORY` row.** Dashboard counts (Applied / Interviews / Offers / Rejected) are computed from history — a job counts toward every stage it has *ever* reached, not just its current status.
- **Upcoming deadlines** = jobs with status `SAVED` and `deadline` within the next 3 days.
- **All Jobs page:** paginated 10 per page; search covers job title, company name, and job description; filter by status; sort by deadline, applied date, or company name.
- **Interviews** are a separate `INTERVIEWS` table (multiple rounds per job), not a field on `JOBS`.
- **Resume / cover letter** are stored as text version labels (`resume_version`, `cover_letter_version`), not files.
- **Single user in v1, no auth** — but keep the schema ready for a future `USERS` table and `user_id` foreign keys.

## Priorities

This is a learning / portfolio project: favor clean architecture and backend test coverage (status changes, dashboard counts, upcoming deadlines, search) over speed. Do not build items listed under "Out of Scope (v1)" in `project-scope.md` unless asked.
