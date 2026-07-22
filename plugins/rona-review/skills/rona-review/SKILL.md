---
name: rona-review
description: 로나 맞춤 스킬을 실제 업무에 적용한 직후, 그 경험을 4차원(사용 도구·외부 출처·시도와 실패·로나 스킬의 부족한 점)으로 정리하고 셀프 인터뷰로 노력 간극을 보강한 뒤 마스킹된 후기를 로나에 제출한다. 사용자가 "/rona-review", "스킬 후기", "리뷰 보낼게", "피드백 보낼게", "스킬 다 썼어", "후기 남길게" 같은 표현을 쓰면 반드시 이 스킬을 발동한다. 다른 스킬과 헷갈리지 말 것 — 이 스킬은 로나 install 스킬을 써본 직후 회수용이다.
allowed-tools: [Read, Write, Bash, Glob]
---

# rona-review — 로나 맞춤 스킬 후기 회수

파트너가 로나 맞춤 스킬을 실제 업무에 적용한 직후, 그 경험을 정밀하게 회수해 로나 DB에 전송한다.

## 핵심 수집 대상

**"로나 스킬이 부족해서 사람이 직접 채운 노력의 정밀한 지도"**

- 로나 스킬만으로 안 됐던 지점
- 사람이 그걸 채우려 한 시도·실패·전환
- 그 과정에서 끌어온 외부 도구·검색·동료·다른 LLM
- 결과적으로 자동화 가능한 부분 (다음 스킬 개선 + 지불 가치 신호)

`아쉬운 점` 같은 정성 평가는 부산물이다. 본질은 **노력 간극의 형태·크기·원인**.

> **리뷰 모드 불변식 (리뷰는 리뷰만).** 이 스킬이 도는 동안에는 **원래 작업을 재개하거나
> 구현하지 않는다.** 파트너가 후기에 "이건 아쉬웠다 / 이걸 더 했어야 한다" 같은 *작업성*
> 피드백을 남겨도, 그 자리에서 고치거나 작업을 이어가지 말고 후기 항목(narrative)으로만
> 기록한다. 리뷰 세션의 유일한 산출은 후기다. (이어서 할 일이 있으면 STEP 6 발송 *이후*
> 딱 한 번 제안한다 — 아래 STEP 6 참조.)

---

## 사용 시점 (파트너 안내용)

> **로나 맞춤 스킬 작업을 마친 직후 같은 세션에서 `/rona-review`를 실행하세요.**
> 며칠 지나면 회상 정확도가 떨어집니다. 작업 종료 후 5분 안에 시작하는 게 가장 좋습니다.

---

## STEP 1. 마커 파일 탐지 — 어느 스킬에 대한 리뷰인가

로나 install 패키지에는 식별용 마커 파일 `.rona-skill.json`이 같이 들어 있다.

```json
{
  "practice_id": "uuid",
  "student_token": "...",
  "title": "스킬 제목",
  "installed_at": "ISO8601"
}
```

### 탐지 로직

1. 현재 작업 디렉토리(`pwd`) 기준으로 `.rona-skill.json` 글로빙:
   ```bash
   find . -maxdepth 4 -name '.rona-skill.json' \
     -not -path '*/node_modules/*' \
     -not -path '*/.git/*' \
     -not -path '*/.next/*' 2>/dev/null
   ```
2. 발견 개수에 따라 분기:
   - **0개**: **바로 "없음"으로 단정하지 말고 한 번 교차확인한다.** 여기서 오분기하면 후기가
     아예 시작되지 않는 *조용한 회수 실패*가 된다. 셸 훅·alias·래퍼가 끼면 `find`가 빈 출력에
     종료코드 1을 내는 일이 실제로 있었다(블라인드 테스트 2회 연속 관측 — 마커는 멀쩡히
     존재했다). 다른 방식으로 한 번 더 본다:
     ```bash
     ls -d .claude/skills/*/.rona-skill.json 2>/dev/null
     ls -a . 2>/dev/null | grep -c '^\.rona-skill\.json$'
     ```
     둘 다 비면 진짜 0개다 → 파트너에게 안내: *"이 폴더에서 로나 스킬을 찾을 수 없어요.
     로나 스킬을 install한 폴더에서 다시 실행해주세요."* → 종료.
     한쪽이라도 파일을 찾으면 **`find`가 실패한 것**이므로 그 결과로 진행하고 discrepancies에
     남긴다: *"find 실패(셸 환경 추정) — 다른 방식으로 마커 탐지"*.
   - **1개**: 자동으로 그 스킬 선택, `practice_id` + `student_token` + `title` + `installed_at` 추출
   - **여러 개**: 파트너에게 리스트 제시 — *"어느 스킬에 대한 리뷰인가요?"* + 번호 선택
3. 선택된 스킬의 `practice_id`, `student_token`, `title`, `installed_at`을 메모리에 보관 — 이후 STEP에서 사용
   (`installed_at`은 STEP 2-2의 세션 파일 교차검증에 쓰인다 — 세션이 install 시점보다 먼저 시작될 수 없다는 하한선)

---

## STEP 2. 세션 jsonl 자동 추출 — 4차원 중 1·2·3번

### 2-1. 세션 디렉토리 찾기

```bash
# 현재 프로젝트 경로를 Claude Code 세션 폴더명 규칙대로 치환.
# 주의: `/`만 바꾸면 안 된다 — Claude Code는 영숫자가 아닌 모든 문자(밑줄 `_`, 점 `.`,
# 공백, 괄호 등)를 개별적으로 `-`로 치환한다. 예: `rona_practice` → `rona-practice`
# (밑줄도 치환). `/`만 치환하면 밑줄 등이 남은 폴더를 가리켜 세션 디렉토리 자체가
# 존재하지 않게 되고, 이후 STEP 2 전체가 조용히 빈 결과로 무력화된다(실측 확인됨).
PROJECT_KEY=$(pwd | sed 's/[^A-Za-z0-9]/-/g')
SESSION_DIR="$HOME/.claude/projects/$PROJECT_KEY"

# 존재 확인 — 없으면 "매칭 0건"이 아니라 *경로 유도 실패*다. 둘을 구분하지 않으면
# 원인(경로 규칙 오류)과 전혀 다른 진단이 discrepancies 에 적힌다.
[ -d "$SESSION_DIR" ] || echo "SESSION_DIR_MISSING: $SESSION_DIR"
```

`SESSION_DIR_MISSING`이 뜨면 자동 추출을 **시도하지 마라**. 파트너에게 *"이 폴더에서 진행한
세션 기록을 찾지 못했어요 — 혹시 다른 폴더에서 작업하셨나요?"* 라고 묻는다.

> **폴더를 알아냈다고 해서 "거기서 다시 실행하세요"라고 하지 마라 — 그러면 STEP 1이 깨진다.**
> 마커 탐지는 `pwd` 기준 `-maxdepth 4`인데, 스킬이 하위 폴더에 설치돼 있으면 상위에서
> 재실행할 때 마커가 depth 5가 되어 안 잡히고 *"로나 스킬을 찾을 수 없어요 → 종료"* 로
> 빠진다(실측 라운드 8: 마커는 하위에서만, 세션은 상위에서만 보이는 교착). **마커 폴더와
> 세션 폴더는 다를 수 있다** — 클로드코드를 상위 폴더에서 띄워놓고 하위 프로젝트를 작업하는
> 건 흔한 사용 방식이다.
>
> 따라서 **재실행시키지 말고, STEP 1에서 이미 얻은 마커 정보는 그대로 둔 채 `SESSION_DIR`만
> 그 폴더 기준으로 다시 계산한다**:
> ```bash
> WORK_DIR="<파트너가 말한 폴더의 절대경로>"
> SESSION_DIR="$HOME/.claude/projects/$(printf '%s' "$WORK_DIR" | sed 's/[^A-Za-z0-9]/-/g')"
> [ -d "$SESSION_DIR" ] || echo "여기도 없음: $SESSION_DIR"
> ```
> discrepancies에 남긴다: *"마커 폴더와 세션 폴더가 다름(작업은 상위 폴더에서 진행) —
> SESSION_DIR 재계산"*.

그래도 못 찾으면 자동 추출을 포기하고 STEP 3 진술로만 채우되
discrepancies에 명시한다: *"세션 디렉토리 없음 — 자동 추출 불가, 진술로 대체"*.

### 2-2. 최신 세션 jsonl 선택

> **왜 mtime 단독으로 고르면 안 되는가**: 같은 프로젝트 폴더에서 여러 Claude 세션(다른 창·다른
> 작업)이 동시에 열려 있으면, mtime이 가장 최신인 파일이 지금 이 스킬을 진행한 세션이 아닐 수
> 있다. 실제로 이 오추출이 발생해 리뷰 데이터가 다른 세션 내용으로 채워진 사례가 있었다
> (2026-07 rona-review 데이터 신뢰도 조사). 따라서 mtime만으로 확정하지 말고, 해당 세션이
> **이 스킬을 실제로 다뤘다는 내용 증거**로 교차검증한다.

