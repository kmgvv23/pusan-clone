# #106 week3: SQLite 저장/조회/수정/삭제 tool 구현

원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/106

작성자: @kmgvv23
작성 시각: 2026-07-15T13:01:08Z
보관 시각 (UTC): 2026-10-06T15:19:25.015347+00:00

## PR 본문

## 과제 목표

- Week2에서 구조화한 요청을 실제로 SQLite에 저장해서, 앱을 껐다 켜도 남는 기록장을 만들어보기
- 저장한 일정을 다시 조회하고 수정/삭제하는 흐름까지 연결해보기

---

## 과제 위치
- 작업 브랜치 : `kimmingyu/week3`  → 본인 통합 브랜치 `kimmingyu/final` 로 PR
- 주요 파일 : `student_parts/week03_build_nanas_logbook.py`

> 제공 코드(`fixed/` 등)는 수정하지 않고, `student_parts/` 의 본인 주차 파일을 구현했습니다.

---

## 과제 범위

- [x] 메인 과제 완료
- [x] 심화 과제까지 완료

---

## 구현한 기능 (메인)

- [x] `save_structured_request` — Week2 구조화 결과를 검증(args_schema) 후 SQLite 저장
- [x] `list_saved_requests` / `get_saved_request` — 저장 요청 목록/단건 조회
- [x] `personal_list_saved_schedules` — 저장된 일정 목록 조회 (kind 기본값 personal_schedule)

---

## 심화 과제

- [x] `personal_update_saved_schedule` — 저장 일정 수정 + 공유 일정 동기화(shared_sync) 반환
- [x] `personal_delete_saved_schedules` / `_delete_saved_schedules` — 조건 기반 삭제 (조건 없으면 거부하는 안전장치)
- [x] `personal_create_schedule` (Week1 호환) — 임시 일정 생성 결과를 SQLite에도 이중 저장
- [x] 레거시 정규화 — `unwrap_legacy_payload` / `_save_input_from` / `save_structured_request_payload`

---

## AI 활용 내역

### store 메서드 확인 후 tool 구현
- AI 활용 내용 : `fixed/app_store.py`의 `save_structured_request`, `list_schedules`, `update_schedule`, `delete_schedules_by_filter` 등 메서드 시그니처와 반환 형태를 먼저 확인했습니다.
- 직접 수정한 부분 : tool 본문은 store 메서드를 호출하고 `tool_result`/`json_payload`로 감싸는 얇은 입구로만 작성했습니다.
- 수정 이유 : SQL은 fixed가 담당하므로 tool은 검증된 인자를 넘기고 반환 형태만 맞추는 게 역할이라고 판단했습니다.

### 삭제 안전장치
- AI 활용 내용 : 조건 없이 전체가 삭제되는 사고를 막는 방법을 고민했습니다.
- 직접 수정한 부분 : `_delete_saved_schedules`에서 schedule_ids나 날짜·제목·시간 조건이 하나도 없고 delete_all도 아니면 ok=False로 거부하도록 가드를 넣었습니다.

### 코드 구조 파악
- AI 활용 내용 : week3 전체 구조(구조화 → SQLite 저장 → 조회/수정/삭제 흐름)와 AppSQLiteStore 메서드 동작을 파악하는 데 활용했습니다.

---

## 구현하면서 고민한 점

- 고민한 점 : Week1 tool은 참석자를 `attendees`, Week3 저장 스키마는 `members`로 이름이 달라서 Week1 호환 생성에서 어떻게 이어줄지 고민했습니다.
- 해결 방법 : `structured_request_from_week01_schedule`에서 `attendees → members`, `id → source_schedule_id`로 변환해 저장 입력을 맞췄습니다.

---

## 과제 회고 (KPT)

