# 視点 3: ゼロから作る場合(端末内処理・オープンモデル・自前実装)

- 調査日: 2026-10-30(出典の確認日もすべて 2026-10-30)
- 調査: サブエージェント(Opus)。一次情報で確認できたことを「確認済み」、二次情報だけのものを「二次情報」、確かめていないことを「未確認」、筆者の見積もりや推論を「推定」と書く。
- 前提: 入力は周囲の日本語の雑談、出力は学習言語のラフな口語音声。入出力とも AirPods。iPhone で使う(Mac 中継も検討する)。

## 1. 結論

1. **最初に作るなら (a)(iOS 内蔵部品だけのアプリ)。ただし (a) のままでは口語にならない。** SpeechAnalyzer(端末内 STT)、Translation framework(端末内翻訳)、AVSpeechSynthesizer をつなぐ構成で、無料・オフライン・音声を端末外に出さずに作れる。一方 Translation framework には口調を指定する API が無い(確認済み)。
2. **口語化は LLM に任せる。現実的なのは (b)(端末内 STT + LLM)。** iOS 27 では Apple の PCC(Private Cloud Compute)モデルを Foundation Models から使える。API キーも開発者側のトークン課金も不要で、ユーザーごとの日次上限がある(確認済み、利用申請が必要)。端末内 Foundation Models や一般のクラウド LLM でも同じ構成になる。
3. **(c) 完全ローカルは、自宅の Mac(MLX / llama.cpp)なら現実的。** iPhone 内だけで LLM を常時動かすのは、発熱・電池・速度の面で実測してからでないと勧められない(推定)。
4. **(d) 翻訳モデルを自作する道は見合わない。** 日本語入力の同時音声翻訳を出しているオープンモデルは見つからない。Hibiki-Zero は仏西葡独→英のみで非商用(確認済み)。SeamlessStreaming は日本語に対応するが非商用で、GPU サーバ前提(確認済み)。
5. **最大のリスクは「周囲の雑談を拾えるか」と「画面ロック中に動き続けるか」。** 最初の 1〜2 週は、iPhone マイクと AirPods マイク(HFP / 高音質録音)の集音比較、ロック中の連続稼働、段ごとの遅延を実測することに充てる。

## 2. 段階分け (a)〜(d) の比較

| 段階 | 構成 | 口語化 | 外部依存 | 遅延の目安(発話終了から再生開始まで、推定) | 工数(推定) | 主な制約 |
|---|---|---|---|---|---|---|
| (a) 内蔵部品のみ | SpeechAnalyzer → Translation → AVSpeechSynthesizer(すべて端末内) | ×(口調パラメータなし。iOS 26.4 以降の `highFidelity` は「より流暢」というだけ) | なし(言語データのダウンロードのみ) | 1.5〜3 秒 | 試作 1〜2 人週、常用品質 4〜6 人週 | iOS 26 以降。対応言語は端末で実行時に確認する必要あり。Apple Intelligence 対応機なら `highFidelity` も使える |
| (b) 端末内 STT + LLM | SpeechAnalyzer → LLM(端末内 Foundation Models / PCC / クラウド LLM)→ TTS | ◎(プロンプトで指定できる) | PCC は Apple のみ(申請、ユーザー日次上限)。クラウド LLM なら API キーと従量課金 | 2〜4 秒 | (a) に加えて 1〜3 人週 | Foundation Models は Apple Intelligence 対応機に限られ、コンテキストは 4096(iOS 27 の新しい機種では 8192)。PCC と クラウド LLM はネット接続が必要 |
| (c) 完全ローカル(オープンモデル) | Mac: STT(SpeechAnalyzer か Whisper 系)+ MLX/llama.cpp の LLM + TTS。iPhone: 同じ構成を端末内で | ◎(プロンプト。微調整も可能) | なし | Mac: 2〜4 秒。iPhone: 未測定 | Mac 中継: 3〜6 人週。iPhone 内完結: 4〜8 人週 | Mac は持ち運び・電源・集音位置が制約になる。iPhone は発熱と電池(未測定) |
| (d) 翻訳モデルを作る・追加学習する | 日本語→学習言語の同時音声翻訳モデルを学習(Hibiki-Zero 型)、または小型 LLM を口語訳向けに微調整 | ◎ | GPU 計算資源 | ─ | S2ST は数十人週以上、LLM の LoRA 微調整は 2〜4 人週 | 学習データ(Hibiki-Zero でも新言語に 1,000 時間未満の音声)、GPU、評価体制が必要。個人の聞き流し用途には過剰 |

工数は「Swift 経験のある個人開発者が 1 人で作る」場合の筆者による見積もりで、根拠となる実測値はない。

## 3. iOS 内蔵機能の実力

### 3.1 SpeechAnalyzer / SpeechTranscriber(STT)

- **確認済み**
  - iOS / iPadOS / macOS / tvOS / visionOS 26.0 以降で使える。watchOS は対象外。[^speechtranscriber]
  - 想定する用途は、講義・会議・会話のような長時間の音声と、マイクから離れた話者の音声。WWDC25 では「long-form and distant audio, such as lectures, meetings, and conversations」と説明されている。[^wwdc25-277]
  - 処理はすべて端末内で行う。モデルは AssetInventory でダウンロードし、アプリのサイズにもアプリのメモリにも含まれない。[^wwdc25-277]
  - 暫定結果(volatile results)を、話されるとほぼ同時に受け取れる。後から確定結果に置き換わる。[^wwdc25-277]
  - 同時に使える言語の数に上限がある。[^wwdc25-277]
  - VAD(発話区間検出)モジュールとして `SpeechDetector` があり、感度を設定できる。Apple は「発話を取りこぼすと精度が下がる」というトレードオフに触れている。[^speechdetector]
  - 端末が対応していない場合は `DictationTranscriber` に切り替えるよう案内されている。[^speechtranscriber]
