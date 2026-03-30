# Reminder Service

A background worker service that handles all email notifications in the Airline Booking system. It consumes messages from a RabbitMQ queue — either sending emails immediately or storing them as scheduled tickets — and runs a cron job every 2 minutes to dispatch pending reminder emails before flights depart.

---

## Architecture Overview

```
BookingService
  |
  v
RabbitMQ (AIRLINE_EXCHANGE → REMINDER_QUEUE)
  |
  v
ReminderService (Port 8000)
  |
  |-- SEND_BASIC_MAIL ──────────────────> Gmail SMTP (immediate send)
  |-- CREATE_TICKET ────────────────────> MySQL (store as PENDING)
  |
  v
Cron Job (every 2 minutes)
  |
  Fetch PENDING tickets where notificationTime <= now
  |
  v
Gmail SMTP ──> User's inbox
```

This service has no public HTTP API. It runs purely as a background event consumer and scheduler.

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Node.js + Express 5.2.1 | Service skeleton (minimal HTTP layer) |
| amqplib 0.10.9 | RabbitMQ consumer (message queue client) |
| nodemailer 7.0.12 | Email sending via Gmail SMTP |
| node-cron 4.2.1 | Cron job scheduler |
| Sequelize 6.37.7 | ORM for MySQL |
| mysql2 3.16.0 | MySQL driver |
| body-parser 2.2.2 | Request body parsing |
| dotenv 17.2.3 | Environment variable loading |
| nodemon 3.1.11 | Auto-restart in development |

---

## Database Design

**Database:** `REMINDER_DB_DEV`

### NotificationTickets Table

| Column | Type | Constraints |
|---|---|---|
| id | INTEGER | Primary Key, Auto Increment |
| subject | STRING | Required |
| content | STRING | Required |
| recepientEmail | STRING | Required |
| status | ENUM | `'PENDING'`, `'SUCCESS'`, `'FAILED'` — Default: `'PENDING'` |
| notificationTime | DATE | Required — when the email should be sent |
| createdAt | DATE | Auto-managed |
| updatedAt | DATE | Auto-managed |

---

## RabbitMQ Integration

### Consumer Setup

On service startup:
1. Connects to RabbitMQ at `MESSAGE_BROKER_URL`.
2. Creates a channel.
3. Asserts a `direct` Exchange with name `EXCHANGE_NAME`.
4. Asserts a Queue named `REMINDER_QUEUE`.
5. Binds the queue to the exchange using `REMINDER_BINDING_KEY` (exact match routing).
6. Attaches a message listener (`channel.consume`).
7. Acknowledges each message with `channel.ack(msg)` after processing (removes it from queue).

### Message Processing

Each incoming message is parsed from JSON and routed by the `service` field:

#### `SEND_BASIC_MAIL` — Immediate Email

```json
{
  "service": "SEND_BASIC_MAIL",
  "data": {
    "mailFrom": "airlineHelpline@gamil.com",
    "mailTo": "user@example.com",
    "mailSubject": "Booking confirmation Mail",
    "mailBody": "Dear user this is to inform you that your booking has been confirmed..."
  }
}
```

**What happens:**
1. ReminderService receives the message.
2. Calls `EmailService.sendBasicEmail(data)`.
3. Nodemailer sends the email immediately via Gmail SMTP.
4. Message is acknowledged and deleted from the queue.

**Triggered by:** BookingService immediately after a booking is confirmed.

---

#### `CREATE_TICKET` — Scheduled Reminder

```json
{
  "service": "CREATE_TICKET",
  "data": {
    "subject": "Ticket Reminder Mail",
    "content": "Dear user this is to remind you of your upcoming flight...",
    "recepientEmail": "user@example.com",
    "notificationTime": "2026-04-01T04:00:00.000Z"
  }
}
```

**What happens:**
1. ReminderService receives the message.
2. Calls `TicketService.createTicket(data)`.
3. Creates a `NotificationTicket` record in MySQL with `status = 'PENDING'`.
4. Message is acknowledged and deleted from the queue.
5. The cron job will pick this up and send the email at `notificationTime`.

**Triggered by:** BookingService immediately after a booking. `notificationTime` is set to `departureTime - 2 hours`.

---

## Cron Job

**Schedule:** `*/2 * * * *` — runs every 2 minutes.

**What it does:**
1. Queries the DB for all NotificationTickets where:
   - `status = 'PENDING'`
   - `notificationTime <= current time`
2. For each matching ticket:
   - Sends email via Gmail SMTP (nodemailer) with the ticket's `subject`, `content`, and `recepientEmail`.
   - Updates `status` from `'PENDING'` to `'SUCCESS'`.
3. Logs result to console.

**Why 2 minutes?** The cron runs frequently enough to ensure reminders are sent close to the scheduled time (max 2-minute delay), without hammering the DB.

**Failure handling:** If a send fails, the ticket currently stays in PENDING state and will be retried on the next cron run. A `FAILED` status is set for permanent failures.

---

## Email Configuration

Uses **nodemailer** with Gmail SMTP transport.

```
Transport: Gmail SMTP
Auth: { user: EMAIL_ID, pass: EMAIL_PASS }
```

