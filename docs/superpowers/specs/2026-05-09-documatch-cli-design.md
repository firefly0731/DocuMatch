# documatch-cli 설계 문서

- 작성일: 2026-05-09
- 출발점: `DocuMatch/backend/app/cli/` 폴더의 기능
- 목표: 새 독립 GitHub 저장소 + PyPI 공개 배포 가능한 product 수준 문서 추출 CLI 라이브러리

## 1. 목적과 범위

### 1.1 한 줄 정의
`documatch-cli`는 PDF/DOCX/XLSX/이미지/HTML 등 다양한 포맷의 문서에서 사용자가 자연어로 지정한 항목을 LLM으로 추출하여 Excel로 출력하는 대화형 CLI 라이브러리이다.

### 1.2 사용자 시나리오 (golden path)
1. 사용자가 `documatch ./samples`를 실행한다.
2. CLI가 폴더를 스캔하여 지원 포맷 파일 목록을 표시한다.
3. 사용자는 스키마 생성용 샘플 파일 1건을 선택한다.
4. 사용자는 자연어로 "어떤 정보를 추출할지" 입력한다 (예: "송장 번호, 금액, 발행일, 공급자명").
5. LLM이 ExtractionSpec(필드 정의)을 자동 생성하고 Rich 테이블로 표시한다.
6. 사용자는 [A]승인 / [R]피드백으로 수정 / [L]저장된 스펙 불러오기 / [B]샘플 재선택 / [Q]종료 중 선택한다.
7. 승인 시 샘플 파일에 대해 추출을 실행하고 결과를 보여준다.
8. 사용자는 결과를 검토하고 추가 피드백 또는 [A]전체 진행 선택.
9. 전체 파일에 대해 배치 추출 (진행률 표시, `b` 키로 중단 가능).
10. 결과 Excel을 사용자가 지정한 경로에 저장 후 종료.

### 1.3 비목표 (Non-goals, MVP 기준)
- Agenda 파이프라인 (LLM 기반 안건 그룹화 + 종합 분석) — DocuMatch 내부에 남기고 라이브러리에서 제외
- LangGraph/LangChain 의존성
- Web UI / REST API
- 외부 entry-point 기반 processor 플러그인 시스템 (v0.4 로드맵)
- 영어/다국어 i18n (한국어 전용)
- 비용 추정 dashboard (v0.2 로드맵)

### 1.4 결정 사항 (브레인스토밍 결과)
| 항목 | 결정 |
|---|---|
| 배포 형태 | 새 독립 GitHub 저장소 + PyPI 공개 |
| 기능 범위 | Simple Batch 모드만 (Agenda 파이프라인 제외) |
| LLM 공급자 | Anthropic + OpenAI 동시 지원 |
| UI 언어 | 한국어 전용 |
| CLI 구조 | 대화형 단일 진입점 + 자동화 플래그 |
| 지원 포맷 | PDF/DOCX/XLSX/XLS/JPG/PNG + TXT/MD/CSV/HTML |
| 패키지명 | PyPI: `documatch-cli`, CLI 명령: `documatch` |
| HITL | 7단계 파이프라인의 6개 지점에서 사용자 피드백 적용 후 동작 |

## 2. 패키지 구조

src layout. 단위 테스트 가능성·책임 경계를 우선한다.

