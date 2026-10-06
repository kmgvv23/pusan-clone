# #208 week6: supervisor와 Nana/Kana 하위 에이전트 분리 및 그룹 시간 결정 구현

원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/208

작성자: @kmgvv23
작성 시각: 2026-08-05T15:24:56Z
보관 시각 (UTC): 2026-10-06T15:19:25.015347+00:00

## PR 본문

## 과제 목표

- 한 agent가 tool을 전부 들고 있던 구조를 supervisor와 하위 에이전트로 나눠보기
- 여러 사람의 바쁜 시간에서 공통 가능 시간을 찾아 최종 회의 시간까지 정해보기

---

## 과제 위치
- 작업 브랜치 : `kimmingyu/week6`  → 본인 통합 브랜치 `kimmingyu/final` 로 PR
- 주요 파일 : `student_parts/week06_kanamate_decides_schedule.py`
- 함께 반영 : `student_parts/week05_load_kanas_past_conversations.py` (공지된 정답 코드 업데이트)

> 제공 코드(`fixed/` 등)는 수정하지 않고, `student_parts/` 의 본인 주차 파일을 구현했습니다.

---

## 과제 범위

- [x] 메인 과제 완료
- [x] 심화 과제까지 완료

---

## 구현한 기능 (메인)

- [x] `week06_prompt_parts` / `nana_prompt_parts` / `kana_prompt_parts` / `supervisor_system_prompt` — 세 agent의 역할 분담 프롬프트
- [x] `nana_agent` — Week 4 tool을 가진 Nana 하위 agent 실행, answer/trace/inner_tool_names 반환
- [x] `kana_agent` — Kana 하위 trace에서 `final_slot_payload` / `final_decision_payload` 끌어올리기

---

## 심화 과제

- [x] `FIND_COMMON_AVAILABLE_SLOTS_DESCRIPTION` / `DECIDE_FINAL_SLOT_DESCRIPTION` — tool이 후보나 최종 시간을 계산하지 않는다는 계약 명시
- [x] `find_common_available_slots_dict` / `find_common_available_slots` — agent가 고른 후보를 busy_rows와 대조해 검증
- [x] `decide_final_slot` — agent가 고른 최종 시간을 계약 payload로 기록

---

## Week 5 정답 코드 업데이트 반영

공지된 A~E 가이드를 제 구현에 맞춰 반영했습니다. 이 rows가 Week 6의 busy_rows 근거가 되므로 먼저 고쳤습니다.

- 앱 일정 row의 `request_kind`를 읽어 개인/그룹을 구분하고, notes에 그룹 참석자를 남깁니다.
- 앱 DB와 공유 저장소가 같은 일정을 다르게 다듬어(제목 소괄호 제거, `end_time` 기본값 치환) 값 비교로는 걸러지지 않는 중복을 키 기반으로 제거했습니다.
- `kind` 필터 제거(A)는 이미 걸고 있지 않아 그대로 두었습니다.

---

## AI 활용 내역

- Week 6 구조(supervisor와 하위 에이전트의 책임 경계, 어디까지가 학생 구현 대상인지)를 파악하는 데 AI를 활용했습니다.
- `fixed/schedule_decision.py`가 무엇을 검증해 주는지 확인해, tool에서 겹침 계산을 다시 짜지 않도록 정리했습니다.
- **왜 이 구조인지 물어봤습니다** : LLM은 확률 기반이라 겹치는 시간을 찾는 것처럼 규칙이 분명한 작업은 코드로 하는 게 낫다고 생각했는데, 이 과제는 후보를 LLM이 고르게 합니다. 이유를 확인해보니 "가장 이른 시간이 좋다", "오전이 집중이 잘 된다" 같은 맥락은 코드로 표현하기 어렵고, 대신 그렇게 고른 후보가 실제로 비는 시간인지는 코드가 검증한다는 분업이었습니다. 판단은 LLM이 하고 검증은 코드가 받는 구조를 보고 나서 납득이 됐습니다.
- 그룹 조율이 중간에 멈추는 원인을 좁히는 과정에서 trace를 어떻게 읽을지 논의했습니다.

