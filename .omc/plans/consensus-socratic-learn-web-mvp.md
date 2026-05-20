# Consensus Plan: socratic-learn-web MVP (v2)

> Status: **pending approval (v3, Architect+Critic iter 2 통과 — Critic APPROVED, Architect의 경미 4건 + Critic Major 4건 patch 흡수)**
> Source spec: `.omc/specs/deep-interview-socratic-learn-web.md` (ambiguity 15%)
> Mode: RALPLAN-DR **short**
> Changelog: 하단 참조

---

## Requirements Summary

CLI 기반 socratic-learn 스킬을 GUI 웹 서비스로 옮긴다. **단 두 가지** 핵심 가치:
1. 확인 질문 번호 입력 등 CLI UX 불편함 해소 (정량 검수 가능해야 함)
2. 학습 이력 영구 회고 (검색→재개 도달 시간 검수 가능해야 함)

규모: 50명 단기 동시 접속. 초기 운영자 단일 LLM 키 → 추후 BYOK.

### 잠금된 기술 결정 (spec 출처)
- Frontend: Compose Multiplatform Web (Wasm)
- Backend: Ktor (Kotlin)
- DB/Auth: Supabase Postgres + Supabase Auth (Google OAuth)
- LLM: Anthropic Claude API via Kotlin SDK, SSE streaming

> **Lock 재협상 조건**: M0 스파이크 3건 중 1건 이상 실패하거나, M3 종료 시점에 "도메인 공유 절감 라인 수 : Wasm yak shaving 시간" 비율이 1:3 이상 역전이면 ADR 재오픈 (Next.js+tRPC 등으로 swap 검토). SSE wire contract를 OpenAPI로 표현해 swap cost를 제한한다.

---

## RALPLAN-DR Summary

### Principles
1. **SSE wire contract + GradingSignal taxonomy의 단일 정의**(좁은 의미의 도메인 공유). 풍부한 동작 공유가 아니라 enum/스키마 계약만 commonMain에 둔다
2. **CLI 우위 사라짐 = 정량 검수**. 모든 UI 결정은 "1사이클당 입력 수 / 회고 도달 시간"으로 검수 가능해야 한다. 미세 디자인 최적화는 후순위
3. **SKILL.md 원칙은 프롬프트 본문에만 비변형 이식**. 출력 포맷은 enum-strict (이모지/한국어 라벨 ≠ enum value)
4. **50명 운영 규모 이상 최적화 금지** (큐/캐시/CDN 도입 X). 단, LLM 측 throttle은 인프라가 아니므로 필수
5. **BYOK 자리만**: schema는 미리, UI/활성은 MVP 외
6. **Prompt cost guard**: 이전 cycles 누적 주입 금지. **요약 후 N토큰 cap 내에서만** 주입. 단일 요청 토큰 예산 명시

### Decision Drivers
1. **SSE wire contract + Taxonomy의 단일 정의 가치** > LLM 생태계 최신성
2. **SSE 스트리밍 응답** 필수 (설명+질문 한 응답 길이)
3. **운영 비용 cap 가능성**: 일일/분당/동시 cap 3종 모두 필요

### Viable Options (진지한 재평가)
| Option | Pros | Cons | Status |
|---|---|---|---|
| **Ktor + Supabase** (선택) | commonMain 좁은 계약 공유, JVM 단일 jar, 사용자 Kotlin 친숙 | Anthropic SDK streaming 미확정(M0 검증), CMP Web 디버깅 거침, Markdown/highlight 라이브러리 결핍 | ✅ 채택(M0 통과 조건부) |
| Hono + Bun + Supabase | 백엔드 최소 부피, Anthropic TS SDK 최신, tRPC+zod로 wire 계약 공유 가능, dogfooding 빠름 | 도메인 로직이 backend에만 있어 공유 자산은 zod schema 정도. CMP 미선택 시 사용자 의도와 멀어짐 | ❌ CMP 잠금 우선. 단 M3 게이트에서 swap 검토 |
| FastAPI + Supabase | LLM 생태계 풍부, 빠른 프로토타이핑 | Kotlin/CMP와 거리, 컨테이너 운영 부담 | ❌ |
| Next.js + tRPC (참조) | dogfooding 가장 빠름, 풍부 생태계 | CMP와 코드 공유 불가, deep-interview에서 사용자가 CMP 명시 | ❌ swap 후보로만 유지 |