```
documatch-cli/
├── pyproject.toml
├── README.md / LICENSE / CHANGELOG.md / CONTRIBUTING.md
├── .env.example
├── .github/workflows/ci.yml
├── .github/workflows/release.yml
├── src/documatch/
│   ├── __init__.py                     # public API: ExtractionEngine, ExtractionSpec, Settings
│   ├── exceptions.py
│   ├── i18n.py                         # 한국어 메시지 dict
│   ├── core/
│   │   ├── engine.py                   # ExtractionEngine — 파이프라인 오케스트레이터
│   │   ├── pipeline.py                 # PipelineState + 단계 전이
│   │   └── models.py                   # ExtractionResult, BatchResult, FieldValue
│   ├── spec/
│   │   ├── models.py                   # ExtractionSpec, Field, FieldType, SummaryTemplate, ReviewRule
│   │   ├── generator.py                # LLM 기반 스펙 자동 생성·수정
│   │   └── store.py                    # JSON 저장/로드 (~/.config/documatch/specs/)
│   ├── llm/
│   │   ├── base.py                     # LLMClient Protocol, LLMMessage, LLMResponse
│   │   ├── anthropic.py                # Claude 구현 (prompt caching 활용)
│   │   ├── openai.py                   # GPT 구현 (Responses API + JSON schema)
│   │   └── factory.py                  # 설정 기반 클라이언트 선택
│   ├── processors/
│   │   ├── base.py                     # DocumentProcessor Protocol, UnifiedDocument
│   │   ├── pdf.py                      # pypdf + pdf2image (Poppler)
│   │   ├── docx.py
│   │   ├── xlsx.py                     # openpyxl, 시트별 분리
│   │   ├── xls.py                      # xlrd
│   │   ├── image.py                    # Pillow → base64
│   │   ├── text.py                     # TXT/MD/CSV/HTML 통합 처리
│   │   ├── registry.py                 # 확장자 → processor 매핑
│   │   └── scanner.py                  # 디렉토리 스캔 + FileEntry 생성
│   ├── extractors/
│   │   ├── batch.py                    # BatchExtractor (N개 문서 → N개 결과)
│   │   └── single.py                   # SingleExtractor (샘플 검토용)
│   ├── exporters/
│   │   └── excel.py                    # openpyxl 기반 Excel 출력
│   ├── interaction/                    # HITL 추상화
│   │   ├── reviewer.py                 # Reviewer Protocol
│   │   ├── interactive.py              # InteractiveReviewer (Rich 기반)
│   │   ├── automation.py               # AutomationReviewer (자동 승인)
│   │   ├── prompts.py                  # 단일/다중 행 입력 헬퍼
│   │   └── feedback.py                 # 피드백 → spec/result 반영
│   ├── cli/
│   │   ├── app.py                      # Click 진입점 (console_scripts)
│   │   ├── interactive.py              # 기본 대화형 흐름
│   │   ├── automation.py               # 비대화형 흐름
│   │   ├── hyperlinks.py               # `documatch hyperlinks fix`
│   │   ├── spec_commands.py            # `documatch spec list/show/delete`
│   │   └── config_commands.py          # `documatch config show`
│   └── config/
│       ├── settings.py                 # pydantic-settings 기반 Settings
│       └── defaults.py                 # 기본 모델/임계값
└── tests/
    ├── unit/
    ├── integration/
    └── fixtures/
        ├── docs/                       # 작은 샘플 문서들
        └── specs/                      # 미리 저장된 spec JSON
```

### 2.1 책임 경계 핵심
- `core/`는 LLM·CLI·파일 형식을 모른다. Engine은 Protocol 인터페이스만 다룬다.
- `interaction/`은 HITL을 1급 시민으로 분리하여 CLI/Web/API 어디서든 재사용 가능하게 한다.
- 자동화 모드는 동일 engine에 `AutomationReviewer`를 주입하여 동일 코드 경로를 탄다.

## 3. 파이프라인 & HITL 흐름

7단계 상태 머신. 각 단계는 `PipelineState`를 입력받아 다음 상태를 반환한다. HITL 지점에서 `Reviewer` 콜백을 호출한다.

```
[0] Init: Settings 로드, LLM 클라이언트 생성 (spec 역할 + extract 역할)
        ↓
[1] Scan: scan_path → entries: list[FileEntry]
        ↓ HITL ① "스캔된 N개 파일 확인 → [A]계속 / [E]포맷 필터 / [Q]종료"
[2] Sample 선택
        ↓ HITL ② "샘플로 사용할 파일 번호 입력"
[3] Spec 생성: SpecGenerator(sample, prompt) → ExtractionSpec
        ↓ HITL ③ "[A]승인 / [R]피드백으로 수정 / [L]저장된 스펙 불러오기 / [B]샘플 재선택 / [Q]"
        ← R 선택 시: feedback → SpecGenerator.refine(spec, feedback) → 다시 ③
[4] Mode 확인: AI 추천 + 사용자 확정
        ↓ HITL ④ "[S]단일값 / [T]테이블 / [B]스키마 수정으로"
[5] Sample 추출: SingleExtractor → ExtractionResult
        ↓ HITL ⑤ "[A]전체 진행 / [R]피드백으로 스펙 수정 후 재추출 / [S]이 건만 저장 / [B]스펙 수정 / [Q]"
        ← R 선택 시: feedback → refine(spec) → re-extract sample → 다시 ⑤
[6] Batch 추출: BatchExtractor.extract_all(entries) (진행률 + 'b' 키 중단)
        ↓
[7] Export
        ↓ HITL ⑥ "저장 경로 입력 (기본값 표시)"
        → 검토필요 항목 강조, 저장 후 종료
```

### 3.1 Reviewer Protocol

```python
class Reviewer(Protocol):
    def confirm_files(self, entries: list[FileEntry]) -> ReviewDecision: ...
    def select_sample(self, entries: list[FileEntry]) -> Path: ...
    def review_spec(self, spec: ExtractionSpec) -> SpecDecision: ...
    def confirm_mode(self, spec: ExtractionSpec) -> ModeDecision: ...
    def review_sample(self, result: ExtractionResult, spec: ExtractionSpec) -> SampleDecision: ...
    def get_save_path(self, default: Path) -> Path: ...
```