- **Keep** : store 메서드를 먼저 확인하고 tool은 얇게 유지하는 방향을 지켰다. 저장/조회 반환 키를 가이드대로 통일했다.
- **Problem** : 이번 주 화요일에서 수요일 사이에 다른 일정이 많아, 코드를 좀 더 깊게 뜯어보지 못한 점이 아쉬웠다.
- **Try** : 다음 주차에서 저장된 기록을 검색(RAG)이나 외부 조율에 어떻게 쓰는지 이어서 확인하기.


## 리뷰와 대화

### 리뷰: @nno0obb / CHANGES_REQUESTED

시각: 2026-07-16T12:31:28Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/106#pullrequestreview-4713664363

@kmgvv23 

안녕하세요, "[멘토] 김홍균" 입니다.

> 멘토마다 소속 조직과 담당 업무가 달라 피드백 방향에 다소 차이가 있을 수 있습니다.
> 빠르게 피드백 드리지 못한 점 양해 부탁드립니다 (_ _)
> 코드에 외에 궁금한 점도 있다면 언제든 편하게 질문 주세요 :D

---

### # Summary

| # | 항목 | 평가 | 근거 |
|---|---|---|---|
| 1 | 과제 구현 정확성 (메인/심화) | ✅  | 메인·심화 모두 구현 및 정상동작 확인 |
| 2 | AI 활용 | ✅ | PR 본문에 구체적으로 기록 |
| 3 | 가독성 & 주석 | ✅ | 코드가 깔끔하게 작성됨 |
| 4 | Python 컨벤션 | ✅ | 위반사항 없음 |
| 5 | Commit 컨벤션 | ⚠️ | 단일 커밋 |

### # Way to go

- `_delete_saved_schedules`에서 조건이 하나도 없고 `delete_all`도 아닌 경우 실패 응답으로 막는 guard를 tool 바깥 helper로 분리한 설계가 좋았습니다.
- `SaveStructuredRequestInput.unwrap_legacy_payload`가 `payload`/`structured_request` wrapper만 짧게 벗기고 실제 필드 검증은 스키마에 맡기는 것에서 의도가 명확히 보이는 점이 좋았습니다.

### # Go the extra mile

- `personal_create_schedule`의 `sqlite_save` 필드가 store의 raw 반환 dict라 안쪽에 `tool_name`이 없는 상태입니다. `save_structured_request_payload(save_input, store=_store())` helper 를 활용해보는 것은 어떠실까요?
- `_delete_saved_schedules`에서 지금은 `title="회의"` 하나만 지정해도 그대로 삭제가 진행되는것 같은데, 같은 제목의 여러 일정이 한꺼번에 지워지는 상황도 있을 수 있지 않을까요..? 그렇다면 어떻게 이를 방지할 수 있을까요..?
- 혹시 멘토에게 따로 묻고 싶은 게 없을까요? 과제와 직접적인 관련 없어도 AI 사용이나 현업에 관한 질문도 괜찮습니다..!

### # Just Once

1. Python 을 사용한다면 `black`, `isort` 와 같은 도구를 사용해서 일관된 컨벤션을 유지해보세요.
2. "Conventional Commit" 을 검색해서 Commit 관련 컨벤션을 공부하고 적용해보세요.
3. 단일 Commit 은 PR 리뷰를 어렵게 하는 주요 원인 중 하나입니다. Commit 을 여러개로 쪼개보아요.
4. AI 활용을 통해 학습한 내용을 정리해보면 좋을 것 같습니다. "docs" 폴더를 만들고 그 안에 내용을 채워넣어보아요.
5. Pytest 와 같은 도구를 사용해 테스트를 자동화할 수 있는지 알아보아요.


### 코드 리뷰 댓글: @nno0obb

시각: 2026-07-16T12:32:17Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/106#discussion_r3595408766
파일: `student_parts/week03_build_nanas_logbook.py` / 줄: 409
커밋: `6aa23a05f05909fdd061bbd875b3c2fa3cb19653`

````diff
@@ -321,9 +393,28 @@ def personal_create_schedule(
 ) -> str:
     """Nana의 개인 일정을 생성하고 Week 3+ 앱 SQLite DB에도 저장합니다."""
 
