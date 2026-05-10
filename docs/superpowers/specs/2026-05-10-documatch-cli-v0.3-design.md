# documatch-cli v0.3.0 설계 문서 — CLI Polish + GUI Deprecation

- 작성일: 2026-05-10
- 출발점: documatch-cli v0.2.1 (CLI + GUI), Windows SmartScreen + macOS TCC로 GUI 배포 환경 마찰 큼
- 목표: GUI 모듈을 archive 처리하고 CLI UX를 production-quality로 끌어올린 v0.3.0 릴리스
- 결과물: `documatch` CLI 단일 entry-point, Rich 기반 대폭 개선된 인터랙션, ~169 테스트

## 0. 브레인스토밍 결정 사항

| 항목 | 결정 | 근거 |
|---|---|---|
| GUI 처분 | **Hard delete** — main에서 모든 GUI 코드/테스트/빌드 자산 제거. archive 태그 생성 X. 필요 시 v0.2.1 commit에서 git checkout으로 복원 | 미래 부활 가능성 낮다고 판단 |
| CLI 개선 우선순위 | (1) 실시간 진행 (2) 단계 헤더+단축키 legend (3) 에러 박스 (4) 표 색상·정렬 | 회사 컴 SmartScreen 사건에서 "멈췄나?" 가 가장 큰 페인이었음 |
| 부가 기능 | `documatch preview <file>` + `--json/--quiet/--summary` 플래그 | 자동화·디버깅 편의 |
| 버전 | v0.3.0 (메이저 — GUI 제거 + CLI 대폭 개편) | 호환성 깨지는 entry-point 제거 |
| 범위 | InteractiveReviewer + 신규 helper 모듈만 변경. Engine/processors/llm 0건 변경 | 검증된 핵심부 보존 |
| 작업량 | 약 25-30 task / 1.5-2주 | v0.2.0 GUI(63 task)의 1/3 |

## 1. 아키텍처

### 1.1 변경 영향 범위

```
src/documatch/
├── cli/                          # 변경 없음 (app.py 일부만)
├── core/                         # 변경 없음
├── llm/                          # 변경 없음
├── processors/                   # 변경 없음
├── extractors/                   # 변경 없음
├── exporters/                    # 변경 없음
├── spec/                         # 변경 없음
├── interaction/
│   ├── interactive.py            # 전면 개편 (헤더, 색상, 진행, 에러)
│   ├── automation.py             # 변경 없음
│   ├── reviewer.py               # 변경 없음 (Protocol 그대로)
│   ├── prompts.py                # 변경 없음
│   └── ui/                       # 신규 모듈 — UI 헬퍼
│       ├── __init__.py
│       ├── header.py             # 단계 헤더 + breadcrumb
│       ├── progress.py           # Rich Live 스피너 + 진행률
│       ├── errors.py             # prominent 에러 박스
│       └── tables.py             # 표 렌더링 헬퍼 (색상·정렬)
├── gui/                          # 삭제 (archive/gui-v0.2.0 태그 보존)
└── __init__.py                   # __version__ = "0.3.0"
```

기존 InteractiveReviewer는 `prompts.py`에서 import한 helper(`choice`, `multiline_input`)를 그대로 쓰고, 새 `interaction/ui/` 모듈에서 시각 요소를 가져온다.

### 1.2 핵심 결정

1. **InteractiveReviewer만 개편**, AutomationReviewer는 그대로 유지 (CI/배치는 시각 요소 불필요)
2. **Reviewer Protocol 변경 0건** — 메서드 시그니처/dataclass 그대로
3. **engine.py 변경 0건** — 호출부 동일
4. **신규 모듈은 InteractiveReviewer 전용** — 다른 곳에서 의존 X

### 1.3 호환성

- `documatch ./samples` CLI 진입점 그대로
- 기존 161 CLI 테스트 그대로 통과 (시각 요소만 추가, 동작 로직 동일)
- 사용자 데이터 (`~/.config/documatch/specs/`, `.env`) 호환
- v0.2.x → v0.3.0: `documatch-gui` 명령 제거, GUI 사용자는 `git checkout archive/gui-v0.2.0`으로 복원 가능

## 2. 핵심 개선 4가지 — 상세

### 2.1 실시간 진행 표시 (Rich Live + 스피너)

**문제**: LLM 호출 5-30초 동안 정적 출력만 있어 멈춘 건지 불안.

**해결**: `interaction/ui/progress.py`에 `live_spinner(message)` context manager + `progress_bar(total)` helper.

```python
# 사용 예 (interactive.py 내부)
async def review_spec(self, spec, ...):
    # 사용자 승인 후 LLM 재요청 시:
    with live_spinner(f"AI에게 스펙 수정 요청 중") as sp:
        sp.update_elapsed()  # 자동: "⠋ AI에게 스펙 수정 요청 중... (3.2s)"
        spec = await self.spec_generator.refine(...)
```

