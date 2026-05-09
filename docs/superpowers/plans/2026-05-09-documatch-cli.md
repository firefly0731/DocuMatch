# documatch-cli Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a product-grade Python CLI library `documatch-cli` (PyPI) that extracts user-specified fields from documents (PDF/DOCX/XLSX/XLS/이미지/TXT/MD/CSV/HTML) using Anthropic or OpenAI LLMs, with 6 human-in-the-loop checkpoints, exporting results to Excel.

**Architecture:** Layered package — `core/` engine + `interaction/` HITL + `llm/` provider abstraction + `processors/` per-format readers + `spec/` extraction schema models + `extractors/` (single/batch) + `exporters/` (Excel) + `cli/` (Click). Pure-Python, all formats core deps, FakeLLMClient for deterministic tests.

**Tech Stack:** Python 3.11+, Click, Rich, pydantic v2, pydantic-settings, anthropic, openai, pypdf, pdf2image (Poppler), python-docx, openpyxl, xlrd, Pillow, beautifulsoup4, chardet, pytest + pytest-asyncio, hatchling, ruff, mypy. Reference spec: `docs/superpowers/specs/2026-05-09-documatch-cli-design.md`.

**Working directory convention:** All commands assume CWD = `/Users/yjban/Desktop/documatch-cli/` (new sibling repo to DocuMatch). Adjust if the user chose a different path.

---

## Phase 1 — Project Scaffolding (Tasks 1-4)

### Task 1: Initialize repository skeleton

**Files:**
- Create: `/Users/yjban/Desktop/documatch-cli/pyproject.toml`
- Create: `/Users/yjban/Desktop/documatch-cli/README.md`
- Create: `/Users/yjban/Desktop/documatch-cli/LICENSE`
- Create: `/Users/yjban/Desktop/documatch-cli/.gitignore`
- Create: `/Users/yjban/Desktop/documatch-cli/.env.example`
- Create: `/Users/yjban/Desktop/documatch-cli/CHANGELOG.md`
- Create: `/Users/yjban/Desktop/documatch-cli/src/documatch/__init__.py`
- Create: `/Users/yjban/Desktop/documatch-cli/tests/__init__.py`

- [ ] **Step 1.1: Create directory and init git**

```bash
mkdir -p /Users/yjban/Desktop/documatch-cli && cd /Users/yjban/Desktop/documatch-cli
git init -b main
mkdir -p src/documatch tests/{unit,integration,fixtures/docs,fixtures/specs}
```

- [ ] **Step 1.2: Write `pyproject.toml`**

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "documatch-cli"
version = "0.1.0"
description = "AI 기반 문서 추출 CLI 툴 — PDF/DOCX/XLSX/이미지/HTML 등에서 원하는 항목을 Excel로 추출"
readme = "README.md"
license = { text = "MIT" }
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

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-cov>=4.1",
    "ruff>=0.4",
    "mypy>=1.10",
    "build>=1.2",
]

[project.scripts]
documatch = "documatch.cli.app:main"

[project.urls]
Homepage = "https://github.com/<owner>/documatch-cli"
Issues = "https://github.com/<owner>/documatch-cli/issues"

