# Quiz Lifecycle — APIs & Detailed Reference 🧩📘

**Purpose:** This document describes the full lifecycle of a quiz on the platform (authoring → configuration → delivery → grading → review) and explains every API involved: method, path, purpose, request/response shapes (validator references), authentication, frontend callers, and important implementation notes.

---

## Quick overview — lifecycle stages

1. Authoring questions & building question banks (teacher)
2. Creating a quiz item and associating question banks (teacher / course)
3. Student workflow: start an attempt → save answers (autosave) → submit attempt
4. Automatic grading and feedback generation (server)
5. Instructor workflows: view attempts, regrade/override, add feedback
6. Analytics & exports (results, performance, CSV exports)
7. Flags & moderation (questions flagged, resolve flags)

---

## Notation & where to find request/response schemas

- Validator classes live under: `backend/src/modules/quizzes/classes/validators/QuizValidator.ts`, `QuestionValidator.ts`, `QuestionBankValidator.ts`.
- Controller implementations: `backend/src/modules/quizzes/controllers/{AttemptController,QuizController,QuestionController,QuestionBankController}.ts`.
- Services: `backend/src/modules/quizzes/services/{AttemptService,QuizService,QuestionService,QuestionBankService}.ts`.
- Frontend typed hooks: `frontend/src/hooks/hooks.ts` (examples: `api.useMutation("post", "/quizzes/{quizId}/attempt")`).

### Repo links (quick jump links)

- Backend
  - AttemptController: `backend/src/modules/quizzes/controllers/AttemptController.ts` — GitHub: https://github.com/vicharanashala/vibe/blob/combined-updates/backend/src/modules/quizzes/controllers/AttemptController.ts
  - QuizController: `backend/src/modules/quizzes/controllers/QuizController.ts` — GitHub: https://github.com/vicharanashala/vibe/blob/combined-updates/backend/src/modules/quizzes/controllers/QuizController.ts
  - QuestionController: `backend/src/modules/quizzes/controllers/QuestionController.ts` — GitHub: https://github.com/vicharanashala/vibe/blob/combined-updates/backend/src/modules/quizzes/controllers/QuestionController.ts
  - QuestionBankController: `backend/src/modules/quizzes/controllers/QuestionBankController.ts` — GitHub: https://github.com/vicharanashala/vibe/blob/combined-updates/backend/src/modules/quizzes/controllers/QuestionBankController.ts
  - Validators: `backend/src/modules/quizzes/classes/validators/QuizValidator.ts` — GitHub: https://github.com/vicharanashala/vibe/blob/combined-updates/backend/src/modules/quizzes/classes/validators/QuizValidator.ts
  - Services: `backend/src/modules/quizzes/services/AttemptService.ts` — GitHub: https://github.com/vicharanashala/vibe/blob/combined-updates/backend/src/modules/quizzes/services/AttemptService.ts

- Frontend
  - API hooks: `frontend/src/hooks/hooks.ts` — GitHub: https://github.com/vicharanashala/vibe/blob/combined-updates/frontend/src/hooks/hooks.ts
  - Quiz player integration: `frontend/src/components/Item-container.tsx` — GitHub: https://github.com/vicharanashala/vibe/blob/combined-updates/frontend/src/components/Item-container.tsx
  - Teacher editor: `frontend/src/app/pages/teacher/components/enhanced-quiz-editor.tsx` — GitHub: https://github.com/vicharanashala/vibe/blob/combined-updates/frontend/src/app/pages/teacher/components/enhanced-quiz-editor.tsx

### Document structure — how this file is organized

- **Overview & lifecycle** (top): Short summary of quiz lifecycle stages from authoring to review.
- **Authoring & banks**: APIs and flow for creating questions, banks, and associating them with quizzes.
- **Attempt delivery**: Endpoints and flow students use to start, save, and submit attempts (the core runtime flow).
- **Grading & moderation**: Automatic grading, instructor overrides, feedback, and flagging.
- **Analytics & exports**: Endpoints for results, analytics, performance metrics, and CSV export.
- **Integration & notes**: Where frontend and backend integrate (hooks, components, services) and implementation caveats.