- **二次情報**
  - 日本語への対応。WWDC25 のスライドに、提供開始時の対応言語として Japanese が載っているという引用がある。[^addpipe]
  - 日本語ストリーミングでの初回結果までの時間。DictationTranscriber の progressive 設定で 0.56 秒という個人の測定がある。ただし合成音声のファイルを入力した測定で、人の声での精度は測っていない。[^issoh]
  - 補足: Apple のドキュメント JSON の `availableLanguages` は「ドキュメントの翻訳言語」を表すもので、対応ロケールの根拠にはならない。[^fm-lang]
- **未確認**
  - 日本語の雑談(離れた話者・重なり・雑音)での認識精度。
  - バックグラウンドや画面ロック中に長時間動かしたときの安定性、およびメモリ・電池への影響。
  - 対応する機種の要件(Apple は「certain hardware requirements」とだけ書いている)。

### 3.2 Translation framework

- **確認済み**
  - `TranslationSession` は端末内で翻訳する。Apple が収集するのは利用指標だけで、原文と訳文は含まれない。[^translationsession]
  - UI なしで使う初期化子 `init(installedSource:target:)` は iOS 26.0 から。言語が未インストールの場合はエラーになるため、事前にダウンロードの許可を取っておく必要がある。[^ts-init]
  - **口調・丁寧さ・文体を指定するパラメータは無い。** 設定項目は source、target、preferredStrategy、version だけ。[^ts-config]
  - iOS 26.4 で `Strategy` が加わった。`highFidelity` は「Apple Intelligence を使い、より流暢に訳す」、`lowLatency` は「従来のモデルで速く訳す」。指定した方式が使えない場合、システムが別の方式を自動で選ぶ。[^ts-strategy]
- **二次情報**
  - 対応言語。iOS 26 で日本語・英語・中国語・韓国語・仏独西などに対応しているという記載がある。[^applewiki] 実際に使えるかは `LanguageAvailability` で実行時に確かめる。[^translation]
- **未確認**
  - 1 文あたりの翻訳時間。
  - `highFidelity` で訳文がどの程度くだけるか。

### 3.3 Foundation Models(端末内 LLM)と翻訳

- **確認済み**
  - WWDC25 Group Lab で Apple は「The Translation API is still preferred... It's not recommended for general world knowledge or translation tasks.」と回答している。[^gl25]
  - WWDC26 Group Lab ではやや柔軟な説明に変わった。Foundation Models が翻訳に対応する言語は Translation framework より少ない、という前提を置いたうえで、決まった訳出なら Translation API、内容が予測できない場合や「different styles」が必要な場合は言語モデルの方がよい、という趣旨の発言をしている。[^gl26]
  - コンテキストは入出力を合わせて 4096 トークン(iOS 26.0)。iOS 27 の新しい機種では 8192。[^gl26][^wwdc26-319][^wwdc26-241]
  - Foundation Models はバックグラウンドでも動く。ただしシステムが混んでいるときや、短時間に要求が多すぎるときはレート制限がかかる。[^gl26]
  - 使えるのは Apple Intelligence に対応した機種と地域だけ。[^slm]
  - **口調の指定(LoRA アダプタ)**: アダプタ学習ツールキットの用途に「specific style」が明記されている。ただし、アダプタはシステムモデルのバージョンごとに学習し直す必要があり、1 つ約 160MB、配布には entitlement が必要。さらにツールキット 26.0.0 が最終版で、**iOS 27 以降では使えない**。[^adapter]
- **iOS 27 の PCC モデル(確認済み)**
  - `PrivateCloudComputeLanguageModel()` に切り替えれば、Apple のサーバ上のモデルを使える。コンテキストは 32K。[^wwdc26-319]
  - 対象はダウンロード数 200 万未満のアプリで、開発者サイトから申請する。
  - 開発者にトークン費用はかからない。ユーザーごとに日次上限があり、iCloud+ で上限が上がる。
  - ユーザーのデータは保存されないと説明されている。オフラインでは使えない。[^wwdc26-319]
- **未確認**
  - PCC の日次上限の具体的な数値と、応答の遅延。連続稼働に耐える上限かどうかは、実測してみないと分からない。
- **速度の参考値(確認済み)**: 2024 年に Apple が公表した約 3B の端末内モデルは、iPhone 15 Pro で初回トークンまでがプロンプト 1 トークンあたり約 0.6ms、生成が約 30 トークン/秒だった。[^afm2024] iOS 27 の新モデルの数値は公表されていない(未確認)。

### 3.4 AVSpeechSynthesizer(TTS)

- **確認済み**
  - 発話を順番に読み上げるキューを持つ。[^avss]
  - `usesApplicationAudioSession` で、アプリ側のオーディオセッションに従わせることができる。[^avss]
  - `write(_:toBufferCallback:)` で音声バッファを受け取れるので、自前の AVAudioEngine に流し込める。[^avss]
  - 声の品質は default / enhanced / premium の 3 段階。enhanced と premium は事前のダウンロードが必要。[^voicequality]
- **未確認**
  - 発声開始までの遅延と、各言語の premium 音声の自然さ(実測と試聴が必要)。
  - 口語らしい抑揚が出るかどうか。

## 4. オープンモデルと実行環境

### 4.1 STT(日本語)

