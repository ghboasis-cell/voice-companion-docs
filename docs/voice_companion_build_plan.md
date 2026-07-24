# VoiceCompanion 作業手順書 兼 運用ルール

**版数: v5.56 ／ 最終更新日: 2026-07-24**

（v5.56: Android正式通話Pluginへの必須移行は工程1〜7で完了した。正式`AndroidVoiceCall`経路はAAB 1064で発信・疑似着信・音・バイブ・AI音量・診断・録音復帰・旧Plugin削除後のPlugin取得を実機確認済みであり、PR #69をAndroid正式通話実装の完了PRとしてmainへマージ可能な状態とする。工程8「初回音声・文間遅延の計測と改善」はAndroid正式Plugin移行の完了条件には含めず、未着手の`[ ]`を維持する後続の性能改善と位置付ける。工程8の実装はPR #69へ含めない。PR #69のマージ後は、別branch・別PRでiOS正式疑似通話・オンデバイスTTS接続へ進む。spec v4.9、実装コード、main、production、DB、migration、RPC、R2、env、secret、Edge Function、iOSは本docs更新では変更しない。）

（v5.55: 正式Android通話実装の工程7「旧AndroidTtfaTestPluginの削除と残存参照整理」を完了した。`AndroidTtfaTestPlugin.java`、`TurnUsageBarrier.java`、`TurnUsageBarrierTest.java`、`scripts/test-android-turn-usage-barrier.mjs`を削除し、`MainActivity.java`から旧Pluginのimportと登録、`src/appMain.ts`から旧Plugin選択・staging分岐・fallback、`src/main.js`から旧TTFA bridge・専用UI・method・listener、`package.json`から旧純Javaテストランナーを削除した。旧Pluginの存在・hash・新旧同時登録を前提とするテストは正式Controller契約へ更新し、`tests/androidLegacyVoiceCallRemoval.test.mjs`を追加した。実行コード内の旧Plugin参照は0件とし、docs・履歴内の旧Plugin名は過去記録として維持する。正式`AndroidVoiceCall`、`VoiceCallRuntime`、`VoiceCallSession`、正式Controller群、オンデバイスTTS、コール音・着信音・バイブ、audio focus・route、usage・終了処理、incoming standby、診断表示、iOS経路、共通resourceとdependencyは維持した。

実装commitは`294167e410ae3a2677212db7d65c70aa2a4929b1`（`refactor: remove legacy Android TTFA plugin`）。GitHub Actions `Android On-Device TTS Test` run `30026699779`はSUCCESSし、`:app:compileDebugJavaWithJavac`、`:app:compileDebugUnitTestJavaWithJavac`、対象JUnit 170件、CIオンデバイスTTS統合Node 26/26がPASSした。ローカルは工程1〜7関連Node 169/169、TypeScript compile、Web build、`git diff --check`、純Java controllerのjavac確認がPASSし、ローカルGradleは実行していない。指定外の全件`npm test`では工程7と無関係な`stagingTtsEnv.test.mjs`の子プロセスstdout検査2件が失敗したが、工程7関連テストは個別PASSしており、この2件は変更していない。同テストはAndroid CI対象外で、Android CIはSUCCESSした。

Android Staging AAB run `30027316701`はSUCCESS。versionCode `1064`、versionName `staging.1.0.64`、artifact `android-staging-aab`、artifact size `80,739,412 bytes`、対象commit `294167e410ae3a2677212db7d65c70aa2a4929b1`。AAB 1064のAndroid実機で、ユーザー発信、疑似着信応答、コール音、着信音、バイブ、音・バイブの適切な停止、通常AI音声の音量、デバッグ表示、録音準備音、録音復帰を確認し、Plugin取得エラーと旧Plugin fallbackがなく、ユーザー報告「問題ない」となった。工程7は完了したため移行順序7を`[x]`とする。工程8は未着手の`[ ]`を維持し、次工程は8「初回音声・文間遅延の計測と改善」。spec v4.9、main、production、DB、migration、RPC、R2、env、secret、Edge Function、iOSは変更していない。PR #69はOpen・Draft・未マージを維持する。）

（v5.54: 正式Android通話実装の工程6「staging切替とAndroid実機確認」を完了した。staging Androidの実通話経路を旧`AndroidTtfaTestPlugin`から正式`AndroidVoiceCall`へ切り替え、`VoiceCallSession`、`VoiceConnectionController`、`VoiceRecordingController`、`VoiceTtsController`、`VoiceUsageController`を通じて、接続、録音、オンデバイスTTS、usage、終了処理を実通話へ接続した。旧`AndroidTtfaTestPlugin`は工程7で削除するため、比較可能な状態で登録・ファイルを残している。

工程6初回のAAB 1061では実機確認に失敗した。正式Controllerの`HttpUrl.get(wssEndpoint)`がWebから渡る`wss://` endpointを処理できず、WebSocket callbackへ到達する前に同期失敗した。また、正式側の診断fieldと既存Web表示契約が一致せず、通話開始前に発生した原因表示も欠落した。WebSocket request生成をOkHttpの`Request.Builder().url(endpoint)`へ変更し、許可schemeを安全に検証しつつ、URL全文・access token・Authorization headerを診断へ出さない契約を維持した。正式Plugin選択、start受信、引数有無、socket開始・失敗・ready・failure、fatalを既存fieldへ正規化し、開始前の安全な診断だけを既存画面で表示可能にした。通信・診断修正commitは`75711ed5d2de923afbb5ae8f97406fa2c4e48fd5`、構造テスト更新を含むcommitは`73ea1402167e68b686534a3f41ef280deceb0f4f`。GitHub Actions `Android On-Device TTS Test` run `30017409317`はSUCCESS。Android Staging AAB run `30017674236`はSUCCESS、versionCode `1062`、versionName `staging.1.0.62`。

AAB 1062の実機では、ユーザー発信・疑似着信応答の通話接続、AI音声、正式Pluginのデバッグ表示、録音準備音、録音復帰は正常だった。一方、ユーザー発信コール音、疑似着信音、疑似着信バイブがなく、通常AI音声が小さい不具合を確認した。修正commit `16e380cffc4cdba67cd2fc5a00dc722a94e5617b`で`VoiceCallSignalController`と`VoiceAudioRouteController`を追加し、発信コール音を着信音処理から分離した。API 31以降は`VibratorManager`を使用し、通常・バイブ・サイレントのringer mode別に着信音とバイブを制御する。正式通話経路へaudio focusと`MODE_IN_COMMUNICATION`を復元し、API 31以降は`setCommunicationDevice`、旧APIは`setSpeakerphoneOn`で受話口優先・利用不可時スピーカーfallbackとした。PCM gain、TTSモデル、AudioTrack volumeは変更していない。GitHub Actions `Android On-Device TTS Test` run `30022090161`はSUCCESSし、`:app:compileDebugJavaWithJavac`と`:app:compileDebugUnitTestJavaWithJavac`の2件、対象JUnit 170件、オンデバイスTTS統合Node 26/26がPASSした。ローカル関連Nodeは130件PASS。

Android Staging AAB run `30022527021`はSUCCESS。versionCode `1063`、versionName `staging.1.0.63`、artifact `android-staging-aab`、artifact size `80,768,910 bytes`、対象commit `16e380cffc4cdba67cd2fc5a00dc722a94e5617b`。AAB 1063のAndroid実機で、ユーザー発信、疑似着信応答、コール音、着信音、バイブ、応答・終了時の音とバイブ停止、通常AI音声の音量、デバッグ表示、録音準備音、録音復帰を確認し、発信・着信とも会話成立、ユーザー報告「問題なし」となった。工程6は完了したため移行順序6を`[x]`とする。次工程は7「旧AndroidTtfaTestPluginの削除と残存参照整理」。spec v4.9、main、production、DB、migration、RPC、R2、env、secret、Edge Function、iOSは変更していない。PR #69はOpen・Draft・未マージを維持する。）

（v5.53: 正式Android通話実装の工程5「課金usage・終了処理の正式構成への移行」を完了した。正式packageへ`VoiceUsageController`を追加し、ユーザーturnの録音開始から終了までの実経過時間（録音中の沈黙を含む）と、通常AI回答の実再生完了時に`VoiceTtsController`から渡される実再生時間だけをturn単位で累積する。固定聞き返し、録音準備音、無音padding、TTS合成待ち、通信待ち、再生されなかった音声はusage対象外とした。`VoiceCallSession`へ狭いusage port/event sinkと終了境界を接続し、既存Web契約の`callUsageUpdated`、`callUsageComplete`、fatal時の`callFatalError`を、既存field名・型・ミリ秒単位のまま通知する。AndroidからSupabaseへ直接書き込まず、`public.users.id`解決、owner付きoutbox、残高判定、通話終了時の一括精算、冪等な未精算管理は既存のOS共通TypeScript/Web・DB側契約を維持する。call ID・turn ID・録音世代・TTS世代・再生世代・session世代を照合し、古い録音/TTS/socket callback、reconnect前の処理、終了後の通知を無視する。turn finalized状態、`terminating`、`usageCompleteSent`、`fatalErrorSent`によりusage二重確定、usage完了二重送信、fatal終了二重送信を防ぐ。終了時はSessionを先に`STOPPING`へ変更して新規処理を禁止し、開いているuser usageを確定・通常再生を安全に取消した後、録音→TTS→socketの順に停止し、turn usage確定→`callUsageComplete`（fatal時は続けて`callFatalError`）の順で一度だけ通知する。工程5 JUnit 31件を追加し、ローカルは`git diff --check`と工程1〜5専用Node構造テスト計49件がPASS。ローカルGradleは実行していない。実装commit `d4ad83e0000798cae4be19bde93dd77ae34ed423`。GitHub Actions `Android On-Device TTS Test` run `30008775873`は`:app:compileDebugJavaWithJavac`、`:app:compileDebugUnitTestJavaWithJavac`、工程5 JUnit 31件、既存工程1〜4とオンデバイスTTS JUnitを含む`:app:testDebugUnitTest`がすべてsuccess（`BUILD SUCCESSFUL`）。旧`AndroidTtfaTestPlugin`、`src/appMain.ts`、`MainActivity.java`、spec v4.9は変更せず、旧Pluginをstaging実通話経路として維持する。main、production、DB、migration、RPC、R2、env、secret、Edge Function、iOS、AABは変更・実行していない。工程5は完了したため移行順序5を`[x]`とする。次工程は6「staging切替とAndroid実機確認」。PR #69はOpen・Draft・未マージを維持する。）

（v5.52: 正式Android通話実装の工程4「オンデバイスTTS・固定聞き返し再生・録音準備音再生の正式Controller移行」を完了した。正式packageへオンデバイス専用`VoiceTtsController`を追加し、通常AI回答の文単位合成・順序付き再生、固定聞き返しのオンデバイス再生、最終文完了後の録音準備音、再生開始/完了/失敗/取消、reconnect・新turn・通話終了時の停止を責務化した。既存`OnDeviceTtsController` / `OnDeviceTtsEngine`を単一の永続pipelineとして包み、共通BERT session、キャラ別TTS session・voice config、常駐AudioTrack、合成executor、再生executor、既存`TtsSentenceSplitter`を再利用する。キャラ変更時だけキャラ側モデルを切り替え、文・turnごとのsession、AudioTrack、executor再作成は追加しない。録音準備音はAAB 1060で確認済みの880Hz・90ms・振幅0.28・fade 8ms、`MODE_STREAM`、API S以降の実効start threshold／旧APIのbuffer capacity取得、threshold不足分だけの末尾無音padding、既存ToneGenerator fallback値を変更していない。call ID・turn ID・TTS要求世代・再生世代・モデル準備世代・character IDを内部pipeline/cue IDと照合し、stop/reconnect/新turn/キャラ変更後の古い合成結果、再生完了、固定聞き返し完了、録音準備音完了、モデル準備完了を無視する。`VoiceCallSession`へ狭いTTS port/event sinkを追加し、TTS eventに録音世代を保持して、通常再生完了・固定聞き返し完了・録音準備音完了またはcue開始失敗時だけSession経由で`VoiceRecordingController`の世代検証付き復帰境界へ通知する。通常回答と固定聞き返しを区別し、固定聞き返しはusage対象にしない。クラウドTTS API、音声URL、mp3、クラウドfallback、token/endpoint診断、会話本文/PCMログは正式Controllerへ移していない。工程4 JUnit 27件と専用Node構造テスト12件を追加し、ローカルは`git diff --check`と工程1〜4専用Nodeテスト計37件がPASS。ローカルGradle、Android SDK download、全件`npm test`、`npm run build`、`npx tsc --noEmit`は実行していない。実装commit `f261277317a6d10f65468cffb14d45f09d155959`。GitHub Actions `Android On-Device TTS Test` run `30006799089`は`:app:compileDebugJavaWithJavac`、`:app:compileDebugUnitTestJavaWithJavac`、工程4 JUnit 27件、既存`VoiceRecordingControllerTest`、`VoiceConnectionControllerTest`、`VoiceCallSessionTest`、オンデバイスTTS JUnitを含む`:app:testDebugUnitTest`がすべてsuccess（`BUILD SUCCESSFUL`）。旧`AndroidTtfaTestPlugin`、`src/appMain.ts`、`MainActivity.java`、spec v4.9は変更せず、旧Pluginをstaging実通話経路として維持する。main、production、DB、migration、R2、env、secret、Edge Function、iOS、AABは変更・実行していない。工程4は完了したため移行順序4を`[x]`とする。次工程は5の課金・終了処理移行。PR #69はOpen・Draft・未マージを維持する。）

（v5.51: 正式Android通話実装PR3「VoiceRecordingController移行」を完了した。旧`AndroidTtfaTestPlugin`の録音実装から、24,000Hz・mono・PCM16、VOICE_RECOGNITION、RMS閾値0.02、最小発話300ms、発話後無音700ms、最大録音8,000ms、PCM送信後のVAD判定と`speech_end`送信順序を確認し、正式packageへ`VoiceRecordingController`を追加した。Controllerはcall ID・turn ID・録音世代、AudioRecord wrapper、録音/start待ち、VAD、speech_end送信済み、固定聞き返し/録音準備音/reconnect後の復帰待ちだけを所有する。録音入力・時計・executor・イベント・PCM/JSON送信先を差し替え可能にし、単一の再利用executor上でread loopを直列化した。PCMはread bufferをoffset/length付きで`VoiceConnectionController.sendPcm`へ直接渡し、JSON化や`Arrays.copyOf`を追加しない。音声検出後の発話終了、無音最大時間の固定聞き返し要求、固定聞き返し完了後の録音復帰、AI音声完了→録音準備音要求→現在世代の完了callback後の録音復帰、reconnect開始時停止→異なる新active turn ready後だけの復帰を境界化した。call/turn/録音世代が古いread・固定聞き返し完了・録音準備音完了・socket readyを無視し、旧callbackが新録音を停止/再開しない。`stop`は冪等でAudioRecord stop/releaseと世代無効化を行い、終了後のPCM/`speech_end`を禁止する。`VoiceCallSession`はactive ready、reconnect開始、fatal/通話終了を録音portへ通知し、録音eventをSession event sinkへ転送する。TTS生成、固定聞き返し文言、録音準備音の音/AudioTrack、usage、socket所有はControllerへ含めない。JUnit 22件とPR3専用Node構造テスト8件を追加し、既存Android CIへ新JUnitを追加した。実装commit `305933b637d0943fd197270ca4a0fecbf174ff28`。ローカルは`git diff --check`とPR3/PR1/PR2専用Nodeテスト計25件がPASSし、Gradle、Android SDK download、全件`npm test`、`npm run build`、`npx tsc --noEmit`は実行していない。GitHub Actions `Android On-Device TTS Test` run `29996719317`は`:app:compileDebugJavaWithJavac`、`:app:compileDebugUnitTestJavaWithJavac`、`VoiceRecordingControllerTest`、既存`VoiceConnectionControllerTest`、`VoiceCallSessionTest`、オンデバイスTTS JUnitを含む`:app:testDebugUnitTest`がすべてsuccess。旧`AndroidTtfaTestPlugin`、`src/appMain.ts`、`MainActivity`、spec v4.9は変更せず稼働経路を維持する。録音責務の正式Controller移行は完了したため移行順序3を`[x]`とする。次工程は4のオンデバイスTTS・固定聞き返し再生・録音準備音再生の正式Controller移行。PR #69はOpen・Draftを維持し、main、production、DB、migration、R2、env、secret、Edge Function、iOSは変更しない。）

（v5.50: 正式Android通話実装PR2「VoiceConnectionController移行」を完了した。旧`AndroidTtfaTestPlugin`の接続責務を実コードから確認し、正式packageへ`VoiceConnectionController`を追加した。Controller APIはincoming standby準備・取消、active開始、次turn standby準備・昇格、手動/自動再接続、全socket終了、PCM binary送信、JSON送信、turn context送信を提供する。所有状態はactive/standbyのsocket・turn ID・ready/connect中フラグ、incoming準備、close要求、接続ready一回通知、再接続中フラグ、自動再接続回数(最大1回)に限定し、通話状態遷移は`VoiceCallSession`の`ConnectionEventSink`へ`ACTIVE_CONNECTING` / `ACTIVE_READY` / `STANDBY_CONNECTING` / `STANDBY_READY` / `STANDBY_PROMOTED` / `CALL_CONNECTION_READY` / `RECONNECT_REQUIRED` / `RECONNECT_STARTED` / `RECONNECT_SUCCEEDED` / `SOCKET_FAILURE` / `SOCKET_CLOSED` / `FATAL_CONNECTION_ERROR`として通知する。ユーザー発信は事前standbyを閉じて必ず新規active socket、疑似着信応答は`allowPreparedConnection=true`かつURL・token・character一致時だけstandby昇格、通話中の次turn standbyは明示APIでのみ準備・昇格する。callbackはsocket identity＋call/turnで現役判定し、stale message/failure/closedを無視する。active/standbyが同一socketなら1回だけcloseし、failure→reconnect開始→旧socket close→新active接続の順序を固定した。PCMは`byte[]`のoffset/lengthから直接`ByteString`へ渡しJSON化・中間`Arrays.copyOf`を行わず、text messageは1回だけ`JSONObject` parseしてtyped viewをSessionへ渡す。新executor/thread hopは追加せず、OkHttp callbackと公開APIは同じ`synchronized`境界へ入る。診断へURL全文、access token、PCM、会話本文を渡さない。JUnit 19件とPR2専用Node構造テスト9件を追加し、旧Pluginと`src/appMain.ts`のhash不変、新旧Plugin登録維持を確認した。実装commit `44aa6d8`、診断helperのJava overload曖昧性修正commit `be89343`。ローカルは`git diff --check`、PR2/PR1専用NodeテストのみPASS。GitHub Actions初回run `29988764077`は診断helperの`null` overload曖昧性でJava compile失敗し、PR2範囲でhelper名を分離後、run `29988976625`で`:app:compileDebugJavaWithJavac`、`:app:compileDebugUnitTestJavaWithJavac`、`VoiceConnectionControllerTest`、既存`VoiceCallSessionTest`、既存オンデバイスTTS JUnitを含む`:app:testDebugUnitTest`がsuccess。旧Pluginと`src/appMain.ts`は稼働経路のままで、AudioRecord・VAD・speech_end判定の録音移行は未着手。移行順序3は接続完了・録音未了の`[~]`とし、次工程は`VoiceRecordingController`移行。spec変更なし。PR #69はOpen・Draftを維持し、main、production、DB、migration、R2、env、secret、Edge Function、iOSは変更しない。）