- `InteractiveReviewer`: Rich 기반, 현재 UX 보존 + 개선
- `AutomationReviewer`: `--auto-approve` 플래그로 모든 결정을 기본값으로 자동 승인

### 3.2 HITL 보존 항목 (현재 구현 → 라이브러리)
- 다중 행 입력 (`_multiline_input`)
- 뒤로가기 키 [B]
- 배치 중간 'b' 키로 중단
- 샘플 재추출 후 재검토 루프
- 저장된 스펙 재사용 [L]

### 3.3 자동화 모드 안전 장치
- `--auto-approve`만 있고 `--spec`/`--spec-file`이 없으면 → 명확한 에러로 즉시 실패
- 이유: LLM이 갓 생성한 스펙을 사람이 검토하지 않은 채 100건 처리하는 사고 방지

## 4. LLM Provider 추상화

### 4.1 인터페이스

```python
@dataclass
class LLMMessage:
    role: Literal["system", "user", "assistant"]
    content: str | list[dict]

@dataclass
class LLMResponse:
    content: str
    usage: TokenUsage         # input/output tokens, cache hits
    raw: Any                  # provider-native object

class LLMClient(Protocol):
    provider: str
    model: str
    supports_vision: bool
    supports_prompt_cache: bool

    async def ainvoke(
        self,
        messages: list[LLMMessage],
        *,
        json_schema: dict | None = None,
        cache_breakpoint: int | None = None,
        temperature: float = 0.0,
        max_tokens: int = 4096,
    ) -> LLMResponse: ...
```

### 4.2 Anthropic 구현
- `anthropic` SDK 사용
- `cache_control: {"type": "ephemeral"}`을 시스템 프롬프트와 spec 직렬화 부분에 자동 부여 → 배치 추출 시 약 90% 절감
- Vision: `image/jpeg` base64 블록 그대로 전달
- JSON 모드: tool use 패턴으로 강제 (`tools=[json_schema_tool]`, `tool_choice="json_schema_tool"`)

### 4.3 OpenAI 구현
- `openai` SDK
- Responses API + `response_format={"type": "json_schema", "json_schema": {...}}`
- Vision: `image_url` 블록(`data:image/jpeg;base64,...`)
- Prompt cache: 자동 캐싱(>1024 token prefix), `cache_breakpoint` 인자는 무시 + DEBUG 로그

### 4.4 역할 분리
| 역할 | 용도 | 권장 모델 |
|---|---|---|
| `spec` | 스키마 생성·수정·정제 | Claude Sonnet / GPT-4o |
| `extract` | 배치 추출 | Claude Haiku / GPT-4o-mini |

사용자가 양쪽을 같은 provider로 강제 가능하며, `spec=anthropic` + `extract=openai` 같은 혼합도 허용한다.

### 4.5 JSON 응답 파싱
- Provider-agnostic `parse_json_response(text)` 헬퍼: fenced code block 제거 + lenient JSON repair
- JSON 강제 모드 우선 시도, 실패 시 텍스트 파싱으로 fallback
- pydantic 모델로 직접 검증하여 잘못된 응답은 즉시 실패 + 1회 재시도

### 4.6 제거되는 의존성
- `langchain`, `langgraph`, `langgraph-checkpoint-sqlite` 모두 제거
- 직접 LLM 호출이 더 단순하고 PyPI 패키지 무게 감소

## 5. 파일 프로세서 레이어

### 5.1 UnifiedDocument

```python
@dataclass
class UnifiedDocument:
    filename: str
    raw_text: str                     # LLM 입력용 텍스트
    page_count: int
    is_scanned: bool                  # PDF 한정
    images_base64: list[str] | None   # Vision LLM용
    metadata: dict                    # sheet_name, encoding 등
```

### 5.2 처리 매핑

| 확장자 | Processor | 의존성 | 처리 |
|---|---|---|---|
| `.pdf` | `PdfProcessor` | pypdf + pdf2image + Poppler | 텍스트 추출 → 빈 결과 시 자동 OCR(이미지+Vision LLM) |
| `.docx` | `DocxProcessor` | python-docx | 단락+표 텍스트 |
| `.xlsx` | `XlsxProcessor` | openpyxl | 시트별로 분리, sheet_name 메타 |
| `.xls` | `XlsProcessor` | xlrd | xlsx 동일 패턴 |
| `.jpg/.jpeg/.png` | `ImageProcessor` | Pillow | base64, raw_text 빈, requires_vision (BMP/GIF/TIFF는 v0.1 비지원) |
| `.txt` | `TextProcessor` | chardet | 자동 인코딩 감지 |
| `.md` | `MarkdownProcessor` | (stdlib) | raw text |
| `.csv` | `CsvProcessor` | (stdlib) | 마크다운 표 변환 |
| `.html` | `HtmlProcessor` | beautifulsoup4 | 본문 추출, script/style 제거 |