배치는:
```python
with progress_bar(total=len(entries), label="추출 중") as pb:
    async for r in batch.extract_all(entries):
        pb.advance(1, status=r.status, label=r.document_id)
```

배치 출력 예:
```
추출 중... ━━━━━━━━━━━━━━━━━━━━━━━━ 60% [3/5] | doc.pdf | ✓2 ✗1 ⚠️0
```

**구현 핵심**: Rich `Live` + `Spinner` + `Progress`. async 호환.

### 2.2 항상 보이는 단계 헤더 + 단축키 legend

**문제**: 어디 있는지·뭘 누를 수 있는지 매번 헤매게 됨.

**해결**: `interaction/ui/header.py`에 `render_step_header(step, total, name)` + `render_action_legend(actions)` helper.

매 HITL 게이트 시작 시:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Step 3 / 7  |  스펙 검토 — invoice_extraction
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

게이트 끝 액션 박스:
```
╭───────────── 선택 ─────────────╮
│  [A] 승인 + 다음 단계         │
│  [R] 피드백 입력 후 재요청    │
│  [L] 저장된 스펙 불러오기     │
│  [B] 샘플 재선택              │
│  [Q] 종료                     │
╰───────────────────────────────╯
> _
```

기존 `Prompt.ask` 호출은 그대로 쓰되, 직전에 헤더+legend를 매번 출력.

### 2.3 에러 박스

**문제**: 에러가 일반 출력에 묻혀 놓침.

**해결**: `interaction/ui/errors.py`에 `render_error_panel(exception)` helper. Rich `Panel` 빨강 테두리 + ⚠️ 아이콘 + 원인 + 조치 안내.

```
╭─ ⚠️  오류 ─────────────────────────────────────────────╮
│                                                       │
│  LLMAuthError: 401 Invalid API key                    │
│                                                       │
│  원인: API 키가 만료됐거나 잘못 입력됨                 │
│  조치: documatch init으로 키 재설정                    │
│                                                       │
╰───────────────────────────────────────────────────────╯
```

각 예외 타입별로 안내 매핑 (`_REMEDY_MAP: dict[type[Exception], str]`).

### 2.4 표 색상·정렬

**문제**: Rich Table 기본 출력이 한글 정렬 깨지고 신뢰도가 다 같은 색.

**해결**: `interaction/ui/tables.py`에 `render_spec_table(spec)`, `render_result_table(result, spec)` helper.

스펙 표:
- 필수 필드: 굵게 + 별표 ★
- 타입: 색상 (string=cyan, number=green, date=yellow)
- 컬럼 너비 자동 (max-width clamp)

결과 표 (단일값 모드):
```
필드명          값                    신뢰도
─────────────────────────────────────────────
invoice_no  ★  INV-2026-0042         0.95  🟢
amount      ★  1,500,000             0.92  🟢
issue_date     2026-05-10            0.45  🟡  ⚠️
supplier       알 수 없음              0.20  🔴  ⚠️
```

신뢰도 색상 임계: ≥0.8 🟢, ≥0.5 🟡, <0.5 🔴 + ⚠️ 검토필요 마크.

테이블 모드는 `QTableView` 같은 거 없으니 Rich Table을 행 단위로.

## 3. 부가 개선

### 3.1 `documatch preview <file>` 신규 명령

```bash
$ documatch preview samples/invoice-001.pdf
파일: invoice-001.pdf  |  PDF, 2 페이지, 12 KB

[첫 50줄]
송장
송장번호: INV-2026-0042
발행일: 2026-05-10
공급자: ABC 주식회사
...

[메타]
페이지 수: 2
텍스트 추출 가능: yes
스캔 PDF: no
```

XLSX는 시트 목록 + 각 시트 row/col 카운트. 이미지는 크기·dimension. 추출 시작 전 빠른 점검용.

`src/documatch/cli/preview_command.py` 신규.

### 3.2 자동화 모드 플래그 확장

기존 `--auto-approve` 외에 추가:
- `--quiet`: 진행 메시지 모두 억제, 결과만 stdout
- `--json`: 최종 결과를 stdout JSON으로 (Excel 저장 X)
- `--summary`: 추출 끝나고 한 줄 요약 (`12 successful, 2 failed, 0.85 avg confidence`)

CI/스크립팅에서 활용.

`run_pipeline`에 플래그 전달, AutomationReviewer가 활용.

## 4. GUI Hard Removal

### 4.1 main 브랜치 정리

삭제:
- `src/documatch/gui/` 디렉토리 전체
- `tests/gui/` 디렉토리 전체 (~100 테스트)
- `documatch_gui.spec` PyInstaller 명세
- `scripts/build_gui.py`, `scripts/generate_icons.py`
- `assets/` 디렉토리 (아이콘)
- `.github/workflows/release.yml`의 `build-gui` 매트릭스 잡

