# documatch-cli v0.3.0 — CLI Polish + GUI Hard Removal Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** GUI 모듈을 main에서 hard delete하고 CLI(InteractiveReviewer)를 Rich 기반 production-quality UX로 전면 개편하여 v0.3.0 릴리스.

**Architecture:** Reviewer Protocol + Engine + processors/extractors/llm 변경 0건. `interaction/interactive.py` 전면 재작성 + `interaction/ui/` 신규 모듈 4개(header / progress / errors / tables). `documatch preview <file>` 신규 서브커맨드 + `--quiet/--json/--summary` 플래그 추가.

**Tech Stack:** Rich (Live, Spinner, Progress, Panel, Table), Click, Pydantic, pytest, pytest-asyncio

**Working directory:** `/Users/yjban/Desktop/documatch-cli/`
**스펙:** `DocuMatch/docs/superpowers/specs/2026-05-10-documatch-cli-v0.3-design.md`
**전제:** v0.2.1 기준 (commit `8330fc4`), 269 테스트 통과, 89% coverage

**Phase 개요 (6 phases / 26 tasks)**:
- Phase 1 (Tasks 1-3): GUI hard delete + pyproject 정리
- Phase 2 (Tasks 4-11): UI helper 모듈 4개 (header/progress/errors/tables)
- Phase 3 (Tasks 12-17): InteractiveReviewer 전면 재작성
- Phase 4 (Tasks 18-20): `documatch preview` 서브커맨드
- Phase 5 (Tasks 21-23): `--quiet/--json/--summary` 플래그
- Phase 6 (Tasks 24-26): 문서·버전·릴리스

## File Structure

```
src/documatch/
├── interaction/
│   ├── interactive.py            # 전면 재작성 (Phase 3)
│   ├── automation.py             # 변경 없음
│   ├── reviewer.py               # 변경 없음
│   ├── prompts.py                # 변경 없음
│   ├── __init__.py               # 변경 없음
│   └── ui/                       # 신규 (Phase 2)
│       ├── __init__.py           # public API export
│       ├── header.py             # render_step_header, render_action_legend
│       ├── progress.py           # live_spinner, progress_bar
│       ├── errors.py             # render_error_panel + REMEDY_MAP
│       └── tables.py             # render_spec_table, render_result_table
├── cli/
│   ├── app.py                    # preview 서브커맨드 + 플래그 추가 (Phase 4-5)
│   ├── _runner.py                # --quiet/--json/--summary 처리 (Phase 5)
│   ├── preview_command.py        # 신규 (Phase 4)
│   └── ... (기존 그대로)
├── core/, llm/, processors/, extractors/, exporters/, spec/, config/   # 변경 0건

# 삭제 (Phase 1):
src/documatch/gui/                # 전체 삭제
tests/gui/                        # 전체 삭제
documatch_gui.spec                # 삭제
scripts/build_gui.py              # 삭제
scripts/generate_icons.py         # 삭제
assets/                           # 전체 삭제
```

---

## Phase 1 — GUI Hard Removal (Tasks 1-3)

> **Phase 1 task 성격**: cleanup/build 작업 — TDD 사이클 적용 안 함. 각 task는 "변경 → 검증" 패턴.

### Task 1: GUI 코드/테스트/빌드 자산 일괄 삭제 (cleanup, non-TDD)

**Files:**
- Delete: `src/documatch/gui/` (전체 디렉토리)
- Delete: `tests/gui/` (전체 디렉토리)
- Delete: `documatch_gui.spec`
- Delete: `scripts/build_gui.py`, `scripts/generate_icons.py`
- Delete: `assets/` (전체 디렉토리)

- [ ] **Step 1.1: 일괄 삭제**

```bash
cd /Users/yjban/Desktop/documatch-cli
rm -rf src/documatch/gui
rm -rf tests/gui
rm -f documatch_gui.spec
rm -f scripts/build_gui.py scripts/generate_icons.py
rm -rf assets
```

- [ ] **Step 1.2: 삭제 확인**

```bash
test ! -d src/documatch/gui && echo "gui/ 삭제됨" || echo "FAIL"
test ! -d tests/gui && echo "tests/gui/ 삭제됨" || echo "FAIL"
test ! -f documatch_gui.spec && echo "spec 삭제됨" || echo "FAIL"
test ! -d assets && echo "assets/ 삭제됨" || echo "FAIL"
```

모두 "삭제됨" 표시되어야 함.

- [ ] **Step 1.3: Commit**

```bash
git add -A
git commit -m "feat(gui): GUI 모듈/테스트/빌드 자산 hard delete (v0.3.0 CLI-only 전환)"
```

---

### Task 2: pyproject.toml 정리 + version bump 0.3.0-dev (build, non-TDD)

**Files:**
- Modify: `pyproject.toml`
- Modify: `src/documatch/__init__.py`
- Modify: `.github/workflows/release.yml`

- [ ] **Step 2.1: pyproject.toml 수정**

`[project]` 섹션의 `version = "0.2.1"`을 다음으로 변경 (v0.3.0 release는 Task 25에서 -dev 제거):
```toml
version = "0.3.0-dev"
```

`[project.scripts]` 섹션을 다음으로 변경 (documatch-gui 제거):
```toml
[project.scripts]
documatch = "documatch.cli.app:main"
```

`[project.optional-dependencies]` 섹션의 `gui = [...]` 블록을 통째로 삭제. `dev` 블록에서 `pyinstaller>=6.6` 라인 제거. 최종 dev 블록:
```toml
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-cov>=4.1",
    "pytest-qt>=4.4",
    "ruff>=0.4",
    "mypy>=1.10",
    "build>=1.2",
    "vulture>=2.13",
]
```

`pytest-qt`도 GUI 테스트 제거됐으니 빼도 되지만 dev tool로 남겨둠 (선택). 깔끔히 하려면 제거.

`pytest-qt`도 제거하는 버전:
```toml
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-cov>=4.1",
    "ruff>=0.4",
    "mypy>=1.10",
    "build>=1.2",
    "vulture>=2.13",
]
```

후자 채택. (GUI 의존성 깨끗하게 정리)

- [ ] **Step 2.2: src/documatch/__init__.py 수정**

현재 값(`__version__ = "0.2.1"`)을 `__version__ = "0.3.0-dev"`로 변경.

- [ ] **Step 2.3: .github/workflows/release.yml 수정 — build-gui matrix 제거**

`.github/workflows/release.yml` 전체 교체:

```yaml
name: Release
on:
  push:
    tags: ["v*"]

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install Poppler
        run: sudo apt-get update && sudo apt-get install -y poppler-utils
      - run: pip install -e ".[dev]"
      - run: ruff check src/ tests/
      - run: pytest -q
      - run: python -m build

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
          files: dist/*
```

- [ ] **Step 2.4: 의존성 재설치**

```bash
cd /Users/yjban/Desktop/documatch-cli
source .venv/bin/activate
pip install -e ".[dev]"
```

- [ ] **Step 2.5: Commit**

```bash
git add pyproject.toml src/documatch/__init__.py .github/workflows/release.yml
git commit -m "build: v0.3.0 — gui extras/entry-point/build matrix 제거 + version 0.3.0"
```

---

### Task 3: 잔존 import 정리 + 테스트 통과 확인 (verification, non-TDD)

**Files:**
- Modify: 잔존 GUI import이 있는 파일 (있다면)
- 또는 변경 0

- [ ] **Step 3.1: 잔존 GUI 참조 검색**

```bash
cd /Users/yjban/Desktop/documatch-cli
grep -rE "documatch\.gui|from documatch\.gui|import documatch\.gui|src/documatch/gui|tests/gui|documatch-gui" \
    src/ tests/ scripts/ pyproject.toml .github/ 2>&1 | grep -v ".venv"
```

기대: 출력 비어 있음(또는 CHANGELOG 같은 docs 한정). 잔존 참조 있으면 해당 파일에서 라인 삭제.

- [ ] **Step 3.2: 전체 테스트 실행**

```bash
pytest -q
```

기대: **약 169 테스트 통과** (269 baseline - 100 GUI 테스트 = 169). 실패하면 잔존 import 또는 GUI에서 정의했던 fixture 의존성 검토.

- [ ] **Step 3.3: Lint**

```bash
ruff check src/ tests/
```

기대: All checks passed!

- [ ] **Step 3.4: Commit (수정사항 있을 때만)**

```bash
git add -A
git commit -m "chore: 잔존 GUI 참조 정리"
```

수정사항 없으면 skip.

Phase 1 완료. CLI만 남은 깨끗한 상태. 169 테스트 통과 유지.

---

## Phase 2 — UI Helper Modules (Tasks 4-11)

### Task 4: ui/__init__.py + header.py — render_step_header

**Files:**
- Create: `src/documatch/interaction/ui/__init__.py` (빈 파일)
- Create: `src/documatch/interaction/ui/header.py`
- Create: `tests/unit/test_ui_header.py`

- [ ] **Step 4.1: 빈 ui/__init__.py 생성**

```bash
mkdir -p src/documatch/interaction/ui
touch src/documatch/interaction/ui/__init__.py
```

- [ ] **Step 4.2: 실패하는 테스트 작성** — `tests/unit/test_ui_header.py`:

```python
"""ui.header — 단계 헤더 + 단축키 legend."""
import io

from rich.console import Console

from documatch.interaction.ui.header import render_step_header


def test_step_header_shows_step_number_and_total():
    console = Console(file=io.StringIO(), force_terminal=False, width=80)
    render_step_header(console, step=3, total=7, name="invoice_extraction")
    out = console.file.getvalue()
    assert "Step 3" in out
    assert "7" in out
    assert "invoice_extraction" in out


def test_step_header_includes_separator_lines():
    console = Console(file=io.StringIO(), force_terminal=False, width=80)
    render_step_header(console, step=1, total=7, name="test")
    out = console.file.getvalue()
    # 위/아래 ━ 구분선 (Rich Rule 또는 직접 출력)
    assert "━" in out or "─" in out
```

- [ ] **Step 4.3: Run — FAIL**

```bash
pytest tests/unit/test_ui_header.py::test_step_header_shows_step_number_and_total -v
```

기대: FAIL with `ModuleNotFoundError: No module named 'documatch.interaction.ui.header'`

- [ ] **Step 4.4: 구현** — `src/documatch/interaction/ui/header.py`:

```python
"""단계 헤더 + 단축키 legend 렌더링."""
from rich.console import Console
from rich.rule import Rule


def render_step_header(console: Console, *, step: int, total: int, name: str) -> None:
    """매 HITL 게이트 시작 시 단계 컨텍스트를 시각적으로 표시.

    예시 출력:
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
          Step 3 / 7  |  invoice_extraction
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    """
    title = f"  Step {step} / {total}  |  {name}"
    console.print()
    console.print(Rule(style="cyan"))
    console.print(title, style="bold")
    console.print(Rule(style="cyan"))
```

- [ ] **Step 4.5: Run — PASS**

```bash
pytest tests/unit/test_ui_header.py -v
```

기대: 2 passed.

- [ ] **Step 4.6: Commit**

