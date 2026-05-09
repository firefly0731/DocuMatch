# documatch-desktop 설계 문서

- 작성일: 2026-05-09
- 출발점: `documatch-cli` v0.1.0 (CLI 라이브러리, GitHub `firefly0731/documatch-cli`, 161 테스트, 89% 커버리지)
- 목표: 같은 코드베이스 위에 PySide6 기반 데스크톱 GUI 앱을 추가해 v0.2.0 릴리즈
- 결과물: Windows `.exe` + macOS `.app`, 회사 컴/개인 컴 모두에서 사용 가능

## 0. 브레인스토밍 결정 사항

| 항목 | 결정 | 이유 |
|---|---|---|
| 주 사용자 | 본인만 (개인 도구) | 코드 사이닝/자동 업데이트/텔레메트리 인프라 모두 불필요 |
| OS 범위 | Windows + macOS 둘 다 | 회사 컴(Windows) + 개인 컴(macOS) |
| 프레임워크 | PySide6 / Qt for Python | 기존 Python 코드 재사용, 단일 프로세스, 네이티브 위젯, PyInstaller 단일 파일 빌드 |
| 화면 레이아웃 | Hybrid (좌 단계 네비 + 중앙 작업 + 우 미리보기) | "풍부한 입력 + 미리보기 + 인라인 편집" 요구에 가장 적합 |
| 편집 가능 영역 | 스펙 필드 / 샘플 결과 셀 / 배치 결과 전체 / 행 추가삭제 — 전부 | 사용자가 직접 수정해가며 피드백 주는 워크플로 |
| CLI 유지 여부 | 유지 (CLI + GUI 동시 entry-point) | CI/배치 자동화에 CLI 여전히 유용 |
| 배포 채널 | GitHub Release private repo에 첨부 | PyPI는 CLI wheel만, GUI는 단일 파일 |

## 1. 아키텍처

### 1.1 단일 프로세스 단일 패키지

같은 `documatch-cli` repo 내부에 `src/documatch/gui/` 모듈을 추가한다. 별도 repo로 분리하지 않음. ExtractionEngine/processors/extractors/factory 모두 그대로 import해서 재사용.

```
┌─────────────────────────────────────────────────────────┐
│  documatch-desktop  (PySide6 단일 프로세스)            │
│                                                          │
│  ┌─────────────┐    ┌─────────────────────────────┐   │
│  │ Qt UI Layer │───▶│ GuiReviewer (Reviewer 구현) │   │
│  │ (MainWindow,│    └──────────────┬──────────────┘   │
│  │  panels,    │                   │                    │
│  │  dialogs)   │            ┌──────▼─────────┐        │
│  └─────────────┘            │ ExtractionEngine│ (재사용) │
│         ▲                   └──────┬─────────┘        │
│         │                          │                   │
│         │   ┌────────────┐  ┌──────▼──────┐           │
│         └───┤ asyncio    │  │ LLM clients │           │
│             │ + qasync   │  │ + processors│           │
│             └────────────┘  └─────────────┘           │
└─────────────────────────────────────────────────────────┘
```

### 1.2 핵심 결정

1. **새 Reviewer 구현체 `GuiReviewer`**: 기존 `InteractiveReviewer`(Rich)/`AutomationReviewer`와 동등한 위치. ExtractionEngine은 어떤 Reviewer가 들어와도 동일하게 동작.
2. **이벤트 루프 통합**: `qasync` 라이브러리로 Qt event loop과 asyncio를 통합. LLM 호출은 native async (anthropic/openai SDK는 이미 httpx 기반), 별도 스레드 불필요.
3. **CLI 유지**: `documatch` (CLI) + `documatch-gui` (GUI) 두 entry-point. 공유: 엔진, processors, extractors, exporters, spec, config, llm 모듈 전부.
4. **API 키 저장**: OS 키체인(`keyring` 라이브러리) 우선 + `~/.config/documatch/.env` 폴백 (CLI와 호환).

### 1.3 의존성 추가 (pyproject.toml)

