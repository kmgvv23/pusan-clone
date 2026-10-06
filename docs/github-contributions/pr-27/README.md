# #27 week1: 개인 일정 CRUD tool 구현

원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/27

작성자: @kmgvv23
작성 시각: 2026-06-30T08:40:52Z
보관 시각 (UTC): 2026-10-06T15:19:25.015347+00:00

## PR 본문

## 과제 목표

- LangChain `@tool` 데코레이터로 LLM이 직접 고를 수 있는 도구를 만드는 방법 이해하기
- 자연어 일정 요청을 구조화된 데이터(JSON)로 변환하고 임시 메모리에 생성/조회/삭제하는 흐름 파악하기
- `session_id` 기준으로 대화 범위를 분리해 다른 세션의 일정을 건드리지 않는 방식 익히기

---

## 과제 위치
- 작업 브랜치 : `kimmingyu/week1`  → 본인 통합 브랜치 `kimmingyu/final` 로 PR
- 주요 파일 : `student_parts/week01_wake_up_nana.py`  (주차별 `student_parts/weekNN_*.py`)

> 제공 코드(`fixed/` 등)는 수정하지 않고, `student_parts/` 의 **본인 주차 파일**을 구현합니다.
> 리뷰어(담당 멘토)는 PR 을 열면 **자동 지정**되니 직접 추가하지 않아도 됩니다.

---

## 구현한 기능

- [x] `personal_create_schedule` — 현재 대화 범위의 개인 일정 생성 (`session_id` 포함, `created_schedule` 반환)
- [x] `personal_list_schedules` — 현재 대화 범위 조회 + `date_from`/`date_to` 날짜 필터
- [x] `personal_delete_schedule` — 현재 대화 범위에서 `schedule_id` 일치 일정만 삭제 (`deleted` 반환)
- [x] 시스템 프롬프트 작성

---

## 도전 기능

- [ ] 없음

---

## AI 활용 내역

### 개인 일정 CRUD 도구 구현
- AI 활용 내용 : 가이드 주석 기반으로 tool 뼈대 구조를 참고했습니다.
- 직접 수정한 부분 : 삭제 시 `PERSONAL_SCHEDULES[:]`에 새 목록을 대입해 리스트 객체를 유지하도록 했고, session_id 조건을 AND로 함께 체크했습니다.
- 수정 이유 : 다른 세션의 같은 ID가 지워지지 않도록 하기 위해서입니다.

---

## 구현하면서 고민한 점

- 고민한 점 : `personal_delete_schedule`에서 다른 세션의 같은 ID를 실수로 지우지 않도록 session_id 조건을 함께 걸어야 한다는 점이 고민됐습니다.
- 해결 방법 : 생성 시 `session_id`를 함께 저장하고, 삭제 시 `_schedule_scope(s) == session_id` 조건을 AND로 함께 체크해서 현재 대화 범위의 일정만 삭제되도록 처리했습니다.

---

## 과제 회고 (KPT)

- **Keep** : 가이드 주석이 상세해서 요구사항을 하나씩 따라가며 구현할 수 있었습니다. 제공된 헬퍼를 재사용하고 반환 JSON 키 규칙을 지킨 점도 좋았습니다.
- **Problem** : 여행 일정이 겹쳐서 충분히 코드를 뜯어보지 못하고 급하게 제출하게 됐습니다. 전반적으로 아쉬움이 남습니다.
- **Try** : 다음 주차엔 시간 여유를 두고 코드 구조를 제대로 파악한 뒤 시작하겠습니다.

## 리뷰와 대화

### 코드 리뷰 댓글: @Gongmeda

시각: 2026-06-30T17:38:59Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/27#discussion_r3500736345
파일: `student_parts/week01_wake_up_nana.py` / 줄: 171
커밋: `877ba46be3b1da1e55eaeed63537f82f7ae1222d`

````diff
@@ -170,24 +170,50 @@ def personal_create_schedule(
 ) -> str:
     """Nana의 개인 일정을 현재 대화의 임시 메모리에 생성합니다."""
````

이 구조에서는 툴의 docstring이 실제 agent가 보는 tool description으로 쓰이는데요.
현재 함수명과 인자명만으로도 기본 흐름은 읽히지만, `date` 형식, `date_from/date_to` 의미 등을 agent가 명확히 알기 어렵습니다.
툴 설명에 “언제 호출하는지 / 인자 형식 / 반환값을 어떻게 해석해야 하는지”를 조금 더 넣으면 tool 선택과 해석이 더 안정적일 것 같습니다!

