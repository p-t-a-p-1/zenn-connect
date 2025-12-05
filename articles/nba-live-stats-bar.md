---
title: "MacのメニューバーにNBAの試合経過を表示する (Python + SwiftBar)"
emoji: "🏀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "uv", "swiftbar", "nba", "mac"]
published: false
---

## はじめに

NBAシーズン真っ只中ですね！🏀
推しのチームの試合経過、気になりませんか？私は気になります。

でも、仕事中にブラウザでスコアボードを常時表示しておくわけにもいかないし、スマホをチラチラ見るのも集中力が削がれます。
「**Macのメニューバーにさりげなく、でもリアルタイムにスコアを表示したい**」
そう思って、自分専用のNBA速報ツールを作ってみました。

今回は、**SwiftBar** と **uv (Python)** を組み合わせて、爆速で自分好みのメニューバーアプリを作る方法を紹介します。

## 作ったもの：NBA Live Stats Bar

Macのメニューバーに、現在行われている試合のスコアを表示するプラグインです。

https://github.com/p-t-a-p-1/nba-live-stats-bar



![進行中の試合がない場合は結果が表示されます](/images/nba-live-stats-bar/demo1.png)
*進行中の試合がない場合は結果が表示されます*

*メニューバーに `🏀 LAL 110-108 GSW (Q4 2:00)` のように表示されます*

クリックするとドロップダウンで以下の情報が見れます。
- 当日の全試合のスコアとステータス
- 進行中の試合の直近のPlay-by-Play（得点経過など）
- 試合開始前の予定時刻

## 技術スタック：uv × SwiftBar が最高すぎる

このツールの裏側は非常にシンプルですが、**依存関係の管理**に少し工夫があります。

- **SwiftBar**: シェルスクリプトやプログラムの標準出力をメニューバーに表示してくれるmacOSアプリ。
- **Python**: データ取得と整形ロジック。
- **uv**: 高速なPythonパッケージマネージャー。**ここがポイントです。**

### なぜ uv なのか？

通常、Pythonで外部ライブラリ（今回は `nba_api`）を使うスクリプトを配布・実行しようとすると、環境構築が面倒です。
`venv` を作って、`pip install` して、SwiftBarからその `venv` のPythonを指定して...となると、手軽さが失われます。

そこで、**uv** の [PEP 723 (Inline Script Metadata)](https://peps.python.org/pep-0723/) サポートを活用しました。

スクリプトの冒頭にこう書くだけです：

```python
#!/opt/homebrew/bin/uv run
# /// script
# requires-python = ">=3.11"
# dependencies = [
#     "nba-api",
# ]
# ///
```

これだけで、スクリプト実行時に `uv` が自動的に：
1. 必要なPythonバージョンを用意し
2. 依存ライブラリ（`nba-api`）をインストールした一時的な仮想環境を作成し
3. スクリプトを実行

してくれます。**ユーザーは `uv` さえ入っていれば、`pip install` すら不要です。**
SwiftBarのプラグインフォルダにスクリプトをポンと置くだけで動く。この「ゼロコンフィグ感」が最高に気持ちいいです。

## 実装のポイント

### 1. データの取得とキャッシュ

NBAのデータは `nba_api` という便利なライブラリ経由で取得していますが、APIへの過剰なアクセスを防ぐため、またオフライン時やAPIエラー時にも表示が消えないように、簡易的なキャッシュ機能を実装しています。

```python
# キャッシュが有効（5分以内）ならそれを表示、なければAPI取得
def main():
    data = get_scoreboard_data()
    
    if data is None:
        data = load_cache()
        if data is None:
            print("🏀 Error | color=red")
            return
    else:
        save_cache(data)
    # ...表示処理
```

### 2. 優先順位付き表示

複数の試合が同時に行われている場合、メニューバーのスペースは限られているので、**「最も接戦」または「終了間際」の試合**を優先的に表示するようにしています。
（推しのチームをハードコードで優先してもいいですが、汎用性を持たせました）

### 3. Play-by-Play の表示

ドロップダウン内では、進行中の試合について「誰が得点したか」などの詳細（Play-by-Play）も見れるようにしました。
これもAPIから取得したデータを、SwiftBarのサブメニュー形式（`--` プレフィックス）で出力するだけです。

```python
# SwiftBarのフォーマットに合わせて出力
print(f"--{time_str} {desc} | font=Monaco size=11")
```

## 使い方

もし使ってみたい方がいれば、以下の手順で導入できます。

1. **SwiftBar** をインストール (`brew install --cask swiftbar`)
2. **uv** をインストール (`curl -LsSf https://astral.sh/uv/install.sh | sh`)
3. リポジトリの `nba_uv.1m.py` を SwiftBar のプラグインフォルダにコピー
4. 実行権限を付与 (`chmod +x ...`)

これだけです！

## まとめ

**SwiftBar** と **uv** を組み合わせると、依存ライブラリを含むPythonスクリプトを、まるで単体バイナリのように手軽に扱えるようになります。
自分専用のちょっとしたツールを作るハードルがグッと下がるので、ぜひ試してみてください。

そしてNBAファンの皆さん、良きNBAライフを！🏀

---
GitHubリポジトリはこちら：
https://github.com/hakozaki-k/nba-live-stats-bar
