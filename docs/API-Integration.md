# API Integration: Frontend ↔ Backend — Comprehensive Reference 🔗

**Purpose:** This document describes all HTTP endpoints the frontend calls (and relevant backend endpoints), their purpose, HTTP method, path, expected request and response shapes (where available), authentication requirements, and where the frontend uses them. It is intended as an integration handbook for developers and QA engineers.

---

## Table of Contents

1. Overview & Auth flow
2. How the frontend calls APIs (client details)
3. Endpoint map by module (Auth, GenAI, Courses, Quizzes, Users, Anomalies, Notifications, Reports, Settings, Projects)
4. Example requests (samples)
5. SSE and streaming endpoints
6. Frontend wrappers & places to look
7. Security & testing checklist
8. Next steps & recommended tests
9. Missed endpoints & inconsistencies

---

## 1. Overview & Auth flow 🔐

- Base URL: frontend uses `import.meta.env.VITE_BASE_URL` as the API base.
- Authorization header (Bearer token): client injects the Firebase token stored in localStorage under `firebase-auth-token`.
  - Header: `Authorization: Bearer <token>`
- Token refresh behavior: the `openapi-fetch` client tries to call a configured `refreshTokenFunction` on 401 and retries the request if refreshed.
- Backend uses `@Authorized()` (routing-controllers) to enforce authentication on protected endpoints and can also apply CASL abilities (`@Ability(...)`) for resource-level checks.

> Where to confirm behavior: `frontend/src/lib/openapi.ts` (fetch middleware) and backend controllers in `backend/src/modules/**/controllers/*.ts`.

---

## 2. How the frontend calls APIs (client details) 💻

- Primary client: `frontend/src/lib/openapi.ts` — sets `baseUrl`, injects Authorization header and retry-on-401 behavior.
- Typed wrapper: `openapi-react-query` client is exposed as `api` and used in `frontend/src/hooks/hooks.ts` to generate typed hooks (`api.useQuery`/`api.useMutation`).
- Many components use `getApiUrl()` helpers or direct `fetch()` for custom flows (e.g., AI job flows, file uploads).

---

## 3. Endpoint map by module — purpose, method, path, body shape, auth, frontend usage 📚

> Each section lists endpoints with: Method | Path | Purpose | Body (brief) | Auth | Frontend references (files)

### A) Authentication — `/auth` 🔐

- POST /auth/signup
  - Purpose: Register a new user (Firebase + app DB)
  - Body: `SignUpBody` = {email, password (min 8), firstName, lastName?}
  - Auth: PUBLIC
  - Frontend: `api.useMutation("post","/auth/signup")` (hooks) and `frontend/src/app/pages/auth-page.tsx`
  - Backend: `backend/src/modules/auth/controllers/AuthController.ts` (method `signup`) & `AuthValidators.ts`

- POST /auth/signup/google
  - Purpose: Google sign-up; server reads provided token from Authorization header
  - Body: `GoogleSignUpBody` = {email, firstName, lastName?}
  - Auth: token required in header (handled client-side)
  - Frontend: `auth-page.tsx` uses `${VITE_BASE_URL}/auth/signup/google/`

- POST /auth/login
  - Purpose: Login via Firebase REST; server calls Google Identity Toolkit and returns token payload
  - Body: `LoginBody` = {email, password}
  - Auth: PUBLIC
  - Backend: uses Firebase endpoint via `fetch()` inside `AuthController.login`.

- PATCH /auth/change-password
  - Purpose: Change password for authenticated user
  - Body: `ChangePasswordBody` = { newPassword, newPasswordConfirm }
  - Auth: @Authorized()
  - Frontend: `api.useMutation("patch","/auth/change-password")`

---

### B) GenAI — `/genai` 🤖

> Core endpoints for creating and controlling AI jobs. All require auth and CASL ability checks.

- POST /genai/jobs
  - Purpose: Start a new job (VIDEO | PLAYLIST)
  - Body: `JobBody` includes: { type, url, transcript?, transcriptParameters?, segmentationParameters?, uploadParameters{courseId, versionId, moduleId?, sectionId?}, questionGenerationParameters? }
  - Auth: @Authorized() + ability checks
  - Frontend: `create-job.tsx`, `AISectionPage.tsx`, `frontend/src/lib/genai-api.ts`

