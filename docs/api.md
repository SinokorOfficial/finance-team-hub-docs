# API

사내 서버판이 가진 경로 전부와, 각 경로에 필요한 권한·본문·응답을 정리한 페이지입니다.
모든 경로는 `server/app.py` 한 파일에 있고, 화면은 연결 층(`server/static/shim.js`)을 통해서만 호출합니다.

- 인증은 **세션 쿠키(`hub_sid`) 하나뿐**입니다. API 토큰·API 키는 없습니다. → [인증](auth.md)
- 응답은 JSON입니다(엑셀 내보내기·첨부 내려받기·화면 파일은 예외).
- 문서를 저장·삭제하면 서버가 `audit`에 기록하고, 접속 중인 모든 화면에 SSE로 알립니다.
- 권한 칸의 '규칙'은 역할·담당자·★ 여부에 따른 재검사를 뜻합니다. → [역할·권한](rbac.md)

## 1. health

| 메서드 · 경로 | 권한 | 하는 일 |
|---|---|---|
| `GET /api/health` | 없음 | 서버 생존 확인. `{ok:true, server, time}`. 배포·감시·예약 작업이 이 경로를 씁니다 |

## 2. auth

| 메서드 · 경로 | 권한 | 하는 일 | 본문 / 응답 |
|---|---|---|---|
| `GET /api/auth/admins` | 없음 | 로그인 화면에 보여 줄 관리자 이름 목록 | → `{names:[…]}` (승인된 관리자만) |
| `GET /api/auth/me` | 로그인 | 세션 복원(내 계정 정보) | → `{account}`. 비밀번호 칸 제외. 세션이 없으면 401 |
| `POST /api/auth/login` | 없음 | 로그인 · 세션 쿠키 발급 | `{email, pw}` → `{ok:true, account}` + `Set-Cookie` / `{ok:false, code}` (`bad`·`pending`·`blocked`) |
| `POST /api/auth/signup` | 없음 | 가입 신청(`status = pending`) | `{email, name, pw}` → `{ok:true}` / `{ok:false, code}` (`email`·`name`·`pw`·`exists`·`exists_pending`) |
| `POST /api/auth/logout` | 로그인 | 세션 줄 삭제 + 쿠키 삭제 | → `{ok:true}` |
| `POST /api/auth/password` | 로그인 | 내 비밀번호 변경 | `{cur, nw}` → `{ok:true}` / `{ok:false, code}` (`short`·`bad_cur`) |
| `POST /api/auth/reset` | 관리자 | 다른 계정 비밀번호 초기화 | `{email}` → `{ok:true, tmp}`. 임시 비밀번호는 이 응답에서 한 번만 나옵니다. 계정이 없으면 404 |

!!! note "로그인 실패는 400·401이 아닙니다"
    로그인·가입·비밀번호 변경의 실패는 HTTP 200에 `{ok:false, code:"…"}`로 돌아옵니다.
    화면이 상황별 안내 문구를 고르기 위한 것입니다. 관리자 전용 초기화만 권한 위반 시 403입니다.

## 3. db — 문서 저장소

경로의 `{coll}`은 `projects`·`events`·`accounts`·`meta`·`links` 다섯 개만 허용하고, 그 밖의 이름은 404입니다.
문서 구조는 [데이터 모델](data-model.md)을 보세요.

| 메서드 · 경로 | 권한 | 하는 일 | 본문 / 응답 |
|---|---|---|---|
| `GET /api/db/{coll}` | 로그인 | 컬렉션 전체 읽기 | → `{docs:[{id, data, version, updatedAt}]}` |
| `GET /api/db/{coll}/{id}` | 로그인 | 문서 1건 읽기 | → `{exists:true, id, data, version, updatedAt}`. 없으면 `{exists:false, id}`(200), 볼 수 없는 문서면 403 |
| `POST /api/db/{coll}` | 로그인 + 규칙 | 문서 생성(서버가 id 발급) | 본문 = 문서 전체 → `{id}` |
| `PUT /api/db/{coll}/{id}` | 로그인 + 규칙 | 문서 통째로 교체 | 본문 = 문서 전체 → `{ok:true, version}` |
| `PATCH /api/db/{coll}/{id}` | 로그인 + 규칙 | 준 칸만 병합 | 본문 = 바꿀 칸. 값이 `null`인 칸은 지웁니다 → `{ok:true, version}`. 문서가 없으면 404 |
| `DELETE /api/db/{coll}/{id}` | 관리자 · 본인 초안 · 담당자 직접 과제 | 문서 삭제 | → `{ok:true}` |

