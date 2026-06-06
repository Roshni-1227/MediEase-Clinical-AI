# MediEase Clinical System (Clinical Handwriting-to-Insight)

> **Purpose:** Analyze uploaded or captured prescription images (and optionally pasted text) to extract medicines, evaluate clinical risk based on symptom keywords + dosage heuristics, store analysis history, and generate a downloadable PDF report.

---

## 1. High-level architecture

MediEase is a **Spring Boot** web application using **Thymeleaf** for server-rendered pages and **Spring Security** for role-based access.

### Core modules (by responsibility)

1. **Web/UI Layer (Thymeleaf + JavaScript)**
   - Upload/Camera/Text UI under `src/main/resources/templates/`.
   - Client-side JS sends requests to application endpoints (mostly JSON via `/api/**`).

2. **Controller Layer**
   - `MainController` routes user interactions:
     - Page endpoints (role dashboards, history, profile)
     - API endpoints (OCR/analysis requests)
     - PDF download endpoints
     - Pharmacy dispensing action

3. **Analysis Pipeline (OCR → AI extraction → Normalization → Fuzzy risk evaluation)**
   - `AnalysisService` orchestrates the end-to-end workflow.
   - Supporting services:
     - `ImagePreprocessingService` (image enhancement pipeline)
     - `OcrSpaceService` (cloud OCR engine; invoked concurrently)
     - `OcrService` (local Tesseract OCR; also invoked concurrently)
     - `AiOrchestratorService` → `GroqTextService` (LLM-driven structured extraction)
     - `MedicineNormalizationService` (synonym alias → canonical mapping)
     - `FuzzyStringMatcher` (string similarity for medicine matching)
     - `FuzzyRiskEvaluator` (symptom severity + final decision rules)

4. **Persistence Layer (JPA)**
   - Entities such as `AnalysisHistory`, `Medicine`, `User`, etc.
   - Repositories expose query methods used by dashboards and APIs.

5. **Reporting Layer (PDF)**
   - `PdfService` renders an `AnalysisResult` into a PDF using **iText**.

6. **Bootstrapping & Configuration**
   - `DataInitializer` seeds initial medicines and default users.
   - `application.properties` defines OCR/AI endpoints, timeouts, security-related behavior, and upload limits.

---

## 2. Tech stack

### Backend
- **Java 17**
- **Spring Boot** 3.2.x
- **Spring MVC** (controllers)
- **Spring Security** (form login, role-based authorization, remember-me)
- **Spring Data JPA** (persistence)
- **H2** (development/runtime) and **PostgreSQL** support in configuration
- **Thymeleaf** (HTML views)
- **tess4j** (Tesseract OCR integration)
- **OkHttp** (HTTP client for Groq API)
- **Jackson** (JSON parsing)
- **iText** (PDF generation)
- **Apache Commons Text** (Levenshtein + Jaro-Winkler similarity)

### Frontend
- Server-rendered HTML (Thymeleaf templates)
- Client-side JavaScript (upload/camera capture and API calls)
- Bootstrap icons/classes are referenced in templates (e.g., `bi bi-*`)

---

## 3. Application configuration (`application.properties`)

All runtime tuning for OCR/AI and storage behavior is controlled here.

### 3.1 Server / dynamic port
- `server.port=0`
  - Spring binds to a **random available port**.
- `MediEaseApplication` uses `local.server.port` to build the `http://localhost:<port>/login` URL.
- Browser auto-opening is guarded by:
  - `mediease.browser.open` (defaults to `false`)
  - headless environment detection via `GraphicsEnvironment.isHeadless()`

### 3.2 Database
- H2 file mode:
  - `spring.datasource.url=jdbc:h2:file:./data/mediease;LOCK_TIMEOUT=10000;DB_CLOSE_DELAY=-1`
- JPA behavior:
  - `spring.jpa.hibernate.ddl-auto=update`
  - `spring.jpa.open-in-view=false`
  - `spring.sql.init.mode=never`

### 3.3 Logging
- `logging.level.root=INFO`
- `logging.level.com.mediease=DEBUG`
- Hibernate logs are reduced:
  - `logging.level.org.hibernate=ERROR`

### 3.4 OCR (OCR.space + Tesseract fallback)
- OCR.space:
  - `ocr.space.api.key=...`
  - `ocr.space.base.url=https://api.ocr.space/parse/image`
- Tesseract:
  - `tesseract.data.path=C:/Program Files/Tesseract-OCR/tessdata`