- POST /genai/jobs/audio-provided
  - Purpose: Start job with uploaded audio file (multipart)
  - Body: multipart file + `JobBody`
  - Auth: @Authorized()

- GET /genai/jobs/:id
  - Purpose: Get job status and metadata
  - Auth: @Authorized()
  - Frontend: Job detail pages (`AISectionPage`, `AiWorkflow`) poll this endpoint.

- GET /genai/:id/tasks/:type/status
  - Purpose: Task-specific status (SEGMENTATION/TRANSCRIPT_GENERATION/QUESTION_GENERATION)
  - Auth: @Authorized()
  - Frontend: many polling points in `AISectionPage.tsx` and `AiWorkflow.tsx` (look for `getApiUrl('/genai/${id}/tasks/${task}/status')`)

- POST /genai/:id/tasks/approve/start and POST /genai/:id/tasks/approve/continue
  - Purpose: Approve task to start / approve and continue
  - Body: `ApproveStartBody` (usePrevious, parameters) for start; continue sends empty body
  - Auth: @Authorized()

- POST /genai/jobs/:id/tasks/rerun | /abort | /stop
  - Purpose: Re-run, abort, or stop tasks
  - Body (rerun): `RerunTaskBody` (usePrevious, parameters)
  - Auth: @Authorized()

- PATCH /genai/jobs/:id/edit/segment-map
  - Purpose: Edit segment map (patch segmentation results)
  - Body: `EditSegmentMapBody` { segmentMap, index }
  - Auth: @Authorized()
  - Frontend: editing UI calls `/genai/jobs/${jobId}/edit/segment-map`

- GET /genai/:id/live
  - Purpose: SSE endpoint for live job updates (server-sent events)
  - Auth: @Authorized()
  - Frontend: `connectToLiveStatusUpdates` helper in `frontend/src/lib/genai-api.ts` connects here.

Reference validators: `backend/src/modules/genAI/classes/validators/GenAIValidators.ts`

---

### C) Courses & Content — `/courses` and related endpoints 📚

- POST /courses/ — create course
- GET /courses/:courseId — get course data
- POST /:courseId/versions — add version
- Module endpoints: create/put/delete/move/toggle-visibility
  - Examples: POST /versions/:versionId/modules, PUT /versions/:versionId/modules/:moduleId, etc.
- Section endpoints: POST /versions/:versionId/modules/:moduleId/sections; PUT .../sections/:sectionId
- Items: POST /versions/:versionId/modules/:moduleId/sections/:sectionId/items, GET /.../items, PUT /versions/:versionId/items/:itemId
- CSV upload: POST /:courseId/versions/:versionId/module/:moduleId/section/:sectionId/items/csv
- Auth: Creation & modification require `@Authorized()`
- Frontend: `frontend/src/hooks/hooks.ts` contains wrappers for all these (see `useCreateCourse`, `useCourseById`, etc.)

---

### D) Quizzes & Attempts — `/quizzes` 📝

- POST /quizzes/{quizId}/attempt — create attempt
- POST /quizzes/{quizId}/attempt/{attemptId}/save — autosave
- POST /quizzes/{quizId}/attempt/{attemptId}/submit — submit final
- GET endpoints: details, analytics, performance, results, attempts export
- Question endpoints: POST/PUT/DELETE /quizzes/questions, POST /quizzes/questions/{questionId}/flag
- Auth: `@Authorized()` for most operations
- Frontend: Course/quiz UI and `hooks.ts` calls.

---

### E) Users, Enrollments & Progress 👥

- POST /users/enrollments/courses/{courseId}/versions/{versionId} — enroll
- POST /users/{userId}/enrollments/.../unenroll — unenroll
- GET /users/enrollments — list
- Progress tracking: POST /progress/courses/{courseId}/versions/{versionId}/start & /stop
- GET /users/watchTime/total
- Auth: Protected

