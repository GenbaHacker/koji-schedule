# HANDOFF — koji-schedule（工事工程表）

最終更新: 2026-11-12

## 現在地
- 本番: https://genbahacker.github.io/koji-schedule/ （GitHub Pages、`GenbaHacker/koji-schedule` main）
- 社内で使う入口（KS-3 以降）: 管理表の Apps Script（`工事工程表連携用.gs`）のウェブアプリ `/exec` URL。github.io 版は貼り付け取り込み専用
- push 済み: `e582357` fix: KS-2 ／ `1638a84` docs: KS-2 ／ `227b0c0` KS-3b ／ `c72ef12` KS-4（2026-11-12 push、Pages built c72ef12、本番 sha256 = HEAD と一致）
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

## KS-3（2026-11-11）社内限定 GAS から画面ごと配信する方式へ

### 症状
清田が GAS 本体を KS-2 の見本に貼り替えた後、koji-schedule の「接続テスト」が「接続失敗：Failed to fetch」。

### 原因（実測）
- ログイン無しで GAS の /exec を呼ぶと `HTTP/1.1 302 Found` → `Location: https://www.google.com/a/eco-sawada.com/ServiceLogin?…`（curl）
- 本番 koji-schedule から fetch → `blocked by CORS policy: No 'Access-Control-Allow-Origin' header`（Playwright Console）
- デプロイ画面の「アクセスできるユーザー」の選択肢が「自分のみ」「株式会社SAWADA 内の全員」だけ（清田のスクショ）。
  Workspace 設定で「全員」を選べない。→ github.io から社内限定の GAS は呼べない。
- 対照（「全員」公開の GAS なら通ること）は未検査: 比較に使った solarhack の LOG_ENDPOINT が 404 だったため。

### 清田の決定
案C: koji-schedule を GAS から配信する（社内限定のまま、案件データを社外に出さない）。

### 修正（index.html）
- `IN_GAS`（`google.script.run` の有無）で判定。GAS 配信時は `google.script.run.kojiGetData()` でデータを取り、起動時に自動同期。
  github.io 版は従来どおり（URL fetch・貼り付け）。
- `GAS_CODE` を配信用に差し替え: `doGet` = GitHub Pages の index.html を UrlFetchApp で取得（CacheService 600秒）→ HtmlService で返す。
  `kojiGetData` = `openById` で管理表を開き、見出し名で列を探して `{ok, data}` を返す。
- 「GASの設定方法」の手順を、独立プロジェクト・実行=自分・アクセス=SAWADA内の全員 に書き換え。
- 独立プロジェクトにする理由: `google.script.run` は同じプロジェクトの全関数を呼べるため、管理表のプロジェクト（自動採番.gs 等）に置かない。

### 検証（生出力）
```
① script parse OK: 1 block(s)
② GAS(kojiGetData) ok=true 件数=88 / 状態欄に案件No.が入った件数=0 / 受電が高圧低圧特高以外=0
③ 貼り付け(見出しなし) 87件 / (見出しつき) 87件、GASと一致: 両方 true
④ 対照 1列ずらし: 見出しなし一致=false(false が正) / 見出しつき一致=true(true が正)
doGet 模擬: {"fetched":1,"url":"https://genbahacker.github.io/koji-schedule/","ttl":600,"title":"工事工程表","meta":"viewport=width=device-width, initial-scale=1"}（2回呼んで取得1回＝キャッシュ有効）
ブラウザ（localhost、google.script.run 模擬を注入）:
  ?gas=ok   → IN_GAS:true, calls:1, status:"接続済", modalOpen:false, 2件とも画面に表示（スクショ確認）、Console 0件
  ?gas=fail → alert「接続エラー：模擬: 権限がありません」, status:"エラー", tasks:0
  模擬なし  → IN_GAS:false, status:"未接続", modalOpen:false, 設定方法タブに新手順あり、Console 0件
```
本物の GAS 上での動作は未検査（清田のデプロイ後に施主検査）。

## KS-3b（2026-11-12）置き場所を既存プロジェクトに変更
- 清田決定: 独立プロジェクトは作らず、管理表の Apps Script の `工事工程表連携用.gs` を配信用コードに置き換える（「他のGASがあると紛らわしい」）。
- 承知のうえのリスク: 同じプロジェクトの `sortByTimestamp`・自動採番・受注管理一覧の関数も、社内の人が開発ツールから清田の権限で呼べる（CLAUDE.md に記載）。
- index.html: 設定手順を置き換え方式に、見本コードの先頭コメントを修正。github.io 版のエラー文の「アクセス=全員」（社内では選べない）を「/exec を直接開く」案内に修正。

### 施主検査（2026-11-12）
清田が `/exec` を開いて工程表がデータ付きで表示されることを確認（初回は旧バージョンのままで JSON が出た → 新しいバージョンで再デプロイして解消）。
社内限定 URL は公開リポジトリに書かない。

## KS-4（2026-11-12）案件No と Drive リンクの表示
- 一覧に「No」列（案件No）。案件名の横に 📁（管理表 AQ列「ファイルパス」に https の URL がある案件だけ。別タブで開く）
- `kojiGetData` が `url` を返す。「ファイルパス」列が無くてもエラーにしない（任意の列）
- 貼り付け取り込みも No を読む。見出しつきで貼れば「ファイルパス」も読む
- 検証（生出力）:
```
GAS ok=true 件数=88 / No入り=88 / url入り=1 → [["KJ0015","イケダ株式会社\n廿日市工場"]]
対照: ファイルパス列なし → ok=true 件数=88 url入り=0
貼り付け 見出しつき: 件数=87 No入り=87 url入り=1 / 見出しなし: 件数=87 No入り=87
fileLink: javascript: は出さない=true / 空=true / 引用符はエスケープされ属性から抜け出せない=true
ブラウザ ?gas=ok: KJ9001 に 📁（href=drive…MOCK, 表示あり）, TJ9002 は 📁 なし, No列 54px, Console 0件（スクショ確認）
既存回帰: ②88件 ③87/87一致 ④対照 false/true
```
- 施主検査（2026-11-12 夜）: 清田が `/exec` で新しい画面の表示を確認（「出た出た」）。push 直後は旧画面のまま → GAS キャッシュ10分＋Pages キャッシュ（max-age=600）で最大20分遅れると判明
- KS-4b（`b170637`）: 配信用 GAS の取得 URL に `?t=` を付け、Pages 側のキャッシュを避ける（反映遅れは GAS 側の最大10分のみに）
- [ ] 清田: `工事工程表連携用.gs` を新しい見本（KS-4b）に貼り替え → 保存 → デプロイを管理 → 鉛筆 → 新しいバージョン（No は貼り替え前から出る。📁 は貼り替え後）

## 未完了（清田の作業）
- [ ] **`工事工程表連携用.gs` を置き換えてデプロイ更新**
  1. 管理表 → 拡張機能 → Apps Script → `工事工程表連携用.gs` の中身を見本コードに置き換えて保存
  2. 関数 `kojiGetData` を選んで「実行」→ 権限を許可（外部URL取得が増える）。エラーが出なければOK
  3. デプロイ → デプロイを管理 → 鉛筆 → 新しいバージョン／実行=自分／アクセス=株式会社SAWADA 内の全員（URLは変わらない）
  4. `/exec` の URL を直接開いて、工程表がデータ付きで表示されるか目視（施主検査）

## 残課題（未着手）
- 手元 remote URL を `GenbaHacker/koji-schedule.git` に更新するか
- favicon 404（見た目のみ）
- 係の割り当て（RULES_agents.md の係表に koji-schedule が無い）