1. `installed_at` 이후 mtime을 가진 후보만 남긴다(설치 이전 세션은 이 스킬을 다룰 수 없다).
   **날짜 파싱에 두 가지 함정이 있다 — 둘 다 실측으로 확인된 것이니 아래 형태를 그대로 쓸 것**:
   - 마커의 `installed_at`은 `new Date().toISOString()` 산출이라 **항상 밀리초가 붙는다**
     (`2026-07-20T09:00:00.000Z`). macOS `date -jf "%Y-%m-%dT%H:%M:%SZ"`는 밀리초를 파싱하지
     못해 빈 문자열을 내고, 그 빈 값을 `[ "$MTIME" -ge "" ]`로 비교하면 zsh에서 **참**이 되어
     하한선 필터가 조용히 전량 통과로 무력화된다. → 소수점 이하를 먼저 잘라낸다.
   - 마커는 UTC(`Z`)인데 `-u` 없이 파싱하면 로컬시각으로 해석해 KST 기준 **9시간** 어긋난다.
     → `-u`를 반드시 붙인다.
   ```bash
   # 밀리초 제거 후 UTC로 파싱. 실패해도 빈 값이 비교로 새지 않게 명시적으로 처리한다.
   INSTALLED_CLEAN=$(printf '%s' "$INSTALLED_AT" | sed 's/\.[0-9]*Z$/Z/')
   INSTALLED_EPOCH=$(date -u -d "$INSTALLED_CLEAN" +%s 2>/dev/null \
     || date -u -jf "%Y-%m-%dT%H:%M:%SZ" "$INSTALLED_CLEAN" +%s 2>/dev/null \
     || true)

   if [ -z "$INSTALLED_EPOCH" ]; then
     # 파싱 실패 — 하한선을 포기하고 진행하되 반드시 discrepancies 에 남긴다.
     # (빈 값을 그대로 비교에 쓰면 조용히 전량 통과하므로 0으로 명시한다.)
     INSTALLED_EPOCH=0
     LOWER_BOUND_FAILED=1
   fi

   CANDIDATES=$(ls -t "$SESSION_DIR"/*.jsonl 2>/dev/null \
     | grep -v '/agent-' \
     | while read -r f; do
         [ -s "$f" ] || continue  # 크기 0 제외
         MTIME=$(stat -f %m "$f" 2>/dev/null || stat -c %Y "$f")
         [ "$MTIME" -ge "$INSTALLED_EPOCH" ] && echo "$f"
       done)
   ```
   `LOWER_BOUND_FAILED=1`이면 discrepancies에 명시한다: *"installed_at 파싱 실패로 설치시각
   하한선 미적용 — 설치 이전 세션이 후보에 포함됐을 수 있음"*.
2. **후보 전원을 훑어 매칭되는 파일을 전부 모은다** (첫 매칭에서 멈추지 않는다 — 같은 폴더에서
   rona-alpha로 여러 스킬을 반복 사용했으면 `practice_id`가 둘 이상의 후보에 우연히 걸릴 수
   있고, 첫 매칭에서 멈추면 그 다중매칭을 놓친 채 조용히 틀린 파일을 확정하게 된다).
   **주의: `for f in $CANDIDATES` 형태로 짜면 안 된다** — zsh(macOS 기본 셸)는 unquoted 변수를
   IFS로 word-split하지 않아 후보가 2개 이상이면 개행이 낀 문자열 전체가 "파일 하나"로 취급되고
   `grep`이 항상 실패해 매칭 로직 전체가 조용히 무력화된다(실측 확인됨 — 이 세션의 실행 셸도
   zsh였음). `while read` 로 줄 단위로 읽어야 bash·zsh 양쪽에서 동일하게 동작한다:
   ```bash
   MATCHES=$(printf '%s\n' "$CANDIDATES" | while read -r f; do
     [ -n "$f" ] || continue
     grep -q "$PRACTICE_ID" "$f" 2>/dev/null && echo "$f"
   done)
   if [ -z "$MATCHES" ]; then
     MATCH_COUNT=0
   else
     MATCH_COUNT=$(printf '%s\n' "$MATCHES" | grep -c .)
   fi
   ```
3. **먼저 후보 자체가 있는지 본다.** `CANDIDATES`가 비었으면(세션 디렉토리는 있는데 `.jsonl`이
   없거나·전부 크기 0이거나·전부 설치시각 이전) 이건 "매칭 0건"과 **다른 상황**이다 — 보여줄
   후보가 없으므로 파트너에게 고르라고 할 수도 없다. 자동 추출을 포기하고 STEP 3 진술로만 채우되
   discrepancies에 명시한다: *"설치 이후 세션 기록 없음 — 자동 추출 불가, 진술로 대체"*.

