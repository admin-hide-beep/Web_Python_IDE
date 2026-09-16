# IDEウェブアプリ 詳細仕様書

- 製品名例: `Pyodide_IDE`
- 現行バージョン表記: **`v1.12.041`**（`<title>Pyodide_IDE(v1.12.041)</title>`）
- 配布成果物: `02_deliverables/index.html`（Host + sandbox 埋め込みの**単一 HTML**）
- 開発ソース: `02_deliverables/src/`（Host / IDE をファイル分割）。`02_deliverables/build.py` で単一 HTML に連結
- GAS 連携雛形: `02_deliverables/gas_integration/`（ランチャー + `gas_call` ブリッジ）
- 言語: UI は `lang` クエリで日本語 / 英語切替（既定は日本語）。コード識別子は実装慣習に従う
- 本仕様は現行実装に準拠する。差分が出た場合は実装を正とし、本書を追随更新する

---

## 1. コンセプト

### 1.1 IDE 部

- **ブラウザだけで動く Python IDE**（インストール不要）
- コード編集（Monaco Editor）と Python 実行（Pyodide）を提供する
- ホスティング想定:
  - `https://`（本番）
  - `http://localhost` / `http://127.0.0.1`（開発）
  - `file:///`（ローカル直接オープン。GitHub 連携は無効）
  - Google Apps Script（GAS）埋め込み（トップレベルでない場合、一部機能を非表示）
  - GAS ランチャー経由の **トップレベル起動**（`about:blank` + `document.write`）。`gas_call` はランチャー opener の `google.script.run` を Host が中継
- **セキュリティ境界**:
  - メイン IDE（Python 実行・エディタ・VFS）は **sandbox iframe** 内で動作する
  - ホストフレームは UI シェル・認証・特権 API（Serial / GitHub / 外部 POST / **GAS opener 中継**）を担当する
  - セルフ XSS や悪意あるスクリプトが iframe 内で動いても、ホストのセッション／トークンを直接奪いにくくする

### 1.2 GitHub 連携部

- リポジトリツリーを `Script_Explorer` に表示する
- PAT 手入力は禁止。OAuth ポップアップ認証（Authorization Code + PKCE）
- Cloudflare Workers が **OAuth コード交換のみ** 中継する
- **access_token はホストフレームのメモリ変数のみ**に保持し、IDE iframe には渡さない
- 開いている GitHub 由来タブの内容をまとめて Push できる

### 1.3 AI Agent 部

次の経路で LLM と IDE をブリッジする。

1. **Chrome 組み込み Gemini Nano**（Prompt API / Summarizer API・セッション型）
2. **Web 版 Gemini** + ブックマークレットによる `postMessage` ブリッジ
3. **外部 LLM API（BYOK）** — OpenAI / Anthropic / Google AI Studio / xAI / Custom（OpenAI 互換）。API はステートレスなため Host が `messages[]` 履歴で文脈を保持する

LLM 応答から所定の JSON コマンドを抽出し、スクリプト実行・代理記憶・ユーザー検証などを行う。

---

## 2. システム構成

### 2.1 論理構成図

```text
┌──────────────────────────────────────────────────────────────────┐
│ Host Frame（信頼境界・トップレベル document）                      │
│  - CSP / ヘッダー UI / AI サイドバー / GitHub モーダル              │
│  - Web Serial / GitHub OAuth+API / fetch プロキシ                  │
│  - githubAccessToken（メモリのみ・非永続）                         │
│  - Gemini Nano / LLM API Agent（BYOK・messages 履歴）/ コマンド解析  │
│  - 外部 DnD 代理読み取り（Entry API）                              │
│  - #ide_container (sandbox iframe, srcdoc)                         │
└───────────────────────────────┬──────────────────────────────────┘
                                │ window.postMessage
┌───────────────────────────────▼──────────────────────────────────┐
│ IDE Frame（非信頼・sandbox / null origin 想定）                    │
│  - Monaco Editor / タブ / Outline                                   │
│  - Pyodide（VFS・実行・構文解析）                                  │
│  - Script_Explorer / VFS_Explorer                                  │
│  - Plot / Preview / Spreadsheet                                    │
│  - connect-src に api.github.com を含めない                         │
└──────────────────────────────────────────────────────────────────┘
           │                              │
           │                              └── Cloudflare Worker（OAuth のみ）
           └── CDN: Pyodide / Monaco / zip.js / jspreadsheet 等
```

### 2.2 物理構成（ソース分割 + 配布は単一 HTML）

#### 2.2.1 配布物（ランタイム）

ブラウザが読む成果物は原則 **1 ファイル HTML**（`02_deliverables/index.html`）+ アイコン等の静的アセット。  
実行時の信頼境界・起動手順は従来どおり（下記）。

| 領域 | 配置（生成後 `index.html`） | 役割 |
|------|------|------|
| Host HTML/CSS | 外側 document | ヘッダー、AI サイドバー、GitHub モーダル、ローダー、デザイントークン |
| Sandbox HTML | 外側の hidden `#sandbox_source` textarea 内に全文埋め込み | IDE 本体ソース |
| Host JS | `#script_ROOT` | 特権処理・ブリッジ・AI・GitHub・UI i18n |
| IDE JS | sandbox 内の複数 script ブロック | エディタ・Pyodide・Explorer・UI i18n 等 |

起動時ホストは次を行う。

1. `top_level_flag` / `dev_flag` / `uiLang` / `statefulMode` / `session_token` / `githubFeatureEnabled` を決定
2. ホスト静的 UI に `applyUiLanguage()` を適用し、`#nano-lang-select` 初期値を `uiLang` に設定
3. `#sandbox_source` の HTML 文字列を取得
4. `<head>` 直後に注入スクリプト `#host_injected_constants` を挿入  
   （`dev_flag` / `top_level_flag` / `githubFeatureEnabled` / **`uiLang`** / **`statefulMode`**）
5. `#ide_container.srcdoc = injected_html`
6. iframe `onload` 後に iframe を表示し、ローダーを消す（GitHub ログイン色の同期もここで実施可）
7. IDE 初回 `script_complete` 後、`statefulMode` なら Script_Explorer を LS から 1 回 restore

#### 2.2.2 開発ソースとビルド

null origin sandbox（`allow-same-origin` なし + `srcdoc`）のため、**実行時に IDE を相対パスの外部 JS として読み込む方式は採用しない**（CSP の `'self'` が opaque origin となり、親 origin のスクリプトと一致しない）。

代わりに開発時のみファイル分割し、ビルドで単一 HTML に戻す。

```text
02_deliverables/
  src/
    host/                 # Host CSS / body 等
      08_script_ROOT/     # Host 特権 JS（機能別 *.js、名前順連結）
    ide/                  # sandbox 用 HTML シェル・CSS・script_*
      11_script_MAIN/     # IDE メイン（機能別 *.js、名前順連結）
    manifest.json
  build.py                # python3 build.py → index.html
  index.html              # 配布用生成物
```

| ソース例 | 内容 |
|----------|------|
| `src/host/02_styles.css` | Host `#root_style` |
| `src/host/08_script_ROOT/*.js` | Host 特権 JS（i18n / GitHub / Serial / Agent / DnD 等）。ビルド時に名前順連結して `#script_ROOT` へ |
| `src/ide/02_styles.css` | IDE `#body_style` |
| `src/ide/11_script_global_variables.js` 等 | IDE 側の小 script ブロック（単一ファイル） |
| `src/ide/11_script_MAIN/*.js` | IDE メイン（message / tabs / explorer / dnd / pyodide 等）。ビルド時に名前順連結 |

**`08_script_ROOT/` 内訳（フェーズ2）**

| ファイル | 役割 |
|----------|------|
| `01_globals_i18n.js` | グローバル変数、UI i18n |
| `02_gemini_nano_base.js` | Gemini Window/Nano 基盤、system prompt、API 拡張コマンド契約 |
| `03_ui_elements.js` | header、サイドバーリサイザ、endpoint、button enable |
| `04_github.js` | GitHub OAuth / API（Host） |
| `05_sandbox_init.js` | srcdoc 初期化、Host→IDE ボタン中継 |
| `06_web_serial.js` | Web Serial |
| `07_agent_management.js` | Gemini Nano Local Agent |
| `07a_llm_api_agent.js` | 外部 LLM API（BYOK）クライアント・会話履歴コンテキスト・UI ルータ |
| `08_external_dnd.js` | 外部 DnD 代理読み取り |
| `09_message_handlers.js` | Host 汎用 `postMessage` 受信（`gas_call` ルーティング含む） |
| `10_gas_bridge.js` | GAS opener ブリッジ（`gas_call` 中継・タイムアウト・結果返送）。IIFE 終端 |

**`11_script_MAIN/` 内訳（フェーズ2）**

| ファイル | 役割 |
|----------|------|
| `01_message_handlers.js` | IDE 汎用受信、Agent Run / VFS tree / フェーズ A 転送 |
| `02_ui_controls.js` | Loading、button、GitHub ボタン UI、Reset VFS |
| `03_monaco_tabs.js` | Monaco、タブ |
| `04_file_explorer.js` | Script/VFS Explorer、読込、ツリー |
| `04a_agent_workspace.js` | API Agent フェーズ A（snapshot / search / read / replace / run current） |
| `05_syntax_outline.js` | 構文チェック、Outline |
| `06_resizer_layout.js` | リサイザ |
| `07_dnd.js` | Editor / VFS / Script DnD |
| `08_pyodide_execution.js` | Pyodide、RUN / STOP |
| `09_initial_load.js` | 初期ロード、Refresh |

**機能 → 主ファイル早見表（LLM / 局所改修用）**

改修時はまず本表で開くファイルを決め、原則それ以外の巨大ソースや生成物 `index.html` は読まない。  
パスはすべて `02_deliverables/src/` 起点。Host = `host/08_script_ROOT/`、IDE MAIN = `ide/11_script_MAIN/`。  
Host ↔ IDE の契約変更は**両側**をセットで確認する（片側だけの修正は取りこぼしやすい）。