```toml
[project.optional-dependencies]
gui = [
    "PySide6>=6.7",
    "qasync>=0.27",
    "keyring>=25",
    "PyMuPDF>=1.24",      # PDF 렌더링/검색
    "mammoth>=1.7",       # DOCX → HTML
]
dev = [
    # ... 기존
    "pytest-qt>=4.4",
    "pyinstaller>=6.6",
]

[project.scripts]
documatch = "documatch.cli.app:main"
documatch-gui = "documatch.gui.app:main"
```

CLI 사용자는 `pip install documatch-cli`, GUI 사용자는 `pip install documatch-cli[gui]` 또는 PyInstaller 빌드 다운로드.

## 2. Reviewer Protocol 확장

### 2.1 Protocol을 async로 전환

GUI 인터랙션은 본질적으로 비동기(버튼 클릭을 Future로 await). 엔진과 자연스럽게 맞물리려면 Protocol 메서드도 async 통일.

```python
class Reviewer(Protocol):
    async def confirm_files(self, entries) -> ReviewDecision: ...
    async def select_sample(self, entries) -> Path | None: ...
    async def get_user_prompt(self, default=None) -> str: ...
    async def review_spec(self, spec, available_specs=None) -> SpecDecision: ...
    async def confirm_mode(self, spec) -> ModeDecision: ...
    async def review_sample(self, result, spec) -> SampleDecision: ...
    async def review_batch_results(self, results, spec) -> BatchResultDecision: ...  # 신규
    async def get_save_path(self, default) -> Path: ...
    async def confirm_save_spec(self, spec) -> str | None: ...
    async def show_progress(self, current, total, label) -> None: ...
    async def show_message(self, level, text) -> None: ...
```

기존 InteractiveReviewer/AutomationReviewer: 메서드 시그니처에 `async` 키워드만 추가, 본문 변경 없음. `ExtractionEngine.run`에서 `self.reviewer.X(...)` → `await self.reviewer.X(...)`.

### 2.2 결정 dataclass에 `modified_*` 필드

GUI가 사용자 편집 결과를 명시적으로 엔진에 돌려주는 통로.

```python
@dataclass
class SpecDecision:
    action: Literal["approve", "refine", "load", "back", "quit"]
    feedback: str | None = None
    spec_name: str | None = None
    modified_spec: ExtractionSpec | None = None      # 신규

@dataclass
class SampleDecision:
    action: Literal["full", "single_save", "refine", "back", "quit"]
    feedback: str | None = None
    modified_spec: ExtractionSpec | None = None      # 신규
    modified_result: ExtractionResult | None = None  # 신규

@dataclass
class BatchResultDecision:                           # 완전 신규
    action: Literal["save", "back", "quit"]
    modified_results: list[ExtractionResult] | None = None
```

엔진 처리 규칙: `modified_*`가 None이 아니면 그쪽을 우선 사용. CLI Reviewer는 항상 None 반환(현재 동작 유지). GUI Reviewer만 편집 결과를 채워서 반환.

### 2.3 새 HITL 게이트 — `review_batch_results`

배치 추출 끝난 후 Excel 저장 직전에 결과 전체를 검토/편집하는 게이트. 7개 단계로 늘어남.

각 Reviewer 구현체의 기본 동작:
- **AutomationReviewer**: 즉시 `BatchResultDecision(action="save")` 반환
- **InteractiveReviewer (CLI)**: 결과 요약 + Y/N 프롬프트, 편집 미지원, 실수 발견 시 `action="back"`으로 재배치 가능
- **GuiReviewer**: 풀 데이터그리드 + 셀 편집 + 행 추가/삭제 + Excel 미리보기

### 2.4 엔진 변경 영향

`engine.py`의 흐름은 `await reviewer.X(...)`로 통일 + `decision.modified_spec`/`modified_result`/`modified_results` 처리 분기 추가. 기존 흐름은 모두 호환.

## 3. 화면 설계

### 3.0 공통 레이아웃

```
┌─────────────────────────────────────────────────────────────┐
│ [상단 툴바: 폴더경로 / 설정 / API키 / 스펙관리]            │
├──────┬──────────────────────────┬───────────────────────────┤
│ 단계 │  메인 작업 영역          │  문서 미리보기            │
│ 네비 │  (단계별로 다름)         │  (PDF/DOCX/XLSX/이미지)   │
│ 1✓   │                          │                           │
│ 2✓   │                          │                           │
│ 3▸   │                          │                           │
│ 4    │                          │                           │
│ 5    │                          │                           │
│ 6    │                          │                           │
│ 7    │                          │                           │
├──────┴──────────────────────────┴───────────────────────────┤
│ [하단 액션바: ← 이전 / 보조 / 다음 →]                      │
└─────────────────────────────────────────────────────────────┘
```

