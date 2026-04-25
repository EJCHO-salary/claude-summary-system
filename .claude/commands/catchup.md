---
description: 가장 최근 대화 요약을 자동으로 불러와 이어서 작업
argument-hint: [파일명 또는 키워드 (선택, 비우면 최신)]
allowed-tools: Read, Bash(ls:*), Bash(test:*)
---

# 이전 세션 이어가기 (Catch-up)

`.claude/summaries/` 디렉토리에서 요약 파일을 찾아 컨텍스트를 복원하고 `next_actions`부터 작업을 이어간다.

## 파일 선택 로직

`$ARGUMENTS` 비어있음 → 가장 최근 파일 자동 선택

`$ARGUMENTS` 주어진 경우:
- `summary-`로 시작하는 완전한 파일명 → 그 파일 사용 (존재 확인 필요)
- 키워드/부분 문자열 → 매칭되는 파일 중 가장 최근 것 사용
- 매칭 없으면 → 사용자에게 알리고 중단

## 절차

### 1. 디렉토리 확인

```bash
test -d .claude/summaries && ls -1 .claude/summaries/summary-*.json 2>/dev/null | wc -l
```

디렉토리가 없거나 결과가 0이면:
> "이전 요약이 없습니다. 작업을 마치고 `/summarize`로 첫 요약을 생성하세요."
출력 후 종료.

### 2. 파일 목록 가져오기 (최신순)

```bash
ls -1 .claude/summaries/summary-*.json 2>/dev/null | sort -r
```

파일명에 타임스탬프(`YYYYMMDD-HHMMSS`)가 포함되어 있어 역순 정렬하면 최신이 맨 위에 온다.

### 3. 대상 파일 결정

- **인자 없음** → 정렬 결과의 첫 번째 파일
- **인자가 `summary-`로 시작** → 그 경로의 파일이 실제 존재하는지 `test -f`로 확인 후 사용
- **인자가 키워드** → `ls` 결과에서 키워드를 포함하는 첫 번째(가장 최근) 파일 선택
- **2개 이상 매칭이 명확한 동순위** → 상위 3개를 보여주고 어느 것을 쓸지 사용자에게 물어본다

### 4. 요약 로드 및 검증

대상 파일을 Read 도구로 읽는다. JSON 파싱 후 다음 필드 존재 확인:
- `meta`, `objective`, `state`, `next_actions`

하나라도 없으면:
> "⚠️ 스키마와 일치하지 않는 요약입니다: {누락 필드}. 그래도 계속할까요?"
사용자 확인을 받는다.

### 5. 컨텍스트 복원 보고

사용자에게 다음을 **간결하게** 보고한다 (장황하게 늘어놓지 말 것):

```
📂 복원: {파일명}
🕐 {meta.timestamp}{meta.branch가 있으면 ` | {branch}@{commit}` 추가}

🎯 목표: {objective.primary_goal}
📍 단계: {state.phase}

✅ 최근 완료: {progress.completed의 마지막 1-2개}
🔧 진행 중: {progress.in_progress 전부}
⚠️  열린 이슈: {issues 중 status가 resolved/workaround 아닌 것 — 개수 + 가장 심각한 것 1개}

▶️  다음 작업:
   1. [{priority}] {next_actions[0].action}
      → {entry_point if exists}
   2. {next_actions[1] 있으면 한 줄로}
```

### 6. 작업 시작

보고 직후 **`next_actions`의 첫 번째 항목** (priority=must 우선, 없으면 should, 그것도 없으면 첫 항목)의 `entry_point`에 해당하는 파일을 Read하고 작업을 시작한다.

단, 다음의 경우 작업 시작 전에 사용자 확인을 받는다:
- `open_questions`에 미해결 질문이 있는 경우 → 질문을 먼저 보여주고 답변 요청
- `state.phase`가 `blocked`인 경우 → "이전 세션에서 블로커가 있었습니다: {issues 중 status=blocked}. 어떻게 진행할까요?"
- `next_actions[0].priority`가 `could`인 경우 → "선택적 작업만 남아있습니다. 진행하시겠습니까?"

### 7. 사용자 선호 적용

`user_preferences`가 있으면 즉시 적용한다:
- `language: "ko"` → 한국어로 응답
- `style`, `avoid` → 응답 톤/형식에 반영

## 주의사항

- 요약 파일은 **참고 자료**다. 코드의 현재 상태와 다를 수 있으므로, 실제 파일을 읽기 전까지 단정하지 말 것.
- `decisions`에 기록된 것을 다시 꺼내지 말 것 — 이미 결정됐다.
- `issues.attempted_fixes`에 있는 방법을 다시 시도하지 말 것 — 이미 실패한 방법이다.
- 보고는 짧게. 사용자는 빠르게 작업을 이어가고 싶어한다.
- 요약과 현재 코드가 어긋나면(파일이 사라졌거나, 이미 수정되었거나) 사용자에게 알리고 조정 방향을 확인한다.
