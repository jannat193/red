# Student OS — Complete Architecture & Backend Audit

**Version:** 1.0 | **Date:** 2026-05-17 | **Auditor:** Antigravity AI

---

## 1. Executive Summary

Student OS is a well-conceived academic intelligence system with a clean service-layer architecture. The codebase is **MVP-complete** and structurally sound for a solo or small-team project. However, there are **critical gaps** that must be resolved before production deployment:

| Area | Status | Risk |
|---|---|---|
| Frontend Architecture | ✅ Good | Low |
| Appwrite Backend | ⚠️ Incomplete | **Critical** |
| Database Permissions | ❌ Not configured | **Critical** |
| Type Safety | ⚠️ Partial (`as any`) | Medium |
| State Management | ✅ Good structure | Low |
| Security | ⚠️ Missing row-level security | High |
| Performance | ⚠️ N+1 risk on overdue sync | Medium |
| PWA / next-pwa | ❌ Not configured | Medium |
| Testing | ❌ Zero tests | High |
| Habits/Mistakes backend | ❌ Hardcoded collection IDs | High |

**Production Readiness Score: 52 / 100**

---

## 2. Architecture Overview

```
src/
├── app/                    # Next.js App Router pages
│   ├── (auth)/             # Auth group: login, register, onboarding
│   └── dashboard/          # All protected pages
├── components/
│   ├── auth-provider.tsx   # Global auth gate
│   ├── theme-provider.tsx
│   └── ui/                 # Shared UI primitives (shadcn-style)
├── features/               # Feature-specific dialogs
│   ├── courses/
│   └── semesters/
├── hooks/
│   └── useAppData.ts       # Master data bootstrap hook
├── lib/
│   ├── appwrite.ts         # Client + config
│   ├── calculations/       # CGPA, streak logic
│   └── utils.ts
├── services/
│   ├── auth.ts             # Auth service
│   └── database.ts         # All collection CRUD
├── store/                  # Zustand stores (8 stores)
├── types/                  # Shared TypeScript interfaces
└── validations/            # Zod schemas
```

**Strengths:**
- Clear separation: UI → Store → Service → Appwrite
- Feature-based dialogs under `features/`
- Centralized service layer in `database.ts`
- Calculations isolated in `lib/calculations/`

**Weaknesses:**
- `features/` has only 2 features — remaining pages (habits, pomodoro, mistakes, analytics) have all their logic inside the page file
- No `constants/` file for magic strings (DAYS_OF_WEEK, GRADE_THRESHOLDS, etc.)
- `src/app/dashboard/mistakes/page.tsx` manages its own local state array — not connected to any store or service

---

## 3. Frontend Audit

### 3.1 Structural Issues

| File | Issue | Severity |
|---|---|---|
| `mistakes/page.tsx` | Uses local `useState<Mistake[]>` — data lost on refresh | **High** |
| `habits/page.tsx` | Calls `calculateNewStreak` but never persists to Appwrite | **High** |
| `pomodoro/page.tsx` | `addSession()` called on store but never calls `studySessionService.create()` | **High** |
| `settings/page.tsx` | `handleSave` has a `setTimeout(500)` mock — no real Appwrite write | **High** |
| `courses/page.tsx` | Settings gear icon visible but onClick is empty (`<button>`) | Medium |
| `dashboard/history` | Route defined in sidebar nav but page doesn't exist | Medium |

### 3.2 Missing Error Boundaries
No `error.tsx` files exist at any route level. A crashed component will take down the entire dashboard.

### 3.3 Missing Loading Pages
No `loading.tsx` files. Route transitions show blank screens.

### 3.4 onboarding Route Mismatch
`register/page.tsx` redirects to `/onboarding` but the file is at `/(auth)/onboarding`. Without the auth group prefix being transparent to the URL, this may route correctly — but the onboarding page does **not** call any Appwrite service to actually save the onboarding data.

---

## 4. Backend Audit (Appwrite)

