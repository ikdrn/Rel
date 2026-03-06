# Rel Type System — 完全仕様書

## 概要

RelはHindley-Milner型推論をベースに、
データベース向けの拡張を加えた静的型システムを採用する。

```
型システムの特徴:
1. 完全な静的型チェック（コンパイル時にほぼ全ての型エラーを検出）
2. 型推論（ほとんどの箇所で型アノテーション不要）
3. Null安全（Option型による明示的なNull処理）
4. 代数的データ型（enum・union型）
5. 構造的部分型（duck typing的な型互換）
6. 型レベルのデータベース制約
```

---

## 1. 型の分類

```
┌────────────────────────────────────────────────────────────┐
│                    Rel Type Hierarchy                      │
│                                                            │
│  Scalar Types                                              │
│  ├── Numeric:  Int, Int8..Int64, UInt, Float, Decimal      │
│  ├── Text:     String, String(n), Text, Char(n)            │
│  ├── Boolean:  Bool                                        │
│  ├── Temporal: Date, Time, Timestamp, TimestampTZ,         │
│  │             Duration, Interval                          │
│  ├── Binary:   Bytes, Bytes(n)                             │
│  ├── ID:       UUID, ULID, Snowflake                       │
│  ├── Network:  IpAddr, MacAddr, Url, Email                 │
│  └── Geo:      Point, Polygon, GeoHash                     │
│                                                            │
│  Container Types                                           │
│  ├── Option<T>     (= T?)                                  │
│  ├── List<T>                                               │
│  ├── Set<T>                                                │
│  ├── Map<K, V>                                             │
│  ├── (T1, T2, ...)  Tuple                                  │
│  └── Stream<T>                                             │
│                                                            │
│  Structured Types                                          │
│  ├── Record:  { field: Type, ... }                         │
│  ├── Enum:    enum Foo { A | B | C }                       │
│  ├── Union:   Type1 | Type2 | ...                          │
│  └── ADT:     type Foo = | A | B(x: T) | C(x: T, y: T)    │
│                                                            │
│  Special Types                                             │
│  ├── Json            (任意のJSONデータ)                    │
│  ├── Json<T>         (型付きJSON)                          │
│  ├── Query<T>        (クエリ型)                            │
│  └── Never           (到達不能)                            │
└────────────────────────────────────────────────────────────┘
```

---

## 2. スカラー型の詳細

### 2.1 数値型

```rel
-- 整数型
Int8    : -128 .. 127
Int16   : -32,768 .. 32,767
Int32   : -2,147,483,648 .. 2,147,483,647
Int64   : -9,223,372,036,854,775,808 .. 9,223,372,036,854,775,807
Int     = Int64  (デフォルト整数型)
UInt    : 0 .. 18,446,744,073,709,551,615
UInt8   : 0 .. 255

-- 浮動小数点型
Float32 : IEEE 754 single precision
Float64 : IEEE 754 double precision
Float   = Float64  (デフォルト浮動小数点型)

-- 固定精度小数型（金融計算）
Decimal(p, s)  : precision=p, scale=s
  p: 全体の桁数 (1-38)
  s: 小数点以下の桁数 (0-p)
  例: Decimal(10, 2) → 最大 99999999.99

-- 数値演算の型規則
Int + Int     → Int
Int + Float   → Float (暗黙的に Int が Float に昇格)
Float + Float → Float
Int + Decimal → Decimal
Decimal + Decimal → Decimal (スケールは大きい方)
Int * Int     → Int  (オーバーフロー時はエラー)
Int / Int     → Float (整数除算は Float を返す)
Int // Int    → Int  (整数除算演算子)
Int % Int     → Int  (剰余)
```

### 2.2 文字列型

```rel
String       : 可変長UTF-8, 最大 ~1GB
String(n)    : 最大n文字, n = 1..65535
Text         = String  (意味的エイリアス, 長文向け)
Char(n)      : 固定長n文字 (パディングあり)

-- バリデーション付き型エイリアス
Email   = String @validate(is_email)
Url     = String @validate(is_url)
Phone   = String @validate(is_phone)
Slug    = String @validate(matches /^[a-z0-9-]+$/)

-- 文字列連結
"hello" ++ " " ++ "world"  →  "hello world"
"count: " ++ 42.to_string()  →  "count: 42"
```

### 2.3 時間型

