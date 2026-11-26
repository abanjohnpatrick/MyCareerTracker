# My Career Tracker

## Problem
Most working people struggle with job hunting. They commonly search LinkedIn, Indeed, and other job portals. The issue is that they often lose track of the jobs they have applied to: what the job description said, which URL it was listed at, and which resume / cover letter they submitted. Sometimes they plan to apply for a job later, but by the time they remember, the deadline has already passed.

## Solution
A personal job tracking portal to store company details, job details, application status, applied date, deadline, interviews, notes, and the resume / cover letter version submitted for each application.

## User
- **v1:** Single user (me). No login or accounts.
- **Future:** Multi-user support. The data model should allow adding a `USERS` table and `user_id` foreign keys later without major restructuring.

## Purpose
Learning / portfolio project. Prioritize clean architecture, good practices, and test coverage over speed of delivery.

## Tech Stack
- **Backend:** Spring Boot (REST API) with Java 21
- **Backend Build Tool:** Maven
- **ORM:** Spring Data JPA
- **Frontend:** React with TypeScript (built with Vite)
- **Data Fetching:** Native Fetch API
- **Styling:** Tailwind CSS
- **UI Components:** shadcn/ui
- **Database:** PostgreSQL
- **Hosting:** Local only (e.g., Docker Compose for backend + frontend + database)

## Features

### Dashboard
- Summary cards showing: **Total Applied**, **Interviews**, **Offers**, **Rejected**, and **Deadlines Coming Soon**.
  - Counts are based on status history: a job counts toward a stage if it has *ever* reached that stage (e.g., a job that went Interviewing → Rejected counts in both Interviews and Rejected).
- A list of jobs with upcoming deadlines so no application is missed.
  - **Upcoming deadline** = job with status `Saved` and a deadline within the next **3 days**.

### All Jobs Page
- Lists all job entries, **10 per page** by default (paginated).
- **Search** by job title, company name, and job description.
- **Filter** by status.
- **Sort** by deadline, applied date, or company name.

### Job Detail Page
- Shows full job information: description, URL, source, company, status, dates, resume / cover letter version.
- Shows the job's notes, interviews, and status history.

### Data Management (CRUD)
- Create, edit, and delete jobs.
- Update a job's status (each change is recorded in status history).
- Create, edit, and delete companies.
- Add, edit, and delete notes on a job.
- Add, edit, and delete interviews on a job.

## Job Statuses
| Status | Meaning |
|---|---|
| `SAVED` | Planning to apply |
| `APPLIED` | Application submitted |
| `INTERVIEWING` | In the interview process |
| `OFFER` | Received an offer |
| `REJECTED` | Application rejected |

## Data

PostgreSQL types. All primary keys are `BIGINT GENERATED ALWAYS AS IDENTITY` (JPA: `@GeneratedValue(strategy = GenerationType.IDENTITY)`). All timestamps are `TIMESTAMPTZ` (Java: `Instant` / `OffsetDateTime`); calendar dates are `DATE` (Java: `LocalDate`). Enum-like columns are stored as `VARCHAR` strings (JPA: `@Enumerated(EnumType.STRING)`) with a `CHECK` constraint.

### Relationships
```
COMPANY 1 ──< JOBS 1 ──< NOTES
                   1 ──< INTERVIEWS
                   1 ──< STATUS_HISTORY
```

