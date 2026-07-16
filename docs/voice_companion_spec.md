# VoiceCompanion 仕様書 v4.5

**版数: v4.5 ／ 最終更新日: 2026-07-16**

変更履歴:

- 2026-07-16 v4.5: A8-2を新設し、アプリ内疑似着信の事前接続、着信音・コール音、固定第一声、マナーモード追従、固定音声素材工程との分離を定義。
- 2026-07-14 v4.4: チャットと通話のコイン計算方式を分離。チャットはAI返信1回ごとの固定消費、通話はユーザーターンの経過時間とAI通常回答音声の実再生時間による時間制とし、AI処理待ち・通信待ちは対象外、通話終了時に1通話分を一括精算する方針へ変更。正式な時間単位・消費量・最低消費・端数処理は未決のままとする。
- 2026-07-11 v4.3: 製品仕様は変更せず、build planの開発運用を機能単位方式へ変更したことに伴い、版数・最終更新日・変更履歴を同期。
- 2026-07-10 v4.2: 製品仕様は変更せず、build planとの版数同期と自立型エージェント運用追加に伴い、版数・最終更新日・変更履歴を更新。

このファイルをVoiceCompanionの正本仕様とする。
作業順、PR方針、検証手順は docs/voice_companion_build_plan.md で管理する。

---

---

# 0. v4.0の前提

## 0-1. 正本ルール

- この仕様書は「何を作るか」を定義する。
- 作業順、PR方針、検証手順は `voice_companion_build_plan.md` で管理する。
- 過去スレの断片や旧メモは正本として扱わない。

## 0-2. ログイン画面なし

VoiceCompanion v1はログイン画面を持たない。

初回起動時、アプリはSupabase Authの匿名セッションを自動作成する。ユーザーに見せる最初の体験は、ログインではなく名前入力とキャラ選択である。

## 0-3. アプリ内ユーザーID

アプリ内ユーザー主キーは `public.users.id` とする。

`auth.users.id` はSupabase Authの認証セッションIDであり、各ユーザー別テーブルの `user_id` として直接使わない。

各ユーザー別テーブルの `user_id` は原則 `public.users(id)` を参照する。

## 0-4. 引き継ぎ

端末変更・再インストール時のアプリ内データ引き継ぎは、Google / Apple / emailログインではなく、引き継ぎコードで行う。

旧端末で引き継ぎコードを発行し、新端末で入力することで、同じ `public.users.id` へ戻す。

引き継ぎコードなしでアプリを削除・再インストールした場合、アプリ内データは原則復元できない。

## 0-5. 本番導線から削除するもの

以下はv1本番導線から削除する。

- Googleログイン
- Appleログイン
- email/passwordログイン
- ログイン画面
- OAuth callback deep link
- SocialLogin依存
- Google client ID前提の本番ログイン
- Apple Sign In capability前提の本番ログイン
- `auth.users.id` をアプリ内ユーザー主キーにする設計
- `RevenueCat app user id = auth.users.id` の設計

email/passwordは過去のデバッグ疎通実績としては残るが、本番導線では使わない。

---

# A. 体験仕様

## A1. アプリの核

VoiceCompanionの核は「AIと、アニメ声で会話する」こと（文字・音声）である。

恋愛・告白は後から乗った付加要素であり、アプリの核ではない。

優先順位は「会話×アニメ声＝主／恋愛＝従」。

実装で迷ったら、会話・アニメ声・キャラが日常に現れる体験を優先し、恋愛要素は削る/後回しにする判断もあり得る。

VoiceCompanionは、ユーザーが自分から開いて話すだけのAIチャットではなく、AIキャラ側からユーザーの日常に現れるアプリである。

ただし通知専用アプリではない。ユーザー発信の疑似LINE・疑似電話も、関係行動として扱う。

主な入口:

- モーニングコール
- デイリー通知
- 気まぐれ通知
- イベント通知
- 特別イベント疑似着信
- 疑似LINE
- 疑似電話
- ランダム猫 / AI猫

AlarmKitやAndroid full-screen intentは、この体験を成立させる実装手段であり、アプリ価値そのものではない。

## A2. v1 MVPスコープ

v1に入れるもの:

- ログイン画面なしの匿名セッション自動作成
- 初回名前入力
- 初回キャラ選択
- 初回好意度/第一印象選択
- 引き継ぎコード発行
- 引き継ぎコード入力
- 人間キャラの選択肢
- 複数キャラ保有構造
- 無料/有料によるAI利用キャラ枠
- ランダム猫 / AI猫
- 疑似LINE
- 疑似電話
- モーニングコール
- デイリー通知 / 気まぐれ通知 / イベント通知
- 通知ON/OFF、サイレント時間帯
- iOS 26以上のAlarmKit導線
- Android full-screen intent notificationを第一候補にしたモーニング導線
- キャラごとの通知ON/OFF
- 告白イベント
- チャット本文・通話見出しの90日自動削除＋ピン留め
- キャラの長期記憶
- コイン/従量課金
- キャラ枠課金
- 月額コインパック
- RevenueCat連携
- サーバ側コイン管理 / 消費ログ
- 退会・アカウント削除
- 安全・コンテンツ方針

v1から外すもの:

- 本格的なアバターアニメーション / Live2D / 3Dモデル
- 声クローン
- ユーザー画像からキャラ化
- キャラからの画像送信
- 自由画像アップロード
- ベクトル記憶
- カレンダー連携
- 多言語配信
- すべての通知を毎回AI生成する重い通知生成
- カスタムAlarmKit音 / アプリ内Web Audioループ音
- 高度な雑音環境対応 / 完璧なVAD最適化
- Realtime API全面移行

Google / Apple / emailログインは「v1ではやらない候補」ではなく、現行v1方針から削除する本番導線である。将来の任意バックアップ用途はv2以降候補に回す。

## A3. 初回起動・オンボーディング

初回起動時の流れ:

1. アプリ起動
2. Supabase anonymous sessionを自動作成
3. `auth.users.id` から `public.users.id` を解決
4. `public.users.display_name` が未設定なら名前入力へ進む
5. キャラ選択へ進む
6. 初回好意度/第一印象を3段階で選ぶ
7. `user_characters` と `character_relationships` の初期行を作成
8. 必要に応じて `user_character_memory`、`notification_preferences`、`device_installations` を作成
9. 疑似LINEホームへ進む

## A3-2. 名前入力

初回に必須なのは氏名または名前のみ。本名である必要はない。

呼び方の初期指定UIは置かない。呼び方は関係進行で変わる体験として扱う。

## A3-3. ログイン画面を出さない

初回起動時にログイン画面は出さない。

セッション作成中は、短い準備中表示のみとする。失敗時は、通信エラーまたは初期化エラーとして扱い、Google/Apple/emailログインへ誘導しない。

## A3-4. 再起動時

既存匿名セッションが残っている場合、そのまま同じ `public.users.id` を解決する。

セッションが失われた場合、新しい匿名ユーザーが作られる。旧データに戻るには引き継ぎコードが必要。

## A3-5. 起動 / 準備中画面

匿名セッションの復元または自動作成、`public.users.id` 解決、初期設定状態の確認中は、短い準備中画面を表示する。

表示文言は「呼出中・・・」とする。「・・・」のみ静かにアニメーションしてよい。