### 5.3 의존성 정책 — 모두 코어
사용자가 `pip install documatch-cli` 한 번으로 모든 포맷이 즉시 동작한다. extras_require 분리하지 않는다.

```toml
dependencies = [
    "pypdf>=4.0",
    "pdf2image>=1.17",
    "Pillow>=10.0",
    "python-docx>=1.1",
    "openpyxl>=3.1",
    "xlrd>=2.0",
    "beautifulsoup4>=4.12",
    "chardet>=5.0",
    "anthropic>=0.40",
    "openai>=1.50",
    "click>=8.1",
    "rich>=13.7",
    "pydantic>=2.5",
    "pydantic-settings>=2.1",
]
```

### 5.4 시스템 바이너리 의존성
- `pdf2image`는 Poppler가 필요. 스캔 PDF에만 사용. 텍스트 PDF는 `pypdf`만으로 동작.
- 처리 흐름:
  1. PDF → `pypdf`로 텍스트 추출
  2. 텍스트가 `pdf_scan_threshold`자(기본 100) 미만이면 스캔 PDF로 판정
  3. Poppler 호출 → 이미지 → Vision LLM
  4. Poppler 미설치 시: 친절한 에러 + 설치 안내 (`brew install poppler` / `apt install poppler-utils` / Windows 가이드 링크)
- README의 Quick Start에 Poppler 설치를 옵션 단계로 명시 (스캔 PDF 안 다루면 생략 가능)

### 5.5 스캐너
- `scan_directory(path) -> list[FileEntry]`: 재귀 + glob, 지원 확장자만 수집
- XLSX는 시트별로 분리 → 한 파일이 여러 `FileEntry` 생성 (현재 동작 유지)
- 정렬: 파일명 기준 (안정적인 결과 순서)
- 무시 패턴: `~$*` 등 임시 파일 기본 제외

### 5.6 Registry

```python
_REGISTRY: dict[str, type[DocumentProcessor]] = {
    ".pdf": PdfProcessor, ".docx": DocxProcessor, ".xlsx": XlsxProcessor,
    ".xls": XlsProcessor, ".jpg": ImageProcessor, ".jpeg": ImageProcessor,
    ".png": ImageProcessor, ".txt": TextProcessor, ".md": MarkdownProcessor,
    ".csv": CsvProcessor, ".html": HtmlProcessor, ".htm": HtmlProcessor,
}

def get_processor(extension: str) -> DocumentProcessor: ...
def supported_extensions() -> tuple[str, ...]: ...
```

외부 entry-point 자동 로딩은 v0.4에서 추가할 수 있도록 dict 기반으로 유지한다.

## 6. ExtractionSpec 모델 & 영속화

### 6.1 모델 (pydantic v2)

```python
class FieldType(str, Enum):
    STRING = "string"
    NUMBER = "number"
    DATE = "date"
    BOOLEAN = "boolean"
    LIST = "list"

class Field(BaseModel):
    name: str
    description: str
    type: FieldType = FieldType.STRING
    required: bool = True
    format: str | None = None
    examples: list[str] | None = None

class ReviewRule(BaseModel):
    condition: Literal["low_confidence", "missing_required"]
    threshold: float | None = None

class SummaryTemplate(BaseModel):
    template: str
    fallback_template: str | None = None

class OutputColumn(BaseModel):
    field_name: str
    excel_header: str
    width: int = 15

class ExtractionSpec(BaseModel):
    schema_version: Literal["1"] = "1"
    name: str
    description: str | None = None
    doc_type_hint: str
    fields: list[Field]
    summary_template: SummaryTemplate
    output_table: list[OutputColumn]
    review_rules: list[ReviewRule] = Field(default_factory=lambda: [
        ReviewRule(condition="low_confidence", threshold=0.7),
        ReviewRule(condition="missing_required"),
    ])
    table_mode: bool = False
    created_at: datetime = Field(default_factory=datetime.utcnow)
    source_sample: str | None = None
```

### 6.2 SpecStore

