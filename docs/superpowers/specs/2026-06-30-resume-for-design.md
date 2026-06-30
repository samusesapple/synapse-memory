# `resume-for` — 채용공고 URL → 회사 맞춤 이력서 초안

작성일: 2026-06-30
상태: 설계 승인 (구현 계획 작성 전)

## 1. 목적

채용공고 URL을 입력하면 그 공고를 크롤링해 직무·요구역량을 추출하고,
사용자의 vault 데이터(ProjectCard + Profile)를 그 회사/직무에 맞도록 다듬어
이력서 초안을 자동 생성한다.

기존 `persona draft-resume <company_id>` 는 vault에 **이미 존재하는 CompanyCard**
를 전제로 한다. `resume-for` 는 그 앞단을 채운다 — **URL → CompanyCard 생성/갱신 →
기존 `draft_resume` 재사용** 흐름으로, 입력 진입점만 "회사 ID"에서 "공고 URL"로
넓힌다.

## 2. 명령어 표면

| 레이어 | 표기 |
| --- | --- |
| 터미널 | `synapse-memory resume-for <job-url> [--lang ko\|en] [--text <file>] [--no-save-card] [--model <m>]` |
| Claude Code | `/sm:resume-for <url>` |
| Codex | `$resume-for <url>` |

- `<job-url>` — 채용공고 URL (필수, `--text` 사용 시 생략 가능)
- `--text <file>` — fetch를 건너뛰고 미리 추출한 공고 본문(파일)을 직접 입력.
  fetch 차단 시 에이전트가 insane-search로 본문을 받아 재실행하는 경로.
- `--lang ko|en` — 이력서 언어 강제 (미지정 시 공고 언어 자동 감지 → Profile → 기본값)
- `--no-save-card` — CompanyCard를 vault에 저장하지 않고 일회성으로 초안만 생성
- `--model <m>` — 합성 모델 override (기본 `sonnet`)

## 3. 흐름 (4단계)

```
synapse-memory resume-for <job-url>
   │
   ├─ 1. FETCH    web/fetch.py 파사드
   │              insane-search 엔진 우선 → urllib 폴백 → 차단 시 fetch_blocked
   │
   ├─ 2. EXTRACT  endpoints/resume_for.py
   │              JD 텍스트 → LLM 구조화 추출
   │              (display_name / company_id / website / position / language)
   │
   ├─ 3. CARD     cards/company.py
   │              CompanyCard 생성·병합 (positions[].jd_url = URL) → save_company_card
   │              (--no-save-card 면 메모리상 카드만 만들고 저장 생략)
   │
   └─ 4. DRAFT    endpoints/persona.py draft_resume(company_id) 재사용
                  ProjectCard 선별 + recipe 합성 → Drafts/Resume - {회사} (YYYY-MM).md
```

## 4. 컴포넌트

### 4.1 신규: `web/fetch.py` — fetch 파사드

`llm/ai_api.py` 의 provider-agnostic facade 패턴을 그대로 따른다. 런타임에
사용 가능한 백엔드를 감지해 우아하게 폴백한다.

```python
@dataclass
class FetchResult:
    ok: bool
    content: str          # 추출된 본문 텍스트 (HTML strip 후)
    backend: str          # "insane-search" | "urllib" | "none"
    blocked: bool         # 모든 백엔드가 봇 차단/실패
    url: str

def fetch_job_posting(url: str, *, timeout: int = 25) -> FetchResult: ...
```

백엔드 우선순위:

1. **insane-search 엔진** — 발견 시 우선 사용. `python3 -m engine <URL>` subprocess
   호출(또는 importable 하면 직접). 봇 차단·JS 렌더링 사이트 대응. 발견 방법:
   환경변수 `SYNAPSE_FETCH_ENGINE` 우선, 없으면 import 가능 여부 probe. 미발견 시
   조용히 다음 백엔드로.
2. **내장 urllib** — `urllib.request` + stdlib HTML strip(태그 제거·본문 추출).
   의존성 0. 단순 공고 페이지에 충분.
