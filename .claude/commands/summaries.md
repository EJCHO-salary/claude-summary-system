---
description: 저장된 대화 요약 목록을 최신순으로 표시
allowed-tools: Read, Bash(ls:*), Bash(test:*)
---

# 요약 목록

`.claude/summaries/` 디렉토리의 요약 파일을 최신순으로 보여준다.

## 절차

### 1. 디렉토리 확인

```bash
test -d .claude/summaries && ls -1 .claude/summaries/summary-*.json 2>/dev/null | wc -l
```

결과가 0이면 → "저장된 요약이 없습니다. `/summarize`로 첫 요약을 만드세요." 출력 후 종료.

### 2. 파일 목록 가져오기 (최신순, 최대 10개)

```bash
ls -1 .claude/summaries/summary-*.json 2>/dev/null | sort -r | head -10
```

### 3. 각 파일에서 요약 메타 추출

각 파일을 Read로 열어 다음만 추출한다 (전체를 다 읽을 필요 없음):
- `meta.timestamp`, `meta.branch` (있으면)
- `objective.primary_goal`
- `state.phase`
- `next_actions[0].action`
- `issues` 중 status가 `resolved` 아닌 것의 개수

### 4. 표 형식으로 출력

```
# 저장된 요약 (최신순, 최대 10개)

| # | 시각 | 브랜치 | 단계 | 목표 | 다음 작업 | 열린 이슈 |
|---|------|--------|------|------|-----------|-----------|
| 1 | 2026-04-25 14:30 | main | implementing | 인증 리팩토링 | auth.ts 토큰 갱신... | 2 |
| 2 | 2026-04-24 17:42 | feat/api | debugging | API 에러 핸들링 | 500 에러 재현... | 1 |
...

이어가기:
  • `/catchup`           → 1번 (최신) 자동 복원
  • `/catchup <키워드>`  → 키워드 포함 파일 중 최신
  • `/catchup <파일명>`  → 특정 파일 직접 지정
```

## 주의

- 표의 "다음 작업" 칸은 너무 길면 30자 정도로 축약 + `...` 추가.
- 파일이 11개 이상이어도 10개까지만 보여주고, 마지막에 "(이전 N개 더 있음)" 정도로 알림.
- 한 줄 요약이라 정보가 부족하다고 느끼면 사용자가 `/catchup <파일명>`으로 전체를 볼 수 있다고 안내.
