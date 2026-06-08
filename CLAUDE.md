# CLAUDE.md — hancomdocs-capture 디버깅/검증 노트

> 공개 배포 예정 스킬. 이 문서에 **계정/이메일/개인 절대경로 등 사적 정보 금지.** 기술적 사실만 기록.

## 구조 한눈에
- `scripts/doctor.js` — 프리플라이트(+`--deep`: 실제 세션 생존 검증).
- `scripts/login.js` — 대화형 로그인(헤디드). 프로파일 락 자가치유. `auth.json` 저장.
- `scripts/hancom.js` — `capture`/`zoom`/`around`/`locate`. 캡처는 pointer-only(headless).
- `scripts/pw-profile/` — 영속 chromium 프로파일(login 전용). 락 충돌 주의.
- `scripts/captures/` — 결과 PNG.
- `auth.json`, `_*.js`(실험 스크립트), `test fixture`, 캡처는 .gitignore.

## STEP 7(텍스트 검색) — 회귀/수정 이력
- **증상(과거):** `around`/`locate`가 `검색칸 탐색 실패`(exit 1)로 깨짐.
- **근본 원인:** 찾기 다이얼로그를 **툴바 하드코딩 좌표**(`click(309,95)`→`click(335,167)`)로 열어서. 창 크기·UI 버전·배율이 바뀌면 클릭이 빗나가 다이얼로그가 안 열림 → 검색 input 못 찾음.
- **수정(현재 코드):** `hancom.js`의 `openFindDialog()`가 **DOM 셀렉터 방식**으로 교체됨.
  - 메인 찾기 버튼: `a[title="찾기"]` 클릭(좌표 무관).
  - 드롭다운 '찾기...' 항목: DOM에서 `offsetParent!==null` 가시성 필터 + 실제 중심좌표를 읽어 클릭.
- **교훈:** capture/zoom은 `detectPageRect` 기반이라 견고. UI 요소 클릭은 **좌표 금지, 셀렉터/DOM-위치**로.

## "파란 원/물방울" — 진짜 원인 규명 (2026-06-06, 정정됨) ★중요
**오진 정정:** 협업자/좀비/원격 마우스 포인터가 **아니었다.** 협업 흔적 측정값은 `cursorChildren:0`,
`no_collaborationusers`, DOM 파란요소 0개 — 다른 세션은 없었다.
- **진짜 정체:** 문서 진입 직후 webhwp 가 띄우는 **'본인 진입 presence' 애니메이션(파란 물방울)**.
  - 위치 **고정** = 페이지1 좌상단(문서 원점, 새 세션 캐럿 위치). 항상 같은 자리(제목 첫 글자).
  - 수명 **~2.5~3초** 후 자연 소멸. 그 뒤로는 안 나타남.
  - webhwp 는 캔버스를 **2장**(문서용 idx0 + presence 오버레이 idx1, 크기 동일) 사용 →
    물방울은 **오버레이 캔버스(idx1)**에 렌더(측정: blue≈501px → 0). DOM 아님.
- **왜 캡처에만 찍혔나:** capture 는 물방울이 떠 있는 ~2~3초 내에 스크린샷을 찍어서. (반대로 5초 뒤
  찍은 진단 스크린샷은 깨끗했음 — 타이밍 문제였다.)
- **수정(현재 코드) = 오버레이 캔버스 숨김(대기 0초):** `hideOverlays` 안의 `hideOverlayCanvases(ed)`가
  스크린샷 직전에 **오버레이 캔버스만 `visibility:hidden`** 처리. 본문(문서 캔버스)은 그대로 →
  물방울/커서만 즉시 사라짐. **기다리지 않음.**
  - 문서/오버레이 식별: 캔버스별 **불투명(alpha>10) 픽셀 수**를 세서, 최댓값의 **5% 미만인 캔버스만**
    오버레이로 보고 숨김(문서 타일 보호). 측정 예: 문서 opaque≈700k / 오버레이 opaque≈499.
    ⚠️ '어두운 픽셀=글자'로 식별하면 **투명 픽셀(rgb0,0,0,a0)이 어둡게 잡혀 문서/오버레이가 뒤집힘** —
    반드시 **불투명/흰 배경 기준**으로 식별할 것(이 함정에 한 번 빠졌었음).
  - 검증: 전체 p1 캡처 ~8.0s → **~6.5s**(대기 2.5s 제거), 본문 멀쩡·물방울 0.