### 3.1 앱 시작 화면

폴더 미선택 상태. 큰 "폴더 열기" 버튼 + 최근 폴더 목록 + 드래그앤드롭. 우측·하단 비어 있음.

### 3.2 Step 1 — 파일 스캔 결과

- **중앙**: `QTreeView` 파일 목록(확장자/파일명/시트명/크기). 상단에 확장자 chip(`PDF: 8 / DOCX: 2 / XLSX: 4시트`). 멀티셀렉트 + 우클릭 "제외".
- **우측**: 선택된 행의 미리보기 thumbnail/텍스트
- **하단**: `← 폴더 변경` / `필터 적용` / `다음 →`

### 3.3 Step 2 — 샘플 선택 + 의도 입력

- **중앙 상단**: 후보 파일 라디오 (PDF/DOCX/TXT 등 우선, xlsx 시트는 후순위)
- **중앙 하단**: 추출 의도 multi-line 입력
- **우측**: 선택된 파일 실시간 미리보기
- **하단**: `← Step 1` / `→ 스키마 생성 (LLM 호출)`

스키마 생성은 백그라운드 비동기, 진행 인디케이터 표시.

### 3.4 Step 3 — 스펙 검토 + 편집

- **중앙**: 스펙 필드 표(이름/타입/필수/설명 인라인 편집) + 요약 템플릿 + 자연어 피드백 박스
- **우측**: 샘플 PDF 미리보기, AI가 식별한 추출 대상 영역 하이라이트
- **하단**: `← 샘플 재선택` / `LLM에 재요청` / `저장된 스펙 불러오기` / `승인 + 다음 →`

승인 시 `decision.modified_spec`에 편집 내용 담아서 반환.

### 3.5 Step 4 — 모드 확정

두 큰 카드(단일값/테이블) + AI 추천 표시. 우측 미리보기 유지. `← Step 3` / `→ 샘플 추출 시작`.

### 3.6 Step 5 — 샘플 결과 검토 + 편집

- **단일값 모드**: 필드별 편집 가능 행(이름/값/신뢰도/검토필요)
- **테이블 모드**: `QTableView` 셀 편집, 행 우클릭 추가/삭제/재추출
- **우측**: 원본 + 클릭한 셀 위치 하이라이트 (best-effort)
- **하단**: `← 모드` / `LLM 재요청` / `스키마 수정` / `이 건만 저장` / `전체 진행 →`

"전체 진행" 시 사용자 편집값을 첫 결과로 사용할지 옵션 제공.

### 3.7 Step 6 — 배치 진행 + 전체 결과 편집

**Phase 6A (추출 중)**:
- 중앙: 진행률 + 처리 중 파일명 + 처리/실패/경고 카운트
- 우측: 가장 최근 결과 미리보기
- 하단: `일시정지` / `중단`

**Phase 6B (완료, 검토)**:
- 중앙: 풀 `QTableView` (Excel-like). 검토필요 행 노랑, 실패 빨강. 셀 편집 + 행 추가/삭제 + 검색/정렬/필터
- 우측: 클릭 행의 원본 미리보기
- 하단: `← 재배치` / `→ 저장`

이 화면이 사실상 "Excel 미리보기 + 편집기".

### 3.8 Step 7 — 저장

- **중앙**: 저장 경로 + 자동 생성 파일명 + Excel 미리보기 thumbnail (results + failed_documents 시트 탭) + "스펙 저장" 체크박스
- **하단**: `← 결과로` / `저장 + 종료`

### 3.9 보조 화면 (모달)

- **API 키 입력 모달**: 6 필드 + "BATCH도 같은 URL?" + "OS 키체인 사용" 체크
- **설정 패널**: LLM URL/모델 변경, 디버그 토글, batch_size, scan_threshold, 데이터 폴더 열기
- **스펙 관리 모달**: 목록 + 미리보기 + 불러오기/삭제/내보내기/가져오기
- **진행 인디케이터**: footer indicator (모달 X)