| 候補 | 実行環境 | 日本語 | ライセンス | 本件への評価 |
|---|---|---|---|---|
| WhisperKit(Argmax) | iOS / macOS、Core ML。マイク入力のストリーム、VAD による分割 | Whisper の多言語モデル(large-v3 turbo の圧縮版 626MB など)。日本語特有の設定はリポジトリに記載なし | MIT。話者付きリアルタイム文字起こしなどは有償の Pro SDK[^whisperkit] | 実績のある端末内 STT。ただし雑談を連続で処理したときの電池・遅延は未測定 |
| kotoba-whisper v2.0 | Whisper 互換(distil-large-v3 と同じ構造)、約 0.8B | 日本語専用に学習。large-v3 より 6.3 倍速い。CER は ReazonSpeech で 11.6(large-v3 は 14.9)、JSUT で 8.4(同 7.1)[^kotoba] | Apache-2.0 | Mac 中継で日本語の精度を上げたいときの候補。ストリーミング・端末内での使用はモデルカードに記載なし |
| NVIDIA parakeet-tdt_ctc-0.6b-ja | NeMo(約 0.6B) | JSUT の CER 6.4、TEDxJP-10k の CER 9.0。句読点付きで出力する[^parakeet-ja] | CC-BY-4.0 | 精度は高い。iOS / MLX への移植状況は未確認、ストリーミングは記載なし |
| ReazonSpeech k2 v2 | ONNX(Next-gen Kaldi)、159M | 日本語専用の RNN-T。約 30 秒までの音声を処理できる[^reazon] | Apache-2.0 | 軽量。iOS では sherpa-onnx をソースからビルドする必要がある(二次情報)[^sherpa] |
| Moonshine | iOS を含む多くのプラットフォーム。ストリーミング最適化 | 日本語の Base / Tiny は「旧式の非ストリーミング」モデル | 日本語モデルは非商用の Moonshine Community License[^moonshine-license] | 個人利用なら試せる。日本語のストリーミング版があるかは未確認 |
| SpeechAnalyzer(内蔵) | OS 内蔵。アプリのメモリ外で動く | 二次情報で対応(3.1 参照) | OS の API | 実装の手間が最も小さい。最初の比較対象にする |

日本語の雑談音声で、上の候補を互いに比べた一次情報は見つからなかった(未確認)。

### 4.2 同時音声翻訳(Speech-to-Speech)モデル

| モデル | 言語 | ライセンス | 実行環境 | 本件への適否 |
|---|---|---|---|---|
| Kyutai Hibiki | 仏→英のみ | 重みは CC-BY 4.0、コードは MIT / Apache | PyTorch、Rust、MLX(macOS)、MLX-swift(iOS、iPhone 16 Pro で試験済み・実験的)。スマートフォン向けに小型の Hibiki-M がある[^hibiki] | 日本語非対応のため使えない |
| Kyutai Hibiki-Zero(3B) | 仏・西・葡・独→英のみ | CC BY-NC-SA 4.0(非商用)。新しい入力言語は 1,000 時間未満の音声で追加できたと報告されている[^hibikizero-hf] | NVIDIA GPU(VRAM 8GB で動く、12GB あれば安全)[^hibikizero-gh]。Apple Silicon の MLX 4bit 版で 1 ステップ約 75ms(M2 Max)という第三者の報告がある(二次情報)[^soniqo] | 日本語非対応。日本語を加えるには (d) の規模の作業が要る |
| Meta SeamlessStreaming / SeamlessM4T v2 | 日本語(jpn)は音声入力・音声出力・テキストすべてに対応[^m4t-readme]。Streaming は入力 101 言語、音声出力 36 言語[^seamless-hf] | 重みは CC-BY-NC 4.0、コードは MIT[^seamless-gh] | 2.5B。CPU 実行は推奨されておらず、CUDA を使うサーバ構成[^seamless-hf] | 日本語→学習言語の同時音声翻訳ができる、唯一のオープンな候補。ただし非商用で、iPhone で動かすのは非現実的。自宅の GPU サーバを使うなら検討の余地はある。口調は制御できない |
| Gemma 3n(E2B / E4B) | 音声入力から文字起こしと翻訳ができる。140 以上の言語で学習 | Gemma ライセンス(取得にゲートあり) | 実質 2B 相当のメモリで動き、低資源の端末向けに設計されている[^gemma3n] | 日本語音声の翻訳品質とストリーミング対応は未確認 |

### 4.3 ローカル LLM による口語訳

- **候補**
  - TranslateGemma(4B / 12B / 27B、55 言語、Gemma ライセンス、入力 2K トークン)。テキスト翻訳専用で、口調を制御する機能の記載はない。[^translategemma] 公式に対応していないプロンプト技法で口調を指示する形になる(推定)。
  - 汎用の小型 LLM(3B〜8B 級)にプロンプトで口語化を指示する方法。モデルごとの日本語→各言語の口語訳の品質は未確認。
- **速度(確認済み、llama.cpp の公開ベンチ、LLaMA 7B Q4_0 の生成速度)**[^llamacpp-bench]
  - M1: 約 14 トークン/秒
  - M2 Pro: 約 38 トークン/秒
  - M4: 約 24 トークン/秒
  - M4 Pro: 約 50 トークン/秒
- **1 行あたりの時間(推定)**: 口語訳 1 行を 20〜40 トークンとすると、Mac の 7B 級で 1〜2 秒程度。
- **iPhone の場合**
  - 3B 級は約 30 トークン/秒(Apple 公表、iPhone 15 Pro の端末内モデル)。[^afm2024]
  - llama.cpp の 3B Q4 で 22〜28 トークン/秒という報告もあるが、これは販売業者のブログである(二次情報)。[^ertas]
  - 1 行訳すのに 1 秒前後かかる計算になる(推定)。常時動かしたときの発熱・電池は未測定。
- **口語化の方法**
  1. **プロンプトで指示する(第一候補)。**「友人同士の口語、短く、縮約形を使う、訳文だけを出力する」と明示し、必要なら言語ごとの例文を数例添える。
  2. **小型モデルの微調整(LoRA)。** 口語の対訳データ(大きな LLM で作った蒸留データを含む)を数千ペア用意する必要がある。Apple のアダプタ学習ツールキットも 5,000 サンプル以上を「complex tasks」の目安としている。[^adapter] Foundation Models のアダプタは iOS 27 では使えないため、微調整するならオープンモデル(MLX / llama.cpp)側で行う。