```rel
Date        : "YYYY-MM-DD"  (タイムゾーンなし)
Time        : "HH:MM:SS.mmm"  (タイムゾーンなし)
Timestamp   : UTC日時 (エポックからのナノ秒)
TimestampTZ : タイムゾーン付き日時

Duration    : 秒単位の時間長 (可算)
Interval    : カレンダー単位の期間 (月・年を含む)

-- 時間リテラル
Date("2026-03-06")
Time("14:30:00")
Timestamp("2026-03-06T14:30:00Z")

-- Duration リテラル
30.ms        -- 30ミリ秒
5.s          -- 5秒
10.m         -- 10分
2.h          -- 2時間
7.days       -- 7日
1.week       -- 1週
3.months     -- 3ヶ月
1.year       -- 1年

-- 時間演算
Timestamp + Duration    → Timestamp
Timestamp - Timestamp   → Duration
Timestamp - Duration    → Timestamp
Date + Interval         → Date

now() - 30.days         -- 30日前のTimestamp
today() - 1.week        -- 1週間前のDate
```

---

## 3. コンテナ型の詳細

### 3.1 Option型（Nullable の再設計）

```
SQLのNULLを廃止し、Option型で代替する。

SQL NULL の問題:
  NULL = NULL   → NULL (!)   直感に反する
  NULL != NULL  → NULL (!)   直感に反する
  1 + NULL      → NULL (!)   伝播するNULL
  NOT NULL      → NULL (!)   三値論理

Rel Option型:
  None == None  → true  (直感的)
  None != None  → false (直感的)
  1 + None      → コンパイルエラー (型安全)
```

```rel
-- Option型の宣言
name:  String?         -- Option<String> の短縮記法
value: Option<Int>     -- 同上（より明示的）

-- Option型の操作
user.phone                    -- Option<String>型

-- デフォルト値 (?? 演算子 = null coalescing)
user.phone ?? "N/A"           -- String型

-- オプショナルチェーン (?. 演算子)
user.profile?.avatar_url      -- Option<Url>型

-- 強制アンラップ (! 演算子, Noneなら実行時エラー)
user.phone!                   -- String型 (Noneならパニック)

-- 期待値付きアンラップ
user.phone.expect("phone is required")   -- エラーメッセージ付き

-- マッピング
user.phone.map(|p| p.to_upper())       -- Option<String>型

-- フラットマッピング
user.profile.flat_map(|p| p.avatar_url)  -- Option<Url>型

-- パターンマッチング
match user.phone {
  Some(p) => send_sms(p),
  None    => send_email(user.email)
}

-- if-let (パターンマッチの省略形)
if let Some(phone) = user.phone {
  send_sms(phone)
}

-- Option同士の演算
Some(5) + Some(3)   -- Some(8)
Some(5) + None      -- None
None    + Some(3)   -- None
None    + None      -- None
```

### 3.2 List型

```rel
-- List<T> 宣言
tags:   List<String>
scores: List<Int>
items:  List<{id: UUID, qty: Int}>

-- Listリテラル
["a", "b", "c"]        : List<String>
[1, 2, 3]              : List<Int>
[]                     : List<Never> (型推論で解決)

-- List操作
xs.length()            -- Int
xs.map(|x| x * 2)     -- List<Int>
xs.filter(|x| x > 0)  -- List<T>
xs.reduce(0, |acc, x| acc + x)  -- T
xs.any(|x| x > 10)    -- Bool
xs.all(|x| x > 0)     -- Bool
xs.first()             -- Option<T>
xs.last()              -- Option<T>
xs.nth(i)              -- Option<T>
xs.contains(value)     -- Bool
xs.distinct()          -- List<T>
xs.sort()              -- List<T> (Orderable要求)
xs.sort_by(|x| x.field)  -- List<T>
xs.flatten()           -- List<U> (List<List<U>>から)
xs.zip(ys)             -- List<(T, U)>
xs.enumerate()         -- List<(Int, T)>
xs.sum()               -- T (Numeric要求)
xs.avg()               -- Float (Numeric要求)
xs[0]                  -- T (範囲外はエラー)
xs[0..3]               -- List<T> (スライス)
xs ++ ys               -- List<T> (結合)
```

### 3.3 Map型

```rel
-- Map<K, V> 宣言
attributes: Map<String, Json>
counts:     Map<String, Int>

-- Mapリテラル
Map { "a": 1, "b": 2 }   : Map<String, Int>
Map {}                    : Map<Never, Never>

-- Map操作
m.get("key")              -- Option<V>
m.get_or("key", default)  -- V
m["key"]                  -- V (存在しない場合エラー)
m.contains_key("key")     -- Bool
m.keys()                  -- List<K>
m.values()                -- List<V>
m.entries()               -- List<(K, V)>
m.map_values(|v| v * 2)   -- Map<K, V2>
m.filter(|k, v| v > 0)    -- Map<K, V>
m.merge(m2)               -- Map<K, V> (m2が優先)
m.size()                  -- Int
```

