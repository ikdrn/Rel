# Rel — Relation Language
## 次世代データベース言語 完全設計仕様書

**バージョン**: 0.1.0-draft
**設計日**: 2026-03-06
**ステータス**: RFC (Request for Comments)

---

## 目次

1. [言語名と概要](#1-言語名と概要)
2. [言語の哲学](#2-言語の哲学)
3. [基本クエリ構文](#3-基本クエリ構文)
4. [JOINの再発明 — リレーション探索](#4-joinの再発明--リレーション探索)
5. [GROUP BY / 集計](#5-group-by--集計)
6. [ORDER / LIMIT / PAGINATION](#6-order--limit--pagination)
7. [スキーマ定義](#7-スキーマ定義)
8. [リレーション定義](#8-リレーション定義)
9. [型システム](#9-型システム)
10. [クエリの再利用](#10-クエリの再利用)
11. [モジュールシステム](#11-モジュールシステム)
12. [実行モデル](#12-実行モデル)
13. [分散クエリ](#13-分散クエリ)
14. [SQLとの比較](#14-sqlとの比較)

---

## 1. 言語名と概要

### 言語名: **Rel**
**Rel** = **Rel**ation Language

### コアコンセプト

```
Rel は「データの流れ」を表現する言語である。
SQLが「何を取得するか」を宣言するのに対して、
Relは「どのようにデータが流れるか」を記述する。
```

### ファイル拡張子
- `.rel`  — クエリファイル
- `.relschema` — スキーマ定義ファイル
- `.relmod` — モジュールファイル

### バージョニング
```rel
#!rel 0.1
```

---

## 2. 言語の哲学

### 2.1 設計思想

Relは以下の7つの原則に基づいて設計された。

```
┌─────────────────────────────────────────────────────────┐
│                    Rel Design Pillars                   │
├─────────────────────────────────────────────────────────┤
│  1. FLOW       データは左から右へ流れる                 │
│  2. SAFETY     型安全、null安全が言語レベルで保証        │
│  3. COMPOSE    クエリは関数のように合成できる           │
│  4. RELATE     リレーションは一級市民である             │
│  5. DISTRIBUTE 分散処理はネイティブに組み込まれている   │
│  6. READABLE   コードは散文のように読める               │
│  7. MINIMAL    構文は最小限、表現力は最大限             │
└─────────────────────────────────────────────────────────┘
```

### 2.2 SQLとの根本的な違い

| 観点 | SQL | Rel |
|------|-----|-----|
| 記述パラダイム | 宣言的（結果を指定） | データフロー型（変換を記述） |
| 読み順 | SELECT→FROM→WHERE（実行順と逆） | FROM→FILTER→SELECT（実行順と一致） |
| NULL処理 | 三値論理（true/false/null） | Option型（Some/None） |
| 型システム | 弱い型（暗黙キャスト多発） | 強い静的型＋型推論 |
| JOIN | キーワードとON句の組み合わせ | リレーション探索演算子 |
| 再利用 | ビュー・ストアドプロシージャ | first-class query |
| モジュール | スキーマ単位のみ | module/namespace/import |
| 分散処理 | 拡張機能・方言依存 | ネイティブ組み込み |
| エラー | ランタイムエラー多発 | コンパイル時に多くを検出 |

### 2.3 解決する問題

```
Problem 1: 書く順序と実行順序の不一致
  SQL:  SELECT name FROM users WHERE age > 18
        ↑第3段階   ↑第1段階  ↑第2段階
  Rel:  users |> where age > 18 |> select name
        ↑第1段階   ↑第2段階       ↑第3段階

Problem 2: JOINの複雑さ
  SQL:  FROM orders JOIN users ON orders.user_id = users.id
  Rel:  orders -> user   (リレーションが定義済みなら自動解決)

Problem 3: NULLの三値論理
  SQL:  NULL != NULL  → NULL (!)
  Rel:  None != None  → false (Option型で明示的に扱う)

Problem 4: ネストしたサブクエリ
  SQL:  SELECT * FROM (SELECT * FROM (SELECT ...))
  Rel:  パイプラインで平坦に記述

Problem 5: 型安全性の欠如
  SQL:  age + 'hello'  → 実行時エラーまたは暗黙変換
  Rel:  age + 'hello'  → コンパイルエラー (Int + String は不正)
```

---

## 3. 基本クエリ構文

### 3.1 構文の基本形

Relのクエリは **パイプライン演算子 `|>`** を使って記述する。

```
<source> |> <operator> |> <operator> |> ... |> <output>
```

### 3.2 SQLとの比較

**SQL:**
```sql
SELECT name
FROM users
WHERE age > 18
```

**Rel:**
```rel
from users
|> where .age > 18
|> select .name
```

### 3.3 構文の詳細説明

```rel
-- 1. データソースを指定
from users

-- 2. フィルタリング（パイプで接続）
|> where .age > 18

-- 3. フィールド選択
|> select .name
```

**ドット記法**: `.field` は現在のレコードのフィールドを参照する。

### 3.4 複数フィールドの選択

```rel
from users
|> where .age > 18
|> select {
     id:   .id,
     name: .name,
     age:  .age
   }
```

**省略記法**: フィールド名と変数名が同じ場合は省略可能
```rel
from users
|> where .age > 18
|> select { .id, .name, .age }
```

### 3.5 フィールドの変換（computed fields）

```rel
from users
|> where .age > 18
|> select {
     .name,
     full_label: .name ++ " (age: " ++ .age.to_string() ++ ")"
   }
```

### 3.6 完全な構文図

```
┌──────────────────────────────────────────────────────────────┐
│  Query Pipeline                                              │
│                                                              │
│  from <source>                                               │
│    │                                                         │
│    ▼                                                         │
│  |> where <predicate>         ← filter rows                  │
│    │                                                         │
│    ▼                                                         │
│  |> join / navigate           ← traverse relations           │
│    │                                                         │
│    ▼                                                         │
│  |> group by <fields>         ← aggregation                  │
│    │                                                         │
│    ▼                                                         │
│  |> having <predicate>        ← filter after group           │
│    │                                                         │
│    ▼                                                         │
│  |> order by <fields>         ← sorting                      │
│    │                                                         │
│    ▼                                                         │
│  |> take <n> skip <m>         ← pagination                   │
│    │                                                         │
│    ▼                                                         │
│  |> select <projection>       ← output shape                 │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. JOINの再発明 — リレーション探索

### 4.1 設計思想

SQLのJOINは「どのキーで結合するか」をクエリのたびに記述する。
Relでは **リレーションをスキーマに定義し、クエリ時は探索するだけ**。

```
SQLのアプローチ: JOIN users ON orders.user_id = users.id
Relのアプローチ: orders -> user   (リレーションは事前定義済み)
```

### 4.2 SQLとの比較

**SQL:**
```sql
SELECT orders.id, users.name
FROM orders
JOIN users ON orders.user_id = users.id
```

**Rel:**
```rel
from orders
|> navigate .user          -- リレーション探索
|> select { orders.id, user.name }
```

または **矢印記法**（インライン）:
```rel
from orders -> user
|> select { orders.id, user.name }
```

### 4.3 ナビゲーション演算子

```
演算子   意味                          対応するSQL
─────────────────────────────────────────────────────────
->       INNER JOIN (存在するもののみ)   INNER JOIN
-?>      LEFT JOIN (任意のリレーション)  LEFT JOIN
<-       逆方向のリレーション探索        逆JOIN
<->      双方向探索                     FULL OUTER JOIN
~>       グラフ探索（再帰）             WITH RECURSIVE
```

### 4.4 複数リレーションの連鎖

```rel
-- orders → user → address の3段階リレーション
from orders -> user -> address
|> where address.country == "Japan"
|> select {
     order_id:    orders.id,
     user_name:   user.name,
     city:        address.city
   }
```

これは以下のSQLと等価:
```sql
SELECT orders.id, users.name, addresses.city
FROM orders
JOIN users ON orders.user_id = users.id
JOIN addresses ON users.address_id = addresses.id
WHERE addresses.country = 'Japan'
```

### 4.5 オプショナルリレーション（LEFT JOIN相当）

```rel
from orders -?> coupon
|> select {
     order_id:  .id,
     -- coupon は Option<Coupon> 型
     discount:  coupon?.discount_rate ?? 0.0
   }
```

**`-?>`** を使うと、リレーション先が存在しない場合は `None` になる（LEFT JOIN相当）。

### 4.6 複数リレーション（多対多）

```rel
-- ユーザーの全タグを取得（多対多）
from users -> tags   -- many-to-many (中間テーブル自動解決)
|> group by users.id, users.name
|> select {
     .id,
     .name,
     tags: collect(tags.name)
   }
```

### 4.7 グラフ探索（再帰的リレーション）

```rel
-- 組織階層の再帰探索
from employees ~> reports_to(depth: 1..10)
|> where .department == "Engineering"
|> select { .id, .name, .level }
```

---

## 5. GROUP BY / 集計

### 5.1 SQLとの比較

**SQL:**
```sql
SELECT country, COUNT(*)
FROM users
GROUP BY country
```

**Rel:**
```rel
from users
|> group by .country
|> select {
     country: .country,
     count:   count()
   }
```

### 5.2 集計関数一覧

```rel
-- 標準集計関数
count()              -- 行数
count(.field)        -- NULL除外カウント
count(distinct .field) -- 重複除外カウント
sum(.field)          -- 合計
avg(.field)          -- 平均
min(.field)          -- 最小値
max(.field)          -- 最大値
stddev(.field)       -- 標準偏差
variance(.field)     -- 分散
median(.field)       -- 中央値

-- コレクション系
collect(.field)              -- リストに収集
collect(distinct .field)     -- 重複除外リスト
first(.field)                -- 最初の値
last(.field)                 -- 最後の値
```

### 5.3 複合集計

```rel
from orders
|> group by .status, .country
|> select {
     status:        .status,
     country:       .country,
     total_orders:  count(),
     total_revenue: sum(.amount),
     avg_order:     avg(.amount),
     max_order:     max(.amount)
   }
|> order by total_revenue desc
```

### 5.4 HAVING相当（集計後フィルタ）

```rel
from users
|> group by .country
|> having count() > 100          -- 集計後フィルタ
|> select {
     country: .country,
     count:   count()
   }
```

### 5.5 ウィンドウ関数

```rel
from orders
|> window {
     rank:       rank() over (partition by .user_id order by .created_at desc),
     running_sum: sum(.amount) over (partition by .user_id order by .created_at)
   }
|> where .rank <= 3        -- 各ユーザーの直近3件
|> select { .id, .user_id, .amount, .rank, .running_sum }
```

---

## 6. ORDER / LIMIT / PAGINATION

### 6.1 ソート構文

```rel
from products
|> order by .price desc, .name asc
|> select { .name, .price }
```

複数条件:
```rel
|> order by (.category asc, .price desc, .name asc)
```

### 6.2 ページング

```rel
-- 最初の20件
from users
|> order by .created_at desc
|> take 20

-- オフセットベース (21〜40件目)
from users
|> order by .created_at desc
|> take 20 skip 20

-- カーソルベース（推奨：分散環境）
from users
|> order by .created_at desc
|> after cursor:"eyJpZCI6MTAwfQ=="
|> take 20
```

### 6.3 カーソルページング詳細

```rel
-- カーソルページングクエリ（分散環境推奨）
from orders
|> where .status == "active"
|> order by .created_at desc, .id desc
|> cursor_paginate {
     after:    $cursor,     -- 入力パラメータ
     per_page: 50
   }
|> select {
     edges: {
       node: { .id, .created_at, .amount },
       cursor: encode_cursor(.created_at, .id)
     },
     page_info: {
       has_next_page,
       end_cursor
     }
   }
```

---

## 7. スキーマ定義

### 7.1 基本テーブル定義

**SQL:**
```sql
CREATE TABLE users (
  id   UUID,
  name TEXT,
  age  INT
)
```

**Rel:**
```rel
entity User {
  id:         UUID          @primary @default(gen_uuid())
  name:       String(255)   @required
  age:        Int           @check(age >= 0 and age <= 150)
  email:      String        @unique @index
  created_at: Timestamp     @default(now())
  updated_at: Timestamp     @auto_update(now())
}
```

### 7.2 スキーマ定義の詳細

```rel
entity Product {
  -- 基本フィールド
  id:           UUID          @primary @default(gen_uuid())
  name:         String(500)   @required
  description:  String?       -- オプショナル（nullable）
  price:        Decimal(10,2) @check(price > 0)
  stock:        Int           @default(0) @check(stock >= 0)
  tags:         List<String>  @default([])
  metadata:     Json          @default({})

  -- 時系列フィールド
  created_at:   Timestamp     @default(now())
  updated_at:   Timestamp     @auto_update(now())
  deleted_at:   Timestamp?    @index  -- ソフトデリート用

  -- インデックス定義
  @index([name, price])
  @index([created_at desc])

  -- テーブルレベル制約
  @check(stock >= 0 or deleted_at != None)
}
```

### 7.3 アノテーション一覧

```
アノテーション              意味
─────────────────────────────────────────────────────────────
@primary                   主キー
@default(expr)             デフォルト値
@auto_update(expr)         更新時に自動設定
@required                  NOT NULL
@unique                    ユニーク制約
@index                     インデックス作成
@index([fields])           複合インデックス
@check(expr)               チェック制約
@foreign(Entity.field)     外部キー（通常はrelationで定義）
@deprecated                非推奨フィールドマーク
@computed(expr)            計算フィールド（ストアしない）
@encrypted                 保存時に暗号化
@immutable                 作成後変更不可
```

### 7.4 スキーマのマイグレーション

```rel
-- マイグレーション定義
migration add_user_phone_v2 {
  version: "2026-03-06-001"
  description: "Add phone field to users"

  up {
    alter User {
      add phone: String?
      add phone_verified: Bool @default(false)
    }
    add_index User.phone
  }

  down {
    alter User {
      drop phone
      drop phone_verified
    }
  }
}
```

---

## 8. リレーション定義

### 8.1 基本的なリレーション定義

```rel
-- 1対多: User has many Orders
relation User.orders -> Order[] via Order.user_id

-- 多対1: Order belongs to User
relation Order.user -> User via Order.user_id

-- 1対1: User has one Profile
relation User.profile -> Profile? via Profile.user_id
```

### 8.2 多対多リレーション

```rel
-- 中間テーブルを明示
relation User.tags <-> Tag[] via UserTag {
  user_id: UUID -> User.id
  tag_id:  UUID -> Tag.id
}
```

### 8.3 自己参照リレーション

```rel
entity Employee {
  id:         UUID    @primary
  name:       String
  manager_id: UUID?   -- 自己参照
}

-- 自己参照リレーション
relation Employee.manager   -> Employee? via Employee.manager_id
relation Employee.reports   -> Employee[] via Employee.manager_id
```

### 8.4 多態的リレーション（Polymorphic）

```rel
-- コメントは複数のエンティティにつけられる
entity Comment {
  id:           UUID    @primary
  body:         String
  target_type:  String  -- "Post" | "Product" | "User"
  target_id:    UUID
}

relation Comment.target -> Post | Product | User via (target_type, target_id)
```

### 8.5 カスタムリレーション（条件付き）

```rel
-- アクティブな注文のみのリレーション
relation User.active_orders -> Order[]
  via Order.user_id
  where Order.status in ["pending", "processing"]
  order by Order.created_at desc
```

### 8.6 グラフリレーション（再帰）

```rel
entity Category {
  id:        UUID   @primary
  name:      String
  parent_id: UUID?
}

-- 再帰的リレーション（ツリー構造）
relation Category.parent    -> Category?   via Category.parent_id
relation Category.children  -> Category[]  via Category.parent_id
relation Category.ancestors -> Category[]  recursive via Category.parent_id
```

---

## 9. 型システム

### 9.1 プリミティブ型

```rel
-- 数値型
Int          -- 64bit 整数 (-9,223,372,036,854,775,808 ~ 9,223,372,036,854,775,807)
Int8         -- 8bit 整数
Int16        -- 16bit 整数
Int32        -- 32bit 整数
Int64        -- 64bit 整数 (Intと同じ)
UInt         -- 符号なし64bit整数
Float        -- 64bit 浮動小数点 (IEEE 754 double)
Float32      -- 32bit 浮動小数点
Decimal(p,s) -- 固定精度小数 (金融計算向け)

-- 文字列型
String       -- 可変長UTF-8文字列
String(n)    -- 最大n文字のUTF-8文字列
Char(n)      -- 固定長n文字
Text         -- 無制限テキスト

-- 真偽値型
Bool         -- true | false

-- 時間型
Date         -- 日付 (YYYY-MM-DD)
Time         -- 時刻 (HH:MM:SS.mmm)
Timestamp    -- 日時 (UTC)
TimestampTZ  -- タイムゾーン付き日時
Duration     -- 時間間隔
Interval     -- 期間

-- バイナリ型
Bytes        -- 可変長バイナリ
Bytes(n)     -- 固定長バイナリ

-- 識別子型
UUID         -- UUID v4/v7
ULID         -- ソート可能な一意ID
Snowflake    -- 分散環境向けID

-- ネットワーク型
IpAddr       -- IPv4 / IPv6 (自動判定)
MacAddr      -- MACアドレス
Url          -- バリデート済みURL
Email        -- バリデート済みメールアドレス

-- 地理型
Point        -- 地理座標 (lat, lon)
Polygon      -- 多角形
GeoHash      -- GeoHash文字列
```

### 9.2 コンテナ型

```rel
-- リスト型（順序あり、重複可）
List<T>           -- 例: List<String>, List<Int>

-- セット型（順序なし、重複不可）
Set<T>            -- 例: Set<UUID>

-- マップ型（キーバリュー）
Map<K, V>         -- 例: Map<String, Int>

-- タプル型（異種複数値）
(T1, T2, T3)      -- 例: (String, Int, Bool)

-- JSON型
Json              -- 任意のJSON値
Json<T>           -- 型付きJSON（スキーマ検証あり）
```

### 9.3 Option型（Nullableの再設計）

```rel
-- SQLのNULLは廃止。代わりにOption型を使用
String?   -- Option<String> の省略記法 = Some(value) | None

-- Optionの操作
user.email ?? "no-email"           -- デフォルト値
user.email?.to_upper()             -- optional chaining
user.email.unwrap()                -- 値を取り出す（Noneならエラー）
user.email.expect("email required") -- カスタムエラーメッセージ付きunwrap

-- パターンマッチング
match user.phone {
  Some(p) => send_sms(p),
  None    => send_email(user.email)
}
```

### 9.4 カスタム型

```rel
-- 型エイリアス
type UserId   = UUID
type Email    = String  @validate(is_email)
type Price    = Decimal(10,2) @check(value > 0)
type Quantity = Int @check(value >= 0)

-- 構造体型
type Address = {
  street:   String,
  city:     String,
  country:  CountryCode,
  zip:      String?
}

-- ユニオン型
type PaymentMethod = "credit_card" | "bank_transfer" | "crypto"

-- 代数的データ型
type OrderStatus =
  | Pending
  | Processing { started_at: Timestamp }
  | Shipped    { tracking_id: String, carrier: String }
  | Delivered  { delivered_at: Timestamp }
  | Cancelled  { reason: String, refund_amount: Decimal? }
```

### 9.5 Enum型

```rel
enum Status {
  Active   = "active"
  Inactive = "inactive"
  Pending  = "pending"
  Deleted  = "deleted"
}

enum Priority {
  Low    = 1
  Medium = 2
  High   = 3
  Critical = 4
}

-- Enumの使用
from tickets
|> where .priority >= Priority.High
|> select { .id, .title, .priority }
```

### 9.6 型推論

```rel
-- 型アノテーションなしでも型は推論される
let user_count = from users |> count()
-- user_count の型: Int （推論）

let names = from users |> select .name
-- names の型: Stream<{ name: String }> （推論）

-- 型エラーの例（コンパイル時に検出）
from users
|> where .name > 100   -- ERROR: String > Int は不正
|> select .name
```

### 9.7 型システム階層図

```
                    ┌──────────┐
                    │   Any    │ (内部型、ユーザー使用不可)
                    └────┬─────┘
          ┌──────────────┼──────────────┐
     ┌────┴───┐    ┌─────┴────┐   ┌────┴────┐
     │Scalar  │    │Compound  │   │Special  │
     └────┬───┘    └─────┬────┘   └────┬────┘
          │              │              │
    ┌─────┼─────┐   ┌────┼────┐   ┌────┼────┐
  Numeric String Bool List Set Map Option Stream
    │       │
  Int    String(n)
  Float  Text
  Decimal Email
```

---

## 10. クエリの再利用

### 10.1 名前付きクエリ

```rel
-- クエリを名前付きで定義
query adult_users {
  from users
  |> where .age >= 18
  |> where .status == Status.Active
}

-- 再利用
from adult_users
|> order by .name
|> select { .id, .name }
```

### 10.2 パラメータ付きクエリ

```rel
-- パラメータを持つクエリ
query users_by_country(country: String) {
  from users
  |> where .country == country
  |> order by .name
}

query users_by_age_range(min_age: Int, max_age: Int) {
  from users
  |> where .age >= min_age and .age <= max_age
}

-- 使用例
from users_by_country("Japan")
|> select { .name, .email }

from users_by_age_range(20, 30)
|> select { .name, .age }
```

### 10.3 クエリの合成

```rel
-- 基底クエリ
query active_users {
  from users
  |> where .status == Status.Active
}

-- 合成クエリ（active_usersを拡張）
query premium_users {
  from active_users              -- 既存クエリを参照
  |> where .plan == "premium"
}

query premium_users_japan {
  from premium_users             -- さらに合成
  |> where .country == "Japan"
}
```

### 10.4 クエリとしての関数

```rel
-- 高階クエリ（クエリを引数に取る）
query with_pagination(base: Query<T>, page: Int, per_page: Int) -> Query<T> {
  from base
  |> order by .id
  |> take per_page skip (page * per_page)
}

-- 使用
from with_pagination(adult_users, page: 0, per_page: 20)
|> select { .id, .name }
```

### 10.5 クエリのマテリアライズ

```rel
-- 事前計算してキャッシュするマテリアライズドクエリ
@materialized(refresh: every 1h)
query daily_revenue_stats {
  from orders
  |> where .created_at >= today() - 30.days
  |> group by date(.created_at), .country
  |> select {
       date:    date(.created_at),
       country: .country,
       revenue: sum(.amount),
       orders:  count()
     }
}
```

---

## 11. モジュールシステム

### 11.1 モジュール定義

```rel
-- analytics.relmod
module analytics {

  -- モジュールレベルのインポート
  import core.types { UUID, Timestamp, Decimal }
  import entities   { User, Order, Product }

  -- モジュール内クエリ
  export query revenue_by_country {
    from orders
    |> group by .country
    |> select {
         country: .country,
         revenue: sum(.amount)
       }
    |> order by revenue desc
  }

  export query top_customers(limit: Int = 10) {
    from users -> orders
    |> group by users.id, users.name
    |> select {
         id:            users.id,
         name:          users.name,
         total_spent:   sum(orders.amount),
         order_count:   count(orders.id)
       }
    |> order by total_spent desc
    |> take limit
  }

  -- モジュール内プライベートクエリ（export なし）
  query _base_orders {
    from orders
    |> where .status != "cancelled"
  }

}
```

### 11.2 モジュールのインポート

```rel
-- 単一インポート
import analytics.revenue_by_country

-- 複数インポート
import analytics { revenue_by_country, top_customers }

-- 名前空間付きインポート
import analytics as an

-- 全インポート
import analytics.*

-- 使用例
from an.revenue_by_country
|> take 5
```

### 11.3 名前空間

```rel
namespace ecommerce {

  namespace analytics {
    export query ...
  }

  namespace inventory {
    export query ...
  }

}

-- 使用
import ecommerce.analytics.*
```

### 11.4 モジュールの依存関係管理

```rel
-- rel.lock (依存定義ファイル)
module_config {
  name:    "my_analytics"
  version: "1.0.0"

  dependencies {
    rel_stdlib: ">=0.1.0"
    rel_geo:    "~0.3.0"   -- 地理演算モジュール
    rel_ml:     "^0.2.0"   -- ML拡張モジュール
  }
}
```

### 11.5 標準ライブラリモジュール

```rel
-- 標準モジュール一覧
import std.math      { abs, ceil, floor, round, sqrt, pow }
import std.string    { upper, lower, trim, pad, split, regex }
import std.datetime  { now, today, parse_date, format_date }
import std.crypto    { hash_sha256, hmac, uuid_v7 }
import std.geo       { distance, within_radius, st_contains }
import std.json      { parse, stringify, path, merge }
import std.array     { map, filter, reduce, zip, flatten }
import std.stats     { percentile, correlation, regression }
```

---

## 12. 実行モデル

### 12.1 コンパイルパイプライン概要

```
┌─────────────────────────────────────────────────────────────┐
│                   Rel Compilation Pipeline                  │
│                                                             │
│  .rel source                                                │
│       │                                                     │
│       ▼                                                     │
│  ┌──────────┐   Tokenize / Lex                              │
│  │  Lexer   │   → Token Stream                              │
│  └────┬─────┘                                               │
│       │                                                     │
│       ▼                                                     │
│  ┌──────────┐   Parse Token Stream                          │
│  │  Parser  │   → AST (Abstract Syntax Tree)                │
│  └────┬─────┘                                               │
│       │                                                     │
│       ▼                                                     │
│  ┌────────────┐  Resolve Names / Check Types                │
│  │  Analyzer  │  → Typed AST                                │
│  └─────┬──────┘                                             │
│        │                                                    │
│        ▼                                                    │
│  ┌───────────────┐  Lower to Logical Plan                   │
│  │ IR Generator  │  → Logical Query IR                      │
│  └──────┬────────┘                                          │
│         │                                                   │
│         ▼                                                   │
│  ┌───────────────┐  Cost-based Optimization                 │
│  │    Planner    │  → Optimized Logical Plan                 │
│  └──────┬────────┘                                          │
│         │                                                   │
│         ▼                                                   │
│  ┌───────────────┐  Generate Physical Execution Steps       │
│  │ Exec Planner  │  → Physical Execution Plan               │
│  └──────┬────────┘                                          │
│         │                                                   │
│         ▼                                                   │
│  ┌───────────────┐  Distributed Planning                    │
│  │ Dist Planner  │  → Distributed Execution Plan            │
│  └──────┬────────┘                                          │
│         │                                                   │
│         ▼                                                   │
│  ┌───────────────┐  Execute on Nodes                        │
│  │   Executor    │  → Result Stream                         │
│  └───────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
```

### 12.2 AST構造

```
クエリ: from users |> where .age > 18 |> select .name

AST:
Pipeline
├── Source
│   └── TableRef("users")
├── Operator: Where
│   └── BinaryExpr
│       ├── FieldAccess("age")
│       ├── Op: GreaterThan
│       └── Literal(Int, 18)
└── Operator: Select
    └── FieldAccess("name")
```

### 12.3 論理クエリプランの例

```
クエリ: from orders -> user |> where user.country == "JP" |> select { orders.id, user.name }

論理プラン (s式表記):
(Project
  [orders.id, user.name]
  (Filter
    (= user.country "JP")
    (HashJoin
      (inner)
      (Scan orders)
      (Scan users)
      (= orders.user_id users.id))))
```

### 12.4 オプティマイザの最適化規則

```
最適化規則の例:

1. Predicate Pushdown（述語プッシュダウン）
   BEFORE: Filter(Scan(T)) → Join → Filter(...)
   AFTER:  Join(Filter(Scan(T)), Filter(Scan(U)))
   効果: JOINする前に行を減らす

2. Column Pruning（カラム剪定）
   BEFORE: Project[name](Scan[id,name,age,email](users))
   AFTER:  Project[name](Scan[name](users))
   効果: 不要なカラムを読まない

3. Join Reordering（JOIN順序最適化）
   コストモデルに基づいて最小コストのJOIN順を選択

4. Index Utilization（インデックス利用）
   WHERE句の条件に対応するインデックスを自動選択

5. Materialized Query Reuse（マテリアライズドクエリ再利用）
   既存のマテリアライズドビューを自動的に再利用
```

### 12.5 物理実行プランの例

```
物理実行プラン:

PhysicalPlan {
  type: DistributedHashJoin,
  build_side: {
    type: IndexScan,
    table: "users",
    index: "idx_users_country",
    predicate: country == "JP",
    projection: [id, name]
  },
  probe_side: {
    type: SequentialScan,
    table: "orders",
    projection: [id, user_id]
  },
  join_keys: [(orders.user_id, users.id)],
  projection: [orders.id, users.name],
  parallelism: 8
}
```

### 12.6 実行エンジンのアーキテクチャ

```
┌──────────────────────────────────────────────────────┐
│                  Execution Engine                    │
│                                                      │
│  ┌─────────────────────────────────────────────────┐ │
│  │  Vectorized Execution Engine (SIMD対応)         │ │
│  │                                                 │ │
│  │  Batch[0..1023]  →  Batch[0..1023]  → ...      │ │
│  │  （行単位ではなくバッチ単位で処理）              │ │
│  └─────────────────────────────────────────────────┘ │
│                                                      │
│  ┌─────────────────────────────────────────────────┐ │
│  │  Columnar Storage Interface                     │ │
│  │  Apache Arrow互換バッファ                       │ │
│  └─────────────────────────────────────────────────┘ │
│                                                      │
│  ┌─────────────────────────────────────────────────┐ │
│  │  Async Streaming Execution                      │ │
│  │  Backpressure対応                               │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

---

## 13. 分散クエリ

### 13.1 設計原則

```
Relの分散処理は「透過的分散」を原則とする。

開発者はシングルノードと同じ構文でクエリを書く。
エンジンが自動的に分散実行プランを生成する。

ただし、パフォーマンスチューニング時は
明示的な分散制御も可能。
```

### 13.2 シャーディング定義

```rel
-- エンティティのシャーディング設定
entity Order {
  id:         UUID      @primary
  user_id:    UUID
  amount:     Decimal
  created_at: Timestamp
}

-- シャーディングポリシー
@shard(
  strategy: hash,          -- hash | range | list
  key:      .user_id,      -- シャードキー
  count:    256,           -- シャード数
  replicas: 3              -- レプリカ数
)

-- 範囲シャーディング（時系列データに最適）
@shard(
  strategy: range,
  key:      .created_at,
  ranges: [
    { to: "2024-01-01", node_group: "cold_storage" },
    { to: "2025-01-01", node_group: "warm_storage" },
    { from: "2025-01-01", node_group: "hot_storage" }
  ]
)
```

### 13.3 分散クエリの実行フロー

```
クエリ: from orders |> where .user_id == "abc" |> sum(.amount)

分散実行フロー:

┌─────────────────────────────────────────────────────────────┐
│  Coordinator Node                                           │
│                                                             │
│  1. クエリ受信・解析                                        │
│  2. シャードキー (.user_id == "abc") を検出                 │
│  3. 対象シャード特定 → Shard #42 のみ                       │
│  4. Shard #42 にクエリ送信                                  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Shard #42                                          │   │
│  │  1. ローカルストレージをスキャン                     │   │
│  │  2. フィルタ適用                                    │   │
│  │  3. 部分集計 (partial_sum = 12500.00)               │   │
│  │  4. 結果をCoordinatorに返却                         │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  5. 部分集計を最終集計 → 12500.00                           │
│  6. クライアントに返却                                      │
└─────────────────────────────────────────────────────────────┘
```

### 13.4 分散集計（Scatter-Gather）

```
クエリ: from orders |> group by .country |> sum(.amount)

Scatter-Gather実行:

Coordinator
    │
    ├──→ Shard #1  →  { JP: 5000, US: 3000 }
    ├──→ Shard #2  →  { JP: 7000, UK: 2000 }
    ├──→ Shard #3  →  { US: 4000, JP: 1000 }
    └──→ Shard #N  →  { ... }

Merge Phase (Coordinator):
  JP: 5000 + 7000 + 1000 = 13000
  US: 3000 + 4000         = 7000
  UK: 2000                = 2000
```

### 13.5 ノード間通信プロトコル

```
┌───────────────────────────────────────────────────────────┐
│  Rel Wire Protocol (RWP) v1                               │
│                                                           │
│  Transport:   gRPC / HTTP2                                │
│  Encoding:    Protocol Buffers (binary)                   │
│  Data Format: Apache Arrow IPC (columnar)                 │
│  Compression: LZ4 / Zstd (自動選択)                      │
│                                                           │
│  Message Types:                                           │
│  ┌──────────────────┬─────────────────────────────────┐  │
│  │  QueryRequest    │ クエリの送信                     │  │
│  │  PartialResult   │ 部分結果のストリーミング返却      │  │
│  │  FinalResult     │ 完了通知 + 最終メタデータ        │  │
│  │  CancelRequest   │ クエリのキャンセル               │  │
│  │  HeartBeat       │ ノードの生存確認                 │  │
│  └──────────────────┴─────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

### 13.6 ストリーム処理

```rel
-- リアルタイムストリーム処理
stream order_events from kafka("orders-topic") {
  format: json,
  schema: {
    order_id:   UUID,
    user_id:    UUID,
    amount:     Decimal,
    event_type: "created" | "updated" | "cancelled",
    ts:         Timestamp
  }
}

-- ストリームクエリ（タンブリングウィンドウ）
from order_events
|> where .event_type == "created"
|> window tumbling(size: 1m)
|> group by .country
|> select {
     window_start: window.start,
     country:      .country,
     order_count:  count(),
     revenue:      sum(.amount)
   }
|> sink kafka("revenue-metrics-topic")

-- スライディングウィンドウ
from order_events
|> window sliding(size: 5m, step: 1m)
|> group by .user_id
|> having sum(.amount) > 10000      -- 5分間で1万以上
|> select { .user_id, total: sum(.amount) }
|> sink alert_service("high_value_users")
```

### 13.7 分散トランザクション

```rel
-- 分散トランザクション（2フェーズコミット）
transaction place_order {
  isolation: serializable,     -- read_committed | repeatable_read | serializable
  timeout:   30s,
  retry:     3

  steps {
    -- ステップ1: 在庫確認・確保
    let product = from products
                  |> where .id == $product_id
                  |> lock(for: update)   -- 悲観的ロック
                  |> single!()           -- 1件のみ、なければエラー

    guard product.stock >= $quantity
      else Error.InsufficientStock

    update products
    |> where .id == $product_id
    |> set { stock: product.stock - $quantity }

    -- ステップ2: 注文作成
    let order_id = insert Order {
      user_id:    $user_id,
      product_id: $product_id,
      quantity:   $quantity,
      amount:     product.price * $quantity,
      status:     OrderStatus.Pending
    }

    -- ステップ3: 支払処理（外部サービス呼び出し）
    let payment = await payment_service.charge($user_id, order.amount)

    -- ステップ4: 注文確定
    update orders
    |> where .id == order_id
    |> set {
         status:      OrderStatus.Processing,
         payment_id:  payment.id
       }

    return { order_id, payment_id: payment.id }
  }
}
```

---

## 14. SQLとの比較

### 14.1 機能比較表

| 機能 | SQL | Rel |
|------|-----|-----|
| **構文の読み順** | SELECT→FROM→WHERE（実行順と逆） | FROM→WHERE→SELECT（実行順と一致） |
| **JOIN記述** | `JOIN ... ON ...` を毎回記述 | `->` 演算子でリレーション探索 |
| **NULL処理** | 三値論理 (T/F/NULL) | Option型 (Some/None) |
| **型システム** | 弱い型・暗黙キャスト | 強い静的型・型推論 |
| **サブクエリ** | ネスト地獄になりやすい | パイプラインで平坦に記述 |
| **クエリ再利用** | VIEW / Stored Procedure | first-class `query` |
| **モジュール** | スキーマ/DB単位のみ | module / namespace / import |
| **型定義** | 限定的 | カスタム型・enum・代数的データ型 |
| **分散処理** | 方言依存・手動設定 | ネイティブ組み込み・透過的 |
| **ストリーム処理** | 非対応（拡張必要） | ネイティブサポート |
| **型安全なJSON** | 限定的 | Json<T> で型付きJSON |
| **再帰クエリ** | WITH RECURSIVE（複雑） | `~>` 演算子で直感的 |
| **トランザクション** | BEGIN/COMMIT/ROLLBACK | transaction ブロック |
| **マイグレーション** | ALTER TABLE（言語外） | migration ブロック（言語内） |
| **コンパイル時エラー** | ほぼなし（実行時エラー） | 型エラー・リレーションエラーを事前検出 |

### 14.2 クエリ書き方の比較

#### シンプルなSELECT
```sql
-- SQL
SELECT id, name, email FROM users WHERE age > 18 ORDER BY name LIMIT 10;
```
```rel
-- Rel
from users
|> where .age > 18
|> order by .name
|> take 10
|> select { .id, .name, .email }
```

#### 複数テーブルのJOIN
```sql
-- SQL
SELECT o.id, u.name, p.name AS product
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE u.country = 'JP'
ORDER BY o.created_at DESC;
```
```rel
-- Rel
from orders -> user -> order_items -> product
|> where user.country == "JP"
|> order by orders.created_at desc
|> select {
     order_id:    orders.id,
     user_name:   user.name,
     product_name: product.name
   }
```

#### GROUP BY + HAVING
```sql
-- SQL
SELECT country, COUNT(*) as cnt, AVG(age) as avg_age
FROM users
WHERE status = 'active'
GROUP BY country
HAVING COUNT(*) > 100
ORDER BY cnt DESC;
```
```rel
-- Rel
from users
|> where .status == Status.Active
|> group by .country
|> having count() > 100
|> order by count() desc
|> select {
     country: .country,
     cnt:     count(),
     avg_age: avg(.age)
   }
```

#### 再帰クエリ
```sql
-- SQL（WITH RECURSIVEは複雑）
WITH RECURSIVE subordinates AS (
  SELECT id, name, manager_id, 0 AS depth
  FROM employees
  WHERE id = $root_id
  UNION ALL
  SELECT e.id, e.name, e.manager_id, s.depth + 1
  FROM employees e
  JOIN subordinates s ON e.manager_id = s.id
  WHERE s.depth < 10
)
SELECT * FROM subordinates;
```
```rel
-- Rel（直感的）
from employees
|> where .id == $root_id
|> follow ~> reports(depth: 0..10)
|> select { .id, .name, .depth }
```

#### NULL処理の違い
```sql
-- SQL（三値論理で混乱しやすい）
SELECT * FROM users WHERE phone IS NOT NULL;
SELECT COALESCE(phone, 'N/A') FROM users;
-- NULL = NULL は FALSE (!)
-- NULL != NULL は NULL (!)
```
```rel
-- Rel（Option型で明確）
from users
|> where .phone != None         -- Noneチェック
|> select {
     .name,
     phone: .phone ?? "N/A"     -- デフォルト値
   }
-- None == None は true（直感的）
```

### 14.3 パフォーマンス特性

| 操作 | SQL (単一ノード) | Rel (分散) |
|------|----------------|-----------|
| シンプルなSELECT | ベースライン | 同等〜高速（インデックス最適化） |
| 複雑なJOIN（10テーブル） | 遅くなりがち | 並列実行で高速 |
| 大規模集計 (10億行) | 数時間 | 分単位（分散集計） |
| リアルタイムストリーム | 非対応 | ミリ秒レイテンシ |
| 再帰クエリ（深い階層） | 限界あり | 深さ制限付きで安全に実行 |

---

## 付録A: 完全なクエリ例

### A.1 ECサイト — 月次売上レポート

```rel
#!rel 0.1
import analytics.*
import std.datetime { format_date, date_trunc }

query monthly_sales_report(year: Int) {
  from orders
  |> where date(.created_at).year == year
  |> where .status != OrderStatus.Cancelled
  |> navigate .order_items -> product
  |> group by
       month:   date_trunc("month", orders.created_at),
       category: product.category
  |> select {
       month:          format_date(.month, "YYYY-MM"),
       category:       .category,
       total_revenue:  sum(order_items.quantity * product.price),
       total_orders:   count(distinct orders.id),
       total_items:    sum(order_items.quantity),
       avg_order_val:  sum(order_items.quantity * product.price) / count(distinct orders.id)
     }
  |> order by .month desc, total_revenue desc
}

-- 実行
from monthly_sales_report(2026)
|> where .total_revenue > 1000000
```

### A.2 SNS — フォロイーのタイムライン

```rel
query timeline(user_id: UUID, cursor: String?, limit: Int = 50) {
  from users
  |> where .id == user_id
  |> navigate .following -> posts
  |> where following.is_active == true
  |> navigate posts -?> media       -- オプショナル
  |> navigate posts -> author
  |> order by posts.created_at desc, posts.id desc
  |> cursor_paginate { after: cursor, per_page: limit }
  |> select {
       edges: {
         id:          posts.id,
         body:        posts.body,
         created_at:  posts.created_at,
         author: {
           id:     author.id,
           name:   author.name,
           avatar: author.avatar_url
         },
         media: media?.map(m => { url: m.url, type: m.media_type })
       },
       page_info: { has_next_page, end_cursor }
     }
}
```

### A.3 不正検知 — リアルタイムストリーム

```rel
import std.stats { z_score }
import std.ml    { anomaly_detect }

-- ユーザーの通常取引パターンを事前計算
@materialized(refresh: every 6h)
query user_transaction_baseline {
  from transactions
  |> where .created_at >= now() - 90.days
  |> group by .user_id
  |> select {
       user_id:    .user_id,
       avg_amount: avg(.amount),
       std_amount: stddev(.amount),
       avg_hourly: count() / 90.0 / 24.0
     }
}

-- リアルタイムストリームで不正検知
stream tx_events from kafka("transactions") {
  format: json,
  schema: Transaction
}

from tx_events
|> navigate .user_id -> user_transaction_baseline as baseline
|> let z = z_score(.amount, baseline.avg_amount, baseline.std_amount)
|> where z > 3.0 or .amount > 500000
|> select {
     transaction_id: .id,
     user_id:        .user_id,
     amount:         .amount,
     z_score:        z,
     risk_level:     if z > 5.0 then "critical" else "high"
   }
|> sink alert_service("fraud-alerts")
```

---

## 付録B: 文法仕様（EBNF）

```ebnf
program         ::= statement*
statement       ::= query_def | entity_def | relation_def | import_stmt | module_def

query_def       ::= annotation* "query" IDENT params? "{" pipeline "}"
params          ::= "(" param ("," param)* ")"
param           ::= IDENT ":" type ("=" expr)?

pipeline        ::= source ("|>" operator)*
source          ::= "from" source_expr
source_expr     ::= IDENT ("->" IDENT)*
operator        ::= where_op | select_op | group_op | order_op | take_op | navigate_op | window_op | having_op

where_op        ::= "where" expr
select_op       ::= "select" projection
group_op        ::= "group" "by" expr ("," expr)*
having_op       ::= "having" expr
order_op        ::= "order" "by" order_term ("," order_term)*
order_term      ::= expr ("asc" | "desc")?
take_op         ::= "take" expr ("skip" expr)?
navigate_op     ::= "navigate" nav_expr

nav_expr        ::= "." IDENT nav_type?
nav_type        ::= "->" | "-?>" | "<-" | "<->" | "~>"

projection      ::= expr | "{" proj_field ("," proj_field)* "}"
proj_field      ::= IDENT ":" expr | "." IDENT | spread_expr
spread_expr     ::= "..." expr

entity_def      ::= annotation* "entity" IDENT "{" field_def* "}"
field_def       ::= IDENT ":" type annotation* NEWLINE

relation_def    ::= "relation" IDENT "." IDENT nav_type IDENT type? ("via" IDENT)?

type            ::= base_type | option_type | list_type | map_type
base_type       ::= IDENT ("(" INT ("," INT)? ")")?
option_type     ::= type "?"
list_type       ::= "List" "<" type ">"
map_type        ::= "Map" "<" type "," type ">"

expr            ::= literal | field_access | binary_expr | unary_expr | call_expr | match_expr
field_access    ::= "." IDENT ("." IDENT)*
binary_expr     ::= expr op expr
op              ::= "==" | "!=" | "<" | "<=" | ">" | ">=" | "and" | "or" | "+" | "-" | "*" | "/" | "??" | "++"
call_expr       ::= IDENT "(" (expr ("," expr)*)? ")"
match_expr      ::= "match" expr "{" match_arm+ "}"
match_arm       ::= pattern "=>" expr ","

annotation      ::= "@" IDENT ("(" annotation_args ")")?
import_stmt     ::= "import" module_path ("{" IDENT ("," IDENT)* "}")?
module_def      ::= "module" IDENT "{" statement* "}"
```

---

## 付録C: エラーメッセージ設計

Relのコンパイラは人間が読めるエラーメッセージを出力する。

```
Error[E001]: Type mismatch
  --> query.rel:5:14
  |
5 |  |> where .age > "adult"
  |                  ^^^^^^^
  |                  expected Int, found String
  |
  hint: .age is of type Int. To compare with a string,
        try parsing: .age > Int.parse("18")
        or use a literal: .age > 18

Error[E042]: Unknown relation
  --> query.rel:3:14
  |
3 |  from orders -> payment
  |                 ^^^^^^^
  |                 relation 'payment' not found on entity 'Order'
  |
  hint: Did you mean 'Order.user'?
        Available relations: user, order_items, coupon?

Error[E103]: Non-nullable field accessed as nullable
  --> query.rel:8:12
  |
8 |  |> select .email ?? "no-email"
  |             ^^^^^
  |             'email' is required (non-nullable), '??' is unnecessary
  |
  hint: Remove the '??' operator, or change the schema to: email: String?
```

---

*Rel Language Specification v0.1.0-draft*
*Copyright 2026 — Rel Language Design Group*