---

### F) Anomalies — `/anomalies` 🚨

- POST /anomalies/record/image (multipart) — upload image anomaly
- POST /anomalies/record/audio (multipart) — upload audio anomaly
- GET endpoints to list anomalies per course/version/user
- DELETE /anomalies/:id — delete an anomaly
- Frontend: `frontend/src/lib/genai-api.ts` has `recordImage` and `recordAudio` helpers; `hooks.ts` includes multipart mutation wrappers.

---

### G) Notifications / Invites — `/notifications/invite` ✉️

- POST /notifications/invite/courses/{courseId}/versions/{versionId} — send invite
- POST /notifications/invite/courses/{courseId}/versions/{versionId}/bulk — bulk invites
- GET /notifications/invite/courses/{courseId}/versions/{versionId} — list invites
- POST /notifications/invite/resend/{inviteId} — resend invite
- POST /notifications/invite/{inviteId}?action=REJECTED — process invite from recipient
- Auth: Protected for creating & listing invites
- Frontend: `hooks.ts` functions + `useProcessInvites` direct fetch helper in `hooks.ts` (see `useProcessInvites(inviteId, action)` implementation)

---

### H) Reports & Exports — `/reports` & /quizzes/\*/attempts/export 📊

- POST /reports/ — generate report
- GET /reports/:courseId/:versionId — fetch reports
- GET /reports/:reportId — report details
- GET /quizzes/{quizId}/attempts/export — export attempts (frontend uses a direct fetch call with `VITE_BASE_URL`)

---

### I) Settings, Course Registration & Projects ⚙️

- Course settings: POST /setting/course-setting, GET /setting/course-setting/:courseId/:versionId, PUT/DELETE proctoring endpoints
- Course registration: endpoints to build forms, create registrations, update bulk statuses
- Projects: POST /project/ and related endpoints (see `ProjectController`)
- Auth: Typically protected with `@Authorized()` and role-based constraints

---

## 4. Example requests (samples) ✍️

- Signup

```http
POST /auth/signup
Content-Type: application/json

{ "email": "user@example.com", "password": "SecureP@ss1", "firstName": "John", "lastName": "Smith" }
```

- Create GenAI Job

```http
POST /genai/jobs
Authorization: Bearer <token>
Content-Type: application/json

{
  "type": "VIDEO",
  "url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ",
  "uploadParameters": {"courseId": "<id>", "versionId": "<id>"},
  "segmentationParameters": {"lam": 4.6, "runs": 25},
  "questionGenerationParameters": {"SOL": 2, "prompt": "Focus on conceptual questions..."}
}
```

### 4.1 Detailed Request Body Structures (examples)

Below are concise request body schemas for commonly used endpoints. Use these as quick references; authoritative field validation exists in backend `classes/validators` files.

- POST /auth/signup (SignUpBody)

```json
{
  "email": "user@example.com",
  "password": "SecureP@ssw0rd",
  "firstName": "John",
  "lastName": "Smith" // optional
}
```

- POST /auth/signup/google (GoogleSignUpBody)

```json
{
  "email": "user@example.com",
  "firstName": "John",
  "lastName": "Smith" // optional
}
```

- POST /auth/login (LoginBody)

```json
{
  "email": "user@example.com",
  "password": "SecureP@ssw0rd"
}
```

- PATCH /auth/change-password (ChangePasswordBody)

```json
{
  "newPassword": "NewP@ss123",
  "newPasswordConfirm": "NewP@ss123"
}
```

- POST /genai/jobs (JobBody)

```json
{
  "type": "VIDEO",
  "url": "https://www.youtube.com/watch?v=...",
  "uploadParameters": {
    "courseId": "<mongoId>",
    "versionId": "<mongoId>",
    "moduleId": "<mongoId> (optional)",
    "sectionId": "<mongoId> (optional)",
    "videoItemBaseName": "video_item",
    "quizItemBaseName": "quiz_item",
    "questionsPerQuiz": 5
  },
  "transcript": { "chunks": [{ "timestamp": [0, 5], "text": "..." }] },
  "transcriptParameters": { "language": "en", "modelSize": "large" },
  "segmentationParameters": { "lam": 4.6, "runs": 25, "noiseId": -1 },
  "questionGenerationParameters": {
    "SOL": 2,
    "SML": 1,
    "NAT": 1,
    "prompt": "..."
  }
}
```