| 機能・改修テーマ | 主に触るファイル | 必要に応じて |
|------------------|------------------|--------------|
| UI 言語 / i18n（Host 静的 UI） | Host `01_globals_i18n.js` | Host CSS `host/02_styles.css` |
| グローバル変数・フラグ初期値（Host） | Host `01_globals_i18n.js` | `ide/11_script_global_variables.js` |
| Gemini Window / Nano 基盤・system prompt | Host `02_gemini_nano_base.js` | Host `07_agent_management.js`、`07a_llm_api_agent.js`（API 用 prompt） |
| Host ヘッダ UI・AI サイドバー幅・endpoint・button enable | Host `03_ui_elements.js` | Host `02_styles.css` |
| GitHub OAuth / API / トークン（特権） | Host `04_github.js` | IDE `02_ui_controls.js`（ボタン表示）、IDE `04_file_explorer.js`（リポジトリファイル） |
| sandbox `srcdoc` 初期化・Host→IDE ボタン中継 | Host `05_sandbox_init.js` | IDE `01_message_handlers.js` |
| Web Serial | Host `06_web_serial.js` | IDE 側メッセージ受信が絡む場合は IDE `01_message_handlers.js` |
| Gemini Nano Local Agent（実行・管理） | Host `07_agent_management.js` | IDE `01_message_handlers.js`（Agent Run）、Host `02_gemini_nano_base.js` |
| 外部 LLM API Agent（BYOK・履歴コンテキスト） | Host `07a_llm_api_agent.js` | Host `02_gemini_nano_base.js`（`sendResponseToAgent` / API prompt）、`07_agent_management.js`（送信ルータ） |
| Agent フェーズ A（snapshot / search / read / replace） | IDE `04a_agent_workspace.js` + Host `02_gemini_nano_base.js` | IDE `01_message_handlers.js`、Host `09_message_handlers.js`、`04_github.js`（github: 読取） |
| 外部フォルダ DnD（Host 代理読取） | Host `08_external_dnd.js` | IDE `07_dnd.js`、Host `09_message_handlers.js` |
| Host 汎用 `postMessage` 受信・結果ハンドラ | Host `09_message_handlers.js` | 対応する IDE 送信側（多くは IDE `01_message_handlers.js`） |
| GAS `gas_call`（opener / google.script.run 中継） | Host `10_gas_bridge.js` | Host `09_message_handlers.js`、`ide/11_script_js_call_by_pyodide.js`、`gas_integration/` |
| IDE 汎用受信 / Agent Run / VFS tree / フェーズ A | IDE `01_message_handlers.js` | Host `09_message_handlers.js`、IDE `04a_agent_workspace.js` |
| Loading・IDE ボタン enable・GitHub ボタン UI・Reset VFS | IDE `02_ui_controls.js` | Host `04_github.js`（実 API） |
| Monaco エディタ・タブ CRUD / タブ DnD | IDE `03_monaco_tabs.js` | IDE `07_dnd.js`、`ide/02_styles.css` |
| Script / VFS Explorer・ツリー・ファイル読込 | IDE `04_file_explorer.js` | IDE `01_message_handlers.js`（VFS tree）、`04a_agent_workspace.js`、Host `04_github.js` |
| 構文チェック・Outline | IDE `05_syntax_outline.js` | IDE `03_monaco_tabs.js` |
| サイド / フッタ リサイザ・レイアウト | IDE `06_resizer_layout.js` | `ide/02_styles.css` |
| Editor / VFS / Script 内部 DnD・VFS mover | IDE `07_dnd.js` | Host `08_external_dnd.js`（外部から来る場合） |
| Pyodide 初期化・RUN / STOP | IDE `08_pyodide_execution.js` | `ide/11_script_js_call_by_pyodide.js`、`ide/11_script_initial_code.js` |
| 初期ロード・Refresh | IDE `09_initial_load.js` | IDE `04_file_explorer.js`、`08_pyodide_execution.js` |
| IDE グローバル変数ブロック | `ide/11_script_global_variables.js` | Host 注入定数は Host `05_sandbox_init.js` 付近の注入処理 |
| Pyodide から呼ぶ JS API（`js_post_message` / `gas_call` pending） | `ide/11_script_js_call_by_pyodide.js` | IDE `08_pyodide_execution.js`、Host `10_gas_bridge.js` |
| 初期表示用コード断片 | `ide/11_script_initial_code.js` | — |
| Host / IDE の見た目（色・配置 CSS） | `host/02_styles.css` または `ide/02_styles.css` | リサイザ論理は IDE `06_resizer_layout.js` |
| script ブロックの増減・連結順 | `src/manifest.json` + `build.py` | 対応する `10_*` / `12_*` / `13_gap_*` HTML 破片 |
| ビルド手順・生成物検証 | `build.py`（`python3 build.py` / `--check`） | 編集は `src/` のみ。`index.html` は生成物 |

```text
# 生成
cd 02_deliverables && python3 build.py

# 生成結果が現 index.html と一致するか（任意）
python3 build.py --check
```

- ビルドは標準ライブラリのみ（外部 pip / npm 不要）
- 編集対象は原則 `src/`。`index.html` を直接改変しない
- `08_script_ROOT/` / `11_script_MAIN/` は **ディレクトリ内 `*.js` をファイル名昇順で連結**（順序を変えると挙動が壊れる）
- フェーズ1–2は物理分割のみ（挙動・バージョン表記は `v1.12.034` のまま。連結後は分割前とバイト一致）

### 2.3 iframe sandbox / allow 属性

```html
<iframe id="ide_container"
  sandbox="allow-scripts allow-modals allow-downloads allow-forms allow-popups allow-presentation"
  allow="clipboard-read; clipboard-write; serial; file-system-read; file-system-write;">
</iframe>
```

- **`allow-same-origin` は付けない**（null origin 化し、ホスト origin のストレージ／Cookie へのアクセスを遮断）
- そのため File System Access の DirectoryEntry 等は IDE 内で制限される → **外部フォルダ DnD はホストが代理**する

### 2.4 プレビュー用 iframe（IDE 内）

HTML プレビュー用:

```html
<iframe id="preview_frame"
  sandbox="allow-scripts allow-forms allow-modals allow-popups allow-presentation"
  allow="serial; file-system-write; clipboard-read; clipboard-write;">
</iframe>
```

---

## 3. 外部依存（CDN）

| ライブラリ | 用途 | バージョン例（実装準拠） |
|------------|------|---------------------------|
| Pyodide | Python ランタイム | `0.29.0`（jsdelivr） |
| Monaco Editor | コードエディタ | `0.49.0`（cdnjs） |
| zip.js | ZIP 作成／復号 | `@zip.js/zip.js@2.7.53` |
| jspreadsheet-ce | CSV スプレッドシート | `4.13.3` |
| jSuites | jspreadsheet 依存 | `4.3.2` |
| micropip | Python パッケージ導入 | Pyodide 同梱 |

- script / link には可能な限り **SRI（integrity）+ crossorigin** を付与する
- 外部スクリプトの追加は方針上、ユーザー指示がある場合に限る

---

## 4. ホスティング条件と機能フラグ

### 4.1 `top_level_flag`

```js
top_level_flag = (window.self === window.top)  // クロスオリジンで top 参照失敗時は false
```

`false`（iframe 埋め込み / GAS 想定）のとき、ホストは次を非表示にする。

- Drive_Dir / Download_html / AI サイドバー開閉 / Serial 関連コントロール

### 4.2 `dev_flag`

```js
dev_flag = (URLSearchParams の dev === "true")
```

`true` のときのみ許可する危険機能:

- HTML プレビュー（Blob URL で preview タブ）
- JavaScript タブの直接実行（`js_inject_script`）
- Python から `js_inject_script` を globals 公開

本番（`dev_flag=false`）では上記を拒否し、警告表示する。

### 4.3 `githubFeatureEnabled`

次をすべて満たすときのみ `true`。

1. `top_level_flag === true`
2. `protocol === "https:"` **または** `http:` かつ hostname が `localhost` / `127.0.0.1`
3. `file:` では常に `false`

`false` 時: GitHub ボタン非表示、OAuth/API 未起動。

### 4.4 セッション用 `session_token`

ホスト起動時に 32 文字のランダムトークンを生成（表示／コピー用途。永続化しない）。

### 4.5 ステートフルモード `statefulMode`（`state` クエリ）

```js
// URLSearchParams の state が文字列 "ful" のときのみ true
// キーは小文字 state のみ（State 大文字は無効）。"1" / "full" / "true" 等も無効
statefulMode = (urlParams.get("state") === "ful")
```

| 項目 | 仕様 |
|------|------|
| 決定タイミング | ホスト起動時に 1 回 |
| 情報源 | クエリパラメータのみ |
| Host → IDE | 注入定数 `const statefulMode = true \| false` |
| IDE 側 | `ideStatefulMode`（注入 `statefulMode === true` のときのみ true） |
| 既定 | 未指定・不正 → `false`（従来どおりメモリのみ） |

**有効時（`state=ful`）の挙動:**

1. Script_Explorer の **`local_upload` ツリー**（ファイル本文 Base64 + 展開パス）を Host の `localStorage` に保持する
2. IDE は null origin のため LS を直接使わず、postMessage で Host が代理 read/write する
3. キー: `Pyodide_IDE.scriptExplorer.v1`
4. 起動後（IDE の Explorer 初期化完了後の初回 `script_complete`）に 1 回 restore
5. Script ヘッダの **更新（⟳）ボタンを × に差し替え**。× の SVG は **`VFS をリセット` と同一**（viewBox 512 / filled polygon）。× で LS 削除 + 表示ツリーを空（confirm あり）
6. **`Ctrl+S` / `Cmd+S`**: アクティブタブ内容を Script_Explorer（`local_upload`）へ書き戻し、既存の persist 経路で LS へ保存（Blob ダウンロードはしない）
7. **対象外:** `github` / `local`（DirectoryHandle）/ VFS / エディタタブ全体の永続化 / GitHub token

**Ctrl+S（ステートフル時）の詳細:**

| 条件 | 挙動 |
|------|------|
| タブに `scriptPath` あり（Explorer から開いた） | そのパスの file ノードを更新 |
| `scriptPath` 無し・名前が一意 | 同名 file を更新 |
| 該当ファイル 0 件（ツリー空含む） | ルートに `file_name_val \|\| "python_script.py"` を新規作成 |
| 同名が複数 | 保存せず警告（Explorer から開き直しを促す） |
| 表示中が `github` / `local` | 保存せず案内メッセージ |
| spreadsheet | `getData()` の CSV 文字列を同様に書き戻し |

- タブは GitHub の `githubPath` と同様に **`scriptPath`** を保持する
- ヘッダー Save ボタンは従来どおり Blob ダウンロード（本モードでも変更しない）

**セキュリティ:**

- オプトインのみ。`state=ful` が無い限り Host は LS を読書きしない
- token 類は保存しない
- 同一オリジンの他タブ／共有 PC では Host origin の LS が読まれる点に注意
- Ctrl+S の書き戻し先はツリー内ノードまたはルート新規のみ（任意 path の捏造書き込みはしない）