---

## 운영 비용 추정 (수치 가이드)

claude-sonnet-4-6 기준 (2026-05 단가 추정 in $3/MTok input, out $15/MTok output. 실측 시점에 정정):
- 한 사이클당 평균 토큰: **in 3K + out 2K**
- 사용자당 일 평균: **3 사이클** (정상 학습 페이스)
- 일일 예상 비용: `50 users × 3 cycles × (3K × $3 + 2K × $15) / 1M = 50 × 3 × ($0.009 + $0.030) = 50 × 3 × $0.039 ≈ $5.85/day`
- 월 예상: **약 $175 (정상)**
- 폭주 시나리오(한 사용자 long cycle 반복, 일 30 사이클): **$58/day/user → 운영자 키 가드 필수**

### 일일 토큰 cap 산식 (역산)
운영자 월 예산 상한 = $300 → 일 $10
사용자당 cap (도달 안 되는 정상값) = `$10 / 50 / ($0.039 per cycle) ≈ 5.1 cycles/day`
→ **사용자당 일일 25K input + 15K output 토큰 cap** (5 사이클 여유)

### 분당/동시 cap
- 분당 동시 SSE 연결: 사용자당 1 (브라우저당 한 cycle stream만)
- 동시 LLM 호출: 운영자 키 분당 cap (Anthropic 기본 50 rpm 가정) → ktor RateLimit으로 user 분당 5 호출 cap

---

## Acceptance Criteria

### 핵심 가치 정량 검수 (가장 위, 가장 중요)
- [ ] **CLI 대비 입력 수**: 한 사이클(설명 받음 → 3-8개 질문 답변 → 채점 받음 → 분기 선택) 동안 사용자 키 입력 + 클릭 수가 CLI 대비 **50% 이하**. 측정: 5개 시나리오를 운영자 본인이 동일 학습 주제로 CLI 1회 + 웹 1회씩 수행하여 비교
- [ ] **회고 도달 시간**: 어제자 세션을 좌측 사이드바에서 검색→재개까지 **3 클릭, 5초 이내**. 운영자 dogfooding 일주일 동안 매일 확인
- [ ] **Dogfooding 7일 체크리스트**: 운영자 본인이 7일 연속 매일 1세션 학습, 7일 차 종료 시점에 "CLI보다 명확히 낫다"고 본인 서명. 5/7일 이상 실패 시 plan 재오픈
- [ ] **세션 재구성 일치**: 어제자 세션을 회고 화면에서 연 표시 결과가, 어제 라이브 화면 결과와 **DOM 텍스트 diff 없음** (스냅샷 테스트)

### Spec 승계 11개 (그대로)
- [ ] 비로그인 사용자가 첫 화면에 접근하면 Google OAuth 로그인 화면으로 리다이렉트된다
- [ ] 로그인 후 첫 화면의 "학습 개념 한 줄 입력" 폼에 제출 시 첫 사이클 응답이 SSE로 스트리밍된다
- [ ] 첫 응답에 (a) 수준 추정, (b) 1단계 분해 개념의 설명(정의/코드/시각화), (c) 3-8개 확인 질문이 한 응답에 모두 포함된다
- [ ] 확인 질문은 각 문항마다 독립 텍스트박스 + "모르겠어요" 토글
- [ ] "한 번에 제출" 클릭 시 모든 답이 한 번에 채점되고 🟢/🟡/🔴 신호가 문항별 표시
- [ ] 채점 응답 끝에 2-4개 분기 선택지 카드 + 자유 입력
- [ ] Markdown(GFM 테이블, 코드 블록), ASCII 다이어그램, 비교표가 CLI보다 명확히 시각 렌더
- [ ] 사이드바 "내 학습 이력" → 세션 목록 → 클릭하면 과거 사이클 회고
- [ ] 과거 세션 "이어서 학습" → 마지막 cycle 끝에서 새 cycle 진입
- [ ] 일일 토큰 cap 초과 → 친절한 한국어 안내 + 입력 차단
- [ ] BYOK 활성 후 본인 키 등록 사용자는 운영자 cap 미적용

