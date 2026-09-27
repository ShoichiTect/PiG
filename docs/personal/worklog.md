# PiG personal worklog

この文書は **`personal` ブランチ専用**の作業ログと upstream 貢献の方針メモです。Stock PiG のドキュメントではないため、upstream（`MichaelKinsy/PiG`）へは出しません。

- Fork: https://github.com/ShoichiTect/PiG （public）
- 基点: `MichaelKinsy/PiG` の `main`。pin は `0.87.1`。`personal` は `ad717b3` 上に載せ替え済み（2026-09-26）。

## リモートとブランチ構成

```text
remotes
  origin -> MichaelKinsy/PiG   (upstream。fetch 専用。commit しない)
  fork   -> ShoichiTect/PiG    (自分の public fork。push 先)

branches
  main        = upstream のミラー。作業しない
  personal    = 常用ブランチ。下記3変更＋この文書を全部載せる（PR には出さない）
  fix/<topic> = PR 用。main から切った curated な変更のみ
```

運用ルール:

- 常用バイナリは `personal` から作る（stock と分けるなら `make install PIG_BIN=...`）。
- upstream へ出す変更は `personal` に一度積んでから、必要 hunk だけ topic branch へ移す。
- pig 固有の好み・実験は `personal` に残し、PR に出さない。
- fork は public。**秘密情報・API キー・私的設定は絶対に載せない**。
- upstream 側の操作は DCO（`git commit --signoff`）必須。`git push -f` は fork のみ、必要なら `--force-with-lease`。

## 運用フロー（確定版）

作業場はこの clone 1つ。`origin`（本家）は fetch 専用、`fork`（自分）は push 先。開発は clone 内で行い、fork は公開・PR の置き場。

```text
① 常用
   git fetch origin
   git switch personal
   git rebase origin/main      # 最新 upstream + worklog
   make install                # 常用バイナリ更新
   git push --force-with-lease fork personal

② 修正ができたら（upstream に出す価値があるもの）
   git fetch origin
   git switch -c fix/<topic> origin/main     # ★ personal ではなく origin/main から
   # 実装・テスト（make ci-contracts 等の前哨）
   git commit --signoff ...
   git push -u fork fix/<topic>

③ issue → PR
   gh issue create --repo MichaelKinsy/PiG ...     # 必要なら（挙動修正は推奨）
   gh pr create --repo MichaelKinsy/PiG --base main --head ShoichiTect:fix/<topic>
   # マージ後: fetch origin → personal を rebase → fork/main も追従
```

- PR 用ブランチは必ず `origin/main` から切る（`personal` から切ると worklog が混入する）。
- issue は毎回必須ではない。doc-only / mechanical は理由を書けば不要。挙動修正は issue 推奨。
- 同一挙動ファミリはバッチする。
- 提出前に `origin/main` へ rebase（ruleset が base 最新化を必須とする）。
- `fork/main` は見た目用ミラー。作業には関与しない。

## 完了した変更（3分割）

### 1. `fix/list-models-catalog-providers` — model 一覧の認証フィルタをカタログ由来に

- 原因: `internal/codingagent/auth_filter.go` が「認証済みプロバイダ」を8件にハードコード。`--list-models` / `ctrl+p` cycle / `/scoped-models` がそこでフィルタし、env キーを持つカタログ provider（opencode-go, deepseek, xai, …）が出なかった。
- upstream: `model-runtime.ts` の `runAvailabilityRefresh` が `checkAuth()` を**全プロバイダ**に対して実行して `available` を導出する。
- 修正: `ReachableProviders` を `ai.ListRuntimeProviders()` + `ollama` から導出。`AuthenticatedProviders` を `hasEnvAuth || HasAnyKey || stored credential` の全カタログ走査に一般化。
- 対象ファイル:
  - `internal/codingagent/auth_filter.go`
  - `internal/codingagent/auth_filter_test.go`
  - `cmd/pig/model_list_catalog_providers_test.go`
  - `parity/scenarios/providers-registry/06-list-models-catalog-env-providers.toml`
- 証跡: `OPENCODE_API_KEY=x` で Pi/PiG の `--list-models` がバイト一致（104 行）。他に deepseek / xai / cerebras / moonshot / radius / bedrock / vertex / 保存済み OAuth(anthropic) でも一致。

### 2. `fix/list-models-no-auth-message` — 認証なしメッセージを upstream 準拠に

- 原因: `cmd/pig/model.go` の `printModelList` が独自文字列 "No models available. Set API keys in environment variables." を出力。
- upstream: `cli/list-models.ts` が `formatNoModelsAvailableMessage()`（`auth-guidance.ts`）を出力。
- 修正: `codingagent.FormatNoModelsAvailableMessage()` を使う。
- 対象ファイル:
  - `cmd/pig/model.go`
  - `cmd/pig/model_list_no_auth_message_test.go`

### 3. `fix/login-api-key-catalog` — `/login` API キー一覧をカタログ由来に

