# データ処理規約 (polars優先 / pandas補完 / numpy)

新規実装は polars を優先する。pandas は既存資産との互換や pandas 専用ライブラリとの連携が必要な場合に限定して使う。

## ベクトル化優先

行ループ・`apply` より、ベクトル演算・組み込みメソッドを使う。

```python
# 誤: 行ループ
for row in df.iter_rows(named=True):
    ...

# 正: ベクトル化 (polars)
df = df.with_columns((pl.col("close") / pl.col("prev_close") - 1).alias("ret"))
```

pandas補完時:

```python
df["ret"] = df["close"] / df["prev_close"] - 1
```

## コピーとビュー

polars の DataFrame は不変な操作(`with_columns` / `filter` など)を基本とするため、pandasの `SettingWithCopyWarning` のような問題は発生しにくい。常に再代入で書く。

```python
sub = df.filter(pl.col("sector") == "Tech").with_columns(pl.lit(0.1).alias("weight"))
```

pandas補完時は、スライスへの代入をせず `.loc` / `.iloc` で明示し、サブセットを保持して加工する場合は `.copy()` する。

```python
# 正
sub = df.loc[df["sector"] == "Tech"].copy()
sub["weight"] = 0.1
```

## dtype(スキーマ)の明示

- 読み込み時にスキーマを指定する。特に日付は明示的に変換し、文字列日付のまま処理しない
- ティッカー・セクター等の低カーディナリティ列は `pl.Categorical` 型を検討する
- 意図しない型・暗黙の型昇格(int→float)に注意し、読み込み直後に `df.schema` を検証する

```python
df = pl.scan_csv(
    path,
    schema_overrides={"ticker": pl.Categorical, "volume": pl.Int64},
    try_parse_dates=True,
).collect()
```

pandas補完時:

```python
df = pd.read_csv(
    path,
    dtype={"ticker": "string", "volume": "int64"},
    parse_dates=["date"],
)
```

## 欠損値(null)の扱い

- 暗黙の `fill_null(0)` / `drop_nulls()` を禁止する。欠損の意味(休場・上場前・データ欠落)を確認し、処理方針をコメントで明記する
- 欠損補完(forward fill 等)は「その時点で観測可能だったか」を必ず考える(→ 08-financial-safety.md のルックアヘッド防止)

```python
# 休場日の価格は直前営業日の値を引き継ぐ(最大5営業日まで)
prices = prices.with_columns(pl.col("close").fill_null(strategy="forward", limit=5))
```

pandas補完時は `skipna=True` がデフォルトで NaN をスキップする点に注意し、無視してよいかを都度判断する。

```python
prices = prices.ffill(limit=5)
```

## 結合の整合性

- 複数の DataFrame を結合する前にキー列の整合を検証する
- `join` 後は行数を検証する(意図しない重複結合・欠落の検知)

```python
merged = prices.join(factors, on=["date", "ticker"], how="left", validate="1:1")
assert merged.height == prices.height, "結合で行数が変化した"
```

pandas補完時は `merge` の自動アラインメントによる暗黙の NaN 発生に注意する。

```python
merged = prices.merge(factors, on=["date", "ticker"], how="left", validate="one_to_one")
assert len(merged) == len(prices), "結合で行数が変化した"
```

## 大規模データ

- lazy API (`pl.scan_csv` / `pl.scan_parquet` + `.collect()`) を使い、必要な列・期間だけ読み込む(述語・射影プッシュダウン)
- 保存形式は CSV でなく parquet を標準とする(型保持・圧縮・高速)
- pandas でしか対応していない処理・ライブラリ連携がある場合のみ pandas を使う。その際も数GB超のデータでは避ける

## in-place 操作について

全体原則(01-general.md)のイミュータビリティに従い、`inplace=True` は使わない。メソッドチェーンまたは再代入で書く。

```python
df = df.sort("date")
```

pandas補完時:

```python
df = df.sort_values("date").reset_index(drop=True)
```
