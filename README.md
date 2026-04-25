# Claude Code 대화 요약 시스템

기본 `/compact`보다 구조화된 방식으로 세션 컨텍스트를 보존하고, 다음 세션에서 슬래시 명령 한 줄로 자동 이어가는 시스템.

## 구성

- **`/summarize`** — 현재 대화를 JSON 스키마로 요약 저장
- **`/catchup`** — 가장 최근 요약을 자동으로 불러와 작업 이어가기
- **`/summaries`** — 저장된 요약 목록 보기

> `/catchup`은 Claude Code 번들 커맨드(`/resume`, `/continue` 등)와 충돌하지 않도록 의도적으로 선택한 이름입니다.

## 설치

프로젝트 루트로 이동한 다음, 아래 방법 중 하나를 선택합니다.

### 방법 1: Claude Code에게 시키기 (가장 쉬움)

Claude Code 세션에서 아래 프롬프트를 붙여넣으면 됩니다. 기존 `.claude/` 유무에 상관없이 안전하게 머지됩니다 (같은 이름의 파일이 있으면 덮어쓰기 전에 사용자에게 확인을 받습니다).

> ```
> https://github.com/EJCHO-salary/claude-summary-system 의 .claude/ 디렉토리를 현재 프로젝트에 설치해줘.
>
> 절차:
> 1. 임시 디렉토리에 shallow clone (--depth 1)
> 2. 현재 프로젝트의 .claude/ 존재 여부 확인
>    - 없으면: 그대로 복사
>    - 있으면: commands/, schemas/, summaries/ 하위만 머지하고, 동명 파일은 덮어쓰기 전에 diff를 보여주고 확인
> 3. 임시 디렉토리 삭제
> 4. 결과 트리 출력 (.claude/ 하위)
>
> 단, .claude/summaries/ 안에 이미 사용자 데이터(summary-*.json)가 있으면 절대 건드리지 말 것.
> ```

### 방법 2: 셸 명령어 (수동)

**`.claude/`가 없는 경우** — 한 줄로 설치:

```bash
git clone --depth 1 https://github.com/EJCHO-salary/claude-summary-system.git /tmp/css \
  && cp -R /tmp/css/.claude . \
  && rm -rf /tmp/css \
  && echo "✅ 설치 완료" && ls .claude
```

**`.claude/`가 이미 있는 경우** — 기존 파일을 보존하면서 머지 (`-n`은 덮어쓰기 방지):

```bash
git clone --depth 1 https://github.com/EJCHO-salary/claude-summary-system.git /tmp/css \
  && mkdir -p .claude/commands .claude/schemas .claude/summaries \
  && cp -Rn /tmp/css/.claude/commands/. .claude/commands/ \
  && cp -Rn /tmp/css/.claude/schemas/. .claude/schemas/ \
  && rm -rf /tmp/css \
  && echo "✅ 머지 완료 (기존 파일은 그대로 유지됨)" && ls .claude/commands .claude/schemas
```

> ⚠️ `-n` 옵션 때문에 같은 이름의 파일이 이미 있으면 **건너뜁니다**. 강제로 덮어쓰려면 `-n`을 제거하세요. `summaries/`는 사용자 데이터이므로 머지에서 제외했습니다.

### 방법 3: 빈 프로젝트라면 직접 클론

```bash
git clone https://github.com/EJCHO-salary/claude-summary-system.git .
```

### 설치 후 트리

```
your-project/
└── .claude/
    ├── commands/
    │   ├── summarize.md
    │   ├── catchup.md
    │   └── summaries.md
    ├── schemas/
    │   └── conversation-summary.schema.json
    └── summaries/   ← /summarize 실행 시 자동 채워짐
```

### 전역(모든 프로젝트) 설치

전역으로 쓰려면 `commands/`만 `~/.claude/commands/`로 옮기되, `schemas/`와 `summaries/`는 프로젝트별로 두는 것을 권장합니다 (요약 내용은 프로젝트 컨텍스트와 묶여있어야 의미가 있음).

```bash
mkdir -p ~/.claude/commands \
  && git clone --depth 1 https://github.com/EJCHO-salary/claude-summary-system.git /tmp/css \
  && cp -n /tmp/css/.claude/commands/*.md ~/.claude/commands/ \
  && rm -rf /tmp/css
```

전역 설치 후 각 프로젝트에서는 스키마만 별도로 배치합니다:

```bash
mkdir -p .claude/schemas \
  && curl -fsSL https://raw.githubusercontent.com/EJCHO-salary/claude-summary-system/main/.claude/schemas/conversation-summary.schema.json \
     -o .claude/schemas/conversation-summary.schema.json
```