### 4.6 UI 言語 `uiLang`（`lang` クエリ）

```js
// URLSearchParams の lang を小文字化
// "en" → "en" / それ以外（未指定・不正含む）→ "ja"
uiLang = (lang === "en") ? "en" : "ja"
```

| 項目 | 仕様 |
|------|------|
| 決定タイミング | ホスト起動時に 1 回（ページリロードで再評価） |
| 情報源 | クエリパラメータのみ（`localStorage` / `navigator.language` は使わない） |
| 言語切替 UI | **持たない**（ステートレス原則。ブックマークで `?lang=en` 等を固定） |
| ホスト → IDE | 注入定数 `const uiLang = "ja" \| "en"` |
| IDE 側 | `ideUiLang`（注入 `uiLang` が `"en"` のときのみ en、それ以外 ja） |
| `document.documentElement.lang` | ホスト / IDE 双方で `uiLang` / `ideUiLang` に同期 |
| `#nano-lang-select` | **初期 value のみ** `uiLang` に連動。セッション作成後は変更不可（既存仕様） |

#### 実装方式

- 同一 HTML 内の辞書オブジェクト + 取得関数 `t(key, vars?)`
  - ホスト: `uiMessages` / `t` / `applyUiLanguage`
  - IDE: `ideUiMessages` / `t` / `applyIdeUiLanguage`（sandbox は別 document のため辞書は二重定義）
- プレースホルダは `{name}` 形式の単純置換
- キー欠落時: 現在言語 → `ja` → キー文字列の順でフォールバック
- 静的 DOM: `data-i18n` / `data-i18n-title` / `data-i18n-placeholder` / `data-i18n-hint`
- 動的: `alert` / `confirm` / `prompt` / ステータス文言などで `t()` を直接呼ぶ
- AI ヒント: `data-i18n-hint="ja|en"` により **選択言語の一方のみ表示**
- Python（micropip 承認 confirm 等）から UI 文言を取る場合: `globalThis.__uiT`（= `t`）を IDE が公開

#### 対象 / 対象外

| 対象（ユーザー向け UI） | 対象外 |
|------------------------|--------|
| ラベル、プレースホルダ、ツールチップ | コードコメント |
| `alert` / `confirm` / `prompt` | `console.*` |
| ローディング・GitHub ステータス・Nano 状態表示 | 変数名・関数名・DOM id / CSS クラス |
| Agent のシステム通知メッセージ（ユーザー／LLM に見える定型文） | サンプル Python コード本文、ファイル名、`err.message` 原文 |

---

## 5. Content Security Policy（CSP）

### 5.1 ホストフレーム CSP（概要）

| ディレクティブ | 許可例 |
|----------------|--------|
| default-src | `'self'` + GAS オリジン |
| script-src | `'self' 'unsafe-inline' 'wasm-unsafe-eval'` + jsdelivr / cdnjs / GAS |
| style-src | `'self' 'unsafe-inline'` + CDN |
| connect-src | `'self'` + CDN + pypi + **api.github.com** + **OAuth Worker** + GAS + **LLM API**（openai / anthropic / generativelanguage.googleapis.com / x.ai）+ **localhost / 127.0.0.1**（ローカル LLM） |
| frame-src | `'self' blob: data:` |
| img-src | `'self' data: blob:` |
| worker-src | `'self' blob:` |

### 5.2 IDE（sandbox）CSP（概要）

| ディレクティブ | 要点 |
|----------------|------|
| connect-src | CDN / pypi 等のみ。**`api.github.com` は含めない** |
| script-src | inline + wasm + CDN |
| frame-src | `data: blob:`（プレビュー用） |

---

## 6. UI レイアウト仕様

### 6.0 ビジュアルデザイン（実装準拠）

ホスト / sandbox 双方に **CSS デザイントークン**（`:root` 変数）を持つ。

| トークン例 | 用途 |
|------------|------|
| `--bg-app` / `--bg-panel` / `--bg-elevated` / `--bg-editor` | 背景階層 |
| `--text-primary` / `--text-secondary` / `--text-muted` | 文字色 |
| `--accent` / `--success` / `--danger` | アクセント・状態色 |
| `--radius-*` / `--shadow-*` / `--transition-fast` | 角丸・影・遷移 |
| `--header-height` | ヘッダー高さ（ホスト 40px 前後） |

- ダーク基調の統一 UI（ヘッダー・タブ・サイドバー・リサイザ・フッター・モーダル・AI 吹き出し）
- フォーカス可視化・ホバーのマイクロインタラクションを含む

### 6.1 ホストフレーム

```text
┌ header ─────────────────────────────────────────────────────────┐
│ [≡Sidebar] [✦AI] [Save] [Stop] [Run] [Serial...] [Baud等]        │
└─────────────────────────────────────────────────────────────────┘
┌ main_wrapper ───────────────────────────────────────────────────┐
│  ┌ ide_container (iframe) ─┐  │ resizer │ AI Sidebar            │
│  │ flex: 1 1 0%; min-width:0│  │         │ (default 幅 目安 350) │
│  └─────────────────────────┘  │         │ Chat / Input          │
└─────────────────────────────────────────────────────────────────┘
```

- 初期: `body.ai-sidebar-hidden` により AI サイドバー非表示
- `#ide_container` は `flex: 1 1 0%` + `min-width: 0` で AI サイドバー表示時も余り幅を吸収
- AI サイドバー幅は縦リサイザで変更（最小幅目安 150px）
- プロンプト入力欄（`#nano-user-input`）の高さは上端ハンドル `#nano-input-resizer` で変更（既定・最小は従来の 1 行相当、offsetHeight 約 54px。上限はチャットが潰れない範囲・目安 280px / サイドバー高さの 45%）。セッション中のみ保持
- リサイズ中は iframe の `pointer-events: none` にし、ドラッグ追従切れを防ぐ

#### ヘッダーボタン（主要）

| ID | 動作 |
|----|------|
| `toggle_sidebar` | IDE 内サイドバー表示切替を postMessage |
| `toggle_ai_sidebar` | AI サイドバー表示/非表示 |
| `save_script` | IDE へ保存クリック転送 |
| `stop_script` | IDE へ停止転送 |
| `run_script` | IDE へ実行転送 |
| `serial_connect` / `serial_cut` | Web Serial 接続/切断（ホスト専任） |
| Serial 入力群 | baud / dataBits / parity / stopBits |

初期状態はヘッダー操作ボタンを **disabled**。IDE から `script_complete` を受けたら有効化。

### 6.2 IDE フレーム

```text
┌ container ──────────────────────────────────────────────────────┐
│ sidebar_area          │ main_content                            │
│ ┌ Script_Explorer ──┐ │ tab_bar                                 │
│ │ [GH][Upload][Ref] │ │ editor | plot | preview | spreadsheet   │
│ └───────────────────┘ │ footer_resizer                          │
│ resizer_h             │ footer: py_result | js_result | ext     │
│ ┌ VFS_Explorer ─────┐ │                                         │
│ │ [Up][Ref][Reset]  │ │                                         │
│ └───────────────────┘ │                                         │
│ resizer_h2            │                                         │
│ ┌ Outline ──────────┐ │                                         │
│ └───────────────────┘ │                                         │
└─────────────────────────────────────────────────────────────────┘
```

- サイドバー幅・Explorer 間高さ・フッタ高さはドラッグリサイザで調整
- フッター 3 ペイン（`#py_result_area` / `#js_result_area` / `#extension_area`）:
  - py / js は基本幅固定寄り、extension が grow して余りを吸収
  - padding を考慮した幅計算と `ResizeObserver` によるクランプ
- クラス `sidebar-hidden` でサイドバー全体を非表示可能
- ローディングオーバーレイ `#loading_overlay` で初期化・実行中をブロック

#### Script_Explorer ボタン

| ID | 機能 |
|----|------|
| `github_connect` | GitHub 連携開始要求（ホストへ `github_connect_request`）。`githubFeatureEnabled` 時のみ表示 |
| `script_upload` | フォルダ選択（`webkitdirectory`）で Script ツリー構築 |
| `refresh_script_explorer` | 現在ソース（local / upload / github）を再描画 or 再取得 |
| `script_mount` | File System Access マウント用（sandbox 制約で通常非表示） |

#### VFS_Explorer ボタン

| ID | 機能 |
|----|------|
| `vfs_upload` | ローカルフォルダを VFS `/mnt/vfs_root` 配下へ書き込み |
| `refresh_vfs_explorer` | VFS を再スキャンしてツリー再構築 |
| `reset_vfs_explorer` | VFS 内容を全削除してルートを再作成（確認ダイアログ必須） |

---

## 7. 初期化シーケンス

### 7.1 ホスト

1. `top_level_flag` / `dev_flag` / `uiLang` / `statefulMode` / `session_token` / `githubFeatureEnabled` を決定
2. `applyUiLanguage(document)` と `#nano-lang-select` 初期値設定
3. ヘッダーボタンを無効化
4. sandbox HTML に定数注入して `srcdoc` 設定（`uiLang` / `statefulMode` 含む）
5. iframe onload → iframe 表示・ローダー非表示
6. ヘッダーボタンは IDE の `script_complete` 受信後に有効化
7. 初回 `script_complete` かつ `statefulMode` のとき Script_Explorer を LS から 1 回 restore

### 7.2 IDE（`window.onload`）

1. 注入 `uiLang` / `statefulMode` から `ideUiLang` / `ideStatefulMode` を確定し、`applyIdeUiLanguage(document)`（`globalThis.__uiT = t` 公開）
2. GitHub ボタン UI セットアップ
3. サイドバーボタン無効 + ローディング ON
4. **`main_ini()`**: Pyodide ロード、micropip、VFS ルート作成、Python 変換関数定義、globals 公開、interrupt buffer
5. Monaco loader → `initialize_monaco_editor()`
6. タブ `+` ボタン、リサイザ、DnD セットアップ
7. Script / VFS 初期表示（ステートフル時は ⟳ を × に差し替え）
8. ホストへ `script_complete` 送信（Explorer 初期化**後**。restore が refresh で消えないようにする）
9. ローディング OFF、サイドバー有効
10. （非同期）Host から `script_explorer_state_restore` を受信したら local_upload ツリーを復元

### 7.3 初期コード

- GAS テンプレート埋め込み `<?= web_initial_code ?>` があればそれを使用
- プレースホルダ未置換時はサンプル Python（numpy 乱数加算）を `main.py` として開く

