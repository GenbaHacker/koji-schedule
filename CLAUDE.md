# CLAUDE.md

## 作業環境
- **パス**: `C:\Obsidian-Vault\10_PROJECTS\sawada\koji-schedule`
- **親ナレッジベース**: `../CLAUDE.md`（SAWADA中間層規約）
- **共有スキル**: `../../../_shared/skills/`

## Overview

A single-file Gantt chart web app (工事工程表 = construction schedule) built entirely in one `index.html` file — no build system, no dependencies, no server required. Open in a browser directly.

## Running

Just open `index.html` in a browser. No build step, no npm install, no server needed.

## Architecture

Everything lives in `index.html` as a single self-contained file with three sections:

1. **CSS** (`<style>`) — CSS custom properties (`--bg`, `--accent`, etc.) define the design tokens. Layout uses flexbox with a fixed left panel (`--left-w: 700px`) and a scrollable chart area.

2. **HTML** — Two-panel layout: `.task-panel` (left, task list) + `.chart-area` (right, Gantt chart). Two modals: `#gasModal` (Google Sheets integration) and `#importModal` (paste import).

3. **JavaScript** (`<script>`) — All logic in one script block. Key globals:
   - `tasks[]` — array of task objects `{id, name, start, end, area, sales, eng, co, power, color}`
   - `scale` — current time scale (`week`, `2weeks`, `month`, `quarter`, `half`, `year`)
   - `viewStart` — `Date` object for the left edge of the chart
   - `sortKey` — current sort field
   - `gasUrl` — GAS Web App URL, persisted in `localStorage` as `gantt_gas_url`

## Data Flow

**Google Sheets integration (GAS):** The app connects to a Google Apps Script web app deployed from a spreadsheet. The GAS code is embedded as a string constant `GAS_CODE` in the JS and displayed to users to copy. The frontend fetches JSON from the GAS URL.

**データ元**: スプレッドシート「自家消費案件管理表（2026）」のシート「案件管理（2026年度）」。
見出しは3行目、データは4行目から。2026-11 に A列へ案件No.（KJ/TJ/OT、`../tools/solarhack/claude/RULES_anken-id.md`）が挿入された。

**配信方式（KS-3, 2026-11-11〜）**: 社内で使う本番は **GAS ウェブアプリが画面ごと配信する**。
- eco-sawada.com の Workspace 設定で、ウェブアプリの公開範囲に「全員」を選べない（「自分のみ」「株式会社SAWADA 内の全員」だけ）。
  そのため github.io から GAS を `fetch` すると、ログイン画面に転送されて CORS で失敗する（Failed to fetch）。
- 解決: 管理表に紐づく Apps Script プロジェクトの **`工事工程表連携用.gs`** に置く（2026-11-12 清田決定）。
  `doGet` が GitHub Pages の index.html を `UrlFetchApp` で取得（10分キャッシュ）して `HtmlService` で返す。
  画面は `google.script.run.kojiGetData()` でデータを取る（`IN_GAS` で判定）。
  デプロイ設定: 次のユーザーとして実行=**自分** ／ アクセス=**株式会社SAWADA 内の全員**。
- 置き場所の決定: 当初は独立プロジェクトを推奨したが、清田が「他のGASがあると紛らわしい」として既存プロジェクトへの置き換えを選んだ。
  **承知のうえのリスク**: `google.script.run` は同じプロジェクトの全関数を呼べるため、社内の人が開発ツールを使えば
  `sortByTimestamp`（降順表示.gs）や 自動採番.gs・受注管理一覧.gs の関数を清田の権限で実行できる。
  同じプロジェクトに関数を足すときは、画面から呼ばれて困るものは名前の末尾を `_` にする（`_` 付きは google.script.run から呼べない）。
  `doGet` / `kojiGetData` と同名の関数を他のファイルに作らないこと。
- github.io 版（GAS外）は従来どおり「GAS連携」URL への fetch と貼り付け取り込み。社内限定の GAS には届かないので、貼り付け専用と考える。
- index.html を push すれば、GAS 側は最大10分で新しい画面になる（GAS の再デプロイは不要）。`GAS_CODE` を変えたときだけ GAS への貼り替え＋新しいバージョンのデプロイが要る。

**GAS のデータ取得（`kojiGetData`）**: 列は**見出し名で探す**（`col('案件名')` 等）。
列の位置を数字で決め打ちしないこと（2026-11-11 KS-2 で、No列の挿入により全列がずれた）。
見出しが見つからない場合は `ok:false` と「見出しが見つからない: ○○」を返す（黙って空データを返さない）。
スプレッドシートは `openById`（自家消費案件管理表（2026））で開く。
GAS 本体は GitHub に入っていない。`GAS_CODE` は利用者がコピーする見本で、本体と同じ内容に保つこと。
GAS を貼り替えたら「デプロイを管理 → 編集 → 新しいバージョン」でデプロイする（URLが変わらない。「新しいデプロイ」だとURLが変わる）。

**Paste import（貼り付け取り込み）**: 見出し行（3行目）を一緒に貼れば `COL_LABELS` の見出し名から列を決める。
見出しなしの場合は `COL` の既定位置（A列から貼った前提、0始まり）:
- B(1)=状態, C(2)=自社担当, F(5)=案件名, I(8)=受電, K(10)=エリア, M(12)=工務担当, O(14)=工事会社, Q(16)=ランク, W(22)=契約予定日, X(23)=着工予定, Y(24)=完工予定
- 取り込み前に `trim()` しない（先頭列が空のとき先頭タブが消えて列が1つずれる）

## デプロイ
- GitHub: `GenbaHacker/koji-schedule`（**public**。顧客データを入れない）
- GitHub Pages（main / ルート）→ https://genbahacker.github.io/koji-schedule/ 。push 後1分ほどで反映
- 手元の remote は旧URL `kiyo819-coder/koji-schedule.git` のまま（GitHub が転送するので push は届く）

## 変更履歴
- 2026-11-12 KS-3b: 配信用 GAS の置き場所を既存プロジェクトの `工事工程表連携用.gs` に変更（清田決定）。設定手順・見本コメント・github.io 版のエラー文（「アクセス=全員」の誤案内）を修正
- 2026-11-11 KS-3: 社内限定の GAS ウェブアプリから画面ごと配信する方式に対応（`IN_GAS` / `google.script.run.kojiGetData()`、GAS見本を配信用に差し替え）。詳細は `HANDOFF.md`
- 2026-11-11 KS-2（`e582357`）: 管理表の A列No挿入による列ずれを修正。GAS見本を見出し名方式に、貼り付けの既定列を現行配置に、先頭タブ消失を修正。詳細は `HANDOFF.md`
- 2026-07-22 KS-1（`bc817f9`）: ガントバーの色を工務担当に紐付け

## Rendering

`render()` calls three sub-functions:
- `renderHeader()` — builds the date header (month row + day row) based on `viewStart` and `scale`
- `renderLeft()` — builds the task list panel; groups rows when sorted by categorical fields
- `renderChart()` — builds Gantt chart rows with absolutely-positioned `.gantt-bar` divs; bar position/width calculated from `diffDays()` and `cellW()`

After `renderChart()`, `attachBarEvents()` adds mouse drag/resize handlers. Scroll sync between the left panel and chart area is handled in `syncScroll()`.
