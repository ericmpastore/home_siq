# HomeSIQ — Application Design Document (MERN Stack)

## Overview

HomeSIQ is a web-first, mobile-ready platform for real estate investors and their tenants to manage rental property operations. The initial release focuses on the maintenance module, giving tenants a way to report repairs and track their status, and giving investors a queue to triage, assign, and update those requests in real time.

---

## Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| Web Frontend | React 18 + Vite (TypeScript) | Fast dev builds, SPA routing, broad ecosystem |
| Mobile | Expo + React Native (iOS first) | Shared component logic with web, fast native build |
| Styling | Tailwind CSS + shadcn/ui | Rapid, consistent design system |
| Backend | Node.js + Express.js | Lightweight REST API, large middleware ecosystem |
| Database | MongoDB Atlas | Flexible document model, Atlas cloud hosting |
| Auth | JWT (access + refresh tokens, bcrypt) | Stateless, works across web and mobile clients |
| File Storage | Cloudinary | Image upload, transformation, and CDN delivery |
| Real-time | Socket.io | Bidirectional events over WebSocket with fallback |
| Push Notifications | Firebase Cloud Messaging (FCM) | Cross-platform push for iOS and Android |

---

## Monorepo Structure

```
App_HomeSIQ_Delivery/
├── server/                               ← Express.js API
│   ├── src/
│   │   ├── config/
│   │   │   ├── db.ts                     ← MongoDB Atlas connection
│   │   │   └── cloudinary.ts             ← Cloudinary SDK config
│   │   ├── middleware/
│   │   │   ├── auth.ts                   ← JWT verify middleware
│   │   │   ├── requireRole.ts            ← Role-guard (tenant / investor)
│   │   │   └── upload.ts                 ← Multer + Cloudinary upload handler
│   │   ├── models/
│   │   │   ├── User.ts                   ← Mongoose User schema
│   │   │   ├── Property.ts
│   │   │   ├── Unit.ts
│   │   │   ├── MaintenanceRequest.ts
│   │   │   ├── StatusUpdate.ts
│   │   │   └── MaintenancePhoto.ts
│   │   ├── routes/
│   │   │   ├── auth.ts                   ← /api/auth — register, login, refresh
│   │   │   ├── users.ts                  ← /api/users — profile read/update
│   │   │   ├── properties.ts             ← /api/properties — CRUD for investor
│   │   │   ├── units.ts                  ← /api/units — CRUD + tenant invite
│   │   │   └── maintenance.ts            ← /api/maintenance — requests, status updates, photos
│   │   ├── services/
│   │   │   ├── fcm.ts                    ← FCM push notification sender
│   │   │   └── socket.ts                 ← Socket.io event emitters
│   │   ├── jobs/
│   │   │   └── overdueCheck.ts           ← node-cron job: daily overdue digest email
│   │   └── index.ts                      ← Express app + Socket.io server bootstrap
│   ├── .env
│   └── package.json
├── web/                                  ← React + Vite SPA
│   ├── src/
│   │   ├── pages/
│   │   │   ├── auth/
│   │   │   │   ├── Login.tsx
│   │   │   │   └── Register.tsx          ← Role selection (tenant / investor)
│   │   │   ├── tenant/
│   │   │   │   ├── Dashboard.tsx         ← Active repairs, announcements
│   │   │   │   ├── maintenance/
│   │   │   │   │   ├── New.tsx           ← Submit repair request form
│   │   │   │   │   └── Detail.tsx        ← Ticket detail: status timeline + ETA
│   │   │   ├── investor/
│   │   │   │   ├── Dashboard.tsx         ← KPIs, pending request count
│   │   │   │   ├── Properties.tsx        ← Property and unit management
│   │   │   │   └── maintenance/
│   │   │   │       ├── Queue.tsx         ← All open tickets across all properties
│   │   │   │       └── Detail.tsx        ← Ticket detail: update status, ETA, notes
│   │   ├── components/
│   │   │   ├── ui/                       ← shadcn/ui base components (button, card, badge…)
│   │   │   ├── maintenance/
│   │   │   │   ├── RequestCard.tsx
│   │   │   │   ├── StatusTimeline.tsx
│   │   │   │   ├── StatusBadge.tsx
│   │   │   │   ├── PriorityBadge.tsx
│   │   │   │   ├── NewRequestForm.tsx    ← Multi-step form with photo upload
│   │   │   │   └── UpdateStatusForm.tsx  ← Investor: set status, ETA, notes
│   │   │   └── layout/
│   │   │       ├── TenantNav.tsx
│   │   │       └── InvestorNav.tsx
│   │   ├── lib/
│   │   │   ├── api.ts                    ← Axios instance with JWT interceptor
│   │   │   ├── socket.ts                 ← Socket.io client singleton
│   │   │   └── types.ts                  ← Shared TypeScript interfaces
│   │   ├── store/
│   │   │   └── auth.ts                   ← Zustand store: user + token state
│   │   └── router.tsx                    ← React Router v6 protected routes
│   └── package.json
├── mobile/                               ← Expo React Native app (iOS first)
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login.tsx
│   │   │   └── register.tsx
│   │   ├── (tenant)/
│   │   │   ├── index.tsx                 ← Tenant dashboard
│   │   │   └── maintenance/
│   │   │       ├── new.tsx               ← Submit request (camera + photo picker)
│   │   │       └── [id].tsx              ← Ticket detail + real-time status
│   │   ├── (investor)/
│   │   │   ├── index.tsx                 ← Investor dashboard
│   │   │   └── maintenance/
│   │   │       ├── queue.tsx
│   │   │       └── [id].tsx
│   ├── components/                       ← Native versions of RequestCard, StatusTimeline…
│   ├── lib/
│   │   ├── api.ts                        ← Same Axios instance pointing to Express API
│   │   ├── socket.ts                     ← Socket.io-client (React Native compatible)
│   │   └── types.ts
│   └── package.json
└── shared/
    └── types.ts                          ← Canonical TypeScript interfaces (re-exported by web + mobile)
```

