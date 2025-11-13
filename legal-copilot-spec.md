다음 전체를 그대로 복사해서 `legal-copilot-spec.md` 같은 파일로 저장한 다음, 코드 모델(Codex CLI)에 붙여넣고 “이 파일을 읽고 Step 1부터 구현해줘” 식으로 사용하면 된다.

---

# Project Spec – Korean Legal Copilot (Attorney Workflow Edition)

> 한국 로펌/개업 변호사용 **Legal Copilot** 플랫폼. FastAPI + Next.js 모노레포 기준으로 **사건 인입 → 컨플릭트 체크 → 연구/RAG → 서면 초안 → 산출물/협업** 전체 워크플로를 한 번에 다룬다.  
> 모든 산출물은 “업무보조 도구”임을 명시하고, 변호사 최종 검토를 전제로 한다.

---

## 0. Purpose & Success Criteria

1. **Attorney-first workflow**: Intake, conflict, 체크업, 초안, 협업 전 과정을 한 화면으로 이어 붙인다.
2. **Grounded intelligence**: 한국 법령/판례 + 내부 자료를 RAG로 묶고, 인용/출처를 강제한다.
3. **Operational guardrails**: 조직·사용자 격리, 감사지표, 비동기 작업(Export/OCR)까지 관리한다.
4. **Developer ergonomics**: FastAPI(백엔드) + Next.js 14(프론트) + Postgres + Redis 워커 + FAISS 인덱스의 현재 레포 구조를 유지하면서 확장한다.

### 0.1 Primary Personas

- **Partner / Senior Attorney**: 사건 승인, 최종 산출물 검토, 리스크 승인.
- **Associate / Junior Attorney**: 사실관계 정리, IRAC 분석, 초안 작성.
- **Paralegal / Staff**: Intake, 컨플릭트 체크, 자료 업로드, 일정 관리.

### 0.2 Current Repo Snapshot

```
/
├─ backend/              # FastAPI entrypoint(main.py), routers/, SQLAlchemy models, Alembic
├─ web/                  # Next.js 14 (App Router) UI, docs/, SPEC.md
├─ AGENTS.md             # Codex 작업 가이드
├─ package.json          # repo-level tooling
└─ legal-copilot-spec.md # (본 파일)
```

---

## 1. Attorney Workflow (Day-in-the-life)

### 1.1 Intake & Conflict Check

1. **Lead capture**: 상담 폼 → `clients`, `matters`, `cases`, `case_parties` 초안 레코드 생성.
2. **Conflict search**: `/api/conflicts/check?q=...` (pg_trgm + ILIKE fallback, `conflict_index` 테이블)로 조직 DB의 기존 고객/상대방과 비교.
3. **Risk memo**: Intake 시점에 태깅되는 사실·키워드(`case_events.source=intake`)를 만들어 뒤 단계에서 RAG 컨텍스트로 재활용.

### 1.2 Case Room Setup

1. **Case summary**: `cases`의 status(`intake → retained → filed → hearing → judgment → closed`), court, type, summary 필드를 유지.
2. **Timeline & tasks**: `case_events`, `issues`, `tasks` 테이블로 사건 히스토리/쟁점/To-do를 분리 관리. UI는 `/cases/[id]` 페이지 탭 구조(Overview, Timeline, Evidence, Drafts, Tasks).
3. **Evidence mapping**: `documents`, `document_texts`, `document_chunks`, `evidence_links`를 통해 문서-쟁점 매핑 및 추후 Draft 프롬프트 구성.

### 1.3 Research & Checkup

1. **Analyze API**: `/api/analyze` (FastAPI router `backend/routers/analyze.py`) → RAG stub + OpenAI → IRAC JSON, citations, disclaimer.
2. **Consult API**: `/api/consult` → 간단 QA(chat) 또는 Analyze 워크플로 재사용. `backend/ingest_pipeline.py`의 `get_indexer`, `parse_korean_query`로 한국어 질의 파싱.
3. **Case Checkup**: `/api/checkup` (in `backend/main.py`) → facts 기반 이슈 추출 + statutes/precedents 추천.
4. **Copilot Research**: `/api/copilot/research` → pgvector/ARRAY 임베딩과 OpenAI 기반 판례 요약.

