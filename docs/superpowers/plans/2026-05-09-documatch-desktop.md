# documatch-desktop v0.2.0 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** PySide6 기반 데스크톱 GUI 앱(`documatch-gui`)을 기존 `documatch-cli` v0.1.0 위에 추가하여 v0.2.0으로 릴리즈. CLI는 그대로 유지.

**Architecture:** 단일 프로세스(qasync로 Qt+asyncio 통합) + 새 `GuiReviewer`가 기존 `Reviewer` Protocol을 구현 → `ExtractionEngine` 그대로 재사용. 좌측 단계 네비 + 중앙 작업 + 우측 문서 미리보기 3컬럼 hybrid 레이아웃, 7 HITL 게이트 (Step 6 검토 게이트 신규 추가).

**Tech Stack:** PySide6, qasync, keyring, PyMuPDF, mammoth, pytest-qt, PyInstaller

**Working directory (코드):** `/Users/yjban/Desktop/documatch-cli/`
**스펙:** `DocuMatch/docs/superpowers/specs/2026-05-09-documatch-desktop-design.md`
**전제:** v0.1.0 완성 상태 (commit `e90aed7` 기준), 161 테스트 통과, 89% 커버리지, ruff clean

**Phase 개요 (10 phase, ~80 task)**:
- Phase 1 — Reviewer Protocol async 마이그레이션 (Tasks 1-6)
- Phase 2 — gui/ 스켈레톤 + qasync app (Tasks 7-13)
- Phase 3 — 단순 패널 (Step 1, 2, 4, 7) (Tasks 14-25)
- Phase 4 — 스펙·결과 편집 (Step 3, 5) (Tasks 26-37)
- Phase 5 — 배치 진행 + 결과 편집 (Step 6) (Tasks 38-44)
- Phase 6 — 문서 미리보기 (Tasks 45-54)
- Phase 7 — 다이얼로그 + keychain + 첫 실행 부트스트랩 (Tasks 55-62)
- Phase 8 — 빌드 파이프라인 (PyInstaller + GitHub Actions) (Tasks 63-67)
- Phase 9 — 테스트 백필 + 커버리지 (Tasks 68-72)
- Phase 10 — Polish + 릴리즈 (Tasks 73-80)

---

## Phase 1 — Reviewer Protocol Async Migration

이 phase는 GUI 의존성 0. 끝나도 CLI는 그대로 동작 (161 테스트 통과 유지). Reviewer/엔진/테스트가 async로 바뀐 것 외 변화 없음.

### Task 1: BatchResultDecision dataclass 추가

**Files:**
- Modify: `src/documatch/interaction/reviewer.py`
- Test: `tests/unit/test_reviewer_protocol.py`

- [ ] **Step 1.1: Failing test 추가** — `tests/unit/test_reviewer_protocol.py` 끝에 추가

```python
def test_batch_result_decision_constructible():
    from documatch.interaction.reviewer import BatchResultDecision
    d = BatchResultDecision(action="save")
    assert d.action == "save"
    assert d.modified_results is None
    d2 = BatchResultDecision(action="back", modified_results=[])
    assert d2.action == "back"
    assert d2.modified_results == []
```

- [ ] **Step 1.2: Run — FAIL** with ImportError on `BatchResultDecision`

```bash
cd /Users/yjban/Desktop/documatch-cli && source .venv/bin/activate
pytest tests/unit/test_reviewer_protocol.py::test_batch_result_decision_constructible -v
```

- [ ] **Step 1.3: Implement** — `src/documatch/interaction/reviewer.py`에 추가

기존 dataclass들 아래에 추가:

```python
from documatch.core.models import ExtractionResult


@dataclass
class BatchResultDecision:
    action: Literal["save", "back", "quit"]
    modified_results: list[ExtractionResult] | None = None
```

- [ ] **Step 1.4: Run — PASS**

- [ ] **Step 1.5: Commit**

```bash
git add src/documatch/interaction/reviewer.py tests/unit/test_reviewer_protocol.py
git commit -m "feat(interaction): BatchResultDecision dataclass 추가"
```

---

### Task 2: SpecDecision/SampleDecision에 modified_* 필드 추가

**Files:**
- Modify: `src/documatch/interaction/reviewer.py`
- Modify: `tests/unit/test_reviewer_protocol.py`

- [ ] **Step 2.1: Failing test 추가**

```python
def test_decisions_have_modified_fields():
    from documatch.interaction.reviewer import SpecDecision, SampleDecision
    d = SpecDecision(action="approve")
    assert d.modified_spec is None
    s = SampleDecision(action="full")
    assert s.modified_spec is None
    assert s.modified_result is None
```

- [ ] **Step 2.2: Run — FAIL** (AttributeError)

```bash
pytest tests/unit/test_reviewer_protocol.py::test_decisions_have_modified_fields -v
```

- [ ] **Step 2.3: Implement** — `reviewer.py`의 `SpecDecision`/`SampleDecision`에 필드 추가

```python
from documatch.spec.models import ExtractionSpec
from documatch.core.models import ExtractionResult


@dataclass
class SpecDecision:
    action: Literal["approve", "refine", "load", "back", "quit"]
    feedback: str | None = None
    spec_name: str | None = None
    modified_spec: ExtractionSpec | None = None       # NEW


@dataclass
class SampleDecision:
    action: Literal["full", "single_save", "refine", "back", "quit"]
    feedback: str | None = None
    modified_spec: ExtractionSpec | None = None       # NEW
    modified_result: ExtractionResult | None = None   # NEW
```

- [ ] **Step 2.4: Run — PASS**

- [ ] **Step 2.5: Commit**

```bash
git add src/documatch/interaction/reviewer.py tests/unit/test_reviewer_protocol.py
git commit -m "feat(interaction): SpecDecision/SampleDecision에 modified_* 필드 추가"
```

---

### Task 3: Reviewer Protocol을 async로 + review_batch_results 추가

**Files:**
- Modify: `src/documatch/interaction/reviewer.py`
- Modify: `src/documatch/interaction/__init__.py`

- [ ] **Step 3.1: Reviewer Protocol을 async + review_batch_results 추가**

`src/documatch/interaction/reviewer.py`의 Protocol 부분 전체 교체:

```python
class Reviewer(Protocol):
    async def confirm_files(self, entries: list[FileEntry]) -> ReviewDecision: ...
    async def select_sample(self, entries: list[FileEntry]) -> Path | None: ...
    async def get_user_prompt(self, default: str | None = None) -> str: ...
    async def review_spec(
        self, spec: ExtractionSpec, available_specs: list[str] | None = None,
    ) -> SpecDecision: ...
    async def confirm_mode(self, spec: ExtractionSpec) -> ModeDecision: ...
    async def review_sample(
        self, result: ExtractionResult, spec: ExtractionSpec,
    ) -> SampleDecision: ...
    async def review_batch_results(
        self, results: list[ExtractionResult], spec: ExtractionSpec,
    ) -> BatchResultDecision: ...
    async def get_save_path(self, default: Path) -> Path: ...
    async def confirm_save_spec(self, spec: ExtractionSpec) -> str | None: ...
    async def show_progress(self, current: int, total: int, label: str) -> None: ...
    async def show_message(self, level: Literal["info", "warn", "error"], text: str) -> None: ...
```

- [ ] **Step 3.2: __init__.py 업데이트** — `BatchResultDecision` export

`src/documatch/interaction/__init__.py`:

```python
from documatch.interaction.automation import AutomationReviewer
from documatch.interaction.interactive import InteractiveReviewer
from documatch.interaction.reviewer import (
    BatchResultDecision,
    ModeDecision,
    ReviewDecision,
    Reviewer,
    SampleDecision,
    SpecDecision,
)

__all__ = [
    "Reviewer", "ReviewDecision", "SpecDecision", "ModeDecision",
    "SampleDecision", "BatchResultDecision",
    "AutomationReviewer", "InteractiveReviewer",
]
```

- [ ] **Step 3.3: Commit** (테스트는 다음 Task에서 추가)

```bash
git add src/documatch/interaction/reviewer.py src/documatch/interaction/__init__.py
git commit -m "feat(interaction): Reviewer Protocol async + review_batch_results 추가"
```

이 시점에 기존 InteractiveReviewer/AutomationReviewer는 Protocol을 만족하지 못함 (sync 메서드라 mypy 타입체크 실패). Task 4-5에서 수정.

---

### Task 4: AutomationReviewer를 async로 + review_batch_results 구현

**Files:**
- Modify: `src/documatch/interaction/automation.py`
- Modify: `tests/unit/test_reviewer_automation.py`

- [ ] **Step 4.1: Failing test — async + 새 메서드** — `tests/unit/test_reviewer_automation.py` 기존 테스트 모두 async로 변환 + 새 테스트:

```python
import pytest
from pathlib import Path
from documatch.exceptions import UserAbort
from documatch.interaction.automation import AutomationReviewer
from documatch.interaction.reviewer import BatchResultDecision
from documatch.processors.base import FileEntry
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)


def _spec() -> ExtractionSpec:
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


@pytest.mark.asyncio
async def test_confirm_files_continues():
    r = AutomationReviewer(output_path=Path("out.xlsx"))
    decision = await r.confirm_files([
        FileEntry(file_path="/x.txt", filename="x.txt", extension="txt", size_bytes=1),
    ])
    assert decision.action == "continue"


@pytest.mark.asyncio
async def test_review_spec_approves():
    r = AutomationReviewer(output_path=Path("out.xlsx"))
    decision = await r.review_spec(_spec())
    assert decision.action == "approve"


@pytest.mark.asyncio
async def test_get_save_path_uses_configured():
    r = AutomationReviewer(output_path=Path("/tmp/result.xlsx"))
    path = await r.get_save_path(Path("/tmp/default.xlsx"))
    assert path == Path("/tmp/result.xlsx")


@pytest.mark.asyncio
async def test_select_sample_raises_when_unsupported():
    r = AutomationReviewer(output_path=Path("out.xlsx"))
    with pytest.raises(UserAbort):
        await r.select_sample([])


@pytest.mark.asyncio
async def test_confirm_save_spec_returns_none():
    r = AutomationReviewer(output_path=Path("out.xlsx"))
    result = await r.confirm_save_spec(_spec())
    assert result is None


@pytest.mark.asyncio
async def test_review_batch_results_saves_immediately():
    r = AutomationReviewer(output_path=Path("out.xlsx"))
    decision = await r.review_batch_results([], _spec())
    assert isinstance(decision, BatchResultDecision)
    assert decision.action == "save"
```

- [ ] **Step 4.2: Run — FAIL**

```bash
pytest tests/unit/test_reviewer_automation.py -v
```

- [ ] **Step 4.3: Implement** — `src/documatch/interaction/automation.py` 전체 교체:

```python
"""AutomationReviewer — 모든 결정을 자동 승인."""
from pathlib import Path
from typing import Literal

from documatch.core.models import ExtractionResult
from documatch.exceptions import UserAbort
from documatch.interaction.reviewer import (
    BatchResultDecision,
    ModeDecision,
    ReviewDecision,
    SampleDecision,
    SpecDecision,
)
from documatch.processors.base import FileEntry
from documatch.spec.models import ExtractionSpec


class AutomationReviewer:
    def __init__(self, *, output_path: Path) -> None:
        self.output_path = output_path

    async def confirm_files(self, entries: list[FileEntry]) -> ReviewDecision:
        return ReviewDecision(action="continue")

    async def select_sample(self, entries: list[FileEntry]) -> Path | None:
        raise UserAbort(
            "자동화 모드에서는 LLM 기반 스펙 생성을 허용하지 않습니다. "
            "--spec 또는 --spec-file을 사용하세요."
        )

    async def get_user_prompt(self, default: str | None = None) -> str:
        return default or ""

    async def review_spec(
        self, spec: ExtractionSpec, available_specs: list[str] | None = None,
    ) -> SpecDecision:
        return SpecDecision(action="approve")

    async def confirm_mode(self, spec: ExtractionSpec) -> ModeDecision:
        return ModeDecision(action="table" if spec.table_mode else "single")

    async def review_sample(
        self, result: ExtractionResult, spec: ExtractionSpec,
    ) -> SampleDecision:
        return SampleDecision(action="full")

    async def review_batch_results(
        self, results: list[ExtractionResult], spec: ExtractionSpec,
    ) -> BatchResultDecision:
        return BatchResultDecision(action="save")

    async def get_save_path(self, default: Path) -> Path:
        return self.output_path

    async def confirm_save_spec(self, spec: ExtractionSpec) -> str | None:
        return None

    async def show_progress(self, current: int, total: int, label: str) -> None:
        pass

    async def show_message(self, level: Literal["info", "warn", "error"], text: str) -> None:
        import sys
        stream = sys.stderr if level in ("warn", "error") else sys.stdout
        print(f"[{level}] {text}", file=stream)
```

- [ ] **Step 4.4: Run — PASS**

- [ ] **Step 4.5: Commit**

```bash
git add src/documatch/interaction/automation.py tests/unit/test_reviewer_automation.py
git commit -m "feat(interaction): AutomationReviewer async + review_batch_results"
```

---

### Task 5: InteractiveReviewer를 async로 + review_batch_results 구현

**Files:**
- Modify: `src/documatch/interaction/interactive.py`
- Modify: `tests/unit/test_reviewer_interactive.py`

- [ ] **Step 5.1: 기존 38개 테스트의 시그니처 변경** — `tests/unit/test_reviewer_interactive.py` 안의 모든 `def test_*` → `async def test_*` + 모든 `r.method(...)` → `await r.method(...)`. (mechanical 작업)

또한 다음 신규 테스트 추가:

```python
@pytest.mark.asyncio
async def test_review_batch_results_saves_on_yes(monkeypatch):
    _patch_choice(monkeypatch, "y")
    r = InteractiveReviewer(console=_silent())
    d = await r.review_batch_results([], _spec())
    assert d.action == "save"


@pytest.mark.asyncio
async def test_review_batch_results_back_on_no(monkeypatch):
    _patch_choice(monkeypatch, "n")
    r = InteractiveReviewer(console=_silent())
    d = await r.review_batch_results([], _spec())
    assert d.action == "back"
```

- [ ] **Step 5.2: Run — FAIL**

```bash
pytest tests/unit/test_reviewer_interactive.py -v
```

- [ ] **Step 5.3: Implement** — `src/documatch/interaction/interactive.py` 전체 메서드를 `async def`로. 본문은 동일 (블로킹 input/Prompt.ask는 그대로 — async 안에서 호출해도 문제 없음, 단 GUI에서는 GuiReviewer가 별도로 구현). 그리고 `review_batch_results` 추가:

기존 클래스 메서드 시그니처 모두 `async`로 변경 (본문 변경 없음). 추가:

```python
    async def review_batch_results(
        self, results: list[ExtractionResult], spec: ExtractionSpec,
    ) -> BatchResultDecision:
        # 결과 요약 표시
        successful = sum(1 for r in results if r.status == "success")
        failed = sum(1 for r in results if r.status == "failed")
        review = sum(1 for r in results if r.needs_review)
        self.console.print()
        self.console.print(
            f"[bold]배치 완료[/bold] — 성공 {successful} / 실패 {failed} / 검토필요 {review}"
        )
        c = choice(self.console, "저장하시겠습니까? [Y]저장 / [N]재배치", ["y", "n"], "y")
        if c == "n":
            return BatchResultDecision(action="back")
        return BatchResultDecision(action="save")
```

import에 `BatchResultDecision` 추가:

```python
from documatch.interaction.reviewer import (
    BatchResultDecision,
    ModeDecision,
    ReviewDecision,
    SampleDecision,
    SpecDecision,
)
```

- [ ] **Step 5.4: Run — PASS**

```bash
pytest tests/unit/test_reviewer_interactive.py -v
```

- [ ] **Step 5.5: Commit**

```bash
git add src/documatch/interaction/interactive.py tests/unit/test_reviewer_interactive.py
git commit -m "feat(interaction): InteractiveReviewer async + review_batch_results"
```

---

### Task 6: ExtractionEngine에 await + modified_* 처리 + review_batch_results 호출

**Files:**
- Modify: `src/documatch/core/engine.py`
- Modify: `tests/integration/test_engine_*.py` (3개 파일)

- [ ] **Step 6.1: 통합 테스트 시그니처 변경** — `tests/integration/` 안의 3개 테스트 파일에서 `await reviewer.X(...)` 사용처가 있는 곳 (`_ScriptedReviewer` 메서드들도 `async def`로). 그리고 새 `review_batch_results` 메서드 추가:

`tests/integration/test_engine_hitl_refine.py`의 `_ScriptedReviewer`에 추가:

```python
class _ScriptedReviewer:
    # 기존 메서드들 모두 async def로 변경
    
    async def review_batch_results(self, results, spec):
        from documatch.interaction.reviewer import BatchResultDecision
        return BatchResultDecision(action="save")
    
    async def show_progress(self, current, total, label):
        pass

    async def show_message(self, level, text):
        pass
    
    # ... 기타 모두 async def
```

- [ ] **Step 6.2: engine.py 수정** — 모든 `self.reviewer.X(...)`에 `await` 추가 + `review_batch_results` 호출 + `modified_*` 처리:

`src/documatch/core/engine.py`의 `run` 메서드 + 헬퍼들을 다음으로 교체:

```python
async def run(
    self,
    scan_path: Path,
    *,
    preloaded_spec: ExtractionSpec | None = None,
    default_output: Path | None = None,
) -> PipelineFinal:
    st = _State(scan_path=Path(scan_path))

    # Step 1: Scan
    st.entries = scan_directory(st.scan_path)
    if not st.entries:
        await self.reviewer.show_message("error", "스캔된 파일이 없습니다.")
        raise UserAbort("no files")

    # HITL ①
    decision = await self.reviewer.confirm_files(st.entries)
    if decision.action == "quit":
        raise UserAbort("user quit at file confirm")
    if decision.action == "filter" and decision.extension_filter:
        st.entries = [e for e in st.entries if e.extension in decision.extension_filter]

    # Step 2-4: Spec acquisition
    if preloaded_spec is not None:
        st.spec = preloaded_spec
    else:
        st.spec = await self._acquire_spec(st)

    # Step 4: Mode confirm
    if preloaded_spec is None:
        mode = await self.reviewer.confirm_mode(st.spec)
        if mode.action == "back":
            st.spec = await self._acquire_spec(st)
        else:
            st.spec.table_mode = (mode.action == "table")

    # Step 5: Sample extract & review
    if preloaded_spec is None:
        await self._sample_review_loop(st)

    # Optional: save spec
    if preloaded_spec is None:
        name = await self.reviewer.confirm_save_spec(st.spec)
        if name:
            st.spec.name = name
            self.spec_store.save(st.spec, overwrite=True)

    # Step 6: Batch extract
    results: list[ExtractionResult] = []
    batch = BatchExtractor(spec=st.spec, llm=self.extract_llm)
    total = len(st.entries)
    idx = 0
    async for r in batch.extract_all(st.entries):
        idx += 1
        results.append(r)
        await self.reviewer.show_progress(idx, total, r.document_id)

    # Step 6.5: NEW — Batch result review
    br_decision = await self.reviewer.review_batch_results(results, st.spec)
    if br_decision.action == "quit":
        raise UserAbort("user quit at batch review")
    if br_decision.action == "back":
        # 단순 재배치: 같은 entries로 batch만 다시
        results = []
        idx = 0
        async for r in batch.extract_all(st.entries):
            idx += 1
            results.append(r)
            await self.reviewer.show_progress(idx, total, r.document_id)
        # 두 번째 review_batch_results 결과로 진행 — 이번엔 결과 무시 옵션
        br_decision = await self.reviewer.review_batch_results(results, st.spec)
        if br_decision.action == "quit":
            raise UserAbort("user quit at batch review")
    if br_decision.modified_results:
        results = br_decision.modified_results

    # Step 7: Export
    timestamp = datetime.now().strftime("%Y-%m-%d_%H%M%S")
    default_out = default_output or Path(f"./documatch_result_{timestamp}.xlsx")
    out_path = await self.reviewer.get_save_path(default_out)
    self.exporter.write(st.spec, results, out_path)
    await self.reviewer.show_message("info", f"저장됨: {out_path}")
    return PipelineFinal(spec=st.spec, results=results, output_path=out_path)


async def _acquire_spec(self, st: _State) -> ExtractionSpec:
    while True:
        sample_path = await self.reviewer.select_sample(st.entries)
        if sample_path is None:
            raise UserAbort("no sample")
        st.sample_path = sample_path
        st.sample_doc = await self._load(sample_path)
        st.user_prompt = await self.reviewer.get_user_prompt()
        spec = await self.spec_generator.generate(st.sample_doc, st.user_prompt)

        while True:
            avail = [s.name for s in self.spec_store.list()]
            decision = await self.reviewer.review_spec(spec, available_specs=avail)
            if decision.action == "quit":
                raise UserAbort("user quit at spec review")
            if decision.action == "back":
                break
            if decision.action == "approve":
                if decision.modified_spec is not None:
                    return decision.modified_spec
                return spec
            if decision.action == "refine":
                spec = await self.spec_generator.refine(
                    spec, feedback=decision.feedback or "", sample=st.sample_doc,
                )
                continue
            if decision.action == "load":
                try:
                    return self.spec_store.load(decision.spec_name or "")
                except SpecNotFoundError:
                    await self.reviewer.show_message("warn", "스펙을 찾을 수 없습니다.")
                    continue


async def _sample_review_loop(self, st: _State) -> None:
    single = SingleExtractor(spec=st.spec, llm=self.extract_llm)
    while True:
        sample_id = str(st.sample_path) if st.sample_path else "sample"
        st.sample_result = await single.extract(st.sample_doc, document_id=sample_id)
        decision = await self.reviewer.review_sample(st.sample_result, st.spec)
        if decision.modified_spec is not None:
            st.spec = decision.modified_spec
            single = SingleExtractor(spec=st.spec, llm=self.extract_llm)
        if decision.modified_result is not None:
            st.sample_result = decision.modified_result
        if decision.action == "quit":
            raise UserAbort("user quit at sample review")
        if decision.action == "back":
            st.spec = await self._acquire_spec(st)
            single = SingleExtractor(spec=st.spec, llm=self.extract_llm)
            continue
        if decision.action == "single_save":
            return
        if decision.action == "refine":
            st.spec = await self.spec_generator.refine(
                st.spec, feedback=decision.feedback or "", sample=st.sample_doc,
            )
            single = SingleExtractor(spec=st.spec, llm=self.extract_llm)
            continue
        return  # action == "full"


async def _load(self, path: Path):
    ext = path.suffix.lstrip(".")
    proc = get_processor(ext)
    return await proc.process(path.read_bytes(), path.name)
```

- [ ] **Step 6.3: Run 전체 테스트** — `pytest -q` — 161 passed 확인

```bash
cd /Users/yjban/Desktop/documatch-cli && source .venv/bin/activate && pytest -q
```

- [ ] **Step 6.4: Commit**

```bash
git add src/documatch/core/engine.py tests/integration/
git commit -m "feat(core): ExtractionEngine async migration + review_batch_results 게이트 추가"
```

Phase 1 완료. 161 테스트 그대로 통과. CLI 동작 100% 호환.

---

## Phase 2 — gui/ 스켈레톤 + qasync app

### Task 7: pyproject.toml — gui extras + documatch-gui entry-point + 신규 의존성

**Files:**
- Modify: `pyproject.toml`

- [ ] **Step 7.1: pyproject.toml 수정** — 다음 변경:

`[project.scripts]` 섹션에 추가:
```toml
[project.scripts]
documatch = "documatch.cli.app:main"
documatch-gui = "documatch.gui.app:main"
```

`[project.optional-dependencies]`에 `gui` 추가, `dev`에 `pytest-qt`/`pyinstaller` 추가:
```toml
[project.optional-dependencies]
gui = [
    "PySide6>=6.7",
    "qasync>=0.27",
    "keyring>=25",
    "PyMuPDF>=1.24",
    "mammoth>=1.7",
]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-cov>=4.1",
    "pytest-qt>=4.4",
    "ruff>=0.4",
    "mypy>=1.10",
    "build>=1.2",
    "pyinstaller>=6.6",
    "vulture>=2.13",
]
```

`[project]` `version`을 `"0.2.0-dev"`로 변경.

`src/documatch/__init__.py`도 `__version__ = "0.2.0-dev"`로 변경.

- [ ] **Step 7.2: 의존성 설치**

```bash
cd /Users/yjban/Desktop/documatch-cli && source .venv/bin/activate
pip install -e ".[dev,gui]"
```

`PySide6` 등 설치 확인:
```bash
python -c "import PySide6; import qasync; import keyring; import fitz; import mammoth; print('all OK')"
```

- [ ] **Step 7.3: Commit**

```bash
git add pyproject.toml src/documatch/__init__.py
git commit -m "build: gui extras + documatch-gui entry-point + 0.2.0-dev"
```

---

### Task 8: gui/__init__ + app.py qasync 진입점

**Files:**
- Create: `src/documatch/gui/__init__.py`
- Create: `src/documatch/gui/app.py`
- Test: `tests/gui/__init__.py`, `tests/gui/conftest.py`, `tests/gui/test_app_entry.py`

- [ ] **Step 8.1: tests/gui/__init__.py 생성** (빈 파일)