Use these sections to quickly find the APIs you need; follow the repo links above to jump directly to implementation files for code-level details.

---

## Stage 1 — Authoring questions & question banks ✍️

### POST /quizzes/questions

- Purpose: Create a new question. Body schema: `QuestionBody` (see `QuestionValidator.ts`).
- Auth: @Authorized()
- Request body (example):
  {
  "title": "What is 2+2?",
  "type": "SELECT_ONE_IN_LOT",
  "lotItems": [{"text":"3"}, {"text":"4", "isCorrect": true}],
  "maxScore": 2,
  "tags": ["math", "arithmetic"]
  }
- Response: { questionId }
- Frontend: Question editor (teacher), `hooks.ts` => `api.useMutation("post", "/quizzes/questions")` (see `enhanced-quiz-editor.tsx`).
- Notes: Questions pass through a `QuestionProcessor` (render + validation) before saving.

### GET /quizzes/questions/:questionId

- Purpose: Retrieve a question details (for editing/viewing).
- Auth: @Authorized()
- Response: `QuestionResponse` (full question with rendered view and metadata).
- Frontend: editors and preview UIs use this to load question data.

### PUT /quizzes/questions/:questionId

- Purpose: Update an existing question.
- Auth: @Authorized() (ability checks ensure creator or allowed role).
- Request body: same as create (`QuestionBody`).
- Response: updated `QuestionResponse`.

### DELETE /quizzes/questions/:questionId

- Purpose: Remove question permanently.
- Auth: @Authorized()
- Response: 204 No Content (OnUndefined).

### POST /quizzes/questions/:questionId/flag

- Purpose: Flag a question (student/instructor) with reason and context for review.
- Auth: @Authorized()
- Request body: `FlagQuestionBody` { reason, courseId, versionId }
- Response: 200 No Content
- Notes: Flags are reviewed by staff via Reports endpoints.

### POST /quizzes/questions/flags/:flagId/resolve

- Purpose: Resolve a flagged question with a status (e.g., RESOLVED or REJECTED).
- Auth: @Authorized()
- Request body: `ResolveFlagBody` { status }

### POST /quizzes/questions/generate-csv-res

- Purpose: AI-assisted generation of questions from transcripts (Claude/LLM integration).
- Auth: (currently may be public or guarded — check controller comments)
- Request body: `GenerateAIQuestionsBody` including `text` with timestamped transcript segments.
- Notes: The endpoint validates transcript timestamps and returns generated question objects.

---

## Stage 2 — Question banks & quiz composition 🧺

Question banks group curated questions that can be attached to quizzes.

### POST /quizzes/question-bank

- Purpose: Create a new question bank
- Auth: @Authorized()
- Request body (example, `CreateQuestionBankBody`):
  {
  "title": "Algebra Basics",
  "description": "Questions on linear algebra",
  "courseId": "<courseId>",
  "courseVersionId": "<versionId>",
  "questions": ["<questionId>"]
  }
- Response: { questionBankId }
- Frontend: question bank management UI, `hooks.ts` => `api.useMutation("post", "/quizzes/question-bank")`.

### GET /quizzes/question-bank/:questionBankId

- Purpose: Get a bank and its questions.
- Auth: @Authorized()
- Response: `QuestionBankResponse` (includes questions array)

### PATCH /quizzes/question-bank/:questionBankId/questions/:questionId/add

- Purpose: Add a question to a bank.
- Auth: @Authorized()
- Response: updated `QuestionBankResponse`

### PATCH /quizzes/question-bank/:questionBankId/questions/:questionId/remove

- Purpose: Remove question from bank.

### PATCH /quizzes/question-bank/:questionBankId/questions/:questionId/replace-duplicate

- Purpose: Duplicate the question and replace it in the bank (used to generate a fresh copy for editing without changing the original reference).
- Response: { newQuestionId }
- Frontend: `enhanced-quiz-editor.tsx` uses `api.useMutation('patch','/quizzes/question-bank/.../replace-duplicate')` to duplicate a question into the bank.

---

## Stage 3 — Associate banks with a quiz (configure behavior) ⚙️

