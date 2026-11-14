# 旅行管理・共同精算サービス 統合設計案

2026年11月13日 · 設計レビュー用

## 1. 推奨方針

**travel-itinerary の React・Vite・Tailwind とスマートフォン向け旅程UIを土台にし、warikan の支出入力・精算表示を React に組み込む。認証、権限、台帳、アルバムは新しいデータ層で実装する。** 旅行IDを共通の軸にして、予定の詳細から支出・写真を追加できるようにする。Firebase Auth、Firestore、信頼できるサーバーを使い、開発は Emulator Suite、検証は分離した Firebase プロジェクトで行う。旧本番のデータは管理者の割当を確認してから移す。

既存のUIと旅程データ形式には再利用の価値がある。旅行の作成・閲覧・更新をそのまま認証付きにするだけでは、旧データの所有者を決められず、権限変更と送金の整合性も担保できない。既存UIを残して入出力の境界を差し替える方が、全面刷新より移行範囲と回帰リスクを抑えやすい。別案は新しい Vite/TypeScript アプリに純粋関数とUI部品を移植する方式。認証ガードとデータ層を切り離した時点で、既存の画面が旧スキーマやブラウザ状態に強く依存し、修正量が移植量を超える場合に切り替える。両アプリを別画面のまま連携する方式は、旅行・参加者・精算の正本が分かれ、同期と移行が増える。

TypeScript は精算、API、Firestore 変換層から導入し、既存 JSX は触れる範囲で移行する。書き込み境界では実行時にも入力を検証する。GitHub は開発者のソース管理とCIに使い、利用者の操作は Firebase UID と旅行内の権限で完結させる。

```text
スマートフォンの React PWA
  ├─ Google ログイン → Firebase Authentication → UID
  ├─ 旅行・予定・支出の読み取り → Firestore Rules → 旅行ごとの文書
  ├─ 招待・権限・台帳の更新 → Firebase callable → 認可とトランザクション
  └─ 写真・動画・ZIP → 認証済みメディア配信 → 非公開オブジェクトストレージ
```

## 2. コード調査