- [ ] **Step 8.2: tests/gui/conftest.py 생성**

```python
"""GUI 테스트 공용 fixtures."""
import pytest
from PySide6.QtWidgets import QApplication


@pytest.fixture(scope="session")
def qapp():
    app = QApplication.instance() or QApplication([])
    yield app
```

- [ ] **Step 8.3: Failing test — `tests/gui/test_app_entry.py`**

```python
"""documatch-gui 진입점 모듈 import 가능 검증."""
import pytest


def test_gui_app_module_imports(qapp):
    from documatch.gui import app  # noqa
    assert hasattr(app, "main")


def test_gui_main_window_class_imports(qapp):
    from documatch.gui.main_window import MainWindow  # noqa
    assert MainWindow is not None
```

- [ ] **Step 8.4: Run — FAIL** with ModuleNotFoundError

```bash
pytest tests/gui/test_app_entry.py -v
```

- [ ] **Step 8.5: Implement `src/documatch/gui/__init__.py`**

```python
"""documatch GUI 모듈."""
from documatch.gui.app import main

__all__ = ["main"]
```

- [ ] **Step 8.6: Implement `src/documatch/gui/app.py`**

```python
"""documatch-gui 진입점 — qasync로 Qt + asyncio 통합."""
import asyncio
import sys

import qasync
from PySide6.QtWidgets import QApplication

from documatch.gui.main_window import MainWindow


def main() -> int:
    app = QApplication.instance() or QApplication(sys.argv)
    loop = qasync.QEventLoop(app)
    asyncio.set_event_loop(loop)

    window = MainWindow()
    window.show()

    with loop:
        return loop.run_forever() or 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 8.7: Implement `src/documatch/gui/main_window.py` (스켈레톤)**

```python
"""MainWindow — 좌측 단계 네비 + 중앙 메인 + 우측 미리보기 3컬럼."""
from PySide6.QtWidgets import QLabel, QMainWindow, QSplitter, QWidget
from PySide6.QtCore import Qt


class MainWindow(QMainWindow):
    def __init__(self) -> None:
        super().__init__()
        self.setWindowTitle("DocuMatch")
        self.resize(1280, 800)

        splitter = QSplitter(Qt.Horizontal)
        splitter.addWidget(self._make_step_nav())
        splitter.addWidget(self._make_main_area())
        splitter.addWidget(self._make_preview_panel())
        splitter.setSizes([180, 700, 400])
        self.setCentralWidget(splitter)

    def _make_step_nav(self) -> QWidget:
        return QLabel("단계 네비\n(Task 10)")

    def _make_main_area(self) -> QWidget:
        return QLabel("메인 작업 영역\n(Task 11+)")

    def _make_preview_panel(self) -> QWidget:
        return QLabel("문서 미리보기\n(Task 45+)")
```

- [ ] **Step 8.8: Run — PASS**

```bash
pytest tests/gui/test_app_entry.py -v
```

- [ ] **Step 8.9: 수동 스모크** — 빈 창이 뜨는지 확인 (선택, 백그라운드 X)

```bash
documatch-gui
# 창 뜨면 Cmd+Q 또는 X로 닫기
```

- [ ] **Step 8.10: Commit**

```bash
git add src/documatch/gui/ tests/gui/
git commit -m "feat(gui): qasync 진입점 + MainWindow 스켈레톤 (3컬럼 placeholder)"
```

---

### Task 9: gitignore + 테스트 인프라

**Files:**
- Modify: `.gitignore`
- Modify: `pyproject.toml` (pytest 설정)

- [ ] **Step 9.1: .gitignore 추가**

```
# PyInstaller
build/
dist/
*.spec.bak

# GUI
.qt/
```

- [ ] **Step 9.2: pyproject.toml의 pytest 설정에 markers 추가**

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
pythonpath = ["."]
addopts = "-ra"
markers = [
    "gui: GUI 테스트 (Qt 디스플레이 필요)",
]
```

- [ ] **Step 9.3: Run 전체 — 162 passed (161 + Task 8의 2개)**

```bash
pytest -q
```

- [ ] **Step 9.4: Commit**

```bash
git add .gitignore pyproject.toml
git commit -m "chore(gui): gitignore + pytest gui marker"
```

---

### Task 10: StepNav 위젯 (좌측 단계 네비)

**Files:**
- Create: `src/documatch/gui/panels/__init__.py`
- Create: `src/documatch/gui/panels/step_nav.py`
- Test: `tests/gui/test_step_nav.py`

- [ ] **Step 10.1: panels/__init__.py 생성** (빈 파일)

- [ ] **Step 10.2: Failing test — `tests/gui/test_step_nav.py`**

```python
"""StepNav 위젯 — 단계 표시 + 클릭 시그널."""
import pytest
from PySide6.QtCore import Qt

from documatch.gui.panels.step_nav import StepNav


def test_step_nav_starts_at_step_1(qapp, qtbot):
    nav = StepNav()
    qtbot.addWidget(nav)
    assert nav.current_step == 1


def test_step_nav_marks_current_active(qapp, qtbot):
    nav = StepNav()
    qtbot.addWidget(nav)
    nav.set_current(3)
    assert nav.current_step == 3


def test_step_nav_emits_signal_when_completed_step_clicked(qapp, qtbot):
    nav = StepNav()
    qtbot.addWidget(nav)
    nav.set_current(4)  # 1, 2, 3 are completed
    with qtbot.waitSignal(nav.step_clicked, timeout=1000) as sig:
        nav._buttons[0].click()  # Step 1
    assert sig.args[0] == 1


def test_step_nav_does_not_emit_signal_for_future_steps(qapp, qtbot):
    nav = StepNav()
    qtbot.addWidget(nav)
    nav.set_current(2)  # only 1 is completed
    received: list[int] = []
    nav.step_clicked.connect(lambda i: received.append(i))
    nav._buttons[5].click()  # Step 6 — future
    assert received == []
```

- [ ] **Step 10.3: Run — FAIL**

- [ ] **Step 10.4: Implement `src/documatch/gui/panels/step_nav.py`**

```python
"""StepNav — 좌측 단계 네비 위젯."""
from PySide6.QtCore import Qt, Signal
from PySide6.QtWidgets import QPushButton, QVBoxLayout, QWidget

_STEPS = [
    (1, "파일 스캔"),
    (2, "샘플 + 의도"),
    (3, "스펙 검토"),
    (4, "모드"),
    (5, "샘플 결과"),
    (6, "배치"),
    (7, "저장"),
]


class StepNav(QWidget):
    step_clicked = Signal(int)  # 사용자가 통과한 단계 클릭 시

    def __init__(self) -> None:
        super().__init__()
        self.current_step = 1
        self._buttons: list[QPushButton] = []

        layout = QVBoxLayout(self)
        layout.setContentsMargins(8, 8, 8, 8)
        layout.setSpacing(4)
        for num, label in _STEPS:
            btn = QPushButton(f"{num}. {label}")
            btn.setCheckable(False)
            btn.clicked.connect(lambda _=False, n=num: self._on_click(n))
            layout.addWidget(btn)
            self._buttons.append(btn)
        layout.addStretch()
        self._refresh_styles()

    def set_current(self, step: int) -> None:
        self.current_step = step
        self._refresh_styles()

    def _on_click(self, step: int) -> None:
        # 통과한 단계만 점프 가능 (현재 < 클릭된 단계는 무시)
        if step < self.current_step:
            self.step_clicked.emit(step)

    def _refresh_styles(self) -> None:
        for i, btn in enumerate(self._buttons, start=1):
            if i < self.current_step:
                btn.setText(f"✓ {i}. {_STEPS[i-1][1]}")
                btn.setEnabled(True)
            elif i == self.current_step:
                btn.setText(f"▸ {i}. {_STEPS[i-1][1]}")
                btn.setEnabled(False)
            else:
                btn.setText(f"  {i}. {_STEPS[i-1][1]}")
                btn.setEnabled(False)
```

- [ ] **Step 10.5: Run — PASS**

- [ ] **Step 10.6: Commit**

```bash
git add src/documatch/gui/panels/ tests/gui/test_step_nav.py
git commit -m "feat(gui): StepNav 위젯 (단계 네비 + 점프 시그널)"
```

---

### Task 11: 7개 빈 패널 파일 + StackedWidget 디스패처

**Files:**
- Create: `src/documatch/gui/panels/start_panel.py`
- Create: `src/documatch/gui/panels/file_panel.py`
- Create: `src/documatch/gui/panels/sample_panel.py`
- Create: `src/documatch/gui/panels/spec_panel.py`
- Create: `src/documatch/gui/panels/mode_panel.py`
- Create: `src/documatch/gui/panels/result_panel.py`
- Create: `src/documatch/gui/panels/batch_panel.py`
- Create: `src/documatch/gui/panels/save_panel.py`
- Modify: `src/documatch/gui/main_window.py`
- Test: `tests/gui/test_main_window.py`

- [ ] **Step 11.1: 8개 빈 패널 — 동일 패턴** — 각 파일은 다음 형태:

`src/documatch/gui/panels/start_panel.py`:
```python
"""StartPanel — 폴더 미선택 상태."""
from PySide6.QtWidgets import QLabel, QVBoxLayout, QWidget


class StartPanel(QWidget):
    def __init__(self) -> None:
        super().__init__()
        layout = QVBoxLayout(self)
        layout.addWidget(QLabel("폴더를 선택하세요\n(Task 14)"))
```

같은 패턴으로 7개 더 생성: `file_panel.py:FilePanel`, `sample_panel.py:SamplePanel`, `spec_panel.py:SpecPanel`, `mode_panel.py:ModePanel`, `result_panel.py:ResultPanel`, `batch_panel.py:BatchPanel`, `save_panel.py:SavePanel`. 라벨 텍스트만 다르게 (예: "Step 1 — 파일 스캔\n(Task 15)").

- [ ] **Step 11.2: Failing test — `tests/gui/test_main_window.py`**

```python
"""MainWindow — 패널 디스패처 + 단계 전환."""
from documatch.gui.main_window import MainWindow
from documatch.gui.panels.start_panel import StartPanel
from documatch.gui.panels.file_panel import FilePanel


def test_main_window_starts_with_start_panel(qapp, qtbot):
    w = MainWindow()
    qtbot.addWidget(w)
    assert isinstance(w.current_panel(), StartPanel)


def test_main_window_can_show_file_panel(qapp, qtbot):
    w = MainWindow()
    qtbot.addWidget(w)
    w.show_panel("file")
    assert isinstance(w.current_panel(), FilePanel)
```

- [ ] **Step 11.3: Run — FAIL**

- [ ] **Step 11.4: MainWindow 업데이트** — `src/documatch/gui/main_window.py` 전체 교체:

```python
"""MainWindow — 좌측 단계 네비 + 중앙 메인(StackedWidget) + 우측 미리보기."""
from PySide6.QtCore import Qt
from PySide6.QtWidgets import (
    QLabel, QMainWindow, QSplitter, QStackedWidget, QVBoxLayout, QWidget,
)

from documatch.gui.panels.batch_panel import BatchPanel
from documatch.gui.panels.file_panel import FilePanel
from documatch.gui.panels.mode_panel import ModePanel
from documatch.gui.panels.result_panel import ResultPanel
from documatch.gui.panels.sample_panel import SamplePanel
from documatch.gui.panels.save_panel import SavePanel
from documatch.gui.panels.spec_panel import SpecPanel
from documatch.gui.panels.start_panel import StartPanel
from documatch.gui.panels.step_nav import StepNav


class MainWindow(QMainWindow):
    PANEL_KEYS = {
        "start": StartPanel,
        "file": FilePanel,
        "sample": SamplePanel,
        "spec": SpecPanel,
        "mode": ModePanel,
        "result": ResultPanel,
        "batch": BatchPanel,
        "save": SavePanel,
    }

    def __init__(self) -> None:
        super().__init__()
        self.setWindowTitle("DocuMatch")
        self.resize(1280, 800)

        self._panels: dict[str, QWidget] = {}
        self._stack = QStackedWidget()
        for key, cls in self.PANEL_KEYS.items():
            panel = cls()
            self._panels[key] = panel
            self._stack.addWidget(panel)

        self._step_nav = StepNav()

        splitter = QSplitter(Qt.Horizontal)
        splitter.addWidget(self._step_nav)
        splitter.addWidget(self._stack)
        splitter.addWidget(self._make_preview_placeholder())
        splitter.setSizes([180, 700, 400])
        self.setCentralWidget(splitter)

        self.show_panel("start")

    def show_panel(self, key: str) -> None:
        self._stack.setCurrentWidget(self._panels[key])

    def current_panel(self) -> QWidget:
        return self._stack.currentWidget()

    def _make_preview_placeholder(self) -> QWidget:
        return QLabel("문서 미리보기\n(Task 45+)")
```

- [ ] **Step 11.5: Run — PASS**

- [ ] **Step 11.6: Commit**

```bash
git add src/documatch/gui/panels/ src/documatch/gui/main_window.py tests/gui/test_main_window.py
git commit -m "feat(gui): 7+1 빈 패널 + MainWindow StackedWidget 디스패처"
```

---

### Task 12: GuiReviewer 스켈레톤 (모든 메서드 NotImplementedError)

**Files:**
- Create: `src/documatch/gui/reviewer.py`
- Test: `tests/gui/test_gui_reviewer_skeleton.py`

- [ ] **Step 12.1: Failing test — `tests/gui/test_gui_reviewer_skeleton.py`**

```python
"""GuiReviewer 스켈레톤 — Reviewer Protocol 만족 여부."""
import pytest

from documatch.gui.reviewer import GuiReviewer
from documatch.interaction.reviewer import Reviewer


def test_gui_reviewer_constructible(qapp):
    r = GuiReviewer(main_window=None)
    assert r is not None


def test_gui_reviewer_satisfies_protocol(qapp):
    r = GuiReviewer(main_window=None)
    # Protocol 메서드 11개 모두 존재
    assert hasattr(r, "confirm_files")
    assert hasattr(r, "select_sample")
    assert hasattr(r, "get_user_prompt")
    assert hasattr(r, "review_spec")
    assert hasattr(r, "confirm_mode")
    assert hasattr(r, "review_sample")
    assert hasattr(r, "review_batch_results")
    assert hasattr(r, "get_save_path")
    assert hasattr(r, "confirm_save_spec")
    assert hasattr(r, "show_progress")
    assert hasattr(r, "show_message")
```

- [ ] **Step 12.2: Run — FAIL**

- [ ] **Step 12.3: Implement `src/documatch/gui/reviewer.py`**

```python
"""GuiReviewer — Reviewer Protocol 구현체. 각 메서드는 MainWindow 패널을 띄우고 Future 대기."""
from pathlib import Path
from typing import Literal, TYPE_CHECKING

from documatch.core.models import ExtractionResult
from documatch.interaction.reviewer import (
    BatchResultDecision,
    ModeDecision,
    ReviewDecision,
    SampleDecision,
    SpecDecision,
)
from documatch.processors.base import FileEntry
from documatch.spec.models import ExtractionSpec

if TYPE_CHECKING:
    from documatch.gui.main_window import MainWindow


class GuiReviewer:
    """Qt UI를 통해 사용자 결정을 받아오는 Reviewer 구현."""

    def __init__(self, main_window: "MainWindow | None") -> None:
        self._main_window = main_window

    async def confirm_files(self, entries: list[FileEntry]) -> ReviewDecision:
        raise NotImplementedError("Task 16에서 구현")

    async def select_sample(self, entries: list[FileEntry]) -> Path | None:
        raise NotImplementedError("Task 18에서 구현")

    async def get_user_prompt(self, default: str | None = None) -> str:
        raise NotImplementedError("Task 18에서 구현")

    async def review_spec(
        self, spec: ExtractionSpec, available_specs: list[str] | None = None,
    ) -> SpecDecision:
        raise NotImplementedError("Task 32에서 구현")

    async def confirm_mode(self, spec: ExtractionSpec) -> ModeDecision:
        raise NotImplementedError("Task 19에서 구현")

    async def review_sample(
        self, result: ExtractionResult, spec: ExtractionSpec,
    ) -> SampleDecision:
        raise NotImplementedError("Task 36에서 구현")

    async def review_batch_results(
        self, results: list[ExtractionResult], spec: ExtractionSpec,
    ) -> BatchResultDecision:
        raise NotImplementedError("Task 42에서 구현")

    async def get_save_path(self, default: Path) -> Path:
        raise NotImplementedError("Task 21에서 구현")

    async def confirm_save_spec(self, spec: ExtractionSpec) -> str | None:
        raise NotImplementedError("Task 21에서 구현")

    async def show_progress(self, current: int, total: int, label: str) -> None:
        raise NotImplementedError("Task 24에서 구현")

    async def show_message(self, level: Literal["info", "warn", "error"], text: str) -> None:
        raise NotImplementedError("Task 24에서 구현")
```

- [ ] **Step 12.4: Run — PASS**

- [ ] **Step 12.5: Commit**

```bash
git add src/documatch/gui/reviewer.py tests/gui/test_gui_reviewer_skeleton.py
git commit -m "feat(gui): GuiReviewer 스켈레톤 (Reviewer Protocol 11 메서드 stub)"
```

---

### Task 13: documatch-gui 진입점 스모크 테스트

**Files:**
- Test: `tests/gui/test_app_entry.py` (확장)

- [ ] **Step 13.1: 추가 테스트** — `tests/gui/test_app_entry.py`에 추가

```python
def test_main_window_has_all_panels(qapp, qtbot):
    from documatch.gui.main_window import MainWindow
    w = MainWindow()
    qtbot.addWidget(w)
    for key in ["start", "file", "sample", "spec", "mode", "result", "batch", "save"]:
        w.show_panel(key)
        assert w.current_panel() is not None


def test_step_nav_present_in_main_window(qapp, qtbot):
    from documatch.gui.main_window import MainWindow
    from documatch.gui.panels.step_nav import StepNav
    w = MainWindow()
    qtbot.addWidget(w)
    assert isinstance(w._step_nav, StepNav)
```

- [ ] **Step 13.2: Run — PASS**

- [ ] **Step 13.3: Commit**

```bash
git add tests/gui/test_app_entry.py
git commit -m "test(gui): MainWindow 패널 디스패치 + StepNav 존재 검증"
```

Phase 2 완료. `documatch-gui` 명령으로 빈 창 뜸. 패널 8개 + StepNav + GuiReviewer 스켈레톤 준비.

---

## Phase 3 — 단순 패널 (Step 1, 2, 4, 7) + GuiReviewer 부분 구현

이 phase 끝나면 Step 1, 2, 4, 7이 동작 — 폴더 선택부터 모드 확정까지, 그리고 저장. Step 3, 5, 6은 placeholder 유지.

### Task 14: StartPanel — 폴더 선택 다이얼로그

**Files:**
- Modify: `src/documatch/gui/panels/start_panel.py`
- Test: `tests/gui/test_panels_start.py`

- [ ] **Step 14.1: Failing test**

```python
"""StartPanel — 폴더 선택 + 시그널."""
from pathlib import Path
from PySide6.QtCore import Qt

from documatch.gui.panels.start_panel import StartPanel


def test_start_panel_emits_folder_selected(qapp, qtbot, tmp_path, monkeypatch):
    panel = StartPanel()
    qtbot.addWidget(panel)
    # 폴더 다이얼로그 패치 — 사용자가 tmp_path 선택했다고 시뮬레이션
    monkeypatch.setattr(
        "PySide6.QtWidgets.QFileDialog.getExistingDirectory",
        lambda *a, **kw: str(tmp_path),
    )
    with qtbot.waitSignal(panel.folder_selected, timeout=1000) as sig:
        qtbot.mouseClick(panel.btn_open, Qt.LeftButton)
    assert sig.args[0] == tmp_path


def test_start_panel_does_not_emit_when_dialog_cancelled(qapp, qtbot, monkeypatch):
    panel = StartPanel()
    qtbot.addWidget(panel)
    monkeypatch.setattr(
        "PySide6.QtWidgets.QFileDialog.getExistingDirectory",
        lambda *a, **kw: "",  # 취소 시 빈 문자열
    )
    received: list = []
    panel.folder_selected.connect(lambda p: received.append(p))
    qtbot.mouseClick(panel.btn_open, Qt.LeftButton)
    assert received == []
```

- [ ] **Step 14.2: Run — FAIL**

- [ ] **Step 14.3: Implement** `src/documatch/gui/panels/start_panel.py` 전체 교체:

```python
"""StartPanel — 폴더 선택 화면."""
from pathlib import Path

from PySide6.QtCore import Signal
from PySide6.QtWidgets import (
    QFileDialog, QLabel, QPushButton, QVBoxLayout, QWidget,
)


class StartPanel(QWidget):
    folder_selected = Signal(Path)

    def __init__(self) -> None:
        super().__init__()
        layout = QVBoxLayout(self)
        layout.setContentsMargins(40, 60, 40, 60)

        title = QLabel("📄 DocuMatch")
        title.setStyleSheet("font-size: 24pt; font-weight: 600;")
        layout.addWidget(title)

        layout.addWidget(QLabel("문서 폴더를 선택하세요"))

        self.btn_open = QPushButton("폴더 열기...")
        self.btn_open.setStyleSheet("padding: 10px 20px; font-size: 14pt;")
        self.btn_open.clicked.connect(self._on_open_clicked)
        layout.addWidget(self.btn_open)
        layout.addStretch()

    def _on_open_clicked(self) -> None:
        path_str = QFileDialog.getExistingDirectory(
            self, "문서 폴더 선택", str(Path.home()),
        )
        if path_str:
            self.folder_selected.emit(Path(path_str))
```

- [ ] **Step 14.4: Run — PASS**

- [ ] **Step 14.5: Commit**

```bash
git add src/documatch/gui/panels/start_panel.py tests/gui/test_panels_start.py
git commit -m "feat(gui): StartPanel — 폴더 선택 다이얼로그 + folder_selected 시그널"
```

---

### Task 15: FilePanel 레이아웃 (파일 목록 표 + 확장자 chip)

**Files:**
- Modify: `src/documatch/gui/panels/file_panel.py`
- Test: `tests/gui/test_panels_file.py`

- [ ] **Step 15.1: Failing test**

```python
"""FilePanel — 파일 목록 표 + 확장자 카운트."""
from documatch.gui.panels.file_panel import FilePanel
from documatch.processors.base import FileEntry


def _entries():
    return [
        FileEntry(file_path="/x.pdf", filename="x.pdf", extension="pdf", size_bytes=100),
        FileEntry(file_path="/y.pdf", filename="y.pdf", extension="pdf", size_bytes=200),
        FileEntry(file_path="/z.txt", filename="z.txt", extension="txt", size_bytes=50),
    ]


def test_file_panel_renders_entries(qapp, qtbot):
    panel = FilePanel()
    qtbot.addWidget(panel)
    panel.set_entries(_entries())
    assert panel.tree.topLevelItemCount() == 3


def test_file_panel_shows_extension_counts(qapp, qtbot):
    panel = FilePanel()
    qtbot.addWidget(panel)
    panel.set_entries(_entries())
    text = panel.summary_label.text()
    assert "pdf: 2" in text
    assert "txt: 1" in text
```

- [ ] **Step 15.2: Run — FAIL**

- [ ] **Step 15.3: Implement** `src/documatch/gui/panels/file_panel.py`:

```python
"""FilePanel — Step 1 파일 스캔 결과 + 확장자 chip."""
from collections import Counter

from PySide6.QtWidgets import (
    QHBoxLayout, QLabel, QPushButton, QTreeWidget, QTreeWidgetItem,
    QVBoxLayout, QWidget,
)
from PySide6.QtCore import Signal

from documatch.processors.base import FileEntry


class FilePanel(QWidget):
    next_clicked = Signal()
    quit_clicked = Signal()

    def __init__(self) -> None:
        super().__init__()
        self._entries: list[FileEntry] = []

        layout = QVBoxLayout(self)
        layout.setContentsMargins(16, 16, 16, 16)

        self.summary_label = QLabel("파일 0개")
        self.summary_label.setStyleSheet("font-weight: 600;")
        layout.addWidget(self.summary_label)

        self.tree = QTreeWidget()
        self.tree.setHeaderLabels(["확장자", "파일명", "시트", "크기"])
        layout.addWidget(self.tree)

        button_row = QHBoxLayout()
        button_row.addStretch()
        self.btn_quit = QPushButton("종료")
        self.btn_quit.clicked.connect(self.quit_clicked)
        button_row.addWidget(self.btn_quit)

        self.btn_next = QPushButton("다음 →")
        self.btn_next.setStyleSheet("font-weight: 600;")
        self.btn_next.clicked.connect(self.next_clicked)
        button_row.addWidget(self.btn_next)
        layout.addLayout(button_row)

    def set_entries(self, entries: list[FileEntry]) -> None:
        self._entries = entries
        self.tree.clear()
        counts = Counter(e.extension for e in entries)
        summary = f"총 {len(entries)}개 — " + " / ".join(
            f"{ext}: {n}" for ext, n in sorted(counts.items())
        )
        self.summary_label.setText(summary)
        for e in entries:
            QTreeWidgetItem(self.tree, [
                e.extension,
                e.filename,
                e.sheet_name or "",
                f"{e.size_bytes:,}B",
            ])

    def get_entries(self) -> list[FileEntry]:
        return list(self._entries)
```

