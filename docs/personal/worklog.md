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
- Node: README 要求は **24.19.0**。Node 26 だと拡張サブプロセスが `register handshake` で落ちる（原因は `module.register()` の DEP0205。下記フォローアップ候補 5）。
  `~/dev/pig` 配下だけ 24.19.0 になるよう **mise + direnv** で固定済み（2026-09-27）:
  - `~/dev/pig/mise.toml`: `[tools]` `node = "24.19.0"`
  - `~/dev/pig/.envrc`: `eval "$(mise direnv)"` + `use mise`（direnv は `~/.zshrc:118` で hook 済み。`direnv allow ~/dev/pig` 済み）
  - `~/.zshrc` は変更していない。ホームや他プロジェクトは従来どおり（Homebrew Node 26）。
  - **pi の bash ツールは `/bin/bash`** で direnv 非適用 → エージェント経由でテストする時は手で通す:
    ```bash
    export PATH="$HOME/.local/share/mise/installs/node/24.19.0/bin:$PATH"
    ```
- パリティ実行は Makefile 経由（`PI_PACKAGE_ROOT` / `PIG_PARITY_PI_BIN` を export する）で行う。`go test` を直接叩く場合はこの2つを渡すこと。

## issue → PR の方針（決定事項）

- **issue 53 提出済み**: https://github.com/MichaelKinsy/PiG/issues/53 （`parity.yml` の項目立て、`issue 50` deepseek を related としてリンク）。issue は **1本**に集約した。3サーフェスは「カタログの手書きサブセット」という単一の根因のため。
- **過去 PR の観測**: maintainer は PR 41/42/43 でテーマ単位に強くバンドル（stacked 含む）。issue↔PR リンクは 60 PR 中 PR 45 の1件のみで「1 issue = 1 PR」の慣習は無い。issue は feature 単位で粗い（総数3件）。
- **PR は1本バンドルを推奨**。PR 用ブランチ `fix/provider-catalog-availability`（**1コミット**、worklog を含まない）を `main` から作成し fork に push 済み。`personal` は3コミット＋worklog のまま常用用に残す。`fix/*` の3 branch は分割を望まれた場合の fallback。
- **本家 `main` の保護ルール（ruleset "Default"）**: `allowed_merge_methods=[squash]` のみ / 承認1件 / 必須チェック "CI result"（`strict_required_status_checks_policy=true` で up-to-date 必須）/ `required_linear_history` / `required_signatures`（squash 時に GitHub が署名）/ CODEOWNERS 自動レビュー。→ 出す前とレビュー中は `origin/main` に rebase。merge commit は作らない。squash されるので PR タイトルが main のコミット件名になる。
- **提出タイミング**: maintainer が「素晴らしいレポート、PR を開いて、0.2.1（最終統合中）に入れたい」と返信（22:18Z）。24時間待ちは不要となった。in-flight の post-login 変更が `ai/api_key_providers.go` に触れており、**カタログ導出版を優先**して解消される。
- **PR 54 提出済み**: https://github.com/MichaelKinsy/PiG/pull/54 （base `main`、head `ShoichiTect:fix/provider-catalog-availability`、`Tracking: issue 53`）。`origin/main`（`25c740a`）に rebase 済み。state OPEN / `MERGEABLE / BLOCKED`（レビュー・必須チェック待ち）。
- **issue 53 に追加コメント済み**: PR リンク、対象ファイル、証跡（byte-equal / `(1/41)`）、スコープの精密化、in-flight 変更があれば rebase する旨。
- PR 本文はテンプレを埋める。Upstream/divergence は **「Matches upstream Pi」** をチェック（`DIVERGENCES.md` への追記は不要）。
- 頻度: CONTRIBUTING は「unattended / high-volume / unreviewed な issue・PR を送るな」と明記。**同一挙動ファミリはバッチ**して送る。
- DCO: 全コミットに `Signed-off-by`（`git commit --signoff`）。CLA は不要。
- 提出先: `MichaelKinsy/PiG:main` へ、`ShoichiTect:<branch>` から。

## CI の失敗と interface inventory 修正（追記）