この画面は通常起動、通知起動、モーニング起動でも違和感がない共通ローディングとして扱う。ただし長く表示せず、起動処理が終わり次第すぐ次画面へ遷移する。

匿名セッション作成や初期化に失敗した場合は通信エラーまたは初期化エラーとして扱い、Google / Apple / emailログインへ誘導しない。

## A3-6. 初回名前入力画面

入力項目:

- 姓
- 名
- せい
- めい

表示文言:

- 本名である必要はありません。

保存項目:

- `family_name`
- `given_name`
- `family_name_kana`
- `given_name_kana`
- `display_name`

`display_name` は、姓と名を連結して作る。

呼び方の初期指定UIは作らない。「キャラがあなたを呼ぶための名前です」のように呼び方を固定する説明も置かない。呼び方は関係進行で変わる体験として扱う。

## A3-7. 初回キャラ選択画面

初回キャラ選択では、人間6人 + 猫2匹の合計8キャラを表示する。

仮キャラ:

- まお
- コハク
- 桜音
- にせ
- fumifumi
- 猩々博士
- AI猫
- ランダム猫

本番キャラ名・本番画像・本番説明は後続PRで差し替える。ここではキャラ選択のUI土台と選択状態保存を優先する。

無料ユーザーがAI利用メインキャラとして選べるのは、人間キャラ1人またはAI猫1匹である。

ランダム猫はAI不使用・コイン消費なしの無料付属扱いであり、初回AI利用メインキャラとしては選択不可にする。画面上には表示するが、「無料付属」など、メイン選択対象ではないことが分かる表示にする。

## A3-8. キャラ名 + 初回関係設定画面

キャラ選択後、選んだキャラの仮画像、キャラ名入力欄、初回関係3択、開始ボタンを表示する。

キャラ名はユーザーが自由入力できる。入力欄には選択キャラのデフォルト名を初期表示する。

初回関係3択:

1. まずは気軽に話したい
2. 仲良くなれそう
3. 最初から気になる

意味:

- まずは気軽に話したい → 低めの初期親密度
- 仲良くなれそう → 標準の初期親密度
- 最初から気になる → 高めの初期親密度 / 恋愛イベント許可寄り

開始時に `user_characters` の初期行と `character_relationships` の初期行を作成する。実際の数値カラム名・保存値はDB schemaに合わせ、不要なDB構造を増やさない。

# A4. キャラ構成

## A4-1. 人間キャラ数

男女両方を狙う。初期案は女性3人・男性3人程度、合計6人程度。

構造上は男女複数キャラ前提。

v1で6人全員か、絞って順次追加かはF章の素材・運用側で決める。

## A4-2. 無料ユーザー範囲

無料ユーザーは、AIを使うメインキャラを1つ選べる。

選択肢は、人間キャラ1人またはAI猫1匹。

ランダム猫はAI不使用・コイン消費なしで無料付属。

初回に必須なのは名前だけ。本名である必要はない。呼び方の初期指定UIは置かない。呼び方は関係進行で変わる体験として扱う。

## A4. キャラ構成

人間キャラは男女複数。初期案は女性3人・男性3人程度、合計6人程度。

無料ユーザーはAIを使うメインキャラを1つ選べる。選択肢は人間キャラ1人またはAI猫1匹。

ランダム猫はAI不使用・コイン消費なしで無料付属。AI猫はAIを使うため、人間キャラ同様にAI利用時コイン消費対象。

キャラごとに持つもの:

- 疑似LINE履歴
- 疑似電話履歴
- 記憶
- 親密度または懐き度
- 呼び方
- 関係状態
- 通知元
- 通知頻度用状態
- 最終会話日
- キャラ画像
- 声
- persona

ユーザー共通で持つもの:

- コイン残高
- 課金状態
- 基本プロフィール
- 通知全体ON/OFF
- サイレント時間帯
- スヌーズ時間
- 端末情報
- 引き継ぎ状態

## A5. 関係性・親密度・呼び方

人間キャラは以下を持つ。

- 親密度
- ユーザー側好意度
- 関係状態
- 恋愛イベント許可フラグ

関係状態:

- `friend`
- `close_friend`
- `romance_event_pending`
- `lover`

猫は懐き度と最後に構った時間を持つ。猫は恋愛状態・告白イベントを持たない。

親密度・懐き度の増減は、キャラ側からの通知・モーニングへの反応と、ユーザー発信のチャット・疑似電話・猫への接触の両方で変化する。

AIが自由に点数を決めない。サーバ固定ルールで増減する。LLMは分類や候補抽出に留め、最終判定はサーバ側が行う。

呼び方は関係の到達点として扱う。登録時に必須なのは名前だけ。呼び方の初期指定UIは置かない。呼び方はキャラごとに持ち、関係値で解禁される。

ユーザーが「〇〇と呼んで」と希望しても、関係値/親密度が足りなければキャラが断る。受ける/断る判定はサーバ固定ルール。セリフ生成はLLMでよい。

## A6. 猫仕様

猫は人間キャラと同じ土俵で疑似LINE・疑似電話・モーニングに乗る。違いは人間語を返さない点。

- 猫は恋愛要素なし
- 猫は親密度ではなく懐き度
- ランダム猫はAIなし・無料
- AI猫は感情/反応分類あり・コイン消費
- 疑似LINEでは鳴き声・しぐさテキスト
- 疑似電話/モーニングでは鳴き声音声
- 鳴き声はTTSではなく音声ファイル
- v1ではなで等のタップUIや本格アニメは持たない

## A7. 疑似LINE

疑似LINEは、ユーザーとキャラの基本接点である。通知を開いた後の入口、通話への導線、イベント導線、通話見出しの履歴表示を兼ねる。

疑似LINE履歴は `chat_messages` に保存する。チャット本文と通話見出しは90日で自動削除する。重要なものはピン留めで残せる。

疑似電話後は、疑似LINEに通話見出しを差し込む。通話本文の逐語ログは疑似LINEには入れない。

文字チャットは、AI返信1回ごとの固定コイン消費を基本とする。チャットの文字数上限があるため、入力・出力トークン数による細かな従量計算は行わない。1返信あたりの正式な消費量はF1で決める。固定文言・システム都合の文言・エラー文言は消費しない。

## A7-1. 疑似LINEホーム

疑似LINEホームは、LINEのトーク一覧のようなメイン画面であり、アプリを開いた後の中心画面である。

表示するもの:

- 上部: トークメイトAI
- 上部: コイン残高
- 上部: 設定アイコン
- 中央: 8キャラのトーク一覧
- 下部ナビ: トーク / キャラ / コイン / 設定

キャラ一覧は8キャラすべてを表示する。初期設定で選んだキャラを一番上に表示し、未選択キャラは選択済みキャラより弱く見せる。

ホームでは最後のメッセージ本文を表示しない。ホームで本文を見せると、ユーザーがチャットを開かずに満足する可能性があるためである。

代わりに状態だけを表示する。状態表示の例:

- 新着あり
- 会話できます
- 追加できます

選択済みキャラをタップした場合は疑似LINEチャット画面へ遷移する。未選択キャラをタップした場合はキャラ管理 / キャラ追加導線へ遷移する。

## A7-2. 疑似LINEチャット仮画面

PR #28時点の疑似LINEチャット画面は、本格AIチャットではなく、ホームからの遷移先を確認するための仮画面でよい。