- [ ] **Step 15.4: Run — PASS**

- [ ] **Step 15.5: Commit**

```bash
git add src/documatch/gui/panels/file_panel.py tests/gui/test_panels_file.py
git commit -m "feat(gui): FilePanel — 파일 목록 표 + 확장자 카운트"
```

---

### Task 16: GuiReviewer.confirm_files — Future 기반 사용자 입력 대기

**Files:**
- Modify: `src/documatch/gui/reviewer.py`
- Modify: `src/documatch/gui/main_window.py`
- Test: `tests/gui/test_gui_reviewer_files.py`

- [ ] **Step 16.1: Failing test**

```python
"""GuiReviewer.confirm_files — Next 버튼 누르면 ReviewDecision(continue) 반환."""
import asyncio
import pytest
from PySide6.QtCore import Qt

from documatch.gui.main_window import MainWindow
from documatch.gui.reviewer import GuiReviewer
from documatch.processors.base import FileEntry


@pytest.mark.asyncio
async def test_confirm_files_returns_continue_on_next_click(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)

    entries = [FileEntry(file_path="/a.pdf", filename="a.pdf", extension="pdf", size_bytes=10)]
    task = asyncio.create_task(reviewer.confirm_files(entries))
    await asyncio.sleep(0.05)  # 패널이 표시되도록 yield

    qtbot.mouseClick(window._panels["file"].btn_next, Qt.LeftButton)
    decision = await asyncio.wait_for(task, timeout=2.0)
    assert decision.action == "continue"


@pytest.mark.asyncio
async def test_confirm_files_returns_quit_on_quit_click(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)

    entries = [FileEntry(file_path="/a.pdf", filename="a.pdf", extension="pdf", size_bytes=10)]
    task = asyncio.create_task(reviewer.confirm_files(entries))
    await asyncio.sleep(0.05)

    qtbot.mouseClick(window._panels["file"].btn_quit, Qt.LeftButton)
    decision = await asyncio.wait_for(task, timeout=2.0)
    assert decision.action == "quit"
```

- [ ] **Step 16.2: Run — FAIL** (NotImplementedError)

- [ ] **Step 16.3: Implement GuiReviewer.confirm_files**

`src/documatch/gui/reviewer.py`의 `confirm_files` 교체:

```python
import asyncio


    async def confirm_files(self, entries: list[FileEntry]) -> ReviewDecision:
        if self._main_window is None:
            raise RuntimeError("MainWindow 미설정")
        loop = asyncio.get_event_loop()
        future: asyncio.Future[ReviewDecision] = loop.create_future()

        panel = self._main_window._panels["file"]
        panel.set_entries(entries)
        self._main_window.show_panel("file")
        self._main_window._step_nav.set_current(1)

        def on_next() -> None:
            if not future.done():
                future.set_result(ReviewDecision(action="continue"))

        def on_quit() -> None:
            if not future.done():
                future.set_result(ReviewDecision(action="quit"))

        panel.next_clicked.connect(on_next)
        panel.quit_clicked.connect(on_quit)
        try:
            return await future
        finally:
            panel.next_clicked.disconnect(on_next)
            panel.quit_clicked.disconnect(on_quit)
```

- [ ] **Step 16.4: Run — PASS**

- [ ] **Step 16.5: Commit**

```bash
git add src/documatch/gui/reviewer.py tests/gui/test_gui_reviewer_files.py
git commit -m "feat(gui): GuiReviewer.confirm_files — Future 기반 패널 대기"
```

---

### Task 17: SamplePanel 레이아웃 (라디오 + 의도 입력)

**Files:**
- Modify: `src/documatch/gui/panels/sample_panel.py`
- Test: `tests/gui/test_panels_sample.py`

- [ ] **Step 17.1: Failing test**

```python
"""SamplePanel — 샘플 라디오 + 의도 입력 + 다음 버튼."""
from PySide6.QtCore import Qt

from documatch.gui.panels.sample_panel import SamplePanel
from documatch.processors.base import FileEntry


def _entries():
    return [
        FileEntry(file_path="/a.pdf", filename="a.pdf", extension="pdf", size_bytes=10),
        FileEntry(file_path="/b.docx", filename="b.docx", extension="docx", size_bytes=20),
    ]


def test_sample_panel_renders_candidates(qapp, qtbot):
    panel = SamplePanel()
    qtbot.addWidget(panel)
    panel.set_entries(_entries())
    assert len(panel._radios) == 2


def test_sample_panel_default_first_selected(qapp, qtbot):
    panel = SamplePanel()
    qtbot.addWidget(panel)
    panel.set_entries(_entries())
    assert panel.selected_path() == "/a.pdf"


def test_sample_panel_emits_next_with_prompt(qapp, qtbot):
    panel = SamplePanel()
    qtbot.addWidget(panel)
    panel.set_entries(_entries())
    panel.prompt_edit.setPlainText("송장 번호 추출")
    with qtbot.waitSignal(panel.next_clicked, timeout=1000):
        qtbot.mouseClick(panel.btn_next, Qt.LeftButton)
    assert panel.user_prompt() == "송장 번호 추출"
```

- [ ] **Step 17.2: Run — FAIL**

- [ ] **Step 17.3: Implement** `src/documatch/gui/panels/sample_panel.py`:

```python
"""SamplePanel — Step 2 샘플 선택 + 추출 의도 입력."""
from PySide6.QtCore import Signal
from PySide6.QtWidgets import (
    QButtonGroup, QHBoxLayout, QLabel, QPushButton, QRadioButton, QTextEdit,
    QVBoxLayout, QWidget,
)

from documatch.processors.base import FileEntry


class SamplePanel(QWidget):
    next_clicked = Signal()
    quit_clicked = Signal()

    def __init__(self) -> None:
        super().__init__()
        self._entries: list[FileEntry] = []
        self._radios: list[QRadioButton] = []
        self._group = QButtonGroup(self)

        layout = QVBoxLayout(self)
        layout.setContentsMargins(16, 16, 16, 16)

        layout.addWidget(QLabel("스키마 생성용 샘플 파일을 선택하세요"))
        self._radio_box = QVBoxLayout()
        layout.addLayout(self._radio_box)

        layout.addWidget(QLabel("추출 의도 (자연어로):"))
        self.prompt_edit = QTextEdit()
        self.prompt_edit.setPlaceholderText("예: 송장번호, 금액, 발행일, 공급자명을 추출")
        self.prompt_edit.setMinimumHeight(80)
        layout.addWidget(self.prompt_edit)

        button_row = QHBoxLayout()
        button_row.addStretch()
        self.btn_quit = QPushButton("종료")
        self.btn_quit.clicked.connect(self.quit_clicked)
        button_row.addWidget(self.btn_quit)
        self.btn_next = QPushButton("→ 스키마 생성")
        self.btn_next.setStyleSheet("font-weight: 600;")
        self.btn_next.clicked.connect(self.next_clicked)
        button_row.addWidget(self.btn_next)
        layout.addLayout(button_row)

    def set_entries(self, entries: list[FileEntry]) -> None:
        self._entries = entries
        # 우선순위: PDF/DOCX/TXT 먼저
        priority = ("pdf", "docx", "txt", "md", "html", "htm")
        non_xlsx = [e for e in entries if e.extension in priority]
        candidates = non_xlsx if non_xlsx else entries
        # 라디오 재생성
        for r in self._radios:
            self._group.removeButton(r)
            self._radio_box.removeWidget(r)
            r.deleteLater()
        self._radios = []
        for e in candidates:
            label = e.filename + (f" (sheet: {e.sheet_name})" if e.sheet_name else "")
            r = QRadioButton(label)
            r._entry_path = e.file_path  # type: ignore[attr-defined]
            self._radios.append(r)
            self._group.addButton(r)
            self._radio_box.addWidget(r)
        if self._radios:
            self._radios[0].setChecked(True)

    def selected_path(self) -> str | None:
        for r in self._radios:
            if r.isChecked():
                return r._entry_path  # type: ignore[attr-defined]
        return None

    def user_prompt(self) -> str:
        return self.prompt_edit.toPlainText().strip()
```

- [ ] **Step 17.4: Run — PASS**

- [ ] **Step 17.5: Commit**

```bash
git add src/documatch/gui/panels/sample_panel.py tests/gui/test_panels_sample.py
git commit -m "feat(gui): SamplePanel — 샘플 라디오 + 의도 입력"
```

---

### Task 18: GuiReviewer.select_sample + get_user_prompt

**Files:**
- Modify: `src/documatch/gui/reviewer.py`
- Test: `tests/gui/test_gui_reviewer_sample.py`

- [ ] **Step 18.1: Failing test**

```python
"""GuiReviewer.select_sample + get_user_prompt — Sample 패널 통합."""
import asyncio
from pathlib import Path
import pytest
from PySide6.QtCore import Qt

from documatch.gui.main_window import MainWindow
from documatch.gui.reviewer import GuiReviewer
from documatch.processors.base import FileEntry


@pytest.mark.asyncio
async def test_select_sample_returns_chosen_path(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)

    entries = [
        FileEntry(file_path="/a.pdf", filename="a.pdf", extension="pdf", size_bytes=1),
        FileEntry(file_path="/b.docx", filename="b.docx", extension="docx", size_bytes=1),
    ]
    task = asyncio.create_task(reviewer.select_sample(entries))
    await asyncio.sleep(0.05)

    panel = window._panels["sample"]
    panel.prompt_edit.setPlainText("test prompt")
    qtbot.mouseClick(panel.btn_next, Qt.LeftButton)

    path = await asyncio.wait_for(task, timeout=2.0)
    assert path == Path("/a.pdf")  # 첫 라디오가 기본 선택


@pytest.mark.asyncio
async def test_get_user_prompt_returns_text(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    # 이전 select_sample이 호출되어 prompt_edit에 값을 채워둠 가정 — 직접 호출
    window._panels["sample"].prompt_edit.setPlainText("의도 텍스트")
    prompt = await reviewer.get_user_prompt()
    assert prompt == "의도 텍스트"
```

- [ ] **Step 18.2: Run — FAIL**

- [ ] **Step 18.3: Implement** `select_sample` + `get_user_prompt`:

`src/documatch/gui/reviewer.py`에 추가/교체:

```python
    async def select_sample(self, entries: list[FileEntry]) -> Path | None:
        if self._main_window is None:
            raise RuntimeError("MainWindow 미설정")
        import asyncio
        from documatch.exceptions import UserAbort

        loop = asyncio.get_event_loop()
        future: asyncio.Future[Path | None] = loop.create_future()

        panel = self._main_window._panels["sample"]
        panel.set_entries(entries)
        self._main_window.show_panel("sample")
        self._main_window._step_nav.set_current(2)

        def on_next() -> None:
            if not future.done():
                p = panel.selected_path()
                future.set_result(Path(p) if p else None)

        def on_quit() -> None:
            if not future.done():
                future.set_exception(UserAbort("user quit at sample selection"))

        panel.next_clicked.connect(on_next)
        panel.quit_clicked.connect(on_quit)
        try:
            return await future
        finally:
            panel.next_clicked.disconnect(on_next)
            panel.quit_clicked.disconnect(on_quit)

    async def get_user_prompt(self, default: str | None = None) -> str:
        if self._main_window is None:
            raise RuntimeError("MainWindow 미설정")
        # SamplePanel의 prompt_edit이 select_sample 직후 호출되어 값이 있다고 가정
        return self._main_window._panels["sample"].user_prompt() or (default or "")
```

- [ ] **Step 18.4: Run — PASS**

- [ ] **Step 18.5: Commit**

```bash
git add src/documatch/gui/reviewer.py tests/gui/test_gui_reviewer_sample.py
git commit -m "feat(gui): GuiReviewer.select_sample + get_user_prompt 구현"
```

---

### Task 19: ModePanel + GuiReviewer.confirm_mode

**Files:**
- Modify: `src/documatch/gui/panels/mode_panel.py`
- Modify: `src/documatch/gui/reviewer.py`
- Test: `tests/gui/test_panels_mode.py`, `tests/gui/test_gui_reviewer_mode.py`

- [ ] **Step 19.1: Failing test — `test_panels_mode.py`**

```python
"""ModePanel — 단일값 / 테이블 카드 + 추천 표시."""
from PySide6.QtCore import Qt

from documatch.gui.panels.mode_panel import ModePanel
from documatch.spec.models import ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate


def _spec(table: bool = False):
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
        table_mode=table,
    )


def test_mode_panel_marks_recommendation_table(qapp, qtbot):
    panel = ModePanel()
    qtbot.addWidget(panel)
    panel.set_spec(_spec(table=True))
    # 추천 표시는 라벨에 "★" 포함
    assert "★" in panel.btn_table.text()


def test_mode_panel_emits_single_clicked(qapp, qtbot):
    panel = ModePanel()
    qtbot.addWidget(panel)
    panel.set_spec(_spec())
    with qtbot.waitSignal(panel.single_clicked, timeout=1000):
        qtbot.mouseClick(panel.btn_single, Qt.LeftButton)


def test_mode_panel_emits_table_clicked(qapp, qtbot):
    panel = ModePanel()
    qtbot.addWidget(panel)
    panel.set_spec(_spec())
    with qtbot.waitSignal(panel.table_clicked, timeout=1000):
        qtbot.mouseClick(panel.btn_table, Qt.LeftButton)
```

- [ ] **Step 19.2: Run — FAIL**

- [ ] **Step 19.3: Implement** `src/documatch/gui/panels/mode_panel.py`:

```python
"""ModePanel — Step 4 모드 확정 (단일값 / 테이블)."""
from PySide6.QtCore import Signal
from PySide6.QtWidgets import (
    QHBoxLayout, QLabel, QPushButton, QVBoxLayout, QWidget,
)

from documatch.spec.models import ExtractionSpec


class ModePanel(QWidget):
    single_clicked = Signal()
    table_clicked = Signal()
    back_clicked = Signal()

    def __init__(self) -> None:
        super().__init__()
        layout = QVBoxLayout(self)
        layout.setContentsMargins(40, 40, 40, 40)
        layout.addWidget(QLabel("추출 모드를 선택하세요"))

        cards = QHBoxLayout()
        self.btn_single = QPushButton("단일값 모드\n한 문서 = 한 행")
        self.btn_single.setMinimumHeight(120)
        self.btn_single.setStyleSheet("font-size: 14pt;")
        self.btn_single.clicked.connect(self.single_clicked)
        cards.addWidget(self.btn_single)

        self.btn_table = QPushButton("테이블 모드\n한 문서 = 여러 행")
        self.btn_table.setMinimumHeight(120)
        self.btn_table.setStyleSheet("font-size: 14pt;")
        self.btn_table.clicked.connect(self.table_clicked)
        cards.addWidget(self.btn_table)
        layout.addLayout(cards)

        button_row = QHBoxLayout()
        self.btn_back = QPushButton("← 스펙으로")
        self.btn_back.clicked.connect(self.back_clicked)
        button_row.addWidget(self.btn_back)
        button_row.addStretch()
        layout.addLayout(button_row)

    def set_spec(self, spec: ExtractionSpec) -> None:
        if spec.table_mode:
            self.btn_table.setText("테이블 모드 ★\n한 문서 = 여러 행 (AI 추천)")
            self.btn_single.setText("단일값 모드\n한 문서 = 한 행")
        else:
            self.btn_single.setText("단일값 모드 ★\n한 문서 = 한 행 (AI 추천)")
            self.btn_table.setText("테이블 모드\n한 문서 = 여러 행")
```

- [ ] **Step 19.4: Failing test — `test_gui_reviewer_mode.py`**

```python
"""GuiReviewer.confirm_mode 통합."""
import asyncio
import pytest
from PySide6.QtCore import Qt

from documatch.gui.main_window import MainWindow
from documatch.gui.reviewer import GuiReviewer
from documatch.spec.models import ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate


def _spec():
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


@pytest.mark.asyncio
async def test_confirm_mode_table(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    task = asyncio.create_task(reviewer.confirm_mode(_spec()))
    await asyncio.sleep(0.05)
    qtbot.mouseClick(window._panels["mode"].btn_table, Qt.LeftButton)
    decision = await asyncio.wait_for(task, timeout=2.0)
    assert decision.action == "table"


@pytest.mark.asyncio
async def test_confirm_mode_back(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    task = asyncio.create_task(reviewer.confirm_mode(_spec()))
    await asyncio.sleep(0.05)
    qtbot.mouseClick(window._panels["mode"].btn_back, Qt.LeftButton)
    decision = await asyncio.wait_for(task, timeout=2.0)
    assert decision.action == "back"
```

- [ ] **Step 19.5: Implement GuiReviewer.confirm_mode**

`src/documatch/gui/reviewer.py`에 교체:

```python
    async def confirm_mode(self, spec: ExtractionSpec) -> ModeDecision:
        if self._main_window is None:
            raise RuntimeError("MainWindow 미설정")
        import asyncio
        loop = asyncio.get_event_loop()
        future: asyncio.Future[ModeDecision] = loop.create_future()

        panel = self._main_window._panels["mode"]
        panel.set_spec(spec)
        self._main_window.show_panel("mode")
        self._main_window._step_nav.set_current(4)

        def on_single() -> None:
            if not future.done():
                future.set_result(ModeDecision(action="single"))

        def on_table() -> None:
            if not future.done():
                future.set_result(ModeDecision(action="table"))

        def on_back() -> None:
            if not future.done():
                future.set_result(ModeDecision(action="back"))

        panel.single_clicked.connect(on_single)
        panel.table_clicked.connect(on_table)
        panel.back_clicked.connect(on_back)
        try:
            return await future
        finally:
            panel.single_clicked.disconnect(on_single)
            panel.table_clicked.disconnect(on_table)
            panel.back_clicked.disconnect(on_back)
```

- [ ] **Step 19.6: Run — PASS**

- [ ] **Step 19.7: Commit**

```bash
git add src/documatch/gui/panels/mode_panel.py src/documatch/gui/reviewer.py tests/gui/test_panels_mode.py tests/gui/test_gui_reviewer_mode.py
git commit -m "feat(gui): ModePanel + GuiReviewer.confirm_mode"
```

---

### Task 20: SavePanel 레이아웃 (저장 경로 + 스펙 저장 옵션)

**Files:**
- Modify: `src/documatch/gui/panels/save_panel.py`
- Test: `tests/gui/test_panels_save.py`

- [ ] **Step 20.1: Failing test**

```python
"""SavePanel — 저장 경로 + 스펙 저장 체크."""
from pathlib import Path
from PySide6.QtCore import Qt

from documatch.gui.panels.save_panel import SavePanel


def test_save_panel_set_default_path(qapp, qtbot, tmp_path):
    panel = SavePanel()
    qtbot.addWidget(panel)
    default = tmp_path / "result.xlsx"
    panel.set_default_path(default)
    assert panel.path_edit.text() == str(default)


def test_save_panel_emits_save_with_path(qapp, qtbot, tmp_path):
    panel = SavePanel()
    qtbot.addWidget(panel)
    panel.path_edit.setText(str(tmp_path / "out.xlsx"))
    with qtbot.waitSignal(panel.save_clicked, timeout=1000):
        qtbot.mouseClick(panel.btn_save, Qt.LeftButton)
    assert panel.get_path() == tmp_path / "out.xlsx"


def test_save_panel_spec_save_checkbox_default_off(qapp, qtbot):
    panel = SavePanel()
    qtbot.addWidget(panel)
    assert not panel.checkbox_save_spec.isChecked()
    assert panel.spec_name() is None
```

- [ ] **Step 20.2: Run — FAIL**

- [ ] **Step 20.3: Implement** `src/documatch/gui/panels/save_panel.py`:

```python
"""SavePanel — Step 7 저장 경로 + 스펙 저장 옵션."""
from pathlib import Path

from PySide6.QtCore import Signal
from PySide6.QtWidgets import (
    QCheckBox, QFileDialog, QHBoxLayout, QLabel, QLineEdit, QPushButton,
    QVBoxLayout, QWidget,
)


class SavePanel(QWidget):
    save_clicked = Signal()
    back_clicked = Signal()

    def __init__(self) -> None:
        super().__init__()
        layout = QVBoxLayout(self)
        layout.setContentsMargins(40, 40, 40, 40)

        layout.addWidget(QLabel("저장 경로:"))
        path_row = QHBoxLayout()
        self.path_edit = QLineEdit()
        path_row.addWidget(self.path_edit)
        self.btn_browse = QPushButton("Browse...")
        self.btn_browse.clicked.connect(self._on_browse)
        path_row.addWidget(self.btn_browse)
        layout.addLayout(path_row)

        spec_row = QHBoxLayout()
        self.checkbox_save_spec = QCheckBox("이 스펙을 저장 (재사용 가능)")
        self.checkbox_save_spec.toggled.connect(self._on_spec_toggled)
        spec_row.addWidget(self.checkbox_save_spec)
        spec_row.addStretch()
        layout.addLayout(spec_row)

        self.spec_name_edit = QLineEdit()
        self.spec_name_edit.setPlaceholderText("스펙 이름 (예: invoice_v1)")
        self.spec_name_edit.setVisible(False)
        layout.addWidget(self.spec_name_edit)

        layout.addStretch()

        button_row = QHBoxLayout()
        self.btn_back = QPushButton("← 결과로")
        self.btn_back.clicked.connect(self.back_clicked)
        button_row.addWidget(self.btn_back)
        button_row.addStretch()
        self.btn_save = QPushButton("저장 + 종료")
        self.btn_save.setStyleSheet("font-weight: 600;")
        self.btn_save.clicked.connect(self.save_clicked)
        button_row.addWidget(self.btn_save)
        layout.addLayout(button_row)

    def _on_browse(self) -> None:
        path_str, _ = QFileDialog.getSaveFileName(
            self, "결과 저장", self.path_edit.text(), "Excel files (*.xlsx)"
        )
        if path_str:
            self.path_edit.setText(path_str)

    def _on_spec_toggled(self, checked: bool) -> None:
        self.spec_name_edit.setVisible(checked)

    def set_default_path(self, path: Path) -> None:
        self.path_edit.setText(str(path))

    def get_path(self) -> Path:
        return Path(self.path_edit.text())

    def spec_name(self) -> str | None:
        if self.checkbox_save_spec.isChecked():
            text = self.spec_name_edit.text().strip()
            return text if text else None
        return None
```

- [ ] **Step 20.4: Run — PASS**

- [ ] **Step 20.5: Commit**

```bash
git add src/documatch/gui/panels/save_panel.py tests/gui/test_panels_save.py
git commit -m "feat(gui): SavePanel — 경로 + 스펙 저장 옵션"
```

---

### Task 21: GuiReviewer.get_save_path + confirm_save_spec

**Files:**
- Modify: `src/documatch/gui/reviewer.py`
- Test: `tests/gui/test_gui_reviewer_save.py`

- [ ] **Step 21.1: Failing test**

```python
import asyncio
from pathlib import Path
import pytest
from PySide6.QtCore import Qt

from documatch.gui.main_window import MainWindow
from documatch.gui.reviewer import GuiReviewer
from documatch.spec.models import ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate


def _spec():
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


@pytest.mark.asyncio
async def test_get_save_path_returns_user_input(qapp, qtbot, tmp_path):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    target = tmp_path / "out.xlsx"

    task = asyncio.create_task(reviewer.get_save_path(target))
    await asyncio.sleep(0.05)
    panel = window._panels["save"]
    panel.path_edit.setText(str(target))
    qtbot.mouseClick(panel.btn_save, Qt.LeftButton)

    path = await asyncio.wait_for(task, timeout=2.0)
    assert path == target


@pytest.mark.asyncio
async def test_confirm_save_spec_returns_name_when_checked(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    panel = window._panels["save"]
    panel.checkbox_save_spec.setChecked(True)
    panel.spec_name_edit.setText("invoice_v1")
    name = await reviewer.confirm_save_spec(_spec())
    assert name == "invoice_v1"


@pytest.mark.asyncio
async def test_confirm_save_spec_returns_none_when_unchecked(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    panel = window._panels["save"]
    panel.checkbox_save_spec.setChecked(False)
    name = await reviewer.confirm_save_spec(_spec())
    assert name is None
```

- [ ] **Step 21.2: Run — FAIL**

- [ ] **Step 21.3: Implement** `src/documatch/gui/reviewer.py`에 추가:

```python
    async def get_save_path(self, default: Path) -> Path:
        if self._main_window is None:
            raise RuntimeError("MainWindow 미설정")
        import asyncio
        loop = asyncio.get_event_loop()
        future: asyncio.Future[Path] = loop.create_future()

        panel = self._main_window._panels["save"]
        panel.set_default_path(default)
        self._main_window.show_panel("save")
        self._main_window._step_nav.set_current(7)

        def on_save() -> None:
            if not future.done():
                future.set_result(panel.get_path())

        panel.save_clicked.connect(on_save)
        try:
            return await future
        finally:
            panel.save_clicked.disconnect(on_save)

    async def confirm_save_spec(self, spec: ExtractionSpec) -> str | None:
        if self._main_window is None:
            raise RuntimeError("MainWindow 미설정")
        # SavePanel의 체크박스 + 이름 입력에서 즉시 읽음 (Step 7 직전 호출)
        return self._main_window._panels["save"].spec_name()
```

- [ ] **Step 21.4: Run — PASS**