---

## 구현하면서 고민한 점

- **description만으로는 부족했습니다** : 처음 그룹 조율을 실행했을 때 Kana가 `candidate_slots`를 비운 채 `find_common_available_slots`를 호출했고, 검증할 후보가 없으니 `decide_final_slot`까지 가지 못한 채 답변이 끝났습니다. description에는 이미 "이 tool은 후보를 계산해주지 않는다"를 적어둔 상태였습니다. Kana 프롬프트에 세 tool을 순서대로 모두 호출해야 끝난다는 점, 후보를 최소 3개 직접 만들어야 한다는 점, busy_rows가 비어 있어도 업무 시간 안에서 후보를 만들어야 한다는 점을 명시하고 나서야 흐름이 끝까지 이어졌습니다.

- **중복을 왜 이렇게 걸러야 하는지 고민했습니다** : Week 5 수정에서 같은 일정이 두 벌로 들어오는 것을 키 기반으로 걸러냈는데, 애초에 저장할 때 형식을 통일하면 중복 자체가 생기지 않을 것 같았습니다. 확인해보니 제목의 소괄호를 지우는 것도, `end_time` 기본값을 채우는 것도 `fixed/`에 있어 수정 대상이 아니었습니다. 근본 원인을 못 고치는 상황에서는 읽어온 뒤에 맞춰서 거르는 방법밖에 없다는 것을 알게 됐습니다. 남이 만든 시스템의 데이터를 쓸 때 흔히 겪는 상황이라고 생각합니다.

- **누적 프롬프트의 방향이 서로 달랐습니다** : supervisor는 5주차까지의 "직접 tool을 호출하라"는 지시를 그대로 물려받는데 이번 주에는 위임만 해야 해서, 그 지시가 이제 하위 에이전트에게 적용된다는 점을 먼저 정리했습니다. 반대로 Kana는 누적이 없어 역할과 날짜 기준부터 처음부터 써야 했습니다.

---

## 멘토님께 드리는 질문

과정을 마무리하면서, 6주 내내 반복해서 걸렸던 질문들이 있어 여쭤보고 싶습니다.

**1) 어디까지 룰 베이스로 하고, 어디부터 LLM에 맡기는 것이 좋을지 궁금합니다**

이번 주차의 시간 후보 선정처럼 규칙이 분명해 보이는 작업도 맥락 판단이 섞이면 LLM에 맡기는 편이 낫다는 걸 배웠습니다. 다만 그 경계를 제 나름의 기준으로 정리하지는 못했습니다. 같은 맥락에서 어디까지 단일 에이전트로 두고 어디서부터 하위 에이전트로 나눠야 하는지도 아직 명확한 기준을 못 세우겠습니다. 실무에서는 이런 경계를 어떤 기준으로 정하시는지 궁금합니다.

**2) 프롬프트로 어디까지 제어하는 것이 적절한지 궁금합니다**

이번 주차는 프롬프트로 제어하는 부분이 정말 많았습니다. tool description에 계약을 적어두어도 지켜지지 않아, 프롬프트에 호출 순서와 후보 개수까지 더 적고 나서야 원하는 대로 동작했습니다. 이렇게 자연어에 의지해 길게 지시하는 방식이 맞는 방향인지 의문이 들었습니다. 결과를 테스트해서 잘 나오기만 하면 프롬프트가 길고 촘촘해져도 괜찮은 것인지, 아니면 프롬프트가 길어질 때 그중 일부는 코드로 옮기는 편이 나은지 궁금합니다.

**3) 확률 기반 모델을 어디까지 신뢰할 수 있게 끌어올릴 수 있는지 궁금합니다**