### 3.5 AI (Groq)
- Groq endpoint and model:
  - `groq.base.url=https://api.groq.com/openai/v1/chat/completions`
  - `groq.text.model=llama-3.1-8b-instant`
- Timeouts:
  - `ai.request.timeout=30000`
  - `ai.connect.timeout=10000`

### 3.6 File upload constraints
- `spring.servlet.multipart.max-file-size=10MB`
- `spring.servlet.multipart.max-request-size=10MB`

---

## 4. Security model (Spring Security)

### 4.1 Authentication & session
- **Form login** at `/login`.
- CSRF is disabled (`csrf.disable()`), and frame options are disabled to allow `h2-console`.
- `rememberMe`:
  - Enabled with key: `mediease-secure-key`
  - Token validity: **7 days** (`86400 * 7`)
- Only **one session** per user is allowed:
  - `maximumSessions(1)`
  - `maxSessionsPreventsLogin(false)`

### 4.2 Authorization by URL pattern
Routes are protected using role checks (pattern-based):
- Public / permitted:
  - `/login`, static assets `/css/**`, `/js/**`, `/images/**`
  - `/h2-console/**`, `/error`
  - OCR/analysis endpoints like `/analyze`, `/test-ocr`, `/api/**`
- Role dashboards:
  - `/admin/**` requires `ROLE_ADMIN`
  - `/doctor/**` requires `ROLE_ADMIN` or `ROLE_DOCTOR`
  - `/pharmacist/**` requires `ROLE_ADMIN` or `ROLE_PHARMACIST`
  - `/analyst/**` requires `ROLE_ADMIN` or `ROLE_ANALYST`
  - `/patient/**` requires `ROLE_ADMIN` or `ROLE_PATIENT`

### 4.3 Login redirect
`authenticationSuccessHandler()` redirects based on the first granted authority:
- ADMIN → `/admin/dashboard`
- DOCTOR → `/doctor/dashboard`
- PHARMACIST → `/pharmacist/dashboard`
- ANALYST → `/analyst/dashboard`
- PATIENT → `/patient/dashboard`

### 4.4 Audit logging hooks
Logout triggers:
- `auditService.logActivity(authentication.getName(), "LOGOUT", ...)`

And page views in controllers (e.g., admin dashboard) also call `auditService.logActivity(...)`.

---

## 5. Default seed data (`DataInitializer`)

### 5.1 Medicines seeding
- `initMedicines(...)` inserts a set of predefined medicines if they do not exist already:
  - Uses `saveIfNotExists(repository, medicine)` with `existsByName`.
- Medicines include:
  - Analgesics, NSAIDs, antibiotics, vitamins, and many enriched entries.
- A simplified fallback insertion exists for a list of brand names via:
  - `createSimpleMedicine(name)`

### 5.2 Users seeding
Default accounts are created if missing:
- `admin` / `admin123` → `ROLE_ADMIN`
- `doctor` / `doctor123` → `ROLE_DOCTOR`
- `pharmacist` / `pharm123` → `ROLE_PHARMACIST`
- `analyst` / `analyst123` → `ROLE_ANALYST`
- `patient` / `patient123` → `ROLE_PATIENT`

---

## 6. Endpoints & workflow (controllers)

> The UI typically interacts with the application using JSON endpoints under `/api/**`.

### 6.1 UI pages (Thymeleaf views)
Key pages routed by `MainController`:
- `/login`
- `/dashboard` (role router)
- Role dashboards:
  - `/admin/dashboard`
  - `/doctor/dashboard`
  - `/pharmacist/dashboard`
  - `/analyst/dashboard`
  - `/patient/dashboard`
- Medicine browsing:
  - `/medicine-db`
  - `/medicine/details/{id}` (and `/medicine/{id}`)
- Upload & analysis:
  - `/upload`
  - `/results`
  - `/history`
- Patient profile:
  - `/patient/profile`
  - `/patient/profile/update`

### 6.2 JSON/API endpoints
#### Analyze image (primary)
- `POST /api/analyze`
- Request:
  - `multipart/form-data` field `prescription` (image)
- Response:
  - `AnalysisResult` as JSON

#### Analyze pasted text
- `POST /api/analyze-text`
- Request:
  - `prescriptionText` (string)
- Response:
  - JSON shaped as `AnalysisResult` built from `AiResponseDTO` output

#### Analyze text (server-side redirect)
- `POST /analyze-text`
- Behavior:
  - Validates input is non-empty
  - Uses `AiOrchestratorService.process(text)`
  - Redirects to `/results` with flash attributes

#### Debug endpoints
- `POST /api/test-analyze`
  - Returns a mocked `AnalysisResult`.