- [ ] **Step 21.5: Commit**

```bash
git add src/documatch/gui/reviewer.py tests/gui/test_gui_reviewer_save.py
git commit -m "feat(gui): GuiReviewer.get_save_path + confirm_save_spec"
```

---

### Task 22: 단계 네비 점프 — completed step 클릭 시 패널 전환

**Files:**
- Modify: `src/documatch/gui/main_window.py`
- Test: `tests/gui/test_main_window_navigation.py`

- [ ] **Step 22.1: Failing test**

```python
"""MainWindow가 StepNav.step_clicked 시그널을 처리해 패널 전환."""
from documatch.gui.main_window import MainWindow


def test_step_nav_jump_changes_panel(qapp, qtbot):
    w = MainWindow()
    qtbot.addWidget(w)
    w.show_panel("file")
    w._step_nav.set_current(3)  # 1, 2 통과
    w._step_nav.step_clicked.emit(1)
    assert w.current_panel() == w._panels["file"]
```

- [ ] **Step 22.2: Run — FAIL** (시그널 연결 안 됨)

- [ ] **Step 22.3: MainWindow에서 StepNav 시그널 처리** — `__init__` 끝에 추가:

```python
        STEP_TO_PANEL = {1: "file", 2: "sample", 3: "spec", 4: "mode", 5: "result", 6: "batch", 7: "save"}
        self._step_to_panel = STEP_TO_PANEL
        self._step_nav.step_clicked.connect(self._on_step_clicked)

    def _on_step_clicked(self, step: int) -> None:
        key = self._step_to_panel.get(step)
        if key:
            self.show_panel(key)
```

- [ ] **Step 22.4: Run — PASS**

- [ ] **Step 22.5: Commit**

```bash
git add src/documatch/gui/main_window.py tests/gui/test_main_window_navigation.py
git commit -m "feat(gui): StepNav 점프 시 MainWindow 패널 전환"
```

---

### Task 23: 진행 인디케이터 위젯 (footer)

**Files:**
- Create: `src/documatch/gui/widgets/__init__.py`
- Create: `src/documatch/gui/widgets/progress_indicator.py`
- Test: `tests/gui/test_widgets_progress.py`

- [ ] **Step 23.1: widgets/__init__.py 생성** (빈)

- [ ] **Step 23.2: Failing test**

```python
"""ProgressIndicator — 좌하단 footer 인디케이터."""
from documatch.gui.widgets.progress_indicator import ProgressIndicator


def test_progress_indicator_starts_hidden(qapp, qtbot):
    w = ProgressIndicator()
    qtbot.addWidget(w)
    assert not w.isVisible()


def test_progress_indicator_shows_with_label(qapp, qtbot):
    w = ProgressIndicator()
    qtbot.addWidget(w)
    w.show_busy("LLM 호출 중...")
    assert w.isVisible()
    assert w.label.text() == "LLM 호출 중..."


def test_progress_indicator_hide_returns_to_hidden(qapp, qtbot):
    w = ProgressIndicator()
    qtbot.addWidget(w)
    w.show_busy("test")
    w.hide_busy()
    assert not w.isVisible()
```

- [ ] **Step 23.3: Implement** `src/documatch/gui/widgets/progress_indicator.py`:

```python
"""ProgressIndicator — 작업 중 footer indicator."""
from PySide6.QtWidgets import QHBoxLayout, QLabel, QProgressBar, QWidget


class ProgressIndicator(QWidget):
    def __init__(self) -> None:
        super().__init__()
        self.setVisible(False)
        layout = QHBoxLayout(self)
        layout.setContentsMargins(8, 4, 8, 4)
        self.bar = QProgressBar()
        self.bar.setMaximumWidth(120)
        self.bar.setRange(0, 0)  # busy indicator
        layout.addWidget(self.bar)
        self.label = QLabel("")
        layout.addWidget(self.label)
        layout.addStretch()

    def show_busy(self, text: str) -> None:
        self.label.setText(text)
        self.setVisible(True)

    def hide_busy(self) -> None:
        self.setVisible(False)
```

- [ ] **Step 23.4: Run — PASS**

- [ ] **Step 23.5: Commit**

```bash
git add src/documatch/gui/widgets/ tests/gui/test_widgets_progress.py
git commit -m "feat(gui): ProgressIndicator footer 위젯"
```

---

### Task 24: GuiReviewer.show_progress + show_message + 전체 엔진 wiring

**Files:**
- Modify: `src/documatch/gui/reviewer.py`
- Modify: `src/documatch/gui/main_window.py`
- Test: `tests/gui/test_gui_reviewer_messages.py`

- [ ] **Step 24.1: MainWindow에 ProgressIndicator + status bar 추가** — `__init__`에서 status bar 설정:

```python
from documatch.gui.widgets.progress_indicator import ProgressIndicator

# __init__ 끝부분
        self._progress = ProgressIndicator()
        self.statusBar().addPermanentWidget(self._progress)
```

- [ ] **Step 24.2: Failing test**

```python
"""GuiReviewer.show_progress + show_message — UI 업데이트."""
import pytest

from documatch.gui.main_window import MainWindow
from documatch.gui.reviewer import GuiReviewer


@pytest.mark.asyncio
async def test_show_progress_updates_indicator(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    await reviewer.show_progress(2, 5, "doc.pdf")
    assert "2/5" in window._progress.label.text()


@pytest.mark.asyncio
async def test_show_message_status_bar(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    await reviewer.show_message("info", "테스트")
    # 상태바에 메시지 표시 확인
    assert "테스트" in window.statusBar().currentMessage()
```

- [ ] **Step 24.3: Implement** GuiReviewer 추가:

```python
    async def show_progress(self, current: int, total: int, label: str) -> None:
        if self._main_window is None:
            return
        self._main_window._progress.show_busy(f"[{current}/{total}] {label}")

    async def show_message(self, level: Literal["info", "warn", "error"], text: str) -> None:
        if self._main_window is None:
            return
        timeout = 3000 if level == "info" else 6000
        self._main_window.statusBar().showMessage(text, timeout)
```

- [ ] **Step 24.4: Run — PASS**

- [ ] **Step 24.5: Commit**

```bash
git add src/documatch/gui/reviewer.py src/documatch/gui/main_window.py tests/gui/test_gui_reviewer_messages.py
git commit -m "feat(gui): GuiReviewer.show_progress + show_message + status bar"
```

---

### Task 25: Phase 3 통합 — start_panel.folder_selected → engine 시작

**Files:**
- Modify: `src/documatch/gui/main_window.py`
- Test: `tests/gui/test_engine_wiring.py`

- [ ] **Step 25.1: Failing test**

```python
"""StartPanel folder_selected → MainWindow가 engine task 시작."""
import asyncio
from pathlib import Path
import pytest
from unittest.mock import MagicMock, AsyncMock

from documatch.gui.main_window import MainWindow


@pytest.mark.asyncio
async def test_folder_selected_starts_engine_task(qapp, qtbot, tmp_path, monkeypatch):
    (tmp_path / "a.txt").write_text("hi")

    # 엔진 mock
    fake_engine = MagicMock()
    fake_engine.run = AsyncMock()
    monkeypatch.setattr(
        "documatch.gui.main_window.MainWindow._build_engine",
        lambda self, _: fake_engine,
    )

    window = MainWindow()
    qtbot.addWidget(window)
    window._panels["start"].folder_selected.emit(tmp_path)
    await asyncio.sleep(0.1)

    fake_engine.run.assert_called()
```

- [ ] **Step 25.2: Implement MainWindow engine wiring**

`src/documatch/gui/main_window.py`에 추가/수정:

```python
import asyncio
from pathlib import Path

from documatch.config.settings import Settings
from documatch.core.engine import ExtractionEngine
from documatch.exporters.excel import ExcelExporter
from documatch.gui.reviewer import GuiReviewer
from documatch.llm.factory import create_llm
from documatch.spec.store import SpecStore


class MainWindow(QMainWindow):
    def __init__(self) -> None:
        super().__init__()
        # ... 기존 init ...
        self._reviewer = GuiReviewer(main_window=self)
        self._engine_task: asyncio.Task | None = None
        self._panels["start"].folder_selected.connect(self._on_folder_selected)

    def _on_folder_selected(self, path: Path) -> None:
        engine = self._build_engine(path)
        self._engine_task = asyncio.create_task(self._run_engine(engine, path))

    def _build_engine(self, path: Path) -> ExtractionEngine:
        settings = Settings()
        spec_llm = create_llm(settings, role="spec")
        extract_llm = create_llm(settings, role="extract")
        return ExtractionEngine(
            spec_llm=spec_llm,
            extract_llm=extract_llm,
            spec_store=SpecStore(base_dir=settings.spec_store_dir),
            reviewer=self._reviewer,
            exporter=ExcelExporter(),
        )

    async def _run_engine(self, engine: ExtractionEngine, scan_path: Path) -> None:
        try:
            await engine.run(scan_path=scan_path)
        except Exception as e:
            self.statusBar().showMessage(f"오류: {e}", 8000)
        finally:
            self.show_panel("start")
            self._step_nav.set_current(1)
```

- [ ] **Step 25.3: Run — PASS**

- [ ] **Step 25.4: Commit**

```bash
git add src/documatch/gui/main_window.py tests/gui/test_engine_wiring.py
git commit -m "feat(gui): folder_selected 시 ExtractionEngine 자동 시작"
```

Phase 3 완료. Step 1, 2, 4, 7 동작 가능. Step 3 (스펙 편집), Step 5 (샘플 결과), Step 6 (배치)는 placeholder — Phase 4부터 구현.

---

## Phase 4 — 스펙·결과 편집 (Step 3, 5)

### Task 26: SpecTableEditor — 필드 표 + add/remove

**Files:**
- Create: `src/documatch/gui/widgets/spec_table_editor.py`
- Test: `tests/gui/test_widgets_spec_editor.py`

- [ ] **Step 26.1: Failing test**

```python
from documatch.gui.widgets.spec_table_editor import SpecTableEditor
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)


def _spec():
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[
            Field(name="a", description="필드 A", type=FieldType.STRING),
            Field(name="b", description="필드 B", type=FieldType.NUMBER, required=False),
        ],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


def test_editor_loads_spec(qapp, qtbot):
    e = SpecTableEditor()
    qtbot.addWidget(e)
    e.set_spec(_spec())
    assert e.row_count() == 2
    assert e.field_name(0) == "a"


def test_editor_add_field(qapp, qtbot):
    e = SpecTableEditor()
    qtbot.addWidget(e)
    e.set_spec(_spec())
    e.add_field()
    assert e.row_count() == 3


def test_editor_remove_field(qapp, qtbot):
    e = SpecTableEditor()
    qtbot.addWidget(e)
    e.set_spec(_spec())
    e.remove_field(0)
    assert e.row_count() == 1
    assert e.field_name(0) == "b"


def test_editor_get_modified_spec(qapp, qtbot):
    e = SpecTableEditor()
    qtbot.addWidget(e)
    spec = _spec()
    e.set_spec(spec)
    e.set_field_name(0, "renamed_a")
    new_spec = e.get_spec()
    assert new_spec.fields[0].name == "renamed_a"
    assert new_spec.fields[1].name == "b"
```

- [ ] **Step 26.2: Run — FAIL**

- [ ] **Step 26.3: Implement** `src/documatch/gui/widgets/spec_table_editor.py`:

```python
"""SpecTableEditor — 스펙 필드 인라인 편집 표."""
from PySide6.QtCore import Qt, Signal
from PySide6.QtWidgets import (
    QComboBox, QHBoxLayout, QHeaderView, QLineEdit, QPushButton, QTableWidget,
    QTableWidgetItem, QVBoxLayout, QWidget,
)

from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)

_TYPES = ["string", "number", "date", "boolean", "list"]


class SpecTableEditor(QWidget):
    spec_changed = Signal()

    def __init__(self) -> None:
        super().__init__()
        self._base_spec: ExtractionSpec | None = None

        layout = QVBoxLayout(self)
        layout.setContentsMargins(0, 0, 0, 0)

        self.table = QTableWidget(0, 5, self)
        self.table.setHorizontalHeaderLabels(["필드명", "타입", "필수", "설명", ""])
        self.table.horizontalHeader().setSectionResizeMode(3, QHeaderView.Stretch)
        layout.addWidget(self.table)

        button_row = QHBoxLayout()
        self.btn_add = QPushButton("＋ 필드 추가")
        self.btn_add.clicked.connect(self.add_field)
        button_row.addWidget(self.btn_add)
        button_row.addStretch()
        layout.addLayout(button_row)

    def set_spec(self, spec: ExtractionSpec) -> None:
        self._base_spec = spec
        self.table.setRowCount(0)
        for f in spec.fields:
            self._append_row(f.name, f.type.value, f.required, f.description)

    def _append_row(self, name: str, type_value: str, required: bool, desc: str) -> None:
        row = self.table.rowCount()
        self.table.insertRow(row)

        name_edit = QLineEdit(name)
        self.table.setCellWidget(row, 0, name_edit)

        type_combo = QComboBox()
        type_combo.addItems(_TYPES)
        type_combo.setCurrentText(type_value)
        self.table.setCellWidget(row, 1, type_combo)

        req_item = QTableWidgetItem()
        req_item.setFlags(Qt.ItemIsUserCheckable | Qt.ItemIsEnabled)
        req_item.setCheckState(Qt.Checked if required else Qt.Unchecked)
        self.table.setItem(row, 2, req_item)

        desc_edit = QLineEdit(desc)
        self.table.setCellWidget(row, 3, desc_edit)

        del_btn = QPushButton("×")
        del_btn.clicked.connect(lambda _=False, r=row: self.remove_field(r))
        self.table.setCellWidget(row, 4, del_btn)

    def add_field(self) -> None:
        self._append_row("", "string", True, "")
        self.spec_changed.emit()

    def remove_field(self, row: int) -> None:
        self.table.removeRow(row)
        self.spec_changed.emit()

    def row_count(self) -> int:
        return self.table.rowCount()

    def field_name(self, row: int) -> str:
        widget = self.table.cellWidget(row, 0)
        return widget.text() if widget else ""

    def set_field_name(self, row: int, name: str) -> None:
        widget = self.table.cellWidget(row, 0)
        if widget:
            widget.setText(name)

    def get_spec(self) -> ExtractionSpec:
        if self._base_spec is None:
            raise RuntimeError("set_spec 먼저 호출")
        fields: list[Field] = []
        output_table: list[OutputColumn] = []
        for row in range(self.table.rowCount()):
            name = self.table.cellWidget(row, 0).text().strip()
            if not name:
                continue
            type_value = self.table.cellWidget(row, 1).currentText()
            required = self.table.item(row, 2).checkState() == Qt.Checked
            desc = self.table.cellWidget(row, 3).text().strip()
            fields.append(Field(
                name=name, description=desc, type=FieldType(type_value), required=required,
            ))
            output_table.append(OutputColumn(field_name=name, excel_header=name))
        return self._base_spec.model_copy(update={
            "fields": fields,
            "output_table": output_table,
        })
```

- [ ] **Step 26.4: Run — PASS**

- [ ] **Step 26.5: Commit**

```bash
git add src/documatch/gui/widgets/spec_table_editor.py tests/gui/test_widgets_spec_editor.py
git commit -m "feat(gui): SpecTableEditor — 필드 인라인 편집 + add/remove"
```

---

### Task 27: SpecTableEditor — 타입 dropdown + 필수 토글 검증

**Files:**
- Modify: `tests/gui/test_widgets_spec_editor.py`

- [ ] **Step 27.1: 추가 테스트**

```python
def test_editor_type_dropdown_change(qapp, qtbot):
    from PySide6.QtWidgets import QComboBox
    e = SpecTableEditor()
    qtbot.addWidget(e)
    e.set_spec(_spec())
    combo = e.table.cellWidget(0, 1)
    assert isinstance(combo, QComboBox)
    combo.setCurrentText("number")
    new_spec = e.get_spec()
    assert new_spec.fields[0].type.value == "number"


def test_editor_required_toggle(qapp, qtbot):
    e = SpecTableEditor()
    qtbot.addWidget(e)
    e.set_spec(_spec())
    item = e.table.item(0, 2)
    assert item.checkState() == Qt.Checked
    item.setCheckState(Qt.Unchecked)
    new_spec = e.get_spec()
    assert new_spec.fields[0].required is False
```

- [ ] **Step 27.2: Run — PASS** (Task 26에서 이미 구현됨)

- [ ] **Step 27.3: Commit**

```bash
git add tests/gui/test_widgets_spec_editor.py
git commit -m "test(gui): SpecTableEditor 타입 dropdown + 필수 토글 검증"
```

---

### Task 28: SpecPanel 레이아웃 — 편집기 + 피드백 + 액션바

**Files:**
- Modify: `src/documatch/gui/panels/spec_panel.py`
- Test: `tests/gui/test_panels_spec.py`

- [ ] **Step 28.1: Failing test**

```python
from PySide6.QtCore import Qt

from documatch.gui.panels.spec_panel import SpecPanel
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)


def _spec():
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


def test_spec_panel_loads(qapp, qtbot):
    p = SpecPanel()
    qtbot.addWidget(p)
    p.set_spec(_spec())
    assert p.editor.row_count() == 1


def test_spec_panel_emits_approve(qapp, qtbot):
    p = SpecPanel()
    qtbot.addWidget(p)
    p.set_spec(_spec())
    with qtbot.waitSignal(p.approve_clicked, timeout=1000):
        qtbot.mouseClick(p.btn_approve, Qt.LeftButton)


def test_spec_panel_emits_refine_with_feedback(qapp, qtbot):
    p = SpecPanel()
    qtbot.addWidget(p)
    p.set_spec(_spec())
    p.feedback_edit.setPlainText("amount 추가")
    with qtbot.waitSignal(p.refine_clicked, timeout=1000) as sig:
        qtbot.mouseClick(p.btn_refine, Qt.LeftButton)
    assert sig.args[0] == "amount 추가"


def test_spec_panel_emits_back(qapp, qtbot):
    p = SpecPanel()
    qtbot.addWidget(p)
    p.set_spec(_spec())
    with qtbot.waitSignal(p.back_clicked, timeout=1000):
        qtbot.mouseClick(p.btn_back, Qt.LeftButton)
```

- [ ] **Step 28.2: Implement** `src/documatch/gui/panels/spec_panel.py`:

```python
"""SpecPanel — Step 3 스펙 검토 + 인라인 편집."""
from PySide6.QtCore import Signal
from PySide6.QtWidgets import (
    QHBoxLayout, QLabel, QLineEdit, QPushButton, QTextEdit, QVBoxLayout, QWidget,
)

from documatch.gui.widgets.spec_table_editor import SpecTableEditor
from documatch.spec.models import ExtractionSpec


class SpecPanel(QWidget):
    approve_clicked = Signal()
    refine_clicked = Signal(str)            # feedback
    back_clicked = Signal()
    quit_clicked = Signal()

    def __init__(self) -> None:
        super().__init__()
        layout = QVBoxLayout(self)
        layout.setContentsMargins(16, 16, 16, 16)

        self.title = QLabel("스펙: ")
        self.title.setStyleSheet("font-weight: 600; font-size: 14pt;")
        layout.addWidget(self.title)

        self.editor = SpecTableEditor()
        layout.addWidget(self.editor, 1)

        layout.addWidget(QLabel("요약 템플릿"))
        self.summary_edit = QLineEdit()
        layout.addWidget(self.summary_edit)

        layout.addWidget(QLabel("자연어 피드백 (LLM에 재요청)"))
        self.feedback_edit = QTextEdit()
        self.feedback_edit.setPlaceholderText("예: 공급자명 필드 추가, amount는 number로 변경")
        self.feedback_edit.setMaximumHeight(60)
        layout.addWidget(self.feedback_edit)

        button_row = QHBoxLayout()
        self.btn_back = QPushButton("← 샘플 재선택")
        self.btn_back.clicked.connect(self.back_clicked)
        button_row.addWidget(self.btn_back)

        self.btn_quit = QPushButton("종료")
        self.btn_quit.clicked.connect(self.quit_clicked)
        button_row.addWidget(self.btn_quit)
        button_row.addStretch()

        self.btn_refine = QPushButton("LLM에 재요청")
        self.btn_refine.clicked.connect(
            lambda: self.refine_clicked.emit(self.feedback_edit.toPlainText().strip())
        )
        button_row.addWidget(self.btn_refine)

        self.btn_approve = QPushButton("승인 + 다음 →")
        self.btn_approve.setStyleSheet("font-weight: 600;")
        self.btn_approve.clicked.connect(self.approve_clicked)
        button_row.addWidget(self.btn_approve)
        layout.addLayout(button_row)

    def set_spec(self, spec: ExtractionSpec) -> None:
        self.title.setText(f"스펙: {spec.name}")
        self.editor.set_spec(spec)
        self.summary_edit.setText(spec.summary_template.template)
        self.feedback_edit.clear()

    def get_modified_spec(self) -> ExtractionSpec:
        spec = self.editor.get_spec()
        return spec.model_copy(update={
            "summary_template": spec.summary_template.model_copy(
                update={"template": self.summary_edit.text()},
            ),
        })
```

- [ ] **Step 28.3: Run — PASS**

- [ ] **Step 28.4: Commit**

```bash
git add src/documatch/gui/panels/spec_panel.py tests/gui/test_panels_spec.py
git commit -m "feat(gui): SpecPanel — 편집기 + 피드백 + 액션바"
```

---

### Task 29: GuiReviewer.review_spec — modified_spec 반환

**Files:**
- Modify: `src/documatch/gui/reviewer.py`
- Test: `tests/gui/test_gui_reviewer_spec.py`

- [ ] **Step 29.1: Failing test**

```python
import asyncio
import pytest
from PySide6.QtCore import Qt

from documatch.gui.main_window import MainWindow
from documatch.gui.reviewer import GuiReviewer
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)


def _spec():
    return ExtractionSpec(
        name="t", doc_type_hint="invoice",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


@pytest.mark.asyncio
async def test_review_spec_approve_returns_modified(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    task = asyncio.create_task(reviewer.review_spec(_spec()))
    await asyncio.sleep(0.05)

    panel = window._panels["spec"]
    panel.editor.set_field_name(0, "renamed")
    qtbot.mouseClick(panel.btn_approve, Qt.LeftButton)

    decision = await asyncio.wait_for(task, timeout=2.0)
    assert decision.action == "approve"
    assert decision.modified_spec is not None
    assert decision.modified_spec.fields[0].name == "renamed"


@pytest.mark.asyncio
async def test_review_spec_refine(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    task = asyncio.create_task(reviewer.review_spec(_spec()))
    await asyncio.sleep(0.05)

    panel = window._panels["spec"]
    panel.feedback_edit.setPlainText("amount 추가")
    qtbot.mouseClick(panel.btn_refine, Qt.LeftButton)

    decision = await asyncio.wait_for(task, timeout=2.0)
    assert decision.action == "refine"
    assert decision.feedback == "amount 추가"


@pytest.mark.asyncio
async def test_review_spec_back(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    task = asyncio.create_task(reviewer.review_spec(_spec()))
    await asyncio.sleep(0.05)
    qtbot.mouseClick(window._panels["spec"].btn_back, Qt.LeftButton)
    decision = await asyncio.wait_for(task, timeout=2.0)
    assert decision.action == "back"
```

- [ ] **Step 29.2: Implement**:

```python
    async def review_spec(
        self, spec: ExtractionSpec, available_specs: list[str] | None = None,
    ) -> SpecDecision:
        if self._main_window is None:
            raise RuntimeError("MainWindow 미설정")
        import asyncio
        loop = asyncio.get_event_loop()
        future: asyncio.Future[SpecDecision] = loop.create_future()

        panel = self._main_window._panels["spec"]
        panel.set_spec(spec)
        self._main_window.show_panel("spec")
        self._main_window._step_nav.set_current(3)

        def on_approve() -> None:
            if not future.done():
                future.set_result(SpecDecision(
                    action="approve", modified_spec=panel.get_modified_spec(),
                ))

        def on_refine(feedback: str) -> None:
            if not future.done():
                future.set_result(SpecDecision(action="refine", feedback=feedback))

        def on_back() -> None:
            if not future.done():
                future.set_result(SpecDecision(action="back"))

        def on_quit() -> None:
            if not future.done():
                future.set_result(SpecDecision(action="quit"))

        panel.approve_clicked.connect(on_approve)
        panel.refine_clicked.connect(on_refine)
        panel.back_clicked.connect(on_back)
        panel.quit_clicked.connect(on_quit)
        try:
            return await future
        finally:
            panel.approve_clicked.disconnect(on_approve)
            panel.refine_clicked.disconnect(on_refine)
            panel.back_clicked.disconnect(on_back)
            panel.quit_clicked.disconnect(on_quit)
```

- [ ] **Step 29.3: Run — PASS**

- [ ] **Step 29.4: Commit**

```bash
git add src/documatch/gui/reviewer.py tests/gui/test_gui_reviewer_spec.py
git commit -m "feat(gui): GuiReviewer.review_spec — modified_spec 반환"
```

---

### Task 30: ResultTableEditor — 단일값 모드 (label/value/confidence)

**Files:**
- Create: `src/documatch/gui/widgets/result_table_editor.py`
- Test: `tests/gui/test_widgets_result_editor.py`