---

## MongoDB Schemas

### Collection: `users`
One document per registered user. Replaces Supabase `auth.users` + `profiles`.

| Field | Type | Notes |
|---|---|---|
| _id | ObjectId | PK |
| email | String | Unique, indexed |
| passwordHash | String | bcrypt hash, never returned in API responses |
| role | String | `"tenant"` or `"investor"` |
| fullName | String | |
| phone | String | |
| avatarUrl | String | Cloudinary URL |
| fcmToken | String | Firebase device token for push, nullable |
| refreshToken | String | Hashed refresh token, nullable |
| createdAt | Date | |

### Collection: `properties`
Owned by an investor. Contains one or more units.

| Field | Type | Notes |
|---|---|---|
| _id | ObjectId | PK |
| investorId | ObjectId | Ref → users |
| name | String | e.g. "Maple Street Fourplex" |
| address | String | |
| city | String | |
| state | String | |
| zip | String | |
| createdAt | Date | |

### Collection: `units`
A rentable unit within a property. Linked to one tenant at a time.

| Field | Type | Notes |
|---|---|---|
| _id | ObjectId | PK |
| propertyId | ObjectId | Ref → properties |
| unitNumber | String | e.g. "2B" |
| tenantId | ObjectId | Ref → users, nullable |
| inviteCode | String | Unique code tenant uses at registration |
| createdAt | Date | |

### Collection: `maintenanceRequests`
Core collection. One document per repair request submitted by a tenant.

| Field | Type | Notes |
|---|---|---|
| _id | ObjectId | PK |
| unitId | ObjectId | Ref → units |
| propertyId | ObjectId | Denormalized for efficient investor queries |
| tenantId | ObjectId | Ref → users |
| title | String | Short summary |
| description | String | Full detail from tenant |
| category | String | See categories below |
| priority | String | See priorities below |
| status | String | See statuses below |
| photoUrls | String[] | Cloudinary URLs attached at submission |
| createdAt | Date | |
| updatedAt | Date | |

### Collection: `statusUpdates`
Append-only log of every status change. Drives the timeline UI.

| Field | Type | Notes |
|---|---|---|
| _id | ObjectId | PK |
| requestId | ObjectId | Ref → maintenanceRequests |
| newStatus | String | The status being set |
| message | String | Investor-facing note to tenant |
| eta | Date | Estimated completion date, nullable |
| updatedBy | ObjectId | Ref → users (investor) |
| createdAt | Date | |

### Collection: `maintenancePhotos`
Photos added to a request after the initial submission (e.g., progress photos from investor).

| Field | Type | Notes |
|---|---|---|
| _id | ObjectId | PK |
| requestId | ObjectId | Ref → maintenanceRequests |
| cloudinaryUrl | String | Full Cloudinary delivery URL |
| cloudinaryPublicId | String | For deletion |
| uploadedBy | ObjectId | Ref → users |
| createdAt | Date | |

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

## REST API Endpoints

### Auth — `/api/auth`
| Method | Path | Description |
|---|---|---|
| POST | `/register` | Create account, return access + refresh tokens |
| POST | `/login` | Validate credentials, return tokens |
| POST | `/refresh` | Exchange refresh token for new access token |
| POST | `/logout` | Invalidate refresh token |