[tool.hatch.build.targets.wheel]
packages = ["src/documatch"]

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "W", "B", "UP"]
ignore = ["E501"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
addopts = "-ra"

[tool.mypy]
python_version = "3.11"
strict = true
ignore_missing_imports = true
```

- [ ] **Step 1.3: Write `.gitignore`**

```
__pycache__/
*.py[cod]
*$py.class
.venv/
venv/
.env
.pytest_cache/
.mypy_cache/
.ruff_cache/
.coverage
htmlcov/
dist/
build/
*.egg-info/
.idea/
.vscode/
.DS_Store
```

- [ ] **Step 1.4: Write `.env.example`**

```
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
DOCUMATCH_DEBUG=false
```

- [ ] **Step 1.5: Write `LICENSE` (MIT)**

Standard MIT text with copyright `2026 documatch-cli authors`.

- [ ] **Step 1.6: Write `README.md` (placeholder)**

```markdown
# documatch-cli

AI 기반 문서 추출 CLI — PDF/DOCX/XLSX/이미지/HTML에서 원하는 항목을 Excel로.

문서는 v0.1.0 릴리즈 전에 채워집니다 (Task 39).
```

- [ ] **Step 1.7: Write `CHANGELOG.md` (Keep a Changelog)**

```markdown
# Changelog

## [Unreleased]
### Added
- 프로젝트 스캐폴딩
```

- [ ] **Step 1.8: Write `src/documatch/__init__.py`**

```python
"""documatch — AI 기반 문서 추출 CLI 라이브러리."""

__version__ = "0.1.0"
```

- [ ] **Step 1.9: Write `tests/__init__.py` (empty file)**

```python
```

- [ ] **Step 1.10: Verify install + smoke**

```bash
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
python -c "import documatch; print(documatch.__version__)"
```
Expected output: `0.1.0`

- [ ] **Step 1.11: Commit**

```bash
git add .
git commit -m "chore: 프로젝트 스캐폴딩 (pyproject, license, gitignore)"
```

---

### Task 2: Exception hierarchy

**Files:**
- Create: `src/documatch/exceptions.py`
- Create: `tests/unit/test_exceptions.py`

- [ ] **Step 2.1: Write failing test `tests/unit/test_exceptions.py`**

```python
import pytest

from documatch.exceptions import (
    ConfigError,
    DependencyError,
    DocuMatchError,
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


def test_all_inherit_from_documatch_error():
    for cls in [
        ConfigError, DependencyError, UnsupportedFormatError, ProcessorError,
        LLMError, SpecError, ExtractionError, UserAbort,
    ]:
        assert issubclass(cls, DocuMatchError)


def test_llm_subclasses():
    assert issubclass(LLMRateLimitError, LLMError)
    assert issubclass(LLMAuthError, LLMError)


def test_spec_not_found_is_spec_error():
    assert issubclass(SpecNotFoundError, SpecError)


def test_raise_and_catch():
    with pytest.raises(DocuMatchError):
        raise ConfigError("missing API key")
```

- [ ] **Step 2.2: Run test — expect FAIL**

```bash
pytest tests/unit/test_exceptions.py -v
```
Expected: ImportError (module doesn't exist).

- [ ] **Step 2.3: Implement `src/documatch/exceptions.py`**

```python
"""모든 라이브러리 예외 계층."""


class DocuMatchError(Exception):
    """모든 라이브러리 에러의 베이스."""


class ConfigError(DocuMatchError):
    """설정·API 키 누락/오타."""


class DependencyError(DocuMatchError):
    """시스템 바이너리 누락 (Poppler 등)."""


class UnsupportedFormatError(DocuMatchError):
    """지원하지 않는 확장자."""


class ProcessorError(DocuMatchError):
    """파일 처리 실패."""


class LLMError(DocuMatchError):
    """LLM 호출/응답 실패."""


class LLMRateLimitError(LLMError):
    """429 — rate limit."""


class LLMAuthError(LLMError):
    """401/403 — 인증 실패."""


class SpecError(DocuMatchError):
    """스펙 검증/직렬화 실패."""


class SpecNotFoundError(SpecError):
    """저장된 스펙 이름 미존재."""


class ExtractionError(DocuMatchError):
    """추출 결과 파싱 실패."""


class UserAbort(DocuMatchError):
    """사용자 [Q] 또는 Ctrl+C."""
```

- [ ] **Step 2.4: Run test — expect PASS**

```bash
pytest tests/unit/test_exceptions.py -v
```
Expected: 4 passed.

- [ ] **Step 2.5: Commit**

```bash
git add src/documatch/exceptions.py tests/unit/test_exceptions.py
git commit -m "feat(exceptions): 예외 계층 구조 추가"
```

---

### Task 3: i18n message catalog

**Files:**
- Create: `src/documatch/i18n.py`
- Create: `tests/unit/test_i18n.py`

- [ ] **Step 3.1: Write failing test**

```python
from documatch.i18n import t


def test_lookup_existing_key():
    assert t("cli.welcome") == "DocuMatch CLI"


def test_lookup_with_format():
    assert t("scan.found", count=5) == "5개 파일 발견"


def test_unknown_key_returns_key():
    assert t("nonexistent.key") == "nonexistent.key"
```

- [ ] **Step 3.2: Run — FAIL**

```bash
pytest tests/unit/test_i18n.py -v
```

- [ ] **Step 3.3: Implement `src/documatch/i18n.py`**

```python
"""한국어 메시지 카탈로그. 향후 i18n 확장 시 dict → 외부 파일로 이전."""
from typing import Any

_MESSAGES: dict[str, str] = {
    "cli.welcome": "DocuMatch CLI",
    "scan.found": "{count}개 파일 발견",
    "scan.empty": "지원되는 파일이 없습니다.",
    "spec.approved": "스키마 승인됨.",
    "spec.refining": "스키마 수정 중...",
    "extract.in_progress": "추출 중...",
    "export.saved": "Excel 저장됨: {path}",
    "abort.user": "사용자가 종료했습니다.",
    "error.poppler_missing": (
        "스캔 PDF 처리에 Poppler가 필요합니다.\n"
        "macOS: brew install poppler\n"
        "Linux: sudo apt install poppler-utils\n"
        "Windows: https://github.com/oschwartz10612/poppler-windows/releases"
    ),
    "error.api_key_missing": "{provider} API 키가 설정되지 않았습니다.",
    "error.spec_not_found": "스펙 '{name}'을 찾을 수 없습니다.",
}


def t(key: str, **kwargs: Any) -> str:
    """메시지 조회 + format 치환. 키 없으면 키 자체 반환."""
    template = _MESSAGES.get(key, key)
    if kwargs:
        try:
            return template.format(**kwargs)
        except (KeyError, IndexError):
            return template
    return template
```

- [ ] **Step 3.4: Run — PASS**

- [ ] **Step 3.5: Commit**

```bash
git add src/documatch/i18n.py tests/unit/test_i18n.py
git commit -m "feat(i18n): 한국어 메시지 카탈로그 추가"
```

---

### Task 4: CI workflow

**Files:**
- Create: `.github/workflows/ci.yml`

- [ ] **Step 4.1: Create workflow**

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest]
        python: ["3.11", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python }}
      - name: Install Poppler (Ubuntu)
        if: matrix.os == 'ubuntu-latest'
        run: sudo apt-get update && sudo apt-get install -y poppler-utils
      - name: Install Poppler (macOS)
        if: matrix.os == 'macos-latest'
        run: brew install poppler
      - run: pip install -e ".[dev]"
      - run: ruff check .
      - run: ruff format --check .
      - run: mypy src/documatch
      - run: pytest --cov=documatch --cov-report=term-missing
      - run: documatch --version
```

- [ ] **Step 4.2: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: GitHub Actions 파이프라인 추가"
```

---

## Phase 2 — Configuration (Task 5)

### Task 5: Settings (env + toml + defaults)

**Files:**
- Create: `src/documatch/config/__init__.py`
- Create: `src/documatch/config/defaults.py`
- Create: `src/documatch/config/settings.py`
- Create: `tests/unit/test_config.py`

- [ ] **Step 5.1: Write failing test `tests/unit/test_config.py`**

```python
import os
from pathlib import Path

import pytest

from documatch.config.settings import LLMRoleConfig, Settings


def test_defaults_when_no_env(monkeypatch):
    for var in ["ANTHROPIC_API_KEY", "OPENAI_API_KEY"]:
        monkeypatch.delenv(var, raising=False)
    s = Settings()
    assert s.anthropic_api_key is None
    assert s.openai_api_key is None
    assert s.llm_spec.provider == "anthropic"
    assert s.llm_extract.model == "claude-haiku-4-5-20251001"
    assert s.pdf_scan_threshold == 100


def test_reads_anthropic_api_key_from_env(monkeypatch):
    monkeypatch.setenv("ANTHROPIC_API_KEY", "sk-ant-test")
    s = Settings()
    assert s.anthropic_api_key == "sk-ant-test"


def test_documatch_prefix_for_other_settings(monkeypatch):
    monkeypatch.setenv("DOCUMATCH_DEBUG", "true")
    monkeypatch.setenv("DOCUMATCH_PDF_SCAN_THRESHOLD", "200")
    s = Settings()
    assert s.debug is True
    assert s.pdf_scan_threshold == 200


def test_nested_llm_config(monkeypatch):
    monkeypatch.setenv("DOCUMATCH_LLM_EXTRACT__PROVIDER", "openai")
    monkeypatch.setenv("DOCUMATCH_LLM_EXTRACT__MODEL", "gpt-4o-mini")
    s = Settings()
    assert s.llm_extract.provider == "openai"
    assert s.llm_extract.model == "gpt-4o-mini"
```

- [ ] **Step 5.2: Run — FAIL (ImportError)**

```bash
pytest tests/unit/test_config.py -v
```

- [ ] **Step 5.3: Implement `src/documatch/config/__init__.py`**

```python
from documatch.config.settings import LLMRoleConfig, Settings

__all__ = ["Settings", "LLMRoleConfig"]
```

- [ ] **Step 5.4: Implement `src/documatch/config/defaults.py`**

```python
"""기본 모델·임계값."""

DEFAULT_SPEC_MODEL = "claude-sonnet-4-6"
DEFAULT_EXTRACT_MODEL = "claude-haiku-4-5-20251001"
DEFAULT_PDF_SCAN_THRESHOLD = 100
DEFAULT_BATCH_SIZE = 20
DEFAULT_TIMEOUT = 120.0
DEFAULT_MAX_RETRIES = 3
```

- [ ] **Step 5.5: Implement `src/documatch/config/settings.py`**

```python
"""사용자 설정. 우선순위: defaults < toml < env < CLI 플래그."""
from pathlib import Path
from typing import Literal

from pydantic import BaseModel, Field
from pydantic_settings import BaseSettings, SettingsConfigDict

from documatch.config.defaults import (
    DEFAULT_BATCH_SIZE,
    DEFAULT_EXTRACT_MODEL,
    DEFAULT_MAX_RETRIES,
    DEFAULT_PDF_SCAN_THRESHOLD,
    DEFAULT_SPEC_MODEL,
    DEFAULT_TIMEOUT,
)


class LLMRoleConfig(BaseModel):
    provider: Literal["anthropic", "openai"] = "anthropic"
    model: str = DEFAULT_SPEC_MODEL
    timeout: float = DEFAULT_TIMEOUT
    max_retries: int = DEFAULT_MAX_RETRIES


class Settings(BaseSettings):
    anthropic_api_key: str | None = Field(default=None, validation_alias="ANTHROPIC_API_KEY")
    openai_api_key: str | None = Field(default=None, validation_alias="OPENAI_API_KEY")

    llm_spec: LLMRoleConfig = LLMRoleConfig()
    llm_extract: LLMRoleConfig = LLMRoleConfig(model=DEFAULT_EXTRACT_MODEL)

    spec_store_dir: Path = Path("~/.config/documatch/specs").expanduser()
    pdf_scan_threshold: int = DEFAULT_PDF_SCAN_THRESHOLD
    extraction_batch_size: int = DEFAULT_BATCH_SIZE
    debug: bool = False

    model_config = SettingsConfigDict(
        env_prefix="DOCUMATCH_",
        env_nested_delimiter="__",
        populate_by_name=True,
        extra="ignore",
    )
```

- [ ] **Step 5.6: Run — PASS**

```bash
pytest tests/unit/test_config.py -v
```

- [ ] **Step 5.7: Commit**

```bash
git add src/documatch/config tests/unit/test_config.py
git commit -m "feat(config): pydantic-settings 기반 Settings + 기본값 모듈"
```

---

## Phase 3 — Spec Models & Store (Tasks 6-7)

### Task 6: ExtractionSpec models

**Files:**
- Create: `src/documatch/spec/__init__.py`
- Create: `src/documatch/spec/models.py`
- Create: `tests/unit/test_spec_models.py`

- [ ] **Step 6.1: Write failing test**

```python
import json

import pytest

from documatch.spec.models import (
    ExtractionSpec,
    Field,
    FieldType,
    OutputColumn,
    ReviewRule,
    SummaryTemplate,
)


def make_spec(name="test_spec") -> ExtractionSpec:
    return ExtractionSpec(
        name=name,
        doc_type_hint="invoice",
        fields=[Field(name="invoice_no", description="송장번호", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="송장 {invoice_no}"),
        output_table=[OutputColumn(field_name="invoice_no", excel_header="송장번호")],
    )


def test_spec_defaults():
    s = make_spec()
    assert s.schema_version == "1"
    assert s.table_mode is False
    assert len(s.review_rules) == 2
    assert s.review_rules[0].condition == "low_confidence"


def test_field_required_default():
    f = Field(name="x", description="d")
    assert f.required is True
    assert f.type == FieldType.STRING


def test_round_trip_json():
    s = make_spec("rt")
    j = s.model_dump_json()
    s2 = ExtractionSpec.model_validate_json(j)
    assert s2 == s


def test_invalid_schema_version_rejected():
    with pytest.raises(ValueError):
        ExtractionSpec(
            schema_version="2",
            name="x",
            doc_type_hint="x",
            fields=[],
            summary_template=SummaryTemplate(template="x"),
            output_table=[],
        )
```

- [ ] **Step 6.2: Run — FAIL**

```bash
pytest tests/unit/test_spec_models.py -v
```

- [ ] **Step 6.3: Implement `src/documatch/spec/__init__.py`**

```python
from documatch.spec.models import (
    ExtractionSpec,
    Field,
    FieldType,
    OutputColumn,
    ReviewRule,
    SummaryTemplate,
)

__all__ = [
    "ExtractionSpec", "Field", "FieldType", "OutputColumn",
    "ReviewRule", "SummaryTemplate",
]
```

- [ ] **Step 6.4: Implement `src/documatch/spec/models.py`**

```python
"""ExtractionSpec 도메인 모델."""
from datetime import datetime
from enum import Enum
from typing import Literal

from pydantic import BaseModel, Field as PydField


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


def _default_review_rules() -> list[ReviewRule]:
    return [
        ReviewRule(condition="low_confidence", threshold=0.7),
        ReviewRule(condition="missing_required"),
    ]


class ExtractionSpec(BaseModel):
    schema_version: Literal["1"] = "1"
    name: str
    description: str | None = None
    doc_type_hint: str
    fields: list[Field]
    summary_template: SummaryTemplate
    output_table: list[OutputColumn]
    review_rules: list[ReviewRule] = PydField(default_factory=_default_review_rules)
    table_mode: bool = False
    created_at: datetime = PydField(default_factory=datetime.utcnow)
    source_sample: str | None = None
```

- [ ] **Step 6.5: Run — PASS**

- [ ] **Step 6.6: Commit**

```bash
git add src/documatch/spec tests/unit/test_spec_models.py
git commit -m "feat(spec): ExtractionSpec + Field 모델 정의"
```

---

### Task 7: SpecStore (save/load/list/delete)

**Files:**
- Create: `src/documatch/spec/store.py`
- Modify: `src/documatch/spec/__init__.py`
- Create: `tests/unit/test_spec_store.py`

- [ ] **Step 7.1: Write failing test**

```python
from pathlib import Path

import pytest

from documatch.exceptions import SpecError, SpecNotFoundError
from documatch.spec.models import (
    ExtractionSpec,
    Field,
    FieldType,
    OutputColumn,
    SummaryTemplate,
)
from documatch.spec.store import SpecStore


def make_spec(name="invoice_v1") -> ExtractionSpec:
    return ExtractionSpec(
        name=name,
        doc_type_hint="invoice",
        fields=[Field(name="no", description="번호", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{no}"),
        output_table=[OutputColumn(field_name="no", excel_header="번호")],
    )


def test_save_and_load_roundtrip(tmp_path: Path):
    store = SpecStore(base_dir=tmp_path)
    spec = make_spec()
    p = store.save(spec)
    assert p.exists()
    loaded = store.load("invoice_v1")
    assert loaded == spec


def test_load_unknown_raises(tmp_path: Path):
    store = SpecStore(base_dir=tmp_path)
    with pytest.raises(SpecNotFoundError):
        store.load("missing")


def test_save_overwrite_protection(tmp_path: Path):
    store = SpecStore(base_dir=tmp_path)
    store.save(make_spec())
    with pytest.raises(SpecError):
        store.save(make_spec(), overwrite=False)
    store.save(make_spec(), overwrite=True)  # OK


def test_list(tmp_path: Path):
    store = SpecStore(base_dir=tmp_path)
    store.save(make_spec("a"))
    store.save(make_spec("b"))
    names = sorted(s.name for s in store.list())
    assert names == ["a", "b"]


def test_delete(tmp_path: Path):
    store = SpecStore(base_dir=tmp_path)
    store.save(make_spec("a"))
    store.delete("a")
    with pytest.raises(SpecNotFoundError):
        store.load("a")


def test_invalid_name_rejected(tmp_path: Path):
    store = SpecStore(base_dir=tmp_path)
    bad = make_spec("../etc/passwd")
    with pytest.raises(SpecError):
        store.save(bad)


def test_load_from_path(tmp_path: Path):
    p = tmp_path / "external.json"
    p.write_text(make_spec("ext").model_dump_json())
    store = SpecStore(base_dir=tmp_path / "store")
    loaded = store.load_from_path(p)
    assert loaded.name == "ext"
```

- [ ] **Step 7.2: Run — FAIL**

- [ ] **Step 7.3: Implement `src/documatch/spec/store.py`**

```python
"""ExtractionSpec JSON 영속화."""
import re
from dataclasses import dataclass
from datetime import datetime
from pathlib import Path

from documatch.exceptions import SpecError, SpecNotFoundError
from documatch.spec.models import ExtractionSpec

_NAME_PATTERN = re.compile(r"^[A-Za-z0-9_-]+$")


@dataclass
class SpecInfo:
    name: str
    doc_type_hint: str
    created_at: datetime
    path: Path


class SpecStore:
    """~/.config/documatch/specs/ 기반 JSON 저장/로드."""

    def __init__(self, base_dir: Path):
        self.base_dir = Path(base_dir)
        self.base_dir.mkdir(parents=True, exist_ok=True)

    def _validate_name(self, name: str) -> None:
        if not _NAME_PATTERN.match(name):
            raise SpecError(
                f"잘못된 스펙 이름: {name!r}. 영숫자/하이픈/언더스코어만 허용."
            )

    def _path_for(self, name: str) -> Path:
        self._validate_name(name)
        return self.base_dir / f"{name}.json"

    def save(self, spec: ExtractionSpec, *, overwrite: bool = False) -> Path:
        path = self._path_for(spec.name)
        if path.exists() and not overwrite:
            raise SpecError(f"스펙 '{spec.name}'이 이미 존재합니다 (overwrite=False).")
        path.write_text(spec.model_dump_json(indent=2))
        return path

    def load(self, name: str) -> ExtractionSpec:
        path = self._path_for(name)
        if not path.exists():
            raise SpecNotFoundError(f"스펙 '{name}'을 찾을 수 없습니다.")
        return ExtractionSpec.model_validate_json(path.read_text())

    def load_from_path(self, path: Path) -> ExtractionSpec:
        if not path.exists():
            raise SpecNotFoundError(f"파일 없음: {path}")
        return ExtractionSpec.model_validate_json(path.read_text())

    def list(self) -> list[SpecInfo]:
        out: list[SpecInfo] = []
        for p in sorted(self.base_dir.glob("*.json")):
            try:
                spec = ExtractionSpec.model_validate_json(p.read_text())
            except Exception:
                continue
            out.append(SpecInfo(
                name=spec.name,
                doc_type_hint=spec.doc_type_hint,
                created_at=spec.created_at,
                path=p,
            ))
        return out

    def delete(self, name: str) -> None:
        path = self._path_for(name)
        if not path.exists():
            raise SpecNotFoundError(f"스펙 '{name}'을 찾을 수 없습니다.")
        path.unlink()
```

- [ ] **Step 7.4: Update `src/documatch/spec/__init__.py`**

```python
from documatch.spec.models import (
    ExtractionSpec,
    Field,
    FieldType,
    OutputColumn,
    ReviewRule,
    SummaryTemplate,
)
from documatch.spec.store import SpecInfo, SpecStore

__all__ = [
    "ExtractionSpec", "Field", "FieldType", "OutputColumn",
    "ReviewRule", "SummaryTemplate", "SpecStore", "SpecInfo",
]
```

- [ ] **Step 7.5: Run — PASS**

- [ ] **Step 7.6: Commit**

```bash
git add src/documatch/spec tests/unit/test_spec_store.py
git commit -m "feat(spec): SpecStore JSON 저장/로드/목록/삭제"
```

---

## Phase 4 — LLM Provider Abstraction (Tasks 8-12)

### Task 8: LLM base types + JSON parser

**Files:**
- Create: `src/documatch/llm/__init__.py`
- Create: `src/documatch/llm/base.py`
- Create: `src/documatch/llm/parsing.py`
- Create: `tests/unit/test_llm_parsing.py`

- [ ] **Step 8.1: Failing test for JSON parser**

```python
import pytest

from documatch.exceptions import ExtractionError
from documatch.llm.parsing import parse_json_response


def test_plain_json():
    assert parse_json_response('{"a": 1}') == {"a": 1}


def test_strips_fenced_block():
    text = "```json\n{\"a\": 2}\n```"
    assert parse_json_response(text) == {"a": 2}


def test_strips_generic_fence():
    assert parse_json_response("```\n{\"a\": 3}\n```") == {"a": 3}


def test_extracts_first_json_object():
    text = "Here is the JSON:\n{\"a\": 4}\nThanks!"
    assert parse_json_response(text) == {"a": 4}


def test_unparseable_raises():
    with pytest.raises(ExtractionError):
        parse_json_response("not json at all")
```

- [ ] **Step 8.2: Run — FAIL**

- [ ] **Step 8.3: Implement `src/documatch/llm/base.py`**

```python
"""LLM Provider 추상화."""
from dataclasses import dataclass, field
from typing import Any, Literal, Protocol


@dataclass
class LLMMessage:
    role: Literal["system", "user", "assistant"]
    content: str | list[dict[str, Any]]


@dataclass
class TokenUsage:
    input_tokens: int = 0
    output_tokens: int = 0
    cache_creation_tokens: int = 0
    cache_read_tokens: int = 0


@dataclass
class LLMResponse:
    content: str
    usage: TokenUsage = field(default_factory=TokenUsage)
    raw: Any = None


class LLMClient(Protocol):
    provider: str
    model: str
    supports_vision: bool
    supports_prompt_cache: bool

    async def ainvoke(
        self,
        messages: list[LLMMessage],
        *,
        json_schema: dict[str, Any] | None = None,
        cache_breakpoint: int | None = None,
        temperature: float = 0.0,
        max_tokens: int = 4096,
    ) -> LLMResponse: ...
```

- [ ] **Step 8.4: Implement `src/documatch/llm/parsing.py`**

```python
"""LLM JSON 응답 파서. fenced block 제거 + lenient JSON 추출."""
import json
import re
from typing import Any

from documatch.exceptions import ExtractionError

_FENCE = re.compile(r"^```(?:json)?\s*\n|\n```\s*$", re.MULTILINE)
_OBJECT = re.compile(r"\{.*\}", re.DOTALL)


def parse_json_response(text: str) -> dict[str, Any]:
    """LLM 응답 → dict. fenced block 제거, 첫 JSON 객체 추출, lenient parsing."""
    cleaned = _FENCE.sub("", text).strip()
    try:
        return json.loads(cleaned)
    except json.JSONDecodeError:
        match = _OBJECT.search(cleaned)
        if not match:
            raise ExtractionError(f"JSON 응답을 찾을 수 없음: {text[:200]}")
        try:
            return json.loads(match.group())
        except json.JSONDecodeError as e:
            raise ExtractionError(f"JSON 파싱 실패: {e}; raw={text[:200]}")
```

- [ ] **Step 8.5: Implement `src/documatch/llm/__init__.py`**

```python
from documatch.llm.base import LLMClient, LLMMessage, LLMResponse, TokenUsage
from documatch.llm.parsing import parse_json_response

__all__ = ["LLMClient", "LLMMessage", "LLMResponse", "TokenUsage", "parse_json_response"]
```

- [ ] **Step 8.6: Run — PASS**

- [ ] **Step 8.7: Commit**

```bash
git add src/documatch/llm tests/unit/test_llm_parsing.py
git commit -m "feat(llm): LLMClient Protocol + JSON 응답 파서"
```

---

### Task 9: FakeLLMClient (test fixture)

**Files:**
- Create: `tests/fakes.py`
- Create: `tests/unit/test_fake_llm.py`

- [ ] **Step 9.1: Failing test**

```python
import pytest

from documatch.llm.base import LLMMessage
from tests.fakes import FakeLLMClient


@pytest.mark.asyncio
async def test_returns_registered_response():
    client = FakeLLMClient(responses=['{"ok": true}'])
    resp = await client.ainvoke([LLMMessage(role="user", content="hi")])
    assert resp.content == '{"ok": true}'
    assert client.call_count == 1


@pytest.mark.asyncio
async def test_pattern_match():
    client = FakeLLMClient()
    client.register("invoice", '{"type": "invoice"}')
    resp = await client.ainvoke([LLMMessage(role="user", content="parse this invoice")])
    assert resp.content == '{"type": "invoice"}'


@pytest.mark.asyncio
async def test_unmatched_raises():
    client = FakeLLMClient(responses=[])
    with pytest.raises(AssertionError):
        await client.ainvoke([LLMMessage(role="user", content="hi")])
```

- [ ] **Step 9.2: Implement `tests/fakes.py`**

```python
"""테스트용 결정적 LLM 클라이언트."""
from typing import Any

from documatch.llm.base import LLMClient, LLMMessage, LLMResponse, TokenUsage


class FakeLLMClient:
    provider = "fake"
    model = "fake-model"
    supports_vision = True
    supports_prompt_cache = False

    def __init__(self, responses: list[str] | None = None) -> None:
        self._queue: list[str] = list(responses or [])
        self._patterns: list[tuple[str, str]] = []
        self.call_count = 0
        self.calls: list[list[LLMMessage]] = []

    def register(self, substring: str, response: str) -> None:
        """입력 메시지에 substring이 포함되면 response 반환."""
        self._patterns.append((substring, response))

    async def ainvoke(
        self,
        messages: list[LLMMessage],
        *,
        json_schema: dict[str, Any] | None = None,
        cache_breakpoint: int | None = None,
        temperature: float = 0.0,
        max_tokens: int = 4096,
    ) -> LLMResponse:
        self.call_count += 1
        self.calls.append(messages)
        joined = " ".join(
            m.content if isinstance(m.content, str) else str(m.content) for m in messages
        )
        for substring, response in self._patterns:
            if substring in joined:
                return LLMResponse(content=response, usage=TokenUsage(), raw=None)
        if self._queue:
            return LLMResponse(content=self._queue.pop(0), usage=TokenUsage(), raw=None)
        raise AssertionError(
            f"FakeLLMClient: 응답이 등록되지 않은 호출. messages={messages}"
        )


# Protocol conformance check at import time (mypy)
_check: LLMClient = FakeLLMClient()
```

- [ ] **Step 9.3: Run — PASS**

- [ ] **Step 9.4: Commit**

```bash
git add tests/fakes.py tests/unit/test_fake_llm.py
git commit -m "test(llm): FakeLLMClient 결정적 테스트 픽스처"
```

---

### Task 10: AnthropicClient

**Files:**
- Create: `src/documatch/llm/anthropic.py`
- Create: `tests/unit/test_llm_anthropic.py`

- [ ] **Step 10.1: Failing test (sans real API)**

```python
from unittest.mock import AsyncMock, MagicMock

import pytest

from documatch.llm.anthropic import AnthropicClient
from documatch.llm.base import LLMMessage


@pytest.mark.asyncio
async def test_invoke_extracts_content_and_usage():
    fake_sdk = MagicMock()
    fake_sdk.messages.create = AsyncMock(return_value=MagicMock(
        content=[MagicMock(text='{"hello": "world"}')],
        usage=MagicMock(
            input_tokens=10, output_tokens=5,
            cache_creation_input_tokens=0, cache_read_input_tokens=2,
        ),
    ))
    client = AnthropicClient(model="claude-haiku-4-5-20251001", api_key="x", _sdk=fake_sdk)
    resp = await client.ainvoke([LLMMessage(role="user", content="hi")])
    assert resp.content == '{"hello": "world"}'
    assert resp.usage.input_tokens == 10
    assert resp.usage.cache_read_tokens == 2


@pytest.mark.asyncio
async def test_system_message_split():
    fake_sdk = MagicMock()
    fake_sdk.messages.create = AsyncMock(return_value=MagicMock(
        content=[MagicMock(text="x")],
        usage=MagicMock(
            input_tokens=1, output_tokens=1,
            cache_creation_input_tokens=0, cache_read_input_tokens=0,
        ),
    ))
    client = AnthropicClient(model="m", api_key="x", _sdk=fake_sdk)
    await client.ainvoke([
        LLMMessage(role="system", content="sys"),
        LLMMessage(role="user", content="u"),
    ])
    kwargs = fake_sdk.messages.create.call_args.kwargs
    assert kwargs["system"] == "sys"
    assert kwargs["messages"] == [{"role": "user", "content": "u"}]
```

- [ ] **Step 10.2: Run — FAIL**

- [ ] **Step 10.3: Implement `src/documatch/llm/anthropic.py`**

```python
"""Anthropic Claude 구현. prompt caching 활용."""
from typing import Any

from documatch.exceptions import LLMAuthError, LLMError, LLMRateLimitError
from documatch.llm.base import LLMMessage, LLMResponse, TokenUsage


class AnthropicClient:
    provider = "anthropic"
    supports_vision = True
    supports_prompt_cache = True

    def __init__(
        self,
        model: str,
        api_key: str,
        *,
        timeout: float = 120.0,
        max_retries: int = 3,
        _sdk: Any | None = None,
    ) -> None:
        self.model = model
        self.timeout = timeout
        self.max_retries = max_retries
        if _sdk is not None:
            self._sdk = _sdk
        else:
            from anthropic import AsyncAnthropic
            self._sdk = AsyncAnthropic(api_key=api_key, timeout=timeout)

    async def ainvoke(
        self,
        messages: list[LLMMessage],
        *,
        json_schema: dict[str, Any] | None = None,
        cache_breakpoint: int | None = None,
        temperature: float = 0.0,
        max_tokens: int = 4096,
    ) -> LLMResponse:
        system_text: str | None = None
        msgs: list[dict[str, Any]] = []
        for m in messages:
            if m.role == "system":
                system_text = m.content if isinstance(m.content, str) else str(m.content)
            else:
                msgs.append({"role": m.role, "content": m.content})

        kwargs: dict[str, Any] = {
            "model": self.model,
            "max_tokens": max_tokens,
            "temperature": temperature,
            "messages": msgs,
        }
        if system_text is not None:
            kwargs["system"] = system_text
        if json_schema is not None:
            kwargs["tools"] = [{
                "name": "extract",
                "description": "Return data matching the schema.",
                "input_schema": json_schema,
            }]
            kwargs["tool_choice"] = {"type": "tool", "name": "extract"}

        try:
            resp = await self._sdk.messages.create(**kwargs)
        except Exception as e:
            msg = str(e).lower()
            if "rate" in msg or "429" in msg:
                raise LLMRateLimitError(str(e)) from e
            if "auth" in msg or "401" in msg or "403" in msg:
                raise LLMAuthError(str(e)) from e
            raise LLMError(str(e)) from e

        if json_schema is not None and resp.content:
            block = resp.content[0]
            content_text = (
                getattr(block, "input", None) and __import__("json").dumps(block.input)
                or getattr(block, "text", "")
            )
        else:
            content_text = "".join(getattr(b, "text", "") for b in resp.content)

        u = resp.usage
        return LLMResponse(
            content=content_text,
            usage=TokenUsage(
                input_tokens=getattr(u, "input_tokens", 0),
                output_tokens=getattr(u, "output_tokens", 0),
                cache_creation_tokens=getattr(u, "cache_creation_input_tokens", 0) or 0,
                cache_read_tokens=getattr(u, "cache_read_input_tokens", 0) or 0,
            ),
            raw=resp,
        )
```

- [ ] **Step 10.4: Run — PASS**

- [ ] **Step 10.5: Commit**

```bash
git add src/documatch/llm/anthropic.py tests/unit/test_llm_anthropic.py
git commit -m "feat(llm): AnthropicClient (prompt caching + json schema tool)"
```

---

### Task 11: OpenAIClient

**Files:**
- Create: `src/documatch/llm/openai.py`
- Create: `tests/unit/test_llm_openai.py`

- [ ] **Step 11.1: Failing test**

```python
from unittest.mock import AsyncMock, MagicMock

import pytest

from documatch.llm.base import LLMMessage
from documatch.llm.openai import OpenAIClient


@pytest.mark.asyncio
async def test_invoke_returns_content():
    fake_sdk = MagicMock()
    fake_sdk.chat.completions.create = AsyncMock(return_value=MagicMock(
        choices=[MagicMock(message=MagicMock(content='{"x": 1}'))],
        usage=MagicMock(prompt_tokens=10, completion_tokens=5),
    ))
    client = OpenAIClient(model="gpt-4o-mini", api_key="x", _sdk=fake_sdk)
    resp = await client.ainvoke([LLMMessage(role="user", content="hi")])
    assert resp.content == '{"x": 1}'
    assert resp.usage.input_tokens == 10
```

- [ ] **Step 11.2: Run — FAIL**

- [ ] **Step 11.3: Implement `src/documatch/llm/openai.py`**

```python
"""OpenAI GPT 구현."""
from typing import Any

from documatch.exceptions import LLMAuthError, LLMError, LLMRateLimitError
from documatch.llm.base import LLMMessage, LLMResponse, TokenUsage


class OpenAIClient:
    provider = "openai"
    supports_vision = True
    supports_prompt_cache = True

    def __init__(
        self,
        model: str,
        api_key: str,
        *,
        timeout: float = 120.0,
        max_retries: int = 3,
        _sdk: Any | None = None,
    ) -> None:
        self.model = model
        self.timeout = timeout
        self.max_retries = max_retries
        if _sdk is not None:
            self._sdk = _sdk
        else:
            from openai import AsyncOpenAI
            self._sdk = AsyncOpenAI(api_key=api_key, timeout=timeout)

    async def ainvoke(
        self,
        messages: list[LLMMessage],
        *,
        json_schema: dict[str, Any] | None = None,
        cache_breakpoint: int | None = None,
        temperature: float = 0.0,
        max_tokens: int = 4096,
    ) -> LLMResponse:
        msgs = [{"role": m.role, "content": m.content} for m in messages]
        kwargs: dict[str, Any] = {
            "model": self.model,
            "messages": msgs,
            "temperature": temperature,
            "max_tokens": max_tokens,
        }
        if json_schema is not None:
            kwargs["response_format"] = {
                "type": "json_schema",
                "json_schema": {"name": "extract", "schema": json_schema, "strict": False},
            }

        try:
            resp = await self._sdk.chat.completions.create(**kwargs)
        except Exception as e:
            msg = str(e).lower()
            if "rate" in msg or "429" in msg:
                raise LLMRateLimitError(str(e)) from e
            if "auth" in msg or "401" in msg or "403" in msg:
                raise LLMAuthError(str(e)) from e
            raise LLMError(str(e)) from e

        content = resp.choices[0].message.content or ""
        u = resp.usage
        return LLMResponse(
            content=content,
            usage=TokenUsage(
                input_tokens=getattr(u, "prompt_tokens", 0),
                output_tokens=getattr(u, "completion_tokens", 0),
            ),
            raw=resp,
        )
```

- [ ] **Step 11.4: Run — PASS**

- [ ] **Step 11.5: Commit**

```bash
git add src/documatch/llm/openai.py tests/unit/test_llm_openai.py
git commit -m "feat(llm): OpenAIClient (response_format json_schema)"
```

---

### Task 12: LLM factory

**Files:**
- Create: `src/documatch/llm/factory.py`
- Modify: `src/documatch/llm/__init__.py`
- Create: `tests/unit/test_llm_factory.py`

- [ ] **Step 12.1: Failing test**

```python
import pytest

from documatch.config.settings import LLMRoleConfig, Settings
from documatch.exceptions import ConfigError
from documatch.llm.factory import create_llm


def test_creates_anthropic(monkeypatch):
    monkeypatch.setenv("ANTHROPIC_API_KEY", "sk-ant-x")
    s = Settings()
    s.llm_spec = LLMRoleConfig(provider="anthropic", model="claude-sonnet-4-6")
    client = create_llm(s, role="spec")
    assert client.provider == "anthropic"
    assert client.model == "claude-sonnet-4-6"


def test_creates_openai(monkeypatch):
    monkeypatch.setenv("OPENAI_API_KEY", "sk-x")
    s = Settings()
    s.llm_spec = LLMRoleConfig(provider="openai", model="gpt-4o")
    client = create_llm(s, role="spec")
    assert client.provider == "openai"


def test_missing_key_raises(monkeypatch):
    monkeypatch.delenv("ANTHROPIC_API_KEY", raising=False)
    s = Settings(anthropic_api_key=None)
    s.llm_spec = LLMRoleConfig(provider="anthropic", model="m")
    with pytest.raises(ConfigError):
        create_llm(s, role="spec")
```

- [ ] **Step 12.2: Run — FAIL**

- [ ] **Step 12.3: Implement `src/documatch/llm/factory.py`**

```python
"""설정 기반 LLMClient 생성."""
from typing import Literal

from documatch.config.settings import Settings
from documatch.exceptions import ConfigError
from documatch.llm.anthropic import AnthropicClient
from documatch.llm.base import LLMClient
from documatch.llm.openai import OpenAIClient


def create_llm(settings: Settings, *, role: Literal["spec", "extract"]) -> LLMClient:
    cfg = settings.llm_spec if role == "spec" else settings.llm_extract
    if cfg.provider == "anthropic":
        if not settings.anthropic_api_key:
            raise ConfigError("ANTHROPIC_API_KEY 환경변수가 설정되지 않았습니다.")
        return AnthropicClient(
            model=cfg.model,
            api_key=settings.anthropic_api_key,
            timeout=cfg.timeout,
            max_retries=cfg.max_retries,
        )
    if cfg.provider == "openai":
        if not settings.openai_api_key:
            raise ConfigError("OPENAI_API_KEY 환경변수가 설정되지 않았습니다.")
        return OpenAIClient(
            model=cfg.model,
            api_key=settings.openai_api_key,
            timeout=cfg.timeout,
            max_retries=cfg.max_retries,
        )
    raise ConfigError(f"알 수 없는 provider: {cfg.provider}")
```

- [ ] **Step 12.4: Update `src/documatch/llm/__init__.py`**

```python
from documatch.llm.anthropic import AnthropicClient
from documatch.llm.base import LLMClient, LLMMessage, LLMResponse, TokenUsage
from documatch.llm.factory import create_llm
from documatch.llm.openai import OpenAIClient
from documatch.llm.parsing import parse_json_response

__all__ = [
    "LLMClient", "LLMMessage", "LLMResponse", "TokenUsage",
    "AnthropicClient", "OpenAIClient", "create_llm", "parse_json_response",
]
```

- [ ] **Step 12.5: Run — PASS**

- [ ] **Step 12.6: Commit**

```bash
git add src/documatch/llm tests/unit/test_llm_factory.py
git commit -m "feat(llm): create_llm factory + role 기반 클라이언트 선택"
```

---

## Phase 5 — File Processors (Tasks 13-21)

### Task 13: Processor base + UnifiedDocument + registry

**Files:**
- Create: `src/documatch/processors/__init__.py`
- Create: `src/documatch/processors/base.py`
- Create: `src/documatch/processors/registry.py`
- Create: `tests/unit/test_registry.py`

- [ ] **Step 13.1: Failing test**

```python
import pytest

from documatch.exceptions import UnsupportedFormatError
from documatch.processors.registry import get_processor, supported_extensions


def test_pdf_lookup_returns_processor():
    p = get_processor("pdf")
    assert p is not None
    assert "pdf" in [e.lstrip(".") for e in p.extensions]


def test_unknown_raises():
    with pytest.raises(UnsupportedFormatError):
        get_processor("xyz")


def test_supported_extensions_includes_core():
    exts = set(supported_extensions())
    assert {".pdf", ".docx", ".xlsx", ".jpg", ".png", ".txt", ".md", ".csv", ".html"} <= exts
```

- [ ] **Step 13.2: Run — FAIL**

- [ ] **Step 13.3: Implement `src/documatch/processors/base.py`**

```python
"""DocumentProcessor Protocol + UnifiedDocument."""
from dataclasses import dataclass, field
from typing import Any, ClassVar, Protocol


@dataclass
class FileEntry:
    file_path: str
    filename: str
    extension: str
    size_bytes: int
    sheet_name: str | None = None


@dataclass
class UnifiedDocument:
    filename: str
    raw_text: str
    page_count: int
    is_scanned: bool = False
    images_base64: list[str] | None = None
    metadata: dict[str, Any] = field(default_factory=dict)


class DocumentProcessor(Protocol):
    extensions: ClassVar[tuple[str, ...]]
    requires_vision: ClassVar[bool]

    async def process(self, content: bytes, filename: str) -> UnifiedDocument: ...
```

- [ ] **Step 13.4: Implement `src/documatch/processors/registry.py`**

Note: registers stub classes for now; concrete processors land in Tasks 14-21. Import lazily to avoid circular deps.

```python
"""확장자 → DocumentProcessor 매핑."""
from typing import TYPE_CHECKING

from documatch.exceptions import UnsupportedFormatError

if TYPE_CHECKING:
    from documatch.processors.base import DocumentProcessor


def _build_registry() -> dict[str, type]:
    from documatch.processors.docx import DocxProcessor
    from documatch.processors.image import ImageProcessor
    from documatch.processors.pdf import PdfProcessor
    from documatch.processors.text import (
        CsvProcessor, HtmlProcessor, MarkdownProcessor, TextProcessor,
    )
    from documatch.processors.xls import XlsProcessor
    from documatch.processors.xlsx import XlsxProcessor

    return {
        ".pdf": PdfProcessor,
        ".docx": DocxProcessor,
        ".xlsx": XlsxProcessor,
        ".xls": XlsProcessor,
        ".jpg": ImageProcessor,
        ".jpeg": ImageProcessor,
        ".png": ImageProcessor,
        ".txt": TextProcessor,
        ".md": MarkdownProcessor,
        ".csv": CsvProcessor,
        ".html": HtmlProcessor,
        ".htm": HtmlProcessor,
    }


_REGISTRY: dict[str, type] | None = None


def _registry() -> dict[str, type]:
    global _REGISTRY
    if _REGISTRY is None:
        _REGISTRY = _build_registry()
    return _REGISTRY


def get_processor(extension: str) -> "DocumentProcessor":
    ext = extension.lower()
    if not ext.startswith("."):
        ext = "." + ext
    cls = _registry().get(ext)
    if cls is None:
        raise UnsupportedFormatError(f"지원하지 않는 확장자: {extension}")
    return cls()


def supported_extensions() -> tuple[str, ...]:
    return tuple(sorted(_registry().keys()))
```

- [ ] **Step 13.5: Implement `src/documatch/processors/__init__.py`**

```python
from documatch.processors.base import DocumentProcessor, FileEntry, UnifiedDocument
from documatch.processors.registry import get_processor, supported_extensions

__all__ = [
    "DocumentProcessor", "FileEntry", "UnifiedDocument",
    "get_processor", "supported_extensions",
]
```

Note: This task's tests will only pass once Tasks 14-21 (concrete processors) are implemented. Mark task done after Step 13.6 if proceeding TDD-style — registry tests are last.

- [ ] **Step 13.6: Defer test run until Task 21**

Skip pytest run for now (registry imports concrete processors not yet created). Continue to Task 14 first.

- [ ] **Step 13.7: Commit base + registry skeleton**

```bash
git add src/documatch/processors/__init__.py src/documatch/processors/base.py src/documatch/processors/registry.py tests/unit/test_registry.py
git commit -m "feat(processors): base + registry 스켈레톤 (concrete impls pending)"
```

---

### Task 14: TextProcessor + MarkdownProcessor

**Files:**
- Create: `src/documatch/processors/text.py`
- Create: `tests/unit/test_processors_text.py`

- [ ] **Step 14.1: Failing test**

```python
import pytest

from documatch.processors.text import MarkdownProcessor, TextProcessor


@pytest.mark.asyncio
async def test_text_utf8():
    p = TextProcessor()
    doc = await p.process("안녕\nworld".encode("utf-8"), "a.txt")
    assert doc.raw_text == "안녕\nworld"
    assert doc.page_count == 1
    assert doc.is_scanned is False


@pytest.mark.asyncio
async def test_text_cp949_detected():
    p = TextProcessor()
    content = "한글".encode("cp949")
    doc = await p.process(content, "a.txt")
    assert "한글" in doc.raw_text


@pytest.mark.asyncio
async def test_markdown_passthrough():
    p = MarkdownProcessor()
    doc = await p.process(b"# title\n\ntext", "a.md")
    assert doc.raw_text == "# title\n\ntext"
```

- [ ] **Step 14.2: Run — FAIL**

- [ ] **Step 14.3: Implement `src/documatch/processors/text.py`** (TextProcessor + MarkdownProcessor only — Csv/Html in next tasks)

```python
"""TXT/MD/CSV/HTML 텍스트 계열 프로세서."""
import csv
import io
from typing import ClassVar

import chardet
from bs4 import BeautifulSoup

from documatch.exceptions import ProcessorError
from documatch.processors.base import UnifiedDocument


def _decode(content: bytes) -> str:
    if not content:
        return ""
    try:
        return content.decode("utf-8")
    except UnicodeDecodeError:
        guess = chardet.detect(content) or {}
        enc = guess.get("encoding") or "utf-8"
        try:
            return content.decode(enc, errors="replace")
        except LookupError:
            return content.decode("utf-8", errors="replace")


class TextProcessor:
    extensions: ClassVar[tuple[str, ...]] = (".txt",)
    requires_vision: ClassVar[bool] = False

    async def process(self, content: bytes, filename: str) -> UnifiedDocument:
        text = _decode(content)
        return UnifiedDocument(filename=filename, raw_text=text, page_count=1)


class MarkdownProcessor:
    extensions: ClassVar[tuple[str, ...]] = (".md",)
    requires_vision: ClassVar[bool] = False

    async def process(self, content: bytes, filename: str) -> UnifiedDocument:
        text = _decode(content)
        return UnifiedDocument(filename=filename, raw_text=text, page_count=1)


class CsvProcessor:
    extensions: ClassVar[tuple[str, ...]] = (".csv",)
    requires_vision: ClassVar[bool] = False

    async def process(self, content: bytes, filename: str) -> UnifiedDocument:
        text = _decode(content)
        try:
            reader = csv.reader(io.StringIO(text))
            rows = list(reader)
        except csv.Error as e:
            raise ProcessorError(f"CSV 파싱 실패: {e}") from e
        if not rows:
            return UnifiedDocument(filename=filename, raw_text="", page_count=1)
        header, body = rows[0], rows[1:]
        md = "| " + " | ".join(header) + " |\n"
        md += "| " + " | ".join(["---"] * len(header)) + " |\n"
        for r in body:
            padded = r + [""] * (len(header) - len(r))
            md += "| " + " | ".join(padded[: len(header)]) + " |\n"
        return UnifiedDocument(
            filename=filename, raw_text=md, page_count=1,
            metadata={"row_count": len(body)},
        )


class HtmlProcessor:
    extensions: ClassVar[tuple[str, ...]] = (".html", ".htm")
    requires_vision: ClassVar[bool] = False

    async def process(self, content: bytes, filename: str) -> UnifiedDocument:
        text = _decode(content)
        try:
            soup = BeautifulSoup(text, "html.parser")
        except Exception as e:
            raise ProcessorError(f"HTML 파싱 실패: {e}") from e
        for tag in soup(["script", "style", "noscript"]):
            tag.decompose()
        body = soup.get_text(separator="\n", strip=True)
        return UnifiedDocument(filename=filename, raw_text=body, page_count=1)
```

- [ ] **Step 14.4: Run — PASS (TextProcessor + MarkdownProcessor tests)**

- [ ] **Step 14.5: Commit**

```bash
git add src/documatch/processors/text.py tests/unit/test_processors_text.py
git commit -m "feat(processors): TXT/MD/CSV/HTML 텍스트 계열 프로세서"
```

---

### Task 15: CsvProcessor + HtmlProcessor tests

**Files:**
- Modify: `tests/unit/test_processors_text.py`

- [ ] **Step 15.1: Append CSV/HTML tests**

```python
@pytest.mark.asyncio
async def test_csv_to_markdown_table():
    from documatch.processors.text import CsvProcessor
    p = CsvProcessor()
    content = b"name,age\nAlice,30\nBob,25\n"
    doc = await p.process(content, "x.csv")
    assert "| name | age |" in doc.raw_text
    assert "Alice" in doc.raw_text
    assert doc.metadata["row_count"] == 2


@pytest.mark.asyncio
async def test_html_strips_script_style():
    from documatch.processors.text import HtmlProcessor
    p = HtmlProcessor()
    content = b"<html><head><style>x</style></head><body><p>hi</p><script>z</script></body></html>"
    doc = await p.process(content, "x.html")
    assert doc.raw_text.strip() == "hi"
```

- [ ] **Step 15.2: Run — PASS**

- [ ] **Step 15.3: Commit**

```bash
git add tests/unit/test_processors_text.py
git commit -m "test(processors): CSV/HTML 처리 검증"
```

---

### Task 16: DocxProcessor

**Files:**
- Create: `src/documatch/processors/docx.py`
- Create: `tests/fixtures/docs/sample.docx` (small generated DOCX)
- Create: `tests/unit/test_processors_docx.py`

- [ ] **Step 16.1: Generate fixture (one-time helper)**

```bash
python -c "
from docx import Document
d = Document()
d.add_heading('Test', 0)
d.add_paragraph('첫 단락')
t = d.add_table(rows=2, cols=2)
t.cell(0,0).text='이름'; t.cell(0,1).text='값'
t.cell(1,0).text='A'; t.cell(1,1).text='1'
d.save('tests/fixtures/docs/sample.docx')
"
```

- [ ] **Step 16.2: Failing test**

```python
from pathlib import Path

import pytest

from documatch.processors.docx import DocxProcessor

FIXTURE = Path(__file__).parent.parent / "fixtures" / "docs" / "sample.docx"


@pytest.mark.asyncio
async def test_docx_extracts_paragraphs_and_tables():
    p = DocxProcessor()
    doc = await p.process(FIXTURE.read_bytes(), "sample.docx")
    assert "첫 단락" in doc.raw_text
    assert "이름" in doc.raw_text
    assert "A" in doc.raw_text
    assert doc.is_scanned is False
```

- [ ] **Step 16.3: Implement `src/documatch/processors/docx.py`**

```python
"""DOCX 프로세서."""
import io
from typing import ClassVar

from docx import Document

from documatch.exceptions import ProcessorError
from documatch.processors.base import UnifiedDocument


class DocxProcessor:
    extensions: ClassVar[tuple[str, ...]] = (".docx",)
    requires_vision: ClassVar[bool] = False

    async def process(self, content: bytes, filename: str) -> UnifiedDocument:
        try:
            doc = Document(io.BytesIO(content))
        except Exception as e:
            raise ProcessorError(f"DOCX 열기 실패: {e}") from e
        parts: list[str] = []
        for para in doc.paragraphs:
            text = para.text.strip()
            if text:
                parts.append(text)
        for table in doc.tables:
            for row in table.rows:
                cells = [c.text.strip() for c in row.cells]
                parts.append(" | ".join(cells))
        return UnifiedDocument(
            filename=filename,
            raw_text="\n".join(parts),
            page_count=1,
        )
```

- [ ] **Step 16.4: Run — PASS**

- [ ] **Step 16.5: Commit**

```bash
git add src/documatch/processors/docx.py tests/unit/test_processors_docx.py tests/fixtures/docs/sample.docx
git commit -m "feat(processors): DOCX 프로세서 + fixture"
```

---

### Task 17: XlsxProcessor (sheet split)

**Files:**
- Create: `src/documatch/processors/xlsx.py`
- Create: `tests/fixtures/docs/sample.xlsx`
- Create: `tests/unit/test_processors_xlsx.py`

- [ ] **Step 17.1: Generate fixture**

```bash
python -c "
from openpyxl import Workbook
wb = Workbook()
ws = wb.active; ws.title='Sheet1'
ws.append(['이름','값']); ws.append(['A',1]); ws.append(['B',2])
ws2 = wb.create_sheet('Sheet2')
ws2.append(['x','y']); ws2.append([10,20])
wb.save('tests/fixtures/docs/sample.xlsx')
"
```

- [ ] **Step 17.2: Failing test**

```python
from pathlib import Path

import pytest

from documatch.processors.xlsx import XlsxProcessor

FIXTURE = Path(__file__).parent.parent / "fixtures" / "docs" / "sample.xlsx"


@pytest.mark.asyncio
async def test_xlsx_default_sheet():
    p = XlsxProcessor()
    doc = await p.process(FIXTURE.read_bytes(), "sample.xlsx")
    assert "이름" in doc.raw_text
    assert "A" in doc.raw_text
    assert doc.metadata["sheet_name"] == "Sheet1"


@pytest.mark.asyncio
async def test_xlsx_specific_sheet():
    p = XlsxProcessor()
    doc = await p.process_sheet(FIXTURE.read_bytes(), "Sheet2")
    assert "x" in doc.raw_text
    assert "10" in doc.raw_text
    assert doc.metadata["sheet_name"] == "Sheet2"


def test_list_sheets():
    p = XlsxProcessor()
    sheets = p.list_sheets(FIXTURE.read_bytes())
    assert sheets == ["Sheet1", "Sheet2"]
```

- [ ] **Step 17.3: Implement `src/documatch/processors/xlsx.py`**

```python
"""XLSX 프로세서. 시트 단위 분리 지원."""
import io
from typing import ClassVar

from openpyxl import load_workbook

from documatch.exceptions import ProcessorError
from documatch.processors.base import UnifiedDocument


def _sheet_to_text(ws) -> str:
    rows: list[str] = []
    for row in ws.iter_rows(values_only=True):
        cells = [str(c) if c is not None else "" for c in row]
        if any(cells):
            rows.append(" | ".join(cells))
    return "\n".join(rows)


class XlsxProcessor:
    extensions: ClassVar[tuple[str, ...]] = (".xlsx",)
    requires_vision: ClassVar[bool] = False

    def list_sheets(self, content: bytes) -> list[str]:
        try:
            wb = load_workbook(io.BytesIO(content), read_only=True, data_only=True)
        except Exception as e:
            raise ProcessorError(f"XLSX 열기 실패: {e}") from e
        try:
            return list(wb.sheetnames)
        finally:
            wb.close()

    async def process(self, content: bytes, filename: str) -> UnifiedDocument:
        try:
            wb = load_workbook(io.BytesIO(content), read_only=True, data_only=True)
        except Exception as e:
            raise ProcessorError(f"XLSX 열기 실패: {e}") from e
        try:
            ws = wb.active
            text = _sheet_to_text(ws)
            return UnifiedDocument(
                filename=filename, raw_text=text, page_count=1,
                metadata={"sheet_name": ws.title},
            )
        finally:
            wb.close()

    async def process_sheet(self, content: bytes, sheet_name: str) -> UnifiedDocument:
        try:
            wb = load_workbook(io.BytesIO(content), read_only=True, data_only=True)
        except Exception as e:
            raise ProcessorError(f"XLSX 열기 실패: {e}") from e
        try:
            if sheet_name not in wb.sheetnames:
                raise ProcessorError(f"시트 '{sheet_name}' 없음.")
            ws = wb[sheet_name]
            text = _sheet_to_text(ws)
            return UnifiedDocument(
                filename=sheet_name, raw_text=text, page_count=1,
                metadata={"sheet_name": sheet_name},
            )
        finally:
            wb.close()
```

- [ ] **Step 17.4: Run — PASS**

- [ ] **Step 17.5: Commit**

```bash
git add src/documatch/processors/xlsx.py tests/unit/test_processors_xlsx.py tests/fixtures/docs/sample.xlsx
git commit -m "feat(processors): XLSX 프로세서 + 시트 분리"
```

---

### Task 18: XlsProcessor

**Files:**
- Create: `src/documatch/processors/xls.py`
- Create: `tests/unit/test_processors_xls.py` (skip if creating .xls fixture is impractical — basic structural test only)

- [ ] **Step 18.1: Implement `src/documatch/processors/xls.py`**

```python
"""XLS (구형 Excel) 프로세서."""
import io
from typing import ClassVar

import xlrd

from documatch.exceptions import ProcessorError
from documatch.processors.base import UnifiedDocument


def _sheet_to_text(sheet) -> str:
    rows: list[str] = []
    for r in range(sheet.nrows):
        cells = [str(sheet.cell_value(r, c)) for c in range(sheet.ncols)]
        if any(cells):
            rows.append(" | ".join(cells))
    return "\n".join(rows)


class XlsProcessor:
    extensions: ClassVar[tuple[str, ...]] = (".xls",)
    requires_vision: ClassVar[bool] = False

    def list_sheets(self, content: bytes) -> list[str]:
        try:
            book = xlrd.open_workbook(file_contents=content)
        except Exception as e:
            raise ProcessorError(f"XLS 열기 실패: {e}") from e
        return book.sheet_names()

    async def process(self, content: bytes, filename: str) -> UnifiedDocument:
        try:
            book = xlrd.open_workbook(file_contents=content)
        except Exception as e:
            raise ProcessorError(f"XLS 열기 실패: {e}") from e
        sheet = book.sheet_by_index(0)
        text = _sheet_to_text(sheet)
        return UnifiedDocument(
            filename=filename, raw_text=text, page_count=1,
            metadata={"sheet_name": sheet.name},
        )

    async def process_sheet(self, content: bytes, sheet_name: str) -> UnifiedDocument:
        try:
            book = xlrd.open_workbook(file_contents=content)
        except Exception as e:
            raise ProcessorError(f"XLS 열기 실패: {e}") from e
        if sheet_name not in book.sheet_names():
            raise ProcessorError(f"시트 '{sheet_name}' 없음.")
        sheet = book.sheet_by_name(sheet_name)
        return UnifiedDocument(
            filename=sheet_name, raw_text=_sheet_to_text(sheet), page_count=1,
            metadata={"sheet_name": sheet_name},
        )
```

- [ ] **Step 18.2: Smoke import test**

```python
from documatch.processors.xls import XlsProcessor


def test_smoke_can_instantiate():
    assert XlsProcessor().extensions == (".xls",)
```

- [ ] **Step 18.3: Commit**

```bash
git add src/documatch/processors/xls.py tests/unit/test_processors_xls.py
git commit -m "feat(processors): XLS 프로세서 (xlrd)"
```

---

### Task 19: ImageProcessor

**Files:**
- Create: `src/documatch/processors/image.py`
- Create: `tests/fixtures/docs/sample.png`
- Create: `tests/unit/test_processors_image.py`

- [ ] **Step 19.1: Generate fixture**

```bash
python -c "
from PIL import Image
img = Image.new('RGB', (40, 20), 'white')
img.save('tests/fixtures/docs/sample.png')
"
```

- [ ] **Step 19.2: Failing test**

```python
import base64
from pathlib import Path

import pytest

from documatch.processors.image import ImageProcessor

FIXTURE = Path(__file__).parent.parent / "fixtures" / "docs" / "sample.png"


@pytest.mark.asyncio
async def test_image_returns_base64():
    p = ImageProcessor()
    doc = await p.process(FIXTURE.read_bytes(), "sample.png")
    assert doc.raw_text == ""
    assert doc.images_base64 is not None and len(doc.images_base64) == 1
    assert base64.b64decode(doc.images_base64[0])  # 디코드 가능해야 함
```

- [ ] **Step 19.3: Implement `src/documatch/processors/image.py`**

```python
"""이미지 (JPG/PNG) 프로세서. Vision LLM 입력용."""
import base64
from typing import ClassVar

from documatch.processors.base import UnifiedDocument


class ImageProcessor:
    extensions: ClassVar[tuple[str, ...]] = (".jpg", ".jpeg", ".png")
    requires_vision: ClassVar[bool] = True

    async def process(self, content: bytes, filename: str) -> UnifiedDocument:
        b64 = base64.b64encode(content).decode("ascii")
        return UnifiedDocument(
            filename=filename,
            raw_text="",
            page_count=1,
            is_scanned=False,
            images_base64=[b64],
        )
```

- [ ] **Step 19.4: Run — PASS**

- [ ] **Step 19.5: Commit**

```bash
git add src/documatch/processors/image.py tests/unit/test_processors_image.py tests/fixtures/docs/sample.png
git commit -m "feat(processors): 이미지 프로세서 (base64)"
```

---

### Task 20: PdfProcessor (text + scan branch)

**Files:**
- Create: `src/documatch/processors/pdf.py`
- Create: `tests/fixtures/docs/text.pdf` (small generated text PDF)
- Create: `tests/unit/test_processors_pdf.py`

- [ ] **Step 20.1: Generate text fixture**

```bash
python -c "
from pypdf import PdfWriter
import io
# Use reportlab if available, else create a minimal PDF
try:
    from reportlab.pdfgen import canvas
    c = canvas.Canvas('tests/fixtures/docs/text.pdf')
    c.drawString(100, 750, 'Hello DocuMatch')
    c.drawString(100, 720, '한글 텍스트')
    c.save()
except ImportError:
    # Fallback: write minimal PDF bytes
    pdf = b'%PDF-1.4\n1 0 obj<</Type/Catalog/Pages 2 0 R>>endobj\n2 0 obj<</Type/Pages/Kids[3 0 R]/Count 1>>endobj\n3 0 obj<</Type/Page/Parent 2 0 R/MediaBox[0 0 612 792]/Contents 4 0 R>>endobj\n4 0 obj<</Length 44>>stream\nBT /F1 12 Tf 100 700 Td (Hello DocuMatch) Tj ET\nendstream\nendobj\nxref\n0 5\n0000000000 65535 f\n%%EOF'
    open('tests/fixtures/docs/text.pdf','wb').write(pdf)
"
```

If reportlab missing, run `pip install reportlab` once for fixture generation only (not added to deps).

- [ ] **Step 20.2: Failing test**

```python
from pathlib import Path
from unittest.mock import AsyncMock, patch

import pytest

from documatch.processors.pdf import PdfProcessor

FIXTURE = Path(__file__).parent.parent / "fixtures" / "docs" / "text.pdf"


@pytest.mark.asyncio
async def test_pdf_text_path_no_poppler_needed():
    p = PdfProcessor(scan_threshold=10)
    doc = await p.process(FIXTURE.read_bytes(), "text.pdf")
    assert "Hello" in doc.raw_text or "DocuMatch" in doc.raw_text
    assert doc.is_scanned is False


@pytest.mark.asyncio
async def test_pdf_falls_back_to_scan_when_text_too_short():
    """텍스트 길이가 threshold 미만이면 Poppler 호출 시도."""
    fake_image = type("Img", (), {})()
    with patch("documatch.processors.pdf.convert_from_bytes", return_value=[fake_image]) as conv, \
         patch("documatch.processors.pdf._image_to_base64", return_value="ZmFrZQ=="):
        p = PdfProcessor(scan_threshold=10_000)  # 항상 scan 분기
        doc = await p.process(FIXTURE.read_bytes(), "scan.pdf")
        assert doc.is_scanned is True
        assert doc.images_base64 == ["ZmFrZQ=="]
        conv.assert_called_once()
```

- [ ] **Step 20.3: Implement `src/documatch/processors/pdf.py`**

```python
"""PDF 프로세서. 텍스트 추출 → 부족하면 Poppler로 이미지 변환."""
import base64
import io
from typing import ClassVar

from pypdf import PdfReader

from documatch.exceptions import DependencyError, ProcessorError
from documatch.processors.base import UnifiedDocument

try:
    from pdf2image import convert_from_bytes
except ImportError:
    convert_from_bytes = None  # type: ignore[assignment]


def _image_to_base64(img) -> str:
    buf = io.BytesIO()
    img.save(buf, format="JPEG")
    return base64.b64encode(buf.getvalue()).decode("ascii")


class PdfProcessor:
    extensions: ClassVar[tuple[str, ...]] = (".pdf",)
    requires_vision: ClassVar[bool] = False

    def __init__(self, scan_threshold: int = 100, max_scan_pages: int = 20) -> None:
        self.scan_threshold = scan_threshold
        self.max_scan_pages = max_scan_pages

    def _extract_text(self, content: bytes) -> tuple[str, int]:
        try:
            reader = PdfReader(io.BytesIO(content))
        except Exception as e:
            raise ProcessorError(f"PDF 열기 실패: {e}") from e
        parts: list[str] = []
        for page in reader.pages:
            try:
                parts.append(page.extract_text() or "")
            except Exception:
                parts.append("")
        return "\n".join(parts), len(reader.pages)

    def _scan_to_images(self, content: bytes) -> list[str]:
        if convert_from_bytes is None:
            raise DependencyError("pdf2image이 설치되지 않았습니다.")
        try:
            images = convert_from_bytes(content, last_page=self.max_scan_pages, fmt="jpeg")
        except Exception as e:
            from documatch.i18n import t
            raise DependencyError(t("error.poppler_missing")) from e
        return [_image_to_base64(img) for img in images]

    async def process(self, content: bytes, filename: str) -> UnifiedDocument:
        text, page_count = self._extract_text(content)
        if len(text.strip()) >= self.scan_threshold:
            return UnifiedDocument(
                filename=filename, raw_text=text, page_count=page_count, is_scanned=False,
            )
        images = self._scan_to_images(content)
        return UnifiedDocument(
            filename=filename, raw_text=text, page_count=page_count,
            is_scanned=True, images_base64=images,
        )
```

- [ ] **Step 20.4: Run — PASS**

- [ ] **Step 20.5: Commit**

```bash
git add src/documatch/processors/pdf.py tests/unit/test_processors_pdf.py tests/fixtures/docs/text.pdf
git commit -m "feat(processors): PDF (텍스트 + 스캔 분기)"
```

---

### Task 21: Scanner + registry test pass

**Files:**
- Create: `src/documatch/processors/scanner.py`
- Create: `tests/unit/test_scanner.py`

- [ ] **Step 21.1: Failing test**

```python
from pathlib import Path

import pytest

from documatch.processors.scanner import scan_directory


def test_scan_picks_up_supported_extensions(tmp_path: Path):
    (tmp_path / "a.pdf").write_bytes(b"%PDF-1.4\n%%EOF")
    (tmp_path / "b.docx").write_bytes(b"PK\x03\x04")
    (tmp_path / "c.txt").write_bytes(b"hello")
    (tmp_path / "ignore.exe").write_bytes(b"")
    entries = scan_directory(tmp_path)
    names = sorted(e.filename for e in entries)
    assert names == ["a.pdf", "b.docx", "c.txt"]


def test_scan_recursive(tmp_path: Path):
    sub = tmp_path / "sub"
    sub.mkdir()
    (sub / "a.txt").write_bytes(b"x")
    entries = scan_directory(tmp_path)
    assert any(e.filename == "a.txt" for e in entries)


def test_scan_xlsx_splits_per_sheet(tmp_path: Path):
    from openpyxl import Workbook
    wb = Workbook(); wb.active.title = "S1"; wb.create_sheet("S2")
    p = tmp_path / "x.xlsx"
    wb.save(p)
    entries = scan_directory(tmp_path)
    sheets = sorted(e.sheet_name for e in entries if e.extension == "xlsx")
    assert sheets == ["S1", "S2"]


def test_scan_skips_temp_files(tmp_path: Path):
    (tmp_path / "~$temp.docx").write_bytes(b"")
    entries = scan_directory(tmp_path)
    assert not any(e.filename.startswith("~$") for e in entries)


def test_registry_unknown_extension_raises():
    from documatch.exceptions import UnsupportedFormatError
    from documatch.processors.registry import get_processor
    with pytest.raises(UnsupportedFormatError):
        get_processor("xyz")
```

- [ ] **Step 21.2: Implement `src/documatch/processors/scanner.py`**

```python
"""디렉토리 스캔 → FileEntry 목록."""
from pathlib import Path

from documatch.processors.base import FileEntry
from documatch.processors.registry import supported_extensions
from documatch.processors.xls import XlsProcessor
from documatch.processors.xlsx import XlsxProcessor


def _is_temp_file(name: str) -> bool:
    return name.startswith("~$") or name.startswith(".")


def scan_directory(path: str | Path) -> list[FileEntry]:
    root = Path(path)
    if not root.exists():
        return []
    if root.is_file():
        files = [root]
    else:
        files = sorted(p for p in root.rglob("*") if p.is_file())
    valid_exts = set(supported_extensions())

    entries: list[FileEntry] = []
    for f in files:
        if _is_temp_file(f.name):
            continue
        ext = f.suffix.lower()
        if ext not in valid_exts:
            continue
        size = f.stat().st_size
        if ext == ".xlsx":
            try:
                sheets = XlsxProcessor().list_sheets(f.read_bytes())
            except Exception:
                sheets = []
            for sheet in sheets:
                entries.append(FileEntry(
                    file_path=str(f), filename=f.name, extension="xlsx",
                    size_bytes=size, sheet_name=sheet,
                ))
            continue
        if ext == ".xls":
            try:
                sheets = XlsProcessor().list_sheets(f.read_bytes())
            except Exception:
                sheets = []
            for sheet in sheets:
                entries.append(FileEntry(
                    file_path=str(f), filename=f.name, extension="xls",
                    size_bytes=size, sheet_name=sheet,
                ))
            continue
        entries.append(FileEntry(
            file_path=str(f), filename=f.name, extension=ext.lstrip("."),
            size_bytes=size, sheet_name=None,
        ))
    return entries
```

- [ ] **Step 21.3: Update `src/documatch/processors/__init__.py`**

```python
from documatch.processors.base import DocumentProcessor, FileEntry, UnifiedDocument
from documatch.processors.registry import get_processor, supported_extensions
from documatch.processors.scanner import scan_directory

__all__ = [
    "DocumentProcessor", "FileEntry", "UnifiedDocument",
    "get_processor", "supported_extensions", "scan_directory",
]
```

- [ ] **Step 21.4: Run all processor tests — PASS**

```bash
pytest tests/unit/test_processors_*.py tests/unit/test_scanner.py tests/unit/test_registry.py -v
```

- [ ] **Step 21.5: Commit**

```bash
git add src/documatch/processors/scanner.py src/documatch/processors/__init__.py tests/unit/test_scanner.py
git commit -m "feat(processors): 디렉토리 스캐너 + XLSX/XLS 시트 분리"
```

---

## Phase 6 — Spec Generator (LLM-backed) (Tasks 22-23)

### Task 22: SpecGenerator.generate

**Files:**
- Create: `src/documatch/spec/generator.py`
- Modify: `src/documatch/spec/__init__.py`
- Create: `tests/unit/test_spec_generator.py`

- [ ] **Step 22.1: Failing test**

```python
import json

import pytest

from documatch.processors.base import UnifiedDocument
from documatch.spec.generator import SpecGenerator
from tests.fakes import FakeLLMClient


@pytest.mark.asyncio
async def test_generate_basic():
    response = json.dumps({
        "name": "invoice_extraction",
        "doc_type_hint": "invoice",
        "fields": [
            {"name": "invoice_no", "description": "송장번호", "type": "string", "required": True},
        ],
        "summary_template": {"template": "송장 {invoice_no}"},
        "output_table": [{"field_name": "invoice_no", "excel_header": "송장번호", "width": 15}],
        "table_mode": False,
    })
    llm = FakeLLMClient(responses=[response])
    gen = SpecGenerator(llm=llm)
    doc = UnifiedDocument(filename="x.pdf", raw_text="송장 #123 ...", page_count=1)
    spec = await gen.generate(doc, user_prompt="송장 번호 추출")
    assert spec.name == "invoice_extraction"
    assert spec.fields[0].name == "invoice_no"
    assert spec.table_mode is False


@pytest.mark.asyncio
async def test_generate_passes_images_when_scanned():
    """스캔 PDF인 경우 이미지가 LLM에 함께 전달되어야 한다."""
    llm = FakeLLMClient(responses=[json.dumps({
        "name": "x", "doc_type_hint": "x",
        "fields": [], "summary_template": {"template": "x"}, "output_table": [], "table_mode": False,
    })])
    gen = SpecGenerator(llm=llm)
    doc = UnifiedDocument(
        filename="s.pdf", raw_text="", page_count=1,
        is_scanned=True, images_base64=["BASE64DATA"],
    )
    await gen.generate(doc, user_prompt="x")
    # 첫 호출의 user 메시지에 이미지가 포함되어야 함
    user_msg = next(m for m in llm.calls[0] if m.role == "user")
    assert isinstance(user_msg.content, list)
    has_image = any(b.get("type") == "image" for b in user_msg.content)
    assert has_image
```

- [ ] **Step 22.2: Failing run**

- [ ] **Step 22.3: Implement `src/documatch/spec/generator.py`**

```python
"""LLM 기반 ExtractionSpec 자동 생성·수정."""
from typing import Any

from documatch.exceptions import ExtractionError
from documatch.llm.base import LLMClient, LLMMessage
from documatch.llm.parsing import parse_json_response
from documatch.processors.base import UnifiedDocument
from documatch.spec.models import ExtractionSpec

_GENERATE_SYSTEM = """당신은 문서 추출 스키마 설계 전문가입니다.
사용자가 제공한 샘플 문서와 추출 의도를 바탕으로 ExtractionSpec JSON을 생성하세요.
응답은 반드시 단일 JSON 객체로만, 코드 펜스 없이 출력합니다.

ExtractionSpec 필드:
- name (string): snake_case 식별자
- doc_type_hint (string): 문서 유형 (invoice, contract 등)
- fields (array): {name, description, type, required}
- summary_template (object): {template} — 추출 결과 1줄 요약
- output_table (array): {field_name, excel_header, width}
- table_mode (boolean): 한 문서에서 여러 행을 추출할 경우 true
"""

_REFINE_SYSTEM = """당신은 ExtractionSpec 수정 전문가입니다.
현재 스펙과 사용자 피드백을 바탕으로 수정된 스펙 JSON을 출력합니다.
응답은 단일 JSON 객체로만 출력합니다.
"""


def _build_user_content(doc: UnifiedDocument, prompt: str) -> str | list[dict[str, Any]]:
    if doc.is_scanned and doc.images_base64:
        blocks: list[dict[str, Any]] = [
            {"type": "text", "text": f"사용자 의도: {prompt}\n\n샘플 문서(스캔, 이미지로 제공)"}
        ]
        for b64 in doc.images_base64[:3]:
            blocks.append({
                "type": "image",
                "source": {"type": "base64", "media_type": "image/jpeg", "data": b64},
            })
        return blocks
    text = doc.raw_text[:10000]
    return f"사용자 의도: {prompt}\n\n샘플 문서:\n{text}"


class SpecGenerator:
    def __init__(self, llm: LLMClient) -> None:
        self.llm = llm

    async def generate(self, sample: UnifiedDocument, user_prompt: str) -> ExtractionSpec:
        messages = [
            LLMMessage(role="system", content=_GENERATE_SYSTEM),
            LLMMessage(role="user", content=_build_user_content(sample, user_prompt)),
        ]
        resp = await self.llm.ainvoke(messages, temperature=0.0, max_tokens=4096)
        try:
            data = parse_json_response(resp.content)
        except ExtractionError:
            raise
        data.setdefault("source_sample", sample.filename)
        return ExtractionSpec.model_validate(data)

    async def refine(
        self,
        current_spec: ExtractionSpec,
        feedback: str,
        sample: UnifiedDocument | None = None,
    ) -> ExtractionSpec:
        current_json = current_spec.model_dump_json(indent=2)
        sample_text = ""
        if sample is not None and sample.raw_text:
            sample_text = f"\n\n샘플 문서 발췌:\n{sample.raw_text[:5000]}"
        user = (
            f"현재 스펙:\n{current_json}\n\n"
            f"사용자 피드백:\n{feedback}\n"
            f"{sample_text}\n\n"
            "위 피드백을 반영한 수정된 ExtractionSpec JSON을 출력하세요."
        )
        messages = [
            LLMMessage(role="system", content=_REFINE_SYSTEM),
            LLMMessage(role="user", content=user),
        ]
        resp = await self.llm.ainvoke(messages, temperature=0.0, max_tokens=4096)
        data = parse_json_response(resp.content)
        return ExtractionSpec.model_validate(data)
```

- [ ] **Step 22.4: Update `src/documatch/spec/__init__.py`**

```python
from documatch.spec.generator import SpecGenerator
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, ReviewRule, SummaryTemplate,
)
from documatch.spec.store import SpecInfo, SpecStore

__all__ = [
    "ExtractionSpec", "Field", "FieldType", "OutputColumn",
    "ReviewRule", "SummaryTemplate", "SpecStore", "SpecInfo", "SpecGenerator",
]
```

- [ ] **Step 22.5: Run — PASS**

- [ ] **Step 22.6: Commit**

```bash
git add src/documatch/spec/generator.py src/documatch/spec/__init__.py tests/unit/test_spec_generator.py
git commit -m "feat(spec): SpecGenerator.generate (LLM 기반 자동 생성)"
```

---

### Task 23: SpecGenerator.refine

**Files:**
- Modify: `tests/unit/test_spec_generator.py`

- [ ] **Step 23.1: Append refine test**

```python
@pytest.mark.asyncio
async def test_refine_applies_feedback():
    base = json.dumps({
        "name": "x", "doc_type_hint": "invoice",
        "fields": [{"name": "no", "description": "n", "type": "string", "required": True}],
        "summary_template": {"template": "{no}"},
        "output_table": [{"field_name": "no", "excel_header": "no", "width": 10}],
        "table_mode": False,
    })
    refined = json.dumps({
        "name": "x", "doc_type_hint": "invoice",
        "fields": [
            {"name": "no", "description": "n", "type": "string", "required": True},
            {"name": "amount", "description": "금액", "type": "number", "required": True},
        ],
        "summary_template": {"template": "{no} {amount}원"},
        "output_table": [
            {"field_name": "no", "excel_header": "no", "width": 10},
            {"field_name": "amount", "excel_header": "금액", "width": 10},
        ],
        "table_mode": False,
    })
    from documatch.spec.generator import SpecGenerator
    from documatch.spec.models import ExtractionSpec
    llm = FakeLLMClient(responses=[refined])
    gen = SpecGenerator(llm=llm)
    spec = ExtractionSpec.model_validate_json(base)
    new_spec = await gen.refine(spec, feedback="amount 필드 추가해줘")
    assert any(f.name == "amount" for f in new_spec.fields)
```

- [ ] **Step 23.2: Run — PASS** (refine already implemented in Task 22.3)

- [ ] **Step 23.3: Commit**

```bash
git add tests/unit/test_spec_generator.py
git commit -m "test(spec): SpecGenerator.refine 검증"
```

---

## Phase 7 — Core Models, Single & Batch Extractors (Tasks 24-26)

### Task 24: Core extraction models

**Files:**
- Create: `src/documatch/core/__init__.py`
- Create: `src/documatch/core/models.py`
- Create: `tests/unit/test_core_models.py`

- [ ] **Step 24.1: Failing test**

```python
from documatch.core.models import ExtractionResult


def test_extraction_result_defaults():
    r = ExtractionResult(document_id="x.pdf")
    assert r.values == {}
    assert r.needs_review is False
    assert r.status == "success"
    assert r.is_table_result is False


def test_table_result_detection():
    r = ExtractionResult(
        document_id="x.pdf",
        rows=[{"a": 1}, {"a": 2}],
        row_confidences=[{"a": 0.9}, {"a": 0.8}],
    )
    assert r.is_table_result is True
    assert r.row_count == 2
```

- [ ] **Step 24.2: Implement `src/documatch/core/models.py`**

```python
"""추출 결과 도메인 모델."""
from dataclasses import dataclass, field
from typing import Any, Literal


@dataclass
class ExtractionResult:
    document_id: str
    values: dict[str, Any] = field(default_factory=dict)
    confidence: dict[str, float] = field(default_factory=dict)
    summary_sentence: str = ""
    needs_review: bool = False
    review_reasons: list[str] = field(default_factory=list)
    status: Literal["success", "partial", "failed"] = "success"
    error_message: str | None = None
    rows: list[dict[str, Any]] | None = None
    row_confidences: list[dict[str, float]] | None = None

    @property
    def is_table_result(self) -> bool:
        return self.rows is not None and len(self.rows) > 0

    @property
    def row_count(self) -> int:
        return len(self.rows) if self.rows else 0
```

- [ ] **Step 24.3: Implement `src/documatch/core/__init__.py`**

```python
from documatch.core.models import ExtractionResult

__all__ = ["ExtractionResult"]
```

- [ ] **Step 24.4: Run — PASS**

- [ ] **Step 24.5: Commit**

```bash
git add src/documatch/core tests/unit/test_core_models.py
git commit -m "feat(core): ExtractionResult 모델"
```

---

### Task 25: SingleExtractor (샘플 검토용)

**Files:**
- Create: `src/documatch/extractors/__init__.py`
- Create: `src/documatch/extractors/prompts.py`
- Create: `src/documatch/extractors/single.py`
- Create: `tests/unit/test_extractors_single.py`

- [ ] **Step 25.1: Failing test**

```python
import json

import pytest

from documatch.extractors.single import SingleExtractor
from documatch.processors.base import UnifiedDocument
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)
from tests.fakes import FakeLLMClient


def _spec() -> ExtractionSpec:
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


@pytest.mark.asyncio
async def test_extracts_values_and_confidence():
    response = json.dumps({
        "values": {"a": "hello"}, "confidence": {"a": 0.95},
        "summary_sentence": "hello"
    })
    llm = FakeLLMClient(responses=[response])
    ex = SingleExtractor(spec=_spec(), llm=llm)
    doc = UnifiedDocument(filename="x.pdf", raw_text="hello world", page_count=1)
    result = await ex.extract(doc, document_id="x.pdf")
    assert result.values["a"] == "hello"
    assert result.confidence["a"] == 0.95
    assert result.status == "success"


@pytest.mark.asyncio
async def test_low_confidence_marks_review():
    response = json.dumps({
        "values": {"a": "x"}, "confidence": {"a": 0.4},
        "summary_sentence": ""
    })
    llm = FakeLLMClient(responses=[response])
    ex = SingleExtractor(spec=_spec(), llm=llm)
    doc = UnifiedDocument(filename="x.pdf", raw_text="x", page_count=1)
    result = await ex.extract(doc, document_id="x.pdf")
    assert result.needs_review is True
    assert any("Low confidence" in r for r in result.review_reasons)


@pytest.mark.asyncio
async def test_table_mode_parses_rows():
    spec = _spec()
    spec.table_mode = True
    response = json.dumps({
        "rows": [{"a": "x1"}, {"a": "x2"}],
        "row_confidences": [{"a": 0.9}, {"a": 0.8}],
        "summary_sentence": "2 rows"
    })
    llm = FakeLLMClient(responses=[response])
    ex = SingleExtractor(spec=spec, llm=llm)
    doc = UnifiedDocument(filename="x.pdf", raw_text="...", page_count=1)
    result = await ex.extract(doc, document_id="x.pdf")
    assert result.row_count == 2
    assert result.is_table_result is True
```

- [ ] **Step 25.2: Implement `src/documatch/extractors/prompts.py`**

```python
"""LLM 프롬프트 빌더 (system + user)."""
import json
from typing import Any

from documatch.processors.base import UnifiedDocument
from documatch.spec.models import ExtractionSpec


def build_system_prompt(spec: ExtractionSpec) -> str:
    fields_desc = "\n".join(
        f"- {f.name} ({f.type.value}, {'필수' if f.required else '선택'}): {f.description}"
        for f in spec.fields
    )
    if spec.table_mode:
        format_block = (
            "응답 JSON 형식:\n"
            "{\n"
            '  "rows": [{"필드명": "값", ...}, ...],\n'
            '  "row_confidences": [{"필드명": 0.0~1.0, ...}, ...],\n'
            '  "summary_sentence": "요약"\n'
            "}"
        )
    else:
        format_block = (
            "응답 JSON 형식:\n"
            "{\n"
            '  "values": {"필드명": "값", ...},\n'
            '  "confidence": {"필드명": 0.0~1.0, ...},\n'
            '  "summary_sentence": "요약"\n'
            "}"
        )
    return (
        f"당신은 문서 추출 전문가입니다. 다음 스키마에 따라 문서에서 정보를 추출합니다.\n\n"
        f"문서 유형: {spec.doc_type_hint}\n"
        f"필드:\n{fields_desc}\n\n"
        f"{format_block}\n"
        "JSON만 출력합니다."
    )


def build_user_content(doc: UnifiedDocument) -> str | list[dict[str, Any]]:
    if doc.is_scanned and doc.images_base64:
        blocks: list[dict[str, Any]] = [
            {"type": "text", "text": f"문서 (스캔 이미지): {doc.filename}"}
        ]
        for b64 in doc.images_base64[:3]:
            blocks.append({
                "type": "image",
                "source": {"type": "base64", "media_type": "image/jpeg", "data": b64},
            })
        return blocks
    text = doc.raw_text[:15000]
    return f"문서: {doc.filename}\n\n{text}"
```

- [ ] **Step 25.3: Implement `src/documatch/extractors/single.py`**

```python
"""SingleExtractor — 1건 추출."""
from documatch.core.models import ExtractionResult
from documatch.exceptions import ExtractionError
from documatch.extractors.prompts import build_system_prompt, build_user_content
from documatch.llm.base import LLMClient, LLMMessage
from documatch.llm.parsing import parse_json_response
from documatch.processors.base import UnifiedDocument
from documatch.spec.models import ExtractionSpec


def _apply_review_rules(spec: ExtractionSpec, result: ExtractionResult) -> None:
    for rule in spec.review_rules:
        if rule.condition == "low_confidence" and rule.threshold is not None:
            if result.is_table_result:
                for r_idx, rc in enumerate(result.row_confidences or []):
                    for fname, conf in rc.items():
                        if conf < rule.threshold:
                            result.needs_review = True
                            result.review_reasons.append(
                                f"Low confidence for {fname} in row {r_idx + 1}: {conf}"
                            )
            else:
                for fname, conf in result.confidence.items():
                    if conf < rule.threshold:
                        result.needs_review = True
                        result.review_reasons.append(f"Low confidence for {fname}: {conf}")
        if rule.condition == "missing_required":
            for f in spec.fields:
                if f.required and not result.values.get(f.name):
                    if not result.is_table_result:
                        result.needs_review = True
                        result.review_reasons.append(f"Missing required field: {f.name}")


class SingleExtractor:
    def __init__(self, spec: ExtractionSpec, llm: LLMClient) -> None:
        self.spec = spec
        self.llm = llm

    async def extract(self, doc: UnifiedDocument, *, document_id: str) -> ExtractionResult:
        messages = [
            LLMMessage(role="system", content=build_system_prompt(self.spec)),
            LLMMessage(role="user", content=build_user_content(doc)),
        ]
        try:
            resp = await self.llm.ainvoke(messages, temperature=0.0, max_tokens=4096)
            data = parse_json_response(resp.content)
        except ExtractionError as e:
            return ExtractionResult(
                document_id=document_id,
                status="partial",
                needs_review=True,
                review_reasons=[f"파싱 실패: {e}"],
                error_message=str(e),
            )

        if self.spec.table_mode and "rows" in data:
            field_names = [f.name for f in self.spec.fields]
            rows = [{fn: r.get(fn) for fn in field_names} for r in data.get("rows", [])]
            confs = data.get("row_confidences", [])
            row_confidences = []
            for i in range(len(rows)):
                rc = confs[i] if i < len(confs) else {}
                row_confidences.append({fn: float(rc.get(fn, 0.0)) for fn in field_names})
            result = ExtractionResult(
                document_id=document_id,
                values=dict(rows[0]) if rows else {},
                confidence=dict(row_confidences[0]) if row_confidences else {},
                summary_sentence=data.get("summary_sentence", ""),
                rows=rows,
                row_confidences=row_confidences,
                status="success",
            )
        else:
            result = ExtractionResult(
                document_id=document_id,
                values=data.get("values", {}),
                confidence={k: float(v) for k, v in data.get("confidence", {}).items()},
                summary_sentence=data.get("summary_sentence", ""),
                status="success",
            )
        _apply_review_rules(self.spec, result)
        return result
```

- [ ] **Step 25.4: Implement `src/documatch/extractors/__init__.py`**

```python
from documatch.extractors.single import SingleExtractor

__all__ = ["SingleExtractor"]
```

- [ ] **Step 25.5: Run — PASS**

- [ ] **Step 25.6: Commit**

```bash
git add src/documatch/extractors tests/unit/test_extractors_single.py
git commit -m "feat(extractors): SingleExtractor + 프롬프트 빌더"
```

---

### Task 26: BatchExtractor (async generator + 진행률)

**Files:**
- Create: `src/documatch/extractors/batch.py`
- Modify: `src/documatch/extractors/__init__.py`
- Create: `tests/unit/test_extractors_batch.py`

- [ ] **Step 26.1: Failing test**

```python
import json

import pytest

from documatch.extractors.batch import BatchExtractor
from documatch.processors.base import FileEntry
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)
from tests.fakes import FakeLLMClient


def _spec() -> ExtractionSpec:
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


@pytest.mark.asyncio
async def test_batch_yields_progress(tmp_path):
    f1 = tmp_path / "a.txt"; f1.write_text("hi")
    f2 = tmp_path / "b.txt"; f2.write_text("bye")
    response = json.dumps({"values": {"a": "x"}, "confidence": {"a": 0.9}, "summary_sentence": "x"})
    llm = FakeLLMClient(responses=[response, response])
    extractor = BatchExtractor(spec=_spec(), llm=llm)
    entries = [
        FileEntry(file_path=str(f1), filename="a.txt", extension="txt", size_bytes=2),
        FileEntry(file_path=str(f2), filename="b.txt", extension="txt", size_bytes=3),
    ]
    progress_calls = 0
    results = []
    async for r in extractor.extract_all(entries):
        progress_calls += 1
        results.append(r)
    assert progress_calls == 2
    assert all(r.status == "success" for r in results)


@pytest.mark.asyncio
async def test_batch_continues_on_individual_failure(tmp_path):
    f1 = tmp_path / "good.txt"; f1.write_text("ok")
    f2 = tmp_path / "bad.txt"; f2.write_text("ok")
    good = json.dumps({"values": {"a": "x"}, "confidence": {"a": 0.9}, "summary_sentence": "x"})
    llm = FakeLLMClient(responses=[good, "garbage non-json"])
    extractor = BatchExtractor(spec=_spec(), llm=llm)
    entries = [
        FileEntry(file_path=str(f1), filename="good.txt", extension="txt", size_bytes=2),
        FileEntry(file_path=str(f2), filename="bad.txt", extension="txt", size_bytes=2),
    ]
    statuses = [r.status async for r in extractor.extract_all(entries)]
    assert statuses[0] == "success"
    assert statuses[1] in ("partial", "failed")
```

- [ ] **Step 26.2: Implement `src/documatch/extractors/batch.py`**

```python
"""BatchExtractor — N개 문서를 async generator로 순회 처리."""
from collections.abc import AsyncIterator
from pathlib import Path

from documatch.core.models import ExtractionResult
from documatch.exceptions import ProcessorError, UnsupportedFormatError
from documatch.extractors.single import SingleExtractor
from documatch.llm.base import LLMClient
from documatch.processors.base import FileEntry
from documatch.processors.registry import get_processor
from documatch.processors.xls import XlsProcessor
from documatch.processors.xlsx import XlsxProcessor
from documatch.spec.models import ExtractionSpec


class BatchExtractor:
    def __init__(self, spec: ExtractionSpec, llm: LLMClient) -> None:
        self.spec = spec
        self.llm = llm
        self._single = SingleExtractor(spec=spec, llm=llm)

    async def _load(self, entry: FileEntry):
        path = Path(entry.file_path)
        content = path.read_bytes()
        ext = entry.extension.lower()
        if entry.sheet_name and ext == "xlsx":
            return await XlsxProcessor().process_sheet(content, entry.sheet_name)
        if entry.sheet_name and ext == "xls":
            return await XlsProcessor().process_sheet(content, entry.sheet_name)
        proc = get_processor(ext)
        return await proc.process(content, entry.filename)

    async def extract_all(self, entries: list[FileEntry]) -> AsyncIterator[ExtractionResult]:
        for entry in entries:
            doc_id = (
                f"{entry.file_path}#{entry.sheet_name}" if entry.sheet_name else entry.file_path
            )
            try:
                doc = await self._load(entry)
            except (ProcessorError, UnsupportedFormatError) as e:
                yield ExtractionResult(
                    document_id=doc_id, status="failed",
                    needs_review=True, review_reasons=[str(e)], error_message=str(e),
                )
                continue
            try:
                result = await self._single.extract(doc, document_id=doc_id)
            except Exception as e:
                yield ExtractionResult(
                    document_id=doc_id, status="failed",
                    needs_review=True, review_reasons=[str(e)], error_message=str(e),
                )
                continue
            yield result
```

- [ ] **Step 26.3: Update `src/documatch/extractors/__init__.py`**

```python
from documatch.extractors.batch import BatchExtractor
from documatch.extractors.single import SingleExtractor

__all__ = ["SingleExtractor", "BatchExtractor"]
```

- [ ] **Step 26.4: Run — PASS**

- [ ] **Step 26.5: Commit**

```bash
git add src/documatch/extractors tests/unit/test_extractors_batch.py
git commit -m "feat(extractors): BatchExtractor (async generator + 부분 실패 격리)"
```

---

## Phase 8 — Excel Exporter (Tasks 27-28)

### Task 27: ExcelExporter (basic)

**Files:**
- Create: `src/documatch/exporters/__init__.py`
- Create: `src/documatch/exporters/excel.py`
- Create: `tests/unit/test_excel_exporter.py`

- [ ] **Step 27.1: Failing test**

```python
from pathlib import Path

from openpyxl import load_workbook

from documatch.core.models import ExtractionResult
from documatch.exporters.excel import ExcelExporter
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)


def _spec() -> ExtractionSpec:
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[
            Field(name="a", description="d", type=FieldType.STRING),
            Field(name="b", description="d", type=FieldType.NUMBER),
        ],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[
            OutputColumn(field_name="a", excel_header="A 헤더", width=12),
            OutputColumn(field_name="b", excel_header="B 헤더", width=10),
        ],
    )


