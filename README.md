# 🎯 Opportunity Tracker & Reminder Service

A production-ready REST API service for tracking career opportunities (internships, hackathons, jobs, scholarships) with automated deadline reminder scheduling, BullMQ background worker processing, and rich email notifications via Nodemailer.

---

## 📑 Table of Contents

- [Overview](#-overview)
- [Key Features](#-key-features)
- [Architecture & Workflow](#-architecture--workflow)
- [Tech Stack](#-tech-stack)
- [Project Directory Structure](#-project-directory-structure)
- [Prerequisites](#-prerequisites)
- [Installation & Setup](#-installation--setup)
- [Environment Variables](#-environment-variables)
- [Running the Application](#-running-the-application)
- [API Reference](#-api-reference)
  - [Authentication](#authentication-apiv1)
  - [Opportunities](#opportunities-apiv1opportunities)
  - [Notifications & Email Testing](#notifications--email-apiv1)
- [Mail & Reminder Notification System](#-mail--reminder-notification-system)
- [Testing Email Delivery](#-testing-email-delivery)
- [License](#-license)

---

## 🌟 Overview

The **Opportunity Tracker & Reminder Service** helps students and professionals stay ahead of critical deadlines. When an opportunity is logged, the service automatically calculates and queues a reminder job set to trigger **5 days before the deadline**. A dedicated background worker executes the job, verifies the opportunity's status, persists an audit trail of the notification in the PostgreSQL database, and delivers a responsive, branded email alert to the applicant.

---

## ✨ Key Features

- **🔐 Robust Authentication & Authorization**
  - JWT access and refresh token pattern with HttpOnly cookies.
  - Refresh token persistence and revocation tracking in PostgreSQL.
  - Token blacklisting in Redis upon logout.
  - Role-Based Access Control (`ADMIN`, `USER`).

- **📌 Opportunity Management**
  - Track categories: `INTERNSHIP`, `HACKATHON`, `JOB`, and `SCHOLARSHIP`.
  - Filter by domain/field (`BACKEND`, `FRONTEND`, `APPLIED_AI`, `FULL_STACK`, `DATA_SCIENCE`).
  - Upcoming dashboard with Redis caching for ultra-fast response times.

- **⏰ Automated Reminder Scheduling (BullMQ + Redis)**
  - Automated 5-day pre-deadline reminder calculation.
  - Immediate alert trigger if the deadline is already within 5 days.
  - Automatic reminder rescheduling when deadlines are updated.
  - Automatic reminder cancellation when opportunities are marked as `APPLIED`, `SELECTED`, or `REJECTED`.

- **📧 High-Deliverability Mail Service (Nodemailer)**
  - Responsive, branded HTML email templates with plain text fallbacks and action buttons.
  - Multi-event support (`DEADLINE_REMINDER`, `WELCOME`, `OPPORTUNITY_CREATED`, `TEST`).
  - Development mode support with Ethereal test accounts and preview URL generation.
  - Configurable SMTP transports (Gmail, Resend, SendGrid, Mailgun, Amazon SES).

- **📊 Notification Audit & Retry System**
  - Every notification is recorded in the `Notification` table with status progression (`PENDING` ➔ `SENT` / `FAILED`).
  - Exponential backoff retry for failed jobs.
  - Endpoint to query notification history and manually retry failed notifications.

---

## 🏗️ Architecture & Workflow

```mermaid
flowchart TD
    User([User / Client]) -->|POST /api/v1/opportunities| API[Express API Server]
    API -->|Save Opportunity| DB[(PostgreSQL Database)]
    API -->|scheduleReminder()| Queue[BullMQ 'reminders' Queue]
    Queue -->|Delayed Job| Redis[(Redis Store)]

    subgraph Background Processing
        Redis -->|Job Ready (5 Days Before Deadline)| Worker[Reminder Worker]
        Worker -->|1. Validate User & Opportunity| DB
        Worker -->|2. Create Notification (PENDING)| DB
        Worker -->|3. sendEmail()| SMTP[Nodemailer / SMTP Server]
        SMTP -->|Deliver Message| Inbox([Recipient Email Inbox])
        Worker -->|4. Update Notification (SENT)| DB
    end

    User -->|PATCH /api/v1/:id/status (APPLIED)| API
    API -->|cancelReminder()| Queue
    Queue -->|Remove Job| Redis
```

---

## 🛠️ Tech Stack

| Category | Technology |
| :--- | :--- |
| **Runtime & Language** | [Node.js](https://nodejs.org/) (ES Modules) & [TypeScript](https://www.typescriptlang.org/) |
| **Web Framework** | [Express 5](https://expressjs.com/) |
| **Database & ORM** | [PostgreSQL](https://www.postgresql.org/) with [Prisma ORM 7](https://www.prisma.io/) (`@prisma/adapter-pg`) |
| **Queue & Worker** | [BullMQ](https://docs.bullmq.io/) backed by [Redis](https://redis.io/) (`ioredis`) |
| **Email Transport** | [Nodemailer](https://nodemailer.com/) |
| **Validation** | [Zod](https://zod.dev/) |
| **Security** | [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken), [bcrypt](https://github.com/kelektiv/node.bcrypt.js), `cookie-parser`, `cors` |
| **Development** | [tsx](https://github.com/privatenumber/tsx), [nodemon](https://nodemon.io/) |

---

## 📁 Project Directory Structure

```
.
├── prisma/
│   └── schema.prisma                 # Database models (User, Opportunity, Notification, RefreshTokens)
├── src/
│   ├── app.ts                        # Express server entry point & middleware setup
│   ├── lib/
│   │   ├── prisma.ts                 # Prisma Client singleton with PostgreSQL adapter
│   │   └── redis.ts                  # Redis connection client & options for BullMQ
│   ├── middlewares/
│   │   ├── asyncHandler.ts           # Async exception wrapper for routes
│   │   ├── auth.middleware.ts        # JWT verification and blacklist checker
│   │   ├── rbac.middleware.ts        # Role-based authorization
│   │   └── validate.middleware.ts    # Zod schema validation
│   ├── modules/
│   │   ├── auth/                     # Authentication (login, register, refresh, logout)
│   │   ├── opportunity/              # Opportunity CRUD, dashboard & reminder scheduling
│   │   ├── notification/             # Notification history & email testing endpoints
│   │   ├── queues/                   # BullMQ queue definitions
│   │   ├── user/                     # User profile routes & services
│   │   └── worker/                   # Email service & BullMQ background worker
│   │       ├── email.service.ts      # Nodemailer templates & dispatch logic
│   │       └── reminder.worker.ts    # BullMQ job worker & DB audit logger
│   └── utils/
│       ├── ApiError.ts               # Custom error abstraction
│       └── jwt.ts                    # JWT token generation and verification
├── .env                              # Environment configuration
├── .env.example                      # Template environment variables
├── package.json                      # Dependencies and scripts
├── test-email.ts                     # Standalone CLI test script for email integration
└── tsconfig.json                     # TypeScript compiler configuration
```

---

## 📦 Prerequisites

Ensure you have the following installed locally:

- **Node.js** >= 18.x
- **PostgreSQL** >= 14
- **Redis** >= 6.x

---

## ⚙️ Environment Variables

Create a `.env` file in the project root with the following configuration (or copy from `.env.example`):

```env
# Application Port
PORT=5000
NODE_ENV=development

# PostgreSQL Database Connection
DATABASE_URL="postgresql://postgres:password@localhost:5432/Opportunity_Tracker"

# JWT Secrets
ACCESS_TOKEN_SECRET="your-super-secret-access-token-key"
REFRESH_TOKEN_SECRET="your-super-secret-refresh-token-key"

# Redis Connection URL
REDIS_URL="redis://localhost:6379"

# SMTP Mail Server Configuration
EMAIL_HOST="smtp.ethereal.email"
EMAIL_PORT=587
EMAIL_SECURE=false
EMAIL_USER="your-user@ethereal.email"
EMAIL_PASSWORD="your-ethereal-password"
EMAIL_FROM="Opportunity Tracker <your-user@ethereal.email>"
EMAIL_FROM_NAME="Opportunity Tracker"
```

> **Tip for Testing with Ethereal:**  
> If you don't have an SMTP account, you can create a free disposable inbox in 5 seconds at [https://ethereal.email/create](https://ethereal.email/create).  
> If `EMAIL_USER` or `EMAIL_PASSWORD` are left blank in development, `email.service.ts` will automatically generate a temporary Ethereal account on boot!

---

## 🚀 Installation & Setup

1. **Clone the repository and install dependencies:**
   ```bash
   npm install
   ```

2. **Set up the Database Schema with Prisma:**
   ```bash
   # Generate Prisma client
   npx prisma generate

   # Push schema to PostgreSQL database
   npx prisma db push
   ```

3. **Verify Redis & PostgreSQL are running:**
   ```bash
   # Test Redis connectivity
   redis-cli ping
   # Expected output: PONG
   ```

---

## 🏃 Running the Application

### 1. Start the API Server (Development Mode)
```bash
npm run dev
```
> The API server boots up on `http://localhost:5000` and automatically starts the reminder worker in-process.

### 2. Run the Worker as an Independent Process (Optional / Production)
In production, you can scale the background worker independently from the HTTP API server:
```bash
npm run worker
```

### 3. Run the Email Integration Test CLI
```bash
npm run test:email
```
> Verifies the SMTP connection and sends test & deadline reminder emails to your configured inbox.

---

## 📡 API Reference

### Authentication (`/api/v1`)

| Method | Endpoint | Description | Auth Required |
| :--- | :--- | :--- | :---: |
| `POST` | `/register` | Register a new user (sends welcome email) | No |
| `POST` | `/login` | Authenticate user & issue tokens | No |
| `POST` | `/auth/refresh` | Issue new access token using refresh token | No |
| `POST` | `/logout` | Invalidate refresh token & blacklist access token | Yes |

### Opportunities (`/api/v1/opportunities` & `/api/v1`)

| Method | Endpoint | Description | Auth Required |
| :--- | :--- | :--- | :---: |
| `POST` | `/opportunities` | Create opportunity & schedule reminder | Yes |
| `PATCH` | `/opportunities/:id` | Update opportunity & reschedule reminder | Yes |
| `PATCH` | `/opportunities/:id/status` | Update status (cancels reminder if applied) | Yes |
| `GET` | `/dashboard` | View upcoming opportunities & counts (cached) | Yes |

#### Opportunity Creation Payload Example (`POST /api/v1/opportunities`)
```json
{
  "title": "Full Stack Engineering Internship",
  "type": "INTERNSHIP",
  "field": "FULL_STACK",
  "source": "USER",
  "verificationStatus": "APPROVED",
  "reviewedAt": "2026-11-11T00:00:00.000Z",
  "sourceLink": "https://careers.google.com",
  "eligiblity_Notes": "Graduating 2027, proficiency in Node.js and React",
  "deadline": "2026-11-25T18:00:00.000Z"
}
```

### Notifications & Email (`/api/v1`)

| Method | Endpoint | Description | Auth Required |
| :--- | :--- | :--- | :---: |
| `GET` | `/email/verify` | Health check for SMTP server connection | No |
| `POST` | `/email/test` | Dispatch an immediate test email | Yes |
| `GET` | `/notifications` | Get paginated list of user's notifications | Yes |
| `POST` | `/notifications/:id/retry` | Manually retry sending a failed notification | Yes |

---

## 📨 Mail & Reminder Notification System

### 1. 5-Day Reminder Calculation
When an opportunity is created or updated, `scheduleReminder()` in `src/modules/opportunity/reminder.services.ts` calculates:
- $\text{reminderTime} = \text{deadline} - 5\text{ days}$
- $\text{delay} = \max(0, \text{reminderTime} - \text{now})$

- **If $\text{deadline} \le \text{now}$**: Deadline has already elapsed; no reminder is scheduled.
- **If $\text{now} < \text{deadline} < \text{reminderTime}$**: The deadline is within 5 days from now; `delay` is set to `0` and an immediate reminder is queued.
- **If $\text{reminderTime} > \text{now}$**: BullMQ delays the job execution until exactly 5 days before the deadline.

### 2. Status Cancellation Guard
If a user applies for an opportunity or changes its status to `APPLIED`, `SELECTED`, or `REJECTED`, `cancelReminder(jobId)` is automatically called to prevent sending unnecessary reminders.

### 3. Worker Safety Checks
When the reminder worker picks up a job:
1. It verifies the user exists and has a valid email address.
2. It verifies the opportunity still exists in the database.
3. It checks `opportunity.status === 'NOT_APPLIED'`. If the user already applied, the job is safely skipped without sending an email.
4. It logs the notification as `PENDING`, sends the email, and marks it `SENT`. If sending fails, it logs `lastError` and sets the status to `QUEUED` or `FAILED` for retry.

---

## 🧪 Testing Email Delivery

You can quickly verify that your mail transport is configured properly using the included CLI utility:

```bash
npm run test:email
```

**Sample Output:**
```
--- Testing Mail Sending Integration ---
Host: smtp.ethereal.email
Port: 587
User: pinkie.legros@ethereal.email
From: Opportunity Tracker <pinkie.legros@ethereal.email>

1. Verifying SMTP connection...
SMTP Status: { success: true, message: 'SMTP server is ready to deliver messages' }

2. Sending Test Email to: pinkie.legros@ethereal.email
📧 [Email Preview URL]: https://ethereal.email/message/WaQKMgKddxQDoou...
Test Email Result: { success: true, messageId: '<d01...>' }

3. Sending Deadline Reminder Email to: pinkie.legros@ethereal.email
📧 [Email Preview URL]: https://ethereal.email/message/WaQKMgKddxQDoov...
Reminder Email Result: { success: true, messageId: '<d02...>' }

--- All Email Tests Completed Successfully! ---
```

Clicking the generated **Preview URL** allows you to view the rendered HTML email directly in your browser.

---

## 📄 License

This project is licensed under the Apache 2.0 License. See the [LICENSE](LICENSE) file for details.

