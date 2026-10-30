# 04. 実機(iPhone SE 第 3 世代 + AirPods)による制約と推奨構成の修正

- 調査日: 2026-10-30(本文中の出典の確認日もすべて 2026-10-30)
- 調査: サブエージェント(Opus)。前提はユーザー判断(`docs/decisions/2026-10-30-initial-constraints.md`): 学習言語は英・中・仏、周囲音のクラウド送信は可、Claude API は月 $50 まで、Mac 中継なし、配布なし。
- 凡例: **[一次]** Apple / Anthropic の公式ドキュメント・サポート・製品ページで確認 / **[二次]** 開発者フォーラムの利用者投稿・ブログのみ / **[推測]** 筆者の推論 / **[未確認]** 調べたが確認できない
- 実機での検証はしていない。API は呼んでいない(価格ページを読んだだけ)。

## 1. 結論

1. **iPhone SE(第 3 世代、A15)は iOS 26 にも iOS 27 にも対応する** [一次]。ただし **Apple Intelligence には非対応**(対象は iPhone 15 Pro 以降と iPhone 16 以降)[一次]。
2. **SE 第 3 世代で使えなくなるもの**: 端末内 Foundation Models、PCC(`PrivateCloudComputeLanguageModel`)、AirPods ライブ翻訳、Translation framework の `highFidelity`(指定しても自動で別方式になる)、Siri AI など Apple Intelligence 系の機能全般 [一次]。空間オーディオの個人プロファイル作成もできない(TrueDepth カメラが無いため)[一次]。
3. **使えるもの**: SpeechAnalyzer / `SpeechTranscriber`(ja)は**動くという報告がある**。A13 世代(iPhone 11、SE 第 2 世代)では使えないという報告があり、SE 第 3 世代は動作機種に挙がっている [二次]。動かなかった場合も `DictationTranscriber` で日本語を端末内で認識できる [一次+推測]。Translation framework の `lowLatency`(ja→en / zh / fr)、AVSpeechSynthesizer(enhanced / premium 音声のダウンロード)、AirPods の高音質録音(iPhone 側の条件は iOS 26 だけ)も使える [一次]。
4. **「AirPods 4 Pro」という製品は無い** [一次]。候補は AirPods 4(ANC 有/無)、AirPods Pro 2、AirPods Pro 3、AirPods 5(2026 年発売)。主入力は iPhone 内蔵マイクにする方針なので、どの機種でも推奨構成は変わらない。違いが出るのはマイクを比べる試験の範囲だけ。
5. **Claude の費用は Sonnet 5.5 でも収まる**: 02 の仮定で Haiku 5.5 は月 $0.86、Sonnet 5.5 は月 $17.28 [一次の単価で計算]。呼び出しを 2 倍にしても Sonnet は $34.56 で収まる。ただし「発話率 100% + 5 秒ごと + トークン 1.3 倍」が重なると $89.86 になり、超える。
6. **修正後の推奨構成**: iPhone SE 内蔵マイク → `SpeechTranscriber`(ja-JP。不可なら `DictationTranscriber`、それも不可ならクラウド STT)→ **Claude Haiku 5.5**(口語化。Sonnet 5.5 と聞き比べる)→ **AVSpeechSynthesizer の enhanced / premium 音声**($0。足りなければクラウド TTS)→ AirPods(A2DP、出力専用)。Translation framework の `lowLatency` は、オフライン時や Claude が使えないときの予備にする。
7. 03 の (b)「端末内 STT + PCC / 端末内 LLM」と (c)「iPhone 内で完結」は SE 第 3 世代では成り立たない。口語化の選択肢はクラウド LLM(Claude)だけになる。

## 2. 確認項目ごとの結果

### 2.1 iOS 26 / iOS 27 への対応