```python
class SpecStore:
    base_dir: Path  # 기본 ~/.config/documatch/specs/, 환경변수로 override

    def save(self, spec: ExtractionSpec, *, overwrite: bool = False) -> Path: ...
    def load(self, name: str) -> ExtractionSpec: ...
    def load_from_path(self, path: Path) -> ExtractionSpec: ...
    def list(self) -> list[SpecInfo]: ...
    def delete(self, name: str) -> None: ...
```

- 파일 이름 검증: 영숫자 + 하이픈/언더스코어만 허용 (path traversal 방지)
- 직렬화: `spec.model_dump_json(indent=2)`
- 로드 시 `schema_version` 체크. v1만 지원, v2 추가 시 마이그레이션 훅을 둔다

### 6.3 SpecGenerator

```python
class SpecGenerator:
    def __init__(self, llm: LLMClient): ...

    async def generate(self, sample: UnifiedDocument, user_prompt: str) -> ExtractionSpec:
        """샘플 + 자연어 의도 → ExtractionSpec
        - 텍스트 + 이미지(있으면) 둘 다 LLM에 전달
        - JSON schema 강제 모드로 응답
        - table_mode 추천을 LLM이 함께 결정
        """

    async def refine(
        self,
        current_spec: ExtractionSpec,
        feedback: str,
        sample: UnifiedDocument | None = None,
    ) -> ExtractionSpec:
        """현재 스펙 + 자연어 피드백 → 수정된 스펙
        - 현재 plan_agent.continue_plan_graph 단순화 버전
        - LangGraph 의존성 없이 stateless 단일 LLM 호출
        """
```

### 6.4 HITL 흐름과의 연결
- `[L]저장된 스펙 불러오기` → `SpecStore.list()` 표시 → 선택 → 단계 [3] 스킵 후 [4]로
- `[A]승인` 후 자동으로 `SpecStore.save(spec, overwrite=False)` 제안 (이름 기본값 = `<doc_type_hint>_<timestamp>`)
- 자동화 모드: `documatch --spec invoice_v1` 또는 `documatch --spec-file ./my-spec.json`로 즉시 사용

## 7. 설정 & CLI UX

### 7.1 Configuration

3-tier override (낮음 → 높음): defaults → config 파일 → 환경 변수 → CLI 플래그

```python
class LLMRoleConfig(BaseModel):
    provider: Literal["anthropic", "openai"] = "anthropic"
    model: str = "claude-sonnet-4-6"
    timeout: float = 120.0
    max_retries: int = 3

class Settings(BaseSettings):
    # API 키는 표준 변수명을 그대로 읽는다 (DOCUMATCH_ prefix 미적용)
    anthropic_api_key: str | None = Field(default=None, validation_alias="ANTHROPIC_API_KEY")
    openai_api_key: str | None = Field(default=None, validation_alias="OPENAI_API_KEY")

    # 그 외 설정은 DOCUMATCH_ prefix
    llm_spec: LLMRoleConfig = LLMRoleConfig()
    llm_extract: LLMRoleConfig = LLMRoleConfig(model="claude-haiku-4-5-20251001")
    spec_store_dir: Path = Path("~/.config/documatch/specs").expanduser()
    pdf_scan_threshold: int = 100
    extraction_batch_size: int = 20
    debug: bool = False

    model_config = SettingsConfigDict(
        env_prefix="DOCUMATCH_",
        env_nested_delimiter="__",
        populate_by_name=True,
        toml_file=[
            Path.cwd() / "documatch.toml",
            Path("~/.config/documatch/config.toml").expanduser(),
        ],
    )
```

API 키는 `ANTHROPIC_API_KEY`/`OPENAI_API_KEY` 표준 변수명을 그대로 읽는다 (Anthropic·OpenAI SDK 관행 일치). 그 외 설정은 `DOCUMATCH_` prefix + `__` nested delimiter를 사용한다.

### 7.2 환경 변수 예시

```
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
DOCUMATCH_LLM_EXTRACT__PROVIDER=openai
DOCUMATCH_LLM_EXTRACT__MODEL=gpt-4o-mini
DOCUMATCH_DEBUG=true
```

### 7.3 config.toml 예시

```toml
[llm_spec]
provider = "anthropic"
model = "claude-sonnet-4-6"

[llm_extract]
provider = "openai"
model = "gpt-4o-mini"

pdf_scan_threshold = 200
```

### 7.4 CLI 진입점

```bash
documatch                                            # 대화형
documatch ./samples                                  # 시작 경로 사전 지정
documatch ./samples --spec invoice_v1                # 저장된 스펙 사용
documatch ./samples --spec invoice_v1 --output result.xlsx --auto-approve   # 완전 자동화
documatch ./samples --spec-file ./my-spec.json --output result.xlsx --auto-approve

documatch hyperlinks fix ./output.xlsx
documatch spec list
documatch spec show invoice_v1
documatch spec delete invoice_v1
documatch config show
documatch --version
documatch --help
```