> **Important:** `EMAIL_PASS` must be a [Gmail App Password](https://support.google.com/accounts/answer/185833), not your regular Google account password. Enable 2FA on the Gmail account, then generate an App Password specifically for this service.

### Nodemailer Mail Options

```js
{
  from: mailFrom,       // sender address
  to: mailTo,           // recipient address
  subject: mailSubject, // email subject
  text: mailBody        // plain text body
}
```

---

## Message Flow: Full Booking Notification Sequence

```
User completes booking
  |
  v
BookingService
  |-- Publish SEND_BASIC_MAIL to RabbitMQ
  |-- Publish CREATE_TICKET to RabbitMQ
  |
  v
RabbitMQ (REMINDER_QUEUE)
  |
  v
ReminderService consumer picks up both messages

Message 1: SEND_BASIC_MAIL
  |
  v
nodemailer.sendMail() ──> User receives "Booking Confirmed" email immediately

Message 2: CREATE_TICKET
  |
  v
DB INSERT NotificationTicket { status: 'PENDING', notificationTime: departureTime - 2h }

... time passes ...

Cron job fires (every 2 min)
  |
  v
SELECT * FROM NotificationTickets WHERE status='PENDING' AND notificationTime <= NOW()
  |
  v
nodemailer.sendMail() ──> User receives "Flight Reminder" email
  |
  v
UPDATE NotificationTickets SET status='SUCCESS' WHERE id=...
```

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `PORT` | Yes | Port the Express server listens on (e.g., 8000) |
| `EMAIL_ID` | Yes | Gmail address used to send emails |
| `EMAIL_PASS` | Yes | Gmail App Password (not regular password) |
| `EXCHANGE_NAME` | Yes | RabbitMQ exchange name — must match BookingService |
| `REMINDER_BINDING_KEY` | Yes | RabbitMQ routing key — must match BookingService |
| `MESSAGE_BROKER_URL` | Yes | RabbitMQ connection URL (e.g., `amqp://localhost`) |

**.env example:**
```
PORT=8000
EMAIL_ID=yourgmailaccount@gmail.com
EMAIL_PASS=your_app_password_here
EXCHANGE_NAME=AIRLINE_EXCHANGE
REMINDER_BINDING_KEY=REMINDER_QUEUE_KEY
MESSAGE_BROKER_URL=amqp://localhost
```

> **Security:** Never commit `.env` to version control. Add it to `.gitignore`.

---

## Database Setup

```bash
# Install dependencies
npm install

# Create the database
cd src
npx sequelize db:create

# Run migrations
npx sequelize db:migrate
```

**Database config** in `src/config/config.json`:
```json
{
  "development": {
    "username": "YOUR_DB_USER",
    "password": "YOUR_DB_PASSWORD",
    "database": "REMINDER_DB_DEV",
    "host": "127.0.0.1",
    "dialect": "mysql"
  }
}
```

---

## Running the Service

```bash
# Development
npx nodemon src/index.js

# Production
node src/index.js
```

> **Prerequisites:**
> - RabbitMQ must be running and accessible at `MESSAGE_BROKER_URL`
> - Gmail App Password must be configured in `.env`
> - MySQL must be running with `REMINDER_DB_DEV` created and migrated

---

## Project Structure

```
ReminderService/
├── package.json
├── src/
│   ├── index.js                        # Entry point: starts Express + RabbitMQ consumer + cron
│   ├── config/
│   │   └── config.json                 # Sequelize DB config
│   ├── migrations/                     # Sequelize migration files
│   ├── models/
│   │   ├── index.js                    # Model loader
│   │   └── notificationticket.js       # NotificationTicket model
│   ├── repositories/
│   │   └── ticket-repository.js        # DB queries for NotificationTicket
│   ├── services/
│   │   ├── email-service.js            # Nodemailer transport setup + send helpers
│   │   └── ticket-service.js           # Business logic for creating/fetching tickets
│   └── utils/
│       ├── message-queue.js            # RabbitMQ channel + subscribe/publish helpers
│       └── cron-jobs.js                # Cron schedule definition + execution logic
```

---

## RabbitMQ Concepts Used

### Exchange (Direct Type)
Messages are published to the **Exchange**, not directly to a queue. The `direct` exchange type means it only routes a message to a queue whose **binding key exactly matches** the message's routing key.

### Queue
The `REMINDER_QUEUE` is a buffer where messages wait until the consumer picks them up. Messages are persistent even if the service restarts.

### Binding
The queue is bound to the exchange with `REMINDER_BINDING_KEY`. Any message published with that exact routing key gets routed into this queue.

### Acknowledgement
After processing each message, the consumer calls `channel.ack(msg)`. Only then does RabbitMQ permanently delete the message from the queue. If the service crashes before acking, the message is re-queued and retried.

### Channel
A virtual connection inside the main TCP connection to RabbitMQ. All operations (subscribe, publish, ack) happen on the channel.

---

## Notification Ticket Status Lifecycle

```
Message CREATE_TICKET received
  |
  v
PENDING   ← Stored in DB, waiting for notificationTime
  |
  v
SUCCESS   ← Email sent successfully by cron job
  |
FAILED    ← Email send failed (handled for permanent failures)
```

---

## Inter-Service Communication

| Direction | Protocol | Description |
|---|---|---|
| BookingService → ReminderService | RabbitMQ (async) | Publishes SEND_BASIC_MAIL and CREATE_TICKET messages |
| ReminderService → Gmail SMTP | SMTP | Sends confirmation and reminder emails |

ReminderService makes no HTTP calls to other microservices.
