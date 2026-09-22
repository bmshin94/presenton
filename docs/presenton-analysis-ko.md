# Presenton 전수조사 & 활용 가이드 (한국어 정리본)

> 이 문서는 Presenton 저장소를 코드/폴더 단위로 전수조사한 뒤,
> "무엇인지 → 어떻게 쓰는지 → 어떤 도움이 되는지 → 어떻게 돈을 버는지"
> 순서로 정리한 요약본입니다.

- **원본(upstream) 저장소:** https://github.com/presenton/presenton
- **내 포크 저장소:** https://github.com/bmshin94/presenton
- **공식 사이트:** https://presenton.ai · **문서:** https://docs.presenton.ai
- **데스크톱 앱 배포용 export 엔진:** https://github.com/presenton/presenton-export
- **라이선스:** Apache 2.0
- **조사 시점 기준 스타:** 약 10.7k ⭐ / 포크 1.6k / 주력 언어 TypeScript
- **분석 기준 버전:** `package.json` → `0.9.10-beta`

---

## 1. Presenton은 한 줄로 뭘까?

**"프롬프트나 문서를 넣으면 AI가 편집 가능한 PPTX/PDF 발표자료를 만들어주는, 완전 오픈소스 셀프호스팅 Gamma 대체제."**

핵심 차별점 4가지:

1. **셀프호스팅** — 내 서버/내 노트북에서 Docker 한 줄로 돌아감. 데이터가 외부로 안 나감.
2. **BYOK (Bring Your Own Key)** — OpenAI, Gemini, Vertex, Azure, Bedrock, Anthropic, Ollama, LM Studio 등 원하는 모델을 골라서 붙임. 구독료가 아니라 내 토큰 값만 냄.
3. **진짜 편집 가능한 산출물** — 이미지로 구운 PPT가 아니라, PowerPoint에서 텍스트 박스를 고칠 수 있는 `.pptx`.
4. **API + MCP 내장** — 사람이 클릭하는 UI뿐 아니라, 다른 프로그램이나 AI 에이전트가 호출하는 "장표 생성 엔진"으로 쓸 수 있음.

저장소의 `VISION.md`가 방향성을 정확히 말해줍니다:
> "Presenton is an open source **document engine**, not a closed design platform.
> It is **infrastructure** for AI native visual workflows."

즉 이 프로젝트의 자기 정체성은 "앱"이 아니라 **"AI 시각 문서 생성 인프라"** 입니다.

---

## 2. 폴더 구조 전수조사

```
presenton/
├─ servers/
│  ├─ fastapi/        ← 두뇌: AI 호출, 슬라이드 생성, DB, MCP 서버 (Python 338개 파일)
│  └─ nextjs/         ← 얼굴: 웹 UI, 슬라이드 에디터, 렌더링/내보내기 (TS/TSX 425개 파일)
├─ templates/         ← 내장 디자인 10종 (general, modern, momentum, executive, ...)
├─ electron/          ← 데스크톱 앱(Win/macOS/Linux) 껍데기
├─ docs/              ← Bedrock, 생성 모드, 템플릿 v2, 리소스 사용량 리포트
├─ scripts/           ← 템플릿 변환기, export 런타임 동기화, 배너 등 빌드 도구
├─ .github/workflows/ ← docker-release, electron 빌드, test-all CI
├─ Dockerfile(.dev)   ← 멀티스테이지 빌드 (python-slim + node22 + nginx)
├─ docker-compose.yml ← production / production-gpu / development / development-gpu 4개 프로파일
├─ start.js           ← FastAPI(8000) + Next.js(3000) + MCP(8001) + nginx 동시 기동 오케스트레이터
├─ layouts.json       ← 슬라이드 레이아웃 사전 (252KB)
└─ nginx.conf         ← 80포트에서 위 3개를 하나로 합쳐주는 리버스 프록시
```

### 2-1. 백엔드 (`servers/fastapi`)