### 7.5 플래그

```
documatch [PATH] [OPTIONS]
  --spec NAME                  저장된 스펙 이름
  --spec-file PATH             외부 JSON 스펙 파일
  --output PATH                결과 Excel 경로
  --auto-approve               모든 HITL 결정을 기본값으로 자동 승인
  --provider [anthropic|openai]   양쪽 역할 동시 지정
  --no-save-spec               스펙 자동 저장 제안 비활성화
  --debug                      LLM 응답 raw 출력
  --config PATH                특정 config.toml 강제 사용
  --fail-on-review             검토 필요 항목 발생 시 비-0 종료
  --fail-on-error N            N건 이상 실패 시 비-0 종료
```

## 8. 에러 처리 & 테스트

### 8.1 예외 계층

```python
class DocuMatchError(Exception): pass
class ConfigError(DocuMatchError): pass            # 설정·API 키 누락/오타
class DependencyError(DocuMatchError): pass        # 시스템 바이너리 누락
class UnsupportedFormatError(DocuMatchError): pass
class ProcessorError(DocuMatchError): pass         # 파일 처리 실패
class LLMError(DocuMatchError): pass
class LLMRateLimitError(LLMError): pass
class LLMAuthError(LLMError): pass
class SpecError(DocuMatchError): pass
class SpecNotFoundError(SpecError): pass
class ExtractionError(DocuMatchError): pass
class UserAbort(DocuMatchError): pass
```

### 8.2 계층별 정책
- **processors**: 한 파일 실패는 `DocumentExtractionResult(status="failed", ...)`로 변환, 배치 전체는 계속 진행
- **llm**: rate limit → exponential backoff 자동 재시도. auth/config 에러 → fail-fast
- **extractors**: JSON 파싱 실패 → 1회 재시도, 그래도 실패하면 `partial` + needs_review=True
- **CLI**: 최상단에서 `DocuMatchError`를 잡아 친화적 메시지 + 종료 코드 (1: 일반, 2: 설정, 130: 사용자 abort)

### 8.3 부분 실패
- 100건 중 5건 실패해도 95건은 Excel 저장 + 실패 5건은 별도 시트 `failed_documents`에 표시
- 자동화 모드 기본은 종료 코드 0 + stderr 경고
- `--fail-on-error N`: N건 이상 처리 실패 시 비-0 종료 (기본 비활성)
- `--fail-on-review`: 검토 필요(needs_review=True) 항목이 1건이라도 있으면 비-0 종료 (CI 게이트용)

### 8.4 Logging
- stdlib `logging`, logger 이름 = `documatch`
- 기본 WARNING. `--debug` 또는 `DOCUMATCH_DEBUG=true`로 DEBUG
- DEBUG 시: LLM 요청/응답 raw, prompt cache 통계, 처리 시간 분해

### 8.5 테스트 디렉토리

```
tests/
├── unit/
│   ├── test_spec_models.py
│   ├── test_spec_store.py
│   ├── test_processors_pdf.py
│   ├── test_processors_docx.py
│   ├── test_processors_xlsx.py
│   ├── test_processors_text.py
│   ├── test_processors_image.py
│   ├── test_scanner.py
│   ├── test_llm_anthropic.py
│   ├── test_llm_openai.py
│   ├── test_llm_factory.py
│   ├── test_excel_exporter.py
│   ├── test_interaction_reviewers.py
│   └── test_config.py
├── integration/
│   ├── test_pipeline_happy_path.py
│   ├── test_pipeline_hitl_refine.py
│   ├── test_pipeline_resume_with_spec.py
│   ├── test_pipeline_partial_failure.py
│   └── test_cli_automation.py
└── fixtures/
    ├── docs/
    └── specs/
```

### 8.6 핵심 도구
- `FakeLLMClient`: `LLMClient` Protocol 구현. 입력 패턴 → 미리 등록된 응답으로 매핑. 모든 통합 테스트가 이를 사용 → CI에서 LLM API 키 불필요, 결정적, 빠름
- `pytest`, `pytest-asyncio`만으로 충분
- Click CLI 테스트: `click.testing.CliRunner`. 자동화 모드 한정 subprocess smoke test 별도

### 8.7 커버리지 목표
| 레이어 | 목표 |
|---|---|
| core / spec / llm | 90% |
| processors | 80% |
| cli / interaction | 70% |
| 전체 | 85% |