def test_writes_results_to_excel(tmp_path: Path):
    spec = _spec()
    results = [
        ExtractionResult(document_id="x.pdf", values={"a": "v1", "b": 100}, summary_sentence="요약"),
        ExtractionResult(document_id="y.pdf", values={"a": "v2", "b": 200}),
    ]
    out = tmp_path / "out.xlsx"
    ExcelExporter().write(spec, results, out)
    wb = load_workbook(out)
    ws = wb["results"]
    headers = [c.value for c in ws[1]]
    assert "A 헤더" in headers and "B 헤더" in headers
    row2 = [c.value for c in ws[2]]
    assert "v1" in row2


def test_failed_documents_separate_sheet(tmp_path: Path):
    spec = _spec()
    results = [
        ExtractionResult(document_id="ok.pdf", values={"a": "x", "b": 1}),
        ExtractionResult(document_id="bad.pdf", status="failed", error_message="boom"),
    ]
    out = tmp_path / "out.xlsx"
    ExcelExporter().write(spec, results, out)
    wb = load_workbook(out)
    assert "failed_documents" in wb.sheetnames
    failed = wb["failed_documents"]
    assert any("bad.pdf" in (c.value or "") for c in failed["A"])


def test_table_mode_expands_rows(tmp_path: Path):
    spec = _spec()
    spec.table_mode = True
    results = [ExtractionResult(
        document_id="t.pdf",
        rows=[{"a": "x1", "b": 1}, {"a": "x2", "b": 2}],
    )]
    out = tmp_path / "out.xlsx"
    ExcelExporter().write(spec, results, out)
    wb = load_workbook(out)
    ws = wb["results"]
    # header + 2 rows
    assert ws.max_row == 3
