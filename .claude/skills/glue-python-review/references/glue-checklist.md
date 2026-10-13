# Glue レビューチェックリスト

## 目次

1. Driver へのデータ集約
2. 分散処理を活かせていない書き方
3. 読み込み時のパーティション（プルーニング）
4. 書き込み時のパーティションとファイル数
5. 再計算とキャッシュ
6. Join とデータの偏り（スキュー）
7. DynamicFrame と DataFrame
8. ジョブブックマークとジョブのライフサイクル
9. 冪等性と書き込みモード
10. 設定・パラメータ・ログ
11. Python Shell ジョブ
12. プロジェクト固有ルール

---

## 1. Driver へのデータ集約

Driver は 1 ノードしかないため、分散データを Driver に集めると、データ量に比例してメモリを使い OOM になる。

| 要注意の呼び出し | 問題 | 代替 |
| --- | --- | --- |
| `df.collect()` | 全件を Driver に転送 | 変換は DataFrame API のまま行い、書き込みは Executor から行う |
| `df.toPandas()` | 全件を Driver の pandas に変換 | Spark の関数で処理する。pandas の書き方が必要なら `mapInPandas` / `applyInPandas` |
| `dyf.toDF().collect()` で行ループ | Driver 上の逐次処理になる | `withColumn` / `when` / `groupBy` などで列単位に処理する |
| `df.take(n)` / `limit(n).collect()` で n が大きい | 同上 | 確認用途なら小さい n に限定する |
| ブロードキャスト対象が大きい | Driver と全 Executor のメモリを消費する | 小さいテーブル（目安: 数十 MB）に限定する |

```python
# NG: Driver に全件を集めてループ
for row in df.collect():
    if row["status"] == "active":
        results.append(transform(row))

# OK: 分散したまま列単位に変換
from pyspark.sql import functions as F
result = df.filter(F.col("status") == "active").withColumn("x", F.upper("name"))
```

例外: 設定値や件数の少ないマスタなど、**上限件数が明らかに小さい**データの `collect()` は問題ない。その場合も、なぜ小さいと言えるのかが分かるようにコメントで残させる。

## 2. 分散処理を活かせていない書き方

- **Python UDF（`@udf`、`F.udf`）**: 行ごとに JVM と Python の間でデータをやり取りするため遅い。まず `pyspark.sql.functions` の組み込み関数で書けないかを確認する。どうしても必要なら `pandas_udf`（Arrow でまとめて処理される）を使う
- **`rdd.map` / `rdd.foreach` などの RDD API**: Catalyst による最適化が効かない。DataFrame API で書けないかを確認する
- **Executor 内での boto3 呼び出し**: `foreach` や UDF の中で 1 行ずつ S3 / DynamoDB API を呼ぶと、スロットリングやコネクション枯渇が起きる。`foreachPartition` でパーティション単位にクライアントを作りバッチ送信するか、Glue のコネクタ・`write_dynamic_frame` で書き込めないかを確認する
- **Driver 側での Python ループによる DataFrame 生成**: 日付ごとに `for` で読み込み → `union` を繰り返すと DAG が肥大化する。パスのリストやパーティション条件でまとめて 1 回で読む

## 3. 読み込み時のパーティション（プルーニング）

Data Catalog から読むとき、条件を付けずに全パーティションを読み、後から `filter` で絞っていないかを確認する。

```python
# NG: 全パーティションを S3 から読んでから絞る
dyf = glueContext.create_dynamic_frame.from_catalog(database="db", table_name="events")
df = dyf.toDF().filter("dt = '2026-10-01'")

# OK: 読む前にパーティションを絞る
dyf = glueContext.create_dynamic_frame.from_catalog(
    database="db",
    table_name="events",
    push_down_predicate="dt = '2026-10-01'",
    # パーティション数が非常に多いテーブルで、Partition Index がある場合はこちらも有効
    additional_options={"catalogPartitionPredicate": "dt = '2026-10-01'"},
    transformation_ctx="events_src",
)
```

- `push_down_predicate` は**パーティション列だけ**に効く。パーティション列でない条件を書いても、読み込むデータは減らない
- `spark.read.parquet(path)` で読む場合は、パーティションのパスまで指定するか、パーティション列に対する `filter` を読み込み直後に置く（Spark のパーティションプルーニングが効くようにする）
- 入力が小さいファイルの大群なら、`additional_options={"groupFiles": "inPartition", "groupSize": "134217728"}` でタスク数の爆発を防ぐ

## 4. 書き込み時のパーティションとファイル数

- **パーティション列の指定**: 後続でよく絞り込まれる列（日付など）を `partitionKeys` に指定しているか。逆に、カーディナリティが高い列（ユーザー ID など）をパーティション列にして、パーティションが大量にできていないか
- **小ファイル問題**: Spark のパーティション数だけファイルが出力される。数 KB のファイルが大量にできると、後続の読み込みと S3 のリスト取得が遅くなる。`coalesce` / `repartition` で出力ファイル数を調整する
  - `coalesce(n)`: シャッフルなしで減らせるが、上流の並列度まで下がることがある
  - `repartition(n, "dt")`: シャッフルが発生するが、パーティション列ごとにまとめられる
- **`coalesce(1)` / `repartition(1)`**: 1 タスクに全データが集まり、並列度がゼロになる。大きいデータでは High として指摘する
- **形式**: 分析用途の出力は Parquet などの列指向形式にする。CSV / JSON 出力には理由があるかを確認する