XAI 수준까지는 아니더라도, 확률 기반인 LLM의 정확도와 일관성을 어느 정도까지 올릴 수 있는지, 그리고 그걸 위해 실무에서 어떤 기법과 테스트를 쓰시는지 궁금합니다. 이번 과제에서는 LLM 호출이 없는 경로만 pytest로 검증하고 라우팅이나 tool 호출 순서는 직접 실행해 눈으로 확인했는데, 매번 결과가 달라질 수 있는 부분을 어떻게 검증하시는지도 함께 여쭤보고 싶습니다.

---

## 과제 회고 (KPT)

- **Keep** : Week 5 수정 사항을 그냥 적용하지 않고 왜 그렇게 해야 하는지 이유까지 뜯어본 점. 지난 주차보다 코드를 더 이해한 상태에서 진행한 점. 이 구조를 왜 이렇게 처리하는지 계속 의문을 가지면서 본 점.
- **Problem** : 아직 어디까지 룰 베이스로 하고 어디부터 LLM에 맡길지, 어디까지 단일 에이전트로 두고 어디서부터 하위 에이전트로 나눌지에 대한 제 기준이 부족합니다. LLM 서비스의 테스트 기법도 더 연구해야 할 것 같습니다.
- **Try** : Problem에 적은 것들을 더 고민하고 찾아보기. 6주 동안 진행한 전체 과정을 한 번씩 다시 훑어보기.


## 리뷰와 대화

### 코드 리뷰 댓글: @doheelab-coder

시각: 2026-08-06T14:25:38Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/208#discussion_r3729630339
파일: `student_parts/week06_kanamate_decides_schedule.py` / 줄: 202
커밋: `3d665aebeeb8b891cffef2579ae7fe24b4325494`