A quiz (item) can have multiple banks associated with counts and configuration.

### POST /quizzes/quiz/:quizId/bank

- Purpose: Associate a bank with a quiz and supply bank-specific config (counts, randomization). Body: `AddQuestionBankBody`.
- Auth: @Authorized() with Quiz-level ability checks.
- Response: 200 No Content
- Frontend: `hooks.ts` => `api.useMutation("post","/quizzes/quiz/{quizId}/bank")` (teacher UI to configure quiz banks).

### DELETE /quizzes/quiz/:quizId/bank/:questionBankId

- Purpose: Remove bank association from quiz

### PATCH /quizzes/quiz/:quizId/bank

- Purpose: Edit bank configuration for a quiz (e.g., counts, parameter map). Request body: `EditQuestionBankBody`.

### GET /quizzes/quiz/:quizId/bank

- Purpose: List banks currently associated with the quiz (for display & editing)
- Response: array of `QuestionBankRef` objects (bankId + count)

---

## Stage 4 — Student attempt lifecycle (Start → Save → Submit) 🧑‍🎓

These endpoints are core to runtime quiz delivery.

### POST /quizzes/:quizId/attempt — Start attempt

- Purpose: Create and return a new attempt with rendered questions and parameter maps.
- Validator(s): `CreateAttemptParams` (path param) and response `CreateAttemptResponse` (`attemptId`, `questionRenderViews`).
- Auth: @Authorized()
- Frontend: invoked by the quiz player; `hooks.ts` maps `api.useMutation('post','/quizzes/{quizId}/attempt')`.

Sample response (expanded):

```json
{
  "attemptId": "60d21b4667d0d8992e610c99",
  "userAttempts": 1,
  "questionRenderViews": [
    {
      "_id": "60d21b4667d0d8992e610c02",
      "text": "What is 2 + {{x}}?",
      "type": "NUMERIC_ANSWER_TYPE",
      "isParameterized": true,
      "parameterMap": { "x": 2 },
      "parameters": [
        { "name": "x", "possibleValues": ["1", "2", "3"], "type": "number" }
      ],
      "hint": "Remember to add the numbers",
      "timeLimitSeconds": 30,
      "points": 5,
      "priority": "MEDIUM"
    },
    {
      "_id": "60d21b4667d0d8992e610c03",
      "text": "Select the correct option:",
      "type": "SELECT_ONE_IN_LOT",
      "isParameterized": false,
      "parameterMap": null,
      "lot": [
        { "_id": "lot1", "text": "3" },
        { "_id": "lot2", "text": "4" }
      ],
      "timeLimitSeconds": 60,
      "points": 2,
      "priority": "LOW"
    }
  ]
}
```

Fields explained:

- `attemptId` (string): ID of the created attempt (server-generated).
- `userAttempts` (number): The total number of attempts the user has made for this quiz (helps the UI show remaining attempts / attempt count).
- `questionRenderViews` (array): The rendered question views for this attempt. Each element is a display-ready, parameterized question instance containing the following useful fields:
  - `_id` (string | ObjectId): Question id.
  - `text` (string): The question text after parameter tags are left (if parameterized the text may include tokens like `{{x}}` — the frontend renders these using `parameterMap`).
  - `type` (QuestionType): One of `SELECT_ONE_IN_LOT`, `SELECT_MANY_IN_LOT`, `ORDER_THE_LOTS`, `NUMERIC_ANSWER_TYPE`, or `DESCRIPTIVE`.
  - `isParameterized` (boolean): Whether this question uses parameters.
  - `parameterMap` (object | null): A map of parameter names to generated values (ParameterMap = Record<string, string | number>). Example: `{ "x": 2 }`.
  - `parameters` (array, optional): When `isParameterized` is true, an array of `IQuestionParameter` describing available parameters: `{ name: string, possibleValues: string[], type: 'number' | 'string' }`.
  - `hint` (string, optional): Short hint text shown in the UI.
  - `timeLimitSeconds` (number): Time limit for the question in seconds.
  - `points` (number): Maximum points for this question.
  - `priority` ("LOW" | "MEDIUM" | "HIGH"): Priority value for ordering or presentation.
  - Type-specific fields (present only for some question types):
    - `lot` (array of lot items) — for `SELECT_ONE_IN_LOT` / `SELECT_MANY_IN_LOT` / `ORDER_THE_LOTS` views. `ILotItem` shape: `{ _id?: string, text: string, explaination?: string }`.