表示するもの:

- 戻る
- キャラ名
- 通話ボタン
- まだ会話はありません
- 入力欄
- 送信ボタン

入力欄と送信ボタンは仮であり、本格AI応答、`chat_messages` へのDB保存、コイン消費は後続PRで実装する。

## A7-3. キャラ管理 / キャラ追加仮画面

未選択キャラをタップしたとき、または下部ナビの「キャラ」から遷移する最低限の仮ページを用意する。

表示するもの:

- キャラ管理
- キャラ追加機能は後続PRで実装します
- 戻るボタン

キャラ追加、複数キャラ保有、キャラ枠課金の本実装は後続PRで扱う。

## A7-4. 設定画面 / テーマ / 規約ページ

設定画面には以下を表示する。

- デザインテーマ
- 通知設定
- モーニングコール設定
- コイン / 課金
- 引き継ぎコード
- アカウント削除
- 利用規約
- プライバシーポリシー

PR #28時点で実動作させるのはデザインテーマ切替のみとする。通知設定、モーニングコール設定、コイン / 課金、引き継ぎコード、アカウント削除は仮モーダルまたは仮ページで受ける。

デザインテーマは2種類とする。

- `soft`: やさしいブルー × ラベンダー
- `dark`: ダークネイビー × シアン

テーマは `localStorage` に保存し、DBには保存しない。テーマ切替のためのmigrationは不要である。

実装はCSS変数で切り替える。`body` またはroot要素に `data-theme="soft"` / `data-theme="dark"` を付け、同じHTML/TS構造で色だけを切り替える。

利用規約とプライバシーポリシーはモーダルではなくページとして開く。PR #28時点では仮ページでよく、正式本文は後続PRで差し替える。

## A8. 疑似電話

疑似電話の入口:

1. ユーザー発信の通話
2. モーニングから会話へ進む通話
3. 特別イベント疑似着信に応答する通話

通常通知は直接通話の入口ではない。疑似LINE画面から通話ボタンで疑似電話へ進む。

iOSではCallKit / VoIP pushを使わない。OSレベル全画面を出せるのはモーニングのAlarmKitのみ。

Androidではモーニングにfull-screen intentを使う。疑似着信音はForeground Serviceで鳴らし、画面表示に依存させない。

特別イベント疑似着信は、通常通知 → 疑似LINE → 電話していい？ → OK → アプリ内疑似着信画面 → 応答 → 疑似電話、の流れにする。

通話開始前に確認するもの:

- 匿名セッション確立済み
- `public.users.id` 解決済み
- 対象キャラ
- マイク権限
- コイン残高
- 通話開始元
- 別通話中でないこと

開始前に最低1課金単位分のコイン残高を確認する。正式な課金単位と開始に必要な最低残高はF1で決める。通話全体の前払いは行わない。

発話中からストリーミング文字起こしを進めるが、AIに渡すのは発話終了が確定した後の確定発話テキストだけ。暫定文字起こしはAIへ渡さず保存しない。

保存するのは確定発話テキスト。

Android実機ではWebViewの `getUserMedia()` に依存しない。

ネイティブ `AudioRecord` で 24kHz / mono / PCM16 を取得して `voice-turn` へ送る。

AIへ送らないもの。

- 無言
- 咳
- 物音
- 短すぎる音
- 意味のないノイズ
- TTS再生音の回り込み

無音判定時間はユーザーに数値を直接出さず5段階で選ばせる。

- かなり速い = 300ms
- 速い = 400ms
- 標準 = 500ms
- ゆっくり = 700ms
- かなりゆっくり = 900ms

AIには、キャラ基本情報、関係情報、ユーザー理解シート、同日短期文脈、直前LINE/通知文脈、現在の通話内履歴、今回の確定ユーザー発話を渡す。全ログを毎回全文で渡さない。

疑似電話中のAI応答はTTSでキャラ音声再生する。応答テキストは通話中に表示しない。通常応答は1〜3文程度。モーニング由来はさらに短め。

固定文言はAI応答ではなくアプリ側の状態文言であり、その再生時間は課金対象にしない。固定文言にTTSを使っても、AI通常回答でなければ課金対象にしない。

通話は1応答ごとの固定消費ではなく、同じ通話内の次の課金対象時間を合算して計算する。

- 各ユーザーターンの経過時間
- AIの通常回答音声が端末で実際に再生された時間

ユーザーターンは、アプリがそのターンの録音を開始してから、そのターンの終了を確定するまでとする。話し始めるまでの時間、実際に話している時間、話の途中で考えている沈黙、言いよどみ、発話終了判定までの時間、何も話さず無音・聞き取り失敗としてターンが終了するまでの時間を含む。

「無音を課金対象にする」とは、ユーザーターン中の沈黙時間をユーザーターンの経過時間に含めるという意味であり、STT、LLM、TTS、通信などの処理待ちを含める意味ではない。

AI側は、通常回答のキャラ音声が端末で実際に再生された時間だけを含める。再生開始前の時間、生成されたが再生されなかった音声は含めない。途中で再生を止めた場合は、実際に再生された部分だけを対象にする。

呼出中、接続前、WebSocket接続待ち、STT処理待ち、LLM回答生成待ち、TTS生成待ち、音声ファイル取得待ち、通信待ち、再接続待ち、アプリ側の処理遅延、固定の聞き返し音声、エラー案内音声、通話前の固定音声、Good morning onlyの固定音声は課金対象時間に含めない。

内部ではターンごとにユーザーターンの経過時間とAI通常回答音声の実再生時間を記録できる構造とする。通話終了時に同じ`call_id`の記録を合算し、1通話分の消費コインをまとめて確定する。通話中にターンごとに即時減算しない。

アプリ強制終了、異常切断、通信断などでも、保存済みの計測記録から未精算通話を確認・精算できる構造を要件とする。具体的な回収・再試行方法は後続の実装調査で決める。

通話終了時に、少なくとも今回消費したコイン数を確認できる構造にする。課金対象時間の内訳や秒数を利用者へ表示するかはF5で決める。

## A8-2. アプリ内疑似着信の接続と音

アプリ内疑似着信では、応答から会話開始までの待ち時間を最小にする。

- 着信画面の表示中に、通話接続の事前準備(WebSocket接続のstandby確立)を裏で進めてよい。ただし録音・課金・call作成は応答するまで開始しない。
- 拒否した場合、事前準備した接続は破棄し、DBに痕跡を残さない。
- 着信中は着信音を鳴らす。応答または拒否で即座に停止する。
- ユーザー発信の疑似電話では、発信開始から接続確立までコール音(呼び出し音)を鳴らす。
- 接続確立後、キャラの固定第一声(「もしもし」等)を再生してから会話を開始する。ユーザー発信・着信応答のどちらも同様とする。
- 第一声は毎回のTTS生成ではなく、キャラごとの固定音声とする。固定音声のため課金対象外(AI通常回答音声に含めない)。
- 固定第一声の音声素材の作成・保存は fixed_voice_assets 整備工程で行う。それまでは第一声なしで、接続と音の仕組みのみ先行実装する。
- 着信音・コール音は生成音声(TTS)を使わず、固定音源または合成音とする。
- 着信音・バイブは端末のマナーモード設定に従う。通常時は着信音+バイブ、マナー(バイブ)時はバイブのみ、サイレント時は無音とする。
- 着信音・コール音の本番音源はF6の音声素材工程で差し替える。

