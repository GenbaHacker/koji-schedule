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

**GAS（スプレッドシートに紐づく Apps Script）**: 列は**見出し名で探す**（`col('案件名')` 等）。
列の位置を数字で決め打ちしないこと（2026-11-11 KS-2 で、No列の挿入により全列がずれた）。
見出しが見つからない場合は `ok:false` と「見出しが見つからない: ○○」を返す（黙って空データを返さない）。
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
- 2026-11-11 KS-2（`e582357`）: 管理表の A列No挿入による列ずれを修正。GAS見本を見出し名方式に、貼り付けの既定列を現行配置に、先頭タブ消失を修正。詳細は `HANDOFF.md`
- 2026-07-22 KS-1（`bc817f9`）: ガントバーの色を工務担当に紐付け

## Rendering

`render()` calls three sub-functions:
- `renderHeader()` — builds the date header (month row + day row) based on `viewStart` and `scale`
- `renderLeft()` — builds the task list panel; groups rows when sorted by categorical fields
- `renderChart()` — builds Gantt chart rows with absolutely-positioned `.gantt-bar` divs; bar position/width calculated from `diffDays()` and `cellW()`

After `renderChart()`, `attachBarEvents()` adds mouse drag/resize handlers. Scroll sync between the left panel and chart area is handled in `syncScroll()`.
