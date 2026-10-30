# 02. 既存サービスを組み合わせて実装する場合(視点 2)

- 調査日: 2026-10-30(本文の「確認日」はすべて 2026-10-30)
- 調査: サブエージェント(Opus)。先行調査 `r.txt` は出典未確認の仮説として扱い、一次情報で確認できたものだけを事実として書いた(検証結果は第 8 章)。
- 凡例: **[確認]** 一次情報(公式ドキュメント・公式価格ページ)で確認 / **[二次]** ブログ・ニュース・フォーラムのみ / **[推測]** 筆者の推論 / **[未確認]** 調べたが確認できない

---

## 1. 結論と推奨構成

1. **口調の指示が効く音声→音声の一体型 API は無い。** OpenAI `gpt-realtime-translate` と Gemini `gemini-3.5-live-translate-preview` は、いずれも公式に「指示(プロンプト)不可」と書かれている [確認]。「ラフな口語」を要件どおりに出すには、テキスト段で LLM に口調を指示する**分割型**が必要になる。
2. **推奨構成(本命)**: iOS ネイティブアプリ。**iPhone 内蔵マイクで集音し、AirPods は A2DP の出力専用にする** → 端末内 VAD + STT(Apple SpeechAnalyzer、またはクラウド STT)→ 小型 LLM(Claude Haiku 5.5 / gpt-5-nano など)で「友だち同士の口語で短く」に訳す → ストリーミング TTS(ElevenLabs Flash v2.5 / Gemini 3.8 Flash-Lite TTS / Apple AVSpeechSynthesizer)。
3. **比較の基準(最初に作る)**: `gpt-realtime-translate`(日本語入力に対応、出力は 13 言語、$0.034/分)を Web ページ + 小さなサーバーでつなぎ、遅延と聞き心地の基準にする。
4. **AirPods のマイクは主入力にしない。** HFP が A2DP より優先される [確認]。iOS 26 の高音質録音は default モード限定・EU 非対応で、「リアルタイム通信には非推奨」と明記されている [確認]。Apple 自身もライブ翻訳では iPhone のマイク併用を案内している [確認]。
5. **月額の目安(月 60 時間、発話率 50% を仮定)**: 一体型 約 $61〜$132、指示型 約 $49〜$173+、分割型 約 $1〜$95(TTS の選び方で大きく変わる)。分割型では、LLM の費用は月 $1〜$3 程度に収まる。
6. **オーケストレーション基盤(Vapi / LiveKit Cloud)は、連続稼働だと固定費が目立つ。** PoC で Pipecat(OSS)を Mac 上で使う程度に留める。

---

## 2. 処理の切り分け

```
[集音]──>[VAD/区切り]──>[STT]──>[口語翻訳]──>[TTS]──>[再生]
iPhone内蔵   端末内           日本語    LLM+口調指示   学習言語   AirPods
マイク推奨   (Silero 等 /     文字起こし (短く・口語)    音声      (A2DP)
             SpeechAnalyzer)

一体型 (A):  [集音]──>[ gpt-realtime-translate / Gemini Live Translate ]──>[再生]
              ※ VAD・STT・翻訳・TTS をまとめて行う。口調の指示は不可

指示型 (B):  [集音]──>[ gpt-realtime-2.1(-mini) / Gemini Live + system instruction ]──>[再生]
              ※ 「訳すだけ・口語で」と指示する。応答を自動生成させない設定が要る

分割型 (C):  [集音]──>[VAD]──>[STT]──>[LLM]──>[TTS]──>[再生]
              ※ 段ごとに差し替え可能。STT は端末内にもクラウドにもできる
```

段ごとの要点:

| 段 | 役割 | 本プロジェクトでの論点 |
|---|---|---|
| 集音 | マイクと音声セッションの設定 | AirPods マイク(HFP)か iPhone マイクか(第 5 章)。周囲の遠い声を拾えるか |
| VAD / 区切り | 発話の区間を切る。無音を送らない | 雑談は途中で切れたり重なったりする。区切りが短すぎると訳が細切れになり、長すぎると遅れる |
| STT | 日本語の文字起こし | 遠い声・重なった声での精度。部分結果(途中経過)の速さ |
| 口語翻訳 | 学習言語のラフな口語にする | 指示が効くこと。出力を短く保つこと |
| TTS | 学習言語の音声にする | 最初の音が出るまでの遅延、ストリーミング対応 |
| 再生 | AirPods へ出力 | A2DP(高音質)を保つ。バックグラウンドでの継続 |

---

## 3. 構成パターンの比較

評価は本プロジェクトの要件(周囲の雑談・AirPods・遅延・口語・連続稼働・コスト)に照らしたもの。遅延の数値は、どの公式ページにも具体値が無かった [未確認]。そのため遅延は構造からの [推測] で書く。