- 原因: `ai/api_key_providers.go` の手書き18件リスト。opencode-go 等が欠落し、名前も upstream とズレ（`Google Gemini` vs `Google`、`ZAI` vs `Z.AI` など）。加えて `internal/codingagent/interactive_commands.go` の `buildAuthProviderName` に並行した Drift した switch があった。
- upstream: `interactive-mode.ts:getLoginProviderOptions("api_key")` が `modelRuntime.getProviders()` を `provider.auth.apiKey` でフィルタし、`provider.name` でソート。
- 修正:
  - `ai/provider_names.go`（新規）: upstream `provider.name` の正規マップ＋`ProviderDisplayName()`。
  - `ai/api_key_providers.go`: `ListRuntimeProviders()` × `BuiltinProviderAuth(id).APIKey != nil` から導出、`localeCompare` 相当でソート。
  - `internal/codingagent/interactive_commands.go`: `buildAuthProviderName` を `ai.ProviderDisplayName` に委譲。
  - `ai/api_key_providers_test.go`（カタログ網羅・ソート順の回帰テスト）。
  - `parity/scenarios/selectors/11-login-api-key-providers.toml`。
- 証跡: 導出結果は upstream の40プロバイダと名前・順序とも一致。`/login → API key` の先頭ページと `(1/41)`（組み込み llama 含む）が Pi/PiG で一致。

### 生成物

- `AGENTS.md` / `parity/coverage.md` は `make coverage` の生成物（手編集しない）。`personal` では両シナリオ反映済み。topic branch では各ブランチで再生成する。

## 検証結果

- `ai` 全テスト PASS。
- パリティ（Node 24.19.0 使用）:
  - `selectors` 10/10 PASS
  - `oauth` 8/8 PASS
  - `providers-registry` 6/6 PASS
- ゲート: `go build ./...` / `go vet` / `make lint-changed`（0 issues）/ `make lint-scenarios` / `make coverage` / `make source-hygiene` / `make divergence-guard` / `go fix -diff` 通過。
- 赤→緑: 追加テストは pristine HEAD で FAIL することを worktree で確認済み。
- **既知の既存不具合（今回と無関係、main でも失敗）**: `TestBuildCommandReportsRelativeTargetFromNestedCheckout` / `TestShellResultRendersExecutedTruncation`。PR 本文に「pre-existing, also fails on main」と明記する。

## 環境メモ

- Go: `go.mod` の toolchain `go1.27.1`。`make doctor` は auto-toolchain のため go missing を出す。必要なら `make setup`。
- Node: README 要求は **24.19.0**。Node 26 だと拡張サブプロセスが `register handshake` で落ちる。mise で導入済み:
  ```bash
  export PATH="$HOME/.local/share/mise/installs/node/24.19.0/bin:$PATH"
  # 恒久化するなら: mise use -g node@24.19.0
  ```
- パリティ実行は Makefile 経由（`PI_PACKAGE_ROOT` / `PIG_PARITY_PI_BIN` を export する）で行う。`go test` を直接叩く場合はこの2つを渡すこと。

## issue → PR の方針（決定事項）

- **issue #53 提出済み**: https://github.com/MichaelKinsy/PiG/issues/53 （`parity.yml` の項目立て、`#50` deepseek を related としてリンク）。issue は **1本**に集約した。3サーフェスは「カタログの手書きサブセット」という単一の根因のため。
- **過去 PR の観測**: maintainer は #41/#42/#43 でテーマ単位に強くバンドル（stacked 含む）。issue↔PR リンクは 60 PR 中 #45 の1件のみで「1 issue = 1 PR」の慣習は無い。issue は feature 単位で粗い（総数3件）。
- **PR は1本バンドルを推奨**。PR 用ブランチ `fix/provider-catalog-availability`（**1コミット**、worklog を含まない）を `main` から作成し fork に push 済み。`personal` は3コミット＋worklog のまま常用用に残す。`fix/*` の3 branch は分割を望まれた場合の fallback。
- **本家 `main` の保護ルール（ruleset "Default"）**: `allowed_merge_methods=[squash]` のみ / 承認1件 / 必須チェック "CI result"（`strict_required_status_checks_policy=true` で up-to-date 必須）/ `required_linear_history` / `required_signatures`（squash 時に GitHub が署名）/ CODEOWNERS 自動レビュー。→ 出す前とレビュー中は `origin/main` に rebase。merge commit は作らない。squash されるので PR タイトルが main のコミット件名になる。
- **提出タイミング**: maintainer が「素晴らしいレポート、PR を開いて、0.2.1（最終統合中）に入れたい」と返信（22:18Z）。24時間待ちは不要となった。in-flight の post-login 変更が `ai/api_key_providers.go` に触れており、**カタログ導出版を優先**して解消される。
- **PR #54 提出済み**: https://github.com/MichaelKinsy/PiG/pull/54 （base `main`、head `ShoichiTect:fix/provider-catalog-availability`、`Tracking: #53`）。`origin/main`（`25c740a`）に rebase 済み。state OPEN / `MERGEABLE / BLOCKED`（レビュー・必須チェック待ち）。
- **issue #53 に追加コメント済み**: PR リンク、対象ファイル、証跡（byte-equal / `(1/41)`）、スコープの精密化、in-flight 変更があれば rebase する旨。
- PR 本文はテンプレを埋める。Upstream/divergence は **「Matches upstream Pi」** をチェック（`DIVERGENCES.md` への追記は不要）。
- 頻度: CONTRIBUTING は「unattended / high-volume / unreviewed な issue・PR を送るな」と明記。**同一挙動ファミリはバッチ**して送る。
- DCO: 全コミットに `Signed-off-by`（`git commit --signoff`）。CLA は不要。
- 提出先: `MichaelKinsy/PiG:main` へ、`ShoichiTect:<branch>` から。