## A9. モーニング

モーニングは、キャラが日常に現れる主要導線。単なるアラームではなく、起床後に疑似電話へ進める入口である。

iOS 26以上ではAlarmKit導線を使う。CallKit / VoIP pushは使わない。カスタムAlarmKit音やアプリ内Web Audioループ音はv1対象外。

Androidはfull-screen intent notificationを第一候補にする。音はForeground Serviceで鳴らし、全画面表示と分離する。

full-screen intentが出ない端末では、音を鳴らしたまま通知を出し、通知タップで同じ疑似着信Activityへ入る。

応答後はMainActivityへ飛ばさない。

ロック解除を前提にした通常アプリ起動ではなく、ロック画面上の専用Activity内でAI通話接続状態へ進める。

AndroidはConnectionService / Telecomを使わない。

モーニングはアラーム系導線として、full-screen intent + Foreground Service + アプリ内疑似通話Activityで構成する。

## A8-4. Good morning only

Good morning only は固定音声のみで消費なし。

Talkで疑似電話へ進んだ後は、ユーザーターンの経過時間とAI通常回答音声の実再生時間を合算する通常の通話課金ルールを適用する。

# A9. 通知

## A10. 通知

v1で扱う通知:

- モーニングコール
- デイリー生成通知
- 気まぐれ通知
- イベント通知

モーニングは通常通知とは別枠。デイリー通知は最低1日1通AI生成。気まぐれ通知は回数上限を持つ。

通知タップ後の疑似LINE画面には、キャラからの入口メッセージを置く。

通知そのものは短くする。

- 入口メッセージは通知生成時に事前生成して保存する
- 通知文と入口メッセージはセットで生成・保存し、同じ文脈IDでつなげる
- 保存先は `notification_candidates` / `notification_logs`
- モーニングは入口メッセージの対象外

## A9-2. デイリー / 気まぐれ / lover通知

通知が送られる条件:

- 通知全体ON
- そのキャラの個別通知ON
- サイレント時間帯外
- 枠の割り当てあり
- 有効な端末通知トークンがある

同じ `public.users.id` に複数端末がぶら下がる可能性を持つ。通知送信時は `device_installations` を参照し、有効な端末へ送る。引き継ぎ後は旧端末を無効化するか、少なくとも新端末との二重通知を防ぐ。

## A11. イベント・告白

イベントは、親密度が上がったから発生するキャラ側からの変化。通知、疑似LINE、疑似電話を組み合わせる。

告白の基本パターン:

1. イベント通知
2. 疑似LINEに入口メッセージ
3. キャラが「少し電話してもいい？」
4. ユーザーがOK
5. アプリ内疑似着信画面
6. 応答
7. 告白用の短い疑似電話
8. ユーザー選択
9. 結果反映

最初から着信ではない。ユーザーがOKした後で疑似着信にする。

## A10-5. 告白の返答

選択肢。

- OK
- 断る
- まだ早い
- 友達でいたい

OK以外はloverにしない。

告白・関係変化・呼び方変更・強めのイベントは、ユーザーの明示同意を挟む。

- 告白: OK / 断る / まだ早い
- 関係状態変更: OK / 今のまま / 戻す
- 呼び方: OK / 今はやめる
- キャラが勝手に変更しない。同意前は `pending`
- 既読だけ・放置は同意ではない
- 未応答なら状態を戻す、または保留する
- しつこく迫らない

## A10-6. lover後

告白OKで関係状態 `lover`。

loverは通知上も会話上も文脈が変わる。

lover後の過去イベントはrememberableにする。

口調・態度・通知の温度・疑似電話やモーニングの一言は、関係状態を `lover` としてLLM文脈に渡すことで自動的に恋人トーンになる。

無料ユーザーでもlover化可能。恋人化を有料限定にしない。

AI会話継続は通常どおりコイン消費。

## A10-7. 告白の課金

告白の最初の固定一言はコイン消費しない。

本題以降にAI生成へ進む場合、通常どおりコイン消費。

告白イベントの最初の本題1往復を無料または固定音声に寄せるかはF章で決める。

# A11. 安全・コンテンツ方針

## A11-1. 基本路線

健全路線。性的表現・過激表現は禁止。

告白OKで関係状態 `lover`。lover後の過去イベントはrememberableにする。

無料ユーザーでもlover化可能。恋人化を有料限定にしない。AI会話継続は通常どおりコイン消費。

## A12. 安全・コンテンツ方針

健全路線。性的表現・過激表現は禁止。恋愛は情緒的・日常的な範囲まで。

禁止:

- 性的描写 / 性行為 / 露骨な誘惑
- 未成年との恋愛・性的文脈
- 自傷・他害の助長
- 違法行為の助長
- 現実の人物へのなりすまし
- 依存を強める過度な束縛・罪悪感誘導

通知文にセンシティブ情報を出さない。恋愛関係でも、ロック画面通知で過度に親密・恥ずかしい文言を出さない。

自傷・他害・医療・法的相談などはキャラ人格より安全テンプレートを優先する。緊急時は公的窓口・地域別ヘルプに誘導する。

---

# B. 課金設計

## B1. 通貨単位

アプリ内通貨はコイン。ユーザー向け表示はコインで統一する。

## B2. 消費しないもの

- デイリー通知のAI生成
- 通知文
- 入口メッセージ
- 固定・テンプレ通知
- モーニングのAlarmKit音
- Good morning onlyの固定音声
- 通話前の最初の固定一言
- 特別イベント最初の固定一言
- 聞き取り失敗・無音・雑音の固定文言
- ランダム猫のAI不使用反応
- 日次処理の要約・分類
- エラー文言

通話では、次の待ち時間・固定音声の再生時間を課金対象時間へ含めない。

- 呼出中・接続前
- WebSocket接続待ち・通信待ち・再接続待ち
- STT処理待ち・LLM回答生成待ち・TTS生成待ち
- 音声ファイル取得待ち・アプリ側の処理遅延
- 固定の聞き返し音声・エラー案内音声の再生時間
- 通話前の固定音声・Good morning onlyの固定音声の再生時間

無音・聞き取り失敗として確定するまでのユーザーターン経過時間は課金対象である。その後に流れる固定の聞き返し音声の再生時間は課金対象外であり、両者を混同しない。

## B3. 消費するもの

- 文字チャットでAIが返答した時の、AI返信1回ごとの固定消費
- 通話中の各ユーザーターンの経過時間
- 通話中のAI通常回答音声が端末で実際に再生された時間
- AI猫がユーザー入力に対して感情分類して鳴き声反応を返す時

チャットは文字数上限を前提とし、入力・出力トークン数による細かな従量計算は行わない。通話は1応答ごとの固定消費にせず、ターンごとに記録した課金対象時間を通話終了時に合算し、1通話分を一括精算する。

固定文言はLLMを呼ばないので消費しない。固定文言にTTSを使っても、AI通常回答でなければ再生時間を加算しない。TTS生成失敗・再生失敗、および生成されたが再生されなかったAI音声は課金対象にしない。

## B4. 無料ユーザー

無料ユーザーに提供するもの:

- 1AIメインキャラ
- ランダム猫
- モーニング導線
- デイリー通知最低1日1通
- 関係進行 / 告白イベント
- 毎日回復コイン