```

- [ ] **Step 27.2: Implement `src/documatch/exporters/excel.py`**

```python
"""ExcelExporter — 추출 결과 → .xlsx."""
from pathlib import Path

from openpyxl import Workbook
from openpyxl.styles import Font

from documatch.core.models import ExtractionResult
from documatch.spec.models import ExtractionSpec

_HEADER_FONT = Font(bold=True)


class ExcelExporter:
    def write(
        self,
        spec: ExtractionSpec,
        results: list[ExtractionResult],
        output_path: Path,
    ) -> Path:
        wb = Workbook()
        ws = wb.active
        ws.title = "results"

        headers = ["문서"] + [c.excel_header for c in spec.output_table]
        if not spec.table_mode:
            headers.append("요약")
        headers.append("검토필요")
        ws.append(headers)
        for cell in ws[1]:
            cell.font = _HEADER_FONT

        for col_idx, col in enumerate(spec.output_table, start=2):
            ws.column_dimensions[ws.cell(row=1, column=col_idx).column_letter].width = col.width

        successful = [r for r in results if r.status != "failed"]
        failed = [r for r in results if r.status == "failed"]

        for r in successful:
            if spec.table_mode and r.is_table_result:
                for row in r.rows or []:
                    line = [r.document_id]
                    for col in spec.output_table:
                        line.append(row.get(col.field_name, ""))
                    line.append("Y" if r.needs_review else "")
                    ws.append(line)
            else:
                line = [r.document_id]
                for col in spec.output_table:
                    line.append(r.values.get(col.field_name, ""))
                if not spec.table_mode:
                    line.append(r.summary_sentence)
                line.append("Y" if r.needs_review else "")
                ws.append(line)

        if failed:
            failed_ws = wb.create_sheet("failed_documents")
            failed_ws.append(["문서", "에러"])
            for cell in failed_ws[1]:
                cell.font = _HEADER_FONT
            for r in failed:
                failed_ws.append([r.document_id, r.error_message or ""])

        output_path.parent.mkdir(parents=True, exist_ok=True)
        wb.save(output_path)
        return output_path