### 1.4 Draft & Document Review

1. **Draft generation**: `/api/drafts/generate` → Document + DocumentText 저장, OpenAI 호출(없으면 Placeholder).
2. **Doc assembly**: `/api/docs/assemble` → `backend/static/assembled` 하위에 DOCX placeholder 저장(추후 템플릿 병합).
3. **Audit middleware** (`backend/middleware/audit.py`)가 모든 요청을 로깅하여 서면 생성/다운로드 기록을 남김.

### 1.5 Delivery & Follow-up

1. **Exports**: FastAPI `/export` (web/README 참고) + `ExportArtifact` 테이블(backend/models/domain.py). PDF/DOCX/PPTX 산출물 + Redis/RQ 워커(`web/worker.py` 참고)로 비동기 처리.
2. **OCR Contracts**: `/ocr_contract` & `/jobs/ocr` → 계약서 업로드 + 위험 탐지 + progress polling(Next.js `contracts` 페이지).
3. **Consult deliverables**: 응답마다 `DISCLAIMER` 필수 노출. `AuditMiddleware`가 변호사가 아닌 사용자의 호출을 차단하도록 확장 가능.

---

## 2. System Architecture & Runtime

### 2.1 Components

| Layer | Stack & Location | Notes |
| --- | --- | --- |
| Backend API | FastAPI (`backend/main.py`, routers/*) | SQLAlchemy models, Alembic migrations, Audit middleware, OpenAI client |
| Worker | Python + RQ (`web/worker.py` -> relocate to backend soon) | Handles OCR, exports, FAISS ingest jobs |
| Frontend | Next.js 14 App Router (`web/src`) | Tailwind + shadcn, pages: Dashboard, Analyze, Contracts, Exports, Cases |
| Database | Postgres 15+ | `backend/db.py`, `backend/models/*`; uses ARRAY(double precision) for embeddings; optional `pgvector` |
| Search/RAG | FAISS index (`web/ingest_pipeline.py` legacy) + SQL chunk tables | `ingest_pipeline.py` exposes `get_indexer`, `parse_korean_query` |
| Cache/Queue | Redis + RQ | Job IDs surfaced via `/jobs/{id}` |
| Storage | Local `./backend/static/assembled`, `./data/exports`, `./data/uploads` | Switchable to S3/R2 later |

### 2.2 Dev Commands

```bash
# Backend (repo root)
cd backend
uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000

# Frontend
cd web
npm install      # 최초 1회
npm run dev      # http://localhost:3000

# Worker (if Redis/RQ configured)
cd backend
python -m rq worker --path . default
```

### 2.3 Environment Baseline

`backend/.env`
```env
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini
EMBEDDING_MODEL=text-embedding-3-small
DATABASE_URL=postgresql+psycopg2://postgres:postgres@localhost:5432/klegal
REDIS_URL=redis://localhost:6379/0
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:3001
JWT_SECRET=dev-secret
```

`web/.env.local`
```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
```

---

## 3. Data & Schema Highlights (SQLAlchemy)

- `organizations`, `users`: multi-tenant boundary, roles(`partner|associate|staff|admin`), `created_at/updated_at`.
- `cases`: status enum-ish string, `summary`(LLM), `created_by`.
- `case_parties`, `case_events`, `issues`, `tasks`: timeline + issue tracker.
- `clients`, `matters`, `documents`, `document_texts`, `document_chunks`: client knowledge base + drafting payload.
- `cases_law`, `cases_law_fulltext`, `case_law_chunks`, `statutes`, `statute_chunks`: RAG corpus (ARRAY embeddings; upgrade to VECTOR when pgvector ready).
- `conflict_index`: denormalized names for fuzzy search (source system populates).
- `audit_logs` (via middleware) recommended for traceability (append-only JSON).

---

## 4. API Surfaces (Backend)

| Feature | Endpoint(s) | File | Notes |
| --- | --- | --- | --- |
| Case workspace | `GET /api/cases/{id}` | `backend/routers/cases.py` | Returns case, parties, events, issues, tasks |
| RAG Analyze | `POST /api/analyze` | `backend/routers/analyze.py` | Builds template prompt per jurisdiction/persona |
| Consult/Chat | `POST /api/consult` | `backend/routers/consult.py` | Chat vs analyze modes, KR index fallback |
| Case Checkup | `POST /api/checkup` | `backend/main.py` | Facts → issues/statutes/precedents heuristics |
| Copilot Research | `POST /api/copilot/research` | `backend/routers/copilot.py` | pgvector search + OpenAI completion |
| Drafts | `POST /api/drafts/generate` | `backend/routers/drafts.py` | Persists Document + DocumentText |
| Docs assemble | `POST /api/docs/assemble` | `backend/routers/docs.py` | Placeholder DOCX writer |
| Conflict check | `GET /api/conflicts/check` | `backend/routers/conflicts.py` | pg_trgm similarity |
| Jobs/OCR/Export | `/jobs/ocr`, `/export`, `/ocr_contract*` | `web/app_server.py`, `backend/main.py` | Queue integration + Postgres persistence |

All endpoints must add disclaimers in responses containing legal text and honor `AuditMiddleware`.

---

## 5. Frontend Experience (Next.js)

- **`/` Dashboard**: latest analyses, exports, OCR jobs.
- **`/analyze`**: IRAC form (jurisdiction/persona toggle) → displays answer, citations, disclaimers, export buttons.
- **`/cases/[id]`**: tabbed view using `CaseDetailTabs` (Overview / Timeline / Evidence / Drafts / Tasks).
- **`/contracts`**: immediate OCR vs queued OCR flows + SWR polling.
- **`/exports`**: artifact list with download links.
- **`DraftsTab`**: draft generator modal + editor, hitting `/api/drafts/generate`.

Use Tailwind + shadcn, keep Emerald accent, avoid over-styling. API helper lives in `web/src/lib/api.ts`.

---

## 6. Implementation Steps (for Codex Agents)

> 아래 Step들을 순차적으로 실행하면 된다. 각 Step은 FastAPI + Next.js 현재 구조를 전제로 하며, “코드는 바로 붙여넣을 수 있게”를 원칙으로 한다.

### [Step 1] Local Dev Baseline (FastAPI + Next.js)

```text
목표: repo 루트에서 백엔드/프론트 실행 루틴을 문서화하고, .env 템플릿을 최신화한다.

1) backend: `uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000` (반드시 repo 루트에서 실행, 이유: 패키지 루트가 `backend`).
2) frontend: `cd web && npm install && npm run dev`.
3) worker(optional): Redis/RQ 환경이면 `python -m rq worker --path backend default`.
4) `.env.example`, `web/.env.example`를 갱신해 OPENAI/DATABASE/REDIS/NEXT_PUBLIC_API_BASE_URL을 명시한다.
5) README/Spec에 위 명령을 복사-붙여넣기 가능하도록 Bash 코드블록으로 남긴다.
```

### [Step 2] Attorney Intake & Conflict Ops

```text
목표: Intake → Conflict → Case 생성 흐름을 API + DB로 정리.

1) SQLAlchemy 모델 보강: `clients`, `matters`, `cases`, `case_parties`, `conflict_index` 스키마와 Alembic 마이그레이션 추가.
2) `/api/conflicts/check`: pg_trgm 확장 사용 시 similarity, 미설치면 ILIKE fallback (이미 구현된 로직을 확장).
3) `/api/cases` POST/PATCH 라우트 작성: Intake 데이터 저장, status 전환, parties/events bulk upsert.
4) Intake 양식(Next.js)에서 바로 conflict search→case 생성까지 이어지도록 API client 작성.
5) 모든 요청은 `AuditMiddleware`로 로깅되고, intake 레코드는 `case_events.source='intake'`로 남긴다.
```

### [Step 3] Research & Checkup Stack

```text
목표: Analyze / Consult / Checkup / Copilot Research를 변호사용 기준으로 강화.

1) `backend/ingest_pipeline.py`의 `get_indexer`, `parse_korean_query`를 활용해 checkup/analyze에 실제 RAG 데이터를 주입.
2) `/api/checkup` 결과 스키마(issues/statutes/precedents)를 명시하고, Postgres `case_law_chunks`, `statute_chunks`를 조회하도록 수정.
3) `/api/analyze` 템플릿(`backend/templates/{jurisdiction}/{persona}.md`)을 채워 변호사/일반 사용자별 톤을 구분.
4) `/api/copilot/research`에 pgvector 우선, 실패 시 Python cosine fallback. filters(court/from/to) 동작 예제 포함.
5) Next.js Analyze/Research UI에서 citation 카드, copy-to-clipboard, disclaimer 표시를 구현.
```

### [Step 4] Draft & Document Assembly

```text
목표: Drafts + Docs 워크플로 완성.

1) `/api/drafts/generate`: OpenAI 호출 실패 시에도 placeholder 반환, `Document`/`DocumentText` insert 보장.
2) `/api/docs/assemble`: 템플릿 엔진(Jinja/DocxTPL) 통합 준비 – 현재는 placeholder DOCX ⇒ TODO 주석, 파일 경로 반환.
3) `/api/drafts/list` (신규)로 사건별 draft 목록을 반환, DraftsTab에서 SWR로 표시.
4) Draft 생성 모달: docType select, instruction textarea, API 호출(fetch/axios) + optimistic UI.
5) Draft 저장 시 `documents` version 필드를 증가시키고, `document_chunks` 큐 작업을 enqueue하여 RAG에 반영한다.
```

### [Step 5] Case Room UI & Timeline

```text
목표: `/cases/[id]` 페이지 탭 구조와 데이터 fetch를 완성.

1) 서버 컴포넌트에서 `GET /api/cases/{id}` 호출 → props로 `CaseDetailTabs`에 전달.
2) 각 탭 컴포넌트(OverviewTab, TimelineTab, EvidenceTab, DraftsTab, TasksTab)를 `web/src/components/cases/`에 분리.
3) TimelineTab: 사건 이벤트를 날짜순으로 그룹화, EvidenceTab: documents ↔ issues 매핑 테이블 뼈대.
4) TasksTab: 상태 필터/정렬, 완료 토글. OverviewTab: status pill, next hearing date, summary markdown.
5) Tailwind + shadcn tabs/accordion을 사용, 모바일 대응(세로 스크롤) 고려.
```

### [Step 6] Guardrails, Logging, QA

```text
목표: 변호사용 감사·품질 체계를 명문화.

1) `backend/middleware/audit.py` 확장: actor(org/user), endpoint, payload 크기, response hash 저장.
2) `/api/health` 및 `/api/meta`(신규)로 버전, git commit, OpenAI model 등을 노출하여 운영 추적.
3) Pytest + Playwright 스모크 테스트 뼈대: Analyze happy path, Draft generation fallback, Conflict search.
4) Error budget: 모든 LLM 응답에 disclaimer + fallback, 실패 시 사용자에게 “검토 필요” 메시지 노출.
5) Docs/README에 운영 수칙(비밀유지, 데이터 분리, 키 교체) 체크리스트를 추가.
```

---

> 이 파일을 Codex에 주입한 뒤 Step 1부터 순차적으로 요청하면, FastAPI + Next.js 기반의 변호사 워크플로를 재현/확장할 수 있다.
