---
name: dashboard-change
description: KPI 대시보드(KPI/pmkt-kpi.html), 데이터 JSON, 또는 파이프라인(C:\bang) 스크립트를 수정할 때 사용. 변경→프리뷰 검증→커밋→배포 검증까지의 필수 절차를 강제한다. 대시보드 UI/차트/표/탭 수정, JSON 스키마 변경, gen_realtime_dashboard.py·teams_notify.ps1 수정, 또는 "대시보드가 안 바뀐다/반영 안 됐다" 류 문제를 다룰 때 항상 사용.
---

# 대시보드 변경 워크플로우

CLAUDE.md의 철칙 R1~R8을 이 절차로 강제한다. **각 단계의 검증 게이트를 통과하기 전에 다음 단계나 완료 보고로 넘어가지 않는다.**

## 1. 변경 대상 파악 + 기존 자산 재사용 (R2)
- 어느 탭/섹션인지 먼저 특정한다(`ALL_VIEWS`, `<h2>`/`card-title` grep).
- 계산이 필요하면 **새 함수를 만들기 전에 기존 집계 함수를 찾아 재사용**한다. 예: 임의 기간 합산은 `aggRangeFull`/`aggChannelRange`/`aggChAdRange`, 월별 값은 전역 `ANNUAL` 배열. 병렬 계산 로직 신설 금지.

## 2. 데이터 관련이면 원본에서 재생성·대조 먼저 (R1)
- "값이 이상하다"는 판단은 `python C:\bang\gen_realtime_dashboard.py` 재실행 + Excel 마스터 셀 대조로 검증한 뒤에만 한다.
- Excel COM 실행 전 `tasklist | grep -i EXCEL`로 마스터가 닫혀 있는지 확인(R8-3).
- 퍼센트 지표의 YoY는 %p(차이)인지 %(상대)인지 라벨과 맞춘다(R7). 비율은 raw 단위에서 계산(R1).

## 3. 프리뷰 검증 게이트 (R3) — 통과 전 보고 금지
- `.claude/launch.json`의 서버를 `preview_start`로 띄운다(없으면 생성).
- 편집 후: 리로드(`preview_eval` `window.location.reload()`) → `preview_console_logs`(level error, **에러 0 확인**) → `preview_eval`/`preview_snapshot`으로 **그 탭의 표/패널 숫자와 변경 결과가 일치하는지 대조**.
- 시각 변화는 마지막에 `preview_screenshot`으로 증빙. 검증 자체는 텍스트 도구로 한다.

## 4. 커밋 · push
- `git add <파일>` → 커밋(메시지 끝 `Co-Authored-By: Claude <noreply@anthropic.com>`) → `git push origin main`.
- push 거부 시: `git fetch origin`으로 무관 변경(대개 ad-data.json) 확인 후 `git pull --rebase origin main` → 재push.
- `C:\bang` 파이프라인 레포는 명시 요청 없이는 커밋하지 않는다.

## 5. 배포 검증 게이트 (R4)
- `gh run list --repo hyunyee/wconcept-dashboard --limit 3`으로 Pages 빌드 성공 확인.
- 실패/멈춤(`Deployment failed, try again later`/무한 queued): `gh run rerun`과 씨름하지 말고 **파이프라인을 한 번 더 실행해 새 데이터 커밋으로 배포를 강제 트리거**한다.
- 프로덕션 재배포는 비가역 액션이므로 승인 먼저(R5).
- 라이브 반영은 `curl -s <live-url> | grep <새 코드 마커>`로 확인.

## 6. 외부 액션은 승인 먼저 (R5) / 시크릿 (R6)
- Teams 실발송·시크릿 커밋 등은 AskUserQuestion으로 확인 후 실행.
- API 키가 화면/스크린샷/입력에 보이면 즉시 폐기·재발급 안내, 절대 에코·저장 안 함.

## 7. 마일스톤이면 memory 갱신
- 완료 후 fable5 로컬 memory(handoff/pipeline-*)를 갱신하되, **모델 무관 규칙은 CLAUDE.md에** 반영한다.
