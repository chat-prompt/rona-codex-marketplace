---
name: rona-review
description: "Rona 맞춤 스킬 사용 후 후기 작성 — 같은 폴더에 여러 스킬이 설치돼 있으면 .rona-skill.json 마커로 어느 스킬에 대한 리뷰인지 식별합니다."
license: MIT
metadata:
  generated_by: rona-skill-install
  student_token: "{{STUDENT_TOKEN}}"
  practice_id: "{{PRACTICE_ID}}"
---

# /rona-review — 맞춤 스킬 후기 남기기

이 슬래시 커맨드는 사용자가 Rona 맞춤 스킬을 한 번 써본 뒤 후기를 남길 때 호출합니다.

## 어느 스킬에 대한 리뷰인지 식별하는 방법

현재 작업 디렉토리에 `.rona-skill.json` 마커가 있으면 그 안의 `practice_id` / `student_token` / `title` 을 사용해 해당 스킬에 대한 리뷰로 처리합니다.

- 마커가 **1개** 있으면: 자동으로 그 스킬에 대한 리뷰로 진행
- 마커가 **여러 개** 있으면: `AskUserQuestion` 으로 묻기 — `options` 에 발견된 스킬 이름들을 박는다 (≥2 필수, ≤4 권장. 5개 이상이면 상위 4개 + "Other").
- 마커가 **없으면**: 안내 메시지 — "먼저 Rona 맞춤 스킬을 install 하세요"

## 진행 흐름

1. `.rona-skill.json` 마커 탐색 (현재 폴더 + `.claude/skills/*/SKILL.md` 의 frontmatter 둘 다 확인)
2. 대상 스킬이 결정되면 **`AskUserQuestion` 한 번** 으로 두 질문을 묶어 받는다.
   install 명령이 `--permission-mode bypassPermissions` 로 진입했으므로 도구 호출
   자체는 허용된다. 단 `options` 는 **반드시 2개 이상** (≥2 schema 강제).

   ```jsonc
   AskUserQuestion({
     questions: [
       {
         question: "<스킬 제목> — 동료에게 권하고 싶나요?",
         header: "추천 여부",
         multiSelect: false,
         options: [
           { label: "yes", description: "권한다 / 다시 쓰겠다" },
           { label: "no",  description: "권하지 않는다 / 아쉬웠다" }
         ]
       },
       {
         question: "평소 방식과 다른 점, 또는 아쉬웠던 점 한 줄",
         header: "한 줄 후기",
         multiSelect: false,
         options: [
           { label: "직접 입력",   description: "자유 텍스트로 답변 (Other 도 자유 입력)" },
           { label: "후기 생략",   description: "yes/no 만 전송하고 마무리" }
         ]
       }
     ]
   })
   ```

   - 사용자는 어떤 질문에서든 **"Other"** 로 자유 텍스트를 추가로 입력할 수 있다 (UI 가 자동 제공). "직접 입력" 옵션을 고른 경우엔 추가 안내 없이 그대로 입력란이 열린다.
   - 두 번째 질문에서 "후기 생략" 을 고르면 `text` 는 빈 문자열로 전송.
   - 결과 매핑: `decision = options[0].label === "yes" ? "yes" : "no"`,
     `text = answers[1].notes ?? options[1].label === "후기 생략" ? "" : <사용자 입력>`.

3. 응답을 `POST {{API_BASE_URL}}/skill/api/log/{{STUDENT_TOKEN}}` 로 전송:
   ```json
   {
     "event_type": "user_note",
     "payload": {
       "narrative": { "decision": "yes" | "no", "text": "<한 줄>" },
       "practice_id": "{{PRACTICE_ID}}"
     }
   }
   ```
4. 200 OK 응답 받으면 사용자에게 "전달했어요, 다음 라운드 개선에 직접 반영됩니다" 안내. 비-200 응답이면 응답 본문을 한 줄 인용한 뒤 사용자에게 재시도 여부를 다시 `AskUserQuestion` 으로 묻기 (yes/no 옵션).

## Fallback — `AskUserQuestion` 자체가 막힌 환경

`bypassPermissions` 가 아닌 모드로 진입했거나 도구 호출이 거절되면, 같은 정보를 일반 채팅 텍스트로 받는다:

```
<스킬 제목> 후기 한 줄 부탁드려요.
1. 동료에게 권하고 싶나요? (yes / no)
2. 평소 방식과 다른 점, 또는 아쉬웠던 점 한 줄 (생략 OK)

예: yes / drop-in 가능성 알려준 게 결정적이었다
예: no / 우리는 단일 프로바이더라 과한 가이드였다
```

이 fallback 경로의 응답 파싱:
- "yes" / "y" / "권한다" / "추천" → `decision: "yes"`
- "no" / "n" / "권하지" / "별로" → `decision: "no"`
- `text` = 응답 전체에서 yes/no 토큰 제거 + trim. 없으면 빈 문자열.

## 주의

- 본문 placeholder (누리님이 본문 작성 후 swap). install 시점에 `{{STUDENT_TOKEN}}` / `{{PRACTICE_ID}}` 치환됨.
- 본 SKILL.md 는 `.claude/skills/rona-review/SKILL.md` 경로에 install 됨. `rona-review` slug 는 고정.
