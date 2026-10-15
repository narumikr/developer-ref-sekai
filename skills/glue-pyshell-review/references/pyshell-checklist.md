# Glue Python Shell レビューチェックリスト

ここに書いた「問題になりやすい書き方」は、コード上で見つけたら即断定するものではなく、**疑って確認するきっかけ**として使う。
小規模なジョブが対象なので、データ量が明らかに小さく今後も増えない処理に、大規模向けの対策を求めない。

## 目次

1. メモリ（1 台に載るか）
2. 失敗の検知と終了ステータス
3. 再実行・失敗時の整合性（冪等性）
4. boto3 と外部呼び出し
5. Athena / Redshift へのクエリ発行
6. データの読み書きと型
7. パラメータ・ログ・ライブラリ
8. 運用設定
9. セキュリティ
10. プロジェクト固有ルール

---

## 1. メモリ（1 台に載るか）

DPU 0.0625 ならメモリは 1 GB しかない。pandas の DataFrame は元のファイルサイズの数倍のメモリを使うことがあるため、「ファイルが 300 MB だから 1 GB に載る」とは言えない。

| 要注意の書き方 | 問題 | 代替 |
| --- | --- | --- |
| `s3.get_object(...)["Body"].read()` で大きいファイルを読む | ファイル全体がメモリに載る | `iter_lines()` / `iter_chunks()` でストリーム処理する、`download_file` で `/tmp` に落としてから分割して読む |
| `pd.read_csv(path)` で全件・全列を読む | 不要な列や行までメモリに載る | `usecols` で必要な列だけ、`chunksize` で分割、`dtype` を明示して型推論の無駄を減らす |
| 複数ファイルを読んで `pd.concat` で 1 つにまとめる | 全ファイルの合計がメモリに載る | ファイルごとに処理して書き出す。どうしても結合が必要なら合計サイズを見積もる |
| 結果をリストに `append` し続けて最後に書き出す | 件数に比例してメモリを使う | 一定件数ごとに書き出す・送信する |

```python
# 要確認: ファイル全体をメモリに載せている
body = s3.get_object(Bucket=bucket, Key=key)["Body"].read()
df = pd.read_csv(io.BytesIO(body))

# 推奨: 必要な列だけ、分割して読む
for chunk in pd.read_csv(
    f"s3://{bucket}/{key}",  # s3fs / awswrangler が使える環境の場合
    usecols=["id", "amount", "created_at"],
    dtype={"id": "string"},
    chunksize=100_000,
):
    process(chunk)
```

- データ量が明らかに小さい（数 MB の設定ファイル、件数の少ないマスタなど）なら、丸ごと読んでよい。その場合も「なぜ小さいと言えるか」をコメントで残させる
- `/tmp` は容量に限りがある一時領域で、ジョブ終了とともに消える。大きいファイルを複数置く処理では容量を確認させる

## 2. 失敗の検知と終了ステータス

小規模ジョブで一番多い事故は「失敗しているのに誰も気づかない」こと。ジョブが失敗ステータスにならなければ、通知も再実行も動かない。

- **例外の握りつぶし**: `except Exception: pass` や、ログを出すだけで処理を続行し、最後まで到達して正常終了していないか
- **一部失敗の扱い**: 100 ファイル中 3 ファイルだけ失敗した場合、ジョブを失敗にするのか、成功にして失敗分を記録するのか。どちらにしても**失敗件数がログに残り、閾値を超えたら例外で終了する**ようになっているか
- **`sys.exit(0)` での早期終了**: エラー時に `sys.exit(0)` していないか。失敗時は例外を送出するか、0 以外で終了する
- **入力が 0 件のとき**: 「入力がない」のが正常なのか異常なのか。上流の遅延で入力がないのに、成功として終わっていないか

```python
# NG: 失敗しても成功扱いになる
for key in keys:
    try:
        process(key)
    except Exception as e:
        logger.error(f"failed: {key} {e}")

# OK: 失敗を集計し、最後にジョブを失敗させる
failed_count = 0
failed_samples = []  # 件数が多くてもメモリを使い過ぎないよう、サンプルだけ保持する
for key in keys:
    try:
        process(key)
    except Exception:
        logger.exception("failed: %s", key)
        failed_count += 1
        if len(failed_samples) < 10:
            failed_samples.append(key)

if failed_count:
    raise RuntimeError(f"{failed_count} / {len(keys)} files failed (sample: {failed_samples})")
```

