---
name: jiwoni-capability
description: 지원이(대시보드 우측 하단 플로팅 자연어 Q&A 팝업)에 새 질문 패턴/능력을 추가하거나 고칠 때 사용. 3단계 파이프라인(의도파싱→계산→문장화)과 탭 스코프 규칙을 지켜야 하는 작업 — 새 질문 유형 지원, jwCompute/jwMockParseIntent/jwGetTabScope/Flow A·B 관련 수정, 지원이 답변이 틀리거나 탭과 안 맞는 문제를 다룰 때 항상 사용.
---

# 지원이 능력 추가 워크플로우

`KPI/pmkt-kpi.html`의 `jw*` 함수군. **숫자를 LLM이 만들게 하지 않는 것**이 이 기능의 존재 이유다.

## 아키텍처 불변식 (절대 어기지 말 것)
1. **의도 파싱** — Flow A(실LLM) 또는 `jwMockParseIntent`(로컬 폴백). 질문 → `{pattern, slots}`.
2. **계산** — 브라우저에서 `jwCompute`가 **기존 검증 집계 함수만** 호출. 여기서 나온 숫자가 유일한 진실.
3. **문장화** — Flow B(실LLM) 또는 `jwMockPhrase`. 계산 결과 JSON을 문장으로만 바꿈, 새 숫자 생성 금지.

LLM은 1·3(언어)만, 2(숫자)는 절대 안 건드린다.

## 새 패턴 추가 절차
1. **compute 함수** `jwComputeXxx(slots, scope)` 작성 — **기존 집계 재사용**. 예: `annual_summary`는 새 집계 없이 전역 `ANNUAL` 배열(연간 표가 이미 채워둠)에서 골라 쓰기/합산만 함. 반환에 `pattern`·`debugRange`·`data` 포함.
2. **dispatch 등록** — `jwCompute`의 dispatch 객체에 `xxx: jwComputeXxx` 추가.
3. **로컬 파서** — `jwMockParseIntent(q, scope)`에 트리거 키워드 추가. 특정 탭 전용이면 `scope.scope` 조건으로 게이트(예: 실적진단은 `scope.scope==='realtime'`, 연간요약은 `'month'`). **순서 주의**: 더 구체적인 조건을 일반 조건(예 highlights)보다 위에 둔다.
4. **문장화** — `jwMockPhrase`에 그 pattern의 `case` 추가.
5. **예시 질문** — `JW_EXAMPLES_BY_SCOPE`의 해당 스코프에, **그 탭에 실제로 있는 항목만** 넣는다(연간 탭엔 채널ID breakdown이 없으니 채널ID 질문을 예시로 넣지 않는 식).

## 탭 스코프 규칙 (`jwGetTabScope`)
각 탭이 **이미 쓰는 기간·비교 관례를 그대로 따른다. 새 규칙을 발명하지 않는다.**
- Daily = 마감 전일 / 전주 동요일
- Weekly = 마감주 / 전주(WoW) — 기존 동작 유지
- 연간 = 이번 달 MTD / 전년 동월(YoY)
- 기간조회 = `#period-from`~`#period-to` 선택 구간 / 전년 동기간(YoY)
- 실시간 = `selectedDate` + 그 탭에서 켜진 비교 토글(`getFilteredCompDates`)
- 주의: `wkOffset()`은 Date 객체를 반환하므로 문자열 날짜를 기대하는 범위 함수에 넘기기 전 `localDateToStr()`로 변환.

## 검증 (R3 준수)
1. **로컬 폴백(mock)에서 먼저** — Flow URL 비워둔 상태로 예시 질문 전부 통과시키고, `preview_eval`로 **답변 수치를 그 탭 표/패널과 대조**.
2. 그다음 실LLM(Flow A/B) 경로 재확인.
3. **Flow A 시스템 프롬프트는 사용자가 Power Automate에서 수동 교체**해야 한다(내가 대신 못 함). 새 패턴을 추가했으면 `jiwoni-flow-spec.md`를 갱신하고 **정확한 교체 문구를 사용자에게 제공**한다.
4. 콘솔 에러 0 확인 후에만 완료 보고.

## 데이터 한계 (정상, 코드로 못 고침)
- `CH_AD`의 세션(sess)은 최근 ~34일만 보유(트래픽 당월 시트 리텐션) → 연간/오래된 기간의 채널ID 랭킹에서 세션이 비어 보일 수 있음.
- 시크릿(R6): Flow A/B 트리거 URL은 공개 JS에 노출됨 → 지출 한도가 방어책. 새 URL을 코드에 넣는 커밋은 승인 먼저(R5).