- PR 54 の初回 CI で `Linux / contracts` のみ失敗: `interface-go-drift`。`parity/interfaces/pig-go.json`（Go パッケージシンボルの生成 inventory）が未更新だった（`ai.ProviderDisplayName` 追加、`builtInAPIKeyProviders` → `builtinProviderNames`、`compareAPIKeyProviderNames`）。
- 原因: **ローカルで `make check`（CONTRIBUTING step 8）を回していなかった**。`make check` → `check-core` → `check-contracts-fast` → `interface-go-drift` で検出できた。`make coverage` は parity dashboard 専用で interface inventory は更新しない。
- 修正: `go run ./parity/cmd/gointerfaces -out parity/interfaces/pig-go.json`。`make ci-contracts` をローカルで全 PASS 確認。
- **メンテナが同じ修正を PR ブランチに直接 push（`46867cc`）**。内容は同一。重複コミットは rebase で破棄しリモートに同期（force-push せず）。PR の新 CI 実行は `action_required`（fork PR のため maintainer の実行承認待ち）。
- `personal` にも同じ inventory 修正を反映。

## PR 54 マージ（確定）

- **PR 54 は 2026-09-26 に squash マージ済み**。`origin/main` = `ad717b3 fix: derive provider availability surfaces from the provider catalog (PR 54)`（14 files, +546/−177）。
- マージコミットには署名2つ（ShoichiTect / Michael Kinsy）。メンテナが `ProviderDisplayName` / `FormatNoModelsAvailableMessage` を `parity/interfaces/pig-go.json` に追加する inventory 再生成を maintainer edit で実施（我々のローカル修正と同一）。
- マージ時に coverage dashboard（`AGENTS.md` / `parity/coverage.md`）も更新。`providers-registry` 6（`06` は `registration-only` タグ通り weak）、`selectors` 10（`11` は behavioral）。
- メンテナは「interface inventory の点を project documentation でも明確化する」とコメント。
- **issue 53 は OPEN のまま**（PR は `Tracking: issue 53` で自動クローズされない）。

## 運用ルール（再発防止）

1. push 前の必須ローカル前哨: `make ci-contracts`（生成 inventory のドリフトを検出）。推奨セット: `make ci-build ci-contracts ci-drift ci-closure ci-parity`。
2. エクスポート識別子やパッケージ境界を変えたら生成 inventory を再生成: `go run ./parity/cmd/gointerfaces -out parity/interfaces/pig-go.json`。
3. macOS では `make check` 全体はホスト依存テストで落ちる（internal/experimental, tui, tests/ci-images, TestShellResultRendersExecutedTruncation 等）。フル再現は devcontainer（Ubuntu 24.04 + Go 1.27.1 / Node 24.19.0 / Python 3.12 / Rust 1.97.1）で `make check`。最終判定は upstream PR CI。
4. fork ブランチへ push 前に `git fetch fork` して maintainer の直接 push を確認し、あれば rebase で合わせる（`git push -f` は禁止）。
5. fork では事前 CI を回せない（`ci.yml` は pull_request / schedule / workflow_dispatch のみ）。自動判定は PR CI だけ。
6. `make coverage` は parity dashboard 専用で `make check` の代替にならない。
7. **cross-reference 防止**: issue / PR 番号の前にシャープ記号を付けない。付けると GitHub がその issue / PR のタイムラインに「関連コミット」として自動表示し、個人メモが upstream の公開 issue に出てしまう。コミットメッセージでも worklog 本文でも「issue 53」「PR 54」のように番号だけで書く。

## 発見した問題（未着手の候補）

### 起動モデルの baseURL 欠落で最初のリクエストが 401（2026-09-27 調査）