### 4.4 ローカル TTS

| 候補 | 言語 | ライセンス | 評価 |
|---|---|---|---|
| AVSpeechSynthesizer(内蔵) | 多数 | OS の API | 実装が最も簡単。品質は premium 音声次第(未試聴) |
| Kokoro-82M | 米英語、英英語、日本語、中国語、スペイン語、フランス語、ヒンディー語、イタリア語、ブラジルポルトガル語。英語以外は「absent or thin」とされ、仏語の学習データは 11 時間未満[^kokoro-voices] | Apache-2.0[^kokoro] | 英語を学ぶ場合は有力。他の言語は品質に注意が必要 |
| Piper(piper1-gpl) | espeak-ng で音素化。多言語の音声あり | GPL-3.0。メンテナを募集中[^piper] | iOS での実績の記載なし。GPL なので配布するなら注意 |
| Argmax TTSKit(Qwen3-TTS 0.6B / 1.7B) | 日本語などのオプションがある | MIT(コード)[^whisperkit] | iOS 18 以降。学習言語の音声品質は未確認 |

## 5. Mac 中継構成

構成としては、Mac のマイク(または外部マイク)で集音して Mac で処理し、Bluetooth(A2DP)で AirPods に出力する形を想定する。

- **利点**
  - 計算資源が大きい。7B 級の LLM なら 1 行 1〜2 秒で訳せる(推定、4.3 参照)。
  - kotoba-whisper のような重い STT や、SeamlessStreaming を GPU なしで試すといった実験の自由度が高い。
  - iPhone のバックグラウンド制約を受けない。
  - SpeechAnalyzer と Translation framework は macOS 26 以降でも使える。[^speechtranscriber][^translation]
  - 集音に AirPods マイクを使わず Mac 本体か外部マイクにすれば、AirPods は A2DP の高音質出力のままにできる。A2DP は出力専用で、HFP は入出力の両方を扱う。[^a2dp]
- **欠点**
  - 持ち運びに向かない。カフェでは Mac を開いたままにする必要があり、マイクの位置も固定される。
  - Bluetooth の届く範囲に縛られる。距離の具体値は未確認で、一般論として 10m 程度とされる(推定)。
  - AirPods が iPhone と Mac の間で自動的に切り替わるため、意図しない経路の変化が起こりうる(推定)。
  - `bluetoothHighQualityRecording` は iOS / iPadOS / Mac Catalyst 26 向けとされていて、ネイティブの macOS アプリで使えるかは未確認。[^bthq]
- **変形案**: iPhone で集音・STT までを行い、テキストだけを自宅の Mac(LAN または VPN 経由)に送って LLM で訳し、結果を iPhone に返して TTS で読み上げる。持ち運べて、しかも外部 SaaS に頼らない形になる。ただしネットワーク経路の遅延と可用性(自宅 Mac の常時稼働)が新しいリスクになる(推定)。
- **向く場面(推定)**: 家でテレビやラジオ、家族の会話を聞き流す用途には Mac 中継が向く。外出先では iPhone 単体の (a) / (b) が現実的。

## 6. iOS 固有の制約

### 6.1 バックグラウンドと画面ロック中の連続録音

- **確認済み**
  - Background Modes の `audio` は、公式には「The app plays audible content in the background」と説明されている。録音についての記述はこの表にはない。[^bgmodes]
  - `AudioRecordingIntent`(iOS 18 以降)を使う場合は、録音を始めるときに Live Activity を開始し、録音中は維持しなければならない。開始しないと録音が止まる。[^audiorecordingintent]
- **二次情報(Apple 開発者フォーラムの開発者の報告、Apple 職員の回答ではない)**[^forum-bg]
  - 完全にバックグラウンドの状態から録音を開始すると `cannotStartRecording` になる。
  - 前面で始めた録音を、そのまま継続することはできる。
- **設計上の結論(推定)**
  - アプリを前面にした状態で開始ボタンを押し、録音を始める。
  - 画面ロック後も同じ AVAudioEngine のセッションを止めずに流し続ける。
  - 電話やアラームなどで中断されたら、再開を処理する。
  - ロック画面での状態表示と停止のために Live Activity を出す。
- **未確認**: 画面ロック中に SpeechAnalyzer・Translation・Foundation Models が制限なく動き続けるか。Foundation Models については、バックグラウンドではレート制限の可能性があると Apple が述べている。[^gl26]

### 6.2 AirPods マイクと Bluetooth の経路

- **確認済み**
  - `allowBluetoothHFP` は、HFP 機器を入力に使えるようにするオプション(`record` / `playAndRecord` カテゴリのみ)。[^hfp]
  - A2DP は出力専用。HFP と A2DP を両方許可すると、HFP が優先される。[^a2dp]
- **推定**: AirPods マイクを入力にすると、出力も HFP の通話品質になる。
- **`bluetoothHighQualityRecording`(iOS 26、確認済み)**[^bthq]
  - Bluetooth の経路が対応していればフル帯域の音声になる。
  - **使えるのは標準(default)のオーディオセッションモードのときだけ。**
  - **EU では現在使えない。**
  - **入力の遅延が増えることがあり、リアルタイムの通話用途には推奨されない。**
  - 対応していない経路では HFP に戻る。