- `POST /api/test-ocr`
  - Runs the full `analysisService.analyzePrescription(file)` and returns it.

#### Medicine JSON fetch
- `GET /api/medicine/{id}` → medicine record as JSON.

#### PDF download
- `GET /download-report`
  - Accepts `@ModelAttribute("lastResult")`.
  - Returns generated PDF bytes (`application/pdf`).
- `GET /patient/download/{id}`
  - Enforces access rules:
    - Patient can download if the history belongs to them or if privileged roles (`ROLE_ADMIN`/`ROLE_DOCTOR`) are present.
  - Uses a simplified reconstruction into `AnalysisResult` and generates the PDF.

### 6.3 Pharmacy dispensing action
- `POST /pharmacist/dispense/{id}`
- Checks current `status` and updates to `DISPENSED`.
- Logs audit activity.

---

## 7. End-to-end analysis pipeline (deep workflow)

The heart of the system is implemented in `AnalysisService#analyzePrescription`.

### Step 1: Image preprocessing (accuracy-first)
`ImagePreprocessingService#preprocess(...)` transforms the incoming image to improve handwriting OCR quality.

Key steps used:
1. **Balanced resize** if dimensions exceed 2000px (scales down proportionally)
2. **Grayscale conversion**
3. **Contrast enhancement** by stretching gray value range
4. **Upscaling** (×2.5)
5. **Adaptive thresholding** tuned for handwriting:
   - Uses integral image for local mean computation
   - Window size: `window = 20`
   - Threshold preserves thin strokes using: `pixel > mean - 5`
   - Output is binary (`TYPE_BYTE_BINARY`), with ink vs background mapping
6. **Sharpening**:
   - Uses a local edge amplification model (`center*5 - neighbors`), clamped 0..255

### Step 2: Convert to bytes for OCR engines
The cleaned image is written into PNG bytes (`ByteArrayOutputStream`) to support parallel OCR flows.

### Step 3: Concurrent OCR execution
OCR is performed in parallel using:
- **Tesseract** (local via `tess4j`)
- **OCR.space** (cloud via `OcrSpaceService`)

Implementation uses `CompletableFuture`:
- One future runs `performOCR(cleanedImage)`.
- Another future runs OCR.space extraction using original uploaded file bytes.

### Step 4: OCR selection logic
- OCR.space text is preferred if it is non-empty.
- If OCR.space returns empty text, the pipeline falls back to Tesseract output.

If both are empty after trimming, the service throws:
- `OCR failed: No text detected by Tesseract or OCR.space`

### Step 5: AI-based structured extraction (Groq + strict JSON)
`AiOrchestratorService.process(finalExtractedText)`:
1. Calls `GroqTextService.extractAndCorrectMedicines(ocrText)`.
2. Parses Groq output into `AiResponseDTO` via Jackson.
3. Performs medicine normalization and fuzzy matching against the medicine database.

#### 5.1 Prompting behavior (very strict contract)
`GroqTextService#callGroq(...)` constructs a system prompt emphasizing:
- **Senior Clinical Handwriting Specialist** role.
- **Exhaustive extraction** (must not skip medicines).
- Strict JSON only:
  - Return ONLY the JSON object
  - No conversational text
- Handles common OCR corruptions (e.g., `Am0x1cillin` → `Amoxicillin`, `5OOmg` → `500mg`).
- Uses clinical plausibility when consolidating fragments.

The request sent to Groq includes:
- `temperature: 0.1`
- `max_tokens: 2048`

#### 5.2 JSON cleaning guard
`AiOrchestratorService#cleanJson(...)` removes common code fences if Groq wraps JSON in ``` or ```json.

### Step 6: Medicine normalization and fuzzy matching
`AiOrchestratorService#applyNormalizationAndFuzzy(...)`:
1. Loads medicine names from the DB and sorts them case-insensitively per request.
2. For each extracted medicine:
   - Normalizes with `MedicineNormalizationService.normalize(...)`:
     - lowercases
     - strips markers like `tab.`, `cap.`, and the word `tablet`
     - applies synonyms from `/medicinesynonyms.json`
   - Finds best match using `FuzzyStringMatcher`.
   - Replaces the extracted name with the canonical matched DB name.

#### Fuzzy matching scoring
`FuzzyStringMatcher#calculateSimilarity`:
- Computes:
  - Levenshtein similarity
  - Jaro-Winkler similarity
- Weighted combination:
  - `0.4 * lev + 0.6 * jw` → scaled to 0..100