---

## 8. Monaco エディタとタブシステム

### 8.1 エディタ設定

- テーマ: `vs-dark`
- 初期言語: `python`
- `automaticLayout: true`
- 内容変更 debounced 300ms で `analyze_python_code`

### 8.2 タブデータ構造

```js
tabs_data[tab_id] = {
  name: string,           // 表示名 / 保存名
  model: MonacoModel|null,// type==="editor" のみ
  type: "editor" | "preview" | "plot" | "spreadsheet",
  content: any,           // preview は Blob URL 等
  scroll_state: any,
  githubPath: string|null // GitHub 相対パス（token は持たない）
}
```

- `active_tab_id`, `tab_counter` で管理
- 同名タブは再オープン時にアクティブ化（`forceUpdate` 時は本文上書き）

### 8.3 言語判定（拡張子）

| 拡張子 | language / type |
|--------|-----------------|
| `.py` | python / editor |
| `.json` | json / editor |
| `.js` | javascript / editor |
| `.html` | html / editor |
| `.css` | css / editor |
| `.txt` | text/plain / editor |
| `.csv` | csv / **spreadsheet** |
| その他 | text/plain / editor |

### 8.4 タブ操作

| 操作 | 仕様 |
|------|------|
| 左クリック | アクティブ化 |
| 中クリック / `[x]` | クローズ（Monaco model dispose） |
| ダブルクリック | 拡張子固定のリネーム（prompt） |
| ドラッグ | タブ並び替え |
| `+` ボタン | 新規 Untitled タブ |

### 8.5 表示モード切替（activate_tab）

同時に 1 つのメイン表示領域のみ表示:

- `editor` → `#editor`
- `preview` → `#preview_container` + iframe src
- `plot` → `#plot_container`
- `spreadsheet` → `#spreadsheet_container`（jspreadsheet インスタンスをタブごと保持）

### 8.6 保存

- ヘッダー Save: アクティブタブ内容を Blob ダウンロード（常に）
- `Ctrl+S`（`Cmd+S`）:
  - **ステートレス**（既定）: ヘッダー Save と同様に Blob ダウンロード
  - **ステートフル**（`state=ful`）: Script_Explorer（`local_upload`）へ書き戻し + Host LS 永続化（§4.5）。ダウンロードしない
- spreadsheet は `getData()` を CSV 文字列化
- モデル無し／タブ無しは no-op

---

## 9. Outline と構文チェック

対象: 言語が `python` のときのみ。

### 9.1 処理

1. Pyodide 上で `ast.parse`
2. 成功時: `OutlineVisitor` で class / function / async function を抽出（`__init__` は除外）
3. 失敗時: `SyntaxError` を Monaco markers（Error）に反映
4. Outline クリックで `revealLineInCenter` + キャレット移動

### 9.2 Outline 項目

```json
{ "type": "class|function", "name": "Foo", "line": 12, "level": 0 }
```

表示: `C: name` / `f: name`、level に応じたインデント。

---

## 10. Pyodide 実行仕様

### 10.1 公開 JS API（Python globals）

| 名前 | 用途 |
|------|------|
| `js_print` / `js_print_clear` | stdout 相当を `#py_result_area` へ |
| `js_add_plot_image` / `js_clear_plot_area` | matplotlib 画像表示 |
| `js_download_func` | ブラウザダウンロード |
| `js_encrypt_zipper` | VFS パスを ZIP 化して DL |
| `js_post_message` | ホスト／プレビューへ postMessage |
| `js_register_python_callback` | serial/preview 受信コールバック登録 |
| `js_get_csv_data` | スプレッドシート CSV 取得 |
| `js_map_global_data` / `js_obj_global_data` | JS↔Py 共有データ |
| `js_inject_script` | **dev_flag 時のみ** |

### 10.2 標準出力

実行前に Python で `sys.stdout` / `sys.stderr` を `Web_Stdout` に差し替え、`write` が `js.js_print` を呼ぶ。  
`js_print` は `#py_result_area` に追記し、長さが約 100,000 文字超で末尾優先に切り詰める（`max_print_len`）。

### 10.3 実行前トランスフォーム

#### (A) `apply_auto_sleep`

- AST 変換: **async 関数内**の `print(...)` の直後に `await asyncio.sleep(0)` を挿入
- 同期 `def` 内には挿入しない（SyntaxError 防止）
- パース失敗時は `"# Parse Error: ..."` を返し、**元コードで実行**

#### (B) `apply_auto_install`

- import を走査し、Pyodide 提供パッケージ集合と名前マッピングから必要パッケージを推定
- マッピング例: `sklearn→scikit-learn`, `cv2→opencv-python`, `PIL→Pillow` 等
- 未知モジュールは `confirm` でユーザー承認後に PyPI インストール試行対象へ
- 冒頭に `import micropip` + `await micropip.install('pkg')` を挿入

#### (C) matplotlib

- `micropip.install` をモンキーパッチし、`matplotlib` 導入時に `plt.show` を差し替え
- `show` 時: 軸メタデータ要約を print、PNG base64 を `js_add_plot_image`、`plt.clf()`

### 10.4 RUN ボタン分岐

| 言語 | 動作 |
|------|------|
| python | トランスフォーム → stdout リダイレクト → `runPythonAsync` |
| html | dev のみ Blob URL プレビュータブ。本番は拒否 |
| javascript | dev のみ `js_inject_script`。本番は拒否 |
| その他 | 未対応メッセージ |

実行中: ローディング ON、サイドバーボタン無効。  
終了時（finally）: stdout 復元、`script_complete` 送信、UI 復帰。

### 10.5 STOP

- `interrupt_buffer[0] = 2`（SIGINT）を短い間隔で複数回書き込み
- SharedArrayBuffer が使えればそれを使用、そうでなければ通常 ArrayBuffer（効きにくい場合あり）
- 3 秒ウォッチドッグで UI 強制解除 + `script_complete`

### 10.6 作業ディレクトリ

- VFS ルート: **`/mnt/vfs_root`**（定数 `MOUNT_PATH`）
- 初期化後 `os.chdir(MOUNT_PATH)`

---

## 11. ツリーデータモデル（Explorer 共通）

### 11.1 ノード共通形

```js
{
  name: string,
  type: "file" | "folder",
  children?: Node[],     // folder
  // 種別ごとの付加:
  script_path?: string,  // Script local/upload 論理パス
  handle?: File|FileSystemHandle, // local / local_upload
  vfs_path?: string,     // VFS 絶対パス
  path?: string,         // GitHub 相対パス
  sha?: string           // GitHub blob sha（任意）
}
```

### 11.2 ソート

フォルダ優先、同種は `localeCompare(name)`。

### 11.3 描画

- `render_file_explorer(tree, type, targetElement)` → `create_tree_level`
- 各行: `.item_row`、選択時 `.selected_item`
- フォルダ: `.folder-item` + `.expanded` / `.collapsed`
- 展開状態はメモリ `Set` で保持（下記）

### 11.4 展開状態維持

```js
explorerExpandedPaths = {
  script: Set<path>,
  vfs: Set<path>
}
```

| ソース | パスキー |
|--------|----------|
| VFS | `item.vfs_path` |
| Script local / local_upload | `item.script_path` |
| GitHub | `item.path`（ルートラッパーは `""`） |

- 開閉クリックで Set 更新
- 再描画時に Set を適用
- DnD 移動成功後はパスをリマップ（移動元 prefix → 新 prefix）
- 選択ハイライトの永続化は必須ではない

---

## 12. Script_Explorer 仕様

### 12.1 状態

```js
scriptExplorerState = { type: null | "local" | "local_upload" | "github", tree: Node[]|null }
githubExplorerActive: boolean
githubRepoContext: { owner, repo, branch } | null
```

### 12.2 local_upload（フォルダ Upload）

1. hidden `input[webkitdirectory]` で複数 File を取得
2. 各 File の `webkitRelativePath` からネストツリー構築
3. ファイルノードに `handle = File` を保持
4. `ensure_script_paths` で論理パス付与
5. クリックで `FileReader` / `file.text()` 等により本文を読み `open_tab`

### 12.3 local（DirectoryHandle マウント）

- File System Access API で DirectoryHandle を保持し再帰ツリー化
- sandbox 制約により実用は限定的。仕様上 type `"local"` をサポートする

### 12.4 github

- ホストが tree を送り `type: "github"` で描画
- ルートに `owner/repo@branch` ラッパーフォルダを被せる
- ファイルクリック → `github_file_request` → 受信本文でタブ open（`githubPath` 設定）
- GitHub 表示中は Script 外部 DnD 受け入れ不可（ホストへ disabled 通知）

### 12.5 内部移動（local_upload）

- DnD MIME: `application/script-path`
- `handle_script_move(src, destFolder)`
  - 自己／子孫への移動禁止
  - 同名衝突時は上書き確認
  - 展開パスリマップ後に再描画

### 12.6 パス正規化