Notes & security considerations:

- The attempt's `questionRenderViews` is a *rendered* view intended for the client. Sensitive solution data (correct answers / scoring logic internals) are not returned in these views — grading is performed server-side using the `parameterMap` and question definitions.
- `parameterMap` values are deterministic for the attempt (used both to render the question and later to grade using the same parameter values).
- The backend `AttemptService` returns `{ attemptId, questionRenderViews, userAttempts }` — the frontend should store `attemptId` so subsequent `save` / `submit` calls reference the same attempt.

Type references (for implementers):

- `ParameterMap` = `Record<string, string | number>` (see `backend/src/modules/quizzes/question-processing/tag-parser/tags/Tag.ts`).
- `IQuestionParameter` = `{ name: string; possibleValues: string[]; type: 'number' | 'string' }` (see `backend/src/shared/interfaces/quiz.ts`).
- `QuestionType` values: `SELECT_ONE_IN_LOT`, `SELECT_MANY_IN_LOT`, `ORDER_THE_LOTS`, `NUMERIC_ANSWER_TYPE`, `DESCRIPTIVE`.
- `Priority` values: `LOW`, `MEDIUM`, `HIGH`.

If you'd like, I can also add a compact JSON schema or Swagger example for `CreateAttemptResponse` to the API validator (`QuizValidator.ts`) so the generated OpenAPI doc includes the full example. Would you like that? 🔧

Notes: The attempt has a server-side timer/limits (configured in quiz object); ability checks ensure student can start.

### POST /quizzes/:quizId/attempt/:attemptId/save — Save answers (autosave)

- Purpose: Save in-progress answers (does not finalize attempt). Body: raw JSON with `answers` array and optional `isSkipped` boolean.
- Auth: @Authorized()
- Implementation note: The controller reads the raw request body stream and parses JSON (`QuestionAnswersBodydto`) — supports larger payloads.
- Response: grading result for the saved batch (e.g., { result: 'PARTIALLY_CORRECT', explanation? }) — this can be used for immediate feedback.
- Frontend: quiz player autosave logic calls this periodically and on navigation (see `Quiz` component & `Item-container`).

### POST /quizzes/:quizId/attempt/:attemptId/submit — Submit attempt

- Purpose: Finalize attempt, grade automatically, and return grading results.
- Auth: @Authorized()
- Body: raw JSON similar to save: { answers: [...], isSkipped?: boolean }
- Response: `SubmitAttemptResponse` containing grading results (total score, per-question feedback, partial results). See `QuizValidator.ts` for the response schema.
- Notes: After submit, attempt state is final; additional instructor regrade/override is possible.

### GET /quizzes/:quizId/attempt/:attemptId — Get attempt details

- Purpose: Fetch attempt object (question details and saved answers) for the user or instructor.
- Auth: @Authorized() (ability checks: view) — used when resuming or reviewing attempts.

### GET /quizzes/:quizId/attempts/export

- Purpose: Export all attempts for a quiz as CSV (AttemptController exports a CSV stream). Useful for manual auditing.
- Auth: typically allowed for instructor roles.
- Frontend: called via `fetch(`${VITE_BASE_URL}/quizzes/${quizId}/attempts/export`)` (see `frontend/src/hooks/hooks.ts`).

---

## Stage 5 — Grading, feedback & moderation 🧾

### Auto-grading

- The server grades certain question types automatically (SOL, SML, OTL, NAT) using `AttemptService` and `QuestionProcessor` (parametric rendering ensures randomized parameters are graded correctly).

### POST /quizzes/quiz/:quizId/submission/:submissionId/score/:score — Override score

- Purpose: Instructor manually overrides a submission score.
- Auth: @Authorized() (ability check: ModifySubmissions)
- Frontend: instructor grading UI (submissions list) uses `hooks.ts` mutation `post /quizzes/quiz/{quizId}/submission/{submissionId}/score/{score}`.