```python
glueContext.write_dynamic_frame.from_options(
    frame=DynamicFrame.fromDF(df.repartition("dt"), glueContext, "out"),
    connection_type="s3",
    connection_options={"path": "s3://bucket/out/", "partitionKeys": ["dt"]},
    format="glueparquet",
    transformation_ctx="out_sink",
)
```

## 5. 再計算とキャッシュ

Spark は Action のたびに DAG を最初から再計算する。

- 同じ DataFrame に対して `count()`、`write`、別の集計などの Action が複数あるなら、`cache()` / `persist()` で再計算を避けられないかを確認する。使い終わったら `unpersist()` する
- **ログ用の `count()`**: 「件数をログに出す」だけのために大きい DataFrame で `count()` すると、読み込みからすべてやり直しになる。必要性を確認する
- 逆に、1 回しか使わない DataFrame への `cache()` はメモリの無駄になる

## 6. Join とデータの偏り（スキュー）

- 片方が小さいテーブルなら `F.broadcast(small_df)` でシャッフルを避けられる
- Join キーに NULL や特定の値が集中していると、1 タスクだけが極端に遅くなる / OOM になる。キーの分布を考慮しているか（NULL を事前に除外する、salting など）
- Glue 4.0 以降（Spark 3.x）では AQE が有効なので、スキュー Join はある程度自動で緩和される。それでも、明らかな偏りはコードで対処すべき
- 直積（`crossJoin`、Join 条件の書き忘れ）がないか

## 7. DynamicFrame と DataFrame

- `toDF()` / `DynamicFrame.fromDF()` の往復が多くないか。DataFrame で処理するならまとめて変換し、入出力の境界でだけ DynamicFrame を使う
- スキーマが揺れる入力（同じ列に文字列と数値が混在するなど）では `resolveChoice` で型を明示しているか。`choice` 型のまま `toDF()` すると予期しない struct 型になる
- `ApplyMapping` で列名・型を明示しているか。暗黙の型推論に頼っていないか
- `spark.read` で `inferSchema=True` を大きい CSV に使うと、推論のためにデータを全件読み込む。スキーマを明示する

## 8. ジョブブックマークとジョブのライフサイクル

ブックマーク（処理済みデータを記録して差分処理する機能）を使うジョブでは以下が揃っていないと、**同じデータを何度も処理したり、データを取りこぼしたりする**。

- `job.init(args["JOB_NAME"], args)` で始まり、`job.commit()` で終わっているか
- ソース / シンクの各呼び出しに**一意で安定した** `transformation_ctx` が付いているか（実行ごとに変わる値を入れていないか）
- 例外で途中終了したときに `job.commit()` が呼ばれない構造になっているか（失敗したのにブックマークが進むと、データを取りこぼす）
- `spark.read` で直接 S3 を読む部分はブックマークの対象外になる点を理解しているか

## 9. 冪等性と書き込みモード

Glue ジョブはリトライや再実行がありうる。同じ入力で 2 回動かしても結果が壊れないかを確認する。

- `mode("append")` で再実行時に重複しないか
- パーティション単位で上書きするなら `spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")` を使い、`overwrite` で**テーブル全体を消していないか**
- `purge_s3_path` などで削除してから書く処理は、途中で失敗するとデータが消えたままになる

## 10. 設定・パラメータ・ログ

- S3 パス、DB 名、日付などがハードコードされていないか。`getResolvedOptions(sys.argv, [...])` でジョブパラメータとして受け取る
- 任意のパラメータを `getResolvedOptions` で必須として扱い、未指定時にジョブが落ちないか
- `print` ではなく `glueContext.get_logger()` または `logging` を使っているか
- 例外を握りつぶしてジョブが「成功」扱いになっていないか
- 認証情報のハードコードがないか（IAM ロール / Secrets Manager / Glue Connection を使う）

## 11. Python Shell ジョブ

Spark ではなく単一マシン上の Python で動く。メモリは最大で 1 DPU = 16 GB（0.0625 DPU なら 1 GB）。

- 大きいファイルを `pd.read_csv` で一度に読んでいないか。`chunksize` での分割読み込みや、`pyarrow` で必要な列だけ読むことを検討する
- データ量が増える見込みがあるなら、そもそも Spark ETL ジョブ（またはAthena / Redshift への処理移譲）にすべきでないか
- boto3 で S3 のオブジェクトを列挙するとき、ページネーション（`get_paginator`）を使っているか

## 12. プロジェクト固有ルール

現時点では未定義。定義されるまでは 1〜11 の一般的な観点だけでレビューする。

<!-- プロジェクト開始後、命名規則・必須パラメータ・出力先の構成・標準の Glue バージョンなどが決まったらここに追記する -->
<!-- レビューで同じ指摘が何度も出たら、それをここにルールとして追加するのもよい -->

---

## 主な参考ドキュメント

- AWS Glue 開発者ガイド: https://docs.aws.amazon.com/glue/latest/dg/
- AWS Glue ジョブのパフォーマンスチューニングのベストプラクティス（AWS Prescriptive Guidance）: https://docs.aws.amazon.com/prescriptive-guidance/latest/tuning-aws-glue-for-apache-spark/
- Spark SQL パフォーマンスチューニング: https://spark.apache.org/docs/latest/sql-performance-tuning.html
