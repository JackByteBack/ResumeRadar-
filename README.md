# ResumeRadar

**ResumeRadar** (formerly Resume ATS Checker) is a web application that checks how well your resume matches a job description — before an Applicant Tracking System (ATS) does. Upload a resume and a job description, get a match percentage plus the matched and missing keywords you need to fix.

## How It Works

1. **Extract** — Apache Tika pulls plain text out of your resume and job description (PDF, DOC, DOCX, TXT, ODT, RTF).
2. **Analyze** — Apache OpenNLP part-of-speech tags the text and extracts the nouns (skills, tools, technologies, roles) from both documents.
3. **Match** — each job-description keyword is fuzzy-matched (`commons-text` FuzzyScore) against your resume keywords at a 70% similarity threshold, so `javascript` still matches `Javascript` and minor spelling differences don't cost you.
4. **Score** — `matchScore = matched keywords × 100 ÷ total JD keywords`, returned with the matched and missing keyword lists.

## Features

- Resume upload in **PDF, DOC, DOCX, TXT, ODT, RTF**
- Job description as a **file** *or* **pasted plain text**
- Match percentage with a visual progress bar (red below 40%, green above)
- Matched skills and missing skills listed separately
- Fuzzy keyword matching — tolerant of casing and spelling variations
- NLP-based keyword extraction (noun tagging), not naive word counting
- Error handling with clear messages (bad format, missing JD, parse failure)
- Single runnable JAR — API and UI served from one process
- Docker support (multi-stage build)

## Tech Stack

| Layer     | Tech |
|-----------|------|
| Backend   | Spring Boot 3.4 (Java 17), Apache OpenNLP 2.3, Apache Tika 2.9, commons-text |
| Frontend  | React 19, Vite 6, Bootstrap 5, axios, react-icons |
| Build     | Maven (frontend-maven-plugin builds the React app into the JAR) |
| Deploy    | Docker (multi-stage: Maven + Node → Temurin 17 JRE) |

## Getting Started

### Prerequisites

- **Java 17+**
- **Apache Maven 3.6+**
- Node.js 18+ is **not** required — the Maven build installs Node automatically via `frontend-maven-plugin`

### Option 1 — Run with Maven

```bash
git clone https://github.com/your-repo/ResumeRadar.git
cd ResumeRadar
mvn clean install
mvn spring-boot:run
```

Open **http://localhost:8080**, upload your resume + job description, click **Analyze**.

> The server port defaults to `8080` and can be changed with the `SERVER_PORT` environment variable.

### Option 2 — Run with Docker

```bash
docker build -t resumeradar .
docker run -p 8080:8080 resumeradar
```

Then open **http://localhost:8080**.

### Option 3 — Frontend dev mode (hot reload)

Run the backend in one terminal:

```bash
mvn spring-boot:run
```

Run the Vite dev server in another:

```bash
cd frontend
npm install
npm run dev
```

The React app calls the API at `http://localhost:8080/api/analyze`, so the backend must be running on port 8080.

## API Reference

### `POST /api/analyze`

`multipart/form-data`

| Field               | Type   | Required | Description |
|---------------------|--------|----------|-------------|
| `resume`            | file   | yes      | Resume file (PDF, DOC, DOCX, TXT, ODT, RTF) |
| `jobDescription`    | file   | one of   | Job description file (same formats) |
| `jobDescriptionText`| string | one of   | Job description as plain text |

**Success — `200`**

```json
{
  "matchScore": 72,
  "matchedSkills": ["python", "docker", "developer"],
  "missingSkills": ["kubernetes", "terraform"]
}
```

**Error — `400` / `500`**

```json
{
  "error": "Invalid resume format. Supported formats: PDF, DOC, DOCX, TXT, ODT, RTF."
}
```

**Example**

```bash
curl -X POST http://localhost:8080/api/analyze \
  -F "resume=@resume.pdf" \
  -F "jobDescriptionText=We are looking for a Python developer with Docker experience."
```

## Project Structure

```
├── src/main/java/com/resume/ats/check/
│   ├── AtsCheckerApplication.java      # Spring Boot entry point
│   ├── controller/AtsCheckerController.java   # POST /api/analyze
│   ├── exception/GlobalExceptionHandler.java  # 400/500 JSON errors
│   ├── models/ATSDetail.java
│   └── utils/
│       ├── FileTextExtractor.java      # Tika text extraction
│       ├── OpenNlpSkillExtractor.java  # POS tagging → noun keywords
│       ├── KeywordMatcher.java         # Orchestrates extraction + matching
│       └── SkillMatcherUtil.java       # Fuzzy matching + score
├── src/main/resources/
│   ├── application.properties          # server.port, thymeleaf flags
│   ├── models/en-pos-maxent.bin        # OpenNLP POS model
│   └── static/                         # Built React app (generated)
├── frontend/                           # React + Vite source
│   ├── src/App.jsx                     # Upload form + results UI
│   └── public/screenshots/
├── Dockerfile                          # Multi-stage build
└── pom.xml                             # Maven build + frontend integration
```



| Variable     | Default | Description |
|--------------|---------|-------------|
| `SERVER_PORT`| `8080`  | HTTP port for the API and UI |

## Troubleshooting

- **`Port 8080 already in use`** — stop the process using it or run `SERVER_PORT=9090 mvn spring-boot:run` (and update the API URL in `frontend/src/App.jsx` when running the Vite dev server).
- **Empty results / zero score** — the OpenNLP model needs `src/main/resources/models/en-pos-maxent.bin`; make sure the build completed without errors.
- **Frontend can't reach the API in dev** — start the backend first; the Vite app hardcodes `http://localhost:8080/api/analyze`.
- **Deployed site can't submit** — `App.jsx` uses an absolute `http://localhost:8080` URL. Change it to the relative `/api/analyze` before deploying anywhere other than your own machine.

## Limitations

- Keyword extraction is **noun-based**, so verbs/adjectives like "lead" or "cross-functional" are ignored.
- Single-language (English) OpenNLP model.
- Score reflects keyword coverage, not formatting, ordering, or context quality.

## License

Licensed under the MIT License.
