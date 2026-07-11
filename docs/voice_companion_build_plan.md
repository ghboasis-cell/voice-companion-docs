# VoiceCompanion 作業手順書 兼 運用ルール

**版数: v4.4 ／ 最終更新日: 2026-07-11**

（2026-07-11 staging(voice-companion-staging)の Anonymous sign-ins を有効化。理由: RLS / `current_app_user_id()` 検証のため。本番挙動に合わせ有効のまま維持。）

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
- 仕様書と本書は、対応が分かるように管理する。両者を常に同じ版数にはしない。**片方だけ内容が変わった場合は、変わった方だけ版数を上げ、変更履歴で対応関係を示す。** 中身が変わっていないファイルの版数を、同期目的だけで上げない。現在の対象仕様は `docs/voice_companion_spec.md` v4.3 正本。

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

## 1-5. AI・外部に渡す時
- 必ず仕様書と本書をセットで前提にさせる。旧メモ・旧specは渡さない・参照させない。
- 一問一答を使うのは、未確定仕様、仕様変更、specとbuild planの矛盾、ユーザー判断が必要な選択、課金・権限・本番運用などの重大判断に限る。
- 正本MDで決定済みの内容は、Codexが利用者から見て一つの機能として成立する単位の実装、必要な修正、自動テスト、ビルド、commit、push、PR作成を連続して進めてよい。途中で「続けてよいか」「テストするか」「commitするか」「PRを作るか」の確認は求めない。
- ユーザーが確認するのは、原則としてPRレビュー後のマージ、本番反映が必要な場合の操作、最終的な実機確認だけとする。実機確認で問題が出た場合は、その機能の範囲内で修正して再確認する。
- 「ボタン1個」など過度に細かく分割せず、利用者が一つの機能として確認できる単位で区切る。
- 推測で仕様を補わない。仕様判断が必要になった場合は作業を停止して報告する。
- 用語を勝手に増やさない（過去に「システム案内」「接続中」等で混乱した経緯）。

## 1-6. Supabase運用
- migration作成とローカル検証は、機能単位の作業内でCodexが進めてよい。
- 本番Supabase反映が必要な場合は、その機能のPRレビュー・マージ後に行い、反映後に必要な実機確認を行う。
- 本番データの破壊的変更は自動実行しない。
- migrationで管理できる変更をSupabase Dashboardで直接行わない。
- Anonymous sign-insなどDashboard設定変更は、実施内容を本書へ記録する。

## 検証環境 (staging)
- staging プロジェクト: voice-companion-staging (ref: <staging-ref>, Seoul)
  - 無料プロジェクト。追加料金なし。7日無操作で自動停止(使う時に手動再開)
  - 用途: スキーマ変更・migration を本番前に検証するための使い捨て環境
- 本番プロジェクト: voice-companion (ref: <prod-ref>, Tokyo)
- 2026-07-11: staging の Anonymous sign-ins を有効化した。理由: RLS / `current_app_user_id()` 検証のため。本番挙動に合わせ、有効のまま維持する。

### 検証フロー
1. staging に link 切り替え: ! supabase link --project-ref <staging-ref>
2. supabase db push で migration を staging に適用
3. supabase inspect db table-stats --linked でテーブル数を確認(本番と一致するか)
4. 問題なければ本番に link を戻して反映

### 厳守
- link 切り替えは必ずユーザーが ! 付きで実行(パスワード管理のため)
- db push 前に必ず link 先が staging か本番かを確認する
- 本番(<prod-ref>)への push は明示的な指示があるときのみ
- supabase start は使わない(ローカルDocker禁止)

## 1-7. モデル運用
- 通常実装はTerraを使う。
- 認証、RLS、課金、引き継ぎ、重大な設計判断、原因不明の問題、機能完了時レビューはSolを使う。
- 検索、ログ整理、文書整形などの軽作業はLunaを使う。

---

# 第2部. 作業手順

## 前提の考え方
主要な設計判断は仕様書で確定済み。ただし「着手前に潰す前提」があり、ここが崩れる場合は設計を見直す。アプリ完成まで完全放置で一気に作らず、利用者から見て一つの機能として成立する単位ごとに進める。各機能の中はCodexが実装、必要な修正、自動テスト、ビルド、commit、push、PR作成まで連続して進める。機能の実機確認で問題が見つかった場合は、その機能の範囲内で修正して再確認する。

---

