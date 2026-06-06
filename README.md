<div align="center">

# 🏥 MediEase Clinical System

### Interactive Prescription Intelligence — OCR + AI Extraction + Fuzzy Risk Analysis

[![Java](https://img.shields.io/badge/Java-17-orange?style=flat-square&logo=openjdk)](https://www.java.com)
[![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.2.x-6DB33F?style=flat-square&logo=springboot)](https://spring.io/projects/spring-boot)
[![Groq](https://img.shields.io/badge/Groq-LLaMA_3.1_8B-blueviolet?style=flat-square)](https://console.groq.com)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?style=flat-square&logo=docker)](https://www.docker.com)
[![Status](https://img.shields.io/badge/Status-Research_Prototype-yellow?style=flat-square)](#overview)

> ⚕️ **Disclaimer:** MediEase is intended for **research and prototyping only**. All AI-generated output must be verified by a licensed medical professional before any clinical use.

</div>

---

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Configuration](#configuration)
- [Security & Roles](#security--roles)
- [Analysis Pipeline](#analysis-pipeline)
- [API Endpoints](#api-endpoints)
- [PDF Report Generation](#pdf-report-generation)
- [Default Credentials](#default-credentials)
- [Docker Deployment](#docker-deployment)
- [Troubleshooting](#troubleshooting)

---

## Overview

MediEase is a **Spring Boot** web application that transforms handwritten prescriptions into structured, risk-evaluated medicine lists. It chains cloud OCR, local Tesseract OCR, and a Groq LLM to extract medicines, normalize them against a database, evaluate clinical risk, and produce professional PDF reports — all in a single interactive workflow.

---

## Features

| Feature | Description |
|---|---|
| 📤 Multi-input support | Upload image, webcam capture, or paste text |
| 🔍 Dual OCR engine | OCR.space (cloud) + Tesseract (local), run concurrently |
| 🤖 AI-powered extraction | Groq LLaMA 3.1 corrects OCR noise, returns strict JSON |
| 🔀 Fuzzy medicine matching | Levenshtein + Jaro-Winkler scored against medicine database |
| ⚠️ Clinical risk evaluation | Symptom severity + dosage risk → SAFE / CAUTION / HIGH_RISK |
| 📄 PDF report generation | Professional in-memory report via iText |
| 👥 Role-based dashboards | Admin, Doctor, Pharmacist, Analyst, Patient |
| 🕒 Analysis history | Per-user persistent history with PDF re-download |
| 💊 Pharmacy workflow | Pharmacist can mark prescriptions as DISPENSED |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.2.x · Spring MVC · Spring Security · Spring Data JPA |
| UI | Thymeleaf · vanilla JS · Bootstrap Icons |
| Database | H2 (dev/runtime) · PostgreSQL (production) |
| OCR — local | Tesseract via `tess4j` |
| OCR — cloud | OCR.space API |
| AI extraction | Groq API · LLaMA 3.1 8B Instant |
| HTTP client | OkHttp |
| PDF generation | iText |
| String matching | Apache Commons Text (Levenshtein + Jaro-Winkler) |
| JSON | Jackson |
| Container | Docker (multi-stage build) |

---

## Quick Start

### Prerequisites

- Java 17+
- Maven 3.9+
- [Tesseract OCR](https://github.com/UB-Mannheim/tesseract/wiki) installed — set the path in `application.properties`
- [Groq API key](https://console.groq.com/)
- [OCR.space API key](https://ocr.space/ocrapi)

### Run locally

```bash
mvn spring-boot:run
```

The app binds to a **random available port** (`server.port=0`). The login URL is printed to the console on startup:

```
http://localhost:<port>/login
```

---

## Project Structure

```
src/main/java/com/mediease/clinicalsystem/
├── controller/
│   └── MainController.java              # Routes, API endpoints, role dashboards
├── service/
│   ├── AnalysisService.java             # Main pipeline orchestrator
│   ├── ImagePreprocessingService.java   # Grayscale, threshold, sharpening
│   ├── OcrService.java                  # Tesseract (local)
│   ├── OcrSpaceService.java             # OCR.space (cloud)
│   ├── AiOrchestratorService.java       # JSON parse + normalization + fuzzy match
│   ├── GroqTextService.java             # Groq API calls and prompt contract
│   └── PdfService.java                  # iText PDF generation
├── normalizer/
│   └── MedicineNormalizationService.java  # Synonym → canonical mapping
└── fuzzy/
    ├── FuzzyStringMatcher.java            # Similarity scoring
    └── FuzzyRiskEvaluator.java            # Symptom scoring + final decision

src/main/resources/
├── templates/                             # Thymeleaf HTML views
├── medicinesynonyms.json                  # Synonym mappings
└── application.properties                 # All runtime configuration
```

---

## Configuration

All runtime settings live in `src/main/resources/application.properties`.

```properties
# Server
server.port=0                             # Random port — change for fixed-port deployments
mediease.browser.open=false

# Database (H2 default)
spring.datasource.url=jdbc:h2:file:./data/mediease;LOCK_TIMEOUT=10000;DB_CLOSE_DELAY=-1
spring.jpa.hibernate.ddl-auto=update

# OCR
ocr.space.api.key=YOUR_OCR_SPACE_KEY
ocr.space.base.url=https://api.ocr.space/parse/image
tesseract.data.path=C:/Program Files/Tesseract-OCR/tessdata

# Groq (LLaMA 3.1)
groq.base.url=https://api.groq.com/openai/v1/chat/completions
groq.text.model=llama-3.1-8b-instant
ai.request.timeout=30000
ai.connect.timeout=10000

# File upload limits
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
```

---

## Security & Roles

Spring Security with form-based login, role-based access control, and single-session enforcement. Remember-me tokens are valid for **7 days**.

| Role | Dashboard URL |
|---|---|
| `ROLE_ADMIN` | `/admin/dashboard` |
| `ROLE_DOCTOR` | `/doctor/dashboard` |
| `ROLE_PHARMACIST` | `/pharmacist/dashboard` |
| `ROLE_ANALYST` | `/analyst/dashboard` |
| `ROLE_PATIENT` | `/patient/dashboard` |

---

## Analysis Pipeline

```
Input (image / text)
       │
       ▼
 ImagePreprocessingService
 Resize → Grayscale → Contrast stretch → Upscale ×2.5 → Adaptive threshold → Sharpen
       │
       ▼
 Concurrent OCR  ─────────────────────────────────────
 OcrSpaceService (cloud)    OcrService (Tesseract/local)
 ──────────────────────────────────────────────────────
       │  (OCR.space preferred; Tesseract fallback)
       ▼
 GroqTextService
 LLM corrects OCR noise → returns strict JSON
       │
       ▼
 AiOrchestratorService
 JSON parse → normalization → fuzzy DB matching
 Score: 0.4 × Levenshtein + 0.6 × Jaro-Winkler  (threshold ≥ 65)
       │
       ▼
 FuzzyRiskEvaluator
 Symptom severity + dosage risk → final decision
       │
       ▼
 AnalysisResult  →  UI + JPA history + PDF report
```

### Risk decision rules

| Condition | Decision |
|---|---|
| Match score < 60 | `HIGH_RISK` |
| Dosage **or** symptom risk is HIGH | `HIGH_RISK` |
| Risk MEDIUM **or** match score < 85 | `CAUTION` |
| Otherwise | `SAFE` |

---

## API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/analyze` | Analyze prescription image (multipart) |
| `POST` | `/api/analyze-text` | Analyze pasted prescription text |
| `GET` | `/api/medicine/{id}` | Fetch medicine details as JSON |
| `GET` | `/download-report` | Download PDF of last analysis result |
| `GET` | `/patient/download/{id}` | Patient-scoped PDF download (ownership enforced) |
| `POST` | `/pharmacist/dispense/{id}` | Mark prescription as DISPENSED |
| `POST` | `/api/test-analyze` | Returns mocked AnalysisResult (debug) |
| `POST` | `/api/test-ocr` | Runs full pipeline on uploaded file (debug) |

---

## PDF Report Generation

Generated in-memory using **iText**. Each report includes:

- Header + generated timestamp
- Two-column info table (Patient vs Clinical)
- Executive summary with colour-coded decision (green = SAFE, red = otherwise)
- Medicines table — name, dosage, risk level, confidence %
- Drug interaction section (if applicable)
- Clinical recommendations
- AI disclaimer footer

---

## Default Credentials

> ⚠️ **Change all default passwords before any shared or production deployment.**

| Username | Password | Role |
|---|---|---|
| `admin` | `admin123` | ADMIN |
| `doctor` | `doctor123` | DOCTOR |
| `pharmacist` | `pharm123` | PHARMACIST |
| `analyst` | `analyst123` | ANALYST |
| `patient` | `patient123` | PATIENT |

---

## Docker Deployment

```bash
# Build
docker build -t mediease .

# Run
docker run -p 8080:8080 mediease
```

The Dockerfile uses a multi-stage build:
- **Build stage:** `maven:3.9.6-eclipse-temurin-21`
- **Run stage:** `eclipse-temurin:21-jre`

> Note: The app defaults to `server.port=0` (random port). Set a fixed port in `application.properties` for Docker deployments.

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| Browser doesn't open on startup | `mediease.browser.open=false` or headless env | Set to `true`; suppressed automatically in Docker/CI |
| OCR returns empty text | Bad Tesseract path or invalid OCR.space key | Verify `tesseract.data.path` and OCR.space API key |
| Groq output parsing fails | LLM returned non-JSON text | Check server-side Groq logs; verify prompt contract |
| Medicines not matching DB | Synonym missing or fuzzy score below 65 | Expand `medicinesynonyms.json`; tune threshold if needed |
| Upload fails with size error | File exceeds 10 MB limit | Compress image or raise `max-file-size` in properties |
| Role redirect / forbidden PDF | Wrong role or patient ownership mismatch | Verify user role authorities and history ownership |

---

<div align="center">

🏥 **MediEase** — AI-assisted clinical decision support.

*Reports are generated by AI and must be verified by a licensed medical professional before any clinical use.*

</div>
