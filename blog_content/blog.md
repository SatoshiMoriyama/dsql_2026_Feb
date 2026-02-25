# Amazon Aurora DSQL 2026年2月のアップデートまとめ — IDENTITY列・SEQUENCEとNUMERIC型インデックスを試してみる

## はじめに

こんにちは！

アプリケーションサービス本部ディベロップメントサービス１課の森山です。

2026年2月、Amazon Aurora DSQL に複数のアップデートがありました。

今回はその中から特に気になった2つをピックアップして、実際に統合クエリエディタから試してみます。

### 2026年2月のアップデート一覧

2月に発表されたアップデートは以下の5つです。

| # | アップデート | 日付 |
| --- | --- | --- |
| 1 | [NUMERIC型のインデックスサポート](https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-aurora-dsql-indexes-numeric-data-type/) | 2026/02/03 |
| 2 | [リージョン拡大（Melbourne, Sydney, Canada Central, Calgary）](https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-aurora-dsql-additional-aws-regions/) | 2026/02/11 |
| 3 | [IDENTITY列とSEQUENCEオブジェクトのサポート](https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-aurora-dsql-adds-identity-columns-sequence/) | 2026/02/13 |
| 4 | [Kiro powers・AIエージェントスキルとの統合](https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-aurora-dsql-integrates-with-kiro-powers-and-agent-skills) | 2026/02/18 |
| 5 | [Go, Python, Node.js 向け新コネクタのリリース](https://aws.amazon.com/about-aws/whats-new/2026/02/aurora-dsql-launches-go-python-nodejs-connectors) | 2026/02/19 |

今回は 1 と 3 を実際に動かしながら確認していきます。

### この記事で学べること

- Aurora DSQL における IDENTITY 列と SEQUENCE オブジェクトの使い方
- NUMERIC 型カラムへのインデックス作成

### 前提知識・条件

- DSQL の概要のご説明は割愛させていただきます
- PostgreSQL の基本的な SQL 構文の知識

## 1. IDENTITY列とSEQUENCEオブジェクトのサポート

まずはこのアップデートについて紹介します。

[https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-aurora-dsql-adds-identity-columns-sequence/:embed:cite]

これまで Aurora DSQL でオートインクリメントな整数 ID を使いたい場合、アプリケーション側で ID 生成ロジックを実装する必要がありました。

### やってみた

#### クラスターの作成

まずは Aurora DSQL のクラスターを作成します。

マネジメントコンソールから「Aurora DSQL」を開き、「クラスターを作成」からクラスターを作成します。

また、今回は動作確認のため、シングルリージョンでの作成とします。

![alt text](<CleanShot 2026-02-26 at 04.36.12@2x.png>)

![alt text](<CleanShot 2026-02-26 at 04.39.02@2x.png>)

なお、2025年12月のアップデートにより、クラスターの作成が数分から数秒に短縮されました。

実際に試してみると、本当にあっという間に作成が完了します。（3秒くらいです。）

[https://aws.amazon.com/about-aws/whats-new/2025/12/amazon-aurora-dsql-cluster-creation-in-seconds:embed:cite]

#### 統合クエリエディタで接続

クラスターが作成できたら、そのまま統合クエリエディタで SQL を実行していきます。

![alt text](<CleanShot 2026-02-26 at 04.41.49@2x.png>)

![alt text](<CleanShot 2026-02-26 at 04.40.46@2x.png>)

統合クエリエディタは 2025年11月に追加された機能で、外部クライアントのインストールや接続設定なしに、マネジメントコンソール上から直接 SQL を実行できます。

[https://aws.amazon.com/jp/about-aws/whats-new/2025/11/amazon-aurora-dsql-integrated-query-editor/:embed:cite]

構文ハイライトやオートコンプリートも備わっており、ちょっとした動作確認にはとても便利です。

クラスター作成から数秒でクエリを実行できる環境が整うので、今回のようなアップデートの検証にもぴったりですね。

#### IDENTITY列を使ったテーブル作成

では、アップデート内容の検証をしてみます。

まず、IDENTITY 列を持つシンプルなテーブルを作成します。

```sql
CREATE TABLE orders (
    id bigint GENERATED ALWAYS AS IDENTITY (CACHE 1),
    item_name varchar(100),
    amount numeric(10, 2),
    PRIMARY KEY (id)
);
```

`GENERATED ALWAYS AS IDENTITY` を指定すると、INSERT 時に `id` を省略しても自動で連番が振られます。

テーブルの作成は成功しますが、統合クエリエディタ上ではエラーが出ています。

![alt text](<CleanShot 2026-02-26 at 04.43.58@2x.png>)


`CACHE` はシーケンスの値を事前にメモリに確保しておく数を指定するパラメータです。

`CACHE 1` を指定しているのは、Aurora DSQL では `CACHE` の明示的な指定が必須であるためです。

なお、DSQL の CACHE は `1` または `65536 以上` のみ指定可能となっています。

`CACHE` の挙動やワークロードに応じた使い分けについては、下記ページに詳細があるのでご確認ください。

[https://docs.aws.amazon.com/aurora-dsql/latest/userguide/create-sequence-syntax-support.html:embed:cite]


では、データを挿入してみます。

```sql
INSERT INTO orders (item_name, amount) VALUES ('ノートPC', 128000.00);
INSERT INTO orders (item_name, amount) VALUES ('マウス', 3500.00);
INSERT INTO orders (item_name, amount) VALUES ('キーボード', 12800.00);
```

結果を確認します。


`id` が自動で 1, 2, 3 と採番されていますね。

![alt text](<CleanShot 2026-02-26 at 04.55.56@2x.png>)

#### SEQUENCEオブジェクトの作成

次に、SEQUENCE オブジェクトを直接作成して使ってみます。

```sql
CREATE SEQUENCE order_seq CACHE 65536 START WITH 1001;
```

こちらもまだ統合クエリエディタが対応できていないみたいです。

![alt text](<CleanShot 2026-02-26 at 04.57.30@2x.png>)

SEQUENCE を使ってデータを挿入します。

```sql
INSERT INTO orders (id, item_name, amount)
    OVERRIDING SYSTEM VALUE
    VALUES (nextval('order_seq'), 'モニター', 45000.00);
```

`GENERATED ALWAYS` で定義した IDENTITY 列に明示的に値を入れるため、`OVERRIDING SYSTEM VALUE` を指定しています。


動作確認してみます。


```sql
SELECT * FROM orders;
```

`order_seq` から採番された 1001 が入っていることが確認できました。

![alt text](<CleanShot 2026-02-26 at 04.58.40@2x.png>)

### UUIDとの使い分け

Aurora DSQL の公式ドキュメントでは、スケーラビリティが重要なワークロードでは UUID を推奨しています。

[https://docs.aws.amazon.com/aurora-dsql/latest/userguide/sequences-identity-columns-working-with.html:embed:cite]

UUID はセッション間の調整なしに生成できるため、分散システムである DSQL との相性が良いです。

一方、SEQUENCE / IDENTITY は人間が読みやすい連番が必要なケース（注文番号、サポートケース番号など）に向いています。

用途に応じて使い分けるのが良さそうですね。

## 2. NUMERIC型のインデックスサポート

次の検証をしていきます。

[https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-aurora-dsql-indexes-numeric-data-type/:embed:cite]

これまで Aurora DSQL では NUMERIC 型のカラムにインデックスを作成できませんでした。

今回のアップデートにより、NUMERIC カラムをプライマリキーやセカンダリインデックスに使用できるようになりました。

### やってみた

先ほど作成した `orders` テーブルの `amount` カラム（NUMERIC型）にインデックスを作成してみます。

なお、Aurora DSQL では `CREATE INDEX` ではなく `CREATE INDEX ASYNC` を使用します。

```sql
CREATE INDEX ASYNC idx_orders_amount ON orders (amount);
```

![alt text](<CleanShot 2026-02-26 at 05.03.11@2x.png>)


これで、例えば金額の範囲検索などでインデックスが活用されるようになります。

レコードを1000件程度に増やして、インデックスの効果を確認してみます。`generate_series` を使ってテストデータを投入します。

```sql
INSERT INTO orders (item_name, amount)
SELECT
    'item-' || g,
    (random() * 100000)::numeric(10,2)
FROM generate_series(1, 1000) AS g;
```

以下のSQLを実行してみます。

```sql
SELECT amount FROM orders WHERE amount >= 10000.00;
```

ここで `SELECT *` ではなく `SELECT amount` としているのは、インデックスに含まれるカラムだけを取得することで、テーブル本体へのアクセスなしにインデックスのみで結果を返せる（Index Only Scan）ようにするためです。`SELECT *` だとオプティマイザが主キーのスキャンを選択するケースがあります。

`EXPLAIN ANALYZE` で実行計画を確認してみます。

![alt text](<CleanShot 2026-02-26 at 05.10.05@2x.png>)

```
Limit  (cost=95978.27..96463.31 rows=10000 width=9) (actual time=1.281..5.958 rows=898 loops=1)
  ->  Index Only Scan using idx_orders_amount on orders  (cost=95978.27..133181.99 rows=767026 width=9) (actual time=1.280..5.876 rows=898 loops=1)
        Index Cond: (amount >= 10000.00)
        -> Storage Scan on idx_orders_amount  (cost=95978.27..133181.99 rows=767026 width=9) (actual rows=898 loops=1)
              Projections: amount
              -> B-Tree Scan on idx_orders_amount  (cost=95978.27..133181.99 rows=767026 width=9) (actual rows=898 loops=1)
                    Index Cond: (amount >= 10000.00)
Planning Time: 0.081 ms
Execution Time: 6.026 ms
```

`Index Only Scan using idx_orders_amount` と表示されており、NUMERIC 型のカラムに作成したインデックスが正しく使われていることが確認できました。

## まとめ

今回は Aurora DSQL の 2026年2月のアップデートから、IDENTITY 列・SEQUENCE オブジェクトのサポートと NUMERIC 型のインデックスサポートを試してみました。

IDENTITY 列と SEQUENCE により、これまでアプリケーション側で実装が必要だった連番 ID の生成がデータベース側で完結できるようになりました。PostgreSQL からの移行もよりスムーズになりそうです。

NUMERIC 型のインデックスサポートも、需要がありそうです。

ただし、CACHE の制約や引き続き残る制約もあるので、導入時には以下の公式ドキュメントで対応状況をご確認ください。

[https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility.html:embed:cite]

個人的には、DSQL が着実に PostgreSQL 互換性を高めてきている印象を受けました。今後のアップデートも楽しみです。

