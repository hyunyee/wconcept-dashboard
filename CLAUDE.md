# CLAUDE.md — W Concept 마케팅 KPI 대시보드

> 이 파일은 **모든 모델·모든 세션에 자동 로드**되는 단일 규칙 소스입니다.
> 아래 철칙(R1~R8)은 "가능하면"이 아니라 **필수 게이트**입니다. 지키기 전엔 "됐다"고 보고하지 마세요.

## 시스템 개요
- **대시보드**: `KPI/pmkt-kpi.html` — 단일 파일 정적 SPA(Vanilla JS + Chart.js). GitHub Pages(`https://hyunyee.github.io/wconcept-dashboard/KPI/pmkt-kpi.html`)로 배포. 백엔드 없음.
- **데이터**: `KPI/pmkt_kpi_data.json` — `C:\bang\gen_realtime_dashboard.py`가 OneDrive Excel 마스터파일을 COM으로 읽어 생성 → git 자동 커밋·push. Windows 작업스케줄러가 하루 5회(09/12/14/16/18시) 구동.
- **레포 2개(별개 git)**:
  - `C:\bang\hyunyee-dashboard` = 이 레포(대시보드·데이터, 커밋 대상).
  - `C:\bang` = 파이프라인 스크립트(gen_realtime_dashboard.py, teams_notify.ps1 등). **명시적 요청 없이는 커밋하지 말 것.**
- 세션 작업 디렉토리는 보통 `KPI/`. 상세 운영지식은 fable5 로컬 memory에만 있으니, **모델 무관 규칙은 이 CLAUDE.md가 유일 소스**.

## 철칙 (Golden Rules — 검증 게이트)

**R1. 숫자는 원본에서, 검증 후 말한다.**
데이터가 이상하다고 단정하기 전에 `python C:\bang\gen_realtime_dashboard.py`로 JSON을 재생성하고 Excel 마스터 셀과 직접 대조한다(기억·추측으로 진단 금지). 값이 어긋나면 **가장 raw한 단위(원 단위, 억 아님)**에서 재계산한다 — 억 단위로 반올림한 뒤 비율(YoY)을 계산하면 오차가 누적된 사례가 있다.

**R2. 기존 검증 함수를 재사용한다. 병렬 계산 로직을 새로 만들지 않는다.**
특히 "지원이"(Q&A): LLM은 **의도 파싱·문장화만** 하고 **모든 숫자는 이미 화면 렌더링에 쓰이는 집계 함수**(`aggWeekFull`/`getFunnelDiagnosis`/전역 `ANNUAL` 배열 등)에서 가져온다. 새 수치 계산 함수를 지어내면 할루시네이션이 생기고 화면 표와 값이 어긋난다.

**R3. 프리뷰로 검증하기 전엔 "됐다"고 하지 않는다.**
브라우저에서 관찰 가능한 변경은: preview 서버 리로드 → `preview_console_logs`(에러 0 확인) → **그 탭 자체의 표/패널 숫자와 결과가 일치하는지 대조**까지 마친 뒤 보고한다. 스크린샷은 최종 증빙용, 검증은 텍스트 도구로.

**R4. 배포는 확인한다. 반영됐다고 가정하지 않는다.**
push 후 `gh run list --repo hyunyee/wconcept-dashboard`로 Pages 빌드 성공을 확인한다. 빌드가 실패/멈추면 `gh run rerun`과 씨름하지 말고 **파이프라인을 한 번 더 돌려 새 데이터 커밋으로 배포를 강제 트리거**한다(검증된 복구법). "알림(Teams)은 오는데 대시보드가 안 바뀜" 증상은 데이터 커밋은 됐는데 **Pages 배포만 실패**한 경우일 수 있으니 이걸 먼저 의심한다.

**R5. 외부·비가역 액션은 승인 먼저.**
Teams 실제 발송, 공개 페이지에 시크릿 커밋, 프로덕션 재배포 등은 AskUserQuestion으로 확인한 뒤 실행한다.

**R6. 시크릿 반사신경.**
API 키를 절대 에코·저장·재사용하지 않는다. 사용자 스크린샷에 `sk-`/`Bearer`/긴 랜덤 토큰이 보이면 즉시 폐기·재발급을 안내한다(과거 반복 사고). 정적 사이트라 클라이언트 JS에 트리거 URL이 노출될 수밖에 없을 때는, 프록시 서버 대신 **Anthropic Console 지출 한도 설정**을 실질적 방어책으로 안내한다.

**R7. 퍼센트 산수.**
이미 퍼센트인 지표(CR, 취반품률, 달성률 등)의 YoY는 상대변화율 `(a-ly)/ly`가 아니라 **두 값의 차 `a-ly` (단위 %p)**다. 라벨과 수식이 일치하는지 항상 확인한다.

**R8. Windows / PowerShell 함정.**
1. PowerShell `ConvertFrom-Json`은 **대소문자만 다른 형제 키를 충돌로 처리**한다 → JSON 키(특히 채널ID)를 만들 때 원본 데이터의 표기 불일치(대소문자·공백)를 canonicalize한다.
2. `pwsh`(PowerShell 7) ≠ `powershell`(5.1) — 기능·구문 다름.
3. Excel COM 작업 전 마스터 파일이 **닫혀 있어야** 한다(`tasklist | grep EXCEL`로 확인).
4. native curl에 UTF-8(한글) 본문을 넘길 땐 셸 인코딩이 깨지니 **파일로 저장 후 `--data-binary @file`** 사용.

## 핵심 파일 지도
- `KPI/pmkt-kpi.html` — 대시보드 전체. 탭 시스템(`ALL_VIEWS`/`showView`), 지원이 팝업(FAB + `jw*` 함수들), Chart.js 렌더러.
- `KPI/pmkt_kpi_data.json` — `generated` + `RAW`(일→시간×[orders,revenue,net,dau,appdau,adcost]) + `DAILY` `DAILY_AD` `TRAFFIC` `CONV_CH` `DAILY_TGTS` `MONTHLY_TGTS` `REAL_MONTHLY` `CH_AD`(chid→[join,first,rev,sess]) `CH_AD_META`.
- `C:\bang\gen_realtime_dashboard.py` — `read_*_sheet()` + `main()`. Excel COM으로 마스터 읽어 JSON 생성·push.
- `C:\bang\teams_notify.ps1` — Teams 알림(웹훅 URL, `#daily`/`#realtime` 앵커 분기).
- `C:\bang\pmkt_hourly_sync.ps1` — 시간별 동기화 오케스트레이션(BIZW/GA4 → Excel → JSON → push → Teams).
- `C:\bang\jiwoni-flow-spec.md` — 지원이 Power Automate Flow A/B 스펙(시스템 프롬프트 포함).

## 커밋 / 배포 규율
- 데이터 커밋("data: … 갱신")은 파이프라인이 자동 생성한다.
- 코드 push가 거부되면 `git pull --rebase origin main`(무관한 ad-data 커밋과 경합) 후 재push.
- 커밋 메시지 끝: `Co-Authored-By: Claude <noreply@anthropic.com>`.
- `C:\bang` 파이프라인 레포는 명시적 요청 없이는 커밋하지 않는다.

## 반복 작업은 skill 사용
- 대시보드/파이프라인 수정 → **`dashboard-change`** skill(변경→프리뷰검증→커밋→배포검증 절차).
- 지원이에 새 질문 패턴/능력 추가 → **`jiwoni-capability`** skill(3단계 파이프라인·탭 스코프 규칙).