````diff
@@ -197,9 +198,14 @@ def week06_prompt_parts() -> list[str]:
 
     return [
         *week05_prompt_parts(),
-        # TODO: Week 6 supervisor agent system prompt를 자유롭게 추가하세요.
-        #   - supervisor는 직접 업무를 처리하지 않고 nana_agent 또는 kana_agent로만 위임합니다.
-        #   - 어떤 요청이 Nana 담당이고 어떤 요청이 Kana 담당인지 판단 기준을 적습니다.
+        "이번 주차부터 너는 직접 일정을 처리하는 agent가 아니라 supervisor다. "
+        "이전 주차에서 직접 tool을 호출하라고 안내한 지시는 하위 에이전트에게 적용되고, "
````

`week05_prompt_parts()`의 마지막 조각에 "여러 사람의 최종 회의 시간을 확정하는 것은 이번 주차 범위가 아니므로 ... 설명하는 데까지만 답한다"가 있습니다(`week05_load_kanas_past_conversations.py:563`). 이번 주차 목표와 반대되는 지시가 supervisor에게 그대로 들어갑니다.

직접 쓰신 문장이 뒤에 있어서 지금 당장 깨지지는 않지만, 답변이 확정 시간을 빠뜨리는 쪽으로 흔들릴 수 있습니다. 여기에 한 문장 더 넣으면 정리됩니다.

```python
        "이전 주차에서 최종 회의 시간 확정은 범위가 아니라고 안내한 지시는 이번 주차에 해제된다. "
        "kana_agent가 final_slot을 돌려주면 그 확정 시간을 최종 답변에 반드시 포함한다.",
```

### 코드 리뷰 댓글: @doheelab-coder

시각: 2026-08-06T14:25:38Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/208#discussion_r3729630351
파일: `tests/test_week06.py` / 줄: 84
커밋: `3d665aebeeb8b891cffef2579ae7fe24b4325494`

````diff
@@ -0,0 +1,125 @@
+"""Week 6 공통 시간 후보 검증과 최종 결정 tool 테스트.
+
+LLM 호출이나 MCP subprocess가 필요 없는 경로만 검증한다.
+busy_rows를 직접 넘겨 후보 검증 규칙을, 선택 값을 직접 넘겨 최종 결정 기록을 확인한다.
+실행: uv run --with pytest python -m pytest -q tests/test_week06.py
+"""
+
+import json
+
+from student_parts.week06_kanamate_decides_schedule import (
+    agent_tool_names,
+    decide_final_slot,
+    find_common_available_slots,
+)
+
+BUSY_ROWS = [
+    {
+        "member_name": "나",
+        "date": "2026-08-10",
+        "start_time": "14:00",
+        "end_time": "15:00",
+        "title": "팀 회의",
+    },
+    {
+        "member_name": "민준",
+        "date": "2026-08-10",
+        "start_time": "10:00",
+        "end_time": "11:00",
+        "title": "API 리뷰",
+    },
+]
+
+
+def _candidate(start_time: str, end_time: str, reason: str = "후보"):
+    return {
+        "date": "2026-08-10",
+        "start_time": start_time,
+        "end_time": end_time,
+        "duration_minutes": 60,
+        "reason": reason,
+    }
+
+
+def _find_common(candidate_slots):
+    return json.loads(
+        find_common_available_slots.invoke(
+            {
+                "member_names": ["민준"],
+                "date_from": "2026-08-10",
+                "date_to": "2026-08-10",
+                "busy_rows": BUSY_ROWS,
+                "candidate_slots": candidate_slots,
+            }
+        )
+    )
+
+
+def test_busy_time_candidate_is_rejected():
+    # busy_rows와 겹치는 후보는 검증에서 제외되고, 비는 시간만 남는다.
+    payload = _find_common([_candidate("09:00", "10:00"), _candidate("14:00", "15:00")])
+    start_times = [slot["start_time"] for slot in payload["candidate_slots"]]
+    assert start_times == ["09:00"]
+
+
+def test_candidate_outside_workday_is_rejected():
+    # 업무 시간(기본 09:00~18:00) 밖의 후보도 남지 않는다.
+    payload = _find_common([_candidate("07:00", "08:00")])
+    assert payload["candidate_slots"] == []
+
+
+def test_iso_datetime_date_bound_is_normalized():
+    # date_from/date_to에 ISO datetime이 들어와도 날짜 부분만 사용한다.
+    payload = json.loads(
+        find_common_available_slots.invoke(
+            {
+                "member_names": ["민준"],
+                "date_from": "2026-08-10T00:00:00",
+                "date_to": "2026-08-10T23:59:59",
+                "busy_rows": BUSY_ROWS,
+                "candidate_slots": [_candidate("09:00", "10:00")],
+            }
+        )
+    )
+    assert len(payload["candidate_slots"]) == 1
````

이 검사는 후보가 1개 남았는지만 봅니다. 그런데 날짜를 잘라내는 처리가 `fixed/schedule_decision.py:58-59`에도 있어서, 직접 쓰신 `normalize_date_bound`를 지워도 결과는 그대로 1개입니다. 이 테스트는 통과하지만 직접 쓰신 처리가 동작했는지까지는 가려내지 못한다는 점만 알아두시면 됩니다.

### 코드 리뷰 댓글: @doheelab-coder

시각: 2026-08-06T14:25:38Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/208#discussion_r3729630358
파일: `student_parts/week05_load_kanas_past_conversations.py` / 줄: 315
커밋: `3d665aebeeb8b891cffef2579ae7fe24b4325494`

````diff
@@ -288,6 +297,46 @@ def _structured_request_from_schedule_row(row: dict[str, Any]) -> StructuredRequ
     )
 
 
+def _my_schedule_notes(request: StructuredRequest) -> str:
+    """내 일정 row가 개인 일정인지, 참석자가 있는 그룹 일정인지 설명합니다."""
+
+    if request.kind != "group_schedule":
+        return "Nana 개인 일정"
+    members = [
+        str(member).strip() for member in (request.members or []) if str(member).strip()
+    ]
+    return (
+        f"Nana 그룹 일정 · 참석자: {', '.join(members)}"
+        if members
+        else "Nana 그룹 일정"
+    )
+
+
+def _dedupe_schedule_rows(rows: list[dict[str, Any]]) -> list[dict[str, Any]]:
````

