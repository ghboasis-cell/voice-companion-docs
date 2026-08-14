# VoiceCompanion 作業手順書 兼 運用ルール

**版数: v5.198 ／ 最終更新日: 2026-08-12**

（v5.198: v5.197の有償コイン無期限化・退会時全消滅をstagingへ反映した。migration `20260812100000_make_paid_coins_permanent_and_forfeit_on_deletion.sql` はstaging project `nqqkbwhrwinxuameyzya`へ適用済みで、remote migration一覧との一致を確認した。`revenuecat-webhook` も同projectへdeploy済み。commit `52a9147`からAndroid Staging AAB workflow run `31566852903`を起動し、10分10秒でSUCCESSした。versionCode `1177`、artifact `android-staging-aab-1177`（80,851,953 bytes、artifact ID `9129968864`）を生成・アップロード済み。変更後の退会・再登録・購入通知を使った実機確認は未実施。productionは未変更。）

（v5.197: **有償コインを買い切り・月額とも無期限にし、任意退会では購入分を含む全コインを消滅させる改修を実装した**（spec v5.62）。forward migration `20260812100000_make_paid_coins_permanent_and_forfeit_on_deletion.sql` は、新規の有償ロットを `expires_at=null` で付与し、旧期限で失効済みの有償ロットを失効時点の残数まで復元する。退会では有償ロットを監査用に残数0で保持し、無償ロットを削除し、表示残高を0にする。退会後の旧ユーザーへ月額更新通知が届いても `account_inactive` で再付与せず、Webhookは200を返して無意味な再送を止める。コイン画面・退会確認・利用規約・ストア設定手順も同じ説明へ変更した。**migrationはstaging・productionへ未適用、変更後の実機確認は未実施。** 有償コイン無期限化に伴う資金決済法上の届出・表示・保全等の要否は、リリース前に専門家確認が必要。）

（v5.196: **通話中に他のアプリへ移ったら通話を終了する形にした**（spec A8 v5.60〜v5.61、AAB 1175で実機確認済み）。これまでは通話を残したままにしていたため接続が切れ、「通信に失敗しました」と出て「もう一度試す」が押せる状態になっていた。時間は課金されているのに何が起きたか分からない。自分で終了を押したときと同じ終わり方にした。**引かれたのにトークへ何も出ないのは分からないという指摘を受け、「他のアプリを開いたため通話を終了しました」を1枚出すようにした**（migration `20260811080000`）。コイン切れの文言は使えない。12枚のうち10枚引かれただけで足りなくなってはいないためである。あわせて、コイン切れのお知らせが2枚並んでいたのを直した（`20260811070000`）。spec B10は初めから「1キャラにつき1枚、新しく出すときは古いものを消す」と定めていたが、実装が消していなかった。終わり方のお知らせは種類を問わず1キャラ1枚とした。）

（v5.195: **Androidだけ「コインが足りなくなったので通話を終了しました」がトークに出なかったのを直した**（利用者の指摘）。`record_coin_shortage_notice` は終わったと記録済みの通話にしかお知らせを出さないが、終わったことを記録する `mark_call_usage_pending` はoutbox経由で、端末から通話完了の知らせが届いてから送られる。終話した瞬間にはまだ送れていないので毎回nullで捨てられていた。iPhoneは端末側が先に完了を出すぶん間に合っていた。記録されるまで数回やり直す形にした。あわせて「コイン0なのにプッシュ通知が来た」を第4段の残りへ書いた（未調査）。）

（v5.194: **残高を使い切って終わった通話が毎回タダになっていたのを直した。** 通話は「その残高で話せる上限」ちょうどで切っているが、記録される時間は止まるまでのぶんだけ数十ms超える。上限は払える最大として決めているので、超えた瞬間の料金は常に払える額より1単位多く（10枚→14枚）、必ず払えない。`settle_call_coins` は1枚も引かないまま `failed` にしており、`failed` は日次の回収対象から外れるので後からも引かれなかった。**「持っている枚数で払える上限を超えた請求はしない」に直した**（12枚なら10枚引いて2枚残す。migration `20260811060000`）。使い切ったからといって残高を0にはしない。精算は終話の後に走るため、ちょうどその時に日付が変わって毎日の無料コインが入っていることがある。記録は書き換えず、実測の課金時間はそのまま残す。**あわせて、Androidにも端末側の締め切りを入れた。** `maxBillableDurationMs` を読んでいたのは iOS だけで、Androidは画面側のタイマーだけが頼りだった。通話中に他のアプリへ移るとタイマーは間引かれ、上限を大きく超えたぶんがそのままタダになる。spec A8 v5.59へ反映済み。**stagingへ未適用・実機未確認。**）

（v5.193: 購入の復元をAAB 1171で実機確認し、2つ直した。**`TRANSFER` が4回とも400で弾かれていた。** `app_user_id` が空で届くイベントなのに、分岐を「`app_user_id` が無ければ400」の判定より後ろに置いていたためである。テストは書いていたが比べる目印を間違えており、実際に弾いていたのはその手前だった。もう1つは、ストア側が失敗を返したのに契約は実際に移っていた件で、判定をサーバーの状態で行う形に変えた。直したあとの実機で、契約の移動・旧側の枠が消えること・新側へ枠と契約が移ること・コインが4800枚**移動**して旧側から消えること・毎日の無料コインが移らないことを、すべて記録で確認した。枠を買ってキャラを追加できることも確認した。**テスト購読は数分で期間が終わり、その期間ぶんとして配られたコインも同じだけ短命である。** `coin_balances.balance` は動いたときだけ書き直す控えなので、失効しても古い数字が残って見える。）

（v5.192: 引き継ぎのあと端末登録の書き込みが拒否される件を直した。アプリが「自分の他の端末を無効にする」更新をかけるが、対象は旧端末の行で、行の `auth_user_id` が今のセッションと違うため端末からは触れない。**spec C5 の「引き継ぎ完了時に旧端末のPush token・着信・通知・モーニングを無効化する」を実装していなかった。** 引き継ぎの中でサーバー側が行う形にした。守りは緩めていない。あわせて、既に引き継ぎ済みで詰まっていたstagingのデータも同じ形で直した。なお、確認の途中でコインが 3009 から 6009 になったのは引き継ぎではなく、テスト購読が数分ごとに更新されて月額ぶんが配られたためである。引き継ぎの16秒前に更新が入っていた。引き継ぎコードはコインを1枚も動かさない。）

（v5.191: 第4段（Androidで確認）に入り、引き継ぎコードをAAB 1171で実機確認した。コインとキャラは引き継がれた。**トークの引き継ぎは未確認**である（確認時にトークをしていなかった）。実機で3つ直した。引き継ぎが**誰がやっても必ず失敗していた**こと（`redeem_transfer_code` が一意制約のある `users.auth_user_id` を書き換えていた。入れ直した端末ではその認証IDを仮ユーザーの行が既に持つ）、引き継ぐ人が捨てるだけの名前とキャラを先に作らされていたこと、引き継ぎ入力の画面で下のボタンが上と重なって見えたことである。あとの2つは利用者の指摘による。）

（v5.190: **枠を買ってもキャラを追加できなくなっていたのを直した。** 実機で「追加できませんでした」と出る。原因は前工程の `20260810020000_release_character_from_ended_slot.sql` で、`assign_character_slot` を書き直したときに `user_characters` へ入れる列が4つ落ちていた（`acquisition_type` / `is_ai_enabled` / `ai_enabled_at` / `name_set_at`）。`acquisition_type` は NOT NULL で既定値が無いため、**このmigrationが当たった時点からキャラ追加は必ず失敗していた。** 誰も気づかないまま1日たっていた。`20260808000000` の形へ戻し、書き直しで変えた中身（枠の `released_at` で見る、買い切りへ移したら連絡を再開する）は残した。stagingへ適用し、実データで `{"ok": true}` を確認した（ロールバック付き）。同じ落ち方を止めるテストを足した。**この種の誤りは自動テストでは出ない。** 関数を書き直すときは、前の版のINSERT列と必ず見比べること。）

（v5.189: migrationを3本stagingへ当て、`revenuecat-webhook` をdeployした。あわせて、当ててから実際に走らせて2つ直した。`coin_transactions.direction` に `out` / `in` は使えず `consume` / `grant` だったこと（**流すまで出ない誤り**である。PL/pgSQLは中のSQLを実行時にしか検査しない）と、毎日の無料コインを移さない形にしたことである。無料コインは毎日配られてその日で失効し、新しい端末でも同じだけ配られるので移す意味が無い。移すと同じ日ぶんを両方が持ち、上限でまとめる処理が要る。spec B8 v5.58。stagingで通しの動きを確認した（全部ロールバックする形で実行）。枠の移動・冪等・コインの移動・未精算と返金回数の引き継ぎ・無料コインを移さないこと、すべて意図どおりだった。）

（v5.188: 購入復元の作り直しを実装した。`TRANSFER` の受け口、枠の移動、コイン移行のRPC、復元前の確認、コイン移行の選択、結果の4通りの文言である。**設計から1点変えた。** 当初は「旧IDの未精算の通話を先に精算する」としていたが、`settle_call_coins` は呼んだ本人の通話しか精算しない（所有者が違うと落ちる）。呼んでいるのは新IDなので旧IDの通話は精算できない。よって未精算が残っていたら何も移さずエラーで終える形にした。あわせて `TRANSFER` は `app_user_id` が空で届くことがあり、共通の入口より後ろに置くと400で弾かれるので前へ置いた。**これで第3段は全件完了。stagingへ未適用・実機未確認。** migrationは3本たまっている。）

（v5.187: コイン切れのお知らせカードが再起動で消える件を直した。端末のメモリにだけ置いていたので、開き直すと消えていた。「数秒で消える帯だと購入導線を押しそびれる」ために残すと決めたのに、開き直しで消えるなら同じである。不在着信と同じくトークの行として残す形にした。作りながら分かったのは、**`calls.end_reason` は列こそ在るが通話終了時には誰も書いていない**ことである。日次の回収処理しか書いておらず、`calls` は端末から更新できないため書かせることもできない。よって「残高不足で終わった」をDBから確かめられず、RPCは自分の・終わっている通話であることだけを確かめ、理由は端末の申告を信じる形にした。偽っても自分のトークに1枚増えるだけで、コインも残高も動かない。**stagingへ未適用。**）

（v5.186: 第3段の引き継ぎコード（C6/C7）を作った。`transfer_codes` はテーブルだけ在って動かす手段が無かったので、発行と入力のRPCを足し、引き継ぎ画面を作った。「購入を復元する」をコイン画面からこの画面へ移し、設定画面の「コイン / 課金」を消した。コードは英数字8文字で `0 O 1 I L` を使わず、有効期限30分、同時に有効なのは1本、入力の失敗5回で15分止める。作りながら3つ潰した。寄せ先に `X` を使っていたこと（`X` はコードに使う文字なので、打ち間違いの `O` が正しいコードに当たる）、`random()` でコードを作っていたこと（種から予測でき他人のアカウントへ入れる）、戻り値の名前が列名と同じでPL/pgSQLが落ちること。**stagingへ未適用・実機未確認。** 実機はまとめて確認する。）

（v5.185: F5の最後の1件「サブスク権利の自動反映」を確定し、これで**第2段（決める）が全件完了**した。引き継ぎコードと購入の復元を別の機能として切り分け、復元では月額契約・枠1つ・次回以降のコインの配り先だけを移す。RevenueCatの Restore Behavior は `Transfer to new App User ID` のまま変えない。spec B8 v5.57。あわせて第3段へ「購入復元の作り直し」を足し、既存実装を読んだうえでの設計（対応表・冪等の門・処理順序・トランザクション境界）を書いた。**実装は未着手。** 設計を引き継ぎ書ではなく本書へ書くのは、引き継ぎ書だけにあると担当が代わるたびに作り直されるためである。）

（v5.184: F5の「消費したコイン枚数を見せるか」と「課金対象時間の内訳・秒数を表示するか」を、まとめて**出さない**で確定した。出すのは残高だけで、タイムラインには1回ごとの枚数も通話の時間も置かない。作るものは無く、いまの実装のままである。spec B10 v5.56。F5の残りは「サブスク権利の自動反映」1件。）

（v5.183: F5の4件のうち「残高が尽きたときの通話終了タイミング」を**0になった瞬間に切る**で確定した。言葉の途中でも待たない。実装は既にこの動きで、仕様書だけが未決のまま残っていた。待つ形を採らないのは、待ったぶんが払われていない時間になり、超過を無料にするか未精算として持ち越すかという別の決めごとが要るためである。spec A8 v5.55。F5の残りは3件。）

（v5.182: 第2段の2件目「spec A8の予告3か所を実装に合わせて直す」を反映した。境目は残り20秒、音源は端末に入っている読み上げ音声（同梱の音声ファイルは使わない）、文言は「残りコインが少なくなっています。」である。いずれも実装が先に変わっており、仕様書だけが古いまま残っていた。spec A8 v5.54へ書いた。新しい文言は「通話が終了します」を言わないが、**このままでよいと利用者が決めた**。）

（v5.181: 第2段の1件目「予告を聞き取り失敗の1・2回目でも鳴らすか」を**鳴らす**で確定した。これは以前に決まっていた話であり、仕様書に書かれていなかったため実装が両OSで割れていた。iPhoneをAndroidへ合わせ、spec A8 v5.54へ明記した。決定済みのことを未決として出し直さないため、決まった内容は必ず仕様書へ書く。）

（v5.180: 課金工程の第1段「通話を直す」を完了として記録した。iPhoneの聞き返しが鳴らない件は、音声経路ではなく接続要求の作り直しで `onDeviceTts` が既定値の `false` へ落ちていたことが原因だった。落ちるとサーバがクラウドの音声データで送り、iPhoneはそれを捨てる作りなので必ず無音になる。全項目を写して解消し、実機とサーバの記録の両方で確認した。あわせて、この工程で入れて外した「通話中の音声経路の組み直し」と、サーバの記録の読み方（端末の行とサーバ自身の行の見分け）を寄り道として残した。同じ読み違いを繰り返さないためである。）

（v5.179: フェーズ3の課金（B）へ「課金工程の残り」を追加し、作業順を第1段〜第6段で確定した。これまで残り作業は引き継ぎ書の中にしか無く、担当が代わるたびに一覧を作り直していた。引き継ぎコード（C6/C7）はこの工程に入れる。spec B8が「買い切りのキャラは引き継ぎコードで元のIDへ戻れた場合だけ戻る」と定めており、無いと確認項目の「購入の復元」が最後まで通らないためである。退会（C9）と入れ直し対策（B8-2）は入れない。どちらも課金の確認を止めないためで、退会は「退会するとコインは消え払い戻さない」という条件を課金から受け取るだけ、入れ直し対策は消す手段を同時に作れば入れ直しでの検証を続けられる。通話が壊れていて課金の確認ができないため、通話の修理を第1段へ置いた。iOSは課金導線を一度も通しておらず、Codemagicを利用者が押す必要があるため、コードを固めてから1回で通す第5段として独立させた。）

（v5.178: フェーズ3-後「入れ直しによる周回を止める」を独立した工程として追加した。spec B8-2の購入停止は入れ直しで丸ごと消えるため、出す前までに端末へ旗を立てる仕組み（Android: Play Integrity API の device recall、iOS: DeviceCheck）が要る。以前「Androidに規約上きれいな手段が無い」としていたのはGoogleの資料で確認した結果、誤りだった。課金の工程では入れない。）

（v5.177: F3の最後の未決だった共通ベースの上乗せを確定し、コインを期限つきロットで動かす仕組みを
入れた（spec v5.41、migration `20260805220000_apply_f1_coin_tariff_and_lots.sql`）。会話量の倍率を
×1.5から×2.0へ改め、上乗せは3段（+7／+9／+11）をやめて、その日に消費したコインの総量で測る2段
（19〜36枚で+1.5、37枚以上で+3）にした。×1.5では告白までの日数が上乗せ+2で頭打ちになり、上の段に
意味が無かったためである。あわせてフェーズ3の課金（B）を `[~]` へ更新した。料金・無料付与・消費順序・
日またぎの失効はstaging実機で確認済みで、残るのは購入・月額のロットを書き込む側だけである。）

（v5.176: F1のコイン数値を実測にもとづいて全項目確定し、spec v5.37・v5.38として正本化した。
消費はチャット1往復3コイン、通話は60秒までで10コイン・以後30秒ごとに4コイン、端数は切り上げ、
通話開始には10コインが必要。無料付与は1日18コイン。買い切りは購入から6か月、月額は未使用分を
1か月だけ繰り越して翌月末で失効し、保有・繰り越しとも枚数の上限は設けない。有効期限を6か月以内に
限るのは資金決済法の適用除外を維持するためである。根拠はstagingの実測で、通話1往復8.63秒・課金時間の
うちユーザー発話59.6%（`call_logs` 678件）、通話1ターン$0.00021、チャット1回$0.00037〜$0.0012
（`chat-reply`と`voice-turn`のusage実測）、文字起こし$0.003/分、音声合成$0から、1コインの原価を
約$0.00042（0.065円）とした。ストア手数料は年商1億円以下ならAppleもGoogleも15%である。
あわせて`chat-reply`の指示文で現在日時を会話履歴の後ろへ移した。分単位で変わる値が履歴より前にあると
入力の3分の2が毎回キャッシュから外れるためで、`cached_tokens` 1,792→5,888、チャット1回$0.00087→$0.00037
となった。`voice-turn`は同じ形のままで、効き目は小さいと見て手を付けていないが確かめてはいない。
モデル比較用に`scripts`のLLMベンチを足した。モーニングの会話登録も実機の指摘4件を直した
（確認カードが出ない・1発目から出る・キャンセルしてもしつこい・AIが書く時刻や日付の揺れ）。
判断はキーワードの門番をやめて`morning_decision`（propose / offer / none）へ一本化し、カードは
1キャラ1枚まで、押した記録はRPCと同じトランザクションでトークへ残す。spec A9-1へv5.39として反映した。
**コインの仕組み自体は未着手である。**`coin_lots`は形だけあり書き込む側が無く、動いているのは
`coin_balances`の数字1つだけで期限を持てない。旧料金の暫定値は`src/appMain.ts:221`（チャット1コイン）、
`src/callBalanceGuard.ts:5`（最低2コイン）、migration `20260711090000`（RPC内の1コイン）、
`20260715170000`（60秒ごと1コイン・最低2）の4か所にある。`npm test` 720件PASS。
productionは未変更。）

（v5.175: A11の告白イベントを、条件成立の判定から結果反映まで通した。仕様変更はない。
新しいテーブルは足さず、既存の`event_states`へ`event_type='confession'`で載せた。進み具合は
`event_status`だけでは「電話に誘っている最中」と「返事を待っている最中」を区別できないため、
payloadの`stage`（scheduled / invited / confessing / awaiting_answer / answered）で持つ。
数値の置き場を分けたまま保つことを設計の軸にした。予約日・再挑戦日・期限は決めた時点で
具体的な日付・時刻へ確定させてpayloadと`expired_at`へ書き、配信するスケジューラも返答を受ける
RPCも日数を持たない。例外は付け替えの割合2つだけで、利用者が直接呼ぶRPCには係数を引数で
渡せないためSQLへ書き、`RELATIONSHIP_CONFIG`と一致することをテストで固定した。
判定は日次処理の関係値更新の直後に置いた。`nextConfessionAttempt`が3条件（または2回目の
条件）を見て、成立した日に1〜7日後のどれか1日を予約する。配信の「いつ」はA10-2の
共通スケジューラが持つ。`contact_schedule_slots`へ`slot_kind='event'`を足し、予約日が来たら
確率ではなく必ず枠を置き、現地8:00〜21:59・近接回避・モーニング回避を通った時刻に
`memory-worker`の`confession_deliver`を呼ぶ。文面と状態遷移は`memory-worker`側にある。
告白イベントは通知枠で出す。疑似着信になるのは利用者が「電話していい？」へOKを押した後で、
A10-2の着信振分（1日1着信・残高・休眠）とは別勘定である。着信OFFの利用者はトークで告白し、
通話を使わない。通話できる残高が無い日は出さずに予約日を1日ずらす。
入口メッセージは通知のタップを待たずにサーバがトークへ入れる。アプリを開かない利用者にも
出す必要があるためで、そのぶん`open_notification_in_chat`はイベント通知では入口メッセージを
入れないようにした。二重に入るのを防ぐためである。
確認メッセージの選択肢はサーバがpayloadへ確定させ、アプリは描くだけにした。「1回目だけ保留を
出す」判断をアプリ側にも実装しないためである。期限切れ後は行が読めなくなるので押せない。
断られたときは`friend`固定ではなく告白前の関係状態へ戻す。単に`friend`を書くと、親友だった
利用者が断っただけで親友を失うためである。
「保留のまま利用者が会話を始めた」場合は、`voice-turn`が誘い中のイベントを見てキャラ側から
切り出し、その通話をイベントへ結びつける。「さっきの話は何？」と聞かれたかをAIに判定させない。
はっきり断られたときは切り出さない。2日後に誘い直す側が正しいためである。
設定画面へ恋愛イベント許可のキャラ別切替を追加した。OFFにする前に進みが減ることを確認し、
差額の25%を友人スコアへ移す。恋人になった後はRPC側で切り替えを拒否する。
A10-7の告白1往復無料も入れた。migration `20260803100000`で`call_logs`へ免除の範囲を持たせ、
`upsert_call_usage`を書き直した。無料になるのは告白したターンの応答音声と、その次のターン全部
である。仕様の対象が「告白のセリフ／ユーザーが返事をしている時間／それに対する応答音声」の
3つで、ターンは「ユーザーが話した時間＋応答音声」で1行のため、この2行にまたがる。
告白したターンでユーザーが話した時間は通常どおり課金する。起点は`voice-turn`が実際に告白へ
入るターンで記録し、一度入ったら上書きしない。精算側は`calls.billable_duration_ms`を読むだけ
なので変更していない。既存の`tests/callUsageRecording.test.mjs`は旧migrationを読んでいるため、
所有者・revision・直列化・コイン不変の条件を新しい定義側でも固定するテストを足した。
保留中にチャットで話しかけられたときの切り出しも`chat-reply`へ入れた。チャットでは告白を
完結させず、電話の件を切り出すだけにする。関係状態はチャット側では一切変えない。
`npm test` 667件PASS、`npx tsc --noEmit`エラーなし、Web buildも通る。
stagingへは2026-08-03に反映した。migration 2本を適用し、`memory-worker` version 22、
`common-contact-scheduler` version 12、`voice-turn` version 37、`chat-reply` version 24 へ
deployした。`MEMORY_WORKER_SECRET`はプロジェクト単位のsecretとして登録済みで、
`common-contact-scheduler`からもそのまま読めるため追加作業は無かった。productionは未変更。
実機確認は未実施である。）

（v5.174: モデルbundleのダウンロードが実用に耐えていなかったのを両OSで直した。実機の初回
セットアップで`voice model preparation failed`となり、そこから3つの実装欠陥が出た。
1つ目は取得層が`new OkHttpClient()`のままで、既定の読み取り10秒を超えて途切れると
100MB規模のDL全体が失敗し、再試行も再開も無かったこと。読み取りを60秒へ延ばし、
全体のタイムアウトは置かず、切れたら最大4回まで`Range`で続きから取り直す。4xxは
何度やっても同じなので再試行しない。落とし切った直後に終了した場合に返る416は、
4xxとして確定失敗にすると完成しているのに永久に進めなくなるため、手元のファイルを
検証へ回す。2つ目は`BufferedOutputStream`が最大8KBを抱えたまま切断で捨てられ、
ディスクの長さと受信量がずれて再開位置を誤り、繋ぎ目のずれた壊れたzipを作る状態だった
こと。バッファを外して受信分がそのままディスクに載るようにした。この欠陥は
MockWebServerで実際に切断して再開させるテストが捕まえた。3つ目は進捗も失敗理由も
画面へ届いておらず、`PrepReporter.NONE`と固定文言のrejectで、動いているのか固まったのか
利用者に判断できなかったこと。両OSとも同じイベント名・同じ形で進捗を送り、例外の本文を
利用者側まで運ぶ。一時ファイル名をURLから決まる固定名にし、失敗時も残すことで、
プロセスが死んだ後の起動でも続きから再開できる。
あわせてバックグラウンド継続を両OSで実装した。Androidは`TtsModelPreparationService`
(foregroundServiceType=dataSync)を準備中だけ立ててプロセスを終了対象から外し、
PARTIAL_WAKE_LOCK(上限1時間)で画面消灯中もCPUを止めない。通知バーへ同じ進捗を出す。
iOSはアプリ内`URLSession`ではサスペンドで止まるため、`URLSessionConfiguration.background`
へ移した。OSのデーモンが転送を担うのでサスペンド・終了後も続き、完了時はOSがアプリを
起こして`AppDelegate`の`handleEventsForBackgroundURLSession`からセッションへ繋ぎ直す。
同一URLの転送が走っていれば合流して二重DLを避け、失敗時は`resumeData`を保存する。
旧`URLSessionBundleFetcher`は削除した。残すとproductionで使われないコードをテストが
検証し続けるためで、実際にhttpsのテストがそうなっていた。
進捗表示は共通モデルとキャラモデルの区別を出さず、2本を1つの割合へまとめた。
本ごとに0〜100%を出すと1本目の完了時に0%へ戻り、やり直したように見えるためである。
Codemagicの`beta_groups: AlarmKit Test`も外した。内部グループには全ビルドが自動公開され
割り当てAPIは外部専用のため、アーカイブ・アップロード・処理がすべて終わった後の最終段
だけが必ず失敗し、通ったビルドが失敗として報告されていた。
`npm test` 632件PASS、Android JUnitもCI PASS。iOSはCodemagicでcompile・AppTests・IPA生成・
TestFlight配布までPASSし、実機で%表示とDL完了、他アプリ移動・画面ロックをまたぐ継続を確認した。
Android staging AABもbuild成功し、実機で%表示・通知バー・DL完了を確認した。
AndroidのDLはiOSより時間がかかるが、区間別の実測はしていないため原因は未特定。
体感で許容できるとして計測は見送った。）