```

- [ ] **Step 27.3: Implement `src/documatch/exporters/__init__.py`**

```python
from documatch.exporters.excel import ExcelExporter

__all__ = ["ExcelExporter"]
```

- [ ] **Step 27.4: Run — PASS**

- [ ] **Step 27.5: Commit**

```bash
git add src/documatch/exporters tests/unit/test_excel_exporter.py
git commit -m "feat(exporters): ExcelExporter (results + failed_documents 시트)"
```

---

### Task 28: Hyperlinks fix utility

**Files:**
- Create: `src/documatch/exporters/hyperlinks.py`
- Create: `tests/unit/test_hyperlinks.py`

- [ ] **Step 28.1: Failing test**

```python
from pathlib import Path

from openpyxl import Workbook, load_workbook

from documatch.exporters.hyperlinks import fix_hyperlinks


def test_converts_absolute_to_relative(tmp_path: Path):
    target_file = tmp_path / "doc.pdf"
    target_file.write_bytes(b"%PDF")
    xlsx = tmp_path / "out.xlsx"
    wb = Workbook()
    ws = wb.active
    ws.cell(row=1, column=1, value="link").hyperlink = str(target_file.resolve())
    wb.save(xlsx)

    fixed = fix_hyperlinks(xlsx)
    assert fixed == 1
    wb2 = load_workbook(xlsx)
    cell = wb2.active.cell(row=1, column=1)
    assert cell.hyperlink.target == "doc.pdf"