- **対応する AirPods(確認済み)**: 発表時点では AirPods 4、AirPods 4(ANC)、AirPods Pro 2(H2 チップ、ビームフォーミングマイク)。[^newsroom]
- **WWDC25 での説明(確認済み)**: この録音モードは「LAV マイクのように声と背景音のバランスを取る」メディア向けのチューニングとされている。[^wwdc25-251]
- **本件への含意(推定)**
  - AirPods マイクは装着者本人の声を拾うように作られている。
  - Apple 自身も、AirPods のライブ翻訳では騒がしい場所で iPhone のマイクを併用し、iPhone を話し手に近づけるよう案内している。[^support-lt]
  - したがって、周囲の雑談を拾うには **iPhone 本体のマイク(テーブルに置く、胸ポケットに入れる等)+ AirPods の A2DP 出力** が第一候補になる。
  - AirPods マイクの高音質モードは、遅延が増えることも含めて比較対象として実測する。

### 6.3 電池と発熱

- 未確認。STT・LLM・TTS を常時動かしたときの 1 時間あたりの電池消費と温度は、実機で測るしかない。
- `SpeechDetector` で無音区間の処理を省けば電力を節約できると、Apple は説明している。[^speechdetector]

## 7. 技術課題と遅延予算

### 7.1 雑談の区切り

- VAD(SpeechDetector、または Silero VAD: MIT、30ms のチャンクを 1 スレッドで 1ms 未満で処理[^silero])で発話区間を切り出す。確定結果の単位で翻訳に回す。
- 短すぎる断片(相づち・言いさし)は捨てるか、次の断片と合わせてから訳す。閾値(最短文字数、最長待ち時間)は設定ファイルに置く。
- 聞き流し用途なので、訳待ちがたまったら古い行を捨てて最新を優先する方針がよい(推定)。

### 7.2 話者の重なり

- SpeechTranscriber に話者分離の機能があるかは未確認。
- WhisperKit 系では SpeakerKit(pyannote v4)が使えるが、リアルタイムの話者付き処理は有償の Pro SDK の機能になる。[^whisperkit]
- 聞き流しの目的なら、話者分離は後回しにして「混ざったら捨てる」でよい(推定)。

### 7.3 遅延予算(推定。実測で置き換える前提)

| 段 | 目安 | 根拠 |
|---|---|---|
| 集音バッファと VAD の終端判定 | 0.3〜0.8 秒 | 推定(無音を何秒続いたら区切りとするかの設定次第) |
| STT の確定 | 0.3〜1.0 秒 | 暫定結果はほぼ即時[^wwdc25-277]。progressive で初回 0.56 秒という報告(二次情報)[^issoh] |
| 翻訳: Translation framework | 0.1〜0.5 秒 | 未計測 |
| 翻訳: 端末内 LLM(3B 級) | 約 1〜1.5 秒 | 0.6ms/プロンプトトークン、30 トークン/秒[^afm2024] から 300 トークン入力・30 トークン出力で試算 |
| 翻訳: Mac の 7B 級 LLM | 約 1〜2 秒 | 24〜50 トークン/秒[^llamacpp-bench] から試算 |
| 翻訳: PCC / クラウド LLM | 0.5〜2 秒 | 未計測(ネットワーク込み) |
| TTS の発声開始 | 0.1〜0.5 秒 | 未計測 |
| Bluetooth A2DP の出力遅延 | 0.1〜0.3 秒 | 一般論(未確認) |
| **合計(発話終了から再生開始まで)** | **(a) 約 1.5〜3 秒、(b)/(c) 約 2〜4 秒** | 上の積み上げ |

## 8. 周囲への配慮

- **設計方針**
  - 処理は端末内(または自宅の Mac)で行い、音声は端末の外に出さない。
  - 録音ファイル・文字起こし・訳文はディスクに保存しない(メモリ上のリングバッファだけにし、デバッグ用の保存も既定では無効)。
  - (b) で外部 LLM を使う場合も、送るのはテキストだけにする。PCC は「user data is never stored」と説明されている。[^wwdc26-319]
- **日本の法令(概要のみ。法的助言ではない)**
  - 個人情報保護委員会のガイドライン(通則編)は、「特定の個人を識別できる音声録音情報」を個人情報の例に挙げている。[^ppc]
  - 同ガイドラインでは、個人情報取扱事業者の「事業」を「一定の目的をもって反復継続して遂行される同種の行為であって、かつ社会通念上事業と認められるもの」としている。[^ppc]
  - 純粋に私的な利用がこの「事業」に当たるかどうかは、今回は確かめていない(未確認)。
  - アプリを配布する場合や、録音を保存する仕様にする場合は、改めて確認が要る。
- **マナー面(推定)**: 公共の場で常時集音していることを周囲は知らない。保存しないこと、送信しないこと、録音中であることをロック画面で示すこと(Live Activity)を設計上の約束にしておく。

## 9. 工数・リスク・ロードマップ

### 9.1 最初の 1〜2 週(何を作り、何を確かめるか)

- **1 週目: (a) の最小パイプラインを iPhone 実機で動かす**
  - 作るもの:
    - AVAudioEngine で集音する(iPhone マイク、出力は AirPods の A2DP)。
    - SpeechAnalyzer(ja、volatile ありの設定)+ SpeechDetector で文字起こしする。
    - Translation framework の lowLatency と highFidelity で翻訳する。
    - AVSpeechSynthesizer で読み上げる。
  - 確かめること:
    - ① `SpeechTranscriber.supportedLocales` に ja が含まれるか、Translation の言語ペアが対応しているか(`LanguageAvailability`)。
    - ② 段ごとの遅延(ログにタイムスタンプを残す)。
    - ③ テレビの日本語番組やカフェの録音を流したときの聞き取れ具合。
    - ④ 画面ロックしたまま 30〜60 分動き続けるか。
    - ⑤ 1 時間あたりの電池消費と温度。
- **2 週目: 集音経路と口語化を比べる**
  - 集音経路: iPhone マイクと、AirPods マイク(HFP / `bluetoothHighQualityRecording`)を比べる。
  - 口語化: 翻訳段を差し替えて比べる。端末内 Foundation Models、PCC(申請が通っていれば)、Mac の MLX LLM、比較用のクラウド LLM の 4 通りで、口語プロンプトを同じにする。
  - 確かめること: 口語らしさ(本人が主観で 5 段階評価)と、遅延・失敗率。