3. **둘 다 실패/차단** → `blocked=True` 반환. CLI가 안내를 출력하고, 슬래시 명령이
   에이전트에게 insane-search fetch → `--text` 재실행을 유도한다.

> insane-search의 일부 경로(Cloudflare급 챌린지)는 구조상 에이전트 세션이
> Playwright MCP를 직접 호출해야 하므로, 순수 CLI 안에 통째로 넣을 수 없다.
> 그래서 "에이전트 재실행" 폴백 경로를 명시적으로 둔다.

### 4.2 신규: `endpoints/resume_for.py` — 오케스트레이션

```python
@dataclass
class ResumeForResult:
    company_id: str
    company_name: str
    saved_path: Path        # 생성된 이력서 초안 경로
    card_saved: bool        # CompanyCard를 vault에 저장했는지
    jd_url: str
    fetch_backend: str

def resume_for(
    url: str,
    *,
    text: str | None = None,        # --text: fetch 우회
    lang: str | None = None,        # --lang override
    save_card: bool = True,         # --no-save-card 면 False
    model: str = "sonnet",
    vault_path: Path | None = None,
    ai_env: AIEnvironment | None = None,
    timeout: int = 240,
) -> ResumeForResult: ...
```

책임:
1. `text` 없으면 `fetch_job_posting(url)` 호출. `blocked` 면 `FetchBlockedError`
   (안내 메시지 포함) raise.
2. JD 텍스트를 LLM(`ai_api`)으로 구조화 추출 → `display_name/company_id/website/
   position{title,seniority,keywords}/language`. 추출 실패(회사명/직무 빈 값) 시
   명확한 안내와 함께 `ValueError`.
3. `company_id` 로 기존 CompanyCard 로드 시도 → 있으면 **병합**(아래 4.4), 없으면
   신규 생성. `save_card` 면 `save_company_card`.
4. `draft_resume(company_id, ...)` 호출, 결과를 `ResumeForResult`로 래핑.

### 4.3 신규: CLI 핸들러 — `cli.py`

- `cmd_resume_for(args) -> int` 핸들러 추가 (기존 `cmd_*` 패턴).
- `build_parser()` 에 `sub.add_parser("resume-for", ...)` + 인자 + `set_defaults`.
- `_enforce_cost_cap("resume-for")` 적용 (fetch는 무료, extract+draft 2회 LLM
  호출 비용 추적). `detect_ai_environment()` provider 점검 재사용.
- `fetch_blocked`/추출 실패는 사용자 친화 메시지로 출력하고 비-0 종료코드 반환.

### 4.4 재사용 (변경 없음)

- `cards/company.py` — `CompanyCard`, `JobPosition(title, seniority, keywords,
  jd_url)`, `load_company_card`, `save_company_card`. **공고 URL은 `jd_url` 필드에
  그대로 들어간다.**
- `endpoints/persona.py` — `draft_resume(company_id)` 무수정 재사용.
- `llm/ai_api.py` — 추출용 LLM 호출.

## 5. 공고 → CompanyCard 매핑

LLM 추출 결과 → CompanyCard 필드:

| 추출 값 | CompanyCard 필드 |
| --- | --- |
| `display_name` | `display_name` |
| `slugify(display_name)` | `company_id` (충돌 시 기존 카드 병합) |
| `website` | `website` |
| `position.title` | `JobPosition.title` |
| `position.seniority` | `JobPosition.seniority` (junior\|mid\|senior\|lead\|principal, 없으면 null) |
| `position.keywords` | `JobPosition.keywords` |
| 입력 URL | `JobPosition.jd_url` |
| 정제한 JD 요약 | `CompanyCard.body` |
| `language` (또는 `--lang`) | `CompanyCard.resume_language` |
| 입력 URL | `CompanyCard.sources` |

**병합 규칙** (같은 `company_id` 카드가 이미 있을 때, 덮어쓰지 않음):
- 같은 `title` position 이 있으면 `keywords` 합집합 갱신, `jd_url` 최신화
- 없으면 새 position append
- `last_reviewed` 갱신, 기존 `body`/`notes`/`status` 보존