4. 후보가 있으면 매칭 개수에 따라 분기 — **매칭 1건만 조용히 통과, 그 외엔 전부 파트너 확인**.
   파트너에게 후보를 제시할 때는 **어느 분기든 아래 요약을 그대로** 쓴다(파일명만 보여주면
   파트너가 고를 수 없다 — 첫 메시지가 있어야 자기 작업을 알아본다):
   **첫 메시지만 보여주면 안 된다** — 한 세션에서 스킬 A→B 를 연속 진행했으면 그 파일은 항상
   A의 문장으로 표시되고, B 후기를 남기려는 파트너는 **정답 파일을 "내 세션이 아니네"라며
   걸러낸다**(실측 라운드 6: 정답 파일이 `경쟁사 가격정책 리서치 스킬 시작할게…`로 표시됨).
   그래서 **첫 문장과 마지막 문장을 같이** 보여준다 — 세션이 어디서 시작해 어디서 끝났는지가
   보여야 파트너가 자기 작업을 알아본다.
   ```bash
   printf '%s\n' "$CANDIDATES" | while read -r f; do
     [ -n "$f" ] || continue
     # `(.message.content // .content)` — 스키마 정규화(2-3의 rona_window 와 같은 이유).
     # 이게 없으면 요약이 **빈 줄**로 나와, 파일명만 보여주지 말라는 위 원칙이 그대로 깨진다.
     TEXTS=$(jq -r 'select(.type=="user") | (.message.content // .content)
       | if type=="string" then . else (.[]? | select(.type=="text") | .text) end' "$f" 2>/dev/null)
     FIRST=$(printf '%s\n' "$TEXTS" | head -1 | cut -c1-50)
     LAST=$(printf '%s\n' "$TEXTS" | grep -v '^$' | tail -1 | cut -c1-50)
     printf '%s | %s\n    처음: %s\n    끝  : %s\n' \
       "$(basename "$f")" "$(date -r "$f" '+%m/%d %H:%M')" "$FIRST" "$LAST"
   done
   ```
   > **하류로 넘기는 값은 `LATEST_FILES`(개행 구분 목록)다.** 대부분 1줄이지만 재개한 경우
   > 여러 줄이 된다 — 아래 STEP 2-3의 `rona_main`/`rona_logs`가 이 목록을 훑는다.

   - **정확히 1건**: 그 파일을 `LATEST_FILES`로 자동 채택하고 확인 없이 진행한다
     (`LATEST_FILES="$MATCHES"`). 단 **"1건이니까
     안전하다"고 방심하지 마라** — 위 각주대로 `practice_id`는 MCP 호출 인자로 *우연히* 한 줄
     남았을 뿐일 수 있고, 그 한 줄이 곧 단일 매칭이 된다. 채택 직후 아래를 찍어 **그 파일이
     이 스킬 전용 세션인지, 다른 작업이 앞에 붙은 세션인지**를 구분한다:
     ```bash
     # `$LATEST_FILES` 를 인용 없이 펼치지 마라 — zsh 는 word-split 하지 않아 개행 낀 문자열
     # 하나를 파일명으로 넘긴다(2번의 CANDIDATES 와 같은 함정). while read 로 훑는다.
     SEL_CAT=$(printf '%s\n' "$LATEST_FILES" | while read -r f; do [ -n "$f" ] && cat "$f"; done)
     PRE=$(printf '%s\n' "$SEL_CAT" | jq -c --arg lb "$INSTALLED_AT" \
       'select((.timestamp // "") != "" and (.timestamp // "") < $lb)' | grep -c .)
     POST=$(printf '%s\n' "$SEL_CAT" | jq -c --arg lb "$INSTALLED_AT" \
       'select((.timestamp // "") >= $lb)' | grep -c .)
     echo "설치 이전 레코드=$PRE / 이후=$POST"
     ```
     `PRE`가 0이 아니면 이 세션은 **설치 전부터 돌던 다른 작업을 품고 있다**. STEP 2-3의 시간창이
     그 구간을 잘라내지만, 사실은 discrepancies에 남긴다: *"세션에 이 스킬 설치 이전 구간
     N레코드 존재 — 시간창으로 절단함"*. `PRE`가 `POST`보다 훨씬 크면 파트너에게 한 번
     확인시켜라(*"이 세션에서 이 스킬 말고 다른 작업도 하셨죠?"*).
   - **0건**: **mtime 최신 파일을 자동으로 집지 마라.** 교차검증이 0건이라는 건 "어느 세션인지
     모른다"는 뜻이고, 이때 mtime 최신은 방금 열어둔 *전혀 무관한* 작업일 가능성이 높다(실측
     사례: 후보 최상위가 "휴가 신청서 써줘" 세션이었다). 위 후보 요약을 그대로 보여주고
     **"어느 세션에서 이 스킬을 진행하셨나요?"** 로 직접 고르게 한다. 목록에 없다고 하면
     자동 추출을 포기하고 STEP 3 인터뷰 진술로만 채운다.
     **discrepancies에 반드시 명시**: *"세션 파일 자동 매칭 실패(내용 교차검증 불통과) —
     파트너 선택으로 확정"* 또는 *"…— 자동 추출 포기, 진술로 대체"*.
     > 참고: `practice_id`는 원래 디스크의 마커 파일에 있는 값이라 대화 로그에 꼭 등장한다는
     > 보장이 없다(claim 응답·MCP 호출 인자로 우연히 남는 경우가 많을 뿐). 그래서 0건은
     > "이 폴더가 틀렸다"가 아니라 **흔히 있는 정상 상황**으로 취급하고 사람에게 묻는다.
   - **2건 이상**: mtime 최신을 조용히 고르지 마라. 다만 **"둘 중 하나가 틀렸다"고 단정하지도
     마라** — 다중매칭의 가장 흔한 원인은 다른 스킬이 섞인 게 아니라 **한 스킬을 중간에 끊었다가
     이어서 한 것**이고, 그때 정답은 "둘 다"다. 그래서 **양자택일이 아니라 세 갈래로** 묻는다.
     위 후보 요약(매칭된 것만)을 보여주고:
     ```
     이 스킬을 언급한 세션이 N개 있어요. 어느 쪽인가요?

       [1] <파일1 요약>
       [2] <파일2 요약>
       [전부] 한 번에 못 끝내서 중간에 끊었다가 이어서 했어요 (여러 세션에 걸침)
     ```
     - 번호 선택 → `LATEST_FILES`는 그 한 줄. discrepancies: *"세션 파일 다중매칭(N건) —
       파트너 선택으로 확정"*
     - **"전부" 선택 → `LATEST_FILES="$MATCHES"` (매칭된 전부를 합본으로 추출한다).**
       discrepancies: *"세션 N개에 걸친 작업 — 파트너 확인 후 합본 추출"*

     > **"전부" 갈래가 없으면 절반이 조용히 사라진다.** 실측(라운드 7): 재개 세션에서 뒤쪽
     > 파일만 채택하자 도구 20회 중 5회(25%), 외부 출처 2건 중 **1건(50%)**, 검색어 3건 중
     > 1건, 시나리오 1개가 유실됐다. 유실된 게 하필 **1차 세션의 초기 탐색 구간**(택소노미
     > 리서치 20분)이라, 후기에는 "이 스킬은 외부 리서치가 별로 필요 없었다"는 **정반대
     > 결론**이 적재된다.
     >
     > 재개 여부는 로그로 먼저 확인할 수 있다 — 한쪽 끝 발화가 "이따 이어서 하자"류이고
     > 다른 쪽 첫 발화가 "아까 하던 거 이어서"류이면, 또는 `get_progress`(재개 시에만 나오는
     > 호출)가 뒤쪽 파일에만 있으면 재개가 거의 확실하다. 그 경우 `[전부]`를 기본값으로
     > 추천해 제시한다.

5. **`LATEST_FILES` 확정 후, 마지막 활동이 얼마나 오래됐는지 본다.** 이 스킬은 "작업 종료 후
   5분 안" 회수를 전제로 설계됐는데(§사용 시점), 지금까지의 검사는 `installed_at` **하한**뿐이라
   *며칠 지난 작업*도 그대로 통과한다.
   ```bash
   # 서브에이전트까지 포함해 "가장 늦은" 활동을 본다 — 메인만 보면 서브에이전트가 더 늦게
   # 끝난 세션에서 실제보다 오래 지난 것으로 계산된다.
   LAST_TS=$(printf '%s\n' "$LATEST_FILES" | while read -r f; do
       [ -n "$f" ] || continue
       cat "$f"
       SUB="$SESSION_DIR/$(basename "$f" .jsonl)/subagents"
       [ -d "$SUB" ] && find "$SUB" -name 'agent-*.jsonl' -exec cat {} + 2>/dev/null
     done | jq -r '.timestamp // empty' | sort | tail -1)
   LAST_CLEAN=$(printf '%s' "$LAST_TS" | sed 's/\.[0-9]*Z$/Z/')
   LAST_EPOCH=$(date -u -d "$LAST_CLEAN" +%s 2>/dev/null \
     || date -u -jf "%Y-%m-%dT%H:%M:%SZ" "$LAST_CLEAN" +%s 2>/dev/null)
   # ⚠️ **분 단위로 찍어라.** 시간 단위(/3600)로 절삭하면 1분도 59분도 똑같이 "0시간"이 되어
   #    바로 아래 30분 임계를 판정할 수 없다(실측 라운드 9: 12분이 "0시간 경과"로 출력됐다).
   [ -n "$LAST_EPOCH" ] && echo "마지막 활동 이후 $(( ( $(date +%s) - LAST_EPOCH ) / 60 ))분 경과"
   ```
   **경과 시간은 길든 짧든 항상 discrepancies에 기록한다**(*"작업 종료 후 N시간 경과 후 회수"*).
   분석 단계에서 갓 나온 후기와 하루 지난 후기를 구분할 근거가 META에 남아야 하기 때문이다.
   그리고 **30분을 넘으면** STEP 3 인터뷰 첫 문장에 **시점 앵커**를 넣는다(*"어제 오후에
   마무리하신 그 작업 기준으로 여쭤볼게요"*) — 앵커 없이 물으면 파트너가 다른 작업과 뒤섞어
   답한다(telescoping).

   > **임계를 높게 잡으면 안 되는 이유.** 파트너의 "방금"은 믿을 수 없다. 실측 라운드 7에서는
   > "방금 끝냈어요"가 실제로 **19시간 전**이었고(그 사이 같은 폴더에서 무관한 세션이 한 번 더
   > 돌았다), 라운드 8에서는 **4시간 전**이었다. 4시간짜리는 웬만한 임계 아래로 조용히
   > 통과하는데, 회상 오차는 이미 충분히 크다. 파트너가 "방금"이라 느끼는 건 그 세션의 마지막
   > 발화가 *"후기 남길게"* 였던 탓이지 실제 경과 시간이 아니다.

> **세션 내 구간 절단 — 어디까지 해결됐나**: 위 로직은 "어느 *파일*이 이 스킬 세션인가"까지만
> 하드닝한다. 한 세션 안에서 스킬 A→B를 연속 진행한 경우는 STEP 2-3의 **`installed_at` 레코드
> 단위 시간창**이 A 구간을 잘라낸다(아래 `rona_logs`/`rona_main` 참조). 다만 **상한이 없어서**
> B를 먼저 하고 A를 나중에 한 순서는 여전히 섞이고, 같은 스킬을 재설치했으면 앞 구간이 잘린다.
> 파트너가 "그 전에/그 후에 다른 작업도 했는데"라고 말하면 STEP 2-3 결과를 그대로 믿지 말고
> 직접 구간을 확인시켜라.

### 2-3. 4차원 중 자동 추출 가능한 3개

> **추출 결과가 0행이면 "도구를 안 썼다"로 해석하지 마라.** 아래 jq들은 Claude Code jsonl
> 스키마(`{"type":"assistant","message":{"content":[{"type":"tool_use",...}]}}`)를 전제한다.
> 파일이 다른 CLI 산출이거나 스키마가 바뀌면 jq는 **종료코드 0에 빈 결과**를 내므로 실패가
> 실패로 보이지 않는다. (1)번 도구 집계가 0행이면 거의 확실히 추출 실패다 — 스킬을 진행했다면
> 도구 호출이 0일 수 없기 때문이다. 이 경우:
> 1. discrepancies에 명시: *"세션 로그 자동 추출 실패(스키마 불일치 추정) — 도구·출처는
>    파트너 진술로 대체"*
> 2. 자동 추출을 포기하고 STEP 3 인터뷰에서 파트너에게 직접 물어 채운다(빈 값으로 제출하지 마라).

> **⚠️ 서브에이전트 로그를 반드시 합쳐라 (안 합치면 오케스트레이션 실습이 통째로 안 보인다).**
> 메인 세션 jsonl에는 서브에이전트 파견이 `Agent`(또는 `Task`) 호출 **1줄**로만 남고, 그
> 서브에이전트가 실제로 쓴 도구·조회한 URL은 **다른 파일**에 있다:
> `$SESSION_DIR/<세션UUID>/subagents/agent-*.jsonl`
> 실측 예: 메인 세션엔 `Agent 8회`만 보이는데 그 서브에이전트들이 실제론 Bash 42·Read 3·Write 1
> (총 46회)을 썼다 — 메인만 집계하면 이게 전부 0으로 계산된다. 조사원 파견형 스킬에서는
> **외부 URL 대부분이 서브에이전트 쪽에만 있으므로** (2)번 외부 출처도 같이 비게 된다.
>
> 그래서 STEP 2-3의 모든 추출은 아래 `rona_logs`(메인 + 서브에이전트 합본) 스트림에 대해 돌린다:
> ```bash
> # 공통 필터: installed_at 시간창 + **timestamp 오름차순 정렬**.
> # 정렬을 빼면 안 되는 이유는 아래 "왜 정렬하는가" 참조.
> rona_window() {
>   jq -r --arg lb "$INSTALLED_AT" '
>       select((.timestamp // "") == "" or (.timestamp // "") >= $lb)
>       # 스키마 정규화 — 이 한 줄이 아래 모든 추출식을 살린다.
>       # `message` 래퍼 없이 `content` 가 최상위에 오는 산출이 있다(구버전·타 CLI·포맷 변경).
>       # 그 경우 `.message.content[]?` 는 종료코드 0에 **빈 결과**를 내므로 실패가 실패로
>       # 보이지 않는다. 여기서 한 번 감싸주면 하류를 하나도 안 고쳐도 된다.
>       | (if has("message") then . else . + {message: {content: .content}} end)
>       | "\(.timestamp // "")\t\(tostring)"' \
>     | awk -F'\t' 'BEGIN{OFS="\t"} {if($1==""){$1=prev}else{prev=$1}; print}' \
>     | sort -s -k1,1 | cut -f2-
> }
>
> # 메인 세션만 (서브에이전트 제외). 재개했으면 LATEST_FILES 에 여러 줄이 들어 있다.
> # `$LATEST` 를 직접 jq 에 물리지 마라 — 시간창·정렬·재개분이 전부 빠진다.
> rona_main() {
>   printf '%s\n' "$LATEST_FILES" | while read -r f; do
>     [ -n "$f" ] && cat "$f"
>   done | rona_window
> }
>
> # 메인 + 각 세션의 서브에이전트 합본.
> rona_logs() {
>   { printf '%s\n' "$LATEST_FILES" | while read -r f; do
>       [ -n "$f" ] || continue
>       cat "$f"
>       # glob(`agent-*.jsonl`) 대신 find — zsh 는 매칭 0건이면 "no matches found" 를 stderr 로
>       # 뱉어 실행자가 실패로 오인한다(파일 없는 경우가 오히려 정상이다). find 는 조용하고,
>       # 나란히 있는 `agent-*.meta.json` 도 정확히 제외한다.
>       SUBAGENT_DIR="$SESSION_DIR/$(basename "$f" .jsonl)/subagents"
>       [ -d "$SUBAGENT_DIR" ] && find "$SUBAGENT_DIR" -name 'agent-*.jsonl' -exec cat {} + 2>/dev/null
>     done
>   } | rona_window
>   return 0
> }
> ```
>
> **왜 정렬하는가 (빼면 조용히 틀린 시나리오가 생긴다).** `cat A B 서브에이전트` 는 **파일 연결
> 순서**일 뿐 시간순이 아니다. 그런데 아래 (3)은 `uniq -c` 로 *인접* 반복을 세고 timestamp 차이로
> 소요시간을 추정한다 — 둘 다 시간순을 전제한다. 실측(라운드 7): 정렬 없이 돌리자 스트림에
> `08:10Z → 07:26Z` 역행이 생겼고, `Bash 4회 연속`이 잡혔는데 실제로는 `02:11Z` 1회와
> `07:40~07:46Z` 3회로 **5시간 반 떨어진 별개 호출**이었다. 파일 경계를 넘어 "연속"이 조작된 것이다.
> 그 가짜 신호는 STEP 3-3 drill 선별 1순위(재시도 횟수)에 그대로 들어가, 파트너에게 **없던 막힘을
> 되묻게** 만든다.
>
> **중간의 `awk` 는 timestamp 없는 레코드를 제자리에 붙잡아 두는 장치다(빼지 마라).** 정렬 키가
> 빈 문자열이면 그 레코드는 **무조건 스트림 맨 앞으로** 튀어나가, 정렬로 지키려던 인접성이
> 그 지점에서 도로 깨진다(실측 라운드 9 재현: `AAA(02:40) → BBB(ts없음) → CCC(02:50)` 를 넣자
> `BBB → AAA → CCC` 로 나왔다). awk 가 직전 레코드의 timestamp를 물려주면 원래 자리에 남는다
> — `sort -s`(안정 정렬)라 키가 같으면 입력 순서가 보존되기 때문이다.
>
> **`installed_at` 레코드 단위 하한이 붙어 있는 이유 (지우지 마라).** 2-2의 하한선은 *파일*
> 단위(mtime)라, 한 세션 파일 안에서 스킬 A→B 를 연속 진행한 경우를 전혀 못 막는다. 마커의
> `installed_at`은 "이 스킬이 존재하기 시작한 시각"이므로 **그 이전 레코드는 정의상 이 스킬의
> 것일 수 없다** — 그래서 레코드 단위로 한 번 더 자른다. 실측(2026-07 라운드 6): 이 절단이
> 없으면 A→B 세션에서 도구 33회 중 27회(82%)와 외부 URL 7건·검색어 4건 **전량**이 다른 스킬
> 것으로 섞여 들어갔고, 이 한 줄을 붙이자 정확히 B의 6회만 남았다.
> 비교는 ISO8601 문자열 사전순이며, 마커와 세션 timestamp 둘 다 밀리초를 포함해 형식이
> 같으므로 안전하다. `timestamp`가 없는 레코드는 통과시킨다(메타성 레코드 유실 방지).
>
> **이 절단으로도 남는 것 (정직하게)**: 상한이 없어서 **B를 먼저 하고 A를 나중에** 한 순서에서는
> A가 그대로 섞인다. 또 같은 스킬을 두 번 설치했으면 나중 `installed_at`이 앞 구간을 잘라먹는다.
> 그래서 아래 (1) 도구 집계 결과가 파트너 진술과 크게 어긋나면 그대로 믿지 말고 확인시켜라.
> 서브에이전트 도구는 후기에 별도 표기한다 — 파트너 본인이 직접 쓴 것과, 위임해서 대신
> 굴린 것은 다른 사실이기 때문이다(4차원 2번 "노력의 형태"에 직결).

#### (1) 사용한 도구·스킬·방법론

**출처**: `tool_use` 블록 자동 집계 (메인 + 서브에이전트)

```bash
# 합본 집계
rona_logs | jq -r 'select(.type == "assistant") | .message.content[]?
  | select(.type == "tool_use") | .name' \
  | sort | uniq -c | sort -rn

# 메인 세션만 — 합본과의 차이가 곧 "서브에이전트에 위임된 몫"이다.
# (`"$LATEST"` 를 직접 물리지 마라 — 시간창이 빠진다. 위 rona_main() 을 쓴다.)
rona_main | jq -r 'select(.type == "assistant") | .message.content[]?
  | select(.type == "tool_use") | .name' \
  | sort | uniq -c | sort -rn
```

> **"메인 = 파트너가 직접 한 것"으로 읽지 마라.** 메인 세션의 도구 호출도 대부분 Claude가
> 실행한 것이고, 파트너는 문장 한 줄을 쳤을 뿐일 수 있다. 이 구분은 *사람 vs AI*가 아니라
> **본체 세션 vs 위임된 서브에이전트**의 구분이다. 4차원 2번(파트너가 채운 노력)은 이 숫자가
> 아니라 STEP 3 인터뷰 진술로 판단해야 한다.

**정리 형식**:
```markdown
## 사용한 도구·스킬

- WebSearch (5회) — 외부 사례 검색
- mcp__context7 (2회) — Vercel AI SDK 문서 조회
- Bash (12회) — 파일 조작, 로컬 테스트
- mcp__playwright (3회) — 브라우저 자동화 시도
```

각 도구마다 "주요 용도" 한 줄을 Claude가 추론해 붙인다 (tool_use input 샘플 보고 판단).

#### (2) 외부 출처

**출처**: `WebSearch` / `WebFetch` 결과 + 사용자 메시지에 등장한 URL/문서 인용

```bash
# 사용자 메시지에서 URL 추출 (파트너가 직접 던진 링크).
# user 의 .message.content 는 배열일 때도, **평문 문자열**일 때도 있다(실측: 배열 152 / 문자열 11).
# `.content[]?` 만 쓰면 문자열 쪽이 조용히 통째로 버려져 링크가 유실된다 — 반드시 정규화한다.
# (시간창 적용 — 다른 스킬 구간에서 파트너가 던진 사내 위키 링크가 이번 스킬 후기로
#  넘어오면 마스킹으로도 못 고친다. "이 스킬이 사내 위키 접근을 요구한다"는 귀속 오류가
#  그대로 남기 때문이다. 실측 라운드 6에서 관측된 오염 경로다.)
rona_main | jq -r 'select(.type == "user") | .message.content
  | if type == "string" then . else (.[]? | select(.type == "text") | .text) end' \
  | grep -oE 'https?://[A-Za-z0-9._~:/?#@!$&*+,;=%-]+' \
  | sed -E 's/[.,;:!?)]+$//' | sort -u
# URL 경계에 주의 — `[^ )]+` 로 끊으면 **한국어 조사와 문장부호가 URL에 들러붙는다.**
# 한국어는 URL 뒤에 공백 없이 조사를 붙이는 게 정상 표기라(`…/2026-06에서`), 그 상태로
# `sort -u` 를 타면 같은 링크가 `…/2026-06`·`…/2026-06에서`·`…/2026-06,` 로 갈라져
# **출처 1건이 3건으로 부풀려진다**(실측 라운드 9 재현). 그래서 URL 허용문자만 받고
# 끝에 남은 문장부호를 떼어낸다.

# 실제로 가져온 URL (WebFetch) — 서브에이전트(조사원) 포함해야 한다.
# 조사원 파견형 스킬은 외부 출처가 거의 전부 서브에이전트 쪽에만 있다.
rona_logs | jq -r 'select(.type == "assistant") | .message.content[]?
  | select(.type == "tool_use" and .name == "WebFetch") | .input.url' | sort -u

# 검색 키워드 (WebSearch) — 이건 "출처"가 아니라 "무엇을 찾으려 했는가"다.
# 위 URL 목록과 절대 합치지 마라: 합치면 실제 출처 3건이 5건으로 부풀려진다(실측 확인).
rona_logs | jq -r 'select(.type == "assistant") | .message.content[]?
  | select(.type == "tool_use" and .name == "WebSearch") | .input.query' | sort -u
```

**정리 형식**:
```markdown
## 외부 출처

- https://vercel.com/docs/ai-sdk — useStream 옵션 확인
- https://github.com/anthropics/sdk — 토큰 카운팅 예시
- (사용자 메시지에서 발견된 사내 위키 등은 마스킹 대상)
```

#### (3) 시도·실패·전환

**출처**: 같은 작업의 반복 호출 + 에러 메시지 + "안 되네", "다시 해보자", "다른 방법" 같은 사용자 신호

자동 추출은 **휴리스틱**: 시간 순으로 묶어 "처음 시도 → 막힘 → 우회"의 3단 시나리오 N개를 만든다.

```bash
# 에러를 포함한 toolResult 블록 (서브에이전트 안에서 난 실패도 포함).
# `.timestamp` 는 최상위 레코드에 있고 tool_result 블록엔 없다 — content[] 로 내려가기 *전에*
# `as $ts` 로 잡아둬야 한다. (`{timestamp: .}` 로 쓰면 블록 자신이 복제될 뿐이라 소요시간
# 추정이 불가능해진다 — 실측 확인된 함정.)
rona_logs | jq -c 'select(.type == "user")
  | .timestamp as $ts
  | .message.content[]?
  | select(.type == "tool_result" and (.content // "" | tostring | test("error|Error|ERROR|failed|Failed")))
  | {timestamp: $ts, text: (.content | tostring)}' | head -20
```

> **에러 jq만 돌리고 끝내지 마라 — 그러면 "조용한 실패"가 통째로 사라진다.** 위 명령은 출처
> 3가지 중 *에러 메시지* 하나만 잡는다. 도구는 성공(exit 0)했는데 결과가 틀려서 파트너가
> "이거 아닌데, 다시"라고만 한 경우 **0행**이 나오고, 그러면 STEP 3-3의 drill 판정이 "로그
> 깨끗함"으로 기울어 `drill_skipped=true`가 잘못 기록된다. 이 스킬이 가장 중요하다고 선언한
> *만족도와 실제 노력의 갭* 신호를 정확히 그 상황에서 놓치는 것이다. 아래 둘을 같이 돌린다:

```bash
# (a) 파트너의 전환·불만 발화 — 에러 없이 조용히 틀린 경우는 이것만 남는다.
rona_main | jq -r 'select(.type == "user") | .timestamp as $ts | .message.content
  | (if type == "string" then . else ([.[]? | select(.type=="text") | .text] | join(" ")) end)
  | select(test("안 ?되|안됐|다시|틀렸|아닌데|아니고|말고|바꿔|대신|이상해|왜 이|다른 방법|직접 (했|하)"))
  | "\($ts) \(.[0:100])"'

# (b) 같은 도구 연속 반복 — "막혀서 계속 두드린" 구간. 3회 이상 연속이면 시나리오 후보다.
# **반드시 시간 구간을 같이 찍는다.** 이름만 세면 사이에 낀 발화·수 시간의 공백이 전부
# 무시돼, 몇 시간 떨어진 별개 호출이 "연속 3회"로 잡힌다(실측 라운드 9 재현: 3시간 20분
# 떨어지고 주제까지 바뀐 Read 3건이 "3회 연속"으로 나왔다). 구간이 보이면 실행자가
# 눈으로 걸러낼 수 있다.
rona_logs | jq -r 'select(.type == "assistant") | .timestamp as $t | .message.content[]?
  | select(.type == "tool_use") | "\(.name)\t\($t)"' \
  | awk -F'\t' '
      function flush(){ if(n>=3) printf "%d 회 연속 — %s (%s ~ %s)\n", n, cur, s, e }
      NR==1 { cur=$1; n=1; s=$2; e=$2; next }
      $1==cur { n++; e=$2; next }
      { flush(); cur=$1; n=1; s=$2; e=$2 }
      END { flush() }'
```

> (a)의 정규식은 **한국어 전환 신호 휴리스틱**이라 과탐·미탐이 다 있다. 잡힌 줄은 시나리오
> *후보*일 뿐이니 앞뒤 맥락을 읽고 진짜 전환인지 판단해 쓴다. 0행이라고 "안 막혔다"로
> 단정하지 마라 — 파트너가 말없이 직접 손봤을 수 있다(그건 STAGE 1 [2/3]에서 받는다).
>
> **(b)의 "연속 N회"를 곧바로 "막힘"으로 읽지 마라 — 성공한 순회가 훨씬 많다.** 먼저 같이 찍힌
> **시간 구간**을 본다. 구간이 몇 시간에 걸쳐 있으면 그건 연속 호출이 아니라 그냥 그 도구를
> 띄엄띄엄 여러 번 쓴 것이다(막힘 아님). 구간이 짧아도, 같은 도구가 실측(라운드 7): `6회 연속 — Read`가 잡혔는데
> 서브에이전트가 `sample_0.txt`~`sample_5.txt`를 순서대로 정독한 것이었고, 정작 진짜 실패
> (yaml 파싱)는 (3a)에 1건으로만 있었다. **판정 기준은 "반복 그 자체"가 아니라 그 구간에
> (3a) 에러나 (a) 전환 발화가 겹치는가**다. 겹치지 않는 반복(Read/Glob 순회, 배치 처리)은
> 시나리오에서 뺀다 — 안 그러면 최대 5개 슬롯을 가짜 막힘이 차지해 진짜 실패가 밀려나고,
> drill이 *"샘플 6개 읽는 데 막히셨죠?"* 로 나가 파트너를 혼란시킨다.

**정리 형식**:
```markdown
## 시도·실패·전환 (자동 추출)

[1] 초기 generateObject Pro 시도 → schema 검증 실패 → Flash로 전환 (총 12분)
[2] WebSearch 키워드 X로 5회 검색 → 적절한 답 못 찾음 → 키워드 Y로 재검색 (총 18분)
[3] tool_use 결과 파싱 실패 → 정규식 3차례 수정 → 통과 (총 8분)
```

- 각 시나리오에 **소요 시간** 추정 (관련 메시지의 timestamp 차이)
- 최대 5개. 더 많으면 임팩트 큰 순서로 절단

### 2-4. 출력 — 중간 산출물

위 3개를 묶어 `./.rona-review-auto-<practice_id 앞 8자>.md` 임시 파일로 저장 (다음 STEP에서 인터뷰 보강용).
**파일명에 practice_id를 넣는 이유**: 같은 폴더에서 여러 로나 스킬을 쓰면 고정 파일명은 직전
스킬의 리뷰 잔여물을 이번 리뷰가 읽어버려 **다른 스킬 데이터로 오염**된다. 같은 이름이 이미
있으면 이번 실행 산출로 **덮어쓴다**(이어붙이지 마라 — 재실행 시 중복 누적).

---

## STEP 3. 인터뷰 — 자기기입 + 조건부 AI 보강 2단계

> 인터뷰는 **두 단계로 끊김 없이** 진행한다. STAGE 1은 **3문항**으로 짧게 — "안 막혔으면 빨리 끝, 막혔으면 그 부분만 더". STAGE 2는 **조건부** — 로그에 우회·재시도 흔적이 있거나 [2/3]에 막힘이 적혔을 때만 1회 drill, 흔적이 전혀 없으면 skip. 안 막힌 사람은 2~3분, 막힌 사람만 4~5분.

### 3-0. 동작 원칙 (반드시 준수)

- **AskUserQuestion 도구 사용 금지.** 텍스트로 한 문항씩 순서대로 묻는다. 한 응답이 들어오기 전에는 다음 문항을 던지지 않는다.
- **인터뷰 문항은 내가 절대 대신 답하지 않는다.** 각 문항은 출력한 뒤 멈춰 **사용자의 실제 응답을 기다린다.** 세션에서 이미 답을 안다고 요약해 채우거나 그럴듯한 답을 지어내지 말 것 — 내가 값을 만들어 넣고 다음 STEP(마스킹·발송)으로 넘어가면 그건 위반이다. 사용자가 답하지 않으면 그 필드는 빈 값/`null` 이지, 내가 지어낸 한 줄이 아니다. (특히 STAGE 2 Q4 `automation_value` 는 세션 내내 도운 직후라 confabulation 고위험 — 반드시 사용자 입으로 받는다.)
- 빈 응답·"패스"·"모름"·Enter-only는 valid 신호. 다음 문항으로 진행한다.
- **척도 라벨은 코드블록 안의 문자 그대로 완성된 문장으로 출력한다.** "1. 잘 맞" 처럼 종결어미를 자르거나 단축하지 말 것 — 파트너가 "말을 하다 마는데?" 같은 인상을 받으면 응답 품질이 떨어진다.
- **"왜?" 질문은 직전 인정 행동에 묶일 때만** — 무의식적 도구 선택은 confabulation 위험 (Nisbett & Wilson 1977).
- **1~10점·만족도 평가·stated preference 척도 금지** — 정본 §목표 "탐색, 합/불 아님". "쓸 거예요?" "통하나요?" 같은 미래 가정 척도는 답정너로 수렴 — 모든 문항은 *행동·사실 회수*에만 집중.
- STAGE 1 입력 전에 STEP 2 자동 추출 결과를 보여주지 않는다 — cue가 들어가면 episodic recall이 오염 (Tulving 1973).
- **불변식 — 설문 단축 여부는 사람 답이 아니라 로그로 판단한다.** [2/3]에 파트너가 "없음 / 괜찮았어요"라고 답해도, STEP 2 자동 추출 로그에 우회·재시도·다른 도구 흔적이 있으면 **무조건 STAGE 2 drill을 1회 한다**. '괜찮았다'는 자기보고를 그대로 믿고 STAGE 2를 건너뛰면 안 된다 (만족도와 실제 노력의 갭 = 핵심 신호). 로그가 깨끗할 때만 skip한다.

### 3-1. 회수 5축 — 인터뷰가 무엇을 모으는가

| 축 | 무엇 | 어디서 회수 | 4차원 병합 |
|---|---|---|---|
| ④ 맞춤 정렬 | 입력 vs 실제 스킬 정렬 | STAGE 1 [1/3] | 신규 (Initiative #3) |
| ① 아쉬운 점(갭) | 로나 스킬 부족분 | STAGE 1 [2/3] 앞부분 → `regret` | 4차원 4번 |
| ② 채운 노력 | 갭을 메우려 한 행동 | STAGE 1 [2/3] 뒷부분 + drill + 로그 → AI 추출 `actions`, 서사는 `regret_resolution` | 4차원 3번 |
| ③ 방법·출처 | URL·도구·사람 | STEP 2 로그 자동 + STAGE 2 drill → `sources_raw` | 4차원 2번 |
| 다음 사람 메모 | 로나가 더 해주면 좋을 것 | STAGE 1 [3/3] → `next_person_memo` | 4차원 4번 (건설 프레이밍) |
| 놓친 행동 | 자동 추출에 있는데 자기기입에 없는 우회 | STAGE 2 drill | 4차원 3번 보강 |

> ⑤ 일반화·⑥ 자동화 가치(stated preference)는 *행동 데이터로 계산* — Week4 리텐션 base는 실제 재방문 데이터, 자동화 가치는 ②③ raw material로 합집합 자동화에 직접 기여. 단 "클로드코드보다 로나라서 도움된 점"은 STAGE 2 Q4(선택)에서 *행동 사실*로만 한 줄 회수한다 (척도 아님).

### 3-2. STAGE 1 — 자기기입 (2~3분, 3문항)

STEP 2 자동 추출(`./.rona-review-auto-<practice_id 앞 8자>.md`) 저장 직후, 결과는 보여주지 말고 곧장 STAGE 1로 들어간다.

**[자기기입 1/3] 맞춤 정렬 ④** — 다음 문구를 그대로 파트너에게 출력하고 응답 대기:

```
잠깐 2~3분만 같이 정리할게요. 첫 번째 —

1/3. 처음 "이런 거 해보고 싶다"고 적으신 거랑, 받은 스킬이 잘 맞던가요?

  1. 잘 맞았어요
  2. 일부만 맞았어요 — 어디가 빗나갔어요?
  3. 거의 안 맞았어요 — 어디가요?

  (2·3 선택 시 한 줄로 어디가 빗나갔는지 같이 적어주세요)
```

**[자기기입 2/3] 막힘 + 직접 손봄 ①②**:

```
2/3. 막혔거나, 스킬 결과를 직접 더 손본 데가 있었나요?
     없으면 "없음", 있으면 그 지점만 짧게 적어주세요.
     (예: "영상 파일을 못 구해줘서 직접 유튜브에서 받아서 넣었어요")
```

> 이 문항은 **막힘(갭)** 과 **메우려 한 행동(해결)** 을 한 번에 받는다. 둘은 META에서 분리한다 (STEP 6 — `regret` vs `regret_resolution`). 여기서 막힘을 적으면 [3/3] 후 STAGE 2 drill 대상이 된다.

**[자기기입 3/3] 다음 사람 메모 (건설 프레이밍)**:

```
3/3. 다음에 같은 스킬을 쓸 사람에게 더 도움이 되려면,
     로나가 뭘 더 해주면 좋을까요? (한 줄)
     (떠오르는 게 없으면 Enter)
```

응답 3건을 받으면 STAGE 2 진입 여부를 판정한다.

### 3-3. STAGE 2 — 조건부 AI 보강

STAGE 1 [2/3] 응답 + STEP 2 자동 추출을 대조해 **drill 실행/skip을 판정**하고, 실행 시 사람이 놓친 행동·출처를 한 문항으로 회수한다.

#### drill 실행/skip 판정 (불변식 §3-0 적용)

**drill을 한다 (`drill_skipped=false`)** — 둘 중 하나라도 참이면:
- (a) STAGE 1 [2/3]에 막힘·직접 손본 지점이 적혔다 (비어있지 않음), **또는**
- (b) [2/3]가 "없음"이어도 STEP 2 자동 추출 로그에 **우회·재시도·다른 도구 흔적**이 있다 — 같은 작업 반복 호출, 에러 후 재시도, WebSearch 폭주(정상치 2~3배), 외부 LLM 언급, 도구 전환 등. (만족도와 실제 노력의 갭은 핵심 신호 — '괜찮았다'를 그대로 믿지 않는다.)

**skip한다 (`drill_skipped=true`)** — 다음을 *모두* 만족할 때만:
- [2/3]가 비었거나 "없음"이고, **그리고** 자동 추출 로그에 우회·재시도·다른 도구 흔적이 전혀 없다.
- skip하면 `drill_skip_reason`에 사유를 한 줄로 적는다 (예: "막힘 없음 + 로그에 재시도·우회·외부도구 흔적 없음"). `drill_response`는 null.

자동 추출 시나리오 선별 우선순위(여러 흔적 중 무엇을 인라인 박을지): 재시도 횟수 > 외부 리서치 도구 폭주 > 외부 LLM 언급 > 우회 존재 > 도구 전환 횟수.

#### drill 문항 (실행 시에만)

자기기입 단계에서 보여주지 않았던 자동 추출 결과를 *이 문항 안에 자연어 한 줄*로 인라인 박는다. 메타 용어("임팩트 상위 시나리오", "WebSearch", "context7" 등)는 한국어로 풀어쓴다 — "**웹 검색 N번 / 외부 블로그 N개 정독에 N분**" 같이.

```
로그를 봤더니, 이번 작업에서 [웹 검색 8번 / 외부 블로그 3개 정독에 25분]쯤
쓰셨더라고요.

그 25분 동안 뭘 찾아가셨는지, 기억나는 범위에서 단계별로 꼼꼼하게
풀어주세요. 한두 줄 요약 말고, 1·2·3… 순서대로 적어주시면 됩니다.

  예)
  1. 처음에 ___ 키워드로 검색 → 결과가 어색
  2. ___ 로 바꿔서 다시 검색 → ___ 블로그 발견
  3. 그걸 보고 "아, 이렇게 만들면 되겠다" 방향 잡힘
  ...

아까 안 적은 행동·출처(다른 LLM·동료·문서)도 함께 알려주세요.
```

- "어떻게 찾아가셨어요?" — *방법* 회수가 핵심. narrative 깊이를 받는다.
- 응답에 "왜 그렇게 됐는지"가 자연스럽게 드러나지 않으면 **한 번만** follow-up 가능. 두 번 이상 캐묻지 않는다 (Nisbett & Wilson confabulation 회피).
- [2/3]가 이미 풍부하고 자동 추출 결과와 큰 갭이 없으면 follow-up 없이 종료 가능.

#### Q4 — 로나 차별 가치 (선택, 행동 사실 1줄)

drill 실행/skip과 무관하게, 마지막에 한 줄 선택 문항:

```
마지막으로 — 클로드코드만 썼을 때보다 로나라서 도움이 된 점이 있다면
한 줄로 알려주세요. (없으면 Enter)
```

- 응답 → `stage2_ai_probe.automation_value`. 없으면 null.
- **이 한 줄은 반드시 사용자가 답한다 — 내가 대신 채우지 않는다.** 세션에서 "도움된 점"을 내가 요약해 넣으면 가짜 차별가치 신호로 측정이 오염된다. 문항을 출력하고 멈춰 응답을 기다리고, Enter·빈 응답이면 `null` — 지어낸 한 줄을 절대 넣지 않는다.
- **stated preference 아님** — "쓸 거예요?"가 아니라 "이번에 도움이 됐던 사실"만 받는다. 미래 가정·만족도 척도 금지.

> v3의 Tier B 자유 한 줄 섹션은 제거됐다. 자유 한 줄은 Q4가 흡수한다. META의 `tier_b_wish`는 항상 null.

### 3-4. 절대 묻지 말 것 (학술 근거)

- ❌ **"왜 그 우회가 필요했나요?"** — 무의식적 도구 선택의 confabulation 위험 (Nisbett & Wilson 1977).
- ❌ **"이 스킬을 1~10점으로 평가하면?"** — 정본 §목표 "탐색, 합/불 아님".
- ❌ **"몇 분 걸렸나요?" 단독 자유 입력** — telescoping (Loftus & Marburger 1983).
- ❌ **"이 부분이 어떻게 개선됐으면 좋을까요?"를 비판 프레이밍으로 필수** — Young 분당 수율 최저. [3/3]은 "다음 사람에게 더 도움되려면"이라는 *건설 프레이밍*으로만 묻는다.
- ❌ **"자동화되면 쓰실 거예요?" "비슷한 일에 통하나요?" stated preference 척도** — acquiescence bias + stated vs revealed gap. 행동 데이터로 *계산*한다. (Q4도 미래 가정이 아니라 *이번 작업의 사실*만 받는다.)

### 3-5. 자기기입 ↔ 자동 추출 ↔ 보강 응답 3-way 대조 — 보존 규칙

이 인터뷰의 진짜 가치는 **세 인풋의 불일치**에 있다. 분석 단계에서 잡기 위해 STEP 4 마스킹 전에 다음을 보존한다:

| 불일치 패턴 | 의미 | 보존 방식 |
|---|---|---|
| STAGE 1 [2/3]에 없는 행동이 자동 추출 로그에 있음 | 의식하지 못한 채우기 작업 | STAGE 2 drill 응답에 그대로 기록 |
| STAGE 2 drill에 나온 출처가 자동 추출 로그에 없음 | 세션 외 행동 (다른 LLM·사람·외부 도구) | STAGE 2 drill 응답에 그대로 기록 |
| STAGE 1 [2/3] "없음"인데 로그에 재시도·우회 흔적 풍부 | 만족도와 실제 노력의 갭 — "AI 가독성" 신호. **drill skip 금지** | drill 강제 + 응답 전체 보존 |
| STAGE 1 [1/3] "잘 맞"인데 [2/3] "막힘 있음" | 정렬 만족과 부족분의 비대칭 — 입력↔결과 갭과 스킬 자체 한계 분리 단서 | 응답 전체 보존 |

불일치는 **제거하지 말고 메타 코멘트에 그대로 남긴다** (STEP 6).

### 3-6. 학술 근거 + 사이클 예시

설계 근거 5가지 이슈(회상 편향·노력 간극·단계별 vs 종합·사후 합리화·부담 vs 깊이)와 학술 출처, v4 3문항+조건부 drill 사이클 입출력 예시, 첫 2~3 사이클 파일럿 회고 항목은 [`references/interview-design.md`](references/interview-design.md)에 정리. STEP 3 동작 중 인터뷰 문구 조정이 필요하거나, 응답 분포가 이상하게 쏠릴 때 참조한다.

### 3-7. 인터뷰 결과 → 4차원 + 5축 병합

STAGE 1 + STAGE 2 응답을 종합해 STEP 2의 자동 추출 md를 다음 구조로 갱신한다:

- **4차원 1번 (사용 도구·스킬)**: 자동 추출 그대로
- **4차원 2번 (외부 출처)**: STEP 2 로그 자동 + STAGE 2 drill 보강 → `sources_raw`
- **4차원 3번 (시도·실패·전환)**: 자동 추출 + STAGE 1 [2/3] 해결 행동(`regret_resolution`) + STAGE 2 drill에서 **AI가 추출한 행동 배열** → `actions` (`actions_method="ai_extracted"`)
- **4차원 4번 (못 미친 지점)**: STAGE 1 [2/3] 막힘(`regret`) + [3/3] 다음 사람 메모(`next_person_memo`)
- **5축 ④ 맞춤 정렬**: STAGE 1 [1/3] — 4차원과 별개로 메타 코멘트(STEP 6)에 구조화 보존 — Initiative KR 분석용

---

## STEP 4. 민감 정보 마스킹

LLM(스킬을 실행하는 Claude 자신)이 직접 판단해 마스킹한다. 외부 API 호출 없음.

### 마스킹 카테고리

| 카테고리 | 치환 |
|---|---|
| 이메일 | `[EMAIL_MASKED]` |
| 전화번호 | `[PHONE_MASKED]` |
| API 키 (sk-, ghp_, xoxb-, AKIA 등) | `[API_KEY_MASKED]` |
| IP 주소 | `[IP_MASKED]` |
| 주민등록번호 | `[ID_MASKED]` |
| 사내 프로젝트명 / 코드명 | `[INTERNAL_PROJECT_MASKED]` |
| 고객명 / 거래처명 (개인·회사) | `[CUSTOMER_MASKED]` |
| 내부 URL (사내망·VPN·관리자 페이지) | `[INTERNAL_URL_MASKED]` |
| 금액·매출 등 영업 비밀로 보이는 수치 | `[FIGURE_MASKED]` |

### 판단 원칙

- 외부에 공유되면 곤란할 가능성이 조금이라도 있으면 **마스킹 우선**
- 공개된 URL (vercel.com, github.com, docs 사이트)은 그대로 유지
- 모호하면 `[REDACTED: 사유]` 형태로 사유 명시
- **마스킹 토큰 옆에 원본을 함께 표시·설명·로그하지 말 것.** 치환은 완전 치환이지 비교 텍스트가 아니다. (예: `[EMAIL_MASKED] (원본: foo@bar.com)` 같은 표기 절대 금지)

### 절차

1. STEP 2 + STEP 3에서 합쳐진 md 전체를 검토
2. 카테고리별로 치환
3. 결과를 `./rona-review-preview.md`로 저장
4. **정규식 사후 가드 (반드시 실행)** — LLM 마스킹이 누락한 정형 패턴(이메일/전화/API키/IP/주민번호)을 결정론적으로 한 번 더 훑는다:

```python
python3 -c "
import re
md = open('./rona-review-preview.md').read()
# 가드 설계:
# 1) 긴 토큰(API 키)을 먼저 처리 — sk- 키 내부의 우연한 숫자 시퀀스를 PHONE이
#    부분 매칭하지 못하도록 (예: sk-...7890123456789 가 두 토큰으로 쪼개지는 버그).
# 2) PHONE/IP/ID는 (?<!\\d)...(?!\\d) 숫자 경계로 가드. \\b는 Python unicode 모드에서
#    한글도 word character로 보기 때문에 '010-1234-5678입니다'·'192.168.1.1과' 같은
#    한국어 인접 케이스를 못 잡음 — 숫자 경계가 한글·영문·공백 어디든 안전.
patterns = [
  (r'[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}', '[EMAIL_MASKED]'),
  (r'(sk-|ghp_|xoxb-|AKIA)[A-Za-z0-9_-]{20,}', '[API_KEY_MASKED]'),
  (r'(?<!\d)01[0-9]-?[0-9]{3,4}-?[0-9]{4}(?!\d)', '[PHONE_MASKED]'),
  (r'(?<!\d)[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}(?!\d)', '[IP_MASKED]'),
  (r'(?<!\d)[0-9]{6}-[1-4][0-9]{6}(?!\d)', '[ID_MASKED]'),
]
for p, t in patterns: md = re.sub(p, t, md)
open('./rona-review-preview.md', 'w').write(md)
print('PII regex sweep done')
"
```

- 이 단계는 **건너뛸 수 없음**. LLM 변덕에도 정형 패턴은 100% 잡힘
- 사내 단어·고객명·내부 URL 같은 LLM 판단 영역은 정규식이 못 잡으므로, **파트너 컨펌 미리보기가 최종 안전망**

---

## STEP 5. 미리보기 + 파트너 컨펌

> 발송이 종착지다. 취소 옵션은 두지 않는다 — 마스킹이 미덥지 않으면 파일을 직접 고치고, 만족하면 발송한다. 파트너가 검토할 시간은 충분히 보장하되, "안 보내고 도망갈 길"을 제도화하지는 않는다.

### 출력

```
마스킹된 후기를 ./rona-review-preview.md 에 저장했어요.
파일을 열어 확인해주세요. 민감 정보가 남아 있으면 직접 편집해도 됩니다.

요약:
- 스킬: <title>
- 사용 도구: N개
- 외부 출처: N개
- 시도·실패·전환: N개 시나리오
- 인터뷰: STAGE 1 3문항 + STAGE 2 조건부 drill(실행/건너뜀) + Q4 답변 완료

발송 준비됐어요. 어떻게 하실래요?
  [1] 발송
  [2] 수정 (파일 직접 편집 후 '준비됨' 입력)
```

### 분기

- **[1] 발송**: STEP 6 진행
- **[2] 수정**: 파트너가 `./rona-review-preview.md` 편집 → "준비됨" 입력 → Read로 다시 읽기 → 미리보기 요약 재제시 → 다시 [1] or [2]. 만족할 때까지 무한 수정 가능하지만 **종착지는 항상 발송**이다.

---

## STEP 6. 로나에 전송

전용 엔드포인트 `POST /api/skill-reviews` 사용. 파트너 자문단 리뷰는 `rona.partner_skill_reviews` 테이블에 적재된다. (이전에는 v1 `/api/devlogs` 에 기생했으나, ID 공간 어긋남 + 의미 충돌로 분리됨.)

### 메타데이터 HTML 코멘트 삽입

마스킹된 md 맨 위에 다음 코멘트를 박는다 (v1 파서는 HTML 코멘트 무시 → 회귀 0):

```html
<!-- RONA_REVIEW_META
{
  "version": "rona-review-v4-3q",
  "skill_title": "<title>",
  "stage1_self_report": {
    "fit_alignment": {
      "score_1_to_3":  2,
      "what_missed":   "STAGE 1 [1/3] free-text (2·3 선택 시)"
    },
    "regret":            "STAGE 1 [2/3]에서 막힌/부족한 '갭'만 (해결 행동 절대 섞지 말 것)",
    "regret_resolution": "STAGE 1 [2/3]에서 그 갭을 메우려 한 '해결 행동' 서사 (신규)",
    "next_person_memo":  "STAGE 1 [3/3] — 다음 사람 위해 로나가 더 해줬으면 하는 것 (신규)",
    "actions":           ["AI가 추출한 행동 배열 ([2/3] 해결서사 + drill + 로그)"],
    "actions_method":    "ai_extracted",
    "sources_raw":       "로그 + drill 자동 도출 출처 (사람에게 안 물음). 없으면 null"
  },
  "stage2_ai_probe": {
    "scenario_summary":  "AI가 보여준 자동 추출 자연어 한 줄",
    "drill_response":    "STAGE 2 drill 응답 또는 null (skip 시)",
    "drill_skipped":     false,
    "drill_skip_reason": "skip 사유 또는 null (신규)",
    "automation_value":  "Q4(선택) — 클로드코드보다 로나라서 도움된 점 또는 null (신규)",
    "tier_b_wish":       null
  },
  "axis_summary": {
    "axis4_fit":        "...",
    "axis1_regret":     "갭(막힘)만 — 해결 행동은 axis2_actions로",
    "axis2_actions":    "갭을 메우려 한 행동·해결 요약",
    "axis3_sources":    "..."
  },
  "discrepancies": [
    "예: STAGE 1 [2/3]에 없는 행동이 자동 추출에 있음 — 웹 검색 8회",
    "예: STAGE 2 drill의 'ChatGPT o3'가 세션 로그에 없음"
  ],
  "session_file": "<basename>",
  "masked_count": N,
  "submitted_at": "ISO8601",
  "completion_source": {
    "source": "closing",
    "trigger_shown": false,
    "dropout_reason": null
  }
}
-->
```

> **⚠️ 데이터 오염 방지 (절대 위반 금지) — `regret`·`axis1_regret` 은 갭만, 해결 행동은 분리.**
> [2/3]는 막힘과 해결 행동을 한 문장으로 받는다. 이를 META로 옮길 때:
> - `regret` 과 `axis_summary.axis1_regret`(요약된 아쉬운 점) 에는 **막힌/부족한 '갭' 만** 넣는다. 해결 서사("직접 ~해서 해결", "~를 받아서 넣음")는 **절대 넣지 않는다.**
> - 그 해결 행동 서사는 `regret_resolution` 과 `axis_summary.axis2_actions` 에 따로 넣는다.
> - 한 문장에 갭과 해결이 교차돼 있으면 **절(節) 단위로 갈라** 갭 절만 `regret`/`axis1_regret`에, 해결 절만 `regret_resolution`/`axis2_actions`에 넣는다.
> - 예1: 입력 "영상 파일을 못 구해줘서 직접 유튜브에서 받아서 넣었어요" →
>   `regret`/`axis1_regret`="스킬이 분석할 영상 파일을 못 구해줌", `regret_resolution`/`axis2_actions`="직접 유튜브에서 받아서 해결".
> - 예2(교차): 입력 "결과 톤이 안 맞아서 ChatGPT로 다시 받아 직접 고쳤어요" →
>   `regret`/`axis1_regret`="결과 톤이 기대와 안 맞음"(갭 절만), `regret_resolution`/`axis2_actions`="ChatGPT로 다시 받아 직접 수정"(해결 절만).
> 개선 루프(`gap-source`)는 `regret` **과 `axis_summary.axis1_regret`** 을 '제품 갭 신호'로 읽는다(둘 다 갭 채널). 게다가 `axis1_regret` 은 `ingest-review` 가 후기 → feedbacks triage 의 제품 개선 신호(severity/priority/group)로도 쓴다. 해결 행동(성공 서사)을 `regret`이나 `axis1_regret`에 합치면 그게 갭으로 오분류돼 개선 제안과 triage 를 동시에 오염시킨다. `actions`/`next_person_memo`/`automation_value`/`regret_resolution`/`drill_skip_reason` 은 gap-source가 읽지 않으므로 안전하다.

### 전송 절차

1. `./rona-review-preview.md`를 Write로 그대로 저장 (heredoc 금지)
2. **STEP 1에서 추출한 마커의 `practice_id` 값을 `install_token` 으로 박아 넣어** Python으로 POST:

```bash
# 예시 — STEP 1에서 읽은 마커 파일의 practice_id 값(실제는 install_token)을 직접 박는다
python3 -c "
import json, urllib.request
md = open('./rona-review-preview.md').read()
data = json.dumps({
  'install_token': '<STEP_1_INSTALL_TOKEN>',
  'tool_used': 'rona-review',
  'markdown_content': md
}).encode()
req = urllib.request.Request('https://rona.so/api/skill-reviews',
  data=data, headers={'Content-Type': 'application/json'})
res = urllib.request.urlopen(req)
print(res.read().decode())
"
```

- API URL은 production 고정: `https://rona.so/api/skill-reviews`
- 마커 파일의 키 이름은 호환을 위해 `practice_id` 그대로 유지 — 값은 실제 `install_token` 이며 엔드포인트는 `install_token`/`practice_id` 둘 다 받음
- `student_token` 필드는 보내지 않는다 — 파트너 자문단 리뷰는 사람 식별이 목적이 아니라 스킬에 대한 피드백이므로
- Windows에서 `python3`이 없으면 `python` 또는 `py`로 대체

### 성공 시

```
발송 완료. 로나에 안전하게 도착했어요. 고맙습니다.
```

발송이 200 OK 로 확인된 *그다음*, 파트너가 리뷰 중에 작업성 피드백(미완 작업·더 했어야 할
것)을 남겼다면 **딱 한 번** "원래 작업을 이어서 도와드릴까요?" 하고 별도로 제안한다. 파트너가
원할 때만 원래 흐름으로 복귀하고, 묻지 않은 채 임의로 작업을 재개하지 않는다. (작업성
피드백이 없었으면 이 제안도 생략하고 깔끔히 마친다.)

### 실패 시 fallback

```
발송에 실패했어요. 수동 업로드 URL로 보내주세요:
https://rona.so/upload?install_token=<STEP_1_INSTALL_TOKEN>

위 URL에서 ./rona-review-preview.md 파일을 업로드하면 됩니다.
```

(업로드 페이지가 `install_token` 쿼리를 아직 처리하지 않으면 운영자가 Neon Console에서 수동 적재.)

---

## 데이터 흐름 — 다운스트림 없음 (수집 전용)

`POST /api/skill-reviews` 는 `rona.partner_skill_reviews` 테이블에 row 만 적재한다.
v1 devlogs 의 후속 플로우(체크포인트 동기화, 챌린지 인증, Mixpanel `devlog_upload`, Slack `## 소감` 알림)는 **호출되지 않는다** — 파트너 자문단 리뷰는 학생 진도가 아니라 스킬 자체에 대한 피드백이므로 의도된 격리. 분석은 운영자가 Neon Console 직접 조회 또는 추후 신설될 `/admin/skill-reviews` 페이지에서 수행.

RONA_REVIEW_META HTML 코멘트는 엔드포인트에서 파싱되어 `stage1_self_report` / `stage2_ai_probe` / `axis_summary` / `discrepancies` / `session_file` / `masked_count` / `submitted_at` / `completion_source` 컬럼으로 분해 저장된다. 파싱 실패 시에도 `markdown_content` 는 무손실 보존.

`completion_source` 는 이 후기가 어디서 왔는지를 호출 맥락으로 판정해 채운다. 끝까지 가지 않고 중간에 마친 흐름(설치 스킬의 "이거면 됐어요 / 다른 방식으로 갈게요" 종료 후 회수)에서 호출됐으면 `source` 를 `"stopword"` 로 두고, 들은 이탈사유 한 줄을 `dropout_reason` 에 담고 `trigger_shown` 을 `true` 로 둔다. 정상적으로 완주하고 마무리 후기로 넘어온 흐름이면 `source` 를 `"closing"`, `trigger_shown` 을 `false`, `dropout_reason` 을 `null` 로 둔다. 이 필드는 출처 분류 1개만 채우며, 설문 4축(사용 도구 / 외부 출처 / 시도와 실패 / 아쉬운 점) 내용은 손대지 않는다.

`dropout_reason` 한 줄은 META 가 in-band JSON 이라는 점을 지켜 안전하게 적는다. 큰따옴표(`"`)·개행은 한 줄 평문으로 풀어 쓰고(필요하면 작은따옴표나 풀어쓴 표현으로 바꾼다), JSON 종결자와 겹치는 `-->` 같은 문자열은 넣지 않는다. JSON 문자열로 안전히 이스케이프된 한 줄이라야 META 전체가 깨지지 않고 후기 구조화 필드가 보존된다.

---

## 주의사항

- 파트너 컨펌 미리보기는 협상 불가능 — "발송" 명시 전에는 절대 API 호출 안 함
- 마스킹 누락 사후 발견 시 운영자가 Neon Console에서 `UPDATE rona.devlogs SET raw_markdown = '[REDACTED — partner request]' WHERE id = '...'`로 수동 처리
- 인터뷰 답변은 파트너가 비워도 됨 (선택 가능)
- 같은 폴더에 여러 로나 스킬이 install된 경우 마커 파일로 구분 — 파트너가 선택
- bash heredoc(`cat << EOF`) 사용 금지 (Windows 비호환)
- `AskUserQuestion` 도구 사용 금지 — 텍스트로 한 문항씩 순서대로 묻는다
