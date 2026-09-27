# PiG 学習用 読み順ノート

この文書は **`personal` ブランチ専用**の学習メモです。PiG を「読んで学ぶ」ための順番と、各資料から何が学べるかのメモ。Stock PiG のドキュメントではないため upstream（`MichaelKinsy/PiG`）へは出しません。

## 前提

- PiG は Pi（TypeScript）の Go 移植。**upstream Pi が常に「正解」**で、PiG はそこに寄せる。
- 追従対象の pin は `coding/pigversion/pigversion.go`、Pi ソースのミラーは `.upstream/current`（`make upstream-mirror` で取得。git 管理外）。
- **全部読まなくていい。** このリポジトリ自身が `docs/README.md` の「Task router」で「変える対象ごとに1本だけ読め」と指示している。読む量より「どの層の話か」を掴むのが大事。

## Phase 0 — 全体像（まず1時間）

| 読むもの | 何が学べるか |
|---|---|
| `README.md` | 何のプロダクトか、install、Pi の pin、パリティ方針 |
| `QUICKSTART.md` | 実際の使い方 |
| `docs/pig-architecture.md` | レイヤマップ、公開パッケージマップ、境界不変条件 |
| `docs/typescript-to-go-porting.md` | 「移植」の翻訳方針（worksheet / interface policy / differential loop） |
| `docs/README.md` | ドキュメントの router。「何を変えるとき何を読むか」 |

## Phase 1 — プロジェクトの憲法

| 読むもの | 何が学べるか |
|---|---|
| `AGENTS.md`（40KB） | ルールブック。Mission / boundary / porting rules / Done criteria / Commit hygiene。**エージェント向けに書かれた規約**の実例としても貴重 |
| `CONTRIBUTING.md`（4KB） | 貢献の原則、DCO、PR 要件、squash merge の説明 |
| `docs/project/CONTEXT.md` | 文脈 |
| `docs/project/repository-quality.md` | 「良いリポジトリ」の基準 |
| `CODE_OF_CONDUCT.md` / `GOVERNANCE.md` / `MAINTAINERS.md` | 運営 |

ポイント: AGENTS.md は「ルールをどう文章化して強制するか」の教材。Done criteria や Loop smells など、独学では見えない観点が並ぶ。

## Phase 2 — 移植の規律（この repo の一番おいしいところ）

| 読むもの | 何が学べるか |
|---|---|
| `DIVERGENCES.md`（92KB） | upstream と**意図的に**違う点の全記録。なぜ divergence を番号付きで管理するのか |
| `PORT_MAP.md`（114KB） | TS → Go の対応表。移植の設計判断 |
| `parity/README.md` | パリティとは何を・どう測るか。scenario / runner / coverage の構造 |
| `parity/coverage.md` | カバレッジの見方（paired scenario と unit evidence の区別） |
| `parity/scenarios/<family>/*.toml` | 実際の「観測可能な振る舞い」の契約 |
| `docs/additive-features.md` | PiG 独自追加の扱い |

ポイント: 「正解が upstream にある」前提での、差分管理・契約テスト・coverage の作り方。

## Phase 3 — ソースの地図

`git ls-files` の規模: `cmd/` 181, `coding/` 525, `ai/` 201, `agent/` 240, `tui/` 340, `internal/` 739, `extensions/` 57, `piglets/` 48, `parity/` 790。

| 読むもの | 何が学べるか |
|---|---|
| `cmd/pig/` | CLI エントリ。startup model の解決など |
| `coding/` | コーディングエージェント本体（例: `coding/model.go`） |
| `ai/` | provider SDK（OpenAI / Anthropic / Google…、auth storage、thinking level） |
| `agent/` | agent harness（例: `agent/harness/pico3`） |
| `tui/` | ターミナル UI |
| `internal/codingagent/` | 内部実装（model registry、auth filter など） |
| `extensions/sdk{,-ts,-py,-rs}` | 拡張 SDK（Go / TS / Python / Rust） |
| `piglets/standard`, `piglets/porter` | Piglet（固めたエージェント定義） |

読み方: 上から全部ではなく、**1本の縦の流れ**を追う。例: `cmd/pig` の起動 → `coding.BuildModel` → `ai.New*Provider` → リクエスト送信。

## Phase 4 — CI/CD とゲート

| 読むもの | 何が学べるか |
|---|---|
| `Makefile`（22KB） | ゲートの全体像。`check` / `check-core` / `verify` / `compliance` / `parity-family` |
| `.github/workflows/ci.yml` | 8 job を `result` に集約する必須チェックのパターン |
| `.github/workflows/security.yml` | CodeQL / gitleaks / dependency-review / source-assurance |
| `.github/workflows/scorecard.yml` + `make compliance` | OpenSSF Scorecard の運用 |
| `release-candidate.yml`, `ci-images.yml`, `npm-publish.yml`, `bootstrap.yml` | リリース・配布 |
| ruleset "Default"（GitHub 側） | 必須は `CI result` / `Security result`、署名・linear history・CodeQL |

ポイント: 「多数 job → 1つの `result` を必須にする」集約パターンと、**生成物ドリフト検出**（`interface-go-drift` / `docs-drift` / `coverage-drift` / `port-map-drift`）。生成物は手編集禁止で、生成器を回す。

## Phase 5 — セキュリティ・供給網

| 読むもの | 何が学べるか |
|---|---|
| `docs/threat-model.md` | 資産・脅威・緩和 |
| `docs/supply-chain.md` | SBOM / provenance / 署名 |
| `docs/project/compliance.md` | バッジと pin の出所 |
| `SECURITY.md` / `.gitleaks.toml` / `REUSE.toml` / `THIRD_PARTY_NOTICES.md` | ポリシーとライセンス |

## Phase 6 — エコシステム・上級

`docs/piglet-*.md`、`docs/extension-*.md`、`docs/runtime-cell-*.md`、`docs/pig-package-spec.md`、`piglets/standard/README.md`、`extensions/RUBRIC.md`。

## 実践 — 1本の PR を最後まで読む

題材は自分の PR #54 / #60 が最適（自分の判断のどこが効いたか分かる）。

1. issue → PR 本文 → diff → CI → review の順に読む。
2. `interface-go-drift` に刺された箇所（`parity/interfaces/pig-go.json`）を自分で再生成して差分を見る。
3. maintainer の squash 後 commit と自分の branch を diff して「何が残り、何が捨てられたか」を見る。

## 学びを回す

- 気づきは `docs/personal/worklog.md` に追記する（このファイルと同じディレクトリ）。
- 「なぜそうなっているか」は `.upstream/current` の Pi ソースと突き合わせる癖をつける。

## リンク

- 作業ログ・方針: `docs/personal/worklog.md`
- パリティ: `parity/README.md`