- POST /genai/jobs/audio-provided (multipart)

Form fields:

- file: binary audio file
- body: (JSON) same as `JobBody` or individual fields such as `uploadParameters`

- POST /genai/:id/tasks/approve/start (ApproveStartBody)

```json
{
  "usePrevious": false,
  "parameters": {
    /* task-specific parameter object */
  }
}
```

- POST /genai/jobs/:id/tasks/rerun (RerunTaskBody)

```json
{
  "usePrevious": true,
  "parameters": {
    /* optional override parameters */
  }
}
```

- PATCH /genai/jobs/:id/edit/segment-map (EditSegmentMapBody)

```json
{
  "segmentMap": [
    /* array of segment objects {start, end, label, metadata?} */
  ],
  "index": 0
}
```

- PATCH /genai/jobs/:id/edit/question (EditQuestionData)

```json
{
  "questionData": {
    /* question json object as stored */
  },
  "index": 1
}
```

- POST /courses/ (CreateCourseBody)

```json
{
  "name": "Intro to Chemistry",
  "description": "Course description",
  "versionName": "v1",
  "versionDescription": "Initial release"
}
```

- POST /courses/versions/{versionId}/modules (Create Module)

```json
{
  "title": "Module title",
  "description": "Optional module description",
  "order": 1
}
```

- POST /versions/{versionId}/modules/{moduleId}/sections (Create Section)

```json
{
  "title": "Section title",
  "description": "Optional section description",
  "order": 1
}
```

- POST /versions/{versionId}/modules/{moduleId}/sections/{sectionId}/items (Create Item)

```json
{
  "title": "Video Lecture 1",
  "type": "VIDEO",
  "content": { "url": "https://..." },
  "order": 1
}
```

- POST /quizzes/{quizId}/attempt (Create Attempt)

```json
{
  "userId": "<userId>",
  "startedAt": "2025-01-01T12:00:00.000Z",
  "answers": []
}
```

- POST /quizzes/{quizId}/attempt/{attemptId}/save (Save Attempt)

```json
{
  "answers": [{ "questionId": "...", "value": ["A"] }],
  "lastSavedAt": "2025-01-01T12:05:00.000Z"
}
```

- POST /anomalies/record/image (multipart)

Form fields:

- file: image with `Content-Type: image/jpeg|png`
- metadata: JSON string (e.g., `{ "courseId": "...", "versionId": "...", "itemId": "..." }`)

- POST /notifications/invite/courses/{courseId}/versions/{versionId}

```json
{
  "recipients": [{ "email": "student@example.com", "role": "student" }],
  "message": "Optional invite message",
  "sendEmail": true
}
```

- POST /reports/ (Generate Report)

```json
{
  "courseId": "<id>",
  "versionId": "<id>",
  "filters": { "dateRange": { "from": "2025-01-01", "to": "2025-12-31" } },
  "type": "COURSE_SUMMARY"
}
```

> See validators for full validation rules in `backend/src/modules/*/classes/validators` (e.g., `GenAIValidators.ts`, `AuthValidators.ts`, `QuizValidators.ts`).

- Connect to SSE live updates

```js
const es = new EventSource(`${VITE_BASE_URL}/genai/${jobId}/live`, {
  withCredentials: true,
});
```

---

## 5. SSE / streaming endpoints — notes 🔄

- `GET /genai/:id/live` is an SSE endpoint used to push real-time job status updates. Client must maintain the connection and handle reconnection logic.
- Ensure CORS + credentials and token validation work for SSE on the deployed environment.

---

## 6. Frontend wrappers & import points 📎