| | A 一体型(翻訳特化) | B 指示型(リアルタイム音声 AI) | C 分割型 |
|---|---|---|---|
| 代表 | OpenAI `gpt-realtime-translate` / Gemini `gemini-3.5-live-translate-preview` | OpenAI `gpt-realtime-2.1(-mini)` / Gemini `gemini-3.8-live` 等 + 指示 | STT + LLM + TTS |
| 口語の指示 | **不可** [確認] | 可(system instructions)[確認] | 可(LLM へのプロンプト)。最も細かく制御できる |
| 遅延 | 小さい。発話の途中から訳し始める設計 [確認: 連続処理の記述] | 発話の区切り(VAD)を待ってから応答する [推測] | 区切り待ち + 3 段の合計で、最も遅くなりやすい [推測] |
| 勝手な応答 | 起きない(訳すだけ)[確認] | 起こりうる。自動応答を止める設定が要る [確認: OpenAI の `create_response:false`] | 起きない(LLM の出力を訳文のみに制限する)[推測] |
| 日本語入力 | OpenAI: 70+ 言語の入力に日本語あり [確認] / Gemini: 言語表に `ja` あり(入出力の区別は記載なし)[確認] | 対応 [確認: Gemini の言語表に ja] | STT 次第(SpeechAnalyzer, Deepgram, AssemblyAI U-3.6 は日本語あり) |
| 出力言語 | OpenAI は 13 言語のみ [確認] / Gemini は 70+ [確認] | 多数 | TTS 次第(ElevenLabs Flash v2.5 は 32 言語) |
| 連続稼働 | OpenAI はセッション上限が [未確認]。無音も送り続ける前提 [確認] | OpenAI Realtime は 1 セッション 60 分まで [確認]、Gemini は 1 接続が約 10 分 [確認] で再接続が必要 | 端末内 STT なら接続の制約は LLM/TTS の単発リクエストだけになる |
| 月額(第 6 章) | 約 $61〜$132 | 約 $49〜$173 以上(文脈の蓄積で増える) | 約 $1〜$95 |
| 実装の手間 | 最小。ブラウザ + 短期キーの発行サーバーで動く | 中程度。応答制御とプロンプトの調整 | 最大。区切り・バッファ・各段の接続 |
| 周囲の人のプライバシー | 音声をクラウドへ送り続ける | 同左 | 端末内 STT なら送るのはテキストだけ |

**判断**: 要件の中心は「ラフな口語」なので、A は口調の点で要件を満たしきれない。A は遅延・聞き心地の**基準**として使い、本命は C とする。B は「1 社で完結させたい」場合の代替で、応答を制御するコストと価格が見合うかを PoC で判断する。

---

## 4. 各段の候補サービス詳細

確認日はすべて 2026-10-30。価格は USD。

### 4.1 一体型(音声→音声の翻訳 API)

**OpenAI `gpt-realtime-translate`**
- 「ストリーミングの speech-to-speech 翻訳モデル」。入力音声の受信中に、訳した音声と文字起こしの差分(delta)を返す。専用エンドポイントは `v1/realtime/translations` で、通常の Realtime エンドポイントでは使えない。[確認] https://developers.openai.com/api/docs/models/gpt-realtime-translate
- 価格は **$0.034/分**(音声の長さで課金)[確認] https://developers.openai.com/api/docs/pricing
- 入力は 70 以上の言語で、言語は自動判別。出力は **13 言語**(Spanish, Portuguese, French, Japanese, Russian, Chinese, German, Korean, Hindi, Indonesian, Vietnamese, Italian, English)。カスタムのプロンプトと声の選択は不可。訳の音声は元の話者の口調・声の高さに追従し、話者が変わると訳の声も変わる。セッションは turn 制ではなく、無音も含めて音声を送り続けるよう案内されている [確認] https://developers.openai.com/cookbook/examples/voice_solutions/realtime_translation_guide
- 接続: ブラウザは WebRTC(サーバーが `client_secrets` で短期の鍵を発行する)、サーバーは WebSocket(24 kHz PCM16)。入力側と訳文側の文字起こしイベントあり [確認] https://developers.openai.com/api/docs/guides/realtime-translation
- 公開日: Microsoft Learn の同モデルの解説ページの日付が 2026-05-07 [確認] https://learn.microsoft.com/en-us/azure/foundry-classic/openai/concepts/gpt-realtime-translate 。OpenAI の発表ページ(openai.com)は取得時に 403 で、直接は確認できなかった。
- 本プロジェクトでの評価: 日本語→英語などの「聞き流し」の基準としては最有力。ただし口調は指定できず、学習言語が 13 言語に入っていない場合は使えない。無音を送らない運用が課金と品質にどう効くかは [未確認]。

**Google `gemini-3.5-live-translate-preview`(Gemini 3.5 Live Translate)**
- 「70+ 言語間の低遅延リアルタイム speech-to-speech 翻訳」。発話の区切りを待たずに訳す。入力は音声のみ。設定できるのは `targetLanguageCode` と `echoTargetLanguage` だけで、「tools や instructions は非対応」[確認] https://ai.google.dev/gemini-api/docs/live-api/live-translate
- 雑音について: ノイズや音楽を除くよう設計されているが、すべての背景音を無視できるとは限らない。複数人が素早く話すと声が 1 つに固定されることがある [確認](同上)
- 価格: 入力 $3.50/1M 音声トークン(**$0.0053/分**)、出力 $21.00/1M(**$0.0315/分**)。音声は 25 トークン/秒。無料枠あり(ただし無料枠のデータは製品改善に使われる)[確認] https://ai.google.dev/gemini-api/docs/pricing
- 接続: WebSocket。クライアントから使う場合は ephemeral token(v1beta)[確認] 上記 live-translate ページ
- 本プロジェクトでの評価: 出力言語が多いので、学習言語の選択肢は OpenAI より広い。周囲の人の音声を送ることになるので、**無料枠は使わない**(データが製品改善に使われるため)。

**その他**: Azure Speech の Live Interpreter(入力言語の自動判別・途中での言語切り替え・personal voice)がある [確認(概要のみ)] https://learn.microsoft.com/en-us/azure/ai-services/speech-service/speech-translation 。価格は公式価格ページで数値が表示されなかった [未確認]。口調の指示についての記述は見つからなかった。

### 4.2 指示型(リアルタイム音声 AI + 指示)