이 helper가 막으려는 중복은 현재 구현에서는 생기지 않습니다. `_collect_member_schedules`가 외부 조회 대상에서 "나"를 이미 빼고 있어서, 공유 저장소의 "나" 복사본은 애초에 들어오지 않습니다.

동작에는 문제가 없고 안전망으로는 쓸모가 있습니다. 다만 PR 본문에 적으신 중복 원인은 실제와 다르다는 점만 알아두시면 됩니다.

### 리뷰: @doheelab-coder / CHANGES_REQUESTED

시각: 2026-08-06T14:34:18Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/208#pullrequestreview-4875685977

week6 1차 리뷰입니다. 메인과 심화를 모두 구현하셨습니다. tool description을 `fixed/schedule_decision.py`가 실제로 후보를 버리는 조건 네 개에 맞춰 쓰신 것을 확인했습니다.

필수 수정은 한 가지입니다. 물려받은 이전 주차 프롬프트에 이번 주차와 반대되는 지시가 남아 있습니다. 나머지 인라인 두 개는 참고입니다. 반영해 주시면 확인 후 승인드리겠습니다.

---

## 질문 세 개 답변

**1) 어디까지 룰 베이스로 하고, 어디부터 LLM에 맡길지**

**기준은 "정답이 입력만으로 정해지는가" 하나입니다.** 같은 입력에 항상 같은 답이 나오고 그 답을 규칙으로 적을 수 있으면 코드입니다. 시간이 겹치는지 계산하는 일이 그렇습니다. 반대로 답이 맥락에 따라 달라지고 규칙으로 적으면 예외가 계속 붙는다면 LLM입니다.

에이전트를 나누는 기준도 같습니다. 한 프롬프트 안에서 규칙이 부딪히거나, tool이 많아져 모델이 엉뚱한 것을 고르기 시작하면 나눕니다.

대신 대가가 있습니다. 위임을 한 번 거칠 때마다 LLM 호출이 늘고, 앞에서 알던 것이 뒤로 전달되지 않습니다. Kana가 오늘 날짜를 모르던 문제가 그것입니다. 그래서 하나로 만들어 보고, 프롬프트가 싸우기 시작하면 자르는 순서로 갑니다.

---

**2) 프롬프트로 어디까지 제어하는 것이 적절한가**

**프롬프트가 긴 것 자체는 문제가 아닙니다.** 문제는 같은 규칙을 여러 곳에 나눠 적는 것입니다. 그러면 나중에 어느 문장이 지금 동작을 만드는지 알 수 없게 됩니다.

"안 지켜져서 문장을 더 세게 쓴다"가 반복되면, 프롬프트를 더 쓸 게 아니라 형식이나 코드로 옮길 신호입니다. 프롬프트는 확률을 높이는 수단이지 보장이 아니라서, description에 계약을 적어도 안 지켜지는 게 정상입니다.

옮길 곳이 형식일 때도 있고 코드일 때도 있습니다. `final_slot`의 `'YYYY-MM-DD HH:MM-HH:MM'` 형식은 지금 description에 말로 적혀 있는데, 이런 건 스키마에 정규식을 걸면 형식이 어긋난 값이 tool에 들어오기 전에 걸립니다.

---

**3) 확률 기반 모델을 어디까지 신뢰할 수 있게 만드나**

**100%는 안 됩니다.** 대신 "얼마나 자주 맞는지"를 숫자로 만들어 관리합니다. 목표를 100%로 잡는 대신, 틀렸을 때 손해가 큰 자리에는 코드 검증을 세우고 나머지는 통과율로 지켜보는 식입니다.