GitHub コネクタで両リポジトリの公開 `main` の先端を取得し、同一コミットのローカル Git オブジェクトで全追跡ファイルの一覧と主要コードを照合した。Firebase コンソール、実データ、実際の公開URLでの挙動、デプロイ済みルール、利用量は未確認。warikan は主要関数を Node.js で抽出実行した。ブラウザE2Eと本番通信を伴う再現は未実施。 [travel-itinerary e87361b](https://github.com/mejir/travel-itinerary/tree/e87361b0d95a8e442498380fb6bd6421ad3603f0) · [warikan 0a2a692](https://github.com/mejir/warikan/tree/0a2a692fd86d0c75a53bd4a48b9ff96d82fd7d55)

| 項目 | 確認した実装 | 再利用・変更方針 |
|---|---|---|
| 旅行・予定 | React 19、Vite 8、Tailwind 4、React Router。Firestore の旅行と予定を購読。旅行の作成・編集、日別タイムライン、予定の作成・編集・削除、場所・カテゴリ・メモ、コメント・リアクション、URLのコピー・LINE共有。旅行一覧は端末の直近10件で、削除は一覧から取り除く操作。 | 日付タブ、タイムライン、予定カード・フォーム、コメント・リアクションの表示、共有シートを再利用。ルーティング、一覧、CRUD、コメントの保存を認証付きのデータ層へ。 |
| 位置・天気 | 端末の現在地、座標がある予定への直線距離、公開 Nominatim の逆ジオコーディング、OpenWeather の現在天気。追跡コード内で地図表示と予定座標の入力UIは確認できなかった。 | 距離計算・天気UIを再利用。場所と移動区間をモデル化し、地図を追加。天気APIをサーバー経由にしてキャッシュする。 |
| 旅行の写真 | 予定ごとに1つの固定 Storage パスに保存。大きい画像は Canvas で縮小・JPEG 化し、Firestore にダウンロードURLを保持。 | 詳細画面の表示導線を再利用。原本の複数アップロード、動画、アルバム、ダウンロードは新設。 |
| 支出・精算 | warikan は単一の `index.html` にCSS・DOM・計算を同居。支出の追加・編集・削除、全員または選択者の均等割り、個別金額指定、立替・負担・差額表示、送金元別の提案、コピー・LINE共有。保存は `warikan_v3` と旧 `warikan_v2` の localStorage。 | 負担者選択、残額プレビュー、送金元別の表示と共有文の構成を再利用。計算を型付きの純粋関数に切り出し、精算台帳と送金記録を新設する。 |
| PWA・配信 | warikan に manifest、固定アセットをキャッシュする Service Worker、`paypay.html` がある。travel-itinerary の追跡ツリーに manifest・Service Worker がなく、Vite の Pages 用 `base` と Router の `basename` が一致していない。両方の追跡ツリーにテスト・CI ワークフローが見当たらない。 | 統合側に manifest とアプリの静的ファイル用キャッシュを実装する。Firebase Hosting の SPA リライトと同一サイトの Google ログインを第一候補とする。Pages を使う場合はサブパスと直リンク、Safari の認証リダイレクトを検証する。 |

根拠は [travel-itinerary の依存関係](https://github.com/mejir/travel-itinerary/blob/e87361b0d95a8e442498380fb6bd6421ad3603f0/package.json)、[旅行一覧・削除](https://github.com/mejir/travel-itinerary/blob/e87361b0d95a8e442498380fb6bd6421ad3603f0/src/pages/HomePage.jsx#L35-L59)、[予定の購読・更新](https://github.com/mejir/travel-itinerary/blob/e87361b0d95a8e442498380fb6bd6421ad3603f0/src/hooks/usePlans.js#L17-L95)、[写真アップロード](https://github.com/mejir/travel-itinerary/blob/e87361b0d95a8e442498380fb6bd6421ad3603f0/src/hooks/usePhotoUpload.js#L57-L81)、[warikan の保存と旧形式の変換](https://github.com/mejir/warikan/blob/0a2a692fd86d0c75a53bd4a48b9ff96d82fd7d55/index.html#L863-L910)、[PWA キャッシュ](https://github.com/mejir/warikan/blob/0a2a692fd86d0c75a53bd4a48b9ff96d82fd7d55/service-worker.js)、[Firebase のリダイレクト認証ガイド](https://firebase.google.com/docs/auth/web/redirect-best-practices)。

### 優先して対処する所見

- **最優先 · 旧旅行データのアクセス。** リポジトリの Firestore ルールは `trips` と `plans` を UTC の2026年12月31日0時まで未認証で読み書き可能にする。Storage ルールは予定写真の固定パスへの読み書きを無期限に許す。同じルールが稼働中なら、第三者による閲覧・改変・削除と、期限到来での Firestore アクセス停止が起こり得る。公開前に実際のルールとバケット設定を読み取り専用で照合する。名前・リアクション・コメントの旧IDは端末の localStorage やクライアント入力に由来するため、所有権の証明には使えない。 [Firestore Rules](https://github.com/mejir/travel-itinerary/blob/e87361b0d95a8e442498380fb6bd6421ad3603f0/firestore.rules) · [Storage Rules](https://github.com/mejir/travel-itinerary/blob/e87361b0d95a8e442498380fb6bd6421ad3603f0/storage.rules) · [旧ユーザー識別](https://github.com/mejir/travel-itinerary/blob/e87361b0d95a8e442498380fb6bd6421ad3603f0/src/utils/user.js)
- **高 · 精算の残高と表示。** 現行の均等割りは端数を支払者、対象外なら最後の負担者に寄せ、通常は合計を保つ。メンバー削除では、その人が支払った明細を消し、他の明細から負担分だけを外すため、負担合計が元の金額を下回り得る。例えば A が払った100円を A/B 各50円で記録し B を削除すると、残る負担は50円となり、送金提案は空になる。貪欲法は最少回数を保証しない。受取残高が8・7・6円、支払残高が9・8・4円なら現行コードは5件を返すが、8→8、9→7、9の残り2→6、4→6の4件で済む。 [負担額計算・削除](https://github.com/mejir/warikan/blob/0a2a692fd86d0c75a53bd4a48b9ff96d82fd7d55/index.html#L954-L1024) · [残高・送金計算](https://github.com/mejir/warikan/blob/0a2a692fd86d0c75a53bd4a48b9ff96d82fd7d55/index.html#L1314-L1397)
- **高 · 精算UIへの文字列埋め込み。** 名前を HTML エスケープして `innerHTML` 内の `onclick` の JavaScript 文字列に埋めている。属性の文字参照を復号したハンドラをローカルで評価し、細工した名前で任意の式が実行されることを確認した。現行は端末ローカル入力だが、共同編集やインポートでは別の利用者から届く値になる。React のテキストノードと関数ハンドラへ移す。README に記載の Discord Webhook は対象コミットの実装に見当たらない。PayPay はアプリを開く導線で、送金記録や照合はない。 [名前の描画](https://github.com/mejir/warikan/blob/0a2a692fd86d0c75a53bd4a48b9ff96d82fd7d55/index.html#L969-L979) · [PayPay 導線](https://github.com/mejir/warikan/blob/0a2a692fd86d0c75a53bd4a48b9ff96d82fd7d55/paypay.html)
- **中 · データ競合・運用。** 予定のコメント・リアクションは同じ予定文書の配列に増え、リアクションの切替は購読中の値を起点にする。写真の書き込みと Firestore 更新も別処理。予定の削除から写真の掃除は呼ばれない。README の天気キー名が実コードと異なり、`VITE_*` の天気キーはクライアントに含まれる。位置変化ごとの公開 Nominatim 呼び出しは、アプリ全体のレート制限と帰属表示の運用が必要なため、初期版は登録した地点からの地図表示に切り替える。 [予定の更新処理](https://github.com/mejir/travel-itinerary/blob/e87361b0d95a8e442498380fb6bd6421ad3603f0/src/hooks/usePlans.js) · [天気取得](https://github.com/mejir/travel-itinerary/blob/e87361b0d95a8e442498380fb6bd6421ad3603f0/src/hooks/useWeather.js) · [Nominatim 利用ポリシー](https://operations.osmfoundation.org/policies/nominatim/)

## 3. 画面と操作

| 画面 | スマートフォンでの操作 |
|---|---|
| ログイン・旅行一覧 | Google ログイン後に「進行中・予定」「過去の旅行」「招待」。招待カードから承認・拒否。作成フォームは旅行名、期間、行き先。 |
| 旅程 | 旅行内の既定タブ。日付、タイムライン、現在天気・予報。予定と移動を登録し、詳細から地図、コメント、写真追加、支出追加へ。移動は出発地・到着地・手段・出発／到着時刻を扱う。 |
| アルバム | 日付・予定・スポットで絞り込み。撮影時刻順、進捗・再試行、原本ダウンロード、写真の選択とZIP。動画は再生操作まで本体の取得を開始しない。 |
| 支出・精算 | 支出フォームは金額・支払者・負担方式・日付・分類・内容・任意の予定。日別／カテゴリ別／個人別の集計、自分の立替・負担・送金確認待ち・残額、全員の送金案。共有前に文面を確認できる。既定は認証が必要な精算画面URLで、金額入り要約は利用者が選んだ場合に共有する。 |
| 設定・メンバー | 旅行情報、招待、ロール管理、アーカイブ、復元・削除。閲覧者には閲覧とダウンロードを提供し、編集アクションは表示しない。 |

旅行内の下部ナビは「旅程・アルバム・支出・設定」。旅程画面を開くときに支出・アルバム全件を購読せず、表示中のタブと期間に読み取りを絞る。PWA はインストールとオフライン時の起動を提供し、未送信の支出や送金を確定表示しない。初期版のキャッシュは公開静的アセットのみとし、個人データはメモリキャッシュ、ログアウト・アカウント切替・権限エラーで表示を破棄する。

地図は最初に API キー不要の Google Maps URL で場所・経路を開けるようにし、画面内は必要な箇所に遅延ロードする Maps Embed を候補とする。Embed は利用料無料だが、制限付きキーと請求アカウントが必要。天気は旅行先を既定とし、現在天気・5日予報をサーバーで地点別にキャッシュし、取得時刻を表示する。位置権限を求めるのは現在地からの距離表示を選んだときに限る。 [Maps URLs](https://developers.google.com/maps/documentation/urls/get-started) · [Maps Embed](https://developers.google.com/maps/documentation/embed/quickstart) · [OpenWeather のプラン](https://openweathermap.org/price)

## 4. Firestore のモデルとアクセス制御

旅行ごとのデータを `trips/{tripId}` の下に置く。旅行内の役割の正本は `members/{uid}`。会計上の参加者を `participants/{participantId}` に分けると、移行時に未登録者や退出者の支出を保存しつつ、アクセス権を付与せずに済む。

| パス | 主なデータ |
|---|---|
| `users/{uid}`、`users/{uid}/tripRefs/{tripId}` | 表示名、本人専用の旅行一覧索引。索引はIDと並び順に絞り、旅行名などは旅行文書から取得する。 |
| `trips/{tripId}`、配下の `members/{uid}` | 名称、期間、IANA タイムゾーン、行き先、状態、スキーマ版、台帳の版。メンバー文書には `admin / participant / viewer`、状態、任意の会計参加者ID。メールは含めない。 |
| `participants/{participantId}`、`places/{placeId}` | 会計用の表示名、参加状態、任意の紐付け UID。地点は名称、住所、座標。旅行外のID参照は拒否する。 |
| `plans/{planId}` と `plans/{planId}/comments/{id}`、`plans/{planId}/reactions/{uid}` | 予定種別、旅行の現地日、開始・終了、場所／移動の起終点、カテゴリ、メモ。コメントは投稿者 UID とサーバー時刻。リアクションは本人の1文書で更新する。移行した旧コメントは旧表示名と移行元を保持する。 |
| `expenses/{id}` | 支払者、整数円、通貨 JPY、負担方式、確定した負担額の配列、利用日、分類、内容、任意の予定ID、作成者 UID、版、取消状態。 |
| `settlements/{revision}` と配下の `transfers/{id}`、`events/{id}` | 台帳の版、提案、支払申告・受領確認・拒否・訂正の履歴、冪等キー。送金先は会計参加者ID。訂正は履歴に追記する。 |
| `media/{id}`、`media/{id}/comments/{id}`、`zipJobs/{id}` | 投稿者、予定／地点、撮影時刻、原本・サムネイル・プレビューのオブジェクト識別子、サイズ、型、ハッシュ、状態。ZIP は依頼者、対象、状態、期限。 |
| `invitations/{inviteId}`、`trips/{tripId}/audit/{id}` | 招待先メール照合用HMAC、付与予定ロール、状態、期限、招待者。招待情報と監査記録の読み書きはサーバー経由。 |

旅行期間と日別集計は現地日 `YYYY-MM-DD`、実時刻は Timestamp とタイムゾーンで保持する。予定は現地日＋開始時刻、支出は利用日＋カテゴリ／支払者、アルバムは予定／地点＋撮影時刻でページングする。安定したIDで同時刻の並びを決める。予定削除は参照済みデータからリンクを外すか、参照用の削除状態を残し、支出や写真の独立した履歴を保つ。

### 招待とサーバー操作

1. `createTrip` は認証済み UID の管理者文書、旅行、会計参加者、一覧索引を同じトランザクションで作成する。
2. `inviteByEmail` は呼び出し元が対象旅行の現役管理者であることを検証する。招待メールアドレスは空白と大小文字を正規化し、Gmail のドットや別名を勝手に統合しない。アプリ内受信箱を初期版の通知先とし、通知メールを使う場合も一般のログインURLへ案内する。メールの送信先は配信に必要な範囲でのみ扱う。
3. `listMyInvites` と `respondToInvite` は Firebase の認証情報とサーバー側のアカウント情報で Google ログインと検証済みメールを確認し、入力されたメールアドレスを照合の根拠にしない。Workspace の `hd` などを確認するときは、Google IDトークンの署名・発行元・対象・期限を検証し、`sub` を対象 Firebase ユーザーの Google プロバイダーIDと照合する。その他のメールを使う Google アカウントにはメールのワンタイム確認を追加する。承認時は招待の期限・失効・状態を再確認し、UIDのメンバー文書と一覧索引を原子的に書く。拒否、期限切れ、再承認での重複作成を防ぐ。 [Google のバックエンド認証ガイド](https://developers.google.com/identity/sign-in/web/backend-auth)
4. `changeMemberRole`、`removeMember` は管理者のみ。旅行の状態と呼び出し元の役割を操作と同じトランザクションで読み、最後の管理者の降格・退出を競合時も防ぐ。会計上の参加者と明細は保存する。将来の招待リンクは別経路として、推測困難なトークンのハッシュ、期限、使用回数、失効を追加する。
5. `saveExpense`、`startSettlement`、`recordTransfer`、`confirmTransfer` はサーバーで権限・版・金額・旅行内の参照を検証する。Admin SDK の処理は Rules を通らないため、各操作で認可し、サービスアカウントの IAM を用途別に絞る。通常の呼び出しは Firebase callable を第一候補とし、大きなバイト列の転送は専用経路にする。 [Callable Functions](https://firebase.google.com/docs/functions/callable) · [Firestore トランザクション](https://firebase.google.com/docs/firestore/manage-data/transactions)

旅行内の権限を Firestore 文書で都度確認すると、参加者が複数旅行で異なる役割を持て、権限変更後のリクエストに反映できる。Auth のカスタムクレームはサービス全体の運用権限に使う余地がある。旅行ごとの役割までトークンに入れると、ロール変更とトークン更新のずれを扱う必要がある。請求アカウントの紐付けを見送る場合もローカルで段階1〜4を進められる。公開運用には、別途信頼できる実行基盤と非公開ストレージの構成・費用を確定する。

| 操作 | 管理者 | 参加者 | 閲覧者 |
|---|---|---|---|
| 旅行・予定・アルバム・精算の閲覧／ダウンロード | 可 | 可 | 可 |
| 予定・地点の編集、コメント・写真・支出の追加 | 可 | 可 | 不可 |
| 写真・コメント・支出の訂正／取消 | 全体 | 自分が登録したもの。確定済み支出は精算の再開手続きが必要 | 不可 |
| 送金の申告／受領確認 | 当事者として操作可。管理者による代理訂正は理由を記録 | 自分の送金を申告／自分宛の受領を確認 | 不可 |
| 旅行情報の編集、招待、ロール変更、削除・復元 | 可 | 不可 | 不可 |

Firestore Rules は既定拒否とし、対象旅行の状態と `members/{request.auth.uid}` で都度認可する。利用者は自分の索引を読み、個別の旅行配下をクエリする。メンバー・索引・招待・監査・台帳・メディア状態のクライアント直接書き込みは拒否する。直接書き込みを許可する予定・コメント・リアクションは、許可フィールド、型、サイズ、作成者の不変性を検証する。アーカイブは読み取り専用、削除申請で通常アクセスを停止し、復元期限後にサーバージョブで子文書・オブジェクト・索引を再試行可能な手順で削除する。 [Firestore Rules とクエリ](https://firebase.google.com/docs/firestore/security/rules-conditions) · [Firestore の削除](https://firebase.google.com/docs/firestore/manage-data/delete-data)

## 5. 1円単位の精算

- 初期版は JPY の安全な整数とし、明細・累積金額に上限を設ける。「全員」は入力時点の有効な会計参加者とし、閲覧権限だけの人は対象から外す。均等割りは商を対象者全員に割り当て、余りの人数だけ安定した参加者ID順に1円ずつ足す。1,001円を3人なら334・334・333円。支払者が負担対象外の支出も登録できる。個別指定は負担者の重複、負数、合計不一致をサーバーで拒否する。保存時に対象と各負担額を確定するので、後からの参加・退会で過去の金額は変わらない。
- 各人の未精算残高は **立替額 − 負担額 ＋ 確認済みの送金額 − 確認済みの受取額**。正は受取、負は支払。全員の残高合計は常に0円。支払者の申告は確認待ちとして表示し、受取者が確認すると台帳に反映する。
- サーバーが台帳の版を固定して送金案を発行する。支払方向・当事者・未申告の残額を検証し、申告中の金額も予約して同時申告の超過を防ぐ。部分送金、再送の重複、拒否後の再申告、確認取り消しを同じ台帳で追跡する。申告後の明細訂正は管理者が精算を再開し、確認待ちを解決したうえで新しい案を作る。確認済み送金は新しい残高にも反映し、全員0円かつ確認待ち0件で精算完了にする。未登録者・退出者の送金は管理者が当事者から確認を取って代理で記録し、代理操作と理由を監査履歴に残す。
- 初期版は再現性のある貪欲法で送金案を作る。同額の並びをIDで固定し、残高がゼロに解消されることを検証する。UI は「送金案」と表示する。全体で厳密に最少の件数を求める場合は、少人数での探索と実行時間の上限を設ける追加機能とする。共有は OS の共有メニュー、コピー、LINE を用意する。 [LINE URL スキーム](https://developers.line.biz/en/docs/messaging-api/using-line-url-scheme/) · [Web Share API](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/share)

## 6. メディアの保存・配信と費用

**写真・短い動画の初期案は東京の Firebase Storage 基盤の Cloud Storage Standard。** ストレージは非公開。サーバーが役割と旅行枠を検証して再開可能なアップロードを開始し、端末から非公開の一時領域に直接送る。予定・スポットの参照、MIME と実ファイル、サイズ、ハッシュ、申請時と確定時の権限を確認して公開可能な状態にする。既定では原本のバイト列を保存し、閲覧用サムネイルは別オブジェクトにする。HEIC は原本を保存し、JPEG等のプレビューを用意する。動画は MP4/H.264 を優先して再生し、MOV/HEVC など端末で再生できない素材は原本ダウンロードを保証する。変換の利用量と再生互換性を見て、必要な形式に再生用プロキシを追加する。

Storage Rules はクライアントの直接読み書き・一覧を既定拒否し、専用のサーバー経路と IAM を使う。再開用セッションURIは限定された一時領域の秘密情報として扱い、ログや共有文には出さない。予約とオブジェクトIDで確定を冪等化し、期限切れ予約を解放して未確定ファイルを掃除する。閲覧・ダウンロードは認証済みメディア配信経由で、ファイルと `Range` リクエストごとにメンバー資格と旅行状態を再確認する。ブラウザの再生・ダウンロードにはメディアホスト限定の短いセッションCookieを使い、発行時のオリジンとログアウト時の失効を検証する。原本・プレビューは `private, no-store` とする。Firebase Hosting のプロキシには60秒の制約があるため、大容量ファイルの配信は専用の Cloud Run サービスで検証する。長寿命の `getDownloadURL` を保存する既存方式は移行し、旧URLのトークン失効までを本番切替の作業に含める。すでに利用者が端末へ保存したファイルは回収できない。 [Firebase のファイル取得](https://firebase.google.com/docs/storage/web/download-files) · [Cloud Storage の再開可能アップロード](https://docs.cloud.google.com/storage/docs/performing-resumable-uploads) · [Firebase セッションCookie](https://firebase.google.com/docs/auth/admin/manage-cookies) · [Hosting の制約](https://firebase.google.com/docs/hosting/cloud-run) · [Cloud Run の制限](https://docs.cloud.google.com/run/quotas)

| 方式 | 費用の目安と採用条件 |
|---|---|
| Firebase Storage／東京 Standard | 試算は保存約 **$0.023/GiB月**、日本向けストレージ直接配信の従量単価 **$0.12/GiB**。保存5 GB・月配信20 GBで約 **$2.34**、保存50 GB・月配信250 GBで約 **$29.01**。新規東京バケットに米国対象リージョンの Storage 無料枠は適用されず、Firebase Storage 利用に Blaze が必要。請求上のリージョン別保存単価と、採用する Cloud Run 経由の転送・実行料金は実測・見積もりで確定する。 |
| Cloudflare R2 Standard | 保存10 GB月、Class A 月100万回、Class B 月1,000万回まで無料枠。保存超過 $0.015/GB月、R2 のインターネット転送は無料。上の2例では操作が無料枠内なら **$0／約$0.60** のオブジェクト保存料。契約の有効化と別の認可・監視が必要。非公開 R2＋Worker で Range と都度認可を検証し、Worker 側の料金も計上する。署名付きURLだけで配信すると、URLを持つ人は有効期限まで利用できる。 |
| Cloudflare Stream＋原本保存 | 例として保存100分・月配信500分なら **$5.50/月** に原本保存費を加える。保存容量1,000分ごとに月$5、配信1,000分あたり$1。可変ビットレート再生と変換運用を任せられるが、提供されるダウンロードは変換後のMP4で、原本は別途保存が必要。再生需要と互換性対応が増えてから判断する。 |

試算は月を通じて同じ保存量、10進 GB から GiB への換算、税・操作料・DB・実行・バックアップ・一時ファイルを除いた概算。配信量にはプレビュー再生・再取得・ZIP も含めて計測する。例として50 GBのアルバムを5人が丸ごと取得すれば250 GBになる。初期値案は**旅行あたり原本5 GB、写真25 MB／件、動画250 MB／件**。予約時に同時アップロードを含めて容量を確保し、枠を超えたら追加受付を止める。自動再生を抑え、日・旅行単位で転送量とZIP実行数を制限する。予算アラートに加え、アップロード・ZIP受付を停止する設定を用意する。動画／ZIP配信が月50 GB前後に継続して達するか、実測費用が合意した予算を超えそうなら R2 の運用コストと比較する。 [Firebase の課金要件](https://firebase.google.com/docs/storage/faqs-storage-changes-announced-sept-2024) · [Google Cloud Storage 料金](https://cloud.google.com/storage/pricing) · [東京の保存 SKU](https://cloud.google.com/skus?currency=USD&filter=1845-1496-2891) · [Cloud Run 料金](https://cloud.google.com/run/pricing) · [R2 料金](https://developers.cloudflare.com/r2/pricing/) · [R2 の契約](https://developers.cloudflare.com/r2/get-started/) · [R2 署名付きURL](https://developers.cloudflare.com/r2/api/s3/presigned-urls/) · [Stream 料金](https://developers.cloudflare.com/stream/pricing/) · [Stream の原本取得](https://developers.cloudflare.com/stream/faq/) · [Firebase の予算アラート](https://firebase.google.com/docs/projects/billing/firebase-pricing-plans)

**ZIP と端末共有。** 選択した写真の原本をサーバージョブが順次読み出し、ZIPを非公開の一時領域に生成する。上限を超える場合は分割し、同一選択のジョブを再利用する。有効期間は24時間を初期案とし、生成・取得ともに認可する。対象の写真が削除されたら、それを含む既存ZIPも失効させる。原本の EXIF には位置情報が含まれ得るため、表示用画像から除去し、原本を共有・取得するときに説明する。iOS／Android ともアプリ内の写真・ファイル選択、OSへの共有、個別保存を提供し、`navigator.canShare` でファイル共有を判定する。大容量はダウンロードか認証付き画面URLを共有する。他アプリからの「共有で取り込む」は対応する Android のインストール済みPWAから検証し、iOS は写真選択を標準導線とする。iOS の共有先としての直接取り込みが必須なら、ネイティブの共有拡張を別案として見積もる。 [Web Share](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/share) · [Web Share Target](https://developer.chrome.com/docs/capabilities/web-apis/web-share-target)

## 7. 移行・バックアップ

1. 本番の実際のルール、プロジェクト／バケット種別とリージョン、旅行・予定・画像の件数、旧URL、管理者を誰に割り当てるかを読み取り専用で棚卸しする。旧URLやブラウザの表示名から権限を推定せず、ユーザーが承認した旅行IDと Google アカウントの対応表を使う。確認できない旅行は管理者用の隔離領域に置く。
2. 旅程、日付、コメント、リアクション、現存する写真をコピーする。旧コメントとリアクションの著者は由来を明示し、写真は取得できるバイト列をハッシュ照合する。圧縮・上書き前の原本が必要なら参加者の手元から再提供してもらう。旅行・予定IDは衝突しない範囲で保存し、旧URLからの案内は認証後にアクセス権を確認する。
3. warikan の保存はオリジン・端末・ブラウザに依存する。旧アプリを使った同じオリジン／ブラウザから JSON を取り出し、新画面のプレビューで対象旅行と支払者・負担者の対応を確認する。既存の確定配分は合計が正しければ保持し、旧形式で配分が無い場合は旧ロジックで復元して確認する。欠損、未知の名前、負担合計不一致は要確認として隔離し、利用日の分からない明細は作成時刻から黙って補完しない。既に行った送金は参加者確認後に取り込む。移行元IDで冪等化し、全件数・支出総額・負担総額・写真ハッシュを照合する。
4. Firestore の管理エクスポート、メディア原本・マニフェスト、Auth の UID 対応と必要なエクスポート、Rules・インデックス・ホスティング構成を保護された保存先に保管する。検証環境への復元でメンバー権限、残高、原本ハッシュを確認する。初期の運用目標案は日次バックアップ、移行直前バックアップ、RPO 24時間、RTO 4時間。管理エクスポートは Blaze が必要。切替では旧側を一時書き込み停止にしたうえで最終差分を取り込み、戻す場合も認証済みの旧版またはメンテナンス画面へ戻す。旧ルールと写真URLの公開経路を閉じる工程は承認後の切替に含める。 [Firestore のエクスポート・インポート](https://firebase.google.com/docs/firestore/manage-data/export-import)

## 8. 実装順と受け入れ条件

| 段階 | 成果物・進め方 | 受け入れ条件とテスト |
|---|---|---|
| 0 · 設計確定 | 実本番ルールの読み取り確認、データ棚卸し、権限表、金額仕様、媒体費用・上限の合意。 | 移行対象と管理者対応、停止期限の扱いを確定。両旧アプリの代表入力を匿名化フィクスチャにする。 |
| 1 · 認証・権限・CI | Emulator 用設定、Google Auth、旅行作成、招待・ロール、データモデル、ルール、監査。 | 未認証・別旅行・閲覧者・参加者・管理者で許可／拒否を自動検証。メール不一致、未検証、失効・期限切れ、二重承認、最終管理者の同時降格、クライアント直接昇格を拒否または安全に処理。 |
| 2 · 旅程・記録・PWA | 旅行 CRUD、アーカイブ・削除、予定・移動・地点、コメント・リアクション、地図・天気、インストール。 | 2端末で予定更新が反映。日跨ぎ・旅行タイムゾーン・直リンク更新・位置拒否・天気障害でも旅程が利用可能。削除・復元とオフライン画面、サインアウト後のデータ非表示を検証。 |
| 3 · 支出・精算 | 3つの負担方式、予定連携、3軸の集計、残高、送金案、部分送金・受領確認・訂正、LINE 共有、旧精算取り込み。 | 1円、1,001円÷3、支払者が対象外、0円配分、非整数・超過、退出、旧形式の不整合、同時送金・再送・古い版を検証。各明細の負担合計一致、全残高の総和0円、部分送金で残額だけが減り、完了後に残高0円。 |
| 4 · 写真・動画 | 非公開アルバム、原本・サムネイル、予定／スポット連携、再開アップロード、個別ダウンロード、ZIP、端末共有。 | 切断・再試行で重複せず、原本ハッシュ一致。偽MIME・容量超過・他旅行ID・削除済みメンバー・期限切れZIPを拒否。iPhone Safari／ホーム画面版、Android Chrome／インストール版で写真・動画・ZIPを実機確認。Range、HEIC、MOV、EXIF、失効、メモリと転送量を検証。 |
| 5 · 移行・限定運用 | 旧データの試験取り込み、バックアップ／復元、費用計測、承認後の切替。 | テスト環境に復元し、件数・権限・合計・ハッシュが一致。再実行が重複を作らず、未割当データを隔離できる。友人グループで旅行作成から精算完了・ZIP取得まで通し、費用・復元時間・切替手順を確認する。 |

CI は lint・型検査・ビルド、Vitest の純粋関数テスト、Emulator Suite の Rules／Functions 統合テスト、Playwright の主要フローで構成する。精算はランダムな整数配分でも総和不変条件を検証する。CI はエミュレーターと合成データを使い、Google OAuth、実バケットのCORS／Range、ネイティブ共有とPWAは分離した検証環境・実機で確認する。管理者の許可だけでなく、不正なID・別旅行・失効後・競合をテストする。 [Firebase Emulator Suite](https://firebase.google.com/docs/functions/local-emulator) · [Firestore Rules のテスト](https://firebase.google.com/docs/firestore/security/get-started#testing_rules)

設計レビューで先に決める項目は、旧旅行の管理者割当、月間予算と動画上限、初期の招待通知をアプリ内だけにするか、iOS で他アプリからの共有取り込みを必須にするかの4点。今回の作業はリポジトリの読み取り、ローカルの検証、公式資料の確認、設計書作成まで。