- API client: `frontend/src/lib/openapi.ts` — baseUrl and token injection
- Typed hooks: `frontend/src/hooks/hooks.ts` — includes `api.useQuery` and `api.useMutation` wrappers for nearly all endpoints
- GenAI helpers: `frontend/src/lib/genai-api.ts` — convenience functions, SSE connector, record audio/image helpers.
- Direct fetch usage: Search for `fetch(` and `getApiUrl()` in `frontend/src/**/*` for places where the `openapi` client is not used.

---

## 7. Security & testing checklist ✅

- Verify protected endpoints return 401 when token missing/expired.
- Confirm token refresh logic behaves correctly: on 401, `refreshTokenFunction` is invoked and request retried exactly once.
- For file upload endpoints (multipart), test missing `Content-Type` handling and large file behavior.
- For GenAI job create/rerun/abort: test ability enforcement (CASL roles) — only allowed instructors/admins for course can create jobs for that course version.
- SSE tests: reconnect behavior and auth failures.
- Rate-limiting: ensure endpoints like login & job creation enforce expected rate limits.

---

## 8. Next steps & recommended deliverables 🧭

- Add an autogenerated CSV of all endpoints (method, path, auth, frontend file references, validator reference). This assists QA for test case generation.
- Add a smoke test suite (e2e) covering: signup/login, token refresh, create GenAI job (mock provider), SSE subscription, file upload, and quiz attempt flow.
- Consider publishing this doc to `docs/` and linking it from project README for cross-team visibility.

---

## Reference file locations in repo 🔎

- Frontend API client: `frontend/src/lib/openapi.ts`
- Frontend hooks: `frontend/src/hooks/hooks.ts`
- GenAI UI: `frontend/src/app/pages/teacher/AISectionPage.tsx`, `AiWorkflow.tsx`, `create-job.tsx`.
- Backend controllers: `backend/src/modules/*/controllers/*.ts`
- Backend validators: `backend/src/modules/*/classes/validators/*.ts`
- GenAI backend: `backend/src/modules/genAI/controllers/GenAIController.ts` & `validators/GenAIValidators.ts`

---

## 9. Missed endpoints & inconsistencies 🔍

During the scan I found a few gaps and mismatches you may want to address. Below are exact items, where they appear, and recommended actions.

- **Frontend-only / missing backend implementations**
  - `/auth/verify` — referenced in `frontend/src/hooks/hooks.ts` but there is no corresponding backend route. Action: implement or remove client usage.
  - `/auth/google` — used by frontend (`useLoginWithGoogle`) but backend exposes `/auth/signup/google` (and `/auth/login`). Clarify whether the intended server path is `/auth/google` or update client calls to match.
  - `/auth/signup/verify` — present in frontend hooks but not implemented on the backend.

- **Frontend references to non-existent backend endpoints**
  - `/activity/known-faces` — commented reference in `frontend/src/components/ai/FaceRecognitionComponent.tsx`, but no `/activity` controller exists on the backend. Action: either add a backend endpoint or remove/commented frontend references.

- **Path parameter mismatches to confirm**
  - Enrollments create: frontend hook uses `POST /users/enrollments/courses/{courseId}/versions/{courseVersionId}`, while backend expects `POST /users/:userId/enrollments/courses/:courseId/versions/:versionId`. Decide whether backend should accept `/users/enrollments/...` as a current-user shorthand or frontend must supply `userId`.

- **Project endpoints not explicitly documented earlier**
  - `POST /project/` — submit project (backend: `ProjectController.submitProject`).
  - `GET /project/course/:courseId/version/:versionId/submissions` — list submissions.

- **Webhook security recommendation**
  - `/genAI/webhook/` is intentionally public for AI worker callbacks. For production, consider adding HMAC/shared-secret validation or IP whitelisting to avoid spoofed updates.

- **Misc / QA notes**
  - Ensure rarely-used bulk endpoints (enrollment/progress bulk updates, export endpoints) are included in the QA CSV and smoke tests.

---

If you'd like, I can also:

- Export a CSV mapping endpoints → frontend usage → validator path for QA ✅
- Add a shorter quick-reference README in `frontend/` and `backend/` (one-line for each endpoint group) ✅

---

_Doc generated automatically by repository scan — update as code changes._