### 9.2 その後(推定)

| 期間 | 内容 |
|---|---|
| 3〜6 週 | 常用品質にする。中断からの復帰、オーディオ経路の変化、Live Activity、設定画面(閾値・言語・文体)、例外時のフォールバック((b) から (a) へ) |
| 必要に応じて +3〜6 週 | Mac 中継アプリ、またはテキストを自宅 Mac に送る構成 |
| 任意 | オープン LLM の LoRA で口語訳に特化させる(対訳データを作るところから) |

### 9.3 リスク

| リスク | 影響 | 対策 |
|---|---|---|
| 周囲の雑談を拾えない(距離・雑音) | 致命的 | 1 週目に実測する。外部マイクや Mac 中継に切り替える |
| 画面ロック中に止まる・制限される | 大 | 前面で開始する、Live Activity、中断からの復帰。実測する |
| SpeechTranscriber が日本語の雑談に弱い | 大 | WhisperKit や kotoba-whisper(Mac)と比べる |
| 口語化の品質が足りない | 中 | 例文付きのプロンプト、上位モデル(PCC・Mac の大型モデル)、LoRA |
| PCC の日次上限に引っかかる | 中 | Translation framework に戻す。Mac に送る |
| 学習言語が Apple の部品で対応していない | 中 | オープンモデルの STT・TTS に切り替える(Kokoro などの対応言語に注意) |
| 発熱・電池 | 中 | VAD で処理を間引く。LLM は Mac / PCC に逃がす |

### 9.4 視点 2(既存サービスの組み合わせ)と比べてゼロから作る価値がある条件

- 次のいずれかに当てはまるなら、ゼロから作る価値がある(推定):
  - 毎日長時間流すので、従量課金を避けたい。
  - 周囲の人の音声を外部のサーバに送りたくない。
  - オフライン(機内・地下)でも使いたい。
  - 口語化の度合い・語彙レベル・字幕表示などを自分で調整したい。
- 逆に、次の場合は視点 2 の構成で十分:
  - 学習言語が Apple の部品(SpeechAnalyzer / Translation / Apple Intelligence)で対応していない。
  - とにかく早く高品質な同時通訳を聞きたい。
  - 実装に週単位の時間を割けない。
- 現実的な落としどころは、視点 2 と 3 の中間にあたる (b) である(推定)。端末内 STT で音声を外に出さず、口語化だけを LLM(PCC・自宅 Mac・クラウドのいずれか)に任せる。

## 10. r.txt の記述の検証結果

| r.txt の記述 | 判定 | 根拠 |
|---|---|---|
| Hibiki-Zero は仏・西・葡・独→英のみ | **確認** | モデルカードと GitHub[^hibikizero-hf][^hibikizero-gh] |
| Hibiki-Zero のライセンスは非商用 | **確認** | CC BY-NC-SA 4.0[^hibikizero-hf]。第三者サイトには CC-BY-4.0 とする記載がある[^soniqo]が、一次情報のモデルカードを優先する |
| Hibiki-Zero は 1,000 時間未満の音声データで新しい入力言語を追加できた | **確認** | モデルカード[^hibikizero-hf] |
| Hibiki-Zero は日本語非対応 | **確認**(対応言語に含まれない) | 同上 |
| Apple は Foundation Models を翻訳に勧めず、Translation を案内している(WWDC25 Group Lab) | **確認** | フォーラムのスレッド 792763[^gl25]。補足: WWDC26 Group Lab では、文体が必要な場合などに言語モデルを選ぶ余地を認める発言がある[^gl26] |
| bluetoothHighQualityRecording は標準の音声モードでしか使えない | **確認** | 「only when using the default audio session mode」[^bthq]。補足: EU 非対応、遅延増加、リアルタイム通話には非推奨 |
| bluetoothHighQualityRecording は iOS 26 で追加された | **確認** | iOS 26.0 以降[^bthq] |
| SpeechAnalyzer は端末内だけで動き、長時間の音声も扱える | **確認** | WWDC25 セッション 277[^wwdc25-277] |
| SpeechAnalyzer は日本語に対応している | **一次情報では未確認(二次情報では対応)** | WWDC25 のスライドの引用[^addpipe]。ドキュメント JSON の言語メタデータは根拠にならない[^fm-lang]。実機で確認が必要 |
| SpeechAnalyzer は無料 | **確認**(OS の API で、課金の記載はない) | [^wwdc25-277] |
| iPhone 内蔵の構成だと訳文は標準的な言い回しになる(口調を出しにくい) | **確認**(口調パラメータが無い) | [^ts-config]。補足: iOS 26.4 で、Apple Intelligence を使う `highFidelity` 方式が追加された[^ts-strategy] |
| 自作アプリは、バックグラウンド音声を宣言し、画面を開いた状態で録音を始める必要がある | **一部確認** | `audio` モードの公式説明は再生のみに触れている[^bgmodes]。前面で開始する必要があるという点は開発者フォーラムの報告(二次情報)[^forum-bg]。AudioRecordingIntent には Live Activity が必須(確認)[^audiorecordingintent] |
| AirPods のマイクはビームフォーミングで、Apple もライブ翻訳で iPhone マイクを併用している | **確認** | Newsroom[^newsroom]、サポートページ[^support-lt] |
| 3-2 の構成(端末内 STT + テキストだけクラウド LLM)は安く、プライバシー面でもまし | **妥当(推論)** | 音声を送らないことは構成から明らか。料金は視点 2 の範囲なので未検証 |
| 費用の目安(1 日約 4 ドルなど) | **対象外・未検証** | 視点 2 の範囲 |

