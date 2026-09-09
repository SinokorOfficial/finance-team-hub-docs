# ADR-0002 · 화면 소스 한 벌, 연결 층만 교체

| 항목 | 내용 |
|---|---|
| 상태 | 채택 (2026-09-07) |
| 결정 | 화면은 `hub.html` 한 파일만 두고, 로그인과 저장소 연결만 어댑터(AUTH 층 · `shim.js`)로 갈아 끼워 Artifact판과 사내 서버판이 같은 소스를 씁니다. |

## 배경

[ADR-0001](0001-artifact-to-onprem.md) 로 서버판을 만들 때 화면 코드를 복사하면 두 벌이 됩니다. 고칠 때마다 양쪽에 같은 수정을 해야 하고, 한쪽만 고친 상태로 방치되면 어느 쪽이 맞는지 알 수 없습니다.

두 환경의 차이는 사실 두 군데뿐이었습니다. **로그인을 누가 검증하는가**(브라우저 안에서 해시 대조 vs 서버가 검증)와 **데이터를 어디에 저장하는가**(Artifact 런타임 `window.claude.use("db")` vs `/api/db/…`)입니다. 화면 그리기·권한 판정·엑셀 파서는 완전히 같습니다.

## 결정

그 두 군데만 갈아 끼울 수 있는 층으로 빼고, 화면 본문은 그 층만 호출합니다.

**로그인 — AUTH 층.** `hub.html` 안에 Artifact판용 구현 `ARTIFACT_AUTH`(계정 문서의 PBKDF2 해시를 브라우저에서 대조)를 두고 마지막에 고릅니다.

```js
const AUTH = window.HUB_AUTH || ARTIFACT_AUTH;
```

서버판에서는 연결 층이 `window.HUB_AUTH` 를 미리 넣어 두므로 `/api/auth/…` 가 검증합니다. 화면은 `AUTH.login`·`AUTH.signup`·`AUTH.restore` 같은 같은 이름만 부릅니다.

**데이터 — shim.js.** `server/static/shim.js` 가 Artifact 런타임과 같은 모양의 `window.claude.use("db")`·`window.claude.use("downloads")` 를 흉내 내어 `/api/…` 호출로 바꿉니다. 컬렉션 구독(`onSnapshot`)은 서버의 SSE 알림을 받아 해당 컬렉션만 다시 읽는 방식이고, 캐시·재접속도 이 파일이 담당합니다. 진짜 Artifact 안에서는 첫 줄에서 스스로 비활성화합니다.

**빌드 — build_static.py.** `hub.html` → `server/static/index.html` 변환입니다.

| 변환 | 이유 |
|---|---|
| SheetJS CDN 태그 → `/vendor/xlsx.full.min.js` + `/shim.js` 삽입 | 인터넷 없이 엑셀 업로드가 동작해야 하고, 연결 층이 본문보다 먼저 실행돼야 함 |
| `<!doctype>`·`<head>`·`<body>` 골격과 favicon 추가 | Artifact 런타임이 감싸 주던 부분을 직접 만듦 |
| `[hidden]{display:none!important}` 보강 | 없으면 숨긴 로그인 게이트가 투명하게 남아 클릭을 가로막음 |

## 결과 (장점 · 감수한 단점)

**장점**

- 화면 수정은 `hub.html` 한 곳에서 끝나고, 두 환경의 동작 차이가 생길 여지가 없습니다.
- 저장소 관련 코드를 한 줄도 고치지 않고 서버판으로 넘어왔습니다.
- 다른 백엔드로 옮길 때도 연결 층만 다시 쓰면 됩니다.

**감수한 단점**

- `static/index.html` 은 빌드 산출물이라 직접 고치면 다음 빌드에서 사라집니다. CI 가 "최신 `hub.html` 로 만들어진 것인지" 검사하므로, 빌드를 빠뜨리면 배포가 막힙니다.
- CDN 태그를 정규식으로 찾으므로 그 태그 모양을 바꾸면 빌드가 멈춥니다(조용히 틀리지는 않습니다).

## 되돌릴 때

Artifact판을 완전히 버리면 `ARTIFACT_AUTH` 블록과 `shim.js` 의 자기 비활성화 분기를 지우고 서버 전용으로 단순화할 수 있습니다. 다른 백엔드를 붙일 때는 `shim.js` 와 `HUB_AUTH` 두 개만 새로 쓰면 화면은 그대로 씁니다.

---

**관련 문서**

- [프론트엔드 아키텍처](../frontend.md)
- [ADR-0001 · Artifact판에서 사내 서버판으로](0001-artifact-to-onprem.md)
- [배포 (GitHub Actions → 사내 서버 PC)](../deployment.md)
