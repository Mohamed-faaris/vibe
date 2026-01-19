# Course Integrity & Safety — Anti-Cheat Features and Recommendations ✅🔒

**Purpose:** A concise reference of existing anti-cheat/safety features in the codebase and recommended controls to maintain academic integrity (detect and deter cheating). Use this as a checklist for product, engineering, and QA work.

---

## 1) Existing Safety Features (what's implemented) 🧭

- **Configurable Proctoring per-course**
  - Backend: `CourseSettingController` (PUT `/setting/course-setting/:courseId/:versionId/proctoring`) — enables detectors and _linearProgressionEnabled_ to enforce sequential progress. (Files: `backend/src/modules/setting/controllers/CourseSettingController.ts`, `CourseSettingValidators.ts`)
  - Purpose: Turn detectors on/off per course/version and require linear progression.

- **Anomaly Recording & Detectors**
  - Endpoints: `POST /anomalies/record/image`, `POST /anomalies/record/audio` (AnomalyController + AnomalyService).
  - Detectors tracked (stats & types): VOICE_DETECTION, NO_FACE, MULTIPLE_FACES, BLUR_DETECTION, HAND_GESTURE_DETECTION, FACE_RECOGNITION, FOCUS, etc. (See `AnomalyService`, `AnomalyRepository`, and `CloudStorageService` for uploads and storage.)
  - Purpose: Capture evidence (images/audio) and structured anomaly types for review and statistics.

- **In-session Random Presence Verifications**
  - Frontend: `floating-video` component triggers randomized small challenges (e.g., thumbs-up) and uses gesture detection to confirm presence. (File: `frontend/src/components/floating-video.tsx`)
  - Purpose: Prevent students from leaving webcam off or outsourcing session to others.

- **Attempt & Quiz Protections**
  - Randomization: Quizzes support randomizing questions and parameterized questions (question bank shuffling and `generateRandomParameterMap`). (Files: `QuestionBankService.ts`, `QuestionProcessor.ts`)
  - Attempt lifecycle endpoints: `POST /quizzes/{quizId}/attempt`, `POST /quizzes/{quizId}/attempt/{attemptId}/save`, `POST /quizzes/{quizId}/attempt/{attemptId}/submit`. (See `AttemptController` and `QuizController`.)
  - Purpose: Random question selection and parameterization makes answer-sharing harder; autosave reduces incentive to cheat by page reload.

- **Watchtime & Progress Tracking**
  - Endpoints: `POST /progress/.../start`, `POST /progress/.../stop`, `GET /users/watchTime/total`, and bulk updates `PATCH /enrollments/bulk-update-watchtime-progress-completeCounts`.
  - Purpose: Detect abnormal watchtime, low engagement, or suspicious spikes of activity.

- **Flagging / Reporting & Moderation Workflow**
  - Endpoints & models: Reports/flags (`POST /reports`, `GET /reports/:courseId/:versionId`, etc.), and `ReportValidators.ts` details report payloads.
  - Purpose: Students/instructors can flag suspicious content, submissions, or behavior for human review.

- **Session/Submission Exports & Auditing**
  - Export endpoints: e.g., `/quizzes/{quizId}/attempts/export` (used to audit and download attempts for investigation).
  - Purpose: Provide evidence bundles (answers, timestamps) for review/audit.

- **Role-based checks & ability enforcement**
  - CASL-based checks and `@Authorized()` on controllers ensure only properly authorized instructors/admins can perform sensitive actions (create jobs, edit content, view flagged items). (See various `@Ability` usages in controllers.)

---

## 2) How these pieces work together (workflow) 🔗

1. Instructor enables proctoring + chosen detectors for the course version.
2. During an exam/attempt, the client may record anomalies (images/audio) and send them to `POST /anomalies/...` with metadata (courseId, versionId, itemId, userId).
3. Server stores files in secure cloud storage and metadata in DB; anomaly stats are produced (`AnomalyService.getAnomalyStats`).
4. If thresholds are exceeded (e.g., multiple `MULTIPLE_FACES` events), an alert/flag should be raised for instructor review (system supports flags and reports which instructors can fetch).
5. Instructors use exports (attempt data and anomalies) and reports to make decisions.

---

## 3) Gaps and Recommended Improvements (practical, prioritized) ⚠️

1. **Automated Alerting & Threshold Rules (High priority) ✅**
   - Implement configurable alert rules (e.g., >3 VOICE_DETECTION OR NO_FACE events during an exam → auto-flag attempt). Provide endpoint(s): `POST /proctoring/alerts` and `GET /proctoring/alerts`.
   - Link alerts to attempts and include anomaly evidence references.