`findBestMatch` enhancements:
- Cleans input by removing non-alphanumeric and formatting markers.
- Adds a substring boost (+15) when either string contains the other after cleaning.
- Final acceptance threshold:
  - returns the best match only when `bestScore >= 65`, otherwise returns original input.

### Step 7: Risk evaluation using symptom keyword fuzziness
`AnalysisService` uses `FuzzyRiskEvaluator`:

#### Symptom keyword analysis
`analyzeSymptomSeverity(symptoms)`:
- Builds a severity dictionary mapping keywords/phrases to integer weights.
- Scores total severity by counting matched keywords.
- Produces:
  - `level` (LOW/MEDIUM/HIGH)
  - `severity` (MILD/MODERATE/SEVERE)
  - `confidence`
  - `explanation`

Decision thresholds in code:
- If no symptoms provided:
  - defaults to LOW/MILD with confidence 100
- If `score >= 10`:
  - HIGH / SEVERE
- If `score >= 5`:
  - MEDIUM / MODERATE
- Else:
  - LOW / MILD

#### Final decision combination rules
`makeFinalDecision(matchScore, dosageRisk, symptomRisk)`:
- If `matchScore < 60` → `HIGH_RISK`
- If dosage risk or symptom risk is HIGH → `HIGH_RISK`
- If dosage risk is MEDIUM **or** symptom risk is MEDIUM **or** `matchScore < 85` → `CAUTION`
- Otherwise → `SAFE`

### Step 8: Clinical recommendations
`generateRecommendations(...)` adds recommendation lines based on the `Decision`:
- SAFE → “Safe prescription detected”
- CAUTION → “Review required…”
- HIGH_RISK → “CRITICAL… Pharmacist review mandatory.”

It also appends symptom context:
- `Symptom Context: <explanation>`

### Step 9: Persist analysis history (JPA)
`saveToHistory(...)`:
- Creates a new `AnalysisHistory` row with:
  - `analysisDate`
  - serialized medicines detected (`mapper.writeValueAsString(medicines)`)
  - `overallDecision`
  - `systemConfidence` (symptomAnalysis.confidence)
  - symptom severity & explanation
  - medicine count
  - processing time in ms
  - original uploaded filename (if available)
- Assigns ownership fields based on logged-in role:
  - If authenticated user is `ROLE_PATIENT`, it sets `patientId = username`
  - Otherwise it sets `analyzedBy = username`

---

## 8. JSON models used in workflow

### 8.1 `AiResponseDTO`
Represents Groq extraction output:
- `medicines[]` where each has:
  - `name`
  - `dosage`
- `symptoms`
- `confidence`

### 8.2 `AnalysisResult`
Represents what the UI and PDF generator consume:
- Medicines: `List<MedicineAnalysis>`
- Symptom fields:
  - `symptomSummary`
  - `symptomRisk`
  - `symptomSeverity`
  - `symptomConfidence`
  - `symptomExplanation`
- Decision & confidence:
  - `overallDecision`
  - `systemConfidence`
- Lists for downstream report sections:
  - `drugInteractions`
  - `duplicateTherapies`
  - `clinicalRecommendations`
- Report metadata:
  - `doctorName`, `hospitalName`, `patientName`, `patientAge`, `prescriptionDate`
- OCR text artifacts:
  - `rawOcrText`
  - `aiProcessedText`
  - `smartRecoveryApplied` flag

---

## 9. PDF report generation (`PdfService`)

`PdfService#generatePrescriptionReport(AnalysisResult result)`:
- Creates a PDF document in memory via `ByteArrayOutputStream`.
- Uses iText layout primitives:
  - Header and generated timestamp
  - Two-column information table (Patient vs Clinical)
  - Executive summary section:
    - Overall decision name
    - Color logic:
      - SAFE → green, else red
    - Clinical insight derived from `result.getSymptomExplanation()`
  - Medicines table with columns:
    - Medicine name
    - Extracted dosage
    - Risk level (`m.getDosageRisk().name()`)
    - Extraction confidence formatted as a percentage
  - Conditional interaction section if `result.getDrugInteractions()` is not empty
  - Recommendations section from `result.getClinicalRecommendations()`
  - Disclaimer/footer:
    - “generated by AI and should be verified by a licensed medical professional.”

---

## 10. Frontend workflow (Upload page)

The UI fragment in `templates/upload.html` provides:

### Tabs
- Upload (drag & drop + choose file)
- Camera (webcam capture to an image)
- Paste Text (textarea input)

### Client-side analysis triggers
- The “Analyze Prescription” button calls `performAnalysis()`:
  - If text tab is active, it POSTs to `/api/analyze-text` with `prescriptionText`.
  - Otherwise it POSTs the selected/captured file to `/api/analyze` under `multipart/form-data` as `prescription`.

