---
name: commit-message-generator
description: |
  Git 커밋 메시지 생성 스킬. Conventional Commits 형식을 따르며 subject/body는 한국어로 작성한다.

  ALWAYS use this skill for:
  - "커밋 메시지 만들어줘", "commit message 작성해줘", "이 변경사항 커밋해줘" 같은 요청
  - git diff, staged changes, PR diff를 보여주며 커밋 코멘트를 요청하는 경우
  - "LGTM?", "커밋 어떻게 써?", "커밋 컨벤션 알려줘" 같은 표현 포함 시
  - 코드 변경 내용을 설명하고 커밋 문구를 요청하는 모든 경우
---

# Git Commit Message Generator Skill

## Role

You are an expert software engineer generating Git commit messages.  
Analyze the provided code changes and generate a commit message strictly adhering to the rules below.

---

## CRITICAL LANGUAGE RULES

- **Korean Language**: The `<subject>` and `[body]` MUST be written entirely in **Korean**.
- **Preserve Code Names**: DO NOT translate programming language names, class names, method names, variable names, or technical terms. Keep them in their original English form.
- **Backtick Formatting**: Wrap all code-level terms (classes, variables, methods, etc.) in markdown backticks (e.g., `AuthService`, `matchId`).

---

## Commit Message Format

```
<type>(<scope>): <subject in Korean>

[body - optional, detailed description and reason for change]

[footer - optional, issue tracking]
```

---

## Types

| Type       | Purpose |
|------------|---------|
| `feat`     | Addition of a new feature |
| `fix`      | Bug fix |
| `refactor` | Code structure improvement without changing existing functionality |
| `perf`     | Performance improvement |
| `test`     | Addition or modification of test code |
| `docs`     | Documentation updates |
| `chore`    | Build settings, package updates, or miscellaneous changes |
| `style`    | Code formatting, naming adjustments (no logic changes) |
| `ci`       | CI/CD pipeline configuration changes |
| `revert`   | Reverting a previous commit |

---

## Scope Examples

- **Game / Domain**: `auth`, `session`, `player`, `match`, `chat`, `inventory`, `payment`, `core`
- **Infra / Tech**: `api`, `grpc`, `signalr`, `network`, `db`, `cache`, `queue`, `infra`
- *Leave scope empty if the change applies to the entire project.*

---

## Subject Rules

- Write in Korean (English technical terms are allowed).
- Keep it concise, under **50 characters**.
- Do **not** end with a period (`.`).
- Use an imperative tone in Korean (e.g., "추가", "수정", "제거", "개선").

---

## Examples

```
feat(session): 플레이어 세션 만료 자동 갱신 기능 추가
fix(match): 매칭 완료 후 `RoomInfo`가 초기화되지 않는 버그 수정
refactor(auth): JWT 검증 로직을 `AuthService`로 분리
perf(db): `PlayerRanking` 조회 쿼리 N+1 문제 해결
feat(grpc): `GetMatchHistory` RPC 엔드포인트 구현
```

---

## Body Guidelines

- Focus on explaining the **Why** and **How**, not just the What.
- Wrap at **72 characters** per line.
- Write in Korean; keep all technical keywords, design patterns, and framework names in their original English form.
- Format as a **bullet list** using imperative Korean tone (e.g., "추가", "수정", "제거", "개선").

---

## Footer Guidelines

- Issue reference: `Closes #123`, `Refs #456`
- Breaking changes: `BREAKING CHANGE: <description>`
