# HANDOFF — koji-schedule（工事工程表）

最終更新: 2026-11-11

## 現在地
- 本番: https://genbahacker.github.io/koji-schedule/ （GitHub Pages、`GenbaHacker/koji-schedule` main）
- 最新: `e582357` fix: KS-2（2026-11-11 push、Pages build `built e582357`）
- 担当: 決まった係なし（RULES_agents.md の係表に未登録）。KS-2 は清田の直接指示で工場（@SOLAR）セッションが実施

## KS-2（2026-11-11）管理表の列ずれ修正

### 症状
清田「GASずれた」。koji-schedule の GAS 連携で、状態・案件名・工期などが別の列の値になっていた。

### 原因
- 管理表「案件管理（2026年度）」の A列に案件No.（KJ0001 等）が挿入され、列も増えていた。
  現行の見出し（3行目）: A No / B 状態 / C 自社担当 / F 案件名 / I 受電 / K エリア / M 工務担当 / O 工事会社 / Q ランク / W 契約予定日 / X 着工予定 / Y 完工予定
- GAS 見本と貼り付け取り込みは、列を位置で決め打ちしていた（A=状態, D=案件名, T=着工, U=完工…）。
- データ読み始めが5行目（`i = 4`）で、4行目の先頭データ（KJ0001）を読み飛ばしていた。
- 貼り付け取り込みの `trim()` が先頭タブを消し、先頭列が空だと列が1つずれていた（改修前からの不具合）。
- ※スプレッドシートに紐づく GAS 本体の中身は未確認。画面の見本コードからの推測。

### 修正（index.html）
- `GAS_CODE`: 3行目の見出し名で列を探す方式。シート名 `案件管理（2026年度）` を指定。見出しが無ければ `ok:false`。
- `COL`: 現行配置に更新。`COL_LABELS` を追加し、見出し行つきの貼り付けは見出しから列を決める。
- `parseTSV` / `doImport`: `trim()` を使わず、改行の正規化と空行除外のみ。

### 検証（生出力）
ローカル（実シートの値。顧客データはリポジトリ外のスクラッチパッドで使用）:
```
① script parse OK: 1 block(s)
② GAS ok=true 件数=88 / 状態欄に案件No.が入った件数=0 / 受電が高圧低圧特高以外=0
③ 貼り付け(見出しなし) 件数=87 / (見出しつき) 件数=87、GASと一致: 見出しなし=true 見出しつき=true
④ 対照 1列ずらし: 見出しなし一致=false(false が正) / 見出しつき一致=true(true が正)
ブラウザ（localhost、localStorage.clear 後）: Console Total messages: 0
```
88件と87件の差は、着工予定 > 完工予定 の1件（貼り付け取り込みは従来どおり除外）。

本番（push 後）:
```
gh api …/pages/builds/latest → built e5823570ef7fedbec7c4019a3bf7871cd5ed4444 2026-11-11T07:24:57Z
本番HTML: COL_LABELS:True / HEADER_ROW:True / old row[19]:False
本番で parseTSV（模擬1件、見出しなし・あり）→ どちらも 案件名/状態/受電/工務/工事会社/ランク/着工/完工 が正しい列
Console: [ERROR] 404 favicon.ico のみ（ファビコン未設置。機能に影響なし）
push証明: git log origin/main..HEAD --oneline → (empty)
```

## 未完了（清田の作業）
- [ ] **管理表に紐づく GAS 本体の貼り替え**。これをやるまで GAS 連携の表示はずれたまま。
  1. 管理表 → 拡張機能 → Apps Script → `doGet` を本番画面「GAS連携 → GASの設定方法」の見本コードに置き換えて保存
  2. デプロイ → デプロイを管理 → 鉛筆 → バージョン「新しいバージョン」→ デプロイ（URL不変）
  3. koji-schedule を再読込して、状態・案件名・工期を目視確認（施主検査）

## 残課題（未着手）
- 手元 remote URL を `GenbaHacker/koji-schedule.git` に更新するか
- favicon 404（見た目のみ）
- 係の割り当て（RULES_agents.md の係表に koji-schedule が無い）