（v5.173: A11の関係値をspec v5.36として正本化し、点数が実際に動くところまで実装した。
仕様書はA5を3スコア構成・加算・減点2系統・節目5つ・呼び方・恋愛イベント許可フラグへ書き直し、
A3-8の初回3択を2段構成、A11へ告白の3条件と発生タイミングと通話にたどり着けない場合、
A10-5を通話中に確定させない形の返答と付け替え、A10-7を告白が始まった地点から1往復無料へ更新した。
A10-2には、読んでいる`intimacy_score`が共通ベースの生値であり6で割って0〜100へ変換することを明記した。
実装は3つに分けた。migration `20260802090000`で`character_relationships`へ友人・恋人スコア、
各スコアの到達最高値、連続マイナス日数、最後に会話した日、会話日数、呼び方の段階、
告白の回数と断られた時点の基準値を追加した。`intimacy_score`は共通ベースとして列名を変えない。
スケジューラとアプリが既に読んでいるためである。`memory-worker/relationshipPolicy.ts`へ
A11の数値と純関数を集約した。DBにもAIにも依存しないので、会話した日を1日ずつ積む
シミュレーションで到達日を検証できる。疎遠減衰は毎晩書かず次の会話日へ遅延精算する。
減衰量は最後に会話した日からの経過日数だけで決まるため結果が変わらない。
migration `20260802100000`で`apply_daily_relationship_delta`と`apply_relationship_milestones`を
追加した。前者は減衰精算・加算・最高値更新・会話日数加算を1トランザクションで行い、
同じ日付を二度積まない。日次処理は失敗すると再試行され、部分的に成功した後の再実行が届くためである。
減衰の係数は引数で受け、SQLとTSへ同じ定数を二重に置かない。`memory-worker`は日次要約と同じ
AI呼び出しへ`conversation_category`を相乗りさせ、プロンプトに点数を出さない。
通知やイベントだけの日は材料には載るが会話ではないので積まない。
2 migrationをstagingへ適用し、`memory-worker`をstaging Function version 21へdeployした。
productionは未変更。残る欠落は2つある。課金ボーナスは、購入コインを消費したかの判定に
ロット単位の消費記録が必要だが仕組みが無いため段を0で固定してある。会話量の境界は
F1の無料付与コイン量が未決のため暫定値2コインを`PROVISIONAL_TIER_THRESHOLDS`へ隔離した。
告白イベントの発生・導線、UI3つ、`src/appMain.ts`の初期値変更は未着手。）

（v5.172: A11の告白イベント運用を確定した。発生は3条件成立から1〜7日後のどこか1日を予約する
形とし、発生条件に通話できる残高を加える。着信OFFの利用者はトークで告白し、「電話していい？」を
断られた場合は回数を消費せず2日後に再度誘い、それでも駄目ならトーク告白へ切り替える。保留中に
利用者が会話を始めたらキャラ側から切り出す。確認メッセージの期限は3日とし、返事待ちの間は
気まぐれ着信を止めてデイリー枠だけにする。2回目は恋人スコア+60かつ会話した日で30日以上を
条件とする。通話は告白が始まった地点から1往復を課金対象外にする（通話の最初からではない）。
付け替えは固定割合ではなく差額方式へ変更し、打ち切りは`max(0, 恋人−友人)×50%`、設定切り替えは
同25%とする。カテゴリが混ざり友人と恋人の比が利用者ごとに違うため、固定割合では効き方が
ばらつき友人スコアが恋人スコアを超える場合があった。キャラ性による差はv1ではセリフと態度だけに
適用し数値は全キャラ共通とする。あわせて親友の判定を総合点から共通ベース＋友人スコア560へ変更し、
関係の種類を決める節目はその種類の物差しだけで見る形にした。会話量2段階はコイン消費量で測り
境界を無料付与の1日分に置くと決めた。残る未決は会話量と課金3段階の境界の実数だけで、F1待ちである。
仕様書本体への反映は未着手。）

（v5.171: A11の関係値数値を確定し、その影響でA10-2スケジューラの親密度換算を引き直した。
共通ベースは会話が成立した日+5、購入コイン消費で最大+11、中身は好意・甘えと相談・打ち明けを
総合+10、雑談+7、挨拶だけ+5とし、会話量の倍率は中身にだけ×1〜×1.5でかける。疎遠減衰と
暴言の減点は友人・恋人スコアにかけ、共通ベースは減らさない。節目は200／450／700／950、
告白は総合1200かつ共通600かつ恋人300、初回3択の初期値は0／50／100とした。
決定内容と根拠は`docs/handoff_a11_relationship_design_20260801.md`にある。
スケジューラは`intimacy_score`を0〜100へ丸める前に6で割るよう修正した。共通ベースが
1日+5になったため、旧実装では会話20日で補正が上限へ張り付き、そこから告白までの100日間
連絡頻度が一切変わらなかった。6で割ると告白の床600が補正値100に対応する。生値を直接
読んでいた`callProbability`も同じ変換を経由させ、3つの式自体は変更していない。
`npm test`606件PASS、`npx tsc --noEmit`エラーなし。stagingへdeployしACTIVE version 11。
productionは未変更。仕様書本体への反映と`src/appMain.ts`の初期値変更は未着手。）

（v5.170: F3 v2の`common-contact-scheduler`をstagingへdeployし、ACTIVE version 10、
更新時刻2026-08-01 10:49:24 UTCを確認した。今日すでに保存済みの枠は変更せず、次回新しく
計画する現地日付から親密度の連続補正を適用する。productionは未変更。）

（v5.169: A10-2の気まぐれ抽選をF3 v2へ修正した。v5.168までは連絡量3段階の基準率だけで
発生を決めてから関係値で相手を選んでいたため、所有キャラが1人だと関係が深くなっても発生率が
変わらなかった。相手を先に関係値重みで選び、そのキャラの親密度0〜100を倍率0.75〜1.25へ
連続変換して、少なめ7%／ふつう15%／多め45%の基準率へ掛けるよう変更した。親密度1点ごとに
倍率0.005ずつ変わり、relationship stateやaffinity levelの段階変更では急変しない。複数キャラの
選択重みも`1 + intimacy_score÷20`へ変更し、25点ごとの切り捨てを廃止した。F3 v1の安全上限、
時間帯、近接回避、休眠、着信振分は維持する。productionは未変更。）

（v5.168: A10-2に関するF3 v1をspec v5.34で正式化した。少なめは週約4回、ふつうは
週約8回、多めは週約9〜10回とし、デイリー発生率50%／100%／100%、気まぐれ発生率
7%／15%／45%、気まぐれ上限1日1回・週3回とした。配信8:00〜21:59、同一キャラ6時間、
別キャラ90分、利用者からの連絡後3時間、モーニング前後90分、7日休眠45%、30日休眠20%、
休眠中は通知のみ、着信最低残高3、無料のみ着信率45%、1日最大1着信を正式値とした。
暫定config名と`provisional-f3`記録を`SCHEDULER_CONFIG`／`f3-v1`へ変更し、3段階の差が
体感できるよう少なめを従来約5回から約4回、多めを従来約9回から約9〜10回へ調整した。
productionは未変更。）

（v5.167: `agent/common-contact-scheduler-build`の最新アプリをiPhone・Androidの両方で実機確認した。
設定画面に連絡量「少なめ／ふつう／多め」が表示されること、録音準備音をON/OFFできること、
OFFでは準備音だけを省いて従来と同じタイミングで録音が再開し、ONでは準備音が鳴ることを確認し、
ユーザー報告「両方確認、問題なし」となった。AndroidはAAB 1105、iOSはユーザーがCodemagicから
build・導入したstagingアプリによる確認で、Codemagic build ID/indexは未記録。これにより連絡量設定と
録音準備音ON/OFFのnative build・両OS実機確認をPASSとする。F3の正式頻度・確率とproduction反映は未実施。）

（v5.166: A10-2・連絡量3段階・録音準備音ON/OFFを含むcommit `437177c`を、
Android Staging AAB workflow run `30692726678`（run number 105）でnative buildした。
staging Supabase設定、TTS bundle設定、Web build、Capacitor sync、Rust/NDK、署名付きAAB生成、
artifact uploadはすべてSUCCESS。versionCode `1105`、versionName `staging.1.0.105`、artifact
`android-staging-aab-1105`（80,771,777 bytes、artifact ID `8816346261`）を確認した。
Android実機での連絡量保存と録音準備音ON/OFFは未確認。iOSはユーザーがCodemagicでbuildするため
こちらからは起動せず、対象branch `agent/common-contact-scheduler-build`をpush済み。
productionは未変更。）

（v5.165: A10-2のJIT配信をstaging実機で完了確認した。修正版反映後、当日分の既存2枠だけを
試験用に再設定し、iPhoneはJST 17:25、Androidは17:30に通常通知を受信した。続いて同じ2枠を
着信へ切り替え、iPhoneは17:40、Androidは17:46に気まぐれ着信を受信した。利用者からの返信・発信は
無かったため3時間後ろ倒し条件には該当していない。試験変更は2026-08-01の2枠だけで、通常plannerの
現地時刻8:00〜21:59、quiet hours、モーニング前後90分回避、翌日以降の日次計画は変更していない。
これにより通知／着信とも、due検出、送信直前生成、Push配信、両OS受信までPASSとする。
productionは未変更。連絡量3段階と録音準備音ON/OFFを含む最新native build・実機確認は次工程。）

（v5.165: Android実機で、18枚の時点で作った着信予定が配信前に0枚へ変わっても届き、応答時だけ
残高不足で止まる不具合を再現した。通常メッセージは3枚、疑似着信は10枚を、予定時だけでなく
生成前とPush送信直前にも再確認し、不足時は送らず相互に振り替えないよう4つのEdge Functionを修正した。
全933 test、TypeScript、Web build、diff検査はPASS。4 Functionをstagingへdeployし、0枚で予定段階と
候補・queue作成済みのPush直前の両方が停止し、provider ID・通知ログ・端末受信0を確認した。
試験データは削除し18枚へ復元済み。）

（v5.164: JIT通知の初回実時刻検証で、JST 15:56のiPhone通知が届かなかった原因を修正した。
通知文と端末別queueは正常に作成されていたが、配信workerが起動時刻を固定した後、約1秒後に作成された
queueの`next_attempt_at`を同じ固定時刻と比較して対象外にしていた。旧常時配信cronはJIT化で停止済みだったため、
そのqueueを拾う次回起動もなかった。queue作成時に明示的な実行時刻を設定し、作成後の現在時刻で取得するよう
変更した。再試行待ちが存在する場合だけ既存の1分due確認から配信workerを起動し、通知の通信失敗再試行は
1回までとした。30分を過ぎた通知候補は送らず終了し、修正反映前の15:56分も遅れて突然届かないよう
`superseded_by_delivery_repair`で終了した。stagingへ`notification-dispatch-worker` version 23と
forward migration `20260801180000`を反映済み。全603 test、TypeScript、diff検査はPASS。
次のJIT予定（Android JST 21:27）で実端末受信を確認する。productionは未変更。）

（v5.163: staging Security AdvisorのWARN 54件をManagement APIで取得し監査した。
37件は匿名サインインを使う本アプリに対して同一警告が対象tableごとに表示されたもので、匿名初回利用を
維持するため許容する。16件はauthenticatedから実行できる`SECURITY DEFINER` RPCで、各RPCの
ログイン解決・本人所有・対象行・実行権限を照合した。15件は本人所有確認済み。1件、
`confirm_morning_alarm_proposal`が任意入力の`p_device_installation_id`を本人端末へ再照合していなかったため、
app user、`auth.uid()`、platform、有効状態をすべて確認し、確定済みalarmのownerも再確認するforward migrationを
stagingへ適用した。RPCは正常なアプリ機能に必要なため、Advisorの形式警告を消す目的で実行権限は剥奪しない。
残る1件は漏洩パスワード保護OFFで、現行の匿名ログインにはパスワード経路がないため、パスワード認証導入前の
必須設定とする。関連9 test suiteとdiff検査はPASS。Performance Advisorの52件とDB lintの未使用変数1件は
別監査対象であり、本更新では変更していない。productionは未変更。）

（v5.162: A8-2の録音準備音ON/OFF設定を実装した。`settings.recording_ready_cue_enabled`は
既定ONとし、設定画面で保存して全通話開始時にAndroid・iOS nativeへ同じbooleanを渡す。
OFF時は音声出力だけを省略し、Androidは既存の140ms、iOSは既存音長相当の100msを待ってから
従来と同じ録音再開callbackへ進む。停止・reset時は遅延処理を取消し、終了後の録音再開を防ぐ。
TypeScript、対象3 test suite、Web build、diff検査はPASS。DB migrationはstaging適用済み。
native compile、両OSの設定ON/OFF実機確認、productionは未実施。）

（v5.161: A10-2を、全settings利用者を毎分走査して文章まで先行生成する試作方式から、
有効かつ通知許可済みPush tokenを持つ利用者だけを対象にするJIT方式へ変更した。30分ごとのplannerは
利用者の現地日付ごとに原子的な計画済み印を一度だけ取得し、8:00〜22:00の許可範囲から1分単位の
不規則な時刻を保存する。1分cronはDBのdue存在確認だけを行い、期限到来がなければEdge Functionを
起動しない。期限到来時に最新の日次要約と直近チャットを読み、通知文を送信直前に生成する。生成中に
利用者から同じキャラへの発信・チャットがあれば候補を破棄し、3時間後へ後ろ倒しして次回は新しい文を
生成する。通知生成は1回再試行後に終了し、通知枠は30分超、着信枠は2分超の遅延を送らない。
着信の30秒期限はscheduler間隔ではなく、第一声生成後にdispatch workerが実際の着信配信を始めた時点から
設定する既存契約を維持した。旧prototype枠と旧通知候補は監査用に保持するが、新方式の送信対象から除外した。
stagingへmigration 7件と`common-contact-scheduler`、`daily-notification-worker`、
`notification-dispatch-worker`を反映し、dry-runを`true`へ戻した。有効対象は2端末・2ユーザー、
JIT枠は2件（JST 15:56、21:27）、計画済み印も2件、候補本文は未生成であることを確認した。
dueなしRPCは`null`を返しEdge起動なし。ローカルは全52 test suite、TypeScript、Web build、
3 Function bundle、diff検査がPASS。実際の時刻到来時の実通知・実着信とproductionは未確認・未変更。）

（v5.160: staging実データでA10-2のdry-run着信判定を確認した。残高基準以上の4ユーザー・8キャラを
今後30日分で非配信予測し、5日・5件の着信判定を確認した。その後、自動cronで実際に
`dry_run_call=true`のdaily枠が1件作られ、通知候補あり・着信候補なしへ安全に振り替わったことを確認した。
切替条件を満たしたため、stagingの`COMMON_CONTACT_SCHEDULER_DRY_RUN`を`false`へ変更した。
手動page 0と自動cronの双方で`dry_run:false`、HTTP 200、timeoutなしを確認し、既存当日枠の冪等再実行では
重複候補0件だった。今後の未割当判定から実着信候補を作る。実着信候補の生成・端末配信・応答、
productionは未確認・未変更。）

（v5.159: A10-2をstagingへ反映した。migration 2件を適用し、`common-contact-scheduler`と
`daily-notification-worker`をdeploy、staging専用secretをEdge FunctionsとVaultへ同期した。
初回の全222ユーザー一括dry-runは79枠を作成した後にpg_netの55秒でtimeoutしたため、30ユーザー×17 shardを
毎分ローテーションする方式へ変更した。修正版は手動page 0で30ユーザーをHTTP 200、
`dry_run:true`、作成0・変更0の冪等再実行まで確認し、自動cronでも共通スケジューラと通知／着信配信workerの
HTTP 200継続を確認した。初回79枠はすべて通知で、実着信は0件。stagingは残高基準以上4ユーザー、
購入・サブスクlot保有0ユーザーのため、dry-run着信候補0件は異常とは判定しない。実着信への切替、
staging実配信・実機確認、productionは未実施。）

（v5.158: A10-2の共通スケジューラをローカル実装した。デイリー／気まぐれを同じ冪等な割当へ統合し、
lover全員＋非lover 1枠、気まぐれの関係値重み、通知／着信振分、1日1着信、無料のみの着信率低下、
休眠減衰、3段階の連絡量、現地時刻抽選、既定夜間・quiet hours・モーニング・同一／別キャラ近接回避、
ユーザー発信後の後ろ倒し／当日中止、dry-runを実装した。`contact_schedule_slots`、
`settings.contact_volume_level`、`users.last_active_at`、活動更新RPC、30ユーザー×17 shardの毎分割当cronと両配信workerの毎分cronを
追加した。暫定値はF3差替え用moduleへ隔離した。`npm test`、`npx tsc --noEmit`、`npm run build`、
`git diff --check`はPASS。migration適用、Function deploy、vault/secret設定、staging実配信、実機確認は未実施。）

（v5.157: spec v5.33で通知・着信の共通スケジューラ（A10-2）を確定したことに伴い、本書の記述を直した。
フェーズ3へ「通知・着信の共通スケジューラ（A10-2）」を独立した項目として追加し、通知（A10）に残っていた
「自動cron有効化」と、日常の気まぐれ着信（A10-1）に残っていた「本番共通スケジューラ」をここへ集約した。
スケジューラを作る工程が、通知の行では「イベント・告白工程で実装する」、着信の行では「本番共通スケジューラ」と
二通りに書かれていた矛盾を解消した。あわせて、モーニング（A9）の行が「キャラ選択時のTTS＋固定音声同時準備」を
完了と書き、「キャラ選択時のモデル事前取得」節が同じ内容を実機確認待ちの`[~]`としていた食い違いを直し、
節の`[~]`を正とした。本更新は文書のみで、コード、DB、migration、Supabase、Edge Function、R2、env、secret、
productionは変更しない。）

