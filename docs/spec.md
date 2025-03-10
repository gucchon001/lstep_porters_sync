# 相談フラグ管理システム 仕様書

## 概要

本システムは、特定の条件（フラグ）に一致するIDを検索し、それらのIDを相談転記先リストに更新するためのツールです。Google Spreadsheetを使用してデータを管理し、自動的に条件に合致するIDを抽出して転記先リストを更新します。さらに、抽出したIDのユーザー情報をCSVファイルとして保存し、PORTERSシステムにインポートする機能も備えています。

## 処理フローの概要

1. **相談フラグの確認と新規IDの抽出**
   - 相談フラグマスタ（M_相談フラグ）からフラグが1に設定されている項目を抽出
   - 現在の仕様では、「オンライン相談中」「LINE相談中」「求人紹介（電話）」などが相談フラグとして設定
   - 友達リストDLデータシートの"対応マーク"列を確認し、抽出したフラグ項目に一致するIDを特定
   - 既存の相談転記先リストに存在しないIDを新規IDとして抽出

2. **相談転記先リストの更新**
   - 抽出した新規IDと相談日（現在の日付）を相談転記先リスト（相談Raw）に追加
   - 同時にログシート（相談者一覧）にもIDを転記

3. **アンケートデータの取得とCSV出力**
   - 新しく追加されたIDのユーザーのアンケートDLデータからレコードを取得
   - 取得したレコードをCSVファイルとして保存（cp932エンコーディング）

4. **PORTERSへのデータインポート**
   - Seleniumを使用してPORTERSシステムにログイン
   - 求職者インポート機能を使用
   - 保存したCSVファイルをアップロード
   - LINE初回アンケート取込形式でインポート

## 詳細処理フロー

### 1. 相談フラグの確認と新規IDの抽出（src/main.py）

1. **環境設定の読み込み**
   - 環境変数とシート設定を読み込む

2. **スプレッドシートへの接続**
   - Google Sheets APIを使用してスプレッドシートに接続

3. **相談フラグマスタの読み込み**
   - M_相談フラグシートからデータを取得
   - フラグ列が1に設定されている項目を抽出

4. **友達リストの読み込み**
   - 友達リストDLデータシートからデータを取得
   - "対応マーク"列の値を確認し、抽出したフラグ項目に一致するIDを特定

5. **既存の相談転記先リストとの比較**
   - 相談転記先リスト（相談Raw）からデータを取得
   - 既に登録されているIDを除外し、新規IDのみを抽出

6. **相談転記先リストの更新**
   - 新規IDと相談日（現在の日付）を相談転記先リストに追加
   - 同時にログシート（相談者一覧）にもIDを転記

### 2. アンケートデータの取得とCSV出力（src/modules/anq_data/analyzer.py）

1. **環境設定の読み込み**
   - 環境変数とシート設定を読み込む

2. **スプレッドシートへの接続**
   - Google Sheets APIを使用してスプレッドシートに接続
   - settings シートからシート名マッピングを取得

3. **アンケートDLデータシートの読み込み**
   - アンケートDLデータシートからすべての値を取得
   - ヘッダー行を取得（1行目）
   - "回答者ID"列のインデックスを特定
   - 指定されたIDのレコードを検索

4. **データの加工と保存**
   - 取得したデータをDataFrameに変換
   - CSVファイルとして保存（cp932エンコーディング）
   - 最新のCSVファイルへのシンボリックリンク（またはコピー）を作成

### 3. PORTERSへのデータインポート（src/modules/porters/importer.py）

1. **ブラウザの初期化**
   - Seleniumを使用してChromeブラウザを起動
   - セレクタ情報の読み込み

2. **PORTERSへのログイン**
   - 環境変数からログイン情報を取得
   - ログインページにアクセス
   - 会社ID、ユーザー名、パスワードを入力してログイン
   - 二重ログインポップアップの処理

3. **求職者インポート機能へのアクセス**
   - メニューから「求職者のインポート」リンクを探索してクリック
   - ポップアップの表示を確認

4. **CSVファイルのアップロード**
   - 「添付」ボタンをクリック
   - CSVファイルを選択
   - 「LINE初回アンケート取込」形式を選択
   - 「次へ」ボタンをクリック

5. **インポート設定と実行**
   - インポート設定画面での必要な設定を行う
   - インポート処理を実行
   - 完了確認

6. **ログアウト**
   - 明示的なログアウト処理を実行
   - ブラウザを終了

## システム構成

### 主要コンポーネント

1. **メインモジュール** (`src/main.py`)
   - システム全体の実行を制御
   - 相談フラグの確認と新規IDの抽出、相談転記先リストの更新を担当

2. **アンケートデータ分析モジュール** (`src/modules/anq_data/analyzer.py`)
   - 新規IDのアンケートデータを取得
   - CSVファイルとして保存

3. **PORTERSインポートモジュール** (`src/modules/porters/importer.py`)
   - PORTERSへのログイン処理
   - ブラウザ操作の基本機能
   - CSVファイルのインポート処理

