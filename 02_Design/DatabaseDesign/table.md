# DynamoDB テーブル設計書

## 1. アクセスパターン一覧
### AP-01：指定日の支出一覧を取得する
用途  
- 今日の支出を確認したい  
- 特定日の支出を確認したい

### AP-02：指定月の支出一覧を取得する
用途  
- 今月の支出合計を確認したい
- 指定月に何にお金を使ったかを確認したい

### AP-03：指定年の支出額の月次集計を取得する
用途  
- 対象年の月ごとの支出額を比較したい（横軸が月の棒グラフ）

### AP-04：指定カテゴリ × 指定月の支出額の月次集計を取得する
用途  
- 指定月はどのカテゴリの支出が多いかを確認したい（カテゴリが占める割合を示す円グラフ）
- 指定カテゴリはどの月に支出額が多いかを確認したい（横軸が月の棒グラフ）
- 予算超過の通知をしたい（支出額合計がわかればよい）

### AP-05：レシート解析結果を保存する
用途  
- レシートの解析結果から支出を分析したい  

## 2. テーブル一覧
Expenses

## 3. PK/SK 設計（メインテーブル）
### Partition Key（PK）：`userId`
ユーザー単位でデータを分離する。

### Sort Key（SK）：`date#expenseId`
- `date` により日付順でソート  
- `expenseId` により同一日の複数支出を区別  
- `BETWEEN` による月次・日次検索が高速  

例：
- PK: USER#123
- SK: 2026-09-01#EXP123

## 4. GSI 設計
採用するのは CategoryIndex のみ。

### GSI-1：CategoryIndex
- PK：`userId`  
- SK：`category#date#expenseId`

用途  
- カテゴリ別支出一覧
- カテゴリ別の月次集計
- カテゴリ別の時系列分析
- 予算超過チェック

例：
- PK: USER#123
- SK: Food#2026-09-01#EXP123

## 5. Item の JSON 構造（1アイテム）
```json
{
  "userId": "USER#123",
  "date#expenseId": "2026-09-01#EXP123",

  "date": "2026-09-01",
  "expenseId": "EXP123",

  "store": "FamilyMart",
  "total": 980,
  "category": "Food",

  "items": [
    { "name": "Sandwich", "price": 480 },
    { "name": "Coffee", "price": 500 }
  ],

  "createdAt": "2026-09-01T10:00:00Z"
}
```

## 6. テーブル定義
TableName: Expenses  
BillingMode: PROVISIONED

ProvisionedThroughput:  
  ReadCapacityUnits: 5  
  WriteCapacityUnits: 5

Primary Key:  
  PK: userId (String)  
  SK: date#expenseId (String)

Attributes:
  - userId (String)
  - date#expenseId (String)
  - date (String)
  - expenseId (String)
  - store (String)
  - total (Number)
  - category (String)
  - items (List)
  - createdAt (String)

Global Secondary Indexes:
  - CategoryIndex  
      PK: userId (String)  
      SK: category#date#expenseId (String)  
      ProvisionedThroughput:  
        ReadCapacityUnits: 5  
        WriteCapacityUnits: 5  

※ Provisioned の初期値は仮置き。実際は CloudWatch のメトリクスを見ながら調整する。

## 7. 設計のポイント
- アクセスパターン駆動設計に準拠
- PK/SK は「ユーザー × 日付順」の最適解
- GSI は CategoryIndex のみに絞り、シンプルかつ拡張性を確保
- Item 構造はレシート解析結果をそのまま保持できる
- 月次・日次・カテゴリ別の全ての検索が高速に実行可能
- Provisioned モードでコスト最適化が可能（同じキャパシティユニットを消費する場合はオンデマンドより低コスト）