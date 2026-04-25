---
description: 대화를 구조화된 JSON 스키마로 요약하여 다음 세션에 전달
argument-hint: [output-filename (선택)]
allowed-tools: Read, Write, Bash(mkdir:*), Bash(git:*), Bash(date:*), Bash(pwd:*)
---

# 대화 요약 → JSON

지금까지의 대화 전체를 `.claude/schemas/conversation-summary.schema.json` 스키마에 맞춰 요약한다.

## 절차

1. **스키마 로드**: `.claude/schemas/conversation-summary.schema.json`을 읽어 필드 구조를 확인한다.

2. **메타 컨텍스트 수집**:
   - `date -u +"%Y-%m-%dT%H:%M:%SZ"` → `meta.timestamp`
   - `date +"%Y%m%d-%H%M%S"` → 파일명용 타임스탬프
   - `git rev-parse --abbrev-ref HEAD 2>/dev/null` → `meta.branch` (실패 시 생략)
   - `git rev-parse --short HEAD 2>/dev/null` → `meta.commit` (실패 시 생략)
   - `pwd` → `state.working_directory`

3. **요약 작성 원칙**:
   - **객관적으로**: 추측 금지. 대화에서 실제로 일어난 것만 기록.
   - **구체적으로**: "버그 수정" ❌ → "auth.ts:42의 토큰 만료 처리 로직 수정" ✅
   - **다음 세션 관점으로**: 새 Claude 인스턴스가 이 JSON만 보고 바로 이어서 작업할 수 있어야 한다.
   - **중복 제거**: 여러 필드에 같은 내용을 복사하지 말 것. 각 필드는 고유한 역할이 있다.
   - **생략 가능 필드**: 해당 없으면 빈 배열/객체 대신 필드 자체를 생략한다 (required 제외).
   - **issues**: 해결된 것도 포함하되 `status: "resolved"`로 남긴다 — 왜 그 방식으로 풀었는지가 중요.
   - **next_actions**: 최소 1개 이상. 각 action은 "어느 파일의 어느 함수부터 손댈지"까지 써라.

4. **출력 경로 결정**:
   - `$ARGUMENTS`가 주어지면 → `.claude/summaries/$ARGUMENTS` (확장자 .json 자동 추가)
   - 없으면 → `.claude/summaries/summary-{YYYYMMDD-HHMMSS}.json`
   - 디렉토리가 없으면 `mkdir -p .claude/summaries`로 생성

5. **저장**:
   - JSON은 2-space indent, UTF-8
   - Write 도구로 저장

6. **확인 메시지**:
   ```
   ✅ 요약 저장: {파일경로}
   📌 다음 작업: {next_actions[0].action}

   다음 세션에서 `/catchup` 으로 자동 복원됩니다.
   ```

## 주의
- 스키마의 `required` 필드는 반드시 채운다.
- `enum`으로 제약된 필드는 정의된 값 중에서만 선택한다.
- 코드 스니펫은 정말 "다시 찾기 번거로운" 것만 넣는다. 전체 파일을 복사하지 않는다.
- 파일명은 반드시 `summary-` 로 시작해야 `/catchup`이 인식한다.
