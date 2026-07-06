# clinerules-quant

資産運用会社のクオンツアナリスト向け Cline ルールセット雛形(Python)。

[everything-claude-code (ECC)](https://github.com/affaan-m/ECC) の rules 体系(common + python 層)を日本語化・統合し、クオンツ業務固有の規約(データ処理・再現性・金融計算安全性・数値精度)を追加したもの。

## 使い方

1. `.clinerules/` ディレクトリをプロジェクトルートへコピーする:

   ```bash
   cp -r clinerules-quant/.clinerules /path/to/your-project/
   ```

2. Cline はプロジェクトルートの `.clinerules/` 内の全 Markdown ファイルを自動的に読み込む。

## 構成

| ファイル | 内容 |
|---|---|
| `01-general.md` | 全体原則(KISS/DRY/YAGNI、イミュータビリティ、命名、エラー処理) |
| `02-python-style.md` | PEP 8、型アノテーション、black/isort/ruff/mypy、推奨パターン |
| `03-testing.md` | pytest、カバレッジ80%、AAA構造、クオンツ特有のテスト観点 |
| `04-security.md` | シークレット管理、SQLパラメータ化、bandit |
| `05-git-workflow.md` | Conventional Commits、コミット前チェックリスト、notebook運用 |
| `06-data-handling.md` | pandas/polars/numpy 規約(ベクトル化、copy/view、dtype、欠損値) |
| `07-reproducibility.md` | 乱数シード、データバージョニング、notebook→モジュール化、環境固定 |
| `08-financial-safety.md` | ルックアヘッドバイアス、生存者バイアス、カレンダー、リターン計算規約 |
| `09-numerical-precision.md` | float比較、Decimal、NaN検知、ゼロ除算ガード |

## ルールの優先順位

番号が大きいファイルほどドメイン特化のルールであり、一般ルールと矛盾する場合は特化ルールが優先される(例: 06 の pandas 規約は 01 のイミュータビリティ原則を pandas 向けに具体化したもの)。

## カスタマイズ指針

- プロジェクトで使わないファイルは削除してよい(例: DB を使わないなら 04 の SQL 節)
- 社内固有の規約(データベンダー名、カレンダーライブラリ、年率換算日数など)は各ファイルの該当箇所を書き換える
- ファイルを追加する場合は `10-` 以降の番号を振り、1ファイル 100〜200 行程度に収める(Cline のコンテキスト消費を抑えるため)
