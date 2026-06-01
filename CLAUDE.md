# CLAUDE.md

이 파일은 Claude Code / Codex 같은 AI 에이전트가 이 저장소의 이슈를 구현할 때
따라야 할 **프로젝트 비전·제약·설계 원칙**을 정리한 길잡이입니다.
(사용자가 틈날 때 이슈를 적어두고, 나중에 집에서 AI에게 구현을 맡기는 워크플로우를 전제로 합니다.)

## 프로젝트 목표

폐쇄망 + 타 조직이 띄워준 Jupyter 환경이라는 **제약 위에서 최대한의 효율**을 내기 위한
Jupyter Lab 유틸리티 모음. Jupyter 기능 / magic word / extension 을 적극 활용합니다.
잔 스니펫보다는 **매직 커맨드·확장 단위의 굵직한 유틸리티**를 지향합니다.

## 환경 제약 (모든 구현의 전제)

- 🚫 **폐쇄망**: 인터넷에서 `pip install`로 새 패키지를 받기 어렵다. 표준 라이브러리 + 이미 설치된 것 위주.
- 🔒 **권한 제약**: 관리자/sudo 권한 없음. 설정은 보통 홈 디렉터리(`~/.ipython`, `~/.jupyter`) 한정.
- 🔁 **서버 재시작 불가**: Jupyter 서버는 타 조직이 띄워준다. 커널 재시작 정도만 가능.
- 👥 **공유 환경**: 여러 사람이 함께 쓰므로 전역 변경은 지양. 네트워크/쿼터/레이트리밋 부담 고려.

구현 형태 선호 순서(설치 부담이 적은 순): **스니펫 → 매직 커맨드 → 함수/모듈 → (가능하면) 확장**.

## 핵심 설계 원칙

### 1. 백엔드 비종속성 (Backend portability) — ⭐ 최우선
외부 스토리지/데이터 접근(쿼리·파일 read/write/ls 등)은 **특정 백엔드에 묶이면 안 된다.**
사내 인프라 변경이나 이직 등으로 백엔드가 바뀌어도(예: AWS S3 → GCP GCS) 사용자 코드는 그대로여야 한다.

- 사용자 코드는 **추상 인터페이스**만 호출한다. 예: `run_query(sql) -> DataFrame`, `Storage.ls(path)`, `Storage.read/write(path)`.
- 백엔드(S3 / GCS / 로컬FS / DB 등)는 **교체 가능한 어댑터(plugin)**로 구현한다.
- 백엔드 선택은 **설정/환경변수/경로 스킴**(`s3://`, `gs://`, `file://`)으로 주입한다.
- 자격증명은 **환경변수/설정으로만 주입**하고, 코드·로그에 절대 노출하지 않는다.
- 새 백엔드 추가가 기존 코드 수정 없이 가능해야 한다(개방-폐쇄 원칙).
- 특정 백엔드 SDK가 없는 환경에서도 **import 단계에서 죽지 않도록 lazy(선택적) import**로 처리한다.
- 여러 기능(쿼리·스토리지·자동완성 등)은 **동일한 어댑터/커넥션 계층을 공유**한다.

### 2. 오프라인 반입 (Offline delivery) — prebuilt wheel
폐쇄망 반입은 두 경로를 함께 지원한다.

1. **startup 스크립트 떨구기** — 가장 가벼운 fallback. `~/.ipython/profile_default/startup/`에 `.py`.
2. **prebuilt wheel + 오프라인 번들** — ⭐ 권장 배포 모델.
   - 이 repo를 설치형 패키지(wheel)로 빌드(`python -m build`).
   - `pip download`/`pip wheel`로 **타깃과 동일한 OS/Python/아키텍처(manylinux 등)** 기준 의존성 wheel을 전부 수집.
   - **prebuilt JupyterLab 확장 wheel**도 함께 담는다 → 폐쇄망 안에서 npm 빌드 없이 labextension 설치 가능.
   - 폐쇄망: `pip install --no-index --find-links=<번들폴더> jupyter_utils[extras]`.
   - 무거운 백엔드 SDK(boto3/google-cloud-storage 등)는 **extras**(`[s3]`, `[gcs]`)로 분리해 필요한 것만 번들에 포함.

