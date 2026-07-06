# Python スタイル規約

## 基本

- PEP 8 に準拠する
- すべての関数シグネチャ(引数・戻り値)に型アノテーションを付ける
- `print()` を本番コードに残さない。`logging` を使う

```python
import logging

logger = logging.getLogger(__name__)

def compute_sharpe(returns: "pd.Series", risk_free: float = 0.0) -> float:
    logger.debug("Sharpe計算: n=%d", len(returns))
    ...
```

## ツールチェーン

| 用途 | ツール |
|---|---|
| フォーマット | black |
| import整列 | isort |
| リント | ruff |
| 型チェック | mypy(または pyright) |
| セキュリティ静的解析 | bandit |

コミット前に必ず実行する: `black . && isort . && ruff check . && mypy src/`

## イミュータブルなデータ構造

DTOや設定値には frozen dataclass / NamedTuple を使う。

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class BacktestConfig:
    start_date: str
    end_date: str
    universe: tuple[str, ...]   # listではなくtupleで不変に
    rebalance_freq: str = "M"
```

## 推奨パターン

### Protocol によるダックタイピング

```python
from typing import Protocol

class PriceSource(Protocol):
    def get_prices(self, tickers: list[str], start: str, end: str) -> "pd.DataFrame": ...

def load_returns(source: PriceSource, tickers: list[str]) -> "pd.DataFrame":
    ...  # 具象クラスに依存しない
```

### コンテキストマネージャでリソース管理

ファイル・DB接続・一時的な状態変更は `with` で確実に解放する。

```python
with engine.connect() as conn:
    df = pd.read_sql(query, conn)
```

### ジェネレータで遅延評価

大きなデータセットの逐次処理はリストを作らずジェネレータで。

```python
def iter_trade_dates(calendar: "pd.DatetimeIndex") -> "Iterator[pd.Timestamp]":
    yield from calendar
```

## 禁止事項

- ミュータブルなデフォルト引数(`def f(x, cache={})`)
- ベアexcept(`except:`)、`except Exception: pass`
- グローバル可変状態(モジュールレベルの可変変数への書き込み)
- 型アノテーションなしの公開関数