- 症状: 新規セッション開始時、`enabledModels` から選んだモデルで最初の送信だけ 401。`/model` で同じ provider/model を選び直すと直る。
- 再現（実測）: auth.json に `opencode-go` のみ、models.json なし。`enabledModels = ["deepseek-v4.1-flash"]`。provider は `opencode-go` なのに baseURL が空 → `ai.NewOpenAIProvider` の既定 `https://api.openai.com/v1` に飛び、`oc_sk_...` が付いて OpenAI が 401。
- 原因: 起動経路 `cmd/pig/buildModelFromRef` が `registry.Resolve` を使う。これは models.json / 動的登録 / env キーしか返さず、組み込み provider の生成カタログ baseURL を返さない（`apiKind` だけ後で補完している）。一方 `/model` / `ctrl+p` の `coding.BuildModel` は `registry.ResolveGeneratedModel` を使うので正しい。
- 実測値: `buildModelFromRef` → `BaseURL=""`、`ResolveGeneratedModel` → `BaseURL="https://opencode.ai/zen/go/v1"`。
- 影響範囲: スイッチで baseURL を直書きしていない組み込み OpenAI 互換 provider 全部（`opencode`, `opencode-go`, `deepseek`, `zai`, `moonshotai`, `cerebras` 等）。upstream は単一の composed model 経路なので PiG 固有の乖離。
- 修正案: `buildModelFromRef` の entry 解決を `coding.BuildModel` と同じにする（生成カタログにあるモデルは `ResolveGeneratedModel`、無いものは従来の `Resolve`）。models.json の上書き優先は `ResolveGeneratedModel` が内部で維持する。
- 検証案: 単体（`buildModelFromRef` の BaseURL）+ 呼び出し側（`selectStartupModel`、main.go:1109 と同じ enabledModels 経路）。任意でパリティシナリオ。
- 状態: **issue 59 / PR 60 として提出済み**（下記「PR 60」）。上記の修正案（entry 解決だけ揃える案）は採用せず、解決と構築を分ける形にした。

## PR 60（起動モデルの baseURL / API 種別）（2026-09-27）

- issue: https://github.com/MichaelKinsy/PiG/issues/59 / PR: https://github.com/MichaelKinsy/PiG/pull/60（base `main`、head `ShoichiTect:fix/startup-model-client-kind`、`78e9ba5`、`Tracking: issue 59`、+239/−499）。
- 調査で分かった追加事実: 起動経路は baseURL だけでなく**クライアント種別も誤る**。`buildModelFromRef` は provider ID で分岐し、`coding.BuildModel` は API 種別で分岐する。例: `opencode-go/minimax-m3` は `anthropic-messages`（`https://opencode.ai/zen/go`）だが、起動経路では OpenAI completions クライアントで送られていた。
- 設計: upstream に合わせて**解決と構築を分けた**。
  - 解決（`cmd/pig` `resolveStartupModelEntry`）: 完全一致カタログ → models.json 定義 → provider 既定フォールバック（`buildFallbackModel` 相当）→ registry entry。フォールバックと警告は解決側に残す。upstream の `buildFallbackModel` は `resolveCliModel` からしか呼ばれない。
  - 構築（`coding.BuildModelFromEntry`）: API 種別分岐・baseURL・鍵解決を起動と `/model` で共有。
  - `coding.BuildModel` にフォールバックを移す案は不採用（upstream の `/model` は未知モデルを構築しないので、公開 API に upstream に無い意味を足すことになる）。
- `/model` 側の唯一の変更: Azure OpenAI Responses に `entry.Env` を渡す（旧 CLI builder は渡していた。`TestResolveModel_ThreadsAzureScopedEnv`）。
- `coding.BuildModelFromEntry` は引数が `internal/codingagent.ModelEntry` なのでモジュール外から呼べない。PR 本文で「internal に移すことも可」と maintainer に委ねた。
- 回帰テスト: `TestStartupModelUsesCatalogBaseURL`、`TestStartupModelUsesCatalogAPIKind`（red は実装前の一時テストで確認）。
- PR 本文を修正（「/model behavior is unchanged」を訂正、`BuildModelFromEntry` の注記）。issue 59 本文を修正（現在形に、ダミー鍵での再現を注記、issue 50 との関連を追記）。

### PR 60 の CI 失敗（無関係と判断）