### Users — `/api/users`
| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/me` | Any | Get own profile |
| PATCH | `/me` | Any | Update name, phone, avatar, FCM token |

### Properties — `/api/properties`
| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/` | Investor | List own properties |
| POST | `/` | Investor | Create property |
| PATCH | `/:id` | Investor | Update property |
| DELETE | `/:id` | Investor | Delete property |
| GET | `/:id/units` | Investor | List units for a property |
| POST | `/:id/units` | Investor | Add a unit |
| PATCH | `/:id/units/:unitId` | Investor | Update unit, assign tenant |

### Maintenance — `/api/maintenance`
| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/` | Investor | All requests across owned properties (filterable) |
| GET | `/my` | Tenant | Own requests |
| POST | `/` | Tenant | Submit new request (multipart: fields + photos) |
| GET | `/:id` | Both | Get single request with status history |
| POST | `/:id/status` | Investor | Post a status update |
| POST | `/:id/photos` | Both | Upload additional photos |

---

## Authentication Flow

1. **Register** — client POSTs credentials + role to `/api/auth/register`
2. **Server** — hashes password with bcrypt, creates `users` document, signs a short-lived JWT access token (15 min) and a long-lived refresh token (7 days, stored hashed in `users.refreshToken`)
3. **Client** — stores access token in memory (React state / Zustand), refresh token in `httpOnly` cookie
4. **Requests** — Axios interceptor attaches `Authorization: Bearer <token>` header
5. **Refresh** — on 401, interceptor calls `/api/auth/refresh` with the cookie before retrying the original request
6. **Role guard** — `requireRole` middleware reads `req.user.role` after JWT verification

---

## Real-time and Notifications

| Trigger | Mechanism | Recipient |
|---|---|---|
| Investor posts status update | Socket.io event `status:updated` to room `request:<id>` | Tenant ticket detail page updates live |
| Investor posts status update | FCM push via `server/services/fcm.ts` | Tenant mobile app |
| Tenant submits new request | Socket.io event `request:new` to room `investor:<id>` | Investor web dashboard badge |
| Tenant submits new request | FCM push | Investor mobile app |
| Request overdue (past ETA) | `node-cron` daily job → email digest | Investor email |

### Socket.io Room Strategy
- Tenants join room `request:<requestId>` on ticket detail open
- Investors join room `investor:<investorId>` on login
- Server emits targeted events rather than broadcasting to all connected clients

---

## Mobile-specific Considerations (iOS First)

- **Camera / Photo Library** — Expo ImagePicker with permission request on first use; images uploaded to Express `/api/maintenance` via multipart form
- **Push Notifications** — Expo Notifications registers device token; token POSTed to `/api/users/me` and stored in `users.fcmToken`; FCM sends via Firebase Admin SDK on the server
- **Offline Drafts** — repair request form state persisted to AsyncStorage; submitted automatically when connection restored
- **Haptic Feedback** — Expo Haptics on status change confirmations and form submission
- **Biometric Auth** — Expo LocalAuthentication for quick re-entry after app backgrounding; access token re-fetched via refresh cookie on unlock

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

## Key User Flows

### Tenant

1. **Register** — choose "I am a Tenant", enter unit invite code provided by investor
2. **Dashboard** — card list of open tickets sorted by priority, each showing status badge and days since submission
3. **New Request** — multi-step form:
   - Step 1: Select category
   - Step 2: Write title and description, attach photos (camera or library)
   - Step 3: Self-assess priority (with guidance text)
   - Step 4: Review and submit → POST to `/api/maintenance`
4. **Ticket Detail** — vertical timeline of `statusUpdates` documents; Socket.io room subscription updates the timeline live without polling

### Investor

1. **Register** — choose "I am an Investor / Property Manager"
2. **Create Properties** — POST to `/api/properties`, then add units with unit numbers
3. **Invite Tenants** — generate invite code per unit; share out-of-band (email, text)
4. **Maintenance Queue** — GET `/api/maintenance` with query params for filtering/sorting; columns: property, unit, tenant, category, priority, status, days open
5. **Ticket Detail** — full request info + photo gallery + status update form:
   - Dropdown: new status
   - Date picker: ETA
   - Text field: note to tenant
   - Submit → POST `/api/maintenance/:id/status` → creates `statusUpdates` document, updates `maintenanceRequests.status`, emits Socket.io event, sends FCM push

---

## Future Modules (Post-MVP)

| Module | Description |
|---|---|
| Rent Payment | ACH / card payments via Stripe, auto-receipts, late fee tracking |
| Lease Management | Upload, e-sign, and store lease documents |
| Vendor Network | Assign repair requests to trusted contractors |
| Announcements | Investor broadcasts to all tenants in a property |
| Inspection Reports | Photo-based move-in / move-out checklists |
| Financial Dashboard | Income, expenses, and NOI per property |