- **폐기된 중간안:** `settleInClip`(클립 안 파란픽셀 사라질 때까지 대기)도 동작했으나 ~2.5s 대기 비용이
  있어, 더 빠른 '오버레이 캔버스 숨김'으로 대체함. (개념은 동일: 물방울은 오버레이 층에만 있다.)
- **잘못된 옛 결론 폐기:** "캔버스라 못 숨김 → 단일 세션 보장만이 해결"은 틀림. 캔버스를 **층으로 나눠**
  오버레이만 가리면 됨. cursorChildren 측정은 '진짜 협업자' 판단엔 유효하나 이 물방울과는 무관.

## 탭 중복 — 부수 수정
- 드라이브는 **'한 번 클릭 = 열기'**. 과거 `openDoc`이 `row.dblclick()`이라 편집기 탭이 2개 뜰 수
  있었음(둘째 방치 탭이 같은 계정 협업자로 잡힐 위험). → **`row.click()`(단일)**로 변경(중복 예방).
  (단, 위 파란 물방울의 원인은 이 중복이 아니라 presence 애니메이션이었다.)

## 2026-06-06 실측 (Session 2, 데스크톱 대화형)
- doctor `READY` ~310ms / `--deep` AUTH_EXPIRED→로그인 후 READY(authLive) ~1.85s.
- capture(full) 오버레이 숨김 적용 후 **~6.5s**(이전 ~8s), grid·zoom 정상.
- STEP 7(검색) 수정 확인. 파란 물방울 = 오버레이 캔버스 숨김으로 제거 확인(전체 p1 여러 번 깨끗).

## 운영 주의
- **두 세션 동시 명령 금지.** login/capture가 같은 `pw-profile`을 공유 → 동시 실행 시 `SingletonLock`
  충돌. (실제 다른 사람이 같은 문서를 편집 중이면 그땐 진짜 협업 커서가 보일 수 있음 — 그건 결함 아님.)
- 불변식: OS 분기 금지(`if(windows)`/`process.platform` 추가 금지), 블라인드 입력 금지(검색은 찾기칸에만),
  capture는 pointer-only(shell-out 재도입 금지).
- 실험 스크립트(`scripts/_*.js`)는 .gitignore — 커밋 안 됨.

## Mac 콜드스타트 실측 (2026-06-08)
대상 코드 = `feat/win-compat` 최신(`5cac41e`, around --zoom band 픽스 포함). **코드 무수정**으로 도는지 증명.
환경: macOS, **Apple Silicon(arm64)**, node v24.14.0, chromium=playwright 1223(arm64). 모두 doctor가 풀경로 인식.

### ① 진짜 콜드스타트 — 별도 `claude -p` 인스턴스 (맥락 0, SKILL.md만) ★본 증거
- 방법: 사전 지식·대화 맥락이 **전혀 없는 새 `claude -p` 프로세스**에 "이 스킬을 처음 받았다 치고 SKILL.md만
  보고 어떤 .hwp의 1쪽을 캡처하라"만 지시. 공정성 위해 **이 CLAUDE.md(자동 로드되는 개발노트)는 잠시 숨겨**
  SKILL.md/ORDER_SPEC/DEBUG_NOTES만 노출. 내부 지식(doctor·격자·캐럿 등) 일절 안 흘림.
- 결과: **막힘 0.** `doctor.js`(→READY, `who:"agent"`) → `hancom.js capture --file ... --page 1` 순서로
  SKILL.md가 시키는 대로 **한 번에 완주**. 산출 PNG = A4 깨끗·**파란 물방울 없음**·본문 보존.