（v5.156: AAB 1104をAndroidへ導入した。アプリがバックグラウンドに残るロック中では、全画面着信、
応答後の通話画面維持、第一声、録音準備音、通常会話まで成立した。履歴からスワイプ終了後のロック中でも
candidate `7ac08d99-768b-4060-aa7e-36e94ec70dc7`で、一瞬の通知から直ちに全画面へ切り替わり、
以後の応答・第一声・録音準備音・通常会話はすべて成立した。candidateとdeliveryも同一installationで
最終`answered`、errorなし。終話後はロック画面へ戻ったが、端末自体のロックを解除せず
`showWhenLocked`の通話Activityだけを終了する正しいセキュリティ動作である。ユーザーは全画面前の
一瞬の通知を許容範囲と判断した。AAB 1104はアプリ前面、別アプリ前面、ロック中、スワイプ終了後ロック中の
実機要件を満たした。記録だけの変更ではpush・再ビルドしない。USB、ローカルGradle、production、DB、
migration、Function、secret、iOSは変更していない。
／v5.155: ロック応答後のActivity表示維持修正commit `812c754`をAndroid On-Device TTS Test run
`30608630277`で検証し、2分50秒でSUCCESSした。Android Staging AAB run `30608785231`も
10分40秒でSUCCESSし、versionCode `1104`、versionName `staging.1.0.104`、artifact
`android-staging-aab-1104`（80,770,313 bytes、artifact ID `8784800197`）の生成・アップロードを
確認した。AAB 1104の端末導入とロック実機再確認は未実施。この記録だけではpush・再ビルドしない。
production、DB、migration、Function、secret、iOSは変更していない。
／v5.154: AAB 1103をAndroidへ導入し、ロック中にcandidate
`3bd9089c-d7da-4776-b832-ebc0eaa9ff96`を1件送信した。full-screen intentの全画面から応答すると
ロック画面へ戻り、解除後は`キャラが話しています`で固着して第一声は聞こえなかった。サーバー上は
candidateとdeliveryが`answered`、errorなしであり、v5.152の誤った`backgrounded`終了は解消していた。
原因は応答時の`stopNativeSpontaneousCallRinging()`が着信音・通知だけでなく
`setShowWhenLocked(false)`と`KEEP_SCREEN_ON`解除まで行い、通話開始中のActivityをロック画面の裏へ
隠していたこと。応答時は`keepActivityVisible=true`で着信信号だけを停止し、通話終了時に通常の
`stopNativeSpontaneousCallRinging()`を必ず呼んでロック上表示を解除するよう、表示所有権を通話終了まで
延長した。ローカル全594テスト、TypeScript、Web build、diff検査はPASS。修正版GitHub native test、
新AAB、ロック実機再確認は未実施。USB、ローカルGradle、production、DB、migration、Function、
secret、iOSは変更していない。
／v5.153: ロック画面応答修正commit `dceb6d7`をAndroid On-Device TTS Test run
`30607125254`で検証し、2分47秒でSUCCESSした。Android Staging AAB run `30607267404`も
11分30秒でSUCCESSし、versionCode `1103`、versionName `staging.1.0.103`、artifact
`android-staging-aab-1103`（80,770,332 bytes、artifact ID `8784244120`）の生成・アップロードを
確認した。AAB 1103の端末導入とロック実機再確認は未実施。この記録だけではpush・再ビルドしない。
production、DB、migration、Function、secret、iOSは変更していない。
／v5.152: AAB 1102をAndroidへ導入した。アプリ前面では全画面・着信音・バイブ・有効な応答ボタンが
PCM準備後に同時開始し、応答、第一声、録音準備音、通常会話まで成立した。別アプリ前面ではheads-up通知、
通知タップ後の全画面、準備後の応答、録音準備音、通常会話まで成立した。ロック中はfull-screen intentで
全画面表示したが、応答操作時に2回とも不在着信へ終了した。candidate
`fcbd15d7-0899-454d-88ee-01d6340f8f15`と`b713524f-aaa6-4fb0-a5fe-2761aef78d6c`は、いずれも
拒否ではなく`outcome_reason=backgrounded`だった。ロック画面の応答時にAndroidが通知する一時的な
inactiveを、Webの`appStateChange`がユーザーのバックグラウンド移行と誤判定していた。
`backgrounded`終了は、アプリ前面でPCM準備中のまま実際に非表示となった
`spontaneousForegroundDisplayPending`時だけに限定し、全画面表示済み着信を終了しないよう修正した。
ローカル全594テスト、TypeScript、Web build、diff検査はPASS。修正版GitHub native test、新AAB、
ロック実機再確認は未実施。USB、ローカルGradle、production、DB、migration、Function、secret、iOSは
変更していない。
／v5.151: 前景表示待機guard修正commit `ed43d4f`をAndroid On-Device TTS Test run
`30605206597`で検証し、3分5秒でSUCCESSした。Android Staging AAB run `30605346779`もSUCCESSし、
versionCode `1102`、versionName `staging.1.0.102`、artifact `android-staging-aab-1102`
（80,770,267 bytes、artifact ID `8783527647`）の生成・アップロードを確認した。AAB 1102の
端末導入と実機再確認は未実施。この記録だけではpush・再ビルドしない。production、DB、migration、
Function、secret、iOSは変更していない。
／v5.150: AAB 1101をAndroidへ導入し、アプリ前面中にcandidate
`b8350b7e-803f-48a8-bc8f-154aa0c3c7c4`を1件送信した。Workerは
`queued=1, sent=1, deferred=0, failed=0`。FCM受信とopen RPCは成立したが、candidateとdeliveryは
即`preparation_failed`となり全画面は表示されなかった。原因はv5.148で前景PCM準備中に
`state.view`を`incomingCall`へ切り替えないよう変更した一方、準備関数入口が
`state.view === "incomingCall"`だけを有効条件としていたため、native TTS呼出し前にfalseを返した
こと。既存着信画面または前景表示待機中のどちらも準備を有効とし、バックグラウンド・拒否・世代更新時は
従来どおり無効化する。AAB 1101は前景FAILで追加送信しない。production、DB、migration、Function、
secret、iOSは変更しない。
／v5.149: 前景着信表示・第一声後準備音修正commit `3f184c1`をAndroid On-Device TTS Test
run `30604011585`で検証し、追加JUnitを含め3分2秒でSUCCESSした。Android Staging AAB run
`30604152607`も10分59秒でSUCCESSし、versionCode `1101`、versionName
`staging.1.0.101`、artifact `android-staging-aab-1101`（80,770,211 bytes、
artifact ID `8783143662`）の生成・アップロードを確認した。AAB 1101の端末導入と実機再確認は未実施。
この記録だけではpush・再ビルドしない。production、DB、migration、Function、secret、iOSは変更していない。
／v5.148: AAB 1100をAndroidへ導入し、アプリ前面中にcandidate
`b87905ea-4b33-497a-814f-bf79cf15b993`を1件送信した。Workerは
`queued=1, sent=1, deferred=0, failed=0`。candidateとdeliveryは同じinstallationで
最終`answered`、errorなし。全画面着信、
応答、第一声、`話してください`への遷移、通常会話は成立し、AAB 1099の固着・再接続は解消した。
一方、第一声PCM準備完了まで無音・無振動の全画面が表示され、第一声後の録音準備音も鳴らなかった。
前景受信はPCM準備完了まで現在画面を維持し、完了時に全画面・着信音・バイブ・有効な応答ボタンを
同時開始するよう修正する。録音準備音欠落は、録音開始前の第一声を`recordingGeneration=-1`として
準備音要求から除外し、さらに第一声再生完了時だけ`readyCueCompleted=true`を先行設定して
turnを切り替えていたことが原因。第一声も通常回答と同じ準備音を要求し、その完了を待ってから
次turnへ進める。録音世代がない第一声の準備音callbackは録音Controllerへ渡さず、Runtimeの
次turn進行だけに使う。準備音ON/OFF設定は後工程。AAB 1100は前景総合FAILとし、
修正版native test・新版AABまで追加送信しない。production、DB、migration、Function、secret、
iOSは変更しない。
／v5.147: Android着信第一声修正commit `274b7f4`をAndroid On-Device TTS Test run
`30602271270`で検証し、2分38秒でSUCCESSした。Android Staging AAB run
`30602392887`も10分56秒でSUCCESSし、versionCode `1100`、versionName
`staging.1.0.100`、artifact `android-staging-aab-1100`（80,769,532 bytes、
artifact ID `8782531967`）の生成・アップロードを確認した。AAB 1100の端末導入と実機再確認は未実施。
この記録だけではpush・再ビルドしない。production、DB、migration、Function、secretは変更していない。
／v5.146: AAB 1099をAndroidへ導入し、アプリ前面中にcandidate
`e513bf04-05eb-44c0-98e5-236d2c72f37d`を1件送信した。Workerは
`queued=1, sent=1, deferred=0, failed=0`、candidateとdeliveryは同じinstallationで
`answered`、errorなし。前面から直接全画面表示され、強制終了せず応答・第一声再生まで成立した。
一方、第一声PCM準備完了まで応答ボタンの待ち時間が長く、第一声は端末を耳へ運ぶ前に始まり、
再生後は`キャラが話しています`で固着して`再接続中`へ落ちたため通常会話FAILとする。
コード上の原因は、第一声用の次turn standbyが先にreadyとなり、その後に再生完了した場合、
`NORMAL_PLAYBACK_COMPLETED`側が`maybeAdvanceTurn()`を呼ばず次turnへ進まない競合である。
再生完了側でも必ず進行判定するよう修正し、アプリ前面受信だけは第一声PCM準備完了後に着信音を
開始する。応答後は既存モーニング基準と同じ1.0秒の受話猶予を置いてから通話接続を開始する。
AAB 1099は通常会話FAILであり、修正版native test・新版AAB前の追加送信は行わない。
production、DB、migration、Function、secretは変更していない。
／v5.145: v5.144の強制終了修正commit `9ed9302`をAndroid On-Device TTS Test run
`30600727202`でnative compile・Gradle検証しSUCCESSした。Android Staging AAB run
`30600863540`も11分1秒でSUCCESSし、versionCode `1099`、versionName `staging.1.0.99`、
artifact `android-staging-aab-1099`（80,769,966 bytes、artifact ID `8782009009`、
digest `sha256:195b08d235176c17eba470894dc171ed0c9d1ac4785c1e4403c193074cee450a`）の
生成・アップロードを確認した。AAB 1099の端末導入と実機再確認は未実施。
production、DB、migration、Function、secretは変更していない。
／v5.144: AAB 1098をAndroidへ導入し、通常通知・全画面通知許可済み、アプリ前面中で
candidate `11589c0f-ae88-4267-8d70-20a20d08c1f9`を1件送信した。Workerは
`queued=1, sent=1, deferred=0, failed=0`、deliveryは`sent`・errorなしだったが、端末では
アプリが落ち「内部エラーのため強制終了しました」と表示された。candidateは`ringing`のまま、
`opened_at`、`response_expires_at`、`opened_installation_id`がすべてnullなので、open RPC前の
native移行中のFAILとする。v5.141で追加した`SpontaneousIncomingCallPlugin.stopRinging()`が
Capacitor plugin threadからActivityのwindow flagsを直接変更し、着信画面へ移った直後に
ロック画面表示用flagsまで解除するコード上の欠陥を修正した。Serviceの音・通知停止とActivity
flags解除を分離し、着信表示中はflagsを維持し、終了時のflags解除だけをAndroid UIスレッドで行う。
AAB 1098は実機FAILであり、修正版native test・新版AAB作成前の再送は行わない。
USB接続、production、DB、migration、Function、secretの変更は行っていない。
／v5.143: Android Staging AAB run `30564975505`でv5.142までの修正を署名付きstaging AABへ
buildした。runは11分6秒でSUCCESSし、versionCode `1098`、versionName `staging.1.0.98`、
artifact `android-staging-aab-1098`（80,769,191 bytes、artifact ID `8768747167`、
digest `sha256:f4686a4ab21622abdc3cb622581f5c7a7d3ecf1fca0c08a254a79b3498741bca`）の
生成・アップロードを確認した。AABの配布、端末導入、Android実機3状態確認、
Play Consoleのfull-screen intent申告は未実施であり、Android実機PASSにはしない。
USB接続、ローカルGradle、production、DB、migration、Function、secretの変更は行っていない。
／v5.142: v5.141のAndroid修正をGitHub Actionsでnative検証した。初回run `30563593643`で
`MainActivity.onResume/onPause`のアクセス修飾子を検出し、Capacitor `BridgeActivity`に合わせて
`public`へ修正した。修正版run `30563888532`ではCapacitor Android sync、Android Java compile、
Gradleの正式voice-call unit testsを含む全工程がPASSした。ローカルGradleとUSB接続は使用していない。
AAB作成、配布、Android実機確認、Play Consoleのfull-screen intent申告は引き続き未実施であり、
Android実機PASSにはしない。
／v5.141: AAB 1097の通知再表示原因を修正した。Android前面ではForeground Service通知を
作らず直接アプリ内着信へ進み、前面外だけ通話通知を一度開始する。通知content/full-screen intentから
Serviceを再開始するフラグとMainActivity再開始処理を削除し、重複FCMでも通知段階30秒を延長しない。
Androidだけは有効期限内の着信画面とnative着信音をopen RPCより先に開始し、Activityへ移った時点で
Foreground Service通知を終了する。同一candidateのlistener/consume重複処理も抑止した。
通常通知と全画面通知の許可状態を初期設定・設定画面へ別表示し、Android 14以降は説明後のユーザー操作で
特別アクセス設定を開き、未許可では着信設定を完了扱いにしない。Google Playのfull-screen intent要件に
合わせ、着信ONの通話通知だけに使用し、未許可時はheads-upへ縮退する。A8の旧「通知タップ限定」記述も
A10-1へ統一した。iOSの通知・着信経路は変更せず、Android即時表示がAndroid限定であることを回帰検査した。
ローカルで全594テスト、TypeScript、Web build、Capacitor Android sync、diff検査はPASS。
Chromebookへ負荷をかけない運用に従いnative compileはローカル実行せず、GitHub Actionsで行う。
AAB作成、配布、実機確認、
Play Consoleのfull-screen intent申告は未実施であり、Android実機PASSにはしない。USB接続は検証手段に
含めない。production、DB、migration、Function、secretは変更していない。
／v5.140: Android `staging.1.0.97`（versionCode `1097`）を実機へ導入し、
candidate `5b0ab079-0273-4b4c-a484-b292f79b6ab3`をAndroidだけへ1件送信した。
配信はcandidate `ringing`、delivery `sent`、Worker
`queued=1, sent=1, deferred=0, failed=0`まで成功したが、実機では通知を何度タップしても
通知が出続け、おそらく約30秒で消え、全画面着信画面へ正常遷移しなかった。
送信時の前面・ホーム・他アプリ・ロック状態と通常通知・全画面通知許可状態は記録できていないため、
いずれも確認済みとしない。応答、第一声準備、通常会話には未到達。
コード上は、通知content/full-screen intentの`startSpontaneousCallRinging=true`を
`MainActivity`が受けて既に稼働中の`SpontaneousCallService`を再度`ACTION_START`し、
通知と30秒タイマーを作り直す経路が症状と整合するが、実機ログ未取得なので原因は未確定。
AAB 1097は実機FAILを維持し、追加送信・修正・調査はユーザー指示により停止して引き継ぐ。
production、DB schema、migration、Function、secretは変更していない。
／v5.139: v5.138修正版commit `f96e078dbae1aa4033029504e0f74d4f5ed294c4`を
Android Staging AAB run `30550118170`でnative compile・署名した。runはSUCCESS、
versionCode `1097`、versionName `staging.1.0.97`、artifact `android-staging-aab-1097`を生成した。
全51テスト、TypeScript、Web build、Capacitor Android sync、native compile、署名、artifact uploadは
PASS。実機への導入、通常通知・全画面通知の初期許可導線、前面直接全画面、未ロックheads-up、
許可済みロック中の自動全画面、open RPC、第一声準備、応答、通常会話は未確認であり、
Android実機は未完了のまま維持する。AAB 1096は使用しない。production、DB、Function、secretは変更していない。
／v5.138: v5.136でAndroidを通知タップ限定へ変更した判断を撤回した。正しい製品要件は、
アプリ前面ではFCM受信時に直接全画面、前面外ではfull-screen intentを付け、ロック中かつ
全画面通知許可済みなら自動全画面、未ロックでホーム画面・他アプリ表示中または権限なしの場合だけ
heads-up通知へ縮退する。通常通知とAndroid 14以降の全画面通知を新規初期設定で案内し、
既存ユーザーの着信設定有効時の起動と着信設定ON時にも同じ確認を通し、設定画面から戻った時点で
再確認する。許可未完了では着信設定を
保存完了扱いにしない。v5.137のAAB 1096は誤った通知タップ限定実装なので実機確認対象にしない。
RPC詳細診断、第一声準備中の応答無効・スピナー・拒否有効は維持する。production、DB、Function、
secretは変更しない。修正版の自動検証、native build、実機確認は未実施。
／v5.137: v5.136実装commit `93ad6937f38337b3edaabcdacd7e5b82a612f7b6`を
Android Staging AAB run `30547572298`でnative compile・署名した。runは10分56秒でSUCCESS、
versionCode `1096`、versionName `staging.1.0.96`、artifact `android-staging-aab-1096`
（80,767,913 bytes、artifact ID `8761712581`）を生成した。実機への導入、通知段階、
通知タップ後の全画面表示、open RPC、第一声準備、応答後の第一声と通常会話は未確認であり、
Android実機は未完了・FAILのまま維持する。production、DB、Function、secretは変更していない。
／v5.136: Android `staging.1.0.95`で発生した気まぐれ着信open失敗を引き継いだ。
対象candidate `0c844a17-7d65-40b4-9e32-bc232ebcb846`をStagingで読み取り、FCM配信は
`sent`まで成功した一方、候補は一度も`opened`にならず`notification_timeout`で不在着信へ
確定した事実を確認した。次回の実機失敗から原因を特定できるよう、open RPCの候補ID、
installation ID、native context受信経路、Supabaseのcode/message/details/hintをStaging診断へ
追加した。v5.135のUI要件を実装し、第一声PCM準備中は拒否を有効なまま、応答だけをスピナー付き
`準備中…`として無効化し、成功時だけ`応答`へ切り替える。準備前の旧auto-answerは廃止した。
Androidはv5.134の二段階契約へ戻し、FCM受信ではActivityやfull-screen intentを自動起動せず、
`{キャラ名}から着信`／`タップして確認`の30秒通知に留め、通知タップ後だけMainActivityへ進む。
ローカルでNode全51 suites、TypeScript、Web build、Capacitor Android sync、diff検査がPASS。
DB、migration、Function、Staging設定、productionは変更していない。native compileと新版実機確認は未実施。
／v5.135: 日常の気まぐれ着信の後続UI要件を確定した。通知タップ後は第一声PCMの生成完了を
待たず、着信音と全画面着信を直ちに開始する。別ローディング画面は設けず、全画面着信内で
拒否を常時有効、応答だけを無効化し、スピナーと`準備中…`を表示する。準備完了時に`応答`へ
切り替え、応答後は待たず通常通話画面へ移って合成済み第一声を再生する。生成進捗による画面遅延、
固定ローディング時間、準備前操作の予約・自動応答は採用しない。本更新は仕様・計画のみであり、
画面コード、native、DB、Staging、production、ビルドは変更しない。
／v5.134: v5.133の確定要件を実装した。通知段階30秒と通知オープン後30秒をDBで分離し、
期限切れ、拒否、バックグラウンド移行、第一声準備失敗を候補ごとの不在着信1件へ集約する。
配信workerは通知前に日時・関係性・記憶・最近の文脈から第一声テキストを生成し、生成成功時だけ
有効端末1台へ配信する。通知タップ後は両OSのオンデバイスTTSで第一声をPCMまで合成し、1回再試行する。
準備前の応答操作は完了まで待ち、成功後は通常通話の既存サーバーturnへ合成済みPCMを渡して再合成しない。
応答確定前に既存の通話最低残高を再確認し、不足時はcall・録音・課金を作らず不在着信へ確定する。
Android Foreground Serviceにも30秒の端末側停止を追加し、気まぐれ着信だけはfull-screen intentの自動起動と
通知上の直接応答／拒否を外して通知タップ後に第二段階へ入るよう統一した。TypeScript、worker bundle、Node全テスト、
両nativeの正式コンパイル、Staging migration／Function反映、実機確認は検証結果に従って別途記録する。
暫定Staging worker version 9はまだ単純60秒のままであり、このローカル実装は未反映。
／v5.133: 実機結果を受けた最終要件を確定した。単純な配信開始から60秒ではなく、
通知段階30秒と通知オープン後の全画面30秒を分離する。通知は`{キャラ名}から着信`／
`タップして確認`とし、期限後タップは対象トークの不在着信を開く。通知無視、拒否、全画面無操作、
第一声準備失敗はいずれもcall・録音・課金なしで冪等な不在着信履歴を残す。第一声テキストは
通知前に生成し、通知タップ後の全画面着信中にオンデバイスTTSで準備して、応答後は通常通話画面で
直ちに再生する。製品上は1ユーザー1有効端末とし、移行時は旧端末を無効化する。
残高による着信／通常メッセージ振分基準は共通スケジューラ工程で決める。実装・検証は未実施。
暫定のStaging worker version 9は単純60秒のため、最終二段階契約へ置き換える。
自動スケジューラ、cron、productionは変更していない。
／v5.132: 最新TestFlight実機でiOS通常通知、28.5秒の通知音、通知タップ、アプリ内疑似着信表示まで
成立した。一方、配信開始から29秒の候補期限ではアプリ起動・読込・応答操作の猶予が足りず、
応答RPCが`spontaneous_call_expired`となる実機不具合を確認した。候補と応答の有効時間を両OS共通で
60秒へ延長し、期限切れ・別端末応答済み・着信終了済みを一般エラーと区別する表示を追加した。
iOS通知音の種類変更は後続の素材工程へ分離する。自動テスト全584件、TypeScript、Web build、
iOS Capacitor sync、配信worker bundle、diff検査はPASSした。Staging workerをversion 9へ更新し
ACTIVEを確認した。native buildと実機再確認は未実施。自動スケジューラ、cron、productionは変更していない。
／v5.131: commit `10ed6ff`のAndroid Staging AAB run `30467617144`は11分6秒でSUCCESSし、
versionCode `1087`、versionName `staging.1.0.87`、artifact `android-staging-aab-1087`
（80,763,172 bytes、artifact ID `8730596663`）を生成した。Stagingの
`spontaneous-call-dispatch-worker`を通常APNs alert方式へ更新し、version 6・ACTIVEを確認した。
自動スケジューラとcronは無いため、自動着信は開始していない。iOS Codemagic buildは
自動起動されず、手動起動とTestFlight実機確認を残す。productionは変更していない。
／v5.130: 実機結果を受け、iOSの日常の気まぐれ着信からPushKit／CallKitを撤去し、
通常APNs alertからアプリ内疑似着信へ進む方式へ変更した。バックグラウンド・終了中・
ロック中は28.5秒の同梱WAVを通知音に指定し、タップ後は同じWAVをアプリ内でループして
バイブを反復する。応答・拒否で停止する。前面受信は通知を重ねず直接疑似着信を開く。
候補期限を配信開始から29秒へ短縮し、期限切れ着信を通常メッセージとして誤処理しない。
Androidの専用FCM／Activity／Foreground Service／通知縮退は維持する。iOS native compile、
新方式の両OS実機確認、Staging worker更新は未実施。自動テスト全581件、TypeScript、
Web build、両OSCapacitor sync、配信worker bundle、diff検査はPASSした。自動スケジューラ、
cron、productionは変更していない。v5.129で確認したiPhoneのCallKit画面は新要件の成立確認には数えない。
／v5.129: 日常の気まぐれ着信のDBと配信workerをStagingへ接続した。
設定3本と着信基盤1本のmigrationをproject `nqqkbwhrwinxuameyzya`へ適用し、履歴一致を確認した。
FCM、APNs、worker認証secretが揃っていることを値を表示せず確認し、
`spontaneous-call-dispatch-worker`をACTIVEにした。AndroidとiPhoneは別の匿名アカウントだが、
ユーザーによる両アプリ更新・起動後、AndroidはFCM token、iPhoneは通常APNs tokenと
独立VoIP tokenが各1台分登録された。自動着信専用secretを追加し、既存の通知／memory worker
secretも引き続き受理する。手動Staging候補をAndroid・iPhoneへ1件ずつ作り、providerは
両方を1回で受理し、candidate=`ringing`、delivery=`sent`、errorなしを確認した。
同じworker実行中に作ったdeliveryのDB既定時刻がworker開始時刻より数ミリ秒後となり、
初回送信対象から外れる不具合を実機試験前に検出した。queue時に`next_attempt_at`を
worker時刻へ固定して修正し、自動テストは全580件PASSした。端末での着信表示・応答・拒否は
確認待ち。自動スケジューラ、cron、productionは変更していない。
／v5.128: Androidの日常の気まぐれ着信を署名付きStaging AABでnative compileした。
初回run `30457929504`は専用FCM ServiceからFirebase Messaging型を参照するapp compile依存が
無く失敗し、依存を明示した。2回目run `30458502612`は保存済み着信contextのJSON変換に
checked exception処理が無く失敗し、不正contextを安全に破棄する処理を追加した。
修正後commit `45dce87`のrun `30459049188`は11分1秒でSUCCESS、versionCode `1086`、
versionName `staging.1.0.86`、artifact `android-staging-aab-1086`の生成・アップロードまで
PASSした。Android実機確認、iOS Codemagic native compile、staging migration／Function／
provider secret、共通スケジューラは未実施。productionは変更していない。
／v5.127: 日常の気まぐれ着信の共通サーバー契約と両OSの受信経路を実装した。
候補・端末別配送・期限・先着応答・拒否・無応答を専用テーブルと認証RPCへ分離し、
応答前はcall・録音・AI・課金・通話見出しを作らない。配信workerはAndroidの
high-priority data FCMとiOSのVoIP APNsを分ける。Androidはモーニングと別のActivity・
Foreground Service・通話通知を使い、Android 14以降で全画面特別アクセスが無ければ
通知へ縮退する。iOSは通常APNs tokenと別にPushKit tokenを保存し、VoIP pushをCallKitへ
即時報告する。AlarmKitは使わない。Web側は気まぐれ着信をイベント着信と別の
`spontaneous`発信元にし、認証済み応答後だけ既存AI通話へ合流する。`npm test`全51テスト
ファイル、`npx tsc --noEmit`、`npm run build`、両OSのCapacitor sync、配信provider bundle、
`git diff --check`はPASS。native compile、staging migration／Function反映、provider secret、
両OSのバックグラウンド・終了中・ロック中実機確認、候補を作る共通スケジューラは未実施。
productionは変更していない。
／v5.126: 次工程を「Android・iOSの日常の気まぐれ着信完成」へ変更した。着信を、
着信設定と専用サイレント時間に従って直接かかる日常の気まぐれ着信と、通知・疑似LINEで
明示確認した後だけ始めるイベント／告白着信に分離する。Androidはモーニングと分離した
通話用high-priority FCM・full-screen intent・Foreground Serviceを使い、許可が無い場合は
通話通知へ縮退する。iOSはPushKit＋CallKitを使い、AlarmKitは登録済みモーニング専用として
流用しない。候補・端末別配信・応答／拒否／無応答の共通サーバー契約を先に作り、
Android、iOSの順に同じ機能単位で実装・実機確認する。応答前のcall作成、録音、AI接続、
課金、通話見出しは禁止する。両OS成立後に、通知と着信を一緒に割り当てる自動スケジューラへ
進む。productionは変更しない。
／v5.125: Android Staging AAB 1083を実機へ導入して再確認したが、トークを開いた際の
初期表示位置は期待どおりにならず、修正成立とは扱わない。日時・日付区切り・新着表示と、
v5.120までに両OS実機確認済みの通知機能は維持する。初期表示位置は今回のmainマージを止めず、
画面制作工程で改めて扱う。iOSでの初期表示位置確認は実施しない。DB、migration、Edge Function、
自動cron、productionは変更していない。
／v5.124: 最新位置修正commit `2779851b65a29189a55894374ca0cb65c8be91c2`を
Android Staging AAB run `30440083582`でbuildした。runは11分1秒でSUCCESS、versionCode `1083`、
versionName `staging.1.0.83`、artifact `android-staging-aab-1083`、size `80,760,370 bytes`、
artifact digest `sha256:6ea9b3e87134fdd67b809d36057334d33ead6ca2c2b0aac51e9574662a79aa7b`。
Android実機への導入、iOS Codemagic staging build、両OSでの最新位置再確認は未実施。
DB、migration、Edge Function、自動cron、productionは変更していない。
／v5.123: 新着・日時表示の実機確認で、日付・時刻は表示されたが、新着がないトークを開いた際に
最新位置まで移動しない場合があった。履歴DOMの更新直後に1回だけ`scrollHeight`を参照しており、
Android／iOS WebViewのレイアウト確定前の高さを使えることが原因だった。初期位置の適用を
2 animation frame後まで待ち、80ms後にも同じ位置を再確認する。新着なしは
`scrollHeight - clientHeight`、新着ありは最古新着の実際の画面内座標から位置を決め、
古い画面の遅延処理はgenerationで無効化する。DB、migration、Edge Functionは変更しない。
`npm test`全50テストファイル、`npx tsc --noEmit`、`npm run build`、`git diff --check`がPASS。
修正版native buildと実機再確認は未実施。
／v5.122: 新着・日時表示commit `5b0cb9e9f9024088858deb0048637c3e562e3ead`を
Android Staging AAB run `30438057591`でbuildした。runは8分53秒でSUCCESS、versionCode `1082`、
versionName `staging.1.0.82`、artifact `android-staging-aab-1082`、size `80,760,317 bytes`、
artifact digest `sha256:b0f7c204e7cfe9ed12d12d1bfd60b3fcaed84ec24f7f6093cb24f2d7f5d327e8`。
Android実機への導入、iOS Codemagic staging build、両OS実機確認は未実施。
DB、migration、Edge Function、自動cron、productionは変更していない。
／v5.121: 疑似LINEの見やすさ改善として、既存`chat_messages.created_at`を使い、送受信本文、
通知入口、通話見出しへ現地時刻を表示し、日付の変わり目へ「今日」「昨日」「月日（曜日）」の
区切りを表示する。端末内へインストール識別子単位のアプリ初回起動時刻と
ユーザー・キャラ別の最終トークオープン時刻を保存し、前回オープン後の履歴だけを「新着」とする。
未訪問キャラは端末のアプリ初回起動時刻を基準にし、新着があれば最古の新着、新着がなければ
最新位置を最初に表示する。読了位置、DB列、migration、Edge Functionは追加しない。
新しい判定・日付整形・最古新着移動のテストを追加し、`npm test`全50テストファイル、
`npx tsc --noEmit`、`npm run build`、Android／iOSの`cap sync`、`git diff --check`がPASSした。
native実機確認は未実施。
／v5.120: Android AAB 1081と修正版iOSを実機へ導入し、前面Pushのキャラ別表示を両OSで再確認した。
花音トークを表示中の花音PushはOS通知も画面内通知UIも出さず、花音の入口本文だけを1件反映してPASSした。
桜音トークを表示中の花音Pushは桜音画面内へ内容を一切表示せず、Android・iOSともOS通知を表示してPASSした。
OS通知タップ後は花音トークへ移動し、本来の入口本文を1件だけ反映した。最終配送行はAndroid
`a07cc264-b645-4a2b-91f1-b253ca98ffd7`、iOS
`e86b9bd9-b9ef-4908-83fe-8befc1243ca3`で、両方とも`sent`、`attempt_count=1`、
`last_error=null`、`outcome=opened_chat`、`entry_message_count=1`を確認した。
自動cronは無効のまま、production変更なし。
／v5.119: 前面PushをOS通知へ直すcommit `7422bc1`をpushし、Android Staging AAB run
`30434369962`で署名付きAABをbuildした。runは12分7秒でSUCCESS、versionCode `1081`、
artifact `android-staging-aab-1081`、size `80,759,119 bytes`、artifact digest
`sha256:18c1503f1d5923f841f1489148c59f5758bd15e9e2a3f3c292672542e0c8ac66`。
Android実機への導入、iOS Codemagic build、両OS実機再確認は未実施。
／v5.118: 両OS実機で同じ花音トークを前面表示中は通知UIなしで入口本文だけが反映されることをPASSした。
一方、桜音トークを前面表示中の花音PushがOS通知ではなく桜音画面上のアプリ内通知UIとして表示され、
製品要件と異なることを確認した。この結果はPASSにしない。別キャラまたはトーク外の前面Pushは、
既存画面へ内容を一切重ねず`LocalNotifications`からOS通知を即時表示し、その通知タップを既存の
候補ID検証・対象トーク遷移へ接続するよう修正した。iOSは前面でbanner/list/soundを許可する。
同一キャラ時の通知抑制と冪等な入口反映は維持する。DB、migration、Edge Function、自動cron、
productionは変更しない。自動検証、native build、両OS実機再確認は未実施。
／v5.117: Android AAB 1080と修正版iOSを実機へ導入し、通知用・着信用それぞれの
サイレント開始・終了が保存されることを両OSで確認した。「トーク内容を表示しない」をONにして、
Android候補 `211ddcc7-881c-4624-bf3f-1e6655cbf3e6`、iOS候補
`b9d33ba4-0c20-4c6b-b15b-80bcd87bfdaf`だけを手動配送した。両OSのロック画面で
本文が「花音からメッセージが届いています」へ置換され、通知タップ後のトーク画面には各候補の
本来の`entry_message_body`が1件だけ表示されることをPASSした。配送行はAndroid
`2a4381cd-1619-430c-9183-a0c24164f33b`、iOS
`3e66f700-e987-4db3-8f3e-132137400845`で、両方とも`sent`、`attempt_count=1`、
`last_error=null`、`outcome=opened_chat`、`entry_message_count=1`を確認した。
自動cronは無効のまま、production変更なし。
／v5.116: Push登録の古い失敗状態を再表示しない修正commit `7281646`をpushし、
Android Staging AAB run `30425836983`で署名付きAABをbuildした。runは11分1秒でSUCCESS、
versionCode `1080`、artifact `android-staging-aab-1080`、size `80,757,618 bytes`、
artifact digest `sha256:586fd51461244a6ae605c6b9f23a23df24f4da0b2f4cb8768e8f895c7f053b09`。
Android実機への導入、iOS Codemagic build、両OSでの再確認は未実施。
／v5.115: AAB 1079とiOS更新後の設定保存で「端末のPush通知登録に失敗しました」と表示された。
設定DB更新はエラー表示より前に成功していたが、起動時など過去の`registrationError`状態を保存操作時に
リセットせず再利用していたため、今回の登録結果ではない古い失敗を表示できる不具合だった。
保存操作の開始時に古い失敗状態を破棄し、保存直前のON/OFFをPush同期へ明示して、OFFからONへ変えた場合も
古い画面状態で無効登録しないようにした。実際に端末登録更新だけが失敗した場合は設定を再読込した後、
「通知設定は保存済み、端末登録更新だけ未完了」と区別して案内する。SQL変更なし。native buildと実機再確認は未実施。
／v5.114: commit `6d19d10`をbranch `agent/daily-notification-foundation`へpushし、
Android Staging AAB run `30424097434`で署名付きAABをbuildした。runは11分5秒でSUCCESS、
versionCode `1079`、artifact `android-staging-aab-1079`、size `80,757,483 bytes`、
artifact digest `sha256:fbcd69dd47fb0352d31021482cd15ed29296907944f0fbbc28286551f268fa33`。
Android実機への導入、iOS Codemagic build、両OSでの分離設定確認は未実施。
／v5.113: migration `20260729092000_split_notification_and_incoming_call_quiet_hours.sql`を
ユーザーがstaging SQL Editorで実行し、Successを確認した。通常通知用の既存時間を着信側へ初期コピーし、
以後は通知用と着信用のサイレント開始・終了を独立して保存できるDB状態まで反映済み。
native buildと実機確認は未実施。
／v5.112: 通常メッセージ通知とキャラ側からの自発的な疑似着信について、サイレント開始・終了を
別々に設定・解除・保存できるUIと保存処理を実装した。既存の`quiet_hours_start/end`は通常通知用として
維持し、`incoming_call_quiet_hours_start/end`を追加するforward migrationを用意した。migration適用時は
従来のサイレント時間を着信側へ初期コピーし、適用直後に着信可能時間が意図せず広がることを防ぐ。
モーニングは両方のサイレント時間から独立する。migrationのstaging適用、native build、実機確認は未実施。
／v5.111: commit `76486d8`をbranch `agent/daily-notification-foundation`へpushし、
Android Staging AAB run `30422118009`で署名付きAABをbuildした。runは10分28秒でSUCCESS、
versionCode `1078`、artifact `android-staging-aab-1078`、size `80,757,465 bytes`、
artifact digest `sha256:b4c2ff4174915f53809a8beea3229efa8d0d2c55fe61cd01acccdc317cd76be0`。
Android実機への導入と通知3項目の確認、iOS Codemagic build・TestFlight確認は未実施。
／v5.110: v5.109の2 migrationをユーザーがstaging SQL Editorで実行し、Successを確認した。
`notification-dispatch-worker`をstaging project `nqqkbwhrwinxuameyzya`へdeployした。
これにより内容非表示設定の配信直前適用と、同一キャラ判定用`character_id`を含むPush payloadの
サーバ反映まで完了した。Webアプリ変更のnative buildと実機確認は未実施。
／v5.109: 通知の次工程として、Push payloadへ`character_id`を追加し、アプリ前面で同じキャラの
トークを表示中は通知UIを出さず、送信ログとの到着競合だけを限定再試行して既存RPCから入口メッセージを
冪等反映するWeb経路を実装した。別キャラまたはトーク外では既存のアプリ内通知UIを維持する。
`settings.notification_content_hidden`と`settings.incoming_call_enabled`のmigration、設定UI、保存処理を追加し、
内容非表示時は配信直前に「{キャラ名}からメッセージが届いています」へ置換する。自発着信設定は通常通知と
独立し、登録済みモーニングには連動しない。migrationのstaging適用、Function deploy、native build、
実機確認は未実施。自発着信の本番スケジューラはイベント・告白工程とともに未実装。
／v5.108: versionCode明示修正commit `ac61e06`をAndroid Staging AAB run
`30366807251`（run number 75）でbuildし、Gradleの`Verified versionCode=1075
versionName=staging.1.0.75`、署名済み`VoiceCompanion-staging-1075.aab`、
番号付きartifact `android-staging-aab-1075`の生成を確認した。AAB 1075をAndroid実機へ導入し、
通知設定保存後に`device_installations`へAndroid・granted・active・FCM tokenありで登録された。
Android端末だけを指定した手動候補はFCM受理1・失敗0となり、端末ロック中の通知受信、
タップによる対象チャット表示、入口メッセージ1件の追加までPASSした。これでiOS／Androidの
バックグラウンド実通知経路はPASS。Androidでアプリ前面中の初回送信は通知欄に表示されず、
`pushNotificationReceived`未処理を残課題として確認した。自動cronはまだ無効。
／v5.107: Android Staging AAB run `30363266417`のartifactをPlay Consoleへ投入すると、
既に使用済みのversionCode `1066`として拒否された。run numberから`1074`になる想定だけを記録し、
完成AABの番号を保証できていなかったため、v5.106の「AAB 1074成立」は撤回する。
再発防止としてversionCode／versionNameをGradle project propertyで明示し、設定値を専用taskで検査して
不一致時はbuildを失敗させる。ZIP artifact名と内部AAB名の両方へversionCodeを付け、
同名`app-release.aab`による取り違えを防ぐ。修正版AAB buildとPlay Console投入は未実施。
／v5.106: iOS実通知確認済みcommit `5f2283b`をAndroid Staging AAB workflow run
`30363266417`（run number 74）でbuildし、Firebase設定を含む署名済みAAB
の生成とworkflow成功までは確認した。ただし完成AABのversionCodeを検査しておらず、Play Consoleで
`1066`として拒否されたため、`1074`のAABが成立したとは扱わない。
自動cronは無効のまま、mainとproductionは未変更。
／v5.105: Codemagic iOS Staging Build index 57をTestFlight実機へ導入し、設定画面でユーザー操作時だけ
通知許可を要求すること、`device_installations`へiOS・granted・active・APNs tokenありで登録されることを
確認した。自動cronは無効のまま手動候補1件を既存queueへ入れ、APNs受理1・失敗0を確認した。
ロック画面通知をタップすると花音の疑似LINEが開き、`notification_logs`は`opened_chat`、
入口メッセージは`source_notification_log_id`に対して1件だけ作られた。iOS実通知はPASS。
残りはAndroidのFirebase入りstaging build・FCM token・実通知、F3正式重み、両OS確認後のcron有効化。
／v5.104: commit `9fb44e8`のCodemagic iOS Staging Build
`6a68a7575c8ba71432377b1f`（index 57）はSUCCESSし、新しいPush対応provisioning profileの適用、
AppTests、署名済み`App.ipa`（43.30 MB）の生成、App Store Connectへのpublishingまで完走した。
iOS native buildは成立し、残りはTestFlight実機での通知許可・APNs token登録・実通知確認。
／v5.103: A10の両OS配信資格情報をstagingへ設定した。Androidは専用Firebase project
`voice-companion-staging`へ`com.ghboasis.voicecompanion`を登録し、`google-services.json`を
アプリへ配置、FCM HTTP v1用service account 3項目をSupabase secretへ保存した。iOSは同Bundle IDの
Push Notifications capabilityを有効化し、既存App Store provisioning profileを再生成した。
Codemagicへ新profileを`voicecompanion-appstore-push`として追加し、staging／production workflowの
参照を切り替えた。VoiceCompanionのProduction topicだけに限定したAPNs key 5項目をstaging secretへ
保存し、Xcode targetへDebug=development／Release=productionの`aps-environment` entitlementを追加した。
秘密鍵はrepositoryへ保存せず、登録後に一時ファイルとDownloads原本を削除した。49テスト、
TypeScript検査、Web build、iOS／Android Capacitor sync、`git diff --check`はPASS。残りはstaging build、
実機token登録と実通知、確認後の自動cron有効化。
／v5.102: A10第二段階として、lover全員の毎日確定、lover不在時の関係値重み付き日替わり1枠、
現地09時台だけの割当、候補への`scheduled_for`設定を実装した。F3確定前の重み式は独立moduleへ隔離した。
設定画面へ通知全体ON/OFF、キャラ個別ON/OFF、quiet hoursを追加し、ユーザー操作時だけOS許可を要求する。
許可済み端末は永続installation IDとAPNs/FCM tokenを`device_installations`へ同期する。iOS AppDelegateへ
Capacitorのremote notification token転送を追加した。通知タップは、送信済み`notification_logs`を検証する
`open_notification_in_chat` RPCが入口メッセージを一度だけ作り、対象キャラの疑似LINEを開く。
端末別の耐久`notification_delivery_attempts` queue、quiet hours再判定、並列claim、指数backoff、
stale lock回復、無効token停止、APNs token認証、FCM HTTP v1認証・送信を実装し、provider受理後だけ
`notification_logs`を作る。migration 2件をstagingへ適用し、`daily-notification-worker` v5と
`notification-dispatch-worker` v2のACTIVE、手動実行で割当0・queue0・送信0の安全な空処理を確認した。
この時点ではstagingにAPNs/FCM secret、iOS Push capability、Android `google-services.json`がまだ無く、
自動cronも未有効化のため実通知は送られない。全自動テスト、TypeScript検査、Web build、
両Function bundleはPASS。
／v5.101: A10第一段階として、デイリー通知文と疑似LINE入口メッセージを同時生成して
`notification_candidates`へ保存する内部Edge Functionを追加した。共通`context_id`、
ユーザー×キャラ×現地日付の冪等キー、通知全体／キャラ個別ON判定、直近7日日次要約・長期記憶・
関係状態の再利用、最近使った話題の参照、ロック画面文面の安全検査と固定文フォールバックを実装した。
候補と配送結果のクライアント書込み権限を除去し、`notification_logs`は実配信受理後だけ作る境界を
固定した。キャラ枠の関係値重み、lover毎日確定、quiet hours、端末トークン、実配信、通知タップ後の
疑似LINE反映は次段階。migrationをstagingへ適用し、`daily-notification-worker` v3 ACTIVEを確認した。
実在する日次要約の対象で候補1件を生成し、通知文・入口メッセージ・共通`context_id`・冪等キーの保存を
確認した。同じ対象の再実行は同じ候補を返し、候補数1、未配信の`notification_logs` 0件を確認した。
全539テスト、TypeScript検査、Web build、Edge Function bundle、`git diff --check`はPASS。
／v5.100: 記憶パイプライン実装後のstaging `voice-turn` v16 ACTIVE後に、確定15ターンの
通話がcompletedまで継続し、10往復または12,000文字で開始する通話内一時要約のターン条件を
実機経路で通過したことをDB行数から確認した。サーバは古い履歴だけを要約して直近4往復を残し、
Android/iOS両方が同じ`history_summary`を受信・次ターンへ再送・終話時破棄する契約を自動検証済み。
箇条書き調の自然さ、日次モデル比較、モーニング固定WAV正式化は機能成立後の品質・素材工程へ分離し、
PR #76のマージ条件には含めない。プロンプトとモデルは本更新で変更していない。
／v5.99: stagingで2026-07-28 03:40〜05:30 JSTの記憶cron 7件が実時刻にすべて成功し、
7月27日分4件の日次runが初回完了、待機・実行中・再試行待ち0件であることを確認した。
食べ物の会話は日次要約と長期記憶へ入り、翌日の実機会話でも保持を確認した。一方、過去会話を
箇条書きのように並べる不自然さは品質課題として残す。本文のない終了通話16件が正式要約の
`pending`を塞いでいたため、通話・ログ・要約行を削除せず`empty`へ終端化し、確定済み本文の遅着時は
DB triggerで`pending`へ戻すmigrationをstagingへ適用した。既存16件は`empty`、`pending`は0。
ROLLBACK付き遅着検査で`empty`から`pending`への復帰を確認し、staging `memory-worker` v2をACTIVE化した。
モーニングの仮固定WAVは後続素材工程で、現在より長く内容のある正式文言へ差し替える。
／v5.98: v5.97の署名済みIPAをTestFlight実機で確認し、iOSモーニングのAlarmKit連携を含む
一連の動作に問題がないことを確認した。iOS実装・native compile・実機確認は完了。
／v5.97: availability修正commit `0711b78`のCodemagic iOS Staging Build
`6a686234b6f0c284b5aa280e`（index 56）はSUCCESSし、AppTestsを含むworkflow完走と
署名済み`App.ipa`（43.30 MB）の生成を確認した。iOS native compileは完了し、残りはTestFlight実機確認。
／v5.96: commit `1ae2106`のCodemagic iOS Staging Build
`6a685fda8134001e1f377f10`はAppTestsのSwift module emitで失敗した。deployment targetのiOS 15に対し、
AlarmKit製品コードの`localeWeekday`だけavailability指定がなく、iOS 16以降の`Locale.Weekday`を
参照していたことが原因。呼び出し元と同じiOS 26限定helperとして明示した。再buildは未実施。
／v5.95: `voice_companion_alarmkit_validation_2026-07-02.md`の検証済み経路を正本として、
Androidで完成した共通WebフローへiOS AlarmKit bridgeを接続した。warm起動時のAlarmKit context取得、
現在鳴っているnative alarmだけの停止／スヌーズ、`応答`のAlarmKit停止完了後の既存AI通話合流、
固定メッセージの共通1.0秒待機とiOS近接監視・自動スリープ抑止・受話口／スピーカー切替を実装した。
設定UI、DB同期、第一声準備、通話、履歴、課金、固定音声選択はAndroidと同じTypeScript層を維持し、
カスタムAlarmKit音、Web Audioループ、iOS専用業務処理は追加していない。全531テスト、TypeScript検査、
Web build、`npx cap sync ios`、`git diff --check`はPASS。iOS native compileとTestFlight実機確認は未実施。
／v5.94: v5.93のAndroid実機で近接消灯が従来より早くなったことを確認した。消灯は固定音声開始後だが
現段階の機能として採用し、Androidモーニングの機能実装はテスト端末で完了。通話画面デザイン・
ボタン配置と固定WAVの長さ・正式文言は後のデザイン・素材工程へ送り、次はiOSモーニングへ進む。
／v5.93: v5.92を含む署名済みstaging AABをAndroid Staging AAB run `30334832842`で作成し、
artifactへ保存した。残る確認は固定メッセージで近接消灯が音声終盤ではなく開始前後に行われるかの実機確認。
／v5.92: v5.91のAndroid native compileと音声通話テストをAndroid On-Device TTS Test
run `30334620063`で確認しPASS。署名済みstaging AABと実機再確認は未実施。
／v5.91: v5.90の実機で全画面表示と通知からの応答はPASS、AI第一声は体感上の劣化なし。
固定メッセージの近接消灯が音声終盤まで遅れるため、近接監視を通話画面表示直後へ移し、1.0秒待機と
再生準備をまたいで維持した。全529テスト、TypeScript検査、Web buildはPASS。Android native compile、
署名済みstaging AAB、実機再確認は未実施。
／v5.90: v5.89までの全画面・通知応答・固定メッセージ近接消灯・AI第一声準備修正を含む
署名済みstaging AABをAndroid Staging AAB run `30332303567`で作成し、artifactへ保存した。
残るAndroid確認は4症状の実機再確認。
／v5.89: v5.88のAndroid native compileと既存音声通話unit testsを
Android On-Device TTS Test run `30332108765`でPASSした。署名済みstaging AABと、
全画面・通知応答・固定メッセージ近接消灯・AI第一声時間の実機再確認は未実施。
／v5.88: Android実機で、全画面疑似着信が不安定、通知の`応答`が設定画面を表示、
固定メッセージで近接消灯しない、AI第一声の短縮を体感できない不具合を確認した。着信Activityを
専用taskへ分離し、新着Intent listenerと通知からの自動応答を追加した。固定メッセージへ通常通話と
同じ近接画面制御を接続した。第一声はユーザーデータ読込前から準備し、応答時の世代無効化をやめ、
通話開始処理と並行中なら最大0.5秒だけ合流を待つ。全529テスト、TypeScript検査、Web buildはPASS。
Android native compile、署名済みstaging AAB、4症状の実機再確認は未実施。
／v5.87: v5.86までの1.0秒待機とAI第一声先行生成を含む署名済みstaging AABを
Android Staging AAB run `30329109721`で作成し、artifactへ保存した。残るAndroid確認は、
固定メッセージの1.0秒待機の体感と、応答から第一声再生までの実機時間。
／v5.86: v5.85のWeb・TypeScript・全テスト・Android native compileを完了した。
Android On-Device TTS Test run `30328884570`はPASS。`voice-turn`はstaging version 18へ
反映しACTIVEを確認した。第一声先行生成を含む認証済み実機確認、iOS native compile、
署名済みstaging AABは未実施。
／v5.85: `メッセージを再生`の待機を0.8秒から1.0秒へ変更した。単独でAABを作り直さず、
モーニング着信中のAI第一声先行生成を同時に実装した。DB残高が通話最低額以上の場合だけ
`voice-turn`の認証付きHTTP経路で日時・関係性・記憶を含む第一声文章を準備し、応答時に
準備済みならWebSocketへ渡してLLM再生成を省く。準備失敗・未完了は従来生成へ戻し、応答時の
最新残高確認、拒否時のcall未作成・コイン未消費は維持する。ローカル検証、staging deploy、
両OS native compile、AAB、実機確認は未実施。
／v5.84: v5.83の0.8秒待機を含む署名済みstaging AABをAndroid Staging AAB
run `30327306505`で作成し、artifactへ保存した。残る確認は0.8秒待機の実機体感。
／v5.83: 実機で全画面疑似着信と通知の操作ボタンを確認した。`メッセージを再生`は受話器を
耳へ当てる前に冒頭が始まるため、通常通話画面の表示後0.8秒待ってから固定音声を開始するようにした。
待機中に終了した場合は再生せず、再再生ボタンと自動ループは追加しない。Web/構造テスト、
TypeScript検査、Web build、修正版AAB、0.8秒の実機確認は未実施。
／v5.82: v5.81の全画面優先修正を含む署名済みstaging AABをAndroid Staging AAB
run `30293713508`で作成し、artifactへ保存した。残るAndroid確認は修正版AABの実機再確認。
／v5.81: v5.80の全画面優先修正をAndroid On-Device TTS Test run `30293438751`で
native compileし、既存の音声通話テストを含めてPASSした。修正版AABと全画面の実機再確認は未実施。
／v5.80: Android実機で`応答`のAI第一声と`メッセージを再生`の通常通話画面遷移を確認した一方、
設定画面を表示してスリープした場合に全画面疑似着信へ入らず、停止もできない経路が見つかった。
全画面表示の実効許可が有効化条件から漏れていたため、Android 14以降では未許可時に端末設定を開き、
許可後の再保存を必須にした。鳴動時はfull-screen intentに加えて専用Activityの直接起動も試し、
通知には`応答`と`停止`を追加した。Web/構造テスト、TypeScript検査、Web buildはPASS。
Android native compileと修正版AAB、実機再確認は未実施。
／v5.79: v5.78のAndroid native compileとstaging反映を完了した。Android On-Device TTS Test
run `30288057765`でnative compile、既存音声通話テスト、AI第一声のuser時間0ms／実再生時間だけを
加算するusageテストをPASSした。staging `voice-turn` version 17へAI第一声生成をdeployし、
ACTIVEを確認した。Android staging AAB workflow run `30288366458`で署名済みAABのbuildとartifact
保存に成功した。残る確認はAndroid実機での`応答`／`メッセージを再生`両経路とiOS native compile
である。productionは変更していない。
／v5.78: モーニング疑似着信後の2経路をspec v5.7へ合わせた。`応答`は固定WAVを使わず、
現在日時・関係性・記憶を含む既存の通話system promptからAIが毎回短い第一声を生成する。
両OSとも第一声の再生完了までは録音せず、AI音声の実再生時間だけを通常通話usageへ加え、
次のturnから通常会話へ進む。`メッセージを再生`は無料の保存済みWAVを通常の通話画面で
受話口／スピーカー切替付きで再生し、ユーザーが終了するまで画面を維持する。終了後はどちらも
既存と同じ「通話」見出しを使う。Androidで固定音声再生中だけ残っていたforeground通知経路を
削除した。Web/Function/構造テスト、TypeScript検査、Web buildはPASS。Android native compile、
iOS native compile、staging `voice-turn` deploy、実機確認は未実施。
／v5.77: 固定モーニング音声の生成対象から花音が漏れていたため、花音の正式AIVMXモデルを
manifestへ追加し、7キャラ×3本＝21本を再生成した。GitHub Actions run `30280685835`で生成・
WAV検証に成功し、花音3本のartifact実体も確認した。手動実行時だけ生成済みartifactから桜音・花音の
6本を選別するstaging専用R2アップロード処理を追加し、run `30282262036`でアップロード、R2上の
サイズ、公開URLからの再取得、SHA-256一致を確認した。staging DBへ
`20260727210000_add_morning_alarm_product_fields.sql`だけを適用し、`fixed_voice_assets`へ
桜音3件・花音3件を登録した。Android On-Device TTS Test run `30280153878`は最新のキャラ選択時
事前取得実装を含めてSUCCESS。productionは変更していない。文言は引き続き仮置きである。
／v5.76: キャラ選択確定時に、会話用TTSの共通bundle・キャラbundleと固定モーニング音声3本を
両OSで準備し、すべて成功した後だけ選択を保存する経路を追加した。導入済みファイルは再利用し、
モーニング設定時は対象キャラの不足分だけ再確認する。取得中は「音声を準備しています…」と表示し、
失敗時は選択を保存せず再試行できる。R2へ配置済みのTTSキャラbundleは現時点で桜音・花音の2人だけ
であることをmanifestから再確認し、未配信キャラを選択可能に表示しない。固定音声のR2／DB登録と
staging反映はまだ行っていない。
／v5.75: 仮文言3種類を6キャラの正式なAIVMXモデルで生成する専用workflowを追加した。
モデルUUID・speaker UUID・SHA-256・ACML 1.0をmanifestへ固定し、公式AivisSpeech Engineで
18本のWAVを生成する。GitHub Actions run 30271054793で、6キャラ×3本、ファイルサイズ、
0.3〜30秒の長さ、WAV読込をすべてPASSし、7日保持のartifactへ保存した。これは生成検証だけで、
R2配置、`fixed_voice_assets`登録、staging／本番反映は行っていない。文言は仮置きのため、
配信前に内容と実音声を確認して差し替え可能とする。
／v5.74: 「メッセージを再生」は毎日同じ1本ではなく、キャラごとに固定音声3本を用意し、
直前に再生した1本を避けて選ぶ仕様へ更新した。当時は音声をアラーム同期時に端末のアプリ専用領域へ
HTTPSから保存し、朝は保存済みファイルだけを再生する。Androidはforeground service、
iOSはAVAudioPlayerでローカル再生し、会話AI、WebSocket、マイク、call作成、コインを使わない。
`fixed_voice_assets.variant_key`と同一キャラ・用途・関係性内の一意制約を追加した。実際の
6キャラ×3本の音声生成・R2配置・DB登録は未実施。
／v5.73: spec v5.4のモーニングコールをローカル実装した。設定画面、複数アラーム、
曜日別週次／一回のみ、キャラ、個別ON/OFF、1〜60分スヌーズ、回数制限、バイブ有無、
チャット／通話後の確認候補、確認後だけ登録するDB RPCを追加した。AndroidはAlarmManagerの
正確なアラーム、再起動／時刻変更後の再登録、foreground alarm音、専用BridgeActivityの
全画面疑似着信を実装した。iOSは検証済みのAlarmKit system sound継続経路を使い、system画面を
「停止」「着信を見る」の2操作、アプリ内を「応答」「メッセージを再生」「拒否」「あとで」
とした。固定メッセージは会話AIへ接続しない境界まで実装したが、キャラ別固定音声素材がまだ
存在しないため再生自体は明示的な未完成状態である。

