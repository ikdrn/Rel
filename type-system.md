# Rel 型システム 完全仕様書 v0.2

## 概要

RelはHindley-Milnerをベースに、エンティティグラフ向けの拡張を加えた静的型システム。

**型システムが解決する問題:**
- SQLの「実行時まで型エラーがわからない」問題
- NULLの三値論理による混乱
- JOINの型安全性の欠如（存在しないカラムを参照できてしまう）
- 集計文脈での非集計フィールド参照（SQLのよくあるバグ源）

---

## 1. 型階層

```
┌─────────────────────────────────────────────────────────────┐
│                      Rel Type System                        │
│                                                             │
│  Scalar                                                     │
│  ├── Numeric:  Int, Int8..Int64, UInt, Float, Decimal(p,s)  │
│  ├── Text:     String, String(n), Text                      │
│  ├── Bool                                                   │
│  ├── Temporal: Date, Time, Timestamp, Duration              │
│  ├── Binary:   Bytes                                        │
│  ├── Identity: UUID, ULID, Snowflake                        │
│  ├── Network:  IpAddr, Url, Email                           │
│  └── Geo:      Point, Polygon                               │
│                                                             │
│  Container                                                  │
│  ├── Option<T>     (T?)                                     │
│  ├── List<T>                                                │
│  ├── Set<T>                                                 │
│  ├── Map<K,V>                                               │
│  └── Stream<T>                                             │
│                                                             │
│  Structural                                                 │
│  ├── Record:  { field: T, ... }                             │
│  ├── Enum:    enum E { A, B(x:T), C(x:T,y:T) }             │
│  └── Union:   T1 | T2 | T3                                  │
│                                                             │
│  Query Types                                                │
│  ├── Pattern<E>    エンティティパターン                      │
│  ├── Result<T>     クエリの結果型                           │
│  └── Stream<T>     ストリーム処理結果                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Option型（NULLの廃止）

```
SQLの問題:
  NULL == NULL    → NULL  （!!! 直感に反する）
  NULL != NULL    → NULL  （!!! 直感に反する）
  1 + NULL        → NULL  （伝播する）
  NOT NULL        → NULL  （三値論理）
  WHERE x = NULL  → 何もマッチしない（IS NULLを使わないといけない）

Relの解決:
  None == None    → false  （エンティティの同一性は別の問題）
  None != None    → false
  1 + None        → コンパイルエラー（Int + Option<Int> は型不一致）
  not None        → コンパイルエラー（Bool 以外に not は使えない）
  { phone: None } → None チェックの明示的な記法
```

```rel
-- 型宣言
phone: String?       -- Option<String> の省略記法

-- 操作
user.phone ?? "N/A"           -- デフォルト値（None なら "N/A"）
user.phone?.length()          -- オプショナルチェーン（None なら None）
user.phone!                   -- 強制アンラップ（None ならパニック）
user.phone.expect("required") -- メッセージ付きアンラップ

-- パターンマッチ（最も安全）
user.phone match {
  Some(p) => use(p),
  None    => fallback()
}

-- Option同士の演算
Some(5) + Some(3)  → コンパイルエラー（Option<Int> + Option<Int> は不正）
-- 正しくは:
(user.age ?? 0) + (other.age ?? 0)  → Int + Int = Int  ✓
```

---

## 3. 代数的データ型（ADT）

```rel
-- バリアントがデータを持てる
enum OrderStatus {
  Draft
  Pending
  Processing { started_at: Timestamp }
  Shipped    { tracking: String, carrier: String }
  Delivered  { at: Timestamp }
  Cancelled  { reason: String, refund: Price? }
}

-- コンストラクト
let s = OrderStatus.Shipped { tracking: "1234-5678", carrier: "Yamato" }

-- デストラクト（パターンマッチで型安全に値を取り出す）
order.status match {
  Draft              => "下書き",
  Pending            => "受付中",
  Processing { started_at } => "処理中: " ++ started_at.format("HH:mm"),
  Shipped { tracking, carrier } => carrier ++ ": " ++ tracking,
  Delivered { at }   => "配達済み: " ++ at.format("MM/dd"),
  Cancelled { reason, refund } => match refund {
    Some(r) => "ＣＡ(" ++ reason ++ ") 返金: " ++ r.to_string(),
    None    => "ＣＡ(" ++ reason ++ ")"
  }
}
```

---

## 4. エンティティ型とリレーション型

```rel
-- エンティティはその名前が型になる
let u: User = User { id: $id } one!

-- リレーション探索の型
User -> .orders -> Order
-- 型: Stream<(User, Order)>

-- 抽出後の型
User { age > 18 }
=> { .id, .name }
-- 型: Stream<{ id: UUID, name: String }>

-- per() 集計後の型
User -> .orders -> Order
=> per(user) { user.name, total: sum(order.amount) }
-- 型: Stream<{ name: String, total: Decimal }>
```

---

## 5. 型推論

```rel
-- 全て型推論される（アノテーション不要）
let count    = User { age > 18 } count()      -- Int
let names    = User => .name                   -- Stream<String>
let user     = User { id: $id } one            -- User
let maybe    = User { email: $e } first        -- User?
let revenue  = Order { status: Delivered } sum(.amount)  -- Decimal

-- 型エラーは全てコンパイル時に検出
User { age > "adult" }
-- Error[E001]: Type mismatch: age(Int) > "adult"(String)

User -> .nonexistent -> Foo
-- Error[E042]: Relation 'nonexistent' not found on User

User { age > 18 } => per(.country) { .id }
-- Error[E050]: 'id' is neither in per() key nor an aggregate
--   hint: Use first(.id), collect(.id), or add to per()
```

---

## 6. 型安全な集計文脈

SQLの有名なバグ: `SELECT user_id, name, SUM(amount) FROM orders GROUP BY user_id`
→ `name` は GROUP BY にないが、多くのDB方言でエラーにならない（結果は不定）

```rel
-- Rel では集計文脈を型システムが追跡する
Order -> .user -> User
=> per(user) {
  user.name,           -- OK: per(user) のキーに含まれる
  user.id,             -- OK: per(user) のキーに含まれる
  order.id,            -- ERROR[E050]: orderはper()のキーではない
  count(order),        -- OK: 集計関数
  sum(order.amount)    -- OK: 集計関数
}
```

---

## 7. 型定義のまとめ

```rel
-- プリミティブ型
Int           : 64bit 整数
Float         : 64bit IEEE 754
Decimal(p,s)  : 固定精度小数（金融計算）
String        : UTF-8 可変長文字列
String(n)     : 最大n文字
Bool          : true | false
Date          : YYYY-MM-DD
Timestamp     : UTC日時
UUID          : RFC 4122 UUID

-- コンテナ型
T?            : Option<T> = Some(T) | None
List<T>       : 順序付きリスト
Set<T>        : 重複なしセット
Map<K,V>      : キーバリューマップ

-- 構造型
{ field: T }  : 匿名レコード型（=> の結果型に使われる）
enum E { ... } : 代数的データ型
type A = B    : 型エイリアス（バリデーション付き可）

-- 型演算
T | U         : ユニオン型（どちらかの型）
T & U         : インターセクション型（両方を満たす）
```