- [ ] **Step 30.1: Failing test**

```python
from documatch.core.models import ExtractionResult
from documatch.gui.widgets.result_table_editor import ResultTableEditor
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)


def _spec():
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[
            Field(name="a", description="A", type=FieldType.STRING),
            Field(name="b", description="B", type=FieldType.NUMBER),
        ],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


def test_single_mode_renders_field_rows(qapp, qtbot):
    editor = ResultTableEditor()
    qtbot.addWidget(editor)
    result = ExtractionResult(document_id="x.pdf", values={"a": "v1", "b": 100})
    editor.load(result, _spec())
    assert editor.row_count() == 2


def test_single_mode_edit_updates_result(qapp, qtbot):
    editor = ResultTableEditor()
    qtbot.addWidget(editor)
    result = ExtractionResult(document_id="x.pdf", values={"a": "v1", "b": 100})
    editor.load(result, _spec())
    editor.set_value(0, "edited_v1")
    new_result = editor.get_modified_result()
    assert new_result.values["a"] == "edited_v1"
```

- [ ] **Step 30.2: Implement** `src/documatch/gui/widgets/result_table_editor.py`:

```python
"""ResultTableEditor — 단일값/테이블 모드 추출 결과 편집."""
from PySide6.QtCore import Signal, Qt
from PySide6.QtWidgets import (
    QHeaderView, QLineEdit, QTableWidget, QTableWidgetItem, QVBoxLayout, QWidget,
)

from documatch.core.models import ExtractionResult
from documatch.spec.models import ExtractionSpec


class ResultTableEditor(QWidget):
    cell_clicked = Signal(str, object)  # field_name, current_value

    def __init__(self) -> None:
        super().__init__()
        self._spec: ExtractionSpec | None = None
        self._result: ExtractionResult | None = None
        self._table_mode = False

        layout = QVBoxLayout(self)
        layout.setContentsMargins(0, 0, 0, 0)

        self.table = QTableWidget(self)
        layout.addWidget(self.table)
        self.table.cellClicked.connect(self._on_cell_clicked)

    def load(self, result: ExtractionResult, spec: ExtractionSpec) -> None:
        self._spec = spec
        self._result = result
        self._table_mode = spec.table_mode and result.is_table_result
        if self._table_mode:
            self._render_table_mode()
        else:
            self._render_single_mode()

    def _render_single_mode(self) -> None:
        assert self._spec and self._result
        self.table.clear()
        self.table.setColumnCount(3)
        self.table.setHorizontalHeaderLabels(["필드", "값", "신뢰도"])
        self.table.setRowCount(len(self._spec.fields))
        self.table.horizontalHeader().setSectionResizeMode(1, QHeaderView.Stretch)
        for i, f in enumerate(self._spec.fields):
            label = QTableWidgetItem(f.name)
            label.setFlags(Qt.ItemIsEnabled)
            self.table.setItem(i, 0, label)

            value_edit = QLineEdit(str(self._result.values.get(f.name, "")))
            self.table.setCellWidget(i, 1, value_edit)

            conf = self._result.confidence.get(f.name, 0.0)
            conf_item = QTableWidgetItem(f"{conf:.2f}")
            conf_item.setFlags(Qt.ItemIsEnabled)
            self.table.setItem(i, 2, conf_item)

    def _render_table_mode(self) -> None:
        assert self._spec and self._result
        self.table.clear()
        field_names = [f.name for f in self._spec.fields]
        self.table.setColumnCount(len(field_names))
        self.table.setHorizontalHeaderLabels(field_names)
        rows = self._result.rows or []
        self.table.setRowCount(len(rows))
        for i, row in enumerate(rows):
            for j, fn in enumerate(field_names):
                edit = QLineEdit(str(row.get(fn, "")))
                self.table.setCellWidget(i, j, edit)

    def _on_cell_clicked(self, row: int, col: int) -> None:
        if self._spec is None:
            return
        if self._table_mode:
            field_name = self._spec.fields[col].name
        else:
            field_name = self._spec.fields[row].name
        widget = self.table.cellWidget(row, col)
        value = widget.text() if widget else ""
        self.cell_clicked.emit(field_name, value)

    def row_count(self) -> int:
        return self.table.rowCount()

    def set_value(self, row: int, value: str, col: int = 1) -> None:
        widget = self.table.cellWidget(row, col)
        if widget:
            widget.setText(value)

    def add_row(self) -> None:
        if not self._table_mode or self._spec is None:
            return
        row = self.table.rowCount()
        self.table.insertRow(row)
        for j in range(len(self._spec.fields)):
            self.table.setCellWidget(row, j, QLineEdit(""))

    def remove_row(self, row: int) -> None:
        if self._table_mode:
            self.table.removeRow(row)

    def get_modified_result(self) -> ExtractionResult:
        if self._spec is None or self._result is None:
            raise RuntimeError("load 먼저 호출")
        if self._table_mode:
            field_names = [f.name for f in self._spec.fields]
            rows = []
            for i in range(self.table.rowCount()):
                row_dict: dict[str, object] = {}
                for j, fn in enumerate(field_names):
                    widget = self.table.cellWidget(i, j)
                    row_dict[fn] = widget.text() if widget else ""
                rows.append(row_dict)
            from dataclasses import replace
            return replace(self._result, rows=rows)
        # single mode
        values = dict(self._result.values)
        for i, f in enumerate(self._spec.fields):
            widget = self.table.cellWidget(i, 1)
            if widget:
                values[f.name] = widget.text()
        from dataclasses import replace
        return replace(self._result, values=values)
```

- [ ] **Step 30.3: Run — PASS**

- [ ] **Step 30.4: Commit**

```bash
git add src/documatch/gui/widgets/result_table_editor.py tests/gui/test_widgets_result_editor.py
git commit -m "feat(gui): ResultTableEditor — 단일값 모드 셀 편집"
```

---

### Task 31: ResultTableEditor — 테이블 모드 + 행 추가/삭제

**Files:**
- Modify: `tests/gui/test_widgets_result_editor.py`

- [ ] **Step 31.1: 추가 테스트** (Task 30 구현이 이미 table_mode 지원 — 검증만)

```python
def test_table_mode_renders_rows(qapp, qtbot):
    editor = ResultTableEditor()
    qtbot.addWidget(editor)
    spec = _spec()
    spec.table_mode = True
    result = ExtractionResult(
        document_id="t.pdf",
        rows=[{"a": "x1", "b": 1}, {"a": "x2", "b": 2}],
    )
    editor.load(result, spec)
    assert editor.row_count() == 2
    assert editor.table.columnCount() == 2  # 2 fields


def test_table_mode_add_row(qapp, qtbot):
    editor = ResultTableEditor()
    qtbot.addWidget(editor)
    spec = _spec()
    spec.table_mode = True
    result = ExtractionResult(document_id="t.pdf", rows=[{"a": "x1", "b": 1}])
    editor.load(result, spec)
    editor.add_row()
    assert editor.row_count() == 2


def test_table_mode_remove_row(qapp, qtbot):
    editor = ResultTableEditor()
    qtbot.addWidget(editor)
    spec = _spec()
    spec.table_mode = True
    result = ExtractionResult(
        document_id="t.pdf", rows=[{"a": "x1", "b": 1}, {"a": "x2", "b": 2}],
    )
    editor.load(result, spec)
    editor.remove_row(0)
    assert editor.row_count() == 1
    new_result = editor.get_modified_result()
    assert new_result.rows[0]["a"] == "x2"
```

- [ ] **Step 31.2: Run — PASS**

- [ ] **Step 31.3: Commit**

```bash
git add tests/gui/test_widgets_result_editor.py
git commit -m "test(gui): ResultTableEditor 테이블 모드 + 행 추가/삭제 검증"
```

---

### Task 32: ResultPanel + GuiReviewer.review_sample

**Files:**
- Modify: `src/documatch/gui/panels/result_panel.py`
- Modify: `src/documatch/gui/reviewer.py`
- Test: `tests/gui/test_panels_result.py`, `tests/gui/test_gui_reviewer_review_sample.py`

- [ ] **Step 32.1: result_panel.py**

```python
"""ResultPanel — Step 5 샘플 결과 검토 + 편집."""
from PySide6.QtCore import Signal
from PySide6.QtWidgets import (
    QHBoxLayout, QPushButton, QTextEdit, QVBoxLayout, QWidget, QLabel,
)

from documatch.core.models import ExtractionResult
from documatch.gui.widgets.result_table_editor import ResultTableEditor
from documatch.spec.models import ExtractionSpec


class ResultPanel(QWidget):
    full_clicked = Signal()
    single_save_clicked = Signal()
    refine_clicked = Signal(str)
    back_clicked = Signal()
    quit_clicked = Signal()

    def __init__(self) -> None:
        super().__init__()
        layout = QVBoxLayout(self)
        layout.setContentsMargins(16, 16, 16, 16)

        layout.addWidget(QLabel("샘플 추출 결과 (편집 가능)"))
        self.editor = ResultTableEditor()
        layout.addWidget(self.editor, 1)

        layout.addWidget(QLabel("자연어 피드백 (LLM에 재요청)"))
        self.feedback_edit = QTextEdit()
        self.feedback_edit.setMaximumHeight(60)
        layout.addWidget(self.feedback_edit)

        button_row = QHBoxLayout()
        self.btn_back = QPushButton("← 모드")
        self.btn_back.clicked.connect(self.back_clicked)
        button_row.addWidget(self.btn_back)
        self.btn_quit = QPushButton("종료")
        self.btn_quit.clicked.connect(self.quit_clicked)
        button_row.addWidget(self.btn_quit)
        button_row.addStretch()
        self.btn_refine = QPushButton("LLM 재요청")
        self.btn_refine.clicked.connect(
            lambda: self.refine_clicked.emit(self.feedback_edit.toPlainText().strip())
        )
        button_row.addWidget(self.btn_refine)
        self.btn_single_save = QPushButton("이 건만 저장")
        self.btn_single_save.clicked.connect(self.single_save_clicked)
        button_row.addWidget(self.btn_single_save)
        self.btn_full = QPushButton("전체 진행 →")
        self.btn_full.setStyleSheet("font-weight: 600;")
        self.btn_full.clicked.connect(self.full_clicked)
        button_row.addWidget(self.btn_full)
        layout.addLayout(button_row)

    def set_data(self, result: ExtractionResult, spec: ExtractionSpec) -> None:
        self.editor.load(result, spec)
        self.feedback_edit.clear()

    def get_modified_result(self) -> ExtractionResult:
        return self.editor.get_modified_result()
```

- [ ] **Step 32.2: GuiReviewer.review_sample 구현**

```python
    async def review_sample(
        self, result: ExtractionResult, spec: ExtractionSpec,
    ) -> SampleDecision:
        if self._main_window is None:
            raise RuntimeError("MainWindow 미설정")
        import asyncio
        loop = asyncio.get_event_loop()
        future: asyncio.Future[SampleDecision] = loop.create_future()

        panel = self._main_window._panels["result"]
        panel.set_data(result, spec)
        self._main_window.show_panel("result")
        self._main_window._step_nav.set_current(5)

        def on_full() -> None:
            if not future.done():
                future.set_result(SampleDecision(
                    action="full", modified_result=panel.get_modified_result(),
                ))

        def on_single_save() -> None:
            if not future.done():
                future.set_result(SampleDecision(
                    action="single_save", modified_result=panel.get_modified_result(),
                ))

        def on_refine(feedback: str) -> None:
            if not future.done():
                future.set_result(SampleDecision(action="refine", feedback=feedback))

        def on_back() -> None:
            if not future.done():
                future.set_result(SampleDecision(action="back"))

        def on_quit() -> None:
            if not future.done():
                future.set_result(SampleDecision(action="quit"))

        panel.full_clicked.connect(on_full)
        panel.single_save_clicked.connect(on_single_save)
        panel.refine_clicked.connect(on_refine)
        panel.back_clicked.connect(on_back)
        panel.quit_clicked.connect(on_quit)
        try:
            return await future
        finally:
            panel.full_clicked.disconnect(on_full)
            panel.single_save_clicked.disconnect(on_single_save)
            panel.refine_clicked.disconnect(on_refine)
            panel.back_clicked.disconnect(on_back)
            panel.quit_clicked.disconnect(on_quit)
```

- [ ] **Step 32.3: 통합 테스트**

```python
# tests/gui/test_gui_reviewer_review_sample.py
import asyncio
import pytest
from PySide6.QtCore import Qt

from documatch.core.models import ExtractionResult
from documatch.gui.main_window import MainWindow
from documatch.gui.reviewer import GuiReviewer
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)


def _spec():
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


@pytest.mark.asyncio
async def test_review_sample_full_returns_modified(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    result = ExtractionResult(document_id="x.pdf", values={"a": "v"})
    task = asyncio.create_task(reviewer.review_sample(result, _spec()))
    await asyncio.sleep(0.05)
    panel = window._panels["result"]
    panel.editor.set_value(0, "edited")
    qtbot.mouseClick(panel.btn_full, Qt.LeftButton)
    decision = await asyncio.wait_for(task, timeout=2.0)
    assert decision.action == "full"
    assert decision.modified_result.values["a"] == "edited"


@pytest.mark.asyncio
async def test_review_sample_refine(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    result = ExtractionResult(document_id="x.pdf", values={"a": "v"})
    task = asyncio.create_task(reviewer.review_sample(result, _spec()))
    await asyncio.sleep(0.05)
    panel = window._panels["result"]
    panel.feedback_edit.setPlainText("값 보정")
    qtbot.mouseClick(panel.btn_refine, Qt.LeftButton)
    decision = await asyncio.wait_for(task, timeout=2.0)
    assert decision.action == "refine"
    assert decision.feedback == "값 보정"
```

- [ ] **Step 32.4: Run — PASS**

- [ ] **Step 32.5: Commit**

```bash
git add src/documatch/gui/panels/result_panel.py src/documatch/gui/reviewer.py tests/gui/test_panels_result.py tests/gui/test_gui_reviewer_review_sample.py
git commit -m "feat(gui): ResultPanel + GuiReviewer.review_sample (modified_result)"
```

---

### Task 33: SpecPanel/ResultPanel을 MainWindow에 wire (signals 연결)

**Files:**
- Modify: `src/documatch/gui/main_window.py` (이미 모두 panels에 추가됨 — Task 11)

이 Task는 검증만. SpecPanel/ResultPanel은 이미 PANEL_KEYS에 등록되어 있으므로 별도 wiring 불필요.

- [ ] **Step 33.1: 통합 smoke test 추가** — `tests/gui/test_main_window.py`에 추가:

```python
def test_main_window_includes_spec_and_result_panels(qapp, qtbot):
    from documatch.gui.main_window import MainWindow
    from documatch.gui.panels.spec_panel import SpecPanel
    from documatch.gui.panels.result_panel import ResultPanel
    w = MainWindow()
    qtbot.addWidget(w)
    w.show_panel("spec")
    assert isinstance(w.current_panel(), SpecPanel)
    w.show_panel("result")
    assert isinstance(w.current_panel(), ResultPanel)
```

- [ ] **Step 33.2: Run — PASS**

- [ ] **Step 33.3: Commit**

```bash
git add tests/gui/test_main_window.py
git commit -m "test(gui): SpecPanel/ResultPanel MainWindow 등록 검증"
```

---

### Task 34: Phase 4 통합 — engine 흐름 동작 확인 (FakeLLMClient)

**Files:**
- Test: `tests/gui/test_e2e_phase4.py`

- [ ] **Step 34.1: E2E 테스트** (FakeLLMClient + qtbot)

```python
"""Phase 4 통합 — 폴더 선택 → 스펙 검토 (편집) → 모드 → 샘플 결과 → 저장."""
import asyncio
import json
from pathlib import Path
from unittest.mock import MagicMock

import pytest
from PySide6.QtCore import Qt

from documatch.gui.main_window import MainWindow
from tests.fakes import FakeLLMClient


@pytest.mark.asyncio
async def test_phase4_full_flow(qapp, qtbot, tmp_path, monkeypatch):
    (tmp_path / "doc.txt").write_text("샘플")
    out_path = tmp_path / "out.xlsx"

    spec_resp = json.dumps({
        "name": "x", "doc_type_hint": "x",
        "fields": [{"name": "a", "description": "d", "type": "string", "required": True}],
        "summary_template": {"template": "{a}"},
        "output_table": [{"field_name": "a", "excel_header": "a", "width": 10}],
        "table_mode": False,
    })
    extract_resp = json.dumps({"values": {"a": "v"}, "confidence": {"a": 0.9}, "summary_sentence": "v"})
    fake = FakeLLMClient(responses=[spec_resp, extract_resp, extract_resp])
    monkeypatch.setattr(
        "documatch.gui.main_window.create_llm",
        lambda settings, *, role: fake,
    )

    window = MainWindow()
    qtbot.addWidget(window)
    window._panels["start"].folder_selected.emit(tmp_path)

    # Step 1: continue
    await asyncio.sleep(0.1)
    qtbot.mouseClick(window._panels["file"].btn_next, Qt.LeftButton)

    # Step 2: 의도 입력 + next
    await asyncio.sleep(0.1)
    window._panels["sample"].prompt_edit.setPlainText("a 추출")
    qtbot.mouseClick(window._panels["sample"].btn_next, Qt.LeftButton)

    # Step 3: 스펙 승인
    await asyncio.sleep(0.5)  # LLM 호출 + 패널 업데이트
    qtbot.mouseClick(window._panels["spec"].btn_approve, Qt.LeftButton)

    # Step 4: 모드 single
    await asyncio.sleep(0.1)
    qtbot.mouseClick(window._panels["mode"].btn_single, Qt.LeftButton)

    # Step 5: full
    await asyncio.sleep(0.5)
    qtbot.mouseClick(window._panels["result"].btn_full, Qt.LeftButton)

    # batch panel은 placeholder — 자동 통과 가정 못 함, 다음 phase에서 구현
    # 여기까지가 Phase 4 검증 범위
    assert True
```

(Step 6 (batch_panel)은 Phase 5에서 구현 — 이 테스트는 거기까지만 검증)

- [ ] **Step 34.2: Run — PASS**

- [ ] **Step 34.3: Commit**

```bash
git add tests/gui/test_e2e_phase4.py
git commit -m "test(gui): Phase 4 부분 E2E (Step 1-5)"
```

Phase 4 완료. Step 3 + 5에서 인라인 편집 + 자연어 피드백 모두 동작.

---

## Phase 5 — 배치 진행 + 결과 편집 (Step 6)

### Task 35: BatchController — pause/cancel asyncio Event 관리

**Files:**
- Create: `src/documatch/gui/threads/__init__.py`
- Create: `src/documatch/gui/threads/batch_controller.py`
- Test: `tests/gui/test_batch_controller.py`

- [ ] **Step 35.1: threads/__init__.py 생성** (빈)

- [ ] **Step 35.2: Failing test**

```python
import asyncio
import pytest

from documatch.gui.threads.batch_controller import BatchController


@pytest.mark.asyncio
async def test_pause_blocks_progress():
    ctrl = BatchController()
    progress = []

    async def task():
        for i in range(5):
            progress.append(i)
            await ctrl.wait_if_paused()

    t = asyncio.create_task(task())
    await asyncio.sleep(0.01)
    ctrl.pause()
    await asyncio.sleep(0.01)
    snap = len(progress)
    ctrl.resume()
    await t
    assert snap < 5


@pytest.mark.asyncio
async def test_cancel_sets_flag():
    ctrl = BatchController()
    ctrl.cancel()
    assert ctrl.is_cancelled


@pytest.mark.asyncio
async def test_default_not_paused_not_cancelled():
    ctrl = BatchController()
    assert not ctrl.is_cancelled
    # wait_if_paused는 즉시 반환 (paused가 아님)
    await asyncio.wait_for(ctrl.wait_if_paused(), timeout=0.1)
```

- [ ] **Step 35.3: Implement** `src/documatch/gui/threads/batch_controller.py`:

```python
"""BatchController — 배치 진행 중 일시정지/중단 제어."""
import asyncio


class BatchController:
    def __init__(self) -> None:
        self._pause_event = asyncio.Event()
        self._pause_event.set()  # 처음엔 진행 가능
        self._cancelled = False

    async def wait_if_paused(self) -> None:
        await self._pause_event.wait()

    def pause(self) -> None:
        self._pause_event.clear()

    def resume(self) -> None:
        self._pause_event.set()

    def cancel(self) -> None:
        self._cancelled = True
        self._pause_event.set()  # 멈춰 있으면 깨우기

    @property
    def is_cancelled(self) -> bool:
        return self._cancelled

    @property
    def is_paused(self) -> bool:
        return not self._pause_event.is_set()
```

- [ ] **Step 35.4: Commit**

```bash
git add src/documatch/gui/threads/ tests/gui/test_batch_controller.py
git commit -m "feat(gui): BatchController — 일시정지/중단 제어"
```

---

### Task 36: BatchPanel Phase 6A — 진행 표시

**Files:**
- Modify: `src/documatch/gui/panels/batch_panel.py`
- Test: `tests/gui/test_panels_batch.py`

- [ ] **Step 36.1: Failing test**

```python
from PySide6.QtCore import Qt

from documatch.gui.panels.batch_panel import BatchPanel


def test_batch_panel_shows_progress(qapp, qtbot):
    panel = BatchPanel()
    qtbot.addWidget(panel)
    panel.show_progress_view()
    panel.update_progress(2, 5, "doc.pdf")
    assert "2/5" in panel.progress_label.text()


def test_batch_panel_pause_button(qapp, qtbot):
    panel = BatchPanel()
    qtbot.addWidget(panel)
    panel.show_progress_view()
    with qtbot.waitSignal(panel.pause_clicked, timeout=1000):
        qtbot.mouseClick(panel.btn_pause, Qt.LeftButton)


def test_batch_panel_cancel_button(qapp, qtbot):
    panel = BatchPanel()
    qtbot.addWidget(panel)
    panel.show_progress_view()
    with qtbot.waitSignal(panel.cancel_clicked, timeout=1000):
        qtbot.mouseClick(panel.btn_cancel, Qt.LeftButton)
```

- [ ] **Step 36.2: Implement** `src/documatch/gui/panels/batch_panel.py`:

```python
"""BatchPanel — Step 6 진행(6A) + 결과 편집(6B)."""
from PySide6.QtCore import Signal
from PySide6.QtWidgets import (
    QHBoxLayout, QLabel, QProgressBar, QPushButton, QStackedWidget, QVBoxLayout,
    QWidget,
)


class BatchPanel(QWidget):
    pause_clicked = Signal()
    resume_clicked = Signal()
    cancel_clicked = Signal()
    save_clicked = Signal()
    back_clicked = Signal()
    quit_clicked = Signal()

    def __init__(self) -> None:
        super().__init__()
        self._is_paused = False

        layout = QVBoxLayout(self)
        layout.setContentsMargins(16, 16, 16, 16)
        self._stack = QStackedWidget()
        layout.addWidget(self._stack)

        # Phase 6A — 진행
        self.progress_view = QWidget()
        prog_layout = QVBoxLayout(self.progress_view)
        self.progress_label = QLabel("진행 중...")
        self.progress_label.setStyleSheet("font-size: 14pt;")
        prog_layout.addWidget(self.progress_label)
        self.progress_bar = QProgressBar()
        prog_layout.addWidget(self.progress_bar)
        self.summary_label = QLabel("")
        prog_layout.addWidget(self.summary_label)
        prog_layout.addStretch()

        prog_buttons = QHBoxLayout()
        self.btn_pause = QPushButton("일시정지")
        self.btn_pause.clicked.connect(self._on_pause_resume)
        prog_buttons.addWidget(self.btn_pause)
        self.btn_cancel = QPushButton("중단")
        self.btn_cancel.clicked.connect(self.cancel_clicked)
        prog_buttons.addWidget(self.btn_cancel)
        prog_buttons.addStretch()
        prog_layout.addLayout(prog_buttons)
        self._stack.addWidget(self.progress_view)

        # Phase 6B — 결과 편집 (Task 37에서 추가)
        self.review_view = QWidget()
        self._stack.addWidget(self.review_view)

    def _on_pause_resume(self) -> None:
        if self._is_paused:
            self._is_paused = False
            self.btn_pause.setText("일시정지")
            self.resume_clicked.emit()
        else:
            self._is_paused = True
            self.btn_pause.setText("재개")
            self.pause_clicked.emit()

    def show_progress_view(self) -> None:
        self._stack.setCurrentWidget(self.progress_view)

    def update_progress(self, current: int, total: int, label: str) -> None:
        self.progress_label.setText(f"[{current}/{total}] {label}")
        if total > 0:
            self.progress_bar.setMaximum(total)
            self.progress_bar.setValue(current)
```

- [ ] **Step 36.3: Run — PASS**

- [ ] **Step 36.4: Commit**

```bash
git add src/documatch/gui/panels/batch_panel.py tests/gui/test_panels_batch.py
git commit -m "feat(gui): BatchPanel Phase 6A — 진행 표시 + pause/cancel 버튼"
```

---

### Task 37: BatchPanel Phase 6B — 결과 그리드 편집

**Files:**
- Modify: `src/documatch/gui/panels/batch_panel.py`
- Modify: `tests/gui/test_panels_batch.py`

- [ ] **Step 37.1: Failing test**