ローカルではTypeScript、Web build、3 Edge Function bundle、Node全46ファイル、
`git diff --check`をPASSした。ローカル環境にはAndroid Gradle wrapperとXcodeがないため、
Android/iOS native compile、staging反映、実機確認は未実施。本番は未変更。
／v5.72: 次機能を課金からモーニングへ変更し、spec v5.4で製品仕様を確定した。
スマホ標準アラームの代替として、複数登録、曜日別週次／一回のみ、キャラ、個別ON/OFF、
1〜60分スヌーズ、1/3/5回または無制限、バイブ有無をアラームごとに持つ。
Androidはfull-screen intentから全画面疑似着信へ直接入り、iOSはAlarmKitの停止／アプリを開く
から全画面疑似着信へ入る。着信操作は応答、固定の「メッセージを再生」、拒否、あとで、とする。
チャット／通話からの依頼は確認候補だけを作り、疑似LINEでユーザーが確認した後だけ登録する。
この時点では実装、staging反映、実機確認は未実施。
／v5.71: 記憶パイプラインをstagingへ反映し、機械的な実行経路を確認した。
`20260721090000_add_kanon_character.sql`、`20260727180000_add_memory_pipeline.sql`、
`20260727190000_preserve_existing_memory.sql`、`20260727200000_bound_memory_retry_window.sql`
をstagingへ適用し、`memory-worker` v1、`chat-reply` v5、`voice-turn` v16をdeployした。
worker専用secret、worker URL、`MEMORY_LLM_MODEL`もstagingへ設定済みで、secretなしの
worker呼び出しはHTTP 401となることを確認した。

