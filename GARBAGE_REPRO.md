# "조용한 garbage 렌더"(모드 ②) 재현 과정 — 디버그팀용

## TL;DR
그 garbage 파일은 **claw-hwp 스킬이 만든 게 아니다.** 스킬(`create.js`)로 생성한 파일(t1~t5)은 전부 한컴독스에서 정상 렌더됐다.
garbage는 내가 **유효한 .hwpx를 바이트 단위로 잘라(절단) 일부러 손상**시킨 결과물이고, 그걸 한컴독스가 에러 없이 "열어서" 깨진 바이트를 텍스트로 토해낸 것이다.
→ **claw-hwp 쪽에 고칠 버그는 없다.** 이건 "깨진 ZIP을 webhwp 뷰어가 어떻게 처리하는가"의 문제(한컴독스 측 동작).

## 정확한 재현 명령

1) 먼저 claw-hwp `create.js`로 **정상** .hwpx 생성 (6961 bytes, 정상 렌더 확인됨):
```bash
echo '{"path":"t4_simple.hwpx","operations":[
  {"type":"setup_document","page_size":"a4"},
  {"type":"append_heading","level":1,"text":"테스트4 HWPX 문서"},
  {"type":"append_paragraph","text":"이건 hwpx 포맷 테스트입니다. ANCHOR_T4 표식 포함."}
]}' | node <claw-hwp>/scripts/create.js
# → {"status":"success","bytes_written":6812,...}  (정상)
```

2) 그 정상 파일을 **앞 2000바이트만 남기고 절단** (← 여기서 손상 발생):
```bash
head -c 2000 testfiles/t4_simple.hwpx > testfiles/broken_truncated.hwpx
# 6961바이트 ZIP → 2000바이트 짜리 '반쪽 ZIP'. central directory 등 뒷부분 소실.
```

3) 캡처 시도:
```bash
node hancom.js capture --file .../broken_truncated.hwpx --page 1
# → 업로드는 됨. 한컴독스가 '열기 실패 다이얼로그'를 안 띄우고
#   ZIP 원시 바이트(mimetype/Contents/header.xml/PK 문자열 + 깨진 글자)를 본문처럼 렌더.
```

## 왜 garbage로 보이나
- `.hwpx`는 ZIP 컨테이너. 앞 2000바이트만 남기면 **유효하지 않은 ZIP**(끝부분/central directory 손실).
- 한컴독스 업로더는 이 파일을 받아줬고, webhwp 뷰어가 **유효 hwpx로 파싱 실패 → 폴백으로 원시 내용을 텍스트로 표시**한 것으로 보임.
- 즉 "열 수 없습니다" 다이얼로그(모드 ①)도 아니고, 정상 문서도 아닌 **중간 상태**. 캡처 도구 입장에선 "성공"으로 통과해 버린다.

## 두 손상 모드 비교 (도구 관점)
| 모드 | 만든 법 | 한컴독스 반응 | 캡처도구 |
|---|---|---|---|
| ① cannot_open | 실제 round-trip 손상 (예: `known_broken_sheetjs_roundtrip.hwp`) | "문서를 열 수 없습니다" 다이얼로그 | ✅ 감지 → `cannot_open` |
| ② 조용한 garbage | **유효 파일을 `head -c`로 절단** (이번 케이스, 인위적) | 다이얼로그 없이 원시 바이트 렌더 | ❌ "성공"처럼 통과 |

## 디버그팀이 볼 포인트
- claw-hwp 버그 아님(스킬 출력은 정상). **절단이 원인.**
- 고치려면 캡처 도구(hancom.js) 쪽에서 ②번을 잡아야 함: 렌더된 본문 텍스트에 ZIP 내부 마커(`mimetypeapplication/hwp+zip`, `Contents/`, `PK`)가 섞여 있으면 "손상 의심" 플래그. (현재 미구현)
- 혹은 업로드 전 로컬에서 파일 시그니처/ZIP 유효성 선검사(`PK\x03\x04` 매직 + central directory 존재 여부)로 거르는 것도 가능.