def test_skips_url(tmp_path: Path):
    xlsx = tmp_path / "out.xlsx"
    wb = Workbook()
    ws = wb.active
    ws.cell(row=1, column=1, value="x").hyperlink = "https://example.com"
    wb.save(xlsx)
    assert fix_hyperlinks(xlsx) == 0
```

- [ ] **Step 28.2: Implement `src/documatch/exporters/hyperlinks.py`**

```python
"""Excel 절대경로 하이퍼링크 → 상대경로 변환."""
import os
from pathlib import Path

from openpyxl import load_workbook


def fix_hyperlinks(excel_path: Path, *, dry_run: bool = False) -> int:
    excel_path = Path(excel_path)
    excel_dir = str(excel_path.parent.resolve())
    wb = load_workbook(str(excel_path))
    fixed = 0
    try:
        for ws in wb.worksheets:
            for row in ws.iter_rows():
                for cell in row:
                    if cell.hyperlink and cell.hyperlink.target:
                        target = cell.hyperlink.target
                        if not os.path.isabs(target):
                            continue
                        if target.startswith(("http://", "https://")):
                            continue
                        if not os.path.exists(target):
                            continue
                        rel = os.path.relpath(target, excel_dir)
                        if not dry_run:
                            cell.hyperlink.target = rel
                        fixed += 1
        if not dry_run and fixed > 0:
            wb.save(str(excel_path))
    finally:
        wb.close()
    return fixed
```

- [ ] **Step 28.3: Run — PASS**

- [ ] **Step 28.4: Commit**

```bash
git add src/documatch/exporters/hyperlinks.py tests/unit/test_hyperlinks.py
git commit -m "feat(exporters): 하이퍼링크 절대→상대 변환 유틸"
```

---

## Phase 9 — HITL Reviewers (Tasks 29-31)

### Task 29: Reviewer Protocol + decision types

**Files:**
- Create: `src/documatch/interaction/__init__.py`
- Create: `src/documatch/interaction/reviewer.py`
- Create: `tests/unit/test_reviewer_protocol.py`

- [ ] **Step 29.1: Implement `src/documatch/interaction/reviewer.py`**

```python
"""Reviewer Protocol + 결정 dataclass."""
from dataclasses import dataclass
from pathlib import Path
from typing import Literal, Protocol

from documatch.core.models import ExtractionResult
from documatch.processors.base import FileEntry
from documatch.spec.models import ExtractionSpec


@dataclass
class ReviewDecision:
    action: Literal["continue", "filter", "quit"]
    extension_filter: list[str] | None = None


@dataclass
class SpecDecision:
    action: Literal["approve", "refine", "load", "back", "quit"]
    feedback: str | None = None
    spec_name: str | None = None


@dataclass
class ModeDecision:
    action: Literal["single", "table", "back"]


@dataclass
class SampleDecision:
    action: Literal["full", "single_save", "refine", "back", "quit"]
    feedback: str | None = None


class Reviewer(Protocol):
    def confirm_files(self, entries: list[FileEntry]) -> ReviewDecision: ...
    def select_sample(self, entries: list[FileEntry]) -> Path | None: ...
    def get_user_prompt(self, default: str | None = None) -> str: ...
    def review_spec(
        self, spec: ExtractionSpec, available_specs: list[str] | None = None,
    ) -> SpecDecision: ...
    def confirm_mode(self, spec: ExtractionSpec) -> ModeDecision: ...
    def review_sample(
        self, result: ExtractionResult, spec: ExtractionSpec,
    ) -> SampleDecision: ...
    def get_save_path(self, default: Path) -> Path: ...
    def confirm_save_spec(self, spec: ExtractionSpec) -> str | None: ...
    def show_progress(self, current: int, total: int, label: str) -> None: ...
    def show_message(self, level: Literal["info", "warn", "error"], text: str) -> None: ...
```

- [ ] **Step 29.2: Implement `src/documatch/interaction/__init__.py`**

```python
from documatch.interaction.reviewer import (
    ModeDecision, ReviewDecision, Reviewer, SampleDecision, SpecDecision,
)

__all__ = [
    "Reviewer", "ReviewDecision", "SpecDecision", "ModeDecision", "SampleDecision",
]
```

- [ ] **Step 29.3: Smoke test**

```python
from documatch.interaction.reviewer import (
    ModeDecision, ReviewDecision, SampleDecision, SpecDecision,
)


def test_decisions_constructible():
    assert ReviewDecision(action="continue").action == "continue"
    assert SpecDecision(action="approve").feedback is None
    assert ModeDecision(action="single").action == "single"
    assert SampleDecision(action="full").action == "full"
```

- [ ] **Step 29.4: Commit**

```bash
git add src/documatch/interaction tests/unit/test_reviewer_protocol.py
git commit -m "feat(interaction): Reviewer Protocol + 결정 타입"
```

---

### Task 30: AutomationReviewer

**Files:**
- Create: `src/documatch/interaction/automation.py`
- Modify: `src/documatch/interaction/__init__.py`
- Create: `tests/unit/test_reviewer_automation.py`

- [ ] **Step 30.1: Failing test**

```python
from pathlib import Path

import pytest

from documatch.exceptions import UserAbort
from documatch.interaction.automation import AutomationReviewer
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


def test_confirm_files_continues():
    r = AutomationReviewer(output_path=Path("out.xlsx"))
    decision = r.confirm_files([
        FileEntry(file_path="/x.txt", filename="x.txt", extension="txt", size_bytes=1),
    ])
    assert decision.action == "continue"


def test_review_spec_approves():
    r = AutomationReviewer(output_path=Path("out.xlsx"))
    assert r.review_spec(_spec()).action == "approve"


def test_get_save_path_uses_configured():
    r = AutomationReviewer(output_path=Path("/tmp/result.xlsx"))
    assert r.get_save_path(Path("/tmp/default.xlsx")) == Path("/tmp/result.xlsx")


def test_select_sample_raises_when_unsupported():
    """자동화 모드는 spec이 사전 제공되어야 하므로 select_sample은 안 불려야 한다."""
    r = AutomationReviewer(output_path=Path("out.xlsx"))
    with pytest.raises(UserAbort):
        r.select_sample([])


def test_confirm_save_spec_returns_none():
    r = AutomationReviewer(output_path=Path("out.xlsx"))
    assert r.confirm_save_spec(_spec()) is None
```

- [ ] **Step 30.2: Implement `src/documatch/interaction/automation.py`**

```python
"""AutomationReviewer — 모든 결정을 자동 승인."""
from pathlib import Path
from typing import Literal

from documatch.core.models import ExtractionResult
from documatch.exceptions import UserAbort
from documatch.interaction.reviewer import (
    ModeDecision, ReviewDecision, SampleDecision, SpecDecision,
)
from documatch.processors.base import FileEntry
from documatch.spec.models import ExtractionSpec


class AutomationReviewer:
    def __init__(self, *, output_path: Path) -> None:
        self.output_path = output_path

    def confirm_files(self, entries: list[FileEntry]) -> ReviewDecision:
        return ReviewDecision(action="continue")

    def select_sample(self, entries: list[FileEntry]) -> Path | None:
        raise UserAbort(
            "자동화 모드에서는 LLM 기반 스펙 생성을 허용하지 않습니다. "
            "--spec 또는 --spec-file을 사용하세요."
        )

    def get_user_prompt(self, default: str | None = None) -> str:
        return default or ""

    def review_spec(
        self, spec: ExtractionSpec, available_specs: list[str] | None = None,
    ) -> SpecDecision:
        return SpecDecision(action="approve")

    def confirm_mode(self, spec: ExtractionSpec) -> ModeDecision:
        return ModeDecision(action="table" if spec.table_mode else "single")

    def review_sample(
        self, result: ExtractionResult, spec: ExtractionSpec,
    ) -> SampleDecision:
        return SampleDecision(action="full")

    def get_save_path(self, default: Path) -> Path:
        return self.output_path

    def confirm_save_spec(self, spec: ExtractionSpec) -> str | None:
        return None

    def show_progress(self, current: int, total: int, label: str) -> None:
        pass

    def show_message(self, level: Literal["info", "warn", "error"], text: str) -> None:
        import sys
        stream = sys.stderr if level in ("warn", "error") else sys.stdout
        print(f"[{level}] {text}", file=stream)
```

- [ ] **Step 30.3: Update `src/documatch/interaction/__init__.py`**

```python
from documatch.interaction.automation import AutomationReviewer
from documatch.interaction.reviewer import (
    ModeDecision, ReviewDecision, Reviewer, SampleDecision, SpecDecision,
)

__all__ = [
    "Reviewer", "ReviewDecision", "SpecDecision", "ModeDecision", "SampleDecision",
    "AutomationReviewer",
]
```

- [ ] **Step 30.4: Run — PASS**

- [ ] **Step 30.5: Commit**

```bash
git add src/documatch/interaction/automation.py src/documatch/interaction/__init__.py tests/unit/test_reviewer_automation.py
git commit -m "feat(interaction): AutomationReviewer (자동 승인)"
```

---

### Task 31: InteractiveReviewer (Rich)

**Files:**
- Create: `src/documatch/interaction/prompts.py`
- Create: `src/documatch/interaction/interactive.py`
- Modify: `src/documatch/interaction/__init__.py`
- Create: `tests/unit/test_reviewer_interactive.py`

- [ ] **Step 31.1: Implement `src/documatch/interaction/prompts.py`**

```python
"""Rich 기반 입력 헬퍼."""
from rich.console import Console
from rich.prompt import Prompt


def multiline_input(console: Console, label: str, default: str | None = None) -> str:
    if default:
        console.print(f"[dim]기본값: {default}[/dim]")
        console.print("[dim]기본값을 사용하려면 바로 Enter를 누르세요.[/dim]")
    console.print("[dim]여러 줄 입력 가능. 빈 줄에서 Enter로 종료.[/dim]")
    lines: list[str] = []
    while True:
        try:
            line = input("  > " if lines else f"{label}: ")
        except EOFError:
            break
        if not line.strip():
            break
        lines.append(line)
    text = "\n".join(lines).strip()
    if not text and default:
        return default
    return text


def choice(console: Console, label: str, options: list[str], default: str) -> str:
    return Prompt.ask(label, choices=options + [o.upper() for o in options], default=default).lower()
```

- [ ] **Step 31.2: Implement `src/documatch/interaction/interactive.py`**

```python
"""InteractiveReviewer — Rich 기반 6개 HITL 게이트."""
from datetime import datetime
from pathlib import Path
from typing import Literal

from rich.console import Console
from rich.panel import Panel
from rich.prompt import Prompt
from rich.table import Table

from documatch.core.models import ExtractionResult
from documatch.exceptions import UserAbort
from documatch.interaction.prompts import choice, multiline_input
from documatch.interaction.reviewer import (
    ModeDecision, ReviewDecision, SampleDecision, SpecDecision,
)
from documatch.processors.base import FileEntry
from documatch.spec.models import ExtractionSpec


class InteractiveReviewer:
    def __init__(self, console: Console | None = None) -> None:
        self.console = console or Console()

    def confirm_files(self, entries: list[FileEntry]) -> ReviewDecision:
        ext_counts: dict[str, int] = {}
        for e in entries:
            ext_counts[e.extension] = ext_counts.get(e.extension, 0) + 1
        self.console.print()
        self.console.print(f"[bold]{len(entries)}개 파일/시트 발견[/bold]")
        for ext, n in sorted(ext_counts.items()):
            self.console.print(f"  .{ext}: {n}")
        c = choice(self.console, "[A]계속 / [E]필터 / [Q]종료", ["a", "e", "q"], "a")
        if c == "q":
            return ReviewDecision(action="quit")
        if c == "e":
            text = Prompt.ask("포함할 확장자 (쉼표 구분, 예: pdf,docx)")
            exts = [s.strip().lower() for s in text.split(",") if s.strip()]
            return ReviewDecision(action="filter", extension_filter=exts)
        return ReviewDecision(action="continue")

    def select_sample(self, entries: list[FileEntry]) -> Path | None:
        non_xlsx = [e for e in entries if e.extension in ("pdf", "docx", "txt", "md", "html", "htm")]
        candidates = non_xlsx if non_xlsx else entries
        self.console.print("[bold]스키마 생성용 샘플 선택[/bold]")
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

    def get_user_prompt(self, default: str | None = None) -> str:
        return multiline_input(self.console, "추출 의도 입력", default=default)

    def review_spec(
        self, spec: ExtractionSpec, available_specs: list[str] | None = None,
    ) -> SpecDecision:
        self._render_spec(spec)
        self.console.print()
        self.console.print("[A]승인 / [R]피드백 수정 / [L]저장된 스펙 / [B]샘플 재선택 / [Q]종료")
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
                return self.review_spec(spec, available_specs)
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

    def confirm_mode(self, spec: ExtractionSpec) -> ModeDecision:
        rec = "테이블" if spec.table_mode else "단일값"
        self.console.print(f"[bold]AI 추천 모드: {rec}[/bold]")
        c = choice(self.console, "[S]단일값 / [T]테이블 / [B]돌아가기", ["s", "t", "b"], "t" if spec.table_mode else "s")
        if c == "b":
            return ModeDecision(action="back")
        return ModeDecision(action="table" if c == "t" else "single")

    def review_sample(
        self, result: ExtractionResult, spec: ExtractionSpec,
    ) -> SampleDecision:
        self._render_result(result, spec)
        self.console.print()
        self.console.print("[A]전체 진행 / [S]이 건만 저장 / [R]피드백 수정 / [B]스키마 수정 / [Q]종료")
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

    def get_save_path(self, default: Path) -> Path:
        ans = Prompt.ask("저장 경로", default=str(default))
        path = Path(ans)
        if path.is_dir() or (not path.suffix):
            path = path / default.name
        return path

    def confirm_save_spec(self, spec: ExtractionSpec) -> str | None:
        ans = Prompt.ask("스펙을 저장할까요? (y/N)", choices=["y", "n", "Y", "N"], default="n")
        if ans.lower() != "y":
            return None
        suggested = f"{spec.doc_type_hint}_{datetime.utcnow().strftime('%Y%m%d_%H%M%S')}"
        return Prompt.ask("스펙 이름", default=suggested)

    def show_progress(self, current: int, total: int, label: str) -> None:
        self.console.print(f"  [{current}/{total}] {label}")

    def show_message(self, level: Literal["info", "warn", "error"], text: str) -> None:
        color = {"info": "cyan", "warn": "yellow", "error": "red"}[level]
        self.console.print(f"[{color}]{text}[/{color}]")

    def _render_spec(self, spec: ExtractionSpec) -> None:
        self.console.print()
        self.console.print(Panel.fit(f"[bold]스키마: {spec.name}[/bold]", border_style="cyan"))
        t = Table(show_header=True, header_style="bold magenta")
        t.add_column("#"); t.add_column("필드"); t.add_column("타입"); t.add_column("필수"); t.add_column("설명")
        for i, f in enumerate(spec.fields, 1):
            t.add_row(str(i), f.name, f.type.value, "OK" if f.required else "", f.description[:40])
        self.console.print(t)
        self.console.print(f"[dim]요약 템플릿: {spec.summary_template.template}[/dim]")

    def _render_result(self, r: ExtractionResult, spec: ExtractionSpec) -> None:
        self.console.print()
        if r.is_table_result:
            self.console.print(f"[bold green]테이블 결과 ({r.row_count}행)[/bold green]")
            t = Table(show_header=True)
            t.add_column("#")
            for f in spec.fields:
                t.add_column(f.name)
            for i, row in enumerate(r.rows or [], 1):
                t.add_row(str(i), *[str(row.get(f.name, "")) for f in spec.fields])
            self.console.print(t)
        else:
            self.console.print("[bold green]추출 결과[/bold green]")
            for f in spec.fields:
                v = r.values.get(f.name, "")
                c = r.confidence.get(f.name, 0.0)
                self.console.print(f"  {f.name}: {v}  [dim]({c:.2f})[/dim]")
            if r.summary_sentence:
                self.console.print(f"  요약: {r.summary_sentence}")
        if r.needs_review:
            self.console.print("[yellow]검토 필요:[/yellow]")
            for reason in r.review_reasons:
                self.console.print(f"  - {reason}")