```bash
git add src/documatch/interaction/ui/ tests/unit/test_ui_header.py
git commit -m "feat(ui): render_step_header — 단계 헤더 헬퍼"
```

---

### Task 5: header.py — render_action_legend

**Files:**
- Modify: `src/documatch/interaction/ui/header.py`
- Modify: `tests/unit/test_ui_header.py`

- [ ] **Step 5.1: 실패하는 테스트 추가** — `tests/unit/test_ui_header.py` 끝에 추가:

```python
def test_action_legend_renders_all_actions():
    from documatch.interaction.ui.header import render_action_legend
    console = Console(file=io.StringIO(), force_terminal=False, width=80)
    actions = [("A", "승인"), ("R", "피드백 수정"), ("Q", "종료")]
    render_action_legend(console, actions, title="선택")
    out = console.file.getvalue()
    assert "[A]" in out and "승인" in out
    assert "[R]" in out and "피드백 수정" in out
    assert "[Q]" in out and "종료" in out
    assert "선택" in out


def test_action_legend_uses_panel_box():
    from documatch.interaction.ui.header import render_action_legend
    console = Console(file=io.StringIO(), force_terminal=False, width=80)
    render_action_legend(console, [("A", "승인")], title="선택")
    out = console.file.getvalue()
    # Panel은 ╭ ╮ ╰ ╯ 또는 ┌ ┐ 등 박스 문자
    assert any(c in out for c in "╭╮╰╯┌┐└┘")
```

- [ ] **Step 5.2: Run — FAIL**

- [ ] **Step 5.3: 구현** — `src/documatch/interaction/ui/header.py` 끝에 추가:

```python
from rich.panel import Panel
from rich.text import Text


def render_action_legend(
    console: Console,
    actions: list[tuple[str, str]],
    *,
    title: str = "선택",
) -> None:
    """단축키 legend 박스. actions = [(key, description), ...].

    예시 출력:
        ╭───────── 선택 ─────────╮
        │  [A] 승인              │
        │  [R] 피드백 수정       │
        │  [Q] 종료              │
        ╰────────────────────────╯
    """
    body = Text()
    for i, (key, desc) in enumerate(actions):
        if i > 0:
            body.append("\n")
        body.append(f"  [{key}] ", style="bold cyan")
        body.append(desc)
    panel = Panel(body, title=title, title_align="left", border_style="cyan")
    console.print(panel)
```

- [ ] **Step 5.4: Run — PASS**

- [ ] **Step 5.5: Commit**

```bash
git add src/documatch/interaction/ui/header.py tests/unit/test_ui_header.py
git commit -m "feat(ui): render_action_legend — 단축키 박스"
```

---

### Task 6: progress.py — live_spinner

**Files:**
- Create: `src/documatch/interaction/ui/progress.py`
- Create: `tests/unit/test_ui_progress.py`

- [ ] **Step 6.1: 실패하는 테스트** — `tests/unit/test_ui_progress.py`:

```python
"""ui.progress — Live 스피너 + progress bar."""
import io
import time

from rich.console import Console

from documatch.interaction.ui.progress import live_spinner


def test_live_spinner_yields_handle():
    console = Console(file=io.StringIO(), force_terminal=False, width=80)
    with live_spinner(console, "테스트 작업 중") as spinner:
        assert spinner is not None
        # 짧게 대기해서 Live 갱신 한 번 일어나도록
        time.sleep(0.05)
    # context exit 후 출력에 메시지 흔적 있어야
    out = console.file.getvalue()
    # Live는 ANSI 처리에 따라 다양 — 메시지 텍스트는 항상 들어감
    assert "테스트 작업 중" in out or len(out) > 0


def test_live_spinner_can_update_message():
    console = Console(file=io.StringIO(), force_terminal=False, width=80)
    with live_spinner(console, "초기 메시지") as spinner:
        spinner.update("새 메시지")
    # 컨텍스트 종료 정상 작동
    assert True
```

- [ ] **Step 6.2: Run — FAIL**

- [ ] **Step 6.3: 구현** — `src/documatch/interaction/ui/progress.py`:

```python
"""Live 스피너 + Progress bar 헬퍼."""
import time
from contextlib import contextmanager
from typing import Iterator

from rich.console import Console
from rich.live import Live
from rich.spinner import Spinner
from rich.text import Text


class SpinnerHandle:
    """live_spinner context의 핸들 — update()로 메시지 변경."""

    def __init__(self, live: Live, spinner: Spinner) -> None:
        self._live = live
        self._spinner = spinner
        self._start = time.time()

    def update(self, message: str) -> None:
        """진행 메시지 변경."""
        self._spinner.text = self._format(message)
        self._live.update(self._spinner)

    def _format(self, message: str) -> Text:
        elapsed = time.time() - self._start
        return Text.from_markup(f"{message} [dim]({elapsed:.1f}s)[/dim]")


@contextmanager
def live_spinner(console: Console, message: str) -> Iterator[SpinnerHandle]:
    """Rich Live 스피너 — async 작업 중 시각적 피드백.

    예시:
        with live_spinner(console, "AI 호출 중") as sp:
            result = await llm.ainvoke(...)
            # 자동으로 스피너 + 경과 시간 표시
    """
    spinner = Spinner("dots", text=Text(message))
    with Live(spinner, console=console, refresh_per_second=10, transient=True) as live:
        handle = SpinnerHandle(live, spinner)
        # 초기 텍스트 적용
        handle.update(message)
        yield handle
```

- [ ] **Step 6.4: Run — PASS**

```bash
pytest tests/unit/test_ui_progress.py -v
```

- [ ] **Step 6.5: Commit**

```bash
git add src/documatch/interaction/ui/progress.py tests/unit/test_ui_progress.py
git commit -m "feat(ui): live_spinner — async 작업 진행 표시"
```

---

### Task 7: progress.py — progress_bar

**Files:**
- Modify: `src/documatch/interaction/ui/progress.py`
- Modify: `tests/unit/test_ui_progress.py`

- [ ] **Step 7.1: 실패하는 테스트 추가** — `tests/unit/test_ui_progress.py` 끝에:

```python
def test_progress_bar_advances_and_completes():
    from documatch.interaction.ui.progress import progress_bar
    console = Console(file=io.StringIO(), force_terminal=False, width=80)
    with progress_bar(console, total=3, label="추출") as pb:
        pb.advance(1, status="success", label="a.pdf")
        pb.advance(1, status="success", label="b.pdf")
        pb.advance(1, status="failed", label="c.pdf")
    out = console.file.getvalue()
    assert "추출" in out


def test_progress_bar_tracks_status_counts():
    from documatch.interaction.ui.progress import progress_bar
    console = Console(file=io.StringIO(), force_terminal=False, width=80)
    with progress_bar(console, total=2, label="추출") as pb:
        pb.advance(1, status="success", label="a")
        pb.advance(1, status="failed", label="b")
        # 카운트 검증
        assert pb.success_count == 1
        assert pb.failed_count == 1
```

- [ ] **Step 7.2: Run — FAIL**

- [ ] **Step 7.3: 구현** — `src/documatch/interaction/ui/progress.py` 끝에:

```python
from rich.progress import (
    BarColumn,
    Progress,
    TaskProgressColumn,
    TextColumn,
)


class ProgressBarHandle:
    """progress_bar context 핸들 — advance()로 진행 + 상태 카운트."""

    def __init__(self, progress: Progress, task_id: int) -> None:
        self._progress = progress
        self._task_id = task_id
        self.success_count = 0
        self.failed_count = 0
        self.review_count = 0

    def advance(self, n: int = 1, *, status: str = "success", label: str = "") -> None:
        if status == "success":
            self.success_count += n
        elif status == "failed":
            self.failed_count += n
        elif status == "needs_review":
            self.review_count += n
        desc = (
            f"{label}  ✓{self.success_count}  ✗{self.failed_count}  ⚠{self.review_count}"
        )
        self._progress.update(self._task_id, advance=n, description=desc)


@contextmanager
def progress_bar(
    console: Console,
    *,
    total: int,
    label: str = "진행 중",
) -> Iterator[ProgressBarHandle]:
    """배치 작업 진행률 bar."""
    progress = Progress(
        TextColumn("[bold cyan]{task.description}"),
        BarColumn(),
        TaskProgressColumn(),
        console=console,
        transient=False,
    )
    with progress:
        task_id = progress.add_task(label, total=total)
        handle = ProgressBarHandle(progress, task_id)
        yield handle
```

- [ ] **Step 7.4: Run — PASS**

- [ ] **Step 7.5: Commit**

```bash
git add src/documatch/interaction/ui/progress.py tests/unit/test_ui_progress.py
git commit -m "feat(ui): progress_bar — 배치 진행률 bar (성공/실패/검토 카운트)"
```

---

### Task 8: errors.py — render_error_panel + REMEDY_MAP

**Files:**
- Create: `src/documatch/interaction/ui/errors.py`
- Create: `tests/unit/test_ui_errors.py`

- [ ] **Step 8.1: 실패하는 테스트** — `tests/unit/test_ui_errors.py`:

```python
"""ui.errors — 에러 prominent 박스."""
import io

from rich.console import Console

from documatch.exceptions import LLMAuthError, LLMRateLimitError, ConfigError
from documatch.interaction.ui.errors import render_error_panel


def test_error_panel_includes_message():
    console = Console(file=io.StringIO(), force_terminal=False, width=80)
    e = LLMAuthError("invalid api key")
    render_error_panel(console, e)
    out = console.file.getvalue()
    assert "오류" in out
    assert "invalid api key" in out


def test_error_panel_includes_remedy_for_known_exception():
    console = Console(file=io.StringIO(), force_terminal=False, width=80)
    e = LLMAuthError("invalid")
    render_error_panel(console, e)
    out = console.file.getvalue()
    assert "documatch init" in out  # auth 오류 안내


def test_error_panel_unknown_exception_shows_generic():
    console = Console(file=io.StringIO(), force_terminal=False, width=80)
    render_error_panel(console, RuntimeError("unexpected"))
    out = console.file.getvalue()
    assert "unexpected" in out


def test_error_panel_red_border():
    console = Console(file=io.StringIO(), force_terminal=False, width=80)
    render_error_panel(console, RuntimeError("x"))
    out = console.file.getvalue()
    # 박스 문자 또는 ANSI red 코드 — 어느 쪽이든 visual 강조 흔적
    assert any(c in out for c in "╭╮╰╯") or "31m" in out  # ANSI red
```

- [ ] **Step 8.2: Run — FAIL**

- [ ] **Step 8.3: 구현** — `src/documatch/interaction/ui/errors.py`:

```python
"""에러 prominent 박스 — Rich Panel 빨강."""
from rich.console import Console
from rich.panel import Panel
from rich.text import Text

from documatch.exceptions import (
    ConfigError,
    DependencyError,
    ExtractionError,
    LLMAuthError,
    LLMError,
    LLMRateLimitError,
    ProcessorError,
    SpecError,
    SpecNotFoundError,
    UnsupportedFormatError,
    UserAbort,
)

REMEDY_MAP: dict[type[Exception], str] = {
    LLMAuthError: "documatch init으로 API 키를 재설정하세요.",
    LLMRateLimitError: "잠시 후 다시 시도하세요. 또는 사용량 한도를 확인하세요.",
    LLMError: "LLM 서버 문제일 수 있습니다. 잠시 후 재시도, 또는 documatch config show로 설정 점검.",
    ConfigError: "documatch config show로 현재 설정 확인 후 누락 값을 채우세요.",
    DependencyError: "필요한 시스템 의존성이 빠져 있습니다. README의 설치 가이드 참고.",
    UnsupportedFormatError: "이 파일 형식은 지원되지 않습니다. 지원 포맷: PDF/DOCX/XLSX/XLS/JPG/PNG/TXT/MD/CSV/HTML.",
    ProcessorError: "파일이 손상되었거나 처리 중 오류 발생. 다른 파일로 재시도.",
    SpecNotFoundError: "documatch spec list로 저장된 스펙 목록 확인.",
    SpecError: "스펙 정의가 잘못되었습니다. JSON 구조 점검.",
    ExtractionError: "LLM 응답 파싱 실패. 잠시 후 재시도하면 해결되는 경우가 많습니다.",
}


def render_error_panel(console: Console, exception: BaseException) -> None:
    """예외를 prominent 빨간 Panel로 표시 — 원인 + 조치 안내.

    예시 출력:
        ╭─ ⚠️  오류 ─────────────────────────────╮
        │                                        │
        │  LLMAuthError: invalid api key         │
        │                                        │
        │  조치: documatch init으로 API 키 재설정 │
        │                                        │
        ╰────────────────────────────────────────╯
    """
    exc_type = type(exception).__name__
    body = Text()
    body.append(f"{exc_type}: ", style="bold red")
    body.append(str(exception))
    body.append("\n\n")

    # 안내 매핑 — MRO를 따라 가장 가까운 매핑 선택
    remedy = "오류 메시지를 확인하고 documatch config show 또는 documatch init으로 점검하세요."
    for cls in type(exception).__mro__:
        if cls in REMEDY_MAP:
            remedy = REMEDY_MAP[cls]
            break
    body.append("조치: ", style="bold yellow")
    body.append(remedy)

    panel = Panel(body, title="⚠️  오류", title_align="left", border_style="red")
    console.print(panel)
```

- [ ] **Step 8.4: Run — PASS**

- [ ] **Step 8.5: Commit**

```bash
git add src/documatch/interaction/ui/errors.py tests/unit/test_ui_errors.py
git commit -m "feat(ui): render_error_panel + REMEDY_MAP — prominent 에러 박스"
```

---

### Task 9: tables.py — render_spec_table

**Files:**
- Create: `src/documatch/interaction/ui/tables.py`
- Create: `tests/unit/test_ui_tables.py`

- [ ] **Step 9.1: 실패하는 테스트** — `tests/unit/test_ui_tables.py`:

```python
"""ui.tables — 스펙·결과 표 색상·정렬."""
import io

from rich.console import Console

from documatch.interaction.ui.tables import render_spec_table
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)


def _spec():
    return ExtractionSpec(
        name="invoice", doc_type_hint="invoice",
        fields=[
            Field(name="invoice_no", description="송장 번호", type=FieldType.STRING, required=True),
            Field(name="amount", description="총액", type=FieldType.NUMBER, required=True),
            Field(name="memo", description="비고", type=FieldType.STRING, required=False),
        ],
        summary_template=SummaryTemplate(template="{invoice_no}"),
        output_table=[
            OutputColumn(field_name="invoice_no", excel_header="송장번호"),
            OutputColumn(field_name="amount", excel_header="금액"),
        ],
    )


def test_spec_table_lists_all_fields():
    console = Console(file=io.StringIO(), force_terminal=False, width=120)
    render_spec_table(console, _spec())
    out = console.file.getvalue()
    assert "invoice_no" in out
    assert "amount" in out
    assert "memo" in out


def test_spec_table_marks_required_fields():
    console = Console(file=io.StringIO(), force_terminal=False, width=120)
    render_spec_table(console, _spec())
    out = console.file.getvalue()
    # 필수 필드는 별표 ★ 또는 (필수) 표시
    # 단순 검증: "★" 또는 "필수" 포함
    assert "★" in out or "필수" in out


def test_spec_table_includes_field_types():
    console = Console(file=io.StringIO(), force_terminal=False, width=120)
    render_spec_table(console, _spec())
    out = console.file.getvalue()
    assert "string" in out
    assert "number" in out
```

- [ ] **Step 9.2: Run — FAIL**

- [ ] **Step 9.3: 구현** — `src/documatch/interaction/ui/tables.py`:

```python
"""스펙·결과 표 색상·정렬 헬퍼."""
from rich.console import Console
from rich.table import Table

from documatch.core.models import ExtractionResult
from documatch.spec.models import ExtractionSpec

_TYPE_STYLES = {
    "string": "cyan",
    "number": "green",
    "date": "yellow",
    "boolean": "magenta",
    "list": "blue",
}


def render_spec_table(console: Console, spec: ExtractionSpec) -> None:
    """스펙 필드 표 — 필수 필드 별표, 타입 색상.

    예시 출력:
        ┏━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━━┓
        ┃ #   ┃ 필드명       ┃ 타입   ┃ 설명          ┃
        ┡━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━━┩
        │ 1 ★ │ invoice_no   │ string │ 송장 번호     │
        │ 2 ★ │ amount       │ number │ 총액          │
        │ 3   │ memo         │ string │ 비고          │
        └─────┴──────────────┴────────┴───────────────┘
    """
    table = Table(show_header=True, header_style="bold cyan", title=f"스펙: {spec.name}")
    table.add_column("#", style="dim", width=4)
    table.add_column("필드명", style="bold")
    table.add_column("타입")
    table.add_column("설명", overflow="fold")
    for i, f in enumerate(spec.fields, 1):
        marker = "★" if f.required else " "
        idx_cell = f"{i} {marker}"
        type_style = _TYPE_STYLES.get(f.type.value, "white")
        type_cell = f"[{type_style}]{f.type.value}[/{type_style}]"
        table.add_row(idx_cell, f.name, type_cell, f.description)
    console.print(table)
    console.print(f"[dim]요약 템플릿: {spec.summary_template.template}[/dim]")
```

- [ ] **Step 9.4: Run — PASS**

- [ ] **Step 9.5: Commit**

```bash
git add src/documatch/interaction/ui/tables.py tests/unit/test_ui_tables.py
git commit -m "feat(ui): render_spec_table — 스펙 필드 표 (필수 별표, 타입 색상)"
```

---

### Task 10: tables.py — render_result_table (단일값 모드)

**Files:**
- Modify: `src/documatch/interaction/ui/tables.py`
- Modify: `tests/unit/test_ui_tables.py`

- [ ] **Step 10.1: 실패하는 테스트 추가** — `tests/unit/test_ui_tables.py`:

```python
def test_result_table_single_mode_shows_values_and_confidence():
    from documatch.core.models import ExtractionResult
    from documatch.interaction.ui.tables import render_result_table
    console = Console(file=io.StringIO(), force_terminal=False, width=120)
    spec = _spec()
    result = ExtractionResult(
        document_id="x.pdf",
        values={"invoice_no": "INV-001", "amount": 100, "memo": "test"},
        confidence={"invoice_no": 0.95, "amount": 0.45, "memo": 0.20},
    )
    render_result_table(console, result, spec)
    out = console.file.getvalue()
    assert "INV-001" in out
    assert "100" in out
    assert "0.95" in out


def test_result_table_low_confidence_marker():
    from documatch.core.models import ExtractionResult
    from documatch.interaction.ui.tables import render_result_table
    console = Console(file=io.StringIO(), force_terminal=False, width=120)
    spec = _spec()
    result = ExtractionResult(
        document_id="x.pdf",
        values={"invoice_no": "X"},
        confidence={"invoice_no": 0.30},
        needs_review=True,
        review_reasons=["Low confidence for invoice_no: 0.30"],
    )
    render_result_table(console, result, spec)
    out = console.file.getvalue()
    # 신뢰도 0.30 → 빨강/⚠️ 마크
    assert "⚠" in out or "0.30" in out
```

- [ ] **Step 10.2: Run — FAIL**

- [ ] **Step 10.3: 구현** — `src/documatch/interaction/ui/tables.py` 끝에 추가:

```python
def render_result_table(
    console: Console,
    result: ExtractionResult,
    spec: ExtractionSpec,
) -> None:
    """추출 결과 표 — 단일값/테이블 모드 자동 분기.

    단일값 모드 예시 (신뢰도 색상):
        ┏━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━┳━━━━━━━━┳━━━━━┓
        ┃ 필드        ┃ 값             ┃ 신뢰도 ┃     ┃
        ┡━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━╇━━━━━━━━╇━━━━━┩
        │ invoice_no  │ INV-001        │ 0.95   │ 🟢  │
        │ amount      │ 100            │ 0.45   │ 🟡  │
        │ memo        │ test           │ 0.20   │ 🔴⚠  │
        └─────────────┴────────────────┴────────┴─────┘
    """
    if spec.table_mode and result.is_table_result:
        _render_result_table_mode(console, result, spec)
    else:
        _render_result_single_mode(console, result, spec)


def _confidence_indicator(conf: float) -> tuple[str, str]:
    """(emoji, color) 반환."""
    if conf >= 0.8:
        return ("🟢", "green")
    if conf >= 0.5:
        return ("🟡", "yellow")
    return ("🔴", "red")


def _render_result_single_mode(
    console: Console,
    result: ExtractionResult,
    spec: ExtractionSpec,
) -> None:
    table = Table(show_header=True, header_style="bold cyan", title=f"추출 결과 — {result.document_id}")
    table.add_column("필드", style="bold")
    table.add_column("값", overflow="fold")
    table.add_column("신뢰도", justify="right")
    table.add_column("상태", justify="center")
    for f in spec.fields:
        value = str(result.values.get(f.name, ""))
        conf = result.confidence.get(f.name, 0.0)
        emoji, color = _confidence_indicator(conf)
        warn = " ⚠" if conf < 0.5 else ""
        status_cell = f"[{color}]{emoji}{warn}[/{color}]"
        conf_cell = f"[{color}]{conf:.2f}[/{color}]"
        table.add_row(f.name, value, conf_cell, status_cell)
    console.print(table)
    if result.summary_sentence:
        console.print(f"[dim]요약: {result.summary_sentence}[/dim]")
    if result.needs_review and result.review_reasons:
        console.print("[yellow]검토 필요 사유:[/yellow]")
        for reason in result.review_reasons:
            console.print(f"  • {reason}")


def _render_result_table_mode(
    console: Console,
    result: ExtractionResult,
    spec: ExtractionSpec,
) -> None:
    """테이블 모드 placeholder — Task 11에서 구현."""
    rows = result.rows or []
    table = Table(show_header=True, header_style="bold cyan", title=f"추출 결과 (행 {len(rows)}개) — {result.document_id}")
    table.add_column("#", style="dim", width=4)
    for f in spec.fields:
        table.add_column(f.name)
    for i, row in enumerate(rows, 1):
        cells = [str(i)] + [str(row.get(f.name, "")) for f in spec.fields]
        table.add_row(*cells)
    console.print(table)
```