### POST /quizzes/quiz/:quizId/submission/:submissionId/regrade — Regrade submission

- Purpose: Provide new grading results to regrade a submission (body: `RegradeSubmissionBody` with per-question regrade payload).
- Auth: @Authorized()
- Response: 200 No Content

### POST /quizzes/quiz/:quizId/submission/:submissionId/question/:questionId/feedback — Add feedback to specific question

- Purpose: Instructor adds feedback text for a question in a submission.
- Auth: @Authorized()
- Body: { feedback: string }

### POST /quizzes/:quizId/user/:userId/reset-attempts — Reset attempts count for a user

- Purpose: Reset allowed attempts for the user (e.g., if admin grants another try).
- Auth: @Authorized()
- Frontend: management UI uses `api.useMutation('post','/quizzes/quiz/{quizId}/user/{userId}/reset-attempts')`.

---

## Stage 6 — Results, analytics & performance 📊

These endpoints allow teachers and admins to analyze quiz performance.

### GET /quizzes/quiz/:quizId/details — Quiz details

- Purpose: Retrieve metadata about the quiz (configuration, banks, counts).
- Auth: @Authorized()
- Response: `QuizDetailsResponse` (see validators)

### GET /quizzes/quiz/:quizId/analytics — Quiz analytics

- Purpose: Retrieve aggregated analytics for the quiz (e.g., scores distribution, question-level stats).
- Auth: @Authorized() (ability: GetStats)
- Response: `QuizAnalyticsResponse`

### GET /quizzes/quiz/:quizId/performance — Per-question performance

- Purpose: Detailed metrics for each question (difficulty, p-value, average score)
- Auth: @Authorized() (ability: GetStats)
- Response: array of `QuizPerformanceResponse`

### GET /quizzes/quiz/:quizId/results — Results for all students

- Purpose: Fetch summarized results for the quiz (useful for leaderboards or grade export)
- Auth: @Authorized()
- Response: `QuizResultsResponse[]`

### GET /quizzes/quiz/:quizId/submissions & GET /quizzes/quiz/:quizId/submissions/:submissionId

- Purpose: Paginated list of submissions (teacher view) & details for a single submission.
- Auth: @Authorized()
- Frontend: teacher's submissions table uses `hooks.ts` `api.useQuery('get','/quizzes/quiz/{quizId}/submissions')` and details route.

---

## Stage 7 — Flags & moderation workflow 🚩

- Flagging a question: `POST /quizzes/questions/:questionId/flag` (student or instructor calls). Stores a flag with reason and context.
- Resolving a flag: `POST /quizzes/questions/flags/:flagId/resolve` — used by admins/instructors to mark flags resolved.
- Reports & audit: flagged questions appear in report endpoints (see `ReportController`), where moderators review and add actions.

---

## Integration & frontend references

- Main typed hooks for quiz APIs: `frontend/src/hooks/hooks.ts` contains the `api.useMutation` and `api.useQuery` bindings for nearly all endpoints discussed (search for `/quizzes` inside that file). Examples:
  - Start attempt: `api.useMutation("post", "/quizzes/{quizId}/attempt")`
  - Save attempt: `api.useMutation("post", "/quizzes/{quizId}/attempt/{attemptId}/save")`
  - Submit: `api.useMutation("post", "/quizzes/{quizId}/attempt/{attemptId}/submit")`
  - Question CRUD: `api.useMutation('post', '/quizzes/questions')`, etc.
- Quiz player & item container: `frontend/src/components/Item-container.tsx` and quiz components interact directly with attempts and call save/submit APIs.
- Teacher UIs: `frontend/src/app/pages/teacher/components/enhanced-quiz-editor.tsx`, `quiz-modal.tsx`, `AnomaliesList.tsx` and other admin pages call bank and submission endpoints.

---

## API-specific implementation notes & gotchas ⚠️