**OpenAI `gpt-realtime-2.1` / `gpt-realtime-2.1-mini`**
- `gpt-realtime-2.1`: 音声 入力 $32 / 出力 $64 / キャッシュ入力 $0.40(1M トークンあたり)。テキスト 入力 $4 / 出力 $24。`-mini` は音声 $10 / $20 [確認] https://developers.openai.com/api/docs/pricing
- 2.1 は 2 の改良版(無音・雑音の扱い、割り込みの挙動など)。コンテキストは 128k [確認] https://developers.openai.com/api/docs/models/gpt-realtime-2.1
- 1 セッションの上限は 60 分。VAD を有効にしたまま `turn_detection.create_response=false` / `interrupt_response=false` にすると自動応答を止められ、クライアントから `response.create` で応答を起こせる [確認] https://developers.openai.com/api/docs/guides/realtime-conversations
- 音声トークンの換算: ユーザー音声 1 トークン/100ms、モデル音声 1 トークン/50ms。VAD が空の入力を除くので、無音は入力トークンにならない。ここは公式ページ(realtime-costs)の検索結果の抜粋で確認した。ページ本体は取得時に 404 で、直接は読めていない [一次情報の抜粋のみ] https://developers.openai.com/api/docs/guides/realtime-costs
- 雑音低減 `noise_reduction` に `near_field` / `far_field` がある。ただし確認できたのは SDK の型定義(OpenAPI から生成されたもの)だけ [二次] https://www.rubydoc.info/gems/openai/OpenAI/Models/Realtime/RealtimeSession/InputAudioNoiseReduction 。iPhone 内蔵マイクで遠い声を拾うなら `far_field` が合うと考えられる [推測]。
- 評価: 指示は効くが、会話モデルなので「答えてしまう」危険がある。毎ターン文脈が積み上がり、その分の入力も課金される(第 6 章)。

**OpenAI `gpt-live-1`(GPT-Live)**: 全二重の会話モデル。$0.05/分(秒単位で課金)、エンドポイントは `v1/live/sessions` [確認] https://developers.openai.com/api/docs/models/gpt-live-1 。`instructions` を指定できることは Microsoft Learn の参照ページで確認した [二次(Azure 側の公式)] https://learn.microsoft.com/azure/foundry/openai/gpt-live-reference 。会話向けの設計なので、通訳専用に使えるかは [未確認]。

**Google Gemini Live(`gemini-3.8-live` / `gemini-3.1-flash-live-preview` ほか)**
- 価格: 音声 入力 $3.00/1M(**$0.005/分**)、出力 $12.00/1M(**$0.018/分**)[確認] https://ai.google.dev/gemini-api/docs/pricing
- system instructions で話す言語を制限できる。日本語(`ja`)対応。自動 VAD は無効化・感度調整ができる。「関係ない内容には応答しない」proactive audio は `gemini-3.1-flash-live` では非対応 [確認] https://ai.google.dev/gemini-api/docs/live-api/capabilities
- セッション: 圧縮なしの音声セッションは 15 分まで。コンテキストの圧縮で無制限に延ばせる。1 接続は約 10 分で、再開トークンの有効期限は 2 時間 [確認] https://ai.google.dev/gemini-api/docs/live-api/session-management
- 評価: 指示型の中では価格が最も安い。約 10 分ごとの再接続処理が必須。翻訳に限定させる指示がどこまで安定するかは [未確認]。

### 4.3 STT(日本語の文字起こし)

| サービス | 日本語 | ストリーミング | 価格 [確認日 2026-10-30] | 本プロジェクトでの評価 |
|---|---|---|---|---|
| **Apple SpeechAnalyzer / `SpeechTranscriber`** | ドキュメントのメタデータに `ja-JP` あり。正式な対応可否は実行時に `supportedLocales` で確認する [確認] | 結果を非同期シーケンスで順に返す [確認] | API 料金なし(OS の機能) | iOS 26 以降。通信が要らずプライバシー面で最良。遠い声・雑談での精度、バックグラウンドで動くか、長時間の安定性は [未確認](実測が必要)。https://developer.apple.com/documentation/speech/speechtranscriber |
| **Deepgram Nova-3** | monolingual に `ja`、multilingual にも日本語あり [確認] | あり。日本語がストリーミング対象かはページ上で明示されていない [未確認] | Nova-3 Mono $0.0077/分、Multi $0.0092/分。初回 $200 分のクレジット [確認] | 安い。無音を課金しないとの FAQ もあるが、ストリーミングでの扱いは要確認。https://deepgram.com/pricing / https://developers.deepgram.com/docs/models-languages-overview |
| **AssemblyAI Universal-3.6 Pro Realtime** | 日本語ページで、日本語対応のリアルタイムモデルとして記載。Universal-Streaming Multilingual は日本語非対応 [確認] | あり | $0.45/時。**接続している時間で課金(アイドル時間も含む)** [確認] | 無音をクライアント側で止めても、接続している限り課金される。https://www.assemblyai.com/pricing / https://www.assemblyai.com/languages/japanese |
| **OpenAI `gpt-live-transcribe` / `gpt-realtime-whisper`** | ISO 639-1 の `languages` 指定に対応。日本語を名指しした記述は無い [未確認] | あり。`delay`(minimal〜xhigh)で遅延と精度を調整できる。サーバー側の VAD は非対応なので、クライアント側の VAD と commit が必要 [確認] | どちらも $0.017/分 [確認] | ガイドでは `gpt-live-transcribe` が推奨。https://developers.openai.com/api/docs/guides/realtime-transcription |
| **Gemini 3.5 Transcribe Live** | (Live API の言語表に ja あり) | あり | 入力 $0.005/分 + 出力。実効で約 $0.009/分 [確認] | https://ai.google.dev/gemini-api/docs/pricing |
| **ElevenLabs Scribe v2 Realtime** | [未確認] | あり | $0.39/時 [確認] | https://elevenlabs.io/pricing/api |
| **Google Cloud STT / Azure Speech** | 対応(一般に知られている)[未確認] | あり | Google: 価格ページの本文を取得できず [未確認]。Azure: 公式ページは数値が「$-」と表示され、二次情報では $1/時 [二次] | 個人開発での利点は小さい |