## 3. 再実行・失敗時の整合性（冪等性）

Python Shell ではジョブブックマークが使えない。「どこまで処理したか」の管理は自前で行う必要がある。冪等性 = 同じ入力で何度実行しても結果が同じになること。
「途中で落ちたら」「もう一度実行したら」「2 つ同時に動いたら」の 3 つを確認する。

- **処理済みの管理方法が決まっているか**: 例として、処理後に入力を `processed/` へ移動する、処理済みのキーを DynamoDB などに記録する、日付パラメータで対象を決めて毎回その日の分を作り直す、など。何もしていないと、毎回すべての入力を再処理する
- **入力を移動・削除する順序**: S3 にはリネームがなく、移動は「コピー → 削除」になる。**出力の書き込みが成功してから**入力を移動・削除しているか。先に消すと、その後の失敗でデータが失われる
- **出力のキー**: 出力先のキーが入力や対象日から決まる（例: `output/dt=2026-10-01/data.csv`）なら、再実行しても上書きになり重複しない。実行時刻や UUID をキーに含めると、再実行のたびにファイルが増えて重複する
- **DB への書き込み**: `INSERT` だけだと再実行で重複する。一意キー＋ `UPSERT`、または対象範囲を `DELETE` してから `INSERT` を同一トランザクションで行うなど
- **同時実行**: 同じ入力・出力を扱うジョブが重なると壊れうる。ジョブ定義の同時実行数（`MaxConcurrentRuns`）を確認する

## 4. boto3 と外部呼び出し

- **ページネーション**: `list_objects_v2` は 1 回で最大 1,000 件しか返さない。`get_paginator` を使っているか。DynamoDB の `scan` / `query`、その他の `list_*` 系 API も同様に、続きのページの取得漏れがないか

  ```python
  # NG: 1,000 件を超えると黙って取りこぼす
  resp = s3.list_objects_v2(Bucket=bucket, Prefix=prefix)
  keys = [o["Key"] for o in resp.get("Contents", [])]

  # OK
  paginator = s3.get_paginator("list_objects_v2")
  keys = [
      o["Key"]
      for page in paginator.paginate(Bucket=bucket, Prefix=prefix)
      for o in page.get("Contents", [])
  ]
  ```

- **リトライ**: boto3 には標準でリトライの仕組みがある（明示的に設定しなければ `legacy` モードで、初回を含め最大 5 回試行する）。**`Config` でリトライを設定していないこと自体は問題にしない**。指摘するのは、大量に API を呼ぶ・スロットリングが起きやすい処理などで、実効のリトライ方針が処理に見合っているか疑わしい場合に限る。確認するときの注意点は以下のとおり
  - 実効設定は、コード内の `Config`、AWS 設定ファイル、環境変数（`AWS_RETRY_MODE` / `AWS_MAX_ATTEMPTS`）のどこで決まっているかを確認する
  - `Config(retries={"max_attempts": n})` は**初回を含まない**（リトライ回数）。設定ファイルや `AWS_MAX_ATTEMPTS` の `max_attempts` は**初回を含む**（総試行回数）。紛らわしいので、`Config` では総試行回数を表す `total_max_attempts` を使うと明確になる
  - boto3 のリトライが対象にするのは、スロットリングや一時的なエラーなど特定のエラーだけ。業務的な失敗（バリデーションエラーなど）はリトライされない
  - 自前でリトライを書いている場合は、待ち時間を指数的に伸ばしているか、boto3 のリトライと二重になって試行回数が膨らんでいないか