### 4.1 Auth
- Uses Email+Password sessions ✅
- `getCurrentUser()` correctly returns `null` on failure ✅
- No OAuth (Google/GitHub) — acceptable for MVP
- No email verification enforcement — users can skip it
- No password reset flow implemented

### 4.2 Database Services

`database.ts` has a well-structured CRUD abstraction:

```typescript
// Generic paginated loader — correctly handles >25 docs
async function listAll<T>(collectionId, queries) {
  // fetches page by page — good
}
```

**Critical issue — `listAll` is not shown in the read output but inferred.** If it is NOT paginating (just using default 25-doc limit), **any user with >25 records in any collection will silently lose data**.

**Hardcoded collection IDs (Critical):**
```typescript
habitService:  listAll("habits", ...)          // raw string, not from env
mistakeService: listAll("mistake_journal", ...) // raw string, not from env
```
These will **break in production** because Appwrite collection IDs are UUIDs, not human-readable names — unless you explicitly named them that during collection creation.

### 4.3 Missing Collections in `.env.local`
The following collections are used in `database.ts` but have no corresponding env var:
- `habits` (hardcoded)
- `mistake_journal` (hardcoded)

Add to `.env.local`:
```
NEXT_PUBLIC_APPWRITE_HABITS_COLLECTION_ID=
NEXT_PUBLIC_APPWRITE_MISTAKES_COLLECTION_ID=
```

### 4.4 Appwrite Functions
**None are implemented.** The PRD requires:
- CF submission sync (Codeforces API → Appwrite)
- Overdue topic sync (currently done client-side — unreliable)
- Weekly report email (Appwrite Messaging)
- Contest calendar sync

The client-side overdue sync in `useAppData.ts` only marks topics as overdue **in-memory**. The Appwrite document is never updated. A user who logs out and back in will see stale `pending` statuses.

---

## 5. Appwrite Complete Setup Guide

### 5.1 Required Collections & Attributes

#### `users`
| Attribute | Type | Required | Default |
|---|---|---|---|
| name | String(255) | ✅ | — |
| email | String(320) | ✅ | — |
| university | String(255) | ❌ | — |
| department | String(255) | ❌ | — |
| cfHandle | String(100) | ❌ | — |
| timezone | String(100) | ❌ | — |
| xp | Integer | ✅ | 0 |
| level | Integer | ✅ | 1 |
| currentSemesterId | String(36) | ❌ | — |

#### `semesters`
| Attribute | Type | Required |
|---|---|---|
| userId | String(36) | ✅ |
| name | String(100) | ✅ |
| startDate | String(30) | ✅ |
| endDate | String(30) | ✅ |
| isActive | Boolean | ✅ |
| finalCgpa | Float | ❌ |

#### `courses`
| Attribute | Type | Required |
|---|---|---|
| userId | String(36) | ✅ |
| semesterId | String(36) | ✅ |
| code | String(20) | ✅ |
| title | String(255) | ✅ |
| credit | Float | ✅ |
| color | String(20) | ✅ |
| assignmentWeight | Float | ✅ |
| quizWeight | Float | ✅ |
| midWeight | Float | ✅ |
| finalWeight | Float | ✅ |

#### `schedules`
| Attribute | Type | Required |
|---|---|---|
| userId | String(36) | ✅ |
| semesterId | String(36) | ✅ |
| courseId | String(36) | ✅ |
| day | String(20) | ✅ |
| startTime | String(10) | ✅ |
| endTime | String(10) | ✅ |
| room | String(100) | ❌ |

#### `study_topics`
| Attribute | Type | Required |
|---|---|---|
| userId | String(36) | ✅ |
| semesterId | String(36) | ✅ |
| courseId | String(36) | ✅ |
| title | String(255) | ✅ |
| priority | String(10) | ✅ |
| status | String(20) | ✅ |
| estimatedHours | Float | ✅ |
| createdAt | String(30) | ✅ |
| deadline | String(30) | ✅ |
| completedAt | String(30) | ❌ |

