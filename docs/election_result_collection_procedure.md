# 2026年衆議院選挙 開票結果収集 手順書

## 目的

2026年2月8日投開票の第51回衆議院議員総選挙について、全290小選挙区の開票結果（当落・得票数・得票率）をWEB検索で収集し、既存のJSONデータに追記・保存する。

---

## 全体フロー

```
前提      作業前にgitコミット（チェックポイント作成）
    ↓
ステップ0  収集管理用JSON作成
    ↓
ステップ1  WEB検索で結果取得（北海道→沖縄、1県ずつ順に）
    ↓
ステップ2  各選挙区の district_status.json を更新
    ↓
ステップ3  収集管理JSON を更新（進捗反映）
    ↓
ステップ4  1県完了ごとに確認 → gitコミット → 次の県へ
    ↓
ステップ5  全県完了後、最終集計
```

---

## 前提: データ保護（gitチェックポイント）

### 目的

既存の `district_status.json`（候補者情報・評価スコア・予測データ等）を安全に保護する。書き込み事故やデータ構造の破損が起きても、いつでも復元できるようにする。

### 作業開始前

```bash
# 現在の状態を確認（クリーンであること）
git status

# チェックポイントを作成
git add -A
git commit -m "選挙結果収集 作業前チェックポイント"
```

### 更新ルール

district_status.json の更新時は以下を厳守する:

1. **追記のみ**: 既存のキー（`scores`, `prediction`, `processing` 等）は一切変更・削除しない
2. **新規キーの追加だけ**: 候補者に `election_result`、トップレベルに `election_summary` を追加するのみ
3. **読み込み→追記→書き戻し**: ファイル全体を読み込み、フィールドを追加して書き戻す。手動でJSONを構築しない

### 復元方法（万が一の場合）

```bash
# 直前のコミットに戻す（1県分だけ巻き戻し）
git diff result/01_北海道/    # 変更内容を確認
git checkout -- result/01_北海道/  # 北海道フォルダだけ復元

# 全体を作業開始前に戻す
git log --oneline -5           # コミットを確認
git checkout <commit-hash> -- result/  # result全体を復元
```

---

## ステップ0: 収集管理用JSONの作成

### ファイル

`result/election_result_status.json`

### 目的

全290選挙区の結果収集進捗を1ファイルで管理する。既存の `status.json`（評価処理の進捗管理）とは別に、選挙結果の収集状況のみを追跡する。

### フォーマット

```json
{
  "election": "第51回衆議院議員総選挙",
  "date": "2026-02-08",
  "last_updated": "2026-02-08T20:00:00+09:00",
  "summary": {
    "total_districts": 290,
    "collected": 0,
    "remaining": 290
  },
  "prefectures": {
    "01": {
      "name": "北海道",
      "total_districts": 12,
      "collected": 0,
      "districts": {
        "1": "pending",
        "2": "pending",
        "...": "..."
      }
    }
  }
}
```

### 選挙区ステータス定義

| 値 | 意味 |
|----|------|
| `pending` | 未収集 |
| `collected` | 結果取得済み・JSON更新済み |
| `not_found` | 検索したが結果未確定/取得不可 |

### 生成方法

既存の `result/status.json` の全選挙区情報を読み取り、全選挙区を `"pending"` で初期化して生成する。

---

## ステップ1: WEB検索による結果取得

### 検索順序

北海道（01）→ 青森（02）→ ... → 沖縄（47）の順に、1県ずつ処理する。

### 検索クエリ例

```
2026年 衆議院選挙 北海道1区 開票結果
2026年 衆院選 北海道1区 得票数
```

情報源の優先順位:
1. NHK選挙速報
2. 各新聞社の選挙特設サイト（読売・朝日・毎日等）
3. 総務省・各選挙管理委員会

### 取得する情報（必須）

| 項目 | 型 | 説明 |
|------|----|------|
| 候補者名 | string | 既存候補者との照合用 |
| 当落 | string | `"当選"` / `"落選"` / `"比例復活"` |
| 得票数 | int | 個人の得票数 |
| 得票率 | float | 得票率（%） |

### 取得する情報（任意・取得できれば）

| 項目 | 型 | 説明 |
|------|----|------|
| 投票率 | float | 選挙区全体の投票率（%） |

### 注意事項

- 候補者名は既存の `district_status.json` の `candidates[].name` と照合する
- 開票途中の数値は取得しない（確定値のみ）
- 比例復活当選は小選挙区では `"落選"`、比例復活情報がある場合は `"比例復活"` とする

---

## ステップ2: district_status.json の更新

### 更新対象ファイル

```
result/{県コード}_{県名}/{N}区/district_status.json
```

例: `result/01_北海道/01区/district_status.json`

### 候補者への追記