## 11. 未確認事項

1. SpeechTranscriber が日本語(ja-JP)に対応していること。実機の `supportedLocales` で確認する。日本語の雑談での認識精度も未確認。
2. 画面ロック中・バックグラウンドで、SpeechAnalyzer・Translation・Foundation Models を長時間連続で動かせるか。電池消費と発熱。
3. AirPods マイク(HFP / 高音質録音)で、離れた周囲の会話をどこまで拾えるか。高音質録音で入力遅延が実際にどれだけ増えるか。
4. Translation framework の `highFidelity` で口語らしさが出るか。翻訳 1 回あたりの遅延。
5. PCC モデルの日次上限の数値、応答遅延、申請の審査状況。日本語→学習言語の口語訳の品質。
6. iOS 27 の新しい端末内モデルの速度(トークン/秒)。
7. AVSpeechSynthesizer の premium 音声の自然さと発声開始の遅延。学習言語ごとの TTS の選択肢(Kokoro の英語以外の品質を含む)。
8. 日本語を入力にする同時音声翻訳(S2ST)のオープンモデルが、SeamlessStreaming 以外にあるか。今回は見つからなかった。Gemma 3n の日本語音声の翻訳品質も未確認。
9. Moonshine に日本語のストリーミングモデルがあるか。parakeet-ja と ReazonSpeech を iOS / MLX で動かした実績。
10. `bluetoothHighQualityRecording` をネイティブの macOS アプリから使えるか。
11. 私的利用が個人情報保護法上の「事業」に当たらないかどうか(本レポートは法的助言ではない)。

---

## 出典(すべて確認日 2026-10-30)