- 결론: **SKILL.md 단독으로 신규 에이전트가 Mac에서 캡처 완주 가능 — 단 happy-path 한정**
  (deps·auth 이미 충족 → doctor READY → capture). 마찰 경로 미검증: 아래 "콜드스타트 검증 범위" 참조.

### ② 전체 명령 표면 점검 (같은 날·같은 환경, 보강)
> 맥락 보유 세션에서 돌린 **보강 점검**(명령 커버리지·Mac 회귀 확인용). 콜드스타트 본 증거는 ①.

| 명령 | 결과 | 시간 |
|---|---|---|
| `doctor.js` / `--deep` | READY / authLive=true(세션 라이브) | ~0.21s / ~2.26s |
| login | **불필요**(세션 유효, 한컴 세션쿠키 무만료). DEPS/AUTH 미충족 경로는 콜드설치 때 별도 증명 | — |
| capture(full) / `--grid` | A4 깨끗·물방울 없음·본문 보존 / 격자·경계 정상 | ~6.9s / ~6.9s |
| zoom `--scale 3` | 선명 (단 좌표는 **격자에서 읽어야**; 추측하면 빈 영역=백지) | ~7.2s |
| around / `--zoom --band 200`(2쪽) | found:true / 매치 줄이 밴드 안 = **band 픽스 회귀 OK** | ~12.5s / ~12.0s |
| locate | 3단서 1쪽 다수결 수렴 | ~39.8s |

### Mac 회귀 점검
- **물방울 제거(본문 보존): OK.** 모든 캡처에서 좌상단 presence 물방울 없음, 본문 100% 보존.
- **캔버스 구조(실측):** 캔버스 2장, CSS 1235×1252. 백킹스토어 DSF 비례(1.5→1852×1878, **2(retina)→2470×2504**).
  불투명 픽셀 문서캔버스 ~1.40M(DSF1.5)/~1.84M(DSF2) vs **오버레이 0** → 5% 휴리스틱이 문서 보존·오버레이만
  숨김. 비율 기준이라 **retina에서도 본문 사라짐 없음**(scale-invariant).
- **찾기 다이얼로그(셀렉터): OK.** `a[title="찾기"]`가 Mac UI에서 동작(around/locate/around --zoom 성공).

### 콜드스타트 검증 범위 (정직한 한계) ★
① 진짜 콜드스타트가 실제로 탄 길 = **happy-path 뿐**: deps·auth 이미 충족(이번 run은 세션이 살아 있었음) →
`doctor` READY → `capture`. 즉 "친화성 OK"는 **이 happy-path에 한정**이다. 정작 콜드스타트에서 제일 잘 막히는
두 마찰 경로는 naive 에이전트로 **이번에 안 닿았다**:
- **AUTH_MISSING/AUTH_EXPIRED → 사용자가 `! node login.js`** (Session-0/로그인 창 가시성 = 콜드스타트 최대 마찰).
  이번 run은 세션이 유효해 **안 거침**. (콜드설치 시 이 경로를 본 적은 있으나 그건 별개 과거 관측이지 이번
  naive-cold 의 직접 증거가 아님 — 미검증으로 둔다.)
- **`zoom` 격자-우선 흐름** — ②(맥락 보유)에서만 백지 1회로 걸렸고, ① 콜드는 capture만 시켜 미도달.

→ 추적표 닫을 때 단서: **"콜드스타트는 happy-path(auth 유효+capture)만 naive 검증; login/zoom 마찰 경로는 미검증."**

### SKILL.md 개선 제안 (1개)
- `zoom`은 격자 좌표를 먼저 안 읽고 추측하면 빈 영역을 잘라 **백지**가 난다(② 중 1회). SKILL.md 경로 B에
  "격자 읽고 zoom"이 이미 있으나, "좌표 추측 금지(빗나가면 백지)" 한 줄 강조하면 콜드 에이전트가 덜 헤맨다.

### 불변식
- OS 분기 없음 / auth 비커밋 / 블라인드입력 없음 / pointer-only / 사적정보·일반화 없음 / 에이전트 login 미실행 : 각 OK.