### 4.4 口語翻訳(LLM / DeepL)

- 指示の実現性: LLM は system prompt で「友だち同士の口語・短く・訳文のみ」を指定できる。これは LLM の基本機能 [確認(各社の API 仕様)]。実際にどれだけ口語らしくなるかは、言語ごとに聞いて確かめる必要がある [未確認]。
- 価格(1M トークンあたり 入力/出力)[確認]:
  - Claude Haiku 5.5: $0.10 / $0.50(プロンプト 10 万トークン以下)。Claude 4.7 以降は新しいトークナイザで、同じ文章でトークン数が約 30% 増える https://platform.claude.com/docs/en/about-claude/pricing
  - Claude Sonnet 5.5: $2 / $10(同上)
  - OpenAI gpt-5-nano: $0.05 / $0.40、gpt-5-mini: $0.25 / $2.00 https://developers.openai.com/api/docs/pricing
  - Gemini 3.5 Flash-Lite: $0.30 / $2.50 https://ai.google.dev/gemini-api/docs/pricing
- DeepL API: `formality` に `less` / `prefer_less`(「よりくだけた言い方」、対応はターゲット言語の一部のみ)。`custom_instructions`(最大 10 個 × 300 文字、例: "Use a friendly, diplomatic tone")、`context`、`model_type`(`latency_optimized` あり)がある [確認] https://developers.deepl.com/api-reference/translate 。スラングまで出せるかは記載が無い [未確認]。API の価格は公式ページで数値を取得できなかった(二次情報では旧 Pro $5.49/月 + $25/100 万文字、新 Growth プラン、などの記述がある)[二次]。
- 評価: 1 件ずつの短い訳なら、LLM の費用は月 $1〜$3(第 6 章)。口調の制御力と費用の両面で LLM が DeepL より有利 [推測]。遅延の面では、推論(reasoning)をしない・最小にする設定と、ストリーミング出力が要る [推測]。

### 4.5 TTS(学習言語の音声)

| サービス | 遅延 | 口調の指示 | 価格 [確認日 2026-10-30] | 備考 |
|---|---|---|---|---|
| **ElevenLabs Flash v2.5** | 約 75ms(アプリ・ネットワークの遅延を除く)、32 言語 [確認] | 声の選択で調整 | Flash/Turbo $0.04/1K 文字。Pro $99/月に 247.5 万文字込み(約 1K 文字 ≒ 1 分)[確認] | 公式の選定ガイドでは、低遅延向けに `eleven_v4_turbo`(約 100ms)を推奨 [確認]。https://elevenlabs.io/docs/overview/models / https://elevenlabs.io/pricing/api |
| **Cartesia Sonic** | 価格ページに記載なし [未確認] | — | 1 文字 1 クレジット。Startup $49/月で 125 万クレジット、超過は約 $0.045/1K(価格表からの計算)[確認(計算)] | https://cartesia.ai/pricing |
| **Deepgram Aura-2** | [未確認] | — | $0.030/1K 文字 [確認] | https://deepgram.com/pricing |
| **OpenAI `gpt-4o-mini-tts`** | ストリーミング対応。低遅延なら `pcm` / `wav` を推奨 [確認] | **`instructions` で口調・抑揚を指示できる** [確認] | テキスト入力 $0.60/1M、音声出力 $12/1M [確認] | **非推奨。2027-01-06 に停止予定**。移行先は Realtime API の `gpt-realtime-2.1-mini` [確認] https://developers.openai.com/api/docs/guides/text-to-speech |
| **Gemini 3.8 Flash-Lite TTS** | [未確認] | (Gemini TTS は自然言語で話し方を指示できるとされるが未確認) | 出力 $6/1M 音声トークン(2026-12-31 まで)→ 2027-01-01 から $12 [確認] | 25 トークン/秒 = 約 $0.009/分(2026 年中)https://ai.google.dev/gemini-api/docs/pricing |
| **Azure Neural TTS** | [未確認] | SSML | 公式は「$-」表示で、二次情報では $15〜16/1M 文字 [二次] | 無料枠 50 万文字/月 [確認] |
| **Apple AVSpeechSynthesizer** | 端末内で処理 | 不可(声の選択のみ)[推測] | 無料 | 声は合成音声らしさが強いが、聞き流しの基準としては使える [推測] |

### 4.6 つなぎ方とオーケストレーション基盤

**Web(Safari)+ 小さなサーバー**
- サーバーは通常の API キーで短期の client secret を発行し、ブラウザはそれで WebRTC の SDP を交換する。これが OpenAI の推奨経路 [確認] https://developers.openai.com/api/docs/guides/realtime-translation 。Gemini も ephemeral token を推奨している [確認] https://ai.google.dev/gemini-api/docs/live-api
- 利点: 実装が最短で、A/B の PoC に向く。欠点: 音声セッションを制御できない(AirPods マイクを避けて iPhone マイクを選べるか不明 [未確認])。画面ロック中の継続も不安定(第 5 章)。

**iOS ネイティブアプリ**
- `AVAudioSession` で入出力の経路を決められる(第 5 章)。Background Modes の `audio` を使える。SpeechAnalyzer を端末内で使える。
- 本命の C はネイティブ前提。クラウドの鍵は短期トークンを発行するサーバー経由にする(Web と同じ考え方)[推測]。

**オーケストレーション基盤**

