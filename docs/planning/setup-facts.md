# 開発環境の準備: ユーザーが手元で導入・確認するものの事実関係

- 調査日: 2026-10-30(出典の確認日もすべて 2026-10-30)
- 調査: サブエージェント(Opus)。前提: 実機 iPhone SE(第 3 世代)+ AirPods 4(ANC、A3055〜A3057)、開発機は Mac、配布なし、口語化は Claude Haiku 5.5、TTS は Apple 内蔵(AVSpeechSynthesizer)。
- 凡例: **[一次]** Apple / Anthropic / OpenAI / Google の公式ページ(App Store の掲載ページを含む)で確認 / **[二次]** ブログ・利用者投稿・第三者データのみ / **[推測]** 筆者の推論 / **[未確認]** 調べたが確認できない / **[実測]** この Mac でコマンドを実行して確認した値
- ファイルのダウンロード、ログイン、API の呼び出しはしていない。Apple の公開ページの HTML は表を読むために取得したが、作業用ディレクトリに置いただけでリポジトリには入れていない。

## 1. 結論(項目ごと)

1. **Xcode**: iOS 26 SDK は Xcode 26.x に入っている。最新は Xcode 26.6(iOS 26.5 SDK)で、macOS Tahoe 26.2 以降が必要。iOS 27 SDK は Xcode 27.x(最新 27.2)に入っていて、**macOS Tahoe 26.6 以降**が必要 [一次]。この Mac は macOS 26.5.1 + Xcode 26.6 なので、**iOS 26 SDK ならこのままビルドできる。Xcode 27 を使うなら macOS の更新が先に要る** [実測]。
2. **無料の Apple Account での実機インストール**: 署名は Xcode の Personal Team で行う。プロファイルの有効期限は 7 日、App ID は 10 個まで(7 日で失効)、登録できる端末は 3 台まで、**1 台に入れられるアプリは 3 個まで** [一次]。本プロジェクトで要る **Background Modes(audio)は無料アカウントでも使える**。マイクと音声認識は capability ではなく Info.plist の利用目的の記載で足りる [一次]。無料では使えないもののうち関係しそうなのは **App Attest、Push Notifications、iCloud** [一次]。
3. **premium / enhanced 音声**: iOS 26・27 では「**設定 > アクセシビリティ > リーダーと読み上げ > 声**」で声と方言を選んでダウンロードする(04 の「読み上げコンテンツ」は旧名)[一次]。enhanced は 1 つ 100MB 以上になることがあり、Wi-Fi が要る [一次]。言語ごとに premium があるかを示す Apple の公開一覧は無い。第三者のデータでは en-US / en-GB / fr-FR / zh-CN / zh-TW / yue-HK に最上位品質の声がある [二次]。
4. **SpeechTranscriber の ja-JP アセット**: **アプリが `AssetInventory` でダウンロードする**(`assetInstallationRequest(supporting:)` → `downloadAndInstall()`)。ユーザーが設定画面で入れるものではない。アセットはシステムが管理し、他のアプリと共有される [一次]。容量は Apple が公表していない [未確認]。
5. **Anthropic API**: Console でアカウントを作る → Billing でクレジットを買う(前払い、1 年で失効、返金なし)→ API キーを作る。**自分で月の上限(spend limit)を設定できる** [一次]。最小購入額は公式ページに書かれていない(第三者は $5 と書いている)[二次]。単価は Haiku 5.5 が入力 $0.10 / 出力 $0.50、Sonnet 5.5 が $2 / $10(1M トークンあたり)[一次]。アプリに API キーを入れると取り出せる、と公式が警告している。公式の代替手段の App Attest は有料の Developer Program が前提になる。したがって無料アカウントの個人利用では「**キーを埋め込み、上限と有効期限で被害を抑える**」のが現実的 [一次 + 推測]。
6. **AirPods 4(ANC)**: ファームウェアは「設定 > Bluetooth > AirPods の (i) > 情報」で確認する。更新は自動で、条件は充電中・iPhone の近く・iPhone が Wi-Fi 接続。現在の最新は **9A348** [一次]。高音質録音(8A356)の対象機種に「AirPods 4 with Active Noise Cancellation」が含まれる [一次]。
7. **既存サービスの試験**: Google 翻訳と ChatGPT はどちらも **iOS 18.0 以降**なので SE 第 3 世代で使える [一次]。**ChatGPT Voice(Live)は Free プランでも使える**が、モデルは GPT-Live-1.1 Quip で、上限は「Limited access」としか書かれていない(数値なし、変更あり)[一次]。