- **一括 API の利用**: 1 件ずつ `put_item` するより、一括 API を使えないか。ただし、一部の項目だけ失敗したときの扱いは API によって異なるので区別する
  - **DynamoDB の `Table.batch_writer()`**（高レベル API）: 未処理の項目（`UnprocessedItems`）を**自動で再送する**。`batch_writer()` を使っているコードに、`UnprocessedItems` の処理がないことを指摘しない
  - **`client.batch_write_item()`** などの低レベル API を直接使う場合: レスポンスの `UnprocessedItems` に未処理の項目が返ることがある。これを再送しないと、**エラーにならずに一部のデータが書き込まれない**。再送処理があるかを確認する
  - その他の一括 API（SQS の `send_message_batch` の `Failed` など）も、部分的な失敗がレスポンスで返る形式のものは同様に確認する
- **クライアントの生成**: ループの中で毎回 `boto3.client(...)` を作っていないか。ループの外で 1 回作って使い回す
- **外部 API（HTTP）**: `requests` などでタイムアウトを指定しているか。指定しないと、応答がないときにジョブのタイムアウトまで待ち続ける。レスポンスのステータスコードを確認しているか

## 5. Athena / Redshift へのクエリ発行

データを Python に持ってこずに SQL で処理させるのは、Python Shell で大きめのデータを扱う良い方法。Athena も Redshift Data API も**非同期実行**で、クエリを投げた API はすぐに返る。完了を確認しないまま次に進むのが典型的な不具合になる。

### 共通

- **完了待ちと失敗判定**: 実行 ID を使って状態を確認し、**失敗系の状態のときに例外にしているか**。完了を待たずに次の処理に進んでいないか
- **待ちの上限**: ポーリングに上限時間があるか。無限ループになっていないか
- **SQL の組み立て**: パラメータを文字列連結で SQL に埋め込んでいないか。値がジョブパラメータなど外部から来るなら、想定外の値で SQL が壊れる。パラメータ化クエリ（Athena の `ExecutionParameters`、Redshift Data API の `Parameters`）や、許可する値の検証を検討する。ただし、これらのパラメータが使えるかは boto3 のバージョンに依存するので、7. の library-set と合わせて確認する
- awswrangler を使っている場合は、`wr.athena.read_sql_query` などが完了待ちや失敗判定を内部で扱っていることが多い。ラッパーの動きを確認した上で、二重に指摘しない

### Athena

- **状態の確認**: `start_query_execution` の後、`get_query_execution` で状態を確認する。`SUCCEEDED` 以外（`FAILED` / `CANCELLED`）を失敗として扱っているか
- **結果の取得**: 大きい結果を `get_query_results` で取ると、ページネーションが必要で遅い。結果を Python で使わないなら `CTAS` / `UNLOAD` / `INSERT INTO` で S3 やテーブルに直接書かせる
- **失敗時の部分出力**: `CTAS` / `INSERT INTO` が失敗・キャンセルされた場合、出力先に**途中まで書かれたファイルが残る**ことがある。Athena が削除しない場合があり、残ったファイルが後続のクエリの結果に含まれてしまう。後続処理がその出力先を読むなら、失敗時に出力先をどう扱うか（削除する、出力先を実行ごとに分けて成功時だけ参照先を切り替える など）が決まっているかを確認する
- **`CTAS` の出力先**: `external_location` は空である必要があり、既存のデータがあると失敗する。再実行時に出力先をどう空けるかが決まっているか（3. の冪等性とあわせて確認）

### Redshift（Data API）

- **状態の確認**: `execute_statement` / `batch_execute_statement` の後、返された `Id` を使って `describe_statement` で状態を確認する。`FINISHED` 以外（`FAILED` / `ABORTED`）を失敗として扱い、`Error` の内容をログに残しているか
- **`WaitTimeSeconds`**: `execute_statement` / `describe_statement` の `WaitTimeSeconds` で 1 回の呼び出しで待てるのは最大 30 秒。それより長くかかりうるクエリでは、状態が `FINISHED` などになるまで繰り返し確認する必要がある。1 回の待ちで完了した前提で次に進んでいないか
- **リクエストの冪等性**: `ClientToken` を指定すると、同じリクエストの再送で SQL が二重に実行されるのを防げる。リトライで同じ `INSERT` などが二重に実行されうる処理では、利用を検討する（3. の DB への書き込みの観点もあわせて確認する）
- **複数の SQL の整合性**: 複数の SQL を `execute_statement` で 1 つずつ投げると、途中で失敗したときに前半だけが反映される。まとめて成否を揃えたいなら `batch_execute_statement` を検討する。既定（`ExecutionMode` が `TRANSACTION`）では 1 つのトランザクションとして実行され、1 つでも失敗すると全体がロールバックされる。`AUTO_COMMIT` を指定している場合は個別にコミットされるので、この前提は成り立たない
- `WaitTimeSeconds` / `ClientToken` / `ExecutionMode` などは比較的新しいパラメータのため、使える boto3 のバージョンかを 7. の library-set と合わせて確認する
- `redshift_connector` / `psycopg2` などのドライバーで直接接続している場合は同期実行になる。その場合はトランザクションのコミット・ロールバックと、接続のタイムアウトを確認する

