# Humanoid Daily 사이트

「휴머노이드·피지컬 AI 주요동향」 뉴스레터의 공개 아카이브와 내부 운영 화면.
빌드 도구가 없다. 파일을 열면 그대로 동작한다.

## 여는 방법

```bash
# 로컬 미리보기 (권장)
python3 -m http.server 8000
#   http://localhost:8000/           공개 홈
#   http://localhost:8000/internal/  내부 홈
```

`index.html`을 **더블클릭해서 열어도 동작한다.** 데이터를 `fetch()`가 아니라
`.js` 전역 할당으로 넣은 것이 그 때문이다 — `file://`에서 `fetch()`는 CORS로 막힌다.

## 구조

```
공개 (누구나)
  index.html        최신호 전문 + 최근 리포트
  newsletter.html   개별 호 (?d=2026-09-03)
  archive.html      항목 단위 검색
  reports.html      리포트 서가
  about.html        체계 소개
  data/             config.js · editions.js · reports.js

내부 (접근 제한)
  internal/index.html      내부 홈 — 오늘 먼저 볼 것
  internal/pool.html       후보풀·승인  ← 매일 쓰는 화면
  internal/shelf.html      레퍼런스 서가
  internal/open.html       미확정 사항(OPEN)
  internal/threads.html    스레드·교차검색·온톨로지
  internal/issues.html     룰 트리거 보드 · 이슈 후보
  internal/pipeline.html   리포트 파이프라인
  internal/system.html     체계도
  internal/data/           refs · opens · threads · candidates · issues · rules.js
```

## ⚠ 내부 데이터를 지키는 방법 — 읽고 나서 고쳐라

**정적 사이트에서 데이터 파일은 URL만 알면 누구나 받는다.**
페이지에 로그인을 걸어도 `internal/data/refs.js`를 직접 요청하면 그대로 나온다.

그래서 이 사이트는 두 가지를 지킨다.

1. 내부 데이터는 **전부 `internal/data/` 아래**에 둔다.
2. **공개 페이지는 `internal/` 경로를 절대 참조하지 않는다.**

이 둘이 지켜지면 호스팅에서 **`internal/*` 경로 하나에만** 접근 제한을 걸면 끝난다.

```
Cloudflare Pages + Zero Trust Access
  1. 저장소를 Pages 에 연결 (빌드 명령 없음, 출력 디렉터리 = 루트)
  2. Zero Trust → Access → Applications → 애플리케이션 추가
  3. 경로에 /internal/*  하나만 지정
  4. 정책 = 담당자 이메일 허용
```

**"공개 홈에도 후보풀을 보여주자"는 제안이 나오면 이 문단을 근거로 막아라.**
공개면이 `internal/` 파일을 하나라도 불러오는 순간 위 접근 제한이 무의미해지고,
아직 승인되지 않은 후보와 미확정 사항이 그대로 공개된다.

점검은 한 줄이면 된다.

```bash
grep -rn "internal/" index.html newsletter.html archive.html reports.html about.html data/
# 아무것도 안 나와야 한다
# (assets/ 는 공개·내부가 함께 쓰는 자산이라 internal-badge 클래스명이 들어 있다 — 경로가 아니다)
```

`scripts/check.py`도 이 조건을 검사한다.

## 매일 갱신

데이터 파일만 바꾼다. HTML을 손대는 날은 서식이 바뀌는 날뿐이다.

| 순서 | 무엇 |
|---|---|
| 1 | `data/editions.js` — 오늘 호를 **맨 앞에** 추가. `text`는 발송한 그대로 |
| 2 | `internal/data/refs.js` — 승인분 + 미승인 후보를 REF로 적재 |
| 3 | `internal/data/opens.js` — 확인 못 한 것 추가, 해소분은 `st:"해소"` (지우지 않는다) |
| 4 | `internal/data/threads.js` — 해당 줄기에 이벤트 추가 |
| 5 | `internal/data/candidates.js` — 다음 날 후보풀로 교체 |
| 6 | `python scripts/dday.py --root .` |
| 7 | `python scripts/rules.py --root .` |
| 8 | `python scripts/check.py --root .` — **통과해야 커밋한다** |

## 자주 깨지는 것

**`.js` 데이터 배열의 문자열에 줄바꿈을 넣지 마라.**
한국어 문장이 길어져도 한 줄로 이어 쓴다. 어기면 `Invalid or unexpected token`으로
**페이지 전체가 하얗게 뜬다.** 데이터가 조금 잘못되는 게 아니다.

**후행 쉼표와 홑따옴표를 쓰지 마라.** 브라우저는 통과시키지만 파이썬 스크립트가 전부 멈춘다.

**섹션 순서를 바꾸지 마라.** 순서는 1~7 고정이고, 승인분이 없는 섹션은
통째로 건너뛴다 — 지우는 것이 아니라 그날 없는 것이다.
'없음' 줄과 워치리스트는 발송본에 넣지 않는다(관리용 화면이 진다).