정확도를 올리는 기법은 대체로 모델이 고를 수 있는 선택지를 줄이는 쪽입니다. temperature를 낮추고, 출력 형식을 고정하고, tool 수와 분기를 줄이고, 판단 기준을 프롬프트에 예시로 보여줍니다.

매번 달라지는 부분은 통과 여부가 아니라 통과율로 봅니다. supervisor를 실제로 호출해 어느 하위 에이전트가 선택됐는지, tool이 어떤 순서로 불렸는지 확인하는 테스트를 만들고, 그 테스트만 여러 번 돌려 20회 중 몇 번 통과하는지 기록합니다. 그 숫자를 남겨 두면 프롬프트를 고친 뒤 떨어졌는지로 회귀를 잡을 수 있습니다.

**신뢰도는 모델을 믿을 만하게 만들어서 얻는 게 아니라, 틀린 출력이 통과하지 못하게 만들어서 얻습니다.**

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-08-09T07:11:42Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/208#discussion_r3742988696
파일: `student_parts/week06_kanamate_decides_schedule.py` / 줄: 202
커밋: `3d665aebeeb8b891cffef2579ae7fe24b4325494`

````diff
@@ -197,9 +198,14 @@ def week06_prompt_parts() -> list[str]:
 
     return [
         *week05_prompt_parts(),
-        # TODO: Week 6 supervisor agent system prompt를 자유롭게 추가하세요.
-        #   - supervisor는 직접 업무를 처리하지 않고 nana_agent 또는 kana_agent로만 위임합니다.
-        #   - 어떤 요청이 Nana 담당이고 어떤 요청이 Kana 담당인지 판단 기준을 적습니다.
+        "이번 주차부터 너는 직접 일정을 처리하는 agent가 아니라 supervisor다. "
+        "이전 주차에서 직접 tool을 호출하라고 안내한 지시는 하위 에이전트에게 적용되고, "
````

5주차 프롬프트 파트는 미쳐 생각 못한 것 같습니다. 지적해주셔서 감사합니다!

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-08-09T07:11:42Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/208#pullrequestreview-4890749569

(본문 없음)

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-08-09T07:37:49Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/208#discussion_r3743048696
파일: `tests/test_week06.py` / 줄: 84
커밋: `3d665aebeeb8b891cffef2579ae7fe24b4325494`

````diff
@@ -0,0 +1,125 @@
+"""Week 6 공통 시간 후보 검증과 최종 결정 tool 테스트.
+
+LLM 호출이나 MCP subprocess가 필요 없는 경로만 검증한다.
+busy_rows를 직접 넘겨 후보 검증 규칙을, 선택 값을 직접 넘겨 최종 결정 기록을 확인한다.
+실행: uv run --with pytest python -m pytest -q tests/test_week06.py
+"""
+
+import json
+
+from student_parts.week06_kanamate_decides_schedule import (
+    agent_tool_names,
+    decide_final_slot,
+    find_common_available_slots,
+)
+
+BUSY_ROWS = [
+    {
+        "member_name": "나",
+        "date": "2026-08-10",
+        "start_time": "14:00",
+        "end_time": "15:00",
+        "title": "팀 회의",
+    },
+    {
+        "member_name": "민준",
+        "date": "2026-08-10",
+        "start_time": "10:00",
+        "end_time": "11:00",
+        "title": "API 리뷰",
+    },
+]
+
+
+def _candidate(start_time: str, end_time: str, reason: str = "후보"):
+    return {
+        "date": "2026-08-10",
+        "start_time": start_time,
+        "end_time": end_time,
+        "duration_minutes": 60,
+        "reason": reason,
+    }
+
+
+def _find_common(candidate_slots):
+    return json.loads(
+        find_common_available_slots.invoke(
+            {
+                "member_names": ["민준"],
+                "date_from": "2026-08-10",
+                "date_to": "2026-08-10",
+                "busy_rows": BUSY_ROWS,
+                "candidate_slots": candidate_slots,
+            }
+        )
+    )
+
+
+def test_busy_time_candidate_is_rejected():
+    # busy_rows와 겹치는 후보는 검증에서 제외되고, 비는 시간만 남는다.
+    payload = _find_common([_candidate("09:00", "10:00"), _candidate("14:00", "15:00")])
+    start_times = [slot["start_time"] for slot in payload["candidate_slots"]]
+    assert start_times == ["09:00"]
+
+
+def test_candidate_outside_workday_is_rejected():
+    # 업무 시간(기본 09:00~18:00) 밖의 후보도 남지 않는다.
+    payload = _find_common([_candidate("07:00", "08:00")])
+    assert payload["candidate_slots"] == []
+
+
+def test_iso_datetime_date_bound_is_normalized():
+    # date_from/date_to에 ISO datetime이 들어와도 날짜 부분만 사용한다.
+    payload = json.loads(
+        find_common_available_slots.invoke(
+            {
+                "member_names": ["민준"],
+                "date_from": "2026-08-10T00:00:00",
+                "date_to": "2026-08-10T23:59:59",
+                "busy_rows": BUSY_ROWS,
+                "candidate_slots": [_candidate("09:00", "10:00")],
+            }
+        )
+    )
+    assert len(payload["candidate_slots"]) == 1
````

fixed까진 전체 확인 못했던 것 같습니다. 말씀해주셔서 감사합니다.

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-08-09T07:37:49Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/208#pullrequestreview-4890790545

(본문 없음)

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-08-09T07:39:04Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/208#discussion_r3743051514
파일: `student_parts/week05_load_kanas_past_conversations.py` / 줄: 315
커밋: `3d665aebeeb8b891cffef2579ae7fe24b4325494`

````diff
@@ -288,6 +297,46 @@ def _structured_request_from_schedule_row(row: dict[str, Any]) -> StructuredRequ
     )
 
 