[^speechtranscriber]: Apple Developer, SpeechTranscriber — https://developer.apple.com/documentation/speech/speechtranscriber (確認日 2026-10-30)
[^wwdc25-277]: WWDC25 "Bring advanced speech-to-text to your app with SpeechAnalyzer" — https://developer.apple.com/videos/play/wwdc2025/277/ (確認日 2026-10-30)
[^speechdetector]: Apple Developer, SpeechDetector — https://developer.apple.com/documentation/speech/speechdetector (確認日 2026-10-30)
[^addpipe]: 二次情報: addpipe blog "A Quick Look at Apple's SpeechAnalyzer API"(WWDC25 スライドを引用)— https://blog.addpipe.com/apple-speechanalyzer-api/ (確認日 2026-10-30)
[^issoh]: 二次情報: 一創「SpeechAnalyzerとは?」(Intel Mac・合成音声による個人の測定)— https://www.issoh.co.jp/tech/details/8929/ (確認日 2026-10-30)
[^fm-lang]: Apple Developer, Supporting languages and locales with Foundation Models — https://developer.apple.com/documentation/foundationmodels/supporting-languages-and-locales-with-foundation-models (確認日 2026-10-30)
[^translationsession]: Apple Developer, TranslationSession — https://developer.apple.com/documentation/translation/translationsession (確認日 2026-10-30)
[^ts-init]: Apple Developer, TranslationSession init(installedSource:target:) — https://developer.apple.com/documentation/translation/translationsession/init(installedsource:target:) (確認日 2026-10-30)
[^ts-config]: Apple Developer, TranslationSession.Configuration — https://developer.apple.com/documentation/translation/translationsession/configuration (確認日 2026-10-30)
[^ts-strategy]: Apple Developer, TranslationSession.Strategy — https://developer.apple.com/documentation/translation/translationsession/strategy (確認日 2026-10-30)
[^translation]: Apple Developer, Translation framework — https://developer.apple.com/documentation/translation (確認日 2026-10-30)
[^applewiki]: 二次情報: The Apple Wiki, Translate app — https://theapplewiki.com/wiki/Translate_app (確認日 2026-10-30)
[^gl25]: Apple Developer Forums, WWDC25 Group Lab – Apple Intelligence(DTS Engineer 投稿)— https://developer.apple.com/forums/thread/792763 (確認日 2026-10-30)
[^gl26]: WWDC26 "Coding Intelligence, Machine Learning & AI Group Lab" — https://developer.apple.com/videos/play/wwdc2026/8121/ (確認日 2026-10-30)
[^wwdc26-319]: WWDC26 "Build with the new Apple Foundation Model on Private Cloud Compute" — https://developer.apple.com/videos/play/wwdc2026/319/ (確認日 2026-10-30)
[^wwdc26-241]: WWDC26 "What's new in the Foundation Models framework" — https://developer.apple.com/videos/play/wwdc2026/241/ (確認日 2026-10-30)
[^slm]: Apple Developer, SystemLanguageModel — https://developer.apple.com/documentation/foundationmodels/systemlanguagemodel (確認日 2026-10-30)
[^adapter]: Apple Developer, Foundation Models adapter training — https://developer.apple.com/apple-intelligence/foundation-models-adapter/ (確認日 2026-10-30)
[^afm2024]: Apple Machine Learning Research, Introducing Apple's On-Device and Server Foundation Models (2024) — https://machinelearning.apple.com/research/introducing-apple-foundation-models (確認日 2026-10-30)
[^avss]: Apple Developer, AVSpeechSynthesizer — https://developer.apple.com/documentation/avfaudio/avspeechsynthesizer (確認日 2026-10-30)
[^voicequality]: Apple Developer, AVSpeechSynthesisVoiceQuality — https://developer.apple.com/documentation/avfaudio/avspeechsynthesisvoicequality (確認日 2026-10-30)
[^whisperkit]: argmaxinc/WhisperKit(GitHub)— https://github.com/argmaxinc/WhisperKit (確認日 2026-10-30)
[^kotoba]: kotoba-tech/kotoba-whisper-v2.0(Hugging Face)— https://huggingface.co/kotoba-tech/kotoba-whisper-v2.0 (確認日 2026-10-30)
[^parakeet-ja]: nvidia/parakeet-tdt_ctc-0.6b-ja(Hugging Face)— https://huggingface.co/nvidia/parakeet-tdt_ctc-0.6b-ja (確認日 2026-10-30)
[^reazon]: reazon-research/reazonspeech-k2-v2(Hugging Face)— https://huggingface.co/reazon-research/reazonspeech-k2-v2 (確認日 2026-10-30)
[^sherpa]: 二次情報: sherpa-onnx の配布ページ(iOS はソースからビルド)— https://pub.dev/packages/sherpa_onnx/versions/1.10.41 (確認日 2026-10-30)
[^moonshine-license]: moonshine-ai/moonshine LICENSE(GitHub)— https://github.com/moonshine-ai/moonshine/blob/main/LICENSE (確認日 2026-10-30)
[^hibiki]: kyutai-labs/hibiki(GitHub)— https://github.com/kyutai-labs/hibiki (確認日 2026-10-30)
[^hibikizero-hf]: kyutai/hibiki-zero-3b-pytorch-bf16(Hugging Face モデルカード)— https://huggingface.co/kyutai/hibiki-zero-3b-pytorch-bf16 (確認日 2026-10-30)
[^hibikizero-gh]: kyutai-labs/hibiki-zero(GitHub)— https://github.com/kyutai-labs/hibiki-zero (確認日 2026-10-30)
[^soniqo]: 二次情報: Soniqo "Traducción de voz Hibiki Zero-3B"(MLX 移植の第三者による測定)— https://soniqo.audio/es/guides/audio-translate (確認日 2026-10-30)
[^m4t-readme]: facebookresearch/seamless_communication, SeamlessM4T README(対応言語表)— https://github.com/facebookresearch/seamless_communication/blob/main/docs/m4t/README.md (確認日 2026-10-30)
[^seamless-hf]: facebook/seamless-streaming(Hugging Face)— https://huggingface.co/facebook/seamless-streaming (確認日 2026-10-30)
[^seamless-gh]: facebookresearch/seamless_communication(GitHub)— https://github.com/facebookresearch/seamless_communication (確認日 2026-10-30)
[^gemma3n]: google/gemma-3n-E2B-it(Hugging Face)— https://huggingface.co/google/gemma-3n-E2B-it (確認日 2026-10-30)
[^translategemma]: google/translategemma-4b-it(Hugging Face)— https://huggingface.co/google/translategemma-4b-it (確認日 2026-10-30)
[^llamacpp-bench]: ggml-org/llama.cpp Discussion #4167 "Performance of llama.cpp on Apple Silicon M-series" — https://github.com/ggml-org/llama.cpp/discussions/4167 (確認日 2026-10-30)
[^ertas]: 二次情報(販売業者のブログ): Ertas "LLM run on iPhone benchmarks" — https://www.ertas.ai/blog/llm-run-on-iphone-benchmarks (確認日 2026-10-30)
[^kokoro]: hexgrad/Kokoro-82M(Hugging Face)— https://huggingface.co/hexgrad/Kokoro-82M (確認日 2026-10-30)
[^kokoro-voices]: hexgrad/Kokoro-82M VOICES.md — https://huggingface.co/hexgrad/Kokoro-82M/blob/main/VOICES.md (確認日 2026-10-30)
[^piper]: OHF-Voice/piper1-gpl(GitHub)— https://github.com/OHF-Voice/piper1-gpl (確認日 2026-10-30)
[^a2dp]: Apple Developer, AVAudioSession.CategoryOptions.allowBluetoothA2DP — https://developer.apple.com/documentation/avfaudio/avaudiosession/categoryoptions-swift.struct/allowbluetootha2dp (確認日 2026-10-30)
[^hfp]: Apple Developer, AVAudioSession.CategoryOptions.allowBluetoothHFP — https://developer.apple.com/documentation/avfaudio/avaudiosession/categoryoptions-swift.struct/allowbluetoothhfp (確認日 2026-10-30)
[^bthq]: Apple Developer, AVAudioSession.CategoryOptions.bluetoothHighQualityRecording — https://developer.apple.com/documentation/avfaudio/avaudiosession/categoryoptions-swift.struct/bluetoothhighqualityrecording (確認日 2026-10-30)
[^wwdc25-251]: WWDC25 "Enhance your app's audio recording capabilities" — https://developer.apple.com/videos/play/wwdc2025/251/ (確認日 2026-10-30)
[^newsroom]: Apple Newsroom "AirPods now more versatile with studio-quality audio recording and camera remote"(2025-06)— https://www.apple.com/newsroom/2025/06/airpods-now-more-versatile-with-studio-quality-audio-recording-and-camera-remote/ (確認日 2026-10-30)
[^support-lt]: Apple Support "Use Live Translation with your AirPods" — https://support.apple.com/en-us/123185 (確認日 2026-10-30)
[^bgmodes]: Apple Developer, Configuring background execution modes — https://developer.apple.com/documentation/xcode/configuring-background-execution-modes (確認日 2026-10-30)
[^audiorecordingintent]: Apple Developer, AudioRecordingIntent — https://developer.apple.com/documentation/appintents/audiorecordingintent (確認日 2026-10-30)
[^forum-bg]: 二次情報: Apple Developer Forums(開発者の報告。Apple 職員の回答ではない)— https://developer.apple.com/forums/thread/120038 、https://developer.apple.com/forums/thread/815725 (確認日 2026-10-30)
[^silero]: snakers4/silero-vad(GitHub)— https://github.com/snakers4/silero-vad (確認日 2026-10-30)
[^ppc]: 個人情報保護委員会「個人情報の保護に関する法律についてのガイドライン(通則編)」— https://www.ppc.go.jp/personalinfo/legal/guidelines_tsusoku/ (確認日 2026-10-30)