| 경로 | 역할 |
|---|---|
| `api/v1/ppt/endpoints/` | 실제 기능 엔드포인트 21종 — `generation`, `presentation`, `slide`, `outlines`, `template`, `theme`, `icons`, `images`, `fonts`, `files`, `chat`, `community` + 프로바이더별(`openai`, `google`, `ollama`, `openrouter`, `anthropic`) |
| `api/v2/ppt/` | **Smart 모드** — 콘텐츠에 맞춰 레이아웃을 적응적으로 생성, 에디터로 스트리밍 |
| `api/v1/admin/` | 관리자 패널: 사용자 관리, **API 키 발급/조회/폐기** |
| `api/v1/auth/` | 로그인(JWT 쿠키), OAuth, 부트스트랩 관리자 계정, 권한(principal) |
| `api/v1/async_tasks/` | 장시간 생성 작업의 **작업 ID 폴링** |
| `api/v1/webhook/` | 생성 완료 시 외부로 콜백 |
| `utils/llm_calls/` | 프롬프트 엔지니어링의 핵심 8종 — 개요 생성, 구조 생성, 슬라이드 내용 생성, 스마트 생성, 슬라이드/HTML 편집, 웹검색 쿼리 생성 |
| `services/` | 문서 파싱, 이미지 생성, 아이콘 검색(FastEmbed 임베딩), 청킹, mem0 메모리, 내보내기 작업, 웹훅 |
| `models/sql/` | DB 테이블 18종 — presentation, slide, template, api_key, user, font_upload, webhook_subscription 등 |
| `mcp_server.py` | **MCP(Model Context Protocol) 서버 480줄** — OpenAPI 스펙에서 툴을 자동 생성하되 화이트리스트로 7개만 노출 |
| `tests/` | unit 72개 + integration + regression(스냅샷) + edge_cases |

**핵심 인사이트:** `mcp_server.py`의 `MCP_TOOL_NAMES`를 보면 AI에게 노출되는 도구는 딱 7개입니다.
`start_standard_presentation`, `start_smart_presentation`, `list_templates`,
`upload_template_assets`, `start_template_generation`, `upload_files`, `get_job_status`.
→ "LLM이 헷갈리지 않게 도구 수를 의도적으로 줄인다"는 설계 철학이 주석에도 명시돼 있습니다.

### 2-2. 프론트엔드 (`servers/nextjs`)

- Next.js **16.2.6** + React **19.2.6** + Redux Toolkit + Tailwind + Radix UI + shadcn 스타일
- `app/(presentation-generator)/` — upload → outline → presentation(에디터) → template-preview → custom-template
- `app/(export)/pdf-maker/` — **내보내기의 비밀**: 슬라이드를 실제 브라우저 페이지로 렌더한 뒤 그 URL을 헤드리스 브라우저가 찍어서 PDF/PPTX로 변환
- `konva`/`react-konva` — 드래그 편집 캔버스, `tiptap` — 리치 텍스트, `recharts` — 차트, `mermaid` — 다이어그램, `katex` — 수식
- `@paciolan/remote-component` + `@babel/standalone` — **런타임에 사용자가 만든 레이아웃 코드(JSX)를 컴파일해서 주입**하는 구조 (커스텀 템플릿의 핵심)

### 2-3. 템플릿 시스템

`templates/<이름>/template.json` 하나가 디자인 전체를 정의합니다:

```jsonc
{
  "id": "general",
  "theme": {
    "colors": { "primary": "#3B82F6", "graph_0": "#EF4444", ... },  // 그래프 색 10종까지 지정
    "fonts":  { "textFont": { "name": "Poppins", "url": "/vendor/fonts/..." } }
  },
  "merged_components": [ { "id": "background_canvas", "variants": [ { "elements": [...] } ] } ]
}
```