| 基盤 | 内容 [確認日 2026-10-30] | 個人開発・本用途での評価 |
|---|---|---|
| **Pipecat** | BSD-2 の OSS、Python。150 以上の AI サービスに対応し、ホスティング先は自由。クライアント SDK に iOS あり [確認] https://docs.pipecat.ai/getting-started/introduction | **Mac 上で C の PoC を数時間で組むのに有用**。本番はサーバー経由の構成になるため、端末内 STT の方針とは合わない [推測] |
| **LiveKit Cloud(Agents)** | 無料の Build プランはエージェント 1,000 分/月。Ship $50/月で 5,000 分、超過 $0.01/分 [確認] https://livekit.com/pricing | 月 3,600 分の連続稼働は無料枠を超える。WebRTC 基盤としては堅いが、1 人で使うには過剰 [推測] |
| **Vapi** | ホスティング $0.05/分 + モデル原価。電話・音声エージェント向け [確認] https://vapi.ai/pricing | 月 3,600 分でホスティング代だけで $180。用途(応答するエージェント)も合わない → 不採用 |

---

## 5. マイク・Bluetooth・iOS 連続稼働の制約

### 5.1 AirPods のマイクを入力に使う場合(HFP)
- `allowBluetoothHFP` は「HFP 機器を音声入力として使えるようにする」オプションで、`record` / `playAndRecord` でのみ使える [確認] https://developer.apple.com/documentation/avfaudio/avaudiosession/categoryoptions-swift.struct/allowbluetoothhfp
- A2DP は「ステレオ、**出力専用**」のプロファイル。HFP と A2DP の両方を許可した場合、両方に対応する機器では **HFP が優先**される [確認] https://developer.apple.com/documentation/avfaudio/avaudiosession/categoryoptions-swift.struct/allowbluetootha2dp
- つまり AirPods のマイクを使うと経路は HFP になり、再生も HFP 側を通る。HFP が狭帯域で音質が落ちることは一般に知られている [推測(Apple の文書に数値の記載は無い)]。高音質録音のオプションが「full-bandwidth を有効にする」と説明されていることから、既定は全帯域ではないと読める [推測]。

### 5.2 iOS 26 の `bluetoothHighQualityRecording`
- iOS 26.0 以降。経路が対応していれば(「一部の AirPods」)入出力を full-bandwidth にする。`allowBluetoothHFP` と併用すると、非対応の経路では HFP にフォールバックする [確認]
- 条件と注意 [確認] https://developer.apple.com/documentation/avfaudio/avaudiosession/categoryoptions-swift.struct/bluetoothhighqualityrecording
  - **audio session のモードが `default` のときだけ**要求できる
  - **EU では現在非対応**
  - **入力遅延が増える可能性があり、リアルタイム通信用途には推奨されない**
  - 対応可否は実行時に `bluetoothMicrophoneExtension.highQualityRecording.isSupported` で確認する
- 対応機種: 発表時点(2025-06)で AirPods 4、AirPods 4(ANC)、AirPods Pro 2 [確認] https://www.apple.com/newsroom/2025/06/airpods-now-more-versatile-with-studio-quality-audio-recording-and-camera-remote/ 。それより後の機種(AirPods Pro 3、AirPods 5 など)は [未確認]。
- 評価: 遅延を重視する本用途では、Apple 自身が非推奨としている。採用しない。

### 5.3 AirPods のマイクは周囲の声に向くか
- AirPods(H2)は「ビームフォーミングマイク」と「声を分離する機能(Voice Isolation)」を持ち、装着者の声をきれいに録る用途で案内されている [確認] 上記 Newsroom
- Apple の AirPods ライブ翻訳でも、騒がしい場所では「AirPods のマイクに加えて iPhone のマイクを使う」こと、「iPhone を話している相手に近づける」ことが案内されている [確認] https://support.apple.com/en-us/123185
- 推論: 装着者以外の、少し離れた雑談を AirPods のマイクだけで拾うのは不利 [推測]。実測で確かめる。

### 5.4 推奨: iPhone 内蔵マイクで拾い、AirPods で聞く
- 設定: `playAndRecord` + `allowBluetoothA2DP`(`allowBluetoothHFP` は付けない)。`playAndRecord` は iOS 10 以降、出力を A2DP 機器に回せる [確認](上記 allowBluetoothA2DP)。A2DP は出力専用なので、入力は内蔵マイクになる [推測(実機で確認する)]。
- 利点: 再生は A2DP の高音質のまま。マイクは iPhone なので、テーブルに置く・胸ポケットに入れるなど、置き場所を調整できる。出力は耳の中なので、iPhone のマイクに訳の音声が回り込みにくい [推測]。
- 試すこと: モード(`default` / `measurement` / `voiceChat`)によって、声の処理(エコー除去・自動ゲイン)が遠い声を削らないか [未確認]。

### 5.5 iOS での連続稼働
**ネイティブアプリ**
- Background Modes の `audio` は「バックグラウンドで音声コンテンツを再生する」モードとして説明されている。録音の継続についての明記は見つからなかった [確認(記述の範囲)] https://developer.apple.com/documentation/xcode/configuring-background-execution-modes
- バックグラウンドから録音を**開始**すると `CannotStartRecording` で失敗する、という報告が Apple Developer Forums にある。前面で開始したセッションを継続する作りが前提になる [二次(開発者フォーラム)] https://developer.apple.com/forums/thread/120038
- 電話・Siri・アラームによる割り込みからの復帰処理が要る [推測]。画面ロック中に 2 時間続けて録音・再生できるかは実機での計測が必須 [未確認]。

**Web(Safari)**
- WebKit Bug 204681: iOS 13 の Safari で、バックグラウンドに 30 秒以上置くと音声トラックが `ended` になる。状態は NEW(未解決、最終更新 2022-07)[確認(バグ報告)] https://bugs.webkit.org/show_bug.cgi?id=204681
- WebKit Bug 226620: WKWebView では `UIBackgroundModes` に audio が無いとミュートされる。ホーム画面に追加した Web アプリでは、バックグラウンドでマイクが止まるという報告(2024 年のコメント)[確認(バグ報告)] https://bugs.webkit.org/show_bug.cgi?id=226620
- iOS 26 の Safari で画面ロック中にマイクと再生が続くかは [未確認]。Web は「画面を点けたまま使う PoC 用」と位置づける。