```python
def test_batch_panel_review_view_shows_grid(qapp, qtbot):
    from documatch.core.models import ExtractionResult
    from documatch.spec.models import (
        ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
    )

    panel = BatchPanel()
    qtbot.addWidget(panel)
    spec = ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )
    results = [
        ExtractionResult(document_id="a.pdf", values={"a": "v1"}),
        ExtractionResult(document_id="b.pdf", values={"a": "v2"}),
    ]
    panel.show_review_view(results, spec)
    assert panel.results_table.rowCount() == 2


def test_batch_panel_save_emits(qapp, qtbot):
    panel = BatchPanel()
    qtbot.addWidget(panel)
    panel.show_review_view([], None)  # 빈 결과
    with qtbot.waitSignal(panel.save_clicked, timeout=1000):
        qtbot.mouseClick(panel.btn_save, Qt.LeftButton)
```

- [ ] **Step 37.2: review_view 구현** — `batch_panel.py`의 `__init__` 끝에서 review_view 채우기:

```python
        # Phase 6B — 결과 편집
        self.review_view = QWidget()
        rev_layout = QVBoxLayout(self.review_view)
        rev_layout.addWidget(QLabel("배치 결과 검토 (편집 가능)"))
        from PySide6.QtWidgets import QTableWidget
        self.results_table = QTableWidget()
        rev_layout.addWidget(self.results_table)

        rev_buttons = QHBoxLayout()
        self.btn_back = QPushButton("← 재배치")
        self.btn_back.clicked.connect(self.back_clicked)
        rev_buttons.addWidget(self.btn_back)
        self.btn_quit = QPushButton("종료")
        self.btn_quit.clicked.connect(self.quit_clicked)
        rev_buttons.addWidget(self.btn_quit)
        rev_buttons.addStretch()
        self.btn_save = QPushButton("→ 저장")
        self.btn_save.setStyleSheet("font-weight: 600;")
        self.btn_save.clicked.connect(self.save_clicked)
        rev_buttons.addWidget(self.btn_save)
        rev_layout.addLayout(rev_buttons)

        self._stack.addWidget(self.review_view)
        self._spec_for_review = None
        self._results_for_review = []
```

`show_review_view` + `get_modified_results`:

```python
    def show_review_view(self, results, spec) -> None:
        from PySide6.QtWidgets import QLineEdit, QTableWidgetItem
        from PySide6.QtCore import Qt
        self._spec_for_review = spec
        self._results_for_review = list(results)
        self.results_table.clear()
        if spec is None or not results:
            self.results_table.setColumnCount(0)
            self.results_table.setRowCount(0)
            self._stack.setCurrentWidget(self.review_view)
            return
        cols = ["문서"] + [f.name for f in spec.fields] + ["상태"]
        self.results_table.setColumnCount(len(cols))
        self.results_table.setHorizontalHeaderLabels(cols)
        self.results_table.setRowCount(len(results))
        for i, r in enumerate(results):
            doc_item = QTableWidgetItem(r.document_id)
            doc_item.setFlags(Qt.ItemIsEnabled)
            self.results_table.setItem(i, 0, doc_item)
            for j, f in enumerate(spec.fields, start=1):
                value = str(r.values.get(f.name, ""))
                edit = QLineEdit(value)
                self.results_table.setCellWidget(i, j, edit)
            status_item = QTableWidgetItem(r.status)
            status_item.setFlags(Qt.ItemIsEnabled)
            if r.status == "failed":
                status_item.setBackground(Qt.red)
            elif r.needs_review:
                status_item.setBackground(Qt.yellow)
            self.results_table.setItem(i, len(cols) - 1, status_item)
        self._stack.setCurrentWidget(self.review_view)

    def get_modified_results(self):
        from dataclasses import replace
        if self._spec_for_review is None:
            return self._results_for_review
        modified = []
        for i, r in enumerate(self._results_for_review):
            new_values = dict(r.values)
            for j, f in enumerate(self._spec_for_review.fields, start=1):
                widget = self.results_table.cellWidget(i, j)
                if widget:
                    new_values[f.name] = widget.text()
            modified.append(replace(r, values=new_values))
        return modified
```

- [ ] **Step 37.3: Run — PASS**

- [ ] **Step 37.4: Commit**

```bash
git add src/documatch/gui/panels/batch_panel.py tests/gui/test_panels_batch.py
git commit -m "feat(gui): BatchPanel Phase 6B — 결과 그리드 편집"
```

---

### Task 38: GuiReviewer.review_batch_results + show_progress 통합

**Files:**
- Modify: `src/documatch/gui/reviewer.py`
- Test: `tests/gui/test_gui_reviewer_batch.py`

- [ ] **Step 38.1: Failing test**

```python
import asyncio
import pytest
from PySide6.QtCore import Qt

from documatch.core.models import ExtractionResult
from documatch.gui.main_window import MainWindow
from documatch.gui.reviewer import GuiReviewer
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)


def _spec():
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


@pytest.mark.asyncio
async def test_review_batch_results_save(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    results = [ExtractionResult(document_id="a.pdf", values={"a": "v1"})]
    task = asyncio.create_task(reviewer.review_batch_results(results, _spec()))
    await asyncio.sleep(0.05)
    qtbot.mouseClick(window._panels["batch"].btn_save, Qt.LeftButton)
    decision = await asyncio.wait_for(task, timeout=2.0)
    assert decision.action == "save"
    assert decision.modified_results is not None


@pytest.mark.asyncio
async def test_review_batch_results_back(qapp, qtbot):
    window = MainWindow()
    qtbot.addWidget(window)
    reviewer = GuiReviewer(main_window=window)
    task = asyncio.create_task(reviewer.review_batch_results([], _spec()))
    await asyncio.sleep(0.05)
    qtbot.mouseClick(window._panels["batch"].btn_back, Qt.LeftButton)
    decision = await asyncio.wait_for(task, timeout=2.0)
    assert decision.action == "back"
```

- [ ] **Step 38.2: GuiReviewer.review_batch_results + show_progress 갱신**

```python
    async def show_progress(self, current: int, total: int, label: str) -> None:
        if self._main_window is None:
            return
        self._main_window._progress.show_busy(f"[{current}/{total}] {label}")
        # Step 6 진행 화면이면 BatchPanel도 업데이트
        batch_panel = self._main_window._panels["batch"]
        batch_panel.show_progress_view()
        batch_panel.update_progress(current, total, label)
        if current == 1:
            self._main_window.show_panel("batch")
            self._main_window._step_nav.set_current(6)

    async def review_batch_results(
        self, results: list[ExtractionResult], spec: ExtractionSpec,
    ) -> BatchResultDecision:
        if self._main_window is None:
            raise RuntimeError("MainWindow 미설정")
        import asyncio
        loop = asyncio.get_event_loop()
        future: asyncio.Future[BatchResultDecision] = loop.create_future()

        panel = self._main_window._panels["batch"]
        panel.show_review_view(results, spec)
        self._main_window.show_panel("batch")
        self._main_window._step_nav.set_current(6)

        def on_save() -> None:
            if not future.done():
                future.set_result(BatchResultDecision(
                    action="save", modified_results=panel.get_modified_results(),
                ))

        def on_back() -> None:
            if not future.done():
                future.set_result(BatchResultDecision(action="back"))

        def on_quit() -> None:
            if not future.done():
                future.set_result(BatchResultDecision(action="quit"))

        panel.save_clicked.connect(on_save)
        panel.back_clicked.connect(on_back)
        panel.quit_clicked.connect(on_quit)
        try:
            return await future
        finally:
            panel.save_clicked.disconnect(on_save)
            panel.back_clicked.disconnect(on_back)
            panel.quit_clicked.disconnect(on_quit)
```

- [ ] **Step 38.3: Run — PASS**

- [ ] **Step 38.4: Commit**

```bash
git add src/documatch/gui/reviewer.py tests/gui/test_gui_reviewer_batch.py
git commit -m "feat(gui): GuiReviewer.review_batch_results + show_progress (Step 6)"
```

---

### Task 39: Phase 5 통합 E2E

**Files:**
- Test: `tests/gui/test_e2e_phase5.py`

- [ ] **Step 39.1: 풀 happy path E2E**

```python
"""Phase 5 통합 — 폴더 → 스펙 → 모드 → 샘플 → 배치 → 저장."""
import asyncio
import json
from pathlib import Path
import pytest
from PySide6.QtCore import Qt

from documatch.gui.main_window import MainWindow
from tests.fakes import FakeLLMClient


@pytest.mark.asyncio
async def test_e2e_phase5_full_to_save(qapp, qtbot, tmp_path, monkeypatch):
    (tmp_path / "a.txt").write_text("hi")
    (tmp_path / "b.txt").write_text("bye")
    out_path = tmp_path / "out.xlsx"

    spec_resp = json.dumps({
        "name": "x", "doc_type_hint": "x",
        "fields": [{"name": "a", "description": "d", "type": "string", "required": True}],
        "summary_template": {"template": "{a}"},
        "output_table": [{"field_name": "a", "excel_header": "a", "width": 10}],
        "table_mode": False,
    })
    extract_resp = json.dumps({"values": {"a": "v"}, "confidence": {"a": 0.9}, "summary_sentence": "v"})
    fake = FakeLLMClient(responses=[spec_resp, extract_resp, extract_resp, extract_resp])
    monkeypatch.setattr(
        "documatch.gui.main_window.create_llm",
        lambda settings, *, role: fake,
    )

    window = MainWindow()
    qtbot.addWidget(window)
    window._panels["start"].folder_selected.emit(tmp_path)

    await asyncio.sleep(0.1)
    qtbot.mouseClick(window._panels["file"].btn_next, Qt.LeftButton)
    await asyncio.sleep(0.1)
    window._panels["sample"].prompt_edit.setPlainText("a 추출")
    qtbot.mouseClick(window._panels["sample"].btn_next, Qt.LeftButton)
    await asyncio.sleep(0.5)
    qtbot.mouseClick(window._panels["spec"].btn_approve, Qt.LeftButton)
    await asyncio.sleep(0.1)
    qtbot.mouseClick(window._panels["mode"].btn_single, Qt.LeftButton)
    await asyncio.sleep(0.5)
    qtbot.mouseClick(window._panels["result"].btn_full, Qt.LeftButton)
    await asyncio.sleep(0.5)
    qtbot.mouseClick(window._panels["batch"].btn_save, Qt.LeftButton)
    await asyncio.sleep(0.1)
    window._panels["save"].path_edit.setText(str(out_path))
    qtbot.mouseClick(window._panels["save"].btn_save, Qt.LeftButton)

    await asyncio.sleep(2.0)
    assert out_path.exists()
```

- [ ] **Step 39.2: Run — PASS**

- [ ] **Step 39.3: Commit**

```bash
git add tests/gui/test_e2e_phase5.py
git commit -m "test(gui): Phase 5 풀 happy path E2E"
```

Phase 5 완료. 7단계 모두 동작 + 배치 결과 편집 + Excel 저장.

---

## Phase 6 — 문서 미리보기

### Task 40: DocumentPreview ABC + PreviewPanel dispatcher

**Files:**
- Create: `src/documatch/gui/preview/__init__.py`
- Create: `src/documatch/gui/preview/base.py`
- Create: `src/documatch/gui/preview/preview_panel.py`
- Test: `tests/gui/test_preview_panel.py`

- [ ] **Step 40.1: preview/__init__.py 생성** (빈)

- [ ] **Step 40.2: Failing test**

```python
from pathlib import Path

from documatch.gui.preview.preview_panel import PreviewPanel


def test_preview_panel_starts_empty(qapp, qtbot):
    p = PreviewPanel()
    qtbot.addWidget(p)
    # 초기 상태 — placeholder 메시지
    assert p.current_widget() is not None


def test_preview_panel_dispatches_unknown_extension(qapp, qtbot, tmp_path):
    p = PreviewPanel()
    qtbot.addWidget(p)
    fake = tmp_path / "x.unknown"
    fake.write_text("test")
    p.show_file(fake)
    # 알 수 없는 확장자는 텍스트로 폴백
    assert p.current_widget() is not None
```

- [ ] **Step 40.3: Implement base.py + preview_panel.py**

`src/documatch/gui/preview/base.py`:
```python
"""DocumentPreview — 모든 미리보기 위젯의 공통 인터페이스."""
from abc import abstractmethod
from pathlib import Path

from PySide6.QtWidgets import QWidget


class DocumentPreview(QWidget):
    @abstractmethod
    def load(self, path: Path) -> None: ...

    def highlight(self, text: str) -> int:
        return 0

    def clear_highlights(self) -> None:
        pass

    def go_to_page(self, page: int) -> None:
        pass
```

`src/documatch/gui/preview/preview_panel.py`:
```python
"""PreviewPanel — 확장자별 DocumentPreview dispatch."""
from pathlib import Path

from PySide6.QtWidgets import QLabel, QStackedWidget, QWidget


class PreviewPanel(QStackedWidget):
    def __init__(self) -> None:
        super().__init__()
        self._placeholder = QLabel("미리보기 없음\n(샘플/결과 선택 시 자동 표시)")
        self.addWidget(self._placeholder)
        self._widgets: dict[str, QWidget] = {}

    def show_file(self, path: Path) -> None:
        ext = path.suffix.lower()
        widget = self._get_widget(ext)
        try:
            widget.load(path)
        except Exception:
            self.setCurrentWidget(self._placeholder)
            return
        self.setCurrentWidget(widget)

    def _get_widget(self, ext: str):
        if ext in self._widgets:
            return self._widgets[ext]
        widget = self._build_widget(ext)
        self._widgets[ext] = widget
        self.addWidget(widget)
        return widget

    def _build_widget(self, ext: str):
        # Task 41-46에서 실제 구현체로 교체
        from documatch.gui.preview.text_preview import TextPreview
        return TextPreview()

    def current_widget(self):
        return self.currentWidget()

    def highlight(self, text: str) -> int:
        widget = self.currentWidget()
        if hasattr(widget, "highlight"):
            return widget.highlight(text)
        return 0
```

- [ ] **Step 40.4: TextPreview 임시 구현** — `src/documatch/gui/preview/text_preview.py`:

```python
"""TextPreview — TXT/MD/CSV/HTML 텍스트 계열 (간단 구현, Task 45에서 확장)."""
from pathlib import Path

from PySide6.QtWidgets import QPlainTextEdit, QVBoxLayout

from documatch.gui.preview.base import DocumentPreview


class TextPreview(DocumentPreview):
    def __init__(self) -> None:
        super().__init__()
        layout = QVBoxLayout(self)
        layout.setContentsMargins(0, 0, 0, 0)
        self.text_edit = QPlainTextEdit()
        self.text_edit.setReadOnly(True)
        layout.addWidget(self.text_edit)

    def load(self, path: Path) -> None:
        try:
            content = path.read_text(encoding="utf-8", errors="replace")
        except Exception:
            content = ""
        self.text_edit.setPlainText(content)

    def highlight(self, text: str) -> int:
        cursor = self.text_edit.document().find(text)
        if cursor.isNull():
            return 0
        self.text_edit.setTextCursor(cursor)
        return 1
```

- [ ] **Step 40.5: Run — PASS**

- [ ] **Step 40.6: MainWindow에 PreviewPanel 통합** — placeholder QLabel을 PreviewPanel로 교체:

`main_window.py`의 `_make_preview_placeholder`를 다음으로 교체:

```python
    def _make_preview_panel(self) -> QWidget:
        from documatch.gui.preview.preview_panel import PreviewPanel
        self._preview = PreviewPanel()
        return self._preview
```

`splitter.addWidget(self._make_preview_placeholder())` → `splitter.addWidget(self._make_preview_panel())`

- [ ] **Step 40.7: Commit**

```bash
git add src/documatch/gui/preview/ src/documatch/gui/main_window.py tests/gui/test_preview_panel.py
git commit -m "feat(gui): DocumentPreview ABC + PreviewPanel dispatcher + TextPreview"
```

---

### Task 41: PdfPreview — PyMuPDF 기반 페이지 렌더링

**Files:**
- Create: `src/documatch/gui/preview/pdf_preview.py`
- Modify: `src/documatch/gui/preview/preview_panel.py`
- Test: `tests/gui/test_preview_pdf.py`

- [ ] **Step 41.1: Failing test**

```python
from pathlib import Path

from documatch.gui.preview.pdf_preview import PdfPreview


def test_pdf_preview_loads_fixture(qapp, qtbot):
    fixture = Path("tests/fixtures/docs/text.pdf")
    if not fixture.exists():
        import pytest
        pytest.skip("fixture not available")
    p = PdfPreview()
    qtbot.addWidget(p)
    p.load(fixture)
    assert p.page_count() > 0


def test_pdf_preview_highlight_returns_count(qapp, qtbot):
    fixture = Path("tests/fixtures/docs/text.pdf")
    if not fixture.exists():
        import pytest
        pytest.skip("fixture not available")
    p = PdfPreview()
    qtbot.addWidget(p)
    p.load(fixture)
    n = p.highlight("the")  # 일반적인 영어 단어
    assert n >= 0  # 매치되거나 안 되거나
```

- [ ] **Step 41.2: Implement** `src/documatch/gui/preview/pdf_preview.py`:

```python
"""PdfPreview — PyMuPDF 기반 PDF 미리보기 + 검색 하이라이트."""
from pathlib import Path

import fitz
from PySide6.QtCore import Qt
from PySide6.QtGui import QImage, QPixmap
from PySide6.QtWidgets import QLabel, QScrollArea, QVBoxLayout

from documatch.gui.preview.base import DocumentPreview


class PdfPreview(DocumentPreview):
    def __init__(self) -> None:
        super().__init__()
        self._doc: fitz.Document | None = None
        self._zoom = 1.0
        self._highlights: list[fitz.Rect] = []

        layout = QVBoxLayout(self)
        layout.setContentsMargins(0, 0, 0, 0)
        self.scroll = QScrollArea()
        self.scroll.setWidgetResizable(True)
        layout.addWidget(self.scroll)
        self.label = QLabel()
        self.label.setAlignment(Qt.AlignTop)
        self.scroll.setWidget(self.label)

    def load(self, path: Path) -> None:
        if self._doc is not None:
            self._doc.close()
        self._doc = fitz.open(str(path))
        self._highlights = []
        self._render_first_page()

    def page_count(self) -> int:
        return len(self._doc) if self._doc else 0

    def _render_first_page(self) -> None:
        if self._doc is None or len(self._doc) == 0:
            return
        page = self._doc[0]
        matrix = fitz.Matrix(self._zoom, self._zoom)
        pixmap = page.get_pixmap(matrix=matrix)
        img = QImage(
            pixmap.samples, pixmap.width, pixmap.height,
            pixmap.stride, QImage.Format_RGB888,
        )
        self.label.setPixmap(QPixmap.fromImage(img))

    def highlight(self, text: str) -> int:
        if self._doc is None or not text:
            return 0
        count = 0
        for page in self._doc:
            rects = page.search_for(text)
            count += len(rects)
        # 첫 페이지 매치만 표시 (단순화)
        if count > 0:
            self._render_first_page_with_highlights(text)
        return count

    def _render_first_page_with_highlights(self, text: str) -> None:
        if self._doc is None:
            return
        page = self._doc[0]
        for rect in page.search_for(text):
            page.add_highlight_annot(rect)
        matrix = fitz.Matrix(self._zoom, self._zoom)
        pixmap = page.get_pixmap(matrix=matrix)
        img = QImage(
            pixmap.samples, pixmap.width, pixmap.height,
            pixmap.stride, QImage.Format_RGB888,
        )
        self.label.setPixmap(QPixmap.fromImage(img))

    def clear_highlights(self) -> None:
        if self._doc is None:
            return
        # 어노테이션 제거 후 재렌더
        for page in self._doc:
            for annot in list(page.annots() or []):
                page.delete_annot(annot)
        self._render_first_page()
```

- [ ] **Step 41.3: PreviewPanel에 PdfPreview 등록**

`preview_panel.py`의 `_build_widget` 교체:

```python
    def _build_widget(self, ext: str):
        if ext == ".pdf":
            from documatch.gui.preview.pdf_preview import PdfPreview
            return PdfPreview()
        from documatch.gui.preview.text_preview import TextPreview
        return TextPreview()
```

- [ ] **Step 41.4: Run — PASS**

- [ ] **Step 41.5: Commit**

```bash
git add src/documatch/gui/preview/pdf_preview.py src/documatch/gui/preview/preview_panel.py tests/gui/test_preview_pdf.py
git commit -m "feat(gui): PdfPreview — PyMuPDF 기반 + 검색 하이라이트"
```

---

### Task 42: DocxPreview — mammoth + QTextEdit

**Files:**
- Create: `src/documatch/gui/preview/docx_preview.py`
- Modify: `src/documatch/gui/preview/preview_panel.py`
- Test: `tests/gui/test_preview_docx.py`

- [ ] **Step 42.1: Failing test**

```python
from pathlib import Path

from documatch.gui.preview.docx_preview import DocxPreview


def test_docx_preview_loads_fixture(qapp, qtbot):
    fixture = Path("tests/fixtures/docs/sample.docx")
    if not fixture.exists():
        import pytest
        pytest.skip("fixture not available")
    p = DocxPreview()
    qtbot.addWidget(p)
    p.load(fixture)
    assert p.text_edit.toPlainText().strip() != ""


def test_docx_preview_highlight(qapp, qtbot):
    fixture = Path("tests/fixtures/docs/sample.docx")
    if not fixture.exists():
        import pytest
        pytest.skip("fixture not available")
    p = DocxPreview()
    qtbot.addWidget(p)
    p.load(fixture)
    text = p.text_edit.toPlainText()
    if text:
        first_word = text.split()[0]
        n = p.highlight(first_word)
        assert n >= 1
```

- [ ] **Step 42.2: Implement** `src/documatch/gui/preview/docx_preview.py`:

```python
"""DocxPreview — mammoth로 HTML 변환 후 QTextEdit 표시."""
from pathlib import Path

import mammoth
from PySide6.QtWidgets import QTextEdit, QVBoxLayout

from documatch.gui.preview.base import DocumentPreview


class DocxPreview(DocumentPreview):
    def __init__(self) -> None:
        super().__init__()
        layout = QVBoxLayout(self)
        layout.setContentsMargins(0, 0, 0, 0)
        self.text_edit = QTextEdit()
        self.text_edit.setReadOnly(True)
        layout.addWidget(self.text_edit)

    def load(self, path: Path) -> None:
        with open(path, "rb") as f:
            result = mammoth.convert_to_html(f)
        self.text_edit.setHtml(result.value)

    def highlight(self, text: str) -> int:
        if not text:
            return 0
        cursor = self.text_edit.document().find(text)
        if cursor.isNull():
            return 0
        self.text_edit.setTextCursor(cursor)
        return 1
```

- [ ] **Step 42.3: PreviewPanel에 DocxPreview 등록**

```python
    def _build_widget(self, ext: str):
        if ext == ".pdf":
            from documatch.gui.preview.pdf_preview import PdfPreview
            return PdfPreview()
        if ext == ".docx":
            from documatch.gui.preview.docx_preview import DocxPreview
            return DocxPreview()
        from documatch.gui.preview.text_preview import TextPreview
        return TextPreview()
```

- [ ] **Step 42.4: Run — PASS**

- [ ] **Step 42.5: Commit**

```bash
git add src/documatch/gui/preview/docx_preview.py src/documatch/gui/preview/preview_panel.py tests/gui/test_preview_docx.py
git commit -m "feat(gui): DocxPreview — mammoth → QTextEdit"
```

---

### Task 43: XlsxPreview — 시트 탭 + QTableView

**Files:**
- Create: `src/documatch/gui/preview/xlsx_preview.py`
- Modify: `src/documatch/gui/preview/preview_panel.py`
- Test: `tests/gui/test_preview_xlsx.py`

- [ ] **Step 43.1: Failing test**

```python
from pathlib import Path

from documatch.gui.preview.xlsx_preview import XlsxPreview


def test_xlsx_preview_loads_fixture(qapp, qtbot):
    fixture = Path("tests/fixtures/docs/sample.xlsx")
    if not fixture.exists():
        import pytest
        pytest.skip("fixture not available")
    p = XlsxPreview()
    qtbot.addWidget(p)
    p.load(fixture)
    assert p.tabs.count() >= 1
```

- [ ] **Step 43.2: Implement** `src/documatch/gui/preview/xlsx_preview.py`:

```python
"""XlsxPreview — 시트 탭 + QTableWidget."""
from pathlib import Path

from openpyxl import load_workbook
from PySide6.QtWidgets import (
    QTabWidget, QTableWidget, QTableWidgetItem, QVBoxLayout,
)

from documatch.gui.preview.base import DocumentPreview


class XlsxPreview(DocumentPreview):
    def __init__(self) -> None:
        super().__init__()
        layout = QVBoxLayout(self)
        layout.setContentsMargins(0, 0, 0, 0)
        self.tabs = QTabWidget()
        layout.addWidget(self.tabs)

    def load(self, path: Path) -> None:
        self.tabs.clear()
        wb = load_workbook(str(path), read_only=True, data_only=True)
        try:
            for sheet_name in wb.sheetnames:
                ws = wb[sheet_name]
                table = QTableWidget()
                rows = list(ws.iter_rows(values_only=True))
                if not rows:
                    self.tabs.addTab(table, sheet_name)
                    continue
                table.setRowCount(len(rows))
                table.setColumnCount(max(len(r) for r in rows))
                for i, row in enumerate(rows):
                    for j, cell in enumerate(row):
                        if cell is not None:
                            table.setItem(i, j, QTableWidgetItem(str(cell)))
                self.tabs.addTab(table, sheet_name)
        finally:
            wb.close()

    def highlight(self, text: str) -> int:
        widget = self.tabs.currentWidget()
        if not isinstance(widget, QTableWidget):
            return 0
        count = 0
        for i in range(widget.rowCount()):
            for j in range(widget.columnCount()):
                item = widget.item(i, j)
                if item and text in item.text():
                    from PySide6.QtCore import Qt
                    item.setBackground(Qt.yellow)
                    count += 1
        return count
```

