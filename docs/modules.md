# 모듈 맵

저장소의 폴더·파일이 각각 무슨 일을 하는지 정리합니다. 표기는 실제 파일명 그대로이고, 마지막 절에 **배포되는 것 / 배포되지 않는 것 / 데이터(git 밖)** 를 구분해 두었습니다.

저장소는 `SinokorOfficial/finance-team-hub`(비공개)이며, 루트에 화면 소스와 관리자 개인 도구, `server/` 에 서버 프로그램과 배포·운영 스크립트, `.github/workflows/` 에 GitHub Actions, `usb/` 에 서버 PC 최초 설치 패키지 원본이 있습니다. **서버 PC로 복사되는 것은 `server/` 폴더뿐입니다.**

## 루트 — 화면과 관리자 도구

| 파일 | 역할 |
|---|---|
| `hub.html` | 화면 소스(단일 파일, 약 210KB). Artifact판·서버판 공용. **화면 수정은 항상 이 파일만** 고칩니다 |
| `verify_saving_parity.py` | 효과 지표가 그룹 AX 대시보드와 같은 기준인지 확인하는 독립 검증기. 허브 코드를 믿지 않고 대시보드 쪽에서 선택지·계산식을 다시 받아 대조합니다(선택지·공식·입력칸·왕복·양식값·필수 규칙 6종) |
| `make_plan_xlsx.py` · `make_collect_xlsx.py` · `make_collect_prefilled_xlsx.py` · `make_collect_merged_xlsx.py` · `make_collect_result_xlsx.py` | 구축 계획 엑셀과 과제 취합 양식(생성·기초 반영·병합·결과 정리) — 초기 취합 단계에서 쓴 도구 |
| `apply_revised_xlsx.py` | 사용자가 직접 고친 취합 결과 엑셀을 읽어 DB 반영용 쓰기 목록을 만드는 도구 |
| `export_status_xlsx.py` | 배포 양식을 Excel COM으로 채우는 옛 현황 엑셀 생성기. 서버판에서는 `server/export_xlsx.py`(openpyxl)가 대신합니다 |
| `make_followup_mail.py` | 담당자별 필수 항목 입력 요청 메일 초안을 Outlook 임시보관함에 저장 |
| `.gitignore` | 데이터·개인정보 제외 규칙(맨 아래 "데이터 — git 밖" 절 참조) |
| `.gitattributes` | 설치용 `.bat` 은 줄바꿈 변환 금지(cp949 + CRLF 유지), `.ps1` 은 CRLF 고정 |

!!! warning "루트의 도구는 서버로 배포되지 않습니다"
    `make_*` · `export_status_xlsx.py` · `apply_revised_xlsx.py` 는 관리자 PC에서 손으로 돌리는 개인 도구입니다. 배포는 `server` 폴더만 복사하므로 서버에 존재하지 않고, Excel·Outlook COM을 쓰는 것이 섞여 있어 서버 PC에서는 돌지도 않습니다.

## `server/` — 서버 프로그램

| 파일 | 역할 |
|---|---|
| `app.py` | 서버 본체(FastAPI 단일 파일). 로그인·세션, 문서 저장소 API, 권한·승인 규칙 재검사, SSE 실시간 알림, 엑셀 내보내기, 첨부파일, 감사 기록·수정 이력, 기동 시 데이터 변환 |
| `requirements.txt` | 파이썬 부품 목록 — `fastapi` · `uvicorn` · `openpyxl` 세 줄 |
| `README.md` | 서버판 설치·배포·운영 안내(비전공자용, 사내 실값 포함) |
| `build_static.py` | `hub.html` → `static/index.html` 변환 |
| `export_xlsx.py` | 팀 현황 엑셀 생성(openpyxl). 취합 양식 서식을 그대로 유지하고 승인된 과제만 채움 |
| `export_group.py` | 그룹 AX 대시보드 "등록된 과제 채우기" 양식 생성 |
| `rebuild_status_template.py` | 현황 양식(`templates/status_template.xlsx`)의 절감 열 구조를 그룹 기준으로 재편하는 스크립트. 이미 재편된 양식이면 아무것도 하지 않습니다 |
| `import_snapshot.py` | Artifact DB 스냅샷(JSON 폴더) → `hub.db` 이관. 초기 이관용이며 실행 전 `hub.db` 를 자동 백업합니다 |
| `check_data.py` | 데이터 점검(읽기 전용) — 계정 가입·승인, 담당자 계정 연결, 필수 항목 충족 현황. DB를 read-only로만 열고 이메일은 가려서 출력합니다 |

### `server/static/` — 브라우저에 내려가는 파일