### 코드 리뷰 댓글: @Gongmeda

시각: 2026-06-30T17:42:22Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/27#discussion_r3500755975
파일: `student_parts/week01_wake_up_nana.py` / 줄: 236
커밋: `877ba46be3b1da1e55eaeed63537f82f7ae1222d`

````diff
@@ -206,7 +232,11 @@ def week01_prompt_parts() -> list[str]:
     """1주차부터 누적되는 system prompt 조각입니다."""
 
     return [
-        # TODO: Week 1 Nana 일정 agent system prompt를 자유롭게 추가하세요.
+        f"""당신은 개인 일정 관리를 도와주는 어시스턴트 Nana입니다.
+오늘 날짜는 {current_app_date_iso()} 입니다.
````

오늘 날짜를 계산하여 시스템 프롬프트에 넣어서 에이전트가 참고할 수 있게 하셨군요! 👍

### 코드 리뷰 댓글: @Gongmeda

시각: 2026-06-30T17:43:01Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/27#discussion_r3500759907
파일: `student_parts/week01_wake_up_nana.py` / 줄: 239
커밋: `877ba46be3b1da1e55eaeed63537f82f7ae1222d`

````diff
@@ -206,7 +232,11 @@ def week01_prompt_parts() -> list[str]:
     """1주차부터 누적되는 system prompt 조각입니다."""
 
     return [
-        # TODO: Week 1 Nana 일정 agent system prompt를 자유롭게 추가하세요.
+        f"""당신은 개인 일정 관리를 도와주는 어시스턴트 Nana입니다.
+오늘 날짜는 {current_app_date_iso()} 입니다.
+
+일정 관련 요청이 오면 적절한 tool을 사용해서 처리하고, 결과를 자연스러운 한국어로 답해주세요.
+날짜나 시간 정보가 부족하면 사용자에게 물어보세요.""",
````

역질문을 유도하는 프롬프트가 인상적이네요! 👍

### 코드 리뷰 댓글: @Gongmeda

시각: 2026-06-30T17:44:46Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/27#discussion_r3500770277
파일: `student_parts/week01_wake_up_nana.py` / 줄: 216
커밋: `877ba46be3b1da1e55eaeed63537f82f7ae1222d`

````diff
@@ -170,24 +170,50 @@ def personal_create_schedule(
 ) -> str:
     """Nana의 개인 일정을 현재 대화의 임시 메모리에 생성합니다."""
 
-    # TODO: PERSONAL_SCHEDULES에 현재 대화 범위의 개인 일정을 생성하세요.
-    ...
+    new_schedule = {
+        "id": _new_personal_id(),
+        "title": title,
+        "date": date,
+        "start_time": start_time,
+        "end_time": end_time,
+        "attendees": attendees if attendees is not None else [],
+        "created_at": _now_iso(),
+        "session_id": current_session_scope(),
+    }
+    PERSONAL_SCHEDULES.append(new_schedule)
+    return _json({"ok": True, "tool_name": "personal_create_schedule", "created_schedule": new_schedule})
 
 
 @tool
 def personal_list_schedules(date_from: str | None = None, date_to: str | None = None) -> str:
     """선택한 시작일과 종료일 범위에 포함되는 Nana의 개인 일정을 조회합니다."""
 
-    # TODO: 현재 대화 범위의 PERSONAL_SCHEDULES를 날짜 조건으로 조회하세요.
-    ...
+    # 현재 세션 것만 가져옴
+    result = _current_session_schedules()
+
+    if date_from:
+        result = [s for s in result if s["date"] >= date_from]
+    if date_to:
+        result = [s for s in result if s["date"] <= date_to]
+
+    return _json({"ok": True, "tool_name": "personal_list_schedules", "schedules": result})
 
 
 @tool
 def personal_delete_schedule(schedule_id: str) -> str:
     """일정 ID에 해당하는 개인 일정을 삭제합니다."""
 
-    # TODO: 현재 대화 범위에서 schedule_id가 일치하는 개인 일정을 삭제하세요.
-    ...
+    session_id = current_session_scope()
+    before_len = len(PERSONAL_SCHEDULES)
+
+    # 같은 세션이면서 id 일치하는 것만 제거
+    PERSONAL_SCHEDULES[:] = [
+        s for s in PERSONAL_SCHEDULES
+        if not (s["id"] == schedule_id and _schedule_scope(s) == session_id)
+    ]
+
+    deleted = before_len - len(PERSONAL_SCHEDULES)
+    return _json({"ok": True, "tool_name": "personal_delete_schedule", "deleted": deleted})
````

삭제 대상이 없을 때도 `{"ok": true, "deleted": 0}`가 반환되어, 모델이 `ok`만 보고 삭제 성공처럼 해석할 여지가 있을 것 같아요.
“현재 대화 범위에서 해당 일정을 찾지 못했다”는 의미가 더 분명히 드러나게 정리해보면 좋을 것 같습니다!

### 리뷰: @Gongmeda / APPROVED

시각: 2026-06-30T17:47:13Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/27#pullrequestreview-4602749614

안녕하세요~
이번 주차 과제 진행하시느라 수고 많으셨습니다.

몇 가지는 기능 오류라기보다는 에이전트가 툴을 더 안정적으로 선택하고 답변을 만들 수 있게 하는 관점의 의견이라 어프루브 드렸습니다.

라인별로 코멘트 남겼으니 확인 부탁드립니다~

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-07-04T00:31:17Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/27#discussion_r3522288650
파일: `student_parts/week01_wake_up_nana.py` / 줄: 171
커밋: `877ba46be3b1da1e55eaeed63537f82f7ae1222d`

````diff
@@ -170,24 +170,50 @@ def personal_create_schedule(
 ) -> str:
     """Nana의 개인 일정을 현재 대화의 임시 메모리에 생성합니다."""
````

docstring이 그대로 tool description으로 들어간다는 점을 놓치고 있었네요. 호출 시점이랑 인자 형식을 명시해야 tool 선택이 안정적이라는 것 배워갑니다! 다음 주차부터 반영하겠습니다!

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-07-04T00:31:17Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/27#pullrequestreview-4628555942

(본문 없음)

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-07-04T00:31:48Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/27#discussion_r3522289290
파일: `student_parts/week01_wake_up_nana.py` / 줄: 216
커밋: `877ba46be3b1da1e55eaeed63537f82f7ae1222d`

````diff
@@ -170,24 +170,50 @@ def personal_create_schedule(
 ) -> str:
     """Nana의 개인 일정을 현재 대화의 임시 메모리에 생성합니다."""
 
-    # TODO: PERSONAL_SCHEDULES에 현재 대화 범위의 개인 일정을 생성하세요.
-    ...
+    new_schedule = {
+        "id": _new_personal_id(),
+        "title": title,
+        "date": date,
+        "start_time": start_time,
+        "end_time": end_time,
+        "attendees": attendees if attendees is not None else [],
+        "created_at": _now_iso(),
+        "session_id": current_session_scope(),
+    }
+    PERSONAL_SCHEDULES.append(new_schedule)
+    return _json({"ok": True, "tool_name": "personal_create_schedule", "created_schedule": new_schedule})
 
 
 @tool
 def personal_list_schedules(date_from: str | None = None, date_to: str | None = None) -> str:
     """선택한 시작일과 종료일 범위에 포함되는 Nana의 개인 일정을 조회합니다."""
 
-    # TODO: 현재 대화 범위의 PERSONAL_SCHEDULES를 날짜 조건으로 조회하세요.
-    ...
+    # 현재 세션 것만 가져옴
+    result = _current_session_schedules()
+
+    if date_from:
+        result = [s for s in result if s["date"] >= date_from]
+    if date_to:
+        result = [s for s in result if s["date"] <= date_to]
+
+    return _json({"ok": True, "tool_name": "personal_list_schedules", "schedules": result})
 
 
 @tool
 def personal_delete_schedule(schedule_id: str) -> str:
     """일정 ID에 해당하는 개인 일정을 삭제합니다."""
 
-    # TODO: 현재 대화 범위에서 schedule_id가 일치하는 개인 일정을 삭제하세요.
-    ...
+    session_id = current_session_scope()
+    before_len = len(PERSONAL_SCHEDULES)
+
+    # 같은 세션이면서 id 일치하는 것만 제거
+    PERSONAL_SCHEDULES[:] = [
+        s for s in PERSONAL_SCHEDULES
+        if not (s["id"] == schedule_id and _schedule_scope(s) == session_id)
+    ]
+
+    deleted = before_len - len(PERSONAL_SCHEDULES)
+    return _json({"ok": True, "tool_name": "personal_delete_schedule", "deleted": deleted})
````

말씀 듣고 보니 ok 값의 의미 자체가 애매했던 것 같습니다. 삭제 실패가 명확히 드러나는 응답 구조로 다음 주차에 개선해보겠습니다. 감사합니다!

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-07-04T00:31:48Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/27#pullrequestreview-4628556522

(본문 없음)

## 변경 파일

- `student_parts/week01_wake_up_nana.py` (+37 / -7)