### 구현 검수 추가
- [ ] **commonMain 단일 계약**: `Cycle`/`Question`/`Answer`/`GradingRow`/`Branch`의 `@Serializable` data class + enum 5종(`GradingSignal`, `QuestionKind`, `BranchKind` 등)이 frontend/backend 양쪽 build에 import 되어 실제 사용된다. **enum value에는 이모지/한국어 금지** (예: `MISCONCEPTION` ⭕, `"🔴"` ❌)
- [ ] **i18n 분리**: `GradingSignal.label(locale)` 같은 라벨 리졸버가 commonMain에 위치, frontend만 호출. backend는 enum-only로 DB write
- [ ] **Wasm 호환 가드**: detekt 룰 또는 `compileOnly` 분리로 `java.time`/`kotlin.io.*`/`Ktor server-side` commonMain 사용 시 빌드 실패
- [ ] **/api/cycle** POST가 Supabase JWT를 검증, 미인증 → 401
- [ ] **LLM 프롬프트** (`backend/src/main/resources/prompts/socratic_cycle.md`)가 SKILL.md 사이클 6단계 + 원칙 6개 모두 인용/임베딩
- [ ] **SSE 첫 토큰 도착** p95 < 3s (운영자 키, claude-sonnet-4-6)
- [ ] **DB write**: cycles 테이블의 questions/answers/grading/branches가 `jsonb`로 보존, 회고 화면이 동일 페이로드로 재구성 가능
- [ ] **HTTP 429 + 한국어 안내**: 일일 cap 초과 시
- [ ] **분당/동시 cap**: 사용자당 분당 5 cycle 호출 초과 시 429
- [ ] **LLM 출력 JSON 계약 강제**: cycle_payload SSE 이벤트가 명시 스키마 통과 못 하면 frontend가 "AI 응답 재시도" 버튼 노출 + backend가 비용 로그 기록
- [ ] **부하 검증**: 동시 50 사용자 모의로 응답 시작 p95 < 3s, 5xx 0건

---

## Implementation Steps (Milestones, dogfoodable 표시 포함)

각 마일스톤 끝에 **dogfoodable**: 운영자 본인이 그 시점 코드로 어디까지 직접 써볼 수 있는가.

### M0. PoC 스파이크 (3건, 1주일 내, **병렬 가능 — 3건 독립**)
*ADR Lock의 invariant — 1건이라도 실패하면 plan 재오픈*
- **M0-a**: Anthropic Kotlin SDK SSE 스트리밍 지원 확인. 미지원 시 raw HTTP + Ktor client flow fallback PoC
- **M0-b**: CMP Web (Wasm)에서 Markdown + 코드 블록 highlight + GFM 테이블 + ASCII 다이어그램 렌더 가능 라이브러리 1개 확정 (없으면 자체 minimal 파서 범위 결정)
- **M0-c**: CMP Web에서 Supabase JS SDK Wasm interop 또는 직접 OAuth redirect 흐름 PoC
- **PoC 보고서 양식**: 각 PoC 끝에 `.omc/poc/M0-{a|b|c}.md`에 (1) 결과 ✅/❌, (2) 막힘 지점, (3) 대안/대체 추천, (4) 측정한 핵심 수치(예: 첫 토큰 지연, 빌드 시간, interop 호환 여부)를 기록
- **dogfoodable**: no (검증만)

