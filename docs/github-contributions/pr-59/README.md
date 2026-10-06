# #59 week2: 자연어 요청 구조화 tool 구현

원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59

작성자: @kmgvv23
작성 시각: 2026-07-08T11:08:22Z
보관 시각 (UTC): 2026-10-06T15:19:25.015347+00:00

## PR 본문

## 과제 목표

- Pydantic 스키마와 structured output(response_format)으로 자연어를 구조화하는 방법 이해하기
- 자연어 요청이나 Week1 tool JSON을 StructuredRequestBatch로 변환하는 흐름 익히기
- 주차별 system prompt를 누적하는 구조(week01_prompt_parts 재사용) 이해하기

---

## 과제 위치
- 작업 브랜치 : `kimmingyu/week2`  → 본인 통합 브랜치 `kimmingyu/final` 로 PR
- 주요 파일 : `student_parts/week02_structure_natural_language_requests.py`

---

## 과제 범위

- [x] 메인 과제 완료
- [x] 심화 과제까지 완료

---

## 구현한 기능

- [x] `StructuredRequest` / `StructuredRequestBatch` 스키마 (필드별 description)
- [x] `week02_tools` / `week02_prompt_parts` / `week02_system_prompt`
- [x] `build_week02_agent` (`response_format=StructuredRequestBatch` 연결)

---

## 심화 과제

- [x] `_coerce_structured_request` — structured output 결과를 StructuredRequest로 정규화/검증
- [x] `extract_structured_request` — agent 없이 `with_structured_output`으로 자연어를 구조화
- [x] `extract_schedule_request` — 위 결과에 ok/tool_name/base_date를 붙여 Week3 저장 tool이 그대로 받을 수 있는 bridge tool

---

## AI 활용 내역

### 이번 주차 할 일 및 코드 구조 파악
- AI 활용 내용 : week2에서 뭘 해야 하는지, 파일 구조(StructuredRequest → Batch → agent 흐름)와 prompt_parts/system_prompt의 차이를 AI로 다시 설명 들으며 이해했다.

### Field description 작성
- AI 활용 내용 : Field의 description을 내가 먼저 적어본 뒤, AI로 더 구체적이고 직관적인 문구로 다듬었다.
- 직접 수정한 부분 : 클로드가 내가 쓴 것보다 알맞게 써준 것 같아서 크게 고친 부분은 없었다.

---

## 구현하면서 고민한 점
막혔던 부분, 고민한 내용, 해결 방법을 자유롭게 적어요.

- 고민한 점 : system prompt를 조각(list)과 완성 문자열(str)로 나누는 이유가 헷갈렸다.
- 해결 방법 : prompt_parts는 주차별로 누적하는 재료(list), system_prompt는 그걸 합친 최종본(str)이라는 역할 분리를 이해하고 구현했다.

---

## 과제 회고 (KPT)

- **Keep** (좋았고 계속 유지할 점) : 저번 주차에는 급하게 했지만, 이번엔 코드 전체를 이해하고 모르는 것은 그냥 넘기지 않고 계속 알아봤다.
- **Problem** (아쉬웠거나 막혔던 점) : structured output(Pydantic response_format)과 일반 tool calling의 차이가 처음엔 잘 안 잡혔다.
- **Try** (다음에 시도해볼 점) : 다음 주차에서 이 StructuredRequest 구조화 결과가 실제 저장(SQLite) 흐름에 어떻게 이어지는지까지 확인해봐야겠다. 또한, 계속 미뤄왔던 Attention, Transformer 등 기본 이론 공부도 병행해야겠다.

## 리뷰와 대화

### 코드 리뷰 댓글: @Gongmeda

시각: 2026-07-09T15:49:17Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#discussion_r3552976245
파일: `student_parts/week02_structure_natural_language_requests.py` / 줄: 157
커밋: `b09560a9cb88ad0500be3954edc569b15a51d575`

````diff
@@ -0,0 +1,266 @@
+from __future__ import annotations
+
+import json
+from typing import Any, Literal
+
+from langchain.agents import create_agent
+from langchain.tools import tool
+from pydantic import BaseModel, Field
+
+from fixed.config import CONFIG
+from fixed.llm import chat_model
+from fixed.runtime_clock import current_app_date_iso
+from student_parts.week01_wake_up_nana import join_system_prompt, week01_prompt_parts, week01_tools
+
+
+RequestKind = Literal["personal_schedule", "group_schedule", "todo", "reminder", "unknown"]
+_WEEK02_AGENT: Any | None = None
+
+
+# [2주차 수강생 구현 가이드]
+#
+# 목표
+#   Week 2의 핵심은 사용자의 한국어 자연어 요청이나 Week 1 tool이 만든 JSON payload를
+#   일정 앱이 읽을 수 있는 StructuredRequest/StructuredRequestBatch로 바꾸는 것입니다.
+#   Week 1이 이미 정해진 인자를 받아 임시 일정을 만들었다면, Week 2는 "내일 오후 3시" 같은
+#   자연어와 created_schedule JSON을 날짜/시간/종류/멤버 필드로 구조화합니다.
+#   구조화 결과는 아직 SQLite, RAG, 외부 멤버 일정 조율 흐름에 저장하지 않습니다.
+#
+# 과제 구성
+#   - 메인과제: Week 2 agent가 자연어 또는 Week 1 tool JSON을 StructuredRequestBatch로
+#     최종 반환하는 세로 슬라이스를 완성합니다.
+#   - 추가 과제: 메인과제에서 만든 StructuredRequest 스키마를 Week 3 이상 저장/조율 흐름에서
+#     재사용할 수 있도록 bridge 함수를 완성합니다.
+#
+# 구현 위치와 사용할 코드
+#   - 이 파일(student_parts/week02_structure_natural_language_requests.py)의
+#     StructuredRequest, StructuredRequestBatch, week02_tools(), week02_prompt_parts(),
+#     week02_system_prompt(), build_week02_agent()를 확인합니다.
+#   - build_week02_agent()는 langchain.agents.create_agent, fixed/llm.py의 chat_model(),
+#     week02_system_prompt(), response_format=StructuredRequestBatch를 사용해 Week 2 agent를 만듭니다.
+#   - week02_tools()는 Week 1 도구 목록을 그대로 가져옵니다. Week 2 agent는 개인 일정 생성 요청에서
+#     personal_create_schedule이 반환한 created_schedule JSON payload를 읽고
+#     response_format=StructuredRequestBatch로 최종 구조화 결과를 확인합니다.
+#   - week02_prompt_parts()는 student_parts/week01_wake_up_nana.py의 week01_prompt_parts() 위에
+#     Week 2 구조화 지시를 추가합니다.
+#   - _coerce_structured_request(), extract_structured_request(), extract_schedule_request()는
+#     Week 3 이상에서 재사용되는 구조화 bridge입니다. Week 2 파일에 있지만 Week 2 agent에
+#     공개되는 tool은 아닙니다.
+#
+# 메인과제 구현 대상
+#   1. StructuredRequest 스키마
+#      - kind/title/date/start_time/end_time/members/priority/reason/original_text 필드가
+#        이후 Week 3 저장 payload의 기준이 됩니다.
+#      - kind는 RequestKind Literal에 들어 있는 값만 허용합니다.
+#      - 각 필드에는 LLM structured output이 이해할 수 있도록 한국어 description을 붙입니다.
+#
+#   2. StructuredRequestBatch 스키마
+#      - requests에는 StructuredRequest 목록을 담고, 요청이 하나뿐이어도 list 형태를 유지합니다.
+#      - base_date에는 상대 날짜 해석 기준일(current_app_date_iso)을 담습니다.
+#
+#   3. Week 2 agent 세로 슬라이스
+#      - week02_tools()는 Week 1 tool 목록을 그대로 반환합니다.
+#      - week02_prompt_parts()와 week02_system_prompt()에는 자연어/Week 1 tool JSON을
+#        StructuredRequestBatch로 구조화하라는 지시를 넣습니다.
+#      - build_week02_agent()에 response_format=StructuredRequestBatch를 연결해
+#        ./run.sh --week2가 동작하게 합니다.
+#      - 개인 일정 생성 요청에서는 Week 1 personal_create_schedule tool 결과의 created_schedule JSON을
+#        LLM이 읽어 StructuredRequestBatch로 최종 변환하는 흐름을 확인합니다.
+#
+# 추가 과제 구현 대상
+#   1. _coerce_structured_request
+#      - LangChain structured output 결과가 이미 StructuredRequest이면 그대로 반환합니다.
+#      - dict이면 StructuredRequest.model_validate(...)로 검증해 반환합니다.
+#      - 예상한 형태가 아니면 RuntimeError를 발생시켜 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   2. extract_structured_request
+#      - chat_model().with_structured_output(StructuredRequest, method="function_calling")를 사용합니다.
+#      - system 메시지에는 join_system_prompt(week02_prompt_parts())를 넣고,
+#        user 메시지에는 text를 넣어 structured LLM을 호출합니다.
+#      - 자연어 또는 JSON 문자열을 StructuredRequest 하나로 검증/구조화합니다.
+#
+#   3. extract_schedule_request
+#      - extract_structured_request(query) 결과에 ok/tool_name/base_date를 붙입니다.
+#      - structured_request에는 model_dump() 결과를 넣고, json.dumps(..., ensure_ascii=False)로 반환합니다.
+#      - Week 3 이상 저장 tool이 structured_request 필드를 그대로 받을 수 있게 만듭니다.
+#
+# StructuredRequest 읽는 법
+#   - kind: personal_schedule, group_schedule, todo, reminder, unknown 중 하나입니다.
+#   - title/date/start_time/end_time: 일정 앱이 실제 저장이나 생성에 사용할 핵심 필드입니다.
+#   - members: 참석자/관련 멤버 list입니다. 모르면 빈 list로 둡니다.
+#   - priority/reason/original_text: 할 일 우선순위, 판단 근거, 원문 보존용 필드입니다.
+#   - 모르는 값을 억지로 만들지 않는 것이 중요합니다. 확실하지 않으면 None 또는 빈 list가 안전합니다.
+#   - date/start_time/end_time은 확실할 때만 YYYY-MM-DD, HH:MM 형식으로 채웁니다.
+#
+# bridge 동작 기준
+#   - 요청이 하나뿐이어도 Week 2 agent의 structured_response에는 StructuredRequest 하나를 담습니다.
+#   - 여러 일정/할 일/알림 의도가 한 문장에 섞이면 Week 2 agent에서는 여러 StructuredRequest로 나눕니다.
+#   - extract_structured_request()는 bridge 용도라 StructuredRequest 하나만 반환합니다.
+#   - Week 1 personal_create_schedule은 이미 분해된 인자로 임시 일정을 생성하고,
+#     Week 2 agent와 bridge는 그 JSON payload를 읽어 저장 가능한 구조로 최종 변환한다는 차이를 비교합니다.
+#
+# 참고 코드
+#   - week01_prompt_parts()
+#      Week 1 system prompt를 이어받아 Week 2 구조화 지시를 누적할 때 사용합니다.
+#   - week01_tools()
+#      Week 1 개인 일정 tool 목록입니다. Week 2 agent는 이 tool 결과 JSON을 구조화 근거로 씁니다.
+#   - extract_structured_request / extract_schedule_request
+#      Week 3 이상에서 DB 저장/조율 tool chain에 쓰는 bridge 코드입니다.
+#      query 문자열이 자연어든 Week 1 tool JSON이든, Python rule/parser로 매핑하지 않고
+#      structured LLM 호출로 구조화한 뒤 JSON tool payload로 감쌉니다.
+#
+# 검증 방법
+#   - 메인과제: ./run.sh --week2로 실행한 뒤 "다음 주 화요일 오후 3시에 철수랑 회의 잡아줘" 같은
+#     문장을 입력합니다. 최종 답변이 StructuredRequestBatch class 형식의 structured_response로
+#     나오는지 확인합니다.
+#   - 추가 과제: Week 3을 실행한 뒤 trace에서 extract_schedule_request 이후
+#     save_structured_request가 호출되는지 봅니다. extract_schedule_request의 반환 JSON에
+#     ok/tool_name/base_date/structured_request가 들어 있는지 확인합니다.
+#
+# 함수별 동작 설명
+#   - StructuredRequest
+#     Week 2 structured output의 중심 스키마입니다. LLM이 자연어에서 뽑은 요청 종류, 제목, 날짜, 시간,
+#     멤버, 우선순위, 근거, 원문을 이 class 필드에 맞춰 반환합니다.
+#
+#   - StructuredRequestBatch
+#     StructuredRequest 여러 개와 base_date를 함께 담는 최종 structured_response 스키마입니다.
+#     요청이 하나뿐이어도 requests list 안에 StructuredRequest 하나를 담습니다.
+#
+#   - week02_tools()
+#     Week 1 개인 일정 tool을 그대로 노출합니다. Week 2 agent는 개인 일정 생성 요청에서
+#     created_schedule JSON을 structured_response의 근거로 사용할 수 있습니다.
+#
+#   - week02_system_prompt() / week02_prompt_parts()
+#     Week 1 prompt 위에 "자연어를 StructuredRequestBatch로 출력한다"는 Week 2 지시를 누적합니다.
+#
+#   - build_week02_agent() / build_week_agent()
+#     response_format=StructuredRequestBatch가 설정된 agent를 만들고 재사용합니다.
+#     build_week_agent()는 실행기가 찾는 표준 entry point입니다.
+#
+#   - _coerce_structured_request(value)
+#     LangChain structured output 결과가 이미 StructuredRequest이면 그대로 쓰고, dict이면 Pydantic 검증을 거쳐
+#     StructuredRequest로 바꿉니다. 예상한 형태가 아니면 오류를 내서 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   - extract_structured_request(text)
+#     agent loop를 새로 만들지 않고 chat_model().with_structured_output(...)만 사용해 자연어 또는 JSON 문자열을
+#     StructuredRequest로 검증/구조화합니다. Week 3 이상에서 저장/조율 직전 입력을 구조화해야 할 때 재사용하는 bridge 함수입니다.
+#
+#   - extract_schedule_request(query)
+#     Week 3 이상 agent가 저장/조율 전에 호출하는 LangChain bridge tool입니다.
+#     extract_structured_request(...) 결과에 ok/tool_name/base_date를 붙여 JSON 문자열로 반환하므로,
+#     이후 저장 tool이 structured_request 필드를 그대로 받을 수 있습니다.
+
+
+class StructuredRequest(BaseModel):
+    """LLM structured output으로 추출되는 2주차 요청 스키마입니다."""
+    
+    kind: RequestKind = Field(description="요청 종류. personal_schedule, group_schedule, todo, reminder, unknown 중 하나로 분류한다.")
````

kind를 RequestKind Literal로 둔 점 좋네요! 👍
이 필드는 이후 저장/조율 흐름의 분기점이 될 수 있어서, 값 범위를 코드 차원에서 좁힌 건 이번 주차 핵심을 잘 잡으신 것 같습니다.

다만 에이전트 입장에서는 각 kind를 언제 선택해야 하는지 기준이 아직 조금 약할 수 있을 것 같아요.
예를 들어 `다음 주 화요일 오후 3시에 철수랑 회의 잡아줘`와 `철수랑 다음 주에 가능한 시간 찾아서 회의 잡아줘`는 둘 다 members가 있지만, 앞의 것은 확정된 개인 일정이고 뒤의 것은 조율이 필요한 group_schedule로 볼 수 있습니다.

`personal_schedule/group_schedule/todo/reminder/unknown` 각각에 대한 짧은 기준이나 예시를 조금 더 보강하고 확인해보시면 좋겠습니다!

### 코드 리뷰 댓글: @Gongmeda

시각: 2026-07-09T15:50:55Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#discussion_r3552987021
파일: `student_parts/week02_structure_natural_language_requests.py` / 줄: 238
커밋: `c456097a242a3fe4a24d2c1732449b666453ca5b`

