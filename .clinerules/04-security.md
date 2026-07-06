# セキュリティ規約

## シークレット管理

- APIキー・パスワード・トークン・DB接続文字列をソースコードにハードコードしない
- 環境変数または秘密管理サービスから読み込む(ローカルは `.env` + python-dotenv、`.env` は必ず `.gitignore` に入れる)
- 必須シークレットは起動時に検証し、欠けていれば即座に失敗させる

```python
import os

def require_env(name: str) -> str:
    value = os.environ.get(name)
    if not value:
        raise RuntimeError(f"必須環境変数 {name} が設定されていません")
    return value

BLOOMBERG_API_KEY = require_env("BLOOMBERG_API_KEY")
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