-    # TODO: Week 1 임시 일정 tool을 호출한 뒤 결과를 StructuredRequest로 바꿔 SQLite에도 저장하세요.
-    # TODO: created 결과에 structured_request와 sqlite_save를 합쳐 JSON 문자열로 반환하세요.
-    ...
+    created = json.loads(
+        week01_personal_create_schedule.invoke(
+            {
+                "title": title,
+                "date": date,
+                "start_time": start_time,
+                "end_time": end_time,
+                "attendees": attendees or [],
+            }
+        )
+    )
+    schedule = created.get("created_schedule", {})
+    save_input = structured_request_from_week01_schedule(schedule)
+    sqlite_save = _store().save_structured_request(save_input.model_dump())
````

여기서도 `tool_result` 를 사용해볼 수 있지 않을까요..?
`save_structured_request_payload` helper 함수 사용도 괜찮을 것 같은데
적용 전과 후에 trace 가 어떻게 달라지는지 테스트를 해보면 좋을 것 같습니다..!

### PR 대화 댓글: @kmgvv23

시각: 2026-07-19T01:06:47Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/106#issuecomment-5013620157

멘토님 안녕하세요. 바쁘신 와중에도 명확하고 실질적으로 큰 도움이 되는 피드백 정말 감사드립니다. 수정 PR이 늦어진 점 죄송합니다. 개선점 관련하여 항목별로 답변드리겠습니다.

**1. `personal_create_schedule`의 `sqlite_save`에 `tool_name`이 없던 부분**
이 함수만 저장 결과를 그대로 넣다 보니 다른 tool들과 달리 `ok`/`tool_name`이 빠져 있었습니다. `save_structured_request_payload(save_input, store=_store())`로 감싸도록 수정해 반환 형식을 다른 tool들과 일관되게 맞췄습니다. 짚어주셔서 감사합니다!

**2. `_delete_saved_schedules`에서 같은 제목 여러 개가 한꺼번에 삭제되는 문제**
중요한 지적 감사합니다. `title="회의"` 하나로 동명의 일정이 전부 삭제되면 의도치 않은 대량 삭제가 될 수 있어, `schedule_ids`도 `delete_all`도 없이 필터로만 삭제하는 경우 먼저 `store.find_schedules(...)`로 매칭 건수를 확인하고 **2건 이상이면 삭제를 보류한 뒤 후보(candidates)를 반환**하도록 guard를 추가했습니다.
(완전히 관련 있는 건 아니지만, 몇 시간 전 AWS 관련 프로젝트를 하다가 업데이트된 소스코드를 올려달라고 AI에 시켰는데, 다른 팀원이 만든 AWS Bedrock 모델을 실수로 삭제한 일이 있었습니다. 예전보다 많이 발전했다고는 해도 여전히 이런 치명적인 실수가 생길 수 있다는 걸 실감하는 계기였고, 그래서 이번 삭제 guard의 필요성에 더 공감하게 되었습니다.)

**3. Commit 컨벤션**
제가 아직 깃헙 협업 경험이 많지 않아 부족했는데, 이번 기회에 Conventional Commit 개념도 숙지하는 등 많은 것을 배웠습니다. 감사합니다. 이번 수정분부터는 `fix:`(sqlite_save) / `feat:`(삭제 guard)처럼 기능 단위로 커밋을 나눠 올렸습니다.
한 가지 궁금한 점이 있는데, 과제처럼 파일 하나를 처음부터 구현하는 경우에는 커밋을 어떤 기준으로 나눠 올리는 게 좋을까요? (스키마/로직/기능 단위 등 여러 방법이 떠올라서, 실무에서 선호하시는 기준이 궁금합니다.)

**4. black / isort**
이 도구들도 이번에 처음 알았는데 정말 유용해 보입니다. 다만 지금 파일에 뒤늦게 적용하면 기존 코드가 전부 재포맷되어 diff가 지저분해질 것 같아, 다음 주차부터 처음부터 적용하겠습니다.

