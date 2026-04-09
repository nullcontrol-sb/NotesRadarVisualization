# NotesRadarVisualization — コード概要

beatmania IIDX のノーツレーダースコアを可視化するシングルページアプリ。
ファイル構成は `index.html`（本体）、`maintenance.html`（補完データ管理ツール）、`images/`（ヘルプ画像）のみ。

---

## データソース

| 役割 | 取得元 |
|---|---|
| レーダーCSV（NOTES/CHORD/PEAK/CHARGE/SCRATCH/SOF-LAN の6種） | Google Sheets `SHEET_ID` / gviz CSV、シート名でシリーズ取得 |
| 難易度補完スプレッドシート（IIDX Difficulty） | `LEVEL_SUPPLEMENT_URL`（別Sheets）、列: `TITLE, SP_B, SP_N, SP_H, SP_A, SP_L` |
| スコアCSV | ユーザーがアップロード（e-AMUSEMENT CSV） |

---

## 主要定数（index.html 内）

```
SHEET_ID             レーダーCSV の Sheets ID
LEVEL_SUPPLEMENT_URL 難易度補完スプレッドシートの URL
RADAR_TYPES          ['NOTES','CHORD','PEAK','CHARGE','SCRATCH','SOF-LAN']
DIFF_MAP             難易度定義 B/N/H/A/L (BEGINNER〜LEGGENDARIA)
SUPPLEMENT_COL_MAP   補完CSVの列名 ↔ diffAbbr の対応
```

### タイトル照合系マップ

```
TITLE_LOOKUP_EXTRA    スコアCSVタイトル → レーダーCSVで試す候補（strict/loose失敗時のフォールバック）
                      例: '0.59' → ["'.59", '.59']

TITLE_DISPLAY_OVERRIDE スコアCSVタイトル → 画面表示名の上書き
                      例: 'VOID' → 'VØID', '0.59' → '.59'

RADAR_TITLE_ALIAS     レーダーCSVタイトル → 補完CSVの正式タイトル
                      タイトル切れ・文字差異の吸収に使用。
                      未プレイリストの supplementTitleSet 照合・レベル取得・表示名に一括適用。
                      例: 'ACT0'→'ACTØ', 'POLKAMANIA'→'POLꓘAMAИIA', '.59'→'0.59'
```

---

## タイトル正規化関数

`titleKeyStrict(title)` — 記号を保持したままNFKD正規化+小文字+空白除去。記号主体タイトル(≡+≡等)に対応。

`titleKeyForDedup(title)` — 重複排除専用キー。`titleKeyLoose` が空文字（記号のみタイトル: 〆/∀/≡+≡）の場合は `titleKeyStrict` にフォールバック。repMap/repMapPlayed の構築とルックアップにはすべてこの関数を使う。

`titleKeyLoose(title)` — CSV間の表記差異を吸収する緩いキー。以下の変換を行う:
- NFKD正規化 → 合成文字除去 → 小文字化
- `ø/œ→o`, `æ→ae`, `ə→e`（シュワ）
- `ł→l`, `đ→d`, `ħ→h`, `ŧ→t`, `ŋ→n`
- ひらがな→カタカナ（U+3041-3096 に +0x60）
- `[^a-z0-9\p{Script=Hiragana}\p{Script=Katakana}\p{Script=Han}]` を除去

---

## 状態変数

```
radarMapByType      { type: { strict: Map, loose: Map } }  レーダーCSVデータ
supplementLevelMap  Map  "titleKeyLoose::diffAbbr" → level（補完CSV由来）
supplementTitleSet  Set  補完CSVに存在するtitleKeyLoose集合（未プレイ除外判定に使用）
resultsByType       { type: entry[] }  プレイ済み結果
unplayedByType      { type: entry[] }  未プレイ候補
scoreRows           スコアCSVのパース結果
activeTab           現在のレーダータイプ ('NOTES' など)
viewMode            'played' | 'unplayed'
unplayedLevelFilter 未プレイ一覧の☆フィルタ値（''=すべて）
```

---

## データ処理フロー

### fetchRadarData()
1. レーダーCSV（6種）と補完CSVを並行取得
2. レーダーCSV → `radarMapByType[type].strict / .loose` を構築
3. 補完CSV → `supplementLevelMap`（難易度取得用）と `supplementTitleSet`（除外判定用）を構築

### processScore()（スコアCSV読み込み後）
**第1パス（resultsByType）:**
- スコアCSVの各行 × 各難易度について、clearType と score が有効なものを処理
- レーダー値 = `score / (notes×2) × radarMax`
- レーダー照合順: strict → loose → TITLE_LOOKUP_EXTRA
- 難易度レベル: スコアCSV `{難易度}` → 補完CSVフォールバック

**第2パス（unplayedByType）:**
- レーダーCSV全エントリを走査し、未プレイ（played判定外）を列挙
- `supplementTitleSet` に存在しない曲は除外（IIDX Difficulty 未収録）
- RADAR_TITLE_ALIAS のエイリアス先タイトルも照合対象
- 難易度レベル:
  - スコアCSV由来 (`levelMap`) の値が `'0'`（譜面未解禁）の場合は補完CSVを優先
  - 補完CSVにもなければ `'0'` に戻す

---

## 描画 (renderCurrentTab / renderUnplayedTab)

### プレイ済みビュー
- `titleKeyLoose(r.title)` をキーとして `repMap` で同タイトル重複排除（VOID/VØID等の表記揺れ対応）
- TOP10（重複排除後）＋弾き出しエントリ＋11位以降を最大20件表示
- 各行に「TOP10入りに必要なスコア」のガイダンスを付与

### 未プレイビュー
- `radarMax` 降順ソート後、最大20件表示
- `☆` 列ヘッダがプルダウン（`#unplayed-level-filter`）になっており難易度でフィルタ可能
- TOP10プレイ済みタイトルと同一の曲は「別難易度」として扱い、比較ガイダンスを付与

---

## maintenance.html

補完スプレッドシートの整合性チェックツール。
- レーダーCSV（NOTES）と補完CSVを照合
- **タイトル切れ**: レーダーCSVが補完CSVのprefix（75%以上一致）→ 切れ箇所を表示、CSVダウンロード修正支援
- **補完未登録**: レーダーにあって補完にない曲
- **孤立エントリ**: 補完にあってレーダーにない曲

`titleKeyLoose` は index.html と同一実装を保つこと。

---

## 既知の表記差異と対処パターン

| 問題 | 対処 |
|---|---|
| スコアCSVと表示名が違う（VOID / 0.59） | `TITLE_DISPLAY_OVERRIDE` |
| スコアCSVとレーダーCSVでタイトルが違う | `TITLE_LOOKUP_EXTRA` |
| レーダーCSVタイトルが途中で切れている | `RADAR_TITLE_ALIAS`（切れタイトル→正式タイトル） |
| レーダーCSVと補完CSVで文字が違う（Ø/0、Lisu文字等） | `RADAR_TITLE_ALIAS`（レーダー表記→補完CSV表記） |
| ひらがな/カタカナ混在 | `titleKeyLoose` 内で自動吸収 |
| Æ/æ、ə 等の特殊ラテン文字 | `titleKeyLoose` 内で自動吸収 |