- `\` → `/`、先頭 `/` 除去
- セグメントに `..` があれば拒否

### 12.7 Refresh

| type | 動作 |
|------|------|
| github | ホストへ `github_refresh_tree_request` |
| local_upload / local | 保持ツリーを再描画 |
| null | 案内メッセージ表示 |

**ステートフル時（`state=ful`）:** 更新ボタンは × に変わり、Refresh ではなく **クリア**（§4.5 / §12.8）を行う。

### 12.8 ステートフル永続化（`state=ful`）

#### 保存タイミング

- `set_script_explorer_tree` で `type === "local_upload"` かつ非空ツリーへ更新したとき（Upload / DnD マージ / 移動など）
- 空ツリー・type null 相当で local 系を捨てたとき → LS キー削除
- `github` / `local` へ切り替えても **LS 上の local_upload スナップショットは消さない**（× または local の空更新でのみ削除）
- 連続更新は世代番号で古い非同期シリアライズ結果を破棄

#### 保存フォーマット（Host `localStorage`）

```js
{
  version: 1,
  savedAt: "<ISO8601>",
  type: "local_upload",
  tree: [
    { name, type: "folder", children: Node[] },
    { name, type: "file", encoding: "base64", contentBase64: "..." }
  ],
  expandedPaths: string[]
}
```

- 実行時の `handle`（File）/ `script_path` は保存しない。復元時に Base64 → `File` 再生成し `ensure_script_paths` でパス付与

#### postMessage

| cmd | 方向 | 内容 |
|-----|------|------|
| `script_explorer_state_save` | IDE → Host | `{ type, tree, expandedPaths }`（シリアライズ済み） |
| `script_explorer_state_clear` | IDE → Host | LS キー削除 |
| `script_explorer_state_restore` | Host → IDE | 保存オブジェクト or `{ empty: true }`（初回 ready 後 1 回） |
| `script_explorer_state_save_error` | Host → IDE | Quota 等の保存失敗通知 |

#### × クリア

1. confirm（i18n: `clearScriptStateConfirm`）
2. IDE → Host `script_explorer_state_clear`
3. メモリ: `scriptExplorerState` 空、`githubExplorerActive=false`、script 展開 Set クリア、hint 表示

---

## 13. VFS_Explorer 仕様

### 13.1 スキャン

- `create_tree_from_vfs(MOUNT_PATH)` で再帰構築
- 表示ルートは `vfs_root` フォルダ自体をラッパー

### 13.2 状態フラグ（`is_mounted` / NativeFS）

| 変数 | 意味 | true / 非 null になる条件 |
|------|------|---------------------------|
| `mounted_dir_handle` | NativeFS で接続中の DirectoryHandle | `mountNativeFS` 成功時のみ |
| `is_mounted` | NativeFS マウント中か | **`mountNativeFS` 成功時のみ `true`**。解除時は必ず `false` と handle `null` |

**禁止:** 次の成功パスで `is_mounted = true` を立てない

- `handle_upload_folder_to_vfs`（memfs 全置換）
- `apply_host_vfs_drop_write` / `handle_vfs_drop`（memfs 追加書き込み）
- その他の memfs `writeFile` 系

### 13.3 VFS クリア共通処理 `clear_vfs_root(options?)`

```text
clear_vfs_root({ restoreCwd?: boolean })
  1. FS.chdir("/")（削除対象上に CWD があると失敗しやすい）
  2. mounted_dir_handle があるときだけ unmount
       - unmount(MOUNT_PATH/handle.name) を試行
       - 必要なら unmount(MOUNT_PATH) を試行
  3. handle = null, is_mounted = false
  4. remove_directory_recursive_vfs(MOUNT_PATH)（失敗は握りつぶし可）
  5. FS.mkdirTree(MOUNT_PATH)
  6. restoreCwd=true なら CWD を MOUNT_PATH に戻し、Python os.chdir も同期
```

| 操作 | clear するか | 備考 |
|------|--------------|------|
| VFS Reset ボタン | する（`restoreCwd: true`） | 確認ダイアログ必須 |
| NativeFS マウント前 | する | 既存マウント / memfs 残骸を消してから mount |
| VFS フォルダ Upload | する | **全置換**。成功後も `is_mounted = true` にしない |
| VFS DnD（ホスト / FileList） | **しない** | **追加書き込みのみ**。フラグも触らない |

「中身があるか」を表す別フラグ（`hasVfsContent` 等）は導入しない。表示はツリー再構築結果に従う。

### 13.4 Upload

- フォルダ選択 → **先に `clear_vfs_root()` でルート全置換** → 相対パスを `MOUNT_PATH` 配下に `FS.writeFile` / `mkdirTree`
- memfs のみ。`is_mounted` は立てない

### 13.5 内部移動

- MIME: `application/vfs-path`
- `FS.rename` 相当の移動
- 展開パスリマップ

### 13.6 右クリック

| 対象 | 動作 |
|------|------|
| ファイル | 確認後 `js_download_vfs_file` |
| フォルダ | パスワード任意の ZIP DL（空=非暗号、OK=ZipCrypto / キャンセル=AES-256） |

### 13.7 ZIP セキュリティ

危険拡張子を含む場合 ZIP 作成を中止する:

`.exe .scr .bat .cmd .sh .ps1 .vbs .js .jar .com .msi .wsf .scf .lnk`

### 13.8 Reset

1. 確認ダイアログ（文言は `t("vfsResetConfirm")` 等で i18n）
2. `clear_vfs_root({ restoreCwd: true })`
3. 再描画

---

## 14. ドラッグ＆ドロップ（外部 OS → Explorer）

### 14.1 背景

sandbox null origin では DirectoryEntry が失敗するため、**ホストが drop を受けてファイルを読み、IDE に書き込み要求を送る**。

### 14.2 フロー

1. IDE が VFS/Script ゾーンの矩形をホストへ通知（`vfs_drop_zone_rect` / `script_drop_zone_rect`）
2. 外部ファイル dragenter を検知 → ホスト capture 開始（オーバーレイ表示、iframe pointer-events none）
3. ポインタ移動中: ホストが IDE へ hit test 座標送信 → IDE がターゲット path を返信（フォルダ行に破線ハイライト）
4. ゾーン外へ出た / DnD キャンセル: ホストが `explorer_dnd_clear` を送り IDE の破線ハイライトを解除
   - OS ファイル DnD はページに `dragend` が来ないため、**dragover 心拍の停止**（約 200ms idle）で capture 終了
   - 併せて document `dragleave`（ウィンドウ外）/ Esc / 内部 DnD の `dragend`、IDE 側 idle 監視でも解除
5. drop: ホストが Entry API / FileSystemHandle / FileList で再帰収集
6. ホスト → IDE: `vfs_drop_write` または `script_drop_write`（パスとテキスト/バイナリ内容）
7. IDE が VFS **追加**書き込み or Script ツリーマージ → 再描画  
   - VFS 側は `clear_vfs_root` しない / `is_mounted` を触らない（Upload の全置換とは異なる）

### 14.3 内部移動との分離

ホストは `application/vfs-path` / `application/script-path` を含む DnD を外部扱いにしない。

### 14.4 DataTransfer 注意

- drop 同期区間でスナップショットを取る（非同期後は中身が空になるブラウザがある）
- 二重処理防止フラグを持つ

---

## 15. ホスト ⇔ IDE 通信プロトコル

メッセージは原則 JSON 互換オブジェクト。共通形:

```js
// IDE → Host
{ cmd: string, req_id?: string, obj?: object, args?: any[] }

// Host → IDE（非同期応答）
{ req_id: string, status: "success"|"error", result?: any, error?: string }
```

### 15.1 IDE → Host（主要 cmd）

| cmd | 目的 |
|-----|------|
| `script_complete` | 初期化完了 / 実行完了。Agent 時は `obj.log` に stdout |
| `vfs_tree_result` | Agent 向け VFS 構造。`obj`: path / tree / fileCount / dirCount / truncated / maxDepth / error? |
| `workspace_snapshot_result` | Agent 向け地図。`obj`: activeTab / openTabs / scriptExplorer / vfs / github / syntax（本文なし） |
| `script_tree_result` | Agent 向け Script_Explorer 構造。`obj`: type / tree / fileCount / dirCount / truncated / maxDepth / error? |
| `repo_search_result` | Agent 向け検索。`obj`: query / scope / hits / skipped / truncated / error? |
| `file_read_result` | Agent 向け本文。`obj`: path / language / content / lineCount / startLine / endLine / truncated / error? |
| `code_replace_result` | Agent 向け置換結果。`obj`: path / replacements / ran / error? |
| `fetch_post` | ホスト経由 POST（URL ホワイトリスト必須） |
| `serial_write` | Serial 操作委譲。`obj.sub_cmd`: connect/write/disconnect |
| `gas_call` | GAS 関数の非同期呼び出し。`obj.funcName` + `args[]`。Host が opener ランチャーへ中継 |
| `github_connect_request` | ログイン or モーダル |
| `github_file_request` | `{ path }` ファイル本文要求 |
| `github_push_request` | `{ files:[{path,content}], message, branch, owner, repo }` |
| `github_refresh_tree_request` | ツリー再取得 |
| `github_open_repo_request` | （将来用） |
| `vfs_drop_zone_rect` / `script_drop_zone_rect` | DnD 矩形 |
| `vfs_external_drag_enter` / `script_external_drag_enter` | capture 開始 |
| `vfs_dnd_target` / `script_dnd_target` | ヒット結果反映用（ホストが IDE へ hit_test 後に IDE が返す想定を含む） |

`js_post_message` は `fetch_post` / `serial_write` / `gas_call` で `pending_host_requests[req_id]` に Promise を登録する。  
Host から `{ req_id, status: "success"|"error", result?, error? }` が返ると Promise が settle し、Python の `await` で戻り値または例外として受け取れる。

**Python → JS 境界（呼び出し側の責務）:**  
`dict` / `list` は PyProxy のまま `postMessage` すると `DataCloneError` になる。  
アプリ側では自動変換しない。呼び出し側で `from pyodide.ffi import to_js` し、`obj` / `args` をプレーン JS 値にしてから渡す。

### 15.2 Host → IDE（主要 cmd）

| cmd | 目的 |
|-----|------|
| `click` | `{ click: "run_script_button" | "stop_script_button" | "save_script_button" | "toggle_sidebar_button" | ... }` |
| `var_val` | `js_obj_global_data` 更新 |
| `serial_rx` | 受信バイト列を Python コールバックへ |
| `agent_run` | `{ code }` エディタにセットして RUN（`agent_run_flag=true`）。Nano / Web ブリッジ用 |
| `agent_run_current` | アクティブタブを上書きせず RUN（API の `script_run` / `script_run_request`） |
| `agent_vfs_tree` | `{ path?, maxDepth? }` VFS 構造走査（Python 非実行）。結果は IDE → Host の `vfs_tree_result` |
| `agent_workspace_snapshot` | 地図取得（本文なし）→ `workspace_snapshot_result` |
| `agent_script_tree` | `{ maxDepth? }` Script_Explorer 走査 → `script_tree_result` |
| `agent_repo_search` | `{ query, scope?, glob?, maxHits?, context? }` → `repo_search_result` |
| `agent_file_read` | `{ path?, startLine?, endLine?, maxChars? }` → `file_read_result`（github: は Host 直読） |
| `agent_code_replace` | `{ path?, oldString, newString, replaceAll? }` → `code_replace_result` |
| `github_auth_state` | `{ loggedIn, loginName }` **token なし** |
| `github_repo_tree` | `{ owner, repo, branch, tree }` |
| `github_file_content` | `{ path, content, sha? }` |
| `github_push_result` | `{ ok, message, commitSha? }` |
| `github_error` | `{ message }` |
| `github_collect_for_push` | IDE が open GitHub タブを収集して `github_push_request` |
| `vfs_dnd_hit_test` / `script_dnd_hit_test` | 座標ヒットテスト |
| `explorer_dnd_clear` | 外部 DnD キャンセル / ゾーン外退出時に VFS・Script の破線ハイライトを解除 |
| `vfs_drop_write` / `script_drop_write` | 外部 drop 書き込みデータ |
| `vfs_drop_error` / `script_drop_error` | エラー表示 |

### 15.3 ホスト受信の送信元検証

- 基本: `event.source === ide_container.contentWindow`
- 例外許可: Gemini Web（`https://gemini.google.com`）の opener、chrome-extension オリジン
- 未知 source は警告して破棄
- GitHub 系 cmd は IDE source のみ受理

