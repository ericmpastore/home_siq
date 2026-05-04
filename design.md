# HomeSIQ — Application Design Document

## Overview

HomeSIQ is a web-first, mobile-ready platform for real estate investors and their tenants to manage rental property operations. The initial release focuses on the maintenance module, giving tenants a way to report repairs and track their status, and giving investors a queue to triage, assign, and update those requests in real time.

---

## Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| Web Frontend | Next.js 14 (App Router, TypeScript) | SEO, server components, fast routing |
| Mobile | Expo + React Native (iOS first) | Shared logic with web, fast native build |
| Styling | Tailwind CSS + shadcn/ui | Rapid, consistent design system |
| Backend / DB | Supabase (Postgres) | Auth, database, storage, and real-time in one |
| File Storage | Supabase Storage | Tenant repair photos |
| Real-time | Supabase Realtime | Live status updates without polling |
| Push Notifications | Expo Push Notifications | Alert tenants and investors on status changes |

---

## Monorepo Structure

```
App_HomeSIQ_Delivery/
├── web/                              ← Next.js web app
│   ├── src/
│   │   ├── app/
│   │   │   ├── (auth)/
│   │   │   │   ├── login/            ← Shared login page
│   │   │   │   └── register/         ← Register + role selection (tenant / investor)
│   │   │   ├── (tenant)/
│   │   │   │   ├── dashboard/        ← Tenant home: active repairs, announcements
│   │   │   │   ├── maintenance/
│   │   │   │   │   ├── new/          ← Submit repair request form
│   │   │   │   │   └── [id]/         ← Ticket detail: status timeline + ETA
│   │   │   ├── (investor)/
│   │   │   │   ├── dashboard/        ← Investor home: KPIs, pending requests
│   │   │   │   ├── properties/       ← Property and unit management
│   │   │   │   ├── maintenance/
│   │   │   │   │   ├── queue/        ← All open tickets across all properties
│   │   │   │   │   └── [id]/         ← Ticket detail: update status, ETA, notes
│   │   │   ├── api/                  ← Next.js API routes (server-side Supabase calls)
│   │   │   └── layout.tsx            ← Root layout, theme provider
│   │   ├── components/
│   │   │   ├── ui/                   ← shadcn/ui base components (button, card, badge…)
│   │   │   ├── maintenance/
│   │   │   │   ├── RequestCard.tsx         ← Ticket summary card
│   │   │   │   ├── StatusTimeline.tsx      ← Visual status progression
│   │   │   │   ├── StatusBadge.tsx         ← Color-coded status pill
│   │   │   │   ├── PriorityBadge.tsx       ← Emergency / Urgent / Standard
│   │   │   │   ├── NewRequestForm.tsx      ← Multi-step form with photo upload
│   │   │   │   └── UpdateStatusForm.tsx    ← Investor: set status, ETA, notes
│   │   │   ├── layout/
│   │   │   │   ├── TenantNav.tsx
│   │   │   │   └── InvestorNav.tsx
│   │   ├── lib/
│   │   │   ├── supabase/
│   │   │   │   ├── client.ts         ← Browser Supabase client
│   │   │   │   ├── server.ts         ← Server Supabase client (RSC / API routes)
│   │   │   │   └── schema.sql        ← Full DB schema + RLS policies
│   │   │   └── types.ts              ← Shared TypeScript interfaces
│   └── package.json
├── mobile/                           ← Expo React Native app (iOS first)
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login.tsx
│   │   │   └── register.tsx
│   │   ├── (tenant)/
│   │   │   ├── index.tsx             ← Tenant dashboard
│   │   │   ├── maintenance/
│   │   │   │   ├── new.tsx           ← Submit request (camera + photo picker)
│   │   │   │   └── [id].tsx          ← Ticket detail + real-time status
│   │   ├── (investor)/
│   │   │   ├── index.tsx             ← Investor dashboard
│   │   │   └── maintenance/
│   │   │       ├── queue.tsx
│   │   │       └── [id].tsx
│   ├── components/                   ← Native versions of RequestCard, StatusTimeline…
│   ├── lib/
│   │   ├── supabase.ts               ← Same Supabase project, mobile client config
│   │   └── types.ts
│   └── package.json
└── supabase/
    ├── migrations/
    │   └── 001_initial_schema.sql
    └── seed.sql                      ← Demo data for development
```

---

## Database Schema

### Table: `profiles`
Extends Supabase `auth.users`. One row per registered user.

| Column | Type | Notes |
|---|---|---|
| id | uuid | FK → auth.users |
| role | enum | `tenant` or `investor` |
| full_name | text | |
| phone | text | |
| avatar_url | text | Supabase Storage path |
| created_at | timestamptz | |

### Table: `properties`
Owned by an investor. Contains one or more units.

| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| investor_id | uuid | FK → profiles |
| name | text | e.g. "Maple Street Fourplex" |
| address | text | |
| city | text | |
| state | text | |
| zip | text | |
| created_at | timestamptz | |

### Table: `units`
A rentable unit within a property. Linked to one tenant at a time.

| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| property_id | uuid | FK → properties |
| unit_number | text | e.g. "2B" |
| tenant_id | uuid | FK → profiles, nullable |
| created_at | timestamptz | |

### Table: `maintenance_requests`
Core table. One row per repair request submitted by a tenant.

| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| unit_id | uuid | FK → units |
| tenant_id | uuid | FK → profiles |
| title | text | Short summary |
| description | text | Full detail from tenant |
| category | enum | See categories below |
| priority | enum | See priorities below |
| status | enum | See statuses below |
| created_at | timestamptz | |
| updated_at | timestamptz | |