各候補者オブジェクトに `election_result` フィールドを追加する。

**更新前:**
```json
{
  "name": "道下大樹",
  "party": "中道改革連合",
  "status": "前職",
  "scores": { ... },
  "prediction": { ... }
}
```

**更新後:**
```json
{
  "name": "道下大樹",
  "party": "中道改革連合",
  "status": "前職",
  "scores": { ... },
  "prediction": { ... },
  "election_result": {
    "result": "当選",
    "votes": 95000,
    "vote_rate": 42.3
  }
}
```

### election_result フィールド定義

| キー | 型 | 必須 | 説明 |
|------|----|------|------|
| `result` | string | Yes | `"当選"` / `"落選"` / `"比例復活"` |
| `votes` | int | Yes | 得票数 |
| `vote_rate` | float | Yes | 得票率（%、小数第1位まで） |

### 選挙区レベルへの追記

district_status.json のトップレベルに `election_summary` を追加する。

```json
{
  "prefecture_code": "01",
  "district_name": "北海道1区",
  "status": "completed",
  "candidates": [ ... ],
  "election_summary": {
    "winner": "道下大樹",
    "winner_party": "中道改革連合",
    "turnout": 55.2,
    "result_updated": "2026-02-08T23:00:00+09:00"
  }
}
```

### election_summary フィールド定義

| キー | 型 | 必須 | 説明 |
|------|----|------|------|
| `winner` | string | Yes | 当選者名 |
| `winner_party` | string | Yes | 当選者の政党 |
| `turnout` | float \| null | No | 投票率（%）。不明なら `null` |
| `result_updated` | string | Yes | 更新日時（ISO 8601） |

---

## ステップ3: 収集管理JSONの更新

1選挙区の結果をdistrict_status.jsonに書き込んだら、`election_result_status.json` を更新する。

```
該当選挙区: "pending" → "collected"
該当県の collected: +1
全体の summary.collected: +1, summary.remaining: -1
last_updated: 現在時刻に更新
```

---

## ステップ4: 1県完了ごとの確認・コミット

1つの県の全選挙区が完了したら、次の県に進む前に確認・コミットする。

### 確認内容

- [ ] 全選挙区のステータスが `"collected"` になっているか
- [ ] 各候補者の `election_result` が漏れなく追記されているか
- [ ] 当選者が各選挙区に1名ずつ存在するか
- [ ] 得票率の合計がおおむね100%前後になるか（無効票分のずれは許容）

### gitコミット（1県ごとのチェックポイント）

```bash
# 変更内容を確認
git diff result/01_北海道/

# 問題なければコミット
git add result/01_北海道/ result/election_result_status.json
git commit -m "選挙結果収集: 北海道（12選挙区）"
```

1県ごとにコミットすることで、問題発生時にその県だけロールバックできる。

---

## ステップ5: 全県完了後の最終集計

全47都道府県の収集が完了したら、以下を行う。

### 5-1. election_result_status.json の最終確認

```json
{
  "summary": {
    "total_districts": 290,
    "collected": 290,
    "remaining": 0
  }
}
```

### 5-2. 全体集計の作成（任意）

必要に応じて政党別当選者数などの集計を行う。

---

## ファイル一覧

| ファイル | 用途 | 更新タイミング |
|---------|------|---------------|
| `result/election_result_status.json` | 収集進捗管理（新規） | 毎選挙区完了時 |
| `result/{県}/{N}区/district_status.json` | 候補者別結果保存（既存に追記） | 毎選挙区完了時 |
| `result/status.json` | 既存の評価進捗（今回は変更しない） | - |

---

## 運用上の注意

1. **開票速報と確定値の区別**: 速報値ではなく「確定」と明示されたデータのみ収集する
2. **候補者名の照合**: WEB上の表記と既存JSONの `name` が一致しない場合（旧姓等）は手動で対応
3. **比例復活**: 小選挙区の当落は `"落選"` として記録し、比例復活が確認できた場合に `"比例復活"` に更新する。比例復活情報は開票当日夜〜翌日に確定するため、後日更新で対応可
4. **中断・再開**: `election_result_status.json` の `"pending"` の選挙区から再開すればよい
5. **1回の検索範囲**: 効率のため、1県まとめて検索して結果を取得し、その後まとめて各選挙区JSONを更新する

---

## データ保護チェックリスト

作業全体を通じて守るべきルール:

- [ ] 作業開始前に `git commit` でチェックポイントを作成したか
- [ ] district_status.json の更新は **フィールド追記のみ** か（既存キーを変更していないか）
- [ ] 1県完了ごとに `git commit` しているか
- [ ] 書き戻し後のJSONが valid か（パースエラーがないか）
- [ ] 既存の `scores`, `prediction`, `processing` が変更されていないか