### M1. 모노레포 골격 + shared 계약 정의 (2 PR)
- **M1-PR1**: Gradle multi-module(`shared`/`backend`/`frontend`), shared/commonMain에 `@Serializable` 도메인(enum-only, 이모지 금지) + i18n 라벨 리졸버 + Wasm 호환 가드(detekt 룰)
- **M1-PR2**: backend Ktor hello world + frontend CMP Web hello world + CI(빌드/lint)
- **dogfoodable**: yes (빈 화면 띄우기까지)

### M2. Auth + DB 스키마 (2 PR)
- **M2-PR1**: Supabase 프로젝트 생성(운영자 수동), SQL migration V1 (`users`, `learning_sessions`, `cycles`, `user_api_keys`), backend `SupabaseJwtPlugin`, `/api/sessions` (GET 빈 리스트)
- **M2-PR2**: frontend OAuth redirect 흐름 (M0-c 결과 적용), 로그인 후 "내 세션" 빈 페이지 렌더
- **dogfoodable**: yes (로그인까지)

### M3. 단일 사이클 E2E (3 PR — Critic 지적 수용 분할)
- **M3a**: `backend/llm/prompts/socratic_cycle.md` (SKILL.md 6원칙/6단계 임베딩) + `ClaudeClient` non-stream 버전 + `POST /api/sessions` 동기 응답 (cycle_payload JSON만, 스트림 없음)
- **M3b**: SSE wire contract OpenAPI 정의 (`docs/sse-contract.yaml`) + `POST /api/sessions/{id}/cycles` SSE 변환 + `cycle_payload` JSON 스키마 강제(파싱 실패 시 backend가 명시 에러 이벤트 발행)
- **M3c**: frontend `CycleScreen`(SSE 스트림 렌더 + M0-b Markdown/highlight 적용 + JSON 파싱 실패 시 "AI 응답 재시도" UI)
- **dogfoodable**: yes (한 사이클 흐름 끝까지 보기, 답변/채점은 아직 없음)
- **M3a 추가 검수**: 5 cycle 실측 → 입력/출력 토큰 평균 회수. 운영 비용 산식의 가정(in 3K + out 2K)과 **2배 이상 차이 시 cap 재계산** (Critic Major #1 흡수)
- **M3 게이트 (측정 방법 명시)**: M3c 종료 시점, **운영자(=PR 작성자)**가 측정. 단위:
  - x축: commonMain에 위치한 enum/계약 정의 LoC (선언 + i18n 리졸버 + Wasm 가드 합산)
  - y축: M0-b + M3c에서 Wasm 관련 yak shaving에 쓴 누적 man-hour (커밋/PR description의 시간 로그 합산)
  - 임계: `man-hour / LoC ≥ 0.05` 이면 1:3 역전으로 본다 (200 LoC 기준 10시간 = 0.05). 역전 시 ADR 재오픈

### M4. 확인 질문 폼 + 채점 (M5와 직렬, Architect 지적 수용)
- frontend `AnswerForm`: cycle payload의 `questions[]`를 받아 문항별 텍스트박스 + "모르겠어요" 토글
- "한 번에 제출" → answers 배열 전송
- backend 채점 + 분기 옵션을 한 SSE 응답에 묶어 반환 (응답 경계: 전체 채점/분기 토큰이 들어온 후 `cycle_payload` 이벤트 단 한 번 → frontend는 그 시점에 GradingTable + BranchCards 동시 렌더). SKILL.md "응답 2 묶음" 원칙 준수
- **SSE payload 계약 (i18n 위치 명확화)**: `cycle_payload.grading[].signal`은 **enum value(`CORRECT`/`PARTIAL`/`MISCONCEPTION`)만 전송**. backend는 절대 한국어 라벨/이모지를 SSE 또는 DB에 직접 쓰지 않는다. frontend `GradingTable`이 commonMain `GradingSignal.label(locale)` 리졸버를 호출해 🟢🟡🔴 + 한국어 라벨로 변환 렌더 (Architect iter 2 #2 흡수)
- **dogfoodable**: yes (1사이클 답변 + 채점까지)

### M5. 분기 + 사이클 연쇄 (M4 직렬 후)
- backend cycle endpoint가 이전 cycles **요약 후** prompt 주입. **Principle #6 강제**: `previous_cycles_summary` 토큰 budget ≤ 2K. raw concat 금지. 직전 cycle의 grading + branch_choice + explain_md 핵심 문단만 추출
- `BranchCards`: 2-4개 카드 + 자유 입력란
- DB write: 매 SSE 종료 시점에 `cycles` insert. **부분 cycle 처리**: 클라이언트 SSE disconnect 감지 시 backend가 `status='partial'`로 row 저장 + 회고 화면에 "중단된 사이클(재시도?)" 표시. 비용 로그 기록
- **dogfoodable**: yes (3사이클 연속 학습)

### M6. 학습 이력 회고 (M5와 부분 병렬 가능 — read-side 트랙)
> M6 ↔ M7/M8 wall-clock 관계: M6 종료 시점에 운영자 dogfooding 7일 시작. M7/M8 작업은 dogfooding 7일과 **wall-clock 병렬 진행**. 단, M7 cap 정책 또는 M8 회귀 게이트 변경이 dogfooding 중인 라이브 세션에 영향(429 응답, 프롬프트 변경 등)을 주면 그 시점부터 dogfooding day 카운트 reset (Architect iter 2 #4 + Critic Major #4 흡수)
- backend: `GET /api/sessions` (최신순, 검색 ILIKE), `GET /api/sessions/{id}`
- `HistorySidebar` + `SessionDetail` + "이어서 학습" 버튼
- **회고 도달 시간 검수 케이스 자동화**: e2e 테스트로 "어제자 세션 검색→재개 3클릭 5초"를 Playwright로 측정
- **dogfoodable**: yes (전체 흐름. 이 시점부터 운영자 7일 dogfooding 시작)

### M7. Rate limit + 비용 가드 (운영자 키)
- backend `RateLimitPlugin`: user_id × day 토큰 count + user_id × minute 호출 count
- 일일 cap 25K in / 15K out, 분당 5호출 (산식 위 참고)
- **재시도 면제 정책**: SSE `cycle_payload` JSON 스키마 위반으로 인한 자동 재시도(frontend "재시도" 버튼 또는 backend 자동 1회)는 **분당 cap에서 면제, 일일 cap에는 포함**. 면제 횟수는 cycle 당 최대 2회. 그 이상은 사용자에게 명시 에러 (Architect iter 2 #3 흡수)
- 한국어 429 메시지 + frontend 토스트
- `user_api_keys` row 있으면 cap 미적용 분기 (BYOK 자리)
- **비용 폭주 자동 알람**: 일일 누적 LLM 비용 **$20 초과 시 Sentry alert + 운영자 이메일** (Critic Major #3 흡수)
- **dogfoodable**: yes (cap 정상 작동 확인)

### M8. LLM 회귀 + 부하 + 배포
- **LLM 회귀 게이트**: `tests/llm/` 5개 케이스(코루틴/Recoil/HTTPS/리액트 hook/JWT). CI에서 머지 전 자동 실행. 합격 기준:
  - (a) cycle_payload JSON 스키마 100% 통과
  - (b) 한국어 응답 비율 100% (별도 Risk row로 추적 — Risks 표 참조)
  - (c) 각 응답에 수준 추정 + 3-8개 질문 + Markdown 시각화 모두 포함
  - (d) 🟢🟡🔴 신호가 골든 정답과 ≥80% 일치 (LLM-as-judge로 보조 채점)
  - (d') **judge 메타 회귀 가드** (Critic Major #2): judge 모델 버전을 **정확한 빌드 id로 핀 고정** (`anthropic.model_id` 명시). 분기당 1회 운영자가 5/5 케이스 수동 spot-check, judge 정확도가 인간 평가와 ≥90% 일치하지 않으면 (d) 비활성화 + 인간 채점으로 전환
  - 1개라도 실패 → 머지 차단
- 부하 스크립트 `tests/load/socratic_load.k6.js`: 동시 50 vu, 1사이클씩
- Fly.io 배포(`fly.toml` + Dockerfile, env: `ANTHROPIC_API_KEY`, `SUPABASE_*`, `JWT_PUBLIC_KEY`). frontend는 Cloudflare Pages 정적. CORS는 backend Ktor `CORS` 플러그인에서 frontend origin 화이트리스트
- Sentry: backend + frontend 양쪽 SDK
- **dogfoodable**: yes (prod URL)

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **프롬프트 토큰 누적 폭주** (이전 cycles concat) | 중 | 고 | Principle #6 강제, M5에서 요약 후 ≤ 2K 토큰 cap. summarize 실패 시 explain_md 첫 200자만 |
| **LLM 출력 JSON 스키마 위반** | 중 | 고 | tool use / structured output 강제. 실패 시 frontend "재시도" UX, backend 비용 로그 |
| **SSE 중도 끊김 → 부분 cycle** | 중 | 중 | backend status='partial' row 저장, 회고에 "재시도?" 표시. 토큰 비용 회수 불가는 운영자 부담으로 인정 |
| **Supabase Auth Wasm 호환 실패** | 중 | 고 | M0-c PoC로 사전 검증. 실패 시 직접 OAuth redirect + JWT 수동 처리 |
| **CMP Wasm Markdown 라이브러리 부재** | 중 | 중 | M0-b PoC. 실패 시 minimal 자체 파서(heading/code/table/list) — 범위 lock |
| **Anthropic Kotlin SDK streaming 미지원** | 중 | 중 | M0-a PoC. 실패 시 raw HTTP fallback 일정 +1 PR |
| **운영자 LLM 비용 폭주** | 중 | 고 | 산식 기반 일일/분당/동시 cap 3종, M7에서 처음부터 적용 |
| **CMP Wasm 디버깅 거침 → dogfooding 속도 저하** | 중 | 중 | M3 게이트에서 측정, 1:3 역전 시 ADR 재오픈(swap 시나리오 보존) |
| **Markdown XSS** (LLM이 `<script>` 출력) | 낮 | 고 | raw HTML 비활성, sanitize 통과 |
| **CORS/쿠키/SameSite** | 낮 | 중 | M2에서 frontend domain 화이트리스트, Supabase Auth callback 같은 origin에 mount |
| **LLM 출력 품질 회귀** | 중 | 중 | M8 LLM 회귀 게이트 CI 차단 |
| **한국어 출력 회귀** (영어/혼용 응답) | 중 | 중 | M8 합격 기준 (b) 100%로 매 머지마다 검증. 프롬프트에 한국어 출력 명시 + 톤 예시 박음 |
| **비용 폭주 운영자 인지 지연** | 중 | 고 | M7 일일 누적 $20 알람 (Sentry + 이메일). cap 발동 전에 운영자 1차 인지 |
| **BYOK 활성 시 키 보안** | (M7+) | 고 | 활성 시점에 AES-GCM + KMS, 등록 1회 ping 검증, 약관/책임 고지 UI 별도 마일스톤 |
| **Wasm 빌드 시간이 dogfooding 사이클 저해** | 중 | 중 | 개발 시 hot reload 사용, prod 빌드만 full Wasm |

---

## Verification Steps (PR/마일스톤 단위)

1. **M0 끝**: 3개 PoC 보고서 `.omc/poc/` 에 기록. 1건 실패 시 STOP
2. **M1 끝**: `./gradlew build` 전체 통과, detekt가 `java.time` import를 commonMain에서 빌드 실패시키는지 확인
3. **M2 끝**: 미인증 cURL → 401, 인증 후 → 200 빈 배열. Supabase studio 4개 테이블 확인
4. **M3a 끝**: prompt 파일 + non-stream 호출로 cycle_payload JSON 1회 회수
5. **M3b 끝**: SSE 이벤트 sequence (`delta*` → `cycle_payload` → `done`) wireshark/curl로 확인. 스키마 위반 case 강제 발생 시 명시 에러 이벤트
6. **M3c 끝**: 브라우저 실연. **M3 게이트 측정** (라인 수 vs Wasm 시간)
7. **M4 끝**: 의도적 오답 3개 케이스로 🔴/🟡/🟢 모두 출현. enum value가 DB에 이모지로 새지 않는지 확인
8. **M5 끝**: 5 cycle 연속, DB round_no 1-5. partial cycle 강제(탭 닫기) 후 회고에 "중단됨" 표시
9. **M6 끝**: Playwright e2e — 어제자 세션 검색→재개 3클릭 5초 측정. **dogfooding 7일 시작**
10. **M7 끝**: cap 임계 부근에서 정확히 429. BYOK row 수동 삽입 시 cap 우회
11. **M8 끝**: LLM 회귀 5/5 통과, prod URL에서 부하 p95 < 3s. dogfooding 7/7일 통과 또는 5/7일 통과 + 가치 선언 서명

---

## ADR (Architecture Decision Record)

### Decision
**CMP Web (Wasm) + Ktor + Supabase + Anthropic Claude API (Kotlin SDK)** 단일 Gradle 모노레포. `shared/commonMain`은 **SSE wire contract + GradingSignal taxonomy의 단일 정의** 역할로 축소(풍부한 동작 공유 아님). M0 스파이크 3건 통과를 ADR Lock의 invariant로 둔다.

### Drivers
1. **계약 공유 가치** (SSE wire + taxonomy enum이 frontend/backend에서 일치) — 학습 도메인의 신호 일관성에 결정적
2. **사용자 Kotlin/Compose 친숙도** (deep-interview 사용자 선언 + Compose 칸반보드 작업 컨텍스트)
3. **JVM 단일 jar 배포** (50명 규모에 충분)
4. **swap cost 제한** (SSE wire contract가 OpenAPI 표현이라 backend 교체가 frontend에 영향 적음)

### Alternatives Considered (진지하게)
- **Hono + Bun + Supabase**: tRPC + zod로 wire 계약 공유 가능, 도메인 로직은 backend에만 있으므로 commonMain 가치가 zod schema 정도로 축소될 수 있음. **재반박**: 학습 도메인의 i18n/presentation 분리(GradingSignal.label)는 frontend에 위치해야 효과적이고, Kotlin 단일 언어 친숙도가 사용자에게 의미 있음. **단**, M3 게이트에서 비교 측정. 1:3 역전이면 이 옵션으로 swap
- **FastAPI + Supabase**: LLM 생태계 풍부. **재반박**: Kotlin/CMP와 거리, 컨테이너 운영 부담. swap 후보 우선순위 낮음
- **Spring Boot (Kotlin)**: Ktor와 같은 언어. **재반박**: 50명 규모에 과하고, Ktor가 coroutine-native라 SSE에 자연스러움
- **Next.js + tRPC**: dogfooding 가장 빠른 대안. **재반박**: deep-interview에서 사용자가 CMP 명시. 가치 검증 속도가 우선이면 ADR 재협상 트리거

### Why Chosen
- 위 alternatives 중 **계약 공유(좁은 의미)** + **Kotlin 친숙도** 두 driver를 동시 충족하는 유일
- M0 스파이크와 M3 게이트로 **잘못된 선택 시 빠른 회수** 경로 확보

### Consequences
- **(+)** 사이클/채점 enum 변경 시 한 곳, 자동 동기화
- **(+)** SSE wire contract OpenAPI 표현으로 backend swap이 frontend에 영향 최소
- **(-)** Anthropic 최신 기능 1-2박자 늦음 → raw HTTP fallback 일정 +1 PR 가능성
- **(-)** CMP Web Markdown/highlight 결핍 → 자체 파서 또는 라이브러리 제한, M0-b로 사전 검증
- **(-)** Wasm 디버깅 거침 → dogfooding 속도 저하 가능, M3 게이트로 모니터링
- **(-)** Supabase Auth Wasm interop 불확정 → M0-c 통과 의존
- **(-)** ADR swap 발동 시 commonMain 코드 폐기 비용: SSE wire contract 부분은 OpenAPI에서 TS/Zod로 재생성 가능, 그 외 enum/i18n 리졸버는 폐기. swap 시점이 M3 게이트면 폐기 LoC가 최소화됨

### Follow-ups
1. M0-a/b/c PoC 즉시 (M1 전)
2. M3 게이트 측정 + 의사결정
3. BYOK 활성: AES-GCM + KMS, ping 검증, 약관/책임 UI (별도 마일스톤)
4. 자료 입력 파이프라인(PR URL/파일/외부 문서) — deferred, 별도 spec
5. 영어 i18n — i18n 리졸버 이미 구조화, 데이터만 추가
6. 회고 화면 검색 강화 (full-text search) — 사용자 50명 → 수백명 시점에

---

## Open Questions (사용자 확인 필요)

1. **CMP Lock 재협상 의향**: dogfooding 속도가 가치 검증의 핵심이면 Next.js+tRPC swap을 적극 검토할 의향이 있는가? (현재 ADR은 M3 게이트에서 결정하도록 설계)
2. **이전 cycles 주입 범위**: Principle #6에서 budget 2K를 잡았는데, 학습 맥락 보존을 우선하면 4K까지 확장 의향?
3. **운영자 월 예산 상한**: 산식의 $300/mo 가정이 맞는가? 더 낮거나 높으면 cap 산식 재계산 필요

---

## Changelog
- 2026-05-20 v1: 초안
- 2026-05-20 v3: Architect+Critic iter 2 피드백 흡수 (Critic APPROVED)
  - M0 직렬→병렬 명시, PoC 보고서 양식 추가
  - M3 게이트 측정 단위/측정자/임계 구체화 (`man-hour/LoC ≥ 0.05`)
  - SSE payload 계약: backend는 enum-only 전송, i18n은 frontend 호출 시점 변환
  - M3a 끝에 토큰 실측 산식 보정 의무
  - M7: SSE 재시도의 분당 cap 면제 정책 + 일일 $20 비용 폭주 자동 알람
  - M8: LLM-as-judge 모델 버전 핀 + 인간 spot-check 가드 (d')
  - Risks 표: 한국어 출력 회귀 + 비용 폭주 인지 지연 별도 row 추가
  - M6 끝에 M6↔M7/M8 wall-clock 병렬 명시 + dogfooding day reset 정책
  - ADR Consequences: swap 시 commonMain 폐기 회수 정책 1줄
- 2026-05-20 v2: Architect+Critic iter 1 피드백 반영
  - Driver #1 "도메인 공유 광역" → "SSE wire contract + GradingSignal taxonomy 좁은 계약"
  - Principle #6 추가: prompt cost guard
  - M0 PoC 스파이크 3건 신설 (a/b/c), ADR Lock의 invariant로 박음
  - M3을 M3a/M3b/M3c로 분할
  - 운영 비용 산식 + 일일/분당/동시 cap 명시
  - 핵심 가치 정량 acceptance 4건 추가 (입력 수 50%, 회고 3클릭 5초, 7일 dogfooding, 세션 재구성 일치)
  - LLM 회귀 게이트 5케이스 + 합격 기준 4종 + CI 차단
  - GradingRow의 이모지 enum 금지 명시, presentation/taxonomy 분리, i18n 리졸버 commonMain
  - Wasm 호환 가드 detekt 룰 구체화
  - SSE 중도 끊김 → partial cycle 회수 UX
  - BYOK 활성 시 추가 작업 (AES-GCM, KMS, ping 검증, 약관 UI) ADR Consequences에 명시
  - 각 마일스톤 dogfoodable 표시
  - alternatives 진지한 재반박 + Next.js+tRPC swap 후보 명시 + M3 게이트 도입
  - Korean LLM 회귀 합격 기준 정량화
  - CORS/쿠키/SameSite 누락 보강
