# 数値精度・検証規約

## float の等値比較禁止

浮動小数点の `==` 比較をしない。許容誤差を明示して比較する。

```python
import numpy as np
import pytest

# 誤
assert portfolio_weights.sum() == 1.0

# 正: 許容誤差を明示
assert np.isclose(portfolio_weights.sum(), 1.0, atol=1e-9)

# テストコードでは
assert sharpe == pytest.approx(1.234, abs=1e-6)
```

- 許容誤差(atol/rtol)は「なぜその値か」を説明できる根拠を持って選ぶ(計算の桁数・データの精度)

## 金額・数量の表現

- 会計的な正確性が必要な金額(約定金額、手数料、NAV)は `Decimal` または最小単位の整数(円、セント)で扱う。float で通貨を加減算しない
- 分析・統計計算(リターン、リスク指標)は float64 でよい。ただし出力時の丸め規約を統一する

```python
from decimal import Decimal

fee = Decimal("1234.56") * Decimal("0.001")   # 正: 文字列からDecimal
fee = Decimal(1234.56) * Decimal(0.001)       # 誤: floatを経由すると誤差が混入
```

## 累積誤差

- 多数の小さい値の合計は誤差が蓄積しうる。大規模な集計は numpy の合計(pairwise summation)や `math.fsum` を使い、素朴なループ加算を避ける
- 分散・標準偏差は自前実装せず numpy / pandas の実装を使う(数値的に安定なアルゴリズムが使われている)

## NaN / inf の伝播検知

- 演算後に NaN / inf が「想定どおりか」を検証する。黙って下流に流さない

```python
weights = signal / signal.abs().sum()
if not np.isfinite(weights).all():
    raise ValueError(f"ウェイトに非有限値: NaN={weights.isna().sum()}件")
```

- pandas の集計関数はデフォルトで NaN をスキップする(`skipna=True`)。NaN を「無視してよいか」を都度判断し、意図を明示する

## ゼロ除算・空データのガード

境界条件を明示的にガードする。numpy はゼロ除算で例外でなく inf/nan を返すため、事前チェックが必要。

```python
def compute_sharpe(returns: pd.Series, risk_free: float = 0.0) -> float:
    if len(returns) < 2:
        raise ValueError("リターン系列が短すぎる")
    vol = returns.std()
    if np.isclose(vol, 0.0):
        raise ValueError("ボラティリティがゼロ: Sharpe未定義")
    return (returns.mean() - risk_free) / vol * np.sqrt(252)
```

## 検証の習慣

- 金融指標の実装は、既知の入力に対する手計算値・参照実装(ベンダー値)との突き合わせテストを必ず書く
- 桁が変わる操作(bp ⇔ %、円 ⇔ 百万円)は変換関数を一箇所に定義し、インラインの `* 10000` を散在させない