---

## 4. 構造型

### 4.1 レコード型（匿名構造体）

```rel
-- インライン型定義
type Point = { x: Float, y: Float }
type Color = { r: Int, g: Int, b: Int }

-- ネストしたレコード型
type Address = {
  street:  String,
  city:    String,
  country: String,
  zip:     String?,
  geo:     { lat: Float, lon: Float }?
}

-- 構造的部分型（structural subtyping）
-- { id: UUID, name: String, email: String } は
-- { id: UUID, name: String } を要求する場所に使える
-- （余分なフィールドは無視される）

-- スプレッド演算子
type UserWithRole = {
  ...User,          -- Userの全フィールドを展開
  role: String
}
```

### 4.2 Enum型

```rel
-- シンプルなEnum
enum Color {
  Red   = "red"
  Green = "green"
  Blue  = "blue"
}

-- Int値のEnum
enum Priority {
  Low      = 1
  Medium   = 2
  High     = 3
  Critical = 4
}

-- Enum操作
let c = Color.Red
c.to_string()    -- "red"
c.to_int()       -- (Intバックドエンドのみ)
Color.from_string("red")  -- Option<Color>

-- Enum比較
Color.Red == Color.Red    -- true
Color.Red != Color.Blue   -- true
Priority.High > Priority.Low  -- true (Orderable)

-- Enumのパターンマッチ
match order.status {
  OrderStatus.Pending    => "受付中",
  OrderStatus.Processing => "処理中",
  OrderStatus.Shipped    => "発送済",
  _                      => "その他"
}
```

### 4.3 代数的データ型（ADT）

```rel
-- 直和型（Tagged Union）
type Shape =
  | Circle    { radius: Float }
  | Rectangle { width: Float, height: Float }
  | Triangle  { base: Float, height: Float }

-- 面積計算（パターンマッチ）
let area = match shape {
  Circle { radius }        => 3.14159 * radius * radius,
  Rectangle { width, height } => width * height,
  Triangle { base, height }   => 0.5 * base * height
}

-- Option型もADT
type Option<T> =
  | Some(T)
  | None

-- Result型（エラーハンドリング）
type Result<T, E> =
  | Ok(T)
  | Err(E)

-- 注文ステータスの詳細なADT
type OrderStatusDetail =
  | Pending
  | Processing { started_at: Timestamp, worker_id: UUID? }
  | Shipped    { tracking: String, carrier: String, eta: Date? }
  | Delivered  { at: Timestamp, signature: String? }
  | Cancelled  { reason: String, refund: Decimal? }

-- ADTの構築
let status = OrderStatusDetail.Shipped {
  tracking: "1234-5678",
  carrier:  "YamatoTransport",
  eta:      Date("2026-03-10")
}

-- ADTのパターンマッチ
match order.status_detail {
  Pending                           => "受付中",
  Processing { started_at, _ }      => "処理中（" ++ started_at.to_string() ++ "）",
  Shipped { tracking, carrier, _ }  => carrier ++ ": " ++ tracking,
  Delivered { at, _ }               => "配達完了 " ++ at.to_string(),
  Cancelled { reason, refund }      => "キャンセル: " ++ reason
}
```

---

## 5. 型推論

```rel
-- 型アノテーションなしで正しく型が推論される

-- 単純なリテラル
let x = 42          -- x: Int
let y = 3.14        -- y: Float
let s = "hello"     -- s: String
let b = true        -- b: Bool
let n = None        -- n: Option<Never> (使用文脈から解決)

-- クエリ結果の型推論
let users = from users |> select { .id, .name }
-- users: Stream<{ id: UUID, name: String }>

let count = from users |> count()
-- count: Int

-- 関数の型推論
let double = |x| x * 2
-- double: Int -> Int (使用文脈から)

-- Option型の伝播
let email = user.email                -- Option<String>
let upper = user.email?.to_upper()    -- Option<String>
let value = user.email ?? "default"   -- String

-- 型推論が失敗する例（アノテーション必要）
let items = []    -- エラー: List<?>の型引数が不明
let items: List<String> = []   -- OK
```

---

## 6. 型の制約（Constraints）