## フェーズ0-前. 正本仕様の確定
- [x] `docs/voice_companion_spec.md` v4.3 を正本として扱う。
- [x] v1本番導線から Google / Apple / email/passwordログイン、ログイン画面、OAuth callback deep link、SocialLogin依存、Google client ID前提、Apple Sign In capability前提を削除済みとして扱う。
- [x] `RevenueCat app user id = auth.users.id` と `auth.users.id` をアプリ内ユーザー主キーにする設計は廃止済み。
- [x] email/password は過去のデバッグ疎通実績としてのみ扱い、本番実装タスクに戻さない。
- [~] v4.3仕様に沿った実装、RLS実動作確認、実機検証は未完了として扱う。PR #28でproduction UI土台と初回起動導線の一部はmerge済み。PR #29後の実機確認で、初回オンボーディング導線は最低限成立していることを確認済み。ただしPhase1全体は未完了。

## フェーズ0. 着手前に潰す前提（最優先・ここが崩れたら設計見直し）

- [x] **声モデルの確保（最優先）**
  完了条件: Aivisで商用可能（ACML または CC0。ACML-NC不可）かつ満足できる声を、必要数見つける。事前生成音声をアプリに保存・同梱してよいか、選定モデルの条件も確認。
  結果: AivisHub上でACML 1.0の候補を確認し、v1仮採用モデルを男女3ずつ決定。ライセンス面ではblockerではない。
  仮採用: 女性 = まお / コハク / 桜音。男性 = にせ / fumifumi / 猩々博士。
  注意: アプリ内キャラ名は声モデル名と分離してよい。実在人物系モデルは本人・公式アプリと誤認させない。
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
  - 残り: アプリ側で `current_app_user_id()` 経由により `public.users.id` を正しく取得できるかの追加確認、引き継ぎコード発行/入力、引き継ぎ用 RPC または Edge Function 設計と実装。
- [~] DBテーブル構築 / v4.3 Auth整合確認（RLS・制約・インデックス）
  - 完了: migration作成・PR #25 merge・local `supabase db reset` による `20260706090000_init_voice_companion_phase1_schema.sql`、`20260707140000_add_transfer_codes.sql`、`20260708090000_align_v4_auth_rls.sql` の適用確認は完了。
  - ただし上記は「migration作成・merge・local db reset確認」の完了であり、「Phase1 DB/RLS全体完了」ではない。
  - PR #28で完了: `20260708120000_add_onboarding_profile_and_cats.sql` を追加し、`public.users` に `family_name` / `given_name` / `family_name_kana` / `given_name_kana` を追加、`public.users` の更新権限をオンボーディング用プロフィール項目に限定、production UI土台で使う仮キャラ8件（人間6人 + AI猫 + ランダム猫）をidempotentにseedする形に更新済み。
  - PR #28で完了: 初回名前入力、初回キャラ選択、初回好意度/第一印象選択、`user_characters` / `character_relationships` 初期行作成のクライアント実装を追加済み。
  - 本番反映済み: `20260706090000_init_voice_companion_phase1_schema.sql`、`20260707140000_add_transfer_codes.sql`、`20260708090000_align_v4_auth_rls.sql`、`20260708120000_add_onboarding_profile_and_cats.sql`、`20260709000000_add_deletion_audit_and_schema_gaps.sql` は本番Supabaseへ `supabase db push` 済み。
  - PR #29後の実機確認: 本番Supabase Dashboardで Anonymous sign-ins をONにし、`Anonymous sign-ins are disabled` エラーが解消したことを確認済み。実機で名前入力 → キャラ選択 → 初回関係選択 → トーク画面と思われる画面まで遷移できた。
  - stagingで確認済み（2026-07-11・PASS）: 匿名セッションから `current_app_user_id()` が本人の `public.users.id` を正しく返し（`user_auth_links` の `is_current=true` は1件のみ）、RLSにより他ユーザーの `settings` / `coin_balances` / `user_characters` / `chat_messages` 行が読めない・フィルタ指定でも取得できないことを確認（漏れなし）。`chat_messages` は本人 `public.users.id` でのinsert成功と、本人のみ読める・別ユーザーから読めないことも確認。anon key + 匿名JWTのみで実施（service_role不使用）、RLSが実際に効いた状態。テストユーザー/データは削除済み。なおこれはstagingのRLS/`current_app_user_id()`動作確認であり、migrationを`db push`して本番前検証する作業ではない（stagingスキーマは既存前提）。
  - 未完了: 本番(voice-companion)での同じRLS実動作・`current_app_user_id()` による本番 `public.users.id` 解決の確認。
  - 未完了: 引き継ぎコード発行/入力のRPCまたはEdge Function設計と実装。
  - RevenueCat連携は `app_user_id = public.users.id` 前提に揃える。`auth.users.id` をRevenueCat app user idにする設計は廃止済み。
  - したがって「DB土台、PR #25 migration、PR #28 migration/クライアント土台、本番Supabaseへの指定migration反映は完了済み」だが「v4.3 DB/RLS完了」ではない。
