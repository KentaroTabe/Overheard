# Overheard

周囲で流れている雑談(ざわめき)を、学習したい言語のラフな口語にリアルタイム翻訳して
AirPods で聞き流すためのシステム。

- 入力: 周囲の雑談(自分の発話ではない)。出力: 学習対象言語の音声。入出力とも AirPods。
- 聞き流し用途。遅延の小ささと連続稼働を優先し、訳文の文法的な正しさは重視しない。
- 文体はラフな口語(現地で実際に使われる言い回し)。

## 現在の状態

調査フェーズ(2026-10-30 開始)。実装はまだ無い。

## ドキュメント

- `docs/research/00-summary.md` … 3 視点の比較と結論
- `docs/research/01-existing-services.md` … そのまま使える既存サービスはあるか
- `docs/research/02-combined-services.md` … 既存サービスの組み合わせで作るならどう切り分けるか
- `docs/research/03-from-scratch.md` … ゼロから実装するならどう作るか
- `docs/research/prior-research.txt` … 調査開始前に別の AI が出した先行調査(出典未検証。上記レポートで検証済み)

## 運用

- `main` は PR 経由で更新する。マージは開発者が行う。

## 名称について

プロジェクト名は Overheard(2026-10-30 に Zawamimi から改名)。GitHub リポジトリ名とローカルフォルダ名は
`Zawamimi` のまま(改名は開発者判断)。
