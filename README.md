# Harness

Claude Code를 다단계 워크플로우로 구동해, 프로젝트를 **스텝 단위로 구현·검증**하는 하네스 템플릿.

기획·설계를 문서로 고정하면, 실행은 `execute.py`가 헤드리스 Claude 세션으로 자동화한다. 매 스텝에 프로젝트 규칙과 설계 문서를 가드레일로 주입하고, 완료된 스텝의 요약을 다음 스텝에 누적 전달하며, 실패 시 스스로 교정한다.

## 디렉토리 구조

```
.
├── CLAUDE.md              # 프로젝트 규칙 (기술 스택 · 아키텍처 · CRITICAL 규칙 · 명령어)
├── docs/
│   ├── PRD.md             # 제품 요구사항 (목표 · 사용자 · 핵심 기능 · MVP 제외)
│   ├── ARCHITECTURE.md    # 디렉토리 구조 · 패턴 · 데이터 흐름 · 상태 관리
│   ├── ADR.md             # 아키텍처 결정 기록 (결정 · 이유 · 트레이드오프)
│   └── UI_GUIDE.md        # 디자인 가이드 (색상 · 컴포넌트 · AI 슬롭 안티패턴)
├── .claude/
│   ├── commands/
│   │   ├── harness.md     # /harness — 스텝 설계 워크플로우
│   │   └── review.md      # /review  — 변경사항 리뷰
│   └── settings.json      # 훅 (Stop: lint/build/test, PreToolUse: 위험 명령 차단)
├── scripts/
│   ├── execute.py         # 스텝 순차 실행기 (가드레일 주입 · 자가 교정 하네스)
│   └── test_execute.py    # execute.py 테스트
└── phases/                # (런타임 생성) task · step 정의
```

## 워크플로우

### 1. 문서 채우기

`CLAUDE.md`와 `docs/*.md`의 `{플레이스홀더}`를 프로젝트에 맞게 채운다. 이 문서들이 매 스텝의 **가드레일**이 되므로, 아키텍처 규칙과 CRITICAL 제약을 구체적으로 적을수록 실행 품질이 올라간다.

### 2. 스텝 설계 — `/harness`

Claude Code에서 `/harness`를 실행하면 docs를 읽고 → 필요한 결정을 논의하고 → task를 여러 step으로 쪼갠 계획을 제안한다. 승인하면 아래를 생성한다:

- `phases/index.json` — 전체 task 현황 인덱스
- `phases/{task}/index.json` — task별 step 목록과 상태
- `phases/{task}/step{N}.md` — 각 step의 지시 · Acceptance Criteria · 검증 절차

설계 원칙: 한 step은 한 모듈만 다루고(scope 최소화), 각 step 파일은 독립 세션에서 실행되도록 자기완결적으로 작성하며, AC는 `npm run build && npm test` 같은 실행 가능한 커맨드로 명시한다.

### 3. 실행 — `execute.py`

```bash
python3 scripts/execute.py {task-name}          # 순차 실행
python3 scripts/execute.py {task-name} --push    # 실행 후 origin에 push
```

`execute.py`가 자동으로 처리하는 것:

| 기능 | 설명 |
|------|------|
| 브랜치 | `feat-{task}` 브랜치 자동 생성 / checkout |
| 가드레일 주입 | `CLAUDE.md` + `docs/*.md`를 매 step 프롬프트에 포함 |
| 컨텍스트 누적 | 완료된 step의 `summary`를 다음 step 프롬프트에 전달 |
| 자가 교정 | 실패 시 최대 3회 재시도, 직전 에러 메시지를 프롬프트에 피드백 |
| 2단계 커밋 | 코드 변경(`feat`)과 메타데이터(`chore`)를 분리 커밋 |
| 상태 기록 | `started_at` · `completed_at` · `failed_at` · `blocked_at` 자동 기록 |

## 상태 & 에러 복구

각 step의 `status`는 `pending` | `completed` | `error` | `blocked` 중 하나다.

- **error**: 원인을 수정한 뒤 해당 step의 `status`를 `pending`으로 바꾸고 `error_message`를 삭제한 후 재실행.
- **blocked**: `blocked_reason`에 적힌 사유(API 키, 외부 인증, 수동 설정 등)를 해결한 뒤 `pending`으로 되돌리고 `blocked_reason`을 삭제한 후 재실행.

## 리뷰 — `/review`

변경사항을 `CLAUDE.md` · `ARCHITECTURE.md` · `ADR.md` 기준으로 검증한다. 아키텍처 준수 · 기술 스택 준수 · 테스트 존재 · CRITICAL 규칙 · 빌드 가능 여부를 체크리스트 표로 출력한다.

## 요구사항

- `claude` CLI — 헤드리스 실행(`claude -p`)에 사용
- Python 3 — 표준 라이브러리만 사용 (외부 의존성 없음)
- `pytest` — 하네스 테스트 실행용

## 테스트

```bash
pytest scripts/test_execute.py
```
