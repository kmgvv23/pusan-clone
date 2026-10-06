# #174 week5: 외부 대화·일정 MCP wrapper 및 멤버 일정 통합 구현

원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174

작성자: @kmgvv23
작성 시각: 2026-07-29T13:19:32Z
보관 시각 (UTC): 2026-10-06T15:19:25.015347+00:00

## PR 본문

## 과제 목표

- 외부 SQLite/MCP 서버에 있는 다른 사람의 대화·일정을 agent가 쓸 수 있게 감싸보기
- 내 일정과 외부 멤버 일정을 같은 구조로 모아서 일정 조율의 기초 데이터를 만들어보기

---

## 과제 위치
- 작업 브랜치 : `kimmingyu/week5`  → 본인 통합 브랜치 `kimmingyu/final` 로 PR
- 주요 파일 : `student_parts/week05_load_kanas_past_conversations.py`
- 함께 반영 : `student_parts/week04_retrieve_nanas_memory.py` (Week 4 심화)

> 제공 코드(`fixed/` 등)는 수정하지 않고, `student_parts/` 의 본인 주차 파일을 구현했습니다.

---

## 과제 범위

- [x] 메인 과제 완료
- [x] 심화 과제까지 완료

---

## 구현한 기능 (메인)

- [x] `search_previous_conversations` — 외부 과거 대화 검색 MCP wrapper
- [x] `load_conversation_messages` — 특정 대화 전체 메시지 로드 (순서 보존)
- [x] `extract_schedules_from_history` — 외부 멤버 대화에서 일정 추출
- [x] `list_shared_schedules` — 공유 일정 저장소 row 조회
- [x] `collect_member_schedules` — 내 일정과 외부 멤버 busy-time을 같은 rows로 통합
- [x] Week 5 라우팅 system prompt

---

## 심화 과제

- [x] `create_shared_schedule` / `delete_shared_schedule` — 공유 일정 저장소 row 등록·삭제 wrapper
- [x] Week 4 심화(`search_conversation_messages`, `search_nana_memory`) — 5주차에 필요할 것 같아 채웠는데, 이미 지난 주차라 리뷰하시기 애매할 것 같아 배포된 baseline 정답 코드를 반영했습니다.

---

## AI 활용 내역

- week5 구조(MCP wrapper의 책임 경계, 어디까지가 학생 구현 대상인지)를 파악하는 데 AI를 활용했습니다.
- `fixed/external_people_store.py`와 `fixed/mcp_client.py`의 helper가 무엇을 정규화해 주는지 확인해, wrapper에서 중복 변환하지 않도록 정리했습니다.
- `collect_member_schedules`에서 내 일정을 언제 포함하는 것이 구조상 맞는지 판단이 서지 않아, 가이드와 helper 시그니처를 근거로 기준을 확인했습니다.

---

## 구현하면서 고민한 점

- **고민** : `collect_member_schedules`에서 내 일정을 어떤 기준으로 포함할지가 애매했습니다. `member_names`에 "나"가 있을 때만 넣을지, 항상 넣을지, 아니면 인자를 따로 둘지 여러 방법이 떠올랐습니다.
- **결정** : helper가 이미 `personal_schedules`를 인자로 받고 있고, 가이드의 검증 방법도 "rows에 '나'와 외부 멤버 일정이 같은 구조로 들어 있는지 확인"이라 **항상 포함**하는 쪽으로 정했습니다. 대신 **MCP 조회 대상에서는 "나"를 제외**했습니다. "나"는 외부 저장소에 없는 사람이라 그대로 넘기면 못 찾거나 내 일정이 중복으로 들어올 수 있어서, "나는 앱 DB에서 / 외부 멤버는 MCP에서"로 출처를 분리했습니다.
- **tool 선택 문제** : Week 4의 `search_conversation_messages`(내 앱 대화)와 Week 5의 `search_previous_conversations`(외부 멤버 대화)는 서로 다른 저장소를 보는데, "예전에 얘기했던 거"처럼 애매한 질문에서는 LLM이 둘 중 무엇을 골라야 할지 판단할 근거가 없었습니다. 주차가 쌓이며 tool이 많아진 상태라, system prompt에 "내 정보는 이전 주차 도구, 외부 멤버는 Week 5 wrapper"라는 문구를 추가했습니다.