関係値進行や告白は有料限定にしない。AI会話量は無料回復コイン範囲に制限される。

## B5. 月額会員 / 定期コイン

サブスクは無制限ではなく、割安な定期コイン付与。

- 月額で一定数コインを付与
- 未使用コインの繰り越し上限を持つ
- サブスク限定で追加キャラ枠・通知上限・一部イベント演出を優遇する余地を残す

## B6. キャラ枠課金

AI利用キャラ枠は1枠=1キャラ固定。入れ替え制ではない。

- 無料ユーザーはAI利用可能なメインキャラを1つだけ選ぶ
- 別キャラでもAIを使いたい場合は追加AI利用キャラ枠を購入
- 追加枠は購入時にキャラを1人固定で割り当てる
- 後から別キャラへ入れ替える仕様にはしない
- ランダム猫はAIを使わないので無料付属・枠消費なし
- AI猫はAIを使うため、人間キャラと同じくAI利用キャラ枠を消費する

## B7. RevenueCat app user id

RevenueCat app user id は `public.users.id` とする。

使わないもの:

- `auth.users.id`
- 端末ID
- GoogleアカウントID
- AppleアカウントID
- email

Supabase Auth anonymous session の `auth.users.id` は認証セッションIDであり、RevenueCat app user idには使わない。

RevenueCat購入確認・Webhook・復元処理は、最終的に `public.users.id` に対してコイン付与・キャラ枠付与を行う。

## B8. 購入復元

購入復元はRevenueCatで確認する。ただし、アプリ内データの復元は `public.users.id` に戻れるかどうかに依存する。

原則:

- 引き継ぎコードがある場合、旧 `public.users.id` に戻す
- その後、RevenueCat app user id も同じ `public.users.id` で扱う
- 引き継ぎコードなしで再インストールした場合、新しい `public.users.id` が作られる
- その場合、購入状態はRevenueCat上で確認できても、旧キャラ記憶・関係値・チャット履歴・コイン残高は原則戻らない

消耗型コインは、復元で再付与しない。RevenueCatイベントID・store_transaction_id・idempotency_keyで二重付与を防ぐ。

非消耗キャラ枠・サブスクentitlementを現在の `public.users.id` へ自動反映するか、サポート確認にするかはF章で実装条件を決める。

## B9. 引き継ぎ失敗時の扱い

引き継ぎ失敗例:

- コード期限切れ
- コード使用済み
- 入力ミス
- 旧端末が手元にない
- RevenueCatでは購入済みだが、旧 `public.users.id` に戻れない

方針:

- 引き継ぎコードなしでは、旧キャラ記憶・関係値・チャット履歴・コイン残高は原則戻せない
- 購入済みの非消耗枠・サブスクはRevenueCatで確認し、現在の `public.users.id` に反映する余地を残す
- 消耗型コインは二重付与防止を優先する
- ユーザーに「引き継ぎコードなしでは会話履歴や記憶は戻せない」ことを明示する

---

# C. アカウント・引き継ぎ設計

## C1. 基本方針

ログイン画面を作らない。

初回起動時にSupabase Auth anonymous sessionを自動作成する。

アプリ内ユーザー主キーは `public.users.id`。

`auth.users.id` は `public.users.auth_user_id` にだけ紐づける。

## C2. 匿名セッション / Supabase client制約

起動時に既存セッションがなければ `signInAnonymously()` を実行する。

Supabase local config と本番Dashboardの両方で Anonymous sign-ins を有効化する必要がある。

匿名セッション作成に失敗した場合は、初期化エラーとして扱う。ログイン画面へ逃がさない。

Supabase client初期化時、env未設定なら空文字で初期化しない。明示的にErrorを出す。

`onAuthStateChange` 内ではDB queryを直接awaitしない。セッション状態を更新した後、別非同期処理で `public.users.id` 解決とユーザーデータロードを行う。

古いload結果で新しいセッション状態を上書きしない。load sequence等で最新の読み込みだけを反映する。

# C3. `auth.users.id` と `public.users.id`

`auth.users.id` は認証セッションID。

`public.users.id` はアプリ内ユーザーID。

解決経路:

```text
Auth session
  auth.users.id
    ↓ user_auth_links.auth_user_id または public.users.auth_user_id
  public.users.id
    ↓
  settings / coin_balances / user_characters / chat_messages / calls / user_character_memory / revenuecat_events / coin_transactions / coin_lots / coin_consumptions
```

各ユーザー別テーブルの `user_id` は `public.users.id` を参照する。

アプリ側にはservice role keyを置かない。

アプリ側はanon / publishable key + RLS前提で動かす。

service role相当の権限が必要な処理は、Edge Functionまたはsecurity-definer RPCへ閉じる。

# C4. 初回ユーザー作成

Authユーザー作成時triggerまたは初回データロードで自動作成するもの:

- `public.users`
- `public.settings`
- `public.coin_balances`

自動作成しないもの:

- `notification_preferences`
- `user_characters`
- `character_relationships`
- `user_character_memory`
- `device_installations`

これらは初回オンボーディングや端末登録時に作成する。

## C5. 端末インストール管理

`device_installations` を持つ。

目的:

- push token管理
- 端末別通知許可状態
- 同一ユーザーの複数端末管理
- 引き継ぎ後の二重通知防止
- 退会時の端末無効化

同じ `public.users.id` に複数端末が紐づく可能性を許容する。

## C6. 引き継ぎコード発行

旧端末で引き継ぎコードを発行する。

- 生コードはDBに保存しない
- DBには `code_hash` のみ保存
- 有効期限を持つ
- 単回使用
- 発行者の `auth.users.id` を保存
- 対象の `public.users.id` を保存
- 発行処理はEdge Functionまたはsecurity-definer RPC経由
- クライアントが `transfer_codes` を直接insert/select/updateしない

## C7. 引き継ぎコード入力

新端末で引き継ぎコードを入力する。

流れ:

1. 新端末で匿名セッション作成
2. 新しい仮 `public.users.id` が作成される
3. 引き継ぎコード入力
4. サーバ側でコード照合
5. 有効期限・使用済み・ハッシュ一致を確認
6. 旧 `public.users.id` に戻す
7. 新端末の `auth.users.id` を旧 `public.users.id` と紐づける
8. 仮ユーザーを無効化または削除候補にする

RLS上の現在ユーザー解決は、原則 `user_auth_links.auth_user_id = auth.uid()` かつ `is_current = true` を使う。

`users.auth_user_id` は現在代表リンクのキャッシュまたは単一現行セッション用の互換カラムとして扱う。矛盾時は `user_auth_links` を正とする。

引き継ぎ完了時は旧端末リンクを `is_current = false` にする。複数端末を許す場合でも、通知対象は `device_installations` で制御し、RLSのユーザー解決と通知送信先を混同しない。

## C8. 再インストール時

引き継ぎコードなしで再インストールした場合、新しい匿名ユーザーになる。

旧チャット履歴・記憶・関係値・コイン残高は原則戻せない。

購入状態はRevenueCatで確認できる場合があるが、アプリ内データ復元とは別問題。

## C9. 退会・削除

退会時に削除するもの:

- 会話
- 記憶
- 行動ログ
- 通知候補
- 端末トークン
- キャラ関係状態

保持または匿名化するもの:

- 購入ログ
- RevenueCatイベントログ
- 監査用の最低限ログ

退会後に同じ端末で起動した場合、新しい匿名ユーザーを作る。退会済みユーザーの引き継ぎコードは使えない。

---

# D. DB設計

## D0. DB全体方針

`public.users.id` がアプリ内user_id。

`public.users.auth_user_id` が `auth.users(id)` を参照する。

他テーブルの `user_id` は `auth.users(id)` ではなく `public.users(id)` を参照する。

古いmigrationに `relationships.user_id references auth.users(id)` のような記述が残っている場合は、v4方針と衝突するため整理対象。

RLSは `auth.uid()` から `public.users.id` を解決して判定する。

RLS上の現在ユーザー解決は、`user_auth_links.auth_user_id = auth.uid()` かつ `is_current = true` を正とする。

`users.auth_user_id` は、現在の代表リンクを参照するキャッシュまたは互換カラムとして扱う。

`users.auth_user_id` と `user_auth_links` が矛盾した場合は、`user_auth_links` を正とする。

引き継ぎ時は新しい `auth_user_id` のリンクをcurrent化し、旧リンクは必要に応じて `is_current = false` にする。

# D1. users

アプリ側ユーザー。

必要項目:

- id
- auth_user_id
- display_name
- created_at
- deleted_at
- deletion_status

`auth_user_id` は現在代表リンクのキャッシュ。履歴とRLS正本は `user_auth_links`。

## D2. user_auth_links

`public.users.id` と `auth.users.id` の履歴管理。

必要項目:

- id
- user_id
- auth_user_id
- link_reason: initial_anonymous / transfer_redeem / admin_repair / legacy_backfill
- linked_at
- unlinked_at
- is_current
- created_at

役割:

- 初回匿名Auth作成
- 引き継ぎ成功
- 過去auth_user_idの監査
- 誤紐づけ調査
- RLSでの現在ユーザー解決

`users.auth_user_id` と `user_auth_links` が矛盾する場合は、`user_auth_links` を正とする。

## D3. settings

RLS上の現在ユーザー判定は、`auth_user_id = auth.uid()` かつ `is_current = true` の行を使う。

引き継ぎ完了時は、新端末の `auth_user_id` をcurrent化する。旧端末のリンクは、複数端末を許容しない場合は `is_current = false` にし、複数端末を許容する場合でも通知対象は `device_installations` 側で制御する。

# D3. settings

通知ジョブが参照する配送用設定は `notification_preferences` に置くが、timezone / quiet hours / snooze など重複しやすい値は原則 `settings` を正とする。`notification_preferences` は通知送信用のキャッシュまたは派生設定として扱い、差分がある場合は `settings` を優先する。

アプリ全体設定のSource of Truthは `settings` とする。

`timezone`、quiet hours、snooze、morning設定、notification ON/OFF は `settings` を正とする。

必要項目。

- user_id
- language
- timezone
- notification_global_enabled
- quiet_hours_start
- quiet_hours_end
- morning_alarm_enabled
- morning_alarm_time
- default_snooze_minutes
- silence_end_ms: 300 / 400 / 500 / 700 / 900

## D4. coin_balances

ユーザーの現在残高。Source of Truth。

必要項目:

- user_id
- balance
- updated_at

## D5. coin_transactions

コインの増減履歴。

必要項目:

- id
- user_id
- direction: grant / consume / refund / adjust / expire
- amount
- reason_type
- reason_ref_id
- idempotency_key
- created_at

## D6. coin_lots

有効期限管理のためのロット。

必要項目:

- id
- user_id
- source_type: purchase / subscription / daily_free / promo
- source_ref_id
- initial_amount
- remaining_amount
- granted_at
- expires_at
- revenuecat_event_id
- store_transaction_id
- product_id

消費時は期限が近いロットから減らす。RevenueCat連携は冪等。

## D7. coin_consumptions

チャット・AI猫の応答単位、および通話終了時の1通話単位の消費明細。デバッグ・返金・誤課金防止。

必要項目:

- id
- user_id
- character_id
- interaction_type: chat / call / ai_cat
- source_message_id / source_call_id / source_call_turn_id
- consumed_amount
- triggered_by: tts_audio_started / call_settled / ai_cat_response_selected
- status: pending / consumed / refunded / skipped
- skip_reason: no_audio / fixed_text / stt_only / insufficient_balance / error
- idempotency_key
- created_at

通話は同じ`call_id`に対して一括精算し、再試行時も二重精算しないidempotency_keyを持つ。

## D8. revenuecat_events

RevenueCat Webhook / purchase restore / entitlement確認の監査テーブル。

必要項目:

- id
- user_id
- revenuecat_app_user_id
- revenuecat_event_id
- store_transaction_id
- product_id
- event_type
- entitlement_id
- processed_at
- raw_payload_json
- idempotency_key

目的:

- Webhook重複防止
- restore時の二重付与防止
- 購入監査
- 消耗/非消耗/サブスクの処理分離

## D9. characters

キャラマスタ。

必要項目:

- id
- default_name
- voice_label
- type: human / random_cat / ai_cat
- gender
- persona_json
- voice_model_id
- visual_asset_id
- is_active

## D10. user_characters

ユーザーが保有/表示しているキャラ。AIメイン枠・通知ON/OFFもここで管理。

必要項目:

- user_id
- character_id
- is_ai_enabled
- is_visible
- notifications_enabled
- acquisition_type: free_main / purchased_slot / included_random_cat / promo / initial_pool
- acquired_at
- ai_enabled_at
- ai_disabled_at
- display_order

## D11. character_relationships

ユーザー×キャラの関係状態。点数・状態・判定用。

必要項目:

- user_id
- character_id
- intimacy_score
- user_affinity_level
- relationship_state: friend / close_friend / romance_event_pending / lover
- romance_event_allowed
- nickname_current
- last_interaction_at
- last_decay_at
- created_at

# D12. user_character_memory

ユーザー×キャラの現在の理解シート / 関係メモ。

必要項目。

- user_id
- character_id
- profile_json
- relationship_json
- preference_json
- memory_json
- safety_json
- updated_at

役割。

- キャラごとに1行
- AIへ渡すユーザー理解シート保存先
- `character_relationships` とは役割分離
  - `character_relationships` = 点数・状態・判定
  - `user_character_memory` = AIへ渡す要約・理解シート

# D13. chat_messages

疑似LINE履歴。通話見出しもここに差し込む。

必要項目:

- id
- user_id
- character_id
- sender: user / character / system
- body
- message_type: text / call_summary_header / event_intro / system_notice / cat_reaction
- source
- source_call_id
- source_notification_log_id
- turn_id
- reply_to_message_id
- generation_status
- pinned
- expires_at
- created_at

# D14. calls

通話単位の親。疑似電話履歴の見出し。

必要項目:

- id
- user_id
- character_id
- source: user_initiated / morning / special_event
- started_at
- ended_at
- duration_sec
- billable_duration_ms
- consumed_coin_id
- settlement_status
- settled_at
- end_reason
- status: active / completed / failed / cancelled
- chat_header_message_id
- summary_id
- pinned
- expires_at

# D15. call_logs

通話の往復ログ。日中の文脈・要約材料。

必要項目:

- id
- call_id
- turn_index
- user_transcript
- assistant_text
- assistant_audio_url
- stt_confidence
- vad_end_reason
- generated_at
- audio_started_at
- user_turn_started_at
- user_turn_ended_at
- user_turn_elapsed_ms
- assistant_audio_played_ms
- should_delete_after_daily_processing
- deleted_at

# D16. fixed_voice_assets

固定音声。

必要項目:

- id
- character_id
- use_case: morning_greeting / call_opening / call_fixed_error / call_end / special_event_opening / cat_meow
- relationship_state
- text
- audio_url
- voice_model_id
- generated_by
- license_info
- created_at

# D17. notification_preferences

通知ジョブ用設定またはキャッシュ。

通知処理で参照しやすい形にした補助テーブルとして扱う。

`timezone`、quiet hours、snooze、morning設定、notification ON/OFF が `settings` と重複する場合は、`settings` を正とする。

`notification_preferences` は通知ジョブ用のキャッシュまたは派生設定であり、ユーザー操作設定のSource of Truthにはしない。

必要項目:

- user_id
- notifications_enabled
- quiet_hours_start
- quiet_hours_end
- morning_enabled
- default_morning_time
- timezone
- snooze_minutes

# D18. notification_candidates

通知候補。

必要項目:

- id
- user_id
- character_id
- notification_type: daily / random / event
- scheduled_for
- title
- body
- entry_message_body
- source_context_ref
- used_topic_ref
- generation_status
- created_at

# D19. notification_logs

実際に送った通知。

必要項目:

- id
- user_id
- character_id
- candidate_id
- device_installation_id
- sent_at
- opened_at
- outcome: ignored / opened_chat / opened_call / dismissed

# D20. device_installations

端末/インストール単位の管理。

必要項目:

- id
- user_id
- auth_user_id
- platform: ios / android
- push_token
- installation_id
- app_version
- os_version
- notifications_permission_status
- is_active
- last_seen_at
- created_at
- disabled_at

用途:

- push token管理
- 旧端末/新端末の二重通知防止
- 引き継ぎ後の新端末有効化
- 旧端末無効化
- モーニング再スケジュール判定

# D21. morning_alarms

モーニングのスケジュール。理解シートには置かない。

必要項目:

- id
- user_id
- character_id
- device_installation_id
- platform: ios / android
- enabled
- local_time
- timezone
- repeat_rule
- label
- platform_alarm_id
- last_scheduled_at
- last_triggered_at
- snooze_until
- created_at / updated_at

# D22. event_states

イベント状態ログ。

必要項目:

- id
- user_id
- character_id
- event_type
- event_status: pending / shown / responded / completed / declined / expired / cancelled
- source_message_id
- source_call_id
- payload
- triggered_at / responded_at / expired_at
- created_at

# D23. call_summaries

通話日次処理後にAI用へ残すもの。

必要項目。

- id
- call_id
- user_id
- character_id
- summary_text
- relationship_delta
- memory_candidates_json
- safety_flags_json
- created_at
- expires_at

# D24. daily_processing_runs

日次処理実行状態。

必要項目:

- id
- user_id
- processing_date
- started_at
- finished_at
- status: running / completed / failed
- input_window_start
- input_window_end
- chat_messages_processed_count
- call_logs_processed_count
- call_summaries_created_count
- memory_updates_count
- error_message

# D25. transfer_codes

匿名サインイン利用者が端末変更・再インストール時に同じ `public.users.id` へ戻るための短期コード管理。

必要項目:

- id
- user_id
- code_hash
- created_by_auth_user_id
- expires_at
- used_at
- used_by_auth_user_id
- attempt_count
- revoked_at
- created_at

運用:

- 生コードはDBに保存しない
- クライアントは直接insert/select/updateしない
- 発行・引き換えはEdge Functionまたはsecurity-definer RPC
- RLSは直接アクセス拒否を基本にする

# D26. account_deletion_audit

退会・削除・匿名化の監査。

必要項目:

- id
- user_id
- deletion_requested_at
- logical_deleted_at
- physical_deleted_at
- purchase_logs_anonymized_at
- status
- reason
- created_at

---

# E. 記憶設計

## E1. 3層の寿命

1. 画面履歴

- チャット本文
- 通話見出し
- 90日で自動削除
- 重要なものはピン留めで残せる

2. 当日処理用の短期ログ

- AIに渡す現在の文脈と日次処理の材料
- 通話ターン本文は日次処理まで保持
- 同日短期文脈専用テーブルは作らない

3. 長期記憶

- キャラがユーザーを理解するための要約・状態
- `user_character_memory` に保存
- 恒久保存

長期記憶例:

- 呼び方
- 好き嫌い
- 生活リズム
- 重要イベント
- 関係状態
- 禁止事項
- 安全フラグ

## E2. 通話ログ保存

通話中は `call_logs` に確定テキストとAI応答テキストを保存する。

音声ファイル保存は必須ではない。

課金・誤課金防止のため、各ターンの録音開始から終了確定までの経過時間と、AI通常回答音声の実再生時間を保存する。固定の聞き返し音声・エラー案内音声の再生時間はAI通常回答音声の実再生時間へ含めない。

通話終了時に同じ`call_id`の保存済み計測記録を合算して一括精算する。強制終了・異常切断・通信断で終了処理が完了しなくても、未精算通話を確認して回収・再試行でき、二重精算しない構造にする。

日次処理後に `call_summaries` / `user_character_memory` に反映し、必要なら `call_logs` の本文は削除する。

## E3. 日次処理

対象:

- その日の日次処理ウィンドウ内のchat_messages
- call_logs
- call_summaries未作成のcalls
- event_states
- notification_logs

出力:

- call_summaries
- user_character_memoryの更新
- relationship/intimacy補正候補
- 翌日以降の通知材料

日次処理でやらない:

- 次の日の通知文を全部確定
- 長期記憶の間引き
- 自動のキャラ関係変化確定
- 自動恋人化

## E4. AIに渡す文脈

毎回固定で渡す:

- キャラpersona
- 関係状態
- 呼び方
- ユーザー理解シートの要約

状況で渡す:

- 同日短期文脈
- 直近の疑似LINE文脈
- 通知/イベント文脈
- 今の通話内の直近履歴
- 関係変化/告白pending状態

渡さない:

- 全履歴全文
- 古いログ全部
- 不要なセンシティブ情報

## E5. 長期記憶更新

AIに自由に記憶を決めさせない。

AIは候補抽出・分類だけ。

サーバ側ルールで採用/不採用を決める。

ユーザーに見せる/消す機能はv1では最小。v2で詳細管理。

---

# F. 開発中に決める数値・検証項目

F章に残すのは、今決めなくても実装順が崩れないものだけ。

## F1. コイン数値

- 無料回復量
- 月額付与量
- チャット1返信あたり固定消費量
- 通話の時間単位
- 通話の1単位あたり消費量
- 通話の最低消費
- 通話時間の端数処理
- 通話開始に必要な最低残高
- 商品価格
- 期限切れ
- 繰り越し上限

## F2. 通話安全上限

- 最大STT受付秒数
- 無音打ち切り秒数
- 聞き取り失敗回数
- 最大通話時間
- 1通話あたり最大消費量

## F3. 関係値数値

- 親密度の増減量
- 告白発生閾値
- 呼び方解禁閾値
- 通知重み
- 下限・上限