→ 색/폰트/도형 좌표가 전부 **데이터**라서, AI가 이 스키마에 맞춰 값만 채우면 새 디자인이 됩니다.
내장 10종: `general`, `modern`, `momentum`, `executive`, `standard`, `dynamic`, `editorial`, `mosaic`, `swift`, `verdant`

### 2-4. 실행 파이프라인 (`start.js`)

```
Docker 컨테이너 :80 (nginx)
 ├─ /            → Next.js  :3000   (UI · 렌더링 · 내보내기 트리거)
 ├─ /api/v1, v2  → FastAPI  :8000   (AI 호출 · DB · 비즈니스 로직)
 └─ /mcp         → MCP      :8001   (AI 에이전트용 도구 서버)
```

`start.js`는 환경변수 → `app_data/userConfig.json` 변환, 디렉터리 권한 설정(K8s PVC 대응),
세 프로세스 기동과 헬스체크까지 담당하는 700줄+ 오케스트레이터입니다.

---

## 3. 언제 쓰면 좋을까?

| 상황 | Presenton이 주는 것 |
|---|---|
| 매주 반복되는 주간/월간 보고 | 데이터만 API로 던지면 PPT가 자동 생성 |
| 사내 자료라 외부 SaaS 금지 | 완전 폐쇄망(air-gapped) 운영 가능 (`PRESENTON_COMMUNITY_ENABLED=false`) |
| 교육/강의자료 대량 생산 | 주제 리스트를 루프 돌려 수십 개 생성 |
| 우리 회사 PPT 양식 강제 | 기존 PPTX를 올리면 AI가 템플릿으로 변환 |
| AI 에이전트에 "장표 능력" 추가 | MCP 붙이면 Claude/Cursor가 발표자료를 직접 생성 |
| 제품에 PPT 내보내기 기능 필요 | REST API를 백엔드로 그대로 사용 |

---

## 4. 설치 및 사용법

### 4-1. 가장 빠른 길 — Docker (권장)

```bash
# macOS / Linux
docker run -it --name presenton -p 5001:80 \
  -v "./app_data:/app_data" ghcr.io/presenton/presenton:latest

# Windows PowerShell
docker run -it --name presenton -p 5001:80 `
  -v "${PWD}\app_data:/app_data" ghcr.io/presenton/presenton:latest
```

→ 브라우저에서 http://localhost:5001 접속. 키는 UI에서 넣어도 됩니다.

키를 환경변수로 고정하고 UI에서 못 바꾸게 잠그는 운영 모드:

```bash
docker run -it --name presenton -p 5001:80 \
  -e LLM="openai" -e OPENAI_API_KEY="sk-..." \
  -e IMAGE_PROVIDER="dall-e-3" \
  -e CAN_CHANGE_KEYS="false" \
  -v "./app_data:/app_data" ghcr.io/presenton/presenton:latest
```

**완전 무료/로컬 구성 (Ollama + Pexels):**

```bash
docker run -it --name presenton --gpus=all -p 5001:80 \
  -e LLM="ollama" -e OLLAMA_MODEL="llama3.2:3b" \
  -e IMAGE_PROVIDER="pexels" -e PEXELS_API_KEY="..." \
  -v "./app_data:/app_data" ghcr.io/presenton/presenton:latest
```

### 4-2. 데스크톱 앱

https://presenton.ai/download 에서 `.dmg`(macOS) / `.exe`(Windows) / `.deb`(Linux).
Docker 몰라도 됨. 단, **데스크톱 앱에서는 MCP 서버가 꺼집니다** (`PRESENTON_ELECTRON=true` → `DISABLE_AUTH=true`라 인증 충돌 방지).

### 4-3. 소스로 개발 환경 띄우기

```bash
cd electron
npm run setup:env   # npm install + uv sync(FastAPI) + Next.js 의존성
npm run dev         # Electron + 백엔드 + UI 동시 기동
```

전체 CI 검증을 로컬에서 그대로 돌리기: `./test-local.sh`
(레포 툴링 테스트 → FastAPI pytest → Next.js test/lint/build/cypress)

### 4-4. API로 쓰기

```bash
curl -X POST http://localhost:5001/api/v1/ppt/presentation/generate \
  -H "Authorization: Bearer sk-presenton-YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "머신러닝 입문",
    "n_slides": 5,
    "language": "Korean",
    "template": "general",
    "export_as": "pptx"
  }'