### 3.1 목록 필터

`GET /api/db/{coll}` 은 쿼리 3개로 걸러 낼 수 있습니다.

| 쿼리 | 뜻 |
|---|---|
| `field` | 비교할 필드 이름. 비어 있으면 필터하지 않습니다 |
| `op` | `==` 또는 `eq`(같음), `!=` 또는 `ne`(다름) |
| `value` | 비교값. JSON으로 읽어 보고, 실패하면 문자열 그대로 씁니다 |

필터는 **읽기 권한으로 가공한 뒤**에 걸립니다. 그래서 일반 계정이 `accounts`를 필터하면 공개 칸만 대상이 됩니다.
화면에서 담당자 계정 목록을 만들 때 `accounts` + `status == approved` 필터를 씁니다.

### 3.2 읽기 권한 가공

| 컬렉션 | 관리자 | 본인 문서 | 그 외 |
|---|---|---|---|
| `accounts` | 전체(비밀번호 칸 제외) | 전체(비밀번호 칸 제외) | 승인된 계정만 `email`·`name`·`role`·`status`·`company` |
| 나머지 4개 | 전체 | 전체 | 전체(초안 숨김은 화면이 처리) |

## 4. lock — 편집 잠금

| 메서드 · 경로 | 권한 | 하는 일 | 본문 / 응답 |
|---|---|---|---|
| `POST /api/lock/{key}` | 로그인 | 잠금 획득 | `{holder?, ttlMs?}`(기본 보유자 = 내 계정, 기본 60,000ms) → `{acquired:true|false}` |
| `DELETE /api/lock/{key}` | 로그인 | 잠금 해제 | → `{ok:true}` |

이미 다른 보유자가 쥐고 있고 아직 만료되지 않았으면 `{acquired:false}`가 옵니다.
만료된 잠금은 다시 획득할 수 있어서, 프로그램이 중간에 죽어도 영구히 잠기지 않습니다.
초기 데이터 적재(`meta/seedlock`)처럼 두 사람이 같은 작업을 동시에 하면 안 되는 곳에 씁니다.

## 5. stream — 실시간 반영(SSE)

| 메서드 · 경로 | 권한 | 하는 일 |
|---|---|---|
| `GET /api/stream` | 로그인 | 문서가 바뀌면 접속 중인 모든 화면에 알립니다(`text/event-stream`) |

- 접속하면 먼저 `retry: 3000` 과 `event: hello` 를 보냅니다. 끊기면 브라우저가 3초 뒤 스스로 다시 붙습니다.
- 문서 1건이 바뀔 때 보내는 내용: `{coll, id, data, version, updatedAt}`. 삭제는 `data`가 `null`입니다.
- `accounts` 변경분은 **받는 사람의 권한에 맞게 가공**해서 보내고, 볼 수 없는 사람에게는 보내지 않습니다.
- 20초 동안 보낼 것이 없으면 주석 한 줄(`: keepalive`)을 보내 연결을 유지합니다.
- 응답 헤더에 `Cache-Control: no-cache`, `X-Accel-Buffering: no`를 붙여 중간에서 버퍼링되지 않게 합니다.
- 재접속에 성공하면 연결 층이 구독 중인 컬렉션을 다시 한 번 통째로 읽어, 끊긴 사이의 변경을 메꿉니다.

## 6. export — 엑셀 내보내기

| 메서드 · 경로 | 권한 | 하는 일 |
|---|---|---|
| `GET /api/export/status.xlsx` | 로그인 | 팀 현황 엑셀(취합 양식 서식 그대로) |
| `GET /api/export/group-fill.xlsx` | 로그인 | 그룹 대시보드 '등록된 과제 채우기' 양식 |