## F4. 記憶の日数・遅延

- チャット本文/通話見出しの90日保持は暫定。実データ量で調整
- 通話ターン本文は日次処理後削除候補。日次処理失敗時の保持上限
- call_summariesの保持日数

## F5. 課金細部

- 月額コインの繰り越し上限
- サブスク終了後の追加AI枠扱い
- 無料メインキャラ変更頻度
- コイン不足時の文言・導線
- 通話終了時に表示する内容
- 課金対象時間の内訳・秒数を利用者へ表示するか
- 通話中の残高不足予告と終了タイミング
- 未精算通話の回収・再試行方針
- アプリ強制終了・異常切断・通信断時の精算方法
- 引き継ぎコードなし再インストール時に、非消耗キャラ枠を自動で現在ユーザーへ反映するか、サポート確認にするか
- サブスクentitlementを現在ユーザーへ自動反映する条件
- 旧 `public.users.id` へ戻れない場合の購入復元UI文言

## F6. 素材・音声

- 声モデルの商用利用可否
- アプリ同梱・事前生成可否
- v1仮採用声モデルのTTS品質
- キャラ画像の素材方針
- 猫の鳴き声素材
- 固定音声の具体作成

## F7. 実機検証

- 匿名初回起動
- 再起動
- 再インストール
- 引き継ぎコード発行/入力
- 旧端末二重通知防止
- RevenueCat購入復元
- iOS AlarmKit本番導線
- Android full-screen intent本番導線
- Android full-screen intentのメーカー別動作（不許可時のフォールバック含む）
- Android 14以降の `USE_FULL_SCREEN_INTENT` 実効付与
- メーカー自動起動/バックグラウンド起動制限下でのモーニング起動
- モーニングから本番AI通話開始
- STTストリーミング
- VAD
- TTS再生
- 残高不足
- 二重消費なし
- 退会

---

# G. v2以降候補

- 任意バックアップ用の外部アカウント連携
- Realtime API全面移行
- ベクトル検索
- 多言語展開
- カレンダー連携
- Live2D / アバターアニメーション
- キャラ衣装・表情差分
- 画像送受信
- ユーザー画像アップロード / ユーザー独自キャラ
- チャット履歴ダウンロード
- 安全設定の詳細管理
- 記憶編集
- 関係リセット / キャラごとの距離感設定
- 通知AI生成の高度化
- 猫の本格アニメーション
- 子ども系キャラ
- 家族系関係

---

# H. 検証済み前提

## H1. 声モデル

AivisHub上で商用利用可能なACML 1.0モデルを確認済み。

v1仮採用:

- 女性: まお / コハク / 桜音
- 男性: にせ / fumifumi / 猩々博士

## H2. iOSモーニング

iOS AlarmKit検証済み。カスタムAlarmKit音やアプリ内Web Audioループ音はv1対象外。

## H3. Androidモーニング

Android実機のPhase0検証（テスト機1台）で、full-screen intentによるロック画面上の疑似着信表示、疑似着信音、応答後のロック解除なしAI通話接続中画面への遷移までは成立。音はActivityではなくForeground Service側で鳴らし、全画面表示と音を分離する方針。

ただしこれは1機種でのPhase0結果であり、全メーカーでの動作は未確認。Android 14以降の `USE_FULL_SCREEN_INTENT` 権限が実効的に付与されない場合や、メーカー独自の自動起動・バックグラウンド起動制限により、full-screen intentが出ず通知バナーのみ・無音になる端末が存在しうる。

方針:

モーニングは特定メーカーだけで成立すればよいものではない。

メーカー差で全画面が出ないことを「許容範囲」で済ませない。

full-screen intentの実効付与とメーカー別の自動起動制限は、本番前の要検証・未解決事項として扱い、F7で検証する。

## H4. 疑似電話TTFA

Android実機で STT → LLM → Aivis TTS → 音声再生まで成立。TTFAは約2秒前後で確認。

---

# I. v4.0で削除したもの

- Googleログイン
- Appleログイン
- email/passwordログイン
- OAuth callback deep link
- SocialLogin依存
- ログイン画面
- Google client ID前提の本番ログイン
- Apple Sign In capability前提の本番ログイン
- `auth.users.id` を各テーブルの `user_id` として直接使う設計
- `RevenueCat app user id = auth.users.id` の設計
- 生の引き継ぎコードをDB保存する設計
- クライアントが `transfer_codes` を直接操作する設計

---

# J. 整合チェックリスト

- [ ] ログイン画面を作る記述が残っていない
- [ ] Google / Apple / emailログインを本番導線にする記述が残っていない
- [ ] OAuth callbackを本番導線にする記述が残っていない
- [ ] SocialLogin依存が残っていない
- [ ] 初回起動は匿名セッション自動作成になっている
- [ ] アプリ内ユーザー主キーは `public.users.id` に統一されている
- [ ] `auth.users.id` を各テーブルの `user_id` として直接使っていない
- [ ] RevenueCat app user id は `public.users.id` に統一されている
- [ ] `user_character_memory` のDB定義がある
- [ ] `character_relationships` と `user_character_memory` の役割が分かれている
- [ ] service role keyをアプリ側に置かない方針が本文にある
- [ ] Supabase clientをenv未設定の空文字で初期化しない
- [ ] `onAuthStateChange` 内でDB queryを直接awaitしない
- [ ] RLSの現在ユーザー解決が `user_auth_links.is_current = true` 基準で明確になっている
- [ ] settings と notification_preferences のSource of Truthが明確である
- [ ] 引き継ぎコードなしで完全復元できるような誤解を招く記述がない
- [ ] `transfer_codes` は生コード非保存になっている
- [ ] `transfer_codes` はRPC/Edge Function経由になっている
- [ ] `device_installations` が通知・端末管理に使われている
- [ ] 旧端末・新端末の二重通知防止が書かれている
- [ ] 購入復元とアプリ内データ復元が分離されている
- [ ] 消耗型コインの二重付与防止が書かれている
- [ ] 退会時の会話・記憶削除と購入ログ匿名化が分離されている
- [ ] 今決めるべき未決がF章に逃げていない
- [ ] F章に残っているのは開発中・実測後で決めてよい数値だけである

---

# K. レビュー指示

```text
まだ既存MDを上書きしないでください。
commit、push、PR作成も禁止です。レビューだけしてください。

対象:
- docs/voice_companion_spec.md
- docs/voice_companion_spec_v4_draft.md
- docs/voice_companion_build_plan.md

目的:
voice_companion_spec_v4_draft.md が、現在のVoiceCompanion仕様をログイン画面なし・Supabase anonymous session方式へ正しく再構成できているかレビューしてください。

重点確認:
1. user_character_memory がDB定義に存在するか。
2. service role keyをアプリ側に置かない方針が本文にあるか。
3. Supabase client初期化時のenv未設定Errorと、onAuthStateChange内await禁止があるか。
4. user_auth_links導入後のRLS current user解決が明確か。
5. settings と notification_preferences の責務が明確か。
6. Google/Apple/email/OAuth/SocialLogin/ログイン画面の本番導線が残っていないか。
7. build_plan.mdを次にどう直すべきか。

出力:
- 結論
- 重大問題
- 軽微問題
- 欠落仕様
- 削除漏れ
- build_plan修正点
- 差し替え前に必ず直すべき最小修正案
```