```

응답: `{ "presentation_id", "path", "edit_path" }`

**슬라이드별 내용을 직접 지정**하고 싶으면 `slides_markdown` 배열 사용 (배열 길이 = 슬라이드 수):

```json
{
  "content": "",
  "slides_markdown": [
    "# Acme Q2 리뷰\n\n매출 **전년 대비 18% 성장**, 리텐션 94%",
    "# 다음 분기\n\n1. 셀프서비스 온보딩\n2. 엔터프라이즈 파일럿 확대"
  ],
  "template": "general", "export_as": "pdf"
}
```

주요 파라미터: `tone`(default/casual/professional/funny/educational/sales_pitch),
`verbosity`(concise/standard/text-heavy), `web_search`, `include_table_of_contents`,
`include_title_slide`, `files`(먼저 `/api/v1/ppt/files/upload`로 업로드)

### 4-5. MCP로 쓰기

VS Code `.vscode/mcp.json` (Claude Desktop / Cursor / Open WebUI도 동일 형식):

```json
{
  "servers": {
    "presenton": {
      "url": "http://localhost:5001/mcp",
      "type": "http",
      "headers": { "Authorization": "Bearer sk-presenton-0123...SECRET" }
    }
  },
  "inputs": []
}
```

생성 계열 툴은 **작업 ID를 즉시 반환**하므로 `get_job_status`로 폴링해야 합니다.

---

## 5. 자주 묻는 질문 정리

### Q. 플러그인이야? 스킬이야? MCP야?

**셋 다 아니고, "MCP 서버를 내장한 완전한 웹 애플리케이션"입니다.**

| 구분 | 맞나? | 이유 |
|---|---|---|
| 플러그인 | ❌ | 다른 앱에 끼워 넣는 게 아니라 혼자 서버로 뜸 |
| 스킬(Skill) | ❌ | 마크다운 지침 묶음이 아니라 Python+TS 실제 서비스 |
| MCP | 🔶 **부분적으로 O** | `/mcp` 엔드포인트를 제공하는 **MCP 서버**. 다만 MCP는 여러 인터페이스 중 하나일 뿐 |

정확한 계층:
```
[제품] 오픈소스 AI 발표자료 생성 플랫폼
  ├─ 인터페이스 1: 웹 UI (Next.js)
  ├─ 인터페이스 2: REST API (FastAPI)
  ├─ 인터페이스 3: MCP 서버 (fastmcp)  ← 이 부분만 "MCP"
  └─ 인터페이스 4: Electron 데스크톱 앱