### 3.10 단계 점프

좌측 네비에서 통과한 단계는 클릭으로 점프 가능. 편집은 보존, 이후 단계는 재실행 필요시 경고 표시.

## 4. 문서 미리보기 구현

### 4.1 추상화

```python
class DocumentPreview(QWidget):
    def load(self, path: Path) -> None: ...
    def highlight(self, text: str) -> None: ...
    def clear_highlights(self) -> None: ...
    def go_to_page(self, page: int) -> None: ...
```

`PreviewPanel`(우측 stacked widget)이 확장자 보고 dispatch.

### 4.2 PDF — PyMuPDF (fitz)

- `fitz.open(path)` → `page.get_pixmap(matrix=fitz.Matrix(zoom, zoom))` → QImage
- 보이는 페이지만 lazy 렌더, LRU 캐시(메모리 100MB 한도)
- 검색: `page.search_for(query)` → `list[Rect]` → QPainter overlay 노란 박스
- 줌: ctrl+휠, zoom_factor 0.5x ~ 4x

### 4.3 DOCX — mammoth + QTextEdit

- `mammoth.convert_to_html(f)` → `QTextEdit.setHtml()` (헤딩/표/굵은체/이미지 base64 보존)
- `QTextEdit.find()`로 텍스트 검색·하이라이트
- QtWebEngine 미사용(바이너리 +50MB 절약)

### 4.4 XLSX — openpyxl + QTableView

- 상단 `QTabBar` 시트 목록 + 본문 `QTableView` + `QStandardItemModel`
- 셀 값만(수식 결과). 풍부한 서식은 v2.0+

### 4.5 이미지 — QGraphicsView + QPixmap

- pan/zoom 0.1x ~ 10x
- 텍스트 검색 N/A (highlight 비활성)

### 4.6 텍스트 계열

- TXT: `QPlainTextEdit` (모노스페이스, readonly)
- MD: `QTextEdit.setMarkdown()`
- CSV: `QTableView` + 파싱
- HTML: `QTextEdit.setHtml()` (script 제거)

### 4.7 셀↔원문 하이라이트 연동

추출 결과 셀 클릭 시 우측에서 해당 값 자동 검색·하이라이트. LLM이 좌표 안 주므로 텍스트 매칭 best-effort. 정확한 OCR 좌표 매핑은 v2.0+ 보류.

### 4.8 성능

- PDF: 첫 페이지 즉시, 나머지 lazy
- DOCX 1MB+: 백그라운드 변환 + 인디케이터
- XLSX 5000+ row: 가상 스크롤 (`QAbstractTableModel`)
- 미리보기 캐시 최근 3개 파일

## 5. 비동기/스레딩 모델

### 5.1 qasync로 Qt + asyncio 통합

```python
def main():
    app = QApplication(sys.argv)
    loop = qasync.QEventLoop(app)
    asyncio.set_event_loop(loop)
    window = MainWindow()
    window.show()
    with loop:
        loop.run_forever()
```

이후 Qt 슬롯이 async coroutine이 될 수 있음 — `@qasync.asyncSlot` 데코레이터.

### 5.2 LLM HTTP는 native async

`anthropic.AsyncAnthropic`, `openai.AsyncOpenAI`, 회사 모드의 `OpenAIClient(base_url=...)` 모두 httpx async. `await llm.ainvoke(...)` 동안 Qt event loop은 계속 돌아 UI 안 멈춤. 별도 스레드 불필요, 기존 LLM 클라이언트 코드 변경 0.

### 5.3 CPU bound은 ThreadPoolExecutor

PDF 렌더링, 대용량 DOCX 변환은 `loop.run_in_executor(None, ...)`로 백그라운드 스레드 실행.

### 5.4 GuiReviewer는 Future로 사용자 입력 대기

```python
class GuiReviewer:
    async def review_spec(self, spec, available_specs=None):
        future = asyncio.get_event_loop().create_future()
        self._main_window.show_spec_panel(spec, on_decision=future.set_result)
        return await future
```

### 5.5 배치 — 일시정지/중단