過去7日分の正式通話要約63件を4回のworker実行で処理し、すべてHTTP 200・タイムアウトなし、
処理待ち0件となった。日次処理はユーザー×キャラ×日付14件のうち13件を完了し、会話材料が
ある12件の日次要約と18件の長期記憶候補を生成した。残る1件は7日範囲外だったため
`retention_expired`・再試行なしで終了し、処理待ち0件となった。

検証中、日次抽出が既存の長期記憶を新規抽出分だけで置き換える危険と、8日前の対象を作りながら
実行側は7日以内だけを扱う境界ずれを発見した。前者は既存JSONへ新規JSONを追加し同じキーだけ
更新するDB trigger、後者は対象作成・claimを厳密な7日へそろえ期限切れrunを閉じるforward
migrationで修正した。rollback付きstaging検査で、既存記憶保持、当日再試行、翌日繰り越しを
PASSした。

stagingの会話はYouTube音声の誤認識や名前確認の反復を含む検証データのため、生成された日次要約・
長期記憶候補は品質合格判定に使わない。雑音を恒久保存しないため、今回生成した18件の
`memory_facts`と、値が一致する`user_character_memory`のキーはstagingから削除した。
12件の日次要約は7日で自動削除される短期情報として残した。正常な短い会話による品質確認、
Android/iOSの長時間通話実機確認、日次モデル比較は未実施で、Draftを維持する。本番は未変更。
なお、既存記憶保護triggerの適用前にstaging日次処理13件が完了したため、それ以前の
stagingテスト用`user_character_memory`のうち新規抽出に現れなかったキーは置き換わり、
復元できない可能性がある。productionデータへの影響はない。
／v5.70: v5.69で確定した記憶パイプラインを実装した。migration、`memory-worker`、
通話終了後の正式要約要求、チャット/通話への4層文脈接続、長時間通話内だけの一時要約、
03:40開始と04:00/05:00再試行、7日削除を追加した。ローカルではTypeScript、Web build、
Edge Function bundle、Node全44ファイルをPASSした。DB migration適用、Edge Function deploy、
secret/Vault設定、staging動作確認、実機確認は未実施のため、機能全体は進行中とする。
1日分が80,000文字を超える場合だけ分割し、部分要約を一つへ再統合する。日次モデル比較は
残っている。
／v5.69: 記憶パイプラインの構成をspec v5.3へ確定した。長期記憶だけでは日常会話の流れを
維持できないため、画面履歴、夜間処理後の未整理文脈、直近7日の日次要約、長期記憶の4層とする。

正式な通話要約は夜間ではなく通話終了後に非同期生成し、次のチャット・通話と夜間処理へ渡す。
長い通話では古い部分だけを一時要約するが通話終了時に捨て、正式な通話要約は保存済み全文から
作り直す。夜間処理ではユーザー×キャラ×日付ごとに、最大1,000文字の日次要約と長期記憶更新
候補を作る。日次要約は直近7日分をAIへ渡す。

最後の夜間処理後のチャットは最大20,000文字、未整理通話要約は1本最大1,000文字・合計最大
5,000文字を渡す。正式な通話要約、日次要約、処理失敗中の通話本文は最大7日保持する。夜間処理
は03:30の既存回収後に実行し、一時失敗だけ04:00と05:00に失敗分を再試行する。それでも失敗
した処理単位は元の日付のまま翌日へ繰り越す。

長期記憶の採用ルール、現在日時、通話の終了日帰属、夜間モデルの設定化、通話の記憶5列読込も
維持する。／v5.68: 記憶パイプライン(D)の設計を詰め、spec v5.2へ反映した。実装は未着手である。
／v5.67: Android近接センサーの復旧を完了した。commit `4cada75`。通話中に端末を顔へ
近づけても画面が消えなくなっていた回帰(2026-07-24 commit `294167e` の旧Plugin一括削除で
移し忘れ)を、新しい`voicecall`実装へ戻した。`VoiceAudioRouteController` と同じく、
判断を持つ`VoiceCallScreenController`と、Android APIを扱うPlugin側の
`AndroidCallScreenBackend`に分けた。判断する層は`android.*`をimportしない。

CI run 30236438278 でGradle unit test(新規7件を含む)がBUILD SUCCESSFUL。staging AAB
1066(run 30236600688)の実機で、顔へ近づけると画面が消え、離すと戻り、消えている間も
会話が続き、通話終了で画面が戻ることを確認した。既存動作の非破壊も確認した。

同じ消え方を繰り返さないよう、責務の所在と配線を固定するNode構造テストを追加した。
iOS・本番Supabase・本番Edge Functionは未変更。／v5.66: iOS移行順序10「暫定経路削除・残存参照整理」を完了し、移行順序1〜10がすべて
終わった。commit `2135bd6` でcloud MP3の受信・キュー・再生をiOSから削除し、Androidと
同じ状態にそろえた。`IOSVoiceTtsController` は381行から139行になり、残るのは録音準備音の
再生だけである。実行コードにcloud TTS/MP3再生の参照は0件。TestFlight実機で通話成立、
録音準備音、合図後の録音再開、固定聞き返し、通話終了が従来どおり動くことを確認した。

これでcloud TTSへの接続はアプリからもserverからも無くなり、iOSとAndroidは同じ作りに
なった。本branchはmainへのマージ段階にある。Android近接センサーの復旧は本移行とは
無関係のAndroid単独作業のため、マージ後に別branchで行う。

本番Supabase・本番Edge Functionは未変更。／v5.65: iOS移行順序8「WebSocketオンデバイス専用契約」を完了した。commit `8d298ac` で
server 側の不要な cloud MP3 生成を停止し、staging の `voice-turn` へ deploy して
iOS・Android 両方の実機で会話成立と体感速度の非悪化を確認した。これで固定聞き返し・
通常回答のいずれも `on_device_tts=1` の通話では cloud TTS へ接続しない。

あわせて、Android実機でひとつ回帰を発見したので後続作業へ積んだ。通話中に端末を顔へ
近づけても画面が消えない。2026-07-13 commit `a8e51e3` で旧`AndroidTtfaTestPlugin`へ
実装した近接センサー制御が、2026-07-24 commit `294167e` の旧Plugin一括削除で
新実装へ移されないまま消えていた。オンデバイスTTSの各工程とは無関係で、3日前からの
回帰である。工程10のあとにAndroid単独作業として直す。

残るのは移行順序10のみ。本番Supabase・本番Edge Functionは未変更。／v5.64: 後続作業を2つ積んだ。実装は行っていない。

1件目。録音準備音(ピッ音)のON/OFF設定。spec v5.1のA8-2へ追記した。これまで仕様に
記述がなく実装だけが存在していた音である。会話が機械的に感じられることを避けたい
利用者が消せるようにする。設定画面の既存のデザインテーマ切替と同じ形で置き、
Android・iOSで同じ動きとする。既定値はF1で決める。

2件目。本番ビルドの配信設定検査。stagingには `scripts/verify_tts_bundle_config.py` の
実行ステップがあるが、productionには無い。設定が欠けたまま本番ビルドを作っても
止まらず、工程10でcloud MP3再生経路を削除したあとは完全に無音のアプリができる。
リリース直前(フェーズ4)に、Codemagicの `production` 環境変数グループへ値が
入っていることを確認したうえで追加する。未設定のまま追加すると本番ビルドが止まるため、
確認を先に行う。

あわせて、cloud TTSへの実接続の現状を整理した。固定聞き返しは `on_device_tts=1` の
とき server がAivisを呼ばずテキストだけを送るため解決済み。通常回答は server が
`on_device_tts` を見ずに毎ターンAivisを呼んでおり、Android・iOSとも受け取っても
鳴らさないだけで、cloud TTSへの接続と費用は残っている。これを止めるのが工程8の残り。
Android側にcloud MP3の受信口・再生コードは残っていない(旧Plugin削除済み、
WebSocketListenerにbinary受信のoverrideなし)。工程10のiOS削除は、iOSをAndroidと
同じ状態へそろえる作業である。／v5.63: iOS移行順序9「staging切替・Codemagic・TestFlight実機確認」を完了した。
実機確認で見つかった2件を修正し、両OSで再確認した。

1件目。commit `47859b9`。iOSは合成に失敗すると通話ごと終了していた。Android
`VoiceCallRuntime` は PLAYBACK_FAILED でも通話を終わらせずturnを進める。iOSは
工程8-8bでcloud MP3側の扱い(音が鳴る前の失敗は継続不能)を持ち込んでいたため、
1文の解析失敗で通話が止まっていた。オンデバイスの失敗の受け口から emitFatal を外し、
Androidと同じくturnを進めるだけにした。

2件目。commit `d86d901` / `a80398b` / `8a390eb`。返事に`「」`や`・`が入ると、その文が
まるごと無音になっていた。原因は共通Rust実装の `punctuation_phone` がモデルの扱える
「。、！？」以外の記号をエラーとして返し、文の解析ごと失敗させていたこと。返り値を
Option へ変え、読み上げに使えない記号は読み飛ばすようにした。飛ばすときは音・
アクセント・文字数・BERTへ渡す文字の4つをまとめて飛ばし、`validate_arrays` の
文字と音の対応を保つ。表示される文は変えない。`…` はNFKCでピリオド3つへ開かれるため
読み飛ばし対象ではなく、間として読む。system promptは変更していない。
本件はAndroid・iOS共通のRust実装の修正であり、iOS工程の固定方針「Android実装を
変更しない」に対する明示合意のうえでの例外である。

無音の返事が続くとユーザーが黙り、STT失敗の聞き返しが3回続いて通話が停止していた。
2件目の修正で根本が解消し、聞き返しの連鎖も起きなくなった。

CI: GitHub Actions run 30210099962 で Rust host tests 6件、Android JNI build、
Gradle unit tests がすべてPASS。Android staging AAB run 30211276533 は versionCode
1065 を生成し成功。iOSはCodemagic stagingでAppTestsと署名付きIPA生成がPASS。
Node自動テストは490件PASS。

実機確認(commit `8a390eb`): iOSで発信、疑似着信応答、通常会話、固定聞き返し、
録音準備音、receiver/speaker切替、近接センサー、interruption(タイマー割り込みで
通話終了画面へ遷移)、通話終了、コイン減少、記号を含む返事の読み上げをすべて確認した。
`「桜」＋「音」`が画面には記号付きで表示され、音声では記号を無視して正しく読まれた。
AndroidはAAB 1065で記号の修正と既存動作の非破壊を確認した。

移行順序7と9を`[x]`とする。8はserver側の不要なcloud MP3生成停止が残るため`[~]`を
維持する。10は未着手。specは変更していない。実装コード以外のAndroid、Edge Function、
DB、migration、RPC、R2、env、secret、production、mainは変更していない。／
v5.62: iOS正式疑似通話のオンデバイスTTS接続を工程8-8a / 8bとして実装し、
commit `f6a0db1`でモデル配信設定の受け取りと`on_device_tts=1`送信、
commit `9fd02f9`でモデル取得・導入・読み込み、LLM文単位合成・順序付き再生、
固定聞き返しテキストの合成、cloud MP3の再生抑止、再生完了後の録音復帰判定を接続した。
Codemagic stagingはbranch `agent/ios-stage5-jpreprocess`、commit `9fd02f9`を
7分39秒で完走し、`Run iOS unit tests (AppTests)`、署名付きIPA生成、
埋め込みInfo.plistのiPhone専用・向き4種・`.xctest`非混入検査がすべてPASSした。
生成された`App.ipa`は43.25MB。
TestFlight実機では初回通話の1回目に「通話サーバーへ接続できません。」が出たが、
再試行では長い初回モデル取得後に通常回答のオンデバイス音声が鳴り、
以降の複数ターンは大きな待ちなしで音声が続いた。cloud MP3との二重再生も報告されていない。
これによりiOS実機arm64での実モデル取得・読み込み・合成・再生と、
通常回答後に次ターンへ戻る基本経路は成立確認済みとする。
最初のWebSocketエラーとモデルダウンロードの因果関係は診断値で確認できていないため、
ダウンロードが原因だったとは断定せず、この推測だけを根拠に接続順序は変更しない。

ただし、仕様書A8-3の完成仕様は「初回・追加のキャラ選択時にモデルを取得し、
Android・iOSで同じ動きにする」である。現行は通話開始時取得の暫定状態であり、
追加キャラ選択の本実装も未完了のため、今回のiOS成立確認へ事前取得実装を混在させない。
キャラ追加導線の実装時に、両OS共通のWeb側入口から選択直後の取得・進捗・失敗・再試行を扱い、
通話開始時の初回取得を廃止する。導入済みモデルをアプリ・端末再起動後も再利用する
native側の永続保存とcache判定は現行実装を維持する。
移行順序7は通常回答の実機再生まで確認済みだが固定聞き返し等の総合確認が残るため`[~]`、
移行順序8はclient側のオンデバイス契約とcloud MP3再生抑止まで成立したが、
server側の不要なcloud MP3生成停止が残るため`[~]`、移行順序9・10は未完了とする。
specをv5.0へ更新し、本更新では実装コード、Android、Edge Function、DB、migration、
RPC、R2、env、secret、Codemagic、production、mainを変更しない。／v5.61: iOS正式疑似通話の工程6「iOSモデルbundle管理」を完了した。
Android `TtsModelBundleInstaller` / `TtsModelDownloader` と同じ判定へ揃えた
`IOSTtsModelBundleManifest`(検証のみ)・`IOSTtsModelBundleInstaller`(展開・照合・差し替え)・
`IOSTtsModelDownloader`(取得・cache再利用・打ち切り)を追加した。同じbundleファイルを
両OSが読むため、schema_version・必須ファイル・license必須・サイズ上限(2GiB)・
storage余裕(16MiB)・DL上限(3GiB)・進捗段階名をすべてAndroidと一致させ、
`tests/iosTtsModelBundle.test.mjs`が両者の食い違いを検出する。
一時領域へ全展開し全ファイルのSHA-256とサイズを照合してからatomicに差し替え、
差し替え失敗時はbackupから復元する。他キャラ・共通bundleには触れない。
Zip Slip対策はパス文字列検査と実パスでの封じ込め確認の二重で行い、
symlink entryは中身を検証できないため拒否する(iOS側の追加防御)。
保存先はApplication Supportとし、モデルは配信元から取り直せるため
iCloudバックアップ対象から除外する(iOS固有の判断)。
ZIP読み取りはZIPFoundation 0.9.20をexactVersionで導入した(iOSにZIP読み取りの
標準機能が無いため)。展開先の決定と安全確認は自前で行い`unzipItem`は使わない。
Codemagic staging index 41で1件失敗し、`safeOutputURL`が絶対パスを単体で
弾けていないことを検出して修正した(`appendingPathComponent`は先頭の"/"を
絶対パスとして扱わず連結するため封じ込め判定を通過していた。実害は無いが
単体で成立しない防御は防御と呼べないため修正)。index 42で全68件がPASS。
App.ipaは14.26MB→14.52MB。
**配信先(R2)が未整備のため、実際のモデル取得・インストールは未確認である。**
確認済みなのは壊れたbundleの拒否・cache再利用・打ち切り・復元の各挙動まで。
実配信での確認は工程9で行う。／v5.60: iOS正式疑似通話の工程5「ONNX Runtime・共通日本語frontend基盤」を完了した。
日本語frontendはAndroidと同じRust実装(`android/app/src/main/rust/jpreprocess_frontend`)を
XCFramework化して共有し、Swiftでの重複実装は行わなかった。C ABI越しの薄い橋渡し
`IOSJapaneseFrontend`だけをSwift側に置き、G2P・アクセント推定はSwiftに一切持たせていない。
ONNX Runtimeはmicrosoft/onnxruntime-swift-package-managerをexactVersion 1.24.2で導入した
(Androidの1.22.0とは版が異なる。iOS向けSwift Packageに1.22.0のtagが存在しないため。
同一モデルをopset互換範囲で実行する)。セッション管理`IOSOnnxRuntimeSessions`は
Android `OnDeviceTtsEngine`と同じく共通BERTを常駐させ、キャラ側TTSはキャラ切替時のみ
読み直し、文・turnごとの再生成を行わない。XCFrameworkはビルド生成物として扱い
commitせず、Xcodeのbuild phaseで組み立てる(Androidがgradleで行っているのと同じ位置づけ。
本番workflowはjpreprocessの明示stepを持たないため、workflow側ではなくXcode側で用意する)。
Codemagic staging index 39でXCFramework・modulemapの解決とSwift側10件のテストを、
index 40でONNX Runtimeのリンクと全30件のテストを確認した。App.ipaは5.36MB→5.86MB→14.26MBと
推移し、ONNX Runtimeが実際に組み込まれていることを確認した。日本語frontendは工程7で
実際に呼び出すまで大部分がstripされるため、サイズはその時点で増える。
自動テストはsimulator実行だが、両ライブラリをリンクした状態のipaで、実機の発信・会話・終話・
再通話が従来どおり動作することを確認した(2026-07-26)。ただし両ライブラリのコード自体は
呼び出し箇所がまだ無いため実機では実行していない。実行の確認は工程7以降で行う。
合成そのもの(テンソル組み立てと推論実行)は工程7で実装する。／v5.59: iOS正式疑似通話の工程3「正式iOS Session／Controller骨組み」と工程4「接続・録音・usage・終了処理移行」を、
TestFlight実機確認まで完了した。6 Controllerへの責務分割とその後の実処理移設を行い、Codemagic staging index 30〜33で
Swiftコンパイルを確認したうえで、実機で発信・会話継続・スピーカー/受話口切替・近接センサーによる画面消灯・聞き返し・
録音準備音・終話・コイン消費を確認した。実機で3件の不具合を検出して修正した。
(1) 通話終了後に次の通話が「前の通話を処理しています」で永久に開始できない。工程4でSessionの終了処理が
activeなturn1件しか確定しておらず、複数turnの通話でWeb側の`finalizedTurnCount === expectedTurnCount`検証に落ちて
完了イベントが破棄され、精算outboxが作られなかった。Android `VoiceUsageController.completeTermination`と同じく
終了時に全turnを確定する`finishAllTurns()`へ変更し、送出直前にも再確認する二重の防御を入れた(commit `c62f34f`)。
(2) 初回応答が約20秒かかる。`AVAudioSession`の`.voiceChat`が自動ゲイン調整で無音時の雑音を底上げし、
固定RMSしきい値を超え続けてしゃべり終わりを検出できず、録音上限20秒まで走っていた。
(3) 再生音が最大音量でも極端に小さい。同じ`.voiceChat`の音声処理が再生も減衰させていた。
(2)(3)は共通原因として`.playAndRecord` mode を`.voiceChat`から`.default`へ変更し、あわせて無音継続判定を
観測した発話ピークに対する相対しきい値(peak×0.15と固定値の大きい方)へ変更した。AI音声再生前に必ず録音を停止する
既存設計のため、マイクとスピーカーが同時に動く区間が無くエコー除去への依存は小さい。VAD診断表示も追加した(commit `00c2ecd`)。
実機再確認で、発話後1.6〜2.0秒で応答が始まること、音量が改善したこと、相対しきい値が大音量発話でも正しく働くこと
(peak=0.3213のとき閾値0.0482で1.6秒検出)を確認した。無音のまま上限20秒に達した場合はサーバーが`stt_recovery`を
送出して聞き返しが流れることも実機で確認した。iOSの聞き返しはクラウド音声で成立しており、
工程7でオンデバイスTTSへ移す際はサーバーが既に送っている`stt_recovery_prompt`(テキスト)を使う。
あわせて花音を通常のキャラ名一覧へ追加し、`VITE_TTS_CHARACTER_BUNDLES`が渡らないiOSビルドでも他キャラと同じ扱いで
表示されるようにした。通話中の電話着信による割り込みは未確認。次工程は5「ONNX Runtime・共通日本語frontend基盤」。
spec v4.9、Android、main、production、DB、migration、RPC、R2、env、secret、Edge Functionは本更新では変更しない。）

（v5.58: iOS正式疑似通話の工程2「契約固定・テスト基盤」を完了した。`allowPreparedConnection`はiOS側で一度も読まれておらず、`incomingPreparationActive`だけで昇格を判定していたため、疑似着信用に準備したstandby socketをユーザー発信が昇格し得る状態だった。この契約を修正し、判定は工程3で`IOSVoiceConnectionController`へ移した。method/event payloadを`tests/fixtures/iosVoiceCallContract.json`へfixture化し、Node構造テストと`tests/iosVoiceCall.test.mjs`の版数pinを更新した。`AppTests` XCTest targetを`ios/App/App.xcodeproj`へ配線し、shared scheme `AppTests`とCodemagic stagingの実行stepを追加した。Codemagic `VoiceCompanion iOS Staging` Build ID `6a648f35ac10b841e75cec39`(index 33、対象commit `7a72f598d7f31c290a5459af251998cf4e1c54e9`)はSUCCESSし、`IOSPreparedConnectionDecisionTests`が9 tests / 0 failures / 0 unexpectedでTEST SUCCEEDED、App.ipaの生成も成功した。よって移行順序2を`[x]`とする。工程3は6 Controllerの実装とSwiftコンパイル確認(index 30 / 31 / 32)まで完了しているが、実機確認が未実施のため`[]`を維持する。次工程は4「接続・録音・usage・終了処理移行」で、本更新では着手しない。IPA内に`.xctest`が含まれないことを検査する処理をstagingの既存IPA検証stepへ統合したが、この検査自体は次回以降のstagingビルドで初めて実行される。spec v4.9、Android、main、production、DB、migration、RPC、R2、env、secret、Edge Function、production workflowは本docs更新では変更しない。）