```

- [ ] **Step 31.3: Update `src/documatch/interaction/__init__.py`**

```python
from documatch.interaction.automation import AutomationReviewer
from documatch.interaction.interactive import InteractiveReviewer
from documatch.interaction.reviewer import (
    ModeDecision, ReviewDecision, Reviewer, SampleDecision, SpecDecision,
)

__all__ = [
    "Reviewer", "ReviewDecision", "SpecDecision", "ModeDecision", "SampleDecision",
    "AutomationReviewer", "InteractiveReviewer",
]
```

- [ ] **Step 31.4: Smoke test (no real interactivity — just construction)**

```python
from documatch.interaction.interactive import InteractiveReviewer


def test_construct():
    r = InteractiveReviewer()
    assert r is not None
```

- [ ] **Step 31.5: Commit**

```bash
git add src/documatch/interaction/prompts.py src/documatch/interaction/interactive.py src/documatch/interaction/__init__.py tests/unit/test_reviewer_interactive.py
git commit -m "feat(interaction): InteractiveReviewer (Rich 기반 6개 HITL 게이트)"
```

---

## Phase 10 — Engine & Pipeline (Tasks 32-33)

### Task 32: ExtractionEngine (orchestrator)

**Files:**
- Create: `src/documatch/core/engine.py`
- Modify: `src/documatch/core/__init__.py`
- Create: `tests/integration/__init__.py`
- Create: `tests/integration/test_engine_happy_path.py`

- [ ] **Step 32.1: Failing integration test (with FakeLLMClient)**

```python
import json
from datetime import datetime
from pathlib import Path

import pytest

from documatch.core.engine import ExtractionEngine
from documatch.interaction.automation import AutomationReviewer
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)
from documatch.spec.store import SpecStore
from tests.fakes import FakeLLMClient


def _spec() -> ExtractionSpec:
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


@pytest.mark.asyncio
async def test_engine_runs_with_preloaded_spec(tmp_path: Path):
    (tmp_path / "a.txt").write_text("hello")
    (tmp_path / "b.txt").write_text("world")
    out = tmp_path / "out.xlsx"
    spec_dir = tmp_path / "specs"

    response = json.dumps({"values": {"a": "x"}, "confidence": {"a": 0.9}, "summary_sentence": "x"})
    spec_llm = FakeLLMClient(responses=[])
    extract_llm = FakeLLMClient(responses=[response, response])

    reviewer = AutomationReviewer(output_path=out)
    engine = ExtractionEngine(
        spec_llm=spec_llm,
        extract_llm=extract_llm,
        spec_store=SpecStore(base_dir=spec_dir),
        reviewer=reviewer,
    )
    final = await engine.run(scan_path=tmp_path, preloaded_spec=_spec())
    assert out.exists()
    assert len(final.results) == 2
    assert all(r.status == "success" for r in final.results)
```

- [ ] **Step 32.2: Implement `src/documatch/core/engine.py`**

```python
"""ExtractionEngine — 7단계 파이프라인 오케스트레이터."""
from dataclasses import dataclass, field
from datetime import datetime
from pathlib import Path

from documatch.core.models import ExtractionResult
from documatch.exceptions import SpecNotFoundError, UserAbort
from documatch.exporters.excel import ExcelExporter
from documatch.extractors.batch import BatchExtractor
from documatch.extractors.single import SingleExtractor
from documatch.interaction.reviewer import Reviewer
from documatch.llm.base import LLMClient
from documatch.processors.base import FileEntry
from documatch.processors.registry import get_processor
from documatch.processors.scanner import scan_directory
from documatch.processors.xls import XlsProcessor
from documatch.processors.xlsx import XlsxProcessor
from documatch.spec.generator import SpecGenerator
from documatch.spec.models import ExtractionSpec
from documatch.spec.store import SpecStore


@dataclass
class PipelineFinal:
    spec: ExtractionSpec
    results: list[ExtractionResult]
    output_path: Path | None = None


@dataclass
class _State:
    scan_path: Path | None = None
    entries: list[FileEntry] = field(default_factory=list)
    spec: ExtractionSpec | None = None
    sample_path: Path | None = None
    sample_doc: object | None = None  # UnifiedDocument
    user_prompt: str = ""
    sample_result: ExtractionResult | None = None


class ExtractionEngine:
    def __init__(
        self,
        *,
        spec_llm: LLMClient,
        extract_llm: LLMClient,
        spec_store: SpecStore,
        reviewer: Reviewer,
        exporter: ExcelExporter | None = None,
    ) -> None:
        self.spec_llm = spec_llm
        self.extract_llm = extract_llm
        self.spec_store = spec_store
        self.reviewer = reviewer
        self.exporter = exporter or ExcelExporter()
        self.spec_generator = SpecGenerator(llm=spec_llm)

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
            self.reviewer.show_message("error", "스캔된 파일이 없습니다.")
            raise UserAbort("no files")

        # HITL ①
        decision = self.reviewer.confirm_files(st.entries)
        if decision.action == "quit":
            raise UserAbort("user quit at file confirm")
        if decision.action == "filter" and decision.extension_filter:
            st.entries = [e for e in st.entries if e.extension in decision.extension_filter]

        # Step 2-4: Spec acquisition
        if preloaded_spec is not None:
            st.spec = preloaded_spec
        else:
            st.spec = await self._acquire_spec(st)

        # Step 4: Mode confirm (skipped when preloaded — already finalized)
        if preloaded_spec is None:
            mode = self.reviewer.confirm_mode(st.spec)
            if mode.action == "back":
                st.spec = await self._acquire_spec(st)
            else:
                st.spec.table_mode = (mode.action == "table")

        # Step 5: Sample extract & review (skip when preloaded — trust saved spec)
        if preloaded_spec is None:
            await self._sample_review_loop(st)

        # Optional: save spec
        if preloaded_spec is None:
            name = self.reviewer.confirm_save_spec(st.spec)
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
            self.reviewer.show_progress(idx, total, r.document_id)

        # Step 7: Export
        timestamp = datetime.now().strftime("%Y-%m-%d_%H%M%S")
        default_out = default_output or Path(f"./documatch_result_{timestamp}.xlsx")
        out_path = self.reviewer.get_save_path(default_out)
        self.exporter.write(st.spec, results, out_path)
        self.reviewer.show_message("info", f"저장됨: {out_path}")
        return PipelineFinal(spec=st.spec, results=results, output_path=out_path)

    async def _acquire_spec(self, st: _State) -> ExtractionSpec:
        while True:
            sample_path = self.reviewer.select_sample(st.entries)
            if sample_path is None:
                raise UserAbort("no sample")
            st.sample_path = sample_path
            st.sample_doc = await self._load(sample_path)
            st.user_prompt = self.reviewer.get_user_prompt()
            spec = await self.spec_generator.generate(st.sample_doc, st.user_prompt)

            while True:
                avail = [s.name for s in self.spec_store.list()]
                decision = self.reviewer.review_spec(spec, available_specs=avail)
                if decision.action == "quit":
                    raise UserAbort("user quit at spec review")
                if decision.action == "back":
                    break  # re-select sample
                if decision.action == "approve":
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
                        self.reviewer.show_message("warn", "스펙을 찾을 수 없습니다.")
                        continue

    async def _sample_review_loop(self, st: _State) -> None:
        single = SingleExtractor(spec=st.spec, llm=self.extract_llm)
        while True:
            sample_id = str(st.sample_path) if st.sample_path else "sample"
            st.sample_result = await single.extract(st.sample_doc, document_id=sample_id)
            decision = self.reviewer.review_sample(st.sample_result, st.spec)
            if decision.action == "quit":
                raise UserAbort("user quit at sample review")
            if decision.action == "back":
                st.spec = await self._acquire_spec(st)
                single = SingleExtractor(spec=st.spec, llm=self.extract_llm)
                continue
            if decision.action == "single_save":
                # Treat as full but with only the sample
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

- [ ] **Step 32.3: Update `src/documatch/core/__init__.py`**

```python
from documatch.core.engine import ExtractionEngine, PipelineFinal
from documatch.core.models import ExtractionResult

__all__ = ["ExtractionResult", "ExtractionEngine", "PipelineFinal"]
```

- [ ] **Step 32.4: Run integration test — PASS**

```bash
pytest tests/integration/test_engine_happy_path.py -v
```

- [ ] **Step 32.5: Commit**

```bash
git add src/documatch/core/engine.py src/documatch/core/__init__.py tests/integration/__init__.py tests/integration/test_engine_happy_path.py
git commit -m "feat(core): ExtractionEngine 오케스트레이터 (7단계 + HITL hooks)"
```

---

### Task 33: Engine integration tests — HITL refine + partial failure

**Files:**
- Create: `tests/integration/test_engine_hitl_refine.py`
- Create: `tests/integration/test_engine_partial_failure.py`

- [ ] **Step 33.1: Refine loop integration test**

```python
import json
from pathlib import Path

import pytest

from documatch.core.engine import ExtractionEngine
from documatch.interaction.reviewer import (
    ModeDecision, ReviewDecision, SampleDecision, SpecDecision,
)
from documatch.spec.store import SpecStore
from tests.fakes import FakeLLMClient


class _ScriptedReviewer:
    """피드백 시퀀스를 지정해 spec refine + sample refine 루프 검증."""
    def __init__(self, output_path: Path):
        self.output_path = output_path
        self.spec_calls = 0
        self.sample_calls = 0

    def confirm_files(self, entries):
        return ReviewDecision(action="continue")

    def select_sample(self, entries):
        return Path(entries[0].file_path)

    def get_user_prompt(self, default=None):
        return "추출"

    def review_spec(self, spec, available_specs=None):
        self.spec_calls += 1
        if self.spec_calls == 1:
            return SpecDecision(action="refine", feedback="필드 변경")
        return SpecDecision(action="approve")

    def confirm_mode(self, spec):
        return ModeDecision(action="single")

    def review_sample(self, result, spec):
        self.sample_calls += 1
        if self.sample_calls == 1:
            return SampleDecision(action="refine", feedback="값 보정")
        return SampleDecision(action="full")

    def get_save_path(self, default):
        return self.output_path

    def confirm_save_spec(self, spec):
        return None

    def show_progress(self, c, t, l): pass

    def show_message(self, lvl, t): pass


@pytest.mark.asyncio
async def test_engine_refine_loops(tmp_path: Path):
    (tmp_path / "a.txt").write_text("doc")
    out = tmp_path / "out.xlsx"

    base_spec_json = json.dumps({
        "name": "x", "doc_type_hint": "x",
        "fields": [{"name": "a", "description": "d", "type": "string", "required": True}],
        "summary_template": {"template": "{a}"},
        "output_table": [{"field_name": "a", "excel_header": "a", "width": 10}],
        "table_mode": False,
    })
    refined_spec_json = base_spec_json  # 동일 구조이지만 LLM이 응답하는 흐름만 검증
    extract_resp = json.dumps({"values": {"a": "x"}, "confidence": {"a": 0.9}, "summary_sentence": "x"})

    spec_llm = FakeLLMClient(responses=[base_spec_json, refined_spec_json, refined_spec_json])
    extract_llm = FakeLLMClient(responses=[extract_resp, extract_resp, extract_resp])

    reviewer = _ScriptedReviewer(output_path=out)
    engine = ExtractionEngine(
        spec_llm=spec_llm, extract_llm=extract_llm,
        spec_store=SpecStore(base_dir=tmp_path / "specs"),
        reviewer=reviewer,
    )
    final = await engine.run(scan_path=tmp_path)
    assert reviewer.spec_calls == 2
    assert reviewer.sample_calls == 2
    assert out.exists()
    assert final.spec.name == "x"
```

- [ ] **Step 33.2: Partial failure integration test**

```python
import json
from pathlib import Path

import pytest

from documatch.core.engine import ExtractionEngine
from documatch.interaction.automation import AutomationReviewer
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)
from documatch.spec.store import SpecStore
from tests.fakes import FakeLLMClient


def _spec() -> ExtractionSpec:
    return ExtractionSpec(
        name="t", doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


@pytest.mark.asyncio
async def test_engine_partial_failure_continues(tmp_path: Path):
    (tmp_path / "ok.txt").write_text("ok")
    (tmp_path / "bad.txt").write_text("bad")
    out = tmp_path / "out.xlsx"

    good = json.dumps({"values": {"a": "x"}, "confidence": {"a": 0.9}, "summary_sentence": "x"})
    extract_llm = FakeLLMClient(responses=[good, "garbage"])
    spec_llm = FakeLLMClient(responses=[])

    reviewer = AutomationReviewer(output_path=out)
    engine = ExtractionEngine(
        spec_llm=spec_llm, extract_llm=extract_llm,
        spec_store=SpecStore(base_dir=tmp_path / "specs"),
        reviewer=reviewer,
    )
    final = await engine.run(scan_path=tmp_path, preloaded_spec=_spec())
    assert out.exists()
    assert any(r.status == "success" for r in final.results)
    assert any(r.status in ("partial", "failed") for r in final.results)

    from openpyxl import load_workbook
    wb = load_workbook(out)
    if any(r.status == "failed" for r in final.results):
        assert "failed_documents" in wb.sheetnames
```

- [ ] **Step 33.3: Run — PASS**

- [ ] **Step 33.4: Commit**

```bash
git add tests/integration/test_engine_hitl_refine.py tests/integration/test_engine_partial_failure.py
git commit -m "test(core): HITL refine 루프 + 부분 실패 통합 검증"
```

---

## Phase 11 — CLI (Tasks 34-37)

### Task 34: Click app entry point + main interactive command

**Files:**
- Create: `src/documatch/cli/__init__.py`
- Create: `src/documatch/cli/app.py`
- Create: `src/documatch/cli/_runner.py`
- Create: `tests/integration/test_cli_smoke.py`

- [ ] **Step 34.1: Implement `src/documatch/cli/_runner.py`** (engine 빌드 + 실행 함수)

```python
"""CLI 진입점에서 ExtractionEngine을 빌드해 실행."""
import asyncio
from pathlib import Path

from documatch.config.settings import Settings
from documatch.core.engine import ExtractionEngine
from documatch.exceptions import ConfigError, UserAbort
from documatch.exporters.excel import ExcelExporter
from documatch.interaction.automation import AutomationReviewer
from documatch.interaction.interactive import InteractiveReviewer
from documatch.interaction.reviewer import Reviewer
from documatch.llm.factory import create_llm
from documatch.spec.models import ExtractionSpec
from documatch.spec.store import SpecStore


def build_engine(
    settings: Settings,
    *,
    reviewer: Reviewer,
) -> ExtractionEngine:
    spec_llm = create_llm(settings, role="spec")
    extract_llm = create_llm(settings, role="extract")
    return ExtractionEngine(
        spec_llm=spec_llm,
        extract_llm=extract_llm,
        spec_store=SpecStore(base_dir=settings.spec_store_dir),
        reviewer=reviewer,
        exporter=ExcelExporter(),
    )


def run_pipeline(
    *,
    scan_path: Path,
    spec_name: str | None,
    spec_file: Path | None,
    output: Path | None,
    auto_approve: bool,
    settings: Settings,
    fail_on_review: bool = False,
    fail_on_error: int | None = None,
) -> int:
    if auto_approve and not (spec_name or spec_file):
        raise ConfigError("--auto-approve는 --spec 또는 --spec-file과 함께 사용해야 합니다.")

    spec_store = SpecStore(base_dir=settings.spec_store_dir)
    preloaded: ExtractionSpec | None = None
    if spec_file:
        preloaded = spec_store.load_from_path(spec_file)
    elif spec_name:
        preloaded = spec_store.load(spec_name)

    if auto_approve:
        reviewer: Reviewer = AutomationReviewer(output_path=output or Path("./out.xlsx"))
    else:
        reviewer = InteractiveReviewer()

    engine = build_engine(settings, reviewer=reviewer)
    try:
        final = asyncio.run(engine.run(
            scan_path=scan_path, preloaded_spec=preloaded, default_output=output,
        ))
    except UserAbort:
        return 130
    except ConfigError as e:
        print(f"설정 오류: {e}")
        return 2

    failed = sum(1 for r in final.results if r.status == "failed")
    review_count = sum(1 for r in final.results if r.needs_review)
    if fail_on_error is not None and failed >= fail_on_error:
        print(f"실패 {failed}건 — fail-on-error 임계값 초과", flush=True)
        return 1
    if fail_on_review and review_count > 0:
        print(f"검토 필요 {review_count}건 — fail-on-review 발동", flush=True)
        return 1
    return 0
```

- [ ] **Step 34.2: Implement `src/documatch/cli/app.py`**

```python
"""documatch CLI 진입점 (Click)."""
from pathlib import Path

import click

from documatch import __version__
from documatch.cli._runner import run_pipeline
from documatch.config.settings import Settings


@click.group(invoke_without_command=True)
@click.argument("scan_path", required=False, type=click.Path(exists=True, path_type=Path))
@click.option("--spec", "spec_name", default=None, help="저장된 스펙 이름")
@click.option("--spec-file", default=None, type=click.Path(exists=True, path_type=Path),
              help="외부 JSON 스펙 파일")
@click.option("--output", "-o", default=None, type=click.Path(path_type=Path),
              help="결과 Excel 경로")
@click.option("--auto-approve", is_flag=True, default=False, help="모든 HITL 자동 승인")
@click.option("--debug", is_flag=True, default=False, help="DEBUG 로그")
@click.option("--fail-on-review", is_flag=True, default=False)
@click.option("--fail-on-error", type=int, default=None, help="N건 이상 실패 시 비-0 종료")
@click.version_option(__version__, prog_name="documatch")
@click.pass_context
def main(
    ctx: click.Context,
    scan_path: Path | None,
    spec_name: str | None,
    spec_file: Path | None,
    output: Path | None,
    auto_approve: bool,
    debug: bool,
    fail_on_review: bool,
    fail_on_error: int | None,
) -> None:
    """documatch — AI 기반 문서 추출 CLI."""
    if ctx.invoked_subcommand is not None:
        return
    if scan_path is None:
        scan_path = Path(click.prompt("스캔 경로 입력", type=str))
    settings = Settings()
    if debug:
        settings.debug = True
    rc = run_pipeline(
        scan_path=scan_path, spec_name=spec_name, spec_file=spec_file,
        output=output, auto_approve=auto_approve, settings=settings,
        fail_on_review=fail_on_review, fail_on_error=fail_on_error,
    )
    ctx.exit(rc)
```

- [ ] **Step 34.3: Implement `src/documatch/cli/__init__.py`**