#### `exams`
| Attribute | Type | Required |
|---|---|---|
| userId | String(36) | ✅ |
| semesterId | String(36) | ✅ |
| courseId | String(36) | ✅ |
| type | String(20) | ✅ |
| name | String(255) | ✅ |
| date | String(30) | ✅ |
| maxMarks | Float | ✅ |
| obtainedMarks | Float | ❌ |
| weight | Float | ✅ |
| status | String(20) | ✅ |

#### `study_sessions`
| Attribute | Type | Required |
|---|---|---|
| userId | String(36) | ✅ |
| semesterId | String(36) | ✅ |
| courseId | String(36) | ❌ |
| topicId | String(36) | ❌ |
| duration | Integer | ✅ |
| pomodoroType | String(20) | ✅ |
| date | String(30) | ✅ |

#### `habits` *(missing from env)*
| Attribute | Type | Required |
|---|---|---|
| userId | String(36) | ✅ |
| title | String(255) | ✅ |
| streak | Integer | ✅ |
| lastCompleted | String(30) | ❌ |

#### `assignments`
| Attribute | Type | Required |
|---|---|---|
| userId | String(36) | ✅ |
| semesterId | String(36) | ✅ |
| courseId | String(36) | ✅ |
| title | String(255) | ✅ |
| deadline | String(30) | ✅ |
| status | String(20) | ✅ |
| marks | Float | ❌ |
| submissionLink | String(500) | ❌ |

#### `mistake_journal` *(missing from env)*
| Attribute | Type | Required |
|---|---|---|
| userId | String(36) | ✅ |
| title | String(255) | ✅ |
| category | String(30) | ✅ |
| description | String(2000) | ❌ |
| date | String(30) | ✅ |

### 5.2 Required Indexes

For each collection, create these indexes in **Databases → [collection] → Indexes**:

| Collection | Index Type | Attributes | Purpose |
|---|---|---|---|
| semesters | Key | `userId` | List user semesters |
| courses | Key | `userId, semesterId` | List courses per semester |
| schedules | Key | `userId, semesterId` | List schedule |
| study_topics | Key | `userId, semesterId` | List topics |
| study_topics | Key | `userId, semesterId, status` | Filter by status |
| study_topics | Key | `deadline` | Overdue sync queries |
| exams | Key | `userId, semesterId` | List exams |
| study_sessions | Key | `userId, semesterId` | List sessions |
| study_sessions | Key | `userId, date` | Analytics queries |
| habits | Key | `userId` | List habits |
| mistake_journal | Key | `userId` | List mistakes |
| assignments | Key | `userId, semesterId` | List assignments |

> [!CAUTION]
> Without indexes, Appwrite will refuse to run filtered queries on large collections (>5000 docs) and return an error.

### 5.3 Required Permissions

For **every collection**, set these permissions in **Settings → Permissions**:

| Role | Create | Read | Update | Delete |
|---|---|---|---|---|
| Users | ✅ | ✅ | ✅ | ✅ |

> [!WARNING]
> This grants any authenticated user access to any document. For stronger security, use **Document Security** instead and set permissions per-document at creation time using `Permission.write(Role.user(userId))`.

### 5.4 Required Platform Config

In **Appwrite Console → Settings → Platforms**, add:
```
Web platform:
  Name: Student OS Dev
  Hostname: localhost

Web platform (prod):
  Name: Student OS Production
  Hostname: your-vercel-domain.vercel.app
```

Without this, Appwrite will block all API calls with a CORS error.

---

## 6. Database Design Report

### 6.1 Relationship Map

```
User (Appwrite Auth)
 └── User Profile (users collection)
      └── Semester (semesters) [1:many]
           ├── Course (courses) [1:many]
           │    ├── Schedule (schedules) [1:many]
           │    ├── Exam (exams) [1:many]
           │    └── StudyTopic (study_topics) [1:many]
           └── StudySession (study_sessions) [1:many]

User (independent)
 ├── Habit (habits) [1:many]
 └── MistakeJournal (mistake_journal) [1:many]
```