### 8.8 CI
- GitHub Actions, Python 3.11/3.12/3.13 매트릭스
- Ubuntu/macOS (Windows weekly로 별도 단순화)
- 단계: ruff → mypy → pytest --cov → coverage 업로드
- 평균 < 3분 목표 (FakeLLMClient 덕)
- PR마다 `documatch --version` smoke test로 설치 검증

## 9. 배포 & 운영

### 9.1 패키징

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "documatch-cli"
version = "0.1.0"
description = "AI 기반 문서 추출 CLI 툴"
readme = "README.md"
license = "MIT"
requires-python = ">=3.11"
keywords = ["document-extraction", "pdf", "llm", "cli", "excel"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Programming Language :: Python :: 3.13",
    "License :: OSI Approved :: MIT License",
    "Topic :: Office/Business",
    "Environment :: Console",
]
# dependencies: 섹션 5.3에 정의된 14개 패키지를 그대로 사용
# (pypdf, pdf2image, Pillow, python-docx, openpyxl, xlrd, beautifulsoup4,
#  chardet, anthropic, openai, click, rich, pydantic, pydantic-settings)

[project.scripts]
documatch = "documatch.cli.app:main"

[tool.hatch.build.targets.wheel]
packages = ["src/documatch"]
```

`authors`, `Homepage`, `Issues` 등 GitHub 소유자/이메일이 들어가는 필드는 신규 저장소 생성 시 확정한다 (구현 단계 첫 task에서 결정).

### 9.2 버전 관리
- SemVer
- 0.x: 베타 (API 변경 가능). 0.1.0이 첫 PyPI 배포
- 1.0.0: spec schema_version 안정화 + 외부 사용자 피드백 1회전 후
- `CHANGELOG.md`: Keep a Changelog 형식, 모든 PR이 entry 추가하도록 PR 템플릿에 체크박스

### 9.3 릴리즈 워크플로
1. `git tag v0.1.0 && git push --tags` → GH Actions 트리거
2. lint + type check + test 통과 확인
3. `python -m build` → wheel + sdist
4. PyPI Trusted Publishing (OIDC) — 토큰을 저장소에 두지 않는다
5. GitHub Release 자동 생성 + CHANGELOG 해당 버전 섹션 → release body
6. 주요 버전은 main에서 `release/v*` 브랜치를 거쳐 수동 PR 머지

### 9.4 문서 (README.md만)
1. 한 줄 설명 + 데모 GIF (asciinema)
2. Quick Start (설치 → API 키 → `documatch ./samples`)
3. 사용 예시 (대화형 / 자동화 / 저장된 스펙 재사용)
4. 지원 파일 포맷 표
5. 환경 변수 / config.toml 레퍼런스
6. FAQ (Poppler 설치, 비용 추정)
7. 기여 가이드 링크

### 9.5 표준 OSS 파일
- `.gitignore` (Python + IDE 표준)
- `.env.example` (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`)
- `LICENSE` (MIT)
- `CONTRIBUTING.md`

### 9.6 DocuMatch 본 저장소 마이그레이션
이 라이브러리 출시 후 `DocuMatch/backend/app/cli/`는 다음 중 하나를 권장:
- `documatch-cli`를 dev dependency로 import → `app.cli`는 wrapper만 남김
- 또는 그대로 유지하고 별도 발전 (이번 프로젝트 범위 외, 사용자 결정)

### 9.7 로드맵 (CHANGELOG에 미리 명시)
| 버전 | 기능 |
|---|---|
| v0.2 | 비용 추정 + 토큰 사용량 dashboard (`documatch usage`) |
| v0.3 | 검토 필요 항목 인라인 수정 UX (`documatch review <result.xlsx>`) |
| v0.4 | entry-point 기반 processor 플러그인 |
| v1.0 | spec schema v2 + 마이그레이션 도구 |

## 10. 성공 기준

이 프로젝트가 v0.1.0으로 PyPI에 배포되었을 때 다음을 만족해야 한다.

1. `pip install documatch-cli` 한 번으로 모든 핵심 포맷이 동작한다 (Poppler는 스캔 PDF 한정 옵션).
2. `documatch ./samples` 한 줄로 대화형 추출이 시작되어 7단계 모두 진행되며 6개 HITL 지점에서 사용자 결정을 받는다.
3. `--auto-approve --spec ...` 조합으로 CI/스크립트에서 비대화형 실행 가능하다.
4. Anthropic / OpenAI 어느 쪽 API 키만 있어도 동작한다 (해당 provider만 설정).
5. 저장된 스펙을 재사용하여 동일 문서 유형의 두 번째 배치를 빠르게 처리할 수 있다.
6. `pytest` 전체 통과율 100%, 커버리지 85% 이상.
7. 작은 PDF 1건 추출이 60초 이내에 완료된다 (네트워크/LLM 지연 별도).