## 사용법

### 1. 작업 종료 시 — 요약 저장

```
/summarize
```

→ `.claude/summaries/summary-20260425-143022.json` 자동 생성

이름 지정도 가능:

```
/summarize auth-refactor-checkpoint
```

→ `.claude/summaries/summary-auth-refactor-checkpoint.json`

> ⚠️ 파일명은 반드시 `summary-`로 시작해야 `/catchup`이 인식합니다. `/summarize`가 자동으로 prefix를 붙여줍니다.

### 2. 새 세션 시작 시 — 자동 복원

```
/catchup
```

→ 가장 최근 요약을 자동으로 찾아 컨텍스트 복원, `next_actions[0]`부터 작업 시작

특정 작업으로 이동:

```
/catchup auth
```

→ 파일명에 `auth`가 포함된 가장 최근 요약을 사용

특정 파일 직접 지정:

```
/catchup summary-20260424-143022.json
```

### 3. 어느 요약을 쓸지 모를 때

```
/summaries
```

→ 최근 10개 요약을 표로 보여줌. 보고 나서 `/catchup`으로 진입.

## 일반적인 워크플로우

```
[월요일 오후, 작업 마무리]
  /summarize
  ✅ 요약 저장: .claude/summaries/summary-20260425-180000.json
  📌 다음 작업: refresh token 회전 로직 구현 (auth.ts:120)

[화요일 오전, 새 세션]
  /catchup
  📂 복원: summary-20260425-180000.json
  🎯 목표: 인증 시스템 리팩토링
  📍 단계: implementing
  ▶️  다음 작업:
     1. [must] refresh token 회전 로직 구현
        → src/auth.ts:120
  ...작업 즉시 재개
```

## `/compact`와의 차이

| 항목 | `/compact` (번들) | 이 시스템 |
|------|-------------------|-----------|
| 형식 | 자연어 산문 | 구조화된 JSON |
| 다음 작업 명시 | ✗ | ✓ (`next_actions.entry_point`) |
| 시도한 해결책 보존 | ✗ | ✓ (`issues.attempted_fixes` — 같은 삽질 방지) |
| 결정 이력 | 흐려짐 | ✓ (`decisions.alternatives_rejected`) |
| 다음 세션으로 이전 | 같은 세션 내에서만 | ✓ (파일로 저장, 영구) |
| 자동 복원 | ✗ | ✓ (`/catchup`) |
| 여러 체크포인트 | ✗ | ✓ (브랜치별/작업별) |
| 기계 판독 가능 | ✗ | ✓ (스크립트로 분석/통합 가능) |

## 스키마 커스터마이징

`.claude/schemas/conversation-summary.schema.json`을 프로젝트 특성에 맞게 수정할 수 있습니다.

예시:
- 인프라 작업 → `state`에 `deployed_environments` 추가
- 데이터 사이언스 → `artifacts`에 `experiment_metrics` 추가
- 프론트엔드 → `state.key_files`에 `component_tree` 추가

수정 후 `/summarize`는 자동으로 새 스키마를 따라갑니다 (커맨드가 매번 스키마를 읽음).

## 팁

- **PR 첨부**: 요약 파일을 git에 커밋하면 PR 리뷰어가 작업 맥락을 한눈에 볼 수 있음
- **팀 공유**: `.claude/` 전체를 커밋하면 팀원들이 같은 커맨드/스키마를 공유
- **민감 정보 주의**: API 키, 비밀번호 같은 게 `artifacts.snippets`에 들어가지 않게 주의
- **요약 빈도**: 작업 흐름이 크게 바뀌는 시점(기능 완료, 디버깅 시작 등)마다 권장
- **오래된 요약 정리**: 한 달에 한 번 정도 `.claude/summaries/`의 오래된 파일 정리

## 트러블슈팅

**`/catchup` 실행 시 "이전 요약이 없습니다"**
→ `.claude/summaries/` 디렉토리가 없거나 `summary-*.json` 패턴 파일이 없음. 먼저 `/summarize` 실행 필요.

**복원했는데 코드 상태가 요약과 다름**
→ 다른 사람이 커밋했거나 git에서 브랜치 이동을 했을 가능성. `/catchup`이 `git status`를 함께 보고하도록 커스터마이징 권장.

**스키마 검증 실패**
→ `summarize.md`에 명시된 `required` 필드를 빼먹은 경우. JSON 파일을 직접 열어 누락 필드 확인.
