# 🩺 MediEase Pro: AI-Based Clinical Prescription Analysis System

MediEase Pro is an enterprise-grade **Spring Boot** web application designed to bridge the gap between unstructured clinical artifacts—such as handwritten or printed medical prescriptions—and structured digital healthcare insights. 

By leveraging a robust, concurrent **OCR pipeline**, state-of-the-art **Generative AI (Groq Cloud LLMs)**, and complex **fuzzy validation engines**, the platform automates the extraction of prescription medicines, performs symptom-aware risk stratification, and compiles downloadable, auditable clinical analytics reports.

---

## 🛠️ Tech Stack & Ecosystem

### Backend Architecture
* **Core Language:** Java 17
* **Framework:** Spring Boot 3.2.x (Spring MVC, Spring Security, Spring Data JPA)
* **Security Model:** Form Login, Role-Based URL Authorization, Concurrent Session Management, Remember-Me, and Audit Logging hooks.
* **OCR Engines:** Tess4J (Local Tesseract integration) & OCR.space REST API execution.
* **Generative AI & Text Parsing:** Groq Cloud APIs (`llama-3.1-8b-instant`), OkHttp, and Jackson Databind.
* **Data Layer:** Relational architecture supporting H2 Database (File Mode for development) and PostgreSQL production routing.
* **String Metrics:** Apache Commons Text (Levenshtein + Jaro-Winkler similarity models).
* **Reporting Engine:** iText PDF Generation Library.
* **Template Engine:** Thymeleaf server-rendered dynamic HTML UI.

### Frontend Presentation
* **Layout Framework:** Bootstrap 5 & Bootstrap Icons (`bi-` layout system).
* **Client-Side Scripting:** Vanilla JavaScript managing asynchronous AJAX fetch payloads (`/api/**`), drag-and-drop file ingestion, and live webcam capture interfaces.

---

## 📂 Project Architecture & Code Map

