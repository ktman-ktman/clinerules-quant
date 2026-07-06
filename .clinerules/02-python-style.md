# Python スタイル規約

## 基本

- PEP 8 に準拠する
- すべての関数シグネチャ(引数・戻り値)に型アノテーションを付ける
- `print()` を本番コードに残さない。`logging` を使う

ログレベルは意味に応じて使い分ける:

| レベル | 用途 |
|---|---|
| `debug` | 開発時のみ必要な詳細情報(中間計算値など) |
| `info` | 正常系の重要イベント(バックテスト開始/終了、データ読み込み件数など) |
| `warning` | 想定範囲内だが注意が必要な状態(欠損補完の実行、フォールバック処理など) |
| `error` | 処理を継続できない異常(例外送出前後) |

f-string ではなく `%s` 形式で遅延評価する(ログレベルが無効な場合に文字列整形コストを払わない)。

```python
import logging

logger = logging.getLogger(__name__)

def compute_sharpe(returns: "pl.Series", risk_free: float = 0.0) -> float:
    logger.debug("Sharpe計算: n=%d", len(returns))  # 正: 遅延評価
    # logger.debug(f"Sharpe計算: n={len(returns)}")  # 誤: 常に文字列整形が走る
    ...
```

## Docstring

- PEP 257 に準拠する。公開モジュール・クラス・関数・メソッドには docstring を書く(`_`prefixのprivate関数は省略可)
- 1行目は簡潔な要約とし、必要なら空行を挟んで詳細説明を続ける
- Google スタイル形式(`Args`, `Returns`, `Raises`, `Yields` セクション)を使う
- docstring 欠如は ruff の `D` ルール群(下記「ツールチェーン」節)により CI で自動的に検出・強制される

```python
def compute_sharpe(returns: "pl.Series", risk_free: float = 0.0) -> float:
    """年率シャープレシオを計算する。

    Args:
        returns: 日次リターン系列。
        risk_free: 無リスク金利(年率)。デフォルトは0。

    Returns:
        年率換算されたシャープレシオ。

    Raises:
        ValueError: returns が空の場合。
    """
    ...
```

## ツールチェーン

| 用途 | ツール |
|---|---|
| フォーマット | black |
| import整列 | isort |
| リント | ruff(docstring規約チェックの `D` ルール群を含む) |
| 型チェック | ty |
| セキュリティ静的解析 | bandit |

`ruff` で docstring 規約(PEP 257 / Google スタイル)もあわせてチェックする。`pyproject.toml` に以下を設定する:

```toml
[tool.ruff.lint]
select = ["E", "F", "D"]  # 既存のルールに加えて D (pydocstyle) を有効化

[tool.ruff.lint.pydocstyle]
convention = "google"
```

コミット前に必ず実行する: `black . && isort . && ruff check . && ty check src/`

## CLI

- CLIエントリポイントは `argparse` ではなく `click` を使う
- サブコマンドは `click.group()` / `@cli.command()` で構成する
- オプション・引数には型を指定する(`click.option(..., type=...)`)
- テストは `click.testing.CliRunner` を使う

```python
import click

@click.group()
def cli() -> None:
    """quantツールのエントリポイント。"""


@cli.command()
@click.option("--start-date", required=True, type=str, help="バックテスト開始日")
@click.option("--end-date", required=True, type=str, help="バックテスト終了日")
def backtest(start_date: str, end_date: str) -> None:
    """バックテストを実行する。"""
    ...
```

## パス操作

- ファイルパス操作は `os.path` ではなく `pathlib.Path` を使う
- パス結合は `/` 演算子を使う
- 型アノテーションでもパスは `Path` を使う(`str` ではなく)

```python
from pathlib import Path

def load_config(base_dir: Path) -> "pl.DataFrame":
    config_path = base_dir / "config" / "settings.yaml"
    if not config_path.exists():
        raise FileNotFoundError(f"設定ファイルが見つかりません: {config_path}")
    ...
```

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
    def get_prices(self, tickers: list[str], start: str, end: str) -> "pl.DataFrame": ...

def load_returns(source: PriceSource, tickers: list[str]) -> "pl.DataFrame":
    ...  # 具象クラスに依存しない
```

### コンテキストマネージャでリソース管理

ファイル・DB接続・一時的な状態変更は `with` で確実に解放する。

```python
with engine.connect() as conn:
    df = pl.read_database(query, connection=conn)
```

### ジェネレータで遅延評価

大きなデータセットの逐次処理はリストを作らずジェネレータで。

```python
def iter_trade_dates(calendar: "pl.Series") -> "Iterator[date]":
    yield from calendar
```

## 禁止事項

- ミュータブルなデフォルト引数(`def f(x, cache={})`)
- ベアexcept(`except:`)、`except Exception: pass`
- グローバル可変状態(モジュールレベルの可変変数への書き込み)
- 型アノテーションなしの公開関数
- 本番コードにおける `print()` の残存
- 公開モジュール・クラス・関数・メソッドの docstring 欠如
