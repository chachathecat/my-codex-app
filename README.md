# Legal Copilot — Dev Quickstart

## Local Servers
```bash
# Backend (run from repo root so `backend` 패키지를 import 가능)
uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000

# Frontend
cd web
npm install        # first run
npm run dev        # http://localhost:3000

# Worker (optional, Redis/RQ 필요)
python -m rq worker --path backend default
```

## Environment Variables
1. 복사: `cp .env.example .env` 후 값 채우기 (OpenAI, DATABASE_URL, REDIS_URL 등).
2. 프론트: `cp web/.env.example web/.env.local` 후 `NEXT_PUBLIC_API_BASE_URL=http://localhost:8000` 유지 또는 변경.

```env
# .env 주요 값
OPENAI_API_KEY=sk-***
OPENAI_MODEL=gpt-4o-mini
EMBEDDING_MODEL=text-embedding-3-small
DATABASE_URL=postgresql+psycopg2://postgres:postgres@localhost:5432/klegal
REDIS_URL=redis://localhost:6379/0
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:3001
JWT_SECRET=dev-secret
```

## Notes
- Next.js는 `/api/*` 요청을 `NEXT_PUBLIC_API_BASE_URL`로 프록시한다.
- FastAPI 서버는 AuditMiddleware가 활성화되어 있으므로, 로컬 테스트 시에도 `.env`의 `JWT_SECRET`, `ALLOWED_ORIGINS` 값을 맞춰야 한다.
- Redis/RQ를 사용하지 않을 경우 OCR/Export 큐 작업은 동작하지 않는다.
- 한국 법령/판례를 Postgres에 채우려면 `backend/docs/STEP2_data_ingest.md`를 참고해 `backend/scripts/import_kr_law.py` 실행 후 `backend/scripts/index_law_corpus.py`로 임베딩을 생성한다.
- 변호사 직역 확장(Agency Roles)을 사용하려면 `.env`에 `HOMETAX_REDIRECT_URL`, `PATENT_SUBMIT_URL`, `REGISTRY_PORTAL_URL`, `LABOR_PORTAL_URL`을 지정하고 `backend/scripts/seed_agency_roles.py`로 Compliance Center 시드를 채운다.
- Filing-Ready 내보내기는 `/api/exports/filing-ready`(FastAPI)와 `/exports` 페이지(Next.js)에서 Case ID를 입력해 생성할 수 있으며, 생성된 PDF/ZIP/CSV는 `static/exports` 하위에 저장된다.

## Agency Roles Quickstart

1. **DB 마이그레이션**: `cd backend && alembic upgrade head` (0016 버전 포함) → Users/Cases 컬럼 + Compliance Center 테이블 생성.
2. **시드**: `python -m backend.scripts.seed_agency_roles` 실행 → 직역별 정책 가드 기본값 작성.
3. **프론트 설정**: `cd web && cp .env.example .env.local` 후 백엔드 URL과 리다이렉트 포털 URL을 입력. Next.js 개발 서버를 `pnpm dev` 또는 `npm run dev`로 실행.
4. **정책 가드 확인**: Consult/Research/Intake/Exports 화면에 새 탭과 Filing-Ready 프리셋이 노출되며, 변리/세무 제출 버튼은 라이선스 등록 여부에 따라 자동으로 비활성화된다.
5. **Playwright E2E**: `cd web && npx playwright test` (또는 `pnpm test:e2e`)로 직역별 UI 가드를 검증한다.