```python
async def _run_batch(self):
    results = []
    async for result in self.batch_extractor.extract_all(self.entries):
        results.append(result)
        self.update_progress(len(results), len(self.entries), result)
        await self._pause_event.wait()  # paused면 여기서 멈춤
    self.show_batch_review(results)
```

- 일시정지: `_pause_event.clear()` → 다음 await에서 멈춤
- 중단: `engine_task.cancel()` → CancelledError, 부분 결과 보존
- 부분 실패는 `BatchExtractor`가 이미 격리 (`status="failed"`)

### 5.6 진행 시그널

`QObject` Signal로 메인 스레드 일관성 유지:
```python
class BatchProgressSignals(QObject):
    progress = Signal(int, int, str)
    item_done = Signal(object)
    finished = Signal(list)
    error = Signal(str)
```

### 5.7 상태 보존 (옵션, 우선순위 낮음)

- 매 5건마다 `~/.config/documatch/state/<session_id>.json`에 누적 결과 직렬화
- 앱 재시작 시 "중단된 작업 재개?" 다이얼로그
- 정상 저장 시 state 파일 삭제

MVP에 안 넣고 v0.2.x로 보류 가능.

### 5.8 에러 전파

| 출처 | 처리 |
|---|---|
| LLMRateLimitError | 토스트 + 자동 재시도 (3회 backoff) |
| LLMAuthError | 모달: "API 키 오류" → 설정 패널 |
| ConfigError | 시작 시 API 키 입력 모달, 도중엔 토스트 |
| ProcessorError/UnsupportedFormatError | 해당 행만 `status=failed`, 배치 계속 |
| UserAbort | 메인 화면 복귀 |
| asyncio.CancelledError | 부분 결과 Step 6B로 |
| 예측 못 한 예외 | 모달 + 디버그 로그 경로 안내 |

### 5.9 디버그 로그

`~/.config/documatch/logs/<session_id>.log`에 stderr/asyncio 트레이스. 설정 패널의 "디버그 폴더 열기" 버튼.

## 6. 테스트 전략

### 6.1 기존 161 테스트 영향

Reviewer async 전환으로 약 30개 테스트 함수가 `async def` + `await`로 마이그레이션. `pytest-asyncio` 자동 모드 이미 설정되어 있어 mechanical 변경. 통과 개수는 161 그대로.

### 6.2 새 의존성

```toml
dev = [
    # 기존
    "pytest-qt>=4.4",
]
```

CI 헤드리스: `xvfb-run pytest tests/gui/` (Linux) 또는 `QT_QPA_PLATFORM=offscreen pytest`.

### 6.3 새 테스트 카테고리 (총 ~100건)

| 카테고리 | 위치 | 추정 |
|---|---|---|
| GuiReviewer 단위 | `tests/gui/test_gui_reviewer.py` | 25 |
| 패널 위젯 | `tests/gui/test_panels_*.py` | 25 |
| 편집 위젯 | `tests/gui/test_widgets_*.py` | 20 |
| 미리보기 위젯 | `tests/gui/test_preview_*.py` | 15 |
| 비동기/스레딩 | `tests/gui/test_async.py` | 10 |
| E2E | `tests/gui/test_e2e_gui.py` | 5 시나리오 |

기존 161 + 신규 100 = **목표 260+ 테스트**.

### 6.4 카테고리별 패턴

- **GuiReviewer**: pytest-qt + asyncio.create_task로 reviewer 메서드 호출, qtbot으로 사용자 액션 시뮬레이션, decision 검증
- **패널/편집 위젯**: 단순 unit test — load/edit/get_state 검증
- **미리보기**: fixture 파일 로드 + highlight 검색 결과 검증
- **비동기**: pause_event, asyncio.Task.cancel() 동작 검증
- **E2E**: FakeLLMClient + qtbot으로 7단계 시뮬레이션, 최종 Excel 산출물 검증

### 6.5 안 할 것 (YAGNI)

- 픽셀 단위 시각 회귀 (스크린샷 비교, 폰트/DPI/플랫폼 차이로 fragile)
- 모든 OS 자동화 — GUI E2E는 macOS만, Windows는 수동
- LLM 응답 다양성 — FakeLLMClient만, 실 API는 수동

### 6.6 수동 스모크 (`docs/MANUAL_SMOKE.md`)