| 파일 | 역할 |
|---|---|
| `index.html` | **빌드 산출물.** `build_static.py` 가 `hub.html` 에서 만들어 냅니다. 직접 고치면 다음 빌드에서 덮어써지고 CI 검사에서도 막힙니다 |
| `shim.js` | 연결 층. 화면이 쓰는 저장소·로그인 인터페이스를 서버 API 호출로 바꿉니다([아키텍처 개요](architecture.md) 참조) |
| `vendor/xlsx.full.min.js` | 엑셀 읽기용 SheetJS 0.18.5. 서버에 함께 넣어 두어 인터넷 없이 동작합니다 |

### `server/templates/` — 엑셀 양식 원본

| 파일 | 역할 |
|---|---|
| `status_template.xlsx` | 팀 현황 엑셀 양식(취합 양식에서 데이터를 비운 판). 드롭다운 목록 시트를 포함합니다 |
| `group_fill_template.xlsx` | 그룹 AX 채우기 양식(기획팀 배포본에서 데이터 행만 비운 판) |

`.gitignore` 는 `*.xlsx` 를 제외하지만 이 두 파일만 예외로 추적합니다(`!server/templates/*.xlsx`).

### `server/` — 배포·실행 스크립트

| 파일 | 역할 |
|---|---|
| `deploy.ps1` | **현재 방식.** 서버 PC의 self-hosted runner가 실행하는 배포 스크립트. 절차는 [배포](deployment.md) 참조 |
| `run.bat` · `install.bat` | 로컬에서 서버 띄우기(`uvicorn app:app --host 0.0.0.0 --port 8002`) / 파이썬 부품 설치(`pip install -r requirements.txt`) |
| `deploy.bat` | 공유 폴더로 코드를 복사하는 **예비 수단**(옛 방식). 대상 경로를 파일 맨 위에서 직접 맞춰야 합니다 |
| `update.bat` | 서버 폴더가 git clone인 경우에 쓰던 **옛 갱신 방식**(`git pull` 후 서버 재시작) |

`deploy.bat` 과 `update.bat` 은 러너 기반 자동 배포로 바뀌면서 예비 수단으로만 남겨 두었습니다. 러너가 죽었을 때의 정식 대안은 서버 PC에서 `deploy.ps1` 을 직접 실행하는 것입니다.

### `server/` — 운영 스크립트

| 파일 | 역할 |
|---|---|
| `watchdog.ps1` | 서버 감시. 5분마다 `/api/health` 를 확인하고 응답이 없으면 포트를 잡고 있는 좀비 프로세스를 정리한 뒤 서버 예약 작업을 다시 실행합니다. 정상일 때는 로그를 남기지 않고, 죽었을 때·되살렸을 때만 기록합니다 |
| `backup_uploads.ps1` | 첨부파일 백업. 매일 `uploads` 를 백업 폴더로 미러하고 날짜별 zip으로 보관(14일)합니다. DB 백업 작업이 `hub.db` 만 복사하는 빈틈을 메웁니다 |
| `diag.ps1` | 서버 PC 원격 진단(읽기 전용). 상태·로그만 출력하며 서버를 건드리지 않습니다. 원격 접속이 막힌 서버 PC를 러너 경유로 들여다보는 용도 |
| `tools/register_tasks.ps1` | 감시·첨부 백업 예약 작업을 등록. 로그인 계정 + LogonType S4U + 최고 권한으로 XML 등록하고, 등록 직후 두 스크립트를 시험 실행해 결과를 보여 줍니다 |
| `tools/작업등록.bat` | 위 스크립트를 관리자 권한으로 승격해 실행하는 더블클릭용 배치 |

!!! warning "예약 작업 등록은 배포로 자동화되지 않습니다"
    스크립트 파일 자체는 배포(코드 복사)로 갱신되지만, **예약 작업 등록은 서버 PC에서 `tools/작업등록.bat` 을 한 번 실행해야** 반영됩니다. 등록 계정과 로그온 방식을 이렇게 정한 이유는 [ADR-0006](adr/0006-scheduled-tasks-user-s4u.md)에 있습니다.

## `.github/workflows/` — GitHub Actions

| 파일 | 언제 | 하는 일 |
|---|---|---|
| `deploy.yml` | `main` 에 push · PR · 수동 | ① GitHub 쪽에서 검사(`app.py` 문법, `index.html` 최신 여부) → ② 통과하면 서버 PC 러너가 `deploy.ps1` 실행. PR에서는 검사만 하고 배포하지 않습니다 |
| `diag.yml` | 수동 실행만 | 서버 PC에서 `diag.ps1` 실행(읽기 전용 진단) |
| `data-check.yml` | 수동 실행만 | 서버 PC에서 `check_data.py` 실행(읽기 전용 데이터 점검). 아무것도 고치지 않습니다 |