- [ ] **Step 10.4: Run — PASS**

- [ ] **Step 10.5: Commit**

```bash
git add src/documatch/interaction/ui/tables.py tests/unit/test_ui_tables.py
git commit -m "feat(ui): render_result_table — 단일값/테이블 모드 분기 + 신뢰도 색상"
```

---

### Task 11: ui/__init__.py 공개 API export (trivial export, non-TDD)

**Files:**
- Modify: `src/documatch/interaction/ui/__init__.py`

- [ ] **Step 11.1: __init__.py 작성**

```python
"""UI helper public API."""
from documatch.interaction.ui.errors import render_error_panel, REMEDY_MAP
from documatch.interaction.ui.header import render_action_legend, render_step_header
from documatch.interaction.ui.progress import live_spinner, progress_bar
from documatch.interaction.ui.tables import render_result_table, render_spec_table

__all__ = [
    "render_step_header",
    "render_action_legend",
    "live_spinner",
    "progress_bar",
    "render_error_panel",
    "REMEDY_MAP",
    "render_spec_table",
    "render_result_table",
]
```

- [ ] **Step 11.2: import 검증**

```bash
python -c "from documatch.interaction.ui import (
    render_step_header, render_action_legend,
    live_spinner, progress_bar,
    render_error_panel, REMEDY_MAP,
    render_spec_table, render_result_table,
)
print('all UI helpers importable')"
```

기대: `all UI helpers importable`.

- [ ] **Step 11.3: 전체 테스트 — UI 모듈 전부 통과 확인**

```bash
pytest tests/unit/test_ui_*.py -v
```

기대: 약 12-15 테스트 통과 (Task 4-10 합산).

- [ ] **Step 11.4: Commit**

```bash
git add src/documatch/interaction/ui/__init__.py
git commit -m "feat(ui): __init__ public API export (8 헬퍼)"
```

Phase 2 완료. 4개 UI 헬퍼 모듈 + 테스트 모두 작성. 다음은 InteractiveReviewer 재작성.

---

## Phase 3 — InteractiveReviewer Rewrite (Tasks 12-17)

기존 `src/documatch/interaction/interactive.py`를 새 UI 헬퍼들 사용하도록 전면 재작성. 핵심 동작(분기·반환값)은 동일하게 유지하면서 시각 출력만 강화.

### Task 12: confirm_files 재작성

**Files:**
- Modify: `src/documatch/interaction/interactive.py`
- Modify: `tests/unit/test_reviewer_interactive.py`

- [ ] **Step 12.1: 기존 테스트 확인** — `tests/unit/test_reviewer_interactive.py`의 `test_confirm_files_*` 3개. 그대로 통과해야 (동작 변경 X).

- [ ] **Step 12.2: 새 검증 테스트 추가** — `tests/unit/test_reviewer_interactive.py` 끝에:

```python
@pytest.mark.asyncio
async def test_confirm_files_renders_step_header(monkeypatch):
    _patch_choice(monkeypatch, "a")
    console = _silent()
    r = InteractiveReviewer(console=console)
    await r.confirm_files(_entries(("x.pdf", "pdf")))
    out = console.file.getvalue()
    assert "Step 1" in out


@pytest.mark.asyncio
async def test_confirm_files_renders_action_legend(monkeypatch):
    _patch_choice(monkeypatch, "a")
    console = _silent()
    r = InteractiveReviewer(console=console)
    await r.confirm_files(_entries(("x.pdf", "pdf")))
    out = console.file.getvalue()
    assert "[A]" in out or "계속" in out  # legend 출력
```

- [ ] **Step 12.3: confirm_files 메서드 재작성** — `src/documatch/interaction/interactive.py`의 `confirm_files` 메서드 교체:

```python
    async def confirm_files(self, entries: list[FileEntry]) -> ReviewDecision:
        from documatch.interaction.ui import render_action_legend, render_step_header

        render_step_header(self.console, step=1, total=7, name="파일 스캔 결과")

        ext_counts: dict[str, int] = {}
        for e in entries:
            ext_counts[e.extension] = ext_counts.get(e.extension, 0) + 1
        self.console.print(f"[bold]{len(entries)}개 파일/시트 발견[/bold]")
        for ext, n in sorted(ext_counts.items()):
            self.console.print(f"  .{ext}: {n}")

        render_action_legend(
            self.console,
            [("A", "계속"), ("E", "확장자 필터"), ("Q", "종료")],
            title="선택",
        )

        c = choice(self.console, "선택", ["a", "e", "q"], "a")
        if c == "q":
            return ReviewDecision(action="quit")
        if c == "e":
            text = Prompt.ask("포함할 확장자 (쉼표 구분, 예: pdf,docx)")
            exts = [s.strip().lower() for s in text.split(",") if s.strip()]
            return ReviewDecision(action="filter", extension_filter=exts)
        return ReviewDecision(action="continue")
```

- [ ] **Step 12.4: Run 기존 + 신규 테스트**

```bash
pytest tests/unit/test_reviewer_interactive.py -v -k confirm_files
```

기대: 5 passed (기존 3 + 신규 2).

- [ ] **Step 12.5: Commit**

```bash
git add src/documatch/interaction/interactive.py tests/unit/test_reviewer_interactive.py
git commit -m "refactor(interactive): confirm_files — 단계 헤더 + legend 적용"
```

---

### Task 13: select_sample 재작성

**Files:**
- Modify: `src/documatch/interaction/interactive.py`
- Modify: `tests/unit/test_reviewer_interactive.py`

- [ ] **Step 13.1: 새 검증 테스트** — 기존 `test_select_sample_*` 6개 그대로 + 추가:

```python
@pytest.mark.asyncio
async def test_select_sample_renders_step_header(monkeypatch):
    _patch_prompt(monkeypatch, "1")
    console = _silent()
    r = InteractiveReviewer(console=console)
    await r.select_sample(_entries(("a.pdf", "pdf")))
    out = console.file.getvalue()
    assert "Step 2" in out
```

- [ ] **Step 13.2: select_sample 재작성**:

```python
    async def select_sample(self, entries: list[FileEntry]) -> Path | None:
        from documatch.interaction.ui import render_step_header

        render_step_header(self.console, step=2, total=7, name="샘플 선택")

        non_xlsx = [e for e in entries if e.extension in ("pdf", "docx", "txt", "md", "html", "htm")]
        candidates = non_xlsx if non_xlsx else entries
        self.console.print("[bold]스키마 생성용 샘플을 선택하세요:[/bold]")
        for i, e in enumerate(candidates, 1):
            label = e.filename
            if e.sheet_name:
                label += f" (sheet: {e.sheet_name})"
            self.console.print(f"  [{i}] {label}")
        ans = Prompt.ask("번호 (q=종료)", default="1")
        if ans.lower() == "q":
            raise UserAbort("사용자가 종료했습니다.")
        try:
            idx = int(ans) - 1
            if 0 <= idx < len(candidates):
                return Path(candidates[idx].file_path)
        except ValueError:
            pass
        return Path(candidates[0].file_path)
```

- [ ] **Step 13.3: Run — PASS**

```bash
pytest tests/unit/test_reviewer_interactive.py -v -k select_sample
```

- [ ] **Step 13.4: Commit**

```bash
git add src/documatch/interaction/interactive.py tests/unit/test_reviewer_interactive.py
git commit -m "refactor(interactive): select_sample — 단계 헤더 적용"
```

---

### Task 14: review_spec 재작성 (스펙 표 + legend + 에러 박스)

**Files:**
- Modify: `src/documatch/interaction/interactive.py`
- Modify: `tests/unit/test_reviewer_interactive.py`

- [ ] **Step 14.1: 새 검증 테스트**:

```python
@pytest.mark.asyncio
async def test_review_spec_renders_spec_table(monkeypatch):
    _patch_choice(monkeypatch, "a")
    console = _silent()
    r = InteractiveReviewer(console=console)
    await r.review_spec(_spec())
    out = console.file.getvalue()
    assert "Step 3" in out
    # 스펙 표에 필드명 포함
    assert "a" in out  # _spec()의 첫 필드명


@pytest.mark.asyncio
async def test_review_spec_renders_action_legend(monkeypatch):
    _patch_choice(monkeypatch, "a")
    console = _silent()
    r = InteractiveReviewer(console=console)
    await r.review_spec(_spec())
    out = console.file.getvalue()
    assert "[A]" in out  # legend
```

- [ ] **Step 14.2: review_spec 재작성** — 기존 `_render_spec` 메서드 제거하고 review_spec 본체에서 직접 ui helper 사용:

```python
    async def review_spec(
        self, spec: ExtractionSpec, available_specs: list[str] | None = None,
    ) -> SpecDecision:
        from documatch.interaction.ui import (
            render_action_legend, render_spec_table, render_step_header,
        )

        render_step_header(self.console, step=3, total=7, name=f"스펙 검토 — {spec.name}")
        render_spec_table(self.console, spec)

        actions = [
            ("A", "승인 + 다음"),
            ("R", "피드백 입력 후 재요청"),
            ("L", "저장된 스펙 불러오기"),
            ("B", "샘플 재선택"),
            ("Q", "종료"),
        ]
        render_action_legend(self.console, actions, title="선택")

        c = choice(self.console, "선택", ["a", "r", "l", "b", "q"], "a")
        if c == "q":
            return SpecDecision(action="quit")
        if c == "b":
            return SpecDecision(action="back")
        if c == "r":
            fb = multiline_input(self.console, "수정 요청")
            return SpecDecision(action="refine", feedback=fb)
        if c == "l":
            if not available_specs:
                self.console.print("[yellow]저장된 스펙이 없습니다.[/yellow]")
                return await self.review_spec(spec, available_specs)
            for i, n in enumerate(available_specs, 1):
                self.console.print(f"  [{i}] {n}")
            ans = Prompt.ask("번호", default="1")
            try:
                idx = int(ans) - 1
                if 0 <= idx < len(available_specs):
                    return SpecDecision(action="load", spec_name=available_specs[idx])
            except ValueError:
                pass
            return SpecDecision(action="back")
        return SpecDecision(action="approve")
```

기존 `_render_spec` 메서드는 별도로 제거하지 말고 그대로 두되 (다른 곳에서 호출 안 함), Phase 3 끝에서 일괄 정리.

- [ ] **Step 14.3: Run — PASS**

```bash
pytest tests/unit/test_reviewer_interactive.py -v -k review_spec
```

- [ ] **Step 14.4: Commit**

```bash
git add src/documatch/interaction/interactive.py tests/unit/test_reviewer_interactive.py
git commit -m "refactor(interactive): review_spec — 헤더 + 스펙 표 + legend"
```

---

### Task 15: confirm_mode + review_sample 재작성

**Files:**
- Modify: `src/documatch/interaction/interactive.py`
- Modify: `tests/unit/test_reviewer_interactive.py`

- [ ] **Step 15.1: 새 검증 테스트**:

```python
@pytest.mark.asyncio
async def test_confirm_mode_renders_step_header(monkeypatch):
    _patch_choice(monkeypatch, "s")
    console = _silent()
    r = InteractiveReviewer(console=console)
    await r.confirm_mode(_spec())
    out = console.file.getvalue()
    assert "Step 4" in out


@pytest.mark.asyncio
async def test_review_sample_renders_result_table(monkeypatch):
    from documatch.core.models import ExtractionResult
    _patch_choice(monkeypatch, "a")
    console = _silent()
    r = InteractiveReviewer(console=console)
    result = ExtractionResult(
        document_id="x.pdf",
        values={"a": "v"},
        confidence={"a": 0.9},
    )
    await r.review_sample(result, _spec())
    out = console.file.getvalue()
    assert "Step 5" in out
    assert "v" in out  # 추출값
    assert "0.9" in out  # 신뢰도
```

- [ ] **Step 15.2: confirm_mode 재작성**:

```python
    async def confirm_mode(self, spec: ExtractionSpec) -> ModeDecision:
        from documatch.interaction.ui import render_action_legend, render_step_header

        render_step_header(self.console, step=4, total=7, name="추출 모드 확정")
        rec = "테이블" if spec.table_mode else "단일값"
        self.console.print(f"[bold]AI 추천 모드: {rec}[/bold]")
        self.console.print("  • 단일값: 한 문서 = Excel 한 행")
        self.console.print("  • 테이블: 한 문서 = Excel 여러 행 (명세서 등)")

        render_action_legend(
            self.console,
            [("S", "단일값"), ("T", "테이블"), ("B", "스펙으로 돌아가기")],
            title="선택",
        )

        c = choice(
            self.console, "선택", ["s", "t", "b"],
            "t" if spec.table_mode else "s",
        )
        if c == "b":
            return ModeDecision(action="back")
        return ModeDecision(action="table" if c == "t" else "single")
```

- [ ] **Step 15.3: review_sample 재작성**:

```python
    async def review_sample(
        self, result: ExtractionResult, spec: ExtractionSpec,
    ) -> SampleDecision:
        from documatch.interaction.ui import (
            render_action_legend, render_result_table, render_step_header,
        )

        render_step_header(self.console, step=5, total=7, name="샘플 결과 검토")
        render_result_table(self.console, result, spec)

        actions = [
            ("A", "전체 진행"),
            ("S", "이 건만 저장"),
            ("R", "피드백 입력 후 재요청"),
            ("B", "스펙으로 돌아가기"),
            ("Q", "종료"),
        ]
        render_action_legend(self.console, actions, title="선택")

        c = choice(self.console, "선택", ["a", "s", "r", "b", "q"], "a")
        if c == "q":
            return SampleDecision(action="quit")
        if c == "b":
            return SampleDecision(action="back")
        if c == "s":
            return SampleDecision(action="single_save")
        if c == "r":
            fb = multiline_input(self.console, "수정 요청")
            return SampleDecision(action="refine", feedback=fb)
        return SampleDecision(action="full")
```

- [ ] **Step 15.4: Run — PASS**

```bash
pytest tests/unit/test_reviewer_interactive.py -v -k "confirm_mode or review_sample"
```

- [ ] **Step 15.5: Commit**

```bash
git add src/documatch/interaction/interactive.py tests/unit/test_reviewer_interactive.py
git commit -m "refactor(interactive): confirm_mode + review_sample — 헤더+결과 표+legend"
```

---

### Task 16: review_batch_results — 진행률 bar + summary

**Files:**
- Modify: `src/documatch/interaction/interactive.py`
- Modify: `tests/unit/test_reviewer_interactive.py`

- [ ] **Step 16.1: 새 검증 테스트**:

```python
@pytest.mark.asyncio
async def test_review_batch_results_renders_step_header(monkeypatch):
    _patch_choice(monkeypatch, "y")
    console = _silent()
    r = InteractiveReviewer(console=console)
    d = await r.review_batch_results([], _spec())
    out = console.file.getvalue()
    assert "Step 6" in out
    assert d.action == "save"


@pytest.mark.asyncio
async def test_review_batch_results_summarizes_counts(monkeypatch):
    from documatch.core.models import ExtractionResult
    _patch_choice(monkeypatch, "y")
    console = _silent()
    r = InteractiveReviewer(console=console)
    results = [
        ExtractionResult(document_id="a", status="success"),
        ExtractionResult(document_id="b", status="failed"),
        ExtractionResult(document_id="c", status="success", needs_review=True),
    ]
    await r.review_batch_results(results, _spec())
    out = console.file.getvalue()
    # 성공 2 / 실패 1 / 검토 1 카운트
    assert "2" in out  # success count
    assert "1" in out  # failed
```

- [ ] **Step 16.2: review_batch_results 재작성**:

```python
    async def review_batch_results(
        self, results: list[ExtractionResult], spec: ExtractionSpec,
    ) -> BatchResultDecision:
        from documatch.interaction.ui import (
            render_action_legend, render_step_header,
        )

        render_step_header(self.console, step=6, total=7, name="배치 결과 검토")

        successful = sum(1 for r in results if r.status == "success")
        failed = sum(1 for r in results if r.status == "failed")
        review = sum(1 for r in results if r.needs_review)
        self.console.print(
            f"[bold]배치 완료[/bold] — "
            f"[green]성공 {successful}[/green] / "
            f"[red]실패 {failed}[/red] / "
            f"[yellow]검토필요 {review}[/yellow]"
        )

        render_action_legend(
            self.console,
            [("Y", "저장 (Excel 출력)"), ("N", "재배치"), ("Q", "종료")],
            title="선택",
        )

        c = choice(self.console, "선택", ["y", "n", "q"], "y")
        if c == "q":
            return BatchResultDecision(action="quit")
        if c == "n":
            return BatchResultDecision(action="back")
        return BatchResultDecision(action="save")
```

- [ ] **Step 16.3: Run — PASS**

```bash
pytest tests/unit/test_reviewer_interactive.py -v -k batch_results
```

- [ ] **Step 16.4: Commit**

```bash
git add src/documatch/interaction/interactive.py tests/unit/test_reviewer_interactive.py
git commit -m "refactor(interactive): review_batch_results — 헤더 + 색상 카운트 + legend"
```

---

### Task 17: show_progress(progress_bar 사용) + 잔존 헬퍼 정리 + 전체 회귀

**Files:**
- Modify: `src/documatch/interaction/interactive.py`
- Modify: `tests/unit/test_reviewer_interactive.py`

- [ ] **Step 17.1: 실패 테스트 추가** — `tests/unit/test_reviewer_interactive.py`에 추가:

```python
@pytest.mark.asyncio
async def test_show_progress_uses_progress_bar(monkeypatch):
    console = _silent()
    r = InteractiveReviewer(console=console)
    await r.show_progress(1, 3, "a.pdf")
    await r.show_progress(2, 3, "b.pdf")
    await r.show_progress(3, 3, "c.pdf")
    out = console.file.getvalue()
    # progress_bar는 Rich Progress의 표시 — % 또는 [#/total]이 보여야 함
    assert "3" in out  # total
    assert "a.pdf" in out or "b.pdf" in out or "c.pdf" in out
```

- [ ] **Step 17.2: show_progress / show_message 재작성** — Task 7의 `progress_bar` helper를 stateful하게 사용:

```python
    async def show_progress(self, current: int, total: int, label: str) -> None:
        # 첫 호출(current=1)에 progress_bar 시작, 마지막 호출(current==total)에 종료
        from documatch.interaction.ui import progress_bar
        if current == 1:
            self._progress_ctx = progress_bar(self.console, total=total, label="배치 추출")
            self._progress_handle = self._progress_ctx.__enter__()
        if hasattr(self, "_progress_handle") and self._progress_handle is not None:
            self._progress_handle.advance(1, status="success", label=label)
            if current >= total:
                self._progress_ctx.__exit__(None, None, None)
                self._progress_handle = None
                self._progress_ctx = None

    async def show_message(self, level: Literal["info", "warn", "error"], text: str) -> None:
        if level == "error":
            from documatch.interaction.ui import render_error_panel
            render_error_panel(self.console, RuntimeError(text))
            return
        color = {"info": "cyan", "warn": "yellow"}[level]
        self.console.print(f"[{color}]{text}[/{color}]")
```

또한 `__init__`에 `self._progress_handle = None; self._progress_ctx = None` 초기화 추가.

**참고**: 실 LLM 호출 site (spec generation/refine, sample extraction)에 `live_spinner` 통합은 engine 변경 필요라서 v0.3.x로 보류. 본 release는 batch progress만 progress_bar 적용.

- [ ] **Step 17.3: 더 이상 사용 안 하는 helper 메서드 제거**

`src/documatch/interaction/interactive.py`의 `InteractiveReviewer` 클래스에서 다음 메서드를 통째로 삭제 (Task 14에서 review_spec, Task 15에서 review_sample이 직접 ui helper를 호출하므로 미사용):
- `_render_spec(self, spec)` — 클래스 안 정의된 메서드
- `_render_result(self, r, spec)` — 클래스 안 정의된 메서드

삭제 후 `grep -n "_render_spec\|_render_result" src/documatch/interaction/interactive.py`로 잔존 참조 0건 확인.

- [ ] **Step 17.4: 사용 안 하는 import 정리**

```python
# 파일 상단 import 정리 — Panel, Table, Text 등 직접 사용 안 하면 삭제
# 최종 import:
from datetime import datetime
from pathlib import Path
from typing import Literal

from rich.console import Console
from rich.prompt import Prompt

from documatch.core.models import ExtractionResult
from documatch.exceptions import UserAbort
from documatch.interaction.prompts import choice, multiline_input
from documatch.interaction.reviewer import (
    BatchResultDecision,
    ModeDecision,
    ReviewDecision,
    SampleDecision,
    SpecDecision,
)
from documatch.processors.base import FileEntry
from documatch.spec.models import ExtractionSpec
```

(클래스 내부에서 ui helpers를 import하기 때문에 상단 import에서는 제거.)

- [ ] **Step 17.5: 전체 회귀**

```bash
pytest -q
ruff check src/ tests/
```

기대: ~169 + 추가된 ~12 신규 테스트 = 약 181 통과. ruff clean.

- [ ] **Step 17.6: Commit**

```bash
git add src/documatch/interaction/interactive.py tests/unit/test_reviewer_interactive.py
git commit -m "refactor(interactive): show_progress + 잔존 헬퍼 정리 — Phase 3 완료"
```

Phase 3 완료. InteractiveReviewer 모든 게이트가 새 UI helper를 사용함.

---

## Phase 4 — `documatch preview` 서브커맨드 (Tasks 18-20)

### Task 18: preview_command 모듈 + app.py 등록 (한 커밋, TDD)

**Files:**
- Create: `src/documatch/cli/preview_command.py`
- Modify: `src/documatch/cli/app.py`
- Create: `tests/integration/test_cli_preview_command.py`

- [ ] **Step 18.1: 실패하는 테스트** — `tests/integration/test_cli_preview_command.py`:

```python
"""documatch preview <file> 서브커맨드."""
from pathlib import Path

from click.testing import CliRunner

from documatch.cli.app import main


def test_preview_pdf_fixture(tmp_path):
    fixture = Path("tests/fixtures/docs/text.pdf")
    if not fixture.exists():
        import pytest
        pytest.skip("fixture not available")
    r = CliRunner().invoke(main, ["preview", str(fixture)])
    assert r.exit_code == 0
    assert "PDF" in r.output or "페이지" in r.output


def test_preview_text_file(tmp_path):
    f = tmp_path / "x.txt"
    f.write_text("첫째 줄\n둘째 줄\n셋째 줄\n")
    r = CliRunner().invoke(main, ["preview", str(f)])
    assert r.exit_code == 0
    assert "첫째 줄" in r.output
    assert "텍스트 추출 가능: yes" in r.output


def test_preview_missing_file_errors(tmp_path):
    r = CliRunner().invoke(main, ["preview", str(tmp_path / "nonexistent.pdf")])
    assert r.exit_code != 0


def test_preview_xlsx_shows_sheet_names_and_dimensions(tmp_path):
    fixture = Path("tests/fixtures/docs/sample.xlsx")
    if not fixture.exists():
        import pytest
        pytest.skip("fixture not available")
    r = CliRunner().invoke(main, ["preview", str(fixture)])
    assert r.exit_code == 0
    assert "시트:" in r.output or "Sheet" in r.output


def test_preview_image_shows_dimensions(tmp_path):
    fixture = Path("tests/fixtures/docs/sample.png")
    if not fixture.exists():
        import pytest
        pytest.skip("fixture not available")
    r = CliRunner().invoke(main, ["preview", str(fixture)])
    assert r.exit_code == 0
    assert "크기:" in r.output or "x" in r.output  # WxH 표시
```

- [ ] **Step 18.2: Run — FAIL**

- [ ] **Step 18.3: 구현** — `src/documatch/cli/preview_command.py`:

```python
"""documatch preview <file> — 파일 첫 50줄 + 메타 정보."""
from pathlib import Path

import click

from documatch.processors.registry import get_processor

_PREVIEW_LINES = 50


@click.command("preview")
@click.argument("file_path", type=click.Path(exists=True, path_type=Path))
def preview_command(file_path: Path) -> None:
    """파일을 텍스트로 변환해 첫 50줄 + 메타 정보를 출력합니다.

    추출 시작 전 빠른 점검 용도. LLM 호출 0회.
    """
    import asyncio

    ext = file_path.suffix.lstrip(".").lower()
    try:
        proc = get_processor(ext)
    except Exception as e:
        raise click.ClickException(f"지원하지 않는 파일: {e}") from e

    content = file_path.read_bytes()
    doc = asyncio.run(proc.process(content, file_path.name))

    click.echo(f"파일: {file_path.name}")
    click.echo(f"형식: {ext.upper()}, 페이지/시트: {doc.page_count}, 크기: {len(content):,}B")

    # 텍스트 추출 가능 여부
    has_text = bool(doc.raw_text.strip())
    click.echo(f"텍스트 추출 가능: {'yes' if has_text else 'no'}")

    # 스캔 PDF 여부
    if ext == "pdf":
        click.echo(f"스캔 PDF: {'yes' if doc.is_scanned else 'no'}")

    # XLSX: 시트 목록 + 각 시트 row/col
    if ext == "xlsx":
        from openpyxl import load_workbook
        wb = load_workbook(file_path, read_only=True, data_only=True)
        try:
            sheet_lines = []
            for sheet_name in wb.sheetnames:
                ws = wb[sheet_name]
                sheet_lines.append(f"  - {sheet_name}: {ws.max_row}행 × {ws.max_column}열")
            click.echo(f"시트: {len(wb.sheetnames)}개")
            for line in sheet_lines:
                click.echo(line)
        finally:
            wb.close()

    # 이미지: width × height
    if ext in ("jpg", "jpeg", "png"):
        from PIL import Image
        with Image.open(file_path) as img:
            click.echo(f"크기: {img.width} x {img.height}")

    if doc.metadata:
        click.echo(f"메타: {doc.metadata}")
    click.echo()
    click.echo("─" * 60)

    text = doc.raw_text
    if not has_text:
        click.echo("[텍스트 추출 결과 비어있음]")
        return
    text_lines = text.splitlines()[:_PREVIEW_LINES]
    for line in text_lines:
        click.echo(line)
    if len(text.splitlines()) > _PREVIEW_LINES:
        click.echo(f"\n... ({len(text.splitlines()) - _PREVIEW_LINES}줄 더 있음)")
```

- [ ] **Step 18.4: app.py 등록** — `src/documatch/cli/app.py` 끝부분의 다른 `main.add_command(...)` 옆에:

```python
from documatch.cli.preview_command import preview_command  # noqa: E402

main.add_command(preview_command)
```

- [ ] **Step 18.5: Run — PASS (등록까지 한 번에 끝나서 모든 테스트 통과)**

```bash
pytest tests/integration/test_cli_preview_command.py -v
```

기대: 5 passed.

- [ ] **Step 18.6: Commit**

```bash
git add src/documatch/cli/preview_command.py src/documatch/cli/app.py tests/integration/test_cli_preview_command.py
git commit -m "feat(cli): preview <file> 명령 + 메타데이터(시트/이미지/텍스트추출) + app 등록"
```

---

### Task 19: preview 수동 스모크 + 통합 검증 (verification, non-TDD)

**Files:**
- Modify: `tests/integration/test_cli_preview_command.py` (선택 — 통합 검증 추가)

- [ ] **Step 19.1: --help에 preview 노출 확인** — `tests/integration/test_cli_preview_command.py`에 추가:

```python
def test_preview_help_shows_in_main_help():
    r = CliRunner().invoke(main, ["--help"])
    assert r.exit_code == 0
    assert "preview" in r.output
```

- [ ] **Step 19.2: Run — PASS**

```bash
pytest tests/integration/test_cli_preview_command.py::test_preview_help_shows_in_main_help -v
```

- [ ] **Step 19.3: 수동 smoke (선택)**

```bash
cd /Users/yjban/Desktop/documatch-cli && source .venv/bin/activate
documatch preview tests/fixtures/docs/text.pdf
documatch preview tests/fixtures/docs/sample.xlsx
documatch preview tests/fixtures/docs/sample.png
```

기대: 각각 PDF 페이지/시트 카운트/이미지 dimension 표시, 첫 50줄 출력.

- [ ] **Step 19.4: Commit**

```bash
git add tests/integration/test_cli_preview_command.py
git commit -m "test(cli): preview 명령이 main --help에 노출 검증"
```

---

### Task 20: Phase 4 회귀 테스트 (verification, non-TDD)

**Files:** 없음 — 검증만.

- [ ] **Step 20.1: 전체 회귀**

```bash
cd /Users/yjban/Desktop/documatch-cli && source .venv/bin/activate
pytest -q
ruff check src/ tests/
```

기대: 모든 테스트 통과 + ruff clean.

문제 발견 시 즉시 수정 후 commit.

Phase 4 완료. `documatch preview <file>` 사용 가능 (PDF/DOCX/XLSX/이미지/텍스트 모두 메타 + 텍스트 50줄).

---

## Phase 5 — `--quiet/--json/--summary` 플래그 (Tasks 21-23)

### Task 21: --quiet 플래그 — 진행 메시지 억제

**Files:**
- Modify: `src/documatch/cli/app.py`
- Modify: `src/documatch/cli/_runner.py`
- Modify: `src/documatch/interaction/automation.py`
- Create: `tests/integration/test_cli_quiet_flag.py`

- [ ] **Step 21.1: 실패하는 테스트** — `tests/integration/test_cli_quiet_flag.py`:

```python
"""--quiet 플래그 — 진행 메시지 억제."""
import json
from pathlib import Path
from unittest.mock import patch

import pytest
from click.testing import CliRunner

from documatch.cli.app import main
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)
from documatch.spec.store import SpecStore
from tests.fakes import FakeLLMClient


@pytest.fixture
def setup(tmp_path, monkeypatch):
    (tmp_path / "a.txt").write_text("hi")
    out = tmp_path / "out.xlsx"
    spec_dir = tmp_path / "specs"
    monkeypatch.setenv("DOCUMATCH_SPEC_STORE_DIR", str(spec_dir))
    monkeypatch.setenv("ANTHROPIC_API_KEY", "x")
    spec = ExtractionSpec(
        name="x", doc_type_hint="x",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )
    SpecStore(base_dir=spec_dir).save(spec)

    response = json.dumps({"values": {"a": "x"}, "confidence": {"a": 0.9}, "summary_sentence": "x"})
    fake = FakeLLMClient(responses=[response] * 50)
    monkeypatch.setattr("documatch.cli._runner.create_llm", lambda s, *, role: fake)
    return tmp_path, out


def test_quiet_suppresses_progress(setup):
    tmp, out = setup
    r = CliRunner().invoke(main, [
        str(tmp), "--spec", "x", "--output", str(out),
        "--auto-approve", "--quiet",
    ])
    assert r.exit_code == 0, r.output
    # 진행 메시지(저장됨, etc) 출력에 없어야 함
    assert "저장됨" not in r.output
```

- [ ] **Step 21.2: Run — FAIL**

- [ ] **Step 21.3: 구현** — `src/documatch/cli/app.py`의 `main` 함수 데코레이터에 `--quiet` 추가:

```python
@click.option("--quiet", is_flag=True, default=False, help="진행 메시지 억제 (--auto-approve와 함께)")
```

`main` 함수 시그니처에 `quiet: bool` 매개변수 추가. `run_pipeline` 호출 시 전달.

`src/documatch/cli/_runner.py`의 `run_pipeline` 시그니처에 `quiet: bool = False` 추가, `AutomationReviewer` 생성 시 전달:

```python
if auto_approve:
    reviewer: Reviewer = AutomationReviewer(output_path=output or Path("./out.xlsx"), quiet=quiet)
```

`src/documatch/interaction/automation.py`의 `AutomationReviewer.__init__`에 `quiet=False` 추가, `show_progress`/`show_message`에서 quiet이면 출력 억제:

```python
class AutomationReviewer:
    def __init__(self, *, output_path: Path, quiet: bool = False) -> None:
        self.output_path = output_path
        self.quiet = quiet

    async def show_progress(self, current: int, total: int, label: str) -> None:
        if self.quiet:
            return
        # 기존 동작 유지
        pass

    async def show_message(self, level: Literal["info", "warn", "error"], text: str) -> None:
        if self.quiet and level == "info":
            return
        import sys
        stream = sys.stderr if level in ("warn", "error") else sys.stdout
        print(f"[{level}] {text}", file=stream)
```

`engine.run`이 호출하는 `show_message("info", f"저장됨: {out_path}")`도 억제됨.

- [ ] **Step 21.4: Run — PASS**

- [ ] **Step 21.5: Commit**

```bash
git add src/documatch/cli/app.py src/documatch/cli/_runner.py src/documatch/interaction/automation.py tests/integration/test_cli_quiet_flag.py
git commit -m "feat(cli): --quiet 플래그 (진행 메시지 억제)"
```