## CI の失敗と interface inventory 修正（追記）

- PR #54 の初回 CI で `Linux / contracts` のみ失敗: `interface-go-drift`。`parity/interfaces/pig-go.json`（Go パッケージシンボルの生成 inventory）が未更新だった（`ai.ProviderDisplayName` 追加、`builtInAPIKeyProviders` → `builtinProviderNames`、`compareAPIKeyProviderNames`）。
- 原因: **ローカルで `make check`（CONTRIBUTING step 8）を回していなかった**。`make check` → `check-core` → `check-contracts-fast` → `interface-go-drift` で検出できた。`make coverage` は parity dashboard 専用で interface inventory は更新しない。
- 修正: `go run ./parity/cmd/gointerfaces -out parity/interfaces/pig-go.json`。`make ci-contracts` をローカルで全 PASS 確認。
- **メンテナが同じ修正を PR ブランチに直接 push（`46867cc`）**。内容は同一。重複コミットは rebase で破棄しリモートに同期（force-push せず）。PR の新 CI 実行は `action_required`（fork PR のため maintainer の実行承認待ち）。
- `personal` にも同じ inventory 修正を反映。

## PR #54 マージ（確定）

- **PR #54 は 2026-09-26 に squash マージ済み**。`origin/main` = `ad717b3 fix: derive provider availability surfaces from the provider catalog (#54)`（14 files, +546/−177）。
- マージコミットには署名2つ（ShoichiTect / Michael Kinsy）。メンテナが `ProviderDisplayName` / `FormatNoModelsAvailableMessage` を `parity/interfaces/pig-go.json` に追加する inventory 再生成を maintainer edit で実施（我々のローカル修正と同一）。
- マージ時に coverage dashboard（`AGENTS.md` / `parity/coverage.md`）も更新。`providers-registry` 6（`06` は `registration-only` タグ通り weak）、`selectors` 10（`11` は behavioral）。
- メンテナは「interface inventory の点を project documentation でも明確化する」とコメント。
- **issue #53 は OPEN のまま**（PR は `Tracking: #53` で自動クローズされない）。

## 運用ルール（再発防止）

1. push 前の必須ローカル前哨: `make ci-contracts`（生成 inventory のドリフトを検出）。推奨セット: `make ci-build ci-contracts ci-drift ci-closure ci-parity`。
2. エクスポート識別子やパッケージ境界を変えたら生成 inventory を再生成: `go run ./parity/cmd/gointerfaces -out parity/interfaces/pig-go.json`。
3. macOS では `make check` 全体はホスト依存テストで落ちる（internal/experimental, tui, tests/ci-images, TestShellResultRendersExecutedTruncation 等）。フル再現は devcontainer（Ubuntu 24.04 + Go 1.27.1 / Node 24.19.0 / Python 3.12 / Rust 1.97.1）で `make check`。最終判定は upstream PR CI。
4. fork ブランチへ push 前に `git fetch fork` して maintainer の直接 push を確認し、あれば rebase で合わせる（`git push -f` は禁止）。
5. fork では事前 CI を回せない（`ci.yml` は pull_request / schedule / workflow_dispatch のみ）。自動判定は PR CI だけ。
6. `make coverage` は parity dashboard 専用で `make check` の代替にならない。

## 次のアクション

1. [x] issue 文案（`parity.yml`）を作成。
2. [x] issue 提出 → **#53**。
3. [x] PR 提出 → **#54**（`Tracking: #53`）。issue #53 に PR リンクと証跡をコメント。
4. [x] CI 修正（`interface-go-drift`）: inventory 再生成。メンテナが同修正を PR ブランチに push（`46867cc`）したためリモートに同期。
5. [x] CI "CI result" → contracts の inventory 修正後にマージ（`ad717b3`）。
6. 追加修正は `personal` に積み、upstream 性のあるものだけ topic branch に移す。
7. [x] `personal` を `origin/main`（`ad717b3`）へ載せ替え。upstream に入った fix / coverage / inventory コミットは破棄し worklog のみ残す。
8. [次] issue #53 に「#54 で解決」とコメントしてクローズを促す。