## 2. 確認項目ごとの結果

### 2.1 Xcode と macOS

| Xcode | 必要な macOS | iOS SDK | 実機に入れてデバッグできる iOS(Device Support) | 根拠 |
|---|---|---|---|---|
| 27.2 / 27.1 / 27 | macOS Tahoe 26.6 以降 | iOS 27.2 / 27.1 / 27 | iOS 17 以降 | [一次] |
| 26.6 / 26.5 | macOS Tahoe 26.2 〜 26.x | iOS 26.5 | iOS 15 以降 | [一次] |
| 26.4.1 | macOS Tahoe 26.2 〜 26.x | iOS 26.4 | iOS 15 以降 | [一次] |
| 26.3 / 26.2 | macOS Sequoia 15.6 〜 Tahoe 26.x | iOS 26.2 | iOS 15 以降 | [一次] |
| 26 / 26.0.1 / 26.1 / 26.1.1 | macOS Sequoia 15.6 〜 Tahoe 26.x | iOS 26 / 26.1 | iOS 15 以降 | [一次] |

- 出典: https://developer.apple.com/xcode/system-requirements/ / https://developer.apple.com/support/xcode/ (2026-10-30)
- この Mac の状態 [実測、2026-10-30]: macOS 26.5.1(25F80)、Xcode 26.6(17F113)、iOS SDK 26.5。
  - **iOS 26 SDK で作るなら、今の Mac と Xcode のままでよい。**
  - Xcode 27 を使うには、先に macOS を 26.6 以降に上げる必要がある。
- 対象 API の OS 要件: `SpeechTranscriber` / `AssetInventory` / `DictationTranscriber` は iOS 26 から [一次、04 §2.3]。iOS 26 SDK で足りる。
- **iPhone が iOS 27 に上がっている場合**: Xcode 26.6 の Device Support の表記は「iOS 15 or later」だが、それより新しい iOS 27 の実機に Xcode 26.6 からインストール・デバッグできるかは公式表からは読み切れない [未確認]。新しい iOS の実機には、対応する Xcode が必要になることが多い [推測]。iPhone の iOS のバージョン(設定 > 一般 > 情報)を先に確かめる。
- iOS 27 SDK が必要になる場面: Anthropic の Swift パッケージ「Claude for Foundation Models」は iOS 27 と Xcode 27 が必要で、まだベータ [一次: §2.5]。本構成は Messages API を `URLSession` で直接呼ぶ想定なので、このパッケージは要らない [推測]。

### 2.2 無料の Apple Account(Personal Team)で自分の iPhone に入れる条件

**制限** [一次] https://developer.apple.com/support/compare-memberships/ (2026-10-30)
- Developer Program に入っていないアカウントは、Xcode で「Personal Team」として扱われる。App ID・端末・証明書・プロファイルは Xcode が直接管理し、**定期的に再インストールが必要**。
- App ID は 10 個まで登録でき、7 日で失効する。
- 端末は 3 台まで登録でき、7 日で失効する。
- 実機にインストールするためのプロファイルは発行から 7 日で失効する。
- **1 台の端末に入れられるアプリは 3 個まで。**
- 「高度な capability の開発とアプリの配布には Developer Program か Enterprise Program が必要」とも書かれている。
- → 本プロジェクトへの影響 [推測]: アプリは 7 日ごとに Mac につないで再ビルド・再インストールすることになる。期限が切れるとアプリは起動しなくなる。連続稼働の試験(2 時間)はこの周期の中に収まる。