릴리즈 전 30분 가량:
- 폰트/색상 정상 (각 OS)
- PDF 한글 폰트, 줌·검색·하이라이트
- DOCX 표·이미지 렌더링
- XLSX 시트 탭
- 키보드 단축키, 다크모드, 한글 IME

### 6.7 커버리지

- 전체: 89% 유지 (Reviewer async 전환은 영향 없음)
- GUI 모듈: 75%+ 목표 (UI 분기 풍부, 100%는 비현실적)
- 통합 후: 85%+ 유지

## 7. 빌드 & 배포

### 7.1 PyInstaller spec

```python
# documatch_gui.spec
import sys
a = Analysis(
    ["src/documatch/gui/app.py"],
    pathex=["src"],
    datas=[
        ("src/documatch/i18n/messages.json", "documatch/i18n"),
        ("assets/icon.png", "assets"),
    ],
    hiddenimports=[
        "anthropic", "openai", "pypdf", "openpyxl", "xlrd",
        "docx", "bs4", "chardet", "pdf2image", "PIL",
        "fitz", "mammoth",
        "keyring.backends.macOS", "keyring.backends.Windows",
    ],
    excludes=["tkinter", "matplotlib"],
)
pyz = PYZ(a.pure, a.zipped_data)
exe = EXE(
    pyz, a.scripts, a.binaries, a.zipfiles, a.datas,
    name="DocuMatch",
    icon="assets/icon.icns" if sys.platform == "darwin" else "assets/icon.ico",
    console=False, onefile=True, upx=False,
)
if sys.platform == "darwin":
    app = BUNDLE(
        exe, name="DocuMatch.app", icon="assets/icon.icns",
        bundle_identifier="com.firefly0731.documatch",
        info_plist={
            "CFBundleShortVersionString": "0.2.0",
            "CFBundleVersion": "0.2.0",
            "NSHighResolutionCapable": "True",
            "LSApplicationCategoryType": "public.app-category.productivity",
        },
    )
```

### 7.2 OS별 빌드

| OS | 명령 | 산출물 | 크기 |
|---|---|---|---|
| Windows | `pyinstaller documatch_gui.spec` | `dist/DocuMatch.exe` | ~80-90 MB |
| macOS | `pyinstaller documatch_gui.spec` | `dist/DocuMatch.app` | ~110 MB (압축 후 ~70) |

크로스 컴파일 불가 — 각 OS에서 빌드 필요.

### 7.3 헬퍼 스크립트

`scripts/build_gui.py`: 빌드 정리 + PyInstaller 실행 + 산출물 안내. `scripts/generate_icons.py`: 1024×1024 원본에서 .ico/.icns 파생.

### 7.4 GitHub Actions (release.yml 확장)

태그 push 시:
1. PyPI 배포 (CLI wheel) — 기존 그대로 (선택)
2. macOS 빌드 → `DocuMatch-macos.zip` 업로드
3. Windows 빌드 → `DocuMatch.exe` 업로드

`runs-on` 매트릭스: `[macos-latest, windows-latest]`. 빌드 시간 5-7분.

### 7.5 배포 흐름 (개인용)

1. 코드 수정 → `git tag v0.2.0 && git push origin v0.2.0`
2. GitHub Actions가 macOS .app + Windows .exe 둘 다 빌드, Release에 첨부
3. 회사 컴/개인 컴에서 GitHub Release 페이지 → 다운로드 → 더블클릭
4. 첫 실행 시 init 모달 → API 키 입력

업데이트는 새 버전 다운로드 → 덮어쓰기. 사용자 데이터(`~/.config/documatch/`)는 별도 위치라 영향 없음.

### 7.6 첫 실행 보안 경고

코드 사이닝 안 하므로 OS 경고 발생 — 본인만 쓰니 우회 OK.
- Windows: SmartScreen → "추가 정보" → "실행"
- macOS: 시스템 환경설정 → 개인정보보호 및 보안 → "DocuMatch 열기" 또는 우클릭 → "열기"

README에 한 줄 안내.

### 7.7 산출물 분리

| 산출물 | 출처 | 사용처 |
|---|---|---|
| `documatch-cli-0.2.0.whl` | `python -m build` | `pip install documatch-cli` |
| `documatch-cli[gui]` extras | 같은 패키지 | PySide6 등 추가 |
| `DocuMatch.exe` / `DocuMatch.app` | PyInstaller | Python 미설치 사용자 |

