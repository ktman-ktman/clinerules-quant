# テスト規約

## 基本方針

- テストフレームワークは pytest
- カバレッジ目標: 80% 以上
- 新機能・バグ修正は「テストを先に書く」(RED → GREEN → REFACTOR)
- テストが落ちたら、原則として実装を直す。テストを直すのは仕様が間違っている場合のみ

```bash
pytest --cov=src --cov-report=term-missing
```

## テスト構造 (AAA パターン)

Arrange(準備)- Act(実行)- Assert(検証)を明確に分ける。

```python
def test_sharpe_is_zero_for_constant_returns():
    # Arrange
    returns = pl.Series([0.01] * 252)

    # Act
    sharpe = compute_sharpe(returns)

    # Assert
    assert sharpe == pytest.approx(0.0, abs=1e-12)
```

## テスト命名

テスト対象の「振る舞い」を説明する名前にする。

```python
def test_returns_empty_frame_when_universe_is_empty(): ...
def test_raises_value_error_when_end_before_start(): ...
def test_falls_back_to_close_price_when_vwap_missing(): ...
```

## テスト分類

`pytest.mark` で unit / integration を分類し、CI で使い分ける。

```python
import pytest

@pytest.mark.unit
def test_zscore_normalization(): ...

@pytest.mark.integration
def test_loads_prices_from_database(): ...
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = ["unit: 高速な単体テスト", "integration: 外部依存を伴うテスト"]
```

## クオンツ特有のテスト観点

- 金融計算(リターン、リスク指標、ウェイト)は既知の手計算値・参照実装と突き合わせる
- 境界条件を必ずテストする: 空データ、1行データ、全NaN、単一銘柄
- float の比較は `pytest.approx` を使い、許容誤差を明示する(→ 09-numerical-precision.md)
- 乱数を使うテストはシードを固定する(→ 07-reproducibility.md)

## テストの独立性

- テスト間で状態を共有しない(実行順に依存しない)
- 外部依存(DB、API)は fixture でモック化し、integration テストのみ実接続する
- 一時ファイルは `tmp_path` fixture を使う