- 失敗は `Linux / test-fast` の `agent/harness/pico3` `TestSchedulerHoldReplacementRunsPendingRecord`（`got "orphaned", want "completed"`、0.00s）だけ。`Linux verification` と `CI result` はその集約。他は全部 pass。
- 無関係の根拠: `go list -deps ./agent/harness/pico3` の PiG 内依存は `coding/pigversion` のみで、PR の変更パッケージに依存しない。直近の失敗 CI 13 件にこのテストの失敗は無い（docs だけの PR でも別の flaky テストで CI は落ちている）。
- ローカルでは再現せず: `go test -race -cpu 1,2,4 -count=500 -run '^TestSchedulerHoldReplacementRunsPendingRecord$' ./agent/harness/pico3` → ok（12.1s）。`GOMAXPROCS=2 go test -race -count=20 ./agent/harness/pico3` → ok（219.0s）。
- 再実行は不可: fork 貢献者で push 権限なし（`permissions.push=false`）。PR にコメントして maintainer に再実行を依頼済み: https://github.com/MichaelKinsy/PiG/pull/60#issuecomment-5851894833
- CI を緑にするのに必要なのは maintainer の再実行だけ。下のフォローアップは CI とは別件。

## フォローアップ候補（PR 60 とは別件・未着手）

どれも PR 60 のマージを止めない。調査・issue・実装・PR が必要になる可能性がある。

1. **pico3 の flake**（優先度: 高。他の PR の CI も赤くする）
   - 仮説（未検証）: `openEnv` → `h.Resume()` が `reconcileOrphans` を非同期 goroutine で走らせる（`agent/harness/pico3/harness.go:351-353`）。テストは `resumeDone` を待たないので、その goroutine が `off()` と `RegisterTaskKind(replacement)` の間に走ると、kind 未登録の live task が orphaned になる。
   - 調査: `reconcileOrphans` に一時的な遅延を入れて再現させるのが最短。
   - issue: maintainer が PR コメントに反応しない、または同じ失敗が再発したら起票。
   - 実装: テスト側の問題なら `resumeDone` を待つ小修正。スケジューラ本体の競合なら maintainer 判断になる可能性が高い。
2. **`coding.BuildModel` が別 provider の caps を借りる**
   - バグは確認済み: `lookupGeneratedModel` が `ai.LookupModel`（bare-id フォールバック付き）を使うため、`github-copilot/gpt-4o` が openai の `gpt-4o`（128000）を借りる。`ai.LookupModelExact` の doc と `TestResolveModel_UnknownUnderProvider_FallsBackAndWarns` はこれを禁止している。
   - 要決定: カタログに無い spec を `/model` が受け取ったとき、エラーにするか caps 0 で通すか。upstream の `/model` にはこの入力自体が無いので PiG 側の仕様になる。
   - 起票前に重複確認が必要（この件ではまだ検索していない）。
3. **Copilot ログイン後の既定モデル**
   - `internal/codingagent/interactive_auth.go:788` が `github-copilot/gpt-4o` を固定文字列で構築している。「upstream は `defaultModelPerProvider[providerId]`（copilot 既定 `gpt-5.4`）を選ぶ」は別エージェントの報告で、upstream ソースでは未確認。
   - 2 を直すとこの箇所の挙動も変わる（借用した caps → caps 0）。2 と 3 は1つの issue/PR にまとめるのが自然かもしれない。
4. **起動時の thinking level の clamp が upstream と違う**（2026-09-27 発見）
   - 場所: `internal/codingagent/interactive_thinking.go` の `initThinkingLevel()`。`idx := min(max(slices.Index(levels, start), 0), maxIdx)` で、サイクル上のインデックスを 0〜maxIdx に丸めているだけ。PiG には `ai.ClampThinkingLevel`（`ai/model_utils.go:92`）があるのに使っていない（コードで確認済み）。
   - upstream: `packages/ai/src/models.ts` `clampThinkingLevel` は、要求レベルがサポートされていればそのまま返し、無ければ `EXTENDED_THINKING_LEVELS` を要求位置から上へ、次に下へ探して最も近いサポート済みレベルを返す。別エージェントの報告では、upstream の初回設定も `_getThinkingLevelForModelSwitch` もこれを通る（upstream 側の呼び出し経路は私は未確認）。
   - 症状（別エージェントの報告。表の値は私は再測定していない）: 「medium が穴、xhigh 未定義、max あり」のモデル（例: `deepseek-v4.1-flash`）で、開始レベルが次のようにずれる。

     | 開始 | PiG | upstream |
     |---|---|---|
     | medium | low | high |
     | low | low | low |
     | high | high | high |
     | max | high | max |

   - 暫定回避: `~/.pig/agent/settings.json` に `defaultThinkingLevel: "high"`（実害は消えている）。
   - 方針: upstream の方が正しいので divergence 登録はしない。直すなら `initThinkingLevel` の clamp を `ai.ClampThinkingLevel` に置き換え、赤→緑の回帰テストを付ける。
   - 取り組むときは `git fetch origin && git switch -c fix/<topic> origin/main`（PR 60 の変更には依存しない想定）。
   - issue/PR にはまだ出さない。着手前に、表の値の再測定・upstream の呼び出し経路の確認・重複確認をする。