4. **フラグ検索モジュール** (`src/modules/consult/consult_flags.py`)
   - 相談フラグマスタから条件に一致するIDを検索

5. **転記リスト更新モジュール** (`src/modules/consult/transfer_list.py`)
   - 検索されたIDを相談転記先リストに更新

6. **設定管理モジュール** (`src/modules/common/settings.py`)
   - スプレッドシートの設定情報を管理

7. **スプレッドシート接続モジュール** (`src/modules/common/spreadsheet.py`)
   - Google Spreadsheetへの接続と基本操作を提供

8. **ユーティリティモジュール**
   - 環境変数管理 (`src/utils/environment.py`)
   - ログ設定 (`src/utils/logging_config.py`)

### データソース

- **Google Spreadsheet**
  - 友達リストDLデータ: ユーザー情報を格納
  - アンケートDLデータ: アンケート回答データを格納
  - 相談フラグマスタ: 相談条件のフラグを管理
  - 相談転記先リスト: 相談対象のIDと相談日を管理
  - ログシート: 相談者IDのログを記録

### 設定ファイル

- **環境変数ファイル** (`config/secrets.env`)
  - Google APIの認証情報
  - スプレッドシートID
  - PORTERSのログイン情報

- **設定ファイル** (`config/settings.ini`)
  - スプレッドシートの各シート名
  - 処理の設定値

## モジュール詳細

### 1. メインモジュール (`src/main.py`)

#### `main()`

**目的**: システム全体の実行を制御

**処理ステップ**:
1. コマンドライン引数の解析
2. 環境変数のロード
3. 実行する処理ブロックの決定
4. 各処理ブロックの実行
   - ブロック1: 相談フラグの確認と新規IDの抽出
   - ブロック2: アンケートデータの取得とCSV出力
   - ブロック3: PORTERSへのデータインポート

### 2. フラグ検索モジュール (`src/modules/consult/consult_flags.py`)

#### `find_ids_with_matching_flags()`

**目的**: 相談フラグマスタから条件に一致するIDを検索

**処理ステップ**:
1. スプレッドシートへの接続
2. シート設定の読み込み
3. 相談フラグマスタからデータ取得
4. フラグが1の項目を抽出
5. 友達リストからデータ取得
6. 対応マークとフラグ項目を比較して一致するIDを抽出
7. 結果の返却

### 3. 転記リスト更新モジュール (`src/modules/consult/transfer_list.py`)

#### `update_consult_transfer_list(matching_ids)`

**目的**: 相談転記先リストにマッチしたIDを追加

**処理ステップ**:
1. スプレッドシートへの接続
2. シート設定の読み込み
3. 相談転記先リストからデータ取得
4. 既存のIDと追加すべきIDを特定
5. 新しい行を作成して追加
6. ログシートにも転記

### 4. アンケートデータ分析モジュール (`src/modules/anq_data/analyzer.py`)

#### `analyze_anq_data(target_ids)`

**目的**: 指定されたIDのアンケートデータを取得してCSVファイルに保存

**処理ステップ**:
1. アンケートデータ分析クラスのインスタンスを作成
2. スプレッドシートに接続
3. アンケートデータを取得
4. CSVファイルに保存
5. 処理結果と保存したCSVファイルのパスを返す

#### `AnqDataAnalysis` クラス

**目的**: アンケートデータの取得と分析を行う

**主要メソッド**:
- `connect_to_spreadsheet()`: スプレッドシートに接続
- `get_anq_data()`: アンケートデータを取得
- `save_to_csv()`: データをCSVファイルとして保存

### 5. PORTERSインポートモジュール (`src/modules/porters/importer.py`)

#### `import_to_porters(csv_path)`

**目的**: CSVファイルのデータをPORTERSシステムにインポート

**処理ステップ**:
1. ブラウザの初期化
2. PORTERSへのログイン
3. 求職者インポート機能へのアクセス
4. CSVファイルのアップロード
5. インポート設定と実行
6. ログアウト

## エラー処理

- 各処理段階でエラーが発生した場合はログに詳細を記録
- 処理の成功/失敗を明確に表示し、失敗時はログを確認するよう促す
- 接続エラーやデータ不整合などの一般的なエラーに対応

## ログ

- ログファイルは `logs` ディレクトリに保存
- ログレベルは `INFO` 以上
- 各処理の開始と終了、重要なステップ、エラーをログに記録

## 注意事項

1. スプレッドシートの構造が変更された場合は、`settings.ini` ファイルの設定を更新する必要があります
2. PORTERSのインターフェースが変更された場合は、セレクタ情報を更新する必要があります
3. 大量のデータを処理する場合は、Google Sheets APIのクォータ制限に注意してください

bash
環境設定
python -m pip install -r requirements.txt
実行（全処理）
python -m src.main
特定のブロックのみ実行
python -m src.main --block 1 # 相談フラグの確認と新規IDの抽出
python -m src.main --block 2 # アンケートデータの取得とCSV出力
python -m src.main --block 3 # PORTERSへのデータインポート
特定のIDを指定して実行
python -m src.main --id 123456789