```

### Q. API 토큰을 꼭 써야 해?

**두 종류의 토큰이 있고, 성격이 완전히 다릅니다.**

**(1) LLM 프로바이더 키 — 사실상 필수(우회 가능)**
- OpenAI / Gemini / Anthropic / Azure / Bedrock 등은 각각의 키 필요 → **유료**
- **회피법 A:** Ollama 또는 LM Studio로 로컬 모델 → **완전 무료**
- **회피법 B:** "Sign in with ChatGPT" — 기존 ChatGPT 구독 계정으로 로그인, 별도 API 키 불필요
- 이미지도 마찬가지: Pexels/Pixabay(무료 스톡) vs DALL·E 3 / Gemini Flash / ComfyUI(로컬)

**(2) Presenton 자체 API 키 (`sk-presenton-...`) — REST/MCP 호출 시 필수**
- 관리자가 `POST /api/v1/admin/api-keys`로 발급 (`{"user_id","label","expiry_days":90}`)
- 조회 `GET /api/v1/admin/api-keys` · 폐기 `POST /api/v1/admin/api-keys/{id}/revoke` · 재확인 `GET .../token`
- Argon2 해시 저장 + 관리자만 복호화 가능, **기본 90일 만료**
- 브라우저 JWT 쿠키는 MCP 자격증명으로 **불인정**
- MCP 프로세스가 이 키를 **단기 내부 세션으로 교환**해서 쓰므로, 장수명 키가 생성 API로 직접 전달되지 않음
- 로컬 개인 사용이면 `DISABLE_AUTH=true`(Electron 기본값)로 생략 가능

### Q. 왜 깃허브에서 유명할까?

1. **⭐10.7k / 포크 1.6k, Trendshift 등재** — 2025년 5월 시작해서 1년 반 만에 달성한 급성장
2. **Gamma/Canva/Beautiful.ai의 무료 대체제** — 월 $10~20 구독을 대체하는 명확한 경제적 동기
3. **"보안팀이 승인해주는" 유일한 선택지** — 회사 기밀이 담긴 PPT를 외부 SaaS에 못 올리는 조직에 폐쇄망 배포 제공
4. **결과물이 진짜 PPTX** — 이미지 박제가 아니라 편집 가능. 경쟁 오픈소스 대비 압도적 실용성
5. **에이전트 시대에 정확히 맞는 포지션** — "AI가 발표자료를 만든다"는 MCP 시대의 킬러 유스케이스
6. **Apache 2.0** — AGPL과 달리 상업적 재활용·재판매 자유
7. **원클릭 배포 버튼** — Docker / Railway / DigitalOcean / Helm 전부 준비
8. **활발한 관리** — 커밋 히스토리에 RTL 텍스트 렌더링, 로컬라이제이션, 스키마 엄격화 등 커뮤니티 PR이 계속 머지됨

### Q. 로컬 에이전트 구축에 도움이 될까?

**엄청 됩니다. 크게 세 가지 층위에서요.**

**(A) 즉시 쓰는 도구로서** — MCP 붙이면 내 에이전트가 "발표자료 만들기" 능력을 획득. 텍스트만 뱉던 에이전트가 산출물을 내놓게 됨.

**(B) 참고할 레퍼런스 아키텍처로서** — 이 저장소에서 훔쳐올 만한 패턴들:
- `mcp_server.py`: **OpenAPI → MCP 툴 자동 생성 + 화이트리스트 필터링** (라우트 전부를 노출하지 않는 법)
- `MCP_TOOL_NAMES`: 자동 생성된 흉측한 툴 이름을 LLM 친화적으로 리네이밍
- **비동기 작업 + 폴링 패턴**: 10~20분 걸리는 작업을 타임아웃 없이 처리 (`MCP_API_TIMEOUT_SECONDS = 600`)
- **키 교환 패턴**: 장수명 API 키 → 단기 내부 세션 (보안 레이어링 교과서)
- `utils/llm_calls/`: 개요 → 구조 → 내용으로 **단계를 쪼갠 프롬프트 체이닝**
- `get_dynamic_models.py` + `schema_utils.py`: LLM이 JSON 스키마를 지키게 강제하는 법
- `services/mem0_*`: 에이전트 장기 메모리 통합 사례
- `score_based_chunker.py`, `icon_finder_service.py`: FastEmbed 임베딩으로 로컬 시맨틱 검색

**(C) 프로바이더 추상화 레이어로서** — `utils/llm_provider.py`, `llm_config.py`가 15개+ 프로바이더를 한 인터페이스로 묶은 코드. 내 에이전트에 그대로 이식 가능.

### Q. 우리가 React나 PHP로 만들 수 있어?

**React: 이미 React입니다.** 프론트엔드 전체가 Next.js 16 + React 19예요.
`servers/nextjs/`를 뜯어고치면 되고, 핵심 재사용 포인트는 `app/(presentation-generator)/presentation/`(에디터),
`(export)/pdf-maker/`(렌더 → 캡처), `templates/*/template.json`(디자인 스키마)입니다.

**PHP: 가능하지만 나눠서 봐야 합니다.**

| 계층 | PHP로 가능? | 현실 |
|---|---|---|
| API 오케스트레이션 | ✅ 쉬움 | LLM 호출/DB/작업큐는 Laravel로 충분 |
| 슬라이드 렌더링 | ⚠️ 어려움 | React 컴포넌트 렌더가 핵심. PHP는 헤드리스 브라우저 호출로 우회 |
| PPTX 생성 | ✅ 가능 | PhpPresentation 라이브러리 존재 (기능은 python-pptx보다 빈약) |
| MCP 서버 | ⚠️ 가능하나 비주류 | PHP MCP SDK는 생태계가 얇음 |

**현실적인 최선의 조합:**
```
[PHP/Laravel]  사용자·결제·관리자·워크플로우  ← 우리 강점 영역
      ↓ REST API 호출
[Presenton]    장표 생성 엔진 (Docker로 그냥 띄움)  ← 만들지 말고 빌려쓰기
      ↓
[React]        필요하면 프론트만 우리 브랜드로 새로 작성
```
Apache 2.0이라 **이렇게 감싸서 상용 판매해도 합법**입니다.

---

## 6. 수익화 아이디어

> 전제: Apache 2.0 = 상업적 이용/수정/재배포/판매 자유.
> 의무는 딱 3개 — ① 라이선스 사본 포함 ② 저작권 고지 유지 ③ 수정 사실 명시.
> 소스 공개 의무 **없음** (AGPL과 결정적 차이).

### 🥇 티어 1 — 진입장벽 낮고 현금화 빠름

**1. "설치해드립니다" 서비스 (SI/구축 대행)**
- 타깃: 오픈소스에 관심은 있지만 Docker를 모르는 중소기업·학원·병원
- 제공: 서버 세팅 + 도메인/SSL + 회사 PPT 양식을 템플릿으로 변환 + 직원 교육 2시간
- 가격: 구축 300~800만원 + 월 운영 30~50만원
- 왜 되나: 커스텀 템플릿 변환은 기술이 필요한데, 고객은 "우리 회사 양식"을 절대 포기 못 함

**2. 한국형 템플릿 마켓플레이스**
- `template.json` 하나가 디자인 전체를 정의 → **템플릿은 그 자체로 팔리는 디지털 상품**
- 상품 예시: 대학 발표 양식팩, 정부 R&D 과제 제안서 양식, 국내 VC 선호 IR 덱, 병원/약국, 부동산 매물 브리핑
- 가격: 단품 1~3만원 / 번들 9.9만원 / 구독 월 9,900원
- 장점: 한 번 만들면 재고 없이 무한 판매, 원본 코드 수정 불필요

**3. 유튜브/강의 콘텐츠**
- "월 2만원 Gamma 구독 끊고 무료로 쓰는 법", "내 PC에서 돌리는 AI PPT 서버"
- 애드센스 + 인프런/클래스101 강의 + 제휴. 위 1·2번의 **유입 퍼널** 역할까지 겸함

### 🥈 티어 2 — 개발 필요, 수익 규모 큼

**4. 버티컬 SaaS (특정 업종 전용 래퍼)**
- 핵심 전략: 범용 툴로 경쟁하지 말고 **한 업종만 깊게** 파기
- 후보 ①: **학원/과외 강사용** — 교재 PDF 업로드 → 차시별 수업 PPT 자동 생성 (월 29,000원)
- 후보 ②: **부동산 중개** — 매물 정보 + 사진 → 브리핑 자료 (월 49,000원)
- 후보 ③: **정부지원사업 컨설팅** — 사업계획서 → 발표 심사용 덱 (건당 10만원)
- 후보 ④: **영업팀** — CRM 데이터 → 고객 맞춤 제안서 (시트당 월 3만원)
- 구조: Laravel/Next.js로 UI·결제·사용자 관리 → Presenton API를 백엔드 엔진으로
- 마진: LLM 비용 건당 200~500원 vs 판매가 → **80%+ 마진**

**5. API 리셀링 (Generation-as-a-Service)**
- 개발자 대상으로 "PPT 생성 API"를 크레딧제 판매. 문서 1줄 = `POST /generate`
- 가격: 100건 9,900원 / 1,000건 79,000원
- 고객: 자사 서비스에 "PPT로 내보내기"가 필요한 국내 SaaS들
- 핵심: Presenton을 GPU 서버에 띄우고 큐잉/레이트리밋/과금만 얹으면 됨

**6. 사내 폐쇄망 구축 (엔터프라이즈)**
- 타깃: 금융/공공/의료/방산 — **외부 SaaS 사용이 규정상 금지된 곳**
- 이게 Presenton의 최고 강점: `PRESENTON_COMMUNITY_ENABLED=false` + Ollama = 인터넷 완전 차단 상태에서 동작
- 가격: 구축 2,000만~1억 + 연간 유지보수 20%
- 경쟁자가 거의 없음 (Gamma는 애초에 입찰 참여 불가)

### 🥉 티어 3 — 장기·고위험고수익

**7. 부족한 기능을 메꾸는 유료 애드온**
- 한국어 폰트팩 & 조판 최적화, 사내 브랜드 가이드 자동 적용 엔진,
  Notion/Slack/Google Drive 커넥터, 발표 스크립트 자동 생성(TTS 연동)
- Apache 2.0이므로 **애드온은 클로즈드 소스로 판매 가능**

**8. 기여 → 신뢰 → 수주 (오픈소스 레버리지)**
- upstream에 한국어 관련 PR을 꾸준히 머지시키면 "Presenton 한국 전문가" 타이틀 확보
- 이 타이틀이 위 1·4·6번의 영업 무기가 됨. 비용 0원, 시간만 투자

### ⚠️ 주의사항

- **LLM 비용이 원가**: 40장 덱 한 건에 수백~수천원. 반드시 **크레딧제/사용량 과금**으로 설계 (무제한 요금제 금지)
- **리소스**: `docs/presenton-resource-usage.md`에 따르면 동시 4명 부하 시 **CPU 순간 337%, 힙 11.8GB** 피크. 인스턴스당 동시 처리량 계산 필수
- **라이선스 준수**: LICENSE·NOTICE 파일 포함 + 수정 사실 명시 (NOTICE는 1.2MB, 절대 지우면 안 됨)
- **상표권**: "Presenton" 이름을 제품명으로 쓰지 말 것. Apache 2.0은 **상표권을 부여하지 않음**. 자체 브랜드 필수
- **업스트림 추종 비용**: 포크를 크게 뜯어고치면 병합 지옥. 가능한 한 **API 래핑으로 분리**할 것

### 💡 추천 순서

```
1단계  콘텐츠(유튜브/블로그)로 인지도 + 유입 만들기        [비용 0]
2단계  템플릿 팩 판매로 첫 현금흐름 검증                    [비용 0]
3단계  구축 대행 1~2건 수주 → 실제 고객의 진짜 페인포인트 학습
4단계  그 페인포인트로 버티컬 SaaS 1개 집중 개발            ← 진짜 승부처
5단계  레퍼런스 쌓이면 엔터프라이즈 폐쇄망 입찰 진입
```

---

## 7. 한 줄 결론

> **Presenton은 "AI PPT 앱"이 아니라 "AI 문서 생성 엔진"입니다.**
> 그래서 이걸 쓰는 최선의 방법은 앱으로 쓰는 게 아니라,
> **엔진으로 빌려 쓰고 그 위에 우리 비즈니스를 얹는 것**입니다.

---

*이 문서는 Claude Code 세션에서 저장소를 전수조사하여 작성되었습니다.*