---

## 과제 회고 (KPT)

- **Keep** : 구현 전에 외부 실습 DB에 실제로 어떤 멤버가 있는지 확인하고 그 데이터로 검증한 것. 판단이 애매한 부분은 넘기지 않고 가이드에서 근거를 찾아 정한 것.
- **Problem** : 코드를 좀 더 많이 뜯어보고 싶었는데 시간 관계상 그러지 못한 점이 아쉬웠다.
- **Try** : 다음 주차(Week 6)에서 이번에 모은 rows가 공통 가능 시간(`find_common_available_slots`)을 찾는 데 어떻게 쓰이는지 이어서 확인하기. 그리고 앞으로도 심화 과제까지 계속 해내기.


## 리뷰와 대화

### 코드 리뷰 댓글: @doheelab-coder

시각: 2026-07-30T15:12:20Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#discussion_r3683890503
파일: `student_parts/week05_load_kanas_past_conversations.py` / 줄: 510
커밋: `d3bddd5bbbb1cc347316c8068f500205cc70498c`

````diff
@@ -390,7 +497,19 @@ def week05_prompt_parts() -> list[str]:
 
     return [
         *week04_prompt_parts(),
-        # TODO: Week 5 Kana history agent system prompt를 자유롭게 추가하세요.
+        "내 개인 정보(메모·저장한 일정·내 앱 대화)는 이전 주차 도구로 처리하고, "
+        "다른 사람(외부 멤버)의 과거 대화와 일정은 Week 5 MCP wrapper로 처리한다. "
+        "내 앱 대화 검색은 search_conversation_messages, 외부 멤버의 과거 대화 검색은 "
+        "search_previous_conversations로 서로 다른 저장소를 본다는 점을 혼동하지 않는다.",
+        "다른 사람의 일정을 물으면 extract_schedules_from_history로 그 멤버의 일정을 추출하고, "
+        "특정 대화의 전체 내용이 필요하면 search_previous_conversations로 conversation_id를 찾은 뒤 "
+        "load_conversation_messages로 불러온다. 공유 일정 저장소 자체를 확인할 때는 list_shared_schedules를 쓴다.",
+        "나와 다른 사람의 일정을 함께 봐야 하는 조율 질문에는 collect_member_schedules를 사용한다. "
+        "member_names에는 조율 대상 전원을 넣고, 내 일정이 필요하면 '나'를 함께 넣는다. "
+        "이 tool은 내 일정과 외부 멤버 일정을 같은 구조로 모아 주므로 출처별로 따로 호출하지 않는다.",
````

프롬프트는 "내 일정이 필요하면 `'나'`를 함께 넣는다"고 안내하는데, 구현은 "나" 여부와 무관하게 내 일정을 항상 포함합니다. PR 본문에서 "항상 포함"을 의도적으로 고르셨으니, 프롬프트를 구현 쪽에 맞추면 됩니다.

```python
        "나와 다른 사람의 일정을 함께 봐야 하는 조율 질문에는 collect_member_schedules를 사용한다. "
        "이 tool은 내 일정을 항상 포함하므로 member_names에는 외부 조율 대상만 넣으면 된다. "
        "이 tool은 내 일정과 외부 멤버 일정을 같은 구조로 모아 주므로 출처별로 따로 호출하지 않는다.",
```

지금 동작에는 문제가 없지만, 안내와 실제가 다르면 나중에 이 tool을 다시 볼 때 헷갈립니다.

### 코드 리뷰 댓글: @doheelab-coder

시각: 2026-07-30T15:12:20Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#discussion_r3683890510
파일: `student_parts/week05_load_kanas_past_conversations.py` / 줄: 327
커밋: `4561142c2c753f194478e383d85e3120bffa0fd4`

````diff
@@ -282,8 +297,59 @@ def _collect_member_schedules(
 ) -> dict[str, Any]:
     """내 일정과 외부 멤버 일정을 같은 row 구조로 합칩니다."""
 
-    # TODO: 내 SQLite/임시 일정과 외부 MCP 일정 rows를 같은 구조로 합치세요.
-    ...
+    normalized_members = normalize_external_member_names(member_names)
+    normalized_date_from, normalized_date_to = normalize_external_schedule_date_bounds(
+        member_names, date_from, date_to
+    )
+
+    # 내 일정은 앱 SQLite/임시 메모리가 권위 있는 출처라 여기서 직접 row로 만든다.
+    rows: list[dict[str, Any]] = []
+    for schedule in personal_schedules:
+        request = _structured_request_from_schedule_row(schedule)
+        if not request.date:
+            continue
+        if normalized_date_from and request.date < normalized_date_from:
+            continue
+        if normalized_date_to and request.date > normalized_date_to:
+            continue
+        rows.append(
+            {
+                "member_name": "나",
+                "title": request.title or "제목 없음",
+                "date": request.date,
+                "start_time": request.start_time or "미정",
+                "end_time": request.end_time or "미정",
+                "notes": schedule.get("notes") or "저장된 내 일정",
+            }
+        )
+
+    # "나"는 외부 저장소에 없는 사람이라 MCP 조회 대상에서 제외한다.
+    external_members = [name for name in normalized_members if name != "나"]
````

이 주석의 이유가 실제 구조와 반대입니다. 외부 저장소에는 "나" 데이터가 있습니다 — 앱이 일정을 저장할 때마다 "나" 이름으로 복사본을 만들어 둡니다(`fixed/external_mcp.py`의 `sync_personal_schedule_to_shared`). 위에서 앱 DB의 내 일정을 이미 넣었는데 외부 복사본까지 받으면 같은 일정이 두 번 들어오기 때문에 빼는 것이 맞습니다. 동작은 정확하니 주석만 고치면 됩니다.

```python
    # 내 일정은 앱 DB에서 이미 읽었고 외부 저장소에도 "나" 복사본이 있으므로,
    # 중복 수집을 막기 위해 MCP 조회 대상에서는 제외한다.
```

### 코드 리뷰 댓글: @doheelab-coder

시각: 2026-07-30T15:12:20Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#discussion_r3683890516
파일: `tests/test_week05.py` / 줄: 6
커밋: `d3bddd5bbbb1cc347316c8068f500205cc70498c`

````diff
@@ -0,0 +1,85 @@
+"""Week 5 외부 멤버 일정 수집 helper 테스트.
+
+MCP subprocess를 띄우지 않는 경로만 검증한다.
+외부 조회가 필요 없는 조건(외부 멤버가 없는 경우)과 내 일정 필터링/중복 제거를 확인한다.
+실행: uv run pytest tests/test_week05.py
+"""
````

설명은 "중복 제거를 확인한다"인데, 현재 테스트 중 어느 것도 `_personal_schedules_for_current_scope()`를 호출하지 않아 중복 제거 경로는 타지 않습니다. 아래처럼 하나 추가하면 설명과 맞습니다.

```python
import student_parts.week05_load_kanas_past_conversations as week05


def test_personal_schedules_dedup_saved_and_pending(monkeypatch):
    saved = {"schedule_id": "sch_1", "title": "회의", "date": "2026-07-10"}
    pending = {"id": "sch_1", "title": "회의", "date": "2026-07-10", "session_id": "s1"}
    other = {"id": "sch_2", "title": "헬스장", "date": "2026-07-11", "session_id": "s1"}

    class FakeStore:
        def __init__(self, _path):
            pass

        def list_schedules(self, limit=12):
            return [saved]

    monkeypatch.setattr(week05, "AppSQLiteStore", FakeStore)
    monkeypatch.setattr(week05, "PERSONAL_SCHEDULES", [pending, other])
    monkeypatch.setattr(week05, "current_session_scope", lambda: "s1")

    assert week05._personal_schedules_for_current_scope() == [saved, other]
```

실행 안내의 `uv run pytest`는 `pytest`가 의존성에 없어 환경에 따라 실패합니다. `uv run --with pytest python -m pytest -q tests/test_week05.py`로 적으면 그대로 돌아갑니다.

### 리뷰: @doheelab-coder / CHANGES_REQUESTED

시각: 2026-07-30T15:15:44Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#pullrequestreview-4820306164

week5 1차 리뷰입니다. 메인 5개와 심화 2개 모두 구현하셨습니다.

내 일정은 앱 DB에서, 외부 멤버 일정은 MCP에서 가져오도록 출처를 나누고 외부 조회에서 "나"를 걸러낸 처리가 좋았습니다. 앱이 일정을 저장할 때 외부 저장소에도 "나" 복사본을 만들어 두기 때문에, 걸러내지 않았다면 같은 일정이 두 번 나왔을 겁니다.

반드시 고칠 것보다는 제안 위주로 세 개를 인라인에 남겼습니다 — 프롬프트와 구현이 어긋난 지점, 주석 교정, 테스트 보강입니다. 반영해 주시면 확인 후 승인드리겠습니다.

### PR 대화 댓글: @kmgvv23

시각: 2026-08-02T07:22:41Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#issuecomment-5156180639

멘토님 안녕하세요! 수정이 늦어져서 죄송합니다... 완전 잊고 있었습니다...
먼저, 바쁘실텐데 꼼꼼하게 리뷰해주셔서 정말 감사합니다!
그리고 코드, PR 본문, 주석을 과제하는 도중 지속 수정하다가 이렇게 안맞는 부분이 많이 발생한 것 같습니다. 확인하고 냈어야하는데 기본적인 실수들이 많은 것 같아 죄송하네요...
주신 피드백 기반으로 수정해봤습니다! 마지막주에는 빠릿빠릿하고 꼼꼼히 확인 후 제출하겠습니다

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-08-02T07:40:31Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#discussion_r3698278140
파일: `student_parts/week05_load_kanas_past_conversations.py` / 줄: 510
커밋: `d3bddd5bbbb1cc347316c8068f500205cc70498c`

````diff
@@ -390,7 +497,19 @@ def week05_prompt_parts() -> list[str]:
 
     return [
         *week04_prompt_parts(),
-        # TODO: Week 5 Kana history agent system prompt를 자유롭게 추가하세요.
+        "내 개인 정보(메모·저장한 일정·내 앱 대화)는 이전 주차 도구로 처리하고, "
+        "다른 사람(외부 멤버)의 과거 대화와 일정은 Week 5 MCP wrapper로 처리한다. "
+        "내 앱 대화 검색은 search_conversation_messages, 외부 멤버의 과거 대화 검색은 "
+        "search_previous_conversations로 서로 다른 저장소를 본다는 점을 혼동하지 않는다.",
+        "다른 사람의 일정을 물으면 extract_schedules_from_history로 그 멤버의 일정을 추출하고, "
+        "특정 대화의 전체 내용이 필요하면 search_previous_conversations로 conversation_id를 찾은 뒤 "
+        "load_conversation_messages로 불러온다. 공유 일정 저장소 자체를 확인할 때는 list_shared_schedules를 쓴다.",
+        "나와 다른 사람의 일정을 함께 봐야 하는 조율 질문에는 collect_member_schedules를 사용한다. "
+        "member_names에는 조율 대상 전원을 넣고, 내 일정이 필요하면 '나'를 함께 넣는다. "
+        "이 tool은 내 일정과 외부 멤버 일정을 같은 구조로 모아 주므로 출처별로 따로 호출하지 않는다.",
````

항상 포함으로 정해놓고 프롬프트는 이전에 고민하던 조건부 방식 문구를 그대로 두었네요. 제안해주신 문구로 교체해 안내와 구현을 맞췄습니다.


### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-08-02T07:40:31Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#pullrequestreview-4837660802

(본문 없음)

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-08-02T07:42:26Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#discussion_r3698283379
파일: `student_parts/week05_load_kanas_past_conversations.py` / 줄: 327
커밋: `4561142c2c753f194478e383d85e3120bffa0fd4`

````diff
@@ -282,8 +297,59 @@ def _collect_member_schedules(
 ) -> dict[str, Any]:
     """내 일정과 외부 멤버 일정을 같은 row 구조로 합칩니다."""
 
-    # TODO: 내 SQLite/임시 일정과 외부 MCP 일정 rows를 같은 구조로 합치세요.
-    ...
+    normalized_members = normalize_external_member_names(member_names)
+    normalized_date_from, normalized_date_to = normalize_external_schedule_date_bounds(
+        member_names, date_from, date_to
+    )
+
+    # 내 일정은 앱 SQLite/임시 메모리가 권위 있는 출처라 여기서 직접 row로 만든다.
+    rows: list[dict[str, Any]] = []
+    for schedule in personal_schedules:
+        request = _structured_request_from_schedule_row(schedule)
+        if not request.date:
+            continue
+        if normalized_date_from and request.date < normalized_date_from:
+            continue
+        if normalized_date_to and request.date > normalized_date_to:
+            continue
+        rows.append(
+            {
+                "member_name": "나",
+                "title": request.title or "제목 없음",
+                "date": request.date,
+                "start_time": request.start_time or "미정",
+                "end_time": request.end_time or "미정",
+                "notes": schedule.get("notes") or "저장된 내 일정",
+            }
+        )
+
+    # "나"는 외부 저장소에 없는 사람이라 MCP 조회 대상에서 제외한다.
+    external_members = [name for name in normalized_members if name != "나"]
````

이것 또한 코드 이해가 부족했고, 계속 파악하던 중 작성한 주석을 그대로 써뒀던 것 같습니다. 결과값은 같겠지만, 주석은 제시해주신것으로 수정했습니다.

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-08-02T07:42:26Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#pullrequestreview-4837666359

(본문 없음)

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-08-02T07:54:07Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#discussion_r3698313913
파일: `tests/test_week05.py` / 줄: 6
커밋: `d3bddd5bbbb1cc347316c8068f500205cc70498c`

````diff
@@ -0,0 +1,85 @@
+"""Week 5 외부 멤버 일정 수집 helper 테스트.
+
+MCP subprocess를 띄우지 않는 경로만 검증한다.
+외부 조회가 필요 없는 조건(외부 멤버가 없는 경우)과 내 일정 필터링/중복 제거를 확인한다.
+실행: uv run pytest tests/test_week05.py
+"""
````

pytest는 이전 멘토님이 제안해주신 방법인데 제대로 파악하지 못한 채 LLM을 활용해 작성하다 보니, 설명만 써두고 정작 중복 제거 경로는 검증하지 못했던 것 같습니다.

주신 코드를 보고 실제 DB에 의존하지 않도록 저장소를 바꿔 테스트하는 방식을 알게 됐습니다. 테스트가 실행 환경의 실제 데이터에 따라 결과가 달라지면 안 된다는 점도 이번에 이해했습니다. 그대로 추가해 설명과 맞췄고, 실행 안내도 uv run --with pytest python -m pytest -q ...로 수정했습니다.

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-08-02T07:54:07Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#pullrequestreview-4837698996

(본문 없음)

### 리뷰: @doheelab-coder / APPROVED

시각: 2026-08-03T14:15:41Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#pullrequestreview-4844837626

승인합니다. 프롬프트·주석·테스트 세 가지 모두 반영을 확인했고, 각 지점에 확인 코멘트를 달았습니다.

### 코드 리뷰 댓글: @doheelab-coder

시각: 2026-08-03T14:15:42Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#discussion_r3704710556
파일: `student_parts/week05_load_kanas_past_conversations.py` / 줄: 510
커밋: `d3bddd5bbbb1cc347316c8068f500205cc70498c`

````diff
@@ -390,7 +497,19 @@ def week05_prompt_parts() -> list[str]:
 
     return [
         *week04_prompt_parts(),
-        # TODO: Week 5 Kana history agent system prompt를 자유롭게 추가하세요.
+        "내 개인 정보(메모·저장한 일정·내 앱 대화)는 이전 주차 도구로 처리하고, "
+        "다른 사람(외부 멤버)의 과거 대화와 일정은 Week 5 MCP wrapper로 처리한다. "
+        "내 앱 대화 검색은 search_conversation_messages, 외부 멤버의 과거 대화 검색은 "
+        "search_previous_conversations로 서로 다른 저장소를 본다는 점을 혼동하지 않는다.",
+        "다른 사람의 일정을 물으면 extract_schedules_from_history로 그 멤버의 일정을 추출하고, "
+        "특정 대화의 전체 내용이 필요하면 search_previous_conversations로 conversation_id를 찾은 뒤 "
+        "load_conversation_messages로 불러온다. 공유 일정 저장소 자체를 확인할 때는 list_shared_schedules를 쓴다.",
+        "나와 다른 사람의 일정을 함께 봐야 하는 조율 질문에는 collect_member_schedules를 사용한다. "
+        "member_names에는 조율 대상 전원을 넣고, 내 일정이 필요하면 '나'를 함께 넣는다. "
+        "이 tool은 내 일정과 외부 멤버 일정을 같은 구조로 모아 주므로 출처별로 따로 호출하지 않는다.",
````

교체 확인했습니다. `member_names`에 외부 조율 대상만 넣으라고 안내가 분명해져서, 프롬프트와 구현이 같은 말을 하게 됐습니다.

### 리뷰: @doheelab-coder / COMMENTED

시각: 2026-08-03T14:15:42Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#pullrequestreview-4845031897

(본문 없음)

### 코드 리뷰 댓글: @doheelab-coder

시각: 2026-08-03T14:15:44Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#discussion_r3704710742
파일: `student_parts/week05_load_kanas_past_conversations.py` / 줄: 327
커밋: `4561142c2c753f194478e383d85e3120bffa0fd4`

````diff
@@ -282,8 +297,59 @@ def _collect_member_schedules(
 ) -> dict[str, Any]:
     """내 일정과 외부 멤버 일정을 같은 row 구조로 합칩니다."""
 
-    # TODO: 내 SQLite/임시 일정과 외부 MCP 일정 rows를 같은 구조로 합치세요.
-    ...
+    normalized_members = normalize_external_member_names(member_names)
+    normalized_date_from, normalized_date_to = normalize_external_schedule_date_bounds(
+        member_names, date_from, date_to
+    )
+
+    # 내 일정은 앱 SQLite/임시 메모리가 권위 있는 출처라 여기서 직접 row로 만든다.
+    rows: list[dict[str, Any]] = []
+    for schedule in personal_schedules:
+        request = _structured_request_from_schedule_row(schedule)
+        if not request.date:
+            continue
+        if normalized_date_from and request.date < normalized_date_from:
+            continue
+        if normalized_date_to and request.date > normalized_date_to:
+            continue
+        rows.append(
+            {
+                "member_name": "나",
+                "title": request.title or "제목 없음",
+                "date": request.date,
+                "start_time": request.start_time or "미정",
+                "end_time": request.end_time or "미정",
+                "notes": schedule.get("notes") or "저장된 내 일정",
+            }
+        )
+
+    # "나"는 외부 저장소에 없는 사람이라 MCP 조회 대상에서 제외한다.
+    external_members = [name for name in normalized_members if name != "나"]
````

주석 수정 확인했습니다. 이유가 "거기 없어서"가 아니라 "이미 한 벌 있어서"라는 게 핵심인데 그대로 반영해 주셨습니다.

한 가지만 덧붙이면, 문자열 `"나"`는 `fixed/external_people_store.py`에 `PERSONAL_SHARED_MEMBER_NAME` 상수로도 있습니다. 지금 동작에는 문제없지만 다음에 손볼 일이 있으면 상수를 import해 쓰는 편이 이름이 바뀔 때 안전합니다.

### 리뷰: @doheelab-coder / COMMENTED

시각: 2026-08-03T14:15:44Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#pullrequestreview-4845032171

(본문 없음)

### 코드 리뷰 댓글: @doheelab-coder

시각: 2026-08-03T14:15:45Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#discussion_r3704710978
파일: `tests/test_week05.py` / 줄: 6
커밋: `d3bddd5bbbb1cc347316c8068f500205cc70498c`

````diff
@@ -0,0 +1,85 @@
+"""Week 5 외부 멤버 일정 수집 helper 테스트.
+
+MCP subprocess를 띄우지 않는 경로만 검증한다.
+외부 조회가 필요 없는 조건(외부 멤버가 없는 경우)과 내 일정 필터링/중복 제거를 확인한다.
+실행: uv run pytest tests/test_week05.py
+"""
````

테스트 추가 확인했습니다.

### 리뷰: @doheelab-coder / COMMENTED

시각: 2026-08-03T14:15:45Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/174#pullrequestreview-4845032451

(본문 없음)

## 변경 파일

- `docs/week05.md` (+50 / -0)
- `student_parts/week04_retrieve_nanas_memory.py` (+128 / -8)
- `student_parts/week05_load_kanas_past_conversations.py` (+142 / -22)
- `tests/test_week05.py` (+106 / -0)
