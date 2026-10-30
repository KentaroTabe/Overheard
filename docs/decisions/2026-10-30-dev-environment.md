# 2026-10-30 開発環境の前提(ユーザー回答)

`docs/planning/setup-checklist.md` の「判断するもの」への回答。

| 項目 | 判断 | 影響 |
|---|---|---|
| Apple Developer Program | 加入済み(他アプリの開発で使用中) | 無料アカウントの制限(署名 7 日、1 台 3 アプリ、App Attest 不可)は無い。将来 App Attest + 中継サーバーに移す選択肢が残る |
| Anthropic API キー | 取得済み。入力先が提示されれば入力できる | Mac の PoC は `.env`(`.gitignore` 済み)、iOS アプリは設定画面から Keychain。ソースにハードコードしない。キーはチャット・リポジトリに貼らない |
| SDK | iOS 26 SDK(Xcode 26.6、macOS 26.5.1)のまま進める | Xcode 27 / macOS 26.6 への更新は不要 |
| iPhone の iOS | 26 系のまま | PoC 中は iOS 27 に上げない(Xcode 26.6 からのインストール可否が未確認) |
| Anthropic ワークスペース | 本アプリ専用に分けない(既存キーは他プロジェクトと共有していないため) | 既存キーとそのワークスペースの spend limit($50)をそのまま使う |
