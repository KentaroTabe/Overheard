# 導入・確認チェックリスト(計画を進める前に手元で行うこと)

- 作成: 2026-10-30。根拠は `setup-facts.md`(一次情報の出典付き)と、この Mac での実測。
- 前提: iPhone SE 第 3 世代(iOS 26.5)、AirPods 4(ANC 付き)、Mac で開発、配布なし、口語化は Claude Haiku 5.5、TTS は Apple 内蔵。

## 0. 実測済み(この Mac、2026-10-30)

| 項目 | 結果 | 判定 |
|---|---|---|
| macOS | 26.5.1 | iOS 26 SDK でのビルドに十分。Xcode 27(iOS 27 SDK)には 26.6 以降が要る |
| Xcode | 26.6(iOS 26.5 SDK、iOS 26.5 / 26.2 シミュレータ) | そのまま使える |
| iPhone SE | Xcode のデバイス一覧に「iPhoneSE (26.5)」として登録済み(確認時はオフライン) | ペアリング済み。iOS 26.5 なので Xcode 26.6 から実機インストール可 |
| AirPods | Mac にペアリング済みの AirPods を検出 | 型番の照合は iPhone 側で行う |
| CLI | gh 2.103.2、Homebrew、node、python3 | 問題なし |

## 1. 導入するもの

| # | 内容 | 手順 | 備考 |
|---|---|---|---|
| 1 | Apple Account を Xcode に登録 | Xcode > Settings > Accounts > + | Personal Team(無料)ができる。有料プログラムは不要(§3 参照) |
| 2 | iPhone の Developer Mode | 設定 > プライバシーとセキュリティ > デベロッパモード → オン → 再起動 → 「有効にする」 | Mac とペアリング後に項目が出る。既に有効なら不要 |
| 3 | Anthropic Console のアカウントとクレジット | console.anthropic.com → Billing でクレジット購入(前払い、1 年で失効、返金なし)→ 本アプリ専用ワークスペース → 月の上限(spend limit)を $50 に設定 → そのワークスペース限定の API キーを有効期限付きで発行 | 最小購入額は購入画面で確認(第三者情報では $5)。キーはチャットやリポジトリに貼らない |
| 4 | premium / enhanced 音声(英・中・仏) | 設定 > アクセシビリティ > リーダーと読み上げ > 声 → 言語 → 声を選んでダウンロード | 1 つ 100MB 以上のことがある。Wi-Fi 推奨。en-US / en-GB / fr-FR / zh-CN / zh-TW に最上位品質の声があるとされる(二次情報)。試聴して品質を判断する |
| 5 | Google 翻訳アプリ、ChatGPT アプリ | App Store | どちらも iOS 18.0 以降。既存サービスの実機試験(01 第 5 章 試験 B・C)に使う |

SpeechTranscriber の日本語アセット(ja-JP)は、ユーザーが設定で入れるものではなく、アプリが `AssetInventory` でダウンロードする。導入作業は不要(PoC アプリ側で実装する)。

## 2. 確認するもの

| # | 内容 | 場所 | 何が分かるか |
|---|---|---|---|
| 1 | iPhone の iOS バージョンが 26.5 のままか | 設定 > 一般 > 情報 | Xcode に 26.5 で登録されている。iOS 27 に上げると Xcode 26.6 からインストールできるかが未確認になるので、**PoC 中は iOS 27 に上げない** |
| 2 | AirPods の型番とファームウェア | 設定 > Bluetooth > AirPods の (i) > 情報 | 型番 A3055〜A3057 なら AirPods 4(ANC)。最新ファームウェアは 9A348。高音質録音(iOS 26)の対象機種 |
| 3 | Developer Mode の有無 | 設定 > プライバシーとセキュリティ | 項目が無ければ Mac にケーブル接続してから再確認 |
| 4 | ChatGPT Voice が Free プランで 10 分使えるか | ChatGPT アプリ > 音声モード | 公式は「Limited access」としか書いていない。足りなければ試験 C は短縮するか省略 |
| 5 | Anthropic のクレジット最小額と試用クレジットの有無 | Console の Billing 画面 | 公式ページに記載なし |

## 3. 判断するもの

| # | 判断 | 選択肢と材料 | 推奨 |
|---|---|---|---|
| 1 | Apple Developer Program(年額)に入るか | 入らない: 署名の有効期限 7 日(7 日ごとに Xcode から再インストール)、1 台 3 アプリまで、App Attest / Push / iCloud 不可。本プロジェクトに要る Background Modes(audio)、マイク、音声認識は無料で可 | **入らない**で始める。常用して 7 日ごとの再インストールが面倒になったら検討 |
| 2 | Claude の API キーの置き方 | 案 1: アプリに埋め込み、ワークスペースの spend limit と有効期限で被害を抑える。案 2: 中継サーバーまたは App Attest(有料プログラム + iOS 27 + Xcode 27 が前提) | 配布しない個人利用なので**案 1**。キーは Keychain に入れ、ソースにはハードコードしない(設定画面から入力) |
| 3 | iOS 26 SDK(現環境)で進めるか、Xcode 27 に移るか | Xcode 27 には macOS 26.6 以降が要る。iOS 26 SDK で SpeechTranscriber、Translation、AVSpeechSynthesizer、Background Modes はすべて使える | **iOS 26 SDK のまま**進める |

## 4. この後の流れ

1. §1 の 1〜5 と §2 の 1〜3 を済ませる(所要 30〜60 分、Wi-Fi 環境)。
2. 既存サービスの実機試験(01 第 5 章の試験 B・C、02 の P0)を行い、集音距離・遅延・ロック中の挙動を記録する(費用ゼロ)。
3. 結果を受けて PoC の計画(`docs/planning/`)を立て、最小アプリの実装をサブエージェントに委託する。