- [~] アプリ骨組み（Capacitor + Vite + TS）／ナビゲーション
  - PR #28で完了: production UI土台、起動/準備中表示、疑似LINEホーム、下部ナビ、設定画面、利用規約仮ページ、プライバシーポリシー仮ページ、仮モーダル、2テーマ切替（DB保存なし・localStorage保存）を追加済み。
  - 未完了: 本格ナビゲーション設計、実機UI検証、通知/モーニング/通話/課金/引き継ぎ/退会など本番機能画面への接続。
- [~] 疑似LINEチャット画面（`chat_messages` の表示・送信。通知タップ後の入口メッセージ、通話見出しの差し込み枠も）
  - PR #28で完了: ホームから選択済みキャラの仮チャット画面へ遷移する土台を追加済み。ホームではメッセージ本文を表示せず、状態表示のみとする方針を反映済み。
  - 未完了: `chat_messages` のDB保存/表示、AI応答、コイン消費、通知タップ後の入口メッセージ、通話見出し差し込み。
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
- Phase1全体は未完了。PR #29後に本番Supabase Dashboardで Anonymous sign-ins をONにし、`Anonymous sign-ins are disabled` エラー解消と、実機で名前入力 → キャラ選択 → 初回関係選択 → トーク画面と思われる画面までの遷移は確認済み。ただしRLS実動作の詳細確認、本格AIチャット/通話/通知/課金/引き継ぎ/退会は未完了。
- Supabase Auth は匿名セッション自動作成へ切り替える。ログイン画面は出さず、`loadUserData()` で `public.users.id` を解決する。PR #28でこの前提のクライアント導線は追加済みだが、実環境確認は未完了。
- 次作業は、既存の第2部の機能順序に従い、利用者から見て一つの機能として成立する単位で開始する。各機能についてCodexは実装、必要な修正、自動テスト、ビルド、commit、push、PR作成まで連続して進める。本番反映が必要な場合はPRレビュー・マージ後に行い、必要な実機確認で問題が出た場合は同じ機能の範囲内で修正して再確認する。
- 引き継ぎコード発行/入力は、RPCまたはEdge Function設計と実装が未完了。
- DB詳細は `voice_companion_spec.md` C章を正とする。

PR #28/#29後の整理:

- 実施済み: PR #28はmerge済み。production UI土台、起動 / 準備中画面、初回名前入力画面、キャラ選択画面、キャラ名 + 初回関係設定画面、`user_characters` / `character_relationships` 初期行作成処理、疑似LINEホーム仮画面、疑似LINEチャット仮画面、キャラ管理仮画面、設定画面、テーマ切替、利用規約 / プライバシーポリシー仮ページを追加済み。
- 実施済み: migration `20260708120000_add_onboarding_profile_and_cats.sql` を追加済み。`public.users` に `family_name` / `given_name` / `family_name_kana` / `given_name_kana` を追加し、`public.characters` に仮キャラ8件をseedし、`public.users` の更新権限をオンボーディングプロフィール項目に限定する。
- PR #29で整理すること: 上記production onboarding UI / 画面構成 / 画面遷移 / テーマ仕様を `docs/voice_companion_spec.md` の正本仕様へ反映し、本書では「UI土台は入った」と「本番機能完成」を分けて記録する。
- 未完了: 本格AIチャット、`chat_messages` のDB保存/表示、AI応答、コイン消費、本格通話、コール画面、通話中画面、通話終了画面、モーニングコール本実装、通知送信、RevenueCat課金、引き継ぎコード発行 / 入力、引き継ぎ用 RPC または Edge Function、退会処理。
- PR #29後の実機確認結果: Supabase Dashboardで Anonymous sign-ins をONにした。これにより `Anonymous sign-ins are disabled` エラーは解消。実機で名前入力 → キャラ選択 → 初回関係選択 → トーク画面と思われる画面まで遷移できたため、PR #29時点の初回オンボーディング導線は最低限成立している。
- 注意: 上記は完成版の本番デザイン確認ではない。現状はフォーム中心の仮UIであり、本番デザイン作り込み、本格AIチャット、通話、通知、課金、引き継ぎ、退会は未完了。
- 本番反映済み: 指定migration 5件は本番Supabaseへ `supabase db push` 済み。
- stagingで確認済み（2026-07-11・PASS）: RLS実動作と `current_app_user_id()` による `public.users.id` 解決を staging で検証し漏れなし（詳細はフェーズ1「DBテーブル構築 / v4.3 Auth整合確認」参照）。
- 未完了: 本番(voice-companion)でのRLS実動作・`current_app_user_id()` が本番で正しい `public.users.id` を返す確認、完成版の本番デザイン確認。