**5. docs 폴더 / AI 학습 정리**
좋은 제안 감사합니다. `docs/` 폴더를 만들어 이번 주차에 배운 내용(structured output → SQLite 저장 흐름, AppSQLiteStore 구조 등)을 정리하겠습니다.

**6. Pytest**
아직 저장→조회→삭제를 직접 함수 호출로만 확인하고 있었는데, pytest로 이 흐름을 assert 기반 테스트로 자동화할 수 있을 것 같아 다음 주차에 적용해보겠습니다.

마지막으로 공부 차원에서 하나 여쭤보고 싶습니다. 요즘 혼자 공부하다 보면, 어떤 부분은 깊이 파고들어 직접 이해하고 어떤 부분은 AI를 어느 정도 신뢰하고 넘어가야 할지 그 깊이의 기준을 잡기가 어렵습니다. 멘토님은 학습하실 때 "여기까지는 직접 이해하고, 이 이상은 도구나 AI를 믿고 넘어간다"는 나름의 기준이 있으신지 궁금합니다.

다시 한번 꼼꼼한 리뷰 감사드립니다. 수정 반영 후 재리뷰 요청 드리겠습니다! 🙇‍♂️

### 리뷰: @nno0obb / APPROVED

시각: 2026-07-19T14:02:17Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/106#pullrequestreview-4713670145

@kmgvv23 

> 멘토님 안녕하세요. 바쁘신 와중에도 명확하고 실질적으로 큰 도움이 되는 피드백 정말 감사드립니다. 수정 PR이 늦어진 점 죄송합니다. 개선점 관련하여 항목별로 답변드리겠습니다.

감사합니다, 굉장히 뿌듯하네요 ㅎㅎ 앞으로도 열심히 하겠습니다..!

> **1. `personal_create_schedule`의 `sqlite_save`에 `tool_name`이 없던 부분** 이 함수만 저장 결과를 그대로 넣다 보니 다른 tool들과 달리 `ok`/`tool_name`이 빠져 있었습니다. `save_structured_request_payload(save_input, store=_store())`로 감싸도록 수정해 반환 형식을 다른 tool들과 일관되게 맞췄습니다. 짚어주셔서 감사합니다!

👍 

> **2. `_delete_saved_schedules`에서 같은 제목 여러 개가 한꺼번에 삭제되는 문제** 중요한 지적 감사합니다. `title="회의"` 하나로 동명의 일정이 전부 삭제되면 의도치 않은 대량 삭제가 될 수 있어, `schedule_ids`도 `delete_all`도 없이 필터로만 삭제하는 경우 먼저 `store.find_schedules(...)`로 매칭 건수를 확인하고 **2건 이상이면 삭제를 보류한 뒤 후보(candidates)를 반환**하도록 guard를 추가했습니다. (완전히 관련 있는 건 아니지만, 몇 시간 전 AWS 관련 프로젝트를 하다가 업데이트된 소스코드를 올려달라고 AI에 시켰는데, 다른 팀원이 만든 AWS Bedrock 모델을 실수로 삭제한 일이 있었습니다. 예전보다 많이 발전했다고는 해도 여전히 이런 치명적인 실수가 생길 수 있다는 걸 실감하는 계기였고, 그래서 이번 삭제 guard의 필요성에 더 공감하게 되었습니다.)

그런 일이 있었군요 ㄷㄷ 그러한 경험들이 성장을 촉진한다고 저는 생각합니다.
많은 사건과 사고들을 보기도 하고 일으키면서(?) 어떻게 예방할 수 있는지 치열하게 고민할수록
빠르게 실력이 늘어나는 걸 체감할 수 있으실 것입니다 ㅎㅎ