2. **Plagiarism & Answer-Similarity Detection (High)**
   - Add server-side similarity/plagiarism checks for textual answers, code submissions, and open-response questions. Integrate with existing exports and reports (or external services like Turnitin, Moss, or open-source semantic checks).

3. **Session Integrity & Tamper-proof Logs (Medium)**
   - Maintain append-only, timestamped logs for critical events (attempt start/stop, anomalies posted, token refresh). Consider signing records with server keys.

4. **Stronger Identity Verification (Medium)**
   - Improve face-recognition flows: match live face snapshots with user profile photo (or require an initial verified selfie). Record similarity score.
   - Add two-factor/time-synced signed exam tokens with limited TTL.

5. **Instructor Review UI + Evidence Bundles (Medium)**
   - Build a Proctoring Review dashboard (list alerts, show timeline of anomalies and linked attempts, export evidence zip). Provide endpoints for bulk review/resolution.

6. **Privacy & Retention Policies (High; compliance)**
   - Add retention policies and consent flows for audio/video capture (configurable per course and per jurisdiction). Provide endpoints to delete stored anomaly data per policy.

7. **Browser/Network Forensics (Medium)**
   - Capture IP, user-agent, and optionally browser fingerprint at attempt start and for each submission. Provide audit endpoints for investigation.

8. **Robustness to Spoofing / Anti-Tampering (Medium)**
   - Detect fake webcams or replay attacks (e.g., liveness detection in face recognition, small randomized challenges like the thumbs-up gesture already present). Record challenge response times.

9. **E2E Tests & QA Scenarios (High)**
   - Add automated tests: anomaly ingestion, alert generation, report creation, and instructor review workflows. Add stress tests for false positive/negative rates of detectors.

---

## 4) Suggested API & Data model additions (quick proposals)

- POST /proctoring/alerts
  - Body: { courseId, versionId, attemptId?, userId?, detector: 'NO_FACE'|'VOICE_DETECTION', count, evidenceIds: [] }
  - Purpose: Persist alerts and feed instructor dashboards; also used to trigger email/push notifications.

- GET /proctoring/alerts?courseId=...&status=PENDING
  - Purpose: Instructor UI to fetch alerts and triage.

- POST /proctoring/review/:alertId/action
  - Body: { action: 'REJECT'|'CONFIRM'|'ESCALATE', comment }
  - Purpose: Capture instructor decisions and comments.

- POST /plagiarism/check
  - Body: {courseId, versionId, submissionId, content}
  - Purpose: Return similarity score & related matches.

- GET /anomalies/:userId/timeline?attemptId=...
  - Purpose: Show a time series of anomalies for visual inspection in the UI.

---

## 5) Tests & Monitoring to add (practical list) 🧪

- Unit tests for anomaly classification and storage.
- Integration tests: anomaly upload → cloud storage → DB record → alert rule trigger.
- E2E: simulated exam where random challenges are issued; verify responses, timestamps, and alert generation.
- Observability: track false positive rates and anomaly counts per course; create dashboards/metrics (Prometheus/Grafana).

---

## 6) Privacy, Legal & UX notes ⚖️

- Add explicit _consent_ banners before recording video/audio for proctoring; store consent metadata.
- Implement retention & deletion policies for stored images/audio to comply with GDPR and other laws.
- Provide students with clear guidance about what is recorded and how it is used.

---

## 7) Helpful repo pointers (where to start looking) 🔎

- Proctoring settings: `backend/src/modules/setting/controllers/CourseSettingController.ts`
- Anomalies: `backend/src/modules/anomalies/controllers/AnomalyController.ts`, `AnomalyService.ts`, `CloudStorageService.ts`
- Progress & watchtime: `backend/src/modules/users/controllers/ProgressController.ts`
- Reports & flags: `backend/src/modules/reports/controllers/ReportController.ts`, `ReportValidators.ts`
- Quiz processing & randomization: `backend/src/modules/quizzes/services/QuestionBankService.ts`, `QuestionProcessor.ts` and `AttemptController.ts`
- Frontend proctoring UI: `frontend/src/components/EditProctoringModal.tsx`, `frontend/src/components/floating-video.tsx`, teacher review pages like `AnomaliesList.tsx`.

---

## 8) Next steps (recommended minimal scope to ship quickly) 🚀

1. Implement alerting rules + `/proctoring/alerts` endpoints and a basic instructor triage UI. (High impact)
2. Add retention/consent UI and server-side retention policy enforcement. (Compliance)
3. Integrate a plagiarism/similarity check for text answers (start with an open-source library or a lightweight vector similarity check). (High value)
4. Add automated tests and observability for detection rates.

---

If you'd like, I can:

- scaffold the instructor alert endpoints and a minimal review UI, or
- generate a CSV checklist that maps detectors → endpoints → test cases for QA.

Which of those would help you move fastest? 💡