- 둘 다 `Content-Disposition: attachment` + `Cache-Control: no-store`로 나갑니다. 파일 이름은 UTF-8로 인코딩합니다.
- 그룹 양식은 변환 중 생긴 예외(선택지 불일치 등)를 응답 헤더 **`X-Export-Info`** 에 URL 인코딩한 JSON으로 함께 돌려줍니다. 화면이 이 값을 읽어 안내로 보여 줍니다.
- 만드는 코드는 `server/export_xlsx.py`·`server/export_group.py`입니다. → [엑셀 양식·업로드·내보내기](excel.md)

## 7. history — 수정 이력

| 메서드 · 경로 | 권한 | 하는 일 |
|---|---|---|
| `GET /api/history/{coll}/{id}?limit=30` | 로그인 | 과제가 언제·누구에 의해·어떻게 바뀌었는지 |

- **`projects`만** 기록합니다. 다른 컬렉션으로 물으면 빈 목록(`{items:[]}`)이 옵니다.
- 응답: `{items:[{ts, who, action, changes:[{k, a, b}]}]}` — `k`는 필드 키, `a`는 이전 값, `b`는 새 값입니다. 항목 이름은 화면이 라벨로 바꿔 보여 줍니다.
- `limit` 기본 30, 최대 100입니다. 로그인한 사람은 모두 볼 수 있습니다. → [ADR-0010](adr/0010-server-side-history.md)

## 8. files — 첨부파일

| 메서드 · 경로 | 권한 | 하는 일 |
|---|---|---|
| `POST /api/files?name=파일이름` | 조회 전용 제외 | 파일 올리기. **본문이 파일 자체**(폼 전송이 아닙니다), 이름은 쿼리로 전달 |
| `GET /api/files/{fid}` | 로그인 | 내려받기 |
| `DELETE /api/files/{fid}` | 올린 사람 · 관리자 | 파일 삭제 |

- 한 개 20MB까지입니다. `Content-Length`로 먼저 걸러 내고, 실제 본문 길이도 다시 확인합니다. 넘으면 413입니다.
- 응답: `{id, name, size, by, at}`. 파일 이름에서 경로·특수문자를 지우고 120자로 자릅니다.
- 내려받기는 **항상 첨부(다운로드)** 로 나갑니다. HTML 같은 파일이 브라우저에서 실행되지 않게 하려는 것입니다.
- `{fid}`가 16자리 16진수가 아니거나 파일이 없으면 404입니다. 삭제는 이미 없는 파일이어도 `{ok:true}`를 돌려줍니다.

## 9. 화면 파일

| 메서드 · 경로 | 권한 | 하는 일 |
|---|---|---|
| `GET /` | 없음(화면 안에서 로그인 게이트) | `static/index.html`을 `Cache-Control: no-cache`로 내려줍니다 |
| `GET /*` | 없음 | 그 밖의 정적 파일(`static/` 폴더 통째로 연결) |

`static/index.html`은 `hub.html` + `shim.js`를 합쳐 만든 빌드 산출물입니다. 직접 고치지 않습니다. → [모듈 맵](modules.md)

## 10. 오류 코드 규칙

| 코드 | 언제 | 응답 |
|---|---|---|
| 400 | 본문이 JSON이 아님 / JSON 객체가 아님 / 빈 파일 업로드 | `{detail:"…"}` |
| 401 | 세션 쿠키가 없거나 만료됨, 계정이 승인 상태가 아님 | `{detail:"로그인이 필요합니다"}` |
| 403 | 역할·담당자·승인 규칙 위반, 볼 수 없는 문서 | `{detail:"…"}` — 사유가 문구로 들어옵니다 |
| 404 | 없는 컬렉션, 병합 대상 문서 없음, 없는 계정·첨부파일 | `{detail:"…"}` |
| 413 | 첨부 20MB 초과 | `{detail:"파일은 20MB까지 올릴 수 있습니다"}` |

연결 층(`shim.js`)은 401을 `unauthenticated` 오류로 바꿔 화면에 알리고(로그인 창으로 돌아감),
그 밖의 오류는 서버가 준 `detail` 문구를 그대로 화면에 띄웁니다.

---

## 관련 문서

- [데이터 모델·스키마](data-model.md) — 이 경로들이 다루는 문서 구조
- [역할·권한 (RBAC)](rbac.md) — 저장·삭제 재검사 규칙
- [인증 (로그인·가입·승인)](auth.md) — 세션 쿠키
- [프론트엔드 아키텍처](frontend.md) — 화면이 API를 부르는 방식