```text
MediEase-Clinical-AI/
│
├── pom.xml                               # Maven project dependencies & build plugins
├── Dockerfile                            # Multi-stage container deployment architecture
│
└── src/
    ├── main/
    │   ├── java/com/mediease/clinicalsystem/
    │   │   ├── controller/               # MainController (Routing, JSON REST APIs, Dashboards)
    │   │   ├── entity/                   # JPA Entities (AnalysisHistory, Medicine, User, Audit)
    │   │   ├── repository/               # Spring Data JPA Repository layer interfaces
    │   │   ├── normalizer/               # MedicineNormalizationService (Canonical mapping)
    │   │   ├── fuzzy/                    # FuzzyStringMatcher, FuzzyRiskEvaluator logic
    │   │   └── service/                  # Core Pipelines (Analysis, Image Preprocessing, Groq, PDF)
    │   │
    │   └── resources/
    │       ├── templates/                # Server-rendered Thymeleaf HTML dashboards & upload UI
    │       ├── static/                   # Production CSS styling and client-side JavaScript engines
    │       ├── medicinesynonyms.json     # Custom synonym-to-canonical abbreviation mappings
    │       └── application.properties    # Configuration for security, ports, DB, and AI timeouts

⚙️ Core Pipeline & Engineering Implementation
The core processing logic is orchestrated within AnalysisService#analyzePrescription across a strict, 9-stage synchronous pipeline:
[Image Ingestion] 
       │
       ▼
[Image Preprocessing] ──► (Grayscale, Resize, Contrast Stretching, Adaptive Thresholding, Sharpening)
       │
       ▼
[Concurrent OCR] ───────► (CompletableFuture parallel execution: Tesseract Local + OCR.space Cloud)
       │
       ▼
[OCR Selection Engine] ─► (Validates text presence; OCR.space preference with Tesseract fallback)
       │
       ▼
[Structured LLM] ───────► (Groq Llama 3.1 Inference with strict JSON formatting prompts, Temp: 0.1)
       │
       ▼
[Fuzzy Normalization] ──► (Alias removal via medicinesynonyms.json + 0.4*Lev + 0.6*JW scoring)
       │
       ▼
[Risk Stratification] ──► (Weight-based keyword matrix evaluation for symptom/dosage risk profiles)
       │
       ▼
[Audit & Persistence] ──► (Captures role metadata, processes performance metrics, stores row to DB)
       │
       ▼
[Reporting Engine] ─────► (Generates highly structured, auditable iText PDF with clinical disclaimers)

Detailed Pipeline Breakdown:
Adaptive Image Enhancement: Before executing OCR, images are balanced, rescaled by 2.5x, and subjected to an adaptive thresholding algorithm optimized for variations in ink and thin pen strokes. It uses an integral image model (window = 20, threshold: pixel > mean - 5) followed by a 5-neighbor edge sharpening mask.
Concurrent Multi-Engine OCR: To mitigate single-point-of-failure extraction limits, the application wraps local Tesseract processing and external OCR.space API calls inside parallel CompletableFuture tracks. It prefers cloud extraction but falls back seamlessly to local engines if an API failure occurs.
Deterministic LLM Structuring: Extracted data is routed to a Groq llama-3.1-8b-instant model with a low temperature setting (0.1). The prompt instructs the system to operate as a Senior Clinical Handwriting Specialist, parsing fragmented text anomalies into a deterministic, strict JSON layout.
Hybrid Fuzzy String Validator: The extracted text undergoes a custom scoring metric calculation combining Levenshtein Distance (40%) and Jaro-Winkler Similarity (60%). If a substring match boost or an absolute threshold of 65% or higher is satisfied against the database, the medicine is normalized to its canonical form.
Heuristic Risk Evaluation: Clinical risk is stratified by checking symptoms against a pre-compiled severity key-matrix. Scores 10 or above flag HIGH_RISK / SEVERE, scores 5 or above flag MEDIUM / MODERATE, and lower metrics default to safe parameters unless combined text matches flag missing safety indicators.
🔒 Security & RBAC Matrix
MediEase implements a strict, role-based authorization model using Spring Security configuration filters. Access targets are broken down below:
ROLE_ADMIN: Protected Pattern: /admin/** | Target Workflow: Global user audits, infrastructure metrics, security configurations.
ROLE_DOCTOR: Protected Pattern: /doctor/** | Target Workflow: Clinical upload workflows, deep diagnostic results history analysis.
ROLE_PHARMACIST: Protected Pattern: /pharmacist/** | Target Workflow: Dispensing history queues, execution updates (STATUS to DISPENSED).
ROLE_ANALYST: Protected Pattern: /analyst/** | Target Workflow: Aggregate healthcare metrics parsing and operational history overviews.
ROLE_PATIENT: Protected Pattern: /patient/** | Target Workflow: Self-profile maintenance and validated personal report access vectors.
Session Isolation: Configured with maximumSessions(1) and maxSessionsPreventsLogin(false) to invalidate stale duplicate sessions.
Audit Hooks: Lifecycle interactions like system logins, explicit logouts, and highly sensitive dashboard views actively trigger asynchronous decoupled database logs via AuditService.
🚀 Environment Configuration & Local Setup
Prerequisites
Java 17 JDK or higher installed.
Maven 3.9+ binary wrapper.
Local installation of Tesseract OCR (with path matching your configuration).
1. Configure Application Variables
Open src/main/resources/application.properties and define your runtime targets and secure authorization credentials:
# Dynamic port configuration (0 opens a random verified available port)
server.port=8080
mediease.browser.open=true

# Local Tesseract System Engine Data Directory Reference
tesseract.data.path=C:/Program Files/Tesseract-OCR/tessdata

# Secure Infrastructure Keys (OCR.space & Groq API)
ocr.space.api.key=YOUR_OFFICIAL_OCR_SPACE_KEY
groq.api.key=YOUR_SECURE_GROQ_CLOUD_API_KEY

2. Execution via Command Line
Compile the source components, validate your localized Maven profile settings, and execute the runtime:
# Clone the repository
git clone https://github.com/Roshni-1227/MediEase-Clinical-AI.git
cd MediEase-Clinical-AI

# Compile dependencies and launch the Spring Boot profile context
mvn clean package -DskipTests
mvn spring-boot:run

Once initialized, navigate your local browser window to http://localhost:8080.
3. Default Seed Access Credentials
During database initialization (DataInitializer), the development runtime seeds several role-based profiles automatically:
Doctor Profile: User: doctor | Password: doctor123
Pharmacist Profile: User: pharmacist | Password: pharm123
Administrative Profile: User: admin | Password: admin123
🐋 Containerized Docker Deployment
The application features a production-ready multi-stage Dockerfile to compile and minimize target runtime deployment layers securely:
# Stage 1: Dynamic Build Compilation Engine
FROM maven:3.9.6-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Minimized JRE Production Target Runtime
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/clinical-system-0.0.1-SNAPSHOT.jar /app/mediease.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "mediease.jar"]
To run using Docker containers:
docker build -t mediease-clinical-ai .
docker run -p 8080:8080 -e ocr.space.api.key="YOUR_KEY" -e groq.api.key="YOUR_KEY" mediease-clinical-ai

🔬 Clinical Disclaimer & Intended Scope
​MediEase Pro functions purely as an intelligent Decision Support Automation tool and research prototype. Automated results, safety insights, and data extraction layers generated within reports are built using statistical text models and must always be reviewed and confirmed by a licensed medical professional before patient administration.