### 15.4 GAS 連携（`gas_call` / opener ブリッジ）

`google.script.run` は HtmlService ランチャーにのみ存在する。トップレベル Host（`document.write` された IDE）は `window.opener` 経由でランチャーに依頼する。

```
Python: await js_post_message("gas_call", "host", "*", {"funcName": "ping"}, [])
  → IDE: pending_host_requests[req_id]
  → Host: バリデーション後 opener.postMessage
  → Launcher: google.script.run.gasDispatch(funcName, args)
  → Host: gas_call_result → IDE { req_id, status, result|error }
  → Python: await の戻り値 / 例外
```

#### IDE → Host

```js
{
  cmd: "gas_call",
  req_id: string,           // IDE 採番
  obj: { funcName: string }, // GAS 関数名（識別子）
  args: any[]               // 引数リスト（JSON シリアライズ可能）
}
```

#### Host → ランチャー（opener）

```js
{
  channel: "pyodide-ide-gas",
  v: 1,
  type: "gas_call",
  req_id: string,
  funcName: string,
  args: any[]
}
```

#### ランチャー → Host

```js
// 成功（result が Python の await 戻り値になる）
{ channel: "pyodide-ide-gas", v: 1, type: "gas_call_result",
  req_id, ok: true, result: any }

// 失敗
{ channel: "pyodide-ide-gas", v: 1, type: "gas_call_result",
  req_id, ok: false, error: string }
```

#### 制約・セキュリティ

| 項目 | 内容 |
|------|------|
| 送信元 | Host は IDE frame のみ受理。ランチャー結果は `event.source === window.opener` のみ |
| channel | `pyodide-ide-gas` かつ `v === 1` |
| funcName | `/^[A-Za-z_][A-Za-z0-9_]*$/` |
| args | 配列必須。JSON 概算 1MB 超は拒否 |
| タイムアウト | 60s（Host）。超過時 IDE へ error |
| opener 無し | 即 error: `GAS launcher not available...`（通常 https ホストでは常にこちら） |
| サーバ | `gasDispatch` の registry ホワイトリストのみ実行（雛形 `gas_integration/Code.gs`） |

#### Python 例

```python
from pyodide.ffi import to_js
import js

result = await js_post_message(
    "gas_call", "host", "*",
    # dict 既定は JS Map になるため、プレーン Object にする
    to_js({"funcName": "addNumbers"}, dict_converter=js.Object.fromEntries),
    to_js([1, 2, 3, 4, 5]),
)
print(result["sum"])  # 15
```

雛形・手順: `02_deliverables/gas_integration/README.md`  
実装: Host `10_gas_bridge.js` / IDE `js_post_message`（pending） / GAS `LauncherClient.html` + `Code.gs`

### 15.5 fetch_post セキュリティ

- `ALLOWED_POST_URLS` ホワイトリストに無い URL は拒否
- payload サイズ上限 10MB
- 成功/失敗を `req_id` 付きで IDE へ返却

---

## 16. GitHub 連携詳細

### 16.1 責務分割

| 層 | 担当 |
|----|------|
| Cloudflare Worker | OAuth login/callback/logout(revoke) のみ。token 永続化しない |
| Host | PKCE、token メモリ保持、GitHub REST、モーダル UI |
| IDE | ツリー表示、編集、push 用本文収集。token 非保持 |

### 16.2 OAuth フロー

1. ホストが `state` と PKCE `code_verifier` / `code_challenge` 生成
2. `window.open(Worker/auth/login?...)`
3. Worker が GitHub authorize へ 302（`prompt=select_account` 推奨）
4. callback で code 交換 → opener へ postMessage（token）
5. ホストは `event.source` / origin / state 検証後のみ token 保存

### 16.3 Worker エンドポイント

| Path | 役割 |
|------|------|
| `GET /auth/login` | 認可リダイレクト |
| `GET /auth/callback` | code 交換 + opener postMessage HTML |
| `POST /auth/logout` | grant revoke（CORS は ALLOWED_OPENER_ORIGINS） |
| `GET /health` | 死活 |

Secrets: `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `ALLOWED_OPENER_ORIGINS`

### 16.4 ホスト API 利用

| 操作 | API |
|------|-----|
| ユーザー | `GET /user` |
| リポジトリ一覧 | `GET /user/repos?per_page=100&sort=updated` |
| ツリー | ref → commit → `git/trees/{sha}?recursive=1` |
| ファイル | `GET /repos/{owner}/{repo}/contents/{path}?ref=`（base64 デコード） |
| Push | blobs → tree(base_tree) → commit → ref PATCH（`force:false`） |

- ツリー変換: flat blob をネスト。**ファイル数上限 5000**
- Push 対象: IDE で `githubPath` を持つ editor タブのみ
- コンフリクト時は強制 push しない

### 16.5 モーダル UI

- 未ログイン表示 / リポジトリ select / ブランチ / コミットメッセージ
- Open in Explorer / Push / Logout / Close
- ラベル・ステータス文言は `uiLang` に追従（`data-i18n` / `t()`）
- Logout:
  1. メモリ token 破棄 + IDE に loggedOut 通知
  2. Worker revoke
  3. 共有 PC では github.com からのログアウトを案内（ステータス文言）

### 16.6 ボタン配置（実装準拠）

- 起動トリガは **IDE の Script_Explorer 内 GitHub ボタン**
- クリックはホストへ中継（ホストが OAuth/モーダル専任）

---

## 17. AI Agent 仕様

### 17.1 コマンドフォーマット（IDE 契約）

#### Gemini → システム: スクリプト実行依頼

```json
{
  "cmd": "script_run_request",
  "script": "Pythonスクリプトを記述"
}
```

#### システム → Gemini: 実行結果

```json
{
  "cmd": "script_result",
  "result": "実行結果を記述"
}
```

#### Gemini → システム: 代理記憶依頼

```json
{
  "cmd": "add_memory",
  "content": [
    { "key_1": "value_1" },
    { "key_2": "value_2" }
  ]
}
```

- 代理記憶オブジェクトへ key/value を追加。重複キーは上書き

#### システムの代理記憶オブジェクト提示形

```json
{
  "cmd": "memory",
  "content": [
    { "key_1": "value_1" },
    { "key_2": "value_2" }
  ]
}
```

実装では `window.agent_memory` をオブジェクトとして保持し、提示時に JSON 文字列化して返してもよい。

#### Gemini → システム: 代理記憶提示依頼

```json
{ "cmd": "memory_presentation" }
```

#### Gemini → ユーザー: 検証依頼

```json
{
  "cmd": "user_verification",
  "message": "ユーザー検証内容"
}
```

- ホストは `alert` 表示後、完了通知を返す

#### Gemini → ユーザー: 目的達成

```json
{
  "cmd": "goal_achieved",
  "message": "目的達成内容"
}
```

- ループを終了し、Gemini への自動返信は行わない

#### Gemini → システム: VFS ディレクトリ構造取得依頼

```json
{
  "cmd": "vfs_tree_request",
  "path": "/mnt/vfs_root",
  "maxDepth": 8
}
```

| フィールド | 必須 | 説明 |
|------------|------|------|
| `cmd` | 必須 | `"vfs_tree_request"` |
| `path` | 任意 | 走査開始パス。省略時は `/mnt/vfs_root`（`MOUNT_PATH`）。相対パスはルート配下として解釈 |
| `maxDepth` | 任意 | 最大深さ（開始パス直下を depth 0）。省略時 `8`。実装は `0`〜`20` にクランプ |

- **目的**: LLM が `os.listdir` 等の Python を `script_run_request` で書かずに VFS 構造を取得し、コンテキスト汚染を軽減する
- **安全規則**: 解決後パスは必ず `/mnt/vfs_root` 配下。`..` による外への脱出は拒否
- **非対象**: ファイル内容の読取・書き込み・削除（構造＝名前と階層のみ）

#### システム → Gemini: VFS ディレクトリ構造結果

```json
{
  "cmd": "vfs_tree_result",
  "path": "/mnt/vfs_root",
  "tree": "vfs_root/\n├── data/\n│   └── sample.csv\n└── notes.txt",
  "fileCount": 2,
  "dirCount": 1,
  "truncated": false,
  "maxDepth": 8
}
```

- `tree`: tree コマンド風プレーンテキスト（フォルダは末尾 `/`）
- 空ディレクトリ時は `path/\n(empty)` 形式
- 深さ打ち切り時は `truncated: true` とし、テキスト末尾に打ち切り注記を付けてよい
- 失敗時は `error` 文字列を付与（`tree` は欠落しうる）

#### 処理フロー（`vfs_tree_request`）

1. Host `executeAgentCommand` が `activeVfsTreeSource` に返送先を記録
2. IDE へ `agent_vfs_tree` + `path` / `maxDepth`
3. IDE がパス正規化・サンドボックス → 深さ制限付き走査 → tree テキスト化
4. IDE → Host: `vfs_tree_result`（`obj` に path / tree / 件数 / truncated / error）
5. Host が `sendResponseToAgent` で Nano / Web ブリッジへ返送

#### システム → Gemini: 複数コマンド検出

```json
{
  "cmd": "Multiple_commands",
  "message": "（UI 言語に応じた定型文。t(\"agentMultiCommand\")）"
}
```

日本語例: `システム通知: 複数のコマンド（JSON）が検出されました。一度に実行できるコマンドは一つだけです。順次実行するか、一つに絞ってください。`

#### システム → Gemini: コマンド未検出

```json
{
  "cmd": "No_Command",
  "message": "（UI 言語に応じた定型文。t(\"agentNoCommand\")）"
}
```

日本語例: `システム通知: 出力の中に json コマンドが発見できませんでした。正しい json フォーマットに修正してください。`

#### 追加（実装）: JSON 構文エラー

```json
{
  "cmd": "JSON_Parse_Error",
  "message": "（UI 言語に応じた定型文。t(\"agentBadJson\", { message })）"
}
```

- 上記システム通知の実文言はホスト `uiMessages` に定義し、`uiLang` に追従する
- システムプロンプト内のフォーマット例も `${t(...)}` で同一文言を埋め込む

### 17.2 応答パース規則

1. `"cmd"\s*:\s*"` が 2 回以上 → `Multiple_commands`
2. ` ```json ... ``` ` ブロック優先抽出、そうでなければ最初の `{` から最後の `}`
3. `JSON.parse` 失敗 → `JSON_Parse_Error`
4. 抽出不能 → `No_Command`