| 項目 | 結果 | 根拠 |
|---|---|---|
| iOS 26 | **対応**。Apple の iPhone ユーザガイドの対応機種一覧に「iPhone SE (3rd generation)」がある | [一次] https://support.apple.com/guide/iphone/iphone-models-compatible-with-ios-26-iphe3fa5df43/26/ios/26 (2026-10-30) |
| iOS 27 | **対応**。iOS 27 の製品ページの対応機種に「iPhone SE (2nd generation and later)」とある | [一次] https://www.apple.com/os/ios/ (2026-10-30) |
| 補足 | 2026 年 5〜6 月に「iOS 27 で SE が外れる」というリーク記事があったが、Apple の発表で否定された | [二次] https://macrumors.com/2026/06/04/ios-27-compatible-iphones-rumor / https://www.macrumors.com/2026/06/08/ios-27-supports-iphone-11-newer/ (2026-10-30) |

### 2.2 Apple Intelligence 非対応と、それで使えない機能

- **Apple Intelligence の対応機種** [一次]: iPhone 15 Pro / 15 Pro Max、iPhone 16 系(16e を含む)、iPhone 17 系、iPhone Air、iPhone 18 Pro 系、iPhone Duo。SE 第 3 世代は含まれない。SE 第 3 世代の技術仕様ページにも Apple Intelligence の記載は無い。
  - https://www.apple.com/apple-intelligence/ (2026-10-30)
  - https://support.apple.com/en-us/111866 (2026-10-30)

| 機能 | SE 第 3 世代で | 根拠 |
|---|---|---|
| Foundation Models(端末内 `SystemLanguageModel`) | **使えない**。利用不可の理由に `deviceNotEligible`(「The device does not support Apple Intelligence.」)がある | [一次] https://developer.apple.com/documentation/foundationmodels/systemlanguagemodel/availability-swift.enum/unavailablereason (2026-10-30) |
| PCC(`PrivateCloudComputeLanguageModel`、iOS 27) | **使えない**。Apple の記事に「PCC is only available on devices that support Apple Intelligence」とある | [一次] https://developer.apple.com/documentation/foundationmodels/adding-server-side-intelligence-with-private-cloud-compute / https://developer.apple.com/documentation/foundationmodels/privatecloudcomputelanguagemodel (2026-10-30) |
| AirPods ライブ翻訳 | **使えない**。条件は「iPhone 15 Pro or later」、iOS 26 以降、Apple Intelligence がオンであること | [一次] https://support.apple.com/en-us/123185 (2026-10-30) |
| AirPods 5 のライブ翻訳・Siri AI | **使えない**。Apple Intelligence 対応の iPhone と iOS 27 が条件 | [一次] https://support.apple.com/en-us/106340 (2026-10-30) |
| Translation framework の `highFidelity`(iOS 26.4+) | **実質使えない**。「Apple Intelligence を使う」方式と定義されている。使えない場合はシステムが端末の能力に応じて別の方式を自動で選ぶ。SE 第 3 世代では `lowLatency`(従来モデル)になると考えられる | [一次] https://developer.apple.com/documentation/translation/translationsession/strategy (2026-10-30)。`lowLatency` に落ちることは [推測] |
| メッセージ / 電話 / FaceTime のライブ翻訳、作文ツール、Visual Intelligence、Siri AI(iOS 27)など | **使えない**(Apple Intelligence の機能として案内されている) | [一次] https://www.apple.com/apple-intelligence/ / https://www.apple.com/ios/feature-availability/ (2026-10-30) |
| 空間オーディオの個人プロファイル作成 | **できない**。「iPhone with TrueDepth camera required」とあり、SE 第 3 世代は Touch ID 機 | [一次] https://support.apple.com/en-us/121204 / https://support.apple.com/en-us/111866 (2026-10-30)。本用途には影響しない [推測] |

### 2.3 SpeechAnalyzer / SpeechTranscriber(iOS 26)