## 6. データの読み書きと型

- **文字コード**: 入力 CSV の文字コード（UTF-8 / Shift_JIS / CP932 など）を明示しているか。既定のまま読むと、想定外のファイルで文字化け・例外になる。BOM 付き UTF-8 の扱いも確認する
- **型推論**: pandas は型を推論するため、ID や郵便番号の先頭の `0` が消える、欠損値を含む整数列が `float` になる、などが起きる。`dtype` で明示しているか
- **金額・精度**: 金額を `float` で計算していないか（`decimal.Decimal` を検討）
- **タイムゾーン**: 「前日分」「対象日」の計算が UTC と JST のどちらの基準か明確か。Glue の実行環境は UTC で動く前提で確認する
- **不正レコード**: パースできない行や必須項目の欠損をどう扱うか（落とす / 別出力 / ジョブを失敗させる）。黙って落としていないか、件数がログに残るか
- **出力形式**: 後続で Athena などから分析するなら、CSV より Parquet などの列指向形式の方が読み込み量が減る。ただし後続が CSV を前提にしているなら無理に変えない

## 7. パラメータ・ログ・ライブラリ

この章の項目は、書き方の形式だけで機械的に指摘しない。**運用上の影響がある場合**（環境ごとに値を変えられずデプロイ事故になる、障害時に原因を追えない など）は指摘し、影響がなく好みの範囲なら 💡Suggestion にとどめる。

- **ハードコード**: S3 パス、バケット名、日付などがハードコードされていないか。環境（開発 / 本番）や実行ごとに変わる値がハードコードされていて、変更のたびにコード修正が必要になる・誤った環境に書き込みうる場合に指摘する。実行ごとに変わらない定数まで外出しを求めない。ジョブパラメータは `awsglue.utils.getResolvedOptions(sys.argv, [...])` で受け取れる
- `getResolvedOptions` は指定した引数をすべて必須として扱う。任意のパラメータを渡していないとジョブが落ちるので、任意のものは `sys.argv` を確認してから取得するなどの対応が必要
- **ログの内容**: 処理対象（ファイル・対象日など）、処理件数、失敗件数と失敗理由など、**障害時に後から調査できる情報が残っているか**を確認する。不足していれば指摘する。`print` の出力も CloudWatch Logs（`/aws-glue/python-jobs/output`）に残るため、`print` を使っていること自体は問題にしない。ログレベルで出し分けたい場合などに `logging` の利用を 💡Suggestion として提案する
- **library-set と同梱ライブラリのバージョン**: Python 3.9 の Python Shell では、ジョブ定義の `library-set`（`analytics` / `none`）によって最初から使えるライブラリが異なる。`analytics` には pandas / awswrangler / boto3 などが**固定のバージョン**で同梱されており、最新版ではない。コードが使っている API や引数（例えば新しめの boto3 の API パラメータ）が、実際の library-set のバージョンで使えるかを確認してから前提にする。`none` なら pandas なども自分で追加する必要がある
- **追加ライブラリのバージョン**: `--additional-python-modules` などで追加・更新するライブラリのバージョンを固定しているか。固定しないと、ある日の実行から挙動が変わる。ジョブの Python バージョンで動くかも確認する

## 8. 運用設定

コードだけでは判断できないことが多い。ジョブ定義（IaC など）があれば合わせて確認し、なければ確認事項として挙げる。