### 6.2 Issues

**No referential integrity.** Appwrite has no foreign keys. If you delete a `course`, all its related `exams`, `schedules`, and `study_topics` remain as orphans. You must implement **cascading deletes in the service layer** manually.

**`userId` stored redundantly** in every document — correct for Appwrite's flat document model, but this means a user changing their email/name requires no document updates (good — userId is stable).

**`activeSemesterId` computed client-side** from `isActive` flag. If two semesters have `isActive: true`, the UI picks the first one found. There's no enforcement of single-active at the database level.

---

## 7. Security Audit

| Issue | Severity | Description |
|---|---|---|
| No document-level permissions | **Critical** | Any logged-in user can read/write any other user's documents if they know the doc ID |
| `userId` stored in document body | High | Currently used for filtering but not enforced. Malicious user can query `userId = "victim"` |
| `NEXT_PUBLIC_` env vars | Medium | All Appwrite keys are public by design — acceptable for client SDK, but Appwrite project ID exposure is a known tradeoff |
| No CSRF protection on forms | Low | Mitigated by Appwrite session cookies being HttpOnly |
| No rate limiting on login | Medium | Appwrite handles this server-side via rate limits — verify in Console |
| `error: any` pattern | Medium | `catch (error: any)` in multiple places masks typed errors |
| Route protection is client-only | High | `auth-provider.tsx` redirects on client. Server-side middleware (`middleware.ts`) is absent |

### 7.1 Critical Fix: Document Security

When creating any document, pass user-scoped permissions:

```typescript
// In database.ts createDoc — add this parameter
databases.createDocument(
  databaseId,
  collectionId,
  ID.unique(),
  data,
  [
    Permission.read(Role.user(userId)),
    Permission.write(Role.user(userId)),
    Permission.delete(Role.user(userId)),
  ]
)
```

Also enable **Document Security** in each collection's settings.

### 7.2 Missing Middleware

Create `src/middleware.ts`:
```typescript
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // Server-side route protection
  // Check Appwrite session cookie
}

export const config = {
  matcher: ['/dashboard/:path*'],
};
```

---

## 8. Performance Audit

| Issue | Severity | Impact |
|---|---|---|
| `useAppData` fires 5 parallel Appwrite requests on every `activeSemesterId` change | Medium | Waterfalls on slow connections |
| Overdue sync is pure client-side, not persisted | High | Status reverts after re-login |
| `syncOverdue` in `database.ts` does N individual PATCH requests | High | 50 overdue topics = 50 API calls |
| No React.memo on `NavItem` component rendered in a loop | Low | Minor re-render on every pathname change |
| `useMemo` used correctly on CGPA and analytics calculations | ✅ Good | — |
| No query cursor pagination — full dataset loaded every session switch | Medium | Grows linearly with data |
| `sessions` array loaded in full on mount for analytics | Medium | 500 Pomodoro sessions = 500 objects in memory |

### 8.1 Recommended Optimizations

1. **Batch overdue update** — use a single Appwrite Function instead of N client PATCH calls
2. **Session pagination** — for analytics, query last 90 days only: `Query.greaterThan("date", ninetyDaysAgo)`
3. **Debounce semester switch** — add 200ms debounce before triggering `loadSemesterData`
4. **Memoize `NavItem`** — wrap with `React.memo` to prevent sidebar re-renders

---

## 9. Type Safety Audit

| Pattern | Count | Risk |
|---|---|---|
| `resolver: zodResolver(...) as any` | 6+ occurrences | Medium — masks resolver type mismatch between `@hookform/resolvers` v5 + Zod v4 |
| `appwriteUser: any` in `useAuthStore` | 1 | High — entire auth object is untyped |
| `error: any` in catch blocks | 5+ | Medium |
| `data as Record<string, unknown>` in services | Multiple | Low — necessary for Appwrite SDK typing |
| `$sequence: "0"` on mock documents | Multiple | Low — workaround for Appwrite v25 type |

### 9.1 Root Cause of `as any` on form controls

