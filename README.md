# socratic-learn-web

소크라테스식 점진 학습 도우미 웹 서비스. Claude Code 스킬 [`socratic-learn`](https://github.com/ujeong/ao-skills/tree/main/skills/socratic-learn)을 GUI로 옮겨, CLI 환경의 불편함(질문 번호 수기 입력, 코드 입력 어려움, 이력 회고 어려움)을 해결한다.

## 핵심 가치

- **GUI 편의성**: 확인 질문 3-8문항을 문항별 텍스트박스로 분리. 번호 매기는 수고 없이 답변
- **이력 영구 보존**: 모든 사이클(설명/질문/채점/분기)을 사용자 계정에 영구 저장, 나중에 회고 가능
- **시각화 일관성**: ASCII 다이어그램/비교표/코드블록을 CLI보다 명확하게 렌더링

## 기술 스택

| 레이어 | 선택 |
|---|---|
| Frontend | Compose Multiplatform (Web/Wasm 타겟) |
| Backend | Ktor (Kotlin) |
| LLM | Anthropic Claude API (Kotlin SDK) |
| DB | Supabase Postgres |
| Auth | Supabase Auth (Google OAuth) |
| 배포 | Fly.io 또는 Render (Ktor jar) |

## 운영 가정

- 50명 단기 동시 접속 규모
- 초기: 운영자가 LLM API key 제공, rate limit으로 보호
- 추후: BYOK (사용자가 본인 Claude API key 등록)

## 스펙

전체 요구사항/제약/검수 기준은 [`.omc/specs/deep-interview-socratic-learn-web.md`](./.omc/specs/deep-interview-socratic-learn-web.md) 참조.