（v5.57: PR #69がmainへマージ済み(merge commit `2454f2482ada367bd7e1de04223dda1f1554ce7a`)となり、Android正式通話Pluginへの必須移行が完了したため、次の主題である「iOS正式疑似通話・オンデバイスTTS接続」の工程1〜10を本書へ追加した。現在の`IOSVoiceCallPlugin.swift`は、接続・録音・クラウドMP3再生・usage・終了・診断など複数の責務を単一Plugin内で持っている。iOSのnative録音、WebSocket、VAD、usage、終了処理は既存コードに実装されているが、現行構成での実機動作確認状態は未確認である。ONNX Runtime、iOSオンデバイスTTS、モデル管理は未実装である。iOS工程は最新Android正式実装の構造・状態遷移・責務境界を設計の参考にするが、Android JavaコードをSwiftへ機械的にコピーしない。現行iOSに実装済みの`AVAudioEngine`、`URLSessionWebSocketTask`、`AVAudioSession`等のiOS固有処理は、新しい正式Swift構成へ移す。工程1「iOS現行責務・依存関係調査」はmain HEAD `2454f2482ada367bd7e1de04223dda1f1554ce7a`で調査済み・調査時の変更なしのため`[x]`とし、工程2〜10は未着手の`[ ]`とする。固定方針として、Android工程8には着手しない、Android実装を変更しない、クラウドTTS不採用、クラウドTTS fallback不採用、`public.users.id`契約維持、nativeからSupabaseへ直接書き込まない、各工程は自動テストと必要な実機確認が終わるまで`[x]`にしない、mainへ直接commitしないことを明記した。本更新はdocs-onlyであり、spec v4.9、Android実装コード、iOS実装コード、Edge Function、DB、migration、RPC、R2、env、secret、Codemagic、production、mainは変更しない。）

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

- [x] コイン数値: 無料の毎日回復量 / 月額付与 / 各商品価格 / チャット1返信の固定消費 / 通話の時間単位 / 通話1単位の消費量 / 端数処理 / 最低消費 / 最大消費 等。**2026-08-05に原価の実測をもとに全項目確定し、spec v5.37・v5.38のF1へ正本化した。**数値そのものはF1を正とする。ここで決めたのは数字だけであり、実装は下のフェーズ3「課金（B）」で行う。
- [ ] 疑似電話の安全上限: 最大STT受付秒数 / 無音打ち切り / 聞き取り失敗回数 等
- [ ] 告白到達の関係値水準 / 未応答の下げ幅 / 拒否打ち切り回数

## フェーズ3. 本格実装（依存順）

- [~] 記憶パイプライン（D）: チャット/通話ログ保存 → 日次処理（夜間バッチ）→ 長期記憶抽出 → AIに渡す組み立て（同日短期文脈は既存から組み立て）
- [~] 課金（B）: RevenueCat app user id は `public.users.id` を使う。RevenueCat購入確認 → Supabaseでコイン付与 → 消費（idempotency_key で二重消費防止）→ 残高不足時の挙動
  - 数値は2026-08-05にF1で全項目確定済み（フェーズ2参照）。
  - **完了（2026-08-06、staging実機確認済み）: コインを期限つきロットで動かす仕組みを入れた。** migration `20260805220000_apply_f1_coin_tariff_and_lots.sql`。料金をF1の確定値（チャット3コイン、通話60秒まで10コイン・以後30秒ごと4コイン、通話開始に10コイン）へ差し替え、無料の1日18コインを利用者のローカル1日ごとに配り直し、使い残しは繰り越さない。消費は失効が近いロットから引く（順序は仕様書に定めが無く、逆にすると無料分が毎日捨てられるためこう決めた）。期限切れは行を消さず0にして失効を台帳へ残す。表示用の残高は生きているロットの残りの合計から作り直す。既存残高は `promo` ロット1本へ引き継いだ（229人ぶん）。
  - staging実機で確認した内容: 345 → アプリ更新で363（無料18枚付与）→ チャットで360 → 通話で350 → チャットで347。DBのロットは無料分が18→2に減り、買った345枚は1枚も減っていない。日をまたいだあと再度開いて363（残り2枚は失効し繰り越されていない）。
  - 旧料金の暫定値4か所は解消済み。`PROVISIONAL_TIER_THRESHOLDS` も廃止し、会話量2段階と共通ベースの上乗せ2段階を、どちらもその日の消費コイン総量から求める形にした（`supabase/functions/memory-worker/relationshipPolicy.ts`）。
  - **残っているもの: 購入・月額のロットを書き込む側。** これが無いため `coin_lots` に入るのは `daily_free` と引き継ぎの `promo` だけである。`common-contact-scheduler/index.ts:265` の有料判定 `hasPaidLot` は読み方としては正しいが、書き込む側が無いので全ユーザーが無料側へ倒れたままである。購入導線（RevenueCat）で解消する。
  - ~~購入導線（RevenueCat）はこの工程に含めない~~ → **2026-08-06に着手した。** ストアとRevenueCatの設定は利用者の作業だが、受け取る側のコードはそれを待たずに作れるため、フェーズ4まで寝かせない。設定手順は`docs/store_setup_20260806.md`にまとめた（商品10個の商品ID・価格・付与枚数、App Store Connect / Google Play Console / RevenueCatの手順、渡してもらう値）。
  - **完了（2026-08-06、stagingのDBで確認済み）: 購入・月額のコインを`coin_lots`へ書き込む側。** migration `20260806060000_add_purchased_coin_grant.sql` の `grant_purchased_coins` と、Edge Function `revenuecat-webhook`。これで手順書に書いていた「書き込む側が無いので全ユーザーが無料側へ倒れたまま」が解消する。
    - 当初は買い切り6か月・月額翌月末として実装したが、spec v5.62とmigration `20260812100000` で廃止した。現在は買い切り・月額とも `expires_at=null` とし、時間経過では失効させない。毎日の無料コインだけは従来どおり利用者のローカル日付で失効する。
    - 二重付与は`coin_lots`の`revenuecat_event_id`と`store_transaction_id`の一意制約、`coin_transactions`の`idempotency_key`で防ぐ。ロットを台帳より先に入れ、どちらかの一意制約に当たれば`already_granted`を返す。台帳だけ残る形を作らないためである。
    - `grant_purchased_coins`はservice roleからだけ呼べる。authenticatedへ`grant execute`しない。呼べると好きな枚数を好きな期限で自分へ配れる。
    - Webhookは`authorization`ヘッダを`REVENUECAT_WEBHOOK_AUTHORIZATION`と突き合わせる。**未設定なら500で受け付けない**（設定漏れで誰でもコインを配れる状態を作らないため）。扱わないイベントと商品IDのずれは200で受け切り（再送しても直らない）、DBの失敗だけ500で再送させる（購入が消えると取り返しがつかない）。
    - 2026-08-06時点のstaging確認では旧期限を確認した。付与・再送冪等・存在しない利用者・台帳・認証未設定時の拒否は現在も維持するが、期限部分はspec v5.62で廃止した。無期限付与、既失効有償分の復元、退会時全消滅、退会後再付与拒否は変更後のmigrationをstagingへ適用して再確認する。
    - **残り: RevenueCatの認証ヘッダをsecretへ設定すること（利用者の作業）、キャラ枠（B6）の実装、コイン画面、アプリからRevenueCatを呼ぶところ。**
  - **キャラ枠の課金を確定した（2026-08-06、spec v5.42 B6）。** 枠に買い切りとサブスク付属の2種類を置き、月額は2,000円で1枠・3,000円で2枠、500円と1,000円の段には付けない。買い切りは1体800円、サブスクを使ったことがある利用者には500円の商品を別に出す。同じ商品に2つの価格を付けられないため、ストアの商品を2つ持って利用履歴で出し分ける。枠から外れたキャラは買い切っていなければ関係値ごと消える。実装はストア商品設定とあわせてフェーズ4。
  - **残高不足まわりを確定し、判定と画面を実装した（2026-08-06、spec v5.44 A8・B10）。実機未確認。** 予告は話せる残り時間が30秒（追加1課金単位＝4コインぶん）を切ったところで1回出す。残り枚数で判定しないのは、残高10〜13コインだと最初の60秒ぶん10コインを払った時点で残り枚数が4を通らず、予告が一度も出ないためである。文言は音声・画面とも「まもなくコインが足りなくなり、通話が終了します。」で共通。秒数は言わない（残り30秒は課金される時間であって時計の30秒ではなく、実時間ではもっと長く話せる）。「なくなる」ではなく「足りなくなる」なのは、話せる時間が課金単位で終わるため残高に端数が残るからである（25コインで始めると150秒＝22コインを消費し3コイン残る）。
    - 入れたもの: 予告の判定を60秒→30秒、予告の帯を新文言（秒数を削除）、会話画面への残高常時表示、通話・送信で足りないときの「コインが足りません」＋購入導線、チャットの送信を止める（保存せず、入力した文章も消さない）、コイン切れの通話をトークへ自動で戻す、トークのタイムラインへ購入導線つきのお知らせを1キャラ1枚。あわせて終了時に残高を0だと決めつける処理を外した（端数が残るため）。`npm test` 732件PASS、`npx tsc --noEmit` エラーなし、`npm run build` の成果物に3文言とも入ることを確認。
    - **入っていないもの: 予告の音声。** 事務的なアプリの声で流すと決めたが、合成音声で作った音声ファイル1本が要る（F6の素材）。いま鳴るものは無く、画面の帯だけである。
    - **入っていないもの: [コインを買う]の飛び先。** コイン画面が未実装のため、押すと「コイン / 課金は後続PRで実装します」を出す。購入導線（RevenueCat）の工程でつなぐ。
    - お知らせカードは端末のメモリだけに持つ。アプリを再起動すると消える。購入画面ができた時点で残し方を見直す。
  - **F5の掃除（2026-08-06、spec v5.45）。** 決まっているのに未決へ置いたままだった3件を正本化して消した。月額コインの繰り越し上限はv5.38のF1で確定済みだった。未精算通話の回収・再試行方針とアプリ強制終了時の精算方法は実装済み・staging確認済みだったのでA8へ書いた（推測で課金しない、端末側outboxの再送、24時間・03:30 JST・100件の日次回収、二重課金の防止）。**F5に残るのは6件**で、無料メインキャラ変更頻度、消費コイン枚数を見せるか（要検討）、課金対象時間の内訳表示、残高が尽きたときの終了タイミング、そして購入導線とセットの3件（引き継ぎ時のキャラ枠、サブスクentitlement、購入復元の文言）である。
### 課金工程の残り（2026-08-10 に順番を決めた）

引き継ぎコード（C6/C7）をこの工程に入れる。spec B8が「買い切りのキャラは引き継ぎコードで
元のIDへ戻れた場合だけ戻る」と定めており、無いと確認項目の「購入の復元」が最後まで通らない
ためである。退会（C9）と入れ直し対策（B8-2）は入れない。課金の確認を止めないため。

**第1段 通話を直す（両OS同時）** → 完了（2026-08-10）

- [x] Android: 聞き取り失敗の回数が0に戻らない（`VoiceCallRuntime` の `recoveryFailureCount`。
      iOSは `done` で0に戻すがAndroidに戻す場所が無い）。iOSと同じ3か所（返事が届いたとき・
      通話の開始・終了）で0に戻す。AAB 1169で実機確認済み
- [x] Android: スピーカーで約1秒で聞き取り失敗（声と判定する大きさが固定値。
      録音打ち切りもAndroid 8秒／iOS 20秒でそろっていない）。拾い口を `VOICE_RECOGNITION` から
      通話用の `VOICE_COMMUNICATION` へ変え、端末が持っていれば回り込み打ち消しと雑音消しも
      付けた。判定の数値は変えていない。AAB 1169で実機確認済み
- [x] iPhone: 無音にすると「もう一度お願いします」で止まる。**原因は音声経路ではなく接続要求の
      作り直しだった。** `IOSVoiceConnectionRequest.with(callId:)` / `with(sttFailureCount:)` が
      `onDeviceTts` を写しておらず、引数の既定値 `false` へ落ちていた。1回目の聞き取り失敗で
      `updateSttFailureCount` が active と standby の両方を作り直すため、以降の接続はすべて
      `on_device_tts=0` になる。印が落ちるとサーバは聞き返しを固定文のテキストではなく
      クラウドの音声データで送り（`voice-turn/index.ts:309`）、iPhoneは届いた音声データを
      捨てる作りなので必ず無音になり、端末からは何の報告も出ない。作り直しで全項目を写して解消。
      Codemagic実機と `function_logs` の両方で確認（同一通話の聞き返し7回すべてが
      `recovery_prompt_text_sent` へ入り、端末から鳴り始め→鳴り終わりが届いた）
- 打ち切り秒数と失敗回数は F2「通話安全上限」が未決。第2段で決める

**この工程での寄り道（同じことをしないこと）**

- iPhoneの症状を、引き継ぎ書に書いてあった「鳴り始めたのに音が流れない」のまま前提にして
  `AVAudioEngine` の組み直しを入れ、ビルドを1回無駄にした。**通話中にengineを止めて繋ぎ直すと、
  第一声のあとが全部無音になる。** 撤去済み。症状は引き継ぎ書ではなくサーバの記録から取り直すこと
- サーバの記録は、端末から来た行が `stage: "android_diagnostic"`、サーバ自身の行が
  `stage: "stt_recovery"` である。**混同すると症状ごと読み違える。** `recovery_audio_started` は
  両方に存在し、`stt_recovery` 側はクラウド分岐でしか出ない

**第2段 決める（作る前に。1件ずつ）**

- [x] 予告を聞き取り失敗の1・2回目でも鳴らすか（今はAndroidだけ鳴る）→ **鳴らす**（決定済みの
      再確認。2026-08-10）。iPhoneをAndroidへ合わせ、聞き返しの区切りで会話へ戻る側でも
      予告を読み上げてから録音準備音・録音再開へ進む。spec A8 v5.54へ反映済み
- [~] F5の4件（消費枚数の表示／課金対象時間の内訳／残高が尽きたときの終了タイミング／
      サブスク権利の自動反映）
  - [x] 残高が尽きたときの終了タイミング → **0になった瞬間に切る**（言葉の途中でも待たない。
        2026-08-10）。実装は既にこの動きで、仕様書だけが未決だった。待つ形を採らないのは、
        待ったぶんの超過を無料にするか未精算にするかという別の決めごとが要るため。
        spec A8 v5.55へ反映済み
  - [x] 消費したコイン枚数を利用者へ見せるか → **出さない**（2026-08-10）
  - [x] 課金対象時間の内訳・秒数を利用者へ表示するか → **出さない**（2026-08-10）。
        上の1件とまとめて決めた。出すのは残高だけ。通話の時間も出さない。実際の時間と
        課金された時間が食い違って見え（実測で5分29秒に対し課金2分1秒）、差の説明が
        タイムラインの1行に収まらないため。spec B10 v5.56へ反映済み。
        **作るものは無い（いまの実装のまま）**
  - [x] サブスクentitlementを現在ユーザーへ自動反映する条件 → **確定**（2026-08-11）。
        spec B8を書き直した。引き継ぎコードと購入の復元を別の機能として切り分け、復元では
        月額契約・枠1つ・次回以降のコインの配り先だけを移す。RevenueCatの Restore Behavior は
        `Transfer to new App User ID` のまま**変えない**。作るものは第3段へ書いた
- [x] spec A8の予告3か所を実装に合わせて直す（文言・同梱ファイル・30秒）→ spec A8 v5.54へ
      反映済み（2026-08-10）。境目は残り20秒、音源は端末の読み上げ音声（同梱ファイルは使わない）、
      文言は「残りコインが少なくなっています。」。この文言は「通話が終了します」を言わないが、
      **このままでよいと利用者が決めた**（2026-08-10）

**第3段 作る**

- [x] 引き継ぎコードの発行と入力（C6/C7）→ 作った（2026-08-11、migration
      `20260811000000_add_transfer_code_rpcs.sql`）。**stagingへ未適用・実機未確認。**
      コードは英数字8文字（`0 O 1 I L` を使わない）、有効期限30分、同時に有効なのは1本、
      入力の失敗5回で15分止める。生のコードは保存せずハッシュだけ置く。端末から旧IDを
      受け取らない（受け取れる形にすると他人のIDを渡して乗っ取れる）
- [x] 「購入を復元する」を引き継ぎ画面へ移す（B8）→ 移した（2026-08-11）
- [x] 設定画面の「コイン / 課金」ボタンを消す → 消した（2026-08-11）
- [x] 購入復元の作り直し（spec B8 v5.57）→ 作った（2026-08-11、migration
      `20260811020000_add_subscription_transfer.sql`）。**stagingへ未適用・実機未確認。**
  - [x] 押したら `restorePurchases()` の前に確認を出す（条件を付けない警告）
  - [x] 復元結果の文言を4通りに分ける
  - [x] Webhookで `TRANSFER` を受ける。枠を旧IDから外して新IDへ付ける。
        **`TRANSFER` は `app_user_id` が空で届くことがある**ので、共通の入口より前に置いた。
        後ろに置くと400で弾かれる。旧IDと新IDは `transferred_from` / `transferred_to` で届く
  - [x] 復元後に「コインも移す／移さない」を選ばせる
  - [x] コイン移行のRPC
  - **設計から1点変えた。** 当初は「旧IDの未精算の通話を先に精算する」としたが、
    `settle_call_coins` は**呼んだ本人の通話しか精算しない**（所有者が違うと
    `call is not available` で落ちる）。呼んでいるのは新IDなので旧IDの通話は精算できない。
    よって**未精算が残っていたら何も移さずエラーで終える**形にした。精算は旧端末が自分で行う

**購入復元の設計（2026-08-11に確定。既存実装を読んで決めた）**

**旧IDをアプリから受け取らない。** 受け取ると、他人のIDを渡してコインを奪える。旧IDと新IDの
対応はRevenueCatの `TRANSFER`（`transferred_from` / `transferred_to`）だけが知っているので、
Webhookが受けた時点でサーバの対応表に記録し、コイン移行RPCはそこから引く。

足すもの。

| 種類 | 名前 | 用途 |
|---|---|---|
| table | `subscription_transfers` | イベントID（一意）・旧ID・新ID・`superseded_at`・枠を移したか・コインを移したか |
| table | `coin_transfers` | コイン移行1回の記録。`transfer_key` 一意。冪等の門と監査を兼ねる |
| 制約 | `coin_transactions_reason_type_check` | `transfer_out` / `transfer_in` を追加 |
| RPC | 枠の移動 | Webhookから呼ぶ |
| RPC | コインの移動 | アプリから呼ぶ。**引数に旧IDを取らない** |

どちらのRPCも `security definer` とし、`anon`・`authenticated` から `revoke` する
（`settle_refund_debt` が既にこの形）。

枠の移動（`TRANSFER` 受信、1トランザクション）。

1. `subscription_transfers` へイベントIDを insert。0行なら処理済みとして返す
2. **`to_user_id` が旧IDの行をすべて `superseded_at = now()` にする**（A→B→C で A→B を無効化）
3. 旧IDの `character_slots` から `source_type = 'subscription'` の枠を外す
   （`20260809200000_refund_deletes_character_and_bans_purchases.sql` と同じ書き方に揃える）
4. 新IDへ `ensure_subscription_character_slots` で枠を付ける
5. 旧IDで枠から外れたキャラを、契約終了と同じ扱いにする

コインの移動（利用者が「移す」を押したときだけ。**1本のRPCに全部入れて1トランザクション**。
Edge Functionから分けて呼ぶと境界が割れる）。

1. `current_app_user_id()` を新IDとし、`subscription_transfers` から `superseded_at` が空で
   `coins_moved = false` の行を引く。無ければ何もせず返す
2. `coin_transfers` へ `transfer_key`（＝イベントID）を insert。0行なら実行済みとして返す
3. 旧・新の `coin_balances` を **uuidの昇順**で `for update`
   （順序を固定しないと逆向きの同時実行でデッドロックする）
4. 旧IDの `settlement_status = 'pending'` の通話を `settle_call_coins(call_id)` で1件ずつ精算。
   **1件でも失敗・精算不能なら例外を投げて全部戻す**
5. `settle_refund_debt(旧ID)` を呼ぶ
6. `coin_lots` を移す
   - `daily_free` 以外 … `remaining_amount > 0` の行の `user_id` を付け替え
   - `daily_free` … 同じ `expires_at` の行が新側に無ければ付け替え。あれば
     `least(daily_free_coin_amount(), 新 + 旧)` へ統合し、旧行は `remaining_amount = 0` にする
     （**行は消さない**）。`daily_free_coin_amount()` は18を返す
7. `refund_debt`: 新 += 旧、旧 = 0
8. `users.refund_count`: 新 += 旧、旧 = 0
9. `coin_transactions` へ2件（旧に `transfer_out`、新に `transfer_in`）
10. `coin_balances` を両側で再計算（`sum(remaining_amount) − refund_debt`）
11. `subscription_transfers.coins_moved = true`

`coin_transactions` と `coin_consumptions` は触らない。過去は旧IDに残す。

**途中失敗**: 4で例外を投げれば1〜11がすべて戻る。「旧から減ったが新に入っていない」は起きない。
**二重実行**: 2の門で止まる。`TRANSFER` の重複受信でも、ボタン連打でも、2回目は何もしない。

**この形の結果（承知のうえで採る）**

- **連鎖しない。** BがAからコインを移さないままC へ移ると、Aのコインはそこに残り、Cから辿って
  集めることはできない。Aの人は基本枠と買い切りのキャラで話せるので、そのコインは使える
- **契約が切れても移せる。** 有効な行が残っていればコインは移せる。払ったコインなので
  取り上げない
- [ ] F5の残り2件（引き継ぎ時のキャラ枠／旧IDへ戻れないときの復元の文言）
- [x] コイン切れのお知らせカードが再起動で消える（端末メモリのみ）→ 直した（2026-08-11、
      migration `20260811010000_record_coin_shortage_in_talk.sql`）。**stagingへ未適用。**
      不在着信と同じくトークの行（`coin_shortage_call_end`）として残す。
      **`calls.end_reason` は列こそ在るが通話終了時には誰も書いていなかった**ので、
      「残高不足で終わった」をDBから判定できない。`calls` は端末から更新できない
      （`20260715190000` で `pinned` だけに絞ってある）ため書かせることもできない。
      そこでRPCは「自分の・終わっている通話であること」だけを確かめ、理由は端末の申告を
      信じる。偽っても自分のトークに1枚増えるだけで、コインも残高も動かない。
      通話1本につき1枚に限るので積み上げられない

**第4段 Androidで確認**

- [x] 引き継ぎコード（AAB 1171で実機確認。コイン・キャラとも引き継がれた。
      **トークの引き継ぎは未確認**。確認時にトークをしていなかったため）
- [x] 購入の復元（AAB 1171で実機確認）。契約が移った記録1件、旧側の月額枠が消えて
      買い切りだけ残ったこと、新側へ枠と契約が移ったこと、コインが4800枚**移動**して
      旧側から消えたこと、毎日の無料コインが移っていないことを記録で確認した
- [x] 枠を買ってキャラを追加できること（`20260811030000` で直した箇所）。
      `acquisition_type='purchased_slot'`・買い切り枠で記録されることを確認した
- [ ] 返金／返金の取り消し／上げた月のコイン差し引き／解約→14日リセット／
      話せないキャラに着信・通知が来ないこと／予告が聞き返しの後に鳴るか
- [ ] コイン切れのお知らせが、アプリを開き直しても残ること
- [ ] **コイン0なのにプッシュ通知が来た**（2026-08-11、利用者の指摘。課金とは別件）。
      話せる残高が無い利用者へ着信・通知を出さない決めごとがある。**未調査**
- [x] 使い切って終わった通話で残高が引かれること（`20260811060000`）。**実機確認済み
      （2026-08-11、AAB 1173）。** 12枚で始めた通話が58,760msで `settled`・10枚引かれ・
      残高2枚になった。直す前後の記録は 60,022ms→failed(0枚)、60,018ms→failed(0枚)、
      直した後 60,031ms→settled(10枚)。精算はサーバー側なのでアプリを入れ替える前から直っていた
- [x] コイン切れのお知らせがトークに出ること（Android）。**実機確認済み**（2026-08-11、
      AAB 1173）。`chat_messages` に `coin_shortage_call_end` が1件残った
- [x] 通話中に他のアプリへ移ったときの扱い。**実機確認済み**（2026-08-11、AAB 1175）。
      通話が終わり、コインが引かれ、トークへ「他のアプリを開いたため通話を終了しました」が
      1枚出て、古いお知らせが消えて1枚だけになった
- [ ] 通話中に他のアプリへ移っても、上限に達したところで通話が切れること
      （Androidの端末側の締め切り）。**確かめられていない。** 他のアプリへ移った時点で
      通話を終了する形にしたため、背面のまま上限に達する状況そのものが起きなくなった。
      締め切り自体は残してある（画面を消したまま通話が続く経路のため）

**実機で見つかって直したもの（第4段）**

- **残高を使い切って終わった通話が、毎回タダになっていた。** 通話は「その残高で話せる上限」
  ちょうどで切っているが、記録される時間は止まるまでのぶんだけ数十ms超える。料金は60秒を
  1msでも超えると次の単位に入る（10枚→14枚）。上限は払える最大として決めているので、
  超えた瞬間の料金は常に払える額より1単位多く、必ず払えない。`settle_call_coins` は
  残高不足として1枚も引かないまま `settlement_status='failed'` にしており、`failed` は
  日次の回収対象（`unrecorded`/`recording`/`pending`）から外れるので後からも引かれない。
  実機の記録は Android 60,018ms・60,022ms が failed、iPhone 57,991ms・54,840ms が settled
  （iPhoneのほうは上限に達しておらず利用者が切った）。**`20260811060000` で「持っている枚数で
  払える上限を超えた請求はしない」に直した**（12枚なら10枚引いて2枚残す）。
  「使い切ったら0にする」とはしない。精算は終話の後に走るため、ちょうどその時に日付が
  変わって毎日の無料コインが入っていることがあり、入ったばかりのぶんまで消える。
  記録は書き換えず、実測の課金時間はそのまま残す。spec A8 v5.59へ反映済み。
  **AAB 1173で実機確認済み**（12枚で始めた通話が58,760msで settled・10枚・残高2枚）