- **DPU**: 0.0625（1 GB）で足りるか、1（16 GB）が必要か。1. のメモリの見積もりと合わせて判断する。「OOM したから 1 DPU にする」前に、全件読み込みなどの原因を疑う
- **タイムアウト**: 既定値任せにせず、通常の処理時間に見合った値になっているか。長すぎると、ハングしたジョブに気づくのが遅れ、その間の料金もかかる
- **リトライ**: ジョブのリトライを有効にするなら、3. の冪等性が前提になる。冪等でないジョブを自動リトライすると重複の原因になる
- **失敗の通知**: ジョブの失敗（`FAILED` / `TIMEOUT`）を EventBridge などで検知し、誰かに通知される仕組みがあるか。2. で失敗を正しく表に出しても、通知がなければ気づけない

## 9. セキュリティ

- 認証情報や API キーのハードコードがないか。AWS リソースへのアクセスはジョブの IAM ロールで行い、DB のパスワードや外部 API のキーは Secrets Manager や Glue Connection から取得する
- IAM ロールの権限が、ジョブが行う操作とリソースに対して最小限か。`*` があるだけで過剰権限と判定しない（AWS の操作には、リソースを個別に指定できず `*` が必要なものもある）。必要な操作・リソースより明らかに広い権限（使わない操作の許可、無関係なバケット全体への書き込み など）を指摘し、`*` を使っている場合は、その理由が説明できるかを確認する
- ログに個人情報や機密値（レコードの中身、トークンなど）を出力していないか
- VPC 内のリソース（RDS など）に接続する場合、Glue Connection の設定が前提として整理されているか

## 10. プロジェクト固有ルール

現時点では未定義。定義されるまでは 1〜9 の一般的な観点だけでレビューする。

<!-- プロジェクト開始後、命名規則・必須パラメータ・出力先の構成・処理済み管理の標準方式などが決まったらここに追記する -->
<!-- レビューで同じ指摘が何度も出たら、それをここにルールとして追加するのもよい -->

---

## 主な参考ドキュメント

### AWS Glue

- Python Shell ジョブ（library-set、同梱ライブラリのバージョン、ログの出力先）: https://docs.aws.amazon.com/glue/latest/dg/add-job-python.html
- AWS Glue のジョブパラメータ: https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-glue-arguments.html
- AWS Glue 開発者ガイド: https://docs.aws.amazon.com/glue/latest/dg/

### boto3

- リトライ（既定の動作、`max_attempts` と `total_max_attempts`）: https://docs.aws.amazon.com/boto3/latest/guide/retries.html
- ページネーション: https://docs.aws.amazon.com/boto3/latest/guide/paginators.html
- DynamoDB（`batch_writer()` の自動再送）: https://docs.aws.amazon.com/boto3/latest/guide/dynamodb.html

### Athena

- `CTAS` の注意点（失敗時に残るファイル、`external_location`）: https://docs.aws.amazon.com/athena/latest/ug/considerations-ctas.html
- パラメータ化クエリ: https://docs.aws.amazon.com/athena/latest/ug/querying-with-prepared-statements.html

### Redshift Data API

- Data API の使い方: https://docs.aws.amazon.com/redshift/latest/mgmt/data-api.html
- ExecuteStatement（`ClientToken`、`WaitTimeSeconds`）: https://docs.aws.amazon.com/redshift-data/latest/APIReference/API_ExecuteStatement.html
- DescribeStatement（状態の確認）: https://docs.aws.amazon.com/redshift-data/latest/APIReference/API_DescribeStatement.html
- BatchExecuteStatement: https://docs.aws.amazon.com/redshift-data/latest/APIReference/API_BatchExecuteStatement.html

### その他

- IAM のベストプラクティス（最小権限）: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- 各サービスの操作ごとのリソース指定の可否（Service Authorization Reference）: https://docs.aws.amazon.com/service-authorization/latest/reference/reference.html
- pandas `read_csv`（`usecols` / `dtype` / `chunksize` / `encoding`）: https://pandas.pydata.org/docs/reference/api/pandas.read_csv.html
- Requests のタイムアウト: https://requests.readthedocs.io/en/latest/user/advanced/#timeouts