---

### Task 22: --summary 플래그 — 한 줄 요약 출력

**Files:**
- Modify: `src/documatch/cli/app.py`
- Modify: `src/documatch/cli/_runner.py`
- Modify: `tests/integration/test_cli_quiet_flag.py`

- [ ] **Step 22.1: 새 테스트 추가** — `test_cli_quiet_flag.py` 끝에:

```python
def test_summary_outputs_one_line(setup):
    tmp, out = setup
    r = CliRunner().invoke(main, [
        str(tmp), "--spec", "x", "--output", str(out),
        "--auto-approve", "--summary",
    ])
    assert r.exit_code == 0, r.output
    # 한 줄 요약 출력
    assert "successful" in r.output or "성공" in r.output
```

- [ ] **Step 22.2: app.py에 --summary 추가**:

```python
@click.option("--summary", is_flag=True, default=False, help="추출 끝에 한 줄 요약 출력")
```

`main` 시그니처에 `summary: bool` 추가, run_pipeline에 전달.

- [ ] **Step 22.3: _runner.py에 summary 출력 추가** — `run_pipeline` 끝부분:

```python
def run_pipeline(*, ..., summary: bool = False) -> int:
    # ... 기존 코드 ...
    
    failed = sum(1 for r in final.results if r.status == "failed")
    review_count = sum(1 for r in final.results if r.needs_review)
    successful = sum(1 for r in final.results if r.status == "success")

    if summary:
        avg_conf = 0.0
        if final.results:
            confs = [
                sum(r.confidence.values()) / max(len(r.confidence), 1)
                for r in final.results if r.confidence
            ]
            avg_conf = sum(confs) / max(len(confs), 1)
        print(
            f"{successful} successful, {failed} failed, "
            f"{review_count} need review, avg confidence {avg_conf:.2f}"
        )
    
    # ... 기존 fail-on-review/error 처리 ...
```

- [ ] **Step 22.4: Run — PASS**

- [ ] **Step 22.5: Commit**

```bash
git add src/documatch/cli/app.py src/documatch/cli/_runner.py tests/integration/test_cli_quiet_flag.py
git commit -m "feat(cli): --summary 플래그 (한 줄 요약 출력)"
```

---

### Task 23: --json 플래그 — JSON으로 stdout 출력

**Files:**
- Modify: `src/documatch/cli/app.py`
- Modify: `src/documatch/cli/_runner.py`
- Modify: `tests/integration/test_cli_quiet_flag.py`

- [ ] **Step 23.1: 새 테스트 추가**:

```python
def test_json_outputs_results_to_stdout_and_skips_excel(setup):
    import json as _json
    tmp, out = setup
    r = CliRunner().invoke(main, [
        str(tmp), "--spec", "x", "--output", str(out),
        "--auto-approve", "--json",
    ])
    assert r.exit_code == 0, r.output
    # stdout에 JSON 출력 (마지막 줄)
    last_lines = [line for line in r.output.splitlines() if line.strip().startswith("{")]
    assert last_lines, f"no JSON line in output: {r.output}"
    parsed = _json.loads(last_lines[-1])
    assert "results" in parsed
    assert isinstance(parsed["results"], list)
    # spec 명세: --json 사용 시 Excel 저장 X
    assert not out.exists(), f"Excel 저장됐음: {out}"
```

- [ ] **Step 23.2: app.py에 --json 추가**:

```python
@click.option("--json", "json_output", is_flag=True, default=False, help="결과를 JSON으로 stdout 출력 (Excel 외)")
```

`main` 시그니처에 `json_output: bool` 추가, run_pipeline에 전달.

- [ ] **Step 23.3: _runner.py에 JSON 출력 추가 + Excel 저장 skip** — engine.run을 호출하기 전·후 흐름 변경. 핵심:
  - `--json` 활성 시 engine 내부의 `exporter.write` 호출을 우회해야 함
  - 가장 단순한 방법: `--json` 활성 시 `output` 경로를 임시 파일로 두고 engine 호출, 끝나면 임시 파일 삭제 + JSON stdout. 또는 engine을 `_run_engine_no_export` 식으로 분기. 후자가 깔끔.

`engine.py`는 변경 X 원칙이지만 `--json`은 export skip이 본질적 변경이라 `_runner.py`에서 처리:

```python
def run_pipeline(*, ..., json_output: bool = False) -> int:
    # ... 기존 코드 (preloaded_spec 로드, reviewer 생성 등) ...

    engine = build_engine(settings, reviewer=reviewer)

    if json_output:
        # Excel 저장 우회: tmp 파일에 저장 후 즉시 삭제
        import tempfile
        with tempfile.NamedTemporaryFile(suffix=".xlsx", delete=False) as tmp_xlsx:
            tmp_path = Path(tmp_xlsx.name)
        try:
            final = asyncio.run(engine.run(
                scan_path=scan_path, preloaded_spec=preloaded, default_output=tmp_path,
            ))
        except UserAbort:
            tmp_path.unlink(missing_ok=True)
            return 130
        except ConfigError as e:
            tmp_path.unlink(missing_ok=True)
            print(f"설정 오류: {e}")
            return 2
        # 임시 Excel 파일 삭제 (JSON 출력만 의미 있음)
        tmp_path.unlink(missing_ok=True)
        if final.output_path and final.output_path.exists() and final.output_path == tmp_path:
            pass  # 이미 위에서 삭제
    else:
        try:
            final = asyncio.run(engine.run(
                scan_path=scan_path, preloaded_spec=preloaded, default_output=output,
            ))
        except UserAbort:
            return 130
        except ConfigError as e:
            print(f"설정 오류: {e}")
            return 2

    if json_output:
        import json as _json
        serialized = []
        for r in final.results:
            serialized.append({
                "document_id": r.document_id,
                "values": r.values,
                "confidence": r.confidence,
                "summary_sentence": r.summary_sentence,
                "needs_review": r.needs_review,
                "review_reasons": r.review_reasons,
                "status": r.status,
                "rows": r.rows,
                "row_confidences": r.row_confidences,
            })
        print(_json.dumps({"results": serialized, "spec": final.spec.name}))

    # ... 기존 fail-on-review/error/summary 처리 ...
```

**참고**: 기존 `try/except` 블록은 if/else 분기 하나로 통합되었음. AutomationReviewer가 받은 `output` 인자도 임시 파일을 받게 됨 — 사용자가 명시한 `--output`은 `--json`과 함께 쓸 때 무시됨 (CHANGELOG에 명시).

- [ ] **Step 23.4: Run — PASS**

- [ ] **Step 23.5: Commit**

```bash
git add src/documatch/cli/app.py src/documatch/cli/_runner.py tests/integration/test_cli_quiet_flag.py
git commit -m "feat(cli): --json 플래그 (결과 JSON으로 stdout 출력)"
```

Phase 5 완료. CI/스크립팅용 자동화 플래그 3종 작동.

---

## Phase 6 — Documentation + Release (Tasks 24-26)

### Task 24: README + CHANGELOG 업데이트 (docs, non-TDD)

**Files:**
- Modify: `README.md`
- Modify: `CHANGELOG.md`

- [ ] **Step 24.1: README — GUI 섹션 제거 + CLI 개선 사항 강조**

기존 README의 "Quick Start (GUI — 추천)" 섹션 통째로 삭제. CLI Quick Start만 남김.

상단 introduction을:
```markdown
# documatch-cli

AI 기반 문서 추출 CLI — PDF/DOCX/XLSX/이미지/HTML 등에서 원하는 항목을 LLM으로 추출해 Excel로 저장합니다.

v0.3.0부터 CLI 전용. Rich 기반 풍부한 시각 피드백 + 자동화 플래그(`--quiet`, `--summary`, `--json`).
```

새 섹션 추가 (CLI Quick Start 아래):

```markdown
## 빠른 점검: documatch preview

LLM 호출 없이 파일 내용을 빠르게 미리보기:

```bash
documatch preview my-invoice.pdf
documatch preview my-invoice.pdf --lines 100  # 더 많은 줄
```

## 자동화 플래그

```bash
# 진행 메시지 억제 (CI/배치)
documatch ./samples --spec invoice --output result.xlsx --auto-approve --quiet

# 한 줄 요약만
documatch ./samples --spec invoice --output result.xlsx --auto-approve --summary

# JSON으로 stdout 출력 (Excel 저장 + JSON 둘 다)
documatch ./samples --spec invoice --output result.xlsx --auto-approve --json
```
```

GUI 다운로드 안내, 시스템 요구사항 GUI 항목 모두 제거.

- [ ] **Step 24.2: CHANGELOG — v0.3.0 항목 추가** — 상단에 prepend:

```markdown
## [0.3.0] — 2026-05-10

### Removed
- GUI 모듈 (`documatch.gui`) 전체 — Windows SmartScreen + macOS Gatekeeper 배포 마찰로 hard delete
- `documatch-gui` entry-point, PyInstaller 빌드 자산
- 100건 GUI 테스트

### Added
- Rich 기반 시각 피드백 — 매 HITL 게이트에 단계 헤더 + 단축키 legend
- 실시간 진행 표시 — LLM 호출 중 스피너 + 경과 시간, 배치 중 progress bar + 카운트
- 에러 prominent 박스 — 빨간 Panel + 원인 + 조치 안내 (LLMAuthError → "documatch init" 등)
- 신뢰도 색상 — 추출 결과 표에서 ≥0.8 🟢, ≥0.5 🟡, <0.5 🔴⚠
- `documatch preview <file>` 신규 명령 — LLM 없이 파일 미리보기
- `--quiet` / `--summary` / `--json` 자동화 플래그

### Changed
- InteractiveReviewer 전면 재작성 — `interaction/ui/` 모듈 4개로 분리 (header/progress/errors/tables)
- AutomationReviewer에 `quiet` 옵션 추가
```

- [ ] **Step 24.3: Commit**

```bash
git add README.md CHANGELOG.md
git commit -m "docs: README + CHANGELOG v0.3.0 — GUI 제거 + CLI Polish 강조"
```

---

### Task 25: 최종 QA + 0.3.0-dev → 0.3.0 (verification, non-TDD)

**Files:** 없음 (검증만)

- [ ] **Step 25.1: 전체 테스트 + 커버리지**

```bash
cd /Users/yjban/Desktop/documatch-cli
source .venv/bin/activate
pytest --cov=documatch --cov-report=term -q
```

기대:
- 약 200건 통과
- 커버리지 89%+ 유지

- [ ] **Step 25.2: Lint**

```bash
ruff check src/ tests/
```

기대: All checks passed!

- [ ] **Step 25.3: 의존성 점검 + wheel 빌드**

```bash
pip install -e ".[dev]"
python -m build
ls dist/
```

기대: `dist/documatch_cli-0.3.0-py3-none-any.whl` + `documatch_cli-0.3.0.tar.gz` 생성.

- [ ] **Step 25.4: install smoke**