5. **Node 26 で TS 拡張ホストが壊れる（`module.register()` の非推奨 / DEP0205）**（2026-09-27 発見。優先度: 中）
   - 症状: Node 26 で TS 拡張のロードが失敗する（`TestHost_Integration_TS*`、`TestNodeRuntime*`、`TestStartupReusesPreTrustExtensionsAcrossEntrypoints` などが落ちる）。
   - 原因（確認済み）: `coding/extension/host/subprocess/runtime-node/register-loader.mjs` が `node:module` の `register()` を使う。Node **v26.0.0** で runtime deprecation（**DEP0205**、`Use module.registerHooks() instead`）。手元実測: Node 24.19.0 は警告なし、26.7.0 は DEP0205 警告が出て register handshake が失敗（`connection closed before register ... EOF`）。
   - 移行の難易度: `register()` は非同期フックを別ローダースレッドで実行、`registerHooks()` は同期フックを同一スレッドで実行。`loader.mjs` は `export async function resolve/load`（中で `await stat`/`readFile`）なので、`registerHooks()` 化には sync 版（`statSync`/`readFileSync`）への書き換えが必要。1 行置換ではない。
   - もう一つの論点: `extensions/sdk-ts/package.json` の `engines.node = ">=22.19.0"`（上限なし）が実態と乖離。`.node-version` = 24.19.0、CI もそれを使用（`ci.yml` 冒頭コメント「Node from `.node-version`」）。宣言上は 26 も可に見えるが実際は壊れる。
   - 未確定: Node 26 で **なぜ** handshake が失敗するか（DEP0205 警告が stderr に混じって壊すのか、`register()` の挙動自体が変わるのか）。issue に書くなら機序は断定しない。
   - 対応候補: (a) `registerHooks()` へ移行して 26 対応、(b) `engines` に上限を入れて「24 系のみ」を明示。upstream に **独立 issue**（PR 60 とは無関係）。
   - 着手前に: Node 公式（`deprecations.html` の DEP0205 / `module.html` の `register`・`registerHooks`）で再確認、upstream に既存 issue がないか重複確認。
   - ローカル対処: 上記「環境メモ」の mise+direnv で `~/dev/pig` を 24.19.0 に固定済み。

進め方: まず PR 60 の再実行とレビューを待つ。並行して 1 の仮説を検証する。

## 次のアクション

1. [x] issue 文案（`parity.yml`）を作成。
2. [x] issue 提出 → **issue 53**。
3. [x] PR 提出 → **PR 54**（`Tracking: issue 53`）。issue 53 に PR リンクと証跡をコメント。
4. [x] CI 修正（`interface-go-drift`）: inventory 再生成。メンテナが同修正を PR ブランチに push（`46867cc`）したためリモートに同期。
5. [x] CI "CI result" → contracts の inventory 修正後にマージ（`ad717b3`）。
6. 追加修正は `personal` に積み、upstream 性のあるものだけ topic branch に移す。
7. [x] `personal` を `origin/main`（`ad717b3`）へ載せ替え。upstream に入った fix / coverage / inventory コミットは破棄し worklog のみ残す。
8. [x] issue 53 へのコメントは行わない（ユーザー判断）。issue は OPEN のまま。
9. [x] 起動モデルの baseURL 欠落 → issue **59** / PR **60** を提出。PR・issue 本文を修正済み。
10. [ ] PR 60: maintainer による CI 再実行とレビューを待つ（pico3 flake 以外は全部 pass）。
11. [ ] pico3 flake の仮説検証（上記「フォローアップ候補」1）。
12. [ ] フォローアップ候補 2・3 の重複確認・upstream 確認・起票。
13. [ ] フォローアップ候補 4（thinking level の clamp）の再測定・upstream 経路確認・重複確認。直すかは未決定。