### 17.3 有効コマンドと安全停止

有効 cmd（**Nano / Web ブリッジ**）:

`script_run_request`, `add_memory`, `memory_presentation`, `user_verification`, `goal_achieved`, `vfs_tree_request`

有効 cmd（**LLM API のみ追加**）:

`workspace_snapshot`, `script_tree_request`, `repo_search`, `file_read`, `code_replace`, `script_run`

- 無効（No_Command / Multiple / ParseError 等、および経路外の cmd）が **連続 3 回** で自律ループ強制停止
- 有効 cmd でカウンタリセット
- Nano が API 専用 cmd を出しても有効扱いにしない（未知としてカウント）

### 17.4 script_run_request / script_run 実行フロー

**Nano / Web ブリッジ（従来）**

1. `activeAgentSource` に `"NANO"` / `"WEB_BRIDGE"` を記録
2. IDE へ `agent_run` + code（エディタ全文を上書き）
3. IDE がコードをセットし `agent_run_flag=true` で RUN
4. 完了時 `script_complete` + `obj.log`
5. ホストが `script_result` を source に応じて返送

**LLM API（フェーズ A）**

1. `script_run` または `script_run_request`（`script` の有無を問わず）
2. IDE へ `agent_run_current`（**タブを上書きしない**）
3. アクティブなエディタタブを `agent_run_flag=true` で RUN
4. 以降は従来どおり `script_result`

#### Gemini → システム: ワークスペース地図（API）

```json
{ "cmd": "workspace_snapshot" }
```

戻り `workspace_snapshot_result`: `activeTab` / `openTabs` / `scriptExplorer`（type と件数） / `vfs`（件数） / `github`（接続状態） / `syntax`（アクティブタブの Error markers 要約）。**本文は含まない**。

#### Gemini → システム: Script ツリー（API）

```json
{ "cmd": "script_tree_request", "maxDepth": 8 }
```

- `maxDepth` 省略時 `8`、`0`〜`20` にクランプ
- 戻りは `vfs_tree_result` と同型の tree 風テキスト + `type`（local_upload / local / github）

#### Gemini → システム: リポジトリ検索（API）

```json
{
  "cmd": "repo_search",
  "query": "def foo",
  "scope": "all",
  "glob": "*.py",
  "maxHits": 30,
  "context": 2
}
```

| フィールド | 必須 | 説明 |
|------------|------|------|
| `query` | 必須 | 正規表現。不正ならリテラルとして検索 |
| `scope` | 任意 | `script` / `vfs` / `tabs` / `all`（既定 `all`） |
| `glob` | 任意 | ファイル名フィルタ（`*` / `**` / `?`） |
| `maxHits` | 任意 | 既定 30、上限 50 |
| `context` | 任意 | ヒット前後行。既定 2、上限 5 |

- GitHub 未取得ファイルの本文は検索しない
- バイナリ・約 256 KiB 超は `skipped`
- 走査ファイル数の上限 200。超過時 `truncated: true`
- Python 非実行

#### Gemini → システム: ファイル読取（API）

```json
{
  "cmd": "file_read",
  "path": "active",
  "startLine": 1,
  "endLine": 80,
  "maxChars": 20000
}
```

| `path` 例 | 対象 |
|-----------|------|
| `active` / `tab:main.py` | 開いているタブ |
| `script:foo/bar.py` | Script_Explorer |
| `vfs:/mnt/vfs_root/a.csv` または `/mnt/vfs_root/...` | VFS（ルート外拒否） |
| `github:src/a.py` | Host が既存 GitHub API で読取（token は Host のまま） |

- `maxChars` 既定 20_000、上限 40_000。超過時 `truncated: true`

#### Gemini → システム: 部分差し替え（API）

```json
{
  "cmd": "code_replace",
  "path": "active",
  "oldString": "一致させる既存断片",
  "newString": "置換後",
  "replaceAll": false
}
```

- `oldString` は空白・インデント含め完全一致
- 0 件、または 2 件以上かつ `replaceAll=false` → 実行せず `error`
- 対象: エディタタブ、`script:` の `local_upload`、`vfs:`。**github: は拒否**
- **実行しない**（`ran: false`）

#### 処理フロー（観察・編集コマンド）

1. Host `executeAgentCommand` が `activeAgentOpSource` に返送先を記録
2. IDE へ対応する `agent_*`（`github:` 読取だけ Host が直接 API）
3. IDE が走査 / 読取 / 置換（Python 非実行）
4. IDE → Host: `*_result`
5. Host が `sendResponseToAgent` で API 履歴へ返送

### 17.5 AI サイドバー UI（Nano / LLM API 共通）

| 要素 | 機能 |
|------|------|
| Provider select（`#llm-provider-select`） | `nano` / `openai` / `anthropic` / `google` / `xai` / `custom` |
| Model input | API 経路のモデル ID。Nano 時は非表示 |
| API Key input（`type=password`） | BYOK。**メモリのみ**（sessionStorage / localStorage に書かない）。タブを閉じると消失。Nano 時は非表示 |
| Base URL input | `custom` 時のみ表示。OpenAI 互換のベース（例: `http://127.0.0.1:11434/v1`） |
| Agentic トグル | ON: コマンド解析と自律ループ。OFF: 通常チャット |
| Summarizer トグル | **Nano 専用**。API 経路では無効（履歴配列が文脈源） |
| 言語 select（`#nano-lang-select`） | **モデル応答言語**ヒント。初期 value は `uiLang`（`ja`/`en`）に連動。Nano セッション作成後は変更不可 |
| 送信 / 中断 | 生成中は AbortController で中断。ボタンラベルは `t("nanoSend")` / `t("nanoAbort")`。Enter 送信 / Shift+Enter 改行 |
| 入力欄高さ | `#nano-input-resizer` を上下ドラッグ。自動伸縮なし。storage 非永続 |
| 過去プロンプト編集 | ユーザー吹き出しのみ。ホバーで Edit。吹き出し内 textarea で編集し送信 / キャンセル（Esc）。生成中は不可。本文は `textContent` / `textarea.value` のみ |
| 編集送信（API） | 当該 user 通以降を `messages[]` とチャット DOM から削除し、置換して再リクエスト。先頭指示なら system の目的も差し替え。IDE 側の実行結果はロールバックしない |
| 編集送信（Nano） | Prompt API の内部履歴は書き換えない。編集結果を新しいユーザー発話として送る。後続 DOM は残す |
| リセット | Nano: セッション破棄。API: `messages[]` クリア。いずれもチャット DOM・`agent_memory` クリア。API キー欄は消さない（ユーザーが手動で消す） |

#### Gemini Nano

- Chrome Prompt API（`LanguageModel` / `ai.languageModel` 等の利用可能 API）に対応
- Summarizer API が使えれば rolling summary を生成
- システムプロンプトにコマンド仕様と目的を埋め込む（定型システム通知は `t()` 経由で UI 言語に合わせる）

#### 外部 LLM API（BYOK）`activeAgentSource = "API"`

| 項目 | 仕様 |
|------|------|
| 実装 | Host `07a_llm_api_agent.js`（`LlmApiAgent`） |
| システムプロンプト | `api_agent_system_prompt`（`02_gemini_nano_base.js`）。Nano 用 `custom_system_prompt` とは別。地図→検索→読取→置換を指示 |
| ステート | API 自体はステートレス。Host が `messages[]`（system 先頭 + user/assistant）をメモリ保持し、毎回リクエストに付与 |
| 履歴トリム | system 以外を直近約 40 通。各メッセージ文字数上限あり |
| OpenAI / xAI / Custom | `POST {base}/chat/completions` + SSE。`Authorization: Bearer`（Custom はキー空可） |
| Anthropic | `POST {base}/v1/messages` + SSE。`x-api-key` + `anthropic-version` |
| Google AI Studio | `streamGenerateContent?alt=sse` + API key |
| エージェントフィードバック | `sendResponseToAgent` → `LLM_API_AGENT.sendSystemFeedback` → 履歴に user として積み再リクエスト |
| CORS | ブラウザ直 `fetch`。プロバイダが CORS 拒否する場合は失敗しうる（エラー表示）。回避は Custom + ローカル互換プロキシ |
| CSP | Host `connect-src` のみ許可。IDE CSP には LLM エンドポイントを含めない |
| セキュリティ | キーは Host のみ。IDE / ログ / 永続ストレージへ渡さない |

既定 Base URL / モデル例:

| Provider | Base URL | モデル例 |
|----------|----------|----------|
| openai | `https://api.openai.com/v1` | `gpt-4o-mini` |
| anthropic | `https://api.anthropic.com` | （実装の既定値） |
| google | `https://generativelanguage.googleapis.com/v1beta` | `gemini-2.0-flash` |
| xai | `https://api.x.ai/v1` | `grok-3-mini` |
| custom | ユーザー入力（既定プレースホルダ `http://127.0.0.1:11434/v1`） | 任意 |

### 17.6 Web Gemini ブックマークレット

- Gemini ページで回答テキストを監視し、JSON コマンドを IDE に postMessage
- IDE は opener/parent へ結果を返す
- 参考実装: `01_referenceMaterials/reference_bookmarklet.html`

---

## 18. Web Serial 仕様

### 18.1 ホスト `Serial_port_manager`

- `serial_connect(options)`: `navigator.serial.requestPort` → open
- 読み取りループで受信データを IDE へ `serial_rx`
- `serial_write(Uint8Array-like)` / `serial_disconnect`
- 未接続時のダミー Modbus 風応答（開発・デモ用）を含んでよい

### 18.2 パラメータ UI

デフォルト例: baud `9600`、dataBits `8`、parity `none`、stopBits `1`（実装の入力初期値に合わせる）

### 18.3 Python 連携

- Python から `js_post_message("serial_write", ...)`
- 受信は `js_register_python_callback` で登録した関数へディスパッチ

---

## 19. プロット・プレビュー・拡張エリア

### 19.1 Plot

- `js_add_plot_image(base64_png)` が plot タブ／コンテナに画像追加
- matplotlib 連携で自動表示

### 19.2 Preview

- HTML コードを Blob URL で preview iframe に表示（dev のみ）
- preview → IDE の message は Python コールバックへ

### 19.3 Footer