+def _my_schedule_notes(request: StructuredRequest) -> str:
+    """내 일정 row가 개인 일정인지, 참석자가 있는 그룹 일정인지 설명합니다."""
+
+    if request.kind != "group_schedule":
+        return "Nana 개인 일정"
+    members = [
+        str(member).strip() for member in (request.members or []) if str(member).strip()
+    ]
+    return (
+        f"Nana 그룹 일정 · 참석자: {', '.join(members)}"
+        if members
+        else "Nana 그룹 일정"
+    )
+
+
+def _dedupe_schedule_rows(rows: list[dict[str, Any]]) -> list[dict[str, Any]]:
````

공지의 week5 변경 반영에만 집중하다 보니, 정작 제 코드와 비교가 소홀 했던 것 같습니다. 앞으로는 가이드를 반영할 때 제 코드 기준으로 다시 확인하는 습관을 들이겠습니다.



### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-08-09T07:39:04Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/208#pullrequestreview-4890792193

(본문 없음)

### PR 대화 댓글: @kmgvv23

시각: 2026-08-09T08:03:18Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/208#issuecomment-5230488718

제가 또 놓쳤던 부분이 많았던 것 같습니다. 리뷰해주셔서 감사합니다.
그리고, 제가 별개로 드렸던 질문에 명확하게 답변주셔서 정말 큰 도움 될 것 같고, 궁금증도 해소된 것 같습니다. 감사합니다!

### 리뷰: @doheelab-coder / APPROVED

시각: 2026-08-10T12:24:51Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/208#pullrequestreview-4896422473

반영 확인했습니다. 승인합니다.

추가하신 두 줄(209-210행) 덕분에 supervisor가 week5에서 물려받은 "확정은 이번 주차 범위가 아니다"를 더 이상 따르지 않습니다.

6주차까지 수고 많으셨습니다.

## 변경 파일

- `docs/week06.md` (+41 / -0)
- `student_parts/week05_load_kanas_past_conversations.py` (+54 / -3)
- `student_parts/week06_kanamate_decides_schedule.py` (+243 / -71)
- `tests/test_week06.py` (+125 / -0)