### 3. 확장(extension)의 현실적 구분
- **IPython 확장**(`%load_ext`, 순수 파이썬): npm 빌드 불필요 → 폐쇄망에서 실현 가능. 매직 다수가 여기 속함.
- **JupyterLab 확장**(JS/TS, npm 빌드): **prebuilt wheel 반입**이 가능할 때만 현실적. 아니면 server-extension + custom CSS/JS로 우회.

## 패키지/구현 컨벤션 (지향점)

- 설치형 파이썬 패키지(`pyproject.toml`, `src/jupyter_utils/` 레이아웃).
- 매직은 entry point / `load_ext`로 등록. 무설치 fallback으로 startup 스크립트도 허용.
- 선택적 의존성은 `extras` + lazy import. 코어는 표준 라이브러리만으로 import 성공해야 한다.
- 결과 표현은 `pandas.DataFrame`을 기본으로 하되, 없으면 list/dict로 폴백.
- 친절한 에러 메시지(빈 입력, 잘못된 경로/변수명, 자격증명 미설정 등).
- 모든 기능에 사용법 예제(도큐스트링/예제 노트북) 포함.

## 굵직한 단위 로드맵 (설계 논의 결과)

| 순서 | 단위 | 설명 |
| --- | --- | --- |
| 0 | 패키지 스캐폴드 | `pyproject.toml`, 매직 entry point, extras 정의 |
| 1 | 오프라인 번들 빌드 파이프라인 | build → pip download → 매니페스트 → 아카이브 |
| 2 | 공통 백엔드 어댑터 계층 | S3/GCS/로컬 어댑터. 아래 기능들의 공통 토대 |
| 2a | `%%sql <var>` (M1) | line에 변수명, cell에 SQL → 결과를 그 변수에 바인딩 (**issue #2**) |
| 2b | `%ls` 스토리지 리스팅 (M3) | 라인 매직 + 함수 + (선택) CLI, Tab 경로 자동완성 (**issue #3**) |
| 2c | 커넥션/백엔드 매니저 (E5) | 활성 백엔드 전환·자격증명 안전 주입 |
| 3 | `%%cache` (M2) | 셀 결과 디스크 캐싱 — 재시작 불가 제약 대응 |
| 4 | prebuilt 확장 편입(E3 등), `%%bg`(M4), `%doctor`(M5), 노트북 자동 백업(E2) | 여력 시 |

기타 후보 매직: `%notify`(셀 완료 알림), `%%retry`, `%env_check`.

## 폼팩터 가이드 (노트북 적합도)

- **라인/셀 매직**: Jupyter에 가장 native. 인라인·빠름, HTML 렌더, **Tab 자동완성 가능**(IPython `complete_command` 훅). 탐색·인터랙션 1순위.
- **파이썬 함수**: 결과를 다음 셀에서 가공·재사용. 매직과 **같은 코어 공유**. 단 문자열 인자 Tab 완성은 어려움(Jedi가 코드만 완성).
- **CLI**: 자동화/터미널용이나, 공유·제약 환경에선 터미널 접근이 막혀 있을 수 있어 **보너스**(entry_point로 저비용 추가).
- 결론: 노트북이면 **매직 + 함수(코어 공유)를 1순위**, CLI는 선택. Tab 자동완성이 필요하면 매직 형태로.

## 이슈 작성 / 구현 규칙

- 이슈는 `.github/ISSUE_TEMPLATE/`의 템플릿(유틸리티·매직·확장·백엔드·패키징·스니펫·버그·아이디어)을 사용한다.
- 모든 구현 이슈에는 **환경 제약 체크 + 의존성/반입 계획 + 완료 조건(Acceptance Criteria)**이 담긴다.
- AI는 구현 시 위 제약·설계 원칙을 어기지 않는지 항상 확인하고, 어겨야 한다면 먼저 이슈에 질문/제안을 남긴다.
- 새 의존성을 추가할 때는 반드시 extras 분리 + lazy import + 오프라인 반입 가능성을 함께 고려한다.

## 참고 (관련 이슈)
- #2 `%%sql` 매직 — line 변수명 + cell SQL → 결과 바인딩
- #3 오브젝트 스토리지 리스팅 `%ls` + Tab 자동완성
