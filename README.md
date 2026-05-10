# DocuMatch — Design Docs for documatch-cli

이 레포지토리는 [`documatch-cli`](https://github.com/firefly0731/documatch-cli)의 **설계 명세(spec) + 실행 계획(plan)** 을 버전 관리하는 docs-only 레포입니다.

코드는 [firefly0731/documatch-cli](https://github.com/firefly0731/documatch-cli)에 있습니다.

## 왜 docs를 별도 레포로 두는가

documatch-cli는 100% 바이브 코딩(자연어 기반 AI-협업 개발)으로 진행됐고, 문맥 유지의 안정성은 **harness engineering** 워크플로 — `spec → plan → subagent 분기 실행 → 교차 리뷰 → atomic commit·tag` — 로 확보했습니다. 이 레포는 그 워크플로의 산출물(설계 결정·계획·핸드오프 노트)이 코드와 독립적으로 누적·버전 관리되는 곳입니다.

새 세션에서 코드(`documatch-cli`)와 본 docs 레포를 함께 열면, 누구든 직전 작업 맥락을 그대로 이어받아 작업을 계속할 수 있습니다.

## 구조

```
docs/superpowers/
├── specs/   # 설계 명세 (요구사항·아키텍처·결정사항)
└── plans/   # 단계별 실행 계획 (TDD task 나열, 각 step에 코드/명령 포함)
```

## 버전별 산출물

| 버전 | 명세 (spec) | 계획 (plan) |
|---|---|---|
| **v0.3.0** — CLI Polish + GUI Hard Removal | [2026-05-10-documatch-cli-v0.3-design.md](docs/superpowers/specs/2026-05-10-documatch-cli-v0.3-design.md) | [2026-05-10-documatch-cli-v0.3.md](docs/superpowers/plans/2026-05-10-documatch-cli-v0.3.md) |
| **v0.2.0** — Desktop GUI (시도 후 폐기) | [2026-05-09-documatch-desktop-design.md](docs/superpowers/specs/2026-05-09-documatch-desktop-design.md) | [2026-05-09-documatch-desktop.md](docs/superpowers/plans/2026-05-09-documatch-desktop.md) |
| **v0.1.0** — CLI 초기 버전 | [2026-05-09-documatch-cli-design.md](docs/superpowers/specs/2026-05-09-documatch-cli-design.md) | [2026-05-09-documatch-cli.md](docs/superpowers/plans/2026-05-09-documatch-cli.md) |

v0.3.1 patch notes(spinner 통합·refine 누적 컨텍스트·UI 표현 정리·다항목 분리 fix)는 v0.3 plan 끝부분의 `## v0.3.1 Patch Notes` 섹션 참조.

## documatch-cli 빠른 소개

수십~수백 건의 비정형 문서(제재공개안·계약서·송장 등)에서 일관된 항목을 Excel로 정리하는 반복 업무를 **Human-in-the-Loop 7단계 게이트** AI CLI로 자동화합니다.

- Code: https://github.com/firefly0731/documatch-cli
- Latest release: https://github.com/firefly0731/documatch-cli/releases/latest

## License

MIT