## 8. 프로젝트 구조 + 마이그레이션

### 8.1 v0.2.0 폴더 구조 (추가분만 표시)

```
documatch-cli/
├── documatch_gui.spec                    # 신규
├── assets/                               # 신규
│   ├── icon-source.png
│   ├── icon.ico
│   ├── icon.icns
│   └── screenshots/
├── scripts/                              # 신규
│   ├── build_gui.py
│   └── generate_icons.py
├── docs/
│   └── MANUAL_SMOKE.md                   # 신규
├── src/documatch/gui/                    # 완전 신규
│   ├── app.py
│   ├── reviewer.py
│   ├── main_window.py
│   ├── panels/
│   │   ├── start_panel.py
│   │   ├── file_panel.py
│   │   ├── sample_panel.py
│   │   ├── spec_panel.py
│   │   ├── mode_panel.py
│   │   ├── result_panel.py
│   │   ├── batch_panel.py
│   │   ├── save_panel.py
│   │   └── step_nav.py
│   ├── widgets/
│   │   ├── spec_table_editor.py
│   │   ├── result_table_editor.py
│   │   ├── api_key_dialog.py
│   │   ├── settings_dialog.py
│   │   ├── spec_manager_dialog.py
│   │   └── progress_indicator.py
│   ├── preview/
│   │   ├── base.py
│   │   ├── pdf_preview.py
│   │   ├── docx_preview.py
│   │   ├── xlsx_preview.py
│   │   ├── image_preview.py
│   │   ├── text_preview.py
│   │   └── preview_panel.py
│   ├── threads/
│   │   └── batch_controller.py
│   └── theme/
│       └── styles.qss
└── tests/gui/                            # 완전 신규
    ├── conftest.py
    ├── test_gui_reviewer.py
    ├── test_panels_*.py (단계별 7개)
    ├── test_widgets_spec_editor.py
    ├── test_widgets_result_editor.py
    ├── test_preview_pdf.py
    ├── test_preview_docx.py
    ├── test_preview_xlsx.py
    ├── test_async_pause_cancel.py
    └── test_e2e_happy_path.py
```

신규 파일/폴더 약 35-40개.

### 8.2 마이그레이션 단계

| Phase | 내용 | 추정 |
|---|---|---|
| **Phase 1** | Reviewer Protocol async 마이그레이션 + `BatchResultDecision`/`modified_*` 추가 + 기존 Reviewer/엔진/테스트 await 처리 | 2-3일 |
| **Phase 2** | gui/ 스켈레톤 — qasync app, MainWindow, StepNav, 빈 패널 7개. `documatch-gui` 명령 동작 | 3-4일 |
| **Phase 3** | 단계별 패널 — Step 1, 2, 4, 7 (단순한 것부터). GuiReviewer와 시그널/future 연결 | 4-5일 |
| **Phase 4** | Step 3 (스펙 편집) + Step 5 (샘플 결과 편집) — `SpecTableEditor`, `ResultTableEditor` | 4-5일 |
| **Phase 5** | Step 6 (배치 진행 + 결과 편집) + 일시정지/중단 + `BatchController` | 4일 |
| **Phase 6** | 미리보기 위젯 — PDF/DOCX/XLSX/Image. 셀↔원문 하이라이트 | 4-5일 |
| **Phase 7** | API 키 다이얼로그 + 설정 + 스펙 관리 + 키체인 + 에러 polish | 3-4일 |
| **Phase 8** | 빌드 파이프라인 — PyInstaller spec, 아이콘, GitHub Actions 매트릭스 | 2-3일 |
| **Phase 9** | 테스트 백필 — 신규 ~100 테스트 + 수동 스모크 체크리스트 | 4-5일 |
| **Phase 10** | Polish — 다크모드, 단축키, 토스트, 한글 IME, README 업데이트 | 3-4일 |

**총 추정 ~5-6주** (한 사람 풀타임).

### 8.3 변경 영향 (기존 v0.1.0 코드)