**Developer Mode(iOS 16 以降)** [一次] https://developer.apple.com/documentation/xcode/enabling-developer-mode-on-a-device (2026-10-30)
1. iPhone を Mac とペアリングする(ケーブルまたは無線。Xcode の Device Hub)。**ペアリングを始めるか、過去にペアリングしたことがないと、設定に Developer Mode の項目は出ない。**
2. iPhone の「設定 > プライバシーとセキュリティ」で、「セキュリティ」の下にある Developer Mode をオンにする。警告が出るので「再起動」を押す。
3. 再起動後の確認画面で「有効にする」を押し、パスコードを入れる。

**開発者の信頼(初回起動時)**
- Apple のヘルプにあるのは Enterprise アプリの手順だけ。内容は「設定 > 一般 > VPN とデバイス管理」で開発者を選んで信頼する、というもの。インターネット接続が必要 [一次] https://help.apple.com/xcode/mac/current/en.lproj/dev96a12fb84.html / https://support.apple.com/118254 (2026-10-30)
- 無料アカウントで署名したアプリでも「VPN とデバイス管理 > デベロッパApp」で信頼する、という報告がある [二次] https://developer.apple.com/forums/thread/743125 / https://developer.apple.com/forums/thread/685271 (2026-10-30)

**capability ごとの可否**(Apple の表の「Apple Developer」列。無料アカウントのこと)[一次] https://developer.apple.com/help/account/reference/supported-capabilities-ios (2026-10-30。表のセルを HTML から読み取った)

| 本プロジェクトとの関係 | capability | 無料アカウント |
|---|---|---|
| **必要**(ロック中の録音・再生) | Background modes(`UIBackgroundModes` の `audio`) | **使える** |
| 使う可能性あり | App groups / Data protection / Keychain sharing | 使える |
| 使う可能性あり(Claude の App Attest 認証) | App Attest | **使えない** |
| 使う可能性あり(Live Activity をサーバーから更新する場合) | Push notifications | **使えない** |
| 設定の同期に使うなら | iCloud(CloudKit / documents / key-value) | **使えない** |
| 関係しない | Siri、Sign in with Apple、In-App Purchase、Network extensions、Associated domains、Increased Debugging Memory Limit など | 使えない |

- **マイク・音声認識は capability ではない** [一次]
  - マイク: Info.plist の `NSMicrophoneUsageDescription` が必須。https://developer.apple.com/documentation/bundleresources/information-property-list/nsmicrophoneusagedescription (2026-10-30)
  - 音声認識の許可(`NSSpeechRecognitionUsageDescription` + `requestAuthorization`)が要るのは `SFSpeechRecognizer` の場合。Apple の記事には「この手順は `SFSpeechRecognizer` だけに適用される」「SpeechAnalyzer の文字起こしモジュールは音声を Apple のサーバーに送らない」とある。https://developer.apple.com/documentation/speech/asking-permission-to-use-speech-recognition (2026-10-30)
  - → `SpeechTranscriber` だけなら、マイクの許可だけで足りると読める [一次の記述からの推測]。予備の `SFSpeechRecognizer` を使う場合は両方のキーを入れておく。
- **有料(年 99 ドル)にしないと使えないもの**のうち、本プロジェクトに関係しうるのは次の 4 つ [一次 + 推測]。
  1. App Attest(Claude API をキーなしで呼ぶ公式の方法。§2.5)
  2. Push 通知
  3. iCloud
  4. 7 日を超えるプロファイル(有料なら 1 年)
  - 最小構成(内蔵マイク → STT → Claude → TTS → AirPods)は無料アカウントの範囲で組める。

### 2.3 AVSpeechSynthesizer の enhanced / premium 音声