세 워크플로우 모두 사내 서버 PC 러너를 `[self-hosted, windows, hub]` 라벨로 지정합니다.

## `usb/` — 서버 PC 최초 설치 패키지 원본

| 파일 | 역할 |
|---|---|
| `설치.bat.utf8.src` | 설치 본체 원본 — 파이썬 확인·설치, 코드 복사, 부품 오프라인 설치, 방화벽, 예약 작업 2개 등록, 서버 시작·응답 확인 |
| `러너연결.bat.utf8.src` | GitHub self-hosted runner 설치·등록(자동 배포용) |
| `제거.bat.utf8.src` | 되돌리기 — 예약 작업·방화벽·폴더 제거(데이터는 남김) |
| `tools/task_server.xml` · `tools/task_backup.xml` · `tools/예약작업_재등록.bat.utf8.src` | 예약 작업 정의 XML(경로·계정 식별자는 설치 때 치환)과 예약 작업만 다시 등록하는 배치 |
| `make_usb.md` · `읽어보세요.txt` | USB 패키지 재생성 절차와 재발 금지 결함 목록 / 설치 담당자용 안내 |
| `CLAUDE_인수인계.md` · `CLAUDE_설치결과.md` | 서버 PC 설치 기록·인수인계 메모(**사내 실값 포함** — 공개 문서로 옮기지 않습니다) |

`.utf8.src` 는 "UTF-8 원본"이라는 뜻입니다. 설치용 배치는 cmd가 한글을 읽도록 cp949 + CRLF여야 하는데 그 상태로 저장소에 두면 깨지기 쉬워서, 저장소에는 UTF-8 원본을 두고 USB 패키지를 만들 때 변환합니다(절차는 `usb/make_usb.md`).

## 배포되는 것 / 안 되는 것 / 데이터

### 서버 PC로 배포되는 것

`server` 폴더 전체입니다. `deploy.ps1` 이 이 폴더를 서버 PC의 설치 폴더로 `robocopy /E`(삭제 없이 덧쓰기) 복사합니다. 따라서 `app.py` · `static/` · `templates/` · `tools/` · 운영 스크립트 · 각종 배치가 모두 갱신됩니다.

### 배포되지 않는 것

| 대상 | 이유 |
|---|---|
| `hub.html` | 서버는 빌드 산출물 `server/static/index.html` 만 씁니다. 원본은 개발 PC에만 필요 |
| 루트의 `make_*.py` · `export_status_xlsx.py` · `apply_revised_xlsx.py` · `verify_saving_parity.py` | 관리자 개인 도구. 복사 범위(`server` 폴더) 밖 |
| `usb/` | 최초 설치용. 서버 PC에는 설치할 때 USB로 한 번 들어갑니다 |
| `.github/workflows/` | GitHub이 읽는 파일(러너 작업 공간에는 내려오지만 서버 폴더로는 복사되지 않음) |
| `start_hub.bat` · `stop_hub.bat` · `backup_db.bat` | 설치할 때 서버 PC에서 실제 경로로 만들어진 파일. 배포 복사에서 **명시적으로 제외**합니다 |

### 데이터 — git 밖, 서버 PC에만

| 대상 | 설명 |
|---|---|
| `server/hub.db` · `server/hub.db.bak-*` | 실제 데이터와 `import_snapshot.py` 가 남긴 자동 백업. 배포 복사에서 제외되어 코드를 갱신해도 보존됩니다 |
| `server/uploads/` | 첨부파일. 복사 범위 밖이라 서버 PC에만 존재합니다 |
| `dbsnapshot*/` · `dbwrites*/` · `server_data_*.json` | 개발 PC에 내려받은 DB 스냅샷·쓰기 목록 |
| `*.xlsx` · `*.csv` · `*.pdf` · `*.msg` | 실값이 든 산출물 일체(`server/templates/*.xlsx` 만 예외) |
| `hub_v*_backup.html` | 옛 화면 백업본(이력은 git이 대신함) |
| `docs/` 폴더의 설명서 `.docx` | 사내 실값이 든 원본 문서로, 커밋하지 않고 로컬에만 둡니다 |

`.gitignore` 가 위 항목을 막고 있습니다. 실값 자료가 필요한 작업(스냅샷 이관, 메일 초안 생성 등)은 개발 PC 또는 서버 PC에서 직접 파일을 다루고, 그 결과물도 커밋하지 않습니다.

---

**관련 문서**

- [아키텍처 개요](architecture.md)
- [배포](deployment.md)
- [서버 PC 운영](operations.md)
- [엑셀 양식·업로드·내보내기](excel.md)
