# リサーチ再現性規約

原則: **すべてのリサーチ結果は、後日・他人が同一環境で再現できなければならない。**

## 乱数シード

乱数を使う処理(ブートストラップ、モンテカルロ、機械学習)は必ずシードを固定し、シード値を設定として明示する。

```python
import random
import numpy as np

SEED = 42

def set_seed(seed: int = SEED) -> None:
    random.seed(seed)
    np.random.seed(seed)
    # 使用する場合: torch.manual_seed(seed), sklearn は random_state 引数で指定
```

- グローバルシードより、`np.random.default_rng(seed)` のようにジェネレータを明示的に渡す方式を優先する

## データのバージョン記録

- リサーチに使ったデータは「いつ・どこから・どの条件で」取得したかを記録する(取得日時、ソース、ユニバース定義、adjustments の有無)
- ポイントインタイムでないデータソース(遡及訂正されるデータ)は取得時点のスナップショットを parquet で保存する
- 結果を出力するとき、入力データのバージョン・期間・ユニバースをメタデータとして併記する

```python
metadata = {
    "data_source": "vendor_x",
    "snapshot_date": "2026-07-01",
    "universe": "TOPIX500",
    "period": ("2015-01-01", "2026-06-30"),
    "code_version": git_commit_hash(),
}
```

## Notebook の位置づけ

- notebook は「探索・可視化」専用。本番・共有されるロジックは `src/` 配下のモジュールに移し、notebook からは import して使う
- notebook にコピペされたロジックの二重管理を禁止する
- コミット前に出力セルをクリアする

## 環境の固定

- 依存関係は lock ファイルで固定する(uv / poetry / pip-tools のいずれか、プロジェクトで統一)
- 「自分のマシンでは動く」を排除するため、Python バージョンも `pyproject.toml` の `requires-python` で明示する

## パスと設定

- 絶対パス・個人環境のパスをコードにハードコードしない
- パス・期間・ユニバース等の実行パラメータは設定ファイル(YAML/TOML)または環境変数に外出しする

```python
# 誤
df = pl.read_parquet("/home/taro/data/prices.parquet")

# 正
df = pl.read_parquet(config.data_dir / "prices.parquet")
```
