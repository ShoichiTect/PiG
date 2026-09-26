# PiG personal worklog

この文書は **`personal` ブランチ専用**の作業ログと upstream 貢献の方針メモです。Stock PiG のドキュメントではないため、upstream（`MichaelKinsy/PiG`）へは出しません。

- Fork: https://github.com/ShoichiTect/PiG （public）
- 基点: `MichaelKinsy/PiG` の `main`（HEAD `385e275` 時点の pin `0.87.1`）

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

- **issue**: まだ出さない。出すときは `parity.yml`（"Difference from Pi"）を使う。内容は「問題＋再現＋Pi/PiG の観測差＋期待契約＋証跡」に絞り、実装方針の詳細は PR に置く。`pig verify` 出力を含めること。
- **PR は3分割**（上記の3変更をそれぞれ独立した topic branch から）。
- PR 本文はテンプレを埋める。Upstream/divergence は **「Matches upstream Pi」** をチェック（`DIVERGENCES.md` への追記は不要）。
- `Tracking:` には、まとめ issue を作ってからその URL を貼るか、bounded な修正として `No issue: <理由>` を書く。
- 頻度: CONTRIBUTING は「unattended / high-volume / unreviewed な issue・PR を送るな」と明記。**同一挙動ファミリはバッチ**して送る。最初の issue で「継続的に upstream したいが、まとめ方の希望はあるか」を先に相談する。
- DCO: 全コミットに `Signed-off-by`（`git commit --signoff`）。CLA は不要。
- 提出先: `MichaelKinsy/PiG:main` へ、`ShoichiTect:<branch>` から。

## 次のアクション（未着手）

1. issue 文案（`parity.yml`）を作成する。
2. まとめ issue を提出する（ユーザー判断）。
3. `fix/*` の各 topic branch から PR を提出する。
4. 追加修正は `personal` に積み、upstream 性のあるものだけ topic branch に移す。
5. upstream `main` の更新を `git fetch origin` して `personal` を rebase する（fork への push は `--force-with-lease`）。