수정:
- `pyproject.toml`: `[project.optional-dependencies] gui = [...]` 제거, `documatch-gui = ...` entry-point 제거
- `pyproject.toml`: `version = "0.3.0"`
- `src/documatch/__init__.py`: `__version__ = "0.3.0"`
- `README.md`: GUI 섹션 제거, CLI 개선 사항 추가
- `CHANGELOG.md`: v0.3.0 항목 추가

### 4.2 부활 가능성 (참고)

별도 archive 태그 만들지 않음. 필요 시 v0.2.1 commit에서 직접 복원 가능:
```bash
# v0.2.1 commit SHA 확인 후
git checkout <v0.2.1-sha> -- src/documatch/gui/ tests/gui/ \
    documatch_gui.spec scripts/build_gui.py scripts/generate_icons.py assets/
# pyproject.toml에 gui extras + entry-point 복원
```

GitHub Release v0.2.1 페이지에 빌드된 .exe/.app은 그대로 유지 (사용자가 다시 다운로드 가능).

## 5. 테스트 전략

### 5.1 기존 테스트 처리

| 테스트 카테고리 | 처리 |
|---|---|
| `tests/unit/test_*` (CLI core) | 그대로 유지, 161건 통과 |
| `tests/unit/test_reviewer_interactive.py` | 새 헬퍼 import 반영, 38건 + 신규 추가 |
| `tests/integration/test_engine_*` | 변경 없음 |
| `tests/integration/test_cli_*` | 변경 없음 |
| `tests/gui/*` | **삭제** (~100건) |

총 269 → 169 테스트 (GUI 제거분) → 신규 추가 약 30건 → **목표 200건+, 커버리지 89%+**

### 5.2 신규 테스트

`tests/unit/test_ui_*.py`:
- `test_ui_header.py` — 헤더·legend 렌더링 (출력 string 검증)
- `test_ui_progress.py` — Live 스피너·progress bar (mock 시간)
- `test_ui_errors.py` — 예외 → 안내 매핑
- `test_ui_tables.py` — 색상·정렬 (Rich Console capture)

`tests/integration/test_cli_preview_command.py`:
- `documatch preview file.pdf` 출력 검증

`tests/unit/test_reviewer_interactive.py` 추가:
- 38 기존 테스트 + 새 헬퍼 통합 검증

## 6. 마이그레이션 단계 (Phase 단위)

| Phase | 내용 | Tasks | 추정 |
|---|---|---|---|
| **Phase 1** | main에서 GUI 코드/테스트/빌드 파일 hard delete + pyproject 정리 | 3 | 0.5일 |
| **Phase 2** | UI helper 모듈 4개 신설 (header / progress / errors / tables) — 단위 테스트 우선 | 8 | 2일 |
| **Phase 3** | InteractiveReviewer 전면 재작성 — UI helper 통합, 6 HITL 게이트 모두 헤더+legend+에러 박스 적용 | 6 | 2일 |
| **Phase 4** | `documatch preview` 서브커맨드 + 통합 테스트 | 3 | 1일 |
| **Phase 5** | `--quiet`/`--json`/`--summary` 플래그 + AutomationReviewer 통합 | 3 | 0.5일 |
| **Phase 6** | README + CHANGELOG + version bump 0.3.0 + tag + push | 3 | 0.5일 |

**총 ~26 task / 약 6.5일** (1주~1.5주, 풀타임 기준)

## 7. 위험 요소

| 위험 | 완화 |
|---|---|
| Rich Live가 일부 터미널(특히 Windows cmd)에서 깨짐 | `Console(force_terminal=True)` 명시 + Windows에서 ASCII 폴백 (스피너 → 점) |
| 한글 폭 계산 깨짐 | Rich는 동아시아 문자 너비 자동 처리. 단위 테스트에서 확인 |
| 기존 38 InteractiveReviewer 테스트 변경 부담 | 헬퍼 분리해서 핵심 동작은 변경 0, 시각 출력만 추가 — 대부분 그대로 통과 예상 |
| GUI 코드 삭제 시 외부 의존자 | 외부 사용자 없음 (private repo) — breaking change OK |
| Rich 버전 호환 | rich>=13.7로 pin (이미 의존성에 있음) |

## 8. 참조

- v0.1.0 설계: `docs/superpowers/specs/2026-05-09-documatch-cli-design.md`
- v0.2.0 GUI 설계 (archive): `docs/superpowers/specs/2026-05-09-documatch-desktop-design.md`
- 핸드오프 doc: `docs/superpowers/specs/2026-05-09-documatch-cli-handoff.md`
- 코드 repo: https://github.com/firefly0731/documatch-cli
- Rich Library: https://rich.readthedocs.io/