- **ダウンロードの場所(iOS 26・iOS 27 とも)** [一次]: 設定アプリ >「アクセシビリティ」>「**リーダーと読み上げ**」>「**声**」(声と方言を選ぶ)。英語版では Accessibility > Read & Speak > Voices。
  - https://support.apple.com/ja-jp/guide/iphone/iph96b214f0/26/ios/26 (2026-10-30)
  - https://support.apple.com/ja-jp/guide/iphone/iph96b214f0/ios (iOS 27 版、2026-10-30)
  - 04 §2.5 の「読み上げコンテンツ > 声」は旧称。
- VoiceOver 側のパス(設定 > アクセシビリティ > VoiceOver > 読み上げ > 言語を追加 > 声 > 高音質)でも、enhanced をダウンロードできる [一次] https://support.apple.com/en-us/111798 (2026-10-30)
- 容量と通信: enhanced 音声は 100MB 以上になることがあり、Wi-Fi 接続時にダウンロードが始まる [一次: 111798]。premium の容量の公式記載は無い [未確認]。
- 品質の段階は default / enhanced / premium の 3 つ。enhanced と premium はダウンロードが必要 [一次、04 §2.5]。
- **言語ごとの premium の有無**: Apple の公開一覧は無い [一次の範囲で未確認]。オープンソースの Readium プロジェクトの音声データ(iOS で最上位品質「high」を持つ声)を、参考として挙げる [二次] https://readium.org/speech/docs/VoicesAndFiltering.html / https://readium.org/speech/json/en.json / fr.json / cmn.json / yue.json (2026-10-30。データの更新日は記載なし)。

| ロケール | 最上位品質のある声(Readium) | 備考 |
|---|---|---|
| en-US | Ava、Zoe(default / enhanced / premium の 3 段)、Alex | Ava には「Ava (Premium)」という別名がある |
| en-GB | Serena、Jamie(3 段) | |
| fr-FR | Audrey(3 段) | fr-CA は Amélie |
| zh-CN(普通話) | Lilian、Lili、Han、Yue | |
| zh-TW(台湾の普通話) | Meijia(low / high の 2 段) | |
| zh-HK | Sinji(**広東語** yue-HK) | 香港の普通話の声はデータに無い |

  - Readium の「high」が Apple の premium に当たるかは [推測](Ava の 3 段と別名から推した)。
  - 確定は実機で行う。ダウンロード後に `AVSpeechSynthesisVoice.speechVoices()` を `quality == .premium` で絞る。未ダウンロードの声を列挙する API は無い [二次、04 §2.5]。
- 補足 [二次、04 §2.5]: iOS 26 では、設定で選んだ声がアプリに反映されない不具合の報告がある。アプリ側で声の識別子を指定する。

### 2.4 SpeechTranscriber(iOS 26)の ja-JP アセット