- **Androidには話せる上限の締め切りが端末側に無かった。** Web側は両OSへ
  `maxBillableDurationMs` を渡しているが、読んでいたのは iOS だけだった
  （`IOSVoiceCallPlugin.scheduleBalanceDeadline`）。Androidは画面側のタイマーだけが頼りで、
  通話中に他のアプリへ移るとタイマーが間引かれ、上限を大きく超えて話せてしまう。
  超えたぶんは上の決めごとにより請求されないので、そのままタダになる。
  `VoiceUsageController` に締め切りを持たせ、達したら端末側で通話を止めて画面側へ
  `balanceExhausted` を知らせる形にした（iOSと同じ扱い）。**実機未確認**
- **Androidだけ「コインが足りなくなったので通話を終了しました」がトークに出なかった**
  （2026-08-11、利用者の指摘）。`record_coin_shortage_notice` は「終わったと**記録済みの**
  通話」にしかお知らせを出さない（`ended_at is null` なら何もせずnullを返す）。終わったことを
  記録するのは outbox の `mark_call_usage_pending` で、端末から通話完了の知らせが届いてから
  送られる。終話した瞬間にはまだ送れていないため、そのまま呼ぶと毎回nullで捨てられていた。
  iPhoneは端末側が先に完了を出すぶん間に合っていた。**待ち時間を決め打ちにせず、
  記録されるまで数回やり直す形にした**（0秒・2秒・10秒・30秒。`requestCallSummary` と同じ形）。
  お知らせが残ったらトークを読み直して画面へ出す。**AAB 1173で実機確認済み**
- 引き継ぎが**誰がやっても必ず失敗していた**。`redeem_transfer_code` が
  `public.users.auth_user_id` を書き換えていたが、この列には一意制約があり、入れ直した端末では
  その認証IDを仮ユーザーの行が既に持っている。`20260811040000` で書き換えをやめ、
  代わりに仮ユーザーを削除候補にした（spec C7 の手順8）
- **引き継ぐ人が、捨てるだけの名前とキャラを先に作らされていた。** 設定へ入るには
  名前とキャラが要り、引き継ぎ画面は設定の中にあったためである。最初に「はじめる」と
  「引き継ぐ」を選ぶ画面を作った（利用者の指摘）
- 引き継ぎ入力の画面で、下のボタンが上と重なって見えた。`.intro` の 20vh の余白のまま
  中身を増やしたためである。この画面だけ余白を減らし、ボタンに間隔を付けた（利用者の指摘）
- **`TRANSFER` が4回とも400で弾かれていた。** `app_user_id` が**空で届く**イベントなのに、
  分岐を「`app_user_id` が無ければ400」の判定より後ろに置いていた。テストは書いていたが、
  比べる目印（`UUID_PATTERN` の判定）を間違えており、実際に弾いていたのはその手前だった。
  分岐を前へ移し、テストは両方の目印と比べるようにした。**置き場所を固定するテストは、
  実際に弾いている場所と比べること**
- 復元でストア側が失敗を返したのに、契約は実際に移っていた。RevenueCatの応答が遅れると
  起きる。返事だけで「確認できませんでした」と出すと、成功しているのに失敗に見える。
  **判定をサーバーの状態で行う形に変えた。** 押した直後には`TRANSFER`が届いていない
  ことがあるので、少し待って数回見に行く。「確認できませんでした」は、ストアにも聞けず
  サーバーにも何も無いときだけ出す
- 引き継ぎのあと、端末登録の書き込みが拒否された（実機で
  `new row violates row-level security policy for table "device_installations"`）。
  アプリが「自分の他の端末を無効にする」更新をかけるが、対象は旧端末の行で、行の
  `auth_user_id` が今のセッションと違うため端末からは触れない。**spec C5 の
  「引き継ぎ完了時に旧端末のPush token・着信・通知・モーニングを無効化する」を
  実装していなかった。** `20260811050000` で引き継ぎの中に入れた。守りは緩めていない

**第5段 iPhoneで確認**（Codemagicは利用者が押すため、コードを固めてから1回で通す）

- [ ] 第4段と同じ範囲を通しで確認。iOSは課金導線を一度も通していない

**第6段 仕上げ**

- [ ] 本書の課金（B）の記録を更新する（2026-08-06で止まっている。枠・乗り換え・返金・
      RevenueCatイベント台帳・コイン画面が未記載）
- [ ] mainへマージする（現在 main より183コミット先。課金の実装はmainに1つも無い）

**利用者の作業（並行）**

- [ ] ストア側4件（RevenueCat認証ヘッダ／GoogleのRTDN／App Store Connectの価格順／商品10個）
- [ ] 入れ直し対策のストア設定は待ち時間があるため依頼だけ先に出す

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
- **2026-08-06に同じ枠の症状を2件追加した（iOS実機、staging `call_logs` で確認）。どちらもプロンプトの質の問題であり、この工程では直さず後工程へ回すと決めた。**
  - **渡した値を「分からない」と答える。** 指示文へ「ユーザーの氏名: すずき やすたか」を渡し、埋もれないよう記憶・要約のあと（現在日時の直前）へ置いたが、「俺のフルネーム知ってる?」に対し4回とも「ここではすずきさんのフルネームは分からないよ」と答えた。値が届いていないのではなく、AIが「ここでは」と自分に制限をかける言い方へ寄る。渡す位置の問題ではないので、指示文の書き方から見直す必要がある。
  - **主語がすり替わる。** 利用者が「今テストしててさ」と言った直後、AIが「うん、テスト中だよ。今は課金まわりの動作確認も進めてるよ」と自分がテストしている話に変えた。さらに「それは俺のことじゃなくて?」に対し「すずきさんの話じゃなくてアプリ側のテストだよ」と押し返した。上の料理→PFCの件と同じ、文脈の持ち主を取り違える症状である。
  - なお、この工程で直した2件（発音がカナに戻ること、名前の連呼が止まること）は同じログで確認できている。7ターン中の呼びかけが3回から1回（通話の最初だけ）へ減った。

### オンデバイスTTS本実装（採用決定・2026-07-18）

- 前提: `poc/on-device-tts`のPoC実験は終了し、オンデバイスTTS採用を決定済み。仕様は`voice_companion_spec.md` A8-3 / H1 / H4を正とする。オンデバイスTTSはv1の6キャラ全員が対象で、桜音は方式・量子化・配信の検証基準モデルとして用いる(桜音1モデルに絞る前提にはしない)。
- [x] 疑似電話・通知のAI音声再生を、クラウドTTS(Aivis)からオンデバイスTTS(ONNX Runtime + Style-Bert-VITS2系AIVMX)へ置き換える現行実装。TTS本体・DeBERTa BERTともに動的INT8、文末(。!?)単位のストリーミング合成・逐次再生（読点分割は不採用）。コード実装・CI greenで、Android実機でも通常AI音声と固定聞き返しのオンデバイス再生を確認済み。正式な最終構成への置き換えは下記8段階で行う。
- [x] 配信を「共通BERT＋キャラ別モデル」方式へ分割するコード実装。声モデルは初期アプリに同梱せず、共通BERT bundle(共有DeBERTa BERT＋共通ライセンス)を端末へ1回だけDL・保存・再利用し、キャラ別bundle(AIVMX＋voice-config＋モデルライセンス)をキャラごとにDL・保存する。キャラ切替(桜音→花音→桜音)でも共通・既取得キャラモデルを再DLせず(キャッシュ)、片方のDL/検証/更新失敗が既存の正常インストールを壊さない(失敗隔離＝原子的入替＋失敗時ロールバック)。設定は`VITE_TTS_COMMON_BUNDLE_URL` / `VITE_TTS_COMMON_VERSION` / `VITE_TTS_CHARACTER_BUNDLES`。※実装・自動テスト・CI green。
- [x] Cloudflare R2(staging)への共通/キャラ別bundle配置と公開URL取得確認。手動実行専用のアップロードworkflowでcommon／桜音／花音の3bundleをアップロード成功(workflow run `29751863325`)、アップロード後のサイズ＋SHA-256 read-back検証が成功。さらに`public_base_url`付きで再実行し、公開URL(`https://pub-7bcdc02f44a14b68a333bc9b9ee07f73.r2.dev`配下)からの取得確認(サイズ＋SHA-256一致)も成功(workflow run `29755001244`)。
- [x] staging AAB build workflowへ`VITE_TTS_*`(`VITE_TTS_COMMON_BUNDLE_URL` / `VITE_TTS_COMMON_VERSION` / `VITE_TTS_CHARACTER_BUNDLES`)をGitHub staging環境変数から注入する配線と確定値の事前検証を追加。common version=`1`・bundle id `sakune`/`kanon`はコード/bundle定義から確認済み。
- [x] staging環境変数への実値設定と花音のstaging DB登録。桜音・花音のstaging `characters.id`を取得(桜音=`3ff18df6-e2e2-49ee-afe8-01257ed53fb5`、花音=`d7410a62-3e02-4d17-b272-14ba0262ae3a`)。花音はstaging DBへ手動追加済みで、再現用migration `supabase/migrations/20260721090000_add_kanon_character.sql`を追加済み(適用はしていない)。GitHub staging環境変数3件(`VITE_TTS_COMMON_BUNDLE_URL` / `VITE_TTS_COMMON_VERSION` / `VITE_TTS_CHARACTER_BUNDLES`)を設定済み。
- [~] `Android Staging AAB` build、端末からの実ダウンロード、Android実機確認。R2配置・公開URL確認・workflow配線・staging環境変数設定・花音DB登録は完了。AAB 1060では通常AI音声、録音準備音の可聴、固定聞き返しのオンデバイスTTS再生、通話画面のボタン位置解決を実機確認済み。初回音声までの長い待ちと文間遅延は未解決であり、正式Android通話実装への移行も未完了のため`[~]`を維持する。
- [x] 量子化(TTS/BERTの動的INT8生成)とその検証はローカルで行わず、CI内で実行(ローカルでの重い処理は禁止)。CIで桜音＋花音の実AIVMXとDeBERTaから共通1＋キャラ2 bundleを生成・検証(run `29744823614` green)。実モデルはGit/AABに入れずCI内のみ。
- [x] モデルbundle取得の耐障害性と、バックグラウンド継続、進捗表示（2026-08-02）。読み取り60秒・最大4回の再試行・`Range`での続きからの再開・4xxは再試行しない・416は完成扱いで検証へ回す。`BufferedOutputStream`を外し、ディスクの長さと受信量を一致させて再開位置のずれ（繋ぎ目のずれた壊れたzip）を潰した。一時ファイル名をURLから決まる固定名にし失敗時も残すことで、プロセスが死んだ後の起動でも続きから再開する。Androidは`TtsModelPreparationService`（foregroundServiceType=dataSync）＋PARTIAL_WAKE_LOCKで、他アプリへ移っても画面を消しても止まらず、通知バーへ進捗を出す。iOSは`URLSessionConfiguration.background`へ移し、サスペンド・終了後もOSのデーモンが転送を続け、完了時は`AppDelegate`の`handleEventsForBackgroundURLSession`で繋ぎ直す。同一URLは合流して二重DLを避け、失敗時は`resumeData`を保持する。旧`URLSessionBundleFetcher`は削除。進捗は両OSとも同じイベント名・同じ形で送り、共通/キャラの区別を出さず2本を1つの割合へまとめる（本ごとだと1本目完了時に0%へ戻りやり直しに見えるため）。失敗時は固定文言ではなく例外の本文を返す。MockWebServerで実際に切断→Range再開→バイト列一致まで検証。`npm test` 632件・Android JUnit・iOS AppTestsすべてPASS。iOS実機（TestFlight）とAndroid実機（staging AAB）の両方で、%表示・DL完了・他アプリ移動と画面ロックをまたぐ継続を確認済み
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

次工程はPR #69のマージ後、別branch・別PRで「iOS正式疑似通話・オンデバイスTTS接続」。PR #69は2026-07-24にmainへマージ済み(merge commit `2454f2482ada367bd7e1de04223dda1f1554ce7a`)であり、以降は下記のiOS移行順序で進める。

現在の位置: 移行順序1〜10すべて完了(2026-07-26)。工程1〜4は実機確認済み、工程5・6はCodemagic staging index 39・40・42の自動テストで確認済み。commit `8a390eb`のstaging buildとTestFlight実機通話により、iOS実機arm64での実モデル取得・読み込み・通常回答のオンデバイス合成・再生・固定聞き返し・録音準備音・割り込み・通話終了・usageまで確認した。同じcommitのAndroid staging AAB 1065で、共通Rust実装の修正が両OSに効いていることも確認した。iOS正式疑似通話・オンデバイスTTS接続は完了であり、mainへのマージ段階にある。キャラ選択時の両OS共通事前取得はA8-3の完成仕様だが、追加キャラ選択の本実装と合わせる後続作業とし、工程8の成立確認へ混在させない。

**正式iOS疑似通話・オンデバイスTTS接続の前提(2026-07-25・v5.57):**

- 現在の`IOSVoiceCallPlugin.swift`は、接続・録音・クラウドMP3再生・usage・終了・診断など複数の責務を単一Plugin内で持っている。
- iOSのnative録音、WebSocket、VAD、usage、終了処理は既存コードに実装されているが、現行構成での実機動作確認状態は未確認である。
- ONNX Runtime、iOSオンデバイスTTS、モデル管理は未実装である。
- iOS工程は、最新Android正式実装の構造・状態遷移・責務境界を設計の参考にする。ただしAndroid JavaコードをSwiftへ機械的にコピーしない。
- 現行iOSに実装済みの`AVAudioEngine`、`URLSessionWebSocketTask`、`AVAudioSession`等のiOS固有処理は、新しい正式Swift構成へ移す。

**iOS工程を通じた固定方針(工程1〜10で変更しない):**

- Android工程8「初回音声・文間遅延の計測と改善」には着手しない。
- Android実装を変更しない。
- クラウドTTSは不採用。
- クラウドTTS fallbackは不採用。
- `public.users.id`契約を維持する。
- nativeからSupabaseへ直接書き込まない。
- 各工程は自動テストと必要な実機確認が終わるまで`[x]`にしない。
- mainへ直接commitしない。

**正式iOS疑似通話・オンデバイスTTS接続への移行順序:**

1. [x] iOS現行責務・依存関係調査: main HEAD `2454f2482ada367bd7e1de04223dda1f1554ce7a`で調査済みで、調査時のコード変更はない。現在の`IOSVoiceCallPlugin.swift`は、接続・録音・クラウドMP3再生・usage・終了・診断など複数の責務を単一Plugin内で持っている。native録音、WebSocket、VAD、usage、終了処理は既存コードに実装されているが、現行構成での実機動作確認状態は未確認である。ONNX Runtime、iOSオンデバイスTTS、モデル管理は未実装であり、工程5〜7で新規に用意する。
2. [x] 契約固定・テスト基盤: Web↔native間のmethod/event契約を固定した。iOSの`IOSVoiceCallPlugin`は`allowPreparedConnection`を一度も読まず`incomingPreparationActive`だけで昇格を判定していたため、ユーザー発信が疑似着信用standby socketを昇格し得る状態だった。`call.getBool("allowPreparedConnection") ?? false`を取得し、偽なら他条件を評価せず昇格しない判定へ修正した。昇格は「flag真＋着信準備有効＋standby socket存在＋url/token/character一致」の全成立時のみとし、判定は工程3で`IOSVoiceConnectionController`へ移した。method/event payloadと昇格判定8ケースを`tests/fixtures/iosVoiceCallContract.json`へfixture化し、Node構造テストと`tests/iosVoiceCall.test.mjs`の版数pinを更新した。`AppTests` XCTest target(`com.apple.product-type.bundle.unit-test`、host application=`App`、`TEST_HOST`/`BUNDLE_LOADER`設定、署名なし)を`project.pbxproj`へ配線し、shared scheme `AppTests`とCodemagic stagingの実行stepを追加した。Codemagic `VoiceCompanion iOS Staging` Build ID `6a648f35ac10b841e75cec39`、index 33、対象commit `7a72f598d7f31c290a5459af251998cf4e1c54e9`はSUCCESS。`IOSPreparedConnectionDecisionTests`は9 tests / 0 failures / 0 unexpectedでTEST SUCCEEDEDし、既存scheme `App`によるApp.ipaの生成も成功した。cloud経路は切り替えておらず、既存の実通話経路を維持している。IPA内に`.xctest`が含まれないことの検査はstagingの既存IPA検証stepへ統合したが、次回以降のstagingビルドで初めて実行されるため現時点では未確認である。
3. [x] 正式iOS Session／Controller骨組み: `IOSVoiceCallPlugin`をCapacitor bridge中心へ縮小し、`IOSVoiceCallSession`、`IOSVoiceConnectionController`、`IOSVoiceRecordingController`、`IOSVoiceUsageController`、`IOSVoiceAudioRouteController`、`IOSVoiceTtsController`を追加する。必要以上に一度に分割せず、工程ごとに責務を移行する。Android正式実装は構造・状態遷移・責務境界の設計参考とし、機械移植はしない。6 Controllerは1移行単位=1責務で実装済みで、`IOSVoiceCallPlugin.swift`は1379行から1109行へ縮小した。Swiftコンパイルは Codemagic staging index 30(工程3-4まで)、index 31(工程3-5まで)、index 32(工程3-6、6責務完了)でいずれもSUCCESSを確認済み。**ただし実機確認が未実施のため未完了とする。** 本工程はsocket・録音・usage・audio route・再生という通話経路の実行時責務をすべて移設しており、コンパイル成功は動作を保証しない。Android正式実装の同種作業(移行順序6)でもAAB 1061のWebSocket URL処理、AAB 1062のコール音・着信音・バイブ・audio routeで、コンパイルが通る不具合が実機で見つかっている。実機確認まで完了した。Codemagic staging index 30(工程3-4まで)、31(工程3-5まで)、32(工程3-6、6責務完了)で
Swiftコンパイルを確認し、TestFlight実機で発信・会話継続・スピーカー/受話口切替・近接センサー・聞き返し・
録音準備音・終話・コイン消費を確認した。`IOSVoiceCallPlugin.swift`は1379行から1109行へ縮小した。
移行順序9のTestFlight総合確認は別途行う。
4. [x] 接続・録音・usage・終了処理移行: `URLSessionWebSocketTask`によるactive接続、standby準備・昇格、reconnect、closeを接続Controllerへ移す。`AVAudioEngine`による24kHz・mono・PCM16録音、VAD、`speech_end`の一回送信を録音Controllerへ移す。usage集計、fatal通知、call/turn/世代によるstale callback拒否をSession境界で固定する。既存のWeb method/event契約は維持し、iOS専用の業務ルールは作らない。
socketの生成・受信ループ・close・自動再接続の予約を接続Controllerへ移し、Pluginは`URLSessionWebSocketTask`に
一切触れない。`speech_end`の一回送信は録音Controllerが行う。終了順序(録音→再生→usage確定→接続)と
usageの計測開始はSessionが仲介する(Android `ControllerBindings`相当)。stale callback拒否はSession境界の
`isStaleCallback`と各Controllerの世代判定を併用する。実機確認で3件の不具合を検出・修正済み(v5.59の変更履歴参照)。
通話中の電話着信による割り込みは未確認。
5. [x] ONNX Runtime・共通日本語frontend基盤: ONNX Runtime iOSを導入する。日本語frontendはRust実装のiOS static libraryまたはXCFramework化を第一候補として検証し、Swiftでfrontend全体を重複実装する案は共有が不能と判明した場合に限る。Android fixtureとの出力一致を自動テストで確認する。model sessionを文・turnごとに再生成しない。
第一候補のXCFramework化が成立したため、Swiftでの重複実装は行っていない。`jpreprocess_frontend`crateへ
iOS向けstaticlibとC ABIを足し(JNIは`#[cfg(target_os = "android")]`で閉じたまま)、
`scripts/build_jpreprocess_ios.sh`が実機・simulatorの2slice構成でXCFrameworkを作る。
ヘッダとmodulemapは`ios/jpreprocess/include`の1か所だけに置き、XCFramework側へは同梱しない
(同一moduleの二重定義を避けるため)。Android fixtureとの一致は
`IOSJapaneseFrontendTests.testParsesSameJsonAsAndroid`が`JpreprocessG2pTest`と同一の期待値で確認し、
`tests/iosJapaneseFrontend.test.mjs`が両者の片側だけの変更を検出する。
model sessionの非再生成は`IOSOnnxRuntimeSessions`の読み込み回数カウンタで検証する。
Codemagic staging index 39・40でテスト30件のPASSを確認し、実機でリンク後のアプリ動作(発信・会話・終話・再通話)も確認した。両ライブラリ自体の実機実行は工程7以降。
6. [x] iOSモデルbundle管理: common BERT bundleとcharacter bundleを`URLSession` downloadで取得し、manifest検証、SHA-256検証、size検証、Zip Slip対策を行う。保存先はApplication Supportとし、一時領域で完全検証してから原子的に切替え、失敗時はrollbackする。取得済みbundleはcache再利用し、common bundleの再DLを防止する。
Android正本と同じ判定へ揃えた3ファイル(`IOSTtsModelBundleManifest` / `IOSTtsModelBundleInstaller` /
`IOSTtsModelDownloader`)で実装した。cache再利用はcommonがversion変更時のみ、characterは未取得か
`requires_common_version`不一致時のみ再取得する。打ち切りは`CancelHandle`で即座に止め、
通信エラーではなく`canceled`として扱う(通話終了を失敗として診断に出さないため)。
XCTest 38件(取り込み22・取得16)とNode 12件で検証し、Codemagic staging index 42で全68件PASS。
実配信からの取得はv5.62のTestFlight実機通話で確認済み。固定聞き返し等を含む総合確認は工程9に残る。
7. [x] iOSオンデバイスTTS: 通常回答を文単位で合成し順序どおり再生する。固定聞き返しと録音準備音を同経路で扱い、generationによるstale処理拒否を行う。usageは通常回答の実再生時間だけ加算し、固定聞き返しと録音準備音は非課金とする。cloud TTS fallbackは実装しない。commit `9fd02f9`でモデル取得・導入・読み込み、通常回答と固定聞き返しの合成配線、順序付き再生、実再生時間usage、stale拒否、再生完了後の録音復帰を実装した。合成失敗時に通話ごと終了していた不具合はcommit `47859b9`でAndroid `VoiceCallRuntime` と同じ「turnを進めるだけ」へそろえた。工程9のTestFlight実機で通常回答、固定聞き返し、録音準備音、割り込み、複数ターン継続を確認し、cloud MP3との二重再生がないことも確認した。録音準備音は従来のローカルcueを維持しており、AI通常回答のcloud MP3 fallbackには戻さない。
8. [x] WebSocketオンデバイス専用契約: `on_device_tts=1`、`stt_recovery_prompt`、LLM text eventを用いる契約へ揃え、iOS側がcloud MP3を再生しないようにし、server側の不要なcloud MP3生成を停止する。iOSローカルTTSの接続成立前にcloud MP3を止めない。commit `f6a0db1` / `9fd02f9`でclient側の契約、LLM text・固定聞き返しtextの受信、cloud MP3の再生抑止まで実装し、実機でオンデバイス音声を確認した。二重再生の明示確認は工程9に残す。ただしserverは現在もcloud MP3を生成・送信しており、iOSが受信後に捨てている。commit `8d298ac`で server 側の不要な cloud MP3 生成を停止した。`runLlmAndTts` へ `onDeviceTts` を渡し、真のときは synthesize を呼ばず null を返し、sendAudio は null を 送信しない。文の切り出し・1文ずつの llm テキスト送信・送信順・`done` の流れ・失敗時の扱いは `on_device_tts=0` のときと同じに保つ。staging の `voice-turn` へ deploy し、iOS・Android 両方の 実機で会話が成立すること、体感速度に変化がないことを確認した。本番Edge Functionは未変更。
9. [x] staging切替・Codemagic・TestFlight実機確認: Swift unit test、TTS env検査、archive、IPA検査を通し、TestFlightで配布する。実機では発信、疑似着信応答、録音、AI通常音声、固定聞き返し、録音準備音、receiver/speaker切替、近接センサー、interruption、usage、終了を確認する。commit `8a390eb`のCodemagic stagingでAppTestsと署名付きIPA生成がPASSし、TestFlight実機で上記すべてを確認した。割り込みはタイマーで再現し、通話が終了して中断画面へ遷移することを確認した。usageはコイン残高の減少で確認した。確認中に見つかった合成失敗時の通話終了(`47859b9`)と、記号を含む文が無音になる共通Rust実装の不具合(`8a390eb`)を修正し、両OSで再確認した。Android側はstaging AAB 1065で記号の修正と既存動作の非破壊を確認した。
10. [x] 暫定経路削除・残存参照整理: cloud MP3再生処理、旧一括Plugin責務、fallback、不要listener、不要テストを削除し、実行コード内のcloud TTS fallback参照が0件であることを確認する。commit `2135bd6`で`IOSVoiceTtsController`から再生キュー・`IOSVoiceQueuedAudio`・`AVAudioPlayer`によるMP3再生・準備token・初回再生観測・実再生時間算出・失敗後の継続を削除し、381行から139行にした。残したのは録音準備音の再生と完了通知だけ。Pluginは音声データの受信をno-opにし(古いserverが送ってきても捨てる)、cloud側のdelegate実装6つと`enqueueAudio`を削除した。`first_audio_started`はオンデバイスの再生開始から出すよう移し、`AVAudioPlayer`内部の4段階は記録元が消えたため`CALL_TIMING_STAGES`から外した。画面文言「Aivis TTS生成中…」を「音声を作っています…」へ変えた。cloud経路を固定していた自動テスト17件を、意図を保ったままオンデバイス側へ向け直した。実行コードに`IOSVoiceQueuedAudio`・`enqueueAudio`・MP3のplayer生成は0件。TestFlight実機で通話成立、録音準備音、合図後の録音再開、固定聞き返し、通話終了が従来どおり動くことを確認した。