| 모듈 | 변경 |
|---|---|
| `core/engine.py` | `await reviewer.X` + `review_batch_results` + `modified_*` 처리 |
| `interaction/reviewer.py` | Protocol async + `BatchResultDecision` + `modified_*` 필드 |
| `interaction/automation.py` | 모든 메서드 async + `review_batch_results` 추가 |
| `interaction/interactive.py` | 모든 메서드 async + `review_batch_results` 추가 |
| `cli/_runner.py` | `await engine.run(...)` (이미 asyncio.run 안에서 호출 중) |
| `pyproject.toml` | `[gui]` extras + `documatch-gui` entry-point + dev `pytest-qt` |
| `tests/unit/test_reviewer_*.py`, `tests/integration/test_engine_*.py` | `async def` + `await` (mechanical) |
| 그 외 (`processors`, `extractors`, `exporters`, `spec`, `llm`, `config`, `cli/*` 비-runner) | **변경 0** |

기존 코드 80% 손대지 않음 — Reviewer 추상화 덕분.

### 8.4 버전 / 릴리즈 전략

- v0.1.x: CLI 패치 (현재 v0.1.0)
- v0.2.0: 첫 GUI 릴리즈 (이 사양 완료 시점)
- v0.2.x: GUI 패치 / 사소한 추가
- v0.3.0+ 후보: 자동 업데이트, 다국어, agenda 파이프라인 부활 등 (사양 외)

각 phase 완료 시 `feat(gui): ...` 커밋 누적, 모든 phase 끝나면 v0.2.0 태그 + GitHub Release.

### 8.5 README 추가 섹션 (예고)

v0.2.0 README:
- Quick Start (GUI): 다운로드 → 실행 → 첫 실행 init
- Quick Start (CLI): 기존 그대로
- 언제 GUI vs CLI: 단발/시각 검토 → GUI, CI/배치/스크립트 → CLI
- 시스템 요구사항: macOS 12+ / Windows 10+
- 다운로드: GitHub Releases 링크
- 첫 실행 보안 경고 우회 가이드

### 8.6 v0.1.0 호환성

- CLI 명령 그대로 동작
- 저장된 스펙 (`~/.config/documatch/specs/`) 호환
- `~/.config/documatch/.env` 양쪽 다 사용
- 외부 의존자 없으니 breaking 영향 없음

## 9. 위험 요소 + 완화

| 위험 | 완화 |
|---|---|
| qasync + Qt event loop 통합 시 미묘한 데드락 | LLM 호출은 항상 `await`, 동기 Qt 코드와 섞지 않음. 문제 발생 시 QThread 격리로 폴백 |
| PyInstaller --onefile 시작 시간 (3-5초) | 사용자 본인만 쓰는 환경이라 수용. 빠른 시작 필요시 directory 모드로 전환 |
| PyMuPDF 라이선스 (AGPL) | 개인용/비상업이라 OK. 상업화 시 PyMuPDF Pro 라이선스 또는 QtPdf로 교체 |
| Qt 한글 IME 입력 이슈 (특히 Windows) | `QInputMethod` 명시적 활성화 + 수동 스모크에서 검증 |
| GitHub Actions macOS 러너 빌드 시 PyMuPDF 의존성 컴파일 실패 가능성 | wheel 우선 설치, 실패 시 venv pin |
| 코드 사이닝 미적용으로 OS 경고 | README에 우회 절차 안내. 본인만 쓰니 OK |
| 대용량 배치(1000+ 파일) 메모리 | 결과를 메모리에 다 들고 있는 현 설계는 이 정도면 충분. 더 크면 디스크 스트리밍 필요 (v0.3) |

## 10. 참조

- 기존 CLI 설계: `docs/superpowers/specs/2026-05-09-documatch-cli-design.md`
- 기존 CLI 구현 계획: `docs/superpowers/plans/2026-05-09-documatch-cli.md`
- 세션 핸드오프: `docs/superpowers/specs/2026-05-09-documatch-cli-handoff.md`
- 코드 repo: https://github.com/firefly0731/documatch-cli
- PySide6 문서: https://doc.qt.io/qtforpython-6/
- qasync: https://github.com/CabbageDevelopment/qasync
- PyMuPDF: https://pymupdf.readthedocs.io/
- mammoth: https://github.com/mwilliamson/python-mammoth
- PyInstaller: https://pyinstaller.org/
