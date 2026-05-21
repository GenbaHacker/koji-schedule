# CLAUDE.md

## 作業環境
- **パス**: `C:\Users\kiyo8\MyObsidian\06_Projects\koji-schedule`
- **親ナレッジベース**: `../../CLAUDE.md`（Obsidian ナレッジ）
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

**Spreadsheet column mapping** (`COL` constant, 0-indexed):
- D(3)=案件名, F(5)=受電, H(7)=エリア, I(8)=自社担当, K(10)=工務担当, M(12)=工事会社, S(18)=着工予定, T(19)=完工予定

**Paste import:** Reads TSV pasted from the spreadsheet using the same column indices.

## Rendering

`render()` calls three sub-functions:
- `renderHeader()` — builds the date header (month row + day row) based on `viewStart` and `scale`
- `renderLeft()` — builds the task list panel; groups rows when sorted by categorical fields
- `renderChart()` — builds Gantt chart rows with absolutely-positioned `.gantt-bar` divs; bar position/width calculated from `diffDays()` and `cellW()`

After `renderChart()`, `attachBarEvents()` adds mouse drag/resize handlers. Scroll sync between the left panel and chart area is handled in `syncScroll()`.
