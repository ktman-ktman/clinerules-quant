# セキュリティ規約

## シークレット管理

- APIキー・パスワード・トークン・DB接続文字列をソースコードにハードコードしない
- 環境変数または秘密管理サービスから読み込む(ローカルは `.env`、`.env` は必ず `.gitignore` に入れる)
- 必須シークレットは起動時に検証し、欠けていれば即座に失敗させる
- 読み込みは `pydantic-settings` の `BaseSettings` に統一する。型アノテーションから自動でパース・バリデーションされ、必須項目の欠落は起動時に `ValidationError` として検出できる。`os.environ.get()` を直接呼ぶ箇所は増やさず、設定オブジェクト経由でアクセスする

```python
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    bloomberg_api_key: str = Field(...)

settings = Settings()  # 必須項目が欠けていれば起動時に ValidationError
```

## SQL

- 文字列連結・f-string でクエリを組み立てない。必ずパラメータ化する

```python
# 誤
query = f"SELECT * FROM prices WHERE ticker = '{ticker}'"

# 正: バインドパラメータはドライバ/SQLAlchemy側で解決させる
stmt = sqlalchemy.text("SELECT * FROM prices WHERE ticker = :ticker")
df = pl.read_database(stmt, connection=conn, execute_options={"parameters": {"ticker": ticker}})
```

## 静的セキュリティスキャン

コミット前に bandit を実行する。

```bash
bandit -r src/
```

## エラーメッセージとログ

- エラーメッセージ・ログに機密情報(キー、接続文字列、顧客情報、ポジション詳細)を含めない
- 例外を上位へ伝播するとき、機密を含むコンテキストをマスクする

## その他

- 外部データ(ベンダーAPI応答、CSV、Excel)を信用せず、読み込み時にスキーマ・型を検証する
- `pickle` を信頼できないソースからロードしない(任意コード実行のリスク)。データ交換は parquet / CSV / JSON を使う
- `eval()` / `exec()` を使わない
- 誤ってコミットしたシークレットは「削除」でなく「ローテーション」する

## コミット前チェックリスト

- [ ] ハードコードされたシークレットがない
- [ ] SQL がパラメータ化されている
- [ ] bandit が警告なしで通る
- [ ] ログ・例外に機密が漏れていない