## 11. 위험 요소

| 위험 | 완화 |
|---|---|
| Anthropic↔OpenAI 응답 형식 차이로 JSON 파싱 깨짐 | json_schema 강제 모드 + lenient parser + pydantic 검증 + 1회 재시도 |
| Poppler 미설치로 사용자 혼란 | 친절한 에러 메시지 + README FAQ |
| LLM 비용 사용자가 예측 못 함 | DEBUG 로그에 토큰/캐시 통계 표시. v0.2에서 dashboard |
| spec 직렬화 호환성 | `schema_version` 필드 + 로드 시 검증, v2 추가 시 마이그레이션 훅 |
| OpenAI Responses API spec 변경 | SDK 버전 핀 + 모델별 어댑터 격리 |

## 12. 참조

- 출발점 코드: `DocuMatch/backend/app/cli/test_runner.py` (2087 lines), `fix_hyperlinks.py`
- 의존 서비스: `app.services.{document_processor, batch_engine, plan_agent, agenda_grouping, synthesis, file_scanner, export, llm_client}`, `app.models.{agenda, extraction_spec}`, `app.core.config`
- 제외할 모듈: `agenda_grouping`, `synthesis`, `plan_agent` (LangGraph 기반)

---

## 13. 구현 현황 (2026-05-09 기준)

### 13.1 완료 상태
- v0.1.0 코드 완성, 161 테스트 통과, 커버리지 89%, ruff clean, vulture 0건
- GitHub: https://github.com/firefly0731/documatch-cli (private)
- 로컬: `/Users/yjban/Desktop/documatch-cli/`
- 최신 commit: `e90aed7 feat(cli): documatch init 서브커맨드 + 첫 실행 자동 부트스트랩`
- 태그 `v0.1.0` 푸시됨, PyPI 미배포

### 13.2 본 스펙 대비 추가된 기능

| 추가 항목 | 위치 | 메모 |
|---|---|---|
| 회사 LLM 서버 모드 | `OpenAIClient.base_url`, `factory._normalize_base_url`, `Settings.{plan,batch}_agent_*` | OpenAI 호환 chat-completions endpoint 사용 (사내 GenAI 게이트웨이 등) |
| `documatch init` 서브커맨드 | `cli/init_command.py` | LLM 서버 정보 6개를 대화형/플래그로 입력해 글로벌 .env에 저장 |
| 첫 실행 자동 부트스트랩 | `cli/_setup.py:ensure_setup` | tabula rasa 시 init 자동 진입, 부분 설정 시 누락 키만 prompt |
| 글로벌+로컬 .env 자동 로드 | `Settings.model_config.env_file` | `~/.config/documatch/.env` → `./.env` 순 (뒤가 우선) |

### 13.3 본 스펙 대비 변경된 부분

| 변경 | 사유 |
|---|---|
| Click `_SmartGroup` custom Group | scan_path 위치 인자와 spec/init/config/hyperlinks 서브커맨드 라우팅 충돌 해결 |
| LLM Protocol에서 `cache_breakpoint`, `json_schema` 파라미터 제거 | 어떤 caller도 전달하지 않는 dead 파라미터 (YAGNI 정리) |
| `LLMClient.supports_vision`, `supports_prompt_cache` 제거 | 한 번도 읽히지 않는 메타데이터 |
| `DocumentProcessor.extensions`, `requires_vision` ClassVar 제거 | registry가 하드코딩 매핑만 사용 — ClassVar는 dead |
| `FieldType(str, Enum)` → `StrEnum` | UP042 lint + Python 3.11+ 표준 |

### 13.4 미완료 / 다음 작업

| 작업 | 상태 |
|---|---|
| PyPI 계정/프로젝트명 등록 | ⏳ |
| Trusted Publishing OIDC 설정 (PyPI ↔ GitHub Actions) | ⏳ |
| `release.yml` 자동 트리거 → PyPI 업로드 | 워크플로 작성됨, 미실행 |
| 수동 스모크 (실 API 키, 6 HITL 게이트) | 사용자가 회사 컴에서 직접 |
| 커버리지 89% → 95% 보강 (xls.py, pdf.py 분기) | 선택 |
| GitHub repo public 전환 검토 | 사용자 결정 필요 (private 유지 가능) |

### 13.5 다음 세션에서 이어받기

세션 핸드오프 가이드: [`docs/superpowers/specs/2026-05-09-documatch-cli-handoff.md`](./2026-05-09-documatch-cli-handoff.md)

새 Claude Code 세션에서 작업 이어받으려면 위 핸드오프 doc을 먼저 읽어 컨텍스트 확보 후 진행.