### 5.6 クラウド側のセッション上限(連続稼働に影響)
- OpenAI Realtime: 1 セッション 60 分まで [確認]。翻訳エンドポイント(`/v1/realtime/translations`)の上限は記載が無い [未確認]。
- Gemini Live: 1 接続は約 10 分。圧縮なしの音声セッションは 15 分まで(圧縮すれば無制限)。再開トークンは 2 時間有効 [確認]。Live Translate に同じ上限が当てはまるかは [未確認]。
- → A/B は、2 時間連続で使うなら自動再接続が必須。

---

## 6. コスト試算(月 60 時間 = 3,600 分)

### 6.1 前提(仮定)
- 稼働 1 日 2 時間 × 30 日 = **3,600 分/月**。
- **発話率 50%**(実際に誰かが話している時間 = 1,800 分)。雑談の場によって大きく変わる [仮定]。
- 訳の音声の長さ ≒ 元の発話の長さ [仮定]。
- 分割型の LLM: 発話 10 秒ごとに 1 回呼ぶ(1,800 分 × 6 = 10,800 回)。1 回あたり入力 600 トークン(指示 + 直前の文脈 + 本文)、出力 40 トークン [仮定]。TTS は 1 回 150 文字(→ 162 万文字/月、ElevenLabs の換算で約 1,620 分の音声)[仮定]。
- 計算スクリプト: 作業用ディレクトリに置いた使い捨てのもの(リポジトリ外)。式は下にすべて書いた。
- 単価はすべて第 4 章の公式ページで確認(確認日 2026-10-30)。

### 6.2 代表 3 構成

