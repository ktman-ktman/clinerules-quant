# データ処理規約 (pandas / polars / numpy)

## ベクトル化優先

行ループ・`apply` より、ベクトル演算・組み込みメソッドを使う。

```python
# 誤: 行ループ
for i in range(len(df)):
    df.loc[i, "ret"] = df.loc[i, "close"] / df.loc[i, "prev_close"] - 1

# 正: ベクトル化
df["ret"] = df["close"] / df["prev_close"] - 1
```

## コピーとビュー (SettingWithCopy 回避)

- スライスへの代入をしない。代入は `.loc` / `.iloc` で明示する
- サブセットを保持して加工する場合は明示的に `.copy()` する

```python
# 誤: ビューへの代入(SettingWithCopyWarning)
sub = df[df["sector"] == "Tech"]
sub["weight"] = 0.1

# 正
sub = df.loc[df["sector"] == "Tech"].copy()
sub["weight"] = 0.1
```

## dtype の明示

- 読み込み時に dtype を指定する。特に日付は `parse_dates` / `pd.to_datetime` で明示的に変換し、文字列日付のまま処理しない
- ティッカー・セクター等の低カーディナリティ列は `category` 型を検討する
- 意図しない `object` 型・暗黙の型昇格(int→float)に注意し、読み込み直後に `df.dtypes` を検証する

```python
df = pd.read_csv(
    path,
    dtype={"ticker": "string", "volume": "int64"},
    parse_dates=["date"],
)
```

## 欠損値の扱い

- 暗黙の `fillna(0)` / `dropna()` を禁止する。欠損の意味(休場・上場前・データ欠落)を確認し、処理方針をコメントで明記する
- 欠損補完(ffill 等)は「その時点で観測可能だったか」を必ず考える(→ 08-financial-safety.md のルックアヘッド防止)

```python
# 休場日の価格は直前営業日の値を引き継ぐ(最大5営業日まで)
prices = prices.ffill(limit=5)
```

## index の整合性

- 複数の DataFrame を演算する前に index / columns の整合を検証する。pandas の自動アラインメントによる暗黙の NaN 発生に注意
- `merge` / `join` 後は行数を検証する(意図しない重複結合・欠落の検知)

```python
merged = prices.merge(factors, on=["date", "ticker"], how="left", validate="one_to_one")
assert len(merged) == len(prices), "結合で行数が変化した"
```

## 大規模データ

- 数GB超のデータは pandas ではなく polars(lazy API)を検討する
- 保存形式は CSV でなく parquet を標準とする(型保持・圧縮・高速)
- 必要な列・期間だけ読み込む(`columns=` / フィルタのプッシュダウン)

## in-place 操作について

全体原則(01-general.md)のイミュータビリティに従い、`inplace=True` は使わない。メソッドチェーンまたは再代入で書く。

```python
df = df.sort_values("date").reset_index(drop=True)
```