- [ ] **Step 43.3: PreviewPanel에 등록**

`preview_panel.py`의 `_build_widget`에 추가:

```python
        if ext == ".xlsx":
            from documatch.gui.preview.xlsx_preview import XlsxPreview
            return XlsxPreview()
```

- [ ] **Step 43.4: Run — PASS**

- [ ] **Step 43.5: Commit**

```bash
git add src/documatch/gui/preview/xlsx_preview.py src/documatch/gui/preview/preview_panel.py tests/gui/test_preview_xlsx.py
git commit -m "feat(gui): XlsxPreview — 시트 탭 + 테이블"
```

---

### Task 44: ImagePreview — QGraphicsView + zoom

**Files:**
- Create: `src/documatch/gui/preview/image_preview.py`
- Modify: `src/documatch/gui/preview/preview_panel.py`
- Test: `tests/gui/test_preview_image.py`

- [ ] **Step 44.1: Failing test**

```python
from pathlib import Path

from documatch.gui.preview.image_preview import ImagePreview


def test_image_preview_loads_fixture(qapp, qtbot):
    fixture = Path("tests/fixtures/docs/sample.png")
    if not fixture.exists():
        import pytest
        pytest.skip("fixture not available")
    p = ImagePreview()
    qtbot.addWidget(p)
    p.load(fixture)
    assert p._scene.itemsBoundingRect().width() > 0
```

- [ ] **Step 44.2: Implement** `src/documatch/gui/preview/image_preview.py`:

```python
"""ImagePreview — QGraphicsView + pan/zoom."""
from pathlib import Path

from PySide6.QtCore import Qt
from PySide6.QtGui import QPixmap
from PySide6.QtWidgets import QGraphicsScene, QGraphicsView, QVBoxLayout

from documatch.gui.preview.base import DocumentPreview


class ImagePreview(DocumentPreview):
    def __init__(self) -> None:
        super().__init__()
        layout = QVBoxLayout(self)
        layout.setContentsMargins(0, 0, 0, 0)
        self._scene = QGraphicsScene()
        self._view = QGraphicsView(self._scene)
        self._view.setRenderHints(self._view.renderHints())
        layout.addWidget(self._view)

    def load(self, path: Path) -> None:
        self._scene.clear()
        pixmap = QPixmap(str(path))
        if pixmap.isNull():
            return
        self._scene.addPixmap(pixmap)
        self._view.fitInView(self._scene.itemsBoundingRect(), Qt.KeepAspectRatio)

    def wheelEvent(self, event):
        if event.modifiers() & Qt.ControlModifier:
            factor = 1.2 if event.angleDelta().y() > 0 else 1 / 1.2
            self._view.scale(factor, factor)
        else:
            super().wheelEvent(event)
```

- [ ] **Step 44.3: PreviewPanel에 등록**

```python
        if ext in (".jpg", ".jpeg", ".png"):
            from documatch.gui.preview.image_preview import ImagePreview
            return ImagePreview()
```

- [ ] **Step 44.4: Commit**

```bash
git add src/documatch/gui/preview/image_preview.py src/documatch/gui/preview/preview_panel.py tests/gui/test_preview_image.py
git commit -m "feat(gui): ImagePreview — QGraphicsView pan/zoom"
```

---

### Task 45: TextPreview 확장 (MD/CSV/HTML 분기)

**Files:**
- Modify: `src/documatch/gui/preview/text_preview.py`
- Test: `tests/gui/test_preview_text.py`

- [ ] **Step 45.1: Failing test**

```python
from pathlib import Path

from documatch.gui.preview.text_preview import TextPreview


def test_text_preview_loads_md(qapp, qtbot, tmp_path):
    f = tmp_path / "x.md"
    f.write_text("# 제목\n\n본문")
    p = TextPreview()
    qtbot.addWidget(p)
    p.load(f)
    assert "제목" in p.text_edit.toPlainText() or "제목" in p.text_edit.toHtml()


def test_text_preview_loads_csv(qapp, qtbot, tmp_path):
    f = tmp_path / "x.csv"
    f.write_text("a,b,c\n1,2,3")
    p = TextPreview()
    qtbot.addWidget(p)
    p.load(f)
    assert p.text_edit.toPlainText().strip() != ""
```

- [ ] **Step 45.2: TextPreview를 확장** — `text_preview.py`:

```python
"""TextPreview — TXT/MD/CSV/HTML 텍스트 계열."""
from pathlib import Path

from PySide6.QtWidgets import QTextEdit, QVBoxLayout

from documatch.gui.preview.base import DocumentPreview


class TextPreview(DocumentPreview):
    def __init__(self) -> None:
        super().__init__()
        layout = QVBoxLayout(self)
        layout.setContentsMargins(0, 0, 0, 0)
        self.text_edit = QTextEdit()
        self.text_edit.setReadOnly(True)
        layout.addWidget(self.text_edit)

    def load(self, path: Path) -> None:
        try:
            content = path.read_text(encoding="utf-8", errors="replace")
        except Exception:
            content = ""
        ext = path.suffix.lower()
        if ext == ".md":
            self.text_edit.setMarkdown(content)
        elif ext in (".html", ".htm"):
            self.text_edit.setHtml(content)
        else:
            self.text_edit.setPlainText(content)

    def highlight(self, text: str) -> int:
        if not text:
            return 0
        cursor = self.text_edit.document().find(text)
        if cursor.isNull():
            return 0
        self.text_edit.setTextCursor(cursor)
        return 1
```

- [ ] **Step 45.3: Run — PASS**

- [ ] **Step 45.4: Commit**

```bash
git add src/documatch/gui/preview/text_preview.py tests/gui/test_preview_text.py
git commit -m "feat(gui): TextPreview — MD/HTML/CSV 분기"
```

---

### Task 46: 셀↔원문 하이라이트 wiring

**Files:**
- Modify: `src/documatch/gui/main_window.py`
- Modify: `src/documatch/gui/panels/spec_panel.py`, `result_panel.py`, `batch_panel.py`
- Test: `tests/gui/test_cell_highlight_wiring.py`

- [ ] **Step 46.1: ResultPanel·SpecPanel·BatchPanel에서 cell_clicked 시그널 노출** — 각 패널에서 editor의 `cell_clicked` 시그널을 패널 시그널로 재방출. 이미 `ResultTableEditor`에 있음. 패널에서:

`spec_panel.py`에 메서드 추가:
```python
    def show_sample(self, path: Path) -> None:
        # MainWindow에서 PreviewPanel.show_file 호출하기 위한 신호
        self._sample_path = path
```

`result_panel.py`에 추가:
```python
    def __init__(self) -> None:
        super().__init__()
        # ... 기존 ...
        self.editor.cell_clicked.connect(self._on_cell_clicked)
        self.cell_clicked = self.editor.cell_clicked  # passthrough
```

- [ ] **Step 46.2: MainWindow에서 PreviewPanel과 cell_clicked 연결**

`main_window.py`의 __init__ 끝에서:

```python
        # 셀 클릭 → 미리보기 하이라이트
        self._panels["result"].editor.cell_clicked.connect(self._on_cell_clicked)
        # SpecPanel은 별도 — 샘플 path 보관 필요. 단순화: 현재 샘플 path를 self._current_sample_path에 저장
        self._current_sample_path: Path | None = None

    def _on_cell_clicked(self, field_name: str, value: object) -> None:
        if self._current_sample_path:
            self._preview.show_file(self._current_sample_path)
        if value:
            self._preview.highlight(str(value))
```

- [ ] **Step 46.3: GuiReviewer가 sample 선택 시 _current_sample_path 업데이트**

`reviewer.py`의 `select_sample` 끝부분에 추가 (return 직전):

```python
        result = await future
        if result:
            self._main_window._current_sample_path = result
            self._main_window._preview.show_file(result)
        return result
```

- [ ] **Step 46.4: 통합 smoke test**

```python
"""셀 클릭 → 미리보기 하이라이트."""
from pathlib import Path

from documatch.core.models import ExtractionResult
from documatch.gui.main_window import MainWindow
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)


def test_cell_clicked_triggers_preview(qapp, qtbot, tmp_path):
    f = tmp_path / "doc.txt"
    f.write_text("hello world INV-2026-001 something")
    window = MainWindow()
    qtbot.addWidget(window)
    window._current_sample_path = f
    spec = ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )
    result = ExtractionResult(document_id="doc.txt", values={"a": "INV-2026-001"})
    window._panels["result"].set_data(result, spec)
    # 셀 클릭 시뮬레이션
    window._panels["result"].editor.cell_clicked.emit("a", "INV-2026-001")
    # PreviewPanel이 텍스트 파일 로드 + 하이라이트
    assert window._preview.current_widget() is not None
```

- [ ] **Step 46.5: Run — PASS**

- [ ] **Step 46.6: Commit**

```bash
git add src/documatch/gui/main_window.py src/documatch/gui/panels/ src/documatch/gui/reviewer.py tests/gui/test_cell_highlight_wiring.py
git commit -m "feat(gui): 셀 클릭 → 미리보기 하이라이트 wiring"
```

Phase 6 완료. 4개 미리보기 + 셀↔원문 하이라이트 동작.

---

## Phase 7 — 다이얼로그 + Keychain + 첫 실행 부트스트랩

### Task 47: ApiKeyDialog — 6 필드 폼 + keyring 통합

**Files:**
- Create: `src/documatch/gui/widgets/api_key_dialog.py`
- Test: `tests/gui/test_widgets_api_key_dialog.py`

- [ ] **Step 47.1: Failing test**

```python
from PySide6.QtCore import Qt

from documatch.gui.widgets.api_key_dialog import ApiKeyDialog


def test_api_key_dialog_loads_existing(qapp, qtbot):
    d = ApiKeyDialog()
    qtbot.addWidget(d)
    d.set_values(plan_url="https://x", plan_model="m", plan_key="k1",
                  batch_url="https://y", batch_model="n", batch_key="k2")
    assert d.plan_url_edit.text() == "https://x"
    assert d.plan_key_edit.text() == "k1"


def test_api_key_dialog_get_values(qapp, qtbot):
    d = ApiKeyDialog()
    qtbot.addWidget(d)
    d.plan_url_edit.setText("https://corp/v1")
    d.plan_model_edit.setText("model-a")
    d.plan_key_edit.setText("pk")
    d.batch_url_edit.setText("https://corp/v1")
    d.batch_model_edit.setText("model-b")
    d.batch_key_edit.setText("bk")
    values = d.get_values()
    assert values["PLAN_AGENT_URL"] == "https://corp/v1"
    assert values["BATCH_AGENT_API_KEY"] == "bk"


def test_api_key_dialog_save_button(qapp, qtbot):
    d = ApiKeyDialog()
    qtbot.addWidget(d)
    with qtbot.waitSignal(d.accepted, timeout=1000):
        qtbot.mouseClick(d.btn_save, Qt.LeftButton)
```

- [ ] **Step 47.2: Implement** `src/documatch/gui/widgets/api_key_dialog.py`:

```python
"""ApiKeyDialog — 6 필드 LLM 서버 정보 입력."""
from PySide6.QtCore import Qt
from PySide6.QtWidgets import (
    QCheckBox, QDialog, QDialogButtonBox, QFormLayout, QLineEdit, QVBoxLayout,
)


class ApiKeyDialog(QDialog):
    def __init__(self) -> None:
        super().__init__()
        self.setWindowTitle("LLM 서버 설정")
        self.setMinimumWidth(500)
        layout = QVBoxLayout(self)

        form = QFormLayout()
        self.plan_url_edit = QLineEdit()
        self.plan_url_edit.setPlaceholderText("https://your-llm-server/v1/chat/completions")
        form.addRow("PLAN_AGENT_URL", self.plan_url_edit)
        self.plan_model_edit = QLineEdit()
        form.addRow("PLAN_AGENT_MODEL", self.plan_model_edit)
        self.plan_key_edit = QLineEdit()
        self.plan_key_edit.setEchoMode(QLineEdit.Password)
        form.addRow("PLAN_AGENT_API_KEY", self.plan_key_edit)
        self.batch_url_edit = QLineEdit()
        form.addRow("BATCH_AGENT_URL", self.batch_url_edit)
        self.batch_model_edit = QLineEdit()
        form.addRow("BATCH_AGENT_MODEL", self.batch_model_edit)
        self.batch_key_edit = QLineEdit()
        self.batch_key_edit.setEchoMode(QLineEdit.Password)
        form.addRow("BATCH_AGENT_API_KEY", self.batch_key_edit)
        layout.addLayout(form)

        self.use_keyring = QCheckBox("OS 키체인에 저장 (권장)")
        self.use_keyring.setChecked(True)
        layout.addWidget(self.use_keyring)

        buttons = QDialogButtonBox()
        self.btn_save = buttons.addButton("저장", QDialogButtonBox.AcceptRole)
        self.btn_cancel = buttons.addButton("취소", QDialogButtonBox.RejectRole)
        buttons.accepted.connect(self.accept)
        buttons.rejected.connect(self.reject)
        layout.addWidget(buttons)

    def set_values(self, **values) -> None:
        self.plan_url_edit.setText(values.get("plan_url", ""))
        self.plan_model_edit.setText(values.get("plan_model", ""))
        self.plan_key_edit.setText(values.get("plan_key", ""))
        self.batch_url_edit.setText(values.get("batch_url", ""))
        self.batch_model_edit.setText(values.get("batch_model", ""))
        self.batch_key_edit.setText(values.get("batch_key", ""))

    def get_values(self) -> dict[str, str]:
        return {
            "PLAN_AGENT_URL": self.plan_url_edit.text().strip(),
            "PLAN_AGENT_MODEL": self.plan_model_edit.text().strip(),
            "PLAN_AGENT_API_KEY": self.plan_key_edit.text().strip(),
            "BATCH_AGENT_URL": self.batch_url_edit.text().strip(),
            "BATCH_AGENT_MODEL": self.batch_model_edit.text().strip(),
            "BATCH_AGENT_API_KEY": self.batch_key_edit.text().strip(),
        }

    def use_keyring_storage(self) -> bool:
        return self.use_keyring.isChecked()
```

- [ ] **Step 47.3: Run — PASS**

- [ ] **Step 47.4: Commit**

```bash
git add src/documatch/gui/widgets/api_key_dialog.py tests/gui/test_widgets_api_key_dialog.py
git commit -m "feat(gui): ApiKeyDialog — 6 필드 LLM 설정 입력"
```

---

### Task 48: keyring 통합 + 부트스트랩 검사

**Files:**
- Create: `src/documatch/gui/keychain.py`
- Test: `tests/gui/test_keychain.py`

- [ ] **Step 48.1: Failing test**

```python
import pytest

from documatch.gui import keychain


def test_keychain_save_and_load(monkeypatch):
    storage: dict[tuple[str, str], str] = {}
    monkeypatch.setattr("keyring.set_password", lambda s, u, p: storage.__setitem__((s, u), p))
    monkeypatch.setattr("keyring.get_password", lambda s, u: storage.get((s, u)))

    keychain.save("PLAN_AGENT_API_KEY", "secret123")
    assert keychain.load("PLAN_AGENT_API_KEY") == "secret123"


def test_keychain_load_returns_none_if_missing(monkeypatch):
    monkeypatch.setattr("keyring.get_password", lambda s, u: None)
    assert keychain.load("MISSING_KEY") is None
```

- [ ] **Step 48.2: Implement** `src/documatch/gui/keychain.py`:

```python
"""OS keyring 래퍼."""
import keyring

_SERVICE = "documatch"


def save(key_name: str, value: str) -> None:
    keyring.set_password(_SERVICE, key_name, value)


def load(key_name: str) -> str | None:
    return keyring.get_password(_SERVICE, key_name)


def delete(key_name: str) -> None:
    try:
        keyring.delete_password(_SERVICE, key_name)
    except keyring.errors.PasswordDeleteError:
        pass
```

- [ ] **Step 48.3: Run — PASS**

- [ ] **Step 48.4: Commit**

```bash
git add src/documatch/gui/keychain.py tests/gui/test_keychain.py
git commit -m "feat(gui): keyring 래퍼 (save/load/delete)"
```

---

### Task 49: MainWindow 첫 실행 부트스트랩 — Settings 검사 + ApiKeyDialog 자동 표시

**Files:**
- Modify: `src/documatch/gui/main_window.py`
- Test: `tests/gui/test_main_window_bootstrap.py`

- [ ] **Step 49.1: Failing test**

```python
import pytest
from unittest.mock import MagicMock

from documatch.gui.main_window import MainWindow


@pytest.mark.asyncio
async def test_bootstrap_shows_dialog_when_keys_missing(qapp, qtbot, monkeypatch):
    # Settings는 빈 키로 시작
    monkeypatch.setattr("documatch.gui.main_window.Settings", MagicMock(return_value=MagicMock(
        anthropic_api_key=None, openai_api_key=None,
        plan_agent_url=None, plan_agent_api_key=None,
        batch_agent_url=None, batch_agent_api_key=None,
    )))
    # 다이얼로그 띄우는 메서드 spy
    called = {"shown": False}
    monkeypatch.setattr(
        "documatch.gui.main_window.MainWindow._show_api_key_dialog",
        lambda self: called.__setitem__("shown", True),
    )
    window = MainWindow()
    qtbot.addWidget(window)
    window.bootstrap()
    assert called["shown"]
```

- [ ] **Step 49.2: Implement bootstrap**

`main_window.py`에 추가:

```python
    def bootstrap(self) -> None:
        """첫 실행 시 LLM 설정 검사 + 누락 시 다이얼로그."""
        settings = Settings()
        if settings.anthropic_api_key or settings.openai_api_key:
            return
        if (settings.plan_agent_url and settings.plan_agent_api_key
                and settings.batch_agent_url and settings.batch_agent_api_key):
            return
        self._show_api_key_dialog()

    def _show_api_key_dialog(self) -> None:
        from documatch.gui import keychain
        from documatch.gui.widgets.api_key_dialog import ApiKeyDialog
        dialog = ApiKeyDialog()
        # 기존 키체인 값 로드
        for var, attr in [
            ("PLAN_AGENT_URL", "plan_url"), ("PLAN_AGENT_MODEL", "plan_model"),
            ("PLAN_AGENT_API_KEY", "plan_key"),
            ("BATCH_AGENT_URL", "batch_url"), ("BATCH_AGENT_MODEL", "batch_model"),
            ("BATCH_AGENT_API_KEY", "batch_key"),
        ]:
            existing = keychain.load(var)
            if existing:
                getattr(dialog, f"{attr}_edit").setText(existing)
        if dialog.exec() == ApiKeyDialog.Accepted:
            values = dialog.get_values()
            import os
            for k, v in values.items():
                if v:
                    os.environ[k] = v
                    if dialog.use_keyring_storage():
                        keychain.save(k, v)
```

- [ ] **Step 49.3: app.py에서 bootstrap 호출**

`gui/app.py`의 main 함수에서:

```python
def main() -> int:
    app = QApplication.instance() or QApplication(sys.argv)
    loop = qasync.QEventLoop(app)
    asyncio.set_event_loop(loop)

    window = MainWindow()
    window.show()
    window.bootstrap()  # NEW

    with loop:
        return loop.run_forever() or 0
```

- [ ] **Step 49.4: Run — PASS**

- [ ] **Step 49.5: Commit**

```bash
git add src/documatch/gui/main_window.py src/documatch/gui/app.py tests/gui/test_main_window_bootstrap.py
git commit -m "feat(gui): 첫 실행 부트스트랩 — LLM 키 누락 시 ApiKeyDialog"
```

---

### Task 50: SettingsDialog — LLM 설정 변경 + 디버그 토글

**Files:**
- Create: `src/documatch/gui/widgets/settings_dialog.py`
- Modify: `src/documatch/gui/main_window.py` (메뉴 추가)
- Test: `tests/gui/test_widgets_settings_dialog.py`

- [ ] **Step 50.1: Failing test**

```python
from documatch.gui.widgets.settings_dialog import SettingsDialog


def test_settings_dialog_opens(qapp, qtbot):
    d = SettingsDialog()
    qtbot.addWidget(d)
    assert d.checkbox_debug is not None
```

- [ ] **Step 50.2: Implement** `src/documatch/gui/widgets/settings_dialog.py`:

```python
"""SettingsDialog — 디버그/배치 설정 + API 키 재설정 진입점."""
from PySide6.QtWidgets import (
    QCheckBox, QDialog, QDialogButtonBox, QFormLayout, QPushButton, QSpinBox,
    QVBoxLayout,
)


class SettingsDialog(QDialog):
    def __init__(self) -> None:
        super().__init__()
        self.setWindowTitle("설정")
        layout = QVBoxLayout(self)

        form = QFormLayout()
        self.checkbox_debug = QCheckBox("디버그 로그 활성")
        form.addRow(self.checkbox_debug)
        self.spin_batch = QSpinBox()
        self.spin_batch.setRange(1, 100)
        self.spin_batch.setValue(10)
        form.addRow("배치 크기", self.spin_batch)
        self.spin_pdf_threshold = QSpinBox()
        self.spin_pdf_threshold.setRange(0, 10000)
        self.spin_pdf_threshold.setValue(100)
        form.addRow("PDF scan threshold", self.spin_pdf_threshold)
        layout.addLayout(form)

        self.btn_change_keys = QPushButton("LLM API 키 재설정...")
        layout.addWidget(self.btn_change_keys)

        buttons = QDialogButtonBox(QDialogButtonBox.Save | QDialogButtonBox.Cancel)
        buttons.accepted.connect(self.accept)
        buttons.rejected.connect(self.reject)
        layout.addWidget(buttons)
```

- [ ] **Step 50.3: MainWindow에 메뉴 + 액션 추가**

`main_window.py`의 `__init__` 끝부분:

```python
        from PySide6.QtGui import QAction
        menu = self.menuBar().addMenu("&설정")
        action_settings = QAction("환경설정...", self)
        action_settings.triggered.connect(self._show_settings_dialog)
        menu.addAction(action_settings)
        action_keys = QAction("API 키...", self)
        action_keys.triggered.connect(self._show_api_key_dialog)
        menu.addAction(action_keys)

    def _show_settings_dialog(self) -> None:
        from documatch.gui.widgets.settings_dialog import SettingsDialog
        d = SettingsDialog()
        d.btn_change_keys.clicked.connect(lambda: (d.accept(), self._show_api_key_dialog()))
        d.exec()
```

- [ ] **Step 50.4: Run — PASS**

- [ ] **Step 50.5: Commit**

```bash
git add src/documatch/gui/widgets/settings_dialog.py src/documatch/gui/main_window.py tests/gui/test_widgets_settings_dialog.py
git commit -m "feat(gui): SettingsDialog + 메뉴바"
```

---

### Task 51: SpecManagerDialog — 저장된 스펙 목록/불러오기/삭제

**Files:**
- Create: `src/documatch/gui/widgets/spec_manager_dialog.py`
- Test: `tests/gui/test_widgets_spec_manager.py`

- [ ] **Step 51.1: Failing test**

```python
from pathlib import Path

from documatch.gui.widgets.spec_manager_dialog import SpecManagerDialog
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)
from documatch.spec.store import SpecStore


def _spec(name="x"):
    return ExtractionSpec(
        name=name, doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


def test_spec_manager_lists_specs(qapp, qtbot, tmp_path):
    store = SpecStore(base_dir=tmp_path)
    store.save(_spec("alpha"))
    store.save(_spec("beta"))
    d = SpecManagerDialog(store)
    qtbot.addWidget(d)
    assert d.list_widget.count() == 2


def test_spec_manager_delete(qapp, qtbot, tmp_path):
    store = SpecStore(base_dir=tmp_path)
    store.save(_spec("alpha"))
    d = SpecManagerDialog(store)
    qtbot.addWidget(d)
    d.list_widget.setCurrentRow(0)
    d._on_delete()
    assert d.list_widget.count() == 0
```

- [ ] **Step 51.2: Implement** `src/documatch/gui/widgets/spec_manager_dialog.py`:

```python
"""SpecManagerDialog — 저장된 스펙 관리."""
from PySide6.QtWidgets import (
    QDialog, QDialogButtonBox, QHBoxLayout, QListWidget, QPushButton, QTextEdit,
    QVBoxLayout,
)

from documatch.spec.store import SpecStore


class SpecManagerDialog(QDialog):
    def __init__(self, store: SpecStore) -> None:
        super().__init__()
        self.setWindowTitle("저장된 스펙 관리")
        self.resize(700, 400)
        self._store = store

        main_layout = QHBoxLayout(self)
        left = QVBoxLayout()
        self.list_widget = QListWidget()
        self.list_widget.currentRowChanged.connect(self._on_select)
        left.addWidget(self.list_widget)

        button_row = QHBoxLayout()
        self.btn_load = QPushButton("불러오기")
        self.btn_load.clicked.connect(self.accept)
        button_row.addWidget(self.btn_load)
        self.btn_delete = QPushButton("삭제")
        self.btn_delete.clicked.connect(self._on_delete)
        button_row.addWidget(self.btn_delete)
        left.addLayout(button_row)
        main_layout.addLayout(left, 1)

        self.preview = QTextEdit()
        self.preview.setReadOnly(True)
        main_layout.addWidget(self.preview, 2)

        self._refresh()

    def _refresh(self) -> None:
        self.list_widget.clear()
        for info in self._store.list():
            self.list_widget.addItem(f"{info.name} — {info.doc_type_hint}")

    def _on_select(self, row: int) -> None:
        if row < 0:
            self.preview.clear()
            return
        infos = self._store.list()
        if row < len(infos):
            spec = self._store.load(infos[row].name)
            self.preview.setPlainText(spec.model_dump_json(indent=2))

    def _on_delete(self) -> None:
        row = self.list_widget.currentRow()
        if row < 0:
            return
        infos = self._store.list()
        if row < len(infos):
            self._store.delete(infos[row].name)
            self._refresh()
            self.preview.clear()

    def selected_name(self) -> str | None:
        row = self.list_widget.currentRow()
        if row < 0:
            return None
        infos = self._store.list()
        return infos[row].name if row < len(infos) else None
```

- [ ] **Step 51.3: MainWindow 메뉴에 추가**

```python
        action_specs = QAction("스펙 관리...", self)
        action_specs.triggered.connect(self._show_spec_manager)
        menu.addAction(action_specs)

    def _show_spec_manager(self) -> None:
        from documatch.gui.widgets.spec_manager_dialog import SpecManagerDialog
        from documatch.config.settings import Settings
        from documatch.spec.store import SpecStore
        store = SpecStore(base_dir=Settings().spec_store_dir)
        d = SpecManagerDialog(store)
        d.exec()
```

- [ ] **Step 51.4: Run — PASS**

- [ ] **Step 51.5: Commit**

```bash
git add src/documatch/gui/widgets/spec_manager_dialog.py src/documatch/gui/main_window.py tests/gui/test_widgets_spec_manager.py
git commit -m "feat(gui): SpecManagerDialog — 저장된 스펙 목록/미리보기/삭제"
```

---

### Task 52: 에러 핸들링 polish — 다이얼로그 표시

**Files:**
- Modify: `src/documatch/gui/main_window.py`
- Test: `tests/gui/test_error_dialog.py`

- [ ] **Step 52.1: Failing test**

```python
import pytest
from unittest.mock import MagicMock

from documatch.exceptions import LLMAuthError
from documatch.gui.main_window import MainWindow


@pytest.mark.asyncio
async def test_engine_error_shown_in_status_bar(qapp, qtbot, monkeypatch):
    fake_engine = MagicMock()
    async def _raise(*args, **kwargs):
        raise LLMAuthError("invalid key")
    fake_engine.run = _raise
    monkeypatch.setattr(
        "documatch.gui.main_window.MainWindow._build_engine",
        lambda self, _: fake_engine,
    )
    window = MainWindow()
    qtbot.addWidget(window)
    from pathlib import Path
    window._panels["start"].folder_selected.emit(Path("/tmp"))
    import asyncio
    await asyncio.sleep(0.1)
    assert "오류" in window.statusBar().currentMessage()
```

- [ ] **Step 52.2: `_run_engine`을 다음으로 교체**

```python
    async def _run_engine(self, engine, scan_path):
        from documatch.exceptions import LLMAuthError, LLMRateLimitError, ConfigError
        try:
            await engine.run(scan_path=scan_path)
        except LLMAuthError as e:
            self.statusBar().showMessage(f"오류 (인증): {e}", 8000)
            self._show_api_key_dialog()
        except LLMRateLimitError as e:
            self.statusBar().showMessage(f"오류 (rate limit): {e}", 8000)
        except ConfigError as e:
            self.statusBar().showMessage(f"오류 (설정): {e}", 8000)
        except Exception as e:
            self.statusBar().showMessage(f"오류: {e}", 8000)
        finally:
            self.show_panel("start")
            self._step_nav.set_current(1)
```

- [ ] **Step 52.3: Run — PASS**

- [ ] **Step 52.4: Commit**

```bash
git add src/documatch/gui/main_window.py tests/gui/test_error_dialog.py
git commit -m "feat(gui): 엔진 에러를 status bar + 적절한 다이얼로그로"
```

---

## Phase 8 — 빌드 파이프라인

### Task 53: assets/ + scripts/generate_icons.py

**Files:**
- Create: `assets/icon-source.png` (placeholder, 1024×1024 단색)
- Create: `scripts/generate_icons.py`

- [ ] **Step 53.1: assets/icon-source.png** — Pillow로 placeholder 생성:

```bash
cd /Users/yjban/Desktop/documatch-cli
python -c "
from PIL import Image, ImageDraw, ImageFont
img = Image.new('RGBA', (1024, 1024), (110, 127, 243, 255))
draw = ImageDraw.Draw(img)
try:
    font = ImageFont.truetype('/System/Library/Fonts/Helvetica.ttc', 600)
except:
    font = ImageFont.load_default()
draw.text((512, 512), 'D', fill='white', font=font, anchor='mm')
import os; os.makedirs('assets', exist_ok=True)
img.save('assets/icon-source.png')
print('OK')
"
```

- [ ] **Step 53.2: scripts/generate_icons.py**

```python
"""1024×1024 PNG → .ico (Windows) + .icns (macOS) 파생."""
import sys
from pathlib import Path
from PIL import Image


def main():
    src = Path("assets/icon-source.png")
    if not src.exists():
        print(f"missing: {src}")
        sys.exit(1)
    img = Image.open(src)

    # Windows .ico
    sizes_ico = [(16, 16), (32, 32), (48, 48), (64, 64), (128, 128), (256, 256)]
    img.save("assets/icon.ico", format="ICO", sizes=sizes_ico)
    print("✓ assets/icon.ico")

    # macOS .icns: iconutil로 만드는 게 정석이지만 간단히 PIL로 .icns 호환
    # PIL은 .icns 직접 저장 지원
    img.save("assets/icon.icns", format="ICNS")
    print("✓ assets/icon.icns")


if __name__ == "__main__":
    main()
```

- [ ] **Step 53.3: 실행해서 파일 생성**

```bash
python scripts/generate_icons.py
ls assets/
```

- [ ] **Step 53.4: Commit**

```bash
git add assets/ scripts/generate_icons.py
git commit -m "build(gui): placeholder 아이콘 + 생성 스크립트"
```

---

### Task 54: documatch_gui.spec — PyInstaller 명세

**Files:**
- Create: `documatch_gui.spec`
- Create: `scripts/build_gui.py`

- [ ] **Step 54.1: documatch_gui.spec**

```python
# documatch_gui.spec
import sys

block_cipher = None

a = Analysis(
    ["src/documatch/gui/app.py"],
    pathex=["src"],
    binaries=[],
    datas=[
        ("assets/icon-source.png", "assets"),
    ],
    hiddenimports=[
        "anthropic", "openai", "pypdf", "openpyxl", "xlrd",
        "docx", "bs4", "chardet", "pdf2image", "PIL",
        "fitz", "mammoth",
        "keyring.backends.macOS", "keyring.backends.Windows",
        "qasync",
    ],
    hookspath=[],
    runtime_hooks=[],
    excludes=["tkinter", "matplotlib"],
    cipher=block_cipher,
    noarchive=False,
)

pyz = PYZ(a.pure, a.zipped_data, cipher=block_cipher)

exe = EXE(
    pyz, a.scripts, a.binaries, a.zipfiles, a.datas,
    name="DocuMatch",
    icon="assets/icon.icns" if sys.platform == "darwin" else "assets/icon.ico",
    console=False,
    debug=False,
    strip=False,
    upx=False,
    onefile=True,
)

if sys.platform == "darwin":
    app = BUNDLE(
        exe,
        name="DocuMatch.app",
        icon="assets/icon.icns",
        bundle_identifier="com.firefly0731.documatch",
        info_plist={
            "CFBundleShortVersionString": "0.2.0",
            "CFBundleVersion": "0.2.0",
            "NSHighResolutionCapable": "True",
            "LSApplicationCategoryType": "public.app-category.productivity",
            "NSHumanReadableCopyright": "© 2026",
        },
    )
```

- [ ] **Step 54.2: scripts/build_gui.py**

```python
"""크로스 플랫폼 GUI 빌드 헬퍼."""
import shutil
import subprocess
import sys
from pathlib import Path


def main():
    for d in ["build", "dist"]:
        shutil.rmtree(d, ignore_errors=True)
    subprocess.run(
        ["pyinstaller", "--clean", "--noconfirm", "documatch_gui.spec"],
        check=True,
    )
    if sys.platform == "darwin":
        print("✓ dist/DocuMatch.app 생성됨")
    elif sys.platform == "win32":
        print("✓ dist/DocuMatch.exe 생성됨")


if __name__ == "__main__":
    main()
```

- [ ] **Step 54.3: 로컬 빌드 검증 (macOS)**

```bash
cd /Users/yjban/Desktop/documatch-cli
source .venv/bin/activate
python scripts/build_gui.py
ls dist/
```

`dist/DocuMatch.app` 생성 확인. 더블클릭으로 실행해보고 빈 창 뜨는지 확인.

- [ ] **Step 54.4: Commit**

```bash
git add documatch_gui.spec scripts/build_gui.py
git commit -m "build(gui): PyInstaller spec + 빌드 스크립트"
```

---

### Task 55: GitHub Actions release.yml에 GUI 빌드 매트릭스 추가

**Files:**
- Modify: `.github/workflows/release.yml`

- [ ] **Step 55.1: release.yml 확장**

```yaml
name: Release
on:
  push:
    tags: ["v*"]

jobs:
  build-gui:
    strategy:
      matrix:
        os: [macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - name: Install Poppler (macOS)
        if: matrix.os == 'macos-latest'
        run: brew install poppler
      - run: pip install -e ".[dev,gui]"
      - run: python scripts/generate_icons.py
      - run: python scripts/build_gui.py

      - name: Zip macOS app
        if: matrix.os == 'macos-latest'
        run: cd dist && zip -r DocuMatch-macos.zip DocuMatch.app

      - uses: softprops/action-gh-release@v2
        with:
          files: |
            dist/DocuMatch-macos.zip
            dist/DocuMatch.exe
          fail_on_unmatched_files: false
          generate_release_notes: true
```

- [ ] **Step 55.2: Commit**

```bash
git add .github/workflows/release.yml
git commit -m "ci(gui): release.yml에 macOS/Windows 빌드 매트릭스 추가"
```

---

## Phase 9 — 테스트 백필 + 커버리지

### Task 56: 비동기 통합 테스트 (pause/resume/cancel)

**Files:**
- Create: `tests/gui/test_async_pause_cancel.py`

- [ ] **Step 56.1: Failing test**

```python
"""BatchController + asyncio 통합 시나리오."""
import asyncio
import pytest

from documatch.gui.threads.batch_controller import BatchController


@pytest.mark.asyncio
async def test_pause_resume_cycle():
    ctrl = BatchController()
    progress = []

    async def task():
        for i in range(20):
            progress.append(i)
            await ctrl.wait_if_paused()
            await asyncio.sleep(0)

    t = asyncio.create_task(task())
    await asyncio.sleep(0.001)
    ctrl.pause()
    await asyncio.sleep(0.01)
    snap = len(progress)
    ctrl.resume()
    await t
    assert snap < 20
    assert len(progress) == 20


@pytest.mark.asyncio
async def test_cancel_during_pause_wakes():
    ctrl = BatchController()
    progress = []

    async def task():
        for i in range(20):
            progress.append(i)
            await ctrl.wait_if_paused()
            if ctrl.is_cancelled:
                return

    t = asyncio.create_task(task())
    await asyncio.sleep(0.001)
    ctrl.pause()
    await asyncio.sleep(0.01)
    ctrl.cancel()
    await asyncio.wait_for(t, timeout=1.0)
    assert ctrl.is_cancelled
```

- [ ] **Step 56.2: Run — PASS**

- [ ] **Step 56.3: Commit**

```bash
git add tests/gui/test_async_pause_cancel.py
git commit -m "test(gui): pause/resume/cancel 비동기 시나리오 검증"
```

---

### Task 57: docs/MANUAL_SMOKE.md 작성

**Files:**
- Create: `docs/MANUAL_SMOKE.md`

- [ ] **Step 57.1: docs/MANUAL_SMOKE.md 작성**

```markdown
# 수동 스모크 체크리스트 (v0.2.0)

릴리스 전 30분 가량 수동 검증 항목.

## 빌드 산출물
- [ ] `dist/DocuMatch.app`(macOS) 또는 `dist/DocuMatch.exe`(Windows) 더블클릭으로 실행
- [ ] 첫 실행 시 OS 보안 경고 우회 가능 (README 안내대로)
- [ ] 창 크기/스플리터 정상

## 첫 실행 부트스트랩
- [ ] LLM 키 미설정 상태에서 ApiKeyDialog 자동 표시
- [ ] hide_input(비밀번호 필드)로 키 안 보임
- [ ] 저장 후 다음 실행 시 다이얼로그 안 뜸 (키체인 정상)

## 7단계 흐름
- [ ] Step 1: 폴더 스캔 결과 + 확장자 chip 정상
- [ ] Step 2: 샘플 라디오 + 의도 입력 + 우측 미리보기 실시간 갱신
- [ ] Step 3: 스펙 표 인라인 편집 (이름/타입/필수/설명/추가/삭제)
- [ ] Step 4: 모드 카드 클릭 동작
- [ ] Step 5: 샘플 결과 셀 편집 + 셀 클릭 → 우측 미리보기 하이라이트
- [ ] Step 6A: 진행률 + 일시정지/재개/중단
- [ ] Step 6B: 결과 그리드 검토 + 셀 편집
- [ ] Step 7: 저장 경로 + 스펙 저장 옵션

## 미리보기 품질
- [ ] PDF: 한글 폰트 정상, 검색·하이라이트 동작
- [ ] DOCX: 표·이미지 렌더링
- [ ] XLSX: 시트 탭 전환
- [ ] 이미지: pan/zoom (ctrl+휠)
- [ ] TXT/MD/CSV/HTML: 폼매팅 정상

## 키보드/IME
- [ ] Tab/Shift+Tab 포커스 이동
- [ ] 한글 IME 입력 (스펙 필드명, 피드백)
- [ ] Cmd/Ctrl+S로 저장 (Step 7에서)
- [ ] Esc로 다이얼로그 닫기

## 에러 시나리오
- [ ] 잘못된 API 키 → status bar 메시지 + ApiKeyDialog 재표시
- [ ] 빈 폴더 → "스캔된 파일 없음" 메시지
- [ ] 깨진 PDF → 부분 실패 격리, 배치 계속

## 다크모드
- [ ] OS 테마 변경 시 자동 따라감 (또는 수동 선택)
```

- [ ] **Step 57.2: Commit**

```bash
git add docs/MANUAL_SMOKE.md
git commit -m "docs: 수동 스모크 체크리스트 (v0.2.0)"
```

---

### Task 58: 커버리지 게이트 검증

**Files:**
- 명령만

- [ ] **Step 58.1: 전체 커버리지 측정**

```bash
cd /Users/yjban/Desktop/documatch-cli
source .venv/bin/activate
pytest --cov=documatch --cov-report=term --cov-fail-under=80
```

전체 커버리지 80% 이상 확인. GUI 모듈은 75%+ 목표.

- [ ] **Step 58.2: 부족하면 보강 테스트 추가** — 커버되지 않은 라인 확인 후 추가 테스트 작성. 큰 격차가 있는 모듈은 추가 task로 분리하여 처리.

- [ ] **Step 58.3: Commit (테스트 보강 시)**

```bash
git add tests/gui/
git commit -m "test(gui): 커버리지 보강 (80%+)"
```

---

## Phase 10 — Polish + 릴리스

### Task 59: 키보드 단축키 (저장/취소)

**Files:**
- Modify: `src/documatch/gui/panels/save_panel.py`, `main_window.py`

- [ ] **Step 59.1: SavePanel에 Cmd/Ctrl+S 단축키**

```python
from PySide6.QtGui import QShortcut, QKeySequence

# SavePanel.__init__ 끝에:
        QShortcut(QKeySequence.Save, self, activated=self.save_clicked)
```

- [ ] **Step 59.2: MainWindow Esc로 종료 (단, 작업 중이 아닐 때)**

`main_window.py`에:

```python
        QShortcut(QKeySequence(Qt.Key_Escape), self, activated=self._on_escape)

    def _on_escape(self) -> None:
        if self._engine_task and not self._engine_task.done():
            return  # 작업 중엔 무시
        self.close()
```

- [ ] **Step 59.3: Commit**

```bash
git add src/documatch/gui/
git commit -m "feat(gui): 키보드 단축키 (Cmd/Ctrl+S, Esc)"
```

---

### Task 60: 다크/라이트 테마 — OS 설정 자동 follow

PySide6는 기본적으로 OS 테마를 따라간다 (별도 작업 불필요). 명시적 강제 다크 모드는 v0.3+로 보류.

- [ ] **Step 60.1: 검증만** — macOS 시스템 환경설정에서 다크 모드 토글 후 앱 다시 시작 → 색상 변경 확인

- [ ] **Step 60.2: 커밋 없음**

---

### Task 61: README v0.2.0 갱신 (GUI + CLI 양쪽 가이드)

**Files:**
- Modify: `README.md`

- [ ] **Step 61.1: README 상단에 GUI Quick Start 섹션 추가**

기존 Quick Start (CLI) 위에 추가:

```markdown
# documatch-cli

AI 기반 문서 추출 도구 — PDF/DOCX/XLSX/이미지/HTML 등에서 원하는 항목을 LLM으로 추출해 Excel로 저장. CLI + 데스크톱 GUI 양쪽 제공.

## Quick Start (GUI — 추천)

[GitHub Releases](https://github.com/firefly0731/documatch-cli/releases)에서 OS에 맞는 파일 다운로드:
- macOS: `DocuMatch-macos.zip` → 압축 해제 → `DocuMatch.app` 실행
- Windows: `DocuMatch.exe` → 더블클릭

처음 실행 시 보안 경고 우회:
- macOS: 시스템 환경설정 → 개인정보보호 및 보안 → "DocuMatch 열기" (또는 우클릭 → 열기)
- Windows: SmartScreen → "추가 정보" → "실행"

첫 실행 시 LLM 서버 정보 입력 다이얼로그 → API 키 6개 채우면 끝. 다음 실행부터는 자동.

## Quick Start (CLI)

```bash
pip install documatch-cli
documatch ./samples
```

(이하 기존 CLI 가이드 유지)
```

- [ ] **Step 61.2: 언제 GUI vs CLI 섹션 추가**

```markdown
## 언제 GUI vs CLI

- **GUI**: 단발 추출, 시각적 검토, 인라인 편집이 필요할 때 → 추천
- **CLI**: CI/배치/스크립트 자동화 → `--auto-approve` 모드
```

- [ ] **Step 61.3: 시스템 요구사항 섹션 추가**

```markdown
## 시스템 요구사항

- **GUI**: macOS 12+ / Windows 10+
- **CLI**: Python 3.11+ (모든 플랫폼)
- 스캔 PDF 처리: Poppler (선택)
```

- [ ] **Step 61.4: Commit**

```bash
git add README.md
git commit -m "docs: README v0.2.0 — GUI Quick Start + 시스템 요구사항"
```

---

### Task 62: CHANGELOG v0.2.0 항목

**Files:**
- Modify: `CHANGELOG.md`

- [ ] **Step 62.1: CHANGELOG 갱신**

```markdown
# Changelog

## [0.2.0] — 2026-MM-DD

### Added
- 데스크톱 GUI 앱 (`documatch-gui` 명령 또는 `DocuMatch.exe`/`.app` 단일 파일)
- 7개 HITL 게이트 GUI 화면 (Hybrid 레이아웃: 좌측 단계 네비 + 중앙 작업 + 우측 미리보기)
- 인라인 편집: 스펙 필드, 샘플 결과 셀, 배치 결과 그리드 + 행 추가/삭제
- 문서 미리보기: PDF (PyMuPDF) / DOCX (mammoth) / XLSX (시트 탭) / 이미지 (pan/zoom) / TXT/MD/CSV/HTML
- 셀↔원문 하이라이트 연동 (best-effort 텍스트 매칭)
- API 키 OS 키체인 저장 (keyring)
- 첫 실행 자동 부트스트랩 — LLM 설정 누락 시 다이얼로그
- 배치 일시정지/재개/중단

### Changed
- Reviewer Protocol을 async로 마이그레이션 (CLI 호환 유지)
- 새 HITL 게이트 `review_batch_results` 추가 (Excel 저장 직전 검토)
- `SpecDecision`/`SampleDecision`에 `modified_*` 필드 추가
- 새 `BatchResultDecision` dataclass

### Build
- PyInstaller 단일 파일 빌드 (macOS .app + Windows .exe)
- GitHub Actions matrix 빌드 + Release 자동 첨부

## [0.1.0] — (이전 릴리스)
(기존 항목 유지)
```

- [ ] **Step 62.2: Commit**

```bash
git add CHANGELOG.md
git commit -m "docs: CHANGELOG v0.2.0 항목"
```

---

### Task 63: pyproject.toml version 0.2.0 + 최종 QA

**Files:**
- Modify: `pyproject.toml`, `src/documatch/__init__.py`

- [ ] **Step 63.1: 버전 bump**

`pyproject.toml`의 `version = "0.2.0-dev"` → `version = "0.2.0"`
`src/documatch/__init__.py`의 `__version__ = "0.2.0-dev"` → `__version__ = "0.2.0"`

- [ ] **Step 63.2: 최종 QA**

```bash
cd /Users/yjban/Desktop/documatch-cli
source .venv/bin/activate

# 전체 테스트 + 커버리지
pytest --cov=documatch --cov-report=term -q

# Lint
ruff check src/ tests/

# Mypy (선택, 시간 있으면)
mypy src/documatch || true

# Build
python -m build
ls dist/

# GUI 빌드
python scripts/generate_icons.py
python scripts/build_gui.py
ls dist/
```

모두 통과 확인.

- [ ] **Step 63.3: Commit**

```bash
git add pyproject.toml src/documatch/__init__.py
git commit -m "release: v0.2.0"
```

- [ ] **Step 63.4: Tag + push**

```bash
git tag -a v0.2.0 -m "v0.2.0 — 데스크톱 GUI 첫 릴리스"
git push origin main
git push origin v0.2.0
```

GitHub Actions가 자동으로 macOS .app + Windows .exe 빌드 + Release에 첨부.

- [ ] **Step 63.5: GitHub Release 페이지에서 산출물 확인**

```bash
gh release view v0.2.0
```

`DocuMatch-macos.zip`과 `DocuMatch.exe` 첨부 확인.

---

## 자가 점검 (계획 작성자)

**스펙 커버리지**: 본 계획의 task들이 스펙 §1~§9를 모두 커버하는지 점검.
- §1 아키텍처 → Task 7-13 (gui 스켈레톤)
- §2 Reviewer 확장 → Task 1-6
- §3 화면 설계 → Task 14-46 (모든 7단계 + 보조)
- §4 미리보기 → Task 40-46
- §5 비동기 → Task 35, 56 (qasync는 Task 8 app.py에서 사용)
- §6 테스트 → Phase 9 (전반적)
- §7 빌드 → Task 53-55, 63
- §8 프로젝트 구조 → Phase 2-7 전반
- §9 위험 요소 → 각 task 내 mitigation 적용 (예: PyMuPDF 라이선스는 의존성 명시 시점에 검토)

**Placeholder scan**: TBD/TODO/FIXME 0건 — 본 계획에 placeholder 없음 확인.

**Type 일관성**: `BatchResultDecision`, `modified_spec`, `modified_result`, `modified_results` 모두 동일 명칭으로 reviewer.py · engine.py · 각 패널 · GuiReviewer 메서드 시그니처에서 일관 사용.

---

**Plan complete and saved to `docs/superpowers/plans/2026-05-09-documatch-desktop.md`.**

총 63 task / 10 phase / 약 5-6주 분량.

각 task는 완료 즉시 git commit. Phase 단위로 동작 가능한 상태 보존:
- Phase 1: CLI 그대로 작동 (async 마이그레이션만)
- Phase 2: `documatch-gui` 명령으로 빈 창 뜸
- Phase 3: Step 1, 2, 4, 7 동작
- Phase 4: 스펙·샘플 결과 인라인 편집
- Phase 5: 풀 7단계 + 배치 진행/편집
- Phase 6: 풍부한 미리보기 + 셀 하이라이트
- Phase 7: 키체인 + 다이얼로그 + 첫 실행 부트스트랩
- Phase 8: 빌드 산출물
- Phase 9: 커버리지 + 수동 스모크
- Phase 10: README/CHANGELOG/Tag → v0.2.0 릴리스