（v5.49: 正式Android通話実装PR1を完了した。現行責務と移行境界を、通話状態・接続・録音・オンデバイスTTS・usage・イベント通知・診断に分離し、正式Capacitor Plugin `AndroidVoiceCall`、thread-safeな`VoiceCallSession`状態機械、controller port、低負荷診断境界を追加した。正式Pluginは未実装のlive controllerへ接続せず明示的な`not_implemented`を返し、`src/appMain.ts`は引き続き旧`AndroidTtfaTestPlugin`を選択するため稼働経路を変更しない。旧Pluginはbyte-for-byte不変で、クラウドTTS・AudioRecord・WebSocket・オンデバイスTTS実装を正式骨組みへ移していない。実装commit `fe252ba`、既存Android CIへの正式通話JUnit追加commit `ade4151`、Capacitor `PluginMethod` import修正commit `531d4de`。ローカルは`git diff --check` PASS、PR1専用`node --test tests/androidVoiceCallSkeleton.test.mjs` 1/1 PASSのみ実行し、Gradle、Android SDK download、全件`npm test`、build、TypeScript再検証は行っていない。GitHub Actions `Android On-Device TTS Test`の初回run `29985863951`は新規Pluginの`PluginMethod` import誤りでJava compile失敗し、PR1範囲の1行修正後run `29986065015`で`:app:compileDebugJavaWithJavac`、`:app:compileDebugUnitTestJavaWithJavac`、`VoiceCallSessionTest`を含む`:app:testDebugUnitTest`がすべて成功した。移行順序1・2を完了とし、次は3の接続・録音移行。PR #69はOpen・Draftを維持し、main、production、DB、migration、R2、env、secret、Edge Function、iOSは変更しない。）

（v5.48: Android正式通話実装の最終方針と移行順序をspec v4.9に対応させた。`AndroidTtfaTestPlugin`は正式実装として残さず、責務分割した新しいAndroid通話実装への移行完了後に丸ごと削除する。新実装はオンデバイスTTS専用とし、クラウドTTS、クラウドTTSフォールバック、クラウドTTSの生成・受信・再生・切替分岐を移さない。モデルsession、AudioTrack、executorを不要に再作成せず、責務分割による余計なJSON変換、PCMコピー、スレッド切替を増やさない。移行工程を、①現行責務・依存関係の調査、②正式実装の骨組み作成、③接続・録音移行、④オンデバイスTTS移行、⑤課金・終了処理移行、⑥staging切替と実機確認、⑦旧Plugin・クラウドTTS完全削除、⑧初回音声・文間遅延改善の8段階に固定した。**AAB 1060のAndroid実機で、録音準備音は可聴、固定聞き返しはオンデバイスTTS経由で再生、通話画面のボタン位置は解決済みと確認した。初回音声までの長い待ちと文間遅延は未解決。** PR #69は本更新着手時点でOpen・Draft、branch `agent/staging-tts-vite-config`、HEAD `4af0f1c46a74ee7ba0e799dfc08b4ba4f18cf6c6`。PR本文はv5.42時点の古い内容だが今回は変更しない。本更新はdocsのみで、実装コード、main、production、DB、migration、R2、env、secretは変更しない。）

（v5.47: AAB 1059のAndroid実機診断で、通常AI音声は再生される一方、AI発話後の録音準備音は複数回とも不可聴だった。`readyCue:`はengine経路の選択、playbackExecutor実行、`playCueTone()`到達、44.1kHz・880Hz・90ms・3969 samples生成、3969 samplesのwrite完了、`play()`後のPLAYING遷移、140ms後の録音再開を示し、例外・途中停止・fallbackはなかった。原因は、MODE_STREAM AudioTrackの再生開始に必要なbuffer/start thresholdが最低4096 framesであるのに対し、cueが3969 framesで127 frames不足し、`play()`後もplayback head 0のまま実再生を開始できなかったこと。修正commit `b3a5cb00c3f4601bf03463281f76d924fffa69bb`で、880Hz・可聴90ms・音量・AudioTrack usage・ToneGenerator fallbackを変えず、API 31以上は実際の`getStartThresholdInFrames()`、API 24〜30はbuffer capacityを必要framesとして取得し、不足分だけcue末尾へ無音PCMをpaddingしてから既存AudioTrackへwriteする。固定127 framesにはせず端末・route差へ追従する。診断へ`bufferSizeFrames` / `bufferCapacityFrames` / `startThresholdFrames` / `audibleSamples` / `paddingSamples` / `writeSamples`と、録音開始直前の`playbackHeadFrames` / `writtenFrames`を追加した。通常TTS、課金、usage、DB、接続方式、TTS合成内容は変更しない。自動確認: `git diff --check` PASS、`npx tsc --noEmit` PASS、`npm run build` PASS、対象NodeテストPASS。`npm test`は246件中245件PASS、唯一のfailは未変更の既知・範囲外`iosVoiceCall.test.mjs`版数pin(v5.42期待、現行v5.47)で今回変更と非関連。Android On-Device TTS Test run `29945664209`は新規padding JUnitを含めsuccess。Android Staging AABはworkflow run `29945902487`、run number 60、HEAD `b3a5cb00c3f4601bf03463281f76d924fffa69bb`、versionCode `1060`、versionName `staging.1.0.60`、artifact `android-staging-aab`、build success。v5.47更新時点ではAndroid実機でのpadding値・playback head進行・ピィ音可聴は未確認であり、復旧済みとは扱わなかった。この未確認状態はv5.48のAAB 1060実機結果で更新済み。PR #69はDraft維持。production、DB、migration、R2、env、secretは変更しない。）

（v5.46: Android実機AAB 1057ではengine経由の録音準備音(ピィ音)も鳴らなかった。原因は未特定で、ピィ音復旧は未完了のまま。鳴らし方、音量、周波数、持続時間、ToneGeneratorのstream、AudioTrackのusageは変更せず、既存の折りたたみデバッグ画面へ`readyCue:`診断計装だけを追加した。1回のcueごとに、`startRecordingAfterReadyCue()`の開始条件(`recordingReadyCueRequired`、`onDeviceTtsIntended`、controller有無、call/turn識別子、standby昇格/active ready)、engine呼出しと戻り値、未初期化時の`ttsSession`/`bertSession`/`voiceConfig`/`audioTrack`状態、ToneGenerator fallback、playbackExecutorのqueue/開始/終了/スレッド/例外、`playCueTone()`のsample rate/周波数/持続時間/PCMサンプル数、`beginUtterance()`前後と`play()`前後のplay state/playback head、`appendUtterance()`サンプル数、AudioTrack `write()`戻り値/合計write数/cue完了予定時刻、cueを消し得るbeginUtterance/stopPlayback/pause/flush/release/controller stop/次turn/socket close・再接続、completionと140ms delay実測値、cue開始から録音開始までの経過を時系列で記録する。通常画面・会話本文・PCM内容・認証情報・URL・秘密情報には露出しない。実装commit `add4988225bdafe58ef6a3cabfe5d6ff7b6baead`、Javaコンパイル修正commit `e9f05445a25524ae7ec07ef77d018c8baa431f57`、非同期JUnit待機修正commit `bf4a40c1ae2c61ed1108c58a4b6b1c5a9bdf5ab2`。ローカル自動確認: `git diff --check` PASS、`npx tsc --noEmit` PASS、`npm run build` PASS、対象Nodeテスト59件PASS。`npm test`は245件中244件PASS、唯一のfailは既知・範囲外の`iosVoiceCall.test.mjs`「iOS第1段階の採用方式と範囲外をbuild planへ固定する」で、未変更テストが旧版数v5.42を固定期待しているため。v5.44時点でも同じ既知failが記録済みで今回変更とは非関連。Android On-Device TTS Testは最初のrun `29938057772`が今回追加箇所のJavaコンパイル不備、修正後run `29938469743`が今回追加テストのexecutor完了待機競合で失敗し、待機を修正したrun `29938916134`で26件PASS・workflow success。Android Staging AABは実装HEAD `e9f05445a25524ae7ec07ef77d018c8baa431f57`、workflow run `29938480330`、run number 59、versionCode `1059`、versionName `staging.1.0.59`、artifact `android-staging-aab`、build success。後続`bf4a40c`はテスト待機だけの変更でアプリ内容は同一のためAABを重複生成していない。Android実機での`readyCue:`ログ取得、ピィ音の可聴、原因特定はいずれも未確認。PR #69はDraft維持。production、DB、migration、R2、env、secret、課金、usage、接続方式、TTS合成内容は変更しない。）

（v5.45: オンデバイスTTS導入後に鳴らなくなっていた録音準備音(ピィ音=AIの発話完了後に次の発話を促す短いビープ)の復旧を試み、engine経由でビープを鳴らす実装を追加した。`TtsEngine`インタフェースに`void playCueTone()`をseamとして追加、`OnDeviceTtsEngine.playCueTone()`はサイン波を常駐トラックのサンプルレートで生成し`beginUtterance()`+`appendUtterance()`で再生、`OnDeviceTtsController.playRecordingReadyCue()`はengine初期化済みなら再生consumer(playbackExecutor)へ積んで`true`を返す、`AndroidTtfaTestPlugin.startRecordingAfterReadyCue()`はまずengine経由を試み失敗時は既存`ToneGenerator`へフォールバックする。commit `e3d7ff9`。Android Staging AAB: workflow run `29934124409`、run number 57、branch `agent/staging-tts-vite-config`、HEAD `e3d7ff92e443e7a31606c29526087122e3451ce1`、versionCode `1057`、versionName `staging.1.0.57`、buildはsuccess、PR #69はDraftのまま。**AAB 1057のAndroid実機結果: engine経由のピィ音も鳴らなかった。原因は未特定。「常駐AudioTrackとの競合(マスク)が根本原因」とした当初の判断は実機結果と一致しないため取り消す。ピィ音復旧は未完了。** 次工程は追加修正ではなく原因調査(playRecordingReadyCueが呼ばれるか/playbackExecutorが実行されるか/playCueToneに到達するか/生成PCMサンプル数とwrite結果/write後にbeginUtterance・pause・flush・stopPlaybackが走らないか/90ms再生完了前に録音開始が走らないか/ピィ音処理時のAudioTrack play stateとplayback head)。PR #69はReady化・マージ不可、Draft維持。本更新はdocsのみ訂正で、spec・コード・production・DB・R2・env・secretは変更しない。）

