# #18 week1: 개인 일정 CRUD tool 구현

원문: https://github.com/kakaotechcampus-4/pusan-clone/pull/18

작성자: @kmgvv23
작성 시각: 2026-06-30T07:26:35Z
보관 시각 (UTC): 2026-10-06T15:19:25.015347+00:00

## PR 본문

## 목표
- 현재 대화 전용 임시 개인 일정 CRUD tool 3개 구현

## 구현 내용
- `personal_create_schedule`: 제목/날짜/시간/참석자 인자로 일정 생성, session_id로 대화 범위 분리
- `personal_list_schedules`: 현재 대화 범위 일정 조회, date_from/date_to 날짜 필터 적용
- `personal_delete_schedule`: 현재 대화 범위의 schedule_id 일치 일정만 삭제

## KPT 회고
- **Keep**: 가이드 주석을 꼼꼼히 읽고 session_scope 분리 요건을 정확히 반영함
- **Problem**: system prompt를 더 다양하게 실험해보지 못함
- **Try**: 다음 주차에서 SQLite 연동 시 스키마 설계를 미리 고민해볼 것

## 리뷰와 대화

## 변경 파일

- `student_parts/week01_wake_up_nana.py` (+34 / -7)