| 領域 | 内容 |
|------|------|
| `#py_result_area` | Python stdout/stderr |
| `#js_result_area` | JS ステータス・ログ |
| `#extension_area` | 拡張表示（必要に応じて） |

---

## 20. セキュリティ要件まとめ

1. **信頼境界**: token / Serial / 外部 fetch はホスト専任
2. **sandbox に `allow-same-origin` を付けない**
3. **GitHub token を postMessage・localStorage・Cookie に載せない**
4. **IDE CSP から api.github.com を除外**
5. **fetch_post は URL ホワイトリスト + サイズ制限**
6. **危険拡張子の ZIP 同梱禁止**
7. **path traversal（`..`）拒否**
8. **dev_flag 無しでは JS 実行・HTML プレビュー禁止**
9. **Agent は 1 応答 1 コマンド、連続失敗で停止**
10. **ホスト message は source 検証**
11. **i18n 文字列は固定リテラル**。ユーザー入力を `innerHTML` へ生挿入しない（`textContent` / 既存の安全な挿入を維持）

---

## 21. コーディング規約（成果物）

- コメントは日本語
- 変数名はキャメルケースを基本（既存スコープの snake_case と混在可だが新規はキャメル優先）
- インデント 4 スペース
- 共通処理は関数化
- 大きな配列の不要コピーを避ける
- 機密はメモリ保持、永続化しない

---

## 22. 成果物ディレクトリ

```text
02_deliverables/
  index.html                 # 本体（Host + sandbox 埋め込み）
  favicon.png
  github_integration/
    cloudflare_worker/
      worker.js              # OAuth 中継
      POLICY.md / PROGRESS.md
01_referenceMaterials/
  reference_index.html
  reference_bookmarklet.html
```

---

## 23. 受け入れ条件（再実装チェックリスト）

### IDE 基本

- [ ] インストール不要でブラウザ起動できる
- [ ] sandbox iframe 内で Monaco + Pyodide が動く
- [ ] `main.py` 初期コードを RUN して stdout が表示される
- [ ] STOP で中断できる（可能な環境で KeyboardInterrupt）
- [ ] 複数タブ、リネーム、CSV スプレッドシート、保存（Ctrl+S）
- [ ] Outline と Python 構文エラーマーカー
- [ ] Script_Explorer へフォルダ Upload → ファイルを開ける
- [ ] `?state=ful` で Script Upload 後リロード → ツリー復元・ファイルが開く
- [ ] `?state=ful` で × ボタン → confirm 後 LS と表示がクリアされ、再リロードでも空
- [ ] `state` 無し / `state=1` / `state=full` / `State=ful` ではステートフルにならない（⟳ のまま）
- [ ] VFS Upload / Refresh / Reset / 内部移動 / 右クリック DL
- [ ] 外部 DnD でフォルダ階層ごと取り込める（ホスト代理）
- [ ] 展開状態が Refresh / DnD 後も維持される
- [ ] リサイザでレイアウト変更可能

### セキュリティ

- [ ] IDE から GitHub token が見えない
- [ ] 本番で JS/HTML 直接実行が拒否される
- [ ] fetch_post が未許可 URL を拒否する

### GitHub

- [ ] トップレベル https/localhost のみボタン表示
- [ ] file:// と iframe 埋め込みでは無効
- [ ] 初期化完了前は操作不能
- [ ] OAuth ログイン → ツリー表示 → 編集 → Push
- [ ] Logout でメモリ破棄（永続ストレージに残らない）

### AI

- [ ] Nano / Web ブリッジから script_run_request で実行できる
- [ ] script_result が返る
- [ ] 複数 cmd / 未検出 / 3 連続失敗の安全動作
- [ ] `vfs_tree_request` で tree 風 `vfs_tree_result` が返る（Python 非実行）
- [ ] `path` が `/mnt/vfs_root` 外のときエラーになる
- [ ] `maxDepth` 打ち切り時に `truncated: true`
- [ ] API 経路のみ `workspace_snapshot` / `script_tree_request` / `repo_search` / `file_read` / `code_replace` / `script_run` が有効
- [ ] Nano のコマンド集合・全文上書き RUN が現行と同じ
- [ ] API の `script_run_request` / `script_run` がタブを上書きせず実行する
- [ ] `code_replace` が 0 件・複数件（replaceAll なし）を実行しない
- [ ] `file_read` の VFS が `/mnt/vfs_root` 外を拒否する
- [ ] GitHub token / API キーが IDE・storage に出ない
- [ ] 入力欄をドラッグで高さ変更でき、最小・最大でクランプされる
- [ ] ユーザー吹き出しだけ編集できる（AI / system / hint は不可）
- [ ] API: 編集送信で当該通以降が消え、新しい指示で続きが生成される
- [ ] Nano: 編集送信は新ターンとして扱われる
- [ ] 生成中は吹き出し編集ができない
- [ ] 吹き出し本文は `textContent` / `textarea.value` のみ（XSS しない）

### UI 言語

- [ ] `?lang=ja` / 未指定で日本語 UI
- [ ] `?lang=en` で英語 UI
- [ ] ホストと IDE の表示言語が一致する
- [ ] `#nano-lang-select` 初期値が `uiLang` と一致する
- [ ] 言語切替 UI が無く、クエリ変更はリロード前提である

### VFS 状態

- [ ] VFS Upload はルート全置換で、`is_mounted` を true にしない
- [ ] VFS DnD は追加のみでルートをクリアしない
- [ ] Reset は `clear_vfs_root` 経由で unmount + 全削除する
- [ ] NativeFS マウント成功時のみ `is_mounted === true`

---

## 24. 非目標（本仕様の対象外）

- フル Git クライアント（branch 作成 UI、merge、PR 作成）
- Worker 経由の GitHub API 全面プロキシ
- GAS 埋め込み上での GitHub 連携有効化
- file:// 上での OAuth
- トークンの安全な長期永続化
- 有料サービスの契約や未指示の外部アカウント作成
- UI 言語のホットスワップ（セレクトボックス等）や localStorage への言語記憶
- 第三言語（`es` 等）の UI 対応
- HTML 分割ビルドツールの導入

---

## 25. 用語集

| 用語 | 意味 |
|------|------|
| Host Frame | 外側 document。特権・認証 |
| IDE Frame | sandbox iframe。編集と Python 実行 |
| VFS | Pyodide 仮想ファイルシステム（`/mnt/vfs_root`） |
| Script_Explorer | 編集対象ソースツリー UI |
| VFS_Explorer | 実行時ファイルツリー UI |
| Agent | JSON コマンドで IDE を操作する LLM ループ |
| PKCE | OAuth 公開クライアント向け拡張 |
| `uiLang` | UI 表示言語（`ja` / `en`）。クエリ `lang` から決定 |
| `statefulMode` | クエリ `state=ful` 時 true。Script_Explorer（local_upload）を Host LS に保持。Ctrl+S で Explorer へ書き戻し |
| `scriptPath` | タブが Script_Explorer（local_upload）由来のときの論理パス（Ctrl+S 書き戻し先） |
| `t(key)` | UI 文言取得関数（ホスト / IDE 各 document に定義） |
| `is_mounted` | **NativeFS マウント中のみ** true のフラグ（memfs 有無とは無関係） |
| `clear_vfs_root` | NativeFS unmount + VFS ルート全削除 + 再作成の共通関数 |

---

## 改訂履歴

| 版 | 内容 |
|----|------|
| 初期 | 概念・AI コマンド骨子 |
| 詳細化 | アーキテクチャ・IDE 詳細・プロトコル・データ構造・受け入れ条件を追加し、ゼロから再実装可能な水準に拡張 |
| v1.12.030 反映 | モダン UI デザイントークン・統一トーン |
| v1.12.031 反映 | ホスト iframe flex 追従、フッター 3 ペイン幅計算 |
| v1.12.032 反映 | `is_mounted` を NativeFS 専用化、`clear_vfs_root`、Upload 全置換 / DnD 追加のみ |
| v1.12.033 反映 | `lang` クエリによる UI 日英切替、`uiLang` 注入、辞書 + `t()` / `data-i18n*`、`#nano-lang-select` 初期値連動。本書を現行実装に同期 |
| v1.12.034 反映 | Agent コマンド `vfs_tree_request` / `vfs_tree_result`。VFS 構造を tree 風テキストで返す（`/mnt/vfs_root` サンドボックス、`maxDepth`、Python 非実行） |
| v1.12.035 反映 | ステートフル: × SVG を VFS リセットと同一化。Ctrl+S で編集中スクリプトを Script_Explorer（local_upload）へ保存 + LS 永続化。タブ `scriptPath` |
| v1.12.036 反映 | GAS `gas_call`: Python → IDE `js_post_message` → Host → opener ランチャー `google.script.run`。非同期戻り値を Promise / await で受け取り。雛形 `gas_integration/` |
| v1.12.037 反映 | （撤回）`js_post_message` 内 PyProxy 自動変換 — 呼び出し側 `to_js` 方針へ |
| v1.12.038 反映 | `js_post_message` は Structured Clone 可能な値のみ受理。Python は `to_js()` 必須（自動変換なし） |
| v1.12.039 反映 | AI Agent BYOK: Provider 選択（OpenAI/Anthropic/Google/xAI/Custom）、API キーはメモリのみ、Host `messages[]` でステートレス API の文脈保持、Host CSP に LLM エンドポイント |
| v1.12.040 反映 | API エージェント フェーズ A: `workspace_snapshot` / `script_tree_request` / `repo_search` / `file_read` / `code_replace` / `script_run`。Nano は現行コマンドのまま。GitHub は読取のみ |
| v1.12.041 反映 | AI サイドバー: プロンプト入力欄の高さドラッグ変更。ユーザー吹き出しの過去プロンプト編集（API は履歴巻き戻し、Nano は新ターン送信） |
| ソース分割フェーズ1 | 開発ソースを `02_deliverables/src/{host,ide}/` に物理分割。`build.py` で単一 `index.html` を生成（ランタイム・バージョン表記は不変。再ビルドは分割前とバイト一致） |
| ソース分割フェーズ2 | `script_ROOT` / `script_MAIN` を機能別ディレクトリ（各 9 ファイル）へ分割。ビルドは名前順連結。ランタイム・`v1.12.034` 不変・連結後バイト一致 |
| **ステートフルモード（現行追記）** | クエリ `state=ful` で Script_Explorer（local_upload）を Host `localStorage` に永続化。更新ボタンを × に変更しクリア。IDE は postMessage 経由（null origin 維持） |