```bash
python -m venv /tmp/dm_v3_smoke
/tmp/dm_v3_smoke/bin/pip install dist/documatch_cli-0.3.0-py3-none-any.whl
/tmp/dm_v3_smoke/bin/documatch --version
/tmp/dm_v3_smoke/bin/documatch --help
/tmp/dm_v3_smoke/bin/documatch preview --help
```

기대: 각 명령 exit 0, `documatch, version 0.3.0` 출력.

- [ ] **Step 25.5: 수동 CLI 스모크 (선택)**

```bash
documatch ./tests/fixtures/docs/  # 또는 evidence/
```

GUI 없이 CLI에서 7단계 흐름 + 새 시각 요소 (헤더, legend, 색상, 진행 bar) 확인.

- [ ] **Step 25.6: 버전 0.3.0-dev → 0.3.0 release commit**

`pyproject.toml`의 `version = "0.3.0-dev"` → `version = "0.3.0"`
`src/documatch/__init__.py`의 `__version__ = "0.3.0-dev"` → `__version__ = "0.3.0"`

```bash
git add pyproject.toml src/documatch/__init__.py
git commit -m "release: v0.3.0"
```

- [ ] **Step 25.7: 문제 발생 시 수정 후 추가 commit**

---

### Task 26: v0.3.0 tag + push (release ops, non-TDD)

**Files:** 없음 (git 작업만)

- [ ] **Step 26.1: tag 생성**

```bash
cd /Users/yjban/Desktop/documatch-cli
git tag -a v0.3.0 -m "v0.3.0 — CLI Polish + GUI Removal

GUI 모듈 hard delete (Windows SmartScreen + macOS Gatekeeper 배포 마찰로).
CLI Rich 기반 production-quality UX:
- 매 HITL 게이트에 단계 헤더 + 단축키 legend
- LLM 호출 실시간 스피너 + 진행률 bar
- 에러 prominent 박스 + 원인·조치 안내
- 추출 결과 신뢰도 색상 (🟢🟡🔴)
- documatch preview 신규 명령
- --quiet/--summary/--json 자동화 플래그

200+ tests, 89%+ coverage."
```

- [ ] **Step 26.2: main + tag push**

```bash
git push origin main
git push origin v0.3.0
```

GitHub Actions Release workflow가 자동 트리거 → wheel 빌드 + GitHub Release 생성 + dist/* 첨부.

- [ ] **Step 26.3: Release 페이지 확인**

```bash
gh run list --repo firefly0731/documatch-cli --limit 2
```

`Release` workflow가 `completed success` 되면 OK. 실패 시 로그 확인 후 수정.

```bash
gh release view v0.3.0 --repo firefly0731/documatch-cli
```

기대: `documatch_cli-0.3.0-py3-none-any.whl` 첨부 확인.

---

## 자가 점검 (계획 작성자)

**스펙 커버리지** (스펙 §1~§7 vs 본 계획 task 매핑):
- §1 아키텍처 → Task 4-11 (UI 모듈), Task 12-17 (InteractiveReviewer)
- §2 핵심 4가지 → Task 4-5 (header), 6-7 (progress), 8 (errors), 9-10 (tables) + 12-17 (적용)
- §3 부가 (preview/플래그) → Task 18-20 (preview), 21-23 (플래그)
- §4 GUI 제거 → Task 1-3
- §5 테스트 → 각 Task 내 TDD + Task 25 최종 QA
- §6 마이그레이션 단계 → Phase 1-6 그대로 매핑
- §7 위험 → Task 6-7에서 Rich Live + Windows ASCII 폴백 고려, Task 24 README 명시

**Placeholder scan**: TBD/TODO/FIXME 0건. 모든 step에 구체적 코드/명령.

**Type 일관성**:
- `render_step_header`, `render_action_legend`, `live_spinner`, `progress_bar`, `render_error_panel`, `render_spec_table`, `render_result_table` — 모든 task에서 동일 명칭 사용
- `SpinnerHandle`, `ProgressBarHandle` — Task 6-7에 정의, 그대로 유지
- `REMEDY_MAP` — Task 8에 정의, public export
- `quiet`, `summary`, `json_output` — Task 21-23에서 일관된 매개변수 이름

---

**Plan complete and saved to `docs/superpowers/plans/2026-05-10-documatch-cli-v0.3.md`.**

총 26 task / 6 phase / 약 1주 분량.

Phase 단위 산출물:
- Phase 1: GUI 제거 후 깨끗한 CLI repo (169 tests, 0.3.0-dev)
- Phase 2: 4개 UI helper 모듈 + 단위 테스트 (~12-15 신규)
- Phase 3: InteractiveReviewer 전면 새 UX (~12 신규 검증, progress_bar 통합)
- Phase 4: documatch preview 작동 (PDF/XLSX/이미지/텍스트 메타)
- Phase 5: --quiet/--summary/--json 작동 (--json은 Excel skip)
- Phase 6: README/CHANGELOG/0.3.0 release/태그·push

---

## Codex Review Incorporation (2026-05-10)

Codex 리뷰 결과 반영 내역:

**적용한 수정**:
- Task 1, 2, 3, 11, 17(부분), 18(통합), 19, 20, 24, 25, 26 — non-TDD cleanup/verification/release/docs로 명시
- Task 2: version 0.3.0 → 0.3.0-dev (Task 25에서 -dev 제거하고 release tag) — v0.2.0 패턴 일치
- Task 17: manual `━─` bar 제거, Task 7의 `progress_bar` helper 사용 (state-keeping으로 stateful 적용)
- Task 17.3: 삭제 대상 메서드 정확히 명시 (`_render_spec`, `_render_result`)
- Task 18+19 통합: Task 18에서 모듈 + register 함께 → Task 19/20은 verification으로 축소
- Task 18: preview에 XLSX 시트 list + row/col, 이미지 dimension, 텍스트 추출 가능 yes/no 메타데이터 추가
- Task 20: `--lines` 플래그 제거 (spec 외 추가 기능). 50줄 고정.
- Task 23: `--json` 시 Excel 저장 skip (스펙 §3.2 정합)
- Task 25: version 0.3.0-dev → 0.3.0 release commit 단계 추가
- Spec §0 결정 표: archive 태그 잔존 문구 제거 — §4.2와 일관

**의도적 미반영 (이유 명시)**:
- 실 LLM 호출 site (spec generation/refine, sample extract)에 `live_spinner` 통합 — engine 변경 필요라서 v0.3.x로 보류 (스펙은 helper 정의만 요구). Task 17에 명시.
- Windows ASCII 폴백 — Rich 6.7+ 기본 cp1252 호환 (✓→[OK] 이슈는 v0.2.0 release 단계에서 해결). v0.3.x 추가 polish로 보류.
- 7단계 통합 end-to-end interactive 테스트 추가 — 기존 `tests/integration/test_engine_*` 3종이 ScriptedReviewer로 동등한 통합 검증 제공. 추가 task 불필요.
- `REMEDY_MAP` 공개 vs 비공개 — 공개로 결정 (확장 가능성 + 다른 모듈에서 참조 가능). 스펙은 그대로.
- Release workflow validation pre-tag — workflow 단순화 후 첫 tag 푸시로 검증. 위험 낮음 (v0.2.0/v0.2.1 검증 완료된 구조 재사용).

---

## v0.3.1 Patch Notes (2026-05-10)

v0.3.0 release 후 사용자 실사용 피드백으로 즉시 추가된 6개 commit. evidence/ PDF 5건으로 실제 추출 검증 완료.

**Release**: https://github.com/firefly0731/documatch-cli/releases/tag/v0.3.1

### Added (사용자 요청)

1. **LLM 호출 site 실시간 spinner** — Codex 리뷰의 v0.3.0 미반영 항목 + 사용자 직접 요청 (`Claude Code의 주황색 별 같은 표시`):
   - Reviewer Protocol에 `show_thinking(message)` / `hide_thinking()` 추가
   - InteractiveReviewer가 `live_spinner` (transient=True 기본) 사용
   - AutomationReviewer는 quiet 시 무음, 아니면 stderr `[thinking] {msg}`
   - Engine 4 LLM call sites (generate, refine ×2, extract) `try/finally`로 wrap
   - commit `ba0d088`

2. **refine 누적 컨텍스트** — 사용자 요청 (`이전 요청 기억해서 짧은 후속 피드백만으로도 안전하게 누적되게`):
   - `_State`에 `refine_history: list[str]` 추가, 매 refine 사이클마다 누적
   - `SpecGenerator.refine()`에 `original_prompt` + `prior_feedbacks` 파라미터 추가
   - 시스템 프롬프트에 "사용자가 명시적으로 제거 요청한 필드만 제거" 원칙 명시
   - 효과: "제재일과 제재내용 추출" → 2개 필드, 이후 "과태료 추가"만 입력 → 3개 필드 (이전엔 모두 다시 적어야 안전)
   - commit `cf52205`

### Changed (UX polish)

3. **사용자 친화적 표현 통일 — "스펙" → "추출 항목"** — 사용자 피드백 ("스펙이라는 말이 와닿지 않음"):
   - Spinner: "AI가 스펙 생성 중" → "AI가 요청하신 항목 정리 중"
   - Spinner: "AI에게 스펙 수정 요청 중" → "AI가 요청 사항 반영 중"
   - Spinner: "샘플 추출 중" → "AI가 샘플 파일에서 값 추출 중"
   - Step 3 헤더, 표 제목, legend, prompt, CLI 도움말 일괄 변경
   - commit `f19d22f`

### Fixed (회귀)

4. **다항목 입력 시 1개만 인식되던 회귀** — 사용자 발견 (`제재일, 제재내용, 과태료 입력해도 1개만 인식`):
   - `_GENERATE_SYSTEM` 프롬프트에 "사용자가 요청한 모든 항목을 빠짐없이 fields 배열에 포함" 핵심 원칙 추가
   - 한국어 연결어("과/와", "그리고", "및", 쉼표, 줄바꿈) 분리 규칙 명시
   - 구체적 예시 추가 ("4개 항목 → 4개 필드")
   - `_REFINE_SYSTEM`도 동일 강화
   - commit `472ea0a`

### Verified

5. **evidence/ 5건 PDF 실사용 검증** — 금융감독원 제재내용 공개안:
   - 7개 필드(금융기관명/제재일/과태료/임원수/직원수/위반내용/관련법규) 추출
   - 5/5 성공, 평균 신뢰도 0.95 (모두 ≥0.85, 검토 필요 0건)
   - --auto-approve --spec-file --summary 모드로 즉시 batch 추출
   - Excel 출력 (한글 헤더 인코딩 정상)

### 운영 노트

- v0.3.1 tag가 처음 push된 commit(`8186b27`)이 reset된 옛 commit이라 잘못된 release workflow가 trigger됨 → 즉시 cancel + tag 삭제 후 HEAD(`cb40d29`)에 재태깅 + 재push
- Release workflow는 정정 commit 기준으로 정상 수행
- v0.3.0 spec의 "v0.3.x 보류" 항목 중 `live_spinner` 통합이 v0.3.1로 앞당겨짐. Windows ASCII 폴백, 7단계 통합 e2e 테스트는 여전히 보류.