### Rendering results
- Results are displayed as a list of detected medicines.
- Clicking a medicine opens a details modal:
  - Fetches `/api/medicine/{id}`.
  - Builds HTML tags for fields such as contraindications, side effects, interactions, dosage forms, and risk level tags.

---

## 11. Docker deployment

A multi-stage Dockerfile exists:

### Build stage
- Uses `maven:3.9.6-eclipse-temurin-21`
- Builds the application jar via:
  - `mvn clean package -DskipTests`

### Run stage
- Uses `eclipse-temurin:21-jre`
- Copies the built jar from `/app/target/clinical-system-0.0.1-SNAPSHOT.jar` to `/app/mediease.jar`
- Exposes port `8080`
- Starts:
  - `java -jar mediease.jar`

> Note: the application uses `server.port=0` (dynamic port) by default, so container networking in Docker should be validated with your deployment setup.

---

## 12. Troubleshooting guide

### 12.1 App starts but browser does not open
- `mediease.browser.open` must be set to `true`.
- If running in headless environment (CI/docker), browser opening is suppressed intentionally.

### 12.2 OCR returns empty text
Common causes:
- Tesseract data path incorrect (`tesseract.data.path`).
- OCR.space API key invalid.
- Image too low quality; tune preprocessing settings (threshold window, contrast stretch, upscaling factor).

Expected pipeline behavior:
- OCR.space text is preferred if non-empty.
- Tesseract used only if OCR.space returns empty.

### 12.3 Groq output parsing fails
- The system expects strict JSON.
- The code strips code fences (```json / ```), but if Groq returns non-JSON text, parsing can still fail.

Mitigations:
- Check prompt contract.
- Log Groq responses server-side.

### 12.4 Medicines are not matching database entries
- Normalization relies on `medicinesynonyms.json`.
- Fuzzy match threshold is `>= 65`.

Mitigations:
- Expand synonyms mapping.
- If needed, adjust string matcher threshold and substring boost behavior.

### 12.5 Upload requests fail with size errors
- Ensure request is below:
  - `max-file-size=10MB`
  - `max-request-size=10MB`

### 12.6 Role redirects or forbidden downloads
- Dashboards rely on authorities like `ROLE_ADMIN`, `ROLE_DOCTOR`, etc.
- Patient PDF access enforces ownership (or privileged roles).

---

## 13. Safety & clinical disclaimer
MediEase provides clinical decision support heuristics and AI-assisted extraction.

The generated PDF includes a disclaimer:
- report is generated by AI and should be verified by a licensed medical professional.

This project is intended for research/prototyping and workflow demonstration, not for autonomous medical diagnosis.

---

## 14. Project structure (where to look)

- `src/main/java/com/mediease/clinicalsystem/controller/MainController.java`
  - Routes, API endpoints, redirects, role dashboards.
- `src/main/java/com/mediease/clinicalsystem/service/AnalysisService.java`
  - Main OCR → AI → normalization → fuzzy decision pipeline.
- `src/main/java/com/mediease/clinicalsystem/service/ImagePreprocessingService.java`
  - Image enhancement steps.
- `src/main/java/com/mediease/clinicalsystem/service/OcrService.java`
  - Tesseract configuration tuned for handwriting.
- `src/main/java/com/mediease/clinicalsystem/service/GroqTextService.java`
  - Prompt contract and Groq calls.
- `src/main/java/com/mediease/clinicalsystem/service/AiOrchestratorService.java`
  - JSON parsing + normalization + fuzzy matching.
- `src/main/java/com/mediease/clinicalsystem/normalizer/MedicineNormalizationService.java`
  - Synonyms and canonical mapping.
- `src/main/java/com/mediease/clinicalsystem/fuzzy/FuzzyStringMatcher.java`
  - Medicine string similarity.
- `src/main/java/com/mediease/clinicalsystem/fuzzy/FuzzyRiskEvaluator.java`
  - Symptom scoring and final decision logic.
- `src/main/java/com/mediease/clinicalsystem/service/PdfService.java`
  - PDF generation.
- `src/main/resources/templates/*.html`
  - UI views (login, dashboards, upload, results, etc.).

---

## 15. Build and run

Typical local run (Maven):
```bash
mvn spring-boot:run
```

Docker build/run (if desired):
```bash
docker build -t mediease .
docker run -p 8080:8080 mediease
```

> Validate the effective HTTP port because the app sets `server.port=0` (dynamic port).