- **アプリがダウンロードする** [一次] https://developer.apple.com/documentation/speech/assetinventory (2026-10-30)
  - 手順は 4 段。
    1. モジュール(`SpeechTranscriber` など)を作る。
    2. ロケールを予約する。`reserve(locale:)`、または自動で予約される。
    3. `assetInstallationRequest(supporting:)` で要求を取得し、`downloadAndInstall()` を呼ぶ。
    4. 完了を待つ。
  - アセットは Apple のサーバーから落とす機械学習モデルで、システムが管理・更新する。他のアプリと共有され、アプリを再起動しても残る。プリインストール済み、または他のアプリが落としていれば、すぐに完了する。
  - 使われないアセットは、システムが購読を外すことがある。
  - アプリが予約できるロケール数には上限(`maximumReservedLocales`)がある。値は端末の空き容量で変わる。超えるとエラーになる [一次] https://developer.apple.com/documentation/speech/assetinventory/maximumreservedlocales / https://developer.apple.com/documentation/speech/assetinventory/assetinstallationrequest(supporting:) (2026-10-30)
  - アプリのバンドルサイズは増えない [二次: WWDC25 277 の書き起こし https://nonstrict.eu/wwdcindex/wwdc2025/277/ 、https://www.argmaxinc.com/blog/apple-and-argmax (2026-10-30)]
- **ユーザー側の操作**: 設定画面で入れる手順は、Apple の文書に無い [一次の範囲]。アプリが初回にダウンロードを始めるときに、通信と空き容量があればよい [推測]。
- **容量**: Apple は公表していない [未確認]。検索結果の要約に「133MB」とあったが、出典とされたページ(Argmax)で該当の数値を確認できなかった [未確認]。実機でダウンロードの前後の空き容量を比べて測る。
- 機種の可否(`isAvailable`)と ja-JP の対応(`supportedLocales`)は、04 §2.3 のとおり実機で確認する。

### 2.5 Anthropic API の利用開始

**手順** [一次]
1. Claude Console(https://platform.claude.com/)でアカウントを作る。https://platform.claude.com/docs/en/get-started (2026-10-30)
2. クレジットを買う: Console の Billing ページ(`platform.claude.com/settings/billing`)で「Buy credits」を押し、金額を入れる。すぐ使える。
   - 支払いはクレジットカード。
   - クレジットは購入から 1 年で失効し、延長できない。**返金されない。**
   - 自動チャージ(auto-reload)もある。
   - 残高が尽きると API は止まる。
   - 出典: https://support.claude.com/en/articles/8977456 / https://platform.claude.com/docs/en/about-claude/pricing (2026-10-30)
3. 新規ユーザーには試用の少額クレジットが付く [一次: pricing の FAQ]。金額は書かれていない [未確認]。
4. **最小購入額は公式ページに書かれていない** [未確認]。第三者の解説は $5 としている [二次] https://dev.classmethod.jp/en/articles/claude-console-buy-credit-create-api-key/ ほか (2026-10-30)。購入画面で確かめる。
5. API キーを作る: Console の API keys ページ(`platform.claude.com/settings/keys`)で「Create key」を押す [一次] https://platform.claude.com/docs/en/get-api-key / https://platform.claude.com/docs/en/manage-claude/authentication (2026-10-30)
   - 名前、有効期限、Linked account(自分なら personal key)を決める。ワークスペースに限定もできる。
   - キー(`sk-ant-` で始まる)は作成時に一度だけ表示される。
   - 有効期限はプリセット(3 時間 / 1 日 / 7 日 / 30 日)、任意の期間、Never から選ぶ。作成後は変えられない。
   - 送り方: `Authorization: Bearer <key>`(従来の `x-api-key` も可)と `anthropic-version: 2023-06-01`。

**月の上限(spend limit)** [一次] https://platform.claude.com/docs/en/api/rate-limits (2026-10-30)
- 組織の上限は Billing ページの「Spend limits」で設定できる(「Adjust limit」または「Set limit」)。ティアの上限を超える値にはできない。
  - ティアの上限: Start は月 $500、Build は $1,000、Scale は $200,000。
  - 新しい組織は、これより低い Evaluation ティアから始まることがある。
- 自分で決めた上限に達すると、API は HTTP 400(`invalid_request_error`、メッセージは「You have reached your specified API usage limits」)を返す。上限を上げるか外すまで止まる。
- ワークスペースごとにも上限を置ける。**ただし Default ワークスペースには置けない。** → 本アプリ専用のワークスペースを作り、そこに上限($50 など)を置き、キーをそのワークスペースに限定する形が取れる [一次 + 推測]。

**単価(標準、1M トークンあたり)** [一次] https://platform.claude.com/docs/en/about-claude/pricing / https://platform.claude.com/docs/en/models/overview (2026-10-30)

| モデル(API ID) | 入力 | 5 分キャッシュ書き込み | キャッシュヒット | 出力 | 備考 |
|---|---|---|---|---|---|
| Claude Haiku 5.5(`claude-haiku-5-5`)、プロンプト 10 万トークン以下 | $0.10 | $0.125 | $0.01 | $0.50 | 10 万超は $0.50 / $2.50。遅延は「Fastest」 |
| Claude Sonnet 5.5(`claude-sonnet-5-5`) | $2 | $2.50 | $0.10 | $10 | 遅延は「Fast」 |

- 補足 [一次]: 両モデルとも thinking は Adaptive で、既定の effort は Haiku 5.5 が `medium`、Sonnet 5.5 が `high`。遅延を詰めるなら effort の指定を検討する [推測]。
- 公式の client SDK は Python / TypeScript / C# / Go / Java / PHP / Ruby と CLI で、**Swift の汎用 SDK は無い** [一次: authentication ページの SDK 一覧]。iOS からは `URLSession` で Messages API を直接呼ぶことになる [推測]。

**iOS アプリにキーを埋め込むことについて**
- 公式の警告 [一次] https://platform.claude.com/docs/en/cli-sdks-libraries/libraries/apple-foundation-models (2026-10-30)
  - アプリに入れたキーは配布したバイナリから取り出せ、取り出した人はこちらの課金でリクエストできる。
  - 公式は `.apiKey` を開発用に限り、リリース前に App Attest かプロキシへ移すよう勧めている。
- 公式の代替手段 [一次] https://platform.claude.com/docs/en/manage-claude/app-attest (2026-10-30)
  - **App Attest**: 端末が Apple の App Attest で正規のアプリであることを証明すると、1 時間有効のトークンが発行される。キーをアプリに入れずに済む。
  - 必要なもの: Apple Developer の Team ID、Xcode の **App Attest capability**、Console での「App integration」登録(admin 以上の権限)。実機のみで、シミュレータは不可。
  - 使い方は Swift パッケージ「Claude for Foundation Models」経由(ベータ。iOS 27・Xcode 27 が必要)。
  - **App Attest capability は無料アカウントでは使えない**(§2.2 の表)。
  - Foundation Models フレームワーク経由のこのパッケージが、Apple Intelligence 非対応の SE 第 3 世代で動くかは [未確認]。
- CORS について
  - ブラウザから直接呼ぶ場合は `anthropic-dangerous-direct-browser-access` ヘッダーが要る、という解説がある [二次] https://simonwillison.net/2024/Aug/23/anthropic-dangerous-direct-browser-access (2026-10-30)
  - CORS はブラウザの仕組みなので、ネイティブ iOS アプリの `URLSession` には関係しない [推測]。
- **個人利用・サーバーなしの現実的な扱い** [推測。公式の推奨ではない]
  - 配布しないので、キーが漏れる経路は主に次の 2 つになる。
    - 端末やビルド成果物から取り出される。
    - キーを Git にコミットしてしまう。
  - 被害を抑える策:
    1. 本アプリ専用のワークスペースを作り、月の上限($50 以下)を置く。キーもそのワークスペースに限定する。
    2. キーに有効期限(30 日など)を付ける。無料署名は 7 日ごとに入れ直すので、そのときに差し替えてもよい。
    3. キーはソースに書かない。ビルド設定(Git 管理外の xcconfig)から入れ、端末では Keychain に置く。またはアプリ内の設定画面で手入力する。
    4. 漏れた疑いがあれば Console で無効化する。
  - 補足: Anthropic は GitHub の secret scanning と連携している [二次: 検索結果の要約のみ、本文は未読] https://support.claude.com/en/articles/9767949 (2026-10-30)

### 2.6 AirPods 4(ANC)のファームウェアと高音質録音

- **確認方法** [一次] https://support.apple.com/en-us/106340 (2026-10-30): 設定 > Bluetooth > AirPods の名前の横の (i) >(下にスクロール)情報 で、ファームウェアのバージョンが見える。
- **更新** [一次: 同上]
  - 充電中で、Wi-Fi につながった iPhone の Bluetooth 圏内にあれば、自動で届く。
  - 手動で促す手順:
    1. iPhone を最新の iOS にする。
    2. AirPods を接続し、iPhone を Wi-Fi につなぐ。
    3. ケースを電源につなぎ、AirPods を入れて蓋を閉じる。
    4. 圏内で 30 分以上待ち、蓋を開けて再接続する。
    5. 版を確認する。上がらなければリセットして再試行する。
- **最新版**: AirPods 4 と AirPods 4(ANC)はともに **9A348**(Custom EQ の追加など)[一次: 同上]。リリース日は記載が無い。
- **高音質録音の対象**: 8A356 のリリースノートに、「AirPods 4、AirPods 4 with Active Noise Cancellation、AirPods Pro 2、AirPods Pro 3」で「iOS 26 に対応する iPhone」と使えるとある [一次: 同上]。04 §2.6 と同じ。
  - API(`bluetoothHighQualityRecording`)側の対象機種は「certain AirPods models」としか書かれていない。実行時に `isSupported` で確認する [一次、04 §2.6]。

### 2.7 既存サービスの実機試験(01 第 5 章 試験 B・C)に要るもの

| サービス | iOS 要件 | 価格 | 根拠 |
|---|---|---|---|
| Google 翻訳 | **iOS 18.0 以降** | 無料 | [一次] https://apps.apple.com/us/app/google-translate/id414706506 (2026-10-30。版 10.44.368、255.5MB) |
| ChatGPT | **iOS 18.0 以降** | 無料(アプリ内課金あり。米国ストアの表示で ChatGPT Go $8.00、Plus $19.99 など) | [一次] https://apps.apple.com/us/app/chatgpt/id6448311069 (2026-10-30。版 1.2026.293、293.2MB) |

- ChatGPT Voice(Live)の利用上限(24 時間の移動枠)[一次] https://help.openai.com/en/articles/20001274-chatgpt-voice (2026-10-30、内蔵ブラウザで閲覧。WebFetch は 403)
  - Free: 「Limited access to GPT-Live-1.1 Quip. Limits may change.」。**数値の記載は無い。**
  - Go: GPT-Live-1.1 Quip で 3 時間。
  - Plus: GPT-Live-1.1 で 3 時間。
  - Pro($100): 15 時間。Pro($200): 無制限(乱用防止の制限付き)。
  - 上限に達すると通知される。
- その他の要点 [一次: 同上]
  - Live / Advanced / Standard は「設定 > Voice」で切り替える。
  - バックグラウンド・画面ロック中も会話を続けられる(制御は「設定 > Voice」)。終わる条件は、利用上限・最大セッション長(数値なし)・強制終了・長い会話がコンテキストの上限に達したとき。
  - 「主に 1 対 1 の会話向けで、複数話者には最適化されていない。人同士の会話に応答してしまうことがある」。
  - Live と Advanced の音声クリップは 30 日保存される。
  - Free でも試験 C は実施できる。ただし 10 分の試験の途中で上限に達する可能性は否定できない [推測]。
- ChatGPT のアカウントが要るかどうか(未ログインで Voice を使えるか)は確認していない [未確認]。

## 3. 未確認事項

1. Xcode 26.6 から iOS 27 の実機にインストール・デバッグできるか。iPhone SE の現在の iOS のバージョン(ユーザーの手元で確認)。
2. Anthropic のクレジットの最小購入額と、新規ユーザーの試用クレジットの額(購入画面で確認)。新しい組織が Evaluation ティアで始まった場合の制限値。
3. Claude for Foundation Models(App Attest)が、Apple Intelligence 非対応の SE 第 3 世代で動くか。本構成では使わない想定なので、優先度は低い。
4. en-US / en-GB / fr-FR / zh-CN / zh-TW の premium 音声の有無(公式の一覧が無い。実機で `speechVoices()` を確認)。zh-HK は広東語の声しか見つからなかった。香港の普通話が要るかは学習目標しだい。
5. SpeechTranscriber の ja-JP アセットの容量。`maximumReservedLocales` の実際の値。
6. 無料署名のアプリの「信頼」の手順(Apple の公式記事は Enterprise 向けしか見つからなかった)。
7. ChatGPT Voice の Free の具体的な上限時間と最大セッション長。未ログインで Voice を使えるか。
8. `SpeechTranscriber` だけを使うとき、`NSSpeechRecognitionUsageDescription` を入れなくても審査外の実機で問題が出ないか(Apple の記述からは不要と読めるが、実機で確認)。