> **3. Commit 컨벤션** 제가 아직 깃헙 협업 경험이 많지 않아 부족했는데, 이번 기회에 Conventional Commit 개념도 숙지하는 등 많은 것을 배웠습니다. 감사합니다. 이번 수정분부터는 `fix:`(sqlite_save) / `feat:`(삭제 guard)처럼 기능 단위로 커밋을 나눠 올렸습니다. 한 가지 궁금한 점이 있는데, 과제처럼 파일 하나를 처음부터 구현하는 경우에는 커밋을 어떤 기준으로 나눠 올리는 게 좋을까요? (스키마/로직/기능 단위 등 여러 방법이 떠올라서, 실무에서 선호하시는 기준이 궁금합니다.)

정해진 기준은 없습니다...만 제 선호는 최대한 잘게 입니다 ㅋㅋ
기능단위보다는 2~3개의 함수단위, 그리고 주석이 달린 커밋을 선호하는 편입니다.
그런데 이는 "느낌" 인 부분이라서 과연 이 커밋을 보고 동료들이, 그리고 AI 가 이해하기 좋을까? 를 고민하고 시도해본다면
본인만의 기준이 생기실 것이라고 생각합니다 :D
그 기준이 앞으로도 계속 업데이트되가며 좋은 커밋으로 이어지기를 바랍니다!

> **4. black / isort** 이 도구들도 이번에 처음 알았는데 정말 유용해 보입니다. 다만 지금 파일에 뒤늦게 적용하면 기존 코드가 전부 재포맷되어 diff가 지저분해질 것 같아, 다음 주차부터 처음부터 적용하겠습니다.

👍 

> **5. docs 폴더 / AI 학습 정리** 좋은 제안 감사합니다. `docs/` 폴더를 만들어 이번 주차에 배운 내용(structured output → SQLite 저장 흐름, AppSQLiteStore 구조 등)을 정리하겠습니다.

👍 

> **6. Pytest** 아직 저장→조회→삭제를 직접 함수 호출로만 확인하고 있었는데, pytest로 이 흐름을 assert 기반 테스트로 자동화할 수 있을 것 같아 다음 주차에 적용해보겠습니다.

👍 

> 마지막으로 공부 차원에서 하나 여쭤보고 싶습니다. 요즘 혼자 공부하다 보면, 어떤 부분은 깊이 파고들어 직접 이해하고 어떤 부분은 AI를 어느 정도 신뢰하고 넘어가야 할지 그 깊이의 기준을 잡기가 어렵습니다. 멘토님은 학습하실 때 "여기까지는 직접 이해하고, 이 이상은 도구나 AI를 믿고 넘어간다"는 나름의 기준이 있으신지 궁금합니다.

이것도 "느낌" 인 부분이라서 어렵네요...
모든 걸 이해할 수는 없고, 그렇다고 모든 걸 알고 넘어갈 수도 없습니다.
그 중간에 선을 그을 수 있는 것이 실력이라고 생각되네요.
그래도 지금 제가 현업을 하면서 제가 가지고 있는 기준은 "당장 의사결정에 필요한가?" 입니다.
이 프로젝트가, 이 개발사항이, 이 지식이 지금 제가 하고 있는 일을 더 잘할 수 있게하는 데 기여하는가?
그렇다면 AI 를 통해 지속적으로 질문하고 그렇지 않다면 넘어가고 있는 것 같습니다 ㅎㅎ

> 다시 한번 꼼꼼한 리뷰 감사드립니다. 수정 반영 후 재리뷰 요청 드리겠습니다! 🙇‍♂️

👍 

---

이번 주차도 고생 많으셨습니다 (_ _)
다음주차도 화이팅입니다!

### PR 대화 댓글: @kmgvv23

시각: 2026-07-19T15:00:45Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/106#issuecomment-5016193184

2차 피드백도 정성스럽게 말씀해주셔서 감사합니다!!
어디까지 꼼꼼히 공부할것인가에 대한 고민은 계속해봐야겠네요...ㅎㅎ
4주차는 더 자세하게 공부해보고 과제 해보도록 하겠습니다. 감사합니다!

## 변경 파일

- `docs/week03.md` (+31 / -0)
- `student_parts/week03_build_nanas_logbook.py` (+203 / -38)