### COMPANY
| Column | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | **PK**, identity | |
| company_name | VARCHAR(255) | NOT NULL, UNIQUE | Uniqueness becomes `(user_id, company_name)` when multi-user is added |
| about | TEXT | NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### JOBS
| Column | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | **PK**, identity | |
| company_id | BIGINT | **FK** → COMPANY.id, NOT NULL, ON DELETE RESTRICT | A company with jobs cannot be deleted |
| job_title | VARCHAR(255) | NOT NULL | |
| job_description | TEXT | NULL | Stored in full in case the posting is taken down |
| status | VARCHAR(20) | NOT NULL, DEFAULT `'SAVED'`, CHECK IN (`SAVED`, `APPLIED`, `INTERVIEWING`, `OFFER`, `REJECTED`) | Current status |
| source | VARCHAR(100) | NULL | e.g., LinkedIn, Indeed, Company Website, Referral |
| job_url | VARCHAR(2048) | NULL | |
| applied_date | DATE | NULL | Expected to be set once status leaves `SAVED` (validated in the service layer) |
| deadline | DATE | NULL | |
| resume_version | VARCHAR(100) | NULL | Text label, e.g., `Resume_v3_Backend` |
| cover_letter_version | VARCHAR(100) | NULL | Text label |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

Indexes: `company_id`, `status`, `deadline`, `applied_date` (support filter, sort, and the upcoming-deadlines query).

### NOTES
| Column | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | **PK**, identity | |
| job_id | BIGINT | **FK** → JOBS.id, NOT NULL, ON DELETE CASCADE | |
| note | TEXT | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

Indexes: `job_id`.

### INTERVIEWS
| Column | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | **PK**, identity | |
| job_id | BIGINT | **FK** → JOBS.id, NOT NULL, ON DELETE CASCADE | |
| interview_date | TIMESTAMPTZ | NOT NULL | Date and time of the interview |
| round_type | VARCHAR(50) | NULL | e.g., Phone Screen, Technical, HR, Final |
| notes | TEXT | NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

Indexes: `job_id`.

### STATUS_HISTORY
| Column | Type | Constraints | Notes |
|---|---|---|---|
| id | BIGINT | **PK**, identity | |
| job_id | BIGINT | **FK** → JOBS.id, NOT NULL, ON DELETE CASCADE | |
| status | VARCHAR(20) | NOT NULL, CHECK IN (`SAVED`, `APPLIED`, `INTERVIEWING`, `OFFER`, `REJECTED`) | The status the job moved to |
| changed_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

Indexes: `(job_id, changed_at)`, `status`.

Append-only: rows are never updated. A row is inserted when a job is created (initial status) and on every status change, in the same transaction as the update to `JOBS.status`.

### Data Rules
- **Dashboard counts** = number of distinct `job_id`s in `STATUS_HISTORY` that have ever reached `APPLIED` / `INTERVIEWING` / `OFFER` / `REJECTED`.
- **Upcoming deadlines** = `JOBS` where `status = 'SAVED'` and `deadline BETWEEN CURRENT_DATE AND CURRENT_DATE + 3`.
- **Search** = case-insensitive partial match (`ILIKE`) on `JOBS.job_title`, `COMPANY.company_name`, and `JOBS.job_description`.
- **Deleting a job** cascades to its notes, interviews, and status history.
- **Future multi-user:** add a `USERS` table and a `user_id BIGINT FK → USERS.id` to `COMPANY` and `JOBS`; child tables inherit ownership through `job_id`.

## Out of Scope (v1)
- Deadline reminders (email, browser notifications, calendar export) — dashboard only for now
- Resume / cover letter file uploads or links — text labels only
- Multiple users, authentication, and accounts
- Online hosting / deployment
- CSV export / backup
- Recruiter / hiring manager contacts
- Salary, location, work setup, and employment type fields
- Additional statuses (Withdrawn, Ghosted, Accepted)

## Success Criteria
- I can create a company and a job, update the job's status, and see the dashboard cards update correctly.
- Jobs with status `Saved` and a deadline within 3 days appear in the dashboard's upcoming deadlines list.
- I can find a job on the All Jobs page using search, filter, and sort, with pagination at 10 per page.
- I can view a job's full details, notes, interviews, and status history on the Job Detail page.
- The whole app (backend, frontend, database) starts locally with a single command.
- The backend has unit and integration tests covering core business logic (status changes, dashboard counts, upcoming deadlines, search).