**Android近接センサーの復旧（移行時の移し忘れ・Android単独）:**

- [x] 通話中に端末を顔へ近づけたとき画面を消す機能を、新しい`voicecall`実装へ戻す。commit `4cada75`で`VoiceCallScreenController`(判断のみ・`android.*`をimportしない)と、Plugin側の`AndroidCallScreenBackend`(SensorManager / PowerManager / Window)に分けて実装した。`VoiceAudioRouteController`と同じ形。音声経路と同じ入口(`prepareAudioRoute` / `releaseAudioRoute`)で開始・終了する。画面を消したまま終わらせないよう、戻してから監視を止める。近接センサーやwake lockが使えない端末では自動スリープの抑止だけを行う。`WAKE_LOCK`権限はmanifestに残っていたため追加不要。Gradle unit test 7件を`android-on-device-tts-test.yml`へ登録し、責務の所在と配線を固定するNode構造テスト6件を追加した。staging AAB 1066の実機で、顔へ近づけると画面が消え、離すと戻り、消えている間も会話が続き、通話終了で画面が戻ることを確認した。
- 2026-07-13 commit `a8e51e3` で旧`AndroidTtfaTestPlugin`へ実装したが、2026-07-24 commit `294167e` の旧Plugin一括削除(2606行)で一緒に消えた。新実装に近接センサーのコードは1行も無い。iOS側は`IOSVoiceAudioRouteController`に残っている。
- 影響は通話中に画面が点いたままになること。頬での誤操作と電池消費が増える。会話自体は成立する。
- 2026-07-26のAndroid実機で発覚。オンデバイスTTSの各工程とは無関係で、3日前から壊れていた。

**録音準備音のON/OFF設定（A8-2・両OS共通）:**

- [~] 設定画面に「AIの発話後に録音準備音を鳴らす」を追加し、既定ONとしてDBへ保存する。staging migrationとローカルWeb buildは完了、アプリ反映と実機確認待ち。
- [~] 設定を全通話開始時にAndroid・iOSのnativeへ渡し、両OSで同じ動きにした。native compileと両OS実機確認待ち。
- [x] 鳴らさない設定でも、Androidは既存140ms、iOSは音長相当100msの無音待機後に同じ完了callbackから録音を再開する。停止・reset時は待機を取り消す。

**本番ビルドの配信設定検査（リリース直前・フェーズ4）:**

- [ ] production workflowへ、stagingと同じ `scripts/verify_tts_bundle_config.py` の実行ステップを追加する。現在stagingにはあるがproductionには無い。
- [ ] 追加前にCodemagicの `production` 環境変数グループへ `VITE_TTS_COMMON_BUNDLE_URL` / `VITE_TTS_COMMON_VERSION` / `VITE_TTS_CHARACTER_BUNDLES` が入っていることを確認する。未設定のまま追加すると本番ビルドが止まる。
- [ ] 工程10でcloud MP3再生経路を削除したあとは、配信設定が欠けたビルドは完全に無音になる。検査なしで本番ビルドを作らない。

**キャラ選択時のモデル事前取得（A8-3完成仕様・追加キャラ導線と同時に実装）:**

- [~] 初回キャラ選択で選択を確定した時点に、共通bundle・選択キャラbundle・固定モーニング音声3本を取得し、すべて成功した後だけ選択を保存する両OS経路を実装した。native compile・実機確認待ち。
- [~] stagingの追加キャラUIも選択確定前に同じ取得経路へ接続した。完成版の追加キャラ購入・選択導線は未実装であり、暫定UIを完成版とは扱わない。
- [~] Web側に両OS共通のモデル／固定音声準備入口、取得中表示、失敗時の未保存・再試行を追加した。詳細なダウンロード進捗率の表示は未実装。
- [~] native downloaderは導入済みの共通bundle・キャラbundle・固定音声を再利用する。両OSのnative compile・再選択実機確認待ち。
- [ ] 両OSの通話開始時に行っている未取得モデルの初回ダウンロードを廃止する。通話中は導入済みモデルの読み込み・warm-upだけを行う。

**記憶パイプライン（D・spec E章）の実装方針:**

- [x] 日次処理は既存の毎晩の自動実行（`pg_cron`）へ相乗りさせる。03:40に対象を作り、03:45にworkerを起動するmigrationをstagingへ適用し、cron 7件とworker HTTP 200を確認した。
- [x] 正式な通話要約は通話終了後に非同期生成し、日次要約へ取り込まれるまで次のチャット・通話へ渡す。長い通話の古い部分は通話中だけの一時要約へまとめ、正式要約は保存済み全文から作り直す。
- [x] 日次処理はユーザー×キャラ×日付単位で、その日のチャット、正式な通話要約、イベント、通知結果から日次要約と長期記憶候補を同時生成する。会話が無い組み合わせは呼ばず、80,000文字超過時だけ分割して最後に日次要約を一つへ統合する。
- [x] 日次要約は1日最大1,000文字、直近7日分を会話AIへ渡し、7日で削除する。最後の成功した夜間処理後のチャットは最大20,000文字、未整理通話要約は1本最大1,000文字・合計最大5,000文字とする。
- [x] 一時的な夜間処理失敗は04:00と05:00に失敗分だけ再試行し、その後は元の日付で翌日へ繰り越す。正式な通話要約と未処理通話本文は最大7日保持する。rollback付きstaging検査で当日再試行・翌日繰り越し・7日超過の終了をPASSした。2026-07-28に03:40／03:45、04:00／04:05、05:00／05:05、05:30の全cronが実時刻に成功し、7月27日分4 runの初回完了と待機0件を確認した。
- [x] 正常な食べ物の会話が日次要約と長期記憶へ保存され、翌日の実機会話で保持されていることを確認した。記憶機能はPASS。過去会話を箇条書きのように並べる不自然さは、機能成立後のプロンプト品質調整へ分離する。
- [x] 長時間通話は、実装済み`voice-turn`のstaging deploy後に確定15ターンの通話がcompletedまで継続し、10往復の一時要約開始条件を実機経路で通過した。古い履歴だけの要約、直近4往復維持、`history_summary`の受信・次ターン再送・終話時破棄はAndroid/iOS共通契約として自動検証済み。
- [~] 日次処理モデルは`MEMORY_LLM_MODEL`で変更可能にした。複数モデルの比較と採用モデル確定は未実施。
- [x] `voice-turn` と `chat-reply` は `user_character_memory` の5列（`profile_json` / `relationship_json` / `preference_json` / `memory_json` / `safety_json`）を読む。

- [~] 通知（A10）: デイリー通知文＋疑似LINE入口の同時生成、共通`context_id`、日付冪等、lover毎日確定／lover不在時の関係値重み1枠、通知全体・キャラ個別ON、quiet hours、有効端末・token同期、端末別耐久queue、APNs/FCM送信、受理後だけの送信ログ、通知タップ時の一度だけの入口メッセージ挿入まで実装済み。staging DBと両Function、FCM/APNs secret、Firebase Android設定、Apple Push capability、Push対応App Store profile、iOS entitlementまで反映した。Codemagic Build index 57のnative buildとTestFlight実機で、iOS token登録、APNs実通知、タップから花音チャット表示、入口メッセージ1件の冪等挿入までPASS。AndroidはversionCode明示・検査・番号付き成果物へ修正したAAB 1075を実機へ導入し、FCM token登録、ロック中の実通知、タップから対象チャット表示、入口メッセージ1件までPASS。両OSのバックグラウンド実通知経路は成立した。アプリ前面では両OS共通の`pushNotificationReceived`処理、同一キャラのトーク表示中だけの通知UI抑制、入口メッセージの冪等反映、内容非表示設定、自発着信の独立設定まで実装・自動検証済み。2 migrationのstaging適用と`notification-dispatch-worker`のstaging deployも完了した。残りはnative build・実機確認、F3正式重み、自動cron有効化。自動cron有効化と自発着信の本番スケジューラは、下の「通知・着信の共通スケジューラ（A10-2）」へ集約する。
- [~] 日常の気まぐれ着信（A10-1）: 通知段階30秒と通知オープン後30秒の二段階期限、通知前のAI第一声テキスト生成、両OSでの着信中PCM事前合成と1回再試行、準備完了までの応答待機、通常通話画面での合成済み第一声再生、拒否／無応答／バックグラウンド移行／準備失敗の冪等な不在着信履歴、期限後タップの対象トーク遷移、有効端末1台への配信、Android端末側30秒停止をStagingへ接続した。CallKit、PushKit、気まぐれ着信用AlarmKit、`ringtoneSoundNamed`は使わない。iOSは既存のTestFlight実機結果を合格とする。Android AAB 1104では、アプリ前面のPCM準備後同時着信、別アプリ前面のheads-up通知、バックグラウンドでのロック全画面、履歴からスワイプ終了後のロック全画面、応答後の画面維持、第一声、録音準備音、通常会話、終話後の安全なロック画面復帰を実機確認し、Android端末側着信工程を完了した。残りは、着信時刻・頻度・対象キャラ・quiet hours・残高による通常メッセージ振分・dry-runを統合する本番共通スケジューラ（下の「通知・着信の共通スケジューラ（A10-2）」へ集約）、録音準備音ON/OFF設定、Play Consoleのfull-screen intent申告、許可OFF・メーカー差を含むリリース前確認である。
- [~] モーニング（A9）: spec v5.15で複数アラーム、曜日／一回のみ、キャラ、スヌーズ、バイブ、両OSの疑似着信操作、会話からの確認登録、応答時のAI生成第一声と着信中の文章先行生成、固定メッセージの通常通話画面／通常履歴と1.0秒待機を確定。Androidは全画面許可必須化、通知の応答／停止、`応答`／`メッセージを再生`の実機確認まで完了。iOSは検証済みAlarmKit標準音経路へAndroid共通Webフローを接続し、warm起動context、対象alarmだけの停止／スヌーズ、AI通話前の停止完了待ち、固定メッセージの近接監視・受話口／スピーカー切替まで自動検証済み。commit `0711b78`のCodemagic iOS Staging Build `6a686234b6f0c284b5aa280e`でnative compile、AppTests、署名済みIPA生成がPASSし、TestFlight実機でも一連の動作に問題がないことを確認した。桜音・花音の固定音声各3本のstaging R2配置とDB登録、AI第一声のstaging `voice-turn`反映も完了。キャラ選択時のTTS＋固定音声同時準備は実装済みだが両OSのnative compile・実機確認が未のため、状態は上の「キャラ選択時のモデル事前取得（A8-3完成仕様・追加キャラ導線と同時に実装）」節の`[~]`を正とする。R2配置済みTTSは桜音・花音だけで、未配信キャラは選択不可。残りは通話画面のデザイン・ボタン配置と、現在の仮固定WAVを試聴したうえで現在より長く朝の呼びかけとして内容のある正式文言へ差し替える素材工程。
- [~] 通知・着信の共通スケジューラ（A10-2）: デイリー枠と気まぐれ枠の共通割当、lover全員＋非lover 1枠、気まぐれ対象の重み選択、通知／着信振分、quiet hoursと既定夜間帯、ユーザー発信後の3時間後ろ倒し／当日中止、同一／別キャラの近接回避、1日最大1着信、無料のみの着信割合低下、休眠減衰、連絡量3段階を実装した。対象は有効かつ通知許可済みPush tokenを持つ利用者だけとし、30分plannerは現地日付ごとに一度だけ分単位の不規則な時刻を保存する。1分cronはdueのDB存在確認だけを行い、due時だけEdgeを起動して最新会話から通知文／着信第一声を生成する。着信30秒期限は第一声生成後の実dispatch開始時から数える。旧prototype候補は送信対象外。stagingの有効2端末・2ユーザーで、通知／着信の送信直前生成と両OS受信、連絡量設定、録音準備音ON/OFFまで実機PASS。F3 v2では連絡量3段階を基準率とし、親密度1点ごとの連続補正で1キャラでも関係が深いほど気まぐれが増え、複数キャラでは深い相手ほど選ばれやすくした。正式値はspec v5.35および`SCHEDULER_CONFIG`へ固定し、staging Function version 10へdeploy済み。A11で共通ベースを1日+5と決めたことを受け、`intimacy_score`を0〜100へ丸める前に`SCHEDULER_CONFIG.intimacyScorePerNormalizedPoint = 6`で割る修正を入れた。旧実装は会話20日で補正が上限へ張り付き、そこから告白までの100日間まったく変化しなかった。告白の床600が補正値100に対応する。生値を直接読んでいた`callProbability`も同じ変換を経由させ、3つの式自体は変更していない。staging Function version 11へdeploy済み。残りはproduction反映判断である
- [ ] イベント・告白（A11）: 発生条件（サーバルール）、通常通知→疑似LINE入口メッセージ→電話していい？→OK→アプリ内疑似着信→応答→疑似電話の導線、告白・関係状態変更・呼び方変更の明示同意フロー、pending状態、未応答時の戻し/保留、lover化と文脈変化、iOS/Android共通の告白導線。**設計・数値・イベント運用は2026-08-02に確定した**（3スコア構成、カテゴリ別加算、疎遠減衰と暴言の減点、呼び方の節目200／450／950は総合点、親友は共通ベース＋友人スコア560、告白は総合1200かつ共通600かつ恋人300、初期値0／50／100、発生は条件成立から1〜7日後のどこか1日、確認メッセージ期限3日、2回目は恋人+60かつ会話30日、打ち切りの付け替えは差額の50%・設定切り替えは差額の25%、告白が始まった地点から1往復無料、着信OFFはトーク告白、電話を断られたら2日後に再挑戦しそれでも駄目ならトーク告白、キャラ性はセリフと態度だけで数値は共通）。**spec v5.36として正本化済み**（A5・A3-8・A11・A10-5・A10-7・A10-2）。決定の経緯と根拠は`docs/handoff_a11_relationship_design_20260801.md`に残す。**点数が動くところまで実装しstagingへ反映した。**migration `20260802090000`で3スコアと関連列を追加、`memory-worker/relationshipPolicy.ts`へ数値と純関数を集約、migration `20260802100000`で`apply_daily_relationship_delta`と`apply_relationship_milestones`を追加、`memory-worker`の日次要約へ`conversation_category`を相乗りさせた。2 migrationをstaging適用、Function version 21へdeploy済み。毎晩その日の会話を分類して3スコアを進め、呼び方と`close_friend`を更新するところまで動く。初回3択の2段構成UIと初期値`0 / 50 / 100`はcommit `d44079f`で実装済み。**告白イベントを条件成立の判定から結果反映まで通した。**migration `20260803090000`で`event_states`への告白イベント（進行中は1キャラ1件の一意制約）、予約・遷移・返答・期限切れ・恋愛イベント許可切替のRPC、`contact_schedule_slots`の`slot_kind='event'`、5分ごとの`invoke-confession-tick` cronを追加した。`memory-worker/confessionPolicy.ts`へ予約日・再挑戦日・期限・選択肢の判断、`confessionWorkflow.ts`へDBの読み書きと文面生成、`common-contact-scheduler`へ予約日が来たイベントの枠取りと配信、`voice-turn`へ告白の通話文脈と保留中の切り出し、`src/confessionEvents.ts`と`appMain.ts`へ誘い／返事のカードと設定の恋愛イベント許可を実装した。予約日・再挑戦日・期限は決めた時点で具体的な日付・時刻へ確定させるので、配信側もRPCも日数を持たない。A10-7の告白1往復無料はmigration `20260803100000`で`call_logs`へ免除の範囲を持たせて実装し、保留中の切り出しは`voice-turn`と`chat-reply`の両方へ入れた。**A11のコードはすべて書き終え、stagingへも反映した。課金ボーナスは未接続**で、購入コインを消費したかの判定にロット単位の消費記録が要るが仕組みが無いため段を0で固定している。会話量の境界はF1未決のため暫定値を`PROVISIONAL_TIER_THRESHOLDS`へ隔離した。両方ともF1のコイン量確定後に入る。**2026-08-04に実機確認を完了した（Android・iOS両方）。**通知→誘い→OK→着信→応答→通話→3択→結果反映、3択の4パターン、着信OFFのトーク告白、2回目の告白（2択・打ち切り・差額50%の付け替え）、切り出しだけで切った場合の誘いへの戻し（回数を消費しない）まで確認した。この工程で4件を修正した。(1)電話に2回誘って駄目だったときのトーク告白が、5分ごとの定期処理から直接トークへ書かれており通知も時間帯ルールも伴わなかった問題を、配信をA10-2のスケジューラへ渡す形へ直した（`aad7dc7`）。(2)3つのActivityでプラグイン登録が食い違っており、モーニングの全画面から設定を開くと着信の許可確認が落ちていた問題を、全Activityで同じ登録に揃えて直した（`96ef904`）。(3)その修正で、着信の呼び出し音停止がモーニングの全画面のロック画面設定まで降ろし、応答時に画面が隠れる回帰を起こしたため、自分が立てた画面にだけ効かせる形へ直した（`33975b3`）。(4)会話の時間帯指示が第一声の生成にしか無く、2ターン目以降が朝の前提で話していた問題を、通話とチャットの全ターンの指示へ入れた（`991aebd`）。**残るのは課金ボーナスと会話量の境界（F1待ち）、および利用者からの告白（F3-2、別工程）である**
- [ ] 猫（A3-6）: ランダム猫（ルール）／AI猫（分類）、懐き度

## フェーズ3-後. 直す課題（実機で見つかったもの）

- [ ] **プッシュ通知で届いた内容が、トーク画面に出ていない**（2026-08-09、利用者が実機で確認）。
  通知をタップして開いた場合は入るかもしれないが、**タップせずにアプリを開いた場合にも出す**のが
  要件である。`20260807235000_record_notification_in_talk_on_send.sql` が「送った時点でトークへ
  残す」形にしてあり、この工程で入れたはずのものが効いていない。タップ時の書き込みを外して
  送信時へ寄せた変更なので、送信側が書けていないと**どちらでも残らない**状態になる。
  通知を送る経路（`notification-dispatch-worker` / `daily-notification-worker`）が
  `chat_messages` へ書けているかをstagingのDBで確かめるところから始める

## フェーズ3-後. 入れ直しによる周回を止める（spec B8-2）

課金の工程とは分ける。**課金の工程では入れないが、出す前までに必ず入れる。**

- [ ] Android: Play Integrity API の device recall（ベータ）で端末に旗を立てる。入れ直しても端末を初期化しても残り、繰り返しの悪用を止めるための公式の仕組みである。保持できるのは3つの値で、旗3本として使える
- [x] iOS: DeviceCheck で同じことをする（開発者ごとに端末あたり2ビット、入れ直しても残る）。native token、Apple API、購入前clearance、bit書込・読戻し、再インストール保持、管理reset、最終`enforce` TestFlight、iOSだけDB強制ON、StoreKit購入画面到達・キャンセル非課金まで実機＋DB PASS
- [x] 旗を立てる条件と読む場所を決める。bit0は購入履歴、bit1は返金による購入停止。購入画面を開く前とアプリ起動後に読み、新しい匿名利用者でbit0だけ立っていれば引き継ぎ・購入復元を案内する。再インストール後の新規匿名ユーザーで`recovery_required`を実機＋DB確認
- [x] **旗を消す操作をこちらに持つ。** staging限定で、service roleが対象ユーザーへ10分・1回限りの許可を発行し、端末自身のDeviceCheck tokenでbitを消す。productionは拒否。許可消費・bit0=false読戻し・再設定までTestFlight実機＋DB確認
- [ ] 必要な設定（Google Cloud側の有効化、Appleの鍵の発行）は利用者の作業である

**入れない場合に何が起きるか:** spec B8-2の購入停止（6か月2回で3か月）は、入れ直しで回数も禁止期間も丸ごと消える。返金1回あたり800円〜3,000円が動くため、止まらないと課金の穴が残ったままになる。無料枠の入れ直しも同じ仕組みで塞がるが、そちらは1回あたり約1.2円（無料18コイン×原価0.065円）で、単独では作る理由に足りない。

**「Androidに規約上きれいな手段が無い」は誤りだった。** 以前の判断は App Set ID がリセットされることを根拠にしていたが、Googleの資料で device recall を確認した（2026-08-09）。手段が無いのではなく、両ストアの設定とサーバー側の作りが要るというだけである。

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
- Z-6（声・素材）→ フェーズ0。声モデルの商用ライセンス確認とv1仮採用は完了。TTS方式はオンデバイスTTS(Style-Bert-VITS2系AIVMX動的INT8)採用を決定済み(2026-07-18)で、v1の6キャラ全員が対象。桜音を検証基準モデルとした実測でTTS品質(INT8劣化なし)・速度基準(1文目 約0.9〜1.0秒)も確認済み。モーニング固定音声は6キャラ×仮文言3本の生成・機械検査まで完了し、試聴・文言確定・配信を残す。ほかにキャラ画像・猫の鳴き声、低速端末対策の方針決定(F6)が残る。詳細はspec A8-3 / H1。
- Z-7（画面設計）→ フェーズ1/3で各画面を作りながら確定。下記「Z-7 画面設計の宿題」も参照

### Z-7 画面設計の宿題（2026-08-03 実機確認での指摘）

現在の画面は検証用の仮であり（`src/styles.css` の冒頭に明記）、デザイン確定時に作り直す。
そのときにまとめて反映する項目を、指摘が出た時点で記録しておく。着手はこの工程の後でよい。

- **下部の4ボタン（トーク・キャラ・コイン・設定）を常時表示しない。** コイン以外は滅多に
  使わないため、画面上部などに1つのアイコンを置き、そこからモーダルで展開する形にする。
  下部を常時4分割で占有するのは、使用頻度に合っていない
- **設定画面から「恋愛イベント」の項目を外す。** キャラページで対象のキャラを選び、その中で
  切り替える形にする。設定一覧にキャラごとの恋愛設定が並ぶのは、体験として興ざめになる
- Z-8（RLS・API）→ フェーズ1。RLS・制約・インデックスのDB土台と指定migration 5件の本番反映は完了済み。v4.3匿名Auth方針でのRLS実動作と `current_app_user_id()` による `public.users.id` 解決は staging で検証済み・PASS（2026-07-11、漏れなし）。今後追加するmigration、Edge Function、secret、RLS/APIは開発中stagingで検証し、本番反映・本番RLS確認・本番`current_app_user_id()`確認はフェーズ4でまとめて行う。API設計、Edge Function整理、アプリ側読み書き実装は各機能工程で進める。匿名サインインと引き継ぎコードを前提にする。`transfer_codes` はテーブル追加のみで、発行/引き換え用 Edge Function / RPC とUIは未実装。
- Z-9（実機検証）→ フェーズ0・フェーズ4。Androidモーニング導線のフェーズ0検証は合格、残りは本番AI接続・フォールバック・リリース前総合確認。下記「Z-10 サーバ機能の検証の型」も参照

### Z-10 サーバ機能の検証の型（2026-08-04 A11の実機検証で確立）

イベント配信や通知のようにサーバが起点の機能は、条件成立を待たずに検証する。
その際に何度もつまずいた点を残す。

- **必ず1人・1キャラに絞る。** 全利用者へ配ってしまった事故がある。AndroidとiPhoneは別アカウント
- **DELETEとINSERTを同じ文に入れない。** 一意制約のあるテーブルでDELETEの効果がINSERTから
  見えず `duplicate key` になる。予約と枠置きも別の文にする
- **配信は必ずスケジューラの枠を通す。** 配信用のFunctionを直接叩くと通知が作られず、
  「通知が来ない」という誤った結論になる（実際にそう報告された）
- 枠は現地8:00〜21:59にしか置けず、置いてから30分以内に配信されないと自動で中止される。
  同じキャラの他の枠とは6時間、利用者から連絡した直後は3時間空ける。
  確実に出したいときは、その利用者の今日の枠を全部消してから置き直す
- 検証用にずらした会話の時刻は、通話の見出しと `calls.created_at` の差で元へ戻せる
- **報告は成果物で確認する。** ソースを直したことと端末で効くことは別である。
  サーバ側だけの修正ならアプリの再ビルドは要らない。逆に、アプリを触ったらビルドが要る