- The Save/Submit endpoints parse body from the raw request stream (they read request data manually) — this supports large answer payloads and avoids the usual `@Body()` decorator.
- Many sensitive endpoints use CASL `@Ability(...)` decorators and `@Authorized()` — ensure token refresh and authorization are correct in client flows.
- Randomization: question banks and question rendering use randomized selection and parameter maps to create unique attempts per student (see `QuestionBankService` and `QuestionProcessor`). Tests should verify randomness and reproducibility for grading.
- Exports: CSV export from attempts uses streaming and sets CSV headers; teacher clients directly fetch and download.

---

## Suggested tests / QA checklist ✅

- Start → Save → Submit flow: verify saved progress is restored and submit returns expected grading for parametric questions.
- Randomization: two attempts from same quiz may have differing parameter maps; ensure grading matches the parameterization used for that attempt.
- Submission edits: instructor regrade & score override should reflect in reports and student result views.
- Flags lifecycle: flagging a question and resolving should be recorded and surfaced in reports.
- Question bank operations: add/remove/replace-duplicate should maintain question references and not corrupt banks.

---

## MongoDB document structure — quiz subsystem 🗄️

**Authoritative references:** `AttemptRepository`, `SubmissionRepository`, `QuestionBankRepository`, `QuizRepository`, `UserQuizMetricsRepository`, `backend/src/modules/quizzes/interfaces/grading.ts`, `QuestionValidator.ts`, `QuizValidator.ts`.

Below are the main collections used by the quiz subsystem, the important fields, example document snippets, and notable indexes.

### `quizzes` — Quiz items (QuizItem)

- **Purpose:** Stores quiz item metadata & settings (attached to course items).
- **Key fields:** `_id`, `name`, `description`, `type`, `details` (IQuizDetails — `questionBankRefs`, `passThreshold`, `maxAttempts`, `quizType`, `releaseTime`, `questionVisibility`, `deadline?`, `approximateTimeToComplete`, `allowPartialGrading`, `allowHint`, `showCorrectAnswersAfterSubmission`, `showExplanationAfterSubmission`, `showScoreAfterSubmission`, `allowSkip`), `isOptional`, `isHidden`, `isDeleted`, `deletedAt`.
- **Notes:** `details.questionBankRefs` contains objects `{ bankId, count, difficulty?, tags?, type? }` (see `IQuestionBankRef` in `shared/interfaces/models.ts`).

Example (abridged):

```
{ _id: "60d...85", name: "Algebra Quiz", type: "QUIZ", details: { questionBankRefs: [{ bankId: "60d...9a", count: 5 }], passThreshold: 0.7, maxAttempts: 3, quizType: "DEADLINE", releaseTime: "2024-06-18T12:00:00Z" }, isDeleted: false }
```

---

### `questionBanks` — Banks of question IDs

- **Purpose:** Group of questions used to compose quizzes.
- **Key fields:** `_id`, `title`, `description`, `courseId`, `courseVersionId`, `questions` (array of question IDs/ObjectIds), `tags`, `createdAt`, `updatedAt`, `isDeleted`, `deletedAt`.
- **Indexes:** `questions_1`, `courseVersionId_1` (created in `QuestionBankRepository.ts`).
- **Notes:** `getById` filters out banks with `isDeleted: true` and filters out soft-deleted questions when returning results.

Example:

```
{ _id: "60d...b2", title: "Algebra Basics", courseVersionId: "60d...c1", questions: ["60d...87","60d...88"], createdAt: "2024-06-01T..." }
```

---

### `questions` — Question documents

- **Purpose:** Actual question definitions and solutions used by banks and attempts.
- **Key fields:** `_id`, `text`, `type`, `isParameterized`, `parameters?`, `hint?`, `timeLimitSeconds`, `points`, `priority`, `solution` fields (type-specific: `correctLotItem`, `incorrectLotItems`, `correctLotItems`, `ordering`, `decimalPrecision`, `upperLimit`, `lowerLimit`, `value`, `expression`, `solutionText`), counters: `skipCount`, `attemptCount`, `attemptedByUsersCount`, and soft-delete metadata `isDeleted`, `deletedAt`.
- **Validators & shapes:** `QuestionBody` / `QuestionResponse` (see `QuestionValidator.ts`).

Example:

```
{ _id: "60d...87", text: "What is 2+2?", type: "NUMERIC_ANSWER_TYPE", points: 5, timeLimitSeconds: 30, solution: { value: 4, decimalPrecision: 0, lowerLimit: 0, upperLimit: 10 } }
```

---

### `quiz_attempts` — In-progress and completed attempts

- **Purpose:** Stores every attempt created when a student starts a quiz (answers can be saved multiple times before submit).
- **Key fields:** `_id`, `quizId`, `userId`, `questionDetails` (array of `{ questionId, parameterMap? }`), `answers` (array of `IQuestionAnswer` — `{ questionId, questionType, answer }`), `isSkipped?`, `createdAt`, `updatedAt`.
- **Indexes:** `quizId_1_userId_1` and `questionDetails_questionId_1` (created in `AttemptRepository.ts`).
- **Notes:** `getAttemptsByQuizId` pipeline joins `questions` and `users` to produce export-friendly views.

Example:

```
{ _id: "60d...90", quizId: "60d...85", userId: "60d...aa", questionDetails: [{ questionId: "60d...87", parameterMap: { x: 2 } }], answers: [{ questionId: "60d...87", questionType: "NUMERIC_ANSWER_TYPE", answer: { value: 4 } }], createdAt: "2024-06-18T..." }
```

---

### `quiz_submission_results` — Final submissions with grading

- **Purpose:** Finalized submission records which include grading results and submission timestamps.
- **Key fields:** `_id`, `quizId`, `userId`, `attemptId`, `submittedAt`, `gradingResult` (`totalScore`, `totalMaxScore`, `overallFeedback`, `gradingStatus`, `gradedAt`, `gradedBy`).
- **Indexes:** `quizId_1_userId_1_attemptId_1`, `quizId_1_gradingStatus_1_submittedAt_-1` (created in `SubmissionRepository.ts`).
- **Notes:** Used to compute analytics (average score, pass rates) and page teacher-facing submissions with text search on user names/emails in aggregation pipeline.

Example:

```
{ _id: "60d...f0", quizId: "60d...85", userId: "60d...aa", attemptId: "60d...90", submittedAt: "2024-06-18T12:45:00Z", gradingResult: { totalScore: 8.5, totalMaxScore: 10, gradingStatus: "PASSED" } }
```

---

### `user_quiz_metrics` — Per-user, per-quiz aggregated metrics

- **Purpose:** Tracks user progress for a quiz (remaining attempts, latest attempt/submission references, skip counts, attempts history).
- **Key fields:** `_id`, `quizId`, `userId`, `latestAttemptStatus` (`ATTEMPTED|SUBMITTED|SKIPPED`), `latestAttemptId`, `latestSubmissionResultId`, `remainingAttempts`, `skipCount`, `attempts` (array of `{ attemptId, submissionResultId? }`).
- **Repo helpers:** `findWithMissingSubmissionIds`, `bulkUpdateMetrics` (see `UserQuizMetricsRepository.ts`).

Example:

```
{ _id: "60d...aa", quizId: "60d...85", userId: "60d...aa", latestAttemptStatus: "SUBMITTED", remainingAttempts: 0, attempts: [{ attemptId: "60d...90", submissionResultId: "60d...f0" }] }
```

---

### Important operational notes ⚠️

- IDs may be stored as either **ObjectId** or **string** in some places; repositories normalize queries to accept both (see `AttemptRepository.getById`, `SubmissionRepository.get`, `UserQuizMetricsRepository.get`).
- Soft deletes: question banks and questions are soft-deleted (`isDeleted` + `deletedAt`) and queries typically filter these out.
- Indexes in repositories are optimized for teacher-facing queries (quiz+user lookups, grading status, per-question analytics).

> Tip: When adding new queries or reports, prefer using repository helpers (they already implement ObjectId/string normalization and aggregation pipelines) to avoid subtle bugs.

---

If you'd like, I can:

- Export this doc to a CSV mapping endpoint → method → validator class → frontend hook location, or
- Scaffold an OpenAPI snippet with full request/response examples for each endpoint to paste into the docs site (Swagger/OpenAPI).

Which would you prefer next? 🔧