**構成 A: OpenAI `gpt-realtime-translate`(一体型)**
- 無音も送り続ける場合(ガイドの推奨どおり): 3,600 分 × $0.034 = **$122.40/月**
- 発話区間だけ送れた場合: 1,800 分 × $0.034 = **$61.20/月**(−50%)
- 注意: 課金が「送った音声の長さ」か「接続していた時間」かは公式に明記が無い [未確認]。また、送るのを止めると訳の品質が落ちる可能性がある(ガイドは送り続けることを推奨)。
- 参考(A'): Gemini 3.5 Live Translate。入力 3,600 × $0.00525 + 出力 1,800 × $0.0315 = $18.90 + $56.70 = **$75.60/月**。入力も絞れれば 1,800 × ($0.00525 + $0.0315) = **$66.15/月**。最悪ケース(出力も 3,600 分)は $132.30。
  - 1 分あたりの単価: 25 トークン/秒 × 60 = 1,500 トークン/分。$3.50 × 1,500 / 1M = $0.00525/分(公式表示 $0.0053)、$21 × 1,500 / 1M = $0.0315/分。

**構成 B: 指示型(Gemini Live / OpenAI Realtime + 「訳すだけ・口語で」)**
- Gemini Live(3.8 Live / 3.1 Flash Live): 入力 3,600 分 × $0.005 + 出力 1,800 分 × $0.018 = $18.00 + $32.40 = **$50.40/月**
- OpenAI `gpt-realtime-2.1-mini`: 入力 600 トークン/分 × $10/1M = $0.006/分、出力 1,200 トークン/分 × $20/1M = $0.024/分。→ 1,800 × ($0.006 + $0.024) = **$54.00/月**(下限)
- OpenAI `gpt-realtime-2.1`: 入力 600 × $32/1M = $0.0192/分、出力 1,200 × $64/1M = $0.0768/分。→ 1,800 × $0.096 = **$172.80/月**(下限)
- 注意: これらは文脈の蓄積を除いた下限。Realtime はターンごとに過去の会話を入力として再処理するので、実際はこれより増える(キャッシュ入力 $0.40/1M で緩和される)。`retention_ratio` や文脈の切り詰めで抑える。Gemini Live の文脈課金の扱いは [未確認]。

**構成 C: 分割型(端末内 STT + 小型 LLM + クラウド TTS)**
- STT: Apple SpeechAnalyzer = **$0**
- LLM: Claude Haiku 5.5 = 6.48M × $0.10/1M + 0.432M × $0.50/1M = $0.648 + $0.216 = **$0.86/月**
  - 入力トークン = 10,800 回 × 600 = 6.48M、出力 = 10,800 × 40 = 0.432M
  - 他のモデル: gpt-5-nano $0.50、gpt-5-mini $2.48(推論トークンが出力に加わればさらに増える)、Gemini 3.5 Flash-Lite $3.02、Claude Sonnet 5.5 $17.28
- TTS(162 万文字/月 ≒ 1,620 分):
  - ElevenLabs Flash: 1,620K 文字 × $0.04/1K = **$64.80**(またはプラン料金。Pro $99/月なら 247.5 万文字込み)
  - Deepgram Aura-2: 1,620 × $0.030 = $48.60
  - Cartesia: Startup $49 + (162 万 − 125 万)× $0.045/1K = $65.65
  - Gemini 3.8 Flash-Lite TTS: 1,620 分 × 1,500 トークン × $6/1M = **$14.58**(2027 年からは $12/1M で $29.16)
  - `gpt-realtime-2.1-mini` の音声出力: 1,620 × 1,200 × $20/1M = $38.88
  - Apple AVSpeechSynthesizer: $0
- **合計**: Apple STT + Haiku 5.5 + ElevenLabs = **約 $66/月**(Pro プランなら約 $100)。Apple STT + Haiku 5.5 + Gemini Flash-Lite TTS = **約 $15/月**。すべて Apple + Haiku = **約 $1/月**。
- クラウド STT を使う場合の加算: Deepgram Nova-3 Mono 3,600 × $0.0077 = $27.72(発話だけなら 1,800 分で $13.86)。`gpt-live-transcribe` 3,600 × $0.017 = $61.20(同 $30.60)。AssemblyAI U-3.6 Pro Realtime は 60 時間 × $0.45 = $27.00(接続時間で課金されるので、無音を止めても減らない)。

### 6.3 無音を送らない工夫の効果(概算)
- 1 分単位の音声課金(A、クラウド STT): 発話率 50% なら約 50% 減る。発話率 30% の静かな場なら約 70% 減る。ただし [未確認] の点がある(OpenAI 翻訳の課金の基準、品質への影響、Deepgram のストリーミングで無音が課金されるか)。
- 接続時間で課金されるもの(AssemblyAI ストリーミング、LiveKit / Vapi): 送るのを止めても減らない。長い無音のときは**接続を切る**必要がある。
- B: OpenAI は VAD が空の入力を除く(公式ページの抜粋)ので、元から発話分だけになる。Gemini Live は入力の課金が送った音声分なので、送るのを止めれば減る [推測]。
- C: 端末内 STT なら STT 費はゼロ。LLM と TTS はもともと発話がある時だけ動くので、無音対策の効果は STT をクラウドにした場合にだけ効く。

---

## 7. PoC の順番

数時間単位で区切り、各段階で 1 つずつ問いに答える。音源は**テレビ・YouTube の日本語の雑談番組**を使い、周囲の他人の音声は録らない(プライバシーのため)。

| 段階 | 作るもの(目安時間) | 確かめること | 合格の目安(仮) |
|---|---|---|---|
| **P0** | コードなし(30 分)。iPhone のボイスメモで「iPhone マイク」と「AirPods マイク」を交互に使い、2〜3 m 先の雑談音を録って聞き比べる | AirPods マイクで周囲の声を拾えるか(第 5 章の仮説)。録音は確認後に削除する | 内蔵マイクの方が明らかに聞き取りやすければ、内蔵マイクの方針で確定 |
| **P1** | Mac のブラウザ + 短期キーを発行する小さなサーバー → `gpt-realtime-translate`(WebRTC)(2〜3 時間) | 遅延(発話開始から訳の音声まで)、日本語の雑談での訳の自然さ、話者の切り替わり、無音時の挙動 | 遅延の基準値を記録する。「聞き流せるか」を主観で判定 |
| **P1'** | 同じ構成で Gemini 3.5 Live Translate(WebSocket、有料枠)(1〜2 時間) | P1 との比較。学習言語が OpenAI の 13 言語外なら、こちらが主になる | 同上 |
| **P2** | Mac 上で Pipecat により分割型を組む: STT(Deepgram または `gpt-live-transcribe`)→ Claude Haiku 5.5(口語プロンプト)→ ElevenLabs Flash(3〜4 時間) | 口語らしさの差(A との聞き比べ)、区切りの長さと遅延の関係、1 分あたりの実費 | A より口語らしく、遅延が許容範囲(例: 発話の終わりから 2 秒以内)[仮の目安] |
| **P3** | iOS ネイティブの最小アプリ: `playAndRecord` + `allowBluetoothA2DP` で内蔵マイク → AirPods。SpeechAnalyzer の日本語で文字起こしを表示(半日) | 経路が本当に内蔵マイク + A2DP になるか。SpeechAnalyzer の精度と速さ。**画面ロック中に 2 時間続くか** | ロック中の 2 時間で欠落なし。電話などの割り込み後に復帰する |
| **P4** | P3 に P2 の LLM・TTS をつなぐ(半日〜1 日) | 実環境(カフェ等)での総合評価、電池消費、月額の実測 | — |
| (任意) **P5** | 指示型 B(Gemini Live + system instruction、自動応答の抑制)(2〜3 時間) | 「訳すだけ」を守れるか。C より遅延・費用で優るか | 勝手な応答が 1 時間に 0 回 |

---

## 8. r.txt の記述の検証結果

対象は主に r.txt 第 2 章と、第 3 章のうち本視点に関わる記述。確認日 2026-10-30。

| # | r.txt の記述 | 判定 | 根拠・補足 |
|---|---|---|---|
| 1 | GPT-Realtime-Translate(2026 年 5 月公開) | **確認(傍証)** | モデルの存在は OpenAI の公式ドキュメントで確認。公開日は Microsoft Learn の解説ページの日付(2026-05-07)と複数のニュースによる。OpenAI の発表ページは 403 で直接は見られていない |
| 2 | 料金 1 分 $0.034(1 時間で約 $2) | **確認** | https://developers.openai.com/api/docs/pricing ($0.034 × 60 = $2.04) |
| 3 | 70 以上の言語から 13 言語へ。出力に日本語・英語・スペイン語・中国語 | **確認** | OpenAI Cookbook に 13 言語の一覧あり |
| 4 | 元の言語は自動判別、区切りを気にせず流しっぱなしで訳す | **確認** | Cookbook:「自動で元言語を検出」「turn 制ではない、無音も送り続ける」 |
| 5 | 別の人が話し始めると訳の声も切り替わる | **確認** | Cookbook:「新しい話者の音声が来ると訳の声が変わる」。「雑談向き」は r.txt の評価で、未検証 |
| 6 | A は口調を指定できない | **確認** | OpenAI:「カスタムのプロンプト非対応」、Gemini:「instructions 非対応」 |
| 7 | Gemini 3.5 Live Translate が存在する | **確認** | `gemini-3.5-live-translate-preview`(preview) |
| 8 | 話し終わりを待たずに訳し続ける | **確認** | 公式:「Translates as the speaker talks without waiting for turns」 |
| 9 | 元の話者から数秒遅れで音声を返す | **確認できず** | 公式ページに遅延の数値は無い |
| 10 | 日本語に対応、指示は受け付けない、背景の雑音は完全には除けない | **確認** | 言語表に `ja`(入出力の区別は記載なし)。instructions 非対応。雑音の記述あり |
| 11 | GPT-Realtime-2 は音声入力 $32 / 出力 $64(1M トークン) | **確認** | 価格ページ。後継の `gpt-realtime-2.1` も同じ価格で、`-2.1-mini` は $10 / $20 |
| 12 | B は A より割高 | **条件付きで確認** | 計算上、`gpt-realtime-2.1` は下限でも $173/月で A($61〜$122)より高い。ただし `-mini`($54〜)や Gemini Live($50)は A と同程度以下 |
| 13 | B は勝手に返事をする恐れ | **推測(妥当)** | 会話モデルなので起こりうる。OpenAI には自動応答を止める設定がある(公式) |
| 14 | SpeechAnalyzer は日本語対応・端末内・長時間対応・無料 | **一部確認** | iOS 26 以降の API としては確認。日本語はメタデータに `ja-JP`(正式には `supportedLocales` で判定)。端末内はモデルのダウンロードの記述から読み取れるが、明記は無い。「長時間の音声」は SpeechTranscriber のページに記載が無い(r.txt の出典は二次ブログ)。API 料金は無い |
| 15 | GPT-Realtime-Whisper 1 分 $0.017 | **確認** | 価格ページとモデルページ。なお現行ガイドでは同じ価格の `gpt-live-transcribe` が推奨 |
| 16 | gpt-4o-mini-tts は読み方を言葉で指示できる | **確認(ただし重要な補足)** | `instructions` は使える。ただし**非推奨で、2027-01-06 に停止予定**。r.txt には書かれていない |
| 17 | ElevenLabs Flash v2.5 は遅延が非常に小さく、32 言語 | **確認** | 約 75ms、32 言語。ただし公式の低遅延推奨は `eleven_v4_turbo` |
| 18 | ブラウザは短期の鍵 + WebRTC が推奨 | **確認** | OpenAI の Realtime translation ガイド |
| 19 | AirPods のマイクはビームフォーミング。Apple もライブ翻訳で iPhone のマイクを併用 | **確認** | Newsroom(ビームフォーミングマイク)。サポート記事(iPhone マイクの併用、iPhone を相手に近づける) |
| 20 | bluetoothHighQualityRecording は標準の音声モードでしか使えない | **確認(不足あり)** | default モード限定は正しい。加えて **EU 非対応**、**入力遅延が増える可能性、リアルタイム通信には非推奨**、という重要な条件が r.txt には無い |
| 21 | 費用: A で 1 日 2 時間 = 約 $4/日、月約 $120 | **確認(計算)** | $0.034 × 120 = $4.08/日、× 30 = $122.40 |
| 22 | 3-2 構成(端末内 STT + テキストを LLM)は、ほぼ LLM の文字代だけ | **一部確認** | LLM 代は月 $1〜$3 程度。ただしクラウド TTS を使うと月 $15〜$65 が加わる(TTS が費用の大半)。r.txt はこの点が抜けている |
| 23 | Apple は端末内 LLM(Foundation Models)を翻訳に勧めない | **確認(フォーラム)** | Apple DTS Engineer による Group Lab のまとめ https://developer.apple.com/forums/thread/792763 |
| 24 | 画面ロック後もマイクを使うには、画面を開いた状態で録音を始めておく | **二次情報のみ** | Apple Developer Forums の報告。公式ドキュメントでの明記は見つからない |

---

## 9. 未確認事項

1. **遅延の数値**: OpenAI / Gemini の翻訳モデル、STT 各社、TTS(ElevenLabs 以外)とも、公式の数値が無い。P1・P2 で実測する。
2. **`gpt-realtime-translate` の課金基準**: 送った音声の長さか、接続していた時間か。無音を止めたときの品質への影響。セッションの上限。
3. **Gemini Live Translate**: セッション・接続の上限が通常の Live と同じか。日本語が入出力のどちらにも対応しているかの明記。
4. **遠い声・重なった声での日本語 STT の精度**: SpeechAnalyzer、Deepgram、AssemblyAI のいずれも、周囲の雑談の条件での公式データは無い。
5. **iOS の連続稼働**: ネイティブアプリで画面ロック中に録音と再生を 2 時間続けられるか(Apple の公式な明記は無い)。Safari の iOS 26 での挙動。
6. **`playAndRecord` + `allowBluetoothA2DP`(HFP なし)で、入力が確実に内蔵マイクになるか**(実機で確認する)。モードによる声の処理が遠い声に与える影響。
7. **高音質録音の対応機種**: AirPods Pro 3 / AirPods 5 など。
8. **価格が取得できなかったもの**: Google Cloud STT(ページの本文を取得できず)、Azure Speech(公式ページで数値が非表示)、DeepL API(公式の価格表を取得できず。`custom_instructions` の提供プランも未確認)、ElevenLabs の各プランの超過単価。
9. **Deepgram のストリーミングで無音が課金されるか**(FAQ では「しない」としつつ、「処理した音声の秒数」で課金とも書いてある)。
10. **OpenAI Realtime の音声トークン換算(1 トークン/100ms・50ms)**: 公式ページの検索抜粋でのみ確認。ページ本体は 404 だった。
11. **Gemini Live の文脈蓄積分の課金**、Gemini TTS の遅延と話し方の指示。
12. **費用試算の仮定**: 発話率 50%、10 秒ごとの LLM 呼び出し、1 回 150 文字。実測で置き換える。
13. **ユーザーの判断が要る点**: 学習言語(OpenAI 翻訳の 13 言語に入るか)。周囲の人の音声をクラウドへ送ることを許容するか(端末内 STT を必須にするか)。月額の上限。