### Table: `status_updates`
Append-only log of every status change. Drives the timeline UI.

| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| request_id | uuid | FK → maintenance_requests |
| new_status | enum | The status being set |
| message | text | Investor-facing note to tenant |
| eta | date | Estimated completion date, nullable |
| updated_by | uuid | FK → profiles (investor) |
| created_at | timestamptz | |

### Table: `maintenance_photos`
Photos attached to a repair request at submission or during updates.

| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| request_id | uuid | FK → maintenance_requests |
| storage_path | text | Supabase Storage bucket path |
| uploaded_by | uuid | FK → profiles |
| created_at | timestamptz | |

---

## Enumerations

### Status (ordered progression)
| Value | Meaning |
|---|---|
| `submitted` | Tenant submitted, not yet seen by investor |
| `acknowledged` | Investor has viewed and confirmed receipt |
| `scheduled` | Repair date set, ETA provided |
| `in_progress` | Work has begun |
| `completed` | Repair finished |
| `on_hold` | Paused (waiting on part, tenant access, etc.) |
| `cancelled` | Request withdrawn or resolved without repair |

### Priority
| Value | Target Response Time |
|---|---|
| `emergency` | Same day |
| `urgent` | 1–3 days |
| `standard` | Within 7 days |
| `planned` | Scheduled at convenience |

### Category
`plumbing` · `electrical` · `hvac` · `appliances` · `structural` · `pest_control` · `general`

---

## Row-Level Security (RLS) Policies

| Table | Tenant Access | Investor Access |
|---|---|---|
| `maintenance_requests` | SELECT / INSERT own unit's rows | SELECT all rows for owned properties; no INSERT |
| `status_updates` | SELECT where request belongs to own unit | INSERT / SELECT for owned properties |
| `maintenance_photos` | SELECT / INSERT own request photos | SELECT for owned properties |
| `units` | SELECT own row | SELECT / INSERT / UPDATE for owned properties |
| `properties` | No access | Full access to own rows |
| `profiles` | SELECT / UPDATE own row | SELECT / UPDATE own row |

---

## Key User Flows

### Tenant

1. **Register** — choose "I am a Tenant", enter unit code provided by investor
2. **Dashboard** — card list of open tickets sorted by priority, each showing status badge and days since submission
3. **New Request** — multi-step form:
   - Step 1: Select category
   - Step 2: Write title and description, attach photos (camera or library)
   - Step 3: Self-assess priority (with guidance text)
   - Step 4: Review and submit
4. **Ticket Detail** — vertical timeline showing each status update with investor note, ETA chip, and timestamp. Subscribes to Supabase Realtime so status changes appear live without a page refresh.

### Investor

1. **Register** — choose "I am an Investor / Property Manager"
2. **Create Properties** — add property address, then add units with unit numbers
3. **Invite Tenants** — enter tenant email + unit; system emails invite link
4. **Maintenance Queue** — sortable, filterable table of all open tickets across all properties. Columns: property, unit, tenant, category, priority, status, days open
5. **Ticket Detail** — full request info + photo gallery + status update form:
   - Dropdown: new status
   - Date picker: ETA
   - Text field: note to tenant
   - Submit → creates `status_update` row, updates `maintenance_requests.status`, fires real-time event + push notification

---

## Real-time and Notifications

| Trigger | Mechanism | Recipient |
|---|---|---|
| Investor posts status update | Supabase Realtime (Postgres changes) | Tenant ticket detail page updates live |
| Investor posts status update | Expo Push Notification (via Supabase Edge Function) | Tenant mobile app |
| Tenant submits new request | Supabase Realtime + push notification | Investor mobile app |
| Request overdue (past ETA) | Scheduled Supabase Edge Function (cron) | Investor email digest |

---

## Mobile-specific Considerations (iOS First)

- **Camera / Photo Library** — Expo ImagePicker with permission request on first use
- **Push Notifications** — Expo Notifications; permission requested on first login; token stored in `profiles`
- **Offline Drafts** — repair request form state persisted to AsyncStorage; submitted automatically when connection is restored
- **Haptic Feedback** — Expo Haptics on status change confirmations and form submission
- **Biometric Auth** — Expo LocalAuthentication for quick re-entry after app backgrounding

---

## Design System

- **Primary color:** Deep navy `#1B2A4A` — trust, professionalism
- **Accent color:** Warm amber `#F59E0B` — action, urgency
- **Success:** `#10B981` (completed status)
- **Warning:** `#F59E0B` (scheduled / on hold)
- **Danger:** `#EF4444` (emergency priority, overdue)
- **Neutral:** Slate gray scale
- **Typography:** Inter (web) / SF Pro (iOS native)
- **Corner radius:** 12px cards, 8px inputs — modern but not playful

### Status Badge Colors
| Status | Background | Text |
|---|---|---|
| Submitted | Slate 100 | Slate 700 |
| Acknowledged | Blue 100 | Blue 700 |
| Scheduled | Amber 100 | Amber 700 |
| In Progress | Violet 100 | Violet 700 |
| Completed | Green 100 | Green 700 |
| On Hold | Orange 100 | Orange 700 |
| Cancelled | Red 100 | Red 700 |

---

## Future Modules (Post-MVP)

| Module | Description |
|---|---|
| Rent Payment | ACH / card payments, auto-receipts, late fee tracking |
| Lease Management | Upload, e-sign, and store lease documents |
| Vendor Network | Assign repair requests to trusted contractors |
| Announcements | Investor broadcasts to all tenants in a property |
| Inspection Reports | Photo-based move-in / move-out checklists |
| Financial Dashboard | Income, expenses, and NOI per property |