```rel
-- 型クラス相当の制約
constraint Numeric<T> {
  -- T が +, -, *, / をサポートすることを保証
  fn add(a: T, b: T) -> T
  fn zero() -> T
}

constraint Orderable<T> {
  -- T が比較可能であることを保証
  fn compare(a: T, b: T) -> Int  -- -1, 0, 1
}

constraint Hashable<T> {
  -- T がMapのキーになれることを保証
  fn hash(a: T) -> UInt64
}

-- 制約付き型パラメータ
query sum_field<T: Numeric>(source: Query<{amount: T}>) -> T {
  from source
  |> aggregate sum(.amount)
}

-- 組み込み制約
-- Aggregatable: count, sum, avg などに使える型
-- Comparable:   = != < > <= >= が使える型
-- Nullable:     ? 記法が使える型（全ての型）
```

---

## 7. Json型

```rel
-- 非構造化JSON（スキーマ不定）
metadata:  Json

-- 型付きJSON（スキーマ検証あり）
settings:  Json<{
  theme:     "light" | "dark",
  language:  String,
  timezone:  String,
  features:  Map<String, Bool>
}>

-- JSONパス操作
user.metadata["key"]                   -- Json
user.metadata.path("$.nested.value")   -- Option<Json>
user.metadata.as_string()              -- Option<String>
user.metadata.as_int()                 -- Option<Int>
user.metadata.as_list()                -- Option<List<Json>>

-- JSON構築
let obj = Json {
  name:  "Alice",
  age:   30,
  tags:  ["admin", "user"]
}

-- JSON同士のマージ
user.metadata.merge({ "new_key": "value" })
```

---

## 8. クエリ型（Query<T>）

```rel
-- クエリ自体を型として扱う
type Query<T> = <...>  -- 組み込み型

-- クエリを変数に代入
let q: Query<User> = from users |> where .age > 18

-- クエリを関数引数に渡す
query with_limit<T>(q: Query<T>, n: Int) -> Query<T> {
  from q |> take n
}

-- クエリの遅延評価
let base = from products |> where .is_active == true
let cheap = from base |> where .price < 1000
let expensive = from base |> where .price > 10000
-- base, cheap, expensive はまだ実行されない

-- 実行（具現化）
let results = cheap |> execute()    -- List<Product>
let stream  = cheap |> stream()     -- Stream<Product>
let count   = cheap |> count()      -- Int
let first   = cheap |> first()      -- Option<Product>
```

---

## 9. 型エラーの例

```
-- エラー1: 型の不一致
from users |> where .age > "adult"
-- Error[E001]: Type mismatch: expected Int, found String

-- エラー2: Optionalの扱い漏れ
from users |> select .phone.to_upper()
-- Error[E002]: .phone is Option<String>, use .phone?.to_upper() or .phone!.to_upper()

-- エラー3: 存在しないフィールド
from users |> select .username
-- Error[E003]: Field 'username' not found on entity 'User'
--   hint: Did you mean 'name'?

-- エラー4: 不明なリレーション
from orders -> payment_method
-- Error[E042]: Relation 'payment_method' not found on entity 'Order'
--   Available: user, items, payment, coupon?

-- エラー5: 集計なしのselect（GROUP BY後）
from orders |> group by .country |> select .id
-- Error[E050]: 'id' is not in GROUP BY and not an aggregate function
--   hint: Use count(), sum(.id), first(.id), or add .id to group by

-- エラー6: NULL安全でない比較
from users |> where .phone == None    -- OK
from users |> where .phone == null    -- Error[E060]: Use 'None' instead of 'null'

-- エラー7: 型が確定できない
let x = if condition then 1 else "hello"
-- Error[E070]: Arms of 'if' have incompatible types: Int vs String
```

---

## 10. 型システムの保証

```
Relの型システムが保証すること:

✅ NULLポインタ参照が起きない（Option型による強制）
✅ 型の暗黙変換が起きない（明示的キャストのみ）
✅ 存在しないフィールドへのアクセスが起きない
✅ 存在しないリレーションの探索が起きない
✅ GROUP BY後の非集計フィールドへのアクセスが起きない
✅ 型エラーのほとんどがコンパイル時に検出される

Relが実行時まで検出できないこと:
⚠  外部キー整合性の違反（スキーマで定義された場合はDB側で保証）
⚠  除算のゼロ割り（実行時エラー）
⚠  ! アンラップ時のNoneパニック
⚠  @check 制約の違反（実行時にチェック）
```