- **Apple Intelligence は条件ではない** [一次]。Apple が示している条件は、iOS 26 以降と「certain hardware requirements」だけ。判定は実行時に `SpeechTranscriber.isAvailable`(「the device's hardware and capabilities」から判定)か `supportedLocales` で行い、非対応なら `DictationTranscriber` を使うよう案内されている。WWDC25 でも Apple Intelligence は「組み合わせると要約機能などが作れる」という扱いで、文字起こしの前提ではない。
  - https://developer.apple.com/documentation/speech/speechtranscriber (2026-10-30)
  - https://developer.apple.com/videos/play/wwdc2025/277/ (2026-10-30)
- **ハードウェアの条件は Apple が公表していない** [一次の範囲で未確認]。開発者フォーラムには次の報告がある [二次]。
  - 使えない: iPhone 11 / 11 Pro / 11 Pro Max、iPhone SE(第 2 世代)。いずれも A13 で、Neural Engine は 8 コア。
  - 使える: iPhone 12〜17 の各系列、**iPhone SE(第 3 世代)**、iPhone 14 Pro、iPhone 15 Pro。
  - 投稿者は「16 コアの Neural Engine が条件ではないか」と推測している。Apple の回答は無い。
  - SE 第 3 世代の A15 は 16 コアの Neural Engine を持つ [一次: https://support.apple.com/en-us/111866]。
  - https://developer.apple.com/forums/thread/807739 (2026-10-30)
  - https://developer.apple.com/forums/thread/806765 (2026-10-30)
  - → **SE 第 3 世代で動く可能性が高い。** ただし根拠は利用者の報告だけなので、実機で `isAvailable` を確認する [推測]。
- **ja の対応**: 提供開始時の対応言語に日本語が含まれていた(WWDC25 のスライドの言語一覧)[二次] https://blog.addpipe.com/apple-speechanalyzer-api/ (2026-10-30)。Apple のドキュメント本文には言語一覧が無い。ドキュメントのメタデータにある ja-JP はドキュメント自体の翻訳言語なので、対応の根拠にならない。実機で `supportedLocales` を確認する。
- **代替 1: `DictationTranscriber`(iOS 26)/ `SFSpeechRecognizer` の端末内認識** [一次]
  - `DictationTranscriber` は「compatible with older devices」とされている。iOS 10 以降の端末内 `SFSpeechRecognizer` と同じモデル・言語・機種に対応する。ネットワーク経由でしか使えない言語は対象外。
    - https://developer.apple.com/documentation/speech/dictationtranscriber (2026-10-30)
    - WWDC25 277(上記)
  - 日本語の端末内音声入力の対応: Apple の機能対応表で、「Dictation (on-device and modeless)」に日本語が載っている [一次] https://www.apple.com/ios/feature-availability/ (2026-10-30)。ここから、`DictationTranscriber` / 端末内 `SFSpeechRecognizer` は ja を扱えると考えられる [推測。実機で `supportsOnDeviceRecognition` を確認する]。
  - 精度は SpeechTranscriber より下がる可能性がある。「遠い声・長い会話」向けと案内されているのは SpeechTranscriber のほう [推測]。
- **代替 2: クラウド STT**。周囲音のクラウド送信が許可されたので使える。候補と単価は 02 §4.3 のとおり(Deepgram Nova-3、`gpt-live-transcribe` など)。Claude の $50 枠の外の費用になる。

### 2.4 Translation framework(ja→en / zh / fr)

- 使える OS: iOS 17.4 以降(UI なしの `TranslationSession` 初期化は iOS 26 から。03 §3.2)[一次] https://developer.apple.com/documentation/translation (2026-10-30)。端末内のモデルで翻訳し、言語データはシステム全体(翻訳アプリを含む)で共有する [一次] https://developer.apple.com/videos/play/wwdc2024/10117/ (2026-10-30)。
- 機種要件の記載は無い。Apple Intelligence が要るのは `highFidelity` だけ(§2.2)[一次]。
- 対応言語: システム全体の翻訳(System-wide translation)に、日本語、英語(英国・米国)、フランス語(フランス)、中国語(北京語、中国本土・台湾)が載っている。対応ペアの判定は `LanguageAvailability.status(from:to:)` で実行時に行う [一次]。
  - https://www.apple.com/ios/feature-availability/ (2026-10-30)
  - https://developer.apple.com/documentation/translation/languageavailability (2026-10-30)
- **簡体 / 繁体の別**: Apple の表記は「Mandarin Chinese (China mainland, Taiwan)」で、地域で分けている。簡体・繁体という表記は使っていない [一次]。`Locale.Language` で zh-Hans と zh-Hant(または zh-CN と zh-TW)の両方が `.supported` になるかは [未確認](実機で確認する)。
- 評価: 口調は指定できない(03 §3.2)。SE 第 3 世代では `lowLatency` しか使えないので、Translation framework は「オフライン時の予備」と「Claude 訳との比較の基準」に限って使う [推測]。

### 2.5 AVSpeechSynthesizer(en / zh / fr)

- 声の品質は default(最初から入っている)/ enhanced / premium の 3 段階。enhanced と premium はダウンロードが必要 [一次] https://developer.apple.com/documentation/avfaudio/avspeechsynthesisvoicequality (2026-10-30)。機種要件の記載は無い [一次の範囲]。
- enhanced 音声は 100MB 以上になることがあり、Wi-Fi でのダウンロードが推奨されている。ダウンロード先は「設定 > アクセシビリティ > 読み上げコンテンツ > 声」[一次] https://support.apple.com/en-us/111798 / https://support.apple.com/guide/iphone/hear-whats-on-the-screen-or-typed-iph96b214f0/ios (2026-10-30)。
- **Siri の声はサードパーティのアプリから使えない**という報告が複数ある [二次]。WWDC26 の Q&A で「新しい Siri の声を開放するか」という質問が出ているが、回答は無い [二次]。
  - https://developer.apple.com/forums/thread/682438 (2026-10-30)
  - https://developer.apple.com/forums/thread/832465 (2026-10-30)
- まだダウンロードしていない声の一覧を取得する API は無い。`speechVoices()` は端末に入っている声だけを返す [二次] https://developer.apple.com/forums/thread/758460 (2026-10-30)。
- iOS 26 で、アクセシビリティの設定で選んだ声がアプリ側に反映されない不具合の報告がある [二次] https://developer.apple.com/forums/thread/804648 (2026-10-30)。アプリ内で声の識別子を明示して選ぶ作りにする [推測]。
- 各言語(en-US / zh-CN / zh-TW / fr-FR)に premium 音声があるか、品質はどうかは [未確認]。端末で `speechVoices()` を列挙し、試聴して確かめる。SE 第 3 世代に固有の制約は見つからなかった。

### 2.6 「AirPods 4 Pro」とマイク

- **「AirPods 4 Pro」は存在しない** [一次]。Apple の比較ページにある現行・旧機種は、AirPods 5 / AirPods 5(ワイヤレス充電ケース)/ AirPods 4 / AirPods 4(ANC)/ AirPods Pro 3 / AirPods Pro 2 / AirPods Max / AirPods Max 2 / AirPods(第 2・第 3 世代)。https://www.apple.com/airpods/compare/ (2026-10-30)
- ユーザーへの確認方法 [一次]: 「設定 > Bluetooth > AirPods の (i)」に型番が出る。https://support.apple.com/en-us/109525 (2026-10-30)

| 候補 | 型番(109525) | マイク(一次) | 高音質録音 / `bluetoothHighQualityRecording` | SE 第 3 世代との組み合わせで使えない機能 |
|---|---|---|---|---|
| AirPods 4(ANC なし) | A3050 / A3053 / A3054 | デュアルビームフォーミングマイク、内向きマイク、音声加速度センサー | 2025-06 の発表とファームウェア 8A356 の対象 [一次] | ライブ翻訳はもともと非対応。空間オーディオの個人プロファイル作成 |
| AirPods 4(ANC) | A3055 / A3056 / A3057 | 同上 | 同上 [一次] | ライブ翻訳(AI 非対応のため)、個人プロファイル作成 |
| AirPods Pro 2 | A2698 / A2699 / A2931(Lightning)、A3047〜A3049(USB-C) | デュアルビームフォーミングマイク、音声加速度センサー(比較ページの記載。技術仕様ページは未読) | 同上 [一次] | ライブ翻訳、個人プロファイル作成 |
| AirPods Pro 3 | A3063 / A3064 / A3065 | デュアルビームフォーミング、内向きマイク、音声加速度センサー | 技術仕様に「Studio-quality audio recording」、8A356 の対象 [一次] | ライブ翻訳、個人プロファイル作成 |
| AirPods 5(2026) | A3531〜A3533 / A3439〜A3441 | デュアルビームフォーミング、内向きマイク | 技術仕様に「Studio-quality audio recording」[一次] | ライブ翻訳と Siri AI(AI 対応機 + iOS 27 が条件)、個人プロファイル作成 |

- 出典(すべて確認日 2026-10-30)
  - AirPods 4: https://support.apple.com/en-us/121203
  - AirPods 4(ANC): https://support.apple.com/en-us/121204
  - AirPods Pro 3: https://support.apple.com/en-us/125135
  - AirPods 5: https://support.apple.com/en-us/148759
  - 高音質録音の発表: https://www.apple.com/newsroom/2025/06/airpods-now-more-versatile-with-studio-quality-audio-recording-and-camera-remote/
  - ファームウェアのリリースノート: https://support.apple.com/en-us/106340
- **高音質録音の iPhone 側の条件は「iOS 26 に対応する iPhone」だけ**で、Apple Intelligence は要らない [一次: 106340]。SE 第 3 世代でも使える。
- API(`bluetoothHighQualityRecording`)側の注意は 02 §5.2 のとおり。default モード限定、EU 非対応、入力遅延が増えるためリアルタイム通信には非推奨。対応機種の一覧は無く、「certain AirPods models」とだけ書かれている [一次] https://developer.apple.com/documentation/avfaudio/avaudiosession/categoryoptions-swift.struct/bluetoothhighqualityrecording (2026-10-30)。ユーザー向けの「スタジオ品質の録音」と API の `isSupported` が同じ範囲かは [未確認]。実行時に `highQualityRecording.isSupported` で確認する。
- 本用途への影響: 主入力は iPhone 内蔵マイクで、AirPods は A2DP の出力専用にする方針(02 §5.4)。この方針ならどの候補でも構成は同じ [推測]。差が出るのは次の 2 点。
  - (1) AirPods のマイクを比較試験に使えるか: AirPods 4〜Pro 3 / 5 はどれも高音質録音の対象。
  - (2) ANC / 外部音取り込みの有無: 周囲の原音と訳音声の聞こえ方の好みに関わる。AirPods 4(ANC なし)は ANC が無い。

### 2.7 iPhone SE 第 3 世代のマイク

- 技術仕様ページにマイクの数と位置は書かれていない [一次の範囲で未確認]。書かれているのは、ビデオ撮影の「Stereo recording」、通話の「Voice Isolation and Wide Spectrum microphone modes」、Bluetooth 5.0 [一次] https://support.apple.com/en-us/111866 (2026-10-30)。ステレオで録画できるので、マイクは 2 個以上あると考えられる [推測]。
- 内蔵マイクは「複数の物理マイクとビームフォーマー」として `AVAudioSessionDataSourceDescription` の一覧で公開される。iOS 14 以降、対応機種では front / back のデータソースに `stereo` の指向特性を設定すると、ステレオで録音できる。対応しているかは、データソースの `supportedPolarPatterns` に `stereo` が含まれるかで判定する。カテゴリは `record` / `playAndRecord` が必要 [一次] https://developer.apple.com/documentation/avfaudio/capturing-stereo-audio-from-built-in-microphones (2026-10-30)。
- SE 第 3 世代でどのデータソース(Bottom / Front / Back など)と指向特性が使えるかは [未確認]。最初の実機確認で `availableInputs` → `dataSources` → `supportedPolarPatterns` を列挙する。周囲の雑談を拾うには、omnidirectional か stereo が向く可能性がある [推測]。
- Apple のサポート記事は、通話用(本体下部)とスピーカーフォン・録画用(本体上部)のマイクを分けて確かめる手順を案内している [一次] https://support.apple.com/en-us/101600 (2026-10-30)。

### 2.8 Claude API の費用(Haiku 5.5 / Sonnet 5.5)

- 単価(標準、1M トークンあたり)[一次] https://platform.claude.com/docs/en/about-claude/pricing (2026-10-30)

| モデル | 入力 | 出力 | キャッシュヒット | 備考 |
|---|---|---|---|---|
| Claude Haiku 5.5(プロンプト 10 万トークン以下) | $0.10 | $0.50 | $0.01 | プロンプトが 10 万トークンを超える要求は $0.50 / $2.50 |
| Claude Sonnet 5.5 | $2 | $10 | $0.10 | 長いコンテキストでも標準単価 |

- 補足 [一次]: Claude 4.7 以降は新しいトークナイザで、同じ文章でもトークン数が約 30% 増える。US 限定の推論(`inference_geo`)を指定すると 1.1 倍。Batch API は半額だが、リアルタイム用途には使えない [推測]。
- 試算(計算は作業用ディレクトリの使い捨てスクリプト。式: 回数 = 発話分 × 毎分の回数、費用 = 入力トークン × 入力単価 + 出力トークン × 出力単価)

| シナリオ | 呼び出し回数/月 | Haiku 5.5 | Sonnet 5.5 |
|---|---|---|---|
| 02 の仮定(発話 1,800 分、10 秒ごと、入力 600 / 出力 40) | 10,800 | **$0.86** | **$17.28** |
| トークン 1.3 倍(02 のトークン数が旧トークナイザ基準だった場合) | 10,800 | $1.12 | $22.46 |
| 5 秒ごと(呼び出し 2 倍) | 21,600 | $1.73 | $34.56 |
| 発話率 100%(3,600 分) | 21,600 | $1.73 | $34.56 |
| 入力 1,500 トークン(文脈を長く持つ) | 10,800 | $1.84 | $36.72 |
| 最悪(3,600 分 + 5 秒ごと + 1.3 倍) | 43,200 | $4.49 | $89.86 |

- **判定**: 02 の仮定なら、Sonnet 5.5 でも $50 の約 1/3 に収まる。仮定が 1 つ悪い方に振れても(2 倍)収まる。複数が重なると超える [計算]。
- $50 枠には、決定事項にある「会話が無いときの出力(コンテンツ生成)」の Claude 費用も入る。この分は今回の試算に含めていない。
- プロンプトキャッシュ(固定の指示部分)を使えば入力費は下がる。ただし最小キャッシュ長と、600 トークンのうちキャッシュできる割合は [未確認] なので、試算には入れていない。
- 運用上の対策 [推測]: 既定は Haiku にし、Sonnet は聞き比べで差がある場合に限って使う。アプリ側で月の呼び出し回数に上限を置く。

## 3. 01〜03 のレポートで修正が要る箇所

| ファイル / 章 | 何が変わるか |
|---|---|
| 01 §1 結論 2・4 | AirPods ライブ翻訳は SE 第 3 世代では使えない(Apple Intelligence 非対応)。「次の一手」の試験対象から外し、Google 翻訳と ChatGPT Voice の 2 つにする |
| 01 §3.1 AirPods ライブ翻訳 | 「本プロジェクトへの適合」に「実機(SE 第 3 世代)では不可」を加える |
| 01 §4 比較表 | AirPods ライブ翻訳の行に「SE 第 3 世代では不可」を加える |
| 01 §5 試験 A | 実施できない(対応機種なし)。試験 B(Google 翻訳)・試験 C(ChatGPT)はどちらも「iOS 18.0 以降」なので SE 第 3 世代でも可 [一次: App Store の互換性欄 https://apps.apple.com/us/app/google-translate/id414706506 / https://apps.apple.com/us/app/chatgpt/id6448311069 (2026-10-30)] |
| 01 §7 未確認事項 | ライブ翻訳に関する項目(「自分の言語」に学習言語を選べるか、ロック中の動作)は対象外になる |
| 02 §1 結論 2(推奨構成) | STT は「SpeechAnalyzer(SE 第 3 世代は動作報告あり)→ 不可なら `DictationTranscriber` → クラウド STT」の順にする。LLM は Claude(Haiku 5.5、Sonnet 5.5 と比べる)に絞る |
| 02 §4.3 STT 表(Apple の行) | 機種要件(A13 では使えないという報告、SE 第 3 世代は動作報告あり、`isAvailable` で判定)と、`DictationTranscriber` による代替を書き足す |
| 02 §4.5 TTS 表(Apple の行) | enhanced / premium 音声はダウンロードが必要、Siri の声は使えない(二次)、iOS 26 で声の選択が反映されない不具合の報告(二次)を加える |
| 02 §5.2 高音質録音 | iPhone 側の条件は iOS 26 だけで、SE 第 3 世代でも使える。対象 AirPods に Pro 3 と AirPods 5 を加える(技術仕様に記載)。§9 の未確認 7 は一部解消 |
| 02 §6.2 構成 C | Sonnet 5.5 の $17.28 を再確認した。悪い方に振れたときの幅($22〜$90)と、無音時コンテンツ生成が別に掛かることを加える |
| 03 §1 結論 1〜3 | (a) は `highFidelity` 無しで成り立つ。(b) の「PCC / 端末内 Foundation Models」は不可で、クラウド LLM だけになる。(c) の「iPhone 内で完結」は Foundation Models が使えないため対象外(Mac 中継も決定で対象外) |
| 03 §2 比較表 | (a) の「Apple Intelligence 対応機なら `highFidelity`」と、(b) の「端末内 FM / PCC」を、SE 第 3 世代では不可と注記する |
| 03 §3.1 SpeechAnalyzer | 未確認だった「対応機種の要件」に、フォーラムの報告(A13 は不可、SE 第 3 世代は可)を二次情報として加える |
| 03 §3.2 Translation | SE 第 3 世代では `highFidelity` が使えず `lowLatency` になる(推測)。未確認の「`highFidelity` の口語らしさ」は対象外 |
| 03 §3.3 Foundation Models / PCC | 節全体が SE 第 3 世代では使えない(`deviceNotEligible`、PCC も AI 対応機のみ) |
| 03 §6.2 AirPods マイク | 高音質録音は SE 第 3 世代でも使える。比較試験はそのまま実施できる |
| 03 §6.3 電池と発熱 | 端末内 LLM の選択肢が消えたので、論点は「マイク + STT + 通信 + TTS を 2 時間続けたときの電池」になる。SE 第 3 世代は筐体が小さく、電池の余裕は小さい(推測) |
| 03 §7.3 遅延予算 | 翻訳段は「PCC / クラウド LLM」から「Claude(クラウド)」だけになる |
| 03 §9.1 最初の 1〜2 週 | 試す対象から「`highFidelity`」「端末内 Foundation Models」「PCC」「Mac の MLX」を外し、「Claude Haiku と Sonnet」「Translation lowLatency(基準)」にする |
| 03 §9.3 リスク | 「PCC の日次上限」の行は対象外。「Claude の月額超過」(上限 $50)を加える |
| 03 §11 未確認事項 4・5 | `highFidelity` と PCC の項目は対象外になる |
| (参考)00 §3・§4・§5 | §3 推奨構成から PCC と端末内 LLM を外す。§4 段階 0 の AirPods ライブ翻訳を外す。§5 の 7(機種)は「不可」で確定 |

## 4. 未確認事項

1. SE 第 3 世代で `SpeechTranscriber.isAvailable` が true になるか。根拠は利用者の報告だけで、Apple は機種要件を公表していない。実機で最初に確かめる。
2. `SpeechTranscriber.supportedLocales` に ja-JP が含まれるか(根拠は二次情報だけ)。`DictationTranscriber` / 端末内 `SFSpeechRecognizer` の ja での `supportsOnDeviceRecognition`。
3. Translation framework で、中国語の簡体(zh-Hans / zh-CN)と繁体(zh-Hant / zh-TW)が別々に `.supported` になるか。ja→fr / ja→en も含めて実機で `LanguageAvailability` を確認する。
4. AVSpeechSynthesizer で en / zh-CN / zh-TW / fr-FR の premium 音声があるか。口語の聞き流しに耐える品質か(試聴が必要)。
5. SE 第 3 世代の内蔵マイクの数・位置・データソース名と、stereo / omnidirectional の指向特性に対応しているか(公式の仕様に記載なし)。
6. ユーザーの AirPods の正確な機種(「4 Pro」は存在しない)。高音質録音 API の `isSupported` が、その機種で true になるか。
7. SE 第 3 世代で、マイク + STT + 通信 + TTS を画面ロック中に 2 時間続けたときの電池・発熱(00 §7 と同じ。小型の電池なので重要度が上がる)。
8. Claude の実トークン数(日本語の入力と英・中・仏の出力。新しいトークナイザでの実数)。プロンプトキャッシュの適用条件。無音時コンテンツ生成の使用量。
