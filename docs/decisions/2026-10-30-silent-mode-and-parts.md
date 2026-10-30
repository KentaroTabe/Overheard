# 2026-10-30 無音時の出力・実機・部品の判断(ユーザー指示)

`2026-10-30-initial-constraints.md` で「確認中」だった項目の確定。

| 項目 | 判断 | 影響 |
|---|---|---|
| 無音時の出力 | 周囲の発話が無い間は、Claude が生成した学習言語の雑談を TTS で流す。**短くする必要はなく、長く続ける** | 生成は「短い一言」ではなく、会話として続く長さを前提にする。トークン消費は翻訳より大きくなるので、月 $50 枠の見積りに加える。発話検知時の割り込み(生成雑談の中断→翻訳)の設計が要る |
| AirPods | AirPods 4(ANC 付き)、型番 A3055〜A3057 | iOS 26 の高音質録音(`bluetoothHighQualityRecording`)の対象機種。主入力は iPhone マイクで変わらず、AirPods マイクは比較試験のみ |
| TTS | Apple 内蔵 AVSpeechSynthesizer から始める(無料)。en / zh / fr の enhanced / premium 音声をダウンロードして試聴 | 品質が足りなければクラウド TTS を再検討(その際の費用は別途判断) |
| 口語化モデル | Claude Haiku 5.5 を既定。聞き比べで Sonnet 5.5 に上げる | アプリ側に呼び出し回数の上限を置く(月 $50 の超過防止) |
| ローカルフォルダ | `/Users/kenta/GitHub/Overheard` に改名(ユーザー実施) | 以前のレポートに書かれた `/Users/kenta/GitHub/Zawamimi` のパスは読み替える |
