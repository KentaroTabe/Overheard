# 導入・確認チェックリスト(計画を進める前に手元で行うこと)

- 作成: 2026-10-30、同日改訂(ユーザー回答: Apple Developer Program 加入済み、API キー取得済み、iOS 26 系のまま、iOS 26 SDK で進める)。根拠は `setup-facts.md`(一次情報の出典付き)と、この Mac での実測。
- 前提: iPhone SE 第 3 世代(iOS 26 系)、AirPods 4(ANC 付き)、Mac で開発、Apple Developer Program 加入済み(他アプリで利用中)、Anthropic API キー取得済み、配布なし、口語化は Claude Haiku 5.5、TTS は Apple 内蔵。

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
| 1 | (済)Apple Developer Program と Xcode のアカウント登録 | 加入済みで他アプリの開発に使用中 | 署名の 7 日制限・3 アプリ制限は無い。本プロジェクトの capability(Background Modes の audio、マイク、音声認識)は追加申請不要 |
| 2 | (要確認)iPhone の Developer Mode | 設定 > プライバシーとセキュリティ > デベロッパモード | 他アプリの実機デバッグで有効化済みの可能性が高い。§2 で確認 |
| 3 | (済)Anthropic API キー | 取得済み | **入力先は PoC の段階で提示する**(§1.1)。それまでチャットやリポジトリに貼らない。Console 側で月の上限(spend limit)を $50 に設定しておくことを勧める(§2) |
| 4 | premium / enhanced 音声(英・中・仏) | 設定 > アクセシビリティ > リーダーと読み上げ > 声 → 言語 → 声を選んでダウンロード | 1 つ 100MB 以上のことがある。Wi-Fi 推奨。en-US / en-GB / fr-FR / zh-CN / zh-TW に最上位品質の声があるとされる(二次情報)。試聴して品質を判断する |
| 5 | Google 翻訳アプリ、ChatGPT アプリ | App Store | どちらも iOS 18.0 以降。既存サービスの実機試験(01 第 5 章 試験 B・C)に使う |

### 1.1 API キーの入力先(予定)

| 段階 | 入力先 | 扱い |
|---|---|---|
| Mac 上の PoC(ブラウザ + 小さなサーバー、Pipecat 等) | リポジトリ直下の `.env`(`ANTHROPIC_API_KEY=...`)。`.gitignore` 済み | 雛形 `.env.example` を用意したときに案内する。コミットしない |
| iOS アプリ | アプリ内の設定画面で入力 → Keychain に保存 | ソースにハードコードしない。Developer Program 加入済みなので、必要になれば App Attest + 中継サーバーに移せる(個人利用の間は不要) |

SpeechTranscriber の日本語アセット(ja-JP)は、ユーザーが設定で入れるものではなく、アプリが `AssetInventory` でダウンロードする。導入作業は不要(PoC アプリ側で実装する)。

## 2. 確認するもの

| # | 内容 | 場所 | 何が分かるか |
|---|---|---|---|
| 1 | (済)iPhone の iOS は 26 系 | ユーザー回答 | Xcode には 26.5 で登録されている。**PoC 中は iOS 27 に上げない**(Xcode 26.6 からインストールできるか未確認のため) |
| 2 | (済)AirPods の型番は A3055〜A3057 = AirPods 4(ANC) | ユーザー回答 | ファームウェアだけ確認(設定 > Bluetooth > (i) > 情報。最新は 9A348)。高音質録音(iOS 26)の対象機種 |
| 3 | Developer Mode が有効か | 設定 > プライバシーとセキュリティ | 他アプリで実機デバッグ済みなら有効。項目が無ければ Mac にケーブル接続してから再確認 |
| 4 | ChatGPT Voice の利用枠(契約プラン次第) | ChatGPT アプリ > 音声モード | Free は「Limited access」(数値なし)、Plus は 3 時間/日。10 分の試験に足りるか |
| 5 | Anthropic Console の月の上限(spend limit) | Console > ワークスペースの Limits | 既存キーのワークスペースに $50 の上限が無ければ設定する。他プロジェクトとキーを共有している場合は、本アプリ専用ワークスペースとキーに分けるのを勧める(使用量を分けて見るため) |

## 3. 判断するもの

| # | 判断 | 選択肢と材料 | 推奨 |
|---|---|---|---|
| 1 | (済)Apple Developer Program | 加入済み | — |
| 2 | (済)Claude の API キーの置き方 | 個人利用なので、Mac の PoC は `.env`、iOS アプリは設定画面から Keychain。中継サーバーと App Attest は配布するときに再検討 | 決定(2026-10-30) |
| 3 | (済)iOS 26 SDK(現環境)で進める | Xcode 27 には macOS 26.6 以降が要る。iOS 26 SDK で SpeechTranscriber、Translation、AVSpeechSynthesizer、Background Modes はすべて使える | 決定(2026-10-30) |
| 4 | Anthropic のワークスペースを本アプリ専用に分けるか | 分ける: 使用量と上限を本アプリ単独で管理できる。分けない: 既存キーをそのまま使える | 分けるのを勧める(他プロジェクトと使用量が混ざらない) |

## 4. この後の流れ

1. §1 の 4〜5(音声のダウンロード、試験用アプリ)と §2 の 3〜5 を済ませる(所要 15〜30 分、Wi-Fi 環境)。
2. 既存サービスの実機試験(01 第 5 章の試験 B・C、02 の P0)を行い、集音距離・遅延・ロック中の挙動を記録する(費用ゼロ)。
3. 結果を受けて PoC の計画(`docs/planning/`)を立て、最小アプリの実装をサブエージェントに委託する。