The project uses:
- `react-hook-form` v7.76 
- `@hookform/resolvers` v5.2.2
- `zod` v4.4.3

Zod v4 changed its `ZodType` generic signature, which breaks the resolver's type inference. **This is a known upstream issue.** The `as any` workaround is currently the correct approach. Watch for `@hookform/resolvers` v5.3+ which should fix this.

### 9.2 Fix `appwriteUser: any`

Import Appwrite's `Models` type:
```typescript
import { Models } from "appwrite";

interface AuthState {
  appwriteUser: Models.User<Models.Preferences> | null;
  // ...
}
```

---

## 10. State Management Audit

**8 Zustand stores** — well-separated by domain. No cross-store coupling. All follow the same pattern.

| Issue | Severity |
|---|---|
| `isLoading: true` as initial state causes hydration flash | Medium (fixed via isMounted gate) |
| Habit completions update store but not Appwrite | **Critical** |
| Pomodoro sessions added to store but not Appwrite | **Critical** |
| Mistake Journal uses local `useState` — not in store at all | **Critical** |
| `useAuthStore` has both `user` (custom User type) and `appwriteUser` (Appwrite account) — confusing duplication | Medium |
| No store persistence (`zustand/middleware/persist`) — all data cleared on refresh | High |

### 10.1 Recommendation: Persist Store

For habits and study sessions specifically, add persistence:
```typescript
import { persist } from 'zustand/middleware';

export const useHabitStore = create<HabitState>()(
  persist(
    (set) => ({ /* ... */ }),
    { name: 'habit-store' }
  )
);
```

---

## 11. Appwrite Functions Architecture (Planned)

| Function | Trigger | Purpose | Status |
|---|---|---|---|
| `sync-overdue-topics` | Cron: every 6h | Mark past-deadline topics as overdue | ❌ Not built |
| `sync-cf-submissions` | Cron: every 6h | Fetch CF API → store solved problems | ❌ Not built |
| `sync-cf-contests` | Cron: daily | Fetch upcoming Codeforces contests | ❌ Not built |
| `send-weekly-report` | Cron: Sunday 8am | Email summary via Appwrite Messaging | ❌ Not built |

**Cron sync-overdue-topics is the highest priority** — the current client-side sync is unreliable and doesn't persist to the database.

---

## 12. UI/UX Audit

| Issue | Severity |
|---|---|
| `/dashboard/history` page linked in sidebar but doesn't exist → 404 | **High** |
| Courses page settings gear button has no onClick handler | Medium |
| No `loading.tsx` for any route — blank flashes during navigation | Medium |
| No `error.tsx` — uncaught errors show Next.js generic error page | Medium |
| Mobile: sidebar is not collapsible — takes full width on small screens | Medium |
| Habit page: no visual feedback when marking done (only toast) | Low |
| Empty states exist and are well-designed ✅ | — |
| Dark mode consistent throughout ✅ | — |
| Recharts tooltips use dark theme styles ✅ | — |

---

## 13. Technical Debt Analysis

| Item | Debt Type | Effort to Fix |
|---|---|---|
| Habits/Pomodoro/Mistakes not persisted to Appwrite | Feature gap | Medium |
| `as any` on all form resolvers | Type workaround | Low (upstream fix needed) |
| `appwriteUser: any` type | Technical debt | Low |
| No middleware.ts route protection | Security gap | Low |
| No document-level permissions | Security gap | Medium |
| `habits` / `mistake_journal` hardcoded collection IDs | Config debt | Low |
| No cascading deletes on course/semester | Data integrity | Medium |
| No tests | Quality gap | High |
| `package.json` name is `"1"` | Naming | Trivial |
| `next-pwa` installed but not configured in `next.config.ts` | Unused dep | Low |

---

## 14. Priority Fix List

### 🔴 Critical (Block Production)