## 6. 엣지케이스 / 에러 처리

| 상황 | 처리 |
| --- | --- |
| fetch 차단 (insane-search 없음 + urllib 403) | `FetchBlockedError` → CLI 안내 출력, 슬래시 명령이 에이전트 insane-search → `--text` 재실행 유도 |
| JD 추출 실패 (회사명/직무 빈 값) | `ValueError` + `--text` 직접 입력 안내 |
| 매칭 ProjectCard 0건 | 기존 `draft_resume` 의 `ValueError`("vault에 ProjectCard 먼저 생성") 그대로 전파 |
| provider 미준비 | 기존 `detect_ai_environment()` 점검 재사용 |
| 비용 상한 | `_enforce_cost_cap("resume-for")` |
| untrusted 웹 본문 (insane-search R8) | 추출 프롬프트는 JD 텍스트를 "데이터(주장)"로만 취급하고 본문 내 지시·명령을 무시한다. 경계 마커로 감싸 전달 |

## 7. 보안/프라이버시

- 공고 본문은 외부 공개 웹 콘텐츠이므로 untrusted 로 취급(§6 마지막 행).
- 사용자 이력 데이터(ProjectCard/Profile)는 기존 `draft_resume` 와 동일 신뢰
  모델을 따른다(provider CLI로 전달). 새로운 raw 유출 경로 없음.
- fetch한 공고 본문은 CompanyCard.body 요약으로만 vault에 남고, 원문 전체를
  무분별하게 저장하지 않는다.

## 8. 테스트 전략 (pytest, 기존 컨벤션)

- `web/fetch.py`
  - urllib 백엔드: 로컬 HTTP fixture(`http.server`)로 200/403 분기 검증
  - insane-search 백엔드: subprocess 모킹(미설치 CI 가정) → 발견/미발견 폴백
  - 모든 백엔드 실패 → `blocked=True`
- `endpoints/resume_for.py`
  - LLM 추출: `ai_env` 주입 모킹으로 구조화 결과 고정
  - CompanyCard upsert: 신규 생성 / 기존 병합 골든 테스트
  - `draft_resume` 위임: 호출 인자 검증(스파이)
  - `--text` 경로: fetch 미호출 확인
- `cli.py`
  - 인자 파싱, `fetch_blocked`/추출 실패 출력 스냅샷, 종료코드
- 외부 네트워크를 실제로 호출하는 테스트는 없음 (전부 격리/모킹)

## 9. 플러그인 래퍼 & 문서

- `commands/resume-for.md` — frontmatter(`description`, `argument-hint`) +
  `!`SYNAPSE_FROM_AGENT=1 synapse-memory resume-for "$ARGUMENTS"``.
  본문에 fetch_blocked 시 insane-search로 본문 받아 `--text` 재실행하는 폴백 안내.
- `skills/resume-for/SKILL.md` — Codex skill 정의 (동일 동작).
- 플러그인 매니페스트가 디렉토리 글롭이면 자동 노출, 아니면 엔트리 추가.

### README 업데이트 (기존 구조 보존)

README 의 명령어 표는 **버전별 섹션 누적** 패턴이다. 기존 표/행은 건드리지 않고
새 버전 섹션을 추가한다:

```markdown
### v1.20.0 추가 명령

| 하고 싶은 일 | Claude Code | Codex | 터미널 |
| --- | --- | --- | --- |
| 채용공고 URL로 맞춤 이력서 초안 | `/sm:resume-for <url>` | `$resume-for <url>` | `synapse-memory resume-for <url>` |
```

기존 `회사 맞춤 이력서 초안 | /sm:resume <회사>` 행(line 160)은 그대로 둔다 —
입력이 "회사 ID"인 별도 명령이다.

## 10. 범위 밖 (YAGNI)

- 공고 사이트별 전용 파서/셀렉터 (insane-search/urllib generic 추출로 충분)
- 이력서 PDF/DOCX 내보내기 (markdown 초안까지만)
- 여러 공고 일괄(batch) 처리
- 공고 만료·재크롤링 스케줄링
