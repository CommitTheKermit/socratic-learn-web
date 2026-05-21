# Consensus Plan v2: 5시간 4인 해커톤 실행 계획 (socratic-learn-web)

> **Status:** pending approval
> **Source spec:** `.omc/specs/deep-interview-hackathon-5h-4person.md` (ambiguity 17%)
> **Mode:** RALPLAN-DR short
> **Generated:** 2026-05-21
> **Revisions:** v1 → v2 (Architect Critical 3 + Important 3 + Minor 2, Critic Critical 1 + Major 5 + Minor 3 + What's-Missing 5 흡수)

## RALPLAN-DR Summary

### Principles (5)
1. **확정 결정 재논의 금지**: spec 의 4-Role, gitflow, 지정 머저, P1 done 정의는 plan 안에서 흔들지 않는다.
2. **계약은 라이브 적응형**: Shared 가 0:05-0:45 안에 commonMain 에 P1-critical 3종만 머지. GradingSignal/persistence/wire doc 은 v2 PR (1:00 이후). 1:30-1:45 의 15분은 "라이브 SDK 응답을 본 후 계약 미세조정 budget" 로 사전 할당.
3. **PR 사이즈 < 30분 + stub 정책**: 30분 초과 시 WIP 분할. 의존 PR 은 stub 200 OK 로 분할 머지 후 점진 교체.
4. **녹색 develop 우선**: develop 빌드 깨짐 = 다음 PR 거절. 5분 안에 hotfix 또는 revert. hotfix 권한은 Shared / FE 백업 모두.
5. **데모 폴백 우선 + 인터페이스 통일**: `FakeClaudeClient` 가 라이브와 동일한 `ClaudeClient` 인터페이스 구현. fixture 는 raw chunk 아닌 `List<SseEvent>` + delay.

### Decision Drivers (Top 3)
1. **e2e 데모 동작 가능성** (4:00 시점 시연 통과가 절대 우선).
2. **병렬성 유지** (Shared 계약 1차 + BE SDK PoC 동시 진행으로 critical path 단축).
3. **충돌 비용 최소화** (수평 레이어 + 단일 머저 + FE 자동 백업).

### Viable Options 비교

| Option | 접근 | Pros | Cons | 채택 |
|---|---|---|---|---|
| A | **Shared-thin 계약 + BE-PoC 병행 + 1:30 v2 budget** (Architect 합성) | spec 정합. 계약 v2 발생을 schedule 항목으로 흡수. | Shared/BE 가 0:30 에 슬랙 미팅 30분 필요 (조율 비용) | **채택** |
| B | Stub-first (가짜 계약으로 시작 → Shared 가 따라잡음) | Shared blocking 제거 | 계약 변경 1회 시 BE+FE 동시 rework, develop 빌드 보전 어려움 | 거절 |
| C | Vertical slice 회귀 | PR 단위 작음 | spec 의 수평 레이어 결정 위반 | invalidate |
| D | Mob 프로그래밍 | 공동 이해 | 4인 동시 한 화면 = 1.5인 산출량 | invalidate |

**Option B invalidation 보강 (Critic Skeptic 지적 반영):** v1 의 invalidation 은 "계약 변경 1회 시 rework 비용 > 1h 대기" 였으나, Option A 도 1:30-1:45 의 계약 v2 가능성을 인정 → 두 option 모두 v2 발생 가능. Option A 우월성의 진짜 근거는 **v2 시점 차이**: B 는 1:00 이전에 BE/FE 가 가짜 계약 위에서 이미 작업 중이므로 v2 가 들어오면 양쪽이 동시에 다시 시작. A 는 1:30 까지 BE/FE 가 진짜 계약 위에서 동작하므로 v2 는 미세조정 (예: error event payload 보강) 수준에 그침.

---

## Requirements Summary

socratic-learn-web 의 plan v3 중 M0-M5 부분 범위를 5시간 / 4인 / 로컬 / no auth / server LLM key / single user 조건에서 **P1 데모 시나리오** 까지 구현. P2 (DB persist) 는 시간 허락 시 down-scoped 분기. 4-Role (FE/BE/Shared/QA) + gitflow + 지정 머저 (FE 자동 백업) 로 통합.

P1 데모 시나리오: 로컬 FE 에서 학습 개념 1개 입력 → 백엔드가 Claude SSE 스트리밍 → 화면에 설명+확인질문 표시 → 각 질문 텍스트박스에 답변 입력 → 제출 버튼으로 서버에 답변 payload 전송.

---

## Acceptance Criteria (테스트 가능)

### P1 (Must, 4:00 까지)
- [ ] **AC1**: `feature/shared-contract` 가 0:45 ± 5분 이전에 develop 에 머지되고, `shared/src/commonMain/kotlin/contract/` 에 **3종만 존재**: `CycleRequest`, `SseEvent` (sealed: Explanation/CheckQuestion/Done/ErrorEvent), `AnswerSubmission` (+ `Answer` inner DTO).
- [ ] **AC2**: develop 머지 직후 매번 `./gradlew build` 성공. 실패 시 5분 이내 hotfix 또는 revert (Shared 또는 FE 백업 머저 권한).
- [ ] **AC3**: 로컬 BE 가 9090 포트 (대체 9091/9092 환경변수 `BE_PORT` 로 조정 가능) 에서 기동, `POST /cycle` 이 `text/event-stream` 응답.
- [ ] **AC4**: 로컬 FE 가 8080 포트 (대체 `FE_PORT` 환경변수) 에서 기동, 입력 폼 렌더.
- [ ] **AC5**: 학습 개념 입력 + "시작" 버튼 → 5초 이내 SSE 첫 청크가 마크다운으로 렌더.
- [ ] **AC6**: 스트리밍 완료 후 확인질문 3-8문항이 별도 텍스트박스로 표시.
- [ ] **AC7**: 답변 입력 + "제출" → `POST /cycle/answers` 200 OK.
- [ ] **AC8a**: 4:00-4:15 에 **라이브 호출 1회 시연 통과**.
- [ ] **AC8b**: 4:00-4:15 에 **fixture 폴백 1회 시연 통과** (`USE_FIXTURE=1` 토글).
- [ ] **AC8c**: 두 모드 전환 30초 이내.

### P2 (Should, down-scoped)
- [ ] **AC9**: `POST /cycle` 호출 시 `cycle_id` 가 로컬 sqlite 또는 Supabase `cycles` 테이블에 row 1건 INSERT. user_id 는 hardcoded `"local-anon"`.
- [ ] ~~AC10 (FE GET 렌더)~~ → **P2 범위에서 제외** (15분 안에 의미 있는 산출 불가, Critic C1/m1 반영). FE 는 호출만 시도하지 않고 BE 가 단독으로 INSERT 검증.

### Quality (Should)
- [ ] **AC11**: main 직푸시 0회 (데모 직전 main 머지 1회 예외).
- [ ] **AC12**: 4:45 이전 데모 리허설 1회 이상.
- [ ] **AC13**: AC8a + AC8b 둘 다 통과 (fixture 와 라이브가 동일 `ClaudeClient` 인터페이스를 거치므로 SSE event sequence 동일성 보장).

---

## Implementation Steps

### Phase 0 (Pre-hackathon, 사전 30분) - **Hard-gate**
- **PoC G0 (필수, 0:00 직전 30분)**: wasmJs target 에서 `ktor-client` SSE 또는 fetch streaming reader 중 어느 쪽이 동작하는지 확인. 미동작 시 NDJSON 폴백 채택. 결과를 `docs/wasm-streaming-decision.md` 1줄 기록.
  - 통과 기준: wasmJs 콘솔에 라인 1건 emit 확인.
  - **미통과 시**: BE `/cycle` 응답을 NDJSON (`application/x-ndjson`) 로 즉시 전환. FE 는 fetch streaming reader 로 라인 단위 파싱. Owner: FE (Phase 0 안에), BE 는 Phase 2 시작 전 통보 받음.
- 4인 환경 사전 확인: JDK 21+, Gradle 8.x, Node.js 20+, `ANTHROPIC_API_KEY` 환경변수 발급/공유.
- repo 의 `main` / `develop` 브랜치 생성 + `main` branch protection (direct push 차단). 미수행 시 Phase 1 첫 5분에 즉시 처리.

### Phase 1: 0:00 - 0:05 킥오프 (5분)
- 4-Role 재확인, develop 체크아웃.
- main protection 사전 미수행 시 Shared 즉시 처리.
- BE/Shared 가 **0:30 슬랙 미팅** 일정 합의 (5분 짜리).

### Phase 2: 0:05 - 0:45 Shared 계약 1차 (40분, hard-cut)
**Shared 단일 작업 (0:05 - 0:45):**
- `feature/shared-contract` 브랜치 생성.
- `shared/build.gradle.kts` 작성. **commonMain 의 kotlinx.serialization wasmJs+jvm 호환 버전 명시** (1.7.x 이상).
- `shared/src/commonMain/kotlin/contract/` 에 **3종만**:
  - `CycleRequest.kt`: `data class CycleRequest(val concept: String, val userId: String = "local-anon")`
  - `SseEvent.kt`: sealed interface + 4 variant (`Explanation(markdownChunk)`, `CheckQuestion(index, text)`, `Done(cycleId, questionCount)`, `ErrorEvent(code, message)`)
  - `AnswerSubmission.kt`: `data class AnswerSubmission(cycleId, answers: List<Answer>)`, `Answer(questionIndex, text)`
- **명시적 v2 분리 (0:45 이후 PR)**: `GradingSignal` enum / `AnswerSubmissionResponse` / `SseProtocol` 인코딩 헬퍼 / `docs/wire-contract.md` (간단 markdown, OpenAPI 형식 아님).
- PR 올림 → Shared 본인 머지.
- **AC1 Gate**: 0:45 ± 5분 develop 머지 완료.

**병렬 작업 (FE/BE/QA, 0:05 - 0:45):**
- **FE**: `frontend/build.gradle.kts` (CMP Wasm target). 8080 빈 페이지 렌더. `feature/fe-skeleton` PR.
- **BE**: `backend/build.gradle.kts` (Ktor + Anthropic Kotlin SDK). 9090 `/health` endpoint. `feature/be-skeleton` PR.
- **BE 추가 (0:30 슬랙 미팅 산출물용)**: 0:05-0:30 안에 **로컬에서 Anthropic SDK 1회 호출** (시스템 프롬프트 + 학습 개념 1건). 응답의 chunk shape / question 추출 가능 여부 확인.
- **0:30 슬랙 미팅 (BE+Shared, 5분)**: BE 의 라이브 응답 결과를 Shared 에게 공유. Shared 가 `SseEvent` 의 sealed variant 4개를 현실 응답에 맞게 미세조정.
- **QA**:
  - `docs/setup-local.md` 작성 (포트 가변, 환경변수, `USE_FIXTURE=1` 토글).
  - **0:30 hard-gate (Critic M5)**: `scripts/dev-watch.sh` 작성 (`fswatch` or `inotifywait` 으로 develop 변경 감지 → `./gradlew build` 자동 실행 → 빌드 결과 텔레그램/디스코드 webhook 또는 로컬 알림).
  - fixture 시드 1-2건 작성: `backend/src/main/resources/fixtures/cycle-coroutine.json` 은 raw chunk 가 아니라 **`List<SseEvent>` JSON + delayMs** 시퀀스.

### Phase 3: 0:45 - 3:30 P1 핵심 구현 (165분)

**BE (`feature/be-cycle-*`) - stub 정책 명시:**
1. `feature/be-cycle-routes`: `backend/src/main/kotlin/cycle/CycleRoutes.kt` + `AnswerRoutes.kt`. **stub return** (200 OK + 빈 SSE 또는 hardcoded 1 event). develop 빌드 깨짐 없이 머지 가능.
2. `feature/be-claude-client`: `ClaudeClient.kt` (인터페이스 + LiveClaudeClient + `FakeClaudeClient`). `FakeClaudeClient` 는 fixture JSON 의 `List<SseEvent>` + delayMs 를 emit. **단위 테스트 1건은 fixture 시퀀스를 동일 SseEmitter path 통과시켜 wire 출력 shape 가 라이브와 일치하는지 검증** (shape drift 자동 감지).
3. `feature/be-sse-emitter`: `SseEmitter.kt` + `PromptBuilder.kt`. routes 가 stub 대신 `ClaudeClient` 인터페이스 호출 → SseEmitter 가 SseEvent 시퀀스를 SSE wire 로 매핑. **fixture/live 양쪽이 동일 emitter path 통과**.
4. `feature/be-answer-route`: `POST /cycle/answers` 는 in-memory ConcurrentHashMap 저장.
- **429 / quota 분기 (Critic What's-Missing 1)**: `LiveClaudeClient` 가 429 catch → `ErrorEvent("RATE_LIMIT", message)` 로 emit. FE 는 이 이벤트 받으면 사용자에게 "API 한도 - fixture 모드 권장" alert.
- **PR<30분 위반 예외**: SseEmitter+ClaudeClient 통합 PR 은 60분까지 허용 (Principle 3 의 stub 정책 항).

**FE (`feature/fe-*`):**
1. `feature/fe-form` (0:45-1:15): `ConceptInputScreen.kt` (개념 입력 + "시작" 버튼). BE skeleton 의 `/health` 호출만으로 우선 동작 확인.
2. `feature/fe-stream` (1:15-2:30): `CycleClient.kt` (ktor-client wasmJs SSE 또는 fetch streaming - Phase 0 결정 따름). `StreamingScreen.kt` (Markdown 청크 누적 렌더). **CMP Markdown 라이브러리 미존재 시** 텍스트 청크 누적 + 코드블록/h1 정규식 처리만.
3. `feature/fe-answers` (2:30-3:15): `AnswerScreen.kt` (확인질문 텍스트박스 그리드 + 제출 → `POST /cycle/answers`).
4. `feature/fe-viewmodel` (3:15-3:30): `CycleViewModel.kt` (화면 전이 input → streaming → answer).

**Shared (`feature/shared-*`, 0:45 이후 머저 role 위주):**
- **PR 큐 관리 hard rule (Critic M1)**: 머지 큐 깊이 ≥ 2 시 FE 가 자동 백업 머저로 동작. 0:45 이후 적용. 단 본인 PR (FE feature/fe-*) 은 Shared 가 머지. **edge case: Shared 부재 + FE 본인 PR 동시 발생 시 BE 가 2차 백업 머저 (read-only review + force-merge OK)**.
- **계약 v2 budget (1:30-1:45, 15분 사전 할당)**: BE 의 첫 SSE 가 동작한 시점에 Shared 가 계약 미세조정 PR 1건 (필요 시). 예: `ErrorEvent` payload 보강, `Done` 의 questionCount semantic. **미사용 시 회수 정책: Shared 가 1:45 즉시 PR 머지 큐 처리 + FE 답변 화면 보조 작업으로 자연 흡수**.
- 시간 남으면 `feature/shared-persistence-dto` (`CycleRecord` 등) 작성, P2 대비.

**QA:**
- 0:30 부터 `scripts/dev-watch.sh` 백그라운드 실행. 빌드 깨짐 자동 알림.
- 1:30 부터 BE `curl -N -X POST http://localhost:9090/cycle -d '{"concept":"코루틴"}'` 수동 SSE 검증.
- 2:30 부터 FE-BE 통합 시연 시도.
- 3:00 부터 e2e 수동 반복 5회.

### Phase 4: 3:30 - 4:00 통합 정리 (30분)
- 마지막 PR 머지 완료.
- QA 가 fixture 토글 켜고 1회, 끄고 라이브 1회 시연 검증.
- hotfix PR (< 10분).

### Phase 5: 4:00 - 4:30 P1 시연 게이트 + P2/폴리시 분기 (30분, Critic C1 분할)
- **4:00 - 4:15: P1 통합 시연 (Hard-gate, AC8a/b/c)**:
  - 라이브 시연 1회 → fixture 폴백 1회 → 전환 30초 이내 확인.
  - 통과 시 P2 분기 진입 결정.
  - 미통과 시 P2 포기 + 시연 시나리오 안정화 (전 인원 hotfix 모드).
- **4:15 - 4:30: hotfix 또는 폴리시**:
  - 통과 분기: P2 (BE 가 schema + INSERT 만) 진입.
  - 미통과 분기: 시연 안정성 강화 (FE UI 텍스트, 에러 메시지).

### Phase 6: 4:30 - 4:45 P2 시도 또는 폴리시 (15분, down-scoped)
**P2 분기 (down-scoped):**
- BE 가 `cycles` 테이블 schema (cycle_id, user_id, concept, created_at) + `/cycle` 호출 시 INSERT 1건만.
- **AC10 (FE GET 렌더) 은 P2 범위에서 삭제** (Critic m1).
- 로컬 sqlite 우선 (Supabase 셋업 5분 절약).

**폴리시 분기:**
- 발표 시 사용할 학습 개념 1-2개 선정 (코루틴, async/await 등).
- FE 시연 텍스트 가독성 (폰트, 줄간격).

### Phase 7: 4:45 - 5:00 데모 리허설 (15분)
- 라이브 1회 + fixture 1회.
- 발표 스크립트: 30초 컨텍스트 → 1분 시연 → 30초 다음 단계.

**Wall-clock 합산 검증**: 5 + 40 + 165 + 30 + 30 + 15 + 15 = **300분 = 5h** ✓

---

## Branch / PR 운영 절차

### Branch 모델
```
main (protected, demo cut only)
 └─ develop (integration, validated)
     ├─ feature/shared-contract       (Shared, 0:05-0:45)
     ├─ feature/shared-contract-v2    (Shared, 1:30-1:45 budget)
     ├─ feature/fe-skeleton           (FE, 0:05-0:30)
     ├─ feature/be-skeleton           (BE, 0:05-0:30)
     ├─ feature/be-cycle-routes       (BE, stub-first, 0:45-1:15)
     ├─ feature/be-claude-client      (BE, 1:15-1:45)
     ├─ feature/be-sse-emitter        (BE, 1:45-2:45, 60min 예외)
     ├─ feature/be-answer-route       (BE, 2:45-3:15)
     ├─ feature/fe-form               (FE, 0:45-1:15)
     ├─ feature/fe-stream             (FE, 1:15-2:30)
     ├─ feature/fe-answers            (FE, 2:30-3:15)
     ├─ feature/fe-viewmodel          (FE, 3:15-3:30)
     └─ feature/qa-fixtures           (QA, 0:30-1:30)
```

### PR 규칙
1. **사이즈**: < 30분. 예외 BE SseEmitter 통합 (60분 까지 허용, stub→실구현 점진 교체 명시 시).
2. **타이틀**: `[layer] short description`.
3. **본문**: 변경 파일 + 의도 1줄.
4. **리뷰**: Shared (지정 머저) 가 5분 이내 머지. 머지 큐 ≥ 2 시 FE 자동 백업 머저.
5. **머지 방식**: squash merge.
6. **머지 후**: 머저가 develop pull + `./gradlew build` 또는 QA 의 `dev-watch.sh` 자동 검증.

### Develop 검증 절차
- QA 의 `scripts/dev-watch.sh` 가 0:30 부터 작동.
- 빌드 깨짐 알림 → 5분 이내 hotfix (Shared 또는 FE 백업 권한) 또는 `git revert <merge_commit>`.
- hotfix vs revert 판단: 원인 명확하고 10줄 이내 수정이면 hotfix. 아니면 즉시 revert.

### 충돌 처리
- 동일 파일 충돌: Shared 가 resolve (자리비움 시 FE 백업).
- 계약 v2 충돌: Shared 의 `feature/shared-contract-v2` 가 1:30-1:45 budget 안에서 처리. BE/FE 는 rebase 권고.

### 포트 충돌 (Critic What's-Missing 4)
- 각자 로컬에서 띄울 때 충돌 가능. `BE_PORT` / `FE_PORT` 환경변수로 인당 별도 포트 배정. `docs/setup-local.md` 에 인당 포트 표 명시 (예: FE=8080+rank, BE=9090+rank).

---

## Demo 폴백 시나리오

### 폴백 1: LLM 호출 실패 (가장 가능성 큼)
- 트리거: API 다운 / 429 / 네트워크 / key 오류
- 조치: `USE_FIXTURE=1` 토글 → `FakeClaudeClient` 가 fixture `List<SseEvent>` + delayMs 시퀀스 emit. **fixture 도 동일 SseEmitter path 통과** → 라이브와 SSE wire 동일성 보장 (Critic M3).
- QA 가 4:00 이전에 fixture 경로 사전 검증.

### 폴백 2: FE 빌드 실패
- 트리거: Wasm 빌드 깨짐 / CMP Markdown 호환
- 조치: BE 의 `GET /cycle/debug` HTML 페이지 (서버사이드 단일 페이지) 로 시연.

### 폴백 3: wasmJs SSE 미동작 (NDJSON 폴백)
- 트리거: ktor-client wasm engine 의 EventSource 미동작
- 조치: BE 응답 헤더 `Accept: application/x-ndjson` 분기 + FE fetch streaming reader 로 라인 처리.
- **Owner: FE (Phase 0 PoC G0 에서 사전 검증, 미통과 시 즉시 적용)**.

### 폴백 4: 시연 인터넷 끊김
- 4:30 까지 노트북 핫스팟 1대 준비. fixture 모드로 오프라인 시연.

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Shared 계약 0:45 까지 미완 | 중 → 낮 | 치명적 | 범위 3종 hard-cut. v2 항목은 1:30-1:45 budget 으로 사전 분리. |
| Anthropic API 라이브 실패 | 중 | 치명적 | FakeClaudeClient 인터페이스 통일. 429 분기 ErrorEvent. |
| CMP Wasm Markdown 미지원 | 중 | 중 | 텍스트 청크 + 정규식 처리 폴백. |
| wasmJs SSE engine 미성숙 | 중 | 큼 | **Phase 0 PoC G0 hard-gate. NDJSON 폴백 owner=FE 명시**. |
| FE-BE 의존성 lock | 중 | 큼 | 양쪽 모두 fixture/stub 으로 단독 테스트 가능. |
| 단일 머저 병목 (~80분 누적) | 중 | 큼 | **머지 큐 ≥ 2 시 FE 자동 백업 머저 hard rule (0:45 이후)**. |
| develop 충돌 다발 | 중 | 중 | 수평 레이어 + Shared resolve 우선. |
| QA 부담 (build watch + e2e + fixture) | 중 → 낮 | 큼 | **dev-watch.sh 자동화 0:30 hard-gate**. QA 의 hands-on 시간 확보. |
| 데모 발표 중 라이브 실패 | 중 | 중 | fixture 워밍업 1회 + 라이브 1회 발표 직전 리허설. |
| **계약 v2 PR 충돌 비용** | 중 | 중 | **1:30-1:45 사전 budget 으로 흡수. v2 발생 시 BE/FE 즉시 rebase.** |
| **포트 충돌 (4인 동일 머신 X, 그래도 동일 docker compose 가능)** | 낮 | 낮 | **`BE_PORT`/`FE_PORT` 환경변수 + 인당 포트 표**. |
| **wall-clock 부족 (Phase 6 P2 15분)** | 중 | 낮 (P2 는 Should) | **P2 down-scope (BE INSERT 만, AC10 삭제)**. |

---

## Verification Steps

### 0:00 (Pre-hackathon)
- [ ] **PoC G0 통과** (wasmJs streaming 결정 + `docs/wasm-streaming-decision.md`)
- [ ] main branch protection 설정
- [ ] 4인 환경 셋업 완료

### 0:30
- [ ] BE/Shared 슬랙 미팅 (5분) 완료, 라이브 응답 shape 공유
- [ ] QA 의 `scripts/dev-watch.sh` 작동 시작
- [ ] FE/BE skeleton PR 머지 후 빌드 green

### 0:45 (AC1 Gate)
- [ ] `shared/.../contract/{CycleRequest,SseEvent,AnswerSubmission}.kt` 3종만 존재 + 컴파일
- [ ] BE/FE 의 shared import 빌드 성공

### 1:45 (계약 v2 budget 종료)
- [ ] 필요 시 `feature/shared-contract-v2` 머지 완료 OR 변경 불필요 확인

### 2:30
- [ ] BE curl SSE 수동 호출 → chunk 수신
- [ ] FE 페이지 1개 이상 동작

### 3:30
- [ ] FE 에서 입력 → 스트리밍 → 답변 제출 1회 통과
- [ ] develop green

### 4:00-4:15 (AC8 Hard-gate)
- [ ] AC8a 라이브 시연 통과
- [ ] AC8b fixture 시연 통과
- [ ] AC8c 모드 전환 30초 이내

### 4:45
- [ ] 발표 스크립트 1회 완주
- [ ] 사용자 입력 오류 (빈 문자열, 매우 긴 문자열) 1회 시도 → 죽지 않음

---

## ADR

### Decision
**Option A (Shared-thin 계약 + BE-PoC 병행 + 1:30 v2 budget + FE 자동 백업 머저)** 를 채택.

### Drivers
1. e2e 데모 동작 가능성
2. 4인 병렬성 유지
3. 충돌 비용 최소화

### Alternatives considered
- **Option B (Stub-first)**: v2 발생 시 BE+FE 가 가짜 계약 위에서 더 일찍 작업 중이라 rework 비용이 Option A 보다 큼.
- **Option C (Vertical slice)**: spec 위반.
- **Option D (Mob)**: 4인 동시 1.5인 산출량.

### Why chosen
- spec 정합. 재논의 비용 0.
- Shared 의 단일 실패점을 3종 hard-cut + 0:45 게이트로 분리.
- **v2 시점 차이**가 Option A 우월 (1:30 의 미세조정 vs B 의 초기 가짜 계약 위 rework).
- 단일 머저 직렬화 비용은 FE 자동 백업 hard rule 로 흡수.

### Consequences
- **+** Shared 가 0:45 게이트만 통과하면 이후 2.75h 가 unblock.
- **+** Fixture 와 라이브가 동일 emitter path → 데모 폴백이 진정한 폴백.
- **-** BE 가 0:05-0:30 안에 SDK 라이브 호출 1회 부담.
- **-** Shared 가 머저 + 본인 코드 + v2 budget 의 3중 책임.
- **-** P2 가 down-scope (FE 이력 렌더 제거) → spec 의 "조회 가능" 보다 weak.

### Follow-ups
- 해커톤 종료 후 plan v3 로 회귀 (M2 Auth 부터).
- SSE 계약의 약점 (error event 분류, NDJSON 폴백) 을 plan v3 ADR followups 에 합류.
- LLM 응답 비결정성 회귀 → plan v3 M8 골든 케이스에 해커톤 fixture 추가.
- FE GET 이력 렌더는 hackathon+1 마일스톤으로 이관.

---

## Changelog

- **v1 (2026-05-21 03:30)**: Planner 초안.
- **v2 (2026-05-21 04:00)**: Architect REVISE + Critic REVISE patch 흡수.
  - **흡수된 patches:**
    - Architect Critical 1 (Shared 1h 과적) → Phase 2 를 0:05-0:45 hard-cut, 계약 3종만.
    - Architect Critical 2 / Critic M1 (단일 머저 병목) → FE 자동 백업 머저 hard rule.
    - Architect Critical 3 / Critic C3 (wasmJs SSE PoC) → Phase 0 PoC G0 hard-gate, NDJSON 폴백 owner=FE.
    - Architect Important 4 / Critic M2 (BE PR<30분 vs 통합) → stub-first 정책 명시, SseEmitter 60분 예외.
    - Architect Important 5 / Critic M3 (Fixture 동일성) → FakeClaudeClient 인터페이스 통일, `List<SseEvent>` + delayMs.
    - Architect Important 6 / Critic M5 (QA 부담) → `scripts/dev-watch.sh` 0:30 hard-gate.
    - Architect Minor 7 / Critic C1 + m1 (P2 15분) → Phase 5 분할 + AC10 삭제 + P2 BE INSERT 만.
    - Architect Minor 8 / Critic m2 (v2 PR risk) → Risk 표 row 추가 + 1:30-1:45 budget 명시.
    - Critic C1 (wall-clock) → 합산 검증 + Phase 5 분할.
    - Critic M4 (AC8 측정성) → AC8a/AC8b/AC8c 분리.
    - Critic m3 (Phase 1 protection) → Phase 1 본문에 명시.
    - Critic What's-Missing 1 (429) → BE ErrorEvent RATE_LIMIT 분기.
    - Critic What's-Missing 2 (NDJSON owner) → Phase 0 PoC G0 의 Owner=FE.
    - Critic What's-Missing 3 (hotfix 권한) → Shared + FE 백업 모두.
    - Critic What's-Missing 4 (포트 충돌) → `BE_PORT`/`FE_PORT` 환경변수.
    - Critic What's-Missing 5 (FE GET 렌더) → AC10 삭제 + Follow-ups 이관.
    - Critic Skeptic (Option B invalidation self-undermining) → v2 시점 차이 논거 보강.

- **v3 (2026-05-21 04:15)**: Architect iter 2 APPROVED + Critic iter 2 APPROVED-WITH-PATCHES.
  - **v3 patches (3건 흡수):**
    - Critic Minor 1 / Architect Synthesis: Shared 부재 + FE 본인 PR 동시 발생 시 BE 가 2차 백업 머저.
    - Critic Minor 2 / Architect NB1: v2 budget 미사용 회수 정책 (1:45 시 머지 큐 + FE 답변 화면 흡수).
    - Critic Minor 3: FakeClaudeClient 단위 테스트가 동일 emitter path 통과 검증 (shape drift 감지).

> **최종 상태: pending approval (consensus 완료, 사용자 실행 승인 대기)**