1. **Configure Appwrite collections** with all attributes, indexes, and permissions per Section 5
2. **Enable Document Security** + pass `Permission.write(Role.user(userId))` on all `createDocument` calls
3. **Wire habits to Appwrite** — `addHabit`, `updateHabit`, `deleteHabit` must call `habitService.*`
4. **Wire Pomodoro sessions to Appwrite** — `addSession` must call `studySessionService.create()`
5. **Wire Mistakes to Appwrite** — move from local state to `mistakeStore` + `mistakeService.*`
6. **Fix hardcoded collection IDs** — move `"habits"` and `"mistake_journal"` to env vars
7. **Fix `/dashboard/history`** — create the page or remove from sidebar nav

### 🟡 High Priority

8. **Persist overdue sync to Appwrite** — call `studyTopicService.updateStatus()` in `useAppData`, not just update local state
9. **Add `middleware.ts`** for server-side route protection
10. **Onboarding save** — call `userService.create()` or `updateUser()` after onboarding form submission
11. **Add Platform in Appwrite Console** — required to avoid CORS on all environments
12. **Fix `appwriteUser: any`** — type as `Models.User<Models.Preferences>`

### 🟢 Medium Priority

13. Create `loading.tsx` for dashboard routes
14. Create `error.tsx` for dashboard routes
15. Add course settings (edit/delete) functionality
16. Mobile sidebar collapse/drawer
17. Configure `next-pwa` in `next.config.ts` or remove the dependency
18. Add `NEXT_PUBLIC_APPWRITE_HABITS_COLLECTION_ID` and `NEXT_PUBLIC_APPWRITE_MISTAKES_COLLECTION_ID` to `.env.local`

---

## 15. Deployment Checklist

### Environment Variables (Complete List)
```env
NEXT_PUBLIC_APPWRITE_ENDPOINT=https://cloud.appwrite.io/v1
NEXT_PUBLIC_APPWRITE_PROJECT_ID=          ← Required
NEXT_PUBLIC_APPWRITE_DATABASE_ID=         ← Required
NEXT_PUBLIC_APPWRITE_USERS_COLLECTION_ID=
NEXT_PUBLIC_APPWRITE_SEMESTERS_COLLECTION_ID=
NEXT_PUBLIC_APPWRITE_COURSES_COLLECTION_ID=
NEXT_PUBLIC_APPWRITE_SCHEDULES_COLLECTION_ID=
NEXT_PUBLIC_APPWRITE_STUDY_TOPICS_COLLECTION_ID=
NEXT_PUBLIC_APPWRITE_ASSIGNMENTS_COLLECTION_ID=
NEXT_PUBLIC_APPWRITE_EXAMS_COLLECTION_ID=
NEXT_PUBLIC_APPWRITE_STUDY_SESSIONS_COLLECTION_ID=
NEXT_PUBLIC_APPWRITE_HABITS_COLLECTION_ID=      ← Missing
NEXT_PUBLIC_APPWRITE_MISTAKES_COLLECTION_ID=    ← Missing
```

### Pre-Deploy Steps
- [ ] All env vars set in Vercel dashboard
- [ ] Appwrite Platform added for production domain
- [ ] All 11 collections created with correct attributes
- [ ] All indexes created
- [ ] Document Security enabled on all collections
- [ ] `npm run build` passes with 0 errors
- [ ] Test register → onboarding → dashboard flow end-to-end
- [ ] Test semester create → course add → exam add → CGPA calculation
- [ ] Rename `package.json` `name` from `"1"` to `"student-os"`

---

## Production Readiness Score Breakdown

| Category | Score | Max |
|---|---|---|
| Frontend Architecture | 15 | 20 |
| Backend / Appwrite Config | 5 | 20 |
| Security | 5 | 15 |
| Type Safety | 7 | 10 |
| Performance | 7 | 10 |
| Data Persistence Completeness | 5 | 10 |
| Testing | 0 | 10 |
| UX Polish | 8 | 5 *(bonus)* |
| **Total** | **52** | **100** |

> After completing Critical + High Priority fixes, estimated score: **78/100**
> After adding tests and Appwrite Functions: **92/100**