## 機能単位の自立型開発運用

第2部の既存の機能順序とチェック状態を維持し、利用者から見て一つの機能として確認できる単位で進める。自動テスト完了だけで機能を `[x]` にせず、当該機能に本番反映または実機確認が必要な場合は、それらが完了してから `[x]` にする。Codexは、正本MDで決定済みの機能について途中の細かい承認を求めず、実装、必要な修正、自動テスト、ビルド、commit、push、PR作成まで連続して進める。

停止して報告するのは、MDに答えがなく勝手に決めると製品仕様が変わる場合、または認証、RLS、課金、本番データの破壊的変更など重大な安全確認が必要な場合だけとする。

## フェーズ2. 前提確認後に決める数値（Z-1〜Z-3）

- [ ] コイン数値: 無料の毎日回復量 / 月額付与 / 各商品価格 / 1応答あたり消費 等
- [ ] 疑似電話の安全上限: 最大STT受付秒数 / 無音打ち切り / 聞き取り失敗回数 等
- [ ] 告白到達の関係値水準 / 未応答の下げ幅 / 拒否打ち切り回数

## フェーズ3. 本格実装（依存順）

- [ ] 記憶パイプライン（D）: チャット/通話ログ保存 → 日次処理（夜間バッチ）→ 長期記憶抽出 → AIに渡す組み立て（同日短期文脈は既存から組み立て）
- [ ] 課金（B）: RevenueCat app user id は `public.users.id` を使う。RevenueCat購入確認 → Supabaseでコイン付与 → 消費（idempotency_key で二重消費防止）→ 残高不足時の挙動
- [ ] 疑似電話（A5）: STT→LLM→TTS、固定文言の原因分岐課金、通話2階建てログ、アプリ内疑似着信画面
- [ ] 通知（A10）: デイリー生成通知（通知文＋入口メッセージの事前生成）、`notification_candidates` / `notification_logs` への保存、同じ文脈IDでの通知文・入口メッセージ連携、関係値重み配分＋lover毎日確定、キャラ個別ON/OFF、`device_installations` を使った有効端末送信、重複防止
- [ ] イベント・告白（A11）: 発生条件（サーバルール）、通常通知→疑似LINE入口メッセージ→電話していい？→OK→アプリ内疑似着信→応答→疑似電話の導線、告白・関係状態変更・呼び方変更の明示同意フロー、pending状態、未応答時の戻し/保留、lover化と文脈変化、iOS/Android共通の告白導線
- [ ] 猫（A3-6）: ランダム猫（ルール）／AI猫（分類）、懐き度

## フェーズ4. リリース前チェック（Z-9）

- [ ] 実機検証全項目（マイク権限 / Android `AudioRecord` 24kHz mono PCM16 / STTストリーミング / VAD / TTS再生 / TTS後の発話待ち復帰 / 残高不足 / 二重消費なし）
- [ ] モーニングからの本番AI通話開始 / 通知→疑似LINE経由の通話開始（Androidは `USE_FULL_SCREEN_INTENT`、メーカー別full-screen intent動作、Foreground Service音、通知タップフォールバック、専用Activity遷移、実AI接続込みで再確認）
- [ ] ストア審査確認（iOSでCallKit不使用の方針が保てているか、疑似電話がガイドライン適合か）
- [ ] ストア商品設定（消耗型コイン / 非消耗キャラ枠 / 自動更新サブスク）

---

## 補足: 未決（Z）と本書の対応
- Z-1〜Z-3（数値）→ フェーズ2
- Z-4（記憶の日数・遅延）→ フェーズ0/3で実測しながら
- Z-6（声・素材）→ フェーズ0。声モデルの商用ライセンス確認とv1仮採用は完了。残りは実際のTTS品質・音声サンプル確認。
- Z-7（画面設計）→ フェーズ1/3で各画面を作りながら確定
- Z-8（RLS・API）→ フェーズ1。RLS・制約・インデックスのDB土台と指定migration 5件の本番反映は完了済み。v4.3匿名Auth方針でのRLS実動作と `current_app_user_id()` による `public.users.id` 解決は staging で検証済み・PASS（2026-07-11、漏れなし）。本番(voice-companion)での同確認は未完了。API設計、Edge Function整理、アプリ側読み書き実装も未完了。匿名サインインと引き継ぎコードを前提にする。`transfer_codes` はテーブル追加のみで、発行/引き換え用 Edge Function / RPC とUIは未実装。
- Z-9（実機検証）→ フェーズ0・フェーズ4。Androidモーニング導線のフェーズ0検証は合格、残りは本番AI接続・フォールバック・リリース前総合確認。