（v5.44: AAB 1049で確認したユーザー発信の回帰(3コール後にデバッグ画面→「接続中」のまま停止・コール音鳴り続け)を調査し、原因特定と修正を行いAndroid実機で解消を確認した。**原因**: ユーザー発信でも以前`prepareIncomingCall`経由でstandby事前接続socketを張り、`startVoiceCall`でそれを昇格(promote)していた。昇格後は`isStandbySocket()`がfalseになりonOpenで`active_success`だけが出る一方、そのstandby socketのWS HTTP-101ハンドシェイクに約55秒かかり、Web側の20秒接続ウォッチドッグ(接続中)を超過して「接続中」停止・発信コール音が鳴り続ける回帰になっていた(初回active socket接続自体の遅延ではない)。**診断計装**(先行commit): `c14fa1f`(`model_load_started`マーカー＋WSライフサイクル診断=connEventパネル追加)、`40f2db5`(ready抑止理由の診断出力＋callId一致`active_ready`時のみウォッチドッグ解除＝遅延接続回復フォールバック)。**修正** commit `50b4c84`: ユーザー発信と疑似着信の経路を明確に分離した。ユーザー発信は`prepareIncomingCall`によるstandby事前接続を使わず、開始判定・call作成後に通常の`connectActiveSocket`経路で新規active WebSocketを張る。standby昇格の許可は疑似着信の応答時(`special_event`)のみとし、`startVoiceCall`へ`allowPreparedConnection`フラグを渡してnative側`usePreparedConnection`をこのフラグでもゲートする。疑似着信のstandby事前接続・応答時昇格・TTS warmUp・課金・usage・DB処理・既存20秒ウォッチドッグは不変。旧挙動を検証していたテスト(`tests/iosVoiceCall.test.mjs` #112、`tests/onDeviceTtsIntegration.test.mjs` #136、`tests/inAppIncomingCall.test.mjs`)を新しい分離挙動(世代bumpのみ・`prepareIncomingCall`不使用・`allowPreparedConnection: source === "special_event"`・native `getBoolean`ゲート・`ttsBundleConfig`は開始/準備の2箇所)へ更新した。自動テスト(ローカル): `npx tsc --noEmit` PASS、`npm test` 243件中242件PASS(唯一のfailは既知・範囲外の#129 iOS `iosVoiceCall.test.mjs`のみ)、`npm run build` PASS。Android Staging AAB: workflow run `29913668224`、run number 56、branch `agent/staging-tts-vite-config`、HEAD `50b4c8459a6dbd2cdbb0eca0bff2ff97bef9e762`、versionCode `1056`、versionName `staging.1.0.56`、buildはsuccess、PR #69はDraftのまま。**AAB 1056のAndroid実機結果(ユーザー発信): 1回目から問題なく通話・会話できることを確認。回帰(「接続中」停止・コール音鳴り続け)は解消。この問題は解決とする。** 補足: 調査途中で「character bundle未完了・model_load未開始のまま会話が始まり音声経路が破損している」疑いが挙がり、TTS準備(bundle導入＋model_load)完了まで録音開始をゲートする追加修正を検討したが、その実機確認はアプリ再インストール直後でTTSモデルのDLが未完了だったための事象と判明したため、追加実装・commit・pushは一切行わず取り消した(コード変更なし)。TTSモデルDL完了後は1回目から正常に話せる。PR #69はReady化・マージ不可、Draft維持。本更新はdocsのみで、spec・コード・production・DB・R2・env・secretは変更しない。）

（v5.43: PR #69(Android追加実装・branch `agent/staging-tts-vite-config`・最新HEAD `ae3b1ce64ef631c38344b2871210b0bf2a74fd75`)のCI・staging deploy・staging AAB作成・Android実機結果を事実記録した。追加実装: ①オンデバイスTTS正常再生後の録音準備音(ピッ音)復旧、②`prepareIncomingCall`で導入済みモデルのみのmodel_load prewarm、③TTS合成producerとAudioTrack再生consumerの分離、④通常回答を最大2文・約80文字目安にするサーバー側制御、⑤固定聞き返しをAndroidオンデバイスTTSで再生する経路、⑥JUnitテストのコンパイル修正。commit: `7eba583`(ピッ音復旧) / `0f0dc1e`(model_load prewarm) / `4b1b8d2`(合成producer/再生consumer分離) / `7f4327f`(最大2文制限) / `05c513a`(固定聞き返しオンデバイス化) / `ae3b1ce`(JUnitテストコンパイル修正)。自動テスト(ローカル): `npm test` 228件PASS、`npx tsc --noEmit` PASS、`npm run build` PASS、`git diff --check` PASS。Denoはローカル未導入のため未実施。GitHub Actions: `Android On-Device TTS Test`(run `29847433103`, HEAD `ae3b1ce`, success)、`Android TTS Bundle Installer Test`(run `29847435776`, HEAD `ae3b1ce`, success)。最初のrun `29846761486` / `29846763742`は追加JUnitテストのヘルパー引数型不一致で失敗し、`ae3b1ce`で修正後にgreen。staging反映: staging `voice-turn`へPR #69最新HEADをdeploy(`voice-turn`はVERSION 13、`chat-reply`は変更なし。production・DB・migration適用・R2・env・secretは変更なし)。Android Staging AAB: workflow run `29848258099`、branch `agent/staging-tts-vite-config`、HEAD `ae3b1ce64ef631c38344b2871210b0bf2a74fd75`、versionCode `1049`、versionName `staging.1.0.49`、artifact `android-staging-aab`、buildはsuccess、PR #69はDraftのまま。**AAB 1049のAndroid実機結果(ユーザー発信): 3コール後にデバッグ画面へ移動→「接続中」のまま停止、コール音が鳴り続け通話開始できない。AAB 1048ではユーザー発信できていたため1049で回帰を確認。**(AI着信): 応答後の会話は可能。1文目と2文目の無音は体感上改善を確認できず。通常回答再生後のピッ音は鳴らなかった。(固定聞き返し): 無言時に音声は再生されたが、実機上ではオンデバイス経路か判定できておらず、再生後のピッ音は鳴らなかった。(操作バー): スピーカー切替/終話ボタンの位置は改善しておらず、既存WindowInsets修正の実機効果は確認できなかった。未完了(すべて未完了として維持): ユーザー発信回帰の原因特定と修正／ピッ音復旧／文間無音削減／固定聞き返しのオンデバイス経路確認／操作バー修正／AndroidオンデバイスTTS実用確認。オンデバイスTTS工程全体も未完了。PR #69はReady化・マージ不可、Draft維持。次工程は実装ではなくAAB 1049の回帰原因調査(原因は未特定のため推測は記さない)。本更新はdocsのみで、spec・コード・production・DB・R2・env・secretは変更しない。）

（v5.42: staging実機でオンデバイスTTSを検証するため、既存の複数キャラ保有方式のまま桜音・花音を追加・トークできる最小UIを`src/appMain.ts`へ追加した。従来「キャラ追加機能は後続PRで実装します」だったキャラ管理画面を、staging限定で桜音・花音を追加・トークできるUIへ置き換えた(このアプリは「現在キャラ1人固定」ではなく、複数の`user_characters`を保有して押したキャラごとにトーク・通話する)。桜音・花音を押すと、未保有なら`user_characters`へ1行upsertする。`display_order`は0固定にせず既存の最大`display_order`+1を割り当てて末尾に加えるだけで、他キャラの`display_order`は変更しない。`selectedCharacterId`を「現在キャラ」として永続化はしない。追加後は押したキャラの`characterId`を`state.chatCharacterId`へ入れてチャット画面を開き、通話開始時は既存どおり`chatCharacterId`をそのまま`ttsBundleConfig(characterId)`へ渡して桜音→`sakune`/花音→`kanon` bundleを解決する(未登録IDや不正JSONは空設定=オンデバイス無効で安全)。対象は`VITE_TTS_CHARACTER_BUNDLES`のキー(=Supabase `characters.id`)に一致するキャラだけで、UUIDはコードへ直書きしない。有効化条件は`voiceDebugEnabled && ttsBundleCharacterIds.length>0`のため、本番(env未設定)ではキャラ管理画面は従来のプレースホルダのまま、追加UIも`characters`への花音取り込みも起きない。`refreshData`のフィルタへbundleキー一致キャラの取り込みを追加した。DB migration・新RPCは追加しない。R2自動DL・キャッシュ・失敗時の非fallback等のnative側挙動は変更していない。`npm test` 205件(新規`tests/stagingCharacterSwitch.test.mjs` 12件を含む。まお保持のまま桜音・花音を追加でき3キャラ同時保有、追加で他キャラの`display_order`を変えないことを検証)、`npx tsc --noEmit`、`npm run build`、`git diff --check`はPASS。**未実施のまま: 本UIを含む新しい`Android Staging AAB` build、Android実機でのオンデバイスTTS工程(model_config〜playback_completed)確認。既存AAB `staging.1.0.44`(run 44)は本UIを含まず、実機でオンデバイスTTSを検証できていない。実機確認完了までオンデバイスTTS工程は未完了とする。** 本更新でspec・production・staging DB・R2は変更しない。花音のmigration `20260721090000_add_kanon_character.sql`は適用しない。）

（v5.41: v5.40で「未実施」としていたstaging配信設定の残りを実施済みへ更新。桜音・花音のstaging `characters.id`を取得(桜音=`3ff18df6-e2e2-49ee-afe8-01257ed53fb5`、花音=`d7410a62-3e02-4d17-b272-14ba0262ae3a`)。花音はstaging DBへ手動追加済みで、これを再現するidempotent migration `supabase/migrations/20260721090000_add_kanon_character.sql`を追加済み(適用はしていない)。GitHub staging環境変数3件(`VITE_TTS_COMMON_BUNDLE_URL` / `VITE_TTS_COMMON_VERSION` / `VITE_TTS_CHARACTER_BUNDLES`)を設定済みで、`VITE_TTS_CHARACTER_BUNDLES`は上記2つの`characters.id`をキーに`sakune`/`kanon` bundleへ対応付ける。R2公開URL取得確認は成功済み(workflow run `29755001244`)。**未実施のまま: `Android Staging AAB` build、Android実機確認、オンデバイスTTS工程全体。** 本更新でspec・production・staging DB(適用)・AABは変更しない。docsのみ更新。）

（v5.40: R2アップロードworkflowを`public_base_url`付きで再実行し、公開URLからの取得確認(サイズ＋SHA-256一致)が成功した(workflow run `29755001244`)。公開base URLは`https://pub-7bcdc02f44a14b68a333bc9b9ee07f73.r2.dev`で、common=`/common/common-staging-bundle.zip`、桜音=`/characters/sakune/sakune-staging-bundle.zip`、花音=`/characters/kanon/kanon-staging-bundle.zip`。あわせてstaging AAB build workflow(`Android Staging AAB`)へ`VITE_TTS_COMMON_BUNDLE_URL` / `VITE_TTS_COMMON_VERSION` / `VITE_TTS_CHARACTER_BUNDLES`をGitHub staging環境変数(`vars.*`)から注入する配線と、確定値(common URL・common version=`1`・bundle id `sakune`/`kanon`とR2 URL)を検証する事前チェック(`scripts/tts_bundle_ci/verify_staging_tts_env.mjs`)を追加した。common versionはbundle生成workflowの`COMMON_VERSION="1"`、bundle idは`sakune`/`kanon`で、いずれもコード・bundle定義から確認した(憶測なし)。ただし`VITE_TTS_CHARACTER_BUNDLES`のキーはSupabaseの`characters.id`(=`gen_random_uuid()`)で環境依存のためコードからは確定できず、かつ花音は現行migrationのcharactersシードに存在しない(桜音のみ)。したがって実キー(桜音/花音のstaging `characters.id`)はstaging DB由来の値としてGitHub staging環境変数側で設定する必要があり、花音は未シードならstaging DBへの登録が前提となる。未設定時はTTS設定オフ(既存挙動)で既存staging AABビルドを壊さない。productionは変更しない。Android実機確認は未実施で、オンデバイスTTS工程全体は未完了。docs＋staging workflow＋検証scriptのみ変更。）

（v5.39: PR #67をmainへマージした(merge commit `45e6c176cf30493f78c6037504856902af1781a1`)。共通BERT＋キャラ別bundleの分割配信実装・R2アップロードworkflowがmainに入った。手動実行専用のR2アップロードworkflowを実行し、common／桜音／花音の3bundleをCloudflare R2(staging)へアップロード成功(workflow run `29751863325`)。アップロード後にサイズとSHA-256 read-backの検証が成功した。ただし`public_base_url`は空で実行したため、公開URLからの取得確認は未実施。`VITE_TTS_COMMON_BUNDLE_URL` / `VITE_TTS_COMMON_VERSION` / `VITE_TTS_CHARACTER_BUNDLES`の設定は未実施。Android実機確認も未実施。したがってオンデバイスTTS工程全体は未完了のまま(「R2アップロード完了、公開URL確認・VITE_TTS_*設定・実機確認待ち」)。本更新はdocsのみで、spec・コード・productionは変更していない。）

（v5.38: オンデバイスTTSの声モデル配信を単一bundleから「共通BERT＋キャラ別モデル」方式へ分割実装し、桜音＋花音の2キャラでCI検証まで完了した。実態は、共通BERT bundle1個(DeBERTa BERT＋共通ライセンス)＋桜音character bundle＋花音character bundleの3構成。Android側は共通BERTを端末へ1回だけ取得・保存して再利用し、既取得キャラは再DLせず、キャラ切替(桜音→花音→桜音)でも共通・既取得キャラモデルを再DLしない。片方のDL/検証/更新失敗は既存の正常インストールを壊さない(原子的入替＋失敗時ロールバック)。設定はcharacter_idごとのURL方式(`VITE_TTS_COMMON_BUNDLE_URL` / `VITE_TTS_COMMON_VERSION` / `VITE_TTS_CHARACTER_BUNDLES`)。CIは実モデルから共通1＋キャラ2を動的INT8量子化・組立・検証する(各キャラのstyle_vectorは各AIVMXのONNX metadata `aivm_style_vectors` から抽出、桜音は既存golden fixtureと一致を検証)。実モデルはGit/AABに入れずCI内のみで扱う。bundle payload実サイズ: common=475,567,269B(約453MiB)、桜音=255,610,450B(約244MiB)、花音=251,080,348B(約239MiB)。端末DL量=共通＋当該キャラ(桜音約697MiB、花音約693MiB)、両方導入でも共通は1回=約937MiB。CIは`Build on-device TTS model bundles (staging)` run `29744823614` と `Android On-Device TTS Test` run `29744823724` がgreen。**Cloudflare R2は未設定・未アップロード、実機確認は未実施。本項は完了ではなく「実装・CI完了、R2配信・実機確認待ち」。** PR #67はDraft維持。specはv4.8へ共通BERT＋キャラ別の配信構成を最小限追記(製品方針は不変)。本更新でmain・production・R2は変更していない。）

（v5.37: 正本 voice-companion-docs のドキュメント2件(オンデバイスTTS採用反映=旧spec v4.6/build_plan v5.25、PR #48 mainマージ状態記録=旧build_plan v5.26)を当リポジトリのドキュメントへ統合した。具体的には、フェーズ0の声モデル項へTTS方式決定を追記、疑似電話のPR #48行へmainマージ済み(2026-07-14、merge commit `1e3bb3d35179eb62216895c749068ebdcd14ec22`)を追記、「オンデバイスTTS本実装(採用決定・2026-07-18)」節を新設、Z-6を更新した。specはv4.7へ統合済み(iOSネイティブ録音反映とオンデバイスTTS採用を併存)。当リポジトリの既存v5.25/v5.26(iOS疑似電話関連)と版数が衝突するため、統合版をv5.37とした。オンデバイスTTSはv1の6キャラ全員が対象で、桜音は方式・量子化・配信の検証基準モデルとして用いる(桜音1モデルに絞る前提にはしない)。本更新は文書のみで、コード・DB migration・Supabase・Edge Function・productionは変更しない。）

（v5.36: iOS初回応答遅延をAndroidユーザー発信経路と再照合し、ユーザー操作後に古い着信準備の破棄完了を最大2秒待つ直列経路、古いprepare結果から新しい発信standbyを破棄し得る遅着cancel、iOSだけが`AVAudioPlayer`生成と`prepareToPlay()`を通話状態queue上で同期実行する差を確認した。計測開始をDOMの開始操作guard先頭へ移し、試行IDで二重開始・失敗・遅着eventを分離した。ユーザー発信は古いcleanupを待たずnative state queue上のstandby置換とDB guardを並行し、古いprepare結果はgenerationで破棄する。iOS player準備はAndroidの`prepareAsync()`と同じく状態管理から分離し、`first_audio_received` / `first_audio_player_ready` / `first_audio_play_invoked` / 再生clock進行後の`first_audio_play_started`を両OS診断へ追加した。再生API成功前の課金開始、usage、精算、Android通話状態機械は変更していない。`npm test` 166件、`npx tsc --noEmit`、`npm run build`、`npx cap sync ios`、`git diff --check`をPASSした。iOS実機は未検証、TestFlight/Codemagicビルドは未実施、productionは未変更。）

（v5.35: Android通話のaudio focus変化を、`AUDIOFOCUS_LOSS` / `AUDIOFOCUS_LOSS_TRANSIENT`だけ安全終了の対象とし、通知音やナビ音声が要求する`AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK`は音量操作も終了処理も行わず通話を継続するよう修正した。iOSの`AVAudioSession` interruption処理、自動復帰、DB migration、Supabase、Edge Function、productionは変更しない。）

（v5.34: Android実機PASS済みの通話状態機械を基準に、iOSへSTT連続失敗回数・回復音声完了待ち・standby昇格・3回目手動復旧、ターン未進行時だけの自動再接続、再接続失敗時のusage確定と全解放、再生成功後だけの課金開始、chunk再生失敗処理、確定・再生済みturnだけの短期履歴を移植した。iOS通話中のidle timer・近接監視・plugin破棄・本体受話口/スピーカー限定をAndroidと揃え、両OSで音声割り込みを検出した場合は自動復帰せず安全終了して共通outbox・一括精算へ合流する。Bluetooth通話対応と割り込み後の自動復帰は将来課題とする。DB migration、Supabase、Edge Function、productionは変更しない。）

（v5.33: iOS初回応答遅延の開始経路をAndroidと比較し、ユーザー発信では両OSともDB outbox照合・残高確認・call作成と並行して`standby=1` WebSocketを先行接続し、native開始時に昇格する構成へ統一した。iOS固有でWebSocket `ready`後に直列実行していた`AVAudioSession`のcategory設定・activateを接続待ちと並行する先行準備へ移し、接続と音声セッションの両方がreadyになり次第録音を開始する。staging診断には通話開始押下を基準とするoutbox・残高・call作成・`audio_session_ready`・`ws_connected`・`recording_started`・`first_transcript`・`first_reply_text`・`first_audio_started`の経過msをOS共通で追加した。usage、合図音、outbox回復、精算・課金規則は変更しない。）

（v5.32: 修正前の実機に永続化された通話usage outboxを、起動時とユーザー発信・疑似着信応答の開始前に本人所有callのDB状態と照合する自己回復を追加した。精算済み・不存在・所有不整合はローカル残留を除去し、通信失敗は保持、24時間超の`pending`はサーバ精算を再試行し、未解決は無条件削除せず既存の日次異常call回収へ委ねる。B3の二重消費防止のため着信応答も共通精算guardを維持し、staging診断にoutbox件数・状態・作成時刻を表示する。あわせて全画面の上部ヘッダーを表示専用にし、チャットの戻る・通話開始、通話中の出力切替・赤い終了ボタンを下部操作エリアへ移した。）

（v5.31: iOS WebViewのsafe area取得が0となる場合にもノッチ・ステータスバー内へ上部操作が入らないよう、`viewport-fit=cover`を維持したままiOS専用の上端最小退避と全画面共通insetを追加した。AI音声再生後はiOS/Androidとも、低音量の録音準備完了合図を課金音声キュー外で1回鳴らし、合図音の再生・解放完了後だけマイク録音を開始してSTTへの回り込みを防ぐ。）

（v5.30: iOS疑似着信の拒否とstandby準備完了・ユーザー発信開始の競合を解消し、遅着prepareの再破棄、破棄完了待ち、2秒タイムアウト、native側の無条件状態初期化により、拒否後の発信が誤った前通話処理中状態へ入らないようにした。）

（v5.29: Xcode 26 SDKの`UIRequiresFullScreen`廃止に伴い、Info.plistの全4方向宣言と実行時コードによる縦固定へ移行し、ビルド実行条件の運用ルール3項を追加した。）

（v5.28: App Store ConnectのITMS-90474再発防止として、iOS AppターゲットのDebug/ReleaseをiPhone専用・portrait専用・`UIRequiresFullScreen=true`へ一本化し、staging/productionのIPA作成後に埋め込みInfo.plistの`UIDeviceFamily=[1]`とportraitのみを提出前検査する。）

（v5.27: iOS版A8-2として、既存のアプリ内疑似着信画面をiOSでも有効化し、着信中にcall未作成・録音未開始・usage/課金未開始の`standby=1` WebSocketを準備する。応答後は既存の最新残高、owner付きpending、session、call作成、マイク権限の共通guardを通過した場合だけ同じsocketをactiveへ昇格し、拒否・画面離脱・開始拒否では準備接続を破棄してDBへ痕跡を残さない。iOS nativeの`IOSCallSignalPlugin`は外部音源やTTSを使わずPCM合成した着信音・コール音を`AVAudioPlayer`で再生し、応答・拒否・接続ready・離脱時にtimerとplayerを停止する。`.ambient` audio sessionにより着信音・コール音はiOSのサイレントスイッチへ従い、着信バイブは`AudioServicesPlaySystemSound(kSystemSoundID_Vibrate)`へ委譲する。iOSにはサイレントスイッチ状態や「バイブのみ」と「完全サイレント」を判別する公開APIがないため、音は確実にswitchへ従わせ、バイブの実動作は端末のシステム設定に従う範囲とする。ユーザー発信・着信応答とも既存TypeScriptのusage、owner outbox、ログ、一括精算、残高判定を共有し、iOS専用の課金・所有者・精算規則は追加しない。固定第一声は`fixed_voice_assets`、本番音源差し替えはF6、イベント発生・通知起動は別工程のままとする。DB migration、Supabase、Edge Function、productionは変更しない。）

（v5.26: PR #61のiOS疑似電話実機ビルド前検証として、`voice-turn`のactive WebSocket接続失敗時にAndroidと同じ最大1回の自動再接続を追加した。失敗turnのusageを確定し、録音・再生・active/standby接続を解放してから500ms待ち、新しいturnで接続し直す。自動再接続も失敗した場合だけ従来の`recoverable=true`を通知し、「もう一度試す」による手動再接続、usage finalized、課金・精算規則は変更しない。iPhone/iPadの対応方向をportraitだけへ限定し、Capacitor view controllerも回転を拒否する。iOS native通話開始時に`UIDevice.isProximityMonitoringEnabled`を有効化し、手動終了、残高終了、fatal、開始状態リセット、WebView離脱から合流する`stopResources`とplugin解放時に必ず無効化する。DB migration、Supabase、Edge Function、productionは変更しない。）

（v5.25: specをv4.6へ更新し、iOS疑似電話の第1段階としてユーザー発信通話を実装した。iOS 15以上の標準APIだけで完結する`AVAudioEngine`を採用し、端末の入力形式から`AVAudioConverter`で24kHz / mono / PCM16へ変換して既存`voice-turn`へbinary送信する。外部音声SDKやWebView `getUserMedia()`を使わず、`AVAudioSession`の`playAndRecord` / `voiceChat`に録音とAI音声再生を集約するため、Androidと同じWebView通話画面を維持しながら端末音声処理をnativeへ閉じられる。1ターン1 WebSocket、再生中の次turn standby、VAD、確定本文、短期履歴、実再生時間、終了barrierをiOS nativeで扱い、`call-protocol.json`から生成した`GeneratedCallProtocol.swift`の`turnCommitted` / `turnContentFinalized` / `usageUpdated` / `usageComplete` / `fatalError`をAndroidと同じpayloadで通知する。残高判定、owner付き永続outbox、usage/log RPC、一括精算、残高不足終了画面は既存TypeScript層を共有し、iOS専用の課金・所有者・精算規則は追加しない。通話画面も既存WebView UIを共有する。iOSの着信画面、着信事前接続、コール音、AlarmKit、CallKitは範囲外とし、ユーザー発信だけを有効にする。`NSMicrophoneUsageDescription`とXcode projectへのnative plugin登録を追加した。DB migration、Supabase、Edge Function、productionは変更しない。iOS実機ビルド・通話確認はMac/Xcode工程へ残す。）

（v5.24: specをv4.5へ更新してA8-2を新設し、その先行範囲としてアプリ内疑似着信の事前接続、ユーザー発信コール音、マナーモード追従を実装した。着信画面表示後は認証sessionと対象キャラだけでAndroid nativeの`standby=1` WebSocketを準備し、録音、usage、課金、`calls`作成は行わない。応答後に既存の最新残高・owner付きpending・session・call作成・マイク権限確認を通過した場合、同じstandby socketをactiveへ昇格して録音を開始する。準備中または準備済みsocketが失敗・終了していれば既存の通常接続へfallbackする。拒否・画面離脱・開始拒否時は準備socketを閉じ、DB書込を残さない。ユーザー発信では開始判定通過後からnativeのSTT ready通知までコール音を鳴らし、接続確立・失敗・終了で停止する。Androidの着信音・コール音は`AudioManager.getRingerMode()`に従い、通常はring stream合成音+バイブ、バイブモードはバイブのみ、サイレントは無音とする。固定第一声は`fixed_voice_assets`工程、本番音源差し替えはF6へ残し、今回は実装しない。DB migration、Supabase、Edge Function、productionは変更しない。）

（v5.23: PR #59のAndroid staging実機確認で、疑似着信画面・拒否・応答はPASSした一方、Web Audio合成着信音が無音だった。従来実装はAndroid WebView上で`AudioContext.resume()`の完了を待たずにoscillatorを予約しており、ユーザー操作の有効範囲を外れてcontextが`suspended`のままになると音声時刻が進まない。AndroidではWebViewの自動再生制約に依存しないCapacitor native pluginへ切り替え、`ToneGenerator`のring streamで二音を2.2秒間隔で反復する。応答・拒否・画面離脱・plugin破棄時はcallbackを除去してtoneを停止・解放し、多重再生を防ぐ。アプリ内foreground着信のためForeground Service、通知権限、外部音源素材は追加せず、browserは既存Web Audioをfallbackとして維持する。DB migration、Supabase、Edge Function、productionは変更しない。）

（v5.22: spec A8の特別イベント用アプリ内疑似着信画面を実装した。保有キャラの表示名と既存のキャラ画像表現、着信状態、応答・拒否ボタンを全画面表示する。応答時は`source=special_event`を付け、ユーザー発信と同じ開始関数へ合流して最新DB残高、owner付きpending/settlement、セッション、対象キャラ、Android native開始結果を同じ順序で検証する。拒否時はcallやDB行を作らず元のチャット画面へ戻る。A8でForeground Service音が指定されるのはモーニングであり、特別イベントはユーザーが疑似LINEで事前にOKした後のforegroundアプリ内画面なので、今回は外部音源・権限・バックグラウンド動作を増やさない小音量のWeb Audio合成着信音を採用した。応答、拒否、画面離脱時にtimer・音源・AudioContextを停止し、多重再生を防ぐ。イベント判定と通知起動は範囲外のため、開発用トリガーはdevまたはstaging接続時だけ、選択済みキャラのチャット本文上部に表示し、本番接続ではDOMへ出さない。現キャラデータの`visual_asset_id`はTODOで実画像素材がないため、既存画面と同じキャラ別avatar表現を画像枠へ表示し、本番素材差し替えはデザイン工程に残す。DB migration、Edge Function、productionは変更しない。）

（v5.21: spec E2/D15の通話ログ保存仕様を完了した。従来は`callTurnCommitted`がAI音声の全再生完了後にだけ本文を保存していたため、確定応答後から再生完了までの異常終了で日次処理用の本文を失う差分があった。AndroidはLLM/TTS正常完了の`done`受信時に確定user transcriptとassistant textを専用`callTurnContentFinalized`イベントへ出し、短期履歴の再生完了条件とは分離する。owner付き永続outboxはcontent revisionで重複・古い更新を除外し、通常AI音声の実再生開始が後着した場合も同じturnを新revisionで補完する。forward migration `20260716210000_complete_call_log_storage.sql`は、`content_revision`・`content_finalized`を追加し、`current_app_user_id()`による`public.users.id`所有者確認、call行lock、固定`search_path`、authenticated限定の拡張`upsert_call_turn_content`で、本文、VAD終了理由、応答生成時刻、通常AI音声開始時刻をusageと同じ`call_logs`行へ冪等保存する。現STTプロトコルにconfidence値はないため推測せずNULLとし、音声ファイル保存はspecどおり任意のため未追加とする。旧4引数RPCは既存アプリ互換として維持する。`should_delete_after_daily_processing`等、将来の`call_summaries`生成・本文削除に必要な既存列を確認したが、日次生成・削除処理そのものは別工程とする。migrationは作成のみでstaging未適用、本番Supabase・本番Edge Functionは未変更。）

（v5.20: PR #57のmigration `20260716190000_add_abnormal_call_daily_recovery.sql`をstagingへ適用して回収を確認したところ、usage計測migration導入前のlegacy call 12件が、本文用`call_logs`を持つ一方で全行`usage_revision=0`・`usage_finalized=false`のため、`skipped_usage_incomplete`を繰り返すことを確認した。forward migration `20260716200000_recover_legacy_unmetered_calls.sql`を追加し、`created_at < 2026-07-14 15:00:00 UTC`、`settlement_status in (unrecorded, recording)`、`billable_duration_ms=0`、ownerあり、全`call_logs.usage_revision=0`をすべて満たすcallだけを一度限り`cancelled`・精算完了へ移す。保存されていない課金時間は推測せず、残高、`coin_transactions`、`coin_consumptions`を変更しない。境界時刻以後、pending/settled/failed、1ms以上、ownerなし、revision 1以上を1件でも持つcallは対象外とする。legacy forward migrationは作成のみでstaging未適用、本番Supabase・本番Edge Functionは未変更。）

（v5.19: 疑似電話の異常終了call日次回収を実装した。migration `20260716190000_add_abnormal_call_daily_recovery.sql`で、最終活動から24時間以上経過したcallを毎日03:30 JST（18:30 UTC）に最大100件、古い順で`FOR UPDATE SKIP LOCKED`により回収する。24時間は通常通話の最大時間ではなく、遅延outboxや長時間通話を誤回収しないための独立した無活動猶予である。03:30 JSTは利用が比較的少ない時間帯、100件は1回のトランザクションで保持するcall・残高行lockを限定しつつ日次で再試行できる件数として採用した。対象抽出用partial indexを追加した。`active`・`unrecorded`・0ms・`call_logs=0`の空callはowner付きcall行をlockして`cancelled`へ回収し、0コイン・台帳なしで精算完了状態にする。終了outboxがないactive callと既存pending callは、全`call_logs`にrevision付きfinalized usageが保存済みで、既存expected件数とも矛盾しない場合だけ、保存値から合計時間とbarrierを復元する。content-only、未finalized、件数不一致、24時間以内の活動は推測精算せず次回へ残す。authenticated用`settle_call_coins`は`current_app_user_id()`を維持し、service-only回収と同じowner照合・call lock・残高lock・idempotency keyを持つ精算coreへ委譲するため、二重減算・二重transaction・二重consumptionを防ぐ。回収RPCはauthenticated/anonへ公開せず、遅着usageはterminal call用triggerで拒否し、content再送は継続できる。migrationは作成のみでstaging未適用、本番Supabase・本番Edge Functionは未変更。）

（v5.18: PR #56のAAB `staging.1.0.24`で、owner付きpending callによる終了直後の再通話ブロックを実機PASSした。一方、開始拒否案内の専用モーダルを`renderChat()`の`</main>`後へ置き、上部配置を`position:fixed`だけに依存していたため、Android WebView実機では通常フローの画面末尾に表示された。該当文言を設定する経路が`callStartNotice`だけであることを確認し、専用モーダルのDOMを通話ボタンと同じ`chat-header`内部へ移した。CSSが適用される場合はボタン直下へabsolute配置し、固定配置に依存せずDOM構造上もチャット上部に残る。pending/settlement開始guard、仮料金、DB、migration、Edge Functionは変更せず、本番Supabase・本番Edge Functionも未変更。）

（v5.17: PR #56のAAB `staging.1.0.23`実機再確認で、残高不足の開始拒否ロジック自体は動作する一方、案内が通話ボタンから見えない位置に出る問題と、残高上限終了直後にcompletion到達前の再開始が通る競合を確認した。終了処理の`markCallPending()`後、`callUsageComplete`到達前はowner付きpending outboxの`expectedTurnCount`が未確定でsettlement itemもまだ存在しないため、settlementだけを見る開始guardを通過していた。開始判定をowner所有pending call（expected未確定を含む）またはsettlementのいずれかが残る間は拒否する方式へ変更し、既存completion・settlement・一括精算フローは維持した。残高不足と前通話処理中の案内は通話ボタン付近へ固定した専用モーダルで表示する。開始判定ではDB残高、pending有無、settlement有無、判定結果を`call_balance_check`の`voiceDebug`へ記録する。DB、migration、Edge Function、仮料金、`settle_call_coins`は変更せず、本番Supabase・本番Edge Functionも未変更。）

（v5.16: PR #56の実機確認で判明した通話再開始の2件を修正した。通話開始時は起動時・画面上の残高キャッシュを使わず、`current_app_user_id()`で既に解決済みの本人`public.users.id`を条件に`coin_balances`の最新行をDBから毎回取得し、call作成前に判定する。残高を使い切った通話の精算が永続outboxに残る間はDB反映前の残高で次のcallを作らず、owner単位のpending settlementとして開始を止める。残高不足、精算待ち、開始context不足、残高取得失敗の各経路を開始中guardの確実な解除と画面案内へ集約し、再起動後を含む無反応経路をなくした。既存`settle_call_coins`、仮料金、DB、migration、Edge Functionは変更せず、本番Supabase・本番Edge Functionも未変更。）

（v5.15: 疑似電話の残高不足事前予告と終了画面を仮仕様で実装した。既存`settle_call_coins`と同じ仮料金（60秒1コイン・端数切り上げ・最低2コイン）から開始時残高で継続可能時間を求め、native usageのrevision別最新値を合算するOS共通`CallBalanceGuard`を追加した。ユーザーターンと通常AI音声再生中だけ残時間を進め、STT/LLM/TTS生成待ち、接続待ち、固定聞き返し中は進めない。Androidには同じ最大課金対象時間を渡し、録音・通常AI音声の各区間だけnative停止期限を設定することでWebView bridge遅延による課金境界超過を防ぐ。残り60秒未満で通話画面へ仮の予告を表示し、0で既存の終了・finalized barrier・一括精算経路へ一度だけ合流して、残高不足専用の仮終了画面を表示する。2コイン未満では新規call作成前に開始を止める。文言・閾値・正式料金はF1/F5で後決めとし、音声での残高不足案内は固定音声素材整備後にF5で決める残作業とする。精算規則・DB・migration・Edge Functionは変更せず、本番Supabase・本番Edge Functionも未変更。）

（v5.14: 疑似LINE履歴のピン留めを実装した。本文メッセージと通話見出しの各行にピン切替を追加し、`security definer`かつ固定`search_path`の`set_chat_message_pinned` RPCが`current_app_user_id()`で解決した`public.users.id`所有者だけの`chat_messages.pinned`を更新する。通話見出しでは同じトランザクション内で対応する本人所有`calls.pinned`も同期し、90日保持対象の両側を一致させる。`chat_messages`の広い直接UPDATE権限は戻さない。PR #55はmainへマージ済み（merge commit: `4ce8ca24b13e17cd751f6296acda203c4bb33acf`）。migration `20260716100000_add_chat_history_pinning.sql`はstaging適用済みで、Android staging AAB workflow run `29428740970`は成功した。本番Supabase・本番Edge Functionは未変更。本人確認、行lock、権限限定、通話見出し同期、UIの失敗時復元を自動テストでPASSした。）

（v5.13: 正本spec A7の未実装だった疑似電話後の通話見出し差し込みを実装した。終了済み本人callだけを対象に、security-definer RPC `create_call_summary_header`がcall行lock下で`message_type=call_summary_header`の固定見出しを1件だけ作成し、`calls.chat_header_message_id`へ紐づける。逐語ログは疑似LINEへ表示しない。owner単位の永続outboxでpending保存後に再送し、content保存・コイン精算とは独立させる。開始失敗callでは見出しを作らない。PR #54はmainへマージ済み（merge commit: `79809d36b94e43f5be46c14006d4685308de7425`）。migration `20260715210000_add_call_summary_header.sql`はstaging適用済みで、Android staging AAB workflow run `29426984799`は成功した。本番Supabase・本番Edge Functionは未変更。冪等作成、owner分離、オフライン復元、content失敗非依存、固定表示を自動テストでPASSした。）

（v5.12: PR #52をmainへマージ（merge commit: `cc01a9cca399f60b666e040f7417f3b554148906`）。通話コイン一括精算、authenticatedクライアントとsecurity-definer RPCの権限分離、全turnのusage確定後だけ精算するfinalized barrier、AndroidのTurnUsage作成と終了処理の競合修正を完了した。staging integrationでは料金境界、冪等精算、残高不足、所有者・管理列権限、並列精算、finalized usage遅着、content失敗非依存を確認し、Android終了競合はJava実スレッドテストで初期化中終了、作成直前終了、0 turn、fatal/manual同時終了、通常・再接続turnをPASSした。staging migrationは適用済み。本番Supabase・本番Edge Functionは未変更。残作業は正式料金、残高不足の事前予告・終了画面、古い空active callと既存pending callの日次回収であり、本番migration反映はリリース直前に行う。）

（v5.11: PR #51をmainへマージ（merge commit: `3a4ed1b352ce5db31695eea6ff2a05f910f88294`）。通話コイン一括精算の仮料金として、`calls.billable_duration_ms`を60秒ごとに1コインへ切り上げ、1ms以上は最低2コイン、最大消費なしで計算する`settle_call_coins` RPCを追加した。`current_app_user_id()`で解決した`public.users.id`とcall所有者を照合し、`pending`だけをcall行・残高行lock下の1トランザクションで精算する。成功時は残高、`coin_transactions`、`coin_consumptions`、call状態を同時更新し、call単位のidempotency keyと`consumed_coin_id`で再実行時の二重減算・二重台帳を防ぐ。残高不足は残高を変えず`skipped/insufficient_balance`を1件記録し、0msは台帳を作らず空call回収対象の`pending`に残す。Solレビューblocker対応としてforward migration `20260715190000_harden_call_settlement_barrier.sql`を追加し、authenticatedの`calls`直接更新を`pinned`だけへ限定、作成列も`user_id`・`character_id`・`source`だけへ限定した。Androidは通話終了時に期待turn数とfinalized turn数を明示通知し、所有者付き永続outboxは全usage finalizedとpending保存の成功後だけ精算する。DBもcall行lock下で期待turn数とfinalized `call_logs`数を照合し、精算後の課金時間変更を拒否する。content再送失敗は精算を止めない。再レビューで見つかったAudioRecord初期化と終了barrierの競合は、TurnUsage作成と終了時finalize・completion snapshotを同じ`TurnUsageBarrier` monitorで直列化して修正した。終了開始後の新規usageを拒否し、初期化中終了、作成直前終了、0 turn、fatal/manual同時終了、通常・再接続turnを実スレッド自動テストでPASSした。両migrationをstagingへ適用し、0ms、59秒、60秒、61秒、120秒、121秒、他ユーザー拒否、管理4列の直接更新拒否、同一call 4並列精算、残高2コインでの別call同時精算、pending後のfinalized usage遅着、精算後usage更新拒否、content失敗非依存を一時データでPASSし、テストデータは削除した。正式料金、残高不足の事前予告、古い空active callと既存pending callの日次回収は未実装。本番Supabase・本番Edge Functionは未変更。）

（v5.10: PR #51のTTS失敗安全処理をstaging `voice-turn` version 12へdeployし、ACTIVEかつ起動時のimport・bundle・runtime errorなしを確認した。Android staging AAB workflow run `29394935531`でversionCode 1019を作成し、実機で正常に通話でき、エラー表示がないことをPASSとして確認した。これによりPR #51の実機確認は完了とする。本番Supabase・本番Edge Functionは未変更。）

（v5.9: PR #50をmainへマージ（merge commit: `9acd551ab41bbe530ce72aa906e64a8c5a5e17d7`）。AAB 1018で分離したTTS失敗処理として、文単位の並列TTS Promiseへ作成直後からreject handlerを付け、最初に観測した失敗だけを採用して`tts_failed`を1回だけ通知する。失敗後はLLM/TTSを中止し、未送信audioと`done`を送らずWebSocketを1011で明示終了する。Aivisの実HTTP statusをエラーpayloadの`http_status`へ入れ、AndroidはWebSocket upgradeのHTTP 101と混同せず表示する。Androidのstage、errorCode、HTTP status、close code、exception classはactive turn開始ごとに初期化し、終了済み前turnのclose callbackで上書きしない。Aivis実APIは呼ばずHTTP 402モックで、先頭・途中・複数同時失敗、通知一回、失敗後audioなし、completedなし、未処理rejectなし、402伝播、前turnの101/1000残留なしを自動テストした。未再生AI音声0ms、再生済み音声は実再生分のみ、通話終了後`settlement_status=pending`、コイン非消費というPR #50のusage規則は変更しない。本番Supabase・本番Edge Functionは未変更で、staging deployと実機確認は未実施。）

（v5.8: PR #50のAndroid staging AAB 1018を実機確認し、新規の空active callなし、0msのcompleted callなし、数秒以内の重複callなし、開始中の二重押し防止をPASSとした。対象call末尾`4acf`は`completed`、22,497ms、`settlement_status=pending`、3 turnであり、PR #50の通話usage記録・空call防止の対象範囲は実機確認済みとする。通話失敗の原因はAivis TTSのHTTP 402であり、並列TTS Promiseの未処理rejectionが発生した。Android診断画面のHTTP 101とclose 1000は失敗turnの値ではなく前turnの残留値だった。TTS Promise rejectionの安全処理、`tts_failed`の一回化、Aivis実HTTP statusの通知、WebSocket明示終了、Android診断値のturn単位初期化、HTTP 402モックテストは別PRへ分離する。時間からコインへの換算・減算と古い空active callの日次回収は未実装のままとし、疑似電話全体と通話コイン消費全体は進行中`[~]`を維持する。）

（v5.7: PR #50のAndroid staging AAB 1017を実機確認し、数ターンの通常会話、固定聞き返し音声、聞き返し後の通常会話継続、画面エラーなしをPASSとした。対象call末尾`8da8`は9 turnすべて`usage_finalized=true`で、ユーザーターン合計45,703ms、通常AI音声7 turn合計75,406ms、`billable_duration_ms=121,109`、`settlement_status=pending`となり、固定聞き返し2 turnのAI再生時間は0ms、コイン残高不変、`coin_transactions`・`coin_consumptions`増加0をstagingで確認した。同時に、TypeScriptがnative開始結果の`ok`・`started`を確認せず、開始不成立callを成功扱いして空callを残す問題を特定した。`ok=true`かつ`started=true`だけを成功とし、false・欠損・不正payload・rejectは同じ失敗経路で、作成時に固定した`public.users.id`所有者のpending outboxへ積んで0ms・終了済みとし、通話画面へ遷移せず再試行可能なチャット画面へ戻す。開始中はDB作成前の再入guardとボタン無効化を行う。強制終了・OS終了・プロセス終了により終了処理自体が実行されず残る古い空のactive callの日次回収は今回実装せず、日次処理工程の残作業とする。時間からコインへの換算・減算等は未実装で、通話コイン消費全体と疑似電話全体は進行中のままとする。）

（v5.6: 時間制通話コイン消費の第1段階として、Androidでユーザーターン経過時間とAI通常回答音声の実再生時間を計測し、OS共通イベント、revision付き保存outbox、`call_logs`同一行upsert、通話終了時の未精算`pending`管理を実装した。migration `20260714150000_add_call_usage_recording.sql` をstagingへ適用し、同一turn再送の二重行防止、通話合計時間、固定音声0ms、本文と計測の同一行統合、残高不変、`coin_transactions`・`coin_consumptions`非作成を確認した。PR #50レビューで、outboxを`public.users.id`所有者単位へ分離し、ID確定前・別ユーザーでは送信せず同じIDへの引き継ぎ後だけ再送するよう修正した。localStorageの読込・書込・容量超過・破損JSONを通話から隔離し、復元payload全体の検証と不正項目破棄、Android再生終了callbackの競合guardを追加した。stagingでcontent/usage同時実行、複数turn同時実行、pending先行、owner付きoutbox再送を再確認した。時間からコインへの換算、減算、最低消費、端数処理、最大消費、残高不足制御、終了画面表示は未実装で、F1の正式数値は未決、通話コイン消費全体と疑似電話全体は進行中のままとする。）

（v5.5: チャットと通話のコイン計算方式を分離したspec v4.4へ対応。チャットはAI返信1回ごとの固定消費、通話はユーザーターンの経過時間とAI通常回答音声の実再生時間をターンごとに記録し、AI処理待ち・通信待ちを除外して通話終了時に1通話分を一括精算する時間制とする。正式な時間単位・消費量・最低消費・最大消費・端数処理は未決であり、通話コイン消費は未実装・疑似電話全体は進行中のままとする。）

（v5.4: PR #48のstandby昇格修正後をstaging `voice-turn` version 11 / Android AAB 1016で実機確認した。同一通話内で、1回目の無音は1回目の固定聞き返し音声と録音再開、2回目の無音は異なる固定聞き返し音声と「聞き取り中」で停止しない録音再開、3回目の無音は3回目の案内音声と「もう一度試す」「通話を終了」の表示、「もう一度試す」後は録音再開と通常発話へのAI通常応答復帰をすべてPASSとして確認した。standby昇格後の2回目無音停止は解消し、PR #48の対象範囲は実機確認完了とする。通話コイン消費は未実装でPR #48には含めない。あわせて、通話終了後に疑似LINEチャット画面へ通話見出しが表示されないことを確認した。正本spec A7の「通話見出しを差し込み、逐語ログは表示しない」仕様に対する既知の未実装として疑似電話の残作業へ記録し、今回は実装しない。）

（v5.3: PR #48のAndroid staging AAB 1016実機確認で、1回目の無音は固定聞き返し音声と録音再開まで成功したが、再度無音にすると「聞き取り中」で停止した。stagingログから、Androidがstandby WebSocketをactiveへ昇格してもFunction側が接続開始時の`standby=true`を固定保持し、2回目の`empty_completed`を`standby_stt_failed`へ誤分岐させることを原因として特定した。Function側のstandby状態を可変にし、昇格後の`context`受信時にactive扱いへ変更する最小修正と自動テストを追加した。Androidの録音・再接続・タイマー・画面処理、通話コイン処理は変更しない。修正後のstaging実機確認は未実施。）

（v5.2: PR #48のstaging Android実機確認で、短期記憶、近接センサーによる画面消灯中の会話継続、回答速度に大きな悪化がないことを確認し、いずれもPASSと記録した。一方、通話途中でUIが「接続中」表示となり停止する事象が再発した。画面回転、近接消灯からの復帰、一定時間のタイムアウトのいずれが原因かは未特定であり、画面復帰原因・タイムアウト原因のいずれとも断定しない。原因特定前にコード修正、タイマー変更、自動再接続は行わない。）

（v5.1: PR #48の第2段階として、Android疑似電話の短期履歴を共通`call-protocol.json`由来の生成定義へ接続した。成功・再生完了した現turnをAndroid nativeから`callTurnCommitted`でTypeScriptへ非同期通知し、共通`CallShortTermHistory`のスナップショットと照合する。ネイティブのstandby昇格・次ターン録音はTypeScript ACKを待たない。自動テスト、TypeScript検査、Web buildは通過したが、Android実機確認、Android/iOS共通化全体、iOS疑似電話、両OSの回答速度確認は未完了のままとする。）

（v5.0: 同一`call_id`内でのみ有効な短期会話履歴を、Androidクライアントのメモリに実装。確定したuser/assistant発話を時系列で保持し、最大10往復かつ本文合計12,000文字までを次ターンの`voice-turn`へ渡す。`gpt-5.4-nano`の応答上限は既存の200 tokenであり、キャラ・関係・ユーザー理解の固定system promptも毎ターン送るため、無制限送信を避けてこの上限とした。古い通常会話から削除し、通話終了・`call_id`変更・失敗ターンでは破棄する。standbyは昇格後、追加済みの最新履歴をactiveとして送信する。長期記憶、DB/migration、本番Supabase、本番Edge Functionは変更しない。）

（v4.9: 2026-07-13 PR #46 `feat: add user initiated voice call flow` をmainへマージ（merge commit: `9006dbfd31d53111897b872eaedefbf8b7eee087`）。ユーザー発信の疑似電話を実装し、1ターン1 WebSocket＋次ターン先行接続へ変更。staging実機で5分29.025秒・15ターン・途中エラーなし、150秒を超える継続通話を確認した。本番Edge Function反映はフェーズ4のリリース直前工程へ繰り越す。実機で会話品質上の文脈保持不足を確認したが、原因分析・対応方法・実装方針は未決であり、PR #46の完了範囲には含めない。）

（v4.8: 2026-07-13 開発中のSupabase運用を明確化。本番Supabaseへの追加migration、Edge Function deploy、secret変更、RLS実動作確認、`current_app_user_id()`本番確認は、機能PRごとには行わずリリース直前工程でまとめて実施する。開発中はstagingで検証し、本番未反映を理由に次の機能実装を停止しない。既に本番反映済みの指定migration 5件は変更しない。specの製品仕様は変更しないためv4.3据え置き。）

（2026-07-11 staging(voice-companion-staging)の Anonymous sign-ins を有効化。理由: RLS / `current_app_user_id()` 検証のため。本番挙動に合わせ有効のまま維持。）

（v4.7: 2026-07-12 PR #38、#39、#42、#43のmainマージ完了を記録。PR #38の疑似LINE会話コアはstaging実機確認済みで、本番Supabaseへの今回追加分の反映と本番実機確認は未実施。PR #39のstaging専用Android AAB workflowはproduction workflowを変更せず、内部テストへアップロードできる状態。PR #42のJWT時刻ずれ回復と日時自動設定案内は自動テスト済み・staging AAB確認済み。PR #43はv4.6への途中記録PRとしてmainへマージ済み。第2部のAuthと疑似LINEは残作業があるため進行中を維持。specの製品仕様は変更しないためv4.3据え置き。）

（v4.6: 2026-07-12 PR #42のJWT時刻ずれ回復に関する途中記録をPR #43でmainへ保存。staging AABで通常起動、メッセージ送信、AI返信を確認済み。実際の端末時刻ずれは安定して再現できないため、再現テストは実施しない。JWT時刻ずれ系エラー検出、自動refresh、再読込、失敗時の日時自動設定案内は自動テスト済み。マージ完了後の状態はv4.7を正とする。specの製品仕様は変更しないためv4.3据え置き。）

（v4.5: 2026-07-12 stagingでPR #38の疑似LINE会話コアを実機確認し、ユーザー発言保存、AI返信、成功時の1応答分コイン消費、失敗時の非消費を確認した事実を記録。`chat-reply` 用 `OPENAI_API_KEY` 設定済み、および `20260712080000_fix_complete_chat_reply_balance_ambiguity.sql` のstaging適用済みを記録。specの製品仕様は変更しないためv4.3据え置き。）

（v4.4: 2026-07-11 staging(voice-companion-staging)でRLS / `current_app_user_id()` を検証し両方PASSした事実を記録（本番での同確認は未完了として据え置き）。あわせてルール1-2を実態に合わせ書き換え: specとbuild planは常に同版数にはせず、片方だけ内容が変わった場合はその方だけ版数を上げ変更履歴で対応関係を示す方針とした。build planをv4.3→v4.4へ、specはv4.3据え置き。）

（v4.3: 複数機能をまとめて実装・確認する旧方式を廃止。利用者から見て一つの機能として成立する単位ごとに、Codexが実装、必要な修正、自動テスト、ビルド、commit、push、PR作成まで連続して進める運用へ変更。ユーザーは原則PRレビュー後のマージ、本番反映が必要な場合の操作、最終的な実機確認を行う。MDに答えがなく製品仕様が変わる判断、または認証・RLS・課金・本番データの破壊的変更に関わる安全確認だけ停止して報告する。）

（v4.2: specとbuild planの版数を同期。本番Supabaseへ指定済みmigration 5件を `supabase db push` 済みとして記録し、旧migration差分確認作業を完了事項へ移動。正本MDで決定済みの作業はCodexが実装・修正・テスト・commit・PR作成まで連続して進める運用、Supabase運用、モデル運用を追加。）

（v4.1: PR #29後の実機確認結果を記録。Supabase Dashboardで Anonymous sign-ins をONにしたことで `Anonymous sign-ins are disabled` エラーは解消し、実機で名前入力 → キャラ選択 → 初回関係選択 → トーク画面と思われる画面まで遷移できた。PR #29時点の初回オンボーディング導線は最低限成立。ただし完成版の本番デザイン確認ではなく、現状はフォーム中心の仮UI。本番デザイン作り込み、本格AIチャット、通話、通知、課金、引き継ぎ、退会は未完了。）

（v4.0: `docs/voice_companion_spec.md` v4.0 正本に合わせて作業計画を更新。v1本番導線から Google / Apple / email / OAuth / SocialLogin / ログイン画面を廃止済みとして整理し、Phase1 Auth は Supabase anonymous session 前提に固定。spec v4.0 正本化は完了済み。PR #25 migration は merge 済みで、local `supabase db reset` による適用確認も完了。PR #28「Rebuild production UI onboarding foundation」は GitHub 上で merge 済みとして、production UI土台・初回起動導線・テーマ切替・オンボーディング仮実装・migration `20260708120000_add_onboarding_profile_and_cats.sql` を反映済みとして記録する。PR #29では、前スレで決定したproduction onboarding UI / 画面構成 / 画面遷移 / テーマ仕様を `docs/voice_companion_spec.md` 正本へ反映し、PR #28/#29後の完了/未完了を本書に整理する。ただし「PR #28 merge」と「Phase1全体完了」は混同しない。）

（v3.0: Phase1 Auth の本番方針を Google / Apple OAuth から、ログイン画面なしの Supabase anonymous session 自動作成へ変更。`public.users.id` をアプリ内主キーとして維持し、引き継ぎは `transfer_codes` + Edge Function/RPC 前提にする。）

（v2.1: PR #13 / PR #16 merge により、Phase1 Supabase Auth の email/password デバッグ疎通確認を main に保存した。`auth.users.id → public.users.auth_user_id → public.users.id → settings / coin_balances` の読み込み確認、Supabase client singleton、env未設定時Error、auth callback内await回避、package-lock整合まで完了。本番導線とアプリ全体へのセッション連携は未完了だったが、v3.0以降は匿名サインインを本番方針とする。）

（v2.0: PR #9 merge により、Phase1 DB schema migration が main に保存された。DB土台は完了扱い。Phase1全体は未完了。次はアプリ側 Supabase Auth 実装。）

（v1.8: Androidモーニング導線のフェーズ0実機検証を合格に更新。full-screen intentによるロック画面上の疑似着信、音、応答後のロック解除なしAI通話接続中画面への遷移が成立。仕様書と本書の版数をv1.8で再同期。）

（v1.7: フェーズ0の声モデル確保について、AivisHub上で商用利用可能なACML 1.0モデルを確認し、v1仮採用6モデルを決定。声モデルのライセンス面はblockerではなくなった。）

このファイルは2つを扱う。
1. **運用ルール** — 仕様書（`voice_companion_spec.md`）と本書の管理・進捗・変更記録のルール。
2. **作業手順** — 何から着手するか。着手順と依存関係、各ステップの完了条件。

仕様書 = 何を作るか（完成形の定義）。本書 = どの順で作るか／どう記録するか。役割を混ぜない。

---

# 第1部. 運用ルール

## 1-1. ファイル構成
- `voice_companion_spec.md` … 仕様書（正本）。常に名前固定・上書き更新。参照先は常にこれ1つ。
- `voice_companion_build_plan.md` … 本書（作業手順＋運用ルール）。同じく固定名・上書き。
- スナップショット … 大きな節目（実装フェーズ移行など）でだけ、その時点のコピーを版付き別名で保存（例 `voice_companion_spec_v1.0.md`）。普段は作らない。

## 1-2. 版数（両ファイル共通）
- ファイル冒頭に「版数・最終更新日」を必ず持つ。
- **マイナー（0.1上げ）** = 項目の追加・修正。通常の更新。
- **メジャー（1.0上げ）** = 大きな方針転換、または実装フェーズ移行の節目。
- 仕様書と本書は、対応が分かるように管理する。両者を常に同じ版数にはしない。**片方だけ内容が変わった場合は、変わった方だけ版数を上げ、変更履歴で対応関係を示す。** 中身が変わっていないファイルの版数を、同期目的だけで上げない。現在の対象仕様は `docs/voice_companion_spec.md` v4.9 正本。

## 1-3. 要件が変わった時のルール
1. 仕様書の該当箇所を**直接書き換える**（正本なので最新が正）。
2. 仕様書冒頭の**変更履歴に1行残す**（例: `2026-07-05 A6-9 告白の発生条件を変更。理由:…`）。
3. 変更が他項目に**波及する場合は、そこも同時に直す**（整合を必ず取る。放置しない）。
4. 大きな方針転換は、該当セクションに一時的に `【変更あり】` を付け、レビュー後に外す。
5. 版数を上げ、最終更新日を更新。

## 1-4. 進捗管理
- 仕様側: 「確定（本文）」と「未決（Z）」で既に分離済み。数値・実測待ちはZに集約。
- 作業側（第2部）: チェックボックスで消化を追う。`[ ]`=未 / `[x]`=完了 / `[~]`=進行中 / `[!]`=保留。
- 「どこまで進んだか」は第2部のチェック状況で判断する。
- 開発中の機能完了判断は、mainへのマージとstagingで必要な検証が完了したかで行う。本番Supabase未反映だけを理由に、機能を未完了扱いにしたり次工程を停止したりしない。
- 本番Supabase反映と本番総合確認は、フェーズ4のリリース直前工程でまとめて管理する。

## 1-5. AI・外部に渡す時
- 必ず仕様書と本書をセットで前提にさせる。旧メモ・旧specは渡さない・参照させない。
- 一問一答を使うのは、未確定仕様、仕様変更、specとbuild planの矛盾、ユーザー判断が必要な選択、課金・権限・本番運用などの重大判断に限る。
- 正本MDで決定済みの内容は、Codexが利用者から見て一つの機能として成立する単位の実装、必要な修正、自動テスト、ビルド、commit、push、PR作成まで連続して進めてよい。途中で「続けてよいか」「テストするか」「commitするか」「PRを作るか」の確認は求めない。
- ユーザーが確認するのは、原則としてPRレビュー後のマージ、stagingで必要な最終実機確認、リリース直前の本番反映操作と本番総合確認とする。実機確認で問題が出た場合は、その機能の範囲内で修正して再確認する。
- 「ボタン1個」など過度に細かく分割せず、利用者が一つの機能として確認できる単位で区切る。
- 推測で仕様を補わない。仕様判断が必要になった場合は作業を停止して報告する。
- 用語を勝手に増やさない（過去に「システム案内」「接続中」等で混乱した経緯）。

## 1-6. ビルド検証運用
- ビルドを検証手段にしない。ビルド実行は「エラー文の要求」「現行Apple公式文書（その場で照合した一次情報）」「修正が最終成果物に入ることの機械検査」の3点が一致したときのみ許可する。一致しない間はビルドを起動しない。
- ipa埋め込みInfo.plistの提出前機械検査を恒久維持する。
- ビルド1回で検証する仮説は1個に限定する。

## 1-7. Supabase運用
- migration作成、Edge Function作成、必要なsecretの定義、RLS/API実装は、機能単位の作業内でCodexが進めてよい。
- 開発中のDB・Edge Function・RLS・API検証はstagingで行う。
- **機能PRをマージするたびに本番Supabaseへ反映しない。**
- **本番Supabaseへの追加migration、Edge Function deploy、secret変更、Dashboard設定変更、RLS実動作確認、`current_app_user_id()`本番確認は、フェーズ4のリリース直前工程でまとめて行う。**
- **本番Supabase未反映、本番RLS未確認、本番`current_app_user_id()`未確認を、開発中の次工程を止める理由にしない。**
- 本番Supabaseを変更できるのは、リリース直前工程でユーザーが明示的に実行を指示した場合だけとする。
- 本番データの破壊的変更は自動実行しない。
- migrationで管理できる変更をSupabase Dashboardで直接行わない。
- Anonymous sign-insなどDashboard設定変更は、実施内容を本書へ記録する。
- 既に本番反映済みの指定migration 5件は完了済みの過去実績として扱い、この運用変更によって戻したり再適用したりしない。

## 検証環境 (staging)
- staging プロジェクト: voice-companion-staging (ref: <staging-ref>, Seoul)
  - 無料プロジェクト。追加料金なし。7日無操作で自動停止(使う時に手動再開)
  - 用途: スキーマ変更・migration・Edge Function・RLS・API・アプリ接続を本番前に検証する環境
- 本番プロジェクト: voice-companion (ref: <prod-ref>, Tokyo)
- 2026-07-11: staging の Anonymous sign-ins を有効化した。理由: RLS / `current_app_user_id()` 検証のため。本番挙動に合わせ、有効のまま維持する。

### 開発中の検証フロー
1. staging に link 切り替え: ! supabase link --project-ref <staging-ref>
2. db push前にlink先がstagingであることを確認する
3. `supabase db push` でmigrationをstagingへ適用する
4. 必要なEdge Function、secret、RLS、RPC、アプリ接続をstagingで検証する
5. 問題がなければ機能PRをレビュー・マージし、次の機能へ進む
6. 開発中は本番へlinkを戻して追加反映しない

### リリース直前の本番反映フロー
1. mainに保存された未反映migration、Edge Function、必要secret、Dashboard設定を一覧化する
2. stagingで対象一式が通っていることを確認する
3. ユーザーの明示指示後に本番へlinkを切り替える
4. 本番へまとめて反映する
5. 本番RLS、`current_app_user_id()`、Edge Function、主要アプリ導線を総合確認する
6. 結果をフェーズ4へ記録する

### 厳守
- link 切り替えは必ずユーザーが ! 付きで実行(パスワード管理のため)
- db push前に必ずlink先がstagingか本番かを確認する
- 本番(<prod-ref>)へのpush・deploy・secret変更・Dashboard設定変更は、リリース直前工程で明示的な指示があるときのみ
- supabase startは使わない(ローカルDocker禁止)

## 1-8. モデル運用
- 通常実装はTerraを使う。
- 認証、RLS、課金、引き継ぎ、重大な設計判断、原因不明の問題、機能完了時レビューはSolを使う。
- 検索、ログ整理、文書整形などの軽作業はLunaを使う。

---

# 第2部. 作業手順

## 前提の考え方
主要な設計判断は仕様書で確定済み。ただし「着手前に潰す前提」があり、ここが崩れる場合は設計を見直す。アプリ完成まで完全放置で一気に作らず、利用者から見て一つの機能として成立する単位ごとに進める。各機能の中はCodexが実装、必要な修正、自動テスト、ビルド、commit、push、PR作成まで連続して進める。機能の実機確認で問題が見つかった場合は、その機能の範囲内で修正して再確認する。

---

## フェーズ0-前. 正本仕様の確定
- [x] `docs/voice_companion_spec.md` v4.4 を正本として扱う。
- [x] v1本番導線から Google / Apple / email/passwordログイン、ログイン画面、OAuth callback deep link、SocialLogin依存、Google client ID前提、Apple Sign In capability前提を削除済みとして扱う。
- [x] `RevenueCat app user id = auth.users.id` と `auth.users.id` をアプリ内ユーザー主キーにする設計は廃止済み。
- [x] email/password は過去のデバッグ疎通実績としてのみ扱い、本番実装タスクに戻さない。
- [~] v4.3仕様に沿った実装と実機検証は未完了として扱う。PR #28でproduction UI土台と初回起動導線の一部はmerge済み。PR #29後の実機確認で、初回オンボーディング導線は最低限成立していることを確認済み。ただしPhase1全体は未完了。本番RLS・本番Supabase総合確認はフェーズ4へ繰り越す。

## フェーズ0. 着手前に潰す前提（最優先・ここが崩れたら設計見直し）

- [x] **声モデルの確保（最優先）**
  完了条件: Aivisで商用可能（ACML または CC0。ACML-NC不可）かつ満足できる声を、必要数見つける。事前生成音声をアプリに保存・同梱してよいか、選定モデルの条件も確認。
  結果: AivisHub上でACML 1.0の候補を確認し、v1仮採用モデルを男女3ずつ決定。ライセンス面ではblockerではない。
  仮採用: 女性 = まお / コハク / 桜音。男性 = にせ / fumifumi / 猩々博士。
  注意: アプリ内キャラ名は声モデル名と分離してよい。実在人物系モデルは本人・公式アプリと誤認させない。
  TTS方式決定(2026-07-18): TTS従量課金では採算が成立しないため、疑似電話・通知のAI音声再生はクラウドTTS(Aivis)ではなくオンデバイスTTS(ONNX Runtime + Style-Bert-VITS2系AIVMX)を採用する。`poc/on-device-tts`のPoC実験は終了・採用決定済み。オンデバイスTTSは上記6キャラ全員が対象で、桜音1モデルに絞る前提にはしない。方式・量子化(TTS本体・DeBERTa BERTともに動的INT8)・配信(Cloudflare R2からキャラ選択時DL)が実際に成立するかの検証・テストには、桜音を基準モデルとして用いる。文末単位ストリーミング再生。桜音モデルは組み込み・再配布可を確認済み(条件: ライセンス文書同梱・非公式/本人無関係の表記)。詳細は`voice_companion_spec.md` A8-3 / H1を正とする。量子化・検証はローカル禁止でstaging AAB workflowのCI内で行う。
- [~] **実機検証: モーニング導線**
  完了条件: iOS AlarmKit（iOS 26）と Android full-screen intent が、画面OFF・ロック中・端末未操作でも期待通り起動し、目覚まし用途として音が鳴ることを実機で確認。
  結果: iOSはAlarmKit検証済み。Androidは2026-07-05の検証で、full-screen intentによるロック画面上の疑似着信表示、疑似着信音、応答後のロック解除なし「AI通話に接続中」画面への遷移が成立。
  残り: Android実機でネイティブ `AudioRecord` による 24kHz / mono / PCM16 取得、`USE_FULL_SCREEN_INTENT` 権限・ストア方針、full-screen intentのメーカー別動作差、Foreground Service音、通知タップフォールバック、応答後にMainActivityではなく専用Activity内でAI通話接続状態へ進む導線を本番AI接続込みで再確認する。
- [x] **実機検証: 疑似電話のTTFA・遅延**
  完了条件: STT→LLM→TTS の体感遅延が実運用で許容内（目安 ~2.74s）に収まることを確認。ここでコスト実測 → コイン単価（Z-1）の材料にする。
  結果: Android実機で STT → LLM → Aivis TTS → 音声再生まで成立。TTFAは約2秒前後（最良例: 1.98秒）で確認。音声二重再生は修正済み。無音判定5段階の方針は決定済み。
  詳細な疑似電話仕様・接続方針は `voice_companion_spec.md` A5 を正とする。

## フェーズ1. 依存しない土台（フェーズ0と並行で着手可）

- [~] Supabase Auth（認証）
  - PR #13 / PR #16 で email/password デバッグ疎通は main に保存済み。ただし本番導線では使わず、本番実装タスクにも戻さない。
  - Phase1 Authの本番方針は、ログイン画面なしの Supabase anonymous session 自動作成に固定する。
  - 初回起動時は既存セッションがなければ `signInAnonymously()` を実行し、`auth.users.id` から `public.users.id` を解決する。
  - `auth.users.id` は認証セッションIDであり、各ユーザー別テーブルの `user_id` として直接使わない。
  - Supabase local config は `enable_anonymous_sign_ins = true` にする。本番Supabase Dashboard側でも Anonymous sign-ins を有効化する。
  - ログインUI、Google / Apple / emailログイン、Google / Apple provider設定、Google client ID設定、Apple capability、OAuth callback deep link、OAuth実機検証、SocialLogin依存は廃止済みとして扱い、Phase1本番タスクから削除する。
  - 完了: PR #25「Align auth mapping to v4: use RPC-resolved app user id, add migration, and adjust client/session flow」は GitHub 上で merge 済み。`20260708090000_align_v4_auth_rls.sql` はユーザー側ローカルの `sudo supabase db reset` で適用成功済み。`WARN: no files matched pattern: supabase/seed.sql` は `seed.sql` が無いことによる警告であり、migration失敗ではない。
  - PR #28で完了: production UI起動時の匿名セッション自動作成/復元導線、`loadUserData()` / `current_app_user_id()` 経由の `public.users.id` 解決前提、名前未設定時の名前入力画面、キャラ選択画面、キャラ名 + 初回関係設定画面、`user_characters` / `character_relationships` 初期行作成処理、疑似LINEホームへの遷移、テーマ切替（localStorage保存）の土台を追加済み。
  - PR #29後の実機確認: Supabase Dashboardで Anonymous sign-ins をONにしたことで `Anonymous sign-ins are disabled` エラーは解消。実機で名前入力 → キャラ選択 → 初回関係選択 → トーク画面と思われる画面まで遷移でき、初回オンボーディング導線は最低限成立していることを確認済み。
  - 本番反映済み: `20260706090000_init_voice_companion_phase1_schema.sql`、`20260707140000_add_transfer_codes.sql`、`20260708090000_align_v4_auth_rls.sql`、`20260708120000_add_onboarding_profile_and_cats.sql`、`20260709000000_add_deletion_audit_and_schema_gaps.sql` は本番Supabaseへ `supabase db push` 済み。
  - 残り: 引き継ぎコード発行/入力、引き継ぎ用 RPC または Edge Function 設計と実装。本番での追加確認はフェーズ4へ繰り越す。
  - 完了: PR #42（mainマージ済み、merge commit: `c90d6e6ee111799af0cd1bf5c51dfb3ff4681ac3`）でJWT時刻ずれ回復を実装。JWT時刻ずれ系エラー時は保存済みrefresh tokenで一度だけ自動回復を試し、`signOut()`、session削除、新規匿名ユーザー作成は行わない。`auth.users.id` と `public.users.id` の同一性確認を維持し、自動回復失敗時は端末の日付と時刻の自動設定を案内する。通常のnetwork errorは従来の一般エラー表示を維持する。staging AABで通常起動、メッセージ送信、AI返信を確認済み。時刻ずれ検出、自動refresh、再読込、失敗時案内は自動テスト済み。実際の端末時刻ずれは安定して再現できないため、再現テストは実施しない。
- [~] DBテーブル構築 / v4.3 Auth整合確認（RLS・制約・インデックス）
  - 完了: migration作成・PR #25 merge・local `supabase db reset` による `20260706090000_init_voice_companion_phase1_schema.sql`、`20260707140000_add_transfer_codes.sql`、`20260708090000_align_v4_auth_rls.sql` の適用確認は完了。
  - ただし上記は「migration作成・merge・local db reset確認」の完了であり、「Phase1 DB/RLS全体完了」ではない。
  - PR #28で完了: `20260708120000_add_onboarding_profile_and_cats.sql` を追加し、`public.users` に `family_name` / `given_name` / `family_name_kana` / `given_name_kana` を追加、`public.users` の更新権限をオンボーディング用プロフィール項目に限定、production UI土台で使う仮キャラ8件（人間6人 + AI猫 + ランダム猫）をidempotentにseedする形に更新済み。
  - PR #28で完了: 初回名前入力、初回キャラ選択、初回好意度/第一印象選択、`user_characters` / `character_relationships` 初期行作成のクライアント実装を追加済み。
  - 本番反映済み: `20260706090000_init_voice_companion_phase1_schema.sql`、`20260707140000_add_transfer_codes.sql`、`20260708090000_align_v4_auth_rls.sql`、`20260708120000_add_onboarding_profile_and_cats.sql`、`20260709000000_add_deletion_audit_and_schema_gaps.sql` は本番Supabaseへ `supabase db push` 済み。
  - PR #29後の実機確認: 本番Supabase Dashboardで Anonymous sign-ins をONにし、`Anonymous sign-ins are disabled` エラーが解消したことを確認済み。実機で名前入力 → キャラ選択 → 初回関係選択 → トーク画面と思われる画面まで遷移できた。
  - stagingで確認済み（2026-07-11・PASS）: 匿名セッションから `current_app_user_id()` が本人の `public.users.id` を正しく返し（`user_auth_links` の `is_current=true` は1件のみ）、RLSにより他ユーザーの `settings` / `coin_balances` / `user_characters` / `chat_messages` 行が読めない・フィルタ指定でも取得できないことを確認（漏れなし）。`chat_messages` は本人 `public.users.id` でのinsert成功と、本人のみ読める・別ユーザーから読めないことも確認。anon key + 匿名JWTのみで実施（service_role不使用）、RLSが実際に効いた状態。テストユーザー/データは削除済み。なおこれはstagingのRLS/`current_app_user_id()`動作確認であり、migrationを`db push`して本番前検証する作業ではない（stagingスキーマは既存前提）。
  - リリース直前へ繰り越し: 本番(voice-companion)での同じRLS実動作・`current_app_user_id()` による本番 `public.users.id` 解決の確認。これは現在の次工程を止める未完了項目として扱わない。
  - 未完了: 引き継ぎコード発行/入力のRPCまたはEdge Function設計と実装。
  - RevenueCat連携は `app_user_id = public.users.id` 前提に揃える。`auth.users.id` をRevenueCat app user idにする設計は廃止済み。
  - したがって「DB土台、PR #25 migration、PR #28 migration/クライアント土台、本番Supabaseへの指定migration 5件の反映、staging RLS確認は完了済み」。今後の追加本番反映と本番総合確認はフェーズ4で行う。
- [~] アプリ骨組み（Capacitor + Vite + TS）／ナビゲーション
  - PR #28で完了: production UI土台、起動/準備中表示、疑似LINEホーム、下部ナビ、設定画面、利用規約仮ページ、プライバシーポリシー仮ページ、仮モーダル、2テーマ切替（DB保存なし・localStorage保存）を追加済み。
  - 未完了: 本格ナビゲーション設計、実機UI検証、通知/モーニング/通話/課金/引き継ぎ/退会など本番機能画面への接続。
- [~] 疑似LINEチャット画面（`chat_messages` の表示・送信。通知タップ後の入口メッセージ、通話見出しの差し込み枠も）
  - PR #28で完了: ホームから選択済みキャラの仮チャット画面へ遷移する土台を追加済み。ホームではメッセージ本文を表示せず、状態表示のみとする方針を反映済み。
  - 完了: PR #38（mainマージ済み、merge commit: `d0288bd86d3a49e3514ca49ed6bd9c42e719cacf`）で疑似LINE会話コアを実装し、staging実機確認済み（2026-07-12）: ユーザー発言の`chat_messages`保存、AI返信の生成・保存・時系列表示、返信成功時の1応答分コイン消費、AI返信失敗時のコイン非消費。stagingの`chat-reply`には`OPENAI_API_KEY`を設定済み。`20260712080000_fix_complete_chat_reply_balance_ambiguity.sql` はstaging適用済み。
  - 疑似LINE会話コアは開発中の確認完了扱い。本番Supabaseへの今回追加分の反映と本番実機確認はフェーズ4へ繰り越し、次工程を止めない。
  - 完了: PR #54（mainマージ済み、merge commit: `79809d36b94e43f5be46c14006d4685308de7425`）で、通話終了後に逐語ログを含まない固定の通話見出しを`chat_messages`へ冪等に1件作成し、`calls.chat_header_message_id`へ紐づける。owner単位の永続outboxでpending保存後に再送し、開始失敗callでは作成しない。migration `20260715210000_add_call_summary_header.sql`はstaging適用済み。Android staging AAB workflow run `29426984799`は成功した。
  - 実装済み・staging未適用: 本文メッセージと通話見出しをピン留めできるUI、および本人所有行だけを更新する`set_chat_message_pinned` RPCを追加した。通話見出しの切替は対応する`calls.pinned`も同一トランザクションで同期する。migration `20260716100000_add_chat_history_pinning.sql`は作成済み。
  - 未完了: 通知タップ後の入口メッセージ。
- [~] キャラ選択・オンボーディング（匿名セッション確立後に名前入力。氏名のみ必須。呼び方初期指定UIは作らない）
  - PR #28で完了: 名前入力、キャラ選択、人間6人 + AI猫 + ランダム猫の表示、ランダム猫の初回AIメイン選択不可表示、キャラ名自由入力、初回関係3択、初期行作成、完了後ホーム遷移を追加済み。
  - PR #29後の実機確認: 実機・本番Supabaseで、名前入力 → キャラ選択 → 初回関係選択 → トーク画面と思われる画面までの導線は最低限成立していることを確認済み。
  - 未完了: 本番キャラ名/画像/説明への差し替え、AI猫/ランダム猫の本格仕様、複数キャラ保有/追加、完成版の本番デザイン確認。
- [~] 引き継ぎコード発行/入力（`transfer_codes` テーブルのみ追加済み。発行/引き換え用 Edge Function / RPC は未実装で、UIからはまだ使えない。生コードは保存せず `code_hash` のみ保存する方針を維持）

フェーズ1補足:
- DB schema migration は main に保存済み。
- PR #25 migration の merge と local `supabase db reset` での適用確認は完了済み。
- DB土台とPR #25 migrationは保存・適用確認済み。PR #28でproduction UI土台、初回オンボーディング導線、テーマ切替、仮ホーム/仮チャット/設定ページ、オンボーディング用profile columns追加、仮キャラ8件seed migrationはmerge済み。
- 本番Supabaseへ、指定migration 5件（`20260706090000_init_voice_companion_phase1_schema.sql`、`20260707140000_add_transfer_codes.sql`、`20260708090000_align_v4_auth_rls.sql`、`20260708120000_add_onboarding_profile_and_cats.sql`、`20260709000000_add_deletion_audit_and_schema_gaps.sql`）を `supabase db push` 済み。
- Phase1全体は未完了。PR #29後に本番Supabase Dashboardで Anonymous sign-ins をONにし、`Anonymous sign-ins are disabled` エラー解消と、実機で名前入力 → キャラ選択 → 初回関係選択 → トーク画面と思われる画面までの遷移は確認済み。本格AIチャット/通話/通知/課金/引き継ぎ/退会は未完了。
- Supabase Auth は匿名セッション自動作成へ切り替える。ログイン画面は出さず、`loadUserData()` で `public.users.id` を解決する。PR #28でこの前提のクライアント導線は追加済みで、stagingのRLS・`current_app_user_id()`確認はPASS済み。
- 次作業は、既存の第2部の機能順序に従い、利用者から見て一つの機能として成立する単位で開始する。各機能についてCodexは実装、必要な修正、自動テスト、ビルド、commit、push、PR作成まで連続して進める。必要なDB・Edge Function・RLS確認はstagingで行い、PRレビュー・マージ後は本番へ反映せず次の機能へ進む。本番反映と本番総合確認はフェーズ4でまとめて行う。
- 引き継ぎコード発行/入力は、RPCまたはEdge Function設計と実装が未完了。
- DB詳細は `voice_companion_spec.md` C章を正とする。

PR #38/#39/#42/#43の整理（2026-07-12）:

- 完了: PR #38（merge commit: `d0288bd86d3a49e3514ca49ed6bd9c42e719cacf`）はmainへマージ済み。疑似LINE会話コアを実装し、staging実機でユーザー発言保存、AI返信生成・保存、成功時の1応答分コイン消費、AI返信失敗時の非消費を確認済み。stagingには`OPENAI_API_KEY`設定済みで、`20260712080000_fix_complete_chat_reply_balance_ambiguity.sql`はstaging適用済み。本番Supabaseへの今回追加分の反映と本番実機確認はフェーズ4へ繰り越し、現在の次工程を止めない。
- 完了: PR #39（merge commit: `9e0c38fa7a5870df1d30014b911f0c108b3cdb22`）はmainへマージ済み。staging専用Android AAB workflowを追加し、production workflowは変更していない。staging用AABを内部テストへアップロードできる状態であり、Supabase本番環境は変更していない。
- 完了: PR #42（merge commit: `c90d6e6ee111799af0cd1bf5c51dfb3ff4681ac3`）はmainへマージ済み。JWT時刻ずれ系エラー時は保存済みrefresh tokenで一度だけ自動回復を試し、`signOut()`、session削除、新規匿名ユーザー作成は行わない。`auth.users.id` と `public.users.id` の同一性確認を維持し、自動回復失敗時は端末の日付と時刻の自動設定を案内する。通常のnetwork errorは従来の一般エラー表示を維持する。staging AABで通常起動、メッセージ送信、AI返信を確認済み。時刻ずれ検出、自動refresh、再読込、失敗時案内は自動テスト済み。実際の端末時刻ずれは安定して再現できないため、再現テストは実施しない。staging AABのversionCode計算は`github.run_number + 1000`であり、workflow通算4回目は1004、次回は1005となる（毎回1004固定ではない）。
- 完了: PR #43（merge commit: `a245c385837acd6fef3b9c7953172c728ec6a304`）はmainへマージ済み。build plan v4.6への途中記録PRであり、PR #42マージ前の状態記録は今回v4.7で更新した。

PR #28/#29後の整理:

- 実施済み: PR #28はmerge済み。production UI土台、起動 / 準備中画面、初回名前入力画面、キャラ選択画面、キャラ名 + 初回関係設定画面、`user_characters` / `character_relationships` 初期行作成処理、疑似LINEホーム仮画面、疑似LINEチャット仮画面、キャラ管理仮画面、設定画面、テーマ切替、利用規約 / プライバシーポリシー仮ページを追加済み。
- 実施済み: migration `20260708120000_add_onboarding_profile_and_cats.sql` を追加済み。`public.users` に `family_name` / `given_name` / `family_name_kana` / `given_name_kana` を追加し、`public.characters` に仮キャラ8件をseedし、`public.users` の更新権限をオンボーディングプロフィール項目に限定する。
- PR #29で整理すること: 上記production onboarding UI / 画面構成 / 画面遷移 / テーマ仕様を `docs/voice_companion_spec.md` の正本仕様へ反映し、本書では「UI土台は入った」と「本番機能完成」を分けて記録する。
- 開発中の確認完了: 疑似LINEの`chat_messages`保存/表示、AI応答、コイン消費はPR #38で実装済み・staging実機確認済み。本番反映・本番実機確認はフェーズ4へ繰り越す。
- 未完了: モーニングコール本実装、通知送信、RevenueCat課金、引き継ぎコード発行 / 入力、引き継ぎ用 RPC または Edge Function、退会処理、通知タップ後の入口メッセージ。
- PR #29後の実機確認結果: Supabase Dashboardで Anonymous sign-ins をONにした。これにより `Anonymous sign-ins are disabled` エラーは解消。実機で名前入力 → キャラ選択 → 初回関係選択 → トーク画面と思われる画面まで遷移できたため、PR #29時点の初回オンボーディング導線は最低限成立している。
- 注意: 上記は完成版の本番デザイン確認ではない。現状はフォーム中心の仮UIであり、本番デザイン作り込み、本格AIチャット、通話、通知、課金、引き継ぎ、退会は未完了。
- 本番反映済み: 指定migration 5件は本番Supabaseへ `supabase db push` 済み。
- stagingで確認済み（2026-07-11・PASS）: RLS実動作と `current_app_user_id()` による `public.users.id` 解決を staging で検証し漏れなし（詳細はフェーズ1「DBテーブル構築 / v4.3 Auth整合確認」参照）。
- リリース直前へ繰り越し: 本番(voice-companion)でのRLS実動作・`current_app_user_id()` が本番で正しい `public.users.id` を返す確認、完成版の本番デザイン確認。これらは現在の次工程を止めない。

## 機能単位の自立型開発運用

第2部の既存の機能順序とチェック状態を維持し、利用者から見て一つの機能として確認できる単位で進める。開発中はmainへのマージとstagingで必要な検証が完了した時点で、その機能の開発確認を完了扱いにできる。本番Supabase反映・本番RLS確認・本番総合実機確認はフェーズ4へ繰り越し、それだけを理由に機能を進行中へ戻したり次工程を停止したりしない。Codexは、正本MDで決定済みの機能について途中の細かい承認を求めず、実装、必要な修正、自動テスト、ビルド、commit、push、PR作成まで連続して進める。

停止して報告するのは、MDに答えがなく勝手に決めると製品仕様が変わる場合、または認証、RLS、課金、本番データの破壊的変更など重大な安全確認が必要な場合だけとする。ただし、既にフェーズ4へ繰り越すと明記された本番反映・本番確認が未実施であること自体は停止理由にしない。

## フェーズ2. 前提確認後に決める数値（Z-1〜Z-3）

- [ ] コイン数値: 無料の毎日回復量 / 月額付与 / 各商品価格 / チャット1返信の固定消費 / 通話の時間単位 / 通話1単位の消費量 / 端数処理 / 最低消費 / 最大消費 等
- [ ] 疑似電話の安全上限: 最大STT受付秒数 / 無音打ち切り / 聞き取り失敗回数 等
- [ ] 告白到達の関係値水準 / 未応答の下げ幅 / 拒否打ち切り回数

## フェーズ3. 本格実装（依存順）

- [ ] 記憶パイプライン（D）: チャット/通話ログ保存 → 日次処理（夜間バッチ）→ 長期記憶抽出 → AIに渡す組み立て（同日短期文脈は既存から組み立て）
- [ ] 課金（B）: RevenueCat app user id は `public.users.id` を使う。RevenueCat購入確認 → Supabaseでコイン付与 → 消費（idempotency_key で二重消費防止）→ 残高不足時の挙動
- [~] 疑似電話（A5）
  - 完了: PR #46（mainマージ済み、merge commit: `9006dbfd31d53111897b872eaedefbf8b7eee087`）で、疑似LINEの通話ボタンからユーザー発信の疑似電話を開始し、Androidネイティブ `AudioRecord`（24kHz / mono / PCM16）→ STT → LLM → Aivis TTS → 音声再生を複数ターン継続できる通話画面を実装。
  - 完了: 1ターン1 WebSocket＋次ターン先行接続。AI音声再生中に次ターン接続を`ready`まで準備し、再生終了後に接続を昇格して録音を開始する。通話画面と`call_id`は通話終了まで維持し、接続ごとの`turn_id`で古い接続イベントを無視する。
  - [x] PR #48: 同じ`call_id`の通話内だけで、確定・再生完了したuser/assistant発話を短期履歴として保持し、次ターンのAIへ時系列で渡す。上限は暫定で最大10往復・本文合計12,000文字（各発話最大1,200文字）で、超過時は古い通常会話から削除する。現在のユーザー発話は履歴に重複させず別のuser messageとして渡す。失敗・未完了・古い`turn_id`の結果は履歴化せず、通話終了または次の`call_id`開始時に破棄する。共通プロトコル土台とAndroid native→TypeScriptの非同期同期は自動テスト済み。staging Android実機では、短期記憶、近接センサーで画面消灯中も会話継続、画面復帰後の会話継続、回答速度に大きな悪化なしをPASSとして確認した。Android staging AAB 1016で判明した、activeへ昇格済みのstandby WebSocketをFunction側が`standby=true`のまま扱い2回目の`empty_completed`を誤分岐させる問題は、staging `voice-turn` version 11で修正した。修正後の同一通話で、1回目・2回目の異なる固定聞き返し音声と各録音再開、2回目に「聞き取り中」で停止しないこと、3回目の案内音声と「もう一度試す」「通話を終了」の表示、手動再試行後の録音再開と通常AI会話復帰をすべてPASSとして確認し、PR #48の対象範囲は実機確認完了とする。通話コイン消費は未実装であり、PR #48には含めない。mainマージ済み(2026-07-14、merge commit: `1e3bb3d35179eb62216895c749068ebdcd14ec22`)。
  - staging実機確認済み: 通話時間5分29.025秒、成功15ターン、途中エラーなし。150秒を超える継続通話を確認。staging `voice-turn` version 6 ACTIVE、staging AABで確認済み。
  - 本番Edge Function反映と本番総合実機確認はフェーズ4へ繰り越す。
  - 残り: 通話コインの正式料金、モーニング／イベント発生ロジックと通知起動導線。これらが未完了のため疑似電話全体は`[~]`を維持する。
    - [x] 通話コイン消費の第1段階: ユーザーターンの録音開始から終了確定までをmonotonic clockで計測し、AI通常回答音声は端末の実再生時間だけを複数チャンク合算する。固定の聞き返し音声・エラー案内音声を音声種別で除外し、終了・途中停止時も同じturnの計測を一度だけ確定する。
    - [x] OS共通のusageイベントとTypeScript記録サービスを追加し、`(call_id, turn_id)`・revision単位で古い更新を無視する。DB保存は次ターン開始を待たせず、通信失敗時は永続outboxへ保持して次回起動時に再送する。AndroidとiOSは同じイベントpayloadへ接続する。
    - [x] stagingへmigration `20260714150000_add_call_usage_recording.sql` を適用。`calls`の合計時間・未精算状態と`call_logs`のターン別時間を追加し、本文保存と計測保存を同じ行へupsertするsecurity-definer RPCを実装した。`current_app_user_id()`による所有者確認、revision冪等性、非負制約、権限制限を含む。
    - [x] staging RPC確認: 同一turn再送は二重行にならず、通常AI音声700msとユーザーターン1,500ms、固定音声0msとユーザーターン1,000msの合計が`billable_duration_ms=3,200`となること、通話終了後`settlement_status=pending`となること、コイン残高不変、`coin_transactions`・`coin_consumptions`増加0を確認した。outboxの失敗保持・再起動復元は自動テストで確認した。
    - [x] PR #50レビュー修正後のstaging再確認: 同じturnへのusage二重送信は`recorded`と`stale_ignored`となり1行だけを維持した。content/usage同時送信は本文と計測を同一行へ保持し、5つの追加turn同時送信を含む6行で`turn_index`重複なし。`pending`送信後のusageは`settlement_status=pending`を維持して最新合計1,100msへ更新した。owner付きoutboxは再起動相当で2件から0件へ再送し、1,000ms・`pending`を保存した。コイン残高不変、`coin_transactions`・`coin_consumptions`増加0を再確認した。既適用migrationは競合安全なcall行lockを持つため変更せず、追加migrationは不要。
    - [x] PR #50 AAB 1017実機確認: 同一通話で数ターン会話、固定聞き返し、聞き返し後の通常会話継続、画面エラーなしをPASSとした。対象call末尾`8da8`は9 turn・重複なし、全turnのユーザー時間合計45,703ms、通常AI音声7 turn合計75,406ms、合計121,109ms、全turn finalized、固定聞き返し2 turnのAI音声0ms、通話終了後pendingを確認した。コイン残高不変、`coin_transactions`・`coin_consumptions`増加0。
    - [x] PR #50 空call防止: Android nativeの開始結果を`ok=true`かつ`started=true`のときだけ成功とし、false・欠損・不正payload・rejectを開始失敗へ統合した。開始失敗時は作成済みcallを所有者付きpending outboxへ冪等に積み、0ms・ログなし・終了済みとし、通話画面へ進まずチャット画面へ戻す。開始中の二重押しはDB作成前にguardし、ボタンも無効化する。staging開始失敗相当call末尾`5990`で0ms・pending・completed・ログ0、正常usage相当call末尾`15b9`で1,500ms・pending・completed・ログ1、別ユーザー更新拒否、コイン関連増加0を確認した。
    - [x] PR #50 AAB 1018実機確認: 新規の空active callなし、0msのcompleted callなし、数秒以内の重複callなし、開始中の二重押し防止をPASSとした。対象call末尾`4acf`は`completed`、22,497ms、`settlement_status=pending`、3 turnであり、PR #50の通話usage記録・空call防止の対象範囲は実機確認済みとする。通話失敗はAivis TTSのHTTP 402が原因であり、usage計測・空call防止の失敗を意味しない。並列TTS Promiseの未処理rejectionが発生し、Android診断画面のHTTP 101とclose 1000は失敗turnではなく前turnの残留値だった。TTS Promise rejectionの安全処理、`tts_failed`の一回化、Aivis実HTTP statusの通知、WebSocket明示終了、Android診断値のturn単位初期化、HTTP 402モックテストは別PRへ分離する。
    - [x] TTS失敗安全処理: 並列TTS Promiseのrejectを作成直後に回収し、最初の失敗だけで`tts_failed`を1回通知する。失敗後はLLM/TTSを中止し、後続audioと`done`を送らずWebSocketを1011で明示終了する。Aivis実HTTP statusをpayloadへ含め、Androidはturn開始時にstage、errorCode、HTTP status、close code、exception classを初期化し、WebSocket upgradeのHTTP 101や終了済みturnのclose 1000をAivis失敗値として残さない。Aivis実APIを使わないHTTP 402モックで先頭・途中・複数同時失敗とusage回帰を自動テスト済み。未再生AI音声0ms、再生済みは実再生分のみ、pending、コイン非消費を維持する。staging `voice-turn` version 12はACTIVEで、workflow run `29394935531`のAndroid staging AAB 1019を実機確認し、正常通話とエラー表示なしをPASSとした。PR #51の実機確認は完了。
    - [x] 異常終了した空active callの日次回収: `active`・`unrecorded`・`billable_duration_ms=0`・`call_logs=0`・ownerありを厳密条件とし、最終活動から24時間経過後に`cancelled`へ回収する。0コイン、transaction/consumptionなしで精算完了状態にし、同じcallを再処理しない。24時間は通常通話の最大時間とは別の無活動猶予であり、毎日03:30 JST、1回100件、古い順、`FOR UPDATE SKIP LOCKED`、対象抽出用partial indexを採用した。既存staging active末尾`d7b2`・`5937`は手動変更せず、migration適用後にこの回収RPCを通す検証対象とする。
    - [x] 通話終了時の仮料金一括精算: `billable_duration_ms`を60秒ごとに1コインへ切り上げ、1ms以上は最低2コイン、最大消費なしとする。`settle_call_coins`は`pending`かつ本人所有のcallだけを1トランザクションで精算し、残高・transaction・consumption・call状態を同時更新する。0msは0コインで台帳を作らず、空call回収対象として`pending`に残す。正式料金はF1/F2で別途決定する。
    - [x] 正常終了の再送・冪等性: 所有者付き永続outboxはAndroidの通話終了イベントが示す期待turn数・finalized turn数を保持し、全usage finalizedとpending保存が成功した後だけ精算RPCを送る。TurnUsage作成と終了時finalize・completion snapshotは同じmonitorで直列化し、終了開始後の新規usageを禁止する。AudioRecord初期化中終了、作成直前終了、0 turn、fatal/manual同時終了、通常・再接続turnの実スレッド自動テストをPASSした。DBもcall行lock下で期待turn数とfinalized `call_logs`数の一致を必須にし、精算後の課金時間変更をtriggerで拒否する。content outboxは精算条件から分離し、保存失敗中も独立して再送する。call単位のidempotency key、call行lock、残高行lock、`consumed_coin_id`により、同じcallの4並列実行でも減算・transaction・consumption各1回をstagingでPASSした。
    - [x] calls課金列の権限分離: forward migration `20260715190000_harden_call_settlement_barrier.sql`でauthenticatedの直接作成列を`user_id`・`character_id`・`source`、直接更新列を`pinned`だけに限定した。`billable_duration_ms`、`settlement_status`、`settled_at`、`consumed_coin_id`の直接更新拒否と、既存security-definer usage・pending・settlement RPCの継続動作をstagingでPASSした。
    - [x] 残高不足: 残高をマイナスにせず減算・transaction作成を行わない。`coin_consumptions`へ`skipped/insufficient_balance`を1件だけ記録し、再実行でも増やさない。残高2コインで2コイン必要な別callを同時精算し、片方だけsettled、片方はinsufficient、残高0、合計減算2コインをstagingでPASSした。
    - [x] 残高不足の事前予告・終了画面（仮仕様）: 開始時残高を仮料金の継続可能時間へ換算し、native usageの最新revisionをturn別に合算する。ユーザーターンと通常AI音声再生中だけ残時間を進め、Androidも各課金対象区間へnative停止期限を設定する。残り60秒未満で仮文言の予告を表示、0で既存の終了outbox・finalized barrier・一括精算へ合流し、残高不足の仮終了画面を表示する。2コイン未満はcall作成前に開始しない。正式料金、予告閾値、文言、終了画面内容はF1/F5で後決めする。音声での残高不足案内は今回対象外で、固定音声素材整備後にF5で決める。
    - [x] 終了outboxなしcall・既存pending callの日次回収: 全turnのrevision付きfinalized usageがDBに保存済みで、既存expected件数とも矛盾しないcallだけ、保存済み`call_logs`から合計時間・expected件数・終了barrierを復元する。content-only、未finalized、件数不一致、最終活動から24時間以内は推測せず翌日再試行する。authenticatedの`settle_call_coins`とservice-only日次回収は同じowner照合・call/残高row lock・call単位idempotency keyの精算coreを利用し、残高不足規則と仮料金を変えず二重減算・二重台帳を防ぐ。migration `20260716190000_add_abnormal_call_daily_recovery.sql`はstaging適用済み。usage計測導入前で全ログrevision 0のlegacy callは、forward migration `20260716200000_recover_legacy_unmetered_calls.sql`が指定UTC境界・owner・0ms・状態条件を満たす場合だけ、推測課金と台帳作成なしで一度限りterminal化する。legacy forward migrationはstaging未適用。
    - [x] 通話ログ全仕様: 正常ターンの確定user transcriptとassistant textを`done`時点で保存し、AI音声全再生完了を待つ短期履歴とは分離する。content revision付きowner別永続outboxと所有者確認RPCにより、再送・後着・重複でも同じ`(call_id, turn_id)`行を維持する。ターン別の録音開始・終了・経過時間、通常AI音声の実再生時間に加え、VAD終了理由、応答生成時刻、通常AI音声開始時刻、本文確定状態を日次処理用に保存する。STT confidenceは値が提供されるまでNULL、音声ファイルは任意仕様のため未保存とする。`call_summaries`生成と本文削除の日次処理自体は記憶パイプラインの別工程とする。
    - [x] アプリ内疑似着信: 特別イベントでユーザーが事前にOKした後のforeground画面として、キャラ名・画像枠・着信状態・応答・拒否を表示する。応答は`special_event`開始元を保存しつつ既存の残高・未精算・セッション・native開始guardへ合流し、拒否はcallを作らず閉じる。アプリ内着信音は小音量の短い合成音とし、応答・拒否・画面離脱で停止する。イベント判定と通知起動は別工程で、当面の確認トリガーはdev/stagingの選択済みキャラのチャットだけに表示する。
    - [x] A8-2接続・音の先行実装: 着信画面中にcall未作成・録音未開始でstandby WebSocketを準備し、応答後の既存開始guard通過時だけ同じsocketをactiveへ昇格する。拒否と開始拒否ではsocketを破棄する。ユーザー発信は接続readyまでコール音を鳴らす。Androidの着信音・コール音・バイブは通常／バイブ／サイレントのringer modeへ従う。固定第一声は`fixed_voice_assets`工程、本番音源はF6で差し替える残作業とする。
    - [x] iOS疑似電話の第1段階: ユーザー発信を`AVAudioEngine` / `AVAudioConverter`による24kHz・mono・PCM16録音、`URLSessionWebSocketTask`、`AVAudioPlayer`へ接続する。`call-protocol.json`由来の共通イベントpayloadをTypeScriptへ通知し、usage計測、owner付きoutbox、残高判定、ログ保存、一括精算、残高不足終了、WebView通話画面をAndroidと共有する。iOS専用の業務ルールは作らない。AlarmKit連携、CallKitは範囲外。
    - [x] iOS A8-2: Androidと同じアプリ内着信画面・応答・拒否を有効化し、着信中はcall/録音/usageを開始せずstandby WebSocketだけを準備する。応答時は共通開始guard後に準備socketを昇格し、拒否・離脱時は破棄する。着信音・ユーザー発信コール音はiOS nativeのPCM合成音とし、`.ambient`でサイレントスイッチ、`AudioServices`で端末のバイブ設定へ従う。固定第一声、本番音源、イベント発生・通知起動は別工程とする。
    - AndroidとiOSの疑似電話は同じOS共通の計算・精算方式へ接続する。AndroidまたはiOSだけで完結する業務ルールにはしない。

### 疑似電話の会話品質: 確認済み・未解決の事実

- 実機通話中、AIが料理を提案した。
- 少し別の会話をした後で、その料理のPFCを質問した。
- AIは先ほど提案した料理内容を保持できておらず、会話が不自然になった。
- 現時点では原因分析・対応方法・実装方針は未決。この事実はPR #46の完了範囲には含めず、今後の会話品質・文脈保持検討時に扱う。

### オンデバイスTTS本実装（採用決定・2026-07-18）

- 前提: `poc/on-device-tts`のPoC実験は終了し、オンデバイスTTS採用を決定済み。仕様は`voice_companion_spec.md` A8-3 / H1 / H4を正とする。オンデバイスTTSはv1の6キャラ全員が対象で、桜音は方式・量子化・配信の検証基準モデルとして用いる(桜音1モデルに絞る前提にはしない)。
- [x] 疑似電話・通知のAI音声再生を、クラウドTTS(Aivis)からオンデバイスTTS(ONNX Runtime + Style-Bert-VITS2系AIVMX)へ置き換える現行実装。TTS本体・DeBERTa BERTともに動的INT8、文末(。!?)単位のストリーミング合成・逐次再生（読点分割は不採用）。コード実装・CI greenで、Android実機でも通常AI音声と固定聞き返しのオンデバイス再生を確認済み。正式な最終構成への置き換えは下記8段階で行う。
- [x] 配信を「共通BERT＋キャラ別モデル」方式へ分割するコード実装。声モデルは初期アプリに同梱せず、共通BERT bundle(共有DeBERTa BERT＋共通ライセンス)を端末へ1回だけDL・保存・再利用し、キャラ別bundle(AIVMX＋voice-config＋モデルライセンス)をキャラごとにDL・保存する。キャラ切替(桜音→花音→桜音)でも共通・既取得キャラモデルを再DLせず(キャッシュ)、片方のDL/検証/更新失敗が既存の正常インストールを壊さない(失敗隔離＝原子的入替＋失敗時ロールバック)。設定は`VITE_TTS_COMMON_BUNDLE_URL` / `VITE_TTS_COMMON_VERSION` / `VITE_TTS_CHARACTER_BUNDLES`。※実装・自動テスト・CI green。
- [x] Cloudflare R2(staging)への共通/キャラ別bundle配置と公開URL取得確認。手動実行専用のアップロードworkflowでcommon／桜音／花音の3bundleをアップロード成功(workflow run `29751863325`)、アップロード後のサイズ＋SHA-256 read-back検証が成功。さらに`public_base_url`付きで再実行し、公開URL(`https://pub-7bcdc02f44a14b68a333bc9b9ee07f73.r2.dev`配下)からの取得確認(サイズ＋SHA-256一致)も成功(workflow run `29755001244`)。
- [x] staging AAB build workflowへ`VITE_TTS_*`(`VITE_TTS_COMMON_BUNDLE_URL` / `VITE_TTS_COMMON_VERSION` / `VITE_TTS_CHARACTER_BUNDLES`)をGitHub staging環境変数から注入する配線と確定値の事前検証を追加。common version=`1`・bundle id `sakune`/`kanon`はコード/bundle定義から確認済み。
- [x] staging環境変数への実値設定と花音のstaging DB登録。桜音・花音のstaging `characters.id`を取得(桜音=`3ff18df6-e2e2-49ee-afe8-01257ed53fb5`、花音=`d7410a62-3e02-4d17-b272-14ba0262ae3a`)。花音はstaging DBへ手動追加済みで、再現用migration `supabase/migrations/20260721090000_add_kanon_character.sql`を追加済み(適用はしていない)。GitHub staging環境変数3件(`VITE_TTS_COMMON_BUNDLE_URL` / `VITE_TTS_COMMON_VERSION` / `VITE_TTS_CHARACTER_BUNDLES`)を設定済み。
- [~] `Android Staging AAB` build、端末からの実ダウンロード、Android実機確認。R2配置・公開URL確認・workflow配線・staging環境変数設定・花音DB登録は完了。AAB 1060では通常AI音声、録音準備音の可聴、固定聞き返しのオンデバイスTTS再生、通話画面のボタン位置解決を実機確認済み。初回音声までの長い待ちと文間遅延は未解決であり、正式Android通話実装への移行も未完了のため`[~]`を維持する。
- [x] 量子化(TTS/BERTの動的INT8生成)とその検証はローカルで行わず、CI内で実行(ローカルでの重い処理は禁止)。CIで桜音＋花音の実AIVMXとDeBERTaから共通1＋キャラ2 bundleを生成・検証(run `29744823614` green)。実モデルはGit/AABに入れずCI内のみ。
- [ ] 本実装には、通話フリーズ切り分け用に STT / LLM / TTS / 再生 の各区間でどこで停滞したかを記録するログを含める。
- [ ] 低速端末対策。基準端末水準に満たない端末でもクラウドTTSとクラウドTTSフォールバックは使わず、オンデバイスTTS経路内の性能改善と必要に応じた対応端末条件の扱いを実測に基づいて決める。F6の未決項目として扱う。
- 実測基準値(参考): POCO X6 Pro / Dimensity 8300-Ultra / CPU EP 4スレッドで、1文目再生開始まで約0.9〜1.0秒、RTF約0.7、初期化約5.5秒(常駐・初回のみ)。iPhone 12相当を当面の仮推奨基準端末とする。

**正式Android通話実装の確定方針:**

- `AndroidTtfaTestPlugin`は検証用の暫定実装であり、正式実装として残さない。新実装へのstaging切替と実機確認後、旧Pluginを丸ごと削除する。
- 新実装は最初から、通話状態、接続、録音、オンデバイスTTS、課金・終了、診断の責務ごとに分割する。
- 新実装はオンデバイスTTS専用とする。クラウドTTS、クラウドTTSフォールバック、クラウドTTSの生成・受信・再生・切替分岐は移行対象外とし、旧Plugin削除時に完全削除する。
- モデルsession、AudioTrack、executorをturn・文ごとに不要に再作成しない。責務分割による余計なJSON変換、PCMコピー、スレッド切替を増やさない。

**共通化実装のv5.41時点の記録(2026-07-21・R2配置＋公開URL確認＋workflow配線＋staging環境変数設定＋花音DB登録まで完了、staging AAB build・実機確認待ち):**

- 構成: 共通BERT bundle1個＋桜音character bundle＋花音character bundle。桜音＝配信検証の基準モデル、花音＝共通化検証用の2体目(同一作者・同一ACML 1.0・同一利用条件)。
- bundle payload実サイズ: common=475,567,269B(約453MiB)、桜音=255,610,450B(約244MiB)、花音=251,080,348B(約239MiB)。端末DL量=共通＋当該キャラ(桜音約697MiB、花音約693MiB)、両方導入でも共通は1回=約937MiB。
- CI: `Build on-device TTS model bundles (staging)` run `29744823614` green、`Android On-Device TTS Test` run `29744823724` green。
- PR #67はmainへマージ済み(merge commit `45e6c176cf30493f78c6037504856902af1781a1`)。
- R2: 手動実行専用workflowでcommon／桜音／花音の3bundleをCloudflare R2(staging)へアップロード成功(workflow run `29751863325`)。アップロード後にサイズとSHA-256 read-back検証が成功。さらに`public_base_url`付き再実行で公開URL(`https://pub-7bcdc02f44a14b68a333bc9b9ee07f73.r2.dev`)からの取得確認も成功(workflow run `29755001244`)。
- staging config: `Android Staging AAB` workflowへ`VITE_TTS_*`をGitHub staging環境変数から注入する配線＋`verify_staging_tts_env.mjs`の事前検証を追加。common version=`1`、bundle id=`sakune`/`kanon`(コード/bundle定義から確認)。GitHub staging環境変数3件は設定済みで、`VITE_TTS_CHARACTER_BUNDLES`のキーは桜音=`3ff18df6-e2e2-49ee-afe8-01257ed53fb5`／花音=`d7410a62-3e02-4d17-b272-14ba0262ae3a`(staging `characters.id`)。
- staging DB: 花音はstaging DBへ手動追加済み。再現用idempotent migration `supabase/migrations/20260721090000_add_kanon_character.sql`を追加済み(適用はしていない)。
- 当時の未了: `Android Staging AAB` build未実行、Android実機確認未実施。現在状態は下記「AAB 1060時点の現在状態」を正とする。

**PR #69 Android追加実装のstaging AAB・実機結果(2026-07-22・v5.43):**

- PR #69(branch `agent/staging-tts-vite-config`・HEAD `ae3b1ce64ef631c38344b2871210b0bf2a74fd75`)で①ピッ音復旧②導入済みモデルのみのmodel_load prewarm③TTS合成producer/AudioTrack再生consumer分離④通常回答を最大2文・約80文字目安にするサーバー側制御⑤固定聞き返しのAndroidオンデバイス化⑥JUnitテストのコンパイル修正を実装。commit `7eba583` / `0f0dc1e` / `4b1b8d2` / `7f4327f` / `05c513a` / `ae3b1ce`。
- 自動テスト: ローカル `npm test` 228件PASS、`npx tsc --noEmit` PASS、`npm run build` PASS、`git diff --check` PASS(Denoはローカル未導入で未実施)。GitHub Actions `Android On-Device TTS Test`(run `29847433103`, HEAD `ae3b1ce`, success)、`Android TTS Bundle Installer Test`(run `29847435776`, HEAD `ae3b1ce`, success)。最初のrun `29846761486` / `29846763742`は追加JUnitテストのヘルパー引数型不一致で失敗し `ae3b1ce` で修正後green。
- staging反映: staging `voice-turn`へPR #69最新HEADをdeploy(`voice-turn`はVERSION 13、`chat-reply`は変更なし。production・DB・migration適用・R2・env・secretは変更なし)。
- Android Staging AAB: workflow run `29848258099`、branch `agent/staging-tts-vite-config`、HEAD `ae3b1ce64ef631c38344b2871210b0bf2a74fd75`、versionCode `1049`、versionName `staging.1.0.49`、artifact `android-staging-aab`、buildはsuccess、PR #69はDraftのまま。
- AAB 1049実機結果(ユーザー発信): 3コール後にデバッグ画面へ移動→「接続中」のまま停止、コール音が鳴り続け通話開始できない。AAB 1048ではユーザー発信できていたため1049で回帰を確認。
- AAB 1049実機結果(AI着信): 応答後の会話は可能。1文目と2文目の無音は体感上改善を確認できず。通常回答再生後のピッ音は鳴らなかった。
- AAB 1049実機結果(固定聞き返し): 無言時に音声は再生されたが、実機上ではオンデバイス経路か判定できておらず、再生後のピッ音は鳴らなかった。
- AAB 1049実機結果(操作バー): スピーカー切替/終話ボタンの位置は改善しておらず、既存WindowInsets修正の実機効果は確認できなかった。
- 未完了(すべて未完了として維持): ユーザー発信回帰の原因特定と修正／ピッ音復旧／文間無音削減／固定聞き返しのオンデバイス経路確認／操作バー修正／AndroidオンデバイスTTS実用確認。オンデバイスTTS工程全体も未完了。PR #69はReady化・マージ不可、Draft維持。
- 次工程: 実装ではなくAAB 1049の回帰原因調査。原因は未特定のため推測した原因は記載しない。

**AAB 1060時点の現在状態(2026-07-23・v5.48):**

- 録音準備音はAndroid実機で可聴確認済み。
- 固定聞き返しはAndroid実機でオンデバイスTTSによる再生を確認済み。
- 通話画面のボタン位置は解決済み。
- 初回音声までの長い待ちと文間遅延は未解決。
- PR #69はOpen・Draftを維持し、mainへはマージしない。

**正式Android通話実装への移行順序:**

1. [x] 現行責務・依存関係の調査(PR1): `AndroidTtfaTestPlugin`の通話状態、接続、録音、TTS、課金usage、終了、診断の責務と依存先を分離し、新実装へ移すcontroller portと、移さず削除するクラウドTTS分岐を固定した。
2. [x] 正式実装の骨組み作成(PR1): 正式Plugin `AndroidVoiceCall`、`VoiceCallSession`状態機械、controller port、診断境界を追加した。旧Pluginと`src/appMain.ts`を変更せず稼働経路として維持し、正式Pluginはlive controllerへ接続しない。
3. [x] 接続・録音移行: PR2で`VoiceConnectionController`へactive接続、incoming/次turn standby準備・昇格、再接続、close、PCM binary/JSON/turn context送信、stale socket/turn判定を移行し、PR3で`VoiceRecordingController`へ24kHz・mono・PCM16のAudioRecord、VAD、speech_end一回送信、無音固定聞き返し要求、固定聞き返し/録音準備音/reconnect後の録音復帰、call/turn/録音世代によるstale無効化、冪等stop/releaseを移行した。PCMはread bufferを直接接続Controllerへ渡す。旧Pluginと`src/appMain.ts`は稼働経路として残し、正式Pluginへの切替は工程6で行う。
4. [x] オンデバイスTTS移行: `VoiceTtsController`へモデル準備状態、通常回答の文単位合成・順序付き再生、固定聞き返し、最終文後の録音準備音、再生完了/失敗/取消、call/turn/TTS/再生/モデル世代によるstale無効化、reconnect・新turn・通話終了時の冪等停止を移行した。既存オンデバイスpipelineの共通BERT session、キャラ別TTS session、常駐AudioTrack、producer/playback executorを再利用し、録音復帰はSession経由で`VoiceRecordingController`へ通知する。クラウドTTS経路は移していない。旧Pluginと`src/appMain.ts`は稼働経路として残し、正式Pluginへの切替は工程6で行う。
5. [x] 課金・終了処理移行: `VoiceUsageController`へユーザーturn実時間と通常AI音声の実再生時間だけを集計するusage責務を移し、固定聞き返し・録音準備音・無音padding・待ち時間を対象外にした。`VoiceCallSession`へusage・終了境界を接続し、call/turn/録音/TTS/再生/session世代によるstale防止、usage・終了通知の二重実行防止、録音→TTS→socket停止順序を固定した。既存の`callUsageUpdated` / `callUsageComplete` / `callFatalError`契約からOS共通TypeScriptのowner付きoutbox、残高判定、一括精算へ接続し、AndroidからDBへ直接書き込まない。
6. [x] staging切替とAndroid実機確認: staging Androidの実通話経路を正式`AndroidVoiceCall`へ切り替えた。AAB 1061のWebSocket URL・診断不具合とAAB 1062のコール音・着信音・バイブ・audio route不具合を修正し、AAB 1063でユーザー発信、疑似着信、通常会話、コール音、着信音、バイブ、AI音量、デバッグ表示、録音準備音、録音復帰を実機確認した。旧Pluginは工程7まで残し、PR #69はDraftを維持する。
7. [x] 旧AndroidTtfaTestPluginの削除と残存参照整理: `AndroidTtfaTestPlugin`本体、旧Plugin専用usage barrier、登録、Web bridge・選択・fallback、専用テストランナーを削除し、実行コード内の旧Plugin参照を0件にした。正式Android通話経路と共通resource・dependencyは維持し、AAB 1064で発着信・音・バイブ・AI音量・診断・録音復帰を実機確認した。
8. [ ] 初回音声・文間遅延の計測と改善（後続の性能改善）: Android正式Pluginへの必須移行は工程7までで完了しており、本工程は移行完了条件に含めない。新実装で区間別実測を取り、未解決の初回音声までの長い待ちと文間遅延を改善する。責務分割による余計なJSON変換、PCMコピー、スレッド切替を増やさない。

次工程はPR #69のマージ後、別branch・別PRで「iOS正式疑似通話・オンデバイスTTS接続」。

- [ ] 通知（A10）: デイリー生成通知（通知文＋入口メッセージの事前生成）、`notification_candidates` / `notification_logs` への保存、同じ文脈IDでの通知文・入口メッセージ連携、関係値重み配分＋lover毎日確定、キャラ個別ON/OFF、`device_installations` を使った有効端末送信、重複防止
- [ ] イベント・告白（A11）: 発生条件（サーバルール）、通常通知→疑似LINE入口メッセージ→電話していい？→OK→アプリ内疑似着信→応答→疑似電話の導線、告白・関係状態変更・呼び方変更の明示同意フロー、pending状態、未応答時の戻し/保留、lover化と文脈変化、iOS/Android共通の告白導線
- [ ] 猫（A3-6）: ランダム猫（ルール）／AI猫（分類）、懐き度

## フェーズ4. リリース前チェック（Z-9）

- [ ] mainに保存された本番未反映migration、Edge Function、secret、Dashboard設定を一覧化する
- [ ] stagingでリリース対象のmigration、Edge Function、RLS、RPC、主要導線を最終確認する
- [ ] ユーザーの明示指示後、本番Supabaseへ追加migrationをまとめて反映する
- [ ] 本番Edge Functionをまとめてdeployし、必要secretとDashboard設定を反映する
- [ ] 本番RLS実動作と`current_app_user_id()`による`public.users.id`解決を確認する
- [ ] 本番Supabase接続のリリース候補アプリで主要導線を総合確認する
- [ ] 実機検証全項目（マイク権限 / Android `AudioRecord` 24kHz mono PCM16 / STTストリーミング / VAD / TTS再生 / TTS後の発話待ち復帰 / 残高不足 / 二重消費なし）
- [ ] Bluetoothヘッドセットでの通話入出力対応（現行は両OSとも本体受話口/スピーカーのみ）
- [ ] 電話着信・他アプリ音声による割り込み終了後の安全な自動復帰（現行はusage確定・outbox・一括精算へ合流して終了し、利用者が新しい通話を開始する）
- [ ] モーニングからの本番AI通話開始 / 通知→疑似LINE経由の通話開始（Androidは `USE_FULL_SCREEN_INTENT`、メーカー別full-screen intent動作、Foreground Service音、通知タップフォールバック、専用Activity遷移、実AI接続込みで再確認）
- [ ] ストア審査確認（iOSでCallKit不使用の方針が保てているか、疑似電話がガイドライン適合か）
- [ ] ストア商品設定（消耗型コイン / 非消耗キャラ枠 / 自動更新サブスク）

---

## 補足: 未決（Z）と本書の対応
- Z-1〜Z-3（数値）→ フェーズ2
- Z-4（記憶の日数・遅延）→ フェーズ0/3で実測しながら
- Z-6（声・素材）→ フェーズ0。声モデルの商用ライセンス確認とv1仮採用は完了。TTS方式はオンデバイスTTS(Style-Bert-VITS2系AIVMX動的INT8)採用を決定済み(2026-07-18)で、v1の6キャラ全員が対象。桜音を検証基準モデルとした実測でTTS品質(INT8劣化なし)・速度基準(1文目 約0.9〜1.0秒)も確認済み。残りはキャラ画像・猫の鳴き声・固定音声素材の作成、低速端末対策の方針決定(F6)。詳細はspec A8-3 / H1。
- Z-7（画面設計）→ フェーズ1/3で各画面を作りながら確定
- Z-8（RLS・API）→ フェーズ1。RLS・制約・インデックスのDB土台と指定migration 5件の本番反映は完了済み。v4.3匿名Auth方針でのRLS実動作と `current_app_user_id()` による `public.users.id` 解決は staging で検証済み・PASS（2026-07-11、漏れなし）。今後追加するmigration、Edge Function、secret、RLS/APIは開発中stagingで検証し、本番反映・本番RLS確認・本番`current_app_user_id()`確認はフェーズ4でまとめて行う。API設計、Edge Function整理、アプリ側読み書き実装は各機能工程で進める。匿名サインインと引き継ぎコードを前提にする。`transfer_codes` はテーブル追加のみで、発行/引き換え用 Edge Function / RPC とUIは未実装。
- Z-9（実機検証）→ フェーズ0・フェーズ4。Androidモーニング導線のフェーズ0検証は合格、残りは本番AI接続・フォールバック・リリース前総合確認。