```python
from documatch.cli.app import main

__all__ = ["main"]
```

- [ ] **Step 34.4: Smoke test**

```python
from click.testing import CliRunner

from documatch.cli.app import main


def test_version_flag():
    result = CliRunner().invoke(main, ["--version"])
    assert result.exit_code == 0
    assert "documatch" in result.output


def test_help_runs():
    result = CliRunner().invoke(main, ["--help"])
    assert result.exit_code == 0
    assert "scan_path" in result.output.lower() or "PATH" in result.output
```

- [ ] **Step 34.5: Run — PASS**

- [ ] **Step 34.6: Commit**

```bash
git add src/documatch/cli/__init__.py src/documatch/cli/app.py src/documatch/cli/_runner.py tests/integration/test_cli_smoke.py
git commit -m "feat(cli): Click 진입점 + 대화형/자동화 통합 실행"
```

---

### Task 35: spec subcommands

**Files:**
- Create: `src/documatch/cli/spec_commands.py`
- Modify: `src/documatch/cli/app.py`
- Create: `tests/integration/test_cli_spec_commands.py`

- [ ] **Step 35.1: Implement `src/documatch/cli/spec_commands.py`**

```python
"""documatch spec list/show/delete."""
import click

from documatch.config.settings import Settings
from documatch.exceptions import SpecNotFoundError
from documatch.spec.store import SpecStore


@click.group("spec")
def spec_group() -> None:
    """저장된 스펙 관리."""


@spec_group.command("list")
def spec_list() -> None:
    store = SpecStore(base_dir=Settings().spec_store_dir)
    infos = store.list()
    if not infos:
        click.echo("저장된 스펙 없음.")
        return
    for info in infos:
        click.echo(f"{info.name}\t{info.doc_type_hint}\t{info.created_at.isoformat()}")


@spec_group.command("show")
@click.argument("name")
def spec_show(name: str) -> None:
    store = SpecStore(base_dir=Settings().spec_store_dir)
    try:
        spec = store.load(name)
    except SpecNotFoundError as e:
        raise click.ClickException(str(e))
    click.echo(spec.model_dump_json(indent=2))


@spec_group.command("delete")
@click.argument("name")
def spec_delete(name: str) -> None:
    store = SpecStore(base_dir=Settings().spec_store_dir)
    try:
        store.delete(name)
    except SpecNotFoundError as e:
        raise click.ClickException(str(e))
    click.echo(f"삭제됨: {name}")
```

- [ ] **Step 35.2: Wire into `src/documatch/cli/app.py`** — append at bottom of file:

```python
from documatch.cli.spec_commands import spec_group  # noqa: E402

main.add_command(spec_group)
```

- [ ] **Step 35.3: Test**

```python
from pathlib import Path

from click.testing import CliRunner

from documatch.cli.app import main
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)
from documatch.spec.store import SpecStore


def _make_spec(name="alpha"):
    return ExtractionSpec(
        name=name, doc_type_hint="t",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )


def test_spec_list_show_delete(tmp_path, monkeypatch):
    monkeypatch.setenv("DOCUMATCH_SPEC_STORE_DIR", str(tmp_path))
    SpecStore(base_dir=tmp_path).save(_make_spec("alpha"))

    runner = CliRunner()
    r = runner.invoke(main, ["spec", "list"])
    assert "alpha" in r.output

    r = runner.invoke(main, ["spec", "show", "alpha"])
    assert "alpha" in r.output

    r = runner.invoke(main, ["spec", "delete", "alpha"])
    assert "삭제됨" in r.output

    r = runner.invoke(main, ["spec", "show", "alpha"])
    assert r.exit_code != 0
```

- [ ] **Step 35.4: Run — PASS**

- [ ] **Step 35.5: Commit**

```bash
git add src/documatch/cli/spec_commands.py src/documatch/cli/app.py tests/integration/test_cli_spec_commands.py
git commit -m "feat(cli): spec list/show/delete 서브커맨드"
```

---

### Task 36: hyperlinks + config subcommands

**Files:**
- Create: `src/documatch/cli/hyperlinks_commands.py`
- Create: `src/documatch/cli/config_commands.py`
- Modify: `src/documatch/cli/app.py`
- Create: `tests/integration/test_cli_hyperlinks.py`

- [ ] **Step 36.1: Implement `src/documatch/cli/hyperlinks_commands.py`**

```python
"""documatch hyperlinks fix."""
from pathlib import Path

import click

from documatch.exporters.hyperlinks import fix_hyperlinks


@click.group("hyperlinks")
def hyperlinks_group() -> None:
    """Excel 하이퍼링크 유틸."""


@hyperlinks_group.command("fix")
@click.argument("target", type=click.Path(exists=True, path_type=Path))
@click.option("--dry-run", is_flag=True, default=False)
def hyperlinks_fix(target: Path, dry_run: bool) -> None:
    if target.is_dir():
        files = sorted(target.rglob("*.xlsx"))
    else:
        files = [target]
    total = 0
    for f in files:
        n = fix_hyperlinks(f, dry_run=dry_run)
        click.echo(f"{f}: {n}건")
        total += n
    label = "변환 예정" if dry_run else "변환 완료"
    click.echo(f"총 {len(files)}개 파일, {total}건 {label}")
```

- [ ] **Step 36.2: Implement `src/documatch/cli/config_commands.py`**

```python
"""documatch config show."""
import json

import click

from documatch.config.settings import Settings


@click.group("config")
def config_group() -> None:
    """설정 진단."""


@config_group.command("show")
def config_show() -> None:
    s = Settings()
    payload = {
        "anthropic_api_key": "set" if s.anthropic_api_key else "missing",
        "openai_api_key": "set" if s.openai_api_key else "missing",
        "llm_spec": s.llm_spec.model_dump(),
        "llm_extract": s.llm_extract.model_dump(),
        "spec_store_dir": str(s.spec_store_dir),
        "pdf_scan_threshold": s.pdf_scan_threshold,
        "extraction_batch_size": s.extraction_batch_size,
        "debug": s.debug,
    }
    click.echo(json.dumps(payload, indent=2, ensure_ascii=False))
```

- [ ] **Step 36.3: Wire into `src/documatch/cli/app.py`**

Append at bottom:

```python
from documatch.cli.config_commands import config_group  # noqa: E402
from documatch.cli.hyperlinks_commands import hyperlinks_group  # noqa: E402

main.add_command(config_group)
main.add_command(hyperlinks_group)
```

- [ ] **Step 36.4: Test**

```python
from pathlib import Path

from click.testing import CliRunner
from openpyxl import Workbook

from documatch.cli.app import main


def test_config_show():
    r = CliRunner().invoke(main, ["config", "show"])
    assert r.exit_code == 0
    assert "spec_store_dir" in r.output


def test_hyperlinks_fix(tmp_path: Path):
    target = tmp_path / "target.pdf"; target.write_bytes(b"%PDF")
    xlsx = tmp_path / "x.xlsx"
    wb = Workbook(); ws = wb.active
    ws.cell(row=1, column=1, value="link").hyperlink = str(target.resolve())
    wb.save(xlsx)
    r = CliRunner().invoke(main, ["hyperlinks", "fix", str(xlsx)])
    assert r.exit_code == 0
    assert "변환 완료" in r.output
```

- [ ] **Step 36.5: Run — PASS**

- [ ] **Step 36.6: Commit**

```bash
git add src/documatch/cli/hyperlinks_commands.py src/documatch/cli/config_commands.py src/documatch/cli/app.py tests/integration/test_cli_hyperlinks.py
git commit -m "feat(cli): hyperlinks fix + config show 서브커맨드"
```

---

### Task 37: CLI automation end-to-end test

**Files:**
- Create: `tests/integration/test_cli_automation_e2e.py`

- [ ] **Step 37.1: Write E2E test**

```python
"""CLI 자동화 모드 E2E — Click runner + monkeypatched LLM."""
import json
from pathlib import Path

import pytest
from click.testing import CliRunner

from documatch.cli.app import main
from documatch.spec.models import (
    ExtractionSpec, Field, FieldType, OutputColumn, SummaryTemplate,
)
from documatch.spec.store import SpecStore
from tests.fakes import FakeLLMClient


@pytest.fixture
def fake_llm(monkeypatch):
    response = json.dumps({"values": {"a": "x"}, "confidence": {"a": 0.9}, "summary_sentence": "x"})
    fake = FakeLLMClient(responses=[response] * 50)

    def _fake_create_llm(settings, *, role):
        return fake

    monkeypatch.setattr("documatch.cli._runner.create_llm", _fake_create_llm)
    return fake


def test_cli_automation_with_spec(tmp_path: Path, fake_llm, monkeypatch):
    (tmp_path / "a.txt").write_text("hi")
    (tmp_path / "b.txt").write_text("bye")
    out = tmp_path / "out.xlsx"
    spec_dir = tmp_path / "specs"
    monkeypatch.setenv("DOCUMATCH_SPEC_STORE_DIR", str(spec_dir))
    monkeypatch.setenv("ANTHROPIC_API_KEY", "x")

    spec = ExtractionSpec(
        name="invoice", doc_type_hint="invoice",
        fields=[Field(name="a", description="d", type=FieldType.STRING)],
        summary_template=SummaryTemplate(template="{a}"),
        output_table=[OutputColumn(field_name="a", excel_header="a")],
    )
    SpecStore(base_dir=spec_dir).save(spec)

    runner = CliRunner()
    r = runner.invoke(main, [
        str(tmp_path), "--spec", "invoice",
        "--output", str(out), "--auto-approve",
    ])
    assert r.exit_code == 0, r.output
    assert out.exists()


def test_cli_auto_approve_without_spec_errors(tmp_path: Path, monkeypatch):
    monkeypatch.setenv("ANTHROPIC_API_KEY", "x")
    (tmp_path / "a.txt").write_text("hi")
    runner = CliRunner()
    r = runner.invoke(main, [str(tmp_path), "--auto-approve"])
    assert r.exit_code != 0
```

- [ ] **Step 37.2: Run — PASS**

- [ ] **Step 37.3: Commit**

```bash
git add tests/integration/test_cli_automation_e2e.py
git commit -m "test(cli): 자동화 E2E + --auto-approve 안전장치 검증"
```

---

## Phase 12 — Documentation, Release Prep, Final QA (Tasks 38-41)

### Task 38: README — full content

**Files:**
- Modify: `README.md`

- [ ] **Step 38.1: Replace `README.md`**

```markdown
# documatch-cli

AI 기반 문서 추출 CLI — PDF/DOCX/XLSX/이미지/HTML/CSV 등에서 원하는 항목을 LLM으로 추출해 Excel로 저장합니다.

## Quick Start

```bash
pip install documatch-cli
export ANTHROPIC_API_KEY=sk-ant-...
documatch ./samples
```

스캔 PDF를 사용하려면 Poppler 설치:
- macOS: `brew install poppler`
- Linux: `sudo apt install poppler-utils`
- Windows: https://github.com/oschwartz10612/poppler-windows/releases

## 사용 예시

대화형(기본):
```bash
documatch ./samples
```

저장된 스펙 재사용 + 검토 단계만 통과:
```bash
documatch ./samples --spec invoice_v1
```

완전 자동화(CI):
```bash
documatch ./samples --spec invoice_v1 --output result.xlsx --auto-approve
```

외부 JSON 스펙 사용:
```bash
documatch ./samples --spec-file ./my-spec.json --output out.xlsx --auto-approve
```

보조 명령:
```bash
documatch spec list
documatch spec show invoice_v1
documatch spec delete invoice_v1
documatch hyperlinks fix ./out.xlsx
documatch config show
```

## 지원 파일 포맷

| 확장자 | 설명 |
|---|---|
| .pdf | 텍스트 PDF + 스캔 PDF (Poppler + Vision LLM) |
| .docx | Word |
| .xlsx | Excel (시트별 분리) |
| .xls | 구형 Excel |
| .jpg/.jpeg/.png | 이미지 (Vision LLM) |
| .txt/.md | 평문/마크다운 |
| .csv | CSV |
| .html/.htm | HTML |

## 설정

환경 변수 또는 `documatch.toml` (CWD 또는 `~/.config/documatch/config.toml`):
```toml
[llm_spec]
provider = "anthropic"
model = "claude-sonnet-4-6"

[llm_extract]
provider = "openai"
model = "gpt-4o-mini"

pdf_scan_threshold = 100
```

환경 변수 (`DOCUMATCH_` prefix, nested는 `__`):
```
ANTHROPIC_API_KEY=...
OPENAI_API_KEY=...
DOCUMATCH_LLM_EXTRACT__PROVIDER=openai
DOCUMATCH_DEBUG=true
```

## Human-in-the-Loop 흐름

7단계 파이프라인 + 6개 확인 지점:
1. 파일 스캔 → 결과 확인
2. 샘플 선택
3. AI가 스키마 생성 → 승인 / 피드백 수정 / 저장된 스펙 불러오기
4. 모드 확정 (단일값 vs 테이블)
5. 샘플 추출 → 결과 확인 / 피드백 수정 후 재추출
6. 전체 배치 실행 (`b` 키로 중단)
7. 저장 경로 확인 → Excel 출력

## 라이선스

MIT
```

- [ ] **Step 38.2: Commit**

```bash
git add README.md
git commit -m "docs: README 본문 작성 (사용법, 포맷, 설정, HITL 흐름)"
```

---

### Task 39: CONTRIBUTING.md + CHANGELOG entry

**Files:**
- Create: `CONTRIBUTING.md`
- Modify: `CHANGELOG.md`

- [ ] **Step 39.1: Write `CONTRIBUTING.md`**

```markdown
# Contributing to documatch-cli

## 개발 환경 셋업

```bash
git clone https://github.com/<owner>/documatch-cli.git
cd documatch-cli
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
```

## 작업 흐름

1. Issue 또는 RFC 생성 (큰 변경)
2. 브랜치 작성 (`feat/...`, `fix/...`)
3. TDD: 실패하는 테스트 → 구현 → 통과
4. `ruff check` + `ruff format` + `mypy src/documatch` + `pytest`
5. CHANGELOG.md `## [Unreleased]`에 항목 추가
6. PR 작성

## 테스트 정책

- 단위 테스트는 `FakeLLMClient`만 사용 (실 API 호출 금지)
- fixture는 작은 샘플만 (수십 KB 단위)
- 새 processor는 fixture + 단위 테스트 필수
- 모든 PR에서 커버리지 85% 이상 유지

## 커밋 메시지

`type(scope): subject` 한국어 가능. 예:
- `feat(cli): --auto-approve 옵션 추가`
- `fix(processors): XLSX 빈 셀 처리`
- `test(spec): refine 루프 통합 테스트 보강`

## 릴리즈 (메인테이너 전용)

1. CHANGELOG `Unreleased` → `[X.Y.Z] - YYYY-MM-DD` 변경
2. `pyproject.toml` version bump
3. `git tag vX.Y.Z && git push --tags`
4. GitHub Actions가 PyPI Trusted Publishing으로 배포
```

- [ ] **Step 39.2: Update `CHANGELOG.md`**

```markdown
# Changelog

## [Unreleased]

### Added
- 7단계 추출 파이프라인 (scan → spec → review → batch → export)
- 6개 HITL 확인 지점 (대화형 모드)
- Anthropic + OpenAI LLM 동시 지원, role 분리(spec/extract)
- 파일 포맷: PDF (스캔 분기 포함) / DOCX / XLSX / XLS / JPG / PNG / TXT / MD / CSV / HTML
- 스펙 영속화 (~/.config/documatch/specs/)
- CLI: 대화형 단일 명령 + `--auto-approve` 자동화 모드
- 서브커맨드: `spec list/show/delete`, `hyperlinks fix`, `config show`
- Excel exporter (results + failed_documents 시트)
```

- [ ] **Step 39.3: Commit**

```bash
git add CONTRIBUTING.md CHANGELOG.md
git commit -m "docs: CONTRIBUTING + CHANGELOG 0.1.0 항목"
```

---

### Task 40: Release workflow + PR template

**Files:**
- Create: `.github/workflows/release.yml`
- Create: `.github/pull_request_template.md`

- [ ] **Step 40.1: Write `release.yml`**

```yaml
name: Release
on:
  push:
    tags: ["v*"]

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install Poppler
        run: sudo apt-get update && sudo apt-get install -y poppler-utils
      - run: pip install -e ".[dev]"
      - run: ruff check .
      - run: mypy src/documatch
      - run: pytest
      - run: python -m build
      - name: Publish to PyPI
        uses: pypa/gh-action-pypi-publish@release/v1
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
          files: dist/*
```

- [ ] **Step 40.2: Write `.github/pull_request_template.md`**

```markdown
## 요약
<!-- 1-3줄로 변경 사항 -->

## 체크리스트
- [ ] CHANGELOG `## [Unreleased]`에 항목 추가
- [ ] 단위/통합 테스트 추가 또는 갱신
- [ ] `ruff check` + `ruff format --check` 통과
- [ ] `mypy src/documatch` 통과
- [ ] 커버리지 85% 이상 유지
```

- [ ] **Step 40.3: Commit**

```bash
git add .github/workflows/release.yml .github/pull_request_template.md
git commit -m "ci: PyPI 릴리즈 워크플로 + PR 템플릿"
```

---

### Task 41: Final QA — coverage, lint, install smoke

- [ ] **Step 41.1: Run full test suite + coverage**

```bash
pytest --cov=documatch --cov-report=term-missing
```
Expected: all tests pass, coverage ≥ 85%.

- [ ] **Step 41.2: Lint + format check**

```bash
ruff check .
ruff format --check .
mypy src/documatch
```

- [ ] **Step 41.3: Build and install smoke**

```bash
python -m build
pip install dist/documatch_cli-0.1.0-py3-none-any.whl --force-reinstall
documatch --version
documatch --help
documatch spec list
documatch config show
```

- [ ] **Step 41.4: Manual smoke — interactive mode**

Place 1-2 small text files in `/tmp/smoke/` and run `documatch /tmp/smoke`. Walk all 6 HITL gates with real Anthropic key (using a tiny doc to keep cost minimal). Confirm Excel saves and looks correct.

- [ ] **Step 41.5: Tag release candidate (do NOT push to remote yet)**

```bash
git tag v0.1.0-rc1
git log --oneline -10
```

- [ ] **Step 41.6: User decision before pushing**

The agent stops here. The plan does not push to remote, create the GitHub repo, or publish to PyPI without explicit user approval. The user reviews the local repo, then runs:

```bash
gh repo create documatch-cli --public --source=. --push
git push origin v0.1.0
```

---

## Self-Review Checklist (planner ran)

**Spec coverage:** All 12 spec sections mapped to tasks.
- §1 목적 → Task 1 (scaffolding)
- §2 패키지 구조 → Tasks 1, 13, 24, 27, 29
- §3 파이프라인 + HITL → Tasks 29, 30, 31, 32, 33
- §4 LLM 추상화 → Tasks 8, 10, 11, 12
- §5 프로세서 → Tasks 13-21
- §6 ExtractionSpec → Tasks 6, 7, 22, 23
- §7 설정 + CLI UX → Tasks 5, 34, 35, 36
- §8 에러 + 테스트 → Tasks 2, 9, 33, 41
- §9 배포 → Tasks 1, 4, 38, 39, 40
- §10 성공 기준 → Task 41 (manual smoke)
- §11 위험 → 완화책이 LLM/processor 태스크에 분산 반영됨

**Placeholder scan:** No "TBD/TODO/implement later" found in tasks. Inline code in every step.

**Type consistency:** `LLMClient`, `LLMMessage`, `LLMResponse`, `ExtractionSpec`, `ExtractionResult`, `FileEntry`, `UnifiedDocument`, `Reviewer`, decision types are referenced consistently across tasks.

**Order:** Each task only depends on previously-defined types/modules.

---

## Execution Handoff

**Plan complete and saved to `docs/superpowers/plans/2026-05-09-documatch-cli.md`. Two execution options:**

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?**