````diff
@@ -0,0 +1,266 @@
+from __future__ import annotations
+
+import json
+from typing import Any, Literal
+
+from langchain.agents import create_agent
+from langchain.tools import tool
+from pydantic import BaseModel, Field
+
+from fixed.config import CONFIG
+from fixed.llm import chat_model
+from fixed.runtime_clock import current_app_date_iso
+from student_parts.week01_wake_up_nana import join_system_prompt, week01_prompt_parts, week01_tools
+
+
+RequestKind = Literal["personal_schedule", "group_schedule", "todo", "reminder", "unknown"]
+_WEEK02_AGENT: Any | None = None
+
+
+# [2주차 수강생 구현 가이드]
+#
+# 목표
+#   Week 2의 핵심은 사용자의 한국어 자연어 요청이나 Week 1 tool이 만든 JSON payload를
+#   일정 앱이 읽을 수 있는 StructuredRequest/StructuredRequestBatch로 바꾸는 것입니다.
+#   Week 1이 이미 정해진 인자를 받아 임시 일정을 만들었다면, Week 2는 "내일 오후 3시" 같은
+#   자연어와 created_schedule JSON을 날짜/시간/종류/멤버 필드로 구조화합니다.
+#   구조화 결과는 아직 SQLite, RAG, 외부 멤버 일정 조율 흐름에 저장하지 않습니다.
+#
+# 과제 구성
+#   - 메인과제: Week 2 agent가 자연어 또는 Week 1 tool JSON을 StructuredRequestBatch로
+#     최종 반환하는 세로 슬라이스를 완성합니다.
+#   - 추가 과제: 메인과제에서 만든 StructuredRequest 스키마를 Week 3 이상 저장/조율 흐름에서
+#     재사용할 수 있도록 bridge 함수를 완성합니다.
+#
+# 구현 위치와 사용할 코드
+#   - 이 파일(student_parts/week02_structure_natural_language_requests.py)의
+#     StructuredRequest, StructuredRequestBatch, week02_tools(), week02_prompt_parts(),
+#     week02_system_prompt(), build_week02_agent()를 확인합니다.
+#   - build_week02_agent()는 langchain.agents.create_agent, fixed/llm.py의 chat_model(),
+#     week02_system_prompt(), response_format=StructuredRequestBatch를 사용해 Week 2 agent를 만듭니다.
+#   - week02_tools()는 Week 1 도구 목록을 그대로 가져옵니다. Week 2 agent는 개인 일정 생성 요청에서
+#     personal_create_schedule이 반환한 created_schedule JSON payload를 읽고
+#     response_format=StructuredRequestBatch로 최종 구조화 결과를 확인합니다.
+#   - week02_prompt_parts()는 student_parts/week01_wake_up_nana.py의 week01_prompt_parts() 위에
+#     Week 2 구조화 지시를 추가합니다.
+#   - _coerce_structured_request(), extract_structured_request(), extract_schedule_request()는
+#     Week 3 이상에서 재사용되는 구조화 bridge입니다. Week 2 파일에 있지만 Week 2 agent에
+#     공개되는 tool은 아닙니다.
+#
+# 메인과제 구현 대상
+#   1. StructuredRequest 스키마
+#      - kind/title/date/start_time/end_time/members/priority/reason/original_text 필드가
+#        이후 Week 3 저장 payload의 기준이 됩니다.
+#      - kind는 RequestKind Literal에 들어 있는 값만 허용합니다.
+#      - 각 필드에는 LLM structured output이 이해할 수 있도록 한국어 description을 붙입니다.
+#
+#   2. StructuredRequestBatch 스키마
+#      - requests에는 StructuredRequest 목록을 담고, 요청이 하나뿐이어도 list 형태를 유지합니다.
+#      - base_date에는 상대 날짜 해석 기준일(current_app_date_iso)을 담습니다.
+#
+#   3. Week 2 agent 세로 슬라이스
+#      - week02_tools()는 Week 1 tool 목록을 그대로 반환합니다.
+#      - week02_prompt_parts()와 week02_system_prompt()에는 자연어/Week 1 tool JSON을
+#        StructuredRequestBatch로 구조화하라는 지시를 넣습니다.
+#      - build_week02_agent()에 response_format=StructuredRequestBatch를 연결해
+#        ./run.sh --week2가 동작하게 합니다.
+#      - 개인 일정 생성 요청에서는 Week 1 personal_create_schedule tool 결과의 created_schedule JSON을
+#        LLM이 읽어 StructuredRequestBatch로 최종 변환하는 흐름을 확인합니다.
+#
+# 추가 과제 구현 대상
+#   1. _coerce_structured_request
+#      - LangChain structured output 결과가 이미 StructuredRequest이면 그대로 반환합니다.
+#      - dict이면 StructuredRequest.model_validate(...)로 검증해 반환합니다.
+#      - 예상한 형태가 아니면 RuntimeError를 발생시켜 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   2. extract_structured_request
+#      - chat_model().with_structured_output(StructuredRequest, method="function_calling")를 사용합니다.
+#      - system 메시지에는 join_system_prompt(week02_prompt_parts())를 넣고,
+#        user 메시지에는 text를 넣어 structured LLM을 호출합니다.
+#      - 자연어 또는 JSON 문자열을 StructuredRequest 하나로 검증/구조화합니다.
+#
+#   3. extract_schedule_request
+#      - extract_structured_request(query) 결과에 ok/tool_name/base_date를 붙입니다.
+#      - structured_request에는 model_dump() 결과를 넣고, json.dumps(..., ensure_ascii=False)로 반환합니다.
+#      - Week 3 이상 저장 tool이 structured_request 필드를 그대로 받을 수 있게 만듭니다.
+#
+# StructuredRequest 읽는 법
+#   - kind: personal_schedule, group_schedule, todo, reminder, unknown 중 하나입니다.
+#   - title/date/start_time/end_time: 일정 앱이 실제 저장이나 생성에 사용할 핵심 필드입니다.
+#   - members: 참석자/관련 멤버 list입니다. 모르면 빈 list로 둡니다.
+#   - priority/reason/original_text: 할 일 우선순위, 판단 근거, 원문 보존용 필드입니다.
+#   - 모르는 값을 억지로 만들지 않는 것이 중요합니다. 확실하지 않으면 None 또는 빈 list가 안전합니다.
+#   - date/start_time/end_time은 확실할 때만 YYYY-MM-DD, HH:MM 형식으로 채웁니다.
+#
+# bridge 동작 기준
+#   - 요청이 하나뿐이어도 Week 2 agent의 structured_response에는 StructuredRequest 하나를 담습니다.
+#   - 여러 일정/할 일/알림 의도가 한 문장에 섞이면 Week 2 agent에서는 여러 StructuredRequest로 나눕니다.
+#   - extract_structured_request()는 bridge 용도라 StructuredRequest 하나만 반환합니다.
+#   - Week 1 personal_create_schedule은 이미 분해된 인자로 임시 일정을 생성하고,
+#     Week 2 agent와 bridge는 그 JSON payload를 읽어 저장 가능한 구조로 최종 변환한다는 차이를 비교합니다.
+#
+# 참고 코드
+#   - week01_prompt_parts()
+#      Week 1 system prompt를 이어받아 Week 2 구조화 지시를 누적할 때 사용합니다.
+#   - week01_tools()
+#      Week 1 개인 일정 tool 목록입니다. Week 2 agent는 이 tool 결과 JSON을 구조화 근거로 씁니다.
+#   - extract_structured_request / extract_schedule_request
+#      Week 3 이상에서 DB 저장/조율 tool chain에 쓰는 bridge 코드입니다.
+#      query 문자열이 자연어든 Week 1 tool JSON이든, Python rule/parser로 매핑하지 않고
+#      structured LLM 호출로 구조화한 뒤 JSON tool payload로 감쌉니다.
+#
+# 검증 방법
+#   - 메인과제: ./run.sh --week2로 실행한 뒤 "다음 주 화요일 오후 3시에 철수랑 회의 잡아줘" 같은
+#     문장을 입력합니다. 최종 답변이 StructuredRequestBatch class 형식의 structured_response로
+#     나오는지 확인합니다.
+#   - 추가 과제: Week 3을 실행한 뒤 trace에서 extract_schedule_request 이후
+#     save_structured_request가 호출되는지 봅니다. extract_schedule_request의 반환 JSON에
+#     ok/tool_name/base_date/structured_request가 들어 있는지 확인합니다.
+#
+# 함수별 동작 설명
+#   - StructuredRequest
+#     Week 2 structured output의 중심 스키마입니다. LLM이 자연어에서 뽑은 요청 종류, 제목, 날짜, 시간,
+#     멤버, 우선순위, 근거, 원문을 이 class 필드에 맞춰 반환합니다.
+#
+#   - StructuredRequestBatch
+#     StructuredRequest 여러 개와 base_date를 함께 담는 최종 structured_response 스키마입니다.
+#     요청이 하나뿐이어도 requests list 안에 StructuredRequest 하나를 담습니다.
+#
+#   - week02_tools()
+#     Week 1 개인 일정 tool을 그대로 노출합니다. Week 2 agent는 개인 일정 생성 요청에서
+#     created_schedule JSON을 structured_response의 근거로 사용할 수 있습니다.
+#
+#   - week02_system_prompt() / week02_prompt_parts()
+#     Week 1 prompt 위에 "자연어를 StructuredRequestBatch로 출력한다"는 Week 2 지시를 누적합니다.
+#
+#   - build_week02_agent() / build_week_agent()
+#     response_format=StructuredRequestBatch가 설정된 agent를 만들고 재사용합니다.
+#     build_week_agent()는 실행기가 찾는 표준 entry point입니다.
+#
+#   - _coerce_structured_request(value)
+#     LangChain structured output 결과가 이미 StructuredRequest이면 그대로 쓰고, dict이면 Pydantic 검증을 거쳐
+#     StructuredRequest로 바꿉니다. 예상한 형태가 아니면 오류를 내서 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   - extract_structured_request(text)
+#     agent loop를 새로 만들지 않고 chat_model().with_structured_output(...)만 사용해 자연어 또는 JSON 문자열을
+#     StructuredRequest로 검증/구조화합니다. Week 3 이상에서 저장/조율 직전 입력을 구조화해야 할 때 재사용하는 bridge 함수입니다.
+#
+#   - extract_schedule_request(query)
+#     Week 3 이상 agent가 저장/조율 전에 호출하는 LangChain bridge tool입니다.
+#     extract_structured_request(...) 결과에 ok/tool_name/base_date를 붙여 JSON 문자열로 반환하므로,
+#     이후 저장 tool이 structured_request 필드를 그대로 받을 수 있습니다.
+
+
+class StructuredRequest(BaseModel):
+    """LLM structured output으로 추출되는 2주차 요청 스키마입니다."""
+    
+    kind: RequestKind = Field(description="요청 종류. personal_schedule, group_schedule, todo, reminder, unknown 중 하나로 분류한다.")
+    title: str | None = Field(default=None, description="일정이나 할 일의 제목. 모르면 None.")
+    date: str | None = Field(default=None, description="YYYY-MM-DD 형식 날짜. 확실할 때만 채우고 모르면 None.")
+    start_time: str | None = Field(default=None, description="HH:MM 형식 시작 시각. 모르면 None.")
+    end_time: str | None = Field(default=None, description="HH:MM 형식 종료 시각. 모르면 None.")
+    members: list[str] = Field(default_factory=list, description="참석자나 관련 멤버 목록. 모르면 빈 리스트.")
+    priority: str | None = Field(default=None, description="할 일의 우선순위. 없으면 None.")
+    reason: str | None = Field(default=None, description="이 분류를 내린 근거. 없으면 None.")
+    original_text: str = Field(default="", description="사용자의 원문 요청을 그대로 보존.")
+
+
+class StructuredRequestBatch(BaseModel):
+    """여러 자연어 의도를 StructuredRequest 목록으로 나누는 메인과제 스키마입니다."""
+
+    requests: list[StructuredRequest] = Field(default_factory=list, description="자연어 요청을 구조화한 StructuredRequest 목록. 요청이 하나여도 리스트에 담기.")
+    base_date: str = Field(default_factory=current_app_date_iso, description="상대 날짜(내일, 다음 주 등)를 해석하는 기준일. 앱 시작 시 OS에서 읽은 오늘 날짜.")
+
+
+def _coerce_structured_request(value: Any) -> StructuredRequest:
+    """LangChain structured output 결과를 StructuredRequest로 정규화합니다."""
+
+    if isinstance(value, StructuredRequest):
+        return value
+    if isinstance(value, dict):
+        return StructuredRequest.model_validate(value)
+    raise RuntimeError(f"예상하지 못한 structured output 형태입니다: {type(value)!r}")
+
+
+def extract_structured_request(text: str) -> StructuredRequest:
+    """Week 3 이상에서 agent를 새로 띄우지 않고 자연어를 StructuredRequest로 바꿉니다."""
+
+    structured_llm = chat_model().with_structured_output(StructuredRequest, method="function_calling")
+    result = structured_llm.invoke(
+        [
+            ("system", join_system_prompt(week02_prompt_parts())),
+            ("user", text),
+        ]
+    )
+    return _coerce_structured_request(result)
+
+
+@tool
+def extract_schedule_request(query: str) -> str:
+    """Week 3 이상 agent가 저장/조율 전에 호출하는 구조화 bridge tool입니다."""
+
+    structured = extract_structured_request(query)
+    payload = {
+        "ok": True,
+        "tool_name": "extract_schedule_request",
+        "base_date": current_app_date_iso(),
+        "structured_request": structured.model_dump(),
+    }
+    return json.dumps(payload, ensure_ascii=False)
+
+
+def week02_tools() -> list[Any]:
+    """Week 2 agent에 Week 1 도구를 노출해 tool JSON을 structured_response 근거로 씁니다."""
+
+    return week01_tools()
+
+
+def week02_system_prompt() -> str:
+    """조각 리스트를 하나로 합쳐 만든 최종 system prompt 문자열입니다. prompt_parts를 join_system_prompt()로 이어붙이고 최종 답변 규칙을 덧붙입니다."""
+
+    return join_system_prompt(
+        [
+            *week02_prompt_parts(),
+            (
+                "최종 답변은 반드시 StructuredRequestBatch 형식으로만 반환한다. "
+                "요청이 하나뿐이어도 requests 목록에 StructuredRequest 하나를 담고, "
+                "여러 의도가 섞이면 여러 StructuredRequest로 나눈다. "
+                "personal_create_schedule tool을 호출한 경우 그 결과 JSON의 created_schedule을 읽어 "
+                "title/date/start_time/end_time/members 필드를 채운다."
````

personal_create_schedule tool 결과를 읽어 StructuredRequest 필드를 채우도록 연결한 점 좋습니다! 👍

다만 Week1 tool 결과에서는 참석자 필드명이 `attendees`이고, Week2 스키마에서는 `members`라서 이 매핑이 에이전트에게는 암묵적일 수 있을 것 같아요.
`created_schedule.attendees는 StructuredRequest.members로 옮긴다`처럼 명시하면 tool result와 structured_response를 비교할 때 더 안정적일 것 같습니다!

### 코드 리뷰 댓글: @Gongmeda

시각: 2026-07-09T15:58:31Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#discussion_r3553036008
파일: `student_parts/week02_structure_natural_language_requests.py` / 줄: 208
커밋: `c456097a242a3fe4a24d2c1732449b666453ca5b`

````diff
@@ -0,0 +1,266 @@
+from __future__ import annotations
+
+import json
+from typing import Any, Literal
+
+from langchain.agents import create_agent
+from langchain.tools import tool
+from pydantic import BaseModel, Field
+
+from fixed.config import CONFIG
+from fixed.llm import chat_model
+from fixed.runtime_clock import current_app_date_iso
+from student_parts.week01_wake_up_nana import join_system_prompt, week01_prompt_parts, week01_tools
+
+
+RequestKind = Literal["personal_schedule", "group_schedule", "todo", "reminder", "unknown"]
+_WEEK02_AGENT: Any | None = None
+
+
+# [2주차 수강생 구현 가이드]
+#
+# 목표
+#   Week 2의 핵심은 사용자의 한국어 자연어 요청이나 Week 1 tool이 만든 JSON payload를
+#   일정 앱이 읽을 수 있는 StructuredRequest/StructuredRequestBatch로 바꾸는 것입니다.
+#   Week 1이 이미 정해진 인자를 받아 임시 일정을 만들었다면, Week 2는 "내일 오후 3시" 같은
+#   자연어와 created_schedule JSON을 날짜/시간/종류/멤버 필드로 구조화합니다.
+#   구조화 결과는 아직 SQLite, RAG, 외부 멤버 일정 조율 흐름에 저장하지 않습니다.
+#
+# 과제 구성
+#   - 메인과제: Week 2 agent가 자연어 또는 Week 1 tool JSON을 StructuredRequestBatch로
+#     최종 반환하는 세로 슬라이스를 완성합니다.
+#   - 추가 과제: 메인과제에서 만든 StructuredRequest 스키마를 Week 3 이상 저장/조율 흐름에서
+#     재사용할 수 있도록 bridge 함수를 완성합니다.
+#
+# 구현 위치와 사용할 코드
+#   - 이 파일(student_parts/week02_structure_natural_language_requests.py)의
+#     StructuredRequest, StructuredRequestBatch, week02_tools(), week02_prompt_parts(),
+#     week02_system_prompt(), build_week02_agent()를 확인합니다.
+#   - build_week02_agent()는 langchain.agents.create_agent, fixed/llm.py의 chat_model(),
+#     week02_system_prompt(), response_format=StructuredRequestBatch를 사용해 Week 2 agent를 만듭니다.
+#   - week02_tools()는 Week 1 도구 목록을 그대로 가져옵니다. Week 2 agent는 개인 일정 생성 요청에서
+#     personal_create_schedule이 반환한 created_schedule JSON payload를 읽고
+#     response_format=StructuredRequestBatch로 최종 구조화 결과를 확인합니다.
+#   - week02_prompt_parts()는 student_parts/week01_wake_up_nana.py의 week01_prompt_parts() 위에
+#     Week 2 구조화 지시를 추가합니다.
+#   - _coerce_structured_request(), extract_structured_request(), extract_schedule_request()는
+#     Week 3 이상에서 재사용되는 구조화 bridge입니다. Week 2 파일에 있지만 Week 2 agent에
+#     공개되는 tool은 아닙니다.
+#
+# 메인과제 구현 대상
+#   1. StructuredRequest 스키마
+#      - kind/title/date/start_time/end_time/members/priority/reason/original_text 필드가
+#        이후 Week 3 저장 payload의 기준이 됩니다.
+#      - kind는 RequestKind Literal에 들어 있는 값만 허용합니다.
+#      - 각 필드에는 LLM structured output이 이해할 수 있도록 한국어 description을 붙입니다.
+#
+#   2. StructuredRequestBatch 스키마
+#      - requests에는 StructuredRequest 목록을 담고, 요청이 하나뿐이어도 list 형태를 유지합니다.
+#      - base_date에는 상대 날짜 해석 기준일(current_app_date_iso)을 담습니다.
+#
+#   3. Week 2 agent 세로 슬라이스
+#      - week02_tools()는 Week 1 tool 목록을 그대로 반환합니다.
+#      - week02_prompt_parts()와 week02_system_prompt()에는 자연어/Week 1 tool JSON을
+#        StructuredRequestBatch로 구조화하라는 지시를 넣습니다.
+#      - build_week02_agent()에 response_format=StructuredRequestBatch를 연결해
+#        ./run.sh --week2가 동작하게 합니다.
+#      - 개인 일정 생성 요청에서는 Week 1 personal_create_schedule tool 결과의 created_schedule JSON을
+#        LLM이 읽어 StructuredRequestBatch로 최종 변환하는 흐름을 확인합니다.
+#
+# 추가 과제 구현 대상
+#   1. _coerce_structured_request
+#      - LangChain structured output 결과가 이미 StructuredRequest이면 그대로 반환합니다.
+#      - dict이면 StructuredRequest.model_validate(...)로 검증해 반환합니다.
+#      - 예상한 형태가 아니면 RuntimeError를 발생시켜 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   2. extract_structured_request
+#      - chat_model().with_structured_output(StructuredRequest, method="function_calling")를 사용합니다.
+#      - system 메시지에는 join_system_prompt(week02_prompt_parts())를 넣고,
+#        user 메시지에는 text를 넣어 structured LLM을 호출합니다.
+#      - 자연어 또는 JSON 문자열을 StructuredRequest 하나로 검증/구조화합니다.
+#
+#   3. extract_schedule_request
+#      - extract_structured_request(query) 결과에 ok/tool_name/base_date를 붙입니다.
+#      - structured_request에는 model_dump() 결과를 넣고, json.dumps(..., ensure_ascii=False)로 반환합니다.
+#      - Week 3 이상 저장 tool이 structured_request 필드를 그대로 받을 수 있게 만듭니다.
+#
+# StructuredRequest 읽는 법
+#   - kind: personal_schedule, group_schedule, todo, reminder, unknown 중 하나입니다.
+#   - title/date/start_time/end_time: 일정 앱이 실제 저장이나 생성에 사용할 핵심 필드입니다.
+#   - members: 참석자/관련 멤버 list입니다. 모르면 빈 list로 둡니다.
+#   - priority/reason/original_text: 할 일 우선순위, 판단 근거, 원문 보존용 필드입니다.
+#   - 모르는 값을 억지로 만들지 않는 것이 중요합니다. 확실하지 않으면 None 또는 빈 list가 안전합니다.
+#   - date/start_time/end_time은 확실할 때만 YYYY-MM-DD, HH:MM 형식으로 채웁니다.
+#
+# bridge 동작 기준
+#   - 요청이 하나뿐이어도 Week 2 agent의 structured_response에는 StructuredRequest 하나를 담습니다.
+#   - 여러 일정/할 일/알림 의도가 한 문장에 섞이면 Week 2 agent에서는 여러 StructuredRequest로 나눕니다.
+#   - extract_structured_request()는 bridge 용도라 StructuredRequest 하나만 반환합니다.
+#   - Week 1 personal_create_schedule은 이미 분해된 인자로 임시 일정을 생성하고,
+#     Week 2 agent와 bridge는 그 JSON payload를 읽어 저장 가능한 구조로 최종 변환한다는 차이를 비교합니다.
+#
+# 참고 코드
+#   - week01_prompt_parts()
+#      Week 1 system prompt를 이어받아 Week 2 구조화 지시를 누적할 때 사용합니다.
+#   - week01_tools()
+#      Week 1 개인 일정 tool 목록입니다. Week 2 agent는 이 tool 결과 JSON을 구조화 근거로 씁니다.
+#   - extract_structured_request / extract_schedule_request
+#      Week 3 이상에서 DB 저장/조율 tool chain에 쓰는 bridge 코드입니다.
+#      query 문자열이 자연어든 Week 1 tool JSON이든, Python rule/parser로 매핑하지 않고
+#      structured LLM 호출로 구조화한 뒤 JSON tool payload로 감쌉니다.
+#
+# 검증 방법
+#   - 메인과제: ./run.sh --week2로 실행한 뒤 "다음 주 화요일 오후 3시에 철수랑 회의 잡아줘" 같은
+#     문장을 입력합니다. 최종 답변이 StructuredRequestBatch class 형식의 structured_response로
+#     나오는지 확인합니다.
+#   - 추가 과제: Week 3을 실행한 뒤 trace에서 extract_schedule_request 이후
+#     save_structured_request가 호출되는지 봅니다. extract_schedule_request의 반환 JSON에
+#     ok/tool_name/base_date/structured_request가 들어 있는지 확인합니다.
+#
+# 함수별 동작 설명
+#   - StructuredRequest
+#     Week 2 structured output의 중심 스키마입니다. LLM이 자연어에서 뽑은 요청 종류, 제목, 날짜, 시간,
+#     멤버, 우선순위, 근거, 원문을 이 class 필드에 맞춰 반환합니다.
+#
+#   - StructuredRequestBatch
+#     StructuredRequest 여러 개와 base_date를 함께 담는 최종 structured_response 스키마입니다.
+#     요청이 하나뿐이어도 requests list 안에 StructuredRequest 하나를 담습니다.
+#
+#   - week02_tools()
+#     Week 1 개인 일정 tool을 그대로 노출합니다. Week 2 agent는 개인 일정 생성 요청에서
+#     created_schedule JSON을 structured_response의 근거로 사용할 수 있습니다.
+#
+#   - week02_system_prompt() / week02_prompt_parts()
+#     Week 1 prompt 위에 "자연어를 StructuredRequestBatch로 출력한다"는 Week 2 지시를 누적합니다.
+#
+#   - build_week02_agent() / build_week_agent()
+#     response_format=StructuredRequestBatch가 설정된 agent를 만들고 재사용합니다.
+#     build_week_agent()는 실행기가 찾는 표준 entry point입니다.
+#
+#   - _coerce_structured_request(value)
+#     LangChain structured output 결과가 이미 StructuredRequest이면 그대로 쓰고, dict이면 Pydantic 검증을 거쳐
+#     StructuredRequest로 바꿉니다. 예상한 형태가 아니면 오류를 내서 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   - extract_structured_request(text)
+#     agent loop를 새로 만들지 않고 chat_model().with_structured_output(...)만 사용해 자연어 또는 JSON 문자열을
+#     StructuredRequest로 검증/구조화합니다. Week 3 이상에서 저장/조율 직전 입력을 구조화해야 할 때 재사용하는 bridge 함수입니다.
+#
+#   - extract_schedule_request(query)
+#     Week 3 이상 agent가 저장/조율 전에 호출하는 LangChain bridge tool입니다.
+#     extract_structured_request(...) 결과에 ok/tool_name/base_date를 붙여 JSON 문자열로 반환하므로,
+#     이후 저장 tool이 structured_request 필드를 그대로 받을 수 있습니다.
+
+
+class StructuredRequest(BaseModel):
+    """LLM structured output으로 추출되는 2주차 요청 스키마입니다."""
+    
+    kind: RequestKind = Field(description="요청 종류. personal_schedule, group_schedule, todo, reminder, unknown 중 하나로 분류한다.")
+    title: str | None = Field(default=None, description="일정이나 할 일의 제목. 모르면 None.")
+    date: str | None = Field(default=None, description="YYYY-MM-DD 형식 날짜. 확실할 때만 채우고 모르면 None.")
+    start_time: str | None = Field(default=None, description="HH:MM 형식 시작 시각. 모르면 None.")
+    end_time: str | None = Field(default=None, description="HH:MM 형식 종료 시각. 모르면 None.")
+    members: list[str] = Field(default_factory=list, description="참석자나 관련 멤버 목록. 모르면 빈 리스트.")
+    priority: str | None = Field(default=None, description="할 일의 우선순위. 없으면 None.")
+    reason: str | None = Field(default=None, description="이 분류를 내린 근거. 없으면 None.")
+    original_text: str = Field(default="", description="사용자의 원문 요청을 그대로 보존.")
+
+
+class StructuredRequestBatch(BaseModel):
+    """여러 자연어 의도를 StructuredRequest 목록으로 나누는 메인과제 스키마입니다."""
+
+    requests: list[StructuredRequest] = Field(default_factory=list, description="자연어 요청을 구조화한 StructuredRequest 목록. 요청이 하나여도 리스트에 담기.")
+    base_date: str = Field(default_factory=current_app_date_iso, description="상대 날짜(내일, 다음 주 등)를 해석하는 기준일. 앱 시작 시 OS에서 읽은 오늘 날짜.")
+
+
+def _coerce_structured_request(value: Any) -> StructuredRequest:
+    """LangChain structured output 결과를 StructuredRequest로 정규화합니다."""
+
+    if isinstance(value, StructuredRequest):
+        return value
+    if isinstance(value, dict):
+        return StructuredRequest.model_validate(value)
+    raise RuntimeError(f"예상하지 못한 structured output 형태입니다: {type(value)!r}")
+
+
+def extract_structured_request(text: str) -> StructuredRequest:
+    """Week 3 이상에서 agent를 새로 띄우지 않고 자연어를 StructuredRequest로 바꿉니다."""
+
+    structured_llm = chat_model().with_structured_output(StructuredRequest, method="function_calling")
+    result = structured_llm.invoke(
+        [
+            ("system", join_system_prompt(week02_prompt_parts())),
+            ("user", text),
+        ]
+    )
+    return _coerce_structured_request(result)
+
+
+@tool
+def extract_schedule_request(query: str) -> str:
````

여기는 심화 과제 부분이었는데 PR 설명에서는 아직 진행하지 않으셨다고 멘션이 되어 있어요!
아직 미완성된 구현이라고 보면 되는걸까요?

### 리뷰: @Gongmeda / CHANGES_REQUESTED

시각: 2026-07-09T16:02:32Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#pullrequestreview-4664603976

안녕하세요~
이번 주차 과제 진행하시느라 수고 많으셨습니다.

PR 본문에서 structured output과 tool calling의 차이, 그리고 prompt_parts/system_prompt 역할 분리를 직접 정리해보신 점이 좋네요!

라인별로 몇 가지 코멘트 남겼으니 확인 부탁드립니다~

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-07-10T09:03:39Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#discussion_r3557684640
파일: `student_parts/week02_structure_natural_language_requests.py` / 줄: 157
커밋: `b09560a9cb88ad0500be3954edc569b15a51d575`

````diff
@@ -0,0 +1,266 @@
+from __future__ import annotations
+
+import json
+from typing import Any, Literal
+
+from langchain.agents import create_agent
+from langchain.tools import tool
+from pydantic import BaseModel, Field
+
+from fixed.config import CONFIG
+from fixed.llm import chat_model
+from fixed.runtime_clock import current_app_date_iso
+from student_parts.week01_wake_up_nana import join_system_prompt, week01_prompt_parts, week01_tools
+
+
+RequestKind = Literal["personal_schedule", "group_schedule", "todo", "reminder", "unknown"]
+_WEEK02_AGENT: Any | None = None
+
+
+# [2주차 수강생 구현 가이드]
+#
+# 목표
+#   Week 2의 핵심은 사용자의 한국어 자연어 요청이나 Week 1 tool이 만든 JSON payload를
+#   일정 앱이 읽을 수 있는 StructuredRequest/StructuredRequestBatch로 바꾸는 것입니다.
+#   Week 1이 이미 정해진 인자를 받아 임시 일정을 만들었다면, Week 2는 "내일 오후 3시" 같은
+#   자연어와 created_schedule JSON을 날짜/시간/종류/멤버 필드로 구조화합니다.
+#   구조화 결과는 아직 SQLite, RAG, 외부 멤버 일정 조율 흐름에 저장하지 않습니다.
+#
+# 과제 구성
+#   - 메인과제: Week 2 agent가 자연어 또는 Week 1 tool JSON을 StructuredRequestBatch로
+#     최종 반환하는 세로 슬라이스를 완성합니다.
+#   - 추가 과제: 메인과제에서 만든 StructuredRequest 스키마를 Week 3 이상 저장/조율 흐름에서
+#     재사용할 수 있도록 bridge 함수를 완성합니다.
+#
+# 구현 위치와 사용할 코드
+#   - 이 파일(student_parts/week02_structure_natural_language_requests.py)의
+#     StructuredRequest, StructuredRequestBatch, week02_tools(), week02_prompt_parts(),
+#     week02_system_prompt(), build_week02_agent()를 확인합니다.
+#   - build_week02_agent()는 langchain.agents.create_agent, fixed/llm.py의 chat_model(),
+#     week02_system_prompt(), response_format=StructuredRequestBatch를 사용해 Week 2 agent를 만듭니다.
+#   - week02_tools()는 Week 1 도구 목록을 그대로 가져옵니다. Week 2 agent는 개인 일정 생성 요청에서
+#     personal_create_schedule이 반환한 created_schedule JSON payload를 읽고
+#     response_format=StructuredRequestBatch로 최종 구조화 결과를 확인합니다.
+#   - week02_prompt_parts()는 student_parts/week01_wake_up_nana.py의 week01_prompt_parts() 위에
+#     Week 2 구조화 지시를 추가합니다.
+#   - _coerce_structured_request(), extract_structured_request(), extract_schedule_request()는
+#     Week 3 이상에서 재사용되는 구조화 bridge입니다. Week 2 파일에 있지만 Week 2 agent에
+#     공개되는 tool은 아닙니다.
+#
+# 메인과제 구현 대상
+#   1. StructuredRequest 스키마
+#      - kind/title/date/start_time/end_time/members/priority/reason/original_text 필드가
+#        이후 Week 3 저장 payload의 기준이 됩니다.
+#      - kind는 RequestKind Literal에 들어 있는 값만 허용합니다.
+#      - 각 필드에는 LLM structured output이 이해할 수 있도록 한국어 description을 붙입니다.
+#
+#   2. StructuredRequestBatch 스키마
+#      - requests에는 StructuredRequest 목록을 담고, 요청이 하나뿐이어도 list 형태를 유지합니다.
+#      - base_date에는 상대 날짜 해석 기준일(current_app_date_iso)을 담습니다.
+#
+#   3. Week 2 agent 세로 슬라이스
+#      - week02_tools()는 Week 1 tool 목록을 그대로 반환합니다.
+#      - week02_prompt_parts()와 week02_system_prompt()에는 자연어/Week 1 tool JSON을
+#        StructuredRequestBatch로 구조화하라는 지시를 넣습니다.
+#      - build_week02_agent()에 response_format=StructuredRequestBatch를 연결해
+#        ./run.sh --week2가 동작하게 합니다.
+#      - 개인 일정 생성 요청에서는 Week 1 personal_create_schedule tool 결과의 created_schedule JSON을
+#        LLM이 읽어 StructuredRequestBatch로 최종 변환하는 흐름을 확인합니다.
+#
+# 추가 과제 구현 대상
+#   1. _coerce_structured_request
+#      - LangChain structured output 결과가 이미 StructuredRequest이면 그대로 반환합니다.
+#      - dict이면 StructuredRequest.model_validate(...)로 검증해 반환합니다.
+#      - 예상한 형태가 아니면 RuntimeError를 발생시켜 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   2. extract_structured_request
+#      - chat_model().with_structured_output(StructuredRequest, method="function_calling")를 사용합니다.
+#      - system 메시지에는 join_system_prompt(week02_prompt_parts())를 넣고,
+#        user 메시지에는 text를 넣어 structured LLM을 호출합니다.
+#      - 자연어 또는 JSON 문자열을 StructuredRequest 하나로 검증/구조화합니다.
+#
+#   3. extract_schedule_request
+#      - extract_structured_request(query) 결과에 ok/tool_name/base_date를 붙입니다.
+#      - structured_request에는 model_dump() 결과를 넣고, json.dumps(..., ensure_ascii=False)로 반환합니다.
+#      - Week 3 이상 저장 tool이 structured_request 필드를 그대로 받을 수 있게 만듭니다.
+#
+# StructuredRequest 읽는 법
+#   - kind: personal_schedule, group_schedule, todo, reminder, unknown 중 하나입니다.
+#   - title/date/start_time/end_time: 일정 앱이 실제 저장이나 생성에 사용할 핵심 필드입니다.
+#   - members: 참석자/관련 멤버 list입니다. 모르면 빈 list로 둡니다.
+#   - priority/reason/original_text: 할 일 우선순위, 판단 근거, 원문 보존용 필드입니다.
+#   - 모르는 값을 억지로 만들지 않는 것이 중요합니다. 확실하지 않으면 None 또는 빈 list가 안전합니다.
+#   - date/start_time/end_time은 확실할 때만 YYYY-MM-DD, HH:MM 형식으로 채웁니다.
+#
+# bridge 동작 기준
+#   - 요청이 하나뿐이어도 Week 2 agent의 structured_response에는 StructuredRequest 하나를 담습니다.
+#   - 여러 일정/할 일/알림 의도가 한 문장에 섞이면 Week 2 agent에서는 여러 StructuredRequest로 나눕니다.
+#   - extract_structured_request()는 bridge 용도라 StructuredRequest 하나만 반환합니다.
+#   - Week 1 personal_create_schedule은 이미 분해된 인자로 임시 일정을 생성하고,
+#     Week 2 agent와 bridge는 그 JSON payload를 읽어 저장 가능한 구조로 최종 변환한다는 차이를 비교합니다.
+#
+# 참고 코드
+#   - week01_prompt_parts()
+#      Week 1 system prompt를 이어받아 Week 2 구조화 지시를 누적할 때 사용합니다.
+#   - week01_tools()
+#      Week 1 개인 일정 tool 목록입니다. Week 2 agent는 이 tool 결과 JSON을 구조화 근거로 씁니다.
+#   - extract_structured_request / extract_schedule_request
+#      Week 3 이상에서 DB 저장/조율 tool chain에 쓰는 bridge 코드입니다.
+#      query 문자열이 자연어든 Week 1 tool JSON이든, Python rule/parser로 매핑하지 않고
+#      structured LLM 호출로 구조화한 뒤 JSON tool payload로 감쌉니다.
+#
+# 검증 방법
+#   - 메인과제: ./run.sh --week2로 실행한 뒤 "다음 주 화요일 오후 3시에 철수랑 회의 잡아줘" 같은
+#     문장을 입력합니다. 최종 답변이 StructuredRequestBatch class 형식의 structured_response로
+#     나오는지 확인합니다.
+#   - 추가 과제: Week 3을 실행한 뒤 trace에서 extract_schedule_request 이후
+#     save_structured_request가 호출되는지 봅니다. extract_schedule_request의 반환 JSON에
+#     ok/tool_name/base_date/structured_request가 들어 있는지 확인합니다.
+#
+# 함수별 동작 설명
+#   - StructuredRequest
+#     Week 2 structured output의 중심 스키마입니다. LLM이 자연어에서 뽑은 요청 종류, 제목, 날짜, 시간,
+#     멤버, 우선순위, 근거, 원문을 이 class 필드에 맞춰 반환합니다.
+#
+#   - StructuredRequestBatch
+#     StructuredRequest 여러 개와 base_date를 함께 담는 최종 structured_response 스키마입니다.
+#     요청이 하나뿐이어도 requests list 안에 StructuredRequest 하나를 담습니다.
+#
+#   - week02_tools()
+#     Week 1 개인 일정 tool을 그대로 노출합니다. Week 2 agent는 개인 일정 생성 요청에서
+#     created_schedule JSON을 structured_response의 근거로 사용할 수 있습니다.
+#
+#   - week02_system_prompt() / week02_prompt_parts()
+#     Week 1 prompt 위에 "자연어를 StructuredRequestBatch로 출력한다"는 Week 2 지시를 누적합니다.
+#
+#   - build_week02_agent() / build_week_agent()
+#     response_format=StructuredRequestBatch가 설정된 agent를 만들고 재사용합니다.
+#     build_week_agent()는 실행기가 찾는 표준 entry point입니다.
+#
+#   - _coerce_structured_request(value)
+#     LangChain structured output 결과가 이미 StructuredRequest이면 그대로 쓰고, dict이면 Pydantic 검증을 거쳐
+#     StructuredRequest로 바꿉니다. 예상한 형태가 아니면 오류를 내서 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   - extract_structured_request(text)
+#     agent loop를 새로 만들지 않고 chat_model().with_structured_output(...)만 사용해 자연어 또는 JSON 문자열을
+#     StructuredRequest로 검증/구조화합니다. Week 3 이상에서 저장/조율 직전 입력을 구조화해야 할 때 재사용하는 bridge 함수입니다.
+#
+#   - extract_schedule_request(query)
+#     Week 3 이상 agent가 저장/조율 전에 호출하는 LangChain bridge tool입니다.
+#     extract_structured_request(...) 결과에 ok/tool_name/base_date를 붙여 JSON 문자열로 반환하므로,
+#     이후 저장 tool이 structured_request 필드를 그대로 받을 수 있습니다.
+
+
+class StructuredRequest(BaseModel):
+    """LLM structured output으로 추출되는 2주차 요청 스키마입니다."""
+    
+    kind: RequestKind = Field(description="요청 종류. personal_schedule, group_schedule, todo, reminder, unknown 중 하나로 분류한다.")
````

별 생각 없이 간과한 부분인데 꼼꼼하게 지적해주셔서 감사합니다! personal과 group을 분류하는 기준이 현재 과제에는 안내가 없는 것 같아, AI를 활용해 코드를 뒤져봤습니다. `fixed/app_store.py`의 `save_structured_request` docstring에서 개인 일정은 "나" busy-time으로, 그룹 일정은 참석자별 busy-time으로 저장한다는 걸 확인했고, 이를 근거로 **personal = 나 혼자의 일정 / group = 참석자 각자의 일정에도 반영되는 일정**으로 기준을 정리해 `kind` 필드 description에 추가했습니다. 혹시 의도한 분류 기준이 따로 있다면 맞추겠습니다!

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-07-10T09:03:39Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#pullrequestreview-4670047452

(본문 없음)

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-07-10T09:04:42Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#discussion_r3557690525
파일: `student_parts/week02_structure_natural_language_requests.py` / 줄: 208
커밋: `c456097a242a3fe4a24d2c1732449b666453ca5b`

````diff
@@ -0,0 +1,266 @@
+from __future__ import annotations
+
+import json
+from typing import Any, Literal
+
+from langchain.agents import create_agent
+from langchain.tools import tool
+from pydantic import BaseModel, Field
+
+from fixed.config import CONFIG
+from fixed.llm import chat_model
+from fixed.runtime_clock import current_app_date_iso
+from student_parts.week01_wake_up_nana import join_system_prompt, week01_prompt_parts, week01_tools
+
+
+RequestKind = Literal["personal_schedule", "group_schedule", "todo", "reminder", "unknown"]
+_WEEK02_AGENT: Any | None = None
+
+
+# [2주차 수강생 구현 가이드]
+#
+# 목표
+#   Week 2의 핵심은 사용자의 한국어 자연어 요청이나 Week 1 tool이 만든 JSON payload를
+#   일정 앱이 읽을 수 있는 StructuredRequest/StructuredRequestBatch로 바꾸는 것입니다.
+#   Week 1이 이미 정해진 인자를 받아 임시 일정을 만들었다면, Week 2는 "내일 오후 3시" 같은
+#   자연어와 created_schedule JSON을 날짜/시간/종류/멤버 필드로 구조화합니다.
+#   구조화 결과는 아직 SQLite, RAG, 외부 멤버 일정 조율 흐름에 저장하지 않습니다.
+#
+# 과제 구성
+#   - 메인과제: Week 2 agent가 자연어 또는 Week 1 tool JSON을 StructuredRequestBatch로
+#     최종 반환하는 세로 슬라이스를 완성합니다.
+#   - 추가 과제: 메인과제에서 만든 StructuredRequest 스키마를 Week 3 이상 저장/조율 흐름에서
+#     재사용할 수 있도록 bridge 함수를 완성합니다.
+#
+# 구현 위치와 사용할 코드
+#   - 이 파일(student_parts/week02_structure_natural_language_requests.py)의
+#     StructuredRequest, StructuredRequestBatch, week02_tools(), week02_prompt_parts(),
+#     week02_system_prompt(), build_week02_agent()를 확인합니다.
+#   - build_week02_agent()는 langchain.agents.create_agent, fixed/llm.py의 chat_model(),
+#     week02_system_prompt(), response_format=StructuredRequestBatch를 사용해 Week 2 agent를 만듭니다.
+#   - week02_tools()는 Week 1 도구 목록을 그대로 가져옵니다. Week 2 agent는 개인 일정 생성 요청에서
+#     personal_create_schedule이 반환한 created_schedule JSON payload를 읽고
+#     response_format=StructuredRequestBatch로 최종 구조화 결과를 확인합니다.
+#   - week02_prompt_parts()는 student_parts/week01_wake_up_nana.py의 week01_prompt_parts() 위에
+#     Week 2 구조화 지시를 추가합니다.
+#   - _coerce_structured_request(), extract_structured_request(), extract_schedule_request()는
+#     Week 3 이상에서 재사용되는 구조화 bridge입니다. Week 2 파일에 있지만 Week 2 agent에
+#     공개되는 tool은 아닙니다.
+#
+# 메인과제 구현 대상
+#   1. StructuredRequest 스키마
+#      - kind/title/date/start_time/end_time/members/priority/reason/original_text 필드가
+#        이후 Week 3 저장 payload의 기준이 됩니다.
+#      - kind는 RequestKind Literal에 들어 있는 값만 허용합니다.
+#      - 각 필드에는 LLM structured output이 이해할 수 있도록 한국어 description을 붙입니다.
+#
+#   2. StructuredRequestBatch 스키마
+#      - requests에는 StructuredRequest 목록을 담고, 요청이 하나뿐이어도 list 형태를 유지합니다.
+#      - base_date에는 상대 날짜 해석 기준일(current_app_date_iso)을 담습니다.
+#
+#   3. Week 2 agent 세로 슬라이스
+#      - week02_tools()는 Week 1 tool 목록을 그대로 반환합니다.
+#      - week02_prompt_parts()와 week02_system_prompt()에는 자연어/Week 1 tool JSON을
+#        StructuredRequestBatch로 구조화하라는 지시를 넣습니다.
+#      - build_week02_agent()에 response_format=StructuredRequestBatch를 연결해
+#        ./run.sh --week2가 동작하게 합니다.
+#      - 개인 일정 생성 요청에서는 Week 1 personal_create_schedule tool 결과의 created_schedule JSON을
+#        LLM이 읽어 StructuredRequestBatch로 최종 변환하는 흐름을 확인합니다.
+#
+# 추가 과제 구현 대상
+#   1. _coerce_structured_request
+#      - LangChain structured output 결과가 이미 StructuredRequest이면 그대로 반환합니다.
+#      - dict이면 StructuredRequest.model_validate(...)로 검증해 반환합니다.
+#      - 예상한 형태가 아니면 RuntimeError를 발생시켜 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   2. extract_structured_request
+#      - chat_model().with_structured_output(StructuredRequest, method="function_calling")를 사용합니다.
+#      - system 메시지에는 join_system_prompt(week02_prompt_parts())를 넣고,
+#        user 메시지에는 text를 넣어 structured LLM을 호출합니다.
+#      - 자연어 또는 JSON 문자열을 StructuredRequest 하나로 검증/구조화합니다.
+#
+#   3. extract_schedule_request
+#      - extract_structured_request(query) 결과에 ok/tool_name/base_date를 붙입니다.
+#      - structured_request에는 model_dump() 결과를 넣고, json.dumps(..., ensure_ascii=False)로 반환합니다.
+#      - Week 3 이상 저장 tool이 structured_request 필드를 그대로 받을 수 있게 만듭니다.
+#
+# StructuredRequest 읽는 법
+#   - kind: personal_schedule, group_schedule, todo, reminder, unknown 중 하나입니다.
+#   - title/date/start_time/end_time: 일정 앱이 실제 저장이나 생성에 사용할 핵심 필드입니다.
+#   - members: 참석자/관련 멤버 list입니다. 모르면 빈 list로 둡니다.
+#   - priority/reason/original_text: 할 일 우선순위, 판단 근거, 원문 보존용 필드입니다.
+#   - 모르는 값을 억지로 만들지 않는 것이 중요합니다. 확실하지 않으면 None 또는 빈 list가 안전합니다.
+#   - date/start_time/end_time은 확실할 때만 YYYY-MM-DD, HH:MM 형식으로 채웁니다.
+#
+# bridge 동작 기준
+#   - 요청이 하나뿐이어도 Week 2 agent의 structured_response에는 StructuredRequest 하나를 담습니다.
+#   - 여러 일정/할 일/알림 의도가 한 문장에 섞이면 Week 2 agent에서는 여러 StructuredRequest로 나눕니다.
+#   - extract_structured_request()는 bridge 용도라 StructuredRequest 하나만 반환합니다.
+#   - Week 1 personal_create_schedule은 이미 분해된 인자로 임시 일정을 생성하고,
+#     Week 2 agent와 bridge는 그 JSON payload를 읽어 저장 가능한 구조로 최종 변환한다는 차이를 비교합니다.
+#
+# 참고 코드
+#   - week01_prompt_parts()
+#      Week 1 system prompt를 이어받아 Week 2 구조화 지시를 누적할 때 사용합니다.
+#   - week01_tools()
+#      Week 1 개인 일정 tool 목록입니다. Week 2 agent는 이 tool 결과 JSON을 구조화 근거로 씁니다.
+#   - extract_structured_request / extract_schedule_request
+#      Week 3 이상에서 DB 저장/조율 tool chain에 쓰는 bridge 코드입니다.
+#      query 문자열이 자연어든 Week 1 tool JSON이든, Python rule/parser로 매핑하지 않고
+#      structured LLM 호출로 구조화한 뒤 JSON tool payload로 감쌉니다.
+#
+# 검증 방법
+#   - 메인과제: ./run.sh --week2로 실행한 뒤 "다음 주 화요일 오후 3시에 철수랑 회의 잡아줘" 같은
+#     문장을 입력합니다. 최종 답변이 StructuredRequestBatch class 형식의 structured_response로
+#     나오는지 확인합니다.
+#   - 추가 과제: Week 3을 실행한 뒤 trace에서 extract_schedule_request 이후
+#     save_structured_request가 호출되는지 봅니다. extract_schedule_request의 반환 JSON에
+#     ok/tool_name/base_date/structured_request가 들어 있는지 확인합니다.
+#
+# 함수별 동작 설명
+#   - StructuredRequest
+#     Week 2 structured output의 중심 스키마입니다. LLM이 자연어에서 뽑은 요청 종류, 제목, 날짜, 시간,
+#     멤버, 우선순위, 근거, 원문을 이 class 필드에 맞춰 반환합니다.
+#
+#   - StructuredRequestBatch
+#     StructuredRequest 여러 개와 base_date를 함께 담는 최종 structured_response 스키마입니다.
+#     요청이 하나뿐이어도 requests list 안에 StructuredRequest 하나를 담습니다.
+#
+#   - week02_tools()
+#     Week 1 개인 일정 tool을 그대로 노출합니다. Week 2 agent는 개인 일정 생성 요청에서
+#     created_schedule JSON을 structured_response의 근거로 사용할 수 있습니다.
+#
+#   - week02_system_prompt() / week02_prompt_parts()
+#     Week 1 prompt 위에 "자연어를 StructuredRequestBatch로 출력한다"는 Week 2 지시를 누적합니다.
+#
+#   - build_week02_agent() / build_week_agent()
+#     response_format=StructuredRequestBatch가 설정된 agent를 만들고 재사용합니다.
+#     build_week_agent()는 실행기가 찾는 표준 entry point입니다.
+#
+#   - _coerce_structured_request(value)
+#     LangChain structured output 결과가 이미 StructuredRequest이면 그대로 쓰고, dict이면 Pydantic 검증을 거쳐
+#     StructuredRequest로 바꿉니다. 예상한 형태가 아니면 오류를 내서 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   - extract_structured_request(text)
+#     agent loop를 새로 만들지 않고 chat_model().with_structured_output(...)만 사용해 자연어 또는 JSON 문자열을
+#     StructuredRequest로 검증/구조화합니다. Week 3 이상에서 저장/조율 직전 입력을 구조화해야 할 때 재사용하는 bridge 함수입니다.
+#
+#   - extract_schedule_request(query)
+#     Week 3 이상 agent가 저장/조율 전에 호출하는 LangChain bridge tool입니다.
+#     extract_structured_request(...) 결과에 ok/tool_name/base_date를 붙여 JSON 문자열로 반환하므로,
+#     이후 저장 tool이 structured_request 필드를 그대로 받을 수 있습니다.
+
+
+class StructuredRequest(BaseModel):
+    """LLM structured output으로 추출되는 2주차 요청 스키마입니다."""
+    
+    kind: RequestKind = Field(description="요청 종류. personal_schedule, group_schedule, todo, reminder, unknown 중 하나로 분류한다.")
+    title: str | None = Field(default=None, description="일정이나 할 일의 제목. 모르면 None.")
+    date: str | None = Field(default=None, description="YYYY-MM-DD 형식 날짜. 확실할 때만 채우고 모르면 None.")
+    start_time: str | None = Field(default=None, description="HH:MM 형식 시작 시각. 모르면 None.")
+    end_time: str | None = Field(default=None, description="HH:MM 형식 종료 시각. 모르면 None.")
+    members: list[str] = Field(default_factory=list, description="참석자나 관련 멤버 목록. 모르면 빈 리스트.")
+    priority: str | None = Field(default=None, description="할 일의 우선순위. 없으면 None.")
+    reason: str | None = Field(default=None, description="이 분류를 내린 근거. 없으면 None.")
+    original_text: str = Field(default="", description="사용자의 원문 요청을 그대로 보존.")
+
+
+class StructuredRequestBatch(BaseModel):
+    """여러 자연어 의도를 StructuredRequest 목록으로 나누는 메인과제 스키마입니다."""
+
+    requests: list[StructuredRequest] = Field(default_factory=list, description="자연어 요청을 구조화한 StructuredRequest 목록. 요청이 하나여도 리스트에 담기.")
+    base_date: str = Field(default_factory=current_app_date_iso, description="상대 날짜(내일, 다음 주 등)를 해석하는 기준일. 앱 시작 시 OS에서 읽은 오늘 날짜.")
+
+
+def _coerce_structured_request(value: Any) -> StructuredRequest:
+    """LangChain structured output 결과를 StructuredRequest로 정규화합니다."""
+
+    if isinstance(value, StructuredRequest):
+        return value
+    if isinstance(value, dict):
+        return StructuredRequest.model_validate(value)
+    raise RuntimeError(f"예상하지 못한 structured output 형태입니다: {type(value)!r}")
+
+
+def extract_structured_request(text: str) -> StructuredRequest:
+    """Week 3 이상에서 agent를 새로 띄우지 않고 자연어를 StructuredRequest로 바꿉니다."""
+
+    structured_llm = chat_model().with_structured_output(StructuredRequest, method="function_calling")
+    result = structured_llm.invoke(
+        [
+            ("system", join_system_prompt(week02_prompt_parts())),
+            ("user", text),
+        ]
+    )
+    return _coerce_structured_request(result)
+
+
+@tool
+def extract_schedule_request(query: str) -> str:
````

심화 과제 공지 관련하여 제가 착각해서 해당 내용도 기본 과제인 걸로 인지하고 있었습니다. 코드는 첫 PR과 변경사항 없고, PR 본문에 심화 완료로 추가했습니다. 확인해주시면 감사하겠습니다!



### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-07-10T09:04:42Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#pullrequestreview-4670053995

(본문 없음)

### 코드 리뷰 댓글: @kmgvv23

시각: 2026-07-10T09:06:42Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#discussion_r3557702056
파일: `student_parts/week02_structure_natural_language_requests.py` / 줄: 238
커밋: `c456097a242a3fe4a24d2c1732449b666453ca5b`

````diff
@@ -0,0 +1,266 @@
+from __future__ import annotations
+
+import json
+from typing import Any, Literal
+
+from langchain.agents import create_agent
+from langchain.tools import tool
+from pydantic import BaseModel, Field
+
+from fixed.config import CONFIG
+from fixed.llm import chat_model
+from fixed.runtime_clock import current_app_date_iso
+from student_parts.week01_wake_up_nana import join_system_prompt, week01_prompt_parts, week01_tools
+
+
+RequestKind = Literal["personal_schedule", "group_schedule", "todo", "reminder", "unknown"]
+_WEEK02_AGENT: Any | None = None
+
+
+# [2주차 수강생 구현 가이드]
+#
+# 목표
+#   Week 2의 핵심은 사용자의 한국어 자연어 요청이나 Week 1 tool이 만든 JSON payload를
+#   일정 앱이 읽을 수 있는 StructuredRequest/StructuredRequestBatch로 바꾸는 것입니다.
+#   Week 1이 이미 정해진 인자를 받아 임시 일정을 만들었다면, Week 2는 "내일 오후 3시" 같은
+#   자연어와 created_schedule JSON을 날짜/시간/종류/멤버 필드로 구조화합니다.
+#   구조화 결과는 아직 SQLite, RAG, 외부 멤버 일정 조율 흐름에 저장하지 않습니다.
+#
+# 과제 구성
+#   - 메인과제: Week 2 agent가 자연어 또는 Week 1 tool JSON을 StructuredRequestBatch로
+#     최종 반환하는 세로 슬라이스를 완성합니다.
+#   - 추가 과제: 메인과제에서 만든 StructuredRequest 스키마를 Week 3 이상 저장/조율 흐름에서
+#     재사용할 수 있도록 bridge 함수를 완성합니다.
+#
+# 구현 위치와 사용할 코드
+#   - 이 파일(student_parts/week02_structure_natural_language_requests.py)의
+#     StructuredRequest, StructuredRequestBatch, week02_tools(), week02_prompt_parts(),
+#     week02_system_prompt(), build_week02_agent()를 확인합니다.
+#   - build_week02_agent()는 langchain.agents.create_agent, fixed/llm.py의 chat_model(),
+#     week02_system_prompt(), response_format=StructuredRequestBatch를 사용해 Week 2 agent를 만듭니다.
+#   - week02_tools()는 Week 1 도구 목록을 그대로 가져옵니다. Week 2 agent는 개인 일정 생성 요청에서
+#     personal_create_schedule이 반환한 created_schedule JSON payload를 읽고
+#     response_format=StructuredRequestBatch로 최종 구조화 결과를 확인합니다.
+#   - week02_prompt_parts()는 student_parts/week01_wake_up_nana.py의 week01_prompt_parts() 위에
+#     Week 2 구조화 지시를 추가합니다.
+#   - _coerce_structured_request(), extract_structured_request(), extract_schedule_request()는
+#     Week 3 이상에서 재사용되는 구조화 bridge입니다. Week 2 파일에 있지만 Week 2 agent에
+#     공개되는 tool은 아닙니다.
+#
+# 메인과제 구현 대상
+#   1. StructuredRequest 스키마
+#      - kind/title/date/start_time/end_time/members/priority/reason/original_text 필드가
+#        이후 Week 3 저장 payload의 기준이 됩니다.
+#      - kind는 RequestKind Literal에 들어 있는 값만 허용합니다.
+#      - 각 필드에는 LLM structured output이 이해할 수 있도록 한국어 description을 붙입니다.
+#
+#   2. StructuredRequestBatch 스키마
+#      - requests에는 StructuredRequest 목록을 담고, 요청이 하나뿐이어도 list 형태를 유지합니다.
+#      - base_date에는 상대 날짜 해석 기준일(current_app_date_iso)을 담습니다.
+#
+#   3. Week 2 agent 세로 슬라이스
+#      - week02_tools()는 Week 1 tool 목록을 그대로 반환합니다.
+#      - week02_prompt_parts()와 week02_system_prompt()에는 자연어/Week 1 tool JSON을
+#        StructuredRequestBatch로 구조화하라는 지시를 넣습니다.
+#      - build_week02_agent()에 response_format=StructuredRequestBatch를 연결해
+#        ./run.sh --week2가 동작하게 합니다.
+#      - 개인 일정 생성 요청에서는 Week 1 personal_create_schedule tool 결과의 created_schedule JSON을
+#        LLM이 읽어 StructuredRequestBatch로 최종 변환하는 흐름을 확인합니다.
+#
+# 추가 과제 구현 대상
+#   1. _coerce_structured_request
+#      - LangChain structured output 결과가 이미 StructuredRequest이면 그대로 반환합니다.
+#      - dict이면 StructuredRequest.model_validate(...)로 검증해 반환합니다.
+#      - 예상한 형태가 아니면 RuntimeError를 발생시켜 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   2. extract_structured_request
+#      - chat_model().with_structured_output(StructuredRequest, method="function_calling")를 사용합니다.
+#      - system 메시지에는 join_system_prompt(week02_prompt_parts())를 넣고,
+#        user 메시지에는 text를 넣어 structured LLM을 호출합니다.
+#      - 자연어 또는 JSON 문자열을 StructuredRequest 하나로 검증/구조화합니다.
+#
+#   3. extract_schedule_request
+#      - extract_structured_request(query) 결과에 ok/tool_name/base_date를 붙입니다.
+#      - structured_request에는 model_dump() 결과를 넣고, json.dumps(..., ensure_ascii=False)로 반환합니다.
+#      - Week 3 이상 저장 tool이 structured_request 필드를 그대로 받을 수 있게 만듭니다.
+#
+# StructuredRequest 읽는 법
+#   - kind: personal_schedule, group_schedule, todo, reminder, unknown 중 하나입니다.
+#   - title/date/start_time/end_time: 일정 앱이 실제 저장이나 생성에 사용할 핵심 필드입니다.
+#   - members: 참석자/관련 멤버 list입니다. 모르면 빈 list로 둡니다.
+#   - priority/reason/original_text: 할 일 우선순위, 판단 근거, 원문 보존용 필드입니다.
+#   - 모르는 값을 억지로 만들지 않는 것이 중요합니다. 확실하지 않으면 None 또는 빈 list가 안전합니다.
+#   - date/start_time/end_time은 확실할 때만 YYYY-MM-DD, HH:MM 형식으로 채웁니다.
+#
+# bridge 동작 기준
+#   - 요청이 하나뿐이어도 Week 2 agent의 structured_response에는 StructuredRequest 하나를 담습니다.
+#   - 여러 일정/할 일/알림 의도가 한 문장에 섞이면 Week 2 agent에서는 여러 StructuredRequest로 나눕니다.
+#   - extract_structured_request()는 bridge 용도라 StructuredRequest 하나만 반환합니다.
+#   - Week 1 personal_create_schedule은 이미 분해된 인자로 임시 일정을 생성하고,
+#     Week 2 agent와 bridge는 그 JSON payload를 읽어 저장 가능한 구조로 최종 변환한다는 차이를 비교합니다.
+#
+# 참고 코드
+#   - week01_prompt_parts()
+#      Week 1 system prompt를 이어받아 Week 2 구조화 지시를 누적할 때 사용합니다.
+#   - week01_tools()
+#      Week 1 개인 일정 tool 목록입니다. Week 2 agent는 이 tool 결과 JSON을 구조화 근거로 씁니다.
+#   - extract_structured_request / extract_schedule_request
+#      Week 3 이상에서 DB 저장/조율 tool chain에 쓰는 bridge 코드입니다.
+#      query 문자열이 자연어든 Week 1 tool JSON이든, Python rule/parser로 매핑하지 않고
+#      structured LLM 호출로 구조화한 뒤 JSON tool payload로 감쌉니다.
+#
+# 검증 방법
+#   - 메인과제: ./run.sh --week2로 실행한 뒤 "다음 주 화요일 오후 3시에 철수랑 회의 잡아줘" 같은
+#     문장을 입력합니다. 최종 답변이 StructuredRequestBatch class 형식의 structured_response로
+#     나오는지 확인합니다.
+#   - 추가 과제: Week 3을 실행한 뒤 trace에서 extract_schedule_request 이후
+#     save_structured_request가 호출되는지 봅니다. extract_schedule_request의 반환 JSON에
+#     ok/tool_name/base_date/structured_request가 들어 있는지 확인합니다.
+#
+# 함수별 동작 설명
+#   - StructuredRequest
+#     Week 2 structured output의 중심 스키마입니다. LLM이 자연어에서 뽑은 요청 종류, 제목, 날짜, 시간,
+#     멤버, 우선순위, 근거, 원문을 이 class 필드에 맞춰 반환합니다.
+#
+#   - StructuredRequestBatch
+#     StructuredRequest 여러 개와 base_date를 함께 담는 최종 structured_response 스키마입니다.
+#     요청이 하나뿐이어도 requests list 안에 StructuredRequest 하나를 담습니다.
+#
+#   - week02_tools()
+#     Week 1 개인 일정 tool을 그대로 노출합니다. Week 2 agent는 개인 일정 생성 요청에서
+#     created_schedule JSON을 structured_response의 근거로 사용할 수 있습니다.
+#
+#   - week02_system_prompt() / week02_prompt_parts()
+#     Week 1 prompt 위에 "자연어를 StructuredRequestBatch로 출력한다"는 Week 2 지시를 누적합니다.
+#
+#   - build_week02_agent() / build_week_agent()
+#     response_format=StructuredRequestBatch가 설정된 agent를 만들고 재사용합니다.
+#     build_week_agent()는 실행기가 찾는 표준 entry point입니다.
+#
+#   - _coerce_structured_request(value)
+#     LangChain structured output 결과가 이미 StructuredRequest이면 그대로 쓰고, dict이면 Pydantic 검증을 거쳐
+#     StructuredRequest로 바꿉니다. 예상한 형태가 아니면 오류를 내서 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   - extract_structured_request(text)
+#     agent loop를 새로 만들지 않고 chat_model().with_structured_output(...)만 사용해 자연어 또는 JSON 문자열을
+#     StructuredRequest로 검증/구조화합니다. Week 3 이상에서 저장/조율 직전 입력을 구조화해야 할 때 재사용하는 bridge 함수입니다.
+#
+#   - extract_schedule_request(query)
+#     Week 3 이상 agent가 저장/조율 전에 호출하는 LangChain bridge tool입니다.
+#     extract_structured_request(...) 결과에 ok/tool_name/base_date를 붙여 JSON 문자열로 반환하므로,
+#     이후 저장 tool이 structured_request 필드를 그대로 받을 수 있습니다.
+
+
+class StructuredRequest(BaseModel):
+    """LLM structured output으로 추출되는 2주차 요청 스키마입니다."""
+    
+    kind: RequestKind = Field(description="요청 종류. personal_schedule, group_schedule, todo, reminder, unknown 중 하나로 분류한다.")
+    title: str | None = Field(default=None, description="일정이나 할 일의 제목. 모르면 None.")
+    date: str | None = Field(default=None, description="YYYY-MM-DD 형식 날짜. 확실할 때만 채우고 모르면 None.")
+    start_time: str | None = Field(default=None, description="HH:MM 형식 시작 시각. 모르면 None.")
+    end_time: str | None = Field(default=None, description="HH:MM 형식 종료 시각. 모르면 None.")
+    members: list[str] = Field(default_factory=list, description="참석자나 관련 멤버 목록. 모르면 빈 리스트.")
+    priority: str | None = Field(default=None, description="할 일의 우선순위. 없으면 None.")
+    reason: str | None = Field(default=None, description="이 분류를 내린 근거. 없으면 None.")
+    original_text: str = Field(default="", description="사용자의 원문 요청을 그대로 보존.")
+
+
+class StructuredRequestBatch(BaseModel):
+    """여러 자연어 의도를 StructuredRequest 목록으로 나누는 메인과제 스키마입니다."""
+
+    requests: list[StructuredRequest] = Field(default_factory=list, description="자연어 요청을 구조화한 StructuredRequest 목록. 요청이 하나여도 리스트에 담기.")
+    base_date: str = Field(default_factory=current_app_date_iso, description="상대 날짜(내일, 다음 주 등)를 해석하는 기준일. 앱 시작 시 OS에서 읽은 오늘 날짜.")
+
+
+def _coerce_structured_request(value: Any) -> StructuredRequest:
+    """LangChain structured output 결과를 StructuredRequest로 정규화합니다."""
+
+    if isinstance(value, StructuredRequest):
+        return value
+    if isinstance(value, dict):
+        return StructuredRequest.model_validate(value)
+    raise RuntimeError(f"예상하지 못한 structured output 형태입니다: {type(value)!r}")
+
+
+def extract_structured_request(text: str) -> StructuredRequest:
+    """Week 3 이상에서 agent를 새로 띄우지 않고 자연어를 StructuredRequest로 바꿉니다."""
+
+    structured_llm = chat_model().with_structured_output(StructuredRequest, method="function_calling")
+    result = structured_llm.invoke(
+        [
+            ("system", join_system_prompt(week02_prompt_parts())),
+            ("user", text),
+        ]
+    )
+    return _coerce_structured_request(result)
+
+
+@tool
+def extract_schedule_request(query: str) -> str:
+    """Week 3 이상 agent가 저장/조율 전에 호출하는 구조화 bridge tool입니다."""
+
+    structured = extract_structured_request(query)
+    payload = {
+        "ok": True,
+        "tool_name": "extract_schedule_request",
+        "base_date": current_app_date_iso(),
+        "structured_request": structured.model_dump(),
+    }
+    return json.dumps(payload, ensure_ascii=False)
+
+
+def week02_tools() -> list[Any]:
+    """Week 2 agent에 Week 1 도구를 노출해 tool JSON을 structured_response 근거로 씁니다."""
+
+    return week01_tools()
+
+
+def week02_system_prompt() -> str:
+    """조각 리스트를 하나로 합쳐 만든 최종 system prompt 문자열입니다. prompt_parts를 join_system_prompt()로 이어붙이고 최종 답변 규칙을 덧붙입니다."""
+
+    return join_system_prompt(
+        [
+            *week02_prompt_parts(),
+            (
+                "최종 답변은 반드시 StructuredRequestBatch 형식으로만 반환한다. "
+                "요청이 하나뿐이어도 requests 목록에 StructuredRequest 하나를 담고, "
+                "여러 의도가 섞이면 여러 StructuredRequest로 나눈다. "
+                "personal_create_schedule tool을 호출한 경우 그 결과 JSON의 created_schedule을 읽어 "
+                "title/date/start_time/end_time/members 필드를 채운다."
````

이것 또한 그저 TODO만 보고 코드를 구현하다 보니 몰랐던 부분인데, 지적해주셔서 감사합니다. 다음부턴 조금 더 세세하게 찾아보겠습니다. week1을 건드리면 안 되는 걸로 알고 있어서, 제안해주신 대로 prompt에 `created_schedule.attendees`를 `members`로 옮기라고 명시해두었습니다.

### 리뷰: @kmgvv23 / COMMENTED

시각: 2026-07-10T09:06:42Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#pullrequestreview-4670067002

(본문 없음)

### 코드 리뷰 댓글: @Gongmeda

시각: 2026-07-12T15:33:43Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#discussion_r3566588922
파일: `student_parts/week02_structure_natural_language_requests.py` / 줄: 157
커밋: `b09560a9cb88ad0500be3954edc569b15a51d575`

````diff
@@ -0,0 +1,266 @@
+from __future__ import annotations
+
+import json
+from typing import Any, Literal
+
+from langchain.agents import create_agent
+from langchain.tools import tool
+from pydantic import BaseModel, Field
+
+from fixed.config import CONFIG
+from fixed.llm import chat_model
+from fixed.runtime_clock import current_app_date_iso
+from student_parts.week01_wake_up_nana import join_system_prompt, week01_prompt_parts, week01_tools
+
+
+RequestKind = Literal["personal_schedule", "group_schedule", "todo", "reminder", "unknown"]
+_WEEK02_AGENT: Any | None = None
+
+
+# [2주차 수강생 구현 가이드]
+#
+# 목표
+#   Week 2의 핵심은 사용자의 한국어 자연어 요청이나 Week 1 tool이 만든 JSON payload를
+#   일정 앱이 읽을 수 있는 StructuredRequest/StructuredRequestBatch로 바꾸는 것입니다.
+#   Week 1이 이미 정해진 인자를 받아 임시 일정을 만들었다면, Week 2는 "내일 오후 3시" 같은
+#   자연어와 created_schedule JSON을 날짜/시간/종류/멤버 필드로 구조화합니다.
+#   구조화 결과는 아직 SQLite, RAG, 외부 멤버 일정 조율 흐름에 저장하지 않습니다.
+#
+# 과제 구성
+#   - 메인과제: Week 2 agent가 자연어 또는 Week 1 tool JSON을 StructuredRequestBatch로
+#     최종 반환하는 세로 슬라이스를 완성합니다.
+#   - 추가 과제: 메인과제에서 만든 StructuredRequest 스키마를 Week 3 이상 저장/조율 흐름에서
+#     재사용할 수 있도록 bridge 함수를 완성합니다.
+#
+# 구현 위치와 사용할 코드
+#   - 이 파일(student_parts/week02_structure_natural_language_requests.py)의
+#     StructuredRequest, StructuredRequestBatch, week02_tools(), week02_prompt_parts(),
+#     week02_system_prompt(), build_week02_agent()를 확인합니다.
+#   - build_week02_agent()는 langchain.agents.create_agent, fixed/llm.py의 chat_model(),
+#     week02_system_prompt(), response_format=StructuredRequestBatch를 사용해 Week 2 agent를 만듭니다.
+#   - week02_tools()는 Week 1 도구 목록을 그대로 가져옵니다. Week 2 agent는 개인 일정 생성 요청에서
+#     personal_create_schedule이 반환한 created_schedule JSON payload를 읽고
+#     response_format=StructuredRequestBatch로 최종 구조화 결과를 확인합니다.
+#   - week02_prompt_parts()는 student_parts/week01_wake_up_nana.py의 week01_prompt_parts() 위에
+#     Week 2 구조화 지시를 추가합니다.
+#   - _coerce_structured_request(), extract_structured_request(), extract_schedule_request()는
+#     Week 3 이상에서 재사용되는 구조화 bridge입니다. Week 2 파일에 있지만 Week 2 agent에
+#     공개되는 tool은 아닙니다.
+#
+# 메인과제 구현 대상
+#   1. StructuredRequest 스키마
+#      - kind/title/date/start_time/end_time/members/priority/reason/original_text 필드가
+#        이후 Week 3 저장 payload의 기준이 됩니다.
+#      - kind는 RequestKind Literal에 들어 있는 값만 허용합니다.
+#      - 각 필드에는 LLM structured output이 이해할 수 있도록 한국어 description을 붙입니다.
+#
+#   2. StructuredRequestBatch 스키마
+#      - requests에는 StructuredRequest 목록을 담고, 요청이 하나뿐이어도 list 형태를 유지합니다.
+#      - base_date에는 상대 날짜 해석 기준일(current_app_date_iso)을 담습니다.
+#
+#   3. Week 2 agent 세로 슬라이스
+#      - week02_tools()는 Week 1 tool 목록을 그대로 반환합니다.
+#      - week02_prompt_parts()와 week02_system_prompt()에는 자연어/Week 1 tool JSON을
+#        StructuredRequestBatch로 구조화하라는 지시를 넣습니다.
+#      - build_week02_agent()에 response_format=StructuredRequestBatch를 연결해
+#        ./run.sh --week2가 동작하게 합니다.
+#      - 개인 일정 생성 요청에서는 Week 1 personal_create_schedule tool 결과의 created_schedule JSON을
+#        LLM이 읽어 StructuredRequestBatch로 최종 변환하는 흐름을 확인합니다.
+#
+# 추가 과제 구현 대상
+#   1. _coerce_structured_request
+#      - LangChain structured output 결과가 이미 StructuredRequest이면 그대로 반환합니다.
+#      - dict이면 StructuredRequest.model_validate(...)로 검증해 반환합니다.
+#      - 예상한 형태가 아니면 RuntimeError를 발생시켜 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   2. extract_structured_request
+#      - chat_model().with_structured_output(StructuredRequest, method="function_calling")를 사용합니다.
+#      - system 메시지에는 join_system_prompt(week02_prompt_parts())를 넣고,
+#        user 메시지에는 text를 넣어 structured LLM을 호출합니다.
+#      - 자연어 또는 JSON 문자열을 StructuredRequest 하나로 검증/구조화합니다.
+#
+#   3. extract_schedule_request
+#      - extract_structured_request(query) 결과에 ok/tool_name/base_date를 붙입니다.
+#      - structured_request에는 model_dump() 결과를 넣고, json.dumps(..., ensure_ascii=False)로 반환합니다.
+#      - Week 3 이상 저장 tool이 structured_request 필드를 그대로 받을 수 있게 만듭니다.
+#
+# StructuredRequest 읽는 법
+#   - kind: personal_schedule, group_schedule, todo, reminder, unknown 중 하나입니다.
+#   - title/date/start_time/end_time: 일정 앱이 실제 저장이나 생성에 사용할 핵심 필드입니다.
+#   - members: 참석자/관련 멤버 list입니다. 모르면 빈 list로 둡니다.
+#   - priority/reason/original_text: 할 일 우선순위, 판단 근거, 원문 보존용 필드입니다.
+#   - 모르는 값을 억지로 만들지 않는 것이 중요합니다. 확실하지 않으면 None 또는 빈 list가 안전합니다.
+#   - date/start_time/end_time은 확실할 때만 YYYY-MM-DD, HH:MM 형식으로 채웁니다.
+#
+# bridge 동작 기준
+#   - 요청이 하나뿐이어도 Week 2 agent의 structured_response에는 StructuredRequest 하나를 담습니다.
+#   - 여러 일정/할 일/알림 의도가 한 문장에 섞이면 Week 2 agent에서는 여러 StructuredRequest로 나눕니다.
+#   - extract_structured_request()는 bridge 용도라 StructuredRequest 하나만 반환합니다.
+#   - Week 1 personal_create_schedule은 이미 분해된 인자로 임시 일정을 생성하고,
+#     Week 2 agent와 bridge는 그 JSON payload를 읽어 저장 가능한 구조로 최종 변환한다는 차이를 비교합니다.
+#
+# 참고 코드
+#   - week01_prompt_parts()
+#      Week 1 system prompt를 이어받아 Week 2 구조화 지시를 누적할 때 사용합니다.
+#   - week01_tools()
+#      Week 1 개인 일정 tool 목록입니다. Week 2 agent는 이 tool 결과 JSON을 구조화 근거로 씁니다.
+#   - extract_structured_request / extract_schedule_request
+#      Week 3 이상에서 DB 저장/조율 tool chain에 쓰는 bridge 코드입니다.
+#      query 문자열이 자연어든 Week 1 tool JSON이든, Python rule/parser로 매핑하지 않고
+#      structured LLM 호출로 구조화한 뒤 JSON tool payload로 감쌉니다.
+#
+# 검증 방법
+#   - 메인과제: ./run.sh --week2로 실행한 뒤 "다음 주 화요일 오후 3시에 철수랑 회의 잡아줘" 같은
+#     문장을 입력합니다. 최종 답변이 StructuredRequestBatch class 형식의 structured_response로
+#     나오는지 확인합니다.
+#   - 추가 과제: Week 3을 실행한 뒤 trace에서 extract_schedule_request 이후
+#     save_structured_request가 호출되는지 봅니다. extract_schedule_request의 반환 JSON에
+#     ok/tool_name/base_date/structured_request가 들어 있는지 확인합니다.
+#
+# 함수별 동작 설명
+#   - StructuredRequest
+#     Week 2 structured output의 중심 스키마입니다. LLM이 자연어에서 뽑은 요청 종류, 제목, 날짜, 시간,
+#     멤버, 우선순위, 근거, 원문을 이 class 필드에 맞춰 반환합니다.
+#
+#   - StructuredRequestBatch
+#     StructuredRequest 여러 개와 base_date를 함께 담는 최종 structured_response 스키마입니다.
+#     요청이 하나뿐이어도 requests list 안에 StructuredRequest 하나를 담습니다.
+#
+#   - week02_tools()
+#     Week 1 개인 일정 tool을 그대로 노출합니다. Week 2 agent는 개인 일정 생성 요청에서
+#     created_schedule JSON을 structured_response의 근거로 사용할 수 있습니다.
+#
+#   - week02_system_prompt() / week02_prompt_parts()
+#     Week 1 prompt 위에 "자연어를 StructuredRequestBatch로 출력한다"는 Week 2 지시를 누적합니다.
+#
+#   - build_week02_agent() / build_week_agent()
+#     response_format=StructuredRequestBatch가 설정된 agent를 만들고 재사용합니다.
+#     build_week_agent()는 실행기가 찾는 표준 entry point입니다.
+#
+#   - _coerce_structured_request(value)
+#     LangChain structured output 결과가 이미 StructuredRequest이면 그대로 쓰고, dict이면 Pydantic 검증을 거쳐
+#     StructuredRequest로 바꿉니다. 예상한 형태가 아니면 오류를 내서 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   - extract_structured_request(text)
+#     agent loop를 새로 만들지 않고 chat_model().with_structured_output(...)만 사용해 자연어 또는 JSON 문자열을
+#     StructuredRequest로 검증/구조화합니다. Week 3 이상에서 저장/조율 직전 입력을 구조화해야 할 때 재사용하는 bridge 함수입니다.
+#
+#   - extract_schedule_request(query)
+#     Week 3 이상 agent가 저장/조율 전에 호출하는 LangChain bridge tool입니다.
+#     extract_structured_request(...) 결과에 ok/tool_name/base_date를 붙여 JSON 문자열로 반환하므로,
+#     이후 저장 tool이 structured_request 필드를 그대로 받을 수 있습니다.
+
+
+class StructuredRequest(BaseModel):
+    """LLM structured output으로 추출되는 2주차 요청 스키마입니다."""
+    
+    kind: RequestKind = Field(description="요청 종류. personal_schedule, group_schedule, todo, reminder, unknown 중 하나로 분류한다.")
````

`save_structured_request` 이후 실제 동기화 흐름까지 찾아보신 점 좋네요! 👍
말씀하신 것처럼 참석자가 있다는 사실만으로 구분하기보다, 개인 busy-time으로만 관리할지 참석자별 일정에도 반영할지가 더 명확한 기준이 될 것 같습니다.
현재 추가하신 description이면 의도가 충분히 드러나는 것 같아요. 확인했습니다~

### 코드 리뷰 댓글: @Gongmeda

시각: 2026-07-12T15:34:13Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#discussion_r3566589805
파일: `student_parts/week02_structure_natural_language_requests.py` / 줄: 238
커밋: `c456097a242a3fe4a24d2c1732449b666453ca5b`

````diff
@@ -0,0 +1,266 @@
+from __future__ import annotations
+
+import json
+from typing import Any, Literal
+
+from langchain.agents import create_agent
+from langchain.tools import tool
+from pydantic import BaseModel, Field
+
+from fixed.config import CONFIG
+from fixed.llm import chat_model
+from fixed.runtime_clock import current_app_date_iso
+from student_parts.week01_wake_up_nana import join_system_prompt, week01_prompt_parts, week01_tools
+
+
+RequestKind = Literal["personal_schedule", "group_schedule", "todo", "reminder", "unknown"]
+_WEEK02_AGENT: Any | None = None
+
+
+# [2주차 수강생 구현 가이드]
+#
+# 목표
+#   Week 2의 핵심은 사용자의 한국어 자연어 요청이나 Week 1 tool이 만든 JSON payload를
+#   일정 앱이 읽을 수 있는 StructuredRequest/StructuredRequestBatch로 바꾸는 것입니다.
+#   Week 1이 이미 정해진 인자를 받아 임시 일정을 만들었다면, Week 2는 "내일 오후 3시" 같은
+#   자연어와 created_schedule JSON을 날짜/시간/종류/멤버 필드로 구조화합니다.
+#   구조화 결과는 아직 SQLite, RAG, 외부 멤버 일정 조율 흐름에 저장하지 않습니다.
+#
+# 과제 구성
+#   - 메인과제: Week 2 agent가 자연어 또는 Week 1 tool JSON을 StructuredRequestBatch로
+#     최종 반환하는 세로 슬라이스를 완성합니다.
+#   - 추가 과제: 메인과제에서 만든 StructuredRequest 스키마를 Week 3 이상 저장/조율 흐름에서
+#     재사용할 수 있도록 bridge 함수를 완성합니다.
+#
+# 구현 위치와 사용할 코드
+#   - 이 파일(student_parts/week02_structure_natural_language_requests.py)의
+#     StructuredRequest, StructuredRequestBatch, week02_tools(), week02_prompt_parts(),
+#     week02_system_prompt(), build_week02_agent()를 확인합니다.
+#   - build_week02_agent()는 langchain.agents.create_agent, fixed/llm.py의 chat_model(),
+#     week02_system_prompt(), response_format=StructuredRequestBatch를 사용해 Week 2 agent를 만듭니다.
+#   - week02_tools()는 Week 1 도구 목록을 그대로 가져옵니다. Week 2 agent는 개인 일정 생성 요청에서
+#     personal_create_schedule이 반환한 created_schedule JSON payload를 읽고
+#     response_format=StructuredRequestBatch로 최종 구조화 결과를 확인합니다.
+#   - week02_prompt_parts()는 student_parts/week01_wake_up_nana.py의 week01_prompt_parts() 위에
+#     Week 2 구조화 지시를 추가합니다.
+#   - _coerce_structured_request(), extract_structured_request(), extract_schedule_request()는
+#     Week 3 이상에서 재사용되는 구조화 bridge입니다. Week 2 파일에 있지만 Week 2 agent에
+#     공개되는 tool은 아닙니다.
+#
+# 메인과제 구현 대상
+#   1. StructuredRequest 스키마
+#      - kind/title/date/start_time/end_time/members/priority/reason/original_text 필드가
+#        이후 Week 3 저장 payload의 기준이 됩니다.
+#      - kind는 RequestKind Literal에 들어 있는 값만 허용합니다.
+#      - 각 필드에는 LLM structured output이 이해할 수 있도록 한국어 description을 붙입니다.
+#
+#   2. StructuredRequestBatch 스키마
+#      - requests에는 StructuredRequest 목록을 담고, 요청이 하나뿐이어도 list 형태를 유지합니다.
+#      - base_date에는 상대 날짜 해석 기준일(current_app_date_iso)을 담습니다.
+#
+#   3. Week 2 agent 세로 슬라이스
+#      - week02_tools()는 Week 1 tool 목록을 그대로 반환합니다.
+#      - week02_prompt_parts()와 week02_system_prompt()에는 자연어/Week 1 tool JSON을
+#        StructuredRequestBatch로 구조화하라는 지시를 넣습니다.
+#      - build_week02_agent()에 response_format=StructuredRequestBatch를 연결해
+#        ./run.sh --week2가 동작하게 합니다.
+#      - 개인 일정 생성 요청에서는 Week 1 personal_create_schedule tool 결과의 created_schedule JSON을
+#        LLM이 읽어 StructuredRequestBatch로 최종 변환하는 흐름을 확인합니다.
+#
+# 추가 과제 구현 대상
+#   1. _coerce_structured_request
+#      - LangChain structured output 결과가 이미 StructuredRequest이면 그대로 반환합니다.
+#      - dict이면 StructuredRequest.model_validate(...)로 검증해 반환합니다.
+#      - 예상한 형태가 아니면 RuntimeError를 발생시켜 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   2. extract_structured_request
+#      - chat_model().with_structured_output(StructuredRequest, method="function_calling")를 사용합니다.
+#      - system 메시지에는 join_system_prompt(week02_prompt_parts())를 넣고,
+#        user 메시지에는 text를 넣어 structured LLM을 호출합니다.
+#      - 자연어 또는 JSON 문자열을 StructuredRequest 하나로 검증/구조화합니다.
+#
+#   3. extract_schedule_request
+#      - extract_structured_request(query) 결과에 ok/tool_name/base_date를 붙입니다.
+#      - structured_request에는 model_dump() 결과를 넣고, json.dumps(..., ensure_ascii=False)로 반환합니다.
+#      - Week 3 이상 저장 tool이 structured_request 필드를 그대로 받을 수 있게 만듭니다.
+#
+# StructuredRequest 읽는 법
+#   - kind: personal_schedule, group_schedule, todo, reminder, unknown 중 하나입니다.
+#   - title/date/start_time/end_time: 일정 앱이 실제 저장이나 생성에 사용할 핵심 필드입니다.
+#   - members: 참석자/관련 멤버 list입니다. 모르면 빈 list로 둡니다.
+#   - priority/reason/original_text: 할 일 우선순위, 판단 근거, 원문 보존용 필드입니다.
+#   - 모르는 값을 억지로 만들지 않는 것이 중요합니다. 확실하지 않으면 None 또는 빈 list가 안전합니다.
+#   - date/start_time/end_time은 확실할 때만 YYYY-MM-DD, HH:MM 형식으로 채웁니다.
+#
+# bridge 동작 기준
+#   - 요청이 하나뿐이어도 Week 2 agent의 structured_response에는 StructuredRequest 하나를 담습니다.
+#   - 여러 일정/할 일/알림 의도가 한 문장에 섞이면 Week 2 agent에서는 여러 StructuredRequest로 나눕니다.
+#   - extract_structured_request()는 bridge 용도라 StructuredRequest 하나만 반환합니다.
+#   - Week 1 personal_create_schedule은 이미 분해된 인자로 임시 일정을 생성하고,
+#     Week 2 agent와 bridge는 그 JSON payload를 읽어 저장 가능한 구조로 최종 변환한다는 차이를 비교합니다.
+#
+# 참고 코드
+#   - week01_prompt_parts()
+#      Week 1 system prompt를 이어받아 Week 2 구조화 지시를 누적할 때 사용합니다.
+#   - week01_tools()
+#      Week 1 개인 일정 tool 목록입니다. Week 2 agent는 이 tool 결과 JSON을 구조화 근거로 씁니다.
+#   - extract_structured_request / extract_schedule_request
+#      Week 3 이상에서 DB 저장/조율 tool chain에 쓰는 bridge 코드입니다.
+#      query 문자열이 자연어든 Week 1 tool JSON이든, Python rule/parser로 매핑하지 않고
+#      structured LLM 호출로 구조화한 뒤 JSON tool payload로 감쌉니다.
+#
+# 검증 방법
+#   - 메인과제: ./run.sh --week2로 실행한 뒤 "다음 주 화요일 오후 3시에 철수랑 회의 잡아줘" 같은
+#     문장을 입력합니다. 최종 답변이 StructuredRequestBatch class 형식의 structured_response로
+#     나오는지 확인합니다.
+#   - 추가 과제: Week 3을 실행한 뒤 trace에서 extract_schedule_request 이후
+#     save_structured_request가 호출되는지 봅니다. extract_schedule_request의 반환 JSON에
+#     ok/tool_name/base_date/structured_request가 들어 있는지 확인합니다.
+#
+# 함수별 동작 설명
+#   - StructuredRequest
+#     Week 2 structured output의 중심 스키마입니다. LLM이 자연어에서 뽑은 요청 종류, 제목, 날짜, 시간,
+#     멤버, 우선순위, 근거, 원문을 이 class 필드에 맞춰 반환합니다.
+#
+#   - StructuredRequestBatch
+#     StructuredRequest 여러 개와 base_date를 함께 담는 최종 structured_response 스키마입니다.
+#     요청이 하나뿐이어도 requests list 안에 StructuredRequest 하나를 담습니다.
+#
+#   - week02_tools()
+#     Week 1 개인 일정 tool을 그대로 노출합니다. Week 2 agent는 개인 일정 생성 요청에서
+#     created_schedule JSON을 structured_response의 근거로 사용할 수 있습니다.
+#
+#   - week02_system_prompt() / week02_prompt_parts()
+#     Week 1 prompt 위에 "자연어를 StructuredRequestBatch로 출력한다"는 Week 2 지시를 누적합니다.
+#
+#   - build_week02_agent() / build_week_agent()
+#     response_format=StructuredRequestBatch가 설정된 agent를 만들고 재사용합니다.
+#     build_week_agent()는 실행기가 찾는 표준 entry point입니다.
+#
+#   - _coerce_structured_request(value)
+#     LangChain structured output 결과가 이미 StructuredRequest이면 그대로 쓰고, dict이면 Pydantic 검증을 거쳐
+#     StructuredRequest로 바꿉니다. 예상한 형태가 아니면 오류를 내서 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   - extract_structured_request(text)
+#     agent loop를 새로 만들지 않고 chat_model().with_structured_output(...)만 사용해 자연어 또는 JSON 문자열을
+#     StructuredRequest로 검증/구조화합니다. Week 3 이상에서 저장/조율 직전 입력을 구조화해야 할 때 재사용하는 bridge 함수입니다.
+#
+#   - extract_schedule_request(query)
+#     Week 3 이상 agent가 저장/조율 전에 호출하는 LangChain bridge tool입니다.
+#     extract_structured_request(...) 결과에 ok/tool_name/base_date를 붙여 JSON 문자열로 반환하므로,
+#     이후 저장 tool이 structured_request 필드를 그대로 받을 수 있습니다.
+
+
+class StructuredRequest(BaseModel):
+    """LLM structured output으로 추출되는 2주차 요청 스키마입니다."""
+    
+    kind: RequestKind = Field(description="요청 종류. personal_schedule, group_schedule, todo, reminder, unknown 중 하나로 분류한다.")
+    title: str | None = Field(default=None, description="일정이나 할 일의 제목. 모르면 None.")
+    date: str | None = Field(default=None, description="YYYY-MM-DD 형식 날짜. 확실할 때만 채우고 모르면 None.")
+    start_time: str | None = Field(default=None, description="HH:MM 형식 시작 시각. 모르면 None.")
+    end_time: str | None = Field(default=None, description="HH:MM 형식 종료 시각. 모르면 None.")
+    members: list[str] = Field(default_factory=list, description="참석자나 관련 멤버 목록. 모르면 빈 리스트.")
+    priority: str | None = Field(default=None, description="할 일의 우선순위. 없으면 None.")
+    reason: str | None = Field(default=None, description="이 분류를 내린 근거. 없으면 None.")
+    original_text: str = Field(default="", description="사용자의 원문 요청을 그대로 보존.")
+
+
+class StructuredRequestBatch(BaseModel):
+    """여러 자연어 의도를 StructuredRequest 목록으로 나누는 메인과제 스키마입니다."""
+
+    requests: list[StructuredRequest] = Field(default_factory=list, description="자연어 요청을 구조화한 StructuredRequest 목록. 요청이 하나여도 리스트에 담기.")
+    base_date: str = Field(default_factory=current_app_date_iso, description="상대 날짜(내일, 다음 주 등)를 해석하는 기준일. 앱 시작 시 OS에서 읽은 오늘 날짜.")
+
+
+def _coerce_structured_request(value: Any) -> StructuredRequest:
+    """LangChain structured output 결과를 StructuredRequest로 정규화합니다."""
+
+    if isinstance(value, StructuredRequest):
+        return value
+    if isinstance(value, dict):
+        return StructuredRequest.model_validate(value)
+    raise RuntimeError(f"예상하지 못한 structured output 형태입니다: {type(value)!r}")
+
+
+def extract_structured_request(text: str) -> StructuredRequest:
+    """Week 3 이상에서 agent를 새로 띄우지 않고 자연어를 StructuredRequest로 바꿉니다."""
+
+    structured_llm = chat_model().with_structured_output(StructuredRequest, method="function_calling")
+    result = structured_llm.invoke(
+        [
+            ("system", join_system_prompt(week02_prompt_parts())),
+            ("user", text),
+        ]
+    )
+    return _coerce_structured_request(result)
+
+
+@tool
+def extract_schedule_request(query: str) -> str:
+    """Week 3 이상 agent가 저장/조율 전에 호출하는 구조화 bridge tool입니다."""
+
+    structured = extract_structured_request(query)
+    payload = {
+        "ok": True,
+        "tool_name": "extract_schedule_request",
+        "base_date": current_app_date_iso(),
+        "structured_request": structured.model_dump(),
+    }
+    return json.dumps(payload, ensure_ascii=False)
+
+
+def week02_tools() -> list[Any]:
+    """Week 2 agent에 Week 1 도구를 노출해 tool JSON을 structured_response 근거로 씁니다."""
+
+    return week01_tools()
+
+
+def week02_system_prompt() -> str:
+    """조각 리스트를 하나로 합쳐 만든 최종 system prompt 문자열입니다. prompt_parts를 join_system_prompt()로 이어붙이고 최종 답변 규칙을 덧붙입니다."""
+
+    return join_system_prompt(
+        [
+            *week02_prompt_parts(),
+            (
+                "최종 답변은 반드시 StructuredRequestBatch 형식으로만 반환한다. "
+                "요청이 하나뿐이어도 requests 목록에 StructuredRequest 하나를 담고, "
+                "여러 의도가 섞이면 여러 StructuredRequest로 나눈다. "
+                "personal_create_schedule tool을 호출한 경우 그 결과 JSON의 created_schedule을 읽어 "
+                "title/date/start_time/end_time/members 필드를 채운다."
````

네, Week 1 코드를 수정하지 않고 Week 2 prompt에서 `created_schedule.attendees`를 `StructuredRequest.members`로 옮기도록 명시한 방향이 적절해 보입니다.
수정된 내용 확인했습니다~

### 코드 리뷰 댓글: @Gongmeda

시각: 2026-07-12T15:34:35Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#discussion_r3566590295
파일: `student_parts/week02_structure_natural_language_requests.py` / 줄: 208
커밋: `c456097a242a3fe4a24d2c1732449b666453ca5b`

````diff
@@ -0,0 +1,266 @@
+from __future__ import annotations
+
+import json
+from typing import Any, Literal
+
+from langchain.agents import create_agent
+from langchain.tools import tool
+from pydantic import BaseModel, Field
+
+from fixed.config import CONFIG
+from fixed.llm import chat_model
+from fixed.runtime_clock import current_app_date_iso
+from student_parts.week01_wake_up_nana import join_system_prompt, week01_prompt_parts, week01_tools
+
+
+RequestKind = Literal["personal_schedule", "group_schedule", "todo", "reminder", "unknown"]
+_WEEK02_AGENT: Any | None = None
+
+
+# [2주차 수강생 구현 가이드]
+#
+# 목표
+#   Week 2의 핵심은 사용자의 한국어 자연어 요청이나 Week 1 tool이 만든 JSON payload를
+#   일정 앱이 읽을 수 있는 StructuredRequest/StructuredRequestBatch로 바꾸는 것입니다.
+#   Week 1이 이미 정해진 인자를 받아 임시 일정을 만들었다면, Week 2는 "내일 오후 3시" 같은
+#   자연어와 created_schedule JSON을 날짜/시간/종류/멤버 필드로 구조화합니다.
+#   구조화 결과는 아직 SQLite, RAG, 외부 멤버 일정 조율 흐름에 저장하지 않습니다.
+#
+# 과제 구성
+#   - 메인과제: Week 2 agent가 자연어 또는 Week 1 tool JSON을 StructuredRequestBatch로
+#     최종 반환하는 세로 슬라이스를 완성합니다.
+#   - 추가 과제: 메인과제에서 만든 StructuredRequest 스키마를 Week 3 이상 저장/조율 흐름에서
+#     재사용할 수 있도록 bridge 함수를 완성합니다.
+#
+# 구현 위치와 사용할 코드
+#   - 이 파일(student_parts/week02_structure_natural_language_requests.py)의
+#     StructuredRequest, StructuredRequestBatch, week02_tools(), week02_prompt_parts(),
+#     week02_system_prompt(), build_week02_agent()를 확인합니다.
+#   - build_week02_agent()는 langchain.agents.create_agent, fixed/llm.py의 chat_model(),
+#     week02_system_prompt(), response_format=StructuredRequestBatch를 사용해 Week 2 agent를 만듭니다.
+#   - week02_tools()는 Week 1 도구 목록을 그대로 가져옵니다. Week 2 agent는 개인 일정 생성 요청에서
+#     personal_create_schedule이 반환한 created_schedule JSON payload를 읽고
+#     response_format=StructuredRequestBatch로 최종 구조화 결과를 확인합니다.
+#   - week02_prompt_parts()는 student_parts/week01_wake_up_nana.py의 week01_prompt_parts() 위에
+#     Week 2 구조화 지시를 추가합니다.
+#   - _coerce_structured_request(), extract_structured_request(), extract_schedule_request()는
+#     Week 3 이상에서 재사용되는 구조화 bridge입니다. Week 2 파일에 있지만 Week 2 agent에
+#     공개되는 tool은 아닙니다.
+#
+# 메인과제 구현 대상
+#   1. StructuredRequest 스키마
+#      - kind/title/date/start_time/end_time/members/priority/reason/original_text 필드가
+#        이후 Week 3 저장 payload의 기준이 됩니다.
+#      - kind는 RequestKind Literal에 들어 있는 값만 허용합니다.
+#      - 각 필드에는 LLM structured output이 이해할 수 있도록 한국어 description을 붙입니다.
+#
+#   2. StructuredRequestBatch 스키마
+#      - requests에는 StructuredRequest 목록을 담고, 요청이 하나뿐이어도 list 형태를 유지합니다.
+#      - base_date에는 상대 날짜 해석 기준일(current_app_date_iso)을 담습니다.
+#
+#   3. Week 2 agent 세로 슬라이스
+#      - week02_tools()는 Week 1 tool 목록을 그대로 반환합니다.
+#      - week02_prompt_parts()와 week02_system_prompt()에는 자연어/Week 1 tool JSON을
+#        StructuredRequestBatch로 구조화하라는 지시를 넣습니다.
+#      - build_week02_agent()에 response_format=StructuredRequestBatch를 연결해
+#        ./run.sh --week2가 동작하게 합니다.
+#      - 개인 일정 생성 요청에서는 Week 1 personal_create_schedule tool 결과의 created_schedule JSON을
+#        LLM이 읽어 StructuredRequestBatch로 최종 변환하는 흐름을 확인합니다.
+#
+# 추가 과제 구현 대상
+#   1. _coerce_structured_request
+#      - LangChain structured output 결과가 이미 StructuredRequest이면 그대로 반환합니다.
+#      - dict이면 StructuredRequest.model_validate(...)로 검증해 반환합니다.
+#      - 예상한 형태가 아니면 RuntimeError를 발생시켜 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   2. extract_structured_request
+#      - chat_model().with_structured_output(StructuredRequest, method="function_calling")를 사용합니다.
+#      - system 메시지에는 join_system_prompt(week02_prompt_parts())를 넣고,
+#        user 메시지에는 text를 넣어 structured LLM을 호출합니다.
+#      - 자연어 또는 JSON 문자열을 StructuredRequest 하나로 검증/구조화합니다.
+#
+#   3. extract_schedule_request
+#      - extract_structured_request(query) 결과에 ok/tool_name/base_date를 붙입니다.
+#      - structured_request에는 model_dump() 결과를 넣고, json.dumps(..., ensure_ascii=False)로 반환합니다.
+#      - Week 3 이상 저장 tool이 structured_request 필드를 그대로 받을 수 있게 만듭니다.
+#
+# StructuredRequest 읽는 법
+#   - kind: personal_schedule, group_schedule, todo, reminder, unknown 중 하나입니다.
+#   - title/date/start_time/end_time: 일정 앱이 실제 저장이나 생성에 사용할 핵심 필드입니다.
+#   - members: 참석자/관련 멤버 list입니다. 모르면 빈 list로 둡니다.
+#   - priority/reason/original_text: 할 일 우선순위, 판단 근거, 원문 보존용 필드입니다.
+#   - 모르는 값을 억지로 만들지 않는 것이 중요합니다. 확실하지 않으면 None 또는 빈 list가 안전합니다.
+#   - date/start_time/end_time은 확실할 때만 YYYY-MM-DD, HH:MM 형식으로 채웁니다.
+#
+# bridge 동작 기준
+#   - 요청이 하나뿐이어도 Week 2 agent의 structured_response에는 StructuredRequest 하나를 담습니다.
+#   - 여러 일정/할 일/알림 의도가 한 문장에 섞이면 Week 2 agent에서는 여러 StructuredRequest로 나눕니다.
+#   - extract_structured_request()는 bridge 용도라 StructuredRequest 하나만 반환합니다.
+#   - Week 1 personal_create_schedule은 이미 분해된 인자로 임시 일정을 생성하고,
+#     Week 2 agent와 bridge는 그 JSON payload를 읽어 저장 가능한 구조로 최종 변환한다는 차이를 비교합니다.
+#
+# 참고 코드
+#   - week01_prompt_parts()
+#      Week 1 system prompt를 이어받아 Week 2 구조화 지시를 누적할 때 사용합니다.
+#   - week01_tools()
+#      Week 1 개인 일정 tool 목록입니다. Week 2 agent는 이 tool 결과 JSON을 구조화 근거로 씁니다.
+#   - extract_structured_request / extract_schedule_request
+#      Week 3 이상에서 DB 저장/조율 tool chain에 쓰는 bridge 코드입니다.
+#      query 문자열이 자연어든 Week 1 tool JSON이든, Python rule/parser로 매핑하지 않고
+#      structured LLM 호출로 구조화한 뒤 JSON tool payload로 감쌉니다.
+#
+# 검증 방법
+#   - 메인과제: ./run.sh --week2로 실행한 뒤 "다음 주 화요일 오후 3시에 철수랑 회의 잡아줘" 같은
+#     문장을 입력합니다. 최종 답변이 StructuredRequestBatch class 형식의 structured_response로
+#     나오는지 확인합니다.
+#   - 추가 과제: Week 3을 실행한 뒤 trace에서 extract_schedule_request 이후
+#     save_structured_request가 호출되는지 봅니다. extract_schedule_request의 반환 JSON에
+#     ok/tool_name/base_date/structured_request가 들어 있는지 확인합니다.
+#
+# 함수별 동작 설명
+#   - StructuredRequest
+#     Week 2 structured output의 중심 스키마입니다. LLM이 자연어에서 뽑은 요청 종류, 제목, 날짜, 시간,
+#     멤버, 우선순위, 근거, 원문을 이 class 필드에 맞춰 반환합니다.
+#
+#   - StructuredRequestBatch
+#     StructuredRequest 여러 개와 base_date를 함께 담는 최종 structured_response 스키마입니다.
+#     요청이 하나뿐이어도 requests list 안에 StructuredRequest 하나를 담습니다.
+#
+#   - week02_tools()
+#     Week 1 개인 일정 tool을 그대로 노출합니다. Week 2 agent는 개인 일정 생성 요청에서
+#     created_schedule JSON을 structured_response의 근거로 사용할 수 있습니다.
+#
+#   - week02_system_prompt() / week02_prompt_parts()
+#     Week 1 prompt 위에 "자연어를 StructuredRequestBatch로 출력한다"는 Week 2 지시를 누적합니다.
+#
+#   - build_week02_agent() / build_week_agent()
+#     response_format=StructuredRequestBatch가 설정된 agent를 만들고 재사용합니다.
+#     build_week_agent()는 실행기가 찾는 표준 entry point입니다.
+#
+#   - _coerce_structured_request(value)
+#     LangChain structured output 결과가 이미 StructuredRequest이면 그대로 쓰고, dict이면 Pydantic 검증을 거쳐
+#     StructuredRequest로 바꿉니다. 예상한 형태가 아니면 오류를 내서 잘못된 LLM 응답을 조용히 통과시키지 않습니다.
+#
+#   - extract_structured_request(text)
+#     agent loop를 새로 만들지 않고 chat_model().with_structured_output(...)만 사용해 자연어 또는 JSON 문자열을
+#     StructuredRequest로 검증/구조화합니다. Week 3 이상에서 저장/조율 직전 입력을 구조화해야 할 때 재사용하는 bridge 함수입니다.
+#
+#   - extract_schedule_request(query)
+#     Week 3 이상 agent가 저장/조율 전에 호출하는 LangChain bridge tool입니다.
+#     extract_structured_request(...) 결과에 ok/tool_name/base_date를 붙여 JSON 문자열로 반환하므로,
+#     이후 저장 tool이 structured_request 필드를 그대로 받을 수 있습니다.
+
+
+class StructuredRequest(BaseModel):
+    """LLM structured output으로 추출되는 2주차 요청 스키마입니다."""
+    
+    kind: RequestKind = Field(description="요청 종류. personal_schedule, group_schedule, todo, reminder, unknown 중 하나로 분류한다.")
+    title: str | None = Field(default=None, description="일정이나 할 일의 제목. 모르면 None.")
+    date: str | None = Field(default=None, description="YYYY-MM-DD 형식 날짜. 확실할 때만 채우고 모르면 None.")
+    start_time: str | None = Field(default=None, description="HH:MM 형식 시작 시각. 모르면 None.")
+    end_time: str | None = Field(default=None, description="HH:MM 형식 종료 시각. 모르면 None.")
+    members: list[str] = Field(default_factory=list, description="참석자나 관련 멤버 목록. 모르면 빈 리스트.")
+    priority: str | None = Field(default=None, description="할 일의 우선순위. 없으면 None.")
+    reason: str | None = Field(default=None, description="이 분류를 내린 근거. 없으면 None.")
+    original_text: str = Field(default="", description="사용자의 원문 요청을 그대로 보존.")
+
+
+class StructuredRequestBatch(BaseModel):
+    """여러 자연어 의도를 StructuredRequest 목록으로 나누는 메인과제 스키마입니다."""
+
+    requests: list[StructuredRequest] = Field(default_factory=list, description="자연어 요청을 구조화한 StructuredRequest 목록. 요청이 하나여도 리스트에 담기.")
+    base_date: str = Field(default_factory=current_app_date_iso, description="상대 날짜(내일, 다음 주 등)를 해석하는 기준일. 앱 시작 시 OS에서 읽은 오늘 날짜.")
+
+
+def _coerce_structured_request(value: Any) -> StructuredRequest:
+    """LangChain structured output 결과를 StructuredRequest로 정규화합니다."""
+
+    if isinstance(value, StructuredRequest):
+        return value
+    if isinstance(value, dict):
+        return StructuredRequest.model_validate(value)
+    raise RuntimeError(f"예상하지 못한 structured output 형태입니다: {type(value)!r}")
+
+
+def extract_structured_request(text: str) -> StructuredRequest:
+    """Week 3 이상에서 agent를 새로 띄우지 않고 자연어를 StructuredRequest로 바꿉니다."""
+
+    structured_llm = chat_model().with_structured_output(StructuredRequest, method="function_calling")
+    result = structured_llm.invoke(
+        [
+            ("system", join_system_prompt(week02_prompt_parts())),
+            ("user", text),
+        ]
+    )
+    return _coerce_structured_request(result)
+
+
+@tool
+def extract_schedule_request(query: str) -> str:
````

PR 본문에서 심화 과제 수행 범위를 수정해주신 부분 확인했습니다.
구현된 bridge 함수도 이번 심화 과제의 의도에 맞게 연결되어 있는 것 같습니다~

### 리뷰: @Gongmeda / APPROVED

시각: 2026-07-12T15:35:18Z
원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/59#pullrequestreview-4680300472

안녕하세요~
피드백 반영해주시느라 수고 많으셨습니다.

기존 코멘트가 모두 잘 반영되었고 기능상 크리티컬한 부분은 없는 것 같아 어프루브 드립니다!

## 변경 파일

- `.github/pull_request_template.md` (+11 / -3)
- `.gitignore` (+0 / -1)
- `fixed/week_agent_registry.py` (+3 / -2)
- `run.sh` (+7 / -4)
- `student_parts/week02_structure_natural_language_requests.py` (+275 / -0)
