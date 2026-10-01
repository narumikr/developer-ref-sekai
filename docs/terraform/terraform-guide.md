# Terraform 環境構築ガイド

---

## FAQ・トラブルシューティング

本章は、Terraform のインストールから AWS 環境構築までの過程で**非開発者がつまずきやすいポイント**を症状別にまとめたものです。
エラーメッセージや状況が自分の症状に近い項目を探し、**上から順に切り分け手順を実行**してください。

- 対象 OS: **Windows / macOS**
- 本手順の認証方式: **AWS Access Portal (IAM Identity Center / SSO)** で発行される一時認証情報を、`~/.aws/credentials` への貼り付けまたは環境変数の export で Terraform に渡します
- 利用する permission set: **`PowerUserAccessIAMFullAccess_set`** （`PowerUserAccess` + `IAMFullAccess` の組み合わせ）

---

### 目次

- [Terraform 環境構築ガイド](#terraform-環境構築ガイド)
  - [FAQ・トラブルシューティング](#faqトラブルシューティング)
    - [目次](#目次)
  - [1. インストール関連](#1-インストール関連)
    - [症状 1-1: `terraform` コマンドが見つからない](#症状-1-1-terraform-コマンドが見つからない)
    - [症状 1-2: バージョンが要件と合わない](#症状-1-2-バージョンが要件と合わない)
    - [症状 1-3: 社内プロキシ環境でダウンロード・`init` が通らない](#症状-1-3-社内プロキシ環境でダウンロードinit-が通らない)
  - [2. AWS 認証・credentials 設定](#2-aws-認証credentials-設定)
    - [症状 2-1: `No valid credential sources found` / `InvalidClientTokenId`](#症状-2-1-no-valid-credential-sources-found--invalidclienttokenid)
      - [方式 A: `~/.aws/credentials` に貼り付けている場合](#方式-a-awscredentials-に貼り付けている場合)
      - [方式 B: 環境変数で export している場合](#方式-b-環境変数で-export-している場合)
    - [症状 2-2: `UnauthorizedOperation` / `AccessDenied`](#症状-2-2-unauthorizedoperation--accessdenied)
    - [症状 2-3: 想定と違う AWS アカウント／リージョンにリソースが作られる](#症状-2-3-想定と違う-aws-アカウントリージョンにリソースが作られる)
    - [症状 2-4: 作業中に突然 `ExpiredToken` が出る（一時認証の失効）](#症状-2-4-作業中に突然-expiredtoken-が出る一時認証の失効)
  - [3. Terraform 実行時エラー](#3-terraform-実行時エラー)
    - [症状 3-1: `terraform init` が失敗する](#症状-3-1-terraform-init-が失敗する)
    - [症状 3-2: `Error acquiring the state lock`](#症状-3-2-error-acquiring-the-state-lock)
    - [症状 3-3: `plan` では差分が無いのに `apply` でエラー](#症状-3-3-plan-では差分が無いのに-apply-でエラー)
    - [症状 3-4: 既存リソースと名前が衝突する (`AlreadyExists` 系)](#症状-3-4-既存リソースと名前が衝突する-alreadyexists-系)
  - [🆘 それでも解決しないとき](#-それでも解決しないとき)

---

## 1. インストール関連

### 症状 1-1: `terraform` コマンドが見つからない

**症状例**
- Windows: `'terraform' は、内部コマンドまたは外部コマンド... として認識されていません。`
- macOS: `zsh: command not found: terraform`

**切り分け手順**

1. ターミナルを**開き直す**（インストール直後は環境変数が反映されていないことがある）
2. 実体の有無を確認する
   - Windows (PowerShell): `Get-Command terraform` もしくは `where.exe terraform`
   - macOS: `which terraform`
3. PATH を確認する
   - Windows (PowerShell): `$env:Path -split ';'`
   - macOS: `echo $PATH | tr ':' '\n'`

**解決策**

- **Windows**: `terraform.exe` を置いたフォルダ（例: `C:\tools\terraform\`）を「システム環境変数 → Path」に追加。PowerShell を**再起動**。
- **macOS**: Homebrew でインストール済みの場合は `brew link terraform` で再リンク。Apple Silicon では PATH に `/opt/homebrew/bin` が含まれているかを確認。

> 💡 PATH を変更したら**ターミナルを完全に開き直す**。既存のタブ・ウィンドウでは反映されません。

---

### 症状 1-2: バージョンが要件と合わない

**症状例**
- `Error: Unsupported Terraform Core version`
- `This configuration does not support Terraform version X.Y.Z`

**切り分け手順**

1. 現在のバージョンを確認: `terraform version`
2. 本リポジトリで要求されるバージョンを確認
   - 要求バージョン: **`required_version >= 1.6.0`**（`versions.tf` に定義）

**解決策**

- バージョン管理ツールの利用を推奨:
  - Windows / macOS 共通: **[tfenv](https://github.com/tfutils/tfenv)** (macOS) / **[tfswitch](https://tfswitch.warrensbox.com/)** (クロスプラットフォーム)
- 単一バージョンのみ使う場合は、公式サイトから対象バージョンの zip を取得し置き換え。

---

### 症状 1-3: 社内プロキシ環境でダウンロード・`init` が通らない

**症状例**
- `Failed to query available provider packages`
- `tls: handshake failure`
- プロバイダのダウンロードが途中で止まる

**切り分け手順**

1. プロキシ設定を確認
   - Windows (PowerShell): `[System.Net.WebProxy]::GetDefaultProxy()`
   - macOS: `scutil --proxy`
2. 環境変数が設定されているか確認
   - `echo $env:HTTPS_PROXY` (PowerShell) / `echo $HTTPS_PROXY` (macOS)

**解決策**

- プロキシ環境変数を設定（**セッション単位**で設定するのが安全）
  ```powershell
  # Windows (PowerShell)
  $env:HTTP_PROXY  = "http://proxy.example.com:8080"
  $env:HTTPS_PROXY = "http://proxy.example.com:8080"
  $env:NO_PROXY    = "localhost,127.0.0.1,.internal.example.com"
  ```
  ```bash
  # macOS
  export HTTP_PROXY="http://proxy.example.com:8080"
  export HTTPS_PROXY="http://proxy.example.com:8080"
  export NO_PROXY="localhost,127.0.0.1,.internal.example.com"
  ```
- 社内 CA 証明書が必要な場合は `SSL_CERT_FILE` に PEM ファイルパスを指定。

---

## 2. AWS 認証・credentials 設定

### 症状 2-1: `No valid credential sources found` / `InvalidClientTokenId`

**症状例**
```
Error: No valid credential sources found
```
```
InvalidClientTokenId: The security token included in the request is invalid
```

本手順では AWS Access Portal で発行された **3 つのキー**（access_key_id / secret_access_key / **session_token**）を、`credentials` ファイルまたは環境変数で Terraform に渡します。方式別に切り分けます。

#### 方式 A: `~/.aws/credentials` に貼り付けている場合

**切り分け手順**

1. ファイル存在を確認
   - Windows (PowerShell): `Test-Path "$env:USERPROFILE\.aws\credentials"`
   - macOS: `ls -la ~/.aws/credentials`
2. 中身を表示して、以下 **4 行**が揃っているか確認
   - Windows: `Get-Content "$env:USERPROFILE\.aws\credentials"`
   - macOS: `cat ~/.aws/credentials`

   期待する形式：
   ```ini
   [default]
   aws_access_key_id     = ASIA...
   aws_secret_access_key = ...
   aws_session_token     = ...
   ```
3. 疎通確認: `aws sts get-caller-identity`

**よくある落とし穴**

- ❌ **`aws_session_token` を貼り忘れている** — SSO では**必須**。access_key と secret だけコピーして止まるパターンが最頻出
- ❌ `[default]` などのプロファイル名の行が無い
- ❌ Windows の「メモ帳」で保存して **UTF-8 BOM** が付いている → 「メモ帳」の保存時に文字コードを **ANSI** または **UTF-8 (BOM なし)** に変更するか、VS Code 等のエディタで保存する
- ❌ 行末やキーの前後に空白・全角スペースが混入している
- ❌ Access Portal 画面を途中で閉じて `aws_session_token` の値が切れている（ブラウザで全文表示→コピー）

#### 方式 B: 環境変数で export している場合

**切り分け手順**

1. **Terraform を実行する同じターミナル**で、3 つの変数が設定されているか確認
   ```powershell
   # Windows (PowerShell)
   $env:AWS_ACCESS_KEY_ID
   $env:AWS_SECRET_ACCESS_KEY
   $env:AWS_SESSION_TOKEN
   ```
   ```bash
   # macOS
   echo $AWS_ACCESS_KEY_ID
   echo $AWS_SECRET_ACCESS_KEY
   echo $AWS_SESSION_TOKEN
   ```
2. 疎通確認: `aws sts get-caller-identity`

**よくある落とし穴**

- ❌ **`AWS_SESSION_TOKEN` を export し忘れている**
- ❌ **別のターミナルで Terraform を実行している** — 環境変数は**開いているセッション単位**でしか有効ではない。export したターミナルと同じウィンドウ・タブで `terraform` を実行する
- ❌ **PowerShell で `export VAR=...` と書いている** — それは bash の構文。PowerShell では `$env:VAR = "値"` が正解
- ❌ **cmd.exe と PowerShell を混同** — cmd では `set VAR=値`、PowerShell では `$env:VAR = "値"`

> ⚠️ `credentials` ファイルは**絶対に Git にコミットしない**こと。既定の `~/.aws/credentials` 配置であれば Git リポジトリ外なので安全ですが、プロジェクト内に `.aws/` を作らないように注意。

---

### 症状 2-2: `UnauthorizedOperation` / `AccessDenied`

**症状例**
- `Error: UnauthorizedOperation: You are not authorized to perform this operation.`
- `AccessDenied: User: arn:aws:iam::123456789012:user/xxx is not authorized to perform: ec2:RunInstances`

**切り分け手順**

1. 自分が今「誰として」操作しているかを確認
   ```
   aws sts get-caller-identity
   ```
2. エラーメッセージから**必要なアクション名**（例: `ec2:RunInstances`）を控える
3. 自分の IAM ユーザー/ロールにそのアクションが許可されているか確認

**解決策**

本プロジェクトで前提とする権限セット: **`PowerUserAccess` + `IAMFullAccess`**
（AWS Access Portal 上の permission set 名: **`PowerUserAccessIAMFullAccess_set`**）

- `PowerUserAccess`: IAM・Organizations 以外のほぼ全サービスへの操作権限
- `IAMFullAccess`: IAM リソース（ロール・ポリシー等）の作成・変更権限

**切り分け手順（追加）**

1. 自分が今ログインしている permission set を確認
   ```
   aws sts get-caller-identity
   ```
   `Arn` が `arn:aws:sts::<AccountId>:assumed-role/AWSReservedSSO_PowerUserAccessIAMFullAccess_set_xxxxxxxx/<username>` の形式になっているか確認。
2. 異なるロール（例: `ReadOnlyAccess` 系）でログインしていた場合は、AWS Access Portal で `PowerUserAccessIAMFullAccess_set` のロールを選び**認証情報を取り直してから** credentials ファイル／環境変数を更新。

---

### 症状 2-3: 想定と違う AWS アカウント／リージョンにリソースが作られる

**症状例**
- 検証用アカウントに作ったつもりが本番アカウントに出来てしまった
- Tokyo に作るはずが Virginia にできている

**切り分け手順**

1. アカウント ID を確認: `aws sts get-caller-identity`
2. リージョン設定を確認（優先度順）
   - 環境変数 `AWS_REGION` / `AWS_DEFAULT_REGION`
   - `~/.aws/config` の `region`
   - Terraform コード内の `provider "aws" { region = "..." }`

**解決策**

- `apply` の前に必ず `terraform plan` の出力冒頭で **AWS アカウント ID とリージョン**を目視確認する運用にする。
- 複数アカウントを扱う場合はプロファイルを明示的に分け、環境変数で切り替える。

> 💡 **非開発者向けチェックリスト**: `apply` を実行する前に以下 3 点を声に出して確認する習慣をつけると事故が激減します：**①誰として ②どのアカウントに ③どのリージョンに** 作ろうとしているか。

---

### 症状 2-4: 作業中に突然 `ExpiredToken` が出る（一時認証の失効）

**症状例**
```
ExpiredToken: The security token included in the request is expired
```
- 作業開始時は動いていたのに、途中から急にエラーになる
- 昨日の credentials ファイルをそのまま使っている

**原因**: AWS Access Portal で発行される一時認証情報には**有効期限**があります（既定で 1 時間程度、設定により最大 12 時間）。期限切れ後は再取得が必要です。

**解決策（再取得フロー）**

1. ブラウザで AWS Access Portal を開く
2. 対象アカウントを選び、permission set **`PowerUserAccessIAMFullAccess_set`** の「**Access keys**」をクリック
3. 画面に表示される 3 つの値（access_key_id / secret_access_key / **session_token**）を全文コピー
4. 自分の運用方式に応じて更新：
   - **方式 A（credentials ファイル）**: `~/.aws/credentials` の該当プロファイルを**丸ごと上書き**（古い `aws_session_token` が残っていると失敗します）
   - **方式 B（環境変数）**: Terraform を実行する**同じターミナル**で 3 つの環境変数を**再 export**
5. 疎通確認: `aws sts get-caller-identity`

> 💡 **習慣化推奨**: `terraform apply` の直前に `aws sts get-caller-identity` を実行して失効を事前検知する。失効したまま `apply` を途中まで走らせると、中途半端にリソースが作られた状態で止まり、復旧が面倒になります。

---

## 3. Terraform 実行時エラー

### 症状 3-1: `terraform init` が失敗する

**症状例**
- `Failed to install provider`
- `Backend configuration changed`

**切り分け手順**

1. ネットワーク接続・プロキシを確認（→ [症状 1-3](#症状-1-3-社内プロキシ環境でダウンロードinit-が通らない)）
2. バックエンド設定変更が原因の場合は `-reconfigure` 付きで再実行
   ```
   terraform init -reconfigure
   ```
3. ローカルの `.terraform/` ディレクトリを削除して再初期化
   - Windows: `Remove-Item -Recurse -Force .terraform`
   - macOS: `rm -rf .terraform`

---

### 症状 3-2: `Error acquiring the state lock`

**症状例**
```
Error: Error acquiring the state lock
Lock Info:
  ID:        xxxxxxxx-xxxx-...
  Who:       user@host
```

**切り分け手順**

1. **他の人が同時に作業していないか**を確認（まずこれ）
2. 直前に自分の `apply` が異常終了していないかを確認
3. エラー表示にある `ID` と `Who` を控える

**解決策**

- 他者が作業中 → 完了を待つ。勝手にロック解除しない。
- 自分の異常終了が原因だと確信できる場合のみ:
  ```
  terraform force-unlock <ID>
  ```

---

### 症状 3-3: `plan` では差分が無いのに `apply` でエラー

**症状例**
- plan は成功、apply で `ResourceAlreadyExists` や `DependencyViolation`
- 他の作業者が同じ環境を同時に操作している兆候

**切り分け手順**

1. 直前に他の人が `apply` していないか確認
2. `terraform refresh` で state を実環境と同期してから再度 `plan`

**解決策**

- state のドリフト（手動変更）が疑われる場合は、まず AWS マネジメントコンソールで該当リソースの状態を確認。
- 手動変更があった場合は、state を再 import するか、該当部分を Terraform コードに反映。

---

### 症状 3-4: 既存リソースと名前が衝突する (`AlreadyExists` 系)

**症状例**
- `BucketAlreadyExists` (S3 はグローバル一意)
- `EntityAlreadyExists` (IAM)

**切り分け手順**

1. エラーで示されたリソース名を控える
2. AWS 上に既に存在しないか確認（別アカウントや別リージョンに作られていないか）

**解決策**

- 命名規則にプレフィックス（環境名・個人識別子）を加えて再実行。
- 既存リソースを Terraform 管理下に取り込む場合は `terraform import`。

---

## 🆘 それでも解決しないとき

問い合わせの際は以下を添えると解決が早いです：

1. 実行した**コマンド全文**
2. **エラーメッセージ全文**（スクリーンショット不可、テキストで）
3. `terraform version` の出力
4. `aws sts get-caller-identity` の出力（アカウント ID / ARN を含む）
5. OS とシェルの種類（例: Windows 11 + PowerShell 7）